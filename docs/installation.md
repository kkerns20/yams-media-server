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

## 6. Mullvad and Gluetun VPN setup

Gluetun provides the VPN network layer for qBittorrent and SABnzbd.

In this setup, the Ubuntu host uses its normal internet connection, while the download clients share Gluetun's network namespace and route their traffic through Mullvad using WireGuard.

Conceptually:

```text
Ubuntu host
   |
   +--> normal host internet
   |
   +--> Docker
         |
         +--> Gluetun --> Mullvad WireGuard --> Internet
                |
                +--> qBittorrent
                +--> SABnzbd
```

This design is useful because if the VPN tunnel fails, qBittorrent and SABnzbd lose connectivity instead of silently falling back to the host connection.

### 6.1 Create or refresh the Mullvad WireGuard configuration

Use Mullvad's WireGuard configuration generator to create a configuration for this Linux/YAMS setup.

The important values are:

```text
PrivateKey
IPv4 Address ending in /32
```

These two values must belong to the same Mullvad WireGuard configuration.

Do not commit the private key to GitHub.

### 6.2 Update the YAMS environment file

Move to the YAMS installation directory:

```bash
cd /opt/yams
```

Create a backup before editing:

```bash
cp .env .env.backup
```

Open the environment file:

```bash
nano .env
```

Update the WireGuard values:

```text
WIREGUARD_PRIVATE_KEY=<private-key>
WIREGUARD_ADDRESSES=10.x.x.x/32
```

Save with `Ctrl+O`, press Enter, then exit with `Ctrl+X`.

The real private key should never appear in this repository.

### 6.3 Verify the environment values safely

The address can be displayed, but the private key should be redacted:

```bash
grep -E 'WIREGUARD_PRIVATE_KEY|WIREGUARD_ADDRESSES' .env | sed -E 's/(PRIVATE_KEY=).*/\1[REDACTED]/'
```

Expected output should look similar to:

```text
WIREGUARD_PRIVATE_KEY=[REDACTED]
WIREGUARD_ADDRESSES=10.x.x.x/32
```

### 6.4 Recreate the VPN-dependent containers

Changing `.env` does not automatically modify an existing container.

Recreate Gluetun and the services that share its network namespace:

```bash
docker compose up -d --force-recreate gluetun qbittorrent sabnzbd
```

This forces Docker Compose to create new container instances using the updated environment values.

### 6.5 Verify Gluetun connectivity

Test raw IP connectivity from inside Gluetun:

```bash
docker exec gluetun ping -c 3 1.1.1.1
```

A successful result confirms that the VPN container can pass traffic.

Then check container health:

```bash
docker compose ps
```

Gluetun should eventually report a healthy state.

### 6.6 Verify qBittorrent is using the VPN

Run:

```bash
yams check-vpn
```

The important result is that qBittorrent's public IP differs from the Ubuntu host's public IP.

That confirms the download client is routed through Mullvad rather than the host connection.

### 6.7 Troubleshooting lesson

During this build, the host network and Docker daemon were both healthy while Gluetun could not pass traffic.

The main symptoms were:

- Gluetun was running but unhealthy.
- Raw IP traffic from inside Gluetun failed.
- DNS lookups inside Gluetun timed out.
- qBittorrent could not report a public IP.
- qBittorrent and SABnzbd were correctly attached to Gluetun's network namespace.

The root cause was an incorrect or stale Mullvad WireGuard private key/address pair in `.env`.

After replacing the matching WireGuard credentials and recreating Gluetun, qBittorrent, and SABnzbd, VPN traffic worked normally.

This reinforced several Docker troubleshooting principles:

- test host networking and container networking separately;
- do not assume `Up` means healthy;
- test raw IP connectivity before blaming DNS;
- understand which containers depend on another container's network namespace;
- recreate containers after changing environment values that are injected at container creation time.
