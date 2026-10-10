# template-claude-remote

Template Docker pour faire tourner [Claude Code](https://claude.com/claude-code) en mode **remote control** sur une machine distante (VPS, serveur perso, etc.). Une fois le conteneur lancé, la session est pilotable depuis claude.ai ou l'application mobile, et travaille sur le dossier monté dans `/workspace`.

Un service [code-server](https://hub.docker.com/r/linuxserver/code-server) (VS Code dans le navigateur) est lancé à côté, sur le même `/workspace`, pour consulter et éditer les fichiers.

## Services

| Service         | Rôle                                   | Accès                      |
| --------------- | -------------------------------------- | -------------------------- |
| `claude-remote` | Session Claude Code en remote control  | claude.ai / app mobile     |
| `code-server`   | VS Code dans le navigateur             | `http://<ip-du-serveur>:8443` |

## Contenu de l'image `claude-remote`

- Ubuntu 24.04
- Claude Code CLI (installeur officiel, version configurable)
- Node.js 22, Python 3 (`pip`, `venv`), `build-essential`
- Git, client OpenSSH (clés d'hôte GitHub pré-enregistrées dans `known_hosts`)
- Outils : `ripgrep`, `jq`, `fzf`, `tree`, `htop`, `zsh`, `curl`, `wget`, `unzip`

Au démarrage, le conteneur exécute :

```bash
claude remote-control --name "${DEVICE_NAME}"
```

## Prérequis

- Docker et Docker Compose
- Un compte Claude
- Une clé SSH autorisée sur GitHub (pour cloner/pousser depuis le conteneur)

## Configuration

### 1. Variables d'environnement

```bash
cp .env.example .env
```

Le fichier `.env` est chargé par les deux services (`env_file`).

**Claude Code remote**

| Variable         | Description                                     |
| ---------------- | ----------------------------------------------- |
| `DEVICE_NAME`    | Nom sous lequel la session apparaît dans Claude |
| `GIT_USER_NAME`  | `user.name` Git configuré dans l'image          |
| `GIT_USER_EMAIL` | `user.email` Git configuré dans l'image         |

`GIT_USER_NAME` et `GIT_USER_EMAIL` sont injectés au build : après modification, il faut reconstruire l'image (`docker compose build`).

**code-server**

| Variable        | Description                                                          |
| --------------- | -------------------------------------------------------------------- |
| `PASSWORD`      | Mot de passe de l'interface web (sans lui, l'accès n'est pas protégé) |
| `SUDO_PASSWORD` | Mot de passe `sudo` dans le terminal de code-server                  |
| `PWA_APPNAME`   | Nom affiché lorsque code-server est installé comme application (PWA) |
| `TZ`            | Fuseau horaire (ex. `Etc/UTC`)                                       |
| `PUID` / `PGID` | Utilisateur/groupe sous lequel tourne code-server (`0` = `root`)     |

`PUID`/`PGID` valent `0` car `claude-remote` écrit dans le workspace en `root` : avec un autre utilisateur, code-server ne pourrait pas modifier les fichiers créés par Claude.

Le port de code-server est fixé à `8443` ; pour en exposer un autre sur l'hôte, modifier le mapping `ports` dans `docker-compose.yaml` (ex. `9000:8443`).

### 2. Volumes

Dans `docker-compose.yaml`, remplacer les chemins d'exemple, **dans les deux services** (même clé SSH, même workspace) :

```yaml
# claude-remote
volumes:
  - /path/to/your/ssh_github_key:/root/.ssh/id_ed25519:ro   # clé SSH privée GitHub
  - claude-config:/root/.claude                             # config + authentification Claude
  - /path/to/your/workspace:/workspace                      # dossier de travail

# code-server
volumes:
  - code-server-config:/config                                # réglages et extensions VS Code
  - /path/to/your/ssh_github_key:/config/.ssh/id_ed25519:ro   # clé SSH privée GitHub
  - /path/to/your/workspace:/workspace                        # dossier de travail
```

Les volumes nommés `claude-config` et `code-server-config` conservent respectivement l'authentification Claude et la configuration de code-server entre les redémarrages.

## Utilisation

### Première connexion

À faire une seule fois, pour authentifier Claude :

```bash
docker compose run --rm claude-remote claude auth login
```

Suivre le lien affiché pour se connecter. Les identifiants sont enregistrés dans le volume `claude-config`.

### Démarrage

Une fois connecté, il suffit de lancer :

```bash
docker compose up -d
```

La session apparaît alors dans Claude sous le nom défini par `DEVICE_NAME`, et code-server est accessible sur `http://<ip-du-serveur>:8443`.

### code-server

Se connecter avec le mot de passe défini dans `PASSWORD` ; l'éditeur s'ouvre directement sur `/workspace`.

Pour utiliser Git depuis le terminal de code-server, configurer l'identité une fois (elle est conservée dans le volume `code-server-config`) :

```bash
git config --global user.name "username"
git config --global user.email "email address"
```

### Commandes utiles

```bash
docker compose logs -f                      # suivre les logs
docker compose exec claude-remote bash      # ouvrir un shell dans le conteneur
docker compose down                         # arrêter
docker compose build --no-cache             # reconstruire (ex. mise à jour de Claude Code)
docker compose pull code-server             # mettre à jour code-server
```

## Version de Claude Code

Par défaut, l'image installe la dernière version. Pour en fixer une, ajouter l'argument de build `CLAUDE_CODE_VERSION` dans `docker-compose.yaml` :

```yaml
build:
  args:
    - CLAUDE_CODE_VERSION=<version>
```

## Sécurité

- Le conteneur tourne en `root` avec `IS_SANDBOX=1` : ne monter dans `/workspace` que ce à quoi Claude doit avoir accès.
- code-server est servi en HTTP simple et donne un terminal `root` sur le workspace : toujours définir `PASSWORD` avec une valeur forte et, sur un serveur exposé à Internet, le placer derrière un reverse proxy HTTPS ou un VPN plutôt que d'ouvrir le port `8443` directement.
- La clé SSH est montée en lecture seule ; privilégier une clé dédiée (deploy key) aux droits limités.
- `.env`, `deploy_key`, `workspace/` et `claude-config/` sont ignorés par Git : ne jamais les committer.
