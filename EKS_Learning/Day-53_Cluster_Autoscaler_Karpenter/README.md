# 📅 Day 53: Cluster Autoscaler vs Karpenter

## 🎯 Learning Objectives
By the end of today, you will understand:
- How Cluster Autoscaler (CA) and Karpenter scale nodes
- Key differences: provisioning, instance selection, and speed
- Setting up both on EKS
- When to choose CA vs Karpenter
- Modern best practices (Karpenter is default for new clusters)

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Pending Pod** | A Pod that has not yet been scheduled/started successfully. |
| **Node provisioning** | Creating or adding worker nodes so workloads have capacity. |
| **Cluster Autoscaler** | A tool that adjusts node-group size based on unschedulable Pods and underused nodes. |
| **Karpenter** | A node-provisioning system that can choose and launch suitable compute capacity from workload requirements. |
| **Consolidation** | Replacing or removing inefficiently used nodes to reduce waste. |
| **Disruption budget** | A policy that limits certain voluntary workload disruptions during maintenance or consolidation. |

**Analogy:** Pod autoscaling adds more workers to a team; node autoscaling adds more desks when the office runs out of space.

**Try / observe:** `kubectl get pods -A --field-selector=status.phase=Pending`, `kubectl describe pod <pod-name>`, and inspect autoscaler/controller logs.

**Troubleshooting clue:** If no node can fit a Pod, check requests, node selectors/affinity, taints/tolerations, available instance types, subnet capacity, and autoscaler permissions.

**Explain it back:** How is adding replicas different from adding nodes? What triggers node provisioning?

---

## 📖 Core Concepts

### The Two Approaches

```
┌───────────────────────────────────────────────────────────────┐
│           NODE AUTOSCALING: TWO APPROACHES                     │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │  CLUSTER AUTOSCALER (CA)                           │     │
│  │  · watches pending pods                             │     │
│  │  · scales EXISTING node groups up/down              │     │
│  │  · groups are pre-defined (fixed instance types)    │     │
│  │  · reactive (seconds-minutes)                       │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │  KARPENTER                                          │     │
│  │  · schedules pods DIRECTLY onto optimal instances   │     │
│  │  · provisions/terminates nodes as needed (jobs)     │     │
│  │  · picks instance types from spec (flexible)        │     │
│  │  · fast (< 1 min) + consolidates to save cost       │     │
│  └─────────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────────┘
```

### Comparison

| Feature | Cluster Autoscaler | Karpenter |
|---------|--------------------|-----------|
| Unit of scaling | Node group | Individual node |
| Instance selection | From node group | From requirements |
| Node types | Pre-set | Any in spec |
| Speed | Slower (group-based) | Faster (direct) |
| Consolidation | Limited | Automatic |
| Spot | Via node group | Mixed/spot from spec |
| Multi-AZ | Per node group | Chosen optimally |
| Complexity | Nodes + ASG | One controller |
| Status | Maintained | AWS-managed, default-recommended |

---

## 🛠️ Option A: Cluster Autoscaler on EKS

```bash
# Associate OIDC (for IRSA)
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve

# Install via IRSA policy + Helm
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=us-east-1 \
  --set rbac.serviceAccount.create=true \
  --set rbac.serviceAccount.name=cluster-autoscaler \
  --set rbac.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=<role-arn>
```

Requirements:
- Node groups must be tagged `k8s.io/cluster-autoscaler/enabled=true`.
- Uses **pending pods** to trigger scaling.

```bash
kubectl -n kube-system logs -f deployment/cluster-autoscaler
# watch for "scale-up: ..." / "removing empty node..."
```

---

## 🛠️ Option B: Karpenter on EKS

### Install

```bash
helm repo add karpenter https://charts.karpenter.sh
helm repo update

helm upgrade --install karpenter karpenter/karpenter \
  --namespace karpenter --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=<karpenter-role> \
  --set settings.clusterName=my-cluster \
  --set settings.interruptionQueue=my-cluster \
  --set controller.resources.requests.cpu=300m
```

### NodePool (NodeClass + NodePool)

```yaml
# ec2nodeclass.yaml - how nodes are created
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiSelectorTerms:
  - alias: al2023@latest
  role: "KarpenterNodeRole-my-cluster"
  subnetSelectorTerms:
  - tags:
      karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
  - tags:
      karpenter.sh/discovery: my-cluster
```

```yaml
# nodepool.yaml - requirements + limits
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64"]
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["on-demand", "spot"]
      - key: karpenter.k8s.aws/instance-category
        operator: In
        values: ["c", "m", "r"]
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
  limits:
    cpu: "100"
```

```bash
kubectl apply -f ec2nodeclass.yaml -f nodepool.yaml
```

Now Karpenter:
- Provisions nodes to fit pending pods (can choose `c7i`, `m7i`, etc. optimally).
- **Consolidates** underutilized nodes to cheaper/fewer instances.
- Handles **interruption** (spot rebalance) automatically.

---

## 🛠️ Test Scaling

```bash
# Deploy a workload that doesn't fit existing nodes
kubectl scale deploy pending-demo --replicas=50
kubectl get nodes -w
kubectl get pods -o wide | grep Pending
```

With Karpenter: watch `kubectl logs -n karpenter deployment/karpenter -f` for "provisioned socket".

---

## 🛠️ Choose: CA vs Karpenter

**Choose Karpenter (recommended for new clusters):**
- Only fully needs pending-pod-driven scaling.
- Want automatic spot + on-demand mixed, consolidation, fewer moving parts.
- Development pace (2026): Karpenter is the AWS default recommendation.

**Choose Cluster Autoscaler when:**
- You already invest in **designed node groups** (e.g., per-tenant/labeled ASGs).
- You rely on node group-level controls and want native ASG lifecycle.
- You run **EKS-A / non-AWS VMs** where Karpenter isn't available.

> Migrating: CA and Karpenter must not both manage the same nodes. Move node groups to Karpenter NodePools, or pick one.

---

## 📝 Interview Questions

1. **What's the difference between Karpenter and Cluster Autoscaler?**
   - **Answer:** CA scales fixed node groups based on pending pods; Karpenter directly schedules pods onto optimally chosen instances, provisions on demand, and consolidates — faster and more flexible.

2. **Why is Karpenter faster?**
   - **Answer:** It bypasses node-group/ASG orchestration and creates instances sized for pending pods immediately, whereas CA must scale a pre-defined group and often over-provisions.

3. **How does Karpenter reduce cost?**
   - **Answer:** It packs pods into the cheapest suitable instance types, mixes spot/on-demand, and continuously consolidates underutilized nodes.

4. **Can you run CA and Karpenter together?**
   - **Answer:** Not on the same nodes — they'd conflict. Use one; migrate workloads from CA-managed groups to Karpenter NodePools.

5. **What are NodePool and EC2NodeClass in Karpenter?**
   - **Answer:** EC2NodeClass defines HOW nodes are created (AMI, subnets, SG, role); NodePool defines scheduling requirements, limits, and disruption/consolidation policy.

---

## 📚 Resources for Review

- [Day 51: Spot Instances](Day-51_Spot_Instances_EKS/README.md)
- [Day 52: Right-Sizing Nodes](Day-52_Right_Sizing_Nodes/README.md)
- [Day 54: Cost Optimization Practice](Day-54_Cost_Optimization_Practice/README.md)

### External
- [Karpenter](https://karpenter.sh/)
- [Cluster Autoscaler on AWS](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler/cloudprovider/aws)

---

## 🎯 Today's Assignment

1. **Install** Cluster Autoscaler on a test cluster and observe group scaling
2. **Install** Karpenter (NodeClass + NodePool)
3. **Trigger** a scale-up and watch both approaches
4. **Compare** time-to-ready and node types chosen
5. **Configure** consolidation and spot mixed in Karpenter

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

**Day 54:** Cost Optimization Practice

---

**Remember:** Karpenter wins for new builds; CA fits existing node-group designs. Pick based on your fleet, not hype! 🚀
