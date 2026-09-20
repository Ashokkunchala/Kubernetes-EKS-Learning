# 🧩 Module G: OPA Gatekeeper — Policy as Code for EKS

> **Prereq:** [Module A Admission Controllers](./Module-A_Admission_Controllers.md) · [Module C Kyverno](./Module-C_Kyverno_Engine.md) · [Day 15 EKS Security](../Day-15_EKS_Security_Best_Practices/README.md)

Kyverno writes policies in plain YAML; OPA Gatekeeper instead evaluates **Rego** — the Open Policy Agent's policy language. This module covers the other half of the policy-engine world: installing Gatekeeper on EKS, writing Rego constraints, and comparing it with Kyverno so you can pick the right engine for your org.

## 🎯 Learning Objectives

By the end of this module, you will understand:
- What OPA / Open Policy Agent is, and how Gatekeeper wires it into admission
- The 3-part Gatekeeper model: `ConstraintTemplate`, `Constraint`, `AdmissionReview`
- How Rego differs from Kyverno's YAML patterns
- 4 worked constraints: block privileged, require label, no `latest` tag, all-or-nothing replicas
- Shifting policies from a *gate* to a *data source* (rego.json input, audit)
- Gatekeeper vs Kyverno — a decision guide for EKS teams
- How Gatekeeper pairs with Argo CD, GitOps, and OpenTelemetry audit data

---

## 📖 Core Concepts

### OPA vs Gatekeeper — what's what

```
┌─────────────────────────────────────────────────────────────┐
│  Open Policy Agent (OPA) — the general-purpose engine        │
│  - evaluates policies written in REGO                        │
│  - can be embedded anywhere (API gateway, k8s, CI, app)      │
│                                                             │
│  Gatekeeper — Kubernetes-specific control plane for OPA      │
│  - runs OPA as a ValidatingWebhook (admission)              │
│  - introduces ConstraintTemplate + Constraint CRDs          │
│  - also does continuous background AUDIT + data collection  │
└─────────────────────────────────────────────────────────────┘
```

OPA is a general policy engine; Gatekeeper is the Kubernetes distribution of it. You write **Rego**, Gatekeeper turns it into admission decisions.

### The 3-piece model

```
ConstraintTemplate (ClusterPolicyTemplate)
   │  defines the schema of inputs + the REGO that will evaluate them
   ▼
Constraint (the actual policy instance)
   │  matches resources (kinds) and passes in `parameters`
   ▼
AdmissionReview  (at request time)
   │  the API server calls Gatekeeper's validating webhook with a Pod/etc.
   ▼
   allow / deny  (+ message)
```

```
┌─────────────────────────────────────────────────────────────┐
│  OPA GATEKEEPER ARCHITECTURE                                 │
│                                                             │
│  kubectl apply ... ──► kube-apiserver                        │
│                            │                                │
│                            ▼ validating webhook             │
│                      gatekeeper-controller-manager           │
│                      └► AdmissionReview (JSON)              │
│                            │                                │
│                            ▼  evaluate REGO                  │
│                      ConstraintTemplate                      │
│                      Constraint(s)                           │
│                      ┌────────────┐                         │
│                      │ OPA engine │ ──► allow / deny        │
│                      └────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 🚀 Hands-On: Install Gatekeeper + 4 Worked Constraints

### Step 1: Install via Helm (or the release manifest)

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update

kubectl create ns gatekeeper-system
helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system \
  --set replicas=3 \
  --set auditInterval=60

# Confirm webhook + audit pods
kubectl get pods -n gatekeeper-system
kubectl get validatingwebhookconfiguration | grep gatekeeper
```

### Step 2: Constraint 1 — Block privileged containers

```yaml
# ct-no-privileged.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8spspprivilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sPSPPrivilegedContainer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8spspprivilegedcontainer
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged
          msg := sprintf("privileged container <%v> is not allowed", [container.name])
        }
```

```yaml
# constraint-no-privileged.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivilegedContainer
metadata:
  name: no-privileged
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters: {}
```

```bash
kubectl apply -f ct-no-privileged.yaml -f constraint-no-privileged.yaml

# GOOD pod → passes
kubectl run good --image=nginx:1.27-alpine

# BAD pod (privileged) → denied
kubectl run bad --image=nginx:1.27-alpine \
  --overrides='{"spec":{"containers":[{"name":"bad","image":"nginx:1.27-alpine","securityContext":{"privileged":true}}]}}'
# Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request:
#   privileged container <bad> is not allowed
```

### Step 3: Constraint 2 — Require a team label (with parameters)

```yaml
# ct-required-labels.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items: { type: string }
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg}] {
          required := input.parameters.labels[_]
          provided := {label | label := input.review.object.metadata.labels[label]}
          not required in provided
          msg := sprintf("missing required label <%v>", [required])
        }
```

```yaml
# constraint-required-labels.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: team-labels
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    labels:
      - team
      - cost-center
```

```bash
kubectl apply -f ct-required-labels.yaml -f constraint-required-labels.yaml

kubectl run no-team --image=nginx:1.27-alpine          # denied: missing team
kubectl run ok --image=nginx:1.27-alpine --labels team=payments,cost-center=22  # allowed
```

### Step 4: Constraint 3 — Deny `latest` image tags

```yaml
# ct-block-latest.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sblocklatesttag
spec:
  crd:
    spec:
      names:
        kind: K8sBlockLatestTag
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sblocklatesttag
        violation[{"msg": msg}] {
          img := input.review.object.spec.containers[_].image
          endswith(img, ":latest")
          msg := sprintf("image <%v> uses forbidden latest tag", [img])
        }
```

### Step 5: Constraint 4 — Enforce a minimum replica count on Deployments

```yaml
# ct-min-replicas.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sminreplicas
spec:
  crd:
    spec:
      names:
        kind: K8sMinReplicas
      validation:
        openAPIV3Schema:
          type: object
          properties:
            min:
              type: integer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sminreplicas
        violation[{"msg": msg}] {
          min := input.parameters.min
          replicas := input.review.object.spec.replicas
          replicas < min
          msg := sprintf("Deployment must have at least %v replicas (got %v)", [min, replicas])
        }
```

```yaml
# constraint-min-replicas.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sMinReplicas
metadata:
  name: min-two-replicas
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  parameters:
    min: 2
```

```bash
kubectl apply -f ct-min-replicas.yaml -f constraint-min-replicas.yaml

# A Deployment with 1 replica → rejected
kubectl create deployment one --image=nginx:1.27-alpine --replicas=1
# Error from server (Forbidden): ... must have at least 2 replicas
```

### Step 6: Audit mode & viewing violations

```yaml
# flip any constraint to audit-first: add
#   enforcementAction: dryrun     # (valid values: deny | dryrun | warn)
```

```bash
# Even without enforcement, Gatekeeper audits existing objects:
kubectl get k8srequiredlabels.team-labels -o json | jq '.status'
# .status.violations[]  ← list of existing non-compliant resources
kubectl get constrainttemplates
kubectl get constraints
```

---

## 🧪 Gatekeeper vs Kyverno — decision guide

```
┌──────────────────────────┬──────────────────┬──────────────────┐
│                          │  Kyverno         │  OPA Gatekeeper  │
├──────────────────────────┼──────────────────┼──────────────────┤
│ Policy language          │ YAML (declarative)│ REGO (full logic)│
│ Best for                 │ Most teams,      │ Complex logic,   │
│                          │ CRUD-ish rules   │ data-driven,     │
│                          │ mutation too     │ cross-resource   │
│ Mutation                 │ ✓ built-in       │ ✗ (validate only)│
│ Learning curve           │ Low              │ Medium–High      │
│ Oracle/data sources      │ context (limited)│ rich `data.*`    │
│ Ecosystem               │ policies gallery  │ OPA library/Rego │
│ Generate resources       │ ✓ generate rules │ ✗ (not native)   │
│ When to pick             │ speed of delivery│ deep/flexible    │
│                          │                  │ policy logic     │
└──────────────────────────┴──────────────────┴──────────────────┘
```

**Rule of thumb:** If your policy is "no privileged, require tag, block latest", Kyverno gets there fastest. If you need **cross-resource data joins**, complex conditions over many objects, or you already standardize on OPA/Rego, use Gatekeeper.

---

## 🛠️ Troubleshooting

```bash
# 1. Webhook blocks EVERYTHING (bad template) — how to recover
kubectl label namespace gatekeeper-system \
  admission.gatekeeper.sh/ignore=true            # exempt the namespace
#   or temporarily delete the broken ConstraintTemplate

# 2. Constraint not firing?
kubectl get constrainttemplates -o yaml          # is REGO valid? (compiles?)
kubectl get constraints -A
#   Check `status.byPod` + `status.violations` for the compiled state

# 3. Inspection of what OPA actually received
#   add temporarily: input under a dryrun constraint then `kubectl get K8sRequiredLabels ... -o json`

# 4. Rego parse/compile errors
#   Test locally:  opa test .   (install `opa` binary)
#   `constrainttemplates.gatekeeper.sh` shows a "Rendering failure" if invalid
```

---

## 🌐 Gatekeeper in GitOps & Observability

- **GitOps:** ConstraintTemplates + Constraints belong in your Git repo, synced by [Argo CD](../Day-31_GitOps_ArgoCD_Intro/README.md) just like workloads — "policy as code" gets the same PR review, drift detection, and rollback as application code.
- **Audit data:** Gatekeeper's `status.violations` feeds compliance dashboards and can be scraped into [Prometheus](../Day-41_Prometheus_Operator/README.md) via `gatekeeper_audit_duration_seconds` etc. — a core input for a control-plane security posture view.
- **Pairing:** Many orgs run **Kyverno for mutation + Gatekeeper for deep audit**, or standardize on one. Both can be admitted alongside each other.

---

## ❓ Interview Questions

1. **What's the difference between OPA and Gatekeeper?**
   - OPA is the general policy engine that evaluates Rego anywhere; Gatekeeper is the Kubernetes distribution that runs OPA as a validating admission webhook and adds ConstraintTemplate/Constraint CRDs + background audit.

2. **How does a ConstraintTemplate differ from a Constraint?**
   - A ConstraintTemplate declares the *schema* (input parameters) and the *Rego* that runs. A Constraint is the concrete instance matched to resource kinds with actual `parameters`. This mirrors the class/object split.

3. **How is Gatekeeper different from Kyverno?**
   - Kyverno uses YAML policies (validation/mutation/generate) with a low learning curve; Gatekeeper uses the general-purpose Rego language, is validate-only, and shines for complex, data-driven rules.

4. **What happens if your Gatekeeper webhook is down with `failurePolicy: Fail`?**
   - Matching requests are rejected cluster-wide — the same single-point-of-failure risk as any validating webhook. Mitigate with high-availability pods and `failurePolicy: Ignore` for non-critical constraints.

5. **How do you roll a new constraint out safely?**
   - Start with `enforcementAction: dryrun` (or `warn`), review `status.violations`, then flip to `deny`. Keep templates in Git with ArgoCD for reviewable, reversible "policy as code."

---

## 📚 Resources

- [OPA docs — policy language (Rego)](https://www.openpolicyagent.org/docs/latest/policy-language/)
- [Gatekeeper docs](https://open-policy-agent.github.io/gatekeeper/website/docs/)
- [Constraint library (batteries included)](https://github.com/open-policy-agent/gatekeeper-library)
- [AWS EKS — Policy as code with Gatekeeper](https://docs.aws.amazon.com/eks/latest/userguide/security-best-practices.html)

---

## ✅ Progress Tracker

```
Module G: OPA Gatekeeper
  [x] OPA vs Gatekeeper conceptual diagram
  [x] ConstraintTemplate / Constraint / AdmissionReview model
  [ ] Installed Gatekeeper via Helm (HA)
  [ ] Constraint: block privileged (worked Rego)
  [ ] Constraint: required labels with parameters
  [ ] Constraint: block latest tag
  [ ] Constraint: min replicas (Deployment)
  [ ] Audit / dryrun rollout + violated the constraints on purpose
  [ ] Read the Kyverno-vs-Gatekeeper decision and picked my default
```

---

## 👀 Next

- **Pair Gatekeeper audit data with your Prometheus stack:** [Day 41 Prometheus Operator](../Day-41_Prometheus_Operator/README.md)
- **Put these policies in Git for GitOps delivery:** [Day 31 GitOps & ArgoCD Intro](../Day-31_GitOps_ArgoCD_Intro/README.md)
- **Compare with the YAML-native engine:** [Module C Kyverno](./Module-C_Kyverno_Engine.md)
- **Runtime-lock the pods you allow through:** [Module J Runtime Security](./Module-J_Runtime_Security.md)

---

**Remember:** Policy that lives only in a wiki is fiction — policy in Git, enforced by Gatekeeper, is an operating control. 🛡️
