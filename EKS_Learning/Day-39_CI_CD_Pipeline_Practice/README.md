# 📅 Day 39: CI/CD Pipeline Practice - End-to-End Pipeline

## 🎯 Learning Objectives
By the end of today, you will:
- Build a complete, working CI/CD pipeline for EKS
- Combine GitHub Actions + ECR + ArgoCD into one flow
- Implement automated tests, security scans, and rollbacks
- Apply GitOps for continuous deployment
- Validate the entire pipeline with a real code change

---
## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Build** | Turn source code into a runnable package or image. |
| **Test** | Check whether the code behaves as expected. |
| **Scan** | Check code/dependencies/images for known vulnerabilities or policy violations. |
| **Push** | Upload an image or artifact to a registry/repository. |
| **Deploy** | Make a chosen version run in the target environment. |
| **Rollback** | Restore a previous known-good version when a release causes problems. |

**Analogy:** A release line builds a product, inspects it, stores the approved package, and installs a version; each stage should produce evidence.

**Try / observe:** Record the source commit, immutable image digest, test/scan results, deployment revision, and rollback command for one practice release.

**Troubleshooting clue:** If the deployed code does not match the source commit, inspect image tags/digests and pipeline artifact handoff rather than assuming the build step used the intended revision.

**Explain it back:** Why is an image digest more reliable for identifying an image than a mutable tag like `latest`?

---


## 📖 Project: Full Pipeline with GitHub Actions + ArgoCD

This is the industry-standard pattern we'll build hands-on:

```
┌────────────┐   ┌──────────┐   ┌──────────┐   ┌────────────┐
│ Developer  │──▶│ GitHub   │──▶│ ECR      │──▶│ GitOps Repo│
│ pushes     │   │ Actions  │   │ (image)  │   │ (manifest) │
└────────────┘   └────┬─────┘   └──────────┘   └─────┬──────┘
                      │                              │
                      │ (build+test+scan)            │ (auto-sync)
                      ▼                              ▼
                 ┌──────────┐                 ┌────────────┐
                 │  Tests   │                 │   ArgoCD   │
                 │  passed  │                 │ reconciles │
                 └──────────┘                 └─────┬──────┘
                                                    ▼
                                              ┌────────────┐
                                              │  EKS       │
                                              │  Cluster   │
                                              └────────────┘
```

---

## 🛠️ Lab Setup (Prerequisites)

```bash
# 1. EKS cluster (if not present)
eksctl create cluster --name pipelines-demo --region us-east-1 \
  --nodegroup-name standard --node-type m5.large --nodes 2

# 2. ECR repos
aws ecr create-repository --repository-name demo-app
aws ecr create-repository --repository-name gitops-repo

# 3. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 4. Set up OIDC provider + deploy role (Day 37 covered this)
eksctl utils associate-iam-oidc-provider --cluster pipelines-demo --approve
```

---

## 🛠️ Step 1: Application Repo

Create `demo-app/` with a simple image and CI config:

```yaml
# demo-app/.github/workflows/ci.yml
name: CI - Build, Test, Scan
on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Unit tests
        run: |
          go test ./...   # or npm test, pytest, etc.
      - name: Build image
        run: docker build -t demo-app:ci -f Dockerfile .
      - name: Security scan (Trivy)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: demo-app:ci
          format: table
          exit-code: '1'
          ignore-unfixed: true
          severity: CRITICAL,HIGH
```

> **Gate:** If Trivy finds CRITICAL/HIGH vulnerabilities, the pipeline fails — no deploy.

---

## 🛠️ Step 2: Release Job (build + push + update GitOps)

```yaml
# demo-app/.github/workflows/release.yml (same file, separate job)
release:
  needs: ci
  runs-on: ubuntu-latest
  environment: production
  if: github.ref == 'refs/heads/main'
  permissions:
    id-token: write
    contents: write
    pull-requests: write
  steps:
    - uses: actions/checkout@v4

    - name: Configure AWS (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::ACCOUNT_ID:role/github-actions-deploy
        aws-region: us-east-1

    - name: Login ECR & push
      id: ecr
      uses: aws-actions/amazon-ecr-login@v2
    - name: Build & push
      env:
        IMAGE_TAG: ${{ github.sha }}
      run: |
        IMAGE=${{ steps.ecr.outputs.registry }}/demo-app:$IMAGE_TAG
        docker build -t $IMAGE .
        docker push $IMAGE
        echo "image=$IMAGE" >> $GITHUB_OUTPUT

    - name: Update GitOps manifest
      run: |
        git clone https://${{ secrets.GH_TOKEN }}@github.com/yourorg/gitops-repo.git
        cd gitops-repo
        sed -i "s|image:.*|image: ${{ steps.ecr.outputs.image }}|" environments/prod/demo-app/deployment.yaml
        git config user.name "CI Bot"
        git config user.email "ci@example.com"
        git add .
        git commit -m "release demo-app ${{ github.sha }}"
        git push origin main
```

> **Key:** CI updates the **GitOps repo**, NOT the cluster directly. ArgoCD notices the change and syncs.

---

## 🛠️ Step 3: GitOps Repo + ArgoCD Application

```yaml
# gitops-repo/environments/prod/demo-app/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourorg/gitops-repo.git
    path: environments/prod/demo-app
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
# Enable auto-sync immediately
kubectl apply -f gitops-repo/environments/prod/demo-app/application.yaml
```

---

## 🛠️ Step 4: Rollback & Rollout Validation

```bash
# Verify the deployment
kubectl -n production rollout status deployment/demo-app

# Check ArgoCD health
argocd app get demo-app-prod
argocd app list

# Simulate a bad release → rollback via Git revert
git revert HEAD --no-edit && git push origin main
# ArgoCD auto-syncs back to the previous good manifest

# Or revert in-cluster immediately (emergency)
kubectl -n production rollout undo deployment/demo-app
# NOTE: ArgoCD selfHeal will REVERT this unless Git also reverts.
```

> **Important:** With `selfHeal: true`, manual `kubectl set image` will be undone. Treat Git as the single source of truth.

---

## 🛠️ Step 5: Full Pipeline Validation

```bash
# 1. Make a code change
echo '# conf' >> demo-app/config.txt
git add . && git commit -m "feature: bump config" && git push origin main

# 2. Watch CI run
#    GitHub → Actions → ci and release jobs

# 3. Confirm new pod version
kubectl -n production get pods
kubectl -n production get deployment demo-app -o jsonpath='{.spec.template.spec.containers[0].image}'

# 4. Confirm ArgoCD is Synced & Healthy
argocd app sync demo-app-prod
argocd app get demo-app-prod
```

---

## 🔧 Troubleshooting

```bash
# ArgoCD app OutOfSync and won't sync
argocd app get <app> --show-operation
kubectl describe application <app> -n argocd

# Image pull error
kubectl -n production describe pod <pod>
aws ecr describe-images --repository-name demo-app

# Argo CD can't read repo: provide credentials through protected shell variables.
argocd repo add https://github.com/yourorg/gitops-repo.git \
  --username "$GITHUB_USER" --password "$GITHUB_TOKEN"
# Use a least-privilege token; do not commit or print its value.
```

---

## 📝 Interview Questions

1. **Walk me through your CI/CD pipeline for EKS.**
   - **Answer:** On merge to main, CI builds & unit-tests, Trivy-scans the image, pushes to ECR, then updates the image tag in the GitOps repo. ArgoCD detects the change and syncs production. Rollbacks are Git reverts.

2. **Why update Git instead of deploying directly?**
   - **Answer:** Git becomes the single source of truth, enabling auditability, review, drift detection (selfHeal), and instant rollback by reverting a commit.

3. **How do you handle a rollback in a GitOps pipeline?**
   - **Answer:** `git revert` the bad commit and push; ArgoCD auto-syncs to the previous manifest. In emergencies, `kubectl rollout undo` works but must be reconciled in Git.

4. **What gates should a production pipeline enforce?**
   - **Answer:** Automated tests, vulnerability scan threshold, container image signing (optional), plus a manual approval for the production environment.

5. **How do you trace a deployed image back to source?**
   - **Answer:** Tag images with the commit SHA (`image:repo:v1.0-<sha>`) or use build metadata labels; record the manifest in GitOps repo.

---

## 📚 Resources for Review

- [Day 36: CodePipeline Integration](Day-36_AWS_CodePipeline_Integration/README.md)
- [Day 37: GitHub Actions for EKS](Day-37_GitHub_Actions_EKS/README.md)
- [Day 34: ArgoCD Sync & Rollback](Day-34_ArgoCD_Sync_Rollback/README.md)

---

## 🎯 Today's Assignment

1. **Stand up** the GitOps repo with prod manifest
2. **Build** CI (test + scan) and Release (push + update Git) jobs
3. **Connect** ArgoCD application with auto-sync + selfHeal
4. **Push** a change end-to-end and verify rollout
5. **Practise** a rollback via `git revert`

---

## 📊 Progress Tracker

```
Week 11-12 CI/CD:
Day 36: AWS CodePipeline Integration ⬜
Day 37: GitHub Actions for EKS ⬜
Day 38: Jenkins on EKS ⬜
Day 39: CI/CD Pipeline Practice ✅ (hands-on)
Day 40: CI/CD Interview Preparation ⬜
```

---

## 🔗 Next Week Preview

**Day 40:** CI/CD Interview Preparation

---

**Remember:** The pipeline is the glue — build it so releasing is boring and safe! 🚀
