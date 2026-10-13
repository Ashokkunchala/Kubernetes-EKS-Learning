# Understanding-First Kubernetes & EKS Study Workflow

> Use this workflow alongside the 60-day curriculum. It is deliberately **understanding-first**, not a fixed 30-day schedule. Work through it in one week if your pace allows, but spend extra time on a topic when you cannot explain or demonstrate it yet.

## The daily learning loop (60–90 minutes)

1. **Read (15–20 min):** Read the relevant day README once without trying to memorize every term.
2. **Collect terms (10 min):** Write down unfamiliar words, abbreviations, error messages and YAML fields.
3. **Translate (10 min):** For each term, write a plain-English definition and one example. Use [Kubernetes & EKS Concepts in Plain English](KUBERNETES_EKS_CONCEPTS_PLAIN_ENGLISH.md).
4. **Run (15–25 min):** Run the lab commands. Start with read-only commands; review any command that creates, deletes or changes cloud resources before running it.
5. **Observe (10 min):** Record expected output, actual output and any difference.
6. **Explain (5–10 min):** Explain the concept aloud as if teaching a teammate.
7. **Review (5 min):** Mark the checkpoint below only if you can explain it without copying the README.

You can split a session across the day. Consistency and evidence of understanding matter more than finishing a lesson quickly.

## One-week orientation path

This is a suggested sequence, not a deadline. The existing day folders remain the detailed source of truth.

| Session | Focus | You should be able to explain |
|---|---|---|
| 1 | Kubernetes foundations | Cluster, control plane, node, Pod, Deployment, desired state |
| 2 | Workloads and health | ReplicaSet, rollout, probes, requests/limits, CrashLoopBackOff |
| 3 | Networking | Service types, DNS, Ingress/controller, CNI, ingress vs egress |
| 4 | Configuration and storage | Namespace, ConfigMap, Secret, PV/PVC, StorageClass, CSI |
| 5 | Identity and security | Authentication vs authorization, IAM vs RBAC, Role/RoleBinding, service account, least privilege |
| 6 | AWS EKS | EKS control plane, VPC/subnets, node groups, EKS access, IRSA/Pod Identity |
| 7 | Operations and recovery | Events/logs, metrics vs logs vs traces, scaling, backup vs replication, RTO/RPO |

For each session, open the matching day README from the root [README.md](README.md), run the practice section, and use the glossary/quick reference when needed.

## Topic checkpoint template

Copy this template into your notes after each session:

```text
Topic:
Plain-English explanation:
What problem does it solve?
Where does it live? (AWS / cluster / namespace / node / Pod / container)
What are 2 related terms?
Command or YAML I tested:
Expected result:
Actual result:
What confused me:
Can I explain it without notes? YES / NOT YET
```

## Security and identity checkpoint

Before moving past identity topics, make sure you can explain these without mixing them up:

- [ ] Authentication proves identity; authorization evaluates permission.
- [ ] Kubernetes RBAC controls Kubernetes API actions; AWS IAM policies control AWS API actions.
- [ ] A Role defines permissions; a RoleBinding connects subjects to a Role/ClusterRole within a namespace.
- [ ] A ClusterRoleBinding grants its referenced ClusterRole at cluster scope.
- [ ] A Kubernetes ServiceAccount is an identity for a workload, not automatically an AWS IAM role.
- [ ] IRSA and EKS Pod Identity are mechanisms for providing workload AWS credentials/permissions; configuration and supported integrations matter.
- [ ] OIDC is an identity protocol/provider relationship, not a permission policy by itself.
- [ ] Base64 is encoding, not encryption.
- [ ] Least privilege means granting only the permissions necessary for the task.
- [ ] `kubectl auth can-i` checks authorization for an API action; it does not prove every authentication path or external AWS permission is correct.

Practice with these commands, replacing placeholders:

```bash
kubectl auth can-i list pods -n <namespace>
kubectl auth can-i delete deployments -n <namespace>
kubectl get serviceaccounts -n <namespace>
kubectl get role,rolebinding -n <namespace>
kubectl get clusterrole,clusterrolebinding
aws sts get-caller-identity
```

Do not assume a denied command has only one possible cause. Confirm the identity, requested verb/resource/namespace, relevant policy/binding, and the error message.

## Safe practice rules

- Prefer local clusters (such as kind or minikube) for Kubernetes-only exercises.
- For EKS, confirm the active AWS account and region before applying changes.
- Review cost-impacting resources before creation; clean them up when the exercise is complete.
- Never commit AWS keys, kubeconfig credentials, tokens, passwords or real secret values.
- Use placeholders in examples and a dedicated non-production namespace/account.
- Do not copy a production deletion or cluster-update command without understanding its scope and rollback plan.

## Completion criteria

A session is complete when you can:

1. Explain the core concepts in your own words.
2. Identify where the relevant resource or identity exists.
3. Run the lab or inspect the configuration and describe the output.
4. Explain one likely failure and the first evidence you would collect.
5. Answer the checkpoint questions without simply repeating a definition.

If you cannot do one of these, mark **NOT YET**, revisit the concept guide, and retry the lab. That is normal learning—not failure.

## References

- [Plain-English Concepts Guide](KUBERNETES_EKS_CONCEPTS_PLAIN_ENGLISH.md)
- [A–Z Glossary](GLOSSARY.md)
- [Keyword Coverage Checklist](KUBERNETES_KEYWORDS_COVERAGE.md)
- [Quick Reference](EKS_QUICK_REFERENCE.md)
- [Troubleshooting Playbook](TROUBLESHOOTING_PLAYBOOK.md)
