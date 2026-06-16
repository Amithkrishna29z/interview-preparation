# Dependency Injection & Middleware in ASP.NET Core — A Spring Developer's Guide

## Overview

If you come from Spring, two things will feel both familiar and slightly off when you move to ASP.NET Core:

1. **Dependency Injection (DI):** Spring made DI the centerpiece of the whole framework with `@Component`, `@Autowired`, and the `ApplicationContext`. ASP.NET Core also puts DI at its core — but the container is *built into the framework itself* (no extra library, no big XML, no component-scanning by default). You register services explicitly in code.

2. **The Request Pipeline (Middleware):** Spring uses Servlet Filters and `HandlerInterceptor`s to wrap requests. ASP.NET Core replaces all of that with a single, ordered **middleware pipeline** — a literal chain of responsibility where each piece can run code before and after the next one.

This guide maps both concepts directly onto what you already know from Spring, so you can be productive (and interview-ready) fast. Everything uses **modern ASP.NET Core (.NET 8+)** with the minimal hosting model (`WebApplication.CreateBuilder`).

---

## Table of Contents

- [Spring → ASP.NET Core DI/Pipeline Mapping](#spring--aspnet-core-dipipeline-mapping)
- [PART A — Dependency Injection](#part-a--dependency-injection)
  - [A1. What is DI / IoC?](#a1-what-is-di--ioc)
  - [A2. The Built-In Container vs Spring's ApplicationContext](#a2-the-built-in-container-vs-springs-applicationcontext)
  - [A3. Registering Services: Singleton, Scoped, Transient](#a3-registering-services-singleton-scoped-transient)
  - [A4. Constructor Injection & Injecting Interfaces](#a4-constructor-injection--injecting-interfaces)
  - [A5. IServiceProvider (the Service Locator)](#a5-iserviceprovider-the-service-locator)
  - [A6. The Captive Dependency Trap](#a6-the-captive-dependency-trap)
  - [A7. Multiple Implementations & IEnumerable<T>](#a7-multiple-implementations--ienumerablet)
  - [A8. Keyed Services (.NET 8)](#a8-keyed-services-net-8)
  - [A9. Factory Registration](#a9-factory-registration)
  - [A10. The Options Pattern (IOptions<T>)](#a10-the-options-pattern-ioptionst)
  - [A11. Disposal of Services](#a11-disposal-of-services)
  - [A12. Third-Party Containers (Autofac)](#a12-third-party-containers-autofac)
- [PART B — Middleware & the Request Pipeline](#part-b--middleware--the-request-pipeline)
  - [B1. What is Middleware?](#b1-what-is-middleware)
  - [B2. Use, Run, Map and the `next` Delegate](#b2-use-run-map-and-the-next-delegate)
  - [B3. Built-In Middleware](#b3-built-in-middleware)
  - [B4. Order Matters](#b4-order-matters)
  - [B5. Writing Custom Middleware](#b5-writing-custom-middleware)
  - [B6. Global Exception Handling (vs @ControllerAdvice) & IExceptionHandler](#b6-global-exception-handling-vs-controlleradvice--iexceptionhandler)
  - [B7. Filters vs Middleware (the real Spring AOP analog)](#b7-filters-vs-middleware-the-real-spring-aop-analog)
  - [B8. Hosted Services / BackgroundService](#b8-hosted-services--backgroundservice)
- [Common Interview Questions](#common-interview-questions)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Spring → ASP.NET Core DI/Pipeline Mapping

| Spring / Java | ASP.NET Core (.NET 8+) | Notes |
|---|---|---|
| `@Component` / `@Service` / `@Repository` | `AddScoped` / `AddTransient` / `AddSingleton` (you choose the lifetime) | .NET has no "stereotype" annotations; the lifetime is chosen at registration, not on the class. |
| `@Autowired` (field/constructor) | **Constructor injection** (the standard) | Field injection effectively doesn't exist; constructor injection is idiomatic. |
| `ApplicationContext` / `BeanFactory` | `IServiceProvider` | The container/root resolver. |
| `@Bean` method in `@Configuration` | `services.AddX(...)` registration in `Program.cs` | Explicit registration replaces `@Bean` factory methods. |
| Bean scope `singleton` (default) | `AddSingleton` | One instance for the whole app lifetime. |
| Bean scope `prototype` | `AddTransient` | New instance every time it's requested. |
| Bean scope `request` (web) | `AddScoped` | One instance per HTTP request. **Scoped ≈ per-request.** |
| Component scanning (`@ComponentScan`) | *(none by default)* — you register explicitly | Some libraries add extension methods (`AddControllers()`, `AddDbContext()`) that batch-register. |
| `@Qualifier("name")` | **Keyed services** `AddKeyedScoped("name", ...)` (.NET 8) | Resolve a specific named implementation. |
| Injecting `List<MyInterface>` (all beans of a type) | Injecting `IEnumerable<MyInterface>` | All registered implementations of an interface. |
| `@Value` / `@ConfigurationProperties` | **Options pattern** `IOptions<MyOptions>` | Strongly-typed config binding. |
| `FactoryBean` / `@Bean` factory method | **Factory registration** `AddScoped(sp => new X(...))` | Build a service with custom logic. |
| `@PostConstruct` | Constructor work, or `IHostedService.StartAsync`, or factory lambda | No direct attribute; do init in the constructor or a hosted service. |
| `@PreDestroy` / `DisposableBean` | `IDisposable` / `IAsyncDisposable` — container disposes for you | The container disposes services it created. |
| Servlet `Filter` (`doFilter` + chain) | **Middleware** (`Use` + `next`) | The closest structural match: wraps the whole request. |
| `HandlerInterceptor` (`preHandle`/`postHandle`) | **Action Filters** (`OnActionExecuting`/`Executed`) | Runs around controller actions, has access to MVC context. |
| `@ControllerAdvice` + `@ExceptionHandler` | **Exception-handling middleware** or `IExceptionHandler` (.NET 8) or **Exception Filters** | Centralized error handling. |
| AOP `@Aspect` / `@Around` (cross-cutting) | **Middleware** (pipeline-wide) or **Filters** (MVC-scoped) or **Decorators** | No general-purpose AOP weaving; you pick the right tool per scope. |
| `@Scheduled` / background jobs | `BackgroundService` / `IHostedService` | Long-running background work. |

---

# PART A — Dependency Injection

## A1. What is DI / IoC?

**Inversion of Control (IoC)** means a class does **not** create its own dependencies — something else (the container) creates them and hands them in. **Dependency Injection** is the most common way to achieve IoC: dependencies are *injected*, usually through the constructor.

This is identical in spirit to Spring. The difference is purely mechanical: where Spring scans for `@Component` and auto-wires, ASP.NET Core asks you to register each service explicitly.

**Think of it like...** a restaurant kitchen. Without DI, the chef (your class) grows the vegetables, raises the chickens, and mines the salt himself — tightly coupled and untestable. With DI, suppliers deliver pre-made ingredients (dependencies) to the chef. The chef just declares "I need vegetables, chicken, salt" and the kitchen manager (the container) makes sure they show up. You can swap a supplier (a mock in tests) without telling the chef.

```csharp
// WITHOUT DI — tightly coupled, hard to test (the chef grows his own food)
public class OrderService
{
    private readonly EmailSender _email = new EmailSender(); // hard-coded dependency
}

// WITH DI — the dependency is declared, not created
// Spring equivalent: @Service class OrderService { @Autowired EmailSender email; }
public class OrderService
{
    private readonly IEmailSender _email; // depend on an INTERFACE, not a concrete class

    // Constructor injection — the container supplies IEmailSender
    public OrderService(IEmailSender email) => _email = email;
}
```

---

## A2. The Built-In Container vs Spring's ApplicationContext

Spring's `ApplicationContext` is huge and does a lot (AOP, events, profiles, scanning). ASP.NET Core's container (`Microsoft.Extensions.DependencyInjection`) is deliberately **minimal**: it does constructor injection and lifetime management, and that's about it. For 95% of apps, that's all you need.

You configure it in `Program.cs` via the `WebApplicationBuilder`:

```csharp
var builder = WebApplication.CreateBuilder(args);

// builder.Services is the IServiceCollection — your "bean definitions"
// This is where every @Bean / @Component registration lives.
builder.Services.AddScoped<IEmailSender, SmtpEmailSender>();
builder.Services.AddControllers(); // an extension method that batch-registers MVC services

var app = builder.Build();   // builds the IServiceProvider (the ApplicationContext)
// ... configure middleware pipeline here ...
app.Run();
```

- `IServiceCollection` (`builder.Services`) = the list of registrations (like your set of `@Bean` definitions).
- `IServiceProvider` (built by `app.Build()`) = the live container that resolves instances (like `ApplicationContext.getBean()`).

---

## A3. Registering Services: Singleton, Scoped, Transient

Three lifetimes. This is the single most important DI concept to nail for an interview.

| Method | Lifetime | Spring equivalent | Use when... |
|---|---|---|---|
| `AddSingleton<I, T>()` | One instance for the **entire app** | `singleton` scope (default) | Stateless services, caches, config, expensive-to-create shared objects. |
| `AddScoped<I, T>()` | One instance **per HTTP request** | `request` scope | Per-request state, `DbContext`/EF Core, "unit of work". **The most common choice.** |
| `AddTransient<I, T>()` | A **new instance every time** it's injected | `prototype` scope | Lightweight, stateless services where a fresh instance is cheap and safe. |

**Think of it like...** a hotel:
- **Singleton** = the building itself. There's exactly one, shared by everyone, for as long as the hotel exists.
- **Scoped** = your room key card. You get one for the duration of *your stay* (one request). Everyone in your party (same request) shares it; the next guest gets a different one.
- **Transient** = a disposable paper cup from the dispenser. Every time you want a drink, you grab a brand-new one.

```csharp
// SINGLETON — one shared instance forever (like a default Spring @Service bean)
builder.Services.AddSingleton<IClock, SystemClock>();

// SCOPED — one per request; THE go-to for EF Core DbContext and repositories
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddDbContext<AppDbContext>();   // AddDbContext registers DbContext as Scoped

// TRANSIENT — brand new every resolve (like Spring prototype)
builder.Services.AddTransient<IPriceCalculator, PriceCalculator>();
```

> **Key rule:** Within a single HTTP request, every place that asks for a Scoped service gets the *same* instance. Across two different requests, they're different instances.

---

## A4. Constructor Injection & Injecting Interfaces

Constructor injection is **the** way in .NET. Unlike Spring, there's no idiomatic field injection — and you almost always inject an interface, not a concrete type.

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderRepository _repo;
    private readonly IEmailSender _email;

    // The container reads this constructor's parameters and supplies them.
    // Spring equivalent: a constructor annotated (implicitly) with @Autowired.
    public OrdersController(IOrderRepository repo, IEmailSender email)
    {
        _repo = repo;
        _email = email;
    }

    [HttpPost]
    public async Task<IActionResult> Create(OrderDto dto)
    {
        var order = await _repo.AddAsync(dto);
        await _email.SendConfirmationAsync(order);
        return Ok(order);
    }
}
```

A common modern style uses a **primary constructor** (C# 12) to cut boilerplate:

```csharp
// Primary constructor — parameters are in scope across the class body
public class OrdersController(IOrderRepository repo, IEmailSender email) : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<IActionResult> Get(int id) => Ok(await repo.GetAsync(id));
}
```

---

## A5. IServiceProvider (the Service Locator)

`IServiceProvider` is the container itself — equivalent to calling `ApplicationContext.getBean()`. You can inject it and resolve services manually, **but prefer constructor injection**; using `IServiceProvider` directly is the "service locator anti-pattern" and hides dependencies.

The one legitimate, common use: creating a **scope** inside a Singleton (e.g., a background service) so you can safely use Scoped services.

```csharp
public class ReportRunner(IServiceProvider provider) // provider = the container
{
    public async Task RunAsync()
    {
        // Manually create a scope = simulate "one request" so Scoped services are valid.
        // Spring analog: programmatically obtaining a request-scoped bean outside a request.
        using IServiceScope scope = provider.CreateScope();
        var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
        await repo.DoWorkAsync();
    } // scope disposed here -> Scoped services disposed
}

// GetService<T>()        -> returns null if not registered
// GetRequiredService<T>() -> throws if not registered (prefer this; fails loudly)
```

---

## A6. The Captive Dependency Trap

**This is the classic .NET DI interview question.** A **captive dependency** happens when a *longer-lived* service holds a reference to a *shorter-lived* one — most commonly injecting a **Scoped** (or Transient) service into a **Singleton**.

Because the Singleton is created once and lives forever, the Scoped service it captured *also* lives forever — defeating "per-request" semantics and frequently causing bugs (e.g., a single `DbContext` shared across all requests and threads → crashes).

**Think of it like...** hiring a permanent employee (Singleton) and handing them a single day-pass visitor badge (Scoped). The badge was meant to expire at end of day, but now it's stuck to a permanent employee forever — it never expires, and security gets confused.

```csharp
// ❌ CAPTIVE DEPENDENCY — Scoped repo trapped inside a Singleton
builder.Services.AddSingleton<ICache, MemoryCache>();
public class MemoryCache(IOrderRepository repo) : ICache { } // repo is Scoped -> trapped!

// In .NET, the default container CATCHES this at startup (in Development) and throws:
// "Cannot consume scoped service 'IOrderRepository' from singleton 'ICache'."
// (Validation is on by default in the Development environment.)
```

**Fixes:**
```csharp
// ✅ Option 1: make the consumer Scoped too (matching lifetimes)
builder.Services.AddScoped<ICache, MemoryCache>();

// ✅ Option 2: inject IServiceProvider / IServiceScopeFactory and create a scope per use
public class MemoryCache(IServiceScopeFactory scopeFactory) : ICache
{
    public void Refresh()
    {
        using var scope = scopeFactory.CreateScope();
        var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
        // use repo within this short-lived scope
    }
}
```

> Rule of thumb: a service may only depend on something with an **equal or longer** lifetime. Singleton → Singleton only. Scoped → Scoped/Singleton. Transient → anything.

---

## A7. Multiple Implementations & IEnumerable<T>

Register the same interface multiple times, then inject them all as `IEnumerable<T>`. This is the .NET equivalent of injecting `List<MyInterface>` in Spring (all beans of that type).

```csharp
public interface INotifier { Task NotifyAsync(string msg); }

// Register THREE implementations of the same interface
builder.Services.AddScoped<INotifier, EmailNotifier>();
builder.Services.AddScoped<INotifier, SmsNotifier>();
builder.Services.AddScoped<INotifier, PushNotifier>();

// Inject ALL of them at once (great for the strategy / fan-out pattern)
public class AlertService(IEnumerable<INotifier> notifiers)
{
    public async Task AlertAllAsync(string msg)
    {
        foreach (var n in notifiers) // runs Email, Sms, AND Push
            await n.NotifyAsync(msg);
    }
}

// NOTE: if you inject a single INotifier, you get the LAST one registered (PushNotifier).
```

---

## A8. Keyed Services (.NET 8)

Before .NET 8, there was no built-in equivalent of Spring's `@Qualifier`. **.NET 8 added keyed services** — register implementations under a key and resolve a specific one.

```csharp
// Register by key — like @Qualifier("fast") / @Qualifier("durable")
builder.Services.AddKeyedScoped<INotifier, EmailNotifier>("durable");
builder.Services.AddKeyedScoped<INotifier, SmsNotifier>("fast");

// Resolve a specific keyed implementation via the [FromKeyedServices] attribute
public class OrderService(
    [FromKeyedServices("fast")] INotifier notifier)   // gets SmsNotifier
{
    // Spring equivalent: @Autowired @Qualifier("fast") Notifier notifier;
}

// Or resolve manually from the provider:
// var n = serviceProvider.GetRequiredKeyedService<INotifier>("durable");
```

---

## A9. Factory Registration

When constructing a service needs logic (read config, pick an implementation at runtime, pass non-DI args), register a **factory lambda**. This is Spring's `FactoryBean` / `@Bean` factory method.

```csharp
// The lambda receives the IServiceProvider (sp) so it can resolve other services.
builder.Services.AddScoped<IPaymentGateway>(sp =>
{
    var config = sp.GetRequiredService<IConfiguration>();
    var useSandbox = config.GetValue<bool>("Payments:Sandbox");

    // Choose the implementation at runtime — something attribute-based DI can't do.
    return useSandbox
        ? new SandboxGateway()
        : new StripeGateway(config["Payments:ApiKey"]!);
});
```

---

## A10. The Options Pattern (IOptions<T>)

The Options pattern binds a config section to a strongly-typed class and injects it — the equivalent of Spring's `@ConfigurationProperties`.

```csharp
// 1) The POCO — mirrors a section of appsettings.json
public class SmtpOptions
{
    public string Host { get; set; } = "";
    public int Port { get; set; }
}
```
```jsonc
// appsettings.json
{
  "Smtp": { "Host": "mail.example.com", "Port": 587 }
}
```
```csharp
// 2) Bind the "Smtp" section to SmtpOptions (like @ConfigurationProperties("smtp"))
builder.Services.Configure<SmtpOptions>(builder.Configuration.GetSection("Smtp"));

// 3) Inject IOptions<SmtpOptions> and read .Value
public class SmtpEmailSender(IOptions<SmtpOptions> options) : IEmailSender
{
    private readonly SmtpOptions _opts = options.Value; // the bound config object
    // _opts.Host, _opts.Port
}
```

> Variants: `IOptionsSnapshot<T>` (re-reads per request — Scoped), `IOptionsMonitor<T>` (live reload + change notifications, safe in Singletons).

---

## A11. Disposal of Services

If a service implements `IDisposable` (or `IAsyncDisposable`), **the container disposes it for you** — you do not call `Dispose()` yourself. This is the .NET equivalent of `@PreDestroy` / `DisposableBean`.

```csharp
public class FileLogger : IDisposable
{
    private readonly StreamWriter _writer = new("log.txt");
    public void Dispose() => _writer.Dispose(); // called automatically by the container
}
```

- **Scoped/Transient** services that the container created are disposed when their **scope** ends (i.e., at the end of the HTTP request).
- **Singleton** services are disposed when the **app shuts down**.
- ⚠️ **Caveat:** if you register an *instance you created yourself* (`AddSingleton(new Thing())`), the container will still dispose it — but for objects you `new` up *outside* of DI, you own disposal. Also, Transients that you resolve and never tie to a scope can leak; prefer scopes.

---

## A12. Third-Party Containers (Autofac)

Spring's container is the only game in town for Spring. In .NET, the built-in container is intentionally simple, so libraries like **Autofac**, **Scrutor** (for assembly scanning/decoration), or **Lamar** exist to add features:

- **Assembly scanning** (auto-register by convention, like `@ComponentScan`).
- **Property injection**, **decorators**, **interceptors** (AOP-style).
- Advanced conditional/registration rules.

```csharp
// Example: swapping in Autofac as the container factory
builder.Host.UseServiceProviderFactory(new AutofacServiceProviderFactory());
```

**Why it's usually unnecessary:** the built-in container plus `IEnumerable<T>`, keyed services (.NET 8), and factory registration cover the vast majority of needs. Reach for a third-party container only when you genuinely need convention-based scanning or decoration at scale. For a junior role, knowing the built-in container well matters far more.

---

# PART B — Middleware & the Request Pipeline

## B1. What is Middleware?

Middleware is software assembled into a **pipeline**: each component receives the `HttpContext`, can run code, then optionally calls the **next** middleware, then can run more code on the way back out. It's the literal **Chain of Responsibility** pattern.

This replaces both **Servlet Filters** *and* (largely) **Spring `HandlerInterceptor`s**. Each piece of middleware wraps the request like a layer of an onion.

**Think of it like...** airport security lanes. Your request (passenger) passes through a sequence of stations: ticket check → baggage scan → metal detector → gate. Each station can either let you proceed to the next or stop you and send you back (short-circuit). On the way back (the response), some stations stamp your boarding pass. The *order* of stations is fixed and meaningful.

```
Request  ──►  [Logging] ──► [Auth] ──► [Routing] ──► [Endpoint]
Response ◄──  [Logging] ◄── [Auth] ◄── [Routing] ◄──────┘
            (each middleware sees the request going in AND the response coming out)
```

---

## B2. Use, Run, Map and the `next` Delegate

You build the pipeline in `Program.cs` after `app = builder.Build()`.

```csharp
var app = builder.Build();

// app.Use — adds middleware that CAN call next() to continue the chain.
// 'next' is the delegate to the rest of the pipeline (like Servlet's chain.doFilter()).
app.Use(async (context, next) =>
{
    Console.WriteLine("Before next");  // runs on the way IN
    await next(context);               // hand off to the next middleware
    Console.WriteLine("After next");   // runs on the way OUT (response side)
});

// app.Run — a TERMINAL middleware. It has no 'next'; the pipeline ends here.
// Nothing registered after a Run() will execute.
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello from the end of the pipeline");
});

// app.Map — branches the pipeline based on the request path.
app.Map("/admin", adminApp =>
{
    adminApp.Run(async ctx => await ctx.Response.WriteAsync("Admin branch"));
});

app.Run();
```

- **`Use`** = a filter that participates and continues (the common case). Call `next` to proceed; *don't* call it to **short-circuit**.
- **`Run`** = the end of the line; produces a response and stops.
- **`Map`** = split into a sub-pipeline for a path prefix.
- **The `next` delegate** is exactly Spring's `chain.doFilter(req, res)` — call it to continue, omit it to stop.

---

## B3. Built-In Middleware

ASP.NET Core ships with ready-made middleware. A typical pipeline:

```csharp
var app = builder.Build();

app.UseExceptionHandler("/error");   // catch unhandled exceptions (must be near the TOP)
app.UseHttpsRedirection();           // redirect http:// -> https://
app.UseStaticFiles();                // serve wwwroot files (css/js/images)
app.UseRouting();                    // match the request to an endpoint
app.UseCors("MyPolicy");             // cross-origin rules (after routing, before auth)
app.UseAuthentication();             // WHO are you? (parse the token/cookie) — sets User
app.UseAuthorization();              // are you ALLOWED? (check roles/policies)
app.MapControllers();                // terminal: dispatch to the matched controller action

app.Run();
```

| Middleware | Purpose | Spring-ish analog |
|---|---|---|
| `UseExceptionHandler` | Global error handling | `@ControllerAdvice` / error filter |
| `UseHttpsRedirection` | Force HTTPS | A redirect filter |
| `UseStaticFiles` | Serve files from `wwwroot` | Spring's resource handler |
| `UseRouting` / `MapControllers` | URL → endpoint matching | DispatcherServlet routing |
| `UseCors` | Cross-origin resource sharing | `@CrossOrigin` / CORS filter |
| `UseAuthentication` | Identify the caller | Spring Security auth filter |
| `UseAuthorization` | Enforce permissions | Spring Security `@PreAuthorize` |

---

## B4. Order Matters

The order you register middleware **is** the order it runs — and getting it wrong silently breaks things. The cardinal rule:

> **`UseAuthentication()` must come before `UseAuthorization()`** — you can't check *whether someone is allowed* before figuring out *who they are*.

```csharp
// ✅ CORRECT
app.UseAuthentication(); // first: establish identity (HttpContext.User is populated)
app.UseAuthorization();  // then: enforce policies using that identity

// ❌ WRONG: authorization runs against an empty/anonymous user -> everything 401/403
app.UseAuthorization();
app.UseAuthentication();
```

Other ordering rules: exception handling goes near the **top** (so it wraps everything below); `UseRouting` must come before `UseAuthorization` (auth needs the matched endpoint's metadata); `UseCors` goes between routing and auth.

---

## B5. Writing Custom Middleware

**Inline** (quick, for small logic):

```csharp
// A simple request-timing middleware (cross-cutting concern, like an @Around aspect)
app.Use(async (context, next) =>
{
    var sw = Stopwatch.StartNew();
    await next(context);                 // let the rest of the pipeline run
    sw.Stop();
    Console.WriteLine($"{context.Request.Path} took {sw.ElapsedMilliseconds}ms");
});
```

**Class-based** (reusable, testable — the standard convention). The framework calls a method named **`InvokeAsync`** and injects the `RequestDelegate next` via the constructor:

```csharp
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;          // the rest of the pipeline
    private readonly ILogger<RequestTimingMiddleware> _logger;

    // Constructor: RequestDelegate is injected by the framework; other params come from DI.
    public RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    // Convention: the method MUST be named InvokeAsync (or Invoke) and take HttpContext.
    // You can add extra params here too — they're resolved PER-REQUEST from DI (scoped-safe).
    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        await _next(context);            // == chain.doFilter() in a Servlet Filter
        sw.Stop();
        _logger.LogInformation("{Path} took {Ms}ms", context.Request.Path, sw.ElapsedMilliseconds);
    }
}

// Register it in the pipeline (order still matters):
app.UseMiddleware<RequestTimingMiddleware>();
```

> ⚠️ Lifetime note: a class-based middleware is effectively a **Singleton** (constructed once). So inject Scoped services as a **parameter of `InvokeAsync`**, not via the constructor — otherwise you create a captive dependency (see A6).

---

## B6. Global Exception Handling (vs @ControllerAdvice) & IExceptionHandler

In Spring you'd write a `@ControllerAdvice` with `@ExceptionHandler` methods. In ASP.NET Core there are two idiomatic approaches.

**Approach 1 — Exception-handling middleware** (catch-all, pipeline-wide):

```csharp
public class ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> log)
{
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await next(context);   // run everything downstream
        }
        catch (Exception ex)        // catches anything that bubbles up
        {
            log.LogError(ex, "Unhandled exception");
            context.Response.StatusCode = 500;
            context.Response.ContentType = "application/json";
            // Return a clean, consistent error body (like a @ControllerAdvice response)
            await context.Response.WriteAsJsonAsync(new { error = "Something went wrong" });
        }
    }
}
// Register near the TOP so it wraps the whole pipeline:
app.UseMiddleware<ExceptionHandlingMiddleware>();
```

**Approach 2 — `IExceptionHandler` (.NET 8, the modern, cleaner way):**

```csharp
// Implement IExceptionHandler — one focused class per concern, like a typed @ExceptionHandler.
public class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext, Exception exception, CancellationToken ct)
    {
        logger.LogError(exception, "Unhandled exception");

        // ProblemDetails is the RFC 7807 standard error shape (built into ASP.NET Core)
        var problem = new ProblemDetails
        {
            Status = exception switch
            {
                NotFoundException => StatusCodes.Status404NotFound,
                _ => StatusCodes.Status500InternalServerError
            },
            Title = "An error occurred"
        };
        httpContext.Response.StatusCode = problem.Status!.Value;
        await httpContext.Response.WriteAsJsonAsync(problem, ct);
        return true; // true = "I handled it"; false = let the next handler try
    }
}

// Wire it up:
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();
// ...
app.UseExceptionHandler(); // invokes registered IExceptionHandler(s)
```

---

## B7. Filters vs Middleware (the real Spring AOP analog)

Both middleware and filters handle cross-cutting concerns, but they operate at **different layers**:

- **Middleware** sits in the raw HTTP pipeline. It runs for *every* request (even non-MVC ones) and has **no knowledge of controllers, actions, model binding, or `[Authorize]` attributes**. Best for app-wide concerns: logging, HTTPS, CORS, global error handling, response compression.

- **Filters** run *inside* MVC, around controller actions. They have rich context: the action arguments, the `ModelState`, the action result, route data. This makes filters the **closest analog to Spring AOP / `HandlerInterceptor`** for controller-level cross-cutting concerns.

**Think of it like...** middleware is the *building's* security (everyone entering the building passes through), while filters are the security *on a specific office floor* (only people headed to that floor, and the guard knows which meeting room you're going to).

Filter types (each maps to a Spring idea):

```csharp
// ACTION FILTER — runs before/after an action (like HandlerInterceptor preHandle/postHandle)
public class LoggingActionFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
        => Console.WriteLine($"Calling {context.ActionDescriptor.DisplayName}");
    public void OnActionExecuted(ActionExecutedContext context)
        => Console.WriteLine("Action done");
}

// EXCEPTION FILTER — handles exceptions from MVC actions (like a scoped @ExceptionHandler)
public class MyExceptionFilter : IExceptionFilter
{
    public void OnException(ExceptionContext context)
    {
        context.Result = new ObjectResult(new { error = context.Exception.Message })
            { StatusCode = 500 };
        context.ExceptionHandled = true;
    }
}

// AUTHORIZATION FILTER — runs first, gates access (like @PreAuthorize)
// [Authorize] itself is implemented as an authorization filter.

// Apply filters: globally, per-controller, or per-action
builder.Services.AddControllers(options =>
{
    options.Filters.Add<LoggingActionFilter>(); // global — runs for all actions
});

[ServiceFilter(typeof(MyExceptionFilter))]       // per-controller / per-action
public class ProductsController : ControllerBase { }
```

**When to use which:**

| Concern | Use |
|---|---|
| Logging every HTTP request, HTTPS redirect, CORS, response compression | **Middleware** |
| Global catch-all error handling | **Middleware / `IExceptionHandler`** |
| Validating `ModelState`, transforming action results, action-level auth | **Filters** |
| Anything needing MVC context (action args, model, result) | **Filters** |

---

## B8. Hosted Services / BackgroundService

For long-running or scheduled background work (Spring's `@Scheduled`, `@Async`, or a `TaskExecutor`), implement `BackgroundService` (a helper base for `IHostedService`). The host starts it at app startup and stops it on shutdown.

```csharp
public class EmailQueueWorker(IServiceProvider services, ILogger<EmailQueueWorker> log)
    : BackgroundService
{
    // ExecuteAsync runs once at startup; the loop keeps it alive until shutdown.
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // BackgroundService is a SINGLETON, so create a scope to use Scoped services
            // (avoids the captive-dependency trap from A6).
            using var scope = services.CreateScope();
            var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
            await repo.ProcessPendingEmailsAsync();

            log.LogInformation("Processed email batch");
            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken); // like a @Scheduled fixedRate
        }
    }
}

// Register it:
builder.Services.AddHostedService<EmailQueueWorker>();
```

> For cron-style scheduling, pair this with a library like **Quartz.NET** or **Hangfire** (closer to Spring's full scheduler/quartz integration).

---

## Common Interview Questions

**1. What is Dependency Injection and how does ASP.NET Core implement it?**
DI is providing a class's dependencies from the outside rather than having it create them, achieving Inversion of Control. ASP.NET Core has a **built-in** container (`Microsoft.Extensions.DependencyInjection`). You register services on `IServiceCollection` (`builder.Services`) and they're resolved (usually via constructor injection) from the `IServiceProvider`. Unlike Spring, registration is **explicit**, not annotation-scanned.

**2. Explain the three service lifetimes and when you'd use each.**
- **Singleton** — one instance for the whole app; use for stateless/shared services, caches, config.
- **Scoped** — one instance per HTTP request; use for `DbContext`, repositories, per-request state (the most common choice).
- **Transient** — a new instance on every resolve; use for lightweight, stateless services. Map to Spring: singleton, request, prototype.

**3. What is a captive dependency? How do you avoid it?**
When a longer-lived service captures a shorter-lived one — classically injecting a **Scoped** service into a **Singleton**. The Scoped instance then lives as long as the Singleton, breaking per-request semantics (e.g., one `DbContext` shared across all requests). Avoid by matching lifetimes, or by injecting `IServiceScopeFactory` and creating a scope per operation. The default container validates and throws in Development.

**4. How do you register and inject multiple implementations of one interface?**
Register the interface multiple times (`AddScoped<INotifier, EmailNotifier>()`, etc.) and inject `IEnumerable<INotifier>` to get them all (like injecting `List<T>` in Spring). Injecting a single `INotifier` yields the *last* registered one.

**5. What is the .NET 8 equivalent of Spring's @Qualifier?**
**Keyed services.** Register with `AddKeyedScoped<I, T>("key")` and inject with `[FromKeyedServices("key")]`, or resolve via `GetRequiredKeyedService<T>("key")`.

**6. What is the Options pattern?**
A way to bind a configuration section to a strongly-typed POCO and inject it as `IOptions<T>` — the .NET equivalent of `@ConfigurationProperties`. Use `IOptionsSnapshot<T>` for per-request reloads and `IOptionsMonitor<T>` for live reload (safe inside Singletons).

**7. Who disposes services, and when?**
The container disposes services it created that implement `IDisposable`/`IAsyncDisposable`. Scoped/Transient are disposed at the end of their scope (request); Singletons at app shutdown. You don't manually dispose container-managed services.

**8. What is middleware? How is it different from a Servlet filter?**
Middleware is a component in the request pipeline that can act before and after calling `next`. It's structurally the same idea as a Servlet Filter (`chain.doFilter` ≈ `await next()`). The whole ASP.NET Core pipeline is an ordered chain of such components — a Chain of Responsibility.

**9. Difference between `app.Use`, `app.Run`, and `app.Map`?**
`Use` adds middleware that can call `next` to continue (or short-circuit by not calling it). `Run` is terminal — it ends the pipeline. `Map` branches the pipeline based on the request path prefix.

**10. Why does middleware order matter? Give the classic example.**
Each middleware wraps those after it, so order determines behavior. The canonical example: `UseAuthentication()` must precede `UseAuthorization()` — you must establish *who* the user is before checking *what they're allowed* to do. Exception handling goes near the top; routing before authorization.

**11. How do you write custom middleware?**
Inline with `app.Use(async (ctx, next) => {...})`, or class-based with a constructor taking `RequestDelegate next` and an `InvokeAsync(HttpContext)` method, registered via `app.UseMiddleware<T>()`. Class-based middleware is effectively a Singleton, so inject Scoped services as `InvokeAsync` parameters, not constructor params.

**12. How do you do global exception handling? Compare to @ControllerAdvice.**
Either a try/catch exception-handling middleware near the top of the pipeline, or (modern, .NET 8) implement `IExceptionHandler` + `AddExceptionHandler<T>()` + `app.UseExceptionHandler()`, returning RFC 7807 `ProblemDetails`. Both centralize error handling like a `@ControllerAdvice`.

**13. Middleware vs Filters — when do you use each?**
Middleware runs in the raw HTTP pipeline for every request and knows nothing about MVC. Filters run inside MVC around controller actions with full context (action args, ModelState, results), making them the closer analog to Spring AOP/`HandlerInterceptor`. Use middleware for app-wide concerns (logging, CORS, HTTPS, global errors); use filters for action-level concerns (model validation, result shaping, `[Authorize]`).

**14. Is the ASP.NET Core / Spring DI relationship one-to-one for AOP?**
No. Spring has general-purpose AOP (`@Aspect`, proxy/weaving). ASP.NET Core has no built-in general AOP; instead you pick the right tool per scope: **middleware** for pipeline-wide concerns, **filters** for MVC-scoped concerns, or the **decorator pattern** (sometimes via Scrutor) for service-level wrapping.

**15. How do you run background work?**
Implement `BackgroundService` (`ExecuteAsync`) and register with `AddHostedService<T>()` — the host runs it from startup to shutdown. Because it's a Singleton, create a DI scope inside it to use Scoped services. For cron scheduling, use Quartz.NET or Hangfire. This is the analog to Spring's `@Scheduled`/background jobs.

---

## Quick Reference Cheat Sheet

```
=== DEPENDENCY INJECTION ===

REGISTER (in Program.cs, on builder.Services):
  AddSingleton<I,T>()   one instance / app        ≈ Spring singleton
  AddScoped<I,T>()      one instance / request     ≈ Spring request scope (USE FOR DbContext)
  AddTransient<I,T>()   new instance / resolve     ≈ Spring prototype

LIFETIME RULE:  consumer lifetime <= dependency lifetime
  Singleton -> Singleton only
  Scoped    -> Scoped or Singleton
  Transient -> anything
  CAPTIVE DEPENDENCY = Scoped/Transient injected into a Singleton (container throws in Dev)

INJECTION:
  Constructor injection (standard)              ≈ @Autowired
  IEnumerable<I>  -> all registered impls       ≈ List<Bean>
  [FromKeyedServices("k")] (.NET 8)             ≈ @Qualifier("k")
  IServiceProvider / GetRequiredService<T>()    ≈ ApplicationContext.getBean() (avoid as locator)
  AddScoped<I>(sp => new T(...))  factory       ≈ @Bean factory method / FactoryBean
  Configure<T>(section) + IOptions<T>           ≈ @ConfigurationProperties

DISPOSAL: container disposes IDisposable services it created (scope end / app shutdown).
THIRD-PARTY: Autofac/Scrutor for scanning/decorators — usually NOT needed.

=== MIDDLEWARE PIPELINE (Program.cs, after app = builder.Build()) ===

  app.Use((ctx, next) => { ...; await next(ctx); ... })   non-terminal (can short-circuit)
  app.Run(ctx => ...)                                      TERMINAL (ends pipeline)
  app.Map("/path", branch => ...)                          branch by path
  next  ==  Servlet chain.doFilter()

RECOMMENDED ORDER (order = behavior):
  UseExceptionHandler   <- near top, wraps everything
  UseHttpsRedirection
  UseStaticFiles
  UseRouting
  UseCors
  UseAuthentication     <- WHO are you   (MUST come before...)
  UseAuthorization      <- are you ALLOWED
  MapControllers        <- terminal dispatch

CUSTOM MIDDLEWARE (class-based):
  ctor(RequestDelegate next, ...) ;  Task InvokeAsync(HttpContext ctx)
  register: app.UseMiddleware<MyMiddleware>()
  (it's a Singleton -> inject Scoped services as InvokeAsync params)

GLOBAL ERRORS (.NET 8):
  class : IExceptionHandler { TryHandleAsync(...) }
  AddExceptionHandler<T>() + AddProblemDetails() + app.UseExceptionHandler()
  ≈ @ControllerAdvice / @ExceptionHandler

MIDDLEWARE vs FILTERS:
  Middleware = raw HTTP pipeline, every request, no MVC context  (logging, CORS, HTTPS)
  Filters    = inside MVC, around actions, full context          ≈ Spring AOP / HandlerInterceptor
  Filter types: Authorization, Action (OnActionExecuting/Executed), Exception, Result

BACKGROUND WORK:
  class : BackgroundService { ExecuteAsync(stoppingToken) }
  AddHostedService<T>()   ≈ @Scheduled / background jobs
  create a scope inside to use Scoped services
```

---

*Last Updated: 2026-06-16*
