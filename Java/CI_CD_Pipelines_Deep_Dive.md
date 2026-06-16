# CI/CD Pipelines — Deep Dive for Full Stack Java Developer Interviews

## Table of Contents

1. [CI/CD Fundamentals](#1-cicd-fundamentals)
2. [GitHub Actions — Deep Dive](#2-github-actions--deep-dive)
3. [Jenkins Pipeline](#3-jenkins-pipeline)
4. [GitLab CI](#4-gitlab-ci)
5. [Branch Strategies](#5-branch-strategies)
6. [Deployment Strategies](#6-deployment-strategies)
7. [Quality Gates in CI](#7-quality-gates-in-ci)
8. [Secrets Management](#8-secrets-management)
9. [Docker in CI/CD](#9-docker-in-cicd)
10. [Deployment to Kubernetes](#10-deployment-to-kubernetes)
11. [Environment Promotion](#11-environment-promotion)
12. [Monitoring Post-Deployment](#12-monitoring-post-deployment)
13. [Interview Questions & Answers](#13-interview-questions--answers)
14. [Quick Reference Cheat Sheet](#14-quick-reference-cheat-sheet)

---

## 1. CI/CD Fundamentals

### 1.1 Precise Definitions

**Continuous Integration (CI)**
The practice where every developer merges code to the shared main branch multiple times per day. Each merge triggers an automated pipeline that builds and tests the code. The goal is to detect integration problems early — within minutes, not days.

- Key idea: Integration is continuous, not a one-time event at the end of a sprint
- Prerequisite: a fast, reliable automated test suite
- Output: a tested, buildable artifact

**Continuous Delivery (CD — Delivery)**
The practice of ensuring the software is always in a releasable state. Every successful CI run produces an artifact that *could* be deployed to production with a single human approval click. The deployment itself may still be manual or gated.

- Key idea: the release decision is business-driven, not engineering-blocked
- Requires: automated testing, staging environment, automated deployment scripts

**Continuous Deployment (CD — Deployment)**
Extends Delivery by removing the human approval gate. Every commit that passes all automated checks is automatically deployed to production without human intervention.

- Key idea: no human touches the deploy button
- Requires: extremely high test coverage, feature flags, automated rollback, robust monitoring

```
Continuous Integration
    └── every commit → build + test → artifact ready

Continuous Delivery
    └── CI + artifact can be deployed any time (human clicks deploy)

Continuous Deployment
    └── CI + artifact is automatically deployed to production
```

**Analogy:** CI is like a factory quality-checking every part as it comes off the line. CD (Delivery) is like having the finished product on the loading dock ready to ship when the business says go. CD (Deployment) is like the truck leaving automatically the moment the product passes inspection.

---

### 1.2 Benefits

| Benefit | Without CI/CD | With CI/CD |
|---------|--------------|-----------|
| Integration risk | "Integration hell" at sprint end | Caught in minutes per commit |
| Feedback speed | Days to weeks | Minutes |
| Deployment frequency | Monthly or quarterly | Multiple times per day |
| Deployment reliability | Manual, error-prone | Automated, repeatable |
| Mean Time to Recovery | Hours or days | Minutes |
| Developer confidence | Low (fear of breaking things) | High (tests catch regressions) |

**Reduced integration risk:** When developers integrate small, frequent changes instead of large batches, merge conflicts and logic conflicts are smaller and easier to resolve.

**Faster feedback:** A developer finds out within 5 minutes if their commit broke tests, rather than discovering it 3 weeks later during a release.

**Repeatable deployments:** The same pipeline script deploys to dev, staging, and production. No snowflake manual steps that differ between environments.

---

### 1.3 Pipeline Stages

```
Source → Build → Test → Scan → Package → Deploy → Verify
```

| Stage | What Happens | Fail Behaviour |
|-------|-------------|----------------|
| **Source** | Checkout code, validate commit message | Block pipeline |
| **Build** | Compile Java, resolve dependencies | Block — nothing to test |
| **Test** | Unit tests, integration tests, coverage check | Block — do not ship broken code |
| **Scan** | SAST, dependency vulnerability scan, code quality | Block or warn (configurable) |
| **Package** | Build Docker image, push to registry | Block — no artifact to deploy |
| **Deploy** | Apply manifests to target environment | Block next stage |
| **Verify** | Smoke tests, health checks, synthetic monitoring | Trigger rollback |

Each stage must pass before the next runs. This is the "fast feedback" principle — expensive stages run only if cheap stages pass.

---

### 1.4 Artifacts

An **artifact** is an immutable, versioned output of the build stage that is stored outside the pipeline and can be deployed to any environment.

**Types of artifacts in a Java/Spring Boot project:**
- JAR file (fat/uber JAR from `mvn package`)
- WAR file (for traditional app servers)
- Docker image (the most common modern artifact)
- Helm chart package (`.tgz`)

**Why artifacts must be immutable:** You build once, then promote the same artifact through dev → staging → production. If you rebuild for each environment, you cannot guarantee what was tested is what was deployed.

**Artifact Repositories:**

| Tool | Type | Notes |
|------|------|-------|
| **Nexus Repository** | Maven, Docker, npm, raw | Self-hosted, popular in enterprises |
| **JFrog Artifactory** | Maven, Docker, npm, Helm, raw | Enterprise, has Xray for security scanning |
| **AWS ECR** | Docker images only | Managed, integrates with ECS/EKS |
| **GitHub Packages** | Maven, Docker, npm | Built into GitHub, good for OSS |
| **Docker Hub** | Docker images | Public default registry |

**Tagging strategy for Docker images:**
```
myapp:latest          ← ANTI-PATTERN (mutable, unpredictable)
myapp:1.2.3           ← Semantic version (good for releases)
myapp:abc1234         ← Git commit SHA (best for traceability)
myapp:1.2.3-abc1234   ← Version + SHA (best of both worlds)
```

---

### 1.5 Environment Promotion

```
dev ──→ staging ──→ production
 ↑          ↑            ↑
auto      auto        manual gate
deploy    deploy    or auto (CD)
```

**Dev:** Auto-deploys every merged commit. Fast feedback for developers. May run only unit tests.

**Staging:** Mirror of production. Deploys release candidates. Runs full integration tests, performance tests, security scans.

**Production:** The live environment. Deploys only promoted, verified artifacts. May require manual approval or be fully automated for CD.

**The promotion pattern:** The Docker image SHA built in CI is the same image deployed to staging and production. Only configuration (env vars, secrets) differs per environment. This is "build once, deploy everywhere."

---

## 2. GitHub Actions — Deep Dive

### 2.1 Core Concepts

| Concept | Description |
|---------|-------------|
| **Workflow** | A YAML file in `.github/workflows/`. An automated process triggered by events. |
| **Event (trigger)** | What starts the workflow: push, PR, schedule, manual, etc. |
| **Job** | A set of steps that run on the same runner (VM). Jobs run in parallel by default. |
| **Step** | A single task within a job. Either a shell command (`run`) or an action (`uses`). |
| **Action** | A reusable unit of code. Can be from the marketplace, your repo, or a Docker image. |
| **Runner** | The VM or container that executes the job. GitHub-hosted or self-hosted. |
| **Context** | Objects containing runtime info: `github`, `env`, `secrets`, `matrix`, `steps`, etc. |

---

### 2.2 Workflow YAML Structure

```yaml
name: CI Pipeline                    # Display name in GitHub UI

on:                                  # TRIGGERS — what starts this workflow
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'             # Every Monday at 2am UTC
  workflow_dispatch:                 # Manual trigger from GitHub UI
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]

env:                                 # Workflow-level environment variables
  JAVA_VERSION: '21'
  IMAGE_NAME: myapp

jobs:
  build:                             # Job ID (referenced by other jobs)
    name: Build and Test             # Display name
    runs-on: ubuntu-latest           # Runner type
    
    env:                             # Job-level env vars (override workflow-level)
      MAVEN_OPTS: -Xmx2g
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4    # Action from marketplace
        with:                        # Inputs to the action
          fetch-depth: 0             # Full history for SonarQube
      
      - name: Run tests
        run: mvn test                # Shell command
        env:                         # Step-level env vars
          DB_URL: ${{ secrets.TEST_DB_URL }}

  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    needs: [build]                   # Wait for 'build' job to succeed
    if: github.ref == 'refs/heads/main'  # Conditional execution
    
    steps:
      - name: Deploy
        run: echo "Deploying..."
```

---

### 2.3 Triggers in Depth

```yaml
on:
  # Push to specific branches or tags
  push:
    branches:
      - main
      - 'release/**'        # Glob pattern
    branches-ignore:
      - 'dependabot/**'
    tags:
      - 'v*'                # Any tag starting with v
    paths:
      - 'src/**'            # Only trigger if these paths changed
      - 'pom.xml'
    paths-ignore:
      - '**.md'             # Ignore markdown changes

  # Pull request events
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]  # PR event types

  # Scheduled (cron syntax)
  schedule:
    - cron: '0 0 * * *'    # Daily at midnight UTC

  # Manual trigger with inputs
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy'
        required: true
        type: string

  # Called by another workflow
  workflow_call:
    inputs:
      image-tag:
        required: true
        type: string
    secrets:
      ECR_ROLE_ARN:
        required: true
```

---

### 2.4 Jobs: Dependencies, Matrix, Conditionals

```yaml
jobs:
  # Matrix strategy: run job for multiple configurations
  test:
    strategy:
      matrix:
        java: [17, 21]
        os: [ubuntu-latest, windows-latest]
      fail-fast: false        # Don't cancel all matrix jobs if one fails
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java }}
          distribution: temurin
      - run: mvn test

  # Job that depends on another
  build-docker:
    needs: test               # Only runs if 'test' succeeds
    runs-on: ubuntu-latest
    steps:
      - run: docker build .

  # Fan-out then fan-in pattern
  deploy-staging:
    needs: build-docker
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy to staging"

  integration-test:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - run: echo "Run integration tests"

  deploy-production:
    needs: [build-docker, integration-test]  # Both must pass
    runs-on: ubuntu-latest
    environment: production                  # Requires reviewer approval
    steps:
      - run: echo "Deploy to production"
```

---

### 2.5 Contexts

```yaml
steps:
  - name: Show context values
    run: |
      echo "Repo: ${{ github.repository }}"
      echo "Branch: ${{ github.ref_name }}"
      echo "Commit SHA: ${{ github.sha }}"
      echo "Event: ${{ github.event_name }}"
      echo "Actor: ${{ github.actor }}"
      echo "Run ID: ${{ github.run_id }}"
      echo "Workflow: ${{ github.workflow }}"

  - name: Use secret
    run: aws configure set aws_access_key_id ${{ secrets.AWS_ACCESS_KEY_ID }}

  - name: Reference matrix value
    run: echo "Java version ${{ matrix.java }}"

  - name: Reference step output
    id: get-version
    run: echo "version=1.2.3" >> $GITHUB_OUTPUT

  - name: Use previous step output
    run: echo "Version is ${{ steps.get-version.outputs.version }}"

  - name: Use env context
    env:
      MY_VAR: "hello"
    run: echo "${{ env.MY_VAR }}"
```

---

### 2.6 Complete Spring Boot CI/CD Pipeline (GitHub Actions)

This is the most important example. Study this end-to-end.

```yaml
# .github/workflows/ci-cd.yml
name: Spring Boot CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: my-springboot-app
  K8S_NAMESPACE: production
  JAVA_VERSION: '21'

jobs:
  # ─────────────────────────────────────────────
  # JOB 1: Build, Test, and Code Quality
  # ─────────────────────────────────────────────
  build-and-test:
    name: Build, Test & Scan
    runs-on: ubuntu-latest

    steps:
      # Step 1: Checkout source code
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0    # Full history required for SonarQube blame info

      # Step 2: Set up JDK
      - name: Set up JDK ${{ env.JAVA_VERSION }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin    # Eclipse Temurin (formerly AdoptOpenJDK)
          cache: maven             # Built-in Maven cache support

      # Step 3: Cache Maven dependencies (faster than re-downloading every run)
      - name: Cache Maven local repository
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      # Step 4: Build and run tests
      - name: Build and test with Maven
        run: mvn clean verify --batch-mode --no-transfer-progress
        # --batch-mode: non-interactive, better log output
        # --no-transfer-progress: suppress download progress bars

      # Step 5: Publish test results
      - name: Publish test results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()    # Run even if previous step failed
        with:
          files: target/surefire-reports/*.xml

      # Step 6: Upload JaCoCo coverage report as artifact
      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: jacoco-report
          path: target/site/jacoco/
          retention-days: 7

      # Step 7: SonarQube analysis
      - name: SonarQube Scan
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          mvn sonar:sonar \
            -Dsonar.projectKey=my-springboot-app \
            -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }} \
            -Dsonar.login=${{ secrets.SONAR_TOKEN }}

      # Step 8: OWASP Dependency Check (SAST)
      - name: OWASP Dependency Check
        run: |
          mvn org.owasp:dependency-check-maven:check \
            -DfailBuildOnCVSS=7 \
            -DskipTestScope=true

      # Step 9: Upload the JAR artifact for downstream jobs
      - name: Upload JAR artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar
          retention-days: 1

  # ─────────────────────────────────────────────
  # JOB 2: Build and Push Docker Image
  # Only runs on push to main (not on PRs)
  # ─────────────────────────────────────────────
  build-docker:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: build-and-test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build-push.outputs.digest }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # Download the JAR built in job 1 (don't rebuild)
      - name: Download JAR artifact
        uses: actions/download-artifact@v4
        with:
          name: app-jar
          path: target/

      # Configure AWS credentials via OIDC (no long-lived access keys)
      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      # Login to Amazon ECR
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      # Generate Docker image metadata (tags, labels)
      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}
          tags: |
            type=sha,prefix=,format=short          # git short SHA: abc1234
            type=semver,pattern={{version}}         # if tagged: 1.2.3
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      # Set up Docker Buildx (for multi-platform builds and cache)
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # Build and push Docker image
      - name: Build and push Docker image
        id: build-push
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha              # Use GitHub Actions cache for Docker layers
          cache-to: type=gha,mode=max

      # Scan image for vulnerabilities with Trivy
      - name: Trivy vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1    # Fail pipeline on CRITICAL/HIGH CVEs

      - name: Upload Trivy results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif

  # ─────────────────────────────────────────────
  # JOB 3: Deploy to Kubernetes
  # ─────────────────────────────────────────────
  deploy:
    name: Deploy to Kubernetes
    runs-on: ubuntu-latest
    needs: build-docker
    environment: production    # Requires reviewer approval in GitHub UI

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      # Update kubeconfig to point to your EKS cluster
      - name: Update kubeconfig for EKS
        run: |
          aws eks update-kubeconfig \
            --name my-eks-cluster \
            --region ${{ env.AWS_REGION }}

      # Update the K8s deployment with the new image
      - name: Deploy to Kubernetes
        run: |
          IMAGE_TAG="${{ github.sha }}"
          ECR_REGISTRY="${{ steps.login-ecr.outputs.registry }}"

          # Update image in deployment
          kubectl set image deployment/springboot-app \
            springboot-app=${ECR_REGISTRY}/${{ env.ECR_REPOSITORY }}:${IMAGE_TAG} \
            --namespace ${{ env.K8S_NAMESPACE }}

          # Wait for rollout to complete
          kubectl rollout status deployment/springboot-app \
            --namespace ${{ env.K8S_NAMESPACE }} \
            --timeout=300s

      # Post-deployment smoke test
      - name: Smoke test
        run: |
          APP_URL="${{ secrets.APP_URL }}"
          HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" ${APP_URL}/actuator/health)
          if [ "$HTTP_STATUS" != "200" ]; then
            echo "Smoke test failed! Status: $HTTP_STATUS"
            kubectl rollout undo deployment/springboot-app \
              --namespace ${{ env.K8S_NAMESPACE }}
            exit 1
          fi
          echo "Smoke test passed. Status: $HTTP_STATUS"
```

**Dockerfile (multi-stage, referenced in the pipeline above):**

```dockerfile
# Stage 1: Build (heavy JDK image)
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY target/*.jar app.jar
# Extract layers for better caching
RUN java -Djarmode=layertools -jar app.jar extract

# Stage 2: Runtime (lean JRE image)
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

# Create non-root user for security
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

# Copy layered content from builder (optimizes layer caching)
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

---

### 2.7 Caching Maven Dependencies

```yaml
# Option 1: Built-in Maven cache in setup-java
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: temurin
    cache: maven    # Automatically caches ~/.m2

# Option 2: Manual cache with actions/cache (more control)
- name: Cache Maven repository
  uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    # Cache key: OS + hash of all pom.xml files
    # Cache is invalidated only when pom.xml changes
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
    restore-keys: |
      ${{ runner.os }}-maven-    # Fallback: any Maven cache for this OS
```

**Why cache matters:** Maven downloads ~200MB of dependencies on a fresh runner. Caching reduces this to near-zero after the first run. Pipeline time drops from ~4 minutes to ~1 minute.

---

### 2.8 OIDC for AWS Authentication (No Long-Lived Keys)

Traditional approach (bad — long-lived access keys in secrets):
```yaml
# BAD: Static access keys can be leaked, don't rotate automatically
- uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

OIDC approach (best practice — short-lived tokens):
```yaml
# GOOD: GitHub gets a short-lived token from AWS STS
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions-role
    aws-region: us-east-1
    # GitHub exchanges its OIDC token for AWS credentials
    # Credentials are valid for ~1 hour only
```

**AWS IAM role trust policy (configured once in AWS):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:*"
      }
    }
  }]
}
```

---

### 2.9 Reusable Workflows and Composite Actions

**Reusable workflow** (called by other workflows):
```yaml
# .github/workflows/deploy-reusable.yml
name: Reusable Deploy Workflow

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string
    secrets:
      AWS_ROLE_ARN:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy ${{ inputs.image-tag }} to ${{ inputs.environment }}
        run: echo "Deploying..."
```

**Calling a reusable workflow:**
```yaml
# .github/workflows/ci.yml
jobs:
  deploy-staging:
    uses: ./.github/workflows/deploy-reusable.yml
    with:
      environment: staging
      image-tag: ${{ github.sha }}
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

**Composite action** (reusable multi-step unit):
```yaml
# .github/actions/setup-and-build/action.yml
name: Setup and Build
description: Sets up Java and builds the project

inputs:
  java-version:
    description: Java version
    default: '21'

runs:
  using: composite
  steps:
    - uses: actions/setup-java@v4
      with:
        java-version: ${{ inputs.java-version }}
        distribution: temurin
        cache: maven
    - run: mvn clean package -DskipTests
      shell: bash
```

---

### 2.10 Environments with Required Reviewers

In GitHub repository settings → Environments → production:
- Add required reviewers (e.g., tech lead must approve)
- Set wait timer (e.g., 5 minutes for smoke tests in staging)
- Add environment-specific secrets (production DB credentials)

```yaml
jobs:
  deploy-prod:
    environment:
      name: production
      url: https://myapp.com    # Link shown in GitHub UI
    runs-on: ubuntu-latest
    steps:
      - run: echo "This only runs after reviewer approves in GitHub UI"
```

---

### 2.11 Self-Hosted vs GitHub-Hosted Runners

| | GitHub-Hosted | Self-Hosted |
|---|---|---|
| **Cost** | Included (2000 min/month free) | Your infrastructure cost |
| **Maintenance** | Zero | You patch, update, secure |
| **Network access** | Public internet only | Can access private VPCs |
| **Performance** | Standard (2-core, 7GB RAM) | Your choice (can be beefy) |
| **Ephemeral** | Yes (fresh VM each run) | Configurable |
| **Use case** | Most CI workloads | Private resources, GPU, speed |

```yaml
# Self-hosted runner
runs-on: self-hosted

# Self-hosted with labels
runs-on: [self-hosted, linux, x64, production]
```

---

## 3. Jenkins Pipeline

### 3.1 Declarative vs Scripted Pipeline

| | Declarative | Scripted |
|---|---|---|
| **Syntax** | Structured, predefined blocks | Groovy DSL, full flexibility |
| **Error handling** | Built-in `post` blocks | `try/catch/finally` |
| **Learning curve** | Lower | Higher (Groovy knowledge needed) |
| **Validation** | Pre-flight validation | Runtime errors only |
| **When to use** | Standard pipelines (95% of cases) | Complex conditional logic |

---

### 3.2 Complete Declarative Jenkinsfile for Spring Boot

```groovy
// Jenkinsfile (Declarative)
pipeline {
    agent any    // Run on any available agent
    // agent { label 'java-agent' }  // Or a specific labeled agent

    // Tool versions (configured in Jenkins Global Tool Configuration)
    tools {
        jdk 'JDK21'
        maven 'Maven3'
    }

    // Pipeline-level environment variables
    environment {
        APP_NAME = 'springboot-app'
        DOCKER_REGISTRY = 'your-ecr-url.amazonaws.com'
        IMAGE_TAG = "${BUILD_NUMBER}-${GIT_COMMIT.take(7)}"
        SONAR_PROJECT_KEY = 'springboot-app'
    }

    // Pipeline options
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))  // Keep last 10 builds
        timeout(time: 30, unit: 'MINUTES')              // Fail if > 30 min
        disableConcurrentBuilds()                        // No parallel builds for same branch
        timestamps()                                     // Add timestamps to log
    }

    // Parameterized build (shows form in Jenkins UI)
    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['dev', 'staging', 'production'], description: 'Deployment target')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip test execution')
        string(name: 'IMAGE_VERSION', defaultValue: 'latest', description: 'Override image version')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm    // Checks out the repository that triggered the build
                sh 'git log -1 --oneline'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile --batch-mode -q'
            }
        }

        stage('Test') {
            when {
                not { params.SKIP_TESTS }    // Skip if parameter set
            }
            steps {
                sh 'mvn test --batch-mode'
            }
            post {
                always {
                    // Publish JUnit test results in Jenkins UI
                    junit 'target/surefire-reports/*.xml'
                    // Publish JaCoCo coverage
                    jacoco(
                        execPattern: 'target/jacoco.exec',
                        classPattern: 'target/classes',
                        sourcePattern: 'src/main/java',
                        minimumLineCoverage: '80'    // Fail if < 80%
                    )
                }
            }
        }

        stage('Code Quality') {
            parallel {
                stage('SonarQube') {
                    steps {
                        withSonarQubeEnv('SonarQube') {    // Uses SonarQube config from Jenkins
                            sh """
                                mvn sonar:sonar \
                                    -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                                    -Dsonar.branch.name=${BRANCH_NAME}
                            """
                        }
                        // Wait for quality gate result
                        timeout(time: 5, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: true
                        }
                    }
                }
                stage('OWASP Dependency Check') {
                    steps {
                        sh 'mvn org.owasp:dependency-check-maven:check -DfailBuildOnCVSS=7'
                        dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
                    }
                }
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests --batch-mode'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Docker Build & Push') {
            when {
                anyOf {
                    branch 'main'
                    branch 'release/*'
                }
            }
            steps {
                script {
                    // withCredentials injects credentials into env vars securely
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                      credentialsId: 'aws-ecr-credentials']]) {
                        sh """
                            # Authenticate Docker to ECR
                            aws ecr get-login-password --region us-east-1 | \
                                docker login --username AWS --password-stdin ${DOCKER_REGISTRY}

                            # Build image
                            docker build -t ${APP_NAME}:${IMAGE_TAG} .

                            # Tag and push
                            docker tag ${APP_NAME}:${IMAGE_TAG} ${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG}
                            docker push ${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG}
                        """
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            when { branch 'main' }
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-dev']) {
                    sh """
                        kubectl set image deployment/${APP_NAME} \
                            ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG} \
                            --namespace dev
                        kubectl rollout status deployment/${APP_NAME} --namespace dev --timeout=120s
                    """
                }
            }
        }

        stage('Deploy to Staging') {
            when { branch 'main' }
            steps {
                input message: 'Deploy to staging?', ok: 'Deploy'    // Manual approval gate
                withKubeConfig([credentialsId: 'kubeconfig-staging']) {
                    sh "kubectl set image deployment/${APP_NAME} ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG} --namespace staging"
                }
            }
        }
    }

    // Post-build actions (run after all stages)
    post {
        always {
            cleanWs()    // Clean workspace regardless of result
        }
        success {
            slackSend(
                color: 'good',
                message: "SUCCESS: ${APP_NAME} #${BUILD_NUMBER} deployed. (<${BUILD_URL}|Open>)"
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "FAILED: ${APP_NAME} #${BUILD_NUMBER} failed. (<${BUILD_URL}|Open>)"
            )
            emailext(
                subject: "Pipeline Failed: ${APP_NAME} #${BUILD_NUMBER}",
                body: "Check Jenkins: ${BUILD_URL}",
                to: "team@company.com"
            )
        }
        unstable {
            // Tests passed but with warnings
            slackSend(color: 'warning', message: "UNSTABLE: ${APP_NAME} #${BUILD_NUMBER}")
        }
    }
}
```

---

### 3.3 Jenkins Shared Libraries

Shared libraries allow reusing pipeline code across multiple Jenkinsfiles.

**Repository structure:**
```
jenkins-shared-library/
├── vars/
│   ├── buildSpringBoot.groovy    # Global variable (callable as buildSpringBoot())
│   └── deployToK8s.groovy
├── src/
│   └── com/company/
│       └── BuildUtils.groovy     # Class-style utilities
└── resources/
    └── deploy-template.sh
```

**vars/buildSpringBoot.groovy:**
```groovy
def call(Map config = [:]) {
    def javaVersion = config.javaVersion ?: '21'
    def skipTests = config.skipTests ?: false

    pipeline {
        agent any
        tools { jdk "JDK${javaVersion}" }
        stages {
            stage('Build') {
                steps {
                    sh "mvn clean ${skipTests ? 'package -DskipTests' : 'verify'}"
                }
            }
        }
    }
}
```

**Using in Jenkinsfile:**
```groovy
@Library('jenkins-shared-library@main') _

buildSpringBoot(
    javaVersion: '21',
    skipTests: false
)
```

---

### 3.4 Multibranch Pipeline

Jenkins Multibranch Pipeline automatically discovers branches and creates a pipeline job for each. It reads the Jenkinsfile from each branch.

```groovy
// In Jenkinsfile — branch-specific logic
pipeline {
    agent any
    stages {
        stage('Deploy') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        deployToProduction()
                    } else if (env.BRANCH_NAME == 'develop') {
                        deployToDev()
                    }
                }
            }
        }
    }
}
```

---

## 4. GitLab CI

### 4.1 .gitlab-ci.yml Structure

```yaml
# .gitlab-ci.yml

# Define pipeline stages in order
stages:
  - build
  - test
  - scan
  - package
  - deploy

# Global variables (available in all jobs)
variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2"
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

# Global cache (inherited by all jobs unless overridden)
cache:
  paths:
    - .m2/

# ─────────────────────────────────────────────
# BUILD STAGE
# ─────────────────────────────────────────────
build:
  stage: build
  image: maven:3.9-eclipse-temurin-21
  script:
    - mvn clean compile --batch-mode -q
  artifacts:
    paths:
      - target/classes/
    expire_in: 1 hour

# ─────────────────────────────────────────────
# TEST STAGE (parallel jobs)
# ─────────────────────────────────────────────
unit-tests:
  stage: test
  image: maven:3.9-eclipse-temurin-21
  script:
    - mvn test --batch-mode
  artifacts:
    when: always    # Upload even if tests fail (to see results)
    reports:
      junit: target/surefire-reports/TEST-*.xml
      coverage_report:
        coverage_format: jacoco
        path: target/site/jacoco/jacoco.xml
  coverage: '/Total.*?([0-9]{1,3})%/'    # Regex to extract coverage %

integration-tests:
  stage: test
  image: maven:3.9-eclipse-temurin-21
  services:
    - postgres:15    # Spin up a Postgres container for integration tests
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    SPRING_DATASOURCE_URL: jdbc:postgresql://postgres/testdb
  script:
    - mvn verify -Pintegration-tests --batch-mode

# ─────────────────────────────────────────────
# SCAN STAGE
# ─────────────────────────────────────────────
sonarqube:
  stage: scan
  image: maven:3.9-eclipse-temurin-21
  script:
    - mvn sonar:sonar
        -Dsonar.projectKey=$CI_PROJECT_NAME
        -Dsonar.host.url=$SONAR_URL
        -Dsonar.login=$SONAR_TOKEN
  allow_failure: false

# ─────────────────────────────────────────────
# PACKAGE STAGE
# ─────────────────────────────────────────────
docker-build:
  stage: package
  image: docker:24
  services:
    - docker:24-dind    # Docker-in-Docker
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE
    # Also tag as 'latest' for main branch
    - |
      if [ "$CI_COMMIT_BRANCH" == "main" ]; then
        docker tag $DOCKER_IMAGE $CI_REGISTRY_IMAGE:latest
        docker push $CI_REGISTRY_IMAGE:latest
      fi
  only:
    - main
    - tags

# ─────────────────────────────────────────────
# DEPLOY STAGE
# ─────────────────────────────────────────────
deploy-staging:
  stage: deploy
  image: bitnami/kubectl:latest
  environment:
    name: staging
    url: https://staging.myapp.com
  before_script:
    - kubectl config set-cluster k8s --server=$K8S_SERVER
    - kubectl config set-credentials admin --token=$K8S_TOKEN
    - kubectl config set-context default --cluster=k8s --user=admin
    - kubectl config use-context default
  script:
    - kubectl set image deployment/springboot-app
        springboot-app=$DOCKER_IMAGE
        --namespace staging
    - kubectl rollout status deployment/springboot-app --namespace staging
  only:
    - main

deploy-production:
  stage: deploy
  image: bitnami/kubectl:latest
  environment:
    name: production
    url: https://myapp.com
  script:
    - kubectl set image deployment/springboot-app springboot-app=$DOCKER_IMAGE --namespace production
  when: manual           # Manual trigger — must click "play" in GitLab UI
  only:
    - main
```

### 4.2 Include and Extend for DRY Configuration

```yaml
# .gitlab-ci.yml — main file
include:
  - local: '.gitlab/ci/build.yml'
  - local: '.gitlab/ci/test.yml'
  - template: Security/SAST.gitlab-ci.yml    # GitLab built-in template

# Using 'extends' to inherit from a base job
.deploy-base:          # Jobs starting with '.' are hidden (not run directly)
  image: bitnami/kubectl:latest
  before_script:
    - kubectl config use-context $K8S_CONTEXT
  script:
    - kubectl set image deployment/app app=$DOCKER_IMAGE --namespace $NAMESPACE

deploy-staging:
  extends: .deploy-base
  variables:
    NAMESPACE: staging
    K8S_CONTEXT: staging-context
  environment: staging

deploy-production:
  extends: .deploy-base
  variables:
    NAMESPACE: production
    K8S_CONTEXT: production-context
  environment: production
  when: manual
```

---

## 5. Branch Strategies

### 5.1 GitFlow

```
main (production-ready)
│
├── hotfix/fix-login-bug ──────────────────────┐
│                                              ↓ merge to main + develop
develop (integration branch)
│
├── feature/user-auth ──────────────────────→ develop
├── feature/payment-api ────────────────────→ develop
│
└── release/1.2.0 ──────────────────────────→ main (tag v1.2.0) + develop
```

**Branch types:**
- `main`: Production code. Only receives merges from `release/*` and `hotfix/*`
- `develop`: Integration branch. Features merge here. Nightly builds
- `feature/*`: One branch per feature/story. Branched from `develop`
- `release/*`: Branched from `develop` when ready. Only bug fixes allowed. Merges to both `main` and `develop`
- `hotfix/*`: Emergency production fix. Branched from `main`. Merges to both `main` and `develop`

**When to use GitFlow:**
- Projects with versioned releases (software products, libraries with semver)
- Multiple versions in production simultaneously (v1.x and v2.x)
- Longer QA cycles (weeks of testing before release)

**Drawbacks:**
- Complex — many branch types, double-merge overhead
- Slow feedback — features sit in `develop` until a release branch is cut
- Hard to do CI/CD — `main` may be weeks behind the latest work

---

### 5.2 Trunk-Based Development (TBD)

```
main (trunk) ────────────────────────────────→
  ↑    ↑    ↑    ↑    ↑    ↑
  |    |    |    |    |    |
 short-lived feature branches (< 1-2 days)
 directly commit for senior devs
```

**Core principles:**
1. Everyone integrates to `main` (the trunk) at least once per day
2. Feature branches are short-lived — merged within hours, not days
3. Never-ending branches don't exist
4. Unfinished features are hidden behind **feature flags**, not behind branches

**Feature flags (decouple deploy from release):**
```java
// Spring Boot with feature flags (Unleash or custom)
@RestController
public class PaymentController {

    @Autowired
    private FeatureFlagService featureFlags;

    @PostMapping("/payment")
    public ResponseEntity<?> processPayment(@RequestBody PaymentRequest req) {
        if (featureFlags.isEnabled("new-payment-gateway", req.getUserId())) {
            return newPaymentGatewayService.process(req);    // New code, dark
        } else {
            return legacyPaymentService.process(req);        // Old code, live
        }
    }
}
```

**Why TBD is preferred for CI/CD:**
- Every commit to main triggers a full CI pipeline (true continuous integration)
- No "merge day" — conflicts are tiny because integrations happen daily
- Enables continuous deployment — main is always deployable
- Feature flags allow zero-risk releases: flip the flag, no deployment needed to enable/disable

**When to use TBD:**
- Small to medium teams (2–50 developers)
- Products with a single production version
- Teams practicing CI/CD and feature flagging

---

### 5.3 GitHub Flow

```
main ──────────────────────────────────────→ (always deployable)
  ↑      ↑
  |      |
feature/user-auth    feature/search
(open PR → review → merge → auto-deploy)
```

**Steps:**
1. Branch from `main` for your feature
2. Commit and push to the feature branch
3. Open a Pull Request — this triggers the CI pipeline
4. Code review, address feedback
5. Merge to `main`
6. `main` is automatically deployed to production

**Difference from TBD:** Feature branches in GitHub Flow can be longer-lived than TBD (days to weeks). No feature flag requirement.

**When to use GitHub Flow:**
- Web applications (only one version in production)
- Small teams, fast iterations
- Open source projects

---

### 5.4 Comparison Table

| | GitFlow | Trunk-Based Dev | GitHub Flow |
|---|---------|-----------------|-------------|
| **Branch complexity** | High (5 branch types) | Low (trunk + short features) | Low (main + features) |
| **Integration frequency** | Weekly/per feature | Daily or per-commit | Per PR |
| **Suitable for CI/CD** | Poor | Excellent | Good |
| **Release cadence** | Versioned, scheduled | Continuous | Continuous |
| **Feature flags needed** | No | Yes | Sometimes |
| **Multiple prod versions** | Yes | No | No |
| **Team size** | Any | Small–Medium | Small |
| **Learning curve** | High | Medium | Low |

---

## 6. Deployment Strategies

### 6.1 Recreate

```
v1 v1 v1  →  [DOWN]  →  v2 v2 v2
```

**How it works:** Stop all v1 instances, then start all v2 instances.

**Pros:** Simple, no version coexistence issues, free (no extra infra)
**Cons:** Downtime during deployment, no gradual rollout
**Use when:** Dev/test environments, batch jobs, services where downtime is acceptable

---

### 6.2 Rolling Deployment

```
v1 v1 v1 v1
v2 v1 v1 v1  (replace 1)
v2 v2 v1 v1  (replace 2)
v2 v2 v2 v1  (replace 3)
v2 v2 v2 v2  (complete)
```

**How it works:** Replace instances one at a time (or in batches). Load balancer routes traffic to healthy instances.

**Kubernetes rolling deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1      # At most 1 pod down at a time
      maxSurge: 1            # At most 1 extra pod during update
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
          image: myrepo/springboot-app:v2
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
```

**Pros:** Zero downtime, incremental, no extra infrastructure cost
**Cons:** Both versions run simultaneously (backward-compatible APIs required), rollback is slower
**Use when:** Standard deployments in production, K8s default strategy

---

### 6.3 Blue-Green Deployment

```
                    ┌─── Load Balancer ────┐
                    │                      │
                  [LIVE]                [IDLE]
               Blue (v1)            Green (v2)
                                    (being tested)

After switch:
                    ┌─── Load Balancer ────┐
                    │                      │
                  [IDLE]                [LIVE]
               Blue (v1)            Green (v2)
```

**How it works:**
1. Maintain two identical production environments (blue = current, green = new)
2. Deploy new version to idle environment (green)
3. Run smoke tests against green
4. Switch load balancer from blue to green (near-instant DNS/LB flip)
5. Keep blue running for quick rollback
6. After confidence, decommission blue (or it becomes the next idle env)

**AWS implementation:**
```yaml
# ALB weighted routing: switch 100% traffic to green
aws elbv2 modify-rule \
  --rule-arn $RULE_ARN \
  --actions '[{
    "Type": "forward",
    "ForwardConfig": {
      "TargetGroups": [
        {"TargetGroupArn": "$BLUE_TG_ARN", "Weight": 0},
        {"TargetGroupArn": "$GREEN_TG_ARN", "Weight": 100}
      ]
    }
  }]'
```

**Pros:** Instant rollback (flip LB back), no mixed versions, test in production-like env before cutover
**Cons:** Double the infrastructure cost, stateful apps (sessions, DB) are complex
**Use when:** High-stakes releases, regulated environments, when you need instant rollback

---

### 6.4 Canary Deployment

```
         ┌─────────────────────────────────┐
Traffic  │ 95% → v1  (existing, stable)   │
         │  5% → v2  (new version, canary) │
         └─────────────────────────────────┘

Monitor error rates, latency, user metrics for v2...
If healthy: 20% → 50% → 100% (gradual rollout)
If bad:     0% → rollback canary
```

**Kubernetes canary with two deployments:**
```yaml
# v1: main deployment (19 replicas = 95%)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-app-stable
spec:
  replicas: 19
  selector:
    matchLabels:
      app: springboot-app
      version: stable

---
# v2: canary deployment (1 replica = 5%)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: springboot-app
      version: canary

---
# Service selects both (no version label)
apiVersion: v1
kind: Service
metadata:
  name: springboot-app
spec:
  selector:
    app: springboot-app    # Matches both stable and canary pods
```

**Argo Rollouts canary (more sophisticated):**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: springboot-app
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5      # 5% traffic to new version
        - pause: {duration: 5m}
        - setWeight: 20
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100
      analysis:             # Automatic rollback on metric degradation
        templates:
          - templateName: success-rate
        startingStep: 1
```

**Pros:** Low blast radius, real-traffic testing, gradual confidence building, automatic rollback
**Cons:** Requires sophisticated monitoring, API backward compatibility needed, complex routing
**Use when:** High-traffic services, risky changes, microservices with good observability

---

### 6.5 Feature Flags

Feature flags let you deploy code to production without releasing it to users. The feature is wrapped in a conditional; the flag can be toggled without a deployment.

```java
// Using Unleash SDK
@Service
public class SearchService {

    @Autowired
    private Unleash unleash;

    public List<Product> search(String query, String userId) {
        if (unleash.isEnabled("elastic-search", new UnleashContext.Builder()
                .userId(userId).build())) {
            return elasticSearchAdapter.search(query);    // New implementation
        }
        return legacyDatabaseSearch(query);              // Old implementation
    }
}
```

**Tools:** LaunchDarkly, Unleash (open source), Split.io, Spring Cloud Config (basic on/off)

**Use cases:**
- Dark launches (deploy but keep off)
- A/B testing
- Kill switches (disable a feature instantly without deployment)
- Gradual rollouts by user segment (10% of users → 50% → 100%)

---

### 6.6 Deployment Strategy Comparison Table

| Strategy | Downtime | Rollback Speed | Cost | Version Coexistence | Use Case |
|----------|----------|---------------|------|---------------------|----------|
| **Recreate** | Yes | Fast (redeploy old) | Low | No | Dev, batch jobs |
| **Rolling** | No | Slow (re-roll back) | Low | Yes (brief) | Standard production |
| **Blue-Green** | No | Instant (LB flip) | 2x | No | Critical services |
| **Canary** | No | Fast (reduce to 0%) | Low+ | Yes | High-traffic, risky changes |
| **Feature Flags** | No | Instant (toggle) | Low | Yes | Continuous deployment |

---

## 7. Quality Gates in CI

### 7.1 Unit Tests with JUnit 5

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <includes>
            <include>**/*Test.java</include>
            <include>**/*Tests.java</include>
        </includes>
        <!-- Fail if no tests run -->
        <failIfNoTests>true</failIfNoTests>
    </configuration>
</plugin>
```

**CI behavior:** `mvn verify` fails (non-zero exit) if any test fails. The pipeline stops.

---

### 7.2 Code Coverage with JaCoCo

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <!-- Instruments code before tests -->
        <execution>
            <id>prepare-agent</id>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <!-- Generates report after tests -->
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
        <!-- Fails build if coverage below threshold -->
        <execution>
            <id>check</id>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>    <!-- 80% line coverage -->
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

---

### 7.3 SonarQube Quality Gate

SonarQube evaluates: code coverage, code smells, bugs, vulnerabilities, duplications, technical debt.

**Quality Gate (default "Sonar Way"):**
- Coverage on new code >= 80%
- No new blocker issues
- No new critical vulnerabilities
- Duplication on new code <= 3%

**CI integration:** `waitForQualityGate` polls SonarQube until the gate passes or fails, then fails the pipeline accordingly.

---

### 7.4 SAST, DAST, Container Scanning

**SAST (Static Application Security Testing):**
Analyzes source code or bytecode for vulnerabilities without running the application.

| Tool | What It Checks | Integration |
|------|---------------|-------------|
| **OWASP Dependency Check** | Known CVEs in Maven dependencies | Maven plugin |
| **Snyk** | Dependencies, IaC, containers | CLI or GitHub Action |
| **SpotBugs** | Java bytecode bugs (null deref, SQL injection patterns) | Maven plugin |
| **Checkstyle** | Code style and conventions | Maven plugin |
| **PMD** | Code quality rules | Maven plugin |
| **SonarQube SAST** | Security hotspots in code | SonarQube plugin |

**DAST (Dynamic Application Security Testing):**
Tests a running application by sending malicious HTTP requests.

```yaml
# OWASP ZAP in GitHub Actions
- name: OWASP ZAP Baseline Scan
  uses: zaproxy/action-baseline@v0.10.0
  with:
    target: 'https://staging.myapp.com'
    rules_file_name: '.zap/rules.tsv'
    fail_action: warn    # warn or block
```

**Container scanning (Trivy):**
```bash
# Scan image for OS package CVEs and app dependencies
trivy image \
  --severity CRITICAL,HIGH \
  --exit-code 1 \
  myrepo/springboot-app:abc1234
```

---

## 8. Secrets Management

### 8.1 Never Commit Secrets

**What counts as a secret:** passwords, API keys, private keys, DB connection strings, tokens, certificates.

**How secrets leak into git:**
- `application.properties` committed with real credentials
- `.env` files not in `.gitignore`
- Hardcoded in test files
- Accidentally committed in a panic fix

**Prevention:**
```bash
# .gitignore
.env
*.env
application-local.properties
*-secret.yml
*.pem
*.key
```

**Secret scanning:** GitHub has built-in secret scanning. Enable it. Also use `git-secrets` or `truffleHog` as a pre-commit hook.

---

### 8.2 GitHub Secrets

```yaml
# In workflow: reference as ${{ secrets.SECRET_NAME }}
- name: Use secret
  run: |
    echo "${{ secrets.DATABASE_PASSWORD }}" | some-tool --password-stdin
  env:
    API_KEY: ${{ secrets.THIRD_PARTY_API_KEY }}
```

**Types:**
- **Repository secrets:** scoped to one repo
- **Environment secrets:** scoped to a specific environment (e.g., production secrets only available in production jobs)
- **Organization secrets:** shared across repos in an org

---

### 8.3 HashiCorp Vault

```yaml
# GitHub Actions: fetch secret from Vault
- name: Import Secrets from Vault
  uses: hashicorp/vault-action@v2
  with:
    url: https://vault.company.com
    token: ${{ secrets.VAULT_TOKEN }}
    secrets: |
      secret/data/myapp/db password | DB_PASSWORD;
      secret/data/myapp/api key | API_KEY

- name: Use fetched secret
  run: echo "DB password is set: ${{ env.DB_PASSWORD != '' }}"
```

**Why Vault:** Dynamic secrets (Vault generates DB credentials per-request that expire), audit log, fine-grained access policies, secret rotation.

---

### 8.4 Kubernetes Secrets vs External Secrets Operator

**Kubernetes Secrets (base64 encoded, not encrypted by default):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: cGFzc3dvcmQ=    # base64("password") — NOT encryption!
```

**External Secrets Operator (syncs from Vault/AWS Secrets Manager to K8s Secrets):**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: db-secret    # Creates a K8s Secret with this name
  data:
    - secretKey: password
      remoteRef:
        key: myapp/production/db
        property: password
```

---

## 9. Docker in CI/CD

### 9.1 Multi-Stage Build for Spring Boot

```dockerfile
# Multi-stage Dockerfile for Spring Boot

# ── Stage 1: Dependency resolution (cached layer) ──
FROM maven:3.9-eclipse-temurin-21 AS deps
WORKDIR /app
# Copy only pom.xml first — this layer is cached unless pom.xml changes
COPY pom.xml .
RUN mvn dependency:go-offline -q

# ── Stage 2: Build ──
FROM deps AS builder
COPY src ./src
RUN mvn clean package -DskipTests -q

# ── Stage 3: Extract layers for optimal caching ──
FROM eclipse-temurin:21-jre-alpine AS extractor
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# ── Stage 4: Production runtime (minimal image) ──
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

# Security: non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Copy in dependency order (least-frequently changed → most)
COPY --from=extractor /app/dependencies/ ./
COPY --from=extractor /app/spring-boot-loader/ ./
COPY --from=extractor /app/snapshot-dependencies/ ./
COPY --from=extractor /app/application/ ./

# JVM tuning for containers
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.JarLauncher"]
```

---

### 9.2 Docker Layer Caching in CI

**GitHub Actions Docker cache:**
```yaml
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha       # Read from GitHub Actions cache
    cache-to: type=gha,mode=max   # Write all layers to cache
```

**Registry-based cache (persists between runs):**
```yaml
- uses: docker/build-push-action@v5
  with:
    cache-from: type=registry,ref=myrepo/myapp:buildcache
    cache-to: type=registry,ref=myrepo/myapp:buildcache,mode=max
```

---

### 9.3 Kaniko for Rootless Builds in Kubernetes

When running CI/CD inside Kubernetes (e.g., self-hosted GitHub Actions runners in K8s), Docker-in-Docker requires privileged mode which is a security risk. Kaniko builds images without requiring root or Docker daemon.

```yaml
# Kubernetes Job using Kaniko
apiVersion: batch/v1
kind: Job
metadata:
  name: build-springboot-app
spec:
  template:
    spec:
      containers:
        - name: kaniko
          image: gcr.io/kaniko-project/executor:latest
          args:
            - "--context=git://github.com/myorg/myapp"
            - "--dockerfile=Dockerfile"
            - "--destination=myrepo/myapp:latest"
            - "--cache=true"              # Enable layer caching
            - "--cache-repo=myrepo/cache"
          volumeMounts:
            - name: docker-config
              mountPath: /kaniko/.docker
      volumes:
        - name: docker-config
          secret:
            secretName: registry-credentials
      restartPolicy: Never
```

---

### 9.4 Image Tagging Strategy

```bash
# Anti-pattern: 'latest' is mutable — you can't know what's running
docker tag myapp:abc1234 myapp:latest   # Every push overwrites latest

# Pattern 1: Git SHA (best for traceability)
IMAGE_TAG=$(git rev-parse --short HEAD)
docker build -t myapp:${IMAGE_TAG} .

# Pattern 2: Semantic version (for releases)
IMAGE_TAG="1.2.3"
docker build -t myapp:${IMAGE_TAG} .

# Pattern 3: Version + SHA (best of both)
IMAGE_TAG="1.2.3-abc1234"

# Pattern 4: Branch + SHA (for multi-environment pipelines)
IMAGE_TAG="${BRANCH_NAME}-${SHORT_SHA}"
# e.g., "main-abc1234", "feature-auth-def5678"
```

---

## 10. Deployment to Kubernetes

### 10.1 kubectl apply in CI/CD

```bash
# Basic: update image
kubectl set image deployment/springboot-app \
  springboot-app=myrepo/springboot-app:abc1234 \
  --namespace production

# Wait for rollout
kubectl rollout status deployment/springboot-app \
  --namespace production \
  --timeout=300s

# Rollback if needed
kubectl rollout undo deployment/springboot-app \
  --namespace production
```

**Kubernetes deployment manifest (full):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-app
  namespace: production
  annotations:
    deployment.kubernetes.io/revision: "1"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: springboot-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: springboot-app
        version: abc1234    # Update this in CI
    spec:
      containers:
        - name: springboot-app
          image: myrepo/springboot-app:abc1234
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: production
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
```

---

### 10.2 Helm Chart Deployment

```bash
# Deploy with Helm in CI
helm upgrade --install springboot-app ./charts/springboot-app \
  --namespace production \
  --values ./charts/springboot-app/values-production.yaml \
  --set image.tag=${IMAGE_TAG} \
  --set image.repository=${ECR_REGISTRY}/${ECR_REPOSITORY} \
  --wait \                # Wait for rollout to complete
  --timeout 5m \
  --atomic               # Rollback automatically on failure
```

**Helm values files per environment:**
```yaml
# values-production.yaml
replicaCount: 3
image:
  repository: myrepo/springboot-app
  tag: latest    # Overridden by --set in CI

resources:
  requests:
    memory: 512Mi
    cpu: 500m
  limits:
    memory: 1Gi
    cpu: 1000m

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10

ingress:
  enabled: true
  hostname: myapp.com
```

---

### 10.3 ArgoCD (GitOps — Pull-Based CD)

ArgoCD watches a Git repository. When the manifest changes, ArgoCD pulls and applies it to Kubernetes — the pipeline doesn't need cluster access.

```
Developer pushes code
       ↓
CI pipeline: build image → push to ECR → update image tag in Git manifest repo
       ↓
ArgoCD detects Git change → pulls new manifest → applies to K8s cluster
```

```yaml
# ArgoCD Application manifest
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: springboot-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/k8s-manifests
    targetRevision: main
    path: apps/springboot-app/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true          # Remove resources deleted from Git
      selfHeal: true       # Re-apply if someone manually changes K8s
    syncOptions:
      - CreateNamespace=true
```

**CI pipeline step to update the manifest:**
```bash
# In GitHub Actions — update image tag in manifest repo
git clone https://github.com/myorg/k8s-manifests
cd k8s-manifests
# Update image tag using yq or sed
yq -i ".image.tag = \"${IMAGE_TAG}\"" apps/springboot-app/production/values.yaml
git add .
git commit -m "ci: update springboot-app image to ${IMAGE_TAG}"
git push
# ArgoCD detects this push and auto-deploys
```

---

## 11. Environment Promotion

### 11.1 Build Once, Deploy Everywhere

```
         ┌─────────┐
         │ Build   │  → Docker image: myapp:abc1234
         └────┬────┘
              │ push to ECR
              ▼
         ┌─────────┐
         │   Dev   │  kubectl set image ... myapp:abc1234
         └────┬────┘
              │ smoke tests pass
              ▼
         ┌─────────┐
         │ Staging │  kubectl set image ... myapp:abc1234 (same image!)
         └────┬────┘
              │ integration/performance tests pass
              │ manual approval
              ▼
         ┌────────────┐
         │ Production │  kubectl set image ... myapp:abc1234 (same image!)
         └────────────┘
```

**What changes per environment:** ConfigMaps, Secrets, replica counts, resource limits — not the image.

---

### 11.2 Environment-Specific Configuration

```yaml
# Kubernetes ConfigMap per environment
apiVersion: v1
kind: ConfigMap
metadata:
  name: springboot-config
  namespace: production
data:
  application.yml: |
    spring:
      datasource:
        url: jdbc:postgresql://prod-db.internal:5432/myapp
      redis:
        host: prod-redis.internal
    logging:
      level:
        root: WARN
```

```yaml
# Helm: different values per environment
# deploy with: helm upgrade ... -f values-production.yaml
# values-dev.yaml
replicaCount: 1
database:
  url: jdbc:postgresql://dev-db:5432/myapp_dev

# values-production.yaml
replicaCount: 3
database:
  url: jdbc:postgresql://prod-db.internal:5432/myapp_prod
```

---

## 12. Monitoring Post-Deployment

### 12.1 Smoke Tests After Deployment

A smoke test is a minimal set of fast checks that verify the application started and its critical paths work.

```bash
#!/bin/bash
# smoke-test.sh — run after every deployment
APP_URL=${1:-"https://myapp.com"}

echo "Running smoke tests against $APP_URL..."

# Health check
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$APP_URL/actuator/health")
[ "$HTTP_STATUS" = "200" ] || { echo "FAIL: health check returned $HTTP_STATUS"; exit 1; }

# Critical API endpoint
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$APP_URL/api/v1/products")
[ "$HTTP_STATUS" = "200" ] || { echo "FAIL: products API returned $HTTP_STATUS"; exit 1; }

# Auth endpoint
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -X POST "$APP_URL/api/v1/auth/health" \
  -H "Content-Type: application/json")
[ "$HTTP_STATUS" != "500" ] || { echo "FAIL: auth endpoint 500"; exit 1; }

echo "All smoke tests passed."
```

---

### 12.2 Automatic Rollback on Metric Degradation

```yaml
# Argo Rollouts: auto-rollback on error rate spike
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.95    # 95% success rate
      failureLimit: 3                         # Fail after 3 consecutive bad readings
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{job="springboot-app",status!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{job="springboot-app"}[5m]))
```

**Kubernetes deployment rollback in CI:**
```bash
# In pipeline smoke test step:
if ! ./smoke-test.sh; then
  echo "Smoke test failed — rolling back deployment"
  kubectl rollout undo deployment/springboot-app --namespace production
  kubectl rollout status deployment/springboot-app --namespace production
  exit 1
fi
```

---

## 13. Interview Questions & Answers

### Q1: What is the difference between Continuous Integration, Continuous Delivery, and Continuous Deployment?

**A:** CI means every developer integrates code to the shared branch multiple times per day, and each integration triggers an automated build and test. The goal is early detection of integration problems.

Continuous Delivery extends CI by ensuring the software is always in a deployable state. Every successful CI build produces an artifact that could be deployed to production with a single manual decision. The automation ensures the artifact is tested and ready; the business decides when to release.

Continuous Deployment goes one step further — the human approval gate is removed. Every commit that passes all automated checks is automatically deployed to production. This requires extremely high confidence in the test suite and robust monitoring with automatic rollback.

---

### Q2: Walk me through a CI/CD pipeline you have set up.

**A:** In my most recent project, we had a GitHub Actions pipeline with three jobs. The first job ran on every push and PR: it checked out the code, set up JDK 21 with Maven cache, ran `mvn verify` which compiled, ran unit tests, and checked JaCoCo coverage (we had an 80% minimum). It also ran a SonarQube analysis and failed the pipeline if the quality gate didn't pass.

The second job ran only on merges to main. It built a multi-stage Docker image — a JDK stage to compile, then a minimal JRE runtime image — and pushed it to AWS ECR with the git commit SHA as the tag. We used OIDC authentication so there were no long-lived AWS credentials in our secrets.

The third job deployed to our EKS cluster by running `kubectl set image` and waiting for the rollout to complete. For production, the job was tied to a GitHub environment that required a reviewer approval before running. After deployment, a smoke test hit `/actuator/health` and if it returned non-200, the pipeline ran `kubectl rollout undo` automatically.

---

### Q3: How do you handle database migrations in a CI/CD pipeline?

**A:** The standard approach is to use Flyway or Liquibase with forward-compatible (backward-compatible) migrations.

The key principle is: **never make a breaking schema change in the same deployment as the code change that requires it.** Instead, use a multi-step process:

1. **Deploy migration only:** Add a new nullable column or new table. The old code runs fine against this schema since the new column is nullable and the old code ignores it.
2. **Deploy application code:** The new code now starts using the new column.
3. **Cleanup migration (if needed):** Make the column NOT NULL, drop old columns.

Flyway integrates with Spring Boot — it runs automatically at startup via `spring.flyway.enabled=true`. Migrations are in `src/main/resources/db/migration/V1__create_users.sql`. The Spring Boot app applies pending migrations when it starts. If a migration fails, the app startup fails and the deployment fails, which triggers the rollback.

For CI, we have a separate integration test job that starts a real Postgres container, applies all Flyway migrations, and runs integration tests. This catches migration errors before they reach production.

---

### Q4: What is the difference between blue-green and canary deployments?

**A:** In blue-green, you maintain two identical production environments. You deploy the new version to the idle (green) environment, verify it works, then flip the load balancer to send 100% of traffic from blue to green in a near-instant cutover. Rollback is equally instant — just flip the load balancer back. The cost is roughly double the infrastructure.

In canary, you don't switch all traffic at once. You route a small percentage — say 5% — of real production traffic to the new version while 95% continues to use the old version. You monitor error rates, latency, and business metrics for the canary. If everything looks good you gradually increase the percentage: 5% → 20% → 50% → 100%. If metrics degrade, you route 0% to the canary and it disappears. The blast radius of a bad deploy is inherently limited.

Blue-green is better when you want instant rollback and can afford the cost. Canary is better for high-traffic services where you want to validate the new version against real traffic before committing, and when you have good observability to detect problems.

---

### Q5: How do you manage secrets in a CI/CD pipeline?

**A:** The primary rule is: secrets never live in code or config files in git.

For GitHub Actions, we store secrets in GitHub Secrets (repository or environment-scoped). For AWS authentication, we use OIDC so the pipeline assumes an IAM role with short-lived credentials — no static access keys stored anywhere. For application secrets at runtime, we use AWS Secrets Manager or HashiCorp Vault with the External Secrets Operator, which syncs secrets into Kubernetes Secrets without ever touching git.

We also use secret scanning on the repository — GitHub's built-in scanner plus pre-commit hooks with `truffleHog` to catch accidental commits. For production environment secrets, they're only available to jobs that reference the "production" GitHub environment, which itself requires reviewer approval.

---

### Q6: What is GitOps? What tools support it?

**A:** GitOps is a deployment practice where the desired state of your infrastructure and application is declared in Git, and an automated agent continuously reconciles the live cluster to match that desired state. Git is the single source of truth. Changes happen by making PRs to the Git repo, not by running `kubectl apply` manually.

The flow: a developer merges a change → CI builds and pushes the Docker image → CI commits the new image tag to the "manifests repo" → the GitOps agent (running inside the cluster) detects the change and applies it.

The key difference from traditional push-based CD: the CI pipeline does not need cluster credentials. The cluster pulls from Git, rather than the pipeline pushing to the cluster. This improves security and makes the system self-healing — if someone manually changes a K8s resource, the GitOps agent reverts it to what Git says it should be.

Tools: **ArgoCD** (most popular, excellent UI, multi-cluster), **Flux CD** (CNCF graduated, GitOps toolkit approach), Rancher Fleet, Jenkins X.

---

### Q7: What is trunk-based development and why is it preferred for CI/CD?

**A:** Trunk-based development means all developers integrate their work to a single shared branch (the "trunk" or `main`) at least once per day. Feature branches exist but are very short-lived — measured in hours or a couple of days, never weeks. Incomplete features are deployed behind feature flags, not kept in branches.

It's preferred for CI/CD because it makes Continuous Integration real. If every developer integrates daily, the `main` branch is always close to a deployable state and integration conflicts are tiny. With GitFlow's long-lived feature branches, you might have weeks of divergence, making CI more of a formality than a genuine integration check.

Feature flags are the enabler: you can deploy unfinished code to production without exposing it to users. When the feature is ready, you flip the flag — no deployment needed. This makes the deployment event boring and low-risk, which is the goal of CI/CD.

---

### Q8: How do you prevent a bad deploy from reaching production?

**A:** It's a layered defence:

1. **Automated tests in CI** — unit tests, integration tests, coverage gates. Don't let untested code reach the build stage.
2. **SonarQube quality gate** — code quality, security hotspots, vulnerability checks must pass.
3. **Container vulnerability scanning** — Trivy scans the Docker image and fails the pipeline on CRITICAL/HIGH CVEs.
4. **Branch protection** — the `main` branch requires PRs with passing status checks and at least one reviewer approval before merging.
5. **Deploy to staging first** — the same image that passed CI is deployed to staging before production. Integration tests and smoke tests run there.
6. **Environment approval** — production deployment requires explicit human approval via GitHub Environments.
7. **Canary or rolling deployment** — even if something slips through, only a fraction of traffic hits the new version initially.
8. **Post-deploy smoke test** — the pipeline checks the health endpoint immediately after deployment and rolls back automatically if it fails.
9. **Monitoring and alerting** — Prometheus/Grafana watches error rate and latency. A spike triggers a PagerDuty alert, and Argo Rollouts can auto-rollback if configured.

---

### Q9: What happens when a pipeline fails midway?

**A:** The pipeline stops at the failing stage — subsequent stages don't run. The developer gets notified (Slack, email) with a link to the failing step and its logs.

The failed commit is not deployed. If the failure is in the build or test stage, the code never even reaches Docker build. If Docker push failed, nothing was deployed. If the deployment step failed (e.g., `kubectl rollout status` timed out), Kubernetes itself would detect that the new pods are failing readiness probes and stop the rolling update, leaving the old pods serving traffic.

The developer investigates the logs, fixes the issue, pushes a new commit, which triggers the pipeline from the beginning. The failed run is immutable — we don't retry midway (with the exception of flaky tests where we might add retry logic at the step level).

In Jenkins, `post { failure { ... } }` blocks send notifications and can trigger cleanup jobs. In GitHub Actions, steps with `if: always()` run regardless of failure (useful for uploading test reports even when tests fail).

---

### Q10: How do you implement zero-downtime deployments?

**A:** For zero-downtime, several pieces must work together:

1. **Rolling update or blue-green strategy** in Kubernetes — `maxUnavailable: 0` ensures pods are only terminated after new ones pass readiness checks.
2. **Readiness and liveness probes** — Kubernetes only routes traffic to pods that pass the readiness probe. A Spring Boot app exposes `/actuator/health/readiness`. Until the app finishes startup and the probe passes, no traffic is sent to the new pod.
3. **Graceful shutdown** — the old pod must finish in-flight requests before shutting down. In Spring Boot: `server.shutdown=graceful` and `spring.lifecycle.timeout-per-shutdown-phase=30s`.
4. **Backward-compatible APIs** — during a rolling update, v1 and v2 pods coexist. Client requests might hit either. New API versions must not break existing clients.
5. **Backward-compatible database migrations** — as described in Q3, never drop a column or change its type while old code is still running.

---

### Q11: What is SAST vs DAST?

**A:** SAST (Static Application Security Testing) analyzes source code, bytecode, or binaries without running the application. It finds vulnerabilities at rest: insecure API usage, SQL injection patterns, hardcoded credentials, known-vulnerable library versions (CVEs). Tools: OWASP Dependency Check, Snyk, SpotBugs, SonarQube Security. Runs early in the pipeline (build/scan stage), fast, no running app needed.

DAST (Dynamic Application Security Testing) tests a running application by sending it HTTP requests that simulate attacks: XSS payloads, SQL injection, broken auth, insecure headers. It can find runtime vulnerabilities SAST cannot (e.g., a SQL injection vulnerability in a dynamically constructed query). Tools: OWASP ZAP, Burp Suite. Runs late in the pipeline (after deployment to staging), slower, requires a deployed app.

Best practice: use both. SAST for early, cheap feedback in CI. DAST against staging for runtime validation.

---

### Q12: How do you test a Docker image in CI?

**A:** Several levels:

1. **Build verification** — if `docker build` succeeds, the image is syntactically valid. Failing builds immediately fail the pipeline.
2. **Container structure tests** (Google's `container-structure-test`): verify the image has the right files, environment variables, exposed ports, and that the entrypoint command runs.
3. **Trivy vulnerability scan** — scan for OS package CVEs and application dependency CVEs in the image.
4. **Smoke test against a running container:**

```bash
# Start the container in CI
docker run -d --name test-app -p 8080:8080 myapp:abc1234

# Wait for startup
sleep 10

# Hit the health endpoint
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/actuator/health)
docker stop test-app

[ "$HTTP_STATUS" = "200" ] || exit 1
```

5. **Integration tests against a running container** using Docker Compose in CI — spin up the app + dependencies (Postgres, Redis) and run a suite of API-level tests.

---

### Q13: How do you handle a hotfix when your main branch has changes you don't want to release?

**A:** This is a GitFlow scenario. You branch `hotfix/fix-critical-bug` directly from the production tag on `main`. You apply the minimal fix, test it, merge it back to `main` (new production tag), and also merge it to `develop` so the fix is incorporated in the next release.

In trunk-based development, the problem is less common because `main` is always near-deployable. But if `main` has in-progress work that isn't ready, you have two options: use a feature flag to hide the incomplete work (so deploying main is safe), or cherry-pick just the hotfix commit to a release branch.

---

### Q14: What is the difference between `needs` and `if` in GitHub Actions?

**A:** `needs` controls **execution order** — a job with `needs: [build]` will not start until the `build` job completes successfully. If `build` fails, the dependent job is skipped.

`if` controls **whether a job or step runs at all** based on a boolean expression. For example: `if: github.ref == 'refs/heads/main'` skips the deploy job entirely for PRs.

They work together:
```yaml
deploy:
  needs: build           # Only run after build succeeds
  if: github.event_name == 'push'   # Only run on direct pushes, not PRs
```

---

### Q15: What is the "artifact promotion" pattern?

**A:** Build the Docker image once in CI, push it to the registry with the git SHA tag, and use that exact same image through all environments. You never rebuild for staging or production.

This gives you certainty: the binary that was tested in CI is the binary that reaches production. If you rebuild, even from the same source, subtle differences in build environment (tool versions, transient network dependencies) could produce a different artifact.

The promotion is tracked in your manifest repository or deployment config: "image `myapp:abc1234` has been promoted to staging, and now promoted to production."

---

### Q16: Explain the concept of "shift-left" in CI/CD.

**A:** Shift-left means moving security and quality checks earlier (to the left) in the development timeline. Instead of discovering a SQL injection vulnerability during a pentest before release (far right), you detect it in the developer's IDE with SonarLint, in the pre-commit hook with static analysis, or at latest in the CI pipeline's SAST scan — all much earlier and much cheaper to fix.

The cost of fixing a bug grows exponentially the later it is found. A bug caught in unit tests takes minutes to fix. A security vulnerability discovered in production can take days and trigger incident response.

Practical shift-left practices: IDE plugins (SonarLint), pre-commit hooks (checkstyle, SpotBugs), SAST in CI, code review gates on PRs, dependency scanning early in the build stage.

---

### Q17: How would you set up CI/CD for a microservices architecture with 15 services?

**A:** You avoid a monorepo pipeline that rebuilds all 15 services on every commit. Instead:

1. **Path-based triggers** — each service's pipeline only runs when files in its directory change:
```yaml
on:
  push:
    paths:
      - 'services/user-service/**'
      - 'shared-libs/**'
```

2. **Shared pipeline templates** — use GitHub Actions reusable workflows or Jenkins shared libraries. Each service's Jenkinsfile is 10 lines that call the shared `buildSpringBoot()` function. Build logic lives in one place.

3. **Independent deployments** — each service has its own pipeline, its own ECR repository, and deploys independently. No "big bang" releases.

4. **Shared integration tests** — a separate pipeline that runs after multiple services are deployed to staging, testing cross-service interactions.

5. **Dependency tracking** — if `shared-lib` changes, trigger tests for all services that depend on it.

---

### Q18: What is a self-healing pipeline?

**A:** A pipeline that automatically detects and recovers from deployment failures without human intervention.

Implementation:
- Readiness probes in Kubernetes prevent bad pods from receiving traffic
- `kubectl rollout status --timeout=300s` fails the pipeline if pods don't become ready
- The pipeline step catches this failure and runs `kubectl rollout undo`
- Argo Rollouts with analysis can automatically roll back if error rates exceed thresholds
- Alerts are sent so humans are aware, but the service was never down

The key is: the pipeline knows the definition of "success" (all pods healthy, smoke tests pass, error rate below threshold) and if that definition isn't met, it returns to the last known-good state automatically.

---

### Q19: What is the difference between `docker build` caching and GitHub Actions caching?

**A:** They're different cache systems serving different purposes.

**Docker layer cache:** Docker caches each layer of a Dockerfile. If a layer's input (the instruction + files it depends on) hasn't changed, Docker reuses the cached layer. In CI, this means: if only your Java source changes, Docker doesn't re-download your Maven dependencies (assuming you structured your Dockerfile to `COPY pom.xml` and `RUN mvn dependency:go-offline` before copying `src/`).

**GitHub Actions cache (`actions/cache`):** Stores arbitrary files between workflow runs. Used for Maven's `~/.m2` directory, npm's `node_modules`, etc. The cache is keyed by a hash (e.g., `hashFiles('**/pom.xml')`). If the key matches, the directory is restored before the build step, saving the download time.

They complement each other: Actions cache saves the Maven dependencies download; Docker layer cache saves the `mvn dependency:go-offline` step in the Dockerfile.

---

### Q20: How do you roll back a Helm release?

**A:** Helm maintains a release history. Each `helm upgrade` creates a new revision.

```bash
# View history
helm history springboot-app --namespace production

# Rollback to previous revision
helm rollback springboot-app --namespace production

# Rollback to a specific revision
helm rollback springboot-app 3 --namespace production

# Helm upgrade with --atomic handles this automatically:
# If the upgrade fails, it auto-rolls back to the previous release
helm upgrade --install springboot-app ./chart \
  --namespace production \
  --atomic \
  --timeout 5m
```

The `--atomic` flag is the CI/CD best practice — if any pod fails to become ready within the timeout, Helm automatically reverts to the previous release.

---

### Q21: What is the purpose of `readinessProbe` vs `livenessProbe` in K8s deployments?

**A:**

**readinessProbe:** "Is this pod ready to receive traffic?" Kubernetes only routes service traffic to pods that pass the readiness probe. During startup, a Spring Boot app takes 10-20 seconds to initialize. The readiness probe prevents the load balancer from sending traffic to an initializing pod (which would return 503). Also used for graceful shutdown — the pod removes itself from the load balancer before stopping.

**livenessProbe:** "Is this pod alive and should keep running?" If the liveness probe fails, Kubernetes restarts the pod. Used to detect deadlocks — the app is still running (the process didn't crash) but is stuck in a bad state and not processing requests.

For Spring Boot: map readiness to `/actuator/health/readiness` and liveness to `/actuator/health/liveness`. These are separate endpoints since Spring Boot 2.3, allowing fine-grained control.

---

### Q22: How do you manage environment-specific configuration across dev, staging, and production?

**A:** Several levels:

1. **Spring profiles** — `application-dev.yml`, `application-prod.yml`. Set `SPRING_PROFILES_ACTIVE=production` as an environment variable in the container. The profile file has non-sensitive config like timeouts, pool sizes, log levels.

2. **Kubernetes ConfigMaps** — for non-sensitive config that differs per environment. The same Docker image reads from the ConfigMap at runtime.

3. **Kubernetes Secrets / External Secrets Operator** — for credentials, API keys, database passwords. Never in config files.

4. **Helm values files** — `values-dev.yaml`, `values-staging.yaml`, `values-production.yaml` control replica counts, resource limits, ingress hostnames, etc. In CI: `helm upgrade ... -f values-${ENVIRONMENT}.yaml`.

The rule: the Docker image contains no environment-specific config. All environment differences are injected at runtime via environment variables and mounted config files.

---

### Q23: What is a "flaky test" and how do you handle it in CI?

**A:** A flaky test passes sometimes and fails other times without any code changes. Common causes: time-dependent assertions, race conditions in async code, tests that depend on external services or specific system state, port conflicts in parallel tests.

How to handle:
1. **Identify** — a test that fails intermittently in CI but always passes locally. GitHub Actions shows test history and you can see the pattern.
2. **Quarantine** — tag the test with `@Disabled` or move it to a separate "flaky" test suite that runs non-blocking (failure doesn't block the pipeline). This stops the noise while you investigate.
3. **Fix** — find the root cause. Mock the external dependency. Fix the race condition. Use `Awaitility` for async assertions instead of `Thread.sleep`.
4. **Retry as last resort** — Surefire allows `<rerunFailingTestsCount>2</rerunFailingTestsCount>` to retry once before marking as failed. Use sparingly — it hides real problems.

---

### Q24: What is the purpose of a `post` section in a Jenkinsfile?

**A:** The `post` section in a Declarative Jenkinsfile defines steps that always run at the end of a pipeline, regardless of success or failure. It's structured by condition:

- `always`: Runs no matter what (cleanup, publish test results)
- `success`: Runs only on success (deploy notification, artifact promotion)
- `failure`: Runs only on failure (alert team, create incident ticket)
- `unstable`: Runs when tests pass with warnings (coverage below threshold)
- `aborted`: Runs if manually cancelled
- `changed`: Runs if the build result changed from the previous run (e.g., first failure or first recovery)

It's the equivalent of a `finally` block in Java, combined with conditional notification logic.

---

### Q25: What are branch protection rules and why are they important in CI/CD?

**A:** Branch protection rules in GitHub prevent direct pushes to important branches (like `main`) and enforce quality gates before merging.

Typical rules for `main`:
- **Require pull request** — no direct commits, must go through a PR
- **Require status checks** — CI pipeline must pass before merging
- **Require review** — at least one (or two) approvals required
- **Dismiss stale reviews** — if new commits are pushed to a PR, existing approvals are dismissed
- **Require branches to be up to date** — PR must be based on the latest main before merging
- **Restrict who can push** — only specific teams or admins can bypass rules

Without branch protection, a developer could merge untested code or skip review. With it, the CI pipeline is the mandatory gate — nothing reaches main that hasn't passed all checks.

---

## 14. Quick Reference Cheat Sheet

### GitHub Actions Key Syntax

```yaml
on: push / pull_request / schedule / workflow_dispatch / workflow_call
runs-on: ubuntu-latest / windows-latest / macos-latest / self-hosted
needs: [job1, job2]                          # Job dependency
if: github.ref == 'refs/heads/main'          # Conditional
environment: production                       # Requires approval
${{ secrets.MY_SECRET }}                     # Secret reference
${{ github.sha }}                            # Commit SHA
${{ github.ref_name }}                       # Branch name
${{ matrix.java }}                           # Matrix value
echo "key=value" >> $GITHUB_OUTPUT           # Set step output
echo "VAR=value" >> $GITHUB_ENV              # Set env for subsequent steps
```

### Key Maven Commands in CI

```bash
mvn clean verify                # Compile + test + coverage check
mvn clean package -DskipTests   # Build JAR, skip tests
mvn sonar:sonar                 # SonarQube analysis
mvn dependency-check:check      # OWASP dependency scan
mvn --batch-mode                # Non-interactive (CI-friendly output)
mvn --no-transfer-progress      # Suppress download progress
```

### kubectl Commands in CI

```bash
kubectl set image deployment/app container=image:tag --namespace ns
kubectl rollout status deployment/app --namespace ns --timeout=300s
kubectl rollout undo deployment/app --namespace ns
kubectl apply -f manifests/ --namespace ns
kubectl get pods --namespace ns
kubectl logs deployment/app --namespace ns
```

### Deployment Strategy When-To-Use

| Situation | Strategy |
|-----------|----------|
| Dev/test environment | Recreate |
| Standard production | Rolling (K8s default) |
| Need instant rollback, cost ok | Blue-Green |
| High traffic, risky change | Canary |
| Decouple deploy from release | Feature Flags |

### Pipeline Stage Ownership

| Stage | Owner | Typical Tools |
|-------|-------|--------------|
| Source | SCM | GitHub, GitLab, Bitbucket |
| Build | Developer | Maven, Gradle |
| Unit Test | Developer | JUnit 5, Mockito |
| Code Quality | Developer + Architect | SonarQube, Checkstyle |
| SAST | Security team | OWASP Dep Check, Snyk |
| Package | DevOps | Docker, Maven |
| Container Scan | Security | Trivy |
| Deploy Staging | DevOps | kubectl, Helm, ArgoCD |
| Integration Test | QA | RestAssured, Cucumber |
| Deploy Production | DevOps + Approval | kubectl, Helm, ArgoCD |
| Smoke Test | QA/DevOps | curl, k6, Postman |
| Monitor | DevOps | Prometheus, Grafana |
