# Kavita on the homelab

Self-hosted [Kavita](https://www.kavitareader.com/) — a fast, cross-platform
reading server for manga, comics, and ebooks — reachable **only over the
tailnet** using the same tailscale-sidecar pattern as
`~/Dev/storyteller-container`, `~/Dev/yamtrack-container`, `~/Dev/romm`, and
`~/Dev/n8n`. No funnel, no published host ports — the web UI is available only at
`https://kavita.tail4fde5e.ts.net` while you're on the tailnet.

## What Kavita is

- A self-hosted reading server and web reader for **manga, comics, and books**.
- Scans your libraries, extracts metadata, generates covers/thumbnails, tracks
  reading progress, and serves the content through a fast web reader and OPDS.
- Published as an official Docker image `jvmilazz0/kavita:latest` (stable
  branch, Ubuntu-based, with amd64/arm64/armv7 builds) — no local build needed.

## How it works

- A **tailscale sidecar** joins the tailnet as host `kavita`, provisions an
  HTTPS cert, and runs `tailscale serve` from `ts-serve.json`. No
  `AllowFunnel` → tailnet-only.
- **`kavita`** shares the sidecar's namespace
  (`network_mode: service:tailscale`) and binds `127.0.0.1:5000`. It is not
  published on the host, so nothing on the LAN/internet can reach it.
- `ts-serve.json` proxies `443` → `127.0.0.1:5000` for `${TS_CERT_DOMAIN}`
  (web UI + API).

Kavita needs **no secret key and no external database** — state lives in the
`kavita_config` volume, so the stack is just the sidecar plus one app container.

## Run

Fill `.env` (`TAILSCALE_AUTH_KEY`, `TS_CERT_DOMAIN`) if you haven't, then:

```bash
docker compose up -d
docker compose logs -f kavita
```

Verify over the tailnet:

```bash
tailscale status | grep kavita
curl -sI https://kavita.tail4fde5e.ts.net
```

## First-time setup

1. Visit `https://kavita.tail4fde5e.ts.net` and create your admin account.
2. Add a **Library** and point it at `/library` (the shared volume), choosing
   the library type (Manga / Comic / Book) and any subfolder (e.g. `/library/manga`).
3. Kavita will scan the folder, generate covers, and watch for new files.

## Configuration

Kavita is configured through its web UI; the compose file only sets `TZ`.

| Variable | Purpose | Default |
|----------|---------|---------|
| `TZ` | Timezone | `America/New_York` |

Container path `/kavita/config` **must not be changed** — it's where Kavita
stores its SQLite database, covers, cache, logs, backups, and JWT keys. It is
persisted in the named volume `kavita_config`.

### Library volume (auto-import)

Media lives in the named volume `kavita_library`, mounted at `/library` (rw).
It's created and seeded once on the homelab, then declared `external` in compose
so it can't be accidentally removed by `docker compose down -v`.

Recommended layout inside the volume:

```
/library/
  manga/
  comics/
  books/
```

**Dropping files in from other containers** — mount the same volume (rw) in the
source container and copy files into it:

```bash
docker run --rm \
  -v kavita-container_kavita_library:/library:rw \
  -v /path/from/your/container:/src:ro \
  alpine sh -c 'cp -r /src/. /library/ && chown -R 1000:1000 /library'
```

If you'd rather serve media that already lives on the homelab disk, replace the
`kavita_library:/library:rw` line with a bind mount
(e.g. `- /srv/kavita/library:/library:rw`) — no other change is needed.

## Security notes

- The instance is intentionally **not** exposed to the LAN/internet; it's safe
  behind the tailnet. Do **not** add published ports or funnel access.
- There is no separate auth secret to leak — the only secret is
  `TAILSCALE_AUTH_KEY` in `.env` (git-ignored). Rotate it in the Tailscale admin
  console if it ever leaks.
- Do **not** set `user:` / `--user` on the Kavita container — the image manages
  its own ownership and can break `/kavita/config` permissions.

## Files

| File              | Purpose                                   |
|-------------------|-------------------------------------------|
| `compose.yaml`    | tailscale sidecar + Kavita                |
| `ts-serve.json`   | `tailscale serve` config (tailnet-only)   |
| `.env`/`.env.example` | secrets/config                        |
