# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A self-hosted homelab managed as one Docker Compose stack per directory:

- `proxy/` — Traefik v3 reverse proxy, `docker-socket-proxy`, `whoami` test service
- `cloud/` — Nextcloud (Apache) + MariaDB + Redis + cron container
- `mail/` — Postfix (`boky/postfix`) outbound SMTP relay with auto-generated OpenDKIM keys
- `immich/` — Immich photo server + machine-learning + Postgres + Valkey + `immich-kiosk`

There is no build/lint/test tooling — these are deployment configs only.

## Commands

Each directory is an independent stack deployed in place:

This infra is deployed to a **remote host via a Docker context** (an
SSH context; run `docker context ls` to find its name). Any command that touches
the engine (pull/up/down/logs/ps) **must** target that context, otherwise it hits
the local Docker daemon. Note `--context` is a flag on `docker`, not on the
`compose` subcommand, so the form is `docker --context <ctx> compose …` (not
`docker compose --context <ctx> …`). `docker compose config` parses local files
only and needs no context.

```bash
cd <stack> && docker --context <ctx> compose up -d                       # deploy / apply changes
cd <stack> && docker --context <ctx> compose pull && docker --context <ctx> compose up -d   # update images
docker --context <ctx> compose logs -f <service>                         # tail a service
cd <stack> && docker compose config                                      # validate compose + .env (local, no context)
```

**Startup order matters** because of shared external networks (see below):
`proxy` first → `mail` next → then `cloud` / `immich`.

## Architecture

**Traefik is the single ingress.** Every web-facing service opts in via container
labels (`traefik.enable=true`, a `Host()` rule on `${DOMAIN}`, `entryPoints=https`,
`tls.certresolver=main`). Only `proxy/` publishes ports 80/443; all other stacks are
reached through Traefik, never directly. Traefik discovers containers through
`docker-socket-proxy` (read-only `tcp://docker_socket_proxy:2375`), not by mounting
the Docker socket itself.

**Cross-stack networking via externally-created Docker networks:**

- `proxy` is created by `proxy/` (with IPv6). `cloud/` and `immich/` join it as
  `external: true`. A service is only routable if it is on the `proxy` network.
- `mail` is created by `mail/`. `cloud/` and `immich/` join it as `external: true`
  so apps can send through Postfix.
- A stack's own DB/redis networks (`nextcloud_db`, `nextcloud_redis`, immich
  `default`) stay internal to that stack.

Consequence: bringing a network-owning stack down/recreating it can detach
dependent stacks; restart consumers after recreating `proxy` or `mail`.

**TLS:** wildcard cert for `*.${DOMAIN}` via Let's Encrypt DNS-01 using the OVH
provider (`OVH_*` env vars). Stored at `/docker/traefik/certs/acme.json`. Adding a
new hostname needs no cert change — just the Traefik labels.

**File provider:** the YAML config under `proxy/traefik/` (mounted to `/files`,
hot-watched) routes hosts that are *not* Docker containers — e.g. a LAN service
addressed by IP. Add off-Docker / LAN services there, not via labels.

**Host volume convention:**
- `/docker/<stack>/...` — app/config/state (small, on system disk)
- `/storage/docker/<stack>/...` and `/storage/pictures` — bulk data (large disk)

## Environment / secrets

Every stack reads a `.env` next to its `docker-compose.yml`. `.env` is gitignored
and **never** committed. `immich/template.env` is the documented template; mirror
that pattern if adding env vars. Required vars are referenced directly in each
compose file (`${DOMAIN}`, `${EMAIL}`, `${OVH_*}`, `${API_USERS}`, DB/Redis
passwords, etc.).

The `*.txt` file in `mail/` is the DKIM public-key DNS TXT record (reference for
DNS config, not consumed by Docker).

## Immich upgrade workflow

This is the most frequent change (see git history: "Upgrade Immich to vX"). Immich
ships its own compose file upstream; this repo keeps a customized version plus
`immich/docker-compose.new.yml`, the pristine upstream file used as a diff base.

To upgrade: fetch the new upstream compose to `docker-compose.new.yml`, diff it
against `docker-compose.yml`, and re-apply the local customizations onto the new
upstream — i.e. the Traefik labels and `proxy`/`mail` networks on `immich-server`,
the `immich-kiosk` service, removal of the upstream published `ports`, and the
`EXT_LIB_LOCATION` external-library mount. Pin the release in `.env`
(`IMMICH_VERSION`). Commit message format: `Upgrade Immich to vX.Y.Z`.
