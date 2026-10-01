# YAMS Architecture

## Mental model

YAMS is not one application. It is a collection of services coordinated with Docker Compose.

```text
Ubuntu host
└── Docker Engine
    └── Docker Compose / YAMS
        ├── Gluetun (VPN network namespace)
        │   ├── qBittorrent
        │   └── SABnzbd
        ├── Sonarr
        ├── Radarr
        ├── Lidarr
        ├── Bazarr
        ├── Prowlarr
        ├── Plex
        ├── Portainer
        └── Watchtower
```

## Host vs container networking

The Ubuntu host has its own normal internet connection.

Gluetun creates a separate VPN path inside Docker using Mullvad WireGuard. qBittorrent and SABnzbd are configured to use Gluetun's network namespace instead of having independent network stacks.

That design means:

- the host can have working internet while Gluetun is broken;
- qBittorrent and SABnzbd can be blocked while other containers continue running;
- Gluetun acts as the VPN gateway and kill switch for those download clients.

## Why ports can appear under Gluetun

When a container uses another container's network namespace, its own ports are effectively exposed through the shared network container. That is why qBittorrent and SABnzbd may not show their own published ports in `docker compose ps`; Gluetun owns the shared network stack.

## Persistence

Containers are disposable runtime instances. Persistent application state should live outside the container through bind mounts or volumes. This lets containers be recreated or updated without intentionally deleting application configuration or media.

## Health vs running state

`docker ps` answers whether a container process is running.

A Docker healthcheck answers whether the service is actually functioning according to its configured test.

For this stack, Gluetun was once `Up` but `unhealthy`, which correctly indicated that the container process existed while the VPN tunnel could not pass usable traffic.
