# Plex + *arr homelab stack

Docker Compose stack for Plex, MediaSage, and a *arr acquisition suite behind [Gluetun](https://github.com/qdm12/gluetun) (Privado OpenVPN).

## Architecture

- **Plex** and **MediaSage** use `network_mode: host` so they can reach each other on `localhost:32400`.
- **bazarr**, **lidarr**, **prowlarr**, **radarr**, **sonarr**, **sabnzbd**, and **configarr** use `network_mode: service:gluetun`, so all traffic goes through the VPN. Their UIs are published on Gluetun’s ports.
- Persistent config and media live under `PLEX_ROOT` on the host (not in this repo).

```
PLEX_ROOT/
  appdata/   # per-service config
  data/      # downloads + media library
```

## Prerequisites

- Docker with Compose v2
- Host directories for `PLEX_ROOT` (see `.env.example`)

## First run

```bash
cp .env.example .env
cp .secrets/gluetun.env.example .secrets/gluetun.env
cp .secrets/mediasage.env.example .secrets/mediasage.env
# Edit the .env and .secrets files, then:
chmod 600 .secrets/gluetun.env .secrets/mediasage.env
docker compose up -d
```

### Secrets

| File | Keys |
|------|------|
| `.secrets/gluetun.env` | `OPENVPN_USER`, `OPENVPN_PASSWORD` |
| `.secrets/mediasage.env` | `PLEX_TOKEN`, `GEMINI_API_KEY` |

Keep real secret files mode `600`. Example files are tracked; real `.env` / `.secrets/*.env` are gitignored.

### UID / GID

LinuxServer images use `PUID` / `PGID` (defaults `1000`). MediaSage uses `user: "1000:1000"` instead — that image expects a container user, not the LinuxServer env vars.

## Ports / UIs

| Service | URL |
|---------|-----|
| Plex | `http://<host>:32400/web` |
| MediaSage | `http://<host>:5765` (image default; host networking) |
| Bazarr | `http://<host>:6767` |
| Lidarr | `http://<host>:8686` |
| Prowlarr | `http://<host>:9696` |
| Radarr | `http://<host>:7878` |
| Sonarr | `http://<host>:8989` |
| SABnzbd | `http://<host>:8080` |
| Configarr | `http://<host>:8282` |

## Updating images

Image tags are pinned in `compose.yaml`. To bump:

1. Change the version tag on the service(s) in `compose.yaml`.
2. `docker compose pull`
3. `docker compose up -d`

## Validation

```bash
docker compose config -q
yamllint compose.yaml
```
