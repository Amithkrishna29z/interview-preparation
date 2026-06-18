# Git, Docker & DevOps Interview Questions

> These are commonly asked in junior developer interviews. Understand the concepts and commands.

---

## Table of Contents
1. [Git Basics](#git-basics)
2. [Git Branching & Merging](#git-branching--merging)
3. [Git Advanced Concepts](#git-advanced-concepts)
4. [Docker Basics](#docker-basics)
5. [Docker Commands](#docker-commands)
6. [Maven & Build Tools](#maven--build-tools)
7. [CI/CD Basics](#cicd-basics)
8. [Quick Revision Summary](#quick-revision-summary)

---

## Git Basics

### Q1: What is Git and why is it used?

Git is a distributed version control system that tracks every change made to code. Every developer has a full copy of the project history, enabling branching and team collaboration.

```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
git --version
```

---

### Q2: What is the difference between Git and GitHub?

| Git | GitHub |
|-----|--------|
| Version control tool on your computer | Website that hosts Git repositories |
| Works locally (offline) | Requires internet |
| Command-line tool | Web interface |

**Analogy:** Git = Word's "Track Changes"; GitHub = Google Drive for code.

---

### Q3: What are the three stages of Git?

1. **Working Directory** — where you edit files
2. **Staging Area** — files ready to commit (`git add`)
3. **Repository** — permanently saved (`git commit`)

```bash
git add filename.java       # stage specific file
git add .                   # stage all changes
git commit -m "Add login feature"
git status                  # check current state
```

```
Working Directory → (git add) → Staging Area → (git commit) → Repository
```

---

### Q4: What are the most common Git commands?

```bash
# SETUP
git init                          # Start new local repository
git clone <url>                   # Download remote repository

# BASIC WORKFLOW
git status                        # Show changed files
git add .                         # Stage all changes
git commit -m "message"           # Save changes
git push origin main              # Upload to GitHub
git pull origin main              # Download latest from GitHub

# VIEWING HISTORY
git log --oneline                 # Short commit history
git diff                          # Show unstaged changes

# UNDOING CHANGES
git restore filename.java         # Discard working dir changes
git restore --staged filename.java # Unstage a file
git reset HEAD~1                  # Undo last commit (keep changes)
git revert <commit-id>            # New commit that undoes a past commit
```

---

### Q5: What is the difference between `git pull` and `git fetch`?

- `git fetch` — downloads remote changes but does NOT merge
- `git pull` — downloads AND merges (equivalent to fetch + merge)

Use `fetch` to inspect changes first; use `pull` when you're ready to update immediately.

---

## Git Branching & Merging

### Q6: What is a Git branch and why is it used?

A branch is a parallel copy of the codebase. You develop a feature in isolation without touching the main working code.

```bash
git branch feature-login          # Create branch
git checkout -b feature-login     # Create AND switch
git switch feature-login          # Switch (Git 2.23+)

git checkout main
git merge feature-login           # Merge branch into main
git branch -d feature-login       # Delete merged branch
```

```
main:    A --- B --- C
                      \
feature:               D --- E --- F
                                    \
merged:  A --- B --- C ------------ G
```

---

### Q7: What is the difference between `git merge` and `git rebase`?

- **Merge** — combines branches and creates a merge commit; preserves full history
- **Rebase** — replays your commits on top of another branch; produces linear history

```bash
# Merge
git checkout main && git merge feature-branch

# Rebase
git checkout feature-branch && git rebase main
```

Use `merge` for shared/public branches. Use `rebase` for local branches only.

> Never rebase public branches that others are working on.

---

### Q8: What is a merge conflict and how do you resolve it?

A merge conflict occurs when two branches changed the same line differently.

```bash
git merge feature-branch   # CONFLICT in UserService.java

# Git marks the file:
# <<<<<<< HEAD
# String name = "Alice";
# =======
# String name = "Bob";
# >>>>>>> feature-branch

# Manually pick the correct version, then:
git add UserService.java
git commit -m "Resolve merge conflict"
```

---

## Git Advanced Concepts

### Q9: What is `git stash`?

Stash temporarily hides uncommitted work so you can switch branches cleanly.

```bash
git stash                          # Save current changes
git stash save "WIP: login feature"
git stash list                     # View all stashes
git stash pop                      # Apply latest and delete it
git stash apply stash@{1}          # Apply specific stash (keep it)
git stash clear                    # Delete all stashes
```

**Scenario:** Mid-feature, urgent bug reported → `git stash` → fix bug → `git stash pop`.

---

### Q10: What is the difference between `git reset` and `git revert`?

| | `git reset` | `git revert` |
|--|-------------|--------------|
| What it does | Moves HEAD back | Creates a new undo commit |
| History | Rewrites history | Preserves history |
| Safe for shared code? | No | Yes |
| Use case | Local fixes before push | Undoing already-pushed commits |

```bash
git reset --soft HEAD~1   # Undo commit, keep changes staged
git reset --hard HEAD~1   # Undo commit AND discard changes
git revert <commit-hash>  # Safe undo for public repos
```

---

## Docker Basics

### Q11: What is Docker and why is it used?

Docker packages your app and all its dependencies (Java, libraries, config) into a container so it runs identically everywhere — solving the "works on my machine" problem.

| Term | Explanation |
|------|-------------|
| **Image** | Blueprint/template (like a recipe) |
| **Container** | Running instance of an image |
| **Dockerfile** | Instructions to build an image |
| **Docker Hub** | Public registry for images |
| **Volume** | Persistent storage for containers |

---

### Q12: What is a Dockerfile?

A Dockerfile contains step-by-step instructions to build a Docker image.

```dockerfile
# Basic Spring Boot Dockerfile
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY target/myapp-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Multi-stage build** (builds inside Docker, smaller final image):
```dockerfile
# Stage 1: Build
FROM maven:3.9-openjdk-17 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Run
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## Docker Commands

### Q13: What are the essential Docker commands?

```bash
# IMAGE COMMANDS
docker images                       # List all images
docker pull openjdk:17              # Download from Docker Hub
docker build -t myapp:1.0 .         # Build from Dockerfile
docker rmi myapp:1.0                # Remove image

# CONTAINER COMMANDS
docker run -d -p 8080:8080 myapp:1.0          # Run in background, map port
docker run --name mycontainer myapp:1.0       # Named container
docker ps                           # List running containers
docker ps -a                        # All containers (including stopped)
docker stop mycontainer             # Stop container
docker rm mycontainer               # Remove container
docker logs mycontainer             # View logs
docker exec -it mycontainer bash    # Terminal inside container

# DOCKER COMPOSE
docker-compose up -d                # Start all services in background
docker-compose down                 # Stop and remove containers
docker-compose logs                 # View logs
```

---

### Q14: What is Docker Compose?

Docker Compose runs multiple containers together with one command — ideal for apps that need a backend + database.

```yaml
# docker-compose.yml — Spring Boot + MySQL
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://db:3306/mydb
      - SPRING_DATASOURCE_USERNAME=root
      - SPRING_DATASOURCE_PASSWORD=password
    depends_on:
      - db
    networks:
      - app-network

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: mydb
    volumes:
      - db-data:/var/lib/mysql
    networks:
      - app-network

volumes:
  db-data:

networks:
  app-network:
```

---

## Maven & Build Tools

### Q15: What is Maven and what is `pom.xml`?

Maven manages your Java project's dependencies and build lifecycle. You declare what libraries you need in `pom.xml` and Maven downloads them automatically.

```xml
<!-- pom.xml — Spring Boot project -->
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>myapp</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
    </dependencies>
</project>
```

**Common Maven Commands:**
```bash
mvn clean package      # Clean + build JAR (most used)
mvn test               # Run tests
mvn spring-boot:run    # Run Spring Boot app locally
mvn install            # Install to local repository
```

---

## CI/CD Basics

### Q16: What is CI/CD?

- **CI (Continuous Integration)** — automatically build and test code on every push
- **CD (Continuous Delivery/Deployment)** — automatically deploy after tests pass

```
Push code → GitHub → GitHub Actions (CI)
                       ↓
              Run Tests → Build → Docker Image
                       ↓
              Deploy to Server (CD) → App is Live
```

**GitHub Actions example:**
```yaml
# .github/workflows/main.yml
name: Build and Test
on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'
    - run: mvn clean package
    - run: mvn test
```

---

## Quick Revision Summary

### Git Commands You Must Know

| Command | Purpose |
|---------|---------|
| `git clone <url>` | Download a repository |
| `git add .` | Stage all changes |
| `git commit -m "msg"` | Save changes |
| `git push origin main` | Upload to GitHub |
| `git pull origin main` | Download latest changes |
| `git checkout -b feature` | Create and switch branch |
| `git merge feature` | Merge branch |
| `git stash` | Temporarily hide changes |
| `git log --oneline` | View commit history |
| `git status` | See what changed |

### Docker Commands You Must Know

| Command | Purpose |
|---------|---------|
| `docker build -t app .` | Build image from Dockerfile |
| `docker run -p 8080:8080 app` | Run container with port mapping |
| `docker ps` | List running containers |
| `docker logs container` | View logs |
| `docker-compose up -d` | Start all services |
| `docker-compose down` | Stop all services |

### Interview Tips

1. **Git flow**: feature branch → PR/MR → review → merge to main
2. **Commit messages**: "Add user login endpoint" not "fix stuff"
3. **Never commit secrets** (passwords, API keys) to Git
4. **Docker**: containers are ephemeral — data is lost on stop unless using volumes
5. **CI/CD**: be ready to draw the pipeline on a whiteboard
