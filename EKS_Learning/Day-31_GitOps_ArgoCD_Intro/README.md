# 📅 Day 31: GitOps Concepts & ArgoCD Introduction

## 🎯 Learning Objectives
By the end of today, you will understand:
- What is GitOps and why it matters
- GitOps principles
- ArgoCD fundamentals
- Git as single source of truth
- Comparing GitOps vs traditional CI/CD

---

## 📖 Core Concepts

### What is GitOps?

**GitOps** is a way to manage infrastructure and applications using Git as the single source of truth. Git becomes the declarative description of the entire system.

```
┌─────────────────────────────────────────────────────────┐
│              GITOPS PRINCIPLES                            │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              GIT REPOSITORY                       │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  1. Declarative                          │   │   │
│  │  │     Everything described in YAML         │   │   │
│  │  │  2. Versioned                            │   │   │
│  │  │     All changes tracked in history       │   │   │
│  │  │  3. Pull-based Deployment                │   │   │
│  │  │     Agents pull desired state            │   │   │
│  │  │  4. Continuously Reconciled               │   │   │
│  │  │     System converges to Git state        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              GITOPS PRINCIPLES                    │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  1. Git is single source of truth       │   │   │
│  │  │  2. Git is desired state                │   │   │
│  │  │  3. Automation enforces state           │   │   │
│  │  │  4. Change is reviewed via PR           │   │   │
│  │  │  5. Rollback = revert in Git            │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### GitOps vs Traditional CI/CD

```
┌─────────────────────────────────────────────────────────┐
│      GITOPS vs TRADITIONAL CI/CD                         │
│                                                         │
│  Traditional (Push-based):                               │
│  ┌──────┐    ┌──────┐    ┌──────┐                       │
│  │ CI   │ →  │ Push │ →  │ CD   │                       │
│  │ Build│    │ image│    │Deploy│                       │
│  └──────┘    └──────┘    └──────┘                       │
│              (manual,                                   │
│               error-prone)                              │
│                                                         │
│  GitOps (Pull-based):                                    │
│  ┌──────┐    ┌──────┐    ┌──────┐                       │
│  │ Git  │ ←  │ArgoCD│ ←  │Cluster│                       │
│  │ Repo │    │ Agent│    │  ─   │                       │
│  └──────┘    └──────┘    └──────┘                       │
│   (source   (pulls,    (converges                       │
│    of truth)  diffs,     to Git)                        │
│               syncs)                                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🖥️ ArgoCD Architecture

```
┌─────────────────────────────────────────────────────────┐
│              ARGOCD ARCHITECTURE                          │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  User                                           │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Git Push → Git Repo Updated             │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └──────────────┬──────────────────────────────────┘   │
│                 │                                      │
│                 ▼                                      │
│  ┌─────────────────────────────────────────────────┐   │
│  │  ArgoCD (in K8s cluster)                         │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  argocd-server (API + UI)               │   │   │
│  │  │  argocd-repo-server (fetches Git)       │   │   │
│  │  │  argocd-application-controller          │   │   │
│  │  └────────────────────┬────────────────────┘   │   │
│  └───────────────────────┼────────────────────────┘   │
│                          │                             │
│  ┌───────────────────────▼────────────────────────┐   │
│  │  Sync Loop                                      │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  1. Pull desired state from Git         │   │   │
│  │  │  2. Compare with live cluster state     │   │   │
│  │  │  3. Detect drift (if different)         │   │   │
│  │  │  4. Apply changes (sync)                │   │   │
│  │  │  5. Verify healthy state                │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 📝 Hands-On: Setup a GitOps Repository

### Step 1: Structure Your Git Repository

```
├── environments/
│   ├── dev/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   ├── staging/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── production/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
├── apps/
│   ├── nginx/
│   │   └── app.yaml
│   └── wordpress/
│       └── app.yaml
└── argo/
    └── project.yaml
```

### Step 2: Create Application Manifests

```yaml
# environments/dev/nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```

```yaml
# environments/dev/nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: dev
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

### Step 3: Git Flow

```bash
# Initialize repository
git init
git add .
git commit -m "Initial GitOps setup"

# Create branch for development
git checkout -b feature/add-nginx

# Make changes
# Modify image, add resources, etc.

# Commit and push
git add .
git commit -m "Add resource limits to nginx"
git push origin feature/add-nginx

# Create PR and merge after review
```

---

## 📝 Key GitOps Practices

### 1. Atomic Changes
```
✅ BEFORE:                        ❌ AFTER:
- Manual kubectl apply            - Only Git changes
- No audit trail                 - Full audit trail
- Risk of drift                  - No drift
- Human error                    - Automated
```

### 2. Environment Promotion
```
Git Flow:
┌─────────┐   ┌─────────┐   ┌─────────┐
│  Dev    │ → │ Staging │ → │  Prod   │
│  branch │   │ branch  │   │  main   │
└─────────┘   └─────────┘   └─────────┘
   ↑ merge       ↑ merge       ↑ (stable)
```

### 3. Rollback Strategy
```
Rollback = Git Revert:
1. Identify bad commit (e.g., abc123)
2. Revert: git revert abc123
3. Push: git push origin main
4. ArgoCD auto-syncs to previous state
```

---

## 📝 Interview Questions

1. **What is GitOps?**
   - **Answer:** An operating model using Git as the single source of truth for declarative infrastructure and applications, with automated reconciliation.

2. **What are the 4 GitOps principles?**
   - **Answer:** Declarative, versioned & immutable, pulled automatically, continuously reconciled.

3. **How is GitOps different from CI/CD?**
   - **Answer:** GitOps is pull-based (ArgoCD polls Git), while traditional CI/CD is push-based (CI pushes to cluster). GitOps gives auditability and rollback.

4. **What is an ArgoCD Application?**
   - **Answer:** A Kubernetes CRD that defines what to deploy (source repo, path) and where (destination cluster/namespace).

5. **Why use ArgoCD over kubectl?**
   - **Answer:** ArgoCD provides Git-based change management, drift detection, automated sync, rollback, and audit trail.

---

## 📚 Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [GitOps Principles](https://www.weave.works/technologies/gitops/)

---

## 🎯 Today's Assignment

1. **Create:** A Git repository with environment structure
2. **Write:** Application manifests for dev/staging/prod
3. **Practice:** Git flow (branch, PR, merge)
4. **Understand:** Why GitOps is preferred

---

## 📊 Progress Tracker

```
Day 30: Advanced IAM Practice ✅
Day 31: GitOps Concepts & ArgoCD Intro ⬜
```

---

## 🔗 Next Day Preview

**Day 32:** ArgoCD Installation & Configuration
- Install ArgoCD on EKS
- Configure Git repository
- Set up projects and applications

---

**Remember:** GitOps turns Git into a versioned, auditable deployment engine! 🚀
