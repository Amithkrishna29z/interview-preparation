# Spring OAuth2, OIDC & Security – Full Stack Java Developer Interview Guide

> Complete reference for OAuth2, OpenID Connect, JWT, Spring Security OAuth2, and related security patterns. Covers concepts, code, and interview Q&A, trimmed to junior Spring Boot scope.

---

## Table of Contents

1. [OAuth2 Fundamentals](#1-oauth2-fundamentals)
2. [OAuth2 Grant Types (Flows)](#2-oauth2-grant-types-flows)
3. [OpenID Connect (OIDC)](#3-openid-connect-oidc)
4. [JWT Deep Dive](#4-jwt-deep-dive)
5. [Spring Security OAuth2 Resource Server](#5-spring-security-oauth2-resource-server)
6. [Spring Security OAuth2 Client](#6-spring-security-oauth2-client)
7. [Social Login: Google & GitHub](#7-social-login-google--github)
8. [Spring Authorization Server](#8-spring-authorization-server)
9. [Method-Level Security](#9-method-level-security)
10. [CORS and CSRF](#10-cors-and-csrf)
11. [Refresh Token Flow](#11-refresh-token-flow)
12. [Security Best Practices](#12-security-best-practices)
13. [Keycloak Integration](#13-keycloak-integration)
14. [Microservice Security Patterns](#14-microservice-security-patterns)
15. [Interview Questions & Answers](#15-interview-questions--answers)
16. [Quick Revision Summary](#16-quick-revision-summary)

---

## 1. OAuth2 Fundamentals

### 1.1 What is OAuth2?

OAuth2 is an **authorization framework** — NOT an authentication protocol. It lets a third-party app get limited access to a user's resources on another service, without the user sharing their password.

**Key distinction:**
- **Authentication** = "Who are you?" (verifying identity)
- **Authorization** = "What are you allowed to do?" (granting permissions)
- OAuth2 handles *authorization only*. OIDC (built on top) adds authentication.

---

### 1.2 OAuth2 Roles

| Role | Description | Example |
|------|-------------|---------|
| **Resource Owner** | The user who owns the data | You (the end user) |
| **Client** | The app requesting access | A third-party web app |
| **Authorization Server** | Issues tokens after user consent | Google, Okta, Keycloak |
| **Resource Server** | Hosts the protected resources | Google Drive API, your backend API |

```
Resource Owner (User)
        |
        | grants consent
        v
Authorization Server ----token----> Client App
                                        |
                                        | uses token
                                        v
                                  Resource Server
                                  (your protected API)
```

---

### 1.3 OAuth2 Tokens

#### Access Token
- Short-lived (typically 5–60 minutes)
- Sent in the `Authorization: Bearer <token>` header
- Can be opaque (random string) or JWT

#### Refresh Token
- Long-lived (hours, days, or weeks)
- Used to get a new Access Token without re-authenticating
- Never sent to Resource Servers — only to the Authorization Server's `/token` endpoint

#### Authorization Code
- Very short-lived (typically 10 minutes, single-use)
- Exchanged for Access + Refresh tokens server-side

#### Token Format: Opaque vs JWT

| Feature | Opaque Token | JWT |
|---------|-------------|-----|
| Format | Random string | `header.payload.signature` |
| Validation | Requires introspection call to auth server | Self-contained, verified locally |
| Revocation | Easy (delete from DB) | Hard (needs blacklist or short TTL) |
| Performance | Slower (network call per request) | Faster (no network call) |
| Use case | When you need instant revocation | Stateless microservices |

---

## 2. OAuth2 Grant Types (Flows)

### 2.1 Authorization Code Flow

**Best for:** Web applications with a backend server (confidential clients)

**Key HTTP steps:**

```http
# 1. Redirect to /authorize (front channel)
GET https://auth.example.com/authorize?response_type=code&client_id=my-client-id
    &redirect_uri=https://myapp.com/callback&scope=openid%20profile%20email&state=xK3mP9qRzY2n

# 2. Auth server redirects back with: ?code=AUTH_CODE&state=xK3mP9qRzY2n

# 3. Exchange code for tokens (back channel, server-to-server)
POST https://auth.example.com/token
grant_type=authorization_code&code=...&redirect_uri=https://myapp.com/callback
&client_id=my-client-id&client_secret=my-client-secret

# 4. Response: { access_token, token_type: "Bearer", expires_in, refresh_token, id_token, scope }
```

**Why the `state` parameter?** Prevents CSRF attacks on the OAuth2 flow. Your app generates a random value, stores it in session, and verifies it matches when the redirect comes back.

**Why the authorization code (not token directly)?** The token never passes through the browser (front channel). The code is exchanged server-to-server (back channel) where the client secret can be used securely.

---

### 2.2 Authorization Code Flow with PKCE

**PKCE** = Proof Key for Code Exchange. **Best for:** SPAs, mobile apps — clients that cannot keep a `client_secret`.

```
1. Client generates random: code_verifier
2. Hashes it:  code_challenge = BASE64URL(SHA256(code_verifier))
3. Sends code_challenge in /authorize request
4. Auth Server stores it alongside the auth code
5. Client sends code_verifier (not the hash) in /token request
6. Auth Server hashes verifier and compares — match required
```

The `/authorize` request adds `code_challenge=<hash>&code_challenge_method=S256`. The `/token` request adds `code_verifier=<original>` and needs **no** client_secret. Always use `S256` (not `plain`).

---

### 2.3 Client Credentials Flow

**Best for:** Machine-to-machine (M2M) communication — no user involved.

```http
POST https://auth.example.com/token
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(client_id:client_secret)

grant_type=client_credentials&scope=read:data write:data
```

**Note:** No refresh token in client credentials flow — just request a new access token when it expires.

---

### 2.4 Device Authorization Flow (Awareness)

For devices with no browser (smart TVs, IoT, CLI tools). The device shows a short `user_code` and URL; the user opens the URL on another device and approves. The device polls `/token` until tokens are issued.

---

### 2.5 Implicit Flow (Deprecated)

Access token returned directly in the URL fragment — exposed in browser history and logs. Replaced by Authorization Code + PKCE. **Never use for new applications.**

---

### 2.6 Resource Owner Password Credentials (Deprecated)

The client receives the user's actual password — defeats the entire purpose of OAuth2. Only acceptable for legacy first-party apps you fully control, and even then discouraged.

---

## 3. OpenID Connect (OIDC)

### 3.1 What is OIDC?

OpenID Connect (OIDC) = **OAuth2 + Identity Layer**

OIDC adds:
1. **ID Token** — a JWT containing user identity claims
2. **UserInfo endpoint** — get user profile information
3. **Discovery document** — machine-readable configuration at `/.well-known/openid-configuration`
4. **Standardized scopes** — `openid`, `profile`, `email`

---

### 3.2 ID Token vs Access Token

| Feature | ID Token | Access Token |
|---------|----------|--------------|
| Purpose | Identity proof (who the user is) | Authorization (what they can do) |
| Format | Always JWT | JWT or opaque |
| Audience (`aud`) | The client application | The Resource Server (API) |
| Contains | User identity claims (sub, email, name) | Scopes, permissions |
| Should be sent to API? | **NO** | **YES** |

**Critical rule:** Never send the ID Token to your Resource Server. The Access Token is what goes to APIs.

---

### 3.3 ID Token Claims

```json
{
  "iss": "https://accounts.google.com",
  "sub": "110169484474386276334",
  "aud": "my-client-id.apps.googleusercontent.com",
  "exp": 1735600000,
  "iat": 1735596400,
  "nonce": "random-nonce-value",
  "email": "amith@gmail.com",
  "email_verified": true,
  "name": "Amith Krishnan"
}
```

| Claim | Meaning | Required? |
|-------|---------|-----------|
| `iss` | Issuer | Yes |
| `sub` | Unique user identifier | Yes |
| `aud` | Which client this token is for | Yes |
| `exp` | Expiration time | Yes |
| `iat` | Issued at | Yes |
| `nonce` | Replay attack prevention | If sent in request |

---

### 3.4 OIDC Scopes

| Scope | Claims returned |
|-------|----------------|
| `openid` | `sub` (required for OIDC) |
| `profile` | `name`, `given_name`, `family_name`, `picture`, `locale` |
| `email` | `email`, `email_verified` |
| `offline_access` | Requests a refresh token |

---

### 3.5 UserInfo Endpoint

```http
GET https://accounts.google.com/oauth2/v3/userinfo
Authorization: Bearer <access_token>
```

---

### 3.6 Discovery Document

Every OIDC provider exposes:
```
GET https://accounts.google.com/.well-known/openid-configuration
GET https://your-keycloak/realms/myrealm/.well-known/openid-configuration
```

Spring Boot uses `issuer-uri` to auto-discover all endpoints (`authorization_endpoint`, `token_endpoint`, `jwks_uri`, etc.) from this document.

---

### 3.7 Nonce — Replay Attack Prevention

A random value sent in the `/authorize` request and embedded in the ID Token. The client verifies it matches what it sent, preventing replay of captured ID Tokens.

---

## 4. JWT Deep Dive

### 4.1 JWT Structure

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9 . eyJzdWIiOiIxMjMifQ . <signature>
        ^HEADER^                                ^PAYLOAD^           ^SIGNATURE^
```

**Header:**
```json
{ "alg": "RS256", "typ": "JWT", "kid": "key-id-1" }
```

**Payload (Claims):**
```json
{
  "iss": "https://auth.example.com",
  "sub": "user-123",
  "aud": ["https://api.example.com"],
  "exp": 1735600000,
  "iat": 1735596400,
  "jti": "unique-token-id-abc123",
  "scope": "read:profile",
  "roles": ["USER", "ADMIN"]
}
```

| Claim | Description |
|-------|-------------|
| `iss` | Issuer |
| `sub` | Subject (user ID) |
| `aud` | Audience (intended recipient) |
| `exp` | Expiration (Unix time) |
| `nbf` | Not Before |
| `iat` | Issued At |
| `jti` | Unique token ID (for blacklisting) |

---

### 4.2 RS256 vs HS256

| Feature | HS256 (Symmetric) | RS256 (Asymmetric) |
|---------|-------------------|-------------------|
| Key type | Single shared secret | Private key (sign) + Public key (verify) |
| Who can verify | Only parties with the secret | Anyone with the public key |
| Use case | Single service | Distributed/microservices |

**Why RS256 is preferred:** The Authorization Server holds the private key; Resource Servers verify with the public key fetched from the JWKS endpoint. No shared secrets needed across services.

---

### 4.3 JWT Validation Steps

```
1. Decode the JWT (verify it's well-formed)
2. Verify signature using the issuer's public key (from JWKS)
3. Check exp — reject if expired
4. Check nbf — reject if too early
5. Check iss — must match expected issuer
6. Check aud — must include this server's identifier
7. Check jti (optional) — for replay prevention with a blacklist
```

**Never skip audience validation.** An access token issued for Service A must be rejected by Service B because `aud` won't match.

---

### 4.4 JWKS Endpoint

```
GET https://auth.example.com/.well-known/jwks.json
```

Returns the public keys used to verify JWT signatures. `kid` in the JWT header matches a key in the JWKS. Spring Boot's `NimbusJwtDecoder` fetches and caches these keys automatically.

---

### 4.5 JWT Security Vulnerabilities (Awareness)

Spring's `NimbusJwtDecoder` is safe against these when you pin the algorithm:
- **`alg: none`** — never accept unsigned tokens; specify allowed algorithms explicitly.
- **RS256 → HS256 confusion** — pin the expected algorithm.
- **Token leakage via logs** — always send tokens in headers, never in URLs.

---

## 5. Spring Security OAuth2 Resource Server

### 5.1 Maven Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

---

### 5.2 Minimal Configuration

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://accounts.google.com
          # Spring auto-discovers jwks-uri from /.well-known/openid-configuration
```

---

### 5.3 SecurityConfig — Complete Example

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(csrf -> csrf.disable())
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter()))
            );
        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(new KeycloakRoleConverter());
        return converter;
    }

    // Reads roles from Keycloak's realm_access.roles claim
    static class KeycloakRoleConverter implements Converter<Jwt, Collection<GrantedAuthority>> {
        @Override
        public Collection<GrantedAuthority> convert(Jwt jwt) {
            Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
            if (realmAccess == null) return List.of();
            @SuppressWarnings("unchecked")
            List<String> roles = (List<String>) realmAccess.get("roles");
            return roles.stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
                .collect(Collectors.toList());
        }
    }
}
```

---

### 5.4 Custom JWT Decoder with Audience Validation

```java
@Bean
public JwtDecoder jwtDecoder() {
    NimbusJwtDecoder jwtDecoder = NimbusJwtDecoder
        .withJwkSetUri("https://auth.example.com/.well-known/jwks.json")
        .build();

    OAuth2TokenValidator<Jwt> withIssuer =
        JwtValidators.createDefaultWithIssuer("https://auth.example.com");
    OAuth2TokenValidator<Jwt> withAudience =
        new JwtClaimValidator<List<String>>(JwtClaimNames.AUD,
            aud -> aud != null && aud.contains("my-api-resource"));

    jwtDecoder.setJwtValidator(
        new DelegatingOAuth2TokenValidator<>(withIssuer, withAudience));
    return jwtDecoder;
}
```

---

### 5.5 Accessing JWT Claims in a Controller

```java
@RestController
@RequestMapping("/api")
public class UserController {

    @GetMapping("/me")
    public Map<String, Object> getMe(@AuthenticationPrincipal Jwt jwt) {
        return Map.of(
            "userId", jwt.getSubject(),
            "email", jwt.getClaimAsString("email"),
            "roles", jwt.getClaimAsStringList("roles")
        );
    }
}
```

---

### 5.6 Opaque Token (Introspection) Alternative

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          introspection-uri: https://auth.example.com/oauth2/introspect
          client-id: my-resource-server
          client-secret: my-secret
```

Introspection calls the auth server for every API request — slower but allows instant revocation.

---

## 6. Spring Security OAuth2 Client

### 6.1 Maven Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

---

### 6.2 application.yml — Multiple Providers

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email

          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: read:user, user:email

          keycloak:
            client-id: my-app
            client-secret: ${KEYCLOAK_SECRET}
            authorization-grant-type: authorization_code
            scope: openid, profile, email
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"

          # Client Credentials (M2M)
          service-client:
            client-id: my-service
            client-secret: ${SERVICE_SECRET}
            authorization-grant-type: client_credentials
            scope: read:data

        provider:
          keycloak:
            issuer-uri: http://localhost:8080/realms/myrealm
```

---

### 6.3 OAuth2 Login Security Config

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/", "/login", "/error").permitAll()
            .anyRequest().authenticated()
        )
        .oauth2Login(oauth2 -> oauth2
            .loginPage("/login")
            .defaultSuccessUrl("/dashboard", true)
            .failureUrl("/login?error=true")
        )
        .logout(logout -> logout.logoutSuccessUrl("/").invalidateHttpSession(true));
    return http.build();
}
```

---

### 6.4 Accessing OAuth2 User in Controller

```java
@GetMapping("/dashboard")
public String dashboard(@AuthenticationPrincipal OidcUser oidcUser, Model model) {
    model.addAttribute("name", oidcUser.getFullName());
    model.addAttribute("email", oidcUser.getEmail());
    return "dashboard";
}

@GetMapping("/github-user")
public String githubUser(@AuthenticationPrincipal OAuth2User oauth2User, Model model) {
    model.addAttribute("login", oauth2User.getAttribute("login"));
    return "profile";
}
```

Use `OidcUser` for OIDC providers (Google, Keycloak) and `OAuth2User` for non-OIDC (GitHub).

---

### 6.5 Making Outbound API Calls with OAuth2 Token

```java
@Bean
public WebClient webClient(OAuth2AuthorizedClientManager authorizedClientManager) {
    ServletOAuth2AuthorizedClientExchangeFilterFunction oauth2 =
        new ServletOAuth2AuthorizedClientExchangeFilterFunction(authorizedClientManager);
    oauth2.setDefaultClientRegistrationId("service-client");
    return WebClient.builder().apply(oauth2.oauth2Configuration()).build();
}
```

Usage in a service:
```java
webClient.get()
    .uri("https://api.example.com/data")
    .attributes(clientRegistrationId("service-client"))
    .retrieve()
    .bodyToMono(String.class);
```

---

## 7. Social Login: Google & GitHub

**Setup (Google):** In Google Cloud Console, create an OAuth 2.0 Client ID and add redirect URI `http://localhost:8080/login/oauth2/code/google`. For GitHub: Developer settings → OAuth Apps, callback `http://localhost:8080/login/oauth2/code/github`.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/", "/login").permitAll()
            .anyRequest().authenticated()
        )
        .oauth2Login(Customizer.withDefaults());
    return http.build();
}
```

**Persisting users (awareness):** Extend `DefaultOAuth2UserService` (or `OidcUserService` for OIDC), override `loadUser`, call `super.loadUser(...)`, then find-or-create a local `User` keyed by provider + provider-id. Register it via `.oauth2Login(o -> o.userInfoEndpoint(u -> u.userService(customUserService)))`.

---

## 8. Spring Authorization Server

### 8.1 What is it?

Spring Authorization Server (SAS) is the official OAuth2/OIDC Authorization Server from the Spring team. Use it when you want to build your own auth server instead of using Keycloak or a cloud provider.

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-oauth2-authorization-server</artifactId>
</dependency>
```

---

### 8.2 Key Beans (Awareness)

Building a full auth server is senior-level; know the main beans:

- `SecurityFilterChain` with `OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http)` and `.oidc(Customizer.withDefaults())`
- `RegisteredClientRepository` — describes each client: grant types, redirect URIs, scopes, PKCE requirement, token TTLs
- `JWKSource` — RSA key pair for signing tokens (public key exposed at JWKS endpoint)
- `AuthorizationServerSettings.builder().issuer("http://localhost:9000").build()`
- `OAuth2TokenCustomizer<JwtEncodingContext>` — inject custom claims (e.g. `roles`) into access tokens

---

### 8.3 Keycloak vs Spring Authorization Server

| Feature | Keycloak | Spring Authorization Server |
|---------|---------|---------------------------|
| Setup | Separate server, Docker/K8s | Embedded in Spring Boot app |
| Features | Rich (users, groups, MFA, social login, admin UI) | Basic OAuth2/OIDC |
| Customization | Via SPIs, complex | Full Java code control |
| Use when | Enterprise, complex user management | Greenfield, full control |

---

## 9. Method-Level Security

### 9.1 Enable Method Security

```java
@Configuration
@EnableMethodSecurity  // replaces deprecated @EnableGlobalMethodSecurity
public class MethodSecurityConfig { }
```

Enables: `@PreAuthorize`, `@PostAuthorize`, `@PreFilter`, `@PostFilter`, `@Secured`, `@RolesAllowed`.

---

### 9.2 @PreAuthorize

Evaluated **before** the method executes. If false, method is not called.

```java
@PreAuthorize("hasRole('ADMIN')")
public List<User> getAllUsers() { ... }

// SpEL with method parameter
@PreAuthorize("#userId == authentication.principal.id")
public UserProfile getProfile(Long userId) { ... }

// Multiple conditions
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
public void updateUser(Long userId, UserUpdateRequest request) { ... }

// Custom bean check
@PreAuthorize("@securityService.canAccessDocument(authentication, #docId)")
public Document getDocument(Long docId) { ... }
```

---

### 9.3 @PostAuthorize

Evaluated **after** the method, with access to `returnObject`.

```java
@PostAuthorize("returnObject.ownerId == authentication.principal.id or hasRole('ADMIN')")
public Order getOrder(Long orderId) { ... }
```

**Warning:** The method still executes. Any side effects happen before the check. Prefer `@PreAuthorize` when possible.

---

### 9.4 @PreFilter and @PostFilter

```java
// Filter input list — only process items owned by current user
@PreFilter("filterObject.ownerId == authentication.principal.id")
public void processOrders(List<Order> orders) { ... }

// Filter output list
@PostFilter("filterObject.status == 'PUBLIC' or filterObject.ownerId == authentication.principal.id")
public List<Document> getAllDocuments() { ... }
```

**Note:** `@PostFilter` is inefficient on large datasets — better to filter at the query level.

---

### 9.5 Annotation Comparison

| Feature | @PreAuthorize | @Secured | @RolesAllowed |
|---------|--------------|----------|---------------|
| SpEL support | Yes | No | No |
| Method params | Yes (#param) | No | No |
| Standard | Spring | Spring | JSR-250 |
| Recommendation | Preferred | Legacy | Use for portability |

---

### 9.6 Role Hierarchy

Without hierarchy, `ADMIN` does NOT automatically have `USER` permissions.

```java
@Bean
public RoleHierarchy roleHierarchy() {
    RoleHierarchyImpl hierarchy = new RoleHierarchyImpl();
    hierarchy.setHierarchy(
        "ROLE_ADMIN > ROLE_MANAGER\n" +
        "ROLE_MANAGER > ROLE_USER\n" +
        "ROLE_USER > ROLE_GUEST"
    );
    return hierarchy;
}

@Bean
public MethodSecurityExpressionHandler methodSecurityExpressionHandler(RoleHierarchy roleHierarchy) {
    DefaultMethodSecurityExpressionHandler handler = new DefaultMethodSecurityExpressionHandler();
    handler.setRoleHierarchy(roleHierarchy);
    return handler;
}
```

---

## 10. CORS and CSRF

### 10.1 What is CORS?

**CORS** = Cross-Origin Resource Sharing. Browsers block JavaScript from calling a different origin (domain, protocol, or port). CORS is a *browser* feature — Postman and server-to-server calls are unaffected.

For non-simple requests (PUT, DELETE, custom headers), the browser sends a preflight `OPTIONS` request first.

---

### 10.2 CORS Configuration in Spring Boot

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://frontend.example.com", "http://localhost:3000"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Requested-With"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

Register in security config: `http.cors(cors -> cors.configurationSource(corsConfigurationSource()));`

**Never use `"*"` with `allowCredentials(true)`.**

---

### 10.3 What is CSRF?

**CSRF** = Cross-Site Request Forgery. An attacker tricks a logged-in user's browser into making a request to your site — the browser automatically includes the session cookie.

Spring Security uses the **Synchronizer Token Pattern**: a random CSRF token is embedded in every HTML form and verified on every state-changing request. `evil.com` can't read the token due to same-origin policy.

---

### 10.4 When to Disable CSRF

```java
// DISABLE for stateless REST APIs using JWT in Authorization header
http.csrf(csrf -> csrf.disable());
```

**Why:** CSRF exploits automatic cookie sending. JWT is sent via the `Authorization` header — browsers cannot set custom headers cross-origin without explicit CORS permission. **If you store JWT in cookies (even HttpOnly), keep CSRF enabled.**

---

### 10.5 SameSite Cookies as CSRF Defense

```yaml
server:
  servlet:
    session:
      cookie:
        same-site: strict
        http-only: true
        secure: true
```

- `SameSite=Strict`: cookie not sent on any cross-origin request
- `SameSite=Lax`: only on top-level navigation GETs (safe for CSRF)

---

## 11. Refresh Token Flow

### 11.1 Why Refresh Tokens?

Short-lived access tokens (15 min) minimize damage if stolen, but re-authenticating every 15 minutes ruins UX. Refresh tokens solve this without exposing the user's credentials.

### 11.2 Refresh Token Request

```http
POST https://auth.example.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
&client_id=my-client-id
&client_secret=my-client-secret
```

---

### 11.3 Refresh Token Rotation

Every time a refresh token is used, a **new** refresh token is issued (old one invalidated). If a stolen token is used by an attacker, the legitimate user's next refresh fails — detecting the theft.

```java
.tokenSettings(TokenSettings.builder()
    .reuseRefreshTokens(false)  // enable rotation
    .build())
```

---

### 11.4 Refresh Token Storage

| Storage | XSS Risk | CSRF Risk | Notes |
|---------|---------|-----------|-------|
| `localStorage` | HIGH | Low | Never use |
| Memory (JS variable) | Low | Low | Lost on page refresh |
| HttpOnly + SameSite=Strict cookie | None | None | Recommended |

**Best practice:**
- Access token: in memory (JavaScript variable)
- Refresh token: HttpOnly, Secure, SameSite=Strict cookie

---

### 11.5 Spring Boot Refresh Token Implementation

```java
@PostMapping("/refresh")
public ResponseEntity<TokenResponse> refreshToken(
        @CookieValue(name = "refresh_token", required = false) String refreshToken,
        HttpServletResponse response) {

    if (refreshToken == null)
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();

    RefreshToken storedToken = refreshTokenService.findByToken(refreshToken)
        .orElseThrow(() -> new RuntimeException("Invalid refresh token"));

    if (storedToken.isExpired()) {
        refreshTokenService.delete(storedToken);
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
    }

    // Rotate: delete old, create new
    refreshTokenService.delete(storedToken);
    String newRefreshToken = refreshTokenService.createRefreshToken(storedToken.getUser());

    Cookie cookie = new Cookie("refresh_token", newRefreshToken);
    cookie.setHttpOnly(true);
    cookie.setSecure(true);
    cookie.setPath("/api/auth/refresh");
    cookie.setMaxAge(7 * 24 * 60 * 60);
    response.addCookie(cookie);

    return ResponseEntity.ok(new TokenResponse(jwtService.generateToken(storedToken.getUser())));
}
```

---

## 12. Security Best Practices

### 12.1 Token Storage

```
DO:
  - Access tokens: memory (JavaScript variable)
  - Refresh tokens: HttpOnly, Secure, SameSite=Strict cookies

DON'T:
  - Never store tokens in localStorage (XSS risk)
  - Never put tokens in URL query params (log exposure)
```

**XSS rule:** If JavaScript can read it, XSS can steal it. HttpOnly cookies are the only browser storage JavaScript cannot access.

---

### 12.2 Token Lifetimes

```
Access Token:  5–15 minutes
Refresh Token: 1–7 days
```

---

### 12.3 Token Revocation Strategies

| Strategy | How | Tradeoff |
|---------|-----|---------|
| Short TTL | 5-15 min expiry | No instant revocation, but low risk window |
| Introspection | Call auth server per request | Instant revocation, adds latency |
| Redis blacklist | Store revoked JTIs in Redis | Near-instant, needs Redis |
| Refresh token rotation | Revoke refresh → re-authenticate | Next refresh fails |

---

### 12.4 Key Security Rules

```
1. PKCE for all public clients (no exceptions, use S256)
2. Always validate aud claim
3. Disable CSRF for header-based JWT APIs
4. RS256 over HS256 (no shared secrets)
5. Use state parameter always
6. HTTPS everywhere
7. Validate all JWT claims: sig → exp → iss → aud
```

---

### 12.5 Secure Headers

```java
http.headers(headers -> headers
    .frameOptions(frame -> frame.deny())
    .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
    .httpStrictTransportSecurity(hsts -> hsts.includeSubDomains(true).maxAgeInSeconds(31536000))
);
```

---

## 13. Keycloak Integration

### 13.1 Core Concepts

| Concept | Description |
|---------|-------------|
| **Realm** | Isolated tenant — separate users, clients, settings |
| **Client** | Your application registered in Keycloak |
| **Role** | Permission label (realm role or client role) |
| **Group** | Collection of users |
| **Identity Provider** | External auth (Google, LDAP) fed into Keycloak |

---

### 13.2 Resource Server Config

Run Keycloak via Docker, create a realm and client in the admin console, then:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/myrealm
```

Keycloak stores roles in `realm_access.roles` and `resource_access.<client>.roles` — use the `KeycloakRoleConverter` from Section 5.3 to map them to `ROLE_*` authorities.

---

### 13.3 Keycloak JWT Structure

```json
{
  "iss": "http://localhost:8080/realms/myrealm",
  "sub": "user-uuid-here",
  "aud": ["my-spring-app", "account"],
  "realm_access": { "roles": ["offline_access", "USER"] },
  "resource_access": {
    "my-spring-app": { "roles": ["MANAGER"] }
  },
  "scope": "openid email profile",
  "preferred_username": "amith",
  "email": "amith@example.com"
}
```

---

## 14. Microservice Security Patterns

These are architecture-level concerns; know them at an awareness level.

### 14.1 Token Propagation

When Service A calls Service B on behalf of a user, forward the user's JWT in the `Authorization: Bearer` header. Pull it from `SecurityContextHolder` inside a Feign `RequestInterceptor` or WebClient filter.

### 14.2 Service-to-Service (Client Credentials)

For calls with no user context (background jobs, internal APIs), use a `client_credentials` client. Let `ServletOAuth2AuthorizedClientExchangeFilterFunction` attach a service token to outbound WebClient calls automatically.

### 14.3 API Gateway Pattern

```
Client ──JWT──> API Gateway (validates JWT, extracts claims)
                      │  passes X-User-Id / X-User-Roles headers
        ┌─────────────┼─────────────┐
   User Service   Order Service  Payment Service
```

Downstream services either trust gateway headers (internal network only) or re-validate the JWT independently (Zero Trust).

### 14.4 Zero Trust Security

Every service validates signature (JWKS), expiry, issuer, and audience independently — a compromised internal service still can't impersonate users.

---

## 15. Interview Questions & Answers

### Q1: What is the difference between OAuth2 and OIDC?

OAuth2 is an **authorization** framework — it answers "what can this app do on your behalf?" OIDC is an **identity layer** on top of OAuth2 that adds authentication — answering "who is this user?" OIDC introduces the ID Token (a JWT with user identity claims), the UserInfo endpoint, and standardized scopes (`openid`, `profile`, `email`). Short version: **OAuth2 = authorization only. OIDC = OAuth2 + authentication.**

---

### Q2: What is the difference between authentication and authorization?

**Authentication** = verifying who you are (credentials, OTP). **Authorization** = determining what you're allowed to do (roles, permissions). You can't authorize without first authenticating. Authentication establishes identity; authorization enforces permissions for that identity.

---

### Q3: Walk me through the Authorization Code Flow step by step.

1. App redirects user to `/authorize` with `client_id`, `redirect_uri`, `scope`, `state`, `response_type=code`
2. User logs in and grants consent at the Authorization Server
3. Auth Server redirects back with a short-lived `code` and `state`
4. App verifies `state` matches (CSRF protection)
5. App backend POSTs to `/token` with `code`, `client_id`, `client_secret`, `redirect_uri`
6. Auth Server returns `access_token`, `refresh_token`, and `id_token`
7. App uses `access_token` as `Bearer` token to call the API

The code exchange happens server-to-server (back channel), so tokens never pass through the browser.

---

### Q4: What is PKCE and why was it introduced?

PKCE protects the Authorization Code Flow for **public clients** (SPAs, mobile apps) that cannot store a `client_secret`. Without PKCE, an intercepted authorization code can be exchanged for tokens. With PKCE: the client generates a random `code_verifier`, hashes it to `code_challenge` (SHA256/Base64URL), sends the challenge in `/authorize`, and sends the original verifier in `/token`. An intercepted code is useless without the `code_verifier`, which was never transmitted.

---

### Q5: When would you use Client Credentials vs Authorization Code flow?

**Authorization Code:** A user is involved — a web app accesses Google Drive on behalf of the user who consents. **Client Credentials:** No user — machine-to-machine. Order Service calls Inventory Service; both are servers. The calling service authenticates with its own `client_id` + `client_secret`. Rule: *Is a human user granting access?* Yes → Authorization Code. No → Client Credentials.

---

### Q6: What is in a JWT? What is the difference between RS256 and HS256?

A JWT has three Base64URL-encoded parts: `header.payload.signature`. Header contains `alg`, `typ`, `kid`. Payload contains claims: `iss`, `sub`, `aud`, `exp`, `iat`, and custom claims like `roles`. The signature is cryptographic proof of integrity.

**HS256:** Single shared secret for signing and verifying — all services need the secret (security risk). **RS256:** Private key signs; public key (freely distributed via JWKS) verifies. No shared secrets across services. RS256 is preferred for OAuth2/microservices.

---

### Q7: How does a Resource Server validate a JWT?

1. Decode header → get `alg` and `kid`
2. Fetch public key from JWKS endpoint using `kid`
3. Verify signature
4. Check `exp` (expired?), `nbf` (not yet valid?)
5. Check `iss` — must match expected issuer
6. Check `aud` — must include this server's identifier
7. Optionally check `jti` against a blacklist

Spring Boot's `NimbusJwtDecoder` handles steps 1–4 automatically; `iss` and `aud` need explicit configuration.

---

### Q8: What is the difference between @PreAuthorize and @PostAuthorize?

`@PreAuthorize` runs **before** the method — if false, the method is never called. Use it to gate access. `@PostAuthorize` runs **after** the method with access to `returnObject` — use it when authorization depends on the result. Key tradeoff: `@PostAuthorize` still executes the method, so any side effects (DB writes) happen before the check. Prefer `@PreAuthorize` whenever possible.

---

### Q9: Why should CSRF protection be disabled for stateless JWT APIs?

CSRF attacks exploit the browser automatically including cookies in cross-origin requests. JWT APIs authenticate via the `Authorization: Bearer` header — browsers cannot set custom headers cross-origin without explicit CORS permission, so a malicious page cannot forge an authenticated request. **However, if you store JWT in a cookie (even HttpOnly), you still need CSRF protection.**

---

### Q10: How do you implement OAuth2 social login in Spring Boot?

1. Add `spring-boot-starter-oauth2-client`
2. Register app with Google/GitHub (get client ID and secret)
3. Configure `application.yml` with registration details
4. Add `.oauth2Login()` to `SecurityFilterChain`
5. Optionally extend `OidcUserService` to persist users in your own DB

Spring auto-handles redirects, PKCE, callback, token exchange, and UserInfo loading. Authenticated user is `@AuthenticationPrincipal OidcUser` (OIDC) or `OAuth2User` (non-OIDC like GitHub).

---

### Q11: What is refresh token rotation?

Every time a refresh token is used, the server **invalidates the old one** and issues a **new one**. If a refresh token is stolen and the attacker uses it first, the real user's next refresh fails — the server detects that an already-used token was presented, suggesting theft, and can invalidate the entire token family. Configure in Spring Authorization Server: `tokenSettings.reuseRefreshTokens(false)`.

---

### Q12: What is the difference between access token and ID token?

| | Access Token | ID Token |
|-|-------------|---------|
| Purpose | Prove authorization to access an API | Prove the user's identity |
| For | Resource Server (your API) | Client application |
| Should be sent to API? | YES | NO |
| Format | JWT or opaque | Always JWT |

The ID Token tells your app "the user is Amith Krishnan with this email." The Access Token grants permission to call APIs. Never send the ID Token to API endpoints.

---

### Q13: What are the risks of storing JWT in localStorage?

Any XSS vulnerability (even in a third-party library) can call `localStorage.getItem('access_token')` and exfiltrate the token. Better alternatives: store access token in memory (lost on refresh but XSS-resistant); store refresh token in HttpOnly cookie (JavaScript cannot access HttpOnly cookies at all). Best balance: access token in memory + refresh token in HttpOnly SameSite=Strict cookie.

---

### Q14: What is the state parameter in OAuth2, and why is it important?

A random unguessable value generated before redirecting to the Authorization Server, included in `/authorize` and echoed back in the callback. The client stores it in session and verifies it matches. Without `state`, an attacker could craft a callback URL and trick a user into linking their account to the attacker's — preventing account hijacking attacks on the OAuth2 flow itself.

---

### Q15: What is the difference between roles and authorities in Spring Security?

Both are `GrantedAuthority` — the difference is naming convention. **Role:** has `ROLE_` prefix (`ROLE_ADMIN`), checked with `hasRole('ADMIN')` (Spring adds the prefix). **Authority/Permission:** no prefix (`READ_USER`, `WRITE_ORDER`), checked with `hasAuthority('READ_USER')`. Use roles for coarse-grained access (ADMIN, USER) and authorities for fine-grained permissions (READ_USER, DELETE_ORDER).

---

### Q16: How does Spring Security's filter chain work with JWT?

1. `BearerTokenAuthenticationFilter` extracts `Authorization: Bearer <token>` header
2. Calls `JwtDecoder.decode()` — validates signature, expiry, issuer
3. Calls `JwtAuthenticationConverter` — converts claims to `JwtAuthenticationToken` with authorities
4. Sets `SecurityContextHolder` with the authentication
5. `AuthorizationFilter` checks `.authorizeHttpRequests()` rules
6. Request reaches the Controller if authorized

On any failure (invalid sig, expired, insufficient authority), `ExceptionTranslationFilter` returns 401 or 403.

---

### Q17: What is @EnableMethodSecurity vs @EnableGlobalMethodSecurity?

`@EnableGlobalMethodSecurity` is deprecated (Spring Security 5.6 and older). `@EnableMethodSecurity` is the modern replacement — it enables `@PreAuthorize`/`@PostAuthorize` by default (no flags needed), uses better AOP proxying, and has a more composable configuration. Old: `@EnableGlobalMethodSecurity(prePostEnabled = true)`. New: just `@EnableMethodSecurity`.

---

### Q18: What is the difference between scope and claim in OAuth2/OIDC?

**Scope:** A permission request (`scope=openid profile email`). Scopes determine which claims appear in tokens and which APIs can be accessed. **Claim:** An assertion inside a token (`"email": "amith@example.com"`, `"exp": 1735600000`). Scopes determine what claims you get — `profile` scope → you get `name`, `given_name`, `picture` claims.

---

### Q19: What is Bearer Token authentication?

"Whoever **bears** (presents) this token is granted access." No additional proof needed — just possession of the token. Format: `Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...`. Spring handles it via `BearerTokenAuthenticationFilter`. Security depends entirely on transport (HTTPS) — if stolen, the thief has full access until expiry.

---

### Q20: What happens if you don't configure CORS and your frontend is on a different origin?

The browser blocks the response (for simple requests) or the request itself (preflight fails). Importantly, CORS is browser-enforced — the request **does reach the server** and is processed, but the browser refuses to hand the response to JavaScript. Preflight OPTIONS requests are blocked before the actual request even hits the server. Fix: configure `CorsConfigurationSource` and register it with `.cors()`.

---

### Q21: What is the difference between oauth2-resource-server and oauth2-client starters?

**oauth2-resource-server:** For APIs that receive and validate tokens. Your service protects endpoints — incoming requests must have a Bearer token. **oauth2-client:** For apps that obtain tokens (social login) or call other services with tokens. A microservice API → resource-server only. A BFF (Backend For Frontend) → often both.

---

### Awareness — senior topics worth a one-liner

**Token introspection** (RFC 7662 — validate opaque tokens via `/introspect`), **JTI blacklists in Redis**, **OAuth 2.1** (PKCE required, Implicit/ROPC removed), **multi-tenancy** via `JwtIssuerAuthenticationManagerResolver`, **OIDC back-channel logout** for global SSO logout.

---

## 16. Quick Revision Summary

### Core Concepts

```
OAuth2 = Authorization framework (not authentication)
OIDC   = OAuth2 + Identity (authentication)
JWT    = Compact, self-contained token (header.payload.signature)
```

### Grant Types Quick Reference

| Flow | Use When |
|------|----------|
| Authorization Code | Web app with backend, user involved |
| Auth Code + PKCE | SPA, mobile app (no client secret possible) |
| Client Credentials | Service-to-service, no user |
| Device Code | TV, IoT, CLI tools |
| Implicit | DEPRECATED |
| ROPC | DEPRECATED |

### Token Types

| Token | TTL | Purpose |
|-------|-----|---------|
| Access Token | 5-60 min | Call APIs (Resource Server) |
| Refresh Token | 1-7 days | Get new access tokens |
| ID Token | Same as access | Client learns user identity |
| Authorization Code | 10 min, 1-use | Exchange for tokens |

### Spring Boot OAuth2 Dependencies

| Dependency | When to use |
|-----------|------------|
| `spring-boot-starter-oauth2-resource-server` | Your API validates tokens |
| `spring-boot-starter-oauth2-client` | Your app gets tokens / social login |
| `spring-security-oauth2-authorization-server` | Build your own auth server |

### Key Security Rules

```
1. PKCE for all public clients (use S256)
2. Short-lived access tokens (15 min)
3. Refresh tokens in HttpOnly SameSite=Strict cookies
4. Never store tokens in localStorage
5. Always validate aud claim
6. Disable CSRF for header-based JWT APIs
7. RS256 over HS256 (no shared secrets)
8. Use state parameter always
9. HTTPS everywhere
10. Validate all JWT claims: sig → exp → iss → aud
```

### Spring Security OAuth2 Config Cheatsheet

```java
// Resource Server (JWT)
http.oauth2ResourceServer(oauth2 -> oauth2
    .jwt(jwt -> jwt.jwtAuthenticationConverter(converter())));

// OAuth2 Client (Social Login)
http.oauth2Login(Customizer.withDefaults());

// Method Security
@EnableMethodSecurity

// CORS
http.cors(cors -> cors.configurationSource(corsSource()));

// CSRF (disable for stateless JWT APIs)
http.csrf(csrf -> csrf.disable());

// Stateless sessions
http.sessionManagement(s -> s.sessionCreationPolicy(STATELESS));
```

**JWT validation order:** decode → verify signature (JWKS) → `exp` → `nbf` → `iss` → `aud` → optional `jti` blacklist.

---

*Last Updated: 2026-06-18*
