# 🃏 EKS & Kubernetes Flashcards — 60 Cards in 10 Groups

Daily spaced-repetition deck. Recite each group aloud once a day; shuffle the order
between days. `**Q:**` then `**A:**` on the following line.

---

## 📦 Core Kubernetes Objects

**Q: What is the smallest deployable unit in Kubernetes?**
**A: The Pod — one or more containers sharing IP/volumes that are always co-scheduled.**

**Q: Which object guarantees a desired number of identical pods and self-heals failures?**
**A: Deployment (apps/v1) — reconciles via a ReplicaSet and provides rolling updates.**

**Q: What does a ReplicaSet do, and why shouldn't you manage it directly?**
**A: It maintains a stable set of pod replicas; manage it through a Deployment to get rollout/history features.**

**Q: Which workload keeps pod identity/stable network names and per-pod PVCs?**
**A: StatefulSet — ordered pods (name-0, name-1) that keep sticky storage and DNS.**

**Q: Which workload runs exactly one pod on every (matching) node?**
**A: DaemonSet — for node-level agents like log shippers, CNI, kube-proxy.**

**Q: What's the difference between a Namespace and a Node?**
**A: A Namespace is a logical cluster partition for scoping objects/quota; a Node is a physical/virtual machine running pods.**

---

## 🗓️ Scheduling & Workload Placement

**Q: How does nodeSelector pin a pod to a node?**
**A: It's a hard matchLabels constraint on node labels; the scheduler only places pods on matching nodes.**

**Q: What do taints and tolerations control?**
**A: Taints repel pods from a node; only pods with a matching toleration can (NoSchedule) or will (PreferNoSchedule) schedule there.**

**Q: What's nodeAffinity used for beyond simple key=value matching?**
**A: Expressive node selection — In/NotIn/Exists/Gt/Lt, with required/preferred (soft weighting) modes.**

**Q: How does podAffinity vs podAntiAffinity differ?**
**A: affinity pulls pods together near matched pods; anti-affinity spreads them apart (commonly per topologyKey=hostname).**

**Q: What are the three QoS classes and how are they derived?**
**A: Guaranteed (reqs==limits), Burstable (reqs≤limits, not equal), BestEffort (no reqs/limits) — from container resources.**

**Q: How does resource requests vs limits affect scheduling vs runtime?**
**A: Requests are reservation for scheduling; limits cap/max throttling (or eviction) of the container.**

---

## 💾 Storage

**Q: What is the relationship between PV and PVC?**
**A: A PV is cluster storage; a PVC is a request binding to a suitable PV; the pod consumes the PVC by mount.**

**Q: What does a StorageClass provide?**
**A: Dynamic provisioning (provisioner), reclaim policy, volume modes and binding mode (Immediate vs WaitForFirstConsumer).**

**Q: What does volumeBindingMode: WaitForFirstConsumer do?**
**A: Defer PV binding until a pod uses the PVC — lets the provisioner pick the AZ/node where the pod lands.**

**Q: Which is the cross-pod/AZ shared filesystem storage on EKS?**
**A: Amazon EFS (NFS). EBS is single-AZ, single-pod. FSx for Lustre is for high-performance HPC workloads.**

**Q: Which is the block storage on EKS for one pod in one AZ?**
**A: Amazon EBS via the EBS CSI driver (gp2/gp3 default) — dynamic, snapshottable, but AZ-bound.**

**Q: What restores a volume from an earlier point in time in CSI EKS?**
**A: VolumeSnapshot → VolumeSnapshotContent → restore a new volume; EBS CSI and EFS CSI both support them.**

---

## 🌐 Networking & Services

**Q: What's the default Service type and its scope?**
**A: ClusterIP — internally reachable only; ClusterIP:None is a headless service with no load-balancing.**

**Q: When would you pick NodePort over LoadBalancer?**
**A: NodePort opens a 30000-32767 host port (manual/small clusters); LoadBalancer wraps it with a cloud LB (ALB/NLB on EKS).**

**Q: What three discovery mechanisms make Service DNS, and which add-on serves it?**
**A: A records (clusterIP, headless sets) + the SRV record for named ports, served by CoreDNS add-on.**

**Q: What does an Ingress do beyond a Service?**
**A: L7 routing — host + path rules route to backend Services; apiVersion networking.k8s.io/v1, pathType Prefix/Exact.**

**Q: What is a NetworkPolicy and how is enforcement done on EKS?**
**A: Pod-level L3/L4 allowlist policy (deny-by-default ingress/egress); enforced via a CNI policy engine (VPC CNI tier or Calico).**

**Q: How does kube-proxy forward traffic within a Service?**
**A: It programs iptables (or IPVS) DNAT rules on every node to load-balance to Ready pod IPs.**

---

## 🔐 Security, RBAC & Identity

**Q: What's the difference between Role and ClusterRole?**
**A: Role is namespace-scoped; ClusterRole is cluster-wide (used for non-namespaced resources or as a template for namespaces).**

**Q: What do RoleBinding and ClusterRoleBinding link?**
**A: They bind a Role/ClusterRole to subjects (users, groups, ServiceAccounts); ClusterRoleBinding grants cluster-wide.**

**Q: How can you test what a user can do before they do it?**
**A: kubectl auth can-i list pods -n app --as alice.**

**Q: What is a Secret and is base64 encoding encryption?**
**A: A small sensitive-data object; base64 is only transport encoding — enable KMS envelope encryption of etcd for real at-rest security.**

**Q: How does securityContext harden containers?**
**A: Sets runAsUser/runAsNonRoot, capabilities, readOnlyRootFilesystem, seccompProfile, and enables Pod Security Standards.**

**Q: What makes a ServiceAccount or Pod assume an AWS IAM role with IRSA?**
**A: An OIDC trust + a serviceAccount annotation (eks.amazonaws.com/role-arn); the projected JWT + AssumeRoleWithWebIdentity gets short-lived creds.**

---

## 🅰️ EKS-Specific Platform

**Q: Who manages the EKS control plane (apiserver, etcd, scheduler, controllers)?**
**A: AWS — it's a managed, highly available service (control plane is $0.10/hr); you manage the data plane (nodes) and workloads.**

**Q: How does the VPC CNI assign pod IPs, and what's the exhaustion fix?**
**A: Pods get VPC subnet IPs via ENIs; prefix delegation (a /28 per ENI) solves IPv4 exhaustion and raises pod density.**

**Q: What is custom networking in VPC CNI?**
**A: Running pods on a dedicated secondary CIDR/subnet so pod IPs don't burn the primary subnet space.**

**Q: What is Security Groups for Pods?**
**A: Branch ENIs (vpc.amazonaws.com/pod-eni) give each pod its own security groups for per-pod network identity.**

**Q: What are EKS managed add-ons?**
**A: AWS-maintained installs/upgrades of vpc-cni, kube-proxy, CoreDNS, EBS/EFS CSI, aws-ebs-csi-driver, load-balancer-controller — version-managed from the console/API.**

**Q: What do managed node groups guarantee that self-managed don't?**
**A: AWS orchestrates node lifecycle (launching, updates, scaling, replacement on failures) — you give up fine-grained node custom bootstrapping.**

---

## 📈 Autoscaling & Cost

**Q: What does the Horizontal Pod Autoscaler (HPA) scale?**
**A: Pod replicas based on observed metrics (CPU/memory, or custom via autoscaling/v2), given min/max + target values.**

**Q: What does the Vertical Pod Autoscaler (VPA) do?**
**A: Adjusts container *requests/limits* over time (based on utilization history), optionally evicting pods to resize (recommender/mutating APIVPA).**

**Q: How does Cluster Autoscaler decide to scale nodes?**
**A: It watches pending unschedulable pods and only changes the desired count of existing node groups within min/max bounds.**

**Q: How does Karpenter model provisioning?**
**A: NodePool policy → it launches the right node from any today's instance type (Spot/OD mix) per pod demand, with consolidation; no per-group min/max.**

**Q: What is a PodDisruptionBudget and when does it matter?**
**A: Ensures minAvailable/maxUnavailable of replicas survive voluntary disruptions — e.g., drains, node upgrades, spot reclamations.**

**Q: What's the recommended practice for stateless autoscaling on EKS?**
**A: HPA for pods + Karpenter (or CA) for nodes + Spot for interruptible tiers + a PDB around critical replicas.**

---

## 🧭 GitOps (ArgoCD)

**Q: What is the core promise of GitOps?**
**A: Git is the single source of truth; a controller continuously reconciles cluster state to the desired state in Git.**

**Q: What ArgoCD resource syncs an app/repo path into a namespace?**
**A: An Application — source (repoURL/path/targetRevision) + destination (cluster server/namespace) + syncPolicy.**

**Q: What do prune and selfHeal each fix?**
**A: prune deletes resources removed from Git; selfHeal reverts out-of-band drift back to Git automatically.**

**Q: What are ArgoCD sync strategies?**
**A: automated (on every check) vs manual (button/CLI), with sync waves/orphaned-resource options for ordering.**

**Q: How does an ApplicationSet scale GitOps to many clusters?**
**A: Generators (list/git/cluster/matrix) template out one Application per cluster/env from the same base manifests.**

**Q: GitOps: Helm vs Kustomize — when each?**
**A: Kustomize (native overlays, no server) for forks/patches by env; Helm (charts, templated, shareable) for complex decoupled installs — both usable by ArgoCD.**

---

## 📊 Observability

**Q: What is Prometheus's core data model?**
**A: Time-series metrics with labels; scraped from /metrics endpoints, queried with PromQL (rate, histogram_quantile, etc.).**

**Q: How does the Prometheus Operator manage targets?**
**A: ServiceMonitor/PodMonitor CRDs declare scrape configs declaratively; prometheus-operator reconciles them into Prometheus.**

**Q: What is a recording rule and an alerting rule? (both via PrometheusRule)**
**A: Pre-computed reused queries (record: expr) and threshold conditions (alert: expr for:) that route to Alertmanager.**

**Q: What three pillars does Grafana tie together for EKS?**
**A: Metrics (Prometheus/CloudWatch), Logs (CloudWatch Logs/Loki/Fluent Bit), Traces (X-Ray/ADOT/Tempo) — correlated by trace/dataset IDs.**

**Q: What does CloudWatch Container Insights add on EKS?**
**A: Out-of-the-box pod/node/container metrics + logs (and control-plane diagnostics) without running your own Prometheus yet.**

**Q: What is AWS X-Ray + a sampling strategy used for?**
**A: Distributed tracing across microservices (service maps, latency); ADOT collector pushes spans; sampling keeps cost sane.**

---

## 🛟 Disaster Recovery

**Q: What is Velero and what does it back up?**
**A: Open-source K8s backup/restore — backs up cluster objects + optionally volumes (incl. EBS snapshots) to S3/blobs.**

**Q: RPO vs RTO — one-line definitions?**
**A: RPO = max acceptable data loss; RTO = max acceptable downtime to restore service.**

**Q: What's the EKS-documented DR recipe in one sentence?**
**A: Control plane (+ etcd) is AWS-managed — restore workloads with GitOps (redeploy), app data with snapshots/EFS/EBS volume snapshots + cross-region replication.**

**Q: How do you make app data available across regions for DR?**
**A: EBS volume snapshots replicated (or CRR S3/EFS cross-region replication, or RDS cross-region read replicas) into the DR region.**

**Q: What is the recommended restore test cadence?**
**A: Scheduled "game days" — restore the full stack into a sandbox cluster at least quarterly and measure real RTO/RPO.**

**Q: Where do your configs, images, and state live so DR isn't a panic?**
**A: Configs in Git (GitOps redeploy), images in ECR (with cross-region replication), state in snapshotted volumes / replicated data stores.**

---

**Rule:** 10 groups × 6 cards. Aim to complete every group once/day — 15 minutes total. 🚀