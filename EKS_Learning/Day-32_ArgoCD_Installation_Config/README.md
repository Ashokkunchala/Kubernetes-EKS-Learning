# 📅 Day 32: ArgoCD Installation & Configuration

## 🎯 Learning Objectives
By the end of today, you will:
- Install ArgoCD on EKS
- Configure Git repository connection
- Set up projects and applications
- Access the ArgoCD UI
- Deploy your first application via GitOps

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Argo CD** | A GitOps continuous-delivery tool that watches Git and reconciles Kubernetes resources. |
| **repo-server** | Argo CD component that retrieves repositories and renders source configuration such as Helm/Kustomize. |
| **application controller** | Component that compares desired and live resources and performs reconciliation. |
| **Application** | An Argo CD custom resource describing a source repository/path and a destination cluster/Namespace. |
| **Namespace** | A logical grouping of namespaced Kubernetes objects. |
| **Repository credentials** | Credentials Argo CD uses to read a private source repository. Store and scope them carefully. |

**Analogy:** repo-server fetches and prepares the blueprint; the controller checks the live building and coordinates changes.

**Try / observe:** `kubectl get pods -n argocd`, `kubectl get applications -n argocd`, `kubectl describe application <app-name> -n argocd`.

**Troubleshooting clue:** If an Application cannot load its source, check repository URL/credentials and rendering errors. If source loads but sync fails, check destination access, RBAC, and resource validation.

**Explain it back:** What is the responsibility of repo-server versus the application controller?

---

## 📖 Core Concepts

### ArgoCD Components

```
┌─────────────────────────────────────────────────────────┐
│              ARGOCD COMPONENTS                            │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  argocd-server                                   │   │
│  │  • API server (REST/gRPC)                        │   │
│  │  • Web UI                                        │   │
│  │  • CLI backend                                   │   │
│  │  • Handles auth, RBAC, proxy to others           │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  argocd-repo-server                              │   │
│  │  • Connects to Git repos (HTTPS/SSH)             │   │
│  │  • Caches repository contents                    │   │
│  │  • Generates manifests (Helm/Kustomize)          │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  argocd-application-controller                   │   │
│  │  • Monitors applications                         │   │
│  │  • Compares desired vs live state                │   │
│  │  • Performs sync operations                      │   │
│  │  • Handles health checks                         │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  argocd-dex-server                               │   │
│  │  • Authentication (SSO)                          │   │
│  │  • OAuth/OIDC providers                          │   │
│  │  • Local users (admin)                           │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Hands-On: Install ArgoCD

### Step 1: Create Namespace

```bash
kubectl create namespace argocd
```

### Step 2: Install ArgoCD

```bash
# Option 1: Using manifest (production)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Option 2: Using Helm (recommended for customization)
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd \
  --namespace argocd \
  --set server.service.type=LoadBalancer \
  --set controller.replicas=1 \
  --set server.replicas=1 \
  --set repoServer.replicas=1
```

### Step 3: Verify Installation

```bash
# Check pods
kubectl get pods -n argocd
# Expected: argocd-server, argocd-repo-server, argocd-application-controller, argocd-redis, argocd-dex-server

# Check services
kubectl get svc -n argocd

# Get the load balancer URL
kubectl get svc argocd-server -n argocd -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### Step 4: Get Admin Password

```bash
# Get initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d; echo

# If using Helm, set your own password via argocd CLI
kubectl port-forward svc/argocd-server -n argocd 8080:443
argocd login localhost:8080 --username admin
# Enter the password at the interactive prompt; do not put it in the command.
argocd account update-password
```

### Step 5: Access ArgoCD UI

```bash
# Port forward to access UI locally
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Open in browser: https://localhost:8080
# Username: admin
# Password: (from step 4)
```

### Step 6: Install ArgoCD CLI

```bash
# Windows (use choco or download)
choco install argocd-cli -y

# Or download directly
Invoke-WebRequest -Uri https://github.com/argoproj/argo-cd/releases/download/v2.9.0/argocd-windows-amd64.exe -OutFile argocd.exe

# Verify
argocd version
```

---

## 📝 Configure ArgoCD

### Step 1: Login via CLI

> **Exposure warning:** The Helm example below sets the Argo CD Service to LoadBalancer, which can create an externally reachable, billable AWS load balancer depending on subnet/controller configuration. For a learning lab, prefer port-forwarding; for shared/production use, configure private access, trusted TLS, authentication, and network restrictions.

```bash
# Port forward
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Login
# --insecure skips TLS certificate verification; use only for this local port-forward lab.
# In production, configure a trusted TLS certificate and omit --insecure.
argocd login localhost:8080 \
  --username admin \
  --insecure
# Enter the password at the interactive prompt.
```

### Step 2: Add Git Repository

```bash
# Set GITHUB_USER and GITHUB_TOKEN securely in your shell; never commit a real token.
# Prefer a least-privilege, short-lived token or a declarative Argo CD repository Secret.
argocd repo add https://github.com/yourorg/gitops-repo.git \
  --username "$GITHUB_USER" \
  --password "$GITHUB_TOKEN"

# Add via CLI (SSH)
argocd repo add git@github.com:yourorg/gitops-repo.git \
  --ssh-private-key-path ~/.ssh/id_rsa

# Verify repositories
argocd repo list
```

### Step 3: Add Cluster

```bash
# ArgoCD automatically adds the local cluster
# Verify
argocd cluster list

# Add a remote cluster (for multi-cluster)
argocd cluster add <context-name>
```

### Step 4: Create AppProject

```yaml
# project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production Applications
  sourceRepos:
  - 'https://github.com/yourorg/gitops-repo.git'
  destinations:
  - namespace: '*'
    server: https://kubernetes.default.svc
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  - group: apps
    kind: Deployment
  - group: ''
    kind: Service
  roles:
  - name: deployer
    description: Can sync applications
    policies:
    - p, proj:production:deployer, applications, sync, production/*, allow
    groups:
    - production-deployers
```

```bash
kubectl apply -f project.yaml
```

### Step 5: Create First Application

```yaml
# app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
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
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
```

```bash
kubectl apply -f app.yaml

# Verify via CLI
argocd app get nginx
argocd app sync nginx
```

---

## 📝 Create App from UI

### Step 1: Access UI
```
1. Open browser: https://localhost:8080
2. Login with admin
3. Click "New App"
```

### Step 2: Fill Application Details
```
General:
  Application Name: my-app
  Project: production
  Sync Policy: Automatic
  Sync Options: Auto-Create Namespace, Prune

Source:
  Repository URL: https://github.com/yourorg/gitops-repo.git
  Revision: main
  Path: environments/dev/my-app

Destination:
  Cluster URL: https://kubernetes.default.svc
  Namespace: dev
```

### Step 3: Click Create & Sync

---

## 📝 Practice Lab: Deploy via GitOps

### Step 1: Update Git Repository

```yaml
# environments/production/nginx/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: production
spec:
  replicas: 3
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
        image: nginx:1.25
        ports:
        - containerPort: 80
```

```bash
git add .
git commit -m "Update nginx to 1.25, scale to 3 replicas"
git push origin main
```

### Step 2: Watch ArgoCD Auto-Sync

```bash
# See application status
kubectl get applications -n argocd
kubectl get application nginx -o yaml | grep -A5 'status:'

# Or via CLI
argocd app get nginx
argocd app sync nginx --async

# Check deployment
kubectl get deployments -n production
kubectl get pods -n production
```

---

## 🐛 Troubleshooting ArgoCD

### Common Issues

```bash
# 1. ArgoCD can't access Git repo
# Check repository status
argocd repo list
# Re-add repo with correct credentials

# 2. Application stuck in "OutOfSync"
# Check diff
argocd app diff nginx
kubectl get application nginx -o yaml | grep conditions

# 3. Sync failed
# Check events
kubectl describe application nginx -n argocd
# Check repo-server logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server

# 4. Server not responding
# Check all pods running
kubectl get pods -n argocd

# 5. Can't login
# Reset password
kubectl delete secret argocd-initial-admin-secret -n argocd
kubectl patch secret argocd-secret -n argocd -p '{"data":{"admin.password":null,"admin.passwordMtime":null}}'
```

---

## 📝 Interview Questions

1. **What are the main ArgoCD components?**
   - **Answer:** Server (API/UI), Repo Server (Git access), Application Controller (sync/health), Dex (auth).

2. **How do you install ArgoCD?**
   - **Answer:** Via Helm chart or kubectl with the install manifests, into the argocd namespace.

3. **What is an AppProject in ArgoCD?**
   - **Answer:** A logical grouping that defines which repositories, clusters, and namespaces applications can access, plus RBAC.

4. **What is the difference between an Application and AppProject?**
   - **Answer:** Application defines what to deploy and where; AppProject sets boundaries/security for groups of applications.

5. **How do you add a Git repository to ArgoCD?**
   - **Answer:** Via CLI (`argocd repo add`), via UI (Settings > Repositories), or declaratively via a Secret.

---

## 📚 Resources

- [ArgoCD Install Guide](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [ArgoCD Architecture](https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture/)
- [ArgoCD Helm Chart](https://artifacthub.io/packages/helm/argo/argo-cd)

---

## 🎯 Today's Assignment

1. **Install:** ArgoCD on your EKS cluster
2. **Configure:** Git repository connection
3. **Create:** An AppProject
4. **Deploy:** Your first application via GitOps
5. **Verify:** Auto-sync works end-to-end

---

## 📊 Progress Tracker

```
Day 31: GitOps Concepts ✅
Day 32: ArgoCD Installation & Config ⬜
```

---

## 🔗 Next Day Preview

**Day 33:** ArgoCD Application Patterns
- Multi-environment deploys
- Helm and Kustomize with ArgoCD
- Application sets patterns
- Sync & rollback strategies

---

**Remember:** ArgoCD automates the "Git to Cluster" pipeline with visibility and auditability! 🚀
