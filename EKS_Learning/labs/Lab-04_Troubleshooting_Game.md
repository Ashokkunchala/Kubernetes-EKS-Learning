# 🧪 Lab 04: The Troubleshooting Game — Fix 4 Broken Deployments

Interviewers love asking "how would you debug a CrashLoopBackOff?". This lab makes you **actually do it** — four deliberately broken Deployments, one at a time, scored like a game.

## 🎯 Lab Goal

Diagnose and fix four broken workloads, each with a different root cause:

| Round | Symptom | Root cause you'll find |
|-------|---------|------------------------|
| 1 | `ImagePullBackOff` | Bad image tag that doesn't exist |
| 2 | `CrashLoopBackOff` | Command exits immediately |
| 3 | Pod `Running` but `0/1` NOT Ready | Readiness probe hits a wrong path |
| 4 | `CreateContainerConfigError` | Secret referenced in `envFrom` is missing |

## 🧪 Scenario

Four teams shipped broken manifests into the `troubleshoot-lab` namespace. Your job: for each, find the **root cause**, apply the **minimal fix**, and **prove** it's fixed — in under 8 minutes each.

```
  ┌──────────────────────────────────────────────┐
  │   troubleshoot-lab (4 broken deployments)    │
  │                                              │
  │  app-imagepull ─► ImagePullBackOff           │
  │  app-crashloop ─► CrashLoopBackOff           │
  │  app-notready  ─► Running, but 0/1 Ready      │
  │  app-nosecret  ─► CreateContainerConfigError │
  └──────────────────────────────────────────────┘
```

## 🛠️ Diagnosis Toolkit (memorize this)

```bash
# 1. See the symptom
kubectl get pods -n troubleshoot-lab -o wide

# 2. See WHY (Events section is the single most valuable screen in kubectl)
kubectl describe pod <pod-name> -n troubleshoot-lab

# 3. See the app's last words
kubectl logs <pod-name> -n troubleshoot-lab
kubectl logs <pod-name> -n troubleshoot-lab --previous   # crashed instance

# 4. Cluster-wide view of what's happening
kubectl get events -n troubleshoot-lab --sort-by=.lastTimestamp

# 5. Drill into the controller level
kubectl get rs,deploy -n troubleshoot-lab
kubectl describe rs <rs-name> -n troubleshoot-lab
```

**Root-cause map — match the symptom to the mechanism:**

```
  ImagePullBackOff        → registry/name/tag/auth problem, image never lands
  ErrImagePull            → (same, earlier stage) + node can't reach registry
  CrashLoopBackOff        → container STARTS then dies (bad cmd, bad config, segfault)
  CreateContainerConfigError → container was never created (bad refs: secret/cm/rbac)
  Running 0/1 (NotReady)  → container is ALIVE but probes say "not ready" (traffic-safe check failed)
  Pending                 → scheduler couldn't place it (capacity / PVC / taint)
```

```bash
kubectl create ns troubleshoot-lab
```

---

## ▶️ Round 1 — `ImagePullBackOff`

**Broken manifest** — apply it and diagnose:

`round1-broken.yaml`:

```yaml
# v1: image tag DOES NOT EXIST → ImagePullBackOff
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-imagepull
  namespace: troubleshoot-lab
spec:
  replicas: 1
  selector: {matchLabels: {app: app-imagepull}}
  template:
    metadata:
      labels: {app: app-imagepull}
    spec:
      containers:
      - name: nginx
        image: nginx:9.9        # 9.9 was never published — deliberate break
        ports: [{containerPort: 80}]
```

```bash
kubectl apply -f round1-broken.yaml
kubectl get pods -n troubleshoot-lab
```

**Expected output** (container never starts):

```
NAME                                READY   STATUS             RESTARTS   AGE
pod/app-imagepull-5f9f7b9d7b-lz1qk   0/1     ImagePullBackOff   0          50s
```

```bash
kubectl describe pod -n troubleshoot-lab -l app=app-imagepull
```

**Expected output** (bottom of Events):

```
Events:
  Type     Reason     Age    From               Message
  ----     ------     ----   ----               -------
  Normal   Scheduled  72s    default-scheduler  Successfully assigned ...
  Normal   BackOff    68s    kubelet            Back-off pulling image "nginx:9.9"
  Warning  Failed     68s    kubelet            Error: ImagePullBackOff
  Normal   Pulling    39s    kubelet            Pulling image "nginx:9.9"
  Warning  Failed     39s    kubelet            Error: image pull failed, rpc error: ...
  Warning  Failed     39s    kubelet            Error: Job back-off limit exceeded
```

### ✅ Fix

```bash
kubectl set image deployment/app-imagepull nginx=nginx:1.27 -n troubleshoot-lab
kubectl rollout status deployment/app-imagepull -n troubleshoot-lab
```

**Expected output:**

```
deployment.apps/app-imagepull image updated
deployment "app-imagepull" successfully rolled out
```

**Verify (one command):**

```bash
kubectl get pods -n troubleshoot-lab -l app=app-imagepull
```

**Expected output:**

```
NAME                                READY   STATUS    RESTARTS   AGE
pod/app-imagepull-5d8c8b7d6f-x2mny   1/1     Running   0          15s
```

**Root cause:** `nginx:9.9` doesn't exist → kubelet can't pull → pod sits in `ImagePullBackOff`. Fix = valid tag.

---

## ▶️ Round 2 — `CrashLoopBackOff`

**Broken manifest:**

`round2-broken.yaml`:

```yaml
# v1: command immediately exits → container crashes in a loop
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-crashloop
  namespace: troubleshoot-lab
spec:
  replicas: 1
  selector: {matchLabels: {app: app-crashloop}}
  template:
    metadata:
      labels: {app: app-crashloop}
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "exit 1"]     # dies instantly — deliberate break
```

```bash
kubectl apply -f round2-broken.yaml
kubectl get pods -n troubleshoot-lab -l app=app-crashloop
```

**Expected output** (note the RESTARTS counter climbing):

```
NAME                                 READY   STATUS             RESTARTS   AGE
pod/app-crashloop-6f99d6f4c8-q3yzp   0/1     CrashLoopBackOff   4          90s
```

Logs tell you the story — the app never even logged anything because it `exit`ed:

```bash
kubectl logs -n troubleshoot-lab -l app=app-crashloop
kubectl logs -n troubleshoot-lab -l app=app-crashloop --previous
```

**Expected output:**

```
(no output from either — the process died before writing anything)
```

### ✅ Fix

Change the command to something that keeps the process alive:

```bash
kubectl patch deployment app-crashloop -n troubleshoot-lab --type='json' -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/command",
   "value":["sh","-c","sleep 3600"]}
]'
kubectl rollout status deployment/app-crashloop -n troubleshoot-lab
```

**Expected output:**

```
deployment.apps/app-crashloop patched
deployment "app-crashloop" successfully rolled out
```

**Verify (one command):**

```bash
kubectl get pods -n troubleshoot-lab -l app=app-crashloop
```

**Expected output:**

```
NAME                                READY   STATUS    RESTARTS   AGE
pod/app-crashloop-7c9c7f6d5c-qq1aa   1/1     Running   0          18s
```

**Root cause:** `exit 1` terminates the container → restartPolicy `Always` retries → each retry dies again → `CrashLoopBackOff`. Real-world variants: a typo'd `command`, a bad entrypoint script, or an app segfaulting on startup config.

---

## ▶️ Round 3 — Pod `Running` but `0/1` (Not Ready)

**Broken manifest:**

`round3-broken.yaml`:

```yaml
# v1: readiness probe points at /readyz which nginx does not serve → 404
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-notready
  namespace: troubleshoot-lab
spec:
  replicas: 1
  selector: {matchLabels: {app: app-notready}}
  template:
    metadata:
      labels: {app: app-notready}
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports: [{containerPort: 80}]
        readinessProbe:
          httpGet:
            path: /readyz        # 404 on nginx → never Ready — deliberate break
            port: 80
          initialDelaySeconds: 2
          periodSeconds: 5
```

```bash
kubectl apply -f round3-broken.yaml
kubectl get pods -n troubleshoot-lab -l app=app-notready
```

**Expected output** (the sneaky one — `Running` but `0/1`!):

```
NAME                                 READY   STATUS    RESTARTS   AGE
pod/app-notready-7d9d8f875c-ab34cd   0/1     Running   0          45s
```

The container is *fine*; the probe is not. Confirm via describe Events:

```bash
kubectl describe pod -n troubleshoot-lab -l app=app-notready
```

**Expected output:**

```
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Normal   Scheduled  58s   default-scheduler  Successfully assigned ...
  Normal   Pulled     58s   kubelet            Container image "nginx:1.27" already present
  Normal   Created    58s   kubelet            Created container nginx
  Normal   Started    58s   kubelet            Started container nginx
  Warning  Unhealthy  56s   kubelet            Readiness probe failed: HTTP probe failed with statuscode: 404
  Normal   Killing    56s   kubelet            Stopping container nginx
  Normal   Pulled     56s   kubelet            Container image "nginx:1.27" already present
  ...
```

**The production consequence** — a NotReady container is *not* added to the Service Endpoints, so the Service starts dropping traffic:

```bash
kubectl get endpoints -n troubleshoot-lab app-notready-svc 2>/dev/null || echo "no service yet"
kubectl get pods -n troubleshoot-lab -l app=app-notready --field-selector=status.phase=Running
```

### ✅ Fix

```bash
kubectl patch deployment app-notready -n troubleshoot-lab --type='json' -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe",
   "value":{"httpGet":{"path":"/","port":80},"initialDelaySeconds":2,"periodSeconds":5}}
]'
kubectl rollout status deployment/app-notready -n troubleshoot-lab
```

**Expected output:**

```
deployment.apps/app-notready patched
deployment "app-notready" successfully rolled out
```

**Verify (one command):**

```bash
kubectl get pods -n troubleshoot-lab -l app=app-notready
```

**Expected output:**

```
NAME                                READY   STATUS    RESTARTS   AGE
pod/app-notready-5f7d8c9b6d-cd12ef   1/1     Running   0          20s
```

**Root cause:** the readiness probe asked nginx for `/readyz`; nginx returns 404 → probe fails → pod never marked Ready → kept out of Service Endpoints → the app 404s for users. Real-world variant: probe path changed in the new image but not in the manifest.

---

## ▶️ Round 4 — `CreateContainerConfigError` (missing Secret)

**Broken manifest:**

`round4-broken.yaml`:

```yaml
# v1: envFrom references a Secret that does not exist → container can't be created
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-nosecret
  namespace: troubleshoot-lab
spec:
  replicas: 1
  selector: {matchLabels: {app: app-nosecret}}
  template:
    metadata:
      labels: {app: app-nosecret}
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "echo DB_USER=$DB_USER; while true; do sleep 30; done"]
        envFrom:
        - secretRef:
            name: my-app-secret      # this Secret was never created — deliberate break
```

```bash
kubectl apply -f round4-broken.yaml
kubectl get pods -n troubleshoot-lab -l app=app-nosecret
```

**Expected output:**

```
NAME                                READY   STATUS                       RESTARTS   AGE
pod/app-nosecret-78fb7c9d8b-fg34hi   0/1     CreateContainerConfigError   0          30s
```

Describe pins the exact missing object:

```bash
kubectl describe pod -n troubleshoot-lab -l app=app-nosecret
```

**Expected output:**

```
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Normal   Scheduled  32s   default-scheduler  Successfully assigned ...
  Warning  Failed     31s   kubelet            Error: secret "my-app-secret" not found
  Warning  Failed     31s   kubelet            Error: CreateContainerConfigError
```

```bash
kubectl get secret -n troubleshoot-lab
```

**Expected output:**

```
No resources found in troubleshoot-lab namespace.
```

### ✅ Fix

Create the missing Secret (the app expects `DB_USER`), or delete `envFrom` if it wasn't needed:

```bash
kubectl create secret generic my-app-secret -n troubleshoot-lab \
  --from-literal=DB_USER=ecommerce \
  --from-literal=DB_PASSWORD='CHANGE-ME-long-enough'
kubectl rollout restart deployment app-nosecret -n troubleshoot-lab
kubectl rollout status deployment/app-nosecret -n troubleshoot-lab
```

**Expected output:**

```
secret/my-app-secret created
deployment.apps/app-nosecret restarted
deployment "app-nosecret" successfully rolled out
```

**Verify (one command):**

```bash
kubectl exec -n troubleshoot-lab deploy/app-nosecret -- sh -c 'echo "DB_USER=$DB_USER"'
```

**Expected output:**

```
DB_USER=ecommerce
```

**Root cause:** `envFrom` → `secretRef: my-app-secret`; the Secret doesn't exist, so the container config can never be assembled → `CreateContainerConfigError`. Real-world variant: secret present in the wrong namespace, or the IRSA/Secrets Manager setup is pointing at a nonexistent name (see [Day-29](../Day-29_Secrets_Manager_Integration/README.md)).

---

## ✅ Scoring Rubric

| Round | Root cause identified (+1) | Minimal fix applied (+1) | Proven with 1 command (+1) |
|-------|:--:|:--:|:--:|
| 1 `ImagePullBackOff` |  |  |  |
| 2 `CrashLoopBackOff` |  |  |  |
| 3 NotReady / probe |  |  |  |
| 4 Missing Secret |  |  |  |

**Total: /12**

| Score | Proficiency |
|-------|-------------|
| 10–12 | Production-grade on-call. You're ready for SRE rotations. |
| 7–9 | Solid. Re-practice the rounds you missed and timebox to 5 min. |
| 4–6 | Comfortable with `describe`/`logs`, but lean on the Events output next time. |
| 0–3 | Re-read [Day-60 Final Review](../Day-60_Final_Review_Certification_Preparation/README.md) troubleshooting sections, then redo the lab from scratch. |

**Timebox discipline:** 8 minutes per round. The Events section of `kubectl describe` should be your first stop every time — most answers live there.

## 🧹 Cleanup

```bash
kubectl delete ns troubleshoot-lab
```

## 🛠️ Troubleshooting

| Event message you'll see | Meaning | Reflex |
|--------------------------|---------|--------|
| `Back-off pulling image` | tag/name/auth problem | `kubectl describe pod` → fix image, or `docker pull` locally to test |
| `Failed to pull image ... manifest unknown` | tag doesn't exist | choose a published tag (verify on Docker Hub) |
| `Error: secret "x" not found` | config ref missing | create Secret/ConfigMap, or fix the reference |
| `Readiness probe failed ... 404` | wrong probe path/port | `kubectl exec <pod> -- curl localhost:<port><path>` to find the real endpoint |
| `ContainerCreating` forever | volume or registry slowness | `kubectl describe pod` Events + check PVC/StorageClass |

## 📚 Resources

- [Debugging Workloads (official)](https://kubernetes.io/docs/tasks/debug/)
- [Debugging Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Debugging Deployments](https://kubernetes.io/docs/tasks/debug/debug-application/debug-deployment/)
- [Kubectl cheat sheet — inspect & debug](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- Related: [Day-20 Networking Interview Prep](../Day-20_Networking_Interview_Preparation/README.md), [Day-45 Monitoring Stack Practice](../Day-45_Monitoring_Stack_Practice/README.md)

**Fastest on-call habit you can build: read the Events section first. 🚀**