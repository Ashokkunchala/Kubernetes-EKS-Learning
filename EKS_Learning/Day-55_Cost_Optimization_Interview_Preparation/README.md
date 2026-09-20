# 📅 Day 55: Cost Optimization Interview Preparation

## 🎯 Learning Objectives
By the end of today, you will:
- Answer cost-optimization questions with concrete numbers
- Explain FinOps strategy and decision trade-offs
- Justify instance/spot/right-sizing choices
- Handle scenario questions about runaway costs
- Link cost optimization to architecture decisions

---

## 📖 Core Concepts - The FinOps Framework

```
┌───────────────────────────────────────────────────────────────┐
│                     FINOPS LIFECYCLE                          │
│                                                               │
│   1. INFORM (measure)     2. OPTIMIZE (act)                   │
│   · tagging/cost alloc    · right-size requests               │
│   · cost visibility       · spot + on-demand mix              │
│   · budget/alert          · Karpenter consolidation           │
│   · benchmark             · Savings Plans baseline            │
│                                                               │
│   3. OPERATE (govern)                                         │
│   · quotas/limits per team                                   │
│   · idle/spot detection                                      │
│   · chargeback/showback                                      │
│   · recurring review cadence                                 │
└───────────────────────────────────────────────────────────────┘
```

### Speak in Numbers

- Spot savings: **60-90%**
- Target efficiency: **~70-90%** of provisioned resources used
- Right-sizing gain: often **15-40%** from accurate requests
- Karpenter consolidation: typically **20-40%** vs static groups
- Savings Plans: **up to 72%** off on-demand on eligible compute

---

## 🗣️ Interview Questions by Level

### Beginner

1. **What is FinOps?**
   - **Answer:** A practice combining finance + operations to get business value from cloud spend — via measurement, optimization, and governance (inform/optimize/operate).

2. **Spot vs On-Demand vs Savings Plans?**
   - **Answer:** Spot = cheapest, interruptible, for flexible work; On-Demand = predictable, no commitment; Savings Plans = discounted on-demand for committed baseline.

3. **What does right-sizing mean on EKS?**
   - **Answer:** Matching requested/allocated resources (CPU/mem per pod and instance type per node) to actual usage so you neither over-pay nor under-provision.

### Intermediate

4. **How would you reduce an EKS bill by 30%?**
   - **Answer:** Measure & tag → find idle/underutilized nodes → enforce accurate requests (VPA) → move fault-tolerant workloads to spot → enable Karpenter consolidation → cover baseline with Savings Plans → clean up idle volumes/snapshots.

5. **How do you attribute cost to a team/namespace?**
   - **Answer:** Cost-allocation tags on nodes/pods + Kubecost/OpenCost to map node cost to namespace/deployment; then budgets & alerts per team.

6. **What's the trade-off of aggressive right-sizing?**
   - **Answer:** Tighter requests lower cost but reduce headroom → more scheduling pressure, risk of throttling/OOM; keep spikes and autoscaling lag in mind.

### Senior / Platform

7. **Design a cost-optimized yet resilient EKS architecture.**
   - **Answer:** Karpenter NodePools mixing spot+on-demand by workload criticality; PDBs + multi-AZ so spot loss is absorbed; VPA-informed requests; Savings Plans on steady baseline; CPU/mem autoscaling; central Kubecost + budgets for governance; idle/expiry policies on nodes and volumes.

8. **How do you handle a cost spike mid-month?**
   - **Answer:** Use the alert to drill into Kubecost/Cost Explorer by service → spot runaway scaling (HPA over-scaling on a buggy metric) → revert, set HPA/KEDA maxes and budgets, and add idle/spot guards.

9. **How do you balance Spot risk for a SLA-critical service?**
   - **Answer:** Keep the SLA-critical component on on-demand/plans (or spot+PDB with multi-replica redundancy), run only redundant/autoscaled slices on spot, and test interruption via fault injection.

10. **When does over-provisioning make sense?**
    - **Answer:** For latency-sensitive services where cold-start or autoscaler lag hurts SLO more than the extra compute cost; sizing must weigh cost vs user experience.

---

## 🧠 Scenario Questions

### Scenario: "Cluster runs 30% utilized nodes but bill keeps rising."
**Answer:** Enable Karpenter consolidation (`WhenUnderutilized`)/CA scale-down; enforce CPU/mem requests; add idle-app detection; set `expireAfter`; review un-optimized workloads; ensure autoscalers (HPA) actually scale down (min replicas).

### Scenario: "A team's namespace cost doubled overnight."
**Answer:** Kubecost allocation → find the workload → check replicas (HPA burst), image size, new resource requests, spot vs on-demand change, or a spill to expensive instance type → tune + set per-namespace budget/ResourceQuota.

---

## 🛠️ Whiteboard Prompts

1. Draw the **FinOps loop** and label each step with an EKS action.
2. Show **cost allocation**: node cost → namespaces via labels/tags.
3. Draw **spot + on-demand + Savings Plans** coverage of demand over time.
4. Explain how **Karpenter consolidation** reduces nodes over a 24h cycle.

---

## 📚 Resources for Review

- Day 51-54 (Spot, Right-sizing, Karpenter/CA, Practice)
- [Day 52: Right-Sizing Nodes](Day-52_Right_Sizing_Nodes/README.md)

---

## 🎯 Today's Assignment

1. **Answer** all level-based questions aloud
2. **Quantify** two cost optimizations (calculate % saved)
3. **Draw** the FinOps loop + cost allocation map
4. **Run** a mock scenario ("bill doubled") interview
5. **Write** a one-page cost optimization runbook

---

## 📊 Progress Tracker

```
Week 17-18 COST OPTIMIZATION COMPLETE:
Day 51: Spot Instances for EKS ⬜
Day 52: Right-Sizing Nodes ⬜
Day 53: Cluster Autoscaler vs Karpenter ⬜
Day 54: Cost Optimization Practice ⬜
Day 55: Cost Optimization Interview Prep ⬜
```

---

## 🔗 Next Week Preview

**Week 19-20:** Disaster Recovery (Backup, Replication, Failover)

---

**Remember:** The best cost answer ties spend to measurable business value — quantify, don't just recite! 🚀
