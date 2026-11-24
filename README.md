# M7011E-GitOps — MapPick deployment

This repo describes everything that runs in the MapPick Kubernetes cluster (k3s). It is the single source of truth: [Argo CD](https://argo-cd.readthedocs.io/) watches the `main` branch and keeps the cluster matching it.

Application code lives in a separate repo, [M7011E-MapPicker](https://github.com/M7011E-CS2/M7011E-MapPicker). That repo's CI builds container images and pushes them to GHCR. It then commits the new image tag here, and Argo CD rolls it out.

- **Public host:** `https://cs2.group-3-m7011e-ltu.se`
- **Namespace:** `mappick` (Argo CD objects live in `argocd`)
- **Ingress controller:** Traefik (k3s default)
- **TLS:** cert-manager + Let's Encrypt

## Repository layout

```
argocd/
  root-app.yaml          App-of-apps. The only manifest applied by hand.
  apps/                  One Argo CD Application per file.
    frontend.yaml          -> charts/nest-service + values/frontend.yaml
    user-service.yaml      -> charts/nest-service + values/user-service.yaml
    infra.yaml             -> infra/ (plain manifests)
charts/
  nest-service/          Generic Helm chart shared by every service
                         (Deployment, Service, Ingress, Traefik Middleware).
values/
  <service>.yaml         Per-service overrides for the generic chart.
infra/
  certificate.yaml       Shared TLS certificate (secret cs2-tls) for the host.
  postgres.yaml          Shared PostgreSQL 17 StatefulSet + init script.
```

> Comments in some manifests say `deploy/...` (e.g. `deploy/values/frontend.yaml`). These files used to sit in a `deploy/` folder of the app repo. In this repo the same paths start at the root.

## How a deploy works

```
push to M7011E-MapPicker
  └─ CI (.github/workflows/build.yml) builds + pushes ghcr.io/m7011e-cs2/<service>:<commit SHA>
       └─ CI commits "deploy: image tag <sha>" here, setting image.tag in values/<service>.yaml (via yq)
            └─ Argo CD sees the change on main and syncs
                 └─ rolling update (maxUnavailable 0, maxSurge 1): new pod must pass readiness first
```

All Applications use automated sync with:

- `prune: true`: anything removed from git is deleted from the cluster.
- `selfHeal: true`: manual changes made with `kubectl edit` are reverted. **Change things here, not in the cluster.**
- Retries: up to 5 attempts with backoff (10s, doubling, capped at 3m).

**Rollback:** `git revert` the `deploy: image tag ...` commit, or set `image.tag` back to an earlier SHA, then push.

## Bootstrapping a cluster

Prerequisites: Argo CD installed in the `argocd` namespace, and cert-manager installed with a `ClusterIssuer` named `letsencrypt-staging` (and `letsencrypt-prod` for real certificates).

1. Create the secrets (see [Secrets](#secrets)). They are never stored in git.
2. Apply the root app once:

   ```sh
   kubectl apply -n argocd -f argocd/root-app.yaml
   ```

   The root app (`mappick-root`) reads `argocd/apps/` and creates the `frontend`, `user-service` and `infra` Applications. These then deploy everything else.

3. Watch progress:

   ```sh
   kubectl get applications -n argocd
   kubectl get pods -n mappick
   ```

The root app deliberately has **no** finalizer. Deleting it does not delete the services. Each child Application **does** have the `resources-finalizer`, so deleting a child Application also deletes its Kubernetes resources.

## Secrets

Secrets are created by hand in the `mappick` namespace and must exist before the pods that use them start.

| Secret            | Keys                       | Used by                                                       |
| ----------------- | -------------------------- | ------------------------------------------------------------- |
| `postgres-admin`  | `password`                 | Postgres superuser `postgres`                                 |
| `user-service-db` | `password`, `DATABASE_URL` | Postgres init script (`password`), user-service (`DATABASE_URL`) |
| `ghcr-pull` (optional) | docker-registry       | Only if the GHCR packages are private. Set `imagePullSecrets` |

```sh
kubectl create namespace mappick   # if it doesn't exist yet

kubectl -n mappick create secret generic postgres-admin \
  --from-literal=password='<admin password>'

kubectl -n mappick create secret generic user-service-db \
  --from-literal=password='<user_service password>' \
  --from-literal=DATABASE_URL='postgresql://user_service:<user_service password>@postgres.mappick.svc.cluster.local:5432/user_service'
```

URL-encode the password inside `DATABASE_URL` if it contains special characters.

## The generic chart: `charts/nest-service`

Each service is the same chart rendered with its own values file. The Helm release name (e.g. `user-service`) becomes the name of every resource. Defaults are in [charts/nest-service/values.yaml](charts/nest-service/values.yaml). The most important keys:

| Key                        | Default                       | Notes                                                                 |
| -------------------------- | ----------------------------- | --------------------------------------------------------------------- |
| `image.repository`         | —                             | **Required**, e.g. `ghcr.io/m7011e-cs2/user-service`                  |
| `image.tag`                | —                             | **Required**. Written by CI, do not edit by hand                      |
| `containerPort`            | `3000`                        | Also passed to the app as `PORT`                                      |
| `env`                      | `{}`                          | Plain env vars, `KEY: value`                                          |
| `secretEnv`                | `{}`                          | `KEY: { secretName, key }`, read from a Secret                          |
| `replicaCount`             | `1`                           |                                                                       |
| `service.port`             | `80`                          | ClusterIP Service, targets the `http` container port                  |
| `ingress.enabled`          | `false`                       |                                                                       |
| `ingress.host`             | `cs2.group-3-m7011e-ltu.se`   |                                                                       |
| `ingress.path`             | —                             | **Required** when ingress is enabled, e.g. `/api/users`               |
| `ingress.stripPrefix`      | `true`                        | Traefik strips the prefix: `/api/users/health` → `/health`            |
| `ingress.tls.secretName`   | `cs2-tls`                     | Shared certificate from `infra/certificate.yaml`                      |
| `ingress.tls.clusterIssuer`| `""`                          | Leave empty, because the shared Certificate requests the cert          |
| `probes.path`              | `/health`                     | Used by both liveness and readiness probes                            |
| `resources`                | 50m/128Mi req, 500m/256Mi lim |                                                                       |

Every pod is hardened by default. It runs as non-root (uid 1000, the `node` user), with a read-only root filesystem (a writable `/tmp` emptyDir is mounted), all capabilities dropped, the RuntimeDefault seccomp profile, and no service account token. Apps must be able to run under these limits.

Preview the rendered manifests locally:

```sh
helm template user-service charts/nest-service -f values/user-service.yaml -n mappick
```

### Current services

| Service        | Image                              | Route         | Port | Health check |
| -------------- | ---------------------------------- | ------------- | ---- | ------------ |
| `frontend`     | `ghcr.io/m7011e-cs2/frontend`      | `/`           | 8080 | `/`          |
| `user-service` | `ghcr.io/m7011e-cs2/user-service`  | `/api/users` (prefix stripped) | 3000 | `/health` |

## Shared infrastructure (`infra/`)

These are plain manifests applied by the `infra` Argo CD app.

### TLS certificate

[infra/certificate.yaml](infra/certificate.yaml) requests **one** certificate (`cs2-tls`) for the host. Every Ingress reuses it. It is not created per Ingress because Let's Encrypt allows only 5 duplicate certificates per week.

It currently uses `letsencrypt-staging`, so browsers will show an untrusted certificate. When staging works, switch `issuerRef.name` to `letsencrypt-prod`.

### PostgreSQL

[infra/postgres.yaml](infra/postgres.yaml) runs a single PostgreSQL 17 instance shared by all services. Each service gets its **own database and user**.

- In-cluster address: `postgres.mappick.svc.cluster.local:5432`
- Storage: 5Gi PVC `data-postgres-0` (k3s `local-path`). The PVC survives pod restarts and is **not** deleted with the StatefulSet.
- The `postgres-init` ConfigMap script creates the `user_service` user and database. It runs **only on first start with an empty volume**.
- The startup probe allows up to 5 minutes for initdb or crash recovery before liveness checks begin.
- **Known workaround:** the pod runs with `appArmorProfile: Unconfined`. On the node (Ubuntu 26.04, kernel 7.0), containerd's default AppArmor profile blocks signals between processes in the same container, and that makes Postgres crash-loop. Remove the workaround once the node is fixed.

## Common tasks

### Add a new service

1. In the app repo, make the service build an image to `ghcr.io/m7011e-cs2/<service>`. Add it to the CI tag-bump step as well.
2. Add `values/<service>.yaml`, using `values/user-service.yaml` as a template. Set `image.repository`, an initial `image.tag`, `ingress.path`, and any `env`/`secretEnv`. Don't use blank lines: `yq` strips them on each bump.
3. Copy `argocd/apps/user-service.yaml` to `argocd/apps/<service>.yaml`, then change `metadata.name`, `helm.releaseName` and `valueFiles`.
4. Push to `main`. The root app picks up the new Application automatically.

### Give a service its own database

1. Add a `CREATE USER` / `CREATE DATABASE` block and its password env var to `infra/postgres.yaml`. This only covers fresh volumes.
2. On the **running** server, create the user and database by hand, because the init script will not run again:

   ```sh
   kubectl -n mappick exec -it postgres-0 -- psql -U postgres
   ```

3. Create a `<service>-db` secret with `password` and `DATABASE_URL`, then reference it in the service's `secretEnv`.

### Remove a service

Delete its file in `argocd/apps/` and push. The root app prunes the Application, and the finalizer deletes its resources.

## Troubleshooting

```sh
kubectl get applications -n argocd                     # sync / health status
kubectl -n argocd describe application <name>          # sync errors
kubectl -n mappick get pods,ingress,certificate
kubectl -n mappick logs deploy/<service>
kubectl -n mappick describe certificate cs2-tls        # TLS issuance problems
```

- **Pod stuck in `CreateContainerConfigError`:** a referenced Secret or key is missing.
- **`ImagePullBackOff`:** the tag doesn't exist yet, or the GHCR package is private and needs `imagePullSecrets`.
- **Pod never becomes Ready:** the app doesn't answer on `probes.path`, or it tries to write outside `/tmp`.
