# firefly — migration plan

## Overview

Firefly III is a self-hosted personal-finance manager. Three containers today: `core` (the app), `db` (MariaDB), `cron` (rules / recurring transactions / budgets). Migrates early because it's the lightest service after cloudflared and exercises the Helm-chart path cheaply.

## Current (Docker) state

- Compose file: `homelab-config/firefly/docker-compose.yml` (+ `docker-compose-importer.yml` for one-off data imports)
- Images:
  - `fireflyiii/core:latest`
  - `mariadb:lts`
  - `alpine` (cron, runs `php artisan firefly-iii:cron`)
- Network: default `bridge`
- Persistence:
  - named volume `firefly_iii_upload` (uploaded files / attachments)
  - named volume `firefly_iii_db` (MariaDB data)
- Ports: `8082:8080` (host → core)
- RAM: ~104 MiB (core) + ~20 MiB (db) + <1 MiB (cron) = ~125 MiB

## Target (k8s)

Install via community Helm chart (TrueCharts or the unofficial `sneakybueno/firefly-iii`). The chart packages the three workloads and the MariaDB dependency.

### PVCs

| PVC | Size | MountPath | Source (Docker → k8s) |
|---|---|---|---|
| `firefly-upload` | 5 GiB | `/var/www/html/storage/upload` | copy from named volume `firefly_iii_upload` |
| `firefly-mariadb` | 5 GiB | `/var/lib/mysql` | restore from `mariadbdump` (see below) |

StorageClass: `microk8s-hostpath` (provided by the `storage` addon).

### HostPath prep (one-time, per [PLAN.md §4 Phase 2](./PLAN.md))

```bash
sudo mkdir -p /var/snap/microk8s/common/storage/default/firefly-upload
sudo mkdir -p /var/snap/microk8s/common/storage/default/firefly-mariadb
```

### Helm install

```bash
microk8s helm3 repo add truecharts https://truecharts.org/charts  # if using TrueCharts
# or: microk8s helm3 repo add sneakybueno https://sneakybueno.github.io/firefly-iii-helm
microk8s helm3 repo update

microk8s helm3 install firefly truecharts/firefly-iii \
  --namespace homelab --create-namespace \
  --set persistence.upload.size=5Gi \
  --set persistence.mariadb.size=5Gi \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host=firefly.example.com \
  --set ingress.hosts[0].paths[0].path=/ \
  --set service.type=ClusterIP \
  --set resources.requests.memory=128Mi \
  --set resources.limits.memory=256Mi
```

If preferring raw manifests over Helm, the three Deployments + PVCs + Service can be authored by hand (more boilerplate, more control). For a learning plan the chart is the lower-risk path.

### Service / Ingress

- ClusterIP Service on port 80, targetPort 8080.
- Ingress on `firefly.example.com` → Service `firefly-firefly-iii` (name depends on chart) port 80.
- Reachable in-LAN at `http://firefly.homelab.svc.cluster.local` or via the Ingress hostname.

## Migration steps

1. **Backup uploads (Docker volume → tarball):**
   ```bash
   docker run --rm -v firefly_iii_upload:/src -v $PWD:/dst alpine \
     tar czf /dst/firefly-upload.tgz -C /src .
   ```
2. **Logical backup of MariaDB:**
   ```bash
   docker exec firefly_iii_db mariadbdump --all-databases \
     -u root -p"$DB_ROOT_PASSWORD" > /home/ricardo/backups/<date>/firefly-mariadb.dump
   ```
3. **Bring up k8s:**
   ```bash
   microk8s helm3 install firefly truecharts/firefly-iii -n homelab ...
   ```
4. **Restore uploads:**
   ```bash
   microk8s kubectl -n homelab exec -it deploy/firefly-firefly-iii -- \
     tar xzf /dev/stdin -C /var/www/html/storage/upload < firefly-upload.tgz
   # (or use an initContainer that fetches from a side-car location)
   ```
5. **Restore MariaDB:**
   ```bash
   microk8s kubectl -n homelab exec -it deploy/firefly-mariadb-0 -- \
     mariadb -u root -p"$MARIADB_ROOT_PASSWORD" < /home/ricardo/backups/<date>/firefly-mariadb.dump
   ```
6. **Stop Docker:**
   ```bash
   docker compose -f homelab-config/firefly/docker-compose.yml down
   ```

## Smoke test

- Web UI loads at `http://firefly.example.com` (or `http://192.168.0.200` via MetalLB once Ingress is wired).
- Login OK with the existing admin account.
- Transaction list and recent activity visible (proves DB restore).
- Budget rule evaluates without error (proves cron is running).
- Container logs:
  ```bash
  microk8s kubectl -n homelab logs -f deploy/firefly-firefly-iii
  microk8s kubectl -n homelab logs -f deploy/firefly-mariadb-0
  microk8s kubectl -n homelab logs -f cronjob/firefly-firefly-iii-cron   # if chart ships a CronJob
  ```

## Rollback

```bash
docker compose -f homelab-config/firefly/docker-compose.yml up -d
# Restore uploads and DB into the Docker stack from the same backups
microk8s helm3 -n homelab uninstall firefly
```

## Notes / risks

- **Chart drift** — community charts can lag upstream Firefly releases; pin a chart version (`--version x.y.z`).
- **MariaDB credentials** — the chart auto-generates a root password. Capture it on first install (`microk8s kubectl -n homelab get secret firefly-mariadb -o jsonpath='{.data.root-password}' | base64 -d`) or override with `--set mariadb.auth.rootPassword=...`.
- **Cron schedule** — the chart usually ships a CronJob; verify the schedule matches the original `docker-compose.yml` cron timing.
- **Importer compose** — `docker-compose-importer.yml` is a one-shot tool, not part of the k8s deployment. Keep it in the repo for ad-hoc use.
