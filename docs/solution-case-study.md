# Solution Case Study: Containerized Media Infrastructure

## Scenario

Build and operate a self-hosted media environment on an Ubuntu system while treating the project as an infrastructure design and troubleshooting exercise rather than a one-time application install.

## Problem

A useful media environment requires multiple services to work together reliably. The system needs persistent storage, service-to-service communication, selective VPN routing, predictable paths, secure credential handling, and enough observability to diagnose failures when one dependency breaks.

## Requirements

### Functional

- Run the media-management services in containers.
- Provide persistent paths for application state and media.
- Route qBittorrent and SABnzbd through a VPN gateway.
- Keep other host and container traffic independent of that VPN path.
- Allow Plex to discover and play imported media.
- Support routine administration through Linux and Docker tooling.

### Operational

- Containers should survive reboots and be safely recreatable.
- Failures should be diagnosable from logs, health state, network tests, and host tools.
- The architecture should be documented well enough to rebuild or migrate later.

### Security

- Do not expose VPN private keys, application credentials, tokens, or `.env` files in Git.
- VPN-dependent clients should fail closed rather than use the normal host connection when the tunnel is unavailable.

## Constraints

- Home-lab hardware rather than enterprise infrastructure.
- A single Linux host currently runs the stack.
- Storage and networking are being expanded incrementally.
- The environment needs to remain understandable and maintainable while it is used as a learning platform.

## Architecture decision

Docker Compose was used to coordinate the applications, with Gluetun providing the VPN network namespace for selected download clients.

```text
Internet
   │
Ubuntu host
   │
Docker Engine
   ├── normal container networking
   │     ├── Prowlarr
   │     ├── Sonarr / Radarr / Lidarr / Bazarr
   │     └── management services
   │
   └── Gluetun network namespace
         ├── qBittorrent
         └── SABnzbd
              │
              ▼
        Mullvad / WireGuard
```

Persistent media and application data are kept outside disposable container layers through host-mounted storage.

## Why this design

The design separates three concerns:

1. **Application lifecycle** — Docker makes services easier to recreate and update.
2. **Data lifecycle** — persistent storage survives container replacement.
3. **Network policy** — only selected applications inherit the VPN gateway's network path.

The main tradeoff is additional dependency complexity. If Gluetun fails, every client sharing its network namespace loses connectivity. That behavior is intentional from a security perspective, but it means troubleshooting has to account for the full dependency chain.

## Validation strategy

A service is not considered healthy only because Docker reports it as running. Validation is performed at several layers:

```text
Host
  ↓
Docker daemon
  ↓
Container state
  ↓
Container health
  ↓
Network path / DNS
  ↓
Application behavior
  ↓
End-user result
```

Examples include testing host connectivity, executing network tests inside Gluetun, checking port ownership, confirming volume visibility, verifying VPN egress, and testing media import/playback.

## Incident 1: VPN path failed while the host remained healthy

### Symptom

The download path stopped working even though the Ubuntu host had internet access and Docker was running.

### Discovery

Host-level connectivity worked. Connectivity from inside Gluetun did not. qBittorrent and SABnzbd depended on Gluetun because they shared its network namespace.

### Analysis

This isolated the failure away from the physical network, ISP connection, and general Docker daemon. DNS failures inside Gluetun were treated as a possible symptom rather than immediately assumed to be the root cause, so raw-IP connectivity was tested separately.

### Root cause

The WireGuard configuration stored for the VPN container was stale or incorrect.

### Resolution

The VPN device/configuration was replaced with a valid matching configuration, and the affected containers were recreated so the new environment values were actually applied.

### Verification

- Gluetun passed raw connectivity tests.
- Docker health state recovered.
- VPN-dependent applications regained connectivity.
- qBittorrent's public IP was verified to differ from the host public IP.

### Lesson

Troubleshooting improved dramatically once the system was treated as a dependency graph rather than as a collection of individual applications.

## Incident 2: Plex was running but unavailable

### Symptom

Docker showed Plex as running, but the application was unavailable.

### Analysis

Application logs indicated a port-binding conflict. Host-level inspection showed a separate Snap installation of Plex already using the same port. Because the container used host networking, the two instances competed directly for the host port.

### Resolution

The legacy Snap service was stopped, disabled, and later removed after the Docker instance was verified. Firewall access was also validated separately.

### Lesson

Container troubleshooting sometimes requires leaving Docker and inspecting the host itself. Process ownership, ports, firewalls, mounts, and service managers remain part of the system.

## Incident 3: Manual workflow succeeded while automation did not

A manual media-import path was proven end to end before continuing to debug automated handoffs. This established a known-good baseline and narrowed the remaining problem to orchestration between applications rather than storage or playback.

This is the troubleshooting pattern I want to carry into future projects:

**prove components independently → establish a working baseline → test interfaces between components → automate only after the underlying path is understood.**

## Business / customer explanation

The value of this design is not the specific media applications. The project demonstrates a reusable approach to technical solutions:

- identify requirements and dependencies;
- isolate workloads with different security requirements;
- preserve data independently from application runtime;
- validate at multiple layers;
- troubleshoot by narrowing the failure domain;
- document design decisions and recovery steps.

Those same principles apply to customer-facing SaaS integrations, cloud workloads, internal applications, and infrastructure deployments.

## Risks and next improvements

Current risks include dependence on a single host, limited storage redundancy, and monitoring that is still evolving. Planned improvements include better service monitoring, backup/recovery documentation, UPS integration, storage expansion, and eventual infrastructure automation.

## Demo outline

A short portfolio demo of this project can follow this sequence:

1. State the problem and requirements.
2. Show the architecture and explain the Gluetun dependency.
3. Explain persistence and why container recreation is safe for application data.
4. Walk through one incident from symptom to root cause.
5. Show how the fix was validated.
6. Close with tradeoffs, risks, and what would change at larger scale.
