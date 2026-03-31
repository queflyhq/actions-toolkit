# Quefly Actions Toolkit

Generic CI/CD framework for GitHub Actions. Composable building blocks that work with any language, any registry, any deploy target.

## Architecture

```
actions-toolkit/
├── actions/                    ← Composite actions (building blocks)
│   ├── setup/                  ← Language & tool setup
│   │   ├── go/                 ← Set up Go + private modules
│   │   ├── node/               ← Set up Node.js + cache
│   │   ├── java/               ← Set up Java + Maven
│   │   └── python/             ← Set up Python + pip
│   │
│   ├── test/                   ← Test runners
│   │   ├── go-test/            ← Go test + coverage
│   │   ├── node-test/          ← npm test / vitest / jest
│   │   ├── java-test/          ← mvn test
│   │   └── python-test/        ← pytest
│   │
│   ├── build/                  ← Build artifacts
│   │   ├── docker-build/       ← Docker build + push (DinD)
│   │   ├── kaniko-build/       ← Kaniko build (daemonless)
│   │   └── sveltekit-build/    ← SvelteKit static build
│   │
│   ├── publish/                ← Package registry publish
│   │   ├── npm-publish/        ← npm registry
│   │   ├── pypi-publish/       ← PyPI
│   │   ├── maven-publish/      ← Maven Central
│   │   ├── nuget-publish/      ← NuGet
│   │   ├── rubygems-publish/   ← RubyGems
│   │   ├── helm-publish/       ← Helm OCI push
│   │   └── github-release/     ← Git tag + changelog + release
│   │
│   ├── deploy/                 ← Deploy targets
│   │   ├── cloudrun-deploy/    ← GCP Cloud Run
│   │   ├── k8s-deploy/         ← kubectl set image
│   │   ├── helm-deploy/        ← Helm upgrade
│   │   └── cloudflare-deploy/  ← Cloudflare Pages / Workers
│   │
│   ├── security/               ← Security scanning
│   │   ├── docker-scout/       ← Docker Scout CVE scan
│   │   └── trivy-scan/         ← Trivy vulnerability scan
│   │
│   └── version/                ← Versioning
│       └── semantic-version/   ← Semver from git tags + commits
│
├── pipelines/                  ← Pre-built pipelines (combine actions)
│   ├── go-service.yml          ← test → build → push → deploy
│   ├── sveltekit-app.yml       ← build → deploy to Cloudflare
│   ├── sdk-publish.yml         ← test → publish to registry
│   └── helm-release.yml        ← package → push OCI → deploy
│
├── runner/
│   └── Dockerfile              ← Custom ARC runner with all tools
│
└── README.md
```

## Usage

### Option 1: Use a pre-built pipeline (simplest)

```yaml
# In your service repo: .github/workflows/ci.yml
jobs:
  ci:
    uses: queflyhq/actions-toolkit/.github/workflows/go-service.yml@main
    with:
      service_name: my-service
      deploy_target: cloudrun
    secrets: inherit
```

### Option 2: Compose your own pipeline from building blocks

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: queflyhq/actions-toolkit/actions/test/go-test@main

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: queflyhq/actions-toolkit/actions/version/semantic-version@main
        id: version
      - uses: queflyhq/actions-toolkit/actions/build/docker-build@main
        with:
          image_name: my-service
          image_tag: ${{ steps.version.outputs.version }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: queflyhq/actions-toolkit/actions/deploy/cloudrun-deploy@main
        with:
          service: my-service
          image: queflyhq/my-service:${{ needs.build.outputs.image_tag }}
```

### Option 3: SDK multi-publish

```yaml
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: queflyhq/actions-toolkit/actions/publish/npm-publish@main
        with:
          token: ${{ secrets.NPM_TOKEN }}
      - uses: queflyhq/actions-toolkit/actions/publish/github-release@main
```

## Pre-built Pipelines

| Pipeline | For | What it does |
|----------|-----|-------------|
| `go-service.yml` | Go microservices | test → version → docker build → push → deploy (Cloud Run / K8s) |
| `sveltekit-app.yml` | SvelteKit apps | install → build → deploy to Cloudflare Pages |
| `sdk-publish.yml` | Any SDK | test → publish to package registry on tag |
| `helm-release.yml` | Helm charts | package → push OCI → deploy |

## Composite Actions

Every action is independent. Mix and match:

| Category | Action | Description |
|----------|--------|-------------|
| **Setup** | `setup/go` | Go + GOPRIVATE + module cache |
| | `setup/node` | Node.js + npm cache |
| | `setup/java` | Java + Maven cache |
| | `setup/python` | Python + pip cache |
| **Test** | `test/go-test` | `go test -race -cover` |
| | `test/node-test` | `npm test` |
| | `test/java-test` | `mvn test` |
| | `test/python-test` | `pytest` |
| **Build** | `build/docker-build` | Docker build + push |
| | `build/kaniko-build` | Kaniko daemonless build |
| | `build/sveltekit-build` | SvelteKit static build |
| **Publish** | `publish/npm-publish` | Publish to npm |
| | `publish/pypi-publish` | Publish to PyPI |
| | `publish/maven-publish` | Publish to Maven Central |
| | `publish/nuget-publish` | Publish to NuGet |
| | `publish/rubygems-publish` | Publish to RubyGems |
| | `publish/helm-publish` | Push Helm chart as OCI |
| | `publish/github-release` | Git tag + GitHub Release |
| **Deploy** | `deploy/cloudrun-deploy` | GCP Cloud Run |
| | `deploy/k8s-deploy` | kubectl rolling update |
| | `deploy/helm-deploy` | Helm upgrade |
| | `deploy/cloudflare-deploy` | Cloudflare Pages / Workers |
| **Security** | `security/docker-scout` | Docker Scout CVE scan |
| | `security/trivy-scan` | Trivy scan |
| **Version** | `version/semantic-version` | Semver from git history |

## Custom Runner

`runner/Dockerfile` — extends the official ARC runner with:
Go, Java 25, Maven, Node.js 20, Python 3, Docker CLI, Helm, kubectl, AWS CLI, gcloud, Terraform, Vault, ArgoCD, Kustomize, Kaniko, GitHub CLI.

```bash
docker build -t queflyhq/devops-runner:latest runner/
```
