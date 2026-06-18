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

### CI vs CD vs Continuous Deployment

**CI (Continuous Integration)**: Developers frequently merge code; each merge triggers automated builds and tests. Goal: detect bugs early, keep main always deployable.

**CD (Continuous Delivery)**: Extends CI — code is automatically built, tested, and prepared for release. Deployment to production requires manual approval.

**Continuous Deployment**: Code that passes all tests is automatically deployed to production — no human approval needed.

```
CI:   Code → Build → Test → [artifact ready]
CD:   CI output → Staging deploy → Acceptance tests → [ready to release]
Continuous Deployment: CI output → Staging → Production [automatic]
```

### Pipeline Stages

```yaml
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
| **AWS CodePipeline** | Cloud | Deep AWS integration |

### GitHub Actions Example

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --coverage

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}/myapp:latest

  deploy-production:
    needs: build
    runs-on: ubuntu-latest
    environment: production    # Requires manual approval
    steps:
      - run: kubectl set image deployment/myapp myapp=ghcr.io/${{ github.repository }}/myapp:latest
```

### Deployment Strategies

| Strategy | Description | Downtime | Rollback |
|---|---|---|---|
| **Recreate** | Stop old, start new | Yes | Slow |
| **Rolling** | Replace instances gradually | No | Moderate |
| **Blue/Green** | Two identical envs, switch traffic | No | Instant |
| **Canary** | Route small % traffic to new version | No | Instant |

**Blue/Green**: Maintain two environments (Blue=idle, Green=active). Deploy v3 to Blue, then switch traffic. Keep Green for instant rollback.

**Canary**: Route 5% of traffic to new version, monitor metrics, then gradually increase to 100%.

---

## Version Control & Git Workflows

### Git Branching Strategies

**GitFlow**: `main` (production), `develop` (integration), `feature/*`, `release/*`, `hotfix/*`. Good for versioned releases.

**Trunk-Based Development** (preferred for CI/CD): All developers push short-lived branches (< 1-2 days) to main. Feature flags hide incomplete features. Enforces CI, reduces integration problems.

**GitHub Flow**: Branch from main → PR → review → merge. Simple and effective.

### Key Git Commands for DevOps

```bash
git tag -a v1.2.3 -m "Release 1.2.3"   # Tag a release
git push origin v1.2.3

git cherry-pick <commit-hash>            # Apply a fix to another branch
git revert <commit-hash>                 # Undo a bad commit safely

git bisect start                         # Find when a bug was introduced
git bisect bad
git bisect good v1.2.0

git rebase -i HEAD~3                     # Squash commits before PR
```

---

## Infrastructure as Code (IaC)

IaC manages infrastructure through **machine-readable configuration files** instead of manual processes. Benefits: reproducibility, version control, automation, and the config IS the documentation.

### IaC Tools Comparison

| Tool | Type | Cloud | Language |
|---|---|---|---|
| **Terraform** | Provisioning | Multi-cloud | HCL |
| **AWS CloudFormation** | Provisioning | AWS only | YAML/JSON |
| **Pulumi** | Provisioning | Multi-cloud | Python/TS/Go |
| **Ansible** | Config Management | Multi-cloud | YAML |

### Terraform Core Concepts

```hcl
# Define provider and remote state
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/main.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"  # Prevents concurrent modifications
    encrypt        = true
  }
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  tags = { Name = "web-server" }
}

output "instance_public_ip" {
  value = aws_instance.web.public_ip
}
```

### Terraform Workflow

```bash
terraform init      # Download providers, initialize backend
terraform validate  # Check syntax
terraform plan      # Preview changes (dry run)
terraform apply     # Apply changes
terraform destroy   # Destroy all resources
```

**State**: The `terraform.tfstate` file maps config to real infrastructure. Store remotely in S3 for team use. Never edit state manually — use `terraform state` commands.

---

## Configuration Management

### Ansible

Ansible automates configuration management and application deployment.

- **Agentless**: Uses SSH, no agent on target hosts
- **Idempotent**: Running a playbook multiple times gives the same result
- **Inventory**: List of managed hosts; **Playbook**: YAML automation tasks; **Role**: Reusable task collection

```yaml
# playbook.yml
- name: Configure web servers
  hosts: webservers
  become: yes
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
ansible -i inventory.yml webservers -m ping       # Test connectivity
ansible-playbook -i inventory.yml playbook.yml    # Run playbook
ansible-playbook playbook.yml --check             # Dry run
ansible-vault encrypt secrets.yml                 # Encrypt secrets
```

---

## Container Orchestration

### Kubernetes Core Concepts

```
Cluster
├── Control Plane
│   ├── API Server          ← Entry point for all K8s commands
│   ├── etcd                ← Cluster state store
│   ├── Scheduler           ← Decides which node runs a pod
│   └── Controller Manager  ← Ensures desired state = actual state
│
└── Worker Nodes
    ├── kubelet             ← Runs pods
    ├── kube-proxy          ← Network rules
    └── Container Runtime   ← containerd/Docker
```

### Kubernetes Objects

```yaml
# Deployment — manages replicas with rolling updates
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
        resources:
          requests: { memory: "64Mi", cpu: "250m" }
          limits:   { memory: "128Mi", cpu: "500m" }
        livenessProbe:
          httpGet: { path: /health, port: 80 }
          initialDelaySeconds: 3
        readinessProbe:
          httpGet: { path: /ready, port: 80 }

---
# Service — stable network endpoint
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
  type: ClusterIP     # ClusterIP | NodePort | LoadBalancer

---
# HPA — auto-scale pods based on CPU
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
kubectl get pods -n production
kubectl describe pod myapp-xyz
kubectl logs myapp-xyz -f
kubectl exec -it myapp-xyz -- bash

kubectl apply -f deployment.yaml
kubectl rollout status deployment/myapp
kubectl rollout undo deployment/myapp        # Rollback
kubectl set image deployment/myapp myapp=myapp:2.0

kubectl scale deployment myapp --replicas=5
kubectl top pods
kubectl port-forward pod/myapp-xyz 8080:80
```

### GitOps with ArgoCD

GitOps: **Git is the single source of truth** for infrastructure and application state.

```
Developer → Push to Git → ArgoCD detects change → Syncs to Kubernetes cluster
```

Principles: declarative desired state in Git; versioned (git history = audit trail); automated sync; drift is corrected automatically.

---

## Monitoring & Observability

### The Three Pillars

| Pillar | Description | Tools |
|---|---|---|
| **Metrics** | Numeric time-series data (CPU, latency, error rate) | Prometheus, Datadog, CloudWatch |
| **Logs** | Timestamped event records | ELK Stack, Loki, Splunk |
| **Traces** | Request flow across services | Jaeger, Zipkin, AWS X-Ray |

### Prometheus + Grafana

```
Application → /metrics endpoint → Prometheus scrapes → Grafana visualizes
                                        ↓
                                  AlertManager → PagerDuty/Slack
```

```yaml
# Alert rule example
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
```

### Key Metrics Methods

**RED Method (services)**: Rate (req/s), Errors (error rate), Duration (latency p50/p95/p99)

**USE Method (resources)**: Utilization (% busy), Saturation (queue depth), Errors (count)

### SLI, SLO, SLA

| Term | Definition | Example |
|---|---|---|
| **SLI** | Metric measuring service behavior | 99th percentile latency = 250ms |
| **SLO** | Target for the SLI | 99% of requests < 500ms |
| **SLA** | Contract with consequences | 99.9% uptime or 10% refund |
| **Error Budget** | 1 - SLO = allowed failure | 0.1% = 43.8 min/month |

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
  "timestamp": "2026-06-18T10:30:00Z",
  "level": "ERROR",
  "service": "payment-api",
  "traceId": "abc123",
  "message": "Payment processing failed",
  "error": "Connection timeout",
  "duration_ms": 5023
}
```

Machines can parse, filter, and query JSON efficiently. Plain text requires fragile regex parsing.

### Log Levels

| Level | Use Case |
|---|---|
| `DEBUG` | Development debugging |
| `INFO` | Normal operational events |
| `WARN` | Unexpected but handled situations |
| `ERROR` | Failures that need attention |
| `FATAL` | System cannot continue |

---

## Site Reliability Engineering (SRE)

SRE applies software engineering principles to operations (Google's approach to DevOps).

### Core Principles

- **Eliminate toil**: Manual, repetitive work that scales with load — automate it
- **Error budgets**: SLO = 99.9% means 0.1% budget to take risks; when depleted, slow down
- **Blameless post-mortems**: Focus on systems, not people
- **Gradual rollouts**: Canary deployments, feature flags

### Incident Management Flow

```
Alert fires → On-call notified
     ↓
Triage: Severity (P1/P2/P3)
     ↓
Mitigation: Restore service ASAP (rollback, scale, redirect)
     ↓
Resolution: Root cause fixed
     ↓
Post-mortem (within 48-72 hours) → Action items to prevent recurrence
```

### Toil vs Engineering Work

| Toil | Engineering Work |
|---|---|
| Manual server restarts | Write auto-restart script |
| SSH into prod to check logs | Set up centralized logging |
| Scale up manually during traffic spikes | Implement HPA in Kubernetes |

---

## Security in DevOps (DevSecOps)

### Shift Left Security

Move security checks earlier in the lifecycle — fixing bugs is cheapest at the developer IDE stage.

```
Developer IDE → CI Build → Staging → Production
    ↑               ↑           ↑           ↑
  secrets         SAST/DAST   pen test   runtime
  detection       dep scan    compliance  monitoring
```

### Security Scanning Types

| Type | Description | Tools |
|---|---|---|
| **SAST** | Scan source code for vulnerabilities | SonarQube, Semgrep |
| **DAST** | Attack running application | OWASP ZAP, Burp Suite |
| **SCA** | Scan dependencies for known CVEs | Snyk, Dependabot |
| **Container Scanning** | Scan images for OS/package CVEs | Trivy, Snyk Container |
| **Secret Detection** | Find leaked credentials in code | GitLeaks, truffleHog |
| **IaC Scanning** | Find misconfigurations in Terraform | Checkov, tfsec |

### Secrets Management

```
# Never do this:
DATABASE_PASSWORD=mysecretpassword  # In code, Dockerfile, or git

# Use instead:
AWS Secrets Manager    → aws secretsmanager get-secret-value
HashiCorp Vault        → vault kv get secret/myapp/db
Kubernetes Secrets     → kubectl create secret generic
# Inject at runtime via environment variables
```

---

## Artifact Management

Build outputs (Docker images, JARs, npm packages, Helm charts) are stored in artifact repositories.

### Artifact Repositories

| Tool | Artifact Types |
|---|---|
| **JFrog Artifactory** | Universal (all types) |
| **Nexus Repository** | Maven, npm, Docker, PyPI |
| **AWS ECR** | Docker images |
| **GitHub Packages** | Docker, npm, Maven, NuGet |

### Semantic Versioning (SemVer): `MAJOR.MINOR.PATCH`
- `1.0.0` → `1.0.1`: bug fix (patch)
- `1.0.0` → `1.1.0`: new feature, backward compatible (minor)
- `1.0.0` → `2.0.0`: breaking change (major)

```bash
myapp:1.2.3         # SemVer (preferred for production)
myapp:git-abc1234   # Git commit SHA (most precise)
myapp:latest        # Risky in prod — avoid
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

### AWS Well-Architected Framework

1. **Operational Excellence** — Run and monitor systems to deliver business value
2. **Security** — Protect data, systems, and assets
3. **Reliability** — Recover from failures and meet demand
4. **Performance Efficiency** — Use resources efficiently
5. **Cost Optimization** — Avoid unnecessary costs
6. **Sustainability** — Minimize environmental impact

---

## Common Interview Questions

### Q1: What is the difference between CI and CD?
CI automatically builds and tests on every commit — catches integration issues early. CD extends CI by preparing releases for deployment but requires manual approval for production. Continuous Deployment goes further: anything passing all tests ships to production automatically with no human gate.

### Q2: What is Infrastructure as Code and why is it important?
IaC manages infrastructure through version-controlled config files instead of manual clicks. Benefits: reproducibility (same code = same infra), auditability via git history, automation, reduced human error, and the ability to test infra changes in lower environments first.

### Q3: What is the difference between blue/green and canary deployments?
Blue/Green keeps two identical environments and switches all traffic at once — zero downtime and instant rollback, but costs double the resources. Canary gradually shifts a small percentage of traffic (e.g., 5%) to the new version, monitors metrics, then rolls forward or back — lower risk and resource-efficient.

### Q4: What are DORA metrics?
Four metrics that measure software delivery performance: **Deployment Frequency** (how often you deploy), **Lead Time for Changes** (commit to production), **Change Failure Rate** (% of deployments causing issues), and **MTTR** (recovery time). Elite teams deploy multiple times per day with under 1 hour lead time.

### Q5: What is a blameless post-mortem?
A post-mortem analyzes an incident to find root causes and prevent recurrence. It's blameless because incidents are systemic failures, not individual ones — blaming people creates a cover-up culture. The focus is on improving processes, systems, and tooling.

### Q6: What is GitOps?
GitOps uses Git as the single source of truth for both application code and infrastructure. Changes go via PRs; an operator like ArgoCD automatically reconciles the cluster to match Git state. Benefits: full audit trail, rollback by reverting a commit, and improved security (cluster pulls changes rather than being pushed to).

### Q7: What is the difference between Ansible and Terraform?
Terraform is a **provisioning** tool — it creates cloud infrastructure (VMs, networks, databases). Ansible is a **configuration management** tool — it configures existing servers (install packages, edit files, start services). They're complementary: Terraform provisions the server, Ansible configures it.

### Q8: What is an error budget?
If your SLO is 99.9% availability, your error budget is 0.1% (~43.8 min/month of allowed downtime). A healthy budget means you can move fast and take risks. When it's depleted, you slow down and prioritize reliability — creating shared language between dev and ops.

### Q9: What is the difference between liveness and readiness probes in Kubernetes?
**Liveness probe**: Is the container still alive? Failure causes Kubernetes to restart it — used to recover from deadlocks. **Readiness probe**: Is the container ready to serve traffic? Failure removes the pod from service endpoints — used during startup or when temporarily overloaded.

### Q10: How do you handle secrets in a CI/CD pipeline?
Never commit secrets to Git. Store them in a secret manager (AWS Secrets Manager, HashiCorp Vault, or GitHub Secrets). Inject at runtime as environment variables. Use short-lived credentials where possible (IAM roles, OIDC). Scan code for accidental commits with tools like GitLeaks.

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
ArgoCD                 Vault (secrets)        MTTR
```

---

*Last updated: 2026-06-18*
