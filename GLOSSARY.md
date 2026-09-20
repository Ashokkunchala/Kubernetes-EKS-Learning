# 📖 EKS & Kubernetes Glossary — A-to-Z Reference

> The entire 60-day program in one dictionary. Terms link to the day where they
> are taught in depth.

---

## 🔤 A

**Admission Controller** — Software that intercepts requests to the API server
*before* they are persisted. Mutating controllers modify the object; validating
controllers accept/reject it. *See: Module-A in `EKS_Learning/modules/`.*

**ALB (Application Load Balancer)** — AWS L7 load balancer used by
EKS + the AWS Load Balancer Controller. Routes on path/host rules to
`TargetType: ip` backends. — 👉 [Day-13](EKS_Learning/Day-13_EKS_Load_Balancing_ALB/README.md)

**API Server** — Front door and heart of the control plane. All `kubectl`
traffic, all controllers' reconciliation, all admission checks pass through it.
— 👉 [Day-01](EKS_Learning/Day-01_Kubernetes_Basics/README.md)

**API Aggregation / API Extension** — The mechanism that registers extra
API servers (custom or third-party) behind the main apiserver, so new resource
types can be served at the core API path (e.g. `metrics.k8s.io`).
— 👉 [Day-34](EKS_Learning/Day-34_ArgoCD_Sync_Rollback/README.md),
[Module-A](EKS_Learning/modules/Module-A_Admission_Controllers.md)

**API Priority and Fairness (APF)** — The apiserver's admission-time flow
control: requests are classified into priority levels (with per-priority
concurrency limits) so an overloaded "list all pods" fan-out can't starve
latency-sensitive writes. — 👉 [Day-16](EKS_Learning/Day-16_VPC_CNI_Plugin/README.md)

**AppArmor** — Mandatory Access Control (MAC) for Linux: per-process profiles
that restrict file/network/`cap` access. On Kubernetes it was historically
applied via annotations; Seccomp is now the more portable default and both are
enforced as runtime hardening. — 👉 [Day-15](EKS_Learning/Day-15_EKS_Security_Best_Practices/README.md)

**Argo CD** — Declarative GitOps tool that continuously syncs your cluster to
the desired state held in a Git repository. — 👉 [Day-31](EKS_Learning/Day-31_GitOps_ArgoCD_Intro/README.md)

**AppArmor** — Mandatory Access Control (MAC) that confines a process to an
allowed set of file/network operations via a node-loaded **profile**; applied
per-pod. Complements Seccomp (syscalls) with path/operation restrictions.
⚠ On EKS, Amazon Linux 2 / Bottlerocket nodes don't enforce AppArmor the way
Ubuntu does — verify on your AMI and lean on Seccomp + capabilities for the
portable baseline. — 👉 [Module-J](EKS_Learning/modules/Module-J_Runtime_Security.md)

**Autoscaler (Cluster Autoscaler)** — Adds/removes worker *nodes* based on
pending unschedulable pods. Pairs with the HPA (pod scaling). — 👉 [Day-53](EKS_Learning/Day-53_Cluster_Autoscaler_Karpenter/README.md)

## 🔤 B

**Bottlerocket** — AWS's minimal, purpose-built Linux for running containers.
Recommended AMI for EKS managed node groups. — 👉 [Day-15](EKS_Learning/Day-15_EKS_Security_Best_Practices/README.md)

**Blue/Green Deployment** — Run old (blue) and new (green) versions side by
side, cut traffic over, and delete the old. — 👉 [Day-34](EKS_Learning/Day-34_ArgoCD_Sync_Rollback/README.md)

## 🔤 C

**Calico** — CNI plugin + policy engine. Provides NetworkPolicies on EKS
(Amazon VPC CNI alone doesn't enforce them) and GlobalNetworkPolicies.
— 👉 [Day-17](EKS_Learning/Day-17_Network_Policies_Calico/README.md)

**Cluster API (CAPI)** — The Kubernetes-native way to provision/manage
clusters (declarative `Cluster`/`Machine`/`MachinePool` resources with
provider implementations: AWS, vSphere, Azure, Bare Metal). Powers EKS
Anywhere's managed-cluster story. — 👉 [Day-46](EKS_Learning/Day-46_EKS_Anywhere/README.md)

**CKA / CKAD** — Certified Kubernetes Administrator / Application Developer.
Performance-based (no multiple choice) exams. — 👉 [`certification/`](EKS_Learning/certification/README.md)

**CNI (Container Network Interface)** — The spec EKS uses to give every pod a
real VPC IP via the VPC CNI plugin. — 👉 [Day-16](EKS_Learning/Day-16_VPC_CNI_Plugin/README.md)

**ConfigMap** — Non-confidential key/value configuration injected into pods as
env vars, files, or mounted volumes. — 👉 [Day-04](EKS_Learning/Day-04_Namespaces_ConfigMaps_Secrets/README.md)

**Container Insights** — CloudWatch agent that ships metrics & logs from EKS
nodes/pods/services to CloudWatch. — 👉 [Day-14](EKS_Learning/Day-14_EKS_Monitoring_Logging/README.md)

**Control Plane** — API server + etcd + scheduler + controller managers. Fully
managed (EKS) and paid per cluster/hour. — 👉 [Day-08](EKS_Learning/Day-08_Introduction_Amazon_EKS/README.md)

## 🔤 D

**DaemonSet** — Runs exactly one pod on every node (telemetry agents, CNI pods,
log shippers). — 👉 [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md)

**Deployment** — Declares desired replicas and manages rolling updates +
rollbacks through a ReplicaSet. — 👉 [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md)

**DestinationRule** — Istio resource defining traffic-policy/load-balancing
rules and subsets for a service. — 👉 [Day-18](EKS_Learning/Day-18_Service_Mesh_Istio/README.md)

**Dark Launch** — Shipping new code to production *without* exposing it to real
users: deploy the version, let synthetic/beta traffic exercise it, and control
release via feature flags rather than a deploy. — 👉 [Module-D](EKS_Learning/modules/Module-D_Argo_Rollouts.md)

**DR (Disaster Recovery)** — Recover from region/AZ failure with RTO/RPO
targets: Velero backups, ECR/S3 replication, RDS cross-region, failover runbooks.
— 👉 [Day-56](EKS_Learning/Day-56_Cluster_Backup_Strategies/README.md)

## 🔤 E

**EBS (Elastic Block Store)** — Block storage for a single EC2 *instance*.
On EKS: `ebs.csi.aws.com`, `ReadWriteOnce` only. — 👉 [Day-12](EKS_Learning/Day-12_EKS_Storage_EBS/README.md)

**EFS (Elastic File System)** — Shared NFS storage, `ReadWriteMany`, ideal for
shared content; mounted via `efs.csi.aws.com` CSI driver. — 👉 [Day-21](EKS_Learning/Day-21_EFS_CSI_Driver/README.md)

**eksctl** — AWS's official CLI that creates EKS clusters from declarative
ClusterConfig YAML (or one-liner). — 👉 [Day-09](EKS_Learning/Day-09_EKS_Cluster_Setup_eksctl/README.md)

**EKS (Amazon Elastic Kubernetes Service)** — Managed Kubernetes control plane
+ managed/self-managed worker nodes running in your VPC. — 👉 [Day-08](EKS_Learning/Day-08_Introduction_Amazon_EKS/README.md)

**etcd** — Distributed key-value store holding all cluster state. Back it up
(e.g. Velero-on-EKS, or `etcdctl snapshot`) for DR. — 👉 [Day-56](EKS_Learning/Day-56_Cluster_Backup_Strategies/README.md)

## 🔤 F

**Fargate** — Serverless compute for EKS: no nodes to manage, pods run in a

**Feature Flags** — Runtime booleans/configuration that toggle code paths in a
deployed binary (often paired with dark launches) so behavior can change
without a redeploy. Tools: LaunchDarkly, Flagsmith, Argo Rollouts' `-setCanary`.
— 👉 [Module-D](EKS_Learning/modules/Module-D_Argo_Rollouts.md)

**Finalizer** — A string on an object that delays deletion: the apiserver
deletes the object only after every listed finalizer is removed (each component
cleans up its external resources and removes its finalizer — e.g. PVs must be
detached/unmounted first). — 👉 [Day-05](EKS_Learning/Day-05_Persistent_Storage/README.md)
sandbox with isolated VPC networking; billing per pod. — 👉 [Day-08](EKS_Learning/Day-08_Introduction_Amazon_EKS/README.md)

**FSx for Lustre** — High-throughput shared FS for HPC/ML workloads on EKS.
— 👉 [Day-22](EKS_Learning/Day-22_FSx_Lustre/README.md)

## 🔤 G

**Gateway API** — The modern successor to Ingress: `Gateway`, `GatewayClass`,
`HTTPRoute` resources. — 👉 [Day-16](EKS_Learning/Day-16_VPC_CNI_Plugin/README.md)

**GitOps** — Git is the single source of truth; a controller (Argo CD) +
webhooks/PRs drive all cluster changes. — 👉 [Day-31](EKS_Learning/Day-31_GitOps_ArgoCD_Intro/README.md)

**Garbage Collection (GC)** — Kubernetes' background cleanup of "orphaned"
objects. An object with an Owner Reference is deleted automatically when its
owner is deleted; objects with `ownerReferences` missing/`controller:false`
become eligible for GC. — 👉 [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md)

**Golden Signals** — The four SRE service-health metrics: **Latency** (time to
serve a request), **Traffic** (demand volume — RPS/QPS), **Errors** (failed
request rate, ideally per-request type), **Saturation** (utilization vs
capacity — how full the service is). — 👉 [Day-45](EKS_Learning/Day-45_Monitoring_Stack_Practice/README.md)

**Grafana** — Visualization/alerting UI on top of Prometheus metrics, CloudWatch,
or Loki. — 👉 [Day-42](EKS_Learning/Day-42_Grafana_Dashboards/README.md)

## 🔤 H

**Helm** — Package manager for Kubernetes: chart = templated YAML + repo.
Releases are tracked in the cluster. — 👉 [Day-32](EKS_Learning/Day-32_ArgoCD_Installation_Config/README.md)

**Helmfile** — Declarative wrapper over Helm: one `helmfile.yaml` pins
repositories, environments, values files, and the exact `helm upgrade
--install` order for many releases, making chart state reproducible across
dev/staging/prod. — 👉 [Day-32](EKS_Learning/Day-32_ArgoCD_Installation_Config/README.md)

**HPA (HorizontalPodAutoscaler)** — Scales *replicas* of a workload based on
metrics (CPU, memory, custom). — 👉 [Day-53](EKS_Learning/Day-53_Cluster_Autoscaler_Karpenter/README.md)

## 🔤 I

**IAM Roles for Service Accounts (IRSA)** — Uses an OIDC provider so pods can
assume specific IAM roles via projected service-account tokens. — 👉 [Day-26](EKS_Learning/Day-26_OIDC_Provider_Setup/README.md),
[Day-27](EKS_Learning/Day-27_IRSA_Multiple_Services/README.md)

**Ingress** — Cluster-level routing rules (host/path) in front of Services;
on EKS typically backed by ALB. — 👉 [Day-03](EKS_Learning/Day-03_Services_Networking/README.md)

**Image Provenance / Image Signing** — Provenance = who/what built an image and
the chain of custody behind it; Signing = cryptographic attestation (Sigstore /
Cosign / Notary) that an image digest came from a trusted builder, so admission
webhooks can reject unsigned or tampered images. — 👉 [Day-15](EKS_Learning/Day-15_EKS_Security_Best_Practices/README.md)

**Istio** — Service mesh: sidecar Envoy proxies give mTLS, traffic shifting,
and observability. — 👉 [Day-18](EKS_Learning/Day-18_Service_Mesh_Istio/README.md)

## 🔤 K

**Karpenter** — Node *provisioner* that launches the right-sized EC2 instance
for pending pods in seconds, replacing the Cluster Autoscaler in most setups.
— 👉 [Day-53](EKS_Learning/Day-53_Cluster_Autoscaler_Karpenter/README.md)

**Kubelet** — Node agent that runs pods and reports status to the API server.
— 👉 [Day-01](EKS_Learning/Day-01_Kubernetes_Basics/README.md)

**Kubernetes (K8s)** — Open-source container orchestration: deployment,
scaling, self-healing, service discovery. — 👉 [Day-01](EKS_Learning/Day-01_Kubernetes_Basics/README.md)

## 🔤 N

**Namespace** — Logical partition of a cluster for isolation/quotas/RBAC scope.
— 👉 [Day-04](EKS_Learning/Day-04_Namespaces_ConfigMaps_Secrets/README.md)

**NetworkPolicy** — Firewall rules between pods (and to/from the outside) using
selectors; VPC CNI doesn't enforce them — use Calico or another policy engine.
— 👉 [Day-17](EKS_Learning/Day-17_Network_Policies_Calico/README.md)

## 🔤 O

**OIDC Provider** — EKS issues an OIDC discovery endpoint; you register it with
IAM so ServiceAccounts can get web identity tokens → IRSA. — 👉 [Day-26](EKS_Learning/Day-26_OIDC_Provider_Setup/README.md)

**OPA Gatekeeper** — The Kubernetes policy engine built on the Open Policy
Agent. Evaluates **Rego** at admission via a validating webhook using a 3-part
model: `ConstraintTemplate` (schema + Rego), `Constraint` (concrete policy with
parameters), and `AdmissionReview` (the request JSON). Includes background audit
for existing resources. The flexible, data-driven alternative to Kyverno's
YAML-native policies. — 👉 [Module-G](EKS_Learning/modules/Module-G_OPA_Gatekeeper.md)

## 🔤 P

**Pod** — Smallest deployable unit: one or more containers sharing netns +
volumes. — 👉 [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md)

**PersistentVolume / PVC / StorageClass** — PV = storage resource; PVC = request;
StorageClass = named provisioner/config. The storage "trinity." — 👉 [Day-05](EKS_Learning/Day-05_Persistent_Storage/README.md)

**Projected Volume** — A Volume that mounts *several* sources into one
directory at once (e.g. a Secret + a ConfigMap + ServiceAccount token), commonly
used by controllers/cloud-provider agents that need multiple data blobs. — 👉 [Day-04](EKS_Learning/Day-04_Namespaces_ConfigMaps_Secrets/README.md)

**Pod Disruption Budget (PDB)** — Guarantees a minimum number of replicas stay
up during *voluntary* disruptions (node drains, upgrades). — 👉 [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md)

**Preemption** — In scheduling, a higher-priority pod (via `PriorityClass`)
evicts lower-priority pods to free node resources; the evicted pods are the
"preemption victims." Disable via `preemptionPolicy: Never`. — 👉 [Module-H](EKS_Learning/modules/Module-H_Advanced_Scheduling.md)

**PriorityClass** — Cluster-scoped object mapping a name to a numeric priority
and whether it's the `globalDefault`. Higher priority `value` = scheduled first
and able to preempt lower-priority pods. — 👉 [Module-H](EKS_Learning/modules/Module-H_Advanced_Scheduling.md)

**Prometheus** — Open-source metrics collection + TSDB; the standard for K8s
monitoring. — 👉 [Day-41](EKS_Learning/Day-41_Prometheus_Operator/README.md)

## 🔤 R

**RBAC** — Role/RoleBinding (namespaced) + ClusterRole/ClusterRoleBinding
(cluster-wide) control who can do what. — 👉 [Day-15](EKS_Learning/Day-15_EKS_Security_Best_Practices/README.md)

**ReplicaSet** — Maintains N identical pods; nearly always driven by a
Deployment. — 👉 [Day-02](EKS_Learning/Day-02_Pods_ReplicaSets_Deployments/README.md)

**Rego** — The declarative policy language used by Open Policy Agent and
Gatekeeper. Policies express "violation if ..." rules evaluated against the
admission request's JSON (`input`), with access to cluster data via `data.*`.
— 👉 [Module-G](EKS_Learning/modules/Module-G_OPA_Gatekeeper.md)

## 🔤 S

**Secrets** — Base64-encoded sensitive values; prefer EBS-backed files,
KMS + Secrets Manager / External Secrets for real production secret handling.
— 👉 [Day-04](EKS_Learning/Day-04_Namespaces_ConfigMaps_Secrets/README.md),
[Day-29](EKS_Learning/Day-29_Secrets_Manager_Integration/README.md)

**SELinux** — Mandatory Access Control built into the Linux kernel (default on
RHEL-family distros) applying security labels to processes/files. On EKS,
Amazon Linux 2 & Bottlerocket have varying enforcement; Seccomp + capabilities
are the portable cross-AMI baseline. — 👉 [Module-J](EKS_Learning/modules/Module-J_Runtime_Security.md)

**Seccomp** — Filters the **system calls** a process can make at the kernel
boundary. `RuntimeDefault` uses the runtime's safe default; `Localhost` mounts a
custom profile; `Unconfined` (the unset default) must be avoided in prod. The
`restricted` Pod Security Standard requires `RuntimeDefault`. — 👉 [Module-J](EKS_Learning/modules/Module-J_Runtime_Security.md)

**Scheduler Profile** — A named set of scheduler plugins (queue-sort, node-score,
bind) configured via `KubeSchedulerConfiguration` to change how pods are
prioritized/bound. On EKS the scheduler is managed by AWS; profiles matter most
on self-managed control planes and are tuned to combine plugins like
`NodeAffinity`, `TaintToleration`, and `PodTopologySpread`. — 👉 [Module-H](EKS_Learning/modules/Module-H_Advanced_Scheduling.md)

**Service** — Stable virtual IP/DNS in front of a set of pods (ClusterIP,
NodePort, LoadBalancer). — 👉 [Day-03](EKS_Learning/Day-03_Services_Networking/README.md)

**Service Mesh** — Dedicated infrastructure layer for L4/L7 traffic: mTLS,
retries, circuit breaking, observability (Istio). — 👉 [Day-18](EKS_Learning/Day-18_Service_Mesh_Istio/README.md)

**Spot Instances** — Up-to-90%-discount, reclaimable EC2 capacity ideal for
fault-tolerant/batch workloads. — 👉 [Day-51](EKS_Learning/Day-51_Spot_Instances_EKS/README.md)

**StatefulSet** — Stable identity (name + PVC) per replica for stateful apps
like databases. — 👉 [Day-05](EKS_Learning/Day-05_Persistent_Storage/README.md)

## 🔤 T

**Topology Spread Constraints** — Modern way to distribute pods evenly across
topology domains (e.g. AZs). You give a `topologyKey`, a `maxSkew`, and a
`whenUnsatisfiable` (`DoNotSchedule` to block, `ScheduleAnyway` to best-effort).
Replaces hand-rolled anti-affinity for balanced HA placement. — 👉 [Module-H](EKS_Learning/modules/Module-H_Advanced_Scheduling.md)

## 🔤 V

**VPC CNI (ipvlan/native)** — Gives each pod a VPC ENI IP → no NAT, native
security-group controls; watch ENI/IP exhaustion in large clusters.
— 👉 [Day-16](EKS_Learning/Day-16_VPC_CNI_Plugin/README.md)

**Velero** — Backup/restore of cluster resources + volumes to S3, cross-region.
— 👉 [Day-56](EKS_Learning/Day-56_Cluster_Backup_Strategies/README.md)

**VPA (VerticalPodAutoscaler)** — Recommends/adjusts CPU+memory *requests*
per pod; often used to right-size. — 👉 [Day-52](EKS_Learning/Day-52_Right_Sizing_Nodes/README.md)

## 🔤 X

**X-Ray** — AWS distributed tracing; on EKS, X-Ray SDK + Daemon/service in
cluster. — 👉 [Day-43](EKS_Learning/Day-43_AWS_XRay_Tracing/README.md)

---

## 🧠 Also See

- [`EKS_QUICK_REFERENCE.md`](EKS_QUICK_REFERENCE.md) — commands & troubleshooting cheat sheet
- [`ARCHITECTURE_PATTERNS.md`](ARCHITECTURE_PATTERNS.md) — design patterns for EKS
- [`TROUBLESHOOTING_PLAYBOOK.md`](TROUBLESHOOTING_PLAYBOOK.md) — real incident runbooks
- [`modules/`](EKS_Learning/modules/README.md) — 10 deep-dive modules (incl. OPA Gatekeeper, Advanced Scheduling, Certificates & TLS, Runtime Security)

*Glossary last updated: September 2026*