# 📅 Day 47: Multi-Cluster Networking

## 🎯 Learning Objectives
By the end of today, you will understand:
- Why you need multi-cluster networking
- Connecting clusters (peering, mesh, transit)
- Service discovery across clusters
- AWS Gateway Load Balancer & transit gateway patterns
- Service Mesh across clusters (Istio, Linkerd)

---

## 📖 Core Concepts

### Multi-Cluster Topologies

```
┌───────────────────────────────────────────────────────────────┐
│                    MULTI-CLUSTER NETWORKING                    │
│                                                               │
│   ┌──────────────┐        ┌──────────────┐                    │
│   │   Cluster A  │        │   Cluster B  │                    │
│   │  (us-east-1) │        │  (us-west-2) │                    │
│   │  ┌─────────┐ │  Mesh  │  ┌─────────┐ │                    │
│   │  │  Svc X  │ │◀──────▶│  │  Svc Y  │ │                    │
│   │  └─────────┘ │        │  └─────────┘ │                    │
│   └──────┬───────┘        └──────┬───────┘                    │
│          │ AWS PrivateLink / VPC Peering                      │
│          └────────────┬──────────────────┘                    │
│                       ▼                                       │
│        ┌──────────────────────────────┐                       │
│        │   Global Service Discovery   │                       │
│        │   (mesh DNS / federation)    │                       │
│        └──────────────────────────────┘                       │
└───────────────────────────────────────────────────────────────┘
```

### Why Multi-Cluster?

- **HA & DR** — survive region/cluster failure.
- **Data locality** — serve users from nearest region.
- **Scale limits** — split large fleets; avoid noisy-neighbor.
- **Isolation** — compliance/tenant separation.

### Connection Options

| Option | Scope | Use case |
|--------|-------|----------|
| VPC Peering | Two VPCs | Simple, low cost, no transit |
| Transit Gateway | Many VPCs | Hub-and-spoke, many clusters |
| AWS PrivateLink | Service endpoints | Secure service-to-service |
| Service Mesh (Istio) | L7 across clusters | Advanced discovery, traffic, security |
| ALB/GWLB across | External exposure | Global ingress |

---

## 🛠️ Step 1: Enable Pod-to-Pod Across VPCs

VPC CNI assigns nodes ENI IPs in the **VPC CIDR**, so pods are routable within the VPC. To route pods across VPCs, use a **Transit Gateway + routing** (or peering with overlapping-free CIDRs):

```bash
# Create a Transit Gateway
aws ec2 create-transit-gateway --options AutoAcceptSharedAttachments=enable

# Attach each EKS VPC
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-xxx \
  --vpc-id vpc-cluster-a --subnet-ids subnet-a1 subnet-a2

# Add routes so each VPC knows the other's CIDRs via TGW
aws ec2 create-route --route-table-id rtb-cluster-a \
  --destination-cidr-block 10.1.0.0/16 --transit-gateway-id tgw-xxx
```

> Ensure the **pod CIDRs don't overlap** across clusters, or routing breaks.

---

## 🛠️ Step 2: Service Discovery Across Clusters

### Option A: CoreDNS federation / external DNS

```bash
# In-cluster Service → external DNS name per region
kubectl -n production annotate svc orders \
  "external-dns.alpha.kubernetes.io/hostname=orders.us-west-2.example.com"
```

### Option B: Service Mesh (Istio) — recommended for advanced workloads

Mesh spreads across clusters; services resolve via the mesh registry.

```bash
# Install Istio on Cluster A and B with meshConfig for multi-network
istioctl install -f profile=multiCluster
```

Register a remote cluster:

```bash
# On the primary cluster
istioctl x create-remote-secret \
  --context kubeconfig-cluster-b | kubectl apply -f - -n istio-system

kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: orders-remote
spec:
  hosts: ["orders.orders-ns.svc.cluster.local"]
  location: MESH_EXTERNAL
  resolution: DNS
  addresses: ["10.1.0.0/16"]
  endpoints:
  - address: orders.orders-ns.global
    ports: { http: 80 }
EOF
```

Now `orders` is discoverable across both clusters with mTLS between meshes.

---

## 🛠️ Step 3: Cross-Cluster Traffic with Istio (Gateway per network)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: cross-network-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port: { number: 15443, name: tls, protocol: TLS }
    hosts: ["*.local"]
    tls:
      mode: AUTO_PASSTHROUGH
```

Services in cluster A can reach services in cluster B through the mesh using the `*.global` suffix automatically handled by Istio.

---

## 🛠️ Step 4: External Global Ingress

Use a **Global Accelerator + EKS ALB/GWLB** for a single global entry point across region clusters:

```bash
# Global Accelerator listener → endpoints (regional EKS ALBs/NLBs)
aws globalaccelerator create-accelerator --name eks-global --enabled
# create listener 443 → endpoint group per region → endpoint (e.g., ALB ARN)
```

---

## 🔧 Troubleshooting

```bash
# Verify cross-cluster DNS / mesh
kubectl -n istio-system get svc istio-ingressgateway
istioctl proxy-status

# Check endpoints resolves across clusters
kubectl run probe --rm -it --image=busybox -- nslookup orders.orders-ns.global

# Ping/curl across
kubectl -n production exec deploy/orders -- curl http://orders.orders-ns.global:80/health
```

---

## 📝 Interview Questions

1. **How do you connect two EKS clusters in different VPCs?**
   - **Answer:** VPC Peering for simple cases or a Transit Gateway for many VPCs; ensure non-overlapping pod CIDRs and add the needed routes.

2. **How do you enable cross-cluster service discovery?**
   - **Answer:** Use a service mesh (Istio) with a global namespace, or external DNS/CoreDNS federation, or a central API gateway that routes to regional endpoints.

3. **What are the security considerations for multi-cluster networking?**
   - **Answer:** mTLS between meshes, non-overlapping CIDRs, network policies, least-privilege IAM per cluster, and PrivateLink/TGW for controlled connectivity.

4. **When would you choose a mesh over direct VPC connectivity?**
   - **Answer:** When you need L7 traffic management, mTLS, observability, and service discovery across clusters — mesh; for simple low-latency L3/L4, direct peering/TGW.

5. **How do you avoid overlapping pod/service CIDRs?**
   - **Answer:** Plan IPAM centrally; assign distinct per-cluster CIDRs, and use a central controller or documented registry to prevent collisions.

---

## 📚 Resources for Review

- [Day 46: EKS Anywhere](Day-46_EKS_Anywhere/README.md)
- [Day 18: Service Mesh with Istio](Day-18_Service_Mesh_Istio/README.md)

### External
- [AWS Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/)
- [Istio multi-cluster](https://istio.io/latest/docs/setup/install/multicluster/)

---

## 🎯 Today's Assignment

1. **Peer** two VPCs (or set up a Transit Gateway) and route pod CIDRs
2. **Enable** cross-cluster DNS with external-dns or CoreDNS
3. **Install** Istio multi-cluster and register a remote cluster
4. **Resolve** a service from the other cluster
5. **Harden** with mTLS and network policies

---

## 📊 Progress Tracker

```
Week 15-16 Multi-Cluster:
Day 46: EKS Anywhere ⬜
Day 47: Multi-Cluster Networking ⬜
Day 48: Cluster Federation ⬜
Day 49: Multi-Cluster Practice ⬜
Day 50: Multi-Cluster Interview Prep ⬜
```

---

## 🔗 Next Week Preview

**Day 48:** Cluster Federation

---

**Remember:** Start with non-overlapping CIDRs — multi-cluster networking fails on the details! 🚀
