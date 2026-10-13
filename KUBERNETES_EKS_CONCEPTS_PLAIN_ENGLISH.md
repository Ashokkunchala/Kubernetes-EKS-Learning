# Kubernetes & EKS Concepts in Plain English

> **Purpose:** Understand the language used in Kubernetes, AWS EKS, DevOps and security. This is a companion to the detailed day-by-day material—not a replacement for it.
>
> **Learning rule:** For every new term, learn (1) what it means, (2) what problem it solves, (3) where it runs, (4) who or what uses it, and (5) how to verify it.

## Start here: Authentication vs Authorization

These words sound similar, but answer different questions.

| Term | Question it answers | Plain-English meaning | Kubernetes/EKS example |
|---|---|---|---|
| **Authentication (AuthN)** | **Who are you?** | Prove your identity. | AWS IAM credentials and the EKS access flow identify a person or role calling the cluster API. |
| **Authorization (AuthZ)** | **What are you allowed to do?** | Check permissions after identity is known. | Kubernetes RBAC may allow a user to list Pods in `dev` but not delete Deployments in `prod`. |
| **Accounting / Auditing** | **What happened?** | Record actions so they can be reviewed. | Kubernetes audit logs and AWS CloudTrail record API activity. |

**Everyday analogy:** At an office entrance, your ID card proves who you are (**authentication**). The security system checks whether your card lets you enter the server room (**authorization**). The entry log records that you entered (**auditing**).

### A Kubernetes example

Suppose a developer runs:

```bash
kubectl get pods -n payments
```

1. **Authentication:** The API server receives an identity for the caller.
2. **Authorization:** The configured authorization system checks whether that identity can `list` `pods` in namespace `payments`.
3. **Admission:** If the request creates or changes an object, admission controls can validate or mutate it. A simple read such as `get pods` is not a create/update admission workflow.
4. **Response:** The request succeeds only if applicable checks pass.

A common distinction:
- **401 Unauthorized** usually means the caller is not authenticated or valid credentials are missing/invalid.
- **403 Forbidden** means the caller was understood but does not have permission for that action (or access was otherwise forbidden).

HTTP responses depend on the component returning them; use the message and logs to confirm the actual cause.

### Kubernetes RBAC: who can do what?

RBAC means **Role-Based Access Control**. It grants API permissions to users, groups or service accounts through roles and bindings.

- **Role:** Permissions within one namespace.
- **ClusterRole:** A reusable set of permissions that can be cluster-wide, or used in a namespace through a RoleBinding.
- **RoleBinding:** Connects subjects to a Role or ClusterRole within one namespace.
- **ClusterRoleBinding:** Connects subjects to a ClusterRole at cluster scope.
- **ServiceAccount:** An identity intended for a workload running in Kubernetes.

Example: allow a service account to read Pods in namespace `demo`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader
  namespace: demo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: demo
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: demo
subjects:
  - kind: ServiceAccount
    name: pod-reader
    namespace: demo
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

Save as `pod-reader-rbac.yaml`. The namespace `demo` must exist first.

```bash
kubectl create namespace demo
kubectl apply -f pod-reader-rbac.yaml
kubectl auth can-i list pods -n demo --as=system:serviceaccount:demo:pod-reader
kubectl auth can-i delete pods -n demo --as=system:serviceaccount:demo:pod-reader
```

Expected idea: listing should be allowed; deleting should be denied. Impersonation tests using `--as` require the person running the command to have permission to impersonate that identity.

> **Do not mix up AWS IAM and Kubernetes RBAC.** IAM controls access to AWS APIs and participates in EKS identity/access configuration. Kubernetes RBAC controls permissions for Kubernetes API resources. An application that calls S3 needs AWS permissions too; Kubernetes RBAC alone does not grant S3 access.

## Four identity/access paths you will hear about in EKS

| Name | Main purpose | Remember it as |
|---|---|---|
| **AWS IAM user/role/policy** | Identity and permissions for AWS APIs; IAM roles can be assumed by people, services or workloads. | AWS account access |
| **EKS access entry / access policy** | Registers an IAM principal for cluster access and can associate EKS access policies. Kubernetes groups/RBAC can also be part of the authorization design. | Connecting AWS identity to cluster access |
| **Kubernetes RBAC** | Grants permissions to Kubernetes API actions and objects. | What can I do to Pods, Deployments, Secrets? |
| **IRSA / EKS Pod Identity** | Gives a workload AWS permissions without distributing long-lived AWS keys into the container. | What can this Pod do in AWS? |
| **OIDC** | An identity protocol used to let a trusted identity provider issue verifiable tokens/claims. | A way to establish trusted identity |
| **JWT / bearer token** | A token presented to prove or carry claims about an identity. A token is not automatically proof that every requested action is permitted. | A credential/token, not a permission policy |

Which EKS access mechanism is used depends on cluster configuration and workload setup. Do not assume every EKS cluster uses only one mechanism.

## The access-control sequence

```text
Human / CI job / Kubernetes workload
              |
              v
       Present identity proof
       (authentication / AuthN)
              |
              v
       Is this request allowed?
       (authorization / AuthZ)
              |
              +---- create/update? ----> Admission checks
              |                              |
              v                              v
       API action proceeds only if required checks pass
```

This is a learning model, not a claim that every implementation uses exactly the same internal order for every request. Kubernetes authorization and admission are distinct stages.

## Similar words that are easy to confuse

| Term A | Term B | Difference |
|---|---|---|
| Authentication | Authorization | Identity proof vs permission check |
| Authorization | Admission control | Permission to perform an API action vs policy checks/mutation on an API object request |
| Encryption | Encoding | Encryption requires a key to recover data; encoding (for example Base64) is a representation change, not confidentiality |
| Secret | ConfigMap | Intended for confidential values vs non-confidential configuration; Kubernetes Secret data still needs suitable access controls and encryption-at-rest configuration |
| Pod | Node | Pod runs workload containers; node is the machine that runs Pods |
| Deployment | Service | Deployment manages desired Pod replicas/updates; Service provides a stable way to reach a group of Pods |
| Ingress | Ingress Controller | Ingress is the routing API object; a controller implements the routing behavior |
| Readiness probe | Liveness probe | Readiness asks whether traffic should be sent to a container; liveness detects when it should be restarted |
| Request | Limit | Resource request influences scheduling and reservation; limit caps runtime resource use |
| HPA | Cluster Autoscaler / Karpenter | HPA scales Pod replicas; node autoscaling adds/removes compute capacity |
| StatefulSet | Deployment | StatefulSet is designed for workloads needing stable identity/storage ordering; Deployment suits interchangeable replicas |
| PersistentVolume (PV) | PersistentVolumeClaim (PVC) | PV is the provisioned storage resource; PVC is a workload's storage request |
| IAM Role | IAM Policy | Role is an identity that can be assumed; policy describes allowed/denied actions and resources |
| Security Group | NetworkPolicy | AWS network-level rules for supported interfaces/resources vs Kubernetes Pod traffic rules enforced by a compatible network implementation |
| Monitoring | Observability | Monitoring checks known signals/conditions; observability uses metrics, logs and traces to investigate behavior |
| Continuous Integration | Continuous Delivery/Deployment | CI integrates/tests changes; delivery keeps changes releasable; deployment automates release to an environment |
| Declarative | Imperative | Describe desired end state vs issue explicit commands to change state |
| Desired state | Current state | What should exist vs what is observed now |
| Idempotent | Repeatable but non-idempotent | Idempotent repeated application converges to the same intended result without accumulating extra effects; a repeatable command may still create duplicates |
| Availability | Durability | Service can be used now vs data survives loss/time |
| RTO | RPO | Target time to restore service vs target amount of data/time that can be lost |

## More essential terms in everyday language

### Kubernetes architecture

- **Cluster:** The whole Kubernetes environment: control plane plus worker capacity.
- **Control plane:** Components that expose the API, store cluster state, schedule workloads and reconcile desired state.
- **API server:** The front door for Kubernetes API requests.
- **etcd:** The key-value store for Kubernetes cluster state.
- **Scheduler:** Chooses a suitable node for an unscheduled Pod.
- **Controller / reconciliation loop:** Repeatedly compares observed state with desired state and tries to close the gap.
- **Kubelet:** Node agent that makes sure assigned Pods/containers are running as specified.
- **Container runtime:** Software that starts and manages containers on a node.
- **CRD:** Custom Resource Definition; extends the Kubernetes API with a new resource type.
- **Operator:** A controller pattern that automates application-specific operational tasks using Kubernetes APIs.

### Workloads and release

- **Image:** Packaged application filesystem/configuration used to create a container.
- **Container:** A running process isolated using operating-system features.
- **Replica:** One copy of a workload Pod.
- **Rollout:** Progress of a new workload version being released.
- **Rollback:** Return to a previous version/configuration.
- **Canary release:** Send a small portion of traffic to a new version before increasing exposure.
- **Blue/green:** Run old and new environments side by side and switch traffic between them.
- **Sidecar:** A helper container that runs alongside an application container in the same Pod.
- **Init container:** A container that must finish its setup task before regular app containers start.
- **CrashLoopBackOff:** A container keeps failing and Kubernetes is delaying restart attempts.
- **ImagePullBackOff:** Kubernetes cannot pull an image and is retrying with backoff.
- **OOMKilled:** A container was killed after exceeding available/enforced memory.
- **Eviction:** Kubernetes removes a Pod from a node, commonly under resource pressure or policy.

### Networking

- **DNS:** Resolves a name to a network destination.
- **ClusterIP:** Internal virtual IP used to reach a Service inside the cluster.
- **NodePort:** Exposes a Service on a port on cluster nodes.
- **LoadBalancer Service:** Requests an external/internal load-balancer integration from the environment.
- **Ingress:** HTTP/HTTPS routing rules for incoming traffic; requires a controller to implement them.
- **CNI:** Container Network Interface; the plugin integration used to configure container networking.
- **CIDR:** Notation for an IP address range, such as `10.0.0.0/16`.
- **Egress / ingress:** Traffic leaving a boundary / entering a boundary. Always identify the reference point (Pod, node, VPC, firewall or load balancer).
- **TLS:** Protects data in transit and helps authenticate endpoints using certificates.
- **mTLS:** Both sides authenticate each other with certificates.
- **Service mesh:** Infrastructure layer for service-to-service traffic controls, identity, telemetry and often encryption.

### Configuration and storage

- **Environment variable:** Named value supplied to a process at runtime.
- **Volume:** Storage mounted into a Pod/container.
- **Dynamic provisioning:** Create storage automatically when a claim requests it, using a StorageClass/driver.
- **CSI driver:** Standard interface plugin through which Kubernetes talks to a storage system.
- **Snapshot:** Point-in-time copy/representation used for recovery, depending on storage support.
- **Stateful workload:** A workload that needs persistent data or stable identity.
- **Secret rotation:** Replace a credential/certificate before or after expiration or compromise.
- **External Secrets:** Pattern/tooling that syncs data from an external secret store into Kubernetes, depending on the controller and configuration.

### AWS / EKS

- **VPC:** Isolated virtual network in AWS.
- **Subnet:** An IP range inside a VPC, associated with Availability Zones.
- **Route table:** Rules that decide where network traffic is directed.
- **NAT Gateway:** Allows private-subnet resources to initiate outbound IPv4 connections without accepting unsolicited inbound connections through that gateway.
- **Security Group:** Stateful virtual firewall attached to supported AWS network interfaces/resources.
- **IAM Role:** An AWS identity with permissions that can be assumed.
- **IAM Policy:** JSON rules that allow or deny actions on resources.
- **Trust policy:** Defines who or what may assume a role.
- **OIDC provider:** Configuration that lets AWS or another relying party trust tokens from an identity provider.
- **IRSA:** IAM Roles for Service Accounts; associates Kubernetes service-account identity with an AWS IAM role through OIDC federation.
- **EKS Pod Identity:** EKS capability to associate Kubernetes service accounts with IAM roles for AWS SDK/CLI access from Pods.
- **Node group:** Set of worker nodes managed together.
- **Add-on:** Packaged cluster component, such as networking or storage integration.
- **ECR:** AWS container image registry.
- **EBS:** AWS block storage commonly used for node-attached persistent volumes.
- **EFS:** AWS managed shared file storage that can be mounted by multiple clients where supported.
- **ALB:** Application Load Balancer; routes HTTP/HTTPS traffic using layer-7 rules.
- **CloudWatch:** AWS monitoring/logging service.
- **CloudTrail:** Records AWS API activity for audit and investigation.

### Reliability, operations and security

- **High availability (HA):** Design to keep the service working through expected component failures.
- **Fault tolerance:** Continue operating despite a defined failure.
- **Disaster recovery (DR):** Restore services/data after a major incident.
- **Backup vs replication:** A recoverable historical copy vs copying changes to another location; replication can also replicate deletion/corruption.
- **RTO / RPO:** Recovery-time target / recovery-point target.
- **SLO / SLI / SLA:** Service objective / measured indicator / formal service commitment.
- **Saturation:** A resource (CPU, memory, disk, connection pool, etc.) is near its capacity.
- **Latency:** Time required for an operation.
- **Throughput:** Amount of work processed per unit of time.
- **Alert fatigue:** Too many low-value alerts make important alerts easier to miss.
- **Least privilege:** Grant only the permissions needed for a task.
- **Defense in depth:** Use multiple independent security controls.
- **Threat model:** Identify assets, likely attackers, trust boundaries and failure/abuse paths.
- **Vulnerability:** A weakness that may be exploitable.
- **Patch:** Update intended to fix defects or vulnerabilities.
- **Audit log:** Record of actions for accountability and investigation.
- **Policy as code:** Express rules in machine-checkable form and evaluate them automatically.
- **Admission controller:** API-server stage that can validate or mutate objects in applicable requests.
- **Pod Security Standards:** Kubernetes security profiles commonly named Privileged, Baseline and Restricted.

## How to decode an unfamiliar word (use this every time)

When you find a term in a README, error, interview question or YAML file, write five lines:

1. **Plain meaning:** Explain it to a junior teammate without using the same jargon.
2. **Problem solved:** What would be hard or unsafe without it?
3. **Where it lives:** AWS account, VPC, cluster, namespace, node, Pod, container, pipeline, or external system?
4. **Related terms:** Which two or three concepts are commonly used with it?
5. **Proof:** Which command, log, dashboard or configuration shows that it works?

Example — **readiness probe**:
- Meaning: a health check that determines whether a container should receive normal traffic.
- Problem: prevents traffic reaching a starting or temporarily unready application.
- Where: configured on a container in a Pod spec.
- Related: Service, endpoints, liveness probe.
- Proof: `kubectl describe pod <pod-name>`, then inspect Pod events and readiness status.

## Troubleshooting phrases translated

| Phrase you see | What it means to investigate first |
|---|---|
| `Unauthorized` | Which identity/credentials reached the API? Is authentication configured correctly? |
| `Forbidden` | Which identity was recognized, what exact action/resource/namespace was requested, and which permission rule applies? |
| `Pending` Pod | Scheduling constraints, requests/capacity, node selectors/affinity, taints/tolerations, PVC status and events |
| `CrashLoopBackOff` | Previous container logs, exit code, startup configuration, probes, dependencies and resource limits |
| `ImagePullBackOff` | Image name/tag, registry reachability, credentials and pull permissions |
| `FailedScheduling` | `kubectl describe pod` events, resource requests, affinity, taints, topology and storage constraints |
| `CreateContainerConfigError` | Referenced Secret/ConfigMap, key name, volume and environment configuration |
| `PVC Pending` | StorageClass, CSI driver, access mode, capacity, topology and provisioning events |
| `context deadline exceeded` | Network path, endpoint reachability, DNS, latency and component logs; don't assume one cause |
| `AccessDenied` from AWS | Which AWS principal is being used, which API action/resource is denied, and IAM/resource policy conditions |

Useful first commands:

```bash
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --all-containers=true
kubectl logs <pod-name> -n <namespace> --previous
kubectl auth can-i list pods -n <namespace>
aws sts get-caller-identity
aws eks describe-cluster --name <cluster-name> --region <region>
```

Replace placeholders before running commands. AWS CLI commands require AWS credentials and permissions. Some checks may need to be performed by a cluster administrator.

## Daily 15-minute terminology drill

- **5 minutes:** Pick five unfamiliar terms from today's lesson.
- **5 minutes:** Explain each term out loud using a real example.
- **5 minutes:** Run one read-only command or inspect one YAML example that proves how a term is used.

Keep a small notebook with columns: `Term | Plain meaning | Example | Related terms | Command/proof`.

## Self-check

Try answering before looking anything up:

1. A user can log in but cannot delete a Pod. Is the problem necessarily authentication?
2. Does Kubernetes RBAC give a Pod permission to read an S3 bucket?
3. What is the difference between a Role and a RoleBinding?
4. Does Base64 encoding make a Kubernetes Secret confidential?
5. Which scales application replicas: HPA or node autoscaling?
6. What does a readiness probe change compared with a liveness probe?
7. Why can a Pod be Running but not Ready?
8. What is the difference between a security group and a Kubernetes NetworkPolicy?

**Answer key**
1. No. Authentication may have succeeded; authorization is a likely area to check.
2. No. AWS permissions such as a workload IAM role are needed for S3 API access.
3. Role defines permissions; RoleBinding attaches permissions to subjects in a namespace.
4. No. Base64 is encoding, not encryption.
5. HPA scales replicas; node autoscaling scales compute nodes/capacity.
6. Readiness controls whether the Pod is considered ready for traffic; liveness can trigger a container restart after repeated failures.
7. Running says the container is running; readiness can still fail due to startup or application health.
8. A security group filters traffic for supported AWS network interfaces/resources; NetworkPolicy defines Pod traffic rules, subject to network-plugin enforcement.

## Further reading

- [Kubernetes authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Kubernetes authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Amazon EKS cluster access](https://docs.aws.amazon.com/eks/latest/userguide/cluster-auth.html)
- [IAM roles for service accounts (IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
- Existing [A–Z Glossary](GLOSSARY.md), [Keyword Coverage Checklist](KUBERNETES_KEYWORDS_COVERAGE.md), and [Quick Reference](EKS_QUICK_REFERENCE.md).
