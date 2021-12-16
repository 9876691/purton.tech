FROM ianpurton/rust-fullstack-devcontainer:latest AS development

COPY *.bash .

RUN cat *.bash >> ~/.bashrc && sudo rm *.bash

# Zola
ARG ZOLA_VERSION=0.12.2
RUN sudo curl -OL https://github.com/getzola/zola/releases/download/v$ZOLA_VERSION/zola-v$ZOLA_VERSION-x86_64-unknown-linux-gnu.tar.gz \
    && sudo tar -xvf zola-v$ZOLA_VERSION-x86_64-unknown-linux-gnu.tar.gz \
    && sudo mv zola /usr/bin/zola \
    && sudo chmod +x /usr/bin/zola

RUN sudo curl -OL https://github.com/gohugoio/hugo/releases/download/v0.90.1/hugo_0.90.1_Linux-64bit.tar.gz \
    && sudo tar -xvf hugo_0.90.1_Linux-64bit.tar.gz \
    && sudo mv hugo /usr/bin/hugo \
    && sudo chmod +x /usr/bin/hugo

# Enable our git hooks and set the permisisons on docker sock.
RUN echo 'sudo chmod 777 /var/run/docker.sock' >> ~/.bashrc

# all the volumes configured in the docker-compose.yml
ARG USERNAME=vscode
