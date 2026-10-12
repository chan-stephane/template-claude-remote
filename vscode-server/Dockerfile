FROM lscr.io/linuxserver/code-server:latest

ENV DEBIAN_FRONTEND=noninteractive

# Outils dev + Python + Node.js 22 (une seule couche apt)
RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential \
      git curl wget ca-certificates gnupg \
      openssh-client \
      zsh \
      ripgrep jq fzf tree htop less procps \
      unzip xz-utils \
      python3 python3-pip python3-venv \
      postgresql-client \
      redis-tools \
 && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
 && apt-get install -y --no-install-recommends nodejs \
 && rm -rf /var/lib/apt/lists/*

# uv : gestionnaire Python rapide (venv, versions, paquets) — contourne la restriction PEP 668
RUN curl -LsSf https://astral.sh/uv/install.sh | env UV_INSTALL_DIR=/usr/local/bin sh

# Clés d'hôte GitHub au niveau système (lues par tous les utilisateurs, dont abc)
RUN ssh-keyscan -t ed25519,rsa github.com >> /etc/ssh/ssh_known_hosts 2>/dev/null \
 && chmod 644 /etc/ssh/ssh_known_hosts

# Identité Git au niveau système (/etc/gitconfig), visible par l'utilisateur abc
ARG GIT_USER_NAME
ARG GIT_USER_EMAIL
RUN git config --system user.name "${GIT_USER_NAME}" \
 && git config --system user.email "${GIT_USER_EMAIL}"

 