# 📅 Day 40: CI/CD Interview Preparation & Assessment

## 🎯 Learning Objectives
By the end of today, you will:
- Answer CI/CD-on-EKS questions confidently
- Explain pipelines conceptually at all seniority levels
- Identify anti-patterns and propose fixes
- Be ready for scenario-based system design questions

---

## 🧠 Plain-English terms before you start

| Term | Plain-English meaning |
|---|---|
| **Continuous Integration (CI)** | Frequently integrating changes and automatically building/testing them. |
| **Continuous Delivery** | Keeping software in a releasable state; production release may require approval. |
| **Continuous Deployment** | Automatically releasing every change that passes the required checks to production. |
| **Artifact** | A versioned output such as a container image or package. |
| **Immutable tag/digest** | An image identifier that cannot silently point to a different build; a digest identifies exact content. |
| **Promotion** | Moving the same tested artifact through environments rather than rebuilding different content for each one. |

**Analogy:** CI checks each new part, delivery prepares a releasable product, and deployment automatically puts approved changes into production.

**Try / observe:** Explain your pipeline with evidence: commit → test/scan → image digest → environment approval → deployment → health verification → rollback.

**Troubleshooting clue:** Separate build failures from registry access, cluster authorization, rollout health, and application failures. The failing stage tells you where to gather evidence first.

**Explain it back:** What is the difference between continuous delivery and continuous deployment?

---

## 📖 Core Concepts - The Big Picture

```
┌───────────────────────────────────────────────────────────────┐
│                 CI/CD ARCHITECTURE ON EKS                     │
│                                                               │
│   CALL CENTER (Dev)              RUNTIME (Prod)               │
│   ┌─────────────────┐            ┌───────────────────┐       │
│   │ Source → Build  │            │ ArgoCD/GitOps     │       │
│   │ Test  → Scan    │──Deploy──▶ │ EKS (prod apps)   │       │
│   │ Push → ECR      │  (Git)     │ SelfHeal/Drift    │       │
│   └─────────────────┘            └───────────────────┘       │
│                                                                 │
│   ⚠ Anti-pattern: direct kubectl from CI into prod           │
│   ⚠ Anti-pattern: storing secrets in Git/manifests            │
│   ⚠ Anti-pattern: mutable image tags (latest)                 │
└───────────────────────────────────────────────────────────────┘
```

### The Three Pillars to Emphasize

1. **Immutable artifacts** — every image is uniquely tagged; never reuse mutable tags in production.
2. **Git as source of truth** — GitOps for CD; approvals and reviews before release.
3. **Shift-left security** — scan code, dependencies, and images early; sign and verify.

---

## 🗣️ Interview Questions by Level

### Beginner (0-2 years)

1. **What is the difference between CI and CD?**
   - **Answer:** CI (Continuous Integration) automatically builds and tests each change. CD (Continuous Delivery/Deployment) automatically ships the tested change to environments. Delivery requires manual approval to prod; Deployment is fully automatic.

2. **What is a pipeline stage?**
   - **Answer:** A discrete phase: fetch source, install deps, build, run tests, scan, package, deploy, smoke-test.

3. **What is ECR and why use it?**
   - **Answer:** Amazon's container registry; stores images privately, integrates with IAM, supports vulnerability scanning and lifecycle policies.

4. **What is `kubectl apply -f` doing?**
   - **Answer:** Sends the manifest to the API server, which creates or updates the desired state of the resources.

### Intermediate (2-5 years)

5. **How would you set up a secure CI/CD for EKS?**
   - **Answer:** OIDC-federated IAM roles (no static keys), least-privilege roles per stage, image scanning gates, secret injection via AWS Secrets Manager/ASM, and GitOps-based deployment with approval gates.

6. **GitHub Actions vs CodePipeline vs Jenkins — how do you choose?**
   - **Answer:** Depends on team & cloud. GitHub Actions = code-adjacent, cloud-agnostic, huge ecosystem. CodePipeline = AWS-native, deep AWS integration. Jenkins = self-hosted, max flexibility, needs maintenance. Choose by lock-in tolerance and ops capacity.

7. **How do you handle secrets in the pipeline?**
   - **Answer:** Inject at runtime from AWS Secrets Manager or GitHub encrypted secrets; never commit them; use external secrets operators for in-cluster secrets.

8. **What is the difference between rolling, blue/green, and canary deployments?**
   - **Answer:** Rolling replaces pods incrementally (default Deployment); blue/green runs two full versions and switches (ALB/route53); canary shifts a small % of traffic to the new version first (Argo Rollouts).

### Senior (5+ years)

9. **How do you design a zero-downtime deploy pipeline?**
   - **Answer:** Rolling updates with maxSurge/maxUnavailable tuned, readiness probes, preStop hooks & graceful shutdown, blue/green via Argo Rollouts when required, and automated rollback on failed health.

10. **How do you ensure traceability from commit to production?**
    - **Answer:** Tag images with the commit SHA, store SBOM/labels, record release in GitOps repo, correlate via deployment annotations, and centralize logs/metrics.

11. **How do you scale a Jenkins/CI on EKS and manage its cost?**
    - **Answer:** Ephemeral Kubernetes agent pods, spot instances for agents, resource limits, caching, and auto-scaling agent pools; pause idle agents.

12. **How would you roll out a change to many microservices safely?**
    - **Answer:** Dependency-aware order, per-service canaries with automated analysis, feature flags, and a central release orchestration; roll back the offending service without affecting others.

---

## 🧠 Scenario Questions

### Scenario 1: "Deployment failed in production mid-release. What now?"
**Answer:**
1. Halt the rollout (`kubectl rollout pause deployment/x`).
2. Assess blast radius (pods, traffic, errors).
3. Rollback immediately: `git revert` + ArgoCD auto-sync (preferred) or `rollout undo`.
4. Preserve the bad version's logs/snapshot for postmortem.
5. Investigate root cause (image, config, dependency) before re-releasing with an approval gate.

### Scenario 2: "Team pushes to `latest` tag, prod shows old image."
**Answer:** Mutable `latest` causes caching/race issues — the node cached an old image. Fix: use immutable tags (`<sha>` / semver), configure `imagePullPolicy: Always` for dev only, and never rely on `latest` in prod.

### Scenario 3: "A stored secret leaks in the repo."
**Answer:** Rotate it immediately, purge with a secret-scanning tool (git-secrets, truffleHog), add a secret scanner to CI, and move secrets to AWS Secrets Manager with gitignore protection.

---

## 🛠️ Build-Your-Answer Framework (STAR for pipelines)

Use **STAR** for behavioral questions:
- **S**ituation: describe the release context.
- **T**ask: what needed to ship.
- **A**ction: the pipeline stages you built/hardened.
- **R**esult: deploy time, rollback speed, failure reduction — quantified.

---

## 📚 Resources for Review

- Day 36-39 (CI/CD lessons in this folder)
- [Day 34: ArgoCD Sync & Rollback](Day-34_ArgoCD_Sync_Rollback/README.md)

### External
- [AWS Well-Architected - CI/CD](https://docs.aws.amazon.com/wellarchitected/latest/operational-excellence-pillar/)
- [GitHub Actions best practices](https://docs.github.com/en/actions)

---

## 🎯 Today's Assignment

1. **Answer** each interview question aloud (record yourself)
2. **Draw** the CI/CD flow on a whiteboard from memory
3. **Run** a practice mock interview with a peer
4. **Pick** one query and write the STAR story
5. **Review** all anti-patterns and their fixes

---

## 📊 Progress Tracker

```
Week 11-12 CI/CD COMPLETE:
Day 36: AWS CodePipeline Integration ⬜
Day 37: GitHub Actions for EKS ⬜
Day 38: Jenkins on EKS ⬜
Day 39: CI/CD Pipeline Practice ⬜
Day 40: CI/CD Interview Preparation ⬜
```

---

## 🔗 Next Week Preview

**Week 13-14:** Monitoring & Observability (Prometheus, Grafana, X-Ray, CloudWatch)

---

**Remember:** Confidence comes from having actually built and broken it. Speak from experience! 🚀
