# REST API & HTTP Concepts Interview Questions

> Every full-stack Java developer must know REST API design inside out!

---

## Table of Contents
1. [HTTP Basics](#http-basics)
2. [REST API Principles](#rest-api-principles)
3. [HTTP Status Codes](#http-status-codes)
4. [REST API Best Practices](#rest-api-best-practices)
5. [Request & Response Design](#request--response-design)
6. [API Versioning](#api-versioning)
7. [Pagination & Filtering](#pagination--filtering)
8. [Quick Revision Summary](#quick-revision-summary)

---

## HTTP Basics

### Q1: What are the main HTTP methods and when to use them?

| Method | Purpose | Request Body | Idempotent? |
|--------|---------|-------------|-------------|
| **GET** | Read/fetch data | No | Yes |
| **POST** | Create new resource | Yes | No |
| **PUT** | Replace entire resource | Yes | Yes |
| **PATCH** | Update part of resource | Yes | Yes |
| **DELETE** | Remove a resource | Optional | Yes |

**Idempotent** = calling the same operation multiple times has the same effect as calling it once.

```bash
GET /api/users/1
POST /api/users          # Body: {"name": "Amith", "email": "amith@gmail.com"}
PUT /api/users/1         # Body: full replacement
PATCH /api/users/1       # Body: {"email": "newemail@gmail.com"}
DELETE /api/users/1
```

---

### Q2: What are HTTP Headers?

Headers are metadata sent with every request/response.

**Common Request Headers:**
```
Content-Type: application/json        // Format of request body
Authorization: Bearer eyJhbGci...     // JWT token
Accept: application/json              // Format client wants back
```

**Common Response Headers:**
```
Content-Type: application/json        // Format of response body
Location: /api/users/123              // URL of created resource (POST 201)
Access-Control-Allow-Origin: *        // CORS header
Cache-Control: max-age=3600           // Cache settings
```

---

## REST API Principles

### Q3: What is REST and what are its principles?

REST (Representational State Transfer) is a set of rules for building APIs.

| Principle | What it means |
|-----------|--------------|
| **Stateless** | Every request is self-contained; server stores no session between requests |
| **Client-Server** | Frontend and backend are separate; communicate only via API |
| **Uniform Interface** | Use nouns in URLs, consistent HTTP methods |
| **Resource-Based** | Everything is a resource identified by URL (`/users`, `/users/1`) |
| **HTTP Methods** | GET=Read, POST=Create, PUT/PATCH=Update, DELETE=Delete |
| **Layered System** | Client doesn't need to know about load balancers, caches, etc. |

```
❌ /getUser, /createUser    ✅ GET /users, POST /users
```

---

### Q4: What is the difference between REST and SOAP?

| Feature | REST | SOAP |
|---------|------|------|
| **Format** | JSON, XML, HTML | XML only |
| **Speed** | Fast (lightweight) | Slower (verbose XML) |
| **Simplicity** | Simple | Complex (WSDL, schemas) |
| **State** | Stateless | Can be stateful |
| **Use case** | Web/mobile APIs | Enterprise, banking, legacy |
| **Error handling** | HTTP status codes | SOAP Fault |

---

## HTTP Status Codes

### Q5: What are the important HTTP status codes?

**2xx — Success**
```
200 OK              → GET/PUT/PATCH succeeded, data returned
201 Created         → POST succeeded, resource created
204 No Content      → DELETE succeeded, no body returned
```

**3xx — Redirection**
```
301 Moved Permanently  → Resource has a new permanent URL
304 Not Modified       → Use cached version
```

**4xx — Client Errors (your fault)**
```
400 Bad Request        → Invalid request data
401 Unauthorized       → Not logged in
403 Forbidden          → Logged in but no permission
404 Not Found          → Resource doesn't exist
409 Conflict           → Duplicate resource (e.g. duplicate email)
422 Unprocessable      → Validation failed
429 Too Many Requests  → Rate limit exceeded
```

**5xx — Server Errors (server's fault)**
```
500 Internal Server Error → Unhandled exception in code
503 Service Unavailable   → Server overloaded or down
504 Gateway Timeout       → Upstream server too slow
```

**Spring Boot example:**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        if (user == null) return ResponseEntity.notFound().build(); // 404
        return ResponseEntity.ok(user);                             // 200
    }

    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody UserDTO dto) {
        User created = userService.create(dto);
        URI location = URI.create("/api/users/" + created.getId());
        return ResponseEntity.created(location).body(created);      // 201
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();                   // 204
    }
}
```

---

## REST API Best Practices

### Q6: What are the URL naming best practices for REST APIs?

```
Use NOUNS, not verbs:
❌ /getUsers, /createOrder    ✅ GET /users, POST /orders

Use lowercase and hyphens:
❌ /UserOrders                ✅ /user-orders

Use plural for collections:
❌ /user/1                    ✅ /users/1

Nested resources for relationships:
✅ /users/1/orders            — All orders of user 1
✅ /categories/3/products     — All products in category 3

Query params for filtering/sorting/pagination:
✅ /users?status=active
✅ /users?sort=name&order=asc
✅ /users?page=2&size=10
```

---

### Q7: What is HATEOAS?

HATEOAS means API responses include links to related actions — the client doesn't need to know URLs in advance.

```json
{
  "id": 1,
  "name": "Amith",
  "_links": {
    "self":   { "href": "/api/users/1" },
    "orders": { "href": "/api/users/1/orders" },
    "delete": { "href": "/api/users/1", "method": "DELETE" }
  }
}
```

```java
@GetMapping("/{id}")
public EntityModel<User> getUser(@PathVariable Long id) {
    User user = userService.findById(id);
    return EntityModel.of(user,
        linkTo(methodOn(UserController.class).getUser(id)).withSelfRel(),
        linkTo(methodOn(UserController.class).getAllUsers()).withRel("users"));
}
```

---

## Request & Response Design

### Q8: What is the difference between DTO and Entity?

- **Entity** = maps to a database table (all fields, including sensitive ones)
- **DTO** = what you expose through the API (only needed fields, no passwords)

```java
// Entity
@Entity
public class User {
    private Long id;
    private String name;
    private String email;
    private String password;  // never expose this
}

// Request DTO
public class CreateUserRequest {
    @NotBlank @Size(min=2, max=50) private String name;
    @NotBlank @Email               private String email;
    @NotBlank @Size(min=8)         private String password;
}

// Response DTO — no password
public class UserResponse {
    private Long id;
    private String name;
    private String email;
    private LocalDateTime createdAt;
}
```

**Why use DTOs?** Hide sensitive fields, control the API contract, decouple API from DB schema, add input validation.

---

### Q9: How do you design a standard API error response?

```java
public class ApiError {
    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String path;
    private Map<String, String> fieldErrors; // for validation errors
}

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException ex, HttpServletRequest req) {
        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
            .forEach(e -> fieldErrors.put(e.getField(), e.getDefaultMessage()));
        return ResponseEntity.badRequest()
            .body(new ApiError(LocalDateTime.now(), 400, "Bad Request", "Validation failed", req.getRequestURI(), fieldErrors));
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(ResourceNotFoundException ex, HttpServletRequest req) {
        return ResponseEntity.status(404)
            .body(new ApiError(LocalDateTime.now(), 404, "Not Found", ex.getMessage(), req.getRequestURI(), null));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleGeneric(Exception ex, HttpServletRequest req) {
        return ResponseEntity.status(500)
            .body(new ApiError(LocalDateTime.now(), 500, "Internal Server Error", "An unexpected error occurred", req.getRequestURI(), null));
    }
}
```

**Sample error responses:**
```json
// 400 Validation Error
{
  "status": 400, "error": "Bad Request", "message": "Validation failed",
  "fieldErrors": { "email": "Invalid email format", "name": "Name must be 2-50 characters" }
}

// 404 Not Found
{ "status": 404, "error": "Not Found", "message": "User with id 99 not found", "path": "/api/users/99" }
```

---

## API Versioning

### Q10: How do you version a REST API?

When you change your API, old clients break. Versioning lets old clients use v1 while new clients use v2.

| Method | Example | Notes |
|--------|---------|-------|
| **URL Path** (most common) | `/api/v1/users` | Simple, visible, widely used |
| **Request Parameter** | `/api/users?version=1` | Easy but pollutes query params |
| **Header** | `API-Version: 1` | Clean URLs, harder to test in browser |
| **Accept Header** | `Accept: application/vnd.myapp.v1+json` | Purist REST, complex |

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    @GetMapping
    public List<UserResponseV1> getUsers() { ... }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    @GetMapping
    public List<UserResponseV2> getUsers() { ... }
}
```

---

## Pagination & Filtering

### Q11: How do you implement pagination in Spring Boot?

Instead of returning thousands of records at once, return a page at a time.

```java
@GetMapping
public ResponseEntity<Page<UserResponse>> getUsers(
        @RequestParam(defaultValue = "0")   int page,
        @RequestParam(defaultValue = "10")  int size,
        @RequestParam(defaultValue = "id")  String sortBy,
        @RequestParam(defaultValue = "asc") String direction) {

    Sort sort = direction.equalsIgnoreCase("desc")
        ? Sort.by(sortBy).descending() : Sort.by(sortBy).ascending();

    Pageable pageable = PageRequest.of(page, size, sort);
    Page<UserResponse> response = userRepository.findAll(pageable).map(UserResponse::new);
    return ResponseEntity.ok(response);
}
```

**Usage:** `GET /api/users?page=0&size=10&sortBy=name&direction=asc`

**Response:**
```json
{
  "content": [...],
  "totalElements": 150,
  "totalPages": 15,
  "size": 10,
  "number": 0,
  "first": true,
  "last": false
}
```

---

## Quick Revision Summary

### HTTP Methods Cheat Sheet

| Method | Use | Success Code |
|--------|-----|-------------|
| GET | Fetch data | 200 |
| POST | Create resource | 201 |
| PUT | Replace resource | 200 |
| PATCH | Update part | 200 |
| DELETE | Remove resource | 204 |

### Status Codes to Remember

| Code | Meaning | When |
|------|---------|------|
| 200 | OK | Successful GET/PUT/PATCH |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Invalid data sent |
| 401 | Unauthorized | Not logged in |
| 403 | Forbidden | Logged in, no permission |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource |
| 500 | Server Error | Bug in server code |

### REST URL Design Rules
```
GET    /api/users           → list all
POST   /api/users           → create new
GET    /api/users/{id}      → get one
PUT    /api/users/{id}      → replace one
PATCH  /api/users/{id}      → update one
DELETE /api/users/{id}      → remove one
GET    /api/users/{id}/orders → get user's orders
```

### Interview Tips
1. Always mention **status codes** when explaining API design
2. Always use **DTOs** — never expose entities directly
3. **401** = not logged in, **403** = logged in but no permission
4. Pagination: use `Page<T>` with `Pageable` in Spring Boot
5. Versioning: URL path versioning is simplest and most common
