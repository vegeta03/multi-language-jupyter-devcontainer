# Use official DevContainers Python image with JupyterLab support
FROM mcr.microsoft.com/devcontainers/python:3.13-bookworm

# Set environment variables
ENV DEBIAN_FRONTEND=noninteractive
ENV SHELL=/bin/bash

# Update system packages
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    unzip \
    git \
    build-essential \
    libtool \
    pkg-config \
    automake \
    autoconf \
    && rm -rf /var/lib/apt/lists/*

# Install Node.js 20 LTS for TypeScript support (required for JupyterLab 4.0)
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs

# Install global npm packages as root BEFORE switching users
RUN npm install -g typescript-jupyter-kernel typescript @types/node @jupyterlab/server-proxy

# Install Go 1.24.4
RUN wget https://go.dev/dl/go1.24.4.linux-amd64.tar.gz \
    && tar -C /usr/local -xzf go1.24.4.linux-amd64.tar.gz \
    && rm go1.24.4.linux-amd64.tar.gz

ENV PATH="/usr/local/go/bin:$PATH"
ENV GOPATH="/home/vscode/go"
ENV PATH="$GOPATH/bin:$PATH"

# Install Java 24 (Eclipse Temurin 24.0.1) from Adoptium
ENV JAVA_VERSION=24 \
    JAVA_HOME=/usr/lib/jvm/java-24-openjdk-amd64

RUN wget -O /tmp/openjdk.tar.gz \
    https://github.com/adoptium/temurin24-binaries/releases/download/jdk-24.0.1%2B9/OpenJDK24U-jdk_x64_linux_hotspot_24.0.1_9.tar.gz \
    && mkdir -p "$JAVA_HOME" \
    && tar -xzf /tmp/openjdk.tar.gz -C "$JAVA_HOME" --strip-components=1 \
    && rm /tmp/openjdk.tar.gz

ENV PATH="$JAVA_HOME/bin:$PATH"

# Install Coursier for Scala package management (AS ROOT)
RUN curl -fLo coursier https://git.io/coursier-cli-linux \
    && chmod +x coursier \
    && mv coursier /usr/local/bin/

# Install cs command AS ROOT to avoid permission issues
RUN coursier bootstrap -o /usr/local/bin/cs coursier:2.1.0 && \
    chmod +x /usr/local/bin/cs

# Switch to vscode user for user-specific installations
USER vscode
WORKDIR /home/vscode

# Install JupyterLab and extensions
RUN pip install --user --no-cache-dir \
    jupyterlab \
    jupyter-lsp \
    jupyterlab-lsp

# Fix PATH for user-installed Python packages
ENV PATH="/home/vscode/.local/bin:$PATH"

# Install TypeScript Jupyter Kernel manually (avoiding CLI bug)
RUN mkdir -p /home/vscode/.local/share/jupyter/kernels/typescript && \
    echo '{ \
        "argv": ["ts-kernel", "{connection_file}"], \
        "display_name": "TypeScript", \
        "language": "typescript", \
        "metadata": { \
            "debugger": true \
        } \
    }' > /home/vscode/.local/share/jupyter/kernels/typescript/kernel.json

# Install GoNB kernel with autocompletion support
RUN go install github.com/janpfeifer/gonb@v0.11.1 \
    && go install golang.org/x/tools/cmd/goimports@latest \
    && go install golang.org/x/tools/gopls@latest

# Install GoNB kernel (no --user flag needed)
RUN gonb --install

# Install IJava kernel for Java
RUN mkdir /tmp/ijava && cd /tmp/ijava \
    && wget https://github.com/SpencerPark/IJava/releases/download/v1.3.0/ijava-1.3.0.zip \
    && unzip ijava-1.3.0.zip \
    && python3 install.py --user \
    && cd /home/vscode && rm -rf /tmp/ijava

# Install Almond kernel for Scala - FIXED VERSION WITH CORRECT CLI INTERFACE
ENV SCALA_VERSION=3.3.0
ENV ALMOND_VERSION=0.14.1

# Create coursier bin directory and add to PATH
RUN mkdir -p /home/vscode/.local/share/coursier/bin
ENV PATH="/home/vscode/.local/share/coursier/bin:${PATH}"

# Use cs setup to install Scala toolchain properly
RUN cs setup --yes --install-dir /home/vscode/.local/share/coursier/bin

# Verify Scala installation
RUN which scala || echo "Scala not found in PATH, using cs launch for wrapper scripts"

# Create wrapper scripts for scala and scalac if they don't exist (fallback)
RUN if ! which scala >/dev/null 2>&1; then \
        echo '#!/bin/bash' > /home/vscode/.local/share/coursier/bin/scala && \
        echo 'exec cs launch scala:'"${SCALA_VERSION}"' -- "$@"' >> /home/vscode/.local/share/coursier/bin/scala && \
        chmod +x /home/vscode/.local/share/coursier/bin/scala; \
    fi

RUN if ! which scalac >/dev/null 2>&1; then \
        echo '#!/bin/bash' > /home/vscode/.local/share/coursier/bin/scalac && \
        echo 'exec cs launch scalac:'"${SCALA_VERSION}"' -- "$@"' >> /home/vscode/.local/share/coursier/bin/scalac && \
        chmod +x /home/vscode/.local/share/coursier/bin/scalac; \
    fi

# Install Almond kernel using updated method with correct repositories and CLI interface
RUN coursier bootstrap \
    -r central \
    -r https://oss.sonatype.org/content/repositories/snapshots \
    sh.almond:scala-kernel_3.6.3:${ALMOND_VERSION} \
    --sources --default=true \
    -o /home/vscode/almond || \
    cs install almond --channel https://github.com/almond-sh/almond || \
    cs install --contrib almond

# Register Almond kernel (FIXED: removed --user flag for 0.14.1+ compatibility)
RUN if [ -f /home/vscode/almond ]; then \
        /home/vscode/almond --install && rm /home/vscode/almond; \
    elif which almond >/dev/null 2>&1; then \
        almond --install; \
    else \
        cs launch almond -- --install; \
    fi

# Verify Almond kernel installation
RUN jupyter kernelspec list | grep -i scala || echo "Scala kernel installation verification failed, but continuing build"

# Create JupyterLab configuration for autocompletion
RUN mkdir -p /home/vscode/.jupyter/lab/user-settings/@jupyterlab/completer-extension/ \
    && echo '{"autoCompletion": {"enabled": true, "continuous": true}}' > \
    /home/vscode/.jupyter/lab/user-settings/@jupyterlab/completer-extension/manager.jupyterlab-settings

# Switch back to root for final setup
USER root

# Expose JupyterLab port
EXPOSE 8888

# Set final user
USER vscode

# Default command
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--port=8888", "--no-browser", "--allow-root"]
