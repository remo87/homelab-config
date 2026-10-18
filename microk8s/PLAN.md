# Plan: Docker to microk8s migration (single-node)

> Status: **Plan / evaluation** — nothing has been executed yet.
> Target environment: homelab on `ubuntu-serv-1` (Ubuntu 24.04.3 LTS).
> Plan date: 2026-09-20.

## 0. Executive summary

**Verdict: feasible, with two caveats:**

1. **RAM is tight** (7.5 GiB total, ~5.3 GiB available for workloads). Docker containers must be stopped **before** bringing microk8s up to avoid OOM during the transition.
2. **Swap is active** (4 GiB). microk8s 1.31+ requires `--ignore-swap` or disabling it. The plan opts for disabling it (cleaner requirement).

Recommended strategy: **incremental migration per service**, starting with the simplest ones (cloudflared, Firefly III) and leaving Immich and Home Assistant for last (more complex).

## Per-application plans

- [cloudflared.md](./cloudflared.md) — Cloudflare Tunnel ingress (migrates first; validates the cluster)
- [firefly.md](./firefly.md) — Firefly III (3 pods: core, MariaDB, cron; Helm chart)
- [immich.md](./immich.md) — Immich (4 pods: server, ML, postgres, valkey; heaviest workload)
- [homeassistant.md](./homeassistant.md) — Home Assistant (special: `hostNetwork`, `hostPort: 8123`, USB radio)
- *Forgejo stays on Docker Compose* — compose at [`homelab-config/forgejo/compose.yaml`](../forgejo/compose.yaml); intentionally excluded from this migration.

---

## 1. Current environment state

### 1.1 Hardware / OS

| Resource | Value |
|---|---|
| Hostname | `ubuntu-serv-1` |
| OS | Ubuntu 24.04.3 LTS (Noble) |
| Kernel | 6.8.0-124-generic |
| CPU | Intel N100, 4 cores / 4 threads |
| Total RAM | 7.5 GiB |
| Available RAM | 4.6 GiB |
| Swap | 4 GiB (`/swap.img`) |
| Disk `/` | 100 GiB ext4 (LVM) |
| Disk `/home` | 815 GiB ext4 (LVM), 658 GiB free |
| `/dev/kvm` | available, rw for group `kvm` |
| cgroups | v2 (`cgroup2fs`) |
| snapd | 2.76.3 |
| microk8s | **not installed** |

### 1.2 Network

| Interface | State | Notes |
|---|---|---|
| `enp3s0` | UP | LAN `192.168.0.17/24`, gateway `192.168.0.1` |
| `wlp1s0` | DOWN | WiFi disabled |
| `docker0` | DOWN | default Docker bridge |
| 4× `br-*` | UP | Docker stack bridges (`172.18–172.21/16`) |
| Local DNS | `127.0.0.1` | (systemd-resolved / dnsmasq) |

### 1.3 Current Docker workloads

| Service | Image | RAM | Persistence | Ports |
|---|---|---|---|---|
| `immich_server` | `ghcr.io/immich-app/immich-server:v2.3.1` | 886 MiB | bind `/home/ricardo/immich/library:/data` | `2283` |
| `immich_machine_learning` | `ghcr.io/immich-app/immich-machine-learning:v2.3.1` | 305 MiB | named volume `model-cache` | — |
| `immich_postgres` | `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0` | 169 MiB | bind `/home/ricardo/immich/postgres:/var/lib/postgresql/data` | `5432` internal |
| `immich_redis` | `valkey/valkey:8` | 13 MiB | ephemeral | — |
| `forgejo` | `codeberg.org/forgejo/forgejo:14` | 68 MiB | bind `./forgejo:/data` | `3000`, `222` |
| `forgejo-db-1` | `postgres:14` | 31 MiB | bind `./postgres:/var/lib/postgresql/data` | `5432` internal |
| `cloudfered_tunnel` | `cloudflare/cloudflared:1800-17533b124c22` | 27 MiB | ephemeral | — |
| `homeassistant` | `ghcr.io/home-assistant/home-assistant:stable` | 557 MiB | bind `/home/ricardo/services/homeassistant/config:/config` + `/run/dbus` + `/dev/ttyUSB0` | `host` + privileged |
| `firefly_iii_core` | `fireflyiii/core:latest` | 104 MiB | named volume `firefly_iii_upload` | `8082` |
| `firefly_iii_db` | `mariadb:lts` | 20 MiB | named volume `firefly_iii_db` | `3306` internal |
| `firefly_iii_cron` | `alpine` | <1 MiB | ephemeral | — |
| **Total** | — | **~2.18 GiB** | — | — |

Compose files live at:

- `homelab-config/forgejo/compose.yaml`
- `/home/ricardo/services/homelab-config/{immich,firefly,homeassistant,cloudflare,nextcloud}/compose*.y*ml`

---

## 2. Target architecture

> *Forgejo stays on Docker Compose (compose file at `homelab-config/forgejo/compose.yaml`); it is intentionally excluded from this migration.*

```
                     ┌─────────────────────────────┐
                     │  MicroK8s (single-node)     │
                     │                             │
  Internet ───► Cloudflare Tunnel (Deployment)         │
                     │             │
                     │             ▼
                     │     ┌──────────────┐
                     │     │  MetalLB L2  │  pool: 192.168.0.250–254
                     │     └──────┬───────┘
                     │            ▼
                     │     ┌──────────────┐
                     │     │   Ingress    │  (microk8s nginx addon)
                     │     │   (nginx)    │  or Kgateway/Contour if Gateway API
                     │     └──────┬───────┘
                     │            │
   ┌────────────┬────┴────┬───────┴──────┬─────────────┐
   ▼            ▼         ▼              ▼             ▼
 Immich       Home Asst     Firefly III   cloudflared
 (3 pods)     (1 pod host)  (3 pods)      (1 pod)
   │            │         │              │
   └─hostPath/PVC (local-path) at /var/snap/microk8s/common/storage/...
```

### 2.1 API Gateway decision

You mentioned **MetalLB + API Gateway Fabric**. Note:

- **MetalLB** in L2: pool `192.168.0.200–192.168.0.210` (outside DHCP).
- **"API Gateway Fabric"** can mean **Kgateway** (Envoy Gateway) or **Contour**.

Initial recommendation: **MetalLB L2 + Nginx Ingress (microk8s `ingress` addon)**. Reasons:

- The `ingress` addon is maintained and battle-tested.
- Kgateway/Contour add CRDs and extra resources that don't bring much value for 5 services.
- You can migrate to Gateway API later without touching the workloads.

If you prefer **API Gateway Fabric from day 1**:

1. `microk8s enable metallb` with pool `192.168.0.200-192.168.0.210`.
2. Install Kgateway or Contour via Helm.
3. Create `Gateway` + `HTTPRoute` resources per service.
4. **Do not** enable the microk8s `ingress` addon (avoids collision).

---

## 3. Resources and sizing

### 3.1 Current consumption (baseline)

Current total: ~2.18 GiB.

Remaining for microk8s: ~5.3 GiB. Estimated microk8s consumption on single-node:

| Component | RAM |
|---|---|
| Kubelet + containerd | ~250 MiB |
| API-server + scheduler + controller-manager | ~300 MiB |
| etcd (single-node) | ~150 MiB |
| CoreDNS (`dns` addon) | ~50 MiB |
| Ingress nginx (`ingress` addon) | ~150 MiB |
| MetalLB (L2 speaker + controller) | ~50 MiB |
| Local-path storage helper | ~10 MiB |
| **microk8s total** | **~960 MiB** |

**Net headroom after migrating workloads: ~5.3 − 2.18 − 0.96 ≈ 2.2 GiB free**. Comfortable for Immich ML spikes.

### 3.2 Planned PVCs (conservative sizing)

| Service | PVC | Size | MountPath | Source |
|---|---|---|---|---|
| immich-server | `immich-library` | 200 GiB | `/usr/src/app/upload` | bind `/home/ricardo/immich/library` |
| immich-ml | `immich-model-cache` | 20 GiB | `/cache` | empty (populated on first start) |
| immich-postgres | `immich-pgdata` | 50 GiB | `/var/lib/postgresql/data` | bind `/home/ricardo/immich/postgres` |
| immich-redis | (no PVC) | — | — | recreatable |
| homeassistant | `ha-config` | 20 GiB | `/config` | bind `/home/ricardo/services/homeassistant/config` |
| firefly-core | `firefly-upload` | 5 GiB | `/var/www/html/storage/upload` | copy from named volume |
| firefly-db | `firefly-mariadb` | 5 GiB | `/var/lib/mysql` | copy from named volume |

StorageClass: `microk8s-hostpath` (provided by the `storage` addon).

---

## 4. Execution plan by phases

### Phase 0 — Pre-flight (~1 h, no downtime)

- LVM snapshot of the root volume as the main safety net:

  ```bash
  sudo lvcreate -s -L 20G -n homelab-snap /dev/ubuntu-vg/ubuntu-lv
  ```

- Additional data backup to `/home/ricardo/backups/<date>/`:

  ```bash
  rsync -a /home/ricardo/immich/ /home/ricardo/backups/<date>/immich/
  rsync -a /home/ricardo/services/forgejo/ /home/ricardo/backups/<date>/forgejo/
  rsync -a /home/ricardo/services/homeassistant/ /home/ricardo/backups/<date>/homeassistant/
  ```

- Logical backup of Immich and Forgejo Postgres:

  ```bash
  docker exec immich_postgres pg_dumpall -U $DB_USERNAME > /home/ricardo/backups/<date>/immich-pg.dump
  docker exec forgejo-db-1 pg_dumpall -U $DB_USERNAME > /home/ricardo/backups/<date>/forgejo-pg.dump
  docker exec firefly_iii_db mariadbdump --all-databases > /home/ricardo/backups/<date>/firefly-mariadb.dump
  ```

- Confirm KVM:

  ```bash
  lsmod | grep kvm
  ```

- Confirm current Cloudflare Tunnel connectivity: Cloudflare dashboard → tunnel `HEALTHY`.

### Phase 1 — Install microk8s (~30 min, no downtime)

- Disable swap temporarily and then persistently:

  ```bash
  sudo swapoff -a
  sudo sed -i 's/^\/swap.img/#\/swap.img/' /etc/fstab
  ```

- Install microk8s (stable channel):

  ```bash
  sudo snap install microk8s --classic --channel=1.35/stable  # ships v1.35.6
  sudo usermod -aG microk8s $USER
  mkdir -p ~/.kube
  newgrp microk8s
  ```

- Enable minimum add-ons:

  ```bash
  microk8s enable dns storage ingress metallb
  # when prompted for MetalLB: 192.168.0.200-192.168.0.210
  ```

- Validate:

  ```bash
  microk8s status --wait-ready
  microk8s kubectl get nodes -o wide
  microk8s kubectl get pods -A
  ```

### Phase 2 — Storage configuration (~1 h)

- Verify default storage class:

  ```bash
  microk8s kubectl get sc
  # should list microk8s-hostpath as (default)
  ```

- Create namespace `homelab`:

  ```bash
  microk8s kubectl create namespace homelab
  ```

- **Recommended data strategy:** direct copy to hostPath at `/var/snap/microk8s/common/storage/default/<pvc>/`. Simpler than dynamic PVCs at this scale.

  ```bash
  sudo mkdir -p /var/snap/microk8s/common/storage/default/immich-library
  sudo cp -a /home/ricardo/immich/library/. /var/snap/microk8s/common/storage/default/immich-library/
  # ... repeat for each PVC
  ```

  Permissions are adjusted with an `initContainer` or the pod's `fsGroup`.

### Phase 3 — Kubernetes manifests (~3–4 h, no downtime)

Recommended structure under `/home/ricardo/k8s-manifests/`:

```
k8s-manifests/
├── 00-namespace/
├── 01-cloudflared/        # first: validates the tunnel — see [cloudflared.md](./cloudflared.md)
├── 02-firefly/            # second: 3 simple pods — see [firefly.md](./firefly.md)
├── 03-immich/             # third: 3 pods + ML — see [immich.md](./immich.md)
└── 04-homeassistant/      # last: special case — see [homeassistant.md](./homeassistant.md)
```

**For Immich and Firefly use community Helm charts** instead of hand-written manifests:

- Immich: `helm install immich immunefi/immich -n homelab` (or TrueCharts).
- Firefly III: custom chart or TrueCharts.
- Home Assistant: hand-written manifest because of its special nature (see §5).
- cloudflared: simple Deployment with `TUNNEL_TOKEN` in a Secret.

### Phase 4 — Switch (estimated 30–60 min downtime)

Suggested order (one service at a time, smoke test before moving on):

1. [cloudflared](./cloudflared.md) — bring up the tunnel, then update Cloudflare dashboard routing.
2. [firefly](./firefly.md) — Helm install, restore MariaDB, switch hostname.
3. [immich](./immich.md) — Helm install, restore Postgres, re-attach library, restart server.
4. [homeassistant](./homeassistant.md) — apply Deployment, validate USB + Zigbee, reboot test.

#### 4.1 Smoke tests per service

| Service | Test | Per-app plan |
|---|---|---|
| cloudflared | Tunnel `HEALTHY` in Cloudflare dashboard, hit public URL → 200 | [cloudflared.md](./cloudflared.md) |
| Firefly III | Login OK, transaction list, budget rule evaluates | [firefly.md](./firefly.md) |
| Immich | Web UI loads at `http://192.168.0.200`, login OK, thumbnails of existing library visible | [immich.md](./immich.md) |
| Home Assistant | UI loads at `http://192.168.0.17:8123`, Zigbee / Z-Wave integrations respond | [homeassistant.md](./homeassistant.md) |

### Phase 5 — Post-migration hardening (~2 h)

- `resources.requests` and `resources.limits` per pod (especially Immich server + ML).
- `PodDisruptionBudget` where applicable.
- `NetworkPolicy` isolating the `homelab` namespace.
- Restrictive NetworkPolicy on Immich: only cloudflared talks to it.
- `HorizontalPodAutoscaler` for Immich server (watch RAM).
- Automated backup with **Velero + Restic** → backup to `/home/ricardo/backups/k8s/`.
- Monitoring (later phase): kube-prometheus-stack or kube-state-metrics.

### Phase 6 — Cleanup (~30 min, after 7 days of validation)

- `docker system prune -a --volumes` (only after confirming k8s is stable!).
- `sudo apt remove docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`.
- `sudo rm -rf /var/lib/docker`.
- Validate persistence after reboot: `sudo reboot` and check pods come up.

---

## 5. Special cases

Per-app plans already document their own special cases (e.g. [homeassistant.md](./homeassistant.md) for `hostNetwork` + `privileged` + USB; [immich.md](./immich.md) for ML cache and PVC sizing). No additional cross-cutting special cases apply.

---

## 6. Risks and mitigations

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| OOM due to insufficient RAM | Medium | High | Stop Docker before bringing k8s up; strict `resources.limits`; disable `dns` addon if too tight (saves 50 MiB) |
| microk8s complains about swap | High | Low | Disable swap in phase 1 (planned) |
| microk8s updates break the cluster (auto snap refresh) | Low | Medium | `sudo snap refresh --hold microk8s` during evaluation; use channel `1.35/stable` (ships v1.35.6, not edge) |
| K8s learning time exceeds the weekend | High | Low | Plan is for learning, no rush |
| Host port conflict between Docker and k8s | Low | Medium | Before phase 4, stop Docker. After phase 4, no k8s host ports remain (all via MetalLB/Ingress, except HA `hostPort`). |

---

## 7. Total estimate

| Phase | Time | Downtime |
|---|---|---|
| 0. Pre-flight | 1 h | 0 |
| 1. Install | 30 min | 0 |
| 2. Storage | 1 h | 0 |
| 3. Manifests | 3–4 h | 0 |
| 4. Switch (4 services) | 1–2 h | **~15 min per service** (rolling) |
| 5. Hardening | 2 h | 0 |
| 6. Cleanup | 30 min | 0 |
| **Total** | **~9–10 h** | **30–60 min aggregated** |

Realistic: **2–3 evenings of work**, given that this is a learning plan.

---

## 8. Recommended incremental roadmap

To reduce risk and validate incrementally:

1. **Day 1 (Phases 0–2)**: pre-flight + microk8s install + storage. Validate empty workloads (nginx welcome).
2. **Day 2 (Phase 3)**: manifests for `cloudflared` and `firefly`. Migrate both. Validate 24 h.
3. **Day 3 (Phase 3)**: manifests for `immich`. Migrate. Validate 24 h.
4. **Day 4 (Phase 3)**: manifests for `homeassistant`. Migrate carefully due to USB. Validate 48 h.
5. **Day 6+**: hardening, backups, Docker cleanup.

---

## 9. Quick reference commands

### 9.1 Cluster

```bash
microk8s status
microk8s kubectl get nodes
microk8s kubectl get pods -A
microk8s kubectl get svc -A
microk8s kubectl get ingress -A
microk8s kubectl logs -n homelab <pod>
microk8s kubectl exec -n homelab <pod> -- bash
```

### 9.2 Snap (updates)

```bash
snap info microk8s
sudo snap refresh --hold microk8s          # freeze updates
sudo snap refresh microk8s                  # update manually
sudo snap switch microk8s --channel=1.35/stable  # switch channel (ships v1.35.6)
```

### 9.3 Debug

```bash
microk8s kubectl describe pod -n homelab <pod>
microk8s kubectl get events -n homelab --sort-by='.lastTimestamp'
sudo crictl --config /var/snap/microk8s/current/args/crictl.yaml ps
journalctl -u snap.microk8s.daemon-kubelite -n 100
```

### 9.4 LVM snapshots

```bash
sudo lvcreate -s -L 20G -n homelab-snap /dev/ubuntu-vg/ubuntu-lv
sudo lvremove /dev/ubuntu-vg/homelab-snap    # when no longer needed
```

### 9.5 Full rollback

If anything goes wrong:

1. `sudo docker compose up -d` in each stack (while the LVM snap is alive, data is intact).
2. Or, worst case: `sudo lvconvert --merge /dev/ubuntu-vg/homelab-snap` and reboot.

---

## 10. Open decisions

- [ ] **Helm charts vs raw manifests**: use community charts (recommended) or write manifests from scratch.
- [ ] **API Gateway Fabric from day 1** or start with nginx ingress and migrate later.
- [ ] **NetworkPolicies**: namespace isolation from the start or later?
- [ ] **Automated backup**: Velero + Restic or a homegrown script with `kubectl get ... -o yaml` + rsync?
- [ ] **Monitoring**: install kube-prometheus-stack now or defer?
- [ ] **Nextcloud** (appears in `/home/ricardo/services/homelab-config/nextcloud/` but is not currently running): include it in the migration?

---

## 11. Next steps

When you decide to execute:

1. Confirm the open decisions in §10.
2. Run Phase 0 (pre-flight).
3. Start the install following §4.

---

> **Note**: this plan is based on a read-only inspection of the environment (Docker, networks, system, compose files). No host files have been modified and nothing has been installed.
