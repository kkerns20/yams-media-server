# YAMS Media Server

A self-hosted Linux and Docker project used as a practical Solutions Engineering case study: define requirements, design an architecture, implement it, troubleshoot failures, validate behavior, and document the result.

## Executive summary

The goal of this project is to run a reliable containerized media environment while building hands-on experience with Linux administration, Docker Compose, persistent storage, service dependencies, VPN-isolated networking, monitoring, and incident troubleshooting.

The technical challenge is not simply installing applications. The stack contains multiple dependent services with different networking and storage requirements. Download clients are intentionally routed through a dedicated VPN gateway, media applications depend on consistent filesystem paths, and application health must be validated independently from container runtime state.

## Requirements

- Run multiple services as a maintainable Docker Compose stack.
- Keep application configuration and media persistent across container recreation.
- Route selected download-client traffic through a VPN without routing the entire host through it.
- Prevent VPN-dependent clients from silently falling back to the host connection.
- Make media available to Plex through consistent storage paths.
- Support remote administration and repeatable troubleshooting.
- Keep credentials and machine-specific secrets out of the public repository.

## Architecture

```text
Ubuntu host
└── Docker Engine / Compose
    ├── Gluetun + Mullvad WireGuard
    │   ├── qBittorrent
    │   └── SABnzbd
    ├── Prowlarr
    ├── Sonarr / Radarr / Lidarr / Bazarr
    ├── Plex
    ├── Portainer
    └── Watchtower

Persistent media and application data
└── host storage / bind mounts
```

A key design decision is that qBittorrent and SABnzbd share Gluetun's network namespace. Gluetun therefore acts as both the VPN gateway and a failure boundary: if the tunnel becomes unusable, the dependent clients lose connectivity rather than reverting to the host's normal network path.

See the detailed [architecture notes](docs/architecture.md).

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

## Design decisions and tradeoffs

**Containers over direct host installs:** improves separation and reproducibility, but adds networking, volume, and service-dependency concepts that must be understood when troubleshooting.

**Dedicated VPN gateway container:** limits VPN routing to selected workloads and provides a kill-switch behavior, but creates an explicit dependency on Gluetun health.

**Persistent host storage:** containers can be recreated without intentionally deleting media or application state, but permissions and path mappings must stay consistent between the host and containers.

**Public documentation with private configuration excluded:** makes the project useful as a portfolio and reference while requiring disciplined handling of `.env` files, tokens, private keys, and backups.

## Incident case study: VPN-dependent services lost connectivity

One of the first major failures looked initially like a download-client problem. The Ubuntu host had normal internet access and Docker itself worked, but Gluetun was unhealthy and the clients sharing its network namespace could not reach the internet.

The investigation separated the system into layers:

```text
Host connectivity
    ↓
Docker runtime
    ↓
Gluetun container health
    ↓
WireGuard/VPN connectivity
    ↓
DNS
    ↓
Dependent applications
```

Testing the host and Gluetun independently showed that the failure was isolated to the VPN path. The underlying cause was stale or incorrect WireGuard configuration. After replacing the matching VPN configuration and recreating the affected containers, raw connectivity was restored and the download client was verified to be using a public IP different from the host.

This case reinforced an important operational principle: **a container can be running while the service it represents is not healthy.**

The full investigation and additional incidents are documented in the [troubleshooting log](docs/troubleshooting.md).

## Validation approach

I validate the system at multiple layers rather than relying on a single status screen:

- host network connectivity
- Docker container state
- Docker healthchecks
- connectivity from inside relevant containers
- DNS resolution
- port ownership on the host
- volume and media-path visibility
- VPN egress verification
- application-level playback or import behavior

## Security

This is a public repository. Secrets and machine-specific configuration are intentionally excluded.

Never commit:

- `.env` files
- WireGuard private keys
- VPN configuration files containing credentials
- Portainer setup tokens
- application credentials
- YAMS backups containing configuration or secrets

## Documentation

- [Solutions Engineering case study](docs/solution-case-study.md)
- [Architecture](docs/architecture.md)
- [Installation notes](docs/installation.md)
- [Docker concepts learned through YAMS](docs/docker-basics.md)
- [Troubleshooting log](docs/troubleshooting.md)

## What I would change at larger scale

For a larger or business-critical environment, I would separate concerns further: dedicated storage with a documented backup policy, centralized monitoring and alerting, configuration management, explicit secrets management, infrastructure-as-code where appropriate, tested recovery procedures, and clearer service-level objectives.

The home-lab implementation is intentionally smaller, but the same architecture, dependency, validation, security, and troubleshooting questions apply.

## Related portfolio

This project is part of my broader [Home Lab / Solutions Engineering portfolio](https://github.com/kkerns20/linux-lab).
