# Git, Docker & DevOps Interview Questions

> 🎯 These are commonly asked in junior developer interviews. Understand the concepts and commands.

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

**Easy Explanation:** Git is a tool that keeps track of changes in your code. Think of it like "undo history" for your project that you can share with teammates.

**Key Points:**
- **Version Control System (VCS)** — tracks every change made to code
- **Distributed** — every developer has a full copy of the project history
- **Branching** — work on new features without breaking the main code
- **Collaboration** — multiple developers can work on the same project

```bash
# Check Git version
git --version

# Set your name and email (one-time setup)
git config --global user.name "Your Name"
git config --global user.email "your@email.com"

# Check configuration
git config --list
```

---

### Q2: What is the difference between Git and GitHub?

| Git | GitHub |
|-----|--------|
| A version control tool installed on your computer | A website (cloud) that hosts Git repositories |
| Works locally (offline) | Requires internet |
| Command-line tool | Has a web interface |
| Free and open source | Free (with paid plans) |
| Made by Linus Torvalds | Made by GitHub Inc. (owned by Microsoft) |

**Simple Analogy:**
- **Git** = the version control software (like Microsoft Word's "Track Changes")
- **GitHub** = where you store and share your code online (like Google Drive for code)

---

### Q3: What are the three stages of Git?

**Easy Explanation:** Think of it like preparing food:
1. **Working Directory** = Your kitchen (where you make changes)
2. **Staging Area (Index)** = Plate (ready to serve, not yet served)
3. **Repository** = Menu/Record (permanently saved)

```bash
# 1. Working Directory — you edit files here
# Make changes to files...

# 2. Stage files (add to staging area)
git add filename.java       # stage specific file
git add .                   # stage ALL changed files

# 3. Commit to Repository (save permanently)
git commit -m "Add user login feature"

# Check current status
git status
```

```
Working Directory → (git add) → Staging Area → (git commit) → Repository
```

---

### Q4: What are the most common Git commands?

```bash
# === SETUP ===
git init                          # Start new local repository
git clone <url>                   # Download remote repository

# === BASIC WORKFLOW ===
git status                        # Show changed files
git add .                         # Stage all changes
git commit -m "message"           # Save changes with message
git push origin main              # Upload to GitHub
git pull origin main              # Download latest from GitHub

# === VIEWING HISTORY ===
git log                           # Show full commit history
git log --oneline                 # Show short commit history
git diff                          # Show unstaged changes
git diff --staged                 # Show staged changes

# === UNDOING CHANGES ===
git restore filename.java         # Discard changes in working dir
git restore --staged filename.java # Unstage a file
git reset HEAD~1                  # Undo last commit (keep changes)
git revert <commit-id>            # Create new commit that undoes changes
```

---

### Q5: What is the difference between `git pull` and `git fetch`?

**Easy Explanation:**
- `git fetch` = Check what's new at the store (look but don't buy)
- `git pull` = Check AND bring the items home (fetch + merge)

```bash
# git fetch — downloads but does NOT merge
git fetch origin
git log origin/main  # see what's new
git merge origin/main  # manually merge

# git pull — downloads AND automatically merges
git pull origin main
# Equivalent to: git fetch + git merge
```

**When to use each:**
- Use `git fetch` when you want to see changes before applying
- Use `git pull` when you trust the remote and want to update immediately

---

## Git Branching & Merging

### Q6: What is a Git branch and why is it used?

**Easy Explanation:** A branch is like a parallel universe for your code. You can experiment without affecting the main working code.

```bash
# === BRANCH COMMANDS ===
git branch                        # List all branches
git branch feature-login          # Create new branch
git checkout feature-login        # Switch to branch
git checkout -b feature-login     # Create AND switch (shortcut)
git switch feature-login          # New way to switch branch (Git 2.23+)

# After making changes on branch:
git checkout main                 # Switch back to main
git merge feature-login           # Merge branch into main

# Delete branch
git branch -d feature-login       # Delete (safe — only if merged)
git branch -D feature-login       # Force delete
```

**Standard Workflow:**
```
main branch:     A --- B --- C
                              \
feature branch:               D --- E --- F
                                           \
after merge:     A --- B --- C ----------- G (merge commit)
```

---

### Q7: What is the difference between `git merge` and `git rebase`?

**Easy Explanation:**
- **Merge** = Combine two branches, keeping both histories. Creates a "merge commit."
- **Rebase** = Move your changes on top of another branch. Creates cleaner, linear history.

```bash
# Merge (keeps all history)
git checkout main
git merge feature-branch
# Result: Creates a merge commit, history looks like a tree

# Rebase (cleaner history)
git checkout feature-branch
git rebase main
# Result: Moves feature commits on top of main, linear history
```

**When to use:**
- `merge` — for public/shared branches (safe, preserves history)
- `rebase` — for local/private branches (cleaner history)

> ⚠️ **Never rebase public branches** that others are working on!

---

### Q8: What is a merge conflict and how do you resolve it?

**Easy Explanation:** A merge conflict happens when two people changed the same line of code differently.

```bash
# Conflict happens during merge:
git merge feature-branch
# ERROR: CONFLICT in UserService.java

# Open the conflicted file — Git marks it like this:
# <<<<<<< HEAD (your current branch)
# String name = "Alice";
# =======
# String name = "Bob";
# >>>>>>> feature-branch

# You manually choose which to keep:
# String name = "Alice";  (or "Bob", or combine them)

# After resolving:
git add UserService.java
git commit -m "Resolve merge conflict"
```

---

## Git Advanced Concepts

### Q9: What is `git stash`?

**Easy Explanation:** Stash = temporarily hide your unfinished work so you can switch branches.

```bash
# Save current uncommitted changes temporarily
git stash
git stash save "Work in progress: login feature"

# List all stashes
git stash list
# stash@{0}: WIP on feature: Work in progress: login feature
# stash@{1}: WIP on main: Half-done bug fix

# Restore stashed changes
git stash pop           # Apply latest stash and delete it
git stash apply stash@{1}  # Apply specific stash (keep it)
git stash drop stash@{0}   # Delete specific stash
git stash clear         # Delete all stashes
```

**Real-world scenario:**
```
You're coding Feature A... boss says "Fix urgent bug NOW!"
→ git stash (hide Feature A work)
→ fix bug, commit
→ git stash pop (continue Feature A)
```

---

### Q10: What is the difference between `git reset` and `git revert`?

| | `git reset` | `git revert` |
|--|-------------|--------------|
| What it does | Moves HEAD back to a previous commit | Creates a new commit that undoes changes |
| History | Rewrites history | Preserves history |
| Safe for shared code? | ❌ No — dangerous if pushed | ✅ Yes — safe |
| Use case | Local fixes before push | Undoing changes already pushed |

```bash
# git reset — rewrite history (local only)
git reset --soft HEAD~1   # Undo commit, keep changes staged
git reset --mixed HEAD~1  # Undo commit, keep changes in working dir
git reset --hard HEAD~1   # Undo commit AND discard changes ⚠️

# git revert — safe undo for public repos
git revert <commit-hash>  # Creates a new "undo" commit
```

---

## Docker Basics

### Q11: What is Docker and why is it used?

**Easy Explanation:** Docker is like a shipping container for software. It packages your app + everything it needs (Java, libraries, configs) so it runs the same way everywhere.

**Problem Docker solves:** "It works on my machine!" 😅

```
Without Docker:
Developer's PC → Works ✅
Test Server   → Fails ❌ (different Java version)
Production    → Fails ❌ (missing library)

With Docker:
Docker Container → Runs exactly the same everywhere ✅✅✅
```

**Key Concepts:**

| Term | Explanation | Real-world Analogy |
|------|-------------|-------------------|
| **Image** | Blueprint/template for a container | Recipe |
| **Container** | Running instance of an image | Cooked dish from recipe |
| **Dockerfile** | Instructions to build an image | Recipe card |
| **Docker Hub** | Public registry for images | App Store for Docker images |
| **Volume** | Persistent storage for containers | External hard drive |
| **Network** | Communication between containers | LAN network |

---

### Q12: What is a Dockerfile?

**Easy Explanation:** A Dockerfile is a text file with step-by-step instructions to build your Docker image.

```dockerfile
# Example Dockerfile for a Spring Boot app

# Step 1: Start from official Java image
FROM openjdk:17-jdk-slim

# Step 2: Set working directory inside container
WORKDIR /app

# Step 3: Copy the built JAR file into container
COPY target/myapp-0.0.1-SNAPSHOT.jar app.jar

# Step 4: Expose port 8080
EXPOSE 8080

# Step 5: Command to run when container starts
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Multi-stage build (optimized — builds inside Docker):**
```dockerfile
# Stage 1: Build
FROM maven:3.9-openjdk-17 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Run (smaller final image)
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
# === IMAGE COMMANDS ===
docker images                       # List all images
docker pull openjdk:17              # Download image from Docker Hub
docker build -t myapp:1.0 .         # Build image from Dockerfile
docker rmi myapp:1.0                # Remove image
docker image prune                  # Remove unused images

# === CONTAINER COMMANDS ===
docker run myapp:1.0                # Create and start container
docker run -d myapp:1.0             # Run in background (detached)
docker run -p 8080:8080 myapp:1.0   # Map port (host:container)
docker run --name mycontainer myapp:1.0   # Give container a name

docker ps                           # List running containers
docker ps -a                        # List all containers (including stopped)
docker stop mycontainer             # Stop container
docker start mycontainer            # Start stopped container
docker rm mycontainer               # Remove container
docker logs mycontainer             # View container logs
docker exec -it mycontainer bash    # Open terminal inside container

# === DOCKER COMPOSE ===
docker-compose up                   # Start all services
docker-compose up -d                # Start in background
docker-compose down                 # Stop and remove containers
docker-compose logs                 # View logs
```

---

### Q14: What is Docker Compose?

**Easy Explanation:** Docker Compose lets you run multiple containers together with one command. Perfect for apps that need a database + backend + frontend.

```yaml
# docker-compose.yml — Spring Boot + MySQL example

version: '3.8'
services:
  # Spring Boot App
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://db:3306/mydb
      - SPRING_DATASOURCE_USERNAME=root
      - SPRING_DATASOURCE_PASSWORD=password
    depends_on:
      - db           # Wait for database to start first
    networks:
      - app-network

  # MySQL Database
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: mydb
    ports:
      - "3306:3306"
    volumes:
      - db-data:/var/lib/mysql   # Persist database data
    networks:
      - app-network

volumes:
  db-data:    # Named volume for persistence

networks:
  app-network:  # Custom network for communication
```

```bash
# Start everything
docker-compose up -d

# Now your app runs on http://localhost:8080
# And connects to MySQL automatically!
```

---

## Maven & Build Tools

### Q15: What is Maven and what is `pom.xml`?

**Easy Explanation:** Maven is like a shopping assistant for your Java project. You tell it what libraries you need (in `pom.xml`) and it downloads them automatically.

```xml
<!-- pom.xml — Spring Boot project example -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <!-- Project Identity -->
    <groupId>com.example</groupId>
    <artifactId>myapp</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>My App</name>

    <properties>
        <java.version>17</java.version>
    </properties>

    <!-- Dependencies (libraries your app needs) -->
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
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

**Common Maven Commands:**
```bash
mvn clean              # Delete target/ folder
mvn compile            # Compile source code
mvn test               # Run tests
mvn package            # Create JAR/WAR file
mvn install            # Install to local repository
mvn spring-boot:run    # Run Spring Boot app
mvn clean package      # Clean + build (most used)
```

---

## CI/CD Basics

### Q16: What is CI/CD?

**Easy Explanation:**
- **CI (Continuous Integration)** = Automatically test your code whenever you push to Git
- **CD (Continuous Delivery/Deployment)** = Automatically deploy your app after tests pass

```
Developer pushes code → GitHub
                         ↓
         GitHub Actions / Jenkins (CI)
                         ↓
          Run Tests → Build App → Create Docker Image
                         ↓
         Deploy to Server (CD) → App is Live! 🚀
```

**Simple GitHub Actions example:**
```yaml
# .github/workflows/main.yml

name: Build and Test

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Set up Java 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'

    - name: Build with Maven
      run: mvn clean package

    - name: Run tests
      run: mvn test
```

---

## Quick Revision Summary

### 🔑 Git Commands You MUST Know

| Command | Purpose |
|---------|---------|
| `git clone <url>` | Download a repository |
| `git add .` | Stage all changes |
| `git commit -m "msg"` | Save changes |
| `git push origin main` | Upload to GitHub |
| `git pull origin main` | Download latest changes |
| `git branch feature` | Create a branch |
| `git checkout feature` | Switch to branch |
| `git merge feature` | Merge branch |
| `git stash` | Temporarily hide changes |
| `git log --oneline` | View commit history |
| `git status` | See what changed |
| `git diff` | See exact changes |

### 🐳 Docker Commands You MUST Know

| Command | Purpose |
|---------|---------|
| `docker build -t app .` | Build image from Dockerfile |
| `docker run -p 8080:8080 app` | Run container |
| `docker ps` | List running containers |
| `docker logs container` | View logs |
| `docker-compose up` | Start all services |
| `docker-compose down` | Stop all services |

### 📝 Interview Tips

1. **Git flow**: Know the standard — feature branch → PR/MR → review → merge to main
2. **Always commit with meaningful messages**: "Add user login endpoint" not "fix stuff"
3. **Never commit secrets** (passwords, API keys) to Git
4. **Docker**: Know that containers are ephemeral (data is lost when stopped unless using volumes)
5. **For interviews**: Draw the CI/CD pipeline on a whiteboard — interviewers love this!
