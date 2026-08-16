# Deployment

Production guide for drawDB Collaborative. For local dev, see the
[README](README.md#local-development).

## Before you expose this anywhere

**The API and WebSocket are unauthenticated.** No accounts, no login, no
per-diagram access control. Anyone who can reach the port can read, edit, and
delete every diagram. Put one of these in front before exposing it beyond
localhost:

- An SSH tunnel or private tunnel (Tailscale, Cloudflare Tunnel, WireGuard).
- A reverse proxy that enforces auth (basic auth, OAuth2 proxy, mTLS).
- A network restriction (firewall rule, private VPC, VPN).

That's why `BIND_HOST` defaults to `127.0.0.1` and `compose.yml` publishes
`127.0.0.1:3000:3000` instead of `3000:3000` — both are deliberate, don't widen
either until something in front is doing the authentication.

## What runs

One container, four jobs:

| Job | Path | Notes |
| --- | --- | --- |
| Frontend | `/` | Static Vite build, SPA fallback |
| Diagram API | `/api/diagrams[/:id]` | JSON, 2 MB body limit |
| Collaboration | `/ws/diagrams/:id` | WebSocket, 2 MB frame limit |
| Storage | `DATABASE_PATH` | SQLite, WAL mode |

No separate frontend host, no external database. `vercel.json` is for a
**static-only** deployment of the editor — no server, no shared storage, no
collaboration.

Requires Docker Compose v2, or Node.js `^20.19.0 || >=22.12.0` for bare metal
(matches `package.json`'s `engines`; the Vite/Rolldown toolchain enforces it).
`better-sqlite3` ships prebuilt binaries, so no C toolchain needed unless
your platform lacks one.

## Quick start

```bash
git clone https://github.com/yms2772/drawdb-collaborative.git
cd drawdb-collaborative
docker compose up --build -d
```

Open `http://localhost:3000`. Diagram URLs are `/diagrams/:id`; anyone
opening the same URL joins the same live session. Data persists in the
`drawdb-data` volume — `docker compose down` keeps it, `down -v` deletes it.

The same Dockerfile builds the published image on every tag push
(`.github/workflows/docker.yml`), for `linux/amd64` and `linux/arm64`, to
`ghcr.io/<owner>/<repo>`.

## Configuration

Copy `.env.sample` to `.env` and edit — Compose reads it automatically, and
`npm start` reads it via your process manager or shell.

| Variable | Default | Purpose |
| --- | --- | --- |
| `PORT` | `3000` | Listen port. Fixed in `compose.yml`; edit the `ports:` line to change the published port. |
| `BIND_HOST` | `127.0.0.1` | Interface to bind. Docker overrides to `0.0.0.0` — the container boundary and the compose port binding are what limit exposure there, not this. |
| `DATABASE_PATH` | `./data/drawdb.sqlite` | SQLite file. `:memory:` disables persistence. |
| `TRUST_PROXY` | unset | Set `1` once a reverse proxy is confirmed in front, so `X-Forwarded-*` headers are honored. Otherwise clients can forge them. |
| `ALLOWED_ORIGINS` | unset | Comma-separated extra origins allowed to open a WebSocket. Same-origin always works; only needed when served from a different host. |

Two limits are compiled in, not configurable: request bodies and WebSocket
frames both cap at 2 MB (`MAX_DOCUMENT_BYTES` / `MAX_MESSAGE_BYTES` in
`server/`). Oversized saves get `413`.

## Reverse proxy

Forward `X-Forwarded-For`/`X-Forwarded-Proto`, pass `Upgrade`/`Connection`
through on `/ws`, and **preserve the `Host` header** — the WebSocket origin
check compares `Origin` against `Host`, so rewriting `Host` breaks same-origin
handshakes with a `403`. Set `TRUST_PROXY=1` once this is in place. The
browser derives `ws://`/`wss://` from its own origin, so no hostname needs
configuring in the frontend.

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Upgrade    $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;   # sessions are long-lived; default 60s kills idle editors
    proxy_send_timeout 3600s;
    client_max_body_size 4m;
}
```

Caddy needs no extra config — `reverse_proxy 127.0.0.1:3000` forwards
WebSockets and headers by default. Add auth there since the app has none:

```caddy
drawdb.example.com {
    basic_auth {
        alice $2a$14$...   # generate with: caddy hash-password
    }
    reverse_proxy 127.0.0.1:3000
}
```

Cookie-based SSO proxies (oauth2-proxy, etc.) work too — the WebSocket
handshake is a normal HTTP request that carries cookies. Just don't let the
proxy strip `Upgrade`.

### Cloudflare Tunnel

No open inbound port needed — `cloudflared` dials out to Cloudflare's edge.
`compose.yml` ships it as an opt-in profile, off by default:

1. Zero Trust dashboard → Networks → Tunnels → create a tunnel → copy its
   token into `.env` as `CLOUDFLARE_TUNNEL_TOKEN`.
2. Public hostname → service `http://drawdb:3000` (the compose service name,
   not `localhost` — `cloudflared` reaches it over the compose network).
3. `docker compose --profile cloudflare up --build -d`

Set `TRUST_PROXY=1` in `.env`. `ALLOWED_ORIGINS` isn't needed — `cloudflared`
preserves the original `Host` header by default, so it already matches
`Origin`; only set it if you've overridden `httpHostHeader` in the tunnel
config.

The app still has no auth of its own — add a Cloudflare Access policy on the
hostname (Zero Trust → Access → Applications) to gate it with SSO/email at
the edge before traffic ever reaches the container.

## Backups

WAL mode spreads state across three files (`drawdb.sqlite`, `-wal`, `-shm`).
Copying just `drawdb.sqlite` while the server runs produces a torn backup.
The runtime image has no `sqlite3` CLI, so the simplest safe option is a cold
volume copy:

```bash
docker compose stop drawdb
docker run --rm -v drawdb-data:/data -v "$PWD":/backup alpine \
  tar czf /backup/drawdb-data.tar.gz -C /data .
docker compose start drawdb
```

For zero-downtime backups, run SQLite's `.backup` (consistent against a live
writer, unlike `cp`) from a throwaway container:

```bash
docker run --rm -v drawdb-data:/data -v "$PWD":/backup \
  keinos/sqlite3 sqlite3 /data/drawdb.sqlite ".backup /backup/drawdb-backup.sqlite"
```

To restore: stop the container, replace the volume contents (all three WAL
files together, or a `.backup` output as the sole `drawdb.sqlite`), start it
again. Schema is `CREATE TABLE IF NOT EXISTS` on startup — no migration step.

## Upgrading

```bash
git pull
docker compose up --build -d
```

The volume survives the rebuild. Back up first across a schema-changing
release. Clients disconnect during restart, reconnect automatically, and
converge on the server's snapshot rather than overwriting it.

## Without Docker

```bash
git clone https://github.com/yms2772/drawdb-collaborative.git
cd drawdb-collaborative
npm ci && npm run build
cp .env.sample .env   # then edit
npm start
```

`npm start` serves `dist/` if it exists — build before starting or you'll get
404s on pages while the API/WebSocket still work fine.

<details>
<summary>Minimal systemd unit</summary>

```ini
[Unit]
Description=drawDB Collaborative
After=network.target

[Service]
Type=simple
User=drawdb
WorkingDirectory=/opt/drawdb-collaborative
EnvironmentFile=/opt/drawdb-collaborative/.env
ExecStart=/usr/bin/node server/index.js
Restart=on-failure
RestartSec=5
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/drawdb-collaborative/data

[Install]
WantedBy=multi-user.target
```

</details>

## Know before you rely on this

- **No auth** — see the top of this doc.
- **No health endpoint.** Probe `GET /api/diagrams` (returns `200` + JSON once
  up) if your orchestrator needs one; nothing declares a `HEALTHCHECK`.
- **No graceful shutdown.** No `SIGTERM` handler, so `docker stop` waits out
  its timeout then `SIGKILL`s. Safe for SQLite (WAL), but WebSocket clients
  drop without a close frame. Add `init: true` to the compose service for
  proper PID 1 signal/zombie handling.
- **Crashes are logged, not fatal.** `uncaughtException`/`unhandledRejection`
  handlers keep the process alive so one bad frame doesn't kill every room —
  but a genuinely broken process just keeps running. Watch for repeated
  `Uncaught exception:` lines.
- **Single instance only.** State is one SQLite file plus in-memory
  collaboration rooms. Two replicas behind a load balancer silently split
  sessions. Scale vertically.
- **No log rotation.** Plain stdout — bound it with Docker's logging driver or
  the systemd journal.
- **Large JS bundle** (~16 MB / ~3 MB gzipped). Make sure the proxy
  compresses responses (nginx: `gzip on;`; Caddy: on by default).

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Editors connect, changes don't sync, `/ws` request shows `403` | Origin check rejected the handshake — proxy rewrote `Host` | Forward the real `Host`, or add the origin to `ALLOWED_ORIGINS` |
| `/ws` request shows `404` | Diagram doesn't exist, or malformed path | Check the diagram id |
| `/ws` hangs, never upgrades | Proxy isn't passing `Upgrade`/`Connection` | Add those proxy headers |
| Sessions drop every ~60s | Proxy's idle timeout is closing the socket | Raise `proxy_read_timeout` |
| Save fails, `413` | Diagram exceeds the 2 MB limit | Raise `MAX_DOCUMENT_BYTES`/`MAX_MESSAGE_BYTES` and rebuild, plus `client_max_body_size` at the proxy |
| `SQLITE_CANTOPEN` on startup | Can't write `DATABASE_PATH`'s directory — usually a root-owned bind mount vs. uid 1000 | `chown 1000:1000` the host dir, or use a named volume |
| Build OOMs | Bundle needs more heap than available | Raise `--max-old-space-size` in the Dockerfile, or build on a bigger host |
| Everything 404s except `/api` | `dist/` missing | Run `npm run build` before `npm start` |
