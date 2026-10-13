# Technical Validation Notes

This file records the second-pass review of the 60-day Kubernetes/EKS learning curriculum. It is deliberately explicit about what was checked and what was **not** tested.

## Scope and status

- **Static review:** Completed a targeted high-risk review across the Day 1–60 lesson READMEs. Searches covered hard-coded sample credentials, unsafe network exposure, stale pricing/version assumptions, mutable image tags, deprecated Karpenter examples, and misleading zero-downtime/automatic-upgrade claims.
- **Manual corrections:** Applied to the lessons listed below.
- **Live execution:** Not performed. Commands have not been run against the user's AWS account or Kubernetes cluster.
- **Manifest validation:** No claim is made that every YAML block has been parsed by kubeconform/kubeval or applied to a disposable cluster. Some examples intentionally contain placeholders and require environment-specific values before use.

## High-impact corrections made

| Lesson | Correction |
|---|---|
| Day 1 | Replaced unsupported company-scale claims with generic use cases; clarified Compose scope and that rolling updates do not guarantee zero downtime. |
| Day 2 | Clarified that RollingUpdate can reduce disruption but cannot guarantee zero downtime. |
| Day 3 | Replaced absolute claims that NodePort is never production-ready or LoadBalancer is automatically production-ready with context-dependent guidance. |
| Day 4 | Corrected Secret security explanation; replaced sample credentials with placeholders/stringData; removed commands that print Secret values and prompted for lab-only credentials. |
| Day 6 | Replaced weak sample Secret values with placeholders. |
| Day 8–9 | Removed stale fixed-cost claims and misleading automatic-upgrade wording; clarified EKS pricing tiers, upgrade planning, and kubectl version compatibility. |
| Day 12 | Removed inline database password and used a Secret reference with a command to create a lab Secret. |
| Day 14, 41–42 | Removed hard-coded Grafana passwords and embedded credentials from examples. |
| Day 15 | Changed ECR scan example to require an image tag that actually exists. |
| Day 16, 53–54 | Updated Karpenter NodePool examples to the v1 API shape, corrected disruption/expiration fields, and clarified add-on and interruption-handling prerequisites. |
| Day 21 | Changed the EFS NFS ingress example so it sources traffic from the EKS node security group rather than `0.0.0.0/0`. |
| Day 32 | Removed invalid password placeholders from Argo CD login commands, documented TLS verification implications, and warned about exposing a LoadBalancer. |
| Day 35 | Made the Argo CD backup example use persistent storage and a version placeholder; documented its limited scope and need for off-cluster copies and restore tests. |
| Day 36 | Changed the CodeBuild image example to use a commit-derived tag rather than publishing `latest`. |
| Day 38–39 | Replaced a default Jenkins admin password and improved repository-token handling. |
| Day 48 | Marked KubeFed examples as historical/educational because the project is archived and no longer actively developed. |
| Day 52 | Clarified that replacing a node group cannot guarantee zero downtime. |
| Day 59 | Replaced HTTP probes against PostgreSQL ports with Service/EndpointSlice checks and conditional PostgreSQL readiness checks; fixed the DR load-balancer hostname placeholder. |

## Before using a hands-on example

1. Confirm the current AWS account and Region with `aws sts get-caller-identity` and your selected kubeconfig context with `kubectl config current-context`.
2. Replace all placeholders (cluster names, role ARNs, image tags, Regions, PVCs, namespaces, passwords, tokens, and DNS names).
3. Check the compatibility of Kubernetes, kubectl, EKS add-ons, Helm charts, Karpenter, Istio, and container images.
4. Review any command that creates billable resources or changes network access before running it.
5. Prefer a disposable lab account/cluster for destructive or security-sensitive exercises.
6. After applying YAML, inspect events and status; do not assume an accepted manifest means the workload is healthy.
7. For production changes, plan rollback, observability, and recovery validation.

## Current references

- [Amazon EKS pricing](https://aws.amazon.com/eks/pricing/) — cluster fees depend on standard versus extended Kubernetes version support; compute, storage, IP addresses, and network transfer are separate costs.
- [kubectl on Windows](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/) and [Kubernetes version skew policy](https://kubernetes.io/releases/version-skew-policy/) — keep kubectl within one minor version of the cluster API server.
- [Karpenter NodePools](https://karpenter.sh/docs/concepts/nodepools/) and [Karpenter NodeClasses](https://karpenter.sh/docs/concepts/nodeclasses/) — use the API schema and disruption fields that match the installed Karpenter release.
- [KubeFed archived repository](https://github.com/kubernetes-retired/kubefed) — the repository is archived and read-only.
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) — base64 is not encryption; access control and encryption configuration still matter.

## Remaining validation work

- Parse all YAML examples with schema-aware validation for the declared Kubernetes/add-on versions.
- Run safe examples in a disposable local cluster and test AWS-dependent commands only in an explicitly chosen lab account.
- Verify Helm chart values, IAM policies/trust relationships, and controller permissions against the exact versions used by each lesson.
- Test restore/failover exercises end to end and capture expected outputs.
- Review examples for cost cleanup, image digest pinning, and any remaining version-sensitive API assumptions.

**Important:** This is a static documentation review report, not a claim that all 60 lessons have passed runtime tests or are production-ready without adaptation.
