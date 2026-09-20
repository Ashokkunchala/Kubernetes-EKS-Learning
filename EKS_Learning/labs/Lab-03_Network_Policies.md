# 🧪 Lab 03: Network Policies — Microsegmentation in One Namespace

Kubernetes allows all pod-to-pod traffic by default. In a real cluster that's unacceptable. This lab builds **microsegmentation** from zero using three labeled apps and a client — and you *prove* each rule with real `wget` results.

## 🎯 Lab Goal

1. Deploy three labeled apps (`app=a`, `app=b`, `app=c`) plus a client pod
2. Apply a **default-deny** policy and watch traffic break
3. Apply an **allow-specific** rule (`a → b` on port 80) and verify +1 allowed, −1 denied with `wget`

## 🧪 Scenario

```
   namespace: netpol-lab
   ┌────────────────────────────────────────────────────────────┐
   │                                                            │
   │   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   │
   │   │ app=a   │   │ app=b   │   │ app=c   │   │ client  │   │
   │   │ nginx   │   │ nginx   │   │ nginx   │   │ busybox │   │
   │   │ :80 wget│   │ :80 wget│   │ :80 wget│   │   wget  │   │
   │   └────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘   │
   │        │             │             │             │        │
   │   DEFAULT:  ALL ⇄ ALL ██████████████ ALLOWED            │
   │   STEP 2 :  ALL BLOCKED (default-deny)                  │
   │   STEP 3 :  a ──► b :80 ONLY  (+1)  everything else(−1) │
   └────────────────────────────────────────────────────────────┘
```

## ✅ Prerequisites

- A CNI that **enforces** NetworkPolicies:

| Runtime | Requirement |
|---------|-------------|
| **EKS** | Install the **Calico** add-on — default Amazon VPC CNI does *not* enforce NetworkPolicies ([Day-17](../Day-17_Network_Policies_Calico/README.md)) |
| **kind** | Install **Cilium** (set `kind` config `networking: {disableDefaultCNI: true}`) or Calico |
| **Minikube** | `minikube start --network-plugin=cni --cni=calico` (or Cilium) |

Confirm it's working before continuing:

```bash
kubectl get pods -n kube-system | grep -iE 'cilium|calico'
```

> ⚠️ If this returns nothing, the rest of the lab "succeeds" silently — pods will still talk to each other. The policy will be *stored* but **not enforced**.

---

## ▶️ Steps

### Step 1 — Deploy three apps + a client pod

Each app is BusyBox serving an HTTP page on port 80 (`httpd`) **and** it bundles `wget`, so we can exec into one app and reach another. The client is plain BusyBox.

`apps.yaml`:

```yaml
# Three micro-segmented apps (a, b, c) + a client pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a
  namespace: netpol-lab
  labels: {app: a}
spec:
  replicas: 1
  selector: {matchLabels: {app: a}}
  template:
    metadata:
      labels: {app: a}
    spec:
      containers:
      - name: httpd
        image: busybox:1.36
        command: ["sh", "-c", "mkdir -p /www && echo 'hello from a' > /www/index.html && httpd -p 80 -h /www -f"]
        ports: [{containerPort: 80}]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-b
  namespace: netpol-lab
  labels: {app: b}
spec:
  replicas: 1
  selector: {matchLabels: {app: b}}
  template:
    metadata:
      labels: {app: b}
    spec:
      containers:
      - name: httpd
        image: busybox:1.36
        command: ["sh", "-c", "mkdir -p /www && echo 'hello from b' > /www/index.html && httpd -p 80 -h /www -f"]
        ports: [{containerPort: 80}]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-c
  namespace: netpol-lab
  labels: {app: c}
spec:
  replicas: 1
  selector: {matchLabels: {app: c}}
  template:
    metadata:
      labels: {app: c}
    spec:
      containers:
      - name: httpd
        image: busybox:1.36
        command: ["sh", "-c", "mkdir -p /www && echo 'hello from c' > /www/index.html && httpd -p 80 -h /www -f"]
        ports: [{containerPort: 80}]
---
apiVersion: v1
kind: Pod
metadata:
  name: client
  namespace: netpol-lab
  labels: {tier: client}
spec:
  containers:
  - name: client
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
---
# One stable DNS name per app so tests don't chase pod IPs
apiVersion: v1
kind: Service
metadata:
  name: svc-a
  namespace: netpol-lab
spec: {selector: {app: a}, ports: [{port: 80, targetPort: 80}]}
---
apiVersion: v1
kind: Service
metadata:
  name: svc-b
  namespace: netpol-lab
spec: {selector: {app: b}, ports: [{port: 80, targetPort: 80}]}
---
apiVersion: v1
kind: Service
metadata:
  name: svc-c
  namespace: netpol-lab
spec: {selector: {app: c}, ports: [{port: 80, targetPort: 80}]}
```

```bash
kubectl create ns netpol-lab
kubectl apply -f apps.yaml
kubectl get pods,svc,endpoints -n netpol-lab -o wide
```

**Expected output:**

```
NAME                        READY   STATUS    RESTARTS   AGE
pod/app-a-6d7c9d5f5b-x1     1/1     Running   0          15s
pod/app-b-6d9f8d7f6b-y2     1/1     Running   0          15s
pod/app-c-7c9e7d6f5c-z3     1/1     Running   0          15s
pod/client                 1/1     Running   0          15s

NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/svc-a ClusterIP   10.109.1.11    <none>        80/TCP    15s
service/svc-b ClusterIP   10.109.1.12    <none>        80/TCP    15s
service/svc-c ClusterIP   10.109.1.13    <none>        80/TCP    15s

NAME              ENDPOINTS        AGE
endpoints/svc-a   10.244.1.20:80   15s
endpoints/svc-b   10.244.1.21:80   15s
endpoints/svc-c   10.244.1.22:80   15s
```

**Baseline** — no policies yet, so everyone can reach everyone:

```bash
kubectl exec -n netpol-lab client -- wget -qO- -T 3 http://svc-a && echo
kubectl exec -n netpol-lab client -- wget -qO- -T 3 http://svc-b && echo
kubectl get networkpolicy -n netpol-lab
```

**Expected output:**

```
hello from a
hello from b
No resources found in netpol-lab namespace.
```

---

### Step 2 — Default-deny: shut everything in the namespace down

`default-deny.yaml`:

```yaml
# Deny ALL ingress to every pod in this namespace (allow-nothing baseline)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: netpol-lab
spec:
  podSelector: {}              # matches every pod in the namespace
  policyTypes:
  - Ingress
```

```bash
kubectl apply -f default-deny.yaml
kubectl exec -n netpol-lab client -- wget -t 2 -T 3 -qO- http://svc-b && echo  "  ← SHOULD NOT PRINT"
```

**Expected output** (exit code non-zero; the `&& echo` should **not** fire):

```
wget: download timed out
```

```bash
echo $?
```

**Expected output:**

```
1
```

> ✅ **Proved:** with zero allow-rules, traffic between pods in the namespace is dead. This is the default posture production teams want: **deny, then explicitly allow.**

---

### Step 3 — Allow only `app=a → app=b` on port 80

`allow-a-to-b.yaml`:

```yaml
# Allow ingress ONLY from app=a to app=b on TCP:80 — single network path open
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-a-to-b
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      app: b               # attached to pods labeled app=b (the victims)
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: a           # the only allowed source
    ports:
    - protocol: TCP
      port: 80
```

> Because Kubernetes NetworkPolicies are **additive**, this rule opens exactly one path through the namespace-wide default-deny.

```bash
kubectl apply -f allow-a-to-b.yaml
kubectl get networkpolicy -n netpol-lab
```

**Expected output:**

```
networkpolicy.networking.k8s.io/default-deny-all created
networkpolicy.networking.k8s.io/allow-a-to-b created

NAME              POD-SELECTOR   AGE
allow-a-to-b      app=b          8s
default-deny-all  <none>         2m
```

Now the full verdict — four `wget` probes from three distinct sources:

```bash
A=$(kubectl get pod -n netpol-lab -l app=a -o jsonpath='{.items[0].metadata.name}')
B=$(kubectl get pod -n netpol-lab -l app=b -o jsonpath='{.items[0].metadata.name}')
C=$(kubectl get pod -n netpol-lab -l app=c -o jsonpath='{.items[0].metadata.name}')

echo "→ a → b (ALLOWED):"  ; kubectl exec -n netpol-lab "$A" -- wget -qO- -T 3 http://svc-b
echo "→ b → a (denied):  " ; kubectl exec -n netpol-lab "$B" -- wget -t 1 -T 3 -qO- http://svc-a
echo "→ c → b (denied):  " ; kubectl exec -n netpol-lab "$C" -- wget -t 1 -T 3 -qO- http://svc-b
echo "→ a → c (denied):  " ; kubectl exec -n netpol-lab "$A" -- wget -t 1 -T 3 -qO- http://svc-c
echo "→ client → b (denied): " ; kubectl exec -n netpol-lab client -- wget -t 1 -T 3 -qO- http://svc-b
```

**Expected output:**

```
→ a → b (ALLOWED): hello from b
→ b → a (denied):  wget: download timed out
→ c → b (denied):  wget: download timed out
→ a → c (denied):  wget: download timed out
→ client → b (denied): wget: download timed out
```

Scoreboard:

| Path | Result | Why |
|------|--------|-----|
| `a → b:80` | ✅ allowed | only matching allow rule |
| `b → a:80` | ❌ denied | no rule selects `app=a` as victim |
| `c → b:80` | ❌ denied | source label is `a`, not `c` |
| `a → c:80` | ❌ denied | no rule selects `app=c` as victim |
| `client → b:80` | ❌ denied | source label is `a`, not `client` |

> ✅ **Proved:** exactly **+1** path opened, **−1** (everything else) stays closed. That is microsegmentation.

---

### ⚠️ Egress-to-DNS caveat (read this before extending)

All the policies above only restrict **Ingress**. DNS resolution (`svc-a`, `svc-b`, …) still works because **Egress** is not denied.

The moment you add an egress `default-deny` policy, every pod loses DNS lookups to CoreDNS — **including the kube-dns pods' own lookups**. If you do that, you must explicitly allow DNS egress, like `dns-allow.yaml`:

```yaml
# REQUIRED companion to any egress default-deny: keep CoreDNS reachable
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-dns
  namespace: netpol-lab
spec:
  podSelector: {}                  # every pod
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

(Optional — verify, then remove to keep the namespace default-deny clean):

```bash
kubectl delete -f allow-egress-dns.yaml   # we don't need it for this lab
```

---

## ✅ Solution Recap

**Checkpoint commands:**

```bash
# Baseline: all traffic allowed
kubectl exec -n netpol-lab client -- wget -qO- -T 3 http://svc-b

# Lockdown: apply default-deny, all traffic dies
kubectl apply -f default-deny.yaml

# Open exactly one path
kubectl apply -f allow-a-to-b.yaml

# Verdict (5 probes, expect 1 success, 4 timeouts)
kubectl get networkpolicy -n netpol-lab
```

**What you proved:**

- [ ] Default cluster behavior: **allow all** pod-to-pod
- [ ] `podSelector: {}` selects every pod in a namespace
- [ ] Default-deny kills traffic instantly, with no downtime or restarts
- [ ] Policies are additive — one allow rule opens one path
- [ ] Source matching uses **pod labels**, not IPs or services
- [ ] Ingress-only policies don't break DNS; egress default-deny does

**Bonus thought (interview favourite):** NetworkPolicies match *pods*, so `kube-proxy` DNATs service traffic to real pod IPs and the policy still applies. That's why the client could not reach `svc-b` even though a Service exists.

## 🧹 Cleanup

```bash
kubectl delete ns netpol-lab
kubectl get networkpolicy -A   # verify nothing is left
```

## 🛠️ Troubleshooting

| Symptom | Likely cause | Check |
|---------|--------------|------|
| Policies applied but **traffic still flows** | CNI doesn't enforce policies (kind default / VPC CNI on EKS) | `kubectl get pods -n kube-system \| grep -iE 'cilium\|calico'` |
| `wget` resolves but connection refused | Different port than 80 | `kubectl exec <pod> -- netstat -lnt` |
| All paths denied even `a→b` | Selector wrong (label on deployment ≠ label on pod) | `kubectl get pods --show-labels -n netpol-lab` |
| DNS breaks after lab | You applied an egress default-deny without the DNS rule | Re-apply `allow-egress-dns.yaml` |

## 📚 Resources

- [Kubernetes docs — Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes docs — Network Policy recipes/use cases](https://kubernetes.io/docs/concepts/services-networking/network-policies/#networkpolicy-resource)
- [EKS — Calico network policies](https://docs.aws.amazon.com/eks/latest/userguide/calico.html)
- [Cilium — network policy docs](https://docs.cilium.io/en/stable/security/policy/)
- Related: [Day-03 Services/Networking](../Day-03_Services_Networking/README.md), [Day-17 Network Policies with Calico](../Day-17_Network_Policies_Calico/README.md)

**You now have a mental model of CIDR-free, label-based firewalling. 🚀**