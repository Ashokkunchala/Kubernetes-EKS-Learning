# 🧪 EKS Hands-On Labs — Index

Short, complete, hands-on labs that **prove** core Kubernetes behaviors with real manifests and real `kubectl` output. Each lab is self-contained, runs in **under 30 minutes**, and works on any cluster — `kind`, Minikube, k3s, or a real Amazon EKS cluster.

## 🎯 Lab Goal

The day READMEs teach concepts; these labs turn them into muscle memory. By the end of all four labs you will have:

- Proved the pod lifecycle end-to-end (deploy → scale → update → rollback → self-heal)
- Taken an app from **stateless** to **stateful** with PVCs and a StatefulSet
- Built **microsegmentation** with Network Policies and verified allow/deny traffic
- **Diagnosed and fixed** four intentionally broken workloads

## 🗺️ Lab Index

| Lab | What you prove | Related Day(s) | Time |
|-----|----------------|----------------|------|
| [Lab-01_Deploy_Scale_Update.md](Lab-01_Deploy_Scale_Update.md) | Pod lifecycle: deploy, scale, rolling update, rollback, self-healing | [Day-02](../Day-02_Pods_ReplicaSets_Deployments/README.md), [Day-03](../Day-03_Services_Networking/README.md) | 20 min |
| [Lab-02_Storage_Stateful_App.md](Lab-02_Storage_Stateful_App.md) | Stateless → stateful: emptyDir, PVC persistence, StatefulSet ordinals | [Day-05](../Day-05_Persistent_Storage/README.md), [Day-12](../Day-12_EKS_Storage_EBS/README.md), [Day-21](../Day-21_EFS_CSI_Driver/README.md) | 30 min |
| [Lab-03_Network_Policies.md](Lab-03_Network_Policies.md) | Microsegmentation: default-deny, allow apps, verify +1/-1 | [Day-03](../Day-03_Services_Networking/README.md), [Day-17](../Day-17_Network_Policies_Calico/README.md) | 25 min |
| [Lab-04_Troubleshooting_Game.md](Lab-04_Troubleshooting_Game.md) | Diagnose & fix 4 broken workloads (scored game) | [Day-20](../Day-20_Networking_Interview_Preparation/README.md), [Day-45](../Day-45_Monitoring_Stack_Practice/README.md), [Day-60](../Day-60_Final_Review_Certification_Preparation/README.md) | 30 min |

Each lab has the same anatomy: `## 🎯 Lab Goal` → `## 🧪 Scenario` → `## ▶️ Steps` (with expected `kubectl` output sandwiched between commands) → `## ✅ Solution Recap` → cleanup.

## ▶️ How to Use

These labs run on **any** Kubernetes cluster:

```bash
# Option A — kind (zero AWS cost, fastest)
kind create cluster --name labs

# Option B — Minikube
minikube start

# Option C — existing EKS cluster
kubectl cluster-info   # just verify your kubeconfig context
```

Then pick a lab, copy-paste the blocks in order, and compare your output to the **expected output** shown under each command.

```bash
kubectl get nodes                          # sanity check
kubectl config current-context
```

## ✅ Prerequisites

| Tool | Purpose | Lab |
|------|---------|-----|
| `kubectl` v1.28+ | Everything — install to match your cluster: `aws eks update-kubeconfig` or `kind`'s auto-config | All |
| `kind` or `minikube` OR an EKS cluster | Where the workloads run | All |
| Policy-enforcing CNI (**Calico** or **Cilium**) | Enforces NetworkPolicies — kind's default CNI does **not** | [Lab-03](Lab-03_Network_Policies.md) |
| local-path provisioner (kind) OR EBS/EFS CSI driver add-on (EKS) | Dynamically provisions volumes | [Lab-02](Lab-02_Storage_Stateful_App.md) |

> On EKS, install the Calico add-on (see [Day-17](../Day-17_Network_Policies_Calico/README.md)) and the EBS CSI driver ([Day-12](../Day-12_EKS_Storage_EBS/README.md)) before running the storage and network policy labs.

If a lab mentions a manifest you'd rather install add-on-free, see the [manifests folder](../manifests/README.md) for a status check reminder of which drivers are already assumed.

## 🧹 Cleanup

Each lab ends with its own cleanup block. The universal reset:

```bash
kubectl delete ns lifecycle-lab storage-lab netpol-lab troubleshoot-lab
kind delete cluster --name labs        # if using kind
```

## 📚 Resources

- [Kubernetes docs — workloads](https://kubernetes.io/docs/concepts/workloads/controllers/)
- [Kubernetes docs — Storage](https://kubernetes.io/docs/concepts/storage/)
- [Kubernetes docs — Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Troubleshooting cluster apps](https://kubernetes.io/docs/tasks/debug/debug-cluster/)

**Rule of thumb:** never trust a claim about Kubernetes behavior until you can point at a `kubectl get` output that proves it. These labs are built around that rule. 🚀