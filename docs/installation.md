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

## 4. Starting and inspecting the stack

## 5. Editing YAMS environment configuration
