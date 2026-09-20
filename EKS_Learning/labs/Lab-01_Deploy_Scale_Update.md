# 🧪 Lab 01: Deploy, Scale, Update — Proving the Pod Lifecycle

Everything in a Deployment's life is visible. In this lab you will witness — not just read about — each stage: create, scale, rolling update, rollback, crash, and self-heal.

## 🎯 Lab Goal

Prove the full pod lifecycle on a live cluster:

1. Create a 3-replica `nginx` Deployment from YAML
2. Expose it with a ClusterIP Service
3. Scale to 5 replicas imperatively
4. Rolling-update the image to `nginx:1.25`
5. Roll back to the previous revision
6. Kill a process and trip the liveness probe — and watch Kubernetes heal itself

## 🧪 Scenario

```
  ┌────────────────────────────────────────────────────────────────┐
  │                POD LIFECYCLE — THIS LAB                         │
  │                                                                │
  │  create ──► scale ──► update ──► rollback ──► crash ──► heal  │
  │   (3)        (5)     1.21→1.25   1.25→1.21     kill 1   restart│
  │                                                                │
  │        ┌───────────────────────────────┐                       │
  │        │  namespace: lifecycle-lab     │                       │
  │        │                              │                       │
  │        │  Deployment: nginx-demo ─────┼──► ReplicaSet ─► Pods │
  │        │  Service:     nginx-demo     │       │               │
  │        │                (ClusterIP)   │       └──► Pods       │
  │        │                              │                        │
  │        └───────────────────────────────┘                       │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘
```

## ✅ Prerequisites

- `kubectl` pointing at any cluster (kind, Minikube, or EKS)
- Internet access so your cluster can pull `nginx` images

---

## ▶️ Steps

### Step 1 — Create the namespace

```bash
kubectl create ns lifecycle-lab
```

**Expected output:**

```
namespace/lifecycle-lab created
```

```bash
kubectl get ns lifecycle-lab
```

```
NAME              STATUS   AGE
lifecycle-lab     Active   3s
```

---

### Step 2 — Deploy a 3-replica nginx Deployment

`deploy.yaml`:

```yaml
# Deploy 3 nginx replicas with readiness + liveness probes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: lifecycle-lab
  labels:
    app: nginx-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
```

```bash
kubectl apply -f deploy.yaml
```

**Expected output:**

```
deployment.apps/nginx-demo created
```

```bash
kubectl get deployments,replicasets,pods -n lifecycle-lab -l app=nginx-demo
```

**Expected output** (the ReplicaSet hash and pod suffixes will differ):

```
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-demo   3/3     3            3           18s

NAME                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-demo-6b8f9d9b6   3         3         3       18s

NAME                             READY   STATUS    RESTARTS   AGE
pod/nginx-demo-6b8f9d9b6-abc12   1/1     Running   0          18s
pod/nginx-demo-6b8f9d9b6-def34   1/1     Running   0          18s
pod/nginx-demo-6b8f9d9b6-ghi56   1/1     Running   0          18s
```

> The Deployment owns a ReplicaSet, which owns the Pods. `3/3` READY = all three passed the readiness probe.

---

### Step 3 — Expose a ClusterIP Service

`service.yaml`:

```yaml
# Stable network endpoint in front of the nginx-demo pods
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo
  namespace: lifecycle-lab
spec:
  type: ClusterIP
  selector:
    app: nginx-demo
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f service.yaml
```

**Expected output:**

```
service/nginx-demo created
```

```bash
kubectl get svc, endpoints -n lifecycle-lab -l app=nginx-demo
kubectl get endpoints -n lifecycle-lab nginx-demo
```

**Expected output** (3 pod IPs appear in the Endpoints object — proof the selector matched):

```
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/nginx-demo   ClusterIP   10.110.12.34    <none>        80/TCP    12s

NAME                 ENDPOINTS                                             AGE
endpoints/nginx-demo 10.244.1.7:80,10.244.2.9:80,10.244.1.12:80   12s
```

---

### Step 4 — Scale to 5 replicas (imperative)

```bash
kubectl scale deployment nginx-demo --replicas=5 -n lifecycle-lab
kubectl get pods -n lifecycle-lab -l app=nginx-demo -w
```

**Expected output** — two new pods appear, and the ReplicaSet catches up:

```
NAME                             READY   STATUS              RESTARTS   AGE
nginx-demo-6b8f9d9b6-abc12       1/1     Running             0          4m
nginx-demo-6b8f9d9b6-def34       1/1     Running             0          4m
nginx-demo-6b8f9d9b6-ghi56       1/1     Running             0          4m
nginx-demo-6b8f9d9b6-jkl78       0/1     ContainerCreating   0          0s
nginx-demo-6b8f9d9b6-jkl78       1/1     Running             0          4s
nginx-demo-6b8f9d9b6-mno90       0/1     ContainerCreating   0          1s
nginx-demo-6b8f9d9b6-mno90       1/1     Running             0          4s
```

```bash
kubectl get deployment nginx-demo -n lifecycle-lab
```

**Expected output:**

```
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
nginx-demo   5/5     5            5           5m
```

> Imperative `kubectl scale` is great for demos and chaos drills; for real environments put `replicas: 5` in Git and let ArgoCD apply it (see Day-31).

---

### Step 5 — Rolling update: set image to `nginx:1.25`

```bash
kubectl set image deployment/nginx-demo nginx=nginx:1.25 -n lifecycle-lab

# Watch the rollout (be patient — new pods need to become READY first)
kubectl rollout status deployment/nginx-demo -n lifecycle-lab
```

**Expected output:**

```
deployment.apps/nginx-demo image updated
Waiting for deployment "nginx-demo" rollout to finish: 0 out of 5 new replicas have been updated...
Waiting for deployment "nginx-demo" rollout to finish: 3 out of 5 new replicas have been updated...
deployment "nginx-demo" successfully rolled out
```

While it runs, watch the old and new ReplicaSets coexist (maxSurge/maxUnavailable default = 25% → you see 6 pods during the transition):

```bash
kubectl get rs -n lifecycle-lab -l app=nginx-demo
```

**Expected output** (a second ReplicaSet appears with a **new hash**, the old one scales down):

```
NAME                       DESIRED   CURRENT   READY   AGE
nginx-demo-6b8f9d9b6       0         0         0       10m    # ← old, 1.21 → draining to 0
nginx-demo-9f4c2d8e7       5         5         5       1m     # ← new, 1.25
```

```bash
kubectl rollout history deployment/nginx-demo -n lifecycle-lab
```

**Expected output:**

```
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

> `CHANGE-CAUSE` is blank because `kubectl set image` carries no annotation. Add one for meaningful history:
> `kubectl annotate deployment nginx-demo kubernetes.io/change-cause="bump to 1.25" -n lifecycle-lab`

---

### Step 6 — Roll back to the previous image

```bash
kubectl rollout undo deployment/nginx-demo -n lifecycle-lab
kubectl rollout status deployment/nginx-demo -n lifecycle-lab
```

**Expected output:**

```
deployment.apps/nginx-demo rolled back
deployment "nginx-demo" successfully rolled out
```

Prove the old image is back:

```bash
kubectl get pods -n lifecycle-lab -l app=nginx-demo -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

**Expected output:**

```
nginx-demo-6b8f9d9b6-pqr11   nginx:1.21
nginx-demo-6b8f9d9b6-stu22   nginx:1.21
nginx-demo-6b8f9d9b6-vwx33   nginx:1.21
nginx-demo-6b8f9d9b6-yza44   nginx:1.21
nginx-demo-6b8f9d9b6-bcd55   nginx:1.21
```

---

### Step 7 — Crash a container: `kill 1`, watch Kubernetes restart it

PID 1 in an nginx container is the master process. Killing it crashes the container.

```bash
POD=$(kubectl get pod -n lifecycle-lab -l app=nginx-demo -o jsonpath='{.items[0].metadata.name}')
echo "POD=$POD"
kubectl exec -it -n lifecycle-lab "$POD" -- kill 1

# Immediately watch the RESTARTS counter climb
kubectl get pods -n lifecycle-lab -l app=nginx-demo -o wide -w
```

**Expected output:**

```
POD=nginx-demo-6b8f9d9b6-pqr11
NAME                             READY   STATUS    RESTARTS   AGE
nginx-demo-6b8f9d9b6-pqr11       0/1     Running   0          3m
nginx-demo-6b8f9d9b6-pqr11       0/1     Running   1          3m     ← restarted by kubelet
nginx-demo-6b8f9d9b6-pqr11       1/1     Running   1          3m     ← back to READY, SAME pod
nginx-demo-6b8f9d9b6-stu22       1/1     Running   0          3m
nginx-demo-6b8f9d9b6-vwx33       1/1     Running   0          3m
nginx-demo-6b8f9d9b6-yza44       1/1     Running   0          3m
nginx-demo-6b8f9d9b6-bcd55       1/1     Running   0          3m
```

> Notice the **pod name did not change**. The restartPolicy (default `Always`) makes the kubelet restart the *same* container in the *same* pod. That is **container-level self-healing**.

---

### Step 8 — Trip the liveness probe: make the HTTP check fail

The current container is healthy again, but the probe checks `GET /`. Remove nginx's index page — the probe now gets a 404, kubelet kills the container, and the liveness probe triggers a restart.

```bash
POD=$(kubectl get pod -n lifecycle-lab -l app=nginx-demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n lifecycle-lab "$POD" -- rm /usr/share/nginx/html/index.html

# Probe runs every 5s; within seconds the container cycles
kubectl get pods -n lifecycle-lab -l app=nginx-demo -w
```

**Expected output** (RESTARTS jumps again as the probe fails):

```
NAME                            READY   STATUS    RESTARTS   AGE
nginx-demo-6b8f9d9b6-pqr11      1/1     Running   1          5m
nginx-demo-6b8f9d9b6-pqr11      0/1     Running   2          5m     ← liveness probe 404 → killed
nginx-demo-6b8f9d9b6-pqr11      1/1     Running   2          5m     ← fresh container, file restored
nginx-demo-6b8f9d9b6-stu22      1/1     Running   0          5m
```

Explain what happened to yourself out loud — this is the exact mechanism that heals a wedged app:

```
  GET /  → 404 (file deleted)
      │
      ▼
 kubelet: liveness probe FAILED
      │
      ▼
 kubelet kills the container (kill signal), then restarts it
      │
      ▼
 fresh filesystem → index.html is back → GET / → 200 → READY
```

Confirm via events:

```bash
kubectl get events -n lifecycle-lab --sort-by=.lastTimestamp | tail -5
```

**Expected output** (your resource names/uuids differ):

```
...
3m         Normal   Killing     pod/nginx-demo-6b8f9d9b6-pqr11   Stopping container nginx
3m         Normal   Pulled      pod/nginx-demo-6b8f9d9b6-pqr11   Container image "nginx:1.21" already present
3m         Normal   Created     pod/nginx-demo-6b8f9d9b6-pqr11   Created container nginx
3m         Normal   Started     pod/nginx-demo-6b8f9d9b6-pqr11   Started container nginx
2m         Normal   Ready       pod/nginx-demo-6b8f9d9b6-pqr11   Pod ready
```

---

### Step 9 — Final state

```bash
kubectl get deployment,rs,svc,pods -n lifecycle-lab -o wide
```

**Expected output:**

```
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-demo   5/5     5            5           22m

NAME                       DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-demo-6b8f9d9b6   5         5         5       22m

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/nginx-demo   ClusterIP   10.110.12.34   <none>        80/TCP    21m

NAME                             READY   STATUS    RESTARTS   AGE
pod/nginx-demo-6b8f9d9b6-pqr11   1/1     Running   2          22m
pod/nginx-demo-6b8f9d9b6-stu22   1/1     Running   0          22m
pod/nginx-demo-6b8f9d9b6-vwx33   1/1     Running   0          22m
pod/nginx-demo-6b8f9d9b6-yza44   1/1     Running   0          22m
pod/nginx-demo-6b8f9d9b6-bcd55   1/1     Running   0          22m
```

---

## ✅ Solution Recap

**Checkpoint commands (complete prove-the-lifecycle set):**

```bash
# 1. Deploy
kubectl apply -f deploy.yaml -f service.yaml -n lifecycle-lab

# 2. Everything healthy after each stage
kubectl get deploy,rs,pods,svc,endpoints -n lifecycle-lab

# 3. Scale
kubectl scale deployment nginx-demo --replicas=5 -n lifecycle-lab

# 4. Rolling update
kubectl set image deployment/nginx-demo nginx=nginx:1.25 -n lifecycle-lab
kubectl rollout status deployment/nginx-demo -n lifecycle-lab

# 5. Rollback
kubectl rollout undo deployment/nginx-demo -n lifecycle-lab
kubectl rollout status deployment/nginx-demo -n lifecycle-lab

# 6. Self-healing (container crash)
kubectl exec -it -n lifecycle-lab <any-pod> -- kill 1
kubectl get pods -n lifecycle-lab -w          # RESTARTS increments

# 7. Self-healing (liveness probe failure)
kubectl exec -n lifecycle-lab <any-pod> -- rm /usr/share/nginx/html/index.html
kubectl get pods -n lifecycle-lab -w          # RESTARTS increments again
```

**What you proved (tick them off):**

- [ ] Deployment → ReplicaSet → Pods ownership chain
- [ ] Service selector → Endpoints populated with pod IPs
- [ ] Imperative scaling works, declarative desired count reconciled
- [ ] Rolling update creates a new ReplicaSet; old one drains
- [ ] Rollback restores the exact previous image
- [ ] Container crash → kubelet restarts the *same* pod
- [ ] Liveness probe failure → container killed and replaced

## 🧹 Cleanup

```bash
kubectl delete ns lifecycle-lab
kubectl get ns lifecycle-lab   # should show "Terminating" then be gone
```

## 🛠️ Troubleshooting

| Symptom | Likely cause | Check |
|---------|--------------|-------|
| Pods stuck `ContainerCreating` | Image registry unreachable | `kubectl describe pod <name>` → Events |
| `ImagePullBackOff` | Typo in image name/tag | `kubectl describe pod <name>` → "Failed to pull image" |
| Pending pods after scaling | Node has no capacity (kind: one node) | `kubectl describe nodes` / `kubectl top nodes` |
| Rollout never finishes | Readiness probe failing on old or new image | `kubectl describe pod <name>` → probe events |
| `kubectl set image` not found error | Container name mismatch | `kubectl get deployment <name> -o yaml` and compare the `name:` field |

## 📚 Resources

- [Kubernetes docs — Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes docs — Rolling updates & rollbacks](https://kubernetes.io/docs/tasks/run-application/rolling-update-replication-controller/)
- [Kubernetes docs — Liveness/readiness/startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [kubectl cheat sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- Related: [Day-02 Pods/ReplicaSets/Deployments](../Day-02_Pods_ReplicaSets_Deployments/README.md), [Day-03 Services](../Day-03_Services_Networking/README.md)

**You have now watched Kubernetes heal itself in real time. 🚀**