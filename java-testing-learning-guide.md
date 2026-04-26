# Java Testing: The Production-Grade Engineering Guide

> A deep, opinionated, real-world guide to testing Java / Spring Boot systems at production quality.
> Written for backend engineers fluent in Java 17/21, Spring Boot, PostgreSQL, JPA/Hibernate, AWS, Docker, and Kubernetes who have done basic unit testing but want to design testing strategies for real systems.

---

## Table of Contents

1. [Testing Fundamentals](#1-testing-fundamentals)
2. [JUnit 5](#2-junit-5)
3. [Mockito](#3-mockito)
4. [Unit Testing in Spring Boot](#4-unit-testing-in-spring-boot)
5. [Integration Testing](#5-integration-testing)
6. [Testing REST APIs](#6-testing-rest-apis)
7. [Testcontainers](#7-testcontainers)
8. [Database Testing](#8-database-testing)
9. [Advanced Testing Topics](#9-advanced-testing-topics)
10. [Testing Reactive Applications](#10-testing-reactive-applications)
11. [Test Design & Strategy](#11-test-design--strategy)
12. [CI/CD Integration](#12-cicd-integration)
13. [Production Best Practices](#13-production-best-practices)
14. [Real-World Scenarios](#14-real-world-scenarios)
15. [Comparisons & Trade-offs](#15-comparisons--trade-offs)
16. [Final Checklist](#16-final-checklist)

---

## 1. Testing Fundamentals

### 1.1 Why Testing Is Critical in Backend Systems

Backend systems are **stateful, concurrent, and remote**. Three properties that make bugs:

1. **Costly** — data corruption compounds; a bad write now is a support ticket in 6 months.
2. **Non-obvious** — race conditions, off-by-one in pagination, and timezone bugs do not surface until production scale.
3. **Hard to reproduce** — "works on my machine" is the default; without tests, every claim is anecdotal.

Tests exist to do three jobs:
- **Prove correctness** now.
- **Detect regressions** as the codebase evolves.
- **Document behavior** — a test is the most honest specification a system has.

A test suite is a **safety net for refactoring**. Without it, every change is a gamble. With it, senior engineers refactor boldly; without it, teams calcify around legacy code.

### 1.2 Types of Testing

| Type | Scope | Speed | Isolation | What It Proves |
|---|---|---|---|---|
| **Unit** | One class / pure function | ms | Total | Business logic is correct |
| **Integration** | Multiple classes + external resource (DB, HTTP, queue) | 100 ms–few s | Controlled | Components wire and cooperate correctly |
| **End-to-end (E2E)** | Whole system, black-box | seconds–minutes | Minimal | Critical user journeys work |
| **Contract** | Between services, shared schema | ms | Partial | Producer and consumer agree on API |
| **Performance** | System under load | minutes–hours | Controlled | System meets SLOs |
| **Security** | Auth, authz, input handling | varies | Varies | System resists misuse |

**Definitions worth enforcing:**
- A **unit test** exercises code **without** touching the network, filesystem, clock, DB, or another thread of your own making. If it does, it's an integration test — regardless of what label you stick on it.
- An **integration test** uses real infrastructure (a real DB via Testcontainers, real HTTP client vs. a real server) for the things you actually integrate with.
- An **E2E test** runs against a deployed system, ideally via the same interface a user would use.

### 1.3 The Test Pyramid

```
                     ▲
                    / \
                   / E2E\            few  (~5%)
                  /─────\
                 / Contract\         some (~10%)
                /─────────\
               /Integration\         more (~25%)
              /──────────\
             /    Unit    \          most (~60%)
            ────────────
```

**Rules of the pyramid:**
- **Most tests should be unit tests.** They're fast, deterministic, and localized — a failing unit test points at one class.
- **Integration tests** prove the wiring (Spring beans, DB queries, serialization).
- **E2E** tests are expensive; reserve them for **critical journeys** (login, checkout, payment).
- **Avoid the "ice cream cone"**: most tests being E2E because devs don't trust their unit tests. That's a symptom of poorly designed code, not a reason to skip unit tests.

**An "hourglass" shape** (lots of unit + lots of E2E, few integration) is also common and usually wrong — integration is where most real bugs hide.

### 1.4 Test Coverage vs Test Quality

Coverage is the **percentage of lines/branches exercised** by tests. It is:
- A **necessary** measurement — you can't test what you don't execute.
- A **poor sufficient** measurement — 100% coverage with no assertions proves nothing.

**What actually matters:**
- **Mutation score** (via PIT / Pitest): does the test fail when the code is deliberately mutated?
- **Behavior coverage**: does each documented behavior have a test?
- **Edge cases**: nulls, empties, boundaries, concurrency, failures.

> Aim for ~80% line coverage as a floor, not a ceiling. Make failing tests fail loudly and meaningfully. Coverage gates in CI are useful; coverage targets as performance reviews are counterproductive.

### 1.5 Deterministic vs Non-Deterministic Tests

A **deterministic** test always produces the same result given the same code. A **flaky** test passes sometimes.

Sources of non-determinism:
- `new Date()` / `Instant.now()` without injection.
- `Random` without a fixed seed.
- Relying on `HashMap` iteration order (unordered).
- `Thread.sleep` for synchronization instead of awaiting a condition.
- Shared state between tests (static variables, DB rows not cleaned up).
- External network calls (DNS, third-party APIs).
- Tests coupled to wall-clock time or timezone.

**The rule:** a flaky test is a bug in the test, not in "the universe." Flaky tests erode trust in the entire suite. Delete them or fix them — never retry them silently.

### 1.6 Common Misconceptions

| Misconception | Reality |
|---|---|
| "100% coverage means tested." | It means executed, not verified. |
| "Mocking everything makes tests fast and reliable." | It makes tests meaningless when integration is the real risk. |
| "Integration tests are slow, so skip them." | The cost of missing a wiring bug in prod dwarfs test time. |
| "Testing getters/setters is responsible." | It pads coverage and tests the language, not your code. |
| "I'll write tests after the feature." | 80% of post-hoc tests mirror the code's bugs. |
| "The test passed, the feature is correct." | Tests only prove absence of the bugs you thought of. |
| "Private methods need tests." | Test via public behavior; private is implementation detail. |

---

## 2. JUnit 5

JUnit 5 (Jupiter) is the default testing framework on the JVM. It is modular (`junit-jupiter-api`, `junit-jupiter-engine`, `junit-jupiter-params`), supports parallel execution, and has a richer extension model than JUnit 4.

### 2.1 Dependencies

```groovy
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    // Includes: junit-jupiter, spring-test, mockito-core, assertj, hamcrest, jsonpath
    testImplementation 'org.junit.jupiter:junit-jupiter-params'
}

test {
    useJUnitPlatform()
}
```

### 2.2 Basic Annotations

```java
class PriceCalculatorTest {

    private PriceCalculator calc;

    @BeforeAll
    static void initAll() {
        // Runs once before all tests in the class. Must be static (by default).
        // Use for: expensive resources (a shared Testcontainer when lifecycle demands it).
    }

    @BeforeEach
    void init() {
        // Runs before each test. Use for: creating a fresh SUT (system under test).
        calc = new PriceCalculator();
    }

    @Test
    void appliesDiscountForVipCustomer() {
        BigDecimal result = calc.priceFor(new VipCustomer(), new BigDecimal("100.00"));
        assertEquals(new BigDecimal("80.00"), result);
    }

    @AfterEach
    void tearDown() {
        // Runs after each test. Use for: cleanup that wasn't handled by lifecycle.
    }

    @AfterAll
    static void tearDownAll() {
        // Runs once after all tests.
    }
}
```

**Lifecycle default is `PER_METHOD`**: JUnit creates a new instance of the test class for each test. This is why `@BeforeAll` must be static.

To hold instance state across tests in a class (rarely useful):
```java
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class SharedInstanceTest { ... }
```

**When to use each:**
- `@BeforeEach` / `@AfterEach` — **default**. Fresh state per test.
- `@BeforeAll` / `@AfterAll` — shared expensive resources.
- `@TestInstance(PER_CLASS)` — rare. Only when you intentionally want shared state.

### 2.3 Assertions

#### JUnit's Built-in Assertions

```java
assertEquals(expected, actual);
assertEquals(expected, actual, "context message");
assertNotEquals(...);
assertTrue(condition);
assertFalse(condition);
assertNull(value);
assertNotNull(value);
assertSame(a, b);           // reference equality
assertArrayEquals(...);
assertIterableEquals(...);

// Exceptions
IllegalArgumentException ex = assertThrows(
    IllegalArgumentException.class,
    () -> calc.priceFor(null, TEN)
);
assertEquals("customer is required", ex.getMessage());

// Grouped assertions — all run even if one fails
assertAll(
    () -> assertEquals("Alice", user.getName()),
    () -> assertEquals(30, user.getAge()),
    () -> assertTrue(user.isActive())
);

// Timeouts
assertTimeout(Duration.ofMillis(200), () -> service.heavy());
assertTimeoutPreemptively(Duration.ofMillis(200), () -> service.heavy());
```

#### AssertJ — Preferred in Practice

AssertJ (bundled with `spring-boot-starter-test`) provides fluent, chainable assertions with far better failure messages.

```java
import static org.assertj.core.api.Assertions.*;

assertThat(result).isEqualTo(new BigDecimal("80.00"));
assertThat(user.getName()).startsWith("Al").isNotBlank();
assertThat(orders).hasSize(3)
    .extracting(Order::getStatus)
    .containsExactly(PAID, PAID, REFUNDED);

assertThat(map).containsEntry("role", "ADMIN").hasSize(2);

assertThatThrownBy(() -> calc.priceFor(null, TEN))
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessage("customer is required");

assertThat(instant).isCloseTo(Instant.now(), within(500, ChronoUnit.MILLIS));
```

**Rule:** use AssertJ for readability. Use `assertThrows` from JUnit only when you need to inspect the exception object directly.

### 2.4 Parameterized Tests

```java
@ParameterizedTest
@ValueSource(strings = {"  ", "\t", "\n", ""})
void blanksAreRejected(String input) {
    assertThatThrownBy(() -> validator.validateName(input))
        .isInstanceOf(ValidationException.class);
}

@ParameterizedTest
@CsvSource({
    "100.00, STANDARD, 100.00",
    "100.00, VIP,      80.00",
    "100.00, EMPLOYEE, 50.00"
})
void applyDiscountByTier(BigDecimal input, CustomerTier tier, BigDecimal expected) {
    assertThat(calc.priceFor(tier, input)).isEqualByComparingTo(expected);
}

@ParameterizedTest
@EnumSource(OrderStatus.class)
void allStatusesSerializable(OrderStatus status) {
    assertThat(json.write(status)).isNotNull();
}

@ParameterizedTest
@MethodSource("invalidEmails")
void rejectsInvalidEmails(String email) {
    assertThat(validator.isEmail(email)).isFalse();
}
static Stream<String> invalidEmails() {
    return Stream.of("a", "a@", "@b", "a@b", "a b@c.com");
}
```

**When to use:**
- **Equivalence classes** (valid/invalid email formats).
- **Table-driven tests** for pricing / transformations.
- **Regression bug lists**: add new case on each prod bug.

**When NOT to use:**
- When the scenarios are meaningfully different and each needs its own setup — write separate tests.

### 2.5 Nested Tests

```java
class OrderServiceTest {

    OrderService service;

    @BeforeEach
    void setup() { service = new OrderService(...); }

    @Nested
    class WhenCustomerIsNew {
        @Test void appliesWelcomeDiscount() { ... }
        @Test void sendsWelcomeEmail() { ... }
    }

    @Nested
    class WhenCustomerIsBanned {
        @Test void rejectsOrder() { ... }
        @Test void doesNotChargeCard() { ... }
    }
}
```

**Why nest?** Tests read like a spec: "OrderService > when customer is new > applies welcome discount." It naturally groups shared preconditions.

**When NOT to nest:** if the nested class is just one test — the nesting adds noise without structure.

### 2.6 Display Names and Tags

```java
@Test
@DisplayName("Rejects transfer when source balance is insufficient")
void t1() { ... }

@Test
@Tag("slow")
void longRunningReport() { ... }
```

Tags let you filter in CI:
```bash
./gradlew test -DincludeTags="!slow"
```

### 2.7 Lifecycle Summary

```
@BeforeAll
   ├── @BeforeEach
   │     @Test (one test)
   │   @AfterEach
   ├── @BeforeEach
   │     @Test (next test)
   │   @AfterEach
@AfterAll
```

### 2.8 JUnit 5 Best Practices

- One assertion concept per test — test names should describe a single behavior.
- Name tests `shouldX_whenY` or `whenY_thenX` or just `rejects_blank_names`. Pick a style; be consistent.
- Keep tests short: arrange / act / assert, visibly separated.
- Never call tests `test1`, `test2`. A test's name is half its documentation.
- Use `@Disabled("reason, ticket")` when skipping — never leave silent skips.
- Treat test code with the same rigor as production code: refactor duplication, extract helpers.

---

## 3. Mockito

Mockito creates **test doubles** — objects that stand in for real collaborators. It is essential for isolating a class under test from its dependencies.

### 3.1 Core Concepts

**Test double taxonomy (Meszaros / Martin Fowler):**

| Double | Purpose |
|---|---|
| **Dummy** | Passed around but never used (fills a parameter). |
| **Stub** | Returns canned answers to calls made during the test. |
| **Spy** | Wraps a real object, records interactions, allows partial stubbing. |
| **Mock** | Pre-programmed with expectations that form a specification of calls it will receive. |
| **Fake** | A working implementation but shortcutting production (e.g., in-memory repo). |

Mockito primarily builds **mocks** and **spies**.

### 3.2 Basic Mocking

```java
class OrderServiceTest {

    PaymentGateway gateway = mock(PaymentGateway.class);
    InventoryClient inventory = mock(InventoryClient.class);
    OrderRepository orders = mock(OrderRepository.class);

    OrderService service = new OrderService(gateway, inventory, orders);

    @Test
    void placesOrderSuccessfully() {
        // Given (stub)
        when(inventory.reserve("SKU-1", 2)).thenReturn(new Reservation("r-1"));
        when(gateway.charge(any(), any())).thenReturn(ChargeResult.success("tx-1"));
        when(orders.save(any())).thenAnswer(inv -> inv.getArgument(0));

        // When
        PlaceOrderResult result = service.place(new PlaceOrderCmd("SKU-1", 2, "card-1"));

        // Then
        assertThat(result.isSuccess()).isTrue();
        verify(inventory).reserve("SKU-1", 2);
        verify(gateway).charge(eq("card-1"), any());
        verify(orders).save(argThat(o -> o.getSku().equals("SKU-1")));
        verifyNoMoreInteractions(gateway, inventory);
    }
}
```

### 3.3 `@Mock`, `@InjectMocks`, `@ExtendWith(MockitoExtension.class)`

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock PaymentGateway gateway;
    @Mock InventoryClient inventory;
    @Mock OrderRepository orders;

    @InjectMocks OrderService service;

    @Test
    void t() {
        when(gateway.charge(any(), any())).thenReturn(ChargeResult.success("tx-1"));
        // ...
    }
}
```

- `@Mock` creates a mock per field.
- `@InjectMocks` constructs the SUT, injecting mocks via **constructor > setter > field**.
- `MockitoExtension` enables `@Mock` resolution and also **strict stubbing** (unused stubs fail the test — a feature, not a bug).

**Caveat on `@InjectMocks`:** it works via reflection and silently ignores missing matches. Prefer explicit constructor injection in tests — fewer surprises.

### 3.4 `when / thenReturn / thenThrow / thenAnswer`

```java
when(repo.findById(1L)).thenReturn(Optional.of(user));
when(repo.findById(2L)).thenReturn(Optional.empty());
when(client.call()).thenThrow(new TimeoutException());

// Dynamic response based on arguments
when(repo.findByEmail(anyString())).thenAnswer(inv -> {
    String email = inv.getArgument(0);
    return email.endsWith("@acme.com") ? Optional.of(internalUser) : Optional.empty();
});

// Consecutive calls
when(repo.nextId()).thenReturn(1L, 2L, 3L);

// Void methods
doThrow(new DuplicateException()).when(repo).delete(any());
doNothing().when(notifier).ping();
```

### 3.5 `verify()`

```java
verify(gateway).charge(eq("card-1"), any());
verify(gateway, times(1)).charge(any(), any());
verify(gateway, never()).refund(any());
verify(gateway, atLeastOnce()).charge(any(), any());

// Order matters
InOrder ord = inOrder(inventory, gateway, orders);
ord.verify(inventory).reserve(any(), anyInt());
ord.verify(gateway).charge(any(), any());
ord.verify(orders).save(any());

verifyNoMoreInteractions(gateway);       // nothing else called on this mock
verifyNoInteractions(auditLog);          // nothing called at all
```

### 3.6 Argument Matchers

```java
verify(repo).save(any());
verify(repo).save(any(User.class));
verify(repo).save(eq(user));
verify(repo).save(argThat(u -> u.isActive() && u.getEmail().endsWith("@acme.com")));

// All-or-nothing rule: if one argument uses a matcher, ALL must.
verify(repo).update(eq(1L), eq("name"));      // ok
verify(repo).update(1L, eq("name"));          // COMPILES but fails at runtime
```

Capturing arguments for inspection:

```java
ArgumentCaptor<Order> captor = ArgumentCaptor.forClass(Order.class);
verify(orders).save(captor.capture());
Order saved = captor.getValue();

assertThat(saved.getStatus()).isEqualTo(PLACED);
assertThat(saved.getTotal()).isEqualByComparingTo("42.00");
```

Or with the annotation:

```java
@Captor ArgumentCaptor<Order> orderCaptor;
```

### 3.7 Spies vs Mocks

A **mock** is a blank slate. Every method returns a default (null, 0, empty) unless stubbed.

A **spy** wraps a real object. Methods call the real implementation unless explicitly stubbed.

```java
List<String> spyList = spy(new ArrayList<>());
spyList.add("a");
spyList.add("b");

// Real method runs
assertThat(spyList).containsExactly("a", "b");

// Stub specific behavior
doReturn(42).when(spyList).size();
assertThat(spyList.size()).isEqualTo(42);
```

**When to use a spy:**
- Adding assertions on a real collaborator you don't own.
- Partial mocking of a legacy class you can't easily break up.

**When NOT to use a spy:**
- On classes *you* wrote. If you need to spy, the class probably has too many responsibilities — refactor.

### 3.8 Stubbing vs Behavior Verification

Two testing styles:

- **State-based**: set up inputs, run, assert *on the result*. Prefer this.
- **Interaction-based** (behavior): assert *which methods were called with what args*.

```java
// State-based — preferred when you have a clear output
Money total = calc.compute(order);
assertThat(total).isEqualByComparingTo("100.00");

// Interaction-based — required when the side effect IS the behavior
service.notify(user);
verify(emailSender).send(eq(user.getEmail()), any());
```

**Rule of thumb:** if the test is full of `verify()` calls, you may be over-specifying the implementation. Mocks should stub what's needed; verify only what matters to the behavior.

### 3.9 Common Pitfalls (IMPORTANT)

| Pitfall | Symptom | Fix |
|---|---|---|
| Mocking value objects / data classes | Brittle, pointless tests | Use real objects or builders |
| Mocking types you don't own (3rd party APIs) | False confidence — real API changes | Use contract tests / a real fake |
| Stubbing fields of static methods without `mockStatic` | `UnnecessaryStubbingException` | Use `Mockito.mockStatic` (scoped) |
| Forgetting argument matchers consistency | Runtime exception | Use `eq()` for all args if any use matchers |
| Over-verifying every call | Test breaks on refactor | Verify only the *behavioral contract* |
| Reusing mocks across tests | Test pollution | Recreate per test (JUnit default) or `reset(mock)` sparingly |
| Using `when().thenReturn()` with `void` | Compile error or confusion | Use `doX().when()` for void |
| Mocking `final` / `static` without `mockito-inline` | `MockitoException` | Add `mockito-inline` in recent versions (enabled by default since 5.x) |
| Testing the mock, not the SUT | Meaningless green tests | Focus on the SUT's output, not the mock's config |
| `UnnecessaryStubbingException` in strict mode | Over-stubbed tests | Delete unused stubs; they are a smell |

### 3.10 Mocking Static Methods

```java
try (MockedStatic<Clock> clockMock = mockStatic(Clock.class)) {
    clockMock.when(Clock::systemUTC).thenReturn(fixed);
    // ... inside try block, Clock.systemUTC() returns fixed
}
```

**Use sparingly.** Mocking statics is a code smell. Prefer to inject a `Clock` bean — then stub it normally.

### 3.11 Mockito BDD Syntax

```java
import static org.mockito.BDDMockito.*;

given(repo.findById(1L)).willReturn(Optional.of(user));
// when
User got = service.get(1L);
// then
then(repo).should().findById(1L);
```

Same mechanics, more readable for BDD-style tests.

---

## 4. Unit Testing in Spring Boot

### 4.1 What Should Be Unit Tested

**Yes:**
- Services with real business logic.
- Validators, mappers, pricing calculators, domain rules.
- Pure functions.
- Complex conditional logic.
- Anything with non-trivial branches.

**No (or rarely):**
- DTOs, entities without behavior.
- Trivial getters/setters.
- Framework configuration classes.
- Controllers (prefer `@WebMvcTest`).
- Repositories (prefer `@DataJpaTest` or Testcontainers).

### 4.2 Don't Load Spring Context for Unit Tests

A Spring unit test should run in **milliseconds**. Loading the application context takes **seconds** — that's integration territory. The moment you use `@SpringBootTest`, you've left unit-test land.

**Bad (slow, over-scoped):**

```java
@SpringBootTest
class PriceCalculatorTest { ... }
```

**Good (fast, isolated):**

```java
@ExtendWith(MockitoExtension.class)
class PriceCalculatorTest {

    @Mock DiscountRepository discounts;
    @InjectMocks PriceCalculator calc;

    @Test
    void applies_vip_discount() {
        when(discounts.forTier(VIP)).thenReturn(new Discount(0.20));
        assertThat(calc.priceFor(VIP, new BigDecimal("100")))
            .isEqualByComparingTo("80.00");
    }
}
```

### 4.3 Testing Services (Business Logic)

```java
@ExtendWith(MockitoExtension.class)
class MoneyTransferServiceTest {

    @Mock AccountRepository accounts;
    @Mock AuditLog audit;

    @InjectMocks MoneyTransferService service;

    @Test
    void transfers_money_between_two_accounts() {
        Account from = new Account(1L, "USD", new BigDecimal("100.00"));
        Account to   = new Account(2L, "USD", new BigDecimal("50.00"));
        when(accounts.findById(1L)).thenReturn(Optional.of(from));
        when(accounts.findById(2L)).thenReturn(Optional.of(to));

        service.transfer(1L, 2L, new BigDecimal("30.00"));

        assertThat(from.getBalance()).isEqualByComparingTo("70.00");
        assertThat(to.getBalance()).isEqualByComparingTo("80.00");
        verify(accounts).save(from);
        verify(accounts).save(to);
        verify(audit).record(any(TransferEvent.class));
    }

    @Test
    void rejects_transfer_when_balance_is_insufficient() {
        Account from = new Account(1L, "USD", new BigDecimal("10.00"));
        Account to   = new Account(2L, "USD", new BigDecimal("50.00"));
        when(accounts.findById(1L)).thenReturn(Optional.of(from));
        when(accounts.findById(2L)).thenReturn(Optional.of(to));

        assertThatThrownBy(() -> service.transfer(1L, 2L, new BigDecimal("30.00")))
            .isInstanceOf(InsufficientFundsException.class);

        verify(accounts, never()).save(any());
        verifyNoInteractions(audit);
    }
}
```

### 4.4 Best Practices

- **Constructor injection** in production code: makes mocking and wiring explicit.
- **Pure functions** where possible: they're trivially testable.
- **Inject `Clock`** instead of calling `Instant.now()` in code.
- **Separate I/O from logic**: a `PricingService` (pure) + a `PricingFacade` (I/O). Test pricing at unit level, facade at integration.
- **Avoid `@Autowired` field injection** — untestable without reflection.
- **Don't unit-test `@Controller`** — tests the framework, not your logic. Use `@WebMvcTest`.

---

## 5. Integration Testing

Integration tests verify that **components cooperate**: beans wire, SQL compiles, JSON serializes, security rules fire.

Spring Boot provides **test slices**: minimal, focused contexts that load only a subset of beans.

### 5.1 `@SpringBootTest`

The full application context. Slowest, most powerful.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class CheckoutFlowIT {

    @LocalServerPort int port;
    @Autowired TestRestTemplate client;

    @Test
    void end_to_end_checkout() {
        ResponseEntity<OrderDto> resp = client.postForEntity(
            "http://localhost:" + port + "/api/orders",
            new PlaceOrderRequest("SKU-1", 2), OrderDto.class);

        assertThat(resp.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(resp.getBody().status()).isEqualTo("PLACED");
    }
}
```

**When:** you need the full stack — controller, security filter, service, repo, DB.

**WebEnvironment options:**
- `MOCK` (default) — no real server; use with `MockMvc`.
- `RANDOM_PORT` — real server on a free port.
- `DEFINED_PORT` — use `server.port`.
- `NONE` — no web environment.

### 5.2 `@WebMvcTest`

Loads **only the web layer**: `@Controller`, `@RestController`, `@ControllerAdvice`, filters, converters. No services, no repositories.

```java
@WebMvcTest(UserController.class)
@Import(SecurityConfig.class)
class UserControllerTest {

    @Autowired MockMvc mvc;
    @MockBean UserService userService;

    @Test
    void getUser_returns_200() throws Exception {
        when(userService.findById(1L)).thenReturn(new UserDto(1L, "alice@acme.com"));

        mvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.email").value("alice@acme.com"));
    }

    @Test
    void postUser_rejects_invalid_payload() throws Exception {
        mvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    { "email": "not-an-email" }
                """))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors[0].field").value("email"));
    }
}
```

**When:** testing controller wiring, validation, serialization, exception handlers, security filters. **Fast.**

**When NOT:** anything downstream of the controller (DB, services — just mock them).

### 5.3 `@DataJpaTest`

Loads **only the JPA layer**: entities, repositories, `EntityManager`. By default, uses an in-memory DB (H2) and runs each test in a rolled-back transaction.

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryIT {

    @Autowired UserRepository users;

    @Test
    void finds_user_by_email() {
        users.save(new User("alice@acme.com", "Alice"));
        Optional<User> got = users.findByEmail("alice@acme.com");
        assertThat(got).isPresent();
    }
}
```

**`Replace.NONE`** uses the configured DB (typically Testcontainers in prod-grade test setups). Without it, Spring swaps in H2 — convenient but risky (see §8.2).

**When:** testing repository methods, queries, mappings.

**Other slices worth knowing:**

| Slice | Loads |
|---|---|
| `@DataR2dbcTest` | R2DBC repositories only |
| `@WebFluxTest` | Reactive web layer |
| `@JsonTest` | Jackson/Gson serializers |
| `@RestClientTest` | `RestTemplate` / `WebClient` + MockServer |
| `@JdbcTest` | `JdbcTemplate` |
| `@DataMongoTest` | Mongo repositories |

### 5.4 Trade-offs

| Scope | Speed | Coverage | Use |
|---|---|---|---|
| Unit (no Spring) | ms | Pure logic | Most tests |
| `@WebMvcTest` | hundreds of ms | Web layer | Controller/validation/security |
| `@DataJpaTest` | hundreds of ms | Persistence | Repo methods, custom queries |
| `@SpringBootTest` | seconds | Whole app | Smoke tests, critical flows |

**Rule:** use the smallest slice that proves the thing you care about.

---

## 6. Testing REST APIs

### 6.1 MockMvc

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mvc;
    @MockBean OrderService orders;
    @Autowired ObjectMapper mapper;

    @Test
    @WithMockUser(roles = "CUSTOMER")
    void create_order_returns_201_with_location() throws Exception {
        when(orders.place(any())).thenReturn(new Order(42L, "PLACED"));

        mvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(mapper.writeValueAsString(new PlaceOrderReq("SKU-1", 2))))
            .andExpect(status().isCreated())
            .andExpect(header().string("Location", "/api/orders/42"))
            .andExpect(jsonPath("$.id").value(42))
            .andExpect(jsonPath("$.status").value("PLACED"));
    }

    @Test
    @WithMockUser(roles = "CUSTOMER")
    void rejects_invalid_request() throws Exception {
        mvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    { "sku": "", "quantity": 0 }
                """))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors", hasSize(2)))
            .andExpect(jsonPath("$.errors[*].field",
                containsInAnyOrder("sku", "quantity")));
    }

    @Test
    void anonymous_is_401() throws Exception {
        mvc.perform(get("/api/orders/1"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = "CUSTOMER")
    void service_error_is_handled_cleanly() throws Exception {
        when(orders.place(any())).thenThrow(new InventoryShortageException("SKU-1"));

        mvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    { "sku": "SKU-1", "quantity": 2 }
                """))
            .andExpect(status().isConflict())
            .andExpect(jsonPath("$.code").value("inventory_shortage"));
    }
}
```

**MockMvc tests validate:**
- Request mapping (path, method, content-type).
- Request body deserialization and validation.
- Response serialization, status, headers.
- Exception handling via `@ControllerAdvice`.
- Security filters and rules.

### 6.2 `WebTestClient` for Reactive Apps

```java
@WebFluxTest(OrderController.class)
class OrderControllerFluxTest {

    @Autowired WebTestClient client;
    @MockBean OrderService orders;

    @Test
    void create_order() {
        when(orders.place(any())).thenReturn(Mono.just(new Order(42L, "PLACED")));

        client.post().uri("/api/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(new PlaceOrderReq("SKU-1", 2))
            .exchange()
            .expectStatus().isCreated()
            .expectHeader().location("/api/orders/42")
            .expectBody()
              .jsonPath("$.id").isEqualTo(42)
              .jsonPath("$.status").isEqualTo("PLACED");
    }
}
```

`WebTestClient` also works against a running server — use it in `@SpringBootTest(webEnvironment = RANDOM_PORT)` setups for reactive E2E.

### 6.3 Validation Testing

```java
@Test
void email_is_required() throws Exception {
    mvc.perform(post("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                { "name": "Alice" }
            """))
        .andExpect(status().isBadRequest())
        .andExpect(jsonPath("$.errors[?(@.field=='email')].message")
            .value("must not be blank"));
}
```

### 6.4 Error-Handling Testing

Always have a `@ControllerAdvice` that translates exceptions to consistent error payloads, and **test it**:

```java
@Test
void domain_exception_becomes_422() throws Exception {
    when(orders.place(any())).thenThrow(new BusinessRuleViolation("cart_empty"));

    mvc.perform(post("/api/orders").contentType(JSON).content("{}"))
        .andExpect(status().isUnprocessableEntity())
        .andExpect(jsonPath("$.code").value("cart_empty"));
}
```

### 6.5 Best Practices

- **Do not serialize JSON by hand**: use `ObjectMapper` or `@JsonTest`.
- **Test validation** — it is part of your API contract.
- **Test error responses** — they are part of your API contract.
- **Do not test every field** in happy path — test the contract (shape), not the data (mocked).
- Use **`@TestPropertySource`** or `@ActiveProfiles("test")` for env-specific settings.

---

## 7. Testcontainers

### 7.1 What Testcontainers Is

Testcontainers is a Java library that **starts real Docker containers** for use in tests: databases, queues, caches, mock HTTP servers, full apps. It manages the lifecycle — pull image, start, wait until healthy, stop and remove — automatically.

**Why it matters:**
- Runs the **same infra** in tests as in prod. An in-memory H2 is not PostgreSQL. An embedded Kafka is not real Kafka. Differences lurk.
- **Disposable**: tests can't pollute each other.
- **CI-friendly**: works in GitHub Actions, GitLab CI, Jenkins agents — anywhere Docker runs.

**Without Testcontainers**, you either:
- Use H2/HSQLDB (fast but *different dialect*, different query semantics, no JSONB, no triggers).
- Maintain docker-compose for dev + separate test infra (drift).
- Run against a shared test DB (flaky, slow, racy).

Testcontainers collapses all of these problems.

### 7.2 Dependencies

```groovy
dependencies {
    testImplementation 'org.testcontainers:junit-jupiter:1.20.6'
    testImplementation 'org.testcontainers:postgresql:1.20.6'
    testImplementation 'org.testcontainers:kafka:1.20.6'
    testImplementation 'org.springframework.boot:spring-boot-testcontainers'
}
```

### 7.3 PostgreSQL Container

```java
@SpringBootTest
@Testcontainers
class OrderServiceIT {

    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("app")
        .withUsername("app")
        .withPassword("app");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
    }

    @Autowired OrderService service;

    @Test
    void persists_and_reads_order() {
        Order saved = service.place(new PlaceOrderCmd("SKU-1", 2));
        assertThat(service.findById(saved.getId())).isPresent();
    }
}
```

**How it works:**
- `@Testcontainers` + `@Container` tells JUnit to start `pg` before tests and stop after.
- `static` + `@Container` → started once per class (shared across tests).
- `@DynamicPropertySource` plugs the random port/URL into Spring's `Environment` before context starts.

### 7.4 Shared Container Pattern (Fast)

Starting PostgreSQL once per test class is good; starting it once per **JVM** is better.

```java
abstract class AbstractIT {

    static final PostgreSQLContainer<?> PG = new PostgreSQLContainer<>("postgres:16-alpine")
        .withReuse(true);

    static {
        PG.start();
    }

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", PG::getJdbcUrl);
        r.add("spring.datasource.username", PG::getUsername);
        r.add("spring.datasource.password", PG::getPassword);
    }
}
```

Extend `AbstractIT` from every integration test. The JVM shuts down the container at exit.

**`withReuse(true)` + `~/.testcontainers.properties` with `testcontainers.reuse.enable=true`** caches the container across test runs in dev (huge speedup). Disable in CI.

### 7.5 Redis Container

```java
@Container
static GenericContainer<?> redis = new GenericContainer<>(DockerImageName.parse("redis:7"))
    .withExposedPorts(6379);

@DynamicPropertySource
static void props(DynamicPropertyRegistry r) {
    r.add("spring.data.redis.host", redis::getHost);
    r.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
}
```

### 7.6 Kafka Container

```java
@Container
static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"));

@DynamicPropertySource
static void props(DynamicPropertyRegistry r) {
    r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
}

@Test
void publishes_and_consumes_events() {
    producer.send(new OrderPlacedEvent("o-1"));
    await().atMost(Duration.ofSeconds(5))
        .untilAsserted(() -> assertThat(sink.received()).hasSize(1));
}
```

For **Kafka consumers** that eventually succeed, use **Awaitility** — never `Thread.sleep`.

### 7.7 Service-Connection Style (Spring Boot 3.1+)

```java
@Testcontainers
@SpringBootTest
class OrderServiceIT {

    @Container @ServiceConnection
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");

    // Spring Boot auto-configures datasource / R2DBC / etc.
}
```

`@ServiceConnection` eliminates the `@DynamicPropertySource` boilerplate for many common services. Preferred for new projects.

### 7.8 Lifecycle Management

| Setup | Lifecycle | Speed |
|---|---|---|
| Instance field `@Container` | Per test | Slowest |
| `static @Container` | Per class | Fast |
| `static { start(); }` without `@Container` | Per JVM | Fastest |
| `withReuse(true)` + local config | Across runs (dev) | Fastest |

**Rule:** always run one DB container per test class at minimum; prefer per-JVM shared container for speed.

**Cleanup strategy** per test (when sharing a container):
- Transactional rollback (`@Transactional` + `@Rollback`) — cleanest.
- `@Sql(scripts = {"/cleanup.sql"}, executionPhase = BEFORE_TEST_METHOD)`.
- Explicit `TRUNCATE` in `@BeforeEach`.
- **Never** rely on test ordering.

### 7.9 Best Practices

- Pin image versions — `postgres:16-alpine`, not `postgres:latest`.
- Use the **same version as production**.
- Run migrations via **Flyway/Liquibase** at test startup — the schema matches prod.
- Don't boot `docker-compose` from tests — Testcontainers handles orchestration.
- Capture container logs on failure for debugging: `container.getLogs()`.
- In CI, ensure the runner has Docker (or use a DIND setup).

---

## 8. Database Testing

### 8.1 Testing Repositories

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
@Testcontainers
class OrderRepositoryIT {

    @Container @ServiceConnection
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired OrderRepository orders;
    @Autowired TestEntityManager em;

    @Test
    void finds_orders_placed_after() {
        em.persistAndFlush(new Order("o-1", Instant.parse("2026-01-01T00:00:00Z"), PLACED));
        em.persistAndFlush(new Order("o-2", Instant.parse("2026-02-01T00:00:00Z"), PLACED));

        List<Order> after = orders.findPlacedAfter(Instant.parse("2026-01-15T00:00:00Z"));

        assertThat(after).extracting(Order::getCode).containsExactly("o-2");
    }
}
```

**`TestEntityManager`** bypasses the repository abstraction for setup — use it to arrange state without triggering your repo's own code.

### 8.2 In-Memory DB vs Real DB

| Aspect | H2 / HSQLDB | Testcontainers Postgres |
|---|---|---|
| Speed | Very fast | Fast after reuse |
| SQL dialect match | Partial | Exact |
| JSONB / arrays / enums | Limited / none | Full |
| Custom types, triggers, functions | Weak | Full |
| CI infra requirements | None | Docker |
| Test realism | Low | High |
| Catches prod bugs | Sometimes | Usually |

**The rule:** if you're using anything beyond ANSI SQL (JSONB, text search, window functions, enums, triggers, stored procedures, concurrency semantics, specific locking), **use Testcontainers**. H2 will lie to you about the behavior, and bugs will find production.

Acceptable H2 usage: small libraries with vanilla SQL, legacy codebases with low test ROI, or when Docker simply isn't available.

### 8.3 Data Setup Strategies

**1. Per-test insert via repository / JPA:**
```java
users.save(new User("alice", ...));
```
Pros: explicit, type-safe. Cons: verbose for complex graphs.

**2. SQL scripts with `@Sql`:**
```java
@Test
@Sql("/fixtures/orders.sql")
void finds_orders() { ... }
```
Pros: close to DB, fast to write. Cons: drifts from entity model.

**3. Test data builders (preferred):**
```java
User u = a(UserBuilder.class).withEmail("alice@acme.com").withRole(ADMIN).build();
```
See §9.1.

**4. Snapshots / fixtures (YAML/JSON):**
Useful for large, stable graphs. Careful: they ossify schema changes.

### 8.4 Transactions in Tests

Spring rolls back `@Transactional` tests by default — perfect for leaving the DB clean.

```java
@SpringBootTest
@Transactional
class OrderFlowIT {

    @Test
    void places_an_order() {
        // DB writes happen inside a TX that will be rolled back
    }
}
```

**Caveats:**
- If the code-under-test opens its own TX and commits, rollback won't undo it.
- `@Transactional` affects **test-driven** changes; async workers or separate threads won't see them unless you flush.
- For controller-level tests that commit (MVC + real service calls), prefer explicit cleanup (`@Sql`, `TRUNCATE`).

**Rollback off when you need to see results:**
```java
@Test
@Rollback(false)
@Sql(scripts = "/cleanup.sql", executionPhase = AFTER_TEST_METHOD)
void debug_scenario() { ... }
```

### 8.5 Testing Queries Precisely

When testing a query, construct **tight fixtures**: the data that should match + the data that should not.

```java
@Test
void excludes_soft_deleted() {
    em.persistAndFlush(activeUser("a@x"));
    em.persistAndFlush(softDeletedUser("b@x"));
    List<User> result = users.findAllActive();
    assertThat(result).extracting(User::getEmail).containsExactly("a@x");
}
```

Include the negative case — without it, you haven't proven filtering works.

---

## 9. Advanced Testing Topics

### 9.1 Test Data Builders

Builders replace `new ThingWithFifteenFields(null, null, ..., null)` with readable construction:

```java
public class UserBuilder {
    private Long id;
    private String email = "user-" + UUID.randomUUID() + "@acme.com";
    private String name = "User";
    private Role role = Role.CUSTOMER;
    private Instant createdAt = Instant.now();

    public static UserBuilder aUser() { return new UserBuilder(); }
    public UserBuilder withEmail(String e) { this.email = e; return this; }
    public UserBuilder asAdmin() { this.role = Role.ADMIN; return this; }
    public User build() { return new User(id, email, name, role, createdAt); }
}

// In tests
User admin = aUser().asAdmin().build();
User alice = aUser().withEmail("alice@acme.com").build();
```

**Benefits:**
- Tests express *what matters* — the rest has sensible defaults.
- Schema changes update the builder, not 200 test files.
- Randomized defaults prevent accidental assumptions (e.g., two users sharing the same email).

**When NOT to use:** tiny value objects with 1–2 fields. Just construct directly.

### 9.2 Fixtures

A **fixture** is the arranged state for a test. Good fixtures are:
- Minimal — include only what affects behavior.
- Explicit — don't hide relevant data in a base class.
- Reusable — but not shared mutably.

Use **builders for scenarios** and **SQL dumps for very large reference data** (lookups, configs).

### 9.3 Contract Testing (Overview)

When multiple services must agree on an API, integration tests of each service do not detect schema drift. **Contract tests** do.

**Two main approaches:**
- **Spring Cloud Contract** — producer defines contracts, consumers test against generated stubs.
- **Pact** — consumer writes expectations; producer verifies them.

**Why it matters:** in microservices, the most common prod bug is "team A changed a field; team B still expects the old shape." CI contract tests prevent this.

**When to use:** you have ≥3 services, independent teams, independent deploy cadence.

**When NOT to use:** monolith, single team, low change rate.

### 9.4 Performance Testing Basics

Categories:
- **Load** — steady traffic at expected volume.
- **Stress** — ramp until failure.
- **Spike** — sudden jumps.
- **Soak** — long-running, memory leaks.

Tools:
- **Gatling** — Scala DSL, JVM-native, great for JVM shops.
- **k6** — JS scripts, CI-friendly.
- **JMeter** — classic, heavy, GUI-first (use non-GUI).

**Rule:** performance tests are not unit tests. Run them in a dedicated environment, not in every PR. Measure p50/p95/p99 latencies and throughput.

**Micro-benchmarks:** use **JMH** — never `System.currentTimeMillis()` loops.

### 9.5 Security Testing Basics

- **Authentication**: hit protected endpoints anonymously → 401.
- **Authorization**: hit an endpoint as the wrong role → 403.
- **Input handling**: SQL injection payloads (`' OR '1'='1`), XSS, path traversal.
- **Mass assignment**: POST `{ "role": "ADMIN" }` on a `/users` endpoint and ensure it is ignored.
- **Rate limiting**: flood the endpoint and assert 429.
- **Dependency scanning**: OWASP Dependency-Check, Snyk, GitHub Dependabot.

**Integrate** ZAP or Burp Suite scans into a nightly pipeline for API fuzzing — but tests inside the suite should enforce the basics.

---

## 10. Testing Reactive Applications (WebFlux)

### 10.1 Testing `Mono` and `Flux`

```java
@Test
void returns_user_mono() {
    Mono<User> result = service.findById(1L);

    StepVerifier.create(result)
        .expectNextMatches(u -> u.getEmail().equals("a@x"))
        .verifyComplete();
}

@Test
void streams_users_in_order() {
    StepVerifier.create(service.streamAll().take(3))
        .expectNextCount(3)
        .verifyComplete();
}

@Test
void errors_on_missing_user() {
    when(repo.findById(99L)).thenReturn(Mono.empty());

    StepVerifier.create(service.findById(99L))
        .expectError(NotFoundException.class)
        .verify();
}
```

### 10.2 `StepVerifier`

`StepVerifier` is the canonical way to test reactive streams.

```java
StepVerifier.create(flux)
    .expectSubscription()
    .expectNext("a")
    .expectNext("b", "c")
    .expectNextMatches(s -> s.startsWith("d"))
    .expectNoEvent(Duration.ofMillis(100))
    .thenCancel()
    .verify();
```

**Virtual time** (for testing time-based operators without real waiting):

```java
StepVerifier.withVirtualTime(() -> Flux.interval(Duration.ofSeconds(1)).take(3))
    .expectSubscription()
    .expectNoEvent(Duration.ofSeconds(1))
    .expectNext(0L)
    .thenAwait(Duration.ofSeconds(1))
    .expectNext(1L)
    .thenAwait(Duration.ofSeconds(1))
    .expectNext(2L)
    .verifyComplete();
```

### 10.3 Testing Reactive Endpoints

```java
@WebFluxTest(UserController.class)
class UserControllerFluxTest {

    @Autowired WebTestClient client;
    @MockBean UserService userService;

    @Test
    void get_user() {
        when(userService.findById(1L)).thenReturn(Mono.just(new UserDto(1L, "a@x")));

        client.get().uri("/api/users/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody().jsonPath("$.email").isEqualTo("a@x");
    }

    @Test
    void streams_users() {
        when(userService.streamAll()).thenReturn(Flux.just(u1, u2, u3));

        client.get().uri("/api/users").accept(MediaType.APPLICATION_NDJSON)
            .exchange()
            .expectStatus().isOk()
            .returnResult(UserDto.class).getResponseBody()
            .as(StepVerifier::create)
            .expectNextCount(3)
            .verifyComplete();
    }
}
```

### 10.4 Pitfalls

- **Not subscribing** — reactive tests must assert via `StepVerifier` or `.block()` inside a test (the one place `.block()` is OK).
- **Forgetting `.verify()`** — the test passes trivially without it.
- **Mixing real time and virtual time** — `withVirtualTime` only works if the supplier creates the `Publisher` lazily.
- **Mocking returning `null`** instead of `Mono.empty()` — cascades into NPEs.
- **Blocking inside operators** — BlockHound in tests catches this. Install it.

```java
@BeforeAll
static void setup() { BlockHound.install(); }
```

---

## 11. Test Design & Strategy

### 11.1 How to Decide What to Test

Ask:
1. **If this breaks, does a user notice or data rot?** → test it.
2. **Is the behavior a contract with another part of the system?** → test it.
3. **Is this a regression-prone area (complex logic, historical bugs)?** → test it.
4. **Is this trivial code?** → skip it.

Prioritize:
- Happy path per feature.
- Edge cases you've seen in bug reports.
- Permission / security boundaries.
- Anything around money, time, or identity.

### 11.2 Test Boundaries

A test should assert **one observable behavior**. A good test fails for one reason.

**Test from the boundary closest to the user:**
- Controller tests prove API shape.
- Service tests prove business rules.
- Repository tests prove queries.

Don't test the **implementation detail** — test the **contract**. If your class's public method takes a DTO and returns a result, that's what to test. How many private methods it has is irrelevant.

### 11.3 Avoiding Over-Testing

Symptoms:
- Every refactor breaks 30 tests.
- Tests fail with `verify(...)` errors that have nothing to do with the behavior.
- Your test file is twice the size of your production file — per class — with no obvious value.

Cures:
- Delete tests that don't test behavior.
- Prefer state-based over interaction-based assertions.
- Mock only external boundaries, not your own classes.

### 11.4 Maintaining Test Suites

- **Review tests like code.** Duplication in tests rots just like duplication in production.
- **Profile suite speed.** Find the slowest 5% and fix them — they dominate CI time.
- **Keep tests readable.** A test failing in 2 years should be understandable on its own.
- **Delete without fear.** A test that has never failed and tests nothing is cost without value.

### 11.5 Flaky Tests and How to Fix Them

**Symptoms:** passes locally, fails in CI. Passes on retry. Fails under load.

**Causes and fixes:**

| Cause | Fix |
|---|---|
| `Thread.sleep` for async code | Use Awaitility: `await().atMost(5s).until(...)` |
| Shared static state | Reset in `@BeforeEach` or avoid shared state |
| Test ordering dependency | Each test sets up its own state; avoid `@TestMethodOrder` |
| Real network / time | Inject `Clock`, mock `WebClient`, use Testcontainers |
| Race condition in SUT | It's a real bug — fix the code |
| DB cleanup left data | Use `@Transactional` rollback or `TRUNCATE` before |
| Resource leak | Close connections, use try-with-resources |

**Do not silently retry flaky tests.** Retries mask real bugs. If you must quarantine, do so in a tagged bucket with a ticket to fix — not forever.

---

## 12. CI/CD Integration

### 12.1 Running Tests in Pipelines

**GitHub Actions:**

```yaml
name: test
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Run unit tests
        run: ./gradlew test --info --no-daemon

      - name: Run integration tests
        run: ./gradlew integrationTest --no-daemon

      - name: Publish test results
        if: always()
        uses: dorny/test-reporter@v1
        with:
          name: JUnit
          path: 'build/test-results/**/*.xml'
          reporter: java-junit

      - name: Coverage
        run: ./gradlew jacocoTestReport
      - name: Upload to Codecov
        uses: codecov/codecov-action@v4
```

### 12.2 Split Suites: Unit vs Integration

In `build.gradle`:

```groovy
sourceSets {
    integrationTest {
        java.srcDir 'src/integrationTest/java'
        resources.srcDir 'src/integrationTest/resources'
        compileClasspath += sourceSets.main.output + sourceSets.test.output
        runtimeClasspath += sourceSets.main.output + sourceSets.test.output
    }
}

configurations {
    integrationTestImplementation.extendsFrom testImplementation
    integrationTestRuntimeOnly.extendsFrom testRuntimeOnly
}

tasks.register('integrationTest', Test) {
    useJUnitPlatform()
    testClassesDirs = sourceSets.integrationTest.output.classesDirs
    classpath = sourceSets.integrationTest.runtimeClasspath
    shouldRunAfter test
}

check.dependsOn integrationTest
```

**Why split?** Unit tests run on every push. Integration tests can run per-PR or nightly. Fast feedback matters.

### 12.3 Parallel Testing

JUnit 5 parallel execution (`src/test/resources/junit-platform.properties`):

```properties
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=same_thread
junit.jupiter.execution.parallel.mode.classes.default=concurrent
junit.jupiter.execution.parallel.config.strategy=dynamic
```

- Classes run concurrently; methods within a class share a thread by default.
- Full method parallelism requires ensuring tests are **independent** — a high bar.

Gradle parallel forks:

```groovy
test {
    maxParallelForks = Runtime.runtime.availableProcessors().intdiv(2) ?: 1
    forkEvery = 100
}
```

### 12.4 Test Reports

- **JUnit XML** → universal format, every CI reads it.
- **JaCoCo** → coverage (XML/HTML).
- **Surefire/Gradle HTML reports** for local debugging.
- **Codecov / SonarQube** for coverage trend.

### 12.5 Flake Triage

- Keep a **flake dashboard** — which tests fail most often in CI.
- **Quarantine** with a tag (`@Tag("flaky")`) and a ticket. Never forever.
- **Cap retries** at 1 with loud logging — don't hide flakes behind 10 retries.

---

## 13. Production Best Practices

### 13.1 Writing Maintainable Tests

- **Arrange / Act / Assert.** Visibly separated, short. If Arrange is 30 lines, extract a builder.
- **One logical assertion per test.** Multiple `assertThat` calls are fine if they're about the same concept.
- **Name for behavior**, not implementation: `rejects_expired_coupons`, not `testCalcPrice3`.
- **Use `@DisplayName`** for complex test names.
- **Avoid conditional logic in tests.** No `if`/`for`/`switch`. If you need branches, write another test.

### 13.2 Naming Conventions

- `MethodName_WhenCondition_ThenResult`: `transfer_whenInsufficientBalance_throwsException`
- `Behavior style`: `rejects_transfer_when_balance_is_insufficient`
- `Given/When/Then` in comments for BDD readability.

Pick one per project and stick with it.

### 13.3 Test Readability

A test should be readable top-to-bottom by someone who has never seen it.

```java
@Test
void rejects_transfer_when_source_has_insufficient_balance() {
    // Given
    Account source = anAccount().withBalance("10.00").build();
    Account target = anAccount().withBalance("50.00").build();
    accountsExist(source, target);

    // When / Then
    assertThatThrownBy(() -> service.transfer(source.getId(), target.getId(), new BigDecimal("30.00")))
        .isInstanceOf(InsufficientFundsException.class);

    verifyNoMoneyMoved();
}
```

Use helper methods (`accountsExist`, `verifyNoMoneyMoved`) to keep the intent front-and-center.

### 13.4 Debugging Failing Tests

- Read the failure message first. JUnit + AssertJ give diffs — use them.
- Run only the failing test: `./gradlew test --tests com.acme.X.shouldY`.
- Enable debug logs: `-Dlogging.level.com.acme=DEBUG`.
- Check for state pollution from other tests: run the test in isolation.
- For integration tests, inspect Testcontainers logs: `container.getLogs()`.

### 13.5 Common Mistakes

| Mistake | Why bad | Fix |
|---|---|---|
| Logic in tests | Tests of tests | Use parameterized tests or separate tests |
| Shared mutable state | Flaky, order-dependent | Fresh state per test |
| Huge setup methods | Unreadable | Use builders, factories |
| Mocking the DB | Misses query bugs | Testcontainers |
| Assert on internal state (private fields) | Breaks refactoring | Assert on observable behavior |
| Catching exceptions to pass test | Hides real bugs | Use `assertThrows` / `assertThatThrownBy` |
| `@Sql` scripts for everything | Drift from entities | Prefer builders |
| `@Transactional` on async flows | Async thread doesn't see TX | Flush or use explicit cleanup |
| Same mock reused across tests | Leaked expectations | `@MockBean` resets per test in Spring |
| Comments explaining what a test does | Rename the test | Name tests so comments aren't needed |

---

## 14. Real-World Scenarios

### 14.1 Microservices Testing Strategy

**Context:** 12 services, 6 teams, Kafka + REST.

**Strategy:**
- **Per service**: rich unit tests for logic; `@DataJpaTest`/`@WebMvcTest` for slices; `@SpringBootTest` + Testcontainers for critical flows.
- **Between services**: Pact contract tests. Consumer expectations published; producer CI verifies.
- **E2E**: nightly test suite running the top 10 user journeys in a staging cluster.

**Tools:** JUnit 5, Mockito, AssertJ, Testcontainers, Pact, Gatling.

**Trade-off:** contract tests add pipeline complexity but eliminate "field removed in service A broke service B" bugs.

### 14.2 High-Load API

**Context:** 30k RPS REST API, Spring Boot + PostgreSQL + Redis.

**Strategy:**
- Unit tests for hot code paths.
- Testcontainers for DB + Redis integration tests.
- **Gatling** load tests in a pre-prod environment on every release candidate.
- SLO-based assertions: p99 < 150 ms under 30k RPS.
- Chaos tests: kill Redis mid-test, ensure graceful degradation.

**Trade-off:** Gatling is slow; don't run on every PR — only on release branches.

### 14.3 Event-Driven System

**Context:** order service publishes to Kafka; inventory and fulfillment consume.

**Strategy:**
- **Producer side**: unit test event serialization; integration test using `KafkaContainer` verifies messages land on topics.
- **Consumer side**: `@EmbeddedKafka` or `KafkaContainer`; publish test messages, assert DB state changes via Awaitility.
- **End-to-end**: start multiple services in Testcontainers (compose), send a request, assert the full chain.

```java
await().atMost(Duration.ofSeconds(10))
    .untilAsserted(() -> assertThat(inventoryRepo.findBySku("SKU-1").getQty()).isEqualTo(98));
```

**Trade-off:** Kafka tests are slower (container + consumer lag). Keep them focused; don't run one per test method.

### 14.4 Legacy Monolith Under Refactor

**Context:** Spring MVC monolith, 700k LOC, 8% coverage, frequent regressions.

**Strategy:**
- Write **characterization tests** for existing behavior before refactoring.
- Add Testcontainers-based integration tests at API boundaries.
- Gradually extract modules; unit-test extracted modules aggressively.
- Mutation testing (PIT) on refactored modules to verify assertion quality.

**Trade-off:** slow climb, but each refactor becomes safer. Don't chase coverage % — chase bug-prone areas.

### 14.5 Financial / Regulated System

**Context:** money movement, audit requirements, strict compliance.

**Strategy:**
- **Deterministic time** everywhere — inject `Clock`.
- Property-based tests (jqwik) for invariants: "no transfer can create money" (total sum before == total sum after).
- Every business rule has an explicit test with a named requirement (`rule-4.2.1`).
- Audit log verified via integration test.
- Liquibase migrations tested against prod-sized dataset snapshot.

**Trade-off:** slower development velocity, but required. Audit trails and determinism non-negotiable.

---

## 15. Comparisons & Trade-offs

### 15.1 Mockito vs Real Dependencies

| Use Mockito when | Use Real Dependencies when |
|---|---|
| Dependency is slow (network, DB) | Dependency is fast and pure |
| Dependency is external (3rd-party API) | Dependency is your own simple class |
| You need to simulate error conditions | Integration is the thing you're testing |
| You're unit-testing a service | You're integration-testing wiring |
| The dependency has complex setup | A real instance is trivial |

**Warning:** mocking too deep produces tests that pass but don't reflect reality. If you're mocking 4 layers deep, your unit test is pretending to be an integration test and failing at both jobs.

### 15.2 In-Memory DB vs Testcontainers

| In-Memory (H2) | Testcontainers (Postgres) |
|---|---|
| Fastest | Slower (but manageable) |
| No Docker needed | Docker required |
| Wrong dialect | Correct dialect |
| Can't test JSONB/arrays/enums | Full support |
| Can pass tests that fail in prod | Tests match prod |
| OK for generic ORM checks | Needed for anything real |

**Default to Testcontainers for anything non-trivial.** H2 is a debt; Testcontainers is an investment.

### 15.3 Unit vs Integration Tests

| Unit | Integration |
|---|---|
| Fast (ms) | Slower (100 ms–s) |
| Localize bugs precisely | Catch wiring/config bugs |
| Easy to write, many of them | Fewer, more valuable |
| Risk: over-mocking hides bugs | Risk: slow, flaky if misused |

**Both are necessary.** Unit tests give you speed and locality. Integration tests give you confidence that the parts actually work together. Skimp on either and you'll feel it — in velocity or in outages.

---

## 16. Final Checklist

### You are production-ready in Java testing if you can:

- [ ] Explain the **test pyramid**, defend a ratio, and recognize when a codebase has drifted into an ice cream cone.
- [ ] Distinguish **unit, integration, E2E, and contract** tests — and know which belongs where.
- [ ] Write **JUnit 5** tests with correct lifecycle, parameterized inputs, nested grouping, and expressive AssertJ assertions.
- [ ] Use **Mockito** correctly: `@Mock`, `@InjectMocks`, argument matchers, spies, and know the pitfalls of over-verification and over-mocking.
- [ ] Design code for testability (constructor injection, `Clock`, separation of I/O from logic).
- [ ] Pick the right **Spring Boot test slice** (`@WebMvcTest`, `@DataJpaTest`, `@WebFluxTest`, `@RestClientTest`) rather than defaulting to `@SpringBootTest`.
- [ ] Test **REST APIs** with `MockMvc` / `WebTestClient`, covering validation, error responses, and security.
- [ ] Spin up **Testcontainers** (Postgres, Redis, Kafka) with per-JVM shared lifecycle and `@ServiceConnection`.
- [ ] Choose **Testcontainers over H2** for anything using Postgres-specific features.
- [ ] Manage test data with **builders** and fixtures, and clean up via transactional rollback or explicit strategies.
- [ ] Test **reactive code** with `StepVerifier`, virtual time, and `WebTestClient`; detect blocking code with BlockHound.
- [ ] Identify and eliminate **flaky tests** — with Awaitility replacing `Thread.sleep` and deterministic time everywhere.
- [ ] Wire tests into **CI/CD** with separate unit/integration stages, parallelism, JUnit XML reports, and coverage gates.
- [ ] Evaluate a test suite's **quality** using mutation testing, not just line coverage.
- [ ] Explain to a skeptical stakeholder the **ROI** of testing: shorter incident times, faster refactoring, fewer regressions, better documentation.
- [ ] Recognize when a test is over-specified, brittle, or testing the framework — and delete it.

---

*End of guide. Save as `java-testing-learning-guide.md`.*
