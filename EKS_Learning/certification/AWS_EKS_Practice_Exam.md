# 🧪 AWS EKS Practice Exam — 30 Scenario Questions

EKS-specific multiple-choice for **AWS Certified DevOps Engineer – Professional** and
**Solutions Architect** candidates. Mock-exam mode: 30 questions, 45 minutes, no notes.
Each question notes which 60-day day maps to the topic. Answer key with rationale at the
bottom.

---

## 📝 Questions

**Q01.** Company: EKS cluster with only a managed node group in `us-east-1a` and `us-east-1b`.
You create a Pod that needs an EBS volume. Where is the best place to confirm high
availability for that Stateful workload?
- A. Place the StatefulSet pod in either AZ; EBS volumes are regionally replicated
- B. Use a multi-AZ node group and let the scheduler pick; EBS is AZ-bound so pods and volumes must share the AZ
- C. Always use EFS instead of EBS so the PV is available across AZs
- D. Deploy the pod to `us-east-1c`; it has the least load

`*Topic: EKS storage/AZ affinity — Day 12 (EBS), Day 21 (EFS)*`

**Q02.** You scaled to 60 pods per node but the kubelet reports "failed to assign an IP to
pod" and the VPC CNI log shows the pool is exhausted. Which **first** fix matches modern
EKS best practice?
- A. Increase the VPC CIDR and re-register the subnets
- B. Enable prefix delegation on the VPC CNI (`ENABLE_PREFIX_DELEGATION: true`)
- C. Switch to AWS EKS Fargate for all pods
- D. Add a second kube-proxy

`*Topic: VPC CNI / IPv4 exhaustion — Day 16*`

**Q03.** A sysadmin adds a new user's IAM Role ARN to the `aws-auth` ConfigMap so they can run
`kubectl get nodes`. What is the valid reason this mapping exists?
- A. It maps IAM principals to Kubernetes RBAC groups
- B. It creates IAM roles inside the cluster control plane
- C. It replaces the cluster's KMS encryption key
- D. It defines which subnet the API server binds to

`*Topic: aws-auth / RBAC — Day 11*`

**Q04.** IRSA (IAM Roles for Service Accounts) is configured. Statement: "the Pod assumes the
IAM role automatically via the service account token." Is this accurate?
- A. Yes — the projected OIDC token validates the STS `AssumeRoleWithWebIdentity` call
- B. No — IRSA requires an instance profile on the worker node
- C. No — the token is only usable by `ClusterRole` bindings
- D. Yes, but only if the cluster runs kube-proxy in IPVS mode

`*Topic: IRSA / OIDC — Day 26-27*`

**Q05.** You want pods to inherit least-privilege AWS permissions with **zero node-level
permissions** for the app, and the cluster is on the newest EKS minor version. Compared to
IRSA, what is the difference EKS Pod Identity brings?
- A. Pod Identity removes the need for RBAC; IRSA needs Roles/ClusterRoles
- B. Pod Identity lets you attach IAM roles directly to workload identities without managing a per-service-account trust policy and can be scoped via conditions
- C. Pod Identity only works on self-managed node groups
- D. IRSA is deprecated and cannot be used on EKS ≥1.27

`*Topic: EKS Pod Identity vs IRSA — Day 27 (IRSA) + current best practices*`

**Q06.** A multi-AZ ALB Ingress forwards `/api→backend-svc:8080`. A user reports the Ingress
IP is unreachable. Which is the most likely root cause when the ALB controller shows it
cannot find subnets?
- A. The ALB IngressClass annotation is wrong
- B. The subnets lack tags `kubernetes.io/role/elb` (public) or `kubernetes.io/role/internal-elb` (private)
- C. The Service's `targetPort` is a string instead of a number
- D. The NodePort range was changed

`*Topic: ALB controller subnet discovery — Day 13*`

**Q07.** A stateful app on EKS needs file-share semantics (multiple pods read/write the same
files). Which storage integration is the intended fit?
- A. EBS CSI with `volumeMode: Filesystem`
- B. EFS CSI driver (or FSx for Lustre for HPC) accessible by multiple pods
- C. EBS gp3 with `MultipleAttachEnabled: true`
- D. Local NVMe instance stores mounted as PVs

`*Topic: EFS vs EBS — Day 21, Day 05*`

**Q08.** Your cluster spikes during a batch and you want node provisioning that (a) launches
nodes *only when pods need them*, (b) can burst into Spot, and (c) packs multiple instance
types. AWS recommends which tool?
- A. Cluster Autoscaler pointing at the Managed Node Group
- B. Karpenter with a NodePool (NodeClass + instance-type mixing + Spot)
- C. Manual ASG replica scaling
- D. EKS Fargate profile with `scalableTarget`

`*Topic: Cluster Autoscaler vs Karpenter — Day 53*`

**Q09.** You drain a node before a spot interruption. What is the most reliable mechanism an
EKS workload should also use so evictions are graceful and coordinated?
- A. `kubectl delete node` first, then drain
- B. A PodDisruptionBudget (PDB) plus cordon/drain by the node lifecycle handler
- C. Dropping the node from the security group
- D. Increasing `terminationGracePeriodSeconds` to `0`

`*Topic: Spot interruption + PDB — Day 51*`

**Q10.** A new managed node group with `--managed-policies` isn't replicating into Control
Plane Auth. Which add-on/version control concern could cause `aws-auth` to be ignored?
- A. Letting a self-managed "bootstrap" conflict write `aws-auth` non-controlled entries, or the node role missing in `aws-auth`
- B. The VPC CNI needs `serviceAccountName` unchanged
- C. Missing `kubernetes.io/cluster` tag on the API server SG
- D. Node groups can't run more than 16 pods

`*Topic: managed vs self-managed node groups — Day 09*`

**Q11.** You want to restrict the EKS API endpoint to originate only from an on-prem peering
connection. What configuration fits?
- A. Set the endpoint public access to "disabled" and attach a private endpoint to the VPC
- B. Enable public access and set an IP allow-list to the office’s public CIDR
- C. Use a VPC CNI custom networking `secondary` ENI
- D. Configure kube-proxy `--cluster-cidr` to /24

`*Topic: control-plane endpoint access — Day 08, Day 15*`

**Q12.** During service outages you see "no cluster endpoints for `default-backend`" and /healthz
fails. Which silent-but-critical EKS add-on commonly causes this in a fresh cluster?
- A. Managed CoreDNS scaled to 1 replica that's not scaled for load
- B. kube-proxy in IPVS mode requires `ipvsadm`
- C. The default storage class is missing
- D. No `NetworkPolicy` allows CoreDNS

`*Topic: CoreDNS add-on scaling — Day 03, Day 14*`

**Q13.** EKS control plane upgrade from 1.27 → 1.28. The AWS-recommended upgrade path is
(a) upgrade control plane, then (b) upgrade node groups... Which statement about Kubernetes
version skew budgets is correct for node groups (kubelet vs apiserver)?
- A. kubelet may be up to 3 minor versions behind the apiserver
- B. kubelet may be up to 2 minor versions behind (and one up to 3 for some)
- C. kubelet must match exactly
- D. kube-proxy is exempt from skew rules

`*Topic: upgrade process / skew — Day 60/09*`

**Q14.** A cluster has 3 Managed Node Groups with min 1 / max 4 each. Cluster Autoscaler reports
`no scale up: node group max size reached`. You want more headroom without changing each
group's max. What's the correct Cluster Autoscaler behavior?
- A. It will still scale up new node groups it discovers automatically
- B. It can only act within existing node groups (min/max bounds)
- C. Raise `expander` weight globally
- D. Add a `karpenter.sh/provisioner-name` label to it

`*Topic: Cluster Autoscaler constraints — Day 53*`

**Q15.** Bob is migrating a legacy app to EKS. Which combination is the **main** set of
per-pod identity controls they'd rely on for "pods should impersonate an IAM role"?
- A. Security Groups for Pods + IRSA/EKS Pod Identity
- B. Secrets + `aws auth` entries
- C. `hostNetwork: true` + instance profile
- D. Network policies per namespace

`*Topic: pod-level IAM — Day 28*`

**Q16.** You enable **Security Groups for Pods** (SGPP). For VPC CNI to bind a security group
to an ENI for a pod, what must be true about the pods using it?
- A. They must be on nodes with `ENABLE_IPV6`
- B. They must use the `vpc.amazonaws.com/pod-eni` resource (a branch ENI bound to the pod)
- C. They must run as `hostNetwork: true`
- D. They must use prefix delegation

`*Topic: SG for pods — Day 16*`

**Q17.** During an incident, `kubectl get nodes` returns nodes but `kubectl get pods` hangs.
The **failed** component is most likely which EKS control plane piece?
- A. The VPC CNI `aws-node` daemonset
- B. `kube-scheduler` (in the managed control plane)
- C. `kube-apiserver` isn't responding (watch endpoints, sellers: watch/etcd)
- D. CoreDNS is offline

`*Topic: control-plane troubleshooting — Day 08, Day 14*`

**Q18.** EKS enables **secret encryption with KMS**. What exactly does the KMS CMK encrypt?
- A. Only Secret objects
- B. All etcd object data written to the control plane etcd (Secrets and other sensitive objects)
- C. Only the TLS certs in the nodes
- D. Only `ServiceAccounts`

`*Topic: KMS envelope encryption — Day 15*`

**Q19.** You define a Fargate Profile `namespaceSelector` to run workloads in namespace
`batch` on Fargate. Critically, the control plane's **cluster-autoscaler/node group** plays
what role for that namespace after the profile is created?
- A. Fargate pods fully bypass EKS-managed node groups; the profile's `namespaceSelector` decides scheduling
- B. Managed node groups must be deleted first, or Fargate won't activate
- C. Only `default` namespace can run on Fargate
- D. Fargate requires `hostNetwork`

`*Topic: Fargate profiles — Day 08*`

**Q20.** Your HPA read `Current metrics: CPU utilization 72%` and scaled 3→6. The app is
latency-sensitive and you need scale-on-requests. Which is the correct `autoscaling` API
today for a custom metric?
- A. `autoscaling/v2beta2` with `metrics[].type: Pods`
- B. `autoscaling/v2` with `type: Pods` (pod metrics) per the stable API
- C. `autoscaling/v1` with `targetCPUUtilizationPercentage`
- D. `autoscaling/v2beta1` with `metrics[].name: requests-per-second`

`*Topic: HPA autoscaling/v2 — Day 44, Day 53*`

**Q21.** Team wants a GitOps-driven deployment. They install Argo CD and sync an Application;
a manual drift appears (someone applied a change outside git). What Argo CD setting keeps
git as source of truth?
- A. `syncPolicy.automated.prune: true`
- B. `syncPolicy.automated.selfHeal: true`
- C. `compareOptions.IgnoreExtensions`
- D. `syncPolicy.retry`

`*Topic: GitOps reconciliation — Day 34*`

**Q22.** A CI/CD job uses `aws ecr-get-login-password` and `kubectl set image`. For the
deployment to be triggered by image update (or, better, reach GitOps), which service-perm
is the pipeline’s IAM **TOKEN** for ECR? (pick the primary role it needs)
- A. `ecr:GetLoginPassword` + `ecr:GetAuthorizationToken` + `Push`/`Pull`
- B. `eks-auth` read token
- C. `s3:GetObject` on the registry bucket
- D. `acm:RequestCertificate` scope

`*Topic: pipeline/ECR/IAM — Day 37*`

**Q23.** You want to be alerted in CloudWatch each time a pod restarts. Which EKS-general
approach gives you pod-scoped restart metrics without wiring per-service Prometheus?
- A. CloudWatch Container Insights (metrics for `pod_restart_count`)
- B. CloudWatch Synthetics
- C. Amazon DevOps Guru for Kubernetes-only
- D. Running `kubectl get events | restart` in cron

`*Topic: Container Insights — Day 14, Day 44*`

**Q24.** "kubectl get pods" works, but a newly-joined worker node does not register. `aws-auth`
has the instance role. What is the **top** thing to check that's EKS-specific?
- A. The node uses the wrong AMI family (e.g., Bottlerocket vs AL2 recompat)
- B. The `kubernetes.io/cluster/<name>` tag with value `shared/owned` is missing from the ASG/subnets (control plane waits for it)
- C. The kube-proxy image is outdated
- D. The node is in a different account

`*Topic: cluster tags / node join — Day 09-10*`

**Q25.** Which OS is purpose-built for running on EKS: secure, minimal security-hardened,
with an immutable* filesystem and a projection of the host kernel, and AWS-signed updates?
- A. Amazon Linux 2023 (AL2023)
- B. Bottlerocket
- C. Ubuntu EKS
- D. Amazon EKS Anywhere's needs

`*Topic: Bottlerocket — Day 08/15*`

**Q26.** A migration tool question: "I want a single declarative YAML that creates a VPC,
subnets, the EKS cluster, and a node group in one command." Which tool best matches?
- A. `eksctl`
- B. `kops`
- C. AWS CLI `eks create-cluster` alone
- D. `kubectl apply` of a `cluster` object

`*Topic: eksctl vs Terraform — Day 09*`

**Q27.** There's an AWS **service quota** that caps your account per-region (e.g., "VPCs per
region" / "EIPs"). For high pod density in one EKS VPC, the key quota watch is usually
which of the following (above ENI addressing)?
- A. The "secondary IP addresses per network interface" limit — prefix delegation multiplies addresses per ENI
- B. The "rule sets per security group"
- C. The "load balancers per region" 20 limit for each ALB/NLB
- D. The "EBS volumes per instance" quota

`*Topic: quotas + CNI addressing — Day 10, Day 16*`

**Q28.** A network policy `to: kube-system` is not blocking traffic between pods in the same
service. Which EKS reality explains why CNI policy enforcement often needs an add-on
beside the built-in NetworkPolicy?
- A. EKS VPC CNI can enforce pod-to-pod but many teams install Calico/Weave for richer policy (and the built-in needs `Calico` policy controller for endpoint rules)
- B. NetworkPolicy doesn't work on EKS
- C. Only AWS `nsx_networking` has policy
- D. Policy only applies with `hostNetwork`

`*Topic: NetworkPolicy/Calico — Day 17*`

**Q29.** During a re-baseline, the team wants per-node pod limits raised. The Modern way to
increase pod density (rather than adding ENIs) on EKS managed node groups is:
- A. Prefix delegation on the VPC CNI (larger /28 range per ENI → up to 110 more IPs)
- B. Setting kubelet `--max-pods` to 500 manually on the AMI
- C. Creating one ENI per pod
- D. Moving the node to an instance family with fewer ENIs but more vCPU

`*Topic: prefix delegation / max-pods — Day 16*`

**Q30.** When the ALB/NLB ingress takes over 20 load balancers, which two tools (pick the pair
**that is the recommended AWS answer**) give you ingress for Kubernetes *and* keep the
"LB per service" quota in check?
- A. Use one shared ALB with path/host routing (many services per ingress) + track quotas
- B. Enable CloudFront in front of every Service
- C. Set `externalTrafficPolicy: Local` on every LB
- D. Use AWS Global Accelerator for every Service

`*Topic: ALB ingress multi-service + quotas — Day 13*`

---

## ✅ Answer Key

| # | Ans | Explanation |
|---|-----|-------------|
| Q01 | **B** | EBS volumes are availability-zone–bound; the PV and the pod must be in the same AZ. EFS/FSx are the cross-AZ filesystem choice. *(Day 12, Day 21)* |
| Q02 | **B** | Prefix delegation assigns a /28 per ENI so one ENI supports up to ~110 pods, massively reducing ENI pressure — the first-class fix for IP exhaustion on standard EKS. *(Day 16)* |
| Q03 | **A** | `aws-auth` (kube-system) maps IAM principals → Kubernetes RBAC users/groups; RBAC authorizes the API calls themselves. *(Day 11)* |
| Q04 | **A** | IRSA works because the kubelet projects a signed OIDC token into the SA; the pod passes it to STS `AssumeRoleWithWebIdentity`, retrieving short-lived role creds — no node IAM needed. *(Day 26-27)* |
| Q05 | **B** | EKS Pod Identity attaches IAM roles to workload identities (pods) via a new trust/`AmazonEKS*` flow, removing the need to craft per-SA trust policies and adding conditions on app namespaces on current EKS versions. *(Day 27 + current best practices)* |
| Q06 | **B** | The ALB controller discovers subnets **by tags** (`kubernetes.io/role/elb` or `/internal-elb`). Missing tags → controller can't create the LoadBalancer. *(Day 13)* |
| Q07 | **B** | EBS is single-node creature (one writer). For one file system shared by many pods, EFS (NFS) — or FSx for Lustre for HPC — is the intended model. *(Day 21, Day 05)* |
| Q08 | **B** | Karpenter provisions *nodes* from schema demand with custom instance-type mixing and Spot bursts — the AWS-recommended autoscaler compared to Cluster Autoscaler's node-group-bound model. *(Day 53)* |
| Q09 | **B** | Graceful eviction = PDB (protect quorum/minAvailable) + cordon/drain; the node lifecycle (ASG:InstanceRebalance) signals interruption and the node role/controller drains. *(Day 51)* |
| Q10 | **A** | Managed node groups register via `aws-auth` automatically; a conflicting bootstrap or missing IAM role entry in `aws-auth` (control plane side) breaks node registration. *(Day 09)* |
| Q11 | **A** | For on-prem only access, disable public endpoint and enable **private** endpoint access (or public-disabled + private). IP allow-listing would open it publicly. *(Day 08, Day 15)* |
| Q12 | **A** | CoreDNS is a managed add-on; in small clusters it commonly runs 1-2 replicas. On-demand load spikes create "no endpoints" for service discovery — scale via the add-on (autoscaler) before production. *(Day 03, Day 14)* |
| Q13 | **B** | kubelet may be up to 2 minor versions behind the apiserver (unofficially n-2..n-3 leniency), which is why the recommended upgrade path upgrades nodes *after* (not before) the control plane. *(Day 09/60)* |
| Q14 | **B** | Cluster Autoscaler doesn't create new node groups — it only changes desired counts *within* configured groups bounded by min/max. That's precisely why Karpenter's schema-driven provisioning wins at scale. *(Day 53)* |
| Q15 | **A** | Pods get identity via SG for Pods (network identity) + IRSA/Pod Identity (IAM identity). *(Day 28)* |
| Q16 | **B** | SGPP uses branch ENIs (`vpc.amazonaws.com/pod-eni`) attached to pods so each pod can have distinct SGs; standard ENI+SG flow with the VPC CNI. *(Day 16)* |
| Q17 | **C** | `nodes` listed but pods hang → the API control path (apiserver/etcd) or RBAC/selector problem. The node list comes from the same apiserver, so a live node list + hanging pod list usually means watch/API pressure or apiserver degraded, not the CNI. *(Day 08, Day 14)* |
| Q18 | **B** | KMS envelope encryption encrypts all etcd object data written by the control plane — Secrets and other sensitive objects at rest. *(Day 15)* |
| Q19 | **A** | Fargate profiles describe which namespaces/labels are scheduled to Fargate; workloads matching them run serverlessly and never touch managed node groups. *(Day 08)* |
| Q20 | **B** | `autoscaling/v2` is stable and supports `Object`, `Pods`, `External`, `Resource` metric types — write pod-request custom metrics there. *(Day 44, Day 53)* |
| Q21 | **B** | `selfHeal: true` makes Argo CD auto-correct drift back to git every sync/60s; `prune` removes resources deleted from git. Both are the GitOps guarantee. *(Day 34)* |
| Q22 | **A** | The pipeline needs `ecr:GetAuthorizationToken` + push/pull to the repo (authenticating against ECR), then `kubectl` (or GitOps) deploys. *(Day 37)* |
| Q23 | **A** | CloudWatch Container Insights exposes pod-level metrics (restarts, CPU/mem) out of the box — the low-friction observability answer on EKS. *(Day 14, Day 44)* |
| Q24 | **B** | EKS requires the `kubernetes.io/cluster/<name>` tag (owned/shared) on the ASG/subnets so control plane registers the new node; a missing tag is a classic "node never joins" case. *(Day 09-10)* |
| Q25 | **B** | Bottlerocket: purpose-built container OS — immutable filesystem, hardened defaults, minimal attack surface, open-source, used on EKS node AMIs. *(Day 08/15)* |
| Q26 | **A** | `eksctl create cluster` provisions VPC+subnets+EKS+node group from one declarative config. Terraform is the IaC alternative; `eks create-cluster` alone doesn't create worker nodes in a single call. *(Day 09)* |
| Q27 | **A** | Pod density beyond ENI count is the leading quota story: prefix delegation multiplies usable secondary IPs per ENI (the "secondary IP addresses per network interface" service quota). *(Day 10, Day 16)* |
| Q28 | **A** | EKS's built-in NetworkPolicy is enforcement via the CNI policy controller; for extra features (and for many, the Calico policy engine) teams install Calico/Weave on top — the native workable path is CNI tier plus a policy engine. *(Day 17)* |
| Q29 | **A** | Prefix delegation /28 per ENI is the modern density lever on managed node groups (combined with enhanced subnet sizing) — far more IPs per node than ENI-only allocation. *(Day 16)* |
| Q30 | **A** | One shared ALB with host/path routing (ingress `paths` → many Services) is the canonical way to stay comfortably inside the load-balancer-per-region quota; keep an eye on the same ALB's on the account. *(Day 13)* |

**Scoring:** ≥24/30 = strong EKS operational head. Revisit the days tagged on any miss. 🚀