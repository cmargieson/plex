# Plex + *arr homelab stack

Docker Compose stack for Plex, MediaSage, and a *arr acquisition suite behind [Gluetun](https://github.com/qdm12/gluetun) (Privado OpenVPN).

## Architecture

- **Plex** and **MediaSage** use `network_mode: host` so they can reach each other on `localhost:32400`.
- **bazarr**, **lidarr**, **prowlarr**, **radarr**, **sonarr**, **sabnzbd**, **configarr**, **slskd**, and **soularr** use `network_mode: service:gluetun`, so all traffic goes through the VPN. Their UIs are published on Gluetun’s ports.
- **slskd** is the Soulseek client (music downloads + sharing `/data/media/music` for uploads).
- **Soularr** polls Lidarr for wanted albums, downloads them via slskd, then triggers Lidarr import.
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
cp .secrets/slskd.env.example .secrets/slskd.env
# Edit the .env and .secrets files, then:
chmod 600 .secrets/gluetun.env .secrets/mediasage.env .secrets/slskd.env
docker compose up -d
```

### Secrets

| File | Keys |
|------|------|
| `.secrets/gluetun.env` | `OPENVPN_USER`, `OPENVPN_PASSWORD` |
| `.secrets/mediasage.env` | `PLEX_TOKEN`, `GEMINI_API_KEY` |
| `.secrets/slskd.env` | `SLSKD_SLSK_USERNAME`, `SLSKD_SLSK_PASSWORD` |

Keep real secret files mode `600`. Example files are tracked; real `.env` / `.secrets/*.env` are gitignored.

### UID / GID

LinuxServer images use `PUID` / `PGID` (defaults `1000`). MediaSage and Soularr use `user: "1000:1000"`. slskd uses `PUID`/`PGID` via `*common-env` so its entrypoint can fix ownership of a fresh `${PLEX_ROOT}/appdata/slskd` mount.

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
| slskd | `http://<host>:5030` |

Soulseek also listens for P2P connections on port `50300` (published via Gluetun). Privado typically does not forward that port, so direct peer connectivity may be limited; downloads and sharing still work via Soulseek relays.

## Soulseek setup (after first `compose up`)

1. Create a Soulseek account if needed ([slsknet.org](https://www.slsknet.org/news/)) and put credentials in `.secrets/slskd.env`, then `docker compose up -d slskd`.
2. Open slskd at `http://<host>:5030` (default login `slskd` / `slskd`) and change the web UI password. Confirm the share scan of `/data/media/music` completes.
3. Create an API key in slskd (Settings → Options → API Keys).
4. Copy the Soularr config example and fill in Lidarr + slskd API keys:

   ```bash
   mkdir -p "${PLEX_ROOT}/appdata/soularr"
   cp appdata/soularr/config.ini.example "${PLEX_ROOT}/appdata/soularr/config.ini"
   # Edit config.ini, then:
   docker compose restart soularr
   ```

5. Check `docker logs soularr --tail 50` for successful Lidarr/slskd connections. Add a wanted album in Lidarr to test the loop.

Soularr and Lidarr talk over `localhost` because they share Gluetun’s network. Completed downloads land under `data/downloads/complete/slskd`; Lidarr imports into `data/media/music`.

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
