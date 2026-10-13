# 📅 Day 44: Custom Metrics with CloudWatch

## 🎯 Learning Objectives
By the end of today, you will understand:
- Kubernetes Event-Driven Autoscaling (KEDA)
- Publishing custom metrics to CloudWatch
- Horizontal Pod Autoscaler (HPA) with custom/external metrics
- CloudWatch Embedded Metric Format (EMF)
- Scaling on custom business metrics

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Custom metric** | A measurement your application or monitoring pipeline defines and publishes. |
| **Namespace (CloudWatch)** | A grouping name for related CloudWatch metrics; distinct from a Kubernetes Namespace. |
| **Dimension** | A key/value label used to distinguish metric series, such as service or environment. |
| **CloudWatch** | AWS monitoring for metrics, logs, alarms, and related telemetry. |
| **HPA** | Horizontal Pod Autoscaler: changes a workload's replica count based on observed metrics. |
| **Metrics adapter** | A bridge that exposes external/custom metrics in a form Kubernetes autoscalers can query. |

**Analogy:** A metric is a gauge reading, a dimension identifies which machine it belongs to, and HPA uses readings to decide how many app copies are needed.

**Try / observe:** Verify the metric exists and has recent values before debugging HPA; then inspect `kubectl get hpa` and `kubectl describe hpa <name>`.

**Troubleshooting clue:** If HPA shows unknown metrics, check the metrics API/adapter, metric name/dimensions, permissions, and whether the target metric is being published.

**Explain it back:** Why can't HPA scale from a custom metric that the metrics API cannot expose?

---

## 📖 Core Concepts

### Scaling Stack on EKS

```
┌───────────────────────────────────────────────────────────────┐
│                     AUTO-SCALING                               │
│                                                               │
│  ┌──────────────┐   Custom/Business metrics                   │
│  │  CloudWatch  │◀──────────────┐                             │
│  │  (EMF)       │               │ app writes metric           │
│  └──────┬───────┘               │                             │
│         │ gets metric            │                             │
│         ▼                       ▼                             │
│  ┌──────────────┐   ┌─────────────────────┐                   │
│  │   KEDA       │   │   HPA (native)      │                   │
│  │ (external)   │   │  CPU/memory/custom  │                   │
│  └──────┬───────┘   └─────────┬───────────┘                   │
│         │                     │                               │
│         └──────────┬──────────┘                               │
│                    ▼                                          │
│            ┌───────────────┐                                  │
│            │ ReplicaSet /  │ → scale Deployment up/down       │
│            │ ScaleTarget   │                                  │
│            └───────────────┘                                  │
└───────────────────────────────────────────────────────────────┘
```

### Metric Paths

- **Resource metrics:** CPU/memory (from cAdvisor/metrics-server) — HPA default.
- **Custom metrics:** exposed via `/metrics`, aggregated to CloudWatch or Prometheus.
- **External metrics:** business metrics not tied to pods (e.g., SQS queue depth) — KEDA's specialty.

---

## 🛠️ Option 1: HPA with Custom Metrics Server

### Install custom-metrics adapter (Prometheus → HPA)

```bash
# Prometheus Adapter exposes app metrics to HPA
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  --set prometheus.url=http://kube-prometheus-prometheus.monitoring.svc:9090
```

```bash
# Enable an HPA on a custom metric (requires adapter rules)
kubectl autoscale deployment my-app --custom-metric=requests_per_second --min=2 --max=20
```

---

## 🛠️ Option 2: Publish Custom Metrics to CloudWatch with EMF

### CloudWatch Agent with EMF

Use the **AWS distro for OTEL (ADOT)** or the CloudWatch agent to emit custom metrics via EMF:

```yaml
# emf-export with put_metric_data via boto3 (simplest)
import boto3, os
cloudwatch = boto3.client('cloudwatch', region_name='us-east-1')

def emit_metric(name, value):
    cloudwatch.put_metric_data(
        Namespace='MyApp',
        MetricData=[{
            'MetricName': name,
            'Value': value,
            'Unit': 'Count',
            'Dimensions': [{'Name': 'Pod', 'Value': os.environ['POD_NAME']}]
        }]
    )
```

### Give pods IRSA permission to publish

```yaml
# Service account for the app
aws eksctl create iamserviceaccount \
  --name my-app-sa --namespace production \
  --attach-policy-arn arn:aws:iam::aws:policy/CloudWatchFullAccess \
  --approve
```

---

## 🛠️ Option 3: KEDA with CloudWatch ScaledObject (Recommended)

KEDA reads **external metrics** (like CloudWatch) and scales HPAs automatically.

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
kubectl create ns keda
helm install keda kedacore/keda --namespace keda
```

### Scale on SQS queue depth

```yaml
# sqs-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: sqs-worker
  namespace: production
spec:
  scaleTargetRef:
    name: worker
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/ACCOUNT/my-queue
      queueLength: "10"
      awsRegion: us-east-1
      identityOwner: operator
  minReplicaCount: 1
  maxReplicaCount: 20
```

```bash
kubectl apply -f sqs-scaledobject.yaml
# Now the worker Deployment scales with queue depth
```

### Scale on CloudWatch metric

```yaml
# cw-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: custom-scale
  namespace: production
spec:
  scaleTargetRef:
    name: my-app
  triggers:
  - type: aws-cloudwatch
    metadata:
      expression: "SELECT AVG(RequestCount) FROM SCHEMA(\"MyApp\", Pod)"
      targetMetricValue: "100"
      metricCollectionTime: "300"
```

---

## 🛠️ Verify Scaling Behavior

```bash
# Check HPA status
kubectl get hpa -n production
kubectl describe hpa my-app -n production

# Watch scale events
kubectl get events -n production --sort-by=.lastTimestamp | grep -i scal

# Confirm KEDA in action
kubectl get scaledobject -n production
kubectl get triggerauthentication -n production
```

---

## 🛠️ CloudWatch Dashboards & Alarms

```bash
# Visualize custom metrics
aws cloudwatch list-metrics --namespace MyApp

# Create an alarm on the custom metric
aws cloudwatch put-metric-alarm \
  --alarm-name high-request-rate \
  --namespace MyApp --metric-name RequestCount \
  --threshold 500 --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 --period 300 --statistic Sum \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT:ops-topic
```

---

## 📝 Interview Questions

1. **What are the three types of metrics used for autoscaling, and when do you use each?**
   - **Answer:** Resource (CPU/memory) for baseline; Custom (Prometheus/EMF app metrics) for workload-specific scaling; External (CloudWatch/SQS) for event/queue-driven scaling via KEDA.

2. **What is KEDA and why use it?**
   - **Answer:** It's a Kubernetes autoscaler that scales Deployments based on external/custom metrics (SQS, CloudWatch, Kafka) — scaling on events, which the native HPA can't do directly.

3. **How do you expose a custom metric to HPA?**
   - **Answer:** Aggregate it (Prometheus adapter or CloudWatch/EKS metric adapter) into the metrics API; native HPA then reads it via `--custom-metric` or the API.

4. **What's the difference between HPA and Cluster Autoscaler/Karpenter?**
   - **Answer:** HPA scales the number of pods; Cluster Autoscaler/Karpenter scale the number of nodes. KEDA drives HPAs from external signals.

5. **How do you send custom metrics to CloudWatch efficiently?**
   - **Answer:** Use Embedded Metric Format (EMF) or batched `put_metric_data` to reduce API calls and cost; dimension metrics by pod/service for granularity.

---

## 📚 Resources for Review

- [Day 53: Cluster Autoscaler vs Karpenter](Day-53_Cluster_Autoscaler_Karpenter/README.md)
- [Day 41: Prometheus Operator](Day-41_Prometheus_Operator/README.md)

### External
- [KEDA](https://keda.sh/docs/)
- [AWS CloudWatch custom metrics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/publishingMetrics.html)

---

## 🎯 Today's Assignment

1. **Install** KEDA
2. **Add** a CloudWatch embedded metric from your app
3. **Create** a ScaledObject scaling on SQS length
4. **Load-test** and confirm pods scale up/down
5. **Set up** a CloudWatch alarm on a custom metric

---

## 📊 Progress Tracker

```
Week 13-14 Monitoring & Observability:
Day 41: Prometheus Operator ⬜
Day 42: Grafana Dashboards ⬜
Day 43: AWS X-Ray Tracing ⬜
Day 44: Custom Metrics with CloudWatch ⬜
Day 45: Monitoring Stack Practice ⬜
```

---

## 🔗 Next Week Preview

**Day 45:** Monitoring Stack Practice - the full observability lab

---

**Remember:** Scale on what matters to the business (queue depth, requests), not just CPU — that's true autoscaling! 🚀
