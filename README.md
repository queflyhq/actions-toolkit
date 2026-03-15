# Centralized CI/CD Module for GitHub

Shared reusable workflows, composite actions, and runner infrastructure for all microservices.

## Repository Structure

```
shared-devops/
│
├── .github/workflows/
│   └── build-and-deploy.yml                # Orchestrator workflow (calls actions below)
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
    uses: queflyhq/shared-devops/.github/workflows/build-and-deploy.yml@main
    with:
      image_name: my-service
      helm_release_name: my-service
      k8s_namespace: my-service
      registry_org: my-dockerhub-org
      vault_addr: https://vault.example.com
      helm_chart: oci://registry-1.docker.io/my-org/my-chart
      helm_chart_version: '1.0.0'
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
| `skip_deploy` | No | `false` | Skip deploy (build-only) |
| `registry_org` | Yes | — | Docker registry org/namespace |
| `vault_addr` | Yes | — | HashiCorp Vault URL |
| `helm_chart` | Yes | — | OCI Helm chart reference |
| `helm_chart_version` | Yes | — | Helm chart version |
| `version_bump` | No | auto-detect | `patch` / `minor` / `major` |
| `health_check_path` | No | — | Health endpoint (e.g. `/actuator/health`) |
| `health_check_port` | No | `8080` | Container port |
| `runs_on` | No | `k8s-runner` | GitHub Actions runner label |

## Pipeline

```
┌─────────────────────────────────────────────┐     ┌──────────────────────────────────┐
│  BUILD JOB                                  │     │  DEPLOY JOB                      │
│  Checkout → Version → Compile → Docker Push │ ──► │  Vault → Helm Upgrade → Verify   │
│  → GitHub Release                           │     │  → Health Check                  │
└─────────────────────────────────────────────┘     └──────────────────────────────────┘
```

| Stage | Job | Description |
|-------|-----|-------------|
| **Version** | build | Semver from git tags + conventional commits |
| **Compile** | build | Maven/npm with dependency caching |
| **Docker** | build | Build & push container image (DinD) |
| **Release** | build | Git tag + changelog + GitHub Release |
| **Deploy** | deploy | Helm upgrade + rollout verify + health check |

## Composite Actions

Each action can also be used independently in custom workflows:

```yaml
- uses: queflyhq/shared-devops/actions/semantic-version@main
- uses: queflyhq/shared-devops/actions/docker-build@main
- uses: queflyhq/shared-devops/actions/kaniko-build@main
- uses: queflyhq/shared-devops/actions/github-release@main
- uses: queflyhq/shared-devops/actions/helm-deploy@main
```
