# 📅 Day 20: Networking Interview Preparation

## 🎯 Learning Objectives
By the end of today, you will:
- Review all networking concepts from Days 1-19
- Practice common interview questions
- Solve real-world networking scenarios
- Complete a networking assessment

---

## 📋 Networking Concepts Review

### Quick Review Quiz

```bash
# Test yourself - answer before revealing

1. What is the default CNI for EKS?
   Answer: VPC CNI (Amazon VPC CNI plugin)

2. What IP range do EKS pods get?
   Answer: IPs from the VPC subnet

3. What are the 4 Service types?
   Answer: ClusterIP, NodePort, LoadBalancer, ExternalName

4. What is the DNS naming convention?
   Answer: <service>.<namespace>.svc.cluster.local

5. What does IRSA mean?
   Answer: IAM Roles for Service Accounts
```

### Networking Components Summary

```
┌─────────────────────────────────────────────────────────┐
│              NETWORKING COMPONENTS SUMMARY                │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Layer 2: VPC CNI                                │   │
│  │  • Assigns VPC IPs to pods                       │   │
│  │  • Direct VPC communication                      │   │
│  │  • No NAT required                               │   │
│  └─────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Layer 3: Network Policies (Calico)              │   │
│  │  • Controls traffic between pods                │   │
│  │  • Default deny                                 │   │
│  │  • Ingress/Egress rules                         │   │
│  └─────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Layer 4: Services & Load Balancing              │   │
│  │  • ClusterIP (internal)                         │   │
│  │  • LoadBalancer (ALB/NLB)                       │   │
│  │  • NodePort (debugging)                         │   │
│  └─────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Layer 7: Service Mesh (Istio)                   │   │
│  │  • Traffic management                           │   │
│  │  • mTLS security                                │   │
│  │  • Observability                                │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 📝 Interview Scenario Practice

### Scenario 1: Applications Can't Communicate

**Problem:** Two services in the same namespace can't communicate.

**Debugging Process:**
```bash
# Step 1: Check if services exist
kubectl get svc -n <namespace>

# Step 2: Check endpoints
kubectl get endpoints -n <namespace>

# Step 3: Check network policies
kubectl get networkpolicies -n <namespace>

# Step 4: Test DNS
kubectl run test --image=busybox --rm -it -n <namespace> -- nslookup <service>

# Step 5: Test direct connection
kubectl exec -it <pod> -n <namespace> -- wget -q --timeout=3 -O- http://<service>
```

**Common Causes:**
- Network policy blocking traffic
- Selector not matching pod labels
- Service not having endpoints
- DNS issues
- Wrong port

### Scenario 2: External Access Not Working

**Problem:** ALB is created but can't access application externally.

**Debugging Process:**
```bash
# Step 1: Check ingress
kubectl get ingress -n <namespace>

# Step 2: Check ALB controller logs
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# Step 3: Check security groups
# Verify ALB SG allows traffic

# Step 4: Check target group
# Check pods are registered and healthy
```

### Scenario 3: High Latency

**Problem:** High latency on specific service.

**Debugging Process:**
```bash
# Step 1: Check request latency
kubectl top pods -n <namespace>

# Step 2: Check network metrics
# Look at container network receive/transmit bytes

# Step 3: Check DNS resolution time
kubectl exec -it <pod> -- time nslookup <service>

# Step 4: Check istio metrics
istioctl dashboard grafana

# Step 5: Check node resources
kubectl top nodes
```

---

## 📝 Interview Questions - Comprehensive

### Beginner Level

1. **How does Kubernetes networking work?**
```
Answer: 
- Flat network model
- Every pod gets its own IP
- Pods can reach all other pods
- Services provide stable access
```

2. **What is the difference between ClusterIP and LoadBalancer?**
```
Answer:
- ClusterIP: Internal only, service-to-service
- LoadBalancer: External access via AWS ELB
```

3. **What is kube-proxy?**
```
Answer:
- Runs on every node
- Maintains network rules
- Handles service networking
- Uses iptables or IPVS
```

4. **How does service discovery work?**
```
Answer:
- Uses DNS (CoreDNS)
- Format: <service>.<namespace>.svc.cluster.local
- Pods resolve services by name
```

5. **What is the difference between Ingress and Service?**
```
Answer:
- Service: TCP/UDP load balancing
- Ingress: HTTP/HTTPS routing with paths and hosts
```

### Intermediate Level

6. **How does VPC CNI assign IPs to pods?**
```
Answer:
- Uses ENIs attached to nodes
- Assigns secondary IPs from VPC subnet
- Warm pool for fast pod creation
- Direct VPC communication
```

7. **How do you implement Network Policies?**
```
Answer:
- Install CNI that supports policies (Calico)
- Create NetworkPolicy resources
- Use selectors for traffic rules
- Apply default deny for security
```

8. **How do you do canary deployments?**
```
Answer:
- Use Istio VirtualService
- Weight-based routing (10% v2)
- Monitor metrics
- Gradually increase weight
```

9. **What is mTLS and why use it?**
```
Answer:
- Mutual TLS: both sides authenticate
- Encrypts service-to-service traffic
- Prevents MITM attacks
- Managed by Istio
```

10. **How does ALB Ingress integrate with EKS?**
```
Answer:
- AWS Load Balancer Controller
- Creates ALB automatically
- Routes to nodes (instance mode)
- Routes to pods (IP mode)
```

### Senior Level

11. **Design a secure multi-service networking architecture.**
```
Answer:
- Private subnets for workers
- VPC CNI for pod networking
- Default deny network policies
- Calico for advanced policies
- Istio for mTLS and traffic management
- ALB/NLB for external access
- CloudWatch/others for monitoring
```

12. **How do you handle DNS resolution at scale?**
```
Answer:
- Use CoreDNS (default)
- Configure autoscaling for CoreDNS
- Use node-local DNS
- Cache responses
```

13. **How do you troubleshoot latency issues?**
```
Answer:
- Measure request time
- Check service-to-service latency
- Use distributed tracing
- Check node resource usage
- Check network throughput
```

14. **What are the best practices for service discovery?**
```
Answer:
- Use meaningful service names
- Namespace-based organization
- ExternalName for external services
- Headless services for stateful apps
```

15. **How do you secure cluster networking?**
```
Answer:
- Public endpoint disabled
- Private subnets
- Network policies
- mTLS
- Security groups
- Allowlists in security groups
```

---

## 📝 Practical Assessment

### Task 1: Design Network Architecture

Create a diagram for a 3-tier application:
- **Frontend tier:** Web UI
- **Application tier:** API service  
- **Data tier:** Database

**Requirements:**
- External access through ALB
- Internal service communication via ClusterIP
- Only frontend can reach API
- Only API can reach database
- All DNS traffic allowed
- mTLS enabled

### Task 2: Debug the Issue

You have the following setup:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: webserver  # Wrong label
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: web  # Actual label
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
```

**Find the bug and fix it:**
- Service selector uses `app: webserver`
- Pod label uses `app: web`
- Fix: Update service selector to match

```bash
# Fix
kubectl edit svc web-service
# Change selector to app: web

# Verify
kubectl get endpoints web-service
```

### Task 3: Load Test Your Setup

```bash
# Install hey (if not installed)
# Windows
choco install hey

# Test 5000 requests with 50 concurrent
hey -n 5000 -c 50 http://<your-service>

# Test with custom headers
hey -n 1000 -c 10 -H "Content-Type: application/json" http://<your-service>/api
```

---

## 📋 Interview Question Bank

### Must-Know Answers (For Quick Review)

1. **"What is the default CNI in EKS?"**
   - VPC CNI - Assigns VPC IPs to pods

2. **"How do pods get IP addresses?"**
   - From VPC subnet via ENIs

3. **"What is the difference between NodePort and LoadBalancer?"**
   - NodePort: Fixed port on node, no cloud LB
   - LoadBalancer: Creates AWS ELB

4. **"How do you secure pod-to-pod traffic?"**
   - Network policies + mTLS

5. **"What metrics do you monitor?"**
   - RED: Rate, Error rate, Duration

---

## 🎯 Today's Assignment

1. **Review:** All networking concepts
2. **Practice:** Debugging scenarios
3. **Solve:** 3-5 interview questions aloud
4. **Design:** A network architecture for your app
5. **Test:** Your cluster performance

---

## 📊 Progress Tracker

```
Week 3-4 Networking Complete:
Day 16: VPC CNI Plugin ✅
Day 17: Network Policies with Calico ✅
Day 18: Service Mesh with Istio ✅
Day 19: Advanced Networking Practice ✅
Day 20: Networking Interview Preparation ⬜
```

---

## 🔗 Next Week Preview

**Week 5-6:** Advanced Storage
- Day 21: EFS CSI Driver
- Day 22: FSx for Lustre
- Day 23: S3 CSI Driver
- Day 24: Volume Snapshots and Restore
- Day 25: Advanced Storage Practice

---

**Remember:** Practice makes perfect! Review concepts and practice interview questions regularly. 🚀
