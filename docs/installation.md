# Installation Notes

These notes document the installation model I am learning through this project. They are not a replacement for the current YAMS documentation.

## Host

- Ubuntu 24.04 LTS
- Docker installed at `/usr/bin/docker`
- Docker Compose available through `docker compose`
- YAMS installed under `/opt/yams`
- Media storage under `/srv/media`

## Verify Docker first

Before troubleshooting YAMS, verify Docker independently:

```bash
docker --version
docker compose version
docker run hello-world
```

If `hello-world` succeeds without `sudo`, both the Docker daemon and current-user permissions are working.

## YAMS files

The current installation uses:

```text
/opt/yams/
├── .env
├── docker-compose.yaml
├── docker-compose.custom.yaml
├── config/
└── yams
```

The `.env` file stores environment-specific values used by Docker Compose. It can contain secrets and must not be committed to GitHub.

## Starting the stack

The YAMS helper exposes commands such as:

```bash
yams start
yams stop
yams restart
yams status
yams check-vpn
```

Underneath, Docker Compose manages the multi-container application.

Useful Docker commands:

```bash
docker ps
docker ps -a
docker compose ps -a
docker compose config --services
```

## Editing environment configuration

Back up the environment file before changing it:

```bash
cd /opt/yams
cp .env .env.backup
nano .env
```

In nano:

- `Ctrl+O` saves
- press Enter to confirm the filename
- `Ctrl+X` exits

If a Compose environment value changes, an existing container may need to be recreated for the new value to be applied.
