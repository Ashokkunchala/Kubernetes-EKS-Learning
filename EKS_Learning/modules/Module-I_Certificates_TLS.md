# 🧩 Module I: Certificates & TLS on EKS — cert-manager, Issuers, ACME

> **Prereq:** [Day 13 EKS Load Balancing ALB](../Day-13_EKS_Load_Balancing_ALB/README.md) · [Day 15 EKS Security](../Day-15_EKS_Security_Best_Practices/README.md)

HTTPS on EKS is easy to get wrong. This module covers the certificate machinery end-to-end: PKI concepts, TLS secrets, and **cert-manager** — the de-facto tool for automatic, rotating certificates with `Issuer`/`ClusterIssuer`, ACME/Let's Encrypt, and internal mTLS.

## 🎯 Learning Objectives

By the end of this module, you will understand:
- The TLS / PKI vocabulary: certificate, CA, private key, CSR, chain
- How EKS lets you terminate TLS (ALB/NLB, Ingress, or in-cluster)
- What `TLS Secret` actually is and how it's referenced
- Installing cert-manager and issuing certs from a ClusterIssuer
- ACME / Let's Encrypt HTTP-01 vs DNS-01 challenges
- Self-signed CA for internal services + mTLS
- Certificate rotation, renewal, and monitoring

---

## 📖 Core Concepts

### The TLS / PKI vocabulary

```
Certificate     → a signed public key binding a CN/SAN to an identity
Certificate Authority (CA) → the trust anchor that signs certs
Private key     → the secret half that proves identity (kept safe!)
CSR (Certificate Signing Request) → signed request from the requester for the CA
Chain           → leaf cert → intermediate → root CA (the "bundle")
SAN (Subject Alternative Name) → the DNS names / IPs the cert is valid for
```

```
Authenticate a service "web.default.svc":
  client ──► web (presents cert signed by CA)
   │          cert = { public key, CN=web.default.svc, SANs, Sig by CA }
   ▼
  client verifies:  CA chain trusted?  SAN matches host?  not expired?  not revoked?
```

### Where TLS terminates on EKS

```
┌─────────────────────────────────────────────────────────────┐
│  TLS TERMINATION OPTIONS ON EKS                             │
│                                                             │
│  A) ALB/NLB (AWS LB controller)  — cert on the load balancer │
│     Ingress with cert-arn annotation / TLS block            │
│  B) NGINX Ingress  — cert in a TLS Secret, terminated at     │
│     the NGINX controller pod                                │
│  C) In-cluster sidecar (istio) — mTLS between services       │
│  D) Service-level TLS (rare)     — app handles it itself     │
└─────────────────────────────────────────────────────────────┘
```

**cert-manager** issues and rotates the secrets for B, C, and inputs for A — removing the manual `openssl`/CA dance.

### The TLS Secret

A Kubernetes `Secret` of `type: kubernetes.io/tls` bundles a cert + key:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-tls
type: kubernetes.io/tls
data:
  tls.crt: <base64(public cert + chain)>
  tls.key: <base64(private key)>
# referenced by Ingress:   tls: [{ hosts: [...], secretName: my-tls }]
```

cert-manager creates these for you from an `issuerRef`, keyed on the requested `dnsNames`.

### Issuer vs ClusterIssuer

```
Issuer         → namespaced; can only issue certs in ITS namespace
ClusterIssuer  → cluster-scoped; usable from any namespace

cert-manager reconciles:
  Certificate (namespaced)
    │  issuerRef → ClusterIssuer "letsencrypt-prod"  (or local Issuer)
    ▼
  CertificateRequest → issuer provisions → stores Secret
```

### HTTP-01 vs DNS-01 ACME challenges (Let's Encrypt)

```
HTTP-01:  Let's Encrypt connects to your domain over HTTP on :80 and
          requests a token file you must serve. Requires a reachable
          HTTP endpoint + allowlist the challenge path in your Ingress.

DNS-01:   Let's Encrypt asks you to publish a TXT record proving domain
          ownership. Works without a public HTTP endpoint (internal-only
          domains), but needs a DNS provider plugin (Route53).
```

---

## 🚀 Hands-On

### Step 1: Install cert-manager

```bash
# 1. Add the Jetstack Helm repo and install
helm repo add jetstack https://charts.jetstack.io
helm repo update

kubectl create ns cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --set installCRDs=true

# 2. Verify the controllers + webhook are ready
kubectl get pods -n cert-manager            # cert-manager, -ca-injector, -webhook
kubectl get crd | grep cert-manager         # Certificates, Issuers, ClusterIssuers...
```

### Step 2: A ClusterIssuer for Let's Encrypt (production + staging)

```yaml
# clusterissuer-letsencrypt.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory   # switch to -staging for testing!
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - http01:
        ingress:
          class: alb            # your Ingress class — the controller routes the challenge
      # (or add a dns01 route53 solver for internal domains)
```

### Step 3: Issue a certificate via Ingress with annotations

```yaml
# ingress-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-tls
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod   # ← issue on-demand
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  ingressClassName: alb
  tls:
  - hosts: ["app.example.com"]
    secretName: app-example-tls          # cert-manager will create this Secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```

```bash
kubectl apply -f ingress-tls.yaml
sleep 60
kubectl get certificate        # Ready=True when issued
kubectl get secret app-example-tls -o json | jq '.type'
# "kubernetes.io/tls"
kubectl describe certificate app-example-tls
#   Events:
#   ... Certificate issued successfully
```

> **Staging vs prod:** use the Let's Encrypt **staging** server until everything works — the prod ACME server rate-limits failed issuance (5 new certs / domain / week).

### Step 4: Internal service certs with a self-signed CA

```yaml
# selfsigned-clusterissuer + ca.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}                 # issues self-signed certs (bootstrap CA only)
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: internal-ca
  secretName: internal-ca-tls     # the CA cert+key in a Secret
  duration: 8760h
  issuerRef:
    name: selfsigned
    kind: ClusterIssuer
```

```yaml
# ca-issuer.yaml   -- a CA that signs real leaf certs from the CA above
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca-issuer
spec:
  ca:
    secretName: internal-ca-tls    # references the CA Secret we just created
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: payment-svc         # certs for payment service (mTLS-able)
  namespace: payments
spec:
  secretName: payment-tls
  dnsNames: ["payment.payments.svc", "payment"]
  issuerRef:
    name: internal-ca-issuer
    kind: ClusterIssuer
```

```bash
kubectl -n payments get certificate payment-svc
# READY=True ...  → the internal CA signed leaf certs for the payments service.
```

### Step 5: Verify & monitor renewal

```bash
# Check the issued cert details
kubectl get secret payment-tls -o jsonpath='{.data.tls.crt}' | base64 -d | openssl x509 -noout -text | grep -E "Subject|Issuer|Not After"

# cert-manager auto-renews before expiry (default: 60% of duration);
# watch for renewal events
kubectl get events -A --field-selector reason=CertificateRenewed
```

---

## 🛠️ Troubleshooting

```bash
# 1. Certificate stuck "Waiting for HTTP-01 challenge"
kubectl describe certificate app-example-tls
#   - is port 80 reachable from the internet? (HTTP-01 needs it)
#   - does the Ingress allow the /.well-known/acme-challenge path?
#   - DNS record points to your ALB/NLB?

# 2. "Failed to get private key" / account errors
kubectl get clusterissuer letsencrypt-prod -o yaml
#   check email + privateKeySecretRef; sometimes re-create the account key

# 3. Rate limited by Let's Encrypt
#   switch to staging, fix the challenge, THEN go prod

# 4. Webhook unreachable (cert-manager webhook down)
kubectl logs -n cert-manager deploy/cert-manager-webhook --tail=50
#   TLS chain issues in the webhook config cause "failed calling webhook"

# 5. Internal cert not trusted by your app
#   you must install the internal CA into the app/service's trust store
#   (mount internal-ca-tls as a Secret and add to Java/Node/Go trust)
```

---

## ❓ Interview Questions

1. **What is a TLS Secret and how do Ingresses use it?**
   - A Secret of `type: kubernetes.io/tls` holding `tls.crt` (public cert + chain) and `tls.key` (private key); an Ingress references it by name so the controller can terminate HTTPS.

2. **Issuer vs ClusterIssuer?**
   - Issuer is namespaced and only issues for its own namespace; ClusterIssuer is cluster-wide and lets any namespace reference it.

3. **HTTP-01 vs DNS-01 challenge?**
   - HTTP-01 proves ownership by serving a token over port 80 at your domain; DNS-01 proves it by publishing a TXT record. DNS-01 works for internal/unreachable domains but needs a DNS provider (e.g. Route53) integration.

4. **How does cert-manager keep certificates valid?**
   - It tracks each `Certificate`'s expiry and automatically re-issues before renewal (default ~60% of the duration), replacing the `Secret`, so you never run certificates by hand.

5. **mTLS vs regular TLS?**
   - Regular TLS authenticates the *server* to the client. mTLS authenticates **both** directions — the client also presents a cert the server verifies. It's how service meshes provide service-to-service security.

6. **Why use a self-signed internal CA instead of public CA for internal services?**
   - Public CAs (Let's Encrypt) need domain control/HTTP reachability and usually charge/rate-limit for SPOF on internal-only services. An internal CA signs any internal DNS name freely; services share that CA in their trust store.

---

## 📚 Resources

- [cert-manager docs](https://cert-manager.io/docs/)
- [Let's Encrypt — how it works / rate limits](https://letsencrypt.org/how-it-works/)
- [AWS EKS — TLS with ALB / LB controller](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html)
- [Kubernetes — Ingress TLS](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls)

---

## ✅ Progress Tracker

```
Module I: Certificates & TLS
  [x] PKI vocabulary + TLS termination options diagram
  [x] Issuer vs ClusterIssuer + HTTP-01 vs DNS-01
  [ ] Installed cert-manager (Helm + CRDs)
  [ ] Created a Let's Encrypt ClusterIssuer (staging → prod)
  [ ] Issued a cert via Ingress annotation, verified the Secret
  [ ] Built a self-signed internal CA + signed leaf certs
  [ ] Explored mTLS with the internal CA
  [ ] Monitored renewal events
```

---

## 👀 Next

- **Use those certs for service-to-service security (mTLS):** [Day 18 Service Mesh Istio](../Day-18_Service_Mesh_Istio/README.md)
- **Load balancing + HTTPS on EKS:** [Day 13 EKS Load Balancing ALB](../Day-13_EKS_Load_Balancing_ALB/README.md)
- **Secure the pods behind the certs:** [Module J Runtime Security](./Module-J_Runtime_Security.md)

---

**Remember:** Certificates expire — the only safe certificate is the one that rotates itself. cert-manager makes "forever valid" obsolete. 🔐
