# Centralized CI/CD Module for GitHub

Shared reusable workflows, composite actions, and runner infrastructure for all microservices.

## Repository Structure

```
shared-devops/
│
├── .github/workflows/
│   └── shared.yml                # Orchestrator workflow (calls actions below)
│
├── actions/
│   ├── semantic-version/action.yml     # Auto version from git tags + commit messages
│   ├── docker-build/action.yml         # Build & push Docker image (DinD)
│   ├── kaniko-build/action.yml         # Build & push Docker image (daemonless)
│   ├── github-release/action.yml       # Git tag + changelog + GitHub Release
│   └── helm-deploy/action.yml          # Deploy to Kubernetes via Helm OCI chart
│
├── runner/
│   └── Dockerfile                      # Custom ARC runner image
│
└── README.md
```

## Usage

Each service repo needs only a thin caller workflow (~25 lines):

```yaml
# .github/workflows/docker-build.yml
name: Build, Release & Deploy

on:
  push:
    branches: [main]
    paths: ['src/**', 'pom.xml', 'Dockerfile', 'helm/**']
  workflow_dispatch:
    inputs:
      version_bump:
        type: choice
        options: [patch, minor, major]

jobs:
  deploy:
    uses: queflyhq/shared-devops/.github/workflows/shared.yml@main
    with:
      image_name: ayush-<service>-service
      helm_release_name: ayush-<service>
      k8s_namespace: ayush-<service>-service
    permissions:
      contents: write
      id-token: write
```

## Workflow Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `image_name` | Yes | — | Docker Hub image name |
| `helm_release_name` | Yes | — | Helm release name |
| `k8s_namespace` | Yes | — | Kubernetes namespace |
| `build_command` | No | `mvn clean package -DskipTests -B` | Build command |
| `skip_build` | No | `false` | Skip build (config-only services) |
| `build_method` | No | `docker` | `docker` (DinD sidecar) or `kaniko` (daemonless) |
| `registry_org` | No | `queflyhq` | Docker registry org/namespace |
| `helm_chart` | No | `oci://...queflyhq/ayush-service` | OCI Helm chart reference |
| `helm_chart_version` | No | `1.2.0` | Shared OCI Helm chart version |
| `version_bump` | No | auto-detect | `patch` / `minor` / `major` |
| `health_check_path` | No | — | Health endpoint (e.g. `/actuator/health`) |
| `health_check_port` | No | `8080` | Container port |
| `runs_on` | No | `k8s-runner` | GitHub Actions runner label |

## Build Methods

### Docker (default)
Uses Docker-in-Docker sidecar. Requires a DinD container alongside the runner. Faster builds with standard Docker caching.

```yaml
build_method: docker
```

### Kaniko
Daemonless builds — no Docker daemon required. Works in rootless/restricted environments. Supports layer caching via registry.

```yaml
build_method: kaniko
```

## Pipeline

```
Checkout → Version → Build → Vault → Container Build → Release → Deploy → Verify
```

| Stage | Action | Description |
|-------|--------|-------------|
| **Version** | `actions/semantic-version` | Semver from git tags + conventional commits |
| **Build** | inline | Maven/npm with dependency caching |
| **Vault** | `hashicorp/vault-action` | Docker Hub credentials via JWT/OIDC |
| **Container** | `actions/docker-build` or `actions/kaniko-build` | Build & push container image |
| **Release** | `actions/github-release` | Git tag + changelog + GitHub Release |
| **Deploy** | `actions/helm-deploy` | Helm upgrade + rollout verify + health check |

## Composite Actions

Each action can also be used independently in custom workflows:

```yaml
- uses: queflyhq/shared-devops/actions/semantic-version@main
- uses: queflyhq/shared-devops/actions/docker-build@main
- uses: queflyhq/shared-devops/actions/kaniko-build@main
- uses: queflyhq/shared-devops/actions/github-release@main
- uses: queflyhq/shared-devops/actions/helm-deploy@main
```
