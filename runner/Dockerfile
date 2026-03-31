# =============================================================================
# Catalyst — universal CI/CD runner with every build tool pre-installed
# Public image: queflyhq/catalyst:latest
#
# Self-hosted (ARC on K8s):  runs-on: k8s-runner
# Container job:             container: queflyhq/catalyst:latest
# Local dev:                 docker run -it queflyhq/catalyst bash
#
# Includes: Go, Java 25, Maven, Node 20, Python 3, Docker CLI, Helm 3,
# kubectl, AWS CLI, gcloud, Terraform, Vault, ArgoCD, Kustomize, Kaniko,
# GitHub CLI, GitLab CLI, Terragrunt, yq, jq
# =============================================================================
FROM ghcr.io/actions/actions-runner:latest

USER root

ENV DEBIAN_FRONTEND=noninteractive

# ── Base packages ──
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl wget unzip ca-certificates gnupg lsb-release \
    jq tar gzip openssh-client gettext-base \
    && rm -rf /var/lib/apt/lists/*

# ── yq ──
RUN curl -fsSL -o /usr/local/bin/yq \
    "https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64" \
    && chmod +x /usr/local/bin/yq

# ── Java 25 (Eclipse Temurin) ──
ARG JAVA_VERSION=25
RUN curl -fsSL "https://api.adoptium.net/v3/binary/latest/${JAVA_VERSION}/ga/linux/x64/jdk/hotspot/normal/eclipse" \
    -o /tmp/jdk.tar.gz \
    && mkdir -p /usr/lib/jvm \
    && tar xzf /tmp/jdk.tar.gz -C /usr/lib/jvm \
    && rm /tmp/jdk.tar.gz \
    && ln -s /usr/lib/jvm/jdk-* /usr/lib/jvm/temurin
ENV JAVA_HOME=/usr/lib/jvm/temurin
ENV PATH="${JAVA_HOME}/bin:${PATH}"

# ── Maven 3.9 ──
ARG MAVEN_VERSION=3.9.9
RUN curl -fsSL https://archive.apache.org/dist/maven/maven-3/${MAVEN_VERSION}/binaries/apache-maven-${MAVEN_VERSION}-bin.tar.gz \
    | tar xz -C /opt \
    && ln -s /opt/apache-maven-${MAVEN_VERSION}/bin/mvn /usr/local/bin/mvn

# ── Node.js 20 LTS ──
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y --no-install-recommends nodejs \
    && rm -rf /var/lib/apt/lists/*

# ── Python 3.12 + pip ──
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 python3-pip python3-venv \
    && rm -rf /var/lib/apt/lists/* \
    && ln -sf /usr/bin/python3 /usr/bin/python

# ── Go ──
ARG GO_VERSION=1.23.6
RUN curl -fsSL "https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz" \
    | tar xz -C /usr/local
ENV PATH="/usr/local/go/bin:${PATH}"

# ── Docker CLI ──
RUN curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
        > /etc/apt/sources.list.d/docker.list \
    && apt-get update && apt-get install -y --no-install-recommends docker-ce-cli \
    && rm -rf /var/lib/apt/lists/*

# ── Helm 3 ──
RUN curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# ── kubectl ──
ARG KUBECTL_VERSION=1.35.0
RUN curl -fsSL -o /usr/local/bin/kubectl \
    "https://dl.k8s.io/release/v${KUBECTL_VERSION}/bin/linux/amd64/kubectl" \
    && chmod +x /usr/local/bin/kubectl

# ── AWS CLI v2 ──
RUN curl -fsSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip \
    && unzip -q /tmp/awscliv2.zip -d /tmp \
    && /tmp/aws/install \
    && rm -rf /tmp/aws /tmp/awscliv2.zip

# ── Google Cloud SDK ──
RUN curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg \
    | gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" \
        > /etc/apt/sources.list.d/google-cloud-sdk.list \
    && apt-get update && apt-get install -y --no-install-recommends google-cloud-cli \
    && rm -rf /var/lib/apt/lists/*

# ── Terraform + Vault CLI ──
RUN curl -fsSL https://apt.releases.hashicorp.com/gpg | gpg --dearmor -o /usr/share/keyrings/hashicorp.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
        > /etc/apt/sources.list.d/hashicorp.list \
    && apt-get update && apt-get install -y --no-install-recommends terraform vault \
    && rm -rf /var/lib/apt/lists/* \
    && setcap -r /usr/bin/vault 2>/dev/null || true

# ── Terragrunt ──
ARG TERRAGRUNT_VERSION=0.77.22
RUN curl -fsSL -o /usr/local/bin/terragrunt \
    "https://github.com/gruntwork-io/terragrunt/releases/download/v${TERRAGRUNT_VERSION}/terragrunt_linux_amd64" \
    && chmod +x /usr/local/bin/terragrunt

# ── ArgoCD CLI ──
ARG ARGOCD_VERSION=2.14.12
RUN curl -fsSL -o /usr/local/bin/argocd \
    "https://github.com/argoproj/argo-cd/releases/download/v${ARGOCD_VERSION}/argocd-linux-amd64" \
    && chmod +x /usr/local/bin/argocd

# ── Kustomize ──
ARG KUSTOMIZE_VERSION=5.6.0
RUN curl -fsSL "https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv${KUSTOMIZE_VERSION}/kustomize_v${KUSTOMIZE_VERSION}_linux_amd64.tar.gz" \
    | tar xz -C /usr/local/bin \
    && chmod +x /usr/local/bin/kustomize

# ── GitLab CLI (glab) ──
ARG GLAB_VERSION=1.52.0
RUN curl -fsSL "https://gitlab.com/gitlab-org/cli/-/releases/v${GLAB_VERSION}/downloads/glab_${GLAB_VERSION}_linux_amd64.tar.gz" \
    | tar xz -C /tmp \
    && mv /tmp/bin/glab /usr/local/bin/glab \
    && chmod +x /usr/local/bin/glab \
    && rm -rf /tmp/bin

# ── Kaniko ──
COPY --from=gcr.io/kaniko-project/executor:latest /kaniko/executor /kaniko/executor

# ── GitHub CLI ──
RUN curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
    | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
        > /etc/apt/sources.list.d/github-cli.list \
    && apt-get update && apt-get install -y --no-install-recommends gh \
    && rm -rf /var/lib/apt/lists/*

# ── Verify ──
RUN echo "=== Tool versions ===" \
    && java --version \
    && mvn --version | head -1 \
    && node --version \
    && npm --version \
    && python3 --version \
    && go version \
    && docker --version \
    && helm version --short \
    && kubectl version --client \
    && aws --version \
    && gcloud --version | head -1 \
    && terraform --version | head -1 \
    && vault --version \
    && terragrunt --version | head -1 \
    && argocd version --client --short \
    && kustomize version \
    && glab --version | head -1 \
    && envsubst --version | head -1 \
    && gh --version | head -1 \
    && git --version \
    && /kaniko/executor version 2>/dev/null || echo "kaniko installed"

# Switch back to runner user (from ARC base image)
USER runner
