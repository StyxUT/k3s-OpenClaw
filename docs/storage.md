---
# Storage (PV/PVC) Notes

This repo mounts persistent state for `openclaw-gateway` at `/home/node/.openclaw`
using a Kubernetes PersistentVolumeClaim (PVC).

Key constraints:

- PersistentVolumes (PVs) are cluster-scoped and bind 1:1 to a PVC.
- PVCs are namespaced.
- If you deploy the workload in a new namespace, you need a new PV for the new
  PVC (even if it points at the same NFS export).

## Recommended: Static NFS PV + PVC

If you are using an existing NFS export (for example, the same export used by an
older PV like `synology-k8s-pv02`), the simplest and most predictable approach is
to create a new static PV and pre-bind it to the namespace-specific PVC.

### 1) Inspect the existing PV (source of truth)

```bash
kubectl get pv synology-k8s-pv02 -o yaml
```

Copy these values for the new PV:

- `spec.nfs.server`
- `spec.nfs.path`
- `spec.capacity.storage`
- `spec.accessModes`
- any `spec.mountOptions`

### 2) Create a new PV for the new namespace/PVC

Fill in `<NFS_SERVER>`, `<NFS_PATH>`, and `<SIZE>` from the PV you inspected.

This example pre-binds to `openclaw/synology-k8s-pv02-openclaw`:

```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: synology-k8s-pv02-openclaw
spec:
  capacity:
    storage: <SIZE>
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  persistentVolumeReclaimPolicy: Retain

  # Disable StorageClass so Kubernetes does not try dynamic provisioning.
  storageClassName: ""

  nfs:
    server: <NFS_SERVER>
    path: <NFS_PATH>

  # Pre-bind to the namespace claim (avoids accidental binding to another PVC).
  claimRef:
    namespace: openclaw
    name: synology-k8s-pv02-openclaw
```

Apply it:

```bash
kubectl apply -f pv-openclaw.yaml
```

### 3) Create (or recreate) the PVC in `openclaw`

Important: `storageClassName` effectively needs to be correct at creation time.
If your cluster has a default StorageClass, a PVC that omits `storageClassName`
may get defaulted and then will not match a static NFS PV.

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: synology-k8s-pv02-openclaw
  namespace: openclaw
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: <SIZE>

  # Must match the PV when using static binding.
  storageClassName: ""
  volumeName: synology-k8s-pv02-openclaw
```

Apply it:

```bash
kubectl apply -f pvc-openclaw.yaml
```

### 4) Verify binding and pod scheduling

```bash
kubectl get pv synology-k8s-pv02-openclaw
kubectl get pvc -n openclaw synology-k8s-pv02-openclaw
kubectl get pods -n openclaw -l app=openclaw-gateway -o wide
```

## Directory isolation within the PV

Even when multiple PVs point at the same NFS export, the workload should use a
dedicated subdirectory.

In this repo, the Deployment uses `subPath` on the volume mounts (for example,
`openclaw/data/.openclaw`) so the pod writes into a specific directory within
the NFS export.

## Troubleshooting

If the PVC stays Pending:

- Check if the PVC was defaulted to a StorageClass:
  - `kubectl get pvc -n openclaw synology-k8s-pv02-openclaw -o yaml`
  - Look for `spec.storageClassName: local-path` (or similar). For static NFS
    PVs, you typically want `storageClassName: ""` on both PV and PVC.
- Check for mismatches:
  - accessModes (e.g., `ReadWriteMany` vs `ReadWriteOnce`)
  - requested size larger than the PV capacity
  - PV already bound to another PVC
