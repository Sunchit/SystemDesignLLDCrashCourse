# Dependency Inversion Principle (DIP)

> Module 01 — SOLID Principles | Lesson 05 of 05
> Prerequisites: SRP, OCP, LSP, ISP | Estimated reading time: 75 minutes

---

## Table of Contents

1. [The Principle Stated Precisely](#1-the-principle-stated-precisely)
2. [High-Level vs Low-Level: What the Terms Actually Mean](#2-high-level-vs-low-level-what-the-terms-actually-mean)
3. [Why Direct Dependencies Hurt](#3-why-direct-dependencies-hurt)
4. [The Dependency Graph Inversion](#4-the-dependency-graph-inversion)
5. [Complete Before/After — Notification System](#5-complete-beforeafter--notification-system)
6. [Complete Before/After — User Repository](#6-complete-beforeafter--user-repository)
7. [Ownership of the Abstraction](#7-ownership-of-the-abstraction)
8. [DIP vs Dependency Injection: They Are Not the Same Thing](#8-dip-vs-dependency-injection-they-are-not-the-same-thing)
9. [DIP and Testability](#9-dip-and-testability)
10. [DI Containers: Spring and Guice](#10-di-containers-spring-and-guice)
11. [Ports and Adapters / Hexagonal Architecture](#11-ports-and-adapters--hexagonal-architecture)
12. [The Common Mistake: Interface Mirroring](#12-the-common-mistake-interface-mirroring)
13. [Interview Questions with Model Answers](#13-interview-questions-with-model-answers)
14. [Assignment](#14-assignment)

---

## 1. The Principle Stated Precisely

Robert C. Martin stated DIP in two rules:

**Rule A.** High-level modules should not depend on low-level modules. Both should depend on abstractions.

**Rule B.** Abstractions should not depend on details. Details should depend on abstractions.

Both rules address the same root problem from different directions. Rule A says the dependency arrows in your code should point toward abstractions, not toward concrete implementations. Rule B says those abstractions must be defined in terms of what callers need, not in terms of what implementors expose. Together they prevent the most destructive form of coupling in large systems: business logic that is structurally entangled with infrastructure.

A violation of DIP is not a style issue. It means your core business logic cannot be compiled, tested, or reasoned about without dragging along your database driver, your email provider, your file system, or whatever other infrastructure detail has been hardwired into it. That is an architectural defect.

---

## 2. High-Level vs Low-Level: What the Terms Actually Mean

The terms "high-level" and "low-level" refer to the position of a module in the dependency hierarchy, not to its complexity or importance.

**High-level module:** Contains policy — the rules of what the system does. It orchestrates workflows. It encodes business logic. It answers questions like "when an order is placed, what must happen?" Examples: `OrderService`, `PaymentProcessor`, `UserRegistrationWorkflow`, `ReportGenerator`.

**Low-level module:** Contains mechanism — the implementation of how something is done. It knows about specific technologies, protocols, and infrastructure. It answers questions like "how do I send bytes over SMTP?" or "how do I write a row to MySQL?" Examples: `EmailNotifier`, `MySQLUserRepository`, `S3FileUploader`, `PDFExporter`.

The critical insight is that high-level modules express the intent of the system. Low-level modules are interchangeable execution details. The intent should not be coupled to any particular execution detail because execution details change frequently (you migrate databases, you add notification channels, you switch cloud providers) while business rules change for fundamentally different reasons.

When `OrderService` contains `new EmailNotifier()`, it has taken on a structural dependency on a specific execution detail. The source of that object — the fact that it is instantiated with `new` — is the dependency. The type annotation is where the dependency lives in the source file. The moment you write that line, your business policy module becomes untestable without an SMTP server and unextensible without editing the policy module itself.

---

## 3. Why Direct Dependencies Hurt

Consider this simple scenario before writing any code:

**Tight coupling.** If `OrderService` directly references `EmailNotifier`, any change to `EmailNotifier`'s constructor, method signatures, or package location ripples up into `OrderService`. The two modules are coupled in both the forward direction (OrderService uses EmailNotifier) and the backward direction (changes to EmailNotifier force changes in OrderService). This is the dependency that DIP inverts.

**Impossible to unit test without real infrastructure.** When `OrderService` instantiates `EmailNotifier` internally, there is no seam where a test can substitute a fake. A unit test of `OrderService.placeOrder()` either sends a real email or fails because the SMTP server is unreachable. Neither outcome is acceptable for a unit test. Your unit test suite slows down, becomes flaky, and loses its value as a fast feedback loop.

**Cannot swap implementations without modifying the caller.** If the business requirement changes from "send email" to "send email and SMS," you must open `OrderService` and edit it. This violates OCP, but the root cause is DIP violation. The high-level module owns the decision about which low-level mechanism to invoke. Once you invert the dependency, the high-level module is blind to which implementation is wired in — you add a new implementation without touching the policy class at all.

**Compilation coupling.** In large Java codebases with modular builds (Gradle multi-project, Maven multi-module), a direct dependency forces the high-level module's build unit to declare a compile-time dependency on the low-level module's build unit. This means infrastructure modules end up on the compile classpath of business logic modules. In a well-structured system, the dependency should go the other way: infrastructure depends on the business module to learn the interface it must implement.

---

## 4. The Dependency Graph Inversion

This ASCII diagram shows exactly what "inverting the dependency" means at the structural level.

### Before DIP: Direct Dependency

```
+----------------+          +------------------+
|  OrderService  | -------> |  EmailNotifier   |
| (high-level)   |          | (low-level)      |
+----------------+          +------------------+
        |
        |                   +------------------+
        +-----------------> |  SMSNotifier     |
                            | (low-level)      |
                            +------------------+
```

The arrows represent "depends on / knows about." `OrderService` knows the concrete type names `EmailNotifier` and `SMSNotifier`. Any new channel requires editing `OrderService`.

### After DIP: Inverted Dependency

```
+----------------+          +------------------------+
|  OrderService  | -------> |  NotificationService   |
| (high-level)   |          |  <<interface>>         |
+----------------+          | (owned by high-level   |
                            |  module's package)     |
                            +------------------------+
                                       ^
                                       |  implements
                    +------------------+------------------+
                    |                                     |
       +---------------------------+     +-----------------------------+
       | EmailNotificationService  |     | SMSNotificationService      |
       | (low-level)               |     | (low-level)                 |
       +---------------------------+     +-----------------------------+
```

The `OrderService` arrow now points at an abstraction (`NotificationService`), not at any concrete class. The concrete classes (`EmailNotificationService`, `SMSNotificationService`) point *toward* the abstraction to satisfy it. The dependency arrow from the infrastructure side has been inverted — it now flows toward the interface, not away from it.

Crucially, the interface lives in the same package or module as `OrderService`, not in the same package as the implementors. This is the ownership rule discussed in section 7.

---

## 5. Complete Before/After — Notification System

### BEFORE: DIP Violated

```java
// Low-level module: knows SMTP details
public class EmailNotifier {

    private final String smtpHost;
    private final int smtpPort;

    public EmailNotifier(String smtpHost, int smtpPort) {
        this.smtpHost = smtpHost;
        this.smtpPort = smtpPort;
    }

    public void sendEmailNotification(String recipientEmail, String subject, String body) {
        // connects to SMTP, sends email
        System.out.println("Sending email via " + smtpHost + ":" + smtpPort
            + " to " + recipientEmail + " | Subject: " + subject);
    }
}

// Low-level module: knows SMS gateway details
public class SMSNotifier {

    private final String gatewayUrl;
    private final String apiKey;

    public SMSNotifier(String gatewayUrl, String apiKey) {
        this.gatewayUrl = gatewayUrl;
        this.apiKey = apiKey;
    }

    public void sendSMSNotification(String phoneNumber, String message) {
        // calls SMS gateway REST API
        System.out.println("Sending SMS via " + gatewayUrl
            + " to " + phoneNumber + " | Message: " + message);
    }
}

// High-level module: directly instantiates and calls concrete low-level classes
// Problems:
//  1. OrderService is coupled to EmailNotifier and SMSNotifier by concrete type
//  2. Adding PushNotifier requires modifying this class (OCP violation, caused by DIP violation)
//  3. Cannot unit-test OrderService without a live SMTP server and SMS gateway
//  4. Method signatures differ (sendEmailNotification vs sendSMSNotification), so
//     OrderService must know the API of every notifier individually
public class OrderService {

    // Hard-wired concrete dependencies — DIP violation
    private final EmailNotifier emailNotifier;
    private final SMSNotifier smsNotifier;

    public OrderService() {
        // OrderService owns the construction of its dependencies — another DIP violation
        this.emailNotifier = new EmailNotifier("smtp.company.com", 587);
        this.smsNotifier   = new SMSNotifier("https://sms.gateway.com/api", "secret-key");
    }

    public void placeOrder(Order order) {
        // Business logic
        order.setStatus(OrderStatus.CONFIRMED);
        order.setConfirmedAt(System.currentTimeMillis());
        saveOrder(order);

        // Notification: hardwired to email and SMS
        String subject = "Order Confirmed: " + order.getId();
        String body    = "Your order for " + order.getItemCount() + " items has been confirmed.";
        emailNotifier.sendEmailNotification(order.getCustomerEmail(), subject, body);

        String sms = "Order " + order.getId() + " confirmed. Total: $" + order.getTotal();
        smsNotifier.sendSMSNotification(order.getCustomerPhone(), sms);

        // Adding push notification requires editing this method — OCP violation
        // pushNotifier.sendPushNotification(...);  // must add a new field, change constructor, add code here
    }

    private void saveOrder(Order order) {
        // persist order
    }
}
```

**What is wrong:**

- `OrderService` creates its own dependencies with `new`. There is no way to substitute a fake for testing.
- `OrderService` calls two different method names (`sendEmailNotification`, `sendSMSNotification`) because it knows both concrete types. It cannot treat them uniformly.
- Adding `PushNotifier` requires: adding a field, calling `new` in the constructor, calling a third method name in `placeOrder`. `OrderService` must be opened and edited.

---

### AFTER: DIP Applied

The restructuring proceeds in three steps:

1. Define the abstraction — an interface owned by the high-level module's package.
2. Make low-level classes implement the interface.
3. Change `OrderService` to depend only on the interface, receiving it from outside (constructor injection).

```java
// Step 1: Define the abstraction in the high-level module's package
// Package: com.company.orders (same package as OrderService)
// This interface belongs to the CALLER, not to the implementors.
public interface NotificationService {

    /**
     * Sends a notification to the customer associated with the given order.
     * Implementations decide the channel (email, SMS, push, etc.).
     * The caller does not know or care which channel is used.
     *
     * @param order   the confirmed order containing customer contact details
     * @param subject a short summary of the notification event
     * @param message the full notification body
     */
    void notifyCustomer(Order order, String subject, String message);
}
```

```java
// Step 2a: Email implementation
// Package: com.company.notifications.email (infrastructure layer)
// This class depends on the interface defined in the business layer — the arrow
// now flows FROM infrastructure TOWARD business, not the other way around.
public class EmailNotificationService implements NotificationService {

    private final String smtpHost;
    private final int smtpPort;

    public EmailNotificationService(String smtpHost, int smtpPort) {
        this.smtpHost = smtpHost;
        this.smtpPort = smtpPort;
    }

    @Override
    public void notifyCustomer(Order order, String subject, String message) {
        String recipient = order.getCustomerEmail();
        // Real implementation would use JavaMail or similar
        System.out.println("[EMAIL] To: " + recipient
            + " | Host: " + smtpHost + ":" + smtpPort
            + " | Subject: " + subject
            + " | Body: " + message);
    }
}
```

```java
// Step 2b: SMS implementation
// Package: com.company.notifications.sms (infrastructure layer)
public class SMSNotificationService implements NotificationService {

    private final String gatewayUrl;
    private final String apiKey;

    public SMSNotificationService(String gatewayUrl, String apiKey) {
        this.gatewayUrl = gatewayUrl;
        this.apiKey = apiKey;
    }

    @Override
    public void notifyCustomer(Order order, String subject, String message) {
        String phone = order.getCustomerPhone();
        String sms   = subject + ": " + message;
        // Real implementation would call SMS gateway REST API
        System.out.println("[SMS] To: " + phone
            + " | Gateway: " + gatewayUrl
            + " | Message: " + sms);
    }
}
```

```java
// Step 2c: Push notification implementation — added WITHOUT touching OrderService
// Package: com.company.notifications.push (infrastructure layer)
public class PushNotificationService implements NotificationService {

    private final String fcmServerKey;

    public PushNotificationService(String fcmServerKey) {
        this.fcmServerKey = fcmServerKey;
    }

    @Override
    public void notifyCustomer(Order order, String subject, String message) {
        String deviceToken = order.getCustomerDeviceToken();
        // Real implementation would call Firebase Cloud Messaging API
        System.out.println("[PUSH] Token: " + deviceToken
            + " | FCM Key: " + fcmServerKey.substring(0, 8) + "..."
            + " | Title: " + subject
            + " | Body: " + message);
    }
}
```

```java
// Step 2d: Composite implementation — sends through all channels
// This is a structural pattern (Composite) layered on top of DIP.
// OrderService still depends on one NotificationService — it has no idea this
// implementation fans out to multiple channels.
// Package: com.company.notifications (infrastructure layer)
import java.util.List;

public class CompositeNotificationService implements NotificationService {

    private final List<NotificationService> delegates;

    public CompositeNotificationService(List<NotificationService> delegates) {
        this.delegates = List.copyOf(delegates);
    }

    @Override
    public void notifyCustomer(Order order, String subject, String message) {
        for (NotificationService delegate : delegates) {
            try {
                delegate.notifyCustomer(order, subject, message);
            } catch (RuntimeException e) {
                // Log failure but do not let one channel failure prevent others
                System.err.println("Notification channel failed: "
                    + delegate.getClass().getSimpleName() + " — " + e.getMessage());
            }
        }
    }
}
```

```java
// Step 3: Refactored OrderService — depends only on the abstraction
// Package: com.company.orders
public class OrderService {

    private final OrderRepository orderRepository;
    private final NotificationService notificationService; // abstraction, not a concrete class

    // Constructor injection: the dependency is provided by the caller, not created here.
    // OrderService has no idea whether this is email, SMS, push, or a composite.
    public OrderService(OrderRepository orderRepository, NotificationService notificationService) {
        this.orderRepository     = orderRepository;
        this.notificationService = notificationService;
    }

    public void placeOrder(Order order) {
        // Business logic unchanged regardless of notification channel
        order.setStatus(OrderStatus.CONFIRMED);
        order.setConfirmedAt(System.currentTimeMillis());
        orderRepository.save(order);

        // Uniform call through the abstraction
        String subject = "Order Confirmed: " + order.getId();
        String message = "Your order for " + order.getItemCount()
            + " items has been confirmed. Total: $" + order.getTotal();
        notificationService.notifyCustomer(order, subject, message);
    }
}
```

```java
// Wiring: this is the composition root — the only place that knows concrete types
// In a Spring application this would be a @Configuration class or use @Autowired
// In a plain Java application it is the main method or an ApplicationFactory
public class ApplicationFactory {

    public static OrderService buildOrderService() {
        // Infrastructure details are assembled here, at the boundary of the application
        NotificationService email = new EmailNotificationService("smtp.company.com", 587);
        NotificationService sms   = new SMSNotificationService("https://sms.gateway.com/api", "secret-key");
        NotificationService push  = new PushNotificationService("fcm-server-key-abc123");

        NotificationService composite = new CompositeNotificationService(
            List.of(email, sms, push)
        );

        OrderRepository repository = new JdbcOrderRepository("jdbc:postgresql://localhost/orders");

        return new OrderService(repository, composite);
    }
}
```

**What changed and why it matters:**

- `OrderService` has zero references to `EmailNotificationService`, `SMSNotificationService`, or `PushNotificationService`. Adding a new channel is purely additive — write a new class, update the wiring. `OrderService` is never touched.
- The method signature is uniform: every implementation conforms to `notifyCustomer(Order, String, String)`. `OrderService` cannot accidentally call email-specific methods on the SMS implementation.
- `OrderService` is now fully testable without any real infrastructure. A test passes a mock `NotificationService`.

---

## 6. Complete Before/After — User Repository

A second, independent example to reinforce the pattern with a different domain.

### BEFORE: DIP Violated

```java
// Low-level module: coupled to MySQL
public class MySQLUserRepository {

    private final Connection connection;

    public MySQLUserRepository(String jdbcUrl, String username, String password)
            throws SQLException {
        this.connection = DriverManager.getConnection(jdbcUrl, username, password);
    }

    public void save(User user) throws SQLException {
        PreparedStatement stmt = connection.prepareStatement(
            "INSERT INTO users (id, email, name) VALUES (?, ?, ?)"
        );
        stmt.setString(1, user.getId());
        stmt.setString(2, user.getEmail());
        stmt.setString(3, user.getName());
        stmt.executeUpdate();
    }

    public User findByEmail(String email) throws SQLException {
        PreparedStatement stmt = connection.prepareStatement(
            "SELECT id, email, name FROM users WHERE email = ?"
        );
        stmt.setString(1, email);
        ResultSet rs = stmt.executeQuery();
        if (rs.next()) {
            return new User(rs.getString("id"), rs.getString("email"), rs.getString("name"));
        }
        return null;
    }
}

// High-level module: directly bound to MySQL
// Problems:
//  1. Cannot test UserService without a real MySQL database
//  2. Migrating to PostgreSQL or MongoDB requires editing UserService
//  3. UserService imports and handles SQLException — a MySQL-specific concern
public class UserService {

    private final MySQLUserRepository userRepository; // DIP violation: concrete type

    public UserService() throws SQLException {
        // DIP violation: UserService constructs its own repository
        this.userRepository = new MySQLUserRepository(
            "jdbc:mysql://localhost:3306/appdb", "root", "password"
        );
    }

    public void registerUser(String id, String email, String name) throws SQLException {
        User existing = userRepository.findByEmail(email);
        if (existing != null) {
            throw new IllegalArgumentException("Email already registered: " + email);
        }
        User user = new User(id, email, name);
        userRepository.save(user);
        System.out.println("User registered: " + email);
    }
}
```

### AFTER: DIP Applied

```java
// Abstraction — defined in the business/domain layer
// Package: com.company.users (same package as UserService)
public interface UserRepository {

    void save(User user);

    User findByEmail(String email);

    boolean existsByEmail(String email);
}
```

```java
// MySQL implementation — infrastructure layer
// Package: com.company.persistence.mysql
import java.sql.*;

public class MySQLUserRepository implements UserRepository {

    private final Connection connection;

    public MySQLUserRepository(String jdbcUrl, String username, String password)
            throws SQLException {
        this.connection = DriverManager.getConnection(jdbcUrl, username, password);
    }

    @Override
    public void save(User user) {
        try {
            PreparedStatement stmt = connection.prepareStatement(
                "INSERT INTO users (id, email, name) VALUES (?, ?, ?)"
            );
            stmt.setString(1, user.getId());
            stmt.setString(2, user.getEmail());
            stmt.setString(3, user.getName());
            stmt.executeUpdate();
        } catch (SQLException e) {
            // Translate infrastructure exception to domain exception
            throw new RepositoryException("Failed to save user: " + user.getEmail(), e);
        }
    }

    @Override
    public User findByEmail(String email) {
        try {
            PreparedStatement stmt = connection.prepareStatement(
                "SELECT id, email, name FROM users WHERE email = ?"
            );
            stmt.setString(1, email);
            ResultSet rs = stmt.executeQuery();
            if (rs.next()) {
                return new User(rs.getString("id"), rs.getString("email"), rs.getString("name"));
            }
            return null;
        } catch (SQLException e) {
            throw new RepositoryException("Failed to find user by email: " + email, e);
        }
    }

    @Override
    public boolean existsByEmail(String email) {
        return findByEmail(email) != null;
    }
}
```

```java
// In-memory implementation — used in tests and local development
// Package: com.company.persistence.inmemory
import java.util.HashMap;
import java.util.Map;

public class InMemoryUserRepository implements UserRepository {

    private final Map<String, User> storageByEmail = new HashMap<>();

    @Override
    public void save(User user) {
        storageByEmail.put(user.getEmail(), user);
    }

    @Override
    public User findByEmail(String email) {
        return storageByEmail.get(email);
    }

    @Override
    public boolean existsByEmail(String email) {
        return storageByEmail.containsKey(email);
    }
}
```

```java
// Refactored UserService — no knowledge of MySQL, PostgreSQL, or any database
// Package: com.company.users
public class UserService {

    private final UserRepository userRepository; // abstraction

    // Constructor injection
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public void registerUser(String id, String email, String name) {
        // Business rule: email must be unique
        if (userRepository.existsByEmail(email)) {
            throw new IllegalArgumentException("Email already registered: " + email);
        }
        User user = new User(id, email, name);
        userRepository.save(user);
    }

    public User getUser(String email) {
        User user = userRepository.findByEmail(email);
        if (user == null) {
            throw new UserNotFoundException("No user found with email: " + email);
        }
        return user;
    }
}
```

Notice that `UserService` no longer handles `SQLException`. The infrastructure exception is translated to a domain exception (`RepositoryException`) inside the repository implementation. The high-level module sees only domain-level exceptions.

---

## 7. Ownership of the Abstraction

This is the most frequently misunderstood aspect of DIP. The interface must be owned by the high-level module, not by the low-level module.

**Wrong ownership:**

```
com.company.notifications.NotificationService  ← interface lives with implementors
com.company.notifications.EmailNotificationService implements NotificationService
com.company.notifications.SMSNotificationService implements NotificationService
com.company.orders.OrderService depends on com.company.notifications.NotificationService
```

Even though an interface exists, `OrderService`'s package must depend on `com.company.notifications` to compile. If the notifications package is in a separate build module, that module is on the compile classpath of the orders module. The dependency direction in the build graph is still: orders depends on notifications.

**Correct ownership:**

```
com.company.orders.NotificationService          ← interface lives with its CALLER
com.company.notifications.EmailNotificationService implements com.company.orders.NotificationService
com.company.notifications.SMSNotificationService implements com.company.orders.NotificationService
com.company.orders.OrderService depends on com.company.orders.NotificationService
```

Now the build graph is: notifications depends on orders (to implement the interface). Orders depends on nothing in the notifications package. This is the inverted dependency. The infrastructure module knows about the business module; the business module does not know about the infrastructure module.

This is what "inversion" means. Not just "use an interface" — but "who owns the interface determines the direction of the dependency arrow."

---

## 8. DIP vs Dependency Injection: They Are Not the Same Thing

This confusion appears in nearly every technical interview. Candidates say "I use dependency injection" when asked about DIP, and interviewers who do not probe further assume they understand the principle. They often do not.

**DIP is a design principle.** It describes a structural property of source code: the direction in which dependencies are allowed to flow. DIP tells you what the dependency graph must look like. It says nothing about how dependencies are created or delivered.

**Dependency Injection (DI) is a technique.** It describes a mechanism for providing dependencies to an object from outside that object. DI is one implementation technique that satisfies DIP, but it is not the only one, and applying DI does not automatically mean you have satisfied DIP.

### Three Forms of Dependency Injection

**Constructor Injection** (preferred for mandatory dependencies):

```java
public class OrderService {

    private final NotificationService notificationService;
    private final OrderRepository orderRepository;

    // All dependencies declared, all required at construction time.
    // The object is always in a valid state after the constructor returns.
    // Makes dependencies explicit and discoverable.
    public OrderService(NotificationService notificationService,
                        OrderRepository orderRepository) {
        this.notificationService = notificationService;
        this.orderRepository     = orderRepository;
    }
}
```

**Setter Injection** (for optional or reconfigurable dependencies):

```java
public class OrderService {

    private NotificationService notificationService;
    private OrderRepository orderRepository;

    // Allows changing the implementation after construction.
    // Risk: object may be used before the dependency is set.
    // Prefer constructor injection for mandatory dependencies.
    public void setNotificationService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    public void setOrderRepository(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

**Method Injection** (when the dependency is per-call, not per-object):

```java
public class ReportService {

    // The output format varies per call, not per instance.
    // Injecting it at the method level is appropriate.
    public Report generateReport(ReportData data, ReportFormatter formatter) {
        return formatter.format(data);
    }
}
```

### You Can Have DIP Without a DI Container

The `ApplicationFactory` in section 5 demonstrates manual wiring. There is no Spring, no Guice, no framework. DIP is satisfied because `OrderService` depends on the abstraction. DI is achieved by constructor injection. No container is involved.

```java
// Pure Java, no framework: DIP + DI without a container
public class Main {
    public static void main(String[] args) throws Exception {
        UserRepository userRepository = new MySQLUserRepository(
            "jdbc:mysql://localhost:3306/appdb", "root", "secret"
        );
        UserService userService = new UserService(userRepository); // DI via constructor
        userService.registerUser("u-001", "alice@example.com", "Alice");
    }
}
```

DIP says: `UserService` must depend on `UserRepository` (abstraction), not on `MySQLUserRepository` (concrete). DI says: the `MySQLUserRepository` instance is passed in, not created inside `UserService`. They cooperate but are separable concepts.

---

## 9. DIP and Testability

DIP is the enabler of fast, isolated unit tests. Once your high-level modules depend only on interfaces, tests substitute lightweight fakes for all infrastructure. No database, no network, no file system.

### Concrete Test Example

```java
// Test double: a simple spy that records calls
// Package: com.company.orders.test (or src/test/java)
public class SpyNotificationService implements NotificationService {

    private int callCount = 0;
    private Order lastOrderNotified;
    private String lastSubject;
    private String lastMessage;

    @Override
    public void notifyCustomer(Order order, String subject, String message) {
        this.callCount++;
        this.lastOrderNotified = order;
        this.lastSubject       = subject;
        this.lastMessage       = message;
    }

    public int getCallCount() { return callCount; }
    public Order getLastOrderNotified() { return lastOrderNotified; }
    public String getLastSubject() { return lastSubject; }
    public String getLastMessage() { return lastMessage; }

    public boolean wasNotified() { return callCount > 0; }
}
```

```java
// Test double: an in-memory repository — reused from section 6
// Already shown: InMemoryUserRepository
```

```java
// Unit test for OrderService — no SMTP, no database, no network
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

public class OrderServiceTest {

    private SpyNotificationService spyNotificationService;
    private InMemoryOrderRepository inMemoryOrderRepository;
    private OrderService orderService;

    @BeforeEach
    void setUp() {
        spyNotificationService  = new SpyNotificationService();
        inMemoryOrderRepository = new InMemoryOrderRepository();
        // Constructor injection: test provides fakes, not real infrastructure
        orderService = new OrderService(inMemoryOrderRepository, spyNotificationService);
    }

    @Test
    void placeOrder_shouldConfirmOrderAndSaveIt() {
        // Arrange
        Order order = new Order("ord-001", "customer@example.com", "+14155551234", 3, 149.99);

        // Act
        orderService.placeOrder(order);

        // Assert: order state
        assertEquals(OrderStatus.CONFIRMED, order.getStatus());
        assertNotNull(order.getConfirmedAt());
        assertTrue(inMemoryOrderRepository.contains("ord-001"));
    }

    @Test
    void placeOrder_shouldNotifyCustomerExactlyOnce() {
        // Arrange
        Order order = new Order("ord-002", "customer@example.com", "+14155551234", 1, 29.99);

        // Act
        orderService.placeOrder(order);

        // Assert: notification was sent
        assertTrue(spyNotificationService.wasNotified(),
            "Customer should have been notified after order placement");
        assertEquals(1, spyNotificationService.getCallCount(),
            "Notification should be sent exactly once");
    }

    @Test
    void placeOrder_shouldIncludeOrderIdInNotificationSubject() {
        // Arrange
        Order order = new Order("ord-003", "customer@example.com", "+14155551234", 2, 89.99);

        // Act
        orderService.placeOrder(order);

        // Assert: notification content contains order ID
        String subject = spyNotificationService.getLastSubject();
        assertTrue(subject.contains("ord-003"),
            "Notification subject should contain the order ID. Actual: " + subject);
    }

    @Test
    void placeOrder_shouldPassCorrectOrderToNotificationService() {
        // Arrange
        Order order = new Order("ord-004", "alice@example.com", "+14155559876", 5, 299.99);

        // Act
        orderService.placeOrder(order);

        // Assert: the correct order was passed to the notification service
        Order notifiedOrder = spyNotificationService.getLastOrderNotified();
        assertEquals("ord-004", notifiedOrder.getId());
        assertEquals("alice@example.com", notifiedOrder.getCustomerEmail());
    }
}
```

These tests execute in milliseconds. They are deterministic. They can run offline. They test `OrderService` behavior in complete isolation from `EmailNotificationService`. None of this is possible when `OrderService` instantiates its own dependencies.

The `SpyNotificationService` is a hand-rolled test double. In real projects you would use a mocking library (Mockito is the Java standard):

```java
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import static org.mockito.Mockito.*;

public class OrderServiceMockitoTest {

    @Test
    void placeOrder_shouldCallNotifyCustomerOnce() {
        // Arrange
        NotificationService mockNotificationService = mock(NotificationService.class);
        OrderRepository mockRepository              = mock(OrderRepository.class);
        OrderService orderService = new OrderService(mockRepository, mockNotificationService);
        Order order = new Order("ord-005", "bob@example.com", "+14155550000", 1, 49.99);

        // Act
        orderService.placeOrder(order);

        // Assert: verify the abstraction was called exactly once with the right order
        verify(mockNotificationService, times(1)).notifyCustomer(eq(order), anyString(), anyString());
    }

    @Test
    void placeOrder_shouldSaveOrderBeforeNotifying() {
        // Arrange
        NotificationService mockNotificationService = mock(NotificationService.class);
        OrderRepository mockRepository              = mock(OrderRepository.class);
        OrderService orderService = new OrderService(mockRepository, mockNotificationService);
        Order order = new Order("ord-006", "carol@example.com", "+14155551111", 2, 99.99);

        // Act
        orderService.placeOrder(order);

        // Assert: verify ordering — save before notify
        InOrder inOrder = inOrder(mockRepository, mockNotificationService);
        inOrder.verify(mockRepository).save(order);
        inOrder.verify(mockNotificationService).notifyCustomer(any(), anyString(), anyString());
    }
}
```

Mockito can generate a mock from any interface because DIP ensures `OrderService` depends on an interface. If `OrderService` depended on `EmailNotificationService` directly, Mockito could still mock the concrete class, but you would be mocking final methods or internal state — a brittle test that tests implementation details rather than behavior.

---

## 10. DI Containers: Spring and Guice

A DI container is a framework that automates the wiring of dependencies. You declare what each class needs (usually via constructor parameters, annotations, or configuration), and the container constructs everything in the correct order, handling the dependency graph automatically.

### Spring Framework

Spring's container is called the ApplicationContext. In modern Spring, you express dependencies with `@Autowired` on constructors and register implementations with stereotype annotations (`@Service`, `@Repository`, `@Component`).

```java
// Spring: high-level service
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final NotificationService notificationService;

    // Spring detects this constructor and injects the registered implementations
    // No @Autowired needed on single-constructor classes since Spring 4.3
    public OrderService(OrderRepository orderRepository,
                        NotificationService notificationService) {
        this.orderRepository     = orderRepository;
        this.notificationService = notificationService;
    }

    public void placeOrder(Order order) {
        order.setStatus(OrderStatus.CONFIRMED);
        orderRepository.save(order);
        notificationService.notifyCustomer(order, "Confirmed: " + order.getId(), "Your order is confirmed.");
    }
}
```

```java
// Spring: infrastructure implementation registered as a bean
import org.springframework.stereotype.Component;

@Component
public class EmailNotificationService implements NotificationService {

    @Override
    public void notifyCustomer(Order order, String subject, String message) {
        System.out.println("[EMAIL] " + order.getCustomerEmail() + " | " + subject);
    }
}
```

```java
// Spring: configuration for selecting between multiple implementations
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.List;

@Configuration
public class NotificationConfiguration {

    // When multiple NotificationService beans exist, use @Bean to define
    // the composite explicitly, injecting all individual implementations
    @Bean
    public NotificationService compositeNotificationService(
            List<NotificationService> allImplementations) {
        return new CompositeNotificationService(allImplementations);
    }
}
```

Spring resolves `List<NotificationService>` to all beans that implement the interface, making it trivial to add a new channel: add a new `@Component` class. The `OrderService` and the `@Configuration` class are never modified.

### Google Guice

Guice uses modules and `bind()` calls to define the wiring explicitly:

```java
import com.google.inject.*;

public class NotificationModule extends AbstractModule {

    @Override
    protected void configure() {
        // Tell Guice: when someone asks for NotificationService, give them CompositeNotificationService
        bind(NotificationService.class).to(CompositeNotificationService.class);
        bind(UserRepository.class).to(MySQLUserRepository.class);
        bind(OrderRepository.class).to(JdbcOrderRepository.class);
    }
}

// Entry point
public class Application {
    public static void main(String[] args) {
        Injector injector = Guice.createInjector(new NotificationModule());
        OrderService orderService = injector.getInstance(OrderService.class);
        // orderService is fully wired with all dependencies
    }
}
```

Both frameworks implement DIP by convention: they inject via constructors (Spring prefers this) and they operate on interfaces, not concrete classes. The frameworks do not enforce DIP — you can still write `@Autowired EmailNotificationService` — but they are designed around the assumption that you follow DIP.

**Key point:** A DI container is a tool for automating wiring. DIP is the principle that makes the wiring meaningful. Without DIP, a DI container gives you indirection without real decoupling.

---

## 11. Ports and Adapters / Hexagonal Architecture

Hexagonal Architecture (also called Ports and Adapters), coined by Alistair Cockburn, is an architectural style in which DIP is the foundational rule applied at the system boundary level, not just within a single class.

### The Core Idea

The domain (your business logic) sits at the center. It communicates with the outside world exclusively through ports — interfaces defined by and owned by the domain. Adapters are the implementations of those ports. Adapters live at the perimeter of the system and translate between the domain's language and the language of external systems (databases, HTTP, message queues, file systems, external APIs).

```
           External Systems / Drivers
          (HTTP clients, test runners, UIs)
                        |
                        v
              +---------+---------+
              |   Driving Ports   |  <- interfaces called BY the domain
              |  (Primary Ports)  |
              |                   |
              |   D O M A I N     |
              |   (business       |
              |    logic)         |
              |                   |
              |   Driven Ports    |  <- interfaces called BY the domain
              |  (Secondary Ports)|     (implemented by infrastructure)
              +---------+---------+
                        |
                        v
             Infrastructure Adapters
        (JDBC, SMTP, REST clients, file I/O)
```

### Ports Versus Adapters in Code

**Port (interface, owned by domain):**

```java
// Package: com.company.orders.port.out (domain layer)
// This is a "driven port" — the domain drives external persistence through it.
// The domain defines this interface; the database adapter implements it.
public interface OrderRepository {
    void save(Order order);
    Order findById(String orderId);
    List<Order> findByCustomerEmail(String email);
}
```

```java
// Package: com.company.orders.port.out (domain layer)
// Another driven port — the domain drives notification through it.
public interface NotificationService {
    void notifyCustomer(Order order, String subject, String message);
}
```

**Adapter (implementation, in infrastructure layer):**

```java
// Package: com.company.infrastructure.persistence (infrastructure layer)
// This adapter implements the domain's port using PostgreSQL.
// The domain does not know PostgreSQL exists.
import org.springframework.jdbc.core.JdbcTemplate;

public class PostgreSQLOrderRepository implements OrderRepository {

    private final JdbcTemplate jdbcTemplate;

    public PostgreSQLOrderRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public void save(Order order) {
        jdbcTemplate.update(
            "INSERT INTO orders (id, customer_email, status) VALUES (?, ?, ?)",
            order.getId(), order.getCustomerEmail(), order.getStatus().name()
        );
    }

    @Override
    public Order findById(String orderId) {
        return jdbcTemplate.queryForObject(
            "SELECT * FROM orders WHERE id = ?",
            (rs, rowNum) -> new Order(
                rs.getString("id"),
                rs.getString("customer_email"),
                OrderStatus.valueOf(rs.getString("status"))
            ),
            orderId
        );
    }

    @Override
    public List<Order> findByCustomerEmail(String email) {
        return jdbcTemplate.query(
            "SELECT * FROM orders WHERE customer_email = ?",
            (rs, rowNum) -> new Order(
                rs.getString("id"),
                rs.getString("customer_email"),
                OrderStatus.valueOf(rs.getString("status"))
            ),
            email
        );
    }
}
```

### The Dependency Rule

In Hexagonal Architecture, all compile-time dependencies point inward — toward the domain. The domain never imports anything from the infrastructure. The infrastructure imports the domain's ports to implement them. This is DIP applied architecturally: the ports (abstractions) are owned by the domain (high-level), and the adapters (details) depend on the ports.

The connection to DIP is exact:
- Ports = abstractions (Rule A and Rule B of DIP)
- Adapters = details (Rule B: details depend on abstractions)
- Domain = high-level module (Rule A: high-level does not depend on low-level)
- Infrastructure = low-level module (Rule A: low-level depends on abstractions)

When you apply DIP correctly at the class level, you are building the foundation for Hexagonal Architecture at the system level. The principle scales.

---

## 12. The Common Mistake: Interface Mirroring

The most widespread DIP anti-pattern is creating an interface for every concrete class with a matching name and identical method signatures, solely to satisfy the letter of the principle without its spirit.

### Anti-Pattern: Interface Mirroring

```java
// Anti-pattern: IEmailNotifier is just EmailNotifier with an 'I' prefix
// This provides indirection but not inversion
public interface IEmailNotifier {
    void sendEmailNotification(String recipientEmail, String subject, String body);
}

public class EmailNotifier implements IEmailNotifier {
    @Override
    public void sendEmailNotification(String recipientEmail, String subject, String body) {
        // implementation
    }
}

// OrderService still depends on email-specific concepts
public class OrderService {

    private final IEmailNotifier emailNotifier; // still email-specific
    private final ISMSNotifier smsNotifier;     // still SMS-specific

    public OrderService(IEmailNotifier emailNotifier, ISMSNotifier smsNotifier) {
        this.emailNotifier = emailNotifier;
        this.smsNotifier   = smsNotifier;
    }

    public void placeOrder(Order order) {
        emailNotifier.sendEmailNotification(order.getCustomerEmail(), "Confirmed", "...");
        smsNotifier.sendSMSNotification(order.getCustomerPhone(), "Confirmed: ...");
    }
}
```

**Why this is wrong:** The interface is defined in terms of the detail (email, SMS), not in terms of what the high-level module needs (customer notification). If you add push notification, you still add `IPushNotifier` and edit `OrderService`. Adding the 'I' prefix changed nothing architecturally. The dependency graph is identical to the original problem.

**How to tell the difference:** A real DIP abstraction is defined by the caller's vocabulary. `NotificationService.notifyCustomer(Order, String, String)` is the caller's vocabulary — the caller knows orders and customers. `IEmailNotifier.sendEmailNotification(String, String, String)` is the implementor's vocabulary — it knows about email, recipients, SMTP subjects.

The interface should describe what the caller needs, expressed in the caller's domain language. If the interface reads like a facade over a specific technology, it is mirroring, not abstraction.

### Additional Interface Mirroring Examples

| Wrong (mirrors concrete) | Right (caller's vocabulary) |
|---|---|
| `IEmailSender.sendEmail(to, subject, body)` | `NotificationService.notifyCustomer(order, subject, message)` |
| `IMySQLRepository.executeQuery(sql, params)` | `UserRepository.findByEmail(email)` |
| `IS3Uploader.putObject(bucket, key, data)` | `FileStorage.store(fileId, content)` |
| `IPDFRenderer.renderToPDF(html, options)` | `ReportExporter.export(reportData, destination)` |

The right-hand column names are stable as long as the business domain is stable. The left-hand column names are coupled to specific technologies that may change.

---

## 13. Interview Questions with Model Answers

---

**Q1. What is the Dependency Inversion Principle, and why is it called "inversion"?**

DIP has two rules. First, high-level modules must not depend on low-level modules — both must depend on abstractions. Second, abstractions must not depend on details — details must depend on abstractions.

It is called "inversion" because without the principle, the natural direction of a dependency is from the module that uses something toward the module that implements it — the ordering module depends on the email module. When you apply DIP, that arrow is inverted: the email module depends on an interface defined in the ordering module's package, and the ordering module depends on that same interface. The module that was previously the dependent is now the one being depended upon (by the infrastructure). The direction of the source-code dependency arrow has flipped.

---

**Q2. What is the difference between DIP and Dependency Injection?**

DIP is a design principle about the structure of source-code dependencies. It mandates that high-level modules depend on abstractions rather than on concrete low-level modules. It says nothing about how those dependencies are instantiated or delivered.

Dependency Injection is a technique for providing dependencies to an object from outside the object rather than having the object construct them internally. Constructor injection, setter injection, and method injection are three forms.

DIP tells you what the dependency graph must look like. DI is one mechanism for achieving that structure. You can satisfy DIP with manual wiring (passing interfaces through constructors in `main`) without any framework. You can also use DI (constructor injection) while still violating DIP, if the injected type is a concrete class rather than an abstraction.

---

**Q3. A colleague says "I use Spring `@Autowired`, so I'm using DIP." Is this correct?**

Not necessarily. Spring provides DI, which is a technique. Whether DIP is satisfied depends on what type is being injected. If the `@Autowired` field is declared as `NotificationService` (interface), DIP is satisfied. If it is declared as `EmailNotificationService` (concrete class), DIP is violated even though DI is being used. The principle is about the declared type in the high-level module, not about the mechanism that delivers the instance.

---

**Q4. Where should the interface live — in the package of the caller or the implementor? Why does it matter?**

The interface must live in the package of the caller — the high-level module. This is what makes the dependency inversion real. When the interface lives with the caller, the build-graph dependency runs from the infrastructure module toward the business module (the infrastructure must import the business module to implement its interface). If the interface lives with the implementors, the business module still has a compile dependency on the infrastructure module to import the interface, and the dependency graph has not been inverted.

In practical terms: `NotificationService` belongs in `com.company.orders`, not in `com.company.notifications`. `EmailNotificationService` in `com.company.notifications` imports `com.company.orders.NotificationService` to implement it.

---

**Q5. How does DIP relate to unit testability? Can you give a concrete example?**

DIP creates seams at the module boundary. A seam is a place in code where you can substitute one behavior for another without editing the class under test. When `OrderService` depends on `NotificationService` (interface), a unit test can pass a `SpyNotificationService` or a Mockito mock that does nothing. The test exercises `OrderService`'s logic — status setting, order saving, calling the notification — without sending a real email or SMS. The test is fast, deterministic, and runs offline.

Without DIP, `OrderService` creates `EmailNotifier` with `new`. There is no seam. The test cannot intercept the notification call. The test must connect to a real SMTP server or accept that the notification code path is untested. DIP is a prerequisite for unit testing business logic in isolation.

---

**Q6. What is the "interface mirroring" anti-pattern? How do you recognize it?**

Interface mirroring is the pattern of creating an interface whose name and method signatures mirror a single concrete class, typically by prepending 'I' to the class name or declaring methods that expose implementation-specific vocabulary. Example: `IEmailNotifier.sendEmailNotification(recipientEmail, subject, body)` mirrors `EmailNotifier` exactly.

You recognize it when: the interface has only one non-test implementation; the method names contain technology-specific terms (email, SQL, PDF, S3); the interface is in the same package as its single implementation; and adding a new "channel" (a new implementation) still requires editing the caller because each channel has a different interface.

Real DIP abstractions are defined in the caller's domain vocabulary, not the implementor's technology vocabulary. They describe what the caller needs, not how the implementor works.

---

**Q7. How does DIP relate to Hexagonal Architecture (Ports and Adapters)?**

Hexagonal Architecture is DIP applied at the architectural level. The domain is the high-level module. Ports are the abstractions (interfaces) it defines and owns. Adapters are the low-level modules that implement those ports. The dependency rule is identical: all compile-time dependencies point inward toward the domain. Infrastructure depends on the domain to implement ports; the domain never imports infrastructure.

DIP at the class level and Hexagonal Architecture at the system level are the same principle operating at different scales. If you apply DIP correctly at the class level — interfaces owned by callers, infrastructure implementing domain-owned interfaces — you are building the foundation of a hexagonal architecture.

---

**Q8. Can you violate DIP even when using an interface?**

Yes. DIP can be violated in three ways even when an interface is present.

First, wrong ownership: the interface lives in the infrastructure package, so the business module still imports the infrastructure package to compile. The build-graph dependency is not inverted.

Second, interface mirroring: the interface is defined in the implementor's vocabulary (technology-specific method names), so the caller still couples to implementation concepts and must be edited when implementations are added or changed.

Third, internal instantiation: the high-level class accepts an interface as a field type but then instantiates a specific concrete class inside a method or constructor: `this.notifier = new EmailNotificationService()`. The interface annotation is present but ignored in practice. The structural dependency is still on the concrete class.

DIP requires that the interface be owned by the caller, defined in the caller's vocabulary, and that the concrete instance be provided from outside the high-level class.

---

**Q9. When would you choose not to apply DIP? Are there valid cases?**

DIP has a cost: it adds indirection, more types, and requires a composition root or DI container. For very small programs, utility classes, or clearly stable, one-implementation relationships, the overhead can exceed the benefit.

Valid cases where DIP may be overkill:
- Internal utility methods that will never be swapped (e.g., string formatting helpers)
- Small scripts or command-line tools with no testing requirement
- Value objects and entities that have no pluggable behavior

The signal that DIP is warranted is any of: the class crosses a process or network boundary (database, email, file system, external API), you need to test the caller in isolation, or the implementation is likely to change (adding new channels, migrating databases). In enterprise application code, the answer is almost always to apply DIP at every infrastructure boundary.

---

**Q10. Walk through designing a system where an `InvoiceService` needs to generate invoices and send them. How would you apply DIP?**

First, identify the high-level and low-level modules. `InvoiceService` is the high-level module — it encodes the business rule "when an invoice is created, generate its document and dispatch it." Generating the document and dispatching it are low-level mechanisms: PDF generation and email delivery are implementation details.

Second, define the abstractions in `InvoiceService`'s package:
- `InvoiceDocumentGenerator` with method `generate(Invoice invoice): byte[]`
- `InvoiceDispatcher` with method `dispatch(Invoice invoice, byte[] document)`

Third, write infrastructure implementations:
- `PDFInvoiceGenerator implements InvoiceDocumentGenerator` — uses iText or Apache PDFBox
- `XLSXInvoiceGenerator implements InvoiceDocumentGenerator` — exports to Excel
- `EmailInvoiceDispatcher implements InvoiceDispatcher` — attaches PDF and sends via SMTP
- `StorageInvoiceDispatcher implements InvoiceDispatcher` — saves to S3 or local filesystem

Fourth, wire in the composition root:
```java
InvoiceService service = new InvoiceService(
    new PDFInvoiceGenerator(),
    new EmailInvoiceDispatcher(smtpConfig)
);
```

`InvoiceService` never references PDF or SMTP. Adding an Excel export means writing `XLSXInvoiceGenerator` and rewiring the composition root. The business logic is unchanged.

---

## 14. Assignment

### Problem Statement

You have inherited the following `ReportGenerator` class. It directly instantiates and uses three concrete collaborators: `PDFExporter`, `ExcelExporter`, and `DatabaseLogger`. The class is neither extensible nor testable.

```java
// Given code — do not modify these concrete classes; refactor around them

// Low-level: exports a report to PDF
public class PDFExporter {

    private final String outputDirectory;

    public PDFExporter(String outputDirectory) {
        this.outputDirectory = outputDirectory;
    }

    public void exportToPDF(String reportTitle, String reportContent) {
        String filename = outputDirectory + "/" + reportTitle.replace(" ", "_") + ".pdf";
        System.out.println("[PDFExporter] Writing PDF to: " + filename);
        System.out.println("[PDFExporter] Content length: " + reportContent.length() + " chars");
        // Real implementation would use iText, Apache PDFBox, etc.
    }
}

// Low-level: exports a report to Excel
public class ExcelExporter {

    private final String outputDirectory;
    private final boolean includeCharts;

    public ExcelExporter(String outputDirectory, boolean includeCharts) {
        this.outputDirectory = outputDirectory;
        this.includeCharts   = includeCharts;
    }

    public void exportToExcel(String reportTitle, String[][] tableData) {
        String filename = outputDirectory + "/" + reportTitle.replace(" ", "_") + ".xlsx";
        System.out.println("[ExcelExporter] Writing Excel to: " + filename);
        System.out.println("[ExcelExporter] Rows: " + tableData.length
            + " | Charts: " + includeCharts);
        // Real implementation would use Apache POI
    }
}

// Low-level: logs report generation events to a database
public class DatabaseLogger {

    private final String jdbcUrl;

    public DatabaseLogger(String jdbcUrl) {
        this.jdbcUrl = jdbcUrl;
    }

    public void logReportGeneration(String reportTitle, String generatedBy, long durationMs) {
        System.out.println("[DatabaseLogger] Logging to " + jdbcUrl + ": report='"
            + reportTitle + "' user='" + generatedBy + "' durationMs=" + durationMs);
        // Real implementation would execute an INSERT
    }
}

// High-level module with DIP violations — this is the class to refactor
public class ReportGenerator {

    // DIP violations: concrete types as fields
    private final PDFExporter pdfExporter;
    private final ExcelExporter excelExporter;
    private final DatabaseLogger databaseLogger;

    public ReportGenerator() {
        // DIP violations: self-construction of dependencies
        this.pdfExporter    = new PDFExporter("/var/reports/pdf");
        this.excelExporter  = new ExcelExporter("/var/reports/excel", true);
        this.databaseLogger = new DatabaseLogger("jdbc:mysql://prod-db:3306/reports");
    }

    public void generateReport(String title, String content, String[][] tableData,
                               String requestedBy) {
        long start = System.currentTimeMillis();

        // Export in both formats
        pdfExporter.exportToPDF(title, content);
        excelExporter.exportToExcel(title, tableData);

        // Log the event
        long duration = System.currentTimeMillis() - start;
        databaseLogger.logReportGeneration(title, requestedBy, duration);
    }
}
```

---

### Your Tasks

**Task 1 — Identify every DIP violation.**

Write a comment block above the refactored code listing each violation by line reference: which field or line is wrong, and why it is a violation.

**Task 2 — Define the abstractions.**

Create two interfaces that `ReportGenerator` should depend on:
- `ReportExporter` — for exporting the report (PDF, Excel, or any future format)
- `ReportLogger` — for recording the generation event (database, file, Splunk, or any future destination)

Define the method signatures using `ReportGenerator`'s vocabulary, not the implementors' vocabulary. The method on `ReportExporter` should not mention "PDF" or "Excel." The method on `ReportLogger` should not mention "database" or "JDBC."

**Task 3 — Write adapter classes.**

Wrap each concrete class in an adapter that implements the appropriate interface:
- `PDFReportExporter implements ReportExporter` — delegates to `PDFExporter`
- `ExcelReportExporter implements ReportExporter` — delegates to `ExcelExporter`
- `DatabaseReportLogger implements ReportLogger` — delegates to `DatabaseLogger`

You may not modify the existing concrete classes. The adapters are the glue layer.

**Task 4 — Refactor `ReportGenerator`.**

Rewrite `ReportGenerator` so it:
- Accepts `List<ReportExporter>` and `ReportLogger` via constructor injection
- Has no references to `PDFExporter`, `ExcelExporter`, or `DatabaseLogger`
- Calls all exporters in the list, making it trivially extensible (adding a new format means writing a new adapter and updating wiring)

**Task 5 — Write a composition root.**

In a `main` method or factory class, wire everything together manually (no Spring). Demonstrate adding a hypothetical `CSVReportExporter` to the exporter list without modifying `ReportGenerator`.

**Task 6 — Write unit tests.**

Write at least three unit tests for the refactored `ReportGenerator`:
- Test that all exporters in the list are called
- Test that the logger is called with the correct report title and requester
- Test behavior when the exporter list is empty (should it throw, log, or silently succeed — define the contract and test it)

Use either hand-rolled test doubles or Mockito.

---

### Expected Outcome

After completing the assignment, `ReportGenerator` should:
- Compile without importing `PDFExporter`, `ExcelExporter`, or `DatabaseLogger`
- Be testable with zero file system or database access
- Support adding a new export format (`CSVReportExporter`, `HTMLReportExporter`) by writing a new class and updating the composition root — without touching `ReportGenerator`
- Support adding a new logging destination (Splunk, CloudWatch) by writing a new class and updating the composition root — without touching `ReportGenerator`

---

### Self-Check Rubric

| Criterion | Check |
|---|---|
| `ReportExporter` method signature uses caller vocabulary (no "PDF", "Excel") | |
| `ReportLogger` method signature uses caller vocabulary (no "database", "JDBC") | |
| Interfaces live in the same package as `ReportGenerator`, not with adapters | |
| `ReportGenerator` constructor accepts abstractions, not concrete types | |
| `ReportGenerator` has zero `new` keyword usage for its dependencies | |
| `ReportGenerator` imports zero infrastructure classes | |
| Composition root is the only file that imports all three concrete classes | |
| Unit tests run without file system or database access | |
| Adding `CSVReportExporter` requires zero changes to `ReportGenerator` | |
| Adapter classes translate method signatures without leaking technology terms | |

---

## Summary

The Dependency Inversion Principle is the structural foundation of testable, extensible, and maintainable software. Its core mandate is simple: the direction of source-code dependencies must be toward abstractions, and those abstractions must be owned by the modules that use them, not the modules that implement them.

The five key takeaways are:

1. High-level modules (policy, business logic) must never directly reference low-level modules (mechanism, infrastructure). The coupling that makes software rigid, untestable, and fragile is always rooted in this mistake.

2. The abstraction belongs to the caller. An interface that lives in the infrastructure package has not inverted anything. The build-graph dependency is still: business depends on infrastructure.

3. DIP and Dependency Injection are different things. DIP is a structural property of the code. DI is a delivery mechanism. One does not imply the other. A class can receive a concrete type via constructor injection (DI without DIP) and a class can depend on an abstraction it creates internally (DIP without DI).

4. Interface mirroring is not DIP. Creating `IEmailNotifier` over `EmailNotifier` adds a type without changing the dependency structure. The test for a real abstraction: does the method signature belong to the caller's vocabulary or the implementor's vocabulary?

5. DIP scales from the class level to the architecture level. Hexagonal Architecture is DIP applied at the system boundary. Every port is an interface owned by the domain. Every adapter is an infrastructure class that implements a domain-owned interface. The principle is the same — only the scale differs.

A software engineer who can identify DIP violations, articulate why they are harmful, design the corrected abstraction ownership, and demonstrate the testability benefit in code is demonstrating the judgment expected at the Senior Engineer level. The principle is not a rule to follow — it is an insight about what makes software structures durable.

---

*Next: Module 02 — Design Principles (DRY, YAGNI, KISS, Coupling & Cohesion)*
