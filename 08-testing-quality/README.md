# Module 08 — Testing and Quality Engineering

This module is about writing tests that actually catch bugs, not tests that make your coverage badge green. There is a difference, and most codebases confuse the two.

---

## Table of Contents

1. [The Testing Pyramid](#1-the-testing-pyramid)
2. [Unit Testing Principles](#2-unit-testing-principles)
3. [TDD — Test-Driven Development](#3-tdd--test-driven-development)
4. [Mocking and Stubbing](#4-mocking-and-stubbing)
5. [Designing for Testability](#5-designing-for-testability)
6. [Integration Testing](#6-integration-testing)
7. [Code Quality Metrics](#7-code-quality-metrics)

---

## 1. The Testing Pyramid

### The Shape and the Ratio

The testing pyramid is a guideline about where to invest your testing effort. It is not a law, but when you violate it systematically, you pay for it.

```
          /\
         /  \
        / E2E\        ~10%   Slow, fragile, expensive to run and fix
       /------\
      /        \
     /  Integr. \      ~20%   Medium speed, test real interactions
    /------------\
   /              \
  /   Unit Tests   \   ~70%   Fast, isolated, cheap to write and run
 /__________________ \
```

**Unit tests (70%)** run in milliseconds. They test a single class or function in isolation. They fail fast, point exactly at the broken code, and you can run 10,000 of them before your coffee is ready.

**Integration tests (20%)** test the interaction between real components — your repository talking to a real (or in-memory) database, your service wiring with real collaborators. Slower than unit tests but catch a class of bugs unit tests cannot: wiring bugs, ORM mapping bugs, transaction boundary bugs.

**End-to-end tests (10%)** start the full application and drive it through a real scenario. They catch things nothing else catches, but they are slow (minutes), flaky (network, timing), and expensive to maintain. You want a small suite of E2E tests covering the happy paths and the most critical failure paths. No more.

### Why the ratio matters

The cost of a test is not just writing it — it is running it 50 times per day in CI, maintaining it when the code changes, and debugging it when it becomes flaky. The further down the pyramid a test lives, the more you pay per test. If you have 5,000 E2E tests, your CI pipeline will take 45 minutes and developers will stop running it.

Feedback loop speed matters just as much as cost. A unit test failure tells you in 300ms exactly which line broke. An E2E test failure tells you 8 minutes later that "something is wrong somewhere." Invest in the fastest feedback first.

### Anti-patterns

**The Ice Cream Cone (Inverted Pyramid)**

```
 ___________________________
/                           \    ~70%   E2E tests
/---------------------------\
/                           \    ~20%   Integration tests
/---------------------------\
/                           \    ~10%   Unit tests
/---------------------------\
```

This happens when teams skip unit tests because "we test it manually anyway" and write only E2E tests. CI takes forever, tests are flaky, every refactor breaks 50 tests, and developers start ignoring failures. This is the most common anti-pattern in mature, slow-moving codebases.

**The Hourglass**

```
          /\
         /  \
        / E2E\        ~40%   Lots of E2E tests
       /------\
      /        \
     /  Integr. \      ~5%    Almost no integration tests
    /------------\
   /              \
  /   Unit Tests   \   ~55%   Lots of unit tests
 /__________________ \
```

This happens when teams mock everything aggressively and skip integration tests. Unit tests pass, E2E tests run slowly, and the integration layer — where most real bugs live — is untested. Typical in teams that discovered mocking and got carried away.

---

## 2. Unit Testing Principles

### FIRST Principles

| Letter | Principle | What it means |
|--------|-----------|---------------|
| **F** | Fast | Runs in milliseconds. No network, no disk, no sleep calls. |
| **I** | Isolated | Test outcome does not depend on execution order. No shared mutable state. |
| **R** | Repeatable | Same result every time, on every machine, at any time of day. |
| **S** | Self-validating | The test itself passes or fails. You do not read output and decide. |
| **T** | Timely | Written at the same time as (ideally before) the production code. |

A test that violates **Isolated** is a test that sometimes passes and sometimes fails depending on which other test ran first. These are called flaky tests and they are worse than no tests, because they train your team to ignore failures.

A test that violates **Repeatable** typically has a dependency on `new Date()`, `Math.random()`, or a system clock. Section 5 shows how to fix this.

### Arrange-Act-Assert (AAA)

Every unit test has three phases. Separate them visually with a blank line.

```java
@Test
void processOrder_withSufficientInventory_chargesCustomerAndReducesStock() {
    // Arrange
    Order order = Order.of("ITEM-42", 3, Money.of(99.99));
    Inventory inventory = new Inventory(Map.of("ITEM-42", 10));
    PaymentGateway gateway = new FakePaymentGateway();
    OrderService service = new OrderService(inventory, gateway);

    // Act
    OrderResult result = service.processOrder(order);

    // Assert
    assertThat(result.isSuccess()).isTrue();
    assertThat(inventory.stockFor("ITEM-42")).isEqualTo(7);
}
```

Never mix phases. If your arrange section has assertions, or your act section creates objects, you will confuse future readers — including yourself.

### One logical assertion per test

"One assertion per test" is often stated too strictly. The real rule is **one logical concept per test**. You can have three `assertThat` calls if they all verify the same outcome. What you cannot do is test two different behaviors in one test method.

Bad:
```java
@Test
void processOrder() {
    // tests successful order AND tests out-of-stock behavior
    // in the same method — when it fails, you don't know which path broke
}
```

Good: two separate test methods, one for each scenario.

### Test Naming

The naming convention `methodName_scenario_expectedBehavior` pays off when tests fail in CI. The test name is your first diagnostic.

**Bad names:**
```
test1()
testProcessOrder()
processOrderTest()
shouldWork()
```

**Good names:**
```
processOrder_withSufficientStock_chargesCustomerAndDecreasesInventory()
processOrder_withInsufficientStock_throwsOutOfStockException()
processOrder_withNullCustomer_throwsIllegalArgumentException()
applyDiscount_forPremiumMember_appliesFifteenPercentDiscount()
applyDiscount_forGuestUser_appliesNoDiscount()
```

When this test fails in a CI build, you read the name and know exactly what broke before you open the code.

### Complete OrderService Example

Here is a realistic domain model with meaningful business logic:

```java
// OrderService.java
package com.example.orders;

import java.util.Objects;

public class OrderService {

    private final InventoryRepository inventoryRepository;
    private final PaymentGateway paymentGateway;
    private final OrderRepository orderRepository;

    public OrderService(InventoryRepository inventoryRepository,
                        PaymentGateway paymentGateway,
                        OrderRepository orderRepository) {
        this.inventoryRepository = Objects.requireNonNull(inventoryRepository);
        this.paymentGateway = Objects.requireNonNull(paymentGateway);
        this.orderRepository = Objects.requireNonNull(orderRepository);
    }

    public OrderResult processOrder(Order order) {
        if (order == null) {
            throw new IllegalArgumentException("Order must not be null");
        }

        int available = inventoryRepository.getStock(order.getSku());
        if (available < order.getQuantity()) {
            return OrderResult.failure("Insufficient stock for SKU: " + order.getSku());
        }

        PaymentResult payment = paymentGateway.charge(order.getCustomerId(), order.getTotalPrice());
        if (!payment.isSuccessful()) {
            return OrderResult.failure("Payment failed: " + payment.getDeclineReason());
        }

        inventoryRepository.decreaseStock(order.getSku(), order.getQuantity());
        Order saved = orderRepository.save(order.withStatus(OrderStatus.CONFIRMED));
        return OrderResult.success(saved.getId());
    }

    public Money calculateTotal(Order order, DiscountPolicy discountPolicy) {
        Money base = order.getUnitPrice().multiply(order.getQuantity());
        double discountRate = discountPolicy.discountRateFor(order.getCustomerTier());
        return base.multiply(1.0 - discountRate);
    }
}
```

```java
// OrderServiceTest.java
package com.example.orders;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private InventoryRepository inventoryRepository;

    @Mock
    private PaymentGateway paymentGateway;

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderService orderService;

    private Order standardOrder;

    @BeforeEach
    void setUp() {
        standardOrder = Order.builder()
                .sku("ITEM-42")
                .quantity(3)
                .unitPrice(Money.of(50.00))
                .customerId("CUST-001")
                .customerTier(CustomerTier.STANDARD)
                .build();
    }

    @Test
    void processOrder_withSufficientStockAndSuccessfulPayment_returnsSuccessAndSavesOrder() {
        // Arrange
        when(inventoryRepository.getStock("ITEM-42")).thenReturn(10);
        when(paymentGateway.charge("CUST-001", Money.of(150.00)))
                .thenReturn(PaymentResult.success("TXN-999"));
        when(orderRepository.save(any(Order.class)))
                .thenReturn(standardOrder.withId("ORD-777").withStatus(OrderStatus.CONFIRMED));

        // Act
        OrderResult result = orderService.processOrder(standardOrder);

        // Assert
        assertThat(result.isSuccess()).isTrue();
        assertThat(result.getOrderId()).isEqualTo("ORD-777");
    }

    @Test
    void processOrder_withInsufficientStock_returnsFailureWithoutChargingCustomer() {
        // Arrange
        when(inventoryRepository.getStock("ITEM-42")).thenReturn(2); // only 2, need 3

        // Act
        OrderResult result = orderService.processOrder(standardOrder);

        // Assert
        assertThat(result.isSuccess()).isFalse();
        assertThat(result.getErrorMessage()).contains("Insufficient stock");
        verify(paymentGateway, never()).charge(anyString(), any(Money.class));
    }

    @Test
    void processOrder_withDeclinedPayment_returnsFailureWithoutDecreasingStock() {
        // Arrange
        when(inventoryRepository.getStock("ITEM-42")).thenReturn(10);
        when(paymentGateway.charge("CUST-001", Money.of(150.00)))
                .thenReturn(PaymentResult.declined("Insufficient funds"));

        // Act
        OrderResult result = orderService.processOrder(standardOrder);

        // Assert
        assertThat(result.isSuccess()).isFalse();
        assertThat(result.getErrorMessage()).contains("Payment failed");
        verify(inventoryRepository, never()).decreaseStock(anyString(), anyInt());
    }

    @Test
    void processOrder_withNullOrder_throwsIllegalArgumentException() {
        assertThatThrownBy(() -> orderService.processOrder(null))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("Order must not be null");
    }

    @Test
    void calculateTotal_forPremiumMember_appliesFifteenPercentDiscount() {
        // Arrange
        Order premiumOrder = standardOrder.withCustomerTier(CustomerTier.PREMIUM);
        DiscountPolicy policy = new TieredDiscountPolicy(); // real object, not a mock

        // Act
        Money total = orderService.calculateTotal(premiumOrder, policy);

        // Assert
        // 3 items x $50 = $150, minus 15% = $127.50
        assertThat(total).isEqualTo(Money.of(127.50));
    }

    @Test
    void calculateTotal_forGuestUser_appliesNoDiscount() {
        // Arrange
        Order guestOrder = standardOrder.withCustomerTier(CustomerTier.GUEST);
        DiscountPolicy policy = new TieredDiscountPolicy();

        // Act
        Money total = orderService.calculateTotal(guestOrder, policy);

        // Assert
        assertThat(total).isEqualTo(Money.of(150.00));
    }
}
```

### Brittle Tests vs Resilient Tests

A brittle test breaks when you change implementation details that do not affect the contract. A resilient test breaks only when the behavior the test describes actually changes.

**Brittle test (tests implementation, not behavior):**
```java
@Test
void processOrder_brittle() {
    orderService.processOrder(standardOrder);

    // This verifies the internal call sequence — it breaks if you
    // refactor to batch the DB calls or reorder them
    InOrder inOrder = inOrder(inventoryRepository, paymentGateway, orderRepository);
    inOrder.verify(inventoryRepository).getStock("ITEM-42");
    inOrder.verify(paymentGateway).charge(any(), any());
    inOrder.verify(inventoryRepository).decreaseStock("ITEM-42", 3);
    inOrder.verify(orderRepository).save(any());
}
```

**Resilient test (tests observable behavior):**
```java
@Test
void processOrder_withValidOrder_confirmationIdIsReturned() {
    when(inventoryRepository.getStock("ITEM-42")).thenReturn(10);
    when(paymentGateway.charge(any(), any())).thenReturn(PaymentResult.success("TXN-1"));
    when(orderRepository.save(any())).thenReturn(confirmedOrder);

    OrderResult result = orderService.processOrder(standardOrder);

    assertThat(result.isSuccess()).isTrue();
}
```

The resilient test does not care about call order, method invocation counts, or which private method was used internally. It cares about what the caller observes.

---

## 3. TDD — Test-Driven Development

### Red-Green-Refactor

TDD is three-phase discipline:

1. **Red**: Write a failing test for the next small piece of behavior. It does not compile yet — that is fine.
2. **Green**: Write the minimum code to make the test pass. Do not over-engineer. Hard-code the return value if that is enough.
3. **Refactor**: Clean up the implementation without breaking the tests. Remove duplication, improve names, extract methods.

The cycle is short — ideally 2-5 minutes per iteration. If a cycle takes 30 minutes, the step is too large. Split it.

### TDD Walkthrough: TokenBucketRateLimiter

We will build a token bucket rate limiter from scratch using TDD. A token bucket allows bursts up to a maximum capacity and refills tokens over time.

**Step 1 — Red: Write the first failing test**

```java
// TokenBucketRateLimiterTest.java
package com.example.ratelimit;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class TokenBucketRateLimiterTest {

    @Test
    void tryAcquire_onFirstRequest_allowsRequest() {
        TokenBucketRateLimiter limiter = new TokenBucketRateLimiter(10, 1.0);

        boolean allowed = limiter.tryAcquire();

        assertThat(allowed).isTrue();
    }
}
```

This does not compile. `TokenBucketRateLimiter` does not exist. That is the red state.

**Step 2 — Green: Minimum code to pass**

```java
// TokenBucketRateLimiter.java
package com.example.ratelimit;

public class TokenBucketRateLimiter {

    public TokenBucketRateLimiter(int capacity, double refillRatePerSecond) {
    }

    public boolean tryAcquire() {
        return true;
    }
}
```

Test passes. Now add the next behavior.

**Step 3 — Red: Test that capacity is enforced**

```java
@Test
void tryAcquire_whenBucketIsEmpty_deniesRequest() {
    TokenBucketRateLimiter limiter = new TokenBucketRateLimiter(3, 1.0);

    limiter.tryAcquire(); // consume token 1
    limiter.tryAcquire(); // consume token 2
    limiter.tryAcquire(); // consume token 3
    boolean result = limiter.tryAcquire(); // bucket empty

    assertThat(result).isFalse();
}
```

Fails — our `tryAcquire()` always returns true.

**Step 4 — Green: Implement token counting**

```java
public class TokenBucketRateLimiter {

    private final int capacity;
    private int tokens;

    public TokenBucketRateLimiter(int capacity, double refillRatePerSecond) {
        this.capacity = capacity;
        this.tokens = capacity;
    }

    public boolean tryAcquire() {
        if (tokens <= 0) {
            return false;
        }
        tokens--;
        return true;
    }
}
```

Both tests pass.

**Step 5 — Red: Tokens refill over time**

Time-dependent code needs a clock abstraction so tests are deterministic (see Section 5 for full discussion).

```java
@Test
void tryAcquire_afterRefillPeriod_allowsRequestAgain() {
    MutableClock clock = new MutableClock();
    TokenBucketRateLimiter limiter = new TokenBucketRateLimiter(1, 1.0, clock);

    limiter.tryAcquire(); // consume the only token
    assertThat(limiter.tryAcquire()).isFalse(); // empty

    clock.advance(Duration.ofSeconds(2)); // advance time by 2 seconds -> 2 new tokens, capped at capacity 1

    assertThat(limiter.tryAcquire()).isTrue();
}
```

**Step 6 — Green: Add time-based refill**

```java
import java.time.Clock;
import java.time.Duration;
import java.time.Instant;

public class TokenBucketRateLimiter {

    private final int capacity;
    private final double refillRatePerSecond;
    private final Clock clock;
    private double tokens;
    private Instant lastRefill;

    public TokenBucketRateLimiter(int capacity, double refillRatePerSecond) {
        this(capacity, refillRatePerSecond, Clock.systemUTC());
    }

    public TokenBucketRateLimiter(int capacity, double refillRatePerSecond, Clock clock) {
        this.capacity = capacity;
        this.refillRatePerSecond = refillRatePerSecond;
        this.clock = clock;
        this.tokens = capacity;
        this.lastRefill = clock.instant();
    }

    public synchronized boolean tryAcquire() {
        refill();
        if (tokens < 1.0) {
            return false;
        }
        tokens -= 1.0;
        return true;
    }

    private void refill() {
        Instant now = clock.instant();
        double secondsElapsed = Duration.between(lastRefill, now).toMillis() / 1000.0;
        tokens = Math.min(capacity, tokens + secondsElapsed * refillRatePerSecond);
        lastRefill = now;
    }
}
```

**Step 7 — Refactor: Extract the MutableClock test helper**

```java
// MutableClock.java — test helper, lives in src/test
package com.example.ratelimit.testutil;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.time.ZoneId;

public class MutableClock extends Clock {

    private Instant now;
    private final ZoneId zone;

    public MutableClock() {
        this(Instant.EPOCH, ZoneId.of("UTC"));
    }

    public MutableClock(Instant initial, ZoneId zone) {
        this.now = initial;
        this.zone = zone;
    }

    public void advance(Duration duration) {
        this.now = this.now.plus(duration);
    }

    @Override
    public ZoneId getZone() { return zone; }

    @Override
    public Clock withZone(ZoneId zone) { return new MutableClock(now, zone); }

    @Override
    public Instant instant() { return now; }
}
```

All three tests pass. Add edge cases:

```java
@Test
void tryAcquire_doesNotExceedCapacityAfterLongIdle() {
    MutableClock clock = new MutableClock();
    TokenBucketRateLimiter limiter = new TokenBucketRateLimiter(5, 1.0, clock);

    // drain the bucket
    for (int i = 0; i < 5; i++) limiter.tryAcquire();
    assertThat(limiter.tryAcquire()).isFalse();

    // wait a very long time
    clock.advance(Duration.ofHours(24));

    // should have exactly capacity tokens, not 24*3600
    int acquired = 0;
    while (limiter.tryAcquire()) acquired++;

    assertThat(acquired).isEqualTo(5);
}

@Test
void tryAcquire_withZeroCapacity_alwaysDenies() {
    TokenBucketRateLimiter limiter = new TokenBucketRateLimiter(0, 1.0);

    assertThat(limiter.tryAcquire()).isFalse();
}
```

### TDD Example 2: NotificationService

The goal: a `NotificationService` that sends notifications through a channel but suppresses duplicate notifications within a cooldown window.

```java
// NotificationServiceTest.java
package com.example.notifications;

import com.example.testutil.MutableClock;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Duration;

import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class NotificationServiceTest {

    @Mock
    private NotificationChannel channel;

    private MutableClock clock;
    private NotificationService service;

    @BeforeEach
    void setUp() {
        clock = new MutableClock();
        service = new NotificationService(channel, Duration.ofMinutes(5), clock);
    }

    @Test
    void send_firstNotificationForRecipient_delegatesToChannel() {
        service.send("user@example.com", "Your order shipped");

        verify(channel).send("user@example.com", "Your order shipped");
    }

    @Test
    void send_duplicateNotificationWithinCooldown_suppressesSecondSend() {
        service.send("user@example.com", "Alert: login from new device");
        service.send("user@example.com", "Alert: login from new device");

        verify(channel, times(1)).send(anyString(), anyString());
    }

    @Test
    void send_sameRecipientDifferentMessage_sendsBothMessages() {
        service.send("user@example.com", "Order shipped");
        service.send("user@example.com", "Order delivered");

        verify(channel, times(2)).send(eq("user@example.com"), anyString());
    }

    @Test
    void send_afterCooldownExpires_sendsAgain() {
        service.send("user@example.com", "Alert");
        clock.advance(Duration.ofMinutes(6)); // past cooldown
        service.send("user@example.com", "Alert");

        verify(channel, times(2)).send("user@example.com", "Alert");
    }
}
```

Implementation driven by these tests:

```java
// NotificationService.java
package com.example.notifications;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

public class NotificationService {

    private final NotificationChannel channel;
    private final Duration cooldown;
    private final Clock clock;
    private final Map<String, Instant> lastSentAt = new HashMap<>();

    public NotificationService(NotificationChannel channel, Duration cooldown, Clock clock) {
        this.channel = Objects.requireNonNull(channel);
        this.cooldown = Objects.requireNonNull(cooldown);
        this.clock = Objects.requireNonNull(clock);
    }

    public void send(String recipient, String message) {
        String key = recipient + "|" + message;
        Instant now = clock.instant();
        Instant last = lastSentAt.get(key);

        if (last != null && now.isBefore(last.plus(cooldown))) {
            return; // suppressed
        }

        channel.send(recipient, message);
        lastSentAt.put(key, now);
    }
}
```

### Outside-In TDD (London School) vs Inside-Out (Chicago School)

These are two different answers to the question: "Where do you start?"

**Inside-Out (Chicago / Classicist)**

Start from the core domain objects and work outward. Write unit tests for the `TokenBucket` before you have a `RateLimiterService` or any HTTP layer. Tests use real collaborators where possible. Mocks are used only for external infrastructure.

Strengths: design emerges naturally from domain logic. You rarely over-engineer.
Weakness: you might build the right internals but wire them together incorrectly.

**Outside-In (London / Mockist)**

Start from the outermost entry point (an HTTP controller or a command handler) and mock all collaborators. Drive the design from the consumer's perspective inward. Each collaborator's interface is defined by how the outer layer needs to use it, not by how it wants to be implemented.

Strengths: every interface is designed from the consumer's perspective. No dead code.
Weakness: heavy use of mocks. Risk of testing the mock rather than the real system (see Section 4). Requires discipline to also write integration tests.

**Practical guidance:** Use Inside-Out for domain logic with complex rules. Use Outside-In when you are building a new service endpoint and want the API contract to drive the internals. Most experienced TDD practitioners blend both.

### When TDD helps vs when it slows you down

TDD helps when:
- The behavior has clear inputs and expected outputs (pure functions, domain rules, algorithms)
- The space of edge cases is large (validation logic, state machines, rate limiters)
- You are refactoring existing code — tests make refactoring safe
- The design is unclear — writing a test forces you to decide the API before the implementation

TDD slows you down when:
- You are doing exploratory programming — you do not know what the right design is yet. Spike first, then delete the spike and write the real thing with TDD.
- The code is primarily orchestration with no logic — a method that calls three other methods in sequence does not benefit much from TDD
- The UI layout is what you are figuring out
- You are in a time-critical incident response

---

## 4. Mocking and Stubbing

### The Four Test Doubles

These terms are often used interchangeably. They mean different things.

**Stub**: Returns pre-configured responses to calls. Does not record interactions. Use when you need a collaborator to return specific data.

```java
// A stub: it returns a fixed value when called
when(inventoryRepository.getStock("ITEM-42")).thenReturn(10);
```

**Mock**: A stub that also records interactions. You can verify that specific calls were made to it. Use when the test needs to confirm that a side-effecting method was (or was not) called.

```java
// A mock: you can verify behavior on it
verify(paymentGateway).charge("CUST-001", Money.of(150.00));
verify(emailService, never()).sendFailureAlert(anyString());
```

**Fake**: A real, working implementation that is simplified for tests. Faster and lighter than the production implementation. A `HashMap`-backed `UserRepository` is a fake. Fakes have logic; stubs do not.

```java
public class FakeOrderRepository implements OrderRepository {
    private final Map<String, Order> store = new HashMap<>();
    private int idCounter = 1;

    @Override
    public Order save(Order order) {
        String id = "ORD-" + idCounter++;
        Order saved = order.withId(id);
        store.put(id, saved);
        return saved;
    }

    @Override
    public Optional<Order> findById(String id) {
        return Optional.ofNullable(store.get(id));
    }
}
```

**Spy**: A real object with some methods intercepted. Use when you want to test a real object but need to override one specific method or verify that it was called.

```java
// A spy wraps a real object
NotificationService realService = new NotificationService(channel, Duration.ofMinutes(5), clock);
NotificationService spy = Mockito.spy(realService);

// Override one method
doReturn(true).when(spy).isWithinBusinessHours();

spy.send("user@example.com", "Hello");
verify(spy).isWithinBusinessHours(); // verify it was called on the real object
```

### Complete Mockito Guide

#### Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
</dependency>
```

Activate Mockito in your test class:

```java
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class MyServiceTest {
    // ...
}
```

#### @Mock

Creates a mock of the interface or class. All methods return default values (`null` for objects, `0` for int, `false` for boolean) unless you stub them.

```java
import org.mockito.Mock;

@ExtendWith(MockitoExtension.class)
class PaymentProcessorTest {

    @Mock
    private PaymentGateway paymentGateway; // all methods return null/false/0 by default

    @Test
    void charge_whenGatewaySucceeds_returnsConfirmationId() {
        // Stub: tell the mock what to return for a specific call
        when(paymentGateway.charge("CUST-001", Money.of(99.99)))
                .thenReturn(PaymentResult.success("TXN-12345"));

        PaymentResult result = paymentGateway.charge("CUST-001", Money.of(99.99));

        assertThat(result.getConfirmationId()).isEqualTo("TXN-12345");
    }
}
```

#### @InjectMocks

Tells Mockito to create an instance of the class under test and inject all `@Mock` fields into it (via constructor, setter, or field injection — constructor is preferred).

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private InventoryRepository inventoryRepository;

    @Mock
    private PaymentGateway paymentGateway;

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderService orderService; // Mockito constructs this with the mocks above

    @Test
    void processOrder_withStockAvailable_callsPaymentGateway() {
        when(inventoryRepository.getStock(anyString())).thenReturn(100);
        when(paymentGateway.charge(anyString(), any())).thenReturn(PaymentResult.success("TXN-1"));
        when(orderRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

        orderService.processOrder(standardOrder);

        verify(paymentGateway).charge("CUST-001", Money.of(150.00));
    }
}
```

#### @Spy

Wraps a real instance. Real methods are called by default; you can override specific ones.

```java
import org.mockito.Spy;

@ExtendWith(MockitoExtension.class)
class AuditServiceTest {

    @Spy
    private AuditLogger auditLogger = new AuditLogger(); // real instance

    @Test
    void recordAction_writesToAuditLog() {
        // Real method runs, but we can verify it was called
        auditLogger.record("USER-1", Action.LOGIN);

        verify(auditLogger).record("USER-1", Action.LOGIN);
    }

    @Test
    void recordAction_whenLoggerFails_doesNotPropagateException() {
        // Override one method on the spy
        doThrow(new RuntimeException("Disk full"))
                .when(auditLogger).flushToDisk();

        // Test that your code handles the failure gracefully
        assertThatCode(() -> auditLogger.record("USER-1", Action.LOGIN))
                .doesNotThrowAnyException();
    }
}
```

#### @Captor and ArgumentCaptor

Use when you need to inspect what object was passed to a mock method. Essential for testing complex arguments that do not have a useful `equals` method.

```java
import org.mockito.ArgumentCaptor;
import org.mockito.Captor;

@ExtendWith(MockitoExtension.class)
class NotificationServiceTest {

    @Mock
    private NotificationChannel channel;

    @Captor
    private ArgumentCaptor<EmailMessage> emailCaptor;

    @InjectMocks
    private NotificationService service;

    @Test
    void sendOrderConfirmation_includesOrderIdAndCustomerName() {
        Order order = Order.builder()
                .id("ORD-500")
                .customerId("CUST-001")
                .customerName("Alice Smith")
                .build();

        service.sendOrderConfirmation(order);

        verify(channel).send(emailCaptor.capture());
        EmailMessage sent = emailCaptor.getValue();

        assertThat(sent.getSubject()).contains("ORD-500");
        assertThat(sent.getBody()).contains("Alice Smith");
        assertThat(sent.getRecipient()).isEqualTo("CUST-001");
    }
}
```

#### Stubbing Patterns

```java
// Return a value
when(repo.findById("1")).thenReturn(Optional.of(user));

// Return different values on successive calls
when(idGenerator.next())
        .thenReturn("ID-001")
        .thenReturn("ID-002")
        .thenReturn("ID-003");

// Throw an exception
when(gateway.charge(any(), any()))
        .thenThrow(new PaymentGatewayException("Timeout"));

// doThrow — use for void methods
doThrow(new RuntimeException("DB down"))
        .when(auditLogger).record(anyString(), any());

// Answer — compute the return value dynamically
when(orderRepository.save(any(Order.class)))
        .thenAnswer(invocation -> {
            Order order = invocation.getArgument(0);
            return order.withId("ORD-" + System.nanoTime());
        });

// Argument matchers — use when you don't care about the exact value
when(repo.findByStatus(any(OrderStatus.class))).thenReturn(List.of());
when(service.process(anyString())).thenReturn(true);
when(service.processWithCount(eq("ITEM-42"), intThat(n -> n > 0))).thenReturn(true);
```

#### verify() patterns

```java
// Was called exactly once (default)
verify(paymentGateway).charge("CUST-001", Money.of(150.00));

// Was never called
verify(emailService, never()).sendFailureAlert(anyString());

// Was called exactly N times
verify(retryHandler, times(3)).attempt(any());

// Was called at least once
verify(logger, atLeastOnce()).warn(anyString());

// Was called at most N times
verify(cacheService, atMost(2)).get(anyString());

// No other interactions happened on this mock
verifyNoMoreInteractions(paymentGateway);

// No interactions at all
verifyNoInteractions(auditLogger);
```

#### Mockito Best Practices

**Verify behavior, not state, when the behavior IS the contract.** For a method that sends an email, you cannot check state — the email is gone. You verify the call happened.

**Use ArgumentCaptor for complex objects.** Do not stub `equals` to make matching work. Capture the argument and assert on its fields.

**Do not stub what you do not need.** If a test sets up `when(repo.findById("1"))` but never actually triggers that path, remove the stub. Unnecessary stubs are noise. Mockito's strict stubs mode will fail on unnecessary stubs — enable it:

```java
@ExtendWith(MockitoExtension.class) // strict stubs are on by default in MockitoExtension
```

**Do not mock value objects.** `Money`, `OrderId`, `Address` — these have no behavior that needs mocking. Construct them directly.

### The Over-Mocking Anti-Pattern

This test passes, but it tests nothing about the production code:

```java
@Test
void processOrder_overtMocked_testsNothing() {
    // Every single collaborator is mocked
    OrderService service = mock(OrderService.class);

    // Stub the method under test itself
    when(service.processOrder(any())).thenReturn(OrderResult.success("ORD-1"));

    OrderResult result = service.processOrder(standardOrder);

    // This verifies only that Mockito stubs work correctly
    assertThat(result.isSuccess()).isTrue();
}
```

This test would pass even if `OrderService.processOrder` threw a `NullPointerException` or deleted the database. You are testing your mock configuration, not your code.

A subtler version:

```java
@Test
void processOrder_tooManyVerifications() {
    when(inventoryRepository.getStock(anyString())).thenReturn(10);
    when(paymentGateway.charge(any(), any())).thenReturn(PaymentResult.success("TXN-1"));
    when(orderRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

    orderService.processOrder(standardOrder);

    // Testing that OrderService calls its dependencies in the right order —
    // this is an implementation detail, not a contract
    InOrder order = inOrder(inventoryRepository, paymentGateway, orderRepository);
    order.verify(inventoryRepository).getStock("ITEM-42");
    order.verify(paymentGateway).charge(eq("CUST-001"), any());
    order.verify(orderRepository).save(any());
}
```

This test breaks on any internal refactor. The actual contract is: given sufficient stock and successful payment, return a success result and save the order. Test that.

### Mock What You Don't Own

The rule: **mock external systems and infrastructure you did not write**. Do not mock your own domain objects.

Mock: `PaymentGateway`, `EmailService`, `UserRepository`, `HttpClient`
Do not mock: `Order`, `Money`, `OrderId`, `CustomerTier`, your own `DiscountPolicy`

If you find yourself mocking a domain object to test another domain object, your design probably has a problem — extract an interface.

---

## 5. Designing for Testability

Hard-to-test code is not just a testing problem. It is a design problem. The same properties that make code hard to test — hidden dependencies, global state, hardcoded collaborators — also make code hard to understand and change.

### Dependency Injection Makes Code Testable

**Hard to test — dependency hidden inside constructor:**

```java
// Bad: hardcoded dependency, impossible to test in isolation
public class ReportService {

    private final ReportRepository repository = new ReportRepository(); // cannot substitute

    public Report generate(String reportId) {
        ReportData data = repository.fetch(reportId);
        return new Report(data.getRows(), data.getHeaders());
    }
}
```

**Testable — dependency injected:**

```java
// Good: dependency injected, can be replaced in tests
public class ReportService {

    private final ReportRepository repository;

    public ReportService(ReportRepository repository) {
        this.repository = Objects.requireNonNull(repository);
    }

    public Report generate(String reportId) {
        ReportData data = repository.fetch(reportId);
        return new Report(data.getRows(), data.getHeaders());
    }
}

// In tests:
@Mock ReportRepository repository;
@InjectMocks ReportService service;
```

Always prefer **constructor injection** over field or setter injection. Constructor injection makes dependencies explicit and means the object cannot exist without its dependencies — which is usually what you want.

### Avoiding Static Methods and Singletons

Static methods and singletons are global state. You cannot replace them in tests.

**Hard to test:**

```java
public class InvoiceService {

    public Invoice createInvoice(Order order) {
        String invoiceId = IdGenerator.generate(); // static — cannot control the output
        LocalDate dueDate = LocalDate.now().plusDays(30); // system clock — non-deterministic
        return new Invoice(invoiceId, dueDate, order.getTotalPrice());
    }
}
```

**If you must use a static utility, wrap it:**

```java
// Wrap the static into an interface
public interface IdGenerator {
    String generate();
}

public class UuidIdGenerator implements IdGenerator {
    @Override
    public String generate() {
        return UUID.randomUUID().toString();
    }
}

// In tests, use a deterministic fake
public class SequentialIdGenerator implements IdGenerator {
    private int counter = 1;
    @Override
    public String generate() { return "ID-" + counter++; }
}
```

### The Seam Concept (Michael Feathers)

A **seam** is a place in the code where you can change the program's behavior without editing the code at that point. Seams are where you insert test doubles.

Feathers identified three types:
- **Object seam**: The behavior changes based on which object is passed in (dependency injection creates object seams)
- **Link seam**: Which class is loaded (classpath manipulation — rarely used in modern Java)
- **Preprocessing seam**: Conditional compilation — not relevant in Java

In practice, every interface creates a seam. When you depend on `PaymentGateway` (interface) instead of `StripePaymentGateway` (class), you have a seam. In tests, you pass a different implementation through that seam.

Refactoring code to be testable is mostly the act of **identifying where seams should be and introducing them**.

### Testing Time-Dependent Code

Code that calls `LocalDate.now()` or `Instant.now()` directly is non-deterministic and cannot be tested reliably. The fix is to inject `java.time.Clock`.

```java
// Hard to test
public class SubscriptionService {

    public boolean isSubscriptionActive(Subscription subscription) {
        return LocalDate.now().isBefore(subscription.getExpiryDate()); // non-deterministic
    }
}

// Testable
public class SubscriptionService {

    private final Clock clock;

    public SubscriptionService(Clock clock) {
        this.clock = clock;
    }

    public boolean isSubscriptionActive(Subscription subscription) {
        LocalDate today = LocalDate.now(clock); // deterministic — clock is injected
        return today.isBefore(subscription.getExpiryDate());
    }
}
```

In production, wire in `Clock.systemUTC()`. In tests:

```java
@Test
void isSubscriptionActive_onExpiryDate_returnsTrue() {
    LocalDate expiryDate = LocalDate.of(2025, 12, 31);
    Clock fixedClock = Clock.fixed(
            LocalDate.of(2025, 12, 31).atStartOfDay(ZoneOffset.UTC).toInstant(),
            ZoneOffset.UTC
    );
    SubscriptionService service = new SubscriptionService(fixedClock);
    Subscription subscription = Subscription.expiring(expiryDate);

    assertThat(service.isSubscriptionActive(subscription)).isTrue();
}

@Test
void isSubscriptionActive_afterExpiryDate_returnsFalse() {
    LocalDate expiryDate = LocalDate.of(2025, 12, 31);
    Clock fixedClock = Clock.fixed(
            LocalDate.of(2026, 1, 1).atStartOfDay(ZoneOffset.UTC).toInstant(),
            ZoneOffset.UTC
    );
    SubscriptionService service = new SubscriptionService(fixedClock);
    Subscription subscription = Subscription.expiring(expiryDate);

    assertThat(service.isSubscriptionActive(subscription)).isFalse();
}
```

### Testing Random Behavior

Do not call `Math.random()` or `new Random()` directly. Inject a seed or a `Supplier<Double>`.

```java
// Hard to test
public class PricingEngine {

    public Money applyDynamicPricing(Money basePrice) {
        double multiplier = 0.9 + Math.random() * 0.2; // 0.9 to 1.1 — non-deterministic
        return basePrice.multiply(multiplier);
    }
}

// Testable
public class PricingEngine {

    private final Supplier<Double> randomSource;

    public PricingEngine(Supplier<Double> randomSource) {
        this.randomSource = randomSource;
    }

    public Money applyDynamicPricing(Money basePrice) {
        double multiplier = 0.9 + randomSource.get() * 0.2;
        return basePrice.multiply(multiplier);
    }
}
```

Production wiring: `new PricingEngine(Math::random)`
Test wiring: `new PricingEngine(() -> 0.5)` — always returns a multiplier of `1.0`

```java
@Test
void applyDynamicPricing_atMidpointRandom_returnsBasePrice() {
    PricingEngine engine = new PricingEngine(() -> 0.5); // deterministic

    Money result = engine.applyDynamicPricing(Money.of(100.00));

    assertThat(result).isEqualTo(Money.of(100.00)); // 0.9 + 0.5 * 0.2 = 1.0 multiplier
}

@Test
void applyDynamicPricing_atMaxRandom_appliesTenPercentMarkup() {
    PricingEngine engine = new PricingEngine(() -> 1.0);

    Money result = engine.applyDynamicPricing(Money.of(100.00));

    assertThat(result).isEqualTo(Money.of(110.00)); // 0.9 + 1.0 * 0.2 = 1.1 multiplier
}
```

### External Services: Interface + Fake Pattern

For every external service (payment gateway, email provider, SMS service, third-party API), define an interface in your domain and provide two implementations: the real one and a fake one for tests.

```java
// The interface lives in your domain
public interface EmailGateway {
    void send(EmailMessage message);
}

// Real implementation (in production code)
public class SendGridEmailGateway implements EmailGateway {
    private final SendGridClient client;

    public SendGridEmailGateway(SendGridClient client) {
        this.client = client;
    }

    @Override
    public void send(EmailMessage message) {
        client.sendEmail(/* translate to SendGrid's API */);
    }
}

// Fake implementation (in test code — src/test/java)
public class InMemoryEmailGateway implements EmailGateway {

    private final List<EmailMessage> sentMessages = new ArrayList<>();

    @Override
    public void send(EmailMessage message) {
        sentMessages.add(message);
    }

    public List<EmailMessage> getSentMessages() {
        return Collections.unmodifiableList(sentMessages);
    }

    public boolean wasEmailSentTo(String recipient) {
        return sentMessages.stream()
                .anyMatch(m -> m.getTo().equals(recipient));
    }

    public void clear() {
        sentMessages.clear();
    }
}
```

Using the fake in tests:

```java
class PasswordResetServiceTest {

    private InMemoryEmailGateway emailGateway = new InMemoryEmailGateway();
    private PasswordResetService service;

    @BeforeEach
    void setUp() {
        service = new PasswordResetService(emailGateway, userRepository, tokenRepository);
    }

    @Test
    void requestReset_forKnownUser_sendsEmailWithResetLink() {
        when(userRepository.findByEmail("alice@example.com"))
                .thenReturn(Optional.of(User.withEmail("alice@example.com")));

        service.requestReset("alice@example.com");

        assertThat(emailGateway.wasEmailSentTo("alice@example.com")).isTrue();
        assertThat(emailGateway.getSentMessages()).hasSize(1);
        assertThat(emailGateway.getSentMessages().get(0).getSubject())
                .contains("Password Reset");
    }
}
```

The fake is richer than a mock — it has state and you can query that state. It does not throw `UnnecessaryStubbingException` when methods are called with different arguments. Use fakes for stateful collaborators; use mocks for simple side-effect verification.

### Side-by-Side Comparison

| Hard to test | Testable |
|---|---|
| `new SomeDependency()` inside constructor | Constructor takes `SomeDependency` as parameter |
| `LocalDate.now()` inline | `LocalDate.now(clock)` with injected `Clock` |
| `Math.random()` inline | `Supplier<Double> randomSource` injected |
| `static EmailService.send(...)` | `EmailGateway` interface injected |
| Singleton `Config.getInstance()` | `Config` injected via constructor |
| `new HttpClient()` inside method | `HttpClient` injected and referenced via interface |

---

## 6. Integration Testing

### When to Write Integration Tests

Write an integration test when:
- You have a repository class with complex JPA queries — unit tests cannot catch mapping errors
- You have a controller with security rules, input validation, or response serialization that needs to be verified as a stack
- You are testing a workflow that spans multiple services and the interaction is what matters
- You need to verify that transaction boundaries are correct (a bug that unit tests with mocks will never catch)

Do not write an integration test for:
- Pure business logic (that belongs in unit tests)
- Happy path scenarios already covered by unit tests (testing the same logic at a higher cost)
- Every method in every class

### TestContainers for Database Testing

TestContainers starts a real database in Docker for your tests, then tears it down. Your tests run against the real database engine, not an H2 substitute.

The pattern: spin up a container, wire your `DataSource` to it, run your tests, container stops.

```java
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import javax.sql.DataSource;
import org.springframework.jdbc.datasource.DriverManagerDataSource;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
class OrderRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
            .withDatabaseName("orders_test")
            .withUsername("test")
            .withPassword("test");

    private OrderRepository repository;

    @BeforeEach
    void setUp() {
        DataSource dataSource = new DriverManagerDataSource(
                postgres.getJdbcUrl(),
                postgres.getUsername(),
                postgres.getPassword()
        );
        repository = new JdbcOrderRepository(dataSource);
        // Run schema migrations here (Flyway/Liquibase)
    }

    @Test
    void save_newOrder_assignsGeneratedId() {
        Order order = Order.builder()
                .sku("ITEM-42")
                .quantity(2)
                .customerId("CUST-001")
                .status(OrderStatus.PENDING)
                .build();

        Order saved = repository.save(order);

        assertThat(saved.getId()).isNotNull();
        assertThat(saved.getId()).startsWith("ORD-");
    }

    @Test
    void findByCustomerId_withMultipleOrders_returnsAllForCustomer() {
        repository.save(orderFor("CUST-001"));
        repository.save(orderFor("CUST-001"));
        repository.save(orderFor("CUST-002")); // different customer

        List<Order> orders = repository.findByCustomerId("CUST-001");

        assertThat(orders).hasSize(2);
        assertThat(orders).allMatch(o -> o.getCustomerId().equals("CUST-001"));
    }

    @Test
    void findById_afterSave_returnsOrderWithCorrectFields() {
        Order order = Order.builder()
                .sku("ITEM-99")
                .quantity(5)
                .customerId("CUST-003")
                .status(OrderStatus.CONFIRMED)
                .build();

        String id = repository.save(order).getId();
        Optional<Order> found = repository.findById(id);

        assertThat(found).isPresent();
        assertThat(found.get().getSku()).isEqualTo("ITEM-99");
        assertThat(found.get().getQuantity()).isEqualTo(5);
        assertThat(found.get().getStatus()).isEqualTo(OrderStatus.CONFIRMED);
    }

    private Order orderFor(String customerId) {
        return Order.builder()
                .sku("ITEM-42")
                .quantity(1)
                .customerId(customerId)
                .status(OrderStatus.PENDING)
                .build();
    }
}
```

Key points: The `@Container` annotation with `static` means the container starts once per test class and is reused across test methods — much faster than one container per test. `@Testcontainers` wires the lifecycle.

### Spring Boot Test Slices

Spring Boot's test slices load only a relevant slice of the application context, not the full context. Full context loading is slow and gives you a big, hard-to-understand test scope.

**@DataJpaTest** — loads only JPA-related beans (repositories, entity manager). Uses an embedded database by default (configurable to use TestContainers).

```java
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
import org.springframework.beans.factory.annotation.Autowired;

@DataJpaTest
class OrderJpaRepositoryTest {

    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    private OrderJpaRepository repository;

    @Test
    void findByStatus_withMultipleStatuses_returnsOnlyMatching() {
        entityManager.persistAndFlush(
                OrderEntity.builder().status(OrderStatus.CONFIRMED).build());
        entityManager.persistAndFlush(
                OrderEntity.builder().status(OrderStatus.PENDING).build());
        entityManager.persistAndFlush(
                OrderEntity.builder().status(OrderStatus.CONFIRMED).build());

        List<OrderEntity> confirmed = repository.findByStatus(OrderStatus.CONFIRMED);

        assertThat(confirmed).hasSize(2);
    }
}
```

**@WebMvcTest** — loads only the web layer (controllers, filters, argument resolvers). Service layer is not loaded; you mock it.

```java
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.MediaType;

import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderService orderService; // Spring-managed mock

    @Test
    void createOrder_withValidBody_returns201AndOrderId() throws Exception {
        when(orderService.processOrder(any())).thenReturn(OrderResult.success("ORD-1"));

        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                {
                                  "sku": "ITEM-42",
                                  "quantity": 3,
                                  "customerId": "CUST-001"
                                }
                                """))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.orderId").value("ORD-1"));
    }

    @Test
    void createOrder_withMissingSku_returns400() throws Exception {
        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                {
                                  "quantity": 3,
                                  "customerId": "CUST-001"
                                }
                                """))
                .andExpect(status().isBadRequest());
    }
}
```

`@WebMvcTest` catches: wrong HTTP status codes, missing validation annotations, incorrect response serialization, wrong URL mappings — none of which unit tests catch.

### In-Memory Test Doubles for External Systems

For external systems that you cannot easily spin up in tests (legacy messaging systems, third-party SaaS APIs), build an in-memory fake and use it across your test suite. You already saw the `InMemoryEmailGateway` pattern. Here is the same for a message queue:

```java
public class InMemoryMessageQueue implements MessageQueue {

    private final Map<String, Queue<Message>> queues = new ConcurrentHashMap<>();

    @Override
    public void publish(String topic, Message message) {
        queues.computeIfAbsent(topic, k -> new LinkedList<>()).add(message);
    }

    @Override
    public Optional<Message> poll(String topic) {
        Queue<Message> queue = queues.get(topic);
        return queue == null ? Optional.empty() : Optional.ofNullable(queue.poll());
    }

    public List<Message> getAllMessages(String topic) {
        return new ArrayList<>(queues.getOrDefault(topic, new LinkedList<>()));
    }

    public int messageCount(String topic) {
        return queues.getOrDefault(topic, new LinkedList<>()).size();
    }

    public void clear() {
        queues.clear();
    }
}
```

This fake is fast, deterministic, and you can query its state directly in assertions. It tests your code's interaction with the queue interface without depending on Kafka or RabbitMQ being available.

---

## 7. Code Quality Metrics

### Test Coverage: What It Means and What It Does Not

Test coverage measures the percentage of production lines (or branches) executed during the test suite. An 80% coverage figure means 80% of lines were executed by at least one test.

**What coverage guarantees**: that specific lines were executed.

**What coverage does not guarantee**:
- That the correct behavior was verified (a test can execute a line without asserting anything about the result)
- That all edge cases are covered (a branch may execute in the happy path but not in the error path)
- That the code is correct

Here is a class with 100% coverage and zero meaningful assertions:

```java
public class PriceCalculator {

    public double calculateFinalPrice(double basePrice, double taxRate, String couponCode) {
        double price = basePrice;

        if ("SAVE10".equals(couponCode)) {
            price = price * 0.9;
        }

        if (taxRate > 0) {
            price = price + (price * taxRate);
        }

        if (price < 0) {
            throw new IllegalArgumentException("Price cannot be negative");
        }

        return price;
    }
}
```

```java
// 100% line coverage. Zero assertions. Completely useless.
@Test
void coverage_without_assertions() {
    PriceCalculator calc = new PriceCalculator();

    calc.calculateFinalPrice(100.0, 0.1, "SAVE10");  // all branches hit
    calc.calculateFinalPrice(100.0, 0.0, null);
    assertThatThrownBy(() -> calc.calculateFinalPrice(-1000.0, 0.0, null))
            .isInstanceOf(IllegalArgumentException.class);
    // No assertion on what the calculated price actually is
}
```

Coverage passes. The code could return `0.0` for everything and all tests would still pass.

**The right target**: Aim for 80% branch coverage as a floor, not a ceiling. Beyond 80%, marginal test coverage (testing trivial getters, logging statements, toString methods) has diminishing value and increasing maintenance cost. Focus on covering business-critical paths fully, not on the percentage.

### Mutation Testing

Mutation testing answers the question coverage cannot: "Do my tests actually verify the right behavior?"

A mutation testing tool (PIT is the standard for Java) makes small syntactic changes to your production code — called **mutants** — and then runs your test suite against each mutant. If a test fails, the mutant is "killed." If all tests pass despite the code being wrong, the mutant "survived."

Example mutations PIT makes:
- `if (stock < quantity)` → `if (stock <= quantity)`
- `return true` → `return false`
- `price * 0.9` → `price * 0.1`
- `if (a && b)` → `if (a || b)`
- Remove a method call
- Negate a conditional

A **mutation score** of 80% means 80% of mutations were caught by your tests. A 40% mutation score with 90% line coverage means your tests exercise code without verifying behavior — a common situation in test suites optimized for coverage metrics.

**PIT Maven plugin configuration:**

```xml
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.15.3</version>
    <dependencies>
        <dependency>
            <groupId>org.pitest</groupId>
            <artifactId>pitest-junit5-plugin</artifactId>
            <version>1.2.1</version>
        </dependency>
    </dependencies>
    <configuration>
        <targetClasses>
            <param>com.example.orders.*</param>
        </targetClasses>
        <targetTests>
            <param>com.example.orders.*Test</param>
        </targetTests>
        <mutationThreshold>75</mutationThreshold>
    </configuration>
</plugin>
```

Run: `mvn test-compile org.pitest:pitest-maven:mutationCoverage`

PIT generates an HTML report showing which mutants survived and where. Surviving mutants tell you exactly which assertions you are missing.

### Cyclomatic Complexity

Cyclomatic complexity measures the number of linearly independent paths through a piece of code. The formula:

```
CC = E - N + 2P
```

Where:
- `E` = number of edges in the control flow graph
- `N` = number of nodes
- `P` = number of connected components (usually 1 per method)

In practice, you calculate it by counting decision points:

```
CC = 1 + (number of: if, else if, while, for, case, catch, &&, ||, ternary ? :)
```

| CC Value | Interpretation |
|---|---|
| 1-5 | Simple, low risk |
| 6-10 | Moderate complexity, still manageable |
| 11-20 | High complexity, hard to test fully |
| 21+ | Very high risk, almost certainly needs refactoring |

**High complexity example (CC = 12):**

```java
// OrderDiscountCalculator — CC = 12
public double calculateDiscount(Order order, Customer customer, LocalDate orderDate) {
    double discount = 0.0;

    // +1: if
    if (customer.getTier() == CustomerTier.PREMIUM) {
        discount += 0.15;
        // +1: if
        if (order.getTotalValue() > 500) {
            discount += 0.05; // loyalty bonus
        }
    // +1: else if
    } else if (customer.getTier() == CustomerTier.GOLD) {
        discount += 0.10;
    // +1: else if
    } else if (customer.getTier() == CustomerTier.SILVER) {
        discount += 0.05;
    }

    // +1: if
    if (order.getItemCount() >= 10) {
        discount += 0.03; // bulk discount
    // +1: else if
    } else if (order.getItemCount() >= 5) {
        discount += 0.01;
    }

    // +1: &&
    // +1: if
    if (orderDate.getMonth() == Month.DECEMBER && orderDate.getDayOfMonth() >= 20) {
        discount += 0.08; // holiday discount
    }

    // +1: if
    if (customer.getReferralCount() > 0) {
        // +1: ? :
        discount += customer.getReferralCount() > 5 ? 0.07 : 0.03;
    }

    // +1: if
    if (discount > 0.35) {
        discount = 0.35; // cap
    }

    return discount;
}
```

CC = 1 + 11 decision points = 12. This method has at least 12 independent paths, which means you need at least 12 tests to get full branch coverage. It is also nearly impossible to reason about. When the business adds a new tier, this method becomes even harder to change correctly.

**Refactored — CC reduced by extracting to polymorphism and strategies:**

```java
// CC = 2: each rule is tested in isolation
public double calculateDiscount(Order order, Customer customer, LocalDate orderDate) {
    List<DiscountRule> rules = List.of(
            new TierDiscountRule(),
            new BulkOrderDiscountRule(),
            new HolidayDiscountRule(),
            new ReferralDiscountRule()
    );

    double total = rules.stream()
            .mapToDouble(rule -> rule.discountFor(order, customer, orderDate))
            .sum();

    return Math.min(total, MAX_DISCOUNT_RATE);
}
```

```java
// TierDiscountRule — CC = 4
public class TierDiscountRule implements DiscountRule {

    @Override
    public double discountFor(Order order, Customer customer, LocalDate orderDate) {
        return switch (customer.getTier()) {
            case PREMIUM -> order.getTotalValue() > 500 ? 0.20 : 0.15;
            case GOLD -> 0.10;
            case SILVER -> 0.05;
            default -> 0.0;
        };
    }
}
```

```java
// BulkOrderDiscountRule — CC = 3
public class BulkOrderDiscountRule implements DiscountRule {

    @Override
    public double discountFor(Order order, Customer customer, LocalDate orderDate) {
        if (order.getItemCount() >= 10) return 0.03;
        if (order.getItemCount() >= 5) return 0.01;
        return 0.0;
    }
}
```

```java
// HolidayDiscountRule — CC = 2
public class HolidayDiscountRule implements DiscountRule {

    @Override
    public double discountFor(Order order, Customer customer, LocalDate orderDate) {
        boolean isHolidaySeason = orderDate.getMonth() == Month.DECEMBER
                && orderDate.getDayOfMonth() >= 20;
        return isHolidaySeason ? 0.08 : 0.0;
    }
}
```

```java
// ReferralDiscountRule — CC = 3
public class ReferralDiscountRule implements DiscountRule {

    @Override
    public double discountFor(Order order, Customer customer, LocalDate orderDate) {
        if (customer.getReferralCount() <= 0) return 0.0;
        return customer.getReferralCount() > 5 ? 0.07 : 0.03;
    }
}
```

Now each rule has CC between 2 and 4. Each one is trivial to test in isolation. The orchestrator method has CC of 2 and does not need to know about discount logic. When you add a new rule (say, a first-purchase discount), you add a new class and add it to the list — no existing code changes.

This is the Strategy pattern, and it is not just better design — it directly reduces cyclomatic complexity and makes the code fully testable.

---

## Summary

| Topic | Key Rule |
|---|---|
| Testing Pyramid | 70% unit / 20% integration / 10% E2E. If CI takes >10 minutes, you have the wrong ratio. |
| FIRST | Violations of Isolated and Repeatable create flaky tests, which are worse than no tests. |
| Naming | `method_scenario_expectedBehavior`. The test name is the first line of your incident report. |
| TDD | Red-Green-Refactor. Keep cycles short (2-5 min). Spike first if the design is unknown. |
| Test Doubles | Stub = returns data. Mock = verifiable. Fake = real logic. Spy = real object, intercepted. |
| Mockito | `@Mock`, `@InjectMocks`, `@Spy`, `@Captor`. Mock what you don't own. |
| Over-mocking | If you mock the system under test, you are testing Mockito, not your code. |
| Testability | Hidden `new`, `static`, and `LocalDate.now()` are the three most common testability killers. Inject them. |
| Clock | Always use `java.time.Clock` injected via constructor. Never `LocalDate.now()` inline. |
| Integration Tests | Use TestContainers for real database tests. Use `@WebMvcTest` for controller tests. |
| Coverage | 80% branch coverage is the floor, not the goal. 100% coverage with no assertions is useless. |
| Mutation Testing | PIT catches what coverage misses. A surviving mutant is a missing assertion. |
| Cyclomatic Complexity | CC > 10 is a smell. Extract rules into strategies. Test each strategy in isolation. |
