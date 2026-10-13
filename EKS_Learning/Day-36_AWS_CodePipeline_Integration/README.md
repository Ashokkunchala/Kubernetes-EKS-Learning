# 📅 Day 36: CI/CD Concepts & AWS CodePipeline Integration

## 🎯 Learning Objectives
By the end of today, you will understand:
- Core CI/CD concepts and pipeline stages
- AWS CodePipeline architecture for EKS
- Integrating CodeCommit, CodeBuild, and CodeDeploy
- Building, testing, and deploying to EKS
- AWS curated pipeline patterns for Kubernetes

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Pipeline** | A sequence of automated steps that moves a change from source to a validated release. |
| **Source stage** | Retrieves the code or configuration that starts the workflow. |
| **Build stage** | Compiles, packages, tests, or scans the software. |
| **Artifact** | A file or package produced by one step and passed to later steps. |
| **Deploy stage** | Releases the approved artifact/configuration to an environment. |
| **IAM role** | An AWS identity assumed by a service or job to call permitted AWS APIs. |

**Analogy:** A pipeline is an assembly line: source provides materials, build produces the package, tests inspect it, and deploy installs it.

**Try / observe:** Trace one commit through each stage and identify the artifact, identity/role, and permissions used at every transition.

**Troubleshooting clue:** A failed stage can be a source permission, build/test issue, missing artifact, or deployment authorization error. Start with the failing stage's logs.

**Explain it back:** Why should a deployment job have fewer permissions than a general administrator? What is an artifact?

---

## 📖 Core Concepts

### What is CI/CD?

```
┌───────────────────────────────────────────────────────────────┐
│                     CONTINUOUS DELIVERY                        │
│                                                               │
│   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐      │
│   │  Source │──▶│  Build  │──▶│  Test   │──▶│ Deploy  │      │
│   │  Code   │   │ Compile │   │  Unit/  │   │  to EKS │      │
│   │         │   │ Package │   │  Integ. │   │         │      │
│   └─────────┘   └─────────┘   └─────────┘   └────┬────┘      │
│        ▲                                          │           │
│   CodeCommit / GitHub                    ┌─────┐  ▼           │
│   (repo webhook → pipeline)              │ ECR │┌──────────┐  │
│                                          └─────┘│  EKS     │  │
│                                                 │ Cluster  │  │
│                                                 └──────────┘  │
└───────────────────────────────────────────────────────────────┘
```

### CI/CD vs GitOps

| Aspect | Traditional CI/CD | GitOps (ArgoCD) |
|--------|-------------------|-----------------|
| Deploy mechanism | Pipeline pushes | Controller pulls |
| Desired state | Scripted | Declared in Git |
| Rollback | Manual/script | Git revert |
| Drift detection | No | Automatic (selfHeal) |
| Best for | App builds, tests | Continuous config |

**Key Insight:** CI builds the artifact; CD deploys it. On EKS, the common split is:
- **CI (CodeBuild/GitHub Actions):** Build image → push to ECR → update manifest.
- **CD (CodeDeploy/ArgoCD):** Apply the new manifest to the cluster.

---

## 🛠️ AWS CodePipeline Setup for EKS

### Step 1: Create the ECR Repository

```bash
# Create ECR repo
aws ecr create-repository \
  --repository-name my-app \
  --region us-east-1 \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256

# Get the repo URI
aws ecr describe-repositories --repository-names my-app \
  --query 'repositories[0].repositoryUri' --output text
```

### Step 2: Create the BuildSpec

```yaml
# buildspec.yaml - placed at repo root
version: 0.2

phases:
  install:
    runtime-versions:
      docker: 20
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/my-app
      - COMMIT_HASH=$(echo "$CODEBUILD_RESOLVED_SOURCE_VERSION" | cut -c 1-12)
      - IMAGE_TAG=$COMMIT_HASH
  build:
    commands:
      - echo Build started on `date`
      - docker build -t $REPOSITORY_URI:$IMAGE_TAG .
      - echo '{"IMAGE_URI":"'$REPOSITORY_URI:$IMAGE_TAG'"}' > imageDetail.json
  post_build:
    commands:
      - echo Build completed on `date`
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - printf '[{"name":"my-app","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
    - imageDetail.json
```

### Step 3: Create the Pipeline

Using AWS CLI:

```bash
# Define pipeline stages config
cat > pipeline.json <<'EOF'
{
  "pipeline": {
    "name": "my-app-pipeline",
    "roleArn": "arn:aws:iam::ACCOUNT_ID:role/service-role/codepipeline-role",
    "artifactStore": {
      "type": "S3",
      "location": "codepipeline-us-east-1-ACCOUNT_ID"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [{
          "name": "Source",
          "actionTypeId": { "category": "Source", "owner": "AWS",
            "provider": "CodeCommit", "version": "1" },
          "configuration": { "RepositoryName": "my-app", "BranchName": "main" },
          "outputArtifacts": [ { "name": "SourceArtifact" } ],
          "runOrder": 1
        }]
      },
      {
        "name": "Build",
        "actions": [{
          "name": "Build",
          "actionTypeId": { "category": "Build", "owner": "AWS",
            "provider": "CodeBuild", "version": "1" },
          "configuration": { "ProjectName": "my-app-build" },
          "inputArtifacts": [ { "name": "SourceArtifact" } ],
          "outputArtifacts": [ { "name": "BuildArtifact" } ],
          "runOrder": 1
        }]
      },
      {
        "name": "Deploy",
        "actions": [{
          "name": "Deploy",
          "actionTypeId": { "category": "Deploy", "owner": "AWS",
            "provider": "CodeDeployToEKS", "version": "1" },
          "configuration": {
            "ApplicationName": "my-app-eks-app",
            "DeploymentGroupName": "my-app-deployment-group"
          },
          "inputArtifacts": [ { "name": "BuildArtifact" } ],
          "runOrder": 1
        }]
      }
    ]
  }
}
EOF

aws codepipeline create-pipeline --cli-input-json file://pipeline.json
```

### Step 4: CodeDeploy EKS Deployment Group

```yaml
# appspec.yaml for EKS (in repo, referenced by Deployment Group)
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:region:account:task-definition/taskdef.json"
        LoadBalancerInfo:
          ContainerName: "my-app"
          ContainerPort: 8080
```

> **Note:** Native `CodeDeployToEKS` requires a Lambda-compatible wrapper; many teams use a `CodePipeline` → `EKS` action via a Lambda function or ShellScript, or hand the manifest to ArgoCD. The most robust 2026 pattern is **CodeBuild builds + ArgoCD deploys**.

---

## 🛠️ Robust Pattern: CodePipeline → ArgoCD for Deployment

This is the recommended production pattern — CI via CodePipeline, CD via ArgoCD:

```yaml
# In CodeBuild post_build: render the manifest
# deploy.yaml with the new image tag
- echo "Writing new manifest for tag $IMAGE_TAG"
- sed "s|image:.*|image: $REPOSITORY_URI:$IMAGE_TAG|" \
    k8s/deployment.yaml > out/deployment.yaml
```

```bash
# Pipeline Deploy stage: trigger ArgoCD sync
aws codepipeline put-job-success-result \
  --job-id $JOB_ID \
  --execution-id $EXECUTION_ID
```

Alternatively, push the updated manifest to the GitOps repo:

```bash
# In post_build
git add k8s/deployment.yaml
git commit -m "Release $IMAGE_TAG"
git push origin main   # ArgoCD auto-syncs the new manifest
```

---

## 🔧 Troubleshooting

```bash
# Check pipeline status
aws codepipeline get-pipeline-state --name my-app-pipeline

# Get CodeBuild logs
aws logs tail /aws/codebuild/my-app-build --follow

# Verify ECR push
aws ecr describe-images --repository-name my-app --query 'imageDetails[*].imageTags'

# Check pods reflect new image
kubectl get deployment my-app -o jsonpath='{.spec.template.spec.containers[0].image}'
kubectl rollout status deployment/my-app
```

---

## 📝 Interview Questions

1. **What are the key stages of a CI/CD pipeline?**
   - **Answer:** Source, Build, Test, Security Scan, Deploy, Verify. Each runs automatically on code changes.

2. **How does AWS CodePipeline integrate with EKS?**
   - **Answer:** CodeCommit/GitHub sources → CodeBuild builds & pushes to ECR → deploy stage updates the EKS deployment (via CodeDeploy EKS action, Lambda, or handing manifest to ArgoCD).

3. **CodePipeline vs GitHub Actions vs Jenkins?**
   - **Answer:** CodePipeline is AWS-native, tightly integrated with IAM/ECR; GitHub Actions is cloud-agnostic with huge marketplace; Jenkins is self-hosted and highly customizable but needs maintenance.

4. **What's the best way to version container images?**
   - **Answer:** Git commit SHA or semantic version tags (e.g., `v1.2.3-<sha>`), always immutable tags for production traceability.

5. **How do you roll back a bad deployment in CodePipeline?**
   - **Answer:** Re-run the previous pipeline execution, or point the deployment back to the last known-good image tag; with GitOps, revert the manifest commit.

---

## 📚 Resources for Review

- [Day 31: GitOps Concepts](Day-31_GitOps_ArgoCD_Intro/README.md)
- [Day 32: ArgoCD Install](Day-32_ArgoCD_Installation_Config/README.md)

### External
- [AWS CodePipeline](https://docs.aws.amazon.com/codepipeline/)
- [AWS CodeBuild for EKS](https://docs.aws.amazon.com/codebuild/)
- [Building CI/CD for EKS](https://docs.aws.amazon.com/eks/latest/userguide/eks-pipeline.html)

---

## 🎯 Today's Assignment

1. **Create** an ECR repository for a demo app
2. **Write** a `buildspec.yaml` that builds & pushes
3. **Create** a CodeBuild project from the spec
4. **Build** a 3-stage CodePipeline (Source → Build → Deploy)
5. **Wire** the pipeline to update a deployment (via GitOps repo or Lambda)

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

**Day 37:** GitHub Actions with EKS

---

**Remember:** A reliable pipeline makes every release boring — that's the goal! 🚀
