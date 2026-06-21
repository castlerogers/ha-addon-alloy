# Changelog

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
