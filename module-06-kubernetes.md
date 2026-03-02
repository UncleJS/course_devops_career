# Module 06: Kubernetes

> Part of the [DevOps Career Course](./README.md) by UncleJS

---

## Table of Contents

- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [Beginner: Kubernetes Architecture](#beginner-kubernetes-architecture)
- [Beginner: Core Objects — Pods, Deployments, Services](#beginner-core-objects--pods-deployments-services)
- [Beginner: kubectl Essentials](#beginner-kubectl-essentials)
- [Beginner: Configuration — ConfigMaps & Secrets](#beginner-configuration--configmaps--secrets)
- [Intermediate: Ingress & Load Balancing](#intermediate-ingress--load-balancing)
- [Intermediate: Persistent Storage](#intermediate-persistent-storage)
- [Intermediate: Health Checks & Self-Healing](#intermediate-health-checks--self-healing)
- [Intermediate: Resource Management & Scaling](#intermediate-resource-management--scaling)
- [Intermediate: StatefulSets & DaemonSets](#intermediate-statefulsets--daemonsets)
- [Intermediate: Helm — Kubernetes Package Manager](#intermediate-helm--kubernetes-package-manager)
- [Intermediate: RBAC & Security](#intermediate-rbac--security)
- [Tools & Commands Reference](#tools--commands-reference)
- [Hands-On Labs](#hands-on-labs)
- [Further Reading](#further-reading)

---

## Overview

Kubernetes (K8s) is the industry-standard platform for running containerized applications at scale. It automates deployment, scaling, load balancing, self-healing, and rolling updates across a cluster of machines.

After learning containers in Module 05, Kubernetes is the natural next step — it answers the question: "How do I run hundreds of containers reliably in production?"

[↑ Back to TOC](#table-of-contents)

---

## Learning Objectives

By the end of this module you will be able to:

- Explain the Kubernetes control plane and worker node architecture
- Create and manage Pods, Deployments, and Services using YAML manifests
- Use `kubectl` fluently for cluster management
- Store configuration in ConfigMaps and sensitive data in Secrets
- Configure Ingress to route external HTTP traffic
- Set up persistent storage with PersistentVolumes and PersistentVolumeClaims
- Configure liveness and readiness probes for self-healing
- Set resource requests/limits and configure Horizontal Pod Autoscaler
- Use Helm to install and manage applications
- Apply RBAC to control access to cluster resources

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Kubernetes Architecture

### Control Plane (Master)

The control plane manages the cluster state. It runs on master node(s).

| Component | Role |
|---|---|
| **kube-apiserver** | The front door — all commands go through it (REST API) |
| **etcd** | Distributed key-value store — holds all cluster state |
| **kube-scheduler** | Decides which node to place each pod on |
| **kube-controller-manager** | Runs controllers that reconcile desired vs actual state |
| **cloud-controller-manager** | Integrates with cloud provider APIs (AWS, Azure, GCP) |

### Worker Nodes

Worker nodes run your application containers.

| Component | Role |
|---|---|
| **kubelet** | Agent on each node — ensures containers are running |
| **kube-proxy** | Network rules — routes traffic to pods |
| **Container runtime** | Runs containers (containerd, CRI-O) |

```
┌─────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                       │
│                                                             │
│  ┌──────────────────┐    ┌──────────┐   ┌──────────┐       │
│  │   Control Plane  │    │  Node 1  │   │  Node 2  │       │
│  │ ┌──────────────┐ │    │ ┌──────┐ │   │ ┌──────┐ │       │
│  │ │ kube-apiserver│ │    │ │ Pod  │ │   │ │ Pod  │ │       │
│  │ │ etcd         │ │    │ │ Pod  │ │   │ │ Pod  │ │       │
│  │ │ scheduler    │ │    │ └──────┘ │   │ └──────┘ │       │
│  │ │ controllers  │ │    │ kubelet  │   │ kubelet  │       │
│  │ └──────────────┘ │    └──────────┘   └──────────┘       │
│  └──────────────────┘                                       │
└─────────────────────────────────────────────────────────────┘
```

### Local Development Clusters

| Tool | Description |
|---|---|
| **minikube** | Single-node cluster in a VM or container |
| **kind** (Kubernetes in Docker) | Multi-node cluster using Docker containers as nodes |
| **k3s** | Lightweight Kubernetes — great for edge/IoT and local dev |
| **k3d** | k3s inside Docker |

```bash
# Install and start minikube
minikube start
minikube status
minikube dashboard          # Open web UI
minikube stop
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Core Objects — Pods, Deployments, Services

### Pod

The smallest deployable unit — one or more containers sharing network and storage.

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod nginx-pod
kubectl delete pod nginx-pod
```

### Deployment

Manages a set of identical pod replicas. Handles rolling updates and rollbacks.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3                       # Run 3 pods
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
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods
kubectl rollout status deployment/nginx-deployment

# Update the image (triggers rolling update)
kubectl set image deployment/nginx-deployment nginx=nginx:1.26

# Roll back to previous version
kubectl rollout undo deployment/nginx-deployment

# View rollout history
kubectl rollout history deployment/nginx-deployment
```

### Service

Exposes pods as a stable network endpoint. Pods come and go; Services stay.

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx                  # Routes to pods with this label
  ports:
  - protocol: TCP
    port: 80                    # Service port
    targetPort: 80              # Pod port
  type: ClusterIP               # Internal only (default)
```

### Service Types

| Type | Accessible From | Use Case |
|---|---|---|
| **ClusterIP** | Inside cluster only | Internal microservice communication |
| **NodePort** | Outside cluster via `NodeIP:NodePort` | Development, testing |
| **LoadBalancer** | Outside via cloud load balancer | Production on cloud providers |
| **ExternalName** | Maps to a DNS name | Accessing external services |

[↑ Back to TOC](#table-of-contents)

---

## Beginner: kubectl Essentials

```bash
# Context management
kubectl config get-contexts                # List clusters
kubectl config use-context my-cluster      # Switch cluster

# Get resources
kubectl get pods                           # Pods in default namespace
kubectl get pods -n kube-system            # Pods in specific namespace
kubectl get pods --all-namespaces          # All namespaces
kubectl get pods -o wide                   # With IP and node info
kubectl get all                            # Pods, deployments, services

# Describe (detailed info + events)
kubectl describe pod <pod-name>
kubectl describe deployment <name>
kubectl describe node <node-name>

# Apply / Delete
kubectl apply -f manifest.yaml             # Create or update
kubectl delete -f manifest.yaml            # Delete from file
kubectl delete pod <pod-name>              # Delete by name
kubectl delete pods --all                  # Delete all pods

# Logs
kubectl logs <pod-name>                    # Pod logs
kubectl logs -f <pod-name>                 # Follow logs
kubectl logs <pod-name> -c <container>     # Specific container in pod

# Exec into a pod
kubectl exec -it <pod-name> -- bash
kubectl exec -it <pod-name> -- sh          # If no bash available

# Port forwarding (development/debugging)
kubectl port-forward pod/<pod-name> 8080:80
kubectl port-forward service/<svc-name> 8080:80

# Namespace management
kubectl create namespace staging
kubectl get namespaces
kubectl config set-context --current --namespace=staging

# Quick dry-run (validate without applying)
kubectl apply -f manifest.yaml --dry-run=client
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Configuration — ConfigMaps & Secrets

### ConfigMap — Non-Sensitive Config

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres-service"
  DATABASE_PORT: "5432"
  APP_ENV: "production"
  nginx.conf: |
    server {
        listen 80;
        location / {
            proxy_pass http://app:8080;
        }
    }
```

```yaml
# Use ConfigMap in a Deployment
spec:
  containers:
  - name: app
    envFrom:
    - configMapRef:
        name: app-config          # All keys become env vars
    volumeMounts:
    - name: config-volume
        mountPath: /etc/nginx
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

### Secret — Sensitive Data

```bash
# Create from command line
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=supersecret

# Create from file
kubectl create secret generic tls-cert \
  --from-file=tls.crt=./cert.pem \
  --from-file=tls.key=./key.pem
```

```yaml
# Secret manifest (values are base64 encoded, NOT encrypted)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=          # echo -n "admin" | base64
  password: c3VwZXJzZWNyZXQ=  # echo -n "supersecret" | base64
```

> ⚠️ **Important**: Kubernetes Secrets are base64-encoded but NOT encrypted by default. Use Sealed Secrets, HashiCorp Vault, or cloud-native secret managers in production.

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Ingress & Load Balancing

Ingress manages external HTTP/HTTPS access to services inside the cluster.

```yaml
# ingress.yaml — requires an Ingress Controller (e.g., Nginx Ingress)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

```bash
# Install Nginx Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# Check ingress
kubectl get ingress
kubectl describe ingress app-ingress
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Persistent Storage

```yaml
# PersistentVolume (PV) — the actual storage resource
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /data/postgres      # For local dev only

---
# PersistentVolumeClaim (PVC) — a request for storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

```yaml
# Use PVC in a Deployment
spec:
  containers:
  - name: postgres
    image: postgres:16
    volumeMounts:
    - name: postgres-storage
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: postgres-storage
    persistentVolumeClaim:
      claimName: postgres-pvc
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Health Checks & Self-Healing

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8080

    # Liveness probe — restart container if it fails
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30     # Wait 30s before first check
      periodSeconds: 10           # Check every 10s
      failureThreshold: 3         # Restart after 3 failures

    # Readiness probe — remove from Service endpoints if fails
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 3

    # Startup probe — for slow-starting apps (disables liveness during startup)
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Resource Management & Scaling

### Resource Requests & Limits

```yaml
resources:
  requests:              # Minimum guaranteed resources
    memory: "128Mi"
    cpu: "250m"          # 250 millicores = 0.25 CPU cores
  limits:                # Maximum allowed resources
    memory: "256Mi"
    cpu: "500m"
```

### Horizontal Pod Autoscaler (HPA)

```bash
# Scale based on CPU usage
kubectl autoscale deployment nginx-deployment \
  --cpu-percent=50 \
  --min=2 \
  --max=10

kubectl get hpa
```

```yaml
# HPA manifest with custom metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### Manual Scaling

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: StatefulSets & DaemonSets

### StatefulSet — For Stateful Applications

Use for databases and anything needing stable network identity or ordered deployment.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:                # Each pod gets its own PVC
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### DaemonSet — Run on Every Node

```yaml
# Run a log collector on every node
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluentd:v1.16
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Helm — Kubernetes Package Manager

Helm packages Kubernetes manifests into reusable **charts**.

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add a chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Search for charts
helm search repo nginx
helm search hub wordpress

# Install a chart
helm install my-nginx bitnami/nginx
helm install my-nginx bitnami/nginx --set replicaCount=3
helm install my-nginx bitnami/nginx -f values.yaml      # Custom values

# List releases
helm list
helm list -A                           # All namespaces

# Upgrade a release
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# View chart values
helm show values bitnami/nginx > default-values.yaml

# Uninstall
helm uninstall my-nginx

# Create your own chart
helm create mychart
# Edit templates/ and values.yaml
helm install myapp ./mychart
helm lint ./mychart                    # Validate chart
helm template ./mychart                # Preview rendered YAML
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: RBAC & Security

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: production

---
# Role — permissions within a namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]

---
# RoleBinding — bind Role to ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-service-account
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Check what a user/service account can do
kubectl auth can-i get pods --as=system:serviceaccount:production:app-service-account
kubectl auth can-i create deployments
```

[↑ Back to TOC](#table-of-contents)

---

## Tools & Commands Reference

| Command | Purpose |
|---|---|
| `kubectl get pods/deployments/services` | List resources |
| `kubectl apply -f file.yaml` | Create or update from manifest |
| `kubectl describe <resource>` | Detailed info and events |
| `kubectl logs -f <pod>` | Follow pod logs |
| `kubectl exec -it <pod> -- bash` | Shell into pod |
| `kubectl port-forward` | Forward port for local access |
| `kubectl rollout undo` | Rollback a deployment |
| `kubectl scale deployment` | Manual scaling |
| `kubectl get hpa` | View autoscaler status |
| `helm install` / `helm upgrade` | Install/upgrade charts |
| `helm list` | List Helm releases |
| `kubectl config use-context` | Switch clusters |

[↑ Back to TOC](#table-of-contents)

---

## Hands-On Labs

### Lab 6.1 — Local Cluster Setup

1. Install `minikube` and start a cluster: `minikube start`
2. Verify: `kubectl cluster-info` and `kubectl get nodes`
3. Enable the dashboard: `minikube dashboard`

### Lab 6.2 — Deploy an Application

1. Create a Deployment manifest for `nginx:1.25` with 3 replicas
2. Apply it: `kubectl apply -f deployment.yaml`
3. Create a Service of type `NodePort`
4. Access the application via `minikube service nginx-service`
5. Update the image to `nginx:1.26` and watch the rolling update

### Lab 6.3 — ConfigMap & Secret

1. Create a ConfigMap with `APP_ENV=production`
2. Create a Secret with a fake database password
3. Create a pod that uses both as environment variables
4. Exec into the pod and verify the values are set: `env | grep APP`

### Lab 6.4 — Autoscaling

1. Deploy a resource-limited application
2. Create an HPA targeting 50% CPU
3. Use a load generator to trigger scaling: `kubectl run load --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://app-service; done"`
4. Watch the HPA scale up: `kubectl get hpa -w`
5. Stop the load and watch scale down

### Lab 6.5 — Helm Chart

1. Install the Nginx Ingress Controller via Helm
2. Install a sample application chart from Bitnami
3. Customize values with a custom `values.yaml`
4. Upgrade the release with a changed replica count

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

- [Kubernetes Official Docs](https://kubernetes.io/docs/)
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [Helm Documentation](https://helm.sh/docs/)
- [CKA Exam Curriculum](https://github.com/cncf/curriculum)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Glossary: Cluster](./glossary.md#c), [Namespace](./glossary.md#n), [Pod](./glossary.md#p), [Helm](./glossary.md#h), [RBAC](./glossary.md#r)
- **Certifications**: CKA, CKAD

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)*
