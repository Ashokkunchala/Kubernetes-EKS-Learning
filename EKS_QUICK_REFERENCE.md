# 📋 EKS Quick Reference Guide

## 🚀 Essential Commands Cheat Sheet

### Cluster Management
```bash
# eksctl commands
eksctl create cluster --name my-cluster --region us-west-2
eksctl get cluster
eksctl delete cluster --name my-cluster --region us-west-2

# kubectl commands
kubectl cluster-info
kubectl get nodes
kubectl get pods --all-namespaces
kubectl get services
kubectl get deployments
```

### Pod Operations
```bash
# Get pods
kubectl get pods
kubectl get pods -o wide
kubectl get pods -n <namespace>

# Describe and logs
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>
kubectl logs -f <pod-name>  # Follow logs

# Execute commands
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -- ls /app

# Delete pod
kubectl delete pod <pod-name>
```

### Deployments
```bash
# Create deployment
kubectl create deployment nginx --image=nginx

# Scale deployment
kubectl scale deployment nginx --replicas=5

# Update deployment
kubectl set image deployment/nginx nginx=nginx:1.21

# Rollback
kubectl rollout undo deployment/nginx
kubectl rollout history deployment/nginx

# Status
kubectl rollout status deployment/nginx
```

### Services
```bash
# Create service
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Get services
kubectl get svc
kubectl get svc -o wide

# Describe service
kubectl describe svc <service-name>
```

### ConfigMaps and Secrets
```bash
# Create ConfigMap
kubectl create configmap app-config --from-file=config.properties
kubectl create configmap app-config --from-literal=key1=value1

# Create Secret
kubectl create secret generic app-secrets --from-literal=password=abc123
kubectl create secret tls tls-secret --cert=cert.pem --key=key.pem

# Get values
kubectl get configmap app-config -o yaml
kubectl get secret app-secrets -o jsonpath='{.data.password}' | base64 -d
```

### Storage
```bash
# Get PVCs
kubectl get pvc

# Get PVs
kubectl get pv

# Get StorageClasses
kubectl get sc
```

### Namespaces
```bash
# Get namespaces
kubectl get ns

# Create namespace
kubectl create namespace dev

# Set default namespace
kubectl config set-context --current --namespace=dev

# Delete namespace
kubectl delete namespace dev
```

### Debugging
```bash
# Describe resources
kubectl describe pod <pod-name>
kubectl describe node <node-name>
kubectl describe svc <service-name>

# Events
kubectl get events --sort-by='.lastTimestamp'

# Top (metrics)
kubectl top nodes
kubectl top pods

# Proxy to API server
kubectl proxy
```

## 🎯 Common Patterns

### Rolling Update Strategy
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
```

### Resource Limits
```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

### Health Checks
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Affinity
```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - web
      topologyKey: kubernetes.io/hostname

  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - web
        topologyKey: kubernetes.io/hostname
```

### Tolerations
```yaml
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "special-user"
  effect: "NoSchedule"
```

## 🔧 EKS Specific Commands

### Cluster Updates
```bash
# Update cluster version
eksctl upgrade cluster --name my-cluster --version 1.27

# Update node groups
eksctl upgrade nodegroup --name my-nodegroup --cluster my-cluster

# Update addons
eksctl utils update-coredns --cluster my-cluster
eksctl utils update-kube-proxy --cluster my-cluster
eksctl utils update-aws-node --cluster my-cluster
```

### IRSA (IAM Roles for Service Accounts)
```bash
# Associate OIDC provider
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve

# Create service account with IAM role
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --name=my-service-account \
  --namespace=default \
  --attach-policy-arn=arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

### Addons
```bash
# Install EBS CSI Driver
eksctl create addon --name aws-ebs-csi-driver --cluster my-cluster --force

# Install Load Balancer Controller
eksctl create addon --name aws-load-balancer-controller --cluster my-cluster --force

# List addons
eksctl get addon --cluster my-cluster
```

### VPC Configuration
```bash
# Check VPC CNI
kubectl get daemonset aws-node -n kube-system -o yaml

# Update VPC CNI
kubectl set image daemonset/aws-node \
  -n kube-system \
  amazon-k8s-cni=602401143452.dkr.ecr.us-west-2.amazonaws.com/amazon-k8s-cni:v1.12.0
```

## 🐛 Troubleshooting Guide

### Pod Issues
```bash
# Pod stuck in Pending
kubectl describe pod <pod-name>  # Check Events
kubectl get events --field-selector involvedObject.name=<pod-name>

# Pod stuck in CrashLoopBackOff
kubectl logs <pod-name> --previous
kubectl describe pod <pod-name>

# Pod stuck in ImagePullBackOff
kubectl describe pod <pod-name>  # Check image name and registry
```

### Node Issues
```bash
# Check node status
kubectl describe node <node-name>

# Check kubelet logs (on node)
journalctl -u kubelet -f

# Cordon/uncordon node
kubectl cordon <node-name>
kubectl uncordon <node-name>

# Drain node
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

### Network Issues
```bash
# Test DNS resolution
kubectl run dns-test --image=busybox --rm -it -- nslookup <service-name>

# Test connectivity
kubectl exec -it <pod> -- curl <service-ip>

# Check endpoints
kubectl get endpoints <service-name>
```

### Storage Issues
```bash
# Check PVC status
kubectl describe pvc <pvc-name>

# Check PV status
kubectl describe pv <pv-name>

# Check StorageClass
kubectl describe sc <sc-name>
```

## 📊 Monitoring Commands

### Prometheus Queries (PromQL)
```promql
# CPU usage
rate(container_cpu_usage_seconds_total{namespace="default"}[5m])

# Memory usage
container_memory_usage_bytes{namespace="default"}

# Network receive
rate(container_network_receive_bytes_total{namespace="default"}[5m])

# Pod restarts
increase(kube_pod_container_status_restarts_total{namespace="default"}[1h])
```

### Grafana Dashboard Import
```bash
# Kubernetes cluster monitoring
Dashboard ID: 315

# Kubernetes pods monitoring
Dashboard ID: 641

# EKS monitoring
Dashboard ID: 11029
```

## 🔐 Security Checklist

- [ ] Use private subnets for worker nodes
- [ ] Enable encryption at rest for secrets
- [ ] Use IRSA for AWS service access
- [ ] Implement Network Policies
- [ ] Scan container images for vulnerabilities
- [ ] Use Pod Security Standards
- [ ] Enable audit logging
- [ ] Restrict cluster endpoint access
- [ ] Use least privilege IAM roles
- [ ] Implement RBAC properly

## 💰 Cost Optimization

### Instance Types
- **t3.medium:** Development/Testing
- **m5.large:** Production workloads
- **c5.xlarge:** CPU-intensive workloads
- **r5.large:** Memory-intensive workloads

### Spot Instances
```bash
# Create managed node group with spot instances
eksctl create nodegroup \
  --cluster my-cluster \
  --name spot-workers \
  --node-type m5.large \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 5 \
  --spot
```

### Cluster Autoscaler
```bash
# Install Cluster Autoscaler
eksctl create addon \
  --name cluster-autoscaler \
  --cluster my-cluster \
  --force
```

---

**Last Updated:** September 2026
