# 📋 Kubernetes (K8s) — Complete DevOps Keyword Master Checklist

> Every keyword from `Kubernetes_K8s_Complete_DevOps_Keywords.txt` (47 categories)
> turned into an **interactive study checklist**, cross-referenced to where each
> topic is taught in this library. Tick boxes as you master each keyword.
>
> **Legend:** ✅ already covered in the library · 🔴 **NEW** — added to
> [`GLOSSARY.md`](GLOSSARY.md) so coverage is complete · ✅→ points to exact resource(s).

---

## 🗺️ Category Coverage Index

| # | Category | Primary coverage | Status |
|---|----------|------------------|--------|
| 01 | Fundamentals | [Day-01](../K8/EKS_Learning/Day-01_Kubernetes_Basics/README.md), [Day-02] | ✅ |
| 02 | Architecture | [Day-01], [Day-08] | ✅ |
| 03 | Pods | [Day-02] | ✅ |
| 04 | Deployments | [Day-02], [Module-D] | ✅ |
| 05 | ReplicaSets | [Day-02] | ✅ |
| 06 | Services | [Day-03] | ✅ |
| 07 | Ingress | [Day-03], [Day-13] | ✅ |
| 08 | Networking | [Day-07], [Day-16], [Day-17] | ✅ |
| 09 | ConfigMaps & Secrets | [Day-04], [Day-06], [Day-29] | ✅ |
| 10 | Health Checks / Probes | [Day-02] | ✅ |
| 11 | Resource Management | [Day-02], [Day-52], [Module-E] | ✅ |
| 12 | Scaling | [Day-53], [Module-E] | ✅ |
| 13 | StatefulSets | [Day-05] | ✅ |
| 14 | DaemonSets | [Day-02] | ✅ |
| 15 | Jobs & CronJobs | [Day-02], [Day-56] | ✅ |
| 16 | Storage | [Day-05], [Day-12], [Day-21]–[Day-23] | ✅ |
| 17 | Scheduling | [Day-02], [Day-15], [**Module-H**](EKS_Learning/modules/Module-H_Advanced_Scheduling.md) | ✅ |
| 18 | Namespaces & Isolation | [Day-04] | ✅ |
| 19 | RBAC & Security | [Day-15], [Module-A] | ✅ |
| 20 | Security Context | [Day-15], [Module-B], [**Module-J**](EKS_Learning/modules/Module-J_Runtime_Security.md) | ✅ |
| 21 | Container Images | [Day-15], [Day-37] | ✅ |
| 22 | Helm | [Day-32] | ✅ |
| 23 | Kustomize | [Day-33], [Module-D] | ✅ |
| 24 | GitOps | [Day-31]–[Day-35] | ✅ |
| 25 | CI/CD with Kubernetes | [Day-36]–[Day-40] | ✅ |
| 26 | AWS EKS | [Day-08]–[Day-15] | ✅ |
| 27 | Monitoring | [Day-14], [Day-41], [Day-45] | ✅ |
| 28 | Logging | [Day-14], [Day-45] | ✅ |
| 29 | Observability | [Day-41]–[Day-45] | ✅ |
| 30 | Troubleshooting | [TROUBLESHOOTING_PLAYBOOK.md] | ✅ |
| 31 | kubectl Commands | [EKS_QUICK_REFERENCE.md] | ✅ |
| 32 | Node Maintenance | [Day-02], [Day-56] | ✅ |
| 33 | High Availability | [Day-56]–[Day-58] | ✅ |
| 34 | Advanced Kubernetes | [Module-A], [Day-60] | ✅ |
| 35 | Service Mesh | [Day-18] | ✅ |
| 36 | Deployment Strategies | [Day-02], [Day-34], [Module-D] | ✅ |
| 37 | Network Security | [Day-17] | ✅ |
| 38 | Production Operations | [Day-52]–[Day-59] | ✅ |
| 39 | YAML Core Fields | [manifests/](EKS_Learning/manifests/README.md) | ✅ |
| 40 | Important Objects | [GLOSSARY.md], all Days | ✅ |
| 41 | Production Tools | spread across Days + Modules | ✅ |
| 42 | Certificates & TLS | [Day-13], [Day-15], [Day-46], [**Module-I**](EKS_Learning/modules/Module-I_Certificates_TLS.md) | ✅ |
| 43 | Policy & Governance | [Module-C], [**Module-G**](EKS_Learning/modules/Module-G_OPA_Gatekeeper.md), [Day-15] | ✅ |
| 44 | Backup & Disaster Recovery | [Day-56]–[Day-59] | ✅ |
| 45 | Multi-Cluster | [Day-46]–[Day-50] | ✅ |
| 46 | Interview Concepts | [Day-60], [certification/] | ✅ |
| 47 | Core DevOps + K8s | whole library | ✅ |

> **Note on links:** use the paths relative to this file's folder (same root as
> the day folders). Click through to each resource from the indexes:
> [Day index](EKS_Learning/README.md) · [Modules](EKS_Learning/modules/README.md)
> · [Manifests](EKS_Learning/manifests/README.md).

---

## 01 · KUBERNETES FUNDAMENTALS
**Primary coverage:** [Day-01](EKS_Learning/Day-01_Kubernetes_Basics/README.md) · [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md) · [Day-03](EKS_Learning/Day-03_Services_Networking/README.md)

- [ ] Kubernetes · K8s · Container Orchestration
- [ ] Cluster · Node · Worker Node · Control Plane · Master Node
- [ ] Kubernetes Object · Declarative Configuration · Desired State · Actual State · Reconciliation
- [ ] Kubernetes Manifest · YAML · JSON
- [ ] Labels · Selectors · Annotations
- [ ] Namespaces · Context · kubeconfig
- [ ] Kubernetes API · API Resource · API Group · API Version · Resource Version
- [ ] Field Selector ✅ ([Day-60](EKS_Learning/Day-60_Final_Review_Certification_Preparation/README.md), kubectl)
- [ ] Label Selector
- [ ] Owner References ✅ ([CKA Part1](EKS_Learning/certification/CKA_Practice_Exam_Part1.md))
- [ ] Finalizers 🔴 → [GLOSSARY](GLOSSARY.md)
- [ ] Garbage Collection 🔴 → [GLOSSARY](GLOSSARY.md)

## 02 · KUBERNETES ARCHITECTURE
**Primary coverage:** [Day-01](EKS_Learning/Day-01_Kubernetes_Basics/README.md) · [Day-08](EKS_Learning/Day-08_Introduction_Amazon_EKS/README.md)

- [ ] API Server · kube-apiserver · etcd · kube-scheduler · kube-controller-manager · Cloud Controller Manager
- [ ] kubelet · kube-proxy
- [ ] Container Runtime · CRI · Containerd · CRI-O
- [ ] Kubernetes Controller · Operator · Kubernetes Operator Pattern
- [ ] Admission Controller · Admission Webhook · Mutating Admission Webhook · Validating Admission Webhook
- [ ] Control Plane · Worker Node · Node Heartbeat · Node Conditions

## 03 · PODS
**Primary coverage:** [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md) · [Module-B](EKS_Learning/modules/Module-B_RuntimeClass_gVisor.md)

- [ ] Pod · Pod Template · Single-Container Pod · Multi-Container Pod
- [ ] Pod Lifecycle · Pod Phase · Pending · Running · Succeeded · Failed · Unknown
- [ ] Pod IP · Pod UID · Pod Sandbox
- [ ] Container Lifecycle · Container State · Waiting · Running · Terminated · Exit Code · Restart Count
- [ ] Restart Policy · Always · OnFailure · Never
- [ ] Init Container · Sidecar Container · Ephemeral Container
- [ ] Shared Network Namespace · Shared Volumes
- [ ] Pod Security Context · Pod-Level Security Context

## 04 · DEPLOYMENTS
**Primary coverage:** [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md) · [Module-D](EKS_Learning/modules/Module-D_Argo_Rollouts.md)

- [ ] Deployment · ReplicaSet · Replica Count
- [ ] Desired Replicas · Available Replicas · Ready Replicas · Updated Replicas
- [ ] Deployment Strategy · RollingUpdate · Recreate · Rolling Deployment · Zero-Downtime Deployment
- [ ] Revision · Revision History · Rollout · Rollback
- [ ] Deployment Progress · Progress Deadline
- [ ] maxSurge · maxUnavailable
- [ ] Paused Rollout · Resumed Rollout
- [ ] Self-Healing · Scaling

## 05 · REPLICASETS
**Primary coverage:** [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md)

- [ ] ReplicaSet · ReplicaSet Controller · Pod Selector
- [ ] Desired Replica Count · Current Replica Count · Available Replica Count
- [ ] Self-Healing · Horizontal Scaling

## 06 · SERVICES
**Primary coverage:** [Day-03](EKS_Learning/Day-03_Services_Networking/README.md) · [Day-20](EKS_Learning/Day-20_Networking_Interview_Preparation/README.md)

- [ ] Service · Service Discovery · ClusterIP · NodePort · LoadBalancer · ExternalName
- [ ] Headless Service
- [ ] Service Selector · Endpoints · EndpointSlice
- [ ] Virtual IP · kube-proxy
- [ ] Service DNS
- [ ] Service Port · Target Port · Node Port
- [ ] Session Affinity
- [ ] External Traffic Policy · Internal Traffic Policy

## 07 · INGRESS
**Primary coverage:** [Day-03](EKS_Learning/Day-03_Services_Networking/README.md) · [Day-13](EKS_Learning/Day-13_EKS_Load_Balancing_ALB/README.md)

- [ ] Ingress · Ingress Controller · Ingress Class · Ingress Rules
- [ ] Host-Based Routing · Path-Based Routing
- [ ] TLS · TLS Secret · HTTPS · SSL Termination
- [ ] Default Backend · Rewrite Rules · Load Balancing
- [ ] NGINX Ingress Controller
- [ ] AWS Load Balancer Controller · Application Load Balancer · ALB
- [ ] Network Load Balancer · NLB

## 08 · NETWORKING
**Primary coverage:** [Day-07](EKS_Learning/Day-07_Kubernetes_Networking_Deep_Dive/README.md) · [Day-16](EKS_Learning/Day-16_VPC_CNI_Plugin/README.md) · [Day-17](EKS_Learning/Day-17_Network_Policies_Calico/README.md)

- [ ] CNI · Container Network Interface
- [ ] Pod Network · Cluster Network · Service Network · Network Namespace
- [ ] Pod-to-Pod Communication · Pod-to-Service Communication · Service-to-Service Communication
- [ ] DNS · CoreDNS
- [ ] NetworkPolicy · Ingress Traffic · Egress Traffic · Network Segmentation · Network Plugin
- [ ] Calico · Cilium · Flannel · AWS VPC CNI
- [ ] kube-proxy · iptables · IPVS
- [ ] Service CIDR · Pod CIDR · Cluster CIDR
- [ ] DNS Policy · DNS Config

## 09 · CONFIGMAPS AND SECRETS
**Primary coverage:** [Day-04](EKS_Learning/Day-04_Namespaces_ConfigMaps_Secrets/README.md) · [Day-06](EKS_Learning/Day-06_ConfigMaps_Secrets_Practice/README.md) · [Day-29](EKS_Learning/Day-29_Secrets_Manager_Integration/README.md)

- [ ] ConfigMap · Configuration Data
- [ ] Environment Variables · env · envFrom · ConfigMap Volume
- [ ] Secret · Opaque Secret · TLS Secret · Docker Registry Secret
- [ ] imagePullSecrets
- [ ] Secret Volume · Secret Environment Variable · Secret Mount
- [ ] Base64 Encoding · Secret Rotation
- [ ] External Secrets · External Secrets Operator · AWS Secrets Manager · AWS Systems Manager Parameter Store

## 10 · HEALTH CHECKS / PROBES
**Primary coverage:** [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md) · [Day-60](EKS_Learning/Day-60_Final_Review_Certification_Preparation/README.md)

- [ ] Liveness Probe · Readiness Probe · Startup Probe
- [ ] HTTP Probe · TCP Probe · gRPC Probe · Exec Probe
- [ ] Initial Delay Seconds · Timeout Seconds · Period Seconds · Failure Threshold · Success Threshold
- [ ] Probe Handler · Readiness Gate
- [ ] Container Health · Application Health

## 11 · RESOURCE MANAGEMENT
**Primary coverage:** [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md) · [Day-52](EKS_Learning/Day-52_Right_Sizing_Nodes/README.md) · [Module-E](EKS_Learning/modules/Module-E_HPA_VPA_KEDA.md)

- [ ] CPU Request · Memory Request · CPU Limit · Memory Limit
- [ ] Resource Requests · Resource Limits
- [ ] ResourceQuota · LimitRange (also [CKA Part1](EKS_Learning/certification/CKA_Practice_Exam_Part1.md))
- [ ] Quality of Service · QoS · Guaranteed · Burstable · BestEffort
- [ ] OOMKilled · CPU Throttling
- [ ] Memory Pressure · Disk Pressure · PID Pressure · Eviction

## 12 · SCALING
**Primary coverage:** [Day-53](EKS_Learning/Day-53_Cluster_Autoscaler_Karpenter/README.md) · [Module-E](EKS_Learning/modules/Module-E_HPA_VPA_KEDA.md)

- [ ] Horizontal Pod Autoscaler · HPA · Vertical Pod Autoscaler · VPA · Cluster Autoscaler
- [ ] Manual Scaling · CPU-Based Scaling · Memory-Based Scaling
- [ ] Custom Metrics · External Metrics · Metrics Server
- [ ] Scaling Policy · Scale Up · Scale Down
- [ ] Min Replicas · Max Replicas · Target Utilization

## 13 · STATEFULSETS
**Primary coverage:** [Day-05](EKS_Learning/Day-05_Persistent_Storage/README.md) · [Lab-02](EKS_Learning/labs/Lab-02_Storage_Stateful_App.md)

- [ ] StatefulSet · Stateful Pod · Stable Network Identity · Stable Storage · Pod Ordinal
- [ ] Headless Service · PersistentVolumeClaim
- [ ] Ordered Deployment · Ordered Termination · Parallel Pod Management
- [ ] OnDelete · RollingUpdate
- [ ] Stateful Application

## 14 · DAEMONSETS
**Primary coverage:** [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md) · [Day-41](EKS_Learning/Day-41_Prometheus_Operator/README.md)

- [ ] DaemonSet · One Pod Per Node · Node-Based Deployment
- [ ] Logging Agent · Monitoring Agent · Security Agent · CNI Agent
- [ ] Node Exporter · Fluent Bit · Filebeat

## 15 · JOBS AND CRONJOBS
**Primary coverage:** [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md) · [Day-56](EKS_Learning/Day-56_Cluster_Backup_Strategies/README.md)

- [ ] Job · CronJob · Completion · Parallelism · Completions
- [ ] Backoff Limit · Active Deadline Seconds
- [ ] Failed Job · Completed Job
- [ ] Job History · Successful Jobs History · Failed Jobs History
- [ ] Concurrency Policy · Forbid · Allow · Replace
- [ ] Suspend · Scheduled Job

## 16 · STORAGE
**Primary coverage:** [Day-05](EKS_Learning/Day-05_Persistent_Storage/README.md) · [Day-12](EKS_Learning/Day-12_EKS_Storage_EBS/README.md) · [Day-21](EKS_Learning/Day-21_EFS_CSI_Driver/README.md) · [Day-22](EKS_Learning/Day-22_FSx_Lustre/README.md) · [Day-23](EKS_Learning/Day-23_Volume_Snapshots_Restore/README.md)

- [ ] Volume · Volume Mount
- [ ] PersistentVolume · PV · PersistentVolumeClaim · PVC
- [ ] StorageClass · Dynamic Provisioning · Static Provisioning
- [ ] CSI · Container Storage Interface · CSI Driver
- [ ] Access Modes · ReadWriteOnce · ReadOnlyMany · ReadWriteMany · ReadWriteOncePod
- [ ] Reclaim Policy · Retain · Delete · Recycle
- [ ] Volume Expansion · Ephemeral Volume · Local Volume
- [ ] EmptyDir · HostPath · ConfigMap Volume · Secret Volume · Projected Volume 🔴 → [GLOSSARY](GLOSSARY.md)
- [ ] EBS CSI Driver · EFS CSI Driver

## 17 · SCHEDULING
**Primary coverage:** [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md) · [Day-15](EKS_Learning/Day-15_EKS_Security_Best_Practices/README.md) · [Day-17](EKS_Learning/Day-17_Network_Policies_Calico/README.md) · [Module-H](EKS_Learning/modules/Module-H_Advanced_Scheduling.md)

- [ ] Kubernetes Scheduler · Scheduling · Node Name · Scheduling Constraints
- [ ] Node Selector
- [ ] Node Affinity · Required Node Affinity · Preferred Node Affinity
- [ ] Pod Affinity · Pod Anti-Affinity · Required Affinity · Preferred Affinity
- [ ] Taints · Tolerations
- [ ] Topology Spread Constraints ✅ ([Module-H](EKS_Learning/modules/Module-H_Advanced_Scheduling.md))
- [ ] Priority ✅ · PriorityClass ✅ · Preemption ✅ ([Module-H](EKS_Learning/modules/Module-H_Advanced_Scheduling.md))
- [ ] Scheduler Profile 🔴 → [GLOSSARY](GLOSSARY.md)
- [ ] Node Conditions · Node Labels

## 18 · NAMESPACES AND RESOURCE ISOLATION
**Primary coverage:** [Day-04](EKS_Learning/Day-04_Namespaces_ConfigMaps_Secrets/README.md)

- [ ] Namespace · Namespace Isolation · Namespace Resource Management
- [ ] ResourceQuota · LimitRange
- [ ] Namespace-Based RBAC · Namespace-Based Deployment
- [ ] Multi-Tenancy · Environment Separation · dev · test · staging · production

## 19 · RBAC AND SECURITY
**Primary coverage:** [Day-15](EKS_Learning/Day-15_EKS_Security_Best_Practices/README.md) · [Module-A](EKS_Learning/modules/Module-A_Admission_Controllers.md) · [Module-C](EKS_Learning/modules/Module-C_Kyverno_Engine.md)

- [ ] RBAC · Role · ClusterRole · RoleBinding · ClusterRoleBinding
- [ ] ServiceAccount
- [ ] Authentication · Authorization · Least Privilege
- [ ] SecurityContext · Pod Security · Pod Security Standards
- [ ] NetworkPolicy
- [ ] Privileged Container · Non-Root Container · Read-Only Root Filesystem · Linux Capabilities
- [ ] Admission Control · Audit Logging
- [ ] API Authentication · API Authorization
- [ ] Service Account Token · Bound Service Account Token

## 20 · SECURITY CONTEXT
**Primary coverage:** [Day-15](EKS_Learning/Day-15_EKS_Security_Best_Practices/README.md) · [Module-B](EKS_Learning/modules/Module-B_RuntimeClass_gVisor.md) · [Module-J](EKS_Learning/modules/Module-J_Runtime_Security.md)

- [ ] runAsUser · runAsGroup · runAsNonRoot · fsGroup
- [ ] readOnlyRootFilesystem · allowPrivilegeEscalation
- [ ] Capabilities · Add Capabilities · Drop Capabilities
- [ ] Seccomp ✅ ([Module-J](EKS_Learning/modules/Module-J_Runtime_Security.md)) · AppArmor ✅ ([Module-J](EKS_Learning/modules/Module-J_Runtime_Security.md)) · SELinux 🔴 → [GLOSSARY](GLOSSARY.md)
- [ ] Privilege Escalation · RunAsUser · RunAsGroup

## 21 · CONTAINER IMAGES
**Primary coverage:** [Day-15](EKS_Learning/Day-15_EKS_Security_Best_Practices/README.md) · [Day-37](EKS_Learning/Day-37_GitHub_Actions_EKS/README.md) · [Day-36](EKS_Learning/Day-36_AWS_CodePipeline_Integration/README.md)

- [ ] Docker Image · Container Image
- [ ] Image Registry · Image Tag · Image Digest
- [ ] ImagePullPolicy · Always · IfNotPresent · Never
- [ ] imagePullSecrets · Private Registry · Public Registry · Container Registry
- [ ] Image Scanning · Vulnerability Scanning
- [ ] Image Signing 🔴 → [GLOSSARY](GLOSSARY.md) · Image Provenance 🔴 → [GLOSSARY](GLOSSARY.md)
- [ ] Amazon ECR · Azure Container Registry · Google Artifact Registry · Docker Hub

## 22 · HELM
**Primary coverage:** [Day-32](EKS_Learning/Day-32_ArgoCD_Installation_Config/README.md)

- [ ] Helm · Helm Chart · Chart · Chart.yaml · values.yaml · Templates · Release
- [ ] Helm Repository · Chart Repository · Helm Install · Helm Upgrade · Helm Rollback · Helm Uninstall
- [ ] Helm Dependency · Helm Lint · Helm Template · Helm Values
- [ ] Helm Hooks · Helm Test · Chart Version · App Version
- [ ] Named Templates · Helpers · Conditionals · Loops
- [ ] Secrets Management
- [ ] Helmfile 🔴 → [GLOSSARY](GLOSSARY.md)

## 23 · KUSTOMIZE
**Primary coverage:** [Day-33](EKS_Learning/Day-33_ArgoCD_Application_Patterns/README.md) · [Module-D](EKS_Learning/modules/Module-D_Argo_Rollouts.md)

- [ ] Kustomize · Base · Overlay · kustomization.yaml
- [ ] Patches · Strategic Merge Patch · JSON Patch
- [ ] ConfigMap Generator · Secret Generator
- [ ] Name Prefix · Name Suffix · Common Labels
- [ ] Environment Overlays · Dev Overlay · Staging Overlay · Production Overlay

## 24 · GITOPS
**Primary coverage:** [Day-31](EKS_Learning/Day-31_GitOps_ArgoCD_Intro/README.md) · [Day-32](EKS_Learning/Day-32_ArgoCD_Installation_Config/README.md) · [Day-35](EKS_Learning/Day-35_ArgoCD_Advanced_Best_Practices/README.md)

- [ ] GitOps · Declarative Infrastructure · Git as Source of Truth
- [ ] Continuous Delivery · Drift Detection · Reconciliation
- [ ] Argo CD · Flux
- [ ] Application · ApplicationSet · Sync · Auto Sync · Manual Sync
- [ ] Health Status · Sync Status · Desired State · Live State
- [ ] Git Repository · Git Revision · Progressive Delivery

## 25 · CI/CD WITH KUBERNETES
**Primary coverage:** [Day-36](EKS_Learning/Day-36_AWS_CodePipeline_Integration/README.md) · [Day-37](EKS_Learning/Day-37_GitHub_Actions_EKS/README.md) · [Day-38](EKS_Learning/Day-38_Jenkins_on_EKS/README.md)

- [ ] CI/CD · Continuous Integration · Continuous Delivery · Continuous Deployment
- [ ] Jenkins · GitHub Actions · GitLab CI/CD · Azure DevOps · AWS CodePipeline
- [ ] Build · Test · Docker Build · Image Push · Container Registry
- [ ] Manifest Deployment · Helm Deployment · Kustomize Deployment
- [ ] Rollout · Rollback
- [ ] Deployment Approval · Deployment Gate
- [ ] Pipeline · Artifact · Artifact Repository

## 26 · AWS EKS
**Primary coverage:** [Day-08](EKS_Learning/Day-08_Introduction_Amazon_EKS/README.md) → [Day-15](EKS_Learning/Day-15_EKS_Security_Best_Practices/README.md) · [Day-26](EKS_Learning/Day-26_OIDC_Provider_Setup/README.md) · [Day-27](EKS_Learning/Day-27_IRSA_Multiple_Services/README.md)

- [ ] Amazon EKS · EKS Cluster
- [ ] Managed Node Group · Self-Managed Node Group · EKS Fargate · EKS Auto Mode
- [ ] AWS VPC CNI
- [ ] IAM · IAM Roles for Service Accounts · IRSA · EKS Pod Identity
- [ ] AWS Load Balancer Controller · ALB · NLB
- [ ] ECR · CloudWatch · AWS Secrets Manager · Parameter Store
- [ ] EBS CSI Driver · EFS CSI Driver
- [ ] AWS IAM Authenticator
- [ ] VPC · Subnet · Security Group · Route Table · NAT Gateway · Internet Gateway
- [ ] Multi-AZ · Availability Zone

## 27 · MONITORING
**Primary coverage:** [Day-14](EKS_Learning/Day-14_EKS_Monitoring_Logging/README.md) · [Day-41](EKS_Learning/Day-41_Prometheus_Operator/README.md) · [Day-42](EKS_Learning/Day-42_Grafana_Dashboards/README.md) · [Day-45](EKS_Learning/Day-45_Monitoring_Stack_Practice/README.md)

- [ ] Kubernetes Metrics · Metrics Server
- [ ] Prometheus · Grafana · Alertmanager
- [ ] kube-state-metrics · Node Exporter
- [ ] Container Metrics · Pod Metrics · Node Metrics · CPU Metrics · Memory Metrics · Network Metrics · Disk Metrics
- [ ] Kubernetes Events · Alerts · Dashboards · Recording Rules
- [ ] Prometheus Operator · ServiceMonitor · PodMonitor

## 28 · LOGGING
**Primary coverage:** [Day-14](EKS_Learning/Day-14_EKS_Monitoring_Logging/README.md) · [Day-45](EKS_Learning/Day-45_Monitoring_Stack_Practice/README.md)

- [ ] Container Logs · Pod Logs · Centralized Logging
- [ ] Fluent Bit · Fluentd · Filebeat
- [ ] Elasticsearch · Logstash · Kibana · OpenSearch · OpenSearch Dashboards
- [ ] CloudWatch Logs · Log Aggregation · Log Rotation · Structured Logging

## 29 · OBSERVABILITY
**Primary coverage:** [Day-41](EKS_Learning/Day-41_Prometheus_Operator/README.md) · [Day-43](EKS_Learning/Day-43_AWS_XRay_Tracing/README.md) · [Day-45](EKS_Learning/Day-45_Monitoring_Stack_Practice/README.md)

- [ ] Metrics · Logs · Traces
- [ ] Distributed Tracing · OpenTelemetry
- [ ] Prometheus · Grafana · Jaeger · Tempo · Alerting
- [ ] SLI · SLO · SLA
- [ ] Golden Signals 🔴 → [GLOSSARY](GLOSSARY.md) · Latency · Traffic · Errors · Saturation
- [ ] Tracing · Correlation IDs

## 30 · TROUBLESHOOTING
**Primary coverage:** [TROUBLESHOOTING_PLAYBOOK.md](TROUBLESHOOTING_PLAYBOOK.md) · [Day-59](EKS_Learning/Day-59_Disaster_Recovery_Practice/README.md) · [Lab-04](EKS_Learning/labs/Lab-04_Troubleshooting_Game.md)

- [ ] CrashLoopBackOff · ImagePullBackOff · ErrImagePull · OOMKilled
- [ ] Pending · ContainerCreating · Terminating · Evicted · FailedScheduling
- [ ] CreateContainerConfigError · CreateContainerError · RunContainerError
- [ ] Readiness Probe Failure · Liveness Probe Failure · Startup Probe Failure
- [ ] Deployment Not Progressing · ReplicaSet Mismatch
- [ ] Incorrect Image · Incorrect Environment Variable · Configuration Failure
- [ ] Service Not Reachable · DNS Failure · Endpoint Missing · NetworkPolicy Blocking Traffic
- [ ] Ingress Failure · LoadBalancer Failure
- [ ] Port Mismatch · targetPort Mismatch
- [ ] Node NotReady · Disk Pressure · Memory Pressure · CPU Pressure · Pod Eviction

## 31 · KUBECTL COMMANDS
**Primary coverage:** [EKS_QUICK_REFERENCE.md](EKS_QUICK_REFERENCE.md) · every Day README

- [ ] kubectl · kubectl config · kubectl config get-contexts · kubectl config use-context
- [ ] kubectl get · kubectl get pods · kubectl get deployments · kubectl get replicasets · kubectl get services · kubectl get ingress · kubectl get configmaps · kubectl get secrets · kubectl get nodes · kubectl get namespaces · kubectl get events · kubectl get pv · kubectl get pvc · kubectl get storageclass
- [ ] kubectl describe · kubectl logs · kubectl logs -f · kubectl logs --previous
- [ ] kubectl exec · kubectl cp · kubectl port-forward
- [ ] kubectl apply · kubectl create · kubectl delete · kubectl edit · kubectl patch · kubectl replace
- [ ] kubectl scale · kubectl expose · kubectl set image · kubectl set env
- [ ] kubectl rollout status · kubectl rollout history · kubectl rollout undo · kubectl rollout pause · kubectl rollout resume
- [ ] kubectl label · kubectl annotate · kubectl explain · kubectl diff
- [ ] kubectl auth can-i · kubectl top
- [ ] kubectl cordon · kubectl drain · kubectl uncordon · kubectl taint
- [ ] kubectl wait · kubectl api-resources · kubectl api-versions · kubectl cluster-info

## 32 · NODE MAINTENANCE
**Primary coverage:** [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md) · [Day-52](EKS_Learning/Day-52_Right_Sizing_Nodes/README.md) · [Day-56](EKS_Learning/Day-56_Cluster_Backup_Strategies/README.md)

- [ ] Node Maintenance · Cordon · Drain · Uncordon
- [ ] Pod Eviction · Pod Disruption Budget · PDB
- [ ] Node Upgrade · Cluster Upgrade · Control Plane Upgrade · Worker Node Upgrade
- [ ] Kubernetes Version · Node Replacement · Rolling Node Upgrade

## 33 · HIGH AVAILABILITY
**Primary coverage:** [Day-56](EKS_Learning/Day-56_Cluster_Backup_Strategies/README.md) · [Day-57](EKS_Learning/Day-57_Cross_Region_Replication/README.md) · [Day-58](EKS_Learning/Day-58_Failover_Procedures/README.md)

- [ ] High Availability · HA · Multi-AZ · Fault Tolerance
- [ ] Disaster Recovery · Backup · Restore
- [ ] etcd Backup · etcd Restore
- [ ] Control Plane HA · Worker Node HA
- [ ] Pod Anti-Affinity · Topology Spread Constraints · Pod Disruption Budget
- [ ] Rolling Updates · Zero Downtime · Failover
- [ ] Recovery Time Objective · Recovery Point Objective

## 34 · ADVANCED KUBERNETES
**Primary coverage:** [Module-A](EKS_Learning/modules/Module-A_Admission_Controllers.md) · [Day-60](EKS_Learning/Day-60_Final_Review_Certification_Preparation/README.md)

- [ ] Custom Resource Definition · CRD · Custom Resource · CR
- [ ] Operator · Custom Controller · Controller Pattern · Operator Pattern
- [ ] Finalizer · Owner Reference · Garbage Collection
- [ ] API Aggregation · API Extension 🔴 → [GLOSSARY](GLOSSARY.md)
- [ ] Admission Controller · Webhook · Mutating Admission Webhook · Validating Admission Webhook · Dynamic Admission Control
- [ ] API Priority and Fairness 🔴 → [GLOSSARY](GLOSSARY.md)

## 35 · SERVICE MESH
**Primary coverage:** [Day-18](EKS_Learning/Day-18_Service_Mesh_Istio/README.md)

- [ ] Service Mesh · Istio · Linkerd · Envoy · Sidecar Proxy
- [ ] mTLS · Mutual TLS
- [ ] Traffic Management · Traffic Routing · Retries · Timeouts · Circuit Breaking
- [ ] Observability · Service-to-Service Security
- [ ] Virtual Service · Destination Rule · Gateway
- [ ] PeerAuthentication · AuthorizationPolicy
- [ ] Traffic Splitting

## 36 · DEPLOYMENT STRATEGIES
**Primary coverage:** [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md) · [Day-34](EKS_Learning/Day-34_ArgoCD_Sync_Rollback/README.md) · [Module-D](EKS_Learning/modules/Module-D_Argo_Rollouts.md)

- [ ] Rolling Update · Recreate · Blue-Green Deployment · Canary Deployment · A/B Deployment
- [ ] Progressive Delivery · Traffic Splitting · Weighted Routing
- [ ] Dark Launch 🔴 → [GLOSSARY](GLOSSARY.md) · Feature Flags 🔴 → [GLOSSARY](GLOSSARY.md)
- [ ] Argo Rollouts · Istio Traffic Routing · NGINX Traffic Routing

## 37 · NETWORK SECURITY
**Primary coverage:** [Day-17](EKS_Learning/Day-17_Network_Policies_Calico/README.md) · [Day-28](EKS_Learning/Day-28_Pod_Level_IAM_Permissions/README.md)

- [ ] NetworkPolicy · Ingress Rules · Egress Rules
- [ ] Default Deny · Allow List
- [ ] Namespace Selector · Pod Selector · IP Block
- [ ] Network Segmentation · Zero Trust Networking
- [ ] mTLS · Service Mesh Security · Security Groups for Pods

## 38 · PRODUCTION OPERATIONS
**Primary coverage:** [Day-52](EKS_Learning/Day-52_Right_Sizing_Nodes/README.md) → [Day-59](EKS_Learning/Day-59_Disaster_Recovery_Practice/README.md) · [ARCHITECTURE_PATTERNS.md](ARCHITECTURE_PATTERNS.md)

- [ ] Production Deployment · Capacity Planning · Resource Optimization · Cost Optimization
- [ ] Cluster Sizing · Node Sizing · Pod Sizing
- [ ] Autoscaling · Security Hardening · Cluster Upgrade
- [ ] Multi-Cluster · Multi-Tenancy · Environment Isolation
- [ ] Disaster Recovery · Backup and Restore · Incident Response · Runbooks
- [ ] Operational Readiness · SRE · Reliability Engineering

## 39 · KUBERNETES YAML CORE FIELDS
**Primary coverage:** [manifests/](EKS_Learning/manifests/README.md) + every Day hands-on YAML

- [ ] apiVersion · kind · metadata · name · namespace
- [ ] labels · annotations
- [ ] spec · replicas · selector · matchLabels · matchExpressions
- [ ] template · containers · container name · image · imagePullPolicy
- [ ] ports · containerPort
- [ ] env · envFrom
- [ ] resources · requests · limits
- [ ] livenessProbe · readinessProbe · startupProbe
- [ ] volumeMounts · volumes
- [ ] serviceAccountName · securityContext
- [ ] affinity · tolerations · nodeSelector · topologySpreadConstraints
- [ ] strategy
- [ ] initContainers · command · args · workingDir
- [ ] imagePullSecrets · terminationGracePeriodSeconds · terminationMessagePolicy

## 40 · IMPORTANT KUBERNETES OBJECTS
**Primary coverage:** [GLOSSARY.md](GLOSSARY.md) · spread across all 60 Days

- [ ] Pod · Deployment · ReplicaSet · StatefulSet · DaemonSet
- [ ] Job · CronJob · Service · Ingress · IngressClass
- [ ] ConfigMap · Secret · Namespace · ServiceAccount
- [ ] Role · ClusterRole · RoleBinding · ClusterRoleBinding
- [ ] NetworkPolicy · ResourceQuota · LimitRange
- [ ] PersistentVolume · PersistentVolumeClaim · StorageClass
- [ ] PodDisruptionBudget · PriorityClass
- [ ] HorizontalPodAutoscaler · VerticalPodAutoscaler
- [ ] Lease · EndpointSlice
- [ ] CustomResourceDefinition · Custom Resource

## 41 · IMPORTANT PRODUCTION TOOLS
**Primary coverage:** [Day-31]–[Day-45] · [Modules A–J](EKS_Learning/modules/README.md) · [EKS_PROJECTS.md](EKS_PROJECTS.md)

- [ ] Docker · Containerd · Helm · Kustomize
- [ ] Argo CD · Argo Rollouts · Flux
- [ ] Prometheus · Grafana · Alertmanager · OpenTelemetry
- [ ] Fluent Bit · Fluentd · Elasticsearch · Logstash · Kibana · OpenSearch
- [ ] Istio · Linkerd
- [ ] Calico · Cilium
- [ ] NGINX Ingress · AWS Load Balancer Controller
- [ ] External Secrets Operator · Cert-Manager ✅ ([Module-I](EKS_Learning/modules/Module-I_Certificates_TLS.md))
- [ ] Velero · Kyverno ✅ · OPA Gatekeeper ✅ ([Module-G](EKS_Learning/modules/Module-G_OPA_Gatekeeper.md))

## 42 · CERTIFICATES AND TLS
**Primary coverage:** [Day-13](EKS_Learning/Day-13_EKS_Load_Balancing_ALB/README.md) · [Day-15](EKS_Learning/Day-15_EKS_Security_Best_Practices/README.md) · [Day-46](EKS_Learning/Day-46_EKS_Anywhere/README.md) · [Module-I](EKS_Learning/modules/Module-I_Certificates_TLS.md)

- [ ] TLS · HTTPS · Certificate · Certificate Authority · CA
- [ ] Private Key · Certificate Signing Request · CSR
- [ ] cert-manager ✅ · Issuer ✅ · ClusterIssuer ✅ ([Module-I](EKS_Learning/modules/Module-I_Certificates_TLS.md))
- [ ] Certificate Rotation · TLS Secret
- [ ] mTLS · Let's Encrypt ✅ · ACME ✅ ([Module-I](EKS_Learning/modules/Module-I_Certificates_TLS.md))

## 43 · POLICY AND GOVERNANCE
**Primary coverage:** [Module-C](EKS_Learning/modules/Module-C_Kyverno_Engine.md) · [Module-G](EKS_Learning/modules/Module-G_OPA_Gatekeeper.md) · [Day-15](EKS_Learning/Day-15_EKS_Security_Best_Practices/README.md) · [Module-A](EKS_Learning/modules/Module-A_Admission_Controllers.md)

- [ ] Policy · Policy Engine · OPA ✅ · Open Policy Agent ✅ · Gatekeeper ✅ · Kyverno ✅
- [ ] Admission Policy · Validation · Mutation
- [ ] Policy Enforcement · Compliance
- [ ] Pod Security Admission · Pod Security Standards
- [ ] Audit · Governance

## 44 · BACKUP AND DISASTER RECOVERY
**Primary coverage:** [Day-56](EKS_Learning/Day-56_Cluster_Backup_Strategies/README.md) · [Day-57](EKS_Learning/Day-57_Cross_Region_Replication/README.md) · [Day-58](EKS_Learning/Day-58_Failover_Procedures/README.md) · [Day-59](EKS_Learning/Day-59_Disaster_Recovery_Practice/README.md)

- [ ] Velero · Cluster Backup · Resource Backup · Persistent Volume Backup
- [ ] etcd Backup · Snapshot · Restore
- [ ] Disaster Recovery · Backup Schedule · Recovery · Failover
- [ ] RTO · RPO

## 45 · MULTI-CLUSTER
**Primary coverage:** [Day-46](EKS_Learning/Day-46_EKS_Anywhere/README.md) · [Day-47](EKS_Learning/Day-47_Multi_Cluster_Networking/README.md) · [Day-48](EKS_Learning/Day-48_Cluster_Federation/README.md)

- [ ] Multi-Cluster · Cluster Federation
- [ ] Cluster API 🔴 → [GLOSSARY](GLOSSARY.md)
- [ ] Hub-and-Spoke · Cross-Cluster Networking · Multi-Region · Multi-AZ
- [ ] Cluster Provisioning · Cluster Lifecycle Management · Fleet Management

## 46 · IMPORTANT INTERVIEW CONCEPTS
**Primary coverage:** [Day-60](EKS_Learning/Day-60_Final_Review_Certification_Preparation/README.md) · [certification/](EKS_Learning/certification/README.md) · all Practice/Interview-Prep Days

- [ ] Pod vs Container · Pod vs Deployment · Deployment vs ReplicaSet
- [ ] Deployment vs StatefulSet · Deployment vs DaemonSet
- [ ] Service vs Ingress · ClusterIP vs NodePort vs LoadBalancer
- [ ] ConfigMap vs Secret
- [ ] Liveness vs Readiness vs Startup Probe
- [ ] PV vs PVC
- [ ] Role vs ClusterRole · RoleBinding vs ClusterRoleBinding
- [ ] HPA vs VPA · HPA vs Cluster Autoscaler
- [ ] Node Affinity vs Pod Affinity · Taints vs Tolerations
- [ ] Rolling vs Blue-Green vs Canary
- [ ] Helm vs Kustomize · CI/CD vs GitOps
- [ ] EKS vs Kubernetes
- [ ] Service Discovery · Self-Healing
- [ ] Desired State · Reconciliation · Declarative Deployment
- [ ] Immutable Infrastructure · Zero-Downtime Deployment

## 47 · CORE DEVOPS + KUBERNETES KEYWORDS
**Primary coverage:** the whole library (CI/CD days, EKS days, monitoring days)

- [ ] Linux · Shell Scripting · Bash
- [ ] Git · GitHub · GitLab · Jenkins
- [ ] CI/CD · Docker · Terraform · Ansible
- [ ] AWS · EKS · ECR · IAM · VPC · ALB · NLB · CloudWatch
- [ ] Helm · Kubernetes · K8s · Argo CD · Prometheus · Grafana
- [ ] GitOps · Infrastructure as Code · IaC
- [ ] Containerization · Orchestration · Microservices · Cloud Native
- [ ] DevSecOps · SRE · Observability · Automation
- [ ] Continuous Delivery · Continuous Deployment
- [ ] Blue-Green Deployment · Canary Deployment · Rolling Deployment · Zero Downtime
- [ ] High Availability · Disaster Recovery
- [ ] Security · Monitoring · Logging · Alerting · Troubleshooting

---

## 📊 How to Use

1. **Tick boxes as you master each keyword** — revisit the linked day BEFORE the box gets checked.
2. **Prioritize the interview path** (from the source file):
   Pods → Deployments → ReplicaSets → Services → Ingress → ConfigMaps/Secrets →
   Probes → Resources → HPA → Storage → Networking → RBAC → Helm → CI/CD →
   GitOps → Argo CD → EKS → Monitoring/Logging → Troubleshooting → Security →
   Advanced Kubernetes.
3. **Every 🔴 NEW term** was added to [`GLOSSARY.md`](GLOSSARY.md) so the 
   checklist is 100% covered by this library — no keyword is left unexplained.
4. **💡 Gaps closed September 2026:** four new modules were added to fill
   keyword gaps — **Module G** (OPA Gatekeeper/Rego), **Module H** (advanced
   scheduling: topology spread, priority, preemption), **Module I**
   (cert-manager & TLS/ACME), and **Module J** (SecurityContext, Seccomp,
   AppArmor). See [`modules/README.md`](EKS_Learning/modules/README.md).
5. **Revisit at milestones:** after Days 15, 30, 45, and at
   [Day-60](EKS_Learning/Day-60_Final_Review_Certification_Preparation/README.md).

---

*Keyword master checklist — generated September 2026 to mirror
`Kubernetes_K8s_Complete_DevOps_Keywords.txt` and mapped onto the 60-day library.*