# 📅 Day 2: Pods, ReplicaSets, and Deployments

## 🎯 Learning Objectives
By the end of today, you will understand:
- What is a Pod and how to create one
- ReplicaSets and their purpose
- Deployments and their benefits
- Essential kubectl commands
- Hands-on practice with YAML manifests

---

## 🧠 Plain-English terms before you start

| Term | Meaning in everyday language | Why it matters |
|---|---|---|
| **Pod** | The smallest unit Kubernetes schedules; it wraps one or more closely related containers. | Containers in a Pod share its network identity and can share mounted volumes. |
| **Replica** | One running copy of an application workload. | Multiple copies can handle traffic and provide resilience. |
| **ReplicaSet** | A controller that keeps the requested number of matching Pods alive. | If a Pod disappears, it creates a replacement. |
| **Deployment** | The higher-level object used to manage stateless app releases and ReplicaSets. | It supports controlled updates and rollbacks. |
| **Label / selector** | A label is metadata; a selector chooses objects with matching labels. | Controllers and Services use selectors to find the right Pods. |
| **Declarative** | You describe the result you want; Kubernetes works out the steps. | A YAML manifest can be reviewed, versioned, and reapplied. |
| **Rolling update** | Replacing old app Pods gradually with new ones. | It can reduce downtime, but bad readiness settings can still disrupt users. |

**Analogy:** A Deployment is the team manager, a ReplicaSet maintains the number of players, and Pods are the individual players.

**EKS connection:** These are Kubernetes objects, so their behavior is mostly the same on EKS and on a local cluster.

**Try / observe:** `kubectl get deploy,rs,pods`, `kubectl describe deployment <name>`, `kubectl rollout status deployment/<name>`, `kubectl rollout history deployment/<name>`.

**Troubleshooting clue:** If a Deployment has fewer ready replicas than desired, compare `kubectl get pods` with `kubectl describe pod <pod-name>`; inspect Events and container logs with `kubectl logs <pod-name>`.

**Explain it back:** Why should you normally create a Deployment instead of a standalone Pod? What does a selector connect?

---

## 📖 Core Concepts

### What is a Pod?

**Pod** is the smallest deployable unit in Kubernetes. It represents a single instance of a running process in your cluster.

**Key Points:**
- Contains one or more containers
- All containers share the same network namespace
- All containers share the same storage volumes
- Has a unique IP address
- Pods are ephemeral (can be terminated and replaced)

### Pod Anatomy

```
┌─────────────────────────────────────────────────────────┐
│                     POD STRUCTURE                        │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │                    Pod                            │   │
│  │  IP: 10.244.1.5                                  │   │
│  │                                                   │   │
│  │  ┌─────────────────┐  ┌─────────────────┐       │   │
│  │  │   Container 1   │  │   Container 2   │       │   │
│  │  │  ┌───────────┐  │  │  ┌───────────┐  │       │   │
│  │  │  │   nginx   │  │  │  │  sidecar  │  │       │   │
│  │  │  │           │  │  │  │  (logging) │  │       │   │
│  │  │  │  Port:80  │  │  │  │  Port:9090 │  │       │   │
│  │  │  └───────────┘  │  │  └───────────┘  │       │   │
│  │  └─────────────────┘  └─────────────────┘       │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐    │   │
│  │  │         Shared Volumes                    │    │   │
│  │  │  ┌─────────────┐  ┌─────────────┐       │    │   │
│  │  │  │   emptyDir  │  │  ConfigMap  │       │    │   │
│  │  │  └─────────────┘  └─────────────┘       │    │   │
│  │  └─────────────────────────────────────────┘    │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐    │   │
│  │  │         Shared Network                    │    │   │
│  │  │  localhost communication                 │    │   │
│  │  └─────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Pod YAML Example

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod
  labels:
    app: my-app
    environment: development
spec:
  containers:
  - name: nginx-container
    image: nginx:1.21
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    env:
    - name: ENVIRONMENT
      value: "development"
```

---

### What is a ReplicaSet?

**ReplicaSet** ensures that a specified number of pod replicas are running at any given time.

**Key Points:**
- Maintains desired pod count
- Creates/deletes pods as needed
- Uses label selectors to manage pods
- Rarely used directly (Deployments use them)

### ReplicaSet Architecture

```
┌─────────────────────────────────────────────────────────┐
│                 REPLICASET STRUCTURE                      │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │               ReplicaSet                         │   │
│  │  Name: my-replicaset                            │   │
│  │  Replicas: 3 (desired)                          │   │
│  │  Selector: app=my-app                           │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Current State                           │   │   │
│  │  │  - Running Pods: 2                       │   │   │
│  │  │  - Desired Pods: 3                       │   │   │
│  │  │  - Action: Create 1 more pod            │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│         ┌───────────────┼───────────────┐              │
│         ▼               ▼               ▼              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │
│  │   Pod 1     │ │   Pod 2     │ │   Pod 3     │      │
│  │ ┌─────────┐ │ │ ┌─────────┐ │ │ ┌─────────┐ │      │
│  │ │   🐳    │ │ │ │   🐳    │ │ │ │   🐳    │ │      │
│  │ │  nginx  │ │ │ │  nginx  │ │ │ │  nginx  │ │      │
│  │ └─────────┘ │ │ └─────────┘ │ │ └─────────┘ │      │
│  │  10.244.1.5 │ │ 10.244.2.3 │ │ 10.244.1.8 │      │
│  └─────────────┘ └─────────────┘ └─────────────┘      │
│                                                         │
│  If Pod 2 crashes:                                      │
│  1. ReplicaSet detects pod failure                      │
│  2. Creates new pod to replace it                       │
│  3. Maintains desired count of 3                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### ReplicaSet YAML Example

```yaml
# replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

---

### What is a Deployment?

**Deployment** provides declarative updates for Pods and ReplicaSets. It's the most common way to deploy applications in Kubernetes.

**Key Points:**
- Manages ReplicaSets (not pods directly)
- Provides rolling updates
- Supports rollback
- Supports pause/resume
- Manages deployment history

### Deployment Architecture

```
┌─────────────────────────────────────────────────────────┐
│                DEPLOYMENT STRUCTURE                       │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │               Deployment                         │   │
│  │  Name: my-deployment                            │   │
│  │  Replicas: 3                                    │   │
│  │  Strategy: RollingUpdate                        │   │
│  │  Image: nginx:1.21 → nginx:1.22                │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Manages ReplicaSets                      │   │   │
│  │  │  ┌─────────────────────────────────┐   │   │   │
│  │  │  │  Old ReplicaSet (nginx:1.21)   │   │   │   │
│  │  │  │  Replicas: 0 (scaling down)    │   │   │   │
│  │  │  └─────────────────────────────────┘   │   │   │
│  │  │  ┌─────────────────────────────────┐   │   │   │
│  │  │  │  New ReplicaSet (nginx:1.22)   │   │   │   │
│  │  │  │  Replicas: 3 (scaling up)      │   │   │   │
│  │  │  └─────────────────────────────────┘   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Rolling Update Process:                                │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Step 1: Create new pod with updated image      │   │
│  │  Step 2: Wait for pod to be ready               │   │
│  │  Step 3: Terminate old pod                      │   │
│  │  Step 4: Repeat until all pods are updated      │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Deployment YAML Example

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

---

## 🔄 Deployment Strategies

### Rolling Update (Default)

```
┌─────────────────────────────────────────────────────────┐
│              ROLLING UPDATE STRATEGY                     │
│                                                         │
│  Initial State (v1):                                    │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐     │
│  │  v1     │ │  v1     │ │  v1     │ │  v1     │     │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘     │
│                                                         │
│  Step 1: Create new pod                                 │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐     │
│  │  v1     │ │  v1     │ │  v1     │ │  v2     │     │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘     │
│                                                         │
│  Step 2: Terminate old pod                              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐                  │
│  │  v1     │ │  v1     │ │  v2     │                  │
│  └─────────┘ └─────────┘ └─────────┘                  │
│                                                         │
│  Step 3: Create another new pod                         │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐     │
│  │  v1     │ │  v1     │ │  v2     │ │  v2     │     │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘     │
│                                                         │
│  Step 4: Continue until all updated                     │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐     │
│  │  v2     │ │  v2     │ │  v2     │ │  v2     │     │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Blue-Green Deployment

```
┌─────────────────────────────────────────────────────────┐
│              BLUE-GREEN DEPLOYMENT                       │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Load Balancer                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Traffic → Blue (v1)                     │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│         ┌───────────────┴───────────────┐              │
│         ▼                               ▼              │
│  ┌─────────────┐                 ┌─────────────┐      │
│  │  Blue (v1)  │                 │  Green (v2)  │      │
│  │  ┌─────┐   │                 │  ┌─────┐   │      │
│  │  │Pod 1│   │                 │  │Pod 1│   │      │
│  │  │Pod 2│   │                 │  │Pod 2│   │      │
│  │  │Pod 3│   │                 │  │Pod 3│   │      │
│  │  └─────┘   │                 │  └─────┘   │      │
│  │  ACTIVE    │                 │  STANDBY   │      │
│  └─────────────┘                 └─────────────┘      │
│                                                         │
│  Deployment Process:                                    │
│  1. Deploy Green (v2) alongside Blue (v1)              │
│  2. Test Green environment                              │
│  3. Switch traffic from Blue to Green                   │
│  4. Keep Blue as rollback option                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Canary Deployment

```
┌─────────────────────────────────────────────────────────┐
│              CANARY DEPLOYMENT                           │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Load Balancer (weighted routing)                │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  90% traffic → v1 (stable)              │   │   │
│  │  │  10% traffic → v2 (canary)              │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│         ┌───────────────┴───────────────┐              │
│         ▼                               ▼              │
│  ┌─────────────┐                 ┌─────────────┐      │
│  │  Stable (v1)│                 │  Canary (v2) │      │
│  │  ┌─────┐   │                 │  ┌─────┐   │      │
│  │  │Pod 1│   │                 │  │Pod 1│   │      │
│  │  │Pod 2│   │                 │  └─────┘   │      │
│  │  │Pod 3│   │                 │             │      │
│  │  │Pod 4│   │                 │             │      │
│  │  └─────┘   │                 │             │      │
│  │  (4 replicas)│               │  (1 replica) │      │
│  └─────────────┘                 └─────────────┘      │
│                                                         │
│  Process:                                               │
│  1. Deploy v2 with 1 replica                           │
│  2. Monitor metrics (error rate, latency)              │
│  3. If metrics are good, increase v2 replicas          │
│  4. Continue until v2 replaces v1                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Essential kubectl Commands

### Pod Commands

```bash
# Create pod from YAML
kubectl apply -f pod.yaml

# List pods
kubectl get pods
kubectl get pods -o wide  # Show more details
kubectl get pods -n <namespace>  # Specific namespace
kubectl get pods --all-namespaces  # All namespaces

# Describe pod (detailed info)
kubectl describe pod <pod-name>

# Get pod logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>  # Multi-container pod
kubectl logs -f <pod-name>  # Follow logs
kubectl logs --previous <pod-name>  # Previous instance logs

# Execute command in pod
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec <pod-name> -- ls /app

# Delete pod
kubectl delete pod <pod-name>
kubectl delete pod <pod-name> --grace-period=0 --force  # Force delete
```

### ReplicaSet Commands

```bash
# List ReplicaSets
kubectl get replicasets
kubectl get rs  # Short form

# Describe ReplicaSet
kubectl describe replicaset <rs-name>

# Scale ReplicaSet
kubectl scale replicaset <rs-name> --replicas=5

# Delete ReplicaSet
kubectl delete replicaset <rs-name>
```

### Deployment Commands

```bash
# Create deployment
kubectl create deployment nginx --image=nginx:1.21

# List deployments
kubectl get deployments
kubectl get deploy  # Short form

# Describe deployment
kubectl describe deployment <deploy-name>

# Scale deployment
kubectl scale deployment <deploy-name> --replicas=5

# Update image
kubectl set image deployment/<deploy-name> nginx=nginx:1.22

# Check rollout status
kubectl rollout status deployment/<deploy-name>

# View rollout history
kubectl rollout history deployment/<deploy-name>

# Rollback deployment
kubectl rollout undo deployment/<deploy-name>
kubectl rollout undo deployment/<deploy-name> --to-revision=2

# Pause/Resume deployment
kubectl rollout pause deployment/<deploy-name>
kubectl rollout resume deployment/<deploy-name>
```

---

## 📝 Practice Exercises

### Exercise 1: Create Your First Pod

```bash
# Create pod using kubectl
kubectl run my-nginx --image=nginx --port=80

# Check pod status
kubectl get pods

# Get detailed information
kubectl describe pod my-nginx

# Access the pod
kubectl exec -it my-nginx -- /bin/bash

# Inside the pod, check nginx
curl localhost

# Exit the pod
exit
```

### Exercise 2: Create Pod from YAML

```yaml
# my-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

```bash
# Apply the YAML
kubectl apply -f my-pod.yaml

# Verify
kubectl get pods
kubectl get pod nginx-pod -o yaml
```

### Exercise 3: Create a Deployment

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
        image: nginx:1.21
        ports:
        - containerPort: 80
```

```bash
# Create deployment
kubectl apply -f nginx-deployment.yaml

# Check status
kubectl get deployments
kubectl get pods -l app=nginx

# Watch pods being created
kubectl get pods -w

# Scale deployment
kubectl scale deployment nginx-deployment --replicas=5

# Verify scaling
kubectl get pods -l app=nginx
```

### Exercise 4: Update and Rollback

```bash
# Update the deployment image
kubectl set image deployment/nginx-deployment nginx=nginx:1.22

# Watch the rollout
kubectl rollout status deployment/nginx-deployment

# Check rollout history
kubectl rollout history deployment/nginx-deployment

# Rollback to previous version
kubectl rollout undo deployment/nginx-deployment

# Verify rollback
kubectl get pods -l app=nginx
```

---

## 🐛 Troubleshooting Guide

### Pod Stuck in Pending

```bash
# Check pod events
kubectl describe pod <pod-name>

# Common causes:
# 1. Insufficient resources
# 2. No matching node
# 3. PVC not bound

# Check node resources
kubectl describe nodes
kubectl top nodes
```

### Pod Stuck in CrashLoopBackOff

```bash
# Check pod logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous

# Common causes:
# 1. Application error
# 2. Configuration issue
# 3. Missing environment variables

# Check pod events
kubectl describe pod <pod-name>
```

### Pod Stuck in ImagePullBackOff

```bash
# Check image name
kubectl describe pod <pod-name>

# Common causes:
# 1. Wrong image name
# 2. Image doesn't exist
# 3. Registry authentication issue

# Test image manually
docker pull <image-name>
```

---

## 📝 Interview Questions

### Beginner Level

1. **What is a Pod in Kubernetes?**
   - **Answer:** A Pod is the smallest deployable unit in Kubernetes. It represents a single instance of a running process and can contain one or more containers that share network namespace and storage volumes.

2. **What is the difference between Pod and Container?**
   - **Answer:** A container is a single process, while a Pod is the smallest deployable unit in Kubernetes that can contain one or more containers. Pods share network namespace and storage volumes.

3. **What is a ReplicaSet?**
   - **Answer:** A ReplicaSet ensures that a specified number of pod replicas are running at any given time. It uses label selectors to manage pods and creates/deletes pods as needed to maintain the desired count.

4. **What is a Deployment?**
   - **Answer:** A Deployment provides declarative updates for Pods and ReplicaSets. It manages ReplicaSets, supports rolling updates, rollbacks, and is the most common way to deploy applications in Kubernetes.

5. **How does a Deployment differ from a ReplicaSet?**
   - **Answer:** A Deployment manages ReplicaSets and provides additional features like rolling updates, rollbacks, and deployment history. ReplicaSets only maintain a desired pod count.

### Intermediate Level

6. **What are the deployment strategies in Kubernetes?**
   - **Answer:** 
     - Rolling Update: Gradually replaces old pods with new ones
     - Blue-Green: Deploys new version alongside old, then switches traffic
     - Canary: Deploys new version to a subset of users

7. **What is maxSurge and maxUnavailable in rolling updates?**
   - **Answer:** 
     - maxSurge: Maximum number of pods that can be created above desired count
     - maxUnavailable: Maximum number of pods that can be unavailable during update

8. **How do you rollback a deployment?**
   - **Answer:** Use `kubectl rollout undo deployment/<name>` to rollback to previous version, or `--to-revision=<number>` for specific version.

9. **What are labels and selectors?**
   - **Answer:** Labels are key-value pairs attached to objects for identification. Selectors are used to filter and select objects based on labels.

10. **How do you update a deployment without downtime?**
    - **Answer:** Use rolling update strategy with appropriate maxSurge and maxUnavailable values, and ensure proper readiness probes are configured.

### Advanced Level

11. **What are liveness and readiness probes?**
    - **Answer:** 
      - Liveness probe: Checks if container is running; restarts if failed
      - Readiness probe: Checks if container is ready to receive traffic; removes from service if failed

12. **How do you perform a canary deployment?**
    - **Answer:** Deploy new version with fewer replicas, use weighted routing in Ingress or Service, monitor metrics, and gradually increase replicas if metrics are good.

13. **What is the difference between RollingUpdate and Recreate strategy?**
    - **Answer:** 
      - RollingUpdate: Gradually replaces old Pods. It can reduce downtime, but zero downtime is not guaranteed; readiness probes, capacity, app compatibility, and traffic handling matter.
      - Recreate: Terminates all old pods before creating new ones (downtime)

14. **How do you pause a deployment rollout?**
    - **Answer:** Use `kubectl rollout pause deployment/<name>` to pause, make multiple changes, then `kubectl rollout resume` to resume.

15. **What are Pod Disruption Budgets (PDB)?**
    - **Answer:** PDB limits the number of pods that can be voluntarily disrupted at the same time, ensuring availability during maintenance.

---

## 📚 Resources for Today

### Official Documentation
- [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)

### Video Tutorials
- Kubernetes Pods Explained by TechWorld with Nana
- Kubernetes Deployments by KodeKloud

---

## 🎯 Today's Assignment

1. **Create:** 3 different pods using YAML
2. **Create:** A deployment with 5 replicas
3. **Update:** The deployment image and observe rolling update
4. **Rollback:** The deployment to previous version
5. **Document:** Differences between Pod, ReplicaSet, and Deployment

---

## 📊 Progress Tracker

```
Day 1: What is Kubernetes?                    ✅
Day 2: Pods, ReplicaSets, Deployments         ⬜
Day 3: Services and Networking                ⬜
...
Day 60: Final Review                          ⬜
```

---

## 🔗 Next Day Preview

**Day 3:** Services and Networking
- Understand Service types (ClusterIP, NodePort, LoadBalancer)
- Create and expose services
- Network connectivity between pods

---

**Remember:** Practice creating and managing pods, ReplicaSets, and deployments! 🚀
