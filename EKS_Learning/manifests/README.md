# 📦 EKS Learning - Production-Ready Kubernetes Manifests

> **Purpose:** Central library of real, runnable Kubernetes / EKS resource manifests used throughout the 60-day program.
> Every YAML file conforms to latest stable API versions and can be applied with `kubectl apply -f <file>`.

## 📂 Folder Index

| Folder | What it contains | Related days |
| --- | --- | --- |
| [`app/`](app/) | Workload manifests: Deployments, Services, ConfigMaps, Secrets | [Day-02](../Day-02_Pods_ReplicaSets_Deployments/README.md), [Day-04](../Day-04_Namespaces_ConfigMaps_Secrets/README.md) |
| [`networking/`](networking/) | Services, Ingress, Network Policies, ALB/NLB configs | [Day-03](../Day-03_Services_Networking/README.md), [Day-13](../Day-13_EKS_Load_Balancing_ALB/README.md), [Day-17](../Day-17_Network_Policies_Calico/README.md) |
| [`storage/`](storage/) | EBS/EFS StorageClasses, PVCs, VolumeSnapshots | [Day-12](../Day-12_EKS_Storage_EBS/README.md), [Day-21](../Day-21_EFS_CSI_Driver/README.md), [Day-23](../Day-23_Volume_Snapshots_Restore/README.md) |
| [`security/`](security/) | Namespaces with PSS, SA + IRSA, RBAC, Network Policies, Pod Security, Quotas | [Day-11](../Day-11_EKS_IAM_Roles_Policies/README.md), [Day-15](../Day-15_EKS_Security_Best_Practices/README.md), [Day-28](../Day-28_Pod_Level_IAM_Permissions/README.md) |
| [`gitops/`](gitops/) | ArgoCD Application + AppProject for GitOps delivery | [Day-31](../Day-31_GitOps_ArgoCD_Intro/README.md), [Day-32](../Day-32_ArgoCD_Installation_Config/README.md), [Day-33](../Day-33_ArgoCD_Application_Patterns/README.md) |
| [`monitoring/`](monitoring/) | Prometheus scrape config, alert rules, Alertmanager, Grafana provisioning | [Day-14](../Day-14_EKS_Monitoring_Logging/README.md), [Day-41](../Day-41_Prometheus_Operator/README.md), [Day-42](../Day-42_Grafana_Dashboards/README.md) |
| [`dr/`](dr/) | Velero Schedules/Backups/Restores, Route53 + RDS failover runbooks | [Day-56](../Day-56_Cluster_Backup_Strategies/README.md), [Day-57](../Day-57_Cross_Region_Replication/README.md), [Day-58](../Day-58_Failover_Procedures/README.md) |
| [`cost/`](cost/) | Karpenter Provisioner, Spot node groups, quotas for cost control | [Day-51](../Day-51_Spot_Instances_EKS/README.md), [Day-52](../Day-52_Right_Sizing_Nodes/README.md), [Day-53](../Day-53_Cluster_Autoscaler_Karpenter/README.md) |

## 🚀 How to use

- Manifests are grouped by concern, not by day, so a day may reference files across folders.
- Always review placeholders (`ACCOUNT_ID`, `your-org`, `my-cluster`) before applying to a live cluster.
- Apply order matters for RBAC/network policy scenarios: `namespace.yaml` first, then ServiceAccount/Role, then workloads.