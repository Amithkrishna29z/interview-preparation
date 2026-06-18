# Full-Stack Integration: Connecting React/Next.js to Spring Boot

> **How to use this guide:** The highest-value topic for a junior full-stack role is the complete request lifecycle — from a React form submit through HTTP to a Spring Boot controller and back. Start with [CORS](#4-cors-the-most-common-full-stack-blocker) (the most common day-one bug), then [JWT on the frontend](#6-jwt-on-the-frontend), then [form validation round-trip](#7-form-handling--backend-validation-display). This guide assumes you already know React basics, JWT concepts, and Bean Validation — see `React_Interview_Questions.md`, `Spring_Security_JWT_Interview_Questions.md`, and `Spring_Bean_Validation_Guide.md` for those foundations.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Calling the Backend: Axios vs fetch](#2-calling-the-backend-axios-vs-fetch)
3. [Async Data Fetching and State Management](#3-async-data-fetching-and-state-management)
4. [CORS — The Most Common Full-Stack Blocker](#4-cors-the-most-common-full-stack-blocker)
5. [Environment Config and Dev Proxy](#5-environment-config-and-dev-proxy)
6. [JWT on the Frontend](#6-jwt-on-the-frontend)
7. [Form Handling and Backend Validation Display](#7-form-handling--backend-validation-display)
8. [Common Full-Stack Bugs](#8-common-full-stack-bugs)
9. [Common Interview Questions](#9-common-interview-questions)
10. [Quick Reference Cheat Sheet](#10-quick-reference-cheat-sheet)

---

## 1. Overview

A junior full-stack role means you own the entire request lifecycle:

```
React UI  →  HTTP request  →  Spring Boot controller  →  DB
         ←  JSON response  ←  service / repository   ←
```

The "glue" between frontend and backend involves four recurring concerns:

| Concern | Key Decision |
|---------|-------------|
| HTTP client | Axios instance (preferred) vs raw fetch |
| Auth | JWT in memory + HttpOnly cookie for refresh |
| CORS | Spring `WebMvcConfigurer` + Security config |
| Validation errors | Backend 400 payload mapped to form fields |

---

## 2. Calling the Backend: Axios vs fetch

### Why Axios over raw fetch

| Feature | fetch (built-in) | Axios |
|---------|-----------------|-------|
| Auto JSON parse | No — must call `.json()` | Yes |
| Request interceptors | No | Yes — essential for JWT |
| Response interceptors | No | Yes — essential for token refresh |
| Base URL config | Manual | Built-in `baseURL` |
| Timeout support | No (AbortController needed) | Yes |

For a full-stack app with JWT auth, Axios interceptors are worth the dependency.

### Create ONE configured Axios instance

Never hardcode URLs at every call site. Create a single instance and import it everywhere.

```js
// src/api/axiosInstance.js
import axios from 'axios';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL, // Vite — see section 5
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

export default api;
```

Then use it anywhere:

```js
// src/api/userApi.js
import api from './axiosInstance';

export const getUsers = () => api.get('/users');
export const createUser = (data) => api.post('/users', data);
export const deleteUser = (id) => api.delete(`/users/${id}`);
```

---

## 3. Async Data Fetching and State Management

Every data-fetching component needs three explicit states: **loading**, **error**, **data**. Skipping any one causes UI bugs.

### Complete fetch component pattern

```jsx
// src/components/UserList.jsx
import { useState, useEffect } from 'react';
import api from '../api/axiosInstance';

function UserList() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false; // prevent state update on unmounted component

    api.get('/users')
      .then(res => {
        if (!cancelled) setUsers(res.data);
      })
      .catch(err => {
        if (!cancelled) setError(err.response?.data?.message ?? 'Failed to load users');
      })
      .finally(() => {
        if (!cancelled) setLoading(false);
      });

    return () => { cancelled = true; }; // cleanup
  }, []); // empty deps = run once on mount

  if (loading) return <p>Loading...</p>;
  if (error)   return <p className="error">{error}</p>;

  return (
    <ul>
      {users.map(u => <li key={u.id}>{u.name}</li>)}
    </ul>
  );
}

export default UserList;
```

### Next.js server components (alternative)

In Next.js 13+ App Router, server components fetch directly without `useEffect`:

```jsx
// app/users/page.jsx  — runs on the SERVER, no client bundle
async function UsersPage() {
  const res = await fetch(`${process.env.API_URL}/users`, {
    cache: 'no-store', // or 'force-cache' / revalidate: 60
  });
  const users = await res.json();

  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

Server components use `process.env.API_URL` (not `NEXT_PUBLIC_*`) because they never reach the browser.

---

## 4. CORS — The Most Common Full-Stack Blocker

### What CORS actually is

CORS (Cross-Origin Resource Sharing) is a **browser** security mechanism. The browser blocks responses from a different origin unless the server explicitly allows it.

> **Different origin** = different protocol, host, OR port. `localhost:5173` (React dev) and `localhost:8080` (Spring Boot) are different origins.

**The key insight:** Your API works fine in Postman and curl because those tools are not browsers — they don't enforce CORS. The error is always in the browser.

### Preflight requests

Before a non-simple request (PUT, DELETE, or any request with a custom header like `Authorization`), the browser sends an `OPTIONS` preflight to ask: "Are you OK with this?". The server must respond with the right headers or the actual request is blocked.

### Spring Boot: @CrossOrigin (quick, per controller)

```java
@CrossOrigin(origins = "http://localhost:5173")
@RestController
@RequestMapping("/api/users")
public class UserController { ... }
```

Use this only for quick prototyping. Prefer the global config for production code.

### Spring Boot: Global WebMvcConfigurer (preferred)

```java
// config/CorsConfig.java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins(
                "http://localhost:5173",   // Vite dev
                "https://your-app.com"     // production
            )
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS") // OPTIONS is required
            .allowedHeaders("*")
            .allowCredentials(true)   // required if sending cookies (e.g., refresh token)
            .maxAge(3600);            // preflight cache: 1 hour
    }
}
```

### CRITICAL: Spring Security blocks preflights before MVC config runs

If you have Spring Security, you **must** also tell it to let CORS through. Without this, the security filter chain rejects the OPTIONS preflight with 401 before your `WebMvcConfigurer` ever runs.

```java
// config/SecurityConfig.java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors(Customizer.withDefaults()) // <-- delegates to your WebMvcConfigurer bean
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated()
            )
            .sessionManagement(sm -> sm
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            );
        return http.build();
    }
}
```

`Customizer.withDefaults()` tells Spring Security to pick up the `CorsConfigurationSource` bean — which your `WebMvcConfigurer` provides automatically.

---

## 5. Environment Config and Dev Proxy

### Never hardcode the API URL

| Build tool | Env var prefix | Access in JS |
|-----------|----------------|-------------|
| Vite | `VITE_` | `import.meta.env.VITE_API_URL` |
| CRA | `REACT_APP_` | `process.env.REACT_APP_API_URL` |
| Next.js (client) | `NEXT_PUBLIC_` | `process.env.NEXT_PUBLIC_API_URL` |
| Next.js (server) | (no prefix) | `process.env.API_URL` |

```bash
# .env.development
VITE_API_URL=http://localhost:8080/api

# .env.production
VITE_API_URL=https://api.your-app.com/api
```

```js
// src/api/axiosInstance.js
const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
});
```

### Dev proxy (avoids CORS in development only)

A proxy rewrites requests so the browser thinks it's talking to the same origin.

**Vite (`vite.config.js`):**

```js
export default {
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
        // optional rewrite: removes /api prefix before forwarding
        // rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
  },
};
```

**CRA (`package.json`):**

```json
{
  "proxy": "http://localhost:8080"
}
```

> **GOTCHA:** The Vite/CRA proxy is a **development-only** Node.js server feature. It does not exist after `npm run build`. In production your app is static files — the proxy is gone. Production must use the real `VITE_API_URL` and your backend must have proper CORS config.

---

## 6. JWT on the Frontend

### Token storage tradeoff

| Storage | Pros | Cons |
|---------|------|------|
| `localStorage` | Simple, survives page reload | Accessible to JS — XSS steals it |
| Memory (JS variable) | XSS can't access it | Lost on page reload |
| HttpOnly cookie | Inaccessible to JS — XSS-safe | Requires `SameSite`/`allowCredentials` setup; CSRF risk |

**Practical junior approach:** Store the short-lived access token in memory (a module-level variable). Store the long-lived refresh token in an HttpOnly cookie with `SameSite=Strict` set by the server. On page reload, immediately call `/auth/refresh` to get a new access token.

### Axios REQUEST interceptor — attach the token

```js
// src/api/axiosInstance.js
let accessToken = null; // module-level — in memory only

export const setAccessToken = (token) => { accessToken = token; };
export const clearAccessToken = () => { accessToken = null; };

api.interceptors.request.use(config => {
  if (accessToken) {
    config.headers['Authorization'] = `Bearer ${accessToken}`;
  }
  return config;
});
```

### Axios RESPONSE interceptor — token refresh on 401

```js
api.interceptors.response.use(
  response => response, // pass through successful responses

  async error => {
    const originalRequest = error.config;

    // Retry once on 401 — the _retry flag prevents infinite loops
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        // Refresh token is in HttpOnly cookie — browser sends it automatically
        const res = await axios.post('/api/auth/refresh', {}, { withCredentials: true });
        const newToken = res.data.accessToken;
        setAccessToken(newToken);
        originalRequest.headers['Authorization'] = `Bearer ${newToken}`;
        return api(originalRequest); // retry original request
      } catch {
        clearAccessToken();
        window.location.href = '/login'; // refresh failed — force re-login
      }
    }

    return Promise.reject(error);
  }
);
```

### Protected route (React Router v6)

```jsx
// src/components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

function ProtectedRoute({ children }) {
  const { isAuthenticated } = useAuth();
  return isAuthenticated ? children : <Navigate to="/login" replace />;
}

// Usage in router:
// <Route path="/dashboard" element={<ProtectedRoute><Dashboard /></ProtectedRoute>} />
```

### On page reload — re-request access token

Since the access token lives in memory, it's gone after a reload. Trigger a refresh on app startup:

```jsx
// src/App.jsx
useEffect(() => {
  axios.post('/api/auth/refresh', {}, { withCredentials: true })
    .then(res => setAccessToken(res.data.accessToken))
    .catch(() => { /* not logged in — stay on public pages */ });
}, []);
```

---

## 7. Form Handling and Backend Validation Display

This is the critical "glue" piece that most tutorials skip.

### The rule: never trust the client

Client-side validation (required fields, email format) is UX — it gives fast feedback. It is NOT security. The backend must **always** re-validate. A user can bypass the browser entirely with curl.

### Backend: Bean Validation + error shaping

```java
// dto/CreateUserRequest.java
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    String name,

    @Email(message = "Must be a valid email")
    @NotBlank(message = "Email is required")
    String email,

    @Size(min = 8, message = "Password must be at least 8 characters")
    String password
) {}

// controller/UserController.java
@PostMapping("/users")
public ResponseEntity<UserResponse> createUser(
        @Valid @RequestBody CreateUserRequest request) {
    return ResponseEntity.status(201).body(userService.create(request));
}
```

```java
// exception/GlobalExceptionHandler.java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Shapes validation errors into a consistent frontend-friendly payload
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidation(
            MethodArgumentNotValidException ex) {

        List<Map<String, String>> fieldErrors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(e -> Map.of(
                "field",   e.getField(),
                "message", e.getDefaultMessage()
            ))
            .toList();

        return ResponseEntity.badRequest().body(Map.of(
            "status",      400,
            "error",       "Validation Failed",
            "fieldErrors", fieldErrors
        ));
    }
}
```

**Response payload for a bad request:**

```json
{
  "status": 400,
  "error": "Validation Failed",
  "fieldErrors": [
    { "field": "email",    "message": "Must be a valid email" },
    { "field": "password", "message": "Password must be at least 8 characters" }
  ]
}
```

### Frontend: catch the 400, map errors to fields

```jsx
// src/components/RegisterForm.jsx
import { useState } from 'react';
import api from '../api/axiosInstance';

function RegisterForm() {
  const [form, setForm]         = useState({ name: '', email: '', password: '' });
  const [fieldErrors, setFieldErrors] = useState({}); // { email: "...", password: "..." }
  const [serverError, setServerError] = useState(null);

  const handleChange = (e) => {
    setForm(prev => ({ ...prev, [e.target.name]: e.target.value }));
    setFieldErrors(prev => ({ ...prev, [e.target.name]: null })); // clear on type
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setFieldErrors({});
    setServerError(null);

    try {
      await api.post('/users', form);
      // success — redirect or show confirmation
    } catch (err) {
      if (err.response?.status === 400) {
        // Map field errors from backend payload
        const errors = {};
        err.response.data.fieldErrors?.forEach(({ field, message }) => {
          errors[field] = message;
        });
        setFieldErrors(errors);
      } else {
        setServerError('Something went wrong. Please try again.');
      }
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <input name="email" value={form.email} onChange={handleChange} />
        {fieldErrors.email && <span className="error">{fieldErrors.email}</span>}
      </div>
      <div>
        <input name="password" type="password" value={form.password} onChange={handleChange} />
        {fieldErrors.password && <span className="error">{fieldErrors.password}</span>}
      </div>
      {serverError && <p className="error">{serverError}</p>}
      <button type="submit">Register</button>
    </form>
  );
}
```

---

## 8. Common Full-Stack Bugs

| Bug | Symptom | Fix |
|-----|---------|-----|
| Missing `http://` in env var | `baseURL` resolves relative — requests go to wrong host | Always include protocol: `http://localhost:8080` |
| Proxy assumed in production | 404 on API calls after deploy | Set `VITE_API_URL` for prod; proxy is dev-only |
| CORS method not listed | Preflight blocked for PUT/DELETE | Add all methods + OPTIONS to `allowedMethods` |
| Spring Security blocks preflight | 401 on OPTIONS before auth headers are set | Add `.cors(Customizer.withDefaults())` to Security config |
| Token in localStorage | XSS can steal token | Move access token to memory; use HttpOnly cookie for refresh |
| Infinite refresh loop | Browser hangs on 401 | Add `_retry` flag to interceptor; check you set it before retrying |
| `allowCredentials` mismatch | Cookies not sent | Both frontend (`withCredentials: true`) and backend (`allowCredentials(true)`) must be set |
| `allowedOrigins("*")` with credentials | Browser error | Wildcard `*` is incompatible with `allowCredentials(true)` — list origins explicitly |

---

## 9. Common Interview Questions

**Q: What is CORS and why does my API work in Postman but fail in the browser?**
CORS is a browser security policy that blocks responses from different origins unless the server opts in via response headers. Postman is not a browser — it sends requests without enforcing the Same-Origin Policy, so it always works. The fix lives on the server: configure Spring Boot to include `Access-Control-Allow-Origin` headers.

**Q: Where should a JWT access token be stored on the frontend?**
The safest approach is to keep the short-lived access token in a JavaScript module variable (in memory), so XSS cannot steal it. The refresh token goes in an HttpOnly cookie the server sets, making it inaccessible to JavaScript. The trade-off is that in-memory tokens are lost on page reload, so the app must call `/auth/refresh` on startup to restore the session.

**Q: How does the frontend display backend validation errors per field?**
The backend returns a 400 with a structured payload — an array of `{field, message}` objects shaped by `@RestControllerAdvice`. The frontend catches the 400 in a `.catch()` block, converts the array into an object keyed by field name, and stores it in state. Each form input then conditionally renders its matching error message below it.

**Q: Why does Spring Security break my CORS config?**
Spring Security's filter chain runs before Spring MVC processes the request. An OPTIONS preflight has no `Authorization` header, so Security rejects it with 401 before your `WebMvcConfigurer` ever runs. Fix: call `http.cors(Customizer.withDefaults())` in your `SecurityFilterChain` — this inserts a CORS filter early in the chain that handles preflights before Security checks for tokens.

**Q: What is the difference between the dev proxy and real CORS config?**
The dev proxy (Vite `server.proxy` or CRA `proxy`) is a Node.js development server feature that rewrites requests to appear same-origin to the browser. It eliminates CORS in development only. After `npm run build`, the static output has no proxy — production must rely on actual CORS headers from the server.

**Q: What is an Axios interceptor and when would you use one?**
An interceptor is a middleware function that runs on every request or response before your component code handles it. A request interceptor is used to attach the JWT `Authorization` header to every outgoing call. A response interceptor is used to detect a 401, silently refresh the token, and retry the failed request — so individual API calls don't need to handle auth errors themselves.

---

## 10. Quick Reference Cheat Sheet

```
AXIOS INSTANCE
  axios.create({ baseURL: import.meta.env.VITE_API_URL, timeout: 10000 })
  Request interceptor  → attach Authorization: Bearer <token>
  Response interceptor → on 401, refresh token, set _retry flag, retry

ENV VARS (never hardcode URLs)
  Vite:    VITE_API_URL      → import.meta.env.VITE_API_URL
  CRA:     REACT_APP_API_URL → process.env.REACT_APP_API_URL
  Next.js: NEXT_PUBLIC_API_URL (client) / API_URL (server components)

DEV PROXY (Vite vite.config.js)
  server.proxy: { '/api': { target: 'http://localhost:8080', changeOrigin: true } }
  ⚠ GONE after build — prod needs real CORS + real base URL

CORS (Spring Boot)
  @CrossOrigin — quick per-controller
  WebMvcConfigurer.addCorsMappings — preferred global config
    .allowedMethods("GET","POST","PUT","DELETE","OPTIONS")  ← OPTIONS is required
    .allowCredentials(true)  ← required for cookies
    .allowedOrigins("http://localhost:5173") ← NO wildcard with credentials
  SecurityFilterChain: http.cors(Customizer.withDefaults()) ← required or Security blocks preflights

JWT STORAGE
  Access token  → JS memory (module variable) — lost on reload, safe from XSS
  Refresh token → HttpOnly cookie (server sets it) — invisible to JS
  On reload     → call /auth/refresh immediately to restore access token

VALIDATION ROUND-TRIP
  Backend: @Valid @RequestBody + @RestControllerAdvice → { fieldErrors: [{field, message}] }
  Frontend: catch 400 → build { field: message } map → render under each input

COMMON GOTCHAS
  allowedOrigins("*") + allowCredentials(true) = browser error → list origins explicitly
  Spring Security + CORS = must add http.cors(...) or preflights get 401
  Proxy in prod = 404 → set VITE_API_URL for production build
  Infinite 401 loop = missing _retry flag in response interceptor
```

---

*Last Updated: 2026-06-18*
