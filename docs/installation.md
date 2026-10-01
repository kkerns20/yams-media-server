# YAMS Installation Notes

This document explains how I built and verified my YAMS media server on Ubuntu 24.04 LTS.

The goal is not just to record commands, but to understand what each layer does:

- Ubuntu provides the host operating system.
- Docker runs the applications in containers.
- Docker Compose manages the multi-container stack.
- YAMS provides the media-server application stack and helper scripts.
- Gluetun provides VPN networking for qBittorrent and SABnzbd.
- Mullvad provides the WireGuard VPN service.

## 1. Host preparation

This server runs Ubuntu 24.04 LTS.

Before installing or troubleshooting YAMS, confirm the operating system:

```bash
cat /etc/os-release
```

Check available disk space:

```bash
df -h
```

Confirm the current user:

```bash
whoami
```

For this build:

- Docker is installed at `/usr/bin/docker`
- YAMS is installed under `/opt/yams`
- Media storage is located under `/srv/media`

## 2. Verify Docker first

YAMS depends on Docker, so Docker should be tested independently before troubleshooting the YAMS stack.

Check Docker Engine:

```bash
docker --version
```

Check Docker Compose:

```bash
docker compose version
```

Verify that the Docker daemon is running:

```bash
systemctl status docker
```

Finally, test Docker with:

```bash
docker run hello-world
```

If `hello-world` succeeds without `sudo`, that confirms:

- the Docker daemon is running;
- the current user has permission to use Docker;
- Docker can download and run an image successfully.

## 3. Understanding the YAMS installation

The current YAMS installation lives under:

```text
/opt/yams/
├── .env
├── docker-compose.yaml
├── docker-compose.custom.yaml
├── config/
└── yams
```

Important pieces:

- `.env` stores environment-specific values and can contain secrets.
- `docker-compose.yaml` defines the main application stack.
- `docker-compose.custom.yaml` is used for custom overrides or additions.
- `config/` stores persistent application configuration.
- `yams` is the helper script used to manage the stack.

Because `.env` can contain credentials and private keys, it should never be committed to a public repository.

## 4. Starting and inspecting the stack

YAMS provides helper commands for managing the stack:

```bash
yams start
yams stop
yams restart
yams status
yams check-vpn
```

Underneath those helper commands, Docker Compose manages the containers.

Useful commands for inspecting the stack:

```bash
docker ps
docker ps -a
docker compose ps -a
docker compose config --services
```

`docker ps` shows running containers.

`docker ps -a` also shows stopped containers.

`docker compose ps -a` shows the state of the containers that belong to the YAMS Compose project.

A container being `Up` does not necessarily mean the service inside it is healthy. Docker healthchecks provide a separate health state when configured.

## 5. Editing YAMS environment configuration

YAMS reads environment-specific values from:

```text
/opt/yams/.env
```

Before editing it, create a backup:

```bash
cd /opt/yams
cp .env .env.backup
```

Open it with:

```bash
nano .env
```

In nano:

- `Ctrl+O` saves the file.
- Press Enter to confirm the filename.
- `Ctrl+X` exits.

An important Docker lesson from this project is that changing `.env` does not automatically change the configuration of an already-created container.

When an environment value changes, the affected containers may need to be recreated:

```bash
docker compose up -d --force-recreate <service-name>
```

For example, after changing the Mullvad WireGuard values, the VPN-dependent containers were recreated with:

```bash
docker compose up -d --force-recreate gluetun qbittorrent sabnzbd
```
