# 📅 Day 13: EKS Load Balancing with ALB

## 🎯 Learning Objectives
By the end of today, you will understand:
- AWS Load Balancer Controller
- ALB Ingress configuration
- NLB Service setup
- Target Group Binding
- Hands-on with load balancing

---

## 🧠 Plain-English terms before you start

| Term | Meaning | Why it matters |
|---|---|---|
| **ALB (Application Load Balancer)** | An AWS load balancer that routes HTTP/HTTPS requests using rules such as hostnames and paths. | It provides an entry point and spreads requests across healthy targets. |
| **Ingress** | A Kubernetes object describing HTTP/HTTPS routing rules. | It expresses desired routing, but does not implement it alone. |
| **Ingress controller** | A controller that watches Ingress objects and configures a real data plane/load balancer. | Without a compatible controller, Ingress rules may do nothing. |
| **Target group** | An AWS grouping of registered destinations that receive load-balanced traffic. | Health checks and target registration determine whether requests can reach Pods. |
| **Health check** | A repeated test used to decide whether a target is ready to receive traffic. | A path, port, or expected status mismatch can make a healthy app appear unhealthy. |
| **TLS certificate** | A certificate used to authenticate a server and enable encrypted HTTPS connections. | HTTPS needs correct certificate, hostname, and listener configuration. |

**Analogy:** Ingress is the routing plan, the controller is the engineer implementing it, and the ALB is the traffic dispatcher.

**Try / observe:** `kubectl get ingress,svc,endpointslices -A`; inspect the AWS load balancer, listener, target group, and health-check reason before changing manifests.

**Troubleshooting clue:** If an ALB target is unhealthy, compare the health-check path/port and expected response with the application's actual response and the security-group path.

**Explain it back:** What is the difference between an Ingress and an Ingress controller? Why can a target be registered but unhealthy?

---

## 📖 Core Concepts

### AWS Load Balancer Controller

```
┌─────────────────────────────────────────────────────────┐
│           AWS LOAD BALANCER CONTROLLER                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Controller Components                           │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Deployment (controller)              │   │   │
│  │  │  • ServiceAccount                        │   │   │
│  │  │  • ClusterRole/ClusterRoleBinding       │   │   │
│  │  │  • Webhook                               │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Features                                        │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • ALB Ingress                          │   │   │
│  │  │  • NLB Service                          │   │   │
│  │  │  • Target Group Binding                  │   │   │
│  │  │  • IP mode for pods                     │   │   │
│  │  │  • WAF integration                      │   │   │
│  │  │  • Shield integration                   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Architecture                                    │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Ingress/Service → Controller → AWS ELB │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### ALB Ingress Architecture

```
┌─────────────────────────────────────────────────────────┐
│              ALB INGRESS ARCHITECTURE                     │
│                                                         │
│  Internet                                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  User                                            │   │
│  │  http://myapp.example.com                       │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Application Load Balancer (ALB)                 │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Listener Rules:                         │   │   │
│  │  │  - /app/* → Target Group 1              │   │   │
│  │  │  - /api/* → Target Group 2              │   │   │
│  │  │  - /*     → Target Group 3              │   │   │
│  │  │                                          │   │   │
│  │  │  Features:                               │   │   │
│  │  │  - SSL termination                      │   │   │
│  │  │  - Path-based routing                   │   │   │
│  │  │  - Host-based routing                   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│         ┌───────────────┼───────────────┐              │
│         ▼               ▼               ▼              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │
│  │  Target     │ │  Target     │ │  Target     │      │
│  │  Group 1    │ │  Group 2    │ │  Group 3    │      │
│  │  (frontend) │ │  (api)      │ │  (backend)  │      │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘      │
│         │               │               │              │
│         ▼               ▼               ▼              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │
│  │  Pods       │ │  Pods       │ │  Pods       │      │
│  │  (frontend) │ │  (api)      │ │  (backend)  │      │
│  └─────────────┘ └─────────────┘ └─────────────┘      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Step-by-Step Setup

### Step 1: Install AWS Load Balancer Controller

```bash
# Install using Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install the controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-eks-cluster \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Step 2: Create ALB Ingress

```yaml
# alb-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-west-2:ACCOUNT:certificate/CERT_ID
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

### Step 3: Deploy Applications

```yaml
# frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
---
# api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
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
      - name: api
        image: my-api:v1.0.0
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 8080
    targetPort: 8080
```

### Step 4: Test

```bash
# Apply manifests
kubectl apply -f frontend-deployment.yaml
kubectl apply -f api-deployment.yaml
kubectl apply -f alb-ingress.yaml

# Get ALB URL
kubectl get ingress my-ingress

# Test
curl http://<ALB-DNS>
```

---

## 📝 Practice Exercises

### Exercise 1: Create ALB Ingress with Path Routing

```yaml
# path-routing.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  - http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
      - path: /products
        pathType: Prefix
        backend:
          service:
            name: product-service
            port:
              number: 80
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
```

---

## 📝 Interview Questions

1. **What is the difference between ALB and NLB?**
   - **Answer:** ALB for HTTP/HTTPS with path-based routing. NLB for TCP/UDP with ultra-low latency.

2. **How does the AWS Load Balancer Controller work?**
   - **Answer:** It watches Ingress/Service resources and creates/manages AWS load balancers automatically.

3. **What are Ingress annotations?**
   - **Answer:** Annotations configure ALB behavior (scheme, target type, SSL, WAF integration).

4. **How to configure SSL/TLS on ALB?**
   - **Answer:** Use ACM certificate ARN in annotations. Configure HTTPS listener with SSL termination.

5. **What is Target Group Binding?**
   - **Answer:** Binds a Kubernetes Service to an existing AWS Target Group for external load balancers.

---

## 📚 Resources

- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [ALB Ingress](https://kubernetes-sigs.github.io/aws-load-balancer-controller/guide/ingress/spec/)

---

## 🎯 Today's Assignment

1. **Install:** AWS Load Balancer Controller
2. **Create:** An ALB Ingress with path routing
3. **Deploy:** Multiple services behind the ALB
4. **Test:** External access to your applications

---

## 📊 Progress Tracker

```
Day 12: EKS Storage with EBS ✅
Day 13: EKS Load Balancing with ALB ⬜
```

---

## 🔗 Next Day Preview

**Day 14:** EKS Monitoring and Logging
- CloudWatch Container Insights
- Prometheus and Grafana
- AWS X-Ray for tracing
- Hands-on with monitoring

---

**Remember:** ALB is the recommended way to expose applications externally on EKS! 🚀
