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
