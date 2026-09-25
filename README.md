# Packer

[![Main](https://github.com/specsnl/packer/actions/workflows/main.yml/badge.svg)](https://github.com/specsnl/packer/actions/workflows/main.yml)

A Docker image with [HashiCorp Packer](https://developer.hashicorp.com/packer), built on top of
[specsnl/ansible](https://github.com/specsnl/ansible).

Pre-installed Packer plugins:

- [docker](https://github.com/hashicorp/packer-plugin-docker)
- [ansible](https://github.com/hashicorp/packer-plugin-ansible)
- [upcloud](https://github.com/UpCloudLtd/packer-plugin-upcloud)

Pulling image from GitHub Container Registry:

```bash
docker pull ghcr.io/specsnl/packer:latest
```

Interactive shell and mounting the current directory:

```bash
docker run -it -v $(pwd):/workspace --rm ghcr.io/specsnl/packer:latest /bin/bash
```

Default workspace: `/workspace`

## Task

This project uses [Task](https://taskfile.dev) (an task runner / build tool).

Available tasks for this project:

```
* build:       Build the Packer image
* lint:        Apply a Dockerfile linter (https://github.com/hadolint/hadolint)
* shell:       Interactive shell with Packer
```
