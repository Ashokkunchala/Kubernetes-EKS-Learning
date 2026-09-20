# 🧩 Module J: Runtime Security — SecurityContext, Seccomp, AppArmor

> **Prereq:** [Day 15 EKS Security](../Day-15_EKS_Security_Best_Practices/README.md) · [Module A Admission Controllers](./Module-A_Admission_Controllers.md) · [Module B RuntimeClass & gVisor](./Module-B_RuntimeClass_gVisor.md)

Containers are not sandboxes by default. This module covers the **per-pod and per-container hardening levera**: `SecurityContext`, Linux **capabilities**, **Seccomp** profiles, and **AppArmor** — the defense-in-depth layers that sit between a normal container and a full sandbox like gVisor.

## 🎯 Learning Objectives

By the end of this module, you will understand:
- The difference between pod-level and container-level `securityContext`
- `runAsUser` / `runAsGroup` / `runAsNonRoot` / `fsGroup` and why non-root matters
- Linux capabilities: add vs drop, and why `NET_RAW`/`SYS_ADMIN` are risky
- Seccomp: `RuntimeDefault`, `Localhost` custom profiles, and how EKS defaults work
- AppArmor: how profiles confine processes and how to apply them
- `readOnlyRootFilesystem` + `allowPrivilegeEscalation`
- How these map to Pod Security Standards (`baseline` vs `restricted`)

---

## 📖 Core Concepts

### The hardening stack (defense in depth)

```
┌─────────────────────────────────────────────────────────────┐
│  LAYERS OF CONTAINER HARDENING (increasing isolation)       │
│                                                             │
│  1. SecurityContext  → user/group, read-only FS, pleescal    │
│  2. Capabilities     → drop Linux caps (NET_RAW, SYS_ADMIN)  │
│  3. Seccomp          → restrict syscalls the kernel allows   │
│  4. AppArmor/SELinux → MAC policy over files/network         │
│  5. Sandbox (gVisor) → replace the kernel (Module B)         │
└─────────────────────────────────────────────────────────────┘
```

Each layer reduces what a compromised process can do. None is a panic-button alone — combine them.

### Pod-level vs container-level securityContext

```
Pod-level securityContext  → applies to the pod AND all containers
  - runAsUser / runAsGroup / seLinuxOptions / fsGroup / supplementalGroups
  - seccompProfile (in podSecurityContext on some versions)

Container-level securityContext → overrides/extends for ONE container
  - runAsUser/runAsGroup/runAsNonRoot (override pod)
  - readOnlyRootFilesystem
  - allowPrivilegeEscalation
  - capabilities (add/drop) — container-only
  - privileged
```

> Rule: run **as non-root**, drop **all caps you don't need**, make the root filesystem **read-only**, and refuse **privilege escalation**.

### Linux capabilities — the ones that actually matter

Capabilities split root into privileges. Containers start with a default set; the dangerous ones should be dropped:

```
High-risk caps to DROP:
  SYS_ADMIN      → mount, namespace, container-escape primitives
  NET_RAW        → raw sockets (packet sniffing, ICMP)
  SYS_PTRACE     → ptrace other processes
  SETUID/SETGID  → run binaries as arbitrary users
  SYS_MODULE     → load kernel modules
Commonly SAFE to keep:
  NET_BIND_SERVICE → bind to ports < 1024
  CHOWN / DAC_OVERRIDE / FOWNER / SETGID  (image-runtime basics)
```

### Seccomp — restrict syscalls

Seccomp filters the **system calls** a process can make at the kernel boundary.

```
Seccomp profiles:
  RuntimeDefault  → the container runtime's sensible default profile
  Localhost       → a custom profile file you mount to /var/lib/kubelet/seccomp
  Unconfined      → NO restriction (default if unset — avoid in prod)
```

On EKS, `RuntimeDefault` is effectively the modern baseline; the `restricted` Pod Security Standard *requires* `RuntimeDefault`. Custom `Localhost` profiles go further for high-risk workloads.

### AppArmor — Mandatory Access Control (MAC)

AppArmor confines a process to an allowed set of paths/operations via a **profile**. Where seccomp filters syscalls, AppArmor restricts file/network/`cap` access per profile.

```
AppArmor application (on kernels/OS with AppArmor, e.g. Ubuntu):
  - profile loaded on the NODE (in /etc/apparmor.d/)
  - pod references it via an annotation (older style) or
    securityContext.appArmorProfile (newer, GA in recent k8s)
  appArmorProfile:
    type: Localhost
    localhostProfile: k8s-apparmor-example-deny-write
```

> ⚠ AppArmor support depends on the node OS/container runtime. On **Amazon Linux 2 / Bottlerocket** EKS nodes, AppArmor is generally **not** actively enforced the same way as on Ubuntu — SELinux (RHEL derivatives) or Seccomp are the practical default on EKS. Always verify enforcement on your actual node image.

### Pod Security Standards mapping

```
baseline     → drops most dangerous caps (SYS_ADMIN, etc.), no privileged,
               no host namespaces, no root escalation
restricted   → baseline + runAsNonRoot, seccomp RuntimeDefault,
               read-only root FS, drop ALL caps, no NET_RAW
```

---

## 🚀 Hands-On

### Step 1: A hardened pod — non-root, read-only FS, drop caps, no escalation

```yaml
# hardened-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened
spec:
  securityContext:                     # pod-level defaults
    runAsNonRoot: true                 # refuses to run as root
    runAsUser: 1000                    # UID 1000 (must exist in image)
    runAsGroup: 3000
    fsGroup: 2000                      # group ownership for volumes
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:1.27-alpine
    securityContext:
      readOnlyRootFilesystem: true     # / is read-only
      allowPrivilegeEscalation: false  # no setuid/exec to become root
      capabilities:
        drop: ["ALL"]                  # drop every capability
    volumeMounts:
    - name: tmp
      mountPath: /tmp                  # only writable path (for pid files)
    - name: cache
      mountPath: /var/cache/nginx
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

```bash
kubectl apply -f hardened-pod.yaml
kubectl get pod hardened -o wide
# falls back to runAsUser if image needs it — verify the container actually runs
```

### Step 2: Verify the hardening with probes

```bash
# Who are we running as?
kubectl exec hardened -- id
# uid=1000(...) gid=3000(...)  ← non-root confirmed

# Can we write to / ?  (should NOT)
kubectl exec hardened -- touch /foo
# touch: /foo: Read-only file system   ← readOnlyRootFilesystem confirmed

# Can we gain root?  (allowPrivilegeEscalation: false blocks setuid)
kubectl exec hardened -- getpcaps 1 2>/dev/null || echo "no caps / pcaps hidden"
```

### Step 3: A custom Localhost seccomp profile

```json
// /var/lib/kubelet/seccomp/deny-mount.json  (on the node)
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    { "names": ["mount", "umount2"], "action": "SCMP_ACT_ERRNO" }
  ]
}
```

```yaml
# seccomp-localhost.yaml
apiVersion: v1
kind: Pod
metadata:
  name: flat-vm
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: deny-mount.json   # filename under /var/lib/kubelet/seccomp
  containers:
  - name: app
    image: alpine
    command: ["sleep", "3600"]
```

> On **EKS** with managed nodes you must place the profile on every node (user data or a DaemonSet) — this is why most teams standardize on `RuntimeDefault` unless they have a heavy security need.

### Step 4: AppArmor (Ubuntu-style nodes only)

First, load a profile on the node, then reference it:

```bash
# On every node (Ubuntu AMIs): load the profile
cat > /etc/apparmor.d/k8s-deny-write <<'EOF'
#include <tunables/global>
profile k8s-deny-write flags=(attach_disconnected) {
  # allow everyone, but deny writes
  deny @{PROC}/* w,
  /etc/** r,
  /usr/** r,
}
EOF
sudo apparmor_parser -r /etc/apparmor.d/k8s-deny-write
```

```yaml
# apparmor-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: aadapter
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/k8s-deny-write
spec:
  containers:
  - name: app
    image: ubuntu
    command: ["sleep", "3600"]
```

```bash
kubectl get pod aadapter -o jsonpath='{.status.containerStatuses[0].state}'
# If AppArmor is loaded, the container runs confined.
```

---

## 🛠️ Troubleshooting

```bash
# 1. Pod stuck Pending / refusees to start after runAsNonRoot
kubectl describe pod <pod>
#   Events: Error: container has runAsNonRoot and image will run as root
#   → set runAsUser to a real non-root UID, or fix the image's default user.
#   → the UID MUST exist in the image's /etc/passwd for fsGroup to work.

# 2. AppArmor annotation ignored / not enforced
#   check the node OS: Amazon Linux 2 & Bottlerocket don't enforce AppArmor like Ubuntu.
#   Look for: "apparmor not enabled" in kubelet logs; fall back to seccomp + caps.

# 3. Seccomp Localhost profile not found
kubectl describe pod <pod>
#   "seccomp profile ... not found" → file must live at
#   /var/lib/kubelet/seccomp/<name> on EVERY scheduled node

# 4. App starts but can't write: readOnlyRootFilesystem
#   mount explicit emptyDir/volume mount points for /tmp, /var/run, logs,
#   or set the app's data dir via env to a writable volume.

# 5. "operation not permitted" at runtime = dropped capability
#   re-add the specific cap (e.g. NET_BIND_SERVICE) with:
#     capabilities: { add: ["NET_BIND_SERVICE"], drop: ["ALL"] }
```

---

## ❓ Interview Questions

1. **Pod-level vs container-level securityContext — what's the difference?**
   - Pod-level sets defaults for the whole pod (user/group, fsGroup, seccompProfile); container-level overrides/extends per container (readOnlyRootFilesystem, allowPrivilegeEscalation, capabilities, privileged).

2. **Why is running as non-root important, and what does `runAsNonRoot` do?**
   - A compromised root process has host-privileged reach; `runAsNonRoot: true` makes kubelet *refuse* to run the container if it would start as UID 0, enforcing least privilege.

3. **What are Linux capabilities and which should you drop?**
   - Capabilities are granular permissions split from root. Drop `ALL` and re-add only what you need; especially kill `NET_RAW`, `SYS_ADMIN`, `SYS_PTRACE`. The `restricted` standard drops all.

4. **Seccomp vs AppArmor — what does each do?**
   - Seccomp filters the **system calls** a process can make at the kernel boundary. AppArmor is a MAC policy restricting **file/network access** per profile. Both reduce attack surface; Seccomp (`RuntimeDefault`) is the configurable baseline, AppArmor is less uniformly supported (esp. on EKS AMIs).

5. **What does the `restricted` Pod Security Standard require?**
   - `runAsNonRoot: true`, seccomp `RuntimeDefault`, read-only root filesystem, drop all capabilities, no `NET_RAW`, no privileged, no host namespaces, no privilege escalation. Enforced via namespace labels (Pod Security Admission) or a policy engine.

6. **How do you enforce these cluster-wide on EKS?**
   - Label namespaces with `pod-security.kubernetes.io/enforce=restricted`, or use Kyverno/Gatekeeper constraints (Modules C & G). EKS nodes default to Seccomp/SELinux-friendly hardening; verify AppArmor support on your AMI before relying on it.

---

## 📚 Resources

- [Kubernetes — Configure a SecurityContext](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Kubernetes — Seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/)
- [Kubernetes — AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/)
- [Linux capabilities (man 7 capabilities)](https://man7.org/linux/man-pages/man7/capabilities.7.html)

---

## ✅ Progress Tracker

```
Module J: Runtime Security
  [x] Hardening stack diagram (context → caps → seccomp → apparmor → sandbox)
  [x] Pod vs container securityContext
  [ ] Hardened pod: non-root + read-only FS + drop ALL + no escalation
  [ ] Verified via `id`, `touch /`, capability probe
  [ ] Custom Localhost seccomp profile (or documented EKS constraint)
  [ ] AppArmor profile demo (on Ubuntu-style node) or documented EKS limitation
  [ ] Mapped each control to baseline/restricted
  [ ] Enforced restricted PSS on a namespace and tested a violating pod
```

---

## 👀 Next

- **Scale past per-pod hardening with cluster policy:** [Module C Kyverno](./Module-C_Kyverno_Engine.md), [Module G OPA Gatekeeper](./Module-G_OPA_Gatekeeper.md)
- **Full container isolation when per-pod hardening isn't enough:** [Module B RuntimeClass & gVisor](./Module-B_RuntimeClass_gVisor.md)
- **Recap what built-in admission enforces:** [Module A Admission Controllers](./Module-A_Admission_Controllers.md)
- **Security recap day:** [Day 15 EKS Security](../Day-15_EKS_Security_Best_Practices/README.md)

---

**Remember:** Containers share the host kernel — every layer you drop (`caps`, then `seccomp`, then `AppArmor`) shrinks what a single exploit can own. 🔒
