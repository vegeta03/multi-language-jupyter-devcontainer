# Use official DevContainers Python image with JupyterLab support
FROM mcr.microsoft.com/devcontainers/python:1-3-bookworm

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

# Install Coursier for Scala package management
RUN curl -fLo coursier https://git.io/coursier-cli-linux \
    && chmod +x coursier \
    && mv coursier /usr/local/bin/

# Switch to vscode user for installations
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

# Install Almond kernel for Scala
ENV SCALA_VERSION=3.3.0
ENV ALMOND_VERSION=0.13.8

# Create coursier bin directory and add to PATH
RUN mkdir -p /home/vscode/.local/share/coursier/bin
ENV PATH="/home/vscode/.local/share/coursier/bin:${PATH}"

# Use cs setup to install Scala toolchain properly
RUN coursier bootstrap -o /usr/local/bin/cs coursier:2.1.0 && \
    chmod +x /usr/local/bin/cs

# Install Scala using cs setup (avoids prebuilt launcher issues)
RUN cs setup --yes --install-dir /home/vscode/.local/share/coursier/bin

# Install specific Scala version if needed
RUN cs install --install-dir /home/vscode/.local/share/coursier/bin \
    scala:${SCALA_VERSION} --force || \
    cs launch scala:${SCALA_VERSION} -- -version

# Bootstrap Almond and register the kernel
RUN coursier bootstrap \
    -r jitpack \
    sh.almond:scala-kernel_3:${ALMOND_VERSION} \
    --sources --default=true \
    -o /home/vscode/almond && \
    /home/vscode/almond --install --user && \
    rm /home/vscode/almond

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
