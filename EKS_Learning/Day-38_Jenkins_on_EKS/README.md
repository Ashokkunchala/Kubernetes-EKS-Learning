# 📅 Day 38: Jenkins on EKS

## 🎯 Learning Objectives
By the end of today, you will understand:
- Deploying Jenkins on EKS with Helm
- Jenkins pipelines for building and deploying to EKS
- Persistent storage for Jenkins (EBS/EFS)
- Credentials and agent scaling in Kubernetes
- Alternatives: running Jenkins agents as Kubernetes pods

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Jenkins controller** | The service that hosts Jenkins configuration, schedules work, and coordinates agents. |
| **Agent** | A worker environment that executes pipeline steps. |
| **Pipeline** | The defined sequence of build, test, scan, and delivery steps. |
| **Credential** | A protected authentication value such as a token, password, or key. |
| **RBAC** | Rules controlling which Kubernetes API operations an identity can perform. |
| **Ephemeral agent** | A temporary worker created for a job and removed afterward. |

**Analogy:** The controller is the dispatcher; agents are workstations. Ephemeral agents are cleaned up after a job to reduce leftover state.

**Try / observe:** Identify where the controller runs, where jobs execute, which ServiceAccount the agent uses, and which secrets are mounted or injected.

**Troubleshooting clue:** If a pipeline cannot access the Kubernetes API, verify the agent's identity and RBAC separately from Jenkins credentials and network reachability.

**Explain it back:** Why should the Jenkins controller and build agents be treated as different responsibilities? Why use ephemeral agents?

---

## 📖 Core Concepts

### Jenkins Architecture on EKS

```
┌───────────────────────────────────────────────────────────────┐
│                       EKS CLUSTER                              │
│                                                               │
│   ┌───────────────────┐          ┌──────────────────────┐     │
│   │   Jenkins Master  │          │  Jenkins Agents      │     │
│   │   (Deployment)    │──────────│  (Kubernetes Pods)   │     │
│   │   - Web UI        │  Jobs    │  - Build agents      │     │
│   │   - Scheduler     │◀────────▶│  - Ephemeral         │     │
│   └─────────┬─────────┘  pod     └──────────┬───────────┘     │
│             │                               │                 │
│        ┌────▼────┐                     ┌────▼────┐            │
│        │  EBS /  │                     │  ECR /  │            │
│        │  EFS    │                     │  Docker│            │
│        │  (data) │                     └─────────┘            │
│        └─────────┘                                            │
└───────────────────────────────────────────────────────────────┘
```

### Key Concepts

- **Master/Controller:** Orchestrates jobs, exposes UI, stores config.
- **Agents:** Execute build jobs. The Kubernetes plugin spins up **ephemeral pods** as agents, giving elastic, isolated build capacity.
- **Jenkins Home:** Must be persistent (PVC) so config, jobs, build history survive restarts.

---

## 🛠️ Step 1: Prepare Persistent Storage

```yaml
# jenkins-pvc.yaml - EBS-backed PVC for Jenkins home
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-home
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp2
  resources:
    requests:
      storage: 20Gi
```

```bash
kubectl create ns jenkins
kubectl apply -f jenkins-pvc.yaml
```

---

## 🛠️ Step 2: Install Jenkins with Helm

```bash
# Add the Jenkins Helm repo
helm repo add jenkins https://charts.jenkins.io
helm repo update

# Install with custom values
helm install jenkins jenkins/jenkins \
  --namespace jenkins \
  --create-namespace \
  --values jenkins-values.yaml
```

`jenkins-values.yaml`:

```yaml
controller:
  adminUser: admin
  adminPassword: "ChangeMe123!"
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "1"
      memory: "2Gi"
  serviceType: ClusterIP
  servicePort: 8080
  installPlugins:
    - kubernetes:latest
    - workflow-aggregator
    - git
    - configuration-as-code
    - aws-credentials
    - docker-workflow
  existingClaim: jenkins-home

agent:
  enabled: true
  podTemplates:
    default:
      containers:
        - name: jnlp
          image: jenkins/inbound-agent:latest
```

Get the admin password and access:

```bash
# Read the generated admin password
kubectl exec --namespace jenkins -it svc/jenkins \
  -c jenkins -- /bin/cat /run/secrets/additional-admin-password

# Port-forward to access UI
kubectl --namespace jenkins port-forward svc/jenkins 8080:8080
# Open http://localhost:8080
```

---

## 🛠️ Step 3: Configure AWS Credentials

Install the **AWS Credentials** plugin, then add credentials in Jenkins:

```bash
# Generate a Jenkins Access Key, or better: use IAM IRSA for the Jenkins pod
```

**Recommended (IRSA):** give Jenkins itself an IAM role via IRSA so agents inherit AWS access:

```yaml
# In jenkins-values.yaml
controller:
  serviceAccount:
    create: true
    name: jenkins
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/jenkins-eks-role
```

```bash
# Create the IAM role for Jenkins (ECR + EKS access)
eksctl create iamserviceaccount \
  --cluster my-eks-cluster \
  --namespace jenkins \
  --name jenkins \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy \
  --approve
```

---

## 🛠️ Step 4: Jenkinsfile (Pipeline-as-Code)

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            spec:
              containers:
              - name: jnlp
                image: jenkins/inbound-agent:latest
              - name: docker
                image: docker:20.10
                command: ["cat"]
                tty: true
                volumeMounts:
                - name: docker-sock
                  mountPath: /var/run/docker.sock
              volumes:
              - name: docker-sock
                hostPath:
                  path: /var/run/docker.sock
            '''
        }
    }
    environment {
        REPOSITORY_URI = "ACCOUNT_ID.dkr.ecr.${AWS_REGION}.amazonaws.com/my-app"
        IMAGE_TAG = "${env.GIT_COMMIT}"
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Build & Push Image') {
            steps {
                container('docker') {
                    sh '''
                        aws ecr get-login-password --region $AWS_REGION \
                          | docker login --username AWS --password-stdin $REPOSITORY_URI
                        docker build -t $REPOSITORY_URI:$IMAGE_TAG .
                        docker push $REPOSITORY_URI:$IMAGE_TAG
                    '''
                }
            }
        }
        stage('Deploy to EKS') {
            steps {
                container('kubectl') {
                    sh '''
                        aws eks update-kubeconfig --name my-eks-cluster --region $AWS_REGION
                        kubectl set image deployment/my-app my-app=$REPOSITORY_URI:$IMAGE_TAG -n production
                        kubectl rollout status deployment/my-app -n production
                    '''
                }
            }
        }
    }
}
```

---

## 🛠️ Step 5: Jenkinsfile Alternative Declarative with Shared Libs

For larger orgs, extract stages into a **shared library**:

```groovy
// vars/deployEKS.groovy - in shared library repo
def call(Map params) {
    sh "aws eks update-kubeconfig --name ${params.cluster} --region ${params.region}"
    sh "kubectl set image deployment/${params.deployment} ${params.deployment}=${params.image} -n ${params.namespace}"
}
```

```groovy
pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps { deployEKS(cluster: 'my-eks', region: 'us-east-1', deployment: 'my-app', image: '...') }
        }
    }
}
```

---

## 📊 Jenkins Metrics & Monitoring

```bash
# Prometheus plugin exposes metrics at /prometheus
# Add a ServiceMonitor to scrape Jenkins
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: jenkins
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: jenkins
  endpoints:
  - port: http
    path: /prometheus
    interval: 30s
```

---

## 📝 Interview Questions

1. **Why run Jenkins on EKS rather than a standalone VM?**
   - **Answer:** Elastic agents as pods, unified platform with other workloads, easier scaling, built-in HA with Kubernetes, and centralized logging/monitoring.

2. **What is the Kubernetes plugin for Jenkins?**
   - **Answer:** It lets agents be specified as pod templates; Jenkins spins up a fresh pod per build, giving isolation and elasticity.

3. **How do you persist Jenkins state on EKS?**
   - **Answer:** A PVC (EBS for single-replica home, EFS if multiple replicas) mounted at `/var/jenkins_home`; config-as-code + backups for disaster recovery.

4. **How do you securely give Jenkins AWS access?**
   - **Answer:** Prefer IRSA — the Jenkins controller pod gets an IAM role via a service account annotation; agents inherit it, no static keys.

5. **Jenkins vs GitOps (ArgoCD)?**
   - **Answer:** Jenkins is excellent for build/CI and complex test orchestration; ArgoCD is the GitOps CD layer that converges the cluster to Git state. They complement each other.

---

## 📚 Resources for Review

- [Day 36: CodePipeline Integration](Day-36_AWS_CodePipeline_Integration/README.md)
- [Day 37: GitHub Actions for EKS](Day-37_GitHub_Actions_EKS/README.md)

### External
- [Jenkins Helm Chart](https://charts.jenkins.io)
- [Jenkins Kubernetes Plugin](https://plugins.jenkins.io/kubernetes/)
- [Jenkins on AWS](https://aws.amazon.com/jenkins/)

---

## 🎯 Today's Assignment

1. **Install** Jenkins on a namespace using Helm
2. **Create** a PVC and confirm persistence across pod restart
3. **Set up** IRSA so Jenkins can talk to ECR/EKS
4. **Write** a Jenkinsfile that builds + deploys
5. **Enable** a Kubernetes agent template and run a build on it

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

**Day 39:** CI/CD Pipeline Practice - build end-to-end

---

**Remember:** Jenkins-as-code and declarative pipelines keep your CI reproducible and reviewable! 🚀
