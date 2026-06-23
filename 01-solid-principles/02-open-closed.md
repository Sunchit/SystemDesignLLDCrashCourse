# Open/Closed Principle (OCP)

## Table of Contents

1. [Definition and Origins](#1-definition-and-origins)
2. [The Cost of Modification vs. Extension](#2-the-cost-of-modification-vs-extension)
3. [Strategy Pattern as the Primary OCP Enabler](#3-strategy-pattern-as-the-primary-ocp-enabler)
4. [Complete Before/After Example: Payment Processing](#4-complete-beforeafter-example-payment-processing)
5. [Plugin Architecture Analogy](#5-plugin-architecture-analogy)
6. [Anti-Pattern: The Modifier Trap](#6-anti-pattern-the-modifier-trap)
7. [Other OCP Enablers: Template Method and Decorator](#7-other-ocp-enablers-template-method-and-decorator)
8. [Practical Limits: When Modification IS the Right Answer](#8-practical-limits-when-modification-is-the-right-answer)
9. [The "Fool Me Once" Rule](#9-the-fool-me-once-rule)
10. [Interview Questions and Model Answers](#10-interview-questions-and-model-answers)
11. [Assignment: Discount Calculation Refactoring](#11-assignment-discount-calculation-refactoring)

---

## 1. Definition and Origins

### Bertrand Meyer's Original Formulation (1988)

Bertrand Meyer introduced the Open/Closed Principle in his book *Object-Oriented Software Construction* (1988). His definition was:

> "A module is said to be open if it is still available for extension. For example, it should be possible to add fields to the data structures it contains, or new elements to the set of functions it performs. A module is said to be closed if it is available for use by other modules. This assumes that the module has been given a well-defined, stable description (the interface in the sense of information hiding)."

Meyer was working in the context of Eiffel and thinking primarily about inheritance. In his view, you extend a class by subclassing it -- you do not touch the parent. The parent is "closed" (published and stable), but the inheritance mechanism keeps it "open" (extensible via subclasses).

### Robert Martin's OOP Interpretation

Robert Martin (Uncle Bob) reframed the principle in the context of modern object-oriented design, particularly in his 1996 paper "The Open-Closed Principle" and later in *Agile Software Development: Principles, Patterns, and Practices* (2002). Martin's formulation is the one most engineers encounter today:

> "Software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification."

The key shift in Martin's interpretation: he moved away from Meyer's reliance on inheritance toward **polymorphism and abstraction**. In Martin's view, you achieve OCP not by subclassing a concrete class but by:

1. Defining an **abstraction** (interface or abstract class) that captures the variation point.
2. Writing the consuming code against that abstraction.
3. Adding new behavior by creating new **implementations** of the abstraction.

The consuming code never changes. New behavior arrives as new classes, not as modifications to existing ones.

### The Core Idea in Plain Language

When a new business requirement arrives:
- **Closed for modification**: You do not open existing, tested, production classes and change their internals.
- **Open for extension**: You add new code -- new classes, new implementations -- that slot into the existing structure without disturbing it.

This is the difference between surgery on a running system and adding a new module to a stable framework.

---

## 2. The Cost of Modification vs. Extension

### Why Modification is Dangerous

Every time you open an existing class and change its implementation, you incur costs that engineers frequently underestimate:

**1. Regression Risk**

The class being modified already works. Its existing behavior is tested and relied upon by callers. Any change -- even a "small" one -- can introduce regressions. A missed edge case, an off-by-one error in an index, an uninitialized variable: all of these are possible in any edit to existing code.

Consider a class that has been in production for two years. Its methods handle dozens of edge cases that were discovered and fixed incrementally. Every modification risks disturbing those hard-won fixes.

**2. Test Suite Disruption**

Existing unit tests cover existing behavior. When you modify a class, existing tests may break -- not because the new behavior is wrong, but because the internal structure changed in a way the tests did not anticipate. You now must update tests that were previously passing, which erodes confidence in the test suite as a safety net.

**3. Violation of Caller Assumptions**

Callers of a class make implicit assumptions about its behavior. They might rely on specific error-handling behavior, specific return values for edge cases, or specific performance characteristics. Modification can silently violate these assumptions, causing bugs that surface far from the change site and are difficult to trace.

**4. Coordination Overhead in Teams**

In a team environment, modifying a shared class creates merge conflicts, requires code reviews from engineers familiar with the original code, and demands that reviewers fully understand the side effects of the change on existing code paths. This coordination cost grows with team size.

**5. Risk in Long if/switch Chains**

The most dangerous form of modification is adding a new branch to an existing if/switch chain. The engineer must:
- Understand every existing branch.
- Ensure the new branch does not overlap with an existing one.
- Handle the default/else case correctly.
- Verify that no other part of the chain assumes the list of cases is exhaustive.

This is error-prone even for experienced engineers, and the difficulty scales with the number of existing branches.

### Why Extension is Safe

Extension -- adding new code without modifying existing code -- is safer because:

1. **Existing code does not change.** All existing tests continue to pass against unchanged code.
2. **Callers are unaffected.** They interact with the same abstraction as before.
3. **Failure is contained.** A bug in the new implementation cannot affect behavior of existing implementations.
4. **Review scope is narrow.** Code reviewers only need to understand the new code, not its interaction with everything that existed before.
5. **Rollback is clean.** If the new implementation is broken, removing it restores the system to its prior state without any cleanup of modified code.

### The Asymmetry

This asymmetry -- modification is risky, extension is safe -- is what motivates OCP as a design principle. If you design your system so that new requirements are satisfied by extension rather than modification, you convert a risky operation into a safe one.

---

## 3. Strategy Pattern as the Primary OCP Enabler

### What the Strategy Pattern Does

The Strategy pattern defines a family of algorithms (or behaviors), encapsulates each one in its own class, and makes them interchangeable. The consumer of the behavior holds a reference to the abstraction (the interface), not to any concrete implementation.

This directly enables OCP:
- Adding a new behavior = adding a new class that implements the interface. Zero modification to existing code.
- The consumer class is closed for modification with respect to new behaviors.
- The consumer is open for extension because any new implementation of the interface can be passed to it.

### Structure

```
<<interface>>
Strategy
+ execute(context): Result
    ^
    |
    +-------------------+-------------------+
    |                   |                   |
ConcreteStrategyA   ConcreteStrategyB   ConcreteStrategyC

Context
- strategy: Strategy
+ setStrategy(strategy: Strategy)
+ executeStrategy(context): Result
```

### Java Example: Sorting Strategies

This example demonstrates the raw mechanics before we move to the richer payment processing example.

```java
// The abstraction -- the variation point
public interface SortStrategy {
    void sort(int[] data);
}

// Implementation A
public class QuickSortStrategy implements SortStrategy {
    @Override
    public void sort(int[] data) {
        // quicksort implementation
        System.out.println("Sorting with QuickSort");
    }
}

// Implementation B
public class MergeSortStrategy implements SortStrategy {
    @Override
    public void sort(int[] data) {
        // mergesort implementation
        System.out.println("Sorting with MergeSort");
    }
}

// Implementation C -- added later, zero modification to above classes
public class HeapSortStrategy implements SortStrategy {
    @Override
    public void sort(int[] data) {
        // heapsort implementation
        System.out.println("Sorting with HeapSort");
    }
}

// The consumer -- closed for modification with respect to sort algorithms
public class DataProcessor {
    private final SortStrategy sortStrategy;

    public DataProcessor(SortStrategy sortStrategy) {
        this.sortStrategy = sortStrategy;
    }

    public void processData(int[] data) {
        sortStrategy.sort(data);
        // ... additional processing
    }
}

// Usage
public class Application {
    public static void main(String[] args) {
        int[] data = {5, 2, 8, 1, 9};

        // Use quicksort
        DataProcessor processor = new DataProcessor(new QuickSortStrategy());
        processor.processData(data);

        // Switch to mergesort -- no modification to DataProcessor
        DataProcessor processor2 = new DataProcessor(new MergeSortStrategy());
        processor2.processData(data);

        // HeapSort added later -- DataProcessor is untouched
        DataProcessor processor3 = new DataProcessor(new HeapSortStrategy());
        processor3.processData(data);
    }
}
```

The `DataProcessor` class has never been modified. Each new sort algorithm is a new class. The class diagram stays closed at the `DataProcessor` level and open at the `SortStrategy` level.

### Why Strategy is the Primary Enabler

The Strategy pattern is the most direct implementation of OCP because:

1. It **explicitly names the variation point** as an interface.
2. It **inverts the dependency** -- the high-level class depends on an abstraction, not on concrete logic.
3. New behavior requires **only adding a new file** -- the cleanest possible form of extension.
4. It is **composable**: a consumer can hold multiple strategies for multiple variation points.

---

## 4. Complete Before/After Example: Payment Processing

This is a realistic industry scenario. A payment processing system must handle multiple payment methods. New payment methods are added as the business expands.

### BEFORE: OCP Violation

```java
// PaymentType.java
public enum PaymentType {
    CREDIT_CARD,
    PAYPAL,
    BANK_TRANSFER
}

// PaymentRequest.java
public class PaymentRequest {
    private final PaymentType paymentType;
    private final double amount;
    private final String currency;

    // Credit card fields
    private String cardNumber;
    private String cardHolderName;
    private String expiryDate;
    private String cvv;

    // PayPal fields
    private String paypalEmail;
    private String paypalToken;

    // Bank transfer fields
    private String bankAccountNumber;
    private String routingNumber;
    private String bankName;

    public PaymentRequest(PaymentType paymentType, double amount, String currency) {
        this.paymentType = paymentType;
        this.amount = amount;
        this.currency = currency;
    }

    public PaymentType getPaymentType() { return paymentType; }
    public double getAmount() { return amount; }
    public String getCurrency() { return currency; }

    public String getCardNumber() { return cardNumber; }
    public void setCardNumber(String cardNumber) { this.cardNumber = cardNumber; }

    public String getCardHolderName() { return cardHolderName; }
    public void setCardHolderName(String cardHolderName) { this.cardHolderName = cardHolderName; }

    public String getExpiryDate() { return expiryDate; }
    public void setExpiryDate(String expiryDate) { this.expiryDate = expiryDate; }

    public String getCvv() { return cvv; }
    public void setCvv(String cvv) { this.cvv = cvv; }

    public String getPaypalEmail() { return paypalEmail; }
    public void setPaypalEmail(String paypalEmail) { this.paypalEmail = paypalEmail; }

    public String getPaypalToken() { return paypalToken; }
    public void setPaypalToken(String paypalToken) { this.paypalToken = paypalToken; }

    public String getBankAccountNumber() { return bankAccountNumber; }
    public void setBankAccountNumber(String bankAccountNumber) { this.bankAccountNumber = bankAccountNumber; }

    public String getRoutingNumber() { return routingNumber; }
    public void setRoutingNumber(String routingNumber) { this.routingNumber = routingNumber; }

    public String getBankName() { return bankName; }
    public void setBankName(String bankName) { this.bankName = bankName; }
}

// PaymentResult.java
public class PaymentResult {
    private final boolean success;
    private final String transactionId;
    private final String message;

    public PaymentResult(boolean success, String transactionId, String message) {
        this.success = success;
        this.transactionId = transactionId;
        this.message = message;
    }

    public boolean isSuccess() { return success; }
    public String getTransactionId() { return transactionId; }
    public String getMessage() { return message; }
}

// PaymentProcessor.java -- THE PROBLEM CLASS
// To add a new payment type, you MUST modify this class.
// This violates OCP.
public class PaymentProcessor {

    public PaymentResult processPayment(PaymentRequest request) {
        // Validation -- grows with every new payment type
        if (request.getPaymentType() == PaymentType.CREDIT_CARD) {
            if (request.getCardNumber() == null || request.getCardNumber().isEmpty()) {
                throw new IllegalArgumentException("Card number is required for credit card payments");
            }
            if (request.getCvv() == null || request.getCvv().isEmpty()) {
                throw new IllegalArgumentException("CVV is required for credit card payments");
            }
            if (request.getExpiryDate() == null || request.getExpiryDate().isEmpty()) {
                throw new IllegalArgumentException("Expiry date is required for credit card payments");
            }
        } else if (request.getPaymentType() == PaymentType.PAYPAL) {
            if (request.getPaypalEmail() == null || request.getPaypalEmail().isEmpty()) {
                throw new IllegalArgumentException("PayPal email is required for PayPal payments");
            }
            if (request.getPaypalToken() == null || request.getPaypalToken().isEmpty()) {
                throw new IllegalArgumentException("PayPal token is required for PayPal payments");
            }
        } else if (request.getPaymentType() == PaymentType.BANK_TRANSFER) {
            if (request.getBankAccountNumber() == null || request.getBankAccountNumber().isEmpty()) {
                throw new IllegalArgumentException("Bank account number is required for bank transfers");
            }
            if (request.getRoutingNumber() == null || request.getRoutingNumber().isEmpty()) {
                throw new IllegalArgumentException("Routing number is required for bank transfers");
            }
        } else {
            throw new IllegalArgumentException("Unsupported payment type: " + request.getPaymentType());
        }

        // Fee calculation -- grows with every new payment type
        double fee;
        if (request.getPaymentType() == PaymentType.CREDIT_CARD) {
            fee = request.getAmount() * 0.025; // 2.5% for credit card
        } else if (request.getPaymentType() == PaymentType.PAYPAL) {
            fee = request.getAmount() * 0.034 + 0.30; // 3.4% + $0.30 for PayPal
        } else if (request.getPaymentType() == PaymentType.BANK_TRANSFER) {
            fee = 1.50; // flat $1.50 for bank transfer
        } else {
            fee = 0;
        }

        double totalAmount = request.getAmount() + fee;

        // Processing -- grows with every new payment type
        String transactionId;
        if (request.getPaymentType() == PaymentType.CREDIT_CARD) {
            transactionId = processCreditCardPayment(request, totalAmount);
        } else if (request.getPaymentType() == PaymentType.PAYPAL) {
            transactionId = processPayPalPayment(request, totalAmount);
        } else if (request.getPaymentType() == PaymentType.BANK_TRANSFER) {
            transactionId = processBankTransferPayment(request, totalAmount);
        } else {
            throw new IllegalStateException("No processor found for: " + request.getPaymentType());
        }

        // Receipt generation -- grows with every new payment type
        String receiptMessage;
        if (request.getPaymentType() == PaymentType.CREDIT_CARD) {
            receiptMessage = String.format(
                "Credit card payment of %.2f %s processed. Card ending in %s. Fee: %.2f",
                request.getAmount(), request.getCurrency(),
                request.getCardNumber().substring(request.getCardNumber().length() - 4),
                fee
            );
        } else if (request.getPaymentType() == PaymentType.PAYPAL) {
            receiptMessage = String.format(
                "PayPal payment of %.2f %s processed for %s. Fee: %.2f",
                request.getAmount(), request.getCurrency(),
                request.getPaypalEmail(), fee
            );
        } else {
            receiptMessage = String.format(
                "Bank transfer of %.2f %s processed from account %s. Fee: %.2f",
                request.getAmount(), request.getCurrency(),
                request.getBankAccountNumber(), fee
            );
        }

        return new PaymentResult(true, transactionId, receiptMessage);
    }

    private String processCreditCardPayment(PaymentRequest request, double totalAmount) {
        // Integration with credit card gateway (Stripe, Braintree, etc.)
        System.out.println("Calling credit card gateway for amount: " + totalAmount);
        return "CC-" + System.currentTimeMillis();
    }

    private String processPayPalPayment(PaymentRequest request, double totalAmount) {
        // Integration with PayPal API
        System.out.println("Calling PayPal API for amount: " + totalAmount);
        return "PP-" + System.currentTimeMillis();
    }

    private String processBankTransferPayment(PaymentRequest request, double totalAmount) {
        // Integration with ACH/bank transfer network
        System.out.println("Initiating bank transfer for amount: " + totalAmount);
        return "BT-" + System.currentTimeMillis();
    }
}
```

**Problems with the BEFORE design:**

1. Adding `CRYPTO` payment requires modifying `PaymentRequest` (new fields), `PaymentType` (new enum value), and four separate if/else chains inside `PaymentProcessor`.
2. Every modification to `PaymentProcessor` risks breaking existing payment paths.
3. The class grows without bound as the business adds payment methods.
4. Tests for existing payment methods must be re-run and may break due to unintended side effects.
5. `PaymentRequest` is a god object that knows about every payment method's fields.
6. The method `processPayment` has high cyclomatic complexity -- every new branch raises it further.

---

### AFTER: OCP-Compliant Design

```java
// PaymentResult.java -- unchanged from before
public class PaymentResult {
    private final boolean success;
    private final String transactionId;
    private final String message;

    public PaymentResult(boolean success, String transactionId, String message) {
        this.success = success;
        this.transactionId = transactionId;
        this.message = message;
    }

    public boolean isSuccess() { return success; }
    public String getTransactionId() { return transactionId; }
    public String getMessage() { return message; }
}

// PaymentStrategy.java -- the abstraction that defines the variation point
public interface PaymentStrategy {
    void validate() throws IllegalArgumentException;
    double calculateFee(double amount);
    String processPayment(double amount, String currency);
    String generateReceiptMessage(double amount, String currency, double fee);
}

// CreditCardPaymentStrategy.java -- encapsulates everything about credit card payments
public class CreditCardPaymentStrategy implements PaymentStrategy {
    private final String cardNumber;
    private final String cardHolderName;
    private final String expiryDate;
    private final String cvv;

    public CreditCardPaymentStrategy(String cardNumber, String cardHolderName,
                                      String expiryDate, String cvv) {
        this.cardNumber = cardNumber;
        this.cardHolderName = cardHolderName;
        this.expiryDate = expiryDate;
        this.cvv = cvv;
    }

    @Override
    public void validate() throws IllegalArgumentException {
        if (cardNumber == null || cardNumber.isEmpty()) {
            throw new IllegalArgumentException("Card number is required for credit card payments");
        }
        if (cvv == null || cvv.isEmpty()) {
            throw new IllegalArgumentException("CVV is required for credit card payments");
        }
        if (expiryDate == null || expiryDate.isEmpty()) {
            throw new IllegalArgumentException("Expiry date is required for credit card payments");
        }
        if (cardNumber.length() < 13 || cardNumber.length() > 19) {
            throw new IllegalArgumentException("Invalid card number length");
        }
    }

    @Override
    public double calculateFee(double amount) {
        return amount * 0.025; // 2.5% for credit card
    }

    @Override
    public String processPayment(double amount, String currency) {
        // Integration with credit card gateway (Stripe, Braintree, etc.)
        System.out.println("Calling credit card gateway for amount: " + amount);
        // In production: call gateway API, handle response, etc.
        return "CC-" + System.currentTimeMillis();
    }

    @Override
    public String generateReceiptMessage(double amount, String currency, double fee) {
        String lastFour = cardNumber.substring(cardNumber.length() - 4);
        return String.format(
            "Credit card payment of %.2f %s processed. Card ending in %s. Fee: %.2f",
            amount, currency, lastFour, fee
        );
    }
}

// PayPalPaymentStrategy.java -- encapsulates everything about PayPal payments
public class PayPalPaymentStrategy implements PaymentStrategy {
    private final String email;
    private final String accessToken;

    public PayPalPaymentStrategy(String email, String accessToken) {
        this.email = email;
        this.accessToken = accessToken;
    }

    @Override
    public void validate() throws IllegalArgumentException {
        if (email == null || email.isEmpty()) {
            throw new IllegalArgumentException("PayPal email is required");
        }
        if (!email.contains("@")) {
            throw new IllegalArgumentException("PayPal email is not valid");
        }
        if (accessToken == null || accessToken.isEmpty()) {
            throw new IllegalArgumentException("PayPal access token is required");
        }
    }

    @Override
    public double calculateFee(double amount) {
        return (amount * 0.034) + 0.30; // 3.4% + $0.30
    }

    @Override
    public String processPayment(double amount, String currency) {
        // Integration with PayPal REST API
        System.out.println("Calling PayPal API for amount: " + amount);
        return "PP-" + System.currentTimeMillis();
    }

    @Override
    public String generateReceiptMessage(double amount, String currency, double fee) {
        return String.format(
            "PayPal payment of %.2f %s processed for %s. Fee: %.2f",
            amount, currency, email, fee
        );
    }
}

// BankTransferPaymentStrategy.java -- encapsulates everything about bank transfers
public class BankTransferPaymentStrategy implements PaymentStrategy {
    private final String accountNumber;
    private final String routingNumber;
    private final String bankName;

    public BankTransferPaymentStrategy(String accountNumber, String routingNumber, String bankName) {
        this.accountNumber = accountNumber;
        this.routingNumber = routingNumber;
        this.bankName = bankName;
    }

    @Override
    public void validate() throws IllegalArgumentException {
        if (accountNumber == null || accountNumber.isEmpty()) {
            throw new IllegalArgumentException("Bank account number is required");
        }
        if (routingNumber == null || routingNumber.isEmpty()) {
            throw new IllegalArgumentException("Routing number is required");
        }
        if (routingNumber.length() != 9) {
            throw new IllegalArgumentException("Routing number must be 9 digits");
        }
    }

    @Override
    public double calculateFee(double amount) {
        return 1.50; // flat fee for bank transfers
    }

    @Override
    public String processPayment(double amount, String currency) {
        // Integration with ACH network
        System.out.println("Initiating ACH transfer for amount: " + amount);
        return "BT-" + System.currentTimeMillis();
    }

    @Override
    public String generateReceiptMessage(double amount, String currency, double fee) {
        return String.format(
            "Bank transfer of %.2f %s processed from account ending in %s at %s. Fee: %.2f",
            amount, currency,
            accountNumber.substring(Math.max(0, accountNumber.length() - 4)),
            bankName, fee
        );
    }
}

// PaymentProcessor.java -- CLOSED for modification, OPEN for extension
// This class will NEVER be modified when a new payment method is added.
public class PaymentProcessor {
    private final PaymentStrategy paymentStrategy;

    public PaymentProcessor(PaymentStrategy paymentStrategy) {
        this.paymentStrategy = paymentStrategy;
    }

    public PaymentResult processPayment(double amount, String currency) {
        // Step 1: Validate -- delegates to strategy
        paymentStrategy.validate();

        // Step 2: Calculate fee -- delegates to strategy
        double fee = paymentStrategy.calculateFee(amount);
        double totalAmount = amount + fee;

        // Step 3: Process -- delegates to strategy
        String transactionId = paymentStrategy.processPayment(totalAmount, currency);

        // Step 4: Generate receipt -- delegates to strategy
        String receiptMessage = paymentStrategy.generateReceiptMessage(amount, currency, fee);

        return new PaymentResult(true, transactionId, receiptMessage);
    }
}
```

### Adding CryptoPaymentStrategy: Zero Modification to Existing Classes

```java
// CryptoPaymentStrategy.java -- a brand-new file, not a modification of anything existing
public class CryptoPaymentStrategy implements PaymentStrategy {
    private final String walletAddress;
    private final String cryptoCurrency;
    private final double exchangeRate; // rate to USD

    public CryptoPaymentStrategy(String walletAddress, String cryptoCurrency, double exchangeRate) {
        this.walletAddress = walletAddress;
        this.cryptoCurrency = cryptoCurrency;
        this.exchangeRate = exchangeRate;
    }

    @Override
    public void validate() throws IllegalArgumentException {
        if (walletAddress == null || walletAddress.isEmpty()) {
            throw new IllegalArgumentException("Wallet address is required for crypto payments");
        }
        if (walletAddress.length() < 26 || walletAddress.length() > 62) {
            throw new IllegalArgumentException("Invalid wallet address length");
        }
        if (cryptoCurrency == null || cryptoCurrency.isEmpty()) {
            throw new IllegalArgumentException("Cryptocurrency type is required");
        }
        if (exchangeRate <= 0) {
            throw new IllegalArgumentException("Exchange rate must be positive");
        }
    }

    @Override
    public double calculateFee(double amount) {
        return amount * 0.01; // 1% for crypto (lower due to no intermediary)
    }

    @Override
    public String processPayment(double amount, String currency) {
        double cryptoAmount = amount / exchangeRate;
        System.out.printf("Sending %.8f %s to wallet %s%n", cryptoAmount, cryptoCurrency, walletAddress);
        // Integration with blockchain API
        return "CRYPTO-" + System.currentTimeMillis();
    }

    @Override
    public String generateReceiptMessage(double amount, String currency, double fee) {
        double cryptoAmount = amount / exchangeRate;
        return String.format(
            "Crypto payment of %.8f %s (%.2f %s) processed to wallet %s. Fee: %.2f %s",
            cryptoAmount, cryptoCurrency, amount, currency,
            walletAddress.substring(0, 8) + "...",
            fee, currency
        );
    }
}

// Application.java -- demonstrating usage
public class Application {
    public static void main(String[] args) {
        // Credit card payment
        PaymentStrategy creditCardStrategy = new CreditCardPaymentStrategy(
            "4111111111111111", "John Doe", "12/26", "123"
        );
        PaymentProcessor creditCardProcessor = new PaymentProcessor(creditCardStrategy);
        PaymentResult ccResult = creditCardProcessor.processPayment(100.00, "USD");
        System.out.println(ccResult.getMessage());

        // PayPal payment
        PaymentStrategy paypalStrategy = new PayPalPaymentStrategy(
            "john.doe@example.com", "PAYPAL_ACCESS_TOKEN_XYZ"
        );
        PaymentProcessor paypalProcessor = new PaymentProcessor(paypalStrategy);
        PaymentResult ppResult = paypalProcessor.processPayment(100.00, "USD");
        System.out.println(ppResult.getMessage());

        // Bank transfer
        PaymentStrategy bankStrategy = new BankTransferPaymentStrategy(
            "123456789", "021000021", "Chase Bank"
        );
        PaymentProcessor bankProcessor = new PaymentProcessor(bankStrategy);
        PaymentResult bankResult = bankProcessor.processPayment(100.00, "USD");
        System.out.println(bankResult.getMessage());

        // Crypto -- added AFTER the initial design, zero modification to PaymentProcessor
        PaymentStrategy cryptoStrategy = new CryptoPaymentStrategy(
            "1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2", "BTC", 65000.0
        );
        PaymentProcessor cryptoProcessor = new PaymentProcessor(cryptoStrategy);
        PaymentResult cryptoResult = cryptoProcessor.processPayment(100.00, "USD");
        System.out.println(cryptoResult.getMessage());
    }
}
```

### What the Refactoring Achieved

| Concern | BEFORE | AFTER |
|---|---|---|
| Adding a new payment type | Modify `PaymentType`, `PaymentRequest`, and `PaymentProcessor` | Create one new class implementing `PaymentStrategy` |
| Validation logic location | Scattered across if/else in `PaymentProcessor` | Encapsulated in each strategy class |
| Fee logic location | Scattered across if/else in `PaymentProcessor` | Encapsulated in each strategy class |
| `PaymentProcessor` complexity | Grows with every new payment type | Fixed; always the same 4-step orchestration |
| Risk of regression when adding new type | High -- you modify the shared class | None -- existing classes are untouched |
| Test scope when adding new type | Must re-test all payment types | Only test the new strategy class |
| `PaymentRequest` | God object with fields for all payment types | Eliminated; each strategy carries its own data |

---

## 5. Plugin Architecture Analogy

OCP is the design principle that makes plugin architectures possible. A plugin system is OCP expressed at the system level rather than the class level.

### How a Plugin System Works

A plugin host defines an interface (or abstract class) that all plugins must implement. The host is compiled, shipped, and closed -- it cannot be modified after release. Plugins are separate modules that implement the host's interface. The host discovers and loads plugins at runtime, typically through a registry, a configuration file, or a directory scan.

```java
// Plugin host defines the contract
public interface ReportGenerator {
    String getSupportedFormat();
    byte[] generateReport(ReportData data);
}

// Host's report service -- closed for modification
public class ReportService {
    private final Map<String, ReportGenerator> generators = new HashMap<>();

    // Plugins register themselves here
    public void registerGenerator(ReportGenerator generator) {
        generators.put(generator.getSupportedFormat(), generator);
    }

    public byte[] generateReport(String format, ReportData data) {
        ReportGenerator generator = generators.get(format);
        if (generator == null) {
            throw new IllegalArgumentException("No generator registered for format: " + format);
        }
        return generator.generateReport(data);
    }
}

// Built-in plugins
public class PdfReportGenerator implements ReportGenerator {
    @Override
    public String getSupportedFormat() { return "PDF"; }

    @Override
    public byte[] generateReport(ReportData data) {
        System.out.println("Generating PDF report");
        return new byte[0]; // actual PDF generation
    }
}

public class ExcelReportGenerator implements ReportGenerator {
    @Override
    public String getSupportedFormat() { return "XLSX"; }

    @Override
    public byte[] generateReport(ReportData data) {
        System.out.println("Generating Excel report");
        return new byte[0]; // actual Excel generation
    }
}

// Third-party plugin -- shipped separately, added to the system without modifying host
public class CsvReportGenerator implements ReportGenerator {
    @Override
    public String getSupportedFormat() { return "CSV"; }

    @Override
    public byte[] generateReport(ReportData data) {
        System.out.println("Generating CSV report");
        return new byte[0]; // CSV generation
    }
}

// Wiring
public class Application {
    public static void main(String[] args) {
        ReportService service = new ReportService();
        service.registerGenerator(new PdfReportGenerator());
        service.registerGenerator(new ExcelReportGenerator());

        // Third-party plugin added dynamically -- zero change to ReportService
        service.registerGenerator(new CsvReportGenerator());

        ReportData data = new ReportData();
        service.generateReport("CSV", data);
    }
}
```

### Real-World Plugin Systems Built on OCP

- **IntelliJ IDEA / VS Code**: The editor defines interfaces for language servers, debuggers, and formatters. Plugins implement those interfaces. The editor never changes when a new language plugin ships.
- **Java's `java.sql.Driver`**: The JDBC `DriverManager` accepts any class that implements `java.sql.Driver`. MySQL, PostgreSQL, Oracle ship their drivers as separate JARs. The JDK is not modified when a new database vendor ships a driver.
- **Spring Framework's `ApplicationContextInitializer`, `BeanFactoryPostProcessor`**: Spring's container is closed. You extend its behavior by implementing these interfaces and registering your implementations.
- **Webpack loaders**: The bundler is closed. Adding support for a new file type means writing a new loader plugin, not modifying Webpack.

The pattern is always the same: a stable host defines an interface, and new capabilities arrive as new implementations, not as modifications to the host.

---

## 6. Anti-Pattern: The Modifier Trap

### Definition

The modifier trap is a design anti-pattern in which every new business requirement triggers a modification to an existing class rather than the creation of a new class. Systems that fall into this trap accumulate large, brittle classes that are expensive to change and expensive to test.

### How It Happens

The modifier trap typically begins with a class that works correctly for an initial set of requirements. When the first new requirement arrives, it seems simpler to add a branch to the existing class than to redesign the abstraction. This is often true in the short term. But as more requirements arrive, the class grows, the branches multiply, and the cost of each subsequent modification increases nonlinearly.

### Example: Notification System

```java
// Initial design -- works fine for email
public class NotificationService {
    public void sendNotification(String recipient, String message, String type) {
        if (type.equals("EMAIL")) {
            sendEmail(recipient, message);
        }
        // That's it, initially. Simple and correct.
    }

    private void sendEmail(String recipient, String message) {
        System.out.println("Sending email to " + recipient + ": " + message);
    }
}
```

Then requirements arrive one by one:

**Requirement 1:** Add SMS support.

```java
// Modification 1 -- seems harmless
public void sendNotification(String recipient, String message, String type) {
    if (type.equals("EMAIL")) {
        sendEmail(recipient, message);
    } else if (type.equals("SMS")) {
        sendSms(recipient, message);
    }
}
```

**Requirement 2:** Add Slack integration.

```java
// Modification 2 -- the class is growing
public void sendNotification(String recipient, String message, String type) {
    if (type.equals("EMAIL")) {
        sendEmail(recipient, message);
    } else if (type.equals("SMS")) {
        sendSms(recipient, message);
    } else if (type.equals("SLACK")) {
        sendSlackMessage(recipient, message);
    }
}
```

**Requirement 3:** Add push notifications with priority levels.

```java
// Modification 3 -- now the method signature must change too
public void sendNotification(String recipient, String message, String type, Integer priority) {
    if (type.equals("EMAIL")) {
        sendEmail(recipient, message);
    } else if (type.equals("SMS")) {
        sendSms(recipient, message);
    } else if (type.equals("SLACK")) {
        sendSlackMessage(recipient, message);
    } else if (type.equals("PUSH")) {
        sendPushNotification(recipient, message, priority);
    }
}
// PROBLEM: Callers that don't use PUSH now have to pass a null priority.
// The method signature change broke the contract for existing callers.
```

**Requirement 4:** Add WhatsApp, but only if the recipient has opted in, and with rate limiting per sender.

```java
// Modification 4 -- the class is now a mess
public void sendNotification(String recipient, String message, String type,
                              Integer priority, boolean optedIn, RateLimiter rateLimiter) {
    if (type.equals("EMAIL")) {
        sendEmail(recipient, message);
    } else if (type.equals("SMS")) {
        sendSms(recipient, message);
    } else if (type.equals("SLACK")) {
        sendSlackMessage(recipient, message);
    } else if (type.equals("PUSH")) {
        sendPushNotification(recipient, message, priority);
    } else if (type.equals("WHATSAPP")) {
        if (!optedIn) {
            throw new IllegalStateException("Recipient has not opted in to WhatsApp notifications");
        }
        if (!rateLimiter.tryAcquire()) {
            throw new IllegalStateException("Rate limit exceeded for WhatsApp sender");
        }
        sendWhatsApp(recipient, message);
    }
}
// The method signature now has 6 parameters, most of which are irrelevant for most channels.
// Every caller must be updated.
// The class is impossible to test in isolation.
```

### Why This Is a Trap

1. **Each modification raises cyclomatic complexity.** More branches means more test cases needed, and more places where a bug can hide.
2. **The method signature grows to accommodate the most complex channel.** Every caller of the simpler channels must pass null or irrelevant values.
3. **Cohesion collapses.** The class knows about email servers, SMS gateways, Slack APIs, push notification services, and WhatsApp Business APIs simultaneously. It cannot be tested without mocking all of these.
4. **Modification risk compounds.** Changing the Slack integration requires opening a class that also contains the credit card processing logic. Any mistake can affect unrelated channels.

### The OCP-Compliant Alternative

```java
public interface NotificationChannel {
    void send(String recipient, String message, NotificationOptions options);
    boolean supports(String channelType);
}

public class NotificationService {
    private final List<NotificationChannel> channels;

    public NotificationService(List<NotificationChannel> channels) {
        this.channels = channels;
    }

    public void sendNotification(String recipient, String message,
                                  String channelType, NotificationOptions options) {
        NotificationChannel channel = channels.stream()
            .filter(c -> c.supports(channelType))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                "No channel registered for type: " + channelType));
        channel.send(recipient, message, options);
    }
}
```

Each channel is a separate class. Adding WhatsApp means creating `WhatsAppNotificationChannel`. The `NotificationService` class is never modified.

---

## 7. Other OCP Enablers: Template Method and Decorator

The Strategy pattern is the most common OCP enabler, but two other patterns also achieve OCP in different contexts.

### Template Method Pattern

Template Method uses inheritance to achieve OCP. A base class defines the skeleton of an algorithm -- the sequence of steps -- and declares abstract methods for the steps that vary. Subclasses provide the varying implementations without changing the algorithm's structure.

```java
// The template -- defines the algorithm skeleton
public abstract class DataExporter {

    // Template method -- the algorithm, fixed and closed
    public final void export(List<Record> records, String destination) {
        List<Record> prepared = prepareData(records);
        String formatted = formatData(prepared);
        writeOutput(formatted, destination);
        logExport(destination, prepared.size());
    }

    // Fixed step -- shared by all subclasses
    private List<Record> prepareData(List<Record> records) {
        return records.stream()
            .filter(r -> r.isValid())
            .collect(java.util.stream.Collectors.toList());
    }

    // Variable step -- each subclass provides its own implementation
    protected abstract String formatData(List<Record> records);

    // Variable step
    protected abstract void writeOutput(String data, String destination);

    // Fixed step -- shared by all subclasses
    private void logExport(String destination, int recordCount) {
        System.out.printf("Exported %d records to %s%n", recordCount, destination);
    }
}

// Concrete implementation: CSV
public class CsvDataExporter extends DataExporter {
    @Override
    protected String formatData(List<Record> records) {
        StringBuilder sb = new StringBuilder();
        sb.append("id,name,value\n");
        for (Record r : records) {
            sb.append(r.getId()).append(",")
              .append(r.getName()).append(",")
              .append(r.getValue()).append("\n");
        }
        return sb.toString();
    }

    @Override
    protected void writeOutput(String data, String destination) {
        System.out.println("Writing CSV to: " + destination);
        // actual file write
    }
}

// Concrete implementation: JSON -- added later, zero modification to DataExporter
public class JsonDataExporter extends DataExporter {
    @Override
    protected String formatData(List<Record> records) {
        StringBuilder sb = new StringBuilder("[");
        for (int i = 0; i < records.size(); i++) {
            Record r = records.get(i);
            sb.append(String.format("{\"id\":%s,\"name\":\"%s\",\"value\":\"%s\"}",
                r.getId(), r.getName(), r.getValue()));
            if (i < records.size() - 1) sb.append(",");
        }
        sb.append("]");
        return sb.toString();
    }

    @Override
    protected void writeOutput(String data, String destination) {
        System.out.println("Writing JSON to: " + destination);
        // actual file write
    }
}
```

**When to use Template Method vs. Strategy:**
- Use Template Method when the algorithm skeleton is fixed and only specific steps vary. The variation is tightly coupled to a defined sequence.
- Use Strategy when the entire algorithm varies and you want to swap it out independently of the consumer. The variation is loosely coupled.
- Template Method uses inheritance; Strategy uses composition. Composition is generally preferred (see Favor Composition over Inheritance), but Template Method is appropriate when subclass specialization is natural and the class hierarchy is shallow.

### Decorator Pattern

The Decorator pattern adds behavior to an object at runtime without modifying the object's class. It achieves OCP by wrapping the original object with a new object that adds the extra behavior.

```java
// The base abstraction
public interface MessageSender {
    void send(String recipient, String message);
}

// Concrete implementation
public class BasicMessageSender implements MessageSender {
    @Override
    public void send(String recipient, String message) {
        System.out.printf("Sending to %s: %s%n", recipient, message);
    }
}

// Decorator: adds logging without modifying BasicMessageSender
public class LoggingMessageSender implements MessageSender {
    private final MessageSender delegate;

    public LoggingMessageSender(MessageSender delegate) {
        this.delegate = delegate;
    }

    @Override
    public void send(String recipient, String message) {
        System.out.printf("[LOG] Sending message to %s at %s%n", recipient, java.time.Instant.now());
        delegate.send(recipient, message);
        System.out.printf("[LOG] Message sent to %s%n", recipient);
    }
}

// Decorator: adds retry logic without modifying BasicMessageSender or LoggingMessageSender
public class RetryingMessageSender implements MessageSender {
    private final MessageSender delegate;
    private final int maxRetries;

    public RetryingMessageSender(MessageSender delegate, int maxRetries) {
        this.delegate = delegate;
        this.maxRetries = maxRetries;
    }

    @Override
    public void send(String recipient, String message) {
        int attempts = 0;
        while (attempts <= maxRetries) {
            try {
                delegate.send(recipient, message);
                return;
            } catch (Exception e) {
                attempts++;
                if (attempts > maxRetries) {
                    throw new RuntimeException("Failed after " + maxRetries + " retries", e);
                }
                System.out.printf("[RETRY] Attempt %d failed, retrying...%n", attempts);
            }
        }
    }
}

// Decorator: adds encryption -- added later, zero modification to existing classes
public class EncryptingMessageSender implements MessageSender {
    private final MessageSender delegate;

    public EncryptingMessageSender(MessageSender delegate) {
        this.delegate = delegate;
    }

    @Override
    public void send(String recipient, String message) {
        String encrypted = encrypt(message);
        delegate.send(recipient, encrypted);
    }

    private String encrypt(String message) {
        // encryption logic
        return "[ENCRYPTED]" + message;
    }
}

// Usage: compose behaviors by stacking decorators
public class Application {
    public static void main(String[] args) {
        MessageSender sender = new LoggingMessageSender(
            new RetryingMessageSender(
                new EncryptingMessageSender(
                    new BasicMessageSender()
                ),
                3
            )
        );

        sender.send("user@example.com", "Hello, world!");
        // Output: logs the send, encrypts the message, retries on failure
    }
}
```

The Decorator pattern is particularly powerful for cross-cutting concerns: logging, retry, caching, rate limiting, authentication. Each concern is a separate decorator. You compose them without modifying any existing class.

---

## 8. Practical Limits: When Modification IS the Right Answer

OCP is a design guideline, not an absolute rule. There are situations where modification is the correct choice and attempting to apply OCP would make the code worse.

### 1. Fixing Bugs

If a class has a bug, the correct answer is to modify the class to fix the bug. You do not create a subclass or a new strategy implementation to work around a bug in existing code. Bug fixes require modification; that is their nature.

OCP is about protecting stable, correct code from the churn of new requirements. It does not apply when the existing code is incorrect.

### 2. Changing the Abstraction Itself

If the interface you defined turns out to be wrong -- if it cannot model a new requirement without being contorted -- then you should modify the interface rather than create a workaround.

Example: You designed a `PaymentStrategy` interface with a single `processPayment(double amount, String currency)` method. A new requirement demands asynchronous payment processing with a callback for completion. This cannot be modeled by your existing interface. The right answer is to change the interface, not to add an `AsyncPaymentStrategy` that misuses the existing one.

Changing an abstraction is expensive (all implementations must update), but it is the right answer when the abstraction is wrong. The alternative -- layering workarounds onto a broken abstraction -- produces code that is harder to understand and maintain than the modification would have been.

### 3. Early-Stage Code Before the Right Abstraction is Clear

In early development, the correct abstraction is often not yet visible. You may have one or two cases that look like they could be abstracted, but you cannot tell what the abstraction should look like until you have seen more cases.

Applying OCP prematurely -- defining an interface before you understand the variation -- often produces an abstraction that does not fit the second or third requirement and must be redesigned anyway. At that point, you have paid the cost of designing the abstraction and also the cost of redesigning it.

The pragmatic approach: start with simple code. Let the requirements accumulate. When the pattern is clear, refactor to introduce the abstraction.

### 4. Performance-Critical Code Paths

Polymorphic dispatch (virtual method calls, interface method resolution) has a small but nonzero overhead compared to direct method calls. In hot paths -- code that is called millions of times per second -- this overhead can matter. In some cases, a direct if/switch chain is more performant and more appropriate than a polymorphic design.

This is a narrow exception. For the vast majority of business logic, the performance difference between polymorphic dispatch and direct calls is unmeasurable. But it is worth knowing the exception exists.

### 5. Internal Implementation Details

OCP applies at the boundary where external requirements drive change. It does not mean that every class should be designed for extension. Internal implementation classes that are used only within a single module, that are not part of a public API, and that are unlikely to need extension are reasonable candidates for keeping simple and concrete. Over-engineering internal implementation details with unnecessary abstractions is a code smell in the other direction.

### The Judgment Call

OCP requires judgment. The question to ask is: "Given what I know about how this code is likely to change, is it worth introducing this abstraction now?" If the answer is yes, apply OCP. If the answer is no (too early, too speculative, too niche), write simple code and refactor when the pattern is clear.

---

## 9. The "Fool Me Once" Rule

### The Problem with Premature Abstraction

One of the most common mistakes engineers make when learning OCP is applying it too early. They see a switch statement with two cases and immediately reach for the Strategy pattern. This produces code that is more complex than the problem requires, harder for other engineers to read, and potentially based on an abstraction that will not survive the second real requirement.

The cost of premature abstraction:
- You have to design an interface before you know what the variation point actually looks like.
- The interface you design is often wrong, because you have only seen one case.
- The resulting code is harder to read than the simple version, without being more maintainable.
- Other engineers must understand the abstraction to make any change, even trivial ones.

### The Rule

The "fool me once" rule (sometimes called the Rule of Three or the Second System Rule in this context) states:

**Wait until you have seen the same kind of change twice before introducing an abstraction.**

The first time a requirement arrives that would require you to add a new branch, add the branch. Keep it simple.

The second time a similar requirement arrives -- confirming that this is indeed a variation point -- introduce the abstraction. Refactor both the original and the new case to use the abstraction.

From that point forward, every new requirement in this category is handled by extension (creating a new implementation), not modification.

### Example

**First requirement:** Add support for email notifications.

```java
public class NotificationService {
    public void notify(String recipient, String message) {
        sendEmail(recipient, message);
    }
}
```

Simple. No abstraction needed yet.

**Second requirement:** Add SMS notifications.

Now you have seen the pattern twice. This is a genuine variation point. Introduce the abstraction:

```java
public interface NotificationChannel {
    void send(String recipient, String message);
}

public class EmailNotificationChannel implements NotificationChannel {
    @Override
    public void send(String recipient, String message) {
        System.out.println("Sending email to " + recipient);
    }
}

public class SmsNotificationChannel implements NotificationChannel {
    @Override
    public void send(String recipient, String message) {
        System.out.println("Sending SMS to " + recipient);
    }
}

public class NotificationService {
    private final NotificationChannel channel;

    public NotificationService(NotificationChannel channel) {
        this.channel = channel;
    }

    public void notify(String recipient, String message) {
        channel.send(recipient, message);
    }
}
```

**All subsequent requirements** (Slack, push, WhatsApp) are handled by creating new `NotificationChannel` implementations. The `NotificationService` is never modified again.

### Why "Fool Me Once"

If you introduced the abstraction at the first requirement, you would have designed an interface based on a sample size of one. The interface would likely be wrong or incomplete by the time the second requirement arrived, requiring a redesign that negated the benefit of early abstraction. By waiting for the second case, you have enough information to design the abstraction correctly.

The engineer who applies OCP after the second case is neither too early (premature abstraction) nor too late (the modifier trap has taken hold). This is the judgment that separates experienced engineers from those still learning the principle.

---

## 10. Interview Questions and Model Answers

### Q1: What is the Open/Closed Principle, and why does it matter?

**Model Answer:**

The Open/Closed Principle states that software entities should be open for extension but closed for modification. In practice, this means that when a new requirement arrives, you satisfy it by writing new code -- a new class, a new implementation of an interface -- rather than modifying existing, tested code.

It matters for two reasons. First, modifying existing code risks regression. Code that is in production and working correctly has been tested and debugged over time. Opening it up to add new behavior introduces the possibility of breaking existing behavior. Second, OCP enables systems to scale with requirements. A system designed around OCP can accommodate new payment methods, new notification channels, new export formats, or new algorithms without growing the complexity of any single class. New requirements add new classes, not new branches to existing classes.

---

### Q2: How does the Strategy pattern relate to OCP?

**Model Answer:**

The Strategy pattern is the most direct implementation of OCP. It works by defining an abstraction -- an interface -- that represents the variation point. The consuming class depends on the abstraction and delegates variable behavior to it. New behavior is added by creating a new class that implements the interface.

The result is that the consuming class satisfies OCP: it is closed for modification (no existing code changes when a new strategy is added) and open for extension (new strategies can be plugged in). The strategy interface defines the "seam" at which extension is possible.

The Strategy pattern is particularly useful when the variation involves a family of algorithms or behaviors that are interchangeable from the consumer's perspective.

---

### Q3: Can you give an example of an OCP violation and explain how you would fix it?

**Model Answer:**

A classic violation is a class with a method containing a switch or if/else chain that dispatches behavior based on a type enum or string. For example, a `ShippingCalculator` that has:

```java
if (type.equals("STANDARD")) { ... }
else if (type.equals("EXPRESS")) { ... }
else if (type.equals("OVERNIGHT")) { ... }
```

To add same-day delivery, you must open the `ShippingCalculator` class and add another branch. This risks breaking existing shipping calculations and requires re-testing all existing paths.

The fix is to extract the varying behavior into a `ShippingStrategy` interface with a `calculateCost(Order order)` method, create `StandardShippingStrategy`, `ExpressShippingStrategy`, and `OvernightShippingStrategy` implementations, and have `ShippingCalculator` accept a `ShippingStrategy` and delegate to it. Adding same-day delivery then requires creating `SameDayShippingStrategy` -- no modification to `ShippingCalculator` or any existing strategy.

---

### Q4: Is OCP about inheritance or composition?

**Model Answer:**

Meyer's original formulation was inheritance-centric -- you extend a class by subclassing it. Martin's reinterpretation shifted to composition and polymorphism via interfaces, which is the dominant modern interpretation.

In practice, OCP is best achieved through composition. You define an interface (the abstraction), and the consumer composes with an instance of that interface at runtime. The implementation is selected externally (by the caller, by a factory, or by a dependency injection container) rather than baked into the consumer.

Inheritance-based OCP (Template Method) is valid but has the limitations of inheritance: tight coupling to the base class, deep class hierarchies that are hard to navigate, and inflexibility when a class needs to vary along multiple dimensions simultaneously. Composition (Strategy, Decorator) avoids these limitations.

---

### Q5: How do you identify where to apply OCP in a codebase?

**Model Answer:**

The key signal is a switch statement or if/else chain that dispatches behavior based on a type. When you see this pattern, ask: "Is this list of cases likely to grow as new requirements arrive?" If yes, the switch is a strong candidate for replacement with a strategy or plugin pattern.

Other signals:
- A class whose method signature keeps growing to accommodate new parameters that only apply to specific cases.
- A class that must be modified every time a business concept is extended (new payment method, new report format, new discount type).
- A class with high cyclomatic complexity driven by type-based branching.
- Test difficulty: if you cannot test one case without also setting up the infrastructure for other cases.

I do not apply OCP preemptively. I wait until I have seen the same kind of change twice (the "fool me once" rule) before I introduce the abstraction. The first time, I add the branch. The second time, I recognize the pattern and refactor to an interface.

---

### Q6: What is the difference between OCP and the Single Responsibility Principle?

**Model Answer:**

SRP says that a class should have one reason to change -- it should be responsible for one cohesive thing. OCP says that when the thing the class is responsible for needs to evolve, that evolution should happen through extension rather than modification.

They are complementary. If a class violates SRP -- if it is responsible for handling credit card payments, PayPal payments, and bank transfers all at once -- then it will inevitably violate OCP as well, because every new payment method gives it a new reason to change. Fixing the SRP violation (by extracting each payment type into its own class) typically also fixes the OCP violation, because each class now handles only its own type and does not need to be modified for other types.

The practical order is: first apply SRP (give each class one responsibility), then OCP comes naturally (each class is small and focused enough that it rarely needs modification).

---

### Q7: When should you NOT apply OCP?

**Model Answer:**

OCP should not be applied mechanically to all code. The most important cases where modification is correct:

1. **Bug fixes.** A bug in existing code requires modifying that code. Creating a subclass or strategy to work around a bug is wrong.

2. **Wrong abstraction.** If the interface itself is the problem -- if it cannot model a new requirement without distortion -- modify the interface. The cost of modifying the abstraction is high (all implementations must update), but it is lower than the cost of maintaining a broken abstraction with workarounds layered on top of it.

3. **Early-stage code.** Before the variation point is clear, premature abstraction produces interfaces that are wrong. It is better to write simple code and refactor when the pattern is clear.

4. **Rare, one-off variation.** If a new requirement is genuinely unique and unlikely to recur, introducing an abstraction for it creates complexity without benefit. A one-line special case is sometimes cleaner than a full strategy implementation.

The key question is always: "Given what I know about the likely evolution of this code, does the complexity of the abstraction pay for itself?"

---

### Q8: How does OCP relate to dependency inversion?

**Model Answer:**

OCP and the Dependency Inversion Principle (DIP) are tightly related. OCP says that consuming code should not need modification when new implementations arrive. DIP says that high-level modules should depend on abstractions, not on concrete details.

Applying DIP is the mechanism by which you achieve OCP. If `PaymentProcessor` depended directly on `CreditCardProcessor` (a concrete class), it would need to be modified every time a new payment method was added. By depending on `PaymentStrategy` (an abstraction, per DIP), `PaymentProcessor` is shielded from those modifications (achieving OCP).

In practice, when you design a class that satisfies DIP, it usually also satisfies OCP for the dimension of variation that the abstraction covers. Conversely, if a class violates OCP, it is almost always because it violates DIP -- it depends on a concrete class or a type enum instead of an abstraction.

---

### Q9: How would you explain OCP to a junior engineer who sees Strategy pattern as "overengineering"?

**Model Answer:**

The objection is legitimate for small codebases early in their life. A switch with three cases is readable and correct. The Strategy pattern for three cases can feel like overkill.

The argument for OCP is not about the current state -- it is about the cost of change over time. A switch with three cases is a switch with three cases today. But payment systems do not stay at three payment types. Notification systems do not stay at one channel. Over 18 months, that switch will have 8 cases. The engineer adding the 8th case must read and understand the first 7. The risk of regression grows with each addition. The test surface grows with each addition.

The Strategy pattern feels like overhead upfront, but it keeps the cost of each addition constant. Adding the 8th payment strategy costs the same as adding the 2nd: write one new class, test it in isolation, inject it where needed. The switch statement's cost grows linearly with the number of cases; the Strategy pattern's cost stays flat.

I also explain the "fool me once" rule: you do not have to introduce the abstraction at the first case. Wait until the second case arrives. At that point, the pattern is clear, the abstraction can be designed correctly, and the refactoring is straightforward.

---

### Q10: How does OCP manifest in real-world frameworks you have used?

**Model Answer:**

OCP is visible throughout modern frameworks:

**Spring Framework:** The `BeanPostProcessor` and `BeanFactoryPostProcessor` interfaces allow you to extend the Spring container's behavior without modifying Spring's code. The container is closed; your customizations are extensions.

**Java's JDBC:** The `java.sql.Driver` interface means that adding a new database driver requires writing a class that implements `Driver` and registering it. The JDBC API never changes; only new driver JARs are added.

**JUnit:** The `Extension` interface in JUnit 5 allows you to add custom setup, teardown, parameterized test sources, and other behaviors. You extend JUnit without modifying its source.

**Servlet filters:** A `javax.servlet.Filter` allows you to add logging, authentication, compression, and other concerns to an HTTP pipeline without modifying the servlet container or the servlets themselves.

**Hibernate's dialect system:** Adding support for a new database in Hibernate requires creating a new `Dialect` subclass. The ORM engine is closed; new database support arrives as extension.

In every case, the framework designers identified the variation points (database vendors, test lifecycle, request pipeline, container customization) and extracted them as interfaces or abstract classes. Callers extend those, not the framework.

---

## 11. Assignment: Discount Calculation Refactoring

### Background

You have been given the following `DiscountCalculator` class that a colleague wrote to handle discounts for an e-commerce platform. The class handles several discount types via a switch statement. Business is growing and the product team has requested two new discount types: a flash sale discount (50% off, time-limited) and a bundle discount (15% off if 3 or more items are purchased).

### Starting Code (OCP Violation)

Your job is to study this code, identify the OCP violation, and refactor it to an OCP-compliant design using the Strategy pattern.

```java
// DiscountType.java
public enum DiscountType {
    NONE,
    PERCENTAGE,
    FIXED_AMOUNT,
    BUY_ONE_GET_ONE,
    LOYALTY
}

// Order.java
public class Order {
    private final String orderId;
    private final double subtotal;
    private final int itemCount;
    private final String customerId;
    private final boolean isLoyaltyMember;

    public Order(String orderId, double subtotal, int itemCount,
                 String customerId, boolean isLoyaltyMember) {
        this.orderId = orderId;
        this.subtotal = subtotal;
        this.itemCount = itemCount;
        this.customerId = customerId;
        this.isLoyaltyMember = isLoyaltyMember;
    }

    public String getOrderId() { return orderId; }
    public double getSubtotal() { return subtotal; }
    public int getItemCount() { return itemCount; }
    public String getCustomerId() { return customerId; }
    public boolean isLoyaltyMember() { return isLoyaltyMember; }
}

// DiscountResult.java
public class DiscountResult {
    private final double originalAmount;
    private final double discountAmount;
    private final double finalAmount;
    private final String discountDescription;

    public DiscountResult(double originalAmount, double discountAmount,
                           double finalAmount, String discountDescription) {
        this.originalAmount = originalAmount;
        this.discountAmount = discountAmount;
        this.finalAmount = finalAmount;
        this.discountDescription = discountDescription;
    }

    public double getOriginalAmount() { return originalAmount; }
    public double getDiscountAmount() { return discountAmount; }
    public double getFinalAmount() { return finalAmount; }
    public String getDiscountDescription() { return discountDescription; }
}

// DiscountCalculator.java -- THIS CLASS VIOLATES OCP
// You must refactor this without modifying this class's internals
// (other than removing it and replacing with OCP-compliant design)
public class DiscountCalculator {

    public DiscountResult calculate(Order order, DiscountType discountType, double discountValue) {
        double discountAmount;
        String description;

        switch (discountType) {
            case NONE:
                discountAmount = 0.0;
                description = "No discount applied";
                break;

            case PERCENTAGE:
                if (discountValue < 0 || discountValue > 100) {
                    throw new IllegalArgumentException("Percentage discount must be between 0 and 100");
                }
                discountAmount = order.getSubtotal() * (discountValue / 100.0);
                description = String.format("%.0f%% percentage discount: -%.2f", discountValue, discountAmount);
                break;

            case FIXED_AMOUNT:
                if (discountValue < 0) {
                    throw new IllegalArgumentException("Fixed discount amount cannot be negative");
                }
                discountAmount = Math.min(discountValue, order.getSubtotal());
                description = String.format("Fixed discount: -%.2f", discountAmount);
                break;

            case BUY_ONE_GET_ONE:
                if (order.getItemCount() < 2) {
                    discountAmount = 0.0;
                    description = "BOGO discount not applicable (fewer than 2 items)";
                } else {
                    // Discount is half the price per item times the number of free items
                    double pricePerItem = order.getSubtotal() / order.getItemCount();
                    int freeItems = order.getItemCount() / 2;
                    discountAmount = pricePerItem * freeItems;
                    description = String.format("Buy one get one: %d free item(s), -%.2f", freeItems, discountAmount);
                }
                break;

            case LOYALTY:
                if (!order.isLoyaltyMember()) {
                    discountAmount = 0.0;
                    description = "Loyalty discount not applicable (not a loyalty member)";
                } else {
                    discountAmount = order.getSubtotal() * 0.10; // 10% loyalty discount
                    description = String.format("Loyalty member discount (10%%): -%.2f", discountAmount);
                }
                break;

            default:
                throw new IllegalArgumentException("Unsupported discount type: " + discountType);
        }

        double finalAmount = order.getSubtotal() - discountAmount;
        return new DiscountResult(order.getSubtotal(), discountAmount, finalAmount, description);
    }
}
```

### Assignment Tasks

**Task 1: Identify the violation**

Write a brief explanation (3-5 sentences) of exactly how the `DiscountCalculator` class violates OCP. Be specific: what must happen to the class when `FLASH_SALE` and `BUNDLE` discount types are added?

**Task 2: Design the abstraction**

Define a `DiscountStrategy` interface. Consider:
- What parameters does the strategy need? (Hint: the order contains context; the discount-specific configuration should come from the strategy's constructor.)
- Should validation be part of the interface?
- Should the description be returned by the interface or computed externally?

**Task 3: Implement all existing discount types as strategies**

Create the following classes:
- `NoDiscountStrategy`
- `PercentageDiscountStrategy`
- `FixedAmountDiscountStrategy`
- `BuyOneGetOneDiscountStrategy`
- `LoyaltyDiscountStrategy`

Each must implement your `DiscountStrategy` interface and encapsulate its own validation, calculation logic, and description.

**Task 4: Implement the two new discount types**

Create:
- `FlashSaleDiscountStrategy`: 50% off the subtotal. Must validate that the current time is within the sale window (accept a `java.time.Instant saleStart` and `java.time.Instant saleEnd` in the constructor).
- `BundleDiscountStrategy`: 15% off if 3 or more items are purchased. If fewer than 3 items, no discount is applied.

Note: These classes should require zero modification to any of the previously written strategy classes or the refactored `DiscountCalculator`.

**Task 5: Refactor DiscountCalculator**

Rewrite `DiscountCalculator` to accept a `DiscountStrategy` and delegate to it. The refactored class should be so simple that it is obviously correct without further testing.

**Task 6: Write unit tests**

Write JUnit 5 unit tests for at least three of your strategy implementations. Each test class should test only its own strategy in isolation -- no mocks of other strategies, no dependency on the `DiscountCalculator`.

### Expected Interface Design (Hint)

```java
public interface DiscountStrategy {
    DiscountResult apply(Order order);
}
```

The strategy's constructor accepts the discount-specific configuration (percentage value, fixed amount, sale window, etc.). The `apply` method receives the order context. Validation happens inside `apply`, throwing `IllegalArgumentException` when the discount cannot be applied.

### What a Correct Solution Demonstrates

1. `DiscountCalculator` has no switch or if/else branching on discount type.
2. Adding `FlashSaleDiscountStrategy` and `BundleDiscountStrategy` required creating two new files and modifying zero existing files.
3. Each strategy class is independently testable: you can instantiate it alone, call `apply`, and assert the result without involving any other class.
4. The `DiscountCalculator` tests do not need to be updated when new strategies are added, because its behavior (delegate to the strategy) has not changed.

### Evaluation Criteria

| Criterion | Weight |
|---|---|
| `DiscountStrategy` interface is correctly defined and captures all variation | 20% |
| All five existing discount types are correctly implemented as strategies | 25% |
| `FlashSaleDiscountStrategy` and `BundleDiscountStrategy` are correctly implemented with no modification to existing classes | 25% |
| `DiscountCalculator` contains no type-based branching and delegates fully to the strategy | 15% |
| Unit tests are isolated per strategy and cover validation and calculation | 15% |

---

*End of Open/Closed Principle module.*
