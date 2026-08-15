# Deployment

This guide covers running drawDB Collaborative in production. For local
development, see the "Local Development" section of the [README](README.md).

## Contents

- [Security model](#security-model)
- [What gets deployed](#what-gets-deployed)
- [Requirements](#requirements)
- [Quick start with Docker Compose](#quick-start-with-docker-compose)
- [Configuration reference](#configuration-reference)
- [Running behind a reverse proxy](#running-behind-a-reverse-proxy)
- [Data, persistence, and backups](#data-persistence-and-backups)
- [Upgrading](#upgrading)
- [Running without Docker](#running-without-docker)
- [Operational notes and current limitations](#operational-notes-and-current-limitations)
- [Troubleshooting](#troubleshooting)

## Security model

**Read this before exposing the service to anything wider than localhost.**

The HTTP API and the WebSocket endpoint are **unauthenticated**. There are no
user accounts, no login, no per-diagram access control, and no rate limiting.
Anyone who can reach the port can list, read, modify, and delete every diagram
on the instance.

The design assumes access control happens in front of the application. Choose
one of these before exposing it:

- Keep it on loopback and reach it through an SSH tunnel or a private tunnel
  service (Tailscale, Cloudflare Tunnel, WireGuard).
- Put it behind a reverse proxy that enforces authentication (HTTP basic auth,
  an OAuth2 proxy, an identity-aware proxy, or an mTLS client certificate).
- Restrict access at the network layer (firewall rule, private VPC, VPN).

Two defaults enforce this posture, and both are deliberate:

- `BIND_HOST` defaults to `127.0.0.1`, so a bare `npm start` never listens on a
  public interface by accident.
- `compose.yml` publishes the port as `127.0.0.1:3000:3000` rather than
  `3000:3000`, so Docker does not punch a hole through the host firewall.

Widening either one publishes an unauthenticated read/write/delete surface.
Only do it once something in front of the app is doing the authentication.

The container itself is reasonably hardened already: it runs as the unprivileged
`node` user (uid 1000), ships only production dependencies, and carries no build
toolchain into the runtime stage.

## What gets deployed

A single container serves all four concerns:

| Concern | Path | Notes |
| --- | --- | --- |
| Frontend | `/` | Static Vite build from `dist/`, with SPA fallback |
| Diagram API | `/api/diagrams`, `/api/diagrams/:id` | JSON, 2 MB request body limit |
| Collaboration | `/ws/diagrams/:diagramId` | WebSocket, 2 MB max frame |
| Storage | `DATABASE_PATH` | SQLite in WAL mode, via `better-sqlite3` |

There is no separate frontend deployment and no external database. The
repository's `vercel.json` covers a **static, frontend-only** deployment of the
upstream drawDB editor; it does not run the collaboration server, so a Vercel
deployment gets no shared storage and no real-time sessions.

## Requirements

- Docker with Compose v2, for the container path.
- Or Node.js `^20.19.0 || >=22.12.0` for a bare-metal install. The build
  toolchain (Vite 8 / Rolldown) requires those minimums; older 20.x releases
  will fail to build.
- A build host with roughly 4 GB of memory available. The Dockerfile sets
  `NODE_OPTIONS="--max-old-space-size=4096"` because the client bundle is large.
- A C toolchain is **not** required on the host: `better-sqlite3` ships prebuilt
  binaries for common platforms. If your platform has no prebuild, npm will fall
  back to compiling from source and will need `python3`, `make`, and a C++
  compiler.

## Quick start with Docker Compose

```bash
git clone https://github.com/yms2772/drawdb-collaborative.git
cd drawdb-collaborative
docker compose up --build -d
```

Open `http://localhost:3000`. Diagram URLs have the form
`/diagrams/:diagramId`, and everyone who opens the same URL joins the same live
session automatically.

The compose service persists SQLite in the named volume `drawdb-data`, mounted
at `/data`, and restarts unless explicitly stopped.

To follow logs or stop the stack:

```bash
docker compose logs -f
docker compose down
```

`docker compose down` leaves the `drawdb-data` volume intact. Adding `-v`
deletes it along with every diagram, so avoid that flag unless you mean it.

### Building the image directly

```bash
docker build -t drawdb-collaborative .
docker run -d --name drawdb \
  -p 127.0.0.1:3000:3000 \
  -v drawdb-data:/data \
  drawdb-collaborative
```

The published image is also built from this Dockerfile by the
`.github/workflows/docker.yml` workflow on every pushed tag, for `linux/amd64`
and `linux/arm64`, and pushed to `ghcr.io/<owner>/<repo>`.

## Configuration reference

All configuration is by environment variable. `.env.sample` documents the same
set; copy it to `.env` for a bare-metal install.

| Variable | Default | Purpose |
| --- | --- | --- |
| `PORT` | `3000` | TCP port the HTTP and WebSocket server listens on. |
| `BIND_HOST` | `127.0.0.1` | Interface to bind. The Docker image overrides this to `0.0.0.0`, because inside the container the listener must accept traffic from the bridge network — the container boundary and the compose port binding are what limit exposure there. |
| `DATABASE_PATH` | `./data/drawdb.sqlite` | SQLite file location. The Docker image sets `/data/drawdb.sqlite`. Parent directories are created automatically. `:memory:` runs without persistence. |
| `TRUST_PROXY` | unset | Set to `1` to make Express trust one proxy hop, so `X-Forwarded-For` and `X-Forwarded-Proto` are honoured. **Only set this when a reverse proxy is guaranteed to be in front** — otherwise clients can forge their own forwarded headers. |
| `ALLOWED_ORIGINS` | unset | Comma-separated extra origins permitted to open a WebSocket, e.g. `https://drawdb.example.com`. Same-origin is always allowed, so this is only needed when the app is served from a different host than the one making the connection. |
| `NODE_ENV` | unset | Set to `production` in the image. Not required by application logic. |

Compose reads an `.env` file from the project directory automatically, or you
can add an `env_file:` key to the service.

### Non-configurable limits

These are compiled in rather than exposed as environment variables:

- 2 MB maximum JSON request body (`MAX_DOCUMENT_BYTES` in `server/index.js`).
- 2 MB maximum WebSocket frame (`MAX_MESSAGE_BYTES` in `server/websocket.js`).

Very large diagrams that exceed these will be rejected with `413` on save.

## Running behind a reverse proxy

The proxy must forward `X-Forwarded-For` and `X-Forwarded-Proto`, and must
allow the WebSocket `Upgrade` and `Connection` headers through on the `/ws`
path. Set `TRUST_PROXY=1` on the application once the proxy is in place.

It must also preserve the original `Host` header. The WebSocket origin check
compares the `Origin` header's host against the request's `Host`, so a proxy
that rewrites `Host` to the upstream address makes every same-origin handshake
fail with `403`. Either forward the real host, or list the public origin in
`ALLOWED_ORIGINS`.

The browser derives `ws://` or `wss://` from the page's own origin, so there is
no public hostname or `localhost` value to configure anywhere in the frontend.

### nginx

```nginx
server {
    listen 443 ssl;
    server_name drawdb.example.com;

    # ssl_certificate / ssl_certificate_key ...

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Required for the collaboration WebSocket.
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Collaboration sessions are long-lived; the 60s default would cut
        # idle editors off every minute.
        proxy_read_timeout  3600s;
        proxy_send_timeout  3600s;

        # Diagram snapshots can approach the server's 2 MB body limit.
        client_max_body_size 4m;
    }
}
```

### Caddy

Caddy proxies WebSockets and sets the forwarded headers without extra
configuration:

```caddy
drawdb.example.com {
    reverse_proxy 127.0.0.1:3000
}
```

### Adding authentication at the proxy

Because the app has no authentication of its own, this is where you add it. The
simplest option that works with WebSockets is HTTP basic auth — browsers replay
the credentials on the `Upgrade` request:

```caddy
drawdb.example.com {
    basic_auth {
        # Generate with: caddy hash-password
        alice $2a$14$...
    }
    reverse_proxy 127.0.0.1:3000
}
```

Cookie-based SSO proxies (oauth2-proxy and similar) also work, since the
WebSocket handshake is an ordinary HTTP request that carries cookies. Verify
that whichever solution you choose does not strip the `Upgrade` header.

## Data, persistence, and backups

All state lives in a single SQLite database at `DATABASE_PATH`, opened in WAL
mode. WAL means the on-disk state is spread across three files:

```
drawdb.sqlite
drawdb.sqlite-wal
drawdb.sqlite-shm
```

**Copying only `drawdb.sqlite` while the server is running produces a torn
backup.** Either stop the container and archive the whole volume, or take a
consistent online backup with SQLite's `.backup` command.

The runtime image is `node:22-bookworm-slim` and does **not** ship the `sqlite3`
CLI, so the simplest reliable option is a cold copy of the volume:

```bash
docker compose stop drawdb
docker run --rm -v drawdb-data:/data -v "$PWD":/backup alpine \
  tar czf /backup/drawdb-data.tar.gz -C /data .
docker compose start drawdb
```

For a hot backup with no downtime, run `.backup` from a throwaway container that
mounts the same volume and does have the CLI. `.backup` is consistent against a
live writer, unlike `cp`:

```bash
docker run --rm -v drawdb-data:/data -v "$PWD":/backup \
  keinos/sqlite3 sqlite3 /data/drawdb.sqlite ".backup /backup/drawdb-backup.sqlite"
```

To restore, stop the container, replace the volume contents, and start it
again. Restore all three WAL files together, or restore a `.backup` output as
the single `drawdb.sqlite` with the `-wal` and `-shm` files removed.

Schema tables are created on startup with `CREATE TABLE IF NOT EXISTS`; there
is no separate migration step to run.

## Upgrading

```bash
git pull
docker compose up --build -d
```

The volume survives the rebuild, so diagrams are preserved. Take a backup first
if the upgrade crosses a release that changes the schema.

Connected clients are disconnected during the restart. They reconnect
automatically, send their last known version, and converge on the server
snapshot, so in-flight edits are not silently lost — a stale client receives the
current snapshot rather than overwriting it.

## Running without Docker

```bash
git clone https://github.com/yms2772/drawdb-collaborative.git
cd drawdb-collaborative
npm ci
npm run build
cp .env.sample .env      # then edit
npm start
```

`npm start` runs `node server/index.js`, which serves `dist/` if it exists. Run
`npm run build` before starting, or the server will respond to page requests
with 404s while the API and WebSocket still work.

A minimal systemd unit:

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

# Hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/drawdb-collaborative/data

[Install]
WantedBy=multi-user.target
```

## Operational notes and current limitations

Know these before putting the service on a critical path:

- **No authentication or authorization.** Covered above. This is the single
  most important deployment consideration.
- **No health check endpoint.** There is no `/healthz`, and neither the
  Dockerfile nor `compose.yml` declares a `HEALTHCHECK`. Orchestrators that
  need one should probe `GET /api/diagrams`, which returns `200` with a JSON
  list once the server is up.
- **No graceful shutdown.** The process installs no `SIGTERM` handler, so
  `docker stop` waits the full timeout (10s by default) and then sends
  `SIGKILL`. SQLite's WAL makes this safe for data integrity, but connected
  WebSocket clients are dropped without a close frame. Node also runs as PID 1
  in the container with no init process; add `init: true` to the compose service
  if you want proper signal and zombie handling.
- **The process deliberately survives crashes.** `uncaughtException` and
  `unhandledRejection` are logged rather than fatal, so one malformed frame
  cannot take every collaboration room down. The trade-off is that a genuinely
  broken process keeps serving instead of restarting — watch the logs for
  repeated `Uncaught exception:` lines.
- **Single instance only.** State lives in one SQLite file and collaboration
  rooms are held in that process's memory. Running two replicas behind a load
  balancer will split sessions and corrupt nothing but will silently diverge.
  Scale vertically, not horizontally.
- **No log rotation or structured logging.** Output goes to stdout via
  `console.log`/`console.error`. Use Docker's logging driver or systemd's
  journal to bound it.
- **Large client bundle.** The main JavaScript chunk is roughly 16 MB
  uncompressed and 3 MB gzipped. Make sure compression is enabled at the proxy;
  nginx needs `gzip on;` with `application/javascript` in `gzip_types`, and
  Caddy compresses by default.

## Troubleshooting

**Editors connect but changes never sync.** The WebSocket handshake is failing.
Check the status code of the failed `/ws/diagrams/...` request in the browser's
network panel:

- `403 Forbidden` — the origin check rejected the handshake. The server compares
  the `Origin` header's host against the request's `Host` header, so a proxy
  that rewrites `Host` to `127.0.0.1` will fail this check. Forward the real
  host (`proxy_set_header Host $host`), or list the public origin in
  `ALLOWED_ORIGINS`.
- `404 Not Found` — the diagram id does not exist, or the path is malformed.
- No response / connection hangs — the proxy is not passing `Upgrade` and
  `Connection` headers on `/ws`.

**Sessions drop about once a minute.** The proxy's read timeout is closing idle
WebSockets. Raise `proxy_read_timeout` as shown above.

**Saves fail with `413 Request body is too large`.** The diagram exceeded the
2 MB limit. Raise `MAX_DOCUMENT_BYTES` in `server/index.js` and
`MAX_MESSAGE_BYTES` in `server/websocket.js` together, plus
`client_max_body_size` at the proxy, then rebuild.

**`SQLITE_CANTOPEN` on startup.** The process cannot write to `DATABASE_PATH`'s
directory. In Docker this usually means a bind mount is owned by root while the
container runs as uid 1000; `chown 1000:1000` the host directory, or use a named
volume, which the image's `install -d -o node -g node /data` already handles.

**Build fails with an out-of-memory error.** Raise the value in
`NODE_OPTIONS="--max-old-space-size=4096"`, or build the image on a larger host
and deploy the resulting image instead of building in place.

**Everything returns 404 except `/api`.** `dist/` is missing. Run
`npm run build` before `npm start`.
