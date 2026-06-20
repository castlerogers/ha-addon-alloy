# Grafana Alloy (Homelab) — Documentation

Ships the full Home Assistant OS systemd journal to the homelab **VictoriaLogs**
instance using Grafana Alloy and the Loki push protocol.

## What it collects

`journald: true` grants the add-on read access to the host systemd journal
(mounted at `/var/log/journal`). On HAOS that single journal contains everything:

- Home Assistant Core
- Supervisor
- every installed add-on
- Docker
- the host OS itself

Alloy tails the journal and forwards each entry to the configured Loki endpoint.

## Configuration

```yaml
loki_url: "https://victorialogs.<home_domain>/insert/loki/api/v1/push"
host_label: homeassistant
log_level: info
```

| Option | Required | Default | Description |
|--------|----------|---------|-------------|
| `loki_url` | yes | — | Loki push endpoint. For VictoriaLogs use the `/insert/loki/api/v1/push` path. Use the same `victorialogs.<home_domain>` FQDN the Grafana datasource uses. |
| `host_label` | no | `homeassistant` | Value of the `host` label attached to every log line. Matches the homelab `cr_alloy` convention so HAOS shows up alongside every other host in VictoriaLogs queries/dashboards. |
| `log_level` | no | `info` | Alloy's own log verbosity (`debug`, `info`, `warn`, `error`). |
| `additional_config` | no | — | Raw Alloy config appended verbatim to the generated config — escape hatch for extra sources/pipelines. |

### TLS

The homelab VictoriaLogs endpoint is fronted by Traefik with a publicly-trusted
certificate, so the add-on (Debian base + `ca-certificates`) validates it with no
extra config — identical to how the `cr_alloy` hosts push over HTTPS. If you ever
move it behind a private CA, add the CA to the image or use `additional_config`
with a `tls_config` block.

## Labels

Every log line carries:

| Label | Source | Notes |
|-------|--------|-------|
| `host` | `host_label` option | Homelab fleet convention. |
| `job` | static `systemd-journal` | Matches the `cr_alloy` journal job. |
| `unit` | `__journal__systemd_unit` | systemd unit, e.g. `docker.service`. |
| `hostname` | `__journal__hostname` | The journal's own hostname. |
| `syslog_identifier` | `__journal_syslog_identifier` | e.g. `homeassistant`, add-on slug. |
| `transport` | `__journal__transport` | `journal`, `stdout`, etc. |
| `container_name` | `__journal_container_name` | Docker container, when present. |
| `level` | `__journal_priority_keyword` | `info`, `error`, … |

Example VictoriaLogs / LogsQL queries:

```
host:homeassistant
host:homeassistant AND level:error
host:homeassistant AND syslog_identifier:homeassistant
```

## Debug UI

Alloy's UI/readiness endpoint is exposed on port `12345`
(`http://<ha-ip>:12345/` and `/-/ready`). The add-on watchdog uses `/-/ready`.

## Updating Alloy

The Alloy binary is pinned and checksum-verified in `Dockerfile`. To bump:

1. Pick the new version from <https://github.com/grafana/alloy/releases>.
2. Fetch the official checksums for the `linux-amd64` and `linux-arm64` zips:

   ```bash
   gh api repos/grafana/alloy/releases/tags/vX.Y.Z \
     --jq '.assets[] | select(.name=="SHA256SUMS") | .browser_download_url' \
     | xargs curl -fsSL | grep -E 'alloy-linux-(amd64|arm64)\.zip$'
   ```

3. Update `ALLOY_VERSION` and the two `ALLOY_SHA256` values in `Dockerfile`.
4. Bump `version:` in `config.yaml` (this is what surfaces the **Update** button
   in Home Assistant).
5. Commit and push. HA shows an update; installing it rebuilds locally.
