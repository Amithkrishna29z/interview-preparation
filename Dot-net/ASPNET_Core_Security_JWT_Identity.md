# ASP.NET Core Security: JWT, Identity & Authorization

## Overview

This guide teaches **security in modern ASP.NET Core (.NET 8+)** for a developer who already knows **Spring Security and JWT** in the Java world. The goal: stop translating in your head and start mapping concepts directly. Almost everything you know from Spring has a near-exact twin in ASP.NET Core — the names differ, but the ideas are identical.

The two pillars of any secure web app are:

- **Authentication** — *"Who are you?"* — proving identity (login, tokens, cookies).
- **Authorization** — *"What are you allowed to do?"* — checking permissions (roles, policies, claims).

In Spring you wire these up with a `SecurityFilterChain`, a `UserDetailsService`, a `JwtAuthenticationFilter`, and annotations like `@PreAuthorize`. In ASP.NET Core you wire them up with **middleware**, **ASP.NET Core Identity**, **JWT Bearer authentication**, and **`[Authorize]` attributes + policies**. Same concepts, different spelling.

This guide is junior-friendly: every concept gets a plain-English analogy, a Spring comparison, and runnable C# code with inline comments.

---

## Table of Contents

- [Spring Security → ASP.NET Core Mapping](#spring-security--aspnet-core-mapping)
- [Authentication vs Authorization](#authentication-vs-authorization)
- [The Claims-Based Identity Model](#the-claims-based-identity-model)
- [Authentication Schemes & Middleware Order](#authentication-schemes--middleware-order)
- [JWT Authentication (The Big One)](#jwt-authentication-the-big-one)
- [Worked Example: A Real Login Endpoint Issuing a JWT](#worked-example-a-real-login-endpoint-issuing-a-jwt)
- [`[Authorize]` and `[AllowAnonymous]`](#authorize-and-allowanonymous)
- [Role-Based Authorization](#role-based-authorization)
- [Policy-Based Authorization](#policy-based-authorization)
- [Claims-Based Authorization](#claims-based-authorization)
- [ASP.NET Core Identity (UserManager & SignInManager)](#aspnet-core-identity-usermanager--signinmanager)
- [Cookie Authentication vs JWT](#cookie-authentication-vs-jwt)
- [Refresh Tokens](#refresh-tokens)
- [Password Hashing & Salting](#password-hashing--salting)
- [Common Security Concerns (HTTPS, CORS, CSRF, Secrets, OWASP)](#common-security-concerns)
- [OAuth2 / OIDC Integration (External Providers)](#oauth2--oidc-integration-external-providers)
- [Common Interview Questions](#common-interview-questions)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Spring Security → ASP.NET Core Mapping

| Spring Security (Java) | ASP.NET Core (.NET 8+) | Notes |
|---|---|---|
| `SecurityFilterChain` / `HttpSecurity` | `app.UseAuthentication()` + `app.UseAuthorization()` middleware | The pipeline that runs on every request |
| `UserDetailsService` | `UserManager<TUser>` / ASP.NET Core Identity | Loads and manages users |
| `UserDetails` | `IdentityUser` | The user entity |
| `@PreAuthorize` / `@Secured` / `@RolesAllowed` | `[Authorize]` attribute | Method/controller-level gate |
| `@PreAuthorize("hasRole('ADMIN')")` | `[Authorize(Roles = "Admin")]` | Role check |
| `@PreAuthorize("hasAuthority('SCOPE_read')")` | `[Authorize(Policy = "CanRead")]` | Fine-grained policy |
| `GrantedAuthority` / roles | `Claim` (esp. role claims) | Permissions attached to the user |
| `BCryptPasswordEncoder` | `IPasswordHasher<TUser>` (PBKDF2 by default) | Hashes/verifies passwords |
| `JwtAuthenticationFilter` (custom) | `.AddJwtBearer()` (built-in) | Validates JWT on each request |
| `JwtDecoder` / `JwtEncoder` | `JwtSecurityTokenHandler` / `JsonWebTokenHandler` | Reads/creates tokens |
| `SecurityContextHolder.getContext().getAuthentication()` | `HttpContext.User` (a `ClaimsPrincipal`) | The current logged-in user |
| `Authentication` object | `ClaimsPrincipal` | The authenticated identity |
| `Principal` | `ClaimsIdentity` / `ClaimsPrincipal.Identity` | The identity inside the principal |
| `AccessDeniedHandler` (403) | Returns `403 Forbidden` automatically | Authenticated but not allowed |
| `AuthenticationEntryPoint` (401) | Returns `401 Unauthorized` automatically | Not authenticated |
| `@EnableWebSecurity` config class | `Program.cs` service registration | Where you configure it all |
| `application.yml` security props | `appsettings.json` + User Secrets / env vars | Configuration |
| CSRF token (`CsrfToken`) | Antiforgery token (`IAntiforgery`) | Same idea, different name |
| `@CrossOrigin` / CORS config | `AddCors()` + `UseCors()` | CORS handling |

> **Mental model:** In Spring, security is a chain of *filters*. In ASP.NET Core, security is a pair of *middleware* (`UseAuthentication` then `UseAuthorization`). Both intercept the request before it reaches your controller.

---

## Authentication vs Authorization

**Authentication (AuthN)** = *proving who you are.*
**Authorization (AuthZ)** = *proving what you're allowed to do.*

**Think of it like...** an airport. **Authentication** is showing your passport at check-in — it proves you are who you claim to be. **Authorization** is your boarding pass — it says *which* flight, *which* seat, and whether you can enter the business-class lounge. You can be authenticated (valid passport) but not authorized (no business-class ticket).

| | Authentication | Authorization |
|---|---|---|
| Question | "Who are you?" | "Are you allowed?" |
| When | First | After authentication |
| Failure code | `401 Unauthorized` | `403 Forbidden` |
| Spring | `UsernamePasswordAuthenticationFilter` | `@PreAuthorize`, `.authorizeHttpRequests()` |
| ASP.NET | `UseAuthentication()` | `UseAuthorization()`, `[Authorize]` |

A common confusion: **`401 Unauthorized` actually means "unauthenticated"** (we don't know who you are), while **`403 Forbidden` means "authenticated but not authorized"** (we know you, but you can't do this). This is identical in Spring and ASP.NET Core.

---

## The Claims-Based Identity Model

This is the single most important concept to internalize. ASP.NET Core represents *everything about the current user* as a bag of **claims**.

A **Claim** is a key/value statement about the user: `name = "alice"`, `role = "Admin"`, `email = "alice@x.com"`, `department = "Sales"`. That's it — just a typed name/value pair.

The hierarchy:

```
ClaimsPrincipal              <- the whole user (HttpContext.User)
  └── ClaimsIdentity         <- one identity (e.g. from a JWT)
        ├── Claim (name)
        ├── Claim (role = Admin)
        └── Claim (email)
```

**Think of it like...** a wallet (`ClaimsPrincipal`). Inside, you might carry several ID cards (`ClaimsIdentity` — one from your government, one from your employer). Each card lists facts about you (`Claim` — name, photo, role). ASP.NET reads your "wallet" to decide what you can do.

### Spring comparison

| ASP.NET Core | Spring Security |
|---|---|
| `ClaimsPrincipal` | `Authentication` |
| `ClaimsIdentity` | `Principal` |
| `Claim` (role claim) | `GrantedAuthority` |
| `HttpContext.User` | `SecurityContextHolder.getContext().getAuthentication()` |

In Spring, a `GrantedAuthority` is essentially a single string permission ("ROLE_ADMIN"). In ASP.NET, a role is just a **claim** whose type is `ClaimTypes.Role`. Roles are a special case of claims — claims are the more general, more powerful concept.

```csharp
// Reading the current user inside a controller action.
// HttpContext.User is the ClaimsPrincipal — the .NET equivalent of
// SecurityContextHolder.getContext().getAuthentication() in Spring.

[ApiController]
[Route("api/[controller]")]
public class ProfileController : ControllerBase
{
    [HttpGet("me")]
    [Authorize] // must be authenticated
    public IActionResult Me()
    {
        // The currently authenticated user. `User` is shorthand for HttpContext.User.
        ClaimsPrincipal user = User;

        // Get a single claim value by type. Returns null if absent.
        string? userName = user.FindFirstValue(ClaimTypes.NameIdentifier); // like the "sub" / user id
        string? email    = user.FindFirstValue(ClaimTypes.Email);

        // IsInRole is the equivalent of Spring's hasRole(...) check.
        bool isAdmin = user.IsInRole("Admin");

        // Enumerate every claim (every "fact" about the user).
        var allClaims = user.Claims.Select(c => new { c.Type, c.Value });

        return Ok(new { userName, email, isAdmin, allClaims });
    }
}
```

---

## Authentication Schemes & Middleware Order

An **authentication scheme** is a named strategy for figuring out who the user is: "JwtBearer", "Cookies", "Google", etc. You register one (or more) and pick a default.

**Think of it like...** different doors into a building. One door checks your JWT badge, another checks a cookie wristband, another redirects you to Google's front desk. A "scheme" is one specific door with its own rules.

The **order of middleware matters** and is a classic source of bugs:

```csharp
var app = builder.Build();

app.UseHttpsRedirection(); // force HTTPS first

app.UseRouting();          // figures out WHICH endpoint matches

app.UseAuthentication();   // 1) WHO are you? (reads JWT/cookie, builds HttpContext.User)
app.UseAuthorization();    // 2) ARE you allowed? (enforces [Authorize])

app.MapControllers();      // finally, run the endpoint

app.Run();
```

Rules to memorize:
- `UseAuthentication()` **must come before** `UseAuthorization()`. You can't check permissions before you know who the user is. (Spring's filter chain enforces the same ordering internally.)
- Both must come **after** `UseRouting()` and **before** `MapControllers()`.
- `UseCors()` goes **before** `UseAuthentication()`.

If `[Authorize]` "doesn't work" (everyone gets in), 90% of the time it's because the two middleware are missing or in the wrong order.

---

## JWT Authentication (The Big One)

### What a JWT actually is

A **JWT (JSON Web Token)** is a self-contained, signed string with three parts separated by dots:

```
header.payload.signature
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhbGljZSIsInJvbGUiOiJBZG1pbiJ9.K3v...signature
```

- **Header** — algorithm & token type, e.g. `{ "alg": "HS256", "typ": "JWT" }`.
- **Payload** — the **claims**, e.g. `{ "sub": "alice", "role": "Admin", "exp": 1718500000 }`.
- **Signature** — header + payload signed with a secret/key. This is what makes it tamper-proof.

The first two parts are **Base64Url-encoded, not encrypted** — anyone can read them (paste into jwt.io). The *signature* guarantees nobody changed the contents. **Never put secrets/passwords in a JWT payload.**

**Think of it like...** a festival wristband with a tamper-evident seal. The wristband openly says "VIP, expires Sunday" (the payload — readable by anyone). The seal (signature) means staff can instantly tell if someone forged or altered it. They don't need to phone the box office (database) — the wristband proves itself. That's why JWT auth is **stateless**: the server doesn't store sessions.

This is exactly the Spring model: a custom `JwtAuthenticationFilter` reads the `Authorization: Bearer <token>` header, validates the signature, and populates the `SecurityContext`. In ASP.NET Core, the built-in **JWT Bearer middleware** does all of that for you.

### Registering JWT Bearer authentication

```csharp
// Program.cs
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// Pull JWT settings from appsettings.json / user secrets / env vars (never hardcode!)
var jwtKey      = builder.Configuration["Jwt:Key"]!;       // the signing secret
var jwtIssuer   = builder.Configuration["Jwt:Issuer"]!;    // who issued the token
var jwtAudience = builder.Configuration["Jwt:Audience"]!;  // who the token is for

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme) // default scheme = "Bearer"
    .AddJwtBearer(options =>
    {
        // This is the .NET equivalent of configuring Spring's JwtDecoder +
        // the validation rules you'd normally write in JwtAuthenticationFilter.
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer           = true,           // check the "iss" claim
            ValidIssuer              = jwtIssuer,

            ValidateAudience         = true,           // check the "aud" claim
            ValidAudience            = jwtAudience,

            ValidateLifetime         = true,           // reject expired tokens ("exp")
            ClockSkew                = TimeSpan.Zero,  // no grace period (default is 5 min)

            ValidateIssuerSigningKey = true,           // verify the signature
            IssuerSigningKey         = new SymmetricSecurityKey(
                                          Encoding.UTF8.GetBytes(jwtKey))
        };
    });

builder.Services.AddAuthorization(); // enables [Authorize]

var app = builder.Build();
app.UseAuthentication();  // reads & validates the Bearer token -> builds HttpContext.User
app.UseAuthorization();
app.MapControllers();
app.Run();
```

```json
// appsettings.json — settings ONLY. The real Key should live in User Secrets / env vars.
{
  "Jwt": {
    "Issuer": "https://myapi.example.com",
    "Audience": "https://myapi.example.com",
    "Key": "REPLACE_VIA_USER_SECRETS"   // placeholder — never commit the real one
  }
}
```

Now any endpoint marked `[Authorize]` requires a valid `Authorization: Bearer <token>` header. The middleware verifies the signature, issuer, audience, and expiry — exactly the checks you'd hand-write in a Spring `JwtAuthenticationFilter`.

---

## Worked Example: A Real Login Endpoint Issuing a JWT

This is the part interviewers love. Here is a complete login flow that verifies a password and **issues** a signed JWT using `JwtSecurityTokenHandler`.

```csharp
using Microsoft.IdentityModel.Tokens;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;

public class TokenService
{
    private readonly IConfiguration _config;
    public TokenService(IConfiguration config) => _config = config;

    public string CreateToken(string userId, string userName, IEnumerable<string> roles)
    {
        // 1) Build the claims — the "facts" that go INTO the token payload.
        //    Same idea as the claims you'd set when building a Spring JWT.
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, userId),                 // "sub" = subject (user id)
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()), // unique token id
            new(ClaimTypes.Name, userName)
        };
        // Add one role claim per role -> becomes hasRole(...) / [Authorize(Roles=...)]
        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        // 2) Build the signing credentials from the secret key (HMAC-SHA256).
        var key   = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_config["Jwt:Key"]!));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        // 3) Describe the token: issuer, audience, claims, expiry, signature.
        var token = new JwtSecurityToken(
            issuer:             _config["Jwt:Issuer"],
            audience:           _config["Jwt:Audience"],
            claims:             claims,
            expires:            DateTime.UtcNow.AddMinutes(15),  // short-lived (use refresh tokens)
            signingCredentials: creds);

        // 4) Serialize to the compact header.payload.signature string.
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

```csharp
// The login endpoint. Verifies credentials, then hands back a JWT.
[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly UserManager<IdentityUser> _userManager; // ASP.NET Core Identity
    private readonly TokenService _tokenService;

    public AuthController(UserManager<IdentityUser> userManager, TokenService tokenService)
    {
        _userManager = userManager;
        _tokenService = tokenService;
    }

    [HttpPost("login")]
    [AllowAnonymous] // login must be reachable without a token
    public async Task<IActionResult> Login([FromBody] LoginDto dto)
    {
        // Find the user (UserManager == Spring's UserDetailsService).
        var user = await _userManager.FindByNameAsync(dto.UserName);
        if (user is null)
            return Unauthorized("Invalid credentials"); // don't reveal which part was wrong

        // Verify the password against the stored hash (uses Identity's password hasher,
        // the equivalent of BCryptPasswordEncoder.matches(...) in Spring).
        if (!await _userManager.CheckPasswordAsync(user, dto.Password))
            return Unauthorized("Invalid credentials");

        // Load roles and mint the token.
        var roles = await _userManager.GetRolesAsync(user);
        var token = _tokenService.CreateToken(user.Id, user.UserName!, roles);

        return Ok(new { token });
    }
}

public record LoginDto(string UserName, string Password);
```

> **Note on the handler classes:** `JwtSecurityTokenHandler` (from `System.IdentityModel.Tokens.Jwt`) is the classic API. The newer `JsonWebTokenHandler` (from `Microsoft.IdentityModel.JsonWebTokens`) is faster and recommended for high-throughput services. Both produce/validate the same tokens; pick `JsonWebTokenHandler` for new high-perf code.

---

## `[Authorize]` and `[AllowAnonymous]`

`[Authorize]` is the direct equivalent of Spring's `@PreAuthorize` / `@Secured`. Put it on a controller or action to require authentication.

```csharp
[Authorize]                 // every action here requires a logged-in user
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    [HttpGet]
    public IActionResult GetMyOrders() => Ok(/* ... */); // requires auth (inherited)

    [HttpGet("public-info")]
    [AllowAnonymous]        // OVERRIDES the class-level [Authorize] — open to everyone
    public IActionResult PublicInfo() => Ok("anyone can see this");
}
```

**Think of it like...** `[Authorize]` is a locked door; `[AllowAnonymous]` is propping one specific door open even though the whole building is locked. `[AllowAnonymous]` always wins over `[Authorize]`. Common use: a controller is `[Authorize]`, but `login`/`register` are `[AllowAnonymous]`.

You can also require auth **globally** so you don't forget:

```csharp
// Every endpoint requires auth by default unless it has [AllowAnonymous].
builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build());
```

---

## Role-Based Authorization

Roles are the simplest authorization check — same as Spring's `hasRole('ADMIN')`.

```csharp
// Spring: @PreAuthorize("hasRole('ADMIN')")
[Authorize(Roles = "Admin")]
[HttpDelete("{id}")]
public IActionResult Delete(int id) => Ok();

// Multiple roles, OR semantics (Admin OR Manager can enter).
// Spring: @PreAuthorize("hasAnyRole('ADMIN','MANAGER')")
[Authorize(Roles = "Admin,Manager")]
[HttpPut("{id}")]
public IActionResult Update(int id) => Ok();

// AND semantics: stack the attributes (must be BOTH Admin AND in HR).
[Authorize(Roles = "Admin")]
[Authorize(Roles = "HR")]
[HttpGet("salaries")]
public IActionResult Salaries() => Ok();
```

For roles to work, the JWT must contain **role claims** (`ClaimTypes.Role`, which maps to the `"role"` claim in the token) — exactly what `CreateToken` above does.

---

## Policy-Based Authorization

Roles get you only so far. **Policies** are ASP.NET Core's flexible, reusable authorization rules — the direct counterpart to Spring's expression-based `@PreAuthorize("...")` and custom `AccessDecisionVoter`s.

A **policy** = a named set of **requirements**. A **requirement** is satisfied by one or more **handlers**.

**Think of it like...** a nightclub guest list (a *policy*). The bouncer (the *handler*) checks the rules (the *requirements*): "must be 21+ AND on the VIP list". Different doors (endpoints) can reuse the same named guest list.

### Simple inline policies

```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    // Spring: @PreAuthorize("hasRole('ADMIN')")
    options.AddPolicy("AdminOnly", p => p.RequireRole("Admin"));

    // Spring: @PreAuthorize("hasAuthority('department:Sales')")
    options.AddPolicy("SalesDept", p => p.RequireClaim("department", "Sales"));

    // Combine requirements (AND).
    options.AddPolicy("SeniorSales", p =>
        p.RequireClaim("department", "Sales")
         .RequireClaim("seniority", "Senior"));
});
```

```csharp
// Apply by name — like referencing a named SpEL expression.
[Authorize(Policy = "AdminOnly")]
[HttpGet("admin-dashboard")]
public IActionResult Dashboard() => Ok();
```

### Custom requirement + handler (e.g. minimum age)

This is the equivalent of writing a custom Spring `AuthorizationManager` / `@PreAuthorize` SpEL function.

```csharp
// 1) The requirement = the "rule" (just data).
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }
    public MinimumAgeRequirement(int minimumAge) => MinimumAge = minimumAge;
}

// 2) The handler = the "bouncer" that decides if the rule is met.
public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        // Look for a "dateofbirth" claim on the current user (context.User == ClaimsPrincipal).
        var dobClaim = context.User.FindFirst(c => c.Type == "dateofbirth");
        if (dobClaim is null) return Task.CompletedTask; // no claim -> requirement NOT met

        var dob = DateTime.Parse(dobClaim.Value);
        var age = DateTime.Today.Year - dob.Year;
        if (dob > DateTime.Today.AddYears(-age)) age--; // adjust if birthday hasn't happened

        if (age >= requirement.MinimumAge)
            context.Succeed(requirement); // PASS — mark the requirement satisfied

        return Task.CompletedTask;        // not succeeding == fail
    }
}
```

```csharp
// 3) Register the handler and the policy.
builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AtLeast18", p => p.Requirements.Add(new MinimumAgeRequirement(18)));
});

// 4) Use it.
[Authorize(Policy = "AtLeast18")]
[HttpGet("adult-content")]
public IActionResult AdultContent() => Ok();
```

---

## Claims-Based Authorization

The most granular check: require a specific claim value. This is what policies often wrap.

```csharp
// Inline (most common):
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanEditArticles",
        p => p.RequireClaim("permission", "articles:edit"));
});

[Authorize(Policy = "CanEditArticles")]
[HttpPut("articles/{id}")]
public IActionResult Edit(int id) => Ok();
```

```csharp
// Imperative check inside an action (when the rule is too dynamic for an attribute):
[HttpGet("doc/{id}")]
[Authorize]
public IActionResult GetDoc(int id)
{
    // Spring equivalent: authentication.getAuthorities().contains(...)
    if (!User.HasClaim("permission", "docs:read"))
        return Forbid(); // returns 403

    return Ok(/* the doc */);
}
```

**Why claims over roles?** Roles are coarse ("Admin"). Claims model fine-grained permissions ("can:refund", "tenant:42", "plan:pro"). A senior tip: prefer **policies built on claims** for real apps; reserve roles for broad buckets.

---

## ASP.NET Core Identity (UserManager & SignInManager)

**ASP.NET Core Identity** is the full membership system: user storage, password hashing, roles, lockout, email confirmation, 2FA. It is the all-in-one equivalent of Spring Security's `UserDetailsService` + `PasswordEncoder` + user/role tables, but batteries-included.

**Think of it like...** a pre-built HR department. Spring gives you the interfaces and you implement `UserDetailsService` + pick `BCryptPasswordEncoder`. Identity hands you a fully staffed department (`UserManager`, `SignInManager`, `RoleManager`) wired to a database.

| Identity class | Spring equivalent | Job |
|---|---|---|
| `IdentityUser` | `UserDetails` | The user entity |
| `IdentityDbContext` | JPA `@Entity` user table | EF Core DB context with user/role tables |
| `UserManager<T>` | `UserDetailsService` (+ create/update) | CRUD users, check passwords, manage roles |
| `SignInManager<T>` | `AuthenticationManager` | Sign in/out, cookie issuance, lockout |
| `RoleManager<T>` | role admin | CRUD roles |
| `IPasswordHasher<T>` | `BCryptPasswordEncoder` | Hash/verify passwords (PBKDF2 default) |

### Setup

```csharp
// 1) DB context inherits IdentityDbContext -> gives you AspNetUsers, AspNetRoles, etc.
public class AppDbContext : IdentityDbContext<IdentityUser>
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
}

// 2) Program.cs — register EF + Identity.
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services
    .AddIdentityCore<IdentityUser>(options =>
    {
        // Password policy — the equivalent of configuring a Spring PasswordEncoder + validators.
        options.Password.RequiredLength = 8;
        options.Password.RequireDigit = true;
        options.Password.RequireUppercase = true;
        // Lockout after repeated failures (brute-force protection).
        options.Lockout.MaxFailedAccessAttempts = 5;
    })
    .AddRoles<IdentityRole>()          // enables RoleManager
    .AddEntityFrameworkStores<AppDbContext>(); // store users in EF Core
```

### Registering a user

```csharp
[HttpPost("register")]
[AllowAnonymous]
public async Task<IActionResult> Register(RegisterDto dto)
{
    var user = new IdentityUser { UserName = dto.UserName, Email = dto.Email };

    // CreateAsync hashes the password (salted PBKDF2) and saves the user.
    // Spring equivalent: encode the password with BCrypt, then save via repository.
    var result = await _userManager.CreateAsync(user, dto.Password);

    if (!result.Succeeded)
        return BadRequest(result.Errors); // password policy violations, duplicate user, etc.

    await _userManager.AddToRoleAsync(user, "User"); // assign default role
    return Ok();
}
```

`UserManager` handles hashing, salting, uniqueness, and validation for you — you never store a plaintext password.

---

## Cookie Authentication vs JWT

Two ways to "remember" a logged-in user. Same dichotomy exists in Spring (session cookie vs bearer token).

| | Cookie Auth | JWT (Bearer) |
|---|---|---|
| State | Server may track session; cookie is opaque | Stateless — all info in the token |
| Sent how | Browser auto-sends cookie | Client manually sets `Authorization: Bearer` header |
| Best for | Server-rendered web apps (MVC, Razor) | APIs, SPAs, mobile, microservices |
| CSRF risk | Yes (auto-sent) — needs antiforgery | No (not auto-sent) if stored correctly |
| Revocation | Easy (drop server session) | Hard (token valid until expiry) — needs blacklist/refresh |
| Scaling | Needs shared session store across servers | Trivial — any server can validate the signature |

**Think of it like...** a cookie is a **coat-check ticket**: small and opaque; the cloakroom (server) holds your coat and looks it up. A JWT is a **printed receipt** that itself lists everything you bought; no lookup needed, but you can't "cancel" it after it's printed.

**Rule of thumb:** building a JSON API / SPA / mobile backend → **JWT**. Building a traditional server-rendered website with login pages → **cookies** (use `SignInManager.PasswordSignInAsync`, which issues the auth cookie for you).

```csharp
// Cookie scheme registration (for server-rendered apps).
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(options =>
    {
        options.LoginPath = "/account/login";       // redirect unauthenticated users here
        options.ExpireTimeSpan = TimeSpan.FromHours(8);
        options.SlidingExpiration = true;           // refresh the window on activity
        options.Cookie.HttpOnly = true;             // JS can't read it (XSS protection)
        options.Cookie.SecurePolicy = CookieSecurePolicy.Always; // HTTPS only
    });
```

---

## Refresh Tokens

A JWT should be **short-lived** (e.g. 15 min) so a stolen token expires fast. But you don't want users logging in every 15 minutes. The fix: a **refresh token**.

**Think of it like...** a movie ticket (access token, JWT) that's only valid for one showing, plus a **season pass** (refresh token) you keep safe at home. When the movie ticket expires, you show the season pass at the desk to get a fresh ticket — without re-buying.

The pattern:
1. On login, return **two** tokens: a short-lived **access token (JWT)** and a long-lived **refresh token** (an opaque random string stored in your DB, tied to the user).
2. Client uses the access token until it expires (gets a `401`).
3. Client calls `/refresh` with the refresh token. Server validates it against the DB, issues a **new** access token (and usually rotates the refresh token).
4. On logout (or theft), you **delete the refresh token** from the DB — instant revocation, which plain JWTs can't do.

```csharp
public class RefreshToken
{
    public string Token { get; set; } = default!;   // cryptographically random opaque string
    public string UserId { get; set; } = default!;
    public DateTime ExpiresUtc { get; set; }         // e.g. 7 days
    public bool IsRevoked { get; set; }
}

[HttpPost("refresh")]
[AllowAnonymous]
public async Task<IActionResult> Refresh([FromBody] string refreshToken)
{
    var stored = await _db.RefreshTokens
        .SingleOrDefaultAsync(t => t.Token == refreshToken);

    // Validate: exists, not expired, not revoked.
    if (stored is null || stored.IsRevoked || stored.ExpiresUtc < DateTime.UtcNow)
        return Unauthorized("Invalid refresh token");

    stored.IsRevoked = true; // rotate: invalidate the old one
    var user  = await _userManager.FindByIdAsync(stored.UserId);
    var roles = await _userManager.GetRolesAsync(user!);

    var newAccess  = _tokenService.CreateToken(user!.Id, user.UserName!, roles);
    var newRefresh = GenerateAndStoreRefreshToken(user.Id); // new random string -> DB

    await _db.SaveChangesAsync();
    return Ok(new { accessToken = newAccess, refreshToken = newRefresh });
}
```

Store refresh tokens **hashed** in the DB (like passwords) and ideally in an `HttpOnly` cookie on the client.

---

## Password Hashing & Salting

You **never** store plaintext passwords. You store a **salted hash**.

- **Hashing** is a one-way function: `hash("p@ss")` → `a8f3...`. You can't reverse it.
- **Salt** is random data mixed in before hashing, so two users with the same password get *different* hashes. This defeats rainbow tables.

**Think of it like...** shredding a document (hashing — you can't un-shred). The **salt** is a unique colored confetti added to each person's document before shredding, so identical documents produce different-looking shreds.

ASP.NET Core Identity does this automatically via `IPasswordHasher<T>` using **PBKDF2** with a per-user salt and many iterations. Spring's equivalent is `BCryptPasswordEncoder` (BCrypt also salts internally). Both are "slow by design" to resist brute force.

```csharp
// You almost never call this directly — UserManager does it.
var user = new IdentityUser { UserName = "alice" };
var hasher = new PasswordHasher<IdentityUser>();

// Hash (on register). Salt is generated internally and embedded in the output string.
string hashed = hasher.HashPassword(user, "P@ssw0rd!");

// Verify (on login). Equivalent to BCryptPasswordEncoder.matches(raw, encoded).
var result = hasher.VerifyHashedPassword(user, hashed, "P@ssw0rd!");
// result == PasswordVerificationResult.Success
```

---

## Common Security Concerns

### HTTPS
Always force HTTPS so tokens/cookies can't be sniffed.

```csharp
app.UseHttpsRedirection();          // redirect HTTP -> HTTPS
app.UseHsts();                       // tell browsers to always use HTTPS (production only)
```

### CORS (Cross-Origin Resource Sharing)
Controls **which other websites' JS** may call your API. Same concept as Spring's `@CrossOrigin` / `CorsConfiguration`. Browsers block cross-origin calls unless you allow them.

```csharp
builder.Services.AddCors(o => o.AddPolicy("Spa", p =>
    p.WithOrigins("https://myapp.com")  // never AllowAnyOrigin + AllowCredentials together
     .AllowAnyHeader()
     .AllowAnyMethod()));

app.UseCors("Spa"); // BEFORE UseAuthentication
```

### CSRF / Antiforgery
A malicious site tricks the browser into making an authenticated request using its **auto-sent cookie**. **This only affects cookie auth, not JWT** (the browser doesn't auto-send the `Authorization` header). For cookie-based forms, ASP.NET Core's `IAntiforgery` issues a token (the analog of Spring's `CsrfToken`); Razor's `<form>` includes it automatically. For pure JWT APIs, CSRF is generally a non-issue.

### Secrets Management
**Never commit secrets** (JWT keys, connection strings, API keys) to source control.

- **Development:** User Secrets — `dotnet user-secrets set "Jwt:Key" "supersecret"`. Stored outside the repo.
- **Production:** environment variables or a secret store (Azure Key Vault, AWS Secrets Manager).
- Configuration is layered: `appsettings.json` < User Secrets < environment variables, with later sources overriding earlier — so prod env vars win.

### OWASP Basics
The OWASP Top 10 is the industry checklist (injection, broken auth, XSS, etc.). Key defenses: validate input, use parameterized queries, hash passwords, enforce HTTPS, apply least-privilege authorization, keep dependencies patched.

### SQL Injection
**Entity Framework Core parameterizes queries automatically**, so LINQ and properly-used `FromSql` are safe by default. The danger is **string concatenation** into raw SQL.

```csharp
// SAFE: EF parameterizes the value. Injection-proof.
var users = db.Users.Where(u => u.Name == userInput).ToList();

// SAFE: FromSqlInterpolated treats {userInput} as a parameter, not literal SQL.
var u = db.Users.FromSqlInterpolated($"SELECT * FROM Users WHERE Name = {userInput}").ToList();

// DANGER: never build SQL by string concatenation -> SQL injection!
// db.Users.FromSqlRaw("SELECT * FROM Users WHERE Name = '" + userInput + "'");
```

### Data Protection API
ASP.NET Core's built-in `IDataProtector` encrypts sensitive data (auth cookies, antiforgery tokens, password-reset tokens) with automatically-managed, rotating keys. You rarely call it directly — the framework uses it under the hood — but it's why cookies/tokens are tamper-proof out of the box.

```csharp
// Manual use, e.g. to protect a value you put in a URL.
var protector = _provider.CreateProtector("ResetTokens");
string protectedText   = protector.Protect("user=42");   // encrypted + signed
string original        = protector.Unprotect(protectedText);
```

---

## OAuth2 / OIDC Integration (External Providers)

**OAuth2** is an authorization *delegation* protocol ("let this app access X on my behalf"). **OIDC (OpenID Connect)** sits on top of OAuth2 to add *authentication* ("log in with Google/Microsoft"). This is exactly Spring Security's `oauth2Login()`.

**Think of it like...** "Sign in with Google." You never give your Google password to the third-party app. Instead Google vouches for you with a signed **ID token** (itself a JWT). The app trusts Google, so it trusts you.

```csharp
// Add external login alongside your normal scheme.
builder.Services.AddAuthentication()
    .AddGoogle(options =>
    {
        // Client id/secret come from the Google Cloud console — keep in User Secrets!
        options.ClientId     = builder.Configuration["Google:ClientId"]!;
        options.ClientSecret = builder.Configuration["Google:ClientSecret"]!;
    })
    .AddOpenIdConnect("MicrosoftOidc", options =>
    {
        options.Authority    = "https://login.microsoftonline.com/<tenant>/v2.0";
        options.ClientId     = builder.Configuration["Ms:ClientId"]!;
        options.ClientSecret = builder.Configuration["Ms:ClientSecret"]!;
        options.ResponseType = "code"; // Authorization Code flow (the secure one)
    });
```

The flow (Authorization Code): your app redirects to the provider → user logs in there → provider redirects back with a code → your app exchanges the code for an ID token + access token → ASP.NET builds a `ClaimsPrincipal` from the token's claims. Conceptually identical to Spring's OAuth2 client. As a junior, know *what it is and when to use it* (offload login to Google/Microsoft/Okta); you rarely implement the protocol by hand.

---

## Common Interview Questions

**1. What's the difference between authentication and authorization?**
Authentication = "who are you?" (proving identity, → 401 if it fails). Authorization = "what can you do?" (checking permissions, → 403 if it fails). AuthN always runs first. In ASP.NET that's `UseAuthentication()` then `UseAuthorization()`.

**2. What is a JWT and what are its three parts?**
A JSON Web Token: `header.payload.signature`, Base64Url-encoded. Header = algorithm/type, Payload = claims (incl. `exp`, `iss`, `aud`), Signature = the signed hash proving integrity. The payload is **readable but tamper-proof** — never put secrets in it. It enables **stateless** auth because the token carries its own claims.

**3. Why does `[Authorize]` sometimes "not work"?**
Almost always middleware order/missing middleware: `UseAuthentication()` must come *before* `UseAuthorization()`, and both after `UseRouting()` and before `MapControllers()`. Without `UseAuthentication`, `HttpContext.User` is never populated, so authorization can't see the user.

**4. Explain the claims-based identity model. How does it map to Spring?**
`HttpContext.User` is a `ClaimsPrincipal` (≈ Spring `Authentication`) containing one or more `ClaimsIdentity` (≈ `Principal`), each holding `Claim`s (key/value facts; role claims ≈ `GrantedAuthority`). Roles in ASP.NET are just claims of type `ClaimTypes.Role`.

**5. Role-based vs policy-based vs claims-based authorization — when do you use each?**
Roles = coarse buckets (`[Authorize(Roles="Admin")]`). Claims = fine-grained facts (`RequireClaim("permission","x")`). Policies = named, reusable rules combining requirements + handlers, and can express arbitrary logic (like Spring SpEL in `@PreAuthorize`). Prefer claims/policies for real-world fine-grained control.

**6. How do you validate a JWT? What does the middleware check?**
Via `TokenValidationParameters`: the **signature** (using the signing key), **issuer** (`iss`), **audience** (`aud`), and **lifetime** (`exp`, with `ClockSkew`). `AddJwtBearer` does all this per request — the equivalent of a custom Spring `JwtAuthenticationFilter`/`JwtDecoder`.

**7. Cookie auth vs JWT — when would you pick each?**
Cookies for server-rendered apps (stateful-ish, browser auto-sends them, but CSRF-prone). JWT for APIs/SPAs/mobile/microservices (stateless, scales trivially, no CSRF since not auto-sent, but hard to revoke before expiry).

**8. Why use refresh tokens?**
Access tokens (JWTs) are short-lived to limit damage if stolen; a refresh token (long-lived, opaque, stored server-side) lets the client get new access tokens without re-login. Crucially, deleting the refresh token from the DB gives you **revocation**, which stateless JWTs lack.

**9. How does ASP.NET Core Identity store passwords? Compare to Spring.**
Identity hashes passwords with **PBKDF2** plus a **per-user salt** and many iterations, via `IPasswordHasher<T>` — never plaintext. Spring's equivalent is `BCryptPasswordEncoder`. Both are intentionally slow to resist brute force; salting defeats rainbow tables.

**10. Is CSRF a concern for JWT APIs? Why or why not?**
Generally no. CSRF exploits the browser **auto-sending cookies**. A JWT in the `Authorization` header is set manually by client JS and isn't auto-sent cross-site, so CSRF doesn't apply. CSRF/antiforgery matters mainly for **cookie-based** auth.

**11. How does Entity Framework protect against SQL injection?**
EF Core **parameterizes** all LINQ queries and `FromSqlInterpolated`/`FromSql` calls — user input becomes a SQL parameter, never concatenated literal SQL. Injection only sneaks in if you build raw SQL strings yourself (e.g. `FromSqlRaw` with concatenation).

**12. Where should secrets like the JWT signing key live?**
Never in source control or `appsettings.json` committed to git. Use **User Secrets** in development and **environment variables / a key vault** in production. Config layering ensures env vars override file values.

**13. What's the difference between `[Authorize]` and `[AllowAnonymous]`?**
`[Authorize]` requires authentication (optionally a role/policy). `[AllowAnonymous]` opens an endpoint even if the controller (or a global fallback policy) requires auth — it always wins. Used for login/register endpoints.

**14. What is OIDC and how does "Sign in with Google" work?**
OIDC = OpenID Connect, an authentication layer over OAuth2. The Authorization Code flow redirects the user to the provider, who authenticates them and returns a code; your app swaps it for an **ID token** (a JWT) and builds a `ClaimsPrincipal`. You never see the user's password. ≈ Spring's `oauth2Login()`.

---

## Quick Reference Cheat Sheet

```text
AUTHN vs AUTHZ
  Authentication = who are you?   -> 401 Unauthorized (really "unauthenticated")
  Authorization  = what can you?  -> 403 Forbidden

MIDDLEWARE ORDER (Program.cs)  *** order matters ***
  UseHttpsRedirection -> UseRouting -> UseCors
  -> UseAuthentication -> UseAuthorization -> MapControllers

CLAIMS MODEL
  HttpContext.User : ClaimsPrincipal   (~ Spring Authentication)
    -> ClaimsIdentity                  (~ Spring Principal)
         -> Claim(type, value)         (role claim ~ GrantedAuthority)
  Read: User.FindFirstValue(ClaimTypes.X) | User.IsInRole("Admin") | User.HasClaim(t,v)

JWT = header.payload.signature  (Base64Url, signed not encrypted)
  Validate w/ TokenValidationParameters:
    ValidateIssuer / ValidateAudience / ValidateLifetime / ValidateIssuerSigningKey
  Register:  AddAuthentication("Bearer").AddJwtBearer(o => o.TokenValidationParameters = ...)
  Issue:     new JwtSecurityToken(issuer, audience, claims, expires, signingCredentials)
             new JwtSecurityTokenHandler().WriteToken(token)
  Header:    Authorization: Bearer <token>

ATTRIBUTES
  [Authorize]                       require auth
  [AllowAnonymous]                  open (wins over [Authorize])
  [Authorize(Roles="Admin,Manager")] role check (OR); stack attrs for AND
  [Authorize(Policy="Name")]        policy check

POLICIES (Program.cs: AddAuthorization)
  AddPolicy("AdminOnly", p => p.RequireRole("Admin"))
  AddPolicy("Sales",     p => p.RequireClaim("department","Sales"))
  Custom: IAuthorizationRequirement + AuthorizationHandler<T> (context.Succeed(req))

IDENTITY
  IdentityUser (~UserDetails) | UserManager (~UserDetailsService) | SignInManager
  RoleManager | IPasswordHasher (PBKDF2, salted) ~ BCryptPasswordEncoder
  Register: CreateAsync(user, pw)   Login: CheckPasswordAsync(user, pw)

COOKIE vs JWT
  Cookie -> server-rendered apps, CSRF-prone, easy revoke
  JWT    -> APIs/SPAs/mobile, stateless, scales, hard to revoke (use refresh tokens)

REFRESH TOKEN
  short JWT (15m) + long opaque refresh token (DB, hashed). /refresh -> new JWT + rotate.
  delete from DB = revocation.

SECURITY CHECKLIST
  HTTPS (UseHttpsRedirection/UseHsts) | CORS (WithOrigins) | CSRF (cookie auth only)
  Secrets: user-secrets (dev) / env vars / key vault (prod) — never commit
  EF parameterizes -> SQL-injection safe (avoid FromSqlRaw + concat)
  Data Protection API encrypts cookies/tokens automatically

SPRING -> ASP.NET QUICK MAP
  SecurityFilterChain -> Use/Authentication+Authorization middleware
  UserDetailsService  -> UserManager / Identity
  @PreAuthorize       -> [Authorize] / Policy
  GrantedAuthority    -> Claim (role)
  BCryptPasswordEncoder -> IPasswordHasher (PBKDF2)
  JwtAuthenticationFilter -> AddJwtBearer
  SecurityContextHolder -> HttpContext.User
```

---

*Last Updated: 2026-06-16*
