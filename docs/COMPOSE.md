# Compose services

Notes for each service in `docker-compose.yml`. The file itself carries no comments; a
setting that needs explaining or a warning is described here.

All services join the external `clarifin` network, created by the app stack. Only
Grafana (3000), Prometheus (9090) and the Collector's OTLP port (4318) are published, and
only on `127.0.0.1`.

## otel-collector

- `user: "0"`: Docker's log files under `/var/lib/docker/containers` are readable only by
  root.
- `/var/run/docker.sock` feeds the `docker_stats` receiver. Access to the socket is root
  access to the host, and `:ro` does not limit the API. Option: put a read-only socket
  proxy in front of it, or replace `docker_stats` with cAdvisor.
- `127.0.0.1:4318` lets an app run on the host (`uv run`) export metrics. Containers use
  `otel-collector:4318` on the network.
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
- `extra_hosts: docker-host:172.17.0.1` must match node-exporter's `--web.listen-address`.

## node-exporter

- `network_mode: host` and `pid: host` make it report the host, not its own container;
  `--path.rootfs=/host` points it at the host root, mounted read-only.
- `--web.listen-address=172.17.0.1:9100` is docker0's address: reachable from containers,
  not from the public interface. If the Docker daemon's `bip` changes, change it here and
  in prometheus' `extra_hosts`.
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
- `sslmode=disable` holds only while the database is on the same Docker network; use
  `require` once it is on another host.
- Query-level statistics (`pg_stat_statements`) are off: they need the extension in the
  database's `shared_preload_libraries` and a restart.

## grafana

- Login is required: sign-up and org creation are off, and `GF_AUTH_BASIC_PASSWORD_POLICY`
  enforces 12+ characters with upper and lower case, a digit and a symbol.
- The admin password and `secret_key` come from `secrets/`, and Grafana reads them **only
  when it creates `grafana-data`**. Afterwards:
  - Reset the admin password with
    `docker compose exec -T grafana grafana cli admin reset-admin-password --password-from-stdin < secrets/grafana_admin_password`.
  - Never change `secret_key` on an existing volume: secrets encrypted with the old key
    become unreadable. Back it up together with `grafana-data`.
- Other users are created by the admin in the UI with the Viewer role; Grafana cannot
  provision users from files. Once more than one person needs access, use SSO.
- Set `GF_SECURITY_COOKIE_SECURE` to `"true"` once Grafana is served over HTTPS; over plain
  HTTP it breaks login.
- `/metrics` needs no login, so anything on the `clarifin` network can read it. Option:
  `GF_METRICS_BASIC_AUTH_USERNAME` and `_PASSWORD`, with matching `basic_auth` in the
  Prometheus job.

## secrets

Files in `secrets/` are gitignored and mounted read-only at `/run/secrets/<name>`. Compose
does not encrypt them. On Linux they must be readable by the container user (Grafana's uid
is 472): `chown 472` and `chmod 400`.
