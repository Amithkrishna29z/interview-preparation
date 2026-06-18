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
12. [Common Interview Questions](#common-interview-questions)
13. [Quick Reference Summary](#quick-reference-summary)

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
| Use case | Full OS isolation | App isolation |

**Analogy**: VMs are like separate houses (each with their own foundation). Containers are like apartments — they share infrastructure but are isolated from each other.

### Why Docker?
- **"Works on my machine" problem**: Eliminates environment inconsistencies
- **Microservices**: Each service runs in its own container
- **CI/CD**: Consistent build and deploy environments
- **Resource efficiency**: Run more workloads on the same hardware

---

## Core Architecture

```
Docker Client (CLI)  →  REST API  →  Docker Daemon (dockerd)
                                          │
                               ┌──────────┴──────────┐
                          containerd             Docker Registry
                               │
                             runc (OCI)
                        (creates containers)
```

### Key Components

| Component | Role |
|---|---|
| **Docker Client** | CLI tool that sends commands to daemon |
| **Docker Daemon** | Background service that manages Docker objects |
| **containerd** | Container runtime (manages container lifecycle) |
| **runc** | Low-level runtime that creates/runs containers |
| **Docker Registry** | Stores and distributes Docker images |

### How Docker Uses Linux Primitives
- **Namespaces**: Isolate processes (PID, Network, Mount, UTS, IPC, User)
- **cgroups**: Limit resource usage (CPU, memory, I/O)
- **OverlayFS**: Layered image file system

---

## Images

A Docker image is a **read-only template** used to create containers — a stack of layers, each representing a filesystem change.

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

Each Dockerfile instruction creates a new layer. Layers are cached and shared — this is why Docker is space-efficient.

### Image Commands

```bash
docker images                          # List local images
docker pull nginx:latest               # Pull from registry
docker push myrepo/myapp:1.0           # Push to registry
docker build -t myapp:1.0 .            # Build from Dockerfile
docker rmi myapp:1.0                   # Remove image
docker image prune                     # Remove dangling images
docker history nginx                   # Show image layers
docker tag nginx myrepo/nginx:custom   # Tag an image
```

### Image Naming Convention

```
[registry]/[username]/[image-name]:[tag]

docker.io/library/nginx:latest         # Official Docker Hub image
gcr.io/myproject/myapp:v1.2.3          # Google Container Registry
```

---

## Containers

A container is a **running instance of an image** — an isolated process with its own filesystem, network, and process space.

```
Created → Running → Paused → Running → Stopped → Removed
```

### Container Commands

```bash
docker run -d -p 8080:80 --name web nginx   # Detached, port mapped, named
docker run -it ubuntu bash                  # Interactive terminal
docker run --rm ubuntu echo "hello"         # Auto-remove after exit

docker ps                    # Running containers
docker ps -a                 # All containers
docker stop <id>             # Graceful stop (SIGTERM → SIGKILL)
docker kill <id>             # Immediate stop (SIGKILL)
docker rm <id>               # Remove stopped container
docker exec -it <id> bash    # Shell into running container
docker logs -f <id>          # Follow logs
docker stats                 # Real-time resource usage
docker inspect <id>          # Full container metadata (JSON)
```

### Resource Limits

```bash
docker run -d --memory="512m" --cpus="1.5" nginx
```

---

## Dockerfile

A Dockerfile is a text file with instructions to build a Docker image.

```dockerfile
FROM node:18-alpine AS builder      # Base image (alpine = smaller)
LABEL maintainer="team@company.com"

ENV NODE_ENV=production             # Persists in container
ARG VERSION=1.0                     # Build-time only, not in final image

WORKDIR /app                        # Set working directory

COPY package.json ./                # Copy dependency manifest first (caching)
RUN npm ci                          # Install dependencies
COPY . .                            # Copy source (after deps for cache efficiency)

EXPOSE 3000                         # Documentation only — doesn't publish port

RUN useradd -m appuser
USER appuser                        # Run as non-root (security best practice)

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

ENTRYPOINT ["node"]                 # Fixed command
CMD ["server.js"]                   # Default args (overridable at runtime)
```

### COPY vs ADD

| Instruction | Use Case |
|---|---|
| `COPY` | Simple file/directory copy — **prefer this** |
| `ADD` | Needs auto-extraction of `.tar.gz` or URL download |

### ENTRYPOINT vs CMD

```dockerfile
# CMD only — can be fully overridden at runtime
CMD ["nginx", "-g", "daemon off;"]

# Both — ENTRYPOINT is fixed, CMD provides default args
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
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

# Stage 2: Production — only runtime artifacts
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
USER node
CMD ["node", "dist/server.js"]
```

**Why multi-stage?** Final image has no build tools, source code, or dev dependencies — much smaller and more secure.

### .dockerignore

```
node_modules/
.git/
*.log
.env
dist/
```

### Dockerfile Best Practices

1. **Order layers by change frequency** — copy dependency files before source code
2. **Minimize layers** — chain `RUN` commands with `&&`
3. **Clean up in same layer** — `RUN apt-get update && apt-get install -y pkg && rm -rf /var/lib/apt/lists/*`
4. **Use specific tags** — `node:18.17.1-alpine` not `node:latest`
5. **Run as non-root user**
6. **Use multi-stage builds**
7. **Use .dockerignore**

---

## Docker Networking

### Network Drivers

| Driver | Description | Use Case |
|---|---|---|
| **bridge** | Default virtual network on host | Single-host container communication |
| **host** | Container shares host network | High performance, no network isolation |
| **none** | No networking | Completely isolated containers |
| **overlay** | Multi-host networking via VXLAN | Docker Swarm, multi-host deployments |

### Custom Bridge Network (Recommended)

```bash
docker network create --driver bridge mynetwork
docker run -d --name db --network mynetwork postgres
docker run -d --name api --network mynetwork myapp
# Containers can resolve each other by name (DNS): ping db
```

**Custom bridge vs default bridge**: Custom networks provide automatic DNS resolution by container name. Default bridge only has IP-based communication.

### Network Commands

```bash
docker network ls
docker network create mynet
docker network inspect mynet
docker network connect mynet <container>
docker network rm mynet
```

### Port Mapping

```bash
docker run -p 8080:80 nginx           # Map host 8080 to container 80
docker run -p 127.0.0.1:80:80 nginx  # Localhost only
docker run -P nginx                   # Auto-map all EXPOSE'd ports
```

---

## Docker Volumes & Storage

| Storage | Managed By | Use Case |
|---|---|---|
| **Volumes** | Docker | Databases, persistent app data (recommended) |
| **Bind Mounts** | Host OS | Dev: live code reload, config injection |
| **tmpfs** | Memory | Sensitive data, high-speed temp storage |

### Volume Commands

```bash
docker volume create mydata
docker run -d -v mydata:/var/lib/postgresql/data postgres   # Named volume
docker run -d -v $(pwd):/app node:18                        # Bind mount
docker run -d --tmpfs /tmp nginx                            # tmpfs

docker volume ls
docker volume inspect mydata
docker volume rm mydata
docker volume prune
```

---

## Docker Compose

Docker Compose defines and runs **multi-container applications** using a YAML file.

```yaml
version: '3.9'

volumes:
  postgres_data:

networks:
  backend:

services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://user:secret@db:5432/mydb
    depends_on:
      db:
        condition: service_healthy
    networks:
      - backend
    restart: unless-stopped
```

### Docker Compose Commands

```bash
docker compose up -d             # Start all services (detached)
docker compose up --build        # Rebuild images before start
docker compose down              # Stop and remove containers/networks
docker compose down -v           # Also remove volumes
docker compose ps                # List services
docker compose logs -f api       # Follow specific service logs
docker compose exec api bash     # Shell into service
docker compose up -d --scale api=3  # Run 3 API instances
```

### depends_on Conditions

```yaml
depends_on:
  db:
    condition: service_healthy              # Wait for healthcheck to pass
  redis:
    condition: service_started             # Wait for container start only
  init:
    condition: service_completed_successfully
```

---

## Docker Registry

| Registry | Description |
|---|---|
| **Docker Hub** | Default public registry (docker.io) |
| **AWS ECR** | Amazon Elastic Container Registry |
| **GCR** | Google Container Registry |
| **Azure ACR** | Azure Container Registry |
| **Harbor** | Self-hosted, open-source |

```bash
docker login
docker tag myapp:latest myregistry.azurecr.io/myapp:1.0
docker push myregistry.azurecr.io/myapp:1.0
docker pull myregistry.azurecr.io/myapp:1.0
```

---

## Docker Swarm vs Kubernetes

| Feature | Docker Swarm | Kubernetes |
|---|---|---|
| **Complexity** | Simple, built into Docker | Complex, steep learning curve |
| **Scaling** | Basic | Advanced auto-scaling (HPA, VPA) |
| **Load Balancing** | Built-in (simple) | Advanced (Ingress controllers) |
| **Self-healing** | Basic | Advanced |
| **Use case** | Simple apps, small teams | Production, complex systems |

---

## Security

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
docker scout cve nginx:latest
trivy image nginx:latest
```

**5. Use trusted base images** — official images, Alpine variants, or distroless images

**6. Never bake secrets into images** — they end up in layers and the registry

---

## Common Interview Questions

### Q1: What is the difference between COPY and ADD?
**COPY** only copies files from the build context. **ADD** can also extract tar archives and fetch remote URLs. Prefer **COPY** unless you need ADD's extra features.

### Q2: What is a dangling image?
An image with no tag (`<none>:<none>` in `docker images`). Created when you rebuild with the same tag — the old image loses its tag. Remove with `docker image prune`.

### Q3: How do containers communicate on the same host?
Via a **Docker network**. Containers on the same custom bridge network reach each other by container name (DNS). Containers on different networks cannot communicate by default.

### Q4: What happens when a container exits?
The container moves to `stopped` state. Its writable layer stays on disk until you `docker rm` it. Data in volumes persists beyond container deletion.

### Q5: How is Docker different from a VM?
Containers share the host OS kernel using **namespaces** (isolation) and **cgroups** (resource limits). VMs run a full OS including kernel. Containers start faster and use less memory, but provide weaker isolation.

### Q6: What is the purpose of HEALTHCHECK?
It tells Docker how to test if a container is healthy. Docker/Swarm/Kubernetes uses this to decide if the container is ready for traffic and whether to restart it.

### Q7: How do you handle secrets in Docker?
- **Development**: Environment variables or `.env` files (not in the image)
- **Production**: Docker Swarm secrets, Kubernetes Secrets, or Vault/AWS Secrets Manager
- **Never**: Bake secrets into images — they're visible in layers and the registry

### Q8: What is the difference between `docker stop` and `docker kill`?
`docker stop` sends **SIGTERM** first, waits for graceful shutdown (10s), then sends SIGKILL. `docker kill` sends **SIGKILL** immediately.

### Q9: What is the difference between CMD and ENTRYPOINT?
**CMD** provides default args and can be fully overridden at runtime. **ENTRYPOINT** is the fixed executable. Together: ENTRYPOINT is the executable, CMD provides its default arguments.

### Q10: How do you reduce Docker image size?
Use Alpine base images, multi-stage builds, chain and clean up RUN commands, use .dockerignore, and remove dev dependencies in production.

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
```

---

*Last updated: 2026-06-18*
