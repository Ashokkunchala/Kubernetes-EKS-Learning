# 📅 Day 51: Spot Instances for EKS

## 🎯 Learning Objectives
By the end of today, you will understand:
- What EC2 Spot Instances are and how pricing works
- Running EKS worker nodes on Spot (managed node groups)
- Spot interruption handling and graceful draining
- Best practices & workload suitability
- Risk management and rebalance recommendations

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Spot Instance** | EC2 capacity offered at a discount that AWS can reclaim when needed. |
| **Interruption notice** | Advance signal, when available, that Spot capacity is about to be reclaimed. |
| **Capacity** | The available CPU, memory, and suitable nodes needed to schedule workloads. |
| **PodDisruptionBudget (PDB)** | A rule limiting certain voluntary disruptions to an application. |
| **Drain** | Evicting Pods from a node so it can be maintained or removed. |
| **On-Demand** | EC2 capacity billed without the Spot interruption model. |

**Analogy:** Spot is a discounted seat that may be recalled; resilient workloads keep spare seats and can move when notified.

**Try / observe:** Review PDBs, replicas, interruption handling, and fallback capacity before moving critical workloads to Spot.

**Troubleshooting clue:** A PDB can block voluntary eviction but does not prevent AWS from reclaiming an instance. Design for interruption.

**Explain it back:** Which workloads suit Spot? Why should critical services have enough replicas and fallback capacity?

---

## 📖 Core Concepts

### Spot Pricing & Interruption

```
┌───────────────────────────────────────────────────────────────┐
│                  SPOT INSTANCES ON EKS                         │
│                                                               │
│   Savings: 60-90% vs On-Demand for interruptible workloads   │
│                                                               │
│   ┌───────────────────────────────────────────────────┐      │
│   │   SPOT CAPACITY POOL (per instance type/zone)     │      │
│   │   price fluctuates with supply/demand             │      │
│   │   AWS reclaims when needed (2-min warning)        │      │
│   └───────────────────────────────────────────────────┘      │
│            │ 2-min notice (Instance Rebalance)                │
│            ▼                                                  │
│   ┌─────────────────────────────────┐                        │
│   │  Graceful handling:             │                        │
│   │  · TerminationHandler DaemonSet  │                        │
│   │  · Node drain (cordon+evict)     │                        │
│   │  · Reschedule pods elsewhere      │                        │
│   └─────────────────────────────────┘                        │
└───────────────────────────────────────────────────────────────┘
```

### Key Facts

- Spot = spare EC2 capacity at low, fluctuating prices.
- **Extra Capacity for EC2:** EKS node groups use this for Spot (preferred).
- **Rebalance recommendation:** 2-minute notice before interruption when capacity/price changes.
- EKS managed node groups handle the **termination → replacement** automatically; your job is to run workloads that tolerate interruption.

---

## 🛠️ Step 1: Create a Spot Managed Node Group

```bash
eksctl create nodegroup \
  --cluster my-cluster \
  --region us-east-1 \
  --name spot-workers \
  --instance-types m5.large,m5a.large,m5d.large,t3.2xlarge \
  --nodes 3 --nodes-min 1 --nodes-max 10 \
  --spot \
  --node-labels="usage=spot" \
  --managed
```

Or via config file:

```yaml
# spot-nodegroup.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: us-east-1
managedNodeGroups:
- name: spot-workers
  instanceTypes: ["m5.large", "m5a.large", "m5d.large", "t3.2xlarge"]
  spot: true
  minSize: 1
  maxSize: 10
  desiredCapacity: 3
  labels:
    usage: spot
  tags:
    k8s.io/cluster-autoscaler/enabled: "true"
```

```bash
eksctl create nodegroup -f spot-nodegroup.yaml
```

> Using **multiple instance types** across **multiple AZs** reduces interruption risk and improves price stability.

---

## 🛠️ Step 2: Isolate Spot vs On-Demand Workloads

Use taints + tolerations + node selector to keep stateful/critical apps OFF spot:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: stateless-worker
spec:
  nodeSelector:
    usage: spot
  tolerations:
  - key: "spot"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: worker
    image: busybox
```

```bash
# Taint the spot node group so only tolerating pods schedule there
eksctl create nodegroup ... --taint spot=true:NoSchedule
# (or in the yaml above)
```

**Rule of thumb:**
- ✅ **Spot-friendly:** stateless web APIs, batch jobs, CI agents, autoscaled workers, canaries.
- ❌ **Avoid Spot:** stateful DBs (unless replicated), single-replica critical services, low-tolerance-of-interruption workloads.

---

## 🛠️ Step 3: Graceful Termination (Node Termination Handler)

**AWS Node Termination Handler** drains nodes before interruption:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm upgrade --install aws-node-termination-handler \
  eks/aws-node-termination-handler \
  --namespace kube-system \
  --set nodeSelector."kubernetes\\.io/os"=linux \
  --set nodeSelector."kubernetes\\.io/arch"=amd64 \
  --set enableSpotInterruptionDraining=true \
  --set enableRebalanceMonitoring=true \
  --set enableScheduledEventDraining=true
```

What it does on interruption notice:
1. **Cordons** the node (no new pods).
2. **Drains** terminating pods gracefully (respects PodDisruptionBudget).
3. Node is removed & replaced by the managed node group.

---

## 🛠️ Step 4: Use PodDisruptionBudget (PDB)

Ensure availability during interruption:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
```

Also set proper `terminationGracePeriodSeconds` in your pods so drains finish.

---

## 🛠️ Step 5: Verify & Monitor

```bash
# Confirm nodes are spot
kubectl get nodes -l usage=spot
aws ec2 describe-instances --filters "Name=instance-lifecycle,Values=spot" \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,SpotInstanceRequestId]' \
  --output table

# Check the termination handler pod
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-node-termination-handler
```

Alert on interruption events:

```bash
aws cloudwatch put-metric-alarm --alarm-name spot-interruptions \
  --metric-name SpotInterruptionCount \
  --namespace "AWS/EC2" --period 300 --evaluation-periods 1 \
  --statistic Sum --threshold 1 --comparison-operator GreaterThanThreshold
```

---

## 📝 Interview Questions

1. **What are Spot Instances and why use them?**
   - **Answer:** Unused EC2 capacity offered at 60-90% discount; ideal for fault-tolerant, stateless, and autoscaled workloads to cut EKS node costs.

2. **How do you handle Spot interruption on EKS?**
   - **Answer:** Use the AWS Node Termination Handler to cordon+drain on the 2-min notice, rely on managed node group to replace nodes, and design workloads with PDBs and multiple replicas.

3. **Which workloads should NOT run on Spot?**
   - **Answer:** Single-replica critical services, stateful workloads without replicas, and services with very low tolerance for interruption. Keep them on On-Demand or dedicated capacity.

4. **How do you keep spot costs stable?**
   - **Answer:** Use many instance types across AZs, avoid single large pools, enable capacity-optimized allocation, and mix Spot + On-Demand node groups.

5. **Spot vs On-Demand vs Savings Plans for EKS?**
   - **Answer:** Spot = lowest price, interruptible; On-Demand = predictable, no commitment; Savings Plans = discount on predictable baseline. Architecture: On-Demand/Savings baseline + Spot for elastic bursts.

---

## 📚 Resources for Review

- [Day 54: Cost Optimization Practice](Day-54_Cost_Optimization_Practice/README.md)
- [Day 53: Cluster Autoscaler vs Karpenter](Day-53_Cluster_Autoscaler_Karpenter/README.md)

### External
- [AWS Spot Instances](https://aws.amazon.com/ec2/spot/)
- [AWS Node Termination Handler](https://github.com/aws/aws-node-termination-handler)

---

## 🎯 Today's Assignment

1. **Create** a Spot managed node group (multi-type, multi-AZ)
2. **Add** the termination handler with interruption draining
3. **Taint** the spot group and add tolerations to suitable workloads
4. **Add** a PDB to a critical deployment
5. **Simulate** an interruption and verify graceful drain

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

**Day 52:** Right-Sizing Nodes

---

**Remember:** Spot is a 60-90% discount — but only if you design for interruption! 🚀
