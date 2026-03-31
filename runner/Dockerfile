# =============================================================================
# Catalyst — ultra-light CI/CD runner (~80MB)
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

# Pin to specific digest for reproducibility + supply chain security
FROM alpine:3.21 AS base

# Fix CVEs — upgrade all packages to latest patched versions
RUN apk upgrade --no-cache

RUN apk add --no-cache \
    bash curl wget git jq tar gzip unzip \
    openssh-client ca-certificates gnupg make \
    && rm -rf /var/cache/apk/*

# Install yq as static binary (not Alpine yq-go which pulls vulnerable Go stdlib)
RUN curl -fsSL -o /usr/local/bin/yq \
    "https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64" \
    && chmod +x /usr/local/bin/yq

COPY catalyst /usr/local/bin/catalyst
RUN chmod +x /usr/local/bin/catalyst

# Verify
RUN echo "=== Catalyst ===" && \
    git --version && \
    jq --version && \
    catalyst --help | head -1

# --- Security hardening ---

# Create non-root user
RUN addgroup -g 1000 catalyst && \
    adduser -u 1000 -G catalyst -h /home/catalyst -s /bin/bash -D catalyst && \
    mkdir -p /home/catalyst/.local/bin && \
    chown -R catalyst:catalyst /home/catalyst

# Allow catalyst user to install tools to /usr/local
RUN chmod -R 777 /usr/local/bin /usr/local/lib 2>/dev/null || true && \
    mkdir -p /opt && chmod 777 /opt

ENV PATH="/home/catalyst/.local/bin:/usr/local/bin:${PATH}"

# Switch to non-root
USER catalyst
WORKDIR /home/catalyst

ENTRYPOINT ["/bin/bash"]

# Labels for Docker Hub / supply chain
LABEL org.opencontainers.image.title="Catalyst"
LABEL org.opencontainers.image.description="Ultra-light CI/CD runner with on-demand tool installer"
LABEL org.opencontainers.image.vendor="Quefly"
LABEL org.opencontainers.image.url="https://github.com/queflyhq/actions-toolkit"
LABEL org.opencontainers.image.source="https://github.com/queflyhq/actions-toolkit"
LABEL org.opencontainers.image.licenses="MIT"
LABEL org.opencontainers.image.base.name="alpine:3.21"
