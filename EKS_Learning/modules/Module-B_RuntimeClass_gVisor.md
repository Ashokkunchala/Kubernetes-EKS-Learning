# 🧩 Module B: RuntimeClass & gVisor Sandboxing

> **Prereq:** [Day 15 EKS Security Best Practices](../Day-15_EKS_Security_Best_Practices/README.md) · [Day 02 Pods & Deployments](../Day-02_Pods_ReplicaSets_Deployments/README.md)

Sometimes you cannot trust the workload. Sandboxing lets the *untrusted* pod run inside an independent user-space kernel instead of the host kernel — so a container escape does not become a host compromise.

## 🎯 Learning Objectives

By the end of this module, you will understand:
- Why untrusted workloads need more than seccomp + capabilities
- What RuntimeClass is and how it switches runtimes per-pod
- How gVisor (`runsc`) isolates workloads on an EKS worker node
- How to install gVisor on node pools and wire it to containerd
- When to pick gVisor vs Kata Containers vs Firecracker (microVMs)
- The security trade-offs (performance, compatibility, image support)

---

## 📖 Core Concepts

### The threat model

```
┌─────────────────────────────────────────────────────────┐
│       UNTRUSTED WORKLOAD → WHERE DOES IT BREAK OUT?     │
│                                                         │
│  Normal container (seccomp + caps only):                │
│  ┌───────────────────────────────┐                     │
│  │  App / User Space (in ns)     │                     │
│  └───────────────┬───────────────┘                     │
│                  │ syscalls                            │
│  ┌───────────────▼───────────────┐                     │
│  │  HOST KERNEL  (shared!)       │ ◄─ escape = game over│
│  └───────────────────────────────┘                     │
│                                                         │
│  gVisor (runsc) sandbox:                               │
│  ┌───────────────────────────────┐                     │
│  │  App / User Space             │                     │
│  └───────────────┬───────────────┘                     │
│                  │ syscalls ->                        │
│  ┌───────────────▼───────────────┐                     │
│  │  SENTRY (user-space kernel)   │  ~90% syscalls      │
│  └───────────────┬───────────────┘  intercepted        │
│                  │ 9p/gofer                    │
│  ┌───────────────▼───────────────┐                     │
│  │  HOST KERNEL                  │  minimal surface    │
│  └───────────────────────────────┘                     │
└─────────────────────────────────────────────────────────┘
```

The host kernel is the trusted computing base (TCB). gVisor moves almost all syscall handling into a user-space component (the **Sentry**), so kernel exploits die at the Sentry instead of the host.

### RuntimeClass = "switch the runtime per pod"

`RuntimeClass` is a cluster-scoped resource that tells the kubelet: *run pods that reference me through this runtime handler*.

```
┌─────────────────────────────────────────────────────────┐
│           HOW RUNTIMECLASS PLUMBS TO PODS               │
│                                                         │
│   RuntimeClass "gvisor"                                  │
│   └─ handler: runsc                                     │
│        └── kubelet → CRI (containerd)                    │
│              └── containerd runtime "runsc" (gVisor)     │
│                                                         │
│   Pod with runtimeClassName: gvisor                      │
│   └─ scheduled to node → kubelet sees the class          │
│        └─ starts pod via containerd runsc handler        │
│              └─ whole pod inside a Sandbox (Sentry)      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

Note: on most nodes **containerd** and **runc** remain the default; `RuntimeClass` simply selects the alternative handler for a specific pod.

---

## 🚀 Hands-On: gVisor on an EKS Worker Node

### Step 1: Install the `runsc` binary on the node (Managed Node Group)

```bash
# EKS AMI bootstrap — run this as USER DATA on the node group.
# It fetches gVisor, installs it to /usr/local/bin, and logs a marker.
#!/bin/bash
set -euo pipefail

# 1. Fetch the runsc binary (same version as gVisor release)
curl -fsSL -o /tmp/runsc \
  https://storage.googleapis.com/gvisor/releases/release/latest/x86_64/runsc
chmod +x /tmp/runsc
mv /tmp/runsc /usr/local/bin/runsc

# 2. Record the install for kubelet verification
cat > /etc/gvisor-version <<EOF
gvisor installed $(runsc --version)
EOF

# 3. On EKS managed node groups, user data runs before kubelet join.
#    For self-managed nodes run `sudo /usr/local/bin/runsc install` then
#    restart containerd  →  registers the runsc runtime with containerd.
```

### Step 2: Register the `runsc` runtime in containerd

```toml
# /etc/containerd/config.toml  — minimum gVisor runtime entry
version = 2

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
  # default runc stays untouched
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"

  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
    runtime_type = "io.containerd.runsc.v1"
    # path to the runsc shim binary
    binary_path = "/usr/local/bin/runsc"
    # example: restrict to 2 CPU · disable direct file access (security!)
    runtime_path = "/dev/null"    # platform device for vfs2gvisor
    privileged_without_host_devices = false
```

```bash
# Mark container start by restarting containerd on the node
sudo systemctl restart containerd
sudo ctr plugins ls | grep runsc   # confirm the runtime is loaded
```

### Step 3: Create the RuntimeClass

```yaml
# runtimeclass-gvisor.yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor            # name pods will reference
handler: runsc            # MUST match the containerd runtimes key
```

### Step 4: Reference it from a Pod

```yaml
# untrusted-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: untrusted-app
  labels:
    app: sandboxed
    pod-security.kubernetes.io/enforce: baseline   # reuse Module A!
spec:
  runtimeClassName: gvisor          # ← the switch
  containers:
  - name: app
    image: nginx:1.27-alpine
    resources:
      limits:
        cpu: "500m"
        memory: "256Mi"
```

```bash
# Verify the pod is actually running under runsc
kubectl get pod untrusted-app -o jsonpath='{.status.runtimeClassName}'
# gvisor

kubectl exec untrusted-app -- cat /proc/version
# Linux version ... (gVisor reports its own kernel running)
```

---

## 🧪 When to use gVisor vs Kata vs Firecracker

```
┌─────────────────────────────────────────────────────────┐
│  RUNTIME               KERNEL      ISOLATION     COST   │
│                                                         │
│  runc (default)        host        namespaces     ~0     │
│  gVisor (Sentry)       own         user-space     syscall│
│                        (user-space)kernel         overhead│
│  Kata Containers       real VM     full hardware   ~1s   │
│                        guest kernel+ hardware      boot  │
│  Firecracker (microVM) real VM     KVM, LZ        low-ms│
│                        guest kernel+ fast boot    boot   │
└─────────────────────────────────────────────────────────┘
```

| Decision | gVisor | Kata Containers | Firecracker |
|---|---|---|---|
| Hardware acceleration | ✗ (pure user-space) | ✓ (VM) | ✓ (KVM) |
| Real kernel time / time sync | clocks virtualized | full | full |
| Compatibility with OCI images | high (GPU needs passthrough) | full | full |
| Startup time | ~ms (same as runc) | seconds (boot kernel) | ~100ms–1s |
| Best fit on EKS | multi-tenant/public workloads, CI runners | workloads needing full kernel feats | serverless-ish isolated functions |
| Node requirement | any AMI (no special caps) | needs virtualized CPUs (KVM/Nitro) | needs KVM |

> On **EKS**, Nitro-based instances make Kata/Firecracker trivial to run, but gVisor is the only one that needs **zero hypervisor setup** and integrates via a plain containerd runtime handler — which is why most EKS sandboxing blueprints start with gVisor.

---

## 🔐 Security trade-offs (remember these)

```
ADVANTAGES:
+ syscall surface shrinks from ~300+ to a hardened subset
+ even a real kernel CVE cannot "see" the app directly
+ per-pod choice — trusted pods keep runc performance

DISADVANTAGES / GOTCHAS:
- performance: ~20–50% slower on syscall-heavy apps
  (I/O hits the middleware layer; straight xv6-style workloads suffer)
- not a full VM: Hypervisor=no, host kernel still handles a few paths (Filesystem via gofer)
- some syscalls unsupported → app must be "sandbox-aware"
- gVisor does NOT protect against host-kernel bugs in the few passthrough paths
- shared /tmp via emptyDir is fine, but hostPath + gVisor is restricted
```

```bash
# Performance sanity check you should run in your own benchmark:
kubectl exec -it untrusted-app -- \
  sh -c 'time dd if=/dev/zero of=/tmp/test bs=1M count=200 oflag=direct'
# Compare against the SAME pod without runtimeClassName.
```

---

## 🛠️ Troubleshooting

```bash
# 1. Pod stuck in "ContainerCreating" after adding RuntimeClass
kubectl describe pod untrusted-app
#   │ Error: rpc error: code = Unknown desc = runtime "runsc"
#   │              not found in runtime options
#   → containerd entry missing or handler name mismatch

# 2. runsc binary missing on node (user data script failed)
kubectl get nodes -o wide; ssh <node>
ls -la /usr/local/bin/runsc         # empty = user data expired

# 3. Pod scheduled to the WRONG node (mixed node groups)
kubectl get pod untrusted-app -o wide
#   add a nodeSelector: gvisor: "true" in the pod spec

# 4. EKS add-on "containerd" keeps overwriting config.toml
#   → put gVisor runtime config under the node group user data AFTER containerd restart
```

---

## ❓ Interview Questions

1. **What is RuntimeClass in Kubernetes?**
   - A cluster-scoped resource that selects a CRI runtime handler per pod; `spec.runtimeClassName` tells the kubelet which container runtime to boot the pod with.

2. **How does gVisor isolate workloads compared to a normal container?**
   - gVisor runs the pod against its own user-space kernel (the Sentry); most syscalls never hit the host kernel, drastically shrinking the kernel-exploitation surface.

3. **gVisor vs Kata Containers — when would you pick each?**
   - gVisor: no hypervisor setup, ms startup, best for CPU-bound multi-tenant workloads. Kata: full hardware-virtualized VM + full syscall compatibility, needed when apps require kernels features gVisor does not implement; costs a kernel-boot latency.

4. **Why isn't seccomp + dropping capabilities enough for untrusted workloads?**
   - They reduce but do not eliminate the host-kernel attack surface; a bug in an allowed syscall remains reachable. Sandboxing replaces the kernel entirely for the workload.

5. **How do you make gVisor available on an EKS node group?**
   - Inject user data that installs `runsc`, add a `runsc` runtime entry to containerd config, restart containerd, then create a RuntimeClass with `handler: runsc`.

---

## 📚 Resources

- [RuntimeClass docs](https://kubernetes.io/docs/concepts/containers/runtime-class/)
- [gVisor containerd quick start](https://gvisor.dev/docs/user_guide/containerd/quick_start/)
- [Kata Containers on EKS (blueprint)](https://aws.github.io/aws-eks-best-practices/security/docs/sandboxing/)
- [AWS Sandboxing workloads best practices](https://docs.aws.amazon.com/eks/latest/userguide/sandbox.html)

---

## ✅ Progress Tracker

```
Module B: RuntimeClass & gVisor
  [x] Threat model diagram drawn
  [x] RuntimeClass → containerd → runsc plumbing explained
  [ ] Installed runsc via EKS node user data
  [ ] Created RuntimeClass + sandboxed Pod
  [ ] Ran a bad-and-good pod side-by-side
  [ ] Performance benchmark vs runc recorded
```

---

## 👀 Next

- **Policy-engine the sandboxing rule (block pods *without* a runtimeClass):** [Module C Kyverno](./Module-C_Kyverno_Engine.md)
- **Allowlist the gvisor RuntimeClass in Pod Security Admission:** [Module A Admission Controllers](./Module-A_Admission_Controllers.md)
- **Mini-PSA recap:** [Day 15 EKS Security](../Day-15_EKS_Security_Best_Practices/README.md)

---

**Remember:** Proximity is privilege — if you cannot trust the code, do not run it on the host kernel. 🔒