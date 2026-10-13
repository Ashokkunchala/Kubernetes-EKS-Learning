# 📅 Day 54: Cost Optimization Practice - FinanceOps Lab

## 🎯 Learning Objectives
By the end of today, you will:
- Implement a cost-optimized EKS reference architecture
- Combine spot, right-sizing, Karpenter, and FinOps tooling
- Build cost dashboards & budgets
- Set saving plans and governance guardrails
- Quantify savings and monitor cost per workload

---
## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Unit cost** | Cost per useful unit, such as per request, transaction, or customer. |
| **Utilization** | How much of provisioned capacity is actually used. |
| **Right-sizing** | Matching resource requests and infrastructure capacity to observed workload demand. |
| **Spot capacity** | Discounted EC2 capacity that can be reclaimed by AWS. |
| **Savings Plans** | AWS pricing commitments that can reduce eligible compute costs in exchange for a usage commitment. |
| **Cost allocation tag** | A tag used to group/report AWS costs by team, service, or environment. |

**Analogy:** Optimisation is not simply buying cheaper supplies; it is measuring cost per useful result while maintaining reliability.

**Try / observe:** Compare cost with utilization and service-level objectives before changing capacity. Tag resources consistently and verify the cost reports are attributing spend correctly.

**Troubleshooting clue:** A lower monthly bill can hide worse latency or reliability. Compare cost, performance, and availability together.

**Explain it back:** Why is cost per request more useful than total cost alone for a growing service?

---


## 📖 The Full Toolkit

```
┌───────────────────────────────────────────────────────────────┐
│                  END-TO-END COST OPTIMIZATION                  │
│                                                               │
│   ┌─────────────┐ ┌──────────────┐ ┌────────────────┐         │
│   │ Spot/OnDemand│ │ Right-sizing │ │ Karpenter/CA  │        │
│   │ mixed pools │ │ (VPA/limits) │ │ (dynamic)     │        │
│   └──────┬──────┘ └──────┬───────┘ └───────┬────────┘        │
│          └───────┬───────┴────────┬────────┘                 │
│                  ▼                ▼                          │
│        ┌──────────────────────────────────────┐              │
│        │        COST OBSERVABILITY             │              │
│        │  Kubecost / OpenCost / Cost Explorer  │              │
│        └──────────────────┬───────────────────┘              │
│                           ▼                                  │
│        ┌──────────────────────────────────────┐              │
│        │  GOVERNANCE + BUDGETS                │              │
│        │  Budgets & alerts · Tags · Savings   │              │
│        │  Plans · Node cleanup · PDB policy   │              │
│        └──────────────────────────────────────┘              │
└───────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Step 1: Baseline the Odds (Measure first)

```bash
# 1. Understand current spend
aws ce get-cost-and-usage \
  --time-period Start=2026-08-01,End=2026-09-01 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE

# 2. Find and label resources
# Add cost-allocation tags to all clusters/nodes/storage
aws eks tag-resource --resource-arn <cluster-arn> --tags team=platform,env=prod

# 3. Enable a FinOps dashboard
helm install kubecost kubecost/cost-analyzer --namespace kubecost
```

**Golden rule:** You can't optimize what you can't measure. Tag everything with team/env.

---

## 🛠️ Step 2: Adopt the Optimal Architecture

```yaml
# Karpenter NodePool: mixed spot+on-demand across families, consolidated
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: cost-optimized
spec:
  template:
    spec:
      nodeClassRef: { name: default }
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]
      - key: karpenter.k8s.aws/instance-category
        operator: In
        values: ["c", "m", "r"]
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 168h
  limits:
    cpu: "64"
```

```bash
# Keep a small on-demand baseline for critical single-replica workloads
# (via a separate NodePool with capacity-type=on-demand + taint)
```

---

## 🛠️ Step 3: Set Resource Buffers Correctly

Use VPA recommendations, then harden requests/limits:

```bash
# Install VPA
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm install vpa fairwinds-stable/vpa --namespace kube-system
```

```yaml
# VPA in recommendation-only mode, review, then apply
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata: { name: api, namespace: production }
spec:
  targetRef: { apiVersion: apps/v1, kind: Deployment, name: api }
  updatePolicy: { updateMode: "Initial" }
```

---

## 🛠️ Step 4: Buy Savings Plans on the Baseline

```bash
# Estimate baseline spend
aws ce get-savings-plans-purchase-recommendation \
  --lookback-period-in-days 30 --term-in-years 1 \
  --payment-option PartialUpfront --savings-plans-type COMPUTE_SP

# Note: cover predictable baseline with Savings Plans; scale elastic on Spot.
```

**Strategy:**
- On-Demand/Plans → steady baseline workloads.
- Spot → elastic swings (batch, CI, autoscaled web).
- Karpenter picks cheapest type within requirements.

---

## 🛠️ Step 5: Budgets, Alerts & Guardrails

```bash
# Create a budget with alerting
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{"BudgetName":"eks-monthly","BudgetLimit":{"Amount":"5000","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST","CostFilters":{"TagKeyValue":["user:team$platform"]}}' \
  --notifications-with-subscribers '[{"Notification":{"ComparisonOperator":"GREATER_THAN","Threshold":80,"NotificationType":"ACTUAL"},"Subscribers":[{"SubscriptionType":"EMAIL","Address":"ops@example.com"}]}]'
```

**Guardrails:**
- **Node cleanup:** Karpenter `expireAfter` removes stale nodes.
- **Namespace quotas:** enforce Limits/ResourceQuota per team.
- **Idle detection:** alert on pods>0 but no traffic (idle apps waste nodes).
- **Spot-only for non-prod:** label non-prod namespaces to spot pools.

---

## 📊 Kubecost Views to Review Weekly

- **Allocations:** cost by namespace/deployment; spot the expensive apps.
- **Efficiency:** target ~80-90% of provisioned resources used.
- **Savings:** idle resources, right-sizing opportunities, spot upgrade.

---

## 🔧 Troubleshooting Cost Issues

```bash
# Unexpected node count
kubectl get nodes
kubectl top nodes   # idle nodes that should be consolidated?

# Pending pods holding nodes open
kubectl get pods -A | grep -c Pending

# Disk/RDS outside cluster inflating bill
# (check non-node resources: EBS snapshots, NAT GW, Fargate)
aws ec2 describe-volumes --query 'Volumes[?State==`available`].[VolumeId,Size]'
```

---

## 📝 Interview Questions

1. **Walk me through your cost-optimization strategy for EKS.**
   - **Answer:** Tag everything → measure (Kubecost/Cost Explorer) → right-size requests (VPA) → use spot + on-demand mixed via Karpenter → buy Savings Plans on the baseline → enforce budgets, quotas, idle detection, and node expiry (FinOps governance).

2. **How do you allocate costs to teams?**
   - **Answer:** Cost-allocation tags (team/env) at node/pod/manifest level; Kubecost maps node costs to namespaces/deployments; budgets alert per team.

3. **How do you balance cost vs resilience?**
   - **Answer:** Keep critical single-replica services on on-demand/plans; put fault-tolerant, autoscaled workloads on spot; use PDBs and multi-AZ so interruption doesn't drop SLOs.

4. **What are the top 3 cost leaks on EKS?**
   - **Answer:** Idle/underutilized nodes (no Karpenter consolidation), oversized pods/over-provisioned requests, and untracked storage/other services (EBS, NAT GW, snapshots, Fargate).

5. **How do you keep node costs from creeping up after a spike?**
   - **Answer:** Karpenter `expireAfter` + consolidation, autoscaling (HPA/KEDA), idle/spot alerts, and weekly Kubecost efficiency review.

---

## 📚 Resources for Review

- Day 51-53 (Spot, Right-sizing, Karpenter/CA)
- [Day 54: Cost Optimization Practice](Day-54_Cost_Optimization_Practice/README.md)

---

## 🎯 Today's Assignment

1. **Tag** all resources (team/env) and enable Kubecost
2. **Implement** the spot+on-demand Karpenter NodePool
3. **Buy/review** Savings Plans recommendation
4. **Set** a monthly budget with 80% alert
5. **Produce** a weekly efficiency report & action list

---

## 📊 Progress Tracker

```
Week 17-18 Cost Optimization:
Day 51: Spot Instances for EKS ⬜
Day 52: Right-Sizing Nodes ⬜
Day 53: Cluster Autoscaler vs Karpenter ⬜
Day 54: Cost Optimization Practice ✅ (hands-on)
Day 55: Cost Optimization Interview Prep ⬜
```

---

## 🔗 Next Week Preview

**Day 55:** Cost Optimization Interview Preparation

---

**Remember:** Cost optimization is an ongoing FinOps loop — measure, act, verify, repeat. Not a one-time project! 🚀
