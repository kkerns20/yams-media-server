# YAMS Media Server

A hands-on Linux and Docker project documenting my self-hosted media server built with [YAMS](https://yams.media/).

This repository is a learning project as much as a server build. It documents how the stack fits together, how Docker Compose manages the services, how VPN routing protects download clients, and how I troubleshoot failures without exposing credentials.

## Current stack

- Ubuntu 24.04 LTS
- Docker Engine + Docker Compose
- YAMS
- Gluetun + Mullvad WireGuard
- qBittorrent
- SABnzbd
- Sonarr
- Radarr
- Lidarr
- Bazarr
- Prowlarr
- Plex
- Portainer
- Watchtower

## Documentation

- [Architecture](docs/architecture.md)
- [Installation notes](docs/installation.md)
- [Docker basics learned through YAMS](docs/docker-basics.md)
- [Troubleshooting log](docs/troubleshooting.md)

## Key lesson so far

A container being `Up` does not mean the application inside it is healthy.

The first major troubleshooting case in this project involved Gluetun starting successfully while its WireGuard tunnel could not pass traffic. The host network and Docker daemon were healthy, but qBittorrent and SABnzbd could not reach the internet because both share Gluetun's network namespace. The root cause was an incorrect/stale Mullvad WireGuard credential pair in YAMS' `.env` file.

After replacing the matching WireGuard private key/address pair and recreating the VPN-dependent containers, Gluetun passed traffic and YAMS confirmed that qBittorrent was using a different public IP from the host.

## Security

This is a public repository. Secrets and machine-specific configuration are intentionally excluded.

Never commit:

- `.env` files
- WireGuard private keys
- VPN configuration files containing credentials
- Portainer setup tokens
- application credentials
- YAMS backups containing configuration/secrets

## Related project

This project is part of my broader [Linux Lab](https://github.com/kkerns20/linux-lab).
