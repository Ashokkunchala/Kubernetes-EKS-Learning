# 📅 Day 45: Monitoring Stack Practice - Full Observability Lab

## 🎯 Learning Objectives
By the end of today, you will:
- Build a complete observability stack: Prometheus + Grafana + X-Ray + CloudWatch
- Add logging with Loki/CloudWatch Logs
- Correlate metrics, logs, and traces
- Set up alerting across the whole stack
- Validate SLOs and incident response

---

## 📖 The Complete Observability Stack

```
┌───────────────────────────────────────────────────────────────┐
│                 FULL OBSERVABILITY ON EKS                     │
│                                                               │
│   METRICS          LOGS            TRACES                     │
│   ┌─────────┐      ┌─────────┐     ┌─────────┐                │
│   │Prometheus│      │  Loki / │     │  X-Ray /│               │
│   │(kube-prom)│     │CloudWatch│     │  Tempo │               │
│   └────┬────┘      └────┬────┘     └────┬────┘               │
│        │                │              │                     │
│        └────────┬───────┴──────┬───────┘                     │
│                 ▼              ▼                              │
│            ┌─────────────────────────┐                       │
│            │          GRAFANA        │                       │
│            │  (single pane of glass) │                       │
│            └─────────────────────────┘                       │
│                      │                                       │
│                      ▼                                       │
│            ┌─────────────────────────┐                       │
│            │   ALERTING (Ops channel)│                       │
│            │   Slack / SNS / PagerDuty│                      │
│            └─────────────────────────┘                       │
└───────────────────────────────────────────────────────────────┘
```

**Pillars:** Measure (metrics), Understand (traces), Inspect (logs).

---

## 🛠️ Step 1: Install Core Stack (Recap)

```bash
# Namespaces
kubectl create ns monitoring observability

# 1. Prometheus/Grafana (Day 41)
helm install kube-prom prometheus-community/kube-prometheus-stack \
  --namespace monitoring

# 2. Loki for logs
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack --namespace monitoring \
  --set grafana.enabled=false

# 3. X-Ray/ADOT (Day 43)
helm install adot aws-observability/aws-observability-collector \
  --namespace observability
```

---

## 🛠️ Step 2: Connect Data Sources in Grafana

```yaml
# datasources (as code, Day 42)
- name: Prometheus
  type: prometheus
  url: http://kube-prom-prometheus.monitoring.svc:9090
- name: Loki
  type: loki
  url: http://loki.monitoring.svc:3100
- name: CloudWatch
  type: cloudwatch
  access: proxy
  jsonData: { authType: ec2_iam, defaultRegion: us-east-1 }
- name: X-Ray
  type: grafana-x-ray-datasource
  jsonData: { authType: ec2_iam, defaultRegion: us-east-1 }
```

---

## 🛠️ Step 3: Unified Service Dashboard

Build one **"Service Overview"** dashboard per microservice with panels:

| Panel | Source | Query |
|-------|--------|-------|
| Request rate | Prometheus | `sum(rate(http_requests_total{svc="orders"}[5m]))` |
| Error rate | Prometheus | `sum(rate(http_requests_total{status=~"5..",svc="orders"}[5m]))` |
| P99 latency | Prometheus | `histogram_quantile(0.99, sum(rate(http_duration_seconds_bucket[5m])) by (le))` |
| Crash loops | Prometheus | `increase(kube_pod_container_status_restarts_total{app="orders"}[30m])` |
| App logs (errors) | Loki | `{namespace="production"} |= \"error\"` |
| Slow traces | X-Ray | Trace latency by service |

Enable **templating** with a `$service` variable so one dashboard serves all services.

---

## 🛠️ Step 4: Centralized Alerting

### Prometheus Rules (via PrometheusRule)

```yaml
groups:
- name: servicehealth
  rules:
  - alert: ServiceDown
    expr: up{job="orders"} == 0
    for: 2m
    annotations: { summary: "orders service unreachable" }
  - alert: HighErrorRate
    expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
    for: 5m
    labels: { severity: critical }
```

### Route alerts to Slack via Alertmanager

```yaml
# alertmanager config (Secret)
route:
  receiver: slack-ops
  group_by: [namespace]
receivers:
- name: slack-ops
  slack_configs:
  - channel: "#ops-alerts"
    api_url: "https://hooks.slack.com/services/T00/B00/XXXX"
```

---

## 🛠️ Step 5: Correlate Traces + Logs (+Incident drills)

```bash
# When an alert fires:
# 1. Check Grafana dashboard for the service
# 2. Jump from HTTP 5xx panel → Loki logs filtered by pod/trace
# 3. Jump from latency panel → X-Ray trace timeline
# 4. Find the failing span → its log line → root cause
```

**Run a drill:**

```bash
# Generate a controlled failure and practice triage
kubectl -n production exec deploy/orders -- curl -X POST localhost:8080/fail
# → watch alert fire, grab trace id, find logs, post incident summary
```

---

## 📊 SLO Validation

```promql
# Availability SLO (99.9% over 30d)
(1 - (sum(rate(http_requests_total{code=~"5.."}[30d])) /
       sum(rate(http_requests_total[30d])))) * 100
```

Alert when the error budget is nearly exhausted:

```promql
# Remaining error budget < 10%
1 - (sum(rate(http_requests_total{code=~"5.."}[30d])) /
      sum(rate(http_requests_total[30d]))) < 0.001
```

---

## 📝 Interview Questions

1. **Describe your full observability stack on EKS.**
   - **Answer:** kube-prometheus-stack for metrics + alerting, Loki/CloudWatch for logs, X-Ray (ADOT) for traces, all visualized in Grafana with a per-service dashboard and correlated trace→log navigation.

2. **How do you correlate metrics, logs, and traces?**
   - **Answer:** Use a common trace ID injected into log lines and metric labels; Grafana scatter/graph links let you jump from an abnormal metric to the matching logs and trace timeline.

3. **How do you set up alerting without noise?**
   - **Answer:** Derive alerts from **error budgets** rather than raw thresholds, group by namespace/service, use `for:` duration to avoid flapping, and route by severity with on-call rotation.

4. **What is an SLO and how do you monitor one?**
   - **Answer:** An availability/latency target (e.g., 99.9% availability, P99 < 500ms) over a rolling window; monitor burn rate and alert before the budget is exhausted.

5. **How do you on-board a new microservice to monitoring?**
   - **Answer:** Add /metrics + HTTP instrumentation, ServiceMonitor, a templated dashboard, PrometheusRule alerts, and structured logs with trace IDs — followed by a service overview review.

---

## 📚 Resources for Review

- Day 41-44 (Prometheus, Grafana, X-Ray, CloudWatch)
- [Day 14: EKS Monitoring & Logging](Day-14_EKS_Monitoring_Logging/README.md)

---

## 🎯 Today's Assignment

1. **Build** the 4-pillar dashboard per microservice
2. **Enable** Loki and visualize error logs
3. **Create** an error-budget alert
4. **Run** a full incident drill end-to-end
5. **Document** the runbook for alert → trace → log triage

---

## 📊 Progress Tracker

```
Week 13-14 MONITORING COMPLETE:
Day 41: Prometheus Operator ✅
Day 42: Grafana Dashboards ✅
Day 43: AWS X-Ray Tracing ✅
Day 44: Custom Metrics with CloudWatch ✅
Day 45: Monitoring Stack Practice ⬜
```

---

## 🔗 Next Week Preview

**Week 15-16:** Multi-Cluster Management (EKS Anywhere, Mesh, Federation)

---

**Remember:** Observability is only as good as the response it drives. Practice the drill! 🚀
