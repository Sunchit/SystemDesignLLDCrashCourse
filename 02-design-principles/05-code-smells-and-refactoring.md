# Code Smells and Refactoring

> "A code smell is a surface indication that usually corresponds to a deeper problem in the system."
> — Martin Fowler, *Refactoring: Improving the Design of Existing Code*

Code smells are not bugs. They are symptoms. They indicate that the design has accumulated debt that will make future changes harder and more error-prone. Recognizing smells is the first step to fixing them. Refactoring is the practice of fixing them without breaking behavior.

---

## Table of Contents

**Code Smells**
1. [Bloaters](#1-bloaters)
2. [Object-Orientation Abusers](#2-object-orientation-abusers)
3. [Change Preventers](#3-change-preventers)
4. [Dispensables](#4-dispensables)
5. [Couplers](#5-couplers)

**Refactoring Techniques**
6. [Core Refactoring Techniques](#6-core-refactoring-techniques)

---

# Code Smells

## 1. Bloaters

Bloaters are code elements that have grown too large to work with comfortably.

---

### Long Method

**Description:** A method that has grown so long it is difficult to understand as a unit. Fowler's rule of thumb: a method longer than 10 lines deserves scrutiny.

**Detection:** Methods spanning more than one screen. Deep nesting. Methods with comments that describe "sections" of the code.

**The smell:**
```java
public void processOrder(Order order, User user, String promoCode) {
    // Validate order
    if (order == null) throw new IllegalArgumentException("Order cannot be null");
    if (order.getItems().isEmpty()) throw new IllegalArgumentException("Order must have items");
    if (user == null) throw new IllegalArgumentException("User cannot be null");
    if (!user.isActive()) throw new IllegalStateException("User account is inactive");

    // Apply promo code
    double discount = 0;
    if (promoCode != null && !promoCode.isEmpty()) {
        PromoCode promo = promoCodeRepository.findByCode(promoCode);
        if (promo != null && promo.isValid() && !promo.isExpired()) {
            if (promo.getType().equals("PERCENTAGE")) {
                discount = order.getSubtotal() * promo.getValue() / 100;
            } else {
                discount = promo.getValue();
            }
            discount = Math.min(discount, order.getSubtotal()); // can't discount more than total
        }
    }

    // Calculate total
    double subtotal = 0;
    for (OrderItem item : order.getItems()) {
        double itemTotal = item.getPrice() * item.getQuantity();
        if (item.isOnSale()) {
            itemTotal *= (1 - item.getSalePercentage() / 100.0);
        }
        subtotal += itemTotal;
    }
    double total = subtotal - discount;
    double tax = total * 0.18;
    double grandTotal = total + tax;

    // Save order
    order.setSubtotal(subtotal);
    order.setDiscount(discount);
    order.setTax(tax);
    order.setTotal(grandTotal);
    order.setStatus("PENDING");
    order.setCreatedAt(LocalDateTime.now());
    orderRepository.save(order);

    // Send confirmation
    String subject = "Order Confirmation #" + order.getId();
    String body = "Dear " + user.getName() + ", your order has been placed.";
    emailService.sendEmail(user.getEmail(), subject, body);

    // Update inventory
    for (OrderItem item : order.getItems()) {
        inventoryService.decrease(item.getProductId(), item.getQuantity());
    }
}
```

**Refactoring: Extract Method**
```java
public void processOrder(Order order, User user, String promoCode) {
    validateOrder(order, user);
    double discount = calculateDiscount(order, promoCode);
    double grandTotal = calculateTotal(order, discount);
    finalizeOrder(order, grandTotal);
    orderRepository.save(order);
    sendConfirmationEmail(user, order);
    updateInventory(order);
}

private void validateOrder(Order order, User user) {
    if (order == null) throw new IllegalArgumentException("Order cannot be null");
    if (order.getItems().isEmpty()) throw new IllegalArgumentException("Order must have items");
    if (user == null) throw new IllegalArgumentException("User cannot be null");
    if (!user.isActive()) throw new IllegalStateException("User account is inactive");
}

private double calculateDiscount(Order order, String promoCode) {
    if (promoCode == null || promoCode.isEmpty()) return 0;
    PromoCode promo = promoCodeRepository.findByCode(promoCode);
    if (promo == null || !promo.isValid() || promo.isExpired()) return 0;
    double discount = promo.getType().equals("PERCENTAGE")
        ? order.getSubtotal() * promo.getValue() / 100
        : promo.getValue();
    return Math.min(discount, order.getSubtotal());
}

private double calculateTotal(Order order, double discount) {
    double subtotal = order.getItems().stream()
        .mapToDouble(this::calculateItemTotal)
        .sum();
    order.setSubtotal(subtotal);
    order.setDiscount(discount);
    double taxable = subtotal - discount;
    double tax = taxable * 0.18;
    order.setTax(tax);
    return taxable + tax;
}

private double calculateItemTotal(OrderItem item) {
    double total = item.getPrice() * item.getQuantity();
    return item.isOnSale() ? total * (1 - item.getSalePercentage() / 100.0) : total;
}
```

---

### Large Class

**Description:** A class that has accumulated too many responsibilities (too many fields, too many methods, too many lines).

**Detection:** Classes over 300 lines. Classes with many unrelated fields. High LCOM score.

**The smell:**
```java
public class UserManager {
    // User profile data
    private String firstName, lastName, email, phone;
    private LocalDate dateOfBirth;

    // Authentication data
    private String passwordHash;
    private int failedLoginAttempts;
    private LocalDateTime lastLoginTime;

    // Address data
    private String street, city, state, zipCode, country;

    // Loyalty program data
    private int loyaltyPoints;
    private String loyaltyTier;

    // Methods for profile
    public String getFullName() { ... }
    public void updateEmail(String email) { ... }

    // Methods for auth
    public boolean authenticate(String password) { ... }
    public void lockAccount() { ... }
    public void resetPassword(String newPassword) { ... }

    // Methods for address
    public String formatAddress() { ... }
    public boolean isAddressComplete() { ... }

    // Methods for loyalty
    public void addPoints(int points) { ... }
    public void upgradeTier() { ... }
    public double getLoyaltyDiscount() { ... }
}
```

**Refactoring: Extract Class**
```java
// Split into focused classes
public class UserProfile {
    private String firstName, lastName, email, phone;
    private LocalDate dateOfBirth;

    public String getFullName() { return firstName + " " + lastName; }
    public void updateEmail(String email) { this.email = email; }
}

public class UserCredentials {
    private String passwordHash;
    private int failedLoginAttempts;
    private LocalDateTime lastLoginTime;

    public boolean authenticate(String password) { ... }
    public void lockAccount() { ... }
    public void resetPassword(String newPassword) { ... }
}

public class UserAddress {
    private String street, city, state, zipCode, country;

    public String format() { return street + ", " + city + ", " + state; }
    public boolean isComplete() { return stream of fields not null/empty; }
}

public class LoyaltyAccount {
    private int points;
    private String tier;

    public void addPoints(int points) { ... }
    public void upgradeTier() { ... }
    public double getDiscount() { ... }
}

// User is now a thin coordinator
public class User {
    private final UserProfile profile;
    private final UserCredentials credentials;
    private final UserAddress address;
    private final LoyaltyAccount loyaltyAccount;
    // ...
}
```

---

### Primitive Obsession

**Description:** Using primitive types (String, int, double) to represent domain concepts that deserve their own classes.

**Detection:** Phone numbers as `String`, money as `double`, IDs as `long`, zip codes as `String`.

**The smell:**
```java
public class Order {
    private long customerId;             // ID — just a long
    private String customerEmail;        // email — just a String
    private double price;                // money as floating point
    private String promoCode;            // promo — just a String
    private String status;               // status — just a String
}

// This compiles but is wrong — money loses precision with double
double price = 99.99 + 0.01; // might not equal 100.0 exactly
```

**Refactoring: Replace Primitive with Value Object**
```java
public final class CustomerId {
    private final long value;
    public CustomerId(long value) {
        if (value <= 0) throw new IllegalArgumentException("Invalid customer ID");
        this.value = value;
    }
    public long getValue() { return value; }
}

public final class Email {
    private final String value;
    private static final Pattern PATTERN = Pattern.compile("^[^@]+@[^@]+\\.[^@]+$");

    public Email(String value) {
        if (!PATTERN.matcher(value).matches())
            throw new IllegalArgumentException("Invalid email: " + value);
        this.value = value.toLowerCase();
    }
    public String getValue() { return value; }
}

public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency;
    }

    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(this.amount.add(other.amount), currency);
    }

    public Money multiply(double factor) {
        return new Money(amount.multiply(BigDecimal.valueOf(factor)), currency);
    }
    // ...
}

public enum OrderStatus { PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED }

public class Order {
    private final CustomerId customerId;
    private final Email customerEmail;
    private Money price;
    private OrderStatus status;
    // Now the types carry their own validation and semantics
}
```

---

### Long Parameter List

**Description:** A method with more than 3-4 parameters. Hard to read at call sites, easy to pass arguments in the wrong order.

**The smell:**
```java
public Order createOrder(
    long customerId, String customerEmail,
    String productId, int quantity, double unitPrice,
    String promoCode, String shippingAddress,
    String paymentToken, boolean isGift) {
    // ...
}

// At call site — which boolean was isGift again?
createOrder(123, "alice@example.com", "PROD-456", 2, 29.99,
            "SUMMER10", "123 Main St", "tok_visa", true);
```

**Refactoring: Introduce Parameter Object**
```java
public class CreateOrderRequest {
    private final long customerId;
    private final String customerEmail;
    private final String productId;
    private final int quantity;
    private final double unitPrice;
    private final String promoCode;
    private final String shippingAddress;
    private final String paymentToken;
    private final boolean isGift;

    // Builder for readable construction
    public static Builder builder() { return new Builder(); }

    public static class Builder {
        // ... builder fields and fluent setters ...
        public CreateOrderRequest build() {
            validate();
            return new CreateOrderRequest(this);
        }
    }
}

public Order createOrder(CreateOrderRequest request) {
    // Clean — one parameter, all fields have names
}

// At call site — readable and safe
Order order = orderService.createOrder(
    CreateOrderRequest.builder()
        .customerId(123)
        .customerEmail("alice@example.com")
        .productId("PROD-456")
        .quantity(2)
        .unitPrice(29.99)
        .promoCode("SUMMER10")
        .shippingAddress("123 Main St")
        .paymentToken("tok_visa")
        .isGift(true)
        .build());
```

---

### Data Clumps

**Description:** Groups of data that always appear together should be their own class.

**The smell:**
```java
// Latitude, longitude, altitude always travel together
public void trackLocation(double latitude, double longitude, double altitude, long deviceId) { ... }
public void calculateDistance(double lat1, double lon1, double lat2, double lon2) { ... }
public void setHomeBase(double homeLatitude, double homeLongitude, double homeAltitude) { ... }
```

**Refactoring: Extract Class**
```java
public final class GeoCoordinate {
    private final double latitude;
    private final double longitude;
    private final double altitude;

    public GeoCoordinate(double latitude, double longitude, double altitude) {
        if (latitude < -90 || latitude > 90) throw new IllegalArgumentException("Invalid latitude");
        if (longitude < -180 || longitude > 180) throw new IllegalArgumentException("Invalid longitude");
        this.latitude = latitude;
        this.longitude = longitude;
        this.altitude = altitude;
    }

    public double distanceTo(GeoCoordinate other) {
        // Haversine formula
        double earthRadius = 6371e3; // meters
        double phi1 = Math.toRadians(this.latitude);
        double phi2 = Math.toRadians(other.latitude);
        double deltaPhi = Math.toRadians(other.latitude - this.latitude);
        double deltaLambda = Math.toRadians(other.longitude - this.longitude);

        double a = Math.sin(deltaPhi / 2) * Math.sin(deltaPhi / 2)
            + Math.cos(phi1) * Math.cos(phi2)
            * Math.sin(deltaLambda / 2) * Math.sin(deltaLambda / 2);
        return earthRadius * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    }
}

public void trackLocation(GeoCoordinate position, long deviceId) { ... }
public double calculateDistance(GeoCoordinate from, GeoCoordinate to) {
    return from.distanceTo(to);
}
```

---

## 2. Object-Orientation Abusers

These smells represent incorrect or incomplete application of OOP principles.

---

### Switch Statements

**Description:** Repeated switch or if-else chains on the same type discriminator. Each new type requires finding and updating every switch.

**The smell:**
```java
public class PaymentProcessor {
    public void process(Payment payment) {
        switch (payment.getType()) {
            case "CREDIT_CARD":
                String cardNumber = payment.getCardNumber();
                String cvv = payment.getCvv();
                chargeCard(cardNumber, cvv, payment.getAmount());
                break;
            case "PAYPAL":
                String paypalId = payment.getPaypalAccountId();
                chargePayPal(paypalId, payment.getAmount());
                break;
            case "BANK_TRANSFER":
                String accountNumber = payment.getBankAccountNumber();
                String routingNumber = payment.getRoutingNumber();
                initiateBankTransfer(accountNumber, routingNumber, payment.getAmount());
                break;
            default:
                throw new IllegalArgumentException("Unknown payment type: " + payment.getType());
        }
    }

    public double calculateFee(Payment payment) {
        switch (payment.getType()) { // Same switch again!
            case "CREDIT_CARD":  return payment.getAmount() * 0.03;
            case "PAYPAL":       return payment.getAmount() * 0.02 + 0.30;
            case "BANK_TRANSFER": return 1.50;
            default: throw new IllegalArgumentException("Unknown payment type");
        }
    }
}
```

**Refactoring: Replace Conditional with Polymorphism**
```java
public interface PaymentMethod {
    void process(double amount);
    double calculateFee(double amount);
}

public class CreditCardPayment implements PaymentMethod {
    private final String cardNumber;
    private final String cvv;

    public CreditCardPayment(String cardNumber, String cvv) {
        this.cardNumber = cardNumber;
        this.cvv = cvv;
    }

    @Override
    public void process(double amount) {
        chargeCard(cardNumber, cvv, amount);
    }

    @Override
    public double calculateFee(double amount) {
        return amount * 0.03;
    }
}

public class PayPalPayment implements PaymentMethod {
    private final String accountId;

    @Override
    public void process(double amount) {
        chargePayPal(accountId, amount);
    }

    @Override
    public double calculateFee(double amount) {
        return amount * 0.02 + 0.30;
    }
}

public class BankTransferPayment implements PaymentMethod {
    private final String accountNumber;
    private final String routingNumber;

    @Override
    public void process(double amount) {
        initiateBankTransfer(accountNumber, routingNumber, amount);
    }

    @Override
    public double calculateFee(double amount) {
        return 1.50;
    }
}

// No switch statement — polymorphic dispatch
public class PaymentProcessor {
    public void process(PaymentMethod method, double amount) {
        method.process(amount);
    }

    public double calculateFee(PaymentMethod method, double amount) {
        return method.calculateFee(amount);
    }
}
```

Now adding a new payment type (e.g., Cryptocurrency) requires only a new class. Zero changes to existing code.

---

### Temporary Field

**Description:** An instance field that is only set and used in certain circumstances. When not in use, it is null or has a default value that means nothing.

**The smell:**
```java
public class OrderProcessor {
    // These are only meaningful during batch processing, null otherwise
    private List<Order> currentBatch;
    private int batchSize;
    private String batchId;
    private LocalDateTime batchStartTime;

    // These are always meaningful
    private final OrderRepository repository;
    private final EmailService emailService;

    public void processBatch(List<Order> orders) {
        // Set temp fields
        this.currentBatch = orders;
        this.batchSize = orders.size();
        this.batchId = UUID.randomUUID().toString();
        this.batchStartTime = LocalDateTime.now();

        processEachOrder();
        logBatchCompletion();

        // Clear temp fields
        this.currentBatch = null;
        this.batchSize = 0;
        this.batchId = null;
        this.batchStartTime = null;
    }
}
```

**Refactoring: Extract Class (Batch Context)**
```java
// The batch-related state belongs to its own class
public class BatchContext {
    private final String batchId;
    private final LocalDateTime startTime;
    private final List<Order> orders;

    public BatchContext(List<Order> orders) {
        this.batchId = UUID.randomUUID().toString();
        this.startTime = LocalDateTime.now();
        this.orders = List.copyOf(orders);
    }

    public String getBatchId()            { return batchId; }
    public LocalDateTime getStartTime()   { return startTime; }
    public List<Order> getOrders()        { return orders; }
    public int getSize()                  { return orders.size(); }
}

public class OrderProcessor {
    private final OrderRepository repository;
    private final EmailService emailService;

    public void processBatch(List<Order> orders) {
        BatchContext context = new BatchContext(orders);
        context.getOrders().forEach(order -> processOrder(order, context));
        logBatchCompletion(context);
    }
}
```

---

### Refused Bequest

**Description:** A subclass inherits methods it does not use or overrides them to throw `UnsupportedOperationException`. The subclass refuses part of the inheritance.

**The smell:**
```java
public class Bird {
    public void fly() { System.out.println("Flying..."); }
    public void eat() { System.out.println("Eating..."); }
    public void makeSound() { System.out.println("Chirp!"); }
}

public class Penguin extends Bird {
    @Override
    public void fly() {
        throw new UnsupportedOperationException("Penguins cannot fly"); // LSP violation
    }
}
```

**Refactoring: Replace Inheritance with Composition or Restructure Hierarchy**
```java
public interface Eatable { void eat(); }
public interface Flyable  { void fly(); }
public interface Vocal    { void makeSound(); }

public abstract class Bird implements Eatable, Vocal {
    @Override
    public void eat() { System.out.println("Eating..."); }
}

// Flying birds also implement Flyable
public class Sparrow extends Bird implements Flyable {
    @Override
    public void fly()       { System.out.println("Flying with wings!"); }
    @Override
    public void makeSound() { System.out.println("Chirp!"); }
}

// Penguin does not implement Flyable — no refused bequest
public class Penguin extends Bird {
    @Override
    public void makeSound() { System.out.println("Squawk!"); }
    public void swim()      { System.out.println("Swimming!"); }
}
```

---

### Alternative Classes with Different Interfaces

**Description:** Two classes do the same thing but have different method names or signatures. Code cannot treat them interchangeably.

**The smell:**
```java
public class SmtpMailer {
    public void sendEmail(String recipient, String title, String content) { ... }
}

public class SendGridClient {
    public void dispatch(String to, String subject, String body) { ... }
}
// Same concept, different vocabulary. Cannot swap one for the other.
```

**Refactoring: Extract Interface**
```java
public interface EmailSender {
    void send(String recipient, String subject, String body);
}

public class SmtpMailer implements EmailSender {
    @Override
    public void send(String recipient, String subject, String body) { ... }
}

public class SendGridClient implements EmailSender {
    @Override
    public void send(String recipient, String subject, String body) { ... }
}
// Now both are interchangeable via the interface.
```

---

## 3. Change Preventers

These smells make it difficult to change code — a change in one place forces changes in many others.

---

### Divergent Change

**Description:** A class must be changed for multiple different reasons. It violates SRP — it has more than one responsibility.

**Detection:** "I need to change this class when we change the database AND when we change the report format AND when we change the notification logic."

**The smell:**
```java
public class CustomerService {
    // Must change when database schema changes
    public Customer findCustomer(long id) {
        String sql = "SELECT id, first_name, last_name, email FROM customers WHERE id = ?";
        // ...
    }

    // Must change when report format changes
    public String generateReport(Customer customer) {
        return "Customer: " + customer.getName() + " | Email: " + customer.getEmail();
    }

    // Must change when notification logic changes
    public void notifyCustomer(Customer customer, String message) {
        // smtp logic
    }
}
```

**Fix:** Split by responsibility — each class changes for one reason only.

---

### Shotgun Surgery

**Description:** A single change requires making many small changes in many different classes. The opposite of Divergent Change.

**Detection:** "To add a new currency, I need to update the OrderService, the InvoiceService, the ReportService, and the CurrencyConverter."

**The smell:**
```java
// Changing the tax rate requires updating all of these:
public class OrderService {
    private static final double TAX = 0.18; // change here
}

public class InvoiceService {
    private static final double TAX_RATE = 0.18; // and here
}

public class QuoteService {
    private static final double GST = 0.18; // and here
}

public class ReportService {
    // hardcoded: total * 1.18 everywhere
}
```

**Refactoring: Move Method / Inline Class — consolidate into one place**
```java
// Single source of truth for tax configuration
public class TaxPolicy {
    private final double rate;

    public static TaxPolicy standard() {
        return new TaxPolicy(0.18);
    }

    public TaxPolicy(double rate) {
        this.rate = rate;
    }

    public double calculateTax(double amount)    { return amount * rate; }
    public double calculateTotal(double amount)  { return amount * (1 + rate); }
    public double getRate()                      { return rate; }
}

// All services inject TaxPolicy — one change propagates everywhere
public class OrderService {
    private final TaxPolicy taxPolicy;
    // ...
}
```

---

### Parallel Inheritance Hierarchies

**Description:** Every time you create a subclass in one hierarchy, you must create a corresponding subclass in another.

**The smell:**
```java
// Animal hierarchy
abstract class Animal { ... }
class Dog extends Animal { ... }
class Cat extends Animal { ... }

// Parallel sound hierarchy — every animal needs a corresponding sound class
abstract class AnimalSound { ... }
class DogSound extends AnimalSound { ... }  // parallel to Dog
class CatSound extends AnimalSound { ... }  // parallel to Cat
```

**Fix:** Merge the hierarchies or use composition.
```java
public interface SoundBehavior {
    void makeSound();
}

public class Animal {
    private final SoundBehavior sound;

    public Animal(SoundBehavior sound) { this.sound = sound; }
    public void makeSound()            { sound.makeSound(); }
}

Animal dog = new Animal(() -> System.out.println("Woof!"));
Animal cat = new Animal(() -> System.out.println("Meow!"));
```

---

## 4. Dispensables

Things that do not need to exist. Removing them makes the code cleaner.

---

### Lazy Class

**Description:** A class that does too little to justify its existence.

```java
// Entire class is a wrapper with no added value
public class UserEmailSender {
    private final EmailService emailService;

    public UserEmailSender(EmailService emailService) {
        this.emailService = emailService;
    }

    public void send(User user, String message) {
        emailService.send(user.getEmail(), message); // pure delegation, no logic added
    }
}
// Remove this class — call emailService directly.
```

---

### Speculative Generality

**Description:** Code that is written "for future use" but serves no current purpose.

```java
// Nobody uses these parameters today
public interface Sorter<T, K extends Comparable<K>> {
    List<T> sort(List<T> items, Comparator<T> comparator, boolean ascending,
                 int limit, int offset, boolean stable); // stable sort never used
}

// The real requirement: sort a list
// Just write: list.stream().sorted().collect(Collectors.toList())
```

---

### Data Class

**Description:** A class with only fields, getters, and setters. No behavior. It exists only to carry data between other classes.

```java
// Data class — a bag of getters and setters with no behavior
public class CustomerDto {
    private String firstName;
    private String lastName;
    private String email;
    private String phone;
    // 10 getters, 10 setters, no methods that do anything meaningful
}
```

**When it's a smell vs when it's fine:**

Data classes that cross architectural boundaries (DTOs between service and controller) are acceptable. Data classes inside the domain that displace behavior from their natural owner are the smell.

**Fix:** Move behavior to where the data lives.
```java
public class CustomerDto {
    private final String firstName;
    private final String lastName;
    private final String email;

    // Behavior belongs here, not scattered elsewhere
    public String getFullName()      { return firstName + " " + lastName; }
    public boolean hasContactInfo()  { return email != null && !email.isEmpty(); }
}
```

---

### Duplicate Code

The quintessential smell. Same logic in multiple places. See the DRY principle chapter for detailed coverage and examples.

---

### Dead Code

**Description:** Code that is never executed — unreachable methods, unused variables, conditions that are always false.

```java
public class OrderService {
    private static final String LEGACY_ENDPOINT = "http://old-api.company.com"; // never used

    public void placeOrder(Order order) {
        if (false) { // unreachable
            sendToLegacySystem(order);
        }
        // ...
    }

    private void sendToLegacySystem(Order order) { // never called
        // 50 lines of code that execute 0 times
    }
}
```

**Fix:** Delete it. Version control has history. Keeping dead code costs cognitive load every time someone reads past it.

---

### Comments

**Description:** When comments are needed to explain what code does, it usually means the code is not clear enough.

```java
// Bad: comment explains what the code is doing (the code should explain itself)
// Check if user is over 18 and account is not locked
if (user.getAge() >= 18 && !user.isLocked()) {

// Bad: commented-out code
// user.setStatus("PENDING"); // we removed this but maybe we'll need it
// orderRepository.archive(order);

// Good: comment explains WHY, not WHAT
// We use 3 retries based on empirical testing — at 4+ retries the benefit
// diminishes while latency for failed calls becomes unacceptable.
private static final int MAX_RETRIES = 3;
```

---

## 5. Couplers

Smells that create unwanted coupling between classes.

---

### Feature Envy

**Description:** A method that uses more of the data and methods of another class than its own.

**The smell:**
```java
public class OrderItem {
    private Product product;
    private int quantity;

    // This method envies the Product class — it accesses product's data heavily
    public double calculateTotal() {
        double basePrice = product.getBasePrice();
        double taxRate = product.getTaxRate();
        String category = product.getCategory();

        double price = basePrice;
        if (category.equals("ELECTRONICS")) {
            price = basePrice * (1 - product.getElectronicsDiscount());
        } else if (category.equals("FOOD")) {
            price = basePrice * (1 - product.getFoodDiscount());
        }

        return price * quantity * (1 + taxRate);
    }
}
```

**Refactoring: Move Method**
```java
public class Product {
    private double basePrice;
    private double taxRate;
    private String category;
    private double discount;

    // Behavior about Product pricing belongs in Product
    public double getPriceAfterDiscount() {
        return basePrice * (1 - discount);
    }

    public double getPriceWithTax() {
        return getPriceAfterDiscount() * (1 + taxRate);
    }
}

public class OrderItem {
    private Product product;
    private int quantity;

    public double calculateTotal() {
        return product.getPriceWithTax() * quantity; // clean — no envy
    }
}
```

---

### Inappropriate Intimacy

**Description:** Two classes are too deeply familiar with each other's internals — they access each other's private or package-private members more than their own.

**The smell:**
```java
public class Order {
    List<OrderItem> items; // package-private — accessible to "friends"
    double totalCache;
    boolean isTotalDirty;
}

public class OrderPrinter {
    public void print(Order order) {
        // Reaching into Order's internals
        if (order.isTotalDirty) {
            order.totalCache = 0;
            for (OrderItem item : order.items) { // accessing package-private field
                order.totalCache += item.getPrice() * item.getQuantity();
            }
            order.isTotalDirty = false;
        }
        System.out.println("Total: " + order.totalCache);
    }
}
```

**Fix:** Encapsulate the behavior in the class that owns the data.
```java
public class Order {
    private final List<OrderItem> items;
    private Double cachedTotal;

    public double getTotal() {
        if (cachedTotal == null) {
            cachedTotal = items.stream()
                .mapToDouble(item -> item.getPrice() * item.getQuantity())
                .sum();
        }
        return cachedTotal;
    }

    public void addItem(OrderItem item) {
        items.add(item);
        cachedTotal = null; // invalidate cache
    }
}

public class OrderPrinter {
    public void print(Order order) {
        System.out.println("Total: " + order.getTotal()); // clean interface
    }
}
```

---

### Message Chains

**Description:** A chain of method calls: `a.getB().getC().getD().getValue()`. The caller is navigating the internal structure of objects — it is coupled to the whole chain.

**The smell:**
```java
public class InvoiceGenerator {
    public String generateInvoice(Order order) {
        // If any link in this chain changes, this code breaks
        String country = order.getCustomer().getAddress().getCountry().getCode();
        String currency = order.getCustomer().getAccount().getCurrency().getSymbol();
        String taxId = order.getCustomer().getTaxProfile().getPrimaryTaxId().getValue();
        // ...
    }
}
```

**Refactoring: Hide Delegate / Move Method (Law of Demeter)**
```java
// Law of Demeter: a method should only call methods on:
// - itself
// - its parameters
// - objects it creates
// - its direct fields

public class Order {
    private Customer customer;

    // Order provides what callers need directly
    public String getCustomerCountryCode() {
        return customer.getAddress().getCountry().getCode();
    }

    public String getCustomerCurrencySymbol() {
        return customer.getAccount().getCurrency().getSymbol();
    }
}

public class InvoiceGenerator {
    public String generateInvoice(Order order) {
        String country = order.getCustomerCountryCode();    // one link
        String currency = order.getCustomerCurrencySymbol(); // one link
        // ...
    }
}
```

---

### Middle Man

**Description:** A class that does nothing but delegate to another class. The opposite of Feature Envy — too much delegation, no own behavior.

**The smell:**
```java
public class UserService {
    private UserRepository userRepository;

    // Every method just passes through to userRepository
    public User findById(long id)         { return userRepository.findById(id); }
    public User save(User user)           { return userRepository.save(user); }
    public void delete(long id)           { userRepository.delete(id); }
    public List<User> findAll()           { return userRepository.findAll(); }
    public boolean exists(long id)        { return userRepository.exists(id); }
    // Zero business logic — pure pass-through
}
```

**Fix:** Remove the middle man. Let callers call `UserRepository` directly. If `UserService` adds no value, it should not exist.

---

# 6. Core Refactoring Techniques

These are the techniques referenced throughout the smell catalog. Each preserves external behavior while improving internal structure.

---

### Extract Method

Split a long method into smaller ones with descriptive names.

```java
// Before
public void sendMonthlyReport(List<Order> orders) {
    // Calculate totals
    double total = 0;
    for (Order o : orders) { total += o.getTotal(); }
    double average = total / orders.size();

    // Format report
    StringBuilder sb = new StringBuilder();
    sb.append("Monthly Report\n");
    sb.append("Total: ").append(total).append("\n");
    sb.append("Average: ").append(average).append("\n");

    // Send email
    emailService.send("finance@company.com", "Monthly Report", sb.toString());
}

// After
public void sendMonthlyReport(List<Order> orders) {
    ReportData data = calculateReportData(orders);
    String report = formatReport(data);
    emailService.send("finance@company.com", "Monthly Report", report);
}

private ReportData calculateReportData(List<Order> orders) {
    double total = orders.stream().mapToDouble(Order::getTotal).sum();
    return new ReportData(total, total / orders.size());
}

private String formatReport(ReportData data) {
    return String.format("Monthly Report%nTotal: %.2f%nAverage: %.2f%n",
        data.getTotal(), data.getAverage());
}
```

---

### Extract Class

Move a group of related fields and methods out of a large class into a new one.

```java
// Before: Person has both name and geo-location concerns
public class Person {
    private String firstName, lastName;
    private double officeLatitude, officeLongitude;
    private double homeLatitude, homeLongitude;

    public String getFullName() { return firstName + " " + lastName; }
    public double getDistanceBetweenOfficeAndHome() {
        // haversine calculation using all four lat/long fields
    }
}

// After: Extract TelephoneNumber analogy — extract Location
public class GeoPoint {
    private final double latitude;
    private final double longitude;

    public double distanceTo(GeoPoint other) { /* haversine */ }
}

public class Person {
    private String firstName, lastName;
    private GeoPoint office;
    private GeoPoint home;

    public String getFullName()               { return firstName + " " + lastName; }
    public double getCommuteDistance()        { return office.distanceTo(home); }
}
```

---

### Move Method

Move a method to the class whose data it uses most.

```java
// Before: calculateRentalCost in Rental uses Movie data heavily
public class Rental {
    private Movie movie;
    private int daysRented;

    public double calculateRentalCost() { // feature envy on Movie
        double result = 0;
        switch (movie.getPriceCode()) {
            case Movie.REGULAR:
                result += 2;
                if (daysRented > 2) result += (daysRented - 2) * 1.5;
                break;
            case Movie.NEW_RELEASE:
                result += daysRented * 3;
                break;
        }
        return result;
    }
}

// After: move to Movie (or better: polymorphism on Movie)
public abstract class Movie {
    public abstract double getRentalCost(int daysRented);
}

public class RegularMovie extends Movie {
    @Override
    public double getRentalCost(int daysRented) {
        double cost = 2;
        if (daysRented > 2) cost += (daysRented - 2) * 1.5;
        return cost;
    }
}

public class NewReleaseMovie extends Movie {
    @Override
    public double getRentalCost(int daysRented) {
        return daysRented * 3;
    }
}

public class Rental {
    private Movie movie;
    private int daysRented;

    public double calculateRentalCost() {
        return movie.getRentalCost(daysRented); // delegated
    }
}
```

---

### Replace Conditional with Polymorphism

Replace a switch or if-else chain with polymorphic dispatch.

Covered in the Switch Statements smell above. The pattern:
1. Identify the discriminator (the type string or enum being switched on)
2. Create an interface with the behavior being dispatched
3. Create one implementation per case
4. Remove the switch — callers work through the interface

---

### Introduce Parameter Object

Replace a long parameter list with a single object.

Covered in the Long Parameter List smell above.

---

### Replace Magic Number with Symbolic Constant

Replace a literal value with a named constant that conveys meaning.

```java
// Before
public double calculateArea(double radius) {
    return 3.14159 * radius * radius; // what is 3.14159? why this precision?
}

// Before
if (status == 3) { // what does 3 mean?
    processRefund();
}

// After
public double calculateArea(double radius) {
    return Math.PI * radius * radius; // clear — standard library constant
}

// After
public enum OrderStatus {
    PENDING(1), CONFIRMED(2), REFUNDED(3), SHIPPED(4), DELIVERED(5);
    private final int code;
    OrderStatus(int code) { this.code = code; }
}

if (status == OrderStatus.REFUNDED) {
    processRefund();
}
```

---

### Encapsulate Field

Make public fields private and provide getters/setters with appropriate access control.

```java
// Before
public class Account {
    public double balance; // public — anyone can set this directly
}

// Someone can do: account.balance = -999999; // bypasses all validation

// After
public class Account {
    private double balance;

    public double getBalance() { return balance; }

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        balance += amount;
    }

    public void withdraw(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        if (amount > balance) throw new IllegalStateException("Insufficient funds");
        balance -= amount;
    }
    // No setter — balance can only change through deposit/withdraw
}
```

---

### Hide Delegate

Introduce a method on the owning class that shields callers from a chain of delegation.

```java
// Before: caller must navigate the object graph
public class Client {
    public void useManager(Department dept) {
        Person manager = dept.getManager();
        manager.increasePayGrade();
    }
}

// After: Department provides a convenience method
public class Department {
    private Person manager;

    public Person getManager() { return manager; }

    // Hides the delegate from external clients
    public void increaseManagerPayGrade() {
        manager.increasePayGrade();
    }
}

public class Client {
    public void useManager(Department dept) {
        dept.increaseManagerPayGrade(); // one call, no chain
    }
}
```

---

## The Refactoring Mindset

These are the principles that govern all refactoring work:

1. **Red-Green-Refactor.** Write a failing test. Make it pass. Refactor. The test ensures you have not broken behavior.

2. **Small steps.** Each refactoring step should be small enough that if something breaks, you know exactly what caused it.

3. **One thing at a time.** Do not rename variables, extract methods, and move classes in the same commit. Pick one.

4. **The boy scout rule.** Leave the code cleaner than you found it. Each time you touch a file, fix one small smell — not because you were asked to, but because it is the right thing to do.

5. **DAMP tests.** Tests should be Descriptive And Meaningful Phrases. They document the intended behavior and will alert you if refactoring breaks it.
