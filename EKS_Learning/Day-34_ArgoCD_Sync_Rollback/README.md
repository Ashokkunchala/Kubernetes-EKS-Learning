# 📅 Day 34: ArgoCD Sync & Rollback Strategies

## 🎯 Learning Objectives
By the end of today, you will understand:
- Sync phases and waves
- Automated vs manual sync
- Rollback strategies
- Progressive delivery (blue-green, canary)
- Argo Rollouts for advanced deployments

---

## 📖 Core Concepts

### ArgoCD Sync Process

```
┌─────────────────────────────────────────────────────────┐
│              ARGOCD SYNC PHASES & WAVES                   │
│                                                         │
│  Sync Phase 1 - PreSync:                                │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Hook: PreSync (before main resources)          │   │
│  │  Example: Run DB migrations                     │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│  Sync Phase 2 - Sync:                                  │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Wave 1: ConfigMaps, Secrets (0)                │   │
│  │  Wave 2: Services (1)                           │   │
│  │  Wave 3: Deployments (2)                        │   │
│  │  Wave 4: Ingress (3)                            │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│  Sync Phase 3 - PostSync:                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Hook: PostSync (after main resources)           │   │
│  │  Example: Run smoke tests                       │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Sync Phase 4 - SyncFail (on failure):                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Hook: SyncFail (if sync fails)                │   │
│  │  Example: Send alert, rollback                 │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Sync Phases & Waves Order

```
┌─────────────────────────────────────────────────────────┐
│              SYNC ORDER                                    │
│                                                         │
│  Phase: PreSync                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  SyncWave 0: DB migrations                      │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Phase: Sync                                            │
│  ┌─────────────────────────────────────────────────┐   │
│  │  SyncWave -10: CRDs                              │   │
│  │  SyncWave -5: Namespaces                         │   │
│  │  SyncWave 0: ConfigMaps, Secrets                 │   │
│  │  SyncWave 1: Services                            │   │
│  │  SyncWave 2: Deployments, StatefulSets           │   │
│  │  SyncWave 3: Ingress, HPA                        │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Phase: PostSync                                        │
│  ┌─────────────────────────────────────────────────┐   │
│  │  SyncWave 0: Smoke tests, notifications          │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Example: Migration Hook

```yaml
# migration-job.yaml (PreSync hook)
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: myapp:migrate-1.0
        command: ["alembic", "upgrade", "head"]
      restartPolicy: Never
  backoffLimit: 1
```

```yaml
# smoke-test.yaml (PostSync hook)
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-test
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: test
        image: curlimages/curl
        command: ["curl", "-f", "http://myapp-service/health"]
      restartPolicy: Never
```

---

## 🛠️ Sync Configurations

### Manual Sync (Controlled)

```yaml
# manual-sync-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: manual-sync
  namespace: argocd
spec:
  project: production
  source:
    repoURL: 'https://github.com/yourorg/gitops-repo.git'
    path: environments/production
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: production
  # NO syncPolicy - manual only
```

```bash
# Manual sync
argocd app sync manual-sync

# Sync with specific revision
argocd app sync manual-sync --revision <commit-hash>

# Sync specific resources
argocd app sync manual-sync --resource apps:Deployment:nginx
```

### Automated Sync with Prune & SelfHeal

```yaml
# automated-sync-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: automated-sync
  namespace: argocd
spec:
  project: production
  source:
    repoURL: 'https://github.com/yourorg/gitops-repo.git'
    path: environments/production
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: production
  syncPolicy:
    automated:
      prune: true        # Remove resources no longer in Git
      selfHeal: true     # Revert manual changes to Git state
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - PruneLast=true     # Prune after sync instead of before
    - ApplyOutOfSyncOnly=true
```

### Sync Retry Policy

```yaml
# sync-retry.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sync-retry
  namespace: argocd
spec:
  syncPolicy:
    automated: {}
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

---

## 🛠️ Rollback Techniques

### Method 1: Rollback via ArgoCD CLI

```bash
# View sync history
argocd app history myapp

# Output:
# ID  DATE  REVISION
# 1    Nov 5  abc123
# 2    Nov 5  def456
# 3    Nov 6  ghi789  (current)

# Rollback to revision 2
argocd app rollback myapp 2

# Verify
argocd app get myapp
```

### Method 2: Rollback via Git (Preferred GitOps way)

```bash
# Find the bad commit
git log --oneline -10

# Revert the bad commit
git revert ghi789
git push origin main

# ArgoCD auto-syncs back to good state
```

### Method 3: Update Application Revision

```yaml
# Point application to a specific revision
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  source:
    repoURL: 'https://github.com/yourorg/gitops-repo.git'
    path: environments/production
    targetRevision: def456   # Specific commit
```

---

## 🛠️ Progressive Delivery with Argo Rollouts

### Install Argo Rollouts

```bash
# Install controller
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Install CLI
# Windows download
Invoke-WebRequest https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-windows-amd64 -OutFile kubectl-argo-rollouts.exe
```

### Blue-Green Deployment

```yaml
# blue-green-rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 5
  revisionHistoryLimit: 3
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
        image: myapp:green
  strategy:
    blueGreen:
      activeService: web-app-active
      previewService: web-app-preview
      autoPromotionEnabled: false
```

```yaml
# services.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-active
  labels:
    app: web-app
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-preview
  labels:
    app: web-app
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
```

```bash
# Watch rollout
kubectl argo rollouts get rollout web-app -n production --watch

# Promote blue to green after review
kubectl argo rollouts promote web-app -n production

# Abort if issues
kubectl argo rollouts abort web-app -n production
```

### Canary Deployment

```yaml
# canary-rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: web-app-canary
  namespace: production
spec:
  replicas: 10
  selector:
    matchLabels:
      app: web-app-canary
  template:
    metadata:
      labels:
        app: web-app-canary
    spec:
      containers:
      - name: web-app
        image: myapp:2.0
        ports:
        - containerPort: 8080
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 5m}
      - setWeight: 30
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 80
      - pause: {duration: 5m}
      - setWeight: 100
```

```bash
# Watch canary rollout
kubectl argo rollouts get rollout web-app-canary -n production --watch

# Promote (skip remaining pauses)
kubectl argo rollouts promote web-app-canary -n production

# Full promotion
kubectl argo rollouts promote web-app-canary -n production --full

# Abort
kubectl argo rollouts abort web-app-canary -n production
```

---

## 📝 Practice Lab: End-to-End Sync & Rollback

### Scenario: Deploy with Sync Order, Then Rollback

```yaml
# Step 1: Create app with hooks (in Git repo)
# order: migration (PreSync) → config → deployment → smoke test (PostSync)
```

```yaml
# Step 2: Deploy
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-app
  namespace: argocd
spec:
  project: production
  source:
    repoURL: 'https://github.com/yourorg/gitops-repo.git'
    path: environments/production/order-app
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
# Apply
kubectl apply -f order-app.yaml

# Watch sync
argocd app sync order-app
kubectl get jobs -n production  # See migration & smoke test
kubectl get pods -n production  # See deployment

# Step 3: Break it - make a bad commit
git commit -m "BAD: broken image tag"
git push origin main

# Watch selfheal try & fail
argocd app get order-app  # OutOfSync/Error

# Step 4: Rollback via Git
git revert BADCOMMIT
git push origin main

# ArgoCD self-heals back to working state
```

---

## 📝 Interview Questions

1. **What are sync phases and waves in ArgoCD?**
   - **Answer:** Phases (PreSync, Sync, PostSync, SyncFail) group resources; waves define order within phases using annotations.

2. **What is self-healing in ArgoCD?**
   - **Answer:** ArgoCD continuously reconciles; if someone changes resources manually, it reverts them to match Git.

3. **How do you do a blue-green deployment?**
   - **Answer:** Two versions (blue=old, green=new) run simultaneously; traffic stays on blue until green is verified, then switches.

4. **How do you do a canary deployment?**
   - **Answer:** Roll out new version to small % of traffic, gradually increase, with pauses for monitoring between steps.

5. **What's the difference between rollback via ArgoCD CLI vs Git?**
   - **Answer:** CLI `argocd app rollback` reverts the app state; Git revert is the GitOps-native way that keeps everything in Git.

---

## 📚 Resources

- [ArgoCD Sync Docs](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/)
- [Argo Rollouts](https://argoproj.github.io/argo-rollouts/)
- [Sync Hooks](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-hooks/)

---

## 🎯 Today's Assignment

1. **Set up:** Sync waves with DB migration + smoke test hooks
2. **Implement:** Both manual and automated sync
3. **Practice:** Rollback via CLI and via Git
4. **Deploy:** Blue-green rollout
5. **Deploy:** Canary rollout with incremental steps

---

## 📊 Progress Tracker

```
Day 33: ArgoCD App Patterns ✅
Day 34: ArgoCD Sync & Rollback ⬜
```

---

## 🔗 Next Day Preview

**Day 35:** ArgoCD Advanced & Best Practices
- RBAC for ArgoCD
- Multi-cluster management
- Monitoring and alerts
- Real-world patterns & interview prep

---

**Remember:** Controlled sync + easy rollback = safe, confident deployments! 🚀
