# Spring Security & JWT Interview Questions

> Security is always asked in Full Stack Java interviews. Master these concepts!

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

Spring Security is a framework that protects your Spring Boot application. It handles authentication (who can log in), authorization (what they can do), and protection against common attacks (CSRF, XSS, etc.).

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

> After adding this dependency, all endpoints are immediately protected. You must log in with default user `user` and the auto-generated password shown in the console.

---

### Q2: What is the Spring Security filter chain?

Every request passes through a series of filters before reaching your controller. If a check fails, the request is rejected early.

```
HTTP Request
     ↓
┌─────────────────────────────────┐
│     Security Filter Chain       │
│  1. UsernamePasswordAuthFilter  │
│  2. BasicAuthenticationFilter   │
│  3. JwtAuthenticationFilter     │
│  4. ExceptionTranslationFilter  │
│  5. FilterSecurityInterceptor   │
└─────────────────────────────────┘
     ↓
Your Controller (if allowed)
```

---

## Authentication vs Authorization

### Q3: What is the difference between Authentication and Authorization?

- **Authentication** = "Who are you?" — verify identity (Login) → failure returns **401**
- **Authorization** = "What can you do?" — check permissions (Roles) → failure returns **403**

| | Authentication | Authorization |
|--|----------------|---------------|
| **Question** | Who are you? | What can you access? |
| **When** | First | After authentication |
| **Failure** | 401 Unauthorized | 403 Forbidden |
| **Spring class** | `AuthenticationManager` | `AccessDecisionManager` |

---

## JWT (JSON Web Token)

### Q4: What is JWT and how does it work?

JWT is like a signed ID card. After login the server issues a token; the client sends it with every request. The server verifies the signature — no database lookup needed (stateless).

**JWT Structure** — 3 parts separated by `.`:
```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhbWl0aCIsInJvbGUiOiJVU0VSIn0.abc123
      HEADER                       PAYLOAD                    SIGNATURE
```

```json
// Header
{ "alg": "HS256", "typ": "JWT" }

// Payload (Claims)
{ "sub": "amith", "role": "USER", "iat": 1715000000, "exp": 1715086400 }

// Signature = HMAC_SHA256(base64(header) + "." + base64(payload), SECRET_KEY)
```

> **Important:** JWT payload is NOT encrypted — it is base64 encoded. Anyone can decode it. Never store passwords in JWT.

---

### Q5: What is the JWT authentication flow?

```
Client                                  Server
  │  POST /login {username, password}     │
  │ ─────────────────────────────────>   │  Validate credentials, generate JWT
  │  { token: "eyJhbGciO..." }           │
  │ <─────────────────────────────────   │
  │                                       │
  │  GET /api/users                       │
  │  Authorization: Bearer eyJhbGciO...  │
  │ ─────────────────────────────────>   │  Verify signature, extract user
  │  { users: [...] }                     │
  │ <─────────────────────────────────   │
```

---

## Spring Security with JWT Implementation

### Q6: How do you implement JWT in Spring Boot?

**Step 1: Dependencies**
```xml
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
@Component
public class JwtUtils {

    @Value("${jwt.secret}")
    private String jwtSecret;

    @Value("${jwt.expiration}")
    private int jwtExpirationMs;

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(jwtSecret.getBytes(StandardCharsets.UTF_8));
    }

    public String generateToken(String username) {
        return Jwts.builder()
                .subject(username)
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + jwtExpirationMs))
                .signWith(getSigningKey())
                .compact();
    }

    public String getUsernameFromToken(String token) {
        return Jwts.parser()
                .verifyWith(getSigningKey()).build()
                .parseSignedClaims(token)
                .getPayload().getSubject();
    }

    public boolean validateToken(String token) {
        try {
            Jwts.parser().verifyWith(getSigningKey()).build().parseSignedClaims(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
}
```

**Step 3: JWT Filter**
```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    @Autowired private JwtUtils jwtUtils;
    @Autowired private UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);

            if (jwtUtils.validateToken(token)) {
                String username = jwtUtils.getUsernameFromToken(token);
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                UsernamePasswordAuthenticationToken auth =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                SecurityContextHolder.getContext().setAuthentication(auth);
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

**Step 4: Security Configuration**
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired private JwtAuthFilter jwtAuthFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
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
        return new BCryptPasswordEncoder();
    }
}
```

**Step 5: Auth Controller**
```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @Autowired private AuthenticationManager authenticationManager;
    @Autowired private JwtUtils jwtUtils;

    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest loginRequest) {
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                loginRequest.getUsername(), loginRequest.getPassword()));

        UserDetails userDetails = (UserDetails) authentication.getPrincipal();
        String token = jwtUtils.generateToken(userDetails.getUsername());
        return ResponseEntity.ok(new LoginResponse(token));
    }
}

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
// Enable in config class:
@EnableMethodSecurity

// Usage on controller methods:
@PreAuthorize("isAuthenticated()")              // must be logged in
@PreAuthorize("hasRole('ADMIN')")               // admin only
@PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER')")  // either role
@PreAuthorize("hasRole('ADMIN') or #id == authentication.name")  // own data or admin
```

---

## CORS & CSRF

### Q8: What is CORS and how do you configure it?

CORS (Cross-Origin Resource Sharing) controls which domains can call your API. Browsers block cross-origin requests by default — e.g., a frontend on `localhost:3000` calling a backend on `localhost:8080`.

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(Arrays.asList("http://localhost:3000", "https://myproduction-site.com"));
    config.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(Arrays.asList("*"));
    config.setAllowCredentials(true);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}

// In SecurityConfig:
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
```

For a single controller: `@CrossOrigin(origins = "http://localhost:3000")`

---

### Q9: What is CSRF and why is it disabled for REST APIs?

CSRF (Cross-Site Request Forgery) is an attack where a malicious site tricks a browser into making requests using the victim's cookies. REST APIs use JWT in headers, not cookies, so CSRF is not a threat — disable it.

```java
// REST API with JWT — disable CSRF:
http.csrf(csrf -> csrf.disable());

// Traditional web app with form login — keep CSRF:
http.csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()));
```

---

## Password Encoding

### Q10: How do you store passwords securely?

Never store plain text passwords. Always hash with BCrypt.

```java
// Encoding on registration:
user.setPassword(passwordEncoder.encode("mypassword123"));
// Stored as: $2a$10$xyz...  (60-char hash)

// Verifying on login:
boolean matches = passwordEncoder.matches("mypassword123", storedHash);

// Bean definition:
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12); // strength 10-12 recommended
}
```

BCrypt properties: each hash is unique (salt included), same password produces different hashes, one-way (irreversible), intentionally slow to resist brute force.

---

## Quick Revision Summary

| Concept | Simple Explanation |
|---------|-------------------|
| **Authentication** | Login — verify who you are |
| **Authorization** | Roles — what you can access |
| **JWT** | Signed token sent with each request |
| **BCrypt** | One-way password hashing |
| **CORS** | Control which domains can call your API |
| **CSRF** | Disable for REST APIs with JWT |
| **SecurityFilterChain** | Configure URL permissions |
| **@PreAuthorize** | Method-level security |

### JWT Interview Q&A

**Q: Where do you store JWT on the client?**
> localStorage/sessionStorage for SPAs, or HttpOnly cookie for better security.

**Q: What happens when JWT expires?**
> Client receives 401 and must re-login or use a refresh token to get a new JWT.

**Q: What is a Refresh Token?**
> A long-lived token used to obtain a new JWT without requiring the user to re-login.

**Q: Can you invalidate a JWT?**
> Not directly — JWT is stateless. Solutions: short expiry, server-side blacklist (DB/Redis), or refresh token rotation.

**Q: Is JWT payload secure?**
> No. It is base64 encoded, not encrypted. Never store sensitive data in the payload.

**Q: What HTTP header carries JWT?**
> `Authorization: Bearer <token>`

### Full JWT Flow in 5 Steps
```
1. POST /login → Server validates credentials
2. Server creates JWT with username + expiry, signs with secret key
3. Client saves JWT
4. Client sends: GET /api/data  +  Authorization: Bearer <token>
5. Server: validate signature → extract user → check permissions → respond
```
