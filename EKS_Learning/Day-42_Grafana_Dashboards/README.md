# 📅 Day 42: Grafana Dashboards

## 🎯 Learning Objectives
By the end of today, you will understand:
- Building and importing Grafana dashboards for EKS
- Connecting Grafana to Prometheus & CloudWatch data sources
- Creating alerts and annotations in Grafana
- Dashboard-as-code with provisioning
- Best practices for visualization and SLO tracking

---

## 📖 Core Concepts

### Grafana Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                          GRAFANA                              │
│                                                               │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐      │
│  │  Dashboards  │   │   Alerts     │   │  Annotations │      │
│  │  (panels)    │   │  (rules)     │   │  (events)    │      │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘      │
│         └────────┬─────────┴─────────┬────────┘               │
│                  ▼                   ▼                        │
│        ┌────────────────────────────────────────┐             │
│        │            DATA SOURCES                 │            │
│        │  · Prometheus (app/node metrics)        │            │
│        │  · CloudWatch (AWS/EKS control plane)   │            │
│        │  · Loki (logs - optional)               │            │
│        │  · Tempo (traces - optional)            │            │
│        │  · X-Ray (traces)                       │            │
│        └────────────────────────────────────────┘             │
└───────────────────────────────────────────────────────────────┘
```

### Why Grafana for EKS

- One pane of glass for **metrics, logs, and traces** (LGTM stack).
- Rich PromQL/CloudWatch integration and templating.
- **Dashboards-as-code** via provisioning JSON — version-controllable.
- Team-customizable SLO dashboards for exec and engineering views.

---

## 🛠️ Step 1: Add Data Sources

### Prometheus

```bash
# Port-forward Prometheus (already installed in Day 41)
kubectl -n monitoring port-forward svc/kube-prometheus-prometheus 9090:9090
```

In Grafana UI: **Configuration → Data sources → Add → Prometheus** → URL `http://kube-prometheus-prometheus.monitoring.svc:9090`.

### CloudWatch (via IRSA on the Grafana pod)

```yaml
# grafana datasource via provisioning yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://kube-prometheus-prometheus.monitoring.svc:9090
      access: proxy
      isDefault: true
    - name: CloudWatch
      type: cloudwatch
      access: proxy
      jsonData:
        authType: ec2_iam
        defaultRegion: us-east-1
```

Give the Grafana service account IRSA:

```bash
eksctl create iamserviceaccount \
  --cluster my-eks-cluster \
  --namespace monitoring \
  --name kube-prometheus-grafana \
  --attach-policy-arn arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess \
  --approve
```

---

## 🛠️ Step 2: Import a Dashboard

Use the official K8s dashboards. In Grafana: **+ → Import** → enter ID → pick Prometheus datasource.

| Dashboard | ID | Use case |
|-----------|----|----------|
| Kubernetes / Cluster | 315 | Cluster overview |
| Kubernetes / Nodes | 1860 | Node CPU/mem/network |
| Kubernetes / Views Global | 15757 | Pods, workloads |
| 1 Node Exporter Full | 1860 | Node metrics |
| Kubernetes Deployment | 13946 | Deployment health |

```bash
# Or import via Grafana API
curl -X POST http://admin:ChangeMe!@localhost:3000/api/dashboards/db \
  -H "Content-Type: application/json" \
  -d @dashboard.json
```

---

## 🛠️ Step 3: Dashboards-as-Code

Configure dashboards declaratively so they're version-controlled:

```yaml
# dashboard-provider.yaml - mount into /etc/grafana/provisioning
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards-provider
  namespace: monitoring
data:
  dashboards.yaml: |
    apiVersion: 1
    providers:
    - name: default
      orgId: 1
      folder: EKS
      type: file
      options:
        path: /var/lib/grafana/dashboards
```

```yaml
# diff: export dashboard JSON and store it:
kubectl -n monitoring get cm -o jsonpath='{.data}' 
# or export via Grafana API then commit to git
```

---

## 🛠️ Step 4: Common EKS Panels (PromQL)

### Cluster CPU & Memory

```promql
# CPU requests vs usage
sum(kube_pod_container_resource_requests{resource="cpu"})                       # requested
sum(rate(container_cpu_usage_seconds_total[5m]))                                # actual
sum(kube_pod_container_resource_limits{resource="cpu"})                         # limit

# Memory
sum(container_memory_working_set_bytes)
sum(kube_pod_container_resource_requests{resource="memory"})
```

### Pod Health

```promql
# Pods by phase
count(kube_pod_status_phase{phase="Running"}) by (namespace)
kube_pod_status_phase{phase="Failed"}
sum(increase(kube_pod_container_status_restarts_total[1h])) by (namespace)
```

### SLO Availability

```promql
# 99.9% availability target (based on a probe/up metric)
(1 - sum(rate(prometheus_http_requests_total{code=~"5.."}[30d])) /
       sum(rate(prometheus_http_requests_total[30d]))) * 100
```

---

## 🛠️ Step 5: Alerts & Annotations in Grafana

### Alert Rule

```yaml
# Via UI: Alerting → Alert rules → New
# Example rule: HighNodeCPU
condition: avg by (instance) (rate(node_cpu_seconds_total{mode!="idle"}[5m])) > 0.85
for: 10m
labels: { severity: critical }
notifications: Slack / SNS / PagerDuty
```

### Deployment Annotation

Add deploy events to a "Time series" panel so dashboards show releases:

```bash
# Enable annotations for deployments, or use a webhook
curl -X POST -H "Content-Type: application/json" \
  -d '{"text":"deployed v1.2.3"}' \
  http://admin:ChangeMe!@localhost:3000/api/annotations?time=... 
```

---

## 📝 Interview Questions

1. **How do you connect Grafana to AWS data?**
   - **Answer:** Add a CloudWatch data source; for authentication prefer IRSA on the Grafana pod (ec2_iam) rather than static keys.

2. **What's the best EKS dashboard setup?**
   - **Answer:** A cluster overview (315), a workload/deployment view by namespace, a node view, plus custom SLO dashboards — all provisioned as code for reproducibility.

3. **How do you build a dashboard for SLO tracking?**
   - **Answer:** Define error budget as a PromQL alert/query, track availability and latency over a rolling window (e.g., 30d), and annotate deploy events to correlate.

4. **How do you keep dashboards consistent across teams?**
   - **Answer:** Use dashboards-as-code (provisioning JSON in git), shared libraries for common panels, and templated variables (namespace/app).

5. **Grafana vs CloudWatch Dashboards?**
   - **Answer:** Grafana is a single, flexible front-end for Prometheus + CloudWatch + logs + traces; CloudWatch dashboards are AWS-native but more limited for cross-service PromQL.

---

## 📚 Resources for Review

- [Day 41: Prometheus Operator](Day-41_Prometheus_Operator/README.md)
- [Day 45: Monitoring Stack Practice](Day-45_Monitoring_Stack_Practice/README.md)

### External
- [Grafana](https://grafana.com/docs/)
- [Grafana CloudWatch data source](https://grafana.com/docs/grafana/latest/datasources/aws-cloudwatch/)

---

## 🎯 Today's Assignment

1. **Add** Prometheus + CloudWatch data sources
2. **Import** a cluster + a workload dashboard
3. **Create** an SLO/availability dashboard with 3 panels
4. **Build** an alert rule and test it
5. **Export** a dashboard to JSON and commit it as code

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

**Day 43:** AWS X-Ray Tracing

---

**Remember:** A dashboard is only useful if someone reads it. Keep it focused on answers, not data noise! 🚀
