# Module 00: Foundation — OOP, Clean Code, and the LLD Mindset

**Target audience:** Software engineers with 2-8 years of experience preparing for Senior Engineer interviews and roles.

**Prerequisites:** Familiarity with at least one object-oriented language. Java is used for all examples in this module.

**Learning objectives:**

- Articulate the four OOP pillars with precision, not just buzzword definitions
- Identify and correct common design mistakes rooted in misunderstood OOP
- Apply clean code principles to produce readable, maintainable Java
- Approach any LLD interview problem using a repeatable, structured framework

---

## Table of Contents

1. [OOP Fundamentals Review](#1-oop-fundamentals-review)
   - 1.1 [Encapsulation](#11-encapsulation)
   - 1.2 [Abstraction](#12-abstraction)
   - 1.3 [Inheritance](#13-inheritance)
   - 1.4 [Polymorphism](#14-polymorphism)
2. [Clean Code Principles](#2-clean-code-principles)
   - 2.1 [Naming Conventions](#21-naming-conventions)
   - 2.2 [Function Design](#22-function-design)
   - 2.3 [Code Organization](#23-code-organization)
   - 2.4 [Comments vs Self-Documenting Code](#24-comments-vs-self-documenting-code)
3. [How to Approach LLD Problems](#3-how-to-approach-lld-problems)
   - 3.1 [The Five-Stage Framework](#31-the-five-stage-framework)
   - 3.2 [Requirements Clarification Strategy](#32-requirements-clarification-strategy)
   - 3.3 [Domain Modeling Approach](#33-domain-modeling-approach)
   - 3.4 [Iterative Design Mindset](#34-iterative-design-mindset)
4. [Module Assignments](#4-module-assignments)

---

## 1. OOP Fundamentals Review

Most engineers can name the four pillars. Very few can define them precisely, distinguish them from each other, or explain the concrete design problems each one solves. This section closes that gap.

---

### 1.1 Encapsulation

#### Theory

**Precise definition:** Encapsulation is the bundling of data (fields) and the operations that act on that data (methods) into a single unit (a class), while restricting direct external access to the internal representation. The class exposes a controlled public interface and hides its implementation details.

Encapsulation solves two problems simultaneously:

1. **Data protection:** External code cannot put an object into an invalid state.
2. **Implementation freedom:** The internal representation can change without breaking callers, as long as the public interface remains stable.

The access modifiers `private`, `protected`, `package-private`, and `public` are the mechanical tools. The design principle is the goal: **hide what can change, expose what must be stable.**

A critical distinction: encapsulation is not just "use getters and setters." A class with a private field and a pair of `getX()`/`setX()` methods with no validation is nominally encapsulated but architecturally hollow — the field might as well be public. Genuine encapsulation means the class enforces its own invariants.

#### Java Code Example

**Bad — public fields, no invariant enforcement:**

```java
// Any caller can set age to -5 or balance to Long.MIN_VALUE.
// The class has no ability to defend itself.
public class BankAccount {
    public String accountNumber;
    public long balanceInPaise;
    public int age;
}
```

**Bad — getters/setters with no validation (encapsulation theater):**

```java
public class BankAccount {
    private long balanceInPaise;

    public long getBalanceInPaise() {
        return balanceInPaise;
    }

    // This setter provides zero protection over a public field.
    public void setBalanceInPaise(long balanceInPaise) {
        this.balanceInPaise = balanceInPaise;
    }
}
```

**Good — the class owns and enforces its invariants:**

```java
public final class BankAccount {

    private final String accountNumber;
    private long balanceInPaise;   // stored in smallest unit to avoid float precision issues
    private final String currency;

    public BankAccount(String accountNumber, long initialBalanceInPaise, String currency) {
        if (accountNumber == null || accountNumber.isBlank()) {
            throw new IllegalArgumentException("Account number must not be blank");
        }
        if (initialBalanceInPaise < 0) {
            throw new IllegalArgumentException("Initial balance cannot be negative");
        }
        if (currency == null || currency.length() != 3) {
            throw new IllegalArgumentException("Currency must be a 3-letter ISO 4217 code");
        }
        this.accountNumber = accountNumber;
        this.balanceInPaise = initialBalanceInPaise;
        this.currency = currency;
    }

    /**
     * Deposits the specified amount. Amount must be positive.
     */
    public void deposit(long amountInPaise) {
        if (amountInPaise <= 0) {
            throw new IllegalArgumentException("Deposit amount must be positive, got: " + amountInPaise);
        }
        this.balanceInPaise += amountInPaise;
    }

    /**
     * Withdraws the specified amount. Throws if insufficient funds.
     */
    public void withdraw(long amountInPaise) {
        if (amountInPaise <= 0) {
            throw new IllegalArgumentException("Withdrawal amount must be positive, got: " + amountInPaise);
        }
        if (amountInPaise > this.balanceInPaise) {
            throw new InsufficientFundsException(
                "Cannot withdraw " + amountInPaise + " from balance of " + this.balanceInPaise
            );
        }
        this.balanceInPaise -= amountInPaise;
    }

    // Read-only access — no setter. Balance changes only through deposit/withdraw.
    public long getBalanceInPaise() {
        return balanceInPaise;
    }

    public String getAccountNumber() {
        return accountNumber;
    }

    public String getCurrency() {
        return currency;
    }
}
```

**Why this is correct:**

- `balanceInPaise` has no setter. The only paths to changing it are `deposit` and `withdraw`, both of which validate their inputs.
- The constructor validates all fields before setting them, so an object of this class can never be in an invalid state after construction.
- `accountNumber` and `currency` are `final` — immutable after construction.
- The `final` class modifier prevents subclasses from inadvertently weakening these guarantees.

#### Common Misconceptions

1. **"Encapsulation means making all fields private and adding getters/setters."**
   This is the most common misconception. Unrestricted getters and setters give the illusion of encapsulation while providing none of its benefits. The question to ask is: does the class enforce its own invariants?

2. **"Encapsulation and information hiding are the same thing."**
   Information hiding is the broader design principle (hide implementation details). Encapsulation is one specific mechanism for achieving it in OOP.

3. **"Breaking encapsulation is always bad."**
   Sometimes returning a reference to a mutable internal collection is acceptable for performance, as long as the contract is documented and the risk is understood. Engineering is about trade-offs, not dogma.

4. **"Encapsulation only applies to fields."**
   Implementation methods (private helpers) are also part of encapsulation. Exposing every helper method as public is as harmful as exposing every field.

5. **"A final class is always better encapsulated."**
   `final` prevents inheritance-based bypass of invariants, which is often desirable. But it also prevents extension. The trade-off depends on whether the class is designed for extension.

#### Interview Questions

**Q1: What is encapsulation and why does it matter in production systems?**

Model answer: Encapsulation is the principle of bundling an object's data with the methods that operate on it, and restricting external access to the internal representation. It matters in production because (1) it prevents objects from being put into invalid states by external code, which reduces bugs that are hard to trace; (2) it allows the internal implementation to change without breaking callers — for example, changing how a balance is stored or calculated — which is essential for evolving a codebase without breaking consumers; and (3) it makes the class's responsibilities explicit, which aids readability and testing.

---

**Q2: I have a `User` class with a `List<Order> orders` field. I expose it via `getOrders()`. Is this well-encapsulated? How would you fix it?**

Model answer: No. Returning a direct reference to the internal list allows callers to add, remove, or clear orders without going through the `User` class, bypassing any invariants the class might want to enforce (e.g., an order must belong to this user, the list must never be null, a maximum order count). To fix it, return an unmodifiable view: `return Collections.unmodifiableList(orders)`. If callers need to add orders, expose a method like `addOrder(Order order)` that validates the input before mutating the list.

---

**Q3: How does encapsulation relate to testability?**

Model answer: Well-encapsulated classes are easier to test because (1) they have clear, narrow interfaces — you test through the public API rather than needing to set arbitrary internal state; (2) invariants are enforced in one place (the class itself), so tests can rely on the class being in a known valid state after any operation; and (3) the internal implementation can be changed (e.g., swapping a data structure) without rewriting tests, because tests are written against the interface, not the implementation.

---

**Q4: Describe a scenario where you intentionally broke encapsulation and explain the trade-off.**

Model answer (illustrative): In a high-throughput data pipeline, I had a class that held a large byte buffer. Returning a copy of the buffer in `getData()` was causing excessive garbage collection pressure. I chose to return a direct reference to the buffer, documented it clearly in the Javadoc as a "live view — do not modify," and added an assertion in tests to verify callers did not modify it. The trade-off was accepting a weaker encapsulation contract in exchange for a 40% reduction in GC pauses. This is acceptable when the risk is understood and documented, but it would be wrong to make it the default approach.

---

**Q5: What is the difference between encapsulation and immutability?**

Model answer: They address related but distinct concerns. Encapsulation controls who can change an object's state and through which paths. Immutability means an object's state cannot change at all after construction. An encapsulated object can still be mutable — it just controls mutations through its methods. An immutable object (e.g., Java's `String`, `LocalDate`) goes further by eliminating mutation entirely, which eliminates the need for defensive copying, simplifies concurrency, and makes the object inherently safe to share. Immutability is a stronger and simpler contract when it is applicable.

---

### 1.2 Abstraction

#### Theory

**Precise definition:** Abstraction is the process of exposing only the essential characteristics of an entity relevant to a given context, while hiding the irrelevant complexity. In OOP, it manifests as defining interfaces (or abstract classes) that represent a contract of what something does, without specifying how it does it.

The key word is "relevant." Abstraction is always relative to a context. A `PaymentGateway` abstraction for an e-commerce system should expose `charge(PaymentRequest)` and `refund(RefundRequest)` — not SSL handshake details, not HTTP retry logic. Those details exist; abstraction ensures they do not leak into the caller.

Abstraction and encapsulation are frequently confused. A clean distinction:
- **Encapsulation** hides internal state and ensures invariants — it is about protection.
- **Abstraction** hides implementation complexity behind a simplified interface — it is about cognitive manageability and decoupling.

In Java, abstraction is primarily expressed through:
- **Interfaces:** Pure contracts. No implementation (prior to default methods). The strongest form.
- **Abstract classes:** Partial implementations with a contract. Use when there is shared behavior that all subtypes should inherit.

#### Java Code Example

**Without abstraction — tight coupling to implementation:**

```java
public class OrderService {

    // Tightly coupled to Stripe. Any change to payment provider
    // requires modifying OrderService itself.
    public void processOrder(Order order) {
        StripeClient stripe = new StripeClient("sk_live_xxx");
        stripe.chargeCard(
            order.getCustomer().getCardToken(),
            order.getTotalAmountInPaise(),
            "INR"
        );
        // ... rest of order processing
    }
}
```

**With abstraction — decoupled from implementation:**

```java
// The contract. OrderService knows only this.
public interface PaymentGateway {
    PaymentResult charge(PaymentRequest request);
    RefundResult refund(RefundRequest request);
}

// One concrete implementation.
public class StripePaymentGateway implements PaymentGateway {

    private final StripeClient stripeClient;

    public StripePaymentGateway(StripeClient stripeClient) {
        this.stripeClient = stripeClient;
    }

    @Override
    public PaymentResult charge(PaymentRequest request) {
        // All Stripe-specific complexity is hidden here.
        StripeChargeParams params = StripeChargeParams.builder()
            .setAmount(request.getAmountInPaise())
            .setCurrency(request.getCurrencyCode())
            .setSource(request.getPaymentToken())
            .build();
        StripeCharge charge = stripeClient.charges().create(params);
        return new PaymentResult(charge.getId(), PaymentStatus.SUCCESS);
    }

    @Override
    public RefundResult refund(RefundRequest request) {
        // Stripe-specific refund logic hidden here.
        stripeClient.refunds().create(request.getTransactionId());
        return new RefundResult(RefundStatus.SUCCESS);
    }
}

// Another implementation — same interface.
public class RazorpayPaymentGateway implements PaymentGateway {

    @Override
    public PaymentResult charge(PaymentRequest request) {
        // Razorpay-specific logic — completely different internals.
        // OrderService does not care.
        return new PaymentResult(generateOrderId(), PaymentStatus.SUCCESS);
    }

    @Override
    public RefundResult refund(RefundRequest request) {
        return new RefundResult(RefundStatus.SUCCESS);
    }

    private String generateOrderId() {
        return "order_" + System.currentTimeMillis();
    }
}

// The consumer. Depends on the abstraction, never on a concrete class.
public class OrderService {

    private final PaymentGateway paymentGateway;

    // Dependency injected — OrderService has no idea which gateway it gets.
    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    public void processOrder(Order order) {
        PaymentRequest request = PaymentRequest.builder()
            .amountInPaise(order.getTotalAmountInPaise())
            .currencyCode(order.getCurrencyCode())
            .paymentToken(order.getCustomer().getPaymentToken())
            .build();

        PaymentResult result = paymentGateway.charge(request);

        if (result.getStatus() != PaymentStatus.SUCCESS) {
            throw new PaymentFailedException("Payment failed for order: " + order.getId());
        }

        order.markAsPaid(result.getTransactionId());
    }
}
```

**The design benefits:**

- `OrderService` can be tested by injecting a mock `PaymentGateway` — no real network calls.
- Switching from Stripe to Razorpay requires zero changes to `OrderService`.
- Adding a new payment provider requires zero changes to any existing class — only a new implementation of `PaymentGateway`.

#### When to Use Abstract Classes vs Interfaces

| Criterion | Prefer Interface | Prefer Abstract Class |
|---|---|---|
| Multiple inheritance needed | Yes — a class can implement many interfaces | No — Java has single class inheritance |
| Shared implementation | No shared logic | Yes — common behavior to inherit |
| Relationship type | "Can do" (capability) | "Is a" (type hierarchy) |
| Versioning concern | Default methods in Java 8+ mitigate | Easier to add non-breaking methods |

A `Flyable` interface is appropriate because many unrelated things can fly (birds, planes, drones). An `Animal` abstract class is appropriate because all animals share some behavior (e.g., eating, breathing) and the hierarchy represents a genuine type relationship.

#### Common Misconceptions

1. **"Abstraction means using abstract classes."**
   Interfaces are often a stronger form of abstraction in Java. The `abstract` keyword is a mechanical tool; abstraction is the design goal.

2. **"Every class should have an interface."**
   This is over-engineering. Create an interface when you genuinely need multiple implementations or when you need to decouple a consumer from a specific implementation for testability. A `UserRepositoryImpl` with no other implementation and no testing requirement for its callers does not need an `IUserRepository` interface.

3. **"Abstraction and encapsulation are the same."**
   As stated above: encapsulation = protection of state; abstraction = simplification of interface. A well-abstracted interface can expose details of state if the context demands it. A well-encapsulated class can have a very concrete, non-abstract interface.

4. **"Abstract classes are always better than interfaces because they provide more."**
   "More" is not always better. Interfaces force a clean separation of contract from implementation, which is usually desirable.

5. **"You should always abstract third-party libraries."**
   You should abstract the boundary of concern, not every class. Abstracting a logging library behind a custom interface adds complexity with little benefit. Abstracting a payment gateway behind an interface adds real, measurable value.

#### Interview Questions

**Q1: What is the difference between abstraction and encapsulation? Give a concrete example.**

Model answer: Encapsulation is about protecting internal state and enforcing invariants — it controls who can change what. Abstraction is about hiding complexity behind a simple interface — it controls what callers need to know. Concrete example: a `DatabaseConnection` class might use encapsulation to protect its internal connection pool (callers can't directly manipulate pool state) and use abstraction to expose a simple `query(String sql, Object... params)` interface that hides JDBC boilerplate, retry logic, and connection management from callers.

---

**Q2: When should you use an interface versus an abstract class?**

Model answer: Use an interface when you are defining a capability that can be implemented by unrelated types (e.g., `Serializable`, `Comparable`), when you need multiple inheritance, or when you want the strongest possible decoupling between contract and implementation. Use an abstract class when you have a genuine type hierarchy ("is-a" relationship) with shared implementation that all subclasses should inherit. In practice, prefer interfaces for defining service contracts (e.g., `UserRepository`, `NotificationService`) and abstract classes for template method patterns where steps vary but the overall algorithm is shared.

---

**Q3: You are building a notification system that sends emails, SMS, and push notifications. How would you apply abstraction here?**

Model answer: Define a `NotificationSender` interface with a single method like `send(Notification notification)`. Create three concrete implementations: `EmailNotificationSender`, `SmsNotificationSender`, and `PushNotificationSender`. The service that triggers notifications depends only on `NotificationSender`, not on any concrete implementation. This allows new channels (e.g., WhatsApp) to be added without changing the service, and each channel can be tested independently. If the three senders share non-trivial initialization or retry logic, an abstract `AbstractNotificationSender` that implements the retry behavior can sit between the interface and the concrete classes.

---

**Q4: What is the cost of abstraction? When does it become a liability?**

Model answer: Abstraction has real costs: indirection (a reader of the code must navigate from call site to interface to implementation to understand what actually happens), increased number of files and types, and potential confusion when an abstraction maps poorly to the domain. It becomes a liability when there is exactly one implementation and there will never be a second one — the interface adds no value while adding navigation overhead. It is also a liability when the abstraction is leaky, meaning callers must know which implementation they have to use it correctly. Good abstractions are stable, narrow, and accurately represent the concept they model.

---

**Q5: Can you give an example of a leaky abstraction and how you would fix it?**

Model answer: A leaky abstraction exposes details of its implementation through the interface. For example, a `DataStore` interface with methods like `executeRawSql(String sql)` and `setAutoCommit(boolean)` leaks the fact that the underlying implementation is a relational database. A service using this interface must know it is talking to a SQL database, defeating the purpose of the abstraction. A fix would be to redesign the interface around domain operations: `save(Entity entity)`, `findById(String id)`, `findByFilter(Filter filter)`. The interface now expresses what the caller needs, not how the storage layer works. SQL specifics move entirely into the implementation.

---

### 1.3 Inheritance

#### Theory

**Precise definition:** Inheritance is a mechanism by which a class (subclass or child class) acquires the fields and methods of another class (superclass or parent class), optionally overriding or extending them. It establishes an "is-a" relationship between the subclass and superclass.

Inheritance serves two purposes, which are frequently conflated and should be treated separately:

1. **Type polymorphism:** A `Dog` can be used wherever an `Animal` is expected.
2. **Code reuse:** A `Dog` inherits common behavior from `Animal` without rewriting it.

The danger is using inheritance for code reuse when there is no genuine "is-a" relationship. This is the root cause of brittle hierarchies and the reason the Gang of Four advised: **"Favor composition over inheritance."**

The Liskov Substitution Principle (LSP) provides the correctness test for inheritance: if you can substitute a subclass wherever the superclass is expected without breaking program correctness, the inheritance is valid. If the substitution breaks things, the hierarchy is wrong regardless of how much code it reuses.

#### Java Code Example

**Bad — inheritance for code reuse, no real "is-a" relationship:**

```java
// Stack "extends" Vector purely to reuse add/remove operations.
// This is exactly what Java's own java.util.Stack does — and it is
// widely considered a design mistake in the JDK.
// Result: Stack exposes all Vector methods (get by index, insert at position, etc.)
// which are not valid operations on a stack.
public class Stack<T> extends Vector<T> {
    public void push(T item) {
        add(item);
    }
    public T pop() {
        return remove(size() - 1);
    }
}

// The broken contract:
Stack<String> stack = new Stack<>();
stack.push("A");
stack.push("B");
stack.add(0, "C");  // Valid because Vector exposes it. But a Stack shouldn't allow this.
```

**Bad — LSP violation:**

```java
public class Rectangle {
    protected int width;
    protected int height;

    public void setWidth(int width)   { this.width = width; }
    public void setHeight(int height) { this.height = height; }

    public int area() { return width * height; }
}

// Geometrically, a Square "is-a" Rectangle. But behaviorally?
public class Square extends Rectangle {
    @Override
    public void setWidth(int side) {
        this.width = side;
        this.height = side;  // Must keep sides equal.
    }

    @Override
    public void setHeight(int side) {
        this.width = side;
        this.height = side;
    }
}

// This code is correct for Rectangle, but broken for Square:
void stretchWidth(Rectangle r) {
    int originalHeight = r.height;
    r.setWidth(r.width * 2);
    // For Rectangle: area doubles. Expected.
    // For Square: height ALSO doubles. Area quadruples. LSP violated.
    assert r.area() == originalHeight * r.width;  // Fails for Square.
}
```

**Good — inheritance where there is a genuine "is-a" with shared behavior:**

```java
// Abstract base: all notifications share a recipient, a subject, and a body.
// The "how to send" step varies by type.
public abstract class Notification {

    private final String recipientId;
    private final String subject;
    private final String body;
    private final Instant createdAt;

    protected Notification(String recipientId, String subject, String body) {
        this.recipientId = Objects.requireNonNull(recipientId);
        this.subject = Objects.requireNonNull(subject);
        this.body = Objects.requireNonNull(body);
        this.createdAt = Instant.now();
    }

    // Template method: shared algorithm, variable step.
    public final void send() {
        validate();
        doSend();
        logDelivery();
    }

    // Subclasses implement this step.
    protected abstract void doSend();

    private void validate() {
        if (recipientId.isBlank() || body.isBlank()) {
            throw new IllegalStateException("Notification is missing required fields");
        }
    }

    private void logDelivery() {
        System.out.printf("[%s] Notification sent to %s: %s%n", createdAt, recipientId, subject);
    }

    public String getRecipientId() { return recipientId; }
    public String getSubject()     { return subject; }
    public String getBody()        { return body; }
}

public class EmailNotification extends Notification {

    private final String toAddress;
    private final EmailClient emailClient;

    public EmailNotification(String recipientId, String toAddress,
                             String subject, String body, EmailClient emailClient) {
        super(recipientId, subject, body);
        this.toAddress = toAddress;
        this.emailClient = emailClient;
    }

    @Override
    protected void doSend() {
        emailClient.send(toAddress, getSubject(), getBody());
    }
}

public class SmsNotification extends Notification {

    private final String phoneNumber;
    private final SmsClient smsClient;

    public SmsNotification(String recipientId, String phoneNumber,
                           String body, SmsClient smsClient) {
        super(recipientId, "SMS", body);
        this.phoneNumber = phoneNumber;
        this.smsClient = smsClient;
    }

    @Override
    protected void doSend() {
        smsClient.send(phoneNumber, getBody());
    }
}
```

**Prefer composition for code reuse without "is-a":**

```java
// If we want Stack to reuse list operations, compose rather than inherit.
public class Stack<T> {

    private final Deque<T> storage = new ArrayDeque<>();

    public void push(T item) {
        storage.push(item);
    }

    public T pop() {
        if (storage.isEmpty()) {
            throw new EmptyStackException();
        }
        return storage.pop();
    }

    public T peek() {
        if (storage.isEmpty()) {
            throw new EmptyStackException();
        }
        return storage.peek();
    }

    public boolean isEmpty() {
        return storage.isEmpty();
    }

    public int size() {
        return storage.size();
    }

    // No exposure of random-access methods. The Stack interface is clean.
}
```

#### Common Misconceptions

1. **"Inheritance is the primary mechanism for code reuse in OOP."**
   Composition is generally superior for code reuse. Inheritance is primarily for type relationships. Overusing inheritance leads to rigid, brittle hierarchies.

2. **"Deep inheritance hierarchies are a sign of good OOP."**
   Deep hierarchies are almost always a design problem. They are hard to reason about, hard to test, and tightly couple all levels. Prefer shallow hierarchies (no more than 2-3 levels deep) or use interfaces.

3. **"If a geometry textbook says Square is-a Rectangle, that means Square should extend Rectangle in code."**
   The behavioral contract matters, not the mathematical definition. In code, inheritance must satisfy LSP. The Square-Rectangle problem shows that mathematical is-a does not always map to behavioral is-a.

4. **"You should always use inheritance to add behavior to a class you don't own."**
   You cannot always subclass (e.g., `final` classes). Even when you can, the Decorator pattern (composition) is usually preferable because it does not couple you to the superclass's internals.

5. **"Protected fields are fine — subclasses need to access them."**
   Exposing protected fields weakens encapsulation at the class hierarchy level. Prefer `protected` methods (or `package-private`) that provide controlled access, and keep fields `private`.

#### Interview Questions

**Q1: What is the Liskov Substitution Principle and why does it matter for inheritance?**

Model answer: LSP states that objects of a subtype must be substitutable for objects of their supertype without altering the correctness of the program. In practical terms, a subclass must honor all the behavioral contracts of the superclass: it cannot strengthen preconditions (require more from callers), weaken postconditions (deliver less), or throw new exceptions that callers do not expect. It matters because if LSP is violated, code that uses the parent type will break when given the subtype, which defeats the entire purpose of polymorphism and inheritance. The Square-Rectangle problem is the canonical example: `Square` violates LSP because it changes the behavior of `setWidth` in a way that breaks code written against `Rectangle`.

---

**Q2: "Favor composition over inheritance" — what does this mean in practice?**

Model answer: It means that when you need to reuse behavior, your first choice should be to hold a reference to an object that has that behavior (composition) rather than extending its class (inheritance). Practical example: instead of `Stack extends Vector`, use `Stack` with a private `Deque` field. Composition is preferred because it avoids inheriting an entire public API you did not want, keeps changes in the composed object from silently affecting the outer object, and allows the composed implementation to be swapped at runtime. Inheritance is still appropriate when there is a genuine type relationship that satisfies LSP and you want polymorphic substitution.

---

**Q3: When is inheritance justified over composition?**

Model answer: Inheritance is justified when (1) there is a genuine "is-a" behavioral relationship that satisfies LSP — not just a mathematical or conceptual one; (2) you want polymorphic substitution, meaning callers should be able to use a `Dog` anywhere an `Animal` is expected; and (3) the shared behavior belongs in a common base and is unlikely to change in ways that would break subclasses. The Template Method pattern is a canonical justified use of inheritance: a base class defines an algorithm's skeleton and subclasses override specific steps. Even here, many modern codebases prefer the Strategy pattern (composition) to achieve the same result without the coupling.

---

**Q4: What is the fragile base class problem?**

Model answer: The fragile base class problem occurs when changes to a superclass break subclasses that were written correctly at the time. Because subclasses depend on the internal implementation details of the superclass (which methods call which other methods, what state changes happen in what order), an implementation change in the superclass can silently break the subclass even without changing the public interface. For example, if a base class `add` method internally calls `addAll`, and a subclass overrides both to count additions, a call to `addAll` will double-count because `addAll` calls `add` internally. Declaring methods `final` in the superclass (to prevent overriding of implementation methods) and depending on interfaces rather than concrete base classes mitigates this problem.

---

**Q5: How do you decide between an abstract class and an interface for a type hierarchy?**

Model answer: Use an interface when you are defining a contract that multiple unrelated types will implement — interfaces express capability without constraining the type hierarchy. Use an abstract class when you have a true type hierarchy (a genuine "is-a" relationship) and there is meaningful shared implementation that all subtypes should inherit. In most service-layer designs, interfaces are the right choice because they decouple consumers from implementation details. Abstract classes shine in the Template Method pattern where an algorithm's skeleton is shared but specific steps vary. When in doubt, start with an interface — it is easier to add an abstract class layer underneath later than to break existing callers by introducing one.

---

### 1.4 Polymorphism

#### Theory

**Precise definition:** Polymorphism (Greek: "many forms") is the ability of different objects to respond to the same message (method call) in different ways. It allows code to be written against an abstraction and have the correct behavior automatically selected at runtime based on the actual type of the object.

Java supports two forms:

1. **Compile-time polymorphism (method overloading):** Multiple methods with the same name but different parameter signatures. The correct method is resolved at compile time by the compiler.
2. **Runtime polymorphism (method overriding):** A subclass provides a specific implementation of a method declared in its superclass or interface. The correct method is resolved at runtime by the JVM via dynamic dispatch.

Runtime polymorphism is the more powerful and design-relevant form. It is the mechanism that makes the Open/Closed Principle achievable: you can add new behavior (new types) without modifying existing code that uses the abstraction.

The power of runtime polymorphism comes from the fact that the **caller does not need to know the concrete type**. This is what allows you to loop over a `List<Shape>` and call `draw()` on each, getting the right behavior for circles, rectangles, and triangles — without a single `instanceof` check or `if-else` chain.

#### Java Code Example

**Compile-time polymorphism (method overloading):**

```java
public class Calculator {

    // Same name, different parameter types — resolved at compile time.
    public int add(int a, int b) {
        return a + b;
    }

    public double add(double a, double b) {
        return a + b;
    }

    public int add(int a, int b, int c) {
        return a + b + c;
    }
}
```

**Runtime polymorphism — the wrong way (using instanceof):**

```java
// This code is the anti-pattern. Every time a new shape is added,
// this method must be modified. It is Open for modification, Closed for extension —
// exactly the opposite of what we want.
public class DrawingEngine {

    public void draw(Object shape) {
        if (shape instanceof Circle) {
            ((Circle) shape).drawCircle();
        } else if (shape instanceof Rectangle) {
            ((Rectangle) shape).drawRect();
        } else if (shape instanceof Triangle) {
            ((Triangle) shape).drawTriangle();
        }
        // Adding a Pentagon requires modifying this method. Bad.
    }
}
```

**Runtime polymorphism — the right way:**

```java
// The abstraction. All shapes know how to draw themselves.
public interface Shape {
    void draw();
    double area();
    String describe();
}

public class Circle implements Shape {
    private final double radius;
    private final Point center;

    public Circle(Point center, double radius) {
        if (radius <= 0) throw new IllegalArgumentException("Radius must be positive");
        this.center = Objects.requireNonNull(center);
        this.radius = radius;
    }

    @Override
    public void draw() {
        System.out.printf("Drawing circle at (%s) with radius %.2f%n", center, radius);
    }

    @Override
    public double area() {
        return Math.PI * radius * radius;
    }

    @Override
    public String describe() {
        return String.format("Circle[center=%s, radius=%.2f]", center, radius);
    }
}

public class Rectangle implements Shape {
    private final int width;
    private final int height;
    private final Point topLeft;

    public Rectangle(Point topLeft, int width, int height) {
        if (width <= 0 || height <= 0) throw new IllegalArgumentException("Dimensions must be positive");
        this.topLeft = Objects.requireNonNull(topLeft);
        this.width = width;
        this.height = height;
    }

    @Override
    public void draw() {
        System.out.printf("Drawing rectangle at (%s), %dx%d%n", topLeft, width, height);
    }

    @Override
    public double area() {
        return (double) width * height;
    }

    @Override
    public String describe() {
        return String.format("Rectangle[topLeft=%s, width=%d, height=%d]", topLeft, width, height);
    }
}

// New shape added later — zero changes to DrawingEngine or any existing code.
public class Triangle implements Shape {
    private final Point a, b, c;

    public Triangle(Point a, Point b, Point c) {
        this.a = a; this.b = b; this.c = c;
    }

    @Override
    public void draw() {
        System.out.printf("Drawing triangle with vertices %s, %s, %s%n", a, b, c);
    }

    @Override
    public double area() {
        // Shoelace formula
        return Math.abs((b.x - a.x) * (double)(c.y - a.y)
                      - (c.x - a.x) * (double)(b.y - a.y)) / 2.0;
    }

    @Override
    public String describe() {
        return String.format("Triangle[%s, %s, %s]", a, b, c);
    }
}

// The engine. Works for any Shape, present or future.
// Adding Pentagon requires zero changes here.
public class DrawingEngine {

    public void drawAll(List<Shape> shapes) {
        shapes.forEach(Shape::draw);
    }

    public double totalArea(List<Shape> shapes) {
        return shapes.stream()
            .mapToDouble(Shape::area)
            .sum();
    }

    public void describeAll(List<Shape> shapes) {
        shapes.stream()
            .map(Shape::describe)
            .forEach(System.out::println);
    }
}

// Usage:
List<Shape> canvas = List.of(
    new Circle(new Point(0, 0), 5.0),
    new Rectangle(new Point(10, 10), 8, 4),
    new Triangle(new Point(0, 0), new Point(3, 0), new Point(0, 4))
);

DrawingEngine engine = new DrawingEngine();
engine.drawAll(canvas);                      // Each shape draws itself correctly.
System.out.println(engine.totalArea(canvas)); // Dispatches to the right area() per type.
```

**Polymorphism with a more realistic domain — discount strategy:**

```java
public interface DiscountPolicy {
    /**
     * Returns the discount amount in paise for the given order.
     * Must return a non-negative value not exceeding the order total.
     */
    long computeDiscountInPaise(Order order);
}

public class NoDiscount implements DiscountPolicy {
    @Override
    public long computeDiscountInPaise(Order order) {
        return 0L;
    }
}

public class PercentageDiscount implements DiscountPolicy {
    private final int discountPercent;  // 1–100

    public PercentageDiscount(int discountPercent) {
        if (discountPercent < 1 || discountPercent > 100) {
            throw new IllegalArgumentException("Discount percent must be 1-100");
        }
        this.discountPercent = discountPercent;
    }

    @Override
    public long computeDiscountInPaise(Order order) {
        return order.getTotalInPaise() * discountPercent / 100;
    }
}

public class FlatDiscount implements DiscountPolicy {
    private final long flatAmountInPaise;

    public FlatDiscount(long flatAmountInPaise) {
        if (flatAmountInPaise <= 0) {
            throw new IllegalArgumentException("Flat discount must be positive");
        }
        this.flatAmountInPaise = flatAmountInPaise;
    }

    @Override
    public long computeDiscountInPaise(Order order) {
        return Math.min(flatAmountInPaise, order.getTotalInPaise());
    }
}

public class OrderPricingService {

    private final DiscountPolicy discountPolicy;

    public OrderPricingService(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }

    public long computeFinalPriceInPaise(Order order) {
        long discount = discountPolicy.computeDiscountInPaise(order);
        return order.getTotalInPaise() - discount;
    }
}
```

#### Common Misconceptions

1. **"Polymorphism requires inheritance."**
   In Java, interface-based polymorphism does not require class inheritance. A `Circle` and a `Rectangle` can both implement `Shape` without sharing a common ancestor class.

2. **"Method overloading and overriding are both forms of polymorphism and are equally important."**
   Both are polymorphism, but they have very different roles. Overriding (runtime polymorphism) is a core design tool. Overloading is a convenience mechanism. Do not confuse them in a design discussion.

3. **"Using instanceof is just a pragmatic form of polymorphism."**
   `instanceof` checks defeat the purpose of polymorphism. They indicate that the caller knows about concrete types, which breaks the abstraction. The correct response to needing type-specific behavior is a virtual method (or the Visitor pattern for more complex dispatch).

4. **"Polymorphism is just about method calls looking the same."**
   The syntactic uniformity is the mechanism. The design value is that callers are decoupled from concrete types, making the system extensible without modification.

5. **"More polymorphism always leads to better code."**
   Polymorphism adds indirection. When there is genuinely only one behavior and no extensibility requirement, a simple method with no overriding is cleaner. Apply polymorphism where variability exists or is likely.

#### Interview Questions

**Q1: What is the difference between compile-time and runtime polymorphism? Give an example of each.**

Model answer: Compile-time polymorphism (static dispatch) is resolved by the compiler based on the declared types of the arguments — method overloading is the primary example. The compiler picks which `add(int, int)` vs `add(double, double)` to call based on the call site. Runtime polymorphism (dynamic dispatch) is resolved by the JVM at runtime based on the actual type of the object — method overriding through interfaces or inheritance. When you call `shape.draw()` on a variable declared as `Shape`, the JVM looks up the method on the actual object at runtime, not on the declared type, and invokes `Circle.draw()` or `Rectangle.draw()` accordingly.

---

**Q2: You see a method with a long if-else chain of instanceof checks. How do you refactor it?**

Model answer: I replace the instanceof chain with a polymorphic method. I identify the varying behavior (what is different for each type), add that behavior as a method to the shared interface or base class, and implement it in each concrete class. The if-else chain in the caller becomes a single virtual method call. If the behavior cannot be added to the class (e.g., it is a third-party class, or it would violate the single responsibility principle by adding unrelated rendering logic to a domain object), I use the Visitor pattern, which externalizes the double dispatch while still eliminating the instanceof chain.

---

**Q3: Can you achieve polymorphism without inheritance?**

Model answer: Yes, through interfaces. In Java, two completely unrelated classes can implement the same interface and be used polymorphically wherever that interface is expected. For example, `String` and `Integer` both implement `Comparable` with no shared class ancestor (beyond `Object`), and both can be used with `Collections.sort`. Duck typing in dynamic languages achieves polymorphism with neither inheritance nor interfaces. In Java, interface-based polymorphism is generally preferred over class inheritance-based polymorphism because it does not constrain the type hierarchy.

---

**Q4: What is covariant return type in Java and how does it relate to polymorphism?**

Model answer: Covariant return types, introduced in Java 5, allow an overriding method to return a more specific type than the method it overrides. For example, if a base class declares `Animal create()`, a subclass can override it with `Dog create()` — `Dog` is a covariant return type because `Dog` is a subtype of `Animal`. This is valid because any caller expecting an `Animal` will still work correctly when given a `Dog`. It relates to polymorphism in that it enables more specific return types in subclasses while preserving the subtype contract, and it enables a form of the Factory Method pattern where each subclass factory returns its specific type without requiring casts by callers.

---

**Q5: Explain dynamic dispatch. How does the JVM decide which method to call when you invoke a virtual method?**

Model answer: Dynamic dispatch is the mechanism by which the JVM resolves a virtual method call at runtime based on the actual type of the object, rather than the declared type of the reference variable. Each object in Java carries a pointer to its class's method table (vtable). When a virtual method is invoked, the JVM dereferences the object's class pointer, looks up the method in the vtable by method signature, and calls the implementation found there. This is why `shape.draw()` calls `Circle.draw()` when `shape` holds a `Circle` instance, regardless of the declared type of `shape`. Static methods, private methods, and final methods are not subject to dynamic dispatch — they are resolved at compile time through static dispatch.

---

## 2. Clean Code Principles

Writing code that other engineers can read, understand, and modify without your presence is not a soft skill — it is a core engineering responsibility. Senior engineers are expected to produce code that is self-evident, maintainable, and has a low cognitive load for reviewers.

The guiding philosophy: **code is read far more than it is written.** Optimize for the reader.

---

### 2.1 Naming Conventions

#### The Core Rule

A name should tell you why something exists, what it does, and how it is used — without requiring a comment to explain it.

If you feel compelled to add a comment to explain a name, the name is wrong.

#### Variable Names

**Bad:**

```java
int d;           // elapsed time in days
List<User> l;    // active users
String s;        // customer email address
boolean flag;    // whether account is active
```

**Good:**

```java
int elapsedDays;
List<User> activeUsers;
String customerEmailAddress;
boolean isAccountActive;
```

**Scope rule:** The length of a name should be proportional to the size of its scope. A loop counter `i` in a three-line loop is fine. A field named `i` in a class is not.

```java
// Acceptable — tiny scope, purpose is obvious from context:
for (int i = 0; i < users.size(); i++) { ... }

// Not acceptable — field with a large scope:
public class UserManager {
    private List<User> u;  // What is u? Never use single letters for fields.
}
```

#### Method Names

Methods should be named as verb phrases that describe the action performed.

**Bad:**

```java
public Order process(long id) { ... }    // Process what? Returns what?
public boolean check(User user) { ... }  // Check what?
public void doStuff() { ... }            // Never use vague verbs.
public Data getData() { ... }            // What data?
```

**Good:**

```java
public Order fulfillOrder(long orderId) { ... }
public boolean isEligibleForLoyaltyProgram(User user) { ... }
public void archiveExpiredSessions() { ... }
public UserProfile fetchUserProfileById(String userId) { ... }
```

**Boolean methods** should read as predicates:

```java
// Bad:
public boolean active(User user) { ... }
public boolean check(Order order) { ... }

// Good:
public boolean isActive(User user) { ... }
public boolean hasOutstandingBalance(Order order) { ... }
public boolean canProcessRefund(Order order) { ... }
```

#### Class Names

Class names should be nouns or noun phrases. Avoid generic suffixes like `Manager`, `Helper`, `Util`, `Processor`, or `Handler` unless the class genuinely is a manager/handler (e.g., `SessionManager` managing a session lifecycle is acceptable; `UserHelper` that does random user-related things is a design smell — what exactly does it help with?).

**Bad:**

```java
public class UserHelper { ... }      // Helper for what, specifically?
public class DataProcessor { ... }   // Processes what data, into what?
public class Misc { ... }            // Never.
public class Utils { ... }           // Where behaviors go to die.
```

**Good:**

```java
public class UserRegistrationService { ... }
public class InvoicePdfGenerator { ... }
public class OrderFulfillmentPipeline { ... }
public class EmailVerificationTokenRepository { ... }
```

#### Constants

Constants should be SCREAMING_SNAKE_CASE and named for what they represent, not what their value is.

```java
// Bad — the name repeats the value:
static final int SEVEN = 7;
static final String ABC = "ABC";

// Bad — magic number with no name:
if (user.getAge() > 18) { ... }

// Good — the name explains the business meaning:
static final int MINIMUM_LEGAL_AGE_YEARS = 18;
static final int MAX_LOGIN_ATTEMPTS = 5;
static final String DEFAULT_CURRENCY_CODE = "INR";
static final Duration SESSION_EXPIRY_DURATION = Duration.ofHours(24);
```

---

### 2.2 Function Design

#### Single Responsibility

A function should do one thing, do it well, and do it only. The "one thing" is defined at the level of abstraction the function operates at. A function that fetches data, validates it, transforms it, and persists it is doing four things.

**The flag argument smell:**

```java
// A boolean parameter that changes the function's behavior is a strong signal
// that the function is doing two things and should be two functions.
public void renderReport(boolean isDraft) {
    if (isDraft) {
        // Render draft...
    } else {
        // Render final...
    }
}

// Better: two clearly named functions.
public void renderDraftReport() { ... }
public void renderFinalReport() { ... }
```

#### Small Functions

Functions should fit on a screen without scrolling. If a function is more than 20-30 lines, it is a strong signal that it is doing too much.

**Bad — one large function doing everything:**

```java
public void placeOrder(String customerId, List<String> productIds, String couponCode) {
    // Step 1: Validate customer
    Customer customer = customerRepository.findById(customerId);
    if (customer == null) {
        throw new CustomerNotFoundException("Customer not found: " + customerId);
    }
    if (!customer.isEmailVerified()) {
        throw new UnverifiedCustomerException("Email not verified for: " + customerId);
    }

    // Step 2: Validate products
    List<Product> products = new ArrayList<>();
    for (String productId : productIds) {
        Product product = productRepository.findById(productId);
        if (product == null) {
            throw new ProductNotFoundException("Product not found: " + productId);
        }
        if (!product.isInStock()) {
            throw new OutOfStockException("Product out of stock: " + productId);
        }
        products.add(product);
    }

    // Step 3: Apply coupon
    long totalPaise = products.stream().mapToLong(Product::getPriceInPaise).sum();
    if (couponCode != null && !couponCode.isBlank()) {
        Coupon coupon = couponRepository.findByCode(couponCode);
        if (coupon == null || coupon.isExpired()) {
            throw new InvalidCouponException("Invalid or expired coupon: " + couponCode);
        }
        totalPaise = totalPaise - coupon.discountInPaise(totalPaise);
    }

    // Step 4: Create and persist order
    Order order = new Order(customer, products, totalPaise);
    orderRepository.save(order);

    // Step 5: Send confirmation email
    emailService.sendOrderConfirmation(customer.getEmail(), order);
}
```

**Good — decomposed into single-purpose methods:**

```java
public Order placeOrder(String customerId, List<String> productIds, String couponCode) {
    Customer customer = validateAndFetchCustomer(customerId);
    List<Product> products = validateAndFetchProducts(productIds);
    long totalPaise = computeTotalWithDiscount(products, couponCode);

    Order order = createAndPersistOrder(customer, products, totalPaise);
    notifyCustomer(customer, order);

    return order;
}

private Customer validateAndFetchCustomer(String customerId) {
    Customer customer = customerRepository.findById(customerId)
        .orElseThrow(() -> new CustomerNotFoundException("Customer not found: " + customerId));
    if (!customer.isEmailVerified()) {
        throw new UnverifiedCustomerException("Email not verified for: " + customerId);
    }
    return customer;
}

private List<Product> validateAndFetchProducts(List<String> productIds) {
    return productIds.stream()
        .map(this::validateAndFetchProduct)
        .collect(Collectors.toList());
}

private Product validateAndFetchProduct(String productId) {
    Product product = productRepository.findById(productId)
        .orElseThrow(() -> new ProductNotFoundException("Product not found: " + productId));
    if (!product.isInStock()) {
        throw new OutOfStockException("Product out of stock: " + productId);
    }
    return product;
}

private long computeTotalWithDiscount(List<Product> products, String couponCode) {
    long totalPaise = products.stream().mapToLong(Product::getPriceInPaise).sum();
    if (couponCode == null || couponCode.isBlank()) {
        return totalPaise;
    }
    return applyDiscount(totalPaise, couponCode);
}

private long applyDiscount(long totalPaise, String couponCode) {
    Coupon coupon = couponRepository.findByCode(couponCode)
        .orElseThrow(() -> new InvalidCouponException("Invalid coupon: " + couponCode));
    if (coupon.isExpired()) {
        throw new InvalidCouponException("Expired coupon: " + couponCode);
    }
    return totalPaise - coupon.discountInPaise(totalPaise);
}

private Order createAndPersistOrder(Customer customer, List<Product> products, long totalPaise) {
    Order order = new Order(customer, products, totalPaise);
    return orderRepository.save(order);
}

private void notifyCustomer(Customer customer, Order order) {
    emailService.sendOrderConfirmation(customer.getEmail(), order);
}
```

The second version reads like a table of contents. Each private method is independently comprehensible and testable.

#### Function Argument Count

Zero arguments is best. One is fine. Two is acceptable. Three demands justification. More than three is almost always wrong — introduce a parameter object.

```java
// Bad — seven parameters, easy to misorder:
public Order createOrder(String customerId, String productId, int quantity,
                         String couponCode, String shippingAddress,
                         String billingAddress, String paymentToken) { ... }

// Good — parameter object makes intent clear and prevents misorderings:
public Order createOrder(CreateOrderRequest request) { ... }

public class CreateOrderRequest {
    private final String customerId;
    private final String productId;
    private final int quantity;
    private final String couponCode;
    private final Address shippingAddress;
    private final Address billingAddress;
    private final String paymentToken;

    // Builder pattern to construct safely.
    public static Builder builder() { return new Builder(); }

    public static class Builder {
        // ... builder fields and methods
    }
}
```

---

### 2.3 Code Organization

#### Package Structure

Organize by feature/domain, not by technical layer. Layer-based organization (all controllers together, all services together) forces you to jump across the package tree to understand a single feature. Feature-based organization co-locates everything needed to understand and change a feature.

**Bad — layer-based (technical organization):**

```
com.example.app
  controllers/
    UserController.java
    OrderController.java
    ProductController.java
  services/
    UserService.java
    OrderService.java
    ProductService.java
  repositories/
    UserRepository.java
    OrderRepository.java
    ProductRepository.java
  models/
    User.java
    Order.java
    Product.java
```

**Good — feature-based (domain organization):**

```
com.example.app
  user/
    UserController.java
    UserService.java
    UserRepository.java
    User.java
    UserRegistrationRequest.java
    UserNotFoundException.java
  order/
    OrderController.java
    OrderService.java
    OrderRepository.java
    Order.java
    OrderItem.java
    OrderFulfillmentService.java
    InsufficientStockException.java
  product/
    ProductController.java
    ProductService.java
    ProductRepository.java
    Product.java
    ProductCatalogService.java
  shared/
    Money.java
    Address.java
    PagedResult.java
```

**Class organization (ordering within a file):**

1. Static constants
2. Static fields
3. Instance fields
4. Constructors
5. Public methods
6. Package-private / protected methods
7. Private methods
8. Inner classes / enums

#### Avoiding Magic Numbers and Strings

```java
// Bad:
if (user.getAge() >= 18 && user.getAge() <= 65) { ... }
if (response.getStatus() == 429) { ... }
Thread.sleep(5000);

// Good:
private static final int MINIMUM_ELIGIBLE_AGE = 18;
private static final int MAXIMUM_ELIGIBLE_AGE = 65;
private static final int HTTP_STATUS_TOO_MANY_REQUESTS = 429;
private static final Duration RETRY_BACKOFF_DURATION = Duration.ofSeconds(5);

if (user.getAge() >= MINIMUM_ELIGIBLE_AGE && user.getAge() <= MAXIMUM_ELIGIBLE_AGE) { ... }
if (response.getStatus() == HTTP_STATUS_TOO_MANY_REQUESTS) { ... }
Thread.sleep(RETRY_BACKOFF_DURATION.toMillis());
```

---

### 2.4 Comments vs Self-Documenting Code

#### The Hierarchy of Documentation

Code documentation quality, from best to worst:

1. **Self-documenting code:** Names and structure are so clear that no explanation is needed.
2. **Javadoc on public API:** Explains the contract — what parameters mean, what is returned, what exceptions are thrown, what preconditions exist.
3. **Explanatory comments:** Explains *why* something unusual is done when the code cannot.
4. **Redundant comments:** Repeats what the code already says. Noise.
5. **Misleading comments:** Comments that are wrong or out of date. Actively harmful.

#### When Comments Are Wrong

```java
// Bad — the comment merely restates the code:
// Increment i by 1
i++;

// Bad — the comment explains what the code does; a better name would eliminate the need:
// Check if the user is over 18
if (user.getAge() > 18) { ... }

// Better — the method name is the comment:
if (isAdult(user)) { ... }

// Bad — commented-out code. Delete it. Version control preserves history.
// userService.sendWelcomeEmail(user);  // disabled until marketing approves template
// orderService.applyLoyaltyPoints(order);
```

#### When Comments Are Right

**Explaining the "why" when the "what" is obvious but the reasoning is not:**

```java
// We intentionally do not validate the signature on health-check responses.
// The health-check endpoint is public and high-frequency; adding signature
// validation here caused a 15% latency increase in staging (2024-03 load test).
// If this endpoint is ever exposed externally, revisit this decision.
if (isHealthCheckRequest(request)) {
    return processWithoutAuthentication(request);
}
```

**Documenting non-obvious algorithm choices:**

```java
// Using Blowfish (bcrypt) rather than SHA-256 because bcrypt is intentionally slow,
// which provides resistance to brute-force attacks on the stored hashes.
// Cost factor 12 was chosen based on benchmarks: ~250ms per hash on our reference hardware,
// which is acceptable for login but prohibitive for bulk attacks.
return BCrypt.hashpw(rawPassword, BCrypt.gensalt(12));
```

**Warning about non-obvious constraints:**

```java
// This method is called from a Hibernate @PrePersist callback.
// Do NOT call any repository methods here — doing so will cause a recursive
// flush that corrupts the transaction. See ticket ENG-4421.
@PrePersist
private void onPrePersist() {
    this.createdAt = Instant.now();
    this.version = UUID.randomUUID().toString();
}
```

**Public API Javadoc (always required for public methods):**

```java
/**
 * Transfers the specified amount from the source account to the destination account.
 *
 * <p>Both accounts must belong to the same currency. The transfer is atomic — either
 * both debits and credits succeed, or neither does. The operation is idempotent with
 * respect to the provided {@code idempotencyKey}: a second call with the same key
 * returns the result of the first call without performing a new transfer.
 *
 * @param sourceAccountId      the account to debit; must exist and have sufficient balance
 * @param destinationAccountId the account to credit; must exist
 * @param amountInPaise        the amount to transfer; must be positive
 * @param idempotencyKey       a unique identifier for this transfer request; used for deduplication
 * @return a {@link TransferResult} containing the transaction ID and timestamps
 * @throws AccountNotFoundException    if either account does not exist
 * @throws InsufficientFundsException  if the source account has insufficient balance
 * @throws CurrencyMismatchException   if the accounts have different currencies
 * @throws IllegalArgumentException    if {@code amountInPaise} is non-positive
 */
public TransferResult transferFunds(String sourceAccountId, String destinationAccountId,
                                    long amountInPaise, String idempotencyKey) {
    // ...
}
```

#### Messy vs Clean Code — Full Example

**Messy:**

```java
public class UM {
    private List<Object[]> d = new ArrayList<>();

    public boolean check(String e, String p) {
        for (Object[] u : d) {
            if (u[0].equals(e) && u[1].equals(hashP(p))) {
                return true;
            }
        }
        return false;
    }

    public void add(String e, String p) {
        // check if exists
        for (Object[] u : d) {
            if (u[0].equals(e)) throw new RuntimeException("exists");
        }
        d.add(new Object[]{e, hashP(p)});
    }

    private String hashP(String p) {
        return Integer.toHexString(p.hashCode()); // TODO: use real hash
    }
}
```

**Clean:**

```java
/**
 * Manages user credentials: registration and authentication.
 *
 * <p>Passwords are stored as bcrypt hashes with cost factor 12.
 * This class is thread-safe.
 */
public class UserCredentialService {

    private final UserCredentialRepository credentialRepository;
    private final PasswordHasher passwordHasher;

    public UserCredentialService(UserCredentialRepository credentialRepository,
                                 PasswordHasher passwordHasher) {
        this.credentialRepository = Objects.requireNonNull(credentialRepository);
        this.passwordHasher = Objects.requireNonNull(passwordHasher);
    }

    /**
     * Registers a new user with the given email and password.
     *
     * @throws DuplicateEmailException if an account with this email already exists
     * @throws IllegalArgumentException if email or password is blank
     */
    public void registerUser(String emailAddress, String rawPassword) {
        validateRegistrationInputs(emailAddress, rawPassword);
        ensureEmailIsNotTaken(emailAddress);

        String hashedPassword = passwordHasher.hash(rawPassword);
        credentialRepository.save(new UserCredential(emailAddress, hashedPassword));
    }

    /**
     * Returns true if the given email/password combination is correct.
     */
    public boolean authenticateUser(String emailAddress, String rawPassword) {
        return credentialRepository.findByEmail(emailAddress)
            .map(credential -> passwordHasher.matches(rawPassword, credential.getHashedPassword()))
            .orElse(false);
    }

    private void validateRegistrationInputs(String emailAddress, String rawPassword) {
        if (emailAddress == null || emailAddress.isBlank()) {
            throw new IllegalArgumentException("Email address must not be blank");
        }
        if (rawPassword == null || rawPassword.isBlank()) {
            throw new IllegalArgumentException("Password must not be blank");
        }
    }

    private void ensureEmailIsNotTaken(String emailAddress) {
        if (credentialRepository.existsByEmail(emailAddress)) {
            throw new DuplicateEmailException("An account with this email already exists: " + emailAddress);
        }
    }
}
```

**Differences:**

| Dimension | Messy | Clean |
|---|---|---|
| Class name | `UM` — meaningless | `UserCredentialService` — self-describing |
| Field name | `d` — unknown purpose | `credentialRepository` — exactly what it is |
| Method names | `check`, `add` — vague verbs | `authenticateUser`, `registerUser` — precise actions |
| Password hashing | `Integer.toHexString(hashCode())` — broken security | Injected `PasswordHasher` — proper dependency |
| Error handling | `throw new RuntimeException("exists")` | `throw new DuplicateEmailException(...)` — typed, informative |
| Testability | Cannot test without instantiating the whole class | Dependencies injected — mockable and testable |
| Javadoc | None | Present on all public methods |

---

## 3. How to Approach LLD Problems

LLD interviews are not a test of your ability to recall design patterns. They are a test of your ability to take an ambiguous problem and, through structured thinking, arrive at a design that is correct, extensible, and clearly communicated.

Most candidates fail LLD rounds not because they do not know the patterns, but because they jump to code before understanding the problem, produce a design that does not satisfy the requirements, or cannot articulate why they made the choices they made.

This section gives you a repeatable framework that works across all LLD problems.

---

### 3.1 The Five-Stage Framework

The framework is not a rigid script. It is a checklist of concerns that must be addressed in roughly this order. Experienced engineers will iterate within stages and move back as understanding deepens.

#### Stage 1: Requirements Clarification (5-10 minutes)

Before writing a single class name, you must understand what you are building. Ask questions to convert an ambiguous problem statement into a set of concrete, scoped requirements.

Output: A written list of functional requirements (what the system does) and non-functional requirements (constraints and quality attributes).

#### Stage 2: Identify Core Entities and Their Relationships (5 minutes)

Extract the nouns from your requirements. These are your candidate classes. Identify the relationships between them (association, aggregation, composition, inheritance).

Output: An entity list with brief descriptions and a rough relationship sketch.

#### Stage 3: Define Interfaces and Contracts (5-10 minutes)

For each entity, decide what it exposes (its public interface) and what it hides. Define the interfaces that decouple key components. This is where you apply the OOP principles from Section 1.

Output: Interface and class skeletons with method signatures.

#### Stage 4: Implement Core Logic (15-20 minutes)

Fill in the implementations, starting with the highest-value or highest-risk paths. Do not write boilerplate first. Write the logic that tests whether your design actually solves the problem.

Output: Working Java classes with core logic implemented.

#### Stage 5: Review and Iterate (5 minutes)

Apply SOLID principles as a checklist. Ask: where is this brittle? Where will this break when requirements change? What have I assumed that might not hold?

Output: A list of known limitations and suggested improvements.

---

### 3.2 Requirements Clarification Strategy

Never accept a problem statement at face value. Interviewers intentionally under-specify. Your questions demonstrate system design thinking.

#### Categories of Questions to Ask

**Scope questions — establish what is in and what is out:**

```
"Should the system support multiple concurrent users, or is this a single-user system?"
"Should I design for persistence (data survives a restart), or is in-memory sufficient for now?"
"Are there any integrations with external systems I should be aware of?"
"Is the API for internal use only, or will it be exposed to third-party clients?"
```

**Scale questions — inform data structure and architecture choices:**

```
"What is the expected number of users / orders / products at launch and in one year?"
"What are the read-to-write ratios? Is this read-heavy or write-heavy?"
"Are there any strict latency requirements?"
```

**Domain-specific questions — catch hidden complexity:**

For a parking lot system:
```
"What vehicle types should the system support? Cars only, or also motorcycles, trucks, buses?"
"Are parking spots differentiated by vehicle type?"
"Is there a fee calculation requirement, or just spot assignment?"
"Should the system handle multiple floors / levels?"
"Do we need to support reserved vs. unreserved spots?"
```

For a library management system:
```
"Can a book have multiple copies? Should we track individual copies or just inventory counts?"
"Should the system support reservations for books currently checked out?"
"Is there a fine calculation requirement for late returns?"
"Should the system handle multiple branches?"
```

**Constraint clarification — avoid assumptions that break the design:**

```
"Should I assume the happy path only, or should I also design for error cases?"
"Are there any fields that are mandatory vs optional on these entities?"
"What happens when [specific edge case] occurs?"
```

#### What Not to Ask

Avoid questions that are purely cosmetic or that do not affect your design:

```
"Should the class be named User or Customer?" — You decide; name it appropriately.
"Should I use Spring Boot?" — Only relevant if the interviewer specified a framework.
"Do you want me to write getters and setters?" — Yes, unless stated otherwise.
```

---

### 3.3 Domain Modeling Approach

Domain modeling is the process of identifying the objects in your system and defining their data, behavior, and relationships.

#### Step 1: Identify Entities (Nouns)

Read your requirements and extract the significant nouns. These become your candidate classes.

Example: "Design a movie ticket booking system."

Nouns: Movie, Show, Screen, Seat, Booking, User, Theatre, Ticket, Payment, CancellationPolicy.

Filter: Not every noun becomes a class. Some are attributes of other entities. `SeatNumber` might be an attribute of `Seat`, not its own class. `ShowTime` might be an attribute of `Show`.

#### Step 2: Identify Behaviors (Verbs)

Extract the significant verbs and assign them to entities. A verb belongs to the entity it acts upon or originates from.

"A user searches for shows." → `ShowSearchService.search(SearchCriteria)`
"A user books a seat." → `BookingService.bookSeat(UserId, ShowId, SeatId)`
"A show has available seats." → `Show.getAvailableSeats()`
"A booking can be cancelled." → `Booking.cancel()`

#### Step 3: Define Relationships

| Relationship | Meaning | Example |
|---|---|---|
| Association | A uses B | `BookingService` uses `PaymentGateway` |
| Aggregation | A has-a B (B exists independently) | `Theatre` has `Screen`s; screens exist without the theatre object |
| Composition | A owns B (B's lifecycle tied to A) | `Order` owns `OrderItem`s; items don't exist outside an order |
| Inheritance | A is-a B | `CreditCardPayment` is-a `Payment` |
| Implementation | A implements B | `RazorpayGateway` implements `PaymentGateway` |

#### Step 4: Define Entity Invariants

For each entity, ask: what must always be true about this object? These become constructor validations and method preconditions.

```java
// Seat invariants:
// - seatNumber must be positive and within screen capacity
// - A seat's status can only transition: AVAILABLE → RESERVED → BOOKED or AVAILABLE → RESERVED → AVAILABLE
// - A seat cannot be booked if it was not first reserved

// Show invariants:
// - showTime must be in the future at time of creation
// - screenId must reference a valid screen
// - totalSeats must match the screen's capacity
// - A show cannot start if it has no confirmed bookings (business rule, not a code invariant)
```

#### Step 5: Identify Abstractions

Look for places where the behavior varies: payment methods, notification channels, discount policies, pricing strategies, cancellation rules. Each variation point is a candidate for an interface.

```
PaymentGateway — varies by provider (Stripe, Razorpay, PayPal)
NotificationSender — varies by channel (email, SMS, push)
PricingPolicy — varies by show type, timing, seat class
CancellationPolicy — varies by how far before the show the cancellation occurs
```

---

### 3.4 Iterative Design Mindset

A common mistake is trying to arrive at a perfect design in one pass. Professional software design is iterative. The first design is a hypothesis; you refine it as you learn more.

#### The Iteration Cycle

```
Draft → Challenge → Refine
```

**Draft:** Produce a first version quickly, accepting that it is imperfect.

**Challenge with "what if" questions:**

```
"What if a third payment provider needs to be added?"
→ Is that a one-line change (new implementation) or does it require modifying existing classes?

"What if the fee calculation formula changes?"
→ Is the formula isolated to one class, or scattered throughout the codebase?

"What if this system needs to be used by 10 concurrent threads?"
→ Are there shared mutable fields that would require synchronization?

"What if a show can have multiple pricing tiers (e.g., standard, premium, VIP)?"
→ Does the current design accommodate this, or would it require a major restructure?
```

**Refine:** Make targeted changes to address the weaknesses surfaced by the challenges.

#### Communicating Your Design in an Interview

Do not code in silence. Talk through your thinking:

1. State what you are about to decide and why.
2. Name the alternatives you considered.
3. Explain why you chose one alternative over the others.
4. Acknowledge the limitations of your choice.

Example: "I'm going to model `PaymentGateway` as an interface rather than an abstract class, because we need to support both Stripe and Razorpay which have completely different SDKs, so there's no shared implementation to extract. The trade-off is that each implementation is fully independent, which means if we ever need to add retry logic across all gateways, we'd have to add it to each one. To handle that, I'd introduce an abstract decorator or a circuit-breaker wrapper at a later stage."

This narration demonstrates that you are not just writing code — you are making reasoned engineering decisions.

#### Design Quality Checklist

Before finalizing your design, ask:

- [ ] Does each class have a single, clearly stated responsibility?
- [ ] Are all dependencies on abstractions (interfaces), not concrete classes?
- [ ] Can I add a new variant (new payment type, new notification channel) without modifying existing classes?
- [ ] Are invariants enforced in constructors and methods?
- [ ] Are the class and method names self-describing?
- [ ] Is the design testable — can I replace all external dependencies with test doubles?
- [ ] Have I handled the obvious error cases (null inputs, not-found entities, invalid states)?
- [ ] Does the design match the level of complexity of the problem, or have I over-engineered?

---

## 4. Module Assignments

These three exercises are designed to be completed before proceeding to Module 01. Each exercise targets a specific weakness engineers commonly carry into Senior Engineer interviews.

---

### Assignment 1: Encapsulation Audit and Repair

**Objective:** Identify and fix encapsulation violations in a provided class.

**The broken class:**

```java
public class ShoppingCart {
    public String userId;
    public ArrayList<String> itemIds = new ArrayList<>();
    public double totalPrice;
    public String couponCode;
    public boolean isCheckedOut;

    public void addItem(String itemId) {
        itemIds.add(itemId);
        // Caller is expected to update totalPrice after calling this.
    }

    public void checkout() {
        isCheckedOut = true;
    }
}
```

**Your tasks:**

1. List every encapsulation violation in `ShoppingCart` with a one-sentence explanation of why each is a violation.

2. Rewrite the class so it:
   - Enforces all relevant invariants (a cart cannot be modified after checkout; `totalPrice` is always consistent with the items; `userId` is mandatory and immutable).
   - Exposes only the operations that make business sense.
   - Uses `Money` or `long` for prices (explain your choice).
   - Handles the `couponCode` field appropriately — when can it be applied, and what prevents it from being applied twice?

3. Write a brief note (3-5 sentences) explaining the relationship between encapsulation and unit testability, with specific reference to the changes you made.

**Deliverable:** A fully rewritten `ShoppingCart.java` with inline comments explaining each design decision.

---

### Assignment 2: Design a Polymorphic Notification System

**Objective:** Practice abstraction, polymorphism, and the Open/Closed Principle together.

**Problem statement:**

Design a notification system for an e-commerce platform. The system must support:
- Email notifications (to/cc/bcc fields, subject, HTML body)
- SMS notifications (phone number, plain text body with 160-character limit)
- In-app push notifications (device token, title, body, action URL)

The system must be extensible: adding WhatsApp notifications in the future should require zero changes to existing classes.

The system also needs a `NotificationDispatcher` that accepts a `List<Notification>` and dispatches each one to the appropriate channel.

**Your tasks:**

1. Define the `Notification` abstraction (interface or abstract class — justify your choice).

2. Implement all three notification types with appropriate validation in their constructors.

3. Implement `NotificationDispatcher`. It should not use `instanceof` or type-checking of any kind.

4. Add a `RateLimitedNotificationSender` that wraps any `NotificationSender` and enforces a maximum of N sends per minute. This should use the Decorator pattern — the wrapper should work with any sender without knowing its concrete type.

5. Write a `main` method that demonstrates the full system: build a list of mixed notification types and dispatch them.

**Constraints:**

- No `instanceof` checks anywhere in your solution.
- The addition of `WhatsAppNotification` should require creating exactly one new class (the implementation). Describe in a comment at the top of the file what changes, if any, would be required to `NotificationDispatcher`.

**Deliverable:** All classes in a single file (for simplicity) with a working `main` method.

---

### Assignment 3: Apply the LLD Framework to a Parking Lot System

**Objective:** Practice the full five-stage LLD framework on a classic problem.

**Problem statement:**

Design a parking lot management system. A parking lot has multiple floors. Each floor has multiple spots. Spots are of different types: compact (for motorcycles and small cars), standard (for standard cars), and large (for trucks and buses). The system should be able to find an available spot for a given vehicle type, assign the vehicle to that spot, and release the spot when the vehicle leaves. Provide a ticket when a vehicle enters and calculate the fee when it exits.

**Stage 1 — Requirements clarification:**

Write down the questions you would ask the interviewer. Then write your own answers, making reasonable assumptions for a mid-sized private parking lot (200-500 spots across 3-5 floors). State each assumption explicitly.

**Stage 2 — Entity identification:**

List all entities. For each entity, specify:
- Its core data attributes
- Its invariants (what must always be true)
- Its primary responsibilities (behaviors it owns)

**Stage 3 — Interface and contract definition:**

Identify the variation points in your design (where the behavior might vary across implementations). Define the interfaces that represent these variation points. Write the Java interface/abstract class stubs with Javadoc.

**Stage 4 — Core implementation:**

Implement at minimum:
- `ParkingLot` — the entry point for the system
- `ParkingFloor` — manages spots on one floor
- `ParkingSpot` — represents a single spot with its state machine (AVAILABLE → OCCUPIED → AVAILABLE)
- `Ticket` — issued on entry, contains entry time and spot assignment
- `FeeCalculator` interface with at least two implementations: `HourlyFeeCalculator` and `FlatRateFeeCalculator`
- `ParkingLotService` — the application service that orchestrates entry and exit

All `ParkingSpot` state transitions should be guarded: it must be impossible to double-assign a spot or release an unoccupied spot.

**Stage 5 — Review:**

Write a section titled "Known Limitations and Improvements" with at least five specific limitations of your current design and the approach you would take to address each one in a production system.

**Constraints:**

- All core logic must be in Java.
- No `instanceof` checks for vehicle-to-spot compatibility matching.
- Concurrency is out of scope, but note where you would add synchronization if it were in scope.
- Fee calculation must be separated from the `ParkingLot` and `Ticket` classes — it must be injectable.

**Deliverable:** A complete Java implementation split into logical files, plus the Stage 1 requirements document and Stage 5 review section as comments or a separate section.

---

## Further Reading

- *Clean Code* — Robert C. Martin. Chapters 1-5 are directly relevant to this module.
- *Effective Java, 3rd Edition* — Joshua Bloch. Items 15-25 cover encapsulation, inheritance, and interfaces with Java-specific depth.
- *Head First Design Patterns, 2nd Edition* — Freeman & Robson. Chapters 1-2 cover strategy and observer patterns as concrete applications of the OOP principles in this module.
- Oracle Java Documentation — [Interfaces and Inheritance](https://docs.oracle.com/javase/tutorial/java/IandI/index.html) for language-level reference.

---

*Module 01: SOLID Principles — next in sequence.*
