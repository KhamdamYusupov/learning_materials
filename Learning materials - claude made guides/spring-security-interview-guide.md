# Spring Security Interview Preparation Guide
## Senior Java Engineer Level — Spring Boot 3+ / Spring Security 6+

---

# Table of Contents

- [A. Core Architecture & Fundamentals](#a-core-architecture--fundamentals)
- [B. Authentication Internals](#b-authentication-internals)
- [C. Authorization Architecture](#c-authorization-architecture)
- [D. JWT, OAuth2 & Modern Security](#d-jwt-oauth2--modern-security)
- [E. Filter Chain Internals](#e-filter-chain-internals-critical)
- [F. Session Management & Concurrency](#f-session-management--concurrency)
- [G. CSRF, CORS & Security Vulnerabilities](#g-csrf-cors--security-vulnerabilities)
- [H. Spring Boot Integration & Production Patterns](#h-spring-boot-integration--production-patterns)
- [I. Performance & Scalability](#i-performance--scalability)
- [J. Testing Security](#j-testing-security)
- [Real-World Scenario Section](#real-world-scenario-section)
- [Bonus Sections](#bonus-sections)
- [Mock Interview Section](#mock-interview-section)

---

# A. Core Architecture & Fundamentals

---

## 1. What is Spring Security?

### Interview Question
> "What is Spring Security and why would you choose it over a custom security implementation?"

### Clear & Senior-Level Answer

Spring Security is a framework that provides authentication, authorization, and protection against common exploits for Java applications. But calling it "just a framework" undersells it — it is a **highly customizable security infrastructure** built on top of the Servlet Filter mechanism (for servlet apps) or WebFilter (for reactive apps).

**What makes it powerful:**
- It intercepts every HTTP request *before* it reaches your controllers via a chain of servlet filters
- It provides a pluggable architecture where every component (how users are authenticated, how decisions are made, how sessions are managed) can be swapped out
- It protects against CSRF, session fixation, clickjacking, and other attacks **by default**

**Why choose it over custom security:**
- Custom security implementations almost always have blind spots — you forget session fixation, you don't handle timing attacks in password comparison, you store passwords with weak hashing
- Spring Security has been battle-tested across thousands of production systems
- It integrates seamlessly with the Spring ecosystem (Spring Boot auto-configuration, Spring Data, Spring MVC)

**Security implications:** Rolling your own auth is the #1 way teams introduce critical vulnerabilities. Spring Security encapsulates decades of security best practices.

**Performance implications:** The filter chain adds a few microseconds per request — negligible compared to any I/O operation your app performs.

### Common Candidate Mistakes
- Describing Spring Security as "a login library" — it's a full security infrastructure
- Not mentioning the filter chain architecture
- Failing to explain *why* custom security is risky
- Not knowing it's built on Servlet Filters

### Follow-Up Questions
- How does Spring Security integrate with the Servlet container?
- What happens if you have Spring Security on the classpath but don't configure it?
- How does Spring Security differ between servlet and reactive stacks?

---

## 2. Authentication vs Authorization

### Interview Question
> "Explain the difference between authentication and authorization, and how Spring Security handles each."

### Clear & Senior-Level Answer

**Authentication** = "Who are you?" — verifying identity.
**Authorization** = "What can you do?" — verifying permissions.

These are two distinct phases in Spring Security's request processing:

1. **Authentication phase**: The `AuthenticationManager` (typically `ProviderManager`) delegates to one or more `AuthenticationProvider` instances. If one succeeds, an `Authentication` object is created and stored in the `SecurityContext`.

2. **Authorization phase**: The `AuthorizationManager` (in Spring Security 6+) or legacy `AccessDecisionManager` evaluates whether the authenticated principal has permission to access the requested resource.

**Critical insight:** Authentication always happens first. You cannot authorize an anonymous user meaningfully (though Spring Security does support anonymous authentication tokens for consistent handling).

**The internal flow:**
```
Request → Filter Chain → Authentication Filter → AuthenticationManager
        → SecurityContext populated → Authorization Filter → Controller
```

**Real-world example:** A user logs in with username/password (authentication). They try to access `/admin/users` — Spring Security checks if their roles include `ADMIN` (authorization). Authentication succeeded, but authorization may fail.

### Common Candidate Mistakes
- Confusing the two or mixing them up under pressure
- Not knowing that Spring Security handles them in separate filters
- Not mentioning `SecurityContext` as the bridge between the two phases
- Saying "authentication checks roles" — no, that's authorization

### Follow-Up Questions
- Where is the `Authentication` object stored between the two phases?
- Can authorization happen without authentication?
- How does `@PreAuthorize` relate to the authorization phase?

---

## 3. SecurityFilterChain Architecture

### Interview Question
> "Explain the SecurityFilterChain architecture in Spring Security 6+."

### Clear & Senior-Level Answer

`SecurityFilterChain` is the **core structural component** of Spring Security. It defines which security filters apply to which requests.

**Simple explanation:** It's an ordered list of security filters that incoming HTTP requests pass through. Each filter performs a specific security function (authentication, CSRF protection, session management, etc.).

**How it works internally:**

1. The Servlet container has a single `DelegatingFilterProxy` registered (named `springSecurityFilterChain`)
2. This delegates to a `FilterChainProxy`
3. `FilterChainProxy` holds one or more `SecurityFilterChain` instances
4. For each request, `FilterChainProxy` iterates through chains and uses the first one whose `RequestMatcher` matches
5. The matching chain's filters execute in order

**Why multiple chains matter:** In production, you often need different security rules for different URL patterns — public APIs, admin endpoints, actuator endpoints, static resources.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    // Chain 1: Public API — no authentication
    @Bean
    @Order(1)
    public SecurityFilterChain publicApiFilterChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/api/public/**")
            .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
            .csrf(csrf -> csrf.disable())
            .build();
    }

    // Chain 2: Admin API — strict authentication + authorization
    @Bean
    @Order(2)
    public SecurityFilterChain adminFilterChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/api/admin/**")
            .authorizeHttpRequests(auth -> auth.anyRequest().hasRole("ADMIN"))
            .httpBasic(Customizer.withDefaults())
            .build();
    }

    // Chain 3: Default — everything else
    @Bean
    @Order(3)
    public SecurityFilterChain defaultFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login", "/register").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults())
            .build();
    }
}
```

**Security implications:** The **order matters critically**. If your permissive chain matches first, requests skip stricter chains. Always order from most specific to least specific.

**Performance implications:** Requests only pass through filters in the *matching* chain. A request to `/api/public/health` only goes through Chain 1's filters — it never touches authentication filters from other chains.

**Trade-off:** Multiple chains give flexibility but add complexity. For simple apps, a single chain with well-structured `requestMatchers` is cleaner.

### Common Candidate Mistakes
- Not knowing that `@Order` determines chain evaluation order
- Confusing `SecurityFilterChain` with the old `WebSecurityConfigurerAdapter` (removed in Spring Security 6)
- Not understanding that only ONE chain processes a request
- Forgetting `securityMatcher` — without it, the chain matches all requests

### Follow-Up Questions
- What happens if two chains have the same `@Order`?
- How does `FilterChainProxy` decide which chain handles a request?
- Can a request be processed by multiple chains?

---

## 4. DelegatingFilterProxy

### Interview Question
> "What is DelegatingFilterProxy and why does Spring Security need it?"

### Clear & Senior-Level Answer

`DelegatingFilterProxy` solves a fundamental integration problem: **Servlet Filters are managed by the Servlet container (Tomcat), but Spring Security filters are Spring beans managed by the ApplicationContext.**

**Simple explanation:** It's a bridge. The Servlet container knows about `DelegatingFilterProxy` (a standard Servlet Filter). When a request arrives, `DelegatingFilterProxy` looks up a Spring bean by name and delegates the actual work to it.

**Internal flow:**
```
Tomcat → DelegatingFilterProxy (Servlet Filter)
       → looks up bean "springSecurityFilterChain" in ApplicationContext
       → delegates to FilterChainProxy (Spring Bean)
       → FilterChainProxy selects the right SecurityFilterChain
       → filters execute
```

**Why it matters:**
- Spring Security filters need dependency injection (they use `AuthenticationManager`, `UserDetailsService`, etc.)
- Standard Servlet Filters can't participate in Spring DI
- `DelegatingFilterProxy` bridges this gap by being a Servlet Filter that delegates to a Spring-managed bean

**In Spring Boot, you never see this.** Spring Boot's auto-configuration registers `DelegatingFilterProxy` for you via `SecurityFilterAutoConfiguration`. But knowing it exists is essential for understanding the architecture.

### Common Candidate Mistakes
- Never having heard of it (senior candidates should understand the full stack)
- Confusing `DelegatingFilterProxy` with `FilterChainProxy`
- Not knowing that Spring Boot auto-registers it

### Follow-Up Questions
- What bean name does `DelegatingFilterProxy` look up by default?
- What would happen if `DelegatingFilterProxy` were removed?
- How is this different in a reactive (WebFlux) application?

---

## 5. AuthenticationManager & ProviderManager

### Interview Question
> "Explain the relationship between AuthenticationManager, ProviderManager, and AuthenticationProvider."

### Clear & Senior-Level Answer

This is the **authentication pipeline**:

- **`AuthenticationManager`** — an interface with one method: `authenticate(Authentication)`. This is the entry point for all authentication.
- **`ProviderManager`** — the default implementation. It holds a list of `AuthenticationProvider` instances and iterates through them.
- **`AuthenticationProvider`** — each one knows how to authenticate a specific type of credential (username/password, JWT, LDAP, etc.).

**Internal flow:**

```
Authentication request arrives
  → AuthenticationManager.authenticate(token)
  → ProviderManager iterates providers:
      Provider 1: "I don't support this token type" → skip
      Provider 2: "I support this" → attempts authentication
        → Success: returns fully populated Authentication object
        → Failure: throws AuthenticationException
  → If no provider succeeds → AuthenticationException
```

**Key design insight:** `ProviderManager` supports a **parent `AuthenticationManager`**. If none of its providers can handle a token, it delegates to the parent. This enables hierarchical authentication — e.g., a global provider for service accounts and chain-specific providers for user authentication.

```java
@Bean
public AuthenticationManager authenticationManager(
        UserDetailsService userDetailsService,
        PasswordEncoder passwordEncoder) {
    DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
    provider.setUserDetailsService(userDetailsService);
    provider.setPasswordEncoder(passwordEncoder);

    return new ProviderManager(provider);
}
```

**Real-world example:** An app that supports both database login and LDAP login. You configure two providers — `DaoAuthenticationProvider` for DB and `LdapAuthenticationProvider` for LDAP. `ProviderManager` tries each until one succeeds.

**Security implications:** The provider chain stops at the first successful authentication. Order matters — if a less secure provider matches first, it wins.

**Performance implications:** Each provider may make a network/DB call. Order providers from most common to least common to minimize unnecessary lookups.

### Common Candidate Mistakes
- Not knowing `ProviderManager` is the default implementation
- Not understanding the parent `AuthenticationManager` delegation
- Confusing `AuthenticationProvider` with `AuthenticationManager`
- Not knowing that providers are checked via `supports(Class<?>)` method

### Follow-Up Questions
- How does `ProviderManager` decide which provider to use?
- What happens if multiple providers support the same token type?
- How do you configure different `AuthenticationManager` instances per `SecurityFilterChain`?

---

## 6. SecurityContext & SecurityContextHolder Strategies

### Interview Question
> "How does SecurityContextHolder work, and what strategies does it support?"

### Clear & Senior-Level Answer

**Simple explanation:** `SecurityContextHolder` is a static utility that stores the currently authenticated user's security information. By default it uses `ThreadLocal`, meaning each thread has its own security context.

**The hierarchy:**
```
SecurityContextHolder
  └── SecurityContext
        └── Authentication
              ├── Principal (who)
              ├── Credentials (proof — usually cleared after auth)
              └── Authorities (permissions)
```

**Three strategies:**

| Strategy | Storage | Use Case |
|---|---|---|
| `MODE_THREADLOCAL` (default) | `ThreadLocal` | Standard servlet apps — one thread per request |
| `MODE_INHERITABLETHREADLOCAL` | `InheritableThreadLocal` | When spawning child threads that need security context |
| `MODE_GLOBAL` | Static field | Standalone apps, desktop apps |

**Why this matters in real systems:**

The `ThreadLocal` strategy is the reason Spring Security "just works" in servlet apps — the request thread carries the authentication info everywhere it goes. But it breaks in two scenarios:

1. **Async processing** (`@Async`, `CompletableFuture`): New threads don't inherit the `ThreadLocal`. You need `DelegatingSecurityContextExecutor` or `SecurityContextHolder.setStrategyName(MODE_INHERITABLETHREADLOCAL)`.

2. **Reactive/WebFlux**: There's no `ThreadLocal` in reactive programming. Spring Security uses `ReactiveSecurityContextHolder` backed by Reactor's `Context`.

```java
// Accessing the current user anywhere in your code
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();
Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();

// In a controller — prefer this injection approach
@GetMapping("/me")
public UserDto currentUser(@AuthenticationPrincipal UserDetails user) {
    return new UserDto(user.getUsername(), user.getAuthorities());
}
```

**Security implications:** If you don't clear the `SecurityContext` after a request completes (in a thread-pool scenario), the next request on that thread could inherit stale authentication. Spring Security's `SecurityContextPersistenceFilter` / `SecurityContextHolderFilter` handles this cleanup.

**Performance implications:** `ThreadLocal` access is essentially O(1) — no performance concern. The real cost is in the persistence strategy (session lookup, token validation).

### Common Candidate Mistakes
- Not knowing about the `InheritableThreadLocal` strategy
- Not understanding why `@Async` breaks security context propagation
- Thinking `SecurityContextHolder` stores data in the HTTP session (it doesn't — the session is a *persistence* mechanism; the holder is the *in-memory* access point)
- Not knowing how to access the authenticated user in controller methods

### Follow-Up Questions
- How do you propagate `SecurityContext` to async threads?
- What happens to `SecurityContext` in WebFlux?
- When is `SecurityContext` cleared?

---

## 7. GrantedAuthority vs ROLE_

### Interview Question
> "What is the relationship between GrantedAuthority and roles in Spring Security?"

### Clear & Senior-Level Answer

`GrantedAuthority` is an interface representing a permission. A **role** is just a `GrantedAuthority` with a `ROLE_` prefix — it's a naming convention, not a separate concept.

```java
// These are equivalent:
.hasRole("ADMIN")           // checks for authority "ROLE_ADMIN"
.hasAuthority("ROLE_ADMIN") // checks for authority "ROLE_ADMIN"

// This checks for a non-role authority:
.hasAuthority("document:read")  // checks for "document:read" exactly
```

**Why the `ROLE_` prefix exists:** Historical convention. `hasRole()` is syntactic sugar that auto-prepends `ROLE_`. This catches many developers off guard.

**Real-world pattern — Roles vs Permissions:**

```java
public class CustomUserDetails implements UserDetails {

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        Set<GrantedAuthority> authorities = new HashSet<>();

        // Roles (coarse-grained)
        user.getRoles().forEach(role ->
            authorities.add(new SimpleGrantedAuthority("ROLE_" + role.getName())));

        // Permissions (fine-grained)
        user.getRoles().stream()
            .flatMap(role -> role.getPermissions().stream())
            .forEach(permission ->
                authorities.add(new SimpleGrantedAuthority(permission.getName())));

        return authorities;
    }
}
```

**Trade-off:**
- **Roles** are coarse-grained: `ROLE_ADMIN`, `ROLE_USER`. Simple but inflexible.
- **Authorities/Permissions** are fine-grained: `user:read`, `user:write`, `report:export`. Flexible but more complex to manage.
- **Best practice in production:** Use roles for broad access levels, permissions for fine-grained control. A role maps to multiple permissions.

### Common Candidate Mistakes
- Not knowing about the `ROLE_` prefix auto-prepending
- Using `hasRole("ROLE_ADMIN")` — this checks for `ROLE_ROLE_ADMIN`
- Confusing roles and authorities as different systems
- Not understanding that `GrantedAuthority` is the base abstraction for both

### Follow-Up Questions
- How would you implement a role hierarchy?
- What's the difference between `hasRole` and `hasAuthority`?
- How do you implement fine-grained permissions?

---

## 8. Password Encoding

### Interview Question
> "Explain password encoding in Spring Security and why BCrypt is the default."

### Clear & Senior-Level Answer

**Simple explanation:** Password encoding is the process of transforming a plaintext password into a hash that's stored in the database. Spring Security NEVER stores raw passwords.

**BCrypt is the default because it's deliberately slow.** That's a feature, not a bug. BCrypt uses a "cost factor" (work factor) that determines how many iterations of hashing are performed. This makes brute-force attacks computationally expensive.

**DelegatingPasswordEncoder — the production recommendation:**

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

This creates a `DelegatingPasswordEncoder` that:
- Defaults to BCrypt for new passwords
- Can verify passwords encoded with any supported algorithm
- Stores the algorithm ID as a prefix: `{bcrypt}$2a$10$...`, `{argon2}...`

**Why `DelegatingPasswordEncoder` matters:** It enables **password migration without downtime**. Old passwords hashed with SHA-256 can still be verified, and when users log in, you can re-hash with BCrypt.

```java
// Custom BCrypt strength (default is 10)
@Bean
public PasswordEncoder passwordEncoder() {
    Map<String, PasswordEncoder> encoders = new HashMap<>();
    encoders.put("bcrypt", new BCryptPasswordEncoder(12)); // cost factor 12
    encoders.put("argon2", Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8());
    encoders.put("scrypt", SCryptPasswordEncoder.defaultsForSpringSecurity_v5_8());
    return new DelegatingPasswordEncoder("bcrypt", encoders);
}
```

**Performance implications of cost factor:**

| Cost Factor | Time per Hash (approx.) |
|---|---|
| 10 (default) | ~100ms |
| 12 | ~400ms |
| 14 | ~1.5s |
| 16 | ~6s |

Higher cost factors are more secure but impact login latency. **Production recommendation:** Use cost factor 12-14 and **cache authenticated sessions** so you're not re-hashing on every request.

**Security implications:**
- BCrypt includes a random salt automatically — two identical passwords produce different hashes
- BCrypt is resistant to rainbow table attacks
- The cost factor can be increased over time as hardware gets faster
- **Never use MD5, SHA-1, or SHA-256 without salting for passwords**

### Common Candidate Mistakes
- Using `NoOpPasswordEncoder` (even in dev — it builds bad habits)
- Not knowing about `DelegatingPasswordEncoder`
- Thinking higher BCrypt cost = always better (it impacts UX and creates DoS risk)
- Not understanding that BCrypt generates its own salt

### Follow-Up Questions
- How do you migrate from SHA-256 to BCrypt without forcing password resets?
- What is the DoS risk with high BCrypt cost factors?
- How does Argon2 compare to BCrypt?

---

## 9. Stateless vs Stateful Security

### Interview Question
> "Compare stateless and stateful security. When would you use each?"

### Clear & Senior-Level Answer

| Aspect | Stateful (Session-Based) | Stateless (Token-Based) |
|---|---|---|
| **State storage** | Server-side session (memory/Redis) | Client-side token (JWT) |
| **Scalability** | Requires session affinity or shared session store | Any server can handle any request |
| **Logout** | Immediate (destroy session) | Hard (token valid until expiry) |
| **Storage cost** | Server memory per user | Zero server storage |
| **Security** | Session ID (opaque, revocable) | JWT (self-contained, not easily revocable) |
| **Mobile-friendly** | Requires cookie support | Works with any client |

**Stateful configuration:**
```java
@Bean
public SecurityFilterChain statefulChain(HttpSecurity http) throws Exception {
    return http
        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
            .maximumSessions(1)
            .maxSessionsPreventsLogin(true))
        .formLogin(Customizer.withDefaults())
        .build();
}
```

**Stateless configuration:**
```java
@Bean
public SecurityFilterChain statelessChain(HttpSecurity http) throws Exception {
    return http
        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .csrf(csrf -> csrf.disable())  // CSRF not needed for stateless
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
        .build();
    }
```

**Real-world trade-off insight:** Most modern architectures use **stateless JWT for APIs** and **stateful sessions for web UIs**. The "JWT everywhere" trend ignores that sessions are simpler and more secure for browser-based apps (built-in CSRF protection with cookies, immediate revocability).

**Security trade-off:** Stateless JWTs cannot be revoked before expiry without adding server-side state (a blocklist), which partially defeats the purpose. For high-security systems, short-lived tokens (5-15 min) with refresh tokens stored server-side is the pragmatic middle ground.

### Common Candidate Mistakes
- Saying "JWT is always better than sessions" — it depends on the use case
- Forgetting to disable CSRF for stateless APIs
- Not understanding that `SessionCreationPolicy.STATELESS` means Spring Security won't create or use HTTP sessions
- Not considering token revocation challenges

### Follow-Up Questions
- How do you handle logout with JWTs?
- When is `STATELESS` not appropriate?
- How does `SessionCreationPolicy.NEVER` differ from `STATELESS`?

---

# B. Authentication Internals

---

## 1. UsernamePasswordAuthenticationFilter

### Interview Question
> "Walk me through what happens when a user submits a login form in Spring Security."

### Clear & Senior-Level Answer

`UsernamePasswordAuthenticationFilter` is the filter that handles form-based login (POST to `/login` by default).

**Step-by-step internal flow:**

1. **Request arrives** — POST `/login` with `username` and `password` parameters
2. **Filter matches** — `UsernamePasswordAuthenticationFilter.attemptAuthentication()` is called
3. **Token creation** — A `UsernamePasswordAuthenticationToken` (unauthenticated) is created from the request parameters
4. **Delegation** — The token is passed to `AuthenticationManager.authenticate()`
5. **Provider selection** — `ProviderManager` iterates providers; `DaoAuthenticationProvider.supports(UsernamePasswordAuthenticationToken.class)` returns `true`
6. **User lookup** — `DaoAuthenticationProvider` calls `UserDetailsService.loadUserByUsername(username)`
7. **Password check** — `PasswordEncoder.matches(rawPassword, encodedPassword)` is called
8. **Success path:**
   - A fully authenticated `UsernamePasswordAuthenticationToken` is created (with authorities)
   - `SecurityContextHolder.getContext().setAuthentication(authResult)` stores it
   - `AuthenticationSuccessHandler.onAuthenticationSuccess()` is called (default: redirect to saved request or `/`)
   - Session fixation protection triggers (new session ID generated)
9. **Failure path:**
   - `AuthenticationException` is thrown
   - `AuthenticationFailureHandler.onAuthenticationFailure()` is called (default: redirect to `/login?error`)
   - `SecurityContextHolder.clearContext()`

**Thread model:** Everything runs on the Servlet container's request-handling thread. The `SecurityContext` is set on the current thread's `ThreadLocal`. After the response is committed, `SecurityContextPersistenceFilter` saves the context to the `HttpSession`.

**Security risks:**
- Credentials are sent in the request body — HTTPS is mandatory
- Without rate limiting, brute force attacks are possible
- Default error messages can leak whether a username exists ("bad credentials" is intentionally vague)

```java
// Customizing the form login behavior
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .formLogin(form -> form
            .loginPage("/custom-login")
            .loginProcessingUrl("/authenticate")
            .usernameParameter("email")
            .passwordParameter("pass")
            .successHandler(customSuccessHandler())
            .failureHandler(customFailureHandler())
        )
        .build();
}
```

### Common Candidate Mistakes
- Not knowing the full flow from filter to provider to UserDetailsService
- Forgetting that the `Authentication` object has two states: unauthenticated and authenticated
- Not mentioning session fixation protection
- Not knowing credentials are cleared from the `Authentication` object after successful auth

### Follow-Up Questions
- What happens if you customize `loginProcessingUrl` but forget to update the form action?
- How does session fixation protection work?
- What's the difference between `successHandler` and `defaultSuccessUrl`?

---

## 2. AuthenticationProvider Deep Dive

### Interview Question
> "When would you implement a custom AuthenticationProvider?"

### Clear & Senior-Level Answer

You implement a custom `AuthenticationProvider` when the built-in providers don't match your authentication mechanism. Common scenarios:

- **Multi-factor authentication** — You need to verify a TOTP code in addition to username/password
- **External authentication** — Verifying against a legacy system, custom SSO, or third-party API
- **Custom token format** — Authenticating API keys, custom headers, or non-standard credentials

```java
@Component
public class ApiKeyAuthenticationProvider implements AuthenticationProvider {

    private final ApiKeyService apiKeyService;

    public ApiKeyAuthenticationProvider(ApiKeyService apiKeyService) {
        this.apiKeyService = apiKeyService;
    }

    @Override
    public Authentication authenticate(Authentication authentication)
            throws AuthenticationException {

        ApiKeyAuthenticationToken token = (ApiKeyAuthenticationToken) authentication;
        String apiKey = token.getApiKey();

        ApiKeyEntity entity = apiKeyService.findByKey(apiKey)
            .orElseThrow(() -> new BadCredentialsException("Invalid API key"));

        if (entity.isExpired()) {
            throw new CredentialsExpiredException("API key expired");
        }

        // Return authenticated token with authorities
        return new ApiKeyAuthenticationToken(
            entity.getOwner(),
            apiKey,
            entity.getAuthorities()
        );
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return ApiKeyAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

**The `supports` method is critical.** `ProviderManager` calls `supports()` before `authenticate()`. If you return `true` for the wrong type, you'll get `ClassCastException`. If you never return `true`, your provider is silently skipped.

**Security risks with custom providers:**
- Forgetting to throw `AuthenticationException` on failure (returning `null` means "I can't handle this" and the next provider tries)
- Timing attacks — if validation takes different times for valid vs invalid keys
- Not logging authentication failures for audit

### Common Candidate Mistakes
- Returning `null` instead of throwing `AuthenticationException` when auth fails
- Implementing `supports()` incorrectly
- Not creating a custom `Authentication` token type for their mechanism
- Forgetting to set the `authenticated` flag on the returned token

### Follow-Up Questions
- What's the difference between returning `null` and throwing `AuthenticationException`?
- How do you register a custom provider with the `AuthenticationManager`?
- How would you implement MFA using a custom provider?

---

## 3. DaoAuthenticationProvider

### Interview Question
> "Explain DaoAuthenticationProvider and its role in the authentication process."

### Clear & Senior-Level Answer

`DaoAuthenticationProvider` is Spring Security's default `AuthenticationProvider` for username/password authentication. It bridges the authentication system with your user data store via `UserDetailsService`.

**Internal flow:**

1. Receives `UsernamePasswordAuthenticationToken`
2. Extracts the username
3. Calls `UserDetailsService.loadUserByUsername(username)`
4. If user not found → `UsernamePasswordNotFoundException` (converted to `BadCredentialsException` to prevent username enumeration)
5. Checks account status (locked, disabled, expired) via `UserDetailsChecker`
6. Calls `PasswordEncoder.matches(presentedPassword, storedPassword)`
7. If match → creates authenticated token with user's authorities
8. Runs post-authentication checks (credentials expired?)

**Key design decision:** Username not found is deliberately converted to `BadCredentialsException`. This prevents attackers from determining valid usernames. You can override this behavior, but doing so is a security trade-off.

```java
@Bean
public AuthenticationProvider daoAuthenticationProvider() {
    DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
    provider.setUserDetailsService(customUserDetailsService);
    provider.setPasswordEncoder(passwordEncoder());
    // Optionally reveal that username was not found (usually not recommended)
    provider.setHideUserNotFoundExceptions(false);
    return provider;
}
```

**Performance consideration:** `loadUserByUsername` hits the database on every authentication attempt. In high-traffic systems, consider caching `UserDetails` (but invalidate the cache when user roles change).

### Common Candidate Mistakes
- Not knowing that `UsernameNotFoundException` is hidden by default
- Not understanding the pre/post authentication checks
- Not knowing where `PasswordEncoder` fits in this flow

---

## 4. UserDetails & UserDetailsService

### Interview Question
> "Explain UserDetails and UserDetailsService. How do you implement them for a real application?"

### Clear & Senior-Level Answer

`UserDetailsService` is a single-method interface that loads user data:
```java
UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
```

`UserDetails` is the contract representing the authenticated user's core information:

```java
public interface UserDetails {
    Collection<? extends GrantedAuthority> getAuthorities();
    String getPassword();
    String getUsername();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}
```

**Production implementation:**

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public CustomUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    @Transactional(readOnly = true)
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {

        User user = userRepository.findByEmailWithRoles(username)
            .orElseThrow(() -> new UsernameNotFoundException(
                "User not found: " + username));

        return new CustomUserDetails(user);
    }
}

public class CustomUserDetails implements UserDetails {

    private final User user;

    public CustomUserDetails(User user) {
        this.user = user;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return user.getRoles().stream()
            .flatMap(role -> {
                // Include role itself and its permissions
                Set<GrantedAuthority> authorities = new HashSet<>();
                authorities.add(new SimpleGrantedAuthority("ROLE_" + role.getName()));
                role.getPermissions().forEach(p ->
                    authorities.add(new SimpleGrantedAuthority(p.getName())));
                return authorities.stream();
            })
            .collect(Collectors.toSet());
    }

    @Override
    public String getPassword() { return user.getPassword(); }

    @Override
    public String getUsername() { return user.getEmail(); }

    @Override
    public boolean isAccountNonExpired() { return true; }

    @Override
    public boolean isAccountNonLocked() { return !user.isLocked(); }

    @Override
    public boolean isCredentialsNonExpired() { return true; }

    @Override
    public boolean isEnabled() { return user.isActive(); }

    // Expose the domain User for downstream use
    public User getDomainUser() { return user; }
}
```

**Critical N+1 issue:** `loadUserByUsername` is called during authentication. If you lazy-load roles/permissions, you'll hit the database multiple times. Always use `JOIN FETCH` or `@EntityGraph`:

```java
@Query("SELECT u FROM User u JOIN FETCH u.roles r JOIN FETCH r.permissions WHERE u.email = :email")
Optional<User> findByEmailWithRoles(@Param("email") String email);
```

**Security consideration:** The `UserDetails` object is stored in the `SecurityContext` (and potentially serialized to the session). Don't include sensitive data like full credit card numbers or other PII that doesn't need to be there.

### Common Candidate Mistakes
- Returning Spring's `User` builder for production code (it's fine for prototyping, not for real apps)
- Not using `JOIN FETCH` for roles — N+1 query problem
- Throwing a generic exception instead of `UsernameNotFoundException`
- Storing the entire JPA entity in `SecurityContext` (causes serialization issues with lazy proxies)

### Follow-Up Questions
- How do you handle the N+1 problem when loading user authorities?
- What happens if `loadUserByUsername` returns a disabled user?
- How does `UserDetails` relate to `@AuthenticationPrincipal`?

---

## 5. AuthenticationEntryPoint

### Interview Question
> "What is AuthenticationEntryPoint and when is it triggered?"

### Clear & Senior-Level Answer

`AuthenticationEntryPoint` is invoked when an **unauthenticated** user tries to access a protected resource. It decides **how to begin authentication**.

**Key insight:** It's NOT called on authentication failure — it's called when no authentication exists at all.

**Common implementations:**

| Implementation | Behavior | Use Case |
|---|---|---|
| `LoginUrlAuthenticationEntryPoint` | Redirects to login page | Web apps with form login |
| `BasicAuthenticationEntryPoint` | Returns 401 with `WWW-Authenticate` header | REST APIs with Basic auth |
| `BearerTokenAuthenticationEntryPoint` | Returns 401 with `Bearer` scheme | OAuth2 Resource Servers |
| `Http403ForbiddenEntryPoint` | Returns 403 | When you don't want to reveal auth mechanism |

```java
// Custom entry point for APIs
@Bean
public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
    return http
        .exceptionHandling(ex -> ex
            .authenticationEntryPoint((request, response, authException) -> {
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                response.setContentType(MediaType.APPLICATION_JSON_VALUE);
                response.getWriter().write("""
                    {"error": "unauthorized", "message": "Authentication required"}
                    """);
            })
        )
        .build();
}
```

**When it's triggered:**
`ExceptionTranslationFilter` catches `AuthenticationException` thrown by downstream filters. If the user is anonymous (not authenticated), it delegates to `AuthenticationEntryPoint`. If the user IS authenticated but lacks permissions, it delegates to `AccessDeniedHandler` instead.

### Common Candidate Mistakes
- Confusing entry point (unauthenticated) with access denied handler (authenticated but unauthorized)
- Not customizing it for REST APIs (default redirects to HTML login page)
- Not knowing `ExceptionTranslationFilter` is what triggers it

---

## 6. Remember-Me Mechanism

### Interview Question
> "How does remember-me authentication work in Spring Security?"

### Clear & Senior-Level Answer

Remember-me allows users to remain authenticated across browser sessions without re-entering credentials. Spring Security offers two implementations:

**1. Simple Hash-Based Token** (default)
- Cookie contains: `base64(username + ":" + expirationTime + ":" + md5Hex(username + ":" + expirationTime + ":" + password + ":" + key))`
- **Risk:** If the password hash is compromised, an attacker can forge remember-me tokens
- **Risk:** Token is valid until expiry — no way to revoke

**2. Persistent Token** (recommended for production)
- Stores a series identifier and token in the database
- Each authentication generates a new token but keeps the same series
- Detects theft: if a series is presented with an old token, all tokens for that user are invalidated

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .rememberMe(remember -> remember
            .tokenRepository(persistentTokenRepository())
            .tokenValiditySeconds(7 * 24 * 60 * 60) // 7 days
            .userDetailsService(userDetailsService)
        )
        .build();
}

@Bean
public PersistentTokenRepository persistentTokenRepository() {
    JdbcTokenRepositoryImpl repo = new JdbcTokenRepositoryImpl();
    repo.setDataSource(dataSource);
    return repo;
}
```

**Security implication:** Remember-me tokens are high-value targets. If stolen, they grant full access. Always use `Secure` and `HttpOnly` cookie flags. Consider limiting what remember-me authenticated users can do (e.g., require full authentication for sensitive operations using `fullyAuthenticated()`).

### Common Candidate Mistakes
- Not knowing the difference between hash-based and persistent token approaches
- Using simple hash-based in production
- Not understanding `isRememberMe()` vs `isFullyAuthenticated()`

---

# C. Authorization Architecture

---

## 1. URL-Based Authorization

### Interview Question
> "How do you configure URL-based authorization in Spring Security 6+?"

### Clear & Senior-Level Answer

URL-based authorization uses `authorizeHttpRequests()` to define access rules per URL pattern. In Spring Security 6+, this uses the `AuthorizationManager` API internally.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth
            // Order: most specific first
            .requestMatchers("/api/admin/**").hasRole("ADMIN")
            .requestMatchers("/api/reports/**").hasAnyRole("ADMIN", "ANALYST")
            .requestMatchers(HttpMethod.POST, "/api/users").hasAuthority("user:create")
            .requestMatchers(HttpMethod.GET, "/api/users/**").hasAuthority("user:read")
            .requestMatchers("/api/public/**", "/health", "/info").permitAll()
            .requestMatchers("/actuator/**").hasRole("OPS")
            // Default: require authentication for everything else
            .anyRequest().authenticated()
        )
        .build();
}
```

**How it works internally:**

1. `AuthorizationFilter` (replaced the old `FilterSecurityInterceptor`) intercepts every request
2. It asks the `AuthorizationManager` (specifically `RequestMatcherDelegatingAuthorizationManager`) to make a decision
3. The manager iterates through registered `RequestMatcher` → `AuthorizationManager` pairs
4. The first matching rule's `AuthorizationManager` makes the decision
5. If denied → `AccessDeniedException` → `ExceptionTranslationFilter` handles it

**Critical rules:**
- **Order matters** — first match wins. Put specific rules before general ones.
- **`anyRequest()` must be last** — it acts as a catch-all
- **Forgetting `anyRequest()` is a security risk** — unmatched URLs might be accessible

**Spring Security 6 change:** `authorizeHttpRequests()` replaced `authorizeRequests()`. The new version dispatches to all dispatcher types (FORWARD, INCLUDE, etc.) by default, which is more secure.

### Common Candidate Mistakes
- Ordering rules incorrectly (putting `permitAll` before `hasRole`)
- Using the deprecated `authorizeRequests()` instead of `authorizeHttpRequests()`
- Forgetting `anyRequest().authenticated()` at the end
- Not knowing that `requestMatchers` replaced `antMatchers` in Spring Security 6

### Follow-Up Questions
- What's the difference between `authorizeRequests()` and `authorizeHttpRequests()`?
- How does `requestMatchers` pattern matching work?
- What happens if no rule matches a request?

---

## 2. Method-Level Security

### Interview Question
> "Explain method-level security annotations and when to use them."

### Clear & Senior-Level Answer

Method-level security applies authorization rules directly on service methods, giving you fine-grained control beyond URL patterns.

**Enable it:**
```java
@Configuration
@EnableMethodSecurity  // replaces @EnableGlobalMethodSecurity
public class MethodSecurityConfig {
}
```

**Key annotations:**

```java
@Service
public class DocumentService {

    // Pre-authorization: checked BEFORE method execution
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteAllDocuments() { ... }

    // SpEL with method parameters
    @PreAuthorize("hasAuthority('document:read') or #document.owner == authentication.name")
    public DocumentDto getDocument(Document document) { ... }

    // Post-authorization: checked AFTER method execution (access to returnObject)
    @PostAuthorize("returnObject.owner == authentication.name or hasRole('ADMIN')")
    public Document findDocument(Long id) {
        return documentRepository.findById(id).orElseThrow();
    }

    // Filter collections before/after
    @PreFilter("filterObject.tenantId == authentication.principal.tenantId")
    public void batchUpdate(List<Document> documents) { ... }

    @PostFilter("filterObject.owner == authentication.name")
    public List<Document> findAll() {
        return documentRepository.findAll();
    }
}
```

**How it works internally:**
- `@EnableMethodSecurity` registers an AOP interceptor (`AuthorizationManagerBeforeMethodInterceptor` / `AfterMethodInterceptor`)
- Spring creates proxies around secured beans
- Before/after method execution, the interceptor evaluates SpEL expressions
- Expressions have access to `authentication`, method parameters, and (for `@PostAuthorize`) `returnObject`

**When to use URL-based vs method-level:**

| URL-Based | Method-Level |
|---|---|
| Coarse-grained (entire endpoints) | Fine-grained (individual operations) |
| Configuration in one place | Authorization close to business logic |
| HTTP-layer only | Works for any Spring bean, not just controllers |
| Simple role/authority checks | Complex rules with SpEL |

**Performance implications:**
- `@PostFilter` loads ALL data then filters — terrible for large datasets. Use query-level filtering instead.
- AOP proxying adds overhead but is negligible for typical apps
- Complex SpEL expressions are evaluated on every invocation — keep them simple

### Common Candidate Mistakes
- Using `@PostFilter` on large collections instead of filtering at the query level
- Using the deprecated `@EnableGlobalMethodSecurity` instead of `@EnableMethodSecurity`
- Not understanding that method security uses AOP proxies (won't work on internal method calls within the same class)
- Writing overly complex SpEL that's hard to test and maintain

### Follow-Up Questions
- Why doesn't `@PreAuthorize` work when calling a method from within the same class?
- How do you test methods annotated with `@PreAuthorize`?
- What's the performance impact of `@PostFilter` on large result sets?

---

## 3. AuthorizationManager (Modern Model)

### Interview Question
> "What is AuthorizationManager and how does it differ from the legacy AccessDecisionManager?"

### Clear & Senior-Level Answer

`AuthorizationManager` is the **simplified authorization API introduced in Spring Security 5.6+** that replaces the complex `AccessDecisionManager`/`AccessDecisionVoter` system.

**Old model (legacy):**
```
AccessDecisionManager
  ├── AffirmativeBased (any voter grants → allow)
  ├── ConsensusBased (majority wins)
  └── UnanimousBased (all must grant)
    └── AccessDecisionVoter[] (each votes GRANT / DENY / ABSTAIN)
```

**New model:**
```java
@FunctionalInterface
public interface AuthorizationManager<T> {
    AuthorizationDecision check(Supplier<Authentication> authentication, T object);
}
```

One method. Clean. Composable.

**Key advantage:** `AuthorizationManager` is simpler and supports reactive patterns. You can compose managers:

```java
// Custom authorization logic
@Bean
public AuthorizationManager<RequestAuthorizationContext> customAuthManager() {
    AuthorizationManager<RequestAuthorizationContext> adminAuth =
        AuthorityAuthorizationManager.hasRole("ADMIN");
    AuthorizationManager<RequestAuthorizationContext> ipAuth =
        (auth, context) -> {
            String ip = context.getRequest().getRemoteAddr();
            boolean isInternal = ip.startsWith("10.") || ip.startsWith("192.168.");
            return new AuthorizationDecision(isInternal);
        };

    // Admin OR internal network
    return (authentication, object) -> {
        if (adminAuth.check(authentication, object).isGranted()) {
            return new AuthorizationDecision(true);
        }
        return ipAuth.check(authentication, object);
    };
}
```

**Trade-off:** The old voter-based system was more structured for complex scenarios where you needed formal voting. The new system is simpler but requires you to compose logic yourself.

### Common Candidate Mistakes
- Still explaining the old `AccessDecisionVoter` model without mentioning `AuthorizationManager`
- Not knowing that `AuthorizationFilter` replaced `FilterSecurityInterceptor`
- Not understanding that `AuthorizationManager` is a functional interface

---

## 4. Role Hierarchy

### Interview Question
> "How do you implement a role hierarchy in Spring Security?"

### Clear & Senior-Level Answer

Role hierarchy defines inheritance between roles. `ROLE_ADMIN` automatically includes all permissions of `ROLE_MANAGER`, which includes all of `ROLE_USER`.

```java
@Bean
public RoleHierarchy roleHierarchy() {
    return RoleHierarchyImpl.fromHierarchy("""
        ROLE_ADMIN > ROLE_MANAGER
        ROLE_MANAGER > ROLE_ANALYST
        ROLE_ANALYST > ROLE_USER
        """);
}

// For method-level security to respect role hierarchy:
@Bean
public MethodSecurityExpressionHandler methodSecurityExpressionHandler(
        RoleHierarchy roleHierarchy) {
    DefaultMethodSecurityExpressionHandler handler =
        new DefaultMethodSecurityExpressionHandler();
    handler.setRoleHierarchy(roleHierarchy);
    return handler;
}
```

**Why it matters:** Without role hierarchy, you have to assign every sub-role explicitly. An admin would need `ROLE_ADMIN, ROLE_MANAGER, ROLE_USER` in the database. With hierarchy, you only assign `ROLE_ADMIN` and it implicitly includes the others.

**Common mistake:** Configuring `RoleHierarchy` bean but forgetting to wire it into `MethodSecurityExpressionHandler` — method-level security won't respect the hierarchy.

---

# D. JWT, OAuth2 & Modern Security

---

## 1. JWT Structure and Validation

### Interview Question
> "Explain JWT structure, validation, and security considerations."

### Clear & Senior-Level Answer

**JWT structure** (three Base64URL-encoded parts separated by dots):
```
Header.Payload.Signature

eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyMSIsInJvbGVzIjpbIkFETUlOIl19.signature
```

- **Header:** Algorithm and token type: `{"alg": "RS256", "typ": "JWT"}`
- **Payload:** Claims (data): `{"sub": "user1", "roles": ["ADMIN"], "exp": 1699999999}`
- **Signature:** `RSASHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), privateKey)`

**Spring Security JWT validation flow:**

```java
@Bean
public SecurityFilterChain apiSecurity(HttpSecurity http) throws Exception {
    return http
        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2
            .jwt(jwt -> jwt
                .decoder(jwtDecoder())
                .jwtAuthenticationConverter(jwtAuthenticationConverter())
            )
        )
        .build();
}

@Bean
public JwtDecoder jwtDecoder() {
    return NimbusJwtDecoder.withPublicKey(rsaPublicKey).build();
}

@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtGrantedAuthoritiesConverter authoritiesConverter =
        new JwtGrantedAuthoritiesConverter();
    authoritiesConverter.setAuthoritiesClaimName("roles");
    authoritiesConverter.setAuthorityPrefix("ROLE_");

    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
    return converter;
}
```

**Internal validation steps (BearerTokenAuthenticationFilter):**

1. Extract token from `Authorization: Bearer <token>` header
2. `JwtDecoder.decode(token)` parses and validates:
   - Signature verification (is the token authentic?)
   - Expiration check (`exp` claim)
   - Not-before check (`nbf` claim)
   - Issuer validation (`iss` claim)
   - Audience validation (`aud` claim)
3. `JwtAuthenticationConverter` converts the `Jwt` to `JwtAuthenticationToken` with authorities
4. Token stored in `SecurityContextHolder`

### HMAC vs RSA

| | HMAC (HS256) | RSA (RS256) |
|---|---|---|
| **Key** | Shared secret | Public/private key pair |
| **Who can create tokens** | Anyone with the secret | Only the private key holder |
| **Who can verify tokens** | Anyone with the secret | Anyone with the public key |
| **Use case** | Single service | Distributed systems |
| **Security** | Secret compromise = full control | Private key compromise = full control, but public key exposure is safe |

**Production recommendation:** Always use RS256 or ES256 for distributed systems. HMAC requires sharing the secret with every service that validates tokens — one compromised service compromises all.

### Attack Scenarios

- **Algorithm confusion attack:** Attacker changes header to `{"alg": "none"}` or `{"alg": "HS256"}` when server expects RS256. **Mitigation:** Always specify expected algorithm in decoder configuration — Spring Security does this correctly.
- **Token theft:** Stolen JWT gives full access until expiry. **Mitigation:** Short expiry (5-15 min) + refresh tokens.
- **Claim injection:** If server blindly trusts claims. **Mitigation:** Validate issuer, audience, and use a trusted signing key.

### Common Candidate Mistakes
- Saying JWT is "encrypted" — it's encoded and signed, not encrypted (unless using JWE)
- Not knowing the difference between HS256 and RS256
- Not mentioning token revocation challenges
- Storing JWTs in `localStorage` (XSS vulnerable) instead of `HttpOnly` cookies

### Follow-Up Questions
- How do you revoke a JWT before expiry?
- What's the difference between JWS and JWE?
- How do you handle key rotation?

---

## 2. Refresh Token Strategy

### Interview Question
> "Design a refresh token strategy for a production API."

### Clear & Senior-Level Answer

**The problem:** Short-lived JWTs are secure but annoy users with frequent re-authentication. Refresh tokens solve this.

**Production pattern:**

```
Access Token:  Short-lived (5-15 min), stateless JWT, carried in Authorization header
Refresh Token: Long-lived (7-30 days), opaque string, stored server-side, sent via HttpOnly cookie
```

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final AuthenticationManager authManager;
    private final JwtService jwtService;
    private final RefreshTokenService refreshTokenService;

    @PostMapping("/login")
    public ResponseEntity<TokenResponse> login(@RequestBody LoginRequest request) {
        Authentication auth = authManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.email(), request.password()));

        String accessToken = jwtService.generateAccessToken(auth);
        RefreshToken refreshToken = refreshTokenService.createRefreshToken(auth.getName());

        ResponseCookie cookie = ResponseCookie.from("refresh_token", refreshToken.getToken())
            .httpOnly(true)
            .secure(true)
            .sameSite("Strict")
            .path("/api/auth/refresh")
            .maxAge(Duration.ofDays(7))
            .build();

        return ResponseEntity.ok()
            .header(HttpHeaders.SET_COOKIE, cookie.toString())
            .body(new TokenResponse(accessToken));
    }

    @PostMapping("/refresh")
    public ResponseEntity<TokenResponse> refresh(
            @CookieValue("refresh_token") String refreshTokenValue) {

        RefreshToken refreshToken = refreshTokenService.validateAndRotate(refreshTokenValue);
        String accessToken = jwtService.generateAccessToken(refreshToken.getUsername());

        // Rotate: old refresh token invalidated, new one issued
        ResponseCookie cookie = ResponseCookie.from("refresh_token", refreshToken.getToken())
            .httpOnly(true).secure(true).sameSite("Strict")
            .path("/api/auth/refresh").maxAge(Duration.ofDays(7))
            .build();

        return ResponseEntity.ok()
            .header(HttpHeaders.SET_COOKIE, cookie.toString())
            .body(new TokenResponse(accessToken));
    }
}
```

**Refresh token rotation:** Every time a refresh token is used, a new one is issued and the old one is invalidated. If a stolen token is used, the legitimate user's next refresh attempt fails (because their token was already rotated by the attacker), alerting them to a breach.

**Security trade-offs:**
- Refresh tokens stored server-side add state but enable revocation
- Token rotation detects theft but complicates concurrent requests (race condition: two requests hit `/refresh` simultaneously)
- Cookie-based refresh tokens are safe from XSS but require CSRF protection on the refresh endpoint

---

## 3. OAuth2 Flows

### Interview Question
> "Explain the OAuth2 Authorization Code flow and Client Credentials flow."

### Clear & Senior-Level Answer

### Authorization Code Flow (with PKCE)

Used for: **User-facing applications** (web apps, mobile apps)

```
1. User clicks "Login with Google"
2. App redirects to: https://accounts.google.com/o/oauth2/auth
     ?response_type=code
     &client_id=YOUR_CLIENT_ID
     &redirect_uri=https://yourapp.com/callback
     &scope=openid email profile
     &code_challenge=CHALLENGE  (PKCE)
     &code_challenge_method=S256
3. User authenticates with Google
4. Google redirects back: https://yourapp.com/callback?code=AUTH_CODE
5. Your server exchanges code for tokens:
     POST https://oauth2.googleapis.com/token
     grant_type=authorization_code&code=AUTH_CODE&code_verifier=VERIFIER
6. Google returns: { access_token, id_token, refresh_token }
```

**Spring Security OAuth2 Client configuration:**

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .oauth2Login(Customizer.withDefaults())
        .oauth2Client(Customizer.withDefaults())
        .build();
}
```

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, email, profile
        provider:
          google:
            issuer-uri: https://accounts.google.com
```

### Client Credentials Flow

Used for: **Service-to-service communication** (no user involved)

```
1. Service A needs to call Service B's API
2. Service A sends: POST /oauth/token
     grant_type=client_credentials
     &client_id=SERVICE_A_ID
     &client_secret=SERVICE_A_SECRET
     &scope=api.read
3. Authorization server returns: { access_token }
4. Service A calls Service B with: Authorization: Bearer <token>
```

```java
// Registering a client credentials client
@Bean
public OAuth2AuthorizedClientManager authorizedClientManager(
        ClientRegistrationRepository registrations,
        OAuth2AuthorizedClientRepository authorizedClients) {

    OAuth2AuthorizedClientProvider provider = OAuth2AuthorizedClientProviderBuilder.builder()
        .clientCredentials()
        .build();

    DefaultOAuth2AuthorizedClientManager manager =
        new DefaultOAuth2AuthorizedClientManager(registrations, authorizedClients);
    manager.setAuthorizedClientProvider(provider);
    return manager;
}
```

### Resource Server vs Authorization Server

- **Authorization Server:** Issues tokens. Handles login, consent, token generation. (Spring Authorization Server project)
- **Resource Server:** Validates tokens and protects APIs. Your typical backend API.
- **Client:** The application requesting access. Your frontend or another service.

A common mistake is trying to make your Resource Server also issue tokens. Keep them separate.

### Common Candidate Mistakes
- Not mentioning PKCE for public clients
- Confusing Authorization Code flow with Implicit flow (deprecated)
- Not knowing the difference between Resource Server and Authorization Server
- Using Client Credentials flow for user-facing apps

---

## 4. Token Revocation and Key Rotation

### Interview Question
> "How do you handle token revocation and key rotation in a JWT-based system?"

### Clear & Senior-Level Answer

### Token Revocation

JWTs are stateless — they're valid until expiry. Revocation options:

**1. Short-lived tokens:** Make access tokens expire in 5-15 minutes. Acceptable latency for most systems.

**2. Token blocklist:**
```java
@Component
public class JwtBlocklistValidator implements OAuth2TokenValidator<Jwt> {

    private final RedisTemplate<String, String> redis;

    @Override
    public OAuth2TokenValidatorResult validate(Jwt jwt) {
        if (redis.hasKey("blocklist:" + jwt.getId())) {
            return OAuth2TokenValidatorResult.failure(
                new OAuth2Error("token_revoked", "Token has been revoked", null));
        }
        return OAuth2TokenValidatorResult.success();
    }
}

@Bean
public JwtDecoder jwtDecoder(JwtBlocklistValidator blocklistValidator) {
    NimbusJwtDecoder decoder = NimbusJwtDecoder.withPublicKey(publicKey).build();

    OAuth2TokenValidator<Jwt> validators = new DelegatingOAuth2TokenValidator<>(
        JwtValidators.createDefaultWithIssuer("https://auth.example.com"),
        blocklistValidator
    );
    decoder.setJwtValidator(validators);
    return decoder;
}
```

**3. Token versioning:** Store a `tokenVersion` per user. Include it in the JWT. On logout/password change, increment the version. Validate that the JWT's version matches the current one. Requires one lightweight DB/cache lookup per request.

### Key Rotation

```java
// Using JWK Set URI — supports automatic key rotation
@Bean
public JwtDecoder jwtDecoder() {
    return NimbusJwtDecoder
        .withJwkSetUri("https://auth.example.com/.well-known/jwks.json")
        .build();
}
```

**Zero-downtime key rotation process:**

1. Generate new key pair
2. Add new public key to JWKS endpoint (now two keys: old and new)
3. Start signing new tokens with new private key
4. Wait for old tokens to expire (max token lifetime)
5. Remove old public key from JWKS endpoint

**The `kid` (Key ID) header in JWT tells resource servers which key to use for verification.** This is how multiple keys coexist during rotation.

### Common Candidate Mistakes
- Saying "JWTs can be revoked easily" — they can't without server-side state
- Not mentioning `kid` header for key rotation
- Not considering the overlap period during key rotation
- Not knowing about JWK Set URI for automatic key fetching

---

# E. Filter Chain Internals (Critical)

---

## 1. Default Filter Ordering

### Interview Question
> "What filters does Spring Security add to the chain, and in what order?"

### Clear & Senior-Level Answer

Spring Security adds approximately 15-30 filters depending on configuration. The key ones, in order:

```
1.  DisableEncodeUrlFilter          - Prevents session ID in URLs
2.  WebAsyncManagerIntegrationFilter - SecurityContext for async
3.  SecurityContextHolderFilter      - Sets up/clears SecurityContext
4.  HeaderWriterFilter               - Security headers (X-Frame-Options, etc.)
5.  CorsFilter                       - CORS handling
6.  CsrfFilter                       - CSRF protection
7.  LogoutFilter                     - Processes logout requests
8.  OAuth2AuthorizationRequestRedirectFilter - OAuth2 login redirect
9.  UsernamePasswordAuthenticationFilter     - Form login
10. BasicAuthenticationFilter        - HTTP Basic auth
11. BearerTokenAuthenticationFilter  - JWT/OAuth2 Resource Server
12. RequestCacheAwareFilter          - Restores cached request after login
13. SecurityContextHolderAwareRequestFilter  - Servlet API integration
14. AnonymousAuthenticationFilter    - Creates anonymous token if none exists
15. ExceptionTranslationFilter       - Handles AuthenticationException, AccessDeniedException
16. AuthorizationFilter              - URL-based authorization checks
```

**Key insight:** Authentication filters (9-11) come BEFORE the authorization filter (16). This ensures the `SecurityContext` is populated before access decisions are made.

**Debugging tip:**
```java
// Add to application.properties to log all filters
logging.level.org.springframework.security=DEBUG

// Or programmatically list all filters
@Bean
public ApplicationRunner securityFilterChainDump(List<SecurityFilterChain> chains) {
    return args -> chains.forEach(chain -> {
        chain.getFilters().forEach(filter ->
            System.out.println(filter.getClass().getSimpleName()));
    });
}
```

---

## 2. OncePerRequestFilter

### Interview Question
> "What is OncePerRequestFilter and why is it important?"

### Clear & Senior-Level Answer

`OncePerRequestFilter` guarantees a filter executes **exactly once per request**, even if the request is forwarded or dispatched internally (e.g., error handling dispatch, forward to JSP).

**Why it matters:** Standard Servlet Filters can execute multiple times per request if the request is forwarded (`RequestDispatcher.forward()`). In security, re-executing authentication or CSRF checks on a forwarded request causes issues.

**All Spring Security filters extend `OncePerRequestFilter`.** When you write a custom filter, you should too:

```java
public class TenantContextFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
            throws ServletException, IOException {

        String tenantId = request.getHeader("X-Tenant-ID");
        if (tenantId != null) {
            TenantContext.setCurrentTenant(tenantId);
        }

        try {
            filterChain.doFilter(request, response);
        } finally {
            TenantContext.clear(); // Always clean up
        }
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        // Skip for static resources
        return request.getRequestURI().startsWith("/static/");
    }
}
```

**Inserting custom filters:**
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .addFilterBefore(new TenantContextFilter(),
            UsernamePasswordAuthenticationFilter.class)
        .addFilterAfter(new AuditFilter(), AuthorizationFilter.class)
        // Or at a specific position:
        .addFilterAt(new CustomAuthFilter(),
            UsernamePasswordAuthenticationFilter.class)
        .build();
}
```

**Common mistake:** Using `addFilterAt` thinking it replaces the existing filter — it doesn't. Both filters execute at that position. To replace, you also need to disable the original.

---

## 3. ExceptionTranslationFilter

### Interview Question
> "Explain how ExceptionTranslationFilter works."

### Clear & Senior-Level Answer

`ExceptionTranslationFilter` is the **bridge between security exceptions and HTTP responses**. It sits just before `AuthorizationFilter` and catches two types of exceptions:

**Flow:**
```
Request → ... → ExceptionTranslationFilter → AuthorizationFilter → Controller
                      ↑                              |
                      |         Catches exceptions    |
                      ←←←←←←←←←←←←←←←←←←←←←←←←←←←←

Exception handling:
1. AuthenticationException → AuthenticationEntryPoint
   (User is NOT authenticated → "Please log in")

2. AccessDeniedException:
   a. User is anonymous → AuthenticationEntryPoint
      (Treat as authentication problem)
   b. User IS authenticated → AccessDeniedHandler
      (User logged in but lacks permission → 403)
```

**Critical detail:** Before invoking `AuthenticationEntryPoint`, `ExceptionTranslationFilter` saves the current request to `RequestCache`. After successful authentication, the user is redirected to the originally requested URL. This is how "redirect to login, then back to the original page" works.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .exceptionHandling(ex -> ex
            .authenticationEntryPoint(customEntryPoint())
            .accessDeniedHandler((request, response, accessDeniedException) -> {
                response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                response.setContentType("application/json");
                response.getWriter().write(
                    "{\"error\": \"forbidden\", \"message\": \""
                    + accessDeniedException.getMessage() + "\"}");
            })
        )
        .build();
}
```

### Common Candidate Mistakes
- Not knowing the difference between `AuthenticationEntryPoint` and `AccessDeniedHandler`
- Not understanding that anonymous users trigger `AuthenticationEntryPoint`, not `AccessDeniedHandler`
- Not knowing about `RequestCache` for post-login redirection

---

## 4. CSRF Filter

### Interview Question
> "How does Spring Security's CSRF protection work?"

### Clear & Senior-Level Answer

`CsrfFilter` protects against Cross-Site Request Forgery by requiring a token on state-changing requests (POST, PUT, DELETE, PATCH).

**How it works:**

1. Server generates a random CSRF token and associates it with the session (or a cookie)
2. The token is made available to the client (via a request attribute, a cookie, or a meta tag)
3. Client must include this token in every state-changing request (as a header or form field)
4. `CsrfFilter` compares the submitted token with the expected token
5. Mismatch → `AccessDeniedException`

**Spring Security 6 default: deferred token**
```java
// Tokens are now lazily loaded by default (Spring Security 6)
// This avoids session creation for every request

// For SPAs that need CSRF token in a cookie:
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler())
        )
        .build();
}
```

**When to disable CSRF:**
- Stateless APIs using Bearer tokens (JWTs) — the token itself is a proof of intent
- APIs consumed only by non-browser clients

```java
// Safe to disable for stateless JWT API
.csrf(csrf -> csrf.disable())
```

**When NOT to disable CSRF:**
- Any endpoint that uses cookies for authentication (session-based auth)
- Form-based web applications
- Mixed apps where some endpoints use sessions

---

# F. Session Management & Concurrency

---

## 1. Session Fixation Protection

### Interview Question
> "What is session fixation and how does Spring Security protect against it?"

### Clear & Senior-Level Answer

**The attack:**
1. Attacker visits the app and gets a session ID: `JSESSIONID=abc123`
2. Attacker tricks victim into using the same session ID (via URL, cookie injection)
3. Victim authenticates — the session `abc123` is now authenticated
4. Attacker uses `abc123` and has victim's authenticated session

**Spring Security protection:** After successful authentication, Spring creates a **new session** and migrates attributes from the old session.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .sessionManagement(session -> session
            // Default: migrate attributes to new session (recommended)
            .sessionFixation().migrateSession()
            // OR: new session with no migration
            // .sessionFixation().newSession()
            // OR: change only the session ID (Servlet 3.1+)
            // .sessionFixation().changeSessionId()
        )
        .build();
}
```

**`changeSessionId()`** is the most efficient option (Servlet 3.1+) — it changes the session ID without creating a new session object, so all attributes are preserved without copying overhead.

---

## 2. Concurrent Session Control

### Interview Question
> "How do you prevent multiple simultaneous logins for the same user?"

### Clear & Senior-Level Answer

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .sessionManagement(session -> session
            .maximumSessions(1)
            // Option A: Prevent new login (existing session stays)
            .maxSessionsPreventsLogin(true)
            // Option B (default): Expire old session (new login wins)
            // .maxSessionsPreventsLogin(false)
            .expiredSessionStrategy(event -> {
                HttpServletResponse response = event.getResponse();
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                response.getWriter().write("Session expired - logged in elsewhere");
            })
        )
        .build();
}

// CRITICAL: Must register this bean for concurrent session control to work
@Bean
public HttpSessionEventPublisher httpSessionEventPublisher() {
    return new HttpSessionEventPublisher();
}
```

**Why `HttpSessionEventPublisher` is required:** Spring Security's `SessionRegistry` needs to know when sessions are destroyed. Without this listener, the registry doesn't get notified of session destruction, leading to phantom sessions that block new logins.

**How it works internally:**
1. `ConcurrentSessionControlAuthenticationStrategy` is called during authentication
2. It queries `SessionRegistry` for existing sessions for this user
3. If max reached:
   - `maxSessionsPreventsLogin(true)`: throws `SessionAuthenticationException`
   - `maxSessionsPreventsLogin(false)`: marks the oldest session as expired
4. `ConcurrentSessionFilter` checks on each request if the session has been marked expired
5. If expired, forces logout and invokes `expiredSessionStrategy`

### Common Candidate Mistakes
- Forgetting to register `HttpSessionEventPublisher`
- Not knowing the difference between `maxSessionsPreventsLogin(true)` and `false`
- Not understanding that this only works with session-based authentication

---

## 3. SecurityContext Persistence and Async

### Interview Question
> "How does SecurityContext behave with async operations?"

### Clear & Senior-Level Answer

**The problem:** `@Async` methods run on a different thread. `SecurityContextHolder` uses `ThreadLocal`. New thread = no `SecurityContext`.

**Solutions:**

```java
// Solution 1: DelegatingSecurityContextExecutor
@Bean
public Executor asyncExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.initialize();
    return new DelegatingSecurityContextAsyncTaskExecutor(executor);
}

// Solution 2: SecurityContextHolder strategy
@PostConstruct
public void init() {
    SecurityContextHolder.setStrategyName(
        SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);
}
// WARNING: InheritableThreadLocal doesn't work well with thread pools
// because threads are reused and the inherited context becomes stale

// Solution 3: Manual propagation
CompletableFuture.supplyAsync(() -> {
    SecurityContext context = SecurityContextHolder.getContext();
    // ... use context
    return result;
}, new DelegatingSecurityContextExecutor(executor));
```

**Best practice:** Use `DelegatingSecurityContextAsyncTaskExecutor` — it captures the `SecurityContext` from the calling thread and sets it on the async thread before execution, then clears it after.

**Reactive (WebFlux):** There is no `ThreadLocal`. Security context is propagated through `Reactor Context`:

```java
ReactiveSecurityContextHolder.getContext()
    .map(SecurityContext::getAuthentication)
    .flatMap(auth -> ...);
```

---

# G. CSRF, CORS & Security Vulnerabilities

---

## 1. CSRF Attack Flow

### Interview Question
> "Explain a CSRF attack and Spring Security's protection mechanism."

### Clear & Senior-Level Answer

**Attack scenario:**
1. User logs into `bank.com` and has a valid session cookie
2. User visits `evil.com` which contains: `<form action="https://bank.com/transfer" method="POST"><input type="hidden" name="amount" value="10000"><input type="hidden" name="to" value="attacker"></form><script>document.forms[0].submit();</script>`
3. Browser automatically attaches the `bank.com` session cookie
4. Bank processes the transfer — the request looks legitimate

**Why the CSRF token stops this:** `evil.com` cannot read the CSRF token because:
- Same-Origin Policy prevents JavaScript on `evil.com` from reading `bank.com` responses
- The token is either in the DOM (form field) or a cookie that must match a header value
- The attacker can trigger the request but can't include the correct token

**SameSite cookies:** Modern browsers support `SameSite` attribute:
- `Strict`: Cookie never sent cross-site (breaks legitimate links)
- `Lax` (default in modern browsers): Sent on top-level GET navigations, not on POST
- `None`: Sent always (requires `Secure` flag)

**When CSRF protection is NOT needed:**
- APIs that use Bearer tokens (not cookies) for authentication — the attacker would need to steal the token, not forge the request
- Stateless APIs with `SessionCreationPolicy.STATELESS`

---

## 2. CORS Configuration

### Interview Question
> "How do you properly configure CORS in Spring Security?"

### Clear & Senior-Level Answer

**Common pitfall:** Configuring CORS on Spring MVC (`@CrossOrigin`) but not on Spring Security. **Spring Security's CORS filter runs before MVC** — if it rejects the preflight, the request never reaches your controller.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))
        .build();
}

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of(
        "https://app.example.com",
        "https://admin.example.com"
    ));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Requested-With"));
    config.setExposedHeaders(List.of("X-Total-Count"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

**Security mistakes:**
- `config.setAllowedOrigins(List.of("*"))` + `config.setAllowCredentials(true)` — this is **rejected by browsers**. If you allow credentials, you must specify exact origins.
- Using `setAllowedOriginPatterns(List.of("*"))` bypasses this check — **extremely dangerous** in production.
- Not restricting `allowedHeaders` — allows attackers to probe for internal headers.

---

## 3. Security Headers

### Interview Question
> "What security headers does Spring Security add by default?"

### Clear & Senior-Level Answer

```
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 0  (disabled — modern browsers use CSP instead)
Strict-Transport-Security: max-age=31536000; includeSubDomains  (only on HTTPS)
```

**Customizing headers:**
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .headers(headers -> headers
            .frameOptions(frame -> frame.sameOrigin())  // Allow iframes from same origin
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'; script-src 'self' cdn.example.com"))
            .httpStrictTransportSecurity(hsts -> hsts
                .maxAgeInSeconds(31536000)
                .includeSubDomains(true)
                .preload(true))
            .permissionsPolicy(permissions -> permissions
                .policy("camera=(), microphone=(), geolocation=()"))
        )
        .build();
}
```

**Content-Security-Policy is the most important header** for XSS prevention. It tells the browser which sources of content (scripts, styles, images) are allowed.

---

# H. Spring Boot Integration & Production Patterns

---

## 1. Auto-Configuration Behavior

### Interview Question
> "What does Spring Boot auto-configure for security, and what are the default pitfalls?"

### Clear & Senior-Level Answer

When `spring-boot-starter-security` is on the classpath, Spring Boot auto-configures:

1. A `SecurityFilterChain` that protects all endpoints (requires authentication)
2. A `UserDetailsService` with a generated password (printed in logs)
3. Form login and HTTP Basic authentication
4. CSRF protection enabled
5. Security headers
6. Session fixation protection

**Default pitfalls:**
- The generated password changes on restart — useless for anything beyond dev
- **All endpoints are protected** — including actuator, static resources, health checks
- CSRF is enabled — breaks API clients that don't handle tokens
- Form login is enabled — returns HTML login page for API 401 responses

**Production pattern — always define your own `SecurityFilterChain`:**
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        // Explicitly configure everything — don't rely on defaults
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(csrf -> csrf.disable())
            .build();
    }
}
```

---

## 2. Microservice Security Patterns

### Interview Question
> "How do you secure microservice-to-microservice communication?"

### Clear & Senior-Level Answer

**Pattern 1: Token Relay (API Gateway pattern)**
```
Client → API Gateway (validates JWT) → Downstream Service (trusts gateway)
```
The gateway validates the token and forwards it (or issues a new internal token) to downstream services.

**Pattern 2: Client Credentials for service-to-service**
```java
// Service A calls Service B using client credentials
@Bean
public WebClient webClient(OAuth2AuthorizedClientManager clientManager) {
    ServletOAuth2AuthorizedClientExchangeFilterFunction oauth2 =
        new ServletOAuth2AuthorizedClientExchangeFilterFunction(clientManager);
    oauth2.setDefaultClientRegistrationId("service-b");

    return WebClient.builder()
        .apply(oauth2.oauth2Configuration())
        .baseUrl("https://service-b.internal")
        .build();
}
```

**Pattern 3: mTLS (mutual TLS)**
- Both client and server present certificates
- No application-level tokens needed
- Strong identity guarantee
- Complex certificate management

**Production recommendation:** Use an **API Gateway** (Spring Cloud Gateway, Kong, Envoy) for external-facing authentication. Use **client credentials** or **mTLS** for internal service communication. Never trust internal network alone — assume breach (zero trust).

---

## 3. External IdP Integration (Keycloak)

### Interview Question
> "How do you integrate Spring Security with Keycloak?"

### Clear & Senior-Level Answer

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.example.com/realms/my-realm
          # Spring Security auto-fetches JWKS from:
          # https://keycloak.example.com/realms/my-realm/.well-known/openid-configuration
```

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2
            .jwt(jwt -> jwt.jwtAuthenticationConverter(keycloakJwtConverter()))
        )
        .build();
}

// Keycloak uses a nested "realm_access.roles" claim structure
@Bean
public JwtAuthenticationConverter keycloakJwtConverter() {
    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(jwt -> {
        Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
        if (realmAccess == null) return Collections.emptyList();

        @SuppressWarnings("unchecked")
        List<String> roles = (List<String>) realmAccess.get("roles");
        if (roles == null) return Collections.emptyList();

        return roles.stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
            .collect(Collectors.toList());
    });
    return converter;
}
```

**Key insight:** The main integration challenge with any IdP is **mapping claims to Spring Security authorities**. Each IdP structures its JWT differently. Keycloak uses `realm_access.roles`, Okta uses `groups`, Auth0 uses custom namespaced claims.

---

# I. Performance & Scalability

---

## 1. Authentication Caching

### Interview Question
> "How do you optimize authentication performance in Spring Security?"

### Clear & Senior-Level Answer

**Problem:** Every authenticated request triggers `UserDetailsService.loadUserByUsername()` if using session-based auth without proper caching, or JWT validation for stateless APIs.

**Solution 1: Cache UserDetails**

```java
@Service
public class CachingUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;
    private final Cache<String, UserDetails> cache;

    public CachingUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
        this.cache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(5))
            .build();
    }

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {
        return cache.get(username, key -> {
            User user = userRepository.findByEmailWithRoles(key)
                .orElseThrow(() -> new UsernameNotFoundException("Not found: " + key));
            return new CustomUserDetails(user);
        });
    }

    public void evictUser(String username) {
        cache.invalidate(username);
    }
}
```

**Caveat:** If a user's roles change, stale cached `UserDetails` means they keep old permissions until cache expires. Always evict on role change.

**Solution 2: Stateless JWT with claims-based authorities**

In stateless APIs, authorities are embedded in the JWT. No DB lookup needed per request. The trade-off: you can't revoke permissions instantly (must wait for token expiry).

**BCrypt cost factor impact:**

| Users/min | Cost 10 (~100ms) | Cost 12 (~400ms) | Cost 14 (~1.5s) |
|---|---|---|---|
| 100 | 10s of CPU | 40s of CPU | 150s of CPU |
| 1,000 | 1.6 min | 6.6 min | 25 min |

**Mitigation:** Only BCrypt on actual login events. Cache authenticated sessions. Don't re-hash on every request.

**Solution 3: Distributed session store (Redis)**

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

```yaml
spring:
  session:
    store-type: redis
  data:
    redis:
      host: redis.example.com
      port: 6379
```

This stores sessions in Redis, enabling:
- Horizontal scaling (any instance can serve any request)
- Session sharing across services
- Centralized session management

**Trade-off:** Adds Redis dependency and network latency (~1-2ms per request). For truly stateless APIs, JWT is simpler.

---

# J. Testing Security

---

## 1. @WithMockUser

### Interview Question
> "How do you test secured endpoints in Spring Security?"

### Clear & Senior-Level Answer

```java
@WebMvcTest(UserController.class)
@Import(SecurityConfig.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    // Test with mock authentication
    @Test
    @WithMockUser(username = "admin", roles = {"ADMIN"})
    void adminCanAccessUserList() throws Exception {
        when(userService.findAll()).thenReturn(List.of(new UserDto("user1")));

        mockMvc.perform(get("/api/users"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$[0].username").value("user1"));
    }

    // Test unauthorized access
    @Test
    void unauthenticatedCannotAccessUserList() throws Exception {
        mockMvc.perform(get("/api/users"))
            .andExpect(status().isUnauthorized());
    }

    // Test forbidden access
    @Test
    @WithMockUser(username = "user", roles = {"USER"})
    void regularUserCannotAccessAdmin() throws Exception {
        mockMvc.perform(get("/api/admin/dashboard"))
            .andExpect(status().isForbidden());
    }

    // Custom authentication with SecurityMockMvcRequestPostProcessors
    @Test
    void testWithJwtToken() throws Exception {
        mockMvc.perform(get("/api/users")
            .with(jwt()
                .jwt(j -> j
                    .subject("user1")
                    .claim("roles", List.of("ADMIN"))
                )
                .authorities(new SimpleGrantedAuthority("ROLE_ADMIN"))
            ))
            .andExpect(status().isOk());
    }

    // Test with custom UserDetails
    @Test
    void testWithCustomUserDetails() throws Exception {
        mockMvc.perform(get("/api/me")
            .with(user(new CustomUserDetails(testUser))))
            .andExpect(status().isOk());
    }
}
```

## 2. Custom @WithMockUser Annotation

```java
@Retention(RetentionPolicy.RUNTIME)
@WithSecurityContext(factory = WithMockAdminSecurityContextFactory.class)
public @interface WithMockAdmin {
    String username() default "admin@test.com";
}

public class WithMockAdminSecurityContextFactory
        implements WithSecurityContextFactory<WithMockAdmin> {

    @Override
    public SecurityContext createSecurityContext(WithMockAdmin annotation) {
        SecurityContext context = SecurityContextHolder.createEmptyContext();

        CustomUserDetails principal = new CustomUserDetails(
            User.builder()
                .email(annotation.username())
                .roles(Set.of(new Role("ADMIN")))
                .build()
        );

        Authentication auth = new UsernamePasswordAuthenticationToken(
            principal, null, principal.getAuthorities());
        context.setAuthentication(auth);
        return context;
    }
}

// Usage
@Test
@WithMockAdmin
void adminCanDeleteUser() throws Exception {
    mockMvc.perform(delete("/api/users/1"))
        .andExpect(status().isNoContent());
}
```

## 3. Integration Testing with Security

```java
@SpringBootTest
@AutoConfigureMockMvc
class SecurityIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void fullLoginFlow() throws Exception {
        // Test the actual authentication flow
        mockMvc.perform(post("/api/auth/login")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"email": "admin@test.com", "password": "password"}
                """))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.accessToken").exists());
    }

    @Test
    void csrfProtection() throws Exception {
        // Without CSRF token — should fail
        mockMvc.perform(post("/api/data"))
            .andExpect(status().isForbidden());

        // With CSRF token — should succeed (assuming authenticated)
        mockMvc.perform(post("/api/data")
            .with(csrf())
            .with(user("testuser")))
            .andExpect(status().isOk());
    }
}
```

### Common Candidate Mistakes
- Using `@WithMockUser` for JWT-based APIs (use `jwt()` post-processor instead)
- Not importing `SecurityConfig` in `@WebMvcTest` (security defaults apply instead)
- Testing authorization without testing both allowed and denied scenarios
- Not testing the authentication flow itself (only testing post-authentication)

---

# Real-World Scenario Section

---

## Scenario 1: Large-Scale Stateless API Security

### Interview Question
> "Design the security architecture for a stateless REST API serving 50,000 requests per second across 20 service instances."

### Clear Senior-Level Answer

**Architecture:**
```
Client → API Gateway (rate limiting, token validation)
       → Service Instances (stateless, JWT validation)
       → Authorization Server (Keycloak / custom)
```

**Key decisions:**
1. **JWT with RS256** — public key distributed to all services, private key only on auth server
2. **Short-lived access tokens (10 min)** — limits exposure from token theft
3. **JWKS endpoint** — services fetch public keys automatically, supports key rotation
4. **No session state** — `SessionCreationPolicy.STATELESS`, no Redis dependency for auth
5. **Claims-based authorization** — roles/permissions embedded in JWT, no DB lookup per request
6. **API Gateway handles:** rate limiting, IP blocking, token format validation
7. **Services handle:** signature verification, claim-based authorization

**Performance:**
- JWT validation is ~0.1ms (RSA signature verification)
- No network call for auth (public key is cached locally)
- At 50K req/s across 20 instances = 2,500 req/s per instance — trivial for JWT validation

**Trade-off:** Cannot revoke access instantly. Mitigation: short token lifetime + event-driven user suspension (push revocation events via message broker).

### Follow-Up Questions
- How do you handle token revocation at this scale?
- What happens if the JWKS endpoint is down?
- How do you handle authorization for complex business rules that can't be embedded in JWT claims?

---

## Scenario 2: Migration from Session-Based to JWT

### Interview Question
> "You need to migrate a monolith from session-based auth to JWT for a microservices split. How?"

### Clear Senior-Level Answer

**Phase 1: Dual-mode authentication**
- Keep session-based auth working
- Add JWT generation on login (return token in response body alongside session cookie)
- Configure Spring Security to accept both:

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED))
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
        .formLogin(Customizer.withDefaults())
        .build();
}
```

**Phase 2: Migrate clients**
- Update API clients to use Bearer tokens instead of session cookies
- Keep web UI on sessions (or migrate to token-based SPA)

**Phase 3: Remove sessions**
- Switch to `SessionCreationPolicy.STATELESS`
- Remove session infrastructure (Redis session store if used)
- Disable CSRF (no longer needed without cookies)

**Key risks:**
- Breaking existing clients during migration
- Losing instant logout capability (sessions → JWT)
- Cookie-dependent features (remember-me) need reimplementation

---

## Scenario 3: Multi-Tenant Security

### Interview Question
> "How do you implement multi-tenant security where users must only access their tenant's data?"

### Clear Senior-Level Answer

**Layer 1: Authentication — embed tenant in security context**

```java
public class TenantAwareJwtConverter implements Converter<Jwt, AbstractAuthenticationToken> {

    @Override
    public AbstractAuthenticationToken convert(Jwt jwt) {
        String tenantId = jwt.getClaimAsString("tenant_id");
        Collection<GrantedAuthority> authorities = extractAuthorities(jwt);

        TenantAwareAuthenticationToken token = new TenantAwareAuthenticationToken(
            jwt, authorities, tenantId);
        return token;
    }
}
```

**Layer 2: Data isolation — Hibernate filter or Spring Data**

```java
@Component
public class TenantFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
            throws ServletException, IOException {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth instanceof TenantAwareAuthenticationToken tenantAuth) {
            TenantContext.setCurrentTenant(tenantAuth.getTenantId());
        }
        try {
            filterChain.doFilter(request, response);
        } finally {
            TenantContext.clear();
        }
    }
}

// Hibernate filter for row-level isolation
@FilterDef(name = "tenantFilter", parameters = @ParamDef(name = "tenantId", type = String.class))
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
@Entity
public class Document {
    @Column(name = "tenant_id", nullable = false)
    private String tenantId;
}
```

**Layer 3: Authorization — tenant-scoped permission checks**

```java
@PreAuthorize("@tenantSecurity.belongsToTenant(authentication, #tenantId)")
@GetMapping("/api/tenants/{tenantId}/documents")
public List<Document> getDocuments(@PathVariable String tenantId) { ... }
```

---

## Scenario 4: Compromised Token Incident Response

### Interview Question
> "A JWT signing key has been compromised. What's your incident response?"

### Clear Senior-Level Answer

**Immediate actions (minutes):**
1. **Rotate the signing key** — generate new key pair, update auth server
2. **Invalidate ALL existing tokens** — since any token signed with the old key could be forged
3. **Force re-authentication** — update JWKS endpoint to only contain the new public key
4. **If using refresh tokens** — invalidate all refresh tokens in the database
5. **Alert monitoring** — flag any requests using old key's `kid` as potential attacks

**Medium-term (hours):**
1. Investigate how the key was compromised
2. Audit logs for suspicious token usage
3. Revoke and re-issue any client credentials
4. Check if any elevated privileges were granted

**Prevention:**
- Store signing keys in HSM (Hardware Security Module) or vault
- Implement automated key rotation (e.g., every 30 days)
- Use separate signing keys per environment (dev, staging, prod)
- Monitor for tokens with unexpected `kid` values

---

## Scenario 5: Enterprise RBAC Implementation

### Interview Question
> "Design an enterprise RBAC system with role hierarchy, fine-grained permissions, and audit trail."

### Clear Senior-Level Answer

**Data model:**
```
User ←→ UserRole ←→ Role ←→ RolePermission ←→ Permission
                       ↓
                  RoleHierarchy
```

**Implementation:**

```java
@Bean
public RoleHierarchy roleHierarchy() {
    return RoleHierarchyImpl.fromHierarchy("""
        ROLE_SUPER_ADMIN > ROLE_ADMIN
        ROLE_ADMIN > ROLE_MANAGER
        ROLE_MANAGER > ROLE_USER
        """);
}

// Custom permission evaluator for fine-grained checks
@Component
public class CustomPermissionEvaluator implements PermissionEvaluator {

    private final PermissionService permissionService;

    @Override
    public boolean hasPermission(Authentication authentication,
                                  Object targetDomainObject,
                                  Object permission) {
        String username = authentication.getName();
        String permStr = (String) permission;
        String targetType = targetDomainObject.getClass().getSimpleName();

        boolean granted = permissionService.hasPermission(username, targetType, permStr);

        // Audit trail
        auditService.log(username, targetType, permStr, granted);
        return granted;
    }

    @Override
    public boolean hasPermission(Authentication authentication,
                                  Serializable targetId,
                                  String targetType,
                                  Object permission) {
        // Object-level permission check
        return permissionService.hasPermission(
            authentication.getName(), targetId, targetType, (String) permission);
    }
}

// Usage in service
@PreAuthorize("hasPermission(#document, 'WRITE')")
public void updateDocument(Document document) { ... }

@PreAuthorize("hasPermission(#id, 'Document', 'READ')")
public Document getDocument(Long id) { ... }
```

---

# Bonus Sections

---

## Most Tricky Spring Security Interview Questions

**1. "What's the difference between `@Secured`, `@RolesAllowed`, and `@PreAuthorize`?"**

| Annotation | Source | SpEL Support | Enabled By |
|---|---|---|---|
| `@Secured` | Spring | No | `@EnableMethodSecurity(securedEnabled = true)` |
| `@RolesAllowed` | JSR-250 | No | `@EnableMethodSecurity(jsr250Enabled = true)` |
| `@PreAuthorize` | Spring | Yes | `@EnableMethodSecurity` (enabled by default) |

Always use `@PreAuthorize` — it's the most flexible and is enabled by default in Spring Security 6.

**2. "What happens when you call a `@PreAuthorize` method from another method in the same class?"**

Nothing. `@PreAuthorize` uses AOP proxies. Internal method calls bypass the proxy. This is the "self-invocation problem." Solution: inject the bean or use `@Lazy` self-reference.

**3. "Explain the difference between `SessionCreationPolicy.NEVER` and `STATELESS`."**

- `NEVER`: Spring Security won't *create* sessions, but will *use* one if it exists (created by other code)
- `STATELESS`: Spring Security won't create *or* use sessions at all. `SecurityContext` is never stored in a session.

**4. "Is `permitAll()` the same as not having security at all?"**

No. `permitAll()` still runs the full filter chain — CORS, CSRF, security headers, etc. The request is *allowed* but *processed*. Without security, none of these protections apply.

**5. "What happens to the `SecurityContext` when a request completes?"**

`SecurityContextHolderFilter` clears the `ThreadLocal` after the request. In session-based apps, the `SecurityContext` was previously saved to the `HttpSession` by `SecurityContextPersistenceFilter` (now deprecated in favor of explicit `SecurityContextRepository`).

**6. "Why does Spring Security convert `UsernameNotFoundException` to `BadCredentialsException`?"**

To prevent **username enumeration**. If the error message says "user not found" vs "wrong password," an attacker learns which usernames exist.

---

## Spring Security vs Keycloak

| Aspect | Spring Security | Keycloak |
|---|---|---|
| **What it is** | Security framework for Java apps | Standalone Identity Provider |
| **Runs as** | Part of your application | Separate server |
| **User management** | You implement it | Built-in (UI, API, federation) |
| **Protocol support** | Validates tokens | Issues and validates tokens (OIDC, SAML) |
| **SSO** | You must build it | Built-in |
| **Social login** | Supported via OAuth2 Client | Built-in with many providers |
| **Best for** | Custom auth logic, embedded security | Enterprise SSO, multi-app environments |

**They're complementary, not competitors.** Typical pattern: Keycloak as the IdP, Spring Security as the Resource Server.

---

## Production Hardening Checklist

- [ ] HTTPS enforced everywhere (HSTS header configured)
- [ ] CSRF enabled for session-based endpoints
- [ ] CORS restricted to specific origins
- [ ] Content-Security-Policy header configured
- [ ] BCrypt cost factor ≥ 12
- [ ] JWT expiry ≤ 15 minutes
- [ ] Refresh tokens stored server-side, rotated on use
- [ ] Signing keys stored in vault/HSM, rotated regularly
- [ ] Rate limiting on authentication endpoints
- [ ] Account lockout after failed attempts (with cooldown, not permanent)
- [ ] Security event logging (login, failed auth, privilege escalation)
- [ ] Dependency scanning (OWASP Dependency-Check)
- [ ] No sensitive data in JWT claims
- [ ] Actuator endpoints secured or disabled in production
- [ ] Error responses don't leak internal details
- [ ] Session fixation protection enabled
- [ ] Concurrent session control configured
- [ ] SQL injection protection (parameterized queries)
- [ ] Input validation on all user-supplied data
- [ ] `HttpOnly`, `Secure`, `SameSite` flags on cookies

---

## Interview Red Flags (What Interviewers Watch For)

- Using deprecated APIs (`WebSecurityConfigurerAdapter`, `authorizeRequests()`)
- Suggesting `NoOpPasswordEncoder` for any environment
- Not knowing the filter chain architecture
- Saying "just disable CSRF" without explaining when that's safe
- Storing passwords with MD5/SHA without salting
- Using `*` for CORS origins in production
- Confusing authentication and authorization
- Not considering token revocation when proposing JWT
- Implementing custom security instead of using Spring Security's extension points
- Not mentioning HTTPS as a baseline requirement

---

## Whiteboard Explanation Guides

### How to Explain SecurityFilterChain on a Whiteboard

```
Draw this flow:

[Tomcat]
   |
[DelegatingFilterProxy]
   |
[FilterChainProxy]
   |
   ├── SecurityFilterChain 1 ("/api/**")
   |    ├── CorsFilter
   |    ├── BearerTokenAuthenticationFilter
   |    ├── ExceptionTranslationFilter
   |    └── AuthorizationFilter
   |
   └── SecurityFilterChain 2 ("/**")
        ├── CsrfFilter
        ├── UsernamePasswordAuthenticationFilter
        ├── ExceptionTranslationFilter
        └── AuthorizationFilter

Key points to mention:
1. Only ONE chain matches per request
2. Filters within a chain execute in order
3. Each filter can short-circuit (stop the chain)
4. Authentication filters populate SecurityContext
5. Authorization filter is always last
```

### How to Explain Authentication Flow on a Whiteboard

```
Draw this flow:

[Login Request (POST /login)]
        |
[UsernamePasswordAuthenticationFilter]
        |
    creates UsernamePasswordAuthenticationToken (unauthenticated)
        |
[AuthenticationManager (ProviderManager)]
        |
    iterates providers
        |
[DaoAuthenticationProvider]
        |
    ├── UserDetailsService.loadUserByUsername()
    ├── PasswordEncoder.matches()
    |
    Success → Authenticated Token → SecurityContext
    Failure → AuthenticationException → FailureHandler
```

---

# Mock Interview Section

---

## Top 50 Senior-Level Spring Security Interview Questions

### Core Architecture (1-10)

1. How does Spring Security integrate with the Servlet container?
2. Explain the difference between `DelegatingFilterProxy`, `FilterChainProxy`, and `SecurityFilterChain`.
3. How do you configure multiple `SecurityFilterChain` beans and control their order?
4. What is the `SecurityContext`, and how does it flow through a request?
5. Explain the `AuthenticationManager` hierarchy and provider chain.
6. What are the `SecurityContextHolder` strategies and when would you change the default?
7. How does Spring Security handle the transition from unauthenticated to authenticated state?
8. What is the purpose of `AnonymousAuthenticationFilter`?
9. How does `ExceptionTranslationFilter` differentiate between authentication and authorization failures?
10. Explain the difference between `authorizeRequests()` and `authorizeHttpRequests()` in Spring Security 6.

### Authentication (11-20)

11. Walk through the complete flow of `UsernamePasswordAuthenticationFilter`.
12. When and why would you implement a custom `AuthenticationProvider`?
13. How does `DaoAuthenticationProvider` handle `UsernameNotFoundException`?
14. Explain the remember-me mechanism — hash-based vs persistent token.
15. How would you implement multi-factor authentication in Spring Security?
16. What is the role of `AuthenticationEntryPoint` vs `AuthenticationFailureHandler`?
17. How does HTTP Basic authentication work internally in Spring Security?
18. Explain the `Authentication` object lifecycle — unauthenticated to authenticated.
19. How do you configure different authentication mechanisms for different URL patterns?
20. How does Spring Security handle password upgrade from legacy hashing algorithms?

### Authorization (21-30)

21. Compare URL-based and method-level authorization — when would you use each?
22. Explain `@PreAuthorize` with SpEL — what variables are available?
23. Why doesn't `@PreAuthorize` work on internal method calls within the same class?
24. How does `AuthorizationManager` differ from the legacy `AccessDecisionManager`?
25. How do you implement a role hierarchy?
26. Explain `@PostFilter` and its performance implications.
27. How would you implement object-level permissions (ACL)?
28. How do you write a custom `PermissionEvaluator`?
29. What is the difference between `hasRole()` and `hasAuthority()`?
30. How does Spring Security handle authorization in reactive (WebFlux) applications?

### JWT & OAuth2 (31-40)

31. Explain JWT structure, validation, and how Spring Security processes Bearer tokens.
32. Compare HMAC (HS256) and RSA (RS256) for JWT signing — trade-offs?
33. Design a refresh token strategy with rotation and theft detection.
34. Explain the OAuth2 Authorization Code flow with PKCE.
35. When would you use Client Credentials flow?
36. How do you configure Spring Security as an OAuth2 Resource Server?
37. How do you map custom JWT claims to Spring Security authorities?
38. What are the challenges of JWT revocation and how do you handle them?
39. Explain key rotation for JWT signing — how do you achieve zero downtime?
40. What is the difference between a Resource Server and an Authorization Server?

### Security Vulnerabilities & Hardening (41-50)

41. Explain a CSRF attack and when it's safe to disable CSRF protection.
42. How do you properly configure CORS in Spring Security?
43. What security headers does Spring Security add by default?
44. How does session fixation protection work?
45. How do you implement concurrent session control?
46. Explain the `Content-Security-Policy` header and why it matters.
47. How do you prevent username enumeration attacks?
48. What is the DoS risk with BCrypt and how do you mitigate it?
49. How do you secure Spring Boot Actuator endpoints?
50. How do you test security configuration — what are the critical test scenarios?

---

## 5 Deep Architectural Discussion Questions

**1.** "You're building a platform that supports both a traditional web application (server-rendered) and a mobile app. The web app uses sessions, and the mobile app uses JWTs. Design the security architecture so both coexist within the same Spring Boot application. Discuss filter chain design, CSRF handling, and session management differences."

**2.** "Your organization is migrating from a monolithic Spring application with form-based login and session management to a microservices architecture with an API gateway. Design the security migration strategy. Address: backward compatibility during migration, token issuance, service-to-service authentication, and how to handle the transition without forcing all users to re-authenticate."

**3.** "Design a multi-tenant SaaS security architecture where: each tenant can have custom roles and permissions, users can belong to multiple tenants, tenant admins can manage their own users, and data isolation must be absolute. Discuss the Spring Security extension points you'd use and the data model."

**4.** "A security audit reveals that your JWT-based system has: no token revocation mechanism, no key rotation process, tokens with 24-hour expiry, and HMAC signing with the secret stored in application.properties. Propose a remediation plan addressing each issue. Prioritize the fixes and discuss trade-offs of each change."

**5.** "Design the authentication and authorization system for a healthcare application that must comply with HIPAA. Requirements: audit logging of all access to patient data, role-based AND attribute-based access control, break-the-glass emergency access, and session timeout after 15 minutes of inactivity. Discuss which Spring Security features you'd use and where you'd need custom implementations."

---

## 3 System Design Questions Involving Spring Security

**1. Secure API Gateway Design**

> "Design a secure API gateway that handles authentication for 15 downstream microservices, supports rate limiting per user, token exchange (converting external tokens to internal tokens), and circuit breaking when the auth server is down. What Spring components would you use?"

**Key discussion points:** Spring Cloud Gateway + Spring Security, `TokenRelay` filter, fallback authentication (cached public keys), token exchange endpoint, Redis-backed rate limiter.

**2. Enterprise SSO Platform**

> "Design an SSO platform for a company with 50+ internal applications. Requirements: single sign-on across all apps, centralized user management, support for SAML and OIDC, MFA enforcement policies per application, and session management (global logout). What's your architecture?"

**Key discussion points:** Keycloak/custom Authorization Server, Spring Security OAuth2 Client for each app, session management with back-channel logout, MFA step-up authentication.

**3. Financial Transaction Security**

> "Design the security layer for a financial API that processes $10M+ in daily transactions. Requirements: mutual TLS for partners, OAuth2 for third-party apps, per-transaction authorization with amount limits, non-repudiation (proof of who authorized what), and PCI DSS compliance. How does Spring Security fit?"

**Key discussion points:** mTLS with `X509AuthenticationFilter`, OAuth2 with rich authorization requests, `@PreAuthorize` with custom evaluator for amount limits, audit interceptors, field-level encryption.

---

*This guide covers the depth and breadth needed for senior-level Spring Security interviews. Study each section, understand the internal flows, and practice explaining them clearly — both verbally and on a whiteboard. The key differentiator at the senior level is not just knowing WHAT to configure, but understanding WHY the architecture works the way it does and WHEN to make trade-offs.*
