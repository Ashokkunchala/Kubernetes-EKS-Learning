# 🧪 Lab 02: From Stateless to Stateful in 30 Minutes

Storage is where most beginners' mental models break: pod death ≠ data loss, but also pod death ≠ data survival. This lab proves exactly when data survives and when it doesn't.

## 🎯 Lab Goal

In 30 minutes, prove the storage hierarchy on a live cluster:

1. `emptyDir` — data **dies** with the pod (stateless)
2. PVC + PV — data **survives** pod replacement (stateful-enough)
3. StatefulSet + `volumeClaimTemplates` — data survives **with stable identity** and per-pod volumes

## 🧪 Scenario

```
  ┌──────────────────────────────────────────────────────────────┐
  │        STATELESS ──► STATEFUL, THREE PROOFS                  │
  │                                                              │
  │  ① emptyDir (volatile)         ② PVC (survives pod)          │
  │                                                              │
  │   Pod ── mounts ──► emptyDir    Pod ──mounts──► PVC ──► PV   │
  │   │                                 │            │           │
  │   ▼                                ▼            ▼           │
  │  pod dies ⇒ data GONE            pod dies ⇒ data STAYS       │
  │                                                              │
  │  ③ StatefulSet (PVC per pod + stable identity)               │
  │                                                              │
  │   web-0 ─► PVC data-web-0        web-1 ─► PVC data-web-1     │
  │      (same PVC remounted if web-0 dies)                      │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
```

## ✅ Prerequisites

- `kubectl` on any cluster
- A working **dynamic provisioner** — one of:

| Cluster | Provisioner | Install |
|---------|-------------|---------|
| kind / Minikube | `local-path` (Rancher) | One-liner below |
| Amazon EKS | `gp2` / `gp3` via **EBS CSI driver** add-on | [Day-12](../Day-12_EKS_Storage_EBS/README.md) |
| EKS (shared multi-writer) | `efs-csi` `RetainMany` | [Day-21](../Day-21_EFS_CSI_Driver/README.md) |

---

## ▶️ Steps

### Step 1 — Make sure a StorageClass exists

**kind / Minikube** — install the local-path provisioner, then use the StorageClass below:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml
```

`local-path-storageclass.yaml` (already included above, shown for reference):

```yaml
# local-path StorageClass (kind/Minikube) — dynamic, host-volume-backed
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

**Amazon EKS** — use the EBS CSI StorageClass (gp2):

```yaml
# gp2 StorageClass (EKS) — requires the EBS CSI driver add-on (Day-12)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
parameters:
  type: gp2
  fsType: ext4
```

Verify:

```bash
kubectl get storageclass
```

**Expected output:**

```
NAME          PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path    rancher.io/local-path   Delete          WaitForFirstConsumer   false                  30s
```

```bash
kubectl create ns storage-lab
```

**Expected output:**

```
namespace/storage-lab created
```

---

### Step 2 — Proof #1: `emptyDir` dies with the pod

`empty-dir.yaml`:

```yaml
# emptyDir — scratch space, wiped when the pod is replaced
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stateless-app
  namespace: storage-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stateless-app
  template:
    metadata:
      labels:
        app: stateless-app
    spec:
      containers:
      - name: writer
        image: busybox:1.36
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
        - name: scratch
          mountPath: /data
      volumes:
      - name: scratch
        emptyDir: {}
```

```bash
kubectl apply -f empty-dir.yaml
kubectl get pods -n storage-lab -l app=stateless-app
```

**Expected output:**

```
NAME                             READY   STATUS    RESTARTS   AGE
stateless-app-6d97ff8f7b-k2r4b   1/1     Running   0          10s
```

Write a file, note the pod name, delete the pod, watch a fresh pod appear, and check whether the file survived:

```bash
POD=$(kubectl get pod -n storage-lab -l app=stateless-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n storage-lab "$POD" -- sh -c 'echo "precious data" > /data/test.txt && cat /data/test.txt'
kubectl delete pod -n storage-lab "$POD"
kubectl get pods -n storage-lab -l app=stateless-app
```

**Expected output** (new pod has a **different suffix**; the old one is gone):

```
precious data
pod "stateless-app-6d97ff8f7b-k2r4b" deleted

NAME                             READY   STATUS    RESTARTS   AGE
stateless-app-6d97ff8f7b-m9nq7   1/1     Running   0          6s
```

Check the file:

```bash
POD=$(kubectl get pod -n storage-lab -l app=stateless-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n storage-lab "$POD" -- cat /data/test.txt
```

**Expected output:**

```
cat: can't open '/data/test.txt': No such file or directory
```

> ✅ **Proved:** `emptyDir` is ephemeral per-pod scratch. Any pod that is replaced loses its data. Good for caches, bad for databases.

---

### Step 3 — Proof #2: a PVC survives pod replacement

A **PersistentVolumeClaim** requests storage; Kubernetes dynamically provisions a **PV** to bind it (or binds an existing one). A Deployment *reuses the same PVC* when its pod is recreated.

`pvc.yaml`:

```yaml
# PVC requesting 1Gi from local-path (or gp2 on EKS)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: storage-lab
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: local-path      # change to "gp2" on EKS
  resources:
    requests:
      storage: 1Gi
```

`pvc-app.yaml` (same as Step 2 but mounts the PVC):

```yaml
# Deployment that persists across pod replacement via the PVC
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pv-app
  namespace: storage-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pv-app
  template:
    metadata:
      labels:
        app: pv-app
    spec:
      containers:
      - name: writer
        image: busybox:1.36
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: app-data
```

```bash
kubectl apply -f pvc.yaml -f pvc-app.yaml
kubectl get pvc,pv -n storage-lab
```

**Expected output** (PV name is auto-generated; STATUS `Bound` on both sides):

```
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc/app-data   Bound    pvc-8f7c2e51-3a1a-45bb-9234-71b2b6d0f1c0   1Gi        RWO            local-path     20s

NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS   AGE
persistentvolume/pvc-8f7c2e51-3a1a-45bb-9234-71b2b6d0f1c0   1Gi        RWO            Delete           Bound    storage-lab/app-data   local-path     20s
```

Write a file, delete the pod, and prove persistence:

```bash
POD=$(kubectl get pod -n storage-lab -l app=pv-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n storage-lab "$POD" -- sh -c 'echo "survives reboots" > /data/test.txt && cat /data/test.txt'
kubectl delete pod -n storage-lab "$POD"
kubectl get pods -n storage-lab -l app=pv-app   # wait for 1/1 Running
```

**Expected output:**

```
survives reboots
pod "pv-app-7c9dd9f5b7-abc12" deleted

NAME                       READY   STATUS    RESTARTS   AGE
pv-app-7c9dd9f5b7-def34    1/1     Running   0          8s
```

And now the payoff:

```bash
POD=$(kubectl get pod -n storage-lab -l app=pv-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n storage-lab "$POD" -- cat /data/test.txt
```

**Expected output:**

```
survives reboots
```

> ✅ **Proved:** pod name changed (new pod), but the PVC is the same object, so the file is still there. **Pod replacement ≠ data loss.**

---

### Step 4 — Proof #3: StatefulSet with `volumeClaimTemplates`

Deployments share one PVC across replicas; StatefulSets give **each pod its own PVC** and a **stable ordinal name** (`web-0`, `web-1`). If `web-1` dies, the *same* PVC (`data-web-1`) is remounted.

`statefulset.yaml`:

```yaml
# StatefulSet + Service so each pod gets a stable identity (web-0, web-1)
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: storage-lab
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
  clusterIP: None            # headless — optional, but the StatefulSet needs serviceName
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: storage-lab
spec:
  serviceName: web
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: writer
        image: busybox:1.36
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: local-path      # change to "gp2" on EKS
      resources:
        requests:
          storage: 1Gi
```

```bash
kubectl apply -f statefulset.yaml
kubectl get pods, pvc -n storage-lab
```

**Expected output** (note the ordering: `web-0` created **before** `web-1`; two independent PVCs):

```
NAME        READY   STATUS    RESTARTS   AGE
pod/web-0   1/1     Running   0          30s
pod/web-1   1/1     Running   0          24s

NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/data-web-0   Bound    pvc-1111....                  1Gi        RWO            local-path     30s
persistentvolumeclaim/data-web-1   Bound    pvc-2222....                  1Gi        RWO            local-path     24s
```

**Ordinal identity test** — each pod stamps its own *name* into its own volume:

```bash
kubectl exec -n storage-lab web-0 -- sh -c 'echo "I am $HOSTNAME" > /data/ordinal.txt && cat /data/ordinal.txt'
kubectl exec -n storage-lab web-1 -- sh -c 'echo "I am $HOSTNAME" > /data/ordinal.txt && cat /data/ordinal.txt'
```

**Expected output:**

```
I am web-0
I am web-1
```

**PVC-remount test** — delete `web-1` (and only `web-1`), let the StatefulSet recreate it, and confirm the file says `web-1` again (the PVC `data-web-1` was re-attached, not re-created):

```bash
kubectl delete pod -n storage-lab web-1
kubectl get pods -n storage-lab -w        # wait for web-1 to be 1/1 Running
kubectl exec -n storage-lab web-1 -- cat /data/ordinal.txt
```

**Expected output:**

```
pod "web-1" deleted
NAME        READY   STATUS    RESTARTS   AGE
web-0       1/1     Running   0          5m
web-1       0/1     Terminating   0      4m
web-1       0/1     Pending        0      0s
web-1       0/1     ContainerCreating   0      0s
web-1       1/1     Running   0          4s

I am web-1
```

```bash
kubectl get pvc -n storage-lab
```

**Expected output** (still exactly 2 PVCs — `data-web-1` was reused, not recreated):

```
NAME          STATUS   VOLUME       ...   AGE
data-web-0    Bound    pvc-1111....        6m
data-web-1    Bound    pvc-2222....        5m
```

> ✅ **Proved:** StatefulSet = stable ordinal identity + per-pod PVC. The volume doesn't churn when the pod churns. This is why databases run as StatefulSets.

---

## ✅ Solution Recap

**Checkpoint commands (tick as you pass each):**

```bash
# Proof 1 — stateless
kubectl exec -n storage-lab <stateless-pod> -- sh -c 'echo x > /data/1'
kubectl delete pod -n storage-lab <stateless-pod>
kubectl exec -n storage-lab <new-pod> -- cat /data/1        # FAILS → emptyDir is ephemeral ✔

# Proof 2 — persistent via PVC
kubectl exec -n storage-lab <pv-pod> -- sh -c 'echo x > /data/1'
kubectl delete pod -n storage-lab <pv-pod>
kubectl exec -n storage-lab <new-pod> -- cat /data/1        # SUCCEEDS → PVC persists ✔

# Proof 3 — StatefulSet identity
kubectl exec -n storage-lab web-0 -- sh -c 'echo "I am $HOSTNAME" > /data/ordinal.txt'
kubectl delete pod -n storage-lab web-1                     # only web-1
kubectl exec -n storage-lab web-1 -- cat /data/ordinal.txt  # "I am web-1" still present ✔
kubectl get pvc -n storage-lab                              # 2 PVCs, never churned ✔
```

**What you proved:**

- [ ] `emptyDir` data vanishes on pod replacement
- [ ] A PVC binds to a dynamically provisioned PV
- [ ] Pod death does **not** delete the PVC → file survives
- [ ] StatefulSet creates pods in ordinal order (`web-0` then `web-1`)
- [ ] `volumeClaimTemplates` gives each pod its own PVC
- [ ] Deleting a StatefulSet pod remounts the **same** PVC

## 🧹 Cleanup

```bash
kubectl delete ns storage-lab
kubectl get pvc -n storage-lab    # empty; reclaimed when the namespace went away
kubectl get pv                     # Delete reclaim policy → PVs auto-deleted
```

## 🛠️ Troubleshooting

| Symptom | Likely cause | Check |
|---------|--------------|------|
| PVC stuck `Pending` | No provisioner / StorageClass doesn't exist | `kubectl describe pvc app-data` → Events |
| PVC `Pending` on EKS | EBS CSI driver add-on missing | `kubectl get pods -n kube-system \| grep ebs` |
| PVC `Pending` after pod scheduled | `WaitForFirstConsumer` needs the pod first | `kubectl get pod` for the app — it must be scheduled |
| Pod stays `ContainerCreating` | Volume mount waiting on storage | `kubectl describe pod <name>` → Events |
| `web-1` won't delete/recreate | StatefulSet replaces by ordinal, never recreates another | `kubectl get sts web -o yaml` |
| Need multiple pods to read/write **same** volume | `ReadWriteOnce` can't cross nodes → use **EFS** | [Day-21](../Day-21_EFS_CSI_Driver/README.md) |

## 📚 Resources

- [Kubernetes docs — Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Kubernetes docs — Persistent Volumes & Claims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes docs — StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [EKS — EBS CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)
- Related: [Day-05 Persistent Storage](../Day-05_Persistent_Storage/README.md), [Day-12 EKS Storage/EBS](../Day-12_EKS_Storage_EBS/README.md), [Day-21 EFS CSI](../Day-21_EFS_CSI_Driver/README.md)

**You can now explain, with `kubectl` output in hand, why databases belong in StatefulSets. 🚀**