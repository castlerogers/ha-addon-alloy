# Castle Rogers Homelab — Home Assistant Add-ons

A small Home Assistant add-on repository for the homelab.

## Add-ons

### [Grafana Alloy (Homelab)](./alloy)

Ships the **full Home Assistant OS systemd journal** (HA Core, Supervisor, every
add-on, Docker, and the host OS) to the homelab **VictoriaLogs** instance using
Grafana Alloy and the Loki push protocol.

This exists because HAOS is a locked-down appliance — the fleet's normal
`cr_alloy` Ansible role can't run on it, so HA was the one host not shipping logs
to VictoriaLogs. The add-on closes that gap and labels its logs with the same
`host` / `job` convention as the rest of the fleet.

## Installation

1. In Home Assistant: **Settings → Add-ons → Add-on Store → ⋮ (top right) →
   Repositories**.
2. Add this repository URL:
   `https://github.com/castlerogers/ha-addon-alloy`
3. Find **Grafana Alloy (Homelab)** in the store, click **Install** (HAOS builds
   the image locally — first build takes a few minutes).
4. Open the **Configuration** tab and set `loki_url` to the homelab VictoriaLogs
   ingest endpoint, then **Start**.

See [alloy/DOCS.md](./alloy/DOCS.md) for configuration options and the
Alloy-version bump procedure.

## Design notes / trust

- Built on the **official Home Assistant base image** (`home-assistant/*-base-debian`).
- The **Alloy binary comes from Grafana's official GitHub releases**, pinned to a
  specific version and **verified against the official `SHA256SUMS`** at build
  time (`sha256sum -c`). An unverified or tampered download fails the build.
- The add-on sends data to exactly one place: the `loki_url` you configure.
  Nothing else is contacted and nothing phones home.
- The entire add-on is a config generator plus the Alloy binary — it's small
  enough to read end-to-end.
