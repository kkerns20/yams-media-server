# Docker Basics Learned Through YAMS

## Image vs container

An **image** is the packaged blueprint for an application.

A **container** is a running or stopped instance created from that image.

Updating or recreating a container does not necessarily erase application data when that data is persisted through volumes or bind mounts.

## Docker daemon

The Docker daemon is the background service responsible for running containers.

Check it with:

```bash
systemctl status docker
```

## Docker Compose

Docker Compose describes a group of related containers as one application stack.

The Compose file defines things such as:

- images
- environment variables
- volumes
- ports
- dependencies
- networking
- restart policies

YAMS provides a helper command around this Compose-managed stack.

## Useful commands

Running containers:

```bash
docker ps
```

All containers, including stopped ones:

```bash
docker ps -a
```

YAMS Compose status:

```bash
cd /opt/yams
docker compose ps -a
```

Recent logs:

```bash
docker logs --tail 50 gluetun
```

Run a command inside a container:

```bash
docker exec gluetun ping -c 3 1.1.1.1
```

## Restart vs recreate

This distinction mattered during VPN troubleshooting.

A restart starts the same existing container again with its existing container configuration.

A recreate builds a new container instance from the current Compose configuration and environment values.

Example:

```bash
docker compose up -d --force-recreate gluetun qbittorrent sabnzbd
```

That command was necessary after changing the Mullvad WireGuard values in `.env`.

## Shared network namespace

qBittorrent and SABnzbd use Gluetun's network namespace.

Conceptually:

```text
qBittorrent ─┐
             ├──> Gluetun ──> Mullvad VPN ──> Internet
SABnzbd ─────┘
```

This is intentional. If the VPN path fails, the download clients lose internet access rather than silently falling back to the host's normal connection.
