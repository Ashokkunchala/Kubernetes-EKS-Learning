# 📅 Day 41: Prometheus Operator for EKS

## 🎯 Learning Objectives
By the end of today, you will understand:
- Prometheus & the Prometheus Operator architecture
- kube-prometheus-stack (Prometheus + Alertmanager + Grafana)
- ServiceMonitors, PodMonitors, and scrape configurations
- PromQL queries for EKS metrics
- AWS EKS Managed Prometheus (AMP) and CloudWatch comparison

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Metric** | A numeric measurement observed over time. |
| **Scrape** | Prometheus periodically requests metrics from a target endpoint. |
| **ServiceMonitor** | A Prometheus Operator custom resource describing how to discover and scrape matching Services. |
| **Operator** | A Kubernetes controller that automates management of a particular application or system. |
| **Time series** | A sequence of timestamped values for one metric and its label set. |
| **Alert rule** | A condition evaluated over metrics that can create an alert when it remains true. |

**Analogy:** Prometheus is a meter reader, ServiceMonitor tells it which meters to read, and an alert rule defines when a reading needs attention.

**Try / observe:** `kubectl get pods,svc -n <monitoring-namespace>`, `kubectl get servicemonitor -A`, and inspect Prometheus targets and rule status.

**Troubleshooting clue:** A ServiceMonitor existing does not prove Prometheus selected it. Check label selectors, target endpoints, Service labels, port names, and scrape errors.

**Explain it back:** What is the difference between a Service and a ServiceMonitor? What does a scrape do?

---

## 📖 Core Concepts

### Prometheus Operator Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                 PROMETHEUS OPERATOR                            │
│                                                               │
│  ┌──────────────┐    ┌──────────────┐     ┌──────────────┐   │
│  │  Prometheus  │    │ Alertmanager │     │   Grafana    │   │
│  │  (StatefulSet│    │   (Deploy)   │     │  (Deploy)    │   │
│  │   -scrape)  │    │   -dedup     │     │  -dashboard  │   │
│  └──────┬───────┘    └──────┬───────┘     └──────┬───────┘   │
│         │                   │                     │           │
│  ┌──────▼───────────────────▼─────────────────────▼───────┐   │
│  │               OPERATOR (controller)                     │   │
│  │  watches CRDs & reconciles:                             │   │
│  │  · ServiceMonitor (app metrics)                         │   │
│  │  · PodMonitor (pod-level)                               │   │
│  │  · PrometheusRule (alert rules)                         │   │
│  │  · AlertmanagerConfig                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Scrape targets:                                        │   │
│  │   · kube-apiserver (kube-metrics)                       │   │
│  │   · kubelet/cAdvisor (node+container)                   │   │
│  │   · node-exporter (node)                                │   │
│  │   · kube-state-metrics (object state)                   │   │
│  │   · App /metrics endpoints (ServiceMonitor)             │   │
│  └─────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────┘
```

### Key CRDs

| CRD | Purpose |
|-----|---------|
| `ServiceMonitor` | Scrape app metrics from a Service's selector |
| `PodMonitor` | Scrape metrics directly from selected Pods |
| `PrometheusRule` | Define alert rules + recording rules |
| `AlertmanagerConfig` | Configure notification routing per-namespace |
| `Prometheus` | Configure the Prometheus instance (retention, storage) |

---

## 🛠️ Install kube-prometheus-stack

### Via Helm (recommended)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create ns monitoring

read -s -p "Lab-only Grafana admin password: " GRAFANA_ADMIN_PASSWORD; echo
helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword="$GRAFANA_ADMIN_PASSWORD" \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
unset GRAFANA_ADMIN_PASSWORD
```

### Verify Components

```bash
kubectl -n monitoring get pods
kubectl -n monitoring get svc
kubectl -n monitoring get crd | grep monitoring.coreos.com
```

Port-forward to the UIs:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-grafana 3000:80
# Grafana: http://localhost:3000 (username: admin; use the password entered above)

kubectl -n monitoring port-forward svc/kube-prometheus-prometheus 9090:9090
# Prometheus UI: http://localhost:9090
```

---

## 🛠️ Scrape Your Application

### Expose `/metrics` then add a ServiceMonitor

```yaml
# app-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: production
  labels:
    release: kube-prometheus   # must match Prometheus selector
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

```yaml
# App Service with the metrics port labeled
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
spec:
  selector:
    app: my-app
  ports:
  - name: http
    port: 8080
    targetPort: 8080
```

Verify the target is scraped:

```bash
# In Prometheus UI: Status → Targets
# Or from CLI if configured
kubectl -n monitoring exec -it $(kubectl -n monitoring get pod -l app.kubernetes.io/name=prometheus -o name | head -1) -- wget -qO- localhost:9090/api/v1/targets | head
```

---

## 🛠️ PromQL Queries for EKS

```promql
# CPU usage of a namespace
sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m]))

# Memory usage of a namespace
sum(container_memory_working_set_bytes{namespace="production"})

# Pod restart count
sum(increase(kube_pod_container_status_restarts_total{namespace="production"}[1h]))

# Node CPU utilization %
100 - (avg by (instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# API server request latency
histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket[5m])) by (le))

# Container network receive
sum(rate(container_network_receive_bytes_total{namespace="production"}[5m]))
```

---

## 🛠️ Alert Rules

```yaml
# app-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
  namespace: production
  labels:
    release: kube-prometheus
spec:
  groups:
  - name: app
    rules:
    - alert: PodCrashLooping
      expr: kube_pod_container_status_restarts_total > 5
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash-looping"
    - alert: HighCPU
      expr: sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m])) by (pod) > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage on {{ $labels.pod }}"
```

---

## 🛠️ Amazon Managed Prometheus (AMP) & CloudWatch

### EKS Control Plane Metrics in CloudWatch (always on)

```bash
# Control plane metrics auto-published to CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/EKS \
  --metric-name cluster_failed_request_count \
  --dimensions Name=cluster_name,Value=my-cluster \
  --start-time 2026-09-01T00:00:00Z --end-time 2026-09-01T01:00:00Z \
  --period 300 --statistics Average
```

### Send Prometheus metrics to AMP

```bash
# Create AMP workspace
aws amp create-workspace --alias my-amp
# Returns workspaceId -> endpoint:
# https://aps-workspaces.<region>.amazonaws.com/workspaces/<id>

# Install the adot/amp agent to remote-write to AMP
helm repo add aws-observability https://aws-observability.github.io/helm-charts
helm install aws-otel-collector aws-observability/aws-otel-collector \
  --namespace monitoring \
  --set mode="deployment" \
  --set config.service.pipelines.metrics.exporters[0]="awsemf"
```

> **Best practice:** Use **kube-prometheus-stack locally** for high-churn app/metrics governance, and **AMP** for scalable, managed long-term storage + PromQL at the platform level. Avoid scraping the control plane yourself.

---

## 📝 Interview Questions

1. **What is the Prometheus Operator?**
   - **Answer:** A controller that manages Prometheus, Alertmanager, and Grafana deployments, and uses CRDs (ServiceMonitor, PrometheusRule) to define scraping and alerting declaratively.

2. **What's the difference between ServiceMonitor and PodMonitor?**
   - **Answer:** ServiceMonitor scrapes via a Service (endpoints); PodMonitor scrapes Pod IPs directly — useful when there's no Service (e.g., background jobs).

3. **How do you monitor your EKS application?**
   - **Answer:** Expose a `/metrics` endpoint, add a ServiceMonitor, verify targets, write PromQL dashboards in Grafana, and add PrometheusRules for alerts.

4. **Prometheus vs CloudWatch for EKS?**
   - **Answer:** CloudWatch gives control-plane + AWS-native metrics with no setup; Prometheus gives powerful PromQL, multi-dimensional app metrics, and the CNCF ecosystem. Use both — AMP bridges them.

5. **How do you store Prometheus data long-term?**
   - **Answer:** Local persistent volume for short-term, remote-write to AMP (or Thanos/Cortex) for long-term scalable storage and cross-cluster queries.

---

## 📚 Resources for Review

- [Day 14: EKS Monitoring & Logging](Day-14_EKS_Monitoring_Logging/README.md)
- [Day 42: Grafana Dashboards](Day-42_Grafana_Dashboards/README.md)

### External
- [Prometheus](https://prometheus.io/docs/)
- [Prometheus Operator](https://prometheus-operator.dev/)
- [Amazon Managed Prometheus](https://docs.aws.amazon.com/prometheus/latest/userguide/)

---

## 🎯 Today's Assignment

1. **Install** kube-prometheus-stack on EKS
2. **Deploy** a sample app and add a ServiceMonitor
3. **Write** 5 PromQL queries for CPU/memory/restarts
4. **Create** at least 2 alert rules
5. **Explore** the Targets and Rules pages in the UI

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

**Day 42:** Grafana Dashboards

---

**Remember:** "If you can't measure it, you can't improve it" — instrument everything! 🚀
