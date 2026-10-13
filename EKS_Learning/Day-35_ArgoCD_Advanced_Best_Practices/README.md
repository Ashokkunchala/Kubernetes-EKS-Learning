# 📅 Day 35: ArgoCD Advanced & Best Practices

## 🎯 Learning Objectives
By the end of today, you will understand:
- ArgoCD RBAC and SSO
- Multi-cluster management
- Monitoring and alerts for ArgoCD
- Backup and restore for ArgoCD
- Production best practices

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **AppProject** | An Argo CD boundary that can restrict permitted repositories, destinations, and resource kinds for Applications. |
| **RBAC** | Role-Based Access Control: permissions that map identities/groups to allowed actions. |
| **Sync policy** | Settings controlling whether and how Argo CD applies changes automatically or manually. |
| **Health status** | Argo CD's interpretation of whether a resource appears healthy according to its resource checks. |
| **Multi-tenancy** | Sharing a platform among teams while separating their permissions, resources, and operational boundaries. |
| **Secret management** | Safely providing credentials without committing plaintext secrets to source control. |

**Analogy:** AppProject is a fenced work area, RBAC is the badge/access list, and sync policy determines who can approve changes automatically.

**Try / observe:** Review project source/destination restrictions, user roles, repository credentials, and automated sync settings before onboarding another team.

**Troubleshooting clue:** An Application may be blocked by project policy even when its YAML is valid. Inspect the Application's conditions and the AppProject allow-lists.

**Explain it back:** How does AppProject reduce risk? Why should repo access and cluster destination permissions both be restricted?

---

## 📖 Core Concepts

### ArgoCD Security Model

```
┌─────────────────────────────────────────────────────────┐
│          ARGOCD SECURITY ARCHITECTURE                     │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Authentication (SSO via Dex)                    │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Local Users (admin)                    │   │   │
│  │  │  OIDC Providers (Okta, Auth0, AWS)      │   │   │
│  │  │  LDAP/AD                                 │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Authorization (RBAC)                            │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Roles: admin, developers, viewers      │   │   │
│  │  │  Projects: dev, staging, prod           │   │   │
│  │  │  Policies: get, sync, create, delete    │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  Example Policy:                               │   │
│  │  p, role:prod-deployer, applications, sync,   │   │
│  │    prod/*, allow                              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  🛡️ Multi-Cluster:                                     │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Control Plane (ArgoCD)                         │   │
│  │  ├── Cluster A (prod)                           │   │
│  │  ├── Cluster B (staging)                        │   │
│  │  └── Cluster C (dev)                            │   │
│  │  (Manage all from one ArgoCD)                   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ ArgoCD RBAC Configuration

### Step 1: Configure RBAC in ConfigMap

```yaml
# argocd-rbac-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-rbac-cm
data:
  policy.csv: |
    # Admin role - full access
    p, role:admin, *, *, *, allow

    # Dev role - manage dev apps only
    p, role:dev, applications, get, dev/*, allow
    p, role:dev, applications, sync, dev/*, allow
    p, role:dev, applications, update, dev/*, allow

    # Staging role
    p, role:staging, applications, get, staging/*, allow
    p, role:staging, applications, sync, staging/*, allow

    # Prod role - production deployers
    p, role:prod-deployer, applications, get, prod/*, allow
    p, role:prod-deployer, applications, sync, prod/*, allow

    # Viewer - read only
    p, role:viewer, applications, get, *, allow

    # Map roles to groups
    g, admin-group, role:admin
    g, dev-group, role:dev
    g, prod-deployers, role:prod-deployer
    g, developers, role:viewer

  policy.default: role:viewer
```

### Step 2: SSO with OIDC (e.g., Okta/Auth0)

```yaml
# argocd-cm.yaml (SSO configuration)
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
data:
  url: https://argocd.example.com
  dex.config: |
    connectors:
    - type: oidc
      id: okta
      name: Okta
      config:
        issuer: https://yourorg.okta.com
        clientID: <client-id>
        clientSecret: $dex.okta.clientSecret
        insecureEnableGroups: true
        userIDClaim: sub
        userNameClaim: email
        groupsNameClaim: groups
```

### Step 3: Projects with RBAC

```yaml
# projects-rbac.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production environment
  sourceRepos:
  - 'https://github.com/yourorg/gitops-repo.git'
  destinations:
  - namespace: production
    server: https://kubernetes.default.svc
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  roles:
  - name: project-admin
    description: Can manage project apps
    policies:
    - p, proj:production:project-admin, applications, *, production/*, allow
    groups:
    - proj-production-ops
  - name: developer
    description: Can view and sync
    policies:
    - p, proj:production:developer, applications, get, production/*, allow
    - p, proj:production:developer, applications, sync, production/*, allow
    groups:
    - proj-production-devs
```

---

## 🛠️ Multi-Cluster Management

### Step 1: Register Remote Clusters

```bash
# List current contexts
kubectl config get-contexts

# Add remote cluster
argocd cluster add <remote-context-name>

# Add cluster with specific name
argocd cluster add dev-cluster --name dev-cluster

# Verify clusters
argocd cluster list

# SERVER NAME STATUS MESSAGE
# https://kubernetes.default.svc  Successful
# https://dev-cluster.example.com  Successful
# https://staging-cluster.example.com  Successful
```

### Step 2: Deploy to Multiple Clusters

```yaml
# multi-cluster-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-dev
  namespace: argocd
spec:
  project: dev
  source:
    repoURL: 'https://github.com/yourorg/gitops-repo.git'
    path: apps/web-app
    targetRevision: main
  destination:
    server: https://dev-cluster.example.com
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Step 3: ApplicationSet for Multi-Cluster

```yaml
# cluster-appset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-apps
  namespace: argocd
spec:
  goTemplate: true
  generators:
  - clusters:
      selector:
        matchLabels:
          environment: production
  template:
    metadata:
      name: '{{name}}-web-app'
    spec:
      project: default
      source:
        repoURL: 'https://github.com/yourorg/gitops-repo.git'
        path: 'environments/{{name}}/web-app'
        targetRevision: main
      destination:
        server: '{{server}}'
        namespace: web-app
```

---

## 🛠️ Monitoring ArgoCD

### Metrics with Prometheus

```yaml
# argo-metrics.yaml
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: argocd
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-server
  endpoints:
  - port: metrics
    interval: 30s
  namespaceSelector:
    matchNames:
    - argocd
```

### Key Metrics to Monitor

```
ArgoCD Metrics:
┌─────────────────────────────────────────────────────────┐
│  Application Health:                                    │
│    argocd_app_info{health_status="Healthy"}             │
│    argocd_app_info{health_status="Degraded"}            │
│    argocd_app_info{health_status="Missing"}             │
│                                                         │
│  Sync Status:                                           │
│    argocd_app_info{sync_status="Synced"}                │
│    argocd_app_info{sync_status="OutOfSync"}             │
│                                                         │
│  Cluster Status:                                        │
│    argocd_cluster_info{server_status="Successful"}      │
│                                                         │
│  Controller Operations:                                 │
│    argocd_app_controller_app_info                      │
│    argocd_app_controller_runtime_reconcile             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Alert Rules

```yaml
# argocd-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: argocd-alerts
  namespace: monitoring
spec:
  groups:
  - name: argocd
    rules:
    - alert: ArgoCDAppOutOfSync
      expr: argocd_app_info{sync_status="OutOfSync"} > 0
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Application {{ $labels.name }} is OutOfSync"
    - alert: ArgoCDAppDegraded
      expr: argocd_app_info{health_status="Degraded"} > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Application {{ $labels.name }} is Degraded"
    - alert: ArgoCDClusterDown
      expr: argocd_cluster_info{server_status!="Successful"} > 0
      for: 5m
      labels:
        severity: critical
```

---

## 🛠️ Backup & Restore ArgoCD

### Backup Applications and Settings

```bash
# List all applications
kubectl get applications -n argocd -o yaml > argocd-apps-backup.yaml

# List all projects
kubectl get appprojects -n argocd -o yaml > argocd-projects-backup.yaml

# Backup ConfigMaps
kubectl get cm argocd-cm argocd-rbac-cm -n argocd -o yaml > argocd-config-backup.yaml

# Backup Secrets (carefully - contains repo creds)
kubectl get secrets -n argocd -l 'app.kubernetes.io/managed-by=argocd.argoproj.io' -o yaml > argocd-secrets-backup.yaml
```

### Automate with CronJob

```yaml
# argocd-backup-cron.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: argocd-backup
  namespace: argocd
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: argocd-backup-sa
          containers:
          - name: backup
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              kubectl get applications -n argocd -o yaml > /backup/apps.yaml && \
              kubectl get appprojects -n argocd -o yaml > /backup/projects.yaml && \
              kubectl get cm argocd-cm argocd-rbac-cm -n argocd -o yaml > /backup/config.yaml
          restartPolicy: OnFailure
```

---

## 📝 Best Practices Checklist

### Repository Structure
- [ ] Use a dedicated gitops repo
- [ ] Branch protection for main
- [ ] PR-based changes with review
- [ ] Use Kustomize/Helm for DRY configs

### Security
- [ ] SSO integration (no shared admin)
- [ ] RBAC per team/environment
- [ ] Least privilege for repo access
- [ ] Audit logs enabled

### Reliability
- [ ] Multiple replicas for ArgoCD server/controller
- [ ] Automated sync with selfHeal + prune
- [ ] Backup ArgoCD config regularly
- [ ] Monitor app health and sync status

### Performance
- [ ] Set resource limits on ArgoCD components
- [ ] Use repo-server caching
- [ ] Configure concurrency limits
- [ ] Use selective sync options

---

## 📝 Interview Questions

1. **How do you secure ArgoCD?**
   - **Answer:** SSO, RBAC per project/team, least privilege, audit logging, and restricting which repos/clusters/namespaces apps can target.

2. **How do you manage multiple clusters with ArgoCD?**
   - **Answer:** Register all clusters with `argocd cluster add` and use ApplicationSets to generate apps per cluster.

3. **How do you back up ArgoCD?**
   - **Answer:** Export Applications, AppProjects, ConfigMaps, and Secrets; schedule automated backups with CronJob.

4. **What metrics should you alert on?**
   - **Answer:** Apps OutOfSync, Apps Degraded, cluster connectivity, reconcile errors, and sync failures.

5. **What's your recommended ArgoCD production setup?**
   - **Answer:** HA install (multiple server/controller replicas), SSO + RBAC, automated sync with selfHeal+prune, monitoring with alert rules, and a backup CronJob.

---

## 📚 Resources for Review

- [Day 31: GitOps Concepts](Day-31_GitOps_ArgoCD_Intro/README.md)
- [Day 32: ArgoCD Install](Day-32_ArgoCD_Installation_Config/README.md)
- [Day 33: App Patterns](Day-33_ArgoCD_Application_Patterns/README.md)
- [Day 34: Sync & Rollback](Day-34_ArgoCD_Sync_Rollback/README.md)

### External
- [ArgoCD RBAC](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/)
- [ArgoCD Metrics](https://argo-cd.readthedocs.io/en/stable/operator-manual/metrics/)

---

## 🎯 Today's Assignment

1. **Configure:** RBAC for multiple teams
2. **Add:** A second cluster to ArgoCD
3. **Set up:** Monitoring alerts
4. **Backup:** ArgoCD configuration
5. **Review:** Best practices checklist

---

## 📊 Progress Tracker

```
Week 9-10 ArgoCD Complete:
Day 31: GitOps & ArgoCD Intro ✅
Day 32: ArgoCD Install & Config ✅
Day 33: App Patterns ✅
Day 34: Sync & Rollback ✅
Day 35: ArgoCD Advanced & Best Practices ⬜
```

---

## 🔗 Next Week Preview

**Week 11-12:** CI/CD Pipelines
- Day 36: CI/CD Concepts & AWS CodePipeline
- Day 37: GitHub Actions for EKS
- Day 38: Jenkins on EKS
- Day 39: CI/CD Practice - End-to-End Pipeline
- Day 40: CI/CD Interview Preparation

---

**Remember:** A secure, monitored, backed-up ArgoCD is a reliable GitOps platform! 🚀
