# 🛒 Capstone: E-Commerce Microservices on EKS

Your final proving ground. Every concept from the 60-day plan lands here in one deplorable, observable, recoverable microservices application. By the end you will have deployed a full e-commerce stack behind an ALB Ingress — and wired it to everything a production platform needs: GitOps, autoscaling, monitoring, RBAC, and backup.

## 🎯 Capstone Goal

Build and operate a complete e-commerce microservices platform on Amazon EKS:

- **Ingress** (ALB) → static **frontend** + 4 microservices (**product / order / user / payment**)
- Backing **Redis** cache + **PostgreSQL** database
- Wrapped in the production toolbox: GitOps (ArgoCD), autoscaling (HPA) + PDB, RBAC, monitoring (Prometheus/Grafana), and DR (Velero)

## 🏗️ Architecture

```
                          Internet
                             │
                             ▼
                    ┌─────────────────┐         ┌───────────────────┐
                    │   ALB Ingress   │         │  Monitoring       │
                    │ (AWS LB Ctrl.)  │ ───────►│  Prometheus (41)  │
                    └────────┬────────┘         │  Grafana (42)     │
                             │ *.example.com   └─────────┬─────────┘
              ┌──────────────┼───────────────┐          │
              ▼              ▼               ▼          │
        ┌───────────┐  ┌───────────┐  ┌───────────┐    │
        │ frontend  │  │ product   │  │  order    │    │
        │ nginx :80 │  │ :8080 ×3  │  │ :8080 ×2  │    │
        │ (nginx)   │  │(+metrics) │  │           │    │
        └───────────┘  └─────┬─────┘  └─────┬─────┘    │
                             │             │           │
                   ┌─────────┼─────────────┼─────────┐ │
                   ▼         ▼             ▼         │ │
             ┌───────────┐ ┌───────────┐ ┌─────────┐ │ │
             │   user    │ │  payment  │ │ Redis   │ │ │
             │  :8080 ×2 │ │ :8080 ×1  │ │ cache   │ │ │
             └─────┬─────┘ └─────┬─────┘ └────┬────┘ │ │
                   │             │             │       │ │
                   └──────┬──────┴──────┬──────┘       │ │
                          ▼             ▼              │ │
                   ┌──────────────────────────────┐    │ │
                   │  PostgreSQL (RDS on EKS)     │    │ │
                   │  persistent, backed up       │    │ │
                   └──────────────────────────────┘    │ │
                                                       │ │
          ┌────────────────────────────────────────────┼─┘
          │  GitOps:   ArgoCD (31) Git repo ──► apply ecommerce/
          │  Security: RBAC read-only (15) · IRSA (27) · NetPol (17)
          │  Scaling:  HPA cpu 60% (53) · PDB (02)
          │  DR:       Velero daily backups (56)
          └────────────────────────────────────────────┘
```

## ✅ Prerequisites

| Component | Why | Reference |
|-----------|-----|-----------|
| EKS cluster (≥ 2 nodes) + `kubectl` + `kubectl apply` | Everything runs here | [Day-09](../Day-09_EKS_Cluster_Setup_eksctl/README.md) |
| AWS Load Balancer Controller | Provisions the ALB for the Ingress | [Day-13](../Day-13_EKS_Load_Balancing_ALB/README.md) |
| Metrics Server | powers HPA CPU autoscaling | [Day-53](../Day-53_Cluster_Autoscaler_Karpenter/README.md) |
| EBS CSI driver (and optionally EFS) | storage for Redis/PostgreSQL/PVCs | [Day-12](../Day-12_EKS_Storage_EBS/README.md), [Day-21](../Day-21_EFS_CSI_Driver/README.md) |
| ArgoCD installed in `argocd` ns | GitOps sync of the `ecommerce/` path | [Day-32](../Day-32_ArgoCD_Installation_Config/README.md) |
| Prometheus Operator (kube-prometheus-stack) committed | ServiceMonitor + alerts | [Day-41](../Day-41_Prometheus_Operator/README.md) |
| Velero installed in `velero` ns | scheduled backups | [Day-56](../Day-56_Cluster_Backup_Strategies/README.md) |
| Calico add-on + IRSA enabled | (optional hardening) | [Day-17](../Day-17_Network_Policies_Calico/README.md), [Day-27](../Day-27_IRSA_Multiple_Services/README.md) |

> 💡 Replace the placeholder repo URL and image names (`your-org/...`) with your own before step 8.

## 📋 Build Checklist (mapped to your 60 days)

Work through in order — each row is an `kubectl apply` that flags a skill from a specific day. Stop and re-read the day before crossing it off.

| # | File | What it proves | Revisit Day(s) |
|---|------|----------------|----------------|
| 0 | `ecommerce/00-namespace.yaml` | Isolation & resource organization | [Day-04](../Day-04_Namespaces_ConfigMaps_Secrets/README.md) |
| 1 | `ecommerce/01-configmap.yaml` | Config as data | [Day-04](../Day-04_Namespaces_ConfigMaps_Secrets/README.md) |
| 2 | `ecommerce/02-secret.yaml` | Secrets (then graduate to Secrets Manager) | [Day-06](../Day-06_ConfigMaps_Secrets_Practice/README.md), [Day-29](../Day-29_Secrets_Manager_Integration/README.md) |
| 3 | `ecommerce/03-frontend.yaml` | Deployment + Service | [Day-02](../Day-02_Pods_ReplicaSets_Deployments/README.md), [Day-03](../Day-03_Services_Networking/README.md) |
| 4 | `ecommerce/04-product-service.yaml` | Probes, resources, multi-replica | [Day-02](../Day-02_Pods_ReplicaSets_Deployments/README.md) |
| 5 | `ecommerce/05-order-service.yaml` | Same pattern, 2 replicas | [Day-02](../Day-02_Pods_ReplicaSets_Deployments/README.md) |
| 6 | `ecommerce/06-user-service.yaml` | Same pattern, 2 replicas | [Day-02](../Day-02_Pods_ReplicaSets_Deployments/README.md) |
| 7 | `ecommerce/07-payment-service.yaml` | Same pattern, 1 replica | [Day-02](../Day-02_Pods_ReplicaSets_Deployments/README.md) |
| 8 | `ecommerce/08-ingress.yaml` | ALB + path routing | [Day-13](../Day-13_EKS_Load_Balancing_ALB/README.md) |
| 9 | `ecommerce/09-rbac.yaml` | Least-privilege RBAC | [Day-11](../Day-11_EKS_IAM_Roles_Policies/README.md), [Day-15](../Day-15_EKS_Security_Best_Practices/README.md) |
| 10 | `ecommerce/10-hpa-pdb.yaml` | Autoscaling + disruption budget | [Day-53](../Day-53_Cluster_Autoscaler_Karpenter/README.md) |
| 11 | `ecommerce/11-argocd-app.yaml` | GitOps delivery | [Day-31](../Day-31_GitOps_ArgoCD_Intro/README.md)–[Day-34](../Day-34_ArgoCD_Sync_Rollback/README.md) |
| 12 | `ecommerce/12-monitoring.yaml` | ServiceMonitor, alerts, Grafana datasource | [Day-41](../Day-41_Prometheus_Operator/README.md), [Day-42](../Day-42_Grafana_Dashboards/README.md) |
| 13 | `ecommerce/13-velero-schedule.yaml` | Scheduled disaster recovery | [Day-56](../Day-56_Cluster_Backup_Strategies/README.md)–[Day-59](../Day-59_Disaster_Recovery_Practice/README.md) |
| 14 | `ecommerce/14-README-deployment.md` | The runbook — apply order + cleanup | all |

**Optional extensions** (don't stop at 14):

- **Network Policy per service** → [Day-17](../Day-17_Network_Policies_Calico/README.md) + microsegmentation drill in [Lab-03](../labs/Lab-03_Network_Policies.md)
- **IRSA** so product/payment can call AWS APIs (SQS, KMS, S3) → [Day-27](../Day-27_IRSA_Multiple_Services/README.md)
- **VPC CNI** deep-dive: per-pod IP exhaustion, `ENABLE_PREFIX_DELEGATION` → [Day-16](../Day-16_VPC_CNI_Plugin/README.md)
- **Canary rollout** of the frontend via ArgoCD `Rollouts` → [Day-35](../Day-35_ArgoCD_Advanced_Best_Practices/README.md)
- **Cost controls**: right-size requests, Karpenter for the compute layer → [Day-52](../Day-52_Right_Sizing_Nodes/README.md), [Day-53](../Day-53_Cluster_Autoscaler_Karpenter/README.md)

## ▶️ Quick Start

```bash
# 1. Be on your EKS cluster
aws eks update-kubeconfig --region <region> --name <cluster>

# 2. Everything in the ecommerce namespace (files 00 → 07)
kubectl apply -f ecommerce/00-namespace.yaml
kubectl apply -f ecommerce/01-configmap.yaml
kubectl apply -f ecommerce/02-secret.yaml
kubectl apply -f ecommerce/03-frontend.yaml
kubectl apply -f ecommerce/04-product-service.yaml
kubectl apply -f ecommerce/05-order-service.yaml
kubectl apply -f ecommerce/06-user-service.yaml
kubectl apply -f ecommerce/07-payment-service.yaml

# 3. ALB Ingress (takes 2-4 min to provision)
kubectl apply -f ecommerce/08-ingress.yaml
kubectl get ingress -n ecommerce -w     # watch ADDRESS fill in

# 4. Hardening + platform add-ons
kubectl apply -f ecommerce/09-rbac.yaml
kubectl apply -f ecommerce/10-hpa-pdb.yaml
kubectl apply -f ecommerce/11-argocd-app.yaml
kubectl apply -f ecommerce/12-monitoring.yaml
kubectl apply -f ecommerce/13-velero-schedule.yaml
```

See [ecommerce/14-README-deployment.md](ecommerce/14-README-deployment.md) for the full runbook — wait, that's files **00→14** in strict order with verification and cleanup.

## 🧹 Cleanup

```bash
kubectl delete ns ecommerce
# optional: delete ArgoCD Application first so it doesn't recreate the namespace
kubectl delete -f ecommerce/11-argocd-app.yaml
```

## 📚 Resources

- The [60-day plan](../README.md) — skip nothing, but especially Days 13, 31, 41, 56
- [Hands-on labs](../labs/README.md) — run the four drills before starting the capstone
- [Runnable manifests](../manifests/README.md) — reference versions of app, networking, and storage manifests

**This capstone is the person you present in interviews. Make it boring, observable, and recoverable. 🚀**