# homeassistant — migration plan

## Overview

Home Assistant is the home-automation hub. Unlike every other workload in this homelab it is **not** fronted by an Ingress or MetalLB: HA needs raw access to the LAN to do mDNS / zeroconf discovery and to talk to USB-attached radios (Zigbee / Z-Wave). It also runs as `privileged: true` for `/dev/ttyUSB*` and Bluetooth. Migrates last because the validation window is the longest (Zigbee / Z-Wave stick re-enumeration after a reboot is finicky).

## Current (Docker) state

- Compose file: `homelab-config/homeassistant/compose.yaml`
- Image: `ghcr.io/home-assistant/home-assistant:stable`
- Network: `host` (not bridged — needs LAN access for mDNS / zeroconf)
- Persistence:
  - bind `/home/ricardo/services/homeassistant/config:/config`
- Privileges: `privileged: true`
- Devices:
  - `/dev/ttyUSB0` (Zigbee coordinator, e.g. ConBee II or ZBT-1)
  - `/run/dbus` (host dbus socket for Bluetooth)
- Ports: `8123:8123` (host → container)
- RAM: ~557 MiB

## Target (k8s)

Hand-written manifest (no community chart handles `hostNetwork` + `privileged` + USB device path well enough for a single-node learning plan to be worth the friction).

### PVC

| PVC | Size | MountPath | Source (Docker → k8s) |
|---|---|---|---|
| `ha-config` | 20 GiB | `/config` | bind `/home/ricardo/services/homeassistant/config` |

StorageClass: `microk8s-hostpath` (provided by the `storage` addon).

### HostPath prep (one-time, per [PLAN.md §4 Phase 2](./PLAN.md))

```bash
sudo mkdir -p /var/snap/microk8s/common/storage/default/ha-config
sudo rsync -a /home/ricardo/services/homeassistant/config/ /var/snap/microk8s/common/storage/default/ha-config/
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: homeassistant
  namespace: homelab
  labels:
    app: homeassistant
spec:
  replicas: 1
  strategy:
    type: Recreate   # privileged + hostNetwork — only one at a time
  selector:
    matchLabels:
      app: homeassistant
  template:
    metadata:
      labels:
        app: homeassistant
    spec:
      hostNetwork: true
      containers:
        - name: homeassistant
          image: ghcr.io/home-assistant/home-assistant:stable
          imagePullPolicy: IfNotPresent
          securityContext:
            privileged: true
          ports:
            - name: http
              hostPort: 8123
              containerPort: 8123
              protocol: TCP
          volumeMounts:
            - name: config
              mountPath: /config
            - name: dbus
              mountPath: /run/dbus
            - name: ttyusb
              mountPath: /dev/ttyUSB0
          resources:
            requests:
              cpu: 100m
              memory: 384Mi
            limits:
              memory: 768Mi
          livenessProbe:
            httpGet:
              path: /
              port: 8123
              host: 127.0.0.1
            initialDelaySeconds: 60
            periodSeconds: 30
            timeoutSeconds: 10
            failureThreshold: 6
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: ha-config
        - name: dbus
          hostPath:
            path: /run/dbus
            type: Directory
        - name: ttyusb
          hostPath:
            path: /dev/ttyUSB0
            type: CharDevice
```

### Service

None. `hostNetwork: true` + `hostPort: 8123` make the pod reachable directly at `http://192.168.0.17:8123`. A ClusterIP Service would be redundant and confusing (mDNS doesn't respect ClusterIPs).

### Stable USB device path (udev rule)

`/dev/ttyUSB0` is **not** guaranteed to be the same device across reboots. Add a udev rule on the host so the Zigbee coordinator always lands on a stable symlink, and update the volume mount to use that path.

```bash
# /etc/udev/rules.d/99-zigbee.rules
# Adjust ATTRS{idVendor}/ATTRS{idProduct} to match your coordinator.
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{idProduct}=="55d4", SYMLINK+="zigbee"
```

Then change the volume mount in the manifest to `/dev/zigbee` (type: `CharDevice`). After `sudo udevadm control --reload && sudo udevadm trigger`, the symlink will be stable across reboots.

## Migration steps

1. **Backup the HA config directory:**
   ```bash
   rsync -a /home/ricardo/services/homeassistant/ /home/ricardo/backups/<date>/homeassistant/
   ```
2. **Identify the USB device path** (verify it's still `/dev/ttyUSB0`):
   ```bash
   ls -l /dev/ttyUSB*
   ls -l /dev/serial/by-id/   # the by-id path is the most stable
   ```
3. **Apply the udev rule** (optional but recommended) and confirm the symlink exists.
4. **Bring up k8s:**
   ```bash
   microk8s kubectl apply -f k8s-manifests/04-homeassistant/
   ```
5. **Validate the pod has the USB device visible inside:**
   ```bash
   microk8s kubectl -n homelab exec -it deploy/homeassistant -- ls -l /dev/ttyUSB0
   # (or /dev/zigbee if using the udev symlink)
   ```
6. **Stop Docker:**
   ```bash
   docker compose -f homelab-config/homeassistant/compose.yaml down
   ```
7. **Reboot validation** (per [PLAN.md §4 Phase 6](./PLAN.md)): `sudo reboot` and confirm HA comes up with the Zigbee / Z-Wave integrations intact.

## Smoke test

- Web UI loads at `http://192.168.0.17:8123`.
- Login OK with the existing account.
- **Zigbee integration responds** — open Developer Tools → Services, call `zigbee2mqtt.ping` (or whichever integration backs the radio). The radio should answer.
- **Z-Wave integration responds** — same pattern, if applicable.
- **Bluetooth** (if used) — confirm a Bluetooth device is visible from HA.
- Container logs:
  ```bash
  microk8s kubectl -n homelab logs -f deploy/homeassistant
  ```

## Rollback

```bash
docker compose -f homelab-config/homeassistant/compose.yaml up -d
microk8s kubectl -n homelab delete deploy homeassistant
```

## Notes / risks

- **HA does not detect Zigbee after migration** — Probability: Medium, Impact: Medium. The most common cause is a changed device path (`/dev/ttyUSB0` → `/dev/ttyUSB1` after reboot) or a permission regression. Mitigation: document the device path before the switch, use a udev rule with a stable symlink (`/dev/zigbee`), and re-add the integration in HA if needed. Validate 48 h before declaring success (longest validation window in this migration).
- **Privileged container** — `securityContext.privileged: true` is required for Bluetooth and most USB radios. Acceptable for a single-node homelab; revisit if the cluster ever becomes multi-tenant.
- **No Ingress / no MetalLB** — by design. MetalLB + Ingress would break mDNS / zeroconf which HA depends on for local device discovery.
- **`hostNetwork` + `hostPort`** — the only port on the cluster that is exposed directly on the host LAN. All other services live behind MetalLB.
- **Config drift** — the HA config directory is mutable and HA writes to it on every change. After migration, `kubectl edit` of the Deployment will not touch the config; `rsync`-back from the hostPath PVC if you ever need to move it back to a stand-alone container.
- **RAM** — ~557 MiB today. With the limit set to 768 MiB, the cluster has ~1.3 GiB headroom for the remaining services. Comfortable but not generous.
