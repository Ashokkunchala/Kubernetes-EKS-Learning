# 🧩 Module H: Advanced Scheduling — Affinity, Topology, Priority, Preemption

> **Prereq:** [Day 02 Pods & Deployments](../Day-02_Pods_ReplicaSets_Deployments/README.md) · [Day 15 EKS Security](../Day-15_EKS_Security_Best_Practices/README.md)

Basic scheduling just needs "a node with enough CPU/memory." Real clusters need *where* pods land, *how spread* they are for resilience, and *which* workloads win when resources get tight. This module is the deep dive on everything past `nodeSelector`.

## 🎯 Learning Objectives

By the end of this module, you will understand:
- `nodeSelector` vs node affinity vs pod affinity — and when each is appropriate
- Required (`requiredDuringScheduling`) vs preferred (`preferredDuringScheduling`) affinity
- Pod anti-affinity and how it spreads replicas across AZs for HA
- Taints and tolerations — the "stickiness" model and real EKS use cases
- `topologySpreadConstraints` — the modern way to spread pods evenly
- `PriorityClass` and preemption — what gets evicted when nodes are full
- The scheduler decision path and how a pod can end up unschedulable

---

## 📖 Core Concepts

### The scheduling decision path (simplified)

```
Pod created
   │
   ▼
Filtering (feasibility):  does ANY node satisfy the pod's hard requirements?
   │  - resources fit?  nodeSelector?  required affinity?  taints tolerated?
   ▼
Scoring:  pick the best of the feasible nodes
   │  - preferred affinity, topology spread, resource fit, inter-pod
   ▼
Bind  →  kubelet brings the pod up
```

Hard rules **filter**; soft rules **score**. A pod with unsatisfiable hard rules stays `Pending` with a `FailedScheduling` event.

### The affinity ladder

```
nodeSelector        → match on node LABELS only (exact key=value)
node affinity       → required/preferred, matchExpressions, weight
pod affinity        → co-locate with OTHER pods that match
pod anti-affinity   → avoid other pods that match (spread for HA)
```

```
┌─────────────────────────────────────────────────────────────┐
│  nodeSelector vs NODE AFFINITY                               │
│                                                             │
│  nodeSelector:                                              │
│    spec.nodeSelector: { "node.kubernetes.io/instance-type":  │
│                         "m5.large" }                        │
│     → exact equality on labels; AND of all entries           │
│                                                             │
│  nodeAffinity (required):                                   │
│    matchExpressions: [{ key: zone, operator: In,            │
│                         values: [us-east-1a] }]             │
│     → richer operators: In/NotIn/Exists/DoesNotExist/Gt/Lt  │
│                                                             │
│  nodeAffinity (preferred):                                  │
│    + weight: 100  → a *soft* scoring preference             │
└─────────────────────────────────────────────────────────────┘
```

### Taints & tolerations — the stickiness model

A **taint** on a node repels pods that don't have a matching **toleration**. Tolerations are the *permission to schedule*; they don't force placement.

```
Taint on node:   key=value:effect
  effect ∈ { NoSchedule, PreferNoSchedule, NoExecute }

toleration:      key=value:operator (Equal | Exists), effect
  operator Exists → tolerates any value for that key
```

**NoExecute** is special: it not only blocks *new* scheduling, it **evicts** pods already running on the node that lack the toleration (with `tolerationSeconds` to delay eviction).

### Topology spread constraints

The modern, balanced way to distribute pods instead of hand-rolled anti-affinity. You define a `topologyKey` (e.g. `topology.kubernetes.io/zone`) and a max skew.

```
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule     # (or ScheduleAnyway)
  labelSelector: { matchLabels: { app: web } }

Desired: even pods across AZs, at most 1 apart (maxSkew).
```

### PriorityClass & preemption

`PriorityClass` (cluster-scoped) sets a numeric priority. Higher priority pods are scheduled first, and can **preempt** (evict) lower-priority pods when there is no room.

```
PriorityClass "critical":   value: 1000000   (system-critical apps)
PriorityClass "high":       value: 1000
PriorityClass "low":        value: 100

If a high-priority pod can't fit:
  1. scheduler finds lower-priority pods to evict
  2. it names a "nominated node" and evicts them
  3. the preempting pod schedules

Guardrails:
  - globalDefault: true  on exactly ONE class
  - preemptionPolicy: Never   to disable preemption for a class
  - PDBs + priorities interplay: PDBs protect the *victim* side
```

---

## 🚀 Hands-On

### Step 1: Node affinity — require a GPU node pool, prefer spot

```yaml
# node-affinity-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-job
  labels:
    app: ml
spec:
  containers:
  - name: ml
    image: nvidia/cuda:12.1-base
    command: ["sleep", "3600"]
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node.kubernetes.io/gpu
            operator: Exists          # must be a GPU node
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: node.kubernetes.io/lifecycle
            operator: In
            values: ["spot"]          # prefer spot if available
```

### Step 2: Pod anti-affinity — spread replicas across AZs (HA)

```yaml
# anti-affinity-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector: { matchLabels: { app: web } }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
      - name: web
        image: nginx:1.27-alpine
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels: { app: web }
              topologyKey: topology.kubernetes.io/zone
              # spreads "web" pods across different AZs
```

> For strict HA use `requiredDuringScheduling` — but be careful: if you have fewer AZs than replicas, the extra replicas will remain `Pending`.

### Step 3: Taints & tolerations

```bash
# Taint a node so only specific workloads use it
kubectl taint nodes node-a dedicated=search:NoSchedule

kubectl get nodes -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\n'
```

```yaml
# toleration-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: search-pod
spec:
  containers:
  - name: search
    image: nginx:1.27-alpine
  tolerations:
  - key: dedicated
    operator: Exists          # tolerate any value of 'dedicated'
    effect: NoSchedule
  # without this toleration the pod would stay Pending
```

```bash
# NoExecute evicts non-toleration pods already running:
kubectl taint nodes node-a reschedule=true:NoExecute
#   → pods without the matching toleration are evicted
kubectl taint nodes node-a reschedule=true:NoExecute-    # remove taint
```

### Step 4: Topology spread — even distribution

```yaml
# topology-spread.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
spec:
  replicas: 6
  selector: { matchLabels: { app: worker } }
  template:
    metadata:
      labels: { app: worker }
    spec:
      containers:
      - name: worker
        image: busybox
        command: ["sleep", "3600"]
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels: { app: worker }
```

### Step 5: Priority classes

```yaml
# priorityclasses.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical
value: 1000000
globalDefault: false
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low
value: 100
globalDefault: false
```

```yaml
# deployment-with-priority.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cashier
spec:
  replicas: 2
  selector: { matchLabels: { app: cashier } }
  template:
    metadata:
      labels: { app: cashier }
    spec:
      priorityClassName: critical     # ← ties into the class above
      containers:
      - name: cashier
        image: nginx:1.27-alpine
```

---

## 🛠️ Troubleshooting

```bash
# 1. Pod stuck Pending → FailedScheduling
kubectl describe pod <pod> | grep -A5 Events
#   "0/5 nodes are available: 1 node(s) didn't match pod anti-affinity,
#    2 node(s) had taint {key:value:NoSchedule} that the pod didn't tolerate,
#    2 insufficient cpu"

# 2. Is it hard vs soft?  If a *preferred* rule blocks, the pod shouldn't Pending —
#    Pending with FailedScheduling almost always means a *required* rule is unmet
#    or a taint lacks a toleration.

# 3. Taint too aggressive?  Remove it:
kubectl taint nodes <node> key=value:NoSchedule-

# 4. Replicas stuck because maxSkew + insufficient nodes:
#    if scheduleAnyway is DoNotSchedule and 2 AZs < 6 replicas evenly,
#    use whenUnsatisfiable: ScheduleAnyway for best-effort spread.

# 5. Preemption not happening as expected?
#    check priorityClassName exists; preemptionPolicy: Never disables it;
#    PDBs can block eviction of the nominator's victims.
```

---

## ❓ Interview Questions

1. **`nodeSelector` vs node affinity vs pod affinity — when?**
   - `nodeSelector`: simple exact-label match. Node affinity: richer operators + required/preferred weights on node labels. Pod affinity/anti-affinity: relationships to *other pods* — co-locate (affinity) or spread apart (anti-affinity) for HA.

2. **What's the difference between `requiredDuringScheduling` and `preferredDuringScheduling`?**
   - Required is a hard filter (node must match or pod stays Pending); preferred is a soft scoring tie-breaker (matches weigh more but the pod still schedules elsewhere).

3. **How do taints and tolerations work together?**
   - A taint repels pods that lack a matching toleration; a toleration grants permission to schedule (for `NoSchedule`) or to keep running (for `NoExecute`, which otherwise evicts).

4. **When would you prefer `topologySpreadConstraints` over pod anti-affinity?**
   - Anti-affinity with a topologyKey spreads pods but doesn't enforce a *balanced count*; topology spread constraints force an even "skew" (e.g. max 1 apart) across topology domains, ideal for HA and fair distribution.

5. **How does priority-based preemption work, and what protects the victims?**
   - Higher-priority pods can evict lower-priority ones to free room (nominated node). Victim protection comes from PodDisruptionBudgets, and you can set `preemptionPolicy: Never` to forbid preemption entirely for a class.

6. **Why might a pod be `Pending` with all nodes looking fine?**
   - An unmet hard rule: taint without toleration, unsatisfiable required affinity, insufficient resources, PVC not bound, or a `maxSkew`/`DoNotSchedule` topology constraint — check `kubectl describe pod` events for `FailedScheduling`.

---

## 📚 Resources

- [Assigning Pods to Nodes (official)](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)

---

## ✅ Progress Tracker

```
Module H: Advanced Scheduling
  [x] Filtering vs scoring (hard vs soft) diagram
  [x] Affinity ladder (nodeSelector → node → pod)
  [ ] Required node affinity: GPU node pool
  [ ] Preferred node affinity: spot preference
  [ ] Pod anti-affinity: AZ spread (HA)
  [ ] Taint + toleration demo (NoSchedule + NoExecute)
  [ ] Topology spread constraints (even distribution)
  [ ] PriorityClass + observed preemption
  [ ] Troubleshoot a Pending pod end-to-end
```

---

## 👀 Next

- **Sandbox / security-context the pods you just scheduled:** [Module J Runtime Security](./Module-J_Runtime_Security.md)
- **Combine taints + node groups with autoscaling:** [Day 53 Cluster Autoscaler vs Karpenter](../Day-53_Cluster_Autoscaler_Karpenter/README.md)
- **PDBs that protect your preferred-spread replicas:** [Day 02 Pods & Deployments](../Day-02_Pods_ReplicaSets_Deployments/README.md)
- **Spread + HA in a full production pattern:** [ARCHITECTURE_PATTERNS.md](../../ARCHITECTURE_PATTERNS.md)

---

**Remember:** Every pod you run makes a scheduling promise — hard rules it cannot break and soft rules it prefers. Design the promise before the pod lands. 🎯
