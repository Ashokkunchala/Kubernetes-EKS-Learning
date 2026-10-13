# 📅 Day 4: Namespaces, ConfigMaps, and Secrets

## 🎯 Learning Objectives
By the end of today, you will understand:
- What are Namespaces and how to use them
- ConfigMaps for non-confidential configuration
- Secrets for sensitive data
- How to mount ConfigMaps and Secrets in pods
- Best practices for configuration management

---

## 🧠 Plain-English terms before you start

| Term | Meaning in everyday language | Why it matters |
|---|---|---|
| **Namespace** | A logical grouping of Kubernetes objects inside one cluster. | It helps organise teams and environments and is a scope for many names and permissions; it is not a complete security boundary by itself. |
| **ConfigMap** | A Kubernetes object for non-secret configuration such as a feature flag or URL. | Configuration can change separately from the container image. |
| **Secret** | A Kubernetes object intended for sensitive values such as passwords or tokens. | Access must be restricted; base64 representation is not encryption. |
| **Environment variable** | A named value passed into a process when it starts. | Apps commonly read settings this way, but changes usually require restarting the process/Pod to take effect. |
| **Volume mount** | A path inside a container connected to storage or injected files. | It can expose ConfigMap/Secret values as files rather than environment variables. |
| **RBAC** | Role-Based Access Control: rules that decide which identity may perform which Kubernetes actions. | A user who can read Pods should not automatically be allowed to read Secrets. |

**Analogy:** A Namespace is a labelled room, a ConfigMap is a noticeboard, and a Secret is a locked envelope whose access must be controlled.

**EKS connection:** Kubernetes Secret access is governed by Kubernetes permissions and cluster configuration. For AWS-managed secrets, use an intentional integration and least-privilege IAM permissions.

**Try / observe:** `kubectl get namespaces`, `kubectl get configmaps,secrets -n <namespace>`, `kubectl describe pod <pod-name> -n <namespace>`. Avoid printing real secret values into terminals, logs, or Git.

**Troubleshooting clue:** A Pod may fail when a referenced ConfigMap/Secret name is wrong or the object is in a different Namespace. Namespaced objects must be looked up in the correct Namespace.

**Explain it back:** What belongs in a ConfigMap versus a Secret? Why does base64 not make a value secret?

---

## 📖 Core Concepts

### What are Namespaces?

**Namespaces** are a way to divide cluster resources between multiple users or teams. They provide a scope for names and can be used to apply access control.

**Use Cases:**
- Development vs Production environments
- Team separation (frontend-team, backend-team)
- Resource isolation
- Access control (RBAC)

### Namespace Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   KUBERNETES CLUSTER                     │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Namespace: development                          │   │
│  │  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │  Frontend   │  │  Backend    │              │   │
│  │  │  Pods       │  │  Pods       │              │   │
│  │  │  Services   │  │  Services   │              │   │
│  │  └─────────────┘  └─────────────┘              │   │
│  │                                                   │   │
│  │  Resource Quota:                                 │   │
│  │  - CPU: 4 cores                                  │   │
│  │  - Memory: 8Gi                                   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Namespace: production                           │   │
│  │  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │  Frontend   │  │  Backend    │              │   │
│  │  │  Pods       │  │  Pods       │              │   │
│  │  │  Services   │  │  Services   │              │   │
│  │  └─────────────┘  └─────────────┘              │   │
│  │                                                   │   │
│  │  Resource Quota:                                 │   │
│  │  - CPU: 8 cores                                  │   │
│  │  - Memory: 16Gi                                  │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Namespace: kube-system                          │   │
│  │  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │  CoreDNS    │  │  kube-proxy │              │   │
│  │  │  Dashboard  │  │  Metrics    │              │   │
│  │  └─────────────┘  └─────────────┘              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Namespace YAML Example

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: dev
    team: backend
```

---

### What are ConfigMaps?

**ConfigMaps** store non-confidential configuration data in key-value pairs. They can be used as environment variables or mounted as files.

**Use Cases:**
- Database connection strings
- Feature flags
- Application configuration
- File configurations

### ConfigMap Architecture

```
┌─────────────────────────────────────────────────────────┐
│                 CONFIGMAP ARCHITECTURE                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  ConfigMap: app-config                          │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  DATA:                                   │   │   │
│  │  │  - DATABASE_HOST: mysql-service         │   │   │
│  │  │  - DATABASE_PORT: 3306                  │   │   │
│  │  │  - LOG_LEVEL: debug                     │   │   │
│  │  │  - CACHE_TTL: 3600                      │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│         ┌───────────────┴───────────────┐              │
│         ▼                               ▼              │
│  ┌─────────────────────┐      ┌─────────────────────┐ │
│  │  Environment        │      │  Volume Mount        │ │
│  │  Variables          │      │  (File)              │ │
│  │                     │      │                     │ │
│  │  ┌───────────────┐ │      │  ┌───────────────┐  │ │
│  │  │  envFrom:     │ │      │  │  volumes:     │  │ │
│  │  │  - configMapRef│ │      │  │  - configMap  │  │ │
│  │  │    name: app   │ │      │  │  mountPath:   │  │ │
│  │  └───────────────┘ │      │  │  /etc/config  │  │ │
│  └─────────────────────┘      │  └───────────────┘  │ │
│                               └─────────────────────┘ │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### ConfigMap YAML Examples

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "mysql-service"
  DATABASE_PORT: "3306"
  LOG_LEVEL: "debug"
  CACHE_TTL: "3600"
  application.properties: |
    server.port=8080
    spring.datasource.url=jdbc:mysql://mysql-service:3306/mydb
    logging.level.root=INFO
```

### Use ConfigMap in Pod

```yaml
# pod-with-configmap.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: my-app:v1.0.0
    # Method 1: Environment variables
    envFrom:
    - configMapRef:
        name: app-config
    
    # Method 2: Individual environment variables
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_HOST
    
    # Method 3: Volume mount
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

---

### What are Secrets?

**Secrets** are Kubernetes objects intended for sensitive values (passwords, tokens, certificates). Their values are commonly represented as base64 in YAML, but base64 is not encryption. A Secret is not automatically safer than a ConfigMap unless access is restricted and storage/configuration protections are in place. Enable encryption at rest where appropriate and grant least-privilege access.

**Use Cases:**
- Database passwords
- API keys
- TLS certificates
- OAuth tokens

### Secrets Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   SECRETS ARCHITECTURE                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Secret: app-secrets                            │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  DATA (Base64 encoded):                  │   │   │
│  │  │  - DB_USERNAME: <base64-placeholder>      │   │   │
│  │  │  - DB_PASSWORD: <base64-placeholder>      │   │   │
│  │  │  - API_KEY: <base64-placeholder>           │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │                                                   │   │
│  │  ⚠️  Note: Base64 is NOT encryption!            │   │
│  │  Use encryption at rest for better security      │   │
│  └─────────────────────────────────────────────────┘   │
│                         │                               │
│         ┌───────────────┴───────────────┐              │
│         ▼                               ▼              │
│  ┌─────────────────────┐      ┌─────────────────────┐ │
│  │  Environment        │      │  Volume Mount        │ │
│  │  Variables          │      │  (File)              │ │
│  └─────────────────────┘      └─────────────────────┘ │
│                                                         │
│  Security Best Practices:                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  ✅ Enable encryption at rest (KMS)             │   │
│  │  ✅ Use RBAC to restrict access                 │   │
│  │  ✅ Rotate secrets regularly                    │   │
│  │  ✅ Use external secrets managers (AWS SM)      │   │
│  │  ❌ Never commit secrets to git                 │   │
│  │  ❌ Don't use in logs                           │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Secrets YAML Examples

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
# stringData accepts plain strings and Kubernetes encodes them when storing the Secret.
# Replace these placeholders with lab-only values; do not commit real credentials.
stringData:
  DB_USERNAME: REPLACE_WITH_LAB_USERNAME
  DB_PASSWORD: REPLACE_WITH_LAB_PASSWORD
  API_KEY: REPLACE_WITH_LAB_API_KEY
```

### Use Secret in Pod

```yaml
# pod-with-secret.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: my-app:latest
    # Method 1: Environment variables
    envFrom:
    - secretRef:
        name: app-secrets
    
    # Method 2: Individual environment variables
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: DB_PASSWORD
    
    # Method 3: Volume mount
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  
  volumes:
  - name: secret-volume
    secret:
      secretName: app-secrets
```

---

## 🛠️ Essential kubectl Commands

### Namespace Commands

```bash
# List namespaces
kubectl get namespaces
kubectl get ns  # Short form

# Create namespace
kubectl create namespace development
kubectl create namespace production

# Set default namespace
kubectl config set-context --current --namespace=development

# Get current namespace
kubectl config view --minify -o jsonpath='{..namespace}'

# Delete namespace
kubectl delete namespace development
```

### ConfigMap Commands

```bash
# Create ConfigMap
kubectl create configmap app-config --from-literal=DATABASE_HOST=mysql-service
kubectl create configmap app-config --from-file=config.properties
kubectl create configmap app-config --from-file=configs/

# List ConfigMaps
kubectl get configmaps
kubectl get cm  # Short form

# Describe ConfigMap
kubectl describe configmap app-config

# Get ConfigMap YAML
kubectl get configmap app-config -o yaml

# Delete ConfigMap
kubectl delete configmap app-config
```

### Secret Commands

```bash
# Create a lab Secret without writing the value directly in command history.
read -s -p "Lab-only DB password: " DB_PASSWORD; echo
kubectl create secret generic app-secrets --from-literal=DB_PASSWORD="$DB_PASSWORD"
unset DB_PASSWORD

# Create from a file only if that file is protected and excluded from Git.
kubectl create secret generic app-secrets-from-file --from-file=db-password.txt
kubectl create secret tls tls-secret --cert=cert.pem --key=key.pem

# Use shell variables for registry credentials; do not use a literal password.
kubectl create secret docker-registry regcred --docker-server=registry.gitlab.com --docker-username="$REGISTRY_USERNAME" --docker-password="$REGISTRY_PASSWORD"

# List Secrets
kubectl get secrets

# Describe Secret
kubectl describe secret app-secrets

# Inspect Secret metadata and keys without printing encoded values.
kubectl describe secret app-secrets
# Do not print Secret data into terminal logs or shared output.

# Decode Secret value
kubectl get secret app-secrets -o jsonpath='{.data.DB_PASSWORD}' | base64 -d

# Delete Secret
kubectl delete secret app-secrets
```

---

## 📝 Practice Exercises

### Exercise 1: Create and Use ConfigMap

```bash
# Create ConfigMap
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=mysql-service \
  --from-literal=DATABASE_PORT=3306 \
  --from-literal=LOG_LEVEL=debug

# Verify
kubectl get configmap app-config -o yaml
```

```yaml
# pod-with-configmap.yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    envFrom:
    - configMapRef:
        name: app-config
```

```bash
# Apply and test
kubectl apply -f pod-with-configmap.yaml
kubectl exec -it configmap-pod -- env | grep DATABASE
```

### Exercise 2: Create and Use Secret

```bash
# Prompt for a lab-only password without echoing it.
read -s -p "Lab-only DB password: " DB_PASSWORD; echo
kubectl create secret generic db-secrets \
  --from-literal=USERNAME=admin \
  --from-literal=PASSWORD="$DB_PASSWORD"
unset DB_PASSWORD

# Verify metadata and key names without exposing values.
kubectl describe secret db-secrets
```

```yaml
# pod-with-secret.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secrets
          key: USERNAME
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secrets
          key: PASSWORD
```

```bash
# Apply and test
kubectl apply -f pod-with-secret.yaml
kubectl exec -it secret-pod -- env | grep DB_
```

### Exercise 3: Mount ConfigMap as Volume

```yaml
# configmap-file.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server {
      listen 80;
      server_name localhost;
      
      location / {
        root /usr/share/nginx/html;
        index index.html;
      }
    }
---
# pod-with-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/conf.d/default.conf
      subPath: nginx.conf
  volumes:
  - name: nginx-config
    configMap:
      name: nginx-config
```

```bash
# Apply and test
kubectl apply -f configmap-file.yaml
kubectl exec -it nginx-pod -- cat /etc/nginx/conf.d/default.conf
```

---

## 🐛 Troubleshooting Guide

### ConfigMap/Secret Not Working

```bash
# 1. Verify ConfigMap/Secret exists
kubectl get configmap <name>
kubectl get secret <name>

# 2. Check data
kubectl get configmap <name> -o yaml
kubectl get secret <name> -o yaml

# 3. Verify pod is using it
kubectl describe pod <pod-name>

# 4. Check pod environment
kubectl exec -it <pod-name> -- env

# 5. Check mounted volume
kubectl exec -it <pod-name> -- ls /path/to/mount
```

### Secret Decoding Issues

```bash
# Decode base64 value
echo "YWRtaW4=" | base64 -d

# Or use kubectl
kubectl get secret <name> -o jsonpath='{.data.KEY}' | base64 -d
```

---

## 📝 Interview Questions

### Beginner Level

1. **What are Namespaces and why use them?**
   - **Answer:** Namespaces provide a way to divide cluster resources between multiple users or teams. They provide scope for names and can be used to apply access control and resource quotas.

2. **What is the difference between ConfigMap and Secret?**
   - **Answer:** ConfigMap is for non-sensitive configuration; Secret is intended for sensitive values. Base64 is only an encoding, not encryption. Protect Secrets with least-privilege RBAC, encryption at rest, and safe delivery/rotation practices.

3. **How are Secrets stored in Kubernetes?**
   - **Answer:** Secrets are stored in etcd as base64 encoded values. They should be encrypted at rest using KMS or external secret managers for production use.

4. **How to mount ConfigMap/Secret in a pod?**
   - **Answer:** Use `envFrom` for environment variables or `volumeMounts` with `volumes` for file mounts in the pod spec.

5. **What is the default namespace?**
   - **Answer:** The default namespace is `default`. It's used when no namespace is specified.

### Intermediate Level

6. **How to use external secrets in Kubernetes?**
   - **Answer:** Use External Secrets Operator or AWS Secrets Manager CSI Driver to sync external secrets with Kubernetes Secrets.

7. **How to enable encryption at rest for Secrets?**
   - **Answer:** Configure encryption provider in kube-apiserver or use AWS KMS for EKS clusters.

8. **What is a ServiceAccount and how does it relate to Secrets?**
   - **Answer:** ServiceAccount provides an identity for pods. Kubernetes automatically creates a Secret for each ServiceAccount containing the API token.

9. **How to rotate Secrets in Kubernetes?**
   - **Answer:** Update the Secret, then restart pods to pick up new values. Use external secret managers for automatic rotation.

10. **What are Resource Quotas and how do they work?**
    - **Answer:** Resource Quotas limit the total resource consumption per namespace. They prevent one team from consuming all cluster resources.

---

## 📚 Resources for Today

### Official Documentation
- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)

---

## 🎯 Today's Assignment

1. **Create:** A namespace called `development`
2. **Create:** A ConfigMap with application configuration
3. **Create:** A Secret with database credentials
4. **Deploy:** A pod using both ConfigMap and Secret
5. **Test:** Environment variables and volume mounts

---

## 📊 Progress Tracker

```
Day 1: What is Kubernetes?                    ✅
Day 2: Pods, ReplicaSets, Deployments         ✅
Day 3: Services and Networking                ✅
Day 4: Namespaces, ConfigMaps, Secrets        ⬜
Day 5: Persistent Storage                     ⬜
...
```

---

## 🔗 Next Day Preview

**Day 5:** Persistent Storage
- PersistentVolumes (PV) and PersistentVolumeClaims (PVC)
- StorageClasses
- Dynamic provisioning
- Hands-on practice with storage

---

**Remember:** Configuration management is crucial for maintaining applications! 🚀
