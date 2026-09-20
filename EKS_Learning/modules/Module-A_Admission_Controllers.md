# 🧩 Module A: Admission Controllers on EKS

> **Prereq:** [Day 15 EKS Security Best Practices](../Day-15_EKS_Security_Best_Practices/README.md)

Admission controllers are the last line of code between a request and etcd. They decide what the API server *accepts*, what it *mutates*, and what it *rejects* — before anything is persisted.

## 🎯 Learning Objectives

By the end of this module, you will understand:
- What admission controllers are and where they run
- Mutating vs validating controllers (and the ordering between them)
- The full admission flow from authentication to etcd
- The built-in `PodSecurity` admission plugin and its namespace labels
- The `PodNodeSelector` controller-manager plugin
- How to write and test your own `ValidatingWebhookConfiguration`

---

## 📖 Core Concepts

### What is an admission controller?

An **admission controller** is a piece of code that intercepts requests to the Kubernetes API server **after** authentication/authorization but **before** the object is persisted to etcd.

```
┌─────────────────────────────────────────────────────────┐
│             KUBERNETES API REQUEST FLOW                  │
│                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐            │
│  │  Client  │ → │ Authenti-│ → │ Authori- │            │
│  │ (kubectl)│   │ cation   │   │ zation   │            │
│  └──────────┘   └──────────┘   └──────────┘            │
│                                   │                     │
│                                   ▼                     │
│  ┌──────────────────────────────────────────────┐      │
│  │            ADMISSION PHASE                    │      │
│  │  ┌─────────────────┐   ┌─────────────────┐   │      │
│  │  │  MUTATING       │ → │  VALIDATING     │   │      │
│  │  │  (change obj)   │   │  (reject obj)   │   │      │
│  │  └─────────────────┘   └─────────────────┘   │      │
│  └──────────────────┬───────────────────────────┘      │
│                     ▼                                   │
│  ┌───────────┐     ┌──────────┐     ┌──────────┐      │
│  │  Schema   │ →   │  Quota   │     │  etcd    │      │
│  │  Validation│    │  checks  │     │ (persist)│      │
│  └───────────┘     └──────────┘     └──────────┘      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

Key rules:
- Controllers run **in-process** (compiled into kube-apiserver) or as **dynamic webhooks** (admission webhooks — external pods).
- Mutating runs **first**, so a validating controller sees the *mutated* object.
- Validating controllers are skipped for **DELETE** and some subresource operations.

### Mutating vs Validating

```
┌─────────────────────────────────────────────────────────┐
│       MUTATING vs VALIDATING ADMISSION                   │
│                                                         │
│  MUTATING (defaults / injection):                       │
│  - Adds default labels, annotations, resource limits     │
│  - Injects sidecars (Istio, K8s sidecar containers)      │
│  - Sets npm (NodeProjected) service-account mounts        │
│  - Can return a "patch" via JSONPatch                   │
│                                                         │
│  VALIDATING (guard rails / policy):                     │
│  - Rejects privileged containers                         │
│  - Enforces image tags / no `latest`                     │
│  - Blocks hostPath, hostNetwork, etc.                    │
│  - Cannot modify the object — only allow or deny         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🧪 Built-in: Pod Security Admission (pod-security-admission)

The OSS built-in `PodSecurity` plugin implements the **Pod Security Standards** (`privileged`, `baseline`, `restricted`) using **namespace labels** — no webhook server needed.

```bash
# Label a namespace to ENFORCE restricted (block violating pods)
kubectl label ns default pod-security.kubernetes.io/enforce=restricted

# AUDIT only — log violations, do not block
kubectl label ns default pod-security.kubernetes.io/audit=restricted

# WARN only — surface a warning to the caller
kubectl label ns default pod-security.kubernetes.io/warn=restricted

# Exempt specific users / runtime classes / namespaces
kubectl label ns kube-system pod-security.kubernetes.io/exempt=runtimeClasses:gvisor
```

```
┌─────────────────────────────────────────────────────────┐
│  POD SECURITY LEVELS                                     │
│  ┌─────────────────────────────────────────────────┐   │
│  │  privileged  → no checks (system pods)          │   │
│  │  baseline    → blocks privilege escalation,     │   │
│  │                host namespaces, added caps      │   │
│  │  restricted  → baseline + runAsNonRoot,         │   │
│  │                seccomp RuntimeDefault,          │   │
│  │                no NET_RAW cap, read-only FS     │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

> **EKS tip:** `pod-security-admission` is the only way to enforce Pod Security Standards *without installing a webhook*. On managed node groups you control `Pod Security Admission` simply via labels.

---

## 🖧 Built-in (controller-manager): PodNodeSelector

`PodNodeSelector` is shipped in **kube-controller-manager**. It injects a default nodeSelector into pods and can **lock** the nodeSelector keys a namespace is allowed to use.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod-workloads
  annotations:
    # pods here must land on Linux, arm64 node pools
    scheduler.alpha.kubernetes.io/node-selector: "node.kubernetes.io/instance-typegg=arm64"
    # lock the *keys* allowed — other keys are rejected
    pod-security.kubernetes.io/enforce: baseline
```

```yaml
# EKS node pool flag on the control plane (configure via add-on / kube-apiserver flags)
# --enable-admission-plugins=PodNodeSelector
# You cannot pass arbitrary kube-apiserver flags on EKS;
# use the EKS add-on "eks-pod-identity-agent" or OSS docs for self-managed clusters.
```

Real-world version — **use Kyverno / cluster policy instead on EKS**, because EKS does not let you toggle `--enable-admission-plugins` freely. (More in [Module C](./Module-C_Kyverno_Engine.md).)

---

## 🚀 Hands-On: A Worked ValidatingWebhookConfiguration

We'll build a validating webhook that **rejects any Pod using image tag `latest`**.

### Step 1: Webhook server (any HTTPS handler)

```go
// webhook-server/main.go — the validator logic
package main

import (
    "encoding/json"
    "io"
    "net/http"
    "strings"
)

type AdmissionReview struct {
    Request struct{ Object struct{ Spec struct{ Containers []struct{ Image string } } } } `json:"request"`
}

func main() {
    http.HandleFunc("/validate", func(w http.ResponseWriter, r *http.Request) {
        body, _ := io.ReadAll(r.Body)
        var ar AdmissionReview
        _ = json.Unmarshal(body, &ar)
        containers := ar.Request.Object.Spec.Containers

        allowed := true
        for _, c := range containers {
            if strings.HasSuffix(c.Image, ":latest") {
                allowed = false
            }
        }

        // AdmissionResponse — Kubernetes expects this exact shape
        resp := map[string]any{
            "apiVersion": "admission.k8s.io/v1",
            "kind":       "AdmissionReview",
            "response": map[string]any{
                "uid":     ar.Request.UID,      // MUST echo the request UID
                "allowed": allowed,
                "status":  map[string]string{"message": "image tag :latest is forbidden"},
            },
        }
        w.Header().Set("Content-Type", "application/json")
        _ = json.NewEncoder(w).Encode(resp)
    })
    _ = http.ListenAndServeTLS(":8443", "/certs/tls.crt", "/certs/tls.key", nil)
}
```

### Step 2: The `ValidatingWebhookConfiguration`

```yaml
# own-ca-bundle: certs signed by a CA whose bundle is embedded below
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: block.latest.images
webhooks:
- name: block-latest.examples.com
  clientConfig:
    service:
      name: webhook-server      # webhook runs inside the cluster
      namespace: webhooks
      path: /validate
    caBundle: <BASE64-CA-PEM>   # CA that signed webhook tls.crt
    # ⚠ OPTIONAL: URL-style external webhook (e.g. an AWS Lambda/API GW)
    # url: "https://block-latest.example.com/validate"
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE", "UPDATE"]
    resources: ["pods"]
  failurePolicy: Fail            # Fail → API rejects on webhook downtime
  sideEffects: None              # required: webhook cannot have side effects
  admissionReviewVersions: ["v1"]
  timeoutSeconds: 5              # API hard-caps webhook at 30s, keep it low
  namespaceSelector:
    matchExpressions:            # skip kube-system & webhooks namespaces
    - key: kubernetes.io/metadata.name
      operator: NotIn
      values: ["kube-system", "webhooks"]
```

### Step 3: Binary search — the "missing webhook" demo

Observe what every dev hits the first time:

```bash
# 1. Delete the deployment that backs a webhook: pods break admission.
kubectl delete deployment webhook-server -n webhooks

# 2. Now ANY pod create fails cluster-wide (failurePolicy: Fail):
kubectl run nginx --image=nginx:latest
# Error from server (InternalError): failed calling webhook ...
#   * the server could not find the requested resource

# 3. If the webhook target is reachable but rejects:
kubectl run bad --image=nginx:latest
# Error from server: admission webhook "block-latest.examples.com" denied the request:
#   image tag :latest is forbidden
```

### Step 4: Testing a valid pod

```bash
# Tagged image passes the webhook
kubectl run good --image=nginx:1.27-alpine
pod/good created

kubectl get validatingwebhookconfiguration block.latest.images
# NAME                  WEBHOOKS   AGE
# block.latest.images   1          5m
```

---

## 🛠️ Troubleshooting

```bash
# 1. Did my webhook even get called? Enable request logs:
kubectl logs deployment/webhook-server -n webhooks | tail -20

# 2. Is the CA bundle correct? (huge cause of x509 errors)
kubectl get validatingwebhookconfiguration block.latest.images -o yaml \
  | grep -A4 "caBundle" | head -8

# 3. Webhook name must be a valid DNS subdomain (no underscore!)
# 4. Failure mode: change failurePolicy to Ignore while you iterate
# 5. On EKS, egress from webhook pods needs the Deployment reachable by Service.
```

---

## ❓ Interview Questions

1. **What is the difference between a mutating and a validating admission controller?**
   - Mutating controllers modify the object before validation (defaults, sidecar injection, patches); validating controllers only allow or deny. Both run in-process or as webhooks; mutating runs first.

2. **Where do admission webhooks sit in the API request path?**
   - After authentication/authorization and before schema validation, quota, and persistence to etcd. Mutating webhooks fire before validating ones.

3. **How do you enforce Pod Security Standards on EKS without third-party software?**
   - Label namespaces with `pod-security.kubernetes.io/enforce=restricted` (plus `audit`/`warn`), activating the built-in `PodSecurity` admission plugin.

4. **What goes wrong when a validating webhook is down and `failurePolicy: Fail`?**
   - Every matching request is rejected cluster-wide — classic "everything broke" incident. Use `Ignore` for non-critical webhooks or add a highly-available webhook deployment.

5. **What is the difference between an admission webhook and a `MutatingWebhookConfiguration`?**
   - Same transport; the former returns an immutable allow/deny verdict, the latter returns a JSONPatch to modify the object before validation.

---

## 📚 Resources

- [Kubernetes Admission Controllers (docs)](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Dynamic Admission Control / Webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [AWS EKS — Pod Security best practices](https://docs.aws.amazon.com/eks/latest/userguide/security-best-practices.html)

---

## ✅ Progress Tracker

```
Module A: Admission Controllers
  [x] Flow diagram: authn → mutating → validating → etcd
  [x] Pod Security Admission labels (enforce / audit / warn)
  [ ] PodNodeSelector annotation demo
  [ ] Deployed a ValidatingWebhookConfiguration + tested good/bad pod
  [ ] Troubleshooting marathon: CA bundle + failurePolicy
```

---

## 👀 Next

- **Upgrade from raw webhooks → policy engine:** [Module C Kyverno](./Module-C_Kyverno_Engine.md)
- **Sandbox the pods you just let in:** [Module B RuntimeClass & gVisor](./Module-B_RuntimeClass_gVisor.md)
- **Base-layer recap:** [Day 15 EKS Security](../Day-15_EKS_Security_Best_Practices/README.md)

---

**Remember:** Everything the control plane accepts is filtered by admission — that is your permanent first defense. 🛡️