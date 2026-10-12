# template-claude-remote

Template Docker pour faire tourner [Claude Code](https://claude.com/claude-code) en mode **remote control** sur une machine distante (VPS, serveur perso, etc.). Une fois la stack lancée, la session est pilotable depuis claude.ai ou l'application mobile, et travaille sur le dossier monté dans `/workspace`.

À côté de Claude, la stack fournit un environnement de développement complet :

- [code-server](https://hub.docker.com/r/linuxserver/code-server) (VS Code dans le navigateur), sur le même `/workspace`, pour consulter et éditer les fichiers ;
- une base **PostgreSQL** et un **Redis**, accessibles depuis Claude et code-server pour les projets développés dans le workspace.

## Services

| Service         | Image                                   | Rôle                                  | Accès                              |
| --------------- | --------------------------------------- | ------------------------------------- | ---------------------------------- |
| `claude-remote` | `claude-remote:latest` (build local)    | Session Claude Code en remote control | claude.ai / app mobile             |
| `code-server`   | `custom-code-server:latest` (build local) | VS Code dans le navigateur          | `http://<ip-du-serveur>:8443`      |
| `postgres`      | `postgres:16-alpine`                    | Base de données PostgreSQL            | `postgres:5432` (réseau interne)   |
| `redis`         | `redis:8.10-alpine`                     | Cache / broker Redis                  | `redis:6379` (réseau interne)      |

Les quatre services partagent le réseau Docker `claude-network`. `claude-remote` et `code-server` ne démarrent qu'une fois `postgres` et `redis` déclarés sains (`healthcheck`).

## Structure du dépôt

```text
.
├── docker-compose.yaml
├── .env.example            # variables de claude-remote et code-server
├── claude-remote/
│   └── Dockerfile          # image Claude Code
├── vscode-server/
│   └── Dockerfile          # image code-server enrichie
└── databases/
    └── .env.example        # identifiants PostgreSQL
```

## Contenu des images

### `claude-remote`

- Ubuntu 24.04
- Claude Code CLI (installeur officiel, version configurable)
- Node.js 22, Python 3 (`pip`, `venv`), `build-essential`
- Git, client OpenSSH (clés d'hôte GitHub pré-enregistrées dans `known_hosts`)
- Outils : `ripgrep`, `jq`, `fzf`, `tree`, `htop`, `zsh`, `curl`, `wget`, `unzip`

Au démarrage, le conteneur exécute :

```bash
claude remote-control --name "${DEVICE_NAME}"
```

### `code-server`

Basée sur `lscr.io/linuxserver/code-server:latest`, avec en plus :

- Node.js 22, Python 3 (`pip`, `venv`), [`uv`](https://docs.astral.sh/uv/), `build-essential`
- Clients de base de données : `psql` (`postgresql-client`) et `redis-cli` (`redis-tools`)
- Git, client OpenSSH (clés d'hôte GitHub enregistrées au niveau système)
- Outils : `ripgrep`, `jq`, `fzf`, `tree`, `htop`, `zsh`, `curl`, `wget`, `unzip`
- Identité Git configurée au niveau système (`/etc/gitconfig`) à partir de `GIT_USER_NAME` / `GIT_USER_EMAIL`

## Prérequis

- Docker et Docker Compose
- Un compte Claude
- Une clé SSH autorisée sur GitHub (pour cloner/pousser depuis les conteneurs)

## Configuration

### 1. Variables d'environnement

```bash
cp .env.example .env
cp databases/.env.example databases/.env
```

`claude-remote` et `code-server` chargent les deux fichiers (`env_file`) : ils disposent donc aussi des identifiants PostgreSQL.

**Claude Code remote** (`.env`)

| Variable         | Description                                     |
| ---------------- | ----------------------------------------------- |
| `DEVICE_NAME`    | Nom sous lequel la session apparaît dans Claude |
| `GIT_USER_NAME`  | `user.name` Git configuré dans les deux images  |
| `GIT_USER_EMAIL` | `user.email` Git configuré dans les deux images |

`GIT_USER_NAME` et `GIT_USER_EMAIL` sont injectés au build : après modification, il faut reconstruire les images (`docker compose build`).

**code-server** (`.env`)

| Variable        | Description                                                           |
| --------------- | --------------------------------------------------------------------- |
| `PASSWORD`      | Mot de passe de l'interface web (sans lui, l'accès n'est pas protégé) |
| `SUDO_PASSWORD` | Mot de passe `sudo` dans le terminal de code-server                   |
| `PWA_APPNAME`   | Nom affiché lorsque code-server est installé comme application (PWA)  |
| `TZ`            | Fuseau horaire (ex. `Etc/UTC`)                                        |
| `PUID` / `PGID` | Utilisateur/groupe sous lequel tourne code-server (`0` = `root`)      |

`PUID`/`PGID` valent `0` car `claude-remote` écrit dans le workspace en `root` : avec un autre utilisateur, code-server ne pourrait pas modifier les fichiers créés par Claude.

Le port de code-server est fixé à `8443` ; pour en exposer un autre sur l'hôte, modifier le mapping `ports` dans `docker-compose.yaml` (ex. `9000:8443`).

**PostgreSQL** (`databases/.env`)

| Variable            | Description                             |
| ------------------- | --------------------------------------- |
| `POSTGRES_USER`     | Utilisateur créé au premier démarrage   |
| `POSTGRES_PASSWORD` | Mot de passe de cet utilisateur         |
| `POSTGRES_DB`       | Base créée au premier démarrage         |

Ces valeurs ne sont prises en compte qu'à l'initialisation du volume `postgres-data` : pour les changer ensuite, modifier l'utilisateur en SQL ou supprimer le volume (perte des données). Remplacer les valeurs d'exemple (`postgres`/`postgres`) par un vrai mot de passe.

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

Volumes nommés :

| Volume               | Contenu                                   |
| -------------------- | ----------------------------------------- |
| `claude-config`      | Configuration et authentification Claude  |
| `code-server-config` | Réglages et extensions de code-server     |
| `postgres-data`      | Données PostgreSQL                        |

Redis n'a pas de volume nommé : son contenu n'est pas conservé lorsque le conteneur est recréé.

## Utilisation

### Première connexion

À faire une seule fois, pour authentifier Claude :

```bash
docker compose run --rm claude-remote claude auth login
```

Suivre le lien affiché pour se connecter. Les identifiants sont enregistrés dans le volume `claude-config`. La commande construit l'image si nécessaire et démarre `postgres` et `redis` au passage.

### Démarrage

Une fois connecté, il suffit de lancer :

```bash
docker compose up -d
```

La session apparaît alors dans Claude sous le nom défini par `DEVICE_NAME`, et code-server est accessible sur `http://<ip-du-serveur>:8443`.

### code-server

Se connecter avec le mot de passe défini dans `PASSWORD` ; l'éditeur s'ouvre directement sur `/workspace`. Git est déjà configuré avec l'identité fournie au build.

### Bases de données

PostgreSQL et Redis ne sont pas exposés sur l'hôte : ils sont joignables uniquement depuis les conteneurs du réseau `claude-network`, par leur nom de service.

| Service    | Hôte       | Port   | Authentification                       |
| ---------- | ---------- | ------ | -------------------------------------- |
| PostgreSQL | `postgres` | `5432` | `POSTGRES_USER` / `POSTGRES_PASSWORD`  |
| Redis      | `redis`    | `6379` | aucune                                 |

Les variables `POSTGRES_*` sont disponibles dans l'environnement de `claude-remote` et de `code-server`. Exemple d'URL de connexion pour une application du workspace :

```text
postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
redis://redis:6379/0
```

Depuis le terminal de code-server :

```bash
PGPASSWORD="$POSTGRES_PASSWORD" psql -h postgres -U "$POSTGRES_USER" -d "$POSTGRES_DB"
redis-cli -h redis ping
```

L'image `claude-remote` n'embarque pas `psql` ni `redis-cli` ; Claude peut néanmoins se connecter via les bibliothèques des projets (Node.js, Python).

Pour atteindre les bases depuis l'hôte (client graphique, par exemple), ajouter un mapping `ports` aux services concernés, de préférence lié à `127.0.0.1` :

```yaml
postgres:
  ports:
    - 127.0.0.1:5432:5432
```

### Commandes utiles

```bash
docker compose logs -f                      # suivre les logs
docker compose ps                           # état et santé des services
docker compose exec claude-remote bash      # ouvrir un shell dans le conteneur Claude
docker compose exec postgres sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"'   # console PostgreSQL
docker compose down                         # arrêter (les volumes sont conservés)
docker compose build --no-cache claude-remote   # reconstruire (ex. mise à jour de Claude Code)
docker compose build --pull code-server     # mettre à jour code-server (nouvelle image de base)
docker compose pull postgres redis          # mettre à jour les images des bases
```

## Version de Claude Code

Par défaut, l'image installe la dernière version. Pour en fixer une, ajouter l'argument de build `CLAUDE_CODE_VERSION` au service `claude-remote` dans `docker-compose.yaml` :

```yaml
build:
  args:
    - CLAUDE_CODE_VERSION=<version>
```

## Sécurité

- Le conteneur `claude-remote` tourne en `root` avec `IS_SANDBOX=1` : ne monter dans `/workspace` que ce à quoi Claude doit avoir accès.
- code-server est servi en HTTP simple et donne un terminal `root` sur le workspace : toujours définir `PASSWORD` avec une valeur forte et, sur un serveur exposé à Internet, le placer derrière un reverse proxy HTTPS ou un VPN plutôt que d'ouvrir le port `8443` directement.
- PostgreSQL et Redis ne publient aucun port ; Redis n'a pas de mot de passe. Ne pas les exposer sur une interface publique sans ajouter d'authentification.
- Claude et code-server ont accès aux identifiants PostgreSQL : n'y stocker que des données de développement.
- La clé SSH est montée en lecture seule ; privilégier une clé dédiée (deploy key) aux droits limités.
- `.env`, `databases/.env`, `deploy_key`, `workspace/` et `claude-config/` sont ignorés par Git : ne jamais les committer.
