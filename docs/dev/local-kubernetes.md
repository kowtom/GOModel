# Local Kubernetes Development

Develop GoModel against a local [kind](https://kind.sigs.k8s.io/) cluster using
[Skaffold](https://skaffold.dev/) for a build → deploy → watch loop. The app is
deployed through the same production [Helm chart](../../helm) with dev overrides,
so you exercise the real Kubernetes, storage and cache code paths locally.

## Prerequisites

Install these once:

| Tool | Purpose | Install |
| --- | --- | --- |
| [Docker](https://docs.docker.com/get-docker/) | Container runtime for kind + image builds | — |
| [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) | Local Kubernetes cluster | `go install sigs.k8s.io/kind@latest` |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | Cluster CLI | — |
| [Helm](https://helm.sh/docs/intro/install/) | Chart rendering | — |
| [Skaffold](https://skaffold.dev/docs/install/) | Build + deploy inner loop | — |

## Layout

```
deploy/local/
  kind-cluster.yaml   # kind cluster (maps NodePort 30080 -> host 8080)
  deps.yaml           # in-cluster Redis + PostgreSQL + MongoDB (ephemeral)
  values.yaml         # Helm dev overrides (stateless: Postgres + Redis)
skaffold.yaml         # build (Dockerfile.dev) + deploy (Helm)
Dockerfile.dev        # fast native-arch build image
```

The dependencies (Redis, PostgreSQL, MongoDB) are deployed once by `make kind-up`
and persist across app redeploys — Skaffold manages only the GoModel app. This
guarantees the datastores are ready before the app's first boot.

## Quick start

```bash
make kind-up     # create the kind cluster + deploy dependencies (one time)
make dev-k8s     # build, load into kind, deploy, watch for changes
```

Then, in another terminal:

```bash
curl localhost:8080/health          # {"status":"ok"}
curl localhost:8080/health/ready     # {"status":"ready", ...} once Postgres is reachable
open http://localhost:8080/admin/dashboard
```

Edit any Go file and Skaffold automatically rebuilds the image, reloads it into
kind, and rolls the pod. Press `Ctrl+C` to stop `skaffold dev` (it uninstalls the
app release; the dependencies stay up).

## Common commands

```bash
make deploy-k8s     # one-shot build + deploy (no watch)
make undeploy-k8s   # remove the app release (dependencies stay up)
make kind-down      # delete the whole cluster (removes dependencies too)
```

## Configuration

Dev overrides live in [`deploy/local/values.yaml`](../../deploy/local/values.yaml):

- **Stateless mode** backed by the in-cluster `postgres` and `redis` Services.
- `image.pullPolicy: IfNotPresent` — the image is built locally and loaded into
  kind (never pulled from a registry). Skaffold rewrites `image.repository`/`tag`.
- `secrets.masterKey: dev-master-key` — **dev only**, never reuse elsewhere.
- `secrets.data.OPENAI_API_KEY: sk-test` — a placeholder so a provider registers
  and the gateway can boot; replace it (or add other keys) for real upstream calls.
- Verbose logging (`LOG_LEVEL=debug`, text format) and audit logging enabled.

### Adding a provider API key

Replace the placeholder or add credentials under `secrets.data` in
`deploy/local/values.yaml`:

```yaml
secrets:
  data:
    OPENAI_API_KEY: "sk-..."
```

Skaffold redeploys on save. Avoid committing real keys — keep them in your local
working copy only.

### Switching to SQLite

For the lightest footprint, deploy in SQLite mode instead of Postgres/Redis:
set `persistence.enabled: true` and remove the `storage`/`cache` blocks from
`deploy/local/values.yaml`. The chart then runs a single-replica StatefulSet
with a PVC at `/app/data`.

## Access

The gateway is reachable at `http://localhost:8080` via the **kind NodePort
mapping**: the Service is a `NodePort` on `30080`, which
`deploy/local/kind-cluster.yaml` maps to host port `8080`. This works whether or
not `skaffold dev` is running (e.g. after `make deploy-k8s`), so no separate
port-forward is needed.

## Troubleshooting

- **`ErrImageNeverPull` / image not found** — ensure you launched via Skaffold
  (`make dev-k8s`/`deploy-k8s`) so the image is loaded into kind. `pullPolicy`
  must be `IfNotPresent` (already set in dev values).
- **App CrashLoopBackOff: `no providers were successfully registered`** — at
  least one `<PROVIDER>_API_KEY` must be set. The dev values ship a placeholder
  `OPENAI_API_KEY: sk-test`; keep it or add your own.
- **App CrashLoopBackOff: `failed to connect to redis` / DNS `server
  misbehaving`** — the dependencies aren't up yet. Run `make kind-up` (it applies
  `deploy/local/deps.yaml` and waits for rollout) before deploying the app.
- **Readiness stuck at 503** — the app returns `503` from `/health/ready` until
  primary storage (Postgres) is reachable. Check `kubectl get pods` and
  `kubectl logs deploy/postgres`.
- **MongoDB not ready** — the replica set self-initializes via the readiness
  probe on first boot; give it up to ~30s.
- **Port 8080 already in use** — stop the conflicting process, or change the
  `hostPort` in `deploy/local/kind-cluster.yaml` (then recreate the cluster).
