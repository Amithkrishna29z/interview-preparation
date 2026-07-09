# .NET Deployment & Production Ops (for Java Developers)

## Overview

The other guides get your code *written*; this one gets it *shipped and observable*. It covers the production concerns a junior .NET dev is expected to at least **recognize and discuss**: publishing, configuration & secrets across environments, structured logging, health checks, and containerizing with Docker. You've done all of this in Java (Spring profiles, Logback, actuator, Dockerized JARs) — here are the .NET equivalents.

You won't run all of this daily as a junior, but interviewers ask *"how would you configure this per environment?"* and *"how do you deploy it?"* — and having concrete answers separates you from candidates who only know localhost. All code targets **.NET 8**. Reuse your Java `Docker_Concepts_Study_Guide.md` and `CI_CD_Pipelines_Deep_Dive.md` for the container/pipeline *concepts* — this guide is the .NET-specific layer on top.

---

## Table of Contents

1. [Java → .NET Quick Mapping](#1-java--net-quick-mapping)
2. [Configuration: appsettings, Environments & the Options Pattern](#2-configuration-appsettings-environments--the-options-pattern)
3. [Secrets: user-secrets, Environment Variables, Key Vault](#3-secrets-user-secrets-environment-variables-key-vault)
4. [Structured Logging (ILogger & Serilog)](#4-structured-logging-ilogger--serilog)
5. [Health Checks & Readiness](#5-health-checks--readiness)
6. [Publishing: `dotnet publish`](#6-publishing-dotnet-publish)
7. [Containerizing with Docker](#7-containerizing-with-docker)
8. [Running Migrations in Production](#8-running-migrations-in-production)
9. [Kestrel, Reverse Proxies & Env Config in Containers](#9-kestrel-reverse-proxies--env-config-in-containers)
10. [A Minimal CI/CD Flow](#10-a-minimal-cicd-flow)
11. [Common Interview Questions](#11-common-interview-questions)
12. [Quick Reference Cheat Sheet](#12-quick-reference-cheat-sheet)

---

## 1. Java → .NET Quick Mapping

**Think of it like...** the same delivery truck, different labels. Spring profiles → .NET environments; Logback → Serilog; actuator health → health checks; a Dockerized JAR → a Dockerized DLL.

| Java / Spring | .NET | Purpose |
|---------------|------|---------|
| `application.properties` / `.yml` | `appsettings.json` | Base config |
| `application-prod.yml` | `appsettings.Production.json` | Per-environment overrides |
| `spring.profiles.active=prod` | `ASPNETCORE_ENVIRONMENT=Production` | Select environment |
| `@Value` / `@ConfigurationProperties` | `IConfiguration` / `IOptions<T>` | Read config |
| Spring profiles | Environments (Development/Staging/Production) | Environment concept |
| Logback / SLF4J | `ILogger<T>` / Serilog | Logging |
| MDC (mapped diagnostic context) | Log scopes / structured properties | Contextual logging |
| Actuator `/health` | Health checks middleware | Liveness/readiness |
| `mvn package` → `.jar` | `dotnet publish` → `.dll` + files | Build artifact |
| `java -jar app.jar` | `dotnet App.dll` | Run the artifact |
| `openjdk` base image | `mcr.microsoft.com/dotnet/aspnet` | Runtime container image |
| Flyway/Liquibase | EF Core migrations | Schema versioning |
| Environment variables | Environment variables (same, `__` for nesting) | Config override |

---

## 2. Configuration: appsettings, Environments & the Options Pattern

.NET builds configuration from **layered sources**, later ones overriding earlier — exactly like Spring's property precedence.

**Load order (later wins):**
```
appsettings.json                          (base, committed)
  -> appsettings.{Environment}.json       (e.g. appsettings.Production.json)
    -> User Secrets (Development only)
      -> Environment variables            (great for containers/CI)
        -> Command-line args
```

The active environment comes from `ASPNETCORE_ENVIRONMENT` (`Development`, `Staging`, `Production`). In dev it defaults to `Development`.

**Reading config three ways:**

```csharp
// 1) Directly via IConfiguration (quick, for one-off values):
var conn = builder.Configuration.GetConnectionString("Default");
var maxItems = builder.Configuration.GetValue<int>("Api:MaxPageSize");

// 2) THE OPTIONS PATTERN (preferred) — bind a config section to a strongly-typed class:
//    appsettings.json:  "EmailSettings": { "From": "no-reply@x.com", "RetryCount": 3 }
public class EmailSettings
{
    public string From { get; set; } = "";
    public int RetryCount { get; set; }
}

// Register in Program.cs — binds the section by name:
builder.Services.Configure<EmailSettings>(builder.Configuration.GetSection("EmailSettings"));

// 3) Inject IOptions<T> where you need it (like Spring's @ConfigurationProperties bean):
public class EmailService(IOptions<EmailSettings> options)
{
    private readonly EmailSettings _settings = options.Value;   // .Value = the bound instance
    public void Send() => Console.WriteLine($"From {_settings.From}, retries {_settings.RetryCount}");
}
```

> **`IOptions` vs `IOptionsSnapshot` vs `IOptionsMonitor`:** `IOptions<T>` is a singleton (read once). `IOptionsSnapshot<T>` re-reads per request (good for scoped services). `IOptionsMonitor<T>` supports live reload on config change. For a junior role, know `IOptions<T>` is the default and *why* the pattern exists (type safety + testability over scattered `Configuration["key"]` string lookups).

**Environment-specific override example** — `appsettings.Production.json` only needs the keys that differ:

```json
{
  "Logging": { "LogLevel": { "Default": "Warning" } },
  "ConnectionStrings": { "Default": "Server=prod-sql;Database=Tasks;..." }
}
```

Check the environment in code:

```csharp
if (app.Environment.IsDevelopment())        // reads ASPNETCORE_ENVIRONMENT
    app.UseSwagger();                        // Swagger only in dev
```

---

## 3. Secrets: user-secrets, Environment Variables, Key Vault

**Never commit secrets** (connection strings with passwords, JWT keys, API keys) to `appsettings.json`. Three tiers:

```bash
# LOCAL DEV — user-secrets: stored OUTSIDE the repo, in your user profile, keyed to the project.
dotnet user-secrets init                                     # adds a UserSecretsId to the .csproj
dotnet user-secrets set "Jwt:Key" "my-dev-secret-key"       # stored in ~/.microsoft/usersecrets/...
dotnet user-secrets set "ConnectionStrings:Default" "Server=localhost;..."
# These auto-load in Development — no code change needed.
```

```bash
# CONTAINERS / CI / PROD — environment variables override any appsettings key.
# Nested keys use '__' (double underscore) instead of ':'  (because ':' is illegal in some shells):
export ConnectionStrings__Default="Server=prod;..."
export Jwt__Key="prod-secret-from-vault"
```

- **Production:** inject via the platform's secret store — Azure Key Vault, AWS Secrets Manager, Kubernetes Secrets, or CI/CD secret variables — surfaced as environment variables. The app code doesn't change; only the config *source* does.

> **Interview soundbite:** "Secrets never live in source control. Locally I use `dotnet user-secrets`; in containers and prod I use environment variables backed by a secret manager (Key Vault / K8s Secrets). Configuration is layered, so the same code reads whichever source is present."

---

## 4. Structured Logging (ILogger & Serilog)

`ILogger<T>` is built in (like SLF4J) — inject it anywhere. **Structured logging** means logging *named data*, not just strings, so logs are queryable in tools like Seq, Elasticsearch, or Application Insights.

```csharp
public class TaskService(AppDbContext db, ILogger<TaskService> logger)   // ILogger<T> injected by DI
{
    public async Task<TaskResponseDto> CreateAsync(CreateTaskDto dto)
    {
        // STRUCTURED: {Title} is a named property, NOT string concatenation.
        // Log sinks capture Title as a queryable field (search "Title = 'X'"), not just text.
        logger.LogInformation("Creating task with title {Title}", dto.Title);

        var entity = new TaskItem { Title = dto.Title };
        db.Tasks.Add(entity);
        await db.SaveChangesAsync();

        logger.LogInformation("Created task {TaskId}", entity.Id);
        return new TaskResponseDto(entity.Id, entity.Title, entity.IsDone, entity.CreatedAt);
    }
}
```

**Log levels** (lowest→highest): `Trace < Debug < Information < Warning < Error < Critical`. Configure the minimum per namespace in `appsettings.json`:

```json
"Logging": {
  "LogLevel": {
    "Default": "Information",
    "Microsoft.EntityFrameworkCore.Database.Command": "Warning"   // silence noisy SQL logs
  }
}
```

**Serilog** (the popular add-on, like choosing Logback) gives rich sinks (files, Seq, Elasticsearch) and JSON output:

```bash
dotnet add package Serilog.AspNetCore
```

```csharp
using Serilog;

// Configure Serilog as the logging provider in Program.cs:
builder.Host.UseSerilog((context, config) =>
    config.ReadFrom.Configuration(context.Configuration)   // read sinks/levels from appsettings
          .WriteTo.Console()                               // structured console output
          .WriteTo.File("logs/app-.log", rollingInterval: RollingInterval.Day));

app.UseSerilogRequestLogging();   // one tidy log line per HTTP request (method, path, status, duration)
```

> **Why structured logging matters (interview):** "`logger.LogInformation("Order {OrderId} failed", id)` keeps `OrderId` as a searchable field. In prod I can query `OrderId = 123` across millions of logs — impossible with plain string concatenation."

---

## 5. Health Checks & Readiness

Health checks are the `/health` endpoint orchestrators (Kubernetes, load balancers) poll to know if your app is alive and ready — the .NET equivalent of Spring Actuator's health endpoint.

```bash
dotnet add package AspNetCore.HealthChecks.SqlServer   # optional: DB check package
```

```csharp
// Register in Program.cs:
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>("database");   // reports Unhealthy if the DB is unreachable

// Map endpoints:
app.MapHealthChecks("/health/live");   // LIVENESS: is the process up? (cheap)
app.MapHealthChecks("/health/ready");  // READINESS: are dependencies (DB) OK? (deeper)
```

- **Liveness** — "is the app running?" If it fails, the orchestrator **restarts** the pod.
- **Readiness** — "can it serve traffic (DB reachable, migrations applied)?" If it fails, the orchestrator **stops routing traffic** but doesn't restart.

In Kubernetes these map to `livenessProbe` and `readinessProbe` (see your `Kubernetes_Learning_Guide.md`).

---

## 6. Publishing: `dotnet publish`

`dotnet build` compiles; **`dotnet publish`** produces the deployable, self-contained set of files (the `.jar`-equivalent step).

```bash
# Standard: framework-dependent (needs .NET runtime installed on the host / in the base image):
dotnet publish src/TasksApi -c Release -o ./publish
#   -c Release  = optimized build (no debug symbols/checks)
#   -o          = output folder -> contains TasksApi.dll + deps + appsettings

# Run the published app:
dotnet ./publish/TasksApi.dll        # like: java -jar app.jar

# Self-contained (bundles the runtime — no .NET needed on the host, bigger output):
dotnet publish -c Release -r linux-x64 --self-contained true

# Single-file + trimmed (one executable, unused code removed — smaller footprint):
dotnet publish -c Release -r linux-x64 -p:PublishSingleFile=true -p:PublishTrimmed=true
```

| Mode | Host needs .NET? | Size | Use when |
|------|------------------|------|----------|
| Framework-dependent | Yes | Small | Base image already has the runtime (most common) |
| Self-contained | No | Large | Host has no .NET; full portability |
| Single-file trimmed | No | Smallest | CLIs, minimal containers |

> **Debug vs Release:** Always publish with `-c Release` for production — it enables optimizations and strips debug checks (like Maven's release profile).

---

## 7. Containerizing with Docker

The idiomatic .NET Dockerfile uses a **multi-stage build**: a big SDK image compiles, then only the runtime + published output ships (small, secure final image). Same pattern as a multi-stage JDK→JRE Java Dockerfile.

**`src/TasksApi/Dockerfile`:**

```dockerfile
# ---- Stage 1: BUILD (uses the full SDK — compiler, restore tools) ----
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy csproj first and restore — Docker layer caching skips re-restore when only code changes:
COPY ["TasksApi.csproj", "./"]
RUN dotnet restore "TasksApi.csproj"

# Copy the rest of the source and publish a Release build:
COPY . .
RUN dotnet publish "TasksApi.csproj" -c Release -o /app/publish

# ---- Stage 2: RUNTIME (tiny aspnet image — no SDK, no compiler) ----
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=build /app/publish .          # copy ONLY the published output from stage 1

EXPOSE 8080                                # Kestrel listens on 8080 in containers (see §9)
ENTRYPOINT ["dotnet", "TasksApi.dll"]      # like: java -jar app.jar
```

```bash
# Build and run:
docker build -t tasks-api ./src/TasksApi
docker run -p 8080:8080 \
    -e ASPNETCORE_ENVIRONMENT=Production \
    -e ConnectionStrings__Default="Server=host.docker.internal;Database=Tasks;..." \
    tasks-api
```

**Key images to know:**
- `mcr.microsoft.com/dotnet/sdk:8.0` — build stage (has the compiler).
- `mcr.microsoft.com/dotnet/aspnet:8.0` — runtime for web apps (smaller).
- `mcr.microsoft.com/dotnet/runtime:8.0` — runtime for console apps (smallest).

> Reuse `Docker_Concepts_Study_Guide.md` for layer caching, `.dockerignore`, and general container concepts — they're identical for .NET.

---

## 8. Running Migrations in Production

You can't run `dotnet ef database update` inside a slim runtime container (no SDK/EF tools). Three real-world strategies:

```csharp
// OPTION A — Apply migrations at startup (simple; fine for small apps / single instance):
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    db.Database.Migrate();        // applies any pending migrations on boot
}
// ⚠️ Risk with multiple instances starting at once — they can race. Prefer B or C at scale.
```

- **Option B — Migration bundle:** `dotnet ef migrations bundle` produces a self-contained executable you run as a separate deploy step (CI job or init container) *before* the app starts.
- **Option C — Generate a SQL script** (`dotnet ef migrations script --idempotent`) and let a DBA / pipeline apply it. Safest for regulated environments.

> **Interview answer:** "For a small service I migrate on startup. At scale I run migrations as a separate pipeline step (a migration bundle or idempotent SQL script) so app instances don't race, and so schema changes are reviewed and reversible."

---

## 9. Kestrel, Reverse Proxies & Env Config in Containers

- **Kestrel** is .NET's built-in web server (the Tomcat equivalent) — fast and cross-platform. It serves your app directly.
- In production it usually sits **behind a reverse proxy** (Nginx, YARP, or a cloud load balancer / Azure App Service) that handles TLS termination, and forwards to Kestrel over HTTP.
- Configure ports via env vars — in .NET 8 containers Kestrel listens on **port 8080** by default (`ASPNETCORE_HTTP_PORTS=8080`), which is why the Dockerfile exposes 8080.

```bash
# Common container env vars:
ASPNETCORE_ENVIRONMENT=Production      # select environment
ASPNETCORE_HTTP_PORTS=8080             # port Kestrel binds (non-root friendly)
ConnectionStrings__Default=...         # nested config via '__'
```

When behind a proxy, enable forwarded headers so the app sees the real client IP/scheme:

```csharp
app.UseForwardedHeaders(new ForwardedHeadersOptions
{
    ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto
});
```

---

## 10. A Minimal CI/CD Flow

A typical GitHub Actions pipeline for this API (concepts identical to your `CI_CD_Pipelines_Deep_Dive.md`, just .NET commands):

```yaml
name: build-test-publish
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      - run: dotnet restore                    # restore NuGet packages
      - run: dotnet build --no-restore -c Release
      - run: dotnet test --no-build -c Release # gate: fail the pipeline if tests fail
      - run: dotnet publish src/TasksApi -c Release -o ./publish
      # then: docker build/push, or deploy ./publish to the host
```

**The pipeline stages map 1:1 to Maven:** `restore` ≈ dependency resolution, `build`, `test` (quality gate), `publish` ≈ `package`, then deploy.

---

## 11. Common Interview Questions

### Q: How does configuration work across environments in .NET?
Config is layered: `appsettings.json` (base) → `appsettings.{Environment}.json` → user-secrets (dev) → environment variables → command-line args, each overriding the previous. The environment is chosen by `ASPNETCORE_ENVIRONMENT`. It's the direct analog of Spring profiles and property precedence.

### Q: What is the Options pattern and why use it?
Binding a config section to a strongly-typed class and injecting it via `IOptions<T>`. It replaces scattered `Configuration["key"]` string lookups with type-safe, testable, discoverable settings — analogous to Spring's `@ConfigurationProperties`.

### Q: Where do secrets go — not appsettings, so where?
Never in source control. Locally: `dotnet user-secrets` (stored in the user profile). In containers/CI/prod: environment variables (nested keys use `__`) backed by a secret manager — Azure Key Vault, AWS Secrets Manager, or Kubernetes Secrets. The code is unchanged; only the config source differs.

### Q: What is structured logging and why prefer it?
Logging named properties (`LogInformation("Order {OrderId} failed", id)`) rather than interpolated strings, so log sinks store `OrderId` as a queryable field. It makes production logs searchable and correlatable. `ILogger<T>` is built in; Serilog adds rich sinks.

### Q: What's the difference between liveness and readiness health checks?
Liveness answers "is the process alive?" — failure triggers a **restart**. Readiness answers "can it serve traffic (dependencies OK)?" — failure **stops routing traffic** without restarting. Orchestrators like Kubernetes poll both.

### Q: `dotnet build` vs `dotnet publish`?
`build` compiles for local development. `publish` produces the self-contained deployable output (DLLs, dependencies, config) — the artifact you ship, run with `dotnet App.dll`. Always publish with `-c Release`.

### Q: Why a multi-stage Dockerfile?
Stage 1 uses the large SDK image to compile; stage 2 copies only the published output into a small runtime image (`aspnet:8.0`). The final image has no compiler or source — smaller, faster to pull, and a reduced attack surface.

### Q: How do you run EF Core migrations in production?
Small apps: `db.Database.Migrate()` at startup. At scale: a separate migration step — an EF migration bundle or an idempotent SQL script applied by the pipeline — so instances don't race and changes are reviewable/reversible.

---

## 12. Quick Reference Cheat Sheet

```
CONFIG LAYERS (later overrides earlier):
  appsettings.json -> appsettings.{Env}.json -> user-secrets(dev) -> env vars -> CLI args
  Environment: ASPNETCORE_ENVIRONMENT = Development | Staging | Production
  Nested env var keys: ConnectionStrings__Default  (double underscore)

OPTIONS PATTERN:
  services.Configure<T>(config.GetSection("Name"));
  inject IOptions<T>, read .Value
  IOptions (singleton) | IOptionsSnapshot (per-request) | IOptionsMonitor (live reload)

SECRETS:
  local:  dotnet user-secrets set "Key" "value"
  prod:   env vars <- Key Vault / K8s Secrets / CI secrets   (never commit!)

LOGGING:
  ILogger<T> injected; LogInformation("... {Prop}", value)  <- structured
  levels: Trace<Debug<Information<Warning<Error<Critical
  Serilog: builder.Host.UseSerilog(...); app.UseSerilogRequestLogging();

HEALTH CHECKS:
  AddHealthChecks().AddDbContextCheck<T>()
  MapHealthChecks("/health/live")   liveness  -> restart on fail
  MapHealthChecks("/health/ready")  readiness -> stop traffic on fail

PUBLISH:
  dotnet publish -c Release -o ./publish     framework-dependent
  dotnet App.dll                             run it
  add -r linux-x64 --self-contained          bundle runtime

DOCKER (multi-stage):
  build:   mcr.microsoft.com/dotnet/sdk:8.0
  runtime: mcr.microsoft.com/dotnet/aspnet:8.0   (EXPOSE 8080)
  ENTRYPOINT ["dotnet", "App.dll"]
  docker run -p 8080:8080 -e ASPNETCORE_ENVIRONMENT=Production ...

MIGRATIONS IN PROD:
  small: db.Database.Migrate() at startup
  scale: migration bundle / idempotent SQL script as a pipeline step

CI/CD: restore -> build -c Release -> test -> publish -> docker build/push -> deploy

JAVA -> .NET:
  application-prod.yml   -> appsettings.Production.json
  spring.profiles.active -> ASPNETCORE_ENVIRONMENT
  @ConfigurationProperties -> IOptions<T>
  Logback/SLF4J          -> ILogger/Serilog
  actuator /health       -> health checks
  mvn package / java -jar -> dotnet publish / dotnet App.dll

GOLDEN RULES:
  1. Layered config; env vars win in containers.
  2. Secrets out of source control — always.
  3. Structured logs (named props), not string concatenation.
  4. Liveness = restart; readiness = stop traffic.
  5. Publish -c Release; ship a multi-stage Docker image.
  6. Run prod migrations as a controlled, non-racing step.
```

---

*Reuse for concepts: `Docker_Concepts_Study_Guide.md`, `Kubernetes_Learning_Guide.md`, `CI_CD_Pipelines_Deep_Dive.md`. Builds on the app from `DotNet_CRUD_WebAPI_Capstone.md`.*

*Last Updated: 2026-07-09*
