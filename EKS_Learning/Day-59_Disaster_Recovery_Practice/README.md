# 📅 Day 59: Disaster Recovery Practice (Game Day Drill)

## 🎯 Learning Objectives

By the end of today, you will be able to:
- Run a complete, production-realistic **DR Game Day** without breaking a cluster
- Prove backups work by **restoring** into a sandbox and validating data
- **Replicate** images, state, and data to a DR region end to end
- Execute an automated **failover** and measure RTO/RPO against targets
- Run a controlled **failback** and capture a post-incident review
- Produce runbooks and a DR scorecard your team can actually use

> **Rule of DR:** *A backup you have never restored is a wish, not a plan.*
> Today you stop wishing and start proving.

Prerequisites: Everything from **Day 56** (backup), **Day 57** (cross-region replication), and **Day 58** (failover procedures).

---
## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Game day** | A planned exercise that simulates a failure to test people, tools, and recovery steps. |
| **Recovery objective** | The agreed target for recovery time (RTO) and acceptable data loss (RPO). |
| **Dependency** | Another system or service required for the application to work, such as DNS, identity, database, or registry. |
| **Restore test** | Proving a backup can recreate usable service, not just that a backup job succeeded. |
| **Evidence** | Logs, timestamps, commands, outputs, and measurements showing what happened. |
| **Postmortem** | A learning-focused review of an incident/exercise, including causes and follow-up actions. |

**Analogy:** A fire drill is useful only if people practise the exit and verify they can reach safety; a game day tests the real recovery path.

**Try / observe:** Record start time, detection time, recovery time, data loss, blockers, and follow-up owners. Use a sandbox and define a stop condition before injecting failure.

**Troubleshooting clue:** If recovery misses the target, separate technology gaps (missing backup/capacity) from process gaps (unclear owner, stale runbook, untested credentials).

**Explain it back:** What evidence proves that a recovery exercise succeeded?

---


## 🖥️ Project: The 3-Hour DR Game Day

```
┌─────────────────────────────────────────────────────────────────┐
│                    DR GAME DAY ARCHITECTURE                       │
│                                                                  │
│  ┌──────────────────────────┐      ┌──────────────────────────┐  │
│  │   PRIMARY REGION (us-west-2) │      │   DR REGION (us-east-1)    │  │
│  │                          │      │                          │  │
│  │  EKS Cluster "prod"      │      │  EKS Cluster "dr"        │  │
│  │  ├─ ArgoCD (GitOps)      │      │  ├─ ArgoCD (GitOps)      │  │
│  │  ├─ app: ecommerce       │      │  └─ (apps DRY until      │  │
│  │  ├─ Velero backups ───────────────►    failover)           │  │
│  │  └─ EBS volumes          │      │                          │  │
│  │                          │      │  ┌────────────────────┐  │  │
│  │  RDS "primary"           │      │  │ RDS read replica  │  │  │
│  │  ECR repo                │      │  │ (promotes on FO)  │  │  │
│  │  S3 bucket ─────────────────────────► (CRR enabled)     │  │  │
│  │  Route53 alias ────────────────► DR ALB (failover)      │  │  │
│  └──────────────────────────┘      └──────────────────────────┘  │
│                                                                  │
│  Velero backup storage: s3://velero-backups (single bucket)       │
│  ECR registry replication: per-repo cross-region rule           │
│  Route53: failover record + health check on primary ALB           │
└─────────────────────────────────────────────────────────────────┘
```

### Why Game Days Beat Paper Plans

| Method | Downtime Risk | Real-World Value | Effort |
|--------|---------------|------------------|--------|
| Reading a DR doc | High | Low | Low |
| Testing a backup restores to a sandbox | Low | High | Medium |
| **Full Game Day drill** | Low (staging/clone) | **Very High** | High |
| Waiting for a real disaster | Certain | "Free" (you're already down) | None |

**Never test DR against your live production primary.** Practice on a clone
of the prod cluster or a staging replica, and only shift real traffic in a
maintenance window with an approved rollback plan.

---

## 📝 Step-by-Step Implementation

### Phase 0: Before You Start — Inventory

```bash
# 1. Confirm backup job is actually running (not just defined)
velero get schedules
velero get backups | tail -5
velero backup describe <latest-backup> --details

# 2. Confirm the backup is restorable FROM the DR region
#    (S3 bucket must be readable by a DR-region Velero install)
aws s3 ls s3://velero-backups/backups/ | tail -5

# 3. Confirm image replication is on for every app repo
aws ecr describe-registry \
  --region us-east-1 --registry-id <ACCOUNT> \
  --query 'replicationConfiguration.replicationRules'

# 4. Confirm DB replica exists and its lag is acceptable
aws rds describe-db-instances \
  --region us-east-1 \
  --query "DBInstances[?ReadReplicaSourceDBInstanceIdentifier!=null]
          .[DBInstanceIdentifier,ReplicaMode,StatusInfos[0].NormalizedLag]"

# 5. Confirm Route53 failover record + health check exist
aws route53 list-health-checks
```

**Gate:** If any inventory item is missing, fix it in Week 19 material
(Days 56-58) *before* running the drill. A failover that starts without
pieces is a failed drill.

---

### Phase 1: Sandbox Restore Test (RPO Proof)

Restore into a scratch cluster first. This proves **your backups work**.

```bash
# 1. Install Velero in the DR cluster pointing to the same bucket
velero install \
  --provider aws \
  --bucket velero-backups \
  --secret-file aws-s3-credentials.txt \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1

# 2. Restore ONLY the app namespace into the DR cluster (no traffic yet)
velero restore create --from-backup <latest-backup> \
  --include-namespaces ecommerce \
  --namespace-mappings ecommerce:ecommerce-restored

# 3. Validate the restore
velero restore describe <restore-name> --details

# 4. Verify data integrity inside the restored database PVC pod
kubectl exec -n ecommerce-restored deploy/order-service -- \
  curl -s http://order-db:5432  > /dev/null || true
kubectl exec -n ecommerce-restored deploy/product-service -- \
  wget -qO- http://product-db:5432 | head -3

kubectl logs -n ecommerce-restored $(kubectl get pods -n ecommerce-restored -o name | head -1) | tail -20

# 5. Record: RPO achieved = time of latest backup vs now
```

**What you just proved:** that a restore path exists, that volumes + app
configs come back, that the DR region can reach the backup bucket.

---

### Phase 2: Controlled Failover Drill (RTO Proof)

Run this in the **DR cluster**, shifting real traffic in a maintenance window.

```bash
# 1. Promote the database (write path now in DR region)
aws rds promote-read-replica \
  --db-instance-identifier ecommerce-dr \
  --region us-east-1

# 2. Point app secrets/env to the DR database endpoint
kubectl -n ecommerce set env deploy/product-service \
  DATABASE_URL=postgres://user:pass@ecommerce-dr.cluster-ro.us-east-1.rds.amazonaws.com:5432/ecommerce
kubectl -n ecommerce set env deploy/order-service \
  DATABASE_URL=postgres://user:pass@ecommerce-dr.cluster-ro.us-east-1.rds.amazonaws.com:5432/ecommerce

# 3. Scale the DR node group up to meet demand
eksctl scale nodegroup --cluster ecommerce-dr \
  --name dr-workers --nodes 3 --nodes-min 2 --nodes-max 6 --region us-east-1

# 4. Ensure ArgoCD has synced the DR overlay (image + configs)
kubectl -n argocd exec deploy/argocd-server -- \
  argocd app sync ecommerce --force
kubectl -n argocd exec deploy/argocd-server -- \
  argocd app wait ecommerce --health

# 5. Verify pods are healthy and serving BEFORE cutting traffic
kubectl get pods -n ecommerce -o wide
kubectl rollout status deploy/frontend -n ecommerce --timeout=180s
curl -s -o /dev/null -w "%{http_code}" http://<dr-alb-.amazonaws.com>/healthz
```

#### Cut traffic (Route53, 30 seconds to decide, minutes to switch)

```bash
# Health check already flipped? Then update the alias target.
aws route53 change-resource-record-sets \
  --hosted-zone-id ZXXXXXXXXXX \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.ecommerce.com",
        "Type": "A",
        "AliasTarget": {
          "DNSName": "dr-alb-1234567890.us-east-1.elb.amazonaws.com",
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# 6. Measure the RTO clock — start timer at decision, stop at healthy traffic
curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" https://app.ecommerce.com/healthz
```

**Record the numbers.**
| Metric | Target | Measured | Pass? |
|--------|--------|----------|-------|
| RPO (data loss) | ≤ 15 min | | |
| RTO (downtime) | ≤ 30 min | | |
| Data validation | 100% | | |

---

### Phase 3: Failback (Return to Primary)

> **Failback is a separate, riskier operation.** Plan it as its own window.

```bash
# 1. Stop writes or run during low traffic; freeze app
# 2. Flip replication direction (configure NEW replica back to primary region)
# 3. Final sync, then promote the PRIMARY again in the DR region
aws rds promote-read-replica --db-instance-identifier ecommerce-primary --region us-east-1

# 4. Point secrets back and redeploy
kubectl -n ecommerce set env deploy/product-service DATABASE_URL=<primary-endpoint>
kubectl -n ecommerce set env deploy/order-service DATABASE_URL=<primary-endpoint>

# 5. Flip DNS back to primary ALB
# 6. Scale DR nodes back down
eksctl scale nodegroup --cluster ecommerce-dr --name dr-workers \
  --nodes 1 --region us-east-1

# 7. Re-run full validation; then take a fresh full backup
velero backup create post-failback
```

**Failback checklist:**
- [ ] Replication direction restored to Primary → DR
- [ ] New backup taken after failback completes
- [ ] Old DR write endpoint retired / quiesced
- [ ] Monitoring + alarms point at correct region
- [ ] Team notified of the final cutover

---

### Phase 4: Chaos Drill (Why Simplicity Wins)

If your primary is healthy, your *real* failover interval is unknown. Run a
staged shutdown to see if the automation is what you think it is.

```bash
# Option A — stop the whole app (simulates "we lost everything")
kubectl scale deploy --all --replicas=0 -n ecommerce --context=prod

# Option B — kill the primary ALB (simulates region networking collapse)
#   (manually delete the ALB in console OR stop the load balancer target)

# Option C — make the Route53 health check fail on purpose
#   (set a maintenance page on the primary /healthz)
```

Then: **run Phases 1-2 as if you had no dashboard.** SCRUM the runbook
with a stopwatch and a colleague who refuses to touch the keyboard.

A drill where you "just did it before" is the most valuable kind — it teaches
muscle memory, not just checklists.

---

## 📝 RTO/RPO Runner Sheet (Copy-Paste Template)

Use this on the day of the drill. Fill in BEFORE hitting "go".

```
DRILL NAME       : ______________________
DATE             : ______________________
PARTICIPANTS     : (scribe / executor / decision-maker)
MAINTENANCE WINDOW: ________ - ________
ESCALATION PATH  : page-on-call @ _____ / Slack #oncall

TARGETS          : RTO = ___ min | RPO = ___ min
DECISION TRIGGER : health check fails for ___ consecutive minutes

TIMELINE
[ ] T-30  pre-check inventory (backup exists, replica synced < 60s lag)
[ ] T-0   decision made, timer starts
[ ] +2    promote RDS replica (DR)
[ ] +5    ArgoCD sync + rollout
[ ] +7    health validation (curl /healthz)
[ ] +9    Route53 cutover
[ ] +12   end-user validation (buy-flow smoke test)
[ ] RTO   = ________ minutes (measured)
[ ] RPO   = ________ minutes (data at time of promotion)

LEARNINGS
1. What went well:  ______________________________________
2. What broke/blocked: ___________________________________
3. New runbook fix needed: ________________________________
4. Owner + due date: ______________________________________
```

---

## 🔬 Practice Labs

### Lab 1: Prove a Backup Restores to a Fresh Cluster
- Create a throwaway `ecommerce-restore-test` namespace in the DR cluster.
- Restore the latest Velero backup into it.
- Validate a pod, an env var, and one row of DB data.
- Delete everything you created after validation (keep the cluster clean).

### Lab 2: Measure Real IP -> RTO
- Time Phase 2 with a stopwatch and a scripted `curl` loop.
- Try twice: once with ArgoCD sync ready, once "cold".
- Compare. Note where you lost minutes — that's your next optimization.

### Lab 3: Simulate the "Impossible" (Region Unreachable)
- `kubectl drain` all nodes in prod (or terminate the ASG) so the primary
  region apps genuinely die, then run a true failover from documentation.
- Work ONLY from the runbook. No improvising.

### Lab 4: DR of the DR (Backup the DR cluster)
- Create a Velero schedule for the DR cluster backups to a **separate** bucket.
- You now have a 3rd copy. Document where it lives and who can restore it.

---

## 🐛 Troubleshooting the Drill

| Problem | Likely Cause | Fix |
|---------|--------------|-----|
| Restore for *.php application errors in `Describe` | Missing `--namespace-mappings` or PVCs `Pending` | Check `kubectl get pvc -n <ns>`; create matching StorageClass; check KMS keys exist in DR region |
| DB replication lag grew > RPO | Writes during drill / oversized transactions | Rerun at low traffic; freeze writes before promote |
| Apps come up but health checks fail | Secrets/env still point at primary DB | Search for stale `DATABASE_URL` in Secrets/ConfigMaps and patch |
| ArgoCD won't sync DR overlay | Botched repo path / RBAC in DR cluster | Validate `argocd app get`; fix `destination.server` |
| Route53 shows healthy but no traffic | ALB DNS change not propagated / evaluate-target-health stale | Force re-query `nslookup app.ecommerce.com`; wait TTL; check target group registered |
| ECR pull fails in DR | Replication rule added after images were pushed | Push new image once more (replication is event-triggered), or use `--all` on rule |

---

## 📝 Interview Questions

1. **What is Game Day testing and why do teams do it?**
   - **Answer:** A rehearsed, schedule-bounded disaster drill (often with chaos
     injection) run against clones or during maintenance windows. Teams do it
     because unexercised runbooks fail in real incidents — Game Days convert
     theory into muscle memory and surface missing dependencies, permissions,
     and stale configs before customers are affected.

2. **How do you prove RPO without losing data?**
   - **Answer:** You can't prove RPO by eyeballing a schedule — you measure it.
     In a sandbox, restore the latest backup and check the *last write* your
     app committed is present (compare a sentinel record written before the
     backup). Lag on the read replica and last Velero `CompletionTimestamp`
     give you the numbers for the network-copy path.

3. **Why is RTO almost always bigger on paper than in practice?**
   - **Answer:** Paper plans assume perfect credentials, warm infra, and known
     states. Practice exposes cold ASGs, missing IAM in the DR region, stale
     secrets, DNS TTLs, and human decision time. The only way to converge
     paper RTO and real RTO is repeated measured drills.

4. **Failover vs failback — which is riskier?**
   - **Answer:** Failback. During failover you move forward into a fresh state;
     during failback you rejoin a region that may have diverged. Split-brain
     (both regions writing) and data conflicts are the classic failure modes —
     always quiesce writes, re-establish replication cleanly, and re-validate
     before cutting DNS back.

5. **What did you personally do to make DR "real" in your last project?**
   - **Answer:** (STAR-style) **S**ituation: 2-year-old offsite backups never
     restored. **T**ask: prove restorability and hit ≤ 30 min RTO. **A**ction:
     ran quarterly sandbox restores, added a chronicle of restore logs +
     a Route53 health-check-based failover, gamed Day 59-style drills with
     measured RTO/RPO. **R**esult: RTO dropped from 6+ hours to ~25 min and
     the next real incident was recovered from the runbook, not a scramble.

---

## 📚 Resources for Review

- [Day 56: Cluster Backup Strategies](Day-56_Cluster_Backup_Strategies/README.md)
- [Day 57: Cross-Region Replication](Day-57_Cross_Region_Replication/README.md)
- [Day 58: Failover Procedures](Day-58_Failover_Procedures/README.md)
- [Day 60: Final Review & Certification](Day-60_Final_Review_Certification_Preparation/README.md)
- [Velero Disaster Recovery Docs](https://velero.io/docs/v1.14/disaster-case/)
- [AWS Well-Architected DR Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel-disaster-recovery-objectives.html)

---

## 🎯 Today's Assignment

1. **Inventory** every backup, replica, and replication rule (Phase 0)
2. **Prove** a sandbox restore of one full app namespace with data validation
3. **Run** one measured failover drill and capture RTO/RPO
4. **Write up** the post-incident review with 3 concrete fixes
5. **File the runbook + scorecard** somewhere your team actually reads
6. **Optional stretch:** run the chaos drill (Phase 4) and do it cold

---

## 📊 Progress Tracker

```
Week 19-20 Disaster Recovery:
Day 56: Cluster Backup Strategies ⬜
Day 57: Cross-Region Replication ⬜
Day 58: Failover Procedures ⬜
Day 59: Disaster Recovery Practice ✅ (after drill)
Day 60: Final Review & Certification ⬜
```

---

## 🔗 Next Day Preview

**Day 60:** Final Review & Certification Preparation
- Full 60-day journey recap and knowledge blueprint
- CKA / CKAD / CKS / AWS certification paths for EKS
- Master interview Q&A across every domain
- Final capstone project and your graduation checklist

---

**Remember:** You don't rise to the occasion in a disaster — you fall to the
level of your training. **Drill it.** 🚀