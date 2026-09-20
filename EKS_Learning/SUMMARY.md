# 📋 EKS Learning Plan Summary

## 🎉 What's Been Created

A complete **60-day EKS learning plan** with **60 detailed day-by-day files**
covering everything from bare Kubernetes basics to production-grade, multi-region
EKS operations:

### ✅ Phase 1: Foundation (Days 1-15) — COMPLETE

**Week 1: Kubernetes Basics**
1. What is Kubernetes?
2. Pods, ReplicaSets, Deployments
3. Services and Networking
4. Namespaces, ConfigMaps, Secrets
5. Persistent Storage
6. ConfigMaps and Secrets Practice
7. Kubernetes Networking Deep Dive

**Week 2: AWS EKS Fundamentals**
8. Introduction to Amazon EKS
9. EKS Cluster Setup with eksctl
10. EKS Networking and VPC
11. EKS IAM Roles and Policies
12. EKS Storage with EBS
13. EKS Load Balancing with ALB
14. EKS Monitoring and Logging
15. EKS Security Best Practices

### ✅ Phase 2–4: Advanced to Expert (Days 16-60) — CONTENT COMPLETE

**Advanced Networking & Storage (16-25):** VPC CNI, Calico, Istio, EFS, FSx,
S3 CSI, snapshots, practice + interview prep

**IAM & Security (26-30):** OIDC, IRSA for multiple services, pod-level IAM,
Secrets Manager, advanced practice

**GitOps & CI/CD (31-40):** ArgoCD (intro → advanced), CodePipeline, GitHub
Actions, Jenkins, pipeline practice + interview prep

**Observability (41-45):** Prometheus Operator, Grafana, X-Ray, custom metrics,
monitoring stack practice

**Multi-Cluster (46-50):** EKS Anywhere, multi-cluster networking, federation,
practice + interview prep

**Cost Optimization (51-55):** Spot instances, right-sizing, Cluster Autoscaler
vs Karpenter, practice + interview prep

**Disaster Recovery & Certification (56-60):** Backup strategies, cross-region
replication, failover procedures, **DR Game Day practice**, and the **final
review & certification preparation** finale

---

## 📁 File Structure

```
K8/
├── README.md                          ← Master index & 60-day tracker
├── GLOSSARY.md                        ← A-to-Z EKS/K8s term dictionary
├── ARCHITECTURE_PATTERNS.md           ← 9 production design patterns
├── TROUBLESHOOTING_PLAYBOOK.md        ← 10 incident runbooks + triage
├── EKS_LEARNING_PLAN.md               ← The 60-day plan (with diagrams)
├── ENHANCED_EKS_LEARNING_PLAN.md      ← Enhanced plan + production issues
├── EKS_PROJECTS.md                    ← 6 real-time implementation projects
├── EKS_QUICK_REFERENCE.md             ← Command cheat sheet
├── EKS_FOUNDATION_REVIEW.md           ← Consolidated review (Days 1-15)
└── EKS_Learning/
    ├── README.md                      ← Day-by-day index
    ├── SUMMARY.md                     ← This file
    ├── manifests/                     ← 50+ runnable YAML files (8 areas)
    ├── modules/                       ← 10 bonus curriculum modules
    ├── labs/                          ← 4 hands-on labs with solutions
    ├── capstone/                      ← Full e-commerce microservices project
    ├── certification/                 ← CKA/CKAD/AWS practice + flashcards
    └── Day-01_Kubernetes_Basics/
    │   └── README.md
    ├── Day-02_Pods_ReplicaSets_Deployments/
    │   └── README.md
    ├── ...
    └── Day-60_Final_Review_Certification_Preparation/
        └── README.md
```

> **60 / 60 day folders now contain full content.** The final review
> (Day 60) includes the complete knowledge blueprint, certification paths,
> master interview Q&A, and the graduation capstone.

---

## 📚 Content in Each Day File

Every day file includes:
1. **Learning Objectives** — clear goals for the day
2. **Core Concepts** — detailed explanations with ASCII diagrams
3. **Hands-On Step-by-Step Tutorials** — commands + YAML to run
4. **Practice Exercises / Labs** — real-world scenarios to verify
5. **Troubleshooting** — common issues and how to diagnose them
6. **Interview Questions** — beginner → intermediate → senior + EKS specific
7. **Resources** — official docs, videos, links
8. **Assignment** — concrete tasks to complete
9. **Progress Tracker** — mark completion per day
10. **Next Day/Week Preview** — what's coming and why it connects

---

## 🧩 Beyond the 60 Days — Bonus Content

| Bonus area | What it contains | Go there |
|------------|------------------|----------|
| 🗂️ **Manifests library** | 50+ real YAML files (app, networking, storage, security, GitOps, monitoring, DR, cost) | [`manifests/`](manifests/README.md) |
| 📚 **Extra modules** | 10 modules: Admission Controllers, RuntimeClass/gVisor, Kyverno, Argo Rollouts, HPA/VPA/KEDA, Local dev, OPA Gatekeeper, Advanced Scheduling, Certificates & TLS, Runtime Security | [`modules/`](modules/README.md) |
| 🧪 **Hands-on labs** | 4 self-paced labs with solutions | [`labs/`](labs/README.md) |
| 🏆 **E-commerce capstone** | Full 15-manifest microservices project | [`capstone/`](capstone/README.md) |
| 📝 **Certification bank** | CKA + CKAD performance tasks, AWS EKS exam, flashcards | [`certification/`](certification/README.md) |
| 📖 **Reference docs** | Glossary, architecture patterns, troubleshooting playbook | [`../GLOSSARY.md`](../GLOSSARY.md) |
| 📊 **Progress tracking** | Master day tracker | [`../README.md`](../README.md) |

---

## 🎯 How to Use This Learning Plan

### Daily Schedule
1. **Morning (2 hours)**
   - 30 min: Read the day's documentation (the day README)
   - 30 min: Watch related videos
   - 1 hour: Hands-on practice in your own AWS account
2. **Evening (1 hour)**
   - 30 min: Review concepts (explain out loud — Feynman technique)
   - 30 min: Practice interview questions from the day's file

### Weekly Review
- **Weekend:** Build a mini-project from [`EKS_PROJECTS.md`](../EKS_PROJECTS.md)
- **Weekend:** Document learnings
- **Weekend:** Review the week's interview questions

### Milestone Reviews
- **After Day 15:** Read [`EKS_FOUNDATION_REVIEW.md`](../EKS_FOUNDATION_REVIEW.md)
- **After Day 60:** Use
  [`Day-60`](Day-60_Final_Review_Certification_Preparation/README.md) to prep
  for certification and job interviews

---

## 📊 Progress Tracking

Track your journey in the master index ([`README.md`](../README.md)) — every
day links to its lesson:

```
Day  1-15:  ✅ (completed)
Day 16-60:  ⬜ (in progress — mark each as you go)
```

---

## 🎯 Success Metrics

By the end of 60 days, you should be able to:
- ✅ Create and manage EKS clusters with eksctl / IaC
- ✅ Deploy, scale, and self-heal applications on EKS
- ✅ Configure networking (VPC CNI, Calico, Istio), storage, and security
- ✅ Set up monitoring + logging + tracing (Prometheus, Grafana, CloudWatch, X-Ray)
- ✅ Implement GitOps (ArgoCD) and CI/CD pipelines
- ✅ Design production-grade, multi-region architectures
- ✅ Optimize cost (spot, Karpenter, right-sizing)
- ✅ Recover from disasters (Velero, cross-region failover, Game Days)
- ✅ Pass EKS/Kubernetes certification exams (CKA, CKAD, AWS DevOps)
- ✅ Answer interview questions confidently

---

## 🚀 Getting Started

1. **Start with [Day 1](Day-01_Kubernetes_Basics/README.md)** and go sequentially
2. **Complete each day's exercises** before moving on
3. **Build the projects** ([`EKS_PROJECTS.md`](../EKS_PROJECTS.md)) to apply knowledge
4. **Review & certify** at the end ([Day 60](Day-60_Final_Review_Certification_Preparation/README.md))

---

**Good luck on your EKS learning journey! 🚀**

**Remember:** Consistency is key! Practice every day, build projects, and
document your learnings. You've got this!

*Summary refreshed: September 2026*