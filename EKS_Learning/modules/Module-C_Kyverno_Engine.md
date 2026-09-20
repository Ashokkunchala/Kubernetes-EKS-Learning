# 🧩 Module C: Kyverno Policy Engine on EKS

> **Prereq:** [Module A Admission Controllers](./Module-A_Admission_Controllers.md) · [Day 15 EKS Security](../Day-15_EKS_Security_Best_Practices/README.md) · [Day 17 Network Policies Calico](../Day-17_Network_Policies_Calico/README.md)

Hand-writing webhooks (like Module A) does not scale to a fleet. Kyverno turns *policies* into *admission + mutation + cleanup logic* — you write YAML policy, Kyverno generates the webhook wiring for you.

## 🎯 Learning Objectives

By the end of this module, you will understand:
- Why a policy engine beats hand-rolled webhooks
- How Kyverno plugs into the admission flow (generated webhooks)
- `ClusterPolicy` vs `Policy`, and validate / mutate / generate kinds
- 4 worked ClusterPolicies (require tag, block `latest`, no privileged, add label)
- Installing Kyverno via Helm and testing good vs bad pods
- How this maps to NIST/NSA Kubernetes hardening guidance

---

## 📖 Core Concepts

### Policy engine vs your own webhook

```
┌─────────────────────────────────────────────────────────┐
│            HAND-WRITTEN WEBHOOK vs KYVERNO              │
│                                                         │
│  Webhook (Module A)         Kyverno                     │
│  - Server(s) to run        - One kyverno Deployment     │
│  - TLS cert + CA every env - Self-managed CA             │
│  - Write controller        - Write YAML policy           │
│  - Deploy per cluster      - Helm + CRDs                 │
│  - No dashboard            - kubectl kyverno + policies  │
│  - Per-webhook rules        - Per-policy rules (CRD)     │
└─────────────────────────────────────────────────────────┘
```

Kyverno still uses **ValidatingWebhookConfigurations under the hood** — but it generates and rotates them from your policy objects. No cert management, no per-policy services.

### How it sits in the flow (from Module A)

```
Request ─► authn ─► authz ─► [ Kyverno MUTATING webhook ] ─► [ Kyverno VALIDATING webhook ] ─► etcd
                                (add defaults)              (enforce policy)
```

### Resource model

```yaml
# Kyverno policy kinds
ClusterPolicy      # cluster-wide, matches any resource, any namespace cluster
Policy             # namespaced enforcement
├── spec.rules[].match    # what resources apply to
├── spec.rules[].validate # validate: if validationFailureAction: Enforce → reject
├── spec.rules[].mutate   # mutate: JSONPatch / overlays to change the object
├── spec.rules[].generate # generate: create linked resources (e.g. roles per NS)
```

`validationFailureAction: Enforce` = reject the request. `Audit` = log only (great for rollout-out).

---

## 🚀 Hands-On: Install + 4 Worked Policies

### Step 1: Install via Helm

```bash
# Add the Kyverno helm repo
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

# Namespace + install (high-availability, with metrics + policy reports)
kubectl create ns kyverno
helm install kyverno kyverno/kyverno --namespace kyverno \
  --set global.replicaCount=2 \
  --set global.imagePullPolicy=IfNotPresent

# Check controllers and the generated webhook
kubectl get pods -n kyverno -l app.kubernetes.io/name=kyverno
kubectl get validatingwebhookconfiguration | grep kyverno
# kyverno-resource-validating-webhook-cfg ...   ← generated for you!
```

### Step 2: Policy 1 — Require an image tag (validate)

```yaml
# require-image-tag.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-tag
spec:
  validationFailureAction: Enforce          # reject non-compliant
  background: false                         # don't scan existing objects
  rules:
  - name: require-tag
    match:
      any:
      - resources:
          kinds: ["Pod"]
    validate:
      message: "Images must specify a tag (e.g. nginx:1.27)"
      pattern:
        spec:
          containers:
          - image: "*:*"                    # must contain a ':'
    podSecurity: {}                          # standard validation, empty = inherit defaults
```

### Step 3: Policy 2 — Block `latest` (validate + audit)

```yaml
# block-latest.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: block-latest
spec:
  validationFailureAction: Enforce
  background: true
  rules:
  - name: block-latest-tag
    match:
      any:
      - resources:
          kinds: ["Pod"]
    validate:
      message: "The 'latest' tag is not allowed — pin a specific version"
      foreach:
      - list: "request.object.spec.containers"
        deny:
          conditions:
            any:
            - key: "{{ element.image }}"
              operator: EndsWith
              value: ":latest"
```

### Step 4: Policy 3 — No privileged containers (validate)

```yaml
# no-privileged.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: no-privileged
spec:
  validationFailureAction: Enforce
  background: true
  rules:
  - name: no-privileged-containers
    match:
      any:
      - resources:
          kinds: ["Pod"]
    validate:
      message: "Privileged containers are forbidden"
      pattern:
        spec:
          containers:
          - securityContext:
              privileged: "false"
    exclude:
      resources:
        namespaces: ["kube-system"]
```

### Step 5: Policy 4 — Add "our" label to every new Pod (mutate)

```yaml
# add-team-label.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-team-label
spec:
  rules:
  - name: inject-team-label
    match:
      any:
      - resources:
          kinds: ["Pod"]
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            team: "platform"       # applied before validation → visible to everyone
```

### Step 6: Test — the good + bad pod

```bash
# GOOD pod (pinned tag, non-privileged) → created
kubectl run good --image=nginx:1.27-alpine
# pod/good created

# BAD pod 1 (latest tag) → rejected by policy 2
kubectl run bad --image=nginx:latest
# Error from server (Forbidden): policy Block-latest ...
#   * image tag :latest is not allowed

# BAD pod 2 (privileged) → rejected by policy 3
kubectl run badpriv --image=nginx:1.27-alpine \
  --overrides='{"spec":{"containers":[{"name":"badpriv","image":"nginx:1.27-alpine","securityContext":{"privileged":true}}]}}'
# Error from server (Forbidden): policy No-privileged ...

# Confirm the label was injected on the good pod
kubectl get pod good --show-labels
# NAME   READY ... LABELS
# good   1/1   ... team=platform
```

```bash
# Preview violations without enforcing (great for rollout)
# set validationFailureAction: Audit, then:
kubectl get policyreport -A
kubectl get cpolr                                  # ClusterPolicyReport
```

---

## 🇳🇸 NIST / NSA Kubernetes Hardening Tie-in

```
NIST SP 800-190  /  NSA-CISA K8s Hardening Guide  →  Kyverno mapping
─────────────────────────────────────────────────────────────
SC-4  Resource isolation ......... policies limiting privileged
AC-6 least privilege ............... no-privileged, drop caps
SI-7 integrity .................... require pinned/immutable tags
SA-15 software supply chain ....... block latest + allowlist repos
CM-8 immutable workloads .......... mutations that add labels/resource
```

> AWS reuses the same guidance in their EKS Best Practices; Kyverno is the *operational* delivery mechanism for those documents.

---

## 🛠️ Troubleshooting

```bash
# 1. Policy silently not enforced?
kubectl get clusterpolicy <name> -o yaml | grep validationFailureAction
#   must be Enforce; Audit only reports.

# 2. Is the webhook actually registered?
kubectl get validatingwebhookconfiguration | grep -i kyverno

# 3. Namespace/exclusion not honored?
#   check `exclude.resources.namespaces` vs `namespaceSelector`
#   (kube-system often needs an explicit exclude or the policy fails)

# 4. Review last failures:
kubectl get cpolr -o yaml | less        # ClusterPolicyReport
kubectl get events -A | grep -i kyverno

# 5. Upgrade path: kyverno upgrades recreate webhooks — keep 2 replicas.
```

---

## ❓ Interview Questions

1. **How is Kyverno different from writing a ValidatingWebhookConfiguration?**
   - Kyverno *generates* and manages the webhook wiring from YAML policies; no servers, CA certs, or code maintenance. Hand-written webhooks give full control but per-policy servetops.

2. **What does `validationFailureAction: Enforce` vs `Audit` mean?**
   - Enforce rejects the API call; Audit generates policy reports and only records violations — used while rolling out a new rule.

3. **What kinds of rules does Kyverno support?**
   - validate (allow/deny), mutate (patch objects before admission), generate (create linked resources on demand), cleanup (delete stale objects).

4. **How do you avoid blocking your own cluster's system pods?**
   - Namespace `exclude` / a `namespaceSelector`, and prefer `ClusterPolicy` with explicit includes over a cluster-wide deny.

5. **What is a ClusterPolicyReport?**
   - A CRD summarizing which resources violate a given policy — wired for audit mode and consumed by dashboards like Grafana.

---

## 📚 Resources

- [Kyverno docs](https://kyverno.io/docs/introduction/)
- [Kyverno policy gallery](https://kyverno.io/policies/)
- [NSA/CISA Kubernetes Hardening Guidance](https://media.defense.gov/2021/Aug/03/2002820425/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE.PDF)
- [AWS EKS Best Practices — security](https://docs.aws.amazon.com/eks/latest/userguide/security-best-practices.html)

---

## ✅ Progress Tracker

```
Module C: Kyverno Policy Engine
  [x] Policy engine vs webhook (conceptual diagram)
  [x] Helm install with HA + metrics
  [ ] ClusterPolicy: require image tag
  [ ] ClusterPolicy: block latest + audit mode
  [ ] ClusterPolicy: no privileged
  [ ] ClusterPolicy: add label (mutate)
  [ ] Good/bad pod acceptance test
  [ ] Documented NIST/NSA mapping
```

---

## 👀 Next

- **Sandbox the pods you allowed through:** [Module B RuntimeClass & gVisor](./Module-B_RuntimeClass_gVisor.md)
- **Extend with NetworkPolicies for egress lockdown:** [Day 17 Network Policies Calico](../Day-17_Network_Policies_Calico/README.md)
- **Under the hood (the webhooks Kyverno generates):** [Module A Admission Controllers](./Module-A_Admission_Controllers.md)

---

**Remember:** A policy engine fails *closed* by default — so roll out with `Audit`, then flip to `Enforce`. 🛡️