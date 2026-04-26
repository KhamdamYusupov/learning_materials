# Keycloak: Complete Production Guide for Java/Spring Boot Engineers

> For backend developers with 3+ years Java experience building secure, production-grade authentication systems from scratch.

---

## Table of Contents

1. [Identity & Access Management Fundamentals](#1-identity--access-management-fundamentals)
2. [Keycloak Fundamentals](#2-keycloak-fundamentals)
3. [Keycloak Architecture & How It Works](#3-keycloak-architecture--how-it-works)
4. [Authentication & Authorization Flows](#4-authentication--authorization-flows)
5. [Keycloak in System Design](#5-keycloak-in-system-design)
6. [Production Best Practices](#6-production-best-practices)
7. [Keycloak + Spring Boot](#7-keycloak--spring-boot)
8. [Advanced Topics](#8-advanced-topics)
9. [Real-World Scenarios](#9-real-world-scenarios)
10. [Comparisons & Trade-offs](#10-comparisons--trade-offs)
11. [Production-Ready Checklist](#11-production-ready-checklist)

---

# 1. Identity & Access Management Fundamentals

## What IAM Is and Why It Matters

Identity and Access Management (IAM) is the discipline of ensuring that **the right entities access the right resources under the right conditions**. In software systems it answers two questions:

1. **Who are you?** (Authentication)
2. **What are you allowed to do?** (Authorization)

Without a proper IAM layer:
- Every service reinvents password hashing, session management, and token validation — and gets it wrong in different ways
- Adding SSO, social login, or MFA requires re-engineering each service
- Auditing (who did what, when) is scattered across codebases
- Compliance (SOC2, GDPR, HIPAA) becomes a nightmare

IAM centralizes this into a dedicated system that every service trusts.

## Authentication vs Authorization

| Concept | Question | Example | Failure Mode |
|---|---|---|---|
| **Authentication** | Who are you? | User provides email + password | Wrong password → 401 Unauthorized |
| **Authorization** | What can you do? | User has `ROLE_ADMIN` | Missing role → 403 Forbidden |

These are distinct. You can be authenticated but not authorized (valid user, wrong role). You cannot be authorized without being authenticated.

**Common mistake**: Conflating the two. A 401 means "you need to authenticate first." A 403 means "I know who you are but you can't do this." Getting these wrong confuses clients and leaks information.

## Core Concepts

### Users
An entity that can authenticate. Can be a human, a service account, or a bot. Has credentials (password, certificate, etc.) and attributes (email, name, department).

### Roles
Named permissions or capabilities. Roles abstract "what you can do" from "who you are."
- **Coarse-grained**: `ROLE_ADMIN`, `ROLE_USER` — simple, but inflexible
- **Fine-grained**: `order:read`, `order:write`, `user:delete` — more expressive

### Groups
Collections of users. Roles are typically assigned to groups, not individual users. A user in the `Finance` group inherits all roles assigned to `Finance`. Groups simplify management at scale (add/remove user from group, not individually assign 20 roles).

### Permissions
The actual access decision: can subject X perform action Y on resource Z? Permissions can be modeled as role assignments (simple) or as policy-based access control (PBAC/ABAC — attribute-based, far more flexible but complex).

### Tokens
Cryptographically signed artifacts that prove identity and carry claims (user ID, roles, expiry). Tokens allow **stateless** authorization — services validate the token signature without calling the auth server on every request.

## Protocols

### OAuth 2.0

OAuth 2.0 is an **authorization framework** (RFC 6749). It defines how a client can obtain limited access to a resource on behalf of a user **without sharing credentials**.

Key roles in OAuth 2.0:
- **Resource Owner**: The user who owns the data
- **Client**: The application requesting access (your Spring Boot app)
- **Authorization Server**: Issues tokens (Keycloak)
- **Resource Server**: Holds protected resources (your API)

OAuth 2.0 defines four grant types (flows) for different scenarios. It is **not** an authentication protocol — it says nothing about who the user is, only that they authorized access.

### OpenID Connect (OIDC)

OIDC is an **authentication layer on top of OAuth 2.0** (adds an identity layer). It standardizes:
- The **ID Token** (a JWT containing user identity claims: `sub`, `email`, `name`)
- The `/userinfo` endpoint
- Standard scopes: `openid`, `profile`, `email`

If OAuth 2.0 answers "this user allowed the app to access their files," OIDC additionally answers "and here is who that user is."

**This is what Keycloak primarily speaks.** When you secure a Spring Boot API with Keycloak, you're using OIDC.

### SAML 2.0

Security Assertion Markup Language. XML-based federation protocol, predates OAuth. Still dominant in enterprise environments (Active Directory Federation Services, legacy HR systems).

- Verbose XML assertions vs compact JWTs
- Browser-based flows (POST/Redirect bindings)
- Strong in enterprise SSO scenarios
- Keycloak supports SAML as both SP (Service Provider) and IdP (Identity Provider)

**When you encounter SAML**: Integrating with enterprise identity providers (Microsoft AD FS, Okta, PingFederate) that don't support OIDC.

## IAM System vs Building Your Own Auth

| Factor | Build Your Own | Use Keycloak |
|---|---|---|
| Time to production | Weeks to months | Hours to days |
| Security risk | High (easy to make subtle mistakes) | Lower (battle-tested) |
| Features | Only what you build | SSO, MFA, social login, LDAP, SAML out of the box |
| Customization | Total | SPI-based extensions |
| Maintenance burden | Yours forever | Community + Red Hat |
| Compliance | You prove it | Easier to certify |

**Build your own only when**: You have extreme performance requirements where Keycloak's overhead is unacceptable, you need features fundamentally incompatible with Keycloak's model, or you're a security product company where auth is core IP.

**For 99% of backend systems**: Use Keycloak (self-hosted) or a managed service (Auth0, Okta). The cost of building secure auth correctly — password hashing, brute-force protection, MFA, token rotation, session management — is vastly underestimated.

---

# 2. Keycloak Fundamentals

## What Keycloak Is

Keycloak is an **open-source Identity and Access Management server** built by Red Hat. It is not just an auth server — it is a complete identity platform:

- **Authentication server**: Handles login, MFA, social login
- **Authorization server**: Issues OAuth 2.0 / OIDC tokens
- **Identity broker**: Federates external identity providers (Google, GitHub, SAML IdPs)
- **User federation**: Syncs users from LDAP/Active Directory
- **Session manager**: Tracks active sessions, supports SSO logout
- **Admin console**: Web UI for managing realms, clients, users, roles
- **Token service**: Issues, validates, and revokes JWTs

Technically, Keycloak runs as a standalone Java application (Quarkus-based since 17.x) with its own database (PostgreSQL in production).

## Core Features

### Single Sign-On (SSO)

Once authenticated to Keycloak, the user is authenticated across **all applications** in the same realm without re-entering credentials.

**How it works**: Keycloak maintains a session cookie (`KEYCLOAK_SESSION`) in the browser. When a user navigates to a second app, that app redirects to Keycloak, which sees the session cookie and issues a token without prompting for login.

**Real value**: In an enterprise with 20 internal apps, users log in once in the morning, use all apps throughout the day.

### Identity Brokering

Keycloak acts as a middleman between your apps and external identity providers (IdPs):

```
Your App ──► Keycloak ──► Google/GitHub/Corporate SAML IdP
                │
                └── Maps external identity to a local Keycloak user
```

Users can log in with "Sign in with Google" without your app ever talking to Google directly. Keycloak handles the OAuth dance with Google and gives your app a standard OIDC token.

### User Federation

Keycloak can read users from external sources (LDAP, Active Directory) without migrating them:

- Users stay in their existing directory
- Keycloak queries LDAP for authentication
- Attributes sync into Keycloak
- Group/role mappings from LDAP

Critical for enterprise deployments where IT already manages Active Directory.

## When to Use vs NOT Use Keycloak

### Use Keycloak When:
- Building multi-service/microservice systems needing centralized auth
- You need SSO across multiple apps
- Enterprise users need LDAP/AD integration
- You need MFA without building it yourself
- Compliance requires auditable auth logs
- You need social login (Google, GitHub, etc.)
- Your team will maintain the auth system long-term

### Do NOT Use Keycloak When:
- Simple single-service app with few users (overhead not worth it)
- Your team can't operate another stateful service in production
- You need sub-millisecond auth overhead (Keycloak adds ~5–50ms per token validation via introspection; use local JWT validation instead)
- Mobile app only (managed services like Firebase Auth or Auth0 are simpler)
- You're already on AWS and Cognito meets your needs (avoid vendor lock-in concern)

---

# 3. Keycloak Architecture & How It Works

## Realms

A **realm** is a security domain — an isolated namespace that manages its own:
- Users
- Clients
- Roles
- Groups
- Authentication flows
- Token settings
- Identity providers

Realms are completely isolated from each other. A user in realm `company-a` has no relationship to a user in realm `company-b`.

```
Keycloak Server
├── master realm (admin only — never use for apps)
├── production realm
│   ├── Users: alice, bob, service-account-payment
│   ├── Clients: frontend-app, backend-api, payment-service
│   └── Roles: ADMIN, USER, PAYMENT_PROCESSOR
└── staging realm
    └── (separate everything)
```

**The `master` realm**: Used to administer Keycloak itself. **Never** use the master realm for your applications. Create a dedicated realm for each environment or tenant.

## Clients

A **client** is an application that interacts with Keycloak. Every application that wants to authenticate users or verify tokens must be registered as a client.

### Client Types

| Type | Description | Use Case |
|---|---|---|
| **Public** | No client secret (cannot keep a secret) | SPAs, mobile apps |
| **Confidential** | Has client secret | Backend services, server-side apps |
| **Bearer-only** | Accepts tokens but doesn't initiate login | Resource servers (APIs) |

### Client Settings (Critical)

- **Valid Redirect URIs**: Where Keycloak may redirect after login. **Wildcards in production are a security vulnerability** — use exact URIs.
- **Web Origins**: CORS allowed origins for token endpoints.
- **Access Type**: Public vs Confidential.
- **Service Account Enabled**: Allows client credentials flow (machine-to-machine).

## Roles

### Realm Roles
Defined at the realm level, available across all clients.
```
Realm: production
Roles: REALM_ADMIN, REALM_USER, REALM_AUDITOR
```

### Client Roles
Defined for a specific client, scoped to that client's functionality.
```
Client: order-service
Roles: ORDER_READ, ORDER_WRITE, ORDER_CANCEL
```

**Which to use**: Client roles for application-specific permissions. Realm roles for cross-cutting concerns (admin access, global flags). Prefer client roles — they're more explicit and less likely to be accidentally granted.

### Composite Roles
A role that contains other roles. Assigning `SUPER_ADMIN` can automatically include `REALM_ADMIN`, `ORDER_WRITE`, etc. Use sparingly — can become hard to audit.

## Groups

Groups are hierarchical containers of users. Roles are assigned to groups; users inherit group roles.

```
Groups:
└── Engineering
    ├── Roles: [CODE_DEPLOY, METRICS_VIEW]
    ├── Backend
    │   └── Roles: [DB_ACCESS]
    └── Frontend
        └── Roles: [CDN_INVALIDATE]
```

A user in `Engineering/Backend` has: `CODE_DEPLOY`, `METRICS_VIEW`, `DB_ACCESS`.

**Group vs Role**: Groups are for organizing users. Roles define permissions. Don't put business logic in group names; use roles for that.

## Sessions

Keycloak maintains two session types:

1. **SSO Session (User Session)**: Created when user logs in. Tied to browser session via cookie. Has a configurable lifespan (default: 30 minutes idle, 10 hours max).

2. **Client Session**: Created per client per SSO session. Tracks which clients a user has active sessions with (for SSO logout).

Sessions are stored in Keycloak's database (or Infinispan cache in clustered mode). Session data enables:
- SSO (see existing session, skip login)
- Token refresh (validate that session still exists before issuing new access token)
- Global logout (invalidate all client sessions when user logs out)

## Tokens

### Access Token
A short-lived JWT (default: 5 minutes) that proves the bearer's identity and permissions. Sent in the `Authorization: Bearer <token>` header. Resource servers validate this **locally** using Keycloak's public key (no network call required).

**Access token claims (typical)**:
```json
{
  "sub": "a1b2c3d4-...",          // User ID (UUID)
  "iss": "https://keycloak.example.com/realms/production",
  "aud": "order-service",          // Intended audience
  "exp": 1705312800,               // Expiry timestamp
  "iat": 1705312500,               // Issued at
  "jti": "unique-token-id",        // JWT ID (for revocation)
  "typ": "Bearer",
  "azp": "frontend-app",           // Authorized party (client that requested)
  "session_state": "abc...",
  "realm_access": {
    "roles": ["REALM_USER"]
  },
  "resource_access": {
    "order-service": {
      "roles": ["ORDER_READ", "ORDER_WRITE"]
    }
  },
  "email": "alice@example.com",
  "preferred_username": "alice",
  "name": "Alice Smith"
}
```

### Refresh Token
A longer-lived token (default: 30 minutes) used to obtain new access tokens without re-authentication. Sent to the token endpoint, Keycloak validates the associated session is still active and issues a new access token.

**Refresh token rotation**: Keycloak can be configured to issue a new refresh token on each use (invalidating the old one). Prevents replay attacks on stolen refresh tokens.

### ID Token
An OIDC-specific JWT containing user identity information (who the user is, not what they can do). Intended for the **client application**, not the resource server. Used to display user info in the UI (name, email, avatar).

**Never send the ID token to your API.** That's what the access token is for.

## Token Lifecycle

```
1. User logs in → Keycloak issues:
   ├── Access Token (5 min TTL)
   ├── Refresh Token (30 min TTL)
   └── ID Token (5 min TTL)

2. Client uses Access Token for API calls
   └── Token expires → client uses Refresh Token

3. Client sends Refresh Token to /token endpoint
   ├── If session still active → new Access Token issued
   └── If session expired/revoked → 401, user must re-login

4. User logs out → SSO session invalidated
   └── Refresh tokens for all clients in session become invalid
```

## Internal Flow of Authentication

```
1. Browser → App: user clicks "Login"
2. App → Browser: redirect to Keycloak /auth endpoint
3. Browser → Keycloak: GET /auth?client_id=frontend&redirect_uri=...&scope=openid
4. Keycloak: render login page
5. User: enter credentials
6. Keycloak: validate credentials
   ├── Check brute force protection
   ├── Verify password hash (Bcrypt)
   └── Run authentication flow (MFA if configured)
7. Keycloak → Browser: redirect to redirect_uri with authorization code
8. Browser → App: deliver code
9. App (backend) → Keycloak: POST /token with code + client_secret
10. Keycloak → App: Access Token + Refresh Token + ID Token
11. App: validate tokens, create session, set cookie
12. App → Browser: authenticated response
```

---

# 4. Authentication & Authorization Flows

## Authorization Code Flow (Most Important)

**When to use**: Web applications with a backend component, mobile apps with PKCE. This is the **recommended flow** for almost every scenario.

**Why it's secure**: The access token is never exposed to the browser. Only a short-lived, single-use authorization code travels through the browser (the insecure channel). The actual token exchange happens server-to-server.

### Step-by-Step Flow

```
Browser        Frontend/App         Keycloak          Resource API
   │                │                   │                    │
   │─── click ──────►│                   │                    │
   │                │──── redirect ─────►│                    │
   │                │   ?client_id=...   │                    │
   │                │   &redirect_uri=...│                    │
   │                │   &scope=openid    │                    │
   │                │   &state=xyz       │                    │
   │                │   &code_challenge= │ (PKCE only)        │
   │◄─ redirect ────│                   │                    │
   │────────────────────────────────────►│                    │
   │              Keycloak login page    │                    │
   │◄────────────────────────────────────│                    │
   │─── credentials ─────────────────────►│                   │
   │              Keycloak validates     │                    │
   │◄─── redirect to redirect_uri ───────│                    │
   │      ?code=AUTH_CODE&state=xyz      │                    │
   │─── deliver code ──►│               │                    │
   │                │   │               │                    │
   │                │── POST /token ────►│                    │
   │                │   code=AUTH_CODE   │                    │
   │                │   client_id=...    │                    │
   │                │   client_secret=.. │                    │
   │                │   code_verifier=.. │ (PKCE only)        │
   │                │◄── tokens ─────────│                    │
   │                │                   │                    │
   │                │─────── API call ─────────────────────────►
   │                │   Authorization: Bearer <access_token>  │
   │                │◄───────── response ──────────────────────│
```

### PKCE (Proof Key for Code Exchange)

PKCE is an extension for public clients (SPAs, mobile apps) that can't keep a client secret.

```
1. Client generates code_verifier (random 43–128 char string)
2. Client computes code_challenge = BASE64URL(SHA256(code_verifier))
3. code_challenge sent in auth request
4. code_verifier sent in token exchange
5. Keycloak verifies: SHA256(code_verifier) == stored code_challenge
```

This prevents authorization code interception attacks. Even if an attacker intercepts the code, they can't exchange it without the `code_verifier`.

**Security considerations**:
- Always validate the `state` parameter (CSRF protection)
- `redirect_uri` must match exactly (no wildcards in production)
- Use short-lived authorization codes (default: 60 seconds in Keycloak)
- Require PKCE for all public clients

---

## Client Credentials Flow

**When to use**: Machine-to-machine (M2M) communication. Service A needs to call Service B without a user involved. Background jobs, microservice internal calls, scheduled tasks.

**No user interaction**: The client authenticates as itself, not on behalf of a user.

### Step-by-Step Flow

```
Service A                    Keycloak                   Service B
    │                            │                           │
    │── POST /token ─────────────►│                           │
    │   grant_type=client_credentials                        │
    │   client_id=payment-service                            │
    │   client_secret=<secret>    │                           │
    │◄── Access Token ────────────│                           │
    │                            │                           │
    │─── API call ────────────────────────────────────────────►
    │    Authorization: Bearer <token>                       │
    │◄─── response ───────────────────────────────────────────│
```

```bash
# Token request
curl -X POST https://keycloak.example.com/realms/production/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=payment-service" \
  -d "client_secret=your-client-secret"
```

**The token contains**:
- `sub` = service account user ID (not a human user)
- `azp` = client ID
- Roles assigned to the service account

**Security considerations**:
- Store client secrets in Vault/Kubernetes Secrets, never in code or config files
- Use short token TTLs (1–5 minutes) for M2M tokens
- Rotate client secrets regularly
- Assign only the minimum roles needed (principle of least privilege)
- Prefer client-scoped roles over realm roles for M2M

---

## Implicit Flow (Discouraged)

**Historical context**: Designed for SPAs before CORS was widely supported. Tokens are returned directly in the URL fragment — no code exchange step.

```
Browser → Keycloak: /auth?response_type=token&...
Keycloak → Browser: redirect to redirect_uri#access_token=...
```

**Why it's discouraged** (do not use for new systems):
1. Access token exposed in browser history, server logs, Referer headers
2. No refresh token support
3. No way to verify the token hasn't been tampered with in transit
4. Removed from OAuth 2.1 spec

**What to use instead**: Authorization Code + PKCE for SPAs. This was previously avoided due to CORS, but Keycloak's token endpoint supports CORS.

---

## Resource Owner Password Credentials (ROPC) Flow

**What it does**: Client collects username and password directly and sends them to the token endpoint.

```
Mobile App → Keycloak: POST /token
  grant_type=password
  username=alice
  password=secret
  client_id=mobile-app
```

**Why to avoid**:
1. Application handles the user's raw credentials — violates the principle of credential delegation
2. Breaks SSO (no browser redirect = no SSO session cookie)
3. Doesn't support MFA
4. Credentials can be logged by accident
5. Deprecated in OAuth 2.1

**Legitimate use cases** (rare): Migration scenarios where you need to validate existing credentials against a new auth system. **Even then, migrate to Authorization Code Flow as soon as possible.**

**What teams do wrong**: Using ROPC because it "feels simpler" for mobile apps. Use Authorization Code + PKCE — modern mobile OAuth libraries make this straightforward.

---

# 5. Keycloak in System Design

## Microservices Authentication

```
                    ┌─────────────────────┐
                    │      Keycloak        │
                    │  (Auth Server)       │
                    └──────────┬──────────┘
                               │ Issues JWTs
                               │ (public key available at /certs)
          ┌────────────────────┼────────────────────┐
          │                    │                    │
    ┌─────▼──────┐    ┌────────▼───────┐    ┌──────▼──────┐
    │  API       │    │  Order         │    │  Payment    │
    │  Gateway   │    │  Service       │    │  Service    │
    └─────┬──────┘    └────────────────┘    └─────────────┘
          │           Each service validates JWT locally
          │           using Keycloak's public key
    ┌─────▼──────┐    (no call to Keycloak per request)
    │  Frontend  │
    └────────────┘
```

**Token validation strategy**: Each microservice downloads Keycloak's public key from `/.well-known/openid-configuration` → `jwks_uri` and validates JWT signatures locally. Zero network calls to Keycloak per request.

**Key rotation**: Keycloak periodically rotates signing keys. The JWKS endpoint always serves current and recent keys. Spring Security's `JwtDecoder` automatically fetches new keys when it encounters an unknown `kid` (key ID) in a token header.

## API Gateway + Keycloak

```
Client
  │
  ▼
API Gateway (Kong / AWS API GW / Nginx)
  │
  ├── Validate Bearer token (JWT validation at gateway)
  │   └── Reject invalid/expired tokens before they reach services
  │
  ├── Route to Order Service
  ├── Route to Payment Service
  └── Route to User Service
           │
           Each service trusts gateway-validated requests
           (or re-validates for defense in depth)
```

**Trade-off — validate at gateway vs each service**:

| Approach | Pros | Cons |
|---|---|---|
| Gateway only | One place to update validation logic | If gateway is bypassed, services are unprotected |
| Each service | Defense in depth, services safe even if gateway misconfigured | Duplicate logic, slower per-service startup (JWKS fetch) |
| Both | Maximum security | Slight overhead |

**Recommendation**: Validate at the gateway for authentication (is the token valid?). Let each service handle authorization (does this user have the right role for this operation?). Never skip service-level authorization checks.

## Frontend + Backend Auth Flow

```
SPA (React/Angular)              Backend API             Keycloak
        │                             │                       │
        │── redirect to login ────────────────────────────────►
        │                             │                       │
        │◄── auth code ───────────────────────────────────────
        │                             │                       │
        │── POST /api/auth/callback ──►                       │
        │   code=AUTH_CODE            │                       │
        │                             │── POST /token ─────────►
        │                             │   code + secret        │
        │                             │◄── tokens ─────────────
        │                             │                       │
        │◄── Set HttpOnly cookie ─────│                       │
        │    (stores refresh token)   │                       │
        │                             │                       │
        │── API requests (cookie) ────►                       │
        │                             │ validates access token │
        │◄── response ────────────────│                       │
```

**Why HttpOnly cookie for refresh token**: The refresh token is the most sensitive credential (long-lived). Storing it in `localStorage` exposes it to XSS attacks. An HttpOnly cookie is inaccessible to JavaScript.

**Access token storage options**:
- **In-memory (JS variable)**: Most secure (not accessible to XSS, not persisted). Lost on page refresh — need to re-fetch from backend or Keycloak on each page load.
- **SessionStorage**: Lost on tab close. Vulnerable to XSS but not persisted.
- **Backend session**: Backend holds tokens, SPA gets a session cookie. Most secure for SPAs.

## SSO Across Multiple Services

```
Service A          Service B         Keycloak
    │                   │                │
    │  User at A ──────►│                │
    │  redirects to login ───────────────►
    │                   │    Login page  │
    │                   │  User logs in  │
    │◄──── auth code ────────────────────│
    │  exchanges code   │     Session    │
    │  gets tokens      │    Created     │
    │                   │                │
    │  User navigates to B               │
    │                   │── redirect ────►
    │                   │   to Keycloak  │
    │                   │◄── Sees SSO ───│
    │                   │    session     │
    │                   │    cookie      │
    │                   │◄── auth code ──│
    │                   │   (no login    │
    │                   │    prompt!)    │
    │                   │ exchanges code │
```

SSO session is maintained by Keycloak via a browser session cookie. All apps in the same realm participate automatically.

**SSO Logout (Critical — often implemented incorrectly)**:

Keycloak supports two logout mechanisms:
1. **Front-channel logout**: Keycloak redirects browser to each client's logout URI. Simpler but fails if browser is closed.
2. **Back-channel logout**: Keycloak POSTs a logout token to each client's `backchannel-logout-uri`. More reliable — doesn't require browser involvement.

Always implement back-channel logout to ensure sessions are cleaned up even when the browser is closed.

## Service-to-Service Authentication

```
Order Service                 Payment Service              Keycloak
      │                              │                         │
      │── POST /token ───────────────────────────────────────►│
      │   client_credentials         │                         │
      │◄── access_token ─────────────────────────────────────│
      │                              │                         │
      │── POST /payments ────────────►                         │
      │   Authorization: Bearer token│                         │
      │                              │ validates JWT locally   │
      │◄── payment result ───────────│                         │
```

**Token caching**: The Order Service should cache the access token until it's about to expire (subtract 30 seconds from expiry), then fetch a new one. Fetching a new token per request is wasteful.

**Failure scenario**: Keycloak is down. Order Service can't get a token. Payment Service calls fail. **Mitigation**: Token caching (services continue working with cached tokens until TTL expires), circuit breaker pattern, Keycloak HA setup.

---

# 6. Production Best Practices

## Realm Design Strategies

### One Realm Per Environment (Minimum)

```
keycloak-server
├── master (admin only)
├── myapp-production
├── myapp-staging
└── myapp-development
```

Never share a realm between environments. Token issuer URLs differ, preventing accidental cross-environment token acceptance.

### Multi-Tenant: One Realm Per Tenant vs Shared Realm

| Strategy | Pros | Cons |
|---|---|---|
| One realm per tenant | Complete isolation, custom flows per tenant | Hard to manage at scale (100+ tenants) |
| Shared realm with groups | Simple management | Less isolation, custom flows harder |
| Hybrid (realm per major tenant) | Balance | Operational complexity |

For SaaS with <50 enterprise tenants: one realm per tenant. For B2C with thousands of users: shared realm with organization groups.

## Role vs Group Modeling

**Principle**: Assign roles to groups, add users to groups. Never assign roles directly to individual users (except service accounts).

```
BAD:
  User Alice → ROLE_ORDER_READ, ROLE_ORDER_WRITE, ROLE_INVENTORY_READ

GOOD:
  Group: Order Managers → ROLE_ORDER_READ, ROLE_ORDER_WRITE, ROLE_INVENTORY_READ
  User Alice → Group: Order Managers
```

**Role naming convention**:
- Use `SCREAMING_SNAKE_CASE` for roles (Spring Security expects this by convention)
- Prefix client roles with the service name: `ORDER_READ`, `PAYMENT_PROCESS`
- Avoid generic names: `READ`, `WRITE` are ambiguous; `ORDER_READ` is not

## Token Size & Performance

JWTs are included in every API request header. Large tokens hurt performance:
- More data transmitted per request
- More time to parse and validate
- HTTP/2 header compression helps but doesn't solve the problem

**What makes tokens large**:
- Many realm and client roles
- Custom mappers adding large claim values
- Long username/email/name values

**Strategies to reduce token size**:
1. **Scope filtering**: Only include roles for the specific audience. Configure `Full Scope Allowed: false` on clients and add specific scope mappings.
2. **Audience restriction**: Use `aud` mapper to limit which services accept the token.
3. **Move profile data to `/userinfo`**: Don't include large profile attributes in access tokens.
4. **Use client scopes**: Clients request only the scopes they need.

## Token Expiration Strategies

| Token | Recommended TTL | Reasoning |
|---|---|---|
| Access Token | 5 minutes | Short enough that stolen tokens expire quickly |
| Refresh Token | 30 minutes (idle) | Balance UX vs security |
| SSO Session | 8–10 hours (max) | Work day, force re-auth overnight |
| Remember Me Session | 30 days | Explicit user choice |

**For high-security APIs** (financial, healthcare): 1-minute access token, 5-minute refresh token. Force re-auth frequently.

**For consumer apps**: 15-minute access token, 24-hour refresh token, 30-day session.

## Securing Endpoints

```
# Keycloak token introspection (for opaque tokens — slower, not recommended)
POST /realms/{realm}/protocol/openid-connect/token/introspect

# Better: Local JWT validation using JWKS
GET /realms/{realm}/protocol/openid-connect/certs
```

**Always use local JWT validation** for resource servers. Token introspection requires a network call to Keycloak for every request — adds 5–50ms latency and creates a hard dependency on Keycloak availability.

**Configure audience validation**: A token issued for `frontend-app` should not be accepted by `payment-service`. Configure audience (`aud`) in the token and validate it in your resource server.

## Handling Logout Properly

The most commonly botched part of Keycloak integration.

### What logout must do:
1. Invalidate the SSO session in Keycloak (back-channel logout)
2. Invalidate refresh token
3. Clear client-side tokens (localStorage, memory)
4. Clear HttpOnly cookie if used

### What logout does NOT do:
- Invalidate already-issued access tokens (they're stateless JWTs)

**The access token problem**: If access tokens have a 5-minute TTL and a user logs out, that access token is valid for up to 5 more minutes. For most systems, this is acceptable. For high-security systems:
- Use very short access token TTL (1 minute)
- Implement token revocation list (check token `jti` against a blocklist in Redis — but this sacrifices statelessness)

### Correct logout flow:

```java
// 1. Redirect user to Keycloak logout endpoint
String logoutUrl = keycloakUrl + "/realms/" + realm + 
    "/protocol/openid-connect/logout" +
    "?post_logout_redirect_uri=" + URLEncoder.encode(appUrl) +
    "&id_token_hint=" + idToken;  // id_token_hint helps Keycloak identify the session
response.sendRedirect(logoutUrl);

// 2. Keycloak back-channel calls your /logout endpoint for other clients in SSO
// 3. Your app clears cookies and local state
```

## Avoiding Common Misconfigurations

| Misconfiguration | Risk | Fix |
|---|---|---|
| Wildcard redirect URIs (`*`) | Open redirect, code injection | Use exact URIs |
| `Full Scope Allowed: true` | Token bloat, over-privilege | Disable, configure explicit scopes |
| No audience validation | Any client can use any token | Add `aud` mapper, validate in resource server |
| Long access token TTL (hours) | Stolen tokens valid for hours | 5 minutes max |
| No brute force protection | Credential stuffing | Enable in Realm Settings → Security Defenses |
| HTTP in production | Token interception | HTTPS everywhere |
| Master realm for apps | Blast radius if compromised | Use dedicated realm |
| Storing client secret in code | Secret exposure | Vault, K8s secrets, env vars |
| Not validating token signature | Forged tokens accepted | Always verify signature |

## Scaling Keycloak

### Horizontal Scaling

Keycloak supports clustering via Infinispan (distributed cache) for:
- SSO session replication
- Authentication session replication
- JWKS cache

```yaml
# Docker Compose cluster setup
keycloak-1:
  environment:
    KC_CACHE: ispn
    KC_CACHE_STACK: kubernetes  # or tcp, udp, jdbc-ping
    JGROUPS_DISCOVERY_PROTOCOL: JDBC_PING  # Database-based discovery

keycloak-2:
  # Same config — nodes discover each other via DB
```

**In Kubernetes**: Use `KC_CACHE_STACK=kubernetes` with Infinispan Kubernetes discovery. Keycloak nodes find each other via Kubernetes API.

### Database: Always PostgreSQL in Production

```
# Keycloak supports:
# - PostgreSQL ✓ (recommended)
# - MySQL/MariaDB ✓
# - Oracle ✓
# - Microsoft SQL Server ✓
# - H2 ✗ (embedded, development only)
```

**Database sizing**: Keycloak's DB holds users, sessions, events, and configuration. For 100k users with active sessions: ~5–10GB. Index on `USER_ENTITY.USERNAME` and `USER_ENTITY.EMAIL` — these are hot lookup paths.

## High Availability Setup

```
                    ┌─────────────┐
                    │  Load       │
                    │  Balancer   │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼───┐  ┌─────▼───┐  ┌───▼─────┐
        │Keycloak │  │Keycloak │  │Keycloak │
        │Node 1   │  │Node 2   │  │Node 3   │
        └─────────┘  └─────────┘  └─────────┘
              │            │            │
              └────────────┼────────────┘
                           │
                    ┌──────▼──────┐
                    │ PostgreSQL  │
                    │ (Primary +  │
                    │  Replica)   │
                    └─────────────┘
```

**Load balancer requirements**:
- **Sticky sessions NOT required** for Keycloak cluster (sessions are replicated via Infinispan)
- Use least-connections or round-robin
- Health check: `GET /health/ready`

**Minimum HA config**:
- 3 Keycloak nodes (odd number for quorum)
- PostgreSQL with replica (for Keycloak's DB)
- Load balancer with health checks

## Backup Strategies

```bash
# 1. Database backup (primary backup)
pg_dump -h keycloak-db -U keycloak keycloak_db > keycloak_backup_$(date +%Y%m%d).sql

# 2. Realm export (Keycloak-level, includes all config but not passwords)
/opt/keycloak/bin/kc.sh export \
  --dir /backup/realms \
  --realm production \
  --users realm_file  # includes user data

# 3. Import
/opt/keycloak/bin/kc.sh import --dir /backup/realms
```

**What realm export includes**: Clients, roles, groups, flows, identity providers, realm settings, users (with hashed credentials if `--users realm_file`).

**What it doesn't preserve**: Active sessions (users must re-login after restore).

**Backup frequency**: Database: continuous (WAL archiving) or hourly snapshots. Realm export: daily, before any configuration changes.

---

# 7. Keycloak + Spring Boot

## Running Keycloak with Docker

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: keycloak_pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

  keycloak:
    image: quay.io/keycloak/keycloak:23.0.4
    command: start-dev  # Use 'start' for production
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: keycloak_pass
      KC_HOSTNAME: localhost
      KC_HTTP_PORT: 8080
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin_pass
    ports:
      - "8080:8080"
    depends_on:
      - postgres

volumes:
  postgres_data:
```

```bash
docker-compose up -d
# Admin console: http://localhost:8080/admin
```

## Realm and Client Setup

### Create Realm via Keycloak CLI

```bash
# Export admin token
TOKEN=$(curl -s -X POST http://localhost:8080/realms/master/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=admin-cli&username=admin&password=admin_pass&grant_type=password" \
  | jq -r '.access_token')

# Create realm
curl -X POST http://localhost:8080/admin/realms \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "realm": "myapp",
    "enabled": true,
    "accessTokenLifespan": 300,
    "ssoSessionIdleTimeout": 1800,
    "ssoSessionMaxLifespan": 36000
  }'

# Create client (confidential, for backend API)
curl -X POST http://localhost:8080/admin/realms/myapp/clients \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "clientId": "order-service",
    "enabled": true,
    "clientAuthenticatorType": "client-secret",
    "bearerOnly": true,
    "publicClient": false
  }'

# Create client (public, for frontend)
curl -X POST http://localhost:8080/admin/realms/myapp/clients \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "clientId": "frontend-app",
    "enabled": true,
    "publicClient": true,
    "redirectUris": ["http://localhost:3000/*"],
    "webOrigins": ["http://localhost:3000"],
    "standardFlowEnabled": true
  }'
```

### Realm Export for Version Control

```json
// realm-export.json — commit this to version control
{
  "realm": "myapp",
  "enabled": true,
  "accessTokenLifespan": 300,
  "clients": [
    {
      "clientId": "order-service",
      "bearerOnly": true,
      "enabled": true
    }
  ],
  "roles": {
    "realm": [
      { "name": "REALM_USER" },
      { "name": "REALM_ADMIN" }
    ]
  }
}
```

---

## Example 1: Securing REST APIs with Keycloak

### Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>
    <!-- For JWT decoding -->
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-oauth2-jose</artifactId>
    </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # Keycloak's JWKS endpoint — Spring fetches public keys automatically
          jwk-set-uri: http://localhost:8080/realms/myapp/protocol/openid-connect/certs
          # Alternatively, use issuer-uri for full OIDC discovery:
          # issuer-uri: http://localhost:8080/realms/myapp

keycloak:
  realm: myapp
  server-url: http://localhost:8080
  client-id: order-service

app:
  security:
    allowed-public-paths:
      - /actuator/health
      - /api/v1/public/**
```

### Security Configuration

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    @Value("${spring.security.oauth2.resourceserver.jwt.jwk-set-uri}")
    private String jwkSetUri;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)  // Stateless API — no CSRF needed
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/api/v1/public/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/orders/**").hasAnyRole("USER", "ADMIN")
                .requestMatchers(HttpMethod.POST, "/api/v1/orders/**").hasRole("USER")
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .decoder(jwtDecoder())
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            );

        return http.build();
    }

    @Bean
    public JwtDecoder jwtDecoder() {
        NimbusJwtDecoder decoder = NimbusJwtDecoder
            .withJwkSetUri(jwkSetUri)
            .build();

        // Validate issuer
        OAuth2TokenValidator<Jwt> issuerValidator = JwtValidators.createDefaultWithIssuer(
            "http://localhost:8080/realms/myapp"
        );
        
        // Validate audience
        OAuth2TokenValidator<Jwt> audienceValidator = new JwtClaimValidator<List<String>>(
            JwtClaimNames.AUD,
            aud -> aud != null && aud.contains("order-service")
        );
        
        OAuth2TokenValidator<Jwt> validators = new DelegatingOAuth2TokenValidator<>(
            issuerValidator,
            audienceValidator
        );
        
        decoder.setJwtValidator(validators);
        return decoder;
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter converter = new JwtGrantedAuthoritiesConverter();
        
        // Tell Spring Security where to find roles in the JWT
        // For realm roles: use "realm_access.roles"
        // For client roles: use "resource_access.order-service.roles"
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            List<GrantedAuthority> authorities = new ArrayList<>();
            
            // Extract realm roles
            Map<String, Object> realmAccess = jwt.getClaim("realm_access");
            if (realmAccess != null) {
                List<String> realmRoles = (List<String>) realmAccess.get("roles");
                if (realmRoles != null) {
                    realmRoles.stream()
                        .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                        .forEach(authorities::add);
                }
            }
            
            // Extract client-specific roles
            Map<String, Object> resourceAccess = jwt.getClaim("resource_access");
            if (resourceAccess != null) {
                Map<String, Object> clientAccess = 
                    (Map<String, Object>) resourceAccess.get("order-service");
                if (clientAccess != null) {
                    List<String> clientRoles = (List<String>) clientAccess.get("roles");
                    if (clientRoles != null) {
                        clientRoles.stream()
                            .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                            .forEach(authorities::add);
                    }
                }
            }
            
            return authorities;
        });
        
        return new JwtAuthenticationConverter() {{
            setJwtGrantedAuthoritiesConverter(converter.getJwtGrantedAuthoritiesConverter());
        }};
    }
}
```

---

## Example 2: JWT Validation & Security Context

```java
// Accessing current user from token in any component
@Component
public class SecurityContextHelper {

    public String getCurrentUserId() {
        return getJwt().getSubject();  // "sub" claim = Keycloak user UUID
    }

    public String getCurrentUsername() {
        return getJwt().getClaimAsString("preferred_username");
    }

    public String getCurrentEmail() {
        return getJwt().getClaimAsString("email");
    }

    public List<String> getCurrentRoles() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        return auth.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .filter(a -> a.startsWith("ROLE_"))
            .map(a -> a.substring(5))
            .toList();
    }

    public boolean hasRole(String role) {
        return SecurityContextHolder.getContext().getAuthentication()
            .getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_" + role));
    }

    private Jwt getJwt() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth instanceof JwtAuthenticationToken jwtAuth) {
            return jwtAuth.getToken();
        }
        throw new IllegalStateException("No JWT authentication in context");
    }
    
    // Get any custom claim
    public <T> T getClaim(String claimName, Class<T> type) {
        return getJwt().getClaim(claimName);
    }
}

// Usage in controller
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;
    private final SecurityContextHelper security;

    @GetMapping
    public List<OrderDto> getMyOrders() {
        // Current user ID from JWT — no need to pass userId as parameter
        String userId = security.getCurrentUserId();
        return orderService.getOrdersForUser(userId);
    }
    
    @GetMapping("/{orderId}")
    public OrderDto getOrder(@PathVariable Long orderId) {
        OrderDto order = orderService.getOrder(orderId);
        
        // Authorization: user can only see their own orders (unless admin)
        String currentUserId = security.getCurrentUserId();
        if (!order.userId().equals(currentUserId) && !security.hasRole("ADMIN")) {
            throw new AccessDeniedException("Not your order");
        }
        
        return order;
    }
}
```

---

## Example 3: Role-Based Authorization

```java
// URL-level authorization (in SecurityConfig)
.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.GET, "/api/v1/orders").hasAnyRole("USER", "ADMIN")
    .requestMatchers(HttpMethod.DELETE, "/api/v1/orders/**").hasRole("ADMIN")
    .requestMatchers("/api/v1/reports/**").hasRole("AUDITOR")
)

// Method-level authorization with @PreAuthorize
@Service
@RequiredArgsConstructor
public class OrderService {

    // Simple role check
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(Long orderId) {
        // Only admins reach this code
    }
    
    // Check specific permission
    @PreAuthorize("hasAuthority('ROLE_ORDER_WRITE')")
    public OrderDto createOrder(CreateOrderRequest request) {
        // Only users with ORDER_WRITE role
    }
    
    // Complex expression with JWT claims
    @PreAuthorize("hasRole('USER') and #userId == authentication.token.subject")
    public List<OrderDto> getUserOrders(String userId) {
        // User can only get their own orders
        // #userId refers to the method parameter
        // authentication.token.subject is the JWT 'sub' claim
    }
    
    // Custom security expression
    @PreAuthorize("@orderSecurityService.canViewOrder(authentication, #orderId)")
    public OrderDto getOrder(Long orderId) {
        return orderRepository.findById(orderId).map(this::toDto).orElseThrow();
    }
    
    @PostAuthorize("returnObject.userId == authentication.token.subject or hasRole('ADMIN')")
    public OrderDto getOrderById(Long orderId) {
        // @PostAuthorize: executes method THEN checks condition on return value
        // Useful when you need to check a property of the returned object
        return orderRepository.findById(orderId).map(this::toDto).orElseThrow();
    }
}

// Custom security service for complex authorization logic
@Service("orderSecurityService")
@RequiredArgsConstructor
public class OrderSecurityService {

    private final OrderRepository orderRepository;
    
    public boolean canViewOrder(Authentication auth, Long orderId) {
        if (auth == null || !auth.isAuthenticated()) return false;
        
        // Admins can see everything
        boolean isAdmin = auth.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
        if (isAdmin) return true;
        
        // Users can only see their own orders
        if (auth instanceof JwtAuthenticationToken jwtAuth) {
            String userId = jwtAuth.getToken().getSubject();
            return orderRepository.existsByIdAndUserId(orderId, userId);
        }
        
        return false;
    }
}
```

---

## Example 4: Method-Level Security

```java
// Enable in configuration
@Configuration
@EnableMethodSecurity(
    prePostEnabled = true,   // @PreAuthorize, @PostAuthorize
    securedEnabled = true,   // @Secured (older style)
    jsr250Enabled = true     // @RolesAllowed (JSR-250)
)
public class MethodSecurityConfig {}

// Controller with method security
@RestController
@RequestMapping("/api/v1/admin")
public class AdminController {

    @GetMapping("/users")
    @PreAuthorize("hasRole('ADMIN')")
    public List<UserDto> getAllUsers() { ... }
    
    @DeleteMapping("/users/{userId}")
    @PreAuthorize("hasRole('SUPER_ADMIN')")
    public void deleteUser(@PathVariable String userId) { ... }
    
    @GetMapping("/reports")
    @Secured({"ROLE_ADMIN", "ROLE_AUDITOR"})  // Either role
    public ReportDto getReport() { ... }
    
    @PostMapping("/config")
    @RolesAllowed("ADMIN")  // JSR-250 style
    public void updateConfig(@RequestBody ConfigDto config) { ... }
}

// Testing secured methods
@SpringBootTest
@WithMockUser(roles = {"ADMIN"})  // Mock Spring Security context for tests
class OrderServiceTest {

    @Test
    void deleteOrder_asAdmin_succeeds() {
        assertDoesNotThrow(() -> orderService.deleteOrder(1L));
    }
    
    @Test
    @WithMockUser(roles = {"USER"})
    void deleteOrder_asUser_throwsAccessDenied() {
        assertThrows(AccessDeniedException.class, () -> orderService.deleteOrder(1L));
    }
}

// For JWT-based tests
@TestConfiguration
public class TestSecurityConfig {
    
    public static SecurityMockMvcRequestPostProcessors.JwtRequestPostProcessor adminJwt() {
        return jwt()
            .jwt(builder -> builder
                .subject("test-admin-uuid")
                .claim("realm_access", Map.of("roles", List.of("ADMIN", "USER")))
                .claim("preferred_username", "test-admin")
                .claim("email", "admin@test.com")
            );
    }
}

// In MVC tests:
mockMvc.perform(delete("/api/v1/orders/1")
    .with(TestSecurityConfig.adminJwt()))
    .andExpect(status().isOk());
```

---

## Example 5: Service-to-Service Authentication (Client Credentials)

```java
// Configuration
@ConfigurationProperties(prefix = "app.keycloak")
@Data
public class KeycloakClientProperties {
    private String serverUrl;
    private String realm;
    private String clientId;
    private String clientSecret;
    private Duration tokenRefreshBuffer = Duration.ofSeconds(30);
}

// Token manager with caching
@Component
@RequiredArgsConstructor
@Slf4j
public class KeycloakTokenManager {

    private final KeycloakClientProperties properties;
    private final WebClient.Builder webClientBuilder;
    
    private volatile String cachedToken;
    private volatile Instant tokenExpiry = Instant.MIN;
    private final Object lock = new Object();
    
    public String getAccessToken() {
        // Fast path: token still valid
        if (isTokenValid()) {
            return cachedToken;
        }
        
        synchronized (lock) {
            // Double-checked locking
            if (isTokenValid()) {
                return cachedToken;
            }
            
            return fetchNewToken();
        }
    }
    
    private boolean isTokenValid() {
        return cachedToken != null && 
               Instant.now().isBefore(tokenExpiry.minus(properties.getTokenRefreshBuffer()));
    }
    
    private String fetchNewToken() {
        log.debug("Fetching new client credentials token");
        
        String tokenUrl = properties.getServerUrl() + 
            "/realms/" + properties.getRealm() + 
            "/protocol/openid-connect/token";
        
        MultiValueMap<String, String> formData = new LinkedMultiValueMap<>();
        formData.add("grant_type", "client_credentials");
        formData.add("client_id", properties.getClientId());
        formData.add("client_secret", properties.getClientSecret());
        
        TokenResponse response = webClientBuilder.build()
            .post()
            .uri(tokenUrl)
            .contentType(MediaType.APPLICATION_FORM_URLENCODED)
            .bodyValue(formData)
            .retrieve()
            .bodyToMono(TokenResponse.class)
            .block(Duration.ofSeconds(5));
        
        if (response == null) {
            throw new RuntimeException("Failed to obtain access token from Keycloak");
        }
        
        cachedToken = response.accessToken();
        tokenExpiry = Instant.now().plusSeconds(response.expiresIn());
        
        return cachedToken;
    }
    
    record TokenResponse(
        @JsonProperty("access_token") String accessToken,
        @JsonProperty("expires_in") long expiresIn,
        @JsonProperty("token_type") String tokenType
    ) {}
}

// WebClient with automatic token injection
@Configuration
@RequiredArgsConstructor
public class ServiceClientConfig {

    private final KeycloakTokenManager tokenManager;
    
    @Bean
    public WebClient paymentServiceClient(
            @Value("${app.services.payment-url}") String paymentServiceUrl) {
        
        return WebClient.builder()
            .baseUrl(paymentServiceUrl)
            .filter(ExchangeFilterFunction.ofRequestProcessor(request -> {
                String token = tokenManager.getAccessToken();
                return Mono.just(ClientRequest.from(request)
                    .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
                    .build());
            }))
            .filter(ExchangeFilterFunction.ofResponseProcessor(response -> {
                if (response.statusCode().value() == 401) {
                    // Token was rejected — force refresh on next call
                    tokenManager.invalidateToken();
                }
                return Mono.just(response);
            }))
            .build();
    }
}

// application.yml
```yaml
app:
  keycloak:
    server-url: http://keycloak:8080
    realm: myapp
    client-id: order-service
    client-secret: ${KEYCLOAK_CLIENT_SECRET}
    token-refresh-buffer: 30s
  services:
    payment-url: http://payment-service:8081
```

---

## Example 6: Custom Claims Handling

### Adding Custom Claims in Keycloak

1. **Protocol Mapper**: Keycloak GUI → Client → Client Scopes → Add Mapper
2. **User Attribute Mapper**: Maps a user attribute to a JWT claim
3. **Script Mapper** (Keycloak 18+ requires extension): Custom logic

```json
// Keycloak Protocol Mapper config (via Admin REST API)
{
  "name": "organization-id",
  "protocol": "openid-connect",
  "protocolMapper": "oidc-usermodel-attribute-mapper",
  "config": {
    "user.attribute": "organizationId",
    "claim.name": "org_id",
    "jsonType.label": "String",
    "id.token.claim": "false",
    "access.token.claim": "true",
    "userinfo.token.claim": "true"
  }
}
```

### Accessing Custom Claims in Spring Boot

```java
// Custom principal to represent authenticated user with all relevant claims
public record KeycloakPrincipal(
    String userId,
    String username,
    String email,
    String organizationId,
    String tenantId,
    List<String> roles,
    Jwt rawToken
) {
    public static KeycloakPrincipal from(JwtAuthenticationToken token) {
        Jwt jwt = token.getToken();
        
        List<String> roles = token.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .filter(a -> a.startsWith("ROLE_"))
            .map(a -> a.substring(5))
            .toList();
        
        return new KeycloakPrincipal(
            jwt.getSubject(),
            jwt.getClaimAsString("preferred_username"),
            jwt.getClaimAsString("email"),
            jwt.getClaimAsString("org_id"),   // custom claim
            jwt.getClaimAsString("tenant_id"), // custom claim
            roles,
            jwt
        );
    }
}

// Resolve KeycloakPrincipal from Spring Security context in any controller
@Component
public class KeycloakPrincipalArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.getParameterType().equals(KeycloakPrincipal.class);
    }
    
    @Override
    public Object resolveArgument(MethodParameter parameter, 
                                   ModelAndViewContainer mavContainer,
                                   NativeWebRequest webRequest, 
                                   WebDataBinderFactory binderFactory) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth instanceof JwtAuthenticationToken jwtAuth) {
            return KeycloakPrincipal.from(jwtAuth);
        }
        throw new IllegalStateException("Not a JWT authentication");
    }
}

// Register the resolver
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    
    @Bean
    public KeycloakPrincipalArgumentResolver keycloakPrincipalArgumentResolver() {
        return new KeycloakPrincipalArgumentResolver();
    }
    
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(keycloakPrincipalArgumentResolver());
    }
}

// Clean controller usage — no boilerplate
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    @GetMapping
    public List<OrderDto> getOrders(KeycloakPrincipal principal) {
        // principal.userId(), principal.organizationId() — clean and typed
        return orderService.getOrdersForOrg(principal.organizationId());
    }
    
    @PostMapping
    public OrderDto createOrder(@RequestBody CreateOrderRequest request,
                                 KeycloakPrincipal principal) {
        return orderService.createOrder(request, principal.userId(), principal.tenantId());
    }
}
```

### Keycloak User Attribute Setup

```bash
# Set custom attribute on user (Admin REST API)
curl -X PUT http://localhost:8080/admin/realms/myapp/users/{userId} \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "attributes": {
      "organizationId": ["org-123"],
      "tenantId": ["tenant-456"]
    }
  }'
```

---

# 8. Advanced Topics

## Custom Authentication Flows

Keycloak's authentication flow is a tree of execution steps. You can customize which steps run and in what order.

**Default browser flow**:
```
1. Cookie check (SSO)
2. Kerberos (optional)
3. Identity Provider Redirector
4. Forms:
   a. Username/Password form
   b. OTP form (if MFA configured)
```

**Custom flow example — Add CAPTCHA on login**:
1. Admin Console → Authentication → Browser → Copy
2. Add execution: Implement `FormAction` SPI
3. Set execution to REQUIRED or CONDITIONAL
4. Bind new flow to realm browser flow

## Custom Keycloak Providers (SPI)

Keycloak has a Service Provider Interface (SPI) system for extensibility. Key SPIs:

| SPI | Use Case |
|---|---|
| `AuthenticatorFactory` | Custom authentication steps (CAPTCHA, custom MFA) |
| `ProtocolMapperSpi` | Custom JWT claim logic |
| `EventListenerProvider` | React to login/logout/failure events |
| `UserStorageProvider` | Custom user data source (not just LDAP) |
| `PasswordPolicyProvider` | Custom password rules |

### Custom Protocol Mapper (add custom claim to JWT)

```java
public class OrganizationClaimMapper extends AbstractOIDCProtocolMapper
    implements OIDCAccessTokenMapper, OIDCIDTokenMapper {

    public static final String PROVIDER_ID = "organization-claim-mapper";

    @Override
    public String getDisplayCategory() {
        return TOKEN_MAPPER_CATEGORY;
    }

    @Override
    public String getDisplayType() {
        return "Organization Claim";
    }

    @Override
    public String getId() {
        return PROVIDER_ID;
    }

    @Override
    protected void setClaim(IDToken token, ProtocolMapperModel mappingModel,
                            UserSessionModel userSession, KeycloakSession keycloakSession,
                            ClientSessionContext clientSessionCtx) {
        
        UserModel user = userSession.getUser();
        
        // Fetch from user attribute
        String orgId = user.getFirstAttribute("organizationId");
        
        // Or fetch from external service/DB
        // String orgId = externalService.getOrgForUser(user.getId());
        
        if (orgId != null) {
            token.getOtherClaims().put("org_id", orgId);
        }
    }
}

// Register as SPI: resources/META-INF/services/org.keycloak.protocol.ProtocolMapper
// Contains: com.example.OrganizationClaimMapper

// Deploy as JAR to: /opt/keycloak/providers/
// Restart Keycloak
```

## Event Listeners

React to Keycloak events (login, logout, registration, failures):

```java
public class AuditEventListenerProvider implements EventListenerProvider {

    private final AuditService auditService;

    @Override
    public void onEvent(Event event) {
        switch (event.getType()) {
            case LOGIN -> auditService.recordLogin(
                event.getUserId(),
                event.getRealmId(),
                event.getIpAddress(),
                Instant.ofEpochMilli(event.getTime())
            );
            case LOGIN_ERROR -> auditService.recordFailedLogin(
                event.getDetails().get("username"),
                event.getError(),
                event.getIpAddress()
            );
            case LOGOUT -> auditService.recordLogout(event.getUserId());
            case REGISTER -> auditService.recordRegistration(event.getUserId());
        }
    }

    @Override
    public void onEvent(AdminEvent event, boolean includeRepresentation) {
        // Admin actions: creating users, changing roles, etc.
        auditService.recordAdminAction(
            event.getAuthDetails().getUserId(),
            event.getOperationType().name(),
            event.getResourceType().name(),
            event.getResourcePath()
        );
    }
}
```

## Identity Brokering (Google, GitHub, etc.)

Configure in Admin Console → Identity Providers → Add Provider → Google

```
Flow:
1. User clicks "Sign in with Google"
2. Keycloak redirects to Google OAuth
3. Google authenticates user, redirects back to Keycloak
4. Keycloak:
   a. First login: Create local Keycloak user, link to Google account
   b. Subsequent logins: Find existing linked account, issue token
5. Keycloak issues standard OIDC token to your app
```

Your app sees a standard Keycloak token regardless of how the user authenticated (password, Google, GitHub, SAML).

**First Login Flow**: Configurable what happens when a new Google user logs in — auto-create account, require profile completion, require email verification, or block until admin approves.

## LDAP Integration

```
# Keycloak LDAP configuration (Admin Console → User Federation → Add LDAP)
Vendor: Active Directory
Connection URL: ldap://ad.company.com:389
Users DN: ou=Users,dc=company,dc=com
Bind DN: cn=keycloak-svc,ou=ServiceAccounts,dc=company,dc=com
Bind Credential: <service account password>
User Object Classes: person, organizationalPerson, user
UUID LDAP Attribute: objectGUID
Username LDAP Attribute: sAMAccountName
```

**LDAP sync modes**:
- **READ_ONLY**: Keycloak reads from LDAP, never writes back
- **WRITABLE**: Keycloak can update LDAP (password changes, attribute updates)
- **UNSYNCED**: Changes stored locally in Keycloak, not synced back

**Group mapping**: Map LDAP groups to Keycloak roles. Members of `CN=Admins,OU=Groups` in AD automatically get `REALM_ADMIN` role in Keycloak.

## Multi-Tenancy with Realms

```
Keycloak Server
├── tenant-acme-corp (realm)
│   ├── Users: acme employees
│   ├── Custom login theme: acme branding
│   └── SSO: integrated with acme.com SAML IdP
│
├── tenant-globex (realm)
│   ├── Users: globex employees
│   └── Google login enabled
│
└── tenant-initech (realm)
    └── Standard email/password
```

**Realm-per-tenant routing**:
```
https://auth.saas.com/realms/{tenantId}/...
```

Clients extract `tenantId` from the issuer URL in the JWT: `iss: https://auth.saas.com/realms/acme-corp`.

**Trade-off**: Many realms → management complexity. Consider a dedicated admin script or Terraform provider (`mrparkers/keycloak`) to manage realms as code.

---

# 9. Real-World Scenarios

## Scenario 1: SaaS Multi-Tenant Authentication

### Context
A B2B SaaS platform with 50 enterprise customers. Each customer has their own users, branding, and potentially their own identity provider (corporate SSO).

### Architecture

```
Customer A (acme.com)         SaaS Platform              Keycloak
     │                             │                         │
     │  Users log in with           │                         │
     │  SAML via corporate SSO      │                         │
     │                             │                         │
  Browser ── SAML → Keycloak ─────────────────────────────────
  (realm: tenant-acme)             │                         │
     │◄── OIDC token ──────────────│                         │

Customer B (globex.com)
     │  Users log in with Google   │
     │                             │
  Browser ── Google OAuth → Keycloak
  (realm: tenant-globex)           │
     │◄── OIDC token ──────────────│
```

### Implementation Approach

1. **Realm per customer**: Complete isolation, custom flows, custom branding
2. **Shared backend API**: Reads tenant from JWT issuer URL
3. **Realm provisioning**: Terraform + Keycloak provider automates realm creation when a new customer signs up
4. **Custom login theme**: Per-realm theme with customer logo/colors

```java
// Backend: extract tenant from JWT
@Component
public class TenantResolver {
    
    public String resolveTenant(Jwt jwt) {
        // "iss": "https://auth.saas.com/realms/tenant-acme"
        String issuer = jwt.getIssuer().toString();
        // Extract "tenant-acme" from the realm path
        return issuer.substring(issuer.lastIndexOf('/') + 1);
    }
}

// Multi-tenant JwtDecoder: validates against the correct realm's JWKS
@Bean
public AuthenticationManagerResolver<HttpServletRequest> multiTenantAuthResolver(
        @Value("${keycloak.server-url}") String serverUrl) {
    
    return request -> {
        // Extract realm from "Authorization" token's "iss" claim
        // or from a request header "X-Tenant-ID"
        String tenantId = resolveTenantFromRequest(request);
        String jwksUri = serverUrl + "/realms/" + tenantId + 
                         "/protocol/openid-connect/certs";
        
        JwtDecoder decoder = NimbusJwtDecoder.withJwkSetUri(jwksUri).build();
        return new JwtAuthenticationProvider(decoder)::authenticate;
    };
}
```

### Why Keycloak

- SAML brokering for enterprise customers without custom code
- Per-realm branding and flows
- Existing LDAP integration support
- Audit logs per realm

---

## Scenario 2: Enterprise Internal SSO

### Context
Large enterprise with 20 internal web applications (HR portal, project management, finance tool, etc.). Currently, each app has its own login. Goal: employees log in once, access all apps.

### Architecture

```
Employee Browser
      │
      ├── Opens HR Portal ──────────────────────────────────────────►
      │                                                    HR Portal │
      │                                              (Spring Boot)   │
      │◄── redirect to Keycloak ────────────────────────────────────│
      │
      ├── Login once at Keycloak (with LDAP/AD credentials)
      │
      │── redirect back with token ─────────────────────────────────►
      │                                                    HR Portal │
      │◄── authenticated ───────────────────────────────────────────│
      │
      ├── Opens Finance Tool ────────────────────────────────────────►
      │                                                 Finance Tool │
      │◄── redirect to Keycloak ────────────────────────────────────│
      │── SSO session detected ─────────────────────────────────────►
      │── redirect with token (NO login prompt) ────────────────────►
      │                                                 Finance Tool │
      │◄── authenticated (seamlessly) ──────────────────────────────│
```

### Implementation Approach

1. **Single `corporate` realm**: All employees in one realm
2. **LDAP federation**: Keycloak reads users from Active Directory
3. **Group mapping**: AD groups → Keycloak client roles (Finance group → finance:read role)
4. **All apps register as clients**: Each app gets its own `client_id`
5. **Back-channel logout**: When a user logs out of any app, all app sessions end

### LDAP Group → Role Mapping

```
AD Group: CN=FinanceTeam → Keycloak Client Role: finance-app:FINANCE_READ
AD Group: CN=FinanceManagers → Keycloak Client Role: finance-app:FINANCE_WRITE
AD Group: CN=IT_Admins → Keycloak Realm Role: REALM_ADMIN
```

---

## Scenario 3: Microservices Security Architecture

### Context
E-commerce platform with 8 microservices. Services call each other. Frontend is a React SPA.

### Architecture

```
React SPA
  │ (Authorization Code + PKCE)
  ▼
API Gateway (Kong)
  │ JWT validation (signature + expiry)
  │ Route based on path
  │
  ├──► Order Service (Spring Boot)
  │       │ Validates JWT locally
  │       │ @PreAuthorize role checks
  │       │
  │       │── Client Credentials ──► Inventory Service
  │       │── Client Credentials ──► Payment Service
  │       │
  ├──► Product Service (Spring Boot)
  │       │ Public reads, authenticated writes
  │
  └──► User Service (Spring Boot)
          │ Manages user profiles
          │ Calls Keycloak Admin API for user creation

Keycloak
  ├── Clients: frontend-app (public), api-gateway, order-service,
  │            inventory-service, payment-service, product-service, user-service
  └── Realm Roles: USER, ADMIN
      Client Roles: order:write, order:read, inventory:update, payment:process
```

### Key Design Decisions

1. **API Gateway validates authentication**: Token signature and expiry. Invalid tokens never reach services.

2. **Services handle authorization**: Each service checks roles for its own endpoints. Defense in depth.

3. **Service accounts per service**: `order-service` client has a service account with only `inventory:read` and `payment:process` roles. Not admin roles.

4. **Short token TTL**: 5-minute access token. Gateway handles the frequent JWKS cache refresh.

5. **Audience restriction**: Order service tokens have `aud: order-service`. Payment service rejects tokens with wrong audience.

```java
// API Gateway filter (pseudocode - Kong/Spring Cloud Gateway)
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    String token = extractBearerToken(exchange.getRequest());
    
    return jwtDecoder.decode(token)
        .flatMap(jwt -> {
            // Pass user info downstream as headers (services trust gateway)
            ServerHttpRequest mutated = exchange.getRequest().mutate()
                .header("X-User-Id", jwt.getSubject())
                .header("X-User-Roles", extractRoles(jwt))
                .build();
            return chain.filter(exchange.mutate().request(mutated).build());
        })
        .onErrorResume(JwtException.class, e -> {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        });
}
```

---

# 10. Comparisons & Trade-offs

## Keycloak vs Auth0

| Factor | Keycloak | Auth0 |
|---|---|---|
| Hosting | Self-hosted (you operate it) | Managed cloud service |
| Cost | Free (open source) + infra cost | Per-MAU pricing (can be expensive at scale) |
| Setup time | Hours to days | Minutes |
| Operations burden | High (you manage upgrades, HA, backups) | None |
| Customization | Full (SPI, themes, flows) | Limited (Actions, Rules) |
| Compliance | You prove it | SOC2, HIPAA-ready certifications |
| Vendor lock-in | None (open source) | High |
| Enterprise features | Enterprise subscription (RH-SSO) | Included |
| Data residency | Full control | Depends on Auth0 region |

**Choose Auth0 when**:
- Small team, speed matters more than cost
- No ops capacity for another stateful service
- Need rapid compliance certifications
- User count is <50k MAU (pricing is reasonable)

**Choose Keycloak when**:
- Cost at scale (>100k MAU, Auth0 gets expensive)
- Data sovereignty/residency requirements
- Need deep customization (custom auth flows, custom SPIs)
- Existing infra team can run it
- Integrating with on-premise LDAP/AD

## Keycloak vs Building Custom Auth

| Factor | Keycloak | Custom Auth |
|---|---|---|
| Development time | Days | Months |
| Security risk | Low (audited, battle-tested) | High (subtle bugs are common) |
| Features | Full-featured out of box | Only what you build |
| MFA support | Built-in (TOTP, WebAuthn) | Build it yourself |
| Social login | Built-in (Google, GitHub, etc.) | Integrate each provider |
| SAML support | Built-in | Complex to implement correctly |
| Maintenance | Keycloak updates handle security patches | You own every CVE |
| Learning curve | Medium (Keycloak concepts) | Low initially, high later |

**Build custom auth only when**:
- You're a security product company and auth is core IP
- Performance requirements are extreme (sub-millisecond auth with millions of concurrent users)
- Keycloak's architecture fundamentally conflicts with your requirements
- Your team includes experienced security engineers who understand cryptography, token security, and session management at depth

**The hidden cost of custom auth**: The initial build is 20% of the effort. Ongoing maintenance (security patches, new protocol support, MFA options, compliance) is 80%. Most teams dramatically underestimate this.

## When Keycloak Is a Bad Choice

1. **Small application, small team**: A 3-person startup with a simple web app doesn't need Keycloak. Use Firebase Auth, Auth0, or Supabase Auth. The operational overhead isn't worth it.

2. **Stateless/serverless architecture**: Keycloak requires a persistent server with a database. If your entire stack is serverless (Lambda, Cloud Run), Keycloak is an awkward fit. Use Cognito or Auth0.

3. **Team can't operate stateful services**: Keycloak clustering, upgrades, backup/restore require operational expertise. If your team isn't comfortable with this, the reliability risk is real.

4. **Embedded auth requirement**: If auth needs to run inside a mobile app or embedded system without a network connection, Keycloak doesn't fit.

5. **AWS-native stack**: If everything is AWS, Cognito integrates tightly with API Gateway, Lambda, and IAM. Keycloak introduces a non-AWS dependency.

6. **Simple API key authentication**: If your API just needs API key auth for B2B clients, Keycloak is massive overkill.

---

# 11. Production-Ready Checklist

**You are production-ready with Keycloak if you can:**

## Fundamentals
- [ ] Explain the difference between OAuth 2.0 and OIDC and what each provides
- [ ] Explain what an access token, refresh token, and ID token are and when to use each
- [ ] Describe the Authorization Code flow step by step, including what travels through the browser and why
- [ ] Explain why ROPC and Implicit flows are deprecated and what to use instead
- [ ] Distinguish between authentication and authorization and why a 401 ≠ 403

## Keycloak Architecture
- [ ] Explain the purpose of a realm and why you create a dedicated realm per environment
- [ ] Distinguish between public and confidential clients and when to use each
- [ ] Explain realm roles vs client roles and model permissions appropriately
- [ ] Describe how SSO works via Keycloak session cookies and back-channel logout
- [ ] Explain how Keycloak cluster nodes share session state via Infinispan

## Security Configuration
- [ ] Configure exact redirect URIs (no wildcards) for all clients
- [ ] Disable `Full Scope Allowed` and configure explicit scope mappings
- [ ] Add audience (`aud`) mappers and validate audience in resource servers
- [ ] Set appropriate token TTLs (5 minutes for access, 30 minutes for refresh)
- [ ] Enable brute-force protection in realm settings
- [ ] Store client secrets in Vault or Kubernetes secrets, never in code
- [ ] Configure HTTPS for all Keycloak endpoints in production

## Spring Boot Integration
- [ ] Configure `spring-boot-starter-oauth2-resource-server` for JWT validation
- [ ] Implement `JwtAuthenticationConverter` to extract roles from Keycloak JWT structure (`realm_access.roles`, `resource_access.<client>.roles`)
- [ ] Validate token issuer and audience in `JwtDecoder`
- [ ] Use `@PreAuthorize` for method-level authorization
- [ ] Access JWT claims cleanly via `JwtAuthenticationToken` in the security context
- [ ] Implement service-to-service auth with client credentials and token caching

## Logout & Session Management
- [ ] Implement back-channel logout endpoint so Keycloak can notify your app of session termination
- [ ] Clear all client-side tokens (memory, cookies) on logout
- [ ] Redirect to Keycloak logout endpoint with `id_token_hint` and `post_logout_redirect_uri`
- [ ] Understand that short-lived access tokens (5 min) are the primary mitigation for stolen tokens

## Production Operations
- [ ] Run Keycloak with PostgreSQL (never H2 in production)
- [ ] Configure Keycloak cluster with Infinispan for session replication
- [ ] Set up load balancer with health check against `/health/ready`
- [ ] Export realm configuration to JSON and commit to version control
- [ ] Schedule database backups and test restoration
- [ ] Monitor Keycloak logs for login failures, token errors, and performance issues
- [ ] Configure Keycloak metrics endpoint and scrape with Prometheus

## Failure Scenarios — Know What Breaks and How
- [ ] **Keycloak downtime**: Cached JWTs remain valid until expiry. New logins and token refreshes fail. Mitigation: HA cluster, circuit breaker in clients.
- [ ] **JWKS key rotation**: Spring Security's `NimbusJwtDecoder` auto-fetches new keys on unknown `kid`. Ensure JWKS endpoint is accessible.
- [ ] **Token expiry flood**: All users' tokens expire simultaneously after a deployment restart. Stagger token TTLs with jitter.
- [ ] **Database connection pool exhaustion**: Keycloak's DB pool is full. Symptoms: login timeouts. Mitigation: tune `db-pool-max-size`, monitor connection count.
- [ ] **Clock skew**: Token `exp` validation fails if server clocks differ >5 seconds. Mitigation: NTP on all nodes, `clockSkew` tolerance in Spring Security.
- [ ] **Session replication lag**: In a cluster, a session created on Node A may not be visible on Node B for milliseconds. Mitigation: sticky sessions during the auth flow (not for API calls).
- [ ] **Misconfigured redirect URI**: Users see "Invalid redirect_uri" error. Fix: exact URI match including trailing slash.
- [ ] **Audience mismatch**: Service rejects valid tokens because `aud` doesn't match. Fix: add audience mapper for each client in Keycloak.

---

*Last reviewed: 2024. Keycloak version 23.x / Quarkus distribution.*
