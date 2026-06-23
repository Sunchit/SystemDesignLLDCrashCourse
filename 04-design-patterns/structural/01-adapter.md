# Adapter Design Pattern

> **Category:** Structural  
> **Also Known As:** Wrapper  
> **GoF Classification:** Structural Pattern  
> **Complexity:** Low-Medium  
> **Usage Frequency:** Very High (one of the most commonly used patterns in enterprise Java)

---

## Table of Contents

1. [Intent and Problem It Solves](#1-intent-and-problem-it-solves)
2. [Real-World Analogy](#2-real-world-analogy)
3. [When to Use / When NOT to Use](#3-when-to-use--when-not-to-use)
4. [UML Class Diagrams](#4-uml-class-diagrams)
5. [Complete Java Implementations](#5-complete-java-implementations)
   - [Object Adapter: Payment Gateway](#51-object-adapter-payment-gateway-adapter)
   - [Object Adapter: Legacy XML to JSON](#52-object-adapter-legacy-xml-to-json-bridge)
   - [Class Adapter (Inheritance-Based)](#53-class-adapter-using-inheritance)
   - [Two-Way Adapter](#54-two-way-adapter)
6. [Real-World Java Ecosystem Examples](#6-real-world-java-ecosystem-examples)
7. [Object Adapter vs Class Adapter Comparison](#7-object-adapter-vs-class-adapter-comparison)
8. [Trade-offs (Pros and Cons)](#8-trade-offs-pros-and-cons)
9. [Common Pitfalls and Gotchas](#9-common-pitfalls-and-gotchas)
10. [Comparison With Related Patterns](#10-comparison-with-related-patterns)
11. [Common Interview Discussion Points](#11-common-interview-discussion-points)
12. [Interview Questions with Model Answers](#12-interview-questions-with-model-answers)

---

## 1. Intent and Problem It Solves

### The Core Problem: Interface Incompatibility

In enterprise software, interface incompatibility is inevitable. Systems evolve independently, third-party libraries impose their own API contracts, and legacy components cannot always be rewritten. The Adapter pattern exists specifically to solve this structural mismatch without modifying either the client or the adaptee.

**The GoF definition:**
> "Convert the interface of a class into another interface that clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces."

### Why This Problem Arises

Consider the following scenarios that every Java developer encounters:

1. **Third-party SDK integration**: You build your application against your own `PaymentProcessor` interface. Three months later, the business decides to integrate Stripe. Stripe's `StripeClient` has a completely different API — different method names, different parameter types, different return types. You cannot modify Stripe's SDK, and you do not want to scatter Stripe-specific calls throughout your business logic.

2. **Legacy system modernization**: An older system exposes data through XML-over-HTTP endpoints. A new microservice expects JSON. You cannot rewrite the legacy system, but you need the two to communicate.

3. **Vendor switching**: Your application currently uses PayPal. Management wants to add Stripe as an option without touching business logic.

4. **Library version upgrades**: A library changes its API in a major version. You wrap the new API in an adapter that still honors the old interface, allowing gradual migration.

### What the Adapter Does

The Adapter sits between the **client** (which calls a target interface) and the **adaptee** (the incompatible class). It:

- Implements the **Target interface** that the client expects
- Holds a reference to (or inherits from) the **Adaptee**
- **Translates** each target method call into the appropriate adaptee call, performing any necessary data conversion, format transformation, or parameter mapping

```
Client --> [Target Interface] --> [Adapter] --> [Adaptee]
```

The client never knows an adapter is involved. The adaptee never knows it is being wrapped. The adapter carries the full translation burden.

### Structural Pattern, Not Behavioral

It is important to understand that Adapter is a **structural** pattern. It is concerned with how objects and classes are composed to form larger structures. It does not add new behavior — it translates existing behavior across interface boundaries. This distinguishes it from patterns like Decorator (which adds behavior) or Proxy (which controls access).

---

## 2. Real-World Analogy

### The Power Plug Adapter

Imagine you are a software engineer traveling from India to the United Kingdom. Your laptop charger has a Type-D plug (round pins). UK power sockets use Type-G plugs (three rectangular pins). The electricity is the same (more or less), the intent is the same (charge the laptop), but the physical interface is incompatible.

You buy a power plug adapter:
- It **accepts** your Type-D plug on one end (the adaptee interface)
- It **exposes** a Type-G plug on the other end (the target interface)
- It does **not change** what the electricity does — it merely makes the connection possible

In code:
- Your **laptop charger** = the Client (has fixed expectations)
- The **UK socket** = the Target interface (what the environment provides)
- The **Indian plug** = the Adaptee (the thing you actually have)
- The **adapter** = the class that bridges them

Critically, neither the socket nor the charger needs to be modified. The adapter absorbs the entire translation burden. This is the essence of the Adapter pattern.

---

## 3. When to Use / When NOT to Use

### When to Use

| Scenario | Rationale |
|---|---|
| Integrating a third-party library whose API does not match your internal interface | Avoids scattering vendor-specific code across the codebase; centralizes the translation |
| Using a legacy component in a new system | Allows existing tested code to be reused without modification |
| Making independent or separately developed classes work together | Decouples client from concrete implementation details |
| Providing a stable API when underlying implementations may change | Swap adaptees without touching client code |
| Testing: wrapping an external dependency to enable mocking | The adapter implements your interface, making it trivially injectable |
| Migrating from one library/API version to another gradually | Old and new can coexist behind a unified interface |

### When NOT to Use

**Do not use Adapter when:**

1. **You control both interfaces**: If you own both the client and the adaptee, redesign one of them to match. An adapter is a workaround for incompatibility you cannot eliminate at the source.

2. **The translation is impossibly complex**: If adapting A to B requires significant business logic, data enrichment from other sources, or complex orchestration, you likely need a more structured solution (Facade, Service, or Anti-Corruption Layer in DDD terms).

3. **Performance is critical and the overhead matters**: Each adapter call adds an indirection layer. In tight inner loops (e.g., game engines, high-frequency trading), this overhead may be unacceptable. Profile before applying.

4. **You are adapting many incompatible classes with no commonality**: If you need to adapt 20 unrelated classes, consider whether a Facade or a redesign of your domain model is more appropriate.

5. **It would be simpler to just rewrite the adaptee**: For small classes where rewriting is cheap and carries no risk, just rewrite it to match the target interface.

6. **You are trying to add behavior, not just translate interfaces**: Use Decorator instead.

---

## 4. UML Class Diagrams

### 4.1 Object Adapter (Composition-Based)

The Object Adapter holds a reference to the Adaptee and delegates calls to it. This is the preferred approach in Java because Java does not support multiple class inheritance.

```
+------------------+         +-------------------+
|   <<interface>>  |         |                   |
|   Target         |         |    Adaptee        |
+------------------+         +-------------------+
| + request()      |         | + specificRequest()|
+------------------+         +-------------------+
        ^                              ^
        |                             / (has-a)
        |                            /
+------------------+               /
|    Adapter       |---(wraps)----/
+------------------+
| - adaptee: Adaptee|
+------------------+
| + request()      |  <-- calls adaptee.specificRequest()
+------------------+
        ^
        |
+------------------+
|    Client        |
+------------------+
| + doWork(Target) |
+------------------+

Relationships:
  Client        ---uses--->   Target (interface)
  Adapter       ---implements-->  Target (interface)
  Adapter       ---has-a--->  Adaptee (composition)
  Client        ---never knows about-->  Adaptee
```

**Sequence Flow:**
```
Client          Adapter         Adaptee
  |                |               |
  |-- request() -->|               |
  |                |--specificReq()-->|
  |                |<-- result ----|
  |<-- result -----|               |
```

### 4.2 Class Adapter (Inheritance-Based)

The Class Adapter inherits from the Adaptee and implements the Target interface. In Java, since you cannot extend two classes, the adaptee must be a class and the target must be an interface.

```
+------------------+         +-------------------+
|   <<interface>>  |         |    Adaptee        |
|   Target         |         |   (concrete class)|
+------------------+         +-------------------+
| + request()      |         | + specificRequest()|
+------------------+         +-------------------+
        ^                              ^
        |                             / (extends)
        |                            /
+----------------------------------+
|    ClassAdapter                  |
|  (implements Target,             |
|   extends Adaptee)               |
+----------------------------------+
| + request()  <-- overrides/calls |
|                 specificRequest()|
+----------------------------------+
        ^
        |
+------------------+
|    Client        |
+------------------+
| + doWork(Target) |
+------------------+

Relationships:
  ClassAdapter  ---implements-->  Target (interface)
  ClassAdapter  ---extends--->   Adaptee (class)
  Client        ---uses--->      Target (interface)
```

### 4.3 Two-Way Adapter

```
+------------------+         +-------------------+
|   <<interface>>  |         |  <<interface>>    |
|   InterfaceA     |         |  InterfaceB       |
+------------------+         +-------------------+
| + operationA()   |         | + operationB()    |
+------------------+         +-------------------+
         ^                              ^
         |                             |
+--------------------------------------+
|         TwoWayAdapter                |
|  (implements InterfaceA & InterfaceB)|
+--------------------------------------+
| - componentA : ConcreteA             |
| - componentB : ConcreteB             |
+--------------------------------------+
| + operationA() --> delegates to B    |
| + operationB() --> delegates to A    |
+--------------------------------------+
```

---

## 5. Complete Java Implementations

### 5.1 Object Adapter: Payment Gateway Adapter

This is the canonical enterprise use case. Your application defines a `PaymentProcessor` interface. You need to integrate two third-party SDKs — one for Stripe, one for PayPal — without leaking their APIs into your business logic.

```java
package com.example.payments;

import java.math.BigDecimal;
import java.util.Currency;
import java.util.Objects;
import java.util.UUID;
import java.util.logging.Logger;

// ============================================================
// 1. TARGET INTERFACE — defined by YOUR application
//    This is the contract your business logic depends on.
// ============================================================

/**
 * The internal payment processing contract that all payment providers
 * must satisfy. Business logic depends ONLY on this interface.
 *
 * <p>This interface represents the "Target" in the Adapter pattern.
 */
public interface PaymentProcessor {

    /**
     * Charges a payment method for the specified amount.
     *
     * @param paymentMethodToken an opaque token representing the payment method
     * @param amount             the amount to charge (must be positive)
     * @param currency           the ISO 4217 currency code
     * @return a {@link PaymentResult} describing the outcome
     * @throws PaymentException if the charge attempt fails unrecoverably
     */
    PaymentResult charge(String paymentMethodToken, BigDecimal amount, Currency currency)
            throws PaymentException;

    /**
     * Refunds a previously completed charge.
     *
     * @param transactionId the ID returned by a successful {@link #charge} call
     * @param amount        the amount to refund; must not exceed the original charge
     * @return a {@link PaymentResult} describing the outcome
     * @throws PaymentException if the refund fails
     */
    PaymentResult refund(String transactionId, BigDecimal amount) throws PaymentException;

    /**
     * Returns the human-readable name of this payment processor.
     *
     * @return provider name
     */
    String getProviderName();
}
```

```java
package com.example.payments;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.Objects;

/**
 * Represents the outcome of a payment operation.
 * Immutable value object.
 */
public final class PaymentResult {

    public enum Status { SUCCESS, DECLINED, ERROR }

    private final String transactionId;
    private final Status status;
    private final String providerMessage;
    private final BigDecimal amount;
    private final Instant timestamp;

    private PaymentResult(Builder builder) {
        this.transactionId   = Objects.requireNonNull(builder.transactionId);
        this.status          = Objects.requireNonNull(builder.status);
        this.providerMessage = builder.providerMessage;
        this.amount          = builder.amount;
        this.timestamp       = Instant.now();
    }

    public String getTransactionId()   { return transactionId; }
    public Status getStatus()          { return status; }
    public String getProviderMessage() { return providerMessage; }
    public BigDecimal getAmount()      { return amount; }
    public Instant getTimestamp()      { return timestamp; }

    public boolean isSuccessful() {
        return status == Status.SUCCESS;
    }

    @Override
    public String toString() {
        return "PaymentResult{txId=" + transactionId
                + ", status=" + status
                + ", msg=" + providerMessage + "}";
    }

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private String transactionId;
        private Status status;
        private String providerMessage;
        private BigDecimal amount;

        public Builder transactionId(String id)   { this.transactionId = id; return this; }
        public Builder status(Status s)           { this.status = s; return this; }
        public Builder providerMessage(String m)  { this.providerMessage = m; return this; }
        public Builder amount(BigDecimal a)       { this.amount = a; return this; }
        public PaymentResult build()              { return new PaymentResult(this); }
    }
}
```

```java
package com.example.payments;

/**
 * Checked exception representing an unrecoverable payment failure.
 */
public class PaymentException extends Exception {

    private final String providerErrorCode;

    public PaymentException(String message, String providerErrorCode) {
        super(message);
        this.providerErrorCode = providerErrorCode;
    }

    public PaymentException(String message, String providerErrorCode, Throwable cause) {
        super(message, cause);
        this.providerErrorCode = providerErrorCode;
    }

    public String getProviderErrorCode() {
        return providerErrorCode;
    }
}
```

```java
package com.example.payments.thirdparty.stripe;

import java.math.BigDecimal;

// ============================================================
// 2. ADAPTEE — Stripe's SDK (third-party, cannot be modified)
//    This simulates what Stripe's actual client looks like.
//    Different method names, different parameter structure.
// ============================================================

/**
 * Simulates the Stripe Java SDK client.
 * This class is provided by Stripe — we cannot change its API.
 *
 * <p>Note: In reality this would be {@code com.stripe.Stripe} /
 * {@code com.stripe.model.Charge}, etc. We simulate it here
 * to keep the example self-contained.
 */
public class StripeClient {

    private final String apiKey;

    public StripeClient(String apiKey) {
        if (apiKey == null || apiKey.isBlank()) {
            throw new IllegalArgumentException("Stripe API key must not be blank");
        }
        this.apiKey = apiKey;
    }

    /**
     * Creates a Stripe charge.
     *
     * @param amountInCents amount in the smallest currency unit (e.g. cents for USD)
     * @param currencyCode  lowercase ISO 4217 code ("usd", "eur")
     * @param source        the payment source token (e.g. "tok_visa")
     * @return a {@link StripeCharge} representing the created charge
     * @throws StripeException on API or network errors
     */
    public StripeCharge createCharge(long amountInCents,
                                     String currencyCode,
                                     String source) throws StripeException {
        // Simulated Stripe API call
        System.out.println("[Stripe SDK] createCharge called: "
                + amountInCents + " " + currencyCode + " source=" + source);

        if (source.equals("tok_declined")) {
            throw new StripeException("card_declined", "Your card was declined.");
        }

        return new StripeCharge("ch_" + System.currentTimeMillis(), amountInCents,
                currencyCode, "succeeded");
    }

    /**
     * Creates a refund for an existing charge.
     *
     * @param chargeId       the Stripe charge ID (starts with "ch_")
     * @param amountInCents  amount to refund in cents; null means full refund
     * @return a {@link StripeRefund} describing the refund
     * @throws StripeException on API or network errors
     */
    public StripeRefund createRefund(String chargeId,
                                     Long amountInCents) throws StripeException {
        System.out.println("[Stripe SDK] createRefund: chargeId=" + chargeId
                + " amount=" + amountInCents);
        return new StripeRefund("re_" + System.currentTimeMillis(), chargeId,
                amountInCents, "succeeded");
    }

    // --------------------------------------------------------
    // Stripe-specific domain objects
    // --------------------------------------------------------

    public static final class StripeCharge {
        private final String id;
        private final long amountInCents;
        private final String currency;
        private final String status;

        public StripeCharge(String id, long amountInCents, String currency, String status) {
            this.id = id; this.amountInCents = amountInCents;
            this.currency = currency; this.status = status;
        }

        public String getId()           { return id; }
        public long getAmountInCents()  { return amountInCents; }
        public String getCurrency()     { return currency; }
        public String getStatus()       { return status; }
    }

    public static final class StripeRefund {
        private final String id;
        private final String chargeId;
        private final long amountInCents;
        private final String status;

        public StripeRefund(String id, String chargeId, long amountInCents, String status) {
            this.id = id; this.chargeId = chargeId;
            this.amountInCents = amountInCents; this.status = status;
        }

        public String getId()          { return id; }
        public String getChargeId()    { return chargeId; }
        public long getAmountInCents() { return amountInCents; }
        public String getStatus()      { return status; }
    }

    public static final class StripeException extends Exception {
        private final String stripeCode;
        public StripeException(String stripeCode, String message) {
            super(message);
            this.stripeCode = stripeCode;
        }
        public String getStripeCode() { return stripeCode; }
    }
}
```

```java
package com.example.payments.adapters;

import com.example.payments.PaymentException;
import com.example.payments.PaymentProcessor;
import com.example.payments.PaymentResult;
import com.example.payments.thirdparty.stripe.StripeClient;
import com.example.payments.thirdparty.stripe.StripeClient.StripeCharge;
import com.example.payments.thirdparty.stripe.StripeClient.StripeException;
import com.example.payments.thirdparty.stripe.StripeClient.StripeRefund;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Currency;
import java.util.Objects;
import java.util.logging.Logger;

// ============================================================
// 3. OBJECT ADAPTER — bridges Stripe SDK to PaymentProcessor
//    Uses COMPOSITION: holds a reference to StripeClient.
// ============================================================

/**
 * Adapts the Stripe SDK ({@link StripeClient}) to the application's
 * {@link PaymentProcessor} interface using the Object Adapter pattern.
 *
 * <p><strong>Translation responsibilities:</strong>
 * <ul>
 *   <li>BigDecimal amounts to Stripe's long (cents)</li>
 *   <li>java.util.Currency to Stripe's lowercase currency string</li>
 *   <li>{@link StripeException} to {@link PaymentException}</li>
 *   <li>Stripe response objects to {@link PaymentResult}</li>
 * </ul>
 *
 * <p>The client ({@code CheckoutService}, {@code SubscriptionService}, etc.)
 * depends only on {@link PaymentProcessor} and is completely unaware that
 * Stripe is involved.
 */
public final class StripePaymentAdapter implements PaymentProcessor {

    private static final Logger log = Logger.getLogger(StripePaymentAdapter.class.getName());

    private final StripeClient stripeClient;

    /**
     * Constructs an adapter wrapping the given Stripe client.
     *
     * @param stripeClient a configured, authenticated Stripe client; must not be null
     */
    public StripePaymentAdapter(StripeClient stripeClient) {
        this.stripeClient = Objects.requireNonNull(stripeClient, "stripeClient must not be null");
    }

    /**
     * {@inheritDoc}
     *
     * <p>Translates to {@link StripeClient#createCharge(long, String, String)}.
     * Converts the {@code amount} to cents (long) and the {@link Currency}
     * to a lowercase ISO code as required by the Stripe API.
     */
    @Override
    public PaymentResult charge(String paymentMethodToken,
                                BigDecimal amount,
                                Currency currency) throws PaymentException {

        Objects.requireNonNull(paymentMethodToken, "paymentMethodToken must not be null");
        Objects.requireNonNull(amount, "amount must not be null");
        Objects.requireNonNull(currency, "currency must not be null");

        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new PaymentException("Charge amount must be positive", "INVALID_AMOUNT");
        }

        // --- TRANSLATION LAYER ---
        // Stripe requires amount in the smallest currency unit (cents for USD)
        long amountInCents = amount
                .setScale(2, RoundingMode.HALF_UP)
                .multiply(BigDecimal.valueOf(100))
                .longValueExact();
        String currencyCode = currency.getCurrencyCode().toLowerCase();

        log.fine(() -> "Delegating charge to Stripe: " + amountInCents + " " + currencyCode);

        try {
            StripeCharge charge = stripeClient.createCharge(amountInCents, currencyCode,
                    paymentMethodToken);

            return PaymentResult.builder()
                    .transactionId(charge.getId())
                    .status("succeeded".equals(charge.getStatus())
                            ? PaymentResult.Status.SUCCESS
                            : PaymentResult.Status.DECLINED)
                    .amount(amount)
                    .providerMessage("Stripe charge: " + charge.getStatus())
                    .build();

        } catch (StripeException e) {
            // Translate Stripe-specific exception to our domain exception
            if ("card_declined".equals(e.getStripeCode())) {
                return PaymentResult.builder()
                        .transactionId("DECLINED")
                        .status(PaymentResult.Status.DECLINED)
                        .amount(amount)
                        .providerMessage(e.getMessage())
                        .build();
            }
            throw new PaymentException(
                    "Stripe charge failed: " + e.getMessage(),
                    e.getStripeCode(),
                    e);
        }
    }

    /**
     * {@inheritDoc}
     *
     * <p>Translates to {@link StripeClient#createRefund(String, Long)}.
     */
    @Override
    public PaymentResult refund(String transactionId, BigDecimal amount) throws PaymentException {
        Objects.requireNonNull(transactionId, "transactionId must not be null");
        Objects.requireNonNull(amount, "amount must not be null");

        long amountInCents = amount
                .setScale(2, RoundingMode.HALF_UP)
                .multiply(BigDecimal.valueOf(100))
                .longValueExact();

        try {
            StripeRefund refund = stripeClient.createRefund(transactionId, amountInCents);

            return PaymentResult.builder()
                    .transactionId(refund.getId())
                    .status("succeeded".equals(refund.getStatus())
                            ? PaymentResult.Status.SUCCESS
                            : PaymentResult.Status.ERROR)
                    .amount(amount)
                    .providerMessage("Stripe refund: " + refund.getStatus())
                    .build();

        } catch (StripeException e) {
            throw new PaymentException(
                    "Stripe refund failed: " + e.getMessage(),
                    e.getStripeCode(),
                    e);
        }
    }

    @Override
    public String getProviderName() {
        return "Stripe";
    }
}
```

```java
package com.example.payments;

import com.example.payments.adapters.StripePaymentAdapter;
import com.example.payments.thirdparty.stripe.StripeClient;

import java.math.BigDecimal;
import java.util.Currency;

// ============================================================
// 4. CLIENT — uses PaymentProcessor; knows nothing of Stripe
// ============================================================

/**
 * The checkout service that orchestrates the payment flow.
 * It depends ONLY on {@link PaymentProcessor}.
 *
 * <p>You can swap Stripe for PayPal, Braintree, or a mock by
 * injecting a different adapter — no changes needed here.
 */
public class CheckoutService {

    private final PaymentProcessor paymentProcessor;

    /** Constructor injection — the preferred approach for testability. */
    public CheckoutService(PaymentProcessor paymentProcessor) {
        this.paymentProcessor = paymentProcessor;
    }

    public void processOrder(String paymentToken, BigDecimal totalAmount) {
        Currency usd = Currency.getInstance("USD");
        try {
            PaymentResult result = paymentProcessor.charge(paymentToken, totalAmount, usd);
            if (result.isSuccessful()) {
                System.out.println("Order confirmed. Transaction ID: "
                        + result.getTransactionId());
            } else {
                System.out.println("Payment declined: " + result.getProviderMessage());
            }
        } catch (PaymentException e) {
            System.err.println("Payment error [" + e.getProviderErrorCode()
                    + "]: " + e.getMessage());
        }
    }

    // --- DEMO ---
    public static void main(String[] args) {
        // Wire up: inject the adapter (could be done by a DI container)
        StripeClient stripeClient = new StripeClient("sk_test_demo123");
        PaymentProcessor processor = new StripePaymentAdapter(stripeClient);
        CheckoutService service = new CheckoutService(processor);

        System.out.println("=== Successful charge ===");
        service.processOrder("tok_visa", new BigDecimal("49.99"));

        System.out.println("\n=== Declined card ===");
        service.processOrder("tok_declined", new BigDecimal("49.99"));
    }
}
```

---

### 5.2 Object Adapter: Legacy XML to JSON Bridge

This example shows how to adapt a legacy system that speaks XML to a modern service that expects JSON — a common scenario in enterprise modernization projects.

```java
package com.example.legacy;

// ============================================================
// TARGET INTERFACE — what the new system expects
// ============================================================

/**
 * Modern data retrieval contract. Consumers expect JSON strings.
 */
public interface CustomerDataService {

    /**
     * Retrieves customer details as a JSON string.
     *
     * @param customerId the customer identifier
     * @return a JSON string representing the customer, or null if not found
     */
    String getCustomerAsJson(String customerId);

    /**
     * Retrieves a customer's order history as a JSON array string.
     *
     * @param customerId the customer identifier
     * @return a JSON array string of orders
     */
    String getOrderHistoryAsJson(String customerId);
}
```

```java
package com.example.legacy;

// ============================================================
// ADAPTEE — the legacy XML-based system we cannot modify
// ============================================================

/**
 * Legacy customer service that communicates via XML.
 * This is the existing system that cannot be rewritten.
 *
 * <p>In reality, this might be a SOAP service, an old EJB, or
 * any system whose interface we cannot control.
 */
public class LegacyXmlCustomerService {

    /**
     * Returns customer data as an XML string.
     *
     * @param custId the customer ID
     * @return XML-formatted customer data
     */
    public String fetchCustomerXml(String custId) {
        // Simulated legacy XML response
        return "<?xml version=\"1.0\" encoding=\"UTF-8\"?>"
                + "<customer>"
                + "<id>" + custId + "</id>"
                + "<firstName>Jane</firstName>"
                + "<lastName>Doe</lastName>"
                + "<email>jane.doe@example.com</email>"
                + "<tier>GOLD</tier>"
                + "</customer>";
    }

    /**
     * Returns order history as an XML string.
     *
     * @param custId the customer ID
     * @return XML-formatted order list
     */
    public String fetchOrdersXml(String custId) {
        return "<?xml version=\"1.0\" encoding=\"UTF-8\"?>"
                + "<orders>"
                + "<order><id>ORD-001</id><amount>120.50</amount><status>DELIVERED</status></order>"
                + "<order><id>ORD-002</id><amount>45.00</amount><status>PROCESSING</status></order>"
                + "</orders>";
    }
}
```

```java
package com.example.legacy;

import org.w3c.dom.Document;
import org.w3c.dom.Node;
import org.w3c.dom.NodeList;
import org.xml.sax.InputSource;

import javax.xml.parsers.DocumentBuilder;
import javax.xml.parsers.DocumentBuilderFactory;
import java.io.StringReader;
import java.util.Objects;
import java.util.logging.Level;
import java.util.logging.Logger;

// ============================================================
// OBJECT ADAPTER — bridges legacy XML service to JSON interface
// ============================================================

/**
 * Adapts the {@link LegacyXmlCustomerService} to the
 * {@link CustomerDataService} interface by converting XML responses
 * to JSON on the fly.
 *
 * <p>This is a classic enterprise integration scenario: the legacy
 * system is expensive or risky to rewrite, so we wrap it.
 *
 * <p><strong>Translation responsibilities:</strong>
 * <ul>
 *   <li>XML string parsing via DOM</li>
 *   <li>Serialization to JSON string</li>
 *   <li>Error normalization (XML parse errors become meaningful exceptions)</li>
 * </ul>
 *
 * <p>Note: In production code, use a proper JSON library (Jackson, Gson)
 * instead of manual string building. The manual approach is used here
 * for zero-dependency compilation.
 */
public final class XmlToJsonCustomerAdapter implements CustomerDataService {

    private static final Logger log =
            Logger.getLogger(XmlToJsonCustomerAdapter.class.getName());

    private final LegacyXmlCustomerService legacyService;
    private final DocumentBuilderFactory xmlFactory;

    public XmlToJsonCustomerAdapter(LegacyXmlCustomerService legacyService) {
        this.legacyService = Objects.requireNonNull(legacyService);
        this.xmlFactory = DocumentBuilderFactory.newInstance();
    }

    @Override
    public String getCustomerAsJson(String customerId) {
        Objects.requireNonNull(customerId, "customerId must not be null");

        String xml = legacyService.fetchCustomerXml(customerId);
        try {
            Document doc = parseXml(xml);
            String id        = getTextContent(doc, "id");
            String firstName = getTextContent(doc, "firstName");
            String lastName  = getTextContent(doc, "lastName");
            String email     = getTextContent(doc, "email");
            String tier      = getTextContent(doc, "tier");

            // Manual JSON construction for zero-dependency demo.
            // In production: use Jackson ObjectMapper or Gson.
            return "{"
                    + "\"id\":\"" + escape(id) + "\","
                    + "\"firstName\":\"" + escape(firstName) + "\","
                    + "\"lastName\":\"" + escape(lastName) + "\","
                    + "\"email\":\"" + escape(email) + "\","
                    + "\"tier\":\"" + escape(tier) + "\""
                    + "}";

        } catch (Exception e) {
            log.log(Level.SEVERE, "Failed to parse customer XML for id=" + customerId, e);
            throw new LegacyIntegrationException(
                    "Failed to retrieve customer " + customerId, e);
        }
    }

    @Override
    public String getOrderHistoryAsJson(String customerId) {
        Objects.requireNonNull(customerId, "customerId must not be null");

        String xml = legacyService.fetchOrdersXml(customerId);
        try {
            Document doc = parseXml(xml);
            NodeList orders = doc.getElementsByTagName("order");
            StringBuilder json = new StringBuilder("[");

            for (int i = 0; i < orders.getLength(); i++) {
                if (i > 0) json.append(",");
                Node order = orders.item(i);
                Document orderDoc = xmlFactory.newDocumentBuilder().newDocument();
                Node imported = orderDoc.importNode(order, true);
                orderDoc.appendChild(imported);

                String id     = getTextContent(orderDoc, "id");
                String amount = getTextContent(orderDoc, "amount");
                String status = getTextContent(orderDoc, "status");

                json.append("{")
                        .append("\"id\":\"").append(escape(id)).append("\",")
                        .append("\"amount\":").append(amount).append(",")
                        .append("\"status\":\"").append(escape(status)).append("\"")
                        .append("}");
            }
            json.append("]");
            return json.toString();

        } catch (Exception e) {
            log.log(Level.SEVERE, "Failed to parse orders XML for id=" + customerId, e);
            throw new LegacyIntegrationException(
                    "Failed to retrieve orders for customer " + customerId, e);
        }
    }

    // -----------------------------------------------
    // Private helpers
    // -----------------------------------------------

    private Document parseXml(String xml) throws Exception {
        DocumentBuilder builder = xmlFactory.newDocumentBuilder();
        return builder.parse(new InputSource(new StringReader(xml)));
    }

    private String getTextContent(Document doc, String tagName) {
        NodeList nodes = doc.getElementsByTagName(tagName);
        if (nodes.getLength() == 0) return "";
        return nodes.item(0).getTextContent().trim();
    }

    /** Minimal JSON string escaping for demo purposes. */
    private String escape(String value) {
        if (value == null) return "";
        return value.replace("\\", "\\\\")
                    .replace("\"", "\\\"")
                    .replace("\n", "\\n")
                    .replace("\r", "\\r");
    }

    // -----------------------------------------------
    // Nested runtime exception for integration errors
    // -----------------------------------------------

    public static class LegacyIntegrationException extends RuntimeException {
        public LegacyIntegrationException(String message, Throwable cause) {
            super(message, cause);
        }
    }
}
```

```java
package com.example.legacy;

/**
 * Demonstrates the XML-to-JSON adapter in action.
 * The client (ModernReportingService) only knows about CustomerDataService.
 */
public class ModernReportingService {

    private final CustomerDataService customerDataService;

    public ModernReportingService(CustomerDataService customerDataService) {
        this.customerDataService = customerDataService;
    }

    public void generateCustomerReport(String customerId) {
        String customerJson = customerDataService.getCustomerAsJson(customerId);
        String ordersJson   = customerDataService.getOrderHistoryAsJson(customerId);

        System.out.println("Customer: " + customerJson);
        System.out.println("Orders:   " + ordersJson);
    }

    public static void main(String[] args) {
        LegacyXmlCustomerService legacyService = new LegacyXmlCustomerService();
        CustomerDataService adapter = new XmlToJsonCustomerAdapter(legacyService);

        ModernReportingService reporting = new ModernReportingService(adapter);
        reporting.generateCustomerReport("CUST-42");
    }
}
```

---

### 5.3 Class Adapter (Using Inheritance)

In Java, Class Adapter requires the adaptee to be a concrete class. The adapter **extends** the adaptee and **implements** the target interface. Because Java has single class inheritance, this only works when the adaptee is the sole superclass you need.

```java
package com.example.classadapter;

// ============================================================
// TARGET INTERFACE
// ============================================================

/**
 * Modern logging interface expected by the application.
 */
public interface StructuredLogger {

    void logInfo(String component, String message);

    void logError(String component, String message, Throwable error);

    void logDebug(String component, String message);
}
```

```java
package com.example.classadapter;

// ============================================================
// ADAPTEE (concrete class we cannot modify)
// ============================================================

/**
 * A legacy logger that only supports unstructured printf-style output.
 * Imagine this is from a vendor library — it cannot be changed.
 */
public class LegacyPrintLogger {

    private final String logLevel;

    public LegacyPrintLogger(String logLevel) {
        this.logLevel = logLevel;
    }

    /**
     * Writes a formatted log entry.
     *
     * @param level   severity level string ("INFO", "ERROR", "DEBUG")
     * @param message the raw log message
     */
    public void log(String level, String message) {
        System.out.printf("[%s] [%s] %s%n", level, logLevel, message);
    }

    /**
     * Writes an error log entry with a stack trace.
     *
     * @param message the error message
     * @param error   the associated throwable
     */
    public void logException(String message, Throwable error) {
        System.err.printf("[ERROR] %s: %s%n", message, error.getMessage());
        error.printStackTrace(System.err);
    }
}
```

```java
package com.example.classadapter;

// ============================================================
// CLASS ADAPTER — inherits from Adaptee, implements Target
//
// IMPORTANT Java note: Because Java does not allow extending two
// classes, the Class Adapter pattern in Java requires:
//   (1) The Target is an INTERFACE (not an abstract class)
//   (2) The Adaptee is the sole CLASS being extended
//
// This is a structural constraint, not a design deficiency.
// ============================================================

/**
 * Class Adapter that extends {@link LegacyPrintLogger} and
 * implements the {@link StructuredLogger} interface.
 *
 * <p>By extending the adaptee, this adapter:
 * <ul>
 *   <li>Inherits all of {@link LegacyPrintLogger}'s behaviour</li>
 *   <li>Can override individual methods when needed</li>
 *   <li>Cannot be used to adapt a different LegacyPrintLogger subclass
 *       (a limitation of class adapters — the binding is static)</li>
 * </ul>
 *
 * <p>Compare with {@code StripePaymentAdapter} (Object Adapter):
 * the Object Adapter uses composition and can wrap any instance at
 * runtime; the Class Adapter uses inheritance and is bound at
 * compile time.
 */
public final class LegacyLoggerClassAdapter
        extends LegacyPrintLogger   // inherits from Adaptee
        implements StructuredLogger  // fulfills Target contract
{

    /**
     * Constructs a class adapter with a specified root log level.
     *
     * @param rootLevel the root level label for all entries (e.g. "APPLICATION")
     */
    public LegacyLoggerClassAdapter(String rootLevel) {
        super(rootLevel);
    }

    @Override
    public void logInfo(String component, String message) {
        // Translate StructuredLogger call → LegacyPrintLogger call
        log("INFO", "[" + component + "] " + message);
    }

    @Override
    public void logError(String component, String message, Throwable error) {
        String formatted = "[" + component + "] " + message;
        if (error != null) {
            logException(formatted, error);  // inherited from LegacyPrintLogger
        } else {
            log("ERROR", formatted);
        }
    }

    @Override
    public void logDebug(String component, String message) {
        log("DEBUG", "[" + component + "] " + message);
    }

    // Demo
    public static void main(String[] args) {
        // The client uses StructuredLogger — has no idea it is
        // backed by LegacyPrintLogger.
        StructuredLogger logger = new LegacyLoggerClassAdapter("ROOT");

        logger.logInfo("OrderService", "Order #42 created successfully");
        logger.logDebug("OrderService", "Computed tax: 8.5%");
        logger.logError("OrderService", "Failed to send confirmation email",
                new RuntimeException("SMTP timeout"));
    }
}
```

---

### 5.4 Two-Way Adapter

A Two-Way Adapter implements two target interfaces, allowing a single object to be used as either type. This is useful when two subsystems must interoperate and each has its own interface.

```java
package com.example.twoway;

// ============================================================
// TWO INTERFACES that need to interoperate
// ============================================================

/**
 * Interface used by the new notification system.
 */
public interface NotificationSender {
    /**
     * Sends a notification to a recipient.
     *
     * @param recipient  the destination (email, phone, etc.)
     * @param subject    a short subject line
     * @param body       the message body
     */
    void sendNotification(String recipient, String subject, String body);

    String getSenderType();
}
```

```java
package com.example.twoway;

/**
 * Interface used by the legacy alerting system.
 */
public interface AlertDispatcher {
    /**
     * Dispatches an alert.
     *
     * @param destination the alert destination
     * @param severity    alert severity: "LOW", "MEDIUM", "HIGH", "CRITICAL"
     * @param details     the alert details text
     */
    void dispatch(String destination, String severity, String details);

    boolean isAvailable();
}
```

```java
package com.example.twoway;

/**
 * Concrete notification sender using email.
 * Represents the "new" system implementation.
 */
public class EmailNotificationSender implements NotificationSender {

    private final String senderAddress;

    public EmailNotificationSender(String senderAddress) {
        this.senderAddress = senderAddress;
    }

    @Override
    public void sendNotification(String recipient, String subject, String body) {
        System.out.printf("[EMAIL] From=%s To=%s Subject='%s' Body='%s'%n",
                senderAddress, recipient, subject, body);
    }

    @Override
    public String getSenderType() {
        return "EMAIL";
    }
}
```

```java
package com.example.twoway;

import java.util.Objects;

// ============================================================
// TWO-WAY ADAPTER
// Implements BOTH interfaces, bridging the two systems.
// ============================================================

/**
 * Two-way adapter that allows a {@link NotificationSender} to be used
 * as an {@link AlertDispatcher}, and an {@link AlertDispatcher} to be
 * used as a {@link NotificationSender}.
 *
 * <p>This is particularly useful when:
 * <ul>
 *   <li>Two systems must interoperate and each expects the other's interface</li>
 *   <li>A gradual migration is underway and both APIs must coexist</li>
 *   <li>A component serves dual roles in different subsystems</li>
 * </ul>
 *
 * <p>The two-way adapter wraps a {@link NotificationSender} as primary and
 * implements {@link AlertDispatcher} by translating alert concepts to
 * notification concepts, and vice versa.
 */
public final class NotificationAlertTwoWayAdapter
        implements NotificationSender, AlertDispatcher {

    private final NotificationSender notificationSender;

    /**
     * Creates a two-way adapter backed by the given notification sender.
     *
     * @param notificationSender the underlying notification sender; must not be null
     */
    public NotificationAlertTwoWayAdapter(NotificationSender notificationSender) {
        this.notificationSender = Objects.requireNonNull(notificationSender);
    }

    // -------------------------------------------------------
    // Implementing NotificationSender — delegates directly
    // -------------------------------------------------------

    @Override
    public void sendNotification(String recipient, String subject, String body) {
        notificationSender.sendNotification(recipient, subject, body);
    }

    @Override
    public String getSenderType() {
        return notificationSender.getSenderType() + "+ALERT_DISPATCHER";
    }

    // -------------------------------------------------------
    // Implementing AlertDispatcher — translates to notification
    // -------------------------------------------------------

    @Override
    public void dispatch(String destination, String severity, String details) {
        // TRANSLATION: alert concepts → notification concepts
        String subject = "[" + severity.toUpperCase() + " ALERT] System Notification";
        String body    = "Severity: " + severity + "\nDetails: " + details;
        notificationSender.sendNotification(destination, subject, body);
    }

    @Override
    public boolean isAvailable() {
        // Could perform a health check; here we return true for simplicity
        return true;
    }

    // Demo
    public static void main(String[] args) {
        NotificationSender emailSender = new EmailNotificationSender("alerts@example.com");
        NotificationAlertTwoWayAdapter adapter = new NotificationAlertTwoWayAdapter(emailSender);

        System.out.println("=== Used as NotificationSender ===");
        NotificationSender asNotifier = adapter;
        asNotifier.sendNotification("user@example.com", "Order Shipped", "Your order is on the way!");

        System.out.println("\n=== Used as AlertDispatcher ===");
        AlertDispatcher asDispatcher = adapter;
        asDispatcher.dispatch("oncall@example.com", "CRITICAL", "Database CPU at 99%");
        System.out.println("Dispatcher available: " + asDispatcher.isAvailable());
    }
}
```

---

## 6. Real-World Java Ecosystem Examples

The Java standard library and its ecosystem are full of Adapter pattern usage. Understanding these examples solidifies pattern recognition.

### 6.1 `Arrays.asList()` — Array to List Adapter

`Arrays.asList()` returns a fixed-size `List` backed by the original array. The `List` interface is the target; the raw array is the adaptee.

```java
import java.util.Arrays;
import java.util.List;

public class ArraysAsListExample {
    public static void main(String[] args) {
        // The raw array (Adaptee) cannot be used where List (Target) is required.
        String[] rawArray = {"alpha", "beta", "gamma"};

        // Arrays.asList() is the Adapter factory method.
        // The returned list is backed by the array — modifications to the
        // list write through to the array, and vice versa.
        List<String> adaptedList = Arrays.asList(rawArray);

        System.out.println("Size: " + adaptedList.size());       // 3
        System.out.println("Get: " + adaptedList.get(1));        // beta
        System.out.println("Contains: " + adaptedList.contains("alpha")); // true

        // IMPORTANT GOTCHA: The returned list is fixed-size.
        // Calling add() or remove() throws UnsupportedOperationException.
        // This is a deliberate trade-off — it reflects the array's inability
        // to resize. The adapter faithfully translates the adaptee's constraints.
        try {
            adaptedList.add("delta"); // throws UnsupportedOperationException
        } catch (UnsupportedOperationException e) {
            System.out.println("Cannot add to fixed-size adapter: " + e.getClass().getName());
        }

        // If you need a fully mutable List, wrap it further:
        List<String> mutableCopy = new java.util.ArrayList<>(adaptedList);
        mutableCopy.add("delta"); // works fine
        System.out.println("Mutable copy: " + mutableCopy);
    }
}
```

### 6.2 `Collections.list()` — Enumeration to List Adapter

`Collections.list(Enumeration<T>)` adapts the legacy `Enumeration` interface (from Java 1.0) to the modern `ArrayList`. This bridges old APIs (e.g., `Hashtable.keys()`) with modern collection code.

```java
import java.util.*;

public class CollectionsListExample {
    public static void main(String[] args) {
        // Hashtable uses the legacy Enumeration interface
        Hashtable<String, Integer> legacyTable = new Hashtable<>();
        legacyTable.put("a", 1);
        legacyTable.put("b", 2);
        legacyTable.put("c", 3);

        // keys() returns Enumeration<String> — the legacy "Adaptee"
        Enumeration<String> enumeration = legacyTable.keys();

        // Collections.list() is the Adapter: converts Enumeration → ArrayList
        // Now we can use modern List operations (sort, stream, etc.)
        List<String> keys = Collections.list(enumeration);
        Collections.sort(keys);
        System.out.println("Sorted keys: " + keys); // [a, b, c]

        // We can also go the other way — List → Enumeration:
        List<String> modernList = Arrays.asList("x", "y", "z");
        Enumeration<String> asEnum = Collections.enumeration(modernList);
        while (asEnum.hasMoreElements()) {
            System.out.print(asEnum.nextElement() + " ");
        }
    }
}
```

### 6.3 `InputStreamReader` — Wrapping InputStream with Character Decoding

`InputStreamReader` is a textbook Object Adapter. The target is the `Reader` interface (character streams). The adaptee is `InputStream` (byte streams). The adapter bridges the byte/character boundary, performing charset decoding.

```java
import java.io.*;
import java.nio.charset.StandardCharsets;

public class InputStreamReaderExample {
    public static void main(String[] args) throws IOException {
        // InputStream (Adaptee) — raw bytes, no charset awareness
        InputStream byteStream = new FileInputStream("/etc/hosts");

        // InputStreamReader (Adapter) — bridges to Reader (Target)
        // Performs UTF-8 decoding in the translation layer
        Reader charStream = new InputStreamReader(byteStream, StandardCharsets.UTF_8);

        // BufferedReader works with Reader — it never sees the InputStream
        try (BufferedReader reader = new BufferedReader(charStream)) {
            String firstLine = reader.readLine();
            System.out.println("First line: " + firstLine);
        }

        // The adapter chain: BufferedReader → InputStreamReader → FileInputStream
        // Each layer knows only about its immediate collaborator's interface.
    }
}
```

### 6.4 `OutputStreamWriter` — Symmetric Output Adapter

The symmetric counterpart to `InputStreamReader`. The target is `Writer` (character output). The adaptee is `OutputStream` (byte output). The adapter encodes characters to bytes using a specified charset.

```java
import java.io.*;
import java.nio.charset.StandardCharsets;

public class OutputStreamWriterExample {
    public static void main(String[] args) throws IOException {
        // OutputStream (Adaptee) — accepts bytes
        OutputStream byteOutput = new FileOutputStream("/tmp/output.txt");

        // OutputStreamWriter (Adapter) — exposes Writer interface (Target)
        // Encodes characters to UTF-8 bytes in the translation layer
        Writer charOutput = new OutputStreamWriter(byteOutput, StandardCharsets.UTF_8);

        try (PrintWriter writer = new PrintWriter(new BufferedWriter(charOutput))) {
            writer.println("Hello from the Adapter pattern!");
            writer.printf("Written at: %s%n", java.time.LocalDateTime.now());
        }
        // The adapter is transparent: PrintWriter only sees Writer,
        // never the underlying OutputStream.
    }
}
```

### 6.5 `slf4j` Logging Facade and Adapters

SLF4J is one of the most prominent Adapter pattern implementations in the Java ecosystem. SLF4J defines the `Logger` interface (the target). Multiple adapter modules translate SLF4J calls to Logback, Log4j2, java.util.logging, and others.

```
Application Code
      |
      v
SLF4J API (org.slf4j.Logger) -- TARGET INTERFACE
      |
      v
SLF4J Adapter JAR (e.g., slf4j-log4j12.jar) -- ADAPTER
      |
      v
Log4j 1.x (underlying logging framework) -- ADAPTEE
```

This allows application code to be written against the SLF4J interface and the actual logging implementation to be swapped by changing a JAR on the classpath — without modifying a single line of application code.

---

## 7. Object Adapter vs Class Adapter Comparison

| Dimension | Object Adapter (Composition) | Class Adapter (Inheritance) |
|---|---|---|
| **Mechanism** | Holds a reference to the adaptee (has-a) | Extends the adaptee class (is-a) |
| **Java feasibility** | Always possible | Only possible when Target is an interface and there is only one adaptee class to extend |
| **Multiple adaptees** | Can compose multiple adaptees | Cannot — Java only allows single class inheritance |
| **Adaptee subclasses** | Can wrap any subclass of the adaptee at runtime | Statically bound to one adaptee class at compile time |
| **Adaptee override** | Cannot override adaptee methods | Can override adaptee methods (add/change behaviour) |
| **Coupling** | Loosely coupled — adaptee can be swapped | Tightly coupled — adapter IS an adaptee subtype |
| **Testability** | Easy to inject a mock adaptee | Harder — must subclass the adapter to mock adaptee behavior |
| **GoF preference** | Preferred in OO design (favors composition) | Acceptable in specific, constrained scenarios |
| **Exposure of adaptee API** | Adaptee API is fully hidden | Adaptee's public methods are inherited and accessible |
| **Java example** | `InputStreamReader`, `StripePaymentAdapter` | `LegacyLoggerClassAdapter` (our example) |
| **Flexibility** | High — runtime flexibility | Low — compile-time binding |

**Bottom line for Java**: Always prefer the Object Adapter. The Class Adapter is occasionally useful when you need to override adaptee behavior or when the subtyping relationship is genuinely meaningful, but the composition-based Object Adapter is the standard choice in idiomatic Java.

---

## 8. Trade-offs (Pros and Cons)

### Pros

**1. Open/Closed Principle compliance**
You can introduce new adapters without modifying existing client code or the adaptee. Adding PayPal support alongside Stripe means writing a new `PayPalPaymentAdapter` — nothing else changes.

**2. Single Responsibility Principle**
The adapter owns all translation logic. Your business service (`CheckoutService`) has no currency-to-cents conversion, no Stripe exception handling, no lowercase currency codes. That all lives in one place.

**3. Legacy code reuse without modification**
Decades-old, battle-tested code that has accumulated edge-case fixes can be preserved and wrapped. Rewriting it risks introducing bugs in code paths that have not been exercised in years.

**4. Vendor independence and pluggability**
Business logic depends on your interface, not on any vendor. Switching from Stripe to Braintree is an adapter swap, not a codebase refactor.

**5. Testability**
Because the client depends on an interface, you can inject a `MockPaymentProcessor` in tests without any external dependencies. The adapter itself can be tested in isolation with a mocked adaptee.

**6. Anti-Corruption Layer in DDD**
In Domain-Driven Design terms, adapters form an Anti-Corruption Layer that prevents external system concepts from leaking into your domain model. Your `PaymentResult` is a domain object; Stripe's `StripeCharge` is not.

### Cons

**1. Additional indirection and complexity**
Every call traverses an extra layer. In most business applications, this is imperceptible. In tight loops or very high-frequency code paths, the object allocation and method dispatch overhead may accumulate.

**2. Proliferation of adapter classes**
For large APIs with many methods, each adapter class can become lengthy. Adapting a 30-method SDK means 30 translation methods, each with potential error handling and type conversion. This code is often mechanical and tedious.

**3. Impedance mismatch cannot always be perfectly bridged**
If the adaptee and target are semantically very different (not just syntactically), the adapter may need to make compromising assumptions. For example, if Stripe supports three-decimal-place currencies and your interface uses `BigDecimal` but your downstream only handles two decimals, the adapter must make a policy decision about rounding — which arguably belongs in business logic.

**4. The adapter can obscure errors**
If the adapter swallows or improperly translates exceptions, debugging becomes harder. A Stripe rate-limit error translated generically to `PaymentException("charge failed")` loses valuable diagnostic information.

**5. Interface explosion if not governed**
Without discipline, teams may create multiple adapters for the same adaptee to satisfy different interfaces, leading to duplicated translation logic. A `StripePaymentAdapter`, `StripeRefundAdapter`, and `StripeBillingAdapter` that each independently convert BigDecimal to cents is an anti-pattern.

**6. Class Adapter's tight coupling is a long-term liability**
If you use a Class Adapter and the adaptee changes its public API in a major version, your adapter is broken and cannot wrap a different subclass. The Object Adapter does not have this problem.

---

## 9. Common Pitfalls and Gotchas

### Pitfall 1: Leaking Adaptee Types Through the Adapter

```java
// BAD: Leaking StripeException through the adapter's interface
public PaymentResult charge(...) throws StripeException { // WRONG
    ...
}

// GOOD: Translate to your domain exception
public PaymentResult charge(...) throws PaymentException { // CORRECT
    try {
        stripeClient.createCharge(...);
    } catch (StripeException e) {
        throw new PaymentException(e.getMessage(), e.getStripeCode(), e);
    }
}
```

If Stripe's exception type appears in your adapter's signature, all callers must import Stripe's SDK. You have failed to isolate the boundary.

### Pitfall 2: Confusing Adapter with Facade

Both wrap existing classes. The Facade provides a simplified interface to a complex subsystem (reducing methods). The Adapter translates an existing interface to a different interface (not simplifying, just translating). A Facade might internally use Adapters, but they are different patterns with different intents.

### Pitfall 3: Putting Business Logic in the Adapter

The adapter's job is translation, not decision-making.

```java
// BAD: Business rule in the adapter
public PaymentResult charge(String token, BigDecimal amount, Currency currency)
        throws PaymentException {
    if (amount.compareTo(new BigDecimal("10000")) > 0) {
        // Fraud check — this does NOT belong in the adapter
        requireManualApproval(token, amount);
    }
    ...
}

// GOOD: Adapter only translates; caller handles business rules
public PaymentResult charge(String token, BigDecimal amount, Currency currency)
        throws PaymentException {
    long cents = toCents(amount);
    try {
        StripeCharge charge = stripeClient.createCharge(cents,
                currency.getCurrencyCode().toLowerCase(), token);
        return toPaymentResult(charge);
    } catch (StripeException e) {
        throw toPaymentException(e);
    }
}
```

### Pitfall 4: Thread Safety

If the adaptee is not thread-safe, neither is your adapter unless you add synchronization. The adapter does not magically fix thread-safety issues in the adaptee.

### Pitfall 5: `Arrays.asList()` Mutability Trap

As shown in the real-world examples section, `Arrays.asList()` returns a fixed-size list. Calling `add()` or `remove()` throws `UnsupportedOperationException`. This surprises many developers who expect a standard mutable `ArrayList`. When you need a mutable list, use `new ArrayList<>(Arrays.asList(...))`.

### Pitfall 6: Ignoring the Adaptee's Lifecycle

If the adaptee holds resources (database connections, file handles, HTTP clients), the adapter must properly participate in the resource lifecycle:

```java
// If StripeClient holds a connection pool, your adapter should
// implement AutoCloseable and close it properly.
public final class StripePaymentAdapter implements PaymentProcessor, AutoCloseable {
    private final StripeClient stripeClient;

    @Override
    public void close() {
        // Delegate resource cleanup to the adaptee
        if (stripeClient instanceof AutoCloseable) {
            try { ((AutoCloseable) stripeClient).close(); }
            catch (Exception e) { log.warning("Failed to close StripeClient: " + e); }
        }
    }
}
```

---

## 10. Comparison With Related Patterns

The Adapter is frequently confused with Decorator, Proxy, Facade, and Bridge because all of them wrap one object in another. The key differentiator is always **intent**.

### 10.1 Adapter vs Decorator

| Dimension | Adapter | Decorator |
|---|---|---|
| **Primary intent** | Translate an incompatible interface to a compatible one | Add responsibilities/behavior to an object without subclassing |
| **Interface change** | YES — the adapter exposes a DIFFERENT interface than the adaptee | NO — the decorator implements the SAME interface as the component |
| **Structural shape** | Adapter implements Target, wraps Adaptee (different types) | Decorator implements Component, wraps Component (same type) |
| **Typical problem** | "These two classes cannot work together because their interfaces differ" | "I need to add logging/caching/validation to this object at runtime" |
| **Client perspective** | Client uses Target interface, unaware of Adaptee | Client uses Component interface; decorator is transparent |
| **Stacking** | Adapters are not typically stacked (one adapter per boundary) | Decorators are designed to be stacked (chain of responsibility) |

```java
// Adapter: Client uses PaymentProcessor, but adaptee is StripeClient (different type)
PaymentProcessor adapter = new StripePaymentAdapter(stripeClient);

// Decorator: Client uses PaymentProcessor, decorators ALSO implement PaymentProcessor
PaymentProcessor loggingProcessor = new LoggingPaymentDecorator(adapter);
PaymentProcessor cachingProcessor = new CachingPaymentDecorator(loggingProcessor);
```

### 10.2 Adapter vs Proxy

| Dimension | Adapter | Proxy |
|---|---|---|
| **Primary intent** | Interface translation | Access control, lazy initialization, remote invocation, caching |
| **Interface change** | YES — exposes a different interface | NO — implements the SAME interface as the subject |
| **Control** | No control intent — just translation | Controls access to the subject |
| **Transparency** | May be transparent to the client | Always transparent (client sees same interface) |
| **Typical uses** | Third-party integration, legacy wrapping | Security proxies, virtual proxies, remote proxies, monitoring |

### 10.3 Adapter vs Facade

| Dimension | Adapter | Facade |
|---|---|---|
| **Primary intent** | Make an incompatible interface compatible | Provide a simplified interface to a complex subsystem |
| **Interface** | Translates existing interfaces — no new simplification | Creates a NEW, simpler interface over the subsystem |
| **Adaptee count** | Usually wraps ONE class | Usually coordinates MULTIPLE classes/subsystems |
| **Interface change** | Converts existing interface, one-to-one (mostly) | Reduces complexity — fewer, simpler methods |
| **Preexisting interface** | Target interface predates the adapter | Facade defines a brand new simplified interface |

**Mnemonic**: Adapter makes things fit. Facade makes things simple. Proxy makes things controlled. Decorator makes things richer.

### 10.4 Adapter vs Bridge

| Dimension | Adapter | Bridge |
|---|---|---|
| **Primary intent** | Retrofit: make two existing classes work together | Proactive: decouple abstraction from implementation before building either |
| **Design timing** | Applied AFTER classes are built (usually) | Designed UPFRONT to allow independent variation |
| **Problem** | Existing incompatibility | Future flexibility — "I want to vary abstraction and implementation independently" |
| **Structural similarity** | Both use composition | Both use composition |
| **Mutability** | Adaptee is usually fixed; adapter absorbs all change | Both abstraction and implementation hierarchies can evolve independently |

The deepest distinction: Adapter is a **patch** for an existing incompatibility. Bridge is an **architectural decision** to ensure two dimensions of variation never become coupled in the first place.

---

## 11. Common Interview Discussion Points

### Point 1: Composition Over Inheritance in Java

The reason Object Adapter is preferred over Class Adapter in Java is rooted in Java's single-inheritance constraint and the broader OOP principle of favoring composition over inheritance. Interviewers expect you to explain why extending a class creates tighter coupling than holding a reference to it, and how composition allows you to swap the adaptee at runtime.

### Point 2: The Anti-Corruption Layer Connection

In Domain-Driven Design, an Anti-Corruption Layer (ACL) is used to translate between your bounded context and an external system's concepts. The Adapter pattern is the primary implementation mechanism for an ACL. When you say "we wrap Stripe in an adapter," you are implementing an ACL — preventing Stripe's domain model from bleeding into your checkout domain.

### Point 3: Adapter vs Open/Closed Principle

The Adapter pattern enables Open/Closed Principle compliance at integration boundaries. Rather than modifying `CheckoutService` every time you add a payment provider, you write a new adapter. The service is closed for modification and open for extension through the adapter mechanism.

### Point 4: Adapter in Hexagonal Architecture

In Hexagonal Architecture (Ports and Adapters), the `PaymentProcessor` interface is a **Port** — a boundary point of the application. The `StripePaymentAdapter` is a **secondary adapter** (also called a driven adapter) that connects the application to an external system. This naming convention is directly derived from the Adapter pattern.

### Point 5: Testing Strategy

Adapters make systems more testable: business logic tests inject a `MockPaymentProcessor` (no Stripe API call). Adapter integration tests use a real Stripe test environment. The two concerns are separated by the adapter boundary.

---

## 12. Interview Questions with Model Answers

---

### Q1: What is the Adapter design pattern and what problem does it solve?

**Model Answer:**

The Adapter is a structural design pattern that converts the interface of one class into an interface that another class expects. It solves the problem of interface incompatibility — when two classes need to collaborate but their APIs are mismatched, and you cannot or should not modify either of them.

The canonical scenario is third-party library integration. Your application defines a `PaymentProcessor` interface with `charge(String token, BigDecimal amount, Currency currency)`. Stripe's SDK has `StripeClient.createCharge(long amountInCents, String currencyCode, String source)`. The semantics are equivalent, but the APIs are completely different — different method names, different parameter types (BigDecimal vs long), different currency representation. You cannot modify Stripe's SDK, and you do not want Stripe-specific code scattered across your business logic.

The Adapter solves this by creating a class (`StripePaymentAdapter`) that implements your `PaymentProcessor` interface and holds a reference to `StripeClient`. Each method on the adapter translates the call: converting BigDecimal to cents, lowercasing the currency code, converting Stripe's response to your `PaymentResult`, and mapping Stripe's exceptions to your `PaymentException`. The client is completely isolated from Stripe.

The pattern also appears in the Java standard library: `InputStreamReader` adapts `InputStream` (byte stream) to `Reader` (character stream) by performing charset decoding in the translation layer; `Arrays.asList()` adapts a raw array to the `List` interface.

---

### Q2: What is the difference between Object Adapter and Class Adapter? Which would you use in Java and why?

**Model Answer:**

The fundamental difference is the mechanism used to connect the adapter to the adaptee.

An **Object Adapter** uses composition: the adapter implements the target interface and holds a private reference to the adaptee instance. When target methods are called, the adapter delegates to the adaptee. This is the GoF-preferred approach because it favors composition over inheritance.

A **Class Adapter** uses inheritance: the adapter extends the adaptee class and implements the target interface. By inheriting from the adaptee, it can directly call inherited methods and even override them to modify behavior.

In Java, I always recommend the **Object Adapter** for the following reasons:

First, Java supports only single class inheritance. The Class Adapter works only when the adaptee is a class (not abstract in a conflicting hierarchy) and the target is an interface. If you later need to adapt a subclass of the adaptee, or if the adaptee already extends something, the Class Adapter is structurally impossible.

Second, the Object Adapter allows runtime flexibility. You can wrap any object that satisfies the adaptee's type at runtime, enabling different adaptee implementations without changing the adapter class. The Class Adapter is bound to one concrete adaptee at compile time.

Third, the Object Adapter hides the adaptee's API entirely. When you extend the adaptee (Class Adapter), all its public methods are accessible on the adapter object — which means callers could bypass the adapter's interface. This breaks encapsulation.

Fourth, testability: in an Object Adapter, you inject a mock adaptee through the constructor. In a Class Adapter, you would need to subclass the adapter to change the adaptee's behavior, which is significantly more cumbersome.

The Class Adapter is occasionally useful when you need to override one or two adaptee methods in the translation — because inheritance lets you do that without calling through an interface. But this is a niche use case that rarely justifies the coupling cost.

---

### Q3: How does the Adapter pattern differ from the Decorator pattern?

**Model Answer:**

This is the most common confusing pair in the structural patterns category, and the confusion is understandable because both patterns wrap one object inside another. The critical difference is **intent** and the **interface relationship**.

The **Adapter** is about interface translation. The adapter implements a **different interface** than the object it wraps. The client cannot use the adaptee directly; the adapter bridges the gap. There is always a semantic mismatch that the adapter resolves. Example: `InputStreamReader` implements `Reader` but wraps an `InputStream` — two completely different interfaces.

The **Decorator** is about behavior extension. The decorator implements the **same interface** as the object it wraps. From the client's perspective, the decorator is transparent — it is used exactly like the original object, except it adds responsibilities (logging, caching, validation) around the core behavior. Example: `BufferedInputStream` implements `InputStream` and wraps an `InputStream` — same interface, added buffering behavior.

A structural way to remember this: if you can substitute the wrapper for the wrapped object from the client's perspective (same interface), it is a Decorator. If the client must use the wrapper because the wrapped object has the wrong interface, it is an Adapter.

There is also a practical implication for stacking. Decorators are designed to be stacked — you can have `LoggingDecorator(CachingDecorator(actualService))` because each decorator satisfies the same interface as the next. Adapters are not typically stacked in this way, because each adapter changes the interface.

One more nuance: you can combine them. A `StripePaymentAdapter` adapts Stripe's API to your `PaymentProcessor` interface. Then a `LoggingPaymentDecorator` implements `PaymentProcessor` and wraps the adapter — adding logging around the already-adapted Stripe calls. The adapter and decorator serve different roles and compose cleanly.

---

### Q4: Walk me through implementing an Adapter for a legacy system integration. What would you consider when designing it?

**Model Answer:**

I will walk through the thought process using the XML-to-JSON adapter as a reference.

**Step 1: Define the Target Interface.** Before writing any adapter code, I define the interface that the new system expects. This must be driven by the consumer's needs, not by what the legacy system provides. In our example, the modern reporting service expects JSON, so the interface declares `getCustomerAsJson()` and `getOrderHistoryAsJson()`.

**Step 2: Understand the Adaptee's API completely.** I map every method I need to call on the legacy system, including its error behavior, its threading model (is it thread-safe?), its resource lifecycle (does it need to be closed?), and any quirks or inconsistencies. I should not discover mid-implementation that the legacy system throws an undocumented checked exception.

**Step 3: Design the translation layer.** This is the core of the adapter. For XML-to-JSON, I need a reliable XML parsing strategy (DOM for small documents, SAX/StAX for large ones) and a JSON serialization strategy. In production, I would use Jackson's `ObjectMapper` rather than manual string building. I map each legacy field to the corresponding JSON property, handling nulls, encoding, and schema differences.

**Step 4: Design error handling carefully.** Legacy systems often have inconsistent error signaling — some errors are exceptions, some are error codes in the return value, some are null returns. The adapter must normalize all of these into the target interface's error contract. I should not leak legacy exception types through the adapter's method signatures.

**Step 5: Consider thread safety.** If the legacy system is not thread-safe, the adapter must either synchronize access or document that it inherits the legacy system's threading constraints. If the `DocumentBuilderFactory` is shared across calls, I create a new `DocumentBuilder` per call rather than reusing one (which is not thread-safe).

**Step 6: Resource lifecycle.** If the legacy system holds resources, the adapter should implement `AutoCloseable` and delegate `close()` to the legacy system.

**Step 7: Write tests at the adapter boundary.** Unit tests mock the legacy system and verify the adapter's translation. Integration tests use the real legacy system.

---

### Q5: How does the Adapter pattern relate to the hexagonal architecture (Ports and Adapters)?

**Model Answer:**

Hexagonal Architecture — also called Ports and Adapters, coined by Alistair Cockburn — is an architectural style where the application core is isolated from external concerns (databases, UIs, external services) by defining interfaces (Ports) at the boundary. Concrete implementations of those interfaces (Adapters) connect the core to the outside world.

The naming is not coincidental: Ports and Adapters is a direct architectural application of the Adapter design pattern.

In hexagonal architecture, there are two categories of adapters. **Primary (driving) adapters** drive the application: an HTTP controller, a Kafka consumer, or a CLI handler all implement or call the application's inbound port. **Secondary (driven) adapters** are driven by the application: a database repository, an external API client, or a message publisher implement the application's outbound port.

In our payment example, `PaymentProcessor` is an **outbound port** — it is the application's way of saying "I need to charge someone; here is the interface for that." `StripePaymentAdapter` is a **secondary adapter** — it connects the port to Stripe's actual API. The business logic (`CheckoutService`) only depends on the port. Stripe is a plug-in detail.

This architecture has profound implications for testability and deployability. In tests, you inject an in-memory `FakePaymentProcessor` as the secondary adapter. In production, you inject the Stripe adapter. In a staging environment, you might inject a sandbox adapter. The core application code is identical in all cases.

The pattern also aligns with the Dependency Inversion Principle: the high-level module (`CheckoutService`) depends on an abstraction (`PaymentProcessor`), and the low-level module (`StripePaymentAdapter`) depends on that same abstraction. The direction of dependency is inverted relative to the direction of control flow.

---

### Q6: What are the trade-offs of using the Adapter pattern? When might it be the wrong choice?

**Model Answer:**

The Adapter pattern has real costs alongside its benefits, and knowing when not to use it is as important as knowing when to use it.

On the benefit side, the most significant advantages are vendor isolation (business logic does not depend on external API details), Open/Closed Principle compliance (you can add providers without modifying consumers), and testability (the adapter boundary makes mocking trivial).

On the cost side, the most significant issues are:

**Impedance mismatch cannot always be cleanly bridged.** If the adaptee and the target have fundamentally different models — not just different syntax — the adapter is forced to make semantic decisions. For example, if your `PaymentProcessor` assumes idempotency (retry-safe charges) but Stripe requires explicit idempotency keys, the adapter must generate and manage those keys, which is arguably business logic. When the translation becomes complex, the adapter gains too much responsibility and should be split or replaced with a more structured integration pattern.

**The adapter can become a maintenance burden.** If the adaptee changes frequently (e.g., major SDK updates), the adapter absorbs all the changes. A Stripe v7 to v8 migration might require rewriting large portions of the adapter if the API changed significantly. This is not inherently bad — it is exactly the isolation you wanted — but the adapter's complexity can grow.

**Proliferation without governance.** Teams sometimes create multiple adapters for the same adaptee to satisfy different internal interfaces, leading to duplicated conversion logic. The BigDecimal-to-cents conversion should live in one place.

**Wrong choice scenarios:** If you control both the client and the adaptee, the right answer is to redesign one interface to be compatible rather than adding an adapter layer. Adapters are a workaround — if the workaround can be avoided by a better initial design, avoid it. Also, if the adaptation is complex enough to require conditional business logic, an orchestration layer (Service, Mediator) is more appropriate than an adapter.

---

### Q7: Explain how `Arrays.asList()` and `InputStreamReader` implement the Adapter pattern. What are the gotchas?

**Model Answer:**

Both are canonical examples of the Adapter pattern in the Java standard library, and both have important behavioral nuances that are common interview traps.

`Arrays.asList()` adapts a raw Java array (the adaptee) to the `List<T>` interface (the target). The raw array cannot be passed where a `List` is expected; `Arrays.asList()` returns a `java.util.Arrays.ArrayList` (an inner class, not `java.util.ArrayList`) that delegates all `List` operations to the underlying array. The translation is straightforward: `get(i)` becomes `array[i]`, `set(i, v)` becomes `array[i] = v`, and `size()` returns `array.length`.

The critical gotcha is that the returned list has **fixed size**. Because arrays cannot be resized, the adapter faithfully preserves this constraint. Calling `add()` or `remove()` throws `UnsupportedOperationException`. Many developers expect a standard mutable `ArrayList` and are surprised. The fix is `new ArrayList<>(Arrays.asList(array))`. Also, the returned list is backed by the original array — mutations through `list.set(i, v)` modify the original array. This is a second surprise for developers who expect a copy.

`InputStreamReader` adapts `InputStream` (the adaptee — byte-oriented) to `Reader` (the target — character-oriented). The translation responsibility is **character encoding**: reading bytes from the `InputStream` and decoding them to `char` values according to a specified `Charset`. The adapter does not store all bytes and decode at the end; it decodes incrementally, which is correct for streaming and important for performance.

The critical gotcha here is that if you create an `InputStreamReader` without specifying a charset, it uses the platform default encoding (`Charset.defaultCharset()`). This default varies by operating system, JVM settings, and locale — making your code non-portable. Always specify the charset explicitly: `new InputStreamReader(stream, StandardCharsets.UTF_8)`. The second gotcha: closing the `InputStreamReader` also closes the underlying `InputStream`. If you need to read from the same `InputStream` after closing the reader, you will fail.

---

### Q8: How would you design an Adapter that handles retry logic and rate limiting imposed by a third-party API?

**Model Answer:**

This is a design question that explores where the adapter's responsibility ends and cross-cutting concerns begin.

My first instinct is that retry logic and rate limiting are not translation concerns — they are resilience concerns. The pure Adapter pattern says: translate the interface, nothing more. Mixing retry logic into the adapter violates the Single Responsibility Principle and makes the adapter hard to test (now you need to test retry behavior too).

My preferred approach is a **layered design**:

```
CheckoutService
      |
      v
PaymentProcessor (interface)
      |
      v
RetryingPaymentProcessor (Decorator — adds retry logic)
      |
      v
RateLimitedPaymentProcessor (Decorator — adds rate limiting)
      |
      v
StripePaymentAdapter (Adapter — translates interface, that is all)
      |
      v
StripeClient (Adaptee)
```

The `RetryingPaymentProcessor` is a Decorator — it implements `PaymentProcessor`, wraps another `PaymentProcessor`, and retries on transient failures (e.g., `PaymentException` with a retriable error code). The `RateLimitedPaymentProcessor` is another Decorator that throttles calls using a `RateLimiter`. The `StripePaymentAdapter` is a pure adapter: it only translates.

This composition approach means:
- Retry logic is testable independently of Stripe
- Rate limiting can be reused with any payment processor, not just Stripe
- The adapter stays small and focused
- You can compose or remove decorators independently

There is one exception: if retry logic is inherently coupled to the adaptee's semantics — for example, if Stripe returns a specific `429 Too Many Requests` HTTP status that must be handled by inspecting a Stripe-specific `retry-after` header — then the adapter must detect this and either retry internally or signal to the outer decorator that this is a rate-limit error with a backoff hint. In that case, the adapter exposes the retry signal in the normalized exception (e.g., `PaymentException` with a `isRetriable()` method and `getRetryAfterMs()` hint), and the decorator decides the retry strategy.

The key principle: the adapter translates semantics; the decorator adds behavior. When they must interact, they do so through the domain types the adapter defines, not through leaking adaptee-specific types.

---

## Summary

| Concept | Key Takeaway |
|---|---|
| **Intent** | Convert an incompatible interface into one the client expects |
| **Mechanism** | Implement Target, wrap Adaptee (Object); or extend Adaptee + implement Target (Class) |
| **Prefer** | Object Adapter (composition) over Class Adapter (inheritance) in Java |
| **Do not confuse with** | Decorator (same interface, adds behavior), Proxy (same interface, controls access), Facade (new simplified interface over complex subsystem) |
| **Best use** | Third-party SDK integration, legacy system modernization, vendor-independent interfaces |
| **Key risk** | Adapter becomes too complex when adaptee and target are semantically (not just syntactically) incompatible |
| **In the wild** | `InputStreamReader`, `OutputStreamWriter`, `Arrays.asList()`, `Collections.list()`, SLF4J adapters, Spring's `HandlerAdapter` |
| **Architectural pattern** | Directly implements Ports and Adapters (Hexagonal Architecture) secondary adapters |
