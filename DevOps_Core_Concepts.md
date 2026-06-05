# DevOps Core Concepts - Interview Preparation Study Guide

---

## Table of Contents
1. [What is DevOps?](#what-is-devops)
2. [CI/CD Pipelines](#cicd-pipelines)
3. [Version Control & Git Workflows](#version-control--git-workflows)
4. [Infrastructure as Code (IaC)](#infrastructure-as-code-iac)
5. [Configuration Management](#configuration-management)
6. [Container Orchestration](#container-orchestration)
7. [Monitoring & Observability](#monitoring--observability)
8. [Logging](#logging)
9. [Site Reliability Engineering (SRE)](#site-reliability-engineering-sre)
10. [Security in DevOps (DevSecOps)](#security-in-devops-devsecops)
11. [Artifact Management](#artifact-management)
12. [Cloud Platforms Overview](#cloud-platforms-overview)
13. [Common Interview Questions](#common-interview-questions)

---

## What is DevOps?

DevOps is a **culture, philosophy, and set of practices** that unifies software development (Dev) and IT operations (Ops) to shorten the development lifecycle and deliver high-quality software continuously.

### DevOps Principles (CALMS Framework)

| Principle | Description |
|---|---|
| **Culture** | Shared responsibility, collaboration, blameless post-mortems |
| **Automation** | Automate repetitive tasks (testing, deployment, provisioning) |
| **Lean** | Eliminate waste, small batch sizes, continuous improvement |
| **Measurement** | Measure everything — performance, feedback, SLOs |
| **Sharing** | Share knowledge, tools, and feedback loops |

### DevOps Lifecycle

```
Plan → Code → Build → Test → Release → Deploy → Operate → Monitor
  ←────────────────── Feedback Loop ──────────────────────────────
```

### Key Metrics (DORA Metrics)

| Metric | Elite | High | Medium | Low |
|---|---|---|---|---|
| **Deployment Frequency** | Multiple/day | Daily/weekly | Weekly/monthly | Monthly/quarterly |
| **Lead Time for Changes** | < 1 hour | 1 day - 1 week | 1 week - 1 month | > 6 months |
| **Change Failure Rate** | 0-15% | 16-30% | 16-30% | 46-60% |
| **MTTR (Recovery Time)** | < 1 hour | < 1 day | 1 day - 1 week | > 6 months |

---

## CI/CD Pipelines

### Continuous Integration (CI)
Developers **frequently merge code** into a shared repository. Each merge triggers automated builds and tests.

**CI Goals**:
- Detect integration bugs early
- Keep the main branch always deployable
- Eliminate "integration hell"

### Continuous Delivery (CD)
Code is **automatically built, tested, and prepared for release** to production. Deployment to production requires manual approval.

### Continuous Deployment
Code that passes all automated tests is **automatically deployed to production** — no human approval needed.

```
CI:   Code → Build → Test → [artifact ready]
CD:   CI output → Staging deploy → Acceptance tests → [ready to release]
Continuous Deployment: CI output → Staging → Production [automatic]
```

### Pipeline Stages

```yaml
# Example: Generic CI/CD pipeline stages
stages:
  - validate    # Lint, static analysis, security scan
  - build       # Compile, build Docker image
  - test        # Unit tests, integration tests
  - scan        # SAST, DAST, dependency vulnerability scan
  - staging     # Deploy to staging environment
  - acceptance  # End-to-end tests, smoke tests
  - production  # Deploy to production (blue/green or canary)
  - verify      # Post-deploy health checks
```

### Popular CI/CD Tools

| Tool | Type | Key Feature |
|---|---|---|
| **Jenkins** | Self-hosted | Most flexible, massive plugin ecosystem |
| **GitHub Actions** | Cloud | Native GitHub integration, free for open source |
| **GitLab CI** | Cloud/Self-hosted | Integrated with GitLab, powerful |
| **CircleCI** | Cloud | Fast, easy parallelism |
| **ArgoCD** | GitOps for K8s | Kubernetes-native, GitOps model |
| **Tekton** | K8s-native | Cloud-native pipeline, vendor-neutral |
| **AWS CodePipeline** | Cloud | Deep AWS integration |

### GitHub Actions Example

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  IMAGE_NAME: myapp
  REGISTRY: ghcr.io

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run unit tests
        run: npm test -- --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ github.repository }}/${{ env.IMAGE_NAME }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to staging
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ needs.build.outputs.image-tag }}

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production    # Requires manual approval
    steps:
      - name: Deploy to production
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ needs.build.outputs.image-tag }}
```

### Deployment Strategies

| Strategy | Description | Downtime | Risk | Rollback |
|---|---|---|---|---|
| **Recreate** | Stop old, start new | Yes | High | Slow |
| **Rolling** | Replace instances gradually | No | Medium | Moderate |
| **Blue/Green** | Two identical envs, switch traffic | No | Low | Instant |
| **Canary** | Route small % traffic to new version | No | Very Low | Instant |
| **A/B Testing** | Route by user segment | No | Very Low | Instant |

#### Blue/Green Deployment

```
                    ┌─── Blue (v1) ───┐ ← Idle
Load Balancer ──────┤
                    └─── Green (v2) ──┘ ← Active (100% traffic)

Deploy v3 to Blue:
                    ┌─── Blue (v3) ───┐ ← Switch traffic here
Load Balancer ──────┤
                    └─── Green (v2) ──┘ ← Keep for rollback
```

#### Canary Deployment

```
Request → Load Balancer → 95% → v1 (stable)
                        → 5%  → v2 (canary)

After validation:
                        → 0%  → v1
                        → 100% → v2
```

---

## Version Control & Git Workflows

### Git Branching Strategies

#### GitFlow

```
main ─────────────────────────────────────────● v1.0
       ↑                                      ↑
hotfix/fix-bug                          release/1.0
                                              ↑
develop ──────────────────────────────────────●──────
              ↑           ↑           ↑
         feature/A   feature/B   feature/C
```

**Branches**:
- `main`: Production-ready code
- `develop`: Integration branch
- `feature/*`: New features (branch from develop)
- `release/*`: Release preparation
- `hotfix/*`: Emergency production fixes

#### Trunk-Based Development (Preferred for CI/CD)

```
main (trunk) ●────●────●────●────●────●  (deployable at all times)
              ↑    ↑    ↑    ↑
          short-lived feature branches (< 1-2 days)
```

- All developers push to main frequently
- Feature flags hide incomplete features
- Enforces CI, reduces integration problems

#### GitHub Flow

```
main ──●────────────────────────●───────
        ↓                       ↑
     feature/xyz ──●──●──●── PR & merge
```

Simple: branch from main, PR, review, merge.

### Key Git Commands for DevOps

```bash
# Tagging releases
git tag -a v1.2.3 -m "Release 1.2.3"
git push origin v1.2.3

# Cherry-pick a fix to production
git cherry-pick <commit-hash>

# Revert a bad commit
git revert <commit-hash>

# Stash work in progress
git stash push -m "WIP: feature X"
git stash pop

# View commit graph
git log --oneline --graph --all

# Find when a bug was introduced
git bisect start
git bisect bad                   # Current is bad
git bisect good v1.2.0           # This was good

# Squash commits before PR
git rebase -i HEAD~3
```

---

## Infrastructure as Code (IaC)

IaC means managing and provisioning infrastructure through **machine-readable configuration files** rather than manual processes.

### Benefits
- **Reproducibility**: Same config = same infrastructure, every time
- **Version Control**: Infrastructure changes tracked in git
- **Automation**: No manual clicks in cloud consoles
- **Documentation**: Config IS the documentation

### IaC Tools Comparison

| Tool | Type | Cloud | Language |
|---|---|---|---|
| **Terraform** | Provisioning | Multi-cloud | HCL |
| **AWS CloudFormation** | Provisioning | AWS only | YAML/JSON |
| **Pulumi** | Provisioning | Multi-cloud | Python/TS/Go |
| **Ansible** | Config Management | Multi-cloud | YAML |
| **Chef/Puppet** | Config Management | Multi-cloud | Ruby DSL |

### Terraform Core Concepts

```hcl
# provider.tf — Define cloud provider
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # Remote state (shared state)
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/main.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"  # State locking
    encrypt        = true
  }
}

provider "aws" {
  region = var.region
}

# variables.tf — Input variables
variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

# main.tf — Resources
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name        = "main-vpc"
    Environment = var.environment
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.public.id

  tags = {
    Name = "web-server"
  }
}

# Data sources (read existing resources)
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# outputs.tf — Output values
output "instance_public_ip" {
  value = aws_instance.web.public_ip
}
```

### Terraform Workflow

```bash
terraform init        # Download providers, initialize backend
terraform validate    # Check syntax
terraform plan        # Preview changes (dry run)
terraform apply       # Apply changes
terraform destroy     # Destroy all resources

terraform fmt         # Format code
terraform state list  # List managed resources
terraform import aws_instance.web i-1234567890  # Import existing resource
terraform workspace new staging  # Create workspace
```

### Terraform State

- **State file** (`terraform.tfstate`): Maps config to real infrastructure
- **Remote state**: Store in S3/GCS for team use
- **State locking**: DynamoDB (AWS) prevents concurrent modifications
- **Never edit state manually**: Use `terraform state` commands

---

## Configuration Management

### Ansible

Ansible automates configuration management, application deployment, and orchestration.

**Key concepts**:
- **Agentless**: Uses SSH, no agent installed on target
- **Idempotent**: Running playbook multiple times gives same result
- **Inventory**: List of managed hosts
- **Playbook**: YAML file defining automation tasks
- **Role**: Reusable collection of tasks, templates, files

```yaml
# inventory.yml
all:
  children:
    webservers:
      hosts:
        web1: { ansible_host: 10.0.1.10 }
        web2: { ansible_host: 10.0.1.11 }
    databases:
      hosts:
        db1: { ansible_host: 10.0.2.10 }

# playbook.yml
---
- name: Configure web servers
  hosts: webservers
  become: yes               # sudo
  vars:
    nginx_port: 80

  tasks:
    - name: Install nginx
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Copy config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx

    - name: Ensure nginx is running
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

```bash
# Ansible commands
ansible -i inventory.yml webservers -m ping          # Test connectivity
ansible-playbook -i inventory.yml playbook.yml       # Run playbook
ansible-playbook playbook.yml --check                # Dry run
ansible-playbook playbook.yml --tags "install"       # Run specific tags
ansible-vault encrypt secrets.yml                    # Encrypt secrets
ansible-galaxy install geerlingguy.nginx             # Install role
```

---

## Container Orchestration

### Kubernetes Core Concepts

```
Cluster
├── Control Plane (Master)
│   ├── API Server          ← Entry point for all K8s commands
│   ├── etcd                ← Distributed key-value store (cluster state)
│   ├── Scheduler           ← Decides which node runs a pod
│   └── Controller Manager  ← Ensures desired state = actual state
│
└── Worker Nodes
    ├── kubelet             ← Agent that runs pods
    ├── kube-proxy          ← Network rules for services
    └── Container Runtime   ← Docker/containerd
```

### Kubernetes Objects

```yaml
# Pod — Smallest deployable unit
apiVersion: v1
kind: Pod
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
    livenessProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 80

---
# Deployment — Manages replica sets
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0

---
# Service — Stable network endpoint
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP     # ClusterIP, NodePort, LoadBalancer

---
# HPA — Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### kubectl Cheat Sheet

```bash
# Context
kubectl config get-contexts
kubectl config use-context prod-cluster

# Resources
kubectl get pods -n production
kubectl get pods -o wide          # With node info
kubectl describe pod myapp-xyz
kubectl logs myapp-xyz -f         # Follow logs
kubectl exec -it myapp-xyz -- bash

# Deploy
kubectl apply -f deployment.yaml
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp       # Rollback
kubectl set image deployment/myapp myapp=myapp:2.0

# Scale
kubectl scale deployment myapp --replicas=5

# Debug
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl top pods
kubectl top nodes
kubectl port-forward pod/myapp-xyz 8080:80
```

### GitOps with ArgoCD

GitOps: **Git is the single source of truth for infrastructure and application state**.

```
Developer → Push to Git repo → ArgoCD detects change
                                     ↓
                            Syncs to Kubernetes cluster
                                     ↓
                            Cluster matches Git state
```

**Principles**:
1. Declarative: Describe desired state in Git
2. Versioned: Git history = audit trail
3. Automatic: Sync is automated
4. Continuously reconciled: Drift is corrected automatically

---

## Monitoring & Observability

### The Three Pillars of Observability

| Pillar | Description | Tools |
|---|---|---|
| **Metrics** | Numeric time-series data (CPU, latency, error rate) | Prometheus, Datadog, CloudWatch |
| **Logs** | Timestamped event records | ELK Stack, Loki, Splunk |
| **Traces** | Request flow across services | Jaeger, Zipkin, AWS X-Ray |

### Metrics with Prometheus + Grafana

```
Application → /metrics endpoint → Prometheus scrapes → Grafana visualizes
                                        ↓
                                  AlertManager → PagerDuty/Slack
```

```yaml
# prometheus.yml — Scrape config
scrape_configs:
  - job_name: 'myapp'
    scrape_interval: 15s
    static_configs:
      - targets: ['myapp:3000']

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

```yaml
# AlertManager rule
groups:
  - name: myapp.rules
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"
```

### Key Metrics to Monitor

**RED Method (for services)**:
- **Rate**: Requests per second
- **Errors**: Error rate
- **Duration**: Latency (p50, p95, p99)

**USE Method (for resources)**:
- **Utilization**: % of time resource is busy
- **Saturation**: Queue depth / backlog
- **Errors**: Error count

### SLI, SLO, SLA

| Term | Definition | Example |
|---|---|---|
| **SLI** | Service Level Indicator — metric that measures a service behavior | 99th percentile latency = 250ms |
| **SLO** | Service Level Objective — target for the SLI | 99% of requests < 500ms |
| **SLA** | Service Level Agreement — contract with consequences | 99.9% uptime or 10% refund |
| **Error Budget** | 1 - SLO = how much failure is allowed | 0.1% = 43.8 min/month |

---

## Logging

### ELK Stack

```
Application → Filebeat/Fluentd → Logstash → Elasticsearch → Kibana
(log files)   (log shipper)    (transform)  (store/index)  (visualize)
```

### Structured Logging (JSON)

```json
{
  "timestamp": "2026-06-05T10:30:00Z",
  "level": "ERROR",
  "service": "payment-api",
  "traceId": "abc123",
  "userId": "user-456",
  "message": "Payment processing failed",
  "error": "Connection timeout",
  "duration_ms": 5023
}
```

**Why structured logging?** Machines can parse, filter, and query JSON efficiently. Plain text requires fragile regex parsing.

### Log Levels

| Level | Use Case |
|---|---|
| `TRACE` | Very detailed debugging, typically disabled in production |
| `DEBUG` | Development debugging |
| `INFO` | Normal operational events (startup, user actions) |
| `WARN` | Unexpected but handled situations |
| `ERROR` | Failures that need attention |
| `FATAL/CRITICAL` | System cannot continue |

---

## Site Reliability Engineering (SRE)

SRE is Google's approach to DevOps: apply software engineering principles to operations.

### SRE Principles

- **Eliminate toil**: Manual, repetitive work that scales with load — automate it
- **Error budgets**: If SLO is 99.9%, the 0.1% error budget allows for risk taking
- **Blameless post-mortems**: Focus on systems, not people
- **Gradual rollouts**: Canary deployments, feature flags
- **Capacity planning**: Predict demand and provision ahead

### Incident Management

```
Alert fires → On-call engineer notified
     ↓
Triage: Severity classification (P1/P2/P3)
     ↓
Incident channel opened (Slack/Teams)
     ↓
Mitigation: Restore service ASAP (rollback, scale, redirect)
     ↓
Resolution: Root cause fixed
     ↓
Post-mortem (within 48-72 hours)
     ↓
Action items: Prevent recurrence
```

### Toil vs Engineering Work

| Toil | Engineering Work |
|---|---|
| Manual server restarts | Write auto-restart script |
| SSH into prod to check logs | Set up centralized logging |
| Scale up manually when traffic spikes | Implement HPA in Kubernetes |
| Manually approve deployments | Automated canary deployment |

---

## Security in DevOps (DevSecOps)

### Shift Left Security

Move security checks **earlier** in the development lifecycle — fix bugs when they're cheapest to fix.

```
Developer IDE → Code Review → CI Build → Staging → Production
    ↑               ↑            ↑           ↑           ↑
  Linting         SAST         DAST      Pen testing  Runtime
  secrets       dep scan     container   compliance   monitoring
  detection     code scan      scan
```

### Security Scanning Types

| Type | Description | Tools |
|---|---|---|
| **SAST** | Static Application Security Testing — scan source code | SonarQube, Semgrep, Checkmarx |
| **DAST** | Dynamic Application Security Testing — attack running app | OWASP ZAP, Burp Suite |
| **SCA** | Software Composition Analysis — scan dependencies | Snyk, Dependabot, OWASP Dependency-Check |
| **Container Scanning** | Scan images for OS/package CVEs | Trivy, Clair, Snyk Container |
| **Secret Detection** | Find leaked credentials in code | GitLeaks, truffleHog, git-secrets |
| **IaC Scanning** | Find misconfigurations in Terraform/CloudFormation | Checkov, tfsec, KICS |

### Secrets Management

```
# Never do this:
DATABASE_PASSWORD=mysecretpassword  # In code, dockerfile, or git

# Use:
AWS Secrets Manager        → aws secretsmanager get-secret-value
HashiCorp Vault            → vault kv get secret/myapp/db
Kubernetes Secrets         → kubectl create secret generic
Environment variables      → injected at runtime by orchestrator
```

### Supply Chain Security

```bash
# Sign images with cosign
cosign sign myregistry/myapp:1.0

# Verify before deployment
cosign verify myregistry/myapp:1.0 --certificate-identity-regexp=...

# Generate SBOM (Software Bill of Materials)
syft myregistry/myapp:1.0 -o cyclonedx-json > sbom.json

# Check SBOM for vulnerabilities
grype sbom:sbom.json
```

---

## Artifact Management

### What are Artifacts?
Build outputs: Docker images, JAR files, npm packages, Helm charts, zip archives.

### Artifact Repositories

| Tool | Artifact Types |
|---|---|
| **JFrog Artifactory** | Universal (all types) |
| **Nexus Repository** | Maven, npm, Docker, PyPI |
| **AWS ECR** | Docker images |
| **GitHub Packages** | Docker, npm, Maven, NuGet |
| **Harbor** | Docker images (self-hosted) |

### Versioning Strategies

**Semantic Versioning (SemVer)**: `MAJOR.MINOR.PATCH`
- `1.0.0` → `1.0.1` (patch: bug fix)
- `1.0.0` → `1.1.0` (minor: new feature, backward compatible)
- `1.0.0` → `2.0.0` (major: breaking change)

**Image Tagging**:
```bash
myapp:1.2.3          # SemVer (production)
myapp:1.2            # Minor version alias
myapp:latest         # Latest release (risky in prod)
myapp:git-abc1234    # Git commit SHA (most precise)
myapp:main-20260605  # Branch + date
```

---

## Cloud Platforms Overview

### AWS Core Services for DevOps

| Category | Service | Use Case |
|---|---|---|
| **Compute** | EC2, ECS, EKS, Lambda | Run applications |
| **Containers** | ECR, ECS, EKS | Container registry and orchestration |
| **CI/CD** | CodePipeline, CodeBuild, CodeDeploy | Build and deploy |
| **IaC** | CloudFormation, CDK | Infrastructure automation |
| **Monitoring** | CloudWatch, X-Ray | Metrics, logs, traces |
| **Secrets** | Secrets Manager, Parameter Store | Configuration and secrets |
| **Networking** | VPC, ALB, Route 53, CloudFront | Network and DNS |
| **Storage** | S3, EFS, EBS | Object, file, block storage |

### AWS Well-Architected Framework Pillars

1. **Operational Excellence**: Run and monitor systems to deliver business value
2. **Security**: Protect data, systems, and assets
3. **Reliability**: Recover from failures and meet demand
4. **Performance Efficiency**: Use resources efficiently
5. **Cost Optimization**: Avoid unnecessary costs
6. **Sustainability**: Minimize environmental impact

---

## Common Interview Questions

### Q1: What is the difference between CI and CD?
**CI (Continuous Integration)**: Automatically build and test code on every commit to detect integration issues early. **CD (Continuous Delivery)**: Extends CI by automatically preparing the release for deployment — requires manual approval for production. **Continuous Deployment**: Fully automated — passes all tests → goes to production with no human approval.

### Q2: What is Infrastructure as Code and why is it important?
IaC manages infrastructure through version-controlled configuration files. Benefits: reproducibility (same code = same infra), auditability (git history), automation (no manual clicks), reduced human error, and the ability to test infrastructure changes in lower environments first.

### Q3: What is the difference between blue/green and canary deployments?
**Blue/Green**: Maintain two identical environments; switch all traffic at once. Zero downtime, instant rollback, but costs double the resources. **Canary**: Gradually shift a small percentage of traffic to the new version (e.g., 5%). Monitor metrics; roll forward or back. Lower risk, resource-efficient, but slower to deploy fully.

### Q4: What are DORA metrics?
DORA (DevOps Research and Assessment) metrics measure software delivery performance: **Deployment Frequency** (how often you deploy), **Lead Time for Changes** (time from commit to production), **Change Failure Rate** (% of deployments causing issues), and **MTTR** (time to recover from failures). Elite teams deploy multiple times per day with < 1 hour lead time.

### Q5: What is a post-mortem and why is it blameless?
A post-mortem analyzes an incident to find root causes and prevent recurrence. It's blameless because incidents are systemic failures, not individual ones — blaming people creates a cover-up culture where problems hide rather than surface. Focus on processes, systems, and tooling improvements.

### Q6: What is GitOps?
GitOps uses Git as the single source of truth for both application code and infrastructure configuration. Changes are made via PRs, merged into Git, and an operator (like ArgoCD) automatically reconciles the actual cluster state to match Git. Benefits: audit trail, rollback by reverting Git commits, security (cluster pulls, not push).

### Q7: What is the difference between Ansible and Terraform?
**Terraform** is a **provisioning** tool — it creates and manages cloud infrastructure (VMs, networks, databases). **Ansible** is a **configuration management** tool — it configures existing servers (install packages, edit files, start services). They're complementary: Terraform provisions the server, Ansible configures it.

### Q8: What is an error budget?
If your SLO is 99.9% availability, your error budget is 0.1% = ~43.8 minutes/month of allowed downtime. When the error budget is healthy, you can move fast and take risks. When depleted, you slow down and prioritize reliability. It creates shared language between dev and ops.

### Q9: What is the difference between liveness and readiness probes in Kubernetes?
**Liveness probe**: Is the container still running? If it fails, Kubernetes restarts the container. Used to recover from deadlocks. **Readiness probe**: Is the container ready to serve traffic? If it fails, Kubernetes removes the pod from the service's endpoints (no traffic). Used during startup and when temporarily overloaded.

### Q10: How do you handle secrets in a CI/CD pipeline?
Never commit secrets to Git. Store in a secret manager (AWS Secrets Manager, HashiCorp Vault, GitHub/GitLab Secrets). Inject into the pipeline as environment variables at runtime. Use short-lived credentials where possible (IAM roles, OIDC). Scan code for accidental secret commits with tools like GitLeaks.

---

## Quick Reference Summary

```
CI/CD Tools:           IaC:                   Monitoring:
Jenkins                Terraform              Prometheus
GitHub Actions         CloudFormation         Grafana
GitLab CI              Ansible                ELK Stack
ArgoCD                 Pulumi                 Datadog

Container:             Security:              DORA Metrics:
Docker                 SAST (Semgrep)         Deploy Frequency
Kubernetes             DAST (ZAP)             Lead Time
Helm                   Trivy (containers)     Change Failure Rate
Istio                  Vault (secrets)        MTTR
```

---

*Last updated: 2026-06-05*
