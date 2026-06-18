# Spring Boot Configuration & Profiles

> **How to use this guide (junior dev):** The single most asked question here is **"How do you manage different DB credentials per environment?"** — start with [section 4](#4-profiles-dev--prod--staging) and be able to explain it out loud. After that, cover `@ConfigurationProperties` and the config precedence order. Everything else supports those two.

---

## Table of Contents

1. [Overview](#1-overview)
2. [application.properties vs application.yml](#2-applicationproperties-vs-applicationyml)
3. [Injecting Properties into Beans](#3-injecting-properties-into-beans)
4. [Profiles: dev / prod / staging](#4-profiles-dev--prod--staging)
5. [Externalized Config & 12-Factor](#5-externalized-config--12-factor)
6. [Config Precedence Order](#6-config-precedence-order)
7. [Common Mistakes & Pitfalls](#7-common-mistakes--pitfalls)
8. [Common Interview Questions](#8-common-interview-questions)
9. [Quick Reference Cheat Sheet](#9-quick-reference-cheat-sheet)

---

## 1. Overview

Spring Boot auto-configures your application from a layered set of property sources — files, environment variables, CLI arguments, and more. As a junior developer you will touch this system every day: setting a port, connecting to a database, switching between local and production credentials. Understanding how it works prevents the most common bugs: wrong database connected, secrets leaking into git, or the wrong profile loading on the server.

---

## 2. application.properties vs application.yml

Both files live in `src/main/resources/` and hold the same data. Spring Boot reads either one automatically.

### Same config written both ways

**application.properties** — flat key=value, simple, universally familiar:

```properties
# Flat key=value — each property on its own line
server.port=8080
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=admin
spring.datasource.password=secret
spring.jpa.hibernate.ddl-auto=update
```

**application.yml** — YAML hierarchy, less repetition for grouped keys:

```yaml
# Indented hierarchy — groups related keys visually
server:
  port: 8080

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: admin
    password: secret
  jpa:
    hibernate:
      ddl-auto: update
```

### When to use which

| Factor | .properties | .yml |
|--------|-------------|------|
| Simple flat config | Great fit | Works, but overkill |
| Deeply nested keys | Verbose (lots of `spring.x.y.z`) | Cleaner hierarchy |
| Multiple profiles in one file | No | Yes — use `---` document separator |
| Team familiarity | Universal | Requires YAML knowledge |
| Tooling (older projects) | Always supported | Supported since Boot 1.x |

**Rule of thumb:** pick one format and stick to it per project. Do **not** mix `application.properties` and `application.yml` for the same profile — Spring Boot loads both but properties files take precedence, causing confusing bugs.

---

## 3. Injecting Properties into Beans

### 3.1 `@Value` — single property injection

Use `@Value` for a single, simple property. Add a default after `:` to avoid startup failures when the property is missing.

```java
@Component
public class AppInfo {

    // Injects the value of app.name; defaults to "MyApp" if not set
    @Value("${app.name:MyApp}")
    private String appName;

    // Injects server.port as an int; defaults to 8080
    @Value("${server.port:8080}")
    private int port;

    // SpEL expression: reads a system property at runtime
    @Value("#{systemProperties['user.home']}")
    private String userHome;
}
```

**Limitation:** `@Value` scatters property keys across many classes. If a key name changes you must hunt down every `@Value`. For related properties, prefer `@ConfigurationProperties`.

---

### 3.2 `@ConfigurationProperties` — type-safe grouped binding

Bind an entire prefix of properties to a POJO. This is the preferred approach for any group of related properties.

**application.yml:**

```yaml
app:
  mail:
    host: smtp.example.com
    port: 587
    username: noreply@example.com
    from: "No Reply <noreply@example.com>"
    retry-count: 3        # relaxed binding: maps to retryCount
```

**Config POJO (constructor binding — immutable, recommended):**

```java
@ConfigurationProperties(prefix = "app.mail")
@Validated                         // triggers Bean Validation on startup
public class MailProperties {

    @NotBlank
    private final String host;

    @Min(1) @Max(65535)
    private final int port;

    private final String username;
    private final String from;
    private final int retryCount;   // relaxed binding: retry-count → retryCount

    // Spring Boot 2.2+ uses constructor binding automatically
    public MailProperties(String host, int port, String username,
                          String from, int retryCount) {
        this.host = host;
        this.port = port;
        this.username = username;
        this.from = from;
        this.retryCount = retryCount;
    }

    // getters only — immutable
    public String getHost()      { return host; }
    public int getPort()         { return port; }
    public String getUsername()  { return username; }
    public String getFrom()      { return from; }
    public int getRetryCount()   { return retryCount; }
}
```

**Register it** (one of two ways):

```java
// Option A: annotate the config class itself
@SpringBootApplication
@EnableConfigurationProperties(MailProperties.class)
public class Application { ... }

// Option B: annotate the POJO with @Component (simpler for small projects)
@Component
@ConfigurationProperties(prefix = "app.mail")
public class MailProperties { ... }
```

**Inject and use it like any other bean:**

```java
@Service
public class EmailService {

    private final MailProperties mail;

    public EmailService(MailProperties mail) {
        this.mail = mail;
    }

    public void send(String to, String subject) {
        // mail.getHost(), mail.getPort(), etc.
    }
}
```

**Relaxed binding** means Spring Boot automatically maps `retry-count` (kebab-case in YAML) → `retryCount` (camelCase in Java). It also accepts `RETRY_COUNT` (env var) and `retry_count`. You never need to match the format exactly.

---

## 4. Profiles: dev / prod / staging

Profiles let you ship one codebase with different config per environment. Think of a profile as a named overlay that replaces or adds to the base `application.yml`.

### File naming convention

```
src/main/resources/
  application.yml              ← base config (all environments)
  application-dev.yml          ← overrides for local development
  application-prod.yml         ← overrides for production
  application-staging.yml      ← overrides for staging
```

**application.yml (base — safe defaults):**

```yaml
spring:
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: validate

app:
  feature-flags:
    new-checkout: false
```

**application-dev.yml (local dev overrides):**

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/myapp_dev
    username: dev_user
    password: dev_pass   # OK here — never commits real secrets
  jpa:
    show-sql: true            # verbose SQL logs locally
    hibernate:
      ddl-auto: create-drop   # rebuild schema on restart

app:
  feature-flags:
    new-checkout: true        # test new features locally
```

**application-prod.yml (production — reads secrets from env vars):**

```yaml
spring:
  datasource:
    url: ${DATABASE_URL}           # injected by the platform (Heroku, K8s, etc.)
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate           # never auto-migrate in prod
```

### Activating a profile

Three ways — listed from most common to least:

```bash
# 1. Environment variable (most common in Docker/K8s/CI)
export SPRING_PROFILES_ACTIVE=prod

# 2. CLI argument (useful for quick local switches)
java -jar app.jar --spring.profiles.active=prod

# 3. Inside application.yml (set default profile; override in prod)
spring:
  profiles:
    active: dev
```

Multiple profiles can be active at once:

```bash
SPRING_PROFILES_ACTIVE=prod,metrics
```

### `@Profile` on beans

Register a bean only when a specific profile is active:

```java
// Only loaded in dev — uses an in-memory stub instead of real AWS S3
@Service
@Profile("dev")
public class LocalStorageService implements StorageService {
    // saves files to local /tmp
}

// Only loaded in prod
@Service
@Profile("prod")
public class S3StorageService implements StorageService {
    // saves files to AWS S3
}

// Active in every profile EXCEPT prod
@Component
@Profile("!prod")
public class DevDataSeeder implements ApplicationRunner {
    // seeds test data on startup
}
```

---

## 5. Externalized Config & 12-Factor

The [12-Factor App](https://12factor.net/config) principle: **store config in the environment, not in code**. Credentials, URLs, and API keys must never be committed to source control.

**Practical rules for a junior dev:**

1. Any secret (`password`, API key, token) lives in an environment variable — never in a committed yml file.
2. `application-prod.yml` should contain only references like `${DATABASE_PASSWORD}` — not the actual value.
3. Add `application-prod.yml` to `.gitignore` if it ever holds real values (or better: keep it empty of secrets and rely on env vars entirely).
4. Use `.env` files locally with a tool like `direnv` or IDE run configurations — never commit `.env`.

```properties
# .gitignore — protect secret files
.env
application-prod.properties
secrets/
```

---

## 6. Config Precedence Order

When the same property is defined in multiple places, Spring Boot applies this priority (higher number = higher priority, wins):

| Priority | Source |
|----------|--------|
| 1 (lowest) | Default values coded in `@Value` / `@ConfigurationProperties` |
| 2 | `application.properties` / `application.yml` (packaged in jar) |
| 3 | `application-{profile}.properties` / `application-{profile}.yml` |
| 4 | `application.properties` / `application.yml` (outside jar, same directory) |
| 5 | OS environment variables (`DATABASE_URL`, `SPRING_PROFILES_ACTIVE`) |
| 6 | JVM system properties (`-Dserver.port=9090`) |
| 7 (highest) | CLI arguments (`--server.port=9090`) |

**Key takeaway:** environment variables always override your yml files. This is intentional — it lets ops teams override settings without touching code.

---

## 7. Common Mistakes & Pitfalls

| Mistake | What goes wrong | Fix |
|---------|----------------|-----|
| Hardcoding secrets in yml | Credentials leak into git history | Use `${ENV_VAR}` placeholders; add secrets to `.gitignore` |
| Wrong profile active on server | App connects to dev DB in prod | Explicitly set `SPRING_PROFILES_ACTIVE` in the deployment env |
| Mixing properties + yml | properties file silently wins; yml ignored | Pick one format per project and delete the other |
| Forgetting env var override | yml value used despite env var being set | Check env var name matches exactly (Spring maps `.` → `_`, lowercase → uppercase) |
| `@Value` with no default, missing property | `IllegalArgumentException` on startup | Add `${prop:defaultValue}` or ensure the property is always provided |
| `@ConfigurationProperties` not registered | Properties not bound, null fields | Add `@EnableConfigurationProperties` or `@Component` to the POJO |
| `ddl-auto=create-drop` in prod | Database wiped on every restart | Set `ddl-auto=validate` in prod profile |

---

## 8. Common Interview Questions

**Q: What is the difference between `@Value` and `@ConfigurationProperties`?**
`@Value` injects a single property directly into a field and is fine for isolated config values. `@ConfigurationProperties` binds a whole group of properties to a POJO, provides type safety, supports validation via `@Validated`, and is easier to test — prefer it for any related set of properties like database or mail settings.

**Q: How do you prevent secrets from leaking into source control?**
Store secrets only in environment variables (or a secrets manager like AWS Secrets Manager / Vault). Reference them in yml as `${DATABASE_PASSWORD}`. Never commit actual credentials; use `.gitignore` for any local files that hold real values.

**Q: What is the Spring Boot config precedence order?**
From lowest to highest: default values → packaged `application.yml` → profile-specific yml → external `application.yml` outside the jar → OS environment variables → JVM system properties → CLI arguments. Environment variables therefore always override yml files, which is how you safely change config per environment without rebuilding the jar.

**Q: How do you activate a profile in production?**
Set the `SPRING_PROFILES_ACTIVE` environment variable on the server or in the container spec (e.g., a Kubernetes `env:` entry or a Docker `-e` flag). Avoid hardcoding the active profile inside `application.yml` for production — it defeats the purpose of environment-specific overrides.

**Q: What is relaxed binding in `@ConfigurationProperties`?**
Spring Boot maps `retry-count` (kebab-case in yml), `RETRY_COUNT` (env var uppercase), and `retry_count` (underscore) all to the Java field `retryCount` (camelCase). You never need to exactly match the format between your YAML key and your Java field name.

**Q: Can you have multiple profiles active at the same time?**
Yes. Set `SPRING_PROFILES_ACTIVE=prod,metrics` — Spring Boot loads `application-prod.yml` and `application-metrics.yml`, merging them with later profiles taking precedence on conflicts.

---

## 9. Quick Reference Cheat Sheet

```
ACTIVATE A PROFILE
  Env var:  SPRING_PROFILES_ACTIVE=prod
  CLI arg:  --spring.profiles.active=prod
  YAML:     spring.profiles.active: dev    (base file only, as default)

INJECT A SINGLE PROPERTY
  @Value("${app.name}")             required — fails on startup if missing
  @Value("${app.name:DefaultApp}")  with fallback default

BIND A GROUP OF PROPERTIES
  @ConfigurationProperties(prefix = "app.mail")
  + @Validated + @EnableConfigurationProperties(MailProperties.class)

PROFILE FILE NAMING
  application.yml              ← base (all profiles)
  application-dev.yml          ← dev overrides
  application-prod.yml         ← prod overrides (use ${ENV_VAR} for secrets)

CONFIG PRECEDENCE (lowest → highest)
  defaults → application.yml → application-{profile}.yml
  → external files → env vars → JVM -D props → CLI --args

RELAXED BINDING (all map to retryCount in Java)
  retry-count   (YAML kebab-case)
  RETRY_COUNT   (env var)
  retry_count   (underscore)

COMMON ENV VARS SPRING BOOT READS
  SPRING_PROFILES_ACTIVE        which profile(s) to load
  SERVER_PORT                   overrides server.port
  DATABASE_URL                  reference as ${DATABASE_URL} in yml
  SPRING_DATASOURCE_URL         directly overrides spring.datasource.url
```

---

*Last Updated: 2026-06-18*
