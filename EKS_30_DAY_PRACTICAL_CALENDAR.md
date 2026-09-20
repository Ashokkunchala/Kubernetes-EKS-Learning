# EKS 30-Day Practical Engineering Calendar

Use this execution track alongside the existing 60-day curriculum.

| Day | Focus | Deliverable |
|---|---|---|
| 1 | Cluster anatomy | Draw kubectl -> API -> controllers -> node -> pod flow |
| 2 | Workloads | Deployment + probes + resources |
| 3 | Services | ClusterIP + DNS + endpoint debugging |
| 4 | Configuration | ConfigMap + Secret + projected volume |
| 5 | Scheduling | taints, tolerations, affinity, topology spread |
| 6 | Storage | StorageClass + PVC + CSI-backed volume |
| 7 | Incident drill | Break a workload and recover it |
| 8 | EKS architecture | AWS + EKS control/data plane diagram |
| 9 | Sandbox cluster | Build and document a test EKS cluster |
| 10 | Add-ons | Inventory, version, validate, document |
| 11 | IAM | Prove service-account AWS permissions |
| 12 | Pod Identity | Implement and compare with IRSA |
| 13 | VPC CNI | IPAM and pod-IP exhaustion drill |
| 14 | Load balancing | ALB health-check drill |
| 15 | Incident drill | Diagnose a synthetic 503 |
| 16 | Network security | default-deny + allow-list |
| 17 | Pod security | PSS + securityContext + non-root |
| 18 | Secrets | Secrets Manager integration |
| 19 | Metrics | Prometheus/Grafana workload dashboard |
| 20 | Logs/traces | CloudWatch + tracing investigation |
| 21 | Pod autoscaling | HPA exercise |
| 22 | Node autoscaling | Karpenter/Cluster Autoscaler comparison |
| 23 | GitOps | Argo CD reconciliation workflow |
| 24 | Progressive delivery | canary or blue/green rollback |
| 25 | CI/CD | PR -> test -> image -> ECR -> deploy |
| 26 | Upgrade | compatibility checklist + rehearsal |
| 27 | Backup | backup + namespace restore |
| 28 | DR | restore + failover tabletop |
| 29 | Capstone | secure multi-service deployment |
| 30 | Interview | 25 scenarios + architecture whiteboard |

## Daily loop

`20 min learn -> 40 min build -> 20 min break/fix -> 10 min notes -> 10 min interview answer`

## Completion rule

Do not mark a day complete because you read it.

A day is complete only when you can:

`RUN -> OBSERVE -> BREAK -> RECOVER -> EXPLAIN`

At the end of every 7 days, rebuild one lab without notes.
