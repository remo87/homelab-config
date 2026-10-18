# immich — migration plan

## Overview

Immich is a self-hosted Google-Photos replacement. Heaviest workload in the homelab (~1.4 GiB RAM total) and the largest persistent footprint (200 GiB library). Four containers today: `server` (the API + UI), `machine_learning` (CLIP + facial recognition), `postgres` (with `pgvector` and `vectors` chord extensions), and `redis` (job queue). Migrates third — after cloudflared and firefly prove the cluster mechanics — because its data size and RAM pressure make it the highest-risk service.

## Current (Docker) state

- Compose file: `homelab-config/immich/docker-compose.yml` (uses `.env` for `IMMICH_VERSION`, DB credentials, etc.)
- Images (from §1.3 of PLAN.md):
  - `ghcr.io/immich-app/immich-server:v2.3.1`
  - `ghcr.io/immich-app/immich-machine-learning:v2.3.1`
  - `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0`
  - `valkey/valkey:8`
- Network: default `bridge`
- Persistence:
  - bind `/home/ricardo/immich/library:/data` (uploads + thumbnails)
  - bind `/home/ricardo/immich/postgres:/var/lib/postgresql/data`
  - named volume `model-cache` (CLIP model weights, ~5 GiB on first start)
- Ports: `2283:2283` (host → server)
- RAM: ~886 + 305 + 169 + 13 = **~1.4 GiB**

## Target (k8s)

Install via the official Immich Helm chart (`immich` from the Immich project, or TrueCharts). The chart bundles all four workloads and their dependencies.

### PVCs

| PVC | Size | MountPath (in pod) | Source (Docker → k8s) |
|---|---|---|---|
| `immich-library` | 200 GiB | `/usr/src/app/upload` | bind `/home/ricardo/immich/library` (already on host) |
| `immich-model-cache` | 20 GiB | `/cache` | empty — populates on first start (~5 GiB download) |
| `immich-pgdata` | 50 GiB | `/var/lib/postgresql/data` | restore from `pg_dumpall` (see below) |
| `immich-redis` (no PVC) | — | — | recreatable, ephemeral |

StorageClass: `microk8s-hostpath` (provided by the `storage` addon).

### HostPath prep (one-time, per [PLAN.md §4 Phase 2](./PLAN.md))

```bash
sudo mkdir -p /var/snap/microk8s/common/storage/default/immich-library
sudo mkdir -p /var/snap/microk8s/common/storage/default/immich-pgdata
sudo cp -a /home/ricardo/immich/library/. /var/snap/microk8s/common/storage/default/immich-library/
```

### Helm install

```bash
microk8s helm3 repo add immich https://immich-app.github.io/immich-charts/
microk8s helm3 repo update

microk8s helm3 install immich immich/immich \
  --namespace homelab --create-namespace \
  --set persistence.library.size=200Gi \
  --set persistence.modelCache.size=20Gi \
  --set persistence.database.size=50Gi \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host=immich.example.com \
  --set ingress.hosts[0].paths[0].path=/ \
  --set resources.server.requests.memory=512Mi \
  --set resources.server.limits.memory=1Gi \
  --set resources.machineLearning.requests.memory=256Mi \
  --set resources.machineLearning.limits.memory=512Mi
```

If preferring raw manifests, the four Deployments + three PVCs + Service can be hand-written. The chart is the lower-risk path for a learning plan.

### Service / Ingress

- ClusterIP Service on port 80, targetPort 2283.
- Ingress on `immich.example.com` → Service `immich-server` port 80.
- Reachable in-LAN at `http://immich.homelab.svc.cluster.local` or via the Ingress hostname.

### HPA (post-migration, per [PLAN.md §4 Phase 5](./PLAN.md))

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: immich-server
  namespace: homelab
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: immich-server
  minReplicas: 1
  maxReplicas: 3
  metrics:
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

> Single-node note: HPA is mostly useful as an alerting/visibility tool here — microk8s on a 4-core/8 GiB host cannot effectively schedule 3 instances of the server (each ~1 GiB). Keep `maxReplicas: 2` or document it as "headroom indicator, not scaling driver."

### NetworkPolicy (post-migration, per [PLAN.md §4 Phase 5](./PLAN.md))

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: immich-ingress
  namespace: homelab
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: immich
      app.kubernetes.io/component: server
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: cloudflared
      ports:
        - port: 2283
          protocol: TCP
    - from:
        - podSelector: {}   # in-namespace
      ports:
        - port: 2283
          protocol: TCP
```

## Migration steps

1. **Logical backup of Postgres:**
   ```bash
   docker exec immich_postgres pg_dumpall -U $DB_USERNAME > /home/ricardo/backups/<date>/immich-pg.dump
   ```
2. **Copy library to hostPath (one-time, can be done in advance):**
   ```bash
   sudo mkdir -p /var/snap/microk8s/common/storage/default/immich-library
   sudo rsync -a /home/ricardo/immich/library/ /var/snap/microk8s/common/storage/default/immich-library/
   ```
3. **Bring up k8s:**
   ```bash
   microk8s helm3 install immich immich/immich -n homelab ...
   ```
4. **Wait for the postgres pod to be ready, then restore:**
   ```bash
   microk8s kubectl -n homelab wait --for=condition=ready pod -l app.kubernetes.io/component=database
   microk8s kubectl -n homelab exec -it deploy/immich-database -- \
     psql -U $DB_USERNAME < /home/ricardo/backups/<date>/immich-pg.dump
   ```
5. **Restart the server so it re-reads restored schema and re-attaches library:**
   ```bash
   microk8s kubectl -n homelab rollout restart deploy/immich-server
   ```
6. **Validate thumbnails** (they're cached in the library PVC under `thumbs/`, so no regeneration needed for already-imported assets). New uploads generate thumbs on demand.
7. **Stop Docker:**
   ```bash
   docker compose -f homelab-config/immich/docker-compose.yml down
   ```

## Smoke test

- Web UI loads at `http://immich.example.com` (or via the Ingress).
- Login OK with the existing admin account.
- Thumbnails of existing library visible (proves library copy).
- Upload a new photo and verify it appears with a generated thumbnail (proves write path + ML inference).
- Container logs:
  ```bash
  microk8s kubectl -n homelab logs -f deploy/immich-server
  microk8s kubectl -n homelab logs -f deploy/immich-machine-learning
  microk8s kubectl -n homelab logs -f deploy/immich-database
  ```

## Rollback

```bash
docker compose -f homelab-config/immich/docker-compose.yml up -d
# Restore Postgres from the same dump
microk8s helm3 -n homelab uninstall immich
```

## Notes / risks

- **Data loss in Immich during migration** — Probability: Low, Impact: Critical. Mitigation: LVM snapshot (per [PLAN.md §4 Phase 0](./PLAN.md)) + `cp -a` / `rsync` of the library + `pg_dumpall` for the database before any switch.
- **Immich ML cache not migrated** — Probability: Medium, Impact: Low. The model-cache PVC starts empty; Immich re-downloads ~5 GiB of CLIP model weights on first start. Bandwidth hit, not a correctness issue.
- **RAM pressure** — Immich is the largest single workload (~1.4 GiB). Combined with the ~960 MiB microk8s baseline, total approaches the 5.3 GiB available RAM. Set explicit `resources.limits` and watch `kubectl top pods` for OOM kills during ML warm-up.
- **Valkey vs Redis** — Docker uses `valkey/valkey:8` (Redis fork). The Immich chart usually defaults to Redis or to bitnami Redis, which is API-compatible. Confirm the chart's value path before installing.
- **Version pinning** — pin `IMMICH_VERSION` to the same `v2.3.1` you were running to avoid surprise migrations.
- **Thumbs regenerate** — if library files are present but the thumbs directory is incomplete, Immich will regenerate on demand (CPU + IO spike on first browse after migration).
