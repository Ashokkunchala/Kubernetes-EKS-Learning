# 📅 Day 19: Advanced Networking Practice

## 🎯 Learning Objectives
By the end of today, you will understand:
- Combined networking setup (VPC CNI + Calico + Istio)
- Real-world networking scenarios
- Performance testing
- Security testing
- Complete microservices networking

---
## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **DNS resolution** | Translating a name into an IP address or other DNS record. |
| **Routing** | Choosing the next network hop for packets. |
| **Security group** | An AWS stateful firewall associated with network interfaces/resources. |
| **NetworkPolicy** | A Kubernetes rule that can limit Pod-to-Pod or other Pod traffic when supported by the CNI. |
| **Packet path** | The full sequence of interfaces, routes, policies, and proxies crossed by traffic. |

**Analogy:** Troubleshooting a network is like tracing a delivery route one checkpoint at a time; do not assume the last checkpoint is the problem.

**Try / observe:** `kubectl get svc,endpointslices -A`, `kubectl get pods -o wide -A`, `kubectl describe networkpolicy -n <namespace>`; test DNS and connectivity from a temporary Pod.

**Troubleshooting clue:** Check in order: name resolution → Service selector/endpoints → Pod readiness/listener → Kubernetes policy → AWS security groups/routes. Use the order as a guide, not a guarantee.

**Explain it back:** Why should you test name resolution separately from TCP connectivity?

---


## 🖥️ Project: Complete E-Commerce Network Setup

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│        E-COMMERCE NETWORK ARCHITECTURE                   │
│                                                         │
│  Internet ────► Istio Ingress Gateway                   │
│                        │                                 │
│                        ▼                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │          Istio VirtualService                     │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  /frontend → productpage service        │   │   │
│  │  │  /api      → api-gateway service        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│         ┌───────────────┼───────────────┐              │
│         ▼               ▼               ▼              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │
│  │  Frontend   │ │  API        │ │  Auth       │      │
│  │  (React)    │ │  Gateway    │ │  Service    │      │
│  │  └─────┐    │ │  └─────┐    │ │  └─────┐    │      │
│  └─────────┘    │  └──────┴────┘    │  └─────┘    │      │
│                 │         │         │             │      │
│                 ▼         ▼         ▼             │      │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  │      │
│  │  Product    │ │  Order      │ │  Payment    │  │      │
│  │  Service    │ │  Service    │ │  Service    │  │      │
│  │  └─────┐    │ │  └─────┐    │ │  └─────┐    │  │      │
│  └─────────┘    │  └──────┴────┘    │  └─────┘    │      │
│                 │         │         │             │      │
│                 │         ▼         │             │      │
│                 │  ┌─────────────┐  │             │      │
│                 │  │  Database   │  │             │      │
│                 │  │  (MySQL)    │  │             │      │
│                 │  └─────────────┘  │             │      │
│                 └───────────────────┘             │      │
│                                                         │
│  Network Policies:                                      │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Default: Deny all                               │   │
│  │  Allow: Frontend → API                           │   │
│  │  Allow: API → Product, Order, Payment            │   │
│  │  Allow: API → Database                           │   │
│  │  Allow: All → DNS                                │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 📝 Step-by-Step Implementation

### Step 1: Create Namespaces and Services

```yaml
# ecommerce-setup.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
  labels:
    istio-injection: enabled
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ecommerce-sa
  namespace: ecommerce
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ecommerce
  labels:
    app: frontend
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        version: v1
    spec:
      serviceAccountName: ecommerce-sa
      containers:
      - name: frontend
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: ecommerce
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: ecommerce
  labels:
    app: api
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
        version: v1
    spec:
      serviceAccountName: ecommerce-sa
      containers:
      - name: api
        image: python:3.9-alpine
        command: ["python", "-m", "http.server", "8080"]
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  namespace: ecommerce
spec:
  selector:
    app: api
  ports:
  - port: 8080
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: ecommerce
  labels:
    app: product
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: product
  template:
    metadata:
      labels:
        app: product
        version: v1
    spec:
      serviceAccountName: ecommerce-sa
      containers:
      - name: product
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: ecommerce
spec:
  selector:
    app: product
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: ecommerce
  labels:
    app: db
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
        version: v1
    spec:
      serviceAccountName: ecommerce-sa
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "REPLACE_WITH_STRONG_LAB_PASSWORD"
        ports:
        - containerPort: 3306
---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: ecommerce
spec:
  selector:
    app: db
  ports:
  - port: 3306
    targetPort: 3306
```

```bash
kubectl apply -f ecommerce-setup.yaml

# Verify all pods running with 2 containers (app + istio-proxy)
kubectl get pods -n ecommerce
```

### Step 2: Apply Network Policies

```yaml
# network-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: ecommerce
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: TCP
      port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-api
  namespace: ecommerce
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
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-services
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      app: product
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-db
  namespace: ecommerce
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
kubectl apply -f network-policies.yaml
```

### Step 3: Configure Istio Routing

```yaml
# istio-config.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ecommerce-gateway
  namespace: ecommerce
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ecommerce-routes
  namespace: ecommerce
spec:
  hosts:
  - "*"
  gateways:
  - ecommerce-gateway
  http:
  - match:
    - uri:
        prefix: /frontend
    route:
    - destination:
        host: frontend
        port:
          number: 80
  - match:
    - uri:
        prefix: /api
    route:
    - destination:
        host: api-gateway
        port:
          number: 8080
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: api-gateway
  namespace: ecommerce
spec:
  host: api-gateway
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 10
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 60s
```

```bash
kubectl apply -f istio-config.yaml

# Get gateway URL
kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### Step 4: Enable mTLS

```yaml
# mtls.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: ecommerce-mtls
  namespace: ecommerce
spec:
  mtls:
    mode: STRICT
```

```bash
kubectl apply -f mtls.yaml
```

### Step 5: Test the Setup

```bash
# Get gateway URL
GATEWAY=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Gateway: $GATEWAY"

# Test frontend
curl -o /dev/null -s -w "%{http_code}\n" http://$GATEWAY/frontend

# Test api
curl -o /dev/null -s -w "%{http_code}\n" http://$GATEWAY/api

# Check connectivity between services
kubectl exec -it -n ecommerce $(kubectl get pods -n ecommerce -l app=frontend -o jsonpath='{.items[0].metadata.name}') -c frontend -- wget -q --timeout=3 -O- http://api-gateway:8080 && echo "Frontend→API: OK" || echo "Frontend→API: BLOCKED"

# Verify mTLS
istioctl proxy-config secret $(kubectl get pods -n ecommerce -l app=api -o jsonpath='{.items[0].metadata.name}') -n ecommerce
```

### Step 6: Performance Testing

```bash
# Install hey (load testing tool)
# macOS: brew install hey
# Windows: choco install hey

# Test gateway performance
hey -n 1000 -c 50 http://$GATEWAY/frontend

# Check Istio metrics
istioctl dashboard grafana

# Check traffic visualization
istioctl dashboard kiali
```

---

## 🐛 Advanced Troubleshooting

```bash
# Check all services in mesh
istioctl proxy-status

# Get proxy config
istioctl proxy-config route <pod-name> -n ecommerce

# Crash loop in sidecar
kubectl logs -n ecommerce <pod-name> -c istio-proxy --previous

# Check mTLS certificates
istioctl proxy-config secret <pod-name> -n ecommerce

# Network policy issues
kubectl get networkpolicies -n ecommerce
kubectl describe networkpolicy <policy-name>
```

---

## 📝 Interview Questions

1. **How do you combine VPC CNI, Calico, and Istio?**
   - **Answer:** VPC CNI for pod networking, Calico for network policies, Istio for service mesh (mTLS, traffic management, observability).

2. **What are the best practices for microservices networking?**
   - **Answer:** Use private subnets, default deny network policies, mTLS for encryption, load balancing, circuit breakers, and monitoring.

3. **How do you handle security across services?**
   - **Answer:** Network policies for L3/L4 security, Istio mTLS for L7 security, RBAC for access control, and secrets management.

4. **What metrics should you monitor?**
   - **Answer:** Request rate, error rate, latency (RED), service availability, resource utilization, and request distribution.

5. **How do you test network configurations?**
   - **Answer:** Deploy test pods, attempt connections, verify blocked/allowed traffic, monitor with Kiali/istioctl, and load testing.

---

## 📚 Resources

- [EKS Networking Best Practices](https://docs.aws.amazon.com/eks/latest/userguide/network-security.html)
- [Istio Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)

---

## 🎯 Today's Assignment

1. **Build:** The complete e-commerce network setup
2. **Apply:** Network policies (default deny)
3. **Configure:** Istio routing and mTLS
4. **Test:** Connectivity and security
5. **Document:** Your networking architecture

---

## 📊 Progress Tracker

```
Day 18: Service Mesh with Istio ✅
Day 19: Advanced Networking Practice ⬜
```

---

## 🔗 Next Day Preview

**Day 20:** Networking Interview Preparation
- Review all networking concepts
- Common interview questions
- Practice scenarios

---

**Remember:** Combine all networking tools for a secure, efficient, and observable cluster! 🚀
