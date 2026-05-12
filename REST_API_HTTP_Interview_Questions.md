# REST API & HTTP Concepts Interview Questions

> 🎯 Every full-stack Java developer must know REST API design inside out!

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

**Easy Explanation:** HTTP methods tell the server what action you want to perform on a resource.

| Method | Purpose | Request Body | Response Body | Idempotent? |
|--------|---------|-------------|---------------|-------------|
| **GET** | Read/fetch data | ❌ No | ✅ Yes | ✅ Yes |
| **POST** | Create new resource | ✅ Yes | ✅ Yes | ❌ No |
| **PUT** | Replace entire resource | ✅ Yes | ✅ Yes | ✅ Yes |
| **PATCH** | Update part of resource | ✅ Yes | ✅ Yes | ✅ Yes |
| **DELETE** | Remove a resource | Optional | Optional | ✅ Yes |
| **OPTIONS** | Get allowed methods | ❌ No | ✅ Yes | ✅ Yes |

**Idempotent** = calling the same operation multiple times has the same effect as calling it once.

```bash
# GET — Read users
GET /api/users
GET /api/users/1

# POST — Create new user
POST /api/users
Body: {"name": "Amith", "email": "amith@gmail.com"}

# PUT — Replace user completely
PUT /api/users/1
Body: {"name": "Amith Updated", "email": "new@gmail.com", "age": 25}

# PATCH — Update only specific fields
PATCH /api/users/1
Body: {"email": "newemail@gmail.com"}

# DELETE — Delete user
DELETE /api/users/1
```

---

### Q2: What are HTTP Headers?

**Easy Explanation:** Headers are extra information sent with every request/response — like metadata.

**Common Request Headers:**
```
Content-Type: application/json        // What format the body is in
Authorization: Bearer eyJhbGci...     // JWT token for authentication
Accept: application/json              // What format client wants back
Accept-Language: en-US                // Preferred language
User-Agent: Mozilla/5.0...            // Browser/client info
```

**Common Response Headers:**
```
Content-Type: application/json        // Format of response body
Status: 200 OK                        // HTTP status
Location: /api/users/123              // URL of created resource (POST)
Access-Control-Allow-Origin: *        // CORS header
Cache-Control: max-age=3600           // Cache settings
```

---

## REST API Principles

### Q3: What is REST and what are its principles?

**Easy Explanation:** REST (Representational State Transfer) is a set of rules for building APIs. A REST API is just a web API that follows these rules.

**6 REST Principles:**

**1. Stateless**
```
Each request must contain ALL information needed.
Server stores NO session/client state between requests.

❌ Wrong: Server remembers previous request
✅ Correct: Every request is independent (JWT in every request)
```

**2. Client-Server Separation**
```
Frontend (React/Angular) and Backend (Spring Boot) are separate.
They communicate only through API.
Benefit: Can change frontend without changing backend.
```

**3. Uniform Interface**
```
Consistent URLs and HTTP methods.
Use nouns (not verbs) in URLs.

❌ Wrong: /getUser, /createUser, /deleteUser
✅ Correct: /users (GET), /users (POST), /users/{id} (DELETE)
```

**4. Resource-Based**
```
Everything is a resource identified by URL.
Resources are nouns (users, orders, products).

✅ /api/users            — User collection resource
✅ /api/users/1          — Single user resource
✅ /api/users/1/orders   — Orders of user 1
```

**5. Use of HTTP Methods**
```
Use correct HTTP methods for correct actions:
GET = Read, POST = Create, PUT/PATCH = Update, DELETE = Delete
```

**6. Layered System**
```
Client doesn't need to know if it's talking to server, 
load balancer, or cache. Each layer is independent.
```

---

### Q4: What is the difference between REST and SOAP?

| Feature | REST | SOAP |
|---------|------|------|
| **Protocol** | HTTP | HTTP, SMTP, TCP |
| **Format** | JSON, XML, HTML | XML only |
| **Speed** | Fast (lightweight) | Slower (verbose XML) |
| **Simplicity** | Simple | Complex (WSDL, schemas) |
| **State** | Stateless | Can be stateful |
| **Use case** | Web/mobile APIs | Enterprise, banking, legacy |
| **Error handling** | HTTP status codes | SOAP Fault |

**Simple analogy:**
- **REST** = Email (simple, flexible, widely used)
- **SOAP** = Legal contract (formal, strict, verbose)

---

## HTTP Status Codes

### Q5: What are the important HTTP status codes?

**Easy Explanation:** Status codes tell you what happened with your request. Think of it like a pizza order update.

**2xx — Success ✅**
```
200 OK              → Request succeeded, data returned (GET/PUT/PATCH)
201 Created         → Resource created successfully (POST)
204 No Content      → Success but no data to return (DELETE)
```

**3xx — Redirection 🔄**
```
301 Moved Permanently  → Resource has a new permanent URL
302 Found (Redirect)   → Temporarily at a different URL
304 Not Modified       → Use cached version
```

**4xx — Client Errors ❌ (Your fault)**
```
400 Bad Request        → Invalid request data sent
401 Unauthorized       → Not logged in (need to authenticate)
403 Forbidden          → Logged in but no permission (wrong role)
404 Not Found          → Resource doesn't exist
405 Method Not Allowed → Wrong HTTP method (POST instead of GET)
409 Conflict           → Resource already exists (duplicate email)
422 Unprocessable      → Validation failed
429 Too Many Requests  → Rate limit exceeded
```

**5xx — Server Errors 💥 (Server's fault)**
```
500 Internal Server Error → Unhandled exception in code
502 Bad Gateway           → Upstream server error
503 Service Unavailable   → Server overloaded or down
504 Gateway Timeout       → Upstream server too slow
```

**Practical Example in Spring Boot:**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        if (user == null) {
            return ResponseEntity.notFound().build();           // 404
        }
        return ResponseEntity.ok(user);                         // 200
    }

    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody UserDTO dto) {
        User created = userService.create(dto);
        URI location = URI.create("/api/users/" + created.getId());
        return ResponseEntity.created(location).body(created);  // 201
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();               // 204
    }
}
```

---

## REST API Best Practices

### Q6: What are the URL naming best practices for REST APIs?

```
✅ CORRECT REST URL Design:

Use NOUNS (not verbs):
❌ /getUsers           ✅ /users
❌ /createOrder        ✅ /orders  (with POST)
❌ /deleteProduct/1    ✅ /products/1  (with DELETE)

Use lowercase and hyphens:
❌ /UserOrders         ✅ /user-orders
❌ /OrderItems         ✅ /order-items

Use plural for collections:
❌ /user/1             ✅ /users/1
❌ /product            ✅ /products

Nested resources for relationships:
✅ /users/1/orders          — All orders of user 1
✅ /users/1/orders/5        — Order 5 of user 1
✅ /categories/3/products   — All products in category 3

Query params for filtering, not paths:
❌ /users/active            ✅ /users?status=active
❌ /users/sorted            ✅ /users?sort=name&order=asc
✅ /users?page=2&size=10    — Pagination
✅ /products?minPrice=100&maxPrice=500  — Filtering
```

---

### Q7: What is HATEOAS?

**Easy Explanation:** HATEOAS (Hypermedia as the Engine of Application State) means your API responses include links to related actions. Client doesn't need to know URLs in advance.

```json
// Without HATEOAS (client must know all URLs):
{
  "id": 1,
  "name": "Amith",
  "email": "amith@gmail.com"
}

// With HATEOAS (response tells client what it can do next):
{
  "id": 1,
  "name": "Amith",
  "email": "amith@gmail.com",
  "_links": {
    "self": { "href": "/api/users/1" },
    "orders": { "href": "/api/users/1/orders" },
    "update": { "href": "/api/users/1", "method": "PUT" },
    "delete": { "href": "/api/users/1", "method": "DELETE" }
  }
}
```

```java
// Spring HATEOAS implementation
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

**Easy Explanation:**
- **Entity** = Maps directly to database table (contains ALL fields)
- **DTO (Data Transfer Object)** = What you expose to the API (only needed fields)

```java
// Entity (Database table mapping)
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;
    private String password;      // ❌ Should NEVER be in API response!
    private String ssn;           // ❌ Sensitive data
    private LocalDateTime createdAt;
}

// Request DTO (What client sends for creating user)
public class CreateUserRequest {
    @NotBlank @Size(min=2, max=50)
    private String name;

    @NotBlank @Email
    private String email;

    @NotBlank @Size(min=8)
    private String password;
}

// Response DTO (What we send back — no password!)
public class UserResponse {
    private Long id;
    private String name;
    private String email;
    private LocalDateTime createdAt;
}

// Controller uses DTOs, not entities
@PostMapping
public ResponseEntity<UserResponse> createUser(@Valid @RequestBody CreateUserRequest req) {
    UserResponse response = userService.createUser(req);
    return ResponseEntity.status(201).body(response);
}
```

**Why use DTOs?**
1. Hide sensitive fields (password, SSN)
2. Control what the client can send/receive
3. Decouple API contract from database schema
4. Add validation only on input

---

### Q9: How do you design a standard API error response?

```java
// Standard error response structure
public class ApiError {
    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String path;
    private Map<String, String> fieldErrors; // for validation errors

    // Constructor, getters...
}

// Global Exception Handler
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Handle validation errors (400)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidationException(
            MethodArgumentNotValidException ex, HttpServletRequest request) {

        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
            .forEach(e -> fieldErrors.put(e.getField(), e.getDefaultMessage()));

        ApiError error = new ApiError(
            LocalDateTime.now(), 400, "Bad Request",
            "Validation failed", request.getRequestURI(), fieldErrors);

        return ResponseEntity.badRequest().body(error);
    }

    // Handle not found (404)
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFoundException(
            ResourceNotFoundException ex, HttpServletRequest request) {

        ApiError error = new ApiError(
            LocalDateTime.now(), 404, "Not Found",
            ex.getMessage(), request.getRequestURI(), null);

        return ResponseEntity.status(404).body(error);
    }

    // Handle all other errors (500)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleGenericException(
            Exception ex, HttpServletRequest request) {

        ApiError error = new ApiError(
            LocalDateTime.now(), 500, "Internal Server Error",
            "An unexpected error occurred", request.getRequestURI(), null);

        return ResponseEntity.status(500).body(error);
    }
}
```

**Sample error responses:**
```json
// 400 Validation Error
{
  "timestamp": "2024-01-15T10:30:00",
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "path": "/api/users",
  "fieldErrors": {
    "email": "Invalid email format",
    "name": "Name must be between 2 and 50 characters"
  }
}

// 404 Not Found
{
  "timestamp": "2024-01-15T10:30:00",
  "status": 404,
  "error": "Not Found",
  "message": "User with id 99 not found",
  "path": "/api/users/99"
}
```

---

## API Versioning

### Q10: How do you version a REST API?

**Why versioning?** When you change your API, old clients break. Versioning lets old clients continue using v1 while new clients use v2.

**Method 1: URL Path (Most common)**
```
/api/v1/users
/api/v2/users
```

**Method 2: Request Parameter**
```
/api/users?version=1
/api/users?version=2
```

**Method 3: Header**
```
GET /api/users
API-Version: 1
```

**Method 4: Accept Header (Content Negotiation)**
```
Accept: application/vnd.myapp.v1+json
Accept: application/vnd.myapp.v2+json
```

```java
// URL Path versioning (most recommended for simplicity)
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
    public List<UserResponseV2> getUsers() { ... } // Different response format
}
```

---

## Pagination & Filtering

### Q11: How do you implement pagination in Spring Boot?

**Easy Explanation:** Instead of returning 10,000 users at once, return 10 per page.

```java
// Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Page<User> findByStatus(String status, Pageable pageable);
}

// Controller
@GetMapping
public ResponseEntity<Page<UserResponse>> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size,
        @RequestParam(defaultValue = "id") String sortBy,
        @RequestParam(defaultValue = "asc") String direction) {

    Sort sort = direction.equalsIgnoreCase("desc") ?
        Sort.by(sortBy).descending() : Sort.by(sortBy).ascending();

    Pageable pageable = PageRequest.of(page, size, sort);
    Page<User> users = userRepository.findAll(pageable);

    // Map to response DTOs
    Page<UserResponse> response = users.map(u -> new UserResponse(u));
    return ResponseEntity.ok(response);
}
```

**API Usage:**
```
GET /api/users?page=0&size=10&sortBy=name&direction=asc
GET /api/users?page=1&size=20&sortBy=createdAt&direction=desc
```

**Response:**
```json
{
  "content": [...],         // Array of users
  "totalElements": 150,     // Total users in database
  "totalPages": 15,         // 150 / 10 = 15 pages
  "size": 10,               // Page size
  "number": 0,              // Current page (0-based)
  "first": true,            // Is first page?
  "last": false             // Is last page?
}
```

---

## Quick Revision Summary

### 🔑 HTTP Methods Cheat Sheet

| Method | Use | Success Code |
|--------|-----|-------------|
| GET | Fetch data | 200 |
| POST | Create resource | 201 |
| PUT | Replace resource | 200 |
| PATCH | Update part | 200 |
| DELETE | Remove resource | 204 |

### 🔑 Status Codes to Remember

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

### 🔑 REST URL Design Rules
```
✅ /api/users             GET  → list all
✅ /api/users             POST → create new
✅ /api/users/{id}        GET  → get one
✅ /api/users/{id}        PUT  → replace one
✅ /api/users/{id}        PATCH → update one
✅ /api/users/{id}        DELETE → remove one
✅ /api/users/{id}/orders GET  → get user's orders
```

### 📝 Interview Tips
1. Always mention **status codes** when explaining API design
2. Always use **DTOs** to separate entity from API response
3. Remember: **401** = not logged in, **403** = logged in but no permission
4. Pagination: Use `Page<T>` with `Pageable` in Spring Boot
5. Versioning: URL path versioning is simplest and most common
