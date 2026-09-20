# 📅 Day 10: EKS Networking and VPC

## 🎯 Learning Objectives
By the end of today, you will understand:
- VPC configuration for EKS
- Subnet types (Public/Private)
- NAT Gateway vs Internet Gateway
- Security Groups for EKS
- Network architecture best practices

---

## 📖 Core Concepts

### EKS VPC Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    VPC (10.0.0.0/16)                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │           Public Subnets (Auto-assign Public IP) │   │
│  │                                                   │   │
│  │  ┌─────────────────────┐  ┌─────────────────────┐│   │
│  │  │  AZ-1a              │  │  AZ-1b              ││   │
│  │  │  10.0.1.0/24        │  │  10.0.2.0/24        ││   │
│  │  │                     │  │                     ││   │
│  │  │  ┌───────────────┐  │  │  ┌───────────────┐  ││   │
│  │  │  │  Internet     │  │  │  │  Internet     │  ││   │
│  │  │  │  Gateway      │  │  │  │  Gateway      │  ││   │
│  │  │  └───────────────┘  │  │  └───────────────┘  ││   │
│  │  └─────────────────────┘  └─────────────────────┘│   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │           Private Subnets (No Public IP)         │   │
│  │                                                   │   │
│  │  ┌─────────────────────┐  ┌─────────────────────┐│   │
│  │  │  AZ-1a              │  │  AZ-1b              ││   │
│  │  │  10.0.3.0/24        │  │  10.0.4.0/24        ││   │
│  │  │                     │  │                     ││   │
│  │  │  ┌───────────────┐  │  │  ┌───────────────┐  ││   │
│  │  │  │  EKS Worker   │  │  │  │  EKS Worker   │  ││   │
│  │  │  │  Nodes        │  │  │  │  Nodes        │  ││   │
│  │  │  └───────────────┘  │  │  └───────────────┘  ││   │
│  │  │                     │  │                     ││   │
│  │  │  ┌───────────────┐  │  │  ┌───────────────┐  ││   │
│  │  │  │  NAT Gateway  │  │  │  │  NAT Gateway  │  ││   │
│  │  │  └───────────────┘  │  │  └───────────────┘  ││   │
│  │  └─────────────────────┘  └─────────────────────┘│   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  EKS Control Plane (AWS Managed)                 │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • API Server ENI (in your VPC)         │   │   │
│  │  │  • Multi-AZ deployment                  │   │   │
│  │  │  • Encrypted communication              │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Subnet Types

```
┌─────────────────────────────────────────────────────────┐
│                   SUBNET TYPES                           │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Public Subnets                                  │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Auto-assign public IP                 │   │   │
│  │  │  • Internet Gateway attached            │   │   │
│  │  │  • Direct internet access               │   │   │
│  │  │                                          │   │   │
│  │  │  Use Cases:                              │   │   │
│  │  │  • NAT Gateways                         │   │   │
│  │  │  • Load Balancers                       │   │   │
│  │  │  • Bastion hosts                        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Private Subnets                                 │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • No public IP                         │   │   │
│  │  │  • Route to NAT Gateway                 │   │   │
│  │  │  • No direct internet access            │   │   │
│  │  │                                          │   │   │
│  │  │  Use Cases:                              │   │   │
│  │  │  • EKS Worker Nodes                     │   │   │
│  │  │  • Databases                            │   │   │
│  │  │  • Application workloads                │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Internet Gateway vs NAT Gateway

```
┌─────────────────────────────────────────────────────────┐
│              INTERNET GATEWAY vs NAT GATEWAY             │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Internet Gateway                                │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Enables internet access              │   │   │
│  │  │  • Bidirectional traffic                 │   │   │
│  │  │  • Attached to VPC                      │   │   │
│  │  │  • Free to use                          │   │   │
│  │  │                                          │   │   │
│  │  │  Traffic Flow:                           │   │   │
│  │  │  Internet ◄──► VPC                       │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  NAT Gateway                                      │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Enables outbound internet            │   │   │
│  │  │  • One-way traffic (outbound only)      │   │   │
│  │  │  • Placed in public subnet              │   │   │
│  │  │  • Charges apply ($0.045/hour)          │   │   │
│  │  │                                          │   │   │
│  │  │  Traffic Flow:                           │   │   │
│  │  │  Private Subnet ──► NAT ──► Internet    │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🔒 Security Groups

### EKS Security Groups

```
┌─────────────────────────────────────────────────────────┐
│              EKS SECURITY GROUPS                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Control Plane Security Group                    │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Allows traffic to API Server         │   │   │
│  │  │  • Port 443 (HTTPS)                     │   │   │
│  │  │  • Inbound from worker nodes            │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Node Security Group                             │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Worker node to worker node           │   │   │
│  │  │  • Worker node to control plane         │   │   │
│  │  │  • Pod to pod communication             │   │   │
│  │  │  • Ports: 10250, 443, 53, 80, 443      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Pod Security Group (Optional)                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • Pod-level network isolation          │   │   │
│  │  │  • Additional security layer            │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Security Group Rules

```
┌─────────────────────────────────────────────────────────┐
│              SECURITY GROUP RULES                        │
│                                                         │
│  Control Plane SG:                                      │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Inbound:                                        │   │
│  │  • Port 443 from Node SG                        │   │
│  │  • Port 443 from admin IP                       │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  Node SG:                                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Inbound:                                        │   │
│  │  • Port 10250 from Control Plane SG             │   │
│  │  • Port 443 from Control Plane SG              │   │
│  │  • Port 443 from Node SG (self)               │   │
│  │  • Port 53 from Node SG (self)                │   │
│  │                                                  │   │
│  │  Outbound:                                       │   │
│  │  • Port 443 to Control Plane SG                │   │
│  │  • Port 443 to internet (for images)           │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 📝 Practice Exercises

### Exercise 1: Review VPC Configuration

```bash
# Get VPC ID from cluster
eksctl get cluster --name my-cluster --region us-west-2

# Get VPC details
aws ec2 describe-vpcs --filters "Name=vpc-id,Values=<vpc-id>"

# Get subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'Subnets[*].[SubnetId,AvailabilityZone,CidrBlock,MapPublicIpOnLaunch]' \
  --output table
```

### Exercise 2: Check Security Groups

```bash
# Get security groups
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'SecurityGroups[*].[GroupId,GroupName]' \
  --output table

# Check rules for node security group
aws ec2 describe-security-group-rules --filters "Name=group-id,Values=<node-sg-id>" \
  --query 'SecurityGroupRules[*].[SecurityGroupRuleId,IpProtocol,FromPort,ToPort,CidrIpv4]' \
  --output table
```

---

## 📝 Interview Questions

1. **Why use private subnets for EKS worker nodes?**
   - **Answer:** Private subnets provide better security by preventing direct internet access. Worker nodes can still access the internet through NAT Gateways for pulling images and updates.

2. **What is the difference between NAT Gateway and Internet Gateway?**
   - **Answer:** Internet Gateway enables bidirectional internet traffic. NAT Gateway enables only outbound internet traffic for private subnets.

3. **How to configure VPC for EKS?**
   - **Answer:** Create a VPC with at least 2 public and 2 private subnets across different AZs. Attach Internet Gateway to public subnets and NAT Gateway to private subnets.

4. **What are the networking requirements for EKS?**
   - **Answer:** Minimum 2 subnets in different AZs, proper CIDR ranges, Internet Gateway (for public), NAT Gateway (for private), and appropriate Security Groups.

5. **How does EKS handle networking?**
   - **Answer:** EKS uses VPC CNI plugin to assign VPC IP addresses to pods. Pods get IPs from the VPC subnet, enabling direct communication with other VPC resources.

---

## 📚 Resources

- [EKS Networking](https://docs.aws.amazon.com/eks/latest/userguide/networking.html)
- [VPC for EKS](https://docs.aws.amazon.com/eks/latest/userguide/setting-up-vpc.html)
- [Security Groups](https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html)

---

## 🎯 Today's Assignment

1. **Review:** Your EKS cluster VPC configuration
2. **Document:** Subnet layout and CIDR ranges
3. **Analyze:** Security Group rules
4. **Plan:** Network architecture for a production cluster

---

## 📊 Progress Tracker

```
Day 9: EKS Cluster Setup with eksctl ✅
Day 10: EKS Networking and VPC ⬜
```

---

## 🔗 Next Day Preview

**Day 11:** EKS IAM Roles and Policies
- IAM Roles for Service Accounts (IRSA)
- Node IAM Role
- EKS Cluster IAM Role
- OIDC Provider for IRSA

---

**Remember:** Proper network design is critical for EKS security and performance! 🚀
