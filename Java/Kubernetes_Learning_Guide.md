# Kubernetes Learning Guide — From Zero to Your First Deployment

> **A hands-on path to learn Kubernetes** built for a junior Java backend developer. Read top to bottom — each section builds on the last. Type every command yourself.

---

## Table of Contents

1. [Why Kubernetes Exists (The Problem)](#1-why-kubernetes-exists-the-problem)
2. [Mental Model: What Kubernetes Actually Does](#2-mental-model-what-kubernetes-actually-does)
3. [Set Up a Local Playground](#3-set-up-a-local-playground)
4. [The Smallest Unit: Pods](#4-the-smallest-unit-pods)
5. [Don't Use Pods Directly: Deployments](#5-dont-use-pods-directly-deployments)
6. [Talking to Your Pods: Services](#6-talking-to-your-pods-services)
7. [Configuration: ConfigMaps & Secrets](#7-configuration-configmaps--secrets)
8. [Keeping Pods Healthy: Probes](#8-keeping-pods-healthy-probes)
9. [Giving Pods Resources: Requests & Limits](#9-giving-pods-resources-requests--limits)
10. [Scaling Your App](#10-scaling-your-app)
11. [Storage That Survives Restarts](#11-storage-that-survives-restarts)
12. [Full Worked Example: Deploy a Spring Boot App](#12-full-worked-example-deploy-a-spring-boot-app)
13. [Debugging When Things Break](#13-debugging-when-things-break)
14. [Your 4-Week Learning Path](#14-your-4-week-learning-path)
15. [Quick Reference Cheat Sheet](#15-quick-reference-cheat-sheet)

---

## 1. Why Kubernetes Exists (The Problem)

Running containers by hand doesn't scale. Containers crash, traffic spikes, servers die — doing this across 50 containers on 10 servers is impossible for a human.

| Problem | What Kubernetes does |
|---------|----------------------|
| Container crashes | **Self-healing** — restarts it automatically |
| Traffic spike | **Auto-scaling** — adds more copies |
| New version | **Rolling update** — zero-downtime swaps |
| Server dies | **Rescheduling** — moves containers to a healthy node |
| Many copies | **Load balancing** — spreads requests across them |

> "K8s" = **K** + 8 letters + **s**.

---

## 2. Mental Model: What Kubernetes Actually Does

The core idea is **desired state**: you declare *what you want*, and Kubernetes works continuously to make reality match.

> Think of it like a thermostat — you set 22°C and it handles the heating/cooling forever. You don't micromanage.

```
You declare:   "I want 3 replicas of my-app:v2 running"
                           │
                           ▼
       ┌───────────────────────────────────┐
       │   Kubernetes Control Plane          │
       │   Desired: 3 pods   Actual: 2 pods  │
       │            └──────► starts 1 more   │
       └───────────────────────────────────┘
```

This **reconciliation loop** — observe → compare → act → repeat — is how almost everything in Kubernetes works.

**The cluster has two kinds of machines:**

- **Control Plane (the brain)** — API Server, Scheduler, etcd (source of truth), Controllers.
- **Worker Nodes (the muscle)** — run your containers via `kubelet` + container runtime.

You talk to the API Server using **`kubectl`** ("kube control").

---

## 3. Set Up a Local Playground

**Option A — `minikube` (recommended):**

```bash
minikube start                 # boots a single-node cluster
kubectl get nodes              # confirm it's Ready
kubectl version --short        # check client + server versions
```

**Option B — Docker Desktop:** Settings → Kubernetes → "Enable Kubernetes".

> `kubectl get <thing>` (list) and `kubectl describe <thing> <name>` (details) are your most-used commands.

---

## 4. The Smallest Unit: Pods

A **Pod** wraps one container (occasionally a few tightly-coupled ones sharing network and disk).

> **Key fact:** Pods are **disposable**. When a pod dies it is never revived — a new pod with a new IP replaces it. Never rely on a specific pod sticking around.

```bash
kubectl run my-first-pod --image=nginx
kubectl get pods
kubectl describe pod my-first-pod   # full details + Events
kubectl logs my-first-pod
kubectl delete pod my-first-pod
```

The same pod as **YAML** (the way you'll actually work):

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.27        # pin a version, never use :latest
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f pod.yaml      # "make reality match this file"
```

> Always prefer `kubectl apply -f` over `kubectl run` — YAML is version-controlled and repeatable.

---

## 5. Don't Use Pods Directly: Deployments

A raw Pod has no self-healing and no scaling. A **Deployment** fixes that — it keeps a chosen number of identical pods running, replaces any that die, and handles rolling updates and rollbacks.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
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
          image: nginx:1.27
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f deployment.yaml
kubectl get pods               # three pods, each with a random suffix

# Watch self-healing:
kubectl delete pod <any-pod-name>
kubectl get pods               # a replacement starts immediately — still 3

# Rolling update and rollback:
kubectl set image deployment/nginx-deployment nginx=nginx:1.28
kubectl rollout status deployment/nginx-deployment
kubectl rollout undo deployment/nginx-deployment
```

> **The hierarchy:** `Deployment` → `ReplicaSet` → `Pods` → `Containers`. Edit the Deployment; everything below follows.

---

## 6. Talking to Your Pods: Services

Pods get new IPs every time they're recreated. A **Service** gives your group of pods one stable name/IP and load-balances traffic across them.

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx                 # routes to all pods labeled app=nginx
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP              # reachable only inside the cluster
```

**The three Service types:**

| Type | Reachable from | Use it for |
|------|----------------|-----------|
| **ClusterIP** (default) | Inside the cluster | Internal service-to-service calls |
| **NodePort** | Outside via `<NodeIP>:<port>` | Quick dev/testing access |
| **LoadBalancer** | Outside via cloud LB | Production public-facing apps |

> Services find their pods via **label selectors**, not IP addresses — labels are the glue connecting everything.

```bash
# Reach a ClusterIP service from your laptop while learning:
kubectl port-forward service/nginx-service 8080:80
```

> For routing many HTTP paths under one external IP, use an **Ingress** + Ingress Controller. Learn Services first.

---

## 7. Configuration: ConfigMaps & Secrets

Never bake config or passwords into your Docker image. Kubernetes separates config from code.

- **ConfigMap** — non-sensitive config (URLs, flags, `application.properties` values).
- **Secret** — sensitive data (passwords, tokens). Base64-encoded.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  SPRING_PROFILES_ACTIVE: "prod"
  APP_GREETING: "Hello from Kubernetes"
---
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_PASSWORD: c3VwZXJzZWNyZXQ=    # echo -n 'supersecret' | base64
```

**Injecting into a pod:**

```yaml
spec:
  containers:
    - name: my-app
      image: my-app:1.0
      envFrom:
        - configMapRef:
            name: app-config          # all keys as env vars
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD
```

> **Secrets are base64-*encoded*, not *encrypted*.** Anyone with kubectl access can decode them. Use RBAC to restrict access and enable encryption-at-rest for real security.

---

## 8. Keeping Pods Healthy: Probes

A running container isn't necessarily ready. A Spring Boot app takes time to start — Kubernetes uses probes to know the difference.

| Probe | Question | Failure action |
|-------|----------|----------------|
| **livenessProbe** | Is the app alive, or deadlocked? | **Restart** the container |
| **readinessProbe** | Is the app ready for traffic right now? | **Remove from Service** until it passes |
| **startupProbe** | Has a slow app finished booting? | Holds off other probes during startup |

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 5
```

> Spring Boot Actuator exposes these endpoints out of the box with `management.endpoint.health.probes.enabled=true`.

---

## 9. Giving Pods Resources: Requests & Limits

- **requests** — the guaranteed minimum; the Scheduler uses this to pick a node.
- **limits** — the hard cap; exceed memory → container is **OOMKilled**; exceed CPU → **throttled**.

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"        # 1000m = 1 core
  limits:
    memory: "512Mi"
    cpu: "500m"
```

> **JVM gotcha:** older JVMs saw the whole node's RAM and caused surprise OOMKills. Use Java 11+ (container-aware) and set `-XX:MaxRAMPercentage=75.0`.

---

## 10. Scaling Your App

**Manual:**

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

**Automatic (HPA):**

```yaml
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
          averageUtilization: 70    # scale up when avg CPU exceeds 70%
```

> **Horizontal** (HPA) = more pods. **Vertical** = bigger pods. Horizontal is preferred because it adds redundancy.

---

## 11. Storage That Survives Restarts

Anything written inside a container is lost when the pod dies. For persistent storage:

- **PersistentVolume (PV)** — an actual disk in the cluster.
- **PersistentVolumeClaim (PVC)** — a pod's request for storage. The pod mounts the PVC; Kubernetes binds it to a matching PV.

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

```yaml
# mounting in a pod:
spec:
  containers:
    - name: db
      image: postgres:16
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-pvc
```

> Most apps should be **stateless** — store state in an external DB, keep nothing in the pod. Use PVCs and **StatefulSets** only when you genuinely need stateful workloads.

---

## 12. Full Worked Example: Deploy a Spring Boot App

**Step 1 — Build & load your image:**

```bash
docker build -t my-springboot-app:1.0 .
minikube image load my-springboot-app:1.0
```

**Step 2 — Config + Secret:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: springboot-config
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SPRING_DATASOURCE_URL: "jdbc:postgresql://postgres-service:5432/appdb"
---
apiVersion: v1
kind: Secret
metadata:
  name: springboot-secret
type: Opaque
data:
  SPRING_DATASOURCE_PASSWORD: c3VwZXJzZWNyZXQ=
```

> `postgres-service` is used as a hostname — inside the cluster a Service name is a DNS name.

**Step 3 — Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: springboot-app
  template:
    metadata:
      labels:
        app: springboot-app
    spec:
      containers:
        - name: springboot-app
          image: my-springboot-app:1.0
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: springboot-config
          env:
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: springboot-secret
                  key: SPRING_DATASOURCE_PASSWORD
          resources:
            requests:
              memory: "512Mi"
              cpu: "300m"
            limits:
              memory: "768Mi"
              cpu: "600m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 40
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 25
            periodSeconds: 5
```

**Step 4 — Service:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: springboot-service
spec:
  selector:
    app: springboot-app
  ports:
    - port: 80
      targetPort: 8080
  type: LoadBalancer
```

**Step 5 — Apply and verify:**

```bash
kubectl apply -f app-config.yaml
kubectl apply -f springboot-deployment.yaml
kubectl apply -f springboot-service.yaml

kubectl get pods -l app=springboot-app         # expect 3 Running
kubectl rollout status deployment/springboot-app
kubectl logs -l app=springboot-app --tail=50

minikube service springboot-service --url
```

---

## 13. Debugging When Things Break

90% of Kubernetes debugging uses four commands — run them in order every time:

```bash
kubectl get pods                       # STEP 1: what's the STATUS?
kubectl describe pod <pod-name>        # STEP 2: scroll to "Events" — the why
kubectl logs <pod-name>                # STEP 3: what did the app print?
kubectl logs <pod-name> --previous     # logs from the previous crashed container
kubectl exec -it <pod-name> -- sh      # STEP 4: shell in to poke around
```

**Common pod statuses:**

| STATUS | Meaning | First thing to check |
|--------|---------|----------------------|
| `Running` | All good | — |
| `Pending` | Can't be scheduled | `describe` → not enough CPU/memory, or PVC unbound |
| `ImagePullBackOff` | Can't download the image | Wrong image name/tag, or missing registry credentials |
| `CrashLoopBackOff` | App starts then crashes, repeatedly | `logs --previous` → almost always an app error (bad config, can't reach DB) |
| `OOMKilled` | Exceeded memory limit | Raise the limit, or fix JVM heap setting |

> **`CrashLoopBackOff` is the #1 thing juniors panic about.** It is not a Kubernetes problem — your container keeps crashing. Run `kubectl logs <pod> --previous` and you'll almost always find a Java stack trace or a "connection refused" error.

---

## 14. Your 4-Week Learning Path

**Week 1 — Foundations**
- Install minikube. Run pods imperatively, then convert to YAML.
- Master `get`, `describe`, `logs`, `apply`, `delete`.
- Checkpoint: deploy nginx as a Deployment with 3 replicas; kill a pod and watch it heal.

**Week 2 — Connectivity & config**
- Services (ClusterIP, NodePort, LoadBalancer) + `port-forward`.
- ConfigMaps & Secrets injected as env vars.
- Checkpoint: deploy two apps that talk to each other via Service DNS names.

**Week 3 — Production concerns**
- Liveness/readiness probes with Spring Boot Actuator.
- Resource requests/limits; cause and fix an `OOMKilled`.
- Rolling updates and rollbacks.
- Checkpoint: deploy your own Spring Boot app (Section 12 end-to-end).

**Week 4 — Going further**
- HPA autoscaling; PersistentVolumeClaims; namespaces.
- Skim **Helm** (K8s package manager) and **Ingress** (HTTP routing).
- Read `Kubernetes_Interview_Questions.md` for rapid revision.
- Checkpoint: explain the journey of an HTTP request from the internet to your Spring Boot pod.

> **Related guides:** `Docker_Concepts_Study_Guide.md` (learn first), `Kubernetes_Interview_Questions.md` (Q&A revision), `Observability_Tracing_Metrics_Logging.md`, `CI_CD_Pipelines_Deep_Dive.md`.

---

## 15. Quick Reference Cheat Sheet

**Object hierarchy:**
```
Deployment → ReplicaSet → Pod → Container
Service     → (selects pods by label) → load-balances traffic
ConfigMap/Secret → injected into pods as env vars or files
PVC → PV → persistent storage mounted into a pod
HPA → watches metrics → scales a Deployment's replicas
```

**The one idea:** *Declare desired state; Kubernetes reconciles reality to match it — forever.*

**Everyday `kubectl` commands:**
```bash
kubectl apply -f file.yaml               # create/update from YAML
kubectl get <pods|svc|deploy|all>        # list resources
kubectl get pods -o wide                 # with node + IP columns
kubectl get pods -l app=myapp            # filter by label
kubectl describe pod <name>              # details + Events
kubectl logs <name>                      # container logs
kubectl logs <name> --previous           # logs from crashed container
kubectl exec -it <name> -- sh            # shell into a container
kubectl port-forward svc/<name> 8080:80  # reach a service from localhost
kubectl scale deploy/<name> --replicas=5
kubectl set image deploy/<name> <c>=<img>:<tag>
kubectl rollout status deploy/<name>
kubectl rollout undo deploy/<name>
kubectl delete -f file.yaml
```

**Service types:**

| Type | Scope | Use |
|------|-------|-----|
| ClusterIP | Internal only | service-to-service |
| NodePort | `NodeIP:port` | dev/testing |
| LoadBalancer | External (cloud LB) | production public apps |

**Probes:**

| Probe | Failure → |
|-------|-----------|
| liveness | container **restarted** |
| readiness | pod **removed from Service** |
| startup | delays other probes for slow boots |

**Resources:** `requests` = guaranteed minimum (scheduling) · `limits` = hard cap (memory over = OOMKilled, CPU over = throttled).

**Debugging order:** `get pods` → `describe pod` (Events) → `logs` → `exec`. `CrashLoopBackOff` = your app is crashing — read `logs --previous`.

**Golden rules:**
1. Use **Deployments**, never bare Pods.
2. Use declarative **YAML + `kubectl apply`**, not one-liners.
3. Keep apps **stateless**; put state in external databases.
4. Always set **resource requests/limits** and **health probes**.
5. **Pin image versions** (`:1.27`), never `:latest` in production.
6. Secrets are base64-**encoded**, not encrypted — protect with RBAC + encryption-at-rest.

---

*Last Updated: 2026-06-18*
