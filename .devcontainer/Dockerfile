FROM mcr.microsoft.com/vscode/devcontainers/javascript-node:24

RUN wget https://github.com/jgm/pandoc/releases/download/3.11/pandoc-3.11-1-amd64.deb && \
    sudo dpkg -i pandoc-3.11-1-amd64.deb && \
    pandoc -v

RUN npm install -g npm@12

RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
     && apt-get -y install --no-install-recommends inotify-tools