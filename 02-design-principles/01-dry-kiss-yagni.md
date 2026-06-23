# DRY, KISS, and YAGNI

> These three principles are the meta-principles of software design. They do not tell you *what* to build — they tell you *how to think* about building it.

---

## Table of Contents

1. [DRY — Don't Repeat Yourself](#1-dry--dont-repeat-yourself)
2. [KISS — Keep It Simple, Stupid](#2-kiss--keep-it-simple-stupid)
3. [YAGNI — You Aren't Gonna Need It](#3-yagni--you-arent-gonna-need-it)

---

# 1. DRY — Don't Repeat Yourself

## Definition

> "Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."
> — Andrew Hunt & David Thomas, *The Pragmatic Programmer*

DRY is not about code. It is about **knowledge**. When the same piece of knowledge exists in two places, those two places will inevitably diverge. That divergence is a bug waiting to happen.

The authoritative representation is called the **Single Source of Truth (SSOT)**.

---

## Types of Duplication

Not all duplication is code duplication. There are four types:

### 1. Code Duplication
The most visible kind. The same logic copy-pasted across files.

```java
// Order service
public double calculateOrderTotal(Order order) {
    double total = 0;
    for (OrderItem item : order.getItems()) {
        total += item.getPrice() * item.getQuantity();
    }
    return total * 1.18; // 18% GST
}

// Invoice service — same logic duplicated
public double calculateInvoiceTotal(Invoice invoice) {
    double total = 0;
    for (InvoiceItem item : invoice.getItems()) {
        total += item.getPrice() * item.getQuantity();
    }
    return total * 1.18; // 18% GST — if GST changes, you must update TWO places
}
```

### 2. Logic Duplication (Structural)
Same logic, different appearance. Harder to spot than copy-paste duplication.

```java
// In UserValidator
if (user.getAge() < 18) {
    throw new IllegalArgumentException("User must be 18 or older");
}

// In RegistrationService — same rule, different form
boolean isEligible = user.getDateOfBirth()
    .isBefore(LocalDate.now().minusYears(18));
```

Both encode the rule "user must be 18+", but in different ways. If the rule changes to 21, you must find both.

### 3. Data Duplication
Derived values stored alongside their source.

```java
// Bad: fullName is always derived from firstName + lastName
// Storing it creates a synchronization problem
class User {
    private String firstName;
    private String lastName;
    private String fullName; // redundant — a second source of truth
}
```

### 4. Documentation Duplication
Comments that restate the code. When the code changes, the comment becomes a lie.

```java
// Bad: comment duplicates the logic
// Returns the price with 18% GST applied
public double getPriceWithTax(double price) {
    return price * 1.18; // 18% GST
}

// If you change to 20%, the comment still says 18%
```

---

## DRY in Domain Models vs Infrastructure

DRY applies differently at different layers:

**Domain Models** — DRY is critical. Business rules must live in exactly one place. If the rule "a discount applies only to orders over $100" appears in the order service, the reporting service, and the notification service, you have three sources of truth for one business rule.

**Infrastructure** — Some repetition is acceptable. Boilerplate configuration, logging setup, and connection pool initialization may look similar across services but represent *different instances of the same pattern*, not the same knowledge.

The distinction: ask yourself "if this changes, does it need to change everywhere?" If yes, it should be unified. If each instance can evolve independently, duplication may be fine.

---

## When DRY is Over-Applied

DRY can become its own enemy. Premature abstraction — unifying things that only *look* similar but are conceptually different — creates coupling that is worse than duplication.

This anti-pattern is sometimes called **WET (Write Everything Twice)** tolerance, or more pointedly, the **AHA principle** (Avoid Hasty Abstractions).

**The Rule of Three:** Wait until you see the same knowledge in three places before abstracting it. Two occurrences might be coincidence. Three occurrences is a pattern.

```java
// Two methods that look the same today:
public void sendWelcomeEmail(User user) {
    String subject = "Welcome to the platform";
    String body = "Hello " + user.getName() + ", welcome!";
    emailService.send(user.getEmail(), subject, body);
}

public void sendPasswordResetEmail(User user) {
    String subject = "Reset your password";
    String body = "Hello " + user.getName() + ", click here to reset.";
    emailService.send(user.getEmail(), subject, body);
}

// Premature DRY — unifying these creates a fragile abstraction:
public void sendEmail(User user, String subject, String bodyTemplate) {
    String body = bodyTemplate.replace("{name}", user.getName());
    emailService.send(user.getEmail(), subject, body);
}
// Now every new email type must fit this template — what if welcome email
// needs HTML, password reset needs expiry tokens, etc.?
```

The cost of wrong abstraction is higher than the cost of duplication. Duplication is visible and easy to fix. Wrong abstraction hides the problem inside a seemingly clean API.

---

## Before/After: Applying DRY

### Before (Violation)

```java
public class OrderService {

    public void placeOrder(Order order) {
        // Validate order
        if (order.getItems().isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one item");
        }
        if (order.getCustomer() == null) {
            throw new IllegalArgumentException("Order must have a customer");
        }

        // Calculate total
        double total = 0;
        for (OrderItem item : order.getItems()) {
            total += item.getPrice() * item.getQuantity();
        }
        double totalWithTax = total * 1.18;

        order.setTotal(totalWithTax);
        orderRepository.save(order);
    }

    public OrderSummary previewOrder(Order order) {
        // Validate order — duplicated
        if (order.getItems().isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one item");
        }
        if (order.getCustomer() == null) {
            throw new IllegalArgumentException("Order must have a customer");
        }

        // Calculate total — duplicated
        double total = 0;
        for (OrderItem item : order.getItems()) {
            total += item.getPrice() * item.getQuantity();
        }
        double totalWithTax = total * 1.18;

        return new OrderSummary(order, totalWithTax);
    }
}
```

### After (DRY Applied)

```java
public class OrderService {

    private static final double GST_RATE = 1.18; // Single source of truth for tax rate

    public void placeOrder(Order order) {
        validateOrder(order);
        double totalWithTax = calculateTotal(order);
        order.setTotal(totalWithTax);
        orderRepository.save(order);
    }

    public OrderSummary previewOrder(Order order) {
        validateOrder(order);
        double totalWithTax = calculateTotal(order);
        return new OrderSummary(order, totalWithTax);
    }

    // Single source of truth for validation rules
    private void validateOrder(Order order) {
        if (order.getItems().isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one item");
        }
        if (order.getCustomer() == null) {
            throw new IllegalArgumentException("Order must have a customer");
        }
    }

    // Single source of truth for price calculation
    private double calculateTotal(Order order) {
        double subtotal = order.getItems().stream()
            .mapToDouble(item -> item.getPrice() * item.getQuantity())
            .sum();
        return subtotal * GST_RATE;
    }
}
```

Now if GST changes to 20%, you change one constant. If validation rules change, you change one method.

---

## DRY Interview Questions

**Q1. What is the DRY principle and what does "knowledge" mean in this context?**

DRY stands for "Don't Repeat Yourself" and states that every piece of knowledge should have a single, authoritative representation. "Knowledge" means business rules, logic, and data — not just code characters. Two methods can look different yet duplicate the same business rule, which is the more dangerous violation.

**Q2. How is DRY different from "no copy-paste coding"?**

DRY is broader. Copy-paste produces code duplication, but DRY also covers logic duplication (same rule expressed differently), data duplication (derived values stored redundantly), and documentation duplication (comments that restate code). You can violate DRY without ever copying code.

**Q3. Can you over-apply DRY? Give an example.**

Yes. Premature abstraction — unifying things that only look similar — creates unwanted coupling. If two methods share the same structure today but for different business reasons, abstracting them ties their evolution together. The Rule of Three helps: wait for three occurrences before abstracting.

**Q4. What is the Single Source of Truth and why does it matter?**

SSOT means there is exactly one place where a given piece of knowledge is defined. When a business rule changes, you change it in one place and the entire system is correct. Without SSOT, a rule change requires finding and updating every duplicate — and missing one creates a bug.

**Q5. How does DRY interact with the Single Responsibility Principle?**

They complement each other. SRP says a class should have one reason to change. DRY says each piece of knowledge should live in one place. Together, they mean each piece of knowledge should live in the class responsible for it. Violating DRY often means a concept is spread across classes that shouldn't own it — which also violates SRP.

---

# 2. KISS — Keep It Simple, Stupid

## Definition

> "Everything should be made as simple as possible, but not simpler."
> — Often attributed to Albert Einstein

In software: prefer the simplest solution that correctly solves the problem. Complexity that is not demanded by the problem is a liability.

The principle was coined in the US Navy in the 1960s and entered software engineering through Kent Beck and others in the XP movement. The "Stupid" is directed at the engineer who builds complex solutions — not the user.

---

## Simplicity as a Design Virtue

Why does simplicity matter?

**Cognitive load.** Every developer who reads your code must load its logic into working memory. Complex code exhausts that budget quickly. Simple code leaves cognitive budget for understanding the problem, not the implementation.

**Maintenance cost.** Code is read far more often than it is written. Complexity multiplies every future maintenance task.

**Bug surface.** Every line of code is a potential bug. Complex code has more lines, more branches, more interactions — and therefore more places where bugs can hide.

**Testing cost.** Complex methods require many test cases to cover their branches. Simple methods are easy to test completely.

---

## Complexity Budgets

Think of every system as having a **complexity budget**: the total amount of complexity humans can understand and manage. There is an irreducible minimum of complexity demanded by the problem — called **essential complexity**. Everything beyond that is **accidental complexity** — complexity introduced by the solution rather than the problem.

Good engineering minimizes accidental complexity while managing essential complexity well.

Examples of accidental complexity:
- Using a framework for a problem that a few functions would solve
- Adding layers of abstraction "for flexibility" that never gets used
- Over-engineering a data structure for a dataset of 50 records
- Using concurrent programming where sequential code is sufficient

---

## Simple vs Simplistic: The Critical Distinction

Simple and simplistic are not the same.

**Simple:** A solution that handles the problem correctly with minimal moving parts. Simple solutions may take more thought upfront — reducing a complex problem to a simple solution is hard work.

**Simplistic:** A solution that handles only the easy cases and ignores edge cases, errors, or correctness. Simplistic solutions are easy to write but create problems downstream.

```java
// Simplistic: ignores null, ignores division by zero, ignores negative inputs
public double average(List<Integer> numbers) {
    int sum = 0;
    for (int n : numbers) sum += n;
    return sum / numbers.size();
}

// Simple: handles edge cases, still readable
public OptionalDouble average(List<Integer> numbers) {
    if (numbers == null || numbers.isEmpty()) {
        return OptionalDouble.empty();
    }
    return numbers.stream()
        .mapToInt(Integer::intValue)
        .average();
}
```

The simple solution is slightly longer but correct. The simplistic solution is shorter but wrong.

---

## Java Examples: Over-Engineered vs Simple

### Example 1: String Formatting

**Over-engineered:**
```java
public interface Formatter<T> {
    String format(T value);
}

public abstract class AbstractFormatter<T> implements Formatter<T> {
    protected abstract String doFormat(T value);

    @Override
    public String format(T value) {
        if (value == null) return "";
        return doFormat(value);
    }
}

public class GreetingFormatterFactory {
    private static final Map<String, Formatter<String>> formatters = new HashMap<>();

    static {
        formatters.put("formal", new AbstractFormatter<String>() {
            @Override
            protected String doFormat(String name) {
                return "Good day, " + name + ".";
            }
        });
        formatters.put("casual", new AbstractFormatter<String>() {
            @Override
            protected String doFormat(String name) {
                return "Hey " + name + "!";
            }
        });
    }

    public Formatter<String> getFormatter(String type) {
        return formatters.getOrDefault(type, formatters.get("formal"));
    }
}

// Usage
String greeting = new GreetingFormatterFactory()
    .getFormatter("casual")
    .format("Alice");
```

**Simple (and sufficient for the problem):**
```java
public String greet(String name, boolean formal) {
    if (name == null || name.isEmpty()) return "";
    return formal ? "Good day, " + name + "." : "Hey " + name + "!";
}
```

The over-engineered version introduces a factory, a strategy interface, an abstract class, and a registry for a problem that is a two-line conditional. It will impress nobody and confuse everyone.

---

### Example 2: Finding the Maximum Value

**Over-engineered:**
```java
public class MaxFinder<T extends Comparable<T>> {
    private final Comparator<T> comparator;

    public MaxFinder() {
        this.comparator = Comparator.naturalOrder();
    }

    public MaxFinder(Comparator<T> comparator) {
        this.comparator = comparator;
    }

    public Optional<T> findMax(Collection<T> collection) {
        return collection.stream()
            .reduce((a, b) -> comparator.compare(a, b) >= 0 ? a : b);
    }

    public T findMaxOrDefault(Collection<T> collection, T defaultValue) {
        return findMax(collection).orElse(defaultValue);
    }
}
```

**Simple:**
```java
public Optional<Integer> findMax(List<Integer> numbers) {
    return numbers.stream().max(Integer::compareTo);
}
```

Unless you actually need generic type support and custom comparators, the generic version is accidental complexity.

---

### Example 3: Checking if a Number is Even

**Over-engineered:**
```java
@FunctionalInterface
public interface NumberPredicate {
    boolean test(int number);
}

public class NumberPredicates {
    public static final NumberPredicate IS_EVEN = n -> n % 2 == 0;
    public static final NumberPredicate IS_ODD = n -> n % 2 != 0;

    private NumberPredicates() {}
}

// Usage
boolean result = NumberPredicates.IS_EVEN.test(4);
```

**Simple:**
```java
boolean isEven = number % 2 == 0;
```

---

## KISS and Performance

A common KISS violation is **premature optimization** — adding complexity to make code faster before profiling has identified a bottleneck.

> "Premature optimization is the root of all evil." — Donald Knuth

Write the simple, correct solution first. Profile. Optimize only the parts the profiler identifies as bottlenecks, and add complexity only where the measurement justifies it.

---

## KISS Interview Questions

**Q1. What does KISS mean and why is simplicity a virtue in software design?**

KISS means Keep It Simple, Stupid — prefer the simplest solution that correctly solves the problem. Simplicity is a virtue because code is read far more often than written. Complex code has higher cognitive load, more bug surface, higher maintenance cost, and is harder to test. The goal is to minimize accidental complexity (complexity introduced by the solution) while managing essential complexity (demanded by the problem).

**Q2. What is the difference between simple and simplistic?**

Simple means the solution handles the problem correctly with minimal moving parts. Simplistic means the solution handles only easy cases and ignores edge cases, error conditions, or correctness constraints. A simplistic solution is shorter but wrong. A simple solution is correct and still readable.

**Q3. How does KISS relate to premature optimization?**

Premature optimization violates KISS by adding complexity (caching layers, bit manipulation, custom data structures) before a performance problem has been demonstrated. This trades the clarity of simple code for speculative performance gains. KISS says: write the clear solution first, profile it, and optimize only what measurements show needs it.

**Q4. Give an example of a KISS violation you have encountered or could imagine.**

A common violation is creating an abstract factory, strategy registry, and interface hierarchy for a problem that a simple method with a conditional would solve. Another is introducing a message queue for inter-component communication when a direct method call would suffice, before the system ever has scaling requirements.

**Q5. How do you decide when complexity is justified?**

When the complexity is demanded by the problem — not the solution. Distributed caching is complex, but it may be essential for a high-traffic system. The question is: does removing this complexity break a real requirement? If the answer is no, the complexity is accidental and should be eliminated.

---

# 3. YAGNI — You Aren't Gonna Need It

## Definition

> "Always implement things when you actually need them, never when you just foresee that you need them."
> — Ron Jeffries, co-creator of Extreme Programming

YAGNI is the principle of not building features or abstractions until they are actually required. It is the engineering discipline of resisting the temptation to design for hypothetical future requirements.

---

## The Cost of Unused Abstractions

Every abstraction has a cost:
- **Writing cost:** Time spent building something unused
- **Reading cost:** Every developer who reads the code must understand the abstraction, even if it adds no value
- **Maintenance cost:** Unused code still breaks — APIs change, libraries update, behavior must be verified
- **Discovery cost:** Future developers must figure out if the abstraction is actually used or can be deleted
- **Flexibility cost:** Abstractions constrain future design — the "plugin points" you built may be the wrong plugin points

When the abstraction is never used, you pay all these costs with zero return.

---

## When to Design for Extensibility vs YAGNI

YAGNI does not mean "write throwaway code." It means "do not build extensibility you cannot currently justify."

**Build extensibility when:**
- You have a concrete upcoming requirement (scheduled, on the roadmap, approved)
- The cost of adding extensibility later is genuinely high (e.g., a public API that cannot be changed without breaking clients)
- The pattern is well-established and the extension points are obviously correct (e.g., Strategy pattern where you know you have multiple algorithms today)

**Apply YAGNI when:**
- You are guessing at future requirements ("we might need to support multiple databases")
- The future use case is speculative ("what if we add internationalization later?")
- The extension would not be used for the next 6-12 months at minimum
- The cost of adding it later is manageable (internal code is much easier to change than public APIs)

The key question: **Do you have a concrete requirement today?** If the answer is no, apply YAGNI.

---

## Red Flags: Premature Generalization

Watch for these patterns as signals of YAGNI violations:

**The "just in case" interface**
```java
// "We only have MySQL now but let's make it injectable just in case"
public interface DatabaseProvider {
    Connection getConnection();
}
// One implementation. Nobody will ever add a second.
```

**The plugin registry with no plugins**
```java
// "Let's support plugins for the payment system"
public class PaymentPluginRegistry {
    private final Map<String, PaymentPlugin> plugins = new HashMap<>();
    public void register(String name, PaymentPlugin plugin) { ... }
    public PaymentPlugin get(String name) { ... }
}
// There is one payment provider. There will never be a plugin.
```

**The configuration class for non-configurable behavior**
```java
// "Let's make the retry count configurable"
public class RetryConfig {
    private int maxRetries = 3;       // always 3
    private long backoffMs = 1000;    // always 1000
    // Setters provided, never called
}
```

**The abstract factory for a single product**
```java
// "Let's abstract the notification sender"
public abstract class NotificationSenderFactory {
    public abstract NotificationSender create();
}
public class EmailNotificationSenderFactory extends NotificationSenderFactory {
    @Override
    public NotificationSender create() { return new EmailNotificationSender(); }
}
// There is only ever one sender type.
```

---

## Java Examples: YAGNI Violations and Fixes

### Example 1: Over-Abstracted Repository

**YAGNI Violation:**
```java
// Designed "in case we need to support multiple data sources"
public interface UserRepository<T, ID> {
    T findById(ID id);
    List<T> findAll();
    T save(T entity);
    void delete(ID id);
    long count();
    boolean exists(ID id);
    Page<T> findAll(Pageable pageable);     // Not used
    List<T> findBySpecification(Spec spec); // Not used, Spec not even defined
}

public interface UserCache<T, ID> {         // Cache abstraction added "just in case"
    Optional<T> get(ID id);
    void put(ID id, T entity);
    void evict(ID id);
}

public class UserRepositoryImpl<T, ID>
    implements UserRepository<T, ID>, UserCache<T, ID> {
    // ... 200 lines of infrastructure nobody asked for
}
```

**YAGNI Applied:**
```java
// Build what you need today
public class UserRepository {
    private final DataSource dataSource;

    public UserRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public Optional<User> findById(long id) { ... }
    public User save(User user) { ... }
    public void delete(long id) { ... }
}
// Add pagination, caching, and generics when there is a concrete requirement for them
```

---

### Example 2: Plugin System Nobody Needs

**YAGNI Violation:**
```java
// "Let's design a plugin system for report formats"
public interface ReportExporter {
    byte[] export(Report report);
    String getFormat();
}

public class ReportExporterRegistry {
    private static final Map<String, ReportExporter> exporters = new HashMap<>();

    public static void register(ReportExporter exporter) {
        exporters.put(exporter.getFormat(), exporter);
    }

    public static ReportExporter get(String format) {
        return exporters.get(format);
    }
}

// In a system that only ever exports to PDF
public class PdfExporter implements ReportExporter {
    public byte[] export(Report report) { /* ... */ }
    public String getFormat() { return "PDF"; }
}

// The registry, interface, and registration are all unnecessary overhead
```

**YAGNI Applied:**
```java
// Export to PDF because that is what is required
public class ReportService {
    public byte[] exportToPdf(Report report) {
        // Direct implementation
        // Refactor to a plugin system WHEN a second format is actually required
    }
}
```

---

### Example 3: Configuration Premature Generalization

**YAGNI Violation:**
```java
public class EmailService {
    private final EmailConfig config;

    public EmailService(EmailConfig config) {
        this.config = config;
    }

    // EmailConfig has 20 fields, 18 of which are never set to anything
    // except their defaults
}

public class EmailConfig {
    private String host = "smtp.company.com";
    private int port = 587;
    private String protocol = "TLS";            // Always TLS
    private int connectionTimeout = 5000;       // Always 5000
    private int readTimeout = 10000;            // Always 10000
    private boolean usePool = false;            // Never changed
    private int poolSize = 10;                  // Unused since usePool = false
    private String proxyHost = null;            // Always null
    private int proxyPort = 0;                  // Always 0
    private boolean enableDkim = false;         // Never true
    private String dkimSelector = null;         // Unused
    // ... 9 more fields never configured
}
```

**YAGNI Applied:**
```java
public class EmailService {
    private static final String SMTP_HOST = "smtp.company.com";
    private static final int SMTP_PORT = 587;

    public void sendEmail(String to, String subject, String body) {
        // Use the constants directly
        // Extract to config when there is actually a requirement to vary them
    }
}
```

---

## YAGNI and Technical Debt

A common objection to YAGNI: "But it will be harder to add later!" This is sometimes true. But consider the alternative:

- You build extensibility for a feature that never comes
- You carry that dead code for months or years
- Future developers waste time understanding it
- When the real requirement arrives, it differs from what you anticipated, and you refactor anyway

The cost of adding it later is usually lower than the cost of carrying premature abstraction. The exception is public APIs that cannot be versioned — design those carefully upfront.

---

## YAGNI Interview Questions

**Q1. What is YAGNI and what problem does it address?**

YAGNI stands for "You Aren't Gonna Need It" — the principle of not building features or abstractions until they are actually required. It addresses the problem of premature generalization: engineers building plugin systems, configuration frameworks, and abstract factories for requirements that never materialize. This unused code has writing, reading, maintenance, and discovery costs with zero return.

**Q2. How do you reconcile YAGNI with designing for extensibility?**

YAGNI and extensibility are not opposites. Build extensibility when you have a concrete, scheduled requirement — not when you are guessing. If you have two payment providers today, build the abstraction. If you have one and "might add another someday," apply YAGNI and build the abstraction when the second one actually arrives. The cost of adding extension points to internal code is usually manageable; the cost of maintaining unused abstractions is certain.

**Q3. What are the costs of building something you don't need?**

Writing cost (time building unused code), reading cost (every developer must understand it), maintenance cost (it still breaks when APIs change), discovery cost (future developers must determine if it is used), and flexibility cost (it may constrain design in the wrong direction when real requirements arrive).

**Q4. When is YAGNI NOT the right answer?**

When the cost of adding something later is genuinely high — primarily for public APIs that cannot be versioned without breaking clients. Internal code is usually cheap to change later. Public APIs require careful upfront design because clients depend on them. Also, when you have a concrete upcoming requirement (approved, scheduled, in sprint) — that is not speculation, that is a real need.

**Q5. How does YAGNI relate to agile development?**

YAGNI is a foundational XP (Extreme Programming) principle and aligns naturally with agile. Agile says build to current requirements and adapt as requirements evolve. YAGNI says do not build ahead of requirements. Together, they argue for short feedback loops and incremental design rather than big upfront design. The agile premise — that requirements will change — actually makes YAGNI stronger: if requirements change, the abstraction you built for the anticipated requirement is now wrong and must be reworked anyway.
