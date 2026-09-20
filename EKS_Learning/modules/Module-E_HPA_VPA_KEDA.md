# 🧩 Module E: Autoscaling Decisions — HPA, VPA, KEDA

> **Prereq:** [Day 44 Custom Metrics CloudWatch](../Day-44_Custom_Metrics_CloudWatch/README.md) · [Day 52 Right-Sizing Nodes](../Day-52_Right_Sizing_Nodes/README.md) · [Day 53 Cluster Autoscaler & Karpenter](../Day-53_Cluster_Autoscaler_Karpenter/README.md)

Three tools scale the *same* resource — your Pod — but answer three *different* questions: **horizontal** (how many?), **vertical** (how big?), and **event** (how bursty?). Choosing the wrong one is a common EKS production mistake.

## 🎯 Learning Objectives

By the end of this module, you will understand:
- The three scaling dimensions and when each applies
- HPA v2 with multiple metric types (resource, object, external, pods)
- `autoscaling.k8s.io/v1` VPA modes: Off / Initial / Recreate / Auto
- KEDA `ScaledObject` for HTTP + queue-style triggers
- A decision flow for picking HPA vs VPA vs KEDA
- The end-to-end scaling flow on EKS (metrics → HPA → replicas → autoscaler → nodes)

---

## 📖 Core Concepts

### The three questions

```
┌─────────────────────────────────────────────────────────┐
│          WHAT ARE WE ACTUALLY SCALING?                   │
│                                                         │
│  HPA  → HOW MANY pods?      (workload varies by load)   │
│  VPA  → HOW BIG is a pod?   (per-pod size varies)       │
│  KEDA → WHEN to fire?       (queues/events/HTTP bursts) │
│                                                         │
│  all three feed the NODE autoscaler (CA/Karpenter):     │
│     replicas increase → pending pods → node capacity    │
│     needed → ClusterAutoscaler or Karpenter adds nodes  │
└─────────────────────────────────────────────────────────┘
```

### Scaling flow on EKS (the full picture)

```
┌─────────────────────────────────────────────────────────┐
│        FULL SCALING LOOP ON EKS                          │
│                                                         │
│  [Metrics Server]   (resource: cpu/mem)                 │
│  [Prometheus]       (custom app metrics)                 │
│  [CloudWatch Adapter] (EKS + CW metrics)                 │
│        │                                                │
│        ▼                                                │
│  [HPA / KEDA ScaledObject]  → reads metric + target      │
│        │                                                │
│        ▼                                                │
│  [Deployment replicas]  scale up/down                    │
│        │                                                │
│        ▼                                                │
│  [Pending Pods on node]          [VPA recommends sizing] │
│        │                                      │          │
│        ▼                                      ▼          │
│  [ClusterAutoscaler / Karpenter]        [VPA updates    │
│        │  adds EC2 node                requests/limits] │
│        ▼                                                │
│  [node group / Spot / instance type]                    │
└─────────────────────────────────────────────────────────┘
```

---

## 🚀 Hands-On

### HPA — `autoscaling/v2` with mixed metrics

```yaml
# hpa-example.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment        # requires pod requests set!
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization          # avg across all replicas
        averageUtilization: 60
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second   # custom metric (Prometheus adapter)
      target:
        type: AverageValue
        averageValue: "100"
  behavior:                      # tune the "rubber band" (v2+)
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5min before shrinking
      policies:
      - type: Percent
        value: 20
        periodSeconds: 60
```

```bash
# Requires the metrics pipeline available to HPA:
kubectl apply -f hpa-example.yaml
kubectl get hpa web-hpa -w
# NAME     REFERENCE      TARGETS          MINPODS  MAXPODS  REPLICAS  AGE
# web-hpa  Deployment/..  45%/60% + 88/100  2        10       6         3m

# Manual break-glass test
kubectl run load --rm -it --restart=Never --image=busybox \
  -- sh -c 'while true; do wget -q -O- http://my-deployment-svc:80 ; done'
```

### VPA — recommender / mutating modes

```yaml
# vpa-example.yaml  (vertical, for right-sizing requests)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  updatePolicy:
    ## Recommendation only — never mutates pods (SAFEST on EKS)
    updateMode: "Off"
    # "Initial"     → set only at pod creation
    # "Recreate"    → terminate+recreate pods to apply new requests
    # "Auto"        → same as Recreate (default)
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: 50m
        memory: 50Mi
      maxAllowed:
        cpu: "4"
        memory: 4Gi
```

```bash
kubectl apply -f vpa-example.yaml
kubectl describe vpa web-vpa | grep -A8 Recommendation
# Recommendation:
#   Container  app:
#     Lower Bound:  100m      128Mi
#     Target:       320m      512Mi
#     Uncapped Target: 320m   512Mi
#     Upper Bound:  4         4Gi
```

> ⚠️ VPA + HPA **must not** target the same metric (cpu/mem) at the same time — they fight. Rule: HPA for the number, VPA only for fixed-burst workloads you know well.

### KEDA — `ScaledObject` for HTTP + queue triggers

```yaml
# scaledobject-http.yaml   HTTP+based scale from 0 (needs KEDA HTTP add-on)
apiVersion: http.keda.sh/v1alpha1
kind: HTTPScaledObject
metadata:
  name: web-scaler
spec:
  hosts: [web.example.com]
  targetPendingRequests: 100          # scale when pending HTTP > 100
  scaledownPeriod: 60
  scaleTargetRef:
    deployment: my-deployment
---
# scaledobject-sqs.yaml   queue trigger (AWS)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-scaler
spec:
  scaleTargetRef:
    name: worker-deployment
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123456789012/my-queue
      queueLength: "100"           # pending messages per worker
      awsRegion: us-east-1
```

```bash
# Install KEDA operators on EKS
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --create-namespace
kubectl get crd scaledobjects.keda.sh   # confirm

# Scale from ZERO: KEDA reaches into HPA's depths — Pods can go 0 → N on demand
```

---

## 🧭 When to choose which — the decision flow

```
Workload has a fixed per-pod size (CPU/mem) + load varies?
   └─ YES → HPA on Resource metrics          (autoscaling/v2)
Workload is batchy/event-y (queue, cron, HTTP latency)?
   └─ YES → KEDA ScaledObject               (fires on events, can hit 0)
Workload is a long-lived, vertical bottleneck (big DB-like pods)?
   └─ YES → VPA (Start "Off" + adopt the Recommendation first)
Mixed?
   └─ HPA (count) across metrics, VPA (sizing) ONLY if embarrassingly static caps

Remember:
  - HPA needs resource REQUESTS set on the Deployment (it reads usage/request)
  - KEDA can replace HPA entirely for event-driven; both can coexist via annotations
  - VPA is destructive (Recreate) — run "Off" → OPA/Kyverno to observe first
  - Node autoscaler (CA/Karpenter) is ADDITIVE — pods scale, then nodes scale
```

---

## 🛠️ Troubleshooting

```bash
# 1. HPA stuck at minReplicas / reports <unknown> metrics:
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods   # is MetricsServer alive?
kubectl top pods                                       # must return rows

# 2. HPA never scales with custom metric:
kubectl describe hpa web-hpa
#   look under "Status.Metrics[0]" — the metric NAME must match the adapter export

# 3. VPA pods churning (Recreate storm): Lower your maxAllowed /
#    watch the "uncapped target" vs current; let it settle 24h.
kubectl get events --sort-by=.lastTimestamp | grep -i vpa

# 4. KEDA ScaledObject not creating pods at 0:
kubectl get hpa -n <ns>            # KEDA creates an HPA under the hood
kubectl describe scaledobject <name>

# 5. On EKS, Prometheus adapter must use the right namespace label
#    (kube-prometheus-stack.data: podinfo...); validate with:
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | grep http_requests
```

---

## ❓ Interview Questions

1. **HPA vs VPA — when would you use each?**
   - HPA scales replicas based on averaged metrics (good for stateless load); VPA adjusts request/limit sizes per pod (good for fixed-capacity workloads). They overlap on CPU/mem and must not be combined there.

2. **Why does HPA need resource requests?**
   - HPA's `averageUtilization` is computed as usage/request; without requests the denominator is undefined and the HPA reports `<unknown>`.

3. **What does KEDA add that HPA cannot do alone?**
   - Event-driven scaling: HTTP, SQS, Kafka, Prometheus, cron... and **scale-to-zero** — HPA with copied metrics cannot go below its configured `minReplicas`.

4. **What is `updateMode: Off` in VPA and why recommend it?**
   - The VPA computes recommendations but does not mutate live pods; you adopt sizes deliberately, avoiding Recreate storms — the recommended onboarding path for existing production apps.

5. **How do pod scaling and node scaling interact on EKS?**
   - HPA/KEDA add pods; pending (unschedulable) pods then trigger ClusterAutoscaler/Karpenter to add EC2 capacity — scale pods first, nodes react second.

---

## 📚 Resources

- [HPA (autoscaling/v2) docs](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [VPA docs](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [KEDA docs](https://keda.sh/docs/)
- [Amazon CloudWatch adapter for EKS](https://docs.aws.amazon.com/eks/latest/userguide/cloudwatch-metrics.html)

---

## ✅ Progress Tracker

```
Module E: Autoscaling Decisions
  [x] Three dimensions: how many / how big / when
  [x] End-to-end scaling flow diagram
  [ ] Applied HPA v2 (Resource + Pods metrics, stabilization)
  [ ] Read a VPA Recommendation, updateMode Off
  [ ] Deployed KEDA HTTPScaledObject + SQS ScaledObject
  [ ] Documented the choose-which flow for my workloads
```

---

## 👀 Next

- **Node-level scaling that reacts to the pods you just added:** [Day 53 Cluster Autoscaler & Karpenter](../Day-53_Cluster_Autoscaler_Karpenter/README.md)
- **Feed HPA real CloudWatch / Prometheus metrics:** [Day 44 Custom Metrics CloudWatch](../Day-44_Custom_Metrics_CloudWatch/README.md)
- **Sizing pods before scaling begins:** [Day 52 Right-Sizing Nodes](../Day-52_Right_Sizing_Nodes/README.md)

---

**Remember:** Scale the **metric** that predicts user pain — HTTP latency and queue depth — not just CPU. 📏