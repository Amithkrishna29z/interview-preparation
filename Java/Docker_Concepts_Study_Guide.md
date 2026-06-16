# Docker Concepts - Interview Preparation Study Guide

---

## Table of Contents
1. [What is Docker?](#what-is-docker)
2. [Core Architecture](#core-architecture)
3. [Images](#images)
4. [Containers](#containers)
5. [Dockerfile](#dockerfile)
6. [Docker Networking](#docker-networking)
7. [Docker Volumes & Storage](#docker-volumes--storage)
8. [Docker Compose](#docker-compose)
9. [Docker Registry](#docker-registry)
10. [Docker Swarm vs Kubernetes](#docker-swarm-vs-kubernetes)
11. [Security](#security)
12. [Performance & Best Practices](#performance--best-practices)
13. [Common Interview Questions](#common-interview-questions)

---

## What is Docker?

Docker is an **open-source containerization platform** that packages applications and their dependencies into lightweight, portable units called **containers**.

### VM vs Container (Critical Interview Topic)

| Feature | Virtual Machine | Container |
|---|---|---|
| OS | Full guest OS per VM | Shares host OS kernel |
| Size | GBs | MBs |
| Startup | Minutes | Seconds/milliseconds |
| Isolation | Hardware-level | Process-level (namespaces) |
| Overhead | High | Low |
| Portability | Limited | High |
| Use case | Full OS isolation | App isolation |

**Analogy**: VMs are like separate houses (each with their own foundation, plumbing, electricity). Containers are like apartments in a building — they share infrastructure but are isolated from each other.

### Why Docker?

- **"Works on my machine" problem**: Docker eliminates environment inconsistencies
- **Microservices**: Each service runs in its own container
- **CI/CD**: Consistent build and deploy environments
- **Resource efficiency**: Run more workloads on the same hardware

---

## Core Architecture

```
┌─────────────────────────────────────────┐
│           Docker Client (CLI)           │
│         docker build / run / pull       │
└──────────────────┬──────────────────────┘
                   │ REST API
┌──────────────────▼──────────────────────┐
│            Docker Daemon (dockerd)       │
│   - Manages images, containers,         │
│     networks, volumes                   │
└──────────┬───────────────────┬──────────┘
           │                   │
┌──────────▼──────┐   ┌────────▼────────┐
│  containerd     │   │  Docker Registry │
│  (runtime)      │   │  (Docker Hub /  │
│                 │   │   private)       │
└──────────┬──────┘   └─────────────────┘
           │
┌──────────▼──────┐
│   runc (OCI)    │
│  (creates       │
│   containers)   │
└─────────────────┘
```

### Key Components

| Component | Role |
|---|---|
| **Docker Client** | CLI tool (`docker` command) that sends commands to daemon |
| **Docker Daemon** | Background service that manages Docker objects |
| **containerd** | Container runtime (manages container lifecycle) |
| **runc** | Low-level runtime that creates/runs containers per OCI spec |
| **Docker Registry** | Stores and distributes Docker images |

### How Docker Uses Linux Primitives

- **Namespaces**: Isolate processes (PID, Network, Mount, UTS, IPC, User namespaces)
- **cgroups (Control Groups)**: Limit resource usage (CPU, memory, I/O)
- **Union File System (OverlayFS)**: Layered image system

---

## Images

A Docker image is a **read-only template** used to create containers. It's a stack of layers, each representing a filesystem change.

### Image Layers

```
┌──────────────────────────┐  ← Writable layer (container layer)
├──────────────────────────┤
│   App layer (COPY ./app) │  ← Read-only image layers
├──────────────────────────┤
│   Dependencies (npm i)   │
├──────────────────────────┤
│   Base OS (ubuntu:22.04) │
└──────────────────────────┘
```

**Key insight**: Each instruction in a Dockerfile creates a new layer. Layers are cached and shared across images — this is why Docker is space-efficient.

### Image Commands

```bash
docker images                          # List local images
docker pull nginx:latest               # Pull from registry
docker push myrepo/myapp:1.0           # Push to registry
docker build -t myapp:1.0 .            # Build from Dockerfile
docker rmi myapp:1.0                   # Remove image
docker image prune                     # Remove dangling images
docker history nginx                   # Show image layers
docker inspect nginx                   # Detailed image metadata
docker tag nginx myrepo/nginx:custom   # Tag an image
```

### Image Naming Convention

```
[registry]/[username]/[image-name]:[tag]

docker.io/library/nginx:latest         # Official image (Docker Hub)
gcr.io/myproject/myapp:v1.2.3          # Google Container Registry
mycompany.azurecr.io/api:prod          # Azure Container Registry
```

---

## Containers

A container is a **running instance of an image** — an isolated process with its own filesystem, network, and process space.

### Container Lifecycle

```
Created → Running → Paused → Running → Stopped → Removed
              ↑                           ↓
           docker start            docker stop/kill
```

### Container Commands

```bash
# Run a container
docker run nginx                              # Foreground
docker run -d nginx                           # Detached (background)
docker run -d -p 8080:80 --name web nginx     # Named, port mapped
docker run -it ubuntu bash                    # Interactive terminal
docker run --rm ubuntu echo "hello"           # Auto-remove after exit

# Manage containers
docker ps                    # Running containers
docker ps -a                 # All containers
docker stop <id>             # Graceful stop (SIGTERM → SIGKILL after timeout)
docker kill <id>             # Immediate stop (SIGKILL)
docker restart <id>          # Stop and start
docker rm <id>               # Remove stopped container
docker rm -f <id>            # Force remove (even if running)

# Interact
docker exec -it <id> bash    # Shell into running container
docker logs <id>             # View container logs
docker logs -f <id>          # Follow logs
docker cp file.txt <id>:/app # Copy file into container
docker stats                 # Real-time resource usage

# Inspect
docker inspect <id>          # Full container metadata (JSON)
docker top <id>              # Running processes inside container
docker port <id>             # Port mappings
```

### Resource Limits

```bash
docker run -d \
  --memory="512m" \          # Max memory
  --cpus="1.5" \             # CPU cores limit
  --memory-swap="1g" \       # Swap limit
  nginx
```

---

## Dockerfile

A Dockerfile is a text file with instructions to build a Docker image.

### Complete Dockerfile Reference

```dockerfile
# Syntax: INSTRUCTION arguments

# ─── Base Image ───────────────────────────────────────────────
FROM ubuntu:22.04                # Start from base image
FROM node:18-alpine AS builder   # Multi-stage: name this stage

# ─── Metadata ─────────────────────────────────────────────────
LABEL maintainer="team@company.com"
LABEL version="1.0"

# ─── Environment ──────────────────────────────────────────────
ENV NODE_ENV=production          # Set env variable (persists in container)
ARG VERSION=1.0                  # Build-time variable (not in final image)

# ─── Working Directory ────────────────────────────────────────
WORKDIR /app                     # Set working directory (creates if absent)

# ─── File Operations ──────────────────────────────────────────
COPY package.json ./             # Copy from build context to image
COPY . .                         # Copy all files
ADD archive.tar.gz /app/         # Like COPY but auto-extracts archives

# ─── Run Commands ─────────────────────────────────────────────
RUN apt-get update && \
    apt-get install -y curl && \  # Chain commands to minimize layers
    rm -rf /var/lib/apt/lists/*   # Clean up in same layer

# ─── Expose Port ──────────────────────────────────────────────
EXPOSE 3000                      # Documentation only (doesn't publish)

# ─── User ─────────────────────────────────────────────────────
RUN useradd -m appuser
USER appuser                     # Run as non-root (security best practice)

# ─── Volumes ──────────────────────────────────────────────────
VOLUME ["/data"]                 # Mount point for external storage

# ─── Health Check ─────────────────────────────────────────────
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# ─── Entry / Command ──────────────────────────────────────────
ENTRYPOINT ["node"]              # Fixed command (cannot be overridden easily)
CMD ["server.js"]                # Default args (can be overridden at runtime)
```

### COPY vs ADD

| Instruction | Use Case |
|---|---|
| `COPY` | Simple file/directory copy — **prefer this** |
| `ADD` | Needs auto-extraction of `.tar.gz` or URL download |

### ENTRYPOINT vs CMD

```dockerfile
# CMD only — can be overridden completely
CMD ["nginx", "-g", "daemon off;"]
# docker run myimage bash → runs bash instead

# ENTRYPOINT only — always runs this
ENTRYPOINT ["nginx"]
# docker run myimage -g "daemon off;" → appends args

# Both — ENTRYPOINT is fixed, CMD provides default args
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
# docker run myimage → runs: nginx -g daemon off;
# docker run myimage -t → runs: nginx -t
```

### Multi-Stage Builds (Critical Topic)

```dockerfile
# Stage 1: Build
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production (only copies built artifacts)
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
USER node
CMD ["node", "dist/server.js"]
```

**Why multi-stage?** Final image only contains runtime artifacts — no build tools, source code, or dev dependencies. Result: much smaller, more secure images.

### .dockerignore

```
node_modules/
.git/
*.log
.env
dist/
coverage/
```

### Dockerfile Best Practices

1. **Order layers by change frequency** — put frequently changing layers last (COPY source code after installing dependencies)
2. **Minimize layers** — chain RUN commands with `&&`
3. **Clean up in same layer** — `RUN apt-get update && apt-get install -y pkg && rm -rf /var/lib/apt/lists/*`
4. **Use specific base image tags** — `node:18.17.1-alpine` not `node:latest`
5. **Run as non-root user** — security best practice
6. **Use multi-stage builds** — reduce final image size
7. **Use .dockerignore** — exclude unnecessary files

---

## Docker Networking

### Network Drivers

| Driver | Description | Use Case |
|---|---|---|
| **bridge** | Default. Creates virtual network on host | Single-host container communication |
| **host** | Container shares host network namespace | High performance, no network isolation |
| **none** | No networking | Completely isolated containers |
| **overlay** | Multi-host networking via VXLAN | Docker Swarm, multi-host deployments |
| **macvlan** | Container gets its own MAC address | Legacy apps needing direct network access |

### Bridge Network (Default)

```
Host Machine
┌─────────────────────────────────────┐
│  docker0 bridge (172.17.0.1)        │
│  ┌──────────────┐ ┌──────────────┐  │
│  │ Container A  │ │ Container B  │  │
│  │ 172.17.0.2   │ │ 172.17.0.3   │  │
│  └──────────────┘ └──────────────┘  │
└─────────────────────────────────────┘
```

### Custom Bridge Network (Recommended)

```bash
# Create network
docker network create --driver bridge mynetwork

# Run containers on same network
docker run -d --name db --network mynetwork postgres
docker run -d --name api --network mynetwork myapp

# Containers can resolve each other by name (DNS)
# api can ping db: ping db → 172.18.0.2
```

**Custom bridge vs default bridge**: Custom bridge networks provide automatic DNS resolution between containers by container name. Default bridge network only has IP-based communication.

### Network Commands

```bash
docker network ls                              # List networks
docker network create mynet                    # Create network
docker network inspect mynet                   # Inspect network
docker network connect mynet <container>       # Connect container to network
docker network disconnect mynet <container>    # Disconnect
docker network rm mynet                        # Remove network
docker network prune                           # Remove unused networks
```

### Port Mapping

```bash
# -p host_port:container_port
docker run -p 8080:80 nginx       # Map host 8080 to container 80
docker run -p 0.0.0.0:80:80 nginx # Bind to all interfaces
docker run -p 127.0.0.1:80:80 nginx # Localhost only
docker run -P nginx               # Auto-map all EXPOSE'd ports
```

---

## Docker Volumes & Storage

### Storage Types

```
┌────────────────────────────────────────────────────┐
│                   Container                        │
│  ┌──────────────────────────────────────────────┐  │
│  │        Writable Container Layer              │  │ ← ephemeral
│  └──────────────────────────────────────────────┘  │
└────────┬─────────────────────────┬─────────────────┘
         │                         │
┌────────▼──────────┐   ┌──────────▼──────────────┐
│   Docker Volume   │   │      Bind Mount          │
│ /var/lib/docker/  │   │  /host/path → /container │
│    volumes/...    │   │  /path                   │
└───────────────────┘   └─────────────────────────┘
```

| Storage | Managed By | Use Case |
|---|---|---|
| **Volumes** | Docker | Databases, persistent app data (recommended) |
| **Bind Mounts** | Host OS | Dev: live code reload, config injection |
| **tmpfs** | Memory | Sensitive data (secrets), high-speed temp storage |

### Volume Commands

```bash
# Create and use volumes
docker volume create mydata
docker run -d -v mydata:/var/lib/postgresql/data postgres
docker run -d --mount source=mydata,target=/data postgres  # explicit syntax

# Bind mount (development)
docker run -d -v $(pwd):/app node:18       # Mount current dir
docker run -d -v /host/path:/container/path nginx

# tmpfs mount
docker run -d --tmpfs /tmp nginx

# Volume management
docker volume ls
docker volume inspect mydata
docker volume rm mydata
docker volume prune                        # Remove unused volumes
```

### When to Use What

- **Volumes**: Databases, application state — Docker manages location, portable, easy to backup
- **Bind mounts**: Local development, config files — you control the path
- **tmpfs**: Secrets, cache — never written to disk

---

## Docker Compose

Docker Compose defines and runs **multi-container applications** using a YAML file.

### Complete docker-compose.yml Example

```yaml
version: '3.9'

# ─── Named Volumes ────────────────────────────────────────────
volumes:
  postgres_data:
  redis_data:

# ─── Named Networks ───────────────────────────────────────────
networks:
  frontend:
  backend:

# ─── Services ─────────────────────────────────────────────────
services:

  # Database
  db:
    image: postgres:15-alpine
    container_name: myapp_db
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Cache
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    networks:
      - backend

  # API Service
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
      target: production          # multi-stage target
    container_name: myapp_api
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://user:secret@db:5432/mydb
      REDIS_URL: redis://redis:6379
    depends_on:
      db:
        condition: service_healthy  # Wait for health check
      redis:
        condition: service_started
    networks:
      - frontend
      - backend
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M

  # Frontend
  web:
    build: ./frontend
    ports:
      - "80:80"
    depends_on:
      - api
    networks:
      - frontend
    restart: unless-stopped
```

### Docker Compose Commands

```bash
# Lifecycle
docker compose up                    # Start all services
docker compose up -d                 # Detached mode
docker compose up --build            # Rebuild images before start
docker compose down                  # Stop and remove containers/networks
docker compose down -v               # Also remove volumes
docker compose restart               # Restart all services

# Operations
docker compose ps                    # List services
docker compose logs                  # All logs
docker compose logs -f api           # Follow specific service logs
docker compose exec api bash         # Shell into service
docker compose run api npm test      # Run one-off command

# Build
docker compose build                 # Build all images
docker compose build api             # Build specific service
docker compose pull                  # Pull latest images

# Scale (without Swarm)
docker compose up -d --scale api=3  # Run 3 API instances
```

### depends_on Conditions

```yaml
depends_on:
  db:
    condition: service_healthy   # Wait for healthcheck to pass
  redis:
    condition: service_started   # Just wait for container start
  init:
    condition: service_completed_successfully  # Wait for one-shot job
```

---

## Docker Registry

### Types of Registries

| Registry | Description |
|---|---|
| **Docker Hub** | Default public registry (docker.io) |
| **AWS ECR** | Amazon Elastic Container Registry |
| **GCR** | Google Container Registry |
| **Azure ACR** | Azure Container Registry |
| **Harbor** | Self-hosted, open-source |
| **GitLab Registry** | Built into GitLab |

### Registry Operations

```bash
# Login
docker login                          # Docker Hub
docker login myregistry.azurecr.io   # Private registry

# Tag and push
docker tag myapp:latest myregistry.azurecr.io/myapp:1.0
docker push myregistry.azurecr.io/myapp:1.0

# Pull
docker pull myregistry.azurecr.io/myapp:1.0

# Run private registry locally
docker run -d -p 5000:5000 --name registry registry:2
docker tag myapp localhost:5000/myapp:1.0
docker push localhost:5000/myapp:1.0
```

---

## Docker Swarm vs Kubernetes

| Feature | Docker Swarm | Kubernetes |
|---|---|---|
| **Complexity** | Simple, built into Docker | Complex, steep learning curve |
| **Scaling** | Manual or basic auto-scale | Advanced auto-scaling (HPA, VPA) |
| **Load Balancing** | Built-in (simple) | Advanced (Ingress controllers) |
| **Storage** | Basic volume support | Rich storage classes, PVs, PVCs |
| **Networking** | Overlay networks | Advanced (CNI plugins) |
| **Self-healing** | Basic | Advanced |
| **Ecosystem** | Limited | Very large |
| **Use case** | Simple apps, small teams | Production, complex systems |

### Swarm Concepts

```bash
# Initialize swarm
docker swarm init --advertise-addr <IP>

# Add workers
docker swarm join --token <token> <manager-ip>:2377

# Deploy stack (like compose for swarm)
docker stack deploy -c docker-compose.yml myapp

# Manage services
docker service ls
docker service scale myapp_api=5
docker service update --image myapp:2.0 myapp_api
docker service rollback myapp_api
```

---

## Security

### Security Best Practices

**1. Use non-root user**
```dockerfile
RUN useradd -r -u 1001 appuser
USER appuser
```

**2. Read-only filesystem**
```bash
docker run --read-only nginx
```

**3. Drop Linux capabilities**
```bash
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx
```

**4. Scan images for vulnerabilities**
```bash
docker scout cve nginx:latest     # Docker Scout
trivy image nginx:latest           # Trivy scanner
```

**5. Use secrets management**
```bash
# Docker Swarm secrets
echo "mypassword" | docker secret create db_password -
docker service create --secret db_password myapp
```

**6. Limit resources**
```bash
docker run --memory="256m" --cpus="0.5" myapp
```

**7. Use trusted base images**
- Official images (nginx, node, postgres)
- Alpine variants (smaller attack surface)
- Distroless images (no shell, package manager)

### Container Isolation

```bash
# Seccomp profile
docker run --security-opt seccomp=profile.json nginx

# AppArmor
docker run --security-opt apparmor=docker-default nginx

# No new privileges
docker run --security-opt no-new-privileges nginx
```

---

## Performance & Best Practices

### Layer Caching Strategy

```dockerfile
# BAD — cache busted every time code changes
COPY . .
RUN npm install

# GOOD — dependencies cached separately from code
COPY package.json package-lock.json ./
RUN npm ci                    # Cache hit unless package.json changes
COPY . .                      # Only this layer rebuilds on code change
```

### Image Size Optimization

```bash
# Use alpine variants
FROM node:18-alpine       # ~130MB vs node:18 ~900MB

# Multi-stage builds (most impactful)
FROM node:18 AS builder   # Build tools, heavy
FROM node:18-alpine       # Only runtime, lightweight

# Minimize layers and clean up
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Check image size
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Analyze layers
docker history myimage
dive myimage               # Third-party tool for layer analysis
```

### Build Performance

```bash
# BuildKit (faster, better caching)
DOCKER_BUILDKIT=1 docker build .

# Build cache mounting (dependencies)
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Parallel stage builds
docker build --target builder .
```

---

## Common Interview Questions

### Q1: What is the difference between COPY and ADD?
**COPY** only copies files/directories from the build context. **ADD** can also extract tar archives and fetch remote URLs. Best practice: use **COPY** unless you specifically need ADD's extra features.

### Q2: What is a dangling image?
An image with no tag (shows as `<none>:<none>` in `docker images`). Created when you rebuild an image with the same tag — the old image loses its tag. Remove with `docker image prune`.

### Q3: How do containers communicate on the same host?
Via a **Docker network**. Containers on the same network can reach each other by container name (DNS resolution). By default, containers on separate networks cannot communicate.

### Q4: What happens when a container exits?
The container transitions to the `stopped` state. Its writable layer persists on disk until you `docker rm` it. Data in volumes persists beyond container deletion.

### Q5: How is Docker different from a VM?
Containers share the host OS kernel using **namespaces** (isolation) and **cgroups** (resource limits). VMs have a full OS including a kernel. Containers are faster to start, use less memory, but provide weaker isolation.

### Q6: What is the purpose of HEALTHCHECK?
It tells Docker how to test if a container is healthy. Docker/Swarm/Kubernetes uses this to determine if the container is ready to receive traffic and whether to restart it.

### Q7: How do you handle secrets in Docker?
- **Development**: Environment variables or `.env` files (not in image)
- **Production**: Docker Swarm secrets, Kubernetes Secrets, or external secret managers (Vault, AWS Secrets Manager)
- **Never**: Bake secrets into images (they end up in layers and registry)

### Q8: What is the difference between `docker stop` and `docker kill`?
- `docker stop`: Sends **SIGTERM** first, waits for graceful shutdown (default 10s), then SIGKILL
- `docker kill`: Sends **SIGKILL** immediately (or specified signal)

### Q9: What is the difference between CMD and ENTRYPOINT?
- **CMD**: Default command/args, can be completely overridden at runtime
- **ENTRYPOINT**: Fixed executable, cannot be easily overridden (use `--entrypoint` flag)
- **Together**: ENTRYPOINT is the executable, CMD provides default arguments

### Q10: How do you reduce Docker image size?
1. Use Alpine base images
2. Multi-stage builds
3. Chain RUN commands and clean up
4. Use .dockerignore
5. Remove dev dependencies in production

---

## Quick Reference Summary

```
Image Commands:          Container Commands:       Network Commands:
docker build -t .        docker run -d -p          docker network create
docker pull              docker ps / ps -a          docker network ls
docker push              docker exec -it bash       docker network connect
docker images            docker logs -f
docker rmi               docker stop / kill         Volume Commands:
docker image prune       docker rm                  docker volume create
                         docker inspect             docker volume ls
                         docker stats               docker volume rm
                         docker cp
```

---

*Last updated: 2026-06-05*
