# syntax=docker/dockerfile:1
# check=error=true

# Latest version: https://hub.docker.com/r/hashicorp/packer/tags?name=light
FROM hashicorp/packer:light-1.16.0 AS packer

# Latest version: https://github.com/specsnl/ansible/releases
FROM ghcr.io/specsnl/ansible:1.12.0

ENV PACKER_CONFIG_DIR=/root/.config/packer
ENV PACKER_PLUGIN_PATH=${PACKER_CONFIG_DIR}/plugins

COPY --from=packer /bin/packer /bin/packer

RUN packer plugins install github.com/hashicorp/docker \
    && packer plugins install github.com/hashicorp/ansible \
    && packer plugins install github.com/UpCloudLtd/upcloud
