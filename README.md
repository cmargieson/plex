# Plex + *arr stack

Plex, MediaSage, and a *arr acquisition suite. Download traffic goes through [Gluetun](https://github.com/qdm12/gluetun) (Privado OpenVPN).

## Layout

| Network | Services |
|---------|----------|
| Host | Plex, MediaSage |
| Gluetun (VPN) | bazarr, lidarr, prowlarr, radarr, sonarr, sabnzbd, configarr, slskd, soularr |

Plex and MediaSage share `localhost` so MediaSage can reach Plex at `:32400`. VPN services share Gluetun’s network; their UIs are published on Gluetun’s ports.

Config and media live under `PLEX_ROOT` on the host:

```
PLEX_ROOT/
  appdata/   # per-service config
  data/      # downloads + media library
```

## Setup

```bash
cp .env.example .env
cp .secrets/gluetun.env.example .secrets/gluetun.env
cp .secrets/mediasage.env.example .secrets/mediasage.env
cp .secrets/slskd.env.example .secrets/slskd.env
# Edit those files, then:
chmod 600 .secrets/*.env
docker compose up -d
```

Requires Docker Compose v2 and a `PLEX_ROOT` directory (see `.env.example`).

### Secrets

| File | Keys |
|------|------|
| `.secrets/gluetun.env` | `OPENVPN_USER`, `OPENVPN_PASSWORD` |
| `.secrets/mediasage.env` | `PLEX_TOKEN`, `GEMINI_API_KEY` |
| `.secrets/slskd.env` | `SLSKD_SLSK_USERNAME`, `SLSKD_SLSK_PASSWORD` |

Example files are tracked; real `.env` and `.secrets/*.env` are gitignored. Keep secrets mode `600`.

LinuxServer images use `PUID`/`PGID` (default `1000`). MediaSage and Soularr run as `user: "1000:1000"`.

## UIs

| Service | URL |
|---------|-----|
| Plex | `http://<host>:32400/web` |
| MediaSage | `http://<host>:5765` |
| Bazarr | `http://<host>:6767` |
| Lidarr | `http://<host>:8686` |
| Prowlarr | `http://<host>:9696` |
| Radarr | `http://<host>:7878` |
| Sonarr | `http://<host>:8989` |
| SABnzbd | `http://<host>:8080` |
| Configarr | `http://<host>:8282` |
| slskd | `http://<host>:5030` |

Soulseek also listens on port `50300`. Privado usually does not forward it, so P2P may rely on relays.

## Soulseek (after first `up`)

1. Put Soulseek credentials in `.secrets/slskd.env` ([account](https://www.slsknet.org/news/) if needed), then `docker compose up -d slskd`.
2. Open `http://<host>:5030` (default `slskd` / `slskd`), change the UI password, and confirm `/data/media/music` is shared.
3. Create an API key in slskd (Settings → Options → API Keys).
4. Install Soularr config:

   ```bash
   mkdir -p "${PLEX_ROOT}/appdata/soularr"
   cp appdata/soularr/config.ini.example "${PLEX_ROOT}/appdata/soularr/config.ini"
   # Add Lidarr + slskd API keys, then:
   docker compose restart soularr
   ```

5. Check `docker logs soularr --tail 50`, then add a wanted album in Lidarr to test.

Soularr and Lidarr talk over `localhost` (shared Gluetun network). Both see completed downloads at `/data/downloads/complete/slskd`; Lidarr imports into `/data/media/music`.

## Updates

1. Bump image tags in `compose.yaml`
2. `docker compose pull`
3. `docker compose up -d`

## Checks

```bash
docker compose config -q
yamllint compose.yaml
```
