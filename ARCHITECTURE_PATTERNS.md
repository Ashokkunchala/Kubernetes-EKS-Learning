# 🏗️ EKS Architecture Patterns — Production Blueprints

> Reusable, battle-tested design patterns for running workloads on EKS.
> Each pattern shows the diagram, the resources it needs, and where the 60-day
> plan teaches it.

---

## 1️⃣ Stateless Web Tier (SMOK)

**Use when:** frontends, APIs, workers that keep no state in the pod.

```
Internet → Route53 → ALB (ALB Ingress Controller)
                          │
                    ┌─────▼─────┐
                    │  Ingress  │  host/path routing
                    └─────┬─────┘
                    ┌─────▼─────┐
                    │  Service  │  ClusterIP DNS: app.ns.svc
                    └─────┬─────┘
                    ┌─────▼─────┐  Deployment
                    │  Pods × N │  HPA(P-CPU) + PDB(min N-1)
                    └───────────┘
```

**Manifests:** [`manifests/app/`](EKS_Learning/manifests/app/) · **Labs:**
[Lab-01](EKS_Learning/labs/Lab-01_Deploy_Scale_Update.md) · **Days:**
[Day-01–03](EKS_Learning/Day-01_Kubernetes_Basics/README.md)

**Checklist:**
- [ ] Deployment with `requests` + `limits` and liveness/readiness probes
- [ ] Service + Ingress (ALB, `TargetType: ip`)
- [ ] HPA (CPU util 60–70%) + PDB
- [ ] Pod anti-affinity to spread across AZs
- [ ] NetworkPolicy: allow only ingress from ALB CIDR

---

## 2️⃣ 12-Factor Stateless + Managed Data

**Use when:** microservices that talk to RDS / ElastiCache / S3 (never store
data in the pod).

```
ALB → svc-a ─┬→ RDS (Postgres, Multi-AZ)
             ├→ ElastiCache (Redis)
             └→ S3 (objects)
  svc-a / svc-b / svc-c   ← go, node, java; ECR images, IRSA per service
```

**Key points:**
- Secrets via EKS Secrets Manager / External Secrets, not plain Secrets
- Each service gets **its own IAM role** via IRSA (least privilege)
- Data layers live *outside* the cluster → zero state, chaos-friendly

**Days:** [Day-11](EKS_Learning/Day-11_EKS_IAM_Roles_Policies/README.md),
[Day-26–30](EKS_Learning/Day-26_OIDC_Provider_Setup/README.md) · **Project:**
[Project 3](EKS_PROJECTS.md)

---

## 3️⃣ Stateful App on EKS (EBS)

**Use when:** databases/persistent message queues that must keep data (~25ms
latency, ReadWriteOnce).

```
StatefulSet (name-0 … name-2)
      │
      ▼
 StatefulSet → PVC name-0 → EBS gp3 volume (attach in same AZ as pod AZ)
```

**Critical rules:**
- EBS is **AZ-bound** → pod must schedule in the volume's AZ (use node
  affinity/`topologyKey` or `WaitForFirstConsumer`)
- Node drain → volume detaches → pod reschedules (may be brief downtime)
- Use **EBS CSI driver** snapshot SC + Velero for backups

**Manifests:** [`manifests/storage/`](EKS_Learning/manifests/storage/) ·
**Day:** [Day-12](EKS_Learning/Day-12_EKS_Storage_EBS/README.md),
[Day-23](EKS_Learning/Day-23_Volume_Snapshots_Restore/README.md)

---

## 4️⃣ Shared Read-Many Data (EFS)

**Use when:** multiple pods must read/write the same data (shared uploads,
ML datasets, content).

```
             ┌──────────────┐
 svc-a ─────►│  EFS (RMW)   │◄───── svc-b
             └──────────────┘
        efs.csi.aws.com + efs-sc StorageClass
```

**Watch out:** EFS is NFS → higher latency, throughput mode matters
(Bursting vs Provisioned vs Elastic). Not for high-IOPS transactional DBs.

**Manifests:** [`efs-pvc.yaml`](EKS_Learning/manifests/storage/efs-pvc.yaml) ·
**Day:** [Day-21](EKS_Learning/Day-21_EFS_CSI_Driver/README.md)

---

## 5️⃣ Microservice Mesh (Istio)

**Use when:** need mTLS everywhere, fine-grained traffic shifting, or detailed
`REQUESTS/s, P99` per service.

```
        Istio IngressGateway (istio-system)
                     │
            VirtualService (routes)
                     │
   svc-a ◄──mTLS──► svc-b ◄──mTLS──► svc-c
     │               │               │
  Sidecar Envoy  Sidecar Envoy   Sidecar Envoy
```

**Patterns:** DestinationRule subsets → canary weight `20/80`; PeerAuthentication
strict mTLS; AuthorizationPolicy per service.

**Manifests:** [`manifests/networking/istio-*`](EKS_Learning/manifests/networking/) ·
**Day:** [Day-18](EKS_Learning/Day-18_Service_Mesh_Istio/README.md)

---

## 6️⃣ GitOps Delivery (Argo CD)

**Use when:** you want everything (apps, configs, cluster add-ons) declared in
Git, with PR-driven change.

```
GitHub (apps repo + infra repo)
        │  watch/compare
        ▼
   Argo CD  ──sync──►  EKS cluster
        │        ┌─────────────┐
        │        │ env staging │
        └────────│ env prod    │  overlays per env
                 └─────────────┘
```

**Manifests:** [`manifests/gitops/`](EKS_Learning/manifests/gitops/) ·
**Day:** [Day-31–35](EKS_Learning/Day-31_GitOps_ArgoCD_Intro/README.md)

---

## 7️⃣ Observability Stack

**Use when:** you must know CPU/mem, latency/error rates, and logs for every
service.

```
 Prometheus ──scrape──► kubecore/pods/nodes + custom /metrics
     │                       │
     ▼                       ▼
 Alertmanager ──► Slack   Grafana ──► dashboards
 Fluent Bit ──► CloudWatch Logs
 X-Ray ──► traces (X-Ray Daemon per node)
```

**Manifests:** [`manifests/monitoring/`](EKS_Learning/manifests/monitoring/) ·
**Days:** [Day-14](EKS_Learning/Day-14_EKS_Monitoring_Logging/README.md),
[Day-41–45](EKS_Learning/Day-41_Prometheus_Operator/README.md)

---

## 8️⃣ Multi-Region Active/Passive DR

**Use when:** RTO < 1h for a region outage.

```
 us-west-2 (active)             us-east-1 (standby)
 ┌─────────────┐                ┌─────────────┐
 │ EKS prod    │  Velero  ─────►│ EKS standby │   scale-up on failover
 │ RDS MultiAZ │  RDS────read──►│ RDS replica │→ promote
 └─────────────┘   ECR/S3 CRR   └─────────────┘
 Route53 failover policy flips DNS to DR ALB
```

**Manifests:** [`manifests/dr/`](EKS_Learning/manifests/dr/) ·
**Labs:** [DR Game Day](EKS_Learning/Day-59_Disaster_Recovery_Practice/README.md) ·
**Days:** [Day-56–60](EKS_Learning/Day-56_Cluster_Backup_Strategies/README.md)

---

## 9️⃣ Cost-Efficient Burst

**Use when:** batch, dev, non-critical tenants must get big capacity cheaply.

```
 On-Demand pool  → PDB-protected critical apps
 Spot pool       → batch / stateless / dev  (maxSize caps spend)
 Karpenter       → launches right-size node per pending pod, ttl clean-up
```

**Manifests:** [`manifests/cost/`](EKS_Learning/manifests/cost/) ·
**Day:** [Day-51–53](EKS_Learning/Day-51_Spot_Instances_EKS/README.md)

---

## 🔀 Pattern Decision Tree

```
Need state in pod?
 ├─ No  → Pattern 1 / 2
 └─ Yes → single-writer? ──► Pattern 3 (EBS)
            │
            └─ multi-reader? ──► Pattern 4 (EFS)

Need mTLS / traffic shifting? → Pattern 5 (Istio)
Want Git-driven delivery?      → Pattern 6 (Argo CD)
Need to know everything?       → Pattern 7 (observability)
Region fails and we survive?   → Pattern 8 (DR)
Budget tight?                  → Pattern 9 (cost burst)
```

---

## 🧠 Also See

- [`EKS_PROJECTS.md`](EKS_PROJECTS.md) — 6 complete projects built on these patterns
- [`GLOSSARY.md`](GLOSSARY.md) — terms
- [`manifests/`](EKS_Learning/manifests/README.md) — ready-made YAML per pattern

*Architecture patterns last updated: September 2026*