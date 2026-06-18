# Maven Build Tool Guide (Spring Boot Focus)

## Overview

Maven is a **build automation and dependency management** tool for Java projects. It handles compiling source code, running tests, packaging your application into a JAR/WAR, and pulling in third-party libraries — all driven by a single configuration file: `pom.xml`.

Spring Boot projects default to Maven, though Gradle is a popular alternative. As a junior developer you will read, write, and debug `pom.xml` files daily.

---

## Table of Contents

1. [What Maven Does](#1-what-maven-does)
2. [pom.xml Structure](#2-pomxml-structure)
3. [Dependencies and Spring Boot Starters](#3-dependencies-and-spring-boot-starters)
4. [Dependency Scopes](#4-dependency-scopes)
5. [The Build Lifecycle](#5-the-build-lifecycle)
6. [Common Maven Commands](#6-common-maven-commands)
7. [spring-boot-maven-plugin](#7-spring-boot-maven-plugin)
8. [Local Repository and Maven Central](#8-local-repository-and-maven-central)
9. [Multi-Module Projects](#9-multi-module-projects)
10. [Gradle Awareness](#10-gradle-awareness)
11. [Common Mistakes](#11-common-mistakes)
12. [Common Interview Questions](#12-common-interview-questions)
13. [Quick Reference Cheat Sheet](#13-quick-reference-cheat-sheet)

---

## 1. What Maven Does

Maven solves two problems that would otherwise be painful:

- **Dependency management** — you declare what libraries you need; Maven downloads them from Maven Central and wires up transitive dependencies automatically.
- **Build automation** — a fixed lifecycle (compile → test → package → …) means every developer and every CI server builds the project the same way.

Think of Maven as a recipe book: you write down the ingredients (`<dependencies>`) and the oven settings, and Maven bakes the JAR.

---

## 2. pom.xml Structure

The **Project Object Model** (`pom.xml`) lives in the project root. Below is an annotated minimal Spring Boot pom.xml.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <!-- ── Parent: inherits Spring Boot's managed dependency versions (BOM) ── -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.5</version>
        <relativePath/> <!-- fetch from repository, not local file system -->
    </parent>

    <!-- ── GAV: the unique identifier for THIS project ── -->
    <groupId>com.example</groupId>       <!-- reverse-domain namespace -->
    <artifactId>my-app</artifactId>      <!-- project/module name -->
    <version>0.0.1-SNAPSHOT</version>    <!-- SNAPSHOT = work in progress -->
    <packaging>jar</packaging>           <!-- jar (default) | war | pom -->

    <!-- ── Properties: override defaults from the parent ── -->
    <properties>
        <java.version>17</java.version>  <!-- sets source/target compiler version -->
    </properties>

    <!-- ── Dependencies ── -->
    <dependencies>

        <!-- Web (REST controllers, embedded Tomcat) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <!-- NO <version> — the parent BOM manages it -->
        </dependency>

        <!-- JPA + Hibernate -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!-- H2 in-memory DB — runtime only, not needed at compile time -->
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- Testing utilities — only on the test classpath -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

    </dependencies>

    <!-- ── Build plugins ── -->
    <build>
        <plugins>
            <!-- Builds the executable fat JAR and enables spring-boot:run -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

### Key concepts

| Concept | Meaning |
|---|---|
| **GAV** | `groupId:artifactId:version` — the globally unique address of any artifact |
| **Parent / BOM** | Bill of Materials: a curated list of compatible library versions so you don't have to specify them |
| **SNAPSHOT** | Version still under development; Maven re-downloads it on each build |
| **RELEASE** | Stable, immutable version (e.g., `1.0.0`) |

---

## 3. Dependencies and Spring Boot Starters

A **starter** is a convenience dependency that bundles everything needed for a feature — you add one line instead of five.

```xml
<!-- Brings in spring-webmvc, jackson, embedded Tomcat, and more -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

Common starters:

| Starter | What it gives you |
|---|---|
| `spring-boot-starter-web` | REST controllers, embedded Tomcat, Jackson JSON |
| `spring-boot-starter-data-jpa` | Hibernate, Spring Data repositories |
| `spring-boot-starter-validation` | Bean Validation (`@NotNull`, `@Size`, …) |
| `spring-boot-starter-security` | Spring Security filters and defaults |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, MockMvc |
| `spring-boot-starter-actuator` | Health/info/metrics endpoints |

**Transitive dependencies** — when you add a starter, Maven also downloads every library *that starter depends on*. Run `mvn dependency:tree` to see the full graph.

**Why no `<version>` tag?** The `spring-boot-starter-parent` inherits from `spring-boot-dependencies` (the BOM), which declares tested, compatible versions for hundreds of libraries. Adding your own version tag overrides the BOM and risks conflicts.

---

## 4. Dependency Scopes

Scope controls when a dependency is on the classpath.

| Scope | Compile | Test | Runtime JAR | Typical use |
|---|---|---|---|---|
| `compile` (default) | yes | yes | yes | Most app code |
| `provided` | yes | yes | no | Servlet API (container provides it at runtime) |
| `runtime` | no | yes | yes | JDBC drivers, H2 |
| `test` | no | yes | no | JUnit, Mockito |

---

## 5. The Build Lifecycle

Maven defines a **fixed sequence of phases**. Running a later phase automatically runs all earlier phases.

```
validate → compile → test → package → verify → install → deploy
```

| Phase | What happens |
|---|---|
| `validate` | Checks pom.xml is well-formed and complete |
| `compile` | Compiles `src/main/java` → `target/classes` |
| `test` | Compiles and runs `src/test/java` (Surefire plugin) |
| `package` | Bundles compiled code into a JAR/WAR in `target/` |
| `verify` | Runs integration tests (Failsafe plugin) |
| `install` | Copies the artifact into `~/.m2` for local use by other projects |
| `deploy` | Uploads the artifact to a remote repository (Nexus, Artifactory) |

`clean` is a separate lifecycle that deletes the `target/` directory. It is not part of the sequence above — you invoke it explicitly.

---

## 6. Common Maven Commands

```bash
# Compile and package into an executable fat JAR
mvn clean package

# Run all unit tests
mvn test

# Skip tests (use sparingly — never in CI)
mvn clean package -DskipTests

# Run the Spring Boot app directly (no JAR needed)
mvn spring-boot:run

# Install artifact to local ~/.m2 (useful for multi-module projects)
mvn install

# Print the full dependency tree (great for debugging conflicts)
mvn dependency:tree

# Show the effective pom after parent merging
mvn help:effective-pom
```

---

## 7. spring-boot-maven-plugin

The `spring-boot-maven-plugin` is what turns a plain JAR into an **executable fat/uber JAR** — a self-contained archive that bundles your code, all dependencies, and an embedded Tomcat or Jetty.

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <!-- version inherited from spring-boot-starter-parent -->
        </plugin>
    </plugins>
</build>
```

What it provides:

- **`repackage` goal** (runs automatically during `package`) — replaces the plain JAR with the fat JAR; the original JAR is renamed with a `.original` suffix.
- **`spring-boot:run` goal** — launches the app in-process without building a JAR first; faster for development.
- **`spring-boot:build-image` goal** — builds an OCI container image using Cloud Native Buildpacks (no Dockerfile needed).

The resulting fat JAR is runnable with: `java -jar target/my-app-0.0.1-SNAPSHOT.jar`

---

## 8. Local Repository and Maven Central

```
~/.m2/repository/
  com/example/my-app/0.0.1-SNAPSHOT/   ← your own installed artifacts
  org/springframework/boot/...          ← downloaded starters
  com/fasterxml/jackson/...             ← transitive dependencies
```

- **Maven Central** (`https://repo1.maven.org/maven2`) is the default public registry.
- Maven checks `~/.m2` first; only downloads from Central if the artifact is missing or is a SNAPSHOT.
- Corporate teams often add a **private repository** (Nexus, Artifactory) in `<repositories>` or `settings.xml` to serve internal artifacts and proxy Central.

---

## 9. Multi-Module Projects

A multi-module project uses a **parent `pom.xml`** with `<packaging>pom</packaging>` to aggregate child modules:

```xml
<!-- root pom.xml -->
<modules>
    <module>api</module>      <!-- REST layer -->
    <module>service</module>  <!-- business logic -->
    <module>domain</module>   <!-- entities, repositories -->
</modules>
```

Running `mvn clean package` from the root builds all modules in dependency order. Child modules inherit `<dependencies>` and `<dependencyManagement>` from the parent, keeping versions consistent across the project.

---

## 10. Gradle Awareness

Gradle is the main alternative to Maven and is the default for Android and many Kotlin projects. Spring Initializr lets you choose either.

**build.gradle (Groovy DSL) equivalent of the pom.xml above:**

```groovy
plugins {
    id 'org.springframework.boot' version '3.2.5'
    id 'io.spring.dependency-management' version '1.1.5'
    id 'java'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
java { sourceCompatibility = '17' }

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'com.h2database:h2'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

**Maven vs Gradle comparison:**

| Dimension | Maven | Gradle |
|---|---|---|
| Config language | XML (pom.xml) | Groovy or Kotlin DSL |
| Build speed | Slower (no incremental) | Faster (incremental + build cache) |
| Convention | Strict, opinionated | Flexible, scriptable |
| Learning curve | Lower for beginners | Steeper (DSL + task graph) |
| Spring Boot default | Yes (Initializr default) | Available option |
| Android | Not used | Required |
| Corporate preference | Very common in enterprise | Growing, especially Kotlin teams |

As a junior developer, master Maven first. You will encounter Gradle — the concepts (dependencies, lifecycle, plugins) transfer directly.

---

## 11. Common Mistakes

| Mistake | Problem | Fix |
|---|---|---|
| Adding `<version>` to a starter | Overrides the BOM; causes version conflicts | Remove the `<version>` tag; let the parent manage it |
| Wrong scope for a driver | Driver not on runtime classpath | Use `<scope>runtime</scope>` for JDBC/H2 drivers |
| Forgetting `clean` before `package` | Stale `.class` files from a renamed class end up in the JAR | Always use `mvn clean package` |
| Depending on a `test`-scoped artifact in main code | Compilation fails on CI | Move it to `compile` scope or restructure |
| Two libraries pulling in different versions of the same transitive dep | `NoSuchMethodError` at runtime | Use `mvn dependency:tree` to find the conflict; add an explicit `<dependency>` to pin the version |
| Missing `spring-boot-maven-plugin` | Plain JAR is not executable; embedded Tomcat missing | Add the plugin to `<build><plugins>` |

---

## 12. Common Interview Questions

**Q: What is a Maven BOM and why does Spring Boot use one?**
A BOM (Bill of Materials) is a special POM that declares a curated set of dependency versions with no actual code. `spring-boot-starter-parent` inherits from `spring-boot-dependencies` (the BOM), so all starter versions are pre-tested for compatibility. You omit `<version>` tags in your pom.xml because the BOM already supplies them, eliminating version-conflict guesswork.

**Q: What is the difference between `mvn package` and `mvn install`?**
`package` compiles, tests, and creates the JAR/WAR in the `target/` directory — it stays local to that project. `install` does everything `package` does, then copies the artifact into `~/.m2/repository`, making it available as a dependency to other Maven projects on the same machine.

**Q: What does the `spring-boot-maven-plugin` add that the standard compiler plugin does not?**
The standard compiler just produces a thin JAR with your own classes. The `spring-boot-maven-plugin` repackages it into a fat/uber JAR that embeds all dependencies and an embedded servlet container (Tomcat), making the JAR self-runnable with `java -jar`. It also provides the `spring-boot:run` goal for quick local development.

**Q: What is a dependency scope and give an example of when you would use `provided`?**
Scope controls on which classpath a dependency appears. `provided` means the dependency is needed to compile but will be supplied by the runtime environment and should not be bundled in the JAR. The classic example is `javax.servlet-api`: when deploying a WAR to an external Tomcat, Tomcat already provides the Servlet API — bundling it would cause class-loading conflicts.

**Q: How do you resolve a dependency version conflict in Maven?**
Run `mvn dependency:tree` to identify which libraries are pulling in conflicting versions of the same artifact. Then add an explicit `<dependency>` entry in your pom.xml with the desired version to override Maven's default nearest-wins resolution, or add a `<dependencyManagement>` block to pin the version project-wide.

---

## 13. Quick Reference Cheat Sheet

```
── pom.xml skeleton ────────────────────────────────────────
<parent>  spring-boot-starter-parent  (BOM)
<groupId> / <artifactId> / <version>  (GAV)
<properties>  java.version
<dependencies>  starters (no <version> needed)
<build><plugins>  spring-boot-maven-plugin

── Lifecycle (left to right) ───────────────────────────────
validate → compile → test → package → verify → install → deploy
clean  (separate — deletes target/)

── Daily commands ───────────────────────────────────────────
mvn clean package          build executable fat JAR
mvn test                   run unit tests only
mvn spring-boot:run        run app without building JAR
mvn install                build + copy to ~/.m2
mvn dependency:tree        inspect full dep graph

── Scopes ───────────────────────────────────────────────────
compile   (default) always on classpath
runtime   not needed to compile (drivers)
test      only during testing
provided  compile only; container supplies at runtime

── Gradle equivalent scopes ─────────────────────────────────
implementation  ≈ compile
runtimeOnly     ≈ runtime
testImplementation ≈ test
compileOnly     ≈ provided
```

---

*Last Updated: 2026-06-18*
