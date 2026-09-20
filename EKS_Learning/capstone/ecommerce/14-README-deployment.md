# 🚀 E-Commerce Capstone — Deployment Runbook

Everything you need to deploy, verify, and clean up the `ecommerce` microservices stack on EKS. Work top to bottom — each file has a `#` header naming its service and the day(s) it proves.

## 📦 Order of Operations

Apply strictly in this order. Files 00–07 are self-contained in the `ecommerce` namespace; 08+ attach platform add-ons (ALB, RBAC, HPA, GitOps, monitoring, backup).

```bash
# 1 — Foundations
kubectl apply -f ecommerce/00-namespace.yaml      # namespace ecommerce
kubectl apply -f ecommerce/01-configmap.yaml      # app-config ConfigMap
kubectl apply -f ecommerce/02-secret.yaml         # db-secret Opaque secret

# 2 — Workloads
kubectl apply -f ecommerce/03-frontend.yaml       # nginx frontend + Service
kubectl apply -f ecommerce/04-product-service.yaml # 3 replicas, HTTP+metrics
kubectl apply -f ecommerce/05-order-service.yaml  # 2 replicas
kubectl apply -f ecommerce/06-user-service.yaml   # 2 replicas
kubectl apply -f ecommerce/07-payment-service.yaml # 1 replica

# 3 — Ingress
kubectl apply -f ecommerce/08-ingress.yaml        # ALB, *.example.com paths

# 4 — Hardening & platform
kubectl apply -f ecommerce/09-rbac.yaml           # read-only ops Role
kubectl apply -f ecommerce/10-hpa-pdb.yaml        # HPA cpu 60% + PDB
kubectl apply -f ecommerce/11-argocd-app.yaml     # GitOps Application
kubectl apply -f ecommerce/12-monitoring.yaml     # ServiceMonitor + alerts
kubectl apply -f ecommerce/13-velero-schedule.yaml # daily backups
```

> If you deploy the ArgoCD `Application` (11), ArgoCD manages everything in the `ecommerce/` path from your git repo. For a pure manual run, skip 11.

## ✔️ Verification Checklist

```bash
# Everything present and healthy
kubectl get ns ecommerce
kubectl get deploy,svc,pods -n ecommerce -o wide

# Expected:
#   frontend 2/2 · product 3/3 · order 2/2 · user 2/2 · payment 1/1
#   all pods READY 1/1, service endpoints populated
kubectl get endpoints -n ecommerce

# Probes passing
kubectl top pods -n ecommerce

# Ingress provisioned (wait for ADDRESS to become an ALB DNS name)
kubectl get ingress -n ecommerce -w
kubectl describe ingress ecommerce-ingress -n ecommerce

# HPA observing metrics (needs Metrics Server)
kubectl get hpa -n ecommerce

# RBAC works as expected (ops-reader CAN read, cannot write)
kubectl auth can-i --as=system:serviceaccount:ecommerce:ops-reader \
  get pods -n ecommerce          # → yes
kubectl auth can-i --as=system:serviceaccount:ecommerce:ops-reader \
  delete pods -n ecommerce        # → no

# Monitoring targets discovered
kubectl get servicemonitor,prometheusrule -n ecommerce

# Backup schedule loaded
kubectl get schedules.velero.io -n velero
```

**Smoke test traffic** (once ALB is up):

```bash
ALB=$(kubectl get ingress -n ecommerce ecommerce-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -v -H "Host: shop.example.com" http://$ALB/                             # frontend → 200
curl -H "Host: shop.example.com" http://$ALB/api/products/                   # product  → 200
curl -H "Host: shop.example.com" http://$ALB/api/orders/                     # order    → 200
curl -H "Host: shop.example.com" http://$ALB/api/users/                      # user     → 200
curl -H "Host: shop.example.com" http://$ALB/api/payments/                   # payment  → 200
```

Expected: every route returns `200` (or the app's health JSON) — not `404`.

## 🧹 Cleanup

```bash
# 1. Remove the ArgoCD Application first so it won't re-create the namespace
kubectl delete -f ecommerce/11-argocd-app.yaml

# 2. Remove the whole namespace (takes down every service + its resources)
kubectl delete ns ecommerce

# 3. Platform leftovers (optional but tidy)
kubectl delete -f ecommerce/13-velero-schedule.yaml
kubectl delete -f ecommerce/12-monitoring.yaml        # only if metrics lived in ecommerce ns
kubectl delete pvc,pv --all -n ecommerce 2>/dev/null   # if namespace delete raced storage

# 4. Down the entire cluster (day-confirmed teardown)
eksctl delete cluster --region <region> --name <cluster>
```

## 🎓 Skills Proven

| Signal | File(s) |
|--------|---------|
| Namespace hygiene, ConfigMap/Secret injection | 00, 01, 02 |
| Deployment + Service + probes + resources | 03–07 |
| ALB path routing | 08 |
| RBAC least privilege | 09 |
| Autoscaling + PDB | 10 |
| GitOps (ArgoCD) | 11 |
| ServiceMonitors, alerts, Grafana datasources | 12 |
| Velero daily backup | 13 |