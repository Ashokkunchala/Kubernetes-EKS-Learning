# 📅 Day 3: Services and Networking

## 🎯 Learning Objectives
By the end of today, you will understand:
- What is a Service in Kubernetes
- Service types: ClusterIP, NodePort, LoadBalancer, ExternalName
- Service discovery and DNS
- Network connectivity between pods
- Hands-on practice with services

---

## 🧠 Plain-English terms before you start

| Term | Meaning in everyday language | Why it matters |
|---|---|---|
| **Pod IP** | An address assigned to a Pod by the cluster network. | Pods can be replaced and their IPs can change, so clients should not depend on a specific Pod IP. |
| **Service** | A stable name and virtual network endpoint for a changing set of Pods. | Clients can reach an app even when individual Pods are replaced. |
| **Selector** | A rule matching Pods by labels. | A Service sends traffic only to Pods matching its selector and considered usable endpoints. |
| **ClusterIP** | A Service address reachable from inside the cluster. | Common for internal app-to-app communication. |
| **NodePort** | A Service exposed on a port on cluster nodes. | Useful for certain labs; it is not automatically a production-grade public entry point. |
| **LoadBalancer** | A Service type that asks the environment to provide a load balancer when supported. | On EKS, integration can provision an AWS load balancer depending on configuration. |
| **DNS / service discovery** | Resolving a human-friendly name to an address. | Applications can call a Service by DNS name instead of tracking Pod IPs. |

**Analogy:** Pods are changing shop staff; a Service is the permanent shop phone number that directs calls to available staff.

**EKS connection:** A `LoadBalancer` Service may create an AWS load balancer and incur cost. Check what your manifest provisions before applying it.

**Try / observe:** `kubectl get svc,endpointslices`, `kubectl describe svc <service-name>`, `kubectl get pods --show-labels`.

**Troubleshooting clue:** A Service with no usable endpoints commonly points to a selector/label mismatch or Pods that are not Ready. Compare the Service selector with Pod labels before debugging routing.

**Explain it back:** Why is a Service more stable than a Pod IP? When would you choose ClusterIP versus LoadBalancer?

---

## 📖 Core Concepts

### What is a Service?

A **Service** is an abstraction that defines a logical set of Pods and a policy to access them. Services enable stable networking for pods.

**Why Services?**
- Pods are ephemeral (can be terminated/replaced)
- Pods have dynamic IP addresses
- Services provide stable DNS names and IP addresses
- Enable load balancing across pods

### Service Types Overview

```
┌─────────────────────────────────────────────────────────┐
│                  SERVICE TYPES                            │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  1. ClusterIP (Default)                         │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Internal only                         │   │   │
│  │  │  • Accessible within cluster             │   │   │
│  │  │  • Use case: Pod-to-Pod communication    │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  2. NodePort                                     │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Exposes on each node's IP             │   │   │
│  │  │  • Fixed port (30000-32767)              │   │   │
│  │  │  • Use case: Development/testing         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  3. LoadBalancer                                │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Creates external load balancer        │   │   │
│  │  │  • Cloud provider specific (AWS ELB)     │   │   │
│  │  │  • Use case: Production external access  │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  4. ExternalName                                │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Maps to external DNS                  │   │   │
│  │  │  • CNAME record                          │   │   │
│  │  │  • Use case: External service reference  │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Service Architecture

```
┌─────────────────────────────────────────────────────────┐
│              SERVICE ARCHITECTURE                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │                    Service                        │   │
│  │  Name: my-service                                │   │
│  │  Type: ClusterIP                                 │   │
│  │  IP: 10.96.0.100                                 │   │
│  │  Port: 80 → 8080 (targetPort)                   │   │
│  │                                                   │   │
│  │  Selector: app=my-app                           │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│         ┌───────────────┼───────────────┐              │
│         ▼               ▼               ▼              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │
│  │   Pod 1     │ │   Pod 2     │ │   Pod 3     │      │
│  │ ┌─────────┐ │ │ ┌─────────┐ │ │ ┌─────────┐ │      │
│  │ │   🐳    │ │ │ │   🐳    │ │ │ │   🐳    │ │      │
│  │ │ Port:   │ │ │ │ Port:   │ │ │ │ Port:   │ │      │
│  │ │ 8080    │ │ │ │ 8080    │ │ │ │ 8080    │ │      │
│  │ └─────────┘ │ │ └─────────┘ │ │ └─────────┘ │      │
│  │ 10.244.1.5  │ │ 10.244.2.3  │ │ 10.244.1.8  │      │
│  └─────────────┘ └─────────────┘ └─────────────┘      │
│                                                         │
│  Traffic Flow:                                           │
│  Client → Service (10.96.0.100:80) → Pod (any:8080)   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🔍 Service Types in Detail

### 1. ClusterIP Service

```
┌─────────────────────────────────────────────────────────┐
│                  CLUSTERIP SERVICE                        │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Client Pod                                      │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  curl my-service.default.svc.cluster.local│  │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  ClusterIP Service                              │   │
│  │  IP: 10.96.0.100                                │   │
│  │  DNS: my-service.default.svc.cluster.local      │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Load Balancing                           │   │   │
│  │  │  - Round Robin                           │   │   │
│  │  │  - Session Affinity (optional)           │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│         ┌───────────────┼───────────────┐              │
│         ▼               ▼               ▼              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │
│  │   Pod 1     │ │   Pod 2     │ │   Pod 3     │      │
│  │  (Backend)  │ │  (Backend)  │ │  (Backend)  │      │
│  └─────────────┘ └─────────────┘ └─────────────┘      │
│                                                         │
│  ✅ Internal access only                                │
│  ✅ Stable DNS name                                    │
│  ✅ Load balancing                                     │
│  ❌ No external access                                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**YAML Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```

### 2. NodePort Service

```
┌─────────────────────────────────────────────────────────┐
│                  NODEPORT SERVICE                        │
│                                                         │
│  External Client                                        │
│  ┌─────────────────────────────────────────────────┐   │
│  │  curl <NodeIP>:<NodePort>                       │   │
│  │  Example: curl 192.168.1.100:30080             │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Worker Node                                     │   │
│  │  IP: 192.168.1.100                              │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  NodePort: 30080                         │   │   │
│  │  │  (Listens on all nodes)                  │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │                         │                       │   │
│  │                         ▼                       │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Service (ClusterIP)                     │   │   │
│  │  │  IP: 10.96.0.100                        │   │   │
│  │  │  Port: 80 → 8080                        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │                         │                       │   │
│  │         ┌───────────────┼───────────────┐      │   │
│  │         ▼               ▼               ▼      │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────┐ │   │
│  │  │   Pod 1     │ │   Pod 2     │ │  Pod 3  │ │   │
│  │  └─────────────┘ └─────────────┘ └─────────┘ │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ✅ External access                                    │
│  ✅ Simple setup                                       │
│  ❌ High port range (30000-32767)                      │
│  ⚠️ Needs deliberate exposure and security controls    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**YAML Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-nodeport
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

### 3. LoadBalancer Service

```
┌─────────────────────────────────────────────────────────┐
│                LOADBALANCER SERVICE                       │
│                                                         │
│  Internet                                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  User                                            │   │
│  │  http://a1b2c3d4-1234567890.us-west-2.elb.amazonaws.com│
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  AWS Elastic Load Balancer (ELB)                │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Distributes traffic across nodes      │   │   │
│  │  │  • Health checks                        │   │   │
│  │  │  • SSL termination                      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│         ┌───────────────┼───────────────┐              │
│         ▼               ▼               ▼              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │
│  │  Worker     │ │  Worker     │ │  Worker     │      │
│  │  Node 1     │ │  Node 2     │ │  Node 3     │      │
│  │ ┌─────────┐ │ │ ┌─────────┐ │ │ ┌─────────┐ │      │
│  │ │  Pod 1  │ │ │ │  Pod 2  │ │ │ │  Pod 3  │ │      │
│  │ └─────────┘ │ │ └─────────┘ │ │ └─────────┘ │      │
│  └─────────────┘ └─────────────┘ └─────────────┘      │
│                                                         │
│  ✅ Can be production-suitable with proper controls    │
│  ✅ External load balancing                            │
│  ✅ Health checks                                      │
│  ❌ Cost (cloud provider charges)                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**YAML Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
  annotations:
    # AWS specific annotations
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

### 4. ExternalName Service

```
┌─────────────────────────────────────────────────────────┐
│                EXTERNALNAME SERVICE                       │
│                                                         │
│  Pod in Cluster                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  curl external-db.default.svc.cluster.local      │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  ExternalName Service                           │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Type: ExternalName                      │   │   │
│  │  │  ExternalName: mydb.example.com         │   │   │
│  │  │  (Creates CNAME record)                 │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  External Database                              │   │
│  │  mydb.example.com                               │   │
│  │  (Outside Kubernetes cluster)                   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ✅ Reference external services                        │
│  ✅ Consistent DNS naming                              │
│  ✅ Easy migration (change external name)              │
│  ❌ No load balancing                                  │
│  ❌ No port definitions                                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**YAML Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: mydb.example.com
```

---

## 🔍 Service Discovery and DNS

### Kubernetes DNS

```
┌─────────────────────────────────────────────────────────┐
│              KUBERNETES DNS SYSTEM                        │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  CoreDNS (DNS Server)                           │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Runs in kube-system namespace         │   │   │
│  │  │  • Provides DNS resolution              │   │   │
│  │  │  • Automatically updates                 │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  DNS Naming Convention                          │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Service: <service-name>.<namespace>.svc.cluster.local│
│  │  │  Example: backend-service.default.svc.cluster.local  │
│  │  │                                          │   │   │
│  │  │  Pod: <pod-ip-dashed>.<namespace>.pod.cluster.local  │
│  │  │  Example: 10-244-1-5.default.pod.cluster.local       │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  DNS Resolution Examples:                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  # Same namespace (short form)                  │   │
│  │  curl backend-service                           │   │
│  │                                                  │   │
│  │  # Different namespace                          │   │
│  │  curl backend-service.production.svc.cluster.local│   │
│  │                                                  │   │
│  │  # External service                             │   │
│  │  curl external-db.default.svc.cluster.local     │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Test DNS Resolution

```bash
# Run a test pod
kubectl run dns-test --image=busybox --rm -it -- /bin/sh

# Inside the pod, test DNS
nslookup backend-service
nslookup backend-service.default.svc.cluster.local

# Test service connectivity
wget -qO- http://backend-service
```

---

## 🛠️ Essential kubectl Commands

### Service Commands

```bash
# Create service
kubectl expose deployment nginx --port=80 --type=ClusterIP
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# List services
kubectl get services
kubectl get svc  # Short form
kubectl get svc -o wide

# Describe service
kubectl describe service <service-name>

# Get endpoints
kubectl get endpoints <service-name>

# Delete service
kubectl delete service <service-name>
```

### Networking Commands

```bash
# Test DNS resolution
kubectl run dns-test --image=busybox --rm -it -- nslookup <service-name>

# Test connectivity
kubectl exec -it <pod-name> -- curl <service-name>

# Check network policies
kubectl get networkpolicies

# View kube-proxy rules (on node)
sudo iptables -t nat -L
```

---

## 📝 Practice Exercises

### Exercise 1: Create ClusterIP Service

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
---
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

```bash
# Apply manifests
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml

# Verify
kubectl get svc
kubectl get pods -l app=nginx

# Test DNS resolution
kubectl run test --image=busybox --rm -it -- nslookup nginx-service
```

### Exercise 2: Create NodePort Service

```yaml
# nginx-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

```bash
# Apply
kubectl apply -f nginx-nodeport.yaml

# Get node IP
kubectl get nodes -o wide

# Test external access
curl http://<node-ip>:30080
```

### Exercise 3: Create LoadBalancer Service

```yaml
# nginx-lb.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

```bash
# Apply
kubectl apply -f nginx-lb.yaml

# Wait for external IP
kubectl get svc nginx-lb -w

# Test external access
curl http://<external-ip>
```

---

## 🐛 Troubleshooting Guide

### Service Not Working

```bash
# 1. Check service exists
kubectl get svc <service-name>

# 2. Check endpoints
kubectl get endpoints <service-name>

# 3. Verify selector matches pod labels
kubectl get pods --show-labels

# 4. Test DNS resolution
kubectl run test --image=busybox --rm -it -- nslookup <service-name>

# 5. Test connectivity
kubectl exec -it <pod> -- curl <service-name>
```

### No Endpoints

```bash
# Check if selector matches pod labels
kubectl get pods --show-labels
kubectl get svc <service-name> -o yaml

# Check if pods are running
kubectl get pods

# Check pod readiness
kubectl describe pod <pod-name>
```

---

## 📝 Interview Questions

### Beginner Level

1. **What are the different types of Kubernetes Services?**
   - **Answer:** 
     - ClusterIP: Internal only
     - NodePort: Exposes on node IP
     - LoadBalancer: Creates external load balancer
     - ExternalName: Maps to external DNS

2. **When would you use ClusterIP vs LoadBalancer?**
   - **Answer:** 
     - ClusterIP: For internal communication between services
     - LoadBalancer: For external access to applications

3. **How does kube-proxy work?**
   - **Answer:** kube-proxy maintains network rules on each node. It uses iptables or IPVS to route traffic to the correct pods based on Service definitions.

4. **What is a Headless Service?**
   - **Answer:** A Headless Service (clusterIP: None) doesn't allocate a ClusterIP. Instead, it returns individual pod IPs directly, useful for StatefulSets.

5. **How does service discovery work in Kubernetes?**
   - **Answer:** Kubernetes uses CoreDNS to provide DNS resolution. Services are accessible via `<service-name>.<namespace>.svc.cluster.local`.

### Intermediate Level

6. **What is the difference between ClusterIP and NodePort?**
   - **Answer:** ClusterIP is internal only, accessible within the cluster. NodePort exposes the service on each node's IP at a static port (30000-32767).

7. **How do you implement sticky sessions in Kubernetes?**
   - **Answer:** Set `sessionAffinity: ClientIP` in Service spec. This ensures requests from the same client go to the same pod.

8. **What are ExternalIPs in Kubernetes?**
   - **Answer:** ExternalIPs are IP addresses that can be defined on the Service and routed to cluster nodes, allowing external access without LoadBalancer.

9. **How do you secure a LoadBalancer service?**
   - **Answer:** Use AWS annotations for SSL termination, implement Network Policies, use security groups, and consider using ALB Ingress instead.

10. **What is the difference between Service and Ingress?**
    - **Answer:** Service provides basic load balancing for TCP/UDP. Ingress provides HTTP/HTTPS routing with path-based and host-based routing rules.

---

## 📚 Resources for Today

### Official Documentation
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [DNS for Services](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

---

## 🎯 Today's Assignment

1. **Create:** A ClusterIP service for your deployment
2. **Create:** A NodePort service and test external access
3. **Test:** DNS resolution between services
4. **Document:** Differences between service types

---

## 📊 Progress Tracker

```
Day 1: What is Kubernetes?                    ✅
Day 2: Pods, ReplicaSets, Deployments         ✅
Day 3: Services and Networking                ⬜
Day 4: Namespaces, ConfigMaps, Secrets        ⬜
...
```

---

## 🔗 Next Day Preview

**Day 4:** Namespaces, ConfigMaps, and Secrets
- Organize resources with namespaces
- Manage configuration with ConfigMaps
- Secure sensitive data with Secrets

---

**Remember:** Services are crucial for networking in Kubernetes! 🚀
