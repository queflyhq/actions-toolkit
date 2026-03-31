# =============================================================================
# Catalyst — ultra-light CI/CD runner
# Public image: queflyhq/catalyst:latest
#
# Philosophy: Ship nothing. Install what you need, when you need it.
# Use the `catalyst` CLI to install tools on demand in your pipeline.
#
# Usage:
#   container: queflyhq/catalyst
#   steps:
#     - run: catalyst install go java node
#     - run: go test ./...
# =============================================================================

FROM alpine:3.21

# Upgrade all packages to latest patched versions
RUN apk upgrade --no-cache

# Minimal base — only essentials, no vulnerable packages
# tar/unzip/wget moved to `catalyst install` on-demand
RUN apk add --no-cache \
    bash git openssh-client ca-certificates make \
    && rm -rf /var/cache/apk/*

# Static binaries (no Alpine package CVEs)
RUN set -eux; \
    # curl — static build from official releases
    CURL_VER="8.14.1"; \
    wget -q "https://github.com/moparisthebest/static-curl/releases/latest/download/curl-amd64" \
      -O /usr/local/bin/curl && chmod +x /usr/local/bin/curl; \
    # jq — static build
    wget -q "https://github.com/jqlang/jq/releases/latest/download/jq-linux-amd64" \
      -O /usr/local/bin/jq && chmod +x /usr/local/bin/jq; \
    # yq — static build
    wget -q "https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64" \
      -O /usr/local/bin/yq && chmod +x /usr/local/bin/yq; \
    # Remove wget after use (it has CVEs)
    apk del wget 2>/dev/null || true

COPY catalyst /usr/local/bin/catalyst
RUN chmod +x /usr/local/bin/catalyst

# Verify
RUN echo "=== Catalyst ===" && \
    git --version && \
    /usr/local/bin/curl --version | head -1 && \
    /usr/local/bin/jq --version && \
    catalyst --help | head -1

# --- Security hardening ---

# Non-root user
RUN addgroup -g 1000 catalyst && \
    adduser -u 1000 -G catalyst -h /home/catalyst -s /bin/bash -D catalyst && \
    mkdir -p /home/catalyst/.local/bin && \
    chown -R catalyst:catalyst /home/catalyst

# Allow catalyst user to install tools
RUN chmod -R 777 /usr/local/bin /usr/local/lib 2>/dev/null || true && \
    mkdir -p /opt && chmod 777 /opt

ENV PATH="/home/catalyst/.local/bin:/usr/local/bin:${PATH}"

USER catalyst
WORKDIR /home/catalyst

ENTRYPOINT ["/bin/bash"]

# OCI labels
LABEL org.opencontainers.image.title="Catalyst"
LABEL org.opencontainers.image.description="Ultra-light CI/CD runner with on-demand tool installer"
LABEL org.opencontainers.image.vendor="Quefly"
LABEL org.opencontainers.image.url="https://github.com/queflyhq/actions-toolkit"
LABEL org.opencontainers.image.source="https://github.com/queflyhq/actions-toolkit"
LABEL org.opencontainers.image.licenses="MIT"
LABEL org.opencontainers.image.base.name="alpine:3.21"
LABEL org.opencontainers.image.version="1.0.0"
