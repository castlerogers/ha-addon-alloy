# Declared before the first FROM so it stays in scope for the `FROM ${BUILD_FROM}`
# stage below (the Supervisor passes it as --build-arg at build time).
ARG BUILD_FROM

# Grafana Alloy binary source — the official multi-arch grafana/alloy image,
# pinned by tag + digest. Renovate's docker manager bumps the tag and the
# @sha256 digest together, so there are no checksums to maintain by hand. The
# digest is the multi-arch index, so each add-on arch (amd64/aarch64) resolves
# to its own variant at build time. To bump, see alloy/DOCS.md ("Updating Alloy").
FROM grafana/alloy:v1.17.0@sha256:41c41849989b7e054ccbadc17938ee1e5592fe26bfbc56ef3ffc109c0b0b2739 AS alloy

FROM ${BUILD_FROM}

# jq parses the add-on options in the s6 init script; ca-certificates lets Alloy
# validate the (publicly-trusted) VictoriaLogs TLS endpoint it pushes to.
RUN apt-get update && apt-get install -y --no-install-recommends \
    jq ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Same official Alloy binary the release zip ships, copied from the digest-pinned
# image above — integrity is guaranteed by the digest, no sha256sum -c needed.
COPY --from=alloy /bin/alloy /usr/bin/alloy

COPY rootfs /

ARG BUILD_ARCH
ARG BUILD_DATE
ARG BUILD_VERSION
LABEL \
    io.hass.name="Grafana Alloy (Homelab)" \
    io.hass.description="Ship HAOS logs to VictoriaLogs via Grafana Alloy" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="addon" \
    io.hass.version="${BUILD_VERSION}"
