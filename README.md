# Centralized CI/CD Module for GitHub

Shared reusable workflows, runner infrastructure, and deployment automation for all microservices.

## Repository Structure

```
shared-devops/
│
├── .github/
│   └── workflows/
│       └── build-deploy.yml            # Reusable workflow (called by all service repos)
│
├── runner/
│   └── Dockerfile                      # Custom ARC runner image
│
├── arc/
│   ├── values-ayush.yaml               # ARC config for AyushHealthcare org
│   └── values-quefly.yaml              # ARC config for queflyhq org
│
└── README.md
```

## Usage

Each service repo needs only a thin caller workflow:

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
    uses: queflyhq/shared-devops/.github/workflows/build-deploy.yml@main
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
| `helm_chart_version` | No | `1.2.0` | Shared OCI Helm chart version |
| `version_bump` | No | auto-detect | `patch` / `minor` / `major` |
| `health_check_path` | No | — | Health endpoint (e.g. `/actuator/health`) |
| `health_check_port` | No | `8080` | Container port |

## Pipeline Stages

```
Version → Build → Vault → Kaniko → Release → Deploy → Verify
```

1. **Version** — Semantic versioning from git tags + commit message conventions
2. **Build** — Application build with Maven/npm dependency caching
3. **Vault** — Docker Hub credentials via HashiCorp Vault (JWT/OIDC auth)
4. **Kaniko** — Build & push container image (no Docker daemon)
5. **Release** — Git tag + GitHub Release with auto-generated changelog
6. **Deploy** — Helm upgrade to Kubernetes using shared OCI chart
7. **Verify** — Rollout status check + optional health check
