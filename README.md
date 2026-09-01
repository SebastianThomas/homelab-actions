# homelab-actions

Composite GitHub Actions for deploying apps to the homelab K3s cluster
([`homelab-infra`](https://github.com/SebastianThomas/homelab-infra)). Each app
lives in its own repo and deploys itself; these are the shared glue.

| Action | Does |
|---|---|
| [`headscale-connect`](headscale-connect/action.yml) | Join the Headscale tailnet on the runner (reach the tailnet-only API / MagicDNS). Direct `tailscale up` (no SaaS-oriented wrapper). Ephemeral node, per-run name (`gha-<repo>-<run_id>-<attempt>`) so concurrent deploys don't collide; the ping gate fails fast (~2 min) instead of hanging. |
| [`kube-secret`](kube-secret/action.yml) | Create/update one opaque Secret in the app's namespace from `KEY=VALUE` lines (values from `${{ secrets.* }}`). Run before `kube-deploy` when the workload needs config secrets. |
| [`kube-deploy`](kube-deploy/action.yml) | Write a kubeconfig from the shared app-deployer token, `kubectl apply` the app's `deploy/` manifests (with `${APP_IMAGE}` substitution), wait for the rollout. |
| [`cnpg-cluster`](cnpg-cluster/action.yml) | Apply a CloudNativePG `Cluster` manifest (a `db/` dir) and wait for it to report Ready. Deliberately separate from `kube-deploy` — DB provisioning never rides along with an app deploy. |

Nothing here is secret — every credential is passed in as an input from the
caller's GitHub Environment.

## Visibility

**Keep this repo public.** Every credential is a caller-supplied input, so
there's nothing to hide, and `uses:` then works from any repo or org with no
setup. A *private* action repo owned by a user is only reachable from other
repos **owned by that same user** (Settings → Actions → Access → "Accessible
from repositories owned by …") — it will **not** resolve from a repo in an
org, which is why some app repos live in their own orgs and hit
"Unable to resolve action … not found".

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

      - name: Image ref            # ghcr.io needs a lowercase path
        run: echo "IMAGE=ghcr.io/${GITHUB_REPOSITORY,,}:${GITHUB_REF_NAME}" >> "$GITHUB_ENV"

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: ${{ env.IMAGE }}

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
          image:     ${{ env.IMAGE }}
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
