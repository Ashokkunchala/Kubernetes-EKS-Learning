# EKS Operations Runbooks

Use these as timed troubleshooting drills.

## RB-01 Pod Pending
```bash
kubectl describe pod <pod> -n <ns>
kubectl get events -n <ns> --sort-by=.lastTimestamp
kubectl get nodes
```
Branch from evidence: insufficient CPU/memory, taint mismatch, unbound PVC, or IP allocation failure.

## RB-02 Service has no endpoints
```bash
kubectl get svc <svc> -n <ns> -o yaml
kubectl get endpoints,endpointslice -n <ns>
kubectl get pods -n <ns> --show-labels
```
Check selector, Ready condition, service port, and targetPort.

## RB-03 ALB/Ingress returns 503
```bash
kubectl describe ingress <name> -n <ns>
kubectl get targetgroupbinding -A
kubectl get svc,endpointslice -n <ns>
```
Trace: client -> listener/rule -> target group -> service/endpoints -> pod -> readiness -> application port.

## RB-04 DNS failure
```bash
kubectl run dns-debug --rm -it --image=busybox:1.36 -- nslookup kubernetes.default.svc
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system logs deployment/coredns --tail=200
```
Separate cluster DNS, service discovery, upstream resolution, and node-level problems.

## RB-05 PVC / volume failure
```bash
kubectl get pvc,pv -A
kubectl describe pvc <pvc> -n <ns>
kubectl describe pod <pod> -n <ns>
kubectl get csidrivers
kubectl get volumeattachments
```

## RB-06 IAM access failure
```bash
kubectl get sa <sa> -n <ns> -o yaml
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns>
aws sts get-caller-identity
```
Verify the intended Pod Identity or IRSA mapping and policy actions. Do not broaden permissions as a diagnostic shortcut.

## RB-07 NetworkPolicy blocks traffic
```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy <name> -n <ns>
kubectl exec -it <pod> -n <ns> -- wget -qO- http://<service>:<port>/health
```
Compare actual labels, namespaces, ports, and policy selectors.

## RB-08 Node NotReady
```bash
kubectl get nodes
kubectl describe node <node>
kubectl get pods -A -o wide | grep <node>
```
Check conditions, pressure, runtime, kubelet, CNI, and recent events.

## RB-09 Rollout stuck
```bash
kubectl rollout status deployment/<deploy> -n <ns>
kubectl rollout history deployment/<deploy> -n <ns>
kubectl describe deployment/<deploy> -n <ns>
kubectl get rs,pods -n <ns>
```

## RB-10 IP exhaustion

Record subnet free IPs, instance type, ENI/IP limits, pod density, VPC CNI settings, and CNI events/logs.

## Incident report template

```text
Incident:
Customer impact:
Start/end time:
Detection:
Evidence:
Root cause:
Mitigation:
Recovery verification:
Permanent fix:
Regression test:
Owner:
```
