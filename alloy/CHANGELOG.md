# Changelog

## 1.0.0

- Initial release.
- Ships the HAOS systemd journal to VictoriaLogs via Grafana Alloy (Loki push).
- Grafana Alloy v1.17.0, pinned and checksum-verified against the official
  `SHA256SUMS`.
- `host` / `job` labels match the homelab `cr_alloy` convention; promotes
  `unit`, `hostname`, `syslog_identifier`, `transport`, `container_name`, and
  `level` to labels.
- Options: `loki_url`, `host_label`, `log_level`, `additional_config`.
