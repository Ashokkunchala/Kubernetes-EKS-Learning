# 🧪 CKA Practice Exam — Part 1 (Performance Tasks)

20 cluster-administrator tasks in CKA style: solve each **in a live terminal, from memory**.
No multiple choice. Each task tells you the desired end state; many include a broken or
partial snippet you must fix. Settings: `kubectl` context is already pointed at the
cluster; `vim` is available. Timebox: **~2.5 hours total**, then grade against the answer key.

---

## 📝 Tasks

### Task 01 — Namespace + Deployment
Create a namespace `team-a`. Inside it create a Deployment named `web` running image
`nginx:1.25` with **3 replicas**. Label the whole workload `tier=frontend`. Confirm all
pods are running before moving on.

**Constraints:** Use a manifest file (not pure imperative) for the Deployment.

**Hint:** `kubectl create ns team-a` then write `web.yaml` with `apiVersion: apps/v1`.

### Task 02 — Scale Imperatively
Scale the Deployment `web` from Task 01 to **6 replicas** with a single imperative
command. Then verify the ReplicaSet count.

**Constraints:** Do not edit the YAML file to scale.

**Hint:** `kubectl scale deployment web -n team-a --replicas=6`.

### Task 03 — Expose as Service
Expose Deployment `web` (namespace `team-a`, port 80) as a **ClusterIP** service named
`web-svc`. Verify endpoints are populated and resolving via CoreDNS
(`nslookup web-svc.team-a.svc.cluster.local` from a pod).

**Constraints:** ClusterIP only — no NodePort, no LoadBalancer.

**Hint:** `kubectl expose deployment web -n team-a --port=80` gives you ClusterIP by default.

### Task 04 — Taint + Toleration
Given node `ip-10-0-1-10.ec2.internal`, add the taint `dedicated=ci:NoSchedule`. Then edit
Deployment `ci-runner` (image `busybox:1.36`, command `sleep 3600`) so its pods can
actually land on that node. Show the pod is running with a toleration.

**Constraints:** The toleration must appear in the deployed manifests; do not remove the taint.

**Hint:** `kubectl taint nodes <node> dedicated=ci:NoSchedule`, then add a `tolerations:`
block matching `key=dedicated, operator=Equal, value=ci, effect=NoSchedule`.

### Task 05 — ConfigMap Volume Mount
Create a ConfigMap `app-config` (namespace `team-a`) with key `APP_ENV=production` from
literal values. Add it to Deployment `web` as a **volume mount** at `/etc/config`, so a
file `APP_ENV` appears inside the container. Verify with `kubectl exec`.

**Constraints:** Must be a volume mount, not an `env:` mapping.

**Hint:** `kubectl create configmap app-config -n team-a --from-literal=APP_ENV=production`.

### Task 06 — RBAC: Fix Alice
User `alice` can authenticate but gets `Forbidden` listing pods in `team-a`. Create the
minimum RBAC objects so `alice` can `get`, `list`, and `watch` pods **only in** `team-a`.

**Constraints:** Scoped to the namespace; do not grant cluster-wide rights.

**Hint:** Role + RoleBinding (namespaced). Put the role name in `resourceNames` only if asked.

### Task 07 — NetworkPolicy Default-Deny
In namespace `secure`, apply a NetworkPolicy that **denies all ingress** except from pods
labeled `app=api`. Create a testing pod with that label and one without to prove the rule.

**Constraints:** `apiVersion: networking.k8s.io/v1`; policy must also allow the pods to
talk to CoreDNS (`kube-system` namespace) for `kubectl exec` based checks to be meaningful.

**Hint:** `podSelector: matchLabels: {app: api}` plus an ingress rule from `kube-system`.

### Task 08 — Drain (Maintenance)
Cordone node `ip-10-0-1-10.ec2.internal`, then drain it so pods evict gracefully. Then
`uncordon` it. During the drain, show that `team-a/web` pods were rescheduled onto other
nodes.

**Constraints:** Use `--ignore-daemonsets`; deletion of the `web` pods is expected but must
be **foreground/deletion-safe** (do not pass `--force`).

**Hint:** `kubectl cordon <node>` → `kubectl drain <node> --ignore-daemonsets` →
`kubectl uncordon <node>`.

### Task 09 — etcd Backup
Back up etcd to `/opt/etcd-backup.db` using `etcdctl`, with the API version and auth flags
shown in the cluster's static pod manifest (`/etc/kubernetes/manifests/etcd.yaml`) or
`kubectl get po -n kube-system -o yaml`. Then verify `snapshot status` shows a valid file.

**Constraints:** Must read the real endpoint/cert paths from the running pod; no hardcoding.

**Hint:** `ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=... --cert=... --key=... snapshot save /opt/etcd-backup.db`.

### Task 10 — PVC + Pod
Create a StorageClass `manual` with `provisioner: kubernetes.io/no-provisioner` and
`volumeBindingMode: WaitForFirstConsumer`. Create a PV `pv-data` (5Gi, ReclaimPolicy
`Retain`) backed by a local hostPath on control-plane, then a PVC claiming it in `team-a`,
then a pod `consumer` mounting the PVC at `/data`. Confirm the PV `Released`/`Bound` status.

**Constraints:** `volumeBindingMode: WaitForFirstConsumer` must be set on the StorageClass.

**Hint:** With waiting first consumer the pod forces binding — create the pod, then check.

### Task 11 — Job Failure Handling
Create a Job `trouble` (image `busybox:1.36`) that runs `sleep 2; exit 1`. It must retry at
most **4 times** and stop completely after. Inspect events; after completion the pods stay
for debugging — ensure `ttlSecondsAfterFinished: 30` cleans them up.

**Constraints:** `backoffLimit: 4`, `restartPolicy: Never`, `apiVersion: batch/v1`.

**Hint:** `kubectl create job trouble --image=busybox:1.36 -- sleep 2; exit 1` then patch
in the fields; watch `kubectl get job` backoff counting to 4.

### Task 12 — DaemonSet
Create a DaemonSet `fluentd-agent` (image `fluentd:latest`, container name `agent`) running
on **every node**, mounted with the host's `log` directory at `/var/log` (hostPath). Verify
it schedules on all nodes and there is no `availableReplicas` gap.

**Constraints:** Must use hostPath `/var/log` **and** `hostPID: true` is not allowed for
this task — daemonset with hostPath only.

**Hint:** `kubectl create deployment` then convert, or just author the DaemonSet manifest with `apiVersion: apps/v1`.

### Task 13 — Diagnostic: Image Pull
Deployment `broken` (image `nginx:doesnotexist`, 1 replica) is in `ImagePullBackOff`.
Use `kubectl describe` + events to identify the cause, change the image to `nginx:1.25`,
and confirm `Available`.

**Constraints:** Diagnose with real events/logs; do not delete/recreate the Deployment.

**Hint:** `kubectl set image deployment/broken broken=nginx:1.25` after confirming the error.

### Task 14 — nodeSelector
Add label `disktype=ssd` to node `ip-10-0-1-11.ec2.internal`. Schedule a pod `pin` that
**only** runs on that node via `nodeSelector`. Verify `NODE` field of the pod matches.

**Constraints:** No taints involved; nodeSpec label must exist before scheduling (or use `--overwrite`).

**Hint:** `kubectl label nodes <node> disktype=ssd` → pod manifest with
`nodeSelector: { disktype: ssd }`.

### Task 15 — Sidecar Shared Volume
Create a multi-container pod `webapp` (namespace `team-a`) with:
- `frontend` container: image `nginx:1.25`, writes logs to `/var/log/nginx`.
- `sidecar` container: image `busybox:1.36`, command `tail -f /var/log/nginx/access.log`.
Both containers share an `emptyDir` mount at `/var/log/nginx`. Confirm `logs` appear in
sidecar output.

**Constraints:** One pod manifest; emptyDir with `medium: Memory` for a fair bonus.

**Hint:** `spec.volumes` + identical `volumeMounts` mountPath on both containers.

### Task 16 — Static Pod Verification
The kubelet managed static pod directory is `/etc/kubernetes/manifests/`. Verify the
control-plane static pods (kube-apiserver, etcd, kube-scheduler, controller-manager) are
all running, and confirm one (kube-apiserver) is managed by kubelet, not a Deployment,
by checking its owner reference.

**Constraints:** Read, don't modify. Evidence must be a `kubectl get pod -n kube-system -o jsonpath` showing owner.

**Hint:** Static pods are suffixed with the node hostname.

### Task 17 — Ingress
Create an Ingress `web-ingress` (namespace `team-a`, `ingressClassName: nginx` optional)
routing host `web.example.com` and path `/` to Service `web-svc:80`. Verify the rule
renders correctly with `kubectl get ingress -o yaml`.

**Constraints:** `apiVersion: networking.k8s.io/v1`; the Ingress must specify `pathType: Prefix`.

**Hint:** Ingress rule → `http.paths[].backend.service.port.number: 80`.

### Task 18 — Resource Limits (Live Update)
Deployment `draincheck` (namespace `team-a`) currently has no limits. Update the running
manifest so every container requests `cpu: 200m, memory: 512Mi` and limits
`cpu: 500m, memory: 1Gi`. The change must not delete the Deployment.

**Constraints:** Use `kubectl edit` or a patch; roll out must be zero-downtime
(`maxUnavailable: 0` in strategy optional).

**Hint:** `kubectl edit deployment draincheck -n team-a`, add `resources:` under `containers[0]`.

### Task 19 — StatefulSet + Headless
Create a headless Service `db` (port 5432) and a StatefulSet `postgres` (image
`postgres:16`, 2 replicas) using the headless service for stable network identities.
Then create a `pgbench` pod in the same namespace and resolve
`postgres-1.db.<ns>.svc.cluster.local`.

**Constraints:** `serviceName: db` in the StatefulSet spec; `clusterIP: None` on the Service.

**Hint:** `kubectl get pods -o wide` shows ordinal pods `postgres-0`, `postgres-1`.

### Task 20 — Quota + LimitRange
Create namespace `capacity`. Add a `ResourceQuota` capping pods at 3, `cpu` requests at
2, and `memory` requests at 4Gi. Add a `LimitRange` defaulting container requests to
`cpu 100m / memory 256Mi` and limits `cpu 1 / memory 1Gi`. Prove quota blocks a 4th pod.

**Constraints:** Both objects must exist after `kubectl get`, and the 4th deployment must
Actually fail with a quota exception.

**Hint:** `kubectl create namespace capacity` → apply quota+limitrange YAML → create
4 small deployments, watch the 4th get `exceeded quota` in describe/events.

---

## ✅ Answer Key

### Task 01
```bash
kubectl create ns team-a
cat <<'EOF' > web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: team-a
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        tier: frontend
    spec:
      containers:
      - name: web
        image: nginx:1.25
        ports:
        - containerPort: 80
EOF
kubectl apply -f web.yaml
kubectl get deploy,pods -n team-a
```
**Rationale:** Namespace scopes the workload; Deployment (apps/v1) owns ReplicaSet+rollout; `tier` label carried on pod template so the Service later selects on it.

### Task 02
```bash
kubectl scale deployment web -n team-a --replicas=6
kubectl get deploy web -n team-a -o jsonpath='{.spec.replicas}'
kubectl get rs -n team-a
```
**Rationale:** Imperative scaling changes `.spec.replicas`; the ReplicaSet controller reconciles pod count. `scale` writes through the Deployment, not the RS directly.

### Task 03
```bash
kubectl expose deployment web -n team-a --port=80 --name=web-svc
kubectl get svc web-svc -n team-a
kubectl get endpoints web-svc -n team-a          # should list 6 pod IPs
kubectl run nslk -it --rm --image=busybox:1.36 --restart=Never -n team-a \
  -- nslookup web-svc.team-a.svc.cluster.local
```
**Rationale:** Default Service type is ClusterIP; endpoints mirror Ready pod IPs selected by `app: web`. CoreDNS resolves `web-svc.team-a.svc.cluster.local`.

### Task 04
```bash
kubectl taint nodes ip-10-0-1-10.ec2.internal dedicated=ci:NoSchedule
kubectl edit deployment ci-runner -n team-a      # add:
#   tolerations:
#   - key: "dedicated"
#     operator: "Equal"
#     value: "ci"
#     effect: "NoSchedule"
kubectl get po -n team-a -o wide
```
**Rationale:** `NoSchedule` rejects new unscheduled pods without a matching toleration. The toleration must match `key/value/effect` exactly; `Equal` avoids wildcards so pods only land on `dedicated=ci` nodes.

### Task 05
```bash
kubectl create configmap app-config -n team-a \
  --from-literal=APP_ENV=production
kubectl edit deployment web -n team-a            # add under container web:
#   volumeMounts:
#   - name: config
#     mountPath: /etc/config
# (and under spec:)
#   volumes:
#   - name: config
#     configMap:
#       name: app-config
kubectl exec -n team-a deploy/web -- cat /etc/config/APP_ENV
```
**Rationale:** A ConfigMap volume exposes each key as a file named after the key. Mounting beats env-injection when the app re-reads config, and updates are (eventually) picked up without recreating the container.

### Task 06
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: team-a
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-pod-reader
  namespace: team-a
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl auth can-i get pods -n team-a --as alice
```
**Rationale:** Role/RoleBinding are namespace-scoped — exactly the "only in team-a" ask. `kubectl auth can-i --as` is the proper verification tool (it works for a static-user subject even if the user has no kubeconfig).

### Task 07
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-only
  namespace: secure
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: TCP
      port: 53
EOF
kubectl apply -f - <<'YOAML'   # 2 test pods: allowed one + denied one
apiVersion: v1
kind: Pod
metadata: { name: allowed, namespace: secure, labels: { app: api } }
spec:
  containers:
  - name: c
    image: nginx:1.25
---
apiVersion: v1
kind: Pod
metadata: { name: denied, namespace: secure, labels: { app: other } }
spec:
  containers:
  - name: c
    image: nginx:1.25
YOAML
```
**Rationale:** NetworkPolicy v1 is additive to the CNI; `podSelector` targets pods it applies **to**, and `ingress.from` governs who may reach them. The `namespaceSelector` opening UDP/TCP 53 keeps CoreDNS working for `kubectl exec`/lookup-based validation. Default-deny is implied for anything not matched.

### Task 08
```bash
kubectl cordon ip-10-0-1-10.ec2.internal
kubectl drain ip-10-0-1-10.ec2.internal --ignore-daemonsets
kubectl get po -n team-a -o wide          # web pods on other nodes now
kubectl uncordon ip-10-0-1-10.ec2.internal
```
**Rationale:** `cordon` blocks new scheduling; `drain` evicts (graceful) pods, honoring PDBs; daemonsets are skipped via `--ignore-daemonsets`. No `--force` — forced drain skips graceful termination and would delete pods with local data.

### Task 09
```bash
kubectl get pod -n kube-system -l component=etcd -o jsonpath='{.items[0].spec.containers[0].command}' 
# read the real values, e.g.:
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db
ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db
```
**Rationale:** CKA always expects you to discover endpoint/cert paths from the live static pod (they change per cluster). `snapshot save` is a point-in-time copy of etcd; `snapshot status` validates the file before you trust it. (Restore would use `snapshot restore --data-dir=...` + kubelet/etcd restart.)

### Task 10
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: manual
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity: { storage: 5Gi }
  volumeMode: Filesystem
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /opt/data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
  namespace: team-a
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests: { storage: 4Gi }
  storageClassName: manual
---
apiVersion: v1
kind: Pod
metadata:
  name: consumer
  namespace: team-a
spec:
  volumes:
  - name: data
    persistentVolumeClaim: { claimName: pvc-data }
  containers:
  - name: c
    image: nginx:1.25
    volumeMounts:
    - name: data
      mountPath: /data
EOF
kubectl get pv pv-data,pvc pvc-data -n team-a
```
**Rationale:** `no-provisioner` + `WaitForFirstConsumer` means the PV binds only once the pod actually mounts the PVC — a classic static/local storage wiring. `Retain` keeps data after PVC deletion so the operator recovers it manually.

### Task 11
```bash
kubectl create job trouble --image=busybox:1.36 --restart=Never -- sleep 2; exit 1
# ^ that only sets -- : use a manifest for control instead:
kubectl create job trouble --image=busybox:1.36 --dry-run=client -o yaml > job.yaml
# edit job.yaml: add backoffLimit: 4, ttlSecondsAfterFinished: 30, restartPolicy: Never
kubectl apply -f job.yaml
kubectl get job trouble -o wide               # backoffLimit reached → Failed
kubectl get po -l job-name=trouble -o wide    # 5 pod failures (1 success none)
```
**Rationale:** `batch/v1` Jobs retry a failed pod until `backoffLimit` (default 6) — here overridden to 4. `restartPolicy: Never` inside a Pod template forces the Job controller (not kubelet) to relaunch; `ttlSecondsAfterFinished` garbage-collects completed Jobs automatically.

### Task 12
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-agent
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: fluentd-agent
  template:
    metadata:
      labels:
        name: fluentd-agent
    spec:
      tolerations:
      - operator: Exists                       # run on control-plane too
      containers:
      - name: agent
        image: fluent/fluentd:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
EOF
kubectl get ds fluentd-agent -n kube-system    # node count == desired
kubectl get po -n kube-system -o wide -l name=fluentd-agent
```
**Rationale:** A DaemonSet runs one pod per node by default — the correct vehicle for log shipping. The `Exists` toleration (no value, matches any taint) is what lets it also schedule on tainted control-plane nodes. hostPath mounts the node's real log directory read-only.

### Task 13
```bash
kubectl get events                                      # ImagePullBackOff "manifest unknown"
kubectl describe deployment broken
kubectl describe pod -l app=broken                      # event: Failed to pull nginx:doesnotexist
kubectl set image deployment/broken broken=nginx:1.25
kubectl rollout status deployment/broken
kubectl get deploy broken -o wide                       # READY 1/1 AVAILABLE
```
**Rationale:** Diagnosis comes from events (`Failed to pull image ... not found`) not guessing. `kubectl set image` is declarative-in-place: it creates a new ReplicaSet and rolls with zero manual deletions.

### Task 14
```bash
kubectl label nodes ip-10-0-1-11.ec2.internal disktype=ssd
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pin
spec:
  containers:
  - name: c
    image: nginx:1.25
  nodeSelector:
    disktype: ssd
EOF
kubectl get pod pin -o jsonpath='{.spec.nodeName}'
```
**Rationale:** `nodeSelector` is a hard scheduling constraint — the scheduler only considers nodes labeled `disktype=ssd`. Label the node **before** creating the pod or it stays `Pending`.

### Task 15
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: webapp
  namespace: team-a
spec:
  containers:
  - name: frontend
    image: nginx:1.25
    volumeMounts:
    - name: shared
      mountPath: /var/log/nginx
  - name: sidecar
    image: busybox:1.36
    command: ["sh", "-c", "tail -f /var/log/nginx/access.log"]
    volumeMounts:
    - name: shared
      mountPath: /var/log/nginx
  volumes:
  - name: shared
    emptyDir: {}
EOF
kubectl logs -n team-a webapp -c sidecar --tail=5     # access.log lines flowing
kubectl exec -n team-a webapp -c frontend -- curl -s localhost >/dev/null
kubectl logs -n team-a webapp -c sidecar --tail=3     # new entries appear
```
**Rationale:** Both containers mount the same `emptyDir` volume at the same path — that's the sidecar pattern. nginx writes access.log into the shared volume; the busybox tail reads it. Bonus accepted: `emptyDir: { medium: Memory }` gives faster (RAM) sharing, at the cost of data loss across node restart.

### Task 16
```bash
kubectl get pod -n kube-system -o wide
kubectl get pod kube-apiserver-<hostname> -n kube-system \
  -o jsonpath='{.metadata.ownerReferences[].kind}'
```
**Rationale:** Control-plane components run as static pods under `/etc/kubernetes/manifests/`. Evidence of kubelet-managed ownership: the pod has **no** ownerReferences (returns empty), is named `<component>-<nodehostname>`, and the kubelet restarts it on file change — no Deployment/ReplicaSet owns it.

### Task 17
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: team-a
spec:
  ingressClassName: nginx
  rules:
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
EOF
kubectl get ingress web-ingress -n team-a -o yaml
```
**Rationale:** `networking.k8s.io/v1` is the only supported Ingress API in modern clusters; the backend is a `service.port.number` (or `.name`). `pathType: Prefix` (vs `Exact`) matches `/anything`. In EKS the `default`/ELB ingress controller is configured via `ingressClassName`.

### Task 18
```bash
kubectl edit deployment draincheck -n team-a
# under containers[0] add:
#   resources:
#     requests:
#       cpu: 200m
#       memory: 512Mi
#     limits:
#       cpu: 500m
#       memory: 1Gi
kubectl rollout status deployment/draincheck -n team-a
kubectl describe deploy draincheck -n team-a   # verify requests/limits applied
```
**Rationale:** `kubectl edit` writes through live objects declaratively; a template change triggers a rolling update (new RS → new pods → old RS scaled to 0), which is zero-downtime by default (maxSurge/maxUnavailable balanced).

### Task 19
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: db
  namespace: team-a
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: team-a
spec:
  serviceName: db
  replicas: 2
  selector:
    matchLabels: { app: postgres }
  template:
    metadata:
      labels: { app: postgres }
    spec:
      containers:
      - name: postgres
        image: postgres:16
        ports:
        - containerPort: 5432
EOF
kubectl run pgbench -it --rm --image=postgres:16 --restart=Never -n team-a \
  -- bash -c 'getent hosts postgres-1.db.team-a.svc.cluster.local'
```
**Rationale:** Headless (`clusterIP: None`) + `serviceName: db` gives each pod a stable DNS record `<pod>.<svc>.<ns>.svc.cluster.local` with its ordinal index (postgres-0, postgres-1). That's how databases discover each other peer-to-peer instead of load-balanced round-robin.

### Task 20
```bash
kubectl create ns capacity
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cap-quota
  namespace: capacity
spec:
  hard:
    pods: "3"
    requests.cpu: "2"
    requests.memory: 4Gi
---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: capacity
spec:
  limits:
  - default:
      cpu: "1"
      memory: 1Gi
    defaultRequest:
      cpu: 100m
      memory: 256Mi
    type: Container
EOF
for i in 1 2 3 4; do kubectl create deployment d$i -n capacity --image=nginx:1.25; done
kubectl describe deploy d4 -n capacity | tail -20   # "exceeded quota"
kubectl get quota -n capacity
```
**Rationale:** `LimitRange` injects defaults at admission time so the quota's CPU/memory counts are predictable. Quota is enforced by the API server during admission — the 4th deployment hits `pods` and `requests.cpu/memory` limits and ReplicaSet creation fails with an `exceeded quota` error visible in describe/events.

---

**Scoring:** 18/20 clean = exam-ready territory. Missed any? Rebuild the failing task twice more, timed, from scratch. 🚀