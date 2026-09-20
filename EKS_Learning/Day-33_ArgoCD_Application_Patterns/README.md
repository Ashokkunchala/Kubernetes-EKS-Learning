# 📅 Day 33: ArgoCD Application Patterns

## 🎯 Learning Objectives
By the end of today, you will understand:
- Using Helm with ArgoCD
- Using Kustomize with ArgoCD
- Multi-environment patterns
- App-of-Apps pattern
- ApplicationSets for automation

---

## 📖 Core Concepts

### ArgoCD Application Patterns

```
┌─────────────────────────────────────────────────────────┐
│          ARGOCD APPLICATION PATTERNS                      │
│                                                         │
│  1. Single App per Environment:                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Git                                        │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  environments/                          │   │   │
│  │  │  ├── dev/nginx/ (deployment+service)     │   │   │
│  │  │  ├── staging/nginx/                      │   │   │
│  │  │  └── prod/nginx/                         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  2. App-of-Apps Pattern:                                │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Git                                        │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  root-app.yaml (creates other apps)     │   │   │
│  │  │  apps/                                  │   │   │
│  │  │  ├── nginx-app.yaml                     │   │   │
│  │  │  ├── wordpress-app.yaml                 │   │   │
│  │  │  └── monitoring-app.yaml                │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  3. ApplicationSet Pattern:                             │
│  ┌─────────────────────────────────────────────────┐   │
│  │  ApplicationSet (one to many)                   │   │
│  │  Generates N Applications from template          │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  env: dev → App for dev                  │   │   │
│  │  │  env: staging → App for staging          │   │   │
│  │  │  env: prod → App for prod                │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Pattern 1: Using Helm with ArgoCD

### Helm App Definition

```yaml
# helm-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-helm
  namespace: argocd
spec:
  project: production
  source:
    repoURL: 'https://github.com/yourorg/helm-charts.git'
    path: charts/nginx
    targetRevision: main
    helm:
      valueFiles:
      - values-production.yaml
      parameters:
      - name: replicaCount
        value: "5"
      - name: image.tag
        value: "1.25"
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: production
```

### Using External Helm Charts

```yaml
# external-helm-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prometheus
  namespace: argocd
spec:
  project: production
  source:
    chart: prometheus
    repoURL: https://prometheus-community.github.io/helm-charts
    targetRevision: 25.0.0
    helm:
      values: |
        serviceAccounts:
          server:
            annotations:
              eks.amazonaws.com/role-arn: arn:aws:iam::123:role/prometheus
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 🛠️ Pattern 2: Using Kustomize with ArgoCD

### Kustomize Structure

```
┌─────────────────────────────────────────────────────────┐
│         KUSTOMIZE DIRECTORY STRUCTURE                     │
│                                                         │
│  base/                                                   │
│  ├── deployment.yaml                                     │
│  ├── service.yaml                                        │
│  └── kustomization.yaml                                  │
│                                                         │
│  overlays/                                               │
│  ├── dev/                                                │
│  │  ├── kustomization.yaml (replicas:1, image:dev)       │
│  │  └── configmap-dev.yaml                               │
│  ├── staging/                                            │
│  │  ├── kustomization.yaml (replicas:3, image:staging)   │
│  │  └── configmap-staging.yaml                           │
│  └── production/                                         │
│     ├── kustomization.yaml (replicas:5, image:prod)      │
│     └── configmap-prod.yaml                              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Base Files

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
```

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: myapp:latest
        ports:
        - containerPort: 8080
```

### Overlay Files

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
patches:
- target:
    kind: Deployment
    name: web-app
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
    - op: replace
      path: /spec/template/spec/containers/0/image
      value: myapp:prod-1.25
```

### ArgoCD App with Kustomize

```yaml
# kustomize-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-prod
  namespace: argocd
spec:
  project: production
  source:
    repoURL: 'https://github.com/yourorg/gitops-repo.git'
    path: overlays/production
    targetRevision: main
    kustomize:
      commonLabels:
        environment: production
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 🛠️ Pattern 3: App-of-Apps

### Root Application

```yaml
# root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    repoURL: 'https://github.com/yourorg/gitops-repo.git'
    path: apps
    targetRevision: main
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Child Applications

```yaml
# apps/nginx-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app
  namespace: argocd
spec:
  project: production
  source:
    repoURL: 'https://github.com/yourorg/gitops-repo.git'
    path: environments/production/nginx
    targetRevision: main
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```yaml
# apps/monitoring-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring-app
  namespace: argocd
spec:
  project: production
  source:
    repoURL: 'https://github.com/yourorg/gitops-repo.git'
    path: environments/production/monitoring
    targetRevision: main
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: monitoring
```

---

## 🛠️ Pattern 4: ApplicationSets

### Git Generator

```yaml
# app-set-git.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: git-based-apps
  namespace: argocd
spec:
  goTemplate: true
  generators:
  - git:
      repoURL: 'https://github.com/yourorg/gitops-repo.git'
      revision: main
      directories:
      - path: apps/*/overlays/*
  template:
    metadata:
      name: '{{index .path.segments 1}}-{{index .path.segments 3}}'
    spec:
      project: production
      source:
        repoURL: 'https://github.com/yourorg/gitops-repo.git'
        targetRevision: main
        path: '{{.path.path}}'
      destination:
        server: 'https://kubernetes.default.svc'
        namespace: '{{index .path.segments 3}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### List Generator

```yaml
# app-set-list.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-env-apps
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: dev
        url: https://dev-cluster.example.com
        path: environments/dev
      - cluster: staging
        url: https://staging-cluster.example.com
        path: environments/staging
      - cluster: prod
        url: https://kubernetes.default.svc
        path: environments/production
  template:
    metadata:
      name: '{{cluster}}-web-app'
    spec:
      project: production
      source:
        repoURL: 'https://github.com/yourorg/gitops-repo.git'
        targetRevision: main
        path: '{{path}}'
      destination:
        server: '{{url}}'
        namespace: '{{cluster}}'
```

---

## 📝 Practice: Multi-Environment Deployment

### Step 1: Create Environment Overlays

```bash
# Create directory structure
mkdir -p apps/web-app/{base,overlays/dev,overlays/staging,overlays/production}

# Create base deployment
cat > apps/web-app/base/deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: myapp:latest
EOF

cat > apps/web-app/base/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
EOF
```

### Step 2: Create Kustomize Overlays

```bash
# Dev overlay
cat > apps/web-app/overlays/dev/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
patches:
- target: {kind: Deployment, name: web-app}
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 1
EOF

# Prod overlay
cat > apps/web-app/overlays/production/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
patches:
- target: {kind: Deployment, name: web-app}
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
EOF

git add .
git commit -m "Add web-app with environment overlays"
git push origin main
```

### Step 3: Create Applications

```yaml
# web-app-dev.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-dev
  namespace: argocd
spec:
  project: production
  source:
    repoURL: 'https://github.com/yourorg/gitops-repo.git'
    path: apps/web-app/overlays/dev
    targetRevision: main
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: dev
  syncPolicy:
    automated: {}
```

```yaml
# web-app-prod.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-prod
  namespace: argocd
spec:
  project: production
  source:
    repoURL: 'https://github.com/yourorg/gitops-repo.git'
    path: apps/web-app/overlays/production
    targetRevision: main
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
kubectl apply -f web-app-dev.yaml -f web-app-prod.yaml
```

---

## 📝 Interview Questions

1. **What is the difference between Helm with ArgoCD vs Kustomize with ArgoCD?**
   - **Answer:** Helm uses charts with templating and values; Kustomize uses base+overlays with patches. ArgoCD supports both natively.

2. **What is the App-of-Apps pattern?**
   - **Answer:** A root Application that manages multiple child Applications, enabling deploying multiple apps from one root.

3. **When would you use an ApplicationSet?**
   - **Answer:** When you need to deploy the same app to multiple clusters/environments or generate apps dynamically from a list/git.

4. **How do you manage different configurations per environment?**
   - **Answer:** Use Kustomize overlays (base for common, overlay per env) or Helm values files per environment.

5. **How do you ensure consistency across environments?**
   - **Answer:** Use a single base (Kustomize) or chart (Helm) with environment-specific values, enforced by ArgoCD sync.

---

## 📚 Resources

- [ArgoCD App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [ArgoCD ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/)
- [Kustomize Docs](https://kubectl.docs.kubernetes.io/)

---

## 🎯 Today's Assignment

1. **Create:** Helm and Kustomize application patterns
2. **Build:** App-of-Apps structure
3. **Use:** ApplicationSet for multi-env deployment
4. **Test:** Sync patterns work

---

## 📊 Progress Tracker

```
Day 32: ArgoCD Install & Config ✅
Day 33: ArgoCD Application Patterns ⬜
```

---

## 🔗 Next Day Preview

**Day 34:** ArgoCD Sync & Rollback Strategies
- Sync phases & waves
- Automated sync policies
- Rollback techniques
- Progressive delivery (blue-green, canary)

---

**Remember:** Choose the right pattern for your deployment complexity - simpler is usually better! 🚀
