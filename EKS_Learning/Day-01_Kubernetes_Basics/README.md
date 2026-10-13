# 📅 Day 1: What is Kubernetes?

## 🎯 Learning Objectives
By the end of today, you will understand:
- What is Kubernetes and why it exists
- Container Orchestration explained
- Kubernetes vs Docker Swarm vs Docker Compose
- Kubernetes Architecture Overview
- Real-time use cases in production

---

## 🧠 Plain-English terms before you start

| Term | Meaning in everyday language | Why it matters |
|---|---|---|
| **Container** | A packaged application plus the files and libraries it needs to run. | It makes the app behave consistently across environments. |
| **Orchestration** | Coordinating many containers automatically. | Someone must place workloads, restart failures, and scale copies. |
| **Cluster** | The group of machines Kubernetes manages as one system. | Workloads can run across multiple machines instead of one server. |
| **Control plane** | The decision-making components that accept requests and coordinate the cluster. | It decides what should happen; it does not normally run your app containers itself. |
| **Worker node** | A machine that provides CPU and memory to run Pods. | If capacity is missing, Pods may remain Pending. |
| **Desired state** | The result you ask Kubernetes to maintain, such as three replicas. | Controllers repeatedly compare the request with reality and correct differences. |

**Analogy:** Kubernetes is like a logistics coordinator: the control plane plans, worker nodes do the work, and the desired state is the order to fulfil.

**EKS connection:** Amazon EKS manages the Kubernetes control plane; you still choose and operate the worker capacity and workloads.

**Try / observe:** `kubectl get nodes`, `kubectl get pods -A`, `kubectl describe node <node-name>`. These are read-only inspection commands; replace placeholders with real names.

**Troubleshooting clue:** A Pod stuck in `Pending` often means Kubernetes cannot place it yet (for example, insufficient CPU/memory or scheduling constraints). Check `kubectl describe pod <pod-name>` and read the Events section.

**Explain it back:** What is the difference between a cluster, a node, and a Pod? Why is Kubernetes called an orchestration platform?

---

## 📖 Core Concepts

### What is Kubernetes?

**Kubernetes (K8s)** is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications.

**Simple Definition:** Think of Kubernetes as an Operating System for your cluster of machines. It manages how your applications run across multiple machines.

### Why Kubernetes?

**Problem it Solves:**
```
Without Kubernetes:
┌─────────────────────────────────────────────────────────┐
│                    Manual Management                      │
│                                                         │
│  Server 1: ┌─────┐ ┌─────┐ ┌─────┐                    │
│            │App1 │ │App2 │ │App3 │  ← Manual scaling   │
│            └─────┘ └─────┘ └─────┘                      │
│                                                         │
│  Server 2: ┌─────┐ ┌─────┐ ┌─────┐                    │
│            │App1 │ │App2 │ │App4 │  ← Manual updates   │
│            └─────┘ └─────┘ └─────┘                      │
│                                                         │
│  ❌ No auto-healing                                      │
│  ❌ No auto-scaling                                      │
│  ❌ Manual load balancing                                 │
│  ❌ No service discovery                                  │
└─────────────────────────────────────────────────────────┘

With Kubernetes:
┌─────────────────────────────────────────────────────────┐
│                    Automated Management                   │
│                                                         │
│  Kubernetes Cluster                                      │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Auto-scaling: ┌─────┐ ┌─────┐ ┌─────┐        │   │
│  │                │App1 │ │App2 │ │App3 │        │   │
│  │                └─────┘ └─────┘ └─────┘        │   │
│  │                                                 │   │
│  │  Auto-healing: If App1 crashes → Auto restart   │   │
│  │  Load balancing: Traffic distributed evenly      │   │
│  │  Service discovery: Apps find each other         │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ✅ Automatic scaling                                    │
│  ✅ Self-healing                                         │
│  ✅ Rolling updates                                      │
│  ✅ Service discovery                                    │
└─────────────────────────────────────────────────────────┘
```

### Container Orchestration Explained

**What is Container Orchestration?**
- Managing containers across multiple hosts
- Automating deployment and scaling
- Ensuring high availability
- Handling failures automatically

**Analogy:** Think of a conductor in an orchestra:
- The conductor (Kubernetes) tells each musician (Container) when to play
- If a musician gets sick, the conductor finds a replacement
- The conductor ensures the music (Application) runs smoothly

---

## 🏗️ Kubernetes Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                     │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              CONTROL PLANE (Master)               │   │
│  │                                                   │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐      │   │
│  │  │   API    │  │ Scheduler│  │Controller│      │   │
│  │  │  Server  │  │          │  │ Manager  │      │   │
│  │  │          │  │          │  │          │      │   │
│  │  │ Entry    │  │ Assigns  │  │ Maintains│      │   │
│  │  │ Point    │  │ Pods to  │  │ Desired  │      │   │
│  │  │ for all  │  │ Nodes    │  │ State    │      │   │
│  │  │ commands │  │          │  │          │      │   │
│  │  └──────────┘  └──────────┘  └──────────┘      │   │
│  │                                                   │   │
│  │  ┌──────────┐  ┌──────────┐                     │   │
│  │  │  etcd    │  │ Cloud    │                     │   │
│  │  │          │  │ Controller│                     │   │
│  │  │ Key-Value│  │ Manager  │                     │   │
│  │  │ Store    │  │ (optional)│                     │   │
│  │  └──────────┘  └──────────┘                     │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              WORKER NODES                         │   │
│  │                                                   │   │
│  │  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │   Node 1    │  │   Node 2    │              │   │
│  │  │ ┌─────────┐ │  │ ┌─────────┐ │              │   │
│  │  │ │  Pods   │ │  │ │  Pods   │ │              │   │
│  │  │ │ ┌─────┐ │ │  │ │ ┌─────┐ │ │              │   │
│  │  │ │ │ 🐳  │ │ │  │ │ │ 🐳  │ │ │              │   │
│  │  │ │ │Pod 1│ │ │  │ │ │Pod 3│ │ │              │   │
│  │  │ │ └─────┘ │ │  │ │ └─────┘ │ │              │   │
│  │  │ │ ┌─────┐ │ │  │ │ ┌─────┐ │ │              │   │
│  │  │ │ │ 🐳  │ │ │  │ │ │ 🐳  │ │ │              │   │
│  │  │ │ │Pod 2│ │ │  │ │ │Pod 4│ │ │              │   │
│  │  │ │ └─────┘ │ │  │ │ └─────┘ │ │              │   │
│  │  │ └─────────┘ │  │ └─────────┘ │              │   │
│  │  │  ┌───────┐  │  │  ┌───────┐  │              │   │
│  │  │  │Kubelet│  │  │  │Kubelet│  │              │   │
│  │  │  └───────┘  │  │  └───────┘  │              │   │
│  │  │  ┌───────┐  │  │  ┌───────┐  │              │   │
│  │  │  │Kube-  │  │  │  │Kube-  │  │              │   │
│  │  │  │proxy  │  │  │  │proxy  │  │              │   │
│  │  │  └───────┘  │  │  └───────┘  │              │   │
│  │  └─────────────┘  └─────────────┘              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Control Plane Components (Detailed)

```
┌─────────────────────────────────────────────────────────┐
│                  CONTROL PLANE DEEP DIVE                  │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │                API Server                         │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Entry point for all cluster operations│   │   │
│  │  │  • Validates and processes requests      │   │   │
│  │  │  • Communicates with etcd               │   │   │
│  │  │  • RESTful API (kubectl talks to this)   │   │   │
│  │  │  • Authentication & Authorization        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │                Scheduler                          │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Watches for unscheduled pods          │   │   │
│  │  │  • Selects best node for pod             │   │   │
│  │  │  • Considers: resources, affinity,       │   │   │
│  │  │    taints, tolerations                   │   │   │
│  │  │  • Binds pod to node                     │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │             Controller Manager                    │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Runs controller loops                 │   │   │
│  │  │  • Node Controller: manages nodes        │   │   │
│  │  │  • Replication Controller: maintains     │   │   │
│  │  │    desired pod count                     │   │   │
│  │  │  • Endpoint Controller: creates endpoints│   │   │
│  │  │  • Service Account Controller: creates   │   │   │
│  │  │    default service accounts              │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │                    etcd                           │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Distributed key-value store           │   │   │
│  │  │  • Stores all cluster data               │   │   │
│  │  │  • Configurations, secrets, state        │   │   │
│  │  │  • Highly available (cluster of 3)       │   │   │
│  │  │  • Consistent and partition-tolerant      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Worker Node Components

```
┌─────────────────────────────────────────────────────────┐
│                  WORKER NODE DEEP DIVE                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │                    Node                           │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │              kubelet                      │   │   │
│  │  │  ┌─────────────────────────────────┐   │   │   │
│  │  │  │  • Agent running on each node    │   │   │   │
│  │  │  │  • Communicates with API Server  │   │   │   │
│  │  │  │  • Manages pods on the node      │   │   │   │
│  │  │  │  • Ensures containers are running│   │   │   │
│  │  │  │  • Reports node status           │   │   │   │
│  │  │  └─────────────────────────────────┘   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │             kube-proxy                    │   │   │
│  │  │  ┌─────────────────────────────────┐   │   │   │
│  │  │  │  • Network proxy on each node    │   │   │   │
│  │  │  │  • Maintains network rules       │   │   │   │
│  │  │  │  • Handles Service networking    │   │   │   │
│  │  │  │  • iptables or IPVS rules        │   │   │   │
│  │  │  │  • Load balances across pods     │   │   │   │
│  │  │  └─────────────────────────────────┘   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │           Container Runtime              │   │   │
│  │  │  ┌─────────────────────────────────┐   │   │   │
│  │  │  │  • Docker (most common)          │   │   │   │
│  │  │  │  • containerd                    │   │   │   │
│  │  │  │  • CRI-O                         │   │   │   │
│  │  │  │  • Actually runs containers      │   │   │   │
│  │  │  └─────────────────────────────────┘   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🔄 How Kubernetes Works

### Request Flow

```
┌─────────────────────────────────────────────────────────┐
│              KUBERNETES REQUEST FLOW                      │
│                                                         │
│  User                                                    │
│  ┌─────────┐                                            │
│  │ kubectl │                                            │
│  └────┬────┘                                            │
│       │                                                 │
│       ▼                                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │              API Server                          │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  1. Receive request                     │   │   │
│  │  │  2. Authenticate user                   │   │   │
│  │  │  3. Authorize action                    │   │   │
│  │  │  4. Validate request                    │   │   │
│  │  │  5. Store in etcd                       │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│       │                                                 │
│       ▼                                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Controller Manager                  │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  1. Detects new desired state           │   │   │
│  │  │  2. Creates/updates resources           │   │   │
│  │  │  3. Ensures actual = desired state      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│       │                                                 │
│       ▼                                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Scheduler                           │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  1. Finds unscheduled pods              │   │   │
│  │  │  2. Evaluates nodes                     │   │   │
│  │  │  3. Binds pod to best node              │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│       │                                                 │
│       ▼                                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Worker Node (kubelet)               │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  1. Receives pod assignment             │   │   │
│  │  │  2. Pulls container images              │   │   │
│  │  │  3. Creates and starts containers       │   │   │
│  │  │  4. Reports status to API Server        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 📊 Kubernetes vs Docker Swarm vs Docker Compose

| Feature | Kubernetes | Docker Swarm | Docker Compose |
|---------|------------|--------------|----------------|
| **Complexity** | High | Low | Very Low |
| **Scalability** | Excellent | Good | Limited |
| **Auto-scaling** | Yes | No | No |
| **Load Balancing** | Built-in | Built-in | Manual |
| **Rolling Updates** | Yes | Yes | Manual |
| **Self-healing** | Yes | Yes | No |
| **Service Discovery** | Yes | Yes | Limited |
| **Production Ready** | Yes | Yes | No (Dev only) |
| **Learning Curve** | Steep | Moderate | Easy |

### When to Use What?

```
┌─────────────────────────────────────────────────────────┐
│                   USE CASE GUIDE                         │
│                                                         │
│  Docker Compose:                                        │
│  ┌─────────────────────────────────────────────────┐   │
│  │  ✅ Local development                           │   │
│  │  ✅ Testing                                     │   │
│  │  ✅ Single-host applications                    │   │
│  │  ❌ Production                                  │   │
│  │  ❌ Multi-host                                  │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Docker Swarm:                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │  ✅ Simple production deployments              │   │
│  │  ✅ Small to medium scale                      │   │
│  │  ✅ Quick setup                                │   │
│  │  ❌ Complex microservices                      │   │
│  │  ❌ Advanced networking                        │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Kubernetes:                                            │
│  ┌─────────────────────────────────────────────────┐   │
│  │  ✅ Enterprise applications                    │   │
│  │  ✅ Complex microservices                      │   │
│  │  ✅ Large scale                                │   │
│  │  ✅ Multi-cloud                                 │   │
│  │  ✅ Advanced networking & storage              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🌍 Real-Time Use Cases

### Companies Using Kubernetes

| Company | Use Case | Scale |
|---------|----------|-------|
| **Netflix** | Streaming infrastructure | Millions of pods |
| **Spotify** | Music delivery platform | 2M+ pods daily |
| **Airbnb** | Microservices | 1000+ services |
| **Samsung** | IoT workloads | Global scale |
| **The New York Times** | Content delivery | High availability |
| **Wikimedia** | Wikipedia infrastructure | Open source |

### Real-World Scenario

```
┌─────────────────────────────────────────────────────────┐
│              NETFLIX - REAL USE CASE                     │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Netflix Architecture                │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  User Request                            │   │   │
│  │  │  (Watch movie)                           │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │                    │                             │   │
│  │                    ▼                             │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Load Balancer (ALB)                     │   │   │
│  │  │  - Distributes traffic                   │   │   │
│  │  │  - Handles SSL termination               │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │                    │                             │   │
│  │                    ▼                             │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Kubernetes Cluster                      │   │   │
│  │  │  ┌─────────────────────────────────┐   │   │   │
│  │  │  │  Microservices:                  │   │   │   │
│  │  │  │  - User Service                  │   │   │   │
│  │  │  │  - Content Service               │   │   │   │
│  │  │  │  - Recommendation Service        │   │   │   │
│  │  │  │  - Payment Service               │   │   │   │
│  │  │  │  - Streaming Service             │   │   │   │
│  │  │  └─────────────────────────────────┘   │   │   │
│  │  │                                          │   │   │
│  │  │  Features:                               │   │   │
│  │  │  • Auto-scaling (peak hours)            │   │   │
│  │  │  • Self-healing (fault tolerance)       │   │   │
│  │  │  • Rolling updates (zero downtime)      │   │   │
│  │  │  • Service discovery (microservices)    │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Hands-On Practice

### Step 1: Install Docker
```bash
# Windows
# Download Docker Desktop from https://www.docker.com/products/docker-desktop

# Verify installation
docker --version
```

### Step 2: Run Your First Container
```bash
# Run nginx container
docker run -d -p 8080:80 nginx

# Check running containers
docker ps

# Stop container
docker stop <container-id>
```

### Step 3: Understand Container vs Pod
```bash
# Container: Single process
docker run nginx

# Pod: One or more containers (Kubernetes concept)
# We'll create this in Day 2
```

---

## 📝 Interview Questions

### Beginner Level

1. **What is Kubernetes and why is it used?**
   - **Answer:** Kubernetes is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications. It's used to manage complex applications across multiple machines, ensuring high availability, scalability, and fault tolerance.

2. **What is the difference between Docker and Kubernetes?**
   - **Answer:** Docker is a container runtime that builds and runs containers on a single host. Kubernetes is an orchestration platform that manages containers across multiple hosts, providing features like auto-scaling, load balancing, and self-healing.

3. **What are the main components of Kubernetes architecture?**
   - **Answer:** 
     - Control Plane: API Server, Scheduler, Controller Manager, etcd
     - Worker Nodes: kubelet, kube-proxy, Container Runtime

4. **What is a container?**
   - **Answer:** A container is a lightweight, standalone, executable package that includes everything needed to run a piece of software, including code, runtime, system tools, and libraries.

5. **Why would you choose Kubernetes over Docker Swarm?**
   - **Answer:** Kubernetes offers more advanced features like auto-scaling, better networking, more storage options, stronger community support, and is more suitable for complex, large-scale applications.

### Intermediate Level

6. **Explain the role of etcd in Kubernetes.**
   - **Answer:** etcd is a distributed key-value store that stores all cluster data, including configurations, secrets, and state. It's the single source of truth for the cluster and is critical for high availability.

7. **How does the Kubernetes scheduler work?**
   - **Answer:** The scheduler watches for unscheduled pods, evaluates nodes based on resource requirements, affinity rules, taints/tolerations, and binds pods to the most suitable node.

8. **What is the difference between a Pod and a Container?**
   - **Answer:** A container is a single process, while a Pod is the smallest deployable unit in Kubernetes that can contain one or more containers that share network namespace and storage volumes.

9. **How does Kubernetes handle self-healing?**
   - **Answer:** Kubernetes uses controllers (like ReplicaSet) to monitor pod health. If a pod fails, the controller automatically creates a new pod to maintain the desired state.

10. **What is the difference between Control Plane and Worker Nodes?**
    - **Answer:** Control Plane manages the cluster (scheduling, API, state), while Worker Nodes run the actual application workloads (pods/containers).

---

## 📚 Resources for Today

### Official Documentation
- [Kubernetes Overview](https://kubernetes.io/docs/concepts/overview/)
- [Kubernetes Architecture](https://kubernetes.io/docs/concepts/overview/components/)

### Video Tutorials
- Kubernetes 101 by TechWorld with Nana (YouTube)
- Kubernetes Explained by IBM Technology (YouTube)

### Reading Material
- "Kubernetes in Action" by Marko Lukša
- "The Kubernetes Book" by Nigel Poulton

---

## 🎯 Today's Assignment

1. **Read:** Kubernetes official documentation overview
2. **Watch:** Kubernetes 101 video (30 minutes)
3. **Practice:** Run 3 different Docker containers
4. **Document:** Write down 5 questions you have about Kubernetes

---

## 📊 Progress Tracker

```
Day 1: What is Kubernetes?                    ⬜
Day 2: Pods, ReplicaSets, Deployments         ⬜
Day 3: Services and Networking                ⬜
...
Day 60: Final Review                          ⬜
```

---

## 🔗 Next Day Preview

**Day 2:** Pods, ReplicaSets, and Deployments
- Create your first Pod
- Understand ReplicaSets
- Deploy applications with Deployments
- Practice kubectl commands

---

**Remember:** Understanding the fundamentals is crucial. Take your time to grasp these concepts before moving forward! 🚀
