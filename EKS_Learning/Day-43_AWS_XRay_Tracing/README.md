# 📅 Day 43: AWS X-Ray Tracing

## 🎯 Learning Objectives
By the end of today, you will understand:
- Distributed tracing concepts (spans, traces, service maps)
- AWS X-Ray SDK and the ADOT collector for EKS
- Instrumenting apps and sending traces from EKS
- Analyzing traces, service maps, and detecting latency
- Observability (Metrics + Logs + Traces) integration

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Trace** | A connected record of one request's path across services. |
| **Span** | One timed operation within a trace, such as a database call. |
| **Latency** | The time a request or operation takes to complete. |
| **Sampling** | Choosing which requests are recorded for tracing to control overhead and cost. |
| **Service map** | A visual view of services and their observed dependencies. |
| **Correlation ID** | An identifier carried across components to connect logs/events for one request. |

**Analogy:** A trace is the full journey of a parcel; each span is one checkpoint, and latency is the time spent at each checkpoint.

**Try / observe:** Follow a single test request and compare its trace, application logs, and metrics using shared IDs/timestamps where available.

**Troubleshooting clue:** Missing traces may mean instrumentation, sampling, permissions, exporter configuration, or network delivery is wrong—not necessarily that the application received no requests.

**Explain it back:** Why can tracing find a slow downstream call that a CPU dashboard cannot explain?

---

## 📖 Core Concepts

### Tracing Anatomy

```
┌───────────────────────────────────────────────────────────────┐
│                  DISTRIBUTED TRACE                             │
│                                                               │
│                     ┌─────────────┐                           │
│        ┌───────────▶│  Frontend   │────────────────────┐      │
│        │  span      └──────┬──────┘  span              │      │
│        │                   │                           │      │
│   ┌────▼────┐     ┌────────▼────────┐      ┌───────────▼───┐  │
│   │ Auth    │     │   Order Svc     │      │  Payments     │  │
│   │ span    │     │  span + sub     │      │  span → SQS   │  │
│   └─────────┘     └────────┬────────┘      └───────────────┘  │
│                            │                                   │
│                     ┌──────▼──────┐                           │
│                     │  MySQL/RDS  │                           │
│                     │  (db span)  │                           │
│                     └─────────────┘                           │
│                                                               │
│  Each segment/span has:                                       │
│   · name, start/end time, duration                            │
│   · annotations (key/value) & metadata                        │
│   · trace id + parent spans (tree structure)                  │
└───────────────────────────────────────────────────────────────┘
```

### Why Tracing on EKS

- Microservices make latency attribution hard — tracing links **requests across pods/services**.
- X-Ray Service Map visualizes dependencies\& hot spots.
- Combined with metrics/logs you get full **root-cause** visibility.

---

## 🛠️ Architecture: ADOT Collector

```
┌───────────────────────────────────────────────────────────────┐
│   APP PODS (instrumented with X-Ray SDK)                      │
│   ┌─────────┐ ┌─────────┐ ┌─────────┐                         │
│   │  Svc A  │ │  Svc B  │ │  Svc C  │  X-Ray SDK udp:2000     │
│   └────┬────┘ └────┬────┘ └────┬────┘ ─────────────────┐      │
│        │           │           │                       │      │
│        └───────────┼───────────┘                       ▼      │
│                    │                          ┌──────────────┐ │
│                    └─────────────────────────▶│    ADOT     │ │
│                                              │  Collector   │ │
│                                              │ (otel/xray)  │ │
│                                              └──────┬───────┘ │
│                                                     ▼         │
│                                           ┌──────────────────┐│
│                                           │   AWS X-Ray     ││
│                                           │  (service:APIs) ││
│                                           └──────────────────┘│
└───────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Step 1: Deploy ADOT Collector

```bash
# Add AWS Observability Helm repo
helm repo add aws-observability https://aws-observability.github.io/helm-charts
helm repo update

kubectl create ns observability

# Deploy collector in deployment mode
helm install adot-collector aws-observability/aws-observability-collector \
  --namespace observability \
  --set nameOverride=aws-otel \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-otel \
  --set image.repository=public.ecr.aws/aws-observability/aws-otel-collector
```

```bash
# Give the collector IRSA permission to send to X-Ray
eksctl create iamserviceaccount \
  --cluster my-eks-cluster \
  --namespace observability \
  --name aws-otel \
  --attach-policy-arn arn:aws:iam::aws:policy/AWSXrayWriteOnlyAccess \
  --approve
```

---

## 🛠️ Step 2: Simple OTEL Collector Config

```yaml
# adot-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-otel-collector
  namespace: observability
data:
  collector.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      batch: {}
    exporters:
      awsxray:
        region: us-east-1
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [awsxray]
```

```bash
kubectl apply -f adot-config.yaml
kubectl -n observability rollout restart deployment aws-otel-collector
```

---

## 🛠️ Step 3: Instrument an App (Python + X-Ray SDK)

```python
# app.py - using the X-Ray SDK
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

patch_all()  # patches http, boto3, sqlalchemy, etc.
xray_recorder.configure(service='orders-api')

from flask import Flask
app = Flask(__name__)

@app.route('/orders')
def orders():
    # segmented and traced automatically (WSGI middleware)
    with xray_recorder.capture('query-db'):
        rows = db_query()   # DB call is auto-patched
    return str(rows)

# Send to local collector
# X-Ray SDK defaults to UDP 127.0.0.1:2000; point it at ADOT:
#   export AWS_XRAY_DAEMON_ADDRESS=<adot-svc>:2000
```

Dockerfile hint:

```dockerfile
ENV AWS_XRAY_DAEMON_ADDRESS=aws-otel-collector.observability.svc:2000
```

### Go with OTEL SDK (alternative)

```go
// spin up OTLP exporter to the collector
// set OTEL_EXPORTER_OTLP_ENDPOINT=http://aws-otel-collector.observability.svc:4318
ctx, span := tracer.Start(ctx, "getOrders")
defer span.End()
```

---

## 🛠️ Step 4: View & Analyze

```bash
# Open X-Ray console (Click into a trace)
# Regions: us-east-1 → CloudWatch → X-Ray traces

# Look for:
#  · Service map: detect slow/erroring dependencies
#  · Trace list: find slow requests
#  · Trace timeline: drill into which span is slow
```

```bash
# Query recent traces via API
aws xray get-traces \
  --trace-ids "$(aws xray get-trace-summaries --start-time "$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)" --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --query 'TraceSummaries[*].Id' --output text)"
```

---

## 🛠️ Step 5: Traces + Metrics + Logs = Full Observability

Link correlated telemetry with a shared trace ID convention:

```yaml
# Log with the trace id so you can jump from a trace to logs
import logging
from aws_xray_sdk.core.models.segment import Segment

trace_id = xray_recorder.current_segment().trace_id
logger.info("processing order", extra={'trace_id': trace_id})
```

In practice, pair:
- **CloudWatch Logs** (structured logs with trace IDs)
- **X-Ray** (traces / latency)
- **Prometheus** (metrics / SLOs)

---

## 📝 Interview Questions

1. **What is distributed tracing and why do you need it on EKS?**
   - **Answer:** It follows a request across multiple services to attribute latency and errors, essential in a microservice/mesh environment where a request spans many pods.

2. **How does ADOT work with X-Ray?**
   - **Answer:** Apps emit OpenTelemetry traces to the ADOT collector (receiver), which batches and exports them to X-Ray (awsxray exporter) — keeping the SDK decoupled from the backend.

3. **What's the difference between a trace, a segment, and a span?**
   - **Answer:** A trace is the end-to-end request; a segment is one service's portion; a span is one unit of work within a segment (e.g., a DB call or an HTTP call).

4. **How do you reduce tracing overhead?**
   - **Answer:** Sampling (e.g., 10% by default), batching in the collector, and only instrumenting critical services/paths.

5. **How do you find the root cause of a slow request with X-Ray?**
   - **Answer:** Open the trace timeline, sort by duration, drill into the slowest span, then jump to related logs/metrics using the trace ID.

---

## 📚 Resources for Review

- [Day 42: Grafana Dashboards](Day-42_Grafana_Dashboards/README.md)
- [Day 41: Prometheus Operator](Day-41_Prometheus_Operator/README.md)

### External
- [AWS X-Ray](https://docs.aws.amazon.com/xray/)
- [ADOT Collector](https://aws-otel.github.io/)

---

## 🎯 Today's Assignment

1. **Deploy** the ADOT collector with X-Ray IRSA
2. **Instrument** a small app with the X-Ray SDK
3. **Send** 50+ requests and view the service map
4. **Find** one slow span and explain the cause
5. **Configure** sampling to limit overhead

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

**Day 44:** Custom Metrics with CloudWatch

---

**Remember:** Metrics tell you SOMETHING is broken; traces tell you WHERE. Use both! 🚀
