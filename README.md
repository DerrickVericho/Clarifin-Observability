# Clarifin Observability

Collects, stores and shows the logs and metrics of the Clarifin app and the host it runs
on. What the app emits (log fields, metric names, endpoints) is its contract, described in
the app repo's `docs/OBSERVABILITY.md`; this repo decides how that output is collected,
kept and queried.

The stack is built to run beside the app on one server: every port stays on localhost or
the internal `clarifin` network, and data lives in Docker volumes on the same disk.

## Architecture

```
SOURCES                         COLLECTION                    STORAGE      VIEW
app containers ─stdout─► Docker json-file ─┐
                                           ├─► OTel Collector ─► Loki ───────┐
app OTel SDK ──────OTLP/HTTP push──────────┤   (logs, docker_stats,          │
Docker API (container CPU, memory) ────────┘    app metrics)                 │
                                                       │                     ├─► Grafana
                                                       └──OTLP push──► Prometheus
host /proc, /sys ─► node-exporter ─────────────scrape──────────────────┤
PostgreSQL ───────► postgres-exporter ─────────scrape──────────────────┤
Collector, Loki, Prometheus, Grafana /metrics ─scrape──────────────────┘
```

- **Logs** are read from the files Docker writes, so the app needs no log exporter.
- **App metrics are pushed** over OTLP: every app process exports its own series, so added
  workers or replicas need no scrape discovery.
- **Everything else is scraped** by Prometheus, so each target has an `up` series and a
  failed Collector is still visible.

| Prometheus `job` | Source | Covers |
|---|---|---|
| (none) | App via Collector | HTTP request rate, status, duration (`http_server_*`) |
| (none) | Collector `docker_stats` | CPU, memory, network, disk I/O per container (`container_*`) |
| `otel-collector` | Collector `:8888` | Queue size, failed exports (`otelcol_*`) |
| `node` | node-exporter | Host CPU, memory, filesystems, network (`node_*`) |
| `postgres` | postgres-exporter | Connections, locks, sizes, table and checkpoint stats (`pg_*`) |
| `loki`, `prometheus`, `grafana` | Their own `/metrics` | Health of the stack itself |

## Run

Prerequisites: Docker with Compose, and the app stack started first, since it creates the
external `clarifin` network.

**First run** (Git Bash, from this folder):

```bash
cp .env.example .env                            # then adjust for this machine
mkdir -p secrets
python -c "import secrets; print(secrets.token_urlsafe(24) + 'Aa1!', end='')" > secrets/grafana_admin_password
python -c "import secrets; print(secrets.token_urlsafe(48), end='')" > secrets/grafana_secret_key
python -c "import secrets; print(secrets.token_urlsafe(32), end='')" > secrets/postgres_exporter_password
docker compose up -d
```

Then create the database role postgres-exporter logs in with (once per database; see
[docs/COMPOSE.md](docs/COMPOSE.md#postgres-exporter)), and set
`OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318` in the app's `.env`.

**Access:** Grafana at `http://127.0.0.1:3000` (user `GRAFANA_ADMIN_USER`, password from
`secrets/grafana_admin_password`), Prometheus at `http://127.0.0.1:9090`; the ports come
from `.env`. On a server,
reach them through an SSH tunnel: `ssh -L 3000:127.0.0.1:3000 <server>`.

**Everyday commands:**

```bash
docker compose ps                              # state of every service
docker compose logs -f <service>               # follow one service's log
docker compose kill -s SIGHUP prometheus       # reload prometheus.yml
docker compose restart otel-collector loki     # apply their config changes
```

## Documents

| Document | Covers |
|---|---|
| [docs/COMPOSE.md](docs/COMPOSE.md) | Each service in `docker-compose.yml`: ports, volumes, secrets, and the procedures that go with them |
| [docs/CONFIG.md](docs/CONFIG.md) | Each config file (Collector, Loki, Prometheus, Grafana): values changed from the defaults, options, and migration paths |
| `AGENTS.md` | Rules for changing this repo |
