# Coupling and Cohesion

> "The goal of design is to create modules that are as independent as possible — loosely coupled and highly cohesive."
> — Robert C. Martin

Coupling and cohesion are the two most fundamental metrics for evaluating software design quality. Every design principle — SOLID, DRY, composition over inheritance — is ultimately in service of achieving low coupling and high cohesion. Understanding these concepts at depth separates a senior engineer from a mid-level one.

---

## Table of Contents

1. [What Is Coupling?](#1-what-is-coupling)
2. [Types of Coupling](#2-types-of-coupling)
3. [Measuring Coupling: Afferent vs Efferent](#3-measuring-coupling-afferent-vs-efferent)
4. [What Is Cohesion?](#4-what-is-cohesion)
5. [Types of Cohesion](#5-types-of-cohesion)
6. [The Coupling-Cohesion Law](#6-the-coupling-cohesion-law)
7. [How They Relate to SOLID](#7-how-they-relate-to-solid)
8. [Package and Module Level Coupling](#8-package-and-module-level-coupling)
9. [Java Examples: Tightly vs Loosely Coupled Systems](#9-java-examples-tightly-vs-loosely-coupled-systems)
10. [Metrics: CBO and LCOM](#10-metrics-cbo-and-lcom)
11. [Refactoring Coupled Code](#11-refactoring-coupled-code)
12. [Interview Questions](#12-interview-questions)

---

## 1. What Is Coupling?

Coupling is the degree to which one module depends on another. High coupling means a change in one module is likely to force a change in another. Low coupling means modules can change independently.

Think of coupling as the **blast radius** of a change: how many other modules must change when you modify this one?

```
Low coupling:          High coupling:

[A] --> [B]            [A] <--> [B]
                         \     /  \
[C] --> [D]               [C] <--> [D]

A change in B only        A change in B forces
affects A.                changes in A, C, D.
```

---

## 2. Types of Coupling

Coupling types are ranked from worst (most tight) to best (loosest). This ranking comes from Stevens, Myers & Constantine's foundational 1974 paper on structured design.

### Level 1: Content Coupling (Worst)

One module directly accesses or modifies the internals of another. In Java, this manifests as accessing private fields via reflection or through package-private visibility exploits.

```java
// Content coupling — OrderProcessor directly modifies Order internals
public class OrderProcessor {
    public void process(Order order) {
        // Directly accessing and modifying internal state
        order.status = "PROCESSING"; // accessing package-private field
        order.items.remove(0);       // mutating internal collection
        order.internalTimestamp = System.currentTimeMillis();
    }
}
```

Any refactoring of `Order`'s internal structure requires changes to `OrderProcessor`.

### Level 2: Common Coupling

Two modules share mutable global state. One module's write can break another module's read unpredictably.

```java
// Common coupling — shared mutable global state
public class AppConfig {
    public static int MAX_CONNECTIONS = 10;  // mutable global
    public static String DB_URL = "jdbc:mysql://localhost/db"; // mutable global
}

public class ConnectionPool {
    public Connection getConnection() {
        if (activeConnections >= AppConfig.MAX_CONNECTIONS) { // reads global
            throw new RuntimeException("Pool exhausted");
        }
        return createConnection(AppConfig.DB_URL);
    }
}

public class AdminService {
    public void reconfigurePool(int maxConn, String url) {
        AppConfig.MAX_CONNECTIONS = maxConn; // mutates global — breaks ConnectionPool
        AppConfig.DB_URL = url;
    }
}
```

### Level 3: External Coupling

Modules depend on an externally imposed format — a file format, a database schema, a wire protocol. Changes to the external format break all modules depending on it.

```java
// External coupling — both modules depend on the same database schema
public class UserService {
    public User loadUser(long id) {
        // SQL tied to schema: columns first_name, last_name, email_addr
        return jdbcTemplate.queryForObject(
            "SELECT first_name, last_name, email_addr FROM users WHERE id = ?",
            userRowMapper, id);
    }
}

public class ReportService {
    public void generateReport() {
        // Same schema dependency — rename email_addr -> email, both break
        jdbcTemplate.query("SELECT first_name, email_addr FROM users", ...);
    }
}
```

### Level 4: Control Coupling

One module controls the logic of another by passing it a flag or control code that dictates what the receiving module does.

```java
// Control coupling — the flag controls internal behavior of the callee
public class PaymentProcessor {
    public void process(Payment payment, String mode) {
        if (mode.equals("CREDIT_CARD")) {
            processCreditCard(payment);
        } else if (mode.equals("PAYPAL")) {
            processPayPal(payment);
        } else if (mode.equals("BANK_TRANSFER")) {
            processBankTransfer(payment);
        }
    }
}

// The caller must know which modes exist — callee's internals leak to caller
paymentProcessor.process(payment, "CREDIT_CARD");
```

Fix: Use polymorphism — each payment type is its own class.

### Level 5: Stamp Coupling (Data Structure Coupling)

One module passes a data structure to another, but the receiving module uses only part of it. The receiver is coupled to the whole structure.

```java
// Stamp coupling — formatAddress only needs city and country,
// but must accept the whole User object
public class EmailService {
    public String formatAddress(User user) {
        // Only uses two fields but is coupled to the entire User class
        return user.getCity() + ", " + user.getCountry();
    }
}
```

Fix: Pass only what is needed — `formatAddress(String city, String country)`.

### Level 6: Data Coupling (Best)

Modules communicate only through parameters that are primitive types or simple data objects, and all parameters are used. This is the loosest acceptable coupling.

```java
// Data coupling — only needed primitives are passed
public class TaxCalculator {
    public double calculateTax(double amount, double rate) {
        return amount * rate;
    }
}
```

### Summary Table

| Type | Description | Badness |
|------|-------------|---------|
| Content | Accesses internals of another module | Worst |
| Common | Shares mutable global state | Very Bad |
| External | Depends on external data format | Bad |
| Control | Passes flags that control callee behavior | Moderate |
| Stamp | Passes whole structure when part is needed | Moderate |
| Data | Passes only needed primitive data | Best |

---

## 3. Measuring Coupling: Afferent vs Efferent

### Afferent Coupling (Ca) — Incoming

The number of classes outside this package that depend on classes inside this package. High afferent coupling means this package is heavily used — it is responsible to many external classes. Changes here break many callers.

```
Package: com.company.utils

Used by:
  com.company.order.OrderService     <-- depends on utils
  com.company.user.UserService       <-- depends on utils
  com.company.payment.PaymentService <-- depends on utils
  com.company.report.ReportService   <-- depends on utils

Ca = 4 (4 external classes depend on this package)
```

High Ca means: "Be careful changing this. Many things will break."

### Efferent Coupling (Ce) — Outgoing

The number of classes inside this package that depend on classes outside this package. High efferent coupling means this package depends on many external packages — it will be affected by changes in many places.

```
Package: com.company.order

Depends on:
  com.company.payment.PaymentService  <-- external dependency
  com.company.inventory.StockChecker  <-- external dependency
  com.company.notification.Emailer    <-- external dependency
  com.company.utils.DateUtils         <-- external dependency

Ce = 4 (this package depends on 4 external packages)
```

High Ce means: "This package will be affected by changes in many places."

### Instability (I)

```
I = Ce / (Ca + Ce)

Range: 0 (stable, many dependents, few dependencies)
       1 (unstable, few dependents, many dependencies)
```

- Stable packages (I near 0): Should be abstract (interfaces/abstract classes), because they are hard to change
- Unstable packages (I near 1): Can be concrete, because they are easy to change

This is the **Stable Dependencies Principle**: depend in the direction of stability. Don't make stable packages depend on unstable ones.

---

## 4. What Is Cohesion?

Cohesion is the degree to which the elements inside a module belong together. A highly cohesive module does one thing and does it well. A low-cohesion module is a grab-bag of unrelated functions.

Think of cohesion as **conceptual relatedness**: do all the parts of this class contribute to a single, well-defined purpose?

```
High cohesion:             Low cohesion:

class OrderCalculator      class Utils
{                          {
  calculateSubtotal()        calculateOrderTotal()
  applyDiscount()            sendEmail()
  calculateTax()             parseDate()
  calculateShipping()        formatCurrency()
  calculateTotal()           resizeImage()
}                            validateEmail()
                           }

All methods serve one      Methods have no
coherent purpose.          conceptual relationship.
```

---

## 5. Types of Cohesion

Ranked from worst to best:

### Level 1: Coincidental Cohesion (Worst)

Parts of a module are grouped together only because they happen to be in the same file or class. There is no logical relationship.

```java
// Coincidental cohesion — a utility grab-bag
public class AppUtils {
    public static double roundToTwoDecimals(double value) { ... }
    public static String formatPhoneNumber(String phone) { ... }
    public static void sendEmail(String to, String body) { ... }
    public static Image resizeImage(Image img, int width) { ... }
    public static boolean isLeapYear(int year) { ... }
    public static Connection getDatabaseConnection() { ... }
}
// Nothing relates to anything else.
```

### Level 2: Logical Cohesion

Parts are grouped because they perform logically similar operations, but the selection is controlled by an external flag.

```java
// Logical cohesion — grouped by "communication" but dispatched by flag
public class Communicator {
    public void send(String message, String type) {
        switch (type) {
            case "EMAIL": sendEmail(message); break;
            case "SMS":   sendSms(message); break;
            case "PUSH":  sendPushNotification(message); break;
        }
    }
}
```

### Level 3: Temporal Cohesion

Parts are grouped because they happen at the same time (e.g., initialization, startup, cleanup) — not because they are logically related.

```java
// Temporal cohesion — grouped because they all run at startup
public class AppStartup {
    public void initialize() {
        loadConfiguration();    // config loading
        connectToDatabase();    // DB connection
        startMetricsServer();   // monitoring
        warmUpCache();          // cache
        sendStartupEmail();     // notification
    }
}
```

### Level 4: Procedural Cohesion

Parts are grouped because they always execute in sequence, but the individual parts do not share data.

```java
public class FileProcessor {
    public void process() {
        openFile();     // step 1
        readContents(); // step 2 — doesn't use result from step 1 directly
        closeFile();    // step 3
        sendReport();   // step 4 — unrelated to file
    }
}
```

### Level 5: Communicational Cohesion

Parts operate on the same data. Better than procedural, but they still do different things with the data.

```java
public class CustomerReport {
    private final Customer customer;

    public void printProfile() { /* uses customer */ }
    public void printOrderHistory() { /* uses customer */ }
    public void printLoyaltyPoints() { /* uses customer */ }
}
// All use customer, but serve different reporting purposes.
```

### Level 6: Sequential Cohesion

The output of one part is the input of the next. Parts are tightly related by data flow.

```java
public class OrderProcessor {
    public Order process(RawOrderRequest request) {
        Order order = parseOrder(request);       // step 1: produces Order
        Order validated = validateOrder(order);  // step 2: consumes Order, produces validated
        Order priced = applyPricing(validated);  // step 3: consumes validated, produces priced
        return priced;                           // output flows through all steps
    }
}
```

### Level 7: Functional Cohesion (Best)

Every element contributes to a single, well-defined task. The module does exactly one thing.

```java
// Functional cohesion — everything here is about calculating order totals
public class OrderTotalCalculator {
    private final TaxPolicy taxPolicy;
    private final DiscountPolicy discountPolicy;
    private final ShippingPolicy shippingPolicy;

    public OrderTotal calculate(Order order) {
        double subtotal = calculateSubtotal(order.getItems());
        double discount = discountPolicy.calculateDiscount(subtotal, order.getCustomer());
        double tax      = taxPolicy.calculateTax(subtotal - discount);
        double shipping = shippingPolicy.calculateShipping(order);
        return new OrderTotal(subtotal, discount, tax, shipping);
    }

    private double calculateSubtotal(List<OrderItem> items) {
        return items.stream()
            .mapToDouble(item -> item.getPrice() * item.getQuantity())
            .sum();
    }
}
```

### Cohesion Summary Table

| Type | Description | Quality |
|------|-------------|---------|
| Coincidental | Random grouping | Worst |
| Logical | Grouped by type, flag-dispatched | Very Poor |
| Temporal | Grouped by time of execution | Poor |
| Procedural | Grouped by sequence | Moderate |
| Communicational | Operate on same data | Good |
| Sequential | Output feeds next input | Very Good |
| Functional | One single purpose | Best |

---

## 6. The Coupling-Cohesion Law

> **Low coupling + High cohesion = Good design**

This is not optional. It is the law.

Why do they work together?

- A highly cohesive class does one thing well, which means it has few reasons to depend on other classes — this naturally produces low coupling.
- A loosely coupled class communicates through narrow interfaces — which means its internal purpose can remain well-defined — producing high cohesion.

They reinforce each other. When you see a class with low cohesion, you will almost always find high coupling nearby.

When you see tight coupling, you will almost always find a class trying to do too many things.

**The Paradox of the Utils Class**

The `Utils` class is low cohesion (coincidental) AND tends to produce high afferent coupling (everything depends on it). It is the canonical violation of both principles simultaneously.

---

## 7. How They Relate to SOLID

| SOLID Principle | Coupling/Cohesion Connection |
|----------------|------------------------------|
| SRP | High cohesion — one class, one reason to change |
| OCP | Low coupling — open for extension without modifying existing code |
| LSP | Low coupling — callers do not need to know the concrete type |
| ISP | High cohesion in interfaces — each interface has one focused role |
| DIP | Low coupling — depend on abstractions, not concretions |

SOLID is the *how*. Coupling and cohesion are the *why*. Following SOLID achieves low coupling and high cohesion.

---

## 8. Package and Module Level Coupling

Coupling is not just a class-level concern. At the package and module level:

### Package by Layer (Low Cohesion, High Coupling)

```
com.company.controllers   --> [UserController, OrderController, ProductController]
com.company.services      --> [UserService, OrderService, ProductService]
com.company.repositories  --> [UserRepo, OrderRepo, ProductRepo]
```

This structure is common but problematic. The "controller" package contains unrelated classes. Adding a new feature (say, Payments) requires touching all three packages. This is called **horizontal slicing**.

### Package by Feature (High Cohesion, Lower Coupling)

```
com.company.user          --> [UserController, UserService, UserRepo, User]
com.company.order         --> [OrderController, OrderService, OrderRepo, Order]
com.company.product       --> [ProductController, ProductService, ProductRepo, Product]
com.company.payment       --> [PaymentController, PaymentService, PaymentRepo, Payment]
```

Each package contains everything related to one feature. Adding a new feature only requires one new package. This is **vertical slicing**.

The acyclic dependencies principle states: **there must be no cycles in the package dependency graph**. Cycles create modules that cannot be understood or built in isolation.

```
// Cycle — violation of acyclic dependencies
com.company.order  depends on  com.company.payment
com.company.payment depends on com.company.user
com.company.user   depends on  com.company.order   <-- cycle!
```

---

## 9. Java Examples: Tightly vs Loosely Coupled Systems

### Tightly Coupled System

```java
// OrderService is coupled to concrete implementations of everything
public class OrderService {
    private MySQLOrderRepository orderRepository;   // concrete class
    private SmtpEmailService emailService;          // concrete class
    private StripePaymentGateway paymentGateway;    // concrete class

    public OrderService() {
        // Creates its own dependencies — tightest possible coupling
        this.orderRepository = new MySQLOrderRepository("jdbc:mysql://localhost/db");
        this.emailService = new SmtpEmailService("smtp.company.com", 587);
        this.paymentGateway = new StripePaymentGateway(System.getenv("STRIPE_KEY"));
    }

    public Order placeOrder(OrderRequest request) {
        // If any of these classes change, this class must change
        Order order = new Order(request);
        Order saved = orderRepository.save(order);
        paymentGateway.charge(request.getCardToken(), order.getTotal());
        emailService.sendOrderConfirmation(request.getEmail(), saved);
        return saved;
    }
}
```

Problems:
- Cannot test `OrderService` without a real MySQL database, SMTP server, and Stripe account
- Changing from MySQL to PostgreSQL requires modifying `OrderService`
- Cannot swap to a different payment gateway without modifying `OrderService`

### Loosely Coupled System

```java
// Abstractions — only these are known to OrderService
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(long id);
}

public interface EmailService {
    void sendOrderConfirmation(String email, Order order);
}

public interface PaymentGateway {
    PaymentResult charge(String token, double amount);
}

// OrderService knows only the abstractions
public class OrderService {
    private final OrderRepository orderRepository;
    private final EmailService emailService;
    private final PaymentGateway paymentGateway;

    // Dependencies injected — OrderService does not know or care about implementations
    public OrderService(OrderRepository orderRepository,
                        EmailService emailService,
                        PaymentGateway paymentGateway) {
        this.orderRepository = orderRepository;
        this.emailService = emailService;
        this.paymentGateway = paymentGateway;
    }

    public Order placeOrder(OrderRequest request) {
        Order order = new Order(request);
        Order saved = orderRepository.save(order);
        paymentGateway.charge(request.getCardToken(), order.getTotal());
        emailService.sendOrderConfirmation(request.getEmail(), saved);
        return saved;
    }
}

// Concrete implementations — each in its own class, easily swappable
public class MySQLOrderRepository implements OrderRepository { ... }
public class PostgreSQLOrderRepository implements OrderRepository { ... }
public class SmtpEmailService implements EmailService { ... }
public class SendGridEmailService implements EmailService { ... }
public class StripePaymentGateway implements PaymentGateway { ... }
public class PayPalPaymentGateway implements PaymentGateway { ... }

// Testing with mocks — no real infrastructure needed
@Test
void testPlaceOrder() {
    OrderRepository mockRepo = mock(OrderRepository.class);
    EmailService mockEmail = mock(EmailService.class);
    PaymentGateway mockPayment = mock(PaymentGateway.class);

    when(mockRepo.save(any())).thenReturn(new Order());
    when(mockPayment.charge(anyString(), anyDouble()))
        .thenReturn(PaymentResult.SUCCESS);

    OrderService service = new OrderService(mockRepo, mockEmail, mockPayment);
    Order result = service.placeOrder(new OrderRequest("card123", "user@example.com"));

    verify(mockEmail).sendOrderConfirmation(eq("user@example.com"), any());
    assertNotNull(result);
}
```

---

## 10. Metrics: CBO and LCOM

### CBO — Coupling Between Object Classes

CBO counts the number of other classes a class is coupled to (excluding inheritance). A class is coupled to another if it uses its methods or fields.

**Acceptable CBO:** 0-5 for most classes. Above 10 is a red flag.

```java
// CBO of OrderService = 6 (coupled to 6 distinct types)
public class OrderService {
    private OrderRepository repo;          // +1
    private EmailService email;            // +1
    private PaymentGateway payment;        // +1
    private InventoryService inventory;    // +1
    private AuditLogger logger;            // +1
    private PricingEngine pricing;         // +1
}
```

High CBO means the class will be affected by changes in many places.

### LCOM — Lack of Cohesion in Methods

LCOM measures how poorly a class's methods are related to each other through shared fields. There are several variants; the most intuitive:

**LCOM4:** Number of connected components in the graph where methods are nodes and two methods are connected if they share a field or one calls the other.

- LCOM4 = 1: Perfectly cohesive (all methods are connected)
- LCOM4 > 1: The class should be split — it contains that many distinct concepts

```java
// LCOM4 = 2 — two disconnected components
// This class should be split into two classes
public class UserManager {

    // Component 1: user authentication
    private String passwordHash;
    private int loginAttempts;

    public boolean authenticate(String password) {
        return checkPassword(password);       // uses passwordHash
    }
    private boolean checkPassword(String pw) {
        return BCrypt.verify(pw, passwordHash); // uses passwordHash
    }

    // Component 2: user profile — no connection to Component 1
    private String firstName;
    private String lastName;
    private String email;

    public String getFullName() {
        return firstName + " " + lastName; // uses firstName, lastName
    }
    public void updateEmail(String email) {
        this.email = email; // uses email
    }
}

// Split into:
public class UserAuthenticator { /* authenticate, checkPassword, passwordHash */ }
public class UserProfile { /* getFullName, updateEmail, firstName, lastName, email */ }
```

---

## 11. Refactoring Coupled Code

### Extract Class

When one class has two responsibilities (LCOM4 > 1), extract one into a new class.

```java
// Before: OrderService handles both order logic and order persistence
public class OrderService {
    private Connection connection;

    public Order createOrder(OrderRequest request) {
        Order order = buildOrder(request);
        validateOrder(order);
        applyBusinessRules(order);

        // Persistence logic mixed with business logic
        PreparedStatement stmt = connection.prepareStatement(
            "INSERT INTO orders (customer_id, total) VALUES (?, ?)");
        stmt.setLong(1, order.getCustomerId());
        stmt.setDouble(2, order.getTotal());
        stmt.execute();
        return order;
    }
}

// After: Extract persistence into its own class
public class OrderRepository {
    private final Connection connection;

    public OrderRepository(Connection connection) {
        this.connection = connection;
    }

    public Order save(Order order) {
        PreparedStatement stmt = connection.prepareStatement(
            "INSERT INTO orders (customer_id, total) VALUES (?, ?)");
        stmt.setLong(1, order.getCustomerId());
        stmt.setDouble(2, order.getTotal());
        stmt.execute();
        return order;
    }
}

public class OrderService {
    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public Order createOrder(OrderRequest request) {
        Order order = buildOrder(request);
        validateOrder(order);
        applyBusinessRules(order);
        return orderRepository.save(order);
    }
}
```

### Extract Interface

When callers depend on a concrete class, extract an interface to reduce coupling.

```java
// Before: tight coupling to SmtpEmailService
public class UserService {
    private SmtpEmailService emailService = new SmtpEmailService();

    public void registerUser(User user) {
        // ...
        emailService.sendWelcome(user.getEmail(), user.getName());
    }
}

// After: depend on interface
public interface EmailService {
    void sendWelcome(String email, String name);
}

public class SmtpEmailService implements EmailService {
    public void sendWelcome(String email, String name) { ... }
}

public class UserService {
    private final EmailService emailService; // interface, not concrete

    public UserService(EmailService emailService) {
        this.emailService = emailService;
    }

    public void registerUser(User user) {
        // ...
        emailService.sendWelcome(user.getEmail(), user.getName());
    }
}
```

---

## 12. Interview Questions

**Q1. What is coupling and why is low coupling desirable?**

Coupling is the degree of dependency between modules. Low coupling means a change in one module does not force changes in others. It is desirable because it reduces the blast radius of changes, makes modules independently testable, and allows the system to evolve without cascading rewrites.

**Q2. What are the types of coupling from worst to best?**

Content coupling (accesses internals) is worst, followed by common coupling (shared mutable state), external coupling (shared data format), control coupling (flag-based dispatch), stamp coupling (passing whole structures), and data coupling (passing only needed primitives) which is the best acceptable form.

**Q3. What is cohesion and what is the best type?**

Cohesion is the degree to which elements within a module belong together. Functional cohesion is the best type: every method in the class contributes to a single, well-defined purpose. A class with functional cohesion does exactly one thing.

**Q4. What is the difference between afferent and efferent coupling?**

Afferent coupling (Ca) counts how many external modules depend on this module — it measures responsibility. Efferent coupling (Ce) counts how many external modules this module depends on — it measures fragility. The Instability metric I = Ce / (Ca + Ce) indicates how easy it is to change a module. Low instability means many things depend on it, so change it carefully.

**Q5. How do coupling and cohesion relate to each other?**

They are complementary forces. High cohesion means the class does one focused thing, which naturally limits the number of external classes it needs to depend on — producing low coupling. Low coupling means the class communicates through narrow interfaces, which helps keep its internal purpose focused — supporting high cohesion. Violating one usually violates the other.

**Q6. What is CBO and what does a high CBO indicate?**

CBO (Coupling Between Object Classes) counts the number of distinct classes a class is coupled to. A high CBO (above 10) indicates a class is a hub — it depends on or is depended upon by many others. This is a warning sign that the class has too many responsibilities or is not using abstractions properly.

**Q7. What is LCOM and how is it used in refactoring?**

LCOM (Lack of Cohesion in Methods) measures how poorly a class's methods are related through shared fields. LCOM4 counts connected components in the method-field usage graph. When LCOM4 > 1, the class contains multiple disconnected concepts and should be split into separate classes, one per connected component.

**Q8. What is the Stable Dependencies Principle?**

The Stable Dependencies Principle states that you should depend in the direction of stability: a stable module (one that many others depend on) should not depend on an unstable one (one that changes frequently). If it did, every change to the unstable module would cascade into the stable one and from there to all its dependents. The remedy is to introduce an abstraction between them.

**Q9. Give an example of control coupling and explain the fix.**

Control coupling occurs when a caller passes a flag to a callee that controls the callee's behavior, e.g., `processor.process(payment, "CREDIT_CARD")`. The fix is polymorphism: define a `PaymentProcessor` interface with a `process(Payment)` method, and create `CreditCardProcessor`, `PayPalProcessor`, etc. The caller selects the right implementation rather than passing a flag.

**Q10. How does package-by-feature differ from package-by-layer and which produces better coupling?**

Package-by-layer groups all controllers together, all services together, etc. Adding a new feature requires touching all three layer packages. Package-by-feature groups all classes for a feature (controller, service, repository, model) into one package. Adding a new feature adds one new package without touching others. Package-by-feature produces higher cohesion (each package is one feature) and lower cross-package coupling (features can be more independent).
