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
upstream — the Traefik labels and `proxy`/`mail` networks on `immich-server`, the
`immich-kiosk` service, removal of the upstream published `ports`, the
`EXT_LIB_LOCATION` external-library mount, and the per-service `networks:` blocks.
In practice the only genuine upstream change most releases is the pinned Valkey
and Postgres image digests — take those, keep everything else.

⚠️ **Keep `${UPLOAD_LOCATION}:/usr/src/app/upload`.** Upstream mounts it at
`/data` (changed in v1.137.0); this library was created on the legacy path and
Immich keeps backward-compat for it. Switching to `/data` orphans every asset
(broken thumbnails) until an `immich-admin change-media-location` DB migration —
do **not** "correct" this line to match upstream.

The app version is pinned via `IMMICH_VERSION` in the gitignored `.env` on the
host, so it is not visible in git — bump it there as part of the upgrade. Before
deploying, `pg_dumpall` the `immich` DB (Immich runs irreversible schema
migrations on server start). Verify post-deploy via `/api/server/version` and an
asset-count check against the pre-upgrade baseline. If the upstream Postgres
image digest is unchanged, `immich_postgres` won't be recreated (no DB engine
migration). Commit only `docker-compose.yml`; message: `Upgrade Immich to vX.Y.Z`.

## Nextcloud upgrade workflow

The second-most-frequent change (git history: "Upgrade to Nextcloud NN"). It is
image-driven: the container entrypoint runs `occ upgrade` automatically on start
when the image version differs from the installed one. Nextcloud is strict —
mishandling corrupts the instance.

**Sequential majors only, latest point release first.** You cannot skip a major
(30→31→32, never 30→32), and you must be on the latest N.x point release before
jumping to N+1. Because the `NN-apache` tag floats to the latest point release,
each major step is two deploys:

1. `pull` + `up -d` the `nextcloud` **and** `cron` services on the current
   `NN-apache` tag → entrypoint upgrades to latest NN.x. Then run `cron.php` ×3
   (drains background migrations — required before the next major).
2. Edit `cloud/docker-compose.yml` to `(NN+1)-apache` on **both** services,
   `pull` + `up -d`, `cron.php` ×3 again.

Expect a ~1 min `502 Bad Gateway` window per recreate while the entrypoint runs.

**Backup before deploying** (host disk runs near-full — stream off-host):
`occ maintenance:mode --on`, `mariadb-dump --single-transaction` of the DB
(gzip, small) and `tar` of `/var/www/html/config`, then `maintenance:mode --off`
before recreating (the entrypoint manages its own maintenance flag). Do **not**
back up `/var/www/html/data` — it is the bulk user-data mount, untouched by the
upgrade.

**Post-upgrade hygiene:** a major upgrade silently disables third-party apps;
`occ app:update --all` then `occ app:enable <app>` to bring them back on a
compatible build. Run `occ db:add-missing-primary-keys` and
`occ db:add-missing-indices`. Also run `occ maintenance:update:htaccess` —
otherwise stale rewrite rules break `.well-known/webfinger|nodeinfo` and trip
the WebDAV / `.well-known` setupchecks (caldav/carddav keep working, which masks
it). And run `occ maintenance:repair --include-expensive` to apply the mimetype
and schema migrations that majors do not auto-run. Verify with `occ setupchecks`,
`occ status` (`needsDbUpgrade:false`), and `status.php` for the version. The many
apps shown "disabled" by `occ app:list` are Nextcloud's default-off shipped apps
— normal; residual `⚠` for "maintenance window start" and "HTTP headers"
(reverse-proxy header hardening, a separate Traefik-middleware concern) are
expected and harmless.

Commit only `cloud/docker-compose.yml` (the major-tag bump; the intermediate
point-release step needs no file change); message: `Upgrade to Nextcloud NN`.
