# 📅 Day 60: Final Review & Certification Preparation

## 🎯 Learning Objectives

By the end of today, you will be able to:
- Recall every major domain from the 60-day journey at interview depth
- Map your skills to the CKA / CKAD / CKS / AWS certification paths
- Answer cross-domain interview questions that combine multiple topics
- Build the final capstone project from scratch (the "graduation exam")
- Create your personal post-course roadmap and portfolio plan

> **Congratulations** — you've reached the finish line. Today is not the end;
> it's the day you prove (to yourself and to employers) that you can design,
> operate, and troubleshoot EKS in production.

---
## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Desired state** | The configuration you want the system to maintain. |
| **Least privilege** | Giving each identity only the permissions needed for its task. |
| **Observability** | Understanding system behavior from metrics, logs, traces, and other evidence. |
| **RTO / RPO** | RTO is the target time to recover; RPO is the acceptable amount of data loss measured in time. |
| **Capstone** | A larger end-to-end project that combines skills from multiple lessons. |
| **Operational readiness** | Evidence that a system can be deployed, monitored, secured, troubleshot, and recovered by its operators. |

**Analogy:** Certification checks whether you know the tools; a production capstone checks whether you can choose the right tool, prove it works, and recover when it fails.

**Try / observe:** For every major domain, explain the concept without notes, run a relevant command, identify one likely failure, and demonstrate how you would verify recovery.

**Troubleshooting clue:** A green deployment status is not the same as a healthy service. Check user-facing behavior, dependencies, metrics/logs, security boundaries, and rollback/recovery readiness.

**Explain it back:** What evidence would convince you that an EKS platform is production-ready rather than merely deployed?

---


## 🗺️ The 60-Day Journey at a Glance

```
Week 1-2   (D 1-7)  Kubernetes core: pods, deployments, services, storage
                    namespaces, configmaps, secrets, networking deep dive
Week 3-4   (D 8-15) EKS foundation: cluster setup, VPC, IAM/IRSA, EBS,
                    ALB ingress, monitoring, security best practices
Week 5-6   (D 16-20) Advanced networking: VPC CNI, Calico, Istio, practice
Week 7-8   (D 21-25) Advanced storage: EFS, FSx, S3 CSI, snapshots, practice
Week 9-10  (D 26-30) IAM & security: OIDC, IRSA multi, pod-level IAM,
                    Secrets Manager, practice
Week 11-12 (D 31-35) GitOps: ArgoCD intro, install, app patterns, sync
                    & rollback, advanced best practices
Week 13-14 (D 36-40) CI/CD: CodePipeline, GitHub Actions, Jenkins on EKS,
                    pipeline practice, interview prep
Week 15-16 (D 41-45) Observability: Prometheus operator, Grafana, X-Ray,
                    custom metrics, monitoring practice
Week 17-18 (D 46-50) Multi-cluster: EKS Anywhere, networking, federation,
                    practice, interview prep
Week 19-20 (D 51-55) Cost: spot instances, right-sizing, CA vs Karpenter,
                    practice, interview prep
Week 21-22 (D 56-60) Disaster recovery: backup, cross-region replication,
                    failover, DR practice, FINAL REVIEW ✅
```

---

## 📚 Final Knowledge Blueprint (By Certification Domain)

### Domain 1: Kubernetes Architecture & Workloads
- [ ] Control plane vs data plane; component responsibilities
- [ ] Pod lifecycle, init containers, sidecars, resource QoS classes
- [ ] Deployments → ReplicaSets → Pods; rolling updates, `maxSurge`/`maxUnavailable`
- [ ] StatefulSets, DaemonSets, Jobs, CronJobs
- [ ] Liveness / readiness / startup probes and when to use each

### Domain 2: Services, Networking & Ingress
- [ ] ClusterIP, NodePort, LoadBalancer, ExternalName, headless services
- [ ] kube-proxy (iptables vs IPVS), CoreDNS, service discovery
- [ ] Ingress + ingressClassName (ALB controller in EKS)
- [ ] Network Policy: default deny, allow rules (Calico / VPC CNI)
- [ ] VPC CNI: IPAM, warm pools, prefix delegation, custom networking, SG for pods

### Domain 3: Storage
- [ ] PV / PVC / StorageClass; `volumeBindingMode: WaitForFirstConsumer`
- [ ] EBS (gp3/io2), EFS, FSx for Lustre, S3 CSI driver — when to use each
- [ ] Volume snapshots, restore, CSI driver architecture
- [ ] Stateful data patterns: StatefulSet + PVC, operators (e.g., RDS)

### Domain 4: AWS EKS Specifics
- [ ] eksctl cluster lifecycle, managed vs self-managed node groups
- [ ] VPC design: public/private subnets, NAT vs IGW, security groups
- [ ] IAM: cluster role, node role, IRSA + OIDC, aws-auth, RBAC
- [ ] Add-ons: VPC CNI, CoreDNS, kube-proxy, EBS CSI, load-balancer-controller
- [ ] Secrets encryption with KMS (etcd), audit logging, cluster endpoint control
- [ ] Pricing model, control plane $0.10/hr, cost optimization levers

### Domain 5: GitOps & CI/CD
- [ ] Git as the source of truth; desired-state reconciliation
- [ ] ArgoCD: projects, apps, sync strategies, PR-based promotion, rollback
- [ ] Pipelines: GitHub Actions / CodePipeline / Jenkins → build → ECR → deploy
- [ ] Image tagging strategies (immutable tags, git SHA), secret injection
- [ ] Progressive delivery: blue-green, canary, Flagger/Kargo

### Domain 6: Observability
- [ ] Prometheus: metrics model, ServiceMonitor CRDs, recording rules
- [ ] Grafana dashboards + alerting; CloudWatch Container Insights
- [ ] AWS X-Ray: tracing, service maps, sampling
- [ ] Logging: Fluent Bit → CloudWatch/OpenSearch; Elastic Stack patterns
- [ ] Custom metrics → HPA (autoscaling/v2)

### Domain 7: Multi-Cluster & Federation
- [ ] EKS Anywhere / EKS Distro for on-prem; AWS outposts consideration
- [ ] Multi-cluster networking (service mesh multi-cluster, inter-cluster DNS)
- [ ] Federation: when to federate vs "paved roads" with GitOps per cluster
- [ ] Centralized observability across fleets; cluster fleet lifecycle

### Domain 8: Cost Optimization
- [ ] Spot instances + interruption handling; node group mixing
- [ ] Right-sizing with VPA recommendations; namespace resource quotas
- [ ] Cluster Autoscaler vs Karpenter (provisioning models)
- [ ] Kubecost/OpenCost, tags + Cost Explorer, Savings Plans vs On-Demand

### Domain 9: Disaster Recovery
- [ ] RPO/RTO definitions; backup (Velero) vs replication vs failover
- [ ] Cross-region: ECR registry replication, RDS read replica, S3 CRR
- [ ] Failover & failback runbooks; Route53/GA health-check cutover
- [ ] Game Days, sandbox restore tests, DR scorecards

### Domain 10: Security Hardening
- [ ] Defense in depth: cluster → node → pod → container → network → data
- [ ] Pod Security Standards (baseline/restricted); securityContext
- [ ] Minimal images (distroless), ECR scanning (Trivy), Cosign signing
- [ ] IRSA vs static creds; Secrets Manager / external secrets operators
- [ ] Least-privilege IAM + RBAC; audit logging; kube-bench / Checkov

---

## 📋 Certification Paths & Recommended Order

| Certification | Focus | Cost* | Prep Time | Best Next Step Today |
|---------------|-------|-------|-----------|----------------------|
| **CKAD** | App-focused Kubernetes (pods, deployments, services, storage) | ~$395 | 3-5 wks | Take 2 mock exams; you're 80% ready after D1-15 + storage D21-25 |
| **CKA** | Cluster admin: install, upgrade, troubleshooting | ~$395 | 4-6 wks | Complement with cluster-admin practice (kubeadm, node repair) |
| **CKS** | Kubernetes security (requires CKA) | ~$395 | 4-6 wks | After CKA; covers PSS, policies, runtime security, images |
| **AWS Certified Developer - Associate** | Build/deploy on AWS (incl. some EKS) | ~$150 | 2-3 wks | Broad AWS familiarity |
| **AWS Certified DevOps Engineer - Professional** | CI/CD + ops at scale (EKS heavy) | ~$300 | 4-6 wks | **Best EKS match** — covers pipelines, IaC, monitoring, DR |
| **AWS Certified Solutions Architect - Associate/Pro** | Architecture across AWS | ~$150/$300 | 4-8 wks | Your VPC/IAM/networking foundation transfers directly |

\* *Prices are indicative and change; always confirm on official exam pages.*

**Practical path for an EKS-focused career:**
```
CKAD (you are nearly ready) → CKA (deepen ops) → AWS DevOps Engineer - Pro
                                          └──> optionally CKS later
```

> **Exam-day reality check:** Kubernetes certs are *performance-based* — you
> solve cluster problems in a live terminal, from memory. Muscle memory for
> `kubectl`/`eksctl`/`vim` beats flashcards. After 60 days of hands-on work
> you already have the repetition; mock exams just add pacing.

---

## ❓ Master Interview Questions (Cross-Domain)

**Q1: You own a stateless app on EKS. How do you take it from "works" to "production-grade"?**
> **A:** Foundation (probes, resources, PDBs, HPA), platform (GitOps +
> rollout strategies, private images in ECR with scanning), access
> (IRSA, RBAC, network policies, PSS), capacity (Karpenter/CA + spot for
> the right tiers), observability (Prometheus/Grafana + CloudWatch +
> X-Ray + Fluent Bit logs), and resilience (multi-AZ, EBS for the DB via
> StatefulSet, Velero backups + Game Days). Every one of those maps to a
> week in this plan — I've done all of them hands-on.

**Q2: Why IRSA instead of putting AWS keys in a Secret?**
> **A:** Static keys leak (env dumps, image layers, logs) and never rotate.
> IRSA fuses the pod's OIDC-issued identity to a service account annotation
> so the pod assumes a scoped IAM role with short-lived STS credentials.
> No keys in the cluster at all. Combined with least-privilege policies
> and rotation, credential exposure surface shrinks to near zero.

**Q3: Your pods are stuck `Pending`. Walk me through the diagnosis.**
> **A:** `kubectl describe pod` events first: `FailedScheduling` = no nodes
> fit (resources, taints) → check `kubectl top nodes` + capacity; if it's
> IPAM, the subnet is out of IPs → check VPC CNI ENI limits and consider
> prefix delegation; `FailedMount` = PVC bound? StorageClass provisioner
> has credentials? KMS key missing in that region? Never guess — read events,
> then logs of the kubelet/CNI/CSI pods.

**Q4: How would you roll out a canary to 10% of traffic on EKS?**
> **A:** GitOps-ready approaches: (1) Argo Rollouts with an ALB weighted
> target group split — the controller manages traffic % natively with
> analysis steps; (2) Istio VirtualService weight routing for mesh users;
> (3) vanilla — a second Deployment exposed via a canary Service and
> weighted DNS. Automation should verify error rate + latency (Prometheus
> queries) before auto-promoting, and auto-abort on failure.

**Q5: What breaks at 10,000 pods that works at 100 pods?**
> **A:** IP address exhaustion (biggest EKS one — plan subnets + prefix
> delegation); CoreDNS and kube-proxy scale; etcd/API watch pressure;
> crash on scheduler throughput; cost (you need Karpenter/spot discipline).
> Design for density: right-size nodes, service loads, watch ENI counts.

**Q6: Compare Cluster Autoscaler and Karpenter.**
> **A:** CA works with node groups/ASGs — it only changes the desired count
> of existing groups, so you're bounded by group templates. Karpenter
> provisions *nodes* directly from scheduling demand (multiple instance
> types, spot/on-demand mix, consolidation, node expiry) — tighter packing,
> faster, smaller nodes. Karpenter is now the AWS-recommended default; CA
> remains for teams that need ASG-driven patterns.

**Q7: How do you handle secrets safely?**
> **A:** Never in git. In-cluster: Secrets object (base64 ≠ encryption —
> enable KMS encryption at rest), External Secrets Operator / Secrets Store
> CSI pulling from AWS Secrets Manager for rotation; IRSA for cloud creds;
> sealed secrets for repo-friendly encryption. Least-privilege RBAC on
> who can read them; audit access.

**Q8: What is your multi-environment promotion flow?**
> **A:** Branch/repo per env (dev → staging → prod), ArgoCD App-of-Apps with
> a Kustomize overlay per env, image bumped via `kustomize edit set image`
> on an automated PR, PR-based sync into dev, manual `argocd app sync`
> to prod behind an approvals gate, rollback = revert the PR (GitOps makes
> rollbacks a git revert, not an ops action).

**Q9: A senior engineer asks why you need a service mesh. Do you?**
> **A:** Only for value > cost (extra proxies are not free). Use Istio/linkerd
> when you need mTLS everywhere, fine-grained traffic split / fault
> injection, and unified telemetry across many services. For a handful of
> services, Envoy per service via NLB/ALB + a service mesh lite (e.g.,
> mTLS via SPIRE, or no mesh) is often the honest answer. I'd start without
> one and adopt for a business-specific need.

**Q10: Walk me through an EKS failure you'd design for and how you'd recover.**
> **A:** Pick one (e.g., full primary region loss): RDS read replica in DR
> region + ECR registry replication + Velero to cross-region bucket + app
> deployed via ArgoCD to a standby cluster + Route53 health-check failover
> runbook. RTO/RPO measured in a Game Day (Day 59), not assumed. Then the
> answer is a runbook, not a prayer.

---

## 🏗️ Final Capstone: The "Graduation Exam"

Build this **from scratch** in less than a weekend, using only your notes.
Do *not* copy-paste whole answers; type every manifest.

**The Production E-Commerce Stack**
```
Requirements (each maps to a stretch of the plan):
 1. eksctl cluster: private subnets, 2 node groups
    (on-demand critical + spot for workers)      → D9-10, D51
 2. IRSA for S3 + Secrets Manager + ECR pulls     → D26-30
 3. EBS CSI with encryption for the DB,
    EFS for shared uploads                        → D12, D21
 4. ALB ingress with HTTPS + ACM cert,
    path-based routing /api → backend             → D13
 5. Network policies default-deny + allow app     → D17
 6. Pod Security Standards restricted for new ns   → D15
 7. Prometheus + Grafana dashboards + alerting    → D41-42
 8. CloudWatch Container Insights + X-Ray trace   → D14, D43
 9. ArgoCD GitOps for app + infra repo            → D31-35
10. CI/CD: PR → tests → ECR push → auto-sync      → D36-40
11. Velero schedule + restore drill (sandbox)     → D56, D59
12. Cost report: node spend vs workload           → D52-54
```

**Assessment rubric:** 10/12 working = you pass. All 12 = you're interview-ready.
Track it here:

```
[ ] 1  [ ] 2  [ ] 3  [ ] 4  [ ] 5  [ ] 6
[ ] 7  [ ] 8  [ ] 9  [ ] 10 [ ] 11 [ ] 12
```

---

## 🚀 What's Next (Beyond Day 60)

1. **Certify:** pick CKAD or AWS DevOps Engineer-Pro, book the exam date
2. **Build:** port your capstone to a real GitHub project (even fake data)
3. **Contribute:** fix docs in kubectl/eksctl/ArgoCD; or open-source a small operator
4. **Blog / teach:** one "how I'd X on EKS" post per week (interviewers read this)
5. **Stay current:** `eksctl`, EKS release notes, aws-eks best practices docs,
   and the CNCF landscape — 1 slack/reddit read per week is enough
6. **Update LinkedIn/resume** with concrete stacks: `EKS, ArgoCD, GitHub Actions,
   Prometheus/Grafana, Terraform, Karpenter, Velero, Istio`

---

## 📊 Progress Tracker — The Complete 60 Days

```
Week 1-2  Foundation  (D1-7):  ✅ ✅ ✅ ✅ ✅ ✅ ✅
Week 3-4  EKS Basics  (D8-15): ✅ ✅ ✅ ✅ ✅ ✅ ✅ ✅
Week 5-6  Networking  (D16-20): ✅ ✅ ✅ ✅ ✅
Week 7-8  Storage     (D21-25): ✅ ✅ ✅ ✅ ✅
Week 9-10 IAM/Security(D26-30): ✅ ✅ ✅ ✅ ✅
Week 11-12 GitOps     (D31-35): ✅ ✅ ✅ ✅ ✅
Week 13-14 CI/CD      (D36-40): ✅ ✅ ✅ ✅ ✅
Week 15-16 Observability(D41-45): ✅ ✅ ✅ ✅ ✅
Week 17-18 Multi-cluster(D46-50): ✅ ✅ ✅ ✅ ✅
Week 19-20 Cost       (D51-55): ✅ ✅ ✅ ✅ ✅
Week 21-22 DR & Final (D56-60): ✅ ✅ ✅ ✅ ✅

TOTAL: 60 / 60 DAYS COMPLETED ✅
```

---

## 📚 Resources & Next Stop

- [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)
- [Certified Kubernetes Application Developer (CKAD) Curriculum](https://github.com/cncf/curriculum)
- [CKA Exam](https://www.cncf.io/certification/cka/)
- [AWS DevOps Engineer - Professional](https://aws.amazon.com/certification/certified-devops-engineer-professional/)
- [Velero Docs](https://velero.io/docs/)
- [ArgoCD Docs](https://argo-cd.readthedocs.io/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

## 🎯 Today's Assignment

1. **Score** yourself on the Final Knowledge Blueprint (⭐ / strong, 🔶 / okay, ⚠ / review)
2. **Book** (or at least schedule) your chosen certification exam
3. **Build** the capstone; mark all 12 boxes honestly
4. **Answer** all 10 master questions aloud without notes
5. **Write** a 5-line summary of each of the 20 weeks for your portfolio

---

## 🔗 It's a Launchpad, Not a Deadline

**Day 61 onwards:**
- Pick a **production problem**, not a chapter, to go deeper on
- Shadow/observe on real incidents where possible
- Teach your first topic (that's how you'll know you truly own it)

---

**Remember:** Sixty days ago this was a folder with a plan. Today you hold a
portfolio of skills that deploys, scales, secures, and recovers workloads on
EKS. Go build something that survives contact with production. 🚀🏁

*Day 60 of 60 — EKS Learning Journey, Completed*
*Last updated: September 2026*