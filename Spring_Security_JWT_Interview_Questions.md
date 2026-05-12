# Spring Security & JWT Interview Questions

> 🎯 Security is always asked in Full Stack Java interviews. Master these concepts!

---

## Table of Contents
1. [Spring Security Basics](#spring-security-basics)
2. [Authentication vs Authorization](#authentication-vs-authorization)
3. [JWT (JSON Web Token)](#jwt-json-web-token)
4. [Spring Security with JWT Implementation](#spring-security-with-jwt-implementation)
5. [Common Security Annotations](#common-security-annotations)
6. [CORS & CSRF](#cors--csrf)
7. [Password Encoding](#password-encoding)
8. [Quick Revision Summary](#quick-revision-summary)

---

## Spring Security Basics

### Q1: What is Spring Security and why is it used?

**Easy Explanation:** Spring Security is a framework that protects your Spring Boot application from unauthorized access. It handles:
- Who can log in (Authentication)
- What they can do after login (Authorization)
- Protection against common attacks (CSRF, XSS, etc.)

**Without Spring Security:** Anyone can access any URL in your app.
**With Spring Security:** You control who accesses what.

**Add to project:**
```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

> 💡 **What happens after adding this dependency?**
> All your endpoints are immediately **protected**! You need to login with default user `user` and auto-generated password (shown in console).

---

### Q2: What is the Spring Security filter chain?

**Easy Explanation:** Every request that comes to your app passes through a series of filters (like security checkpoints). Each filter checks something specific.

```
HTTP Request
     ↓
┌─────────────────────────────────┐
│     Security Filter Chain       │
│                                 │
│  1. UsernamePasswordAuthFilter  │  ← Handles login form
│  2. BasicAuthenticationFilter   │  ← Handles HTTP Basic Auth
│  3. JwtAuthenticationFilter     │  ← Custom JWT filter
│  4. ExceptionTranslationFilter  │  ← Handles security errors
│  5. FilterSecurityInterceptor   │  ← Checks permissions
└─────────────────────────────────┘
     ↓
Your Controller (if allowed)
```

**Key Point:** Requests are rejected BEFORE reaching your controller if security checks fail.

---

## Authentication vs Authorization

### Q3: What is the difference between Authentication and Authorization?

**Easy Explanation:**
- **Authentication** = "Who are you?" — Verify identity (Login)
- **Authorization** = "What can you do?" — Check permissions (Roles)

```
Authentication Example:
Username: amith
Password: password123
→ "Yes, you are Amith" ✅ (You are now logged in)

Authorization Example:
Amith tries to access /admin/dashboard
→ "Amith has role USER, not ADMIN" ❌ (Access denied)
```

| | Authentication | Authorization |
|--|----------------|---------------|
| **Question** | Who are you? | What can you access? |
| **When** | Before authorization | After authentication |
| **Example** | Login with credentials | Role-based access control |
| **Failure** | 401 Unauthorized | 403 Forbidden |
| **Spring class** | `AuthenticationManager` | `AccessDecisionManager` |

---

## JWT (JSON Web Token)

### Q4: What is JWT and how does it work?

**Easy Explanation:** JWT is like a signed ID card. Once you login, the server gives you a token (ID card). You show this token with every request instead of logging in each time.

```
Without JWT (Session-based):
1. User logs in → Server stores session in memory
2. User sends request → Server checks session ID in database
3. Problem: Server needs to store session, doesn't scale

With JWT (Stateless):
1. User logs in → Server creates and signs a JWT token
2. User sends request with JWT in header
3. Server verifies JWT signature (NO database lookup!)
4. Benefit: Stateless, scales easily
```

**JWT Structure:** A JWT has 3 parts separated by dots `.`

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhbWl0aCIsInJvbGUiOiJVU0VSIn0.abc123
      ↑                              ↑                          ↑
   HEADER                         PAYLOAD                  SIGNATURE
(algorithm)              (user data / claims)           (secret key sign)
```

**Decoded:**
```json
// Header
{
  "alg": "HS256",
  "typ": "JWT"
}

// Payload (Claims)
{
  "sub": "amith",           // Subject (username)
  "role": "USER",
  "iat": 1715000000,        // Issued at
  "exp": 1715086400         // Expires at
}

// Signature = HMAC_SHA256(base64(header) + "." + base64(payload), SECRET_KEY)
```

> ⚠️ **Important:** JWT payload is NOT encrypted! It's only base64 encoded. Anyone can decode it. Never store passwords in JWT.

---

### Q5: What is the JWT authentication flow?

```
┌──────────┐                              ┌──────────┐
│  Client  │                              │  Server  │
└──────────┘                              └──────────┘
      │                                        │
      │  POST /login {username, password}       │
      │ ──────────────────────────────────────>│
      │                                        │ Validate credentials
      │                                        │ Generate JWT token
      │  { token: "eyJhbGciO..." }             │
      │ <──────────────────────────────────────│
      │                                        │
      │  GET /api/users                        │
      │  Authorization: Bearer eyJhbGciO...    │
      │ ──────────────────────────────────────>│
      │                                        │ Verify JWT signature
      │                                        │ Extract username
      │                                        │ Load user details
      │  { users: [...] }                      │
      │ <──────────────────────────────────────│
```

---

## Spring Security with JWT Implementation

### Q6: How do you implement JWT in Spring Boot?

**Step 1: Add Dependencies**
```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.3</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
```

**Step 2: JWT Utility Class**
```java
// JwtUtils.java
@Component
public class JwtUtils {

    @Value("${jwt.secret}")
    private String jwtSecret;         // Secret key from application.properties

    @Value("${jwt.expiration}")
    private int jwtExpirationMs;      // Expiry time (e.g., 86400000 = 24 hours)

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(jwtSecret.getBytes(StandardCharsets.UTF_8));
    }

    // Generate token from username
    public String generateToken(String username) {
        return Jwts.builder()
                .subject(username)
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + jwtExpirationMs))
                .signWith(getSigningKey())
                .compact();
    }

    // Extract username from token
    public String getUsernameFromToken(String token) {
        return Jwts.parser()
                .verifyWith(getSigningKey())
                .build()
                .parseSignedClaims(token)
                .getPayload()
                .getSubject();
    }

    // Validate token
    public boolean validateToken(String token) {
        try {
            Jwts.parser()
                .verifyWith(getSigningKey())
                .build()
                .parseSignedClaims(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false; // Invalid or expired token
        }
    }
}
```

**Step 3: JWT Filter (Intercepts every request)**
```java
// JwtAuthFilter.java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    @Autowired
    private JwtUtils jwtUtils;

    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        // 1. Get Authorization header
        String authHeader = request.getHeader("Authorization");

        // 2. Check if it has Bearer token
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7); // Remove "Bearer "

            // 3. Validate token
            if (jwtUtils.validateToken(token)) {
                String username = jwtUtils.getUsernameFromToken(token);

                // 4. Load user details
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                // 5. Set authentication in security context
                UsernamePasswordAuthenticationToken auth =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());

                SecurityContextHolder.getContext().setAuthentication(auth);
            }
        }

        // 6. Continue filter chain (go to next filter / controller)
        filterChain.doFilter(request, response);
    }
}
```

**Step 4: Security Configuration**
```java
// SecurityConfig.java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired
    private JwtAuthFilter jwtAuthFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // Disable CSRF (not needed for REST APIs with JWT)
            .csrf(csrf -> csrf.disable())

            // Configure URL permissions
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()   // Public endpoints
                .requestMatchers("/api/admin/**").hasRole("ADMIN")  // Admin only
                .anyRequest().authenticated()                  // All others need login
            )

            // Stateless session (JWT is stateless)
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            // Add JWT filter before UsernamePasswordAuthenticationFilter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // Hash passwords!
    }
}
```

**Step 5: Auth Controller (Login endpoint)**
```java
// AuthController.java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private JwtUtils jwtUtils;

    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest loginRequest) {
        // 1. Authenticate user
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                loginRequest.getUsername(),
                loginRequest.getPassword()
            )
        );

        // 2. Get username
        UserDetails userDetails = (UserDetails) authentication.getPrincipal();

        // 3. Generate JWT token
        String token = jwtUtils.generateToken(userDetails.getUsername());

        // 4. Return token
        return ResponseEntity.ok(new LoginResponse(token));
    }
}

// DTOs
record LoginRequest(String username, String password) {}
record LoginResponse(String token) {}
```

**application.properties:**
```properties
jwt.secret=mySecretKey1234567890123456789012345678  # min 32 chars for HS256
jwt.expiration=86400000  # 24 hours in milliseconds
```

---

## Common Security Annotations

### Q7: What are the main Spring Security method-level annotations?

```java
@RestController
@RequestMapping("/api")
public class UserController {

    // Anyone can access (no login required)
    @GetMapping("/public")
    public String publicEndpoint() {
        return "Anyone can see this";
    }

    // Must be logged in (any role)
    @GetMapping("/profile")
    @PreAuthorize("isAuthenticated()")
    public String userProfile() {
        return "Your profile";
    }

    // Must have ADMIN role
    @GetMapping("/admin")
    @PreAuthorize("hasRole('ADMIN')")
    public String adminPanel() {
        return "Admin panel";
    }

    // Must have either ADMIN or MANAGER role
    @GetMapping("/manage")
    @PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER')")
    public String manageUsers() {
        return "Manage users";
    }

    // User can only access their own data (unless admin)
    @GetMapping("/users/{id}")
    @PreAuthorize("hasRole('ADMIN') or #id == authentication.name")
    public String getUserById(@PathVariable String id) {
        return "User " + id + " data";
    }
}

// Enable method security in config:
@Configuration
@EnableMethodSecurity  // Required for @PreAuthorize to work!
public class SecurityConfig { ... }
```

---

## CORS & CSRF

### Q8: What is CORS and how do you configure it?

**Easy Explanation:** CORS (Cross-Origin Resource Sharing) controls which websites can call your API.

**Problem:** Your frontend is at `http://localhost:3000` but backend at `http://localhost:8080`. Browser blocks this by default for security!

**Solution: Configure CORS**
```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();

        config.setAllowedOrigins(Arrays.asList(
            "http://localhost:3000",          // React dev server
            "https://myproduction-site.com"   // Production
        ));
        config.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(Arrays.asList("*"));  // Allow all headers
        config.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);  // Apply to all paths
        return source;
    }
}

// In SecurityConfig:
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
```

**Quick CORS for controller:**
```java
@CrossOrigin(origins = "http://localhost:3000")
@RestController
public class UserController { ... }
```

---

### Q9: What is CSRF and why is it disabled for REST APIs?

**Easy Explanation:**
- **CSRF** = Cross-Site Request Forgery — an attack where a malicious site tricks your browser into making requests to your API
- **Why disabled for REST?** REST APIs use JWT/token auth in headers, not cookies. CSRF only attacks cookie-based sessions.

```java
// For web apps (form-based): KEEP CSRF enabled
http.csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()));

// For REST APIs with JWT: DISABLE CSRF
http.csrf(csrf -> csrf.disable());
```

---

## Password Encoding

### Q10: How do you store passwords securely?

**NEVER store plain text passwords!**

```java
// ❌ WRONG: Plain text password
user.setPassword("mypassword123");

// ✅ CORRECT: BCrypt hashed password
@Autowired
private PasswordEncoder passwordEncoder;

user.setPassword(passwordEncoder.encode("mypassword123"));
// Stored as: $2a$10$xyz...  (60 character hash)

// Verifying password during login:
boolean matches = passwordEncoder.matches("mypassword123", storedHash);
```

**BCryptPasswordEncoder:**
- Each hash is unique (salt is included)
- Same password → different hashes each time
- Cannot be reversed (one-way hashing)
- Slow by design (makes brute force attacks hard)

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);  // strength 10-12 is recommended
}
```

---

## Quick Revision Summary

### 🔑 Key Concepts

| Concept | Simple Explanation |
|---------|-------------------|
| **Authentication** | Login — verify who you are |
| **Authorization** | Roles — what you can access |
| **JWT** | Signed token passed with each request |
| **BCrypt** | One-way password hashing |
| **CORS** | Control which domains can call your API |
| **CSRF** | Disable for REST APIs with JWT |
| **SecurityFilterChain** | Configure URL permissions |
| **@PreAuthorize** | Method-level security |

### 🔑 JWT Interview Q&A

**Q: Where do you store JWT on the client side?**
> localStorage or sessionStorage (for SPAs), or HttpOnly cookie (more secure)

**Q: What happens when JWT expires?**
> Client gets 401 Unauthorized, must login again or use refresh token

**Q: What is a Refresh Token?**
> A long-lived token used to get a new JWT without re-logging in

**Q: Can you invalidate a JWT?**
> JWT is stateless — you can't invalidate it directly. Solutions:
> - Short expiration time
> - Blacklist in database/Redis
> - Refresh token rotation

**Q: Is JWT payload secure?**
> ❌ No! It's base64 encoded, not encrypted. Anyone can decode it. Never store sensitive data!

**Q: What HTTP header carries JWT?**
> `Authorization: Bearer <token>`

### 🏗️ Full JWT Flow in 5 Steps
```
1. POST /login → Server validates credentials
2. Server creates JWT with username + expiry, signs with secret key
3. Client saves JWT
4. Client sends: GET /api/data + Authorization: Bearer <token>
5. Server: validate signature → extract user → check permissions → respond
```
