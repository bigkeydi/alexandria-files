# Alexandria-Files

Docker Compose stack for Plex-adjacent tooling: requests (Petio / Overseerr), *arr clients, a landing dashboard (Heimdall / Homarr), and Nginx Proxy Manager (NPM).

Run all commands from **this directory** (paths in `docker-compose.yml` are relative to here).

## Prerequisites

- [Docker](https://docs.docker.com/engine/install/) and Docker Compose v2
- A copy of `.env` created from `.env.example` (see below)
- For the `shared_folder_09` volume: a NAS or SMB share reachable from the host, with `CIFS_*` variables set in `.env` (optional if you remove or override that volume in compose)

## Quick start

1. Copy the environment template and fill in secrets:

   ```powershell
   Copy-Item .env.example .env
   # Edit .env — never commit .env
   ```

2. Adjust **host paths** in `docker-compose.yml` if needed (Sonarr/Radarr mounts for `D:\`, `G:\`, `E:\`, etc. are machine-specific).

3. Start the stack:

   ```powershell
   cd D:\Docker\Alexandria-Files
   docker compose up -d
   ```

4. Check status:

   ```powershell
   docker compose ps
   ```

## Services and default ports

| Service | Host port | Notes |
|--------|-----------|--------|
| Petio | `7777` | Requires MongoDB (`mongo` service) |
| MongoDB | _(internal)_ | Data: `./mongo/db` — **back this up** (`mongodump`) |
| Prowlarr | `9696` | |
| Overseerr | `5055` | |
| Heimdall | `11080` (HTTP), `11443` (HTTPS) | Uses `./heimdall/config` → **`/config`** in the container |
| Nginx Proxy Manager | `80`, `81`, `443` | Admin UI: `http://<host>:81` |
| MariaDB (for NPM) | _(internal)_ | Data: `./srv/config/nginxproxymanager/db` |
| Radarr (4K) | `10200` | Maps to app port `7878` |
| Sonarr | `8989` | |
| Watchtower | _(none)_ | Updates images on schedule; uses `WATCHTOWER_DISCORD_URL` in `.env` |
| Homarr | `7575` | |

Networks: most services use Compose default networks; Petio and Mongo use `petio-network-v2`.

## Repository layout

Tracked in Git:

- `docker-compose.yml` — stack definition
- `.env.example` — template for variables (no secrets)
- `.gitignore` — excludes secrets and runtime data

**Not** tracked (local / runtime): `petio/`, `mongo/`, `srv/`, `overseerr/`, `heimdall/`, `sonarr/`, `radarr-4k/`, `homarr/`, `.env`, optional `mongo-backups/`, `_docker-root-residual/`, etc. See `.gitignore` for the full list.

## Environment variables

Copy `.env.example` → `.env` and set at least:

- Timezone: `GLOBAL_TZ`
- NPM / MariaDB: `NPM_*`, `MYSQL_*`
- CIFS (if used): `CIFS_MOUNT_OPTS`, `CIFS_DEVICE`
- Arr PUID/PGID: `RADARR_*`, `SONARR_*`
- Watchtower (optional): `WATCHTOWER_DISCORD_URL`

## Backups

- **MongoDB (Petio):** Petio data lives in Mongo. Use periodic dumps, e.g.:

  ```powershell
  docker exec mongo mongodump --db=petio --out=/tmp/petio-dump
  docker cp mongo:/tmp/petio-dump/petio ".\mongo-backups\petio-$(Get-Date -Format yyyyMMdd-HHmmss)"
  docker exec mongo rm -rf /tmp/petio-dump
  ```

- **NPM:** Preserve `./srv/config/nginxproxymanager/` (including `db` and `letsencrypt`).

- **Other apps:** Each `*/config` folder under this directory is that app’s persistent state.

## Heimdall volume

LinuxServer **Heimdall** stores all data under **`/config`**. This repo mounts `./heimdall/config:/config`. Do **not** mount to `/app/config` — that leaves real data in an anonymous Docker volume and makes the host folder look empty.

## CIFS / SMB volume

`shared_folder_09` is defined as a Compose **volume** backed by CIFS. The host must allow Docker to mount SMB (credentials in `.env`). If you do not use that share, comment out the volume and remove its references from Radarr/Sonarr after replacing with local paths you prefer.

## Updating images

Watchtower is configured to update images on a cron schedule. For a one-off manual refresh:

```powershell
docker compose pull
docker compose up -d
```

## Security

- Do **not** commit `.env` or API keys stored under app config directories.
- After rotating Plex or NPM credentials, update the relevant `<app>/config` files or UI settings.
