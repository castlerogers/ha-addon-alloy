# Changelog

## 1.2.0

- **Container journal entries now carry `stream`, not `level`.** For a real
  systemd unit, journal `PRIORITY` is severity. For an entry written by docker's
  journald log driver (these carry `CONTAINER_NAME`) it only encodes which
  stream the line came from — stdout is stamped `info` (6), stderr `error` (3) —
  regardless of what the line says. Promoting that to `level` inverted the
  signal: a chatty stderr logger read as 100% errors, while a container's real
  errors on stdout read as `info`. Priority is now promoted to `level` only for
  non-container entries; container entries get `stream` = `stdout`/`stderr`,
  matching the homelab `cr_alloy` convention where `job:docker` carries `stream`
  and `level` is reserved for actual severity.

  This keeps the fleet-wide `level:(error OR crit)` tripwire honest. It also
  means silencing an add-on via `exclude_syslog_identifiers` becomes purely a
  log-volume decision — it is no longer a prerequisite for a trustworthy error
  count, so previously-excluded add-ons can be shipped again if you want their
  logs back.

  **Breaking for queries:** `host:homeassistant level:error` no longer matches
  add-on/container output. Use `stream:stderr` for that — and read it as
  "written to stderr", not "is an error".

- **`exclude_syslog_identifiers` entries now match either add-on prefix.**
  Supervisor 2026.07 renamed add-on containers and syslog identifiers from
  `addon_<hash>_<slug>` to `app_<hash>_<slug>`. Relabel regexes are fully
  anchored, so an entry pinned to one prefix silently stopped matching across
  that rename — the add-on simply resumed shipping, with nothing to indicate the
  rule had gone dead (castlerogers/infra#2110: ~78k spurious error lines in 24h).
  An `addon_`- or `app_`-prefixed entry is now compiled to `(addon|app)_<rest>`
  and matches either. Entries that aren't add-on identifiers are unchanged.

## 1.1.1

- Pass `--disable-reporting` to `alloy run` to suppress Alloy's anonymous usage
  report to `stats.grafana.org`. That endpoint is blocked at our egress (resolves
  to `0.0.0.0`), so the ~4-hourly POST failed and flooded the add-on log — HAOS
  tags add-on stdout as `level=error`, so it landed in VictoriaLogs as ~2,880
  spurious error lines/day. Log shipping was never affected; this only stops the
  noise at the source.

## 1.1.0

- Add `exclude_syslog_identifiers` option: a list of syslog identifiers whose
  journal entries are dropped before shipping (relabel `action=drop`). Lets you
  silence chatty add-ons (e.g. Ring-MQTT, which writes everything to stderr and
  so lands in VictoriaLogs as `level=error`) without a rebuild — just edit the
  list in the Configuration tab and restart. Default `[]` ships everything.

## 1.0.1

- Renovate now keeps the `grafana/alloy` binary current, pinned by tag **and**
  `@sha256` digest (no more hand-maintained checksums).
- Fix add-on build: drop `@sha256` digests from `build.yaml` `build_from` (the
  Supervisor rejects them and silently falls back to its default Alpine base).

## 1.0.0

- Initial release.
- Ships the HAOS systemd journal to VictoriaLogs via Grafana Alloy (Loki push).
- Grafana Alloy v1.17.0, pinned and checksum-verified against the official
  `SHA256SUMS`.
- `host` / `job` labels match the homelab `cr_alloy` convention; promotes
  `unit`, `hostname`, `syslog_identifier`, `transport`, `container_name`, and
  `level` to labels.
- Options: `loki_url`, `host_label`, `log_level`, `additional_config`.
