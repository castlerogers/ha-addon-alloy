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
exclude_syslog_identifiers:
  - app_03cabcc9_ring_mqtt
```

| Option | Required | Default | Description |
|--------|----------|---------|-------------|
| `loki_url` | yes | — | Loki push endpoint. For VictoriaLogs use the `/insert/loki/api/v1/push` path. Use the same `victorialogs.<home_domain>` FQDN the Grafana datasource uses. |
| `host_label` | no | `homeassistant` | Value of the `host` label attached to every log line. Matches the homelab `cr_alloy` convention so HAOS shows up alongside every other host in VictoriaLogs queries/dashboards. |
| `log_level` | no | `info` | Alloy's own log verbosity (`debug`, `info`, `warn`, `error`). |
| `exclude_syslog_identifiers` | no | `[]` | List of syslog identifiers whose journal entries are dropped before shipping (relabel `action=drop`). Use to hold down log volume from very chatty add-ons. See [Excluding noisy add-ons](#excluding-noisy-add-ons). |
| `additional_config` | no | — | Raw Alloy config appended verbatim to the generated config — escape hatch for extra sources/pipelines. |

### Excluding noisy add-ons

Some add-ons are extremely chatty — Ring-MQTT emits ~47k lines/day of routine
device telemetry. Add each offender's syslog identifier to
`exclude_syslog_identifiers` and its journal entries are dropped before they're
shipped. They stay visible in the add-on's own log viewer; they just don't reach
VictoriaLogs:

```yaml
exclude_syslog_identifiers:
  - app_03cabcc9_ring_mqtt
```

This is a **log-volume** decision, nothing more. It used to also be the only way
to keep the fleet error tripwire honest, because stderr-logging add-ons landed in
VictoriaLogs as `level=error`; since 1.2.0 container output carries `stream`
instead of `level`, so a chatty add-on no longer distorts error counts and there
is no signal-quality reason to drop it. See [Labels](#labels).

Find the exact identifier in VictoriaLogs — it's the `syslog_identifier` label
(add-ons appear as `app_<hash>_<slug>`):

```
host:homeassistant | stats by (syslog_identifier) count() as n | sort by (n desc)
```

Each value is treated as an anchored regex, so `app_.*_ring_mqtt` works too if
you'd rather not pin the install-specific hash.

An entry that starts with `addon_` or `app_` is compiled to match **either**
prefix. Supervisor 2026.07 renamed add-on containers and syslog identifiers from
`addon_<hash>_<slug>` to `app_<hash>_<slug>`, and because these regexes are fully
anchored, a rule pinned to the old prefix silently stopped matching — the add-on
resumed shipping with nothing to indicate the rule had gone dead. Either spelling
now works and survives the rename in both directions.

After editing, **Save** and **Restart** the add-on — the Alloy config is
regenerated on start. The startup banner echoes `Excluded ids:` so you can
confirm what the add-on actually read.

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
| `level` | `__journal_priority_keyword` | Severity — `info`, `error`, … **Non-container entries only.** |
| `stream` | `__journal_priority_keyword` | `stdout` / `stderr`. **Container entries only.** |

### Why `level` and `stream` are mutually exclusive

Journal `PRIORITY` means two different things depending on who wrote the entry.
For a real systemd unit it is severity. For an entry emitted by docker's journald
log driver — these are the ones carrying `container_name` — it only records which
stream the line came from: stdout is stamped `info` (6), stderr `error` (3),
whatever the line actually says.

Labelling that as `level` inverts the signal. A container that logs everything to
stderr reads as 100% errors, while a container's genuine errors written to stdout
read as `info`. So priority is promoted to `level` only for non-container
entries; container entries get `stream` instead. That mirrors the homelab
`cr_alloy` convention, where `job:docker` carries `stream` and `level` is
reserved for real severity — and it keeps the fleet-wide
`level:(error OR crit)` tripwire meaningful.

Practical consequence: to find what an add-on wrote to stderr, query
`stream:stderr`, not `level:error` — and read it as "written to stderr", not "is
an error". To find actual problems in an add-on, match on content.

Example VictoriaLogs / LogsQL queries:

```
host:homeassistant
host:homeassistant AND level:error                     # host/systemd severity
host:homeassistant AND stream:stderr                   # container stderr output
host:homeassistant AND syslog_identifier:homeassistant
```

## Debug UI

Alloy's UI/readiness endpoint is exposed on port `12345`
(`http://<ha-ip>:12345/` and `/-/ready`). The add-on watchdog uses `/-/ready`.

## Updating Alloy

The Alloy binary is copied from the official `grafana/alloy` image, pinned by
tag **and** digest in `Dockerfile`. **Renovate keeps it current automatically**
— the GitHub Renovate instance on icecrown watches this repo and opens a PR
bumping the `grafana/alloy` tag and its `@sha256` digest together. There are no
checksums to maintain by hand; the digest guarantees binary integrity.

After a Renovate (or manual) bump, one deliberate manual step remains:

1. Bump `version:` in `config.yaml`. This is what surfaces the **Update** button
   in Home Assistant. Renovate does not touch it, so the add-on only updates when
   you choose to (HA add-ons update on demand, never silently).
2. Merge. HA shows an update; installing it rebuilds the add-on locally against
   the new digest-pinned binary.

To bump manually instead of waiting for Renovate, edit the `grafana/alloy` tag
in `Dockerfile` and refresh its digest:

```bash
docker buildx imagetools inspect grafana/alloy:vX.Y.Z | grep '^Digest:'
```

The Home Assistant base images in `build.yaml` use plain `:bookworm` tags — HA's
add-on build config rejects `@sha256` digests in `build_from` (the Supervisor
silently falls back to its default Alpine base if it sees one). They stay current
on their own: the Supervisor builds with `--pull`, so every rebuild fetches the
latest `bookworm` base.
