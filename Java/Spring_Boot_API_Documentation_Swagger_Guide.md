# Spring Boot API Documentation with springdoc-openapi / Swagger UI

## Overview

API documentation is how you tell the rest of the team — and your future self — what your endpoints do. Good docs let frontend developers call your API without reading your source code, let QA test interactively without Postman, and act as a living contract between services. This guide covers the only correct tool for Spring Boot 3: **springdoc-openapi**. If you reach for SpringFox, stop — it is abandoned and incompatible with Spring Boot 3.

---

## Table of Contents

1. [OpenAPI vs Swagger UI](#openapi-vs-swagger-ui)
2. [Why Not SpringFox?](#why-not-springfox)
3. [Setup & Auto-Discovery](#setup--auto-discovery)
4. [Enriching Docs with Annotations](#enriching-docs-with-annotations)
5. [Documented Controller Example](#documented-controller-example)
6. [Documenting DTOs with @Schema](#documenting-dtos-with-schema)
7. [Global API Info Bean](#global-api-info-bean)
8. [JWT Bearer Security Scheme](#jwt-bearer-security-scheme)
9. [Customizing Paths & Disabling in Production](#customizing-paths--disabling-in-production)
10. [Common Mistakes & Pitfalls](#common-mistakes--pitfalls)
11. [Common Interview Questions](#common-interview-questions)
12. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## OpenAPI vs Swagger UI

These two terms are often confused.

| Term | What it is |
|---|---|
| **OpenAPI Specification** | A language-agnostic JSON/YAML **standard** describing your API — endpoints, request/response shapes, auth schemes. Think of it as a blueprint. |
| **Swagger UI** | An interactive **web page** that reads that blueprint and renders a clickable, try-it-now interface. |
| **springdoc-openapi** | A Spring library that **generates** the OpenAPI spec by scanning your `@RestController` classes at startup, then serves both the raw JSON and the Swagger UI. |

> **Analogy**: OpenAPI is the architectural blueprint; Swagger UI is the 3-D model that clients walk through.

**Why it matters in practice:**
- **Frontend devs** see every endpoint, required fields, and response shapes without hunting through source code.
- **QA / manual testing** can fire real HTTP requests straight from the browser.
- **Service contracts** — the JSON spec at `/v3/api-docs` can be imported into Postman, code-generated into client SDKs, or validated in CI.

---

## Why Not SpringFox?

SpringFox was the de-facto standard for years, but:

- Its last release was **2020**. It is effectively **abandoned**.
- It is **incompatible with Spring Boot 3** and Spring Framework 6 (fails with `NullPointerException` on startup due to Spring MVC path-matching changes).
- Reaching for SpringFox in an interview signals outdated knowledge.

**Always use `springdoc-openapi` for Spring Boot 3.**

---

## Setup & Auto-Discovery

Add one dependency — that is all that is required for a working Swagger UI.

```xml
<!-- Maven — Spring Boot 3.x -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.5.0</version>  <!-- use latest 2.x -->
</dependency>
```

```groovy
// Gradle
implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.5.0'
```

After adding the dependency and restarting, two URLs are live automatically:

| URL | What you get |
|---|---|
| `/swagger-ui.html` | Interactive Swagger UI |
| `/v3/api-docs` | Raw OpenAPI JSON spec |
| `/v3/api-docs.yaml` | Raw OpenAPI YAML spec |

**No `@EnableSwagger2` or any configuration class is needed.** springdoc scans every `@RestController` and builds the spec automatically.

> **Interview Tip**: The artifact name changed between major versions. Spring Boot 2 used `springdoc-openapi-ui`. Spring Boot 3 uses `springdoc-openapi-starter-webmvc-ui`. Using the wrong one is a common setup mistake.

---

## Enriching Docs with Annotations

Zero-config gives you something, but annotations make docs clear and useful.

| Annotation | Where it goes | What it does |
|---|---|---|
| `@Tag(name, description)` | Controller class | Groups all endpoints under a named section |
| `@Operation(summary, description)` | Controller method | Describes what an endpoint does |
| `@ApiResponse(responseCode, description)` | Controller method | Documents one possible HTTP response |
| `@ApiResponses({...})` | Controller method | Groups multiple `@ApiResponse` entries |
| `@Parameter(description, required)` | Method parameter | Describes a path/query/header param |
| `@Schema(description, example)` | DTO field | Documents a request/response field |

All annotations come from the `io.swagger.v3.oas.annotations` package.

---

## Documented Controller Example

```java
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;

@Tag(name = "Products", description = "CRUD operations for the product catalogue")
@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @Operation(
        summary = "Get product by ID",
        description = "Returns a single product. Returns 404 if the product does not exist."
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Product found"),
        @ApiResponse(responseCode = "404", description = "Product not found"),
        @ApiResponse(responseCode = "401", description = "Unauthorized — JWT required")
    })
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getById(
            @Parameter(description = "Numeric product ID", required = true)
            @PathVariable Long id) {
        return ResponseEntity.ok(productService.findById(id));
    }

    @Operation(summary = "Create a new product")
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "Product created"),
        @ApiResponse(responseCode = "400", description = "Validation error in request body"),
        @ApiResponse(responseCode = "401", description = "Unauthorized — JWT required")
    })
    @PostMapping
    public ResponseEntity<ProductResponse> create(@Valid @RequestBody CreateProductRequest request) {
        ProductResponse created = productService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
}
```

> **Interview Tip**: Always document error responses (`400`, `401`, `404`). A Swagger UI that only shows 200 is half the story and misleads frontend developers.

---

## Documenting DTOs with @Schema

`@Schema` annotates DTO fields to show descriptions and example values in the Swagger UI.

```java
import io.swagger.v3.oas.annotations.media.Schema;

@Schema(description = "Request body for creating a new product")
public class CreateProductRequest {

    @Schema(description = "Display name of the product", example = "Wireless Keyboard")
    @NotBlank
    private String name;

    @Schema(description = "Price in USD, must be greater than zero", example = "49.99")
    @NotNull
    @Positive
    private BigDecimal price;

    @Schema(description = "Initial stock quantity", example = "100", minimum = "0")
    @Min(0)
    private int stock;

    // getters / setters / constructors omitted
}
```

The `example` value populates the "try it out" form in Swagger UI, saving frontend devs from guessing what to type.

---

## Global API Info Bean

Register an `OpenAPI` bean to set the title, version, description, and contact info that appear at the top of Swagger UI.

```java
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.info.Info;

@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI apiInfo() {
        return new OpenAPI()
            .info(new Info()
                .title("Product Catalogue API")
                .version("1.0.0")
                .description("REST API for managing the product catalogue. Requires JWT authentication.")
                .contact(new Contact()
                    .name("Backend Team")
                    .email("backend@example.com")));
    }
}
```

---

## JWT Bearer Security Scheme

If your API is secured with JWT, add a security scheme so Swagger UI shows an **Authorize** button. Once a token is entered, every "Try it out" request automatically sends the `Authorization: Bearer <token>` header.

```java
import io.swagger.v3.oas.models.Components;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.security.SecurityRequirement;
import io.swagger.v3.oas.models.security.SecurityScheme;

@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI apiInfo() {
        final String schemeName = "bearerAuth";

        return new OpenAPI()
            .info(new Info()
                .title("Product Catalogue API")
                .version("1.0.0"))
            // Apply JWT auth globally to all endpoints
            .addSecurityItem(new SecurityRequirement().addList(schemeName))
            .components(new Components()
                .addSecuritySchemes(schemeName, new SecurityScheme()
                    .name(schemeName)
                    .type(SecurityScheme.Type.HTTP)
                    .scheme("bearer")
                    .bearerFormat("JWT")));
    }
}
```

After this, the Swagger UI displays a padlock icon on each endpoint. Click **Authorize**, paste a JWT, and subsequent "Try it out" requests include the token automatically.

---

## Customizing Paths & Disabling in Production

Control springdoc's behaviour via `application.properties` / `application.yml`.

```properties
# Change the Swagger UI path (default: /swagger-ui.html)
springdoc.swagger-ui.path=/docs

# Change the OpenAPI JSON path (default: /v3/api-docs)
springdoc.api-docs.path=/api-spec

# Disable Swagger UI entirely (for production)
springdoc.swagger-ui.enabled=false

# Disable the OpenAPI JSON endpoint
springdoc.api-docs.enabled=false
```

**Recommended production pattern** — use Spring profiles:

```properties
# application-prod.properties
springdoc.swagger-ui.enabled=false
springdoc.api-docs.enabled=false
```

```properties
# application-dev.properties
springdoc.swagger-ui.enabled=true
springdoc.api-docs.enabled=true
```

> **Security Warning**: Never expose Swagger UI in production without authentication. The interactive UI gives anyone with network access a complete map of your API and a tool to call every endpoint. At a minimum, put it behind an IP allowlist or require a login.

---

## Common Mistakes & Pitfalls

### 1. Using SpringFox with Spring Boot 3
```
NullPointerException on startup — SpringFox cannot handle Spring Boot 3's
path-matching strategy. Replace with springdoc-openapi-starter-webmvc-ui.
```

### 2. Wrong artifact name for Spring Boot 3
```xml
<!-- WRONG — Spring Boot 2 artifact, breaks on Boot 3 -->
<artifactId>springdoc-openapi-ui</artifactId>

<!-- CORRECT — Spring Boot 3 artifact -->
<artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
```

### 3. Exposing Swagger UI in production without protection
The interactive UI is a full map of your API. Disable it in production or gate it behind authentication.

### 4. Not documenting error responses
```java
// INCOMPLETE — only documents the happy path
@Operation(summary = "Get product")
@GetMapping("/{id}")
public ProductResponse getById(@PathVariable Long id) { ... }

// BETTER — documents what can go wrong too
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "Product found"),
    @ApiResponse(responseCode = "404", description = "Product not found"),
    @ApiResponse(responseCode = "401", description = "Unauthorized")
})
```

### 5. Forgetting to add `@Schema` examples on DTOs
Without example values, the Swagger UI "Try it out" form is empty. Frontend devs have to guess valid inputs. Add `example = "..."` to key fields.

---

## Common Interview Questions

### Q: What is the difference between OpenAPI and Swagger?

OpenAPI is a language-agnostic **specification** (a JSON/YAML standard) that describes an API's structure — endpoints, request/response shapes, auth schemes. Swagger UI is an **interactive web interface** that reads an OpenAPI spec and renders it as clickable documentation with a built-in HTTP client. springdoc-openapi generates the spec automatically and serves the Swagger UI at `/swagger-ui.html`.

---

### Q: Why should you use springdoc-openapi instead of SpringFox?

SpringFox has not been maintained since 2020 and fails on startup with Spring Boot 3 due to incompatibilities with Spring Framework 6's path-matching changes. springdoc-openapi is the actively maintained replacement, supports Spring Boot 3, and requires only a single dependency with no additional configuration to get a working Swagger UI.

---

### Q: How do you add JWT authentication support to Swagger UI?

Register an `OpenAPI` bean in a `@Configuration` class. Add a `SecurityScheme` of type `HTTP` with scheme `bearer` to the `Components`, then add a `SecurityRequirement` referencing that scheme to the `OpenAPI` object. This adds an **Authorize** button to Swagger UI where you can paste a token; subsequent "Try it out" requests automatically include the `Authorization: Bearer` header.

---

### Q: How do you prevent Swagger UI from being accessible in production?

Set `springdoc.swagger-ui.enabled=false` and `springdoc.api-docs.enabled=false` in `application-prod.properties` (or the equivalent prod profile). This disables both the interactive UI and the raw JSON spec endpoint so they are not reachable in the live environment.

---

### Q: What annotations do you use to document a controller endpoint?

`@Tag` on the class groups related endpoints in Swagger UI. `@Operation(summary=...)` on the method describes what it does. `@ApiResponses` with one or more `@ApiResponse` entries documents possible HTTP response codes. `@Parameter` describes individual path or query parameters. All come from `io.swagger.v3.oas.annotations`.

---

## Quick Reference Cheat Sheet

```
TERMS:
  OpenAPI Spec   → JSON/YAML blueprint describing your API (the standard)
  Swagger UI     → interactive web page that reads the blueprint
  springdoc      → library that auto-generates the spec from @RestController classes

DEPENDENCY (Spring Boot 3):
  groupId:    org.springdoc
  artifactId: springdoc-openapi-starter-webmvc-ui
  version:    2.x  (NOT springdoc-openapi-ui — that's Spring Boot 2)

  !! SpringFox is DEAD — do not use with Spring Boot 3 !!

AUTO-GENERATED URLS (zero config):
  /swagger-ui.html  → interactive Swagger UI
  /v3/api-docs      → raw OpenAPI JSON
  /v3/api-docs.yaml → raw OpenAPI YAML

KEY ANNOTATIONS (io.swagger.v3.oas.annotations):
  @Tag(name, description)          → on class:  groups endpoints in UI
  @Operation(summary, description) → on method: describes the endpoint
  @ApiResponse(responseCode, desc) → on method: documents one HTTP status
  @ApiResponses({...})             → on method: groups multiple @ApiResponse
  @Parameter(description, required)→ on param:  describes path/query param
  @Schema(description, example)    → on field:  documents DTO field in UI

GLOBAL API INFO BEAN:
  @Bean OpenAPI apiInfo() {
      return new OpenAPI().info(new Info().title(...).version(...));
  }

JWT SECURITY SCHEME:
  .addSecurityItem(new SecurityRequirement().addList("bearerAuth"))
  .components(new Components().addSecuritySchemes("bearerAuth",
      new SecurityScheme().type(HTTP).scheme("bearer").bearerFormat("JWT")))
  → adds Authorize button to Swagger UI

PROPERTIES:
  springdoc.swagger-ui.path=/docs         → change UI path
  springdoc.api-docs.path=/api-spec       → change JSON spec path
  springdoc.swagger-ui.enabled=false      → disable UI (use in prod)
  springdoc.api-docs.enabled=false        → disable spec (use in prod)

TOP PITFALLS:
  - using SpringFox with Spring Boot 3 → fails on startup
  - wrong artifact (springdoc-openapi-ui vs -starter-webmvc-ui)
  - Swagger UI exposed in production without protection
  - only documenting 200 responses, omitting 400/401/404
  - no @Schema examples → Swagger UI "Try it out" form is empty
```

---

*Last Updated: 2026-06-18*
