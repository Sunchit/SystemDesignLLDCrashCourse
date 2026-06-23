# API Design for Senior Engineers

## Table of Contents

1. [Internal API vs External API Design](#1-internal-api-vs-external-api-design)
2. [Method Signature Design](#2-method-signature-design)
3. [Checked vs Unchecked Exceptions in Java APIs](#3-checked-vs-unchecked-exceptions-in-java-apis)
4. [Fluent API / Builder Pattern API Design](#4-fluent-api--builder-pattern-api-design)
5. [Defensive API Design](#5-defensive-api-design)
6. [Immutability as an API Design Principle](#6-immutability-as-an-api-design-principle)
7. [Principle of Least Surprise](#7-principle-of-least-surprise)
8. [Documentation as API Design](#8-documentation-as-api-design)
9. [API Design Mistakes (Anti-Patterns)](#9-api-design-mistakes-anti-patterns)
10. [Contract-First Design](#10-contract-first-design)
11. [Versioning Strategy for APIs](#11-versioning-strategy-for-apis)
12. [Senior vs Mid-Level Thinking](#12-senior-vs-mid-level-thinking)
13. [10 Interview Questions on API Design](#13-10-interview-questions-on-api-design)

---

## 1. Internal API vs External API Design

### The Fundamental Distinction

An API is a contract. But not all contracts carry the same weight of enforcement or the same cost of change. The most important question a senior engineer asks before designing any API is: **who will consume this, and what leverage do they have over me?**

**Internal APIs** are consumed by code you own or by teammates on the same codebase. If you need to change the signature, you change all the callers. A bad internal API decision is annoying but recoverable. You refactor, update all call sites, run the tests, and move on. The blast radius is contained to your own team's sprint.

**External APIs** are consumed by code you do not own — other teams inside the company, or external customers and partners. When you change an external API signature, you break callers you may not even know about. You cannot grep across their repositories. The blast radius extends to downstream systems, deployment schedules, SLA negotiations, and potentially contractual obligations. A bad external API decision can lock you into that shape for years.

### Library API vs Service API

These are two distinct flavors of external API, each with its own failure mode.

**Library API**: Distributed as a JAR or package. The consumer compiles against it. A change to a method signature causes a compile-time error in the caller's build. You control the artifact but not the caller's deployment schedule. If you change `UserService.findUser(int id)` to `UserService.findUser(UUID id)`, every caller must recompile and redeploy. This is a **source-breaking change**.

**Service API**: Distributed as a running service over a network. The consumer makes HTTP, gRPC, or messaging calls at runtime. A change to a REST endpoint causes a runtime error or silent data corruption. You do not even get compile-time safety. This makes service API evolution more dangerous, because breaks are discovered in production, not in the build pipeline.

### The Blast Radius of a Bad API Decision

Consider the cost at each stage:

| Stage of Discovery | Cost |
|---|---|
| Design review (pre-code) | Near zero — change a document |
| During implementation | Low — refactor your own code |
| During code review | Low to medium — change before merge |
| After merge, before release | Medium — update all internal callers |
| After release, internal consumers | High — coordinated refactor across teams |
| After release, external consumers | Very high — may require parallel versioning |
| After years of adoption | Extremely high — may be impossible |

This asymmetry explains why senior engineers invest disproportionate thought in external API design and treat internal API design with productive pragmatism. The internal API can be improved continuously. The external API must be designed for the long run.

### Designing with Change in Mind

Internal APIs should be designed for **clarity and coherence** within the codebase they serve. They can evolve as requirements evolve. The cost of change is bounded.

External APIs should be designed with the following constraints:
- Every public method name, parameter type, and return type is a commitment.
- Every field in a response payload that a client reads is a commitment.
- Additive changes (new optional fields, new endpoints) are safe.
- Removing or renaming anything is a breaking change.
- Changing semantics without changing signatures is the most insidious breaking change — it breaks clients that cannot detect the break at compile time.

The discipline of thinking like an API author — not just an API implementer — is the core of senior engineering judgment.

---

## 2. Method Signature Design

### Naming: Reveal Intent

A method name is the first documentation a caller reads. It should answer: **what does this do?** without requiring the caller to read the body.

Bad names hide intent:
- `process()` — process what?
- `handle()` — handle how?
- `calc(int x, int y)` — calculate what?
- `getData()` — what data? from where?

Good names reveal intent:
- `chargeCustomerForOrder(CustomerId customerId, OrderId orderId)`
- `findActiveSubscriptionsByCustomer(CustomerId customerId)`
- `calculateMonthlyRecurringRevenue(YearMonth period)`

**Conventions**:
- Queries return data: `find`, `get`, `calculate`, `fetch`, `list`, `count`
- Commands mutate state: `create`, `update`, `delete`, `cancel`, `activate`, `charge`
- Predicates return boolean: `isEligible`, `hasExpired`, `canProcessRefund`
- Avoid abbreviations: `calcMRR()` is harder to search and understand than `calculateMonthlyRecurringRevenue()`

### Parameter Design: The Rule of Four

When a method requires more than four parameters, it is a signal that:
1. The method has too many responsibilities, or
2. Several parameters form a cohesive concept that deserves its own type.

More than four parameters create several problems:
- Callers must remember argument order, which the compiler cannot help with for same-type parameters.
- `scheduleReport(true, false, true, false)` is unreadable at the call site.
- Adding a fifth parameter later is a source-breaking change in library APIs.

The solution is the **parameter object** (also called a value object or request object):

```java
// Bad: six parameters, two booleans, order is ambiguous
public Report generateReport(
    LocalDate startDate,
    LocalDate endDate,
    String format,
    boolean includeHeaders,
    boolean includeSummary,
    int maxRows
) { ... }

// Good: parameter object encapsulates related inputs
public final class ReportRequest {

    private final LocalDate startDate;
    private final LocalDate endDate;
    private final ReportFormat format;
    private final boolean includeHeaders;
    private final boolean includeSummary;
    private final int maxRows;

    private ReportRequest(Builder builder) {
        this.startDate = Objects.requireNonNull(builder.startDate, "startDate must not be null");
        this.endDate = Objects.requireNonNull(builder.endDate, "endDate must not be null");
        this.format = Objects.requireNonNull(builder.format, "format must not be null");
        this.includeHeaders = builder.includeHeaders;
        this.includeSummary = builder.includeSummary;
        this.maxRows = builder.maxRows > 0 ? builder.maxRows : Integer.MAX_VALUE;
    }

    public LocalDate getStartDate() { return startDate; }
    public LocalDate getEndDate() { return endDate; }
    public ReportFormat getFormat() { return format; }
    public boolean isIncludeHeaders() { return includeHeaders; }
    public boolean isIncludeSummary() { return includeSummary; }
    public int getMaxRows() { return maxRows; }

    public static Builder builder(LocalDate startDate, LocalDate endDate, ReportFormat format) {
        return new Builder(startDate, endDate, format);
    }

    public static final class Builder {
        private final LocalDate startDate;
        private final LocalDate endDate;
        private final ReportFormat format;
        private boolean includeHeaders = true;
        private boolean includeSummary = false;
        private int maxRows = Integer.MAX_VALUE;

        private Builder(LocalDate startDate, LocalDate endDate, ReportFormat format) {
            this.startDate = startDate;
            this.endDate = endDate;
            this.format = format;
        }

        public Builder withHeaders() {
            this.includeHeaders = true;
            return this;
        }

        public Builder withoutHeaders() {
            this.includeHeaders = false;
            return this;
        }

        public Builder withSummary() {
            this.includeSummary = true;
            return this;
        }

        public Builder limitTo(int maxRows) {
            if (maxRows <= 0) {
                throw new IllegalArgumentException("maxRows must be positive, got: " + maxRows);
            }
            this.maxRows = maxRows;
            return this;
        }

        public ReportRequest build() {
            return new ReportRequest(this);
        }
    }
}

// Clean method signature
public Report generateReport(ReportRequest request) { ... }

// Clean call site
Report report = reportService.generateReport(
    ReportRequest.builder(LocalDate.of(2024, 1, 1), LocalDate.of(2024, 12, 31), ReportFormat.PDF)
        .withSummary()
        .limitTo(1000)
        .build()
);
```

### Return Type Design

**Return the most specific useful type.** If a method returns a `List` when it could return an `ImmutableList`, the caller cannot tell whether mutating the returned list is safe. If a method returns `Object`, all type safety is lost.

Hierarchy of specificity for collections:
- `Collection<T>` — caller knows it is iterable, size is known, but cannot index
- `List<T>` — caller can index by position
- `ImmutableList<T>` — caller knows mutation is impossible and the method owns the data

**Never return null from a method that might have no result.** Null return types force callers to add null checks that they will forget. Use `Optional<T>` for single results that may be absent:

```java
// Bad: caller must remember to null-check
public User findUserByEmail(String email) {
    // returns null if not found
}

// Good: absence is explicit in the type
public Optional<User> findUserByEmail(String email) {
    return userRepository.findByEmail(email);
}

// Caller is forced to handle the empty case
Optional<User> maybeUser = userService.findUserByEmail("user@example.com");
User user = maybeUser.orElseThrow(() -> new UserNotFoundException(email));
```

For collections, never return null — return an empty collection:

```java
// Bad
public List<Order> findOrdersByCustomer(CustomerId customerId) {
    if (!customerExists(customerId)) {
        return null; // caller must null-check
    }
    return orderRepository.findByCustomer(customerId);
}

// Good
public List<Order> findOrdersByCustomer(CustomerId customerId) {
    if (!customerExists(customerId)) {
        return Collections.emptyList();
    }
    return orderRepository.findByCustomer(customerId);
}
```

### Boolean Parameters: An Anti-Pattern

Boolean parameters create call sites that are unreadable and fragile:

```java
// Bad: what do true and false mean?
userService.createUser("Alice", "alice@example.com", true, false);

// What does this method look like?
public User createUser(String name, String email, boolean sendWelcomeEmail, boolean requireEmailVerification) { ... }
```

At the call site, `true, false` is meaningless without reading the method signature. When a new requirement adds a third boolean, the source-breaking change affects every caller.

Solutions:

**Use enums:**
```java
public enum WelcomeEmailPolicy { SEND, SUPPRESS }
public enum EmailVerificationPolicy { REQUIRED, SKIP }

public User createUser(String name, String email, WelcomeEmailPolicy welcomePolicy, EmailVerificationPolicy verificationPolicy) { ... }

// Call site is self-documenting
userService.createUser("Alice", "alice@example.com", WelcomeEmailPolicy.SEND, EmailVerificationPolicy.REQUIRED);
```

**Use overloads for common cases:**
```java
// Default: send welcome email, require verification
public User createUser(String name, String email) {
    return createUser(name, email, WelcomeEmailPolicy.SEND, EmailVerificationPolicy.REQUIRED);
}

// Full control for specific cases
public User createUser(String name, String email, WelcomeEmailPolicy welcomePolicy, EmailVerificationPolicy verificationPolicy) { ... }
```

### Full Example: Bad vs Good Method Signature

```java
// =====================
// BAD: avoid this
// =====================
public class OrderService {

    // What is the int? Priority? Days? Version?
    // Boolean parameters are unreadable at call site
    // Returns null on failure — caller must remember to check
    // "proc" abbreviation obscures intent
    public Object procOrder(String id, boolean flag1, boolean flag2, int n) {
        return null;
    }
}

// =====================
// GOOD: well-designed
// =====================
public enum FulfillmentPriority {
    STANDARD,
    EXPEDITED,
    OVERNIGHT
}

public enum NotificationPreference {
    EMAIL_AND_SMS,
    EMAIL_ONLY,
    NONE
}

public final class OrderFulfillmentRequest {

    private final OrderId orderId;
    private final FulfillmentPriority priority;
    private final NotificationPreference notificationPreference;
    private final int maxRetryAttempts;

    private OrderFulfillmentRequest(Builder builder) {
        this.orderId = Objects.requireNonNull(builder.orderId, "orderId must not be null");
        this.priority = Objects.requireNonNull(builder.priority, "priority must not be null");
        this.notificationPreference = builder.notificationPreference;
        this.maxRetryAttempts = builder.maxRetryAttempts;
    }

    public OrderId getOrderId() { return orderId; }
    public FulfillmentPriority getPriority() { return priority; }
    public NotificationPreference getNotificationPreference() { return notificationPreference; }
    public int getMaxRetryAttempts() { return maxRetryAttempts; }

    public static Builder builder(OrderId orderId, FulfillmentPriority priority) {
        return new Builder(orderId, priority);
    }

    public static final class Builder {
        private final OrderId orderId;
        private final FulfillmentPriority priority;
        private NotificationPreference notificationPreference = NotificationPreference.EMAIL_ONLY;
        private int maxRetryAttempts = 3;

        private Builder(OrderId orderId, FulfillmentPriority priority) {
            this.orderId = orderId;
            this.priority = priority;
        }

        public Builder notifyVia(NotificationPreference preference) {
            this.notificationPreference = Objects.requireNonNull(preference);
            return this;
        }

        public Builder withMaxRetryAttempts(int maxRetryAttempts) {
            if (maxRetryAttempts < 0 || maxRetryAttempts > 10) {
                throw new IllegalArgumentException(
                    "maxRetryAttempts must be between 0 and 10, got: " + maxRetryAttempts
                );
            }
            this.maxRetryAttempts = maxRetryAttempts;
            return this;
        }

        public OrderFulfillmentRequest build() {
            return new OrderFulfillmentRequest(this);
        }
    }
}

public interface OrderService {

    /**
     * Initiates the fulfillment process for the specified order.
     *
     * @param request the fulfillment request; must not be null
     * @return the created fulfillment record with tracking information
     * @throws OrderNotFoundException if no order exists for the given order ID
     * @throws OrderAlreadyFulfilledException if the order has already been fulfilled
     * @throws FulfillmentException if the fulfillment system is unavailable
     */
    FulfillmentRecord fulfillOrder(OrderFulfillmentRequest request);
}
```

---

## 3. Checked vs Unchecked Exceptions in Java APIs

### The Original Intent of Checked Exceptions

James Gosling designed checked exceptions as a way to make the compiler enforce that callers acknowledge error conditions they are expected to handle. The intent was sound: if a method can fail due to an **external, recoverable** condition, force the caller to acknowledge that possibility.

The canonical cases were:
- `IOException` — the file might not exist, the network might be down. These are external conditions the application might meaningfully recover from.
- `SQLException` — the database might reject the query. The application might fall back or retry.

### Why Modern Java Leans Toward Unchecked

In practice, checked exceptions have proven to be more painful than helpful in most scenarios:

1. **They leak implementation details.** A service interface that throws `SQLException` is telling callers it uses JDBC. When you switch to an ORM, you must change the interface signature — a source-breaking change for every caller.

2. **They promote bad swallowing.** Developers forced to handle checked exceptions they cannot meaningfully handle often write `catch (IOException e) { /* ignore */ }` or rethrow as `RuntimeException` anyway, which defeats the purpose.

3. **They are incompatible with lambdas.** You cannot use a method that throws a checked exception inside a `Stream.map()` without try-catch noise. This is a fundamental ergonomic failure.

4. **Frameworks like Spring use unchecked.** `DataAccessException`, `TransactionException`, and all Spring framework exceptions are unchecked. The industry has largely converged on this.

### When to Use Checked Exceptions

Use a checked exception when:
- The failure is **externally caused** (not a programming error).
- The caller is **expected to recover** from it in a meaningful, application-specific way.
- Recovery logic is not generic — each caller must decide what to do.

Example: A file parsing API where the caller might want to skip malformed files, prompt the user, or log and continue:

```java
public interface CsvParser {
    List<Record> parse(InputStream input) throws MalformedCsvException;
}
```

Here, `MalformedCsvException` is checked because different callers have different recovery strategies and the compiler should ensure they are not forgotten.

### When to Use Unchecked Exceptions

Use unchecked exceptions for:
- **Programming errors**: invalid arguments, null where null is not allowed, index out of bounds. These represent bugs in the caller — they should be fixed, not caught.
- **Infrastructure failures**: database unreachable, external service timeout. These are usually not recoverable in-flight and belong in a global error handler.
- **Unrecoverable states**: invariant violations, corrupt data. Catching these would leave the system in an inconsistent state.

### Exception Translation

The most important pattern for service-layer exception design is **exception translation**: wrapping low-level infrastructure exceptions in domain-level exceptions. This prevents infrastructure details from leaking through the API boundary.

### Full Java Example: Service Layer with Proper Exception Hierarchy

```java
// ============================
// Domain exception hierarchy
// ============================

/**
 * Base exception for all domain-level failures in the payment system.
 * Unchecked to avoid forced handling at infrastructure boundaries.
 */
public class PaymentDomainException extends RuntimeException {

    public PaymentDomainException(String message) {
        super(message);
    }

    public PaymentDomainException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * Thrown when a payment cannot be processed due to a business rule violation.
 * Examples: insufficient funds, card declined, expired card.
 * Callers may want to communicate this to the end user.
 */
public class PaymentDeclinedException extends PaymentDomainException {

    private final DeclineReason declineReason;

    public PaymentDeclinedException(DeclineReason reason, String message) {
        super(message);
        this.declineReason = Objects.requireNonNull(reason);
    }

    public DeclineReason getDeclineReason() {
        return declineReason;
    }
}

public enum DeclineReason {
    INSUFFICIENT_FUNDS,
    CARD_EXPIRED,
    FRAUD_SUSPECTED,
    CARD_BLOCKED,
    GATEWAY_DECLINED
}

/**
 * Thrown when the payment gateway is temporarily unavailable.
 * The operation may be retried. The cause contains the underlying technical failure.
 */
public class PaymentGatewayUnavailableException extends PaymentDomainException {

    public PaymentGatewayUnavailableException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * Thrown when attempting to process a payment for an order that does not exist.
 */
public class OrderNotFoundException extends PaymentDomainException {

    private final OrderId orderId;

    public OrderNotFoundException(OrderId orderId) {
        super("No order found with id: " + orderId);
        this.orderId = orderId;
    }

    public OrderId getOrderId() {
        return orderId;
    }
}

// ============================
// Service interface
// ============================

public interface PaymentService {

    /**
     * Processes a payment for the given order.
     *
     * @param orderId the ID of the order to pay for; must not be null
     * @param paymentMethod the payment method to charge; must not be null
     * @return a payment receipt with transaction details
     * @throws OrderNotFoundException if the order does not exist
     * @throws PaymentDeclinedException if the payment is declined by the gateway
     * @throws PaymentGatewayUnavailableException if the gateway cannot be reached
     */
    PaymentReceipt processPayment(OrderId orderId, PaymentMethod paymentMethod);
}

// ============================
// Implementation with exception translation
// ============================

public class PaymentServiceImpl implements PaymentService {

    private final OrderRepository orderRepository;
    private final PaymentGatewayClient gatewayClient;

    public PaymentServiceImpl(OrderRepository orderRepository, PaymentGatewayClient gatewayClient) {
        this.orderRepository = Objects.requireNonNull(orderRepository);
        this.gatewayClient = Objects.requireNonNull(gatewayClient);
    }

    @Override
    public PaymentReceipt processPayment(OrderId orderId, PaymentMethod paymentMethod) {
        Objects.requireNonNull(orderId, "orderId must not be null");
        Objects.requireNonNull(paymentMethod, "paymentMethod must not be null");

        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        GatewayResponse response;
        try {
            response = gatewayClient.charge(paymentMethod, order.getTotalAmount());
        } catch (GatewayConnectionException e) {
            // Translate infrastructure exception to domain exception.
            // The cause is preserved for logging and debugging.
            throw new PaymentGatewayUnavailableException(
                "Payment gateway is unavailable. Retry is safe.", e
            );
        } catch (GatewayApiException e) {
            // Translate gateway-level decline codes into domain exceptions.
            DeclineReason reason = mapGatewayDeclineCode(e.getCode());
            throw new PaymentDeclinedException(reason,
                "Payment declined: " + reason.name().toLowerCase().replace('_', ' ')
            );
        }

        return PaymentReceipt.from(order, response);
    }

    private DeclineReason mapGatewayDeclineCode(String gatewayCode) {
        return switch (gatewayCode) {
            case "INSUFFICIENT_FUNDS"   -> DeclineReason.INSUFFICIENT_FUNDS;
            case "EXPIRED_CARD"         -> DeclineReason.CARD_EXPIRED;
            case "SUSPECTED_FRAUD"      -> DeclineReason.FRAUD_SUSPECTED;
            case "CARD_BLOCKED"         -> DeclineReason.CARD_BLOCKED;
            default                     -> DeclineReason.GATEWAY_DECLINED;
        };
    }
}
```

Notice that `GatewayConnectionException` and `GatewayApiException` — which are infrastructure types — never escape the service layer. Callers only ever see domain exceptions. This is exception translation done correctly.

---

## 4. Fluent API / Builder Pattern API Design

### Why Fluent APIs Are Usable APIs

A fluent API reads like prose. The caller constructs an object or orchestrates an operation through a chain of method calls, each of which returns a context that guides the next step. The result is code that is both self-documenting and difficult to misuse.

Compare:
```java
// Without fluent API: order of arguments matters, intent is obscure
HttpRequest req = new HttpRequest("POST", "https://api.example.com/orders", headers, body, 30, true);

// With fluent API: self-documenting, order does not matter for optional parts
HttpRequest req = HttpRequest.post("https://api.example.com/orders")
    .header("Authorization", "Bearer " + token)
    .header("Content-Type", "application/json")
    .body(requestBody)
    .timeout(Duration.ofSeconds(30))
    .followRedirects()
    .build();
```

### Builder Pattern: Mandatory vs Optional Fields

The builder pattern should make mandatory fields unavoidable and optional fields convenient. A common mistake is making all fields optional on the builder, which allows callers to call `build()` with nothing set, producing a broken object.

**Design principle**: Mandatory fields should be required at construction time (constructor parameters of the builder), not discovered at `build()` time through validation.

```java
// Bad: all fields optional — caller can build an invalid object
HttpRequest req = new HttpRequest.Builder()
    .build(); // No URL set — will fail at runtime

// Good: mandatory fields in builder constructor
HttpRequest req = HttpRequest.post("https://api.example.com/orders") // URL is mandatory
    .build(); // Valid — has everything required
```

### Full Java Example: QueryBuilder

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;

/**
 * A fluent builder for constructing type-safe SQL SELECT queries.
 *
 * <p>Usage:
 * <pre>
 *   Query query = Query.from("orders")
 *       .select("id", "customer_id", "total_amount")
 *       .where("status = ?", "PENDING")
 *       .and("created_at > ?", LocalDate.now().minusDays(30))
 *       .orderBy("created_at", SortDirection.DESC)
 *       .limit(100)
 *       .build();
 * </pre>
 *
 * <p>The table name is mandatory and must be provided at entry point.
 * All other clauses are optional with sensible defaults (SELECT *, no WHERE, no limit).
 */
public final class Query {

    private final String table;
    private final List<String> selectedColumns;
    private final List<WhereClause> whereClauses;
    private final List<OrderByClause> orderByClauses;
    private final int limit;
    private final int offset;

    private Query(Builder builder) {
        this.table = builder.table;
        this.selectedColumns = Collections.unmodifiableList(new ArrayList<>(builder.selectedColumns));
        this.whereClauses = Collections.unmodifiableList(new ArrayList<>(builder.whereClauses));
        this.orderByClauses = Collections.unmodifiableList(new ArrayList<>(builder.orderByClauses));
        this.limit = builder.limit;
        this.offset = builder.offset;
    }

    /**
     * Begins building a SELECT query against the given table.
     *
     * @param table the table name; must not be null or blank
     * @return a builder for further query configuration
     */
    public static Builder from(String table) {
        if (table == null || table.isBlank()) {
            throw new IllegalArgumentException("table must not be null or blank");
        }
        return new Builder(table);
    }

    public String getTable() { return table; }
    public List<String> getSelectedColumns() { return selectedColumns; }
    public List<WhereClause> getWhereClauses() { return whereClauses; }
    public List<OrderByClause> getOrderByClauses() { return orderByClauses; }
    public int getLimit() { return limit; }
    public int getOffset() { return offset; }

    public boolean hasLimit() { return limit > 0; }
    public boolean hasOffset() { return offset > 0; }

    @Override
    public String toString() {
        StringBuilder sql = new StringBuilder("SELECT ");
        sql.append(selectedColumns.isEmpty() ? "*" : String.join(", ", selectedColumns));
        sql.append(" FROM ").append(table);

        if (!whereClauses.isEmpty()) {
            sql.append(" WHERE ");
            for (int i = 0; i < whereClauses.size(); i++) {
                if (i > 0) sql.append(" AND ");
                sql.append(whereClauses.get(i).getCondition());
            }
        }

        if (!orderByClauses.isEmpty()) {
            sql.append(" ORDER BY ");
            List<String> parts = new ArrayList<>();
            for (OrderByClause clause : orderByClauses) {
                parts.add(clause.getColumn() + " " + clause.getDirection().name());
            }
            sql.append(String.join(", ", parts));
        }

        if (limit > 0) sql.append(" LIMIT ").append(limit);
        if (offset > 0) sql.append(" OFFSET ").append(offset);

        return sql.toString();
    }

    public static final class Builder {

        private final String table;
        private final List<String> selectedColumns = new ArrayList<>();
        private final List<WhereClause> whereClauses = new ArrayList<>();
        private final List<OrderByClause> orderByClauses = new ArrayList<>();
        private int limit = -1;
        private int offset = -1;

        private Builder(String table) {
            this.table = table;
        }

        /**
         * Specifies the columns to select. If not called, SELECT * is used.
         *
         * @param columns one or more column names; must not be null or contain null
         * @return this builder
         */
        public Builder select(String... columns) {
            Objects.requireNonNull(columns, "columns must not be null");
            for (String column : columns) {
                if (column == null || column.isBlank()) {
                    throw new IllegalArgumentException("Column names must not be null or blank");
                }
                selectedColumns.add(column);
            }
            return this;
        }

        /**
         * Adds a WHERE clause. Multiple calls are combined with AND.
         *
         * @param condition the SQL condition with ? placeholders; must not be null or blank
         * @param parameters the bind parameters
         * @return this builder
         */
        public Builder where(String condition, Object... parameters) {
            validateCondition(condition);
            whereClauses.add(new WhereClause(condition, parameters));
            return this;
        }

        /**
         * Alias for {@link #where(String, Object...)} to allow readable chaining:
         * {@code .where(...).and(...)}.
         */
        public Builder and(String condition, Object... parameters) {
            return where(condition, parameters);
        }

        /**
         * Adds an ORDER BY clause.
         *
         * @param column    the column to order by; must not be null or blank
         * @param direction the sort direction; must not be null
         * @return this builder
         */
        public Builder orderBy(String column, SortDirection direction) {
            if (column == null || column.isBlank()) {
                throw new IllegalArgumentException("column must not be null or blank");
            }
            Objects.requireNonNull(direction, "direction must not be null");
            orderByClauses.add(new OrderByClause(column, direction));
            return this;
        }

        /**
         * Limits the number of results returned.
         *
         * @param limit must be a positive integer
         * @return this builder
         */
        public Builder limit(int limit) {
            if (limit <= 0) {
                throw new IllegalArgumentException("limit must be positive, got: " + limit);
            }
            this.limit = limit;
            return this;
        }

        /**
         * Skips the first N results.
         *
         * @param offset must be a non-negative integer
         * @return this builder
         */
        public Builder offset(int offset) {
            if (offset < 0) {
                throw new IllegalArgumentException("offset must be non-negative, got: " + offset);
            }
            this.offset = offset;
            return this;
        }

        /**
         * Builds and returns the immutable {@link Query}.
         *
         * @return the constructed query
         */
        public Query build() {
            return new Query(this);
        }

        private void validateCondition(String condition) {
            if (condition == null || condition.isBlank()) {
                throw new IllegalArgumentException("condition must not be null or blank");
            }
        }
    }

    // ---- Supporting types ----

    public static final class WhereClause {
        private final String condition;
        private final Object[] parameters;

        public WhereClause(String condition, Object[] parameters) {
            this.condition = condition;
            this.parameters = parameters.clone();
        }

        public String getCondition() { return condition; }
        public Object[] getParameters() { return parameters.clone(); }
    }

    public static final class OrderByClause {
        private final String column;
        private final SortDirection direction;

        public OrderByClause(String column, SortDirection direction) {
            this.column = column;
            this.direction = direction;
        }

        public String getColumn() { return column; }
        public SortDirection getDirection() { return direction; }
    }

    public enum SortDirection { ASC, DESC }
}
```

**Usage:**
```java
Query query = Query.from("orders")
    .select("id", "customer_id", "total_amount", "status")
    .where("status = ?", "PENDING")
    .and("created_at > ?", LocalDate.now().minusDays(30))
    .orderBy("created_at", Query.SortDirection.DESC)
    .limit(100)
    .offset(200)
    .build();

System.out.println(query);
// SELECT id, customer_id, total_amount, status FROM orders
// WHERE status = ? AND created_at > ? ORDER BY created_at DESC LIMIT 100 OFFSET 200
```

### When NOT to Use Fluent APIs

Fluent APIs have costs:
- **They are harder to debug.** A chain of 10 method calls produces a single stack frame at the call site if something throws.
- **They are harder to subclass.** Method chaining breaks down in inheritance hierarchies because `return this` returns the parent type.
- **They add boilerplate.** A simple two-parameter method does not need a builder. Reserve the pattern for genuinely complex construction with many optional parameters.

Rule of thumb: if the object has more than three meaningful optional configuration points, a builder is warranted. For simpler cases, a plain constructor or static factory method is cleaner.

---

## 5. Defensive API Design

### Null Safety at API Boundaries

Every public method that accepts parameters is an API boundary. Parameters should be validated immediately at entry, before any computation begins. This is the **fail-fast** principle applied to API design.

Annotations `@NonNull` and `@Nullable` (from `org.springframework.lang`, `javax.annotation`, or `org.jetbrains.annotations`) are not enforced by the Java compiler but are valuable for:
- Documentation of intent
- IDE null-safety analysis
- Static analysis tools (SpotBugs, Error Prone, NullAway)

```java
import org.springframework.lang.NonNull;
import org.springframework.lang.Nullable;

public interface CustomerRepository {

    @NonNull
    Customer save(@NonNull Customer customer);

    @NonNull
    Optional<Customer> findById(@NonNull CustomerId id);

    @NonNull
    List<Customer> findByEmail(@Nullable String email); // null means "any email"
}
```

### Input Validation at API Boundaries

Use `Objects.requireNonNull` for null checks. For richer validation, create a reusable `Preconditions` utility or use Guava's.

```java
import java.util.Objects;

public class Preconditions {

    private Preconditions() {
        // utility class
    }

    public static <T> T requireNonNull(T value, String paramName) {
        if (value == null) {
            throw new IllegalArgumentException(paramName + " must not be null");
        }
        return value;
    }

    public static String requireNonBlank(String value, String paramName) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException(paramName + " must not be null or blank");
        }
        return value;
    }

    public static int requirePositive(int value, String paramName) {
        if (value <= 0) {
            throw new IllegalArgumentException(paramName + " must be positive, got: " + value);
        }
        return value;
    }

    public static int requireInRange(int value, int min, int max, String paramName) {
        if (value < min || value > max) {
            throw new IllegalArgumentException(
                paramName + " must be between " + min + " and " + max + ", got: " + value
            );
        }
        return value;
    }

    public static void checkState(boolean condition, String message) {
        if (!condition) {
            throw new IllegalStateException(message);
        }
    }

    public static void checkArgument(boolean condition, String message) {
        if (!condition) {
            throw new IllegalArgumentException(message);
        }
    }
}
```

### Full Java Example: PaymentService with Defensive Design

```java
import java.math.BigDecimal;
import java.util.Objects;

public final class Money {

    private final BigDecimal amount;
    private final String currencyCode;

    private Money(BigDecimal amount, String currencyCode) {
        this.amount = amount;
        this.currencyCode = currencyCode;
    }

    public static Money of(BigDecimal amount, String currencyCode) {
        Objects.requireNonNull(amount, "amount must not be null");
        Preconditions.requireNonBlank(currencyCode, "currencyCode");

        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("amount must not be negative, got: " + amount);
        }
        if (currencyCode.length() != 3) {
            throw new IllegalArgumentException(
                "currencyCode must be a 3-letter ISO 4217 code, got: " + currencyCode
            );
        }

        return new Money(amount.setScale(2, java.math.RoundingMode.HALF_UP), currencyCode.toUpperCase());
    }

    public BigDecimal getAmount() { return amount; }
    public String getCurrencyCode() { return currencyCode; }

    public boolean isZero() { return amount.compareTo(BigDecimal.ZERO) == 0; }

    @Override
    public String toString() { return amount + " " + currencyCode; }
}

public interface PaymentService {

    /**
     * Charges the specified payment method for the given amount.
     *
     * <p>Preconditions:
     * <ul>
     *   <li>{@code paymentMethodId} must not be null</li>
     *   <li>{@code amount} must not be null and must be positive</li>
     *   <li>{@code idempotencyKey} must not be null or blank — used to deduplicate retries</li>
     * </ul>
     *
     * <p>Postconditions:
     * <ul>
     *   <li>On success, the returned receipt has a non-null transaction ID</li>
     *   <li>Charging twice with the same idempotency key returns the same receipt (idempotent)</li>
     * </ul>
     *
     * @param paymentMethodId the payment method to charge
     * @param amount          the amount to charge; must be positive and non-null
     * @param idempotencyKey  a unique key to ensure at-most-once processing
     * @return a receipt confirming the charge
     * @throws PaymentDeclinedException           if the gateway declines the charge
     * @throws PaymentGatewayUnavailableException if the gateway cannot be reached
     */
    PaymentReceipt charge(
        PaymentMethodId paymentMethodId,
        Money amount,
        String idempotencyKey
    );
}

public class PaymentServiceImpl implements PaymentService {

    private final PaymentGatewayClient gatewayClient;
    private final IdempotencyStore idempotencyStore;

    public PaymentServiceImpl(
        PaymentGatewayClient gatewayClient,
        IdempotencyStore idempotencyStore
    ) {
        this.gatewayClient = Objects.requireNonNull(gatewayClient, "gatewayClient must not be null");
        this.idempotencyStore = Objects.requireNonNull(idempotencyStore, "idempotencyStore must not be null");
    }

    @Override
    public PaymentReceipt charge(
        PaymentMethodId paymentMethodId,
        Money amount,
        String idempotencyKey
    ) {
        // Validate all inputs at the boundary immediately
        Objects.requireNonNull(paymentMethodId, "paymentMethodId must not be null");
        Objects.requireNonNull(amount, "amount must not be null");
        Preconditions.requireNonBlank(idempotencyKey, "idempotencyKey");

        if (amount.isZero()) {
            throw new IllegalArgumentException("amount must be positive, got zero");
        }

        // Idempotency check: return cached result if we already processed this key
        return idempotencyStore.findByKey(idempotencyKey)
            .orElseGet(() -> executeCharge(paymentMethodId, amount, idempotencyKey));
    }

    private PaymentReceipt executeCharge(
        PaymentMethodId paymentMethodId,
        Money amount,
        String idempotencyKey
    ) {
        GatewayResponse response;
        try {
            response = gatewayClient.charge(paymentMethodId, amount);
        } catch (GatewayConnectionException e) {
            throw new PaymentGatewayUnavailableException("Gateway unreachable during charge", e);
        } catch (GatewayApiException e) {
            throw new PaymentDeclinedException(mapDeclineReason(e.getCode()), e.getMessage());
        }

        PaymentReceipt receipt = PaymentReceipt.from(response);

        // Postcondition check: ensure our own invariant is satisfied
        Preconditions.checkState(
            receipt.getTransactionId() != null,
            "Gateway returned a success response without a transaction ID — this is a bug"
        );

        idempotencyStore.store(idempotencyKey, receipt);
        return receipt;
    }

    private DeclineReason mapDeclineReason(String gatewayCode) {
        return switch (gatewayCode) {
            case "INSUFFICIENT_FUNDS" -> DeclineReason.INSUFFICIENT_FUNDS;
            case "EXPIRED_CARD"       -> DeclineReason.CARD_EXPIRED;
            default                   -> DeclineReason.GATEWAY_DECLINED;
        };
    }
}
```

Key defensive design moves in this example:
1. All parameters are validated at the very first lines of the public method.
2. The idempotency key prevents duplicate charges on retries.
3. A postcondition assertion catches a contract violation from an external dependency.
4. Infrastructure exceptions are translated to domain exceptions before leaving the service.

---

## 6. Immutability as an API Design Principle

### Why Immutable Return Types Are Safer APIs

When a method returns a mutable object, it creates an implicit question: **does this return a defensive copy or a live reference to internal state?**

If the caller mutates the returned object and it is a live reference, you have just allowed external code to modify the internals of your service's data — a classic encapsulation breach. If you return a defensive copy every time, you waste memory unnecessarily. If you forget to make a defensive copy, you have a bug that is very hard to detect.

The solution is to return **immutable objects**. The question disappears entirely. The caller can hold the reference, pass it around, and never worry about whether their changes will corrupt the service's state — because changes are impossible.

### Immutable Collections as Return Values

Never return a `java.util.ArrayList` or `java.util.HashMap` directly from a service method. Return an immutable view:

```java
// Bad: caller can modify your internal list
public List<Order> getPendingOrders() {
    return this.pendingOrders; // Caller can call pendingOrders.clear()!
}

// Also bad: slightly safer but still mutable
public List<Order> getPendingOrders() {
    return new ArrayList<>(this.pendingOrders); // Defensive copy, but still mutable
}

// Good: truly immutable
public List<Order> getPendingOrders() {
    return Collections.unmodifiableList(new ArrayList<>(this.pendingOrders));
}

// Better (Java 10+): more intention-revealing
public List<Order> getPendingOrders() {
    return List.copyOf(this.pendingOrders);
}
```

### Value Objects in APIs

Domain value objects (Money, Address, CustomerId, OrderId) should be immutable. They represent a quantity or identity, not a mutable entity. They are safe to share across threads and across layers without copying.

```java
public final class CustomerId {

    private final String value;

    private CustomerId(String value) {
        this.value = value;
    }

    public static CustomerId of(String value) {
        Preconditions.requireNonBlank(value, "CustomerId value");
        return new CustomerId(value.trim());
    }

    public String getValue() { return value; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof CustomerId)) return false;
        return value.equals(((CustomerId) o).value);
    }

    @Override
    public int hashCode() { return value.hashCode(); }

    @Override
    public String toString() { return "CustomerId{" + value + "}"; }
}
```

### Full Java Example: Returning Immutable Domain Objects

```java
import java.time.Instant;
import java.util.List;
import java.util.Objects;

public final class OrderSummary {

    private final OrderId orderId;
    private final CustomerId customerId;
    private final List<LineItem> lineItems;   // immutable
    private final Money totalAmount;
    private final OrderStatus status;
    private final Instant createdAt;

    private OrderSummary(Builder builder) {
        this.orderId = Objects.requireNonNull(builder.orderId, "orderId");
        this.customerId = Objects.requireNonNull(builder.customerId, "customerId");
        this.lineItems = List.copyOf(
            Objects.requireNonNull(builder.lineItems, "lineItems")
        );
        this.totalAmount = Objects.requireNonNull(builder.totalAmount, "totalAmount");
        this.status = Objects.requireNonNull(builder.status, "status");
        this.createdAt = Objects.requireNonNull(builder.createdAt, "createdAt");
    }

    public OrderId getOrderId() { return orderId; }
    public CustomerId getCustomerId() { return customerId; }
    public List<LineItem> getLineItems() { return lineItems; }   // already immutable
    public Money getTotalAmount() { return totalAmount; }
    public OrderStatus getStatus() { return status; }
    public Instant getCreatedAt() { return createdAt; }

    /**
     * Returns a new OrderSummary with the updated status.
     * The original is unchanged.
     */
    public OrderSummary withStatus(OrderStatus newStatus) {
        return new Builder()
            .orderId(this.orderId)
            .customerId(this.customerId)
            .lineItems(this.lineItems)
            .totalAmount(this.totalAmount)
            .status(Objects.requireNonNull(newStatus))
            .createdAt(this.createdAt)
            .build();
    }

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private OrderId orderId;
        private CustomerId customerId;
        private List<LineItem> lineItems;
        private Money totalAmount;
        private OrderStatus status;
        private Instant createdAt;

        public Builder orderId(OrderId orderId) { this.orderId = orderId; return this; }
        public Builder customerId(CustomerId customerId) { this.customerId = customerId; return this; }
        public Builder lineItems(List<LineItem> lineItems) { this.lineItems = lineItems; return this; }
        public Builder totalAmount(Money totalAmount) { this.totalAmount = totalAmount; return this; }
        public Builder status(OrderStatus status) { this.status = status; return this; }
        public Builder createdAt(Instant createdAt) { this.createdAt = createdAt; return this; }
        public OrderSummary build() { return new OrderSummary(this); }
    }
}

public interface OrderQueryService {

    /**
     * Returns an immutable summary of the order. The returned object is safe
     * to hold across threads and will not reflect future state changes.
     */
    Optional<OrderSummary> findOrderSummary(OrderId orderId);

    /**
     * Returns an immutable list of order summaries for the customer.
     * The list and all elements within it are unmodifiable.
     */
    List<OrderSummary> findOrdersByCustomer(CustomerId customerId);
}
```

Because `OrderSummary` is immutable and `List.copyOf` is used, every caller gets a value that cannot be tampered with. Thread safety for reads is guaranteed without synchronization.

---

## 7. Principle of Least Surprise

### API Should Behave as Callers Expect

The principle of least surprise states that a function should behave consistently with how a reasonable developer, familiar with the conventions of the language and your codebase, would expect it to behave based solely on its name and signature.

Violating this principle produces code that works perfectly on the happy path, passes code review because the implementation is correct, and then causes production incidents because callers made reasonable assumptions that turned out to be wrong.

### Naming Consistency Across the Codebase

If your codebase uses `findById` for database lookups in 20 classes, introduce `lookupById` in class 21 and callers will lose time discovering it is the same thing. Consistency is a feature. Pick a naming convention and apply it uniformly.

Common conventions to standardize:
- `findBy*` vs `getBy*` — choose one for repository lookups
- `save` vs `create` vs `persist` — choose one for persisting new entities
- `update` vs `modify` vs `patch` — choose one for mutations
- `delete` vs `remove` — choose one for deletions

### Side-Effect Transparency

A method that has a name suggesting a query (`get`, `find`, `list`) but also modifies state is a severe violation of least surprise. A method that returns `void` but silently sends an email violates it in a different way. Side effects must be visible in the name.

```java
// BAD: name suggests a query, but actually creates a record in the audit log
public User getUser(UserId userId) {
    auditLog.record("USER_ACCESSED", userId); // hidden side effect
    return userRepository.findById(userId).orElseThrow();
}

// GOOD: separate concerns — query is a query, audit is explicit
public User findUser(UserId userId) {
    return userRepository.findById(userId).orElseThrow(() -> new UserNotFoundException(userId));
}

public void recordUserAccess(UserId userId, AuditContext context) {
    auditLog.record("USER_ACCESSED", userId, context);
}
```

### Idempotency Design

An operation is idempotent if calling it multiple times produces the same result as calling it once. Idempotent APIs are safer to use in distributed systems where retries are necessary.

Design idempotency explicitly:
- `PUT` operations that replace a resource are naturally idempotent.
- `POST` operations that create resources are not idempotent unless you enforce an idempotency key.
- `DELETE` operations should be idempotent — deleting an already-deleted resource should succeed silently (or return 404, but not 500).

### Full Java Example: Surprising vs Unsurprising APIs

```java
// =============================================
// BAD: multiple violations of least surprise
// =============================================
public class BadOrderService {

    // Surprise 1: "get" sounds like a pure query, but it sends an email
    public Order getOrder(String id) {
        Order order = repository.findById(id);
        notificationService.sendViewNotification(id); // unexpected side effect
        return order;
    }

    // Surprise 2: "cancel" sounds like it cancels, but actually archives
    public void cancelOrder(String id) {
        Order order = repository.findById(id);
        order.setStatus("ARCHIVED");    // callers expect "CANCELLED"
        repository.save(order);
    }

    // Surprise 3: returns null for "not found" — callers who forget null checks get NPE
    public Order findByCustomer(String customerId) {
        return repository.findByCustomer(customerId); // may be null
    }

    // Surprise 4: not idempotent — throws on second call for same order
    public void processRefund(String orderId) {
        Order order = repository.findById(orderId);
        if (order.getStatus().equals("REFUNDED")) {
            throw new IllegalStateException("Already refunded"); // breaks retry logic
        }
        // process refund...
    }
}

// =============================================
// GOOD: no surprises
// =============================================
public class GoodOrderService {

    // Pure query — no side effects, name matches behavior
    public Optional<Order> findOrderById(OrderId orderId) {
        Objects.requireNonNull(orderId, "orderId must not be null");
        return repository.findById(orderId);
    }

    // Side effect clearly named — separate from the query
    public void recordOrderView(OrderId orderId, UserId viewedBy) {
        Objects.requireNonNull(orderId, "orderId must not be null");
        Objects.requireNonNull(viewedBy, "viewedBy must not be null");
        auditService.recordView(orderId, viewedBy);
    }

    // Cancellation does what it says — status is CANCELLED
    public Order cancelOrder(OrderId orderId, CancellationReason reason) {
        Objects.requireNonNull(orderId, "orderId must not be null");
        Objects.requireNonNull(reason, "reason must not be null");

        Order order = findOrderById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        if (order.getStatus() == OrderStatus.CANCELLED) {
            return order; // idempotent: already done
        }
        if (!order.isCancellable()) {
            throw new OrderNotCancellableException(orderId, order.getStatus());
        }

        return repository.save(order.withStatus(OrderStatus.CANCELLED).withCancellationReason(reason));
    }

    // Idempotent: second call with same orderId is a no-op, not an error
    public RefundReceipt processRefund(OrderId orderId, String idempotencyKey) {
        Objects.requireNonNull(orderId, "orderId must not be null");
        Preconditions.requireNonBlank(idempotencyKey, "idempotencyKey");

        return idempotencyStore.findByKey(idempotencyKey)
            .map(stored -> (RefundReceipt) stored)
            .orElseGet(() -> executeRefund(orderId, idempotencyKey));
    }

    private RefundReceipt executeRefund(OrderId orderId, String idempotencyKey) {
        // actual refund logic
        RefundReceipt receipt = refundGateway.processRefund(orderId);
        idempotencyStore.store(idempotencyKey, receipt);
        return receipt;
    }
}
```

---

## 8. Documentation as API Design

### Javadoc as a Contract, Not Decoration

Most engineers treat Javadoc as something written after the fact, if at all. Senior engineers understand that Javadoc written before or during implementation is a design activity: it forces you to articulate what the method does, what it requires, what it guarantees, and how it fails. If you cannot write the Javadoc clearly, the design is probably not clear either.

Javadoc is a contract between the API author and the caller. It defines:
- What the method **requires** as preconditions
- What the method **guarantees** as postconditions
- What the method **does** in prose
- What exceptions it **may throw** and under what conditions
- Whether it is **thread-safe**
- Whether it is **idempotent**

### What to Document

`@param` — not just the name, but the constraints: null-safety, range, format, encoding.
`@return` — what is in the return value, what it means, whether it can be null/empty.
`@throws` — the condition that causes the exception, not just that it might throw.
Thread safety — whether the method is safe to call from multiple threads concurrently.
Idempotency — whether repeated calls produce the same result.
Side effects — any mutations, I/O, or external calls the caller cannot see from the signature.

### Full Java Example: Well-Documented vs Undocumented Service

```java
// ============================================
// BAD: no documentation — caller has no contract
// ============================================
public interface UndocumentedUserService {

    User createUser(String name, String email, String role);

    User findUser(String id);

    void updateUserRole(String userId, String role);

    void deactivateUser(String userId);

    List<User> listUsers(String role, int page, int size);
}


// ============================================
// GOOD: documented service with full contract
// ============================================

/**
 * Manages user lifecycle operations including creation, retrieval, role management, and deactivation.
 *
 * <p>Thread safety: Implementations of this interface must be thread-safe. All methods may be
 * called concurrently from multiple threads.
 *
 * <p>Transactions: Methods that mutate state ({@link #createUser}, {@link #updateUserRole},
 * {@link #deactivateUser}) are executed within a transaction. If any step fails, the entire
 * operation is rolled back.
 */
public interface UserService {

    /**
     * Creates a new active user with the specified name, email, and role.
     *
     * <p>The email address must be unique within the system. If a user with the same email
     * already exists, the call fails immediately without creating a duplicate.
     *
     * <p>A welcome email is sent asynchronously after the user is created. Failure to send
     * the email does not roll back the user creation.
     *
     * @param name  the user's display name; must not be null or blank; max 200 characters
     * @param email the user's email address; must be a valid RFC 5321 address; must not be null;
     *              must be unique within the system
     * @param role  the role to assign to the new user; must not be null
     * @return the newly created user with a system-generated ID and creation timestamp
     * @throws DuplicateEmailException if a user with the given email already exists
     * @throws InvalidEmailException   if the email address does not conform to RFC 5321
     * @throws IllegalArgumentException if name is null, blank, or exceeds 200 characters;
     *                                  or if role is null
     */
    User createUser(String name, String email, UserRole role);

    /**
     * Retrieves the user with the specified ID.
     *
     * <p>This is a read-only operation with no side effects.
     *
     * @param userId the ID of the user to retrieve; must not be null
     * @return an {@link Optional} containing the user, or {@link Optional#empty()} if no user
     *         exists with the given ID
     * @throws IllegalArgumentException if userId is null
     */
    Optional<User> findUserById(UserId userId);

    /**
     * Updates the role of an existing active user.
     *
     * <p>This operation is idempotent: assigning the same role the user already has results in
     * no state change and returns successfully.
     *
     * <p>Deactivated users cannot have their role updated. Attempting to update a deactivated
     * user's role throws {@link UserDeactivatedException}.
     *
     * @param userId  the ID of the user whose role is to be updated; must not be null
     * @param newRole the new role to assign; must not be null
     * @throws UserNotFoundException    if no user exists with the given ID
     * @throws UserDeactivatedException if the user is deactivated
     * @throws IllegalArgumentException if userId or newRole is null
     */
    void updateUserRole(UserId userId, UserRole newRole);

    /**
     * Deactivates the user with the specified ID.
     *
     * <p>A deactivated user cannot log in and is excluded from all operational queries.
     * Deactivation is reversible via a separate reactivation operation not defined in this interface.
     *
     * <p>This operation is idempotent: deactivating an already-deactivated user returns
     * successfully without error.
     *
     * <p>All active sessions for the user are invalidated immediately as part of this operation.
     *
     * @param userId the ID of the user to deactivate; must not be null
     * @throws UserNotFoundException if no user exists with the given ID
     * @throws IllegalArgumentException if userId is null
     */
    void deactivateUser(UserId userId);

    /**
     * Returns a page of users, optionally filtered by role.
     *
     * <p>Users are returned in ascending order of creation time. Deactivated users are excluded
     * from results unless {@code includeDeactivated} is true on the provided page request.
     *
     * @param role     the role to filter by; may be null to retrieve users of any role
     * @param pageable pagination and sorting parameters; must not be null; page index is zero-based
     * @return an immutable list of users matching the criteria; never null; may be empty
     * @throws IllegalArgumentException if pageable is null, or if page index is negative,
     *                                  or if page size is not between 1 and 200
     */
    List<User> listUsers(UserRole role, PageRequest pageable);
}
```

---

## 9. API Design Mistakes (Anti-Patterns)

### 1. Leaking Implementation Details

An API that exposes the types, structure, or behavior of its underlying implementation is tightly coupled to that implementation. Changing the implementation breaks the API.

```java
// BAD: the return type is a Hibernate entity — callers see lazy-loaded fields,
// session management requirements, and JPA annotations
public HibernateUserEntity findUser(long id) { ... }

// BAD: the return type reveals that we use JDBC ResultSet internally
public ResultSet findUsers(String role) { ... }

// GOOD: return a domain type owned by the API
public Optional<User> findUser(UserId id) { ... }
public List<User> findUsersByRole(UserRole role) { ... }
```

### 2. Returning Mutable Internal State

```java
// BAD: callers can modify the cart's items list, corrupting internal state
public class ShoppingCart {
    private final List<CartItem> items = new ArrayList<>();

    public List<CartItem> getItems() {
        return items; // live reference — mutating this corrupts the cart
    }
}

// GOOD: return an immutable copy
public class ShoppingCart {
    private final List<CartItem> items = new ArrayList<>();

    public List<CartItem> getItems() {
        return List.copyOf(items); // immutable — callers cannot corrupt state
    }
}
```

### 3. Boolean Soup Parameters

```java
// BAD: four booleans in sequence — unreadable, fragile
public void sendNotification(String userId, boolean email, boolean sms, boolean push, boolean inApp) { ... }

// Call site is incomprehensible
notificationService.sendNotification("user-123", true, false, true, false);

// GOOD: use a dedicated type
public enum NotificationChannel { EMAIL, SMS, PUSH, IN_APP }

public void sendNotification(UserId userId, Set<NotificationChannel> channels) { ... }

// Self-documenting call site
notificationService.sendNotification(
    UserId.of("user-123"),
    EnumSet.of(NotificationChannel.EMAIL, NotificationChannel.PUSH)
);
```

### 4. Overloading Abuse

Overloading is useful when the alternatives are logically equivalent operations on different types. It becomes an anti-pattern when overloads have subtly different semantics that callers cannot distinguish from the name alone.

```java
// BAD: three overloads with different semantics — the int vs long ambiguity
// causes subtle bugs when values exceed Integer.MAX_VALUE
public User findUser(int id) { ... }        // legacy ID
public User findUser(long id) { ... }       // new ID — different storage
public User findUser(String id) { ... }     // email lookup — completely different logic

// GOOD: distinct method names for distinct operations
public Optional<User> findUserByLegacyId(int legacyId) { ... }
public Optional<User> findUserById(UserId id) { ... }
public Optional<User> findUserByEmail(EmailAddress email) { ... }
```

### 5. Long Parameter Lists

Already covered in Section 2. The solution is always a parameter object or builder.

```java
// BAD: seven parameters — order matters, same-type parameters are interchangeable by mistake
public void scheduleDelivery(String orderId, String address, String city, String state,
                              String zip, String country, String deliveryInstructions) { ... }

// Compiler will not catch transposed string arguments:
scheduleDelivery(orderId, address, zip, city, state, country, instructions); // wrong order, compiles

// GOOD: parameter object makes the structure explicit
public void scheduleDelivery(OrderId orderId, DeliveryAddress address) { ... }
```

### 6. Temporal Coupling

Temporal coupling is when two method calls must be invoked in a specific order, but the API does not enforce this. Violations cause runtime errors or subtle bugs.

```java
// BAD: caller must call open() before read(), but there is nothing to enforce this
public class FileProcessor {
    public void open(String path) { ... }
    public String read() { ... }    // throws if open() was not called
    public void close() { ... }
}

// Caller who forgets open() gets a confusing error at runtime
FileProcessor processor = new FileProcessor();
processor.read(); // NullPointerException or IllegalStateException

// GOOD: use a factory method that returns an already-open processor
// Construction IS initialization
public class FileProcessor implements Closeable {

    private final BufferedReader reader;

    private FileProcessor(BufferedReader reader) {
        this.reader = reader;
    }

    public static FileProcessor open(Path path) throws IOException {
        return new FileProcessor(Files.newBufferedReader(path));
    }

    public String readLine() throws IOException {
        return reader.readLine();
    }

    @Override
    public void close() throws IOException {
        reader.close();
    }
}

// Usage: temporal dependency is structurally enforced
try (FileProcessor processor = FileProcessor.open(Path.of("/var/data/input.txt"))) {
    String line;
    while ((line = processor.readLine()) != null) {
        process(line);
    }
}
```

### Full Comparison Table

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Leaking implementation details | Tightly couples callers to implementation | Return domain types |
| Returning mutable internal state | External code can corrupt internal state | Return `List.copyOf()` or unmodifiable collections |
| Boolean soup | Call site is unreadable, adding a boolean is breaking | Use enums or named option types |
| Overloading abuse | Callers cannot distinguish semantically different operations | Use distinct method names |
| Long parameter lists | Order matters, same-type confusion, breaking to add params | Parameter object / builder |
| Temporal coupling | Callers must know initialization order | Construction = initialization; use factory methods |

---

## 10. Contract-First Design

### Define the API Before the Implementation

Contract-first design is the practice of specifying the **interface** before writing any implementation code. For a Java service, this means writing the interface (or abstract class) and its full Javadoc before touching the concrete class.

This discipline produces better designs for several reasons:

1. **You are forced to think from the caller's perspective.** When writing code, it is natural to expose implementation details. When writing an interface, you are forced to hide them.

2. **The design is reviewable before any code is written.** A one-page interface review is faster and cheaper than a five-file implementation review.

3. **It enables parallel development.** Once the interface is agreed upon, team A implements it while team B writes code that depends on it. Neither is blocked.

4. **It drives testability.** An interface is trivially mockable. Code that depends on interfaces is trivially unit-testable.

### Interface-First Development

```java
// Step 1: Write the interface with complete Javadoc
// This is your contract — reviewable, discussable, not yet implemented

/**
 * Provides inventory management operations for the warehouse system.
 *
 * <p>All mutations are transactional and will roll back on failure.
 * All reads are consistent reads and will not return uncommitted data.
 */
public interface InventoryService {

    /**
     * Reserves the specified quantity of a product for a pending order.
     *
     * @param productId the product to reserve; must not be null
     * @param quantity  the quantity to reserve; must be positive
     * @param orderId   the order for which the reservation is being made; must not be null
     * @return the reservation token required to confirm or release the reservation
     * @throws InsufficientStockException if available stock is less than the requested quantity
     * @throws ProductNotFoundException   if the product does not exist in the catalog
     */
    ReservationToken reserveStock(ProductId productId, int quantity, OrderId orderId);

    /**
     * Confirms a previously made reservation, permanently reducing available stock.
     *
     * <p>Idempotent: confirming an already-confirmed reservation succeeds without error.
     *
     * @param token the reservation token from {@link #reserveStock}; must not be null
     * @throws ReservationExpiredException   if the reservation has expired
     * @throws ReservationNotFoundException  if no reservation exists for the given token
     */
    void confirmReservation(ReservationToken token);

    /**
     * Releases a reservation, returning the reserved quantity to available stock.
     *
     * <p>Idempotent: releasing an already-released reservation succeeds without error.
     *
     * @param token the reservation token from {@link #reserveStock}; must not be null
     * @throws ReservationNotFoundException  if no reservation exists for the given token
     */
    void releaseReservation(ReservationToken token);

    /**
     * Returns the current available stock for a product, excluding quantities reserved
     * by pending orders.
     *
     * @param productId the product to query; must not be null
     * @return the available quantity; never negative
     * @throws ProductNotFoundException if the product does not exist
     */
    int getAvailableStock(ProductId productId);
}

// Step 2: Write tests against the interface (test-driven development)
// Step 3: Implement the interface
// Step 4: Wire up the implementation — callers never know the concrete type
```

### How This Changes Design Conversations on Teams

When a team reviews a 500-line implementation class, the conversation naturally focuses on the code: whether the algorithm is correct, whether there are null checks, whether the exception handling is right. These are important, but they are the wrong questions to answer first.

When a team reviews a 30-line interface with Javadoc, the conversation naturally focuses on the contract: should this method exist at all? Is the name right? Are the error cases correct? What should happen on retry? Is this operation idempotent?

These are the questions that, if answered incorrectly, require refactoring the implementation and updating every caller. Getting them right before writing code is the highest-leverage design activity a senior engineer can perform.

Contract-first design codifies this into a workflow: write the interface, review it, get consensus, then implement.

---

## 11. Versioning Strategy for APIs

### Additive-Only Evolution

The safest form of API evolution is additive: **only add, never remove or rename**. For service APIs, this means:
- Adding new optional fields to request/response payloads is safe.
- Adding new endpoints is safe.
- Adding new enum values requires care — callers with switch statements will not handle the new value.
- Changing field types, removing fields, or renaming fields is a breaking change.

For library APIs, additive changes include:
- Adding new methods to a class (safe if the class is not designed for extension).
- Adding new methods to an interface — **breaking** for all implementations unless default methods are used.
- Adding new default methods to an interface — safe for existing implementations.

### When to Bump the Major Version

Bump the major version (v1 to v2, `1.0.0` to `2.0.0`) when:
- You are removing a method, endpoint, or field.
- You are changing the semantics of an existing method without changing its signature.
- You are changing a method signature.
- You are making a change that requires all callers to update their code.

Major versions indicate: "This is not backward compatible. Migration is required."

### URL Versioning vs Header Versioning (REST)

Two dominant patterns for REST API versioning:

**URL versioning**: The version is in the path.
```
GET /v1/orders/{id}
GET /v2/orders/{id}
```
Pros: Visible, cacheable, easy to route to different handlers. Cons: URLs should identify resources, not API versions — violates REST purity.

**Header versioning**: The version is in a request header.
```
GET /orders/{id}
Accept: application/vnd.example.orders-v2+json
```
Pros: URLs remain stable. Cons: Harder to test in a browser, caching is more complex, less discoverable.

**Practical recommendation**: For internal APIs and developer-facing APIs, URL versioning is pragmatic and discoverable. For pure REST API purists in a hypermedia context, header versioning is correct. Most large engineering organizations (Google, Stripe, Twilio) use URL versioning.

### Full Java Example: Versioning a Service Interface

```java
// ============================================================
// V1: Original interface — deployed and in use
// ============================================================

/**
 * @deprecated Use {@link UserServiceV2} which supports multi-tenancy.
 * Migration guide: replace {@link #findById(String)} with
 * {@link UserServiceV2#findById(UserId, TenantId)}.
 */
@Deprecated(since = "2.0.0", forRemoval = true)
public interface UserServiceV1 {
    Optional<User> findById(String userId);
    User createUser(String name, String email);
}

// ============================================================
// V2: Evolved interface — backward compatible where possible,
// uses default methods to ease migration
// ============================================================

public interface UserServiceV2 {

    /**
     * Finds a user within a specific tenant.
     *
     * @param userId   the user ID; must not be null
     * @param tenantId the tenant context; must not be null
     * @return the user, or empty if not found
     */
    Optional<User> findById(UserId userId, TenantId tenantId);

    /**
     * Creates a user within a specific tenant.
     *
     * @param request the user creation request; must not be null
     * @return the created user
     */
    User createUser(CreateUserRequest request);

    /**
     * Lists all users within a tenant.
     * Added in v2 — not present in v1.
     *
     * @param tenantId the tenant context; must not be null
     * @param pageable pagination parameters; must not be null
     * @return an immutable list of users; never null
     */
    List<User> listUsers(TenantId tenantId, PageRequest pageable);
}

// ============================================================
// Adapter: bridge V1 callers to V2 implementation
// Allows gradual migration without forcing simultaneous updates
// ============================================================

public class UserServiceV1Adapter implements UserServiceV1 {

    private final UserServiceV2 delegate;
    private final TenantId defaultTenantId;

    public UserServiceV1Adapter(UserServiceV2 delegate, TenantId defaultTenantId) {
        this.delegate = Objects.requireNonNull(delegate);
        this.defaultTenantId = Objects.requireNonNull(defaultTenantId);
    }

    @Override
    @Deprecated
    public Optional<User> findById(String userId) {
        return delegate.findById(UserId.of(userId), defaultTenantId);
    }

    @Override
    @Deprecated
    public User createUser(String name, String email) {
        return delegate.createUser(
            CreateUserRequest.builder()
                .name(name)
                .email(email)
                .tenantId(defaultTenantId)
                .build()
        );
    }
}

// ============================================================
// REST controller: URL versioning strategy
// ============================================================

@RestController
public class UserController {

    private final UserServiceV1 userServiceV1;  // for /v1 — deprecated, for migration period
    private final UserServiceV2 userServiceV2;  // for /v2 — current

    public UserController(UserServiceV1 userServiceV1, UserServiceV2 userServiceV2) {
        this.userServiceV1 = userServiceV1;
        this.userServiceV2 = userServiceV2;
    }

    @GetMapping("/v1/users/{id}")
    @Deprecated
    public ResponseEntity<User> getUserV1(@PathVariable String id) {
        return userServiceV1.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @GetMapping("/v2/users/{id}")
    public ResponseEntity<User> getUserV2(
        @PathVariable String id,
        @RequestHeader("X-Tenant-Id") String tenantId
    ) {
        return userServiceV2.findById(UserId.of(id), TenantId.of(tenantId))
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}
```

---

## 12. Senior vs Mid-Level Thinking

This section captures the key differences in how a senior engineer and a mid-level engineer approach API design decisions. The differences are not about syntax or language mastery — they are about perspective, foresight, and the weight placed on different concerns.

| Dimension | Mid-Level Engineer | Senior Engineer |
|---|---|---|
| **Primary question** | "Does this method work?" | "Is this the right contract for callers to build against?" |
| **When design happens** | After the implementation is mostly written | Before the first line of implementation is written |
| **Null handling** | Returns null when no result, adds a comment | Returns `Optional<T>`, never returns null from a query method |
| **Exception design** | Throws whatever exception is convenient | Designs a domain exception hierarchy; translates infrastructure exceptions |
| **Parameter count** | Adds parameters as needed | Extracts parameter objects after three parameters; uses builders after five |
| **Boolean parameters** | Uses `boolean` for on/off behavior | Uses enums or separate overloads; reads the call site, not the declaration |
| **Mutability** | Returns internal collections directly | Returns `List.copyOf()` or unmodifiable views; designs value objects to be immutable |
| **Documentation** | Writes Javadoc after the code is done | Writes Javadoc as a design activity; treats `@throws` as part of the API contract |
| **Versioning** | Modifies the interface when requirements change | Evolves the interface additively; uses adapters and deprecation for migration |
| **Naming** | Names match implementation (e.g., `processData`) | Names reveal intent from the caller's perspective (e.g., `calculateInvoiceTotalForPeriod`) |
| **Side effects** | Hidden inside methods with misleading names | Explicit in method names; query methods have no side effects |
| **Idempotency** | Not considered | Designed upfront for operations that may be retried |
| **Change cost awareness** | Thinks about the implementation cost of the change | Thinks about the cost to all callers and the blast radius of a bad decision |
| **Testing** | Tests the implementation against its own internals | Tests against the interface; uses fakes and mocks so tests are decoupled from implementation |
| **Temporal coupling** | Creates `init()` and `open()` methods callers must call first | Uses constructor/factory patterns so construction equals initialization |
| **Error granularity** | Throws a generic `Exception` or `RuntimeException` | Throws specific domain exceptions with enough detail to diagnose and recover |
| **API review focus** | Reviews that the code is correct | Reviews that the contract is right — is this the best shape for long-term callers? |

A mid-level engineer who writes correct, tested, clean code is valuable. A senior engineer who designs APIs that the team can safely depend on for years — whose APIs are hard to misuse, whose error conditions are explicit, whose contracts can evolve without breaking callers — creates value that compounds over time.

---

## 13. 10 Interview Questions on API Design

---

### Question 1

**What is the difference between a checked and an unchecked exception in Java, and when should you use each in a service API?**

**Answer:**

A checked exception is a subclass of `Exception` but not `RuntimeException`. The Java compiler requires that callers either catch it or declare it in their own `throws` clause. An unchecked exception is a subclass of `RuntimeException`. The compiler does not require acknowledgment.

The original intent was that checked exceptions model externally caused, recoverable failures where the caller has a meaningful recovery strategy. The canonical examples are `IOException` (the network might be down, you might retry or fail over) and `ParseException` (the input might be malformed, you might prompt the user).

Modern Java practice, and the entire Spring framework, leans heavily toward unchecked exceptions for several reasons:
1. Checked exceptions leak implementation details. A `throws SQLException` declaration tells callers you use JDBC — when you switch to an ORM, you must change the interface and break all callers.
2. Checked exceptions are incompatible with lambda streams without noisy try-catch blocks.
3. Most infrastructure failures (database down, network unreachable) are not recoverable in a meaningful way at the call site — they belong in a global error handler.

**Decision rule:**
- Use **checked** when the failure is externally caused, the caller has a meaningful and application-specific recovery path, and omitting the catch would be a logical error (e.g., `MalformedCsvException` in a file parsing API where different callers need different recovery strategies).
- Use **unchecked** for programming errors (`IllegalArgumentException`, `NullPointerException`), infrastructure failures translated to domain exceptions, and any situation where a generic recovery strategy (log, return error response, retry) is appropriate.

Always translate infrastructure exceptions to domain exceptions at the service layer boundary. Never let `SQLException`, `JedisException`, or `HttpClientErrorException` escape a service layer.

---

### Question 2

**You are designing a Java method that needs to accept several configuration options. How do you decide between a plain parameter list, a parameter object, and a builder?**

**Answer:**

The decision depends on three factors: number of parameters, optionality, and future extensibility.

**Plain parameter list** is appropriate when:
- There are three or fewer parameters.
- All parameters are required.
- The signature is unlikely to change.

```java
public Order placeOrder(CustomerId customerId, ProductId productId, int quantity)
```

**Parameter object** is appropriate when:
- There are four or more parameters that form a coherent concept.
- All or most parameters are required.
- The object will be reused (e.g., passed through multiple layers, stored for retry).

```java
public Order placeOrder(PlaceOrderRequest request)
```

**Builder** is appropriate when:
- There are many optional parameters.
- Readability at the call site matters more than construction brevity.
- You want to enforce constraints during construction.
- You want to make mandatory vs. optional fields structurally clear.

```java
public Order placeOrder(PlaceOrderRequest.builder(customerId, productId, quantity)
    .withPriority(FulfillmentPriority.EXPEDITED)
    .withGiftMessage("Happy Birthday")
    .build())
```

The key insight is that **the call site, not the implementation, is the primary design target**. Write the call site you want to read in six months, then design the method signature to produce it.

---

### Question 3

**Why is returning null from a Java service method considered an anti-pattern, and what should you do instead?**

**Answer:**

Returning null from a method that might have no result is an anti-pattern for three reasons:

1. **It is invisible at the call site.** A method with a return type of `User` gives the caller no indication that null is a possible return value. The caller must know — from memory or documentation — to add a null check. This knowledge is never enforced by the compiler.

2. **It forces defensive null checks everywhere.** Every caller must add `if (result == null)` before using the value. This is noise in the code that obscures the actual logic. Forgetting one null check produces a `NullPointerException` that may surface far from the original call.

3. **It is semantically ambiguous.** Does null mean "not found"? "Not applicable"? "Error occurred"? "Deferred"? Null has no semantics — you must infer them from context.

The correct alternatives:

**`Optional<T>`** for a single result that may be absent:
```java
public Optional<User> findUserByEmail(String email) {
    return userRepository.findByEmail(email);
}

// Caller is forced to handle the empty case
User user = userService.findUserByEmail("x@example.com")
    .orElseThrow(() -> new UserNotFoundException(email));
```

**Empty collection** for methods that return zero or more results:
```java
public List<Order> findOrdersByCustomer(CustomerId customerId) {
    return orderRepository.findByCustomer(customerId); // guaranteed non-null, may be empty
}
```

**Throw an exception** when absence is a violation of a precondition (the caller should have verified existence before calling):
```java
// If the contract says "orderId must refer to an existing order"
public Order getOrder(OrderId orderId) {
    return orderRepository.findById(orderId)
        .orElseThrow(() -> new OrderNotFoundException(orderId));
}
```

`Optional` was added to Java 8 precisely to make optionality an explicit, type-safe concern. Returning `Optional` is not about avoiding null internally — it is about making the absence of a value visible in the API contract.

---

### Question 4

**What is temporal coupling in API design, and how do you eliminate it?**

**Answer:**

Temporal coupling occurs when an API requires the caller to invoke methods in a specific sequence, but this requirement is not encoded in the API's type system. Violating the sequence produces a runtime failure — usually a `NullPointerException` or `IllegalStateException` — at a location far removed from the point of failure.

Classic example:
```java
DatabaseConnection conn = new DatabaseConnection();
conn.connect("localhost", 5432);  // must be called first
ResultSet rs = conn.query("SELECT 1");  // fails with NPE if connect() was not called
conn.close();
```

There is nothing in the type system preventing a caller from calling `query()` before `connect()`. The coupling is temporal — it exists only in the documentation (if any) and the developer's memory.

**Elimination strategies:**

1. **Make construction equal initialization.** Use a static factory method that performs all initialization steps and returns a fully ready object. The two-step initialization is collapsed into one:
```java
try (DatabaseConnection conn = DatabaseConnection.openTo("localhost", 5432)) {
    ResultSet rs = conn.query("SELECT 1");
}
// connect() is not a separate step — the connection is open from the moment of construction
```

2. **Use a step builder.** If there are multiple required steps, model each step as a distinct type returned from the previous step's completion:
```java
public interface ConnectionStep { AuthenticationStep connectTo(String host, int port); }
public interface AuthenticationStep { DatabaseConnection authenticateWith(String user, String password); }

// Usage is enforced by the type system — you cannot call query() before authentication
DatabaseConnection conn = ConnectionBuilder.start()
    .connectTo("localhost", 5432)
    .authenticateWith("user", "secret");
```

3. **Use `Closeable` / try-with-resources.** When a resource must be opened and closed, implement `Closeable` and rely on try-with-resources to enforce the lifecycle.

Temporal coupling is a design smell, not a code smell. It cannot be detected by static analysis — it requires understanding the intended usage sequence.

---

### Question 5

**How do you design an API to be idempotent, and why does it matter?**

**Answer:**

An operation is idempotent if applying it multiple times has the same observable effect as applying it once. In distributed systems, network failures and timeouts make retries necessary. If the original request succeeded but the response was lost, the client cannot know — so it retries. If the operation is not idempotent, the retry causes a duplicate.

**Why it matters:** A payment charged twice, an email sent twice, an order created twice. These are correctness bugs, not performance bugs. The system appears to work but produces wrong results.

**Design patterns for idempotency:**

1. **Idempotency keys.** The caller generates a unique key for each logical operation and includes it in the request. The server records the result against the key. On retry, the server returns the stored result:
```java
public PaymentReceipt chargeCard(PaymentRequest request, String idempotencyKey) {
    return idempotencyStore.findByKey(idempotencyKey)
        .orElseGet(() -> {
            PaymentReceipt receipt = gateway.charge(request);
            idempotencyStore.store(idempotencyKey, receipt);
            return receipt;
        });
}
```

2. **Natural idempotency.** Design operations as state assignments rather than state transitions. `PUT /users/{id}/status` with body `{"status": "ACTIVE"}` is naturally idempotent — calling it twice does not double-activate. `POST /users/{id}/activate` is not — it may fire an activation event each time.

3. **Conditional operations.** Use ETags and `If-Match` headers in REST APIs to make writes conditional on the current state, preventing stale updates from succeeding.

4. **Safe failure on duplicate.** For create operations where idempotency keys are unavailable, define a natural uniqueness constraint and return the existing resource instead of failing:
```java
// Instead of throwing DuplicateException, return the existing resource
public Subscription createOrFindSubscription(CustomerId customerId, PlanId planId) {
    return subscriptionRepository.findByCustomerAndPlan(customerId, planId)
        .orElseGet(() -> subscriptionRepository.save(new Subscription(customerId, planId)));
}
```

The key design principle is: **idempotency should be designed upfront**, not retrofitted after the first production duplicate incident.

---

### Question 6

**What is exception translation, and why is it important in a layered architecture?**

**Answer:**

Exception translation is the practice of catching exceptions from one layer (infrastructure, third-party library) and wrapping them in exceptions that belong to the domain layer. The translated exception carries the original as its `cause` (for logging and debugging) but exposes only domain-level semantics to callers.

**Why it is important:**

1. **Encapsulation.** A `UserService` that throws `MongoWriteException` is telling the caller that it uses MongoDB. When the team switches to PostgreSQL, the exception type changes — breaking every caller. A `UserService` that throws `UserPersistenceException` is implementation-neutral.

2. **Abstraction.** Domain layers should express failures in domain terms. "The payment was declined" is a domain fact. "The HTTP client received a 402 from Stripe with body X" is an infrastructure detail. These belong at different levels of abstraction.

3. **Caller usability.** Callers that depend on the service cannot meaningfully catch `HibernateException` or `JedisException` — they have no business reason to know about those types. Catching `ServiceUnavailableException` is meaningful and actionable.

**Implementation:**

```java
@Override
public User findUserById(UserId userId) {
    try {
        return userRepository.findById(userId.getValue());
    } catch (MongoException e) {
        // Infrastructure exception is the cause — preserved for logs
        // Domain exception is what callers see
        throw new UserRepositoryUnavailableException(
            "Failed to retrieve user " + userId + " from storage", e
        );
    }
}
```

The key discipline: at every layer boundary, catch the layer's outbound exceptions and rethrow in the inbound layer's vocabulary. This creates clean, independently evolvable layers where each layer only knows about the exceptions it owns.

---

### Question 7

**How would you design a fluent API in Java? What are the tradeoffs compared to a traditional API?**

**Answer:**

A fluent API is a design that allows chained method calls, where each method returns a context object (typically `this` or a new state object) that enables the next step. The result is call sites that read like prose.

**Design principles:**

1. **Each method returns the builder/context** so calls can be chained.
2. **Mandatory fields go in the entry-point method** (static factory or constructor) so they cannot be omitted.
3. **Optional fields go in the builder methods** with sensible defaults.
4. **The terminal method** (`build()`, `execute()`) validates the complete configuration and constructs the result.
5. **Validation on setters** fails fast rather than accumulating errors for `build()` time.

```java
HttpRequest request = HttpRequest.post("https://api.example.com/users")
    .header("Authorization", "Bearer " + token)
    .jsonBody(userPayload)
    .timeout(Duration.ofSeconds(10))
    .retryOn(IOException.class).withMaxAttempts(3)
    .build();
```

**Advantages:**
- Self-documenting call sites — the intent is visible without reading the method signature.
- Compiler-guided construction — the IDE completes the next method in the chain.
- Mandatory parameters are structurally enforced at the entry point.

**Tradeoffs and limitations:**
- **Harder to extend via inheritance.** `return this` returns the parent type, breaking the chain for subclasses. Solutions include generics (`Builder<T extends Builder<T>>`) but this is complex.
- **Harder to debug.** A single chained statement produces one stack frame at the call site. Split the chain into variables when debugging.
- **Adds boilerplate.** For simple two- or three-parameter construction, a plain constructor is cleaner and equally readable.
- **Mutable builder state.** The builder is mutable by design. In multi-threaded scenarios, builders should not be shared across threads.

Use fluent APIs for genuinely complex object construction with many optional parts. Do not use them for simple method calls — the overhead is not justified.

---

### Question 8

**What is the "principle of least surprise" in API design, and can you give examples of violations?**

**Answer:**

The principle of least surprise (also called the principle of least astonishment) states that an API should behave consistently with how a well-informed, reasonable developer would expect it to behave, based solely on its name and signature. If a caller has to read the source code to know what a method does, the API has violated this principle.

**Violations and fixes:**

**Violation 1: Query methods with hidden side effects.**
```java
// Surprise: "get" implies a pure read, but this sends an audit event
public User getUser(UserId userId) {
    auditLog.record("USER_READ", userId); // hidden mutation
    return repository.findById(userId);
}
// Fix: use separate methods; name the query method to exclude side effects
public Optional<User> findUserById(UserId userId) { ... }
public void recordUserAccess(UserId userId, ActorId actor) { ... }
```

**Violation 2: Inconsistent naming conventions.**
```java
// Some repositories use findBy*, others use getBy*, others use lookupBy*
// Caller must search the codebase each time to find the right method
userRepository.findByEmail(email);
orderRepository.getByCustomerId(customerId);
paymentRepository.lookupByTransactionId(transactionId);
// Fix: standardize on one verb across the entire codebase
```

**Violation 3: Non-idempotent operations without indication.**
```java
// Surprise: calling this twice sends two emails
public void sendConfirmation(OrderId orderId) {
    emailService.send(buildConfirmationEmail(orderId)); // not idempotent
}
// Fix: make idempotency explicit or enforce it
public void sendConfirmationIfNotSent(OrderId orderId) { ... }
```

**Violation 4: Exception types that contradict the operation.**
```java
// Surprise: a find operation throws IllegalStateException instead of NotFoundException
public User findUser(UserId userId) {
    if (!exists(userId)) throw new IllegalStateException("User not found");
}
// Fix: throw an appropriate domain exception
public Optional<User> findUser(UserId userId) { ... }
// or
public User getUser(UserId userId) { // "get" implies presence is a precondition
    return repository.findById(userId).orElseThrow(() -> new UserNotFoundException(userId));
}
```

Violations of least surprise are particularly dangerous because they produce bugs that are invisible in code review — the code is correct by inspection, but incorrect in behavior when callers apply their reasonable assumptions.

---

### Question 9

**How do you approach versioning a Java library API when you need to make a breaking change?**

**Answer:**

Breaking changes in a library API require a structured approach because you cannot update all callers simultaneously — they compile independently and may not update immediately or ever.

**Step 1: Exhaust additive options first.** Can you express the new behavior without removing or changing existing methods?
- Add a new overload.
- Add a new method with a different name.
- Add a default method to an interface.
- Add a new class alongside the existing one.

```java
// V1: existing method — do NOT change
public Optional<User> findById(String userId) { ... }

// V2: new method added, V1 still works
public Optional<User> findById(UserId userId) { ... }
```

**Step 2: Deprecate the old API.** Mark it with `@Deprecated`, document the replacement in the Javadoc, set `forRemoval = true` if removal is planned, and note the version in `since`:

```java
/**
 * @deprecated since 2.1.0. Use {@link #findById(UserId)} for type-safe lookups.
 *             This method will be removed in version 3.0.0.
 */
@Deprecated(since = "2.1.0", forRemoval = true)
public Optional<User> findById(String userId) {
    return findById(UserId.of(userId));
}
```

**Step 3: If a true breaking change is unavoidable, bump the major version** and document a migration guide. Use a version-specific package if the APIs need to coexist in the same classpath:

```java
// V1 in package com.example.user.v1
// V2 in package com.example.user.v2
// Adapter bridges V1 callers to V2 implementation
```

**Step 4: Provide migration tooling.** For large internal codebases, consider a codemods script (Rewrite recipes for Java) that automates the migration. Reducing the cost of migration increases the likelihood that callers will actually migrate.

**Key principle:** The library author owns the migration burden, not the library callers. Make it as easy as possible for callers to move to the new API.

---

### Question 10

**A teammate adds a new boolean parameter to an existing public service method. What are the problems with this, and how would you refactor it?**

**Answer:**

Adding a boolean parameter to an existing public method creates several problems:

**Problem 1: Breaking change.** Every call site that uses positional arguments must be updated. If the method is an interface method, all implementations must add the parameter.

**Problem 2: Unreadable call sites.** `userService.createUser("Alice", "alice@example.com", true)` — what does `true` mean? The caller must navigate to the method signature to understand it.

**Problem 3: Combinatorial explosion.** One boolean is manageable. Two booleans (`true, false`) are becoming unclear. Three booleans (`true, false, true`) are incomprehensible and impossible to test exhaustively. Each boolean doubles the test matrix.

**Problem 4: Future-proofing is false.** A boolean can only represent two states. If the requirement changes to three states (e.g., "send email", "send SMS", "send both"), the boolean is now wrong and must be replaced — another breaking change.

**Refactoring options:**

**Option A: Use an enum** — captures the semantics, extensible to new values without API changes:
```java
public enum NotificationStrategy { SEND_EMAIL, SEND_SMS, SEND_BOTH, SUPPRESS }

public User createUser(String name, String email, NotificationStrategy notificationStrategy) { ... }
```

**Option B: Use method overloads** — default behavior with no parameter, explicit behavior with the option:
```java
// Default: always send welcome email
public User createUser(String name, String email) {
    return createUser(name, email, NotificationStrategy.SEND_EMAIL);
}

// Explicit: caller chooses
public User createUser(String name, String email, NotificationStrategy strategy) { ... }
```

**Option C: Use a parameter object** — encapsulates all options, extensible without changing the method signature:
```java
public User createUser(CreateUserRequest request) { ... }

// Adding a new option later: add a field to CreateUserRequest, not to the method signature
CreateUserRequest request = CreateUserRequest.builder("Alice", "alice@example.com")
    .notifyVia(NotificationStrategy.SEND_EMAIL)
    .build();
```

Option C is the most future-proof for methods that are likely to grow more configuration options. The method signature becomes a stable contract regardless of how many options are added to `CreateUserRequest`.

The root lesson: **method signatures are a public contract, not an internal detail.** Adding a boolean is the path of least resistance for the implementer and the highest-resistance path for every caller.
