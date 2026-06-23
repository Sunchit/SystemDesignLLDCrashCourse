# Error Handling, Resilience Patterns, and Observability

**Course:** Industry-Grade Low-Level Design for Senior Software Engineers
**Module:** 05 — Senior Engineer Design Thinking

---

## Table of Contents

- [Part 1: Error Handling Design](#part-1-error-handling-design)
  - [1. Exception Design Philosophy](#1-exception-design-philosophy)
  - [2. Checked vs Unchecked Exceptions in Practice](#2-checked-vs-unchecked-exceptions-in-practice)
  - [3. Exception Hierarchy Design](#3-exception-hierarchy-design)
  - [4. Error Codes vs Exceptions](#4-error-codes-vs-exceptions)
  - [5. Result Types / Either Pattern in Java](#5-result-types--either-pattern-in-java)
  - [6. Fail-Fast vs Fail-Safe](#6-fail-fast-vs-fail-safe)
  - [7. Error Propagation Strategy](#7-error-propagation-strategy)
  - [8. Wrapping Third-Party Exceptions](#8-wrapping-third-party-exceptions)
- [Part 2: Resilience Patterns at the Code Level](#part-2-resilience-patterns-at-the-code-level)
  - [9. Retry with Exponential Backoff](#9-retry-with-exponential-backoff)
  - [10. Circuit Breaker Pattern](#10-circuit-breaker-pattern)
  - [11. Bulkhead Pattern](#11-bulkhead-pattern)
  - [12. Timeout Pattern](#12-timeout-pattern)
  - [13. Fallback Pattern](#13-fallback-pattern)
  - [14. Graceful Degradation](#14-graceful-degradation)
- [Part 3: Observability Considerations in Design](#part-3-observability-considerations-in-design)
  - [15. Designing for Observability from the Start](#15-designing-for-observability-from-the-start)
  - [16. Structured Logging Design](#16-structured-logging-design)
  - [17. Metrics Instrumentation at Design Level](#17-metrics-instrumentation-at-design-level)
  - [18. Distributed Tracing Hooks](#18-distributed-tracing-hooks)
  - [19. Health Check Design](#19-health-check-design)
  - [20. Senior vs Mid-level Thinking](#20-senior-vs-mid-level-thinking)

---

# Part 1: Error Handling Design

## 1. Exception Design Philosophy

Exceptions exist to communicate that something unexpected has happened — something that the caller cannot reasonably ignore. That definition immediately implies a discipline: exceptions should not be used as a normal control-flow mechanism.

### The Core Principle

When a method throws an exception it is saying: "I cannot produce the result you asked for, and the reason is important enough that you cannot simply check a return value." Using exceptions for expected, routine outcomes — such as checking whether a user exists — is an abuse of the mechanism. It is expensive (stack unwinding, object allocation), it clutters the call stack in profilers and logs, and it trains callers to expect exceptions during normal operation, which eventually leads to broad, poorly-scoped catch blocks.

### Two Failure Modes

Every failure in a system falls into one of two categories. Getting this classification right determines how you respond.

**Programming errors (bugs):** The caller violated the contract. A null was passed where a non-null was required. An index was out of bounds. A method was called on an object in an invalid state. These failures indicate a defect in the code. The correct response is to fail loudly and immediately so the defect is discovered and fixed. Catching and suppressing programming errors hides bugs.

**Operational errors (expected failures):** The environment did not cooperate. A network call timed out. A database row was not found. A downstream service returned 503. These failures are expected in any distributed system. The correct response is to handle them deliberately: retry, fall back, return an appropriate error to the caller, or propagate in a structured way.

### Fail Fast for Programming Errors

When you detect a programming error, do not attempt to recover. Throw immediately.

```java
public void processPayment(Payment payment) {
    // Fail fast — this is a contract violation, not an operational error
    Objects.requireNonNull(payment, "payment must not be null");
    if (payment.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
        throw new IllegalArgumentException(
            "payment amount must be positive, got: " + payment.getAmount());
    }
    // proceed
}
```

Using `Objects.requireNonNull` and `IllegalArgumentException` here is intentional. These are unchecked exceptions precisely because they represent bugs in the caller — they should propagate to a top-level error boundary where they will be logged and surfaced as a 500 or a crash.

### Handle Gracefully for Operational Errors

Operational errors should be caught at the layer that can meaningfully respond to them.

```java
public PaymentResult processPayment(Payment payment) {
    try {
        return paymentGateway.charge(payment);
    } catch (PaymentGatewayUnavailableException e) {
        // Operational failure — we know how to handle this
        log.warn("Payment gateway unavailable, attempting fallback", e);
        return fallbackProcessor.charge(payment);
    }
}
```

The key discipline: do not catch operational errors at a layer where you cannot do anything meaningful with them. Catching and re-throwing a bare `Exception` without adding context is almost always wrong.

---

## 2. Checked vs Unchecked Exceptions in Practice

Java's checked exception system was designed with good intentions: force callers to acknowledge that a method can fail. In practice the distinction has proven difficult to apply consistently, and most modern Java frameworks have abandoned checked exceptions for unchecked.

### When to Use Checked Exceptions

A checked exception is appropriate when **all three** of these conditions hold:

1. The failure is an operational error, not a programming error.
2. The caller can realistically recover from it at the point of call (not just propagate it up).
3. The exception is part of the public contract of the method — a caller who does not handle it has written incomplete code.

The canonical example from the JDK is `IOException` on file operations. A caller opening a file really should think about what to do if the file does not exist.

```java
// Checked: the caller is expected to handle this
public interface ConfigLoader {
    Config load(Path configFile) throws ConfigNotFoundException;
}
```

Use checked exceptions sparingly. If you find yourself with many checked exceptions in a service API, it is usually a sign that your abstraction level is wrong.

### When to Use Unchecked Exceptions

Unchecked exceptions (`RuntimeException` and subclasses) are appropriate for:

- **Programming errors:** null arguments, out-of-range values, illegal state.
- **Infrastructure failures that the caller cannot recover from:** cannot connect to database on startup, configuration missing.
- **Situations where forcing the caller to handle is counterproductive:** the exception will propagate to a global handler anyway.

```java
// Unchecked: this is always a programming error
public BigDecimal divide(BigDecimal dividend, BigDecimal divisor) {
    if (divisor.compareTo(BigDecimal.ZERO) == 0) {
        throw new ArithmeticException("divisor must not be zero");
    }
    return dividend.divide(divisor, RoundingMode.HALF_UP);
}
```

### Why Modern Java Frameworks Use Unchecked Exceptions

Spring, JPA, and most modern Java frameworks use unchecked exceptions almost exclusively. The reasons are pragmatic:

1. **Checked exceptions do not compose well with lambdas and streams.** `Function<T, R>` does not declare any checked exceptions, so you cannot throw a checked exception inside a stream pipeline without an ugly wrapper.
2. **Checked exceptions leak implementation details.** If your service layer catches `SQLException` and re-throws it as a checked exception, you have bound the interface contract to a specific persistence technology.
3. **Most callers cannot recover locally.** In a web application, nearly every exception ultimately propagates to a global exception handler that converts it to an HTTP response. Forcing every intermediate layer to declare `throws SomeCheckedException` adds noise without adding safety.

Spring's `DataAccessException` hierarchy is the clearest example: all persistence exceptions are unchecked, organized into a rich hierarchy, and translated from technology-specific exceptions into a technology-agnostic hierarchy.

### Full Java Example: Service Exception Hierarchy

```java
// -------------------------------------------------------
// Base unchecked exception for all application exceptions
// -------------------------------------------------------
public abstract class AppException extends RuntimeException {

    private final String errorCode;

    protected AppException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    protected AppException(String errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }

    public String getErrorCode() {
        return errorCode;
    }
}

// -------------------------------------------------------
// Domain layer — business rule violations
// -------------------------------------------------------
public class DomainException extends AppException {
    public DomainException(String errorCode, String message) {
        super(errorCode, message);
    }
}

// -------------------------------------------------------
// Infrastructure layer — I/O, network, external systems
// -------------------------------------------------------
public class InfrastructureException extends AppException {
    public InfrastructureException(String errorCode, String message, Throwable cause) {
        super(errorCode, message, cause);
    }
}

// -------------------------------------------------------
// Application layer — orchestration / use-case failures
// -------------------------------------------------------
public class ApplicationException extends AppException {
    public ApplicationException(String errorCode, String message) {
        super(errorCode, message);
    }

    public ApplicationException(String errorCode, String message, Throwable cause) {
        super(errorCode, message, cause);
    }
}
```

The hierarchy gives you a single catch point at each layer while preserving specificity deeper in the tree.

---

## 3. Exception Hierarchy Design

A good exception hierarchy acts as a vocabulary. It lets callers express intent ("I want to handle any payment failure") and simultaneously lets diagnostics be precise ("it was specifically a fraud decline").

### Design Principles

**Root at the right level of abstraction.** The root of a service's exception hierarchy should correspond to the service's domain, not to technical mechanisms.

**Keep the tree shallow.** Three levels is usually sufficient: base, category, specific. Deep hierarchies are hard to navigate and easy to misuse.

**Name exceptions for what happened, not how it happened.** `PaymentDeclinedException` is better than `GatewayResponseCode4023Exception`.

**Include enough information to act on.** Every exception should carry the data necessary for an alert, a log message, and a caller response without requiring a debugger.

### Full Java Example: Payment Service Exception Hierarchy

```java
// ============================================================
// ROOT: all payment domain exceptions
// ============================================================
public abstract class PaymentException extends RuntimeException {

    private final String errorCode;
    private final String paymentId;

    protected PaymentException(String errorCode, String paymentId, String message) {
        super(message);
        this.errorCode = errorCode;
        this.paymentId = paymentId;
    }

    protected PaymentException(String errorCode, String paymentId,
                               String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
        this.paymentId = paymentId;
    }

    public String getErrorCode()  { return errorCode; }
    public String getPaymentId()  { return paymentId; }
}

// ============================================================
// CATEGORY: validation failures (business rule violations)
// ============================================================
public class PaymentValidationException extends PaymentException {
    private final String fieldName;

    public PaymentValidationException(String paymentId, String fieldName, String message) {
        super("PAYMENT_VALIDATION_FAILED", paymentId, message);
        this.fieldName = fieldName;
    }

    public String getFieldName() { return fieldName; }
}

// ============================================================
// CATEGORY: processing failures (gateway / acquirer)
// ============================================================
public abstract class PaymentProcessingException extends PaymentException {
    protected PaymentProcessingException(String errorCode, String paymentId,
                                         String message, Throwable cause) {
        super(errorCode, paymentId, message, cause);
    }
}

// SPECIFIC: card was declined by the issuing bank
public class PaymentDeclinedException extends PaymentProcessingException {

    public enum DeclineReason {
        INSUFFICIENT_FUNDS,
        CARD_EXPIRED,
        FRAUD_SUSPECTED,
        DO_NOT_HONOR,
        UNKNOWN
    }

    private final DeclineReason declineReason;

    public PaymentDeclinedException(String paymentId, DeclineReason declineReason,
                                    String message) {
        super("PAYMENT_DECLINED", paymentId, message, null);
        this.declineReason = declineReason;
    }

    public DeclineReason getDeclineReason() { return declineReason; }
}

// SPECIFIC: gateway timed out or returned an unexpected error
public class PaymentGatewayException extends PaymentProcessingException {

    public PaymentGatewayException(String paymentId, String message, Throwable cause) {
        super("PAYMENT_GATEWAY_ERROR", paymentId, message, cause);
    }
}

// SPECIFIC: duplicate payment detected
public class DuplicatePaymentException extends PaymentProcessingException {

    private final String existingPaymentId;

    public DuplicatePaymentException(String paymentId, String existingPaymentId) {
        super("DUPLICATE_PAYMENT", paymentId,
              "Payment " + paymentId + " is a duplicate of " + existingPaymentId, null);
        this.existingPaymentId = existingPaymentId;
    }

    public String getExistingPaymentId() { return existingPaymentId; }
}

// ============================================================
// CATEGORY: state machine violations
// ============================================================
public class PaymentStateException extends PaymentException {

    private final String currentState;
    private final String attemptedTransition;

    public PaymentStateException(String paymentId, String currentState,
                                 String attemptedTransition) {
        super("PAYMENT_INVALID_STATE", paymentId,
              "Cannot transition payment " + paymentId + " from state " + currentState
              + " via " + attemptedTransition);
        this.currentState = currentState;
        this.attemptedTransition = attemptedTransition;
    }

    public String getCurrentState()       { return currentState; }
    public String getAttemptedTransition() { return attemptedTransition; }
}

// ============================================================
// Example usage at the service layer
// ============================================================
public class PaymentService {

    public void refund(String paymentId) {
        Payment payment = paymentRepository.findById(paymentId)
            .orElseThrow(() -> new PaymentValidationException(
                paymentId, "paymentId", "Payment not found: " + paymentId));

        if (!payment.getStatus().equals(PaymentStatus.CAPTURED)) {
            throw new PaymentStateException(
                paymentId, payment.getStatus().name(), "REFUND");
        }

        try {
            gateway.refund(paymentId, payment.getAmount());
        } catch (GatewayTimeoutException e) {
            throw new PaymentGatewayException(paymentId,
                "Gateway timed out during refund", e);
        }
    }
}
```

With this hierarchy, an HTTP exception handler can catch `PaymentDeclinedException` and return a 422 with a localized message, catch `PaymentGatewayException` and return a 502, and catch the root `PaymentException` for everything else.

---

## 4. Error Codes vs Exceptions

### When Error Codes Make Sense

Error codes — returning an integer or enum from a function to signal failure — are the right choice in a small number of situations:

- **Embedded or systems programming** where heap allocation is expensive or impossible.
- **Performance-critical inner loops** where the overhead of exception object construction and stack walking is measurable.
- **Cross-language boundaries** such as JNI calls or C library wrappers, where exceptions cannot propagate.
- **Protocol-level communication** between services (HTTP status codes, gRPC status codes). These are error codes at the boundary; within the service they are translated to exceptions.

### When Exceptions Are Better

In application-layer Java code, exceptions are almost always superior to error codes because:

- They cannot be accidentally ignored (unlike return values, which callers can forget to check).
- They carry structured context (message, cause chain, custom fields).
- They propagate automatically through the call stack.
- They separate the happy path from the error path, making both easier to read.

### Hybrid Approach: Error Codes Within Exceptions

The most pragmatic approach is to use exceptions as the transport mechanism while embedding error codes for machine-readable identification. This gives you both the propagation semantics of exceptions and the precise categorization of error codes.

```java
public enum PaymentErrorCode {
    INSUFFICIENT_FUNDS("PAY-001", "Insufficient funds"),
    CARD_EXPIRED      ("PAY-002", "Card has expired"),
    GATEWAY_TIMEOUT   ("PAY-003", "Payment gateway timed out"),
    DUPLICATE_PAYMENT ("PAY-004", "Duplicate payment detected"),
    INVALID_AMOUNT    ("PAY-005", "Payment amount is invalid");

    private final String code;
    private final String defaultMessage;

    PaymentErrorCode(String code, String defaultMessage) {
        this.code = code;
        this.defaultMessage = defaultMessage;
    }

    public String getCode()           { return code; }
    public String getDefaultMessage() { return defaultMessage; }
}

// Exception carries both the enum (for switch/matching) and the string code
// (for serialization to API responses and logs)
public class PaymentFailureException extends RuntimeException {

    private final PaymentErrorCode paymentErrorCode;

    public PaymentFailureException(PaymentErrorCode paymentErrorCode) {
        super(paymentErrorCode.getDefaultMessage());
        this.paymentErrorCode = paymentErrorCode;
    }

    public PaymentFailureException(PaymentErrorCode paymentErrorCode,
                                   String detail, Throwable cause) {
        super(paymentErrorCode.getDefaultMessage() + ": " + detail, cause);
        this.paymentErrorCode = paymentErrorCode;
    }

    public PaymentErrorCode getPaymentErrorCode() { return paymentErrorCode; }

    public String getMachineCode() { return paymentErrorCode.getCode(); }
}

// At the HTTP boundary, the machine code goes into the response body
// so clients can handle errors programmatically without parsing strings
public class ApiErrorResponse {
    private final String code;    // "PAY-001"
    private final String message; // human-readable
    private final String traceId;

    public ApiErrorResponse(String code, String message, String traceId) {
        this.code = code;
        this.message = message;
        this.traceId = traceId;
    }

    public String getCode()    { return code; }
    public String getMessage() { return message; }
    public String getTraceId() { return traceId; }
}
```

---

## 5. Result Types / Either Pattern in Java

### The Appeal of Result Types

In functional programming traditions (Haskell, Rust, Scala), functions that can fail return a `Result<T, E>` type — either a success value or an error value — rather than throwing. This makes the error path visible in the type signature and forces callers to handle both cases.

The appeal in Java:

- Errors become explicit in the type system. A caller who forgets to check for failure gets a compile error (or at least a type mismatch), not a surprise exception at runtime.
- Works well with stream pipelines and lambdas.
- Fits naturally in functional-style code where methods are composed rather than sequenced.

### Full Java Implementation of Result\<T, E\>

```java
import java.util.NoSuchElementException;
import java.util.Optional;
import java.util.function.Consumer;
import java.util.function.Function;

/**
 * Represents a computation that either succeeds with a value of type T
 * or fails with an error of type E.
 *
 * Instances are immutable; this class is safe to share across threads.
 */
public final class Result<T, E> {

    private final T value;
    private final E error;
    private final boolean success;

    private Result(T value, E error, boolean success) {
        this.value = value;
        this.error = error;
        this.success = success;
    }

    // -------------------------------------------------------
    // Factory methods
    // -------------------------------------------------------

    public static <T, E> Result<T, E> success(T value) {
        if (value == null) {
            throw new IllegalArgumentException("success value must not be null; use Optional if absence is valid");
        }
        return new Result<>(value, null, true);
    }

    public static <T, E> Result<T, E> failure(E error) {
        if (error == null) {
            throw new IllegalArgumentException("error must not be null");
        }
        return new Result<>(null, error, false);
    }

    // -------------------------------------------------------
    // Query methods
    // -------------------------------------------------------

    public boolean isSuccess() { return success; }
    public boolean isFailure() { return !success; }

    public T getValue() {
        if (!success) {
            throw new NoSuchElementException("Result is a failure; check isSuccess() before calling getValue()");
        }
        return value;
    }

    public E getError() {
        if (success) {
            throw new NoSuchElementException("Result is a success; check isFailure() before calling getError()");
        }
        return error;
    }

    public Optional<T> toOptional() {
        return success ? Optional.of(value) : Optional.empty();
    }

    // -------------------------------------------------------
    // Transformation methods (enables functional chaining)
    // -------------------------------------------------------

    /**
     * Maps the success value. If this is a failure, returns the same failure.
     */
    public <U> Result<U, E> map(Function<T, U> mapper) {
        if (success) {
            return Result.success(mapper.apply(value));
        }
        return Result.failure(error);
    }

    /**
     * Chains a second Result-producing operation. Short-circuits on failure.
     */
    public <U> Result<U, E> flatMap(Function<T, Result<U, E>> mapper) {
        if (success) {
            return mapper.apply(value);
        }
        return Result.failure(error);
    }

    /**
     * Maps the error value. If this is a success, returns the same success.
     */
    public <F> Result<T, F> mapError(Function<E, F> mapper) {
        if (!success) {
            return Result.failure(mapper.apply(error));
        }
        return Result.success(value);
    }

    /**
     * Executes the appropriate consumer depending on the outcome.
     */
    public Result<T, E> peek(Consumer<T> onSuccess, Consumer<E> onFailure) {
        if (success) {
            onSuccess.accept(value);
        } else {
            onFailure.accept(error);
        }
        return this;
    }

    /**
     * Returns the value on success, or a default value on failure.
     */
    public T getOrElse(T defaultValue) {
        return success ? value : defaultValue;
    }

    /**
     * Returns the value on success, or throws a RuntimeException wrapping the error.
     */
    public T getOrThrow(Function<E, ? extends RuntimeException> exceptionMapper) {
        if (success) {
            return value;
        }
        throw exceptionMapper.apply(error);
    }

    @Override
    public String toString() {
        return success
            ? "Result.success(" + value + ")"
            : "Result.failure(" + error + ")";
    }
}
```

### Full Java Example: Using Result for Payment Processing

```java
public enum PaymentError {
    INSUFFICIENT_FUNDS,
    CARD_EXPIRED,
    GATEWAY_UNAVAILABLE,
    DUPLICATE_PAYMENT,
    VALIDATION_FAILED
}

public class PaymentRequest {
    private final String paymentId;
    private final String cardToken;
    private final BigDecimal amount;
    private final String currency;

    public PaymentRequest(String paymentId, String cardToken,
                          BigDecimal amount, String currency) {
        this.paymentId = paymentId;
        this.cardToken = cardToken;
        this.amount = amount;
        this.currency = currency;
    }

    public String getPaymentId() { return paymentId; }
    public String getCardToken() { return cardToken; }
    public BigDecimal getAmount() { return amount; }
    public String getCurrency()  { return currency; }
}

public class ChargeReceipt {
    private final String transactionId;
    private final BigDecimal chargedAmount;

    public ChargeReceipt(String transactionId, BigDecimal chargedAmount) {
        this.transactionId = transactionId;
        this.chargedAmount = chargedAmount;
    }

    public String getTransactionId()    { return transactionId; }
    public BigDecimal getChargedAmount() { return chargedAmount; }
}

public class PaymentProcessor {

    private final PaymentGateway gateway;
    private final IdempotencyStore idempotencyStore;

    public PaymentProcessor(PaymentGateway gateway, IdempotencyStore idempotencyStore) {
        this.gateway = gateway;
        this.idempotencyStore = idempotencyStore;
    }

    /**
     * Processes a payment. Returns Result.success with a ChargeReceipt on success,
     * or Result.failure with a PaymentError on any expected failure.
     *
     * Throws IllegalArgumentException for programming errors (null inputs, etc.).
     */
    public Result<ChargeReceipt, PaymentError> processPayment(PaymentRequest request) {
        Objects.requireNonNull(request, "request must not be null");

        // Validate amount — programming error if violated
        if (request.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("amount must be positive");
        }

        // Idempotency check — operational path
        if (idempotencyStore.exists(request.getPaymentId())) {
            return Result.failure(PaymentError.DUPLICATE_PAYMENT);
        }

        // Delegate to gateway
        try {
            String transactionId = gateway.charge(
                request.getCardToken(),
                request.getAmount(),
                request.getCurrency()
            );
            ChargeReceipt receipt = new ChargeReceipt(transactionId, request.getAmount());
            idempotencyStore.record(request.getPaymentId(), transactionId);
            return Result.success(receipt);

        } catch (InsufficientFundsGatewayException e) {
            return Result.failure(PaymentError.INSUFFICIENT_FUNDS);
        } catch (CardExpiredGatewayException e) {
            return Result.failure(PaymentError.CARD_EXPIRED);
        } catch (GatewayUnavailableException e) {
            return Result.failure(PaymentError.GATEWAY_UNAVAILABLE);
        }
    }
}

// -------------------------------------------------------
// Caller: uses functional chaining
// -------------------------------------------------------
public class CheckoutService {

    private final PaymentProcessor paymentProcessor;
    private final OrderService orderService;

    public CheckoutService(PaymentProcessor paymentProcessor, OrderService orderService) {
        this.paymentProcessor = paymentProcessor;
        this.orderService = orderService;
    }

    public Result<OrderConfirmation, String> checkout(
            PaymentRequest paymentRequest, Order order) {

        return paymentProcessor.processPayment(paymentRequest)
            .flatMap(receipt -> orderService.confirm(order, receipt.getTransactionId()))
            .mapError(paymentError -> switch (paymentError) {
                case INSUFFICIENT_FUNDS  -> "Your card has insufficient funds.";
                case CARD_EXPIRED        -> "Your card has expired.";
                case GATEWAY_UNAVAILABLE -> "Payment service is temporarily unavailable.";
                case DUPLICATE_PAYMENT   -> "This payment has already been processed.";
                case VALIDATION_FAILED   -> "Payment details are invalid.";
            });
    }
}
```

### Tradeoffs

| Aspect | Result\<T, E\> | Exceptions |
|--------|----------------|-----------|
| Error visibility | Explicit in type signature | Implicit; requires reading docs |
| Composition with streams | Natural with map/flatMap | Requires try-catch wrappers |
| Call-site verbosity | Higher | Lower for happy path |
| Unexpected errors | Still use exceptions | Still use exceptions |
| Team familiarity | Lower in Java ecosystem | Higher |
| Compile-time enforcement | Yes, if E is an enum or sealed type | Only with checked exceptions |

**Comparison to `Optional<T>`:** `Optional` signals the possible absence of a value; it carries no information about why the value is absent. `Result<T, E>` carries the specific error. Use `Optional` when absence is normal and uninteresting (e.g., finding a record that may or may not exist). Use `Result` when the error type matters to the caller.

---

## 6. Fail-Fast vs Fail-Safe

### Fail-Fast

A fail-fast system detects errors at the earliest possible point and stops immediately. The philosophy: an incorrect intermediate state is worse than no result at all, because corrupted partial state can cause hard-to-diagnose failures downstream.

Fail-fast characteristics:
- Throw at the first sign of invalid input or violated invariant.
- Do not attempt to continue processing with invalid data.
- Surface errors close to their origin, making root cause obvious.
- Preferred for programming errors, data integrity violations, and any situation where continuing would produce incorrect output.

### Fail-Safe

A fail-safe system continues operating when it encounters an error, producing a degraded but still correct result. The philosophy: partial progress is better than no progress when errors are expected and isolated.

Fail-safe characteristics:
- Skip or ignore erroneous items, recording the error for later review.
- Continue processing the remaining valid inputs.
- Appropriate for batch operations, data imports, and any scenario where items are independent.

### Full Java Example: Fail-Fast vs Fail-Safe in a Data Import Pipeline

```java
public class CustomerRecord {
    private final String customerId;
    private final String email;
    private final String name;

    public CustomerRecord(String customerId, String email, String name) {
        this.customerId = customerId;
        this.email = email;
        this.name = name;
    }

    public String getCustomerId() { return customerId; }
    public String getEmail()      { return email; }
    public String getName()       { return name; }
}

public class ImportResult {
    private final int successCount;
    private final List<ImportError> errors;

    public ImportResult(int successCount, List<ImportError> errors) {
        this.successCount = successCount;
        this.errors = Collections.unmodifiableList(errors);
    }

    public int getSuccessCount()        { return successCount; }
    public List<ImportError> getErrors() { return errors; }
    public boolean hasErrors()          { return !errors.isEmpty(); }
}

public class ImportError {
    private final int lineNumber;
    private final String customerId;
    private final String reason;

    public ImportError(int lineNumber, String customerId, String reason) {
        this.lineNumber = lineNumber;
        this.customerId = customerId;
        this.reason = reason;
    }

    public int getLineNumber()  { return lineNumber; }
    public String getCustomerId() { return customerId; }
    public String getReason()   { return reason; }

    @Override
    public String toString() {
        return "line=" + lineNumber + " customerId=" + customerId + " reason=" + reason;
    }
}

public class CustomerImportService {

    private final CustomerRepository customerRepository;
    private final CustomerValidator validator;

    public CustomerImportService(CustomerRepository customerRepository,
                                  CustomerValidator validator) {
        this.customerRepository = customerRepository;
        this.validator = validator;
    }

    /**
     * FAIL-FAST strategy.
     *
     * Validates the entire batch before writing any records.
     * If any record is invalid, throws immediately without writing anything.
     * Use when atomicity is required: all records must be imported or none.
     */
    public void importFailFast(List<CustomerRecord> records) {
        Objects.requireNonNull(records, "records must not be null");
        if (records.isEmpty()) {
            return;
        }

        // Validate all records first — fail before touching the database
        List<ImportError> validationErrors = new ArrayList<>();
        for (int i = 0; i < records.size(); i++) {
            CustomerRecord record = records.get(i);
            try {
                validator.validate(record);
            } catch (ValidationException e) {
                validationErrors.add(new ImportError(i + 1, record.getCustomerId(), e.getMessage()));
            }
        }

        if (!validationErrors.isEmpty()) {
            // Fail fast: report all validation errors at once, write nothing
            String summary = validationErrors.stream()
                .map(ImportError::toString)
                .collect(Collectors.joining("\n  ", "Import failed with validation errors:\n  ", ""));
            throw new BatchImportException(summary, validationErrors);
        }

        // All valid — write atomically
        customerRepository.saveAll(records);
    }

    /**
     * FAIL-SAFE strategy.
     *
     * Processes each record independently. Invalid or failed records are skipped
     * and recorded in the result; valid records are written regardless.
     * Use when partial success is acceptable and records are independent.
     */
    public ImportResult importFailSafe(List<CustomerRecord> records) {
        Objects.requireNonNull(records, "records must not be null");

        List<ImportError> errors = new ArrayList<>();
        int successCount = 0;

        for (int i = 0; i < records.size(); i++) {
            CustomerRecord record = records.get(i);
            int lineNumber = i + 1;

            try {
                validator.validate(record);
                customerRepository.save(record);
                successCount++;
            } catch (ValidationException e) {
                errors.add(new ImportError(lineNumber, record.getCustomerId(),
                    "Validation failed: " + e.getMessage()));
            } catch (DuplicateKeyException e) {
                errors.add(new ImportError(lineNumber, record.getCustomerId(),
                    "Duplicate customer ID — skipped"));
            } catch (DataAccessException e) {
                errors.add(new ImportError(lineNumber, record.getCustomerId(),
                    "Database error: " + e.getMessage()));
            }
            // Any other RuntimeException propagates — infra failure is not fail-safe
        }

        return new ImportResult(successCount, errors);
    }
}
```

**When to choose each:**

| Situation | Strategy | Reason |
|-----------|----------|--------|
| Financial data import where partial writes would corrupt accounts | Fail-fast | Atomicity is mandatory |
| Bulk email list import where each record is independent | Fail-safe | Partial success is better than total rejection |
| Configuration loading on startup | Fail-fast | Continuing with bad config causes obscure failures later |
| Event processing pipeline with millions of events | Fail-safe | One bad event should not stop the pipeline |
| Test assertions | Fail-fast | Surface all failures in one run (but this is a different axis) |

---

## 7. Error Propagation Strategy

### The Key Decision: Bubble vs Catch-and-Rethrow

**Bubbling** is appropriate when the current layer has no additional context to add and no action to take. The exception propagates naturally to a layer that can act on it. Over-catching — catching exceptions just to let them pass through — is noise.

**Catch-and-rethrow (translation)** is appropriate when:
- You are crossing an architectural boundary (e.g., infrastructure to domain).
- The original exception is from a third-party library whose type should not escape your API.
- You can add meaningful context that is not present in the original exception.

### When to Translate Exceptions

The rule of thumb: exceptions should be expressed in the vocabulary of the layer that throws them.

A `CustomerRepository` implemented with JPA should not let `JPA PersistenceException` escape into the service layer. The service layer should receive `CustomerNotFoundException` or `CustomerPersistenceException` — exceptions that make sense in business terms.

### Preserving the Original Cause

Always preserve the original cause when translating exceptions. Losing the original exception destroys the stack trace that contains the root cause. Java provides `initCause` and the standard `Throwable(String message, Throwable cause)` constructor for this.

### Full Java Example: Layered Exception Translation

```java
// -------------------------------------------------------
// Infrastructure layer: wraps JPA / JDBC exceptions
// -------------------------------------------------------
@Repository
public class JpaCustomerRepository implements CustomerRepository {

    private final EntityManager em;

    public JpaCustomerRepository(EntityManager em) {
        this.em = em;
    }

    @Override
    public Customer findById(String customerId) {
        try {
            CustomerEntity entity = em.find(CustomerEntity.class, customerId);
            if (entity == null) {
                // Not a JPA exception — this is our own domain exception
                throw new CustomerNotFoundException(customerId);
            }
            return toModel(entity);
        } catch (PersistenceException e) {
            // JPA exception translated to domain infrastructure exception.
            // The original JPA exception is preserved as the cause.
            throw new CustomerDataAccessException(
                "Failed to load customer: " + customerId, e);
        }
    }

    @Override
    public void save(Customer customer) {
        try {
            CustomerEntity entity = toEntity(customer);
            em.persist(entity);
        } catch (EntityExistsException e) {
            // Specific JPA case — translate to meaningful domain exception
            throw new DuplicateCustomerException(customer.getCustomerId(), e);
        } catch (PersistenceException e) {
            throw new CustomerDataAccessException(
                "Failed to save customer: " + customer.getCustomerId(), e);
        }
    }

    private Customer toModel(CustomerEntity entity) { /* mapping */ return null; }
    private CustomerEntity toEntity(Customer customer) { /* mapping */ return null; }
}

// -------------------------------------------------------
// Domain exceptions (no JPA in the type hierarchy)
// -------------------------------------------------------
public class CustomerNotFoundException extends DomainException {
    public CustomerNotFoundException(String customerId) {
        super("CUSTOMER_NOT_FOUND", "Customer not found: " + customerId);
    }
}

public class DuplicateCustomerException extends DomainException {
    public DuplicateCustomerException(String customerId, Throwable cause) {
        super("DUPLICATE_CUSTOMER", "Customer already exists: " + customerId);
        initCause(cause); // preserve the original JPA stack trace
    }
}

public class CustomerDataAccessException extends InfrastructureException {
    public CustomerDataAccessException(String message, Throwable cause) {
        super("CUSTOMER_DATA_ACCESS_ERROR", message, cause);
    }
}

// -------------------------------------------------------
// Service layer: adds business context when rethrowing
// -------------------------------------------------------
@Service
public class CustomerService {

    private final CustomerRepository customerRepository;
    private static final Logger log = LoggerFactory.getLogger(CustomerService.class);

    public CustomerService(CustomerRepository customerRepository) {
        this.customerRepository = customerRepository;
    }

    public Customer getCustomerForOrder(String customerId, String orderId) {
        try {
            return customerRepository.findById(customerId);
        } catch (CustomerNotFoundException e) {
            // Add business context: we know WHY we were looking this customer up
            log.warn("Customer {} not found while processing order {}",
                     customerId, orderId);
            // Re-throw as-is — the exception already says enough
            throw e;
        } catch (CustomerDataAccessException e) {
            // Infrastructure failure — wrap with business context and re-throw
            // The original exception (and its cause chain) is preserved
            throw new ApplicationException(
                "ORDER_CUSTOMER_LOAD_FAILED",
                "Could not load customer " + customerId + " for order " + orderId,
                e  // <-- the CustomerDataAccessException, which wraps the JPA exception
            );
        }
    }
}
```

The resulting cause chain when a JPA error occurs:

```
ApplicationException: Could not load customer C123 for order O456
  caused by: CustomerDataAccessException: Failed to load customer: C123
    caused by: PersistenceException: JDBC connection refused
      caused by: java.net.ConnectException: Connection refused
```

Every layer is represented. A developer reading the stack trace knows exactly where the failure crossed each boundary.

---

## 8. Wrapping Third-Party Exceptions

### Why Third-Party Exceptions Must Not Escape Your Boundary

If `AmazonS3Exception` (from the AWS SDK) escapes your `StorageService` interface, you have:

1. Leaked an implementation detail into all callers.
2. Made it impossible to switch from S3 to GCS without changing every caller.
3. Forced every caller to depend on the AWS SDK at compile time.
4. Committed to the AWS SDK's exception hierarchy — if AWS changes it, your code breaks everywhere.

This is the **anti-corruption layer** principle applied to exceptions. Your service boundary defines its own exception vocabulary, and everything inside the boundary translates external exceptions before they escape.

### Full Java Example: Wrapping AWS SDK Exceptions

```java
// -------------------------------------------------------
// Your storage abstraction — no AWS types
// -------------------------------------------------------
public interface ObjectStorageService {
    void upload(String bucket, String key, byte[] data);
    byte[] download(String bucket, String key);
    void delete(String bucket, String key);
}

// -------------------------------------------------------
// Your exception vocabulary
// -------------------------------------------------------
public class StorageException extends InfrastructureException {
    public StorageException(String message, Throwable cause) {
        super("STORAGE_ERROR", message, cause);
    }
}

public class StorageObjectNotFoundException extends StorageException {
    private final String bucket;
    private final String key;

    public StorageObjectNotFoundException(String bucket, String key, Throwable cause) {
        super("Object not found: " + bucket + "/" + key, cause);
        this.bucket = bucket;
        this.key = key;
    }

    public String getBucket() { return bucket; }
    public String getKey()    { return key; }
}

public class StorageAccessDeniedException extends StorageException {
    public StorageAccessDeniedException(String bucket, String key, Throwable cause) {
        super("Access denied for: " + bucket + "/" + key, cause);
    }
}

public class StorageCapacityException extends StorageException {
    public StorageCapacityException(String message, Throwable cause) {
        super(message, cause);
    }
}

// -------------------------------------------------------
// AWS S3 implementation: all AWS exceptions stop here
// -------------------------------------------------------
public class S3ObjectStorageService implements ObjectStorageService {

    private final S3Client s3Client;
    private static final Logger log = LoggerFactory.getLogger(S3ObjectStorageService.class);

    public S3ObjectStorageService(S3Client s3Client) {
        this.s3Client = s3Client;
    }

    @Override
    public void upload(String bucket, String key, byte[] data) {
        try {
            PutObjectRequest request = PutObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .build();
            s3Client.putObject(request, RequestBody.fromBytes(data));
        } catch (S3Exception e) {
            // Translate before the AWS exception escapes
            throw translateS3Exception(bucket, key, "upload", e);
        }
    }

    @Override
    public byte[] download(String bucket, String key) {
        try {
            GetObjectRequest request = GetObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .build();
            return s3Client.getObjectAsBytes(request).asByteArray();
        } catch (S3Exception e) {
            throw translateS3Exception(bucket, key, "download", e);
        }
    }

    @Override
    public void delete(String bucket, String key) {
        try {
            DeleteObjectRequest request = DeleteObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .build();
            s3Client.deleteObject(request);
        } catch (S3Exception e) {
            throw translateS3Exception(bucket, key, "delete", e);
        }
    }

    /**
     * Translates AWS SDK exceptions into our storage vocabulary.
     * The original AWS exception is always preserved as the cause.
     */
    private StorageException translateS3Exception(String bucket, String key,
                                                   String operation, S3Exception e) {
        int statusCode = e.statusCode();

        log.error("S3 {} failed for {}/{}: status={} awsCode={}",
                  operation, bucket, key, statusCode, e.awsErrorDetails().errorCode());

        return switch (statusCode) {
            case 404 -> new StorageObjectNotFoundException(bucket, key, e);
            case 403, 401 -> new StorageAccessDeniedException(bucket, key, e);
            case 507 -> new StorageCapacityException(
                "Storage capacity exceeded for bucket: " + bucket, e);
            default -> new StorageException(
                "S3 " + operation + " failed for " + bucket + "/" + key
                + " (HTTP " + statusCode + ")", e);
        };
    }
}
```

Any caller of `ObjectStorageService` sees only `StorageException` and its subclasses. The AWS SDK is entirely hidden. Switching to Google Cloud Storage means implementing a new class and changing one dependency injection binding — not modifying callers.

---

# Part 2: Resilience Patterns at the Code Level

## 9. Retry with Exponential Backoff

### When to Retry

Not every failure is worth retrying. Retry only for **transient failures** — errors that are expected to resolve themselves if you wait and try again:

- Network timeouts
- HTTP 429 (rate limited) and 503 (service unavailable)
- Database connection pool exhaustion (briefly)
- Temporary file system errors

Do **not** retry:

- HTTP 400 (bad request) — retrying an invalid request will always fail.
- HTTP 401 / 403 (auth errors) — retrying does not fix permissions.
- Business rule violations — the error is deterministic.
- HTTP 404 — the resource does not exist.

### Retry with Jitter

Exponential backoff without jitter causes **thundering herd**: if many clients all fail at the same moment and all wait exactly 1s, then 2s, then 4s, they all retry simultaneously and flood the recovering server. Adding random jitter spreads retries across a window, smoothing load.

**Full decorrelated jitter formula:** `sleep = random_between(base, previous_sleep * 3)` — this is one of several jitter strategies; "full jitter" (`sleep = random_between(0, cap)`) is another. The key is randomness.

### Full Java Implementation from Scratch

```java
import java.util.concurrent.Callable;
import java.util.concurrent.ThreadLocalRandom;
import java.util.function.Predicate;

/**
 * Executes a callable with exponential backoff and optional jitter.
 *
 * This implementation is deliberately dependency-free so it can be
 * used in any context including integration tests.
 */
public class RetryExecutor {

    private final int maxAttempts;
    private final long initialDelayMillis;
    private final long maxDelayMillis;
    private final double backoffMultiplier;
    private final boolean jitterEnabled;
    private final Predicate<Exception> retryPredicate;
    private final Sleeper sleeper; // injectable for testing

    private RetryExecutor(Builder builder) {
        this.maxAttempts = builder.maxAttempts;
        this.initialDelayMillis = builder.initialDelayMillis;
        this.maxDelayMillis = builder.maxDelayMillis;
        this.backoffMultiplier = builder.backoffMultiplier;
        this.jitterEnabled = builder.jitterEnabled;
        this.retryPredicate = builder.retryPredicate;
        this.sleeper = builder.sleeper;
    }

    /**
     * Executes the operation, retrying on failure according to configuration.
     *
     * @param operation the callable to execute
     * @param <T>       return type
     * @return          the result of a successful invocation
     * @throws RetryExhaustedException if all attempts fail
     */
    public <T> T execute(Callable<T> operation) {
        Exception lastException = null;
        long delayMillis = initialDelayMillis;

        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                return operation.call();
            } catch (Exception e) {
                lastException = e;

                if (!retryPredicate.test(e)) {
                    // Non-transient failure — do not retry
                    throw new RetryAbortedException(
                        "Non-retryable exception on attempt " + attempt, e);
                }

                if (attempt == maxAttempts) {
                    // Exhausted all attempts
                    break;
                }

                long actualDelay = jitterEnabled ? applyJitter(delayMillis) : delayMillis;

                try {
                    sleeper.sleep(actualDelay);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RetryAbortedException("Retry interrupted", ie);
                }

                // Compute next delay with cap
                delayMillis = Math.min((long) (delayMillis * backoffMultiplier), maxDelayMillis);
            }
        }

        throw new RetryExhaustedException(
            "Operation failed after " + maxAttempts + " attempts", lastException);
    }

    /**
     * Applies full jitter: returns a random value in [initialDelayMillis, delayMillis].
     * This spreads retries across a window rather than synchronizing them.
     */
    private long applyJitter(long delayMillis) {
        long low = Math.min(initialDelayMillis, delayMillis);
        return low + (long) (ThreadLocalRandom.current().nextDouble() * (delayMillis - low));
    }

    // -------------------------------------------------------
    // Exceptions
    // -------------------------------------------------------

    public static class RetryExhaustedException extends RuntimeException {
        public RetryExhaustedException(String message, Throwable cause) {
            super(message, cause);
        }
    }

    public static class RetryAbortedException extends RuntimeException {
        public RetryAbortedException(String message, Throwable cause) {
            super(message, cause);
        }
    }

    // -------------------------------------------------------
    // Sleeper interface — injectable for testing
    // -------------------------------------------------------

    @FunctionalInterface
    public interface Sleeper {
        void sleep(long millis) throws InterruptedException;

        static Sleeper real() {
            return Thread::sleep;
        }

        static Sleeper noOp() {
            return millis -> {};
        }
    }

    // -------------------------------------------------------
    // Builder
    // -------------------------------------------------------

    public static Builder builder() {
        return new Builder();
    }

    public static final class Builder {
        private int maxAttempts = 3;
        private long initialDelayMillis = 100;
        private long maxDelayMillis = 30_000;
        private double backoffMultiplier = 2.0;
        private boolean jitterEnabled = true;
        private Predicate<Exception> retryPredicate = e -> true; // retry everything by default
        private Sleeper sleeper = Sleeper.real();

        public Builder maxAttempts(int maxAttempts) {
            if (maxAttempts < 1) throw new IllegalArgumentException("maxAttempts must be >= 1");
            this.maxAttempts = maxAttempts;
            return this;
        }

        public Builder initialDelayMillis(long initialDelayMillis) {
            this.initialDelayMillis = initialDelayMillis;
            return this;
        }

        public Builder maxDelayMillis(long maxDelayMillis) {
            this.maxDelayMillis = maxDelayMillis;
            return this;
        }

        public Builder backoffMultiplier(double backoffMultiplier) {
            this.backoffMultiplier = backoffMultiplier;
            return this;
        }

        public Builder withJitter(boolean jitterEnabled) {
            this.jitterEnabled = jitterEnabled;
            return this;
        }

        /** Only retry when this predicate returns true for the thrown exception. */
        public Builder retryOn(Predicate<Exception> retryPredicate) {
            this.retryPredicate = retryPredicate;
            return this;
        }

        /** Retry only on the specified exception types. */
        @SafeVarargs
        public final Builder retryOn(Class<? extends Exception>... exceptionTypes) {
            this.retryPredicate = e -> {
                for (Class<? extends Exception> type : exceptionTypes) {
                    if (type.isInstance(e)) return true;
                }
                return false;
            };
            return this;
        }

        public Builder sleeper(Sleeper sleeper) {
            this.sleeper = sleeper;
            return this;
        }

        public RetryExecutor build() {
            return new RetryExecutor(this);
        }
    }
}

// -------------------------------------------------------
// Usage example
// -------------------------------------------------------
public class NotificationService {

    private final EmailGateway emailGateway;
    private final RetryExecutor retryExecutor;

    public NotificationService(EmailGateway emailGateway) {
        this.emailGateway = emailGateway;
        this.retryExecutor = RetryExecutor.builder()
            .maxAttempts(4)
            .initialDelayMillis(200)
            .maxDelayMillis(10_000)
            .backoffMultiplier(2.0)
            .withJitter(true)
            .retryOn(TransientEmailGatewayException.class)
            .build();
    }

    public void sendWelcomeEmail(String userId, String emailAddress) {
        try {
            retryExecutor.execute(() -> {
                emailGateway.send(emailAddress, "Welcome!", buildWelcomeBody(userId));
                return null;
            });
        } catch (RetryExecutor.RetryExhaustedException e) {
            throw new NotificationFailedException(
                "Failed to send welcome email to " + emailAddress + " after all retries", e);
        }
    }

    private String buildWelcomeBody(String userId) { return "Welcome, " + userId; }
}
```

---

## 10. Circuit Breaker Pattern

### The Problem Without Circuit Breakers

Without a circuit breaker, when a downstream service is down:
1. Each call times out after (say) 30 seconds.
2. Threads accumulate waiting for those timeouts.
3. The thread pool exhausts, and the calling service becomes unresponsive.
4. The problem cascades to callers of the calling service.

### States: CLOSED, OPEN, HALF-OPEN

**CLOSED:** Normal operation. Calls pass through. Failures are counted.

**OPEN:** Downstream is failing. Calls are rejected immediately (fail-fast) without touching the downstream service. After a configured wait period, transitions to HALF-OPEN.

**HALF-OPEN:** Probe state. A limited number of calls are allowed through. If they succeed, transition back to CLOSED. If they fail, return to OPEN.

### Full Java Implementation

```java
import java.time.Clock;
import java.time.Duration;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.atomic.AtomicReference;
import java.util.function.Supplier;

/**
 * Thread-safe circuit breaker implementation.
 *
 * Counts failures in a fixed window. Opens when the failure count exceeds
 * the threshold. Transitions to HALF-OPEN after a wait duration.
 */
public class CircuitBreaker {

    public enum State { CLOSED, OPEN, HALF_OPEN }

    // Configuration
    private final String name;
    private final int failureThreshold;       // open after this many failures
    private final int halfOpenProbeCount;     // allow this many calls in HALF-OPEN
    private final Duration openTimeout;       // wait before probing in HALF-OPEN
    private final Clock clock;

    // State
    private final AtomicReference<State> state = new AtomicReference<>(State.CLOSED);
    private final AtomicInteger failureCount  = new AtomicInteger(0);
    private final AtomicInteger successCount  = new AtomicInteger(0);
    private final AtomicInteger probeCount    = new AtomicInteger(0);
    private final AtomicLong openedAt         = new AtomicLong(0);

    public CircuitBreaker(String name, int failureThreshold,
                          int halfOpenProbeCount, Duration openTimeout) {
        this(name, failureThreshold, halfOpenProbeCount, openTimeout, Clock.systemUTC());
    }

    // Package-private for testing with a controllable clock
    CircuitBreaker(String name, int failureThreshold, int halfOpenProbeCount,
                   Duration openTimeout, Clock clock) {
        this.name = name;
        this.failureThreshold = failureThreshold;
        this.halfOpenProbeCount = halfOpenProbeCount;
        this.openTimeout = openTimeout;
        this.clock = clock;
    }

    /**
     * Executes the supplier if the circuit is CLOSED or HALF-OPEN (probing).
     * Throws CircuitBreakerOpenException if the circuit is OPEN.
     */
    public <T> T execute(Supplier<T> operation) {
        State currentState = evaluateState();

        if (currentState == State.OPEN) {
            throw new CircuitBreakerOpenException(
                "Circuit breaker '" + name + "' is OPEN — calls are blocked");
        }

        try {
            T result = operation.get();
            recordSuccess(currentState);
            return result;
        } catch (Exception e) {
            recordFailure(currentState);
            throw e;
        }
    }

    /**
     * Evaluates the current state, handling the OPEN -> HALF-OPEN transition.
     */
    private State evaluateState() {
        State current = state.get();

        if (current == State.OPEN) {
            long openedAtMillis = openedAt.get();
            long nowMillis = clock.millis();
            if (nowMillis - openedAtMillis >= openTimeout.toMillis()) {
                // Attempt transition from OPEN to HALF-OPEN
                if (state.compareAndSet(State.OPEN, State.HALF_OPEN)) {
                    probeCount.set(0);
                    successCount.set(0);
                    failureCount.set(0);
                    return State.HALF_OPEN;
                }
                // Another thread already transitioned — re-read
                return state.get();
            }
        }

        return current;
    }

    private void recordSuccess(State stateAtCallTime) {
        if (stateAtCallTime == State.HALF_OPEN) {
            int probes = probeCount.incrementAndGet();
            int successes = successCount.incrementAndGet();

            if (probes >= halfOpenProbeCount && successes == probes) {
                // All probes succeeded — close the circuit
                transitionToClosed();
            }
        } else {
            // CLOSED: reset failure count on success
            failureCount.set(0);
        }
    }

    private void recordFailure(State stateAtCallTime) {
        if (stateAtCallTime == State.HALF_OPEN) {
            // Any failure in HALF-OPEN reopens the circuit
            transitionToOpen();
        } else {
            // CLOSED: increment failure count and check threshold
            int failures = failureCount.incrementAndGet();
            if (failures >= failureThreshold) {
                transitionToOpen();
            }
        }
    }

    private void transitionToOpen() {
        if (state.compareAndSet(State.CLOSED, State.OPEN)
                || state.compareAndSet(State.HALF_OPEN, State.OPEN)) {
            openedAt.set(clock.millis());
            failureCount.set(0);
        }
    }

    private void transitionToClosed() {
        state.set(State.CLOSED);
        failureCount.set(0);
        successCount.set(0);
        probeCount.set(0);
    }

    public State getState() { return state.get(); }
    public String getName()  { return name; }

    public static class CircuitBreakerOpenException extends RuntimeException {
        public CircuitBreakerOpenException(String message) {
            super(message);
        }
    }
}

// -------------------------------------------------------
// Integration: wrapping a service call
// -------------------------------------------------------
public class PaymentGatewayClient {

    private final RawPaymentGateway rawGateway;
    private final CircuitBreaker circuitBreaker;
    private static final Logger log = LoggerFactory.getLogger(PaymentGatewayClient.class);

    public PaymentGatewayClient(RawPaymentGateway rawGateway) {
        this.rawGateway = rawGateway;
        this.circuitBreaker = new CircuitBreaker(
            "payment-gateway",
            5,                        // open after 5 consecutive failures
            3,                        // probe with 3 calls in HALF-OPEN
            Duration.ofSeconds(30)    // wait 30 seconds before probing
        );
    }

    public ChargeResponse charge(ChargeRequest request) {
        try {
            return circuitBreaker.execute(() -> rawGateway.charge(request));
        } catch (CircuitBreaker.CircuitBreakerOpenException e) {
            log.warn("Circuit breaker open — payment gateway unavailable");
            throw new PaymentGatewayUnavailableException(
                "Payment gateway circuit breaker is open", e);
        } catch (Exception e) {
            log.error("Payment gateway call failed", e);
            throw new PaymentGatewayException("Gateway call failed", e);
        }
    }
}
```

---

## 11. Bulkhead Pattern

### The Problem Without Bulkheads

If all operations in a service share a single thread pool or connection pool, one slow dependency can exhaust the pool. Threads blocked waiting for the slow dependency leave no threads available for fast, healthy operations. The entire service becomes unresponsive.

The bulkhead pattern (named after the watertight compartments in ships) isolates resource pools by concern. A slow payment gateway cannot starve the thread pool serving health checks or order lookups.

### Full Java Example: BulkheadExecutor

```java
import java.util.concurrent.*;
import java.util.function.Supplier;

/**
 * A bulkhead that limits concurrent access to a resource using a dedicated,
 * bounded thread pool.
 *
 * Operations that cannot be submitted (because the pool queue is full) fail
 * immediately with BulkheadRejectedException rather than blocking indefinitely.
 */
public class BulkheadExecutor {

    private final String name;
    private final ExecutorService executor;
    private final int maxConcurrentCalls;
    private final long timeoutMillis;

    /**
     * @param name               identifier for logging and metrics
     * @param maxConcurrentCalls maximum simultaneously executing calls
     * @param queueCapacity      maximum number of calls waiting to execute
     * @param timeoutMillis      how long to wait for a result before aborting
     */
    public BulkheadExecutor(String name, int maxConcurrentCalls,
                            int queueCapacity, long timeoutMillis) {
        this.name = name;
        this.maxConcurrentCalls = maxConcurrentCalls;
        this.timeoutMillis = timeoutMillis;

        this.executor = new ThreadPoolExecutor(
            maxConcurrentCalls,         // core pool size
            maxConcurrentCalls,         // max pool size (fixed — no unbounded growth)
            0L, TimeUnit.MILLISECONDS,
            new ArrayBlockingQueue<>(queueCapacity),  // bounded queue
            new ThreadFactory() {
                private final AtomicInteger count = new AtomicInteger(0);
                @Override
                public Thread newThread(Runnable r) {
                    Thread t = new Thread(r, name + "-bulkhead-" + count.incrementAndGet());
                    t.setDaemon(true);
                    return t;
                }
            },
            new ThreadPoolExecutor.AbortPolicy() // reject when full
        );
    }

    /**
     * Executes the operation within the bulkhead.
     *
     * @throws BulkheadRejectedException if the pool is at capacity
     * @throws BulkheadTimeoutException  if the operation exceeds the timeout
     */
    public <T> T execute(Supplier<T> operation) {
        Future<T> future;
        try {
            future = executor.submit(operation::get);
        } catch (RejectedExecutionException e) {
            throw new BulkheadRejectedException(
                "Bulkhead '" + name + "' is at capacity — call rejected");
        }

        try {
            return future.get(timeoutMillis, TimeUnit.MILLISECONDS);
        } catch (TimeoutException e) {
            future.cancel(true);
            throw new BulkheadTimeoutException(
                "Bulkhead '" + name + "' timed out after " + timeoutMillis + "ms");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            future.cancel(true);
            throw new BulkheadRejectedException("Bulkhead '" + name + "' interrupted", e);
        } catch (ExecutionException e) {
            Throwable cause = e.getCause();
            if (cause instanceof RuntimeException) throw (RuntimeException) cause;
            throw new RuntimeException("Unexpected checked exception in bulkhead", cause);
        }
    }

    public void shutdown() {
        executor.shutdown();
    }

    // -------------------------------------------------------
    // Exceptions
    // -------------------------------------------------------

    public static class BulkheadRejectedException extends RuntimeException {
        public BulkheadRejectedException(String message) { super(message); }
        public BulkheadRejectedException(String message, Throwable cause) { super(message, cause); }
    }

    public static class BulkheadTimeoutException extends RuntimeException {
        public BulkheadTimeoutException(String message) { super(message); }
    }
}

// -------------------------------------------------------
// Wiring multiple bulkheads in a service
// -------------------------------------------------------
@Configuration
public class BulkheadConfig {

    @Bean
    public BulkheadExecutor paymentGatewayBulkhead() {
        return new BulkheadExecutor(
            "payment-gateway",
            10,        // max 10 concurrent payment gateway calls
            20,        // up to 20 calls can wait in the queue
            5_000      // 5-second timeout
        );
    }

    @Bean
    public BulkheadExecutor fraudCheckBulkhead() {
        return new BulkheadExecutor(
            "fraud-check",
            5,         // fraud check is slower; limit to 5 concurrent calls
            10,
            3_000
        );
    }
}

@Service
public class OrderService {

    private final BulkheadExecutor paymentBulkhead;
    private final BulkheadExecutor fraudBulkhead;
    private final PaymentGateway paymentGateway;
    private final FraudDetectionService fraudService;

    public OrderService(BulkheadExecutor paymentGatewayBulkhead,
                        BulkheadExecutor fraudCheckBulkhead,
                        PaymentGateway paymentGateway,
                        FraudDetectionService fraudService) {
        this.paymentBulkhead = paymentGatewayBulkhead;
        this.fraudBulkhead = fraudCheckBulkhead;
        this.paymentGateway = paymentGateway;
        this.fraudService = fraudService;
    }

    public OrderResult placeOrder(Order order) {
        // Fraud check in its own bulkhead — cannot starve payment operations
        FraudDecision fraudDecision = fraudBulkhead.execute(
            () -> fraudService.evaluate(order));

        if (fraudDecision.isFraudulent()) {
            throw new FraudulentOrderException(order.getOrderId());
        }

        // Payment in its own bulkhead — cannot starve fraud checks
        ChargeReceipt receipt = paymentBulkhead.execute(
            () -> paymentGateway.charge(order.getPaymentDetails(), order.getTotal()));

        return new OrderResult(order.getOrderId(), receipt.getTransactionId());
    }
}
```

---

## 12. Timeout Pattern

### Why Timeouts Are Mandatory

Every call to an external system must have a timeout. Without a timeout, a hung connection holds a thread forever, eventually exhausting the thread pool. "It will time out eventually" is not an acceptable answer — the default TCP timeout is measured in minutes, which is catastrophic for a web server.

### How to Determine the Right Timeout Value

- **P99 latency of the dependency** under normal load. Setting timeout below this will cause false failures.
- **Your own SLA.** If your endpoint must respond in 500ms and the call is the only downstream call, timeout must be well below 500ms.
- **Business requirement.** A payment that takes 10 seconds is an unacceptable user experience regardless of technical feasibility.
- **Start conservatively and tighten** based on observed latency data. Do not guess.

### Full Java Example: CompletableFuture with Timeout

```java
import java.util.concurrent.*;

public class TimeoutExample {

    private final ExecutorService executor;
    private final InventoryService inventoryService;
    private static final Logger log = LoggerFactory.getLogger(TimeoutExample.class);

    public TimeoutExample(ExecutorService executor, InventoryService inventoryService) {
        this.executor = executor;
        this.inventoryService = inventoryService;
    }

    /**
     * Checks inventory with a strict timeout. Returns empty Optional if the call
     * times out, allowing the caller to fall back.
     */
    public Optional<InventoryStatus> checkInventoryWithTimeout(
            String productId, long timeoutMillis) {

        CompletableFuture<InventoryStatus> future = CompletableFuture
            .supplyAsync(() -> inventoryService.checkStock(productId), executor)
            .orTimeout(timeoutMillis, TimeUnit.MILLISECONDS);

        try {
            return Optional.of(future.get());
        } catch (ExecutionException e) {
            Throwable cause = e.getCause();
            if (cause instanceof TimeoutException) {
                log.warn("Inventory check timed out after {}ms for product {}",
                         timeoutMillis, productId);
                return Optional.empty();
            }
            throw new InventoryServiceException(
                "Inventory check failed for product: " + productId, cause);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return Optional.empty();
        }
    }

    /**
     * Alternative: CompletableFuture.completeOnTimeout() to supply a default
     * value directly rather than returning Optional.
     */
    public InventoryStatus checkInventoryOrDefault(String productId,
                                                    InventoryStatus defaultStatus,
                                                    long timeoutMillis) {
        return CompletableFuture
            .supplyAsync(() -> inventoryService.checkStock(productId), executor)
            .completeOnTimeout(defaultStatus, timeoutMillis, TimeUnit.MILLISECONDS)
            .exceptionally(ex -> {
                log.error("Inventory check failed for product {}", productId, ex);
                return defaultStatus;
            })
            .join();
    }
}

// -------------------------------------------------------
// Timeout configuration per dependency (centralized)
// -------------------------------------------------------
public class TimeoutConfiguration {

    // These values come from configuration (application.yaml) in practice
    public static final long PAYMENT_GATEWAY_TIMEOUT_MS  = 5_000;
    public static final long FRAUD_SERVICE_TIMEOUT_MS    = 2_000;
    public static final long INVENTORY_SERVICE_TIMEOUT_MS = 800;
    public static final long NOTIFICATION_SERVICE_TIMEOUT_MS = 3_000;
    public static final long DATABASE_QUERY_TIMEOUT_MS   = 1_000;
}
```

---

## 13. Fallback Pattern

### The Idea

When the primary path fails (gateway down, timeout, circuit open), return a response from a secondary source — a cache, a pre-computed default, or a degraded version of the real answer.

A fallback should be:
- **Clearly identified** — log that a fallback was used; do not silently swap in stale data.
- **Acceptable to the user** — stale data from 5 minutes ago may be fine; data from 3 days ago may not be.
- **Clearly bounded** — define what "acceptable staleness" means for each piece of data.

### Full Java Example: PaymentService with Fallback to Cache

```java
import java.time.Duration;
import java.time.Instant;
import java.util.Optional;

public class PaymentMethodService {

    private final PaymentMethodGateway gateway;
    private final PaymentMethodCache cache;
    private final CircuitBreaker circuitBreaker;
    private static final Logger log = LoggerFactory.getLogger(PaymentMethodService.class);

    // Fallback is acceptable only if cached data is no older than this
    private static final Duration MAX_STALE_DURATION = Duration.ofMinutes(15);

    public PaymentMethodService(PaymentMethodGateway gateway,
                                 PaymentMethodCache cache) {
        this.gateway = gateway;
        this.cache = cache;
        this.circuitBreaker = new CircuitBreaker(
            "payment-method-gateway", 5, 3, Duration.ofSeconds(20));
    }

    /**
     * Returns the saved payment methods for a user.
     * Falls back to a cached copy if the gateway is unavailable.
     * Returns an empty list if no fallback is available.
     */
    public List<PaymentMethod> getPaymentMethods(String userId) {
        try {
            List<PaymentMethod> methods = circuitBreaker.execute(
                () -> gateway.fetchPaymentMethods(userId));

            // Refresh cache on successful call
            cache.put(userId, methods);
            return methods;

        } catch (CircuitBreaker.CircuitBreakerOpenException e) {
            log.warn("Circuit open — falling back to cached payment methods for user {}",
                     userId);
            return fallbackToCachedPaymentMethods(userId, "circuit-open");

        } catch (PaymentGatewayException e) {
            log.warn("Gateway error — falling back to cached payment methods for user {}",
                     userId, e);
            return fallbackToCachedPaymentMethods(userId, "gateway-error");
        }
    }

    private List<PaymentMethod> fallbackToCachedPaymentMethods(String userId,
                                                                String reason) {
        Optional<CachedEntry<List<PaymentMethod>>> cached = cache.get(userId);

        if (cached.isEmpty()) {
            log.warn("No cached payment methods available for user {} (reason={})",
                     userId, reason);
            return Collections.emptyList();
        }

        CachedEntry<List<PaymentMethod>> entry = cached.get();
        Duration staleness = Duration.between(entry.getCachedAt(), Instant.now());

        if (staleness.compareTo(MAX_STALE_DURATION) > 0) {
            log.warn("Cached payment methods for user {} are too stale ({} minutes). " +
                     "Returning empty list. reason={}", userId,
                     staleness.toMinutes(), reason);
            return Collections.emptyList();
        }

        log.info("Serving cached payment methods for user {} age={}s reason={}",
                 userId, staleness.getSeconds(), reason);
        return entry.getValue();
    }
}

// -------------------------------------------------------
// Cache abstraction
// -------------------------------------------------------
public interface PaymentMethodCache {
    void put(String userId, List<PaymentMethod> methods);
    Optional<CachedEntry<List<PaymentMethod>>> get(String userId);
}

public class CachedEntry<T> {
    private final T value;
    private final Instant cachedAt;

    public CachedEntry(T value, Instant cachedAt) {
        this.value = value;
        this.cachedAt = cachedAt;
    }

    public T getValue()      { return value; }
    public Instant getCachedAt() { return cachedAt; }
}
```

---

## 14. Graceful Degradation

### The Design Principle

Graceful degradation means the system identifies which features are **non-critical** and designs those features so they can be switched off or replaced with a cheaper substitute when their dependencies fail. Critical features — those without which the application cannot serve its primary purpose — are never degraded.

**Example:** An e-commerce site's critical path is: browse products, add to cart, checkout, receive confirmation. The recommendation engine ("customers also bought") is non-critical. If the recommendation service is unavailable, show popular items or nothing — do not fail the checkout.

### Identifying Degradable Features

Ask for each feature: "If this fails, is the user's primary goal blocked?"
- Personalized recommendations: No — degrade.
- Payment processing: Yes — do not degrade; surface the error.
- Real-time stock count: Maybe — show "check availability" instead of an exact count.
- Search autocomplete: No — fall back to full search.

### Full Java Example: Recommendation Service with Graceful Degradation

```java
public class Product {
    private final String productId;
    private final String name;
    private final BigDecimal price;

    public Product(String productId, String name, BigDecimal price) {
        this.productId = productId;
        this.name = name;
        this.price = price;
    }

    public String getProductId() { return productId; }
    public String getName()      { return name; }
    public BigDecimal getPrice() { return price; }
}

public enum RecommendationSource {
    PERSONALIZED,    // from ML model based on user history
    POPULAR_ITEMS,   // pre-computed top sellers
    EDITORIAL,       // hand-curated fallback
    EMPTY            // nothing to show
}

public class RecommendationResult {
    private final List<Product> products;
    private final RecommendationSource source;
    private final boolean degraded;

    public RecommendationResult(List<Product> products,
                                RecommendationSource source,
                                boolean degraded) {
        this.products = Collections.unmodifiableList(products);
        this.source = source;
        this.degraded = degraded;
    }

    public List<Product> getProducts() { return products; }
    public RecommendationSource getSource() { return source; }
    public boolean isDegraded()         { return degraded; }
}

@Service
public class RecommendationService {

    private final PersonalizationEngine personalizationEngine;
    private final PopularItemsCache popularItemsCache;
    private final EditorialProductList editorialList;
    private final CircuitBreaker personalizationCircuitBreaker;
    private static final Logger log = LoggerFactory.getLogger(RecommendationService.class);

    private static final int RECOMMENDATION_COUNT = 8;

    public RecommendationService(PersonalizationEngine personalizationEngine,
                                  PopularItemsCache popularItemsCache,
                                  EditorialProductList editorialList) {
        this.personalizationEngine = personalizationEngine;
        this.popularItemsCache = popularItemsCache;
        this.editorialList = editorialList;
        this.personalizationCircuitBreaker = new CircuitBreaker(
            "personalization-engine", 5, 3, Duration.ofSeconds(30));
    }

    /**
     * Returns recommendations for a user's product page.
     * Degrades through three levels:
     *   1. Personalized (best)
     *   2. Popular items (cached, no external call)
     *   3. Editorial list (static, always available)
     */
    public RecommendationResult getRecommendations(String userId, String currentProductId) {
        // Level 1: Personalized
        try {
            List<Product> personalized = personalizationCircuitBreaker.execute(
                () -> personalizationEngine.recommend(userId, currentProductId,
                                                       RECOMMENDATION_COUNT));
            if (!personalized.isEmpty()) {
                return new RecommendationResult(personalized,
                    RecommendationSource.PERSONALIZED, false);
            }
        } catch (CircuitBreaker.CircuitBreakerOpenException e) {
            log.info("Personalization circuit open — degrading to popular items. userId={}",
                     userId);
        } catch (Exception e) {
            log.warn("Personalization engine error — degrading to popular items. userId={}",
                     userId, e);
        }

        // Level 2: Popular items (from local cache, no network call)
        List<Product> popular = popularItemsCache.getTopProducts(RECOMMENDATION_COUNT);
        if (!popular.isEmpty()) {
            log.debug("Serving popular items as recommendation fallback. userId={}", userId);
            return new RecommendationResult(popular, RecommendationSource.POPULAR_ITEMS, true);
        }

        // Level 3: Editorial list (hardcoded or DB-backed, always available)
        List<Product> editorial = editorialList.getProducts(RECOMMENDATION_COUNT);
        if (!editorial.isEmpty()) {
            log.debug("Serving editorial list as recommendation fallback. userId={}", userId);
            return new RecommendationResult(editorial, RecommendationSource.EDITORIAL, true);
        }

        // Level 4: Return nothing rather than an error — this is a non-critical feature
        log.warn("All recommendation sources exhausted. Returning empty. userId={}", userId);
        return new RecommendationResult(Collections.emptyList(),
            RecommendationSource.EMPTY, true);
    }
}
```

The caller (controller) can check `result.isDegraded()` to optionally suppress the recommendations section entirely, preventing a "0 recommendations" UI state.

---

# Part 3: Observability Considerations in Design

## 15. Designing for Observability from the Start

### Observability Is Not an Afterthought

Observability means being able to understand the internal state of a system by examining its external outputs — without having to deploy new code or attach a debugger. A system designed without observability in mind is a system you cannot operate in production.

The three pillars:

| Pillar | What it answers | Primary tool |
|--------|-----------------|-------------|
| Logs | What happened? | ELK, Splunk, CloudWatch Logs |
| Metrics | How much / how often / how fast? | Prometheus, Datadog |
| Traces | Where did the time go? | Jaeger, Zipkin, AWS X-Ray |

These pillars must be connected. A trace ID that appears in a log entry and in a metric label allows you to jump from "this error rate spiked" to "here is the trace of a failing request" to "here is every log line from that request" without manual correlation.

### How Design Decisions Affect Observability

- **Large, monolithic methods** are hard to trace — you cannot tell which part took 800ms.
- **Fire-and-forget async calls without context propagation** lose the trace context.
- **Swallowed exceptions** produce no log, no metric, no trace span — they disappear.
- **Generic catch blocks** (`catch (Exception e) { log.error("Error", e); }`) produce unstructured logs that cannot be queried or alerted on specifically.

Design for observability: every logical operation should produce a span, every failure should produce a structured log event, and every boundary crossing should propagate the correlation ID.

### Correlation IDs

A correlation ID (also called a request ID or trace ID) is a unique identifier assigned to a request at the point of ingress and carried through every component that touches that request.

Without correlation IDs, debugging a multi-service transaction means grepping logs from multiple services, sorting them by timestamp, and hoping the timestamps are synchronized. With correlation IDs, you filter all logs from all services on a single value and see the complete picture.

**Implementation pattern (simplified):**

```java
// Assigned at the API gateway or the first service to receive the request
public class CorrelationIdFilter implements Filter {

    private static final String HEADER_NAME = "X-Correlation-Id";
    private static final String MDC_KEY     = "correlationId";

    @Override
    public void doFilter(ServletRequest req, ServletResponse res,
                          FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpReq = (HttpServletRequest) req;
        String correlationId = httpReq.getHeader(HEADER_NAME);

        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        // Place in MDC so every log statement in this thread automatically includes it
        MDC.put(MDC_KEY, correlationId);

        // Return the correlation ID in the response so callers can log it
        ((HttpServletResponse) res).setHeader(HEADER_NAME, correlationId);

        try {
            chain.doFilter(req, res);
        } finally {
            MDC.remove(MDC_KEY);
        }
    }
}
```

When propagating to downstream services, read the correlation ID from the MDC and add it to outbound request headers.

---

## 16. Structured Logging Design

### What to Log: Events, Not State Dumps

A log entry should describe an **event** — something that happened — not a dump of object state. A developer reading logs at 2am needs to know what happened and why, not the full contents of a DTO.

Good: `"Payment authorized. paymentId=PAY-123 amount=99.99 currency=USD gateway=stripe duration_ms=234"`

Bad: `"Processing payment: Payment{id=PAY-123, amount=99.99, currency=USD, cardToken=tok_..., customerId=C456, createdAt=2024-01-01, ...}"`

### Log Levels

| Level | When to use | Example |
|-------|-------------|---------|
| ERROR | A request failed in a way that requires attention. An alert should fire. | Payment gateway returned 500 |
| WARN | Something unexpected happened but the request succeeded (degraded path, retry needed). | Served stale data from cache; primary call failed |
| INFO | A significant business event occurred. | Payment authorized, order placed, user registered |
| DEBUG | Diagnostic detail useful during development or investigation. Not on in production. | Sending HTTP request to {url} with headers {h} |
| TRACE | Extreme detail. Almost never enabled in production. | Entering method X with parameter Y |

### What NOT to Log

- **Passwords, tokens, API keys** — obvious but worth stating.
- **Full PAN (credit card numbers)** — a PCI violation.
- **PII in uncontrolled fields** — full email addresses, SSNs, dates of birth. Log user IDs instead; correlate to PII in a separate system if needed.
- **Large payloads** — log the key identifiers and size, not the full body.

### Full Java Example: Well-Instrumented Service with SLF4J

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import java.time.Instant;

@Service
public class PaymentService {

    private static final Logger log = LoggerFactory.getLogger(PaymentService.class);

    private final PaymentRepository paymentRepository;
    private final PaymentGateway gateway;
    private final PaymentValidator validator;

    public PaymentService(PaymentRepository paymentRepository,
                          PaymentGateway gateway,
                          PaymentValidator validator) {
        this.paymentRepository = paymentRepository;
        this.gateway = gateway;
        this.validator = validator;
    }

    public PaymentResult initiatePayment(PaymentRequest request) {
        String paymentId = request.getPaymentId();

        // Structured context added to every log statement in this method scope
        MDC.put("paymentId", paymentId);
        MDC.put("customerId", request.getCustomerId());
        MDC.put("currency", request.getCurrency());

        long startNs = System.nanoTime();

        try {
            log.debug("Initiating payment. amount={}", request.getAmount());

            validator.validate(request);
            log.debug("Payment validation passed.");

            Payment payment = buildPayment(request);
            paymentRepository.save(payment);

            log.debug("Payment record persisted. status=PENDING");

            ChargeResponse chargeResponse = gateway.charge(
                request.getCardToken(),
                request.getAmount(),
                request.getCurrency()
            );

            payment.markCaptured(chargeResponse.getTransactionId());
            paymentRepository.save(payment);

            long durationMs = (System.nanoTime() - startNs) / 1_000_000;

            // INFO: significant business event — keep message terse and structured
            log.info("Payment authorized. transactionId={} durationMs={}",
                     chargeResponse.getTransactionId(), durationMs);

            return PaymentResult.success(paymentId, chargeResponse.getTransactionId());

        } catch (ValidationException e) {
            log.warn("Payment validation failed. reason={}", e.getMessage());
            return PaymentResult.failure(paymentId, "VALIDATION_FAILED", e.getMessage());

        } catch (PaymentDeclinedException e) {
            long durationMs = (System.nanoTime() - startNs) / 1_000_000;
            // WARN: decline is expected in the normal course of business
            log.warn("Payment declined. declineReason={} durationMs={}",
                     e.getDeclineReason(), durationMs);
            return PaymentResult.failure(paymentId, "DECLINED", e.getMessage());

        } catch (PaymentGatewayException e) {
            long durationMs = (System.nanoTime() - startNs) / 1_000_000;
            // ERROR: gateway failure is unexpected and requires attention
            log.error("Payment gateway error. durationMs={}", durationMs, e);
            throw e; // re-throw for global handler to surface as 502

        } finally {
            // Always clean up MDC — these keys belong to this request only
            MDC.remove("paymentId");
            MDC.remove("customerId");
            MDC.remove("currency");
        }
    }

    private Payment buildPayment(PaymentRequest request) {
        return new Payment(request.getPaymentId(), request.getCustomerId(),
                           request.getAmount(), request.getCurrency(),
                           PaymentStatus.PENDING, Instant.now());
    }
}
```

The `MDC` (Mapped Diagnostic Context) ensures that `paymentId`, `customerId`, and `currency` appear automatically in every log statement within the method — no need to include them in every `log.info(...)` call. The logging appender is configured to include MDC fields in the JSON output.

---

## 17. Metrics Instrumentation at Design Level

### What to Measure

The **USE method** (Utilization, Saturation, Errors) and **RED method** (Rate, Errors, Duration) together cover the most important dimensions:

| Dimension | What it tells you | Micrometer type |
|-----------|-------------------|-----------------|
| Request rate (throughput) | How busy is the service? | Counter |
| Error rate | What fraction of requests are failing? | Counter |
| Latency (p50, p95, p99) | How fast? Where are outliers? | Timer (Histogram) |
| Saturation | Are we near capacity? (queue depth, thread pool utilization) | Gauge |
| Downstream call metrics | Which dependency is slow / failing? | Timer + Counter per dependency |

### Instrument Types

**Counter:** Monotonically increasing. Use for counts of events. Never reset. Examples: requests served, payments processed, errors thrown.

**Gauge:** Point-in-time value. Can go up or down. Use for current state. Examples: active connections, queue depth, cache size, thread pool utilization.

**Timer (Histogram):** Records both count and duration. Automatically calculates percentiles. Use for any operation with a duration: HTTP requests, DB queries, external calls.

### Tagging Strategy

Tags (labels) allow you to slice and dice metrics. Good tags:
- `service`: the service name.
- `endpoint` or `operation`: what was called.
- `outcome`: `success`, `failure`, `timeout`.
- `dependency`: name of the external system (for client-side metrics).

Avoid high-cardinality tags like user IDs, payment IDs, or request IDs — these create millions of unique time series and will kill your metrics backend.

### Full Java Example: Instrumenting with Micrometer

```java
import io.micrometer.core.instrument.*;

@Service
public class InstrumentedPaymentService {

    private static final Logger log = LoggerFactory.getLogger(InstrumentedPaymentService.class);

    private final PaymentGateway gateway;
    private final MeterRegistry meterRegistry;

    // Pre-registered meters — instantiate once, reuse per call.
    // This is important: registering meters inside a method body on every call
    // is expensive and creates duplicate registrations.
    private final Counter paymentAttemptsCounter;
    private final Counter paymentSuccessCounter;
    private final Counter paymentFailureCounter;
    private final Timer paymentDurationTimer;
    private final Timer gatewayCallTimer;

    public InstrumentedPaymentService(PaymentGateway gateway, MeterRegistry meterRegistry) {
        this.gateway = gateway;
        this.meterRegistry = meterRegistry;

        this.paymentAttemptsCounter = Counter.builder("payments.attempts")
            .description("Total number of payment attempts")
            .tag("service", "payment-service")
            .register(meterRegistry);

        this.paymentSuccessCounter = Counter.builder("payments.results")
            .description("Payment results by outcome")
            .tag("service", "payment-service")
            .tag("outcome", "success")
            .register(meterRegistry);

        this.paymentFailureCounter = Counter.builder("payments.results")
            .description("Payment results by outcome")
            .tag("service", "payment-service")
            .tag("outcome", "failure")
            .register(meterRegistry);

        this.paymentDurationTimer = Timer.builder("payments.duration")
            .description("End-to-end payment processing duration")
            .tag("service", "payment-service")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(meterRegistry);

        this.gatewayCallTimer = Timer.builder("payments.gateway.call.duration")
            .description("Duration of calls to the payment gateway")
            .tag("service", "payment-service")
            .tag("dependency", "payment-gateway")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(meterRegistry);

        // Gauge: register once — Micrometer will poll the supplier on each scrape
        Gauge.builder("payments.gateway.circuit.state", gateway,
                      g -> encodeCircuitState(g.getCircuitState()))
            .description("Circuit breaker state (0=CLOSED, 1=HALF_OPEN, 2=OPEN)")
            .tag("dependency", "payment-gateway")
            .register(meterRegistry);
    }

    public PaymentResult processPayment(PaymentRequest request) {
        paymentAttemptsCounter.increment();

        return paymentDurationTimer.record(() -> {
            try {
                ChargeResponse response = gatewayCallTimer.record(
                    () -> gateway.charge(request.getCardToken(),
                                         request.getAmount(),
                                         request.getCurrency()));

                paymentSuccessCounter.increment();
                return PaymentResult.success(request.getPaymentId(),
                                             response.getTransactionId());

            } catch (PaymentDeclinedException e) {
                // Declined is a distinct outcome — track separately
                meterRegistry.counter("payments.results",
                    "service", "payment-service",
                    "outcome", "declined",
                    "decline_reason", e.getDeclineReason().name().toLowerCase()
                ).increment();

                return PaymentResult.failure(request.getPaymentId(),
                    "DECLINED", e.getMessage());

            } catch (Exception e) {
                paymentFailureCounter.increment();
                throw e;
            }
        });
    }

    private double encodeCircuitState(CircuitBreaker.State state) {
        return switch (state) {
            case CLOSED    -> 0.0;
            case HALF_OPEN -> 1.0;
            case OPEN      -> 2.0;
        };
    }
}
```

With this instrumentation, you can build dashboards showing:
- Payments per second (rate of `payments.attempts`)
- Success rate (`payments.results{outcome="success"}` / `payments.attempts`)
- p99 latency (`payments.duration` histogram)
- Gateway call p99 latency vs overall payment p99 — the delta reveals non-gateway overhead
- Circuit breaker state — an alert fires when the state is OPEN (value == 2)

---

## 18. Distributed Tracing Hooks

### The Problem Distributed Tracing Solves

In a microservice architecture, a single user request may touch five services. If the request takes 2 seconds, which service is responsible? Without distributed tracing you have no way to know — you have five separate log files and no correlation beyond a shared timestamp range.

Distributed tracing instruments each service to record **spans** — timed segments of work. A span knows its parent span ID, so all spans from a single request form a tree. You can visualize the tree and see exactly which service and which operation consumed the time.

### Span Creation, Propagation, Context

**Trace context** (the current trace ID, parent span ID, and sampling flag) must be:
1. **Extracted** from incoming requests (HTTP headers: `traceparent`, `X-B3-TraceId`, etc.).
2. **Activated** in the processing thread so that new spans become children of the incoming parent.
3. **Propagated** to outgoing calls as HTTP headers.
4. **Restored** when crossing thread boundaries (async calls, executor submissions).

### Full Java Example: Propagating Trace Context Through an Async Call

This example uses the OpenTelemetry API, which is the current standard.

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.*;
import io.opentelemetry.context.Context;
import io.opentelemetry.context.Scope;
import io.opentelemetry.context.propagation.TextMapPropagator;
import io.opentelemetry.context.propagation.TextMapSetter;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;

@Service
public class OrderFulfillmentService {

    private final InventoryService inventoryService;
    private final NotificationService notificationService;
    private final ExecutorService asyncExecutor;

    private final Tracer tracer = GlobalOpenTelemetry.getTracer(
        "order-fulfillment-service", "1.0.0");

    public OrderFulfillmentService(InventoryService inventoryService,
                                    NotificationService notificationService,
                                    ExecutorService asyncExecutor) {
        this.inventoryService = inventoryService;
        this.notificationService = notificationService;
        this.asyncExecutor = asyncExecutor;
    }

    /**
     * Fulfills an order synchronously and triggers an async notification.
     * The trace context is explicitly propagated to the async thread.
     */
    public FulfillmentResult fulfillOrder(Order order) {
        Span span = tracer.spanBuilder("fulfillOrder")
            .setAttribute("order.id", order.getOrderId())
            .setAttribute("order.customer_id", order.getCustomerId())
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            // Synchronous child span — automatically parented to 'span'
            reserveInventory(order);

            FulfillmentResult result = new FulfillmentResult(order.getOrderId(), "FULFILLED");

            // Capture the current context BEFORE submitting to executor.
            // Context.current() captures the current active span.
            Context currentContext = Context.current();

            // Async work — explicitly wrap with context to preserve parent span
            CompletableFuture.runAsync(
                () -> sendFulfillmentNotificationInContext(order, currentContext),
                asyncExecutor
            );

            span.setStatus(StatusCode.OK);
            return result;

        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end(); // Always end spans in finally
        }
    }

    private void reserveInventory(Order order) {
        // Child span — automatically parented because the parent is 'current'
        Span inventorySpan = tracer.spanBuilder("reserveInventory")
            .setAttribute("order.id", order.getOrderId())
            .startSpan();

        try (Scope scope = inventorySpan.makeCurrent()) {
            inventoryService.reserve(order.getOrderId(), order.getItems());
            inventorySpan.setStatus(StatusCode.OK);
        } catch (Exception e) {
            inventorySpan.recordException(e);
            inventorySpan.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            inventorySpan.end();
        }
    }

    private void sendFulfillmentNotificationInContext(Order order, Context parentContext) {
        // Make the parent context current in the async thread
        try (Scope scope = parentContext.makeCurrent()) {
            Span notifSpan = tracer.spanBuilder("sendFulfillmentNotification")
                .setAttribute("order.id", order.getOrderId())
                .startSpan();

            try (Scope notifScope = notifSpan.makeCurrent()) {
                notificationService.sendFulfillmentConfirmation(
                    order.getCustomerId(), order.getOrderId());
                notifSpan.setStatus(StatusCode.OK);
            } catch (Exception e) {
                notifSpan.recordException(e);
                notifSpan.setStatus(StatusCode.ERROR, e.getMessage());
                // Do not re-throw — this is async; the caller cannot receive the exception
                LoggerFactory.getLogger(OrderFulfillmentService.class)
                    .error("Failed to send fulfillment notification for order {}",
                           order.getOrderId(), e);
            } finally {
                notifSpan.end();
            }
        }
    }
}

// -------------------------------------------------------
// Propagating trace context in outbound HTTP calls
// -------------------------------------------------------
public class TracingAwareHttpClient {

    private final HttpClient httpClient;
    private final TextMapPropagator propagator =
        GlobalOpenTelemetry.getPropagators().getTextMapPropagator();

    public TracingAwareHttpClient(HttpClient httpClient) {
        this.httpClient = httpClient;
    }

    public HttpResponse<String> get(String url) throws IOException, InterruptedException {
        HttpRequest.Builder requestBuilder = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .GET();

        // Inject current trace context into the outbound HTTP headers
        // so the downstream service can extract it and parent its spans correctly
        propagator.inject(Context.current(), requestBuilder,
            (builder, key, value) -> builder.header(key, value));

        return httpClient.send(requestBuilder.build(), HttpResponse.BodyHandlers.ofString());
    }
}
```

The trace from a single user request now includes spans from the fulfillment service, the inventory service (a separate process), and the notification service — all linked under one trace ID that originated at the API gateway.

---

## 19. Health Check Design

### Liveness vs Readiness vs Startup

These three probe types have distinct semantics and different consequences when they fail:

**Liveness:** Is the process alive and not deadlocked? If liveness fails, Kubernetes kills and restarts the pod. Liveness checks should be minimal and fast — only check that the process is responsive, not that all dependencies are healthy. Checking a slow database in a liveness probe can cause healthy pods to be incorrectly restarted.

**Readiness:** Is the service ready to serve traffic? If readiness fails, Kubernetes removes the pod from the load balancer (but does not restart it). Check all dependencies required to serve requests: database connectivity, cache connectivity, required configuration loaded. This check can be slower.

**Startup:** Has the service finished initializing? Only used for pods with slow startup (e.g., warming a cache). Prevents readiness/liveness checks from running before the service is ready for them.

### Full Java Example: Health Check Interface with Implementations

```java
// -------------------------------------------------------
// Core abstraction
// -------------------------------------------------------
public enum HealthStatus { UP, DOWN, DEGRADED }

public class HealthCheckResult {
    private final String name;
    private final HealthStatus status;
    private final String message;
    private final long durationMs;

    private HealthCheckResult(String name, HealthStatus status,
                               String message, long durationMs) {
        this.name = name;
        this.status = status;
        this.message = message;
        this.durationMs = durationMs;
    }

    public static HealthCheckResult up(String name, long durationMs) {
        return new HealthCheckResult(name, HealthStatus.UP, "OK", durationMs);
    }

    public static HealthCheckResult down(String name, String message, long durationMs) {
        return new HealthCheckResult(name, HealthStatus.DOWN, message, durationMs);
    }

    public static HealthCheckResult degraded(String name, String message, long durationMs) {
        return new HealthCheckResult(name, HealthStatus.DEGRADED, message, durationMs);
    }

    public String getName()       { return name; }
    public HealthStatus getStatus() { return status; }
    public String getMessage()    { return message; }
    public long getDurationMs()   { return durationMs; }
    public boolean isHealthy()    { return status == HealthStatus.UP || status == HealthStatus.DEGRADED; }
}

@FunctionalInterface
public interface HealthCheck {
    HealthCheckResult check();
}

// -------------------------------------------------------
// Database health check
// -------------------------------------------------------
@Component
public class DatabaseHealthCheck implements HealthCheck {

    private final DataSource dataSource;
    private static final String VALIDATION_QUERY = "SELECT 1";
    private static final long TIMEOUT_SECONDS = 2;

    public DatabaseHealthCheck(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public HealthCheckResult check() {
        long start = System.currentTimeMillis();
        try (Connection conn = dataSource.getConnection()) {
            conn.setNetworkTimeout(Executors.newSingleThreadExecutor(),
                (int) TIMEOUT_SECONDS * 1000);
            try (PreparedStatement stmt = conn.prepareStatement(VALIDATION_QUERY)) {
                stmt.executeQuery();
            }
            return HealthCheckResult.up("database", System.currentTimeMillis() - start);
        } catch (Exception e) {
            return HealthCheckResult.down("database", e.getMessage(),
                System.currentTimeMillis() - start);
        }
    }
}

// -------------------------------------------------------
// Redis / cache health check
// -------------------------------------------------------
@Component
public class CacheHealthCheck implements HealthCheck {

    private final RedisTemplate<String, String> redisTemplate;

    public CacheHealthCheck(RedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    @Override
    public HealthCheckResult check() {
        long start = System.currentTimeMillis();
        try {
            String response = redisTemplate.getConnectionFactory()
                .getConnection()
                .ping();
            long duration = System.currentTimeMillis() - start;

            if ("PONG".equalsIgnoreCase(response)) {
                return HealthCheckResult.up("redis-cache", duration);
            }
            return HealthCheckResult.degraded("redis-cache",
                "Unexpected PING response: " + response, duration);

        } catch (Exception e) {
            return HealthCheckResult.down("redis-cache", e.getMessage(),
                System.currentTimeMillis() - start);
        }
    }
}

// -------------------------------------------------------
// Aggregator: combines all checks and determines overall health
// -------------------------------------------------------
@Component
public class HealthAggregator {

    private final List<HealthCheck> criticalChecks;  // DOWN here = DOWN overall
    private final List<HealthCheck> optionalChecks;  // DOWN here = DEGRADED overall

    public HealthAggregator(DatabaseHealthCheck dbCheck, CacheHealthCheck cacheCheck) {
        // Database is critical — if it is down, the service cannot function
        this.criticalChecks = List.of(dbCheck);
        // Cache is optional — the service can function without it (degraded)
        this.optionalChecks  = List.of(cacheCheck);
    }

    public AggregatedHealth getHealth() {
        List<HealthCheckResult> allResults = new ArrayList<>();

        boolean anyCriticalDown = false;
        for (HealthCheck check : criticalChecks) {
            HealthCheckResult result = check.check();
            allResults.add(result);
            if (result.getStatus() == HealthStatus.DOWN) {
                anyCriticalDown = true;
            }
        }

        boolean anyOptionalDown = false;
        for (HealthCheck check : optionalChecks) {
            HealthCheckResult result = check.check();
            allResults.add(result);
            if (result.getStatus() == HealthStatus.DOWN) {
                anyOptionalDown = true;
            }
        }

        HealthStatus overallStatus;
        if (anyCriticalDown) {
            overallStatus = HealthStatus.DOWN;
        } else if (anyOptionalDown) {
            overallStatus = HealthStatus.DEGRADED;
        } else {
            overallStatus = HealthStatus.UP;
        }

        return new AggregatedHealth(overallStatus, allResults);
    }
}

public class AggregatedHealth {
    private final HealthStatus status;
    private final List<HealthCheckResult> checks;

    public AggregatedHealth(HealthStatus status, List<HealthCheckResult> checks) {
        this.status = status;
        this.checks = Collections.unmodifiableList(checks);
    }

    public HealthStatus getStatus()           { return status; }
    public List<HealthCheckResult> getChecks() { return checks; }
}

// -------------------------------------------------------
// HTTP endpoints
// -------------------------------------------------------
@RestController
@RequestMapping("/internal")
public class HealthController {

    private final HealthAggregator healthAggregator;

    public HealthController(HealthAggregator healthAggregator) {
        this.healthAggregator = healthAggregator;
    }

    /**
     * Liveness: Is the process responsive?
     * Returns 200 always if the process is running.
     * Do NOT check dependencies here — a down database should not kill a live pod.
     */
    @GetMapping("/alive")
    public ResponseEntity<Map<String, String>> alive() {
        return ResponseEntity.ok(Map.of("status", "UP"));
    }

    /**
     * Readiness: Can the service handle traffic?
     * Checks all dependencies. Returns 503 if any critical dependency is down.
     */
    @GetMapping("/ready")
    public ResponseEntity<AggregatedHealth> ready() {
        AggregatedHealth health = healthAggregator.getHealth();
        int httpStatus = health.getStatus() == HealthStatus.DOWN ? 503 : 200;
        return ResponseEntity.status(httpStatus).body(health);
    }
}
```

---

## 20. Senior vs Mid-level Thinking

The table below captures the qualitative difference between how a senior engineer and a mid-level engineer approach decisions in error handling and resilience.

| Decision Area | Mid-Level Thinking | Senior Engineer Thinking |
|---------------|--------------------|--------------------------|
| Exception types | Uses `Exception` or `RuntimeException` directly | Designs a meaningful exception hierarchy; each exception carries structured context |
| When to catch | Catches broadly to avoid compile errors or to prevent crashes | Catches only what can be meaningfully handled; lets the rest propagate |
| Error messages | Generic strings like "Operation failed" | Contains the failed operation, relevant IDs, and the proximate cause |
| Third-party exceptions | Lets `AmazonS3Exception` propagate to service callers | Wraps at the infrastructure boundary; callers never see third-party types |
| Checked vs unchecked | Defaults to checked because "Java requires it" | Consciously chooses unchecked; checked only when caller can realistically recover at the call site |
| Retry logic | Ad-hoc `for` loop; no backoff; retries all errors | Exponential backoff with jitter; classifies transient vs non-transient before retrying |
| Circuit breakers | Aware of the concept; uses a library without understanding internals | Can implement a circuit breaker; configures thresholds from latency data; monitors state as a metric |
| Timeouts | Sometimes remembers to set them; uses default library timeouts | Mandatory for every external call; derived from observed p99 latency and own SLA |
| Fallbacks | "We'll add a fallback later" | Fallback strategy defined at design time; staleness bounds documented; degraded state is observable |
| Graceful degradation | Binary: works or fails | Non-critical features identified and made independently degradable from design |
| Observability | Adds `log.info` statements where the code is interesting | Designs for logs, metrics, and traces together; correlation IDs threaded through; structured log fields from day one |
| Logging | `log.error("Something went wrong: " + e)` | Structured fields; appropriate level; no PII; exception passed as second argument (never concatenated) |
| Metrics | Adds counters for "interesting" numbers | Instruments every operation with a timer; tracks success rate and latency percentiles; tags by outcome |
| Distributed tracing | Knows what traces are | Ensures context propagates across async boundaries; adds semantic attributes to spans; closes spans in finally blocks |
| Health checks | Single `/health` endpoint that hits the database | Separate liveness and readiness; critical vs optional dependencies distinguished; staleness bounds on checks |
| Failure classification | All failures treated the same | Distinguishes programming errors (bugs), transient operational errors, and permanent operational errors; each handled differently |
| Result types | Not considered | Evaluates `Result<T, E>` vs exceptions vs `Optional` deliberately based on the error model of each operation |
| Fail-fast vs fail-safe | Uses whichever was most recently familiar | Chooses explicitly based on atomicity requirements and whether items are independent |
| Exception propagation | Catches, logs, and swallows | Translates at layer boundaries; always preserves cause chain; adds context at each translation |
| Resilience testing | Tests happy path; assumes resilience "just works" | Tests circuit breaker transitions; tests retry exhaustion; injects failures in staging (chaos engineering) |
| Production incident mindset | Adds more logging after an incident | Designs so every incident produces a complete, correlated audit trail before the first incident occurs |

---

## Summary

Error handling and resilience are not features added to a design — they are the design. A senior engineer's instinct is to ask, for every operation: "What happens when this fails? How will I know it failed? What will the system do next?"

The patterns covered in this module form a coherent toolkit:

- **Exception hierarchy design** gives errors a vocabulary that callers can act on and logs can be filtered by.
- **Result types** make failure explicit in the type system where exception semantics are awkward.
- **Fail-fast** catches programming errors at the earliest possible moment; **fail-safe** tolerates operational variance in batch processing.
- **Retry with jitter** handles transient failures without amplifying load.
- **Circuit breakers** prevent cascade failures by fast-failing rather than blocking.
- **Bulkheads** prevent one slow dependency from starving all operations.
- **Timeouts** are mandatory; they are derived from SLAs and observed latency, not guessed.
- **Fallbacks** and **graceful degradation** preserve the critical path when non-critical dependencies fail.
- **Observability** — structured logs, metrics with semantic tags, and distributed traces with propagated context — enables you to understand what happened in production without a debugger.

These patterns compose. A payment gateway call should have a bulkhead (to bound concurrency), a circuit breaker (to stop calling a down service), a timeout (within the circuit breaker), a retry (for transient errors, before the circuit opens), a fallback (serve cached data if the circuit is open), and full instrumentation (a timer, a success/failure counter, and a span) — all applied in layers, each doing one thing.

The ability to design systems this way — not reactively after the first outage, but proactively from the first design review — is what distinguishes senior from mid-level engineering.
