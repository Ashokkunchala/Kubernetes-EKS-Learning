# 🧩 Module F: Local Development for EKS Practice

> **Prereq:** [Day 08 Introduction to Amazon EKS](../Day-08_Introduction_Amazon_EKS/README.md) · [Day 09 EKS Cluster Setup eksctl](../Day-09_EKS_Cluster_Setup_eksctl/README.md) · [Day 02 Pods, ReplicaSets, Deployments](../Day-02_Pods_ReplicaSets_Deployments/README.md)

You should not burn an EKS cluster to learn Kubernetes. A laptop cluster mirrors 95% of the API surface for free; this module shows you the cheap path AND what stays EKS-only.

## 🎯 Learning Objectives

By the end of this module, you will understand:
- kind vs minikube vs k3d vs k3s — which fits which job
- Installing kind with an Ingress controller on a config-driven cluster
- Deploying the Day-2 demo app (nginx Deployment) plus a Service + Ingress
- Which EKS-specific features you must emulate locally (LB, PVC, IAM)
- `eksctl` local-cluster tips and the EKS parity checklist

---

## 📖 Core Concepts

### The local toolbox

```
┌─────────────────────────────────────────────────────────┐
│              LAPTOP K8s CLUSTER OPTIONS                  │
│                                                         │
│  kind   → multi-node "cluster in Docker", slow but vocal │
│  minikube → single-node, batteries included (drivers)    │
│  k3d    → k3s IN Docker — tiny, fast, SDK-like           │
│  k3s    → real edge-certifiable distro, lightweight      │
│                                                         │
│  All four give you: api server + controllers + kubelet   │
│  None give you:  EKS control plane, VPC CNI, ALB,        │
│                  IRSA / Pod Identity, EBS CSI by default │
└─────────────────────────────────────────────────────────┘
```

### Comparison table

| | **kind** | **minikube** | **k3d** | **k3s** |
|---|---|---|---|---|
| Container runtime | containerd in Docker | Docker/containerd/CRI-O | containerd inside containers | embedded containerd |
| Nodes | 1–N (Docker) | 1 (multi-node addons) | 1–N (Docker) | 1–N (systemd) |
| Multi-node | ✅ | ⚠ via profiles | ✅ | ✅ |
| Ingress out of box | manual (Ingress-NGINX) | addons (`minikube addons enable ingress`) | manual | Traefik bundled |
| EKS "feel" | closest control-plane semantics | easy on Windows/mac drivers | fastest restart | edge/CI, ARM-friendly |
| Best for | **learning EKS APIs**, CI | quick app demos, GPU test | caching/many clusters | prod-ish edge clusters |
| Load balancer | none/Ingress | MetalLB addon | none/Ingress | built-in LB via klipper |

> Recommendation for THIS learning library: **kind** — it runs real control-plane bits (including webhooks & CRDs), is cheap to script, and is the official `kubectl`/CI default. Use k3d when you want speed or throwaway test clusters.

---

## 🚀 Hands-On: kind + Ingress + the Day-2 demo app

### Step 1: Install kind + check Docker

```bash
# Windows (PowerShell, winget) or manual binary — pick one
winget install kind                              # or:
# curl -Lo kind.exe https://kind.sigs.k8s.io/dl/v0.30.0/kind-windows-amd64
# then move kind.exe into a PATH directory

docker version                                    # must be running!
kind version
```

### Step 2: Cluster config (with the ports your Ingress will need)

```yaml
# kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: eks-lab
nodes:
- role: control-plane
  extraPortMappings:          # map host ports → nodePorts for the ingress LB
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
networking:
  apiServerAddress: 127.0.0.1
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/16
```

```bash
kind create cluster --config kind-cluster.yaml
kubectl cluster-info --context kind-eks-lab
kubectl get nodes                             # 1 node, Ready
```

### Step 3: Ingress controller (mirrors ALB/Ingress-nginx on EKS)

```bash
# The same Ingress-nginx you'd use on EKS (or ALB add-on there)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller --timeout=90s
```

> On EKS, this controller would be the AWS Load Balancer Controller + `ingressClassName: alb`. Locally, `ingressClassName: nginx` + `hostPort:80` plays the same role.

### Step 4: The Day-2 demo app (nginx)

```yaml
# demo-app.yaml — same shape as Day 02 `my-deployment`
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  labels:
    app: nginx-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.21          # from Day 02
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-deployment
spec:
  selector:
    app: nginx-demo
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: demo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-deployment
            port:
              number: 80
```

```bash
kubectl apply -f demo-app.yaml
kubectl get deploy,svc,ingress

# Local /etc/hosts add:  127.0.0.1 demo.local
curl http://demo.local/ -H "Host: demo.local"
# welcome to nginx ...
```

---

## 🔁 Mirroring EKS-Specific Features Locally

```
EKS feature                 LOCAL REPLACEMENT              WHY
────────────────────────   ────────────────────────   ─────────────────
AWS ALB / NLB (Day 13)        Ingress-NGINX (kind)     Traefik/nginx LB
EBS CSI Driver (Day 12)       local-path / hostPath    CSI needs real EBS
IRSA / Pod Identity (Day27)   kube2iam + IAM mock      needs no AWS metadata
VPC CNI (Day 16)              CNI default (flannel)    IPAMD requires VPC
Secrets Manager (Day 29)      plain Secrets / AWS mock  LocalStack
Karpenter (Day 53)            ClusterAutoscaler?       needs real EC2 API
Monitoring (Day 41)           kube-prometheus-stack     works identically!
```

```yaml
# local PVC (no EBS!) — a hostPath/local PersistentVolume for Day-21 practice
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-path
  hostPath:
    path: /var/lib/kind-data    # writable mount from the node container
```

---

## 🧪 EKS local-dev extras

```bash
# 1) eksctl  "local" flavor — a single-command Dockerized fake cluster
#   (experimental, mirrors eksctl CLI ergonomics with local infra)
eksctl local create cluster --name dev --version 1.31

# 2) LocalStack for AWS mock (S3/SQS/Secrets Manager) shared with kind:
docker run --rm -p 4566:4566 localstack/localstack
# then point AWS SDKs / IRSA pods at http://host.docker.internal:4566

# 3) For CI: create kind clusters per PR branch and destroy after
kind delete cluster --name eks-lab
```

> Tip: The EKS **add-ons** (CNI, kube-proxy, CoreDNS) also run on kind if you `kubectl apply` their manifests — most behave identically except anything requiring EC2/VPC metadata.

---

## 🛠️ Troubleshooting

```bash
# 1. "Unable to connect to the server" → Docker not running / context stale
kind get clusters
kubectl config current-context

# 2. Ingress creates but won't answer on :80
#    → control-plane hostPort ran BEFORE the ingress install; recreate:
kind delete cluster --name eks-lab && kind create cluster --config kind-cluster.yaml

# 3. Node shows NotReady after reboot (Windows): restart Docker then
kind delete cluster --name eks-lab && kind create cluster --config kind-cluster.yaml

# 4. MetalLB (load-balancer Service type): install MetalLB not Ingress,
#    then use `type: LoadBalancer` services to get EXTERNAL-IP 172.18.0.x
kubectl get svc my-deployment      # EXTERNAL-IP appears once MetalLB runs

# 5. ImagePullBackOff on ARM/macOS for platform-specific images:
#    build multi-arch or use images that publish linux/arm64
```

---

## ❓ Interview Questions

1. **What does kind give you that a real EKS cluster does not?**
   - A full local API server + controllers in Docker: same RBAC, webhooks, CRDs, and kubectl UX, so you can practice the control-plane surface for free; the trade-offs are production semantics (load balancing, IAM, EBS, networking).

2. **Why is kind preferred over minikube for CI workflows?**
   - kind is Docker-native, scriptable (`kind create cluster`), multi-node with a YAML config, and destroys instantly — matching ephemeral CI per-PR isolation.

3. **How do you locally emulate an AWS Load Balancer?**
   - Install Ingress-NGINX (or MetalLB for `type: LoadBalancer`); the EKS equivalent is the AWS Load Balancer Controller creating ALB/NLB, which you can't run locally without AWS.

4. **When would you pick k3d over kind?**
   - When you want speed and tiny footprint to spin many throwaway clusters (microservice integration tests); kind is the better control-plane fidelity pick.

5. **How do you test IRSA/Pod Identity locally?**
   - Mock the credentials chain: use kube2iam style sidecars pointing at LocalStack, or switch to a local env-var-based AWS SDK config; there is no EC2 instance metadata on kind.

---

## 📚 Resources

- [kind docs](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [kind + ingress guide](https://kind.sigs.k8s.io/docs/user/ingress/)
- [minikube docs](https://minikube.sigs.k8s.io/docs/)
- [k3d docs](https://k3d.io/)
- [eksctl local cluster (experimental)](https://eksctl.io/usage/local-cluster/)

---

## ✅ Progress Tracker

```
Module F: Local Development
  [x] Tool comparison (kind/minikube/k3d/k3s)
  [x] kind cluster from a config YAML + hostPorts
  [x] Ingress-NGINX installed (mirrors ALB)
  [ ] Day-02 my-deployment demo wired through Ingress
  [ ] Local PersistentVolume (no EBS!) exercised
  [ ] LocalStack / eksctl local tips attempted
  [ ] Documented the EKS-parity checklist for the team
```

---

## 👀 Next

- **Take your local practice to a real cluster:** [Day 09 EKS Cluster Setup eksctl](../Day-09_EKS_Cluster_Setup_eksctl/README.md)
- **What the local Ingress is replicating (ALB):** [Day 13 EKS Load Balancing ALB](../Day-13_EKS_Load_Balancing_ALB/README.md)
- **Re-deploy Day 2 objects with the real knobs:** [Day 02 Pods, ReplicaSets, Deployments](../Day-02_Pods_ReplicaSets_Deployments/README.md)

---

**Remember:** A laptop cluster is for learning the *API*; EKS is for learning the *platform*. Do both — cheap first. 💻