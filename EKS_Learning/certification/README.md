# 🎓 EKS & Kubernetes Certification Prep

Hands-on exam prep that plugs straight into the 60-day learning plan. Everything here is
**simulator-style**: performance tasks (CKA/CKAD) and scenario-based EKS multiple choice
(AWS DevOps / Solutions Architect), plus 60 flashcards for spaced repetition.

> **Exam-day reality (repeat after Day 60):** Kubernetes certs are *performance-based*.
> You solve clusters in a live terminal from memory. Type the answers, don't read them.

---

## 📋 What This Folder Covers

| Certificate | Level | What you practice here |
|-------------|-------|------------------------|
| **CKA** (Certified Kubernetes Administrator) | Performance-based | Day 1-7 + Day 56 etcd/backup, Day 53 autoscaling: taints, drains, RBAC, NetworkPolicy, etcd backup/restore |
| **CKAD** (Certified Kubernetes Application Developer) | Performance-based | Day 1-7 + Day 21-25 storage: deployments, probes, jobs, cronjobs, PVCs, sidecars, ingress, rollouts |
| **AWS Certified DevOps Engineer – Professional** | Multiple choice | EKS-native: CNI, IRSA/Pod Identity, add-ons, Karpenter vs CA, Fargate, upgrades, quotas |
| **AWS Certified Solutions Architect (Associate/Pro)** | Multiple choice | EKS architecture: control plane, VPC design, networking, storage, HA, cost |

### Files

| File | What it is | How to use it |
|------|-----------|---------------|
| `README.md` | This overview + study schedule | Read first, print the schedule |
| `CKA_Practice_Exam_Part1.md` | 20 CKA-style performance tasks | Do every task by hand, timed |
| `CKAD_Practice_Exam_Part1.md` | 20 CKAD-style workload-authoring tasks | Do every task by hand, timed |
| `AWS_EKS_Practice_Exam.md` | 30 EKS multiple-choice questions | Mock-exam mode, 45 min, no notes |
| `FLASHCARDS.md` | 60 Q/A flashcards, 10 topic groups | Daily spaced repetition (Anki-style) |

---

## 🗺️ Study Schedule (2-Week Exam Sprint)

The fastest path depends on which cert you're chasing. Each sprint reuses the 60-day
folders already in `EKS_Learning/` — this folder adds the **exam mechanics**.

### CKA Track (cluster admin & ops) — ~10 days

```
Day  1  Week 1 of plan: pods ↔ deployments ↔ services   → Day-01..03 READMEs
Day  2  Scheduling, taints/tolerations, nodeSelector     → Day-02 README
Day  3  Networking + NetworkPolicy                       → Day-03, Day-07 READMEs
Day  4  Storage (PV/PVC/StorageClass)                    → Day-05, Day-12 READMEs
Day  5  RBAC, ServiceAccounts, SecurityContext           → Day-11, Day-15 READMEs
Day  6  Cluster maintenance: drain, cordon, upgrade      → Day-08, Day-09 READMEs
Day  7  etcd backup + restore (the classic CKA task)     → Day-56 README
Day  8  Logging, tools, kubectl forms, troubleshooting   → Day-14 README
Day  9  Full timed run of CKA_Practice_Exam_Part1        → THIS FOLDER
Day 10  Weak-topic rerun + flashcards (Scheduling/DR)    → THIS FOLDER
```

### CKAD Track (workload authoring) — ~10 days

```
Day  1  Workloads: Deploy, RS, StatefulSet               → Day-02 README
Day  2  Services: ClusterIP, NodePort, LoadBalancer      → Day-03 README
Day  3  ConfigMaps + Secrets (env, volume, envFrom)      → Day-04, Day-06 READMEs
Day  4  Probes + resources + sidecars                    → Day-02 README
Day  5  Jobs + CronJobs + concurrency policy             → Day-02 README
Day  6  Storage: PVCs, ephemeral volumes, size limits    → Day-05, Day-21 READMEs
Day  7  SA + securityContext + labels/selectors          → Day-11, Day-15 READMEs
Day  8  Ingress (ALB in EKS) + rollouts/rollback         → Day-13 README
Day  9  Full timed run of CKAD_Practice_Exam_Part1       → THIS FOLDER
Day 10  Weak-topic rerun + flashcards (Workloads/Storage)→ THIS FOLDER
```

### AWS EKS Track (DevOps-Pro / SA) — ~14 days

```
Week 1  EKS foundations: control plane, eksctl, VPC     → Day-08..11 READMEs
Week 2  Operations: CNI, IAM, add-ons, upgrades, quotas  → Day-16, Day-26..30 READMEs
Week 3  Production patterns: ALB/NLB, EBS/EFS, CA/Karpenter, Fargate, Bottlerocket
                                                          → Day-13, Day-21, Day-53 READMEs
Week 4  Mock exam: AWS_EKS_Practice_Exam.md + flashcards → THIS FOLDER
```

---

## 🧠 How to Actually Study

1. **Type, never paste.** One point per task is often "did you get the exact API shape
   right?" — the muscle memory is the grade.
2. **Time yourself.** CKA gives ~2h/17 questions, CKAD ~2h/16-20 questions. Give each
   exam file 3 hours and stop at the deadline.
3. **Use `kubectl explain`** when stuck in a task — it works in the real exam too.
4. **Score, don't grade.** Redo the missed tasks twice more on separate days.
5. **Run the flashcard groups out loud** once per day — they reinforce the 60-day
   vocabulary the examiners reuse.

---

## 📚 Reference Anchors From the 60-Day Plan

- **CKA/CKAD core:** `EKS_Learning/Day-01_Kubernetes_Basics`, `Day-02_Pods_ReplicaSets_Deployments`, `Day-03_Services_Networking`, `Day-04_Namespaces_ConfigMaps_Secrets`
- **Storage:** `Day-05_Persistent_Storage`, `Day-21_EFS_CSI_Driver`, `Day-23_Volume_Snapshots_Restore`
- **Networking:** `Day-07_Kubernetes_Networking_Deep_Dive`, `Day-16_VPC_CNI_Plugin`, `Day-17_Network_Policies_Calico`
- **IAM/Security:** `Day-11_EKS_IAM_Roles_Policies`, `Day-26_OIDC_Provider_Setup`, `Day-27_IRSA_Multiple_Services`, `Day-15_EKS_Security_Best_Practices`
- **EKS ops:** `Day-08_Introduction_Amazon_EKS`, `Day-09_EKS_Cluster_Setup_eksctl`, `Day-10_EKS_Networking_VPC`, `Day-13_EKS_Load_Balancing_ALB`
- **Autoscaling/cost:** `Day-51_Spot_Instances_EKS`, `Day-53_Cluster_Autoscaler_Karpenter`
- **Backup/DR:** `Day-56_Cluster_Backup_Strategies`, `Day-58_Failover_Procedures`
- **Final review:** `Day-60_Final_Review_Certification_Preparation`

---

## 🚀 Before You Book the Exam

- [ ] Score ≥75% on `AWS_EKS_Practice_Exam.md` (AWS track)
- [ ] Complete all 40 performance tasks across the two CKA/CKAD files
- [ ] Recite all 60 flashcards without looking
- [ ] Rebuild the Day 60 capstone from memory (the 12-box grading rubric)
- [ ] Book the exam — now, before "one more day of studying" turns into a month

**Remember:** the plan gave you the skills; these files give you the exam shape.
Both matter. Go type. 🚀🏁