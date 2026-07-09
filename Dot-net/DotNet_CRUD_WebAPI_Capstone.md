# Capstone: Build a Complete CRUD Web API (for Java Developers)

## Overview

Every other guide teaches a *piece*. This one **assembles them into one runnable project** — the single most valuable thing you can do before interviewing. You'll build a **Tasks API** (a to-do backend) from `dotnet new` to a running, tested, secured endpoint, combining: **ASP.NET Core + EF Core + DI + DTOs + validation + JWT auth + xUnit tests**.

**Think of it like...** the Spring Boot "PetClinic" or a `spring-boot-starter` CRUD demo — the reference project you can rebuild from memory and talk through in an interview. When an interviewer says *"walk me through a project you built,"* this is your answer.

Build it once by typing (don't copy-paste), then rebuild it from scratch a second time without looking. All code targets **.NET 8**. Reference the deeper guides (linked throughout) when you want the *why* behind each piece.

---

## Table of Contents

1. [What We're Building](#1-what-were-building)
2. [Step 0: Prerequisites & Project Layout](#2-step-0-prerequisites--project-layout)
3. [Step 1: Create the Solution & Projects](#3-step-1-create-the-solution--projects)
4. [Step 2: The Entity & DbContext (EF Core)](#4-step-2-the-entity--dbcontext-ef-core)
5. [Step 3: DTOs & Validation](#5-step-3-dtos--validation)
6. [Step 4: The Service Layer (DI)](#6-step-4-the-service-layer-di)
7. [Step 5: The Controller (CRUD Endpoints)](#7-step-5-the-controller-crud-endpoints)
8. [Step 6: Wire It Up in Program.cs](#8-step-6-wire-it-up-in-programcs)
9. [Step 7: Migrations & Run It](#9-step-7-migrations--run-it)
10. [Step 8: Add JWT Authentication](#10-step-8-add-jwt-authentication)
11. [Step 9: Write Tests](#11-step-9-write-tests)
12. [Step 10: Error Handling & Polish](#12-step-10-error-handling--polish)
13. [How to Talk About This in an Interview](#13-how-to-talk-about-this-in-an-interview)
14. [Quick Reference Cheat Sheet](#14-quick-reference-cheat-sheet)

---

## 1. What We're Building

A REST API for managing tasks, with the standard CRUD verbs:

| Method | Route | Purpose | Spring analog |
|--------|-------|---------|---------------|
| `GET` | `/api/tasks` | List all (paged) | `@GetMapping` |
| `GET` | `/api/tasks/{id}` | Get one | `@GetMapping("/{id}")` |
| `POST` | `/api/tasks` | Create | `@PostMapping` |
| `PUT` | `/api/tasks/{id}` | Update | `@PutMapping` |
| `DELETE` | `/api/tasks/{id}` | Delete | `@DeleteMapping` |

**Architecture (classic layered, same as Spring):**

```
HTTP request
   │
   ▼
Controller  ── validates input, returns HTTP results   (like @RestController)
   │
   ▼
Service     ── business logic, maps entity <-> DTO      (like @Service)
   │
   ▼
DbContext   ── EF Core, talks to the database           (like a Spring Data repository)
   │
   ▼
Database (SQLite for this demo — zero setup)
```

We use **SQLite** so there's nothing to install — the DB is a single file. Swapping to SQL Server later is a one-line change.

---

## 2. Step 0: Prerequisites & Project Layout

```bash
dotnet --version          # need 8.0.x or newer
```

Final structure we'll create:

```
TasksApi/                       # solution folder
├── TasksApi.sln
├── src/
│   └── TasksApi/               # the Web API project
│       ├── Program.cs
│       ├── Models/Task.cs          entity
│       ├── Dtos/TaskDtos.cs        request/response DTOs
│       ├── Data/AppDbContext.cs    EF Core context
│       ├── Services/ITaskService.cs + TaskService.cs
│       ├── Controllers/TasksController.cs
│       └── appsettings.json
└── tests/
    └── TasksApi.Tests/         # xUnit test project
```

---

## 3. Step 1: Create the Solution & Projects

```bash
# 1) Create the solution + folders
mkdir TasksApi && cd TasksApi
dotnet new sln -n TasksApi                       # like a parent Maven pom that groups modules

# 2) Create the Web API project under src/
dotnet new webapi -o src/TasksApi --use-controllers   # --use-controllers = classic controllers, not minimal APIs
dotnet sln add src/TasksApi/TasksApi.csproj

# 3) Create the test project under tests/
dotnet new xunit -o tests/TasksApi.Tests
dotnet sln add tests/TasksApi.Tests/TasksApi.Tests.csproj
dotnet add tests/TasksApi.Tests reference src/TasksApi   # tests reference the API project

# 4) Add the packages we'll need to the API project
cd src/TasksApi
dotnet add package Microsoft.EntityFrameworkCore.Sqlite       # EF Core + SQLite provider
dotnet add package Microsoft.EntityFrameworkCore.Design       # for 'dotnet ef' migrations
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer  # JWT auth
cd ../..

# 5) Add test packages
cd tests/TasksApi.Tests
dotnet add package Moq                                        # mocking
dotnet add package Microsoft.EntityFrameworkCore.InMemory     # in-memory DB for tests
cd ../..

# 6) Install the EF Core CLI tool (once per machine) for migrations
dotnet tool install --global dotnet-ef
```

> **Spring parallel:** `dotnet new webapi` ≈ Spring Initializr with Web + a starter. `dotnet add package` ≈ adding a `<dependency>` to `pom.xml`.

---

## 4. Step 2: The Entity & DbContext (EF Core)

**`src/TasksApi/Models/Task.cs`** — the entity (like a JPA `@Entity`):

```csharp
namespace TasksApi.Models;

// EF Core maps this class to a 'TaskItem' table. (Named TaskItem to avoid clashing with System.Threading.Tasks.Task.)
public class TaskItem
{
    public int Id { get; set; }                       // convention: 'Id' => primary key, auto-increment
    public string Title { get; set; } = string.Empty; // non-nullable; default avoids null warnings
    public bool IsDone { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}
```

**`src/TasksApi/Data/AppDbContext.cs`** — the EF Core context (like Spring's `EntityManager` + repositories combined):

```csharp
using Microsoft.EntityFrameworkCore;
using TasksApi.Models;

namespace TasksApi.Data;

public class AppDbContext : DbContext
{
    // DI passes options (connection string, provider) in via this constructor:
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    // Each DbSet<T> is a table you can query with LINQ (like a JpaRepository<TaskItem, int>):
    public DbSet<TaskItem> Tasks => Set<TaskItem>();

    // Optional: configure the model (constraints, indexes) — the Fluent API:
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<TaskItem>()
            .Property(t => t.Title)
            .IsRequired()
            .HasMaxLength(200);   // maps to a NOT NULL, VARCHAR(200) column
    }
}
```

> Deeper dive: `Entity_Framework_Core_Guide.md`.

---

## 5. Step 3: DTOs & Validation

**Never expose entities directly.** DTOs decouple your API contract from your database schema (same rule as Spring). `src/TasksApi/Dtos/TaskDtos.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace TasksApi.Dtos;

// What clients SEND when creating a task. DataAnnotations = validation (like Bean Validation @NotBlank):
public record CreateTaskDto(
    [Required, StringLength(200, MinimumLength = 1)] string Title
);

// What clients SEND when updating:
public record UpdateTaskDto(
    [Required, StringLength(200, MinimumLength = 1)] string Title,
    bool IsDone
);

// What we RETURN to clients (never leak CreatedAt internals if we don't want to — here we include it):
public record TaskResponseDto(int Id, string Title, bool IsDone, DateTime CreatedAt);
```

We use `record` types because DTOs are immutable data carriers — value equality, concise syntax. See `CSharp_Modern_Language_Features.md` for records/`init`.

> **Validation flow:** `[ApiController]` auto-checks `[Required]`/`[StringLength]` and returns **400 Bad Request** with error details *before* your action runs — no manual `ModelState.IsValid` check needed. (Spring's `@Valid` equivalent.)

---

## 6. Step 4: The Service Layer (DI)

Business logic lives in a service behind an interface — so the controller depends on an abstraction and tests can mock it. `src/TasksApi/Services/ITaskService.cs`:

```csharp
using TasksApi.Dtos;

namespace TasksApi.Services;

public interface ITaskService
{
    Task<IEnumerable<TaskResponseDto>> GetAllAsync(int page, int pageSize);
    Task<TaskResponseDto?> GetByIdAsync(int id);          // null => not found
    Task<TaskResponseDto> CreateAsync(CreateTaskDto dto);
    Task<bool> UpdateAsync(int id, UpdateTaskDto dto);    // false => not found
    Task<bool> DeleteAsync(int id);                       // false => not found
}
```

`src/TasksApi/Services/TaskService.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using TasksApi.Data;
using TasksApi.Dtos;
using TasksApi.Models;

namespace TasksApi.Services;

// Primary constructor (C# 12): 'db' is injected and usable throughout the class.
public class TaskService(AppDbContext db) : ITaskService
{
    public async Task<IEnumerable<TaskResponseDto>> GetAllAsync(int page, int pageSize)
    {
        return await db.Tasks
            .OrderByDescending(t => t.CreatedAt)
            .Skip((page - 1) * pageSize)      // pagination — offset
            .Take(pageSize)                   // page size — limit
            .Select(t => new TaskResponseDto(t.Id, t.Title, t.IsDone, t.CreatedAt))  // project to DTO in SQL
            .ToListAsync();                   // async DB round-trip (never block on .Result)
    }

    public async Task<TaskResponseDto?> GetByIdAsync(int id)
    {
        var t = await db.Tasks.FindAsync(id);           // FindAsync = fast PK lookup
        return t is null ? null : new TaskResponseDto(t.Id, t.Title, t.IsDone, t.CreatedAt);
    }

    public async Task<TaskResponseDto> CreateAsync(CreateTaskDto dto)
    {
        var entity = new TaskItem { Title = dto.Title };  // map DTO -> entity
        db.Tasks.Add(entity);
        await db.SaveChangesAsync();                      // INSERT; entity.Id now populated
        return new TaskResponseDto(entity.Id, entity.Title, entity.IsDone, entity.CreatedAt);
    }

    public async Task<bool> UpdateAsync(int id, UpdateTaskDto dto)
    {
        var entity = await db.Tasks.FindAsync(id);
        if (entity is null) return false;                 // signal 404 to the controller
        entity.Title = dto.Title;                         // EF tracks the change automatically
        entity.IsDone = dto.IsDone;
        await db.SaveChangesAsync();                      // UPDATE only the changed columns
        return true;
    }

    public async Task<bool> DeleteAsync(int id)
    {
        var entity = await db.Tasks.FindAsync(id);
        if (entity is null) return false;
        db.Tasks.Remove(entity);
        await db.SaveChangesAsync();                      // DELETE
        return true;
    }
}
```

> Deeper dive: `DependencyInjection_And_Middleware_Guide.md`, `CSharp_Async_Await_Concurrency.md`.

---

## 7. Step 5: The Controller (CRUD Endpoints)

`src/TasksApi/Controllers/TasksController.cs` — thin: it maps HTTP to the service and picks the right status code.

```csharp
using Microsoft.AspNetCore.Mvc;
using TasksApi.Dtos;
using TasksApi.Services;

namespace TasksApi.Controllers;

[ApiController]                       // enables auto model validation + 400s (like Spring's @RestController)
[Route("api/[controller]")]          // "[controller]" -> "tasks"; full route = /api/tasks
public class TasksController(ITaskService service) : ControllerBase
{
    // GET /api/tasks?page=1&pageSize=20
    [HttpGet]
    public async Task<ActionResult<IEnumerable<TaskResponseDto>>> GetAll(int page = 1, int pageSize = 20)
    {
        var tasks = await service.GetAllAsync(page, pageSize);
        return Ok(tasks);            // 200 with body
    }

    // GET /api/tasks/5
    [HttpGet("{id:int}")]            // ':int' route constraint -> non-numeric ids 404 automatically
    public async Task<ActionResult<TaskResponseDto>> GetById(int id)
    {
        var task = await service.GetByIdAsync(id);
        return task is null ? NotFound() : Ok(task);   // 404 or 200
    }

    // POST /api/tasks
    [HttpPost]
    public async Task<ActionResult<TaskResponseDto>> Create(CreateTaskDto dto)
    {
        var created = await service.CreateAsync(dto);
        // 201 Created + a Location header pointing at the new resource (REST best practice):
        return CreatedAtAction(nameof(GetById), new { id = created.Id }, created);
    }

    // PUT /api/tasks/5
    [HttpPut("{id:int}")]
    public async Task<IActionResult> Update(int id, UpdateTaskDto dto)
    {
        var ok = await service.UpdateAsync(id, dto);
        return ok ? NoContent() : NotFound();          // 204 on success, 404 if missing
    }

    // DELETE /api/tasks/5
    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id)
    {
        var ok = await service.DeleteAsync(id);
        return ok ? NoContent() : NotFound();
    }
}
```

> **Status-code discipline is interview gold:** 200 (OK), 201 (Created, with `Location`), 204 (No Content — successful PUT/DELETE with no body), 400 (validation), 404 (not found), 401/403 (auth). Deeper dive: `ASPNET_Core_Web_API_Guide.md`.

---

## 8. Step 6: Wire It Up in Program.cs

`src/TasksApi/Program.cs` — the composition root (like Spring's auto-config + `@Bean` registrations, but explicit):

```csharp
using Microsoft.EntityFrameworkCore;
using TasksApi.Data;
using TasksApi.Services;

var builder = WebApplication.CreateBuilder(args);

// --- REGISTER SERVICES (the DI container) ---
builder.Services.AddControllers();                          // MVC controllers
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();                           // Swagger UI (interactive API docs)

// EF Core with SQLite; connection string from appsettings.json:
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("Default")));

// Our service — Scoped = one instance per HTTP request (correct lifetime for anything using DbContext):
builder.Services.AddScoped<ITaskService, TaskService>();

var app = builder.Build();

// --- CONFIGURE THE MIDDLEWARE PIPELINE (order matters!) ---
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();                                     // browse to /swagger
}
app.UseHttpsRedirection();
app.UseAuthentication();                                    // must come BEFORE Authorization
app.UseAuthorization();
app.MapControllers();                                       // route requests to controllers

app.Run();
```

Add the connection string to **`appsettings.json`**:

```json
{
  "ConnectionStrings": {
    "Default": "Data Source=tasks.db"
  },
  "Logging": { "LogLevel": { "Default": "Information" } }
}
```

---

## 9. Step 7: Migrations & Run It

**Migrations** are versioned schema changes — EF generates SQL from your model (like Flyway/Liquibase, but code-first from your entities).

```bash
cd src/TasksApi

# 1) Create the first migration (EF diffs your model against an empty DB):
dotnet ef migrations add InitialCreate     # generates Migrations/*.cs

# 2) Apply it — creates tasks.db with the TaskItem table:
dotnet ef database update

# 3) Run the API:
dotnet run
```

Now open **`https://localhost:<port>/swagger`** and exercise the endpoints, or use curl:

```bash
# Create a task -> 201 Created
curl -k -X POST https://localhost:7001/api/tasks \
     -H "Content-Type: application/json" \
     -d '{"title":"Learn .NET"}'

# List tasks -> 200
curl -k https://localhost:7001/api/tasks

# Update -> 204
curl -k -X PUT https://localhost:7001/api/tasks/1 \
     -H "Content-Type: application/json" \
     -d '{"title":"Learn .NET well","isDone":true}'

# Delete -> 204
curl -k -X DELETE https://localhost:7001/api/tasks/1
```

> **When the schema changes** (add a property), repeat: `dotnet ef migrations add <Name>` then `dotnet ef database update`. Never edit an applied migration — add a new one.

---

## 10. Step 8: Add JWT Authentication

Protect the write endpoints so only authenticated callers can mutate data. This is a condensed version of `ASPNET_Core_Security_JWT_Identity.md`.

**In `Program.cs`, register JWT bearer auth** (before `var app = builder.Build();`):

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var jwtKey = builder.Configuration["Jwt:Key"]!;   // load secret from config (not hard-coded)

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,                 // reject expired tokens
            ValidateIssuerSigningKey = true,         // verify the signature
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtKey))
        };
    });
builder.Services.AddAuthorization();
```

**Protect the controller** — add `[Authorize]` to require a valid token:

```csharp
using Microsoft.AspNetCore.Authorization;

[ApiController]
[Route("api/[controller]")]
[Authorize]                       // ALL actions now require a valid JWT (like Spring @PreAuthorize)
public class TasksController(ITaskService service) : ControllerBase
{
    [HttpGet]
    [AllowAnonymous]              // ...except this one, which stays public
    public async Task<ActionResult<IEnumerable<TaskResponseDto>>> GetAll(int page = 1, int pageSize = 20)
    // ...
}
```

Add secrets to `appsettings.json` (in production use environment variables / user-secrets, **never commit real keys**):

```json
"Jwt": {
  "Key": "this-is-a-demo-key-min-32-chars-long-change-me!!",
  "Issuer": "TasksApi",
  "Audience": "TasksApiClients"
}
```

> **Interview point:** JWT is *stateless* auth — the server validates the signature and claims on every request without a session store. Deeper dive: `ASPNET_Core_Security_JWT_Identity.md`.

---

## 11. Step 9: Write Tests

Two flavors interviewers expect: a **unit test** (service logic against an in-memory DB) and a **mock-based test** (controller against a mocked service).

**`tests/TasksApi.Tests/TaskServiceTests.cs`** — unit test with EF InMemory:

```csharp
using Microsoft.EntityFrameworkCore;
using TasksApi.Data;
using TasksApi.Dtos;
using TasksApi.Services;
using Xunit;

public class TaskServiceTests
{
    // Helper: a fresh in-memory DbContext per test (isolated, no real DB needed):
    private static AppDbContext NewDb()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(Guid.NewGuid().ToString())   // unique name => isolated per test
            .Options;
        return new AppDbContext(options);
    }

    [Fact]                                              // xUnit's @Test
    public async Task CreateAsync_AddsTask_AndReturnsIt()
    {
        // Arrange
        using var db = NewDb();
        var service = new TaskService(db);

        // Act
        var result = await service.CreateAsync(new CreateTaskDto("Write tests"));

        // Assert
        Assert.True(result.Id > 0);                     // got a generated id
        Assert.Equal("Write tests", result.Title);
        Assert.False(result.IsDone);
        Assert.Equal(1, await db.Tasks.CountAsync());   // persisted exactly one row
    }

    [Fact]
    public async Task GetByIdAsync_ReturnsNull_WhenMissing()
    {
        using var db = NewDb();
        var service = new TaskService(db);
        var result = await service.GetByIdAsync(999);
        Assert.Null(result);
    }
}
```

**`tests/TasksApi.Tests/TasksControllerTests.cs`** — mock the service with Moq:

```csharp
using Microsoft.AspNetCore.Mvc;
using Moq;
using TasksApi.Controllers;
using TasksApi.Dtos;
using TasksApi.Services;
using Xunit;

public class TasksControllerTests
{
    [Fact]
    public async Task GetById_Returns404_WhenServiceReturnsNull()
    {
        // Arrange — mock the service to return null (not found):
        var mock = new Mock<ITaskService>();
        mock.Setup(s => s.GetByIdAsync(42)).ReturnsAsync((TaskResponseDto?)null);
        var controller = new TasksController(mock.Object);

        // Act
        var result = await controller.GetById(42);

        // Assert — the ActionResult's underlying result is a NotFound (404):
        Assert.IsType<NotFoundResult>(result.Result);
    }
}
```

Run everything:

```bash
dotnet test           # discovers and runs all [Fact]/[Theory] tests in the solution
```

> Deeper dive: `DotNet_Testing_xUnit_Moq_Guide.md`.

---

## 12. Step 10: Error Handling & Polish

Add global exception handling so unhandled errors return clean JSON, not stack traces. A simple middleware in `Program.cs`:

```csharp
// Built-in problem-details handler (RFC 7807) — returns structured error JSON:
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/error");        // route unhandled exceptions here
}
app.Map("/error", (HttpContext ctx) =>
    Results.Problem(title: "An unexpected error occurred.", statusCode: 500));
```

**Production checklist for this API:**

- [ ] Connection string & JWT key come from **environment variables / user-secrets**, not `appsettings.json`.
- [ ] `[Authorize]` on mutating endpoints; `[AllowAnonymous]` only where intended.
- [ ] DTOs everywhere — entities never cross the HTTP boundary.
- [ ] `async`/`await` all the way down; no `.Result`/`.Wait()`.
- [ ] Pagination on list endpoints (never return unbounded result sets).
- [ ] `ILogger` injected and used for key operations (see `ASPNET_Core_Web_API_Guide.md` §15).
- [ ] Tests green: `dotnet test`.
- [ ] Swagger reachable in dev for manual verification.

---

## 13. How to Talk About This in an Interview

When asked *"tell me about something you built,"* narrate it in this order (2 minutes):

1. **What & why:** "A REST API for managing tasks — a full CRUD backend in ASP.NET Core 8 with EF Core."
2. **Architecture:** "Layered — thin controllers, a service layer for business logic behind an interface, and EF Core for data access. Controllers depend on the `ITaskService` abstraction, registered as `Scoped` in the DI container."
3. **Key decisions:** "DTOs decouple the API contract from entities. `[ApiController]` gives automatic model validation. I return proper status codes — 201 with a `Location` header on create, 204 on update/delete, 404 when missing."
4. **Cross-cutting concerns:** "JWT bearer auth for stateless security, global exception handling returning problem-details JSON, and pagination on the list endpoint."
5. **Testing:** "Unit tests on the service with EF Core's in-memory provider, plus controller tests using Moq to isolate the service."
6. **What I'd add next:** "AutoMapper for DTO mapping, FluentValidation for complex rules, structured logging with Serilog, and containerizing with Docker."

That last point shows growth awareness — interviewers love it. (See `DotNet_Deployment_And_Ops.md` for the Docker/prod side.)

---

## 14. Quick Reference Cheat Sheet

```
SCAFFOLD:
  dotnet new sln -n X
  dotnet new webapi -o src/X --use-controllers
  dotnet new xunit -o tests/X.Tests
  dotnet sln add <proj>
  dotnet add <proj> reference <other>
  dotnet add package <name>

EF CORE PACKAGES:  EntityFrameworkCore.Sqlite, .Design  (+ dotnet tool install -g dotnet-ef)
TEST PACKAGES:     Moq, EntityFrameworkCore.InMemory

LAYERS:
  Controller (HTTP + status codes)  ->  Service (logic, DTO<->entity)  ->  DbContext (EF)

DI LIFETIMES (Program.cs):
  AddScoped   -> per request   (use for anything touching DbContext)  <-- default choice
  AddTransient-> per resolve
  AddSingleton-> one for app lifetime

CONTROLLER STATUS CODES:
  Ok(x)               200
  CreatedAtAction(..) 201 + Location header
  NoContent()         204   (PUT/DELETE success)
  BadRequest()        400   (auto from [ApiController] on invalid model)
  NotFound()          404
  Unauthorized()      401   Forbid() 403

MIGRATIONS:
  dotnet ef migrations add <Name>
  dotnet ef database update
  (never edit an applied migration — add a new one)

RUN & TEST:
  dotnet run           (browse /swagger)
  dotnet test

PIPELINE ORDER (Program.cs):
  UseHttpsRedirection -> UseAuthentication -> UseAuthorization -> MapControllers

GOLDEN RULES:
  1. Thin controllers, logic in services behind interfaces.
  2. DTOs at the boundary — never expose entities.
  3. async all the way; no .Result/.Wait().
  4. Correct status codes: 201+Location, 204, 400, 404, 401/403.
  5. Secrets from config/env, never committed.
  6. Rebuild this project from memory before interviews.
```

---

*This capstone combines: `ASPNET_Core_Web_API_Guide.md`, `Entity_Framework_Core_Guide.md`, `DependencyInjection_And_Middleware_Guide.md`, `ASPNET_Core_Security_JWT_Identity.md`, `DotNet_Testing_xUnit_Moq_Guide.md`. Deploy it with `DotNet_Deployment_And_Ops.md`.*

*Last Updated: 2026-07-09*
