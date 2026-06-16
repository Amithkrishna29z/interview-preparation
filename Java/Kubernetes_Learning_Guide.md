# Kubernetes Learning Guide — From Zero to Your First Deployment

> 🎯 **A hands-on, step-by-step path to *learn* Kubernetes** (not just memorize answers). Built for a junior Java backend developer. Once you finish this, read `Kubernetes_Interview_Questions.md` for rapid interview revision.

> 💡 **How to use this guide:** Read top to bottom. Each section builds on the last. Type out every command yourself — Kubernetes is a muscle you build by *doing*, not reading.

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

Before Kubernetes, deploying an app looked like this:

- You build a Docker container for your Spring Boot app.
- You run it on a server with `docker run`.
- Then reality hits:
  - The container **crashes at 3 AM** → your app is down until someone notices.
  - Traffic **spikes on Black Friday** → one container can't handle it, and you manually start more.
  - You need a **new version deployed** → you stop the old one, start the new one, and pray there's no downtime.
  - One server **dies** → all containers on it vanish.

Doing this by hand for one container is annoying. Doing it for **50 containers across 10 servers** is impossible for a human.

> 🔑 **Think of it like an air traffic controller.** A single pilot can fly one plane. But an airport with 200 planes needs a control tower that tracks every flight, reroutes around storms, assigns runways, and reacts instantly when something goes wrong. Kubernetes is the control tower for your containers.

**Kubernetes (K8s) automates** the boring, error-prone parts:

| Problem | What Kubernetes does for you |
|---------|------------------------------|
| Container crashes | **Self-healing** — restarts it automatically |
| Traffic spike | **Auto-scaling** — adds more copies |
| New version | **Rolling update** — swaps versions with zero downtime |
| Server dies | **Rescheduling** — moves containers to a healthy server |
| Many copies of an app | **Load balancing** — spreads requests across them |

> 📝 "K8s" is shorthand: **K** + 8 letters + **s**. Same word, fewer keystrokes.

---

## 2. Mental Model: What Kubernetes Actually Does

The single most important idea in Kubernetes is the **desired state** model.

You don't tell Kubernetes *how* to do things step by step (imperative). You tell it **what you want** (declarative), and it works tirelessly to make reality match.

> 🔑 **Think of it like a thermostat.** You set the temperature to 22°C (your *desired state*). You don't manually turn the heater on and off — the thermostat constantly checks the *current* temperature and acts to close the gap. Kubernetes does the same: "I want 3 copies of my app running" → if one dies and you have 2, it starts a 3rd. Forever.

```
You declare:   "I want 3 replicas of my-app:v2 running"
                            │
                            ▼
        ┌───────────────────────────────────┐
        │   Kubernetes Control Plane          │
        │   (constantly compares)             │
        │                                     │
        │   Desired: 3 pods   Actual: 2 pods  │
        │            └──────► starts 1 more ◄─┘
        └───────────────────────────────────┘
```

This is called a **reconciliation loop**: *observe current state → compare to desired state → take action → repeat forever.* Almost everything in Kubernetes works this way.

**The cluster has two kinds of machines** (for a deeper breakdown see `Kubernetes_Interview_Questions.md` → Architecture):

- **Control Plane (the brain)** — decides *what* should run *where*. Contains the API Server (front door), Scheduler (placement), etcd (the database of truth), and Controllers (the reconciliation loops).
- **Worker Nodes (the muscle)** — actually run your containers. Each has a `kubelet` (agent that talks to the brain) and a container runtime (e.g. containerd).

You, the developer, mostly talk to the **API Server** using a command-line tool called **`kubectl`** ("kube control").

---

## 3. Set Up a Local Playground

You don't need a cloud account or 10 servers to learn. You run a whole cluster on your laptop.

**Option A — `minikube` (recommended for learning):**

```bash
# Install minikube (see minikube.sigs.k8s.io for your OS), then:
minikube start                 # boots a single-node cluster in a VM/container
minikube status                # confirm it's running

kubectl get nodes              # ask the cluster: what machines do you have?
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   1m    v1.30.0
```

**Option B — Docker Desktop:** enable Kubernetes in Settings → Kubernetes → "Enable Kubernetes". `kubectl` is configured automatically.

> 💡 **`kubectl` is your remote control.** It sends your wishes to the API Server. Every command in this guide is something you can type right now. Get used to `kubectl get <thing>` (list) and `kubectl describe <thing> <name>` (details) — you'll use them constantly.

```bash
kubectl version --short        # check client + server versions
kubectl get all                # list everything in the default namespace (empty for now)
```

---

## 4. The Smallest Unit: Pods

A **Pod** is the smallest thing Kubernetes runs. A Pod wraps **one container** (occasionally a few tightly-coupled ones that must share a network and disk).

> 🔑 **Think of it like a pod of peas.** The pod is the shell; your container is the pea inside. Kubernetes never hands you a bare pea — it always wraps it in a pod so it can manage it consistently.

> ❗ **Key fact:** Pods are **disposable**. They get created, killed, and replaced all the time. A pod that dies is **never** revived — a *new* pod with a *new* IP address replaces it. Never rely on a specific pod sticking around. (This is *why* we need Deployments and Services, coming up next.)

Let's run your first pod imperatively (quick, throwaway style):

```bash
# Run a single nginx pod (a tiny web server) just to see it work
kubectl run my-first-pod --image=nginx

kubectl get pods               # watch it start
# NAME            READY   STATUS    RESTARTS   AGE
# my-first-pod    1/1     Running   0          10s

kubectl describe pod my-first-pod   # full details: events, IP, node, why it might be failing
kubectl logs my-first-pod           # see the container's stdout logs
kubectl delete pod my-first-pod     # clean up
```

The same pod written **declaratively** as YAML (the way you'll actually work):

```yaml
# pod.yaml
apiVersion: v1                 # which API version this object uses
kind: Pod                      # the type of object we're creating
metadata:
  name: my-first-pod           # the pod's name (must be unique in its namespace)
  labels:
    app: nginx                 # labels are key/value tags — used later to find this pod
spec:
  containers:
    - name: nginx              # name of the container inside the pod
      image: nginx:1.27        # pin a version, never rely on :latest in real work
      ports:
        - containerPort: 80    # the port the app listens on inside the container
```

```bash
kubectl apply -f pod.yaml      # "make reality match this file"
kubectl get pod my-first-pod -o yaml   # see the FULL object, including fields K8s filled in
```

> 💡 **Always prefer YAML files (`kubectl apply -f`) over `kubectl run`.** YAML is version-controlled, reviewable, and repeatable — it's infrastructure-as-code. `kubectl run` is fine for quick experiments only.

---

## 5. Don't Use Pods Directly: Deployments

You almost never create Pods directly. Why? Because a raw Pod has **no self-healing and no scaling** — if it dies, it's gone.

Instead you create a **Deployment**. A Deployment is a manager that:
- Keeps a chosen number of identical pods running (via a **ReplicaSet** under the hood).
- Replaces any pod that dies.
- Rolls out new versions gradually (zero-downtime updates).
- Lets you roll back to a previous version.

> 🔑 **Think of it like a restaurant manager.** You tell the manager "I always want 3 waiters on the floor." If a waiter calls in sick, the manager calls in a replacement — you don't micromanage each one. You set the *policy*; the manager maintains it.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3                  # DESIRED STATE: always keep 3 pods running
  selector:
    matchLabels:
      app: nginx               # this Deployment manages pods with the label app=nginx
  template:                    # the "cookie cutter" for every pod it creates
    metadata:
      labels:
        app: nginx             # pods get this label (must match the selector above)
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f deployment.yaml

kubectl get deployment         # see the deployment
kubectl get pods               # now THREE pods, with random name suffixes
# nginx-deployment-7c5ddbdf54-abcde   1/1   Running
# nginx-deployment-7c5ddbdf54-fghij   1/1   Running
# nginx-deployment-7c5ddbdf54-klmno   1/1   Running

# Watch self-healing in action:
kubectl delete pod nginx-deployment-7c5ddbdf54-abcde
kubectl get pods               # K8s immediately starts a replacement — still 3!
```

**Updating to a new version (rolling update):**

```bash
# Change the image; K8s replaces pods gradually, never dropping below capacity
kubectl set image deployment/nginx-deployment nginx=nginx:1.28
kubectl rollout status deployment/nginx-deployment   # watch the rollout progress
kubectl rollout undo deployment/nginx-deployment     # oops — roll back to previous version
```

> 📝 **The hierarchy:** `Deployment` → manages → `ReplicaSet` → manages → `Pods` → contain → `Containers`. You edit the Deployment; everything below it follows.

---

## 6. Talking to Your Pods: Services

Problem: pods are disposable and each gets a **new IP** when recreated. If your frontend hard-codes a pod's IP, it breaks the moment that pod is replaced. And you have 3 nginx pods — which IP do you even use?

A **Service** solves this. It gives your group of pods **one stable name and IP**, and **load-balances** requests across all healthy pods behind it.

> 🔑 **Think of it like a company's main phone number.** Employees (pods) come and go, desk extensions change, but the public number stays the same. You call the main number; the switchboard (Service) routes you to whoever's available. Callers never need to know individual extensions.

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx                 # route traffic to all pods labeled app=nginx
  ports:
    - port: 80                 # the port the Service exposes
      targetPort: 80           # the port on the pods to forward to
  type: ClusterIP              # default: reachable only INSIDE the cluster
```

```bash
kubectl apply -f service.yaml
kubectl get service nginx-service
# NAME            TYPE        CLUSTER-IP      PORT(S)
# nginx-service   ClusterIP   10.96.120.50    80/TCP
```

**The three Service types you must know:**

| Type | Reachable from | Use it for |
|------|----------------|-----------|
| **ClusterIP** (default) | Inside the cluster only | Internal communication (e.g. backend → database) |
| **NodePort** | Outside, via `<NodeIP>:<port>` | Quick testing / dev access |
| **LoadBalancer** | Outside, via a cloud load balancer | Production public-facing apps (cloud only) |

> 🔑 **How a Service finds its pods:** it uses the **label selector** (`app: nginx`), *not* IP addresses. This is why labels matter — they're the glue connecting Services to Pods, and Deployments to Pods.

```bash
# Quickly reach a ClusterIP service from your laptop while learning:
kubectl port-forward service/nginx-service 8080:80
# Now open http://localhost:8080 in your browser — you hit the nginx pods!
```

> 💡 For exposing many HTTP routes under one external IP with paths/hostnames (e.g. `/api` → backend, `/` → frontend), you graduate to an **Ingress** + Ingress Controller. Learn Services solidly first; Ingress builds on them.

---

## 7. Configuration: ConfigMaps & Secrets

Never bake configuration (database URLs, feature flags) or passwords into your Docker image. You'd need to rebuild the image for every environment. Kubernetes separates config from code.

- **ConfigMap** — non-sensitive config (URLs, flags, `application.properties` values).
- **Secret** — sensitive data (passwords, API keys, tokens). Base64-encoded and handled more carefully by the cluster.

> 🔑 **Think of it like a recipe vs. the ingredients.** The Docker image is the recipe (fixed). ConfigMaps/Secrets are the ingredients you supply at cooking time — same recipe, different ingredients for dev vs. production.

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  SPRING_PROFILES_ACTIVE: "prod"          # plain key/value config
  APP_GREETING: "Hello from Kubernetes"
---
# secret.yaml  (values are base64-encoded, NOT encrypted — see note below)
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_PASSWORD: c3VwZXJzZWNyZXQ=           # echo -n 'supersecret' | base64
```

**Injecting them into a pod as environment variables:**

```yaml
# inside a container spec:
spec:
  containers:
    - name: my-app
      image: my-app:1.0
      envFrom:
        - configMapRef:
            name: app-config              # pull ALL keys from the ConfigMap as env vars
      env:
        - name: DB_PASSWORD               # pull a single value from the Secret
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD
```

> ❗ **Secrets are base64-*encoded*, not *encrypted*.** Anyone with `kubectl get secret -o yaml` access can decode them. For real security, enable encryption-at-rest in etcd and use RBAC to restrict who can read Secrets (or an external vault). Base64 just keeps binary-safe data, not attackers out.

---

## 8. Keeping Pods Healthy: Probes

A container being "up" doesn't mean it's *ready to serve traffic*. A Spring Boot app takes 20 seconds to start; during that time the process is running but the app isn't ready. Kubernetes uses **probes** to know the difference.

| Probe | Question it answers | What happens if it fails |
|-------|---------------------|--------------------------|
| **livenessProbe** | "Is the app alive, or stuck/deadlocked?" | Kubernetes **restarts** the container |
| **readinessProbe** | "Is the app ready to receive traffic *right now*?" | Pod is **removed from the Service** (no traffic) until it passes |
| **startupProbe** | "Has a slow-starting app finished booting?" | Holds off the other probes until startup completes |

> 🔑 **Think of it like a new employee's first day.** *Readiness* = "Are you set up and ready to take customer calls?" (if not, don't route calls to them yet, but don't fire them). *Liveness* = "Are you still responsive, or did you faint at your desk?" (if unresponsive, send them home and call a replacement).

```yaml
# inside a container spec — perfect fit for Spring Boot Actuator:
livenessProbe:
  httpGet:
    path: /actuator/health/liveness   # Spring Boot Actuator liveness endpoint
    port: 8080
  initialDelaySeconds: 30             # wait 30s before the first check (app boot time)
  periodSeconds: 10                   # then check every 10s
readinessProbe:
  httpGet:
    path: /actuator/health/readiness  # readiness endpoint (DB connected? etc.)
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 5
```

> 💡 **Backend dev tip:** Spring Boot Actuator exposes `/actuator/health/liveness` and `/actuator/health/readiness` out of the box (`management.endpoint.health.probes.enabled=true`). This is the standard way Java apps integrate with Kubernetes health checks.

---

## 9. Giving Pods Resources: Requests & Limits

You tell Kubernetes how much CPU and memory each pod needs. This drives scheduling (where pods land) and protects the cluster from one greedy pod starving the rest.

- **requests** — the *guaranteed minimum*. The Scheduler uses this to pick a node with enough free capacity.
- **limits** — the *hard maximum*. Exceed the memory limit and the container is **killed (OOMKilled)**; exceed CPU and it's **throttled** (slowed, not killed).

> 🔑 **Think of it like booking a hotel room.** *Request* = "I need at least a room with 2 beds" (the hotel guarantees it and reserves space). *Limit* = "I won't use more than the mini-bar allowance" (go over and you get cut off). Requests reserve; limits cap.

```yaml
# inside a container spec:
resources:
  requests:
    memory: "256Mi"            # guarantee 256 mebibytes of RAM
    cpu: "250m"                # guarantee 0.25 of a CPU core (1000m = 1 core)
  limits:
    memory: "512Mi"            # kill the container if it exceeds 512Mi
    cpu: "500m"                # throttle the container above 0.5 core
```

> ❗ **JVM gotcha:** older JVMs ignored container memory limits and saw the whole node's RAM, causing surprise OOMKills. Use Java 11+ (container-aware by default) and set the heap relative to the limit, e.g. `-XX:MaxRAMPercentage=75.0`. This is a *very* common real-world bug for Java-on-Kubernetes.

---

## 10. Scaling Your App

**Manual scaling** — change the replica count:

```bash
kubectl scale deployment nginx-deployment --replicas=5   # now run 5 pods
kubectl get pods                                         # watch 2 new pods appear
```

**Automatic scaling (HPA — Horizontal Pod Autoscaler)** — let Kubernetes add/remove pods based on load:

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment     # which Deployment to scale
  minReplicas: 2               # never go below 2 pods
  maxReplicas: 10              # never go above 10 pods
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # add pods when avg CPU across pods exceeds 70%
```

> 🔑 **Think of it like a shop hiring seasonal staff.** Quiet Tuesday → 2 staff. Black Friday rush → automatically bring in up to 10. When the rush ends, send the extras home. You set the floor, the ceiling, and the trigger; the system handles the rest.

> 📝 **Horizontal vs Vertical scaling:** *Horizontal* (HPA) = more pods (the usual K8s way). *Vertical* = bigger pods (more CPU/RAM each). Horizontal is preferred because it adds redundancy too.

---

## 11. Storage That Survives Restarts

Pods are disposable, so **anything written inside a pod's container is lost when the pod dies.** That's fine for stateless web apps, but databases need persistence.

Two key objects:

- **PersistentVolume (PV)** — an actual piece of storage in the cluster (a disk).
- **PersistentVolumeClaim (PVC)** — a pod's *request* for storage ("I need 5Gi"). The pod mounts the PVC; Kubernetes binds it to a matching PV.

> 🔑 **Think of it like a coat check.** The PVC is your ticket ("I want to store one coat"). The PV is the actual hook your coat hangs on. You hand over the ticket (mount the PVC), and you get the *same* coat back even after the pod that hung it up is long gone.

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce            # mounted read-write by a single node
  resources:
    requests:
      storage: 5Gi            # ask for 5 gibibytes
```

```yaml
# mounting it in a pod:
spec:
  containers:
    - name: db
      image: postgres:16
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data   # where the container sees the storage
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-pvc                       # connect to the PVC above
```

> 💡 **Stateless vs Stateful:** Most apps should be **stateless** — store state in an external database, keep zero data in the pod. This makes scaling and self-healing trivial. Reach for PVCs and **StatefulSets** (a Deployment variant for databases needing stable identity + storage) only when you genuinely run stateful workloads.

---

## 12. Full Worked Example: Deploy a Spring Boot App

Let's tie it all together. You have a Spring Boot REST API, containerized as `my-springboot-app:1.0`. Here's a production-shaped set of manifests.

**Step 1 — Build & load your image** (so minikube can see it):

```bash
# Build your Spring Boot image (from your project with a Dockerfile)
docker build -t my-springboot-app:1.0 .
minikube image load my-springboot-app:1.0    # make the local image available to minikube
```

**Step 2 — Config + Secret:**

```yaml
# app-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: springboot-config
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SPRING_DATASOURCE_URL: "jdbc:postgresql://postgres-service:5432/appdb"  # note: uses the DB SERVICE name as hostname
---
apiVersion: v1
kind: Secret
metadata:
  name: springboot-secret
type: Opaque
data:
  SPRING_DATASOURCE_PASSWORD: c3VwZXJzZWNyZXQ=     # base64 of 'supersecret'
```

> 🔑 **Notice `postgres-service` is used as a hostname.** Inside the cluster, a Service name *is* a DNS name. Your app connects to `postgres-service:5432` and Kubernetes resolves it to the database pods. No IPs anywhere.

**Step 3 — Deployment (with everything you've learned):**

```yaml
# springboot-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-app
spec:
  replicas: 3                                 # 3 copies for availability + load spreading
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
                name: springboot-config        # inject all config keys
          env:
            - name: SPRING_DATASOURCE_PASSWORD # inject the secret value
              valueFrom:
                secretKeyRef:
                  name: springboot-secret
                  key: SPRING_DATASOURCE_PASSWORD
          resources:                            # protect the cluster + drive scheduling
            requests:
              memory: "512Mi"
              cpu: "300m"
            limits:
              memory: "768Mi"
              cpu: "600m"
          livenessProbe:                        # restart if deadlocked
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 40
            periodSeconds: 10
          readinessProbe:                       # no traffic until DB is connected & ready
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 25
            periodSeconds: 5
```

**Step 4 — Service (stable address + load balancing):**

```yaml
# springboot-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: springboot-service
spec:
  selector:
    app: springboot-app                        # route to the 3 app pods
  ports:
    - port: 80                                 # external port on the service
      targetPort: 8080                         # container port
  type: LoadBalancer                           # public access in the cloud (NodePort on minikube)
```

**Step 5 — Apply it all and verify:**

```bash
kubectl apply -f app-config.yaml
kubectl apply -f springboot-deployment.yaml
kubectl apply -f springboot-service.yaml

kubectl get pods -l app=springboot-app         # -l filters by label; expect 3 Running
kubectl rollout status deployment/springboot-app
kubectl logs -l app=springboot-app --tail=50   # tail logs across all matching pods

minikube service springboot-service --url      # get a reachable URL on minikube
```

🎉 You just deployed a load-balanced, self-healing, config-driven, health-checked Spring Boot app on Kubernetes — the same shape you'd use in production.

---

## 13. Debugging When Things Break

90% of Kubernetes debugging uses just a handful of commands. Learn this loop.

```bash
kubectl get pods                       # STEP 1: what's the STATUS? (the clue is here)
kubectl describe pod <pod-name>        # STEP 2: scroll to "Events" at the bottom — the why
kubectl logs <pod-name>                # STEP 3: what did the app itself print?
kubectl logs <pod-name> --previous     # logs from the PREVIOUS crashed container
kubectl exec -it <pod-name> -- sh      # STEP 4: shell INTO the container to poke around
```

**Decode the common pod statuses** (this table saves hours):

| STATUS | What it means | First thing to check |
|--------|---------------|----------------------|
| `Running` | All good | — |
| `Pending` | Can't be scheduled | `describe` → not enough CPU/memory on any node, or PVC unbound |
| `ImagePullBackOff` | Can't download the image | Wrong image name/tag, or private registry needs credentials |
| `CrashLoopBackOff` | App starts then crashes, repeatedly | `logs --previous` → it's almost always an **app error** (bad config, can't reach DB) |
| `OOMKilled` | Used more memory than its limit | Raise the memory limit, or fix the leak / JVM heap setting |
| `ErrImagePull` | Image pull failed | Same as ImagePullBackOff |

> 🔑 **Think of it like a doctor's diagnosis.** `get pods` = check the temperature (the symptom). `describe` = read the chart and history (Events). `logs` = listen to what the patient says hurts. Don't guess — follow the loop in order every time.

> 💡 **`CrashLoopBackOff` is the #1 thing juniors panic about.** It is *not* a Kubernetes problem — it means *your container keeps exiting*. Run `kubectl logs <pod> --previous` and you'll almost always see a Java stack trace or a "connection refused" — fix that.

---

## 14. Your 4-Week Learning Path

A realistic, hands-on schedule. Do, don't just read.

**Week 1 — Foundations**
- Install minikube. Run pods imperatively, then convert to YAML.
- Master `get`, `describe`, `logs`, `apply`, `delete`.
- ✅ *Checkpoint:* deploy nginx as a Deployment with 3 replicas; kill a pod and watch it heal.

**Week 2 — Real connectivity & config**
- Services (ClusterIP, NodePort, LoadBalancer) + `port-forward`.
- ConfigMaps & Secrets injected as env vars.
- ✅ *Checkpoint:* deploy two apps that talk to each other via Service DNS names.

**Week 3 — Production concerns**
- Liveness/readiness probes (with Spring Boot Actuator).
- Resource requests/limits; cause and fix an `OOMKilled`.
- Rolling updates and rollbacks.
- ✅ *Checkpoint:* deploy your own Spring Boot app (Section 12 end-to-end).

**Week 4 — Going further**
- HPA autoscaling; PersistentVolumeClaims; namespaces.
- Skim **Helm** (package manager for K8s) and **Ingress** (HTTP routing).
- Then read `Kubernetes_Interview_Questions.md` for rapid revision.
- ✅ *Checkpoint:* explain, out loud, the journey of an HTTP request from the internet to your Spring Boot pod.

> 📚 **Related guides in this repo:** `Docker_Concepts_Study_Guide.md` (containers — learn first), `Kubernetes_Interview_Questions.md` (Q&A revision), `Observability_Tracing_Metrics_Logging.md` (monitoring your pods), `CI_CD_Pipelines_Deep_Dive.md` (deploying to K8s automatically).

---

## 15. Quick Reference Cheat Sheet

**Core objects (the mental hierarchy):**
```
Deployment → ReplicaSet → Pod → Container
Service     → (selects pods by label) → load-balances traffic
ConfigMap/Secret → injected into pods as env vars or files
PVC → PV → persistent storage mounted into a pod
HPA → watches metrics → scales a Deployment's replicas
```

**The one idea to remember:** *You declare desired state; Kubernetes reconciles reality to match it — forever.*

**Everyday `kubectl` commands:**
```bash
kubectl apply -f file.yaml             # create/update from YAML (your main verb)
kubectl get <pods|svc|deploy|all>      # list resources
kubectl get pods -o wide               # list with node + IP columns
kubectl get pods -l app=myapp          # filter by label
kubectl describe pod <name>            # details + Events (debugging starts here)
kubectl logs <name>                    # container logs
kubectl logs <name> --previous         # logs from a crashed container
kubectl exec -it <name> -- sh          # shell into a container
kubectl port-forward svc/<name> 8080:80  # reach a service from localhost
kubectl scale deploy/<name> --replicas=5
kubectl set image deploy/<name> <c>=<img>:<tag>   # trigger a rolling update
kubectl rollout status deploy/<name>
kubectl rollout undo deploy/<name>     # roll back
kubectl delete -f file.yaml            # remove what the file created
```

**Service types:**

| Type | Scope | Use |
|------|-------|-----|
| ClusterIP | Internal only | service-to-service |
| NodePort | `NodeIP:port` | dev/testing |
| LoadBalancer | External (cloud LB) | production public apps |

**Probes:**

| Probe | Fails → |
|-------|---------|
| liveness | container **restarted** |
| readiness | pod **removed from Service** (no traffic) |
| startup | delays liveness/readiness for slow boots |

**Resources:** `requests` = guaranteed minimum (scheduling) · `limits` = hard cap (memory over = OOMKilled, CPU over = throttled).

**Debugging order:** `get pods` (status) → `describe pod` (Events) → `logs` (app output) → `exec` (poke inside). `CrashLoopBackOff` ≈ your app is crashing — read `logs --previous`.

**Golden rules:**
1. Prefer **Deployments**, never bare Pods.
2. Prefer declarative **YAML + `kubectl apply`**, not imperative one-liners.
3. Treat apps as **stateless**; put state in external databases.
4. Always set **resource requests/limits** and **health probes**.
5. **Pin image versions** (`:1.27`), never use `:latest` in real deployments.
6. Secrets are base64-**encoded**, not encrypted — protect them with RBAC + encryption-at-rest.

> 🚀 **You've got the foundation.** Build the Section 12 example yourself, break it on purpose, and fix it. That's how Kubernetes actually clicks.
