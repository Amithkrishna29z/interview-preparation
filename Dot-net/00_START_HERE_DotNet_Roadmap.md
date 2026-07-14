# 🚀 .NET Study Roadmap for Java Developers — START HERE

> **Who this is for:** You already know **Java** and want to land a **junior .NET / C# developer** job. This folder mirrors your `Java/` study guides, but for the .NET world — and **every topic is mapped back to its Java equivalent** so you learn by translation, not from scratch.

---

## Table of Contents

1. [The 60-Second Pitch: Java vs .NET](#the-60-second-pitch-java-vs-net)
2. [How to Use This Folder](#how-to-use-this-folder)
3. [The Big Java → .NET Mapping Table](#the-big-java--net-mapping-table)
4. [The .NET Ecosystem Explained (for Java Devs)](#the-net-ecosystem-explained-for-java-devs)
5. [Your Study Order (4-Week Plan)](#your-study-order-4-week-plan)
6. [The Guides in This Folder](#the-guides-in-this-folder)
7. [What You Can Reuse From Your Java Folder](#what-you-can-reuse-from-your-java-folder)
8. [Setup: Get .NET Running in 10 Minutes](#setup-get-net-running-in-10-minutes)
9. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## The 60-Second Pitch: Java vs .NET

**Think of it like learning a second language from the same family.** Java and C# were both inspired by C/C++ and share ~80% of their concepts. If Java is **Spanish**, C# is **Italian** — different words, but the grammar rhymes. Microsoft designed C# in 2000 by looking at Java and asking "what would we improve?"

- **Java** runs on the **JVM**. **C#** runs on the **CLR** (Common Language Runtime). Both compile to an intermediate bytecode, both have a garbage collector, both are statically typed and object-oriented.
- **Java** uses the **Spring** ecosystem for web apps. **C#** uses **ASP.NET Core**.
- **Java** uses **JPA/Hibernate** for databases. **C#** uses **Entity Framework Core**.
- The biggest mindset shift: C# has **more built-in language features** (properties, `async/await`, LINQ, `struct` value types, nullable reference types) where Java relies on libraries or boilerplate.

**Good news:** Because you know Java, you can realistically be interview-ready for a junior .NET role in **3–4 weeks** of focused study. You're not learning to program — you're learning new syntax for skills you already have.

---

## How to Use This Folder

1. **Start with the language** (Guides 1–7). Don't jump to web frameworks until C# syntax feels natural.
2. **Every guide has a `Java → .NET Mapping` table near the top.** Read that first — it anchors the new material to what you know.
3. **Type the code out.** Don't just read. Create a project (`dotnet new console`) and run every snippet.
4. **Finish with the interview guide** (Guide 13) one week before applying.
5. Each guide follows the same 5-part structure your Java guides use: TOC, "Think of it like..." analogies, line-commented code, Common Interview Questions, and a Cheat Sheet.

---

## The Big Java → .NET Mapping Table

### Language & Runtime

| Java | .NET / C# | Notes |
|---|---|---|
| JVM | CLR (Common Language Runtime) | Both run intermediate bytecode |
| Bytecode (`.class`) | IL / MSIL (Intermediate Language) | JIT-compiled at runtime |
| JDK | .NET SDK | Compiler + runtime + tools |
| JRE | .NET Runtime | Just runs apps |
| `javac` | `dotnet build` / `csc` | Compiler |
| `java MyApp` | `dotnet run` | Run an app |
| `.jar` | `.dll` / `.exe` | Compiled output |
| Maven / Gradle | NuGet + `.csproj` / MSBuild | Dependency + build |
| `pom.xml` / `build.gradle` | `.csproj` (XML) | Project file |
| Maven Central | NuGet.org | Package registry |
| `package com.x;` | `namespace Com.X;` | Code organization |
| `import java.util.List;` | `using System.Collections.Generic;` | Imports |

### Core Language Features

| Java | C# | Notes |
|---|---|---|
| `String` | `string` | C# has lowercase aliases for built-in types |
| `int`, `long`, `double` | `int`, `long`, `double` | Nearly identical |
| `Integer` (boxed) | `int?` / `Nullable<int>` | C# nullable value types |
| `final` | `readonly` (fields) / `const` | Different keywords by context |
| `static final` constant | `const` / `static readonly` | |
| getter/setter boilerplate | **Properties** (`{ get; set; }`) | C# bakes this into the language |
| `interface` | `interface` | Same idea |
| `abstract class` | `abstract class` | Same |
| `enum` | `enum` (+ richer with extensions) | |
| `record` (Java 14+) | `record` | Both have value-based records |
| `var` (Java 10+) | `var` | Type inference |
| Lambdas `x -> x+1` | Lambdas `x => x+1` | `=>` instead of `->` |
| Streams API | **LINQ** | C#'s killer feature |
| `Optional<T>` | nullable refs `T?` + `??` | Different null strategy |
| Checked exceptions | **No checked exceptions** | C# only has unchecked |
| `synchronized` | `lock` | Mutual exclusion |
| `CompletableFuture` | `Task<T>` + `async/await` | C# has cleaner async |
| `switch` expression | `switch` expression + pattern matching | |
| `==` vs `.equals()` | `==` (overloadable) vs `.Equals()` | Subtle differences |

### Frameworks & Libraries

| Java | .NET | Purpose |
|---|---|---|
| Spring Boot | ASP.NET Core | Web framework |
| Spring MVC `@RestController` | ASP.NET `[ApiController]` | REST controllers |
| Spring DI (`@Autowired`) | Built-in DI (`IServiceCollection`) | Dependency injection |
| JPA / Hibernate | Entity Framework Core | ORM |
| `@Entity` | EF entity class + `DbSet<T>` | DB mapping |
| Spring Data Repository | `DbContext` + LINQ | Data access |
| Spring Security | ASP.NET Core Identity + JWT | Auth |
| JUnit | xUnit / NUnit / MSTest | Unit testing |
| Mockito | Moq / NSubstitute | Mocking |
| SLF4J / Logback | `ILogger` / Serilog | Logging |
| Jackson | `System.Text.Json` / Newtonsoft.Json | JSON |
| Lombok | Properties + records (built-in) | No Lombok needed |
| Tomcat | Kestrel | Web server |
| `application.properties` | `appsettings.json` | Config |

---

## The .NET Ecosystem Explained (for Java Devs)

**Think of it like a city you're visiting** — you already know how cities work (roads, shops, transit); you just need the local names.

- **.NET** (just ".NET", formerly ".NET Core" / ".NET 5/6/7/8") — the **modern, cross-platform** runtime. This is what you learn. Runs on Windows, Linux, macOS. **Use .NET 8 (LTS) or newer.**
- **.NET Framework** (4.x) — the **old**, Windows-only version (2002–2019). You'll see it in legacy jobs but **don't start here**. It's like learning Java 8 in a Java 21 world.
- **C#** — the language (like Java the language).
- **CLR** — the runtime/VM (like the JVM).
- **NuGet** — package manager (like Maven Central + the `mvn` command combined).
- **`dotnet` CLI** — your Swiss-army tool: `dotnet new`, `dotnet build`, `dotnet run`, `dotnet test`, `dotnet add package`.
- **IDEs:** **Visual Studio** (Windows, full-featured, like IntelliJ Ultimate), **VS Code** + C# Dev Kit (lightweight, cross-platform), **JetBrains Rider** (the IntelliJ of .NET — if you love IntelliJ, you'll feel at home).

> **Key takeaway:** When a tutorial says ".NET Core" or ".NET 5/6/7/8", it's all the same modern .NET. When it says ".NET Framework 4.x", that's the legacy branch.

---

## Your Study Order (4-Week Plan)

### Week 1 — The Language
- **Day 1–2:** Guide 1 (Fundamentals) — syntax, types, properties, namespaces + Guide 15 (Modern C# 8–12 syntax — read alongside; NRTs, patterns, records)
- **Day 3–4:** Guide 2 (OOP & Type System) — classes, interfaces, generics, structs, records
- **Day 5:** Guide 3 (Collections) + Guide 14 (Delegates, Events & Functional C# — *heavily interviewed, no Java-only equivalent*)
- **Day 6–7:** Guide 4 (LINQ) — *this is the big C# superpower; spend real time here*

### Week 2 — Intermediate Language
- **Day 1–2:** Guide 5 (Async/Await) — *very heavily interviewed; different from Java threads*
- **Day 3:** Guide 6 (Exception Handling)
- **Day 4–5:** Guide 7 (CLR, Memory & GC)
- **Day 6–7:** Review + build a small console app (e.g., a to-do CLI)

### Week 3 — Web & Data
- **Day 1–3:** Guide 8 (ASP.NET Core Web API) — *the core job skill*
- **Day 4–5:** Guide 9 (Entity Framework Core)
- **Day 6–7:** Guide 10 (Dependency Injection & Middleware)

### Week 4 — Production Skills & Interview Prep
- **Day 1–2:** Guide 11 (Security / JWT / Identity)
- **Day 3:** Guide 12 (Testing)
- **Day 4–5:** Guide 16 (CRUD Web API Capstone) — *build it end-to-end, then rebuild from memory* + Guide 17 (Deployment & Ops — config, secrets, logging, Docker)
- **Day 6–7:** Guide 13 (Interview Questions) — drill until fluent

---

## The Guides in This Folder

| # | Guide | What You'll Learn | Java Analog |
|---|---|---|---|
| 0 | [**00_START_HERE_DotNet_Roadmap.md**](00_START_HERE_DotNet_Roadmap.md) | This file — your map | — |
| 1 | [**CSharp_Fundamentals_For_Java_Devs.md**](CSharp_Fundamentals_For_Java_Devs.md) | Syntax, types, properties, namespaces | Core Java |
| 2 | [**CSharp_OOP_And_Type_System.md**](CSharp_OOP_And_Type_System.md) | Classes, interfaces, generics, structs, records | OOP / Generics / Enums |
| 3 | [**CSharp_Collections_Guide.md**](CSharp_Collections_Guide.md) | List, Dictionary, HashSet, etc. | Collections Framework |
| 4 | [**CSharp_LINQ_Guide.md**](CSharp_LINQ_Guide.md) | Query data like a pro | Streams & Lambdas |
| 5 | [**CSharp_Async_Await_Concurrency.md**](CSharp_Async_Await_Concurrency.md) | `async`/`await`, `Task`, parallelism | Multithreading |
| 6 | [**CSharp_Exception_Handling_Guide.md**](CSharp_Exception_Handling_Guide.md) | try/catch/finally, custom exceptions | Exception Handling |
| 7 | [**DotNet_CLR_Memory_And_GC.md**](DotNet_CLR_Memory_And_GC.md) | How the runtime + GC work | JVM Internals / GC |
| 8 | [**ASPNET_Core_Web_API_Guide.md**](ASPNET_Core_Web_API_Guide.md) | Build REST APIs | Spring Boot / REST |
| 9 | [**Entity_Framework_Core_Guide.md**](Entity_Framework_Core_Guide.md) | Database access (ORM) | JPA / Hibernate |
| 10 | [**DependencyInjection_And_Middleware_Guide.md**](DependencyInjection_And_Middleware_Guide.md) | DI container + request pipeline | Spring DI / AOP |
| 11 | [**ASPNET_Core_Security_JWT_Identity.md**](ASPNET_Core_Security_JWT_Identity.md) | Auth & authorization | Spring Security |
| 12 | [**DotNet_Testing_xUnit_Moq_Guide.md**](DotNet_Testing_xUnit_Moq_Guide.md) | Unit & integration tests | JUnit / Mockito |
| 13 | [**DotNet_Junior_Interview_Questions.md**](DotNet_Junior_Interview_Questions.md) | 100+ Q&A to drill | Junior interview prep |
| 14 | [**CSharp_Delegates_Events_Functional.md**](CSharp_Delegates_Events_Functional.md) | Delegates, events, Func/Action, closures | Functional interfaces / Observer |
| 15 | [**CSharp_Modern_Language_Features.md**](CSharp_Modern_Language_Features.md) | C# 8–12 syntax, NRTs, patterns, records | Java 21 modern features |
| 16 | [**DotNet_CRUD_WebAPI_Capstone.md**](DotNet_CRUD_WebAPI_Capstone.md) | Build a full CRUD API end-to-end | Spring PetClinic-style build |
| 17 | [**DotNet_Deployment_And_Ops.md**](DotNet_Deployment_And_Ops.md) | Config, secrets, logging, Docker, publish | Spring profiles / Actuator / Dockerized JAR |
| 18 | [**SQL_Server_Guide_For_DotNet.md**](SQL_Server_Guide_For_DotNet.md) | SQL Server + T-SQL, connecting via ADO.NET/EF Core | MySQL/PG + JDBC (the .NET default DB) |

---

## What You Can Reuse From Your Java Folder

These topics are **language-agnostic** — the concepts transfer 1:1 to .NET. **Don't re-study them; just reuse your existing notes:**

- **SQL & Databases** → [`MySQL_Interview_Questions.md`](../Java/MySQL_Interview_Questions.md), [`PostgreSQL_Interview_Questions.md`](../Java/PostgreSQL_Interview_Questions.md), [`SQL_Advanced_Window_Functions.md`](../Java/SQL_Advanced_Window_Functions.md), [`Database_Concepts_Interview_Questions.md`](../Java/Database_Concepts_Interview_Questions.md) (EF Core still generates SQL — same fundamentals). **For SQL Server / T-SQL dialect specifics and how .NET connects to it, see Guide 18** — [`SQL_Server_Guide_For_DotNet.md`](SQL_Server_Guide_For_DotNet.md).
- **Docker / Kubernetes** → [`Docker_Concepts_Study_Guide.md`](../Java/Docker_Concepts_Study_Guide.md), [`Kubernetes_Learning_Guide.md`](../Java/Kubernetes_Learning_Guide.md) (.NET apps containerize the same way)
- **Networking / REST / HTTP** → [`Networking_Interview_Questions.md`](../Java/Networking_Interview_Questions.md), [`REST_API_HTTP_Interview_Questions.md`](../Java/REST_API_HTTP_Interview_Questions.md)
- **System Design / Microservices** → [`System_Design_Microservices_Interview_Questions.md`](../Java/System_Design_Microservices_Interview_Questions.md), [`Microservices_Saga_CQRS_EventSourcing.md`](../Java/Microservices_Saga_CQRS_EventSourcing.md)
- **Design Patterns** → [`GoF_Design_Patterns_Complete.md`](../Java/GoF_Design_Patterns_Complete.md) (patterns are universal; syntax differs slightly)
- **Git / CI-CD / DevOps** → [`Git_Docker_DevOps_Interview_Questions.md`](../Java/Git_Docker_DevOps_Interview_Questions.md), [`CI_CD_Pipelines_Deep_Dive.md`](../Java/CI_CD_Pipelines_Deep_Dive.md)
- **Data Structures & Algorithms** → [`DSA_Interview_Questions.md`](../Java/DSA_Interview_Questions.md), [`DSA_Dynamic_Programming.md`](../Java/DSA_Dynamic_Programming.md)
- **HR / Behavioral** → [`HR_Behavioral_Interview_Questions.md`](../Java/HR_Behavioral_Interview_Questions.md)
- **Kafka / Redis / Messaging** → concepts identical; .NET has `Confluent.Kafka` and `StackExchange.Redis` client libraries

> **The rule:** If a topic is about a *concept* (how databases work, how HTTP works, how to design a system), reuse Java notes. If it's about *.NET-specific syntax or tooling*, use this folder.

---

## Setup: Get .NET Running in 10 Minutes

```bash
# 1. Install the .NET SDK (8.0 LTS or newer) from https://dotnet.microsoft.com/download
# 2. Verify it works:
dotnet --version          # should print 8.0.x or higher

# 3. Create your first app:
dotnet new console -o HelloDotNet   # like 'mvn archetype:generate' for a console app
cd HelloDotNet
dotnet run                          # prints "Hello, World!"

# 4. Common commands you'll use daily:
dotnet new webapi -o MyApi   # create a Web API project (like Spring Initializr)
dotnet add package Newtonsoft.Json   # add a dependency (like adding to pom.xml)
dotnet build                 # compile
dotnet test                  # run tests
dotnet watch run             # hot-reload dev server
```

**Recommended IDE:** If you use IntelliJ for Java, get **JetBrains Rider** (same shortcuts, same feel). Otherwise **Visual Studio Community** (free, Windows) or **VS Code + C# Dev Kit** (cross-platform).

---

## Quick Reference Cheat Sheet

```
THE ONE-PAGE TRANSLATION:

Runtime:     JVM            → CLR
SDK:         JDK            → .NET SDK
Build:       Maven/Gradle   → dotnet CLI + NuGet
Language:    Java           → C#
Web:         Spring Boot    → ASP.NET Core
ORM:         JPA/Hibernate  → Entity Framework Core
DI:          @Autowired     → built-in IServiceCollection
Testing:     JUnit+Mockito  → xUnit+Moq
Logging:     SLF4J          → ILogger/Serilog
JSON:        Jackson        → System.Text.Json
Config:      application.properties → appsettings.json
Server:      Tomcat         → Kestrel

SYNTAX QUICK HITS:
  System.out.println(x)   →  Console.WriteLine(x)
  String                  →  string
  x -> x + 1              →  x => x + 1
  list.stream()...        →  list.Where()/Select()... (LINQ)
  getName()/setName()     →  public string Name { get; set; }
  Optional<T>             →  T?  (nullable reference)
  CompletableFuture<T>    →  Task<T>
  @Override               →  override (keyword, not annotation)
  implements / extends    →  : (colon for both)
  package                 →  namespace
  final field             →  readonly field

GOLDEN RULES:
  1. Use modern .NET (8+), NOT legacy .NET Framework 4.x
  2. Master LINQ and async/await — most-tested C# topics
  3. ASP.NET Core + EF Core = your core job skills
  4. Reuse Java notes for SQL, Docker, system design, DSA
  5. Build one real CRUD Web API before interviewing
```

---

*Study order: Guides 1 → 17. Language + Guides 14–15 in Weeks 1–2; build the Guide 16 capstone and skim Guide 17 in Week 4; drill interview questions (13) last. Guide 18 (SQL Server) pairs with Guide 9 (EF Core) in Week 3.*

*Last Updated: 2026-07-14*
