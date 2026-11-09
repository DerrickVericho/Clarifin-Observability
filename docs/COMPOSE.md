# Compose services

Notes for each service in `docker-compose.yml`. The file itself carries no comments; a
setting that needs explaining or a warning is described here.

All services join the external `clarifin` network, created by the app stack. Only
Grafana, Prometheus and the Collector's OTLP port are published, and only on `127.0.0.1`.

## Environment

Values that differ between machines come from `.env` (copied from `.env.example`, and
gitignored). Every variable is required: a missing one stops `docker compose` instead of
falling back to a value meant for another machine.

| Variable | Used for |
|---|---|
| `GRAFANA_PORT`, `PROMETHEUS_PORT`, `OTLP_HTTP_PORT` | Host ports, on `127.0.0.1` |
| `DOCKER_HOST_IP` | docker0's address: node-exporter's listen address and Prometheus' `docker-host` |
| `GRAFANA_ADMIN_USER` | Admin login; read only when `grafana-data` is created |
| `GRAFANA_COOKIE_SECURE` | `true` once Grafana is served over HTTPS |
| `POSTGRES_HOST`, `POSTGRES_PORT`, `POSTGRES_DB`, `POSTGRES_SSLMODE` | postgres-exporter's connection. The port is the one PostgreSQL listens on at `POSTGRES_HOST` (5432 inside the `db` container), not the port the app stack publishes on the host |
| `POSTGRES_EXPORTER_USER` | The role postgres-exporter logs in as |

Passwords and keys are not variables: they are files in `secrets/`, mounted as Compose
secrets. A variable shows up in `docker inspect` and in the environment of every process
in the container; a secret file does not.

## otel-collector

- No container healthcheck: the official image is `scratch` with only the Collector
  binary, so it has no program to run one with, and Docker would not restart an unhealthy
  container anyway. Its health is `up{job="otel-collector"}` in Prometheus. Option: an
  image with a static busybox added, plus the `health_check` extension on
  `127.0.0.1:13133`.
- `user: "0"`: Docker's log files under `/var/lib/docker/containers` are readable only by
  root.
- `/var/run/docker.sock` feeds the `docker_stats` receiver. Access to the socket is root
  access to the host, and `:ro` does not limit the API. Option: put a read-only socket
  proxy in front of it, or replace `docker_stats` with cAdvisor.
- The published OTLP port lets an app run on the host (`uv run`) export metrics.
  Containers use `otel-collector:4318` on the network.
- Its own metrics are on `:8888`, scraped as `job="otel-collector"` (see
  [CONFIG.md](CONFIG.md#self-telemetry)).
- `otelcol-storage` holds the file offsets and the persistent export queues. Losing it
  drops queued data and skips lines written since the last read, because reading then
  restarts at the end of each file.

## loki

- `loki-data` holds chunks, index, WAL and compactor state; it is the log history.
  Retention keeps it bounded (see [CONFIG.md](CONFIG.md#loki)).

## prometheus

- `--web.enable-otlp-receiver` accepts the Collector's pushes at `/api/v1/otlp`.
- Retention is the default 15 days with no size cap. Option: `--storage.tsdb.retention.time`
  and `--storage.tsdb.retention.size`.
- `extra_hosts` maps `docker-host` to `DOCKER_HOST_IP`, node-exporter's listen address.

## node-exporter

- `network_mode: host` and `pid: host` make it report the host, not its own container;
  `--path.rootfs=/host` points it at the host root, mounted read-only.
- It listens on `DOCKER_HOST_IP:9100`, docker0's address: reachable from containers, not
  from the public interface. The default is `172.17.0.1`; if the Docker daemon's `bip`
  differs, set the variable to it (`ip -4 addr show docker0`).
- A host firewall that drops traffic from Docker bridges to the host (for example `ufw`
  with default deny) must allow port 9100 on docker0.
- Option: on Linux mount the root as `/:/host:ro,rslave` so mounts added later are seen.
  Docker Desktop rejects `rslave`, so the file uses `ro`.
- Under Docker Desktop the "host" is its Linux VM, not Windows.

## postgres-exporter

Logs in to the app database as `postgres_exporter`, a role with `pg_monitor`: it reads the
statistics views but no table data, and may hold at most 3 connections so a misbehaving
exporter cannot take the backend's slots.

The role lives in the database cluster, not in the app's migrations, because its password
is a secret of this repo. Create it once per cluster, and again whenever the app's database
volume is recreated (otherwise `pg_up` is 0):

```bash
{ printf "CREATE ROLE postgres_exporter LOGIN CONNECTION LIMIT 3 PASSWORD '%s';\n" "$(cat secrets/postgres_exporter_password)"; echo "GRANT pg_monitor TO postgres_exporter;"; } | docker exec -i clarifin-db psql -v ON_ERROR_STOP=1 -U clarifin -d clarifin
```

The SQL goes through stdin, so the password never appears in a process list. To rotate it,
write the new value to the file, run the same command with
`ALTER ROLE postgres_exporter PASSWORD '%s';` alone, and `docker compose up -d
--force-recreate postgres-exporter`.

- `--collector.stat_checkpointer`: PostgreSQL 17 moved checkpoint statistics out of
  `pg_stat_bgwriter`.
- `--collector.long_running_transactions` shows transactions that hold back vacuum and locks.
- `--collection-timeout` stays at or below Prometheus' `scrape_timeout` (10s), so a slow
  database never stacks connections.
- `DATA_SOURCE_URI` and `DATA_SOURCE_PASS_FILE` become `--datasource.*` flags in v0.22.
- `POSTGRES_SSLMODE=disable` holds only while the database is on the same Docker network;
  use `require` once it is on another host.
- Query-level statistics (`pg_stat_statements`) are off: they need the extension in the
  database's `shared_preload_libraries` and a restart.

## grafana

- Login is required: sign-up and org creation are off, and `GF_AUTH_BASIC_PASSWORD_POLICY`
  enforces 12+ characters with upper and lower case, a digit and a symbol.
- The admin user (`GRAFANA_ADMIN_USER`), its password and `secret_key` (both from
  `secrets/`) are read **only when Grafana creates `grafana-data`**. Afterwards:
  - Reset the admin password with
    `docker compose exec -T grafana grafana cli admin reset-admin-password --password-from-stdin < secrets/grafana_admin_password`.
  - Never change `secret_key` on an existing volume: secrets encrypted with the old key
    become unreadable. Back it up together with `grafana-data`.
- Other users are created by the admin in the UI with the Viewer role; Grafana cannot
  provision users from files. Once more than one person needs access, use SSO.
- Set `GRAFANA_COOKIE_SECURE=true` once Grafana is served over HTTPS; over plain HTTP it
  breaks login.
- `/metrics` needs no login, so anything on the `clarifin` network can read it. Option:
  `GF_METRICS_BASIC_AUTH_USERNAME` and `_PASSWORD`, with matching `basic_auth` in the
  Prometheus job.

## secrets

Files in `secrets/` are gitignored and mounted read-only at `/run/secrets/<name>`. Compose
does not encrypt them. On Linux they must be readable by the container user (Grafana's uid
is 472): `chown 472` and `chmod 400`.
