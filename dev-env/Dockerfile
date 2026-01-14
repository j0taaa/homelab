FROM node:latest

# Install system utilities
RUN apt-get update && apt-get install -y \
    git \
    curl \
    wget \
    vim \
    nano \
    zip \
    unzip \
    jq \
    tree \
    htop \
    ca-certificates \
    gnupg \
    lsb-release \
    && rm -rf /var/lib/apt/lists/*

# Install Docker CLI
RUN curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list && \
    apt-get update && \
    apt-get install -y docker-ce-cli && \
    rm -rf /var/lib/apt/lists/*

# Install OpenCode system-wide
RUN curl -fsSL https://opencode.ai/install | bash && \
    mv /root/.opencode/bin/opencode /usr/local/bin/opencode

# Install Claude Code and Vercel CLI globally
RUN npm install -g @anthropic-ai/claude-code vercel

# Git configuration - set these when building or override at runtime
ARG GIT_USER_NAME="Your Name"
ARG GIT_USER_EMAIL="your.email@example.com"

WORKDIR /app

# Create a non-root user for security
RUN groupadd -r nodejs && useradd -r -g nodejs -m nodejs

# Set ownership of the app directory
RUN chown -R nodejs:nodejs /app

USER nodejs

# Configure Git for nodejs user
RUN git config --global user.name "${GIT_USER_NAME}" && \
    git config --global user.email "${GIT_USER_EMAIL}" && \
    git config --global init.defaultBranch main && \
    git config --global credential.helper store

# Store GitHub credentials if PAT is provided via environment variable
CMD if [ -n "$GITHUB_PAT" ]; then \
      echo "https://${GITHUB_USER}:${GITHUB_PAT}@github.com" > ~/.git-credentials; \
    fi && \
    exec bash
