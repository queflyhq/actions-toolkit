# Quefly Actions Toolkit

**Zero-config CI/CD framework for GitHub Actions.** Auto-detects your stack, runs tests, builds, scans, and deploys — anywhere.

Built on **Catalyst** (`queflyhq/catalyst`) — a universal runner image with every tool pre-installed.

---

## Quick Start

Add this to **any repo** — that's it:

```yaml
# .github/workflows/ci.yml
name: CI/CD
on: [push]

jobs:
  pipeline:
    uses: queflyhq/actions-toolkit/.github/workflows/auto-pipeline.yml@main
    with:
      deploy_target: cloudrun   # or k8s, cloudflare, aws-ecs, ssh, lambda, terraform, none
    secrets: inherit
```

**Auto-detects:** Go, Java, Node.js, Python, Ruby, PHP, .NET, Rust, SvelteKit, Next.js, Terraform/Terragrunt

**Auto-runs:** detect → test → version → build → scan → deploy

---

## Catalyst Runner

Universal CI/CD runner with every build tool pre-installed. No setup steps needed.

```bash
docker pull queflyhq/catalyst:latest
```

**Available tags:**

| Tag | Java | Go | Node | kubectl |
|-----|------|----|------|---------|
| `latest` | 25 | 1.25 | 20 | 1.35 |
| `java-21` | 21 | 1.23 | 20 | 1.31 |
| `java-17` | 17 | 1.22 | 20 | 1.30 |

**Included tools:** Go, Java, Maven, Node.js, Python, Docker CLI, Helm 3, kubectl, AWS CLI, gcloud, Terraform, Vault, Terragrunt, ArgoCD, Kustomize, Kaniko, GitHub CLI, GitLab CLI, yq, jq

**Auto-built** on Dockerfile change + weekly rebuild → pushed to Docker Hub + GHCR.

---

## Architecture

```
actions-toolkit/
├── .github/workflows/                ← Reusable pipelines
│   ├── auto-pipeline.yml             ← Zero-config: detects stack, does everything
│   ├── go-service.yml                ← Go: test → build → scan → deploy
│   ├── sveltekit-app.yml             ← SvelteKit: build → Cloudflare Pages
│   ├── sdk-publish.yml               ← SDK: test → multi-registry publish
│   └── catalyst-build.yml            ← Builds the runner image
│
├── actions/                           ← Composable building blocks
│   ├── auth/
│   │   └── cloud-identity/            ← GCP WIF, AWS OIDC, Azure Federated
│   │
│   ├── secrets/
│   │   ├── vault-secrets/             ← HashiCorp Vault
│   │   ├── aws-secrets/               ← AWS Secrets Manager
│   │   └── gcp-secrets/               ← GCP Secret Manager
│   │
│   ├── test/
│   │   ├── go-test/                   ← go test -race -cover
│   │   ├── node-test/                 ← npm test
│   │   ├── java-test/                 ← mvn test / gradle test
│   │   └── python-test/               ← pytest / unittest
│   │
│   ├── build/
│   │   ├── docker-build/              ← Docker build + push (DinD)
│   │   ├── kaniko-build/              ← Kaniko (daemonless)
│   │   └── sveltekit-build/           ← SvelteKit static build
│   │
│   ├── publish/
│   │   ├── npm-publish/               ← npm registry
│   │   ├── pypi-publish/              ← PyPI
│   │   ├── maven-publish/             ← Maven Central
│   │   ├── nuget-publish/             ← NuGet
│   │   ├── rubygems-publish/          ← RubyGems
│   │   ├── helm-publish/              ← Helm OCI
│   │   └── github-release/            ← Git tag + GitHub Release
│   │
│   ├── deploy/
│   │   ├── cloudrun-deploy/           ← GCP Cloud Run
│   │   ├── k8s-deploy/               ← kubectl / kustomize
│   │   ├── helm-deploy/              ← Helm upgrade
│   │   ├── cloudflare-deploy/         ← Cloudflare Pages / Workers
│   │   ├── aws-ecs-deploy/            ← AWS ECS Fargate
│   │   ├── lambda-deploy/             ← AWS Lambda (zip / image)
│   │   ├── terraform-deploy/          ← Terraform / Terragrunt
│   │   ├── ansible-deploy/            ← Ansible playbooks
│   │   └── ssh-deploy/               ← Any server via SSH
│   │
│   ├── security/
│   │   ├── trivy-scan/                ← Trivy CVE scan
│   │   └── docker-scout/             ← Docker Scout
│   │
│   └── version/
│       └── semantic-version/          ← Auto semver from commits
│
└── runner/
    └── Dockerfile                     ← Catalyst runner image
```

---

## Usage Patterns

### 1. Zero-config (auto-detect everything)

```yaml
jobs:
  ci:
    uses: queflyhq/actions-toolkit/.github/workflows/auto-pipeline.yml@main
    secrets: inherit
```

### 2. Pick a pre-built pipeline

```yaml
# Go microservice
jobs:
  ci:
    uses: queflyhq/actions-toolkit/.github/workflows/go-service.yml@main
    with:
      service_name: my-api
      deploy_target: k8s
    secrets: inherit
```

```yaml
# SvelteKit to Cloudflare
jobs:
  ci:
    uses: queflyhq/actions-toolkit/.github/workflows/sveltekit-app.yml@main
    with:
      project: my-website
    secrets: inherit
```

```yaml
# SDK publish on tag
jobs:
  publish:
    uses: queflyhq/actions-toolkit/.github/workflows/sdk-publish.yml@main
    with:
      language: node
      publish_npm: true
    secrets: inherit
```

### 3. Compose your own from building blocks

```yaml
jobs:
  deploy:
    runs-on: k8s-runner   # or: container: queflyhq/catalyst
    steps:
      - uses: actions/checkout@v4

      # Authenticate (no static keys)
      - uses: queflyhq/actions-toolkit/actions/auth/cloud-identity@main
        with:
          provider: gcp
          gcp_workload_identity_provider: projects/123/locations/global/...
          gcp_service_account: deploy@project.iam.gserviceaccount.com

      # Fetch secrets
      - uses: queflyhq/actions-toolkit/actions/secrets/gcp-secrets@main
        with:
          project: my-project
          secrets: |
            db-url DB_URL
            db-password DB_PASSWORD

      # Test
      - uses: queflyhq/actions-toolkit/actions/test/go-test@main

      # Version
      - uses: queflyhq/actions-toolkit/actions/version/semantic-version@main
        id: ver

      # Build
      - uses: queflyhq/actions-toolkit/actions/build/docker-build@main
        with:
          image_name: my-service
          image_tag: ${{ steps.ver.outputs.version }}

      # Deploy
      - uses: queflyhq/actions-toolkit/actions/deploy/k8s-deploy@main
        with:
          deployment: my-service
          image: queflyhq/my-service:${{ steps.ver.outputs.version }}
```

### 4. Infrastructure as Code

```yaml
jobs:
  infra:
    runs-on: k8s-runner
    steps:
      - uses: actions/checkout@v4
      - uses: queflyhq/actions-toolkit/actions/deploy/terraform-deploy@main
        with:
          working_directory: ./infra
          action: apply
          gcp_credentials: ${{ secrets.GCP_CREDENTIALS }}
```

### 5. On-prem / bare metal

```yaml
jobs:
  deploy:
    runs-on: k8s-runner
    steps:
      - uses: actions/checkout@v4
      - uses: queflyhq/actions-toolkit/actions/deploy/ansible-deploy@main
        with:
          playbook: deploy.yml
          inventory: hosts.ini
          ssh_key: ${{ secrets.SSH_KEY }}
          become: true
```

---

## Versioning

Auto-detected from conventional commits since last tag:

| Commit prefix | Bump |
|---------------|------|
| `feat:` | minor |
| `fix:`, `perf:`, `docs:`, `chore:` | patch |
| `feat!:`, `BREAKING CHANGE` | major |

---

## Preflight Checks

Every action verifies its required tools exist before running. If a tool is missing, the pipeline fails immediately with a clear error:

```
::error::Go is not installed. Use queflyhq/catalyst runner or install Go.
```

---

## Contributing

1. Add new actions under `actions/<category>/<name>/action.yml`
2. Every action must have a `Preflight` step
3. Use `$GITHUB_STEP_SUMMARY` for output
4. Keep actions generic — no org-specific defaults

## License

MIT
