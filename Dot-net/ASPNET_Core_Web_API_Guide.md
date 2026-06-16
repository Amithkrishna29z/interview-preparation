# ASP.NET Core Web API — A Spring Boot Developer's Guide

## Overview

If you already know **Spring Boot**, you know 80% of ASP.NET Core — the names are just different. Both are batteries-included web frameworks built on a similar philosophy: convention over configuration, dependency injection everywhere, a request pipeline made of pluggable pieces, and annotations/attributes that turn plain classes into HTTP endpoints.

This guide teaches **ASP.NET Core Web API (.NET 8+)** by constantly mapping each concept back to its Spring equivalent. You'll learn both ways to build APIs in .NET — **Controller-based APIs** (the closest match to `@RestController`) and **Minimal APIs** (a lighter, route-handler style with no Spring equivalent) — plus model binding, validation, configuration, JSON, DI, Swagger, CORS, `HttpClient`, and logging.

By the end you should be able to read, write, and talk confidently about a real .NET Web API in an interview.

> Terminology bridge: In Spring, the framework is "Spring Boot," the language is Java, the build tool is Maven/Gradle, and the package manager is Maven Central. In .NET, the framework is "ASP.NET Core," the language is C#, the build tool + CLI is `dotnet`, and the package manager is NuGet.

---

## Table of Contents

- [Spring Boot → ASP.NET Core Mapping](#spring-boot--aspnet-core-mapping)
- [1. Project Structure & Program.cs (Minimal Hosting Model)](#1-project-structure--programcs-minimal-hosting-model)
- [2. Two Styles: Controller-Based vs Minimal APIs](#2-two-styles-controller-based-vs-minimal-apis)
- [3. Controllers, Routing & Route Constraints](#3-controllers-routing--route-constraints)
- [4. HTTP Verb Attributes](#4-http-verb-attributes)
- [5. Model Binding](#5-model-binding)
- [6. Returning Results (IActionResult / ActionResult<T>)](#6-returning-results-iactionresult--actionresultt)
- [7. DTOs & Model Validation (DataAnnotations + ModelState)](#7-dtos--model-validation-dataannotations--modelstate)
- [8. Configuration (appsettings.json, Options Pattern, Secrets)](#8-configuration-appsettingsjson-options-pattern-secrets)
- [9. Dependency Injection in Controllers](#9-dependency-injection-in-controllers)
- [10. JSON Serialization with System.Text.Json](#10-json-serialization-with-systemtextjson)
- [11. Middleware (Overview)](#11-middleware-overview)
- [12. Swagger / OpenAPI, API Versioning](#12-swagger--openapi-api-versioning)
- [13. CORS & HTTPS Redirection](#13-cors--https-redirection)
- [14. Calling Other APIs: HttpClient / IHttpClientFactory](#14-calling-other-apis-httpclient--ihttpclientfactory)
- [15. Logging with ILogger](#15-logging-with-ilogger)
- [16. Health Checks](#16-health-checks)
- [17. Complete Worked Example: ProductsController CRUD](#17-complete-worked-example-productscontroller-crud)
- [Common Interview Questions](#common-interview-questions)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Spring Boot → ASP.NET Core Mapping

| Spring Boot | ASP.NET Core | Notes |
|---|---|---|
| `@RestController` | `[ApiController]` + `ControllerBase` | `[ApiController]` adds automatic 400s, binding inference |
| `@Controller` (returns views) | `Controller` (MVC base, has View support) | Use `ControllerBase` for pure APIs |
| `@RequestMapping("/api")` | `[Route("api/[controller]")]` | `[controller]` token = class name minus "Controller" |
| `@GetMapping` | `[HttpGet]` | |
| `@PostMapping` | `[HttpPost]` | |
| `@PutMapping` | `[HttpPut]` | |
| `@DeleteMapping` | `[HttpDelete]` | |
| `@PatchMapping` | `[HttpPatch]` | |
| `@RequestBody` | `[FromBody]` | Often inferred with `[ApiController]` |
| `@PathVariable` | `[FromRoute]` | Often inferred from route template |
| `@RequestParam` | `[FromQuery]` | |
| `@RequestHeader` | `[FromHeader]` | |
| Form data binding | `[FromForm]` | |
| `@Service` / `@Component` / `@Repository` | Plain class + `builder.Services.AddScoped<T>()` | No stereotype attribute; you register in DI |
| `@Autowired` (field) | Constructor injection (preferred) | Constructor injection is idiomatic in .NET |
| `ResponseEntity<T>` | `IActionResult` / `ActionResult<T>` | |
| `ResponseEntity.ok(x)` | `Ok(x)` | |
| `ResponseEntity.notFound()` | `NotFound()` | |
| `ResponseEntity.badRequest()` | `BadRequest()` | |
| `ResponseEntity.created(uri)` | `CreatedAtAction(...)` / `Created(...)` | |
| `@ControllerAdvice` / `@ExceptionHandler` | Exception-handling **middleware** or **exception filter** | Middleware is the modern default |
| `@Valid` + Bean Validation | `[ApiController]` auto-validates DataAnnotations | `@NotNull` → `[Required]`, etc. |
| `@NotNull` / `@NotBlank` | `[Required]` | |
| `@Size(min,max)` | `[StringLength]` / `[MinLength]` / `[MaxLength]` | |
| `@Min` / `@Max` | `[Range]` | |
| `@Email` | `[EmailAddress]` | |
| `application.properties` / `application.yml` | `appsettings.json` | |
| `application-prod.properties` | `appsettings.Production.json` | Selected by `ASPNETCORE_ENVIRONMENT` |
| `@Value("${...}")` | `IConfiguration["..."]` or Options pattern | |
| `@ConfigurationProperties` | `IOptions<T>` (Options pattern) | |
| `spring.profiles.active` | `ASPNETCORE_ENVIRONMENT` (Development/Staging/Production) | |
| Spring Initializr | `dotnet new webapi` | |
| `mvn spring-boot:run` | `dotnet run` | |
| `pom.xml` / `build.gradle` | `.csproj` | |
| Maven Central | NuGet | |
| Jackson (`ObjectMapper`) | `System.Text.Json` | Newtonsoft.Json is the optional alternative |
| `RestTemplate` / `WebClient` | `HttpClient` / `IHttpClientFactory` | |
| Logback + SLF4J `Logger` | `ILogger<T>` (Microsoft.Extensions.Logging) | |
| Springdoc / Swagger UI | Swashbuckle / `Swagger UI` | |
| Spring Boot Actuator health | `AddHealthChecks()` + `/health` | |
| `SpringApplication.run(...)` | `WebApplication.CreateBuilder(args)` + `app.Run()` | |

---

## 1. Project Structure & Program.cs (Minimal Hosting Model)

**Think of it like...** `Program.cs` is your `main()` method + `@SpringBootApplication` class fused together. In Spring you have a `DemoApplication.java` with `SpringApplication.run(...)`; in .NET you have `Program.cs` where you both *configure* services (like a `@Configuration` class) and *run* the app.

Create a project:

```bash
# Spring equivalent: Spring Initializr -> download zip
dotnet new webapi -n ShopApi      # controller-based template
cd ShopApi
dotnet run                          # like: mvn spring-boot:run
```

A typical layout:

```
ShopApi/
├── Program.cs                 # main() + DI config + pipeline (the heart)
├── appsettings.json           # application.properties
├── appsettings.Development.json
├── ShopApi.csproj             # pom.xml
├── Controllers/
│   └── ProductsController.cs   # @RestController classes
├── Models/                     # entities / domain
├── Dtos/                       # request/response DTOs
└── Services/                   # @Service classes (interfaces + impls)
```

The **minimal hosting model** (`.NET 6+`) replaced the old `Startup.cs` + `Program.cs` split. Everything now lives in one file:

```csharp
// Program.cs — top-level statements, no class/Main boilerplate needed.

// 1) Create the builder. Like SpringApplication context being prepared.
var builder = WebApplication.CreateBuilder(args);

// 2) REGISTER SERVICES into the DI container.
//    This block is your @Configuration / component scanning equivalent.
//    Everything before builder.Build() = configuring the container.
builder.Services.AddControllers();           // enable @RestController-style controllers
builder.Services.AddEndpointsApiExplorer();  // metadata for Swagger
builder.Services.AddSwaggerGen();            // Swagger doc generator (Springdoc)
builder.Services.AddScoped<IProductService, ProductService>(); // register @Service

// 3) Build the app. After this, the service container is "frozen".
var app = builder.Build();

// 4) CONFIGURE THE REQUEST PIPELINE (middleware order matters!).
//    This is like Spring's filter chain — but explicit and ordered.
if (app.Environment.IsDevelopment())          // like checking active profile
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();   // force HTTPS
app.UseAuthorization();      // auth checks
app.MapControllers();        // wire up controller routes

// 5) Start listening. Equivalent to SpringApplication.run() blocking.
app.Run();
```

Mental model: **`builder.Services.Add...`** = "register beans." **`app.Use...` / `app.Map...`** = "build the filter/handler chain." The order of `app.Use...` calls is the order middleware runs — unlike Spring where filter order is often annotation/`@Order` driven.

---

## 2. Two Styles: Controller-Based vs Minimal APIs

ASP.NET Core gives you **two** ways to define endpoints. Spring only has the controller style.

### Style A — Controller-based (closest to Spring `@RestController`)

```csharp
[ApiController]                       // @RestController behaviors
[Route("api/[controller]")]           // @RequestMapping("/api/products")
public class ProductsController : ControllerBase
{
    [HttpGet("{id}")]                 // @GetMapping("/{id}")
    public IActionResult Get(int id)  // @PathVariable inferred
        => Ok(new { Id = id, Name = "Widget" }); // ResponseEntity.ok(...)
}
```

### Style B — Minimal APIs (no Spring equivalent — think route lambdas)

**Think of it like...** a super-lightweight router where each route is a lambda, similar to JavaScript Express or Spring's functional `RouterFunction`, but cleaner.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IProductService, ProductService>();
var app = builder.Build();

// Define endpoints inline. DI parameters are injected automatically.
app.MapGet("/api/products/{id:int}", (int id, IProductService svc) =>
{
    var product = svc.GetById(id);
    return product is null ? Results.NotFound() : Results.Ok(product);
    // Results.* is the Minimal-API analog of Ok()/NotFound() helpers.
});

app.MapPost("/api/products", (CreateProductDto dto, IProductService svc) =>
{
    var created = svc.Create(dto);
    return Results.Created($"/api/products/{created.Id}", created);
});

app.Run();
```

**When to use which:**

| Use Controllers when... | Use Minimal APIs when... |
|---|---|
| Large API, many endpoints | Small service / microservice |
| You want filters, model binding attributes, `[ApiController]` conveniences | You want minimal ceremony / fastest startup |
| Team familiar with MVC / Spring style | Prototypes, lambdas, internal tools |
| You need rich conventions (validation, content negotiation) out of the box | You'll group routes manually |

For a junior .NET job, **master controllers first** (they map directly to your Spring knowledge), then understand Minimal APIs exist.

---

## 3. Controllers, Routing & Route Constraints

**Think of it like...** `[Route]` + `[ApiController]` together equal `@RestController` + `@RequestMapping`.

```csharp
[ApiController]                          // Adds: auto 400 on invalid model,
                                         // binding source inference, attribute-routing requirement
[Route("api/[controller]")]              // [controller] = "Products" (class minus "Controller")
                                         // => base route "api/products"
public class ProductsController : ControllerBase
{
    // Full route: GET api/products
    [HttpGet]
    public IActionResult GetAll() => Ok();

    // Full route: GET api/products/42
    // Route template with a parameter, like @GetMapping("/{id}")
    [HttpGet("{id}")]
    public IActionResult GetOne(int id) => Ok(id);

    // Route CONSTRAINTS restrict what matches. Spring uses regex in @PathVariable;
    // .NET has named constraints: :int, :guid, :min(1), :alpha, :length(...), etc.
    [HttpGet("{id:int:min(1)}")]         // only matches positive integers
    public IActionResult GetConstrained(int id) => Ok(id);

    // Override the base route entirely with a leading "/"
    [HttpGet("/api/health-ping")]
    public IActionResult Ping() => Ok("pong");
}
```

`ControllerBase` (no View support — for APIs) vs `Controller` (adds Razor View support — for MVC web pages). For Web APIs you almost always inherit `ControllerBase`.

Common route constraints: `int`, `bool`, `datetime`, `decimal`, `guid`, `long`, `alpha`, `min(n)`, `max(n)`, `range(a,b)`, `length(n)`, `regex(...)`.

---

## 4. HTTP Verb Attributes

**Think of it like...** the `@GetMapping`/`@PostMapping` family, one-to-one.

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    [HttpGet]                       // @GetMapping            — read collection
    public IActionResult List() => Ok();

    [HttpGet("{id}")]               // @GetMapping("/{id}")   — read one
    public IActionResult Get(int id) => Ok(id);

    [HttpPost]                      // @PostMapping           — create
    public IActionResult Create([FromBody] OrderDto dto) => Ok(dto);

    [HttpPut("{id}")]               // @PutMapping("/{id}")   — full replace
    public IActionResult Replace(int id, [FromBody] OrderDto dto) => NoContent();

    [HttpPatch("{id}")]             // @PatchMapping("/{id}") — partial update
    public IActionResult Patch(int id, [FromBody] OrderDto dto) => NoContent();

    [HttpDelete("{id}")]            // @DeleteMapping("/{id}")— delete
    public IActionResult Delete(int id) => NoContent();
}
```

---

## 5. Model Binding

**Think of it like...** the `@RequestBody` / `@PathVariable` / `@RequestParam` family. With `[ApiController]`, .NET *infers* the source so you often don't even write the attribute:

- Complex type → `[FromBody]` (inferred)
- Route-template name match → `[FromRoute]` (inferred)
- Otherwise simple type → `[FromQuery]` (inferred)

```csharp
[ApiController]
[Route("api/[controller]")]
public class SearchController : ControllerBase
{
    // GET api/search/electronics?page=2&size=20&minPrice=10
    [HttpGet("{category}")]
    public IActionResult Search(
        [FromRoute] string category,    // @PathVariable
        [FromQuery] int page = 1,       // @RequestParam(defaultValue="1")
        [FromQuery] int size = 10,      // @RequestParam
        [FromHeader(Name = "X-Tenant")] string? tenant = null) // @RequestHeader
        => Ok(new { category, page, size, tenant });

    // POST api/search  with JSON body
    [HttpPost]
    public IActionResult Advanced([FromBody] SearchDto dto)  // @RequestBody
        => Ok(dto);

    // POST api/search/upload  with multipart/form-data
    [HttpPost("upload")]
    public IActionResult Upload([FromForm] IFormFile file)   // form/multipart binding
        => Ok(file.FileName);
}
```

| Attribute | Source | Spring equivalent |
|---|---|---|
| `[FromBody]` | Request body (JSON) | `@RequestBody` |
| `[FromRoute]` | Route template | `@PathVariable` |
| `[FromQuery]` | Query string | `@RequestParam` |
| `[FromHeader]` | HTTP header | `@RequestHeader` |
| `[FromForm]` | Form / multipart | form binding |

Note: only **one** `[FromBody]` parameter is allowed per action (same as Spring — one body).

---

## 6. Returning Results (IActionResult / ActionResult<T>)

**Think of it like...** `ResponseEntity<T>`. Two main return-type choices:

- **`IActionResult`** — return *any* result; type isn't documented in the signature. Like `ResponseEntity<?>`.
- **`ActionResult<T>`** — best of both: you can return `T` directly *or* a helper like `NotFound()`. Swagger documents `T`. **Preferred for typed endpoints.**

```csharp
[HttpGet("{id}")]
public ActionResult<ProductDto> Get(int id)   // like ResponseEntity<ProductDto>
{
    var product = _service.GetById(id);
    if (product is null)
        return NotFound();                     // 404 — ResponseEntity.notFound()
    return product;                            // 200 + body (implicit Ok) — auto-wrapped
}

[HttpPost]
public ActionResult<ProductDto> Create(CreateProductDto dto)
{
    var created = _service.Create(dto);
    // 201 Created + Location header pointing to the GET endpoint.
    // 1st arg = action name, 2nd = route values, 3rd = body.
    return CreatedAtAction(nameof(Get), new { id = created.Id }, created);
}
```

Common result helpers (all from `ControllerBase`):

| Helper | Status | Spring equivalent |
|---|---|---|
| `Ok(obj)` | 200 | `ResponseEntity.ok(obj)` |
| `Created(uri, obj)` / `CreatedAtAction(...)` | 201 | `ResponseEntity.created(uri)` |
| `NoContent()` | 204 | `ResponseEntity.noContent().build()` |
| `BadRequest(err)` | 400 | `ResponseEntity.badRequest()` |
| `Unauthorized()` | 401 | `status(401)` |
| `Forbid()` | 403 | `status(403)` |
| `NotFound()` | 404 | `ResponseEntity.notFound()` |
| `Conflict()` | 409 | `status(409)` |
| `StatusCode(500, err)` | any | `ResponseEntity.status(...)` |

---

## 7. DTOs & Model Validation (DataAnnotations + ModelState)

**Think of it like...** Bean Validation (`@Valid` + `@NotNull`/`@Size`/`@Email`). In .NET the annotations are called **DataAnnotations**, and with `[ApiController]` validation runs **automatically** — an invalid model returns a `400` with a problem-details body *before your action even runs*. (No `@Valid` needed on the parameter.)

```csharp
using System.ComponentModel.DataAnnotations;

public class CreateProductDto
{
    [Required]                                 // @NotNull / @NotBlank
    [StringLength(100, MinimumLength = 2)]     // @Size(min=2, max=100)
    public string Name { get; set; } = string.Empty;

    [Range(0.01, 100000)]                      // @Min/@Max (or @DecimalMin/@DecimalMax)
    public decimal Price { get; set; }

    [EmailAddress]                             // @Email
    public string? ContactEmail { get; set; }

    [Required]
    [RegularExpression(@"^[A-Z]{3}-\d{4}$")]   // @Pattern(regexp=...)
    public string Sku { get; set; } = string.Empty;
}
```

With `[ApiController]`, the auto-400 looks like this (you write nothing):

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "Name": ["The Name field is required."],
    "Price": ["The field Price must be between 0.01 and 100000."]
  }
}
```

If you need to validate **manually** (e.g. in Minimal APIs, or for custom rules), inspect `ModelState`:

```csharp
[HttpPost]
public IActionResult Create(CreateProductDto dto)
{
    if (!ModelState.IsValid)                   // like binding result has errors
        return BadRequest(ModelState);         // returns the error dictionary

    // ... proceed
    return Ok();
}
```

> `ModelState` is the .NET equivalent of Spring's `BindingResult`. With `[ApiController]` you rarely touch it because the framework short-circuits invalid requests for you.

---

## 8. Configuration (appsettings.json, Options Pattern, Secrets)

**Think of it like...** `application.properties` is now `appsettings.json`, and `@ConfigurationProperties` is now the **Options pattern** (`IOptions<T>`).

`appsettings.json`:

```json
{
  "Logging": {
    "LogLevel": { "Default": "Information", "Microsoft.AspNetCore": "Warning" }
  },
  "AllowedHosts": "*",
  "ProductApi": {
    "BaseUrl": "https://api.example.com",
    "TimeoutSeconds": 30,
    "ApiKey": "REPLACE_ME"
  }
}
```

**Environment-specific config** layers on top, like Spring profiles. `appsettings.Production.json` overrides `appsettings.json` when `ASPNETCORE_ENVIRONMENT=Production` (the analog of `spring.profiles.active=prod`). Order of precedence (last wins): `appsettings.json` → `appsettings.{Environment}.json` → User Secrets (dev) → Environment Variables → command-line args.

**The Options pattern** binds a config section to a strongly typed class:

```csharp
// 1) The strongly-typed settings class — like an @ConfigurationProperties POJO.
public class ProductApiOptions
{
    public string BaseUrl { get; set; } = string.Empty;
    public int TimeoutSeconds { get; set; }
    public string ApiKey { get; set; } = string.Empty;
}

// 2) Register/bind it in Program.cs:
builder.Services.Configure<ProductApiOptions>(
    builder.Configuration.GetSection("ProductApi"));   // bind "ProductApi" section

// 3) Inject IOptions<T> wherever you need it:
public class ProductService
{
    private readonly ProductApiOptions _opts;
    public ProductService(IOptions<ProductApiOptions> opts)
        => _opts = opts.Value;     // .Value unwraps the bound instance
}
```

**Quick reads** (no class) use `IConfiguration` — the `@Value("${...}")` equivalent:

```csharp
string url = builder.Configuration["ProductApi:BaseUrl"];          // ":" is the section separator
int timeout = builder.Configuration.GetValue<int>("ProductApi:TimeoutSeconds");
```

**User Secrets** keep dev secrets out of source control (like a local untracked properties file):

```bash
dotnet user-secrets init
dotnet user-secrets set "ProductApi:ApiKey" "super-secret-dev-key"
```

In production you'd typically use **environment variables** (e.g. `ProductApi__ApiKey` — note the double underscore replaces the `:`) or a vault.

---

## 9. Dependency Injection in Controllers

**Think of it like...** `@Autowired`, but you register services explicitly and inject via the **constructor** (the idiomatic way). DI is built into ASP.NET Core — no extra container needed. (A separate guide covers DI deeply; this is the controller-relevant subset.)

Register with one of three lifetimes:

```csharp
builder.Services.AddTransient<IEmailSender, EmailSender>();  // new instance every time
builder.Services.AddScoped<IProductService, ProductService>(); // one per HTTP request (default for web)
builder.Services.AddSingleton<ICache, MemoryCache>();         // one for app lifetime
```

| Lifetime | .NET | Spring analog |
|---|---|---|
| `AddSingleton` | one per app | default singleton bean |
| `AddScoped` | one per HTTP request | `@RequestScope` |
| `AddTransient` | new each resolution | `@Scope("prototype")` |

Inject into a controller:

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;
    private readonly ILogger<ProductsController> _logger;

    // Constructor injection — the framework supplies these (like @Autowired ctor).
    public ProductsController(IProductService service, ILogger<ProductsController> logger)
    {
        _service = service;
        _logger = logger;
    }
}
```

---

## 10. JSON Serialization with System.Text.Json

**Think of it like...** Jackson + `ObjectMapper`. The built-in serializer is **`System.Text.Json`** (fast, no extra dependency). By default it serializes to **camelCase** (so `ProductName` → `"productName"`), matching typical JS clients.

Configure globally in `Program.cs`:

```csharp
builder.Services.AddControllers()
    .AddJsonOptions(options =>
    {
        // camelCase is the default; this is explicit.
        options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
        // Don't serialize null properties (like @JsonInclude(NON_NULL)).
        options.JsonSerializerOptions.DefaultIgnoreCondition =
            System.Text.Json.Serialization.JsonIgnoreCondition.WhenWritingNull;
    });
```

Per-property attributes (the Jackson `@JsonProperty` / `@JsonIgnore` analogs):

```csharp
using System.Text.Json.Serialization;

public class ProductDto
{
    [JsonPropertyName("product_id")]   // @JsonProperty("product_id")
    public int Id { get; set; }

    public string Name { get; set; } = string.Empty;

    [JsonIgnore]                       // @JsonIgnore — never serialized
    public string InternalNotes { get; set; } = string.Empty;
}
```

> If you need advanced behavior (polymorphism quirks, lenient parsing), the team may swap in **Newtonsoft.Json** via `AddNewtonsoftJson()` — the older, more feature-rich library, closest in flexibility to Jackson.

---

## 11. Middleware (Overview)

**Think of it like...** the Servlet **filter chain** / Spring's `OncePerRequestFilter`. Middleware are components that run in order on every request and can short-circuit or pass to the next. A dedicated guide covers middleware deeply — here's the essentials.

```csharp
// Each app.Use... is a link in the chain; ORDER MATTERS (top runs first).
app.UseHttpsRedirection();   // 1
app.UseAuthentication();     // 2  who are you?
app.UseAuthorization();      // 3  are you allowed?
app.MapControllers();        // 4  terminal: route to controller

// Inline custom middleware (like a quick filter):
app.Use(async (context, next) =>
{
    Console.WriteLine($"--> {context.Request.Method} {context.Request.Path}");
    await next();            // call the next middleware (like chain.doFilter)
    Console.WriteLine($"<-- {context.Response.StatusCode}");
});
```

**Global exception handling** — the `@ControllerAdvice` / `@ExceptionHandler` equivalent — is done with exception-handling middleware (e.g. `app.UseExceptionHandler(...)`) or an `IExceptionFilter`. Prefer the middleware approach in modern .NET.

---

## 12. Swagger / OpenAPI, API Versioning

**Think of it like...** Springdoc OpenAPI / Swagger UI. The standard .NET library is **Swashbuckle**.

```csharp
builder.Services.AddEndpointsApiExplorer();   // discovers endpoints for the doc
builder.Services.AddSwaggerGen();             // generates the OpenAPI document

var app = builder.Build();

if (app.Environment.IsDevelopment())          // expose only in dev (common practice)
{
    app.UseSwagger();                          // serves /swagger/v1/swagger.json
    app.UseSwaggerUI();                        // serves the interactive UI at /swagger
}
```

> Note: as of .NET 9 the default template uses `Microsoft.AspNetCore.OpenApi` (`AddOpenApi()`) instead of Swashbuckle, but **Swashbuckle is still the most common in real codebases** and what you'll see in most interviews.

**API versioning** (brief) keeps old clients working as your API evolves — the analog of Spring's URL/header versioning. Add the `Asp.Versioning.Mvc` NuGet package:

```csharp
builder.Services.AddApiVersioning(o =>
{
    o.DefaultApiVersion = new ApiVersion(1, 0);
    o.AssumeDefaultVersionWhenUnspecified = true;
    o.ReportApiVersions = true;     // adds api-supported-versions response header
});
```

```csharp
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]  // URL-segment versioning: /api/v1/products
public class ProductsController : ControllerBase { }
```

---

## 13. CORS & HTTPS Redirection

**Think of it like...** Spring's `@CrossOrigin` / `WebMvcConfigurer.addCorsMappings`.

```csharp
// 1) Register a CORS policy (services phase).
builder.Services.AddCors(options =>
{
    options.AddPolicy("frontend", policy =>
        policy.WithOrigins("https://myapp.com")  // allowed origins
              .AllowAnyHeader()
              .AllowAnyMethod());
});

var app = builder.Build();

// 2) Enable it in the pipeline — BEFORE MapControllers, AFTER routing.
app.UseCors("frontend");

// HTTPS redirection: send HTTP requests to HTTPS (like Spring's requiresSecure()).
app.UseHttpsRedirection();
```

Order matters: `UseCors` must come before the endpoints it protects.

---

## 14. Calling Other APIs: HttpClient / IHttpClientFactory

**Think of it like...** `RestTemplate` (legacy) / `WebClient` (reactive) in Spring. In .NET, `HttpClient` is the client, but you should **never `new HttpClient()` per call** (socket exhaustion). Instead use **`IHttpClientFactory`** — the recommended, pooled approach.

```csharp
// 1) Register a typed client in Program.cs (binds an HttpClient to a service class).
builder.Services.AddHttpClient<WeatherClient>(client =>
{
    client.BaseAddress = new Uri("https://api.weather.com/");
    client.Timeout = TimeSpan.FromSeconds(10);
});
```

```csharp
// 2) The typed client receives a properly-pooled HttpClient via DI.
public class WeatherClient
{
    private readonly HttpClient _http;
    public WeatherClient(HttpClient http) => _http = http;  // injected by the factory

    public async Task<Forecast?> GetForecastAsync(string city)
    {
        // GetFromJsonAsync = GET + deserialize in one call (System.Text.Json).
        // Like restTemplate.getForObject(url, Forecast.class) but async.
        return await _http.GetFromJsonAsync<Forecast>($"forecast?city={city}");
    }

    public async Task<Forecast?> CreateAsync(ForecastRequest req)
    {
        var resp = await _http.PostAsJsonAsync("forecast", req);  // POST + serialize body
        resp.EnsureSuccessStatusCode();                            // throw on non-2xx
        return await resp.Content.ReadFromJsonAsync<Forecast>();
    }
}
```

Note the **`async`/`await`** + `Task<T>` pattern: .NET is async-first for I/O. `Task<T>` is roughly Java's `CompletableFuture<T>`, and `await` is like `.get()` but non-blocking.

---

## 15. Logging with ILogger

**Think of it like...** SLF4J `Logger` + Logback. `ILogger<T>` is built in and injected via DI — no `LoggerFactory.getLogger(...)` boilerplate.

```csharp
public class ProductsController : ControllerBase
{
    private readonly ILogger<ProductsController> _logger;  // <T> sets the category (like the class name)

    public ProductsController(ILogger<ProductsController> logger) => _logger = logger;

    [HttpGet("{id}")]
    public IActionResult Get(int id)
    {
        // Structured logging: {Id} is a named placeholder, NOT string concat.
        // logger.info("Fetching product {}", id) in SLF4J.
        _logger.LogInformation("Fetching product {ProductId}", id);

        try { /* ... */ }
        catch (Exception ex)
        {
            // First arg is the exception (like logger.error(msg, ex)).
            _logger.LogError(ex, "Failed to fetch product {ProductId}", id);
            return StatusCode(500);
        }
        return Ok();
    }
}
```

Log levels (low→high): `Trace`, `Debug`, `Information`, `Warning`, `Error`, `Critical`. Configure thresholds in `appsettings.json` under `"Logging"` (shown in section 8) — the analog of Logback levels.

---

## 16. Health Checks

**Think of it like...** Spring Boot Actuator's `/actuator/health`.

```csharp
builder.Services.AddHealthChecks();   // register the health check service
// ... add custom checks like .AddCheck<DbHealthCheck>("database") as needed

var app = builder.Build();
app.MapHealthChecks("/health");       // exposes GET /health -> "Healthy" / 503
```

Hitting `GET /health` returns `200 Healthy` when up, or `503` when a registered check fails — useful for Kubernetes liveness/readiness probes and load balancers.

---

## 17. Complete Worked Example: ProductsController CRUD

A full, idiomatic slice: **DTOs → service interface + impl → controller**, with validation, DI, logging, and proper status codes.

**DTOs** (`Dtos/ProductDtos.cs`):

```csharp
using System.ComponentModel.DataAnnotations;

// Response shape returned to clients (never expose entities directly).
public record ProductDto(int Id, string Name, decimal Price);

// Request shape for creation — validated automatically by [ApiController].
public class CreateProductDto
{
    [Required, StringLength(100, MinimumLength = 2)]   // @NotBlank @Size(2,100)
    public string Name { get; set; } = string.Empty;

    [Range(0.01, 100000)]                              // @DecimalMin/@Max
    public decimal Price { get; set; }
}
```

**Service** (`Services/IProductService.cs`):

```csharp
public interface IProductService           // like a @Service interface
{
    IEnumerable<ProductDto> GetAll();
    ProductDto? GetById(int id);            // ? => nullable, may not exist
    ProductDto Create(CreateProductDto dto);
    bool Update(int id, CreateProductDto dto);
    bool Delete(int id);
}

// In-memory implementation (a real one would use EF Core / a DbContext).
public class ProductService : IProductService
{
    private readonly List<ProductDto> _products = new()
    {
        new(1, "Keyboard", 49.99m),
        new(2, "Mouse", 19.99m)
    };
    private int _nextId = 3;

    public IEnumerable<ProductDto> GetAll() => _products;

    public ProductDto? GetById(int id) => _products.FirstOrDefault(p => p.Id == id);

    public ProductDto Create(CreateProductDto dto)
    {
        var product = new ProductDto(_nextId++, dto.Name, dto.Price);
        _products.Add(product);
        return product;
    }

    public bool Update(int id, CreateProductDto dto)
    {
        var index = _products.FindIndex(p => p.Id == id);
        if (index == -1) return false;
        _products[index] = new ProductDto(id, dto.Name, dto.Price);
        return true;
    }

    public bool Delete(int id)
    {
        var existing = _products.FirstOrDefault(p => p.Id == id);
        if (existing is null) return false;
        _products.Remove(existing);
        return true;
    }
}
```

**Controller** (`Controllers/ProductsController.cs`):

```csharp
[ApiController]
[Route("api/[controller]")]                 // => api/products
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;
    private readonly ILogger<ProductsController> _logger;

    // Constructor injection (the @Autowired-ctor equivalent).
    public ProductsController(IProductService service, ILogger<ProductsController> logger)
    {
        _service = service;
        _logger = logger;
    }

    // GET api/products
    [HttpGet]
    public ActionResult<IEnumerable<ProductDto>> GetAll()
    {
        _logger.LogInformation("Listing all products");
        return Ok(_service.GetAll());           // 200 + array
    }

    // GET api/products/5
    [HttpGet("{id:int}")]
    public ActionResult<ProductDto> GetById(int id)
    {
        var product = _service.GetById(id);
        if (product is null)
            return NotFound();                  // 404
        return product;                         // 200 (auto-wrapped via ActionResult<T>)
    }

    // POST api/products  (body validated automatically; invalid => auto 400)
    [HttpPost]
    public ActionResult<ProductDto> Create(CreateProductDto dto)
    {
        var created = _service.Create(dto);
        // 201 Created + Location header -> GET api/products/{newId}
        return CreatedAtAction(nameof(GetById), new { id = created.Id }, created);
    }

    // PUT api/products/5
    [HttpPut("{id:int}")]
    public IActionResult Update(int id, CreateProductDto dto)
    {
        if (!_service.Update(id, dto))
            return NotFound();                  // 404 if it didn't exist
        return NoContent();                     // 204 — success, no body
    }

    // DELETE api/products/5
    [HttpDelete("{id:int}")]
    public IActionResult Delete(int id)
    {
        if (!_service.Delete(id))
            return NotFound();
        return NoContent();                     // 204
    }
}
```

**Wire it up** (`Program.cs`):

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddScoped<IProductService, ProductService>();  // register the @Service

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

Run it with `dotnet run`, then open `https://localhost:<port>/swagger` to try every endpoint — your full CRUD API is live.

---

## Common Interview Questions

**1. What's the difference between `ControllerBase` and `Controller`?**
`ControllerBase` provides API features (model binding, `Ok()`, `NotFound()`, etc.) without View support — use it for Web APIs. `Controller` extends `ControllerBase` and adds Razor View support for MVC web pages. For a Web API you inherit `ControllerBase`. (Spring analog: `@RestController` vs `@Controller`.)

**2. What does `[ApiController]` actually do?**
It opts the controller into API conventions: automatic `400` responses when model validation fails (no manual `ModelState` check), binding-source inference (so `[FromBody]`/`[FromRoute]` are often unnecessary), attribute-routing requirement, and problem-details error responses. It's the behavior that makes a controller feel like `@RestController`.

**3. Controllers vs Minimal APIs — when do you choose each?**
Controllers suit large APIs needing rich conventions (filters, model binding attributes, content negotiation) and map cleanly to MVC/Spring mental models. Minimal APIs suit small services/microservices and prototypes where you want the least ceremony and fastest startup. Controllers scale better organizationally; Minimal APIs are leaner.

**4. Explain the minimal hosting model and `Program.cs`.**
Since .NET 6, `Program.cs` uses top-level statements and merges the old `Program`+`Startup` files. You get a `WebApplicationBuilder` (configure DI/services), call `Build()` to produce a `WebApplication`, configure the middleware pipeline with `app.Use...`/`app.Map...`, then `app.Run()`. It's `main()` + `@SpringBootApplication` + `@Configuration` in one place.

**5. What's the difference between `IActionResult` and `ActionResult<T>`?**
`IActionResult` returns any result but doesn't document the body type. `ActionResult<T>` lets you return either a `T` directly or a result helper like `NotFound()`, and Swagger documents `T`. Prefer `ActionResult<T>` for typed endpoints. (Both echo `ResponseEntity<T>`.)

**6. How does model validation work and how does it compare to Spring?**
You annotate DTO properties with DataAnnotations (`[Required]`, `[StringLength]`, `[Range]`, `[EmailAddress]`). With `[ApiController]`, invalid models are rejected automatically with a `400` + problem-details body before your action runs — no `@Valid` and no manual `BindingResult` check. In Minimal APIs or custom flows you check `ModelState.IsValid` yourself (the `BindingResult` equivalent).

**7. Explain the three DI lifetimes.**
`Singleton` = one instance for the whole app. `Scoped` = one per HTTP request (the default for request-bound services like EF `DbContext`). `Transient` = a new instance each time it's resolved. Picking wrong (e.g. injecting a Scoped service into a Singleton) causes captive-dependency bugs.

**8. How does configuration and the Options pattern work?**
Config comes from layered providers (`appsettings.json` → `appsettings.{Env}.json` → user secrets → env vars → CLI args, last wins). The Options pattern binds a config section to a typed class via `services.Configure<T>(...)`, injected as `IOptions<T>` — the `@ConfigurationProperties` analog. For quick reads use `IConfiguration["Section:Key"]` (the `@Value` analog). Environment is selected by `ASPNETCORE_ENVIRONMENT`.

**9. What is `CreatedAtAction` and why use it for POST?**
It returns `201 Created` with a `Location` response header pointing at the GET endpoint for the new resource, plus the created body. This is the RESTful convention for creation (vs. a plain `Ok()`), equivalent to Spring's `ResponseEntity.created(uri)`.

**10. Why use `IHttpClientFactory` instead of `new HttpClient()`?**
Creating a new `HttpClient` per request exhausts sockets (each holds a connection in `TIME_WAIT`), while a single static one doesn't honor DNS changes. `IHttpClientFactory` pools and recycles handlers correctly, supports named/typed clients, and integrates resilience policies. It's the recommended replacement for `RestTemplate`/`WebClient`.

**11. How does model binding decide where a parameter comes from?**
With `[ApiController]`, inference applies: complex types bind from the body (`[FromBody]`), names matching the route template bind from the route (`[FromRoute]`), and remaining simple types bind from the query string (`[FromQuery]`). You can override with explicit `[From*]` attributes. Only one `[FromBody]` parameter is allowed per action.

**12. How do you do global exception handling (vs `@ControllerAdvice`)?**
Use exception-handling middleware (`app.UseExceptionHandler(...)`) or an `IExceptionFilter`. Middleware is the modern default: it wraps the whole pipeline, catches unhandled exceptions, and converts them to consistent problem-details responses — the `@ControllerAdvice` + `@ExceptionHandler` equivalent.

**13. What serializer does ASP.NET Core use, and what casing does it default to?**
`System.Text.Json` by default (fast, low-allocation), serializing to **camelCase**. You configure it via `AddJsonOptions(...)` and control individual properties with `[JsonPropertyName]` / `[JsonIgnore]`. Teams needing more flexibility can swap in Newtonsoft.Json. (Jackson/`ObjectMapper` analog.)

**14. What's `async`/`await` and `Task<T>` in a controller action?**
`Task<T>` represents an in-progress async operation (≈ `CompletableFuture<T>`); `await` suspends without blocking the thread, freeing it to serve other requests. Use `async Task<ActionResult<T>>` for I/O-bound actions (DB, HTTP calls) to maximize throughput. It's .NET's async-first model for scalable web apps.

---

## Quick Reference Cheat Sheet

```
CREATE & RUN
  dotnet new webapi -n MyApi      # scaffold (Spring Initializr)
  dotnet run                      # run (mvn spring-boot:run)
  dotnet add package <Name>       # add NuGet dep (Maven dependency)

PROGRAM.CS SHAPE
  var builder = WebApplication.CreateBuilder(args);
  builder.Services.Add...         # register beans / @Configuration
  var app = builder.Build();
  app.Use... / app.Map...         # middleware pipeline (filter chain, ORDER MATTERS)
  app.Run();                      # SpringApplication.run()

CONTROLLER SKELETON
  [ApiController]                 # @RestController behaviors
  [Route("api/[controller]")]     # @RequestMapping; [controller] = class minus "Controller"
  class XController : ControllerBase { ctor injection for services }

HTTP VERBS              ROUTING                       BINDING
  [HttpGet]              [Route("api/[controller]")]   [FromBody]   = @RequestBody
  [HttpPost]            [HttpGet("{id:int}")]          [FromRoute]  = @PathVariable
  [HttpPut]            constraints: :int :guid         [FromQuery]  = @RequestParam
  [HttpPatch]                       :min(n) :alpha      [FromHeader] = @RequestHeader
  [HttpDelete]                                          [FromForm]   = form/multipart

RESULTS (ControllerBase)            RETURN TYPES
  Ok(x)            -> 200            IActionResult        = ResponseEntity<?>
  CreatedAtAction  -> 201            ActionResult<T>      = ResponseEntity<T> (preferred)
  NoContent()      -> 204
  BadRequest(x)    -> 400
  NotFound()       -> 404
  StatusCode(500)  -> any

VALIDATION (DataAnnotations)         [ApiController] => auto 400 on invalid model
  [Required]            = @NotNull/@NotBlank
  [StringLength(n)]     = @Size
  [Range(a,b)]          = @Min/@Max
  [EmailAddress]        = @Email
  [RegularExpression]   = @Pattern
  ModelState.IsValid    = BindingResult

DI LIFETIMES
  AddSingleton  -> one per app
  AddScoped     -> one per request (default for web/DbContext)
  AddTransient  -> new each resolve

CONFIG
  appsettings.json                 = application.properties
  appsettings.{Env}.json           = application-{profile}.properties
  ASPNETCORE_ENVIRONMENT           = spring.profiles.active
  builder.Configuration["A:B"]     = @Value("${a.b}")
  services.Configure<T>(section)   = @ConfigurationProperties -> inject IOptions<T>
  dotnet user-secrets set k v      = local dev secrets
  Env var override: A__B (double underscore)

JSON (System.Text.Json) — camelCase by default
  [JsonPropertyName("x")] = @JsonProperty
  [JsonIgnore]            = @JsonIgnore
  AddControllers().AddJsonOptions(...)

HTTP CLIENT
  builder.Services.AddHttpClient<MyClient>(...)
  await http.GetFromJsonAsync<T>(url)     # restTemplate.getForObject (async)
  await http.PostAsJsonAsync(url, body)

LOGGING                              SWAGGER
  ILogger<T> via ctor                AddEndpointsApiExplorer(); AddSwaggerGen();
  _logger.LogInformation("{X}", x)   app.UseSwagger(); app.UseSwaggerUI();  -> /swagger

CORS / HTTPS / HEALTH
  AddCors(...); app.UseCors("name");
  app.UseHttpsRedirection();
  AddHealthChecks(); app.MapHealthChecks("/health");  # Actuator health
```

---

*Last Updated: 2026-06-16*
