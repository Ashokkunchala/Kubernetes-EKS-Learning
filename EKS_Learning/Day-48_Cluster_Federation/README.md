# 📅 Day 48: Cluster Federation

## 🎯 Learning Objectives
By the end of today, you will understand:
- What cluster federation is and the problems it solves
- KubeFed / Federation V2 concepts
- When to federate vs use GitOps per cluster
- Multi-cluster scheduling, placement, and overrides
- Modern alternatives (ArgoCD multi-cluster, GitOps)

---

## 📖 Core Concepts

### Federation Model

```
┌───────────────────────────────────────────────────────────────┐
│                         FEDERATION                             │
│                                                               │
│  ┌──────────────────────────────────────────────┐            │
│  │           CONTROL PLANE (KubeFed host)       │            │
│  │  defines FEDERATED resources + Placement     │            │
│  └──────┬──────────┬──────────┬─────────────────┘            │
│         │          │          │                                 │
│    ┌────▼────┐ ┌───▼─────┐ ┌──▼─────┐                       │
│    │ Cluster1│ │Cluster2 │ │Cluster3│                       │
│    │ (prod)  │ │(stage)  │ │(dev)   │                       │
│    │  fed     │ │  fed    │ │  fed   │                       │
│    │  config  │ │  config │ │ config │                       │
│    └─────────┘ └─────────┘ └────────┘                       │
│                                                               │
│  Placement rules decide WHICH clusters get WHICH resources.   │
│  Federation propagates + reconciles the desired state.        │
└───────────────────────────────────────────────────────────────┘
```

### Core Concepts

- **Federated Type API:** Describes resources to distribute (e.g., `FederatedDeployment`).
- **Placement:** Selects target clusters via `ReplicaSchedulingPreference`.
- **Overrides:** Cluster-specific tweaks (e.g., image tag or replicas per cluster).
- **Propagation:** The host cluster pushes down to member clusters and reconciles drift.

---

## 🛠️ Step 1: Install KubeFed

```bash
# Use Helm to deploy KubeFed on the host cluster
helm repo add kubefed https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
helm repo update

# Create a namespace and install
kubectl create ns kube-system  # already exists
helm install kubefed kubefed/kubefed --create-namespace --namespace kube-fed-system
```

```bash
# Add member clusters
kubefedctl join member-1 \
  --host-cluster-context host-ctx \
  --cluster-context member-1-ctx \
  --v=2

kubefedctl join member-2 \
  --host-cluster-context host-ctx \
  --cluster-context member-2-ctx
```

```bash
# Verify
kubectl -n kube-fed-system get kubefedclusters
```

---

## 🛠️ Step 2: Federate a Deployment

```bash
# Enable federation for the Deployment type
kubefedctl enable deployment | kubectl apply -f -
```

Create a federated deployment:

```yaml
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: web-frontend
  namespace: production
spec:
  template:
    metadata:
      labels: { app: web-frontend }
    spec:
      replicas: 2
      selector: { matchLabels: { app: web-frontend } }
      template:
        metadata:
          labels: { app: web-frontend }
        spec:
          containers:
          - name: web
            image: myorg/web:v1
```

```bash
kubectl apply -f federated-deployment.yaml
```

---

## 🛠️ Step 3: Placement (Replica Scheduling)

Control which clusters get how many replicas:

```yaml
apiVersion: scheduling.kubefed.io/v1alpha1
kind: ReplicaSchedulingPreference
metadata:
  name: web-frontend
  namespace: production
spec:
  targetKind: FederatedDeployment
  totalReplicas: 6
  clusters:
    member-1:
      weight: 2
      maxReplicas: 4
    member-2:
      weight: 1
      minReplicas: 1
  overrides:
  - clusterName: member-1
    clusterOverrides:
    - path: "/spec/template/spec/containers/0/image"
      value: "myorg/web:v1-eu"
```

> The scheduler keeps total replicas ≈ `totalReplicas` distributed per the weights.

---

## 🛠️ Step 4: Overrides for Cluster-Specific Config

```yaml
apiVersion: types.kubefed.io/v1beta1
kind: FederatedConfigMap
metadata:
  name: app-config
  namespace: production
spec:
  template:
    data:
      region: default
  overrides:
  - clusterName: member-1
    clusterOverrides:
    - path: "/data/region"
      value: "eu-west-1"
  - clusterName: member-2
    clusterOverrides:
    - path: "/data/region"
      value: "us-east-1"
```

---

## 🛠️ Federation vs Modern GitOps (Important)

Federation (KubeFed) is **largely legacy** — most teams now use **GitOps per cluster**:

| Concern | KubeFed Federation | GitOps (ArgoCD multi-cluster) |
|---------|--------------------|-------------------------------|
| Propagate changes | Host pushes down | ArgoCD syncs manifests per cluster |
| Placement | Replica scheduling | ApplicationSet generators/clusters |
| Overrides | Federation overrides | Kustomize/Helm per environment |
| Git as source | No (API driven) | Yes (SSOT) |
| Drift healing | Native federation | selfHeal |
| Maturity | De-emphasized | Actively developed |

**Recommendation:** Use **ArgoCD + ApplicationSet** for multi-cluster workload deployment; reserve federation ideas (placement/overrides) as design concepts.

---

## 🛠️ ArgoCD Multi-Cluster as the Modern Approach

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: appset-clusters
  namespace: argocd
spec:
  generators:
  - clusters: {}   # all registered clusters
  template:
    metadata:
      name: '{{name}}-web'
    spec:
      project: default
      source:
        repoURL: https://github.com/yourorg/gitops-repo.git
        path: 'environments/{{name}}/web'
        targetRevision: main
      destination:
        server: '{{server}}'
        namespace: production
```

---

## 📝 Interview Questions

1. **What is cluster federation?**
   - **Answer:** A control-plane pattern to manage and propagate identical/similar workloads across multiple Kubernetes clusters from a single API, with placement and overrides.

2. **KubeFed vs ArgoCD multi-cluster?**
   - **Answer:** KubeFed propagates from a host API; ArgoCD uses Git as the single source of truth and syncs per cluster. ArgoCD is the de-facto modern choice (GitOps).

3. **What are placement and overrides?**
   - **Answer:** Placement decides which clusters run a workload and how many replicas; overrides tailor cluster-specific values (image, region config) on top of the base template.

4. **When would you still use federation instead of GitOps?**
   - **Answer:** Rarely today; only if you need tight, API-driven dynamic scheduling across many clusters without central Git. Most orgs prefer per-cluster GitOps for auditability.

5. **How do you avoid config drift across many clusters?**
   - **Answer:** GitOps with selfHeal (ArgoCD) reconciles drift automatically; plus centralized policies (Kyverno/OPA) enforced at admission in each cluster.

---

## 📚 Resources for Review

- [Day 35: ArgoCD Advanced](Day-35_ArgoCD_Advanced_Best_Practices/README.md)
- [Day 33: ArgoCD Application Patterns](Day-33_ArgoCD_Application_Patterns/README.md)

### External
- [KubeFed](https://github.com/kubernetes-sigs/kubefed)
- [ArgoCD AppicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/)

---

## 🎯 Today's Assignment

1. **Install** KubeFed on a host cluster (or simulate)
2. **Federate** a deployment and a ConfigMap
3. **Configure** replica scheduling across 2 clusters
4. **Set** per-cluster overrides and verify propagation
5. **Compare** with an ArgoCD ApplicationSet and note the differences

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

**Day 49:** Multi-Cluster Practice

---

**Remember:** Federation is a tool, GitOps is a discipline — today GitOps wins production. Know both! 🚀
