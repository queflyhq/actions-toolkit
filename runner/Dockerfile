# =============================================================================
# Catalyst — ultra-light CI/CD runner (~50MB)
# Public image: queflyhq/catalyst:latest
#
# Philosophy: Ship nothing. Install what you need, when you need it.
# Use the `catalyst` CLI to install tools on demand in your pipeline.
#
# Usage:
#   container: queflyhq/catalyst
#   steps:
#     - run: catalyst install go java node    # only what this job needs
#     - run: go test ./...
# =============================================================================
FROM alpine:3.21

RUN apk add --no-cache \
    bash curl wget git jq yq tar gzip unzip \
    openssh-client ca-certificates gnupg make

COPY catalyst /usr/local/bin/catalyst
RUN chmod +x /usr/local/bin/catalyst

RUN echo "=== Catalyst ===" && \
    git --version && \
    jq --version && \
    catalyst --help | head -1

ENTRYPOINT ["/bin/bash"]
