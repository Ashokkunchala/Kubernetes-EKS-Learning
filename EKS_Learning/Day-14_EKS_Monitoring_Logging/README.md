# 📅 Day 14: EKS Monitoring and Logging

## 🎯 Learning Objectives
By the end of today, you will understand:
- CloudWatch Container Insights
- Prometheus and Grafana setup
- AWS X-Ray for tracing
- Fluent Bit for logging
- Hands-on monitoring setup

---

## 🧠 Plain-English terms before you start

| Term | Meaning | Why it matters |
|---|---|---|
| **Metric** | A numeric measurement recorded over time, such as CPU usage or request count. | Metrics help answer “how much?” and reveal trends. |
| **Log** | A timestamped record of an event or message. | Logs help explain what an application or component reported. |
| **Trace** | A connected view of one request moving across services. | Traces help locate latency or failure across multiple components. |
| **Prometheus** | A monitoring system that collects and queries time-series metrics. | It can power dashboards and alert rules for Kubernetes workloads. |
| **CloudWatch** | AWS's monitoring service for metrics, logs, alarms, and related telemetry. | It integrates with AWS services and can collect EKS workload signals. |
| **Alert** | A rule that notifies someone when a meaningful condition occurs. | Useful alerts point to user impact or an actionable risk, not just noise. |

**Analogy:** Metrics are the dashboard gauges, logs are the event diary, and traces are the route map for one request.

**Try / observe:** `kubectl logs <pod-name>`, `kubectl describe pod <pod-name>`, and inspect the configured Prometheus/CloudWatch dashboards and alert history.

**Troubleshooting clue:** If a dashboard is blank, check whether the collector is running, the target is being scraped, labels/filters match, and the time range is correct.

**Explain it back:** Which question is best answered by metrics, logs, and traces respectively?

---

## 📖 Core Concepts

### Monitoring Architecture

```
┌─────────────────────────────────────────────────────────┐
│              EKS MONITORING STACK                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Metrics Collection                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Prometheus                               │   │   │
│  │  │  - Cluster metrics                       │   │   │
│  │  │  - Application metrics                   │   │   │
│  │  │  - Custom metrics                        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  CloudWatch Container Insights            │   │   │
│  │  │  - Node metrics                          │   │   │
│  │  │  - Pod metrics                           │   │   │
│  │  │  - Network metrics                       │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Visualization                        │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Grafana                                 │   │   │
│  │  │  - Custom dashboards                    │   │   │
│  │  │  - Real-time graphs                     │   │   │
│  │  │  - Alerting rules                        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  CloudWatch Dashboard                    │   │   │
│  │  │  - Pre-built dashboards                 │   │   │
│  │  │  - AWS integration                      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Logging                              │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Fluent Bit                               │   │   │
│  │  │  - Container logs                        │   │   │
│  │  │  - Application logs                      │   │   │
│  │  │  - Audit logs                            │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  CloudWatch Logs                         │   │   │
│  │  │  - Log storage                           │   │   │
│  │  │  - Log queries                           │   │   │
│  │  │  - Alarms                                │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Alerting                            │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  Prometheus Alertmanager                 │   │   │
│  │  │  - Slack notifications                  │   │   │
│  │  │  - PagerDuty integration                │   │   │
│  │  │  - Email alerts                         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### CloudWatch Container Insights

```
┌─────────────────────────────────────────────────────────┐
│           CLOUDWATCH CONTAINER INSIGHTS                   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  What it Monitors:                               │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • CPU utilization                      │   │   │
│  │  │  • Memory utilization                   │   │   │
│  │  │  • Disk usage                           │   │   │
│  │  │  • Network I/O                          │   │   │
│  │  │  • Pod count                            │   │   │
│  │  │  • Container restarts                   │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Components:                                     │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  • CloudWatch Agent                     │   │   │
│  │  │  • Fluent Bit                            │   │   │
│  │  │  • OpenTelemetry Collector              │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Step-by-Step Setup

### Step 1: Install Container Insights

```bash
# Install Container Insights
eksctl create addon \
  --name amazon-cloudwatch-observability \
  --cluster my-eks-cluster \
  --region us-west-2 \
  --force
```

### Step 2: View Metrics

```bash
# View cluster metrics
aws cloudwatch get-metric-statistics \
  --namespace ContainerInsights \
  --metric-name cpu_utilization \
  --dimensions Name=ClusterName,Value=my-eks-cluster \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average
```

### Step 3: Install Prometheus Stack

```bash
# Add Helm repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus stack (includes Grafana)
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123
```

### Step 4: Access Grafana

```bash
# Get Grafana password
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d

# Port forward to Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Access at http://localhost:3000
```

---

## 📝 Practice Exercises

### Exercise 1: Set Up Monitoring Stack

```bash
# Install Container Insights
eksctl create addon \
  --name amazon-cloudwatch-observability \
  --cluster my-cluster \
  --force

# Verify
kubectl get pods -n amazon-cloudwatch
```

### Exercise 2: Install Prometheus and Grafana

```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Install Prometheus stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --set grafana.adminPassword=admin123

# Check installation
kubectl get pods -n monitoring
```

### Exercise 3: Create Custom Dashboard

```yaml
# custom-dashboard.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  my-dashboard.json: |
    {
      "dashboard": {
        "title": "My EKS Dashboard",
        "panels": [
          {
            "title": "CPU Usage",
            "type": "graph",
            "targets": [
              {
                "expr": "rate(container_cpu_usage_seconds_total[5m])",
                "legendFormat": "{{pod}}"
              }
            ]
          }
        ]
      }
    }
```

---

## 📝 Interview Questions

1. **What is CloudWatch Container Insights?**
   - **Answer:** It's a monitoring solution for containers that provides CPU, memory, disk, and network metrics for EKS clusters.

2. **How does Prometheus work with Kubernetes?**
   - **Answer:** Prometheus scrapes metrics endpoints from pods and nodes. It stores time-series data and provides querying with PromQL.

3. **What is the difference between Prometheus and CloudWatch?**
   - **Answer:** Prometheus for real-time metrics and alerting. CloudWatch for AWS service monitoring and log aggregation.

4. **How to set up custom dashboards?**
   - **Answer:** Use Grafana with Prometheus data source. Create dashboards with PromQL queries.

5. **What is Fluent Bit and why use it?**
   - **Answer:** Fluent Bit is a lightweight log processor that collects container logs and forwards them to CloudWatch Logs or other destinations.

---

## 📚 Resources

- [Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html)
- [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator)

---

## 🎯 Today's Assignment

1. **Install:** Container Insights on your cluster
2. **Set up:** Prometheus and Grafana
3. **Create:** A custom Grafana dashboard
4. **Configure:** Alerts for high CPU/memory

---

## 📊 Progress Tracker

```
Day 13: EKS Load Balancing with ALB ✅
Day 14: EKS Monitoring and Logging ⬜
```

---

## 🔗 Next Day Preview

**Day 15:** EKS Security Best Practices
- Pod Security Standards
- Network Policies
- Secret encryption
- Image scanning

---

**Remember:** Monitoring is essential for production clusters! 🚀
