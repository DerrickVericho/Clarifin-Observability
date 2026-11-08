# AGENTS.md

Compose project for the Clarifin observability stack. The app repo's `AGENTS.md` applies
here too, except its Python code rules. What the app emits is defined in the app repo's
`docs/OBSERVABILITY.md`; this repo adapts to it and never redefines it.

## Documents

| Document | Governs |
|---|---|
| `README.md` | Purpose, architecture, how to run |
| `docs/COMPOSE.md` | Services in `docker-compose.yml` and their procedures |
| `docs/CONFIG.md` | Config files: Collector, Loki, Prometheus, Grafana provisioning |

A change to a service or a config value updates its section in the same change.

## Comments

- `docker-compose.yml` has no comments.
- Other config files keep a comment only when changing the line without reading the docs
  would break something silently or irreversibly: a value that must match another file,
  a step that cannot be undone, a value that looks wrong on purpose, or a silent effect.
  One line, phrased as a warning.
- Reasons, options, tradeoffs, procedures and commented-out alternatives go in `docs/`.
  Never mark lines as changed; git records that.

## Rules

- Never print or commit anything in `secrets/`.
- Pin every image to an exact version.
- Publish ports only on `127.0.0.1`.

## Commands

```bash
export MSYS_NO_PATHCONV=1                     # Git Bash only: keeps container paths intact
docker compose config -q                      # validate the compose file
docker compose exec -T prometheus promtool check config /etc/prometheus/prometheus.yml
docker compose kill -s SIGHUP prometheus      # reload prometheus.yml without a restart
```
