# cloudflared — migration plan

## Overview

Cloudflare Tunnel client that fronts the public services hosted on this homelab. Outbound-only: it opens a long-lived connection to Cloudflare's edge and exposes services by hostname, no inbound ports required on the LAN. Migrates first because validating the tunnel in microk8s is the cheapest end-to-end smoke test of the whole cluster.

## Current (Docker) state

- Compose file: `homelab-config/cloudflare/compose.yaml`
- Image: `cloudflare/cloudflared:1800-17533b124c22`
- Network: `tunneled` (internal bridge shared with services the tunnel routes to)
- Persistence: **none** (stateless)
- Ports: **none exposed** on host
- RAM: ~27 MiB

## Target (k8s)

Single-namespace Deployment, no Service, no Ingress, no PVC. Outbound-only connectivity to `api.trycloudflare.com` and Cloudflare edge.

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflared
  namespace: homelab
type: Opaque
stringData:
  TUNNEL_TOKEN: "<REDACTED — set via: microk8s kubectl create secret generic cloudflared -n homelab --from-literal=TUNNEL_TOKEN=..."
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: homelab
  labels:
    app: cloudflared
spec:
  replicas: 1
  strategy:
    type: Recreate   # only one tunnel at a time
  selector:
    matchLabels:
      app: cloudflared
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:1800-17533b124c22
          imagePullPolicy: IfNotPresent
          command: ["cloudflared", "tunnel", "--no-autoupdate", "run"]
          envFrom:
            - secretRef:
                name: cloudflared
          resources:
            requests:
              cpu: 10m
              memory: 24Mi
            limits:
              memory: 64Mi
          livenessProbe:
            exec:
              command: ["cloudflared", "tunnel", "info"]
            initialDelaySeconds: 30
            periodSeconds: 60
            timeoutSeconds: 10
          readinessProbe:
            exec:
              command: ["cloudflared", "tunnel", "info"]
            initialDelaySeconds: 10
            periodSeconds: 30
      restartPolicy: Always
```

### Service

None. The tunnel is outbound; traffic flows from Cloudflare edge → tunnel pod → in-cluster Service of the target app (see [firefly.md](./firefly.md), [immich.md](./immich.md)). Routing rules are configured in the Cloudflare dashboard, not in k8s.

## Migration steps

1. Get the tunnel token from the Cloudflare Zero Trust dashboard (or `cloudflared tunnel token <name>` if recreating).
2. Create the Secret in-cluster:
   ```bash
   microk8s kubectl create namespace homelab   # if not already created
   microk8s kubectl -n homelab create secret generic cloudflared \
     --from-literal=TUNNEL_TOKEN=<token>
   ```
3. Apply the Deployment:
   ```bash
   microk8s kubectl apply -f k8s-manifests/01-cloudflared/
   ```
4. Validate: `microk8s kubectl -n homelab logs -f deploy/cloudflared` should show `Registered tunnel connection` and `HEALTHY` events.
5. **Switch routing**: in the Cloudflare dashboard, change the public hostname records (e.g. `git.example.com`, `immich.example.com`, `firefly.example.com`) from `127.0.0.1:<host-port>` to the in-cluster Service ClusterIP or hostname. For a single-node setup, the simplest is to keep pointing at the LAN IP of the host with the new in-cluster ports — but the canonical fix is to use the Service DNS name (e.g. `immich.homelab.svc.cluster.local:80`) once the apps are migrated.
6. Stop the Docker stack once stable: `docker compose -f homelab-config/cloudflare/compose.yaml down`.

## Smoke test

```bash
# 1. Tunnel pod is running and registered
microk8s kubectl -n homelab get pods -l app=cloudflared
microk8s kubectl -n homelab logs deploy/cloudflared | grep -E "Registered|HEALTHY"

# 2. Cloudflare dashboard shows the connector HEALTHY
# 3. Hit a public hostname routed through the tunnel
curl -I https://<your-public-hostname>
```

## Rollback

```bash
docker compose -f homelab-config/cloudflare/compose.yaml up -d
microk8s kubectl -n homelab delete deploy cloudflared
# Revert Cloudflare dashboard routing back to 127.0.0.1:<host-port>
```

## Notes / risks

- **Cloudflare Tunnel pointing at a dead IP** — Probability: Low, Impact: High. After migrating any backend service that the tunnel fronts, the public hostname must be re-pointed in the Cloudflare dashboard **before** the Docker stack goes down. Mitigation: change DNS / tunnel ingress only after the in-cluster service is responding to smoke tests.
- **Token rotation** — the Secret contains a long-lived token. Rotation requires recreating the Secret and restarting the pod. Document the rotation in `homelab-config/cloudflare/`.
- **No host port, no PVC** — single-node failure is total outage; consider a `PodDisruptionBudget` only if the cluster grows.
