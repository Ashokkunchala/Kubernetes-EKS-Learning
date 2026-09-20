# 🧩 Module D: Progressive Delivery with Argo Rollouts

> **Prereq:** [Day 31 GitOps Intro](../Day-31_GitOps_ArgoCD_Intro/README.md) · [Day 33 ArgoCD Application Patterns](../Day-33_ArgoCD_Application_Patterns/README.md) · [Day 34 ArgoCD Sync & Rollback](../Day-34_ArgoCD_Sync_Rollback/README.md)

Kubernetes Deployments give you rolling updates and instant rollback — but no *canary* analysis, no *blue-green* step control, and no automated "was it actually good?". Argo Rollouts is the progressive-delivery layer that adds those steps.

## 🎯 Learning Objectives

By the end of this module, you will understand:
- What the built-in Deployment strategy can and cannot do
- The Argo Rollouts controller + CRD and the `kubectl argo rollouts` interface
- A BlueGreen `Rollout` + `AnalysisTemplate` end to end
- How Argo Rollouts differs from ArgoCD's Application sync
- Where Flagger fits and how it compares

---

## 📖 Core Concepts

### The Deployment's limits

```
Deployment (built-in)                        Rollout (Argo)
─────────────────                           ──────────────
Recreate  (old→new)      RollingUpdate      Canary    (x% → y% → 100%)
RollingUpdate (surface)   └── 2 strategies   BlueGreen (v1 warm up → swap)
  └ minReadySeconds        └ no analysis     └ analysis (Prometheus / web)
  └ maxSurge/maxUnavailable  └ no verification
  └ no gates                └ no manual promote
  └ fails after readiness    └ auto-abort on metric breach
```

In short: Deployments answer *"did my pods become Ready?"*. Rollouts answer *"is this release actually good enough to take 100% traffic?"*

### Argo Rollouts architecture

```
┌─────────────────────────────────────────────────────────┐
│                   ARGO ROLLOUTS                          │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  rollout-controller (in cluster)                 │   │
│  │  watches Rollout + AnalysisTemplate + Metrics    │   │
│  └───────┬──────────────────────────┬───────────────┘   │
│          │ apply                     │ analyze          │
│  ┌───────▼──────────┐       ┌────────▼──────────────┐  │
│  │  ReplicaSets     │       │  prometheus /        │  │
│  │  stable (v1)     │◄────×►│  web / job query      │  │
│  │  canary (v2)     │ gate  │  success/failure      │  │
│  └───────┬──────────┘       └────────┬──────────────┘  │
│          │ traffic: service + ingress│                  │
│  ┌───────▼───────────────────────────▼───────────────┐ │
│  │  BlueGreen: preview svc ──► promo svc (cutover)    │ │
│  │  Canary:   incremental weight in stable Service    │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### InstallStep — the "analysis hooks" workflow

An `AnalysisTemplate` runs analysis **in steps**: each step can hold traffic, query metrics, wait for a job, or fail. Steps are skipped/kept per `setWeight`, `pause`, and `canary` metadata.

---

## 🚀 Hands-On: BlueGreen Rollout + AnalysisTemplate

### Step 1: Install Argo Rollouts

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f \
  https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# CLI for promote/abort + dashboard
curl -sSLo kubectl-argo-rollouts https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts
sudo mv kubectl-argo-rollouts /usr/local/bin/

# Live dashboard (optional)
kubectl argo rollouts dashboard
```

### Step 2: AnalysisTemplate — "did 5xxs spike?"

```yaml
# analysis-template-http.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: http-success-rate
spec:
  args:
  - name: namespace
  metrics:
  - name: success-rate
    interval: 30s                     # sample every 30s
    count: 2                          # gather 2 samples before passing
    successCondition: result >= 0.98  # >98% non-5xx = good
    failureLimit: 1                   # 1 bad sample → abort/mark fail
    provider:
      prometheus:
        address: http://prometheus.monitoring:9090
        query: |
          sum(rate(http_requests_total{status=~"5..", namespace="{{args.namespace}}"}[1m]))
          /
          sum(rate(http_requests_total{namespace="{{args.namespace}}"}[1m]))
```

### Step 3: The Rollout (BlueGreen)

```yaml
# rollout-bluegreen.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-bluegreen
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-bluegreen
  template:
    metadata:
      labels:
        app: my-bluegreen
    spec:
      containers:
      - name: app
        image: public.ecr.aws/docker/library/nginx:1.27-alpine
        ports:
        - containerPort: 80
  strategy:
    blueGreen:
      activeService: my-bluegreen-active      # serves prod traffic
      previewService: my-bluegreen-preview    # warm-up target for new version
      autoPromotionEnabled: false             # require manual promote
      autoPromotionSeconds: 0
      analysis:
        templates:
        - templateName: http-success-rate
          args:
          - name: namespace
            value: default
```

```yaml
# the two services backing the Rollout
apiVersion: v1
kind: Service
metadata:
  name: my-bluegreen-active
spec:
  selector:
    app: my-bluegreen
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-bluegreen-preview
spec:
  selector:
    app: my-bluegreen
  ports:
  - port: 80
    targetPort: 80
```

### Step 4: Drive the rollout

```bash
kubectl argo rollouts create rollout-bluegreen.yaml

# Simulate a new release (new image tag)
kubectl argo rollouts set image my-bluegreen app=nginx:1.28-alpine

# Watch phases:
#   new RS (v2) is scaled up → analysis runs → autoPromotionEnabled=false
kubectl argo rollouts get rollout my-bluegreen

# Promote manually when metrics are green
kubectl argo rollouts promote my-bluegreen

# If analysis fails → controller auto-aborts back to stable
kubectl argo rollouts abort my-bluegreen
kubectl argo rollouts get rollout my-bluegreen   # Status: Degraded, Aborted
```

---

## 🧩 Argo Rollouts vs ArgoCD Application Sync

These are **different layers** — a common interview trap:

```
ArgoCD = DELIVERY OF DECLARED STATE (GitOps)
  └ watches git repo → syncs ANY manifests (incl. a Rollout)
  └ team: "is the repo's YAML live in the cluster?"

Argo Rollouts = DEPLOYMENT STRATEGY (progressive delivery)
  └ takes over the Deployment lifecycle once in cluster
  └ team: "is the NEW RELEASE getting 100% traffic safely?"

Flow:
  Git  ──ArgoCD──►  Rollout object in cluster
                     └──Argo Rollouts──► ReplicaSets + analysis
```

So you keep **both**: ArgoCD syncs the `Rollout` manifest from Git; Argo Rollouts decides how that revision takes traffic. ArgoCD's own *sync window / sync phases* gate *when* Git applies; they do not do canary traffic shaping.

```
┌─────────────────────────────────────────────────────────┐
│   ARGOCD                         ARGO ROLLOUTS           │
│   app lifecycle                  deployment strategy     │
│   - git repo as source           - canary / blue-green   │
│   - sync / self-heal / prune     - analysis (metrics)   │
│   - diff against Git             - pause/promote/abort  │
│   - rollback = revert in Git     - rollback = auto aborts│
│   - works for ANY resource       - only Deployments &    │
│                                    StatefulSet workloads │
└─────────────────────────────────────────────────────────┘
```

---

## ⚖️ Argo Rollouts vs Flagger

| | Argo Rollouts | Flagger |
|---|---|---|
| Install | operator + CRDs, own UI | runs on Hello-Flagger (Prometheus), no own UI |
| Traffic shaping | uses Service/Ingress + native CD services | uses service mesh / ingress providers (Istio, AppMesh) |
| Analysis | AnalysisTemplate CRs; flexible webhooks | baked-in canary metrics via providers |
| Rollout CRD | additive `Rollout` CRD | hooks the built-in `Deployment` via mutations |
| Best EKS setup | AppMesh/Ingress-nginx CRDs | needs Istio or AppMesh wired first |

On EKS with **App Mesh or Istio**, Flagger is excellent. Without a mesh, Argo Rollouts (native `Services` + built-in analysis) is the lower-friction path.

---

## 🛠️ Troubleshooting

```bash
# 1. Rollout stuck "Progressing" forever → check analysis job
kubectl get analysisrun -n default      # AnalysisRun per Rollout revision
kubectl describe analysisrun <name>

# 2. "rollout aborted" but no metric breach → look at failureLimit vs count
#    count:2 + failureLimit:1 → ANY single bad sample within first 2 fails.

# 3. Services not picking up RS (traffic stays on stable):
kubectl get svc my-bluegreen-active -o yaml | grep selector

# 4. AnalysisTemplate does not exist → Rollout won't start:
kubectl get at -n default                # AnalysisTemplate `at`

# 5. On EKS with ingress-nginx: ensure `allow-snippet-annotations: "true"` if
#    using canary ingress annotations, or use the stable Service approach.
```

---

## ❓ Interview Questions

1. **What problem does Argo Rollouts solve beyond Deployments?**
   - Gradual, *verified* rollouts: canary/blue-green traffic shaping, metric-driven auto-promote/abort, and pause/promote workflows that RollingUpdate lacks.

2. **How do ArgoCD and Argo Rollouts interoperate?**
   - ArgoCD keeps Git's manifests live (including the `Rollout` CRD object); Argo Rollouts then manages the deployment strategy of that object. Different layers: GitOps vs progressive delivery.

3. **What is `autoPromotionEnabled: false` doing in BlueGreen?**
   - New version is built and analyzed in the preview environment but traffic still goes to stable until an explicit `kubectl argo rollouts promote`.

4. **How does an AnalysisTemplate abort a release?**
   - The controller runs metric queries on an interval; if successCondition fails more times than failureLimit, the AnalysisRun is marked failed and the Rollout aborts, keeping stable replicas.

5. **Rollouts vs Flagger — when would you pick each?**
   - Rollouts: no mesh required, own CRD/UI, direct. Flagger: when you already run Istio/AppMesh, it mutates the built-in Deployment and integrates with the mesh's traffic split.

---

## 📚 Resources

- [Argo Rollouts docs](https://argoproj.github.io/argo-rollouts/)
- [BlueGreen strategy reference](https://argoproj.github.io/argo-rollouts/features/bluegreen/)
- [AnalysisTemplate reference](https://argoproj.github.io/argo-rollouts/features/analysis/)
- [Flagger docs](https://flagger.app/)

---

## ✅ Progress Tracker

```
Module D: Progressive Delivery
  [x] Deployment limits vs Rollouts capabilities
  [x] InstallStep / controller + CLI explained
  [ ] Installed Argo Rollouts (operator)
  [ ] Created AnalysisTemplate: http-success-rate
  [ ] Created BlueGreen Rollout + 2 Services
  [ ] Executed promote + abort cycles
  [ ] Drew ArgoCD vs Rollouts vs Flagger comparison
```

---

## 👀 Next

- **Put the Rollout in Git + sync it:** [Day 32 ArgoCD Installation & Config](../Day-32_ArgoCD_Installation_Config/README.md) and [Day 33 ArgoCD Application Patterns](../Day-33_ArgoCD_Application_Patterns/README.md)
- **Automate promote when metrics are green (pipeline side):** [Day 37 GitHub Actions EKS](../Day-37_GitHub_Actions_EKS/README.md)
- **Full recap of the GitOps arc:** [Day 35 ArgoCD Advanced Best Practices](../Day-35_ArgoCD_Advanced_Best_Practices/README.md)

---

**Remember:** You only get to "100% traffic" when the analysis agrees — promote with evidence, not hope. 📈