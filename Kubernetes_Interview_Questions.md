# Kubernetes Interview Questions & Study Guide

## Overview

Kubernetes (K8s) is the industry-standard container orchestration platform. It is heavily tested in backend, DevOps, and cloud engineering interviews. This guide covers architecture, core objects, networking, storage, security, and common interview questions.

---

## Table of Contents

1. [What is Kubernetes?](#what-is-kubernetes)
2. [Architecture](#architecture)
3. [Core Objects](#core-objects)
4. [Services & Networking](#services--networking)
5. [Storage](#storage)
6. [ConfigMaps & Secrets](#configmaps--secrets)
7. [Scaling & Autoscaling](#scaling--autoscaling)
8. [Deployments & Rollouts](#deployments--rollouts)
9. [Namespaces & RBAC](#namespaces--rbac)
10. [Health Checks](#health-checks)
11. [Resource Management](#resource-management)
12. [Helm](#helm)
13. [kubectl Cheat Sheet](#kubectl-cheat-sheet)
14. [Common Interview Questions](#common-interview-questions)
15. [Quick Reference](#quick-reference)

---

## What is Kubernetes?

Kubernetes is an open-source container orchestration system that automates:
- **Deployment** of containerized applications
- **Scaling** up or down based on demand
- **Self-healing** by restarting failed containers
- **Load balancing** across healthy pods
- **Rolling updates** with zero downtime

### Kubernetes vs Docker

| | Docker | Kubernetes |
|---|---|---|
| Scope | Single container management | Multi-container orchestration |
| Scaling | Manual | Automatic (HPA) |
| Self-healing | No | Yes |
| Load Balancing | Basic | Built-in |
| Multi-host | Docker Swarm | Native |

---

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                  │
│                                                      │
│  ┌─────────────────────────────────────────────────┐ │
│  │               Control Plane (Master)            │ │
│  │  ┌────────────┐  ┌──────────────┐  ┌────────┐  │ │
│  │  │ API Server │  │  Scheduler   │  │  etcd  │  │ │
│  │  └────────────┘  └──────────────┘  └────────┘  │ │
│  │  ┌──────────────────────────────┐               │ │
│  │  │     Controller Manager       │               │ │
│  │  └──────────────────────────────┘               │ │
│  └─────────────────────────────────────────────────┘ │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │  Worker Node  │  │  Worker Node │  │ Worker Node│ │
│  │  ┌─────────┐  │  │  ┌────────┐ │  │            │ │
│  │  │ kubelet │  │  │  │ Pod(s) │ │  │            │ │
│  │  │kube-proxy│ │  │  └────────┘ │  │            │ │
│  │  │Container │ │  │  Container  │  │            │ │
│  │  │ Runtime  │ │  │  Runtime    │  │            │ │
│  │  └─────────┘  │  └────────────┘  └────────────┘ │
│  └──────────────┘                                   │
└──────────────────────────────────────────────────────┘
```

### Control Plane Components

| Component | Role |
|---|---|
| **API Server** | Entry point for all K8s commands; validates and processes REST requests |
| **etcd** | Distributed key-value store; single source of truth for cluster state |
| **Scheduler** | Assigns pods to nodes based on resource availability and constraints |
| **Controller Manager** | Runs controllers that watch cluster state and reconcile to desired state |
| **Cloud Controller Manager** | Integrates with cloud provider APIs (AWS, GCP, Azure) |

### Worker Node Components

| Component | Role |
|---|---|
| **kubelet** | Agent on each node; ensures containers in pods are running and healthy |
| **kube-proxy** | Network proxy; maintains iptables rules for Service routing |
| **Container Runtime** | Runs containers — containerd, CRI-O (Docker deprecated in K8s 1.24+) |

---

## Core Objects

### Pod

The **smallest deployable unit** in Kubernetes. A pod wraps one or more containers that share network and storage.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  containers:
    - name: app
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

> **Interview Tip**: You almost never create Pods directly. Use a Deployment, which manages pods via ReplicaSets.

### ReplicaSet

Ensures a specified number of identical pod replicas are running at all times.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-app-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: nginx:1.25
```

> Deployments manage ReplicaSets. Use Deployments, not ReplicaSets directly.

### Deployment

Manages ReplicaSets and provides declarative updates — rolling updates, rollbacks.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # max extra pods during update
      maxUnavailable: 0  # min pods that must stay available
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: my-app:v2
          ports:
            - containerPort: 8080
```

### StatefulSet

Like a Deployment but for **stateful applications** (databases, message queues). Provides:
- Stable, unique pod names (`pod-0`, `pod-1`, `pod-2`)
- Stable network identity (DNS: `pod-0.service.namespace.svc.cluster.local`)
- Ordered, graceful deployment and scaling
- Persistent storage per pod via VolumeClaimTemplates

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres"
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
          image: postgres:15
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

### DaemonSet

Ensures **one pod runs on every node** (or selected nodes). Used for:
- Log collectors (Fluentd, Filebeat)
- Monitoring agents (Prometheus Node Exporter)
- Network plugins (CNI)

### Job & CronJob

```yaml
# Job — run to completion once
apiVersion: batch/v1
kind: Job
spec:
  completions: 1
  template:
    spec:
      containers:
        - name: worker
          image: my-job:latest
      restartPolicy: OnFailure

# CronJob — scheduled Jobs
apiVersion: batch/v1
kind: CronJob
spec:
  schedule: "0 2 * * *"   # every day at 2am (cron syntax)
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cleanup
              image: my-cleanup:latest
          restartPolicy: OnFailure
```

---

## Services & Networking

A **Service** provides a stable IP and DNS name for a set of pods (pods are ephemeral; their IPs change).

### Service Types

```
ClusterIP (default)
  └── Internal IP only; accessible within the cluster
  └── Use for: internal microservice communication

NodePort
  └── Exposes service on each node's IP at a static port (30000–32767)
  └── Use for: simple external access (dev/testing)

LoadBalancer
  └── Provisions a cloud load balancer (AWS ELB, GCP LB)
  └── Use for: production external access on cloud

ExternalName
  └── Maps service to a DNS name (CNAME)
  └── Use for: accessing external services by name
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
spec:
  type: ClusterIP           # or NodePort, LoadBalancer
  selector:
    app: my-app             # routes to pods with this label
  ports:
    - protocol: TCP
      port: 80              # service port
      targetPort: 8080      # pod's container port
```

### Ingress

An **Ingress** is an API object that manages external HTTP/HTTPS access to services, providing:
- Path-based routing
- Host-based routing
- TLS termination

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: tls-secret
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

### DNS in Kubernetes

Every service gets a DNS entry:
```
<service-name>.<namespace>.svc.cluster.local

# Example
my-app.default.svc.cluster.local
postgres.database.svc.cluster.local
```

---

## Storage

### Volumes vs PersistentVolumes

```
Volume
  └── Tied to pod lifecycle — deleted when pod is deleted
  └── Types: emptyDir, hostPath, configMap, secret

PersistentVolume (PV)
  └── Cluster-wide storage resource, independent of pod lifecycle
  └── Provisioned by admin or dynamically via StorageClass

PersistentVolumeClaim (PVC)
  └── User's request for storage (size, access mode)
  └── Binds to a matching PV
```

```yaml
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce   # RWO: one node | RWX: many nodes | ROX: many nodes (read-only)
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard

# Use PVC in Pod
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
containers:
  - name: app
    volumeMounts:
      - mountPath: /data
        name: data
```

### StorageClass

Defines how storage is dynamically provisioned (AWS EBS, GCP Persistent Disk, etc.)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete      # Delete | Retain
allowVolumeExpansion: true
```

---

## ConfigMaps & Secrets

### ConfigMap — non-sensitive configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
```

```yaml
# Use as environment variables
envFrom:
  - configMapRef:
      name: app-config

# Use as volume (file)
volumes:
  - name: config-vol
    configMap:
      name: app-config
```

### Secret — sensitive configuration

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=   # base64 encoded
  DB_USER: YWRtaW4=
```

```yaml
# Use in pod
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: DB_PASSWORD
```

> **Interview Tip**: Secrets are base64 encoded, NOT encrypted by default. For real security, use external secret managers (AWS Secrets Manager, HashiCorp Vault) with External Secrets Operator.

---

## Scaling & Autoscaling

### Manual Scaling

```bash
kubectl scale deployment my-app --replicas=5
```

### Horizontal Pod Autoscaler (HPA)

Automatically scales pod replicas based on CPU/memory utilization or custom metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # scale up if avg CPU > 70%
```

### Vertical Pod Autoscaler (VPA)

Adjusts CPU/memory **requests and limits** for pods based on actual usage. Pods may need to be restarted to apply new resource values.

### Cluster Autoscaler

Automatically adds or removes **nodes** in the cluster based on pending pods and node utilization. Works with cloud provider node groups (ASGs on AWS).

---

## Deployments & Rollouts

### Deployment Strategies

```yaml
# RollingUpdate (default) — gradually replaces old pods with new
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%         # extra pods allowed during update
    maxUnavailable: 25%   # pods that can be unavailable

# Recreate — kill all old pods, then create new (downtime!)
strategy:
  type: Recreate
```

### Rollout Commands

```bash
# Check rollout status
kubectl rollout status deployment/my-app

# View rollout history
kubectl rollout history deployment/my-app

# Rollback to previous version
kubectl rollout undo deployment/my-app

# Rollback to specific revision
kubectl rollout undo deployment/my-app --to-revision=2

# Pause rollout (canary-style)
kubectl rollout pause deployment/my-app
kubectl rollout resume deployment/my-app
```

---

## Namespaces & RBAC

### Namespaces

Logical isolation within a cluster. Resources in different namespaces don't interfere.

```bash
# Common namespaces
default        # where your apps go if no namespace specified
kube-system    # K8s system components
kube-public    # publicly readable
kube-node-lease # node heartbeats

kubectl get pods -n kube-system
kubectl create namespace staging
```

### RBAC (Role-Based Access Control)

```yaml
# Role — permissions within a namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]

# RoleBinding — assign Role to a user/service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: default
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

> **ClusterRole** and **ClusterRoleBinding** apply cluster-wide (across all namespaces).

---

## Health Checks

```yaml
livenessProbe:      # Is the container alive? If not, restart it.
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10  # wait before first check
  periodSeconds: 10        # check every 10s
  failureThreshold: 3      # restart after 3 failures

readinessProbe:     # Is the container ready to receive traffic?
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
  # Pod removed from Service endpoints if probe fails (no traffic sent)

startupProbe:       # Is the container still starting up?
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30     # give 30 × 10s = 5 minutes to start
  periodSeconds: 10
  # Disables liveness/readiness until startup succeeds
```

| Probe | On Failure | Use Case |
|---|---|---|
| `livenessProbe` | Restart container | Detect deadlocks, hung processes |
| `readinessProbe` | Remove from Service | Graceful startup, temporary unavailability |
| `startupProbe` | Restart container | Slow-starting apps |

---

## Resource Management

```yaml
resources:
  requests:           # minimum guaranteed resources (used by scheduler)
    cpu: "100m"       # 100 millicores = 0.1 CPU
    memory: "128Mi"   # 128 MiB
  limits:             # maximum allowed resources
    cpu: "500m"       # 500 millicores = 0.5 CPU
    memory: "256Mi"   # 256 MiB
```

### QoS Classes

| Class | Condition | Behavior |
|---|---|---|
| **Guaranteed** | requests == limits for all containers | Last to be evicted |
| **Burstable** | requests < limits | Evicted if node is under pressure |
| **BestEffort** | No requests or limits set | First to be evicted |

### LimitRange & ResourceQuota

```yaml
# LimitRange — default limits per pod/container in a namespace
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-mem-limits
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"

# ResourceQuota — total resource cap for a namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
spec:
  hard:
    pods: "20"
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
```

---

## Helm

Helm is the **package manager for Kubernetes** — it bundles K8s manifests into reusable, versioned **charts**.

```bash
# Add a chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for a chart
helm search repo nginx

# Install a chart
helm install my-nginx bitnami/nginx

# Install with custom values
helm install my-app ./my-chart -f values.yaml --set image.tag=v2

# Upgrade a release
helm upgrade my-app ./my-chart -f values.yaml

# Rollback to previous release
helm rollback my-app 1

# List installed releases
helm list

# Uninstall
helm uninstall my-app
```

### Chart Structure

```
my-chart/
  Chart.yaml         ← chart metadata (name, version, description)
  values.yaml        ← default configuration values
  templates/
    deployment.yaml  ← templated K8s manifests
    service.yaml
    ingress.yaml
    _helpers.tpl     ← reusable template helpers
```

---

## kubectl Cheat Sheet

```bash
# --- Context & Cluster ---
kubectl config get-contexts
kubectl config use-context my-cluster
kubectl cluster-info

# --- Pods ---
kubectl get pods
kubectl get pods -n kube-system
kubectl get pods -o wide           # show node info
kubectl describe pod my-pod
kubectl logs my-pod
kubectl logs my-pod -c container   # multi-container pod
kubectl logs -f my-pod             # follow/tail logs
kubectl exec -it my-pod -- bash    # exec into pod
kubectl delete pod my-pod

# --- Deployments ---
kubectl get deployments
kubectl apply -f deployment.yaml
kubectl delete -f deployment.yaml
kubectl scale deployment my-app --replicas=5
kubectl set image deployment/my-app app=my-app:v2
kubectl rollout status deployment/my-app
kubectl rollout undo deployment/my-app

# --- Services ---
kubectl get services
kubectl expose deployment my-app --type=ClusterIP --port=80

# --- Debugging ---
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl top pods                   # CPU/memory usage (metrics-server required)
kubectl top nodes
kubectl port-forward pod/my-pod 8080:8080
kubectl run tmp --image=busybox --rm -it -- sh   # debug pod

# --- Namespaces ---
kubectl get all -n my-namespace
kubectl create namespace my-ns

# --- Output formats ---
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
```

---

## Common Interview Questions

### Q: What is the difference between a Deployment and a StatefulSet?

| | Deployment | StatefulSet |
|---|---|---|
| Pod identity | Random names (`pod-abc123`) | Stable names (`pod-0`, `pod-1`) |
| Storage | Shared or none | Dedicated PVC per pod |
| Scaling order | Arbitrary | Ordered (0 → 1 → 2) |
| Deletion order | Arbitrary | Reverse order (2 → 1 → 0) |
| Use case | Stateless apps (web, API) | Stateful apps (DB, Kafka, Zookeeper) |

---

### Q: What is the difference between a Service and an Ingress?

- **Service**: Layer 4 (TCP/UDP) load balancing within the cluster. Routes traffic to pods by label selector. ClusterIP, NodePort, or LoadBalancer.
- **Ingress**: Layer 7 (HTTP/HTTPS) routing. Routes by hostname and URL path. Requires an Ingress Controller (NGINX, Traefik). Single entry point for multiple services.

---

### Q: What happens when a pod crashes in Kubernetes?

1. **kubelet** detects the container has exited
2. K8s checks the **restartPolicy** (`Always`, `OnFailure`, `Never`)
3. Container is restarted with **exponential backoff** (10s → 20s → 40s → ... max 5min)
4. Pod shows `CrashLoopBackOff` if it keeps failing
5. **ReplicaSet** ensures desired replica count is maintained — if pod is unrecoverable, new pod is scheduled

---

### Q: What is the difference between `livenessProbe` and `readinessProbe`?

- **livenessProbe**: Checks if the container is **alive**. Failure causes K8s to **restart** the container. Use to detect deadlocks.
- **readinessProbe**: Checks if the container is **ready to serve traffic**. Failure removes the pod from **Service endpoints** (no restart). Use for startup grace periods and temporary unavailability.

---

### Q: How does Kubernetes handle rolling updates?

1. A new ReplicaSet is created for the new version
2. Pods from the new ReplicaSet are spun up (`maxSurge`)
3. Old pods are terminated (`maxUnavailable`) only after new pods pass readiness probes
4. This continues until all pods are running the new version
5. Old ReplicaSet is kept at 0 replicas (for rollback)

---

### Q: What is etcd and why is it critical?

`etcd` is a distributed key-value store that stores **all cluster state** — pod definitions, deployments, secrets, node status. If etcd is lost and has no backup, the cluster state is gone. In production, etcd runs as a cluster with 3 or 5 nodes for high availability.

---

### Q: What is a Namespace and when would you use it?

Namespaces provide **virtual clusters** within a physical cluster. Use cases:
- Separate environments: `dev`, `staging`, `production`
- Separate teams: each team gets their own namespace with RBAC
- Resource isolation: ResourceQuotas per namespace
- Avoid naming conflicts between teams

---

## Quick Reference

```
Pod            → smallest deployable unit (one or more containers)
ReplicaSet     → ensures N pod replicas are running
Deployment     → manages ReplicaSets, rolling updates, rollbacks
StatefulSet    → ordered, stable-identity pods for stateful apps
DaemonSet      → one pod per node (logging, monitoring agents)
Job            → run-to-completion workload
CronJob        → scheduled Jobs
Service        → stable IP/DNS + load balancing for pods
Ingress        → HTTP/HTTPS routing by host/path
ConfigMap      → non-sensitive config (env vars, files)
Secret         → sensitive config (base64 encoded — use vault for real security)
PV/PVC         → persistent storage independent of pod lifecycle
HPA            → auto-scale pods on CPU/memory
VPA            → auto-adjust pod resource requests/limits
Namespace      → virtual cluster for isolation
RBAC           → Role + RoleBinding | ClusterRole + ClusterRoleBinding
Helm           → K8s package manager (charts)
```

---

*Last Updated: 2026-06-04*
