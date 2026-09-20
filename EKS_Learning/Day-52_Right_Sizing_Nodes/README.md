# 📅 Day 52: Right-Sizing Nodes

## 🎯 Learning Objectives
By the end of today, you will understand:
- How to right-size EKS worker nodes
- CPU/memory analysis with metrics
- Picking instance families by workload profile
- Setting pod resource requests/limits correctly
- Continuous right-sizing with Kubernetes tools

---

## 📖 Core Concepts

### Right-Sizing = Match Capacity to Demand

```
┌───────────────────────────────────────────────────────────────┐
│                  RIGHT-SIZING FLOW                             │
│                                                               │
│   Measure ──▶ Analyze ──▶ Adjust ──▶ Validate loop            │
│     │            │           │           │                    │
│   CPU/mem     Utilization   instance     re-monitor          │
│   (Prom./Kubecost)  per   type + pod   cost/avail change     │
│                      workload    requests/limits              │
│                                                               │
│   GOALS:                                                      │
│    · No wasted node capacity (over-provisioning)              │
│    · No pod scheduling failures (under-provisioning)          │
│    · Pods packed efficiently (bin-packing)                    │
└───────────────────────────────────────────────────────────────┘
```

### Instance Family by Workload

| Profile | Instance families | Use case |
|---------|-------------------|----------|
| General purpose | m5, m6i, m7i | Balanced apps |
| Compute | c5, c6i, c7i | CPU-heavy services |
| Memory | r5, r6i, r5a | Caches, in-memory |
| Burstable | t3, t3a | Low/steady CPU |
| GPU | g4dn, g5 | ML/inference |

---

## 🛠️ Step 1: Measure Utilization

```bash
# Enable metrics-server for basic CPU/mem
kubectl top nodes
kubectl top pods -n production

# Deep metrics via Prometheus queries (Day 41)
# Per-node CPU utilization
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Per-node memory
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

# Pod CPU vs requests
sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m])) by (pod) /
    sum(kube_pod_container_resource_requests{resource="cpu",namespace="production"}) by (pod)
```

---

## 🛠️ Step 2: Analyze & Decide

Look for:
- **Node CPU > 70-80% sustained** → too small OR pods mis-packed.
- **Node memory > 80% sustained** → risk of OOM; size up or spread.
- **Low utilization (< 30%)** → over-provisioned, shrink or consolidate.
- **Pods with no requests/limits** → cannot schedule efficiently; add them.

**Compute the ideal instance size:**

```
Total node allocatable CPU  = (sum cpu requests + sum cpu usage headroom)
Required nodes = ceil(workload_cpu_demand / node_cpu_capacity × packing_target)
```

---

## 🛠️ Step 3: Right-Size Pod Requests/Limits

Start from measured usage, add headroom for spikes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: api, namespace: production }
spec:
  template:
    spec:
      containers:
      - name: api
        image: myorg/api:v2
        resources:
          requests:
            cpu: "250m"      # measured avg ~200m, headroom
            memory: "256Mi"
          limits:
            cpu: "1"
            memory: "512Mi"
```

**Tool-assisted:**
- **Vertical Pod Autoscaler (VPA)** recommends requests from historical usage:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata: { name: api-recommender, namespace: production }
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: Initial   # only recommend, don't auto-update
```

---

## 🛠️ Step 4: Right-Size the Node Instance Type

Change the node group instance type:

```yaml
managedNodeGroups:
- name: workers
  instanceType: m5.large   # was m5.xlarge
  minSize: 2
```

Or roll to a better-size with zero downtime via a new node group labeled correctly:

```bash
# Create a new sized node group, then drain old one
eksctl create nodegroup -f new-size.yaml
kubectl drain m5-xlarge-node-xyz --ignore-daemonsets --delete-emptydir-data
eksctl delete nodegroup --cluster my-cluster --name old-size
```

---

## 🛠️ Step 5: Continuous Right-Sizing (Kubecost / OpenCost)

Install **Kubecost** or **OpenCost** for actionable cost data per workload:

```bash
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost \
  --set kubecostToken=""
```

Key views:
- **Allocations** → cost per namespace/deployment.
- **Efficiency** → % of provisioned resources actually used (target 80-90%).
- **Savings** → idle/right-sizing recommendations.

---

## 🔧 Troubleshooting Right-Sizing

```bash
# Pod pending (insufficient CPU) due to over-packing
kubectl describe pod <pod> | tail -20
# -> check events: FailedScheduling "insufficient cpu"

# Node memory pressure
kubectl describe node <node> | grep -A5 Memory
# -> SystemOOM / kubelet eviction

# Top culprit pods
kubectl top pod -n production --sort-by=memory
```

---

## 📝 Interview Questions

1. **How do you right-size an EKS node pool?**
   - **Answer:** Measure CPU/mem per node & pod, identify over/under-provisioning, right-size pod requests (VPA-assisted), and pick an instance type whose allocatable matches the packed workload with ~20-30% headroom.

2. **Why are pod requests/limits important for cost?**
   - **Answer:** Requests drive scheduler bin-packing and Cluster Autoscaler node count; accurate requests/limits = efficient packing = fewer, cheaper nodes.

3. **Instance type larger but fewer nodes vs smaller but more nodes?**
   - **Answer:** Depends on workload spread, failover blast radius, and bin-packing efficiency. Fewer large nodes = fewer management endpoints but bigger blast radius; smaller nodes scale more granularly.

4. **How do you avoid OOMKills while right-sizing memory?**
   - **Answer:** Set requests near steady-state usage and limits above peak; monitor `kube_pod_container_status_terminated_reason="OOMKilled"`; use VPA recommendations.

5. **What's a healthy node utilization target?**
   - **Answer:** ~70-90% requests-vs-allocatable packing for cost efficiency, with headroom so Cluster Autoscaler/Karpenter can scale without latency.

---

## 📚 Resources for Review

- [Day 41: Prometheus Operator](Day-41_Prometheus_Operator/README.md)
- [Day 54: Cost Optimization Practice](Day-54_Cost_Optimization_Practice/README.md)

### External
- [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [Kubecost](https://www.kubecost.com/)

---

## 🎯 Today's Assignment

1. **Measure** node + pod utilization over a day
2. **List** the top over- and under-provisioned workloads
3. **Add** VPA recommendations for 3 services
4. **Right-size** pod requests/limits for one workload
5. **Change** a node group instance type and validate no impact

---

## 📊 Progress Tracker

```
Week 17-18 Cost Optimization:
Day 51: Spot Instances for EKS ⬜
Day 52: Right-Sizing Nodes ⬜
Day 53: Cluster Autoscaler vs Karpenter ⬜
Day 54: Cost Optimization Practice ⬜
Day 55: Cost Optimization Interview Prep ⬜
```

---

## 🔗 Next Week Preview

**Day 53:** Cluster Autoscaler vs Karpenter

---

**Remember:** Right-sized doesn't mean smallest — it means the least capacity that meets your SLOs with safe headroom! 🚀
