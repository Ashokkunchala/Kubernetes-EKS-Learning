# 📅 Day 37: GitHub Actions with EKS

## 🎯 Learning Objectives
By the end of today, you will understand:
- Building CI/CD with GitHub Actions for EKS
- OIDC-based IAM for GitHub Actions (no long-lived keys)
- Building, testing, and pushing to ECR
- Deploying to EKS with kubectl/kustomize/Helm
- Reusable workflows and environment promotion

---

## 📖 Core Concepts

### GitHub Actions + EKS Flow

```
┌───────────────────────────────────────────────────────────────┐
│                   GITHUB ACTIONS PIPELINE                      │
│                                                               │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐  │
│   │  Push /  │──▶│  Build   │──▶│ Test/Scan│──▶│  Deploy  │  │
│   │ PR event │   │  Docker  │   │          │   │  to EKS  │  │
│   └──────────┘   └──────────┘   └──────────┘   └────┬─────┘  │
│        │                                              │       │
│     Filters:                                    ┌─────▼─────┐ │
│     main / PRs                                  │  EKS via  │ │
│                                                  │ Kubeconfig│ │
│                                                  │ (OIDC)    │ │
│                                                  └───────────┘ │
└───────────────────────────────────────────────────────────────┘
```

### Why OIDC Over Access Keys

- **GitHub → AWS OIDC:** GitHub presents a token signed by GitHub's OIDC issuer.
- **AWS trusts it** via a Web Identity role, minting short-lived STS credentials.
- **No static `AWS_ACCESS_KEY_ID`** in secrets → far lower leak risk.
- Role is scoped to one GitHub repo/branch/environment.

---

## 🛠️ Step 1: Set Up OIDC IAM Role

```bash
# Output GitHub's OIDC thumbprint & provider via CloudFormation
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

Create the trust policy (`oidc-role.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:yourorg/my-app:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

```bash
aws iam create-role \
  --role-name github-actions-eks-deploy \
  --assume-role-policy-document file://oidc-role.json

# Attach least-privilege policy for ECR + EKS
aws iam attach-role-policy \
  --role-name github-actions-eks-deploy \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser
```

---

## 🛠️ Step 2: Core Workflow (.github/workflows/deploy.yml)

```yaml
name: Build and Deploy to EKS

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: my-app
  EKS_CLUSTER: my-eks-cluster

permissions:
  id-token: write
  contents: read

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::ACCOUNT_ID:role/github-actions-eks-deploy
          role-session-name: github-actions
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image to ECR
        id: build
        env:
          ECR_REGISTRY: ${{ steps.ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:$IMAGE_TAG .
          docker push $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Configure kubectl
        run: |
          aws eks update-kubeconfig --region ${{ env.AWS_REGION }} \
            --name ${{ env.EKS_CLUSTER }}

      - name: Deploy to EKS with kustomize
        run: |
          cd k8s/overlays/prod
          kustomize edit set image my-app=${{ steps.build.outputs.image }}
          kustomize build | kubectl apply -f -
          kubectl rollout status deployment/my-app -n production
```

---

## 🛠️ Step 3: Environment Promotion

Split build/test from deploy with environments and approval gates:

```yaml
deploy-staging:
  needs: build
  environment: staging
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - name: Configure AWS
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::ACCOUNT_ID:role/github-actions-staging
        aws-region: ${{ env.AWS_REGION }}
    - name: Release to staging
      run: |
        aws eks update-kubeconfig --name ${{ env.EKS_CLUSTER }} --region us-east-1
        kubectl set image deployment/my-app my-app=${{ needs.build.outputs.image }} -n staging

deploy-production:
  needs: deploy-staging
  environment: production   # requires manual approval in GitHub UI
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - name: Configure AWS
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::ACCOUNT_ID:role/github-actions-production
        aws-region: ${{ env.AWS_REGION }}
    - name: Release to production
      run: |
        aws eks update-kubeconfig --name ${{ env.EKS_CLUSTER }} --region us-east-1
        kubectl set image deployment/my-app my-app=${{ needs.build.outputs.image }} -n production
```

> Enable **"Required reviewers"** on the `production` environment in GitHub repo settings for a manual approval gate before production deploys.

---

## 🛠️ Optional: Secrets Management

```bash
# Store kubeconfig or cluster name as repo/environment secrets
gh secret set EKS_CLUSTER --body my-prod-cluster
gh secret set AWS_ACCOUNT_ID --body 123456789012
gh secret set DEPLOY_ROLE_ARN --body arn:aws:iam::123456789012:role/github-actions-eks-deploy

# Reference in workflow
env:
  EKS_CLUSTER: ${{ secrets.EKS_CLUSTER }}
```

---

## 🧪 Testing Locally (act)

```bash
# Run workflow locally with Docker
brew install act
act -W .github/workflows/deploy.yml -s AWS_ACCOUNT_ID=123456789012
```

---

## 📝 Interview Questions

1. **How do you authenticate GitHub Actions to AWS securely?**
   - **Answer:** Use OIDC federation — GitHub signs a token, AWS issues short-lived STS credentials via a federated role scoped to the specific repo/branch. No static keys.

2. **What are the main jobs/stages in a GitHub Actions EKS pipeline?**
   - **Answer:** Checkout → Configure AWS (OIDC) → Login ECR → Build/Push → Configure kubectl → Apply manifests → Rollout status.

3. **How do you promote between staging and production?**
   - **Answer:** Use `needs` for order, GitHub **environments** with required reviewers as the approval gate, and separate OIDC roles per environment for isolation.

4. **GitHub Actions vs CodePipeline for EKS?**
   - **Answer:** GitHub Actions keeps CI near the code, free for public repos, huge marketplace; CodePipeline is AWS-native with deep IAM integration. Many teams use GitHub Actions for CI and ArgoCD for CD.

5. **How do you prevent drift between the pipeline and cluster state?**
   - **Answer:** Combine with GitOps (ArgoCD): the pipeline only builds/pushes images and updates the GitOps repo; ArgoCD reconciles cluster to the declared state.

---

## 📚 Resources for Review

- [Day 36: CodePipeline Integration](Day-36_AWS_CodePipeline_Integration/README.md)
- [Day 31: GitOps Concepts](Day-31_GitOps_ArgoCD_Intro/README.md)

### External
- [GitHub Actions docs](https://docs.github.com/actions)
- [AWS configure-credentials action](https://github.com/aws-actions/configure-aws-credentials)
- [GitHub Actions + OIDC pattern](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)

---

## 🎯 Today's Assignment

1. **Create** the OIDC identity provider and federated deploy role
2. **Write** a `build-and-deploy.yml` workflow
3. **Add** environment promotion with an approval gate
4. **Push** the image through ECR and deploy to EKS
5. **Test** the workflow with a commit on `main`

---

## 📊 Progress Tracker

```
Week 11-12 CI/CD:
Day 36: AWS CodePipeline Integration ⬜
Day 37: GitHub Actions for EKS ⬜
Day 38: Jenkins on EKS ⬜
Day 39: CI/CD Pipeline Practice ⬜
Day 40: CI/CD Interview Preparation ⬜
```

---

## 🔗 Next Week Preview

**Day 38:** Jenkins on EKS

---

**Remember:** Security and reproducibility come first — OIDC, immutable tags, and promo gates! 🚀
