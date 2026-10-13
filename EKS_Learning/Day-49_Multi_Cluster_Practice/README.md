# 📅 Day 49: Multi-Cluster Practice - Hands-On Lab

## 🎯 Learning Objectives
By the end of today, you will:
- Build a 2-cluster (or simulated) multi-cluster setup
- Deploy the same workload to both clusters via GitOps
- Route traffic and validate failover
- Set up cross-cluster monitoring
- Test a realistic multi-cluster scenario end-to-end

---
## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Context** | A kubeconfig selection for a cluster and identity. |
| **Failover** | Moving traffic or workload responsibility from a failed/unavailable system to a healthy alternative. |
| **Traffic shifting** | Changing how much traffic goes to each destination. |
| **Observability** | Using metrics, logs, traces, and other evidence to understand system behavior. |
| **Isolation** | Keeping access, failures, or resources in one environment from unnecessarily affecting another. |
| **Health check** | A test used to decide whether a component can serve work. |

**Analogy:** Two clusters are two sites; failover redirects customers to the backup site, and traffic shifting controls how quickly that redirect happens.

**Try / observe:** Confirm the context before commands, run a harmless read-only query against each cluster, and document a tested failover and return path.

**Troubleshooting clue:** Print `kubectl config current-context` and verify the expected cluster before changes. A correct command against the wrong context is still dangerous.

**Explain it back:** What should be verified before failover? Why should failback be tested too?

---


## 📖 The Lab: Two-Region Active/Active

```
┌───────────────────────────────────────────────────────────────┐
│                2-REGION ACTIVE/ACTIVE LAB                      │
│                                                               │
│   Global Ingress (ALB / Global Accelerator)                   │
│                │                                              │
│        ┌───────┴───────┐                                      │
│        ▼               ▼                                      │
│  ┌────────────┐  ┌────────────┐                               │
│  │ Cluster A  │  │ Cluster B  │                               │
│  │ us-east-1  │  │ us-west-2  │                               │
│  │  ┌───────┐ │  │  ┌───────┐ │                               │
│  │  │  app  │ │  │  │  app  │ │                               │
│  │  └───────┘ │  │  └───────┘ │                               │
│  │  ArgoCD    │  │  ArgoCD    │                               │
│  └────────────┘  └────────────┘                               │
│         │                │                                    │
│         └──────┬─────────┘                                    │
│                ▼                                              │
│       Central GitOps Repo (SSOT)                              │
└───────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Step 1: Create Two Clusters (+ ArgoCD)

```bash
# Cluster A
eksctl create cluster --name eks-a --region us-east-1 \
  --nodegroup-name standard --node-type m5.large --nodes 2

# Cluster B
eksctl create cluster --name eks-b --region us-west-2 \
  --nodegroup-name standard --node-type m5.large --nodes 2

# Install ArgoCD in both (or one ArgoCD controlling both)
kubectl --context eks-a create ns argocd
kubectl --context eks-a apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Register cluster B into A's ArgoCD:

```bash
argocd cluster add "context-eks-b" --name eks-b
argocd cluster list
```

---

## 🛠️ Step 2: GitOps Repo Per Region

```
gitops-repo/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── east/
│   │   └── kustomization.yaml   # region=us-east-1, replicas=3
│   └── west/
│       └── kustomization.yaml   # region=us-west-2, replicas=3
```

`base/kustomization.yaml`:

```yaml
resources:
  - deployment.yaml
  - service.yaml
commonLabels:
  app: web
```

`overlays/east/kustomization.yaml`:

```yaml
resources:
  - ../../base
patches:
  - target: { kind: Deployment, name: web }
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
      - op: add
        path: /spec/template/spec/containers/0/env
        value:
        - name: REGION
          value: us-east-1
```

---

## 🛠️ Step 3: ApplicationSet for Both Regions

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: web-apps
  namespace: argocd
spec:
  goTemplate: true
  generators:
  - list:
      elements:
      - cluster: eks-a
        region: east
      - cluster: eks-b
        region: west
  template:
    metadata:
      name: 'web-{{.region}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/yourorg/gitops-repo.git
        path: 'overlays/{{.region}}'
        targetRevision: main
      destination:
        server: '{{.cluster}}'
        namespace: web
      syncPolicy:
        automated: { prune: true, selfHeal: true }
```

```bash
kubectl --context eks-a apply -f applicationset.yaml
argocd app list   # should show web-east + web-west
```

---

## 🛠️ Step 4: Validate & Route

```bash
# Both clusters running 3 replicas
kubectl --context eks-a -n web get deploy,svc
kubectl --context eks-b -n web get deploy,svc

# Query the REGION env
kubectl --context eks-a -n web exec deploy/web -- printenv REGION
kubectl --context eks-b -n web exec deploy/web -- printenv REGION

# Expose via ALB/Ingress per region & create a Global Accelerator listener
```

---

## 🛠️ Step 5: Failover Test

```bash
# Simulate region A outage: stop Ingress for A / delete cluster A endpoints
kubectl --context eks-a delete svc web-ingress
# Global Accelerator health-checks fail A → routes all to B
# App still served from us-west-2
curl https://global.example.com/health   # responds from B

# Restore A and watch traffic rebalance
kubectl --context eks-a apply -f overlays/east/service.yaml
```

**Also test config change propagate:**

```bash
# Edit base/deployment.yaml → image v2
git add . && git commit -m "bump to v2" && git push
# ArgoCD syncs BOTH clusters to v2 automatically
```

---

## 🛠️ Cross-Cluster Monitoring

Centralized Grafana scraping both clusters:

```yaml
# In central Grafana, add both Prometheus sources
- name: Prometheus-East
  type: prometheus
  url: http://prom-east.monitoring.svc:9090
- name: Prometheus-West
  type: prometheus
  url: http://prom-west.monitoring.svc:9090
```

Or remote-write both to **AMP** and query centrally (best practice):

```bash
# Each cluster runs prometheus remote-write → shared AMP workspace
helm install amp-agent ... --set endpoint=https://aps-workspaces...amazonaws.com
```

---

## 🔧 Troubleshooting

```bash
# App stuck OutOfSync on one cluster
argocd app get web-east --show-operation

# Cluster unreachable in ArgoCD
argocd cluster get eks-b

# Cross-region traffic not failing over
aws globalaccelerator list-endpoint-groups --accelerator-arn ...
aws elbv2 describe-target-health --target-group-arn <tg>
```

---

## 📝 Interview Questions

1. **How do you deploy the same app to multiple clusters from one place?**
   - **Answer:** ArgoCD with an ApplicationSet — one GitOps repo, generators select clusters, and each cluster syncs the same or an overridden manifest.

2. **How do you handle per-region differences?**
   - **Answer:** Kustomize overlays or Helm values per region (replicas, region env, image), referenced by the ApplicationSet template.

3. **How do you fail over between regions?**
   - **Answer:** Global Accelerator (or Route53 + ALB) health-checks regional endpoints; on failure it routes traffic to the healthy region. Data failover (RDS global / DynamoDB global tables) is separate.

4. **How do you monitor many clusters?**
   - **Answer:** Remote-write all clusters to a central store (AMP/Thanos) and query centrally, or scrape each into a central Grafana via multiple data sources.

5. **How do you keep secrets consistent across clusters?**
   - **Answer:** External Secrets / AWS Secrets Manager referencing the same secret, or sealed secrets committed to Git per cluster.

---

## 📚 Resources for Review

- [Day 47: Multi-Cluster Networking](Day-47_Multi_Cluster_Networking/README.md)
- [Day 48: Cluster Federation](Day-48_Cluster_Federation/README.md)

---

## 🎯 Today's Assignment

1. **Stand up** two clusters (or use kind/eksctl)
2. **Centralize** deployment via ArgoCD ApplicationSet
3. **Create** Kustomize overlays (east/west)
4. **Test** config push to both + failover
5. **Set up** central monitoring for both clusters

---

## 📊 Progress Tracker

```
Week 15-16 Multi-Cluster:
Day 46: EKS Anywhere ⬜
Day 47: Multi-Cluster Networking ⬜
Day 48: Cluster Federation ⬜
Day 49: Multi-Cluster Practice ✅ (hands-on)
Day 50: Multi-Cluster Interview Prep ⬜
```

---

## 🔗 Next Week Preview

**Day 50:** Multi-Cluster Interview Preparation

---

**Remember:** Build it once, run it everywhere — GitOps makes multi-cluster the normal case, not the exception! 🚀
