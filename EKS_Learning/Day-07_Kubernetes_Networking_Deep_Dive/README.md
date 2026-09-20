# 📅 Day 7: Kubernetes Networking Deep Dive

## 🎯 Learning Objectives
By the end of today, you will understand:
- How Kubernetes networking works
- Pod-to-Pod communication
- Service discovery and DNS
- Network Policies
- CNI (Container Network Interface)

---

## 📖 Core Concepts

### Kubernetes Networking Model

```
┌─────────────────────────────────────────────────────────┐
│              KUBERNETES NETWORKING MODEL                  │
│                                                         │
│  Key Principles:                                        │
│  ┌─────────────────────────────────────────────────┐   │
│  │  1. Every Pod gets its own IP address            │   │
│  │  2. Pods can communicate with all other Pods     │   │
│  │  3. Agents on a node can communicate with all    │   │
│  │     Pods on that node                            │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │                    Node 1                         │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Pod A (10.244.1.5)                      │   │   │
│  │  │  ┌─────────────────────────────────┐   │   │   │
│  │  │  │  eth0: 10.244.1.5               │   │   │   │
│  │  │  │  Gateway: 10.244.1.1            │   │   │   │
│  │  │  └─────────────────────────────────┘   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Pod B (10.244.1.6)                      │   │   │
│  │  │  ┌─────────────────────────────────┐   │   │   │
│  │  │  │  eth0: 10.244.1.6               │   │   │   │
│  │  │  │  Gateway: 10.244.1.1            │   │   │   │
│  │  │  └─────────────────────────────────┘   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  CNI Plugin (Calico/Flannel/Weave)       │   │   │
│  │  │  Manages networking for all pods         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Pod-to-Pod Communication

```
┌─────────────────────────────────────────────────────────┐
│              POD-TO-POD COMMUNICATION                     │
│                                                         │
│  Same Node Communication:                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Node 1                                          │   │
│  │  ┌─────────────┐      ┌─────────────┐          │   │
│  │  │  Pod A      │ ───► │  Pod B      │          │   │
│  │  │ 10.244.1.5  │      │ 10.244.1.6  │          │   │
│  │  └─────────────┘      └─────────────┘          │   │
│  │                                                   │   │
│  │  Traffic stays within node (fast)                 │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Cross-Node Communication:                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Node 1                    Node 2                 │   │
│  │  ┌─────────────┐         ┌─────────────┐       │   │
│  │  │  Pod A      │ ──────► │  Pod C      │       │   │
│  │  │ 10.244.1.5  │         │ 10.244.2.3  │       │   │
│  │  └─────────────┘         └─────────────┘       │   │
│  │                                                   │   │
│  │  Traffic routes through network (slower)          │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### CNI (Container Network Interface)

```
┌─────────────────────────────────────────────────────────┐
│                CNI ARCHITECTURE                           │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Kubernetes API Server                            │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  CNI Plugin                                      │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Configures networking for pods        │   │   │
│  │  │  • Assigns IP addresses                  │   │   │
│  │  │  • Sets up routes                        │   │   │
│  │  │  • Handles pod network isolation         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│         ┌───────────────┼───────────────┐              │
│         ▼               ▼               ▼              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │
│  │  Calico     │ │  Flannel    │ │  Weave      │      │
│  │  ─────────  │ │  ─────────  │ │  ─────────  │      │
│  │  Network    │ │  Overlay    │ │  Overlay    │      │
│  │  Policies   │ │  Network    │ │  Network    │      │
│  │  BGP        │ │  VXLAN      │ │  Mesh       │      │
│  └─────────────┘ └─────────────┘ └─────────────┘      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🔍 Service Discovery

### How Services Work

```
┌─────────────────────────────────────────────────────────┐
│              SERVICE DISCOVERY FLOW                       │
│                                                         │
│  1. Service Created                                     │
│  ┌─────────────────────────────────────────────────┐   │
│  │  kubectl expose deployment nginx --port=80       │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  2. Service Gets ClusterIP and DNS Entry                │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Service: nginx-service                          │   │
│  │  ClusterIP: 10.96.0.100                          │   │
│  │  DNS: nginx-service.default.svc.cluster.local    │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  3. Pods Get Endpoints                                  │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Endpoints:                                      │   │
│  │  - 10.244.1.5:80                                │   │
│  │  - 10.244.2.3:80                                │   │
│  │  - 10.244.1.8:80                                │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  4. Traffic Routing                                     │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Client → Service (10.96.0.100) → Pod (any)     │   │
│  │                                                   │   │
│  │  kube-proxy handles load balancing                │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### DNS Resolution

```
┌─────────────────────────────────────────────────────────┐
│                KUBERNETES DNS                            │
│                                                         │
│  DNS Naming Convention:                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │  <service-name>.<namespace>.svc.cluster.local   │   │
│  │                                                   │   │
│  │  Examples:                                       │   │
│  │  • nginx-service.default.svc.cluster.local      │   │
│  │  • mysql-service.database.svc.cluster.local     │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Short Forms:                                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │  • Same namespace: <service-name>               │   │
│  │  • Different namespace: <service-name>.<ns>     │   │
│  │  • External: <external-name>                    │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  CoreDNS:                                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  • Runs in kube-system namespace               │   │
│  │  • Provides DNS resolution                      │   │
│  │  • Automatically updates when services change   │   │
│  │  • Configurable with custom forwarders          │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛡️ Network Policies

### What are Network Policies?

**Network Policies** are a Kubernetes resource that controls traffic flow between pods, namespaces, and external endpoints.

### Network Policy Architecture

```
┌─────────────────────────────────────────────────────────┐
│              NETWORK POLICY ARCHITECTURE                  │
│                                                         │
│  Without Network Policy:                                │
│  ┌─────────────────────────────────────────────────┐   │
│  │  All pods can communicate with all pods          │   │
│  │                                                   │   │
│  │  ┌─────┐     ┌─────┐     ┌─────┐              │   │
│  │  │ Pod │ ◄─► │ Pod │ ◄─► │ Pod │              │   │
│  │  │  A  │     │  B  │     │  C  │              │   │
│  │  └─────┘     └─────┘     └─────┘              │   │
│  │    ▲           ▲           ▲                    │   │
│  │    └───────────┴───────────┘                    │   │
│  │         No restrictions                         │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  With Network Policy:                                   │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Only allowed traffic is permitted               │   │
│  │                                                   │   │
│  │  ┌─────┐     ┌─────┐     ┌─────┐              │   │
│  │  │ Pod │ ──► │ Pod │ ──► │ Pod │              │   │
│  │  │  A  │     │  B  │     │  C  │              │   │
│  │  └─────┘     └─────┘     └─────┘              │   │
│  │    ▲           ▲           ▲                    │   │
│  │    └───────────┴───────────┘                    │   │
│  │         Controlled access                       │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Network Policy YAML Examples

```yaml
# deny-all.yaml - Default deny all traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

```yaml
# allow-frontend-to-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

```yaml
# allow-dns-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

---

## 🛠️ Essential kubectl Commands

```bash
# Network Policies
kubectl get networkpolicies
kubectl get netpol  # Short form
kubectl describe networkpolicy <name>
kubectl apply -f network-policy.yaml

# Test DNS
kubectl run dns-test --image=busybox --rm -it -- nslookup <service-name>

# Test connectivity
kubectl exec -it <pod-name> -- curl <service-name>
kubectl exec -it <pod-name> -- ping <pod-ip>

# Check kube-proxy rules (on node)
sudo iptables -t nat -L -n
```

---

## 📝 Practice Exercises

### Exercise 1: Test Pod-to-Pod Communication

```bash
# Create two deployments
kubectl create deployment nginx1 --image=nginx --replicas=2
kubectl create deployment nginx2 --image=nginx --replicas=2

# Expose them
kubectl expose deployment nginx1 --port=80
kubectl expose deployment nginx2 --port=80

# Test DNS resolution
kubectl run test --image=busybox --rm -it -- nslookup nginx1
kubectl run test --image=busybox --rm -it -- nslookup nginx2

# Test connectivity
kubectl exec -it <nginx2-pod> -- curl nginx1
```

### Exercise 2: Implement Network Policy

```yaml
# default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# allow-frontend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from: []
  egress:
  - to: []
```

```bash
# Apply network policies
kubectl apply -f default-deny.yaml
kubectl apply -f allow-frontend.yaml

# Test connectivity (should fail for non-frontend pods)
kubectl exec -it <backend-pod> -- curl frontend-service
```

---

## 🐛 Troubleshooting Guide

### Pod Cannot Connect to Another Pod

```bash
# 1. Check DNS resolution
kubectl run test --image=busybox --rm -it -- nslookup <service-name>

# 2. Check service endpoints
kubectl get endpoints <service-name>

# 3. Check network policies
kubectl get networkpolicies

# 4. Test connectivity directly
kubectl exec -it <pod> -- ping <pod-ip>

# 5. Check kube-proxy logs
kubectl logs -n kube-system <kube-proxy-pod>
```

### DNS Resolution Issues

```bash
# 1. Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 2. Check CoreDNS logs
kubectl logs -n kube-system <coredns-pod>

# 3. Test DNS from pod
kubectl exec -it <pod> -- nslookup kubernetes.default
```

---

## 📝 Interview Questions

1. **How does Kubernetes networking work?**
   - **Answer:** Kubernetes networking follows a flat network model where every Pod gets its own IP. Pods can communicate with all other Pods without NAT. CNI plugins handle the actual networking implementation.

2. **What is CNI (Container Network Interface)?**
   - **Answer:** CNI is a specification for configuring network interfaces in Linux containers. Kubernetes uses CNI plugins (Calico, Flannel, Weave) to implement networking.

3. **What are Network Policies?**
   - **Answer:** Network Policies control traffic flow between pods, namespaces, and external endpoints. They provide network-level security and isolation.

4. **How does service discovery work in Kubernetes?**
   - **Answer:** Kubernetes uses CoreDNS for service discovery. Services get DNS entries in the format `<service-name>.<namespace>.svc.cluster.local`. Pods can resolve services using DNS.

5. **What is the difference between ClusterIP and Headless Service?**
   - **Answer:** ClusterIP provides a single virtual IP and load balances across pods. Headless Service (clusterIP: None) returns individual pod IPs directly.

---

## 📚 Resources

- [Kubernetes Networking](https://kubernetes.io/docs/concepts/services-networking/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [DNS for Services](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

---

## 🎯 Today's Assignment

1. **Test:** Pod-to-Pod communication across nodes
2. **Test:** Service DNS resolution
3. **Implement:** A Network Policy to restrict traffic
4. **Document:** Kubernetes networking concepts

---

## 📊 Progress Tracker

```
Week 1 Complete:
Day 1: What is Kubernetes?                    ✅
Day 2: Pods, ReplicaSets, Deployments         ✅
Day 3: Services and Networking                ✅
Day 4: Namespaces, ConfigMaps, Secrets        ✅
Day 5: Persistent Storage                     ✅
Day 6: ConfigMaps and Secrets Practice        ✅
Day 7: Kubernetes Networking Deep Dive        ✅
```

---

## 🔗 Next Week Preview

**Week 2:** AWS EKS Fundamentals
- Day 8: Introduction to Amazon EKS
- Day 9: EKS Cluster Setup with eksctl
- Day 10: EKS Networking and VPC
- Day 11: EKS IAM Roles and Policies
- Day 12: EKS Storage with EBS
- Day 13: EKS Load Balancing with ALB
- Day 14: EKS Monitoring and Logging
- Day 15: EKS Security Best Practices

---

**Remember:** Networking is fundamental to Kubernetes! Practice with real clusters to understand the concepts better. 🚀
