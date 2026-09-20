# Amazon EKS Production Engineering Guide

A production-oriented operating layer for the existing 60-day curriculum.

## Production mental model

Think in five layers: AWS foundation, EKS control plane, data plane, platform add-ons, and workloads.

For every component ask: who owns it, how does it change, how is it tested, how is it observed, and how is it recovered?

## Identity

Learn both EKS Pod Identity and IRSA. Current AWS guidance recommends EKS Pod Identity for supported workloads and add-ons; IRSA remains important for compatibility and existing environments.

References:
- https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html
- https://docs.aws.amazon.com/eks/latest/userguide/add-ons-iam.html

Practice one least-privilege service-account-to-IAM mapping, prove allowed AWS API calls, remove one permission and verify the expected failure, then test cross-account delegation.

## Add-ons

Treat VPC CNI, CoreDNS, kube-proxy, CSI drivers, load balancer controller, and Pod Identity Agent as versioned platform dependencies. AWS-managed add-ons provide lifecycle management and validated versions. EKS Auto Mode can provide several common platform capabilities itself.

Reference:
- https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html

Keep an inventory with component, version, configuration, dependency, upgrade test, failure signal, and recovery procedure.

## Networking drills

Amazon VPC CNI gives Pods VPC networking and manages ENI/IP capacity on nodes.

Reference:
- https://docs.aws.amazon.com/eks/latest/best-practices/vpc-cni.html

Evidence commands:

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl -n kube-system get pods -l k8s-app=aws-node -o wide
kubectl -n kube-system logs daemonset/aws-node --tail=100
kubectl get events -A --sort-by=.lastTimestamp
```

Break one path in a sandbox: IP capacity, NetworkPolicy, DNS, Service selector, or ALB health checks. Record symptom -> evidence -> root cause -> fix -> prevention.

## Production workload baseline

Review immutable image references, requests and limits, startup/readiness/liveness probes, securityContext, Pod Security Standards, least-privilege ServiceAccounts, PDB where justified, topology spread where justified, scaling policy, graceful termination, telemetry, and rollback.

## Delivery model

```text
PR -> tests -> image build -> scan/sign -> ECR -> desired-state change
   -> GitOps reconciliation -> rollout -> telemetry gates -> rollback/approval
```

Keep the same application artifact across environments. Prefer immutable tags or digests.

## Observability

Cover latency, traffic, errors, and saturation. Add Pending pods, node pressure, API errors, CNI IP usage, PVC errors, and rollout health.

Every alert should include impact, first checks, dashboard/query, safe mitigation, escalation, and verification.

## Autoscaling

Separate pod scaling (HPA/VPA), node scaling (Cluster Autoscaler/Karpenter), and capacity selection (instance type, architecture, topology, purchase option).

## Upgrade discipline

Inventory Kubernetes/add-on/controller/CRD/AMI versions, identify deprecated APIs, test in non-production, verify capacity headroom and PDBs, verify restore procedures, upgrade one environment, watch health signals, and recover using the safest evidence-based action.

## Disaster recovery

Define RTO/RPO per workload. Separate Kubernetes object backup, volume recovery, external data recovery, artifact recovery, DNS/traffic failover, and human runbook execution. Test restore and measure actual recovery time.

## Incident response

Use:

`Stabilize -> Observe -> Isolate -> Mitigate -> Verify -> Recover -> Prevent`

Do not use broad admin permissions as a debugging shortcut; collect evidence first.

## Portfolio evidence

For each topic keep an architecture diagram, exact commands, source manifests/Helm/Kustomize/Terraform, expected output, one intentionally broken case, root-cause notes, security review, cost notes, and cleanup/recovery steps.
