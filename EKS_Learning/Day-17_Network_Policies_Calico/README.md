# 📅 Day 17: Network Policies with Calico

## 🎯 Learning Objectives
By the end of today, you will understand:
- What is Calico and how it works
- Installing Calico on EKS
- Creating Network Policies
- Testing traffic restrictions
- Real-world network policy examples

---

## 📖 Core Concepts

### What is Calico?

**Calico** is a networking and network security solution for Kubernetes. It provides:
- Network Policies (like Kubernetes but more advanced)
- Enhanced routing
- Workload isolation
- Observed network traffic

```
┌─────────────────────────────────────────────────────────┐
│              CALICO ARCHITECTURE                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Calico Components                               │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  calico-kube-controllers                │   │   │
│  │  │  - Watches workloads                    │   │   │
│  │  │  - Syncs policies                       │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  calico-node (DaemonSet)                │   │   │
│  │  │  - Runs on each node                    │   │   │
│  │  │  - Manages iptables                     │   │   │
│  │  │  - Enforces policies                    │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  felix                                  │   │   │
│  │  │  - Programs data plane                  │   │   │
│  │  │  - iptables rules                       │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Traffic Flow                                     │   │
│  │                                                   │   │
│  │  Pod A ──► Pod B (allowed)                       │   │
│  │  Pod C ──► Pod B (blocked - no policy)           │   │
│  │                                                   │   │
│  │  Felix enforces policies via iptables            │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ How to Get Calico on EKS

### Method 1: Install via Add-on

```bash
# Install Calico add-on
eksctl create addon \
  --name "calico-addon" \
  --cluster my-cluster \
  --force
```

### Method 2: Install via Manifest

```bash
# Download Calico manifest
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# Check installation
kubectl get pods -n calico-system
```

---

## 📝 Practical Lab: Network Policies with Calico

### Setup the Lab

```yaml
# lab-apps.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shopping
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
  namespace: shopping
  labels:
    app: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
  namespace: shopping
  labels:
    app: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: shopping
  labels:
    app: db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password123"
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: shopping
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: shopping
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: db-service
  namespace: shopping
spec:
  selector:
    app: db
  ports:
  - port: 3306
    targetPort: 3306
```

```bash
# Apply lab
kubectl apply -f lab-apps.yaml

# Verify all running
kubectl get pods -n shopping -o wide
```

### Step 1: Test Default (All Traffic Allowed)

```bash
# Check connectivity before policies
WEB_POD=$(kubectl get pods -n shopping -l app=web -o jsonpath='{.items[0].metadata.name}')
DB_POD=$(kubectl get pods -n shopping -l app=db -o jsonpath='{.items[0].metadata.name}')

# Web can connect to DB (should work)
kubectl exec -it -n shopping $WEB_POD -- curl -s http://db-service 2>/dev/null || echo "Connection attempted"
```

### Step 2: Apply Default Deny

```yaml
# policy-default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: shopping
spec:
  podSelector: {}  # Applies to all pods
  policyTypes:
  - Ingress
  - Egress
```

```bash
kubectl apply -f policy-default-deny.yaml

# Now all traffic is blocked
kubectl exec -it -n shopping $(kubectl get pods -n shopping -l app=web -o jsonpath='{.items[0].metadata.name}') -- curl -s --connect-timeout 3 http://api-service 2>/dev/null && echo "Connected" || echo "BLOCKED ✓"
```

### Step 3: Allow Web → API Only

```yaml
# policy-web-to-api.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-to-api
  namespace: shopping
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 80
```

```bash
kubectl apply -f policy-web-to-api.yaml

# Web can connect to API (should work)
kubectl exec -it -n shopping $(kubectl get pods -n shopping -l app=web -o jsonpath='{.items[0].metadata.name}') -- curl -s --connect-timeout 3 http://api-service >/dev/null && echo "Web→API: ALLOWED ✓" || echo "FAILED"
```

### Step 4: Allow API → DB Only

```yaml
# policy-api-to-db.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-to-db
  namespace: shopping
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 3306
```

```bash
kubectl apply -f policy-api-to-db.yaml
```

### Step 5: Test All Restrictions

```yaml
# test-pods.yaml
apiVersion: v1
kind: Pod
metadata:
  name: attacker
  namespace: shopping
  labels:
    app: attacker
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
```

```bash
kubectl apply -f test-pods.yaml

# Attacker cannot connect to any service
kubectl exec -it -n shopping attacker -- wget -q --timeout=3 -O- http://api-service 2>/dev/null && echo "Attacker→API: ALLOWED" || echo "Attacker→API: BLOCKED ✓"
kubectl exec -it -n shopping attacker -- wget -q --timeout=3 -O- http://web-service 2>/dev/null && echo "Attacker→Web: ALLOWED" || echo "Attacker→Web: BLOCKED ✓"
```

---

## 🔧 Advanced Calico Policies

### Namespace-Based Policy

```yaml
# namespace-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090
```

### IP Block Policy

```yaml
# ip-block-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-external
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 203.0.113.0/24   # Allowed external IPs
    ports:
    - protocol: TCP
      port: 443
```

### Allow All Ingress from Namespace

```yaml
# allow-ingress-from-ns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-from-dev
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: all
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: development
```

---

## 🐛 Troubleshooting Network Policies

```bash
# Check all network policies
kubectl get networkpolicies -n shopping

# Get details of a policy
kubectl describe networkpolicy web-to-api -n shopping

# Check Calico pods
kubectl get pods -n calico-system

# Check calico-node logs
kubectl logs -n calico-system <calico-node-pod>

# Test connectivity
kubectl run test-pod --image=busybox --rm -it -- wget -q --timeout=3 -O- http://api-service
```

---

## 📝 Interview Questions

1. **What is Calico and why use it?**
   - **Answer:** Calico is a networking and network security solution that provides network policies, advanced routing, and workload isolation. It's more feature-rich than default EKS networking.

2. **How does Calico differ from the default network policy?**
   - **Answer:** Calico provides advanced features like IP blocks, external networks, and more granular controls. It also supports BGP routing.

3. **What is the default deny policy?**
   - **Answer:** Default deny blocks all traffic to/from pods. You must explicitly allow traffic with policies.

4. **How do Network Policies relate to Calico?**
   - **Answer:** Calico implements Kubernetes Network Policies and adds its own `NetworkPolicy` CRDs for advanced features.

5. **How to test if a network policy works?**
   - **Answer:** Deploy test pods (like busybox) and attempt connections to verify allowed/blocked traffic.

---

## 📚 Resources

- [Calico Documentation](https://docs.tigera.io/calico/latest/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [EKS Calico Setup](https://docs.aws.amazon.com/eks/latest/userguide/calico.html)

---

## 🎯 Today's Assignment

1. **Install:** Calico on your cluster
2. **Create:** A default deny policy
3. **Allow:** Web→API→DB traffic only
4. **Test:** Verify attackers are blocked
5. **Document:** Your network policy design

---

## 📊 Progress Tracker

```
Day 16: VPC CNI Plugin ✅
Day 17: Network Policies with Calico ⬜
```

---

## 🔗 Next Day Preview

**Day 18:** Service Mesh with Istio
- Install Istio
- Configure traffic routing
- Implement mTLS
- Observability with Kiali

---

**Remember:** Calico gives you fine-grained control over pod traffic! 🚀
