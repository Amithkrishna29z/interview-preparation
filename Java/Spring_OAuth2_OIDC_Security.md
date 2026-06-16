# Spring OAuth2, OIDC & Security – Full Stack Java Developer Interview Guide

> Complete reference for OAuth2, OpenID Connect, JWT, Spring Security OAuth2, and related security patterns. Covers concepts, code, and 40+ interview Q&A.

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
15. [Interview Questions & Answers (40+)](#15-interview-questions--answers-40)
16. [Quick Revision Summary](#16-quick-revision-summary)

---

## 1. OAuth2 Fundamentals

### 1.1 What is OAuth2?

OAuth2 is an **authorization framework** — NOT an authentication protocol. It defines how a third-party application can obtain limited access to a user's resources on another service, without the user sharing their password.

**Real-world analogy:**
- You want to give a parking valet access to your car, but you don't want to hand over your master house key along with it.
- OAuth2 gives the valet a "valet key" — limited access (can start engine, park) without full access (can't open trunk, glove box).

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
- Used by the Client to access the Resource Server
- Sent in the `Authorization: Bearer <token>` header
- Can be opaque (random string) or JWT

#### Refresh Token
- Long-lived (hours, days, or weeks)
- Used to obtain a new Access Token without re-authenticating the user
- Never sent to Resource Servers — only to the Authorization Server's `/token` endpoint
- Must be stored securely (HttpOnly cookie or secure server-side storage)

#### Authorization Code
- Very short-lived (typically 10 minutes, single-use)
- Intermediate token exchanged for Access + Refresh tokens
- Not used to access resources directly

#### Token Format: Opaque vs JWT

| Feature | Opaque Token | JWT |
|---------|-------------|-----|
| Format | Random string (e.g., `abc123xyz`) | `header.payload.signature` |
| Validation | Requires introspection call to auth server | Self-contained, verified locally |
| Revocation | Easy (delete from DB) | Hard (needs blacklist or short TTL) |
| Performance | Slower (network call per request) | Faster (no network call) |
| Use case | When you need instant revocation | Stateless microservices |

---

### 1.4 OAuth2 vs OAuth1

| Feature | OAuth1 | OAuth2 |
|---------|--------|--------|
| Signature | HMAC-SHA1 on every request | Bearer token (SSL required) |
| Complexity | Complex (request signing) | Simpler |
| Token types | Single access token | Access + refresh tokens |
| Mobile support | Poor | Good |
| HTTPS required | No (self-signed requests) | Yes (mandatory) |
| Status | Legacy/deprecated | Current standard |

OAuth1 required cryptographic signing of every HTTP request. OAuth2 relies on HTTPS for transport security and uses bearer tokens — much simpler but requires TLS everywhere.

---

## 2. OAuth2 Grant Types (Flows)

### 2.1 Authorization Code Flow

**Best for:** Web applications with a backend server (confidential clients)

**Step-by-step flow:**

```
User         Browser         Your App (Client)     Auth Server      Resource Server
 |              |                   |                    |                 |
 |--click login->|                   |                    |                 |
 |              |--redirect to AS--->|                    |                 |
 |              |                   |--GET /authorize?-->|                 |
 |              |                   |  client_id=...     |                 |
 |              |                   |  redirect_uri=...  |                 |
 |              |                   |  response_type=code|                 |
 |              |                   |  scope=openid...   |                 |
 |              |                   |  state=random123   |                 |
 |              |<--- login page ---|                    |                 |
 |--credentials->|                   |                    |                 |
 |              |--submit form------>|                    |                 |
 |              |                   |--POST /login------->|                 |
 |              |                   |<--redirect with ----|                 |
 |              |                   |  ?code=AUTH_CODE   |                 |
 |              |                   |  &state=random123  |                 |
 |              |                   |                    |                 |
 |              |                   |--POST /token------->|                 |
 |              |                   |  grant_type=       |                 |
 |              |                   |  authorization_code|                 |
 |              |                   |  code=AUTH_CODE    |                 |
 |              |                   |  redirect_uri=...  |                 |
 |              |                   |  client_id=...     |                 |
 |              |                   |  client_secret=... |                 |
 |              |                   |<-- access_token ---|                 |
 |              |                   |    refresh_token   |                 |
 |              |                   |    id_token        |                 |
 |              |                   |                    |                 |
 |              |                   |----GET /api/data with Bearer token-->|
 |              |                   |<---protected data----------------------|
```

**HTTP Request Examples:**

Step 1 — Redirect user to Authorization Server:
```
GET https://auth.example.com/authorize?
    response_type=code
    &client_id=my-client-id
    &redirect_uri=https://myapp.com/callback
    &scope=openid%20profile%20email
    &state=xK3mP9qRzY2n
```

Step 2 — Authorization Server redirects back with code:
```
GET https://myapp.com/callback?
    code=SplxlOBeZQQYbYS6WxSbIA
    &state=xK3mP9qRzY2n
```

Step 3 — Exchange code for tokens (server-to-server):
```http
POST https://auth.example.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https://myapp.com/callback
&client_id=my-client-id
&client_secret=my-client-secret
```

Step 4 — Token response:
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
  "id_token": "eyJhbGciOiJSUzI1NiJ9...",
  "scope": "openid profile email"
}
```

**Why the `state` parameter?** Prevents CSRF attacks on the OAuth2 flow itself. Your app generates a random value, stores it in session, and verifies it matches when the redirect comes back.

**Why the authorization code (not token directly)?** The token never passes through the browser (front channel). The code is exchanged server-to-server (back channel) where the client secret can be used securely.

---

### 2.2 Authorization Code Flow with PKCE

**PKCE** = Proof Key for Code Exchange (RFC 7636)

**Best for:** SPAs (React, Angular), mobile apps, public clients (clients that cannot keep a secret)

**Problem PKCE solves:** Public clients cannot store a `client_secret` safely. An attacker who intercepts the authorization code can exchange it for tokens. PKCE prevents this.

**How it works:**

```
1. Client generates a cryptographically random string: code_verifier
   e.g., "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"

2. Client hashes it: code_challenge = BASE64URL(SHA256(code_verifier))
   e.g., "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"

3. Client sends code_challenge to /authorize (not the verifier)

4. Auth Server stores the code_challenge alongside the auth code

5. Client sends code_verifier (not the hash) to /token

6. Auth Server hashes the verifier and compares with stored challenge
   If they match → issue tokens
   If not → reject (the interceptor doesn't have code_verifier)
```

**HTTP Request Examples:**

Step 1 — Generate PKCE values (client-side):
```javascript
// JavaScript example
const codeVerifier = generateRandomString(64); // store this
const codeChallenge = base64UrlEncode(sha256(codeVerifier));
```

Step 2 — Authorization request with PKCE:
```
GET https://auth.example.com/authorize?
    response_type=code
    &client_id=my-spa-client
    &redirect_uri=https://myapp.com/callback
    &scope=openid%20profile
    &state=abc123
    &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
    &code_challenge_method=S256
```

Step 3 — Token exchange with PKCE (no client_secret needed):
```http
POST https://auth.example.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https://myapp.com/callback
&client_id=my-spa-client
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

**Key point:** `code_challenge_method=S256` means SHA-256. Plain (`plain`) is also supported but not recommended.

**PKCE vs no PKCE:**
- Without PKCE: if attacker intercepts the code → can exchange for tokens (has client_secret from decompiled mobile app)
- With PKCE: intercepted code is useless without the `code_verifier` which was never transmitted

---

### 2.3 Client Credentials Flow

**Best for:** Machine-to-machine (M2M) communication — no user involved

**Use cases:**
- Microservice A calls Microservice B's protected endpoint
- A batch job or cron job calls an API
- Backend service calls a third-party API

**Flow:**

```
Client (Service A)                      Auth Server
        |                                    |
        |---POST /token -------------------->|
        |   grant_type=client_credentials    |
        |   client_id=service-a              |
        |   client_secret=secret             |
        |   scope=read:data                  |
        |                                    |
        |<-- access_token -------------------|
        |                                    |
        |---GET /api/data (Bearer token)---->Resource Server (Service B)
```

**HTTP Request:**
```http
POST https://auth.example.com/token
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(client_id:client_secret)

grant_type=client_credentials
&scope=read:data write:data
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "read:data write:data"
}
```

**Note:** No refresh token in client credentials flow — simply request a new access token when it expires.

---

### 2.4 Device Authorization Flow (Device Code)

**Best for:** Devices with no browser or limited input (smart TVs, IoT, CLI tools)

```
Device (TV)                  Auth Server              User's Phone/Browser
    |                             |                          |
    |--POST /device_authorization>|                          |
    |  client_id=tv-app           |                          |
    |<-- device_code,             |                          |
    |    user_code=BCDF-HKMP      |                          |
    |    verification_uri=...     |                          |
    |                             |                          |
    | Show on TV:                 |                          |
    | "Go to example.com/activate |                          |
    |  and enter: BCDF-HKMP"      |                          |
    |                             |<--user enters code--------|
    |                             |<--user approves------------|
    |                             |                          |
    |--poll POST /token ---------->|                          |
    |  grant_type=urn:ietf:..     |                          |
    |  device_code=...            |                          |
    |<-- access_token (when done)-|                          |
```

---

### 2.5 Implicit Flow (Deprecated)

**Why deprecated:**
- Access token returned directly in the URL fragment (`#access_token=...`)
- Token exposed in browser history, referrer headers, server logs
- No refresh token
- Replaced by Authorization Code + PKCE for public clients

**Never use for new applications.**

---

### 2.6 Resource Owner Password Credentials (Deprecated)

```http
POST /token
grant_type=password
&username=user@example.com
&password=plaintext_password
&client_id=my-app
```

**Why deprecated:**
- The client (third-party app) receives the user's actual password
- This defeats the entire purpose of OAuth2 (delegated access without sharing credentials)
- Trust model violation: user must trust the client app with credentials
- Only acceptable for legacy first-party apps you fully control, and even then discouraged

---

## 3. OpenID Connect (OIDC)

### 3.1 What is OIDC?

OpenID Connect (OIDC) = **OAuth2 + Identity Layer**

OAuth2 tells you *what* a user can do (authorization). OIDC tells you *who* the user is (authentication).

OIDC adds:
1. **ID Token** — a JWT containing user identity claims
2. **UserInfo endpoint** — get user profile information
3. **Discovery document** — machine-readable configuration
4. **Standardized scopes** — `openid`, `profile`, `email`

```
OAuth2:   "This token allows access to read photos"
OIDC:     "This token belongs to amith@example.com, born 1995, name: Amith Krishnan"
```

---

### 3.2 ID Token vs Access Token

| Feature | ID Token | Access Token |
|---------|----------|--------------|
| Purpose | Identity proof (who the user is) | Authorization (what they can do) |
| Format | Always JWT | JWT or opaque |
| Audience (`aud`) | The client application | The Resource Server (API) |
| Consumer | The Client application | The Resource Server |
| Contains | User identity claims (sub, email, name) | Scopes, permissions |
| Should be sent to API? | NO | YES |

**Critical rule:** Never send the ID Token to your Resource Server API. It is meant for the Client only to learn about the user. The Access Token is what goes to APIs.

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
  "name": "Amith Krishnan",
  "picture": "https://lh3.googleusercontent.com/...",
  "given_name": "Amith",
  "family_name": "Krishnan",
  "locale": "en"
}
```

| Claim | Meaning | Required? |
|-------|---------|-----------|
| `iss` | Issuer — who issued this token | Yes |
| `sub` | Subject — unique user identifier at issuer | Yes |
| `aud` | Audience — which client this token is for | Yes |
| `exp` | Expiration time (Unix timestamp) | Yes |
| `iat` | Issued at (Unix timestamp) | Yes |
| `nonce` | Replay attack prevention | If sent in request |
| `email` | User's email | With `email` scope |
| `name` | User's full name | With `profile` scope |
| `picture` | Profile photo URL | With `profile` scope |

---

### 3.4 OIDC Scopes

| Scope | Claims returned |
|-------|----------------|
| `openid` | `sub` (required for OIDC) |
| `profile` | `name`, `given_name`, `family_name`, `picture`, `locale`, `birthdate` |
| `email` | `email`, `email_verified` |
| `address` | `address` (formatted, street, city, etc.) |
| `phone` | `phone_number`, `phone_number_verified` |
| `offline_access` | Requests a refresh token |

---

### 3.5 UserInfo Endpoint

After receiving an access token with `openid` scope, the client can call:

```http
GET https://accounts.google.com/oauth2/v3/userinfo
Authorization: Bearer <access_token>
```

Response:
```json
{
  "sub": "110169484474386276334",
  "name": "Amith Krishnan",
  "email": "amith@gmail.com",
  "email_verified": true,
  "picture": "https://..."
}
```

---

### 3.6 Discovery Document

Every OIDC provider exposes a well-known configuration endpoint:

```
GET https://accounts.google.com/.well-known/openid-configuration
GET https://your-keycloak/realms/myrealm/.well-known/openid-configuration
```

Response (partial):
```json
{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
  "scopes_supported": ["openid", "email", "profile"],
  "response_types_supported": ["code", "token", "id_token"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "id_token_signing_alg_values_supported": ["RS256"]
}
```

Spring Boot uses `issuer-uri` to auto-discover all endpoints from this document.

---

### 3.7 Nonce — Replay Attack Prevention

The `nonce` is a random value included in the authorization request and embedded in the ID Token.

```
Client generates: nonce = "abc123"
Sends in /authorize request
ID Token comes back with: "nonce": "abc123"
Client verifies nonce matches what it sent
```

**Why?** Prevents replay attacks where an attacker reuses a captured ID Token. Since the nonce is tied to a specific session, a replayed token with an old nonce would fail validation.

---

## 4. JWT Deep Dive

### 4.1 JWT Structure

A JWT consists of three Base64URL-encoded parts separated by dots:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkFtaXRoIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
      ^HEADER^                                  ^PAYLOAD^                                              ^SIGNATURE^
```

#### Header
```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "key-id-1"
}
```
- `alg`: Signing algorithm (RS256, HS256, ES256)
- `typ`: Token type (always "JWT")
- `kid`: Key ID — which key was used to sign (used to look up correct key from JWKS)

#### Payload (Claims)
```json
{
  "iss": "https://auth.example.com",
  "sub": "user-123",
  "aud": ["https://api.example.com"],
  "exp": 1735600000,
  "nbf": 1735596400,
  "iat": 1735596400,
  "jti": "unique-token-id-abc123",
  "scope": "read:profile write:data",
  "roles": ["USER", "ADMIN"],
  "email": "amith@example.com"
}
```

**Registered Claims (standardized):**

| Claim | Full Name | Description |
|-------|-----------|-------------|
| `iss` | Issuer | Who issued the token |
| `sub` | Subject | Who the token is about (user ID) |
| `aud` | Audience | Who the token is intended for |
| `exp` | Expiration | When the token expires (Unix time) |
| `nbf` | Not Before | Token not valid before this time |
| `iat` | Issued At | When the token was issued |
| `jti` | JWT ID | Unique identifier for this token |

**Public Claims:** Registered in IANA JWT registry (e.g., `email`, `name`)

**Private Claims:** Custom claims agreed upon between parties (e.g., `roles`, `tenant_id`)

#### Signature

For RS256:
```
signature = RSA_SIGN(
    private_key,
    SHA256(base64url(header) + "." + base64url(payload))
)
```

---

### 4.2 RS256 vs HS256

| Feature | HS256 (Symmetric) | RS256 (Asymmetric) |
|---------|-------------------|-------------------|
| Algorithm | HMAC-SHA256 | RSA with SHA-256 |
| Key type | Single shared secret | Private key (sign) + Public key (verify) |
| Who can verify | Only parties with the secret | Anyone with the public key |
| Key distribution | Secret must be shared | Public key freely distributed |
| Use case | Single service | Distributed/microservices |
| Risk if key leaked | Full compromise | Only signing compromised |

**Why RS256 is preferred in OAuth2:**

With RS256:
- The Authorization Server holds the **private key** and signs tokens
- Resource Servers hold the **public key** (fetched from JWKS endpoint) and verify tokens
- No shared secrets — Resource Servers never need the private key
- Adding a new microservice? Just point it to the JWKS endpoint. No secret sharing required.

With HS256:
- Every service that validates tokens needs the same shared secret
- Sharing a secret across many services is a security nightmare
- If one service is compromised, the secret is exposed

---

### 4.3 JWT Validation Steps

When a Resource Server receives a JWT, it MUST:

```java
// Validation order matters
1. Decode the JWT (verify it's well-formed)
2. Verify signature using the issuer's public key (from JWKS)
3. Check exp (expiration) — reject if expired
4. Check nbf (not before) — reject if too early
5. Check iss (issuer) — must match expected issuer
6. Check aud (audience) — must include this server's identifier
7. Check jti (optional) — for replay prevention with a blacklist
```

**Never skip audience validation.** If Token A is issued for Service A and an attacker replays it against Service B (which also accepts the same issuer), Service B should reject it because `aud` doesn't include Service B.

---

### 4.4 JWK Set (JWKS)

The JWKS endpoint serves the public keys used to verify JWT signatures.

```
GET https://auth.example.com/.well-known/jwks.json
```

Response:
```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "alg": "RS256",
      "kid": "key-id-1",
      "n": "0vx7agoebGcQSuuPiLJXZptN9nndrQmbXEps2...",
      "e": "AQAB"
    }
  ]
}
```

- `kty`: Key type (RSA, EC)
- `use`: Usage (`sig` for signature, `enc` for encryption)
- `kid`: Key ID — matches the `kid` in JWT header
- `n`, `e`: RSA public key components (modulus and exponent)

Spring Boot's `NimbusJwtDecoder` fetches and caches these keys automatically.

---

### 4.5 JWT Security Vulnerabilities

#### 1. Algorithm Confusion Attack (none algorithm)

Old JWT libraries accepted `"alg": "none"` — no signature required.

```json
// Attacker crafts:
{
  "alg": "none",
  "typ": "JWT"
}
// with any claims, no signature
```

**Mitigation:** Always explicitly specify allowed algorithms. Never accept `none`.

#### 2. RS256 → HS256 Confusion Attack

If a library accepts both RS256 and HS256, attacker signs a JWT with HS256 using the *public key* as the HMAC secret (since the public key is publicly known).

**Mitigation:** Pin the expected algorithm explicitly.

#### 3. `jku` / `x5u` Header Injection

Some libraries fetch the JWKS from the URL in the JWT header itself (`jku` claim). Attacker points to their own JWKS.

**Mitigation:** Never trust key URLs from the JWT itself. Only use configured/trusted JWKS URIs.

#### 4. JWT Leakage via Logs

JWTs in URLs (Implicit flow) get logged in server access logs.

**Mitigation:** Always put tokens in headers, never in URLs.

---

## 5. Spring Security OAuth2 Resource Server

### 5.1 Maven Dependency

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

---

### 5.2 Minimal Configuration

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://accounts.google.com
          # Spring auto-discovers jwks-uri from /.well-known/openid-configuration
```

Or with explicit JWKS URI:
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: https://accounts.google.com/.well-known/certs
```

---

### 5.3 SecurityConfig — Complete Example

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.convert.converter.Converter;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.web.SecurityFilterChain;

import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // enables @PreAuthorize, @PostAuthorize
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // Disable session creation — we're stateless (JWT-based)
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            // Disable CSRF — stateless API, no cookies for auth
            .csrf(csrf -> csrf.disable())
            // CORS configuration
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            // Authorization rules
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            // Configure as OAuth2 Resource Server with JWT
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            );

        return http.build();
    }

    /**
     * Converts JWT claims to Spring Security GrantedAuthorities.
     * Keycloak puts roles inside realm_access.roles claim.
     * Adjust the claim path to match your Auth Server.
     */
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(new KeycloakRoleConverter());
        return converter;
    }

    /**
     * Custom converter that reads roles from Keycloak's JWT structure.
     */
    static class KeycloakRoleConverter
            implements Converter<Jwt, Collection<GrantedAuthority>> {

        @Override
        public Collection<GrantedAuthority> convert(Jwt jwt) {
            // Keycloak stores roles in: realm_access.roles
            Map<String, Object> realmAccess =
                jwt.getClaimAsMap("realm_access");

            if (realmAccess == null || realmAccess.isEmpty()) {
                return List.of();
            }

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

### 5.4 Custom JWT Decoder with Additional Validation

```java
@Bean
public JwtDecoder jwtDecoder() {
    NimbusJwtDecoder jwtDecoder = NimbusJwtDecoder
        .withJwkSetUri("https://auth.example.com/.well-known/jwks.json")
        .build();

    // Add custom validators
    OAuth2TokenValidator<Jwt> withIssuer =
        JwtValidators.createDefaultWithIssuer("https://auth.example.com");

    // Validate audience claim
    OAuth2TokenValidator<Jwt> withAudience =
        new JwtClaimValidator<List<String>>(JwtClaimNames.AUD,
            aud -> aud != null && aud.contains("my-api-resource"));

    OAuth2TokenValidator<Jwt> validator =
        new DelegatingOAuth2TokenValidator<>(withIssuer, withAudience);

    jwtDecoder.setJwtValidator(validator);

    return jwtDecoder;
}
```

---

### 5.5 Accessing JWT Claims in a Controller

```java
@RestController
@RequestMapping("/api")
public class UserController {

    // Method 1: Using @AuthenticationPrincipal
    @GetMapping("/me")
    public Map<String, Object> getMe(
            @AuthenticationPrincipal Jwt jwt) {

        return Map.of(
            "userId", jwt.getSubject(),
            "email", jwt.getClaimAsString("email"),
            "roles", jwt.getClaimAsStringList("roles"),
            "issuedAt", jwt.getIssuedAt()
        );
    }

    // Method 2: Using SecurityContextHolder
    @GetMapping("/profile")
    public String getProfile() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        JwtAuthenticationToken jwtAuth = (JwtAuthenticationToken) auth;
        Jwt jwt = jwtAuth.getToken();

        return "Hello " + jwt.getClaimAsString("name");
    }

    // Method 3: Using Authentication parameter
    @GetMapping("/authorities")
    public Collection<? extends GrantedAuthority> getAuthorities(
            Authentication authentication) {
        return authentication.getAuthorities();
    }
}
```

---

### 5.6 Opaque Token (Introspection) Alternative

When you have opaque tokens (not JWT), configure introspection:

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          introspection-uri: https://auth.example.com/oauth2/introspect
          client-id: my-resource-server
          client-secret: my-secret
```

```java
// SecurityConfig.java
.oauth2ResourceServer(oauth2 -> oauth2
    .opaqueToken(opaque -> opaque
        .introspectionUri("https://auth.example.com/oauth2/introspect")
        .introspectionClientCredentials("my-resource-server", "my-secret")
    )
)
```

Introspection sends a request to the auth server for *every* API request — slower but allows instant revocation.

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
          # Google
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"

          # GitHub
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: read:user, user:email

          # Keycloak
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
          # Google and GitHub are built-in providers, no need to configure
```

---

### 6.3 OAuth2 Login Security Config

```java
@Configuration
@EnableWebSecurity
public class OAuth2LoginSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login", "/error").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")                    // custom login page
                .defaultSuccessUrl("/dashboard", true)  // redirect after login
                .failureUrl("/login?error=true")
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService()) // customize user loading
                )
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/")
                .clearAuthentication(true)
                .invalidateHttpSession(true)
            );

        return http.build();
    }
}
```

---

### 6.4 Accessing OAuth2 User in Controller

```java
@Controller
public class DashboardController {

    @GetMapping("/dashboard")
    public String dashboard(
            @AuthenticationPrincipal OidcUser oidcUser,  // for OIDC providers
            Model model) {

        model.addAttribute("name", oidcUser.getFullName());
        model.addAttribute("email", oidcUser.getEmail());
        model.addAttribute("picture", oidcUser.getPicture());
        model.addAttribute("subject", oidcUser.getSubject());

        return "dashboard";
    }

    @GetMapping("/github-user")
    public String githubUser(
            @AuthenticationPrincipal OAuth2User oauth2User,  // for non-OIDC
            Model model) {

        model.addAttribute("login", oauth2User.getAttribute("login"));
        model.addAttribute("avatar", oauth2User.getAttribute("avatar_url"));

        return "profile";
    }
}
```

---

### 6.5 Making Outbound API Calls with OAuth2 Token

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient(OAuth2AuthorizedClientManager authorizedClientManager) {
        ServletOAuth2AuthorizedClientExchangeFilterFunction oauth2 =
            new ServletOAuth2AuthorizedClientExchangeFilterFunction(authorizedClientManager);
        oauth2.setDefaultClientRegistrationId("service-client");

        return WebClient.builder()
            .apply(oauth2.oauth2Configuration())
            .build();
    }

    @Bean
    public OAuth2AuthorizedClientManager authorizedClientManager(
            ClientRegistrationRepository clientRegistrationRepository,
            OAuth2AuthorizedClientRepository authorizedClientRepository) {

        OAuth2AuthorizedClientProvider authorizedClientProvider =
            OAuth2AuthorizedClientProviderBuilder.builder()
                .authorizationCode()
                .refreshToken()
                .clientCredentials()
                .build();

        DefaultOAuth2AuthorizedClientManager manager =
            new DefaultOAuth2AuthorizedClientManager(
                clientRegistrationRepository, authorizedClientRepository);
        manager.setAuthorizedClientProvider(authorizedClientProvider);

        return manager;
    }
}
```

Usage in service:
```java
@Service
public class DataService {

    private final WebClient webClient;

    public DataService(WebClient webClient) {
        this.webClient = webClient;
    }

    public Mono<String> fetchProtectedData() {
        return webClient
            .get()
            .uri("https://api.example.com/data")
            .attributes(clientRegistrationId("service-client"))
            .retrieve()
            .bodyToMono(String.class);
    }
}
```

---

## 7. Social Login: Google & GitHub

### 7.1 Complete Google OAuth2 Login Example

**Step 1: Google Cloud Console Setup**
1. Go to https://console.cloud.google.com
2. Create a project → Enable "Google+ API" or "People API"
3. Credentials → Create OAuth 2.0 Client ID
4. Application type: Web application
5. Authorized redirect URIs: `http://localhost:8080/login/oauth2/code/google`
6. Copy Client ID and Client Secret

**Step 2: application.yml**
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: 123456789-abcdefgh.apps.googleusercontent.com
            client-secret: GOCSPX-xxxxxxxx
            scope: openid, profile, email
```

**Step 3: SecurityConfig**
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(Customizer.withDefaults()); // uses default login page

        return http.build();
    }
}
```

Spring Boot auto-generates a login page at `/login` with a "Sign in with Google" button.

---

### 7.2 Custom OAuth2UserService — Persisting Users

```java
@Service
public class CustomOAuth2UserService extends DefaultOAuth2UserService {

    private final UserRepository userRepository;

    public CustomOAuth2UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        // Load user from OAuth2 provider (calls UserInfo endpoint)
        OAuth2User oAuth2User = super.loadUser(userRequest);

        String provider = userRequest.getClientRegistration().getRegistrationId(); // "google", "github"
        String providerId = oAuth2User.getAttribute("sub");   // Google uses "sub"
        // GitHub uses "id": providerId = oAuth2User.getAttribute("id").toString();
        String email = oAuth2User.getAttribute("email");
        String name = oAuth2User.getAttribute("name");

        // Find or create user in our database
        User user = userRepository
            .findByProviderAndProviderId(provider, providerId)
            .orElseGet(() -> {
                User newUser = new User();
                newUser.setProvider(provider);
                newUser.setProviderId(providerId);
                newUser.setEmail(email);
                newUser.setName(name);
                newUser.setRole("ROLE_USER");
                return userRepository.save(newUser);
            });

        // Update name/email in case they changed
        user.setEmail(email);
        user.setName(name);
        userRepository.save(user);

        return oAuth2User;
    }
}
```

Register in SecurityConfig:
```java
.oauth2Login(oauth2 -> oauth2
    .userInfoEndpoint(userInfo -> userInfo
        .userService(customOAuth2UserService)
    )
)
```

---

### 7.3 Custom OidcUserService (for OIDC providers like Google)

```java
@Service
public class CustomOidcUserService extends OidcUserService {

    private final UserRepository userRepository;

    @Override
    public OidcUser loadUser(OidcUserRequest userRequest) throws OAuth2AuthenticationException {
        OidcUser oidcUser = super.loadUser(userRequest);

        String email = oidcUser.getEmail();
        String googleId = oidcUser.getSubject();

        // Merge with local user database
        User user = userRepository.findByEmail(email)
            .orElseGet(() -> createNewUser(oidcUser));

        // Build custom OidcUser with our application roles
        Set<GrantedAuthority> authorities = new HashSet<>(oidcUser.getAuthorities());
        authorities.add(new SimpleGrantedAuthority(user.getRole()));

        return new DefaultOidcUser(authorities, oidcUser.getIdToken(), oidcUser.getUserInfo());
    }

    private User createNewUser(OidcUser oidcUser) {
        User user = new User();
        user.setEmail(oidcUser.getEmail());
        user.setName(oidcUser.getFullName());
        user.setRole("ROLE_USER");
        return userRepository.save(user);
    }
}
```

---

### 7.4 GitHub OAuth App Setup

1. GitHub → Settings → Developer settings → OAuth Apps → New OAuth App
2. Application name: My App
3. Homepage URL: `http://localhost:8080`
4. Authorization callback URL: `http://localhost:8080/login/oauth2/code/github`
5. Copy Client ID and Client Secret

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: read:user, user:email
```

Note: GitHub is NOT an OIDC provider, so use `OAuth2User` (not `OidcUser`) and `DefaultOAuth2UserService`.

---

## 8. Spring Authorization Server

### 8.1 What is Spring Authorization Server?

Spring Authorization Server (SAS) is the official, first-party OAuth2/OIDC Authorization Server from the Spring team. It replaces the deprecated Spring Security OAuth2 Authorization Server.

**Use cases:**
- Build your own OAuth2/OIDC authorization server
- Centralized auth for your microservices ecosystem
- When you don't want to depend on Keycloak or a cloud provider

**Maven dependency:**
```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-oauth2-authorization-server</artifactId>
</dependency>
```

---

### 8.2 Minimal Spring Authorization Server Setup

```java
@Configuration
@EnableWebSecurity
public class AuthServerConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain authorizationServerSecurityFilterChain(HttpSecurity http)
            throws Exception {

        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);

        http.getConfigurer(OAuth2AuthorizationServerConfigurer.class)
            .oidc(Customizer.withDefaults()); // enable OIDC support

        http.exceptionHandling(exceptions -> exceptions
            .defaultAuthenticationEntryPointFor(
                new LoginUrlAuthenticationEntryPoint("/login"),
                new MediaTypeRequestMatcher(MediaType.TEXT_HTML)
            )
        );

        return http.build();
    }

    @Bean
    @Order(2)
    public SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .formLogin(Customizer.withDefaults());

        return http.build();
    }

    @Bean
    public RegisteredClientRepository registeredClientRepository() {
        RegisteredClient webClient = RegisteredClient
            .withId(UUID.randomUUID().toString())
            .clientId("my-web-app")
            .clientSecret("{bcrypt}" + new BCryptPasswordEncoder().encode("secret"))
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .redirectUri("http://localhost:3000/callback")
            .scope(OidcScopes.OPENID)
            .scope(OidcScopes.PROFILE)
            .scope(OidcScopes.EMAIL)
            .clientSettings(ClientSettings.builder()
                .requireAuthorizationConsent(true)
                .requireProofKey(true)  // enforce PKCE
                .build())
            .tokenSettings(TokenSettings.builder()
                .accessTokenTimeToLive(Duration.ofMinutes(15))
                .refreshTokenTimeToLive(Duration.ofDays(7))
                .reuseRefreshTokens(false) // refresh token rotation
                .build())
            .build();

        RegisteredClient serviceClient = RegisteredClient
            .withId(UUID.randomUUID().toString())
            .clientId("order-service")
            .clientSecret("{bcrypt}" + new BCryptPasswordEncoder().encode("order-secret"))
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
            .scope("read:orders")
            .scope("write:orders")
            .build();

        return new InMemoryRegisteredClientRepository(webClient, serviceClient);
    }

    @Bean
    public JWKSource<SecurityContext> jwkSource() {
        RSAKey rsaKey = generateRsa();
        JWKSet jwkSet = new JWKSet(rsaKey);
        return new ImmutableJWKSet<>(jwkSet);
    }

    private RSAKey generateRsa() {
        try {
            KeyPairGenerator gen = KeyPairGenerator.getInstance("RSA");
            gen.initialize(2048);
            KeyPair keyPair = gen.generateKeyPair();
            return new RSAKey.Builder((RSAPublicKey) keyPair.getPublic())
                .privateKey(keyPair.getPrivate())
                .keyID(UUID.randomUUID().toString())
                .build();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    @Bean
    public AuthorizationServerSettings authorizationServerSettings() {
        return AuthorizationServerSettings.builder()
            .issuer("http://localhost:9000")
            .build();
    }
}
```

---

### 8.3 Custom Token Claims

```java
@Bean
public OAuth2TokenCustomizer<JwtEncodingContext> tokenCustomizer(UserRepository userRepository) {
    return context -> {
        if (OAuth2TokenType.ACCESS_TOKEN.equals(context.getTokenType())) {
            Authentication principal = context.getPrincipal();

            // Add custom claims to access token
            Set<String> roles = principal.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toSet());

            context.getClaims()
                .claim("roles", roles)
                .claim("tenant_id", "tenant-abc")
                .claim("user_id", getUserId(principal));
        }
    };
}
```

---

### 8.4 Keycloak vs Spring Authorization Server

| Feature | Keycloak | Spring Authorization Server |
|---------|---------|---------------------------|
| Setup | Separate server, Docker/K8s | Embedded in Spring Boot app |
| Features | Rich (users, groups, MFA, social login, admin UI) | Basic OAuth2/OIDC |
| Learning curve | Higher (Keycloak concepts) | Lower (Spring concepts) |
| Customization | Via SPIs, complex | Full Java code control |
| Production readiness | Battle-tested | Newer, growing |
| Use when | Enterprise, complex user management | Greenfield, full control, microservices |

---

## 9. Method-Level Security

### 9.1 Enable Method Security

```java
@Configuration
@EnableMethodSecurity  // replaces deprecated @EnableGlobalMethodSecurity
public class MethodSecurityConfig {
    // That's all you need
}
```

`@EnableMethodSecurity` enables:
- `@PreAuthorize` and `@PostAuthorize`
- `@PreFilter` and `@PostFilter`
- `@Secured`
- `@RolesAllowed` (JSR-250, enabled via `jsr250Enabled = true`)

---

### 9.2 @PreAuthorize

Evaluated **before** the method executes. If false, method is not called.

```java
@Service
public class UserService {

    // Require ADMIN role
    @PreAuthorize("hasRole('ADMIN')")
    public List<User> getAllUsers() {
        return userRepository.findAll();
    }

    // Require a specific authority (permission string)
    @PreAuthorize("hasAuthority('READ_USER')")
    public User getUserById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }

    // SpEL with method parameter — user can only access their own data
    @PreAuthorize("#userId == authentication.principal.id")
    public UserProfile getProfile(Long userId) {
        return profileRepository.findByUserId(userId);
    }

    // Multiple conditions
    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
    public void updateUser(Long userId, UserUpdateRequest request) {
        // ADMIN can update any user, users can update themselves
    }

    // Check custom bean
    @PreAuthorize("@securityService.canAccessDocument(authentication, #docId)")
    public Document getDocument(Long docId) {
        return documentRepository.findById(docId).orElseThrow();
    }

    // Check collection parameter
    @PreAuthorize("hasRole('MANAGER') and #request.amount <= 10000")
    public void approvePurchase(PurchaseRequest request) {
        // MANAGER can approve, but only up to 10,000
    }
}
```

Custom security bean referenced via `@`:
```java
@Component("securityService")
public class SecurityService {

    private final DocumentRepository documentRepository;

    public boolean canAccessDocument(Authentication auth, Long docId) {
        Document doc = documentRepository.findById(docId).orElseThrow();
        String username = auth.getName();
        return doc.getOwner().equals(username) ||
               auth.getAuthorities().stream()
                   .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
    }
}
```

---

### 9.3 @PostAuthorize

Evaluated **after** the method executes, with access to the return value via `returnObject`.

```java
// User can only get a resource if they own it
// Method runs, but result is withheld if condition is false
@PostAuthorize("returnObject.ownerId == authentication.principal.id or hasRole('ADMIN')")
public Order getOrder(Long orderId) {
    return orderRepository.findById(orderId).orElseThrow();
}
```

**Warning:** `@PostAuthorize` still executes the method. Side effects (DB writes) happen before the check. Use `@PreAuthorize` when possible to avoid unnecessary method execution.

---

### 9.4 @PreFilter and @PostFilter

Filter collections passed to or returned from a method.

```java
// Filter input list — only process items owned by the current user
@PreFilter("filterObject.ownerId == authentication.principal.id")
public void processOrders(List<Order> orders) {
    // 'orders' list is filtered before this method sees it
    orders.forEach(this::process);
}

// Filter output list — return only items user can see
@PostFilter("filterObject.status == 'PUBLIC' or filterObject.ownerId == authentication.principal.id")
public List<Document> getAllDocuments() {
    return documentRepository.findAll(); // full list fetched, then filtered
}
```

**Note:** `filterObject` refers to each element in the collection. `@PostFilter` is inefficient on large datasets — better to filter at the query level.

---

### 9.5 @Secured

Simpler than `@PreAuthorize` — role check only, no SpEL.

```java
@Secured("ROLE_ADMIN")
public void deleteUser(Long id) {
    userRepository.deleteById(id);
}

@Secured({"ROLE_USER", "ROLE_ADMIN"})
public List<Product> getProducts() {
    return productRepository.findAll();
}
```

---

### 9.6 @RolesAllowed (JSR-250)

Standard Java annotation, same as `@Secured`:

```java
@EnableMethodSecurity(jsr250Enabled = true)
public class MethodSecurityConfig {}

// Usage:
@RolesAllowed("ADMIN")
public void sensitiveOperation() {}

@RolesAllowed({"USER", "ADMIN"})
public List<Item> listItems() { return List.of(); }
```

---

### 9.7 Comparison: @PreAuthorize vs @Secured vs @RolesAllowed

| Feature | @PreAuthorize | @Secured | @RolesAllowed |
|---------|--------------|----------|---------------|
| SpEL support | Yes (full) | No | No |
| Multiple roles | Via SpEL | Via array | Via array |
| Method params | Yes (#param) | No | No |
| Return value check | No (use @Post) | No | No |
| Standard | Spring | Spring | JSR-250 |
| Recommendation | Preferred | Legacy | Use for portability |

---

### 9.8 Role Hierarchy

Without hierarchy: `ADMIN` does NOT automatically have `USER` permissions.
With hierarchy: define inheritance explicitly.

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
public MethodSecurityExpressionHandler methodSecurityExpressionHandler(
        RoleHierarchy roleHierarchy) {
    DefaultMethodSecurityExpressionHandler handler =
        new DefaultMethodSecurityExpressionHandler();
    handler.setRoleHierarchy(roleHierarchy);
    return handler;
}
```

Now `@PreAuthorize("hasRole('USER')")` is also satisfied by `ADMIN` and `MANAGER`.

---

## 10. CORS and CSRF

### 10.1 What is CORS?

**CORS** = Cross-Origin Resource Sharing

**Same-Origin Policy:** Browsers block JavaScript from making requests to a different origin (domain, protocol, or port) than the page was loaded from. This is a *browser* security feature — not a server feature.

```
Frontend at: https://frontend.example.com
Backend API at: https://api.example.com

Without CORS config → browser BLOCKS the request
With CORS config → browser ALLOWS it (after server says so)
```

CORS only applies to browser-based requests. Curl, Postman, server-to-server calls — no CORS restriction.

**Preflight Request:**
For non-simple requests (PUT, DELETE, custom headers), the browser first sends an `OPTIONS` request:
```http
OPTIONS /api/users HTTP/1.1
Origin: https://frontend.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Content-Type, Authorization
```

Server responds (if allowed):
```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://frontend.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 3600
```

---

### 10.2 CORS Configuration in Spring Boot

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();

        // Specific origins (NEVER use "*" with credentials)
        config.setAllowedOrigins(List.of(
            "https://frontend.example.com",
            "http://localhost:3000"   // for local development
        ));

        // Or use patterns:
        // config.setAllowedOriginPatterns(List.of("https://*.example.com"));

        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Requested-With"));
        config.setExposedHeaders(List.of("X-Total-Count")); // headers frontend can read
        config.setAllowCredentials(true); // needed for cookies
        config.setMaxAge(3600L); // cache preflight for 1 hour

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);

        return source;
    }
}
```

Register in security config:
```java
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
```

Or use `@CrossOrigin` on a controller (for simple cases):
```java
@CrossOrigin(origins = "http://localhost:3000", maxAge = 3600)
@RestController
@RequestMapping("/api")
public class ApiController {}
```

---

### 10.3 CORS with JWT (Stateless APIs)

For REST APIs using JWT (Authorization header), CORS is simpler:
- No cookies → `allowCredentials` can be `false`
- `allowedHeaders` must include `Authorization`

```java
config.setAllowedOrigins(List.of("*"));  // OK without credentials
config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
config.setAllowCredentials(false);  // no cookies
```

---

### 10.4 What is CSRF?

**CSRF** = Cross-Site Request Forgery

**Attack scenario:**
1. User is logged into `bank.com` (has session cookie)
2. User visits malicious `evil.com`
3. `evil.com` has hidden form: `<form action="bank.com/transfer" method="POST">`
4. Browser automatically includes the `bank.com` session cookie
5. Bank server thinks the request is from the user → money transferred!

**Why this works:** Browsers automatically send cookies for a domain with every request to that domain, regardless of which page initiated it.

---

### 10.5 Spring Security CSRF Protection

Spring Security uses the **Synchronizer Token Pattern**:
1. Server generates a random CSRF token per session
2. Token is embedded in every HTML form (hidden field)
3. Every state-changing request (POST, PUT, DELETE) must include the token
4. Server validates the token — evil.com can't read it (same-origin policy)

```html
<!-- Thymeleaf auto-includes CSRF token -->
<form th:action="@{/transfer}" method="post">
    <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}"/>
    <!-- fields -->
</form>
```

---

### 10.6 When to Disable CSRF

```java
// DISABLE CSRF for stateless REST APIs using JWT in Authorization header
http.csrf(csrf -> csrf.disable());
```

**Why JWT APIs don't need CSRF:**
- CSRF exploits *automatic* cookie sending
- JWT is sent via `Authorization: Bearer <token>` header — browsers CANNOT set custom headers cross-origin without explicit CORS permission
- `evil.com` cannot set the `Authorization` header from cross-origin JavaScript
- Therefore, no CSRF risk for header-based authentication

**When to KEEP CSRF enabled:**
- Traditional web apps using session cookies for authentication
- If you're using cookies to store JWT (even HttpOnly cookies)

---

### 10.7 SameSite Cookies as CSRF Defense

Modern alternative to CSRF tokens:

```java
// Configure session cookie with SameSite=Strict
server:
  servlet:
    session:
      cookie:
        same-site: strict  # or "lax"
        http-only: true
        secure: true
```

- `SameSite=Strict`: cookie not sent on any cross-origin request
- `SameSite=Lax`: cookie sent on top-level navigation GET requests only (safe for CSRF)
- Most modern browsers support this

---

## 11. Refresh Token Flow

### 11.1 Why Refresh Tokens?

Short-lived access tokens (15 minutes) minimize damage if stolen, but re-authenticating every 15 minutes would ruin UX. Refresh tokens solve this:

```
Access Token: 15 min TTL → used to call APIs
Refresh Token: 7 day TTL → used only to get new access tokens
```

### 11.2 Refresh Token Request

```http
POST https://auth.example.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
&client_id=my-client-id
&client_secret=my-client-secret
```

Response:
```json
{
  "access_token": "new_access_token_here",
  "token_type": "Bearer",
  "expires_in": 900,
  "refresh_token": "new_refresh_token_here"
}
```

---

### 11.3 Refresh Token Rotation

Every time you use a refresh token, you get a **new refresh token** (the old one is invalidated).

**Benefits:**
- If a refresh token is stolen and used by an attacker, the next use by the legitimate user (or vice versa) invalidates the token — both get logged out
- Allows detection of refresh token theft

Spring Authorization Server configuration:
```java
.tokenSettings(TokenSettings.builder()
    .reuseRefreshTokens(false)  // enable rotation
    .build())
```

---

### 11.4 Refresh Token Storage

| Storage | XSS Risk | CSRF Risk | Notes |
|---------|---------|-----------|-------|
| `localStorage` | HIGH (JS accessible) | Low | Never use for sensitive tokens |
| Memory (JS variable) | Low | Low | Lost on page refresh |
| HttpOnly cookie | None (JS inaccessible) | HIGH (CSRF) | Best: HttpOnly + SameSite + Secure |
| HttpOnly + SameSite=Strict | None | None | Recommended for web apps |

**Best practice for web apps:**
```
Access token: in memory (JavaScript variable)
Refresh token: HttpOnly, Secure, SameSite=Strict cookie
```

This way:
- XSS can't steal the refresh token (HttpOnly)
- CSRF can't use the cookie (SameSite=Strict)
- Access token in memory is lost on refresh — use refresh token to get a new one

---

### 11.5 Spring Boot Refresh Token Implementation

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final RefreshTokenService refreshTokenService;
    private final JwtService jwtService;
    private final UserRepository userRepository;

    @PostMapping("/refresh")
    public ResponseEntity<TokenResponse> refreshToken(
            @CookieValue(name = "refresh_token", required = false) String refreshToken,
            HttpServletResponse response) {

        if (refreshToken == null) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }

        // Validate refresh token
        RefreshToken storedToken = refreshTokenService.findByToken(refreshToken)
            .orElseThrow(() -> new RuntimeException("Invalid refresh token"));

        if (storedToken.isExpired()) {
            refreshTokenService.delete(storedToken);
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .body(new TokenResponse("Refresh token expired, please login again"));
        }

        // Rotate: delete old, create new
        refreshTokenService.delete(storedToken);
        String newRefreshToken = refreshTokenService.createRefreshToken(storedToken.getUser());

        // Set new refresh token as HttpOnly cookie
        Cookie cookie = new Cookie("refresh_token", newRefreshToken);
        cookie.setHttpOnly(true);
        cookie.setSecure(true);  // HTTPS only
        cookie.setPath("/api/auth/refresh");
        cookie.setMaxAge(7 * 24 * 60 * 60); // 7 days
        response.addCookie(cookie);

        // Generate new access token
        String newAccessToken = jwtService.generateToken(storedToken.getUser());

        return ResponseEntity.ok(new TokenResponse(newAccessToken));
    }
}
```

---

### 11.6 Frontend Token Handling (React)

```javascript
// Axios interceptor for automatic token refresh
import axios from 'axios';

let accessToken = null; // stored in memory

const api = axios.create({ baseURL: 'https://api.example.com' });

// Attach access token to every request
api.interceptors.request.use(config => {
    if (accessToken) {
        config.headers.Authorization = `Bearer ${accessToken}`;
    }
    return config;
});

// Auto-refresh on 401
api.interceptors.response.use(
    response => response,
    async error => {
        const originalRequest = error.config;

        if (error.response?.status === 401 && !originalRequest._retry) {
            originalRequest._retry = true;

            try {
                // Refresh token is in HttpOnly cookie — sent automatically
                const response = await axios.post('/api/auth/refresh',
                    {}, { withCredentials: true });

                accessToken = response.data.accessToken;
                originalRequest.headers.Authorization = `Bearer ${accessToken}`;

                return api(originalRequest); // retry original request
            } catch (refreshError) {
                accessToken = null;
                window.location.href = '/login';
                return Promise.reject(refreshError);
            }
        }
        return Promise.reject(error);
    }
);
```

---

## 12. Security Best Practices

### 12.1 Token Storage

```
DO:
  - Access tokens: memory (JavaScript variable)
  - Refresh tokens: HttpOnly, Secure, SameSite=Strict cookies
  - Never store in localStorage or sessionStorage

DON'T:
  - Never store tokens in localStorage (XSS risk)
  - Never put tokens in URL query params (log exposure)
  - Never log full tokens (even partial is risky)
```

**XSS + localStorage = game over:**
Any XSS vulnerability allows `document.cookie` to be blocked (HttpOnly), but `localStorage.getItem('token')` is always accessible from JavaScript.

---

### 12.2 Token Lifetimes

```
Access Token:  5–15 minutes (short, limits exposure window)
Refresh Token: 1–7 days (depends on sensitivity)
ID Token:      Same as access token (used only for session establishment)
```

---

### 12.3 Audience Validation

Always validate the `aud` claim to prevent token substitution attacks:

```java
// Without audience validation:
// Token issued for Service A can be replayed against Service B

// With audience validation:
OAuth2TokenValidator<Jwt> audienceValidator =
    new JwtClaimValidator<List<String>>(JwtClaimNames.AUD,
        aud -> aud.contains("my-api"));
```

---

### 12.4 Token Revocation Strategies

| Strategy | How | Tradeoff |
|---------|-----|---------|
| Short TTL | Access tokens expire in 5-15 min | No instant revocation, but low risk window |
| Introspection | Call auth server on every request | Instant revocation, but adds latency |
| Redis blacklist | Store revoked JTIs in Redis, check on each request | Near-instant, needs Redis |
| Refresh token rotation | Revoke refresh token → user re-authenticates | Next refresh attempt fails |

Redis blacklist example:
```java
@Component
public class JwtBlacklist {

    private final RedisTemplate<String, String> redisTemplate;

    public void blacklist(String jti, Duration ttl) {
        redisTemplate.opsForValue().set("blacklist:" + jti, "revoked", ttl);
    }

    public boolean isBlacklisted(String jti) {
        return Boolean.TRUE.equals(redisTemplate.hasKey("blacklist:" + jti));
    }
}
```

---

### 12.5 State Parameter

Always use `state` in authorization requests to prevent CSRF on the OAuth2 flow:

```java
// Spring Security handles this automatically
// Manual implementation:
String state = UUID.randomUUID().toString();
session.setAttribute("oauth2_state", state);

// On callback, verify:
if (!state.equals(request.getParameter("state"))) {
    throw new SecurityException("State mismatch — possible CSRF attack");
}
```

---

### 12.6 PKCE for All Public Clients

Any client that cannot keep a secret (SPAs, mobile apps, CLI tools) MUST use PKCE:
- No exceptions
- `code_challenge_method=S256` only (never `plain`)

---

### 12.7 Rate Limiting Token Endpoints

```java
// Using Bucket4j for rate limiting
@Component
public class RateLimitingFilter extends OncePerRequestFilter {

    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        if (request.getRequestURI().startsWith("/oauth2/token")) {
            String clientIp = request.getRemoteAddr();
            Bucket bucket = buckets.computeIfAbsent(clientIp,
                ip -> Bucket.builder()
                    .addLimit(Bandwidth.classic(10, Refill.greedy(10, Duration.ofMinutes(1))))
                    .build());

            if (!bucket.tryConsume(1)) {
                response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
                return;
            }
        }
        chain.doFilter(request, response);
    }
}
```

---

### 12.8 Secure Headers

```java
http.headers(headers -> headers
    .frameOptions(frame -> frame.deny())
    .xssProtection(xss -> xss.enable())
    .contentSecurityPolicy(csp -> csp
        .policyDirectives("default-src 'self'; script-src 'self'"))
    .httpStrictTransportSecurity(hsts -> hsts
        .includeSubDomains(true)
        .maxAgeInSeconds(31536000))
);
```

---

## 13. Keycloak Integration

### 13.1 Core Concepts

| Concept | Description |
|---------|-------------|
| **Realm** | Isolated tenant — separate users, clients, settings |
| **Client** | Your application registered in Keycloak |
| **User** | An end user in a realm |
| **Role** | Permission label (realm role or client role) |
| **Group** | Collection of users |
| **Identity Provider** | External auth (Google, LDAP, SAML) fed into Keycloak |

---

### 13.2 Docker Setup

```yaml
# docker-compose.yml
version: '3.8'
services:
  keycloak:
    image: quay.io/keycloak/keycloak:23.0.0
    command: start-dev
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports:
      - "8080:8080"
```

Access: `http://localhost:8080` → Admin console → Create realm `myrealm` → Create client `my-spring-app`

---

### 13.3 Spring Boot + Keycloak Resource Server

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/myrealm
          # Spring auto-discovers:
          # jwks-uri: http://localhost:8080/realms/myrealm/protocol/openid-connect/certs
```

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(keycloakConverter()))
            );
        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter keycloakConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            // Keycloak realm roles
            Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
            List<String> realmRoles = realmAccess != null
                ? (List<String>) realmAccess.get("roles")
                : List.of();

            // Keycloak client roles
            Map<String, Object> resourceAccess = jwt.getClaimAsMap("resource_access");
            List<String> clientRoles = List.of();
            if (resourceAccess != null) {
                Map<String, Object> clientAccess =
                    (Map<String, Object>) resourceAccess.get("my-spring-app");
                if (clientAccess != null) {
                    clientRoles = (List<String>) clientAccess.get("roles");
                }
            }

            List<GrantedAuthority> authorities = new ArrayList<>();
            realmRoles.stream()
                .map(r -> new SimpleGrantedAuthority("ROLE_" + r.toUpperCase()))
                .forEach(authorities::add);
            clientRoles.stream()
                .map(r -> new SimpleGrantedAuthority("ROLE_" + r.toUpperCase()))
                .forEach(authorities::add);

            return authorities;
        });
        return converter;
    }
}
```

---

### 13.4 Keycloak JWT Structure

```json
{
  "exp": 1735600000,
  "iat": 1735596400,
  "jti": "unique-id",
  "iss": "http://localhost:8080/realms/myrealm",
  "aud": ["my-spring-app", "account"],
  "sub": "user-uuid-here",
  "typ": "Bearer",
  "azp": "my-spring-app",
  "session_state": "session-uuid",
  "realm_access": {
    "roles": ["offline_access", "uma_authorization", "USER"]
  },
  "resource_access": {
    "my-spring-app": {
      "roles": ["MANAGER"]
    },
    "account": {
      "roles": ["manage-account"]
    }
  },
  "scope": "openid email profile",
  "email_verified": true,
  "name": "Amith Krishnan",
  "preferred_username": "amith",
  "email": "amith@example.com"
}
```

---

## 14. Microservice Security Patterns

### 14.1 Token Propagation

When Service A calls Service B, it should forward the user's JWT:

```java
@Configuration
public class FeignClientConfig {

    @Bean
    public RequestInterceptor tokenPropagationInterceptor() {
        return requestTemplate -> {
            // Get current authentication's JWT and forward it
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth instanceof JwtAuthenticationToken jwtAuth) {
                String tokenValue = jwtAuth.getToken().getTokenValue();
                requestTemplate.header("Authorization", "Bearer " + tokenValue);
            }
        };
    }
}
```

With WebClient:
```java
@Bean
public WebClient userContextWebClient() {
    return WebClient.builder()
        .filter((request, next) -> {
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth instanceof JwtAuthenticationToken jwtAuth) {
                return next.exchange(
                    ClientRequest.from(request)
                        .header("Authorization", "Bearer " + jwtAuth.getToken().getTokenValue())
                        .build()
                );
            }
            return next.exchange(request);
        })
        .build();
}
```

---

### 14.2 Service-to-Service (Client Credentials)

For direct service-to-service calls not on behalf of a user:

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          order-service:
            client-id: order-service
            client-secret: ${ORDER_SERVICE_SECRET}
            authorization-grant-type: client_credentials
            scope: read:inventory
        provider:
          order-service:
            token-uri: http://auth-server/oauth2/token
```

```java
@Service
public class InventoryClient {

    private final WebClient webClient;

    public InventoryClient(WebClient.Builder builder,
            OAuth2AuthorizedClientManager manager) {

        ServletOAuth2AuthorizedClientExchangeFilterFunction oauth2 =
            new ServletOAuth2AuthorizedClientExchangeFilterFunction(manager);
        oauth2.setDefaultClientRegistrationId("order-service");

        this.webClient = builder
            .baseUrl("http://inventory-service")
            .apply(oauth2.oauth2Configuration())
            .build();
    }

    public Inventory getInventory(String productId) {
        return webClient.get()
            .uri("/api/inventory/{id}", productId)
            .retrieve()
            .bodyToMono(Inventory.class)
            .block();
    }
}
```

---

### 14.3 API Gateway Pattern

```
                    ┌─────────────────────────────────────────┐
                    │              API Gateway                 │
  Client ──JWT────> │  1. Validate JWT (signature, exp, aud)  │
                    │  2. Extract user info from claims        │
                    │  3. Pass X-User-Id, X-User-Roles headers│
                    └───────────────┬─────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
              User Service    Order Service   Payment Service
             (trusts gateway) (trusts gateway) (trusts gateway)
```

In downstream services:
```java
// Option 1: Trust X-User headers from gateway (internal network only)
@GetMapping("/orders")
public List<Order> getOrders(
        @RequestHeader("X-User-Id") String userId,
        @RequestHeader("X-User-Roles") String roles) {
    return orderService.getOrdersForUser(userId);
}

// Option 2: Still validate JWT in each service (Zero Trust)
// Every service validates the token independently
// No service trusts headers blindly
```

---

### 14.4 Zero Trust Security

```
Principle: Never trust, always verify
Every service:
  1. Validates the JWT signature (using JWKS)
  2. Checks token expiration
  3. Validates issuer (expected auth server)
  4. Validates audience (its own service identifier)
  5. Applies authorization rules
```

This ensures that even if an internal service is compromised, it cannot impersonate users to other services.

---

## 15. Interview Questions & Answers (40+)

---

### Q1: What is the difference between OAuth2 and OIDC?

**Answer:**
OAuth2 is an **authorization** framework that allows third-party applications to obtain limited access to a user's resources on a service. It answers the question "What can this app do on your behalf?"

OIDC (OpenID Connect) is an **identity layer** built on top of OAuth2. It adds authentication — answering "Who is this user?" OIDC introduces the ID Token (a JWT containing user identity claims), the UserInfo endpoint, and standardized scopes like `openid`, `profile`, and `email`.

In short: **OAuth2 = authorization only. OIDC = OAuth2 + authentication.**

---

### Q2: What is the difference between authentication and authorization?

**Answer:**
- **Authentication** = Verifying who you are. "Are you really Amith?" — Proven via credentials (password, biometric, OTP).
- **Authorization** = Determining what you're allowed to do. "Amith can read, but cannot delete." — Checked after authentication.

You cannot authorize without first authenticating. Authentication establishes identity; authorization checks permissions for that identity.

---

### Q3: Walk me through the Authorization Code Flow step by step.

**Answer:**
1. User clicks "Login" — app redirects to Authorization Server's `/authorize` with `client_id`, `redirect_uri`, `scope`, `state`, `response_type=code`
2. User logs in at the Authorization Server and grants consent
3. Auth Server redirects back to `redirect_uri` with a short-lived `code` and the `state` parameter
4. App verifies `state` matches what it sent (CSRF protection)
5. App (backend) makes a server-to-server POST to `/token` with the `code`, `client_id`, `client_secret`, and `redirect_uri`
6. Auth Server validates and returns `access_token`, `refresh_token`, and `id_token`
7. App uses `access_token` to call the Resource Server API

The code never grants access itself — it must be exchanged with `client_secret` on the back channel, keeping tokens out of the browser.

---

### Q4: What is PKCE and why was it introduced?

**Answer:**
PKCE (Proof Key for Code Exchange) was introduced to protect the Authorization Code Flow for **public clients** — clients that cannot safely store a `client_secret` (SPAs, mobile apps).

Without PKCE, if an attacker intercepts the authorization code, they can exchange it for tokens (they already have the `client_secret` from decompiling the mobile app).

With PKCE:
1. Client generates a random `code_verifier` (stored locally)
2. Hashes it to create `code_challenge` = SHA256(code_verifier), encoded as Base64URL
3. Sends `code_challenge` in the `/authorize` request
4. Sends `code_verifier` (not the hash) in the `/token` request
5. Server hashes the verifier and compares — match required

An intercepted authorization code is useless without the original `code_verifier`, which was never transmitted.

---

### Q5: When would you use Client Credentials flow vs Authorization Code flow?

**Answer:**
- **Authorization Code Flow**: When a **user** is involved. A web app wants to access Google Drive on behalf of the user. The user consents; tokens are scoped to that user.

- **Client Credentials Flow**: When **no user** is involved — machine-to-machine. Order Service calls Inventory Service. Both are servers. There's no user to authenticate or consent. The client (Order Service) identifies itself with `client_id` + `client_secret` and gets a token scoped to its service permissions.

Rule of thumb: *Is a human user granting access?* Yes → Authorization Code. No → Client Credentials.

---

### Q6: What is in a JWT? What is the difference between RS256 and HS256?

**Answer:**
A JWT has three parts separated by dots: `header.payload.signature`, each Base64URL encoded.

- **Header**: `alg` (signing algorithm), `typ` ("JWT"), `kid` (key ID)
- **Payload**: Claims — `iss`, `sub`, `aud`, `exp`, `iat`, custom claims like `roles`
- **Signature**: Cryptographic proof of integrity

**HS256 (symmetric):** Uses a single shared secret for both signing and verification. All services that need to verify the token must have the secret — security risk in distributed systems.

**RS256 (asymmetric):** Uses a private/public key pair. The Authorization Server signs with the **private key** (secret). Any Resource Server can verify with the **public key** (freely distributed via JWKS endpoint). No shared secrets needed. Preferred for OAuth2 in microservices.

---

### Q7: How does a Resource Server validate a JWT?

**Answer:**
1. Parse the JWT and decode the header to get `alg` and `kid`
2. Fetch the public key from the JWKS endpoint (using `kid` to select the right key)
3. Verify the JWT signature using the public key
4. Check `exp` — reject if expired
5. Check `nbf` — reject if not yet valid
6. Check `iss` — must match the expected issuer (your auth server)
7. Check `aud` — must include this Resource Server's identifier
8. (Optional) Check `jti` against a blacklist for revocation

Spring Boot's `NimbusJwtDecoder` handles steps 1–5 automatically. Steps 6–7 need explicit configuration.

---

### Q8: What is a JWK Set?

**Answer:**
A JWK Set (JWKS) is a JSON document published by the Authorization Server at `/.well-known/jwks.json` (or similar URL). It contains the public keys used to verify JWT signatures.

```json
{
  "keys": [{ "kty": "RSA", "use": "sig", "alg": "RS256", "kid": "key-1", "n": "...", "e": "AQAB" }]
}
```

Resource Servers fetch this endpoint to get the public key(s). The `kid` (Key ID) in the JWT header tells the Resource Server which key to use when multiple keys exist (useful for key rotation — both old and new keys can coexist during rollover).

---

### Q9: What is the difference between @PreAuthorize and @PostAuthorize?

**Answer:**
- **@PreAuthorize**: Evaluated **before** the method runs. If the expression is false, the method is never called. Use this to gate access. Example: `@PreAuthorize("hasRole('ADMIN')")`

- **@PostAuthorize**: Evaluated **after** the method runs, with access to `returnObject`. Use this when authorization depends on the return value. Example: `@PostAuthorize("returnObject.ownerId == authentication.principal.id")`

**Key tradeoff:** @PostAuthorize still executes the method — any side effects (DB writes, emails) occur before authorization is checked. Prefer @PreAuthorize whenever possible. Use @PostAuthorize only when you must inspect the return value.

---

### Q10: Why should CSRF protection be disabled for stateless JWT APIs?

**Answer:**
CSRF attacks exploit the browser's automatic inclusion of cookies in cross-origin requests. If you authenticate via session cookies, a malicious page can trick the browser into sending authenticated requests to your API.

JWT-based APIs send tokens in the `Authorization: Bearer <token>` header. The browser's CORS policy prevents cross-origin JavaScript from setting custom headers without explicit CORS permission from the server. A `evil.com` page cannot set the `Authorization` header on a request to `api.example.com` — so CSRF is not applicable.

Disabling CSRF removes the overhead of CSRF token management for APIs that use header-based authentication. **However, if you store JWT in a cookie (even HttpOnly), you still need CSRF protection.**

---

### Q11: How do you implement OAuth2 social login in Spring Boot?

**Answer:**
1. Add `spring-boot-starter-oauth2-client` dependency
2. Register app with Google/GitHub (get client ID and secret)
3. Configure `application.yml` with registration details
4. Add `.oauth2Login()` to `SecurityFilterChain`
5. Optionally implement `OAuth2UserService`/`OidcUserService` to persist users

Spring Boot auto-handles: redirecting to provider, PKCE (for OIDC), callback handling, token exchange, loading user info from UserInfo endpoint. The authenticated user is available as `@AuthenticationPrincipal OidcUser` (for OIDC providers) or `OAuth2User` (non-OIDC like GitHub).

---

### Q12: What is refresh token rotation?

**Answer:**
Refresh token rotation means every time a refresh token is used to get a new access token, the server **invalidates the old refresh token** and issues a **new one**. The client must store and use the new refresh token for the next refresh.

**Security benefit:** If a refresh token is stolen, when the attacker uses it, the legitimate user's next refresh attempt will fail (their token was invalidated by the attacker's use). The server can detect this — if an already-used refresh token is presented, it suggests theft, and the server can invalidate the entire refresh token family.

Configure in Spring Authorization Server: `tokenSettings.reuseRefreshTokens(false)`.

---

### Q13: How do you propagate security context between microservices?

**Answer:**
Two main patterns:

**1. JWT Propagation (user context):** When Service A calls Service B on behalf of a user, it forwards the user's JWT in the `Authorization: Bearer` header. Service B validates the JWT and sees the same user context. Implemented via Feign interceptors or WebClient filters.

**2. Client Credentials (service context):** When a service calls another service with no user context (background job, internal API), it uses the Client Credentials flow to get its own service token. The downstream service sees the calling service's identity, not a user.

**API Gateway pattern:** Gateway validates JWT once, extracts claims, and passes them as trusted internal headers (`X-User-Id`, `X-User-Roles`). Downstream services trust these headers (they're on the internal network). More efficient but requires network trust. Zero Trust alternative: every service validates the JWT independently.

---

### Q14: What is the difference between access token and ID token?

**Answer:**
| | Access Token | ID Token |
|-|-------------|---------|
| Purpose | Prove authorization to access an API | Prove the user's identity |
| For | Resource Server (your API) | Client application |
| Contains | Scopes, permissions | User claims: email, name, sub |
| Should be sent to API? | YES | NO |
| Format | JWT or opaque | Always JWT |

The ID Token is evidence of successful authentication — it tells your app "the user is Amith Krishnan with this email." The Access Token grants permission to call APIs. Never send the ID Token to API endpoints.

---

### Q15: What are the risks of storing JWT in localStorage?

**Answer:**
**XSS (Cross-Site Scripting):** If any JavaScript on your page can run malicious code (XSS vulnerability — even in a third-party library you included), it can call `localStorage.getItem('access_token')` and exfiltrate the token. The attacker can then impersonate the user from anywhere.

**Alternatives:**
- Store access token in memory (JavaScript variable) — lost on refresh, but XSS-resistant
- Store refresh token in HttpOnly cookie — JavaScript cannot access HttpOnly cookies at all
- Access token in memory + refresh token in HttpOnly SameSite cookie = best balance

The general rule: **If JavaScript can read it, XSS can steal it.** HttpOnly cookies are the only browser storage that JavaScript cannot access.

---

### Q16: What is Spring Authorization Server?

**Answer:**
Spring Authorization Server is the official OAuth2/OIDC Authorization Server from the Spring team, released as a standalone project. It allows you to build your own Authorization Server embedded in a Spring Boot application.

Features: Authorization Code + PKCE, Client Credentials, Refresh Token flows, OIDC support, JWT signing with RSA, RegisteredClient configuration, custom token claims, JWKS endpoint.

**When to use vs Keycloak:**
- **Keycloak:** Need rich user management, social login federation, admin UI, MFA, enterprise features — use Keycloak (separate server)
- **Spring Authorization Server:** Building a simple auth server you fully control in Java, embedding in your microservices ecosystem, when you don't want to run and maintain a separate Keycloak server

---

### Q17: What is the state parameter in OAuth2, and why is it important?

**Answer:**
The `state` parameter is a random, unguessable value generated by the client before redirecting to the Authorization Server. It's included in the `/authorize` request and echoed back in the redirect callback.

The client stores the state in the user's session and verifies that the `state` in the callback matches what was stored. If they don't match, the callback is rejected.

**Purpose:** Prevents CSRF attacks on the OAuth2 flow itself. Without `state`, an attacker could craft a callback URL and trick a logged-in user into linking their account to the attacker's account (account hijacking).

---

### Q18: What is the nonce in OIDC?

**Answer:**
The `nonce` is a random value included in the OIDC authorization request. The Authorization Server embeds it in the ID Token. The client verifies the `nonce` in the ID Token matches what it sent.

**Purpose:** Prevents replay attacks. If an attacker captures an ID Token and tries to replay it in a different session, the `nonce` won't match the client's expected value, so the authentication is rejected. The `nonce` ties the ID Token to a specific authentication session.

---

### Q19: What is the difference between roles and authorities in Spring Security?

**Answer:**
In Spring Security, they're the same underlying concept (`GrantedAuthority`), but with a naming convention:

- **Role:** A `GrantedAuthority` with the `ROLE_` prefix. `ROLE_ADMIN`, `ROLE_USER`.
  - Checked with `hasRole('ADMIN')` — Spring adds the prefix internally
  - Or `hasAuthority('ROLE_ADMIN')` — explicit full string

- **Authority/Permission:** A `GrantedAuthority` without the `ROLE_` prefix. `READ_USER`, `WRITE_ORDER`.
  - Checked with `hasAuthority('READ_USER')`
  - More fine-grained than roles

Best practice: Use roles for coarse-grained access (ADMIN, USER) and authorities for fine-grained permissions (READ_USER, DELETE_ORDER). Roles can imply authorities via role hierarchy.

---

### Q20: How does Spring Security's filter chain work with JWT?

**Answer:**
When a JWT request arrives:

1. `BearerTokenAuthenticationFilter` (added by `.oauth2ResourceServer()`) intercepts requests
2. Extracts `Authorization: Bearer <token>` header
3. Calls `JwtDecoder.decode()` — validates signature, expiry, issuer
4. Calls `JwtAuthenticationConverter` — converts JWT claims to `JwtAuthenticationToken` with `GrantedAuthority` list
5. Sets `SecurityContextHolder.getContext().setAuthentication(jwtAuthToken)`
6. Request proceeds to `AuthorizationFilter` which checks `.authorizeHttpRequests()` rules
7. If authorized, request reaches the Controller

If any step fails (invalid signature, expired token, insufficient authority), Spring throws an exception handled by `ExceptionTranslationFilter`, returning 401 or 403.

---

### Q21: What is token introspection?

**Answer:**
Token introspection (RFC 7662) is the process of validating an opaque (non-JWT) access token by calling the Authorization Server's introspection endpoint.

```http
POST /oauth2/introspect
Authorization: Basic base64(resource-server-id:secret)
Content-Type: application/x-www-form-urlencoded

token=opaque_token_here
```

Response:
```json
{ "active": true, "sub": "user-123", "scope": "read:data", "exp": 1735600000 }
```

**Trade-offs vs JWT:**
- Introspection: network call per request (slower), but instant revocation works
- JWT: local validation (faster), but revocation is hard without a blacklist

Use introspection when you need true instant revocation. Use JWT for performance-critical, stateless systems where short TTL provides adequate security.

---

### Q22: How would you handle a compromised refresh token?

**Answer:**
With refresh token rotation:
1. Attacker uses the stolen refresh token → gets new access + refresh tokens
2. Legitimate user attempts refresh with their (now invalidated) token → server detects reuse
3. Server recognizes this as a potential theft indicator → invalidates the **entire refresh token family** for that user
4. User is forced to re-authenticate

Without rotation, the attacker has indefinite access until the refresh token expires.

Additional measures:
- Store refresh token hash (not plain) in DB
- Associate refresh tokens with IP/user-agent and flag suspicious changes
- Implement absolute expiry (refresh token TTL regardless of usage)
- Provide users with "revoke all sessions" capability

---

### Q23: What is the difference between Authorization Code + PKCE vs Implicit flow?

**Answer:**
**Implicit flow (deprecated):** Returns access token directly in the URL fragment: `https://app.com/callback#access_token=...`. The token is exposed in browser history, referrer headers, server logs.

**Auth Code + PKCE:** Returns an authorization code in the URL. The code is exchanged for tokens in a back-channel request (from client code to auth server). Tokens never appear in URLs. PKCE prevents the code from being stolen and exchanged without the `code_verifier`.

PKCE gives public clients (SPAs) the security level that was previously only available to confidential clients with client secrets. The IETF deprecated Implicit flow in OAuth 2.1; PKCE is the required replacement.

---

### Q24: What is @EnableMethodSecurity and how is it different from @EnableGlobalMethodSecurity?

**Answer:**
`@EnableGlobalMethodSecurity` was the legacy annotation (deprecated in Spring Security 5.6). `@EnableMethodSecurity` is the modern replacement (Spring Security 5.6+).

Key differences:
- `@EnableMethodSecurity` enables `@PreAuthorize` and `@PostAuthorize` by default (no flags needed)
- Better uses Spring AOP's proxy mechanism
- More composable configuration
- `@EnableGlobalMethodSecurity(prePostEnabled = true)` is the old equivalent

```java
// Old (deprecated):
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)

// New:
@EnableMethodSecurity  // prePost enabled by default
// or
@EnableMethodSecurity(securedEnabled = true, jsr250Enabled = true)
```

---

### Q25: How do you test Spring Security in unit tests?

**Answer:**
```java
@WebMvcTest(UserController.class)
@Import(SecurityConfig.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    // Test with mock JWT
    @Test
    @WithMockUser(username = "amith", roles = {"ADMIN"})
    void shouldReturnUsersForAdmin() throws Exception {
        mockMvc.perform(get("/api/admin/users"))
            .andExpect(status().isOk());
    }

    // Test with JWT token details
    @Test
    void shouldReturnProfileForAuthenticatedUser() throws Exception {
        mockMvc.perform(get("/api/me")
                .with(jwt()
                    .jwt(jwt -> jwt
                        .subject("user-123")
                        .claim("email", "amith@example.com")
                        .claim("roles", List.of("USER")))))
            .andExpect(status().isOk());
    }

    // Test unauthorized
    @Test
    void shouldReturn401WhenNoToken() throws Exception {
        mockMvc.perform(get("/api/me"))
            .andExpect(status().isUnauthorized());
    }

    // Test forbidden
    @Test
    @WithMockUser(roles = {"USER"})
    void shouldReturn403ForUserAccessingAdminEndpoint() throws Exception {
        mockMvc.perform(get("/api/admin/users"))
            .andExpect(status().isForbidden());
    }
}
```

---

### Q26: What is the Authorization Server's discovery document and why is it useful?

**Answer:**
The discovery document (RFC 8414) is a JSON file at `/.well-known/openid-configuration` (or `/.well-known/oauth-authorization-server`). It contains all the metadata needed to interact with the Authorization Server:

- `authorization_endpoint`, `token_endpoint`, `userinfo_endpoint`
- `jwks_uri` (where to get public keys)
- `issuer` (expected value for `iss` claim validation)
- Supported grant types, scopes, algorithms

**Why useful:** Clients don't need hardcoded endpoint URLs. Just configure `issuer-uri` in Spring Boot and it auto-discovers everything from the discovery document. This makes switching auth providers (Keycloak → Okta → Custom) easier — just change `issuer-uri`.

---

### Q27: What is the difference between scope and claim in OAuth2/OIDC?

**Answer:**
- **Scope:** A permission request. The client requests scopes in the authorization request (`scope=openid profile email`). The user consents to these scopes. Scopes determine which claims are included in the tokens and which APIs can be accessed.

- **Claim:** An assertion in a token about the user or the token itself. Claims are the actual data: `"email": "amith@example.com"`, `"sub": "user-123"`, `"exp": 1735600000`.

Scopes determine *what claims you get*. `profile` scope → you get `name`, `given_name`, `picture` claims. `email` scope → you get `email`, `email_verified` claims.

---

### Q28: How do you implement logout with OAuth2?

**Answer:**
Logout in OAuth2 has two parts:

**1. Local logout:** Clear the local session/token storage
```java
http.logout(logout -> logout
    .logoutUrl("/logout")
    .clearAuthentication(true)
    .invalidateHttpSession(true)
    .deleteCookies("JSESSIONID", "refresh_token")
    .logoutSuccessUrl("/")
);
```

**2. OIDC RP-Initiated Logout (complete logout):** Redirect to the Authorization Server's `end_session_endpoint` to also invalidate the SSO session
```java
http.oidcLogout(oidcLogout -> oidcLogout
    .backChannel(Customizer.withDefaults())
);
```

Or manual redirect:
```
GET https://auth.example.com/logout?
    id_token_hint=<id_token>
    &post_logout_redirect_uri=https://myapp.com/
    &state=random123
```

Without OIDC RP-Initiated Logout, the user is logged out of your app but still has an active SSO session at the Authorization Server — clicking "Login" would immediately log them back in without prompting for credentials.

---

### Q29: What is Bearer Token authentication and how does Spring Security handle it?

**Answer:**
Bearer Token authentication means: "Whoever **bears** (presents) this token is granted access." No additional proof of identity is required — just possession of the token.

Header format: `Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...`

Spring Security handles it via `BearerTokenAuthenticationFilter` (added when you configure `.oauth2ResourceServer()`). The filter:
1. Looks for `Authorization: Bearer` header (or optionally `access_token` query param — not recommended)
2. Extracts the token string
3. Passes it to the configured `AuthenticationManager` which uses `JwtDecoder` or introspection
4. On success, populates `SecurityContext`

The "bearer" nature means token security entirely depends on transport security (HTTPS). If the token is stolen, the thief has full access until the token expires.

---

### Q30: What is OpenID Connect's Hybrid Flow?

**Answer:**
The Hybrid Flow is an OIDC flow where the authorization response includes both a `code` **and** tokens (ID token and/or access token) directly.

`response_type` can be: `code id_token`, `code token`, or `code id_token token`

```
response_type=code id_token → get code + ID token in the front channel
                              then exchange code for access + refresh tokens in back channel
```

**Use case:** When the client needs the ID Token immediately (to get user info for the UI) but still needs to exchange the code for an access token and refresh token via the back channel.

Rarely needed in practice — Authorization Code + PKCE with immediate UserInfo call covers most cases.

---

### Q31: How would you implement multi-tenancy with OAuth2 in Spring Boot?

**Answer:**
```java
@Bean
public JwtDecoder jwtDecoder() {
    // Validate issuer dynamically based on tenant
    return token -> {
        // Extract issuer from token (unverified decode first)
        String issuer = JWTParser.parse(token).getJWTClaimsSet().getIssuer();

        // Look up tenant-specific JWKS URI
        String jwksUri = tenantRepository.getJwksUriForIssuer(issuer);
        if (jwksUri == null) {
            throw new JwtException("Unknown issuer: " + issuer);
        }

        NimbusJwtDecoder decoder = NimbusJwtDecoder
            .withJwkSetUri(jwksUri).build();

        return decoder.decode(token);
    };
}
```

Or use `JwtIssuerAuthenticationManagerResolver` (Spring Security built-in):
```java
http.oauth2ResourceServer(oauth2 -> oauth2
    .authenticationManagerResolver(
        new JwtIssuerAuthenticationManagerResolver(
            "https://tenant1.auth.example.com",
            "https://tenant2.auth.example.com"
        )
    )
);
```

---

### Q32: What happens if you don't configure CORS and your frontend is on a different origin?

**Answer:**
The browser's **preflight request** (OPTIONS) or the actual request will fail. The browser checks the response for `Access-Control-Allow-Origin` header. If absent or not matching the origin, the browser blocks the response with a CORS error in the console.

The request **does reach the server** (CORS is browser-enforced, not server-enforced). The server processes it and responds, but the browser refuses to give the response to the JavaScript. This means:
- GET requests might succeed at the server level (data fetched) but JS can't read the response
- State-changing requests (POST, PUT) are blocked by preflight before they even hit the server

Correct fix: configure `CorsConfigurationSource` bean and register it with `.cors()` in Spring Security.

---

### Q33: What is the JTI claim and how is it used for token revocation?

**Answer:**
`jti` (JWT ID) is a unique identifier for each token. It allows individual tokens to be revoked without affecting others.

**Token revocation with JTI blacklist:**
```java
// On logout:
blacklistRepository.save(jwt.getId(), jwt.getExpiresAt()); // store JTI until token would expire

// On each request, add JTI check to JWT decoder:
OAuth2TokenValidator<Jwt> jtiValidator = jwt -> {
    if (blacklistRepository.isBlacklisted(jwt.getId())) {
        return OAuth2TokenValidatorResult.failure(
            new OAuth2Error("token_revoked", "Token has been revoked", null));
    }
    return OAuth2TokenValidatorResult.success();
};
```

Store in Redis with TTL matching token expiry — no need to keep entries after the token would have expired anyway.

---

### Q34: What is the difference between spring-boot-starter-oauth2-resource-server and spring-boot-starter-oauth2-client?

**Answer:**
- **oauth2-resource-server:** For APIs that receive and validate tokens. Your service is the **Resource Server** — it protects endpoints. Incoming requests must have a Bearer token. Configure with `.oauth2ResourceServer(oauth2 -> oauth2.jwt(...))`.

- **oauth2-client:** For applications that obtain tokens and use them to call other services, OR for applications that implement social login (where your app is the **Client**). Configure with `.oauth2Login()` or `OAuth2AuthorizedClientManager` for outbound calls.

A microservice API → resource-server only.
A BFF (Backend For Frontend) → both: oauth2-client (to get tokens from auth server on behalf of user) and possibly resource-server (to validate tokens from the API gateway).

---

### Q35: How do you prevent the "confused deputy" problem in microservices?

**Answer:**
The confused deputy problem: Service A has high privileges. Service B (with lower privileges) tricks Service A into doing something privileged on its behalf.

**Prevention with proper JWT audience validation:**
- Each service should only accept tokens with its own `aud` (audience) claim
- A token issued for Service A cannot be used against Service B

**Prevention with scopes:**
- Tokens should have minimum necessary scopes
- Service B's client credentials token has `read:inventory` — it cannot request `delete:users` scope

**Prevention with mTLS (Mutual TLS):**
- Services authenticate each other with certificates, not just bearer tokens
- Even with a stolen token, the attacker must also present the valid client certificate

**Sender-constrained tokens (DPoP):** Tokens are bound to a specific client's key pair — cannot be replayed by a different client.

---

### Q36: What is OAuth 2.1 and how is it different from OAuth 2.0?

**Answer:**
OAuth 2.1 (still in draft) consolidates OAuth 2.0 best practices and deprecates insecure flows:

**Changes from 2.0:**
1. PKCE **required** for Authorization Code flow (was optional)
2. Implicit flow **removed**
3. Resource Owner Password Credentials **removed**
4. Redirect URIs must be exact matches (no wildcards)
5. Refresh tokens for public clients must be sender-constrained or use rotation
6. Bearer tokens in query strings **not recommended**

OAuth 2.1 is essentially "OAuth 2.0 minus the dangerous parts, plus PKCE everywhere."

---

### Q37: How does Spring Security handle the SecurityContext in async or reactive scenarios?

**Answer:**
In traditional (servlet) Spring Security, `SecurityContextHolder` uses a `ThreadLocal` — the security context is bound to the current thread.

**Problem with async:** When you spawn a new thread (via `@Async`, `CompletableFuture`, thread pools), the `ThreadLocal` is not inherited.

**Solutions:**

```java
// 1. DelegatingSecurityContextExecutor
@Bean
public Executor securityAwareExecutor() {
    return new DelegatingSecurityContextExecutor(Executors.newCachedThreadPool());
}

// 2. Propagate manually
SecurityContext context = SecurityContextHolder.getContext();
CompletableFuture.runAsync(() -> {
    SecurityContextHolder.setContext(context);
    // do work
    SecurityContextHolder.clearContext();
});
```

**Reactive (WebFlux):** `SecurityContextHolder` doesn't work. Use `ReactiveSecurityContextHolder`:
```java
Mono<String> result = ReactiveSecurityContextHolder.getContext()
    .map(ctx -> ctx.getAuthentication().getName());
```

---

### Q38: What are the security implications of JWT algorithm "none"?

**Answer:**
The JWT specification allows `"alg": "none"` — a token with no signature. Any JWT library that honors this allows an attacker to:
1. Decode any JWT (Base64 is not encryption)
2. Modify claims (change `sub`, `roles`, `exp`)
3. Set `"alg": "none"` in the header
4. Submit the modified token (no signature needed)

Vulnerable libraries would accept this as valid.

**In Spring Security:** `NimbusJwtDecoder` is safe — it requires explicit algorithm configuration and rejects `none`. Always configure:
```java
NimbusJwtDecoder.withJwkSetUri(jwksUri)
    .jwsAlgorithm(SignatureAlgorithm.RS256)  // explicit algorithm
    .build();
```

This CVE class (CVE-2015-9235 and similar) is why you should never roll your own JWT validation.

---

### Q39: How would you implement fine-grained authorization (not just roles)?

**Answer:**
For resource-level authorization (user can edit *their own* post, not all posts):

**Option 1: @PreAuthorize with SpEL**
```java
@PreAuthorize("@postService.isOwner(authentication, #postId)")
public void deletePost(Long postId) {}
```

**Option 2: Spring Security ACL (Access Control Lists)**
Full ACL framework — stores per-object permissions in database:
```java
mutableAclService.createAcl(new ObjectIdentityImpl(Post.class, postId));
acl.insertAce(0, BasePermission.WRITE, new PrincipalSid(userId), true);
```

**Option 3: Check inside service method**
```java
public void deletePost(Long postId, Authentication auth) {
    Post post = repository.findById(postId).orElseThrow();
    if (!post.getOwnerId().equals(auth.getName())) {
        throw new AccessDeniedException("Not the post owner");
    }
    repository.delete(post);
}
```

For most applications, @PreAuthorize with a custom security bean is the cleanest approach without the complexity of Spring ACL.

---

### Q40: What is OIDC Back-Channel Logout?

**Answer:**
OIDC Back-Channel Logout allows the Authorization Server to directly notify the Client (your app) when a user logs out, even if the user didn't log out from your app.

```
User logs out from Auth Server → 
Auth Server POSTs to https://myapp.com/logout/connect/back-channel/{registrationId} →
Spring Security invalidates the user's session
```

This ensures that if a user logs out of Keycloak (or their SSO session expires), all applications in the SSO ecosystem are notified and invalidate their local sessions — true global logout.

Configure in Spring Security:
```java
http.oidcLogout(logout -> logout
    .backChannel(backChannel -> backChannel
        .logoutUri("/logout/connect/back-channel/{registrationId}")
    )
);
```

Without this, a user who "logs out" of the IdP still has active sessions in individual apps until they expire.

---

### Q41: How do you configure different security rules for different API versions?

**Answer:**
```java
@Bean
public SecurityFilterChain apiV1Chain(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/api/v1/**")
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(HttpMethod.GET, "/api/v1/public/**").permitAll()
            .anyRequest().hasRole("USER")
        )
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
    return http.build();
}

@Bean
@Order(1)  // higher priority
public SecurityFilterChain apiV2Chain(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/api/v2/**")
        .authorizeHttpRequests(auth -> auth
            .anyRequest().hasRole("PREMIUM_USER")
        )
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
    return http.build();
}
```

Multiple `SecurityFilterChain` beans can coexist. `securityMatcher()` limits each chain to specific URL patterns. `@Order` controls which chain takes priority for overlapping patterns.

---

### Q42: What is Mutual TLS (mTLS) and how does it complement OAuth2?

**Answer:**
Standard TLS: Client verifies the server's certificate. Server trusts anyone who connects.
**Mutual TLS (mTLS):** Both client and server present certificates. The server verifies the client's certificate.

In microservices, mTLS:
- Ensures Service A can only call Service B if it has the trusted client certificate
- Prevents network-level attacks (even if an attacker is on the internal network)
- Used as "Sender-Constrained Tokens" — token is cryptographically bound to the client's certificate

Spring Boot mTLS configuration:
```yaml
server:
  ssl:
    client-auth: need  # require client certificate
    trust-store: classpath:truststore.jks
    trust-store-password: ${TRUSTSTORE_PASSWORD}
    key-store: classpath:keystore.jks
    key-store-password: ${KEYSTORE_PASSWORD}
```

OAuth2 Certificate-Bound Access Tokens (RFC 8705): The access token's `cnf.x5t#S256` claim contains the SHA-256 thumbprint of the client's certificate. Resource Server validates that the presented certificate matches the token binding.

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
| Implicit | DEPRECATED — don't use |
| ROPC | DEPRECATED — don't use |

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
1. PKCE for all public clients (no exceptions)
2. Short-lived access tokens (15 min)
3. Refresh tokens in HttpOnly SameSite cookies
4. Never store tokens in localStorage
5. Always validate aud claim
6. Disable CSRF for header-based JWT APIs
7. RS256 over HS256 (no shared secrets)
8. Use state parameter always
9. HTTPS everywhere (OAuth2 relies on TLS)
10. Validate all JWT claims (sig, exp, iss, aud)
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

// Role Hierarchy
@Bean RoleHierarchy roleHierarchy() { ... }

// CORS
http.cors(cors -> cors.configurationSource(corsSource()));

// CSRF (disable for stateless JWT APIs)
http.csrf(csrf -> csrf.disable());

// Stateless sessions
http.sessionManagement(s -> s.sessionCreationPolicy(STATELESS));
```

### JWT Validation Order

```
1. Decode & parse structure
2. Verify signature (JWKS public key)
3. Check exp (not expired)
4. Check nbf (not before)
5. Check iss (expected issuer)
6. Check aud (includes this server)
7. Check jti (not blacklisted) — optional
```

---

*This guide covers OAuth2, OIDC, JWT, Spring Security OAuth2 Resource Server, OAuth2 Client, Social Login, Spring Authorization Server, Method Security, CORS, CSRF, Refresh Tokens, Keycloak, and Microservice Security Patterns — all critical areas for Full Stack Java Developer interviews.*
