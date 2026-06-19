# Spring OAuth2, OIDC & Security — Awareness Notes

> **Scope note (junior job prep):** Deep OAuth2/OIDC (grant-type internals, PKCE, Keycloak, building an Authorization Server, method-security internals, microservice token propagation, refresh-token rotation) is an **advanced topic deferred for later study**. This file is trimmed to a concept-awareness section. Your junior-level security essentials — JWT basics, authentication vs authorization, CORS/CSRF, BCrypt, `@PreAuthorize` — are covered in **`Spring_Security_JWT_Interview_Questions.md`**, which is kept in full. The deep deep-dive remains in git history for later.

---

## The Concepts Worth Knowing Now

- **OAuth2 = authorization framework** ("what can this app do on your behalf?"). It lets an app get limited access to a user's resources without the user sharing their password.
- **OIDC (OpenID Connect) = OAuth2 + an identity layer** ("who is this user?"). It adds the **ID Token** (a JWT of identity claims), a UserInfo endpoint, and standard scopes (`openid`, `profile`, `email`).
- One-liner: **OAuth2 = authorization. OIDC = OAuth2 + authentication.**

### The four OAuth2 roles
| Role | Who | Example |
|---|---|---|
| Resource Owner | the user | you |
| Client | the app requesting access | your web/mobile app |
| Authorization Server | issues tokens | Google, Keycloak, Okta |
| Resource Server | hosts the protected API | your backend |

### Grant types at a glance
| Flow | Use when |
|---|---|
| Authorization Code | web app with a backend, user involved |
| Authorization Code + PKCE | SPA / mobile (can't keep a client secret) |
| Client Credentials | service-to-service, no user |
| Implicit / Password (ROPC) | **deprecated — don't use** |

### Token types
| Token | Purpose | Sent to your API? |
|---|---|---|
| Access token | call APIs | **yes** (`Authorization: Bearer`) |
| Refresh token | get a new access token | no (only to the auth server) |
| ID token (OIDC) | tells the client who the user is | **no** |

## What a Junior Most Likely Touches

- **Social login** ("Log in with Google/GitHub") uses the `spring-boot-starter-oauth2-client` starter and `.oauth2Login()` — Spring handles the redirect, callback, and token exchange for you.
- **Protecting an API** that receives JWTs uses the `spring-boot-starter-oauth2-resource-server` starter with an `issuer-uri` — Spring auto-discovers the keys and validates tokens.

> **Interview soundbite:** "OAuth2 is for authorization, OIDC adds authentication on top. For social login I'd use the OAuth2 client starter; to protect an API I'd use the resource-server starter with an issuer URI. I know the Authorization Code + PKCE flow conceptually, but I've focused on the JWT-secured Spring Boot basics."

---

*Trimmed to awareness level for junior job prep. Restore the full OAuth2/OIDC deep-dive from version control when you're ready to study it.*
