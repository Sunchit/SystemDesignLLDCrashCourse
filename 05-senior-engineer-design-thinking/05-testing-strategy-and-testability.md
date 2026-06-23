# Testing Strategy and Designing for Testability

## Introduction

Testing is not a phase that happens after development. It is a design discipline. The quality of your tests reflects the quality of your design, and the difficulty of writing tests is a direct measurement of how coupled, opaque, and rigid your system is. Senior engineers treat testability as a first-order design concern, alongside performance, scalability, and correctness.

This document covers testing strategy end-to-end: from the principles that make code testable, through the mechanics of unit and integration testing, to full TDD walkthroughs. Every concept is grounded in Java and applied to realistic production scenarios.

---

## 1. Testability as a Design Quality Metric

### Testability Is a First-Order Design Concern

Most engineers treat testing as something done after the code is written. This is backwards. Testability is a property of the design itself, and it must be considered at every design decision — when choosing between static and instance methods, when deciding how to handle time, when deciding whether a class should create its own dependencies or receive them.

If you defer testability decisions to the end, you will find that the code structure resists change. You will write tests that are brittle, slow, and dependent on environment state. You will write tests that require spinning up an entire application to verify a single calculation.

### The Cost of Untestable Code

Untestable code carries compounding costs:

- **Slow feedback loops.** Without fast unit tests, developers rely on manual testing or integration test suites that take minutes or hours. Bugs discovered later are far more expensive to fix.
- **Regression fear.** Without test coverage, every change to a class carries the risk of silently breaking something. Developers become conservative and stop refactoring. The codebase ossifies.
- **Debugging tax.** Untestable code is usually tightly coupled. When something breaks, the failure surface is large. Narrowing down the bug requires significant manual effort.
- **Onboarding friction.** New engineers cannot understand what a component does in isolation. They must trace through the full call graph to understand a single behavior.

The cost of not testing is not zero. It is deferred and hidden, which makes it worse.

### How Testability Correlates with Good Design

Testability is not a separate concern from design quality. It is the same thing measured differently.

- **High cohesion** means a class has one clear responsibility. A class with one responsibility is easy to instantiate in a test with minimal setup. A class with ten responsibilities requires ten times the setup and tests that are hard to read.
- **Low coupling** means dependencies are injected, not created. Injected dependencies can be replaced with test doubles. Created dependencies cannot.
- **Explicit dependencies** mean that everything a class needs to do its job is visible at the constructor boundary. Hidden dependencies — static calls, global state, thread-local context — cannot be observed or replaced in tests.

When a class is hard to test, it is almost always because it violates one of these principles. The test difficulty is the signal; the design violation is the cause.

### The "Hard to Test, Hard to Change" Principle

There is a tight correlation between testable code and changeable code. The structural properties that make code testable — dependency injection, small classes, pure functions, explicit interfaces — are the same properties that make code easy to modify.

When you add a feature to a tightly coupled, statically-wired system, you must modify multiple classes in coordination. When you add a feature to a loosely coupled, injected system, you modify or add the relevant class, and the test tells you immediately whether you broke anything.

This is why senior engineers push back hard on "we'll add tests later." The design decisions that make code untestable are made during development. Once the structure is in place, retrofitting tests requires refactoring the design, not just writing test cases.

---

## 2. What Makes Code Hard to Test

### 2.1 Static Methods: Hidden Dependencies, No Substitution

Static methods look convenient. They require no instantiation and are called directly. The problem is that callers cannot substitute them in tests.

```java
// Untestable: static dependency
public class DiscountCalculator {
    public double calculate(Order order) {
        User user = UserRepository.findById(order.getUserId()); // static call to DB
        if (user.isPremium()) {
            return order.getTotal() * 0.10;
        }
        return 0;
    }
}
```

`UserRepository.findById` is a static call. There is no way to test `DiscountCalculator.calculate` without hitting the database. You cannot pass a fake `UserRepository`. You cannot control what `findById` returns. Every test becomes an integration test.

**The refactor: inject the dependency as an instance.**

```java
public class DiscountCalculator {
    private final UserRepository userRepository;

    public DiscountCalculator(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public double calculate(Order order) {
        User user = userRepository.findById(order.getUserId());
        if (user.isPremium()) {
            return order.getTotal() * 0.10;
        }
        return 0;
    }
}
```

Now `UserRepository` is an interface. In the test, you inject a mock. In production, you inject the real implementation.

### 2.2 Singleton Pattern: Global State and Test Order Dependency

Singletons that hold mutable state are a testing hazard. State modified in one test leaks into the next.

```java
// Untestable: mutable singleton
public class RateLimiter {
    private static final RateLimiter INSTANCE = new RateLimiter();
    private final Map<String, Integer> requestCounts = new HashMap<>();

    private RateLimiter() {}

    public static RateLimiter getInstance() {
        return INSTANCE;
    }

    public boolean isAllowed(String clientId) {
        int count = requestCounts.getOrDefault(clientId, 0);
        if (count >= 100) return false;
        requestCounts.put(clientId, count + 1);
        return true;
    }
}
```

If `testA` calls `isAllowed("client1")` 99 times, `testB` will find `client1` already at 99 requests and fail in ways that depend entirely on test execution order. The test suite becomes non-deterministic.

**The refactor: instantiate per test, inject where needed.**

```java
public class RateLimiter {
    private final Map<String, Integer> requestCounts = new HashMap<>();
    private final int limit;

    public RateLimiter(int limit) {
        this.limit = limit;
    }

    public boolean isAllowed(String clientId) {
        int count = requestCounts.getOrDefault(clientId, 0);
        if (count >= limit) return false;
        requestCounts.put(clientId, count + 1);
        return true;
    }
}
```

Each test creates its own `RateLimiter` instance. State is completely isolated between tests.

### 2.3 Tight Coupling: `new` Keyword Inside Logic

Calling `new` inside a business method hard-wires the class to a specific implementation.

```java
// Untestable: new inside business logic
public class OrderService {
    public void placeOrder(Order order) {
        PaymentGateway gateway = new StripePaymentGateway(); // hard-wired
        gateway.charge(order.getTotal(), order.getPaymentMethod());
        new OrderRepository().save(order); // another hidden instantiation
    }
}
```

There is no way to test `placeOrder` without making real calls to Stripe and a real database. The class controls its own dependencies. Tests have no leverage.

**The refactor: receive dependencies from outside.**

```java
public class OrderService {
    private final PaymentGateway paymentGateway;
    private final OrderRepository orderRepository;

    public OrderService(PaymentGateway paymentGateway, OrderRepository orderRepository) {
        this.paymentGateway = paymentGateway;
        this.orderRepository = orderRepository;
    }

    public void placeOrder(Order order) {
        paymentGateway.charge(order.getTotal(), order.getPaymentMethod());
        orderRepository.save(order);
    }
}
```

### 2.4 Hidden Dependencies: Time, Randomness, File System

`System.currentTimeMillis()`, `new Date()`, `Math.random()`, and direct file I/O are all non-deterministic or environment-dependent. Any code that uses them directly is non-deterministic in tests.

```java
// Untestable: hidden time dependency
public class SessionManager {
    public boolean isSessionValid(Session session) {
        long now = System.currentTimeMillis(); // not injectable, not controllable
        return now - session.getCreatedAt() < 30 * 60 * 1000;
    }
}
```

You cannot write a reliable test for `isSessionValid` that exercises the "session has expired" path without sleeping for 30 minutes or manipulating system time.

**The refactor: inject a `Clock`.**

```java
public class SessionManager {
    private final Clock clock;

    public SessionManager(Clock clock) {
        this.clock = clock;
    }

    public boolean isSessionValid(Session session) {
        long now = clock.millis();
        return now - session.getCreatedAt() < 30 * 60 * 1000;
    }
}

// In test:
Clock fixedClock = Clock.fixed(Instant.parse("2025-01-01T10:00:00Z"), ZoneOffset.UTC);
SessionManager manager = new SessionManager(fixedClock);
```

Now you can set the clock to any instant. You can test "just barely valid", "just barely expired", and "created in the future" without sleeping.

### 2.5 God Classes: Too Many Responsibilities

A class that does too much is impossible to test in isolation because every test requires the full weight of all its responsibilities.

```java
// Untestable: God class
public class OrderProcessor {
    public void process(Order order) {
        // validates order
        // computes discount
        // calls payment gateway
        // updates inventory
        // sends email
        // updates analytics
        // writes audit log
    }
}
```

A test for the discount calculation must also set up a working payment gateway, inventory service, email client, and analytics sink. The feedback loop collapses. Every test becomes an end-to-end test by accident.

**The refactor: decompose into focused, injectable collaborators, each of which can be tested in isolation.**

---

## 3. Design Decisions That Enable Testing

### 3.1 Dependency Injection: Constructor Injection Preferred

There are three common forms of dependency injection: constructor injection, setter injection, and field injection. Constructor injection is preferred for testability because:

- All dependencies are visible at the construction site. There are no hidden fields set after construction.
- The object is fully initialized at construction time. There is no window where it exists in a partially valid state.
- The compiler enforces that all required dependencies are provided. There is no `NullPointerException` at runtime because a setter was not called.
- Test code does not need a DI framework. You `new` the class directly with the dependencies you want.

```java
// Preferred: constructor injection
public class NotificationService {
    private final EmailClient emailClient;
    private final SmsClient smsClient;
    private final Clock clock;

    public NotificationService(EmailClient emailClient, SmsClient smsClient, Clock clock) {
        this.emailClient = emailClient;
        this.smsClient = smsClient;
        this.clock = clock;
    }
}

// In test: no Spring context, no reflection, no magic
NotificationService service = new NotificationService(
    mockEmailClient,
    mockSmsClient,
    Clock.fixed(Instant.now(), ZoneOffset.UTC)
);
```

### 3.2 Dependency Inversion: Depend on Abstractions

The dependency inversion principle (the D in SOLID) says that high-level modules should not depend on low-level modules. Both should depend on abstractions.

In practice: your business logic classes should depend on interfaces, not on concrete implementations. This allows tests to provide alternative implementations that are fast, deterministic, and observable.

```java
// Interface (abstraction)
public interface PaymentGateway {
    PaymentResult charge(Money amount, PaymentMethod method);
}

// Production implementation
public class StripePaymentGateway implements PaymentGateway {
    public PaymentResult charge(Money amount, PaymentMethod method) {
        // real HTTP call to Stripe
    }
}

// The business class depends on the interface, not Stripe
public class OrderService {
    private final PaymentGateway paymentGateway; // interface, not StripePaymentGateway

    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

### 3.3 Clock Abstraction

`java.time.Clock` is the standard Java abstraction for time. It can be fixed, offset, or ticking. Inject it everywhere you need the current time.

```java
public class TokenExpiryService {
    private final Clock clock;
    private final Duration tokenLifetime;

    public TokenExpiryService(Clock clock, Duration tokenLifetime) {
        this.clock = clock;
        this.tokenLifetime = tokenLifetime;
    }

    public boolean isExpired(Token token) {
        Instant now = clock.instant();
        return now.isAfter(token.getIssuedAt().plus(tokenLifetime));
    }

    public Instant calculateExpiry(Instant issuedAt) {
        return issuedAt.plus(tokenLifetime);
    }
}
```

In tests:
```java
Instant baseTime = Instant.parse("2025-06-01T12:00:00Z");
Clock fixedClock = Clock.fixed(baseTime, ZoneOffset.UTC);
TokenExpiryService service = new TokenExpiryService(fixedClock, Duration.ofHours(1));

Token token = new Token(baseTime.minusSeconds(3601)); // issued 3601 seconds ago
assertTrue(service.isExpired(token)); // one second past the hour window
```

### 3.4 Random Abstraction

For anything involving randomness, inject a `java.util.Random` (or a `java.util.function.Supplier<Double>` for flexibility). In tests, use a seeded `Random` for determinism.

```java
public class CouponCodeGenerator {
    private final Random random;

    public CouponCodeGenerator(Random random) {
        this.random = random;
    }

    public String generate() {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 8; i++) {
            sb.append((char) ('A' + random.nextInt(26)));
        }
        return sb.toString();
    }
}

// In test:
Random seeded = new Random(42L); // same seed = same sequence every time
CouponCodeGenerator generator = new CouponCodeGenerator(seeded);
assertEquals("XGKPTFIM", generator.generate()); // deterministic
```

### 3.5 File System Abstraction

Instead of calling `new File(path)` or `Paths.get(path)` directly, accept `Path` or `java.nio.file.FileSystem` as a dependency. In tests, use `Jimfs` (an in-memory file system from Google) or a temp directory.

```java
public class ConfigLoader {
    private final Path configPath;

    public ConfigLoader(Path configPath) {
        this.configPath = configPath;
    }

    public Map<String, String> load() throws IOException {
        return Files.readAllLines(configPath)
            .stream()
            .filter(line -> line.contains("="))
            .collect(Collectors.toMap(
                line -> line.split("=")[0].trim(),
                line -> line.split("=")[1].trim()
            ));
    }
}

// In test using a temp file:
@Test
void should_load_key_value_pairs() throws IOException {
    Path tempFile = Files.createTempFile("config", ".properties");
    Files.writeString(tempFile, "host=localhost\nport=8080");

    ConfigLoader loader = new ConfigLoader(tempFile);
    Map<String, String> config = loader.load();

    assertEquals("localhost", config.get("host"));
    assertEquals("8080", config.get("port"));
}
```

### 3.6 Separating Side Effects from Pure Logic

Pure functions — those that take inputs and return outputs with no side effects — are the easiest thing to test. You call them with inputs and assert on outputs. No mocks, no setup, no teardown.

The key technique is to separate computation from side effects. Move the logic that decides *what to do* into pure methods. Move the code that *does it* into a thin coordination layer.

```java
// Hard to test: logic and side effects mixed
public class PricingService {
    public void applyDiscount(Order order) {
        if (order.getTotal().compareTo(new BigDecimal("100")) > 0) {
            BigDecimal discount = order.getTotal().multiply(new BigDecimal("0.10"));
            order.setTotal(order.getTotal().subtract(discount));
            orderRepository.save(order); // side effect mixed with calculation
        }
    }
}

// Easy to test: pure calculation separated from side effect
public class PricingCalculator {
    // Pure function: no dependencies, no side effects
    public BigDecimal calculateDiscount(BigDecimal orderTotal) {
        if (orderTotal.compareTo(new BigDecimal("100")) > 0) {
            return orderTotal.multiply(new BigDecimal("0.10"));
        }
        return BigDecimal.ZERO;
    }
}

public class PricingService {
    private final PricingCalculator calculator;
    private final OrderRepository repository;

    public PricingService(PricingCalculator calculator, OrderRepository repository) {
        this.calculator = calculator;
        this.repository = repository;
    }

    public void applyDiscount(Order order) {
        BigDecimal discount = calculator.calculateDiscount(order.getTotal());
        if (discount.compareTo(BigDecimal.ZERO) > 0) {
            order.setTotal(order.getTotal().subtract(discount));
            repository.save(order);
        }
    }
}
```

`PricingCalculator.calculateDiscount` has no dependencies. It is testable without any mocking infrastructure.

### 3.7 Full Refactor: OrderProcessor

Here is a full before/after refactor of a hard-to-test `OrderProcessor`:

**Before: untestable**

```java
public class OrderProcessor {
    public void process(String orderId) {
        // Fetches from DB directly
        Order order = Database.query("SELECT * FROM orders WHERE id = " + orderId);

        // Applies discount using static utility
        double discount = DiscountUtils.calculate(order);

        // Calls payment gateway directly
        boolean paid = new StripeGateway().charge(order.getUserId(), order.getTotal() - discount);

        if (paid) {
            // Updates inventory with static call
            InventorySystem.deduct(order.getItems());
            // Sends email with hard-coded SMTP
            new SmtpEmailSender("smtp.mycompany.com").send(
                order.getUserEmail(),
                "Order Confirmed",
                "Your order " + orderId + " has been placed."
            );
            // Writes to audit log file
            new FileWriter("/var/log/orders.log").write(orderId + " processed at " + new Date());
        }
    }
}
```

**After: testable**

```java
// Interfaces for all external collaborators
public interface OrderRepository {
    Order findById(String orderId);
    void save(Order order);
}

public interface PaymentGateway {
    PaymentResult charge(String userId, BigDecimal amount);
}

public interface InventoryService {
    void deduct(List<OrderItem> items);
}

public interface EmailService {
    void send(String to, String subject, String body);
}

public interface AuditLogger {
    void logOrderProcessed(String orderId, Instant timestamp);
}

// Pure business logic: testable without any mocking
public class DiscountCalculator {
    public BigDecimal calculate(Order order) {
        if (order.isEligibleForDiscount()) {
            return order.getTotal().multiply(new BigDecimal("0.10"));
        }
        return BigDecimal.ZERO;
    }
}

// Coordinator: thin, testable with mocks
public class OrderProcessor {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final InventoryService inventoryService;
    private final EmailService emailService;
    private final AuditLogger auditLogger;
    private final DiscountCalculator discountCalculator;
    private final Clock clock;

    public OrderProcessor(
            OrderRepository orderRepository,
            PaymentGateway paymentGateway,
            InventoryService inventoryService,
            EmailService emailService,
            AuditLogger auditLogger,
            DiscountCalculator discountCalculator,
            Clock clock) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
        this.inventoryService = inventoryService;
        this.emailService = emailService;
        this.auditLogger = auditLogger;
        this.discountCalculator = discountCalculator;
        this.clock = clock;
    }

    public void process(String orderId) {
        Order order = orderRepository.findById(orderId);
        BigDecimal discount = discountCalculator.calculate(order);
        BigDecimal chargeAmount = order.getTotal().subtract(discount);

        PaymentResult result = paymentGateway.charge(order.getUserId(), chargeAmount);

        if (result.isSuccess()) {
            inventoryService.deduct(order.getItems());
            emailService.send(
                order.getUserEmail(),
                "Order Confirmed",
                "Your order " + orderId + " has been placed."
            );
            auditLogger.logOrderProcessed(orderId, clock.instant());
        }
    }
}
```

Each collaborator is an interface. Each can be mocked independently. `DiscountCalculator` has no dependencies at all. `OrderProcessor` can be tested by verifying that it calls the right collaborators with the right arguments.

---

## 4. The Test Pyramid

The test pyramid is a model for the distribution of tests in a healthy codebase. It describes the relative count, speed, and scope of tests at three levels.

### The Pyramid

```
           /\
          /E2E\         <- Few, slow, expensive, high confidence in integration
         /------\
        /  Integ  \     <- Moderate number, medium speed, test component boundaries
       /------------\
      /  Unit Tests  \  <- Many, fast, cheap, test isolated behavior
     /-----------------\
```

### Unit Tests: Fast, Isolated, Many

Unit tests test a single class or a small cluster of closely related classes in complete isolation from external dependencies. All collaborators are replaced with test doubles.

- Execution time: milliseconds per test.
- Target: hundreds to thousands per service.
- Dependency: no database, no network, no file system.
- Value: fast feedback, precise failure localization, drives design through pressure.

### Integration Tests: Slower, Real Dependencies, Fewer

Integration tests verify that two or more components work correctly together. They use real implementations for some dependencies — a real database, a real message queue, a real HTTP server — while still bounding the scope.

- Execution time: seconds per test.
- Target: tens to low hundreds per service.
- Dependency: typically a database, sometimes a message queue.
- Value: catches issues that unit tests miss — SQL query correctness, ORM mapping errors, serialization mismatches.

### End-to-End Tests: Slowest, Full Stack, Fewest

End-to-end tests exercise the full system from the outside: HTTP requests against a running service, with real databases and real external dependencies (or high-fidelity stubs). They verify that user-visible behaviors work across the full call graph.

- Execution time: seconds to minutes per test.
- Target: a small number of critical user journeys (e.g., 10-30 per service).
- Dependency: everything running, or close to it.
- Value: catch deployment and configuration errors, validate critical paths.

### The Anti-Pattern: Inverted Pyramid

The inverted pyramid is a common failure mode in organizations without a deliberate testing strategy. It looks like this:

```
     /-----------------\
    /   E2E Tests (many) \
   /----------------------\
  /  Integration Tests (few) \
 /----------------------------\
/   Unit Tests (almost none)   \
```

Symptoms: the CI pipeline takes 45 minutes to run. Flaky tests that fail intermittently dominate team focus. When a test fails, it is not obvious which of the 50 components involved is responsible. Engineers stop running tests locally because they are too slow. The feedback loop collapses.

### Guideline Ratios

These are approximations, not rules:

| Level       | Proportion | Execution Time | Typical Count |
|-------------|------------|----------------|---------------|
| Unit        | 70-80%     | < 100ms each   | 500-2000      |
| Integration | 15-20%     | 1-10s each     | 50-200        |
| E2E         | 5-10%      | 10s-2min each  | 10-30         |

The goal is maximum confidence at minimum cost. Unit tests are cheap. Invest heavily in them. E2E tests are expensive. Reserve them for the most critical paths.

---

## 5. Test-Driven Development

TDD is a development technique, not just a testing technique. Its primary value is that it forces you to think about how code will be used before you write it, producing designs with better interfaces and fewer hidden dependencies.

### Red-Green-Refactor Cycle

1. **Red**: Write a failing test for the next small behavior. Run the test suite. See the new test fail (and only the new test). This confirms the test is actually testing something.
2. **Green**: Write the minimum production code to make the test pass. Do not over-engineer. Do not add behavior not required by the test.
3. **Refactor**: Clean up the code without changing behavior. Eliminate duplication, improve naming, extract abstractions. Run tests after every change to confirm they still pass.

Repeat. Each cycle should take minutes, not hours.

### Outside-In TDD (London School)

Outside-in TDD starts from the outermost boundary of the feature — a failing acceptance test or integration test — and drives inward toward the domain. At each layer, you discover the interfaces needed by the layer above and implement them.

This style emphasizes collaboration and design. The acceptance test defines what "done" looks like. As you implement inward, you discover what interfaces lower layers must provide. This produces well-defined contracts between layers.

It requires comfort with mocking: inner layers are mocked while outer layers are being implemented, then inner layers are filled in and mocked when implementing deeper layers.

### Inside-Out TDD (Chicago School)

Inside-out TDD starts from the core domain logic — the most stable, most behavior-rich part of the system — and drives outward toward infrastructure and coordination layers.

This style emphasizes domain correctness. The domain is tested without mocks (it has no external dependencies), producing a highly cohesive core. The outer layers are implemented once the core is solid.

It can lead to design discovery: you find the right domain model by writing tests for it and seeing what interfaces emerge naturally.

### When TDD Is Most Valuable

- **Complex business logic**: discount rules, pricing algorithms, workflow state machines. The logic is non-trivial, and incorrect behavior is easy to ship without tests.
- **Refactoring**: before refactoring existing code, write tests that describe current behavior. Then refactor. Green tests confirm you did not change semantics.
- **Bug fixing**: write a failing test that reproduces the bug before fixing it. The test becomes a permanent regression guard.

### When TDD Can Slow You Down

- **Exploratory code**: when you do not yet know the right design, TDD can be premature. Spike first (without tests), learn the shape of the solution, then delete the spike and TDD the real implementation.
- **UI layout and rendering**: the surface area is visual, not behavioral. Screenshot tests are fragile. Write tests for UI behavior (button clicks, form validation), not for pixel layout.
- **Research spikes**: when you are integrating with an unfamiliar external API and need to understand its behavior, spike first.

### Full TDD Example: RateLimiter (Token Bucket)

We will implement a token bucket rate limiter using TDD. Each step shows the test first, then the minimum implementation.

**Token bucket algorithm**: the bucket holds up to `capacity` tokens. Tokens are added at a rate of `refillRate` per second. Each request consumes one token. If the bucket is empty, the request is rejected.

**Step 1: Red — first request is allowed when bucket has capacity**

```java
// RateLimiterTest.java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;
import java.time.Clock;
import java.time.Instant;
import java.time.ZoneOffset;

class RateLimiterTest {

    @Test
    void should_allow_request_when_bucket_has_tokens() {
        Clock clock = Clock.fixed(Instant.parse("2025-01-01T00:00:00Z"), ZoneOffset.UTC);
        RateLimiter limiter = new RateLimiter(10, 1.0, clock); // capacity=10, refill=1/sec

        boolean allowed = limiter.tryAcquire();

        assertTrue(allowed);
    }
}
```

**Minimum implementation to pass:**

```java
public class RateLimiter {
    private final int capacity;
    private final double refillRate;
    private final Clock clock;
    private double tokens;
    private Instant lastRefillTime;

    public RateLimiter(int capacity, double refillRate, Clock clock) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.clock = clock;
        this.tokens = capacity;
        this.lastRefillTime = clock.instant();
    }

    public boolean tryAcquire() {
        if (tokens >= 1) {
            tokens -= 1;
            return true;
        }
        return false;
    }
}
```

**Step 2: Red — request is rejected when bucket is empty**

```java
@Test
void should_reject_request_when_bucket_is_empty() {
    Clock clock = Clock.fixed(Instant.parse("2025-01-01T00:00:00Z"), ZoneOffset.UTC);
    RateLimiter limiter = new RateLimiter(2, 1.0, clock); // capacity=2

    limiter.tryAcquire(); // consume token 1
    limiter.tryAcquire(); // consume token 2
    boolean allowed = limiter.tryAcquire(); // bucket empty

    assertFalse(allowed);
}
```

This test passes with the current implementation — `tryAcquire` already handles the empty case.

**Step 3: Red — tokens refill after time passes**

```java
@Test
void should_allow_request_after_tokens_are_refilled() {
    Instant start = Instant.parse("2025-01-01T00:00:00Z");

    // First, drain the bucket
    Clock startClock = Clock.fixed(start, ZoneOffset.UTC);
    RateLimiter limiter = new RateLimiter(1, 1.0, startClock); // capacity=1, 1 token/sec
    limiter.tryAcquire(); // drain

    // Now advance time by 1 second and use a new clock for the next call
    // We need a mutable clock for this — use a wrapper
    MutableClock mutableClock = new MutableClock(start);
    RateLimiter limiter2 = new RateLimiter(1, 1.0, mutableClock);
    limiter2.tryAcquire(); // drain

    mutableClock.advance(Duration.ofSeconds(1)); // 1 second passes
    boolean allowed = limiter2.tryAcquire(); // should be refilled

    assertTrue(allowed);
}
```

We need a `MutableClock` helper for tests:

```java
// Test helper: not in production code
class MutableClock extends Clock {
    private Instant now;

    MutableClock(Instant start) {
        this.now = start;
    }

    public void advance(Duration duration) {
        this.now = now.plus(duration);
    }

    @Override
    public ZoneId getZone() { return ZoneOffset.UTC; }

    @Override
    public Clock withZone(ZoneId zone) { return this; }

    @Override
    public Instant instant() { return now; }
}
```

**Updated implementation to pass — add refill logic:**

```java
public boolean tryAcquire() {
    refill();
    if (tokens >= 1) {
        tokens -= 1;
        return true;
    }
    return false;
}

private void refill() {
    Instant now = clock.instant();
    double elapsed = Duration.between(lastRefillTime, now).toNanos() / 1_000_000_000.0;
    double tokensToAdd = elapsed * refillRate;
    tokens = Math.min(capacity, tokens + tokensToAdd);
    lastRefillTime = now;
}
```

**Step 4: Red — tokens do not exceed capacity on refill**

```java
@Test
void should_not_exceed_capacity_when_refilling() {
    Instant start = Instant.parse("2025-01-01T00:00:00Z");
    MutableClock clock = new MutableClock(start);
    RateLimiter limiter = new RateLimiter(5, 1.0, clock); // capacity=5

    clock.advance(Duration.ofSeconds(100)); // 100 tokens would be added, but cap is 5

    // Should allow exactly 5 requests
    for (int i = 0; i < 5; i++) {
        assertTrue(limiter.tryAcquire(), "Request " + (i + 1) + " should be allowed");
    }
    assertFalse(limiter.tryAcquire(), "Request 6 should be rejected");
}
```

This passes with the current implementation because of `Math.min(capacity, ...)`.

**Step 5: Red — partial token accumulation**

```java
@Test
void should_accumulate_fractional_tokens_over_time() {
    Instant start = Instant.parse("2025-01-01T00:00:00Z");
    MutableClock clock = new MutableClock(start);
    RateLimiter limiter = new RateLimiter(10, 2.0, clock); // 2 tokens per second, start full

    // Drain all tokens
    for (int i = 0; i < 10; i++) {
        limiter.tryAcquire();
    }

    // Advance by 0.4 seconds — should accumulate 0.8 tokens, not enough for one
    clock.advance(Duration.ofMillis(400));
    assertFalse(limiter.tryAcquire());

    // Advance by another 0.1 seconds — total 0.5s = 1.0 tokens, enough for one
    clock.advance(Duration.ofMillis(100));
    assertTrue(limiter.tryAcquire());
}
```

**Step 6: Refactor**

After all tests pass, refactor for clarity:

```java
public class RateLimiter {
    private final int capacity;
    private final double refillRatePerSecond;
    private final Clock clock;
    private double availableTokens;
    private Instant lastRefillTimestamp;

    public RateLimiter(int capacity, double refillRatePerSecond, Clock clock) {
        if (capacity <= 0) throw new IllegalArgumentException("Capacity must be positive");
        if (refillRatePerSecond <= 0) throw new IllegalArgumentException("Refill rate must be positive");

        this.capacity = capacity;
        this.refillRatePerSecond = refillRatePerSecond;
        this.clock = clock;
        this.availableTokens = capacity;
        this.lastRefillTimestamp = clock.instant();
    }

    public synchronized boolean tryAcquire() {
        refillTokens();
        if (availableTokens >= 1.0) {
            availableTokens -= 1.0;
            return true;
        }
        return false;
    }

    private void refillTokens() {
        Instant now = clock.instant();
        double secondsElapsed = Duration.between(lastRefillTimestamp, now).toNanos() / 1_000_000_000.0;
        double tokensToAdd = secondsElapsed * refillRatePerSecond;
        availableTokens = Math.min(capacity, availableTokens + tokensToAdd);
        lastRefillTimestamp = now;
    }
}
```

---

## 6. Unit Testing Principles

### FIRST

**F — Fast**: Unit tests must run in milliseconds. A slow unit test has a hidden dependency on something external. Find and remove it.

**I — Isolated**: Each test must be independent. It must not depend on the result of any other test. It must be able to run in any order, alone, or as part of the full suite. Tests that share state (through static fields, singletons, or test order) are not isolated.

**R — Repeatable**: The same test, run on any machine, at any time, must produce the same result. Non-repeatable tests depend on environment: the current time, available network, file system state, or random values.

**S — Self-validating**: Tests must produce a binary pass/fail result. A test that requires a human to inspect log output to determine whether it passed is not a test.

**T — Timely**: Tests must be written at the same time as the code they test, or before. Tests written six months after the code are documentation, not a design tool.

### One Logical Assertion Per Test

The guideline — not a strict rule — is that each test should verify one thing. This makes the test name meaningful (it says exactly what is being verified), makes failures precise (you know what broke immediately), and keeps tests small and readable.

"One logical assertion" does not mean one `assertEquals` call. It means one logical concept. If you are asserting that an order was confirmed, you might check both the status field and the confirmation timestamp — two assertions, one concept.

### Arrange-Act-Assert (AAA) Pattern

Every test should have three distinct sections:

```java
@Test
void should_apply_10_percent_discount_when_order_exceeds_100() {
    // Arrange: set up the system under test and its inputs
    DiscountCalculator calculator = new DiscountCalculator();
    BigDecimal orderTotal = new BigDecimal("150.00");

    // Act: invoke the behavior under test
    BigDecimal discount = calculator.calculateDiscount(orderTotal);

    // Assert: verify the expected outcome
    assertEquals(new BigDecimal("15.00"), discount);
}
```

Separating these sections with blank lines or comments makes the test easy to scan. A test where setup and assertion are intermixed is difficult to read and often indicative of a poorly structured test.

### Test Naming Convention

Use the convention: `should_[expectedBehavior]_when_[condition]`.

```
should_allow_request_when_bucket_has_tokens
should_reject_request_when_bucket_is_empty
should_apply_10_percent_discount_when_order_exceeds_100
should_throw_InsufficientFundsException_when_balance_is_negative
should_send_confirmation_email_when_payment_succeeds
```

This convention produces a test suite whose method names read as a behavioral specification of the class. Running `mvn test -pl my-service` and reading the output tells you exactly what the class does and what it guarantees.

### Test Data Builders

When tests require complex objects, constructing them inline bloats the test and buries the relevant fields in noise. Test data builders (using the builder pattern) allow tests to express only the fields relevant to the scenario.

```java
// Test data builder
public class OrderBuilder {
    private String id = "ORD-001";
    private String userId = "USR-001";
    private String userEmail = "customer@example.com";
    private BigDecimal total = new BigDecimal("50.00");
    private List<OrderItem> items = new ArrayList<>();
    private PaymentMethod paymentMethod = PaymentMethod.CREDIT_CARD;

    public static OrderBuilder anOrder() {
        return new OrderBuilder();
    }

    public OrderBuilder withId(String id) {
        this.id = id;
        return this;
    }

    public OrderBuilder withTotal(BigDecimal total) {
        this.total = total;
        return this;
    }

    public OrderBuilder withUserId(String userId) {
        this.userId = userId;
        return this;
    }

    public OrderBuilder withItem(OrderItem item) {
        this.items.add(item);
        return this;
    }

    public Order build() {
        return new Order(id, userId, userEmail, total, items, paymentMethod);
    }
}

// Test using the builder: only relevant field is expressed
@Test
void should_apply_discount_when_order_total_exceeds_100() {
    Order order = OrderBuilder.anOrder()
        .withTotal(new BigDecimal("150.00"))
        .build();

    BigDecimal discount = calculator.calculate(order);

    assertEquals(new BigDecimal("15.00"), discount);
}
```

The test reads clearly. The irrelevant details (userId, email, paymentMethod) are handled by sensible defaults in the builder.

### Full Example: PaymentService Unit Tests (FIRST + AAA + Builders)

```java
// PaymentService.java
public class PaymentService {
    private final PaymentGateway paymentGateway;
    private final PaymentRepository paymentRepository;
    private final CurrencyConverter currencyConverter;
    private final Clock clock;

    public PaymentService(
            PaymentGateway paymentGateway,
            PaymentRepository paymentRepository,
            CurrencyConverter currencyConverter,
            Clock clock) {
        this.paymentGateway = paymentGateway;
        this.paymentRepository = paymentRepository;
        this.currencyConverter = currencyConverter;
        this.clock = clock;
    }

    public PaymentResult processPayment(PaymentRequest request) {
        if (paymentRepository.existsByIdempotencyKey(request.getIdempotencyKey())) {
            return PaymentResult.duplicate(request.getIdempotencyKey());
        }

        Money chargeAmount = request.getCurrency().equals("USD")
            ? request.getAmount()
            : currencyConverter.convert(request.getAmount(), request.getCurrency(), "USD");

        try {
            GatewayResponse response = paymentGateway.charge(chargeAmount, request.getPaymentMethod());
            Payment payment = Payment.builder()
                .idempotencyKey(request.getIdempotencyKey())
                .amount(chargeAmount)
                .status(PaymentStatus.COMPLETED)
                .processedAt(clock.instant())
                .build();
            paymentRepository.save(payment);
            return PaymentResult.success(response.getTransactionId());
        } catch (InsufficientFundsException e) {
            return PaymentResult.failure("INSUFFICIENT_FUNDS", e.getMessage());
        } catch (GatewayTimeoutException e) {
            return PaymentResult.failure("GATEWAY_TIMEOUT", e.getMessage());
        }
    }
}
```

---

## 7. Mocking and Stubbing with Mockito

### Mock vs Stub vs Spy

**Stub**: A test double that returns pre-programmed responses. It does not verify how it was called. Use it when the test needs a dependency to return a specific value.

**Mock**: A test double that both returns pre-programmed responses and verifies that it was called in the expected way. Use it when the test needs to verify that a side-effecting collaborator was called correctly.

**Spy**: A wrapper around a real object that delegates method calls to the real implementation by default, but allows you to override specific methods and verify calls. Use sparingly — usually a sign that the design could be cleaner.

| Double | Returns Canned Responses | Verifies Calls | Uses Real Logic |
|--------|--------------------------|----------------|-----------------|
| Stub   | Yes                      | No             | No              |
| Mock   | Yes                      | Yes            | No              |
| Spy    | Yes (or delegates)       | Yes            | Yes (by default)|

In Mockito, both mocks and stubs are created with `@Mock` or `mock()`. The distinction is in how you use them: `when(...).thenReturn(...)` for stubbing, `verify(...)` for mock verification.

### Setting Up Mocks with @Mock and @InjectMocks

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private PaymentGateway paymentGateway;

    @Mock
    private InventoryService inventoryService;

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderService orderService;

    // @InjectMocks creates an OrderService and injects the @Mock fields
    // It tries constructor injection first, then setter injection, then field injection
    // Prefer constructor injection so @InjectMocks works reliably
}
```

### Argument Matchers

Mockito provides argument matchers for flexible stubbing:

```java
// Exact value matching
when(paymentGateway.charge(new Money("100.00", "USD"), any())).thenReturn(success);

// Type matching
when(paymentGateway.charge(any(Money.class), any(PaymentMethod.class))).thenReturn(success);

// Custom predicate
when(paymentGateway.charge(
    argThat(money -> money.getAmount().compareTo(BigDecimal.ZERO) > 0),
    any()
)).thenReturn(success);

// Any non-null value
when(orderRepository.findById(anyString())).thenReturn(Optional.of(order));
```

Important: if you use an argument matcher for one argument, you must use matchers for all arguments in that stub.

### Verifying Interactions

```java
// Verify a method was called exactly once with specific arguments
verify(emailService).send("user@example.com", "Order Confirmed", any());

// Verify a method was called a specific number of times
verify(auditLogger, times(1)).logOrderProcessed(eq("ORD-001"), any(Instant.class));

// Verify a method was never called
verify(emailService, never()).send(any(), eq("Order Failed"), any());

// Verify no more interactions occurred on a mock
verifyNoMoreInteractions(paymentGateway);
```

### Capturing Arguments with ArgumentCaptor

When you need to verify not just that a method was called, but what it was called *with* — and the argument is complex — use `ArgumentCaptor`:

```java
@Test
void should_save_payment_with_correct_details_on_success() {
    // Arrange
    when(paymentGateway.charge(any(), any())).thenReturn(GatewayResponse.success("TXN-123"));

    // Act
    paymentService.processPayment(request);

    // Assert: capture the Payment object passed to save()
    ArgumentCaptor<Payment> paymentCaptor = ArgumentCaptor.forClass(Payment.class);
    verify(paymentRepository).save(paymentCaptor.capture());

    Payment savedPayment = paymentCaptor.getValue();
    assertEquals(PaymentStatus.COMPLETED, savedPayment.getStatus());
    assertEquals("idempotency-key-001", savedPayment.getIdempotencyKey());
    assertNotNull(savedPayment.getProcessedAt());
}
```

### Full Mockito Example: OrderService

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock PaymentGateway paymentGateway;
    @Mock InventoryService inventoryService;
    @Mock OrderRepository orderRepository;
    @Mock EmailService emailService;

    @InjectMocks OrderService orderService;

    private Order testOrder;

    @BeforeEach
    void setUp() {
        testOrder = OrderBuilder.anOrder()
            .withId("ORD-001")
            .withTotal(new BigDecimal("75.00"))
            .withItem(new OrderItem("ITEM-A", 2))
            .build();
    }

    @Test
    void should_charge_payment_and_confirm_order_when_stock_available() {
        // Arrange
        when(inventoryService.isAvailable(testOrder.getItems())).thenReturn(true);
        when(paymentGateway.charge(any(), any()))
            .thenReturn(GatewayResponse.success("TXN-456"));

        // Act
        OrderResult result = orderService.placeOrder(testOrder);

        // Assert
        assertTrue(result.isSuccess());
        verify(paymentGateway).charge(
            eq(new Money(new BigDecimal("75.00"), "USD")),
            eq(testOrder.getPaymentMethod())
        );
        verify(inventoryService).deduct(testOrder.getItems());
        verify(orderRepository).save(testOrder);
        verify(emailService).send(
            eq(testOrder.getUserEmail()),
            eq("Order Confirmed"),
            anyString()
        );
    }

    @Test
    void should_reject_order_and_not_charge_when_stock_unavailable() {
        // Arrange
        when(inventoryService.isAvailable(testOrder.getItems())).thenReturn(false);

        // Act
        OrderResult result = orderService.placeOrder(testOrder);

        // Assert
        assertFalse(result.isSuccess());
        assertEquals("OUT_OF_STOCK", result.getFailureCode());
        verify(paymentGateway, never()).charge(any(), any());
        verify(orderRepository, never()).save(any());
        verify(emailService, never()).send(any(), any(), any());
    }

    @Test
    void should_return_failure_and_not_deduct_inventory_when_payment_fails() {
        // Arrange
        when(inventoryService.isAvailable(testOrder.getItems())).thenReturn(true);
        when(paymentGateway.charge(any(), any()))
            .thenThrow(new InsufficientFundsException("Balance too low"));

        // Act
        OrderResult result = orderService.placeOrder(testOrder);

        // Assert
        assertFalse(result.isSuccess());
        assertEquals("INSUFFICIENT_FUNDS", result.getFailureCode());
        verify(inventoryService, never()).deduct(any());
    }

    @Test
    void should_capture_correct_order_saved_to_repository() {
        // Arrange
        when(inventoryService.isAvailable(any())).thenReturn(true);
        when(paymentGateway.charge(any(), any())).thenReturn(GatewayResponse.success("TXN-789"));

        // Act
        orderService.placeOrder(testOrder);

        // Assert: verify the saved order state
        ArgumentCaptor<Order> orderCaptor = ArgumentCaptor.forClass(Order.class);
        verify(orderRepository).save(orderCaptor.capture());
        Order savedOrder = orderCaptor.getValue();

        assertEquals(OrderStatus.CONFIRMED, savedOrder.getStatus());
        assertEquals("TXN-789", savedOrder.getTransactionId());
    }
}
```

### When to Mock vs When to Use Real Implementations

Not every dependency needs to be mocked. The guideline:

- **Mock** when the dependency is external (database, HTTP, message queue, email, file system), non-deterministic (time, random), or when you need to control its behavior for a specific test scenario.
- **Use real implementations** when the dependency is a pure function or value object that adds no testing complexity. Mocking `BigDecimal` arithmetic or a simple `DiscountCalculator` with no dependencies is unnecessary and obscures what the test is actually asserting.

### The Over-Mocking Anti-Pattern

Over-mocking is testing the implementation rather than the behavior. It produces tests that break when you refactor, even when behavior is unchanged.

```java
// Over-mocked test: testing implementation details
@Test
void should_process_order() {
    orderService.placeOrder(order);

    // These verifications are testing HOW it works, not WHAT it does
    verify(validator).validateOrderItems(order.getItems());
    verify(pricingEngine).computeTotal(order.getItems());
    verify(discountApplier).applyDiscounts(order);
    verify(taxCalculator).computeTax(order);
    // ... every internal method call is verified
}
```

If you refactor `OrderService` to combine `pricingEngine` and `discountApplier` into a single `PricingService`, this test breaks — even though the behavior is identical. The test is coupled to the implementation, not the contract.

**Better: verify observable outcomes.**

```java
@Test
void should_process_order() {
    GatewayResponse gatewayResponse = GatewayResponse.success("TXN-123");
    when(paymentGateway.charge(any(), any())).thenReturn(gatewayResponse);

    OrderResult result = orderService.placeOrder(order);

    // Assert on observable outcome: was the order confirmed?
    assertTrue(result.isSuccess());
    assertEquals("TXN-123", result.getTransactionId());
    verify(orderRepository).save(argThat(o -> o.getStatus() == OrderStatus.CONFIRMED));
}
```

---

## 8. Integration Testing Strategy

### What Integration Tests Should Verify

Integration tests fill the gap between unit tests (which test classes in isolation) and E2E tests (which test the full stack). They verify:

- That your SQL queries produce the correct results against a real schema.
- That your ORM mappings are correct (field names, types, relationships).
- That your repository layer correctly persists and retrieves domain objects.
- That your REST controllers serialize/deserialize requests and responses correctly.
- That your message producer and consumer correctly serialize and route messages.

Unit tests cannot catch these because they mock all external systems. E2E tests catch them too late and too slowly.

### Testing with Real Databases

**H2 In-Memory Database**: H2 is a Java in-memory database that starts in milliseconds and runs SQL queries with reasonable MySQL/PostgreSQL compatibility (via compatibility mode). It is appropriate for tests that verify basic repository behavior and simple queries.

Limitations: H2 does not support every database-specific feature. Complex queries, stored procedures, or database-specific SQL functions may behave differently. For complex query logic, use Testcontainers.

```java
// application-test.yml (Spring Boot)
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;MODE=MySQL
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop

// Repository integration test
@DataJpaTest
@ActiveProfiles("test")
class OrderRepositoryIntegrationTest {

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void should_persist_and_retrieve_order_with_items() {
        // Arrange
        Order order = OrderBuilder.anOrder()
            .withId("ORD-001")
            .withItem(new OrderItem("ITEM-A", 2))
            .build();

        // Act
        orderRepository.save(order);
        Order retrieved = orderRepository.findById("ORD-001").orElseThrow();

        // Assert
        assertEquals("ORD-001", retrieved.getId());
        assertEquals(1, retrieved.getItems().size());
        assertEquals("ITEM-A", retrieved.getItems().get(0).getItemId());
    }

    @Test
    void should_find_orders_placed_after_given_date() {
        Instant cutoff = Instant.parse("2025-01-01T00:00:00Z");
        orderRepository.save(buildOrderAt(Instant.parse("2024-12-31T00:00:00Z")));
        orderRepository.save(buildOrderAt(Instant.parse("2025-01-02T00:00:00Z")));

        List<Order> orders = orderRepository.findOrdersPlacedAfter(cutoff);

        assertEquals(1, orders.size());
    }
}
```

**Testcontainers**: Testcontainers starts real Docker containers (MySQL, PostgreSQL, Kafka, Redis) as part of the test lifecycle. It provides the highest fidelity — your tests run against exactly the same database version as production.

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryContainerTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("orders_test")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureDataSource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void should_execute_complex_aggregation_query() {
        // Test queries that use MySQL-specific features
    }
}
```

### Testing with Real Message Queues

For Kafka-based systems, Testcontainers provides a Kafka container. An embedded Kafka broker (via `spring-kafka-test`) is faster but less representative.

```java
@SpringBootTest
@Testcontainers
class OrderEventPublisherTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

    @DynamicPropertySource
    static void kafkaProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Autowired
    private OrderEventPublisher publisher;

    @Test
    void should_publish_order_confirmed_event_to_kafka() throws Exception {
        publisher.publishOrderConfirmed("ORD-001");

        // Consume from Kafka and assert
        ConsumerRecords<String, String> records = pollKafka("order-events", Duration.ofSeconds(5));
        assertThat(records).hasSize(1);
        assertThat(records.iterator().next().value()).contains("ORD-001");
    }
}
```

### Slice Tests

Spring Boot provides test slices that load only the parts of the application context relevant to the component under test:

- `@DataJpaTest`: loads only JPA repositories, entity classes, and the data source. Does not load controllers, services, or other beans. Fast and focused.
- `@WebMvcTest`: loads only the Spring MVC layer (controllers, filters, interceptors). Does not load the JPA layer. Service and repository dependencies must be mocked.
- `@RestClientTest`: loads only components needed to test REST clients.

These are faster than `@SpringBootTest` because they load a subset of the context.

### Test Data Management

Integration tests that share a database must handle isolation. Options:

- **`@Transactional` on tests**: Spring rolls back the transaction after each test. Fast and clean, but means each test cannot verify commit behavior.
- **`@Sql` scripts**: run SQL before and after each test to insert and delete data explicitly. More control, slightly more verbose.
- **Testcontainers with per-test containers**: extreme isolation, but slow because containers start per test.
- **Truncate tables in `@BeforeEach`**: direct and simple, but requires knowing all table names.

For most cases, `@Transactional` on the test class is sufficient. If you need to verify commit behavior (e.g., testing that a `@Transactional` service method commits), use explicit truncation or a fresh container.

---

## 9. Contract Testing

### Why Integration Tests Do Not Catch Consumer-Breaking Changes

In a microservices architecture, Service A (consumer) calls Service B (provider). Integration tests for Service B verify that Service B works correctly in isolation. But they do not verify that Service B's API still satisfies the assumptions that Service A has about it.

A developer changes Service B's response schema — removing a field that Service A depends on. Service B's integration tests all pass (they test Service B's behavior, not its contract with A). Service A's tests all pass (they mock Service B). Both services deploy. Production breaks.

### Consumer-Driven Contract Testing

Contract testing addresses this by encoding the consumer's assumptions about the provider's API as a formal contract, and then verifying that the provider satisfies that contract — independently of the consumer.

The flow:

1. Service A (consumer) defines a contract: "When I call `GET /payments/{id}`, I expect a response with these fields."
2. The contract is published to a shared registry (the Pact Broker).
3. Service B (provider) runs contract verification: it replays the consumer's expected interactions and verifies that its real implementation matches.
4. If Service B makes a breaking change, the contract verification fails before deployment.

### Pact Framework Overview

Pact is the most widely used consumer-driven contract testing framework. It has libraries for Java, Node.js, Go, Python, and others.

**Consumer side** (Service A): defines the expected interactions using a Pact DSL.

**Provider side** (Service B): runs the Pact verifier against its real implementation.

The contract (pact file) is a JSON document that describes the expected request and the minimum acceptable response.

### Full Java Example: Payment Service Contract

**Consumer test (Order Service defining its contract with Payment Service):**

```java
// In the Order Service codebase
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "PaymentService")
class PaymentServiceContractTest {

    @Pact(consumer = "OrderService")
    public RequestResponsePact createPaymentPact(PactDslWithProvider builder) {
        return builder
            .given("a payment can be processed")
            .uponReceiving("a request to create a payment")
                .method("POST")
                .path("/payments")
                .headers(Map.of("Content-Type", "application/json"))
                .body(new PactDslJsonBody()
                    .stringType("orderId", "ORD-001")
                    .decimalType("amount", 75.00)
                    .stringType("currency", "USD")
                    .stringType("paymentMethod", "CREDIT_CARD")
                )
            .willRespondWith()
                .status(201)
                .headers(Map.of("Content-Type", "application/json"))
                .body(new PactDslJsonBody()
                    .stringType("paymentId")         // any string, shape matters
                    .stringType("status", "COMPLETED")
                    .stringType("transactionId")     // any string
                    .datetime("processedAt", "yyyy-MM-dd'T'HH:mm:ssZ")
                )
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "createPaymentPact")
    void should_create_payment_successfully(MockServer mockServer) {
        // The mock server implements the contract above
        PaymentServiceClient client = new PaymentServiceClient(mockServer.getUrl());

        PaymentResponse response = client.createPayment(
            new PaymentRequest("ORD-001", new BigDecimal("75.00"), "USD", "CREDIT_CARD")
        );

        assertEquals("COMPLETED", response.getStatus());
        assertNotNull(response.getPaymentId());
        assertNotNull(response.getTransactionId());
    }
}
```

**Provider verification (Payment Service verifying the contract):**

```java
// In the Payment Service codebase
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Provider("PaymentService")
@PactBroker(url = "https://pact-broker.mycompany.com")
class PaymentServicePactVerificationTest {

    @LocalServerPort
    private int port;

    @BeforeEach
    void setUp(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void pactVerificationTestTemplate(PactVerificationContext context) {
        context.verifyInteraction();
    }

    @State("a payment can be processed")
    void setupPaymentCanBeProcessed() {
        // Set up any state the provider needs to satisfy this interaction
        // e.g., ensure the payment gateway mock is configured to accept payments
    }
}
```

When the Payment Service runs this test, it fetches the contracts from the Pact Broker and replays each consumer's expected interactions against the real running service. If the service returns a different schema or status code, the verification fails and the build is blocked.

---

## 10. Full TDD Example: Rate Limiter (Complete Walkthrough)

The previous TDD walkthrough introduced the rate limiter incrementally. Here is the complete, final test class alongside the complete implementation:

### Complete Test Class

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;
import java.time.Duration;
import java.time.Instant;
import java.time.ZoneOffset;

class RateLimiterTest {

    private static final Instant BASE_TIME = Instant.parse("2025-01-01T00:00:00Z");

    @Test
    void should_allow_request_when_bucket_has_tokens() {
        MutableClock clock = new MutableClock(BASE_TIME);
        RateLimiter limiter = new RateLimiter(10, 1.0, clock);

        assertTrue(limiter.tryAcquire());
    }

    @Test
    void should_reject_request_when_bucket_is_empty() {
        MutableClock clock = new MutableClock(BASE_TIME);
        RateLimiter limiter = new RateLimiter(2, 1.0, clock);

        limiter.tryAcquire();
        limiter.tryAcquire();
        boolean result = limiter.tryAcquire();

        assertFalse(result);
    }

    @Test
    void should_allow_exactly_capacity_requests_before_rejecting() {
        MutableClock clock = new MutableClock(BASE_TIME);
        RateLimiter limiter = new RateLimiter(5, 1.0, clock);

        for (int i = 0; i < 5; i++) {
            assertTrue(limiter.tryAcquire(), "Request " + (i + 1) + " should be allowed");
        }
        assertFalse(limiter.tryAcquire(), "Request 6 should be rejected");
    }

    @Test
    void should_allow_request_after_tokens_refill_over_time() {
        MutableClock clock = new MutableClock(BASE_TIME);
        RateLimiter limiter = new RateLimiter(1, 1.0, clock); // 1 token/sec

        limiter.tryAcquire(); // drain the single token

        clock.advance(Duration.ofSeconds(1)); // 1 second passes

        assertTrue(limiter.tryAcquire()); // refilled
    }

    @Test
    void should_not_exceed_capacity_when_long_time_passes_without_requests() {
        MutableClock clock = new MutableClock(BASE_TIME);
        RateLimiter limiter = new RateLimiter(5, 10.0, clock); // 10 tokens/sec, cap 5

        // drain all tokens
        for (int i = 0; i < 5; i++) limiter.tryAcquire();

        clock.advance(Duration.ofSeconds(100)); // 1000 tokens would accumulate

        // Should still only have 5 (capacity)
        for (int i = 0; i < 5; i++) {
            assertTrue(limiter.tryAcquire(), "Request " + (i + 1) + " should be allowed");
        }
        assertFalse(limiter.tryAcquire(), "Should not exceed capacity");
    }

    @Test
    void should_accumulate_partial_tokens_between_requests() {
        MutableClock clock = new MutableClock(BASE_TIME);
        RateLimiter limiter = new RateLimiter(10, 2.0, clock); // 2 tokens/sec

        // drain all 10 tokens
        for (int i = 0; i < 10; i++) limiter.tryAcquire();

        // 0.4 seconds = 0.8 tokens accumulated — not enough
        clock.advance(Duration.ofMillis(400));
        assertFalse(limiter.tryAcquire(), "0.8 tokens accumulated — not enough for a request");

        // another 0.1 seconds = total 0.5s = 1.0 tokens — enough
        clock.advance(Duration.ofMillis(100));
        assertTrue(limiter.tryAcquire(), "1.0 token accumulated — should be allowed");
    }

    @Test
    void should_reject_immediately_if_constructed_with_zero_initial_tokens() {
        MutableClock clock = new MutableClock(BASE_TIME);
        // A rate limiter that starts empty (custom constructor)
        RateLimiter limiter = new RateLimiter(5, 1.0, 0.0, clock);

        assertFalse(limiter.tryAcquire());
    }

    @Test
    void should_throw_when_capacity_is_zero_or_negative() {
        MutableClock clock = new MutableClock(BASE_TIME);

        assertThrows(IllegalArgumentException.class,
            () -> new RateLimiter(0, 1.0, clock));
        assertThrows(IllegalArgumentException.class,
            () -> new RateLimiter(-1, 1.0, clock));
    }

    @Test
    void should_throw_when_refill_rate_is_zero_or_negative() {
        MutableClock clock = new MutableClock(BASE_TIME);

        assertThrows(IllegalArgumentException.class,
            () -> new RateLimiter(10, 0.0, clock));
        assertThrows(IllegalArgumentException.class,
            () -> new RateLimiter(10, -1.0, clock));
    }
}
```

### Complete Implementation

```java
import java.time.Clock;
import java.time.Duration;
import java.time.Instant;

public class RateLimiter {
    private final int capacity;
    private final double refillRatePerSecond;
    private final Clock clock;

    private double availableTokens;
    private Instant lastRefillTimestamp;

    public RateLimiter(int capacity, double refillRatePerSecond, Clock clock) {
        this(capacity, refillRatePerSecond, capacity, clock);
    }

    public RateLimiter(int capacity, double refillRatePerSecond, double initialTokens, Clock clock) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("Capacity must be a positive integer, got: " + capacity);
        }
        if (refillRatePerSecond <= 0) {
            throw new IllegalArgumentException("Refill rate must be positive, got: " + refillRatePerSecond);
        }
        if (initialTokens < 0 || initialTokens > capacity) {
            throw new IllegalArgumentException("Initial tokens must be between 0 and capacity");
        }

        this.capacity = capacity;
        this.refillRatePerSecond = refillRatePerSecond;
        this.clock = clock;
        this.availableTokens = initialTokens;
        this.lastRefillTimestamp = clock.instant();
    }

    public synchronized boolean tryAcquire() {
        refillTokens();
        if (availableTokens >= 1.0) {
            availableTokens -= 1.0;
            return true;
        }
        return false;
    }

    private void refillTokens() {
        Instant now = clock.instant();
        double secondsElapsed = Duration.between(lastRefillTimestamp, now).toNanos() / 1_000_000_000.0;
        double tokensToAdd = secondsElapsed * refillRatePerSecond;
        availableTokens = Math.min(capacity, availableTokens + tokensToAdd);
        lastRefillTimestamp = now;
    }
}
```

### MutableClock Test Helper

```java
import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.time.ZoneId;
import java.time.ZoneOffset;

class MutableClock extends Clock {
    private Instant now;

    MutableClock(Instant start) {
        this.now = start;
    }

    public void advance(Duration duration) {
        this.now = now.plus(duration);
    }

    @Override
    public ZoneId getZone() {
        return ZoneOffset.UTC;
    }

    @Override
    public Clock withZone(ZoneId zone) {
        return this;
    }

    @Override
    public Instant instant() {
        return now;
    }
}
```

---

## 11. Full Unit Test Example: Payment Service

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.time.Clock;
import java.time.Instant;
import java.time.ZoneOffset;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @Mock private PaymentGateway paymentGateway;
    @Mock private PaymentRepository paymentRepository;
    @Mock private CurrencyConverter currencyConverter;

    private PaymentService paymentService;
    private final Clock fixedClock = Clock.fixed(
        Instant.parse("2025-06-01T10:00:00Z"),
        ZoneOffset.UTC
    );

    @BeforeEach
    void setUp() {
        paymentService = new PaymentService(
            paymentGateway,
            paymentRepository,
            currencyConverter,
            fixedClock
        );
    }

    // --- Successful Payment ---

    @Test
    void should_return_success_result_with_transaction_id_when_payment_succeeds() {
        // Arrange
        PaymentRequest request = buildUsdPaymentRequest("100.00", "idem-001");
        when(paymentRepository.existsByIdempotencyKey("idem-001")).thenReturn(false);
        when(paymentGateway.charge(any(), any()))
            .thenReturn(GatewayResponse.success("TXN-ABCDE"));

        // Act
        PaymentResult result = paymentService.processPayment(request);

        // Assert
        assertTrue(result.isSuccess());
        assertEquals("TXN-ABCDE", result.getTransactionId());
    }

    @Test
    void should_save_completed_payment_record_when_payment_succeeds() {
        // Arrange
        PaymentRequest request = buildUsdPaymentRequest("100.00", "idem-001");
        when(paymentRepository.existsByIdempotencyKey(any())).thenReturn(false);
        when(paymentGateway.charge(any(), any())).thenReturn(GatewayResponse.success("TXN-001"));

        // Act
        paymentService.processPayment(request);

        // Assert: verify the saved payment record
        ArgumentCaptor<Payment> captor = ArgumentCaptor.forClass(Payment.class);
        verify(paymentRepository).save(captor.capture());

        Payment saved = captor.getValue();
        assertEquals(PaymentStatus.COMPLETED, saved.getStatus());
        assertEquals("idem-001", saved.getIdempotencyKey());
        assertEquals(Instant.parse("2025-06-01T10:00:00Z"), saved.getProcessedAt());
    }

    // --- Insufficient Funds ---

    @Test
    void should_return_failure_with_insufficient_funds_code_when_gateway_rejects_payment() {
        // Arrange
        PaymentRequest request = buildUsdPaymentRequest("500.00", "idem-002");
        when(paymentRepository.existsByIdempotencyKey(any())).thenReturn(false);
        when(paymentGateway.charge(any(), any()))
            .thenThrow(new InsufficientFundsException("Card declined"));

        // Act
        PaymentResult result = paymentService.processPayment(request);

        // Assert
        assertFalse(result.isSuccess());
        assertEquals("INSUFFICIENT_FUNDS", result.getFailureCode());
    }

    @Test
    void should_not_save_payment_record_when_insufficient_funds() {
        // Arrange
        PaymentRequest request = buildUsdPaymentRequest("500.00", "idem-002");
        when(paymentRepository.existsByIdempotencyKey(any())).thenReturn(false);
        when(paymentGateway.charge(any(), any()))
            .thenThrow(new InsufficientFundsException("Card declined"));

        // Act
        paymentService.processPayment(request);

        // Assert
        verify(paymentRepository, never()).save(any());
    }

    // --- Gateway Timeout ---

    @Test
    void should_return_failure_with_gateway_timeout_code_when_gateway_times_out() {
        // Arrange
        PaymentRequest request = buildUsdPaymentRequest("100.00", "idem-003");
        when(paymentRepository.existsByIdempotencyKey(any())).thenReturn(false);
        when(paymentGateway.charge(any(), any()))
            .thenThrow(new GatewayTimeoutException("Request timed out after 30s"));

        // Act
        PaymentResult result = paymentService.processPayment(request);

        // Assert
        assertFalse(result.isSuccess());
        assertEquals("GATEWAY_TIMEOUT", result.getFailureCode());
    }

    // --- Duplicate Payment Prevention ---

    @Test
    void should_return_duplicate_result_without_charging_when_idempotency_key_already_exists() {
        // Arrange
        PaymentRequest request = buildUsdPaymentRequest("100.00", "idem-004");
        when(paymentRepository.existsByIdempotencyKey("idem-004")).thenReturn(true);

        // Act
        PaymentResult result = paymentService.processPayment(request);

        // Assert
        assertTrue(result.isDuplicate());
        verify(paymentGateway, never()).charge(any(), any());
    }

    @Test
    void should_not_save_payment_record_when_request_is_duplicate() {
        // Arrange
        PaymentRequest request = buildUsdPaymentRequest("100.00", "idem-004");
        when(paymentRepository.existsByIdempotencyKey("idem-004")).thenReturn(true);

        // Act
        paymentService.processPayment(request);

        // Assert
        verify(paymentRepository, never()).save(any());
    }

    // --- Currency Conversion ---

    @Test
    void should_convert_currency_to_usd_before_charging_when_request_currency_is_not_usd() {
        // Arrange
        PaymentRequest request = buildPaymentRequest("85.00", "EUR", "idem-005");
        Money usdAmount = new Money(new BigDecimal("93.50"), "USD");

        when(paymentRepository.existsByIdempotencyKey(any())).thenReturn(false);
        when(currencyConverter.convert(
            eq(new Money(new BigDecimal("85.00"), "EUR")),
            eq("EUR"),
            eq("USD")
        )).thenReturn(usdAmount);
        when(paymentGateway.charge(eq(usdAmount), any()))
            .thenReturn(GatewayResponse.success("TXN-EUR-001"));

        // Act
        PaymentResult result = paymentService.processPayment(request);

        // Assert
        assertTrue(result.isSuccess());
        verify(currencyConverter).convert(any(), eq("EUR"), eq("USD"));
        verify(paymentGateway).charge(eq(usdAmount), any());
    }

    @Test
    void should_not_invoke_currency_converter_when_currency_is_already_usd() {
        // Arrange
        PaymentRequest request = buildUsdPaymentRequest("100.00", "idem-006");
        when(paymentRepository.existsByIdempotencyKey(any())).thenReturn(false);
        when(paymentGateway.charge(any(), any())).thenReturn(GatewayResponse.success("TXN-001"));

        // Act
        paymentService.processPayment(request);

        // Assert
        verifyNoInteractions(currencyConverter);
    }

    // --- Test Data Helpers ---

    private PaymentRequest buildUsdPaymentRequest(String amount, String idempotencyKey) {
        return buildPaymentRequest(amount, "USD", idempotencyKey);
    }

    private PaymentRequest buildPaymentRequest(String amount, String currency, String idempotencyKey) {
        return PaymentRequest.builder()
            .amount(new Money(new BigDecimal(amount), currency))
            .currency(currency)
            .paymentMethod(PaymentMethod.CREDIT_CARD)
            .idempotencyKey(idempotencyKey)
            .userId("USR-001")
            .build();
    }
}
```

---

## 12. Senior vs Mid-Level Thinking

| Decision Area | Mid-Level Thinking | Senior Thinking |
|---|---|---|
| When to write tests | After the code is working, to prove it works | Before the code exists, to drive the design |
| What to mock | Everything except the class under test | Only what is necessary — real implementations where they add clarity |
| Test granularity | One large test per method | Multiple small tests per behavior, each testing one scenario |
| Test naming | `testProcessOrder()`, `test1()` | `should_reject_order_when_inventory_is_insufficient()` — reads as a spec |
| When a test is hard to write | Accepts the difficulty, writes a complex test | Treats difficulty as a design signal — refactors the production code first |
| Coverage | Aims for 80% line coverage | Aims for 100% behavior coverage — every meaningful path has a test, line percentage is a side effect |
| Integration vs unit | Writes integration tests for everything because "that's how you know it really works" | Heavy investment in fast unit tests, targeted integration tests for persistence and serialization |
| Flaky tests | Reruns them, adds a retry, marks as known flaky | Treats every flaky test as a high-priority bug — non-determinism in tests reveals non-determinism in code |
| Test data | Constructs objects inline with all fields | Uses test data builders so tests express only what is relevant |
| Assertions | `assertNotNull(result)`, `assertTrue(result.isSuccess())` | Specific: `assertEquals("TXN-123", result.getTransactionId())`, captures and asserts on complex objects |
| Time in tests | `Thread.sleep(1000)` to wait for async operations | Injects a controllable clock; eliminates sleep from tests |
| Test doubles | Uses `@Mock` for everything without thinking | Distinguishes mocks (verify calls) from stubs (return values) — uses the right one |
| TDD | Writes tests after implementation when asked | Defaults to TDD for complex logic; knows when to spike without tests |
| Over-mocking | Mocks every collaborator and verifies every call | Verifies only observable outcomes — changes to internal collaborator structure should not break tests |
| Slow tests | Accepts slow tests as the cost of confidence | Investigates every slow unit test — finds and removes the hidden external dependency |
| Contract testing | Writes integration tests against a shared test environment | Implements consumer-driven contract tests so breaking changes are caught at the provider before deployment |

---

## 13. Testing Anti-Patterns

### Anti-Pattern 1: Tests That Test Nothing

A test that asserts on a trivial property — that a non-null object is not null, that a method returns without throwing — provides no value and creates false confidence in coverage metrics.

```java
// Useless test
@Test
void testCreateOrder() {
    Order order = new Order("ORD-001", "USR-001");
    assertNotNull(order); // This will never fail
}
```

Every test must assert that a specific, meaningful behavior occurred. If removing the assertion would not change whether the test passes or fails, the test is not testing anything.

### Anti-Pattern 2: Brittle Tests That Test Implementation Details

A brittle test fails when the implementation changes but the behavior does not. This creates maintenance burden and trains engineers to distrust test failures.

```java
// Brittle: tests the exact sequence of internal method calls
@Test
void should_process_order() {
    service.process(order);
    InOrder inOrder = inOrder(validator, pricer, gateway, repo, emailer);
    inOrder.verify(validator).validate(order);
    inOrder.verify(pricer).computePrice(order);
    inOrder.verify(gateway).charge(order.getTotal());
    inOrder.verify(repo).save(order);
    inOrder.verify(emailer).sendConfirmation(order);
}
```

If you refactor `process()` to combine pricing and validation into a single step, this test fails — even though the behavior is identical. Test the outcome, not the sequence of internal method calls.

### Anti-Pattern 3: Slow Unit Tests

A unit test that takes more than a few hundred milliseconds is not a unit test. It has a hidden dependency on something external: a database, a file system, a real HTTP server, or `Thread.sleep`.

Slow unit tests destroy the feedback loop. When running all tests takes 10 minutes, developers stop running them locally. The test suite becomes a CI gate rather than a design tool.

Find the external dependency. Inject it. Replace it with a fast in-memory alternative in tests.

### Anti-Pattern 4: Test Pollution (Tests Affecting Each Other)

Test pollution occurs when one test modifies shared state that affects subsequent tests. The suite becomes non-deterministic: tests pass in one order and fail in another.

Common causes:
- Mutable static fields that are not reset between tests
- Singleton instances with mutable state (the Singleton anti-pattern)
- Database records inserted in one test and visible in the next (missing transaction rollback)
- Thread-local state not cleaned up

The fix: ensure each test creates its own instances and operates on its own state. Use `@BeforeEach` to reset any shared state. Use `@Transactional` or table truncation for database tests.

### Anti-Pattern 5: Testing the Mock, Not the Code

This occurs when a test stubs a dependency to return a value and then asserts that it returned that value. The test does not exercise any production code at all.

```java
// Testing the mock, not the code
@Test
void should_return_user() {
    User expectedUser = new User("USR-001");
    when(userRepository.findById("USR-001")).thenReturn(Optional.of(expectedUser));

    User result = userRepository.findById("USR-001").orElseThrow(); // calling the mock directly!

    assertEquals(expectedUser, result);
}
```

This test never calls any production code. It only verifies that Mockito works. The system under test is missing.

### Anti-Pattern 6: Over-Reliance on E2E Tests

End-to-end tests are expensive. They are slow to run, difficult to debug, and sensitive to environment state. A suite dominated by E2E tests produces a slow CI pipeline, flaky failures, and poor failure localization.

The right role for E2E tests is to verify critical user journeys at the system level — things like "a user can place an order and receive a confirmation email." They should not be used to verify discount calculation logic, rate limiting behavior, or currency conversion. Those belong in fast unit tests.

When a bug is caught by an E2E test, the correct response is not to add another E2E test. It is to ask why the unit tests did not catch it, and add coverage at the appropriate level.

---

## Summary

Testability is not a separate concern from software design. It is the same concern expressed differently. The structural properties that make code testable — small classes, injected dependencies, explicit interfaces, pure functions, no global state — are the same properties that make code maintainable, evolvable, and correct.

Senior engineers internalize the following:

- If a test is hard to write, the design is the problem. Fix the design.
- The test pyramid exists for a reason. Invest heavily in unit tests. Use integration tests for persistence and serialization. Use E2E tests sparingly for critical user journeys.
- TDD is a design tool, not just a testing technique. The pressure of writing the test first reveals bad designs before they are committed to.
- Mocks exist to control and observe collaborators. Over-mocking couples tests to implementations, not contracts.
- Contract testing is the only reliable way to catch consumer-breaking API changes in a microservices architecture.
- Every flaky test is a bug. Every slow unit test is a hidden external dependency. Neither should be tolerated.
