# 📅 Day 6: ConfigMaps and Secrets in Practice

## 🎯 Learning Objectives
By the end of today, you will understand:
- Real-world ConfigMap and Secret usage patterns
- Environment variable injection
- Volume mounting strategies
- Best practices for configuration management
- Security considerations

---

## 🧠 Plain-English terms before you start

| Term | Everyday meaning | Why it matters |
|---|---|---|
| **Configuration** | Settings that control how an app behaves, such as a log level or endpoint. | It should usually be changed without rebuilding the app image. |
| **ConfigMap** | Kubernetes storage for non-sensitive configuration. | It lets a Pod consume settings as environment variables or files. |
| **Secret** | Kubernetes storage designed for sensitive values. | Only authorised identities and workloads should be able to read it. |
| **Base64** | A way to represent bytes as text, not a protection mechanism. | Anyone who obtains a base64 value can decode it. |
| **Environment variable** | A key/value passed to the running process. | Many apps read configuration this way at startup. |
| **Least privilege** | Give each person or workload only the access it needs. | It limits damage if a credential or Pod is compromised. |

**Analogy:** A ConfigMap is a public instruction sheet; a Secret is a locked envelope. Base64 is like writing the envelope contents in a different alphabet—not locking it.

**EKS connection:** For production credentials, consider AWS Secrets Manager or Parameter Store integration, encryption at rest, restricted IAM/RBAC, and rotation.

**Try / observe:** `kubectl describe pod <pod-name>`, `kubectl get configmap <name> -o yaml`. Do not run commands that print real Secret data into shared logs or commit secret values to Git.

**Troubleshooting clue:** If a setting is missing, verify the object name, Namespace, key name, and whether the Pod uses an environment variable or mounted file. A running process may need a restart to read updated environment values.

**Explain it back:** Why is base64 not encryption? How would you restrict which workload can read a Secret?

---

## 📖 Core Concepts

### ConfigMap Creation Methods

```
┌─────────────────────────────────────────────────────────┐
│              CONFIGMAP CREATION METHODS                   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  1. From Literal Values                         │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  kubectl create configmap my-config \    │   │   │
│  │  │    --from-literal=key1=value1 \          │   │   │
│  │  │    --from-literal=key2=value2            │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  2. From File                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  kubectl create configmap my-config \    │   │   │
│  │  │    --from-file=config.properties         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  3. From Directory                              │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  kubectl create configmap my-config \    │   │   │
│  │  │    --from-file=configs/                  │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  4. From YAML                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  kubectl apply -f configmap.yaml         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Secret Creation Methods

```
┌─────────────────────────────────────────────────────────┐
│                SECRET CREATION METHODS                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  1. From Literal Values                         │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  kubectl create secret generic my-secret│   │   │
│  │  │    --from-literal=username=admin \      │   │   │
│  │  │    --from-literal=password=<lab-only-value>    │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  2. From File                                   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  kubectl create secret generic my-secret│   │   │
│  │  │    --from-file=db-password.txt          │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  3. TLS Secret                                  │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  kubectl create secret tls my-tls \     │   │   │
│  │  │    --cert=cert.pem \                     │   │   │
│  │  │    --key=key.pem                         │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  4. Docker Registry Secret                      │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  kubectl create secret docker-registry \ │   │   │
│  │  │    my-regcred \                          │   │   │
│  │  │    --docker-server=registry.gitlab.com \ │   │   │
│  │  │    --docker-username=user \              │   │   │
│  │  │    --docker-password=<token-placeholder>  │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🔧 Real-World Examples

### Example 1: Database Configuration

```yaml
# database-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: database-config
data:
  DB_HOST: "mysql-service"
  DB_PORT: "3306"
  DB_NAME: "ecommerce"
  DB_MAX_CONNECTIONS: "100"
  DB_TIMEOUT: "30"
  application.properties: |
    spring.datasource.url=jdbc:mysql://mysql-service:3306/ecommerce
    spring.datasource.hikari.maximum-pool-size=100
    spring.datasource.hikari.connection-timeout=30000
    spring.jpa.hibernate.ddl-auto=validate
---
# database-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-secrets
type: Opaque
# stringData accepts plain strings; replace placeholders with lab-only values.
stringData:
  DB_USERNAME: REPLACE_WITH_LAB_USERNAME
  DB_PASSWORD: REPLACE_WITH_LAB_PASSWORD
---
# database-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: my-backend:latest
        envFrom:
        - configMapRef:
            name: database-config
        - secretRef:
            name: database-secrets
        volumeMounts:
        - name: app-config
          mountPath: /app/config
          readOnly: true
      volumes:
      - name: app-config
        configMap:
          name: database-config
          items:
          - key: application.properties
            path: application.properties
```

### Example 2: Nginx Configuration

```yaml
# nginx-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    worker_processes auto;
    events {
        worker_connections 1024;
    }
    http {
        server {
            listen 80;
            server_name localhost;
            
            location / {
                root /usr/share/nginx/html;
                index index.html;
            }
            
            location /api {
                proxy_pass http://backend-service:8080;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
            }
        }
    }
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>My App</title>
    </head>
    <body>
        <h1>Welcome to My Application</h1>
    </body>
    </html>
---
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: nginx-config
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
```

### Example 3: TLS Certificate

```yaml
# tls-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
---
# ingress-with-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

---

## 🔒 Security Best Practices

```
┌─────────────────────────────────────────────────────────┐
│              CONFIGURATION SECURITY                      │
│                                                         │
│  ✅ Do:                                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │  • Use Secrets for sensitive data                │   │
│  │  • Enable encryption at rest                     │   │
│  │  • Use RBAC to restrict access                   │   │
│  │  • Rotate secrets regularly                      │   │
│  │  • Use external secret managers (AWS SM)         │   │
│  │  • Mount secrets as volumes (not env vars)       │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ❌ Don't:                                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │  • Commit secrets to git                        │   │
│  │  • Log secrets                                  │   │
│  │  • Use base64 as encryption                     │   │
│  │  • Store secrets in ConfigMaps                  │   │
│  │  • Share secrets across namespaces              │   │
│  │  • Use default service accounts                 │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🛠️ Practice Exercises

### Exercise 1: Multi-Container Pod with ConfigMaps

```yaml
# multi-container-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  config.json: |
    {
      "database": {
        "host": "mysql-service",
        "port": 3306
      },
      "cache": {
        "host": "redis-service",
        "port": 6379
      }
    }
---
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: app
    image: my-app:latest
    envFrom:
    - configMapRef:
        name: app-config
  - name: sidecar
    image: log-sidecar:latest
    volumeMounts:
    - name: config
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: app-config
```

### Exercise 2: Secret with Volume Mount

```yaml
# secret-volume.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: REPLACE_WITH_LAB_USERNAME
  password: REPLACE_WITH_LAB_PASSWORD
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: my-app:latest
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
      items:
      - key: username
        path: db-username
      - key: password
        path: db-password
```

---

## 📝 Interview Questions

1. **How do you manage configuration across environments?**
   - **Answer:** Use separate ConfigMaps/Secrets per environment (dev, staging, prod). Use Kustomize or Helm for templating. Store in Git and use GitOps for deployment.

2. **What is the difference between envFrom and env?**
   - **Answer:** `envFrom` imports all keys from ConfigMap/Secret as environment variables. `env` allows selecting specific keys or setting custom names.

3. **How do you update ConfigMap/Secret without restarting pods?**
   - **Answer:** For environment variables, pods must be restarted. For volume mounts, updates propagate automatically (with a delay).

4. **What are the security implications of using Secrets?**
   - **Answer:** Secrets are only base64 encoded, not encrypted. Enable encryption at rest, use RBAC, and consider external secret managers for production.

5. **How do you test applications with different configurations?**
   - **Answer:** Create different ConfigMaps for each test scenario. Use environment variables or volume mounts to inject configuration.

---

## 📚 Resources

- [ConfigMap Best Practices](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secret Best Practices](https://kubernetes.io/docs/concepts/configuration/secret/)

---

## 🎯 Today's Assignment

1. **Create:** A ConfigMap with multiple configuration files
2. **Create:** A Secret with database credentials
3. **Deploy:** An application using both ConfigMap and Secret
4. **Test:** Configuration updates and data persistence

---

## 📊 Progress Tracker

```
Day 1-5: ✅ Complete
Day 6: ConfigMaps and Secrets Practice ⬜
Day 7: Kubernetes Networking Deep Dive ⬜
...
```

---

## 🔗 Next Day Preview

**Day 7:** Kubernetes Networking Deep Dive
- Pod-to-Pod communication
- Network Policies
- DNS in Kubernetes
- Hands-on networking exercises

---

**Remember:** Proper configuration management is crucial for production applications! 🚀
