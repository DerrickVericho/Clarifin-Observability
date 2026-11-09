# Configuration

Notes for each config file: what differs from the defaults and why, the options considered,
and how a value changes when the stack moves. Comments in the files themselves are kept for
warnings only.

## OTel Collector

`otel-collector/config.yaml`

### Receivers

- **`otlp`**, HTTP only on 4318. The app's SDK exports over HTTP; gRPC (4317) is not opened.
- **`filelog`** reads Docker's `json-file` logs (`/var/lib/docker/containers/*/*-json.log`).
  - `start_at: end` applies only when no offset is stored: the first start skips old logs,
    and later starts resume from the offsets in `file_storage`.
  - The `container` operator parses the Docker format. `add_metadata_from_filepath` stays
    false because it parses Kubernetes paths; Docker paths hold only the container ID.
  - Only containers with the `tag` log option are shipped (the `filter` operator). The tag
    becomes `service.name`, which Loki indexes as `service_name`. The app stack sets the
    tag on its containers; this stack's own containers do not set it and are not collected.
- **`docker_stats`** polls the Docker API every 30s for container CPU, memory, network and
  block I/O. Its series carry `container_name` but no `job` label.

### Processors and exporters

- `batch` groups data before export. Option, not set: `memory_limiter`, which refuses data
  instead of letting the Collector run out of memory; add it once a container memory limit
  is set.
- Both exporters use `otlp_http` to Loki's `/otlp` and Prometheus' `/api/v1/otlp`, with a
  persistent `sending_queue` in `file_storage` and retries for up to 30m. Data survives a
  Collector restart or a 30-minute outage of either store.
- The 30m retry window is why Prometheus accepts samples up to 30m old
  (`out_of_order_time_window`); change both together.

### Self-telemetry

The Collector serves its own metrics on `:8888`, scraped as `job="otel-collector"`; `up`
of that job is its health. Useful series: `otelcol_exporter_queue_size` (data waiting to
be sent), `otelcol_exporter_send_failed_*`, and `otelcol_receiver_accepted_*` /
`otelcol_receiver_refused_*` per receiver.

Under Kubernetes, add the `health_check` extension on `0.0.0.0:13133` for the kubelet's
liveness and readiness probes.

### Moving off Docker

Only the receiver block changes; processors, exporters and Loki stay the same.

- **VM with systemd:** replace `filelog` with the `journald` receiver filtered on the
  app's units. The app's JSON stdout is unchanged.
- **Kubernetes:** point `filelog` at `/var/log/pods/*/*/*.log`, keep the `container`
  operator (it detects the containerd and CRI-O formats), and set
  `add_metadata_from_filepath: true` so namespace, pod and container names replace the tag.

## Loki

`loki/config.yaml`. A single process (the default `target: all`) with every component inside, storing
on the local filesystem. That suits one server; the next step is object storage, then
separate read and write components.

- **Single tenant.** `auth_enabled: false` makes every request use tenant `fake`. It is not
  a login: Loki is reachable only on the `clarifin` network.
- **Schema.** One period from 2020-10-24: TSDB index, schema v13, 24h index tables. Periods
  are append-only; editing one makes the data written under it unreadable.
- **WAL** on `/loki/wal`, so a crash loses no acknowledged lines. `replay_memory_ceiling`
  (4GB) must stay below the container memory limit once one is set.
- **Retention.** The compactor deletes data older than `retention_period` (720h, 30 days)
  every 10 minutes, after a 2h delay. It is time-based only: disk use is roughly the daily
  ingest times 30, so watch `loki-data` with `node_filesystem_avail_bytes`. Retention
  requires `delete_request_store`.
- **Limits** stay at the defaults (4 MB/s ingestion, 3 MB/s per stream, 256KB lines,
  5000 streams), so a runaway log loop is rejected instead of filling the disk. Rejected
  lines show in `loki_discarded_samples_total`.
- **`allow_structured_metadata: true`** is required by the OTLP endpoint: log attributes
  that are not labels are stored as structured metadata.
- **Queries** may span 721h and return 5000 lines; results are cached in memory (100MB).
- **`analytics.reporting_enabled: false`** stops usage reports to Grafana Labs.

### Moving to object storage

Add the S3 credentials under `storage_config`, then append a period that starts on a
future date; data before it stays on the filesystem and stays readable.

```yaml
storage_config:
  aws:
    s3: https://<access_key>:<secret_key>@<endpoint>/<bucket>
    s3forcepathstyle: true

schema_config:
  configs:
    - from: 2020-10-24          # unchanged
      ...
    - from: <a date after the rollout>
      store: tsdb
      object_store: s3
      schema: v13
      index:
        prefix: index_
        period: 24h
```

Credentials belong in a secret, not in the file. MinIO on the same disk adds no
durability; the point is storage that outlives the server.

## Prometheus

`prometheus/prometheus.yml`

- **`scrape_interval: 15s`** for scraped targets. App metrics arrive by push every 60s
  instead, so they are coarser.
- **`out_of_order_time_window: 30m`** accepts samples the Collector replays after an
  outage; keep it at least the Collector's retry window.
- **OTLP `promote_resource_attributes`** turns `service.version`,
  `deployment.environment.name` and `container.name` into labels. Every promoted attribute
  multiplies series by its distinct values, so only low-cardinality attributes belong
  here. The rest stays in `target_info`.
- **Scrape jobs** are listed in the README. `docker-host` resolves to node-exporter's
  listen address through `extra_hosts` (see [COMPOSE.md](COMPOSE.md#node-exporter)).
- **Grafana's `/metrics` is unauthenticated**; see
  [COMPOSE.md](COMPOSE.md#grafana) for the option.

## Grafana provisioning

`grafana/provisioning/`

- **Datasources** have fixed `uid`s (`prometheus`, `loki`) that dashboards reference.
  `timeInterval: 60s` matches the app's export interval: with a shorter interval,
  `$__rate_interval` holds one sample and `rate()` returns nothing.
- **Dashboards** load from `dashboards/json/` with `allowUiUpdates: false`: UI edits are
  lost on restart, so change the JSON. The **Clarifin** dashboard shows uploads by status,
  upload duration, container CPU and memory, and failed requests from the logs.
- **Counting app requests:** `increase()` misses the first request of a new series, since a
  counter's first sample is already 1. Each app process has its own `instance` and its
  series never resets, so counts use
  `last_over_time(x[r]) - (x offset r or last_over_time(x[r]) * 0)` instead.
