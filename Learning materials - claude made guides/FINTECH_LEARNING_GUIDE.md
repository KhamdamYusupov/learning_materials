# Fintech & Digital Banking Learning Guide for Java Backend Engineers

> **Audience:** Mid-level Java backend developer with no prior fintech experience
> **Context:** Digital bank operating in Tashkent, Uzbekistan
> **Goal:** Get productive fast in a real-world fintech environment

---

## Table of Contents

1. [Fintech & Digital Banking Fundamentals](#1-fintech--digital-banking-fundamentals)
2. [Payments Ecosystem](#2-payments-ecosystem-global--uzbekistan-context)
3. [Regulation & Compliance](#3-regulation--compliance)
4. [Core Backend Concepts for Fintech](#4-core-backend-concepts-for-fintech)
5. [Java Backend Stack](#5-java-backend-stack)
6. [System Design for Payments](#6-system-design-for-payments)
7. [Security in Fintech](#7-security-in-fintech)
8. [Real-World Scenarios](#8-real-world-scenarios)
9. [Glossary](#9-glossary)

---

## 1. Fintech & Digital Banking Fundamentals

### 1.1 How Digital Banks Work

A **digital bank** (neobank) is a bank with no physical branches. Everything — account opening, transfers, loans, card management — happens through a mobile app or web interface.

**Simple mental model:**

```
┌─────────────┐     ┌──────────────┐     ┌──────────────────┐
│  Mobile App  │────▶│  Backend API │────▶│  Core Banking     │
│  (Customer)  │◀────│  (Your code) │◀────│  System (CBS)     │
└─────────────┘     └──────────────┘     └──────────────────┘
                           │
                    ┌──────┴──────┐
                    │  External   │
                    │  Partners   │
                    │ (Visa, HUMO,│
                    │  CBU, etc.) │
                    └─────────────┘
```

**Your code sits in the middle layer.** You translate user actions into transactions in the core banking system and coordinate with external partners (card networks, regulators, payment providers).

### 1.2 Digital Bank vs. Traditional Bank

| Aspect | Traditional Bank | Digital Bank |
|--------|-----------------|--------------|
| Channels | Branches, ATMs, online | Mobile-first, API-first |
| Infrastructure | Mainframes, monoliths | Cloud/on-prem microservices |
| Speed of delivery | Months | Days to weeks |
| Customer onboarding | In-person, paper-based | Remote, KYC via app |
| Cost structure | High (real estate, staff) | Lower (engineering-heavy) |
| Integration approach | Batch processing, files | Real-time APIs, events |

**Key insight:** In a digital bank, *your backend IS the bank*. There's no branch clerk to fix a mistake manually. Every edge case must be handled in code.

### 1.3 Core Banking System (CBS) Overview

The CBS is the "source of truth" for all financial data. It manages:

- **Customer accounts** — balances, account types, status
- **Ledger** — the immutable record of every financial movement (double-entry bookkeeping)
- **Interest & fees** — calculations, accruals
- **Limits & controls** — daily transfer limits, overdraft rules

**Double-entry bookkeeping** — the single most important concept:

Every financial transaction has **two sides**. Money doesn't appear or disappear; it moves.

```
Transfer 100,000 UZS from Account A to Account B:

  DEBIT   Account A    -100,000 UZS
  CREDIT  Account B    +100,000 UZS
  ────────────────────────────────
  Net change:                   0
```

If the debits and credits don't balance, something is wrong. This is the foundation of financial integrity.

**In Uzbekistan**, CBS platforms vary — some banks use international solutions (Temenos, Finastra), while newer digital banks may build custom ledgers or use modern platforms. Regardless of the platform, the principles are the same.

---

## 2. Payments Ecosystem (Global + Uzbekistan Context)

### 2.1 Key Players in a Payment

Think of a card payment as a chain:

```
┌──────────┐   ┌─────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Customer │──▶│Merchant/ │──▶│ Acquirer  │──▶│  Card    │──▶│  Issuer  │
│(Cardholder)│ │  POS/App │   │  (Bank)  │   │ Network  │   │  (Bank)  │
└──────────┘   └─────────┘   └──────────┘   └──────────┘   └──────────┘
```

- **Cardholder** — the person paying (your bank's customer)
- **Merchant** — the store or service accepting payment
- **Acquirer** — the bank that processes payments on behalf of the merchant
- **Card Network** — routes the transaction between acquirer and issuer (Visa, Mastercard, UzCard, HUMO)
- **Issuer** — the bank that issued the card to the cardholder (*this is likely your bank*)
- **PSP (Payment Service Provider)** — a company that connects merchants to multiple payment methods (e.g., Stripe, PayMe, Click in Uzbekistan)
- **Payment Gateway** — the technology layer that captures and encrypts card details and routes them to the acquirer/PSP

### 2.2 Card Networks in Uzbekistan

Uzbekistan has a unique payments landscape with **local card networks** alongside international ones:

| Network | Type | Notes |
|---------|------|-------|
| **UzCard** | Local (dominant) | Operated by the National Interbank Processing Center (NIPC). Most widely used in Uzbekistan. UZS-denominated. |
| **HUMO** | Local | Operated by the Unified National Payment System (UNPS). Growing network, government-backed. |
| **Visa** | International | Used for international transactions, premium cards |
| **Mastercard** | International | Similar to Visa, growing presence |
| **UnionPay** | International | Increasingly common due to regional trade |

**Practical implication:** Your backend will likely need to integrate with **multiple card networks simultaneously**. A single customer may have both a UzCard and a Visa card. Each network has its own:
- Message format (ISO 8583 variants, proprietary APIs)
- Settlement schedule
- Error codes
- Certification/testing requirements

### 2.3 Payment Flows

#### Authorization Flow (Real-time)

When a customer swipes their card or pays online:

```
1. Customer taps card at terminal
2. Terminal → Acquirer → Card Network → YOUR BANK (Issuer)
3. Your system checks:
   - Is the card valid and active?
   - Does the customer have sufficient balance?
   - Does this trip any fraud rules?
   - Is the amount within daily limits?
4. Your system responds: APPROVED or DECLINED (with reason code)
5. Response travels back: Your Bank → Network → Acquirer → Terminal
6. Customer sees "Payment Approved"

Total time budget: typically < 3 seconds end-to-end
```

**This is mission-critical code.** If your authorization service is down, cards stop working everywhere.

#### The Four Stages of a Payment

```
┌───────────────┐   ┌───────────┐   ┌────────────┐   ┌────────────┐
│ Authorization │──▶│  Capture   │──▶│ Settlement │──▶│   Funding  │
│ (Hold funds)  │   │(Confirm)   │   │(Net amounts)│  │(Move money)│
└───────────────┘   └───────────┘   └────────────┘   └────────────┘
```

1. **Authorization** — "Can this customer pay 50,000 UZS?" Funds are *held* but not yet moved.
2. **Capture** — The merchant confirms the transaction. "Yes, charge the 50,000 UZS."
3. **Settlement** — The card network calculates net amounts between all banks (usually end-of-day batch).
4. **Funding** — Actual money moves between bank accounts.

#### Refunds and Reversals

- **Reversal** — Cancels an authorization *before* settlement. Fast, no money has moved yet.
- **Refund** — Returns money *after* settlement. Creates a new transaction in the opposite direction. Takes days.
- **Chargeback** — Customer disputes a charge through their bank. Formal process with evidence, deadlines, and fees.

### 2.4 Local Payment Systems in Uzbekistan

Beyond card payments, Uzbekistan has a growing ecosystem of digital payments:

- **PayMe, Click, Apelsin** — popular mobile payment apps that function as PSPs
- **P2P transfers** — person-to-person transfers via UzCard/HUMO are extremely popular
- **QR payments** — growing adoption for merchant payments
- **SWIFT / correspondent banking** — for international transfers (USD, EUR)

**Central Bank of Uzbekistan (CBU)** operates the national payment infrastructure and regulates all payment systems. All banks must connect to CBU's systems for interbank settlements.

---

## 3. Regulation & Compliance

### 3.1 PCI-DSS Basics

**PCI-DSS** (Payment Card Industry Data Security Standard) is a set of security requirements for anyone who stores, processes, or transmits cardholder data.

**The golden rule: Never store full card numbers (PAN) in your systems if you can avoid it.**

Key PCI-DSS requirements for developers:

| Requirement | What It Means for You |
|------------|----------------------|
| Encrypt cardholder data | Use AES-256 for data at rest, TLS 1.2+ in transit |
| Restrict access to card data | Role-based access, need-to-know basis |
| Never store CVV/CVC | After authorization, CVV must be deleted immediately |
| Mask PAN in logs/displays | Show only last 4 digits: `**** **** **** 1234` |
| Maintain audit trails | Log who accessed what, when |
| Regular vulnerability testing | Code reviews, penetration testing |

```java
// WRONG - Never do this
log.info("Processing payment for card: " + cardNumber);

// RIGHT - Mask sensitive data
log.info("Processing payment for card: ****" + cardNumber.substring(cardNumber.length() - 4));

// EVEN BETTER - Use a dedicated masking utility
log.info("Processing payment for card: {}", CardMasker.mask(cardNumber));
```

### 3.2 KYC / AML

**KYC (Know Your Customer)** — verifying who your customers are before letting them use financial services.

Typical KYC flow for a digital bank in Uzbekistan:
1. Customer downloads app
2. Scans passport or national ID
3. Takes a selfie (liveness check)
4. System verifies identity against government databases
5. Risk scoring determines account tier (limits)

**AML (Anti-Money Laundering)** — detecting and preventing financial crime.

Your code may need to:
- Flag transactions above certain thresholds (e.g., large cash deposits)
- Detect suspicious patterns (structuring, rapid movement of funds)
- Screen customers against sanctions lists (OFAC, UN, local lists)
- Generate Suspicious Activity Reports (SARs) for the compliance team

```java
// Simplified transaction monitoring example
public class TransactionMonitor {

    private static final BigDecimal REPORTING_THRESHOLD =
        new BigDecimal("100_000_000"); // 100M UZS (~$8,000 USD approx.)

    public RiskAssessment evaluate(Transaction tx) {
        var signals = new ArrayList<RiskSignal>();

        // Large transaction
        if (tx.getAmount().compareTo(REPORTING_THRESHOLD) >= 0) {
            signals.add(RiskSignal.LARGE_AMOUNT);
        }

        // Unusual frequency
        long recentTxCount = txRepo.countByCustomerIdAndCreatedAfter(
            tx.getCustomerId(), Instant.now().minus(Duration.ofHours(1)));
        if (recentTxCount > 10) {
            signals.add(RiskSignal.HIGH_FREQUENCY);
        }

        // New account with large transaction
        Customer customer = customerRepo.findById(tx.getCustomerId());
        if (customer.getAccountAge().toDays() < 30
                && tx.getAmount().compareTo(new BigDecimal("50_000_000")) >= 0) {
            signals.add(RiskSignal.NEW_ACCOUNT_LARGE_TX);
        }

        return new RiskAssessment(signals);
    }
}
```

### 3.3 Local Regulatory Considerations (Uzbekistan)

- **Central Bank of Uzbekistan (CBU)** is the primary regulator for all banking and payment activities
- **Licensing** — digital banks need a banking license from CBU
- **Currency controls** — Uzbekistan has regulations around foreign currency transactions; UZS is the primary currency
- **Data localization** — financial data may need to be stored within Uzbekistan's borders (check current regulations)
- **Reporting** — banks must submit regular reports to CBU on transactions, balances, and compliance metrics
- **Interest rate regulations** — CBU sets reference rates and may impose caps
- **Consumer protection** — rules around fee disclosure, dispute resolution

> **Note:** Regulations evolve. Always consult your compliance team for current requirements. As a developer, your job is to build systems flexible enough to adapt to regulatory changes.

---

## 4. Core Backend Concepts for Fintech

### 4.1 Transaction Management & Idempotency

In fintech, **every operation that moves money must be idempotent** — if the same request is sent twice (due to network retry, user double-tap, etc.), it must produce the same result and charge the customer only once.

**Idempotency key pattern:**

```java
@RestController
@RequestMapping("/api/v1/transfers")
public class TransferController {

    @PostMapping
    public ResponseEntity<TransferResult> createTransfer(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody TransferRequest request) {

        // 1. Check if we've already processed this request
        Optional<TransferResult> existing = transferService
            .findByIdempotencyKey(idempotencyKey);

        if (existing.isPresent()) {
            // Return the same response — do NOT process again
            return ResponseEntity.ok(existing.get());
        }

        // 2. Process the transfer
        TransferResult result = transferService.execute(idempotencyKey, request);
        return ResponseEntity.status(HttpStatus.CREATED).body(result);
    }
}
```

**Database-level idempotency:**

```sql
CREATE TABLE transfers (
    id              BIGSERIAL PRIMARY KEY,
    idempotency_key VARCHAR(64) NOT NULL UNIQUE, -- prevents duplicates at DB level
    from_account    VARCHAR(20) NOT NULL,
    to_account      VARCHAR(20) NOT NULL,
    amount          DECIMAL(18,2) NOT NULL,
    currency        VARCHAR(3) NOT NULL DEFAULT 'UZS',
    status          VARCHAR(20) NOT NULL,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW()
);
```

### 4.2 Concurrency and Data Consistency

Money requires **absolute correctness**. Two simultaneous withdrawals from the same account must not both succeed if the balance only covers one.

#### Pessimistic Locking

Lock the row while you work on it. Other transactions wait.

```java
public interface AccountRepository extends JpaRepository<Account, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Account a WHERE a.accountNumber = :accountNumber")
    Optional<Account> findByAccountNumberForUpdate(
        @Param("accountNumber") String accountNumber);
}
```

```java
@Service
public class TransferService {

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public TransferResult execute(String idempotencyKey, TransferRequest request) {

        // Lock both accounts (always in consistent order to prevent deadlocks!)
        String firstAccount = min(request.getFromAccount(), request.getToAccount());
        String secondAccount = max(request.getFromAccount(), request.getToAccount());

        Account first = accountRepo.findByAccountNumberForUpdate(firstAccount)
            .orElseThrow(() -> new AccountNotFoundException(firstAccount));
        Account second = accountRepo.findByAccountNumberForUpdate(secondAccount)
            .orElseThrow(() -> new AccountNotFoundException(secondAccount));

        // Determine which is sender, which is receiver
        Account sender = first.getAccountNumber().equals(request.getFromAccount())
            ? first : second;
        Account receiver = first.getAccountNumber().equals(request.getToAccount())
            ? first : second;

        // Check balance
        if (sender.getBalance().compareTo(request.getAmount()) < 0) {
            throw new InsufficientFundsException(sender.getAccountNumber());
        }

        // Move money
        sender.debit(request.getAmount());
        receiver.credit(request.getAmount());

        // Record the transfer
        Transfer transfer = Transfer.builder()
            .idempotencyKey(idempotencyKey)
            .fromAccount(sender.getAccountNumber())
            .toAccount(receiver.getAccountNumber())
            .amount(request.getAmount())
            .currency("UZS")
            .status(TransferStatus.COMPLETED)
            .build();

        transferRepo.save(transfer);

        return TransferResult.success(transfer);
    }
}
```

**Why lock in consistent order?** If Transaction A locks Account 1 then tries to lock Account 2, while Transaction B locks Account 2 then tries to lock Account 1, you get a **deadlock**. Always lock in the same order (e.g., by account number ascending).

#### Optimistic Locking

Use a version column. If someone else changed the row since you read it, your update fails, and you retry.

```java
@Entity
@Table(name = "accounts")
public class Account {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "account_number", unique = true, nullable = false)
    private String accountNumber;

    @Column(nullable = false, precision = 18, scale = 2)
    private BigDecimal balance;

    @Version  // JPA optimistic locking
    private Long version;

    public void debit(BigDecimal amount) {
        if (this.balance.compareTo(amount) < 0) {
            throw new InsufficientFundsException(this.accountNumber);
        }
        this.balance = this.balance.subtract(amount);
    }

    public void credit(BigDecimal amount) {
        this.balance = this.balance.add(amount);
    }
}
```

**When to use which:**

| Scenario | Recommended Approach |
|----------|---------------------|
| Money transfers between accounts | Pessimistic locking |
| Updating user profile | Optimistic locking |
| High-contention account (merchant) | Pessimistic + consider sharding |
| Read-heavy, rare writes | Optimistic locking |

### 4.3 Event-Driven Architecture

In banking, many processes are triggered by events rather than direct API calls:

```
Payment Completed ──▶ Send SMS notification
                 ──▶ Update account balance cache
                 ──▶ Check fraud rules
                 ──▶ Update loyalty points
                 ──▶ Generate accounting entry
```

**Why events?** Decoupling. The payment service shouldn't know about SMS, fraud, or loyalty. Each concern subscribes to the event independently.

```java
// Domain event
public record PaymentCompletedEvent(
    String transactionId,
    String accountId,
    BigDecimal amount,
    String currency,
    String merchantName,
    Instant timestamp
) {}

// Publishing (using Spring's ApplicationEventPublisher or Kafka)
@Service
public class PaymentService {

    private final KafkaTemplate<String, PaymentCompletedEvent> kafka;

    @Transactional
    public PaymentResult processPayment(PaymentRequest request) {
        // ... process payment ...

        // Publish event AFTER transaction commits
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronization() {
                @Override
                public void afterCommit() {
                    kafka.send("payment.completed",
                        payment.getAccountId(),  // key for partitioning
                        new PaymentCompletedEvent(
                            payment.getId(),
                            payment.getAccountId(),
                            payment.getAmount(),
                            payment.getCurrency(),
                            payment.getMerchantName(),
                            Instant.now()
                        ));
                }
            });

        return PaymentResult.success(payment);
    }
}

// Consuming
@Component
public class NotificationListener {

    @KafkaListener(topics = "payment.completed", groupId = "notification-service")
    public void onPaymentCompleted(PaymentCompletedEvent event) {
        smsService.send(
            customerRepo.findPhoneByAccountId(event.accountId()),
            String.format("Payment of %s %s at %s",
                event.amount(), event.currency(), event.merchantName())
        );
    }
}
```

**Critical pattern: Transactional Outbox**

Publishing an event and committing a database transaction are two separate operations. If the app crashes between them, you either lose the event or process without committing. The **outbox pattern** solves this:

```
1. Within the SAME database transaction:
   - Save the business data (e.g., transfer record)
   - Save the event to an "outbox" table

2. A separate process reads the outbox table and publishes to Kafka

3. After successful publish, mark the outbox entry as sent
```

```sql
CREATE TABLE outbox (
    id          BIGSERIAL PRIMARY KEY,
    aggregate_type VARCHAR(50) NOT NULL,   -- e.g., 'Payment'
    aggregate_id   VARCHAR(50) NOT NULL,   -- e.g., payment ID
    event_type     VARCHAR(100) NOT NULL,  -- e.g., 'PaymentCompleted'
    payload        JSONB NOT NULL,
    created_at     TIMESTAMP NOT NULL DEFAULT NOW(),
    published_at   TIMESTAMP              -- NULL means not yet published
);
```

### 4.4 Microservices in Banking Systems

A typical digital bank backend might be structured as:

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Account   │  │   Payment   │  │    Card     │  │   Customer  │
│   Service   │  │   Service   │  │   Service   │  │   Service   │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │                │
═══════╪════════════════╪════════════════╪════════════════╪══════════
       │           Message Bus (Kafka)                    │
═══════╪════════════════╪════════════════╪════════════════╪══════════
       │                │                │                │
┌──────┴──────┐  ┌──────┴──────┐  ┌─────┴───────┐  ┌────┴────────┐
│  Notification│  │   Fraud    │  │   Lending   │  │  Reporting  │
│   Service   │  │   Service   │  │   Service   │  │   Service   │
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
```

**Key principle: Each service owns its data.** The Account Service is the only service that can directly read/write account balances. Other services request data via APIs or consume events.

**Saga pattern** for distributed transactions:

A money transfer between two banks can't use a single database transaction. Instead, use a **saga** — a sequence of local transactions coordinated by events:

```
Transfer Saga:
1. Debit sender's account (Account Service)
   ✓ Success → proceed
   ✗ Failure → reject transfer

2. Send to external bank (Payment Gateway)
   ✓ Success → complete
   ✗ Failure → compensate: credit sender's account back
```

### 4.5 Fault Tolerance, Retries, and Resilience

In fintech, downstream systems **will** fail. Networks go down, services restart, databases hit connection limits. Your code must handle this gracefully.

#### Retry with Exponential Backoff

```java
@Configuration
public class RetryConfig {

    @Bean
    public RetryTemplate paymentRetryTemplate() {
        return RetryTemplate.builder()
            .maxAttempts(3)
            .exponentialBackoff(
                Duration.ofMillis(500),   // initial interval
                2.0,                       // multiplier
                Duration.ofSeconds(5))     // max interval
            .retryOn(TransientException.class)  // only retry transient errors
            .build();
    }
}

// Usage
@Service
public class ExternalPaymentGateway {

    private final RetryTemplate retryTemplate;

    public PaymentResponse sendPayment(PaymentRequest request) {
        return retryTemplate.execute(context -> {
            log.info("Attempt {} to send payment {}",
                context.getRetryCount() + 1, request.getId());
            return httpClient.post("/payments", request, PaymentResponse.class);
        });
    }
}
```

#### Circuit Breaker (Resilience4j)

If an external system is down, stop hammering it. "Open the circuit" and fail fast.

```java
@Service
public class UzCardGateway {

    private final CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("uzcard");

    public AuthorizationResponse authorize(AuthorizationRequest request) {
        return circuitBreaker.executeSupplier(() -> {
            // Call UzCard API
            return uzCardClient.authorize(request);
        });
    }
}
```

**Circuit breaker states:**

```
CLOSED (normal) ──[failures exceed threshold]──▶ OPEN (fail fast)
                                                      │
                                               [wait duration]
                                                      │
                                                      ▼
                                               HALF_OPEN (test)
                                              ┌─[success]──▶ CLOSED
                                              └─[failure]──▶ OPEN
```

---

## 5. Java Backend Stack

### 5.1 Spring Boot Best Practices for Fintech

#### Structured Configuration

```java
@Configuration
@ConfigurationProperties(prefix = "bank.transfer")
@Validated
public class TransferProperties {

    @NotNull
    private BigDecimal dailyLimit;          // e.g., 50,000,000 UZS

    @NotNull
    private BigDecimal singleTransactionMax; // e.g., 10,000,000 UZS

    @Min(1) @Max(10)
    private int maxRetries = 3;

    @NotNull
    private Duration timeout = Duration.ofSeconds(30);

    // getters, setters...
}
```

```yaml
# application.yml
bank:
  transfer:
    daily-limit: 50000000
    single-transaction-max: 10000000
    max-retries: 3
    timeout: 30s
```

#### Health Checks

In banking, monitoring is not optional. Expose health of all critical dependencies:

```java
@Component
public class CoreBankingHealthIndicator implements HealthIndicator {

    private final CoreBankingClient cbsClient;

    @Override
    public Health health() {
        try {
            cbsClient.ping();
            return Health.up()
                .withDetail("system", "Core Banking")
                .withDetail("responseTime", cbsClient.getLastResponseTimeMs() + "ms")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("system", "Core Banking")
                .withException(e)
                .build();
        }
    }
}
```

#### Structured Logging

Every log line in fintech should be traceable to a specific transaction:

```java
@Component
public class TransactionContext {

    public void setContext(String transactionId, String customerId) {
        MDC.put("transactionId", transactionId);
        MDC.put("customerId", customerId);
    }

    public void clear() {
        MDC.clear();
    }
}

// In logback-spring.xml, include MDC fields:
// %d{ISO8601} [%thread] %-5level %logger - txId=%X{transactionId} cust=%X{customerId} - %msg%n
```

### 5.2 Database Design & Transactions

#### Isolation Levels — When to Use What

| Level | Dirty Read | Non-repeatable Read | Phantom Read | Use Case |
|-------|-----------|-------------------|-------------|----------|
| READ_UNCOMMITTED | Yes | Yes | Yes | Never in fintech |
| READ_COMMITTED | No | Yes | Yes | General queries |
| REPEATABLE_READ | No | No | Yes | Balance checks |
| SERIALIZABLE | No | No | No | Critical financial ops |

```java
// For balance inquiry — needs consistent read
@Transactional(isolation = Isolation.REPEATABLE_READ, readOnly = true)
public BigDecimal getBalance(String accountNumber) {
    return accountRepo.findBalanceByAccountNumber(accountNumber);
}

// For money movement — needs strongest guarantees
@Transactional(isolation = Isolation.SERIALIZABLE)
public void transferFunds(String from, String to, BigDecimal amount) {
    // ...
}
```

> **Practical note:** In PostgreSQL, `REPEATABLE_READ` actually provides snapshot isolation, which prevents phantoms too. Know your database's actual behavior, not just the SQL standard names.

#### Financial Amount Storage

**Never use `float` or `double` for money.** Use `BigDecimal` in Java and `DECIMAL`/`NUMERIC` in the database.

```java
// WRONG — floating point errors
double balance = 0.1 + 0.2; // = 0.30000000000000004

// RIGHT — exact decimal arithmetic
BigDecimal balance = new BigDecimal("0.1").add(new BigDecimal("0.2")); // = 0.3

// Alternative: store amounts as smallest unit (e.g., tiyin for UZS)
// 1 UZS = 100 tiyin
// Store 50,000.00 UZS as the long value 5_000_000
long amountInTiyin = 5_000_000L;
```

#### Audit Trail

Every financial table should have audit fields:

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditableEntity {

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    @CreatedBy
    @Column(name = "created_by", updatable = false)
    private String createdBy;

    @LastModifiedBy
    @Column(name = "updated_by")
    private String updatedBy;
}
```

### 5.3 Messaging Systems (Kafka)

Kafka is the backbone of event-driven banking. Key patterns:

#### Exactly-Once Processing

Banking events must be processed **exactly once**. Not "at least once" with duplicates.

```java
@KafkaListener(topics = "transfers", groupId = "ledger-service")
public void processTransfer(ConsumerRecord<String, TransferEvent> record) {
    String eventId = record.headers()
        .lastHeader("event-id").value().toString();

    // Deduplicate: check if we've already processed this event
    if (processedEventRepo.existsById(eventId)) {
        log.info("Skipping duplicate event: {}", eventId);
        return;
    }

    // Process within a transaction
    ledgerService.recordEntry(record.value());

    // Mark as processed (same transaction)
    processedEventRepo.save(new ProcessedEvent(eventId, Instant.now()));
}
```

#### Topic Design

```
payments.authorization.requests    -- incoming auth requests
payments.authorization.responses   -- auth decisions
transfers.initiated                -- new transfer requests
transfers.completed                -- successful transfers
transfers.failed                   -- failed transfers
accounts.balance.updated           -- balance change notifications
compliance.screening.requests      -- AML screening requests
compliance.screening.results       -- screening results
```

### 5.4 API Design

#### RESTful API for Banking

```java
@RestController
@RequestMapping("/api/v1/accounts")
public class AccountController {

    // GET /api/v1/accounts/{accountNumber}/balance
    @GetMapping("/{accountNumber}/balance")
    public ResponseEntity<BalanceResponse> getBalance(
            @PathVariable String accountNumber) {
        // ...
    }

    // POST /api/v1/accounts/{accountNumber}/transfers
    @PostMapping("/{accountNumber}/transfers")
    public ResponseEntity<TransferResponse> initiateTransfer(
            @PathVariable String accountNumber,
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @Valid @RequestBody TransferRequest request) {
        // ...
    }
}
```

#### Error Handling — Standard Error Response

```java
public record ApiError(
    String code,           // Machine-readable: "INSUFFICIENT_FUNDS"
    String message,        // Human-readable: "Account balance is too low"
    String traceId,        // For debugging: "abc-123-def"
    Instant timestamp
) {}

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(InsufficientFundsException.class)
    public ResponseEntity<ApiError> handleInsufficientFunds(
            InsufficientFundsException ex) {
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY)
            .body(new ApiError(
                "INSUFFICIENT_FUNDS",
                "Account balance is insufficient for this transaction",
                MDC.get("traceId"),
                Instant.now()
            ));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleUnexpected(Exception ex) {
        log.error("Unexpected error", ex);
        // Never expose internal details to the client
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ApiError(
                "INTERNAL_ERROR",
                "An unexpected error occurred. Please try again.",
                MDC.get("traceId"),
                Instant.now()
            ));
    }
}
```

#### API Versioning

In banking, you can't just break existing API contracts — mobile apps in the field may be months behind.

```
/api/v1/transfers  ← current version
/api/v2/transfers  ← new version with breaking changes
```

Support old versions for a deprecation period. Communicate timelines clearly.

---

## 6. System Design for Payments

### 6.1 Designing a Payment Processing Service

Here's a high-level architecture for a payment processing system:

```
                          ┌──────────────────────┐
                          │     API Gateway       │
                          │  (Rate limiting, Auth)│
                          └──────────┬───────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
            ┌───────▼──────┐ ┌──────▼───────┐ ┌─────▼───────┐
            │   Transfer   │ │   Payment    │ │    Card     │
            │   Service    │ │   Service    │ │   Service   │
            └───────┬──────┘ └──────┬───────┘ └─────┬───────┘
                    │               │               │
              ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐
              │  Account   │  │   Fraud   │  │   HSM     │
              │  Service   │  │  Engine   │  │ (Crypto)  │
              └─────┬──────┘  └───────────┘  └───────────┘
                    │
         ┌──────────┼──────────┐
         │          │          │
    ┌────▼───┐ ┌───▼────┐ ┌───▼────┐
    │ UzCard │ │  HUMO  │ │  Visa  │
    │Gateway │ │Gateway │ │Gateway │
    └────────┘ └────────┘ └────────┘
```

#### Payment Processing Flow (Detailed)

```java
@Service
@Slf4j
public class PaymentOrchestrator {

    public PaymentResult processPayment(PaymentRequest request) {

        String paymentId = generatePaymentId();

        // Step 1: Validate
        validationService.validate(request); // amount, currency, accounts exist

        // Step 2: Fraud check
        FraudResult fraudResult = fraudEngine.screen(request);
        if (fraudResult.isBlocked()) {
            return PaymentResult.declined(paymentId, "FRAUD_SUSPECTED");
        }

        // Step 3: Check limits
        if (!limitsService.checkWithinLimits(request)) {
            return PaymentResult.declined(paymentId, "LIMIT_EXCEEDED");
        }

        // Step 4: Reserve funds (debit sender)
        accountService.hold(request.getFromAccount(), request.getAmount());

        try {
            // Step 5: Route to appropriate payment network
            PaymentGateway gateway = gatewayRouter.resolve(request);
            GatewayResponse response = gateway.submit(request);

            if (response.isSuccessful()) {
                // Step 6a: Confirm the hold → actual debit
                accountService.confirmHold(request.getFromAccount(),
                    request.getAmount());
                return PaymentResult.success(paymentId, response.getNetworkRef());
            } else {
                // Step 6b: Release the hold
                accountService.releaseHold(request.getFromAccount(),
                    request.getAmount());
                return PaymentResult.declined(paymentId, response.getDeclineReason());
            }
        } catch (Exception e) {
            // Step 6c: Release hold on any error
            accountService.releaseHold(request.getFromAccount(), request.getAmount());
            throw e;
        }
    }
}
```

### 6.2 Handling Partial Failures and Retries

The most dangerous situation in payments: **you sent the money but didn't get a response**. Did it succeed or fail?

```java
/**
 * State machine for payment status.
 * Every payment moves through these states — never skipping one.
 */
public enum PaymentStatus {
    CREATED,          // Request received
    VALIDATED,        // Passed all checks
    FUNDS_HELD,       // Money reserved from sender
    SUBMITTED,        // Sent to payment network
    // --- Danger zone: between SUBMITTED and the next state,
    //     we don't know the real status ---
    COMPLETED,        // Network confirmed success
    FAILED,           // Network confirmed failure
    HOLD_RELEASED,    // Funds returned to sender after failure
    REQUIRES_REVIEW   // Unknown state — needs manual investigation
}
```

```java
/**
 * Reconciliation job: finds payments stuck in SUBMITTED state
 * and queries the payment network for their actual status.
 */
@Scheduled(fixedDelay = 60_000) // every minute
public void reconcileStuckPayments() {
    List<Payment> stuck = paymentRepo.findByStatusAndSubmittedBefore(
        PaymentStatus.SUBMITTED,
        Instant.now().minus(Duration.ofMinutes(5))  // stuck for >5 min
    );

    for (Payment payment : stuck) {
        try {
            PaymentGateway gateway = gatewayRouter.resolve(payment);
            GatewayStatusResponse status = gateway.queryStatus(
                payment.getNetworkReference());

            switch (status.getState()) {
                case CONFIRMED -> completePayment(payment);
                case REJECTED  -> failPayment(payment);
                case UNKNOWN   -> escalateForReview(payment);
            }
        } catch (Exception e) {
            log.error("Failed to reconcile payment {}", payment.getId(), e);
            // Will retry on next cycle
        }
    }
}
```

### 6.3 High Availability & Scalability

#### Availability Requirements

Banking systems typically target **99.99% uptime** (roughly 52 minutes of downtime per year).

```
99.9%  = 8h 45m downtime/year  (unacceptable for banking)
99.99% = 52m downtime/year     (target for most banking services)
99.999% = 5m downtime/year     (target for card authorization)
```

#### Strategies

1. **No single point of failure** — every component runs on 2+ instances
2. **Database replication** — primary + at least one synchronous replica
3. **Multi-zone deployment** — survive a datacenter outage
4. **Graceful degradation** — if the fraud engine is down, process low-risk payments with basic rules; queue high-risk ones
5. **Blue-green deployments** — deploy new versions without downtime

#### Scaling Card Authorization

Card authorization is the most latency-sensitive operation. Every millisecond counts.

```
Typical latency budget for card authorization:
┌──────────────────────────────┬──────────┐
│ Network transit              │  200ms   │
│ Your processing              │  100ms   │  ← This is YOUR budget
│ Database lookup              │   20ms   │
│ Fraud check                  │   30ms   │
│ Balance check                │   10ms   │
│ Response formatting          │    5ms   │
│ Buffer for retries/jitter    │   35ms   │
└──────────────────────────────┴──────────┘
Total: ~400ms (network wants response within 2-5 seconds)
```

Tips for keeping authorization fast:
- **Cache hot accounts** in Redis (but invalidate carefully)
- **Pre-compute fraud scores** rather than running full models inline
- **Use connection pooling** aggressively (HikariCP)
- **Avoid distributed transactions** in the authorization path

---

## 7. Security in Fintech

### 7.1 Encryption

#### At Rest

All sensitive data stored in databases must be encrypted:

```java
@Entity
public class Customer {

    @Id
    private Long id;

    private String name; // Not sensitive in isolation

    @Convert(converter = EncryptedStringConverter.class)
    private String nationalId; // PII — must be encrypted

    @Convert(converter = EncryptedStringConverter.class)
    private String phoneNumber; // PII
}

@Converter
public class EncryptedStringConverter implements AttributeConverter<String, String> {

    private final EncryptionService encryptionService;

    @Override
    public String convertToDatabaseColumn(String attribute) {
        return encryptionService.encrypt(attribute);
    }

    @Override
    public String convertToEntityAttribute(String dbData) {
        return encryptionService.decrypt(dbData);
    }
}
```

**In practice**, many banks use:
- **Transparent Data Encryption (TDE)** at the database level
- **Application-level encryption** for highly sensitive fields (card numbers, national IDs)
- **HSM (Hardware Security Module)** for key management — encryption keys never exist in plain text in software

#### In Transit

```yaml
# application.yml — enforce TLS
server:
  ssl:
    enabled: true
    protocol: TLS
    enabled-protocols: TLSv1.3,TLSv1.2
    ciphers: TLS_AES_256_GCM_SHA384,TLS_AES_128_GCM_SHA256
```

All internal service-to-service communication should also use TLS (mutual TLS / mTLS is preferred).

### 7.2 Secure Coding Practices

```java
// 1. Never log sensitive data
log.info("Customer {} transferred {} UZS", customerId, amount);       // OK
log.info("Card number: {}", cardNumber);                               // NEVER

// 2. Parameterize all queries (prevent SQL injection)
// WRONG:
String query = "SELECT * FROM accounts WHERE id = '" + accountId + "'";
// RIGHT (JPA handles parameterization):
@Query("SELECT a FROM Account a WHERE a.id = :id")
Account findById(@Param("id") Long id);

// 3. Validate all input
@PostMapping("/transfers")
public ResponseEntity<?> transfer(@Valid @RequestBody TransferRequest req) {
    // @Valid + Bean Validation annotations handle input validation
}

public class TransferRequest {
    @NotNull
    @Positive
    @DecimalMax("10000000") // max 10M UZS per transaction
    private BigDecimal amount;

    @NotBlank
    @Pattern(regexp = "^[0-9]{20}$") // 20-digit account number
    private String toAccount;
}

// 4. Use constant-time comparison for sensitive values
// WRONG (timing attack vulnerable):
if (providedToken.equals(storedToken)) { ... }
// RIGHT:
if (MessageDigest.isEqual(
        providedToken.getBytes(UTF_8), storedToken.getBytes(UTF_8))) { ... }
```

### 7.3 Authentication & Authorization

#### OAuth2 + JWT Architecture

```
┌────────┐     ┌──────────────┐     ┌──────────────┐
│ Mobile │────▶│  Auth Server │────▶│   User DB    │
│  App   │     │ (Keycloak/   │     │              │
│        │◀────│  custom)     │     │              │
│        │ JWT └──────────────┘     └──────────────┘
│        │
│        │ JWT  ┌──────────────┐
│        │────▶ │  API Gateway │──▶ Microservices
└────────┘      │(validates JWT)│
                └──────────────┘
```

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable()) // Stateless API, no CSRF needed
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/public/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/v1/accounts/**").hasRole("CUSTOMER")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 ->
                oauth2.jwt(jwt -> jwt.jwtAuthenticationConverter(
                    jwtAuthConverter())))
            .build();
    }
}
```

**Important:** In banking, authorization is not just role-based. You need:
- **Account-level access control** — Customer A cannot access Customer B's account
- **Operation-level limits** — even authenticated users have transfer limits
- **Step-up authentication** — high-value transactions require additional verification (OTP, biometric)

```java
@PreAuthorize("@accountSecurity.isOwner(#accountNumber, authentication)")
@GetMapping("/{accountNumber}/transactions")
public List<Transaction> getTransactions(@PathVariable String accountNumber) {
    // Only the account owner can see transactions
}
```

---

## 8. Real-World Scenarios

### 8.1 Preventing Duplicate Transactions

**The problem:** Customer taps "Pay" twice. Network retries a request. A message is delivered twice from Kafka. All of these can cause duplicate charges.

**Solution: Idempotency at every layer**

```java
@Service
public class TransferService {

    @Transactional
    public TransferResult processTransfer(String idempotencyKey,
                                           TransferRequest request) {
        // Layer 1: Application-level check
        Optional<Transfer> existing = transferRepo
            .findByIdempotencyKey(idempotencyKey);
        if (existing.isPresent()) {
            return TransferResult.fromExisting(existing.get());
        }

        // Layer 2: Database unique constraint on idempotency_key
        // Even if two threads pass Layer 1 simultaneously,
        // only one INSERT will succeed
        try {
            Transfer transfer = createAndExecuteTransfer(idempotencyKey, request);
            return TransferResult.success(transfer);
        } catch (DataIntegrityViolationException e) {
            // Unique constraint violation — another thread already processed this
            Transfer existing2 = transferRepo
                .findByIdempotencyKey(idempotencyKey).orElseThrow();
            return TransferResult.fromExisting(existing2);
        }
    }
}
```

### 8.2 Handling Payment Failures and Retries

```java
/**
 * Payment state machine with retry logic.
 *
 * Key principle: Only retry when the error is TRANSIENT.
 * Never retry business declines (insufficient funds, invalid account).
 */
@Service
public class PaymentProcessor {

    public PaymentResult process(PaymentRequest request) {
        Payment payment = createPayment(request);

        for (int attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
            try {
                GatewayResponse response = gateway.submit(payment);

                if (response.isApproved()) {
                    payment.complete(response.getReference());
                    return PaymentResult.success(payment);
                } else {
                    // Business decline — do NOT retry
                    payment.decline(response.getDeclineCode());
                    return PaymentResult.declined(payment);
                }

            } catch (GatewayTimeoutException e) {
                // Timeout — we don't know the status
                log.warn("Attempt {} timed out for payment {}",
                    attempt, payment.getId());

                // Query the gateway to check actual status before retrying
                Optional<GatewayStatus> status = gateway.query(payment.getId());
                if (status.isPresent()) {
                    if (status.get().isApproved()) {
                        payment.complete(status.get().getReference());
                        return PaymentResult.success(payment);
                    } else if (status.get().isDeclined()) {
                        payment.decline(status.get().getDeclineCode());
                        return PaymentResult.declined(payment);
                    }
                }
                // Status unknown — retry
                backoff(attempt);

            } catch (GatewayConnectionException e) {
                // Connection error — safe to retry (request never reached gateway)
                log.warn("Attempt {} connection failed for payment {}",
                    attempt, payment.getId());
                backoff(attempt);
            }
        }

        // All attempts exhausted
        payment.markForReview();
        alertService.notifyOps("Payment {} stuck after {} attempts",
            payment.getId(), MAX_ATTEMPTS);
        return PaymentResult.pending(payment);
    }

    private void backoff(int attempt) {
        try {
            Thread.sleep((long) Math.pow(2, attempt) * 500);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### 8.3 Reconciliation with External Providers

**Reconciliation** = making sure your records match the external provider's records. This is a daily ritual in banking.

```
Your System says:            UzCard says:
TX-001: 50,000 UZS ✓        TX-001: 50,000 UZS ✓     → MATCH
TX-002: 30,000 UZS ✓        TX-002: 30,000 UZS ✓     → MATCH
TX-003: 25,000 UZS ✓        (missing)                  → MISMATCH: exists locally, not at UzCard
(missing)                    TX-004: 15,000 UZS ✓     → MISMATCH: exists at UzCard, not locally
TX-005: 100,000 UZS         TX-005: 99,000 UZS        → MISMATCH: amount differs
```

```java
@Service
@Slf4j
public class ReconciliationService {

    /**
     * Daily reconciliation job.
     * Compares our transaction records with the card network's settlement file.
     */
    @Scheduled(cron = "0 0 6 * * *") // 6 AM daily
    public void reconcileWithUzCard() {
        LocalDate settlementDate = LocalDate.now().minusDays(1); // yesterday

        // 1. Get our transactions
        List<Transaction> ourTxns = transactionRepo
            .findByNetworkAndSettlementDate("UZCARD", settlementDate);

        // 2. Get UzCard's settlement file
        List<SettlementRecord> uzCardRecords = uzCardClient
            .downloadSettlement(settlementDate);

        // 3. Compare
        ReconciliationResult result = reconcile(ourTxns, uzCardRecords);

        // 4. Handle mismatches
        for (Mismatch mismatch : result.getMismatches()) {
            log.warn("Reconciliation mismatch: {}", mismatch);

            switch (mismatch.getType()) {
                case MISSING_AT_NETWORK -> {
                    // We have it, they don't. Might need to void/reverse.
                    mismatch.setAction(Action.INVESTIGATE);
                }
                case MISSING_LOCALLY -> {
                    // They have it, we don't. We missed recording a transaction.
                    mismatch.setAction(Action.CREATE_ADJUSTMENT);
                }
                case AMOUNT_DIFFERS -> {
                    // Amounts don't match. Could be currency conversion, fees.
                    mismatch.setAction(Action.INVESTIGATE);
                }
            }

            mismatchRepo.save(mismatch);
        }

        // 5. Report
        reconciliationReportService.generate(result);

        log.info("Reconciliation complete: {} matched, {} mismatches",
            result.getMatchCount(), result.getMismatches().size());
    }
}
```

### 8.4 Dealing with Inconsistent External Systems

External systems (card networks, government APIs, partner banks) are often:
- **Slow** — response times of 5-30 seconds are not unusual
- **Unreliable** — scheduled downtime, unscheduled downtime, rate limits
- **Inconsistent** — different error codes for the same problem, undocumented behaviors
- **Legacy** — XML/SOAP instead of REST, ISO 8583 binary messages, fixed-width file formats

**Strategy: Build an anti-corruption layer**

```java
/**
 * Anti-corruption layer for UzCard integration.
 * Translates between your clean domain model and UzCard's legacy API.
 */
@Service
public class UzCardAdapter implements PaymentNetworkGateway {

    @Override
    public AuthorizationResponse authorize(AuthorizationRequest request) {
        // Translate to UzCard format
        UzCardAuthMessage uzCardMessage = UzCardMessageBuilder.builder()
            .messageType("0100")                    // ISO 8583 auth request
            .pan(request.getCardNumber())
            .amount(formatAmount(request.getAmount()))
            .currency("860")                         // UZS currency code
            .merchantId(request.getMerchantId())
            .terminalId(request.getTerminalId())
            .build();

        // Call UzCard
        UzCardResponseMessage response = uzCardClient.send(uzCardMessage);

        // Translate back to your domain model
        return AuthorizationResponse.builder()
            .approved("00".equals(response.getResponseCode()))
            .declineReason(mapDeclineCode(response.getResponseCode()))
            .networkReference(response.getRrn())
            .build();
    }

    private String mapDeclineCode(String uzCardCode) {
        return switch (uzCardCode) {
            case "00" -> null; // Approved
            case "51" -> "INSUFFICIENT_FUNDS";
            case "54" -> "EXPIRED_CARD";
            case "61" -> "EXCEEDS_LIMIT";
            case "91" -> "ISSUER_UNAVAILABLE";
            default   -> "UNKNOWN_DECLINE:" + uzCardCode;
        };
    }
}
```

**Timeout and fallback strategy:**

```java
@Service
public class ResilientUzCardGateway {

    @CircuitBreaker(name = "uzcard", fallbackMethod = "fallback")
    @TimeLimiter(name = "uzcard") // configured timeout: e.g., 10 seconds
    @Retry(name = "uzcard")       // only for connection errors, not timeouts
    public CompletableFuture<AuthorizationResponse> authorize(
            AuthorizationRequest request) {
        return CompletableFuture.supplyAsync(() ->
            uzCardAdapter.authorize(request));
    }

    /**
     * Fallback: when UzCard is completely down.
     * Options depend on business rules:
     * - Stand-In Processing: approve low-risk transactions offline
     * - Decline all: safest but worst customer experience
     */
    public CompletableFuture<AuthorizationResponse> fallback(
            AuthorizationRequest request, Throwable t) {
        log.error("UzCard unavailable, applying stand-in rules", t);

        if (standInRules.canApproveOffline(request)) {
            // Record for later reconciliation
            offlineApprovalRepo.save(new OfflineApproval(request));
            return CompletableFuture.completedFuture(
                AuthorizationResponse.approvedStandIn());
        }
        return CompletableFuture.completedFuture(
            AuthorizationResponse.declined("NETWORK_UNAVAILABLE"));
    }
}
```

---

## 9. Glossary

| Term | Definition |
|------|-----------|
| **Acquirer** | Bank that processes card payments on behalf of merchants |
| **AML** | Anti-Money Laundering — detecting and preventing financial crime |
| **Authorization** | Real-time decision to approve or decline a card transaction |
| **Capture** | Confirming a previously authorized transaction for settlement |
| **CBS** | Core Banking System — the central ledger and account management system |
| **Chargeback** | A forced reversal of a payment, initiated by the cardholder's bank |
| **Circuit Breaker** | A pattern that stops calling a failing service to let it recover |
| **Double-entry Bookkeeping** | Every transaction has equal debits and credits, ensuring the books balance |
| **Fintech** | Financial Technology — technology applied to financial services |
| **HSM** | Hardware Security Module — dedicated hardware for cryptographic operations |
| **HUMO** | A local card payment network in Uzbekistan |
| **Idempotency** | The property that performing an operation multiple times has the same effect as performing it once |
| **ISO 8583** | International standard for financial transaction card messaging |
| **Issuer** | The bank that issued the card to the customer |
| **KYC** | Know Your Customer — verifying customer identity before providing services |
| **Ledger** | The immutable record of all financial transactions |
| **mTLS** | Mutual TLS — both client and server verify each other's certificates |
| **Neobank** | A digital-only bank with no physical branches |
| **Outbox Pattern** | Writing events to a database table (outbox) within the same transaction as business data, then publishing to a message broker separately |
| **PAN** | Primary Account Number — the card number |
| **PCI-DSS** | Payment Card Industry Data Security Standard |
| **PSP** | Payment Service Provider — connects merchants to payment methods |
| **Reconciliation** | Comparing your records with an external party's records to find and resolve discrepancies |
| **Refund** | Returning money to a customer after settlement has occurred |
| **Reversal** | Canceling a transaction before settlement |
| **Saga** | A pattern for managing distributed transactions through a sequence of local transactions with compensating actions |
| **Settlement** | The process of transferring net funds between banks, typically in batches |
| **Stand-In Processing** | Approving transactions locally when the network is unavailable, based on pre-defined rules |
| **TDE** | Transparent Data Encryption — database-level encryption |
| **UzCard** | The dominant local card payment network in Uzbekistan |
| **UZS** | Uzbekistani So'm — the national currency of Uzbekistan |

---

> **Final advice:** When in doubt, remember the two cardinal rules of fintech engineering:
> 1. **Money must never be created or destroyed** — every debit has a credit.
> 2. **When you don't know the state, don't guess** — query, reconcile, and escalate.
>
> Welcome to fintech. Build carefully.
