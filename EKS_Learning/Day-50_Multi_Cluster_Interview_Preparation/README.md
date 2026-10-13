# 📅 Day 50: Multi-Cluster Interview Preparation

## 🎯 Learning Objectives
By the end of today, you will:
- Explain multi-cluster architecture confidently
- Answer scenario & system-design questions
- Contrast federation, mesh, GitOps, and managed options
- Be ready for SRE/Platform-Engineer-style interviews

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Active-active** | Multiple sites/clusters serve traffic at the same time. |
| **Active-passive** | One site serves normally while another is prepared to take over. |
| **Blast radius** | How much of the system a fault can affect. |
| **Failover** | Switching to a healthy alternate system. |
| **Consistency** | How closely replicated data or state agrees across systems. |
| **Federation** | Coordinating selected resources/policies across independent clusters through tooling. |

**Analogy:** Active-active is two open service counters; active-passive is one open counter with a ready backup. Both need clear data ownership.

**Try / observe:** Compare designs by availability, consistency, complexity, cost, recovery time, and failure isolation—not just cluster count.

**Troubleshooting clue:** A second cluster does not guarantee availability if both depend on the same database, DNS, identity provider, or network.

**Explain it back:** What shared dependencies can defeat a multi-cluster availability design?

---

## 📖 Core Concepts - The Decision Framework

```
┌───────────────────────────────────────────────────────────────┐
│         WHY MULTI-CLUSTER?  →  THEN CHOOSE                     │
│                                                               │
│  HA / DR (region failure)      → 2 regions, failover           │
│  Data locality (latency)       → per-region clusters           │
│  Scale / noisy-neighbor        → split fleets                  │
│  Compliance / tenant isolation → dedicated clusters            │
│  On-prem + cloud               → EKS-A + Managed EKS           │
│                                                               │
│  THEN: CONNECT + MOVE WORKLOADS                               │
│   · Networking: peering/TGW/mesh                              │
│   · Deploy: ArgoCD/ApplicationSet (GitOps)                    │
│   · Discover: mesh/CoreDNS/external DNS                       │
│   · Observe: central AMP/Grafana                              │
└───────────────────────────────────────────────────────────────┘
```

### Control Plane Options Compared

| Option | Control plane | Best for |
|--------|---------------|----------|
| Multiple managed EKS | AWS-managed each | Most production cases |
| EKS Anywhere on-prem | You-managed | Residency/latency |
| Federation (KubeFed) | Host pushes | (Legacy) uniform fleet |
| Single ArgoCD + many clusters | GitOps | Deploy orchestration |
| Service mesh | Mesh control plane | L7 traffic across clusters |

---

## 🗣️ Interview Questions by Level

### Beginner
1. **Why would you have more than one EKS cluster?**
   - **Answer:** HA/DR across regions, data locality (lower latency), scaling limits, and isolation for tenants or compliance.

2. **What's the difference between single-region and multi-region clusters?**
   - **Answer:** Single-region = one control plane in one AZ region; multi-region = multiple, independent control planes that must be coordinated (DNS/health-based failover, data replication).

3. **What is a kubeconfig?**
   - **Answer:** A file holding cluster endpoints and credentials; you use `--context` to select among clusters in one kubeconfig.

### Intermediate
4. **How do you secure cluster-to-cluster comms?**
   - **Answer:** Private connectivity (VPC peering/Transit Gateway/PrivateLink), mTLS via a mesh, network policies, and least-privilege IAM scoped per cluster.

5. **How do you do service discovery across clusters?**
   - **Answer:** A service mesh's global namespace (Istio), external-dns + CoreDNS federation, or a central API gateway routing to regional services.

6. **How do you keep manifests consistent across clusters?**
   - **Answer:** GitOps with ArgoCD ApplicationSet; Kustomize/Helm overlays for per-region differences; central Git as single source of truth.

7. **What is failover and how does it work?**
   - **Answer:** Directing traffic away from a failed region/cluster. Health checks (Global Accelerator/Route53) detect failure and route to healthy endpoints; data failover needs replicated storage.

### Senior / Platform
8. **Design a multi-region active-active EKS architecture.**
   - **Answer:** Two regions with independent managed EKS clusters; Global Accelerator/Route53 latency+health routing; global DB (Aurora Global / DynamoDB Global Tables); GitOps (ArgoCD) deploying identical apps; central AMP/Grafana; each region can fail over independently.

9. **How do you manage config drift across many clusters?**
   - **Answer:** GitOps selfHeal reconciles drift; central admission policies (Kyverno/OPA) enforce standards; and automated compliance checks (kube-bench, policy reports).

10. **How do you handle image/version rollout across clusters progressively?**
    - **Answer:** Canary to one region/cluster first, monitor SLOs, then roll to the rest; ArgoCD sync waves / ApplicationSet promotions by environment.

---

## 🧠 Scenario Questions

### Scenario: "A new region joins the fleet. Walk me through onboarding."
**Answer:**
1. Provision the EKS cluster (or EKS-A) in the new region.
2. Connect networking (Transit Gateway/peer), ensure non-overlapping CIDRs.
3. Register the cluster in ArgoCD (`argocd cluster add`).
4. Create the ApplicationSet entry / overlay for the region.
5. Sync and validate (rollout, health, DNS, traffic).
6. Add the region to global ingress health-check group and monitoring.

### Scenario: "Traffic goes to the wrong (or dead) region."
**Answer:** Check health checks; verify regional ALB target-health; confirm Global Accelerator/Route53 routing; look at ISP/DNS TTL; ensure data replicas are current in the target region.

### Scenario: "A microservice only exists in one cluster but must be in all."
**Answer:** Standardize per-cluster app sets; move the service into the shared GitOps repo, add its manifests to base/ and an overlay per region; sync all clusters.

---

## 🛠️ Whiteboard Practice Prompts

1. Draw the arrows for **request flow in a 2-region active-active** setup (edge → regional LB → service → data).
2. Explain **where each part fails** and what happens.
3. Draw **GitOps propagation** from a single repo to N clusters.
4. Show the **data replication** strategy (global DB vs async) and its consistency trade-offs.

---

## 📚 Resources for Review

- Day 46-49 (EKS Anywhere, Networking, Federation, Practice)
- [Day 35: ArgoCD Advanced](Day-35_ArgoCD_Advanced_Best_Practices/README.md)

---

## 🎯 Today's Assignment

1. **Answer** all level-based questions aloud
2. **Draw** the 2-region architecture from memory
3. **Explain** your GitOps multi-cluster flow in 2 minutes
4. **Run** a mock system-design interview
5. **Write** a short "onboarding a new region" runbook

---

## 📊 Progress Tracker

```
Week 15-16 MULTI-CLUSTER COMPLETE:
Day 46: EKS Anywhere ⬜
Day 47: Multi-Cluster Networking ⬜
Day 48: Cluster Federation ⬜
Day 49: Multi-Cluster Practice ⬜
Day 50: Multi-Cluster Interview Prep ⬜
```

---

## 🔗 Next Week Preview

**Week 17-18:** Cost Optimization (Spot, Right-sizing, Karpenter)

---

**Remember:** Multi-cluster interviews reward decision-making, not just commands. Justify your choices! 🚀
