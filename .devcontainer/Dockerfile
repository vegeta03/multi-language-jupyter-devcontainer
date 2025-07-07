# Use official DevContainers Python image with JupyterLab support
FROM mcr.microsoft.com/devcontainers/python:1-3-bullseye

# Set environment variables
ENV DEBIAN_FRONTEND=noninteractive
ENV SHELL=/bin/bash

# Update system packages
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    unzip \
    build-essential \
    git \
    && rm -rf /var/lib/apt/lists/*

# Install Node.js 20 LTS for TypeScript support
RUN curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get install -y nodejs

# Install Go 1.24.4
RUN wget https://go.dev/dl/go1.24.4.linux-amd64.tar.gz \
    && tar -C /usr/local -xzf go1.24.4.linux-amd64.tar.gz \
    && rm go1.24.4.linux-amd64.tar.gz
ENV PATH="/usr/local/go/bin:$PATH"
ENV GOPATH="/home/vscode/go"
ENV PATH="$GOPATH/bin:$PATH"

# Install Java 24 (OpenJDK)
RUN apt-get update && \
    apt-get install -y --no-install-recommends openjdk-24-jdk && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# set JAVA_HOME for OpenJDK 24
ENV JAVA_HOME=/usr/lib/jvm/java-24-openjdk-amd64
ENV PATH="$JAVA_HOME/bin:${PATH}"

# Install Scala 3.7.1
ARG SCALA_VERSION=3.7.1
RUN wget -q https://github.com/scala/scala3/releases/download/${SCALA_VERSION}/scala3-${SCALA_VERSION}-x86_64-pc-linux.tar.gz \
    && tar -xzf scala3-${SCALA_VERSION}-x86_64-pc-linux.tar.gz -C /usr/local/share \
    && ln -s /usr/local/share/scala3-${SCALA_VERSION} /usr/local/share/scala3 \
    && ln -s /usr/local/share/scala3/bin/scala  /usr/local/bin/scala  \
    && ln -s /usr/local/share/scala3/bin/scalac /usr/local/bin/scalac \
    && rm scala3-${SCALA_VERSION}-x86_64-pc-linux.tar.gz

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

# Install TypeScript Jupyter Kernel with autocompletion
RUN npm install -g typescript-jupyter-kernel typescript @types/node
RUN ts-kernel install --user

# Install GoNB kernel with autocompletion support
RUN go install github.com/janpfeifer/gonb@latest \
    && go install golang.org/x/tools/cmd/goimports@latest \
    && go install golang.org/x/tools/gopls@latest
RUN gonb --install --user

# Install IJava kernel for Java
RUN mkdir /tmp/ijava && cd /tmp/ijava \
    && wget https://github.com/SpencerPark/IJava/releases/download/v1.3.0/ijava-1.3.0.zip \
    && unzip ijava-1.3.0.zip \
    && python3 install.py --user \
    && cd /home/vscode && rm -rf /tmp/ijava

# Install Almond kernel for Scala
ENV SCALA_VERSION=3.7.1
ENV ALMOND_VERSION=0.17.0
RUN coursier bootstrap \
    -r jitpack \
    -i user -I user:sh.almond:scala-kernel-api_$SCALA_VERSION:$ALMOND_VERSION \
    sh.almond:scala-kernel_$SCALA_VERSION:$ALMOND_VERSION \
    --sources --default=true \
    -o /home/vscode/almond
RUN /home/vscode/almond --install --user
RUN rm -f /home/vscode/almond

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
