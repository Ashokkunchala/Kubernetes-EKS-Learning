# 📅 Day 58: Failover Procedures

## 🎯 Learning Objectives
By the end of today, you will understand:
- Failover vs failback concepts
- Failover strategies: active/passive, active/active
- Orchestrating EKS + data + traffic failover
- Automated vs manual failover and runbooks
- Testing failover safely

---

## 📖 Core Concepts

### Failover Strategies

```
┌───────────────────────────────────────────────────────────────┐
│                FAILOVER STRATEGIES                            │
│                                                               │
│  ACTIVE / PASSIVE                                             │
│  ┌─────────┐          ┌─────────┐                             │
│  │ Region A│  TAKES   │ Region B│    B is warm standby        │
│  │ (hold), │  OVER    │ (standby)│   data replicated          │
│  │ traffic ──────────▶│  promote│                            │
│  └─────────┘          └─────────┘                             │
│                                                               │
│  ACTIVE / ACTIVE                                              │
│  ┌─────────┐          ┌─────────┐                             │
│  │ Region A│◀────────▶│ Region B│    both serve; on failure   │
│  │ traffic │  shed    │ traffic │    remaining absorbs all    │
│  └─────────┘          └─────────┘                             │
│                                                               │
│  GitOps / IaC lets you (re)create the cluster & apps in B.    │
└───────────────────────────────────────────────────────────────┘
```

### Failover Layers

| Layer | What happens |
|-------|--------------|
| Traffic | Global Accelerator / Route53 health-based reroute |
| Cluster | ArgoCD syncs apps to standby cluster (or standby pre-synced) |
| Data | Promote replica/global DB; restore replicated volumes |
| DNS/Ingress | Cut over global endpoint to new region |

---

## 🛠️ Step 1: Design the DR State

`dr-config` — declarative, region-parameterized:

```yaml
# appset - deploy identical app to primary and standby
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: app
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - name: primary
        region: us-east-1
        active: "true"
      - name: standby
        region: us-west-2
        active: "false"
  template:
    metadata:
      name: '{{.name}}-app'
    spec:
      project: default
      source:
        repoURL: https://github.com/yourorg/gitops.git
        path: 'overlays/{{.region}}'
        targetRevision: main
      destination:
        server: '{{.name}}'
        namespace: app
```

Use a **config flag** to decide which region is "active" (e.g., ingress DNS points there).

---

## 🛠️ Step 2: Automate Traffic Failover

### With Route53 (health check based)

```bash
# Health check on primary ALB
aws route53 create-health-check --caller-reference hc1 \
  --health-check-config '{"Type":"HTTP","ResourcePath":"/health","IPAddress":"<alb-domain>","Port":443}'

# Associate health check with a failover record (PRIMARY/SECONDARY)
aws route53 change-resource-record-sets --hosted-zone-id ZONE \
  --change-batch file://failover-record.json
```

```json
{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "app.example.com.",
      "Type": "A",
      "SetIdentifier": "primary",
      "Failover": "PRIMARY",
      "HealthCheckId": "hc1",
      "AliasTarget": { "HostedZoneId": "ZALB", "DNSName": "<primary-alb>" }
    }
  }]
}
```

> Route53 automatically switches to the SECONDARY (standby ALB) when the primary health check fails.

### With Global Accelerator

```bash
# GA endpoint groups per region; when primary endpoint unhealthy, GA shifts traffic
aws globalaccelerator create-endpoint-group \
  --listener-arn <listener> --endpoint-group-region us-east-1 \
  --endpoint-configurations '[{"EndpointId":"<primary-alb-arn>","Weight":128}]'
```

---

## 🛠️ Step 3: Promote Data in the DR Region

**Option A — RDS read replica promote (compute, minutes):**

```bash
# Promote west read replica to primary
aws rds promote-read-replica \
  --db-instance-identifier app-replica-west \
  --region us-west-2
```

**Option B — Aurora Global DB (RPO≈0, faster):**

```bash
# Fail over the primary cluster region
aws rds failover-global-cluster \
  --global-cluster-identifier aurora-global \
  --target-db-cluster-identifier arn:aws:rds:us-west-2:ACCOUNT:cluster:app-west
```

**Option C — Restore from backups (higher RPO):**

```bash
# Restore EBS-backed PVC from Velero / snapshot copy (Day 56)
velero restore create --from-backup prod-full --namespace-mappings production:production
```

---

## 🛠️ Step 4: Execute the Failover (Runbook)

**Automated decision** (health-check driven) for traffic; **runbook** for data:

```
1. DETECT: alert/monitor confirms primary region/cluster degraded.
2. DECIDE: is it a region failure or just a service? (DR only for region.)
3. TRAFFIC: Route53/GA health check already reroutes to standby (auto).
4. DATA: promote standby DB (replica promote / Aurora global failover).
5. APPS: standby already synced via ArgoCD; verify rollout healthy.
6. VALIDATE: smoke test via global endpoint; check metrics.
7. DECLARE: update status in incident tool; notify stakeholders.
```

---

## 🛠️ Step 5: Failback (when primary recovers)

```
1. Re-provision / fix primary region (GitOps sync, data replays).
2. Re-sync data from standby → primary (careful with direction).
3. Switch traffic back to primary (revert Route53/GA weights).
4. Monitor for stability before fully reverting.
```

> Always **failback during a low-traffic window** and keep failover path rehearsed.

---

## 🛠️ Testing Failover Safely (Game Day)

```bash
# Reduce primary ALB healthy weight in Global Accelerator to 0 without deleting
aws globalaccelerator update-endpoint-weights \
  --endpoint-configurations '[{"EndpointId":"<primary-alb>","Weight":0}]'
# Observe traffic shift to standby; run smoke tests; then restore weight
```

**Best practice:** run regular DR drills in a staging replica of prod to validate RPO/RTO without impacting customers.

---

## 📝 Interview Questions

1. **What is failover and how is it different from backup?**
   - **Answer:** Failover is switching to a standby/redundant system with minimal downtime; backup is restoring from stored snapshots (often higher RTO). Failover keeps services up; backup recovers data.

2. **Active/active vs active/passive?**
   - **Answer:** Active/active serves traffic from all regions and absorbs failure by shedding to remaining; active/passive keeps a warm standby promoted on failure. Active/active has better RTO but needs conflict-free multi-writer data.

3. **How do you automate EKS failover?**
   - **Answer:** Route53/Global Accelerator health checks reroute traffic; ArgoCD keeps standby apps synced; data promotion is scripted (RDS promote / Aurora global failover); runbooks + alarms handle decision and validation.

4. **What's a failback and why is it risky?**
   - **Answer:** Returning to the original region after recovery; risky due to data divergence and split-brain if both regions wrote. Do it in a controlled window with replication restored first.

5. **How do you test failover without disrupting production?**
   - **Answer:** Game Days on a staging replica, GA weight shifting in a maintenance window, and canary traffic tests — all with rollback ready.

---

## 📚 Resources for Review

- [Day 56: Backup](Day-56_Cluster_Backup_Strategies/README.md)
- [Day 57: Cross-Region Replication](Day-57_Cross_Region_Replication/README.md)
- [Day 59: DR Practice](Day-59_Disaster_Recovery_Practice/README.md)

---

## 🎯 Today's Assignment

1. **Define** RPO/RTO and pick a failover strategy
2. **Set up** Route53/GA health-check failover
3. **Write** a failover runbook (detect → promote data → validate → failback)
4. **Promote** a replica/global DB in a test
5. **Run** a failover Game Day and record results

---

## 📊 Progress Tracker

```
Week 19-20 Disaster Recovery:
Day 56: Cluster Backup Strategies ⬜
Day 57: Cross-Region Replication ⬜
Day 58: Failover Procedures ⬜
Day 59: Disaster Recovery Practice ⬜
Day 60: Final Review & Certification ⬜
```

---

## 🔗 Next Week Preview

**Day 59:** Disaster Recovery Practice

---

**Remember:** Failover you've never practiced will fail you in production. Drill it! 🚀
