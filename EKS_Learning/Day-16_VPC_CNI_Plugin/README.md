# 📅 Day 16: VPC CNI Plugin

## 🎯 Learning Objectives
By the end of today, you will understand:
- What is VPC CNI Plugin
- How VPC CNI assigns IPs
- Custom networking (new architecture)
- Prefix delegation
- Security groups for pods

---

## 🧠 Plain-English terms before you start

| Term | Meaning | Why it matters |
|---|---|---|
| **CNI** | The interface Kubernetes uses to configure Pod networking. | A broken CNI can prevent Pods from receiving network access. |
| **Amazon VPC CNI** | The EKS networking plugin that assigns VPC addresses to Pods. | Pod IP capacity depends on subnet addresses and EC2 network-interface limits/configuration. |
| **ENI** | An Elastic Network Interface, a virtual network card in AWS. | The number and configuration of ENIs affect available Pod IP capacity. |
| **IPAM** | IP Address Management: allocating and tracking addresses. | Address exhaustion can block Pod creation even when CPU and memory are available. |
| **Prefix delegation** | Assigning IP address prefixes to interfaces rather than allocating every address individually. | It can improve IP allocation efficiency when correctly supported and configured. |
| **Warm pool** | Extra IPs or network capacity kept ready for upcoming Pods. | It trades faster Pod startup against reserved address capacity. |

**Analogy:** The VPC is a set of numbered seats; IPAM assigns seats, ENIs are access lanes, and prefix delegation reserves a block of seats at once.

**Try / observe:** `kubectl get pods -n kube-system`, `kubectl describe node <node-name>`, and review VPC CNI logs from the `aws-node` Pods.

**Troubleshooting clue:** If Pod creation reports IP allocation errors, check subnet free IPs, ENI/IP limits, CNI health, and node configuration before changing workload resources.

**Explain it back:** Why can a cluster have spare CPU but still fail to create a Pod? What problem does prefix delegation address?

---

## 📖 Core Concepts

### What is VPC CNI?

**Amazon VPC CNI** is the default networking plugin for EKS. It integrates Kubernetes networking with Amazon VPC networking, giving pods IP addresses from the VPC subnet.

```
┌─────────────────────────────────────────────────────────┐
│             VPC CNI PLUGIN ARCHITECTURE                   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              VPC (10.0.0.0/16)                    │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Subnet A (10.0.1.0/24)  Pod IPs        │   │   │
│  │  │  ┌───────────┐ ┌───────────┐           │   │   │
│  │  │  │ ENI 1     │ │ ENI 2     │           │   │   │
│  │  │  │ 10.0.1.10 │ │ 10.0.1.20 │           │   │   │
│  │  │  └───────────┘ └───────────┘           │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │                                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Subnet B (10.0.2.0/24)  Pod IPs        │   │   │
│  │  │  ┌───────────┐ ┌───────────┐           │   │   │
│  │  │  │ ENI 3     │ │ ENI 4     │           │   │   │
│  │  │  │ 10.0.2.10 │ │ 10.0.2.20 │           │   │   │
│  │  │  └───────────┘ └───────────┘           │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │                                                   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Worker Node                         │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Node ENI (primary): 10.0.1.10          │   │   │
│  │  │                                          │   │   │
│  │  │  ┌─────────────────────────────────┐   │   │   │
│  │  │  │  ENI 1 → IP 10.0.1.11 → Pod 1  │   │   │   │
│  │  │  │  ENI 1 → IP 10.0.1.12 → Pod 2  │   │   │   │
│  │  │  │  ENI 2 → IP 10.0.1.13 → Pod 3  │   │   │   │
│  │  │  └─────────────────────────────────┘   │   │   │
│  │  │                                          │   │   │
│  │  │  aws-node (CNI DaemonSet)               │   │   │
│  │  │  - Manages ENIs                         │   │   │
│  │  │  - Assigns IPs to pods                  │   │   │
│  │  │  - Configures routes                    │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Pods have VPC IPs → Direct communication              │
│  with VPC resources (RDS, ELB, etc.)                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### How VPC CNI Assigns IPs

```
┌─────────────────────────────────────────────────────────┐
│           VPC CNI IP ASSIGNMENT                          │
│                                                         │
│  1. Node Starts                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  • EC2 instance launches                        │   │
│  │  • Primary ENI attached                         │   │
│  │  • aws-node DaemonSet starts                    │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  2. CNI Allocates IPs                                  │
│  ┌─────────────────────────────────────────────────┐   │
│  │  • Attaches additional ENIs to node             │   │
│  │  • Allocates secondary IPs                      │   │
│  │  • Same subnet as node                          │   │
│  │  • Tracks in warm pool                          │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  3. Pod Created                                      │
│  ┌─────────────────────────────────────────────────┐   │
│  │  • CNI picks IP from warm pool                  │   │
│  │  • Pod gets VPC IP                              │   │
│  │  • Direct VPC communication                     │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  4. Pod Terminated                                   │
│  ┌─────────────────────────────────────────────────┐   │
│  │  • IP returned to pool                          │   │
│  │  • Available for next pod                       │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### IP Address Limits

| Instance Type | ENIs | IPs per ENI | Max IPs per Node |
|---------------|------|-------------|------------------|
| t3.medium | 3 | 6 | 18 |
| t3.large | 3 | 12 | 36 |
| m5.large | 3 | 10 | 30 |
| m5.xlarge | 4 | 15 | 60 |

---

## 🛠️ Hands-On Practice

### Exercise 1: Check VPC CNI Status

```bash
# Check VPC CNI DaemonSet
kubectl get daemonset -n kube-system aws-node

# Get CNI pods
kubectl get pods -n kube-system -l k8s-app=aws-node

# Check CNI version
kubectl describe daemonset aws-node -n kube-system | grep "Image:"
```

### Exercise 2: Check Pod IPs

```bash
# Deploy test pods
kubectl create deployment nginx --image=nginx --replicas=3

# Check pod IPs (should be VPC subnet IPs)
kubectl get pods -o wide

# Get details
kubectl describe pod <pod-name>

# Check pod networking
kubectl exec -it <pod-name> -- ip addr show eth0
```

### Exercise 3: Update VPC CNI

```bash
# Inspect the installed add-on/version first. If it is already installed, update it:
eksctl get addon --cluster my-cluster
# Choose a version compatible with your EKS cluster, then update it explicitly:
eksctl update addon --name vpc-cni \
  --cluster my-cluster \
  --version <compatible-addon-version>
# Use `eksctl create addon` only when the add-on is not installed.

# Check version
kubectl describe daemonset aws-node -n kube-system | grep "Image:"
```

### Exercise 4: Enable Prefix Delegation

```bash
# Enable prefix delegation only after checking VPC CNI version and subnet prerequisites.
# Prefix delegation needs suitable contiguous subnet IP prefixes and compatible instance types.
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true

# Restart aws-node after planning the change; monitor rollout and Pod IP allocation.
kubectl rollout restart daemonset aws-node -n kube-system
kubectl rollout status daemonset/aws-node -n kube-system
```

---

## 🐛 Troubleshooting

### Pod IP Issues

```bash
# Check CNI logs
kubectl logs -n kube-system <aws-node-pod>

# Check CNI configuration
kubectl exec -it -n kube-system <aws-node-pod> -- cat /etc/cni/net.d/10-aws.conflist

# Check IP pool
kubectl exec -it -n kube-system <aws-node-pod> -- aws-cni-support.sh
```

### No IPs Available

```bash
# Check subnet size
# Each node needs IPs for ENIs + pods
# Ensure subnet has enough IPs

# Check ENI count
aws ec2 describe-network-interfaces \
  --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'NetworkInterfaces[*].[NetworkInterfaceId,PrivateIpAddress]'
```

---

## 📝 Interview Questions

1. **What is VPC CNI?**
   - **Answer:** VPC CNI is the default networking plugin for EKS that assigns VPC IP addresses to pods, enabling direct communication with VPC resources.

2. **How does VPC CNI differ from other CNIs?**
   - **Answer:** VPC CNI uses VPC IPs for pods, so no NAT or overlay network is needed. Other CNIs (Calico, Flannel) use overlay networks.

3. **What are the IP address limits for pods?**
   - **Answer:** Limited by number of ENIs and IPs per ENI per instance type. Prefix delegation can increase the limit.

4. **What is prefix delegation?**
   - **Answer:** Assigns /28 prefix (16 IPs) per ENI instead of /32, increasing the number of IPs available without adding ENIs.

5. **How do you troubleshoot pod networking?**
   - **Answer:** Check CNI logs, pod IP addresses, subnet capacity, and security group rules.

---

## 📚 Resources

- [VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/pod-networking.html)
- [Custom Networking](https://docs.aws.amazon.com/eks/latest/userguide/cni-custom-network.html)
- [Prefix Delegation](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html)

---

## 🎯 Today's Assignment

1. **Check:** Your cluster's CNI configuration
2. **Deploy:** Pods and verify VPC IP assignment
3. **Test:** Pod-to-pod communication
4. **Enable:** Prefix delegation if needed

---

## 📊 Progress Tracker

```
Day 15: EKS Security Best Practices ✅
Day 16: VPC CNI Plugin ⬜
```

---

## 🔗 Next Day Preview

**Day 17:** Network Policies with Calico
- Install Calico
- Implement network policies
- Test traffic restrictions

---

**Remember:** VPC CNI gives pods native VPC networking! 🚀
