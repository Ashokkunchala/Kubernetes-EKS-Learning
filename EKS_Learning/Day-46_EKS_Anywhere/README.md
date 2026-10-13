# 📅 Day 46: EKS Anywhere

## 🎯 Learning Objectives
By the end of today, you will understand:
- What Amazon EKS Anywhere (EKS-A) is and when to use it
- Running EKS on-premises (bare metal / vSphere)
- EKS-A vs EKS Managed vs EKS Connector
- Cluster lifecycle with the EKS-A CLI (eksctl anywhere)
- GitOps and curated packages in EKS-A

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **EKS Anywhere** | AWS-supported tooling for creating and operating Kubernetes clusters in customer-managed environments. |
| **Control plane** | Components that expose the Kubernetes API and coordinate cluster state. |
| **Bare metal** | Physical servers rather than virtual machines. |
| **Cluster lifecycle** | The work of creating, upgrading, scaling, maintaining, and retiring a cluster. |
| **Distribution** | A packaged Kubernetes implementation and its supporting components. |
| **Management responsibility** | The tasks your team must perform even when software/tools are supported by a vendor. |

**Analogy:** EKS Anywhere provides a managed approach to the blueprint and tools, but the customer still owns the physical site and many operational responsibilities.

**Try / observe:** Separate which components are vendor-managed/supportable from which are your team's responsibility: hardware, networking, upgrades, backups, monitoring, and incident response.

**Troubleshooting clue:** In on-prem environments, check infrastructure prerequisites and network/DNS/certificate reachability before treating a failed cluster component as an application issue.

**Explain it back:** How does EKS Anywhere differ from the EKS managed control plane in AWS?

---

## 📖 Core Concepts

### EKS Anywhere Positioning

```
┌───────────────────────────────────────────────────────────────┐
│                    KUBERNETES ON AWS                           │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  EKS (cloud) │  │ EKS Anywhere │  │ EKS Connector│         │
│  │  managed      │  │  on-prem     │  │  link ext    │        │
│  │  control plane│  │  self-managed│  │  clusters    │        │
│  │   + nodes     │  │  control +   │  │  to console  │        │
│  └──────────────┘  │  nodes        │  └──────────────┘         │
│                    └──────────────┘                           │
│                                                               │
│  EKS-A control plane runs ON YOUR infra (vSphere/bare metal)  │
│  You manage the control plane + nodes (Day-2 is on you).     │
└───────────────────────────────────────────────────────────────┘
```

### When to Choose EKS-A

- **Data residency / compliance** — workloads must stay on-premises.
- **Latency** — workloads need low latency near existing on-prem data.
- **Consistency** — run the same EKS APIs/tooling as cloud.
- **You have capacity** — you own lifecycle & HA for control plane.

**Trade-off:** You give up AWS-managed control plane; you gain data locality and reduced cloud spend for heavy on-prem workloads.

---

## 🛠️ EKS-A Components

| Component | Role |
|-----------|------|
| `eksctl anywhere` | CLI to create/manage clusters |
| **Bootstrap cluster** | Temporary Kubernetes cluster that provisions the target |
| **Management cluster** | Long-lived, runs workload clusters (multi-cluster) |
| **Workload cluster** | Runs your applications |
| **CAPV/CAPI** | Cluster API providers (vSphere, Bare Metal) |
| **Curated packages** | Preinstalled add-ons (Cilium, ArgoCD, cert-manager) |

---

## 🛠️ Step 1: Install EKS-A CLI

```bash
# On an admin machine
export EKSA_RELEASE_VERSION=v0.18.0
curl "https://anywhere.eks.amazonaws.com/releases/eks-a/${EKSA_RELEASE_VERSION}/artifacts/eks-a/v0.18.0/linux/amd64/eksctl-anywhere-v0.18.0-linux-amd64.tar.gz" -o eks-a.tar.gz
tar -xzf eks-a.tar.gz
mv eksctl-anywhere /usr/local/bin/

eksctl anywhere version
```

> CLI + Docker on the admin host act as the **bootstrap** mechanism that creates the management/workload cluster on vSphere or bare metal.

---

## 🛠️ Step 2: Generate & Edit Cluster Config

```bash
# vSphere example
eksctl anywhere generate clusterconfig mycluster \
  --provider vsphere > cluster.yaml
```

`cluster.yaml` (key fields):

```yaml
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: Cluster
metadata:
  name: myworkload
  namespace: default
spec:
  clusterNetwork:
    cni: cilium
    pods:
      cidrBlocks: [192.168.0.0/16]
    services:
      cidrBlocks: [10.96.0.0/12]
  controlPlaneConfiguration:
    count: 3
    machineGroupRef: { name: myworkload-control-plane }
  workerNodeGroupConfigurations:
  - count: 3
    machineGroupRef: { name: myworkload-md-0 }
  externalEtcdConfiguration:
    count: 3
    machineGroupRef: { name: myworkload-etcd }
  gitOpsRef:
    kind: GitOpsConfig
    name: my-gitops
--- # vSphereDatacenterConfig + MachineConfigs follow
```

---

## 🛠️ Step 3: Create the Cluster

```bash
# Create (bootstrap → management → workloads)
eksctl anywhere create cluster -f cluster.yaml

# Verify
export KUBECONFIG=$PWD/myworkload/myworkload-eks-a-cluster.kubeconfig
kubectl get nodes
kubectl get pods -A

# Get cluster info
eksctl anywhere get clusters
```

> For multi-cluster: create a **management** cluster first, then workload clusters under it — giving central GitOps and lifecycle management.

---

## 🛠️ Step 4: Enable GitOps (Flux)

Add a GitOpsConfig to sync cluster config from Git:

```yaml
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: GitOpsConfig
metadata:
  name: my-gitops
spec:
  flux:
    github:
      owner: yourorg
      repository: eksa-gitops
      branch: main
      clusterConfigPath: clusters/myworkload
      fluxSystemNamespace: flux-system
      personal: false
      provideClusterRole: true
      namespace: eksa-system
```

```bash
eksctl anywhere apply cluster -f cluster.yaml --kubeconfig $KUBECONFIG
```

---

## 🛠️ Step 5: Curated Packages & Day-2 Ops

```bash
# Install a curated package (e.g., cert-manager, ArgoCD)
eksctl anywhere create package cert-manager \
  --cluster myworkload --kubeconfig $KUBECONFIG

# Upgrade the cluster
eksctl anywhere upgrade cluster -f cluster.yaml

# Delete
eksctl anywhere delete cluster -f cluster.yaml
```

### Maintenance & Backup

- **etcd backup:** `etcdutl snapshot save` from etcd pods.
- **Upgrade path:** Always test on a staging EKS-A; upgrade control plane + nodes in a controlled order.
- **Node OS updates:** Rebuild node machines from the machine image using the documented roll strategy.

---

## 📝 Interview Questions

1. **What is EKS Anywhere and when would you use it?**
   - **Answer:** EKS run on your own on-premises infra (vSphere/bare metal) with the same EKS API and tooling — for data residency, low latency, or consistency when you must keep workloads on-prem.

2. **How is EKS-A different from managed EKS?**
   - **Answer:** In EKS-A you operate the control plane, etcd, and nodes yourself; managed EKS runs those for you. Same tooling (kubectl, eksctl) but different operational ownership.

3. **What is a management vs workload cluster in EKS-A?**
   - **Answer:** The management cluster runs the lifecycle/GitOps tooling and hosts workload clusters; workload clusters run your apps.

4. **How do you upgrade EKS-A safely?**
   - **Answer:** Follow the documented upgrade path: upgrade the management cluster, then workload clusters, node-by-node, with etcd backups and staged testing.

5. **EKS-A vs EKS Connector?**
   - **Answer:** EKS-A is a full managed-by-you cluster running on-prem; EKS Connector merely registers external/on-prem clusters so they appear in the AWS console — it does not run or manage anything.

---

## 📚 Resources for Review

- [Day 47: Multi-Cluster Networking](Day-47_Multi_Cluster_Networking/README.md)
- [Day 48: Cluster Federation](Day-48_Cluster_Federation/README.md)

### External
- [Amazon EKS Anywhere](https://anywhere.eks.amazonaws.com/)
- [EKS-A docs](https://anywhere.eks.amazonaws.com/docs/)

---

## 🎯 Today's Assignment

1. **Install** the EKS-A CLI and verify version
2. **Generate** a cluster config for your provider (vSphere or bare metal)
3. **Review** the control-plane, etcd, and worker machine configs
4. **Enable** GitOps for a cluster config repo
5. **Document** an upgrade + backup runbook for EKS-A

---

## 📊 Progress Tracker

```
Week 15-16 Multi-Cluster:
Day 46: EKS Anywhere ⬜
Day 47: Multi-Cluster Networking ⬜
Day 48: Cluster Federation ⬜
Day 49: Multi-Cluster Practice ⬜
Day 50: Multi-Cluster Interview Prep ⬜
```

---

## 🔗 Next Week Preview

**Day 47:** Multi-Cluster Networking (Service Mesh, Mesh across clusters)

---

**Remember:** EKS-A is real K8s you own — treat the control plane like production infrastructure! 🚀
