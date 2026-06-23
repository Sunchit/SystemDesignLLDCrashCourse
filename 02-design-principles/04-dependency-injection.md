# Dependency Injection

> "Don't call us, we'll call you."
> — The Hollywood Principle, the informal name for Inversion of Control

Dependency Injection is the single most important pattern for writing testable, maintainable Java applications. Every senior Java engineer must understand it thoroughly — not just how to use a DI framework, but why it exists, what problem it solves, and how to apply it without any framework at all.

---

## Table of Contents

1. [The Problem DI Solves](#1-the-problem-di-solves)
2. [DI Definition and the IoC Container Concept](#2-di-definition-and-the-ioc-container-concept)
3. [The Three Types of Injection](#3-the-three-types-of-injection)
4. [Constructor Injection: Why It Is Preferred](#4-constructor-injection-why-it-is-preferred)
5. [Manual DI (No Framework)](#5-manual-di-no-framework)
6. [DI with Spring](#6-di-with-spring)
7. [Service Locator Pattern: The Anti-Pattern Debate](#7-service-locator-pattern-the-anti-pattern-debate)
8. [DI and Testability](#8-di-and-testability)
9. [DI in Layered Architecture](#9-di-in-layered-architecture)
10. [Common DI Mistakes](#10-common-di-mistakes)
11. [Interview Questions](#11-interview-questions)

---

## 1. The Problem DI Solves

Consider this class:

```java
public class OrderService {
    private final OrderRepository repository;
    private final EmailService emailService;

    public OrderService() {
        // OrderService creates its own dependencies
        this.repository = new MySQLOrderRepository();
        this.emailService = new SmtpEmailService("smtp.company.com");
    }

    public Order placeOrder(OrderRequest request) {
        Order order = createOrder(request);
        Order saved = repository.save(order);
        emailService.sendConfirmation(saved);
        return saved;
    }
}
```

Problems:

**Problem 1: Untestability.** To test `placeOrder()`, you need a real MySQL database and a real SMTP server. Tests are slow, flaky, and require infrastructure.

**Problem 2: Rigidity.** To use PostgreSQL instead of MySQL, you must modify `OrderService`. It violates the Open/Closed Principle.

**Problem 3: Hidden dependencies.** The dependencies of `OrderService` are invisible from the outside. You learn what it needs only by reading its constructor.

**Problem 4: Violation of Single Responsibility.** `OrderService` is responsible for *creating* its dependencies AND *using* them. These are two different responsibilities.

**Root cause:** `OrderService` controls the *construction* of its own dependencies. Dependency Injection inverts this control.

---

## 2. DI Definition and the IoC Container Concept

**Dependency Injection** is a pattern where an object receives its dependencies from an external source rather than creating them itself.

**Inversion of Control (IoC)** is the broader principle: instead of a class pulling in what it needs (traditional control flow), the framework/container pushes dependencies in (inverted control flow).

**IoC Container** is a framework that manages object creation and wiring. It knows how to create objects, what their dependencies are, and how to inject them. Spring is the most widely used IoC container in Java.

```
Traditional flow:          IoC flow:

OrderService               Container
  creates                    creates OrderService
  MySQLOrderRepository  -->   creates MySQLOrderRepository
  creates                    creates SmtpEmailService
  SmtpEmailService           injects MySQLOrderRepository --> OrderService
                             injects SmtpEmailService    --> OrderService
```

The container owns the wiring. `OrderService` only declares what it needs.

---

## 3. The Three Types of Injection

### Constructor Injection

Dependencies are passed through the constructor.

```java
public class OrderService {
    private final OrderRepository repository;
    private final EmailService emailService;

    public OrderService(OrderRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }
}
```

### Setter Injection

Dependencies are passed through setter methods after construction.

```java
public class OrderService {
    private OrderRepository repository;
    private EmailService emailService;

    public void setRepository(OrderRepository repository) {
        this.repository = repository;
    }

    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }
}
```

### Interface Injection

The class implements an injection interface that the container calls to provide the dependency. Rarely used in modern Java.

```java
public interface RepositoryInjector {
    void injectRepository(OrderRepository repository);
}

public class OrderService implements RepositoryInjector {
    private OrderRepository repository;

    @Override
    public void injectRepository(OrderRepository repository) {
        this.repository = repository;
    }
}
```

---

## 4. Constructor Injection: Why It Is Preferred

Constructor injection is the recommended default in Spring, and the preferred type in the Java community. Here is why:

### 1. Immutability

Constructor injection allows fields to be `final`. The dependency cannot be changed after construction — the object's state is consistent for its entire lifetime.

```java
// Constructor injection — field can be final
public class OrderService {
    private final OrderRepository repository; // final — immutable reference

    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}

// Setter injection — field cannot be final
public class OrderService {
    private OrderRepository repository; // mutable — can be swapped or nulled

    public void setRepository(OrderRepository repository) {
        this.repository = repository;
    }
}
```

### 2. Mandatory Dependencies Are Enforced

If a dependency is required, it must be provided at construction time. You cannot create an `OrderService` without an `OrderRepository` — the compiler enforces this. With setter injection, you can create an object and forget to call a setter, getting a `NullPointerException` at runtime.

```java
// With constructor injection:
OrderService service = new OrderService(); // COMPILE ERROR — repository required

// With setter injection:
OrderService service = new OrderService(); // Compiles fine
service.processOrder(request);            // NullPointerException — forgot to inject
```

### 3. Clear Dependency Contract

The constructor signature lists all dependencies explicitly. Anyone reading the class knows immediately what it needs. With field injection (a Spring anti-pattern using `@Autowired` directly on fields), dependencies are invisible from the outside.

```java
// Constructor injection — dependencies visible in constructor signature
public OrderService(OrderRepository repository,
                    EmailService emailService,
                    PaymentGateway paymentGateway) { ... }

// Field injection — dependencies hidden inside the class
public class OrderService {
    @Autowired private OrderRepository repository;        // invisible from outside
    @Autowired private EmailService emailService;         // invisible from outside
    @Autowired private PaymentGateway paymentGateway;    // invisible from outside
}
```

### 4. Reveals Design Problems

A constructor with 7 or 8 parameters is a code smell. It tells you the class has too many dependencies and probably too many responsibilities. This signal is invisible when using field injection — you can have 15 `@Autowired` fields and the class still looks clean from the outside.

### 5. No Framework Required for Testing

With constructor injection, testing requires no framework at all:

```java
OrderService service = new OrderService(
    new InMemoryOrderRepository(),
    new FakeEmailService()
);
```

With field injection, you need a Spring context or a reflection-based injection mechanism just to set up your test.

### When Setter Injection Is Acceptable

Setter injection is appropriate for **optional** dependencies — where the class can function with a default if the dependency is not provided.

```java
public class ReportService {
    private ReportFormatter formatter = new DefaultFormatter(); // default

    // Optional: caller can override the formatter
    public void setFormatter(ReportFormatter formatter) {
        this.formatter = Objects.requireNonNull(formatter);
    }
}
```

---

## 5. Manual DI (No Framework)

You do not need Spring to use DI. Manual DI — wiring objects together at the application entry point — is often the right choice for smaller applications and is essential to understand before using frameworks.

### The Composition Root

The **composition root** is the single place in an application where all objects are wired together. It is the top of the dependency graph. In a Java application, this is typically `main()` or an `Application` class.

```java
// Interfaces — the contracts
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(long id);
}

public interface EmailService {
    void sendConfirmation(Order order);
}

public interface PaymentGateway {
    PaymentResult charge(String token, double amount);
}

// Concrete implementations
public class MySQLOrderRepository implements OrderRepository {
    private final DataSource dataSource;

    public MySQLOrderRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public Order save(Order order) {
        // JDBC implementation
        return order;
    }

    @Override
    public Optional<Order> findById(long id) {
        // JDBC implementation
        return Optional.empty();
    }
}

public class SmtpEmailService implements EmailService {
    private final String host;
    private final int port;

    public SmtpEmailService(String host, int port) {
        this.host = host;
        this.port = port;
    }

    @Override
    public void sendConfirmation(Order order) {
        // SMTP implementation
    }
}

public class StripePaymentGateway implements PaymentGateway {
    private final String apiKey;

    public StripePaymentGateway(String apiKey) {
        this.apiKey = apiKey;
    }

    @Override
    public PaymentResult charge(String token, double amount) {
        // Stripe implementation
        return PaymentResult.SUCCESS;
    }
}

// OrderService — depends only on interfaces
public class OrderService {
    private final OrderRepository repository;
    private final EmailService emailService;
    private final PaymentGateway paymentGateway;

    public OrderService(OrderRepository repository,
                        EmailService emailService,
                        PaymentGateway paymentGateway) {
        this.repository = Objects.requireNonNull(repository);
        this.emailService = Objects.requireNonNull(emailService);
        this.paymentGateway = Objects.requireNonNull(paymentGateway);
    }

    public Order placeOrder(OrderRequest request) {
        Order order = new Order(request);
        PaymentResult result = paymentGateway.charge(
            request.getPaymentToken(), order.getTotal());
        if (!result.isSuccessful()) {
            throw new PaymentFailedException("Payment failed: " + result.getMessage());
        }
        Order saved = repository.save(order);
        emailService.sendConfirmation(saved);
        return saved;
    }
}

// Composition Root — the ONLY place where wiring happens
public class Application {

    public static void main(String[] args) {
        // Build the dependency graph bottom-up
        DataSource dataSource = buildDataSource();
        String smtpHost = System.getenv("SMTP_HOST");
        int smtpPort = Integer.parseInt(System.getenv("SMTP_PORT"));
        String stripeKey = System.getenv("STRIPE_API_KEY");

        // Leaf dependencies first
        OrderRepository orderRepository = new MySQLOrderRepository(dataSource);
        EmailService emailService = new SmtpEmailService(smtpHost, smtpPort);
        PaymentGateway paymentGateway = new StripePaymentGateway(stripeKey);

        // Inject into higher-level service
        OrderService orderService = new OrderService(
            orderRepository, emailService, paymentGateway);

        // Start the application with the fully wired object graph
        startServer(orderService);
    }

    private static DataSource buildDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(System.getenv("DB_URL"));
        config.setUsername(System.getenv("DB_USER"));
        config.setPassword(System.getenv("DB_PASSWORD"));
        return new HikariDataSource(config);
    }
}
```

The composition root is the only place in the application that knows about concrete implementations. Everything else depends on interfaces.

---

## 6. DI with Spring

Spring's IoC container manages the dependency graph automatically based on configuration.

### Annotation-Based Configuration (Modern Spring)

```java
// Mark as a Spring-managed bean
@Repository
public class MySQLOrderRepository implements OrderRepository {
    private final JdbcTemplate jdbcTemplate;

    // Spring injects JdbcTemplate — constructor injection
    @Autowired // optional in modern Spring if single constructor
    public MySQLOrderRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public Order save(Order order) {
        jdbcTemplate.update(
            "INSERT INTO orders (customer_id, total) VALUES (?, ?)",
            order.getCustomerId(), order.getTotal());
        return order;
    }
}

@Service
public class SmtpEmailService implements EmailService {
    @Value("${smtp.host}")      // inject from application.properties
    private String host;

    @Value("${smtp.port}")
    private int port;

    @Override
    public void sendConfirmation(Order order) {
        // SMTP implementation using host and port
    }
}

@Service
public class OrderService {
    private final OrderRepository repository;
    private final EmailService emailService;
    private final PaymentGateway paymentGateway;

    // Spring injects these at startup
    public OrderService(OrderRepository repository,
                        EmailService emailService,
                        PaymentGateway paymentGateway) {
        this.repository = repository;
        this.emailService = emailService;
        this.paymentGateway = paymentGateway;
    }

    public Order placeOrder(OrderRequest request) {
        // ...
    }
}
```

### Java-Based Configuration

```java
@Configuration
public class AppConfiguration {

    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(System.getenv("DB_URL"));
        return new HikariDataSource(config);
    }

    @Bean
    public OrderRepository orderRepository(DataSource dataSource) {
        return new MySQLOrderRepository(dataSource);
    }

    @Bean
    public EmailService emailService(
            @Value("${smtp.host}") String host,
            @Value("${smtp.port}") int port) {
        return new SmtpEmailService(host, port);
    }

    @Bean
    public PaymentGateway paymentGateway(
            @Value("${stripe.api.key}") String apiKey) {
        return new StripePaymentGateway(apiKey);
    }

    @Bean
    public OrderService orderService(OrderRepository repository,
                                     EmailService emailService,
                                     PaymentGateway paymentGateway) {
        return new OrderService(repository, emailService, paymentGateway);
    }
}
```

### Spring Scopes

```java
@Component
@Scope("singleton")  // Default: one instance per container
public class OrderService { ... }

@Component
@Scope("prototype")  // New instance per injection
public class ShoppingCart { ... }

@Component
@Scope("request")    // New instance per HTTP request
public class RequestContext { ... }
```

### Conditional Beans

```java
// Use StripePaymentGateway in production, FakePaymentGateway in tests
@Bean
@Profile("production")
public PaymentGateway stripePaymentGateway() {
    return new StripePaymentGateway(System.getenv("STRIPE_KEY"));
}

@Bean
@Profile("test")
public PaymentGateway fakePaymentGateway() {
    return new FakePaymentGateway();
}
```

---

## 7. Service Locator Pattern: The Anti-Pattern Debate

The Service Locator is often presented as an alternative to DI. It involves a central registry that classes use to look up their dependencies.

```java
// Service Locator
public class ServiceLocator {
    private static final Map<Class<?>, Object> services = new HashMap<>();

    public static <T> void register(Class<T> type, T instance) {
        services.put(type, instance);
    }

    @SuppressWarnings("unchecked")
    public static <T> T locate(Class<T> type) {
        T service = (T) services.get(type);
        if (service == null) {
            throw new IllegalStateException("No service registered for: " + type);
        }
        return service;
    }
}

// Usage in a class
public class OrderService {

    public Order placeOrder(OrderRequest request) {
        // Classes pull their own dependencies
        OrderRepository repository = ServiceLocator.locate(OrderRepository.class);
        EmailService email = ServiceLocator.locate(EmailService.class);
        // ...
    }
}
```

### Why Service Locator Is Considered an Anti-Pattern

**1. Hidden dependencies.** `OrderService`'s constructor signature shows no dependencies. You must read the body of every method to discover what it needs. The dependency contract is invisible.

**2. Testing is harder.** To test `OrderService`, you must prime the `ServiceLocator` with the right test implementations before the test and clean up after. Tests become stateful and order-dependent.

```java
// Test setup with Service Locator — fragile
@BeforeEach
void setUp() {
    ServiceLocator.register(OrderRepository.class, new FakeOrderRepository());
    ServiceLocator.register(EmailService.class, new FakeEmailService());
    // Forget to clean up -> next test sees wrong implementations
}
```

**3. Global mutable state.** The Service Locator is a global map. Multiple tests running in parallel can interfere with each other.

**4. Runtime failures.** If a service is not registered, you get a runtime exception. With constructor injection, a missing dependency is a startup failure — you know immediately at launch, not when a particular code path executes.

### When Service Locator Is Acceptable

Despite its reputation as an anti-pattern, Service Locator has legitimate uses:

- In framework internals, where you must retrieve services without an injected reference (e.g., legacy plugin systems)
- As the backbone of an IoC container itself (the container uses a locator internally)
- When integrating with code you cannot modify that uses static method calls

Mark Seemann (author of *Dependency Injection in .NET*) argues that Service Locator is not inherently bad, but is a bad choice when the consuming code is application code you own — because you lose the dependency transparency that makes DI valuable.

---

## 8. DI and Testability

This is the primary reason DI is used in professional Java development.

### Without DI — Impossible to Unit Test

```java
public class UserRegistrationService {

    public void register(UserRequest request) {
        // Directly creates dependencies — no way to substitute in tests
        UserRepository repo = new MySQLUserRepository();
        EmailService email = new SmtpEmailService();
        AuditLogger audit = new DatabaseAuditLogger();

        User user = new User(request.getName(), request.getEmail());
        repo.save(user);
        email.sendWelcome(user);
        audit.log("User registered: " + user.getId());
    }
}

// "Test" — actually an integration test that requires MySQL, SMTP, and DB audit table
@Test
void testRegister() {
    UserRegistrationService service = new UserRegistrationService();
    service.register(new UserRequest("Alice", "alice@example.com"));
    // How do you verify email was sent? How do you verify audit was logged?
    // How do you test failure scenarios? How do you run this without infrastructure?
}
```

### With DI — Clean, Fast, Isolated Unit Tests

```java
public class UserRegistrationService {
    private final UserRepository repository;
    private final EmailService emailService;
    private final AuditLogger auditLogger;

    public UserRegistrationService(UserRepository repository,
                                   EmailService emailService,
                                   AuditLogger auditLogger) {
        this.repository = repository;
        this.emailService = emailService;
        this.auditLogger = auditLogger;
    }

    public User register(UserRequest request) {
        if (repository.existsByEmail(request.getEmail())) {
            throw new DuplicateUserException("Email already registered");
        }
        User user = new User(request.getName(), request.getEmail());
        User saved = repository.save(user);
        emailService.sendWelcome(saved);
        auditLogger.log("User registered: " + saved.getId());
        return saved;
    }
}

// Unit tests — no infrastructure, runs in milliseconds
class UserRegistrationServiceTest {

    private UserRepository mockRepo;
    private EmailService mockEmail;
    private AuditLogger mockAudit;
    private UserRegistrationService service;

    @BeforeEach
    void setUp() {
        mockRepo = mock(UserRepository.class);
        mockEmail = mock(EmailService.class);
        mockAudit = mock(AuditLogger.class);
        service = new UserRegistrationService(mockRepo, mockEmail, mockAudit);
    }

    @Test
    void register_shouldSaveUserAndSendWelcomeEmail() {
        // Arrange
        UserRequest request = new UserRequest("Alice", "alice@example.com");
        when(mockRepo.existsByEmail("alice@example.com")).thenReturn(false);
        when(mockRepo.save(any(User.class))).thenAnswer(inv -> inv.getArgument(0));

        // Act
        User result = service.register(request);

        // Assert
        assertNotNull(result);
        assertEquals("Alice", result.getName());
        verify(mockRepo).save(any(User.class));
        verify(mockEmail).sendWelcome(any(User.class));
        verify(mockAudit).log(contains("User registered"));
    }

    @Test
    void register_shouldThrowWhenEmailAlreadyExists() {
        when(mockRepo.existsByEmail("alice@example.com")).thenReturn(true);

        assertThrows(DuplicateUserException.class,
            () -> service.register(new UserRequest("Alice", "alice@example.com")));

        verify(mockRepo, never()).save(any());
        verify(mockEmail, never()).sendWelcome(any());
    }

    @Test
    void register_shouldLogSuccessfulRegistration() {
        when(mockRepo.existsByEmail(anyString())).thenReturn(false);
        when(mockRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));

        service.register(new UserRequest("Bob", "bob@example.com"));

        verify(mockAudit).log(argThat(msg -> msg.contains("User registered")));
    }
}
```

Tests run in milliseconds, verify behavior precisely, and need no infrastructure. This is only possible because of DI.

---

## 9. DI in Layered Architecture

In a typical layered architecture (Controller -> Service -> Repository), DI flows downward:

```
Presentation Layer (Controller)
        |
        | injects
        v
Application Layer (Service)
        |
        | injects
        v
Data Layer (Repository)
        |
        | injects
        v
Infrastructure (DataSource, JdbcTemplate)
```

```java
// Data layer
public interface ProductRepository {
    Optional<Product> findById(long id);
    List<Product> findAll();
    Product save(Product product);
}

@Repository
public class JpaProductRepository implements ProductRepository {
    private final EntityManager em;

    public JpaProductRepository(EntityManager em) {
        this.em = em;
    }

    @Override
    public Optional<Product> findById(long id) {
        return Optional.ofNullable(em.find(Product.class, id));
    }
    // ...
}

// Application layer
public interface ProductService {
    ProductDto getProduct(long id);
    List<ProductDto> getAllProducts();
    ProductDto createProduct(CreateProductRequest request);
}

@Service
public class ProductServiceImpl implements ProductService {
    private final ProductRepository repository;
    private final ProductMapper mapper;

    // Constructor injection — Spring auto-detects single constructor in Spring 4.3+
    public ProductServiceImpl(ProductRepository repository, ProductMapper mapper) {
        this.repository = repository;
        this.mapper = mapper;
    }

    @Override
    public ProductDto getProduct(long id) {
        Product product = repository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException("Product not found: " + id));
        return mapper.toDto(product);
    }
    // ...
}

// Presentation layer
@RestController
@RequestMapping("/api/products")
public class ProductController {
    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductDto> getProduct(@PathVariable long id) {
        return ResponseEntity.ok(productService.getProduct(id));
    }
    // ...
}
```

Each layer depends on an abstraction of the layer below. No layer creates its own dependencies. The composition root (Spring container) wires everything.

---

## 10. Common DI Mistakes

### Mistake 1: Field Injection in Application Code

```java
// Anti-pattern: field injection
@Service
public class OrderService {
    @Autowired
    private OrderRepository repository; // hidden dependency, cannot be final, reflection-based

    @Autowired
    private EmailService emailService;
}

// Correct: constructor injection
@Service
public class OrderService {
    private final OrderRepository repository;
    private final EmailService emailService;

    public OrderService(OrderRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }
}
```

### Mistake 2: Injecting the Container

```java
// Anti-pattern: injecting ApplicationContext
@Service
public class OrderService {
    @Autowired
    private ApplicationContext context; // Service Locator in disguise

    public void doSomething() {
        OrderRepository repo = context.getBean(OrderRepository.class); // locator pattern
    }
}
```

This defeats the purpose of DI. You have just re-introduced Service Locator.

### Mistake 3: Creating Dependencies Inside a DI-Managed Bean

```java
// Anti-pattern: creating own dependencies inside a Spring bean
@Service
public class ReportService {
    public Report generate() {
        ReportFormatter formatter = new PdfFormatter(); // not injected — untestable
        // ...
    }
}

// Correct: inject the formatter
@Service
public class ReportService {
    private final ReportFormatter formatter;

    public ReportService(ReportFormatter formatter) {
        this.formatter = formatter;
    }

    public Report generate() {
        return formatter.format(...);
    }
}
```

### Mistake 4: Circular Dependencies

```java
// Circular dependency — Spring will throw BeanCurrentlyInCreationException
@Service
public class ServiceA {
    public ServiceA(ServiceB b) { ... } // A needs B
}

@Service
public class ServiceB {
    public ServiceB(ServiceA a) { ... } // B needs A — circular!
}
```

The fix: extract the shared logic into a third class that neither depends on the other.

```java
@Service
public class SharedLogicService { /* common behavior */ }

@Service
public class ServiceA {
    public ServiceA(SharedLogicService shared) { ... }
}

@Service
public class ServiceB {
    public ServiceB(SharedLogicService shared) { ... }
}
```

### Mistake 5: Too Many Constructor Parameters (God Class)

```java
// 8 constructor parameters — this class has too many responsibilities
public class OrderService {
    public OrderService(
        OrderRepository orderRepo,
        CustomerRepository customerRepo,
        ProductRepository productRepo,
        InventoryService inventory,
        PaymentGateway payment,
        EmailService email,
        AuditLogger audit,
        PricingEngine pricing) { ... }
}
```

When constructor injection reveals this smell, do not switch to field injection — fix the underlying design by splitting the class.

---

## 11. Interview Questions

**Q1. What is Dependency Injection and what problem does it solve?**

DI is a pattern where a class receives its dependencies from an external source rather than creating them itself. It solves the problems of tight coupling (classes are coupled to concrete implementations), untestability (cannot substitute test doubles), and violation of SRP (classes responsible for both construction and use of their dependencies).

**Q2. What is Inversion of Control and how does it relate to DI?**

IoC is the broader principle: instead of application code pulling in what it needs (traditional flow), a container pushes in what is needed (inverted flow). DI is the mechanism that implements IoC. The IoC container creates objects, resolves their dependencies, and injects them — the application code simply declares what it needs.

**Q3. What are the three types of dependency injection and which is preferred?**

Constructor injection (dependencies passed through the constructor), setter injection (dependencies set via setter methods), and interface injection (class implements an injection interface). Constructor injection is preferred because it enables immutable fields, enforces mandatory dependencies at compile time, makes the dependency contract visible, and enables testing without a framework.

**Q4. Why is field injection in Spring considered an anti-pattern?**

Field injection (`@Autowired` on fields) hides dependencies — you must read the class body to know what it needs. Fields cannot be `final`, so immutability is lost. Testing requires reflection-based injection or a Spring context. A constructor with many parameters signals a design problem; many `@Autowired` fields hide the same problem.

**Q5. What is a composition root and why should there be only one?**

The composition root is the single place where all object dependencies are wired together — typically `main()` or a Spring `@Configuration` class. Having one composition root means concrete type names appear in exactly one place. All other code depends only on interfaces. This makes swapping implementations, profiling configurations, and testing easy because only the composition root changes.

**Q6. Compare Service Locator to Dependency Injection.**

Service Locator provides a global registry that classes query to find their dependencies. DI has dependencies pushed in from outside. The key difference is transparency: with DI, the dependency contract is visible at the class boundary (constructor signature). With Service Locator, dependencies are hidden inside method bodies. This makes Service Locator harder to test (requires priming the locator) and harder to reason about (must read implementation to discover dependencies).

**Q7. How does DI enable unit testing?**

With DI, you inject test doubles (mocks, stubs, fakes) in place of real dependencies. The class under test operates on the mock, not real infrastructure. Tests are isolated, deterministic, and run without databases or network services. Without DI, classes create their own dependencies and there is no seam to insert test doubles.

**Q8. What is a circular dependency and how do you resolve it?**

A circular dependency exists when A depends on B and B depends on A (possibly transitively). In Spring, this causes `BeanCurrentlyInCreationException`. The resolution is to extract the shared behavior into a third class C, where both A and B depend on C. Another approach is to introduce an event-based mechanism (A publishes an event, B listens) to break the direct dependency.

**Q9. In a layered architecture, how should DI be structured?**

Each layer depends on an abstraction of the layer below. Controllers depend on service interfaces. Services depend on repository interfaces. Repositories depend on data source abstractions. No layer depends on a concrete implementation of another layer. The composition root (Spring container) wires all concrete implementations to their interfaces. This ensures each layer is independently testable.

**Q10. What signals a class has too many dependencies?**

A constructor with 5+ parameters is a warning sign. 7+ parameters is a clear signal that the class has too many responsibilities. The remedy is to split the class — not to hide the problem by switching to field injection. The visibility that constructor injection provides is a feature: it forces you to confront design problems that field injection would mask.
