# Verdaccio — homelab npm mirror

In-cluster proxy/cache for `https://registry.npmjs.org`. Insulates
CI builds from the public registry's TLS path so transient
Cloudflare hiccups, Node-TLS bugs, and upstream changes can't
break the build.

## Why

GitHub-Actions self-hosted runners on the homelab K8s cluster
fetch every public dep over the open internet on every CI run.
That's fragile:

- Node 20.20.x has documented streaming-GCM failures against
  Cloudflare (which `registry.npmjs.org` sits behind), surfacing
  as `ERR_SSL_CIPHER_OPERATION_FAILED`.
- Pod recreation lands on different K8s nodes with different
  egress paths / MTU.
- Cloudflare quietly tweaks TLS config on the registry edge.

Verdaccio caches every package on first request. After warm-up,
builds never touch the public registry — they hit
`http://verdaccio.verdaccio.svc.cluster.local:4873` over plain
HTTP inside the cluster, which has none of those failure modes.

## Install / upgrade

From the homelab K8s box (`askmohanty@192.168.29.119`):

```bash
helm repo add verdaccio https://charts.verdaccio.org
helm repo update
helm upgrade --install verdaccio verdaccio/verdaccio \
  -n verdaccio --create-namespace \
  -f deploy/verdaccio/values.yaml
```

Wait for ready:

```bash
kubectl -n verdaccio rollout status deploy/verdaccio
```

Smoke-test from inside the cluster:

```bash
kubectl -n verdaccio port-forward svc/verdaccio 4873:4873 &
curl -s http://localhost:4873/-/ping  # → pong
curl -sI http://localhost:4873/lodash | head -3  # → 200 + content-type
```

## CI integration

Reusable workflows route npm through Verdaccio via `.npmrc`:

```ini
registry=http://verdaccio.verdaccio.svc.cluster.local:4873/
@queflyhq:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GIT_BOT_TOKEN}
```

Public packages → Verdaccio (which proxies + caches npmjs.org).
Private `@queflyhq/*` packages → GitHub Packages directly.

## Storage

10 GiB PVC on `local-path` storage class. Survives pod
restarts. To clear the cache:

```bash
kubectl -n verdaccio exec deploy/verdaccio -- rm -rf /verdaccio/storage/*
kubectl -n verdaccio rollout restart deploy/verdaccio
```

## Future work

- Switch from `local-path` to a network-replicated PV when we
  want the cache to survive node failure.
- Expose via the existing `nginx-gateway` if devs want to point
  their local npm at the mirror over Cloudflare tunnel.
- Pin upstream `npmjs.org` certs once we want full
  reproducibility against supply-chain compromise.
