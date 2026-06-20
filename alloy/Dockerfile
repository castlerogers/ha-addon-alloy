ARG BUILD_FROM
FROM ${BUILD_FROM}

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl bash jq unzip ca-certificates \
    && rm -rf /var/lib/apt/lists/*

ARG BUILD_ARCH
# Grafana Alloy — pinned and checksum-verified against the official SHA256SUMS
# for the release below. The binaries come straight from Grafana's GitHub
# releases; the sha256sum -c check fails the build if the download is tampered
# with or corrupted. To bump the version, see alloy/DOCS.md ("Updating Alloy").
RUN set -eux; \
    ALLOY_VERSION="1.17.0"; \
    case "${BUILD_ARCH}" in \
        amd64)   ALLOY_ARCH="linux-amd64"; ALLOY_SHA256="a905f3c750855323e12b23630150972956e24c993f0c9170ca2f0af4bd442d8e" ;; \
        aarch64) ALLOY_ARCH="linux-arm64"; ALLOY_SHA256="169f2b3b0f0bff30b9fd7b8c924b950c4aa910e20f39979abbb510f1c3e9daf7" ;; \
        *) echo "Unsupported architecture: ${BUILD_ARCH}" >&2; exit 1 ;; \
    esac; \
    curl -fsSL "https://github.com/grafana/alloy/releases/download/v${ALLOY_VERSION}/alloy-${ALLOY_ARCH}.zip" -o /tmp/alloy.zip; \
    echo "${ALLOY_SHA256}  /tmp/alloy.zip" | sha256sum -c -; \
    unzip /tmp/alloy.zip -d /tmp/alloy-extract; \
    mv "/tmp/alloy-extract/alloy-${ALLOY_ARCH}" /usr/bin/alloy; \
    chmod +x /usr/bin/alloy; \
    rm -rf /tmp/alloy.zip /tmp/alloy-extract

COPY rootfs /

ARG BUILD_DATE
ARG BUILD_VERSION
LABEL \
    io.hass.name="Grafana Alloy (Homelab)" \
    io.hass.description="Ship HAOS logs to VictoriaLogs via Grafana Alloy" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="addon" \
    io.hass.version="${BUILD_VERSION}"
