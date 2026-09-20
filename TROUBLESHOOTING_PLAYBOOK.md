# 🚑 EKS Troubleshooting Playbook

> Real incident runbooks, in order of likelihood. Each runbook: **symptoms →
> quick triage → root causes → fix → verify**. Complements the command cheat
> sheet in [`EKS_QUICK_REFERENCE.md`](EKS_QUICK_REFERENCE.md).

---

## ⚡ Runbook 1 — Pod stuck in `Pending`

**Symptoms:** `kubectl get pods` shows `Pending` forever; `kubectl describe pod`
shows `FailedScheduling`.

**Triage:**
```bash
kubectl describe pod <pod> -n <ns>          # look for FailedScheduling event
```

**Root causes & fixes:**
| Cause | Detection | Fix |
|---|---|---|
| No node has enough CPU/mem | "insufficient cpu/memory" in events | reduce requests, scale nodegroup, or enable autoscaler/Karpenter |
| Taints don't match tolerations | `taint added` events | add toleration or remove taint |
| Node selector/affinity unsatisfiable | unschedulable events show labels | fix nodeSelector labels or add nodegroup labels |
| PVC not bound (EBS pending) | events show `waiting for first consumer` | check StorageClass → `kubectl get pvc` status |
| Too many pods on node (ENI/IP limit) | `MaxPods` on nodes | prefix delegation or more nodes |

**Verify:** `kubectl get pod -o wide` shows a node and `Running`.

---

## ⚡ Runbook 2 — `ImagePullBackOff` / `ErrImagePull`

**Symptoms:** container never starts; `RESTARTS` count may grow.

```bash
kubectl describe pod <pod> | grep -A5 Events
# "Failed to pull image ... repository does not exist"  →  name gone wrong
# "pull access denied" / "authorization failed"         →  ECR auth issue
# "no basic auth credentials"                           →  imagePullSecrets missing
# "dial tcp: lookup ... no such host"                   →  DNS / no internet egress
```

**Fixes:** correct image name+tag; set up ECR (`aws ecr get-login-password | docker login`), add `imagePullSecrets`, fix egress NetworkPolicy to allow `443`.

---

## ⚡ Runbook 3 — CrashLoopBackOff

**Symptoms:** pod restarts continuously; `RESTARTS` nonzero.

```bash
kubectl logs <pod> --previous          # the crash output (container = OOM or panic?)
kubectl describe pod <pod>             # Last State: Exit Code
```

**Common causes:**
- **Exit 137 / OOMKilled** → increase memory `limits`, or fix a memory leak
- **Exit 1** (app panic) → read the `--previous` logs
- **Liveness probe failing** → app up but probe path/port wrong; check
  `kubectl describe pod` for `Liveness probe failed`
- **Secret/ConfigMap missing** → container fails at startup parsing env
- **read-only filesystem** → restricted PSS + app writing to `/tmp`

---

## ⚡ Runbook 4 — Node `NotReady` / cordoned

**Symptoms:** `kubectl get nodes` shows `NotReady` or `SchedulingDisabled`; new
pods unschedulable.

```bash
kubectl get nodes -o wide        # which node + internal IP
kubectl describe node <node>     # Conditions + events
# KubeletNotReady → agent/driver issue on node
# aws-node / kube-proxy not healthy → CNI broken
```

**Checks & fixes:**
1. `ssh` to node → `systemctl status kubelet`, `journalctl -u kubelet -f`
2. Managed node group: reuse old AMI (roll back `eksctl upgrade nodegroup`)
3. If entire nodegroup down → ASG replacement; check `aws ec2 describe-instances`
4. Uncordon: `kubectl uncordon <node>`

---

## ⚡ Runbook 5 — Service "Connection refused / no route"

**Symptoms:** app is Running but `curl <service>` fails or DNS doesn't resolve.

```
┌────────────┐  kubectl port-forward ok?  ┌────────────┐
│ DNS test:  │───────────────────────────►│ Service    │
│ nslookup   │                            │ selectors? │
└────────────┘                            └─────┬──────┘
                                                ▼
                                     Pod labels match selector?
```

**Steps:**
```bash
kubectl get endpoints <svc>                  # empty → selector mismatch
kubectl describe svc <svc>                   # check selector + ports
kubectl get pods --show-labels               # compare pod labels vs selector
kubectl run test --image=busybox --rm -it -- wget -qO- http://<svc>.<ns>.svc:80
```
- ClusterIP only resolves inside cluster → use `curl http://service:port`
  from a pod, not from your laptop/Mac.

---

## ⚡ Runbook 6 — Can't talk to `aws-load-balancer-controller` / ALB never `Active`

**Symptoms:** Ingress created but ALB not provisioned; annotation errors.

```bash
kubectl get ingress -A
kubectl -n kube-system logs deploy/aws-load-balancer-controller --tail=50
# "AccessDenied" → IRSA role missing required actions
# "target group ... not found" → wrong subnets annotation
# "failed to find ingress class" → wrong class / missing --ingress-class flag
```

**Checklist:** AWS Load Balancer Controller installed, `--with-oidc` + IRSA
roles correct, public/private subnets tagged
(`kubernetes.io/cluster/<name>: shared`, `kubernetes.io/role/elb: 1`).

---

## ⚡ Runbook 7 — HPA never scales / metrics missing

**Symptoms:** `kubectl get hpa` shows `<unknown>` targets.

```bash
kubectl describe hpa <name>        # events: failed to get metrics
kubectl top nodes / top pods       # metrics-server installed?
kubectl get apiservice | grep metrics
```

**Fix:** install metrics-server; ensure metrics pipeline (Prometheus-Adapter /
KEDA / CloudWatch) is annotated `prometheus.io/scrape=true`; HPA target type
matches metric name (`custom`, `object`, `pods`, or `resource`).

---

## ⚡ Runbook 8 — Intermittent 5xx from ALB / backend

**Symptoms:** end users see 502/503 under load.

```bash
alb logs (or `kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller`)
kubectl get pods -o wide | grep <app>      # pods spread across nodes?
kubectl top pods                          # limits reached → CPU throttling / OOM
kubectl get events                        # Evicting / NodePressure → check disk
```

**Fix:** right-size limits, increase replicas/HPA, add PDB (not a scaling fix),
check `disk-pressure` on nodes (clean up images/logs).

---

## ⚡ Runbook 9 — Disk Pressure on Nodes

```bash
kubectl describe node <node> | grep -i pressure
ssh <node> && df -h / && docker system prune -af     # or containerd
# fix: container runtime cleanup, trash-free /var/lib/docker/containerd,
# add EBS volume / increase instance size
```

---

## ⚡ Runbook 10 — Cluster Upgrade Failed / Add-on stuck

```bash
eksctl get cluster --name <c> --region <r>
eksctl upgrade cluster --name <c>   # control plane: 2x minor max
eksctl upgrade nodegroup --cluster <c> --name <ng>   # capacity: must re-use AMI
```

**Golden rules on EKS:** upgrade control plane first, then nodegroup; never
jump >1 minor version in one shot; upgrade add-ons (`eksctl create addon -f`)
*before* or alongside nodegroup.

---

## 🧰 Universal Five-Command Triage

```bash
kubectl get events --sort-by='.lastTimestamp' -A | tail -40   # 1
kubectl get pods -A -o wide                                    # 2
kubectl get nodes                                              # 3
kubectl describe <resource>                                    # 4
kubectl logs <pod> --previous                                   # 5
```

---

## 🧠 Also See

- [`EKS_QUICK_REFERENCE.md`](EKS_QUICK_REFERENCE.md) — commands, probes, security checklist
- [`EKS_Learning/labs/Lab-04_Troubleshooting_Game.md`](EKS_Learning/labs/Lab-04_Troubleshooting_Game.md) — practice debugging 4 broken stacks
- [`EKS_Learning/modules/README.md`](EKS_Learning/modules/README.md) — deeper dives

*Playbook last updated: September 2026*