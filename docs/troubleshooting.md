# Troubleshooting Log

## Case 1: Gluetun unhealthy and download clients had no internet

### Symptoms

The YAMS stack started, but Gluetun repeatedly became unhealthy.

Observed behavior included:

- Ubuntu host internet worked normally.
- `docker run hello-world` succeeded.
- Gluetun could not ping `1.1.1.1`.
- DNS lookups from inside Gluetun timed out.
- `yams check-vpn` could not retrieve qBittorrent's public IP.
- Gluetun rotated through multiple Mullvad endpoints.
- The WireGuard interface existed and transmitted packets but received no usable responses.
- qBittorrent and SABnzbd were configured to share Gluetun's network namespace.

### Narrowing the problem

Host connectivity was verified independently:

```bash
ping -c 3 1.1.1.1
ping -c 3 github.com
```

Then connectivity was tested from inside Gluetun:

```bash
docker exec gluetun ping -c 3 1.1.1.1
```

The host succeeded while Gluetun failed. This showed that the problem was not the Ubuntu host's general internet connection.

Docker policy routing and the WireGuard interface were also present, which reduced the likelihood that the problem was simply missing container routing.

### Root cause

The Mullvad WireGuard private key/address configuration stored in `/opt/yams/.env` was incorrect or stale.

### Fix

A fresh Mullvad WireGuard device/configuration was created. The matching private key and IPv4 tunnel address were then placed into:

```text
WIREGUARD_PRIVATE_KEY=...
WIREGUARD_ADDRESSES=10.x.x.x/32
```

The actual private key is intentionally not stored in this repository.

Because changing `.env` does not modify the configuration of an already-created container, the VPN-dependent containers were recreated:

```bash
cd /opt/yams
docker compose up -d --force-recreate gluetun qbittorrent sabnzbd
```

### Verification

First, raw connectivity through Gluetun:

```bash
docker exec gluetun ping -c 3 1.1.1.1
```

Result: packet replies were received with 0% loss.

Then:

```bash
yams check-vpn
```

Result: YAMS reported that qBittorrent's public IP differed from the host public IP, confirming that qBittorrent was routed through the VPN.

### Lessons learned

1. `Up` and `healthy` are not the same thing.
2. Test host networking and container networking separately.
3. A DNS timeout may be a symptom of a deeper tunnel failure; testing a raw IP helps distinguish the two.
4. qBittorrent and SABnzbd depend on Gluetun because they share its network namespace.
5. Changes to Compose environment values often require container recreation, not merely a restart.
6. VPN private keys, tokens, and `.env` files should never be committed to a public repository.

---

## Case 2: Plex container was running but the Plex server was unavailable

### Symptoms

Docker reported the Plex container as running, but the Plex server was unavailable from the web interface.

The container log repeatedly showed Plex starting, while the Plex application log contained:

```text
ERROR - HttpServer: Error binding acceptor: Address in use
DEBUG - Exiting due to bind address already in use
```

The media mount itself was healthy. Plex could see files under `/data/tvshows`.

### Investigation

The port owner was checked with:

```bash
sudo ss -ltnp | grep :32400
```

The process using port 32400 was then identified:

```bash
ps -fp <PID>
sudo readlink -f /proc/<PID>/exe
sudo cat /proc/<PID>/cgroup
```

The executable path showed a separate Snap installation of Plex:

```text
/snap/plexmediaserver/...
```

The YAMS Plex container uses host networking, so both Plex installations were competing for the same host port.

### Fix

The Snap Plex service was first stopped and disabled:

```bash
sudo snap stop --disable plexmediaserver
```

Once port 32400 was released, the Docker/YAMS Plex process successfully bound to it.

After verifying that the Docker Plex instance worked, the old Snap installation was removed:

```bash
sudo snap remove plexmediaserver
```

UFW was also active, so Plex's port was allowed:

```bash
sudo ufw allow 32400/tcp
```

### Verification

```bash
snap list | grep plex
sudo ss -ltnp | grep :32400
docker top plex
```

The final `docker top plex` output showed the actual `Plex Media Server` process and Plex successfully read and transcoded media from `/data/tvshows`.

### Lessons learned

1. A running container can contain a failed application process while its supervisor keeps the container alive.
2. Port conflicts can happen outside Docker, so host-level tools such as `ss`, `ps`, `/proc`, and system service managers are important.
3. Host networking means the container shares the host's network namespace and therefore competes for host ports directly.
4. Verify mounts, application health, port ownership, and firewall rules separately.

---

## Case 3: Manual media workflow works; Sonarr/Radarr automation still needs triage

### Confirmed working path

A media file downloaded successfully through qBittorrent and was visible under:

```text
/data/downloads/torrents
```

Sonarr's Manual Import feature was then used with **Hardlink** mode to import the episode into:

```text
/data/tvshows
```

The host path was verified under `/srv/media/tvshows`, and Plex successfully discovered and played the imported file.

The current known-good workflow is:

```text
qBittorrent
  -> /data/downloads/torrents
  -> Sonarr/Radarr manual import
  -> hardlink into /data/tvshows or /data/movies
  -> Plex scan
  -> playback
```

Hardlinks are preferred over moving completed downloads because the download client can continue seeding while the library receives its own directory entry without duplicating the file's data blocks.

### Outstanding problem

The automatic Sonarr/Radarr handoff is not yet reliable.

The next troubleshooting pass should verify, in order:

1. Prowlarr can find a release.
2. Sonarr/Radarr can grab the release.
3. The correct qBittorrent category is assigned.
4. qBittorrent can complete the download through Gluetun.
5. Completed Download Handling is enabled.
6. Sonarr/Radarr recognizes the completed download and imports it.
7. Plex scans the resulting library path.

Useful places to inspect include Sonarr/Radarr **Activity -> Queue**, **History**, application logs, qBittorrent categories, and Completed Download Handling settings.

### Lesson learned

When debugging an automated pipeline, first prove each component independently and preserve a known-good manual workflow. That provides a baseline while automation is investigated.
