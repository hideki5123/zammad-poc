# zammad-tailscale-poc

[![validate](https://github.com/hideki5123/zammad-tailscale-poc/actions/workflows/validate.yml/badge.svg)](https://github.com/hideki5123/zammad-tailscale-poc/actions/workflows/validate.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

Spin up a full [Zammad](https://zammad.org) helpdesk locally with Docker Compose,
reachable **only from inside your VPN** (Tailscale by default) — no public
exposure, no TLS, no OAuth apps. Meant for evaluating Zammad before committing
to a production rollout.

日本語版は [README.ja.md](README.ja.md) をどうぞ。

## How it works

This repo is a thin overlay on the [official
`zammad-docker-compose`](https://github.com/zammad/zammad-docker-compose)
stack — upstream is cloned by `setup.sh`, never modified and never vendored,
so you keep getting upstream updates with a plain `git pull`.

The overlay does exactly three things:

- **Binds the web UI to one interface.** A `docker-compose.override.yml`
  replaces upstream's `0.0.0.0:8080` publish with
  `<your VPN IP>:<port>` using Compose's `ports: !override` — the
  [officially recommended](https://docs.zammad.org/en/latest/install/docker-compose.html)
  way to customize the stack without touching its files.
- **Seeds sane first-boot settings.** `ZAMMAD_FQDN` / `ZAMMAD_HTTP_TYPE` are
  set so cookies, WebSockets and generated links match the URL you actually
  browse (plain HTTP on a non-standard port).
- **Disables the built-in backup service** (upstream's own
  `scenarios/disable-backup-service.yml`) — pointless for a throwaway
  evaluation instance.

Everything else — PostgreSQL, Redis, memcached, Elasticsearch, the init
container that migrates and seeds the database — is stock upstream.

## Requirements

- Docker with Compose **v2.24.4+** (needed for `ports: !override`)
- [Tailscale](https://tailscale.com/) up and running (or any VPN — pass your
  interface IP by hand, see below)
- ~4 GB free RAM (Elasticsearch included); `vm.max_map_count >= 262144`
  ([Elasticsearch requirement](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#_set_vm_max_map_count_to_at_least_262144))

## Quick start

```bash
git clone https://github.com/hideki5123/zammad-tailscale-poc.git
cd zammad-tailscale-poc
./setup.sh            # clones upstream, writes .env (Tailscale IP autodetected)
docker compose up -d  # first boot runs DB migrations — takes a minute or two
```

Then open `http://<your-magicdns-name>:8081` **from a device inside your
tailnet** and walk through the Getting Started wizard (create the admin
account — no email verification; the email channel step can be skipped).

Not on Tailscale? Pass your VPN address explicitly:

```bash
BIND_IP=10.8.0.5 ZAMMAD_HOST=myhost.vpn.internal ./setup.sh
```

## Configuration

`setup.sh` writes a `.env` (never committed); adjust and re-`up` as needed.

| Variable | Default | Purpose |
|---|---|---|
| `BIND_IP` | your Tailscale IPv4 | The **only** interface the UI is published on |
| `ZAMMAD_UI_PORT` | `8081` | Host port for the web UI |
| `ZAMMAD_FQDN` | MagicDNS name + port | Seeded into Zammad on **first boot only** |
| `ZAMMAD_HTTP_TYPE` | `http` | Keep `http` — see gotchas below |
| `POSTGRES_PASS` | random | DB password, effective on **first boot only** |
| `TZ` | host timezone | Container timezone |

All upstream variables (versions, Elasticsearch heap, …) work too — see
[`.env.dist`](https://github.com/zammad/zammad-docker-compose/blob/master/.env.dist)
in the upstream repo.

## Day-2 operations

```bash
docker compose logs -f zammad-init   # first boot stuck? look here first
docker compose down                  # stop (data survives in named volumes)
docker compose down -v               # full reset, including the database
git -C zammad-docker-compose pull && docker compose pull && docker compose up -d   # upgrade
```

## Gotchas (learned the hard way)

- **Never set `http_type` to `https` while serving plain HTTP.** The session
  cookie becomes `Secure`-only, the browser drops it, and every login fails
  with *"CSRF token verification failed"*. If you did: fix it via
  `docker compose exec zammad-railsserver bundle exec rails r "Setting.set('http_type', 'http')"`
  and restart the rails server.
- **`ZAMMAD_FQDN` / `ZAMMAD_HTTP_TYPE` only apply on first boot** (they are
  database seeds). Later changes go through `Setting.set(...)` as above.
- **`POSTGRES_PASS` only applies on first boot** — the postgres volume keeps
  the credentials it was initialized with. Changing it later requires
  `docker compose down -v`.
- **Sending email with a port in the FQDN is broken** upstream
  ([zammad#5127](https://github.com/zammad/zammad/issues/5127)): the default
  notification sender becomes `noreply@host:8081`, which is not a valid
  address. Before configuring SMTP, set a port-free sender under
  *Channels → Email → Settings → Notification Sender*.
- WebSockets work out of the box — the bundled nginx proxies `/ws` and
  `/cable` same-origin, and the seeded FQDN keeps ActionCable's origin check
  happy.

## Scope

This is an **evaluation setup**, deliberately not production-grade: plain
HTTP (your VPN provides the transport security), no backups, no real mail
delivery. For production, put a TLS-terminating reverse proxy in front,
switch `http_type` to `https`, re-enable the backup service, and read the
[upstream docs](https://docs.zammad.org/en/latest/install/docker-compose.html).

## License

The files in this repository are [MIT](LICENSE)-licensed. Zammad itself and
the `zammad-docker-compose` stack (cloned at setup time, not distributed
here) are licensed under [AGPL-3.0](https://github.com/zammad/zammad/blob/develop/LICENSE).
