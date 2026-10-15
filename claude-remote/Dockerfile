FROM ubuntu:24.04

ARG CLAUDE_CODE_VERSION=latest

# Prevent interactive prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

# Developer tooling (sudo retiré : inutile en root)
RUN apt-get update && apt-get install -y \
      build-essential \
      git curl wget ca-certificates \
      openssh-client \
      bash zsh \
      ripgrep jq fzf tree htop less \
      procps \
      unzip xz-utils \
      python3 python3-pip python3-venv \
    && rm -rf /var/lib/apt/lists/*

# Install Node.js 22 via NodeSource
RUN curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && \
    apt-get install -y nodejs && \
    rm -rf /var/lib/apt/lists/*

# Pre-populate SSH known_hosts with GitHub host keys (avoids interactive prompt)
RUN mkdir -p /root/.ssh && \
    ssh-keyscan -t ed25519,rsa github.com >> /root/.ssh/known_hosts 2>/dev/null && \
    chmod 700 /root/.ssh && \
    chmod 644 /root/.ssh/known_hosts

# Prepare Claude config directory, onboarding marker, and workspace trust
RUN mkdir -p /root/.claude && \
    echo '{"hasCompletedOnboarding":true,"installMethod":"native","remoteDialogSeen":true,"projects":{"/workspace":{"hasTrustDialogAccepted":true,"allowedTools":[]}}}' > /root/.claude/.claude.json

# Install Claude Code CLI using the official installer
RUN curl -fsSL https://claude.ai/install.sh | bash -s -- "${CLAUDE_CODE_VERSION}" && \
    /root/.local/bin/claude --version

ENV PATH="/root/.local/bin:${PATH}"

ENV IS_SANDBOX=1

ENV CLAUDE_CONFIG_DIR=/root/.claude

RUN mkdir -p /workspace

WORKDIR /workspace

ARG GIT_USER_NAME
ARG DEVICE_NAME
ARG GIT_USER_EMAIL

ENV GIT_USER_NAME=${GIT_USER_NAME}
ENV GIT_USER_EMAIL=${GIT_USER_EMAIL}
ENV DEVICE_NAME=${DEVICE_NAME}

RUN git config --global user.name "${GIT_USER_NAME}" && \
    git config --global user.email "${GIT_USER_EMAIL}"

RUN claude mcp add playwright -- npx @playwright/mcp@latest --cdp-endpoint http://browser:9222 --cdp-header "Authorization: Bearer ${CDP_AUTH_TOKEN}"

CMD claude remote-control --name "${DEVICE_NAME}"