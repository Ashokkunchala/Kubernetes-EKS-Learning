# 🧩 EKS Learning Library — Bonus Modules

> Beyond the core 60 days: deep-dive modules that tackle the advanced topics you'll face in production EKS work and senior-level interviews.

Each module follows the same format as the day READMEs: Learning Objectives → Core Concepts → Hands-On → Troubleshooting → Interview Questions → Progress Tracker → Next.

```
┌─────────────────────────────────────────────────────────┐
│           MODULE LAYOUT (60-DAY LIBRARY MAP)             │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Days 1–10   Foundations + Cluster Setup         │   │
│  ├─────────────────────────────────────────────────┤   │
│  │  Days 11–30  IAM, Storage, Network Security      │   │
│  ├─────────────────────────────────────────────────┤   │
│  │  Days 31–45  GitOps, CI/CD, Monitoring           │   │
│  ├─────────────────────────────────────────────────┤   │
│  │  Days 46–60  Multi-Cluster, Cost, DR            │   │
│  └───────────────────────┬─────────────────────────┘   │
│                          ▼                             │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Modules A–J  (Advanced Topic Deep-Dives)        │   │
│  │  A  Admission Controllers                        │   │
│  │  B  RuntimeClass & gVisor Sandboxing             │   │
│  │  C  Kyverno Policy Engine                        │   │
│  │  D  Argo Rollouts (Progressive Delivery)         │   │
│  │  E  HPA / VPA / KEDA Autoscaling                 │   │
│  │  F  Local Development for EKS Practice           │   │
│  │  G  OPA Gatekeeper (Policy as Code / Rego)       │   │
│  │  H  Advanced Scheduling (Affinity, Topology,     │   │
│  │     Priority & Preemption)                       │   │
│  │  I  Certificates & TLS (cert-manager, ACME)      │   │
│  │  J  Runtime Security (SecurityContext, Seccomp,  │   │
│  │     AppArmor)                                    │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🎯 Learning Objectives

By studying the bonus modules, you will understand:
- How the Kubernetes API server filters and mutates every write request
- How to sandbox untrusted workloads with RuntimeClass and gVisor
- How to enforce organization policy centrally with Kyverno
- How to ship releases safely with canary/blue-green progressive delivery
- How to right-size autoscaling across HPA, VPA, and KEDA
- How to replicate EKS behavior on your laptop cheaply
- How to enforce policy as code with OPA Gatekeeper and Rego
- How to control exactly where pods land (affinity, topology, priority)
- How to automate certificates and TLS with cert-manager
- How to harden workloads with SecurityContext, Seccomp, and AppArmor

---

## 📖 Module Index

| Module | Topic | Strongest Pairing |
|--------|-------|-------------------|
| [Module A](./Module-A_Admission_Controllers.md) | Admission Controllers | [Day 15 Security](../Day-15_EKS_Security_Best_Practices/README.md) |
| [Module B](./Module-B_RuntimeClass_gVisor.md) | RuntimeClass & gVisor | [Day 15 Security](../Day-15_EKS_Security_Best_Practices/README.md) |
| [Module C](./Module-C_Kyverno_Engine.md) | Kyverno Policy Engine | [Day 15 Security](../Day-15_EKS_Security_Best_Practices/README.md) |
| [Module D](./Module-D_Argo_Rollouts.md) | Progressive Delivery | [Day 31 GitOps Intro](../Day-31_GitOps_ArgoCD_Intro/README.md) |
| [Module E](./Module-E_HPA_VPA_KEDA.md) | Autoscaling (HPA/VPA/KEDA) | [Day 53 Cluster Autoscaler & Karpenter](../Day-53_Cluster_Autoscaler_Karpenter/README.md) |
| [Module F](./Module-F_Local_Development.md) | Local Development | [Day 08 Introduction to Amazon EKS](../Day-08_Introduction_Amazon_EKS/README.md) |
| [Module G](./Module-G_OPA_Gatekeeper.md) | OPA Gatekeeper (Policy as Code) | [Module C Kyverno](./Module-C_Kyverno_Engine.md) · [Day 15 Security](../Day-15_EKS_Security_Best_Practices/README.md) |
| [Module H](./Module-H_Advanced_Scheduling.md) | Advanced Scheduling | [Day 02 Pods & Deployments](../Day-02_Pods_ReplicaSets_Deployments/README.md) · [Day 53 Cluster Autoscaler](../Day-53_Cluster_Autoscaler_Karpenter/README.md) |
| [Module I](./Module-I_Certificates_TLS.md) | Certificates & TLS (cert-manager) | [Day 13 ALB](../Day-13_EKS_Load_Balancing_ALB/README.md) · [Day 15 Security](../Day-15_EKS_Security_Best_Practices/README.md) |
| [Module J](./Module-J_Runtime_Security.md) | Runtime Security | [Module B gVisor](./Module-B_RuntimeClass_gVisor.md) · [Day 15 Security](../Day-15_EKS_Security_Best_Practices/README.md) |

---

## 🗺️ Suggested Reading Order

```
Cluster runs → (A) guard it → (B) sandbox it → (C) police it
   → (D) ship it safely → (E) scale it → (F) rebuild it locally
   → (G) enforce it (Rego) → (H) place it → (I) secure it (TLS) → (J) harden it
```

1. **Module F** first if you have no running cluster — it is the cheapest lab.
2. **Module A → B → C → J** — a security arc: API server gatekeeping, runtime isolation, org-wide policy, then per-workload hardening (Seccomp/AppArmor/SecurityContext).
3. **Module G** after Module C — the Rego-based alternative to Kyverno's YAML policies; pick your engine.
4. **Module H** before autoscaling — it decides *where* pods land; then Module E decides *how fast*.
5. **Module I** after Day 13 — automate the TLS your ALB/Ingress needs.
6. **Module D** after the ArgoCD days (31–35) — it is the delivery layer on top of GitOps.
7. **Module E** after Day 52/53 — it decides *how fast* the cluster autoscaler needs to react.

---

## 🛠️ Troubleshooting Tips (Shared Across Modules)

```bash
# Everything in Kubernetes is a webhook or a controller — always check logs:
kubectl logs -n kube-system <controller-pod> --tail=50

# Admission webhook problems usually look like this:
#  Error from server: Internal error occurred: failed calling webhook ...

# Certificate issues are the #1 webhook failure — recheck the CA bundle:
kubectl get validatingwebhookconfiguration <name> -o yaml | grep -A5 clientConfig

# Controller-manager filters (PodNodeSelector, etc.) require a fresh kube-apiserver:
# an EKS control plane add-on restart is managed by AWS — rebuild flags via add-on config.
```

---

## ✅ Progress Tracker

```
60-Day Core Library:
  Days 1–10  ✅
  Days 11–20 ✅
  Days 21–30 ✅
  Days 31–40 ✅
  Days 41–50 ✅
  Days 51–60 ✅

Bonus Modules:
  Module A  Admission Controllers         ⬜
  Module B  RuntimeClass & gVisor         ⬜
  Module C  Kyverno Policy Engine         ⬜
  Module D  Argo Rollouts                 ⬜
  Module E  HPA / VPA / KEDA              ⬜
  Module F  Local Development             ⬜
  Module G  OPA Gatekeeper                ⬜
  Module H  Advanced Scheduling           ⬜
  Module I  Certificates & TLS            ⬜
  Module J  Runtime Security              ⬜
```

---

## 👀 Next

- Finished **Module A?** Go to [Module C Kyverno](./Module-C_Kyverno_Engine.md) — webhooks handwritten in A become policies in C.
- Finished **Module C?** Compare engines in [Module G OPA Gatekeeper](./Module-G_OPA_Gatekeeper.md), then harden workloads in [Module J Runtime Security](./Module-J_Runtime_Security.md).
- Finished **Module D?** Revisit the full [GitOps arc](../Day-35_ArgoCD_Advanced_Best_Practices/README.md) to see where Rollouts plug into ArgoCD.
- Finished **Module E?** Go back to [Day 53](../Day-53_Cluster_Autoscaler_Karpenter/README.md) to connect pod-level scaling to node-level scaling.
- Finished **Module F?** Stand up a real cluster with [Day 09 eksctl setup](../Day-09_EKS_Cluster_Setup_eksctl/README.md).
- Finished **Module G?** Place and harden what you just admitted: [Module H Advanced Scheduling](./Module-H_Advanced_Scheduling.md) then [Module J Runtime Security](./Module-J_Runtime_Security.md).
- Finished **Module H?** Tie placement to scaling with [Day 53](../Day-53_Cluster_Autoscaler_Karpenter/README.md).
- Finished **Module I?** Turn certs into mTLS with [Day 18 Istio](../Day-18_Service_Mesh_Istio/README.md).
- Finished **Module J?** Go full isolation with [Module B RuntimeClass & gVisor](./Module-B_RuntimeClass_gVisor.md).

---

**Remember:** Certifications prove the basics — modules prove the production instincts. 🚀