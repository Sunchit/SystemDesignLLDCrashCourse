# Factory Method Pattern

**Category:** Creational Design Pattern
**GoF Reference:** Gang of Four, "Design Patterns: Elements of Reusable Object-Oriented Software" (1994)
**Aliases:** Virtual Constructor

---

## Table of Contents

1. [Intent and the Problem It Solves](#1-intent-and-the-problem-it-solves)
2. [When to Use / When NOT to Use](#2-when-to-use--when-not-to-use)
3. [UML Class Diagram](#3-uml-class-diagram)
4. [Core Mechanism: Deferring Instantiation to Subclasses](#4-core-mechanism-deferring-instantiation-to-subclasses)
5. [Real-World Analogy](#5-real-world-analogy)
6. [Complete Java Implementation: Logger Factory](#6-complete-java-implementation-logger-factory)
7. [Complete Java Implementation: Payment Processor Factory](#7-complete-java-implementation-payment-processor-factory)
8. [How Factory Method Enables OCP](#8-how-factory-method-enables-ocp)
9. [Real-World Examples in Java and Frameworks](#9-real-world-examples-in-java-and-frameworks)
10. [Trade-offs: Pros and Cons](#10-trade-offs-pros-and-cons)
11. [Common Pitfalls and Gotchas](#11-common-pitfalls-and-gotchas)
12. [Three-Way Comparison: Factory Method vs Abstract Factory vs Simple Factory](#12-three-way-comparison-factory-method-vs-abstract-factory-vs-simple-factory)
13. [Interview Questions with Model Answers](#13-interview-questions-with-model-answers)

---

## 1. Intent and the Problem It Solves

### The Official Definition

> Define an interface for creating an object, but let subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses.

### The "new" Keyword Problem

Every time you write `new ConcreteClass()` in client code, you are hard-wiring a dependency. The `new` keyword is the single most common source of tight coupling in object-oriented systems. It looks innocent, but it violates a foundational principle: code should depend on abstractions, not concretions.

Consider a naive logging setup:

```java
// PROBLEM: Tight coupling via the "new" keyword
public class OrderService {

    public void placeOrder(Order order) {
        // Hard-coded dependency on ConsoleLogger
        ConsoleLogger logger = new ConsoleLogger();
        logger.log("Order placed: " + order.getId());

        // ... business logic
    }
}
```

This code has several problems:

1. **The `OrderService` must know the exact class name `ConsoleLogger`.** If the class is renamed, moved to another package, or replaced, `OrderService` must change.
2. **Testing is painful.** You cannot easily substitute a mock or in-memory logger because the dependency is created internally.
3. **Adding a new logger type (e.g., `FileLogger`) requires modifying `OrderService`.** This violates the Open/Closed Principle.
4. **Reuse is impossible.** If you want the same `OrderService` logic with a different logger in a different deployment environment, you cannot achieve it without touching the class.

### Why Interfaces Alone Are Not Enough

A common first attempt is to introduce an interface:

```java
// BETTER, BUT STILL FLAWED
public class OrderService {

    public void placeOrder(Order order) {
        // Still using "new" — still coupled to a concrete class
        ILogger logger = new ConsoleLogger();
        logger.log("Order placed: " + order.getId());
    }
}
```

The variable is now typed to the interface `ILogger`, which is good. But the `new ConsoleLogger()` call still binds the class to the concrete implementation at compile time. The problem has moved one line but has not been solved.

### What Factory Method Solves

The Factory Method pattern extracts the `new` keyword call into a dedicated method — the factory method — and places it in a class specifically designed to be overridden. Client code calls the factory method and receives a product typed to the abstract interface. Client code never knows or cares which concrete class was instantiated.

```java
// The "factory method" pattern extracts creation into an overridable hook
public abstract class OrderService {

    // Factory method — subclasses decide what to create
    protected abstract ILogger createLogger();

    public void placeOrder(Order order) {
        // No "new ConcreteClass()" here — decoupled
        ILogger logger = createLogger();
        logger.log("Order placed: " + order.getId());
    }
}

// A concrete creator decides which product to instantiate
public class ConsoleOrderService extends OrderService {

    @Override
    protected ILogger createLogger() {
        return new ConsoleLogger();  // "new" is isolated here
    }
}
```

Now `OrderService` is completely decoupled from any concrete logger. To switch logging strategies, you create a new subclass of `OrderService` — you never touch the existing code.

---

## 2. When to Use / When NOT to Use

### When to Use Factory Method

**1. When you do not know ahead of time what class to instantiate.**
The exact type of object needed may vary based on runtime configuration, environment, or user input. The factory method defers this decision to subclasses or overriding code.

**2. When you want subclasses to specify the objects they create.**
A framework can define the skeleton of an algorithm using a factory method, letting applications plug in the specific product types they need.

**3. When you want to encapsulate object creation logic.**
Construction may be non-trivial: reading from config files, connecting to registries, applying pooling or caching. Centralizing this in a factory method keeps client code clean.

**4. When you need to support multiple product families in a single hierarchy.**
Different subclasses of a creator can produce entirely different product families while the algorithm that uses them remains unchanged.

**5. When working with parallel class hierarchies.**
If you have a hierarchy of `Document` types (Resume, Report, Letter) and a corresponding hierarchy of `Spell Checker` types, a factory method on the `Document` base class can return the correct `SpellChecker` for that document type.

### When NOT to Use Factory Method

**1. When creation is simple and unlikely to change.**
If you are always creating the same concrete type and the code is internal to a module, a simple `new` call is cleaner. Premature abstraction adds complexity without benefit.

**2. When subclassing is too heavy a mechanism.**
Factory Method is inheritance-based. If your design already favors composition over inheritance, consider the Strategy pattern for dependency injection or a Simple Factory instead.

**3. When you need to create families of related objects.**
If the products come in coordinated families (e.g., a `WindowsButton` must pair with a `WindowsCheckbox`), Abstract Factory is a better fit. A single Factory Method only handles one product type at a time.

**4. When the factory logic is trivial and a static method will do.**
If all you need is `LoggerFactory.getLogger(String name)` as a single static lookup, a Simple Factory (even though it is not a GoF pattern) is simpler and sufficient.

**5. When you are using a dependency injection framework.**
Spring, Guice, and CDI already handle object creation and wiring. Layering Factory Method on top of a DI container is often redundant and adds unnecessary indirection.

---

## 3. UML Class Diagram

```
                        <<abstract>>
                         Creator
                    +-----------------+
                    |                 |
                    | + someOperation()|
                    | # createProduct()|  <--- Factory Method (abstract)
                    |                 |
                    +-----------------+
                           / \
                            |
               (inherits/overrides)
                            |
          +-----------------+-----------------+
          |                                   |
  ConcreteCreatorA                    ConcreteCreatorB
  +------------------+                +------------------+
  |                  |                |                  |
  |# createProduct() |                |# createProduct() |
  |  return new      |                |  return new      |
  |  ConcreteProductA|                |  ConcreteProductB|
  +------------------+                +------------------+
          |                                   |
          | creates                           | creates
          v                                   v
  ConcreteProductA                    ConcreteProductB
  +------------------+                +------------------+
  |                  |                |                  |
  | + operation()    |                | + operation()    |
  +------------------+                +------------------+
          |                                   |
          +-----------------------------------+
                          |
                   <<interface>>
                      Product
                  +-------------+
                  |             |
                  |+ operation()|
                  |             |
                  +-------------+


Relationships:
  Creator         ---->  Product          (depends on the abstraction)
  ConcreteCreatorA ----> ConcreteProductA (creates, knows the concrete type)
  ConcreteCreatorB ----> ConcreteProductB (creates, knows the concrete type)
  ConcreteProductA ....> Product          (implements)
  ConcreteProductB ....> Product          (implements)
```

### Participants

| Participant | Role |
|---|---|
| `Product` | Defines the interface that all products must implement. The creator works only with this interface. |
| `ConcreteProduct` | Implements the `Product` interface. The actual object that gets created. |
| `Creator` | Declares the factory method that returns a `Product`. May define a default implementation. Contains the "template" business logic that calls `createProduct()`. |
| `ConcreteCreator` | Overrides the factory method to return an instance of a `ConcreteProduct`. This is the only class that knows which concrete product to instantiate. |

---

## 4. Core Mechanism: Deferring Instantiation to Subclasses

The power of Factory Method lies in how Java's dynamic dispatch (polymorphism) is used to break the dependency between the algorithm and the object creation.

### Step-by-Step Walkthrough

1. The `Creator` defines an abstract method `createProduct()` that returns the `Product` interface.
2. The `Creator` also defines one or more non-abstract methods (the "template" operations) that call `createProduct()` internally.
3. Because `createProduct()` is abstract, the actual call is resolved at runtime based on the actual type of the `Creator` subclass that was instantiated.
4. The `ConcreteCreator` overrides `createProduct()` and returns a specific `ConcreteProduct`.
5. The client instantiates a `ConcreteCreator` but then calls the inherited template operation. The template operation internally calls the overridden `createProduct()`, getting the right product type without knowing which one it is.

```java
// This is the key insight — the Creator's operation calls createProduct()
// but it has no idea at compile time what createProduct() will return

public abstract class Creator {

    // The factory method
    protected abstract Product createProduct();

    // The template operation that uses the factory method
    public void templateOperation() {
        // At compile time: "Product" — no concrete class knowledge
        // At runtime: whichever ConcreteProduct the subclass creates
        Product product = createProduct();
        product.operation();
    }
}
```

This is sometimes called the "Template Method meets Factory Method" combination, because the creator's operations act as a template that has a hook (`createProduct()`) filled in by subclasses.

---

## 5. Real-World Analogy

### The Staffing Agency Analogy

Imagine a company (the `Creator`) that needs a software engineer (the `Product`). The company knows how to onboard an engineer: give them a laptop, assign a project, set up code access. But the company does not hire directly — it uses a staffing agency.

Different staffing agencies (`ConcreteCreators`) specialize in different types of engineers:
- `JavaDevAgency` sends a `JavaEngineer` (`ConcreteProductA`)
- `PythonDevAgency` sends a `PythonEngineer` (`ConcreteProductB`)
- `FullStackAgency` sends a `FullStackEngineer` (`ConcreteProductC`)

The company's onboarding process (the template operation) works the same regardless of which engineer arrives. The company only cares that the engineer can `writeCode()` and `attendStandup()` — the `Product` interface. Which agency is used (and thus which engineer shows up) is decided externally, not by the onboarding process itself.

This is exactly the Factory Method: the "factory" (agency) is a separate concern from the "consumer" (company). The consumer defines what it needs (interface), and the factory decides what to provide (concrete type).

---

## 6. Complete Java Implementation: Logger Factory

This example builds a complete, production-quality logging system using the Factory Method pattern.

### 6.1 The Product Interface

```java
package com.course.patterns.factory.logger;

/**
 * Product interface — defines the contract all loggers must fulfill.
 * The Creator (LoggerFactory) works only against this type.
 */
public interface ILogger {

    void log(String message);

    void logInfo(String message);

    void logWarning(String message);

    void logError(String message, Throwable throwable);

    /**
     * Returns the name of this logger — useful for diagnostics.
     */
    String getLoggerName();
}
```

### 6.2 Concrete Product A: ConsoleLogger

```java
package com.course.patterns.factory.logger;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

/**
 * ConcreteProduct — writes log output to standard output.
 * Suitable for development environments and local debugging.
 */
public class ConsoleLogger implements ILogger {

    private static final DateTimeFormatter FORMATTER =
            DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

    private final String loggerName;

    public ConsoleLogger(String loggerName) {
        if (loggerName == null || loggerName.isBlank()) {
            throw new IllegalArgumentException("Logger name must not be blank");
        }
        this.loggerName = loggerName;
    }

    @Override
    public void log(String message) {
        System.out.printf("[%s] [%s] %s%n",
                LocalDateTime.now().format(FORMATTER), loggerName, message);
    }

    @Override
    public void logInfo(String message) {
        System.out.printf("[%s] [INFO] [%s] %s%n",
                LocalDateTime.now().format(FORMATTER), loggerName, message);
    }

    @Override
    public void logWarning(String message) {
        System.out.printf("[%s] [WARN] [%s] %s%n",
                LocalDateTime.now().format(FORMATTER), loggerName, message);
    }

    @Override
    public void logError(String message, Throwable throwable) {
        System.err.printf("[%s] [ERROR] [%s] %s — %s%n",
                LocalDateTime.now().format(FORMATTER), loggerName,
                message, throwable.getMessage());
        throwable.printStackTrace(System.err);
    }

    @Override
    public String getLoggerName() {
        return loggerName;
    }
}
```

### 6.3 Concrete Product B: FileLogger

```java
package com.course.patterns.factory.logger;

import java.io.BufferedWriter;
import java.io.FileWriter;
import java.io.IOException;
import java.io.PrintWriter;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

/**
 * ConcreteProduct — writes log output to a file.
 * Suitable for production environments where logs need to be retained.
 */
public class FileLogger implements ILogger {

    private static final DateTimeFormatter FORMATTER =
            DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

    private final String loggerName;
    private final String filePath;

    public FileLogger(String loggerName, String filePath) {
        if (loggerName == null || loggerName.isBlank()) {
            throw new IllegalArgumentException("Logger name must not be blank");
        }
        if (filePath == null || filePath.isBlank()) {
            throw new IllegalArgumentException("File path must not be blank");
        }
        this.loggerName = loggerName;
        this.filePath = filePath;
    }

    @Override
    public void log(String message) {
        writeToFile(String.format("[%s] [%s] %s",
                LocalDateTime.now().format(FORMATTER), loggerName, message));
    }

    @Override
    public void logInfo(String message) {
        writeToFile(String.format("[%s] [INFO] [%s] %s",
                LocalDateTime.now().format(FORMATTER), loggerName, message));
    }

    @Override
    public void logWarning(String message) {
        writeToFile(String.format("[%s] [WARN] [%s] %s",
                LocalDateTime.now().format(FORMATTER), loggerName, message));
    }

    @Override
    public void logError(String message, Throwable throwable) {
        writeToFile(String.format("[%s] [ERROR] [%s] %s — %s",
                LocalDateTime.now().format(FORMATTER), loggerName,
                message, throwable.getMessage()));
    }

    @Override
    public String getLoggerName() {
        return loggerName;
    }

    private void writeToFile(String entry) {
        // append=true so we don't overwrite on each call
        try (PrintWriter writer = new PrintWriter(
                new BufferedWriter(new FileWriter(filePath, true)))) {
            writer.println(entry);
        } catch (IOException e) {
            System.err.println("FileLogger failed to write: " + e.getMessage());
        }
    }
}
```

### 6.4 Concrete Product C: DatabaseLogger

```java
package com.course.patterns.factory.logger;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.time.LocalDateTime;

/**
 * ConcreteProduct — persists log entries to a relational database.
 * Suitable for audit trails and compliance-sensitive logging.
 *
 * Assumes a table with schema:
 *   CREATE TABLE application_logs (
 *       id          BIGINT AUTO_INCREMENT PRIMARY KEY,
 *       logged_at   TIMESTAMP NOT NULL,
 *       logger_name VARCHAR(255) NOT NULL,
 *       level       VARCHAR(20)  NOT NULL,
 *       message     TEXT         NOT NULL,
 *       error_detail TEXT
 *   );
 */
public class DatabaseLogger implements ILogger {

    private static final String INSERT_SQL =
            "INSERT INTO application_logs (logged_at, logger_name, level, message, error_detail) "
          + "VALUES (?, ?, ?, ?, ?)";

    private final String loggerName;
    private final String jdbcUrl;
    private final String dbUser;
    private final String dbPassword;

    public DatabaseLogger(String loggerName,
                          String jdbcUrl,
                          String dbUser,
                          String dbPassword) {
        this.loggerName = loggerName;
        this.jdbcUrl    = jdbcUrl;
        this.dbUser     = dbUser;
        this.dbPassword = dbPassword;
    }

    @Override
    public void log(String message) {
        persist("INFO", message, null);
    }

    @Override
    public void logInfo(String message) {
        persist("INFO", message, null);
    }

    @Override
    public void logWarning(String message) {
        persist("WARN", message, null);
    }

    @Override
    public void logError(String message, Throwable throwable) {
        persist("ERROR", message, throwable != null ? throwable.getMessage() : null);
    }

    @Override
    public String getLoggerName() {
        return loggerName;
    }

    private void persist(String level, String message, String errorDetail) {
        // In production, use a connection pool (e.g., HikariCP) instead of
        // DriverManager — shown here simplified for clarity.
        try (Connection conn = DriverManager.getConnection(jdbcUrl, dbUser, dbPassword);
             PreparedStatement stmt = conn.prepareStatement(INSERT_SQL)) {

            stmt.setObject(1, LocalDateTime.now());
            stmt.setString(2, loggerName);
            stmt.setString(3, level);
            stmt.setString(4, message);
            stmt.setString(5, errorDetail);
            stmt.executeUpdate();

        } catch (SQLException e) {
            System.err.println("DatabaseLogger persistence failed: " + e.getMessage());
        }
    }
}
```

### 6.5 The Abstract Creator: LoggerFactory

```java
package com.course.patterns.factory.logger;

/**
 * Creator (abstract) — declares the factory method and provides
 * template operations that use the product without knowing its type.
 *
 * Subclasses must implement createLogger() to specify which concrete
 * ILogger to instantiate.
 */
public abstract class LoggerFactory {

    /**
     * THE FACTORY METHOD.
     * Subclasses override this to return whichever ILogger they manage.
     *
     * @param loggerName a logical name identifying this logger instance
     * @return a fully initialized ILogger ready for use
     */
    protected abstract ILogger createLogger(String loggerName);

    /**
     * Template operation: obtain a logger and immediately log an info message.
     * Client code calls this and never touches the "new" keyword.
     */
    public final void logInfo(String loggerName, String message) {
        ILogger logger = createLogger(loggerName);
        logger.logInfo(message);
    }

    /**
     * Template operation: obtain a logger and log an error with a cause.
     */
    public final void logError(String loggerName, String message, Throwable cause) {
        ILogger logger = createLogger(loggerName);
        logger.logError(message, cause);
    }

    /**
     * Exposes the raw logger for callers who need richer control.
     * This is the most common usage pattern in real frameworks.
     */
    public final ILogger getLogger(String loggerName) {
        return createLogger(loggerName);
    }
}
```

### 6.6 Concrete Creator A: ConsoleLoggerFactory

```java
package com.course.patterns.factory.logger;

/**
 * ConcreteCreator — produces ConsoleLogger instances.
 * Used in development and local testing environments.
 */
public class ConsoleLoggerFactory extends LoggerFactory {

    @Override
    protected ILogger createLogger(String loggerName) {
        // This is the ONLY place in the codebase that knows about ConsoleLogger
        return new ConsoleLogger(loggerName);
    }
}
```

### 6.7 Concrete Creator B: FileLoggerFactory

```java
package com.course.patterns.factory.logger;

/**
 * ConcreteCreator — produces FileLogger instances bound to a configured file path.
 * Used in staging and production batch workloads.
 */
public class FileLoggerFactory extends LoggerFactory {

    private final String logFilePath;

    public FileLoggerFactory(String logFilePath) {
        if (logFilePath == null || logFilePath.isBlank()) {
            throw new IllegalArgumentException("logFilePath must not be blank");
        }
        this.logFilePath = logFilePath;
    }

    @Override
    protected ILogger createLogger(String loggerName) {
        return new FileLogger(loggerName, logFilePath);
    }
}
```

### 6.8 Concrete Creator C: DatabaseLoggerFactory

```java
package com.course.patterns.factory.logger;

/**
 * ConcreteCreator — produces DatabaseLogger instances.
 * Used where audit trails and compliance persistence are required.
 */
public class DatabaseLoggerFactory extends LoggerFactory {

    private final String jdbcUrl;
    private final String dbUser;
    private final String dbPassword;

    public DatabaseLoggerFactory(String jdbcUrl, String dbUser, String dbPassword) {
        this.jdbcUrl    = jdbcUrl;
        this.dbUser     = dbUser;
        this.dbPassword = dbPassword;
    }

    @Override
    protected ILogger createLogger(String loggerName) {
        return new DatabaseLogger(loggerName, jdbcUrl, dbUser, dbPassword);
    }
}
```

### 6.9 Service Using the Factory (Client Code)

```java
package com.course.patterns.factory.logger;

/**
 * Client / Service class — completely decoupled from any concrete logger.
 * It works against LoggerFactory (the Creator abstraction) and ILogger (the Product abstraction).
 *
 * Notice: zero import of ConsoleLogger, FileLogger, or DatabaseLogger.
 */
public class OrderService {

    private final LoggerFactory loggerFactory;

    // Dependency is injected — the factory decides what kind of logger to use
    public OrderService(LoggerFactory loggerFactory) {
        this.loggerFactory = loggerFactory;
    }

    public void placeOrder(String orderId, double amount) {
        ILogger logger = loggerFactory.getLogger(OrderService.class.getName());

        logger.logInfo("Placing order: " + orderId + " for $" + amount);

        try {
            // Simulate business logic
            if (amount <= 0) {
                throw new IllegalArgumentException("Amount must be positive");
            }
            logger.logInfo("Order " + orderId + " placed successfully.");

        } catch (Exception e) {
            logger.logError("Failed to place order: " + orderId, e);
            throw e;
        }
    }
}
```

### 6.10 Driver / Main Class

```java
package com.course.patterns.factory.logger;

/**
 * Demonstrates swapping logger implementations at the composition root
 * without touching OrderService or any business logic.
 */
public class LoggerFactoryDemo {

    public static void main(String[] args) {

        System.out.println("=== DEV ENVIRONMENT: Console Logging ===");
        LoggerFactory devFactory = new ConsoleLoggerFactory();
        OrderService devService  = new OrderService(devFactory);
        devService.placeOrder("ORD-001", 99.99);

        System.out.println("\n=== STAGING ENVIRONMENT: File Logging ===");
        LoggerFactory stagingFactory = new FileLoggerFactory("/tmp/app-staging.log");
        OrderService stagingService  = new OrderService(stagingFactory);
        stagingService.placeOrder("ORD-002", 149.50);

        System.out.println("\n=== PRODUCTION ENVIRONMENT: Database Logging ===");
        LoggerFactory prodFactory = new DatabaseLoggerFactory(
                "jdbc:mysql://prod-db:3306/logs", "app_user", "secret");
        OrderService prodService = new OrderService(prodFactory);
        prodService.placeOrder("ORD-003", 299.00);

        System.out.println("\n=== Error scenario ===");
        devService.placeOrder("ORD-ERR", -50.0);
    }
}
```

---

## 7. Complete Java Implementation: Payment Processor Factory

This example demonstrates a more complex domain — payment processing — with a richer product interface and additional concerns like transaction validation and receipts.

### 7.1 Supporting Types

```java
package com.course.patterns.factory.payment;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

/**
 * Represents a payment request submitted by the client.
 */
public final class PaymentRequest {

    private final String customerId;
    private final BigDecimal amount;
    private final String currency;
    private final String description;

    public PaymentRequest(String customerId, BigDecimal amount,
                          String currency, String description) {
        if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        this.customerId  = customerId;
        this.amount      = amount;
        this.currency    = currency;
        this.description = description;
    }

    public String getCustomerId()  { return customerId; }
    public BigDecimal getAmount()  { return amount; }
    public String getCurrency()    { return currency; }
    public String getDescription() { return description; }
}
```

```java
package com.course.patterns.factory.payment;

import java.math.BigDecimal;
import java.time.LocalDateTime;

/**
 * Represents the outcome of a payment attempt.
 */
public final class PaymentReceipt {

    public enum Status { SUCCESS, FAILED, PENDING }

    private final String transactionId;
    private final Status status;
    private final BigDecimal amount;
    private final String currency;
    private final String processorName;
    private final LocalDateTime processedAt;
    private final String failureReason;

    private PaymentReceipt(Builder builder) {
        this.transactionId = builder.transactionId;
        this.status        = builder.status;
        this.amount        = builder.amount;
        this.currency      = builder.currency;
        this.processorName = builder.processorName;
        this.processedAt   = builder.processedAt;
        this.failureReason = builder.failureReason;
    }

    public String getTransactionId()  { return transactionId; }
    public Status getStatus()         { return status; }
    public BigDecimal getAmount()     { return amount; }
    public String getCurrency()       { return currency; }
    public String getProcessorName()  { return processorName; }
    public LocalDateTime getProcessedAt() { return processedAt; }
    public String getFailureReason()  { return failureReason; }

    @Override
    public String toString() {
        return String.format("PaymentReceipt{txId=%s, status=%s, amount=%s %s, processor=%s, at=%s}",
                transactionId, status, amount, currency, processorName, processedAt);
    }

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private String transactionId;
        private Status status;
        private BigDecimal amount;
        private String currency;
        private String processorName;
        private LocalDateTime processedAt;
        private String failureReason;

        public Builder transactionId(String val)  { this.transactionId = val; return this; }
        public Builder status(Status val)         { this.status = val;        return this; }
        public Builder amount(BigDecimal val)     { this.amount = val;        return this; }
        public Builder currency(String val)       { this.currency = val;      return this; }
        public Builder processorName(String val)  { this.processorName = val; return this; }
        public Builder processedAt(LocalDateTime v){ this.processedAt = v;   return this; }
        public Builder failureReason(String val)  { this.failureReason = val; return this; }
        public PaymentReceipt build()             { return new PaymentReceipt(this); }
    }
}
```

### 7.2 The Product Interface: PaymentProcessor

```java
package com.course.patterns.factory.payment;

/**
 * Product interface — all payment processors must implement this contract.
 * The abstract creator (PaymentProcessorFactory) and all client code
 * depend ONLY on this type.
 */
public interface PaymentProcessor {

    /**
     * Validates that the payment request is acceptable for this processor
     * (e.g., currency support, amount limits, required fields).
     *
     * @param request the payment request to validate
     * @throws IllegalArgumentException if the request is not valid for this processor
     */
    void validate(PaymentRequest request);

    /**
     * Processes the payment and returns a receipt.
     *
     * @param request the validated payment request
     * @return a PaymentReceipt describing the outcome
     */
    PaymentReceipt process(PaymentRequest request);

    /**
     * Initiates a refund for a previous transaction.
     *
     * @param transactionId the original transaction to refund
     * @return a PaymentReceipt for the refund operation
     */
    PaymentReceipt refund(String transactionId);

    /**
     * Returns a human-readable name for this processor (e.g., "CreditCard", "PayPal").
     */
    String getProcessorName();
}
```

### 7.3 Concrete Product A: CreditCardProcessor

```java
package com.course.patterns.factory.payment;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Set;
import java.util.UUID;

/**
 * ConcreteProduct — processes payments via credit/debit card networks.
 */
public class CreditCardProcessor implements PaymentProcessor {

    private static final Set<String> SUPPORTED_CURRENCIES =
            Set.of("USD", "EUR", "GBP", "CAD", "AUD");

    private static final BigDecimal MAX_TRANSACTION = new BigDecimal("50000.00");

    private final String apiKey;
    private final String merchantId;

    public CreditCardProcessor(String apiKey, String merchantId) {
        this.apiKey     = apiKey;
        this.merchantId = merchantId;
    }

    @Override
    public void validate(PaymentRequest request) {
        if (!SUPPORTED_CURRENCIES.contains(request.getCurrency())) {
            throw new IllegalArgumentException(
                    "CreditCardProcessor does not support currency: " + request.getCurrency());
        }
        if (request.getAmount().compareTo(MAX_TRANSACTION) > 0) {
            throw new IllegalArgumentException(
                    "Amount exceeds credit card transaction limit of " + MAX_TRANSACTION);
        }
    }

    @Override
    public PaymentReceipt process(PaymentRequest request) {
        validate(request);

        // In production, this would call Stripe / Braintree / Adyen API
        System.out.printf("[CreditCard] Charging %s %s for customer %s via merchant %s%n",
                request.getAmount(), request.getCurrency(),
                request.getCustomerId(), merchantId);

        return PaymentReceipt.builder()
                .transactionId(UUID.randomUUID().toString())
                .status(PaymentReceipt.Status.SUCCESS)
                .amount(request.getAmount())
                .currency(request.getCurrency())
                .processorName(getProcessorName())
                .processedAt(LocalDateTime.now())
                .build();
    }

    @Override
    public PaymentReceipt refund(String transactionId) {
        System.out.printf("[CreditCard] Refunding transaction: %s%n", transactionId);

        return PaymentReceipt.builder()
                .transactionId(UUID.randomUUID().toString())
                .status(PaymentReceipt.Status.SUCCESS)
                .amount(BigDecimal.ZERO) // refund amount would be looked up
                .currency("USD")
                .processorName(getProcessorName())
                .processedAt(LocalDateTime.now())
                .build();
    }

    @Override
    public String getProcessorName() {
        return "CreditCard";
    }
}
```

### 7.4 Concrete Product B: PayPalProcessor

```java
package com.course.patterns.factory.payment;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

/**
 * ConcreteProduct — processes payments via the PayPal REST API.
 */
public class PayPalProcessor implements PaymentProcessor {

    private static final BigDecimal MAX_TRANSACTION = new BigDecimal("10000.00");

    private final String clientId;
    private final String clientSecret;
    private final String environment; // "sandbox" or "live"

    public PayPalProcessor(String clientId, String clientSecret, String environment) {
        this.clientId     = clientId;
        this.clientSecret = clientSecret;
        this.environment  = environment;
    }

    @Override
    public void validate(PaymentRequest request) {
        if (request.getAmount().compareTo(MAX_TRANSACTION) > 0) {
            throw new IllegalArgumentException(
                    "PayPal transaction limit is " + MAX_TRANSACTION);
        }
        if (request.getCustomerId() == null || request.getCustomerId().isBlank()) {
            throw new IllegalArgumentException("PayPal requires a customer ID (email or PayPal ID)");
        }
    }

    @Override
    public PaymentReceipt process(PaymentRequest request) {
        validate(request);

        // In production, call PayPal Orders v2 API using OAuth2 token
        System.out.printf("[PayPal] Processing %s %s for %s in %s environment%n",
                request.getAmount(), request.getCurrency(),
                request.getCustomerId(), environment);

        return PaymentReceipt.builder()
                .transactionId("PAYPAL-" + UUID.randomUUID().toString())
                .status(PaymentReceipt.Status.SUCCESS)
                .amount(request.getAmount())
                .currency(request.getCurrency())
                .processorName(getProcessorName())
                .processedAt(LocalDateTime.now())
                .build();
    }

    @Override
    public PaymentReceipt refund(String transactionId) {
        System.out.printf("[PayPal] Refunding transaction: %s%n", transactionId);

        return PaymentReceipt.builder()
                .transactionId("PAYPAL-REFUND-" + UUID.randomUUID().toString())
                .status(PaymentReceipt.Status.SUCCESS)
                .amount(BigDecimal.ZERO)
                .currency("USD")
                .processorName(getProcessorName())
                .processedAt(LocalDateTime.now())
                .build();
    }

    @Override
    public String getProcessorName() {
        return "PayPal";
    }
}
```

### 7.5 Concrete Product C: CryptoProcessor

```java
package com.course.patterns.factory.payment;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Set;
import java.util.UUID;

/**
 * ConcreteProduct — processes payments via cryptocurrency networks.
 * Supports Bitcoin (BTC), Ethereum (ETH), and stablecoins (USDC).
 */
public class CryptoProcessor implements PaymentProcessor {

    private static final Set<String> SUPPORTED_COINS = Set.of("BTC", "ETH", "USDC");

    private final String walletAddress;
    private final String networkEndpoint;

    public CryptoProcessor(String walletAddress, String networkEndpoint) {
        this.walletAddress   = walletAddress;
        this.networkEndpoint = networkEndpoint;
    }

    @Override
    public void validate(PaymentRequest request) {
        if (!SUPPORTED_COINS.contains(request.getCurrency())) {
            throw new IllegalArgumentException(
                    "CryptoProcessor supports: " + SUPPORTED_COINS
                    + " but received: " + request.getCurrency());
        }
    }

    @Override
    public PaymentReceipt process(PaymentRequest request) {
        validate(request);

        // In production, broadcast a signed transaction to the blockchain node
        System.out.printf("[Crypto] Broadcasting %s %s to wallet %s via %s%n",
                request.getAmount(), request.getCurrency(),
                walletAddress, networkEndpoint);

        return PaymentReceipt.builder()
                .transactionId("0x" + UUID.randomUUID().toString().replace("-", ""))
                .status(PaymentReceipt.Status.PENDING) // blockchain confirmation pending
                .amount(request.getAmount())
                .currency(request.getCurrency())
                .processorName(getProcessorName())
                .processedAt(LocalDateTime.now())
                .build();
    }

    @Override
    public PaymentReceipt refund(String transactionId) {
        // Crypto refunds are new outbound transactions — not reversals
        System.out.printf("[Crypto] Initiating refund transaction for: %s%n", transactionId);

        return PaymentReceipt.builder()
                .transactionId("0x" + UUID.randomUUID().toString().replace("-", ""))
                .status(PaymentReceipt.Status.PENDING)
                .amount(BigDecimal.ZERO)
                .currency("BTC")
                .processorName(getProcessorName())
                .processedAt(LocalDateTime.now())
                .build();
    }

    @Override
    public String getProcessorName() {
        return "Crypto";
    }
}
```

### 7.6 Abstract Creator: PaymentProcessorFactory

```java
package com.course.patterns.factory.payment;

/**
 * Creator (abstract) — declares the factory method and houses the
 * reusable payment orchestration logic.
 *
 * This class knows HOW to run a payment workflow; subclasses decide
 * WHICH processor to use.
 */
public abstract class PaymentProcessorFactory {

    /**
     * THE FACTORY METHOD.
     * Subclasses override this to return their specific PaymentProcessor.
     *
     * @return a fully initialized PaymentProcessor
     */
    protected abstract PaymentProcessor createProcessor();

    /**
     * Template operation: validate and process a payment in one call.
     * This entire orchestration is reused across all subclasses.
     */
    public final PaymentReceipt pay(PaymentRequest request) {
        PaymentProcessor processor = createProcessor();
        System.out.printf("[Factory] Using processor: %s%n", processor.getProcessorName());

        // Pre-processing hook (can be extended in subclasses if needed)
        beforePayment(request, processor);

        PaymentReceipt receipt = processor.process(request);

        // Post-processing hook
        afterPayment(receipt, processor);

        return receipt;
    }

    /**
     * Template operation: issue a refund via the processor this factory creates.
     */
    public final PaymentReceipt refund(String transactionId) {
        PaymentProcessor processor = createProcessor();
        return processor.refund(transactionId);
    }

    /**
     * Hook method — subclasses may override to add pre-payment logic
     * (e.g., fraud screening, rate limiting). Default is no-op.
     */
    protected void beforePayment(PaymentRequest request, PaymentProcessor processor) {
        // default: no-op
    }

    /**
     * Hook method — subclasses may override to add post-payment logic
     * (e.g., analytics events, notifications). Default is no-op.
     */
    protected void afterPayment(PaymentReceipt receipt, PaymentProcessor processor) {
        // default: no-op
    }
}
```

### 7.7 Concrete Creators

```java
package com.course.patterns.factory.payment;

/**
 * ConcreteCreator — produces CreditCardProcessor instances.
 */
public class CreditCardProcessorFactory extends PaymentProcessorFactory {

    private final String apiKey;
    private final String merchantId;

    public CreditCardProcessorFactory(String apiKey, String merchantId) {
        this.apiKey     = apiKey;
        this.merchantId = merchantId;
    }

    @Override
    protected PaymentProcessor createProcessor() {
        return new CreditCardProcessor(apiKey, merchantId);
    }

    @Override
    protected void beforePayment(PaymentRequest request, PaymentProcessor processor) {
        // Credit card payments get fraud screening first
        System.out.printf("[CreditCardFactory] Running fraud screen for customer: %s%n",
                request.getCustomerId());
    }
}
```

```java
package com.course.patterns.factory.payment;

/**
 * ConcreteCreator — produces PayPalProcessor instances.
 */
public class PayPalProcessorFactory extends PaymentProcessorFactory {

    private final String clientId;
    private final String clientSecret;

    public PayPalProcessorFactory(String clientId, String clientSecret) {
        this.clientId     = clientId;
        this.clientSecret = clientSecret;
    }

    @Override
    protected PaymentProcessor createProcessor() {
        String env = System.getenv("APP_ENV") != null
                ? System.getenv("APP_ENV") : "sandbox";
        return new PayPalProcessor(clientId, clientSecret, env);
    }
}
```

```java
package com.course.patterns.factory.payment;

/**
 * ConcreteCreator — produces CryptoProcessor instances.
 */
public class CryptoProcessorFactory extends PaymentProcessorFactory {

    private final String walletAddress;
    private final String networkEndpoint;

    public CryptoProcessorFactory(String walletAddress, String networkEndpoint) {
        this.walletAddress   = walletAddress;
        this.networkEndpoint = networkEndpoint;
    }

    @Override
    protected PaymentProcessor createProcessor() {
        return new CryptoProcessor(walletAddress, networkEndpoint);
    }

    @Override
    protected void afterPayment(PaymentReceipt receipt, PaymentProcessor processor) {
        // Crypto payments need additional confirmation monitoring
        if (receipt.getStatus() == PaymentReceipt.Status.PENDING) {
            System.out.printf("[CryptoFactory] Scheduling confirmation poller for tx: %s%n",
                    receipt.getTransactionId());
        }
    }
}
```

### 7.8 Checkout Service (Client)

```java
package com.course.patterns.factory.payment;

import java.math.BigDecimal;

/**
 * Client code — works entirely against the abstraction layer.
 * No import of CreditCardProcessor, PayPalProcessor, or CryptoProcessor.
 */
public class CheckoutService {

    private final PaymentProcessorFactory processorFactory;

    public CheckoutService(PaymentProcessorFactory processorFactory) {
        this.processorFactory = processorFactory;
    }

    public PaymentReceipt checkout(String customerId, BigDecimal total,
                                   String currency, String description) {
        PaymentRequest request = new PaymentRequest(customerId, total, currency, description);

        System.out.println("=== Initiating checkout ===");
        PaymentReceipt receipt = processorFactory.pay(request);
        System.out.println("=== Checkout complete: " + receipt + " ===\n");

        return receipt;
    }
}
```

### 7.9 Payment Demo

```java
package com.course.patterns.factory.payment;

import java.math.BigDecimal;

public class PaymentFactoryDemo {

    public static void main(String[] args) {

        // Composition root: pick the factory based on config/environment
        PaymentProcessorFactory creditCardFactory =
                new CreditCardProcessorFactory("sk_live_abc123", "MERCHANT_001");

        PaymentProcessorFactory paypalFactory =
                new PayPalProcessorFactory("CLIENT_ID_XYZ", "CLIENT_SECRET_XYZ");

        PaymentProcessorFactory cryptoFactory =
                new CryptoProcessorFactory("bc1q...", "https://node.bitcoin.org");

        // Same CheckoutService, different factories
        CheckoutService creditService = new CheckoutService(creditCardFactory);
        creditService.checkout("CUST-001", new BigDecimal("149.99"), "USD", "Premium subscription");

        CheckoutService paypalService = new CheckoutService(paypalFactory);
        paypalService.checkout("customer@email.com", new BigDecimal("75.00"), "EUR", "Digital goods");

        CheckoutService cryptoService = new CheckoutService(cryptoFactory);
        cryptoService.checkout("CUST-003", new BigDecimal("0.0025"), "BTC", "NFT purchase");
    }
}
```

---

## 8. How Factory Method Enables OCP

The Open/Closed Principle states: software entities should be **open for extension** but **closed for modification**.

Factory Method is one of the cleanest ways to achieve OCP in practice.

### The Extension Scenario

Suppose the business decides to add a **bank transfer** payment method. Without Factory Method, you would need to hunt through `CheckoutService` and modify it. With Factory Method:

**Step 1: Create the new product (no existing code modified)**

```java
package com.course.patterns.factory.payment;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

/**
 * New ConcreteProduct — zero modifications to existing code required.
 */
public class BankTransferProcessor implements PaymentProcessor {

    private final String bankCode;
    private final String accountNumber;

    public BankTransferProcessor(String bankCode, String accountNumber) {
        this.bankCode      = bankCode;
        this.accountNumber = accountNumber;
    }

    @Override
    public void validate(PaymentRequest request) {
        if (request.getAmount().compareTo(new BigDecimal("1000000")) > 0) {
            throw new IllegalArgumentException("Bank transfer limit exceeded");
        }
    }

    @Override
    public PaymentReceipt process(PaymentRequest request) {
        validate(request);
        System.out.printf("[BankTransfer] Initiating ACH transfer of %s %s to account %s%n",
                request.getAmount(), request.getCurrency(), accountNumber);

        return PaymentReceipt.builder()
                .transactionId("ACH-" + UUID.randomUUID())
                .status(PaymentReceipt.Status.PENDING)
                .amount(request.getAmount())
                .currency(request.getCurrency())
                .processorName(getProcessorName())
                .processedAt(LocalDateTime.now())
                .build();
    }

    @Override
    public PaymentReceipt refund(String transactionId) {
        return PaymentReceipt.builder()
                .transactionId("ACH-REFUND-" + UUID.randomUUID())
                .status(PaymentReceipt.Status.PENDING)
                .amount(BigDecimal.ZERO)
                .currency("USD")
                .processorName(getProcessorName())
                .processedAt(LocalDateTime.now())
                .build();
    }

    @Override
    public String getProcessorName() { return "BankTransfer"; }
}
```

**Step 2: Create the new creator (no existing code modified)**

```java
package com.course.patterns.factory.payment;

/**
 * New ConcreteCreator — plugged into the existing hierarchy.
 * CheckoutService, PaymentProcessorFactory, and all existing code
 * remain completely unchanged.
 */
public class BankTransferProcessorFactory extends PaymentProcessorFactory {

    private final String bankCode;
    private final String accountNumber;

    public BankTransferProcessorFactory(String bankCode, String accountNumber) {
        this.bankCode      = bankCode;
        this.accountNumber = accountNumber;
    }

    @Override
    protected PaymentProcessor createProcessor() {
        return new BankTransferProcessor(bankCode, accountNumber);
    }
}
```

**Step 3: Wire it up at the composition root**

```java
PaymentProcessorFactory bankFactory =
        new BankTransferProcessorFactory("SWIFT-CHASE", "ACC-12345678");
CheckoutService bankService = new CheckoutService(bankFactory);
bankService.checkout("CUST-B2B", new BigDecimal("25000"), "USD", "B2B invoice payment");
```

The existing classes `CheckoutService`, `PaymentProcessorFactory`, `CreditCardProcessorFactory`, `PayPalProcessorFactory`, `CryptoProcessorFactory`, `PaymentProcessor`, `PaymentRequest`, and `PaymentReceipt` were all **closed for modification** while the system was **open for extension** via the new classes.

---

## 9. Real-World Examples in Java and Frameworks

### 9.1 java.util.Calendar.getInstance()

```java
// This is a factory method (static variant) that defers to the JVM
// to return the correct Calendar subclass for the current locale and timezone
Calendar cal = Calendar.getInstance();
// Returns GregorianCalendar or BuddhistCalendar or JapaneseImperialCalendar
// depending on Locale — the caller never knows which subclass it got

Calendar japanCal = Calendar.getInstance(Locale.JAPAN);
// Returns a different concrete subclass for Japan's calendar system
```

The pattern: `Calendar` is the abstract creator/product. `getInstance()` is the factory method. `GregorianCalendar` is the concrete product.

### 9.2 java.sql.DriverManager.getConnection()

```java
// The JDBC DriverManager acts as the factory
Connection conn = DriverManager.getConnection(
        "jdbc:postgresql://localhost:5432/mydb", "user", "password");

// The caller receives a Connection (product interface)
// The actual object is a PostgreSQL-specific class (org.postgresql.jdbc.PgConnection)
// The caller never imports or touches the PgConnection class
```

Registering a new JDBC driver extends the system without modifying `DriverManager`:
```java
// Adding MySQL support — zero changes to DriverManager
Class.forName("com.mysql.cj.jdbc.Driver"); // driver self-registers
Connection mysqlConn = DriverManager.getConnection("jdbc:mysql://host/db", ...);
```

### 9.3 java.util.Iterator

```java
// Collection.iterator() is a factory method
List<String> list = new ArrayList<>();
Iterator<String> iter = list.iterator(); // factory method — returns ArrayListItr

Set<String> set = new HashSet<>();
Iterator<String> setIter = set.iterator(); // factory method — returns HashSet$KeyIterator

// The caller uses only the Iterator interface — no knowledge of the concrete class
while (iter.hasNext()) {
    System.out.println(iter.next());
}
```

Each collection's `iterator()` method is the factory method, returning the concrete iterator appropriate for that collection's internal structure.

### 9.4 Spring's FactoryBean

```java
import org.springframework.beans.factory.FactoryBean;

/**
 * Spring's FactoryBean is a first-class Factory Method participant.
 * Spring calls getObject() — the factory method — instead of constructing
 * the bean directly.
 */
public class ConnectionPoolFactoryBean implements FactoryBean<DataSource> {

    private String url;
    private int poolSize;

    // Factory method — Spring calls this to get the actual bean
    @Override
    public DataSource getObject() throws Exception {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setMaximumPoolSize(poolSize);
        return new HikariDataSource(config);
    }

    @Override
    public Class<?> getObjectType() {
        return DataSource.class;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }

    // setters omitted for brevity
}
```

Any code that `@Autowired DataSource dataSource` gets the object produced by `getObject()` — the factory method. The consuming bean has zero knowledge of `HikariDataSource`.

### 9.5 java.nio.charset.Charset.forName()

```java
Charset utf8  = Charset.forName("UTF-8");    // returns sun.nio.cs.UTF_8 (internal class)
Charset latin = Charset.forName("ISO-8859-1"); // returns sun.nio.cs.ISO_8859_1
// Callers work with Charset — no knowledge of the concrete implementation
```

### 9.6 javax.xml.parsers.DocumentBuilderFactory

```java
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
// newInstance() reads META-INF/services and system properties to pick
// the correct factory subclass — pure Factory Method

DocumentBuilder builder = factory.newDocumentBuilder(); // another factory method
Document doc = builder.parse(new File("config.xml"));
```

---

## 10. Trade-offs: Pros and Cons

### Pros

**1. Eliminates tight coupling to concrete classes.**
Client code depends only on the abstract `Product` interface and the abstract `Creator`. Concrete classes are isolated to the factory and the composition root.

**2. Single Responsibility Principle.**
Each `ConcreteCreator` has one job: create a specific product. Construction logic (configuration reading, connection pooling, parameter validation) is not scattered across business code.

**3. Open/Closed Principle.**
Adding a new product type requires only two new classes: a `ConcreteProduct` and a `ConcreteCreator`. No existing code is modified.

**4. Testability.**
You can inject a `MockLoggerFactory` or `TestPaymentProcessorFactory` in unit tests. The business logic under test is completely isolated from I/O.

**5. Named creators are self-documenting.**
`new FileLoggerFactory("/var/log/app.log")` is far more readable than a configuration flag on a god-factory.

**6. Supports parameterized creation.**
The factory method can accept arguments (as shown in `createLogger(String loggerName)`), allowing fine-grained control over each product instance.

### Cons

**1. Class explosion.**
Every new product type requires two new classes: the product and its creator. In large systems, this can lead to dozens of small, shallow classes.

**2. Inheritance coupling.**
The pattern is inheritance-based. If you later want to change the creator hierarchy, refactoring is harder than with composition-based alternatives (Strategy pattern + DI).

**3. Clients must subclass or know the creator.**
Someone at the composition root still needs to choose `new CreditCardProcessorFactory(...)`. The decision of which factory to use is not eliminated — it is moved, not removed.

**4. Can be over-engineered for simple use cases.**
For a small application with one logging destination that never changes, adding a factory hierarchy is unnecessary complexity. Know when a simple `new` is the right call.

**5. Factory method must return consistent types.**
If the factory method signature changes (e.g., adding a required parameter), all `ConcreteCreator` subclasses must be updated.

---

## 11. Common Pitfalls and Gotchas

### Pitfall 1: Confusing Factory Method with Simple Factory

A Simple Factory is a class with a static `create(String type)` method and a big `switch` statement. It is NOT a GoF pattern because it is not open for extension without modification.

```java
// SIMPLE FACTORY — anti-pattern for extensibility
public class LoggerSimpleFactory {
    public static ILogger create(String type) {
        switch (type) {
            case "console": return new ConsoleLogger("default");
            case "file":    return new FileLogger("default", "/tmp/app.log");
            // Adding "database" MODIFIES this class — violates OCP
            default: throw new IllegalArgumentException("Unknown type: " + type);
        }
    }
}
```

Factory Method uses inheritance and method overriding to achieve extension without modification.

### Pitfall 2: Making the Factory Method Static

The whole point of Factory Method is that it is virtual (overridable). A static method cannot be overridden — it can only be hidden (shadowed). Static factory methods are the Simple Factory pattern, not Factory Method.

```java
// WRONG — static cannot be overridden, breaking the pattern
public abstract class LoggerFactory {
    public static ILogger createLogger(String name) { // static kills polymorphism
        return new ConsoleLogger(name);
    }
}
```

### Pitfall 3: Leaking Concrete Types Back to Clients

```java
// WRONG — the creator leaks the concrete type
public class ConsoleLoggerFactory extends LoggerFactory {
    // Return type is ConsoleLogger, not ILogger
    @Override
    protected ConsoleLogger createLogger(String name) { // do not do this
        return new ConsoleLogger(name);
    }
}
```

The factory method must return the abstract product type (`ILogger`). If you return the concrete type, client code will start depending on it, defeating the purpose.

### Pitfall 4: Putting Business Logic in the Factory Method

The factory method's only job is to instantiate and return a product. Business logic belongs in the template operations, not in `createProduct()`.

```java
// WRONG — mixing creation and business logic
@Override
protected ILogger createLogger(String name) {
    ILogger logger = new ConsoleLogger(name);
    logger.logInfo("Logger created");  // side effect in factory method — bad
    return logger;
}
```

### Pitfall 5: Not Making the Factory Method Protected (or package-private)

The factory method is an extension point for subclasses, not part of the public API. Making it `public` exposes an internal hook that callers should never call directly — they call the template operations.

```java
// Factory method should be protected, not public
protected abstract ILogger createLogger(String loggerName); // correct
public abstract ILogger createLogger(String loggerName);    // exposes internal hook unnecessarily
```

### Pitfall 6: Returning null from the Factory Method

A factory method should never return `null`. If it cannot create a product, it should throw a descriptive exception. Returning `null` will cause `NullPointerException` deep in the template operations, far from the actual source of the problem.

```java
// WRONG
@Override
protected PaymentProcessor createProcessor() {
    if (apiKey == null) return null; // causes NPE later — terrible
}

// CORRECT
@Override
protected PaymentProcessor createProcessor() {
    if (apiKey == null || apiKey.isBlank()) {
        throw new IllegalStateException("CreditCardProcessorFactory requires a valid API key");
    }
    return new CreditCardProcessor(apiKey, merchantId);
}
```

### Pitfall 7: Overusing the Pattern

Not every `new` needs a factory. If the constructed object is a simple value object, a data transfer object, or an internal implementation detail that will never change, a factory is ceremony without benefit. Reserve Factory Method for objects whose concrete type varies by environment, configuration, or extension.

---

## 12. Three-Way Comparison: Factory Method vs Abstract Factory vs Simple Factory

### Overview Table

| Dimension | Simple Factory | Factory Method | Abstract Factory |
|---|---|---|---|
| GoF Pattern? | No (idiom) | Yes | Yes |
| Mechanism | Static method + switch | Inheritance + method override | Object composition |
| Creates | One product type | One product type | Families of related products |
| Extension | Modify the factory (OCP violation) | Add a new subclass | Add a new factory class |
| Complexity | Low | Medium | High |
| Best for | Fixed, known set of products | Varying product per subclass | Ensuring product families are consistent |
| Coupling | Client coupled to factory class | Client coupled to Creator abstraction | Client coupled to AbstractFactory abstraction |
| Inheritance required? | No | Yes (Creator hierarchy) | No (composition) |

### Simple Factory in Detail

```java
// NOT a GoF pattern — commonly used in practice, but violates OCP
public class NotificationFactory {

    public static Notification create(String channel) {
        return switch (channel) {
            case "email" -> new EmailNotification();
            case "sms"   -> new SmsNotification();
            case "push"  -> new PushNotification();
            // Adding "slack" requires modifying this class
            default -> throw new IllegalArgumentException("Unknown channel: " + channel);
        };
    }
}
// Usage:
Notification n = NotificationFactory.create("email");
```

**When to use Simple Factory:** The set of products is small, known at compile time, and rarely changes. The simplicity outweighs the OCP concern. Great for getting started quickly.

### Factory Method in Detail

```java
// GoF Factory Method — extension via subclassing
public abstract class NotificationService {
    protected abstract Notification createNotification(); // factory method

    public void send(String message) {
        Notification n = createNotification();
        n.deliver(message);
    }
}

public class EmailNotificationService extends NotificationService {
    @Override
    protected Notification createNotification() {
        return new EmailNotification("smtp.company.com", 587);
    }
}

// Adding Slack: new SlackNotificationService — zero changes to existing code
public class SlackNotificationService extends NotificationService {
    @Override
    protected Notification createNotification() {
        return new SlackNotification("https://hooks.slack.com/...");
    }
}
```

**When to use Factory Method:** One product type varies. The creator class has meaningful template operations. You want inheritance-based extension.

### Abstract Factory in Detail

Abstract Factory produces *families* of related objects. The key distinction: all products in the family must be compatible with each other.

```java
// Abstract Factory — creates families of related UI components
public interface UIComponentFactory {
    Button createButton();       // one product in the family
    TextField createTextField(); // another product in the family
    Dialog createDialog();       // yet another product in the family
}

// Concrete factory A — Windows family (all products are Windows-styled)
public class WindowsUIFactory implements UIComponentFactory {
    @Override public Button createButton()       { return new WindowsButton(); }
    @Override public TextField createTextField() { return new WindowsTextField(); }
    @Override public Dialog createDialog()       { return new WindowsDialog(); }
}

// Concrete factory B — macOS family (all products are macOS-styled)
public class MacOSUIFactory implements UIComponentFactory {
    @Override public Button createButton()       { return new MacOSButton(); }
    @Override public TextField createTextField() { return new MacOSTextField(); }
    @Override public Dialog createDialog()       { return new MacOSDialog(); }
}

// Client — receives the factory through dependency injection
public class Application {
    private final UIComponentFactory factory;

    public Application(UIComponentFactory factory) {
        this.factory = factory;
    }

    public void render() {
        Button btn       = factory.createButton();       // Windows or macOS, consistently
        TextField field  = factory.createTextField();
        Dialog dialog    = factory.createDialog();
        // Guaranteed: btn, field, and dialog are all from the same OS family
    }
}
```

**When to use Abstract Factory:** Your system needs to enforce that products from the same family are always used together. A Windows button must never appear alongside a macOS dialog.

### Decision Guide

```
Do you need to create FAMILIES of related objects that must be compatible?
    YES → Abstract Factory
    NO  ↓

Do you need to vary WHICH concrete type is instantiated based on subclass?
    YES → Factory Method
    NO  ↓

Is the set of concrete types fixed and small, and is simplicity paramount?
    YES → Simple Factory (static method)
    NO  → reconsider your design
```

---

## 13. Interview Questions with Model Answers

---

### Q1: What is the Factory Method pattern, and how does it differ from simply calling "new"?

**Answer:**

The Factory Method pattern is a creational GoF design pattern that defines an interface for creating an object, but lets subclasses decide which class to instantiate. The key distinction from calling `new` directly is that `new` hard-codes a compile-time dependency on a concrete class, while Factory Method moves that instantiation call into an overridable method, making it a runtime decision controlled by polymorphism.

When you write `new ConsoleLogger()` inside a service class, that service is now permanently bound to `ConsoleLogger`. You cannot substitute a different implementation without modifying the service. This violates the Dependency Inversion Principle, which states that high-level modules should not depend on low-level modules — both should depend on abstractions.

Factory Method solves this by introducing two layers of abstraction: a `Product` interface (the thing being created) and a `Creator` abstract class (the thing that creates it). The `Creator` defines a factory method — an abstract or virtual method — that returns a `Product`. Concrete subclasses of `Creator` override this method to return specific `ConcreteProduct` instances. The business logic in `Creator` calls the factory method but only knows about the `Product` interface, making it completely decoupled from the concrete implementation.

The practical result: you can add new product types by adding new subclasses, without touching any existing code. The system is open for extension and closed for modification, which is the Open/Closed Principle.

---

### Q2: How does Factory Method enforce the Open/Closed Principle?

**Answer:**

The Open/Closed Principle states that software entities should be open for extension but closed for modification. Factory Method achieves this through Java's polymorphism mechanism.

The `Creator` class and all its template operations (the business logic that uses the product) are the "closed" part — they are written once, tested, and never modified again when you add new product types. The "open" part is the subclassing mechanism: to add a new product, you create a new `ConcreteProduct` class (implementing the `Product` interface) and a new `ConcreteCreator` class (extending `Creator` and overriding `createProduct()`).

For example, if you have a `LoggerFactory` with template operations like `logInfo()` and `logError()`, and you decide to add a `CloudWatchLogger`, you add two new classes: `CloudWatchLogger` (implements `ILogger`) and `CloudWatchLoggerFactory` (extends `LoggerFactory`, overrides `createLogger()`). The `LoggerFactory` class, the template operations inside it, and all other concrete creators like `ConsoleLoggerFactory` and `FileLoggerFactory` remain completely untouched.

This is materially different from a Simple Factory, which has a switch statement. Adding a new product to a switch-based factory requires modifying the factory class itself — violating OCP. With Factory Method, the extension point is the inheritance hierarchy, and Java's `@Override` mechanism ensures that subclasses can provide new behavior without touching parent classes.

In practice, this means that when a new payment method is added (say, Apple Pay), the entire existing payment infrastructure — the abstract factory, all existing concrete factories, the checkout service — has zero lines changed. Only new files are added. This makes Factory Method a powerful tool for building systems that evolve over time without accumulating modification risk.

---

### Q3: What is the difference between the Factory Method pattern and the Abstract Factory pattern?

**Answer:**

Both are GoF creational patterns that abstract object creation, but they solve different problems at different scales.

Factory Method creates one type of product. It is an inheritance-based pattern where the `Creator` has one abstract factory method, and `ConcreteCreator` subclasses override it to produce different implementations of one `Product` type. The relationship is one-to-one: one creator hierarchy for one product hierarchy. The primary intent is to let subclasses decide which concrete product to instantiate.

Abstract Factory creates families of related products. It is a composition-based pattern where an `AbstractFactory` interface declares multiple factory methods — one for each product in the family — and `ConcreteFactory` implementations ensure all products they create are compatible with each other. For example, a `WindowsUIFactory` creates `WindowsButton`, `WindowsTextField`, and `WindowsDialog`, all of which are styled consistently for the Windows platform. A `MacOSUIFactory` creates the corresponding macOS variants. The client receives an `AbstractFactory` and calls it to build a complete, coherent set of components, knowing they will work together.

The critical difference is the family constraint. Abstract Factory guarantees that a `WindowsButton` is never paired with a `MacOSDialog` — because all products come from the same factory. Factory Method has no such constraint; different calls to the factory method could theoretically produce different products.

In terms of implementation: Factory Method uses class inheritance and one virtual method. Abstract Factory uses object composition and an interface with multiple methods. Abstract Factory is often built on top of multiple Factory Methods inside each concrete factory.

Use Factory Method when you have one varying product. Use Abstract Factory when you need coordinated families of related products.

---

### Q4: Can you explain what "defers instantiation to subclasses" means?

**Answer:**

"Defers instantiation to subclasses" is the canonical description of what Factory Method does, and it refers to the use of Java's dynamic dispatch (runtime polymorphism) to delay the decision of which concrete class to instantiate until an actual subclass instance is involved.

In a non-factory design, when `OrderService` writes `new ConsoleLogger()`, the compiler resolves this at compile time. The exact class is known and baked into the bytecode. There is no deferral — the decision is made at write-time.

With Factory Method, the `Creator` class defines `protected abstract ILogger createLogger()` but never calls `new ConsoleLogger()`. When the template operation calls `ILogger logger = createLogger()`, the compiler only knows that a method named `createLogger` will be called and will return an `ILogger`. Which method body actually executes is determined at runtime, based on the actual type of the `Creator` object in memory.

If the object in memory is a `ConsoleLoggerFactory`, then `ConsoleLoggerFactory.createLogger()` is called, and it returns a `ConsoleLogger`. If the object is a `DatabaseLoggerFactory`, then `DatabaseLoggerFactory.createLogger()` runs, returning a `DatabaseLogger`. The `Creator`'s template operations are completely unaware of this distinction — they see only `ILogger`.

The word "defers" is apt because the parent class essentially says "I need to create a product here, but I am going to let whoever instantiated me determine what kind of product gets created." The instantiation decision is delegated upward to whoever is constructing the `ConcreteCreator` — typically the composition root or a dependency injection framework. This is how Factory Method bridges the gap between framework code (the `Creator`) and application code (the `ConcreteCreator`).

---

### Q5: How would you test a class that uses the Factory Method pattern?

**Answer:**

The Factory Method pattern significantly improves testability compared to direct instantiation, and the testing strategy has two layers: testing the creator (the factory) and testing the client that uses the factory.

To test the concrete creator, you verify that it returns the correct product type and that the product is properly configured. This is straightforward unit testing:

```java
@Test
void consoleLoggerFactoryCreatesConsoleLogger() {
    LoggerFactory factory = new ConsoleLoggerFactory();
    ILogger logger = factory.getLogger("test");
    assertThat(logger).isInstanceOf(ConsoleLogger.class);
    assertThat(logger.getLoggerName()).isEqualTo("test");
}
```

To test the client (e.g., `OrderService`), you inject a test double for the factory. Because `OrderService` depends on `LoggerFactory` (the abstraction), you can create a `TestLoggerFactory` that returns an in-memory logger — no file I/O, no database, no console output:

```java
// Test double — captures log messages in memory
class InMemoryLogger implements ILogger {
    private final List<String> messages = new ArrayList<>();
    @Override public void logInfo(String message) { messages.add("INFO: " + message); }
    @Override public void logError(String message, Throwable t) { messages.add("ERROR: " + message); }
    // other methods...
    public List<String> getMessages() { return messages; }
}

class InMemoryLoggerFactory extends LoggerFactory {
    private final InMemoryLogger logger = new InMemoryLogger();
    @Override protected ILogger createLogger(String name) { return logger; }
    public InMemoryLogger getCapturedLogger() { return logger; }
}

@Test
void orderServiceLogsSuccessfulOrder() {
    InMemoryLoggerFactory factory = new InMemoryLoggerFactory();
    OrderService service = new OrderService(factory);

    service.placeOrder("ORD-001", 99.99);

    assertThat(factory.getCapturedLogger().getMessages())
        .anyMatch(m -> m.contains("ORD-001"));
}
```

This is possible only because `OrderService` depends on the abstraction `LoggerFactory` rather than on `ConsoleLogger` directly. The test creates the `OrderService` with a test factory — the Factory Method pattern made the seam available for the test double.

Alternatively, using Mockito:

```java
@Test
void orderServiceUsesProvidedLogger() {
    ILogger mockLogger = mock(ILogger.class);
    LoggerFactory mockFactory = mock(LoggerFactory.class);
    when(mockFactory.getLogger(anyString())).thenReturn(mockLogger);

    OrderService service = new OrderService(mockFactory);
    service.placeOrder("ORD-TEST", 50.0);

    verify(mockLogger).logInfo(contains("ORD-TEST"));
}
```

---

### Q6: When would you choose Factory Method over dependency injection?

**Answer:**

This is an excellent real-world trade-off question. Factory Method and dependency injection (DI) both decouple clients from concrete types, but they operate at different layers and serve different purposes.

Dependency injection (DI) is the right tool when the concrete implementation is known at application startup and remains constant for the lifetime of the object being created. For example, a `CheckoutService` that always uses a `PayPalProcessor` in production can have that processor injected by Spring or Guice. DI makes the dependency explicit in the constructor, which improves readability and testability. It works especially well when you have a DI container managing the object lifecycle.

Factory Method is the right tool when the concrete type to be created is determined at runtime, possibly for each individual call. If `CheckoutService` needs to create a new `PaymentProcessor` for each transaction, and the type of processor depends on the customer's payment method preference (which is only known at request time), DI cannot inject the right processor at startup because the decision has not been made yet. A factory is needed to create the correct processor on demand.

A practical example: an `OrderService` that always uses a `ConsoleLogger` in development is a good candidate for DI — inject `ILogger` directly. But a service that creates temporary worker objects (like HTTP clients per request, or logger instances per class name) is a good candidate for Factory Method — inject the factory, call it when needed.

In Spring applications, these often coexist. Spring manages the factory as a singleton bean (DI handles wiring the factory). The factory, in turn, uses Factory Method to create per-request or per-operation objects that Spring does not manage.

A rule of thumb: if the product's concrete type varies per call or per operation, use Factory Method. If the product type is fixed for the application's lifetime, use DI directly.

---

### Q7: What is a "hook method" in the context of Factory Method, and how does it relate to Template Method?

**Answer:**

Factory Method and Template Method are deeply connected — in fact, the GoF book describes them as "often used together." Understanding their relationship reveals the full power of the Factory Method pattern.

Template Method defines the skeleton of an algorithm in a base class and lets subclasses fill in specific steps via overridable hook methods. The algorithm's structure is fixed; only the variable steps change.

Factory Method is, architecturally, a special kind of hook method used specifically for object creation. The `Creator` class defines a template operation (the algorithm skeleton) that includes a step: "create a product." Instead of hard-coding which product to create, it delegates this step to an abstract factory method — the hook. Subclasses implement the hook to return their specific product.

In the `PaymentProcessorFactory` example, the `pay()` method is the template method. It defines a fixed algorithm: call `beforePayment()`, call `processor.process()`, call `afterPayment()`. The `createProcessor()` is the Factory Method hook. `beforePayment()` and `afterPayment()` are additional hooks for pre/post processing, with default no-op implementations that subclasses may optionally override.

This dual-hook design gives tremendous flexibility. A `CreditCardProcessorFactory` overrides `createProcessor()` (required hook — specifies which product) and also overrides `beforePayment()` (optional hook — adds fraud screening). A `CryptoProcessorFactory` overrides both `createProcessor()` and `afterPayment()` (for confirmation polling). The base template method `pay()` is untouched.

Understanding this relationship matters in interviews because it demonstrates you grasp not just the mechanics of Factory Method but its role in larger design compositions. Many real-world framework designs (Spring's `AbstractApplicationContext`, Hibernate's session management, JDBC's connection templates) use this Template Method + Factory Method combination.

---

### Q8: What are the main criticisms of the Factory Method pattern, and how would you address them?

**Answer:**

Factory Method is not without legitimate criticisms, and a balanced answer is important in senior-level interviews.

The most common criticism is class proliferation. Every new product type requires two new classes: the concrete product and its concrete creator. In a large system with many product variants, this can result in dozens of shallow classes, each with only a few lines of code. Critics argue this creates navigational overhead — a developer needs to understand the entire hierarchy before making changes.

The response is nuanced: class proliferation is a real cost, but it is a cost paid upfront in return for long-term stability. The alternative — a switch-based factory or scattered `if-else` chains — is superficially simpler but becomes increasingly expensive to maintain as the number of product types grows. Each modification to a switch factory is a change to a central, heavily-tested class, carrying regression risk. The Factory Method distributes that risk across isolated, independently testable classes. That said, if a system genuinely has only two or three fixed product types, a Simple Factory may be more pragmatic.

The second criticism is that Factory Method relies on inheritance, which modern Java design tends to avoid in favor of composition. Inheritance hierarchies are rigid — changing the `Creator`'s constructor signature forces all `ConcreteCreator` subclasses to update. If the system has many creators, this ripple effect is painful.

The composition-based answer to this criticism is to use a functional interface or a `Supplier<Product>` lambda as the factory, which achieves the same decoupling without inheritance:

```java
public class LoggerService {
    private final Supplier<ILogger> loggerSupplier;

    public LoggerService(Supplier<ILogger> loggerSupplier) {
        this.loggerSupplier = loggerSupplier;
    }

    public void doWork() {
        ILogger logger = loggerSupplier.get();
        logger.logInfo("Working...");
    }
}

// Usage — no subclassing needed
LoggerService service = new LoggerService(() -> new ConsoleLogger("app"));
```

This `Supplier`-based approach is often preferable in modern Java (8+) for simple cases. Reserve the full Factory Method class hierarchy for complex cases where the factory itself needs state, configuration, or overridable template operations.

---

### Q9: How would you implement a registry-based factory that combines Factory Method with a registry to avoid the proliferation problem?

**Answer:**

A registry-based factory is a sophisticated evolution of the pattern that eliminates some of the class proliferation criticism while preserving extensibility. The idea is to combine Factory Method's abstraction with a runtime registry that maps keys to factory methods (or `Supplier`s).

```java
package com.course.patterns.factory.registry;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Supplier;

/**
 * Registry-based factory — combines Factory Method with a self-registration mechanism.
 * Products register themselves at startup; clients look up by key.
 * New product types can be added by calling register() — no class hierarchy needed.
 */
public final class LoggerRegistry {

    // Thread-safe registry of factory suppliers
    private static final Map<String, Supplier<ILogger>> REGISTRY = new ConcurrentHashMap<>();

    // Static initializer block registers the built-in types
    static {
        register("console",  () -> new ConsoleLogger("default"));
        register("file",     () -> new FileLogger("default", "/tmp/app.log"));
        register("database", () -> new DatabaseLogger("default", "jdbc:...", "user", "pass"));
    }

    private LoggerRegistry() {} // utility class — no instances

    /**
     * Register a new logger type. Calling code can extend the factory
     * at runtime without modifying this class.
     */
    public static void register(String key, Supplier<ILogger> factory) {
        if (key == null || key.isBlank()) {
            throw new IllegalArgumentException("key must not be blank");
        }
        if (factory == null) {
            throw new IllegalArgumentException("factory supplier must not be null");
        }
        REGISTRY.put(key.toLowerCase(), factory);
    }

    /**
     * Look up a logger by type key.
     */
    public static ILogger getLogger(String type, String loggerName) {
        Supplier<ILogger> supplier = REGISTRY.get(type.toLowerCase());
        if (supplier == null) {
            throw new IllegalArgumentException(
                    "No logger registered for type '" + type + "'. "
                  + "Registered types: " + REGISTRY.keySet());
        }
        return supplier.get();
    }
}

// Adding a new type at startup — zero changes to LoggerRegistry
LoggerRegistry.register("cloudwatch", () -> new CloudWatchLogger("us-east-1", "my-log-group"));
ILogger cwLogger = LoggerRegistry.getLogger("cloudwatch", "OrderService");
```

This approach trades the strict OCP compliance of the pure Factory Method (where you never modify existing classes) for the convenience of a single registry class. It is a pragmatic middle ground widely used in plugin systems, codec registries (like Java's `ImageIO`), and serialization frameworks.

The pure Factory Method hierarchy and the registry approach are complementary: you can use Factory Method within each registered `Supplier` to handle complex construction, while the registry handles the dispatch.

---

*End of Factory Method Pattern Guide*

---

**Related Patterns:**
- [Abstract Factory](./03-abstract-factory.md) — Creates families of related objects
- [Singleton](./04-singleton.md) — Ensures only one instance of a class exists
- [Builder](./05-builder.md) — Constructs complex objects step by step
- [Template Method](../../05-behavioral-patterns/template-method.md) — Defines algorithm skeleton with subclass-filled steps
- [Strategy](../../05-behavioral-patterns/strategy.md) — Composition-based alternative for varying behavior
