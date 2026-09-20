# 🧪 CKAD Practice Exam — Part 1 (Workload Authoring)

20 application-developer tasks: author Deployments, inject config, wire probes, run
Jobs/CronJobs, attach storage, add sidecars, control rollouts, and expose apps. All from
memory in a live terminal. Timebox: **~2.5 hours**, then grade against the answer key.

> CKAD is about *writing manifests fast and correctly*. When in doubt,
> `kubectl create ... --dry-run=client -o yaml` gives you the skeleton.

---

## 📝 Tasks

### Task 01 — Deployment + Service
Create a Deployment `hello` (image `nginx:1.25`, 2 replicas) with container port **80**.
Expose it as a ClusterIP Service `hello-svc` so it's reachable on port **80**. Verify
endpoints = 2.

**Constraints:** Manifest must include a `ports.containerPort` on the container.

**Hint:** Create pod template with `kubectl create deployment hello --image=nginx:1.25 --replicas=2 --dry-run=client -o yaml` then expose.

### Task 02 — NodePort Service
Create Service `frontend` type **NodePort** (port 80 → targetPort `web` by name) pointing
at pods labeled `app=frontend`. Confirm the allocated nodePort is in the 30000-32767 range.

**Constraints:** Use `targetPort: web` (named), not a number.

**Hint:** Named targetPort requires the container to declare `name: web` in ports.

### Task 03 — ConfigMap + Secret as Env
Create ConfigMap `app-config` with `LOG_LEVEL=info` and Secret `app-secret` with
`DB_PASSWORD=hunter2hunter2`. A pod `envio` (image `nginx:1.25`) must receive
`LOG_LEVEL` from the ConfigMap and `DB_PASSWORD` from the Secret, then print them via
`env`. Verify both are set (`kubectl exec ... -- env`).

**Constraints:** All via `valueFrom.configMapKeyRef` / `valueFrom.secretKeyRef` with the
right `key:`.

**Hint:** base64 the secret value yourself: `printf 'hunter2hunter2' | base64`.

### Task 04 — ConfigMap as Volume
Mount ConfigMap `app-config` into pod `cfvol` (image `nginx:1.25`) at `/etc/app` as
**individual files** (one file per key). Verify `cat /etc/app/LOG_LEVEL`.

**Constraints:** Volume mount; `items:` not required but permitted — keep it simple.

**Hint:** `volumes: [{name: cfg, configMap: {name: app-config}}]` + mountPath `/etc/app`.

### Task 05 — Liveness Probe
Edit Deployment `ping` so its container restarts when the app stops responding: HTTP GET
on `/healthz` port 8080, delay 5s, period 10s, failureThreshold 3. Start with
`kubectl create deployment ping --image=nginx:1.25 --dry-run=client -o yaml` and patch in
the probe + a command container if needed.

**Constraints:** `livenessProbe` block with `httpGet`, `initialDelaySeconds`, `periodSeconds`.

**Hint:** nginx won't fail by itself — add `command: ["sh","-c","rm -f /usr/share/nginx/html/healthz; sleep 30; touch /usr/share/nginx/html/healthz"]` so the first 30s the probe 404s and kubelet restarts it (observable via `RESTARTS`).

### Task 06 — Readiness Probe
Deployment `api` must only receive traffic when `/ready` returns 200. Add a readiness
probe (port 8080) that gates the Service endpoints; verify a pod with a broken ready
endpoint never appears in `kubectl get endpoints`.

**Constraints:** Readiness **must not** restart the container — no liveness involved.

**Hint:** readinessProbe differs semantically: failure = RemoveEndpoints, not restart. Check `kubectl get svc` shows READY 1/1 vs endpoints count 0 when the file is missing.

### Task 07 — Startup Probe
App `slowboot` needs **90 seconds** to become healthy but its liveness check is port 8080.
Add a `startupProbe` (TCP check, 30s period, 6 failures) that *defers* the liveness probe
from killing it during boot.

**Constraints:** Both probes coexist; liveness must not fire before startup succeeds.

**Hint:** kubelet won't run liveness until startupProbe passes — that's exactly what it exists for.

### Task 08 — Sidecar On Demand
Create pod `duo` with containers `main` (nginx:1.25) and `logger` (busybox:1.36, command
`sh -c "tail -f /shared/nginx.log"`) sharing an **emptyDir** volume mounted at `/shared`
in both. Add `securityContext.runAsUser: 1000` to the main container.

**Constraints:** Shared mount must be referenced in both containers; emptyDir only.

**Hint:** Same volume name + same mountPath in each container; nginx main writes to the shared dir via a custom command or `wget` test.

### Task 09 — One-Off Job
Create Job `backfill` (image `busybox:1.36`) running `sh -c "ps aux > /dev/null"` that
runs **exactly once** and completes. Show `COMPLETIONS 1/1`.

**Constraints:** `apiVersion: batch/v1`, `restartPolicy: OnFailure`, `completions: 1`.

**Hint:** `kubectl create job backfill --image=busybox:1.36 -- sh -c "ps aux > /dev/null"`.

### Task 10 — CronJob
Create CronJob `etl` (image `busybox:1.36`) running `echo "run every 5 min"`, schedule
`*/5 * * * *`, with `concurrencyPolicy: Forbid` so overlapping runs are skipped. Verify
`kubectl get cronjob` shows the schedule and a successfully completed Job appears within ~6 minutes.

**Constraints:** `schedule`, `jobTemplate.spec`, `concurrencyPolicy` all set.

**Hint:** `kubectl create cronjob etl --image=busybox:1.36 --schedule="*/5 * * * *" -- echo "run every 5 min"` then patch in `concurrencyPolicy: Forbid`.

### Task 11 — PVC ReadWriteOnce
Create a 1Gi `ReadWriteOnce` PVC in namespace `data` using the default storage
class. Deploy pod `cache` (redis:7) mounting it at `/data`. Verify the PV is `Bound` and
the data directory is writable.

**Constraints:** Don't create a PV yourself — dynamically provisioned. `resources.requests.storage: 1Gi`.

**Hint:** In EKS the default storageClass is `gp2`/`gp3` backed by the EBS CSI driver; the PVC alone triggers provisioning.

### Task 12 — Service Account
Create ServiceAccount `cicd` in namespace `ci`. Use it in a Deployment `runner` (image
`alpine:3.19`, command `sleep 3600`). Verify the pod's `serviceAccountName` and that the
automounted token exists at `/var/run/secrets/kubernetes.io/serviceaccount/token`.

**Constraints:** Deployment must reference the SA by name; no implicit `default`.

**Hint:** `kubectl create sa cicd -n ci` then `--serviceaccount` or the manifest field.

### Task 13 — nodeSelector Two-Way
Label node `ip-10-0-1-12.ec2.internal` with `purpose=gpu`. Create a pod `gpu-task` that
must land there (`nodeSelector`), and a second pod `general` that explicitly tolerates
nodes **with or without** that label.

**Constraints:** `gpu-task` must schedule to the labeled node; `general` to *any* node.

**Hint:** `nodeSelector: {purpose: gpu}` forces the hard choice; for "anywhere", just omit nodeSelector — anti-labels aren't required for the second pod.

### Task 14 — Host-Based Ingress
Create Ingress `shop` (ingressClassName `nginx`) routing `shop.example.com/api` to Service
`api-svc:80` and `shop.example.com/` to Service `web-svc:80`. Both services already
exist/have endpoints. Verify rules in `kubectl get ingress -o yaml`.

**Constraints:** `networking.k8s.io/v1`, `pathType: Prefix`, two `paths` under one host.

**Hint:** Backend is `service.name` + `service.port.number` now — no more annotation strings.

### Task 15 — Rollout Update + Rollback
Deployment `site` is at image `nginx:1.25`. Update to `nginx:1.26` with
`kubectl set image`, watch `rollout status`, then patch to a **bad** image
`nginx:doesnotexist`, wait for it to fail, and `rollout undo` to `nginx:1.26`. Show the
ReplicaSet history (`rollout history`).

**Constraints:** Must use `set image` and `rollout undo`; no manual replica juggling.

**Hint:** `kubectl set image deployment/site site=nginx:1.26` → update → then a bad value forces `Progressing→ImagePullBackOff`; `undo` reverts to the *previous* revision.

### Task 16 — Scale + History
Scale Deployment `worker` to 7 replicas, then run two successive `set image` updates.
Show `kubectl rollout history deployment/worker` lists 3 revisions and confirm the
running image matches revision 3.

**Constraints:** All via imperative commands; `kubectl rollout history` output must show ≥3 entries.

**Hint:** Every rollout template change = +1 revision; plain `scale` (replicas change alone) does not create a depot revision but still displays.

### Task 17 — Imperative Everything
Using **only imperative kubectl commands** (no YAML files, no `apply -f`), re-create:
- Deployment `imp` (image `nginx:1.25`) in ns `imper`, 3 replicas
- Service `imp-svc` type NodePort port 80
- ConfigMap `imp-cfg` with two literals
Then verify every object in one `kubectl get` call.

**Constraints:** Zero manifest files; every action is a single command line.

**Hint:** `kubectl create deployment`, `kubectl expose deployment --type=NodePort`, `kubectl create configmap --from-literal`.

### Task 18 — limits + requests
Pod `thrifty` (image `busybox:1.36`, command `sleep 3600`) must request
`cpu 100m / memory 64Mi` and be capped at `cpu 200m / memory 128Mi`. Confirm via describe
under `Limits` and `Requests`.

**Constraints:** Set on the container; hints in `resources.requests/limits` only.

**Hint:** These make a `Burstable` QoS pod; `describe pod` prints `Limits:` and `Requests:`.

### Task 19 — Pod Anti-Affinity
Deployment `shard` (3 replicas) must **not** co-locate two pods of the same app on one
node. Add `podAntiAffinity` with `preferredDuringSchedulingIgnoredDuringExecution` and
verify each pod's `spec.nodeName` differs.

**Constraints:** Same labels it selects on; `topologyKey: kubernetes.io/hostname`.

**Hint:** `labelSelector.matchLabels: {app: shard}` + anti-affinity = spread. Check with `kubectl get po -o wide`.

### Task 20 — envFrom FK
Pod `envmix` (image `nginx:1.25`) must consume **all** keys from ConfigMap `app-config`
and Secret `app-secret` in one shot, then override `LOG_LEVEL` to `debug`. Verify
`kubectl exec ... -- env | grep LOG_LEVEL` shows `debug`.

**Constraints:** `envFrom[].configMapRef` + `envFrom[].secretRef` + one explicit `env` override.

**Hint:** Order matters in a subtle way — explicit `env` (lit/valueFrom) wins over `envFrom`, which satisfies the "override" requirement.

---

## ✅ Answer Key

### Task 01
```bash
kubectl create ns app
kubectl create deployment hello -n app --image=nginx:1.25 --replicas=2 \
  --dry-run=client -o yaml > hello.yaml
# add to hello.yaml:  containers[-1].ports: { containerPort: 80 }
kubectl apply -f hello.yaml
kubectl expose deployment hello -n app --port=80 --name=hello-svc
kubectl get endpoints hello-svc -n app        # 2 IPs
```
**Rationale:** The Service selects pods by the Deployment's `app: hello` pod label; `containerPort` is informational but required by the task. Endpoints populate only for Ready pods.

### Task 02
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: web          # named, matches container port name below
EOF
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels: { app: frontend }
spec:
  selector: { matchLabels: { app: frontend } }
  template:
    metadata: { labels: { app: frontend } }
    spec:
      containers:
      - name: web
        image: nginx:1.25
        ports:
        - name: web
          containerPort: 80
EOF
kubectl get svc frontend -o jsonpath='{.spec.ports[0].nodePort}'
```
**Rationale:** `targetPort: web` resolves to the container port named `web`. NodePort auto-allocates 30000-32767 — a memorable pointer for CKAD ("the range you can quote from memory").

### Task 03
```bash
kubectl create configmap app-config --from-literal=LOG_LEVEL=info
kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD=hunter2hunter2
kubectl run envio --image=nginx:1.25 --restart=Never --dry-run=client -o yaml > envio.yaml
# edit envio.yaml containers[0].env:
cat <<'EOF' >> /dev/null   # i.e. add under containers[0] in envio.yaml
#    env:
#    - name: LOG_LEVEL
#      valueFrom:
#        configMapKeyRef:
#          name: app-config
#          key: LOG_LEVEL
#    - name: DB_PASSWORD
#      valueFrom:
#        secretKeyRef:
#          name: app-secret
#          key: DB_PASSWORD
EOF
kubectl apply -f envio.yaml
kubectl exec envio -- env | grep -E 'LOG_LEVEL|DB_PASSWORD'
```
**Rationale:** `valueFrom.*KeyRef` binds a single key from a ConfigMap/Secret into an env var at runtime. The API server injects decoded values; no manual base64 handling needed at consume-time.

### Task 04
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: cfvol
spec:
  containers:
  - name: web
    image: nginx:1.25
    volumeMounts:
    - name: cfg
      mountPath: /etc/app
  volumes:
  - name: cfg
    configMap:
      name: app-config
EOF
kubectl exec cfvol -- cat /etc/app/LOG_LEVEL     # info
```
**Rationale:** A ConfigMap volume projects each key as a *file* named after the key at the mount path. Unlike env vars (immutable for the pod lifetime), volume-based updates eventually appear in the container.

### Task 05
```bash
kubectl create deployment ping --image=nginx:1.25 --dry-run=client -o yaml > ping.yaml
# edit ping.yaml: set command + livenessProbe
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: { name: ping, labels: { app: ping } }
spec:
  replicas: 1
  selector: { matchLabels: { app: ping } }
  template:
    metadata: { labels: { app: ping } }
    spec:
      containers:
      - name: ping
        image: nginx:1.25
        command: ["sh","-c","rm -f /usr/share/nginx/html/healthz; sleep 30; touch /usr/share/nginx/html/healthz; sleep 3600"]
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 3
        ports:
        - containerPort: 80
EOF
kubectl get po -l app=ping -w        # watch RESTARTS increment ~35-45s in
```
**Rationale:** Kubelet polls `/healthz:8080`; while the file is absent (404) for `failureThreshold` periods, it kills the container → restart counter jumps. The delay+period math means the app gets ~30s grace after `initialDelaySeconds`.

### Task 06
```bash
kubectl create deployment api --image=nginx:1.25 --replicas=1 --dry-run=client -o yaml > api.yaml
# edit api.yaml: add readinessProbe (and a service to observe endpoints)
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: { name: api, labels: { app: api } }
spec:
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      containers:
      - name: api
        image: nginx:1.25
        command: ["sh","-c","sleep 30; touch /usr/share/nginx/html/ready; sleep 3600"]
        readinessProbe:
          httpGet: { path: /ready, port: 8080 }
          periodSeconds: 5
        ports: [{ containerPort: 8080 }]
EOF
kubectl expose deployment api --port=80 --target-port=8080
kubectl get endpoints api      # 0 IPs until the probe file exists, then 1
```
**Rationale:** Readiness failure removes the pod from Service endpoints **without** restarting it. The endpoint count dip (0 → 1) is the observable proof — liveness checks would show RESTARTS; readiness won't.

### Task 07
```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: { name: slowboot }
spec:
  selector: { matchLabels: { app: slowboot } }
  template:
    metadata: { labels: { app: slowboot } }
    spec:
      containers:
      - name: slowboot
        image: nginx:1.25
        command: ["sh","-c","sleep 90; touch /usr/share/nginx/html/healthz; sleep 3600"]
        startupProbe:
          tcpSocket: { port: 8080 }
          periodSeconds: 30
          failureThreshold: 6
        livenessProbe:
          httpGet: { path: /healthz, port: 8080 }
          periodSeconds: 10
EOF
kubectl get po -l app=slowboot -w
```
**Rationale:** `startupProbe` gates every other probe: until it passes (max 30s×6 = 180s window), liveness/readiness are deferred. The slow 90s boot quietly clears startup before liveness ever fires. Without it the liveness `httpGet` during boot would kill `slowboot`.

### Task 08
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: duo
  labels: { app: duo }
spec:
  containers:
  - name: main
    image: nginx:1.25
    securityContext:
      runAsUser: 1000
    volumeMounts:
    - name: shared
      mountPath: /shared
  - name: logger
    image: busybox:1.36
    command: ["sh","-c","tail -f /shared/nginx.log"]
    volumeMounts:
    - name: shared
      mountPath: /shared
  volumes:
  - name: shared
    emptyDir: {}
EOF
kubectl exec duo -c main -- sh -c 'echo "line1" > /shared/nginx.log'
kubectl logs duo -c logger       # tail shows the line
```
**Rationale:** The sidecar pattern: two containers in one pod share an `emptyDir` volume so the logger can tail a file the main writes. `runAsUser: 1000` makes the main container non-root while the shared dir is world-writable via emptyDir's default perms.

### Task 09
```bash
kubectl create job backfill --image=busybox:1.36 -- sh -c "ps aux > /dev/null"
kubectl get job backfill -o wide     # COMPLETIONS 1/1
```
**Rationale:** `kubectl create job` wraps a Pod template with `restartPolicy: OnFailure` and `completions: 1`. One successful pod = job Complete; failures retry automatically up to `backoffLimit` (default 6).

### Task 10
```bash
kubectl create cronjob etl --image=busybox:1.36 \
  --schedule="*/5 * * * *" -- echo "run every 5 min"
kubectl patch cronjob etl --type=merge -p '{"spec":{"concurrencyPolicy":"Forbid"}}'
kubectl get cronjob etl -o wide
sleep 360; kubectl get jobs -o wide   # a completed job appears after one period
```
**Rationale:** CronJob → creates a Job each tick that the schedule matches. `Forbid` skips a run if the previous one hasn't finished (no overlapping execution); `Allow`/`Replace` are the alternatives. `--` separates kubectl args from the pod command.

### Task 11
```bash
kubectl create ns data
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cache-pvc
  namespace: data
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests: { storage: 1Gi }
EOF
kubectl get pvc cache-pvc -n data          # Bound (provisioner creates PV)
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: { name: cache, namespace: data }
spec:
  selector: { matchLabels: { app: cache } }
  template:
    metadata: { labels: { app: cache } }
    spec:
      containers:
      - name: redis
        image: redis:7
        volumeMounts: [{ name: v, mountPath: /data }]
      volumes:
      - name: v
        persistentVolumeClaim: { claimName: cache-pvc }
EOF
kubectl -n data exec deploy/cache -- bash -c 'ls -la /data'
```
**Rationale:** A StorageClass-backed PVC dynamically provisions a PV on first request (no PV manifest). On EKS that's the `gp2`/`gp3` EBS CSI default. RWO = one node at a time — correct for a single redis pod.

### Task 12
```bash
kubectl create ns ci
kubectl create sa cicd -n ci
kubectl create deployment runner -n ci --image=alpine:3.19 --replicas=1 \
  --dry-run=client -o yaml > runner.yaml
# edit runner.yaml: spec.template.spec.serviceAccountName: cicd
kubectl apply -f runner.yaml
kubectl get po -n ci -o jsonpath='{.items[0].spec.serviceAccountName}'
kubectl -n ci exec deploy/runner -- ls /var/run/secrets/kubernetes.io/serviceaccount/token
```
**Rationale:** Apps get a ServiceAccount's projected token at
`/var/run/secrets/kubernetes.io/serviceaccount/` — identity for the in-cluster API. Using `cicd` instead of `default` is the least-privilege habit CKAD expects; explicit SA names also matter for RBAC and IRSA/EKS.

### Task 13
```bash
kubectl label nodes ip-10-0-1-12.ec2.internal purpose=gpu
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata: { name: gpu-task }
spec:
  containers:
  - name: c
    image: nginx:1.25
  nodeSelector:
    purpose: gpu
---
apiVersion: v1
kind: Pod
metadata: { name: general }
spec:
  containers:
  - name: c
    image: nginx:1.25
EOF
kubectl get pods -o wide          # gpu-task.nodeName == ip-10-0-1-12...
```
**Rationale:** The scheduler treats `nodeSelector` labels as hard constraints (`RequiredDuringScheduling`). The pod with no selector schedules on any matching node — it will never be placed on the tainted/labeled one unless told to.

### Task 14
```bash
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop
spec:
  ingressClassName: nginx
  rules:
  - host: shop.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port: { number: 80 }
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port: { number: 80 }
EOF
kubectl get ingress shop -o yaml
```
**Rationale:** Ingress v1 defines backends via `service.name` + `service.port.number`. Longer `path /api` wins over `/` for `/api/*` requests (Ingress matches most-specific prefix first), so each host routes by path to the right Service.

### Task 15
```bash
kubectl set image deployment/site site=nginx:1.26
kubectl rollout status deployment/site
kubectl set image deployment/site site=nginx:doesnotexist
kubectl rollout status deployment/site         # stays Progressing (ImagePullBackOff)
kubectl rollout undo deployment/site            # reverts to revision 2 (nginx:1.26)
kubectl rollout status deployment/site
kubectl rollout history deployment/site
```
**Rationale:** `set image` triggers a new ReplicaSet (revision). The broken image stalls rollout (image pull failure) but the Deployment tracks history, so `undo` restores the previous revision's template — with rolling behavior, not a manual restore.

### Task 16
```bash
kubectl scale deployment worker --replicas=7
kubectl set image deployment/worker worker=nginx:1.24
kubectl set image deployment/worker worker=nginx:1.25
kubectl rollout history deployment/worker     # 3 revisions
kubectl get deploy worker -o jsonpath='{.spec.template.spec.containers[0].image}'
```
**Rationale:** Template changes create revisions (v1 created, v2 = 1.24, v3 = 1.25). `scale` alone doesn't add a template revision — history shows only template-changing rollouts, which is why the pair of `set image` calls yields exactly 3 entries.

### Task 17
```bash
kubectl create ns imper
kubectl create deployment imp -n imper --image=nginx:1.25 --replicas=3
kubectl expose deployment imp -n imper --type=NodePort --port=80 --name=imp-svc
kubectl create configmap imp-cfg -n imper \
  --from-literal=FIRST=one --from-literal=SECOND=two
kubectl get all -n imper
```
**Rationale:** Imperative `create`/`expose` is the CKAD speed path — every object exists without a single YAML file. `--replicas` on create, not `--scale`; NodePort via the `--type` flag on `expose`.

### Task 18
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata: { name: thrifty }
spec:
  containers:
  - name: c
    image: busybox:1.36
    command: ["sleep","3600"]
    resources:
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi
EOF
kubectl describe pod thrifty | grep -A3 -E 'Limits|Requests'
```
**Rationale:** `requests` = scheduling guarantee, `limits` = hard cap. requests≠limits → QoS class `Burstable` (middle tier); this pod could be evicted under memory pressure (unlike `Guaranteed`).

### Task 19
```bash
kubectl create deployment shard --image=nginx:1.25 --replicas=3 \
  --dry-run=client -o yaml > shard.yaml
# edit shard.yaml: add podAntiAffinity under spec.template.spec.affinity
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: { name: shard }
spec:
  replicas: 3
  selector: { matchLabels: { app: shard } }
  template:
    metadata: { labels: { app: shard } }
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels: { app: shard }
              topologyKey: kubernetes.io/hostname
      containers:
      - name: shard
        image: nginx:1.25
EOF
kubectl get po -l app=shard -o wide       # nodeName differs across the 3
```
**Rationale:** `podAntiAffinity` expresses "avoid nodes hosting pods matching X" over `topologyKey: kubernetes.io/hostname` (per-node). `preferred…` = soft (scheduler tries, won't fail); `required…` = hard. This is the reason "replicas spread across nodes" works.

### Task 20
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata: { name: envmix }
spec:
  containers:
  - name: web
    image: nginx:1.25
    envFrom:
    - configMapRef: { name: app-config }
    - secretRef: { name: app-secret }
    env:
    - name: LOG_LEVEL
      value: debug
EOF
kubectl exec envmix -- env | grep LOG_LEVEL     # debug (override wins)
```
**Rationale:** `envFrom` bulk-imports *every* key from the ConfigMap/Secret as env vars, while the explicit `env` entry for `LOG_LEVEL` takes precedence — an explicit assignment overrides a same-key `envFrom` entry (and avoids the configmap-key collision footgun).

---

**Scoring:** 18/20 with working probes + exact API shapes = you're competitive. Redo every
failed task once more and keep the flashcard groups warm. 🚀