# CI/CD Pipelines — Deep Dive for Full Stack Java Developer Interviews

## 1. CI/CD Fundamentals

### 1.1 Definitions

**Continuous Integration (CI):** Every developer merges code to the shared main branch multiple times per day. Each merge triggers an automated build and test. Goal: detect integration problems in minutes, not days.

**Continuous Delivery (CD — Delivery):** Every successful CI run produces an artifact that *could* be deployed to production with a single manual approval. The release decision is business-driven, not engineering-blocked.

**Continuous Deployment (CD — Deployment):** Every commit that passes automated checks is automatically deployed to production — no human touches the deploy button. Requires extremely high test coverage, feature flags, and automated rollback.

```
Continuous Integration   → every commit: build + test → artifact ready
Continuous Delivery      → CI + artifact can deploy any time (human clicks deploy)
Continuous Deployment    → CI + artifact is automatically deployed to production
```

---

### 1.2 Benefits

| Benefit | Without CI/CD | With CI/CD |
|---------|--------------|-----------|
| Integration risk | "Integration hell" at sprint end | Caught in minutes |
| Feedback speed | Days to weeks | Minutes |
| Deployment frequency | Monthly/quarterly | Multiple times per day |
| Deployment reliability | Manual, error-prone | Automated, repeatable |
| Mean Time to Recovery | Hours or days | Minutes |

---

### 1.3 Pipeline Stages

```
Source → Build → Test → Scan → Package → Deploy → Verify
```

| Stage | What Happens | Fail Behaviour |
|-------|-------------|----------------|
| **Source** | Checkout code | Block pipeline |
| **Build** | Compile, resolve dependencies | Block |
| **Test** | Unit/integration tests, coverage | Block |
| **Scan** | SAST, dependency CVE scan, code quality | Block or warn |
| **Package** | Build Docker image, push to registry | Block |
| **Deploy** | Apply manifests to target environment | Block next stage |
| **Verify** | Smoke tests, health checks | Trigger rollback |

Each stage must pass before the next runs — expensive stages only run if cheap stages pass.

---

### 1.4 Artifacts

An **artifact** is an immutable, versioned build output that can be deployed to any environment. Common types: JAR, Docker image, Helm chart.

**Why immutable:** Build once, promote the same artifact through dev → staging → production. Rebuilding per environment breaks the guarantee that what was tested is what was deployed.

**Docker image tagging:**
```
myapp:latest          ← ANTI-PATTERN (mutable)
myapp:abc1234         ← Git commit SHA (best for traceability)
myapp:1.2.3-abc1234   ← Version + SHA (best of both worlds)
```

**Artifact repositories:** Nexus / JFrog Artifactory (self-hosted enterprise), AWS ECR (Docker, managed), GitHub Packages (built-in), Docker Hub (public).

---

### 1.5 Environment Promotion

```
dev → staging → production
auto deploy  auto deploy  manual gate (or auto for CD)
```

The same Docker image SHA is deployed to every environment. Only config (env vars, secrets) differs — this is "build once, deploy everywhere."

---

## 2. GitHub Actions — Deep Dive

### 2.1 Core Concepts

| Concept | Description |
|---------|-------------|
| **Workflow** | YAML file in `.github/workflows/`. Triggered by events. |
| **Event** | What starts the workflow: push, PR, schedule, manual, etc. |
| **Job** | Steps that run on the same runner. Jobs run in parallel by default. |
| **Step** | A single task: shell command (`run`) or action (`uses`). |
| **Action** | Reusable unit from the marketplace, your repo, or Docker. |
| **Runner** | The VM/container executing the job. |

---

### 2.2 Workflow YAML Structure

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'         # Every Monday at 2am UTC
  workflow_dispatch:             # Manual trigger from GitHub UI

env:
  JAVA_VERSION: '21'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: mvn test
        env:
          DB_URL: ${{ secrets.TEST_DB_URL }}

  deploy:
    runs-on: ubuntu-latest
    needs: [build]                          # Wait for 'build' job
    if: github.ref == 'refs/heads/main'    # Only on main branch
    steps:
      - run: echo "Deploying..."
```

---

### 2.3 Jobs: Dependencies, Matrix, Conditionals

```yaml
jobs:
  test:
    strategy:
      matrix:
        java: [17, 21]
      fail-fast: false
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java }}
          distribution: temurin
      - run: mvn test

  build-docker:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - run: docker build .

  deploy-production:
    needs: [build-docker, integration-test]   # Both must pass
    environment: production                    # Requires reviewer approval
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy to production"
```

---

### 2.4 Contexts

```yaml
steps:
  - run: echo "Branch: ${{ github.ref_name }}, SHA: ${{ github.sha }}"

  - name: Set a step output
    id: get-version
    run: echo "version=1.2.3" >> $GITHUB_OUTPUT

  - run: echo "Version is ${{ steps.get-version.outputs.version }}"
```

Key contexts: `github.sha`, `github.ref_name`, `github.event_name`, `secrets.*`, `matrix.*`, `steps.<id>.outputs.*`.

---

### 2.5 Complete Spring Boot CI/CD Pipeline (GitHub Actions)

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
  JAVA_VERSION: '21'

jobs:
  build-and-test:
    name: Build, Test & Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0              # Full history for SonarQube

      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin
          cache: maven

      - name: Build and test
        run: mvn clean verify --batch-mode --no-transfer-progress

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: jacoco-report
          path: target/site/jacoco/

      - name: SonarQube Scan
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          mvn sonar:sonar \
            -Dsonar.projectKey=my-springboot-app \
            -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }}

      - name: OWASP Dependency Check
        run: mvn org.owasp:dependency-check-maven:check -DfailBuildOnCVSS=7

      - name: Upload JAR
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar

  build-docker:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: build-and-test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - uses: actions/download-artifact@v4
        with:
          name: app-jar
          path: target/

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: 1

  deploy:
    name: Deploy to Kubernetes
    runs-on: ubuntu-latest
    needs: build-docker
    environment: production          # Requires reviewer approval

    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name my-eks-cluster --region ${{ env.AWS_REGION }}

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/springboot-app \
            springboot-app=${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }} \
            --namespace production
          kubectl rollout status deployment/springboot-app --namespace production --timeout=300s

      - name: Smoke test
        run: |
          HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" ${{ secrets.APP_URL }}/actuator/health)
          if [ "$HTTP_STATUS" != "200" ]; then
            kubectl rollout undo deployment/springboot-app --namespace production
            exit 1
          fi
```

**Multi-stage Dockerfile:**
```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

---

### 2.6 Caching Maven Dependencies

```yaml
# Option 1: Built-in (simplest)
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: temurin
    cache: maven

# Option 2: Manual cache (more control)
- uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
    restore-keys: ${{ runner.os }}-maven-
```

Caching reduces pipeline time from ~4 minutes to ~1 minute by avoiding 200MB of repeated downloads.

---

### 2.7 OIDC for AWS Authentication

Instead of static AWS access keys, OIDC lets GitHub exchange a short-lived token for an IAM role. Credentials expire in ~1 hour — no leaked long-lived keys.

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions-role
    aws-region: us-east-1
```

---

### 2.8 Reusable Workflows and Environments

**Reusable workflow** (DRY technique — invoke shared deploy/build logic):
```yaml
jobs:
  deploy-staging:
    uses: ./.github/workflows/deploy-reusable.yml
    with:
      environment: staging
      image-tag: ${{ github.sha }}
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

**Environments with required reviewers:** Configure in GitHub Settings → Environments → production. Add required reviewers and environment-scoped secrets. Reference in a job with `environment: production` — the job pauses until approved.

---

## 3. Jenkins Pipeline

### 3.1 Declarative vs Scripted

| | Declarative | Scripted |
|---|---|---|
| **Syntax** | Structured blocks | Groovy DSL, full flexibility |
| **Error handling** | Built-in `post` blocks | `try/catch/finally` |
| **Learning curve** | Lower | Higher |
| **When to use** | Standard pipelines (95%) | Complex conditional logic |

---

### 3.2 Complete Declarative Jenkinsfile for Spring Boot

```groovy
pipeline {
    agent any
    tools { jdk 'JDK21'; maven 'Maven3' }

    environment {
        APP_NAME        = 'springboot-app'
        DOCKER_REGISTRY = 'your-ecr-url.amazonaws.com'
        IMAGE_TAG       = "${BUILD_NUMBER}-${GIT_COMMIT.take(7)}"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        timestamps()
    }

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['dev', 'staging', 'production'])
        booleanParam(name: 'SKIP_TESTS', defaultValue: false)
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build') {
            steps { sh 'mvn clean compile --batch-mode -q' }
        }

        stage('Test') {
            when { not { params.SKIP_TESTS } }
            steps { sh 'mvn test --batch-mode' }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                    jacoco(execPattern: 'target/jacoco.exec', minimumLineCoverage: '80')
                }
            }
        }

        stage('Code Quality') {
            parallel {
                stage('SonarQube') {
                    steps {
                        withSonarQubeEnv('SonarQube') {
                            sh 'mvn sonar:sonar -Dsonar.projectKey=${APP_NAME}'
                        }
                        timeout(time: 5, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: true
                        }
                    }
                }
                stage('OWASP Check') {
                    steps {
                        sh 'mvn org.owasp:dependency-check-maven:check -DfailBuildOnCVSS=7'
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
            when { anyOf { branch 'main'; branch 'release/*' } }
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                      credentialsId: 'aws-ecr-credentials']]) {
                        sh """
                            aws ecr get-login-password --region us-east-1 | \
                                docker login --username AWS --password-stdin ${DOCKER_REGISTRY}
                            docker build -t ${APP_NAME}:${IMAGE_TAG} .
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
                            ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG} --namespace dev
                        kubectl rollout status deployment/${APP_NAME} --namespace dev --timeout=120s
                    """
                }
            }
        }

        stage('Deploy to Staging') {
            when { branch 'main' }
            steps {
                input message: 'Deploy to staging?', ok: 'Deploy'
                withKubeConfig([credentialsId: 'kubeconfig-staging']) {
                    sh "kubectl set image deployment/${APP_NAME} ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG} --namespace staging"
                }
            }
        }
    }

    post {
        always   { cleanWs() }
        success  { slackSend(color: 'good',    message: "SUCCESS: ${APP_NAME} #${BUILD_NUMBER}") }
        failure  { slackSend(color: 'danger',  message: "FAILED: ${APP_NAME} #${BUILD_NUMBER}") }
        unstable { slackSend(color: 'warning', message: "UNSTABLE: ${APP_NAME} #${BUILD_NUMBER}") }
    }
}
```

---

### 3.3 Jenkins Shared Libraries & Multibranch

**Shared libraries** let you reuse pipeline code across projects. Logic lives in `vars/buildSpringBoot.groovy`; each project's Jenkinsfile calls it:
```groovy
@Library('jenkins-shared-library@main') _
buildSpringBoot(javaVersion: '21', skipTests: false)
```

**Multibranch Pipeline** auto-discovers branches and creates a pipeline job per branch. Combine with `when { branch 'main' }` blocks for branch-specific deploy logic.

---

## 4. GitLab CI

### 4.1 .gitlab-ci.yml Structure

```yaml
stages: [build, test, package, deploy]

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
cache:
  paths: [.m2/]

build:
  stage: build
  image: maven:3.9-eclipse-temurin-21
  script: [mvn clean compile --batch-mode -q]

unit-tests:
  stage: test
  image: maven:3.9-eclipse-temurin-21
  script: [mvn test --batch-mode]
  artifacts:
    when: always
    reports:
      junit: target/surefire-reports/TEST-*.xml

docker-build:
  stage: package
  image: docker:24
  services: [docker:24-dind]
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE
  only: [main, tags]

deploy-production:
  stage: deploy
  image: bitnami/kubectl:latest
  environment: { name: production, url: https://myapp.com }
  script:
    - kubectl set image deployment/springboot-app springboot-app=$DOCKER_IMAGE --namespace production
  when: manual
  only: [main]
```

GitLab keeps config DRY with `include` (pull in other YAML files or built-in templates) and `extends` (inherit from a hidden base job). This is equivalent to reusable workflows / Jenkins shared libraries.

---

## 5. Branch Strategies

### 5.1 GitFlow

```
main (production-ready)
├── hotfix/fix-login-bug ──────────────────→ main + develop
develop (integration branch)
├── feature/user-auth ─────────────────────→ develop
└── release/1.2.0 ──────────────────────→ main (tag v1.2.0) + develop
```

- `main`: Production only — receives merges from `release/*` and `hotfix/*`
- `develop`: Integration branch; features merge here
- `feature/*`: One branch per feature, from `develop`
- `release/*`: From `develop` when ready; bug fixes only; merges to both `main` + `develop`
- `hotfix/*`: Emergency fix from `main`; merges to both `main` + `develop`

**When to use:** Versioned releases, multiple prod versions, longer QA cycles.
**Drawbacks:** Complex double-merge overhead; `main` may be weeks behind latest work; poor fit for CI/CD.

---

### 5.2 Trunk-Based Development (TBD)

Everyone integrates to `main` at least once per day. Feature branches live hours, not weeks. Incomplete features are hidden behind **feature flags**, not branches.

```java
@PostMapping("/payment")
public ResponseEntity<?> processPayment(@RequestBody PaymentRequest req) {
    if (featureFlags.isEnabled("new-payment-gateway", req.getUserId())) {
        return newPaymentGatewayService.process(req);   // New code, dark
    }
    return legacyPaymentService.process(req);           // Old code, live
}
```

**Why TBD is preferred for CI/CD:** Every commit to main triggers a full CI pipeline; conflicts are tiny; main is always deployable; feature flags allow zero-risk releases (flip the flag, no deployment needed).

---

### 5.3 GitHub Flow

Branch from `main` → push → open PR (triggers CI) → review → merge → auto-deploy. Feature branches can live days to weeks (unlike TBD's hours). Good for web apps and small teams.

---

### 5.4 Comparison Table

| | GitFlow | Trunk-Based Dev | GitHub Flow |
|---|---------|-----------------|-------------|
| **Branch complexity** | High (5 types) | Low | Low |
| **Integration frequency** | Weekly/per feature | Daily | Per PR |
| **Suitable for CI/CD** | Poor | Excellent | Good |
| **Feature flags needed** | No | Yes | Sometimes |
| **Multiple prod versions** | Yes | No | No |

---

## 6. Deployment Strategies

### 6.1 Recreate
Stop all v1 → start all v2. **Pros:** Simple, no version coexistence. **Cons:** Downtime. **Use:** Dev/test, batch jobs.

### 6.2 Rolling Deployment

Replace instances one at a time. Load balancer routes to healthy instances only.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1   # At most 1 pod down at a time
    maxSurge: 1         # At most 1 extra pod during update
```

Include `readinessProbe` on `/actuator/health/readiness` so pods only receive traffic when ready.

**Pros:** Zero downtime, no extra infra. **Cons:** Both versions coexist briefly (API must be backward-compatible). **Use:** Standard production, K8s default.

### 6.3 Blue-Green Deployment

Two identical environments. Deploy to idle (green), smoke test, flip load balancer from blue to green. Keep blue for instant rollback.

**Pros:** Instant rollback, no mixed versions. **Cons:** 2x infra cost, complex for stateful apps. **Use:** High-stakes releases, regulated environments.

### 6.4 Canary Deployment

Route a small % of real traffic (e.g. 5%) to the new version, monitor, then gradually increase: 5% → 20% → 50% → 100%. Roll back by routing 0% to canary.

**Pros:** Low blast radius, real-traffic testing, automatic rollback. **Cons:** Needs good observability, API backward compatibility. **Use:** High-traffic services, risky changes.

### 6.5 Feature Flags

Deploy code to production without releasing it. Toggle a flag to enable/disable a feature — no deployment needed.

```java
@Service
public class SearchService {
    @Autowired private Unleash unleash;

    public List<Product> search(String query, String userId) {
        if (unleash.isEnabled("elastic-search", new UnleashContext.Builder().userId(userId).build())) {
            return elasticSearchAdapter.search(query);   // New
        }
        return legacyDatabaseSearch(query);              // Old
    }
}
```

**Tools:** LaunchDarkly, Unleash (open source), Split.io. **Use cases:** Dark launches, A/B testing, kill switches, gradual rollouts by user segment.

### 6.6 Strategy Comparison

| Strategy | Downtime | Rollback Speed | Cost | Use Case |
|----------|----------|---------------|------|----------|
| **Recreate** | Yes | Fast | Low | Dev, batch jobs |
| **Rolling** | No | Slow | Low | Standard production |
| **Blue-Green** | No | Instant | 2x | Critical services |
| **Canary** | No | Fast | Low+ | High-traffic, risky changes |
| **Feature Flags** | No | Instant | Low | Continuous deployment |

---

## 7. Quality Gates in CI

### 7.1 Unit Tests & Coverage (JaCoCo)

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution><id>prepare-agent</id><goals><goal>prepare-agent</goal></goals></execution>
        <execution><id>report</id><phase>test</phase><goals><goal>report</goal></goals></execution>
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
                                <minimum>0.80</minimum>   <!-- 80% line coverage -->
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

`mvn verify` fails with a non-zero exit if any test fails or coverage drops below threshold.

---

### 7.2 SonarQube Quality Gate

Evaluates: coverage, code smells, bugs, vulnerabilities, duplications.

**Default "Sonar Way" gate:** Coverage on new code ≥ 80%, no new blocker issues, no new critical CVEs, duplication ≤ 3%.

In Jenkins/Actions: `waitForQualityGate` polls SonarQube and fails the pipeline if the gate fails.

---

### 7.3 SAST, DAST, Container Scanning

**SAST** (static — no running app):

| Tool | What It Checks |
|------|---------------|
| OWASP Dependency Check | Known CVEs in Maven deps |
| Snyk | Dependencies, IaC, containers |
| SpotBugs | Java bytecode bugs |
| SonarQube | Security hotspots in code |

**DAST** (dynamic — tests a running app by sending attack-like requests). Tool: OWASP ZAP against staging.

```yaml
- uses: zaproxy/action-baseline@v0.10.0
  with:
    target: 'https://staging.myapp.com'
    fail_action: warn
```

**Container scanning (Trivy):**
```bash
trivy image --severity CRITICAL,HIGH --exit-code 1 myrepo/springboot-app:abc1234
```

---

## 8. Secrets Management

### 8.1 Never Commit Secrets

Never commit passwords, API keys, DB connection strings, or tokens. Add to `.gitignore`:
```
.env
application-local.properties
*.pem
*.key
```
Enable GitHub's built-in secret scanning. Use `truffleHog` as a pre-commit hook.

### 8.2 GitHub Secrets

```yaml
- name: Use secret
  env:
    API_KEY: ${{ secrets.THIRD_PARTY_API_KEY }}
  run: some-tool --key "$API_KEY"
```

**Types:** Repository (one repo), Environment (scoped to production env), Organization (shared across repos).

### 8.3 HashiCorp Vault & Kubernetes Secrets

**Vault** (enterprise-scale): stores secrets externally; issues dynamic/short-lived DB credentials; full audit logging and automatic rotation.

**Kubernetes Secrets** are only base64-encoded, **not** encrypted — don't commit real secrets to Git. Use the **External Secrets Operator** to sync from Vault/AWS Secrets Manager into K8s Secrets at runtime.

---

## 9. Docker in CI/CD

### 9.1 Multi-Stage Build

```dockerfile
# Stage 1: Dependency cache layer
FROM maven:3.9-eclipse-temurin-21 AS deps
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -q    # Cached unless pom.xml changes

# Stage 2: Build
FROM deps AS builder
COPY src ./src
RUN mvn clean package -DskipTests -q

# Stage 3: Minimal runtime
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=builder /app/target/*.jar app.jar
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

Key points: copy `pom.xml` before `src/` so the dependency layer is cached; use JRE (not JDK) in the runtime image; run as a non-root user.

### 9.2 Image Tagging Strategy

Avoid `latest` (mutable). Use Git SHA for traceability:
```bash
IMAGE_TAG=$(git rev-parse --short HEAD)
docker build -t myapp:${IMAGE_TAG} .
```

---

## 10. Deployment to Kubernetes

### 10.1 kubectl in CI/CD

```bash
# Update image
kubectl set image deployment/springboot-app \
  springboot-app=myrepo/springboot-app:abc1234 --namespace production

# Wait for rollout
kubectl rollout status deployment/springboot-app --namespace production --timeout=300s

# Rollback
kubectl rollout undo deployment/springboot-app --namespace production
```

**Key deployment manifest fields:**
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    spec:
      containers:
        - name: springboot-app
          image: myrepo/springboot-app:abc1234
          resources:
            requests: { memory: "256Mi", cpu: "250m" }
            limits:   { memory: "512Mi", cpu: "500m" }
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            initialDelaySeconds: 20
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            initialDelaySeconds: 30
```

### 10.2 Helm Chart Deployment

```bash
helm upgrade --install springboot-app ./charts/springboot-app \
  --namespace production \
  --values ./charts/springboot-app/values-production.yaml \
  --set image.tag=${IMAGE_TAG} \
  --wait \
  --timeout 5m \
  --atomic    # Auto-rollback on failure
```

### 10.3 ArgoCD (GitOps)

ArgoCD watches a Git repo of manifests. When the manifest changes, it pulls and applies to Kubernetes — the pipeline needs no cluster credentials.

```
CI: build image → push to ECR → commit new image tag to manifest repo
ArgoCD: detects Git change → pulls manifest → applies to K8s cluster
```

Key benefit: cluster pulls changes (more secure); drift is auto-corrected.

---

## 11. Environment Promotion

### 11.1 Build Once, Deploy Everywhere

The same Docker image SHA (`myapp:abc1234`) deploys to dev → staging → production. Only config differs per environment:
- Non-sensitive config: Spring profiles, Kubernetes ConfigMaps
- Credentials: Kubernetes Secrets / External Secrets Operator
- Infra settings (replicas, resources): Helm values files (`values-production.yaml`)

The Docker image holds no environment-specific config — all differences are injected at runtime via `SPRING_PROFILES_ACTIVE` and Kubernetes manifests.

---

## 12. Monitoring Post-Deployment

### 12.1 Smoke Tests

```bash
#!/bin/bash
APP_URL=${1:-"https://myapp.com"}
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$APP_URL/actuator/health")
[ "$HTTP_STATUS" = "200" ] || { echo "FAIL: health returned $HTTP_STATUS"; exit 1; }
echo "Smoke tests passed."
```

### 12.2 Automatic Rollback

```bash
if ! ./smoke-test.sh; then
  kubectl rollout undo deployment/springboot-app --namespace production
  exit 1
fi
```

Advanced setups (Argo Rollouts) query Prometheus metrics and auto-roll-back when success rate drops below threshold.

---

## 13. Interview Questions & Answers

### Q1: What is the difference between CI, Continuous Delivery, and Continuous Deployment?

**A:** CI means every developer integrates to the shared branch multiple times per day; each integration triggers automated build and test. Continuous Delivery extends CI so every successful build produces an artifact ready to deploy with one manual decision. Continuous Deployment removes even that gate — every passing commit deploys automatically to production.

---

### Q2: Walk me through a CI/CD pipeline you have set up.

**A:** We used GitHub Actions with three jobs. Job 1 ran on every push/PR: set up JDK 21, ran `mvn verify` (tests + 80% JaCoCo coverage), and ran SonarQube — failing the pipeline if the quality gate didn't pass. Job 2 ran on merge to main: built a multi-stage Docker image and pushed to AWS ECR using OIDC (no long-lived keys), tagged with the git SHA. Job 3 deployed to EKS via `kubectl set image` + rollout wait, gated behind a GitHub environment requiring reviewer approval; a smoke test on `/actuator/health` auto-triggered `kubectl rollout undo` on failure.

---

### Q3: How do you handle database migrations in a CI/CD pipeline?

**A:** Use Flyway/Liquibase with forward-compatible migrations. The rule: never make a breaking schema change in the same deployment as the code that requires it. Step 1 — deploy migration only (add a nullable column; old code ignores it). Step 2 — deploy app code that uses the new column. Step 3 — cleanup migration (make NOT NULL, drop old columns). Spring Boot applies pending Flyway migrations at startup; if migration fails, startup fails and the deployment fails, triggering rollback.

---

### Q4: What is the difference between blue-green and canary deployments?

**A:** Blue-green maintains two identical environments; you deploy to idle (green), verify, then flip the load balancer 100% from blue to green — instant rollback by flipping back. Cost is ~2x infra. Canary routes a small % (e.g. 5%) of real traffic to the new version, monitors metrics, then gradually increases to 100%; rollback by routing 0% to canary. Blue-green is better when you need instant rollback; canary is better for high-traffic services where you want real-traffic validation with a limited blast radius.

---

### Q5: How do you manage secrets in a CI/CD pipeline?

**A:** Secrets never live in code or git. In GitHub Actions we use GitHub Secrets (repo/environment-scoped) and OIDC for AWS (no static keys). At runtime, apps read from AWS Secrets Manager or HashiCorp Vault via the External Secrets Operator, which syncs into Kubernetes Secrets without touching git. We enable GitHub's secret scanning and use pre-commit hooks with `truffleHog`. Production secrets are only accessible to jobs referencing the "production" environment, which requires reviewer approval.

---

### Q6: What is GitOps?

**A:** GitOps declares desired infrastructure/app state in Git; an agent (ArgoCD or Flux) continuously reconciles the live cluster to match it. You change things via PRs, not manual `kubectl apply`. The pipeline builds/pushes the image, then commits the new image tag to a manifests repo; ArgoCD detects the change and applies it. The pipeline needs no cluster credentials, and drift is auto-corrected.

---

### Q7: What is trunk-based development and why is it preferred for CI/CD?

**A:** All developers integrate to `main` at least once per day; feature branches are short-lived (hours, not weeks); incomplete features are hidden behind feature flags. It's preferred for CI/CD because every commit triggers a full pipeline and `main` is always near-deployable — unlike long-lived branches where integration problems accumulate. Feature flags let you deploy unfinished code safely; enabling a feature requires just a flag flip, no deployment.

---

### Q8: How do you prevent a bad deploy from reaching production?

**A:** Layered defence: automated tests + coverage gates in CI; SonarQube quality gate; container scanning (Trivy fails on CRITICAL/HIGH); branch protection requiring passing checks + review; deploy to staging first; environment approval gate for production; rolling/canary deployment to limit blast radius; post-deploy smoke test with auto-rollback on failure; Prometheus/Grafana alerting on error rate spikes.

---

### Q9: What happens when a pipeline fails midway?

**A:** The pipeline stops at the failing stage — subsequent stages don't run. The failed commit is not deployed. If tests fail, code never reaches Docker build; if rollout fails, Kubernetes stops the rolling update, leaving old pods serving traffic. The developer fixes the issue, pushes a new commit, and the pipeline restarts from the beginning. `post { failure {} }` blocks in Jenkins / `if: always()` steps in Actions handle notifications and cleanup even on failure.

---

### Q10: How do you implement zero-downtime deployments?

**A:** Rolling update with `maxUnavailable: 0` — pods are only terminated after new ones pass readiness checks. Spring Boot exposes `/actuator/health/readiness`; Kubernetes only routes traffic to pods that pass it. Configure graceful shutdown: `server.shutdown=graceful` and `spring.lifecycle.timeout-per-shutdown-phase=30s` so in-flight requests complete. APIs must be backward-compatible since v1 and v2 pods coexist during the rollout, and database migrations must be forward-compatible for the same reason.

---

### Q11: What is SAST vs DAST?

**A:** SAST (Static) analyzes source/bytecode without running the app — finds insecure API usage, vulnerable libraries, hardcoded credentials. Runs early and fast. Tools: OWASP Dependency Check, Snyk, SpotBugs, SonarQube. DAST (Dynamic) tests a running app by sending attack-like HTTP requests — catches runtime issues SAST can't. Runs against staging. Tools: OWASP ZAP, Burp Suite. Best practice: use both.

---

### Q12: How do you test a Docker image in CI?

**A:** Build verification (`docker build` success); Trivy CVE scan; smoke test against a running container; integration tests with Docker Compose (app + Postgres/Redis). Smoke test example:
```bash
docker run -d --name test-app -p 8080:8080 myapp:abc1234
sleep 10
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/actuator/health)
docker stop test-app && [ "$HTTP_STATUS" = "200" ] || exit 1
```

---

### Q13: How do you handle a hotfix when main has changes you don't want to release?

**A:** GitFlow scenario: branch `hotfix/fix` directly from the production tag on `main`, apply the minimal fix, merge back to `main` (new tag), and also merge to `develop`. In trunk-based development this is less common — either use a feature flag to hide incomplete work (making `main` safe to deploy), or cherry-pick just the hotfix commit to a release branch.

---

### Q14: What is the difference between `needs` and `if` in GitHub Actions?

**A:** `needs` controls execution order — a job won't start until listed jobs complete successfully; if a dependency fails, the job is skipped. `if` controls whether a job/step runs at all based on a boolean expression. They work together:
```yaml
deploy:
  needs: build                             # Run after build succeeds
  if: github.event_name == 'push'         # Only on direct pushes, not PRs
```

---

### Q15: What is the "artifact promotion" pattern?

**A:** Build the Docker image once in CI, push with the git SHA tag, then deploy that exact same image through dev → staging → production. Never rebuild for each environment. This guarantees the binary tested in CI is the binary reaching production. Per-environment differences (replica counts, resource limits, credentials) are handled via Helm values files and Kubernetes Secrets.

---

### Q16: Explain "shift-left" in CI/CD.

**A:** Moving security and quality checks earlier in the development timeline. A bug caught in unit tests takes minutes to fix; a production security vulnerability can take days and trigger incident response. Practices: SonarLint in the IDE, pre-commit hooks (Checkstyle, SpotBugs), SAST in CI, code review gates on PRs, dependency scanning early in the build stage.

---

### Q17: What is a self-healing pipeline?

**A:** A pipeline that detects and recovers from deployment failures without human intervention. Pieces: readiness probes keep traffic off bad pods; `kubectl rollout status --timeout` fails the pipeline if pods don't become ready; the pipeline catches the failure and runs `kubectl rollout undo`; Argo Rollouts auto-rolls-back on metric degradation. The system returns to the last known-good state automatically.

---

### Q18: What is the difference between Docker layer caching and GitHub Actions caching?

**A:** Docker layer cache reuses a Dockerfile layer when its inputs haven't changed — so copying `pom.xml` and running `dependency:go-offline` before copying `src/` means the dependency layer is cached unless `pom.xml` changes. GitHub Actions cache (`actions/cache`) stores arbitrary files (like `~/.m2`) between workflow runs, keyed by a hash of `pom.xml`, saving ~3 minutes of dependency downloads.

---

### Q19: How do you roll back a Helm release?

**A:** Helm maintains a release history. `helm rollback springboot-app --namespace production` reverts to the previous revision. Use `--atomic` on `helm upgrade` so failures auto-rollback without manual intervention:
```bash
helm upgrade --install springboot-app ./chart --namespace production --atomic --timeout 5m
```

---

### Q20: What is the purpose of `readinessProbe` vs `livenessProbe`?

**A:** `readinessProbe` — "Is this pod ready to receive traffic?" Kubernetes only routes to pods passing it. Prevents traffic to initializing Spring Boot pods (startup takes 10-20s) and enables graceful shutdown. `livenessProbe` — "Is this pod alive?" Kubernetes restarts the pod on failure. Detects deadlocks where the process is running but stuck. Map readiness to `/actuator/health/readiness` and liveness to `/actuator/health/liveness` (separate endpoints since Spring Boot 2.3).

---

### Q21: How do you manage environment-specific configuration?

**A:** The Docker image holds no environment-specific config — all differences inject at runtime. Spring profiles (`SPRING_PROFILES_ACTIVE=production`) for non-sensitive config; Kubernetes ConfigMaps for per-environment values; Kubernetes Secrets/External Secrets Operator for credentials; Helm values files (`values-production.yaml`) for replica counts and resource limits.

---

### Q22: What is a "flaky test" and how do you handle it?

**A:** A test that passes sometimes and fails other times without code changes. Common causes: time-dependent assertions, race conditions, external service dependencies, port conflicts. Fix it: identify (intermittent CI failures), quarantine with `@Disabled` (stops noise while you investigate), then fix the root cause — mock the external dependency, fix the race condition, use `Awaitility` for async assertions instead of `Thread.sleep`. Retry (`<rerunFailingTestsCount>2</rerunFailingTestsCount>`) is a last resort — it hides real problems.

---

### Q23: What are branch protection rules?

**A:** GitHub settings that prevent direct pushes to `main` and enforce quality gates before merging. Typical rules: require PR (no direct commits), require passing status checks (CI must pass), require review approvals, dismiss stale reviews on new pushes, require branch to be up to date. Without them, untested code can reach main; with them, the CI pipeline is the mandatory gate.

---

## 14. Quick Reference Cheat Sheet

### GitHub Actions Key Syntax

```yaml
on: push / pull_request / schedule / workflow_dispatch / workflow_call
runs-on: ubuntu-latest / windows-latest / self-hosted
needs: [job1, job2]                         # Job dependency
if: github.ref == 'refs/heads/main'         # Conditional
environment: production                      # Requires approval
${{ secrets.MY_SECRET }}                    # Secret reference
${{ github.sha }}                           # Commit SHA
${{ github.ref_name }}                      # Branch name
${{ matrix.java }}                          # Matrix value
echo "key=value" >> $GITHUB_OUTPUT          # Set step output
echo "VAR=value" >> $GITHUB_ENV             # Set env for subsequent steps
```

### Key Maven Commands in CI

```bash
mvn clean verify                # Compile + test + coverage check
mvn clean package -DskipTests   # Build JAR, skip tests
mvn sonar:sonar                 # SonarQube analysis
mvn dependency-check:check      # OWASP dependency scan
mvn --batch-mode                # Non-interactive (CI-friendly)
mvn --no-transfer-progress      # Suppress download progress
```

### kubectl Commands in CI

```bash
kubectl set image deployment/app container=image:tag --namespace ns
kubectl rollout status deployment/app --namespace ns --timeout=300s
kubectl rollout undo deployment/app --namespace ns
kubectl apply -f manifests/ --namespace ns
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

---

*Last Updated: 2026-06-18*
