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

## 7. Verify the YAMS services

Starting the stack is only the first check. A container can be running while the application inside it is not actually usable.

Verification should happen at three levels:

1. Is the container running?
2. Is the container healthy, when a healthcheck exists?
3. Can the application itself be reached and used?

### 7.1 Check the whole stack

From the YAMS directory:

```bash
cd /opt/yams
docker compose ps -a
```

This shows the status of every service in the Compose project.

Look for:

- `Up` — the container process is running.
- `healthy` — the configured healthcheck is succeeding.
- `unhealthy` — the process is running, but the healthcheck is failing.
- `Exited` — the container is stopped.

A useful follow-up is:

```bash
docker ps
```

This provides a quick view of currently running containers and their published ports.

### 7.2 Check recent logs

If a service is not behaving as expected, inspect its recent logs:

```bash
docker logs --tail 50 <container-name>
```

Examples:

```bash
docker logs --tail 50 gluetun
docker logs --tail 50 plex
docker logs --tail 50 sonarr
```

For live troubleshooting, follow the logs:

```bash
docker logs -f <container-name>
```

Press `Ctrl+C` to stop following the log output.

### 7.3 Verify the web interfaces

Most YAMS applications expose a local web interface.

For this installation, the commonly used ports include:

| Service | Port | Purpose |
| --- | ---: | --- |
| Portainer | 9000 | Docker management |
| qBittorrent | 8080 | Torrent client |
| SABnzbd | 8081 | Usenet download client |
| Sonarr | 8989 | TV automation |
| Radarr | 7878 | Movie automation |
| Lidarr | 8686 | Music automation |
| Bazarr | 6767 | Subtitle automation |
| Prowlarr | 9696 | Indexer management |

In this setup, qBittorrent and SABnzbd share Gluetun's network namespace, so their web ports are published through Gluetun rather than directly by those containers.

The exact published ports should always be confirmed from the running Compose stack:

```bash
docker compose ps
```

A web interface can normally be reached using the server's LAN address:

```text
http://<server-ip>:<port>
```

For example:

```text
http://<server-ip>:8989
```

Do not expose these management interfaces directly to the public internet unless they are intentionally secured for remote access.

### 7.4 Verify Plex

Plex behaves slightly differently from the other services and may not show a normal published port in `docker compose ps`, depending on how networking is configured.

Check the container:

```bash
docker ps | grep plex
```

Check recent logs:

```bash
docker logs --tail 50 plex
```

A useful sign of a working Plex server is that its internal service is listening on port `32400`.

The Plex web interface is typically accessed on the local network using:

```text
http://<server-ip>:32400/web
```

### 7.5 Verify the VPN-dependent download clients

qBittorrent and SABnzbd share Gluetun's network namespace.

Because of that relationship, a running qBittorrent or SABnzbd container does not prove that it has internet access.

First verify Gluetun:

```bash
docker exec gluetun ping -c 3 1.1.1.1
```

Then verify the VPN routing:

```bash
yams check-vpn
```

The qBittorrent public IP should differ from the Ubuntu host public IP.

This confirms that qBittorrent is using the VPN path instead of the host's normal connection.

### 7.6 Check container health directly

To inspect the health state of a specific container:

```bash
docker inspect <container-name> --format='{{json .State.Health}}'
```

For a simpler status:

```bash
docker inspect <container-name> --format='{{.State.Status}} {{if .State.Health}}{{.State.Health.Status}}{{end}}'
```

Example:

```bash
docker inspect gluetun --format='{{.State.Status}} {{if .State.Health}}{{.State.Health.Status}}{{end}}'
```

This helps distinguish:

```text
running healthy
```

from:

```text
running unhealthy
```

### 7.7 Verification checklist

After starting or changing the stack, verify:

- Docker Compose shows the expected containers running.
- Gluetun becomes healthy.
- Gluetun can reach a raw internet IP.
- `yams check-vpn` confirms qBittorrent is using a different public IP.
- Sonarr, Radarr, Prowlarr, and the other web interfaces load locally.
- Plex responds on its local web interface.
- Application logs do not show repeated startup failures.
- No service is unexpectedly restarting or exiting.

The main lesson is that service verification should move from the outside in:

```text
Docker daemon
    ↓
Container running
    ↓
Container health
    ↓
Network connectivity
    ↓
Application web interface
    ↓
Application function
```

That makes troubleshooting much faster because each layer can be tested independently.

## 8. Persistence, restart behavior, and backups

Containers should be treated as replaceable runtime instances. Important application data should live outside the container through bind mounts or Docker volumes.

This matters because containers may be:

- restarted;
- recreated after configuration changes;
- replaced during image updates;
- removed and rebuilt during troubleshooting.

Persistent configuration should survive those operations as long as the mapped storage is preserved.

### 8.1 Check restart policies

A container restart policy controls what Docker does after a process exits or the Docker daemon restarts.

Check a container's restart policy with:

```bash
docker inspect <container-name> --format='{{.HostConfig.RestartPolicy.Name}}'
```

Example:

```bash
docker inspect plex --format='{{.HostConfig.RestartPolicy.Name}}'
```

A common policy in this stack is:

```text
unless-stopped
```

This generally means Docker will restart the container automatically unless it was intentionally stopped.

### 8.2 Understand what survives a container recreation

Recreating a container does not necessarily mean losing its settings.

The important distinction is:

```text
Container
    ↓ replaceable
Persistent configuration / media
    ↓ should remain outside the container
```

To inspect mounts:

```bash
docker inspect <container-name> --format='{{json .Mounts}}'
```

For easier reading, Docker Compose can also help show how storage is mapped:

```bash
docker compose config
```

Never assume data is persistent without checking the configured mounts.

### 8.3 YAMS backups

YAMS provides a backup helper:

```bash
yams backup
```

Backups should be treated as sensitive because they may contain application configuration or credentials.

Do not commit YAMS backup archives to a public GitHub repository.

A backup is only useful if it can eventually be restored, so restoration should be tested before relying on backups as the only recovery plan.

---

## 9. Storage layout and permissions

Media-server problems are often storage or permission problems rather than application problems.

For this build, media storage is located under:

```text
/srv/media
```

### 9.1 Inspect the storage location

Check the directory:

```bash
ls -ld /srv/media
```

Check available space:

```bash
df -h /srv/media
```

If needed, inspect the directory tree:

```bash
find /srv/media -maxdepth 2 -type d
```

### 9.2 Ownership and permissions

Linux permissions determine whether applications can read, write, rename, move, or delete files.

Useful checks:

```bash
ls -ld /srv/media
ls -l /srv/media
```

To see the current user and group IDs:

```bash
id
```

Many LinuxServer.io containers use a user ID and group ID supplied through environment variables such as `PUID` and `PGID`.

The host permissions and container user IDs should be compatible with the storage directories used by the applications.

### 9.3 Why consistent paths matter

Applications such as Sonarr, Radarr, qBittorrent, SABnzbd, and Plex may need to refer to the same files.

Consistent path mappings make that easier.

Conceptually:

```text
Host filesystem
/srv/media
    |
    +--> download client
    +--> Sonarr / Radarr
    +--> Plex
```

If different containers see the same host directory under inconsistent internal paths, imports and moves can become harder to troubleshoot.

---

## 10. Service relationships

The YAMS stack works because the services cooperate rather than operate independently.

A simplified relationship looks like:

```text
Prowlarr
   |
   +--> Sonarr ----\
   +--> Radarr -----+--> qBittorrent / SABnzbd --> download
   +--> Lidarr ----/              |
                                  v
                              /srv/media
                                  |
                                  v
                                Plex
```

Bazarr works alongside the media-management applications to manage subtitles.

Portainer provides a graphical interface for inspecting Docker.

Watchtower is used to monitor or manage container image updates depending on its configuration.

### 10.1 Verify containers can resolve each other

Docker Compose normally provides service-name DNS within the Compose network.

A useful test from one container is:

```bash
docker exec <container-name> getent hosts <service-name>
```

Example:

```bash
docker exec sonarr getent hosts prowlarr
```

This can help distinguish an application configuration problem from a Docker DNS or networking problem.

### 10.2 Use service names when appropriate

Inside a Docker Compose network, applications can often communicate using Compose service names rather than the host LAN IP.

For example, one service may be reachable internally using a hostname such as:

```text
prowlarr
radarr
sonarr
```

The exact application URLs and ports should be taken from the active Compose configuration.

---

## 11. Updating the stack

Updates should be approached as a controlled change rather than a blind restart.

Before updating:

1. Confirm the stack is healthy.
2. Make a backup when appropriate.
3. Check available disk space.
4. Record any important custom configuration.
5. Confirm that secrets are stored outside the Git repository.

Useful status commands:

```bash
cd /opt/yams
docker compose ps -a
yams status
```

YAMS includes an update-related helper:

```bash
yams update-containers
```

After an update, repeat the verification process from Section 7:

```bash
docker compose ps -a
docker logs --tail 50 <container-name>
docker exec gluetun ping -c 3 1.1.1.1
yams check-vpn
```

The goal is to verify the system after the change instead of assuming that a successful update command means every service is working correctly.

---

## 12. Security and secret management

This repository is public, so credentials and machine-specific secrets must remain outside Git.

Never commit:

```text
.env
WireGuard private keys
VPN configuration files containing credentials
Portainer setup tokens
application passwords or API keys
backup archives containing configuration
private certificates or private keys
```

The repository `.gitignore` is intended to reduce accidental commits of sensitive files, but `.gitignore` is not a substitute for checking changes before committing.

Always review:

```bash
git status
git diff
```

before:

```bash
git add
git commit
git push
```

### 12.1 Check staged files before committing

After `git add`, review what is staged:

```bash
git diff --cached
```

This is especially important in infrastructure repositories where configuration files may contain secrets.

### 12.2 If a secret is accidentally committed

Removing a secret from the latest file is not enough if it already exists in Git history.

The credential should be treated as exposed and rotated or replaced.

The Git history may also need to be rewritten before the repository is considered clean.

---

## 13. Troubleshooting workflow

When something breaks, troubleshoot one layer at a time instead of changing several things at once.

A useful order is:

```text
1. Host
2. Docker daemon
3. Compose stack
4. Container state
5. Container health
6. Network
7. Application
8. Service-to-service integration
9. Storage and permissions
```

### 13.1 Host checks

```bash
cat /etc/os-release
df -h
ip addr
ping -c 3 1.1.1.1
getent hosts github.com
```

### 13.2 Docker checks

```bash
systemctl status docker
docker version
docker compose version
docker ps
docker ps -a
```

### 13.3 Compose checks

```bash
cd /opt/yams
docker compose ps -a
docker compose config --services
```

### 13.4 Container logs

```bash
docker logs --tail 100 <container-name>
```

Follow live output:

```bash
docker logs -f <container-name>
```

### 13.5 Health state

```bash
docker inspect <container-name> --format='{{.State.Status}} {{if .State.Health}}{{.State.Health.Status}}{{end}}'
```

### 13.6 Networking from inside a container

Raw IP test:

```bash
docker exec <container-name> ping -c 3 1.1.1.1
```

DNS test:

```bash
docker exec <container-name> getent hosts github.com
```

If raw IP traffic works but DNS resolution fails, investigate DNS.

If raw IP traffic also fails, the problem is lower in the network path.

### 13.7 Inspect container networking

```bash
docker inspect <container-name> --format='{{.HostConfig.NetworkMode}}'
```

For qBittorrent and SABnzbd, this is especially useful because they share Gluetun's network namespace.

### 13.8 Inspect routes when needed

For deeper networking problems:

```bash
docker exec gluetun ip addr
docker exec gluetun ip rule
docker exec gluetun ip route show table all
```

These commands help verify that the WireGuard interface and policy routing exist.

---

## 14. Routine maintenance checklist

A simple periodic check can catch problems before they become confusing.

### Quick health check

```bash
cd /opt/yams
docker compose ps -a
yams check-vpn
df -h
```

Review any container that is:

```text
unhealthy
Exited
Restarting
```

### After configuration changes

Verify:

- the expected containers were recreated if required;
- Gluetun is healthy;
- qBittorrent remains behind the VPN;
- application web interfaces still load;
- storage paths are accessible;
- no unexpected errors repeat in the logs.

### After a system reboot

Verify:

```bash
systemctl status docker
docker compose ps -a
yams check-vpn
```

Confirm that services using restart policies returned as expected.

---

## 15. Recovery mindset

The goal of this project is not to create a server that never fails.

The goal is to build a server that is understandable and recoverable.

A healthy recovery strategy includes:

- known installation steps;
- documented configuration locations;
- backups;
- secrets stored outside Git;
- repeatable verification commands;
- understanding which services depend on one another;
- troubleshooting one layer at a time.

This repository is intended to become that recovery reference as the server evolves.
