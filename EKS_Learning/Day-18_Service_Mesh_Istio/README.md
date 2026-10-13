# 📅 Day 18: Service Mesh with Istio

## 🎯 Learning Objectives
By the end of today, you will understand:
- What is a Service Mesh
- Installing Istio on EKS
- Traffic management
- mTLS security
- Observability tools
- Practical canary deployments

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Service mesh** | A platform layer that manages service-to-service traffic, identity, security, and telemetry. |
| **Sidecar proxy** | A helper proxy running beside an app container and handling its network traffic. |
| **mTLS** | Mutual TLS: both ends authenticate each other with certificates while traffic is encrypted. |
| **Traffic policy** | Rules controlling how requests are routed, retried, balanced, or limited. |
| **Telemetry** | Operational signals such as metrics, logs, and traces. |
| **Control plane** | The components that distribute configuration to proxies; the data plane is the traffic-handling proxies themselves. |

**Analogy:** The mesh is a managed road system, sidecars are traffic officers, and mTLS is checking both drivers' identities before opening a secure channel.

**Try / observe:** `kubectl get pods -n istio-system`, `istioctl analyze`, and inspect proxy configuration only after confirming Istio is installed.

**Troubleshooting clue:** If traffic fails after mesh injection, compare proxy status, mTLS mode, destination rules, and application ports. A healthy app container does not guarantee a healthy proxy path.

**Explain it back:** What does a sidecar do? How does mTLS differ from encryption without peer authentication?

---

## 📖 Core Concepts

### What is a Service Mesh?

A **Service Mesh** is a dedicated infrastructure layer for handling service-to-service communication. It provides traffic management, security, and observability.

```
┌─────────────────────────────────────────────────────────┐
│               ISTIO SERVICE MESH                          │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Data Plane                                      │   │
│  │                                                   │   │
│  │  ┌─────────────────────┐  ┌─────────────────────┐│   │
│  │  │  Service A           │  │  Service B           ││   │
│  │  │  ┌───────────────┐  │  │  ┌───────────────┐  ││   │
│  │  │  │   App Code    │  │  │  │   App Code    │  ││   │
│  │  │  └───────┬───────┘  │  │  └───────┬───────┘  ││   │
│  │  │  ┌───────┴───────┐  │  │  ┌───────┴───────┐  ││   │
│  │  │  │  Envoy Proxy  │◄─┼──┼─►│  Envoy Proxy  │  ││   │
│  │  │  │  (Sidecar)    │  │  │  │  (Sidecar)    │  ││   │
│  │  │  └───────────────┘  │  │  └───────────────┘  ││   │
│  │  └─────────────────────┘  └─────────────────────┘│   │
│  │                                                   │   │
│  └─────────────────────────────────────────────────┘   │
│                          │                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Control Plane                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  istiod                                  │   │   │
│  │  │  - Configuration                        │   │   │
│  │  │  - Certificate management               │   │   │
│  │  │  - Policy enforcement                   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Add-ons                                 │   │   │
│  │  │  - Prometheus (Metrics)                  │   │   │
│  │  │  - Grafana (Visualization)               │   │   │
│  │  │  - Kiali (Traffic Dashboard)            │   │   │
│  │  │  - Jaeger (Tracing)                     │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Ingress/Egress                          │   │   │
│  │  │  - Ingress Gateway                       │   │   │
│  │  │  - Egress Gateway                        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Key Features

```
┌─────────────────────────────────────────────────────────┐
│                 ISTIO KEY FEATURES                       │
│                                                         │
│  🌐 Traffic Management:                                 │
│  • Load balancing                                       │
│  • Canary deployments                                   │
│  • Blue-green deployments                               │
│  • Circuit breakers                                     │
│  • Traffic mirroring                                    │
│                                                         │
│  🔒 Security:                                           │
│  • mTLS (mutual TLS)                                    │
│  • Authorization                                       │
│  • Certificate management                               │
│  • Service identity                                    │
│                                                         │
│  📊 Observability:                                      │
│  • Metrics collection                                   │
│  • Distributed tracing                                  │
│  • Access logs                                          │
│  • Service meshes                                       │
│                                                         │
│  ⚙️ Resilience:                                         │
│  • Timeouts                                             │
│  • Retries                                              │
│  • Circuit breakers                                     │
│  • Fault injection                                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Hands-On: Install Istio

### Step 1: Download Istio

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.20.0

# Add to PATH
export PATH=$PWD/bin:$PATH

# Verify
istioctl version
```

### Step 2: Install base components

```bash
# Install Istio with demo profile (for learning)
istioctl install --set profile=demo -y

# Verify installation
kubectl get pods -n istio-system
```

### Step 3: Install Add-ons

```bash
# Install Kiali, Prometheus, Grafana, Jaeger
kubectl apply -f samples/addons
# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app=kiali -n istio-system --timeout=300s
```

---

## 📝 Practical Lab: Microservices with Canary Deployment

### Step 1: Create BookInfo Services (Example App)

```bash
# Deploy BookInfo sample application
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# Verify all services are running
kubectl get services,deployments,pods
```

### Step 2: Enable Sidecar Injection

```bash
# Label namespace for sidecar injection
kubectl label namespace default istio-injection=enabled

# Restart deployments to inject sidecars
kubectl rollout restart deployment -n default

# Verify sidecars (should have 2 containers)
kubectl get pods -n default
```

### Step 3: Create Ingress Gateway

```yaml
# gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
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
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        prefix: /productpage
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

```bash
kubectl apply -f gateway.yaml

# Get gateway URL
kubectl get svc istio-ingressgateway -n istio-system
```

### Step 4: Canary Deployment with Reviews Service

```yaml
# canary-reviews.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
```

```bash
kubectl apply -f canary-reviews.yaml

# Refresh product page multiple times (about 10% should show v2)
# Open Kiali to visualize traffic
istioctl dashboard kiali
```

### Step 5: Gradual Traffic Shift

```bash
# Shift to 50/50
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v2
      weight: 50
EOF
```

### Step 6: Enable mTLS

```yaml
# mtls.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

```bash
kubectl apply -f mtls.yaml

# All service-to-service traffic is now encrypted
```

---

## 📝 Practical Lab: Traffic Mirroring

```yaml
# mirroring.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-mirror
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
    mirror:
      host: reviews
      subset: v2
    mirrorPercent: 100
```

```bash
# This sends 100% of traffic to v1 but mirrors a copy to v2
kubectl apply -f mirroring.yaml
```

---

## 📝 Practical Lab: Circuit Breaking

```yaml
# circuit-breaker.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: circuit-breaker
spec:
  host: productpage
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        http2MaxRequests: 1
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 30s
      baseEjectionTime: 30s
```

```bash
kubectl apply -f circuit-breaker.yaml
```

---

## 📝 Practice Exercises

### Exercise 1: Fault Injection

```yaml
# fault-injection.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-delay
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 50
        fixedDelay: 5s
    route:
    - destination:
        host: ratings
        subset: v1
```

```bash
# Inject 5s delay on 50% of requests to ratings
kubectl apply -f fault-injection.yaml

# Test - half the requests should be slow
for i in {1..10}; do
  echo "Request $i: $(curl -o /dev/null -s -w '%{http_code} (%{time_total}s)' $PRODUCTPAGE_URL)"
done
```

### Exercise 2: Timeout and Retry

```yaml
# timeout-retry.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-timeout
spec:
  hosts:
  - ratings
  http:
  - timeout: 3s
    route:
    - destination:
        host: ratings
        subset: v1
  - retries:
      attempts: 3
      perTryTimeout: 2s
```

---

## 🐛 Troubleshooting Istio

```bash
# Check sidecar containers
kubectl get pods -n default -o wide

# Inspect sidecar
kubectl exec -it <pod> -c istio-proxy -- pilot-agent request GET stats
kubectl exec -it <pod> -c istio-proxy -- curl localhost:15000/config_dump

# Check Istio control plane
kubectl get pods -n istio-system

# View Kiali dashboard
istioctl dashboard kiali

# View Grafana
istioctl dashboard grafana
```

---

## 📝 Interview Questions

1. **What is Service Mesh and why use it?**
   - **Answer:** Service Mesh provides service-to-service communication with traffic management, security (mTLS), and observability, without modifying application code.

2. **What is the difference between Istio and Kubernetes native networking?**
   - **Answer:** Istio adds a sidecar proxy (Envoy) to each pod providing advanced features like canary deployment, circuit breaking, mTLS, and distributed tracing.

3. **What is mTLS and how does Istio implement it?**
   - **Answer:** mTLS encrypts and authenticates both sides of service-to-service communication. Istio handles certificate issuance and rotation automatically.

4. **How do you do canary deployments with Istio?**
   - **Answer:** Use VirtualService and DestinationRule to route weighted traffic between versions (e.g., 90% v1, 10% v2).

5. **What is Kiali?**
   - **Answer:** Kiali is a dashboard for Istio that visualizes the service mesh, shows traffic flows, and helps with troubleshooting.

---

## 📚 Resources

- [Istio Documentation](https://istio.io/latest/docs/)
- [Istio on EKS](https://docs.aws.amazon.com/eks/latest/userguide/istio.html)

---

## 🎯 Today's Assignment

1. **Install:** Istio on your cluster
2. **Deploy:** BookInfo sample application
3. **Configure:** Canary deployment with traffic splitting
4. **Enable:** mTLS for service security
5. **Visualize:** Traffic with Kiali

---

## 📊 Progress Tracker

```
Day 17: Network Policies with Calico ✅
Day 18: Service Mesh with Istio ⬜
```

---

## 🔗 Next Day Preview

**Day 19:** Advanced Networking Practice
- Combined lab: VPC CNI + Calico + Istio
- Real-world networking scenarios
- Performance testing

---

**Remember:** Service Mesh gives you powerful traffic management and security for microservices! 🚀
