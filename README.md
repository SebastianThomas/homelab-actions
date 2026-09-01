# homelab-actions

Composite GitHub Actions for deploying apps to the homelab K3s cluster
([`homelab-infra`](https://github.com/SebastianThomas/homelab-infra)). Each app
lives in its own repo and deploys itself; these are the shared glue.

| Action | Does |
|---|---|
| [`headscale-connect`](headscale-connect/action.yml) | Join the Headscale tailnet on the runner (reach the tailnet-only API / MagicDNS). Ephemeral node, auto-logout. |
| [`kube-deploy`](kube-deploy/action.yml) | Write a kubeconfig from a namespace-scoped SA token, `kubectl apply` the app's manifests (with `${APP_IMAGE}` substitution), wait for the rollout. |

Nothing here is secret — every credential is passed in as an input from the
caller's GitHub Environment.

## Using it from a private repo

If this repo is **private**: Settings → Actions → General → **Access** →
"Accessible from repositories owned by the user SebastianThomas". Then any repo
you own can `uses:` it. (Public repo → nothing to configure.)

## Example — an app repo's `.github/workflows/release.yml`

```yaml
name: release
on:
  push:
    tags: ["v*"]        # deploy v1.2.3 by pushing tag v1.2.3
  workflow_dispatch:     # re-run against an old tag to roll back

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.ref_name }}

      - uses: SebastianThomas/homelab-actions/headscale-connect@v1
        with:
          auth-key:     ${{ secrets.TS_AUTHKEY }}
          login-server: ${{ secrets.HEADSCALE_URL }}
          ping:         kube-cp-01

      - uses: SebastianThomas/homelab-actions/kube-deploy@v1
        with:
          server:    ${{ secrets.KUBE_API }}
          ca:        ${{ secrets.KUBE_CA }}
          token:     ${{ secrets.KUBE_TOKEN }}
          namespace: myapp
          image:     ghcr.io/${{ github.repository }}:${{ github.ref_name }}
```

`deploy/` in the app repo holds `deployment.yaml` + `service.yaml` (see
[`homelab-infra/kubernetes/apps/README.md`](https://github.com/SebastianThomas/homelab-infra/blob/main/kubernetes/apps/README.md)).
`deployment.yaml` uses `image: ${APP_IMAGE}` as a placeholder.

### Secrets the app repo needs (GitHub Environment `production`)

| Secret | From |
|---|---|
| `KUBE_API` | `https://kube-cp-01.ts.homelab.sthomas.ch:6443` |
| `KUBE_CA` | `kubectl -n <ns> get secret deployer-token -o jsonpath='{.data.ca\.crt}'` |
| `KUBE_TOKEN` | `kubectl -n <ns> get secret deployer-token -o jsonpath='{.data.token}' \| base64 -d` |
| `HEADSCALE_URL` | `https://headscale.homelab.sthomas.ch` |
| `TS_AUTHKEY` | `headscale preauthkeys create --user <id> --reusable --ephemeral --expiration 100y` |

## Versioning

Tag `v1`, `v1.0.0`, … and move the `v1` major tag forward on compatible changes.
Consumers pin `@v1`.
