# 100 LLD Interview Questions — Senior Software Engineer Preparation

This reference covers 110 questions across six categories plus a bonus senior/staff-level section. Every question includes a complete model answer at Senior Engineer depth with Java code where it materially illustrates the concept.

---

## Category 1: OOP Fundamentals

### Q1. What is encapsulation and why does it matter beyond just getters and setters?

**Model Answer:** Encapsulation is the bundling of data and the methods that operate on that data into a single unit (the class), with controlled access to the internal state. Its real purpose is to enforce invariants: by making fields private and exposing only behavior-oriented methods, you guarantee the object is always in a valid state — getters and setters are a side effect, not the goal. A `BankAccount` that exposes `setBalance(double)` has no encapsulation at all; one that exposes `deposit(double amount)` and `withdraw(double amount)` can validate constraints in those methods and maintain the invariant that balance never goes negative. The key insight is that the unit of encapsulation is the *class invariant*, not the field.

```java
// Poor encapsulation — exposes raw state
public class BankAccount {
    public double balance;
}

// Proper encapsulation — exposes behavior, enforces invariant
public class BankAccount {
    private double balance;
    private final String accountId;

    public BankAccount(String accountId, double initialBalance) {
        if (initialBalance < 0) throw new IllegalArgumentException("Initial balance cannot be negative");
        this.accountId = accountId;
        this.balance = initialBalance;
    }

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Deposit amount must be positive");
        this.balance += amount;
    }

    public void withdraw(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        if (amount > balance) throw new IllegalStateException("Insufficient funds");
        this.balance -= amount;
    }

    public double getBalance() { return balance; }
}
```

---

### Q2. What is the difference between abstraction and encapsulation?

**Model Answer:** Abstraction is about hiding *complexity* — exposing only what is relevant to the caller and suppressing implementation details, operating at the design/interface level. Encapsulation is about hiding *state* — protecting the internal data of an object from direct external manipulation, operating at the implementation level. Abstraction answers "what does this do?"; encapsulation answers "how is the internal state protected?". You can have abstraction without encapsulation (an interface with no implementation), and encapsulation without meaningful abstraction (a class with all fields private but a getter for every field), but in well-designed systems both work together.

```java
// Abstraction: the caller knows only that Shape has an area — not how it's computed
public interface Shape {
    double area();
    double perimeter();
}

// Encapsulation inside the concrete class: radius is private, invariant enforced
public class Circle implements Shape {
    private final double radius; // encapsulated state

    public Circle(double radius) {
        if (radius <= 0) throw new IllegalArgumentException("Radius must be positive");
        this.radius = radius;
    }

    @Override
    public double area() { return Math.PI * radius * radius; } // abstraction

    @Override
    public double perimeter() { return 2 * Math.PI * radius; }
}
```

---

### Q3. When should you use inheritance and when should you prefer composition?

**Model Answer:** Inheritance is appropriate only when a true "is-a" relationship exists across the *entire* lifetime of the object and the subclass can substitute the parent without behavioral surprises (Liskov Substitution Principle). Composition is preferred when you want to reuse *behavior* rather than claim a type relationship, when the relationship is "has-a", or when the behavior you need changes at runtime. A classic mistake is extending `ArrayList` to make a `UserList` just to reuse `add()`/`remove()` — this exposes 30+ unintended methods and tightly couples `UserList` to `ArrayList`'s internal contract. Prefer composition: hold a `List<User>` as a field and delegate only the methods you intend to expose.

```java
// Inheritance misuse: UserList is NOT-A ArrayList in any meaningful domain sense
public class UserList extends ArrayList<User> {
    public void addVerifiedUser(User u) {
        if (u.isVerified()) add(u); // leaks 29 other ArrayList methods
    }
}

// Composition: clean, controlled API
public class UserList {
    private final List<User> users = new ArrayList<>();

    public void addVerifiedUser(User u) {
        if (u.isVerified()) users.add(u);
    }

    public List<User> getAll() {
        return Collections.unmodifiableList(users);
    }

    public int size() { return users.size(); }
}
```

---

### Q4. What is the difference between runtime polymorphism and compile-time polymorphism in Java?

**Model Answer:** Compile-time polymorphism (static dispatch) is resolved by the compiler based on the method signature — it is implemented through method overloading, where multiple methods share a name but differ in parameter type or count. Runtime polymorphism (dynamic dispatch) is resolved at runtime based on the actual type of the object — it is implemented through method overriding, where a subclass provides a different implementation of a parent method, and the JVM uses the vtable to dispatch to the correct implementation. The practical significance is that compile-time polymorphism cannot vary behavior based on the runtime type of the object, while runtime polymorphism is the mechanism behind the Strategy and Template Method patterns.

```java
// Compile-time polymorphism (overloading)
public class Printer {
    public void print(String s)  { System.out.println("String: " + s); }
    public void print(int i)     { System.out.println("Int: " + i); }
    public void print(double d)  { System.out.println("Double: " + d); }
}

// Runtime polymorphism (overriding)
public abstract class Animal {
    public abstract String sound();
}

public class Dog extends Animal {
    @Override public String sound() { return "Woof"; }
}

public class Cat extends Animal {
    @Override public String sound() { return "Meow"; }
}

// Dynamic dispatch: actual type determines which sound() is called
Animal a = new Dog();
System.out.println(a.sound()); // prints "Woof" — resolved at runtime
```

---

### Q5. Explain the "favor composition over inheritance" principle with a concrete scenario.

**Model Answer:** The principle exists because inheritance creates a static, compile-time coupling between child and parent: any change to the parent class risks breaking the child, the child is forced to inherit all methods (even irrelevant ones), and behavior cannot be changed at runtime. Composition lets you assemble behavior from interchangeable parts, swap implementations at runtime, and avoid the fragile base class problem. Consider a `Logger` that needs to write to a file or a database: using inheritance forces you into a single chain (`FileLogger extends Logger`), but with composition you inject a `LogWriter` strategy that can be swapped without touching `Logger`. This also directly enables the Open/Closed Principle.

```java
// Inheritance: tight coupling, cannot swap at runtime
public class FileLogger extends Logger {
    @Override protected void writeLog(String msg) { /* write to file */ }
}

// Composition: flexible, testable, supports runtime swapping
public interface LogWriter {
    void write(String message);
}

public class FileLogWriter implements LogWriter {
    @Override public void write(String message) { /* write to file */ }
}

public class DatabaseLogWriter implements LogWriter {
    @Override public void write(String message) { /* write to DB */ }
}

public class Logger {
    private final LogWriter writer;

    public Logger(LogWriter writer) { this.writer = writer; } // injected

    public void log(String message) {
        writer.write("[" + Instant.now() + "] " + message);
    }
}
```

---

### Q6. What is the difference between method overriding and method overloading, and when is each appropriate?

**Model Answer:** Overriding provides a subclass-specific implementation of a method already defined in a parent class — same name, same parameter list, same or covariant return type, and resolved at runtime via dynamic dispatch. Overloading provides multiple methods with the same name but different parameter signatures in the same class — resolved at compile time. Override when a subtype needs to specialize inherited behavior (polymorphism). Overload when the same logical operation applies to different input types. A common gotcha with overloading is that the resolution uses the *declared* (static) type, not the runtime type, which can produce surprising results when a variable is declared as a supertype.

```java
// Overriding: resolved at runtime
public class Vehicle {
    public String describe() { return "I am a vehicle"; }
}

public class Car extends Vehicle {
    @Override
    public String describe() { return "I am a car"; } // JVM picks this based on actual type
}

// Overloading: resolved at compile time
public class MathUtils {
    public int add(int a, int b) { return a + b; }
    public double add(double a, double b) { return a + b; }
    public int add(int a, int b, int c) { return a + b + c; }
}

// Overloading gotcha with declared type
Vehicle v = new Car();
v.describe();          // "I am a car" — overriding uses runtime type
// But for overloading:
public class Processor {
    public void process(Object o) { System.out.println("Object"); }
    public void process(String s) { System.out.println("String"); }
}
Object o = "hello";
new Processor().process(o); // prints "Object" — compile-time type is Object
```

---

### Q7. When should you choose an interface over an abstract class in Java, and how do Java 8 default methods change the calculus?

**Model Answer:** Use an interface when you want to define a behavioral contract that multiple unrelated classes can implement — Java supports only single inheritance, so interfaces allow a class to commit to multiple contracts. Use an abstract class when you want to share implementation across a hierarchy of related types, particularly when that shared code relies on common state fields. Java 8 default methods blurred this by allowing interfaces to carry default implementations, enabling mixin-style composition. However, abstract classes remain the right choice when you need constructors, non-public methods, or instance fields to hold shared state; interfaces cannot have these.

```java
// Interface: contract for unrelated types (Flyable applies to Bird and Drone)
public interface Flyable {
    void takeOff();
    void land();

    // Java 8 default method: shared behavior without abstract class
    default void hover() {
        System.out.println("Hovering in place");
    }
}

// Abstract class: shared state + partial implementation for a related hierarchy
public abstract class AbstractReport {
    protected final String title;
    protected final LocalDate generatedAt;

    protected AbstractReport(String title) {
        this.title = title;
        this.generatedAt = LocalDate.now();
    }

    public abstract String generateContent();

    public final String render() {
        return "=== " + title + " (" + generatedAt + ") ===\n" + generateContent();
    }
}
```

---

### Q8. What are covariant return types in Java and what do they enable?

**Model Answer:** A covariant return type allows an overriding method to declare a return type that is a subtype of the return type declared in the parent method. Java has supported this since Java 5. It enables fluent APIs, builder patterns, and more type-safe factory hierarchies without requiring unsafe casts at the call site. The most common use is in builder inheritance: a child builder's `build()` or `self()` method can return the child type instead of the parent type, so the caller gets full IDE autocomplete and type safety.

```java
// Covariant return type in a builder hierarchy
public class Animal {
    protected String name;

    public Animal build() { return this; } // returns Animal
}

public class Dog extends Animal {
    private String breed;

    @Override
    public Dog build() { return this; } // covariant: Dog is a subtype of Animal

    public Dog withBreed(String breed) {
        this.breed = breed;
        return this;
    }
}

// At the call site — no cast needed
Dog d = new Dog().withBreed("Labrador").build(); // type-safe
```

---

### Q9. What is the equals() and hashCode() contract in Java and what are the common pitfalls?

**Model Answer:** The contract states: if `a.equals(b)` is true, then `a.hashCode()` must equal `b.hashCode()`; the converse need not hold. Breaking this contract causes objects to "disappear" from `HashMap` or `HashSet` — you can insert an object and then fail to find it because the hashCode used for lookup differs from the one used for insertion. Common pitfalls include implementing `equals()` but forgetting to override `hashCode()`, using mutable fields in `hashCode()` computation (so the hash changes after the object is stored), and using `==` inside `equals()` instead of structural comparison.

```java
public class UserId {
    private final String value;

    public UserId(String value) {
        this.value = Objects.requireNonNull(value);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof UserId)) return false;
        UserId other = (UserId) o;
        return Objects.equals(value, other.value);
    }

    @Override
    public int hashCode() {
        return Objects.hash(value); // must use same field(s) as equals
    }
}

// Pitfall demo: mutable hashCode
public class BadKey {
    public String name; // mutable!

    @Override public int hashCode() { return name.hashCode(); }
    @Override public boolean equals(Object o) { /* ... */ return false; }
}

Map<BadKey, String> map = new HashMap<>();
BadKey k = new BadKey(); k.name = "Alice";
map.put(k, "value");
k.name = "Bob"; // hash changes — key is now lost in the map
map.get(k);     // returns null
```

---

### Q10. How do you design a truly immutable class in Java, including the edge case of mutable fields?

**Model Answer:** A truly immutable class requires: all fields `private final`; no setters; the class itself declared `final` (or all methods `final`) to prevent subclass mutation; and — critically — defensive copies of any mutable fields both in the constructor (to guard against the caller mutating the passed-in object) and in the getter (to guard against the caller mutating the returned object). Failing to make defensive copies is the single most common mistake when attempting immutability.

```java
public final class ImmutableOrder {
    private final String orderId;
    private final List<String> items;          // List is mutable!
    private final LocalDateTime createdAt;     // LocalDateTime is already immutable

    public ImmutableOrder(String orderId, List<String> items) {
        this.orderId = Objects.requireNonNull(orderId);
        // Defensive copy IN the constructor: caller's list changes won't affect us
        this.items = List.copyOf(items); // Java 10+ unmodifiable copy
        this.createdAt = LocalDateTime.now();
    }

    public String getOrderId() { return orderId; }

    // Defensive copy OUT in the getter: caller cannot mutate our internal list
    public List<String> getItems() {
        return Collections.unmodifiableList(items);
        // or: return new ArrayList<>(items);
    }

    public LocalDateTime getCreatedAt() { return createdAt; }
}
```

---

### Q11. What is the difference between shallow copy and deep copy in Java, and when does Cloneable fall short?

**Model Answer:** A shallow copy duplicates the object's fields by value: for primitive fields this means independent copies, but for reference fields it means both the original and the copy point to the *same* underlying objects. A deep copy recursively duplicates all referenced objects so the copy is entirely independent. `Cloneable` in Java is poorly designed: it is a marker interface with no `clone()` method, `Object.clone()` performs a shallow copy by default, and overriding it to do deep copy is verbose and error-prone, especially with inheritance hierarchies. The better alternatives are a copy constructor, a static factory `copy()` method, or serialization for complex graphs.

```java
// Shallow copy pitfall
public class Team implements Cloneable {
    private String name;
    private List<String> members; // mutable shared reference

    @Override
    public Team clone() throws CloneNotSupportedException {
        return (Team) super.clone(); // shallow! members list is shared
    }
}

// Deep copy via copy constructor (preferred)
public class Team {
    private final String name;
    private final List<String> members;

    // Copy constructor — deep copy
    public Team(Team other) {
        this.name = other.name;
        this.members = new ArrayList<>(other.members); // independent copy
    }
}
```

---

### Q12. Describe the object lifecycle in Java including creation, initialization, and destruction. How does finalize() compare to Cleaner?

**Model Answer:** An object's lifecycle begins with `new`: memory is allocated on the heap, fields are zero-initialized, instance initializer blocks run, then the constructor body executes. The object lives as long as there are strong references to it; when unreachable, it becomes eligible for garbage collection. `finalize()` was the pre-Java 9 hook for cleanup before GC, but it is non-deterministic, slows GC, can resurrect objects, and was deprecated in Java 9 and removed in Java 18. The `java.lang.ref.Cleaner` API (Java 9+) is the proper alternative: it registers a cleaning action (a `Runnable`) that runs when the object becomes phantom-reachable, executing in a dedicated thread and never allowing object resurrection.

```java
import java.lang.ref.Cleaner;

public class NativeResource implements AutoCloseable {
    private static final Cleaner CLEANER = Cleaner.create();

    private final long nativeHandle;
    private final Cleaner.Cleanable cleanable;

    public NativeResource(long handle) {
        this.nativeHandle = handle;
        // The state object must NOT hold a reference to NativeResource (would prevent GC)
        CleaningAction action = new CleaningAction(handle);
        this.cleanable = CLEANER.register(this, action);
    }

    @Override
    public void close() {
        cleanable.clean(); // explicit close: deterministic
    }

    private static class CleaningAction implements Runnable {
        private final long handle;
        CleaningAction(long handle) { this.handle = handle; }

        @Override
        public void run() {
            System.out.println("Releasing native handle: " + handle);
            // freeNativeResource(handle);
        }
    }
}
```

---

### Q13. How does inheritance create tight coupling, and what is the alternative?

**Model Answer:** Inheritance couples the subclass to the parent's *entire* implementation: any change to the parent's non-private methods or behavior can silently break all subclasses — this is the "fragile base class problem." The subclass also inherits all public and protected methods whether it wants them or not, exposing the parent's full API surface through the child. The alternative is composition with delegation: the class holds a reference to a collaborator, delegates specific operations, and remains isolated from unintended changes in that collaborator's other methods.

```java
// Tight coupling via inheritance: changing Stack in java.util affects all subclasses
public class LoggingList extends ArrayList<String> {
    @Override
    public boolean add(String item) {
        System.out.println("Adding: " + item);
        return super.add(item); // tightly coupled to ArrayList internals
    }
    // Inherits 30+ methods caller never intended to expose
}

// Loose coupling via composition
public class LoggingList {
    private final List<String> delegate = new ArrayList<>();

    public boolean add(String item) {
        System.out.println("Adding: " + item);
        return delegate.add(item); // only delegates what we choose
    }

    public String get(int index) { return delegate.get(index); }
    public int size() { return delegate.size(); }
    // We expose exactly what we want — nothing more
}
```

---

### Q14. How does the Liskov Substitution Principle constrain your inheritance hierarchy design?

**Model Answer:** LSP states that a subtype must be behaviorally substitutable for its supertype: any code written against the supertype must work correctly when given a subtype instance, with no changes and no surprises. This goes beyond method signatures — it constrains preconditions (a subtype may not strengthen them), postconditions (a subtype may not weaken them), and invariants (a subtype must preserve the supertype's invariants). In practice, LSP violations are the clearest signal that inheritance is being misused: if you need to check `instanceof` before calling a method, or if a subclass throws `UnsupportedOperationException` for an inherited method, the hierarchy is wrong.

```java
// LSP violation: Square strengthens preconditions of Rectangle's setters
public class Rectangle {
    protected int width, height;
    public void setWidth(int w) { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int area() { return width * height; }
}

public class Square extends Rectangle {
    @Override public void setWidth(int w) { this.width = w; this.height = w; } // breaks postcondition
    @Override public void setHeight(int h) { this.width = h; this.height = h; }
}

// Code written for Rectangle is broken by Square:
void doubleWidth(Rectangle r) {
    r.setWidth(r.width * 2);
    // Expected: area doubles. With Square: both sides change — area quadruples
}
```

---

### Q15. How do Java 8 default methods act as mixins, and how is the diamond problem resolved?

**Model Answer:** A default method in an interface provides a concrete implementation that any implementing class inherits without needing to override it — this is mixin-style behavior because a class can "mix in" behavior from multiple interfaces simultaneously, something impossible with abstract classes. The diamond problem arises when two interfaces provide default methods with the same signature: Java resolves it by requiring the implementing class to explicitly override the method and choose which interface's default to call (or provide its own implementation), making the resolution explicit rather than implicit. Class methods always take priority over interface default methods, and more specific interfaces take priority over less specific ones in the absence of a class override.

```java
public interface Swimmer {
    default String move() { return "Swimming"; }
}

public interface Runner {
    default String move() { return "Running"; }
}

// Diamond problem: both interfaces have move() — compiler forces resolution
public class Triathlete implements Swimmer, Runner {
    @Override
    public String move() {
        // Must explicitly choose or combine
        return Swimmer.super.move() + " and " + Runner.super.move();
    }
}

// Mixin composition: Logger gets auditing behavior without abstract class
public interface Auditable {
    default void audit(String action) {
        System.out.println("AUDIT [" + Instant.now() + "]: " + action);
    }
}

public interface Loggable {
    default void log(String message) {
        System.out.println("LOG: " + message);
    }
}

public class OrderService implements Auditable, Loggable {
    public void placeOrder(Order order) {
        log("Placing order: " + order.getId());
        audit("ORDER_PLACED");
        // ... business logic
    }
}
```

---

## Category 2: SOLID Principles

### Q16. What is the Single Responsibility Principle and what does the "axis of change" mean?

**Model Answer:** The Single Responsibility Principle states that a class should have only one reason to change — meaning it should have one primary responsibility, accountable to one stakeholder or one part of the system. Robert Martin's "axis of change" metaphor means that if two different actors (e.g., the accounting department and the HR department) could independently demand changes to the same class, that class has two responsibilities and will become a source of merge conflicts and unintended side effects. SRP is not about limiting the number of methods; it's about cohesion — all the methods should serve the same reason to exist.

```java
// Good SRP: each class has exactly one reason to change
public class Order {
    // Changes only when order domain rules change
    private final String orderId;
    private final List<LineItem> items;
    public Money total() { /* ... */ return null; }
}

public class OrderRepository {
    // Changes only when persistence strategy changes
    public void save(Order order) { /* ... */ }
    public Order findById(String id) { /* ... */ return null; }
}

public class OrderEmailNotifier {
    // Changes only when notification format/channel changes
    public void sendConfirmation(Order order, Customer customer) { /* ... */ }
}
```

---

### Q17. The following UserService class violates SRP. What is wrong and why?

```java
public class UserService {
    public User createUser(String name, String email) { /* ... */ return null; }
    public User findById(Long id) { /* ... */ return null; }
    public void updateUser(User user) { /* ... */ }
    public void deleteUser(Long id) { /* ... */ }

    public void sendWelcomeEmail(User user) {
        // constructs SMTP client, builds email body, sends
    }

    public byte[] generateUserReport(User user) {
        // queries data, formats as PDF using iText
        return new byte[0];
    }
}
```

**Model Answer:** This `UserService` has three distinct axes of change: user CRUD business logic (driven by product requirements), email sending behavior (driven by the marketing or infrastructure team changing SMTP providers or templates), and PDF report generation (driven by the reporting team changing format or content). A change to the email template should not require modifying, retesting, and redeploying the user CRUD logic. The class violates SRP by serving three different stakeholders simultaneously, making it a maintenance liability and a merge conflict hotspot on any large team.

---

### Q18. Refactor the UserService above to conform to SRP.

**Model Answer:** Split the class along its three axes of change: `UserService` retains only domain operations and persistence delegation, `UserEmailService` owns all email concerns, and `UserReportService` owns all reporting concerns. Each class now has a single reason to change, can be independently tested with mocks for its collaborators, and can be deployed or modified without risk of breaking the other two concerns.

```java
// Responsibility 1: user domain + persistence delegation
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User createUser(String name, String email) {
        User user = new User(name, email);
        return userRepository.save(user);
    }

    public User findById(Long id) { return userRepository.findById(id); }
    public void updateUser(User user) { userRepository.save(user); }
    public void deleteUser(Long id) { userRepository.deleteById(id); }
}

// Responsibility 2: email notifications
public class UserEmailService {
    private final EmailClient emailClient;

    public UserEmailService(EmailClient emailClient) {
        this.emailClient = emailClient;
    }

    public void sendWelcomeEmail(User user) {
        String body = buildWelcomeEmailBody(user);
        emailClient.send(user.getEmail(), "Welcome!", body);
    }

    private String buildWelcomeEmailBody(User user) {
        return "Hello " + user.getName() + ", welcome to our platform!";
    }
}

// Responsibility 3: report generation
public class UserReportService {
    private final PdfGenerator pdfGenerator;

    public UserReportService(PdfGenerator pdfGenerator) {
        this.pdfGenerator = pdfGenerator;
    }

    public byte[] generateUserReport(User user) {
        ReportData data = collectReportData(user);
        return pdfGenerator.generate(data);
    }

    private ReportData collectReportData(User user) { /* ... */ return null; }
}
```

---

### Q19. What is the Open/Closed Principle and how does it relate to abstraction?

**Model Answer:** The Open/Closed Principle states that software entities should be open for extension but closed for modification — you should be able to add new behavior without changing existing, tested code. The mechanism that makes this possible is abstraction: by programming to an interface or abstract type, you allow new implementations to be added (open for extension) without touching the client code that depends on the abstraction (closed for modification). Without abstraction, every new variant requires a change to existing if/else or switch logic, which risks breaking existing behavior and grows the class indefinitely.

```java
// OCP enabled by abstraction
public interface PaymentProcessor {
    void process(Payment payment);
}

public class CreditCardProcessor implements PaymentProcessor {
    @Override public void process(Payment payment) { /* credit card logic */ }
}

public class PayPalProcessor implements PaymentProcessor {
    @Override public void process(Payment payment) { /* PayPal logic */ }
}

// PaymentService is closed for modification — adding CryptoProcessor requires no change here
public class PaymentService {
    private final PaymentProcessor processor;

    public PaymentService(PaymentProcessor processor) {
        this.processor = processor;
    }

    public void pay(Payment payment) {
        processor.process(payment); // open for extension via new implementations
    }
}
```

---

### Q20. The following discount calculator violates OCP. What is wrong?

```java
public class DiscountCalculator {
    public double calculate(Customer customer, double price) {
        if (customer.getType() == CustomerType.REGULAR) {
            return price * 0.95;
        } else if (customer.getType() == CustomerType.PREMIUM) {
            return price * 0.85;
        } else if (customer.getType() == CustomerType.VIP) {
            return price * 0.75;
        }
        return price;
    }
}
```

**Model Answer:** This violates OCP because adding a new customer type (e.g., `EMPLOYEE`) requires modifying `DiscountCalculator.calculate()`, a class that is supposedly stable business logic. Every modification reopens the class to regression risk: a careless change to the `PREMIUM` branch while adding `EMPLOYEE` could silently alter the premium discount. The if/else chain will grow linearly with each new type, and the class becomes a perpetual modification target — the opposite of "closed for modification."

---

### Q21. Refactor the discount calculator to be open for extension but closed for modification.

**Model Answer:** Introduce a `DiscountStrategy` abstraction and move each discount rule into its own implementation. The `DiscountCalculator` depends only on the abstraction and never needs to change when new customer types are added. Map the strategy to the customer type at the composition root (or via a factory/registry), not inside the calculator itself.

```java
// Strategy abstraction
public interface DiscountStrategy {
    double apply(double price);
}

// Individual strategies — each is independently testable
public class RegularDiscount implements DiscountStrategy {
    @Override public double apply(double price) { return price * 0.95; }
}

public class PremiumDiscount implements DiscountStrategy {
    @Override public double apply(double price) { return price * 0.85; }
}

public class VipDiscount implements DiscountStrategy {
    @Override public double apply(double price) { return price * 0.75; }
}

// Adding EMPLOYEE discount: zero changes to existing classes
public class EmployeeDiscount implements DiscountStrategy {
    @Override public double apply(double price) { return price * 0.60; }
}

// Calculator is now closed for modification
public class DiscountCalculator {
    private final Map<CustomerType, DiscountStrategy> strategies;

    public DiscountCalculator(Map<CustomerType, DiscountStrategy> strategies) {
        this.strategies = strategies;
    }

    public double calculate(CustomerType type, double price) {
        DiscountStrategy strategy = strategies.getOrDefault(type, p -> p);
        return strategy.apply(price);
    }
}

// Wiring at composition root
Map<CustomerType, DiscountStrategy> strategies = new EnumMap<>(CustomerType.class);
strategies.put(CustomerType.REGULAR, new RegularDiscount());
strategies.put(CustomerType.PREMIUM, new PremiumDiscount());
strategies.put(CustomerType.VIP, new VipDiscount());
DiscountCalculator calc = new DiscountCalculator(strategies);
```

---

### Q22. What is the Liskov Substitution Principle and why does it go beyond method signature compatibility?

**Model Answer:** LSP, formulated by Barbara Liskov, states that objects of a supertype should be replaceable with objects of a subtype without altering the correctness of the program. Method signature compatibility (compile-time) is a necessary but not sufficient condition: LSP also requires behavioral compatibility — the subtype must honor the supertype's preconditions (it cannot demand stricter inputs), postconditions (it cannot promise weaker outputs), and invariants (it cannot break the supertype's guaranteed state). A `Square extends Rectangle` compiles and type-checks fine but violates LSP behaviorally because `Square.setWidth()` has a side effect on height that `Rectangle.setWidth()` does not have.

---

### Q23. The following Square/Rectangle example violates LSP. Why?

```java
public class Rectangle {
    protected int width, height;

    public void setWidth(int w) { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int area() { return width * height; }
}

public class Square extends Rectangle {
    @Override public void setWidth(int w) { this.width = w; this.height = w; }
    @Override public void setHeight(int h) { this.width = h; this.height = h; }
}
```

**Model Answer:** `Rectangle`'s contract implies that `setWidth` and `setHeight` are *independent* — a postcondition of `setWidth(5)` is that `width == 5` and `height` is unchanged. `Square` violates this postcondition because `setWidth(5)` also changes `height`. Any code that uses `Rectangle` and calls `setWidth` and `setHeight` independently (e.g., to resize an image) will compute the wrong area when given a `Square`. The test `doubleWidth(rect)` below demonstrates the breakage: the assertion passes for `Rectangle` but fails for `Square`.

```java
void doubleWidth(Rectangle r) {
    int originalHeight = r.height;
    r.setWidth(r.width * 2);
    // Post-condition: height should be unchanged
    assert r.height == originalHeight; // FAILS for Square
    assert r.area() == r.width * r.height; // also broken for Square
}
```

---

### Q24. How do you fix the Square/Rectangle LSP violation?

**Model Answer:** The fix is to break the inheritance relationship. Neither `Square` nor `Rectangle` should extend the other. Instead, both can implement a common `Shape` interface that exposes only behavior that both can honor without violating each other's postconditions — in this case, `area()` and `perimeter()`. If you need a method to "resize" a shape, define it separately on each type since the semantics differ fundamentally.

```java
public interface Shape {
    int area();
    int perimeter();
}

public class Rectangle implements Shape {
    private int width, height;

    public Rectangle(int width, int height) {
        this.width = width; this.height = height;
    }

    public Rectangle withWidth(int w) { return new Rectangle(w, height); }
    public Rectangle withHeight(int h) { return new Rectangle(width, h); }

    @Override public int area() { return width * height; }
    @Override public int perimeter() { return 2 * (width + height); }
}

public class Square implements Shape {
    private final int side;

    public Square(int side) { this.side = side; }
    public Square withSide(int s) { return new Square(s); }

    @Override public int area() { return side * side; }
    @Override public int perimeter() { return 4 * side; }
}
// Both implement Shape — no LSP violation, no shared mutable state contract
```

---

### Q25. What is the Interface Segregation Principle and what problem does a "fat interface" cause?

**Model Answer:** The Interface Segregation Principle states that no client should be forced to depend on methods it does not use — interfaces should be narrow and role-specific rather than monolithic. A "fat interface" forces implementing classes to provide implementations (even empty or exception-throwing ones) for methods irrelevant to them, creating unnecessary coupling between the implementor and unrelated operations and making the interface harder to implement correctly. The symptom is `UnsupportedOperationException` throws for methods the implementing class does not support.

```java
// Fat interface: forces all implementors to deal with all operations
public interface Worker {
    void work();
    void eat();
    void sleep();
    void takeVacation();
}

// RobotWorker violates ISP — forced to implement biological behaviors
public class RobotWorker implements Worker {
    @Override public void work() { System.out.println("Processing data..."); }
    @Override public void eat() { throw new UnsupportedOperationException("Robots don't eat"); }
    @Override public void sleep() { throw new UnsupportedOperationException("Robots don't sleep"); }
    @Override public void takeVacation() { throw new UnsupportedOperationException("Robots don't vacation"); }
}
```

---

### Q26. Describe the ISP violation in the Worker/RobotWorker example.

**Model Answer:** The `Worker` interface conflates two distinct concerns: work-related behavior (`work()`) and biological needs (`eat()`, `sleep()`, `takeVacation()`). `RobotWorker` is forced to depend on and implement biological methods it cannot support, so it throws `UnsupportedOperationException` — a runtime lie that breaks the contract of the interface. Any code that calls `worker.eat()` on a collection of `Worker` objects will fail at runtime for robots, defeating the purpose of polymorphism. This is an ISP violation because `RobotWorker` is forced to depend on an interface portion it does not use.

---

### Q27. Refactor the Worker interface to conform to ISP.

**Model Answer:** Split the fat interface along its natural responsibility boundaries: `Workable` for work behavior, `BiologicalNeeds` (or separate `Eatable`, `Sleepable`) for biological functions. `RobotWorker` implements only `Workable`, `HumanWorker` implements both. Consumers that need only work-related behavior depend on `Workable`; those managing biological schedules depend on `BiologicalNeeds` — no class is forced to implement what it does not support.

```java
// Segregated interfaces
public interface Workable {
    void work();
}

public interface Eatable {
    void eat();
}

public interface Sleepable {
    void sleep();
}

public interface HumanWorkerCapabilities extends Workable, Eatable, Sleepable {
    void takeVacation();
}

// RobotWorker implements only what it supports
public class RobotWorker implements Workable {
    @Override public void work() { System.out.println("Processing data..."); }
}

// HumanWorker implements all capabilities
public class HumanWorker implements HumanWorkerCapabilities {
    @Override public void work() { System.out.println("Writing code..."); }
    @Override public void eat() { System.out.println("Having lunch..."); }
    @Override public void sleep() { System.out.println("Sleeping 8 hours..."); }
    @Override public void takeVacation() { System.out.println("On vacation!"); }
}

// Scheduler only cares about Workable — works for both humans and robots
public class WorkScheduler {
    public void schedule(List<Workable> workers) {
        workers.forEach(Workable::work);
    }
}
```

---

### Q28. What is the Dependency Inversion Principle and why is it called "inversion"?

**Model Answer:** DIP states two things: high-level modules should not depend on low-level modules — both should depend on abstractions; and abstractions should not depend on details — details should depend on abstractions. It is called "inversion" because it flips the traditional layered dependency direction: instead of the business logic (`OrderService`) directly importing and depending on the infrastructure class (`MySQLOrderRepository`), both depend on an interface (`OrderRepository`) that the business logic defines. The "ownership" of the abstraction moves up to the high-level module, inverting the dependency arrow in the module dependency diagram.

```java
// Traditional (uninverted): high-level depends on low-level
// OrderService --> MySQLOrderRepository  (wrong direction)

// DIP (inverted): both depend on abstraction
// OrderService --> OrderRepository <-- MySQLOrderRepository  (correct)

public interface OrderRepository { // defined in the domain layer (high-level owns it)
    Order findById(String id);
    void save(Order order);
}

public class OrderService { // high-level module
    private final OrderRepository repository; // depends on abstraction, not detail
    public OrderService(OrderRepository repository) { this.repository = repository; }
}

public class MySQLOrderRepository implements OrderRepository { // detail depends on abstraction
    @Override public Order findById(String id) { /* SQL query */ return null; }
    @Override public void save(Order order) { /* SQL insert/update */ }
}
```

---

### Q29. The following OrderService violates DIP. What is wrong?

```java
public class OrderService {
    private MySQLOrderRepository repository = new MySQLOrderRepository();

    public void placeOrder(Order order) {
        repository.save(order);
    }
}
```

**Model Answer:** `OrderService` directly instantiates `MySQLOrderRepository` inside itself, creating a hard compile-time dependency on a concrete infrastructure class. This means: you cannot test `OrderService` without a real MySQL database; you cannot swap the repository implementation (e.g., to PostgreSQL or an in-memory stub) without modifying `OrderService`; and the `new` keyword inside the class means `OrderService` also violates the "don't call us, we'll call you" principle by controlling its own dependency graph. The high-level module is controlling the low-level detail rather than being injected with it.

---

### Q30. Refactor the OrderService to depend on an abstraction and show how a DI container would wire it.

**Model Answer:** Extract the `OrderRepository` interface, constructor-inject it into `OrderService`, and let the DI container (Spring, Guice, or manual wiring in main) provide the concrete implementation. This makes `OrderService` fully testable with a mock or in-memory repository and makes infrastructure swappable without touching business logic.

```java
// Abstraction defined in the domain layer
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(String id);
}

// High-level module depends on abstraction via constructor injection
public class OrderService {
    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = Objects.requireNonNull(orderRepository);
    }

    public void placeOrder(Order order) {
        // business validation
        if (order.getItems().isEmpty()) throw new IllegalArgumentException("Order must have items");
        orderRepository.save(order);
    }
}

// Low-level detail implements the abstraction
@Repository
public class MySQLOrderRepository implements OrderRepository {
    private final JdbcTemplate jdbc;
    public MySQLOrderRepository(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    @Override public void save(Order order) { /* SQL */ }
    @Override public Optional<Order> findById(String id) { /* SQL */ return Optional.empty(); }
}

// Spring DI container wiring (annotation-based)
@Service
public class OrderServiceSpring {
    private final OrderRepository orderRepository;

    @Autowired
    public OrderServiceSpring(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
    // Spring injects MySQLOrderRepository automatically via @Repository
}

// Manual wiring (pure Java — for testing or non-Spring contexts)
// OrderRepository repo = new MySQLOrderRepository(jdbcTemplate);
// OrderService service = new OrderService(repo);

// In tests: inject a stub
// OrderRepository stubRepo = new InMemoryOrderRepository();
// OrderService testService = new OrderService(stubRepo);
```

---
## Category 3: Design Patterns

### Q31. How do you make a Singleton thread-safe in Java, and why is the enum singleton preferred?

**Model Answer:** The naive singleton (check-then-act without synchronization) has a race condition where two threads can simultaneously see the instance as null and both create instances. Double-checked locking with `volatile` fixes this for Java 5+: the `volatile` keyword prevents instruction reordering that could cause a thread to see a partially constructed instance. However, the enum singleton is universally preferred because the JVM guarantees that enum instances are created exactly once during class loading (which is inherently thread-safe), enum singletons survive serialization correctly (ordinary singletons need `readResolve()`), and they are immune to reflection-based instantiation attacks.

```java
// Double-checked locking (correct but verbose)
public class ConfigManager {
    private static volatile ConfigManager instance;

    private ConfigManager() {}

    public static ConfigManager getInstance() {
        if (instance == null) {                    // first check (no lock)
            synchronized (ConfigManager.class) {
                if (instance == null) {            // second check (with lock)
                    instance = new ConfigManager();
                }
            }
        }
        return instance;
    }
}

// Initialization-on-demand holder (lazy, thread-safe, no synchronization overhead)
public class ConfigManager {
    private ConfigManager() {}

    private static class Holder {
        static final ConfigManager INSTANCE = new ConfigManager();
    }

    public static ConfigManager getInstance() { return Holder.INSTANCE; }
}

// Enum singleton (preferred: serialization-safe, reflection-safe)
public enum ConfigManager {
    INSTANCE;

    private Properties config = new Properties();

    public String get(String key) { return config.getProperty(key); }
    public void load(InputStream in) throws IOException { config.load(in); }
}
```

---

### Q32. What is the difference between Factory Method and Abstract Factory patterns, and when does each apply?

**Model Answer:** Factory Method defines an interface for creating a single product and lets subclasses decide which concrete class to instantiate — it is a single-product creation hook used within a class hierarchy. Abstract Factory provides an interface for creating *families* of related objects without specifying their concrete classes — it is used when you need multiple related products that must be consistent with each other (e.g., a UI widget toolkit where buttons and checkboxes must match the same theme). Factory Method is a single-level abstraction; Abstract Factory is a two-level abstraction (factory of factories).

```java
// Factory Method: subclass decides which product to create
public abstract class NotificationFactory {
    public abstract Notification createNotification(String message);

    public void notify(String message) {
        Notification n = createNotification(message); // hook
        n.send();
    }
}

public class EmailNotificationFactory extends NotificationFactory {
    @Override
    public Notification createNotification(String message) {
        return new EmailNotification(message);
    }
}

public class SmsNotificationFactory extends NotificationFactory {
    @Override
    public Notification createNotification(String message) {
        return new SmsNotification(message);
    }
}

// Abstract Factory: family of related products
public interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
    TextField createTextField();
}

public class MaterialUIFactory implements UIFactory {
    @Override public Button createButton() { return new MaterialButton(); }
    @Override public Checkbox createCheckbox() { return new MaterialCheckbox(); }
    @Override public TextField createTextField() { return new MaterialTextField(); }
}

public class CupertinoUIFactory implements UIFactory {
    @Override public Button createButton() { return new CupertinoButton(); }
    @Override public Checkbox createCheckbox() { return new CupertinoCheckbox(); }
    @Override public TextField createTextField() { return new CupertinoTextField(); }
}

// Client: uses the factory to build a consistent UI family
public class Application {
    private final UIFactory factory;
    public Application(UIFactory factory) { this.factory = factory; }

    public void buildUI() {
        Button btn = factory.createButton();
        Checkbox cb = factory.createCheckbox();
        // All components are from the same consistent family
    }
}
```

---

### Q33. How does the Builder pattern handle mandatory versus optional fields, and how does method chaining work?

**Model Answer:** The Builder pattern separates complex object construction from its representation by requiring mandatory fields through the Builder's constructor (enforcing they are always supplied) and optional fields through fluent setter methods that return the builder itself for method chaining. This avoids the telescoping constructor anti-pattern (multiple overloaded constructors for every combination of optional parameters) and makes construction code readable and compile-time-safe. The `build()` method at the end performs final validation before constructing the immutable target object.

```java
public final class HttpRequest {
    // Mandatory fields
    private final String url;
    private final HttpMethod method;
    // Optional fields
    private final Map<String, String> headers;
    private final String body;
    private final int timeoutMs;

    private HttpRequest(Builder builder) {
        this.url = builder.url;
        this.method = builder.method;
        this.headers = Collections.unmodifiableMap(new HashMap<>(builder.headers));
        this.body = builder.body;
        this.timeoutMs = builder.timeoutMs;
    }

    public static class Builder {
        // Mandatory: passed to constructor
        private final String url;
        private final HttpMethod method;
        // Optional: have sensible defaults
        private Map<String, String> headers = new HashMap<>();
        private String body = null;
        private int timeoutMs = 5000;

        public Builder(String url, HttpMethod method) {  // mandatory in constructor
            this.url = Objects.requireNonNull(url, "url must not be null");
            this.method = Objects.requireNonNull(method, "method must not be null");
        }

        public Builder header(String name, String value) {
            this.headers.put(name, value);
            return this; // method chaining
        }

        public Builder body(String body) {
            this.body = body;
            return this;
        }

        public Builder timeoutMs(int timeoutMs) {
            if (timeoutMs <= 0) throw new IllegalArgumentException("Timeout must be positive");
            this.timeoutMs = timeoutMs;
            return this;
        }

        public HttpRequest build() {
            if (method == HttpMethod.POST && body == null) {
                throw new IllegalStateException("POST requests require a body");
            }
            return new HttpRequest(this);
        }
    }
}

// Usage: readable, mandatory fields enforced, optional fields default
HttpRequest req = new HttpRequest.Builder("https://api.example.com/orders", HttpMethod.POST)
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token123")
    .body("{\"item\": \"book\"}")
    .timeoutMs(3000)
    .build();
```

---

### Q34. When should you use the Prototype pattern (clone) versus a copy constructor in Java?

**Model Answer:** The Prototype pattern (via `clone()`) is appropriate when the type of the object to copy is unknown at compile time — you have a reference to a supertype and want a copy of whatever concrete subtype it actually is, without `instanceof` checks. A copy constructor is preferred in all other cases: it is explicit, type-safe, does not require implementing `Cloneable`, handles deep copy clearly, and works correctly in class hierarchies without the fragility of `super.clone()` chains. The `Cloneable`/`clone()` mechanism in Java is widely considered broken API design (see Effective Java Item 13).

```java
// Prototype: useful when concrete type is unknown at runtime
public abstract class Shape implements Cloneable {
    protected String color;
    public abstract Shape clone(); // covariant return
    public abstract double area();
}

public class Circle extends Shape {
    private double radius;
    public Circle(double radius, String color) { this.radius = radius; this.color = color; }

    @Override
    public Circle clone() {
        try {
            return (Circle) super.clone(); // OK for primitive/immutable fields
        } catch (CloneNotSupportedException e) { throw new AssertionError(); }
    }
    @Override public double area() { return Math.PI * radius * radius; }
}

// Copy constructor: preferred for known types
public class Circle {
    private final double radius;
    private final String color;

    public Circle(Circle other) {  // copy constructor
        this.radius = other.radius;
        this.color = other.color;
    }
}

// Prototype shines here: works without knowing the concrete subclass
Shape original = getShapeFromSomewhere(); // could be Circle, Rectangle, Triangle...
Shape copy = original.clone();            // correct copy regardless of runtime type
```

---

### Q35. Explain the Object Pool pattern, its lifecycle management, and thread safety considerations.

**Model Answer:** An Object Pool pre-allocates a set of expensive-to-create objects (database connections, thread objects, SSL sessions) and recycles them instead of creating and destroying them on every use. The pool manages lifecycle: a client acquires an object, uses it, and returns it (even on failure, usually in a `finally` block); the pool validates the returned object before making it available again, and may grow or shrink within configured bounds. Thread safety requires synchronization around the acquire/release operations — typically using a `BlockingQueue<T>` for the idle pool (which handles blocking semantics naturally) or a semaphore to limit concurrent acquisitions.

```java
public class ConnectionPool {
    private final BlockingDeque<Connection> pool;
    private final int maxSize;
    private final ConnectionFactory factory;

    public ConnectionPool(ConnectionFactory factory, int maxSize) {
        this.factory = factory;
        this.maxSize = maxSize;
        this.pool = new LinkedBlockingDeque<>(maxSize);
        // Pre-warm with minimum connections
        for (int i = 0; i < maxSize / 2; i++) {
            pool.offer(factory.createConnection());
        }
    }

    public Connection acquire(long timeoutMs) throws InterruptedException {
        Connection conn = pool.poll(timeoutMs, TimeUnit.MILLISECONDS);
        if (conn == null) throw new RuntimeException("Connection pool exhausted");
        if (!conn.isValid()) {
            conn.close();
            conn = factory.createConnection(); // replace invalid connection
        }
        return conn;
    }

    public void release(Connection conn) {
        if (conn != null && conn.isValid()) {
            pool.offer(conn); // return to pool
        } else if (conn != null) {
            conn.close(); // discard invalid connection
        }
    }
}

// Usage with try-finally to ensure release on exception
Connection conn = pool.acquire(1000);
try {
    conn.execute("SELECT ...");
} finally {
    pool.release(conn); // always returned to pool
}
```

---

### Q36. Distinguish between Adapter, Facade, and Proxy patterns. What is each used for?

**Model Answer:** The Adapter pattern converts an incompatible interface into the interface a client expects — it is about interface translation, bridging two pre-existing, incompatible interfaces. The Facade pattern provides a simplified, unified interface to a complex subsystem — it is about simplification, hiding complexity behind a single entry point without changing the subsystem's interface. The Proxy pattern provides a surrogate with the *same* interface as the real object — it is about controlling access, adding cross-cutting concerns (caching, security, logging, lazy loading) without the client knowing a proxy is in use.

```java
// Adapter: LegacyPaymentGateway has incompatible API — adapt it to PaymentProcessor
public interface PaymentProcessor {
    void processPayment(double amount, String currency);
}

public class LegacyPaymentGateway {
    public void makeTransaction(int amountInCents, String currencyCode) { /* legacy */ }
}

public class LegacyPaymentAdapter implements PaymentProcessor {
    private final LegacyPaymentGateway legacy;
    public LegacyPaymentAdapter(LegacyPaymentGateway legacy) { this.legacy = legacy; }

    @Override
    public void processPayment(double amount, String currency) {
        int cents = (int)(amount * 100);
        legacy.makeTransaction(cents, currency); // translate interface
    }
}

// Facade: hide complex subsystem
public class OrderFacade {
    private final InventoryService inventory;
    private final PaymentService payment;
    private final ShippingService shipping;
    private final NotificationService notification;

    public void placeOrder(Order order) {
        inventory.reserve(order);
        payment.charge(order);
        shipping.schedule(order);
        notification.sendConfirmation(order);
        // Single method hides 4-service orchestration
    }
}

// Proxy: same interface, adds caching
public class CachingProductRepository implements ProductRepository {
    private final ProductRepository delegate;
    private final Map<String, Product> cache = new ConcurrentHashMap<>();

    public CachingProductRepository(ProductRepository delegate) { this.delegate = delegate; }

    @Override
    public Product findById(String id) {
        return cache.computeIfAbsent(id, delegate::findById); // transparent to caller
    }
}
```

---

### Q37. How does the Decorator pattern connect to the Open/Closed Principle, and how is it exemplified in Java I/O?

**Model Answer:** The Decorator pattern attaches additional responsibilities to an object dynamically by wrapping it in a decorator that implements the same interface and delegates to the wrapped object while adding behavior before or after the delegation. This is a concrete realization of OCP: you extend behavior (adding buffering, compression, encryption) without modifying existing classes. Java's I/O framework (`InputStream`, `OutputStream`, `Reader`, `Writer`) is the canonical Java Decorator example: `BufferedInputStream` wraps any `InputStream` to add buffering; `GZIPInputStream` wraps it to add decompression — each is a decorator layered independently.

```java
// Java I/O: classic Decorator chain
InputStream raw    = new FileInputStream("data.txt");
InputStream buffered = new BufferedInputStream(raw);        // adds buffering
InputStream decompressed = new GZIPInputStream(buffered);   // adds decompression

// Custom Decorator example: logging decorator for a service
public interface OrderRepository {
    Order findById(String id);
    void save(Order order);
}

public class LoggingOrderRepository implements OrderRepository {
    private final OrderRepository delegate;
    private final Logger log = LoggerFactory.getLogger(getClass());

    public LoggingOrderRepository(OrderRepository delegate) { this.delegate = delegate; }

    @Override
    public Order findById(String id) {
        log.debug("Finding order by id: {}", id);
        Order order = delegate.findById(id);
        log.debug("Found order: {}", order);
        return order;
    }

    @Override
    public void save(Order order) {
        log.debug("Saving order: {}", order.getId());
        delegate.save(order);
        log.debug("Order saved successfully");
    }
}

// Layered decorators — each adds one concern
OrderRepository base = new MySQLOrderRepository(dataSource);
OrderRepository logged = new LoggingOrderRepository(base);
OrderRepository cached = new CachingOrderRepository(logged);
OrderRepository metered = new MeteredOrderRepository(cached);
// metered.findById(...) triggers: metrics -> cache -> log -> SQL
```

---

### Q38. How does the Composite pattern enable uniform treatment of leaf nodes and composites in Java?

**Model Answer:** The Composite pattern composes objects into tree structures to represent part-whole hierarchies and lets clients treat individual objects (leaves) and compositions of objects (composites) uniformly through a common interface. The key is that both `Leaf` and `Composite` implement the same `Component` interface, so a client calling `component.render()` does not need to know whether it is rendering a single widget or an entire panel of widgets. The `Composite` implements each operation by iterating over its children and delegating, naturally making operations recursive.

```java
// Component: uniform interface for both
public interface FileSystemComponent {
    String getName();
    long size();
    void display(String indent);
}

// Leaf
public class File implements FileSystemComponent {
    private final String name;
    private final long sizeBytes;

    public File(String name, long sizeBytes) { this.name = name; this.sizeBytes = sizeBytes; }

    @Override public String getName() { return name; }
    @Override public long size() { return sizeBytes; }
    @Override public void display(String indent) {
        System.out.println(indent + name + " (" + sizeBytes + " bytes)");
    }
}

// Composite: contains children, delegates operations
public class Directory implements FileSystemComponent {
    private final String name;
    private final List<FileSystemComponent> children = new ArrayList<>();

    public Directory(String name) { this.name = name; }

    public void add(FileSystemComponent component) { children.add(component); }
    public void remove(FileSystemComponent component) { children.remove(component); }

    @Override public String getName() { return name; }

    @Override
    public long size() {
        return children.stream().mapToLong(FileSystemComponent::size).sum(); // recursive
    }

    @Override
    public void display(String indent) {
        System.out.println(indent + "[" + name + "]");
        children.forEach(c -> c.display(indent + "  ")); // recursive delegation
    }
}

// Client treats File and Directory uniformly
FileSystemComponent root = new Directory("root");
FileSystemComponent src = new Directory("src");
src.add(new File("Main.java", 1024));
src.add(new File("App.java", 2048));
root.add(src);
root.add(new File("README.md", 512));
root.display(""); // recursively displays entire tree
System.out.println("Total: " + root.size()); // 3584 bytes — works on the whole tree
```

---

### Q39. Explain the Bridge pattern and provide a Java example of separating abstraction from implementation.

**Model Answer:** The Bridge pattern decouples an abstraction (the high-level control layer) from its implementation (the platform or lower-level layer) so that both can vary independently. Without the bridge, a class hierarchy for `Shape x Color` would require `RedCircle`, `BlueCircle`, `RedSquare`, `BlueSquare` — M×N subclasses. With the bridge, `Shape` holds a reference to a `Renderer` implementation; you add new shapes without creating new renderers and vice versa. The "bridge" is the reference from the abstraction to the implementation.

```java
// Implementation interface (the bridge)
public interface Renderer {
    void renderCircle(double radius);
    void renderRectangle(double width, double height);
}

// Concrete implementations
public class VectorRenderer implements Renderer {
    @Override public void renderCircle(double r) { System.out.println("Vector circle r=" + r); }
    @Override public void renderRectangle(double w, double h) { System.out.println("Vector rect " + w + "x" + h); }
}

public class RasterRenderer implements Renderer {
    @Override public void renderCircle(double r) { System.out.println("Raster circle r=" + r + " (pixels)"); }
    @Override public void renderRectangle(double w, double h) { System.out.println("Raster rect " + w + "x" + h); }
}

// Abstraction: holds reference to implementation (the bridge)
public abstract class Shape {
    protected final Renderer renderer;
    protected Shape(Renderer renderer) { this.renderer = renderer; }
    public abstract void draw();
}

// Refined abstractions
public class Circle extends Shape {
    private final double radius;
    public Circle(double radius, Renderer renderer) { super(renderer); this.radius = radius; }
    @Override public void draw() { renderer.renderCircle(radius); }
}

public class Rectangle extends Shape {
    private final double width, height;
    public Rectangle(double w, double h, Renderer renderer) { super(renderer); width = w; height = h; }
    @Override public void draw() { renderer.renderRectangle(width, height); }
}

// Adding new shapes/renderers doesn't require changing existing code
Shape c = new Circle(5.0, new VectorRenderer());
Shape r = new Rectangle(3.0, 4.0, new RasterRenderer());
c.draw(); r.draw();
```

---

### Q40. What is the Flyweight pattern? Distinguish intrinsic from extrinsic state and explain when the memory savings justify the complexity.

**Model Answer:** The Flyweight pattern reduces memory consumption by sharing common state across many fine-grained objects. Intrinsic state is the immutable, shared part stored inside the flyweight (e.g., a character's font, style, and size in a text editor). Extrinsic state is the context-dependent part passed in at use time (e.g., the character's position on the page). The complexity is justified when you have a very large number of objects that share significant common state — the canonical example is rendering thousands of tree objects in a game where each tree shares the same mesh/texture (intrinsic) but has its own position/scale (extrinsic).

```java
// Flyweight: shared intrinsic state
public final class TreeType {
    private final String name;
    private final String color;
    private final byte[] texture; // large, shared

    public TreeType(String name, String color, byte[] texture) {
        this.name = name; this.color = color; this.texture = texture;
    }

    public void draw(int x, int y) { // x,y are extrinsic — passed in
        System.out.printf("Drawing %s tree at (%d,%d)%n", name, x, y);
    }
}

// Flyweight Factory: manages the cache of shared flyweights
public class TreeTypeFactory {
    private static final Map<String, TreeType> cache = new HashMap<>();

    public static TreeType getTreeType(String name, String color, byte[] texture) {
        return cache.computeIfAbsent(name, k -> new TreeType(k, color, texture));
    }
}

// Context: holds extrinsic state + reference to flyweight
public class Tree {
    private final int x, y;               // extrinsic
    private final TreeType type;          // reference to shared flyweight

    public Tree(int x, int y, TreeType type) { this.x = x; this.y = y; this.type = type; }
    public void draw() { type.draw(x, y); }
}

// Forest of 1,000,000 trees shares only a handful of TreeType objects
List<Tree> forest = new ArrayList<>();
TreeType oak = TreeTypeFactory.getTreeType("Oak", "Green", largeOakTexture);
for (int i = 0; i < 1_000_000; i++) {
    forest.add(new Tree(random.nextInt(1000), random.nextInt(1000), oak));
}
// Memory: 1M Tree objects (small) + 1 TreeType (large texture shared once)
```

---

### Q41. How do Strategy and State patterns differ structurally and semantically?

**Model Answer:** Both patterns use an object that holds a reference to an interface and delegates behavior to it — structurally they look nearly identical. The semantic difference is who controls the switch and why: in Strategy, the algorithm is selected externally by the client and does not change based on internal object state (the context is passive); in State, the state object itself can trigger transitions to other states based on internal events (the context is active and stateful). A traffic light uses State (transitions from Green to Yellow to Red are driven by internal timer events); a sorting algorithm selector uses Strategy (the client picks bubble sort or merge sort externally and the algorithm has no opinion about the context's state).

```java
// Strategy: client controls selection, no internal transitions
public interface SortStrategy { void sort(int[] data); }
public class QuickSort implements SortStrategy { @Override public void sort(int[] d) { /* quicksort */ } }
public class MergeSort implements SortStrategy { @Override public void sort(int[] d) { /* mergesort */ } }

public class Sorter {
    private SortStrategy strategy;
    public void setStrategy(SortStrategy s) { this.strategy = s; }
    public void sort(int[] data) { strategy.sort(data); }
}

// State: object controls its own transitions
public interface TrafficLightState {
    void handle(TrafficLight light);
    String getColor();
}

public class GreenState implements TrafficLightState {
    @Override public void handle(TrafficLight light) { light.setState(new YellowState()); }
    @Override public String getColor() { return "GREEN"; }
}

public class YellowState implements TrafficLightState {
    @Override public void handle(TrafficLight light) { light.setState(new RedState()); }
    @Override public String getColor() { return "YELLOW"; }
}

public class RedState implements TrafficLightState {
    @Override public void handle(TrafficLight light) { light.setState(new GreenState()); }
    @Override public String getColor() { return "RED"; }
}

public class TrafficLight {
    private TrafficLightState state = new GreenState();
    public void setState(TrafficLightState s) { this.state = s; }
    public void tick() { state.handle(this); } // state drives its own transition
    public String getColor() { return state.getColor(); }
}
```

---

### Q42. What are the pitfalls of the Observer pattern in Java — memory leaks, event ordering, and thread safety?

**Model Answer:** Memory leaks occur when observers are registered but never deregistered — the subject holds a strong reference to the observer, preventing garbage collection even after the UI component or service has been "logically" destroyed; the fix is to use `WeakReference` for observers or enforce explicit deregistration. Event ordering issues arise when one observer's handler modifies the subject's state and triggers additional notifications mid-dispatch, causing observers to receive events out of the expected order or causing `ConcurrentModificationException` on the observer list. Thread safety is a concern when notifications fire from a background thread while the observer list is being modified; fix by synchronizing list modifications and dispatching on a snapshot copy, or using a `CopyOnWriteArrayList`.

```java
public class EventBus<T> {
    // CopyOnWriteArrayList: thread-safe iteration, no ConcurrentModificationException
    private final List<Observer<T>> observers = new CopyOnWriteArrayList<>();

    public void subscribe(Observer<T> observer) { observers.add(observer); }
    public void unsubscribe(Observer<T> observer) { observers.remove(observer); }

    public void publish(T event) {
        // Iterate over snapshot (CopyOnWriteArrayList handles this)
        for (Observer<T> observer : observers) {
            try {
                observer.onEvent(event);
            } catch (Exception e) {
                // Isolate: one observer failure should not block others
                log.error("Observer failed: {}", e.getMessage());
            }
        }
    }
}

// Memory leak fix: weak references
public class WeakObserverRegistry<T> {
    private final List<WeakReference<Observer<T>>> refs = new CopyOnWriteArrayList<>();

    public void subscribe(Observer<T> observer) {
        refs.add(new WeakReference<>(observer));
    }

    public void publish(T event) {
        refs.removeIf(ref -> ref.get() == null); // prune dead refs
        refs.stream()
            .map(WeakReference::get)
            .filter(Objects::nonNull)
            .forEach(o -> o.onEvent(event));
    }
}
```

---

### Q43. How do you implement undo/redo using the Command pattern in Java?

**Model Answer:** The Command pattern encapsulates an operation as an object with an `execute()` method. To support undo, each command also implements an `undo()` method that reverses the operation; the receiver stores enough state before execution to support reversal. The invoker maintains two stacks: an undo stack (commands executed so far) and a redo stack (commands that have been undone). On execute, push to undo stack and clear redo stack. On undo, pop from undo stack, call `undo()`, and push to redo stack. On redo, pop from redo stack, call `execute()`, and push to undo stack.

```java
public interface Command {
    void execute();
    void undo();
}

public class TextEditor {
    private final StringBuilder text = new StringBuilder();

    public void insert(int pos, String str) { text.insert(pos, str); }
    public void delete(int pos, int len) { text.delete(pos, pos + len); }
    public String getText() { return text.toString(); }
}

public class InsertCommand implements Command {
    private final TextEditor editor;
    private final int position;
    private final String text;

    public InsertCommand(TextEditor editor, int position, String text) {
        this.editor = editor; this.position = position; this.text = text;
    }

    @Override public void execute() { editor.insert(position, text); }
    @Override public void undo() { editor.delete(position, text.length()); }
}

public class CommandHistory {
    private final Deque<Command> undoStack = new ArrayDeque<>();
    private final Deque<Command> redoStack = new ArrayDeque<>();

    public void execute(Command cmd) {
        cmd.execute();
        undoStack.push(cmd);
        redoStack.clear(); // new command invalidates redo history
    }

    public void undo() {
        if (!undoStack.isEmpty()) {
            Command cmd = undoStack.pop();
            cmd.undo();
            redoStack.push(cmd);
        }
    }

    public void redo() {
        if (!redoStack.isEmpty()) {
            Command cmd = redoStack.pop();
            cmd.execute();
            undoStack.push(cmd);
        }
    }
}
```

---

### Q44. How does the Chain of Responsibility pattern work in filter chains and middleware pipelines?

**Model Answer:** In Chain of Responsibility, a request is passed along a chain of handlers; each handler decides whether to process the request, pass it to the next handler, or both. In a filter/middleware context, every handler in the chain typically both processes the request (e.g., adding a header, checking auth) and delegates to the next handler, giving each filter the opportunity to wrap the request and response. Java's Servlet filter chain is the canonical example; modern frameworks implement the same pattern as a `Handler` list or as a `next` function passed recursively.

```java
public interface RequestHandler {
    void handle(HttpRequest request, HttpResponse response, RequestHandler next);
}

// Authentication filter
public class AuthFilter implements RequestHandler {
    @Override
    public void handle(HttpRequest req, HttpResponse res, RequestHandler next) {
        String token = req.getHeader("Authorization");
        if (token == null || !isValid(token)) {
            res.setStatus(401);
            return; // short-circuit: do not call next
        }
        next.handle(req, res, null); // pass to next in chain
    }
    private boolean isValid(String token) { return token.startsWith("Bearer "); }
}

// Logging filter
public class LoggingFilter implements RequestHandler {
    @Override
    public void handle(HttpRequest req, HttpResponse res, RequestHandler next) {
        long start = System.currentTimeMillis();
        System.out.println("Request: " + req.getMethod() + " " + req.getPath());
        next.handle(req, res, null);
        System.out.println("Response: " + res.getStatus() + " in " + (System.currentTimeMillis() - start) + "ms");
    }
}

// Pipeline builder
public class FilterChain {
    private final List<RequestHandler> filters;
    private final RequestHandler finalHandler;

    public FilterChain(List<RequestHandler> filters, RequestHandler finalHandler) {
        this.filters = filters;
        this.finalHandler = finalHandler;
    }

    public void execute(HttpRequest req, HttpResponse res) {
        buildChain(0).handle(req, res, null);
    }

    private RequestHandler buildChain(int index) {
        if (index >= filters.size()) return finalHandler;
        RequestHandler current = filters.get(index);
        RequestHandler rest = buildChain(index + 1);
        return (req, res, ignored) -> current.handle(req, res, rest);
    }
}
```

---

### Q45. When should you use Template Method (inheritance-based variation) versus Strategy (composition-based variation)?

**Model Answer:** Template Method defines the skeleton of an algorithm in a base class and lets subclasses override specific steps — use it when the overall algorithm structure is stable and fixed, and only specific steps vary, especially when those steps need access to the parent class's state or protected helper methods. Strategy externalizes the entire varying algorithm and injects it — use it when the algorithm can change independently of the context, when you want runtime swappability, or when the varying part has no dependency on the host object's internals. Template Method creates a tighter coupling (subclass depends on parent) and cannot be changed at runtime; Strategy is looser and more testable.

```java
// Template Method: fixed algorithm skeleton, steps overridden by subclasses
public abstract class DataProcessor {
    // Template method: algorithm skeleton (final = cannot be overridden)
    public final void process() {
        readData();
        processData();
        writeResults();
        cleanup();
    }

    protected abstract void readData();
    protected abstract void processData();
    protected abstract void writeResults();

    protected void cleanup() { System.out.println("Default cleanup"); } // hook with default
}

public class CsvDataProcessor extends DataProcessor {
    @Override protected void readData() { System.out.println("Reading CSV"); }
    @Override protected void processData() { System.out.println("Parsing CSV records"); }
    @Override protected void writeResults() { System.out.println("Writing to DB"); }
}

// Strategy: whole algorithm is injected — swappable at runtime
public interface DataProcessingStrategy {
    void process(DataContext context);
}

public class CsvProcessingStrategy implements DataProcessingStrategy {
    @Override public void process(DataContext ctx) { /* CSV logic */ }
}

public class XmlProcessingStrategy implements DataProcessingStrategy {
    @Override public void process(DataContext ctx) { /* XML logic */ }
}

public class DataPipeline {
    private DataProcessingStrategy strategy;

    public void setStrategy(DataProcessingStrategy strategy) { this.strategy = strategy; }

    public void run(DataContext ctx) {
        strategy.process(ctx); // swappable at runtime
    }
}
```

---

### Q46. How does the Iterator pattern work in Java, and what is the difference between external and internal iteration?

**Model Answer:** The Iterator pattern provides a way to sequentially access elements of a collection without exposing its underlying structure. Java's `Iterable<T>` and `Iterator<T>` interfaces form the standard external iteration contract: the caller explicitly controls iteration by calling `hasNext()` and `next()`. Internal iteration, introduced with Java 8's `forEach` and `Stream` API, inverts control — the caller provides a function and the collection applies it to each element, allowing the collection to optimize traversal (e.g., parallel streams). Custom iterators are valuable for tree traversal, infinite sequences, or lazy loading from a data source.

```java
// Custom external iterator: tree pre-order traversal
public class TreeNode<T> {
    T value;
    List<TreeNode<T>> children = new ArrayList<>();

    public TreeNode(T value) { this.value = value; }
}

public class PreOrderIterator<T> implements Iterator<T> {
    private final Deque<TreeNode<T>> stack = new ArrayDeque<>();

    public PreOrderIterator(TreeNode<T> root) {
        if (root != null) stack.push(root);
    }

    @Override public boolean hasNext() { return !stack.isEmpty(); }

    @Override
    public T next() {
        if (!hasNext()) throw new NoSuchElementException();
        TreeNode<T> node = stack.pop();
        // Push children in reverse so leftmost is processed first
        ListIterator<TreeNode<T>> it = node.children.listIterator(node.children.size());
        while (it.hasPrevious()) stack.push(it.previous());
        return node.value;
    }
}

// Making Tree Iterable (external iteration)
public class Tree<T> implements Iterable<T> {
    private final TreeNode<T> root;
    public Tree(TreeNode<T> root) { this.root = root; }

    @Override
    public Iterator<T> iterator() { return new PreOrderIterator<>(root); }
}

// Usage: for-each loop works because of Iterable
Tree<Integer> tree = buildTree();
for (int val : tree) { System.out.print(val + " "); }

// Internal iteration via Stream
tree.stream().filter(v -> v > 5).forEach(System.out::println);
```

---

### Q47. How does the Mediator pattern reduce coupling between components, and when should you use it?

**Model Answer:** The Mediator pattern introduces a central coordinator object that encapsulates how a set of objects interact — instead of each object having direct references to all others (O(n^2) relationships), each object only knows about the mediator (O(n) relationships). This reduces coupling dramatically in complex UIs (where clicking a button might enable/disable 5 other controls) or in chat systems (where a message from one user must be routed to others). The downside is that the mediator can become a "god object" if it grows too large; mitigate by keeping mediator interactions event-driven and by decomposing large mediators into smaller ones.

```java
// Mediator interface
public interface DialogMediator {
    void notify(Component sender, String event);
}

// Concrete mediator: an authentication dialog
public class AuthDialog implements DialogMediator {
    private final TextField usernameField;
    private final TextField passwordField;
    private final Button loginButton;
    private final Checkbox rememberMeCheckbox;

    public AuthDialog(TextField u, TextField p, Button b, Checkbox c) {
        this.usernameField = u; this.passwordField = p;
        this.loginButton = b; this.rememberMeCheckbox = c;
        // Register mediator with each component
        u.setMediator(this); p.setMediator(this); b.setMediator(this); c.setMediator(this);
    }

    @Override
    public void notify(Component sender, String event) {
        if (sender == usernameField && event.equals("changed")) {
            loginButton.setEnabled(!usernameField.getText().isEmpty() && !passwordField.getText().isEmpty());
        } else if (sender == rememberMeCheckbox && event.equals("checked")) {
            passwordField.setVisible(!rememberMeCheckbox.isChecked());
        }
    }
}

// Components only reference the mediator
public abstract class Component {
    protected DialogMediator mediator;
    public void setMediator(DialogMediator m) { this.mediator = m; }
    protected void changed(String event) { mediator.notify(this, event); }
}

public class TextField extends Component {
    private String text = "";
    public void setText(String t) { this.text = t; changed("changed"); }
    public String getText() { return text; }
}
```

---

### Q48. Describe the Memento pattern and how it captures and restores object state in Java.

**Model Answer:** The Memento pattern captures an object's internal state into a separate "memento" object without exposing that state's implementation details, and allows the object to be restored to that state later. The three roles are: Originator (the object whose state is saved), Memento (the snapshot of state — should be immutable), and Caretaker (stores and manages mementos without inspecting their contents). In Java, a natural implementation uses a private nested class for the memento to prevent external access to internal state while still being accessible from the originator.

```java
public class TextDocument {
    private String content;
    private int cursorPosition;

    public TextDocument(String content) { this.content = content; this.cursorPosition = 0; }

    public void type(String text) {
        content = content.substring(0, cursorPosition) + text + content.substring(cursorPosition);
        cursorPosition += text.length();
    }

    // Create memento: snapshot of current state
    public Memento save() {
        return new Memento(content, cursorPosition);
    }

    // Restore from memento
    public void restore(Memento memento) {
        this.content = memento.content;
        this.cursorPosition = memento.cursorPosition;
    }

    // Memento: private nested class — only TextDocument can read its internals
    public static final class Memento {
        private final String content;
        private final int cursorPosition;

        private Memento(String content, int cursorPosition) {
            this.content = content;
            this.cursorPosition = cursorPosition;
        }
    }

    @Override public String toString() { return content; }
}

// Caretaker: manages history without knowing memento internals
public class DocumentHistory {
    private final Deque<TextDocument.Memento> history = new ArrayDeque<>();

    public void save(TextDocument doc) { history.push(doc.save()); }

    public void undo(TextDocument doc) {
        if (!history.isEmpty()) doc.restore(history.pop());
    }
}

// Usage
TextDocument doc = new TextDocument("Hello");
DocumentHistory history = new DocumentHistory();
history.save(doc);     // save "Hello"
doc.type(" World");    // now "Hello World"
history.save(doc);
doc.type("!!!");       // now "Hello World!!!"
history.undo(doc);     // back to "Hello World"
history.undo(doc);     // back to "Hello"
```

---

### Q49. How does the Visitor pattern let you add operations to a class hierarchy without modifying the classes?

**Model Answer:** The Visitor pattern separates an algorithm from the object structure it operates on by adding an `accept(Visitor)` method to each element class. New operations are added as new `Visitor` implementations — none of the element classes change. The double-dispatch mechanism (calling `element.accept(visitor)` which then calls `visitor.visit(element)`) ensures the correct overload is called for the concrete element type even when referenced through the abstract type. The main drawback is that adding new element types requires updating all existing visitors.

```java
// Element hierarchy: each accepts a visitor
public interface DocumentElement {
    void accept(DocumentVisitor visitor);
}

public class Text implements DocumentElement {
    private final String content;
    public Text(String content) { this.content = content; }
    public String getContent() { return content; }
    @Override public void accept(DocumentVisitor v) { v.visit(this); } // double dispatch
}

public class Image implements DocumentElement {
    private final String url;
    private final int width, height;
    public Image(String url, int w, int h) { this.url = url; width = w; height = h; }
    public String getUrl() { return url; }
    public int getWidth() { return width; }
    public int getHeight() { return height; }
    @Override public void accept(DocumentVisitor v) { v.visit(this); }
}

// Visitor interface: one method per concrete element type
public interface DocumentVisitor {
    void visit(Text text);
    void visit(Image image);
}

// New operation as a new visitor — no changes to element classes
public class HtmlExportVisitor implements DocumentVisitor {
    private final StringBuilder sb = new StringBuilder();

    @Override public void visit(Text text) { sb.append("<p>").append(text.getContent()).append("</p>"); }
    @Override public void visit(Image img) {
        sb.append("<img src='").append(img.getUrl())
          .append("' width='").append(img.getWidth())
          .append("' height='").append(img.getHeight()).append("'/>");
    }

    public String getHtml() { return sb.toString(); }
}

public class WordCountVisitor implements DocumentVisitor {
    private int count = 0;
    @Override public void visit(Text text) { count += text.getContent().split("\\s+").length; }
    @Override public void visit(Image img) { /* images have no words */ }
    public int getCount() { return count; }
}

// Usage
List<DocumentElement> doc = List.of(new Text("Hello world"), new Image("logo.png", 100, 50), new Text("Goodbye"));
HtmlExportVisitor html = new HtmlExportVisitor();
doc.forEach(el -> el.accept(html));
System.out.println(html.getHtml());
```

---

### Q50. What is the Interpreter pattern and when should you use a parser library instead?

**Model Answer:** The Interpreter pattern defines a grammar for a simple language and provides an interpreter to evaluate sentences in that grammar — each grammar rule becomes a class, and parsing constructs an AST of interpreter objects. It is appropriate for simple, stable grammars (SQL-like query filters, math expression evaluators, domain-specific rule engines) where the grammar rarely changes and the implementation team is comfortable building a recursive descent parser. For complex or evolving grammars (real SQL, programming languages, configuration languages like YAML/TOML), use a dedicated parser library (ANTLR, JavaCC, PEG.js) because the Interpreter pattern scales poorly — adding new operators requires adding classes and touching the entire grammar.

```java
// Simple expression interpreter: evaluates "3 + 5 * 2" style expressions
public interface Expression {
    int interpret();
}

public class NumberExpression implements Expression {
    private final int value;
    public NumberExpression(int value) { this.value = value; }
    @Override public int interpret() { return value; }
}

public class AddExpression implements Expression {
    private final Expression left, right;
    public AddExpression(Expression l, Expression r) { left = l; right = r; }
    @Override public int interpret() { return left.interpret() + right.interpret(); }
}

public class MultiplyExpression implements Expression {
    private final Expression left, right;
    public MultiplyExpression(Expression l, Expression r) { left = l; right = r; }
    @Override public int interpret() { return left.interpret() * right.interpret(); }
}

// Build AST for: 3 + 5 * 2  (respecting precedence)
Expression expr = new AddExpression(
    new NumberExpression(3),
    new MultiplyExpression(new NumberExpression(5), new NumberExpression(2))
);
System.out.println(expr.interpret()); // 13

// When to use a library instead: use ANTLR for this
// grammar Expr; expr: term (('+' | '-') term)* ;
// term: factor (('*' | '/') factor)* ;
// factor: NUMBER | '(' expr ')' ;
```

---

### Q51. What is the Null Object pattern and how does it eliminate null checks in Java?

**Model Answer:** The Null Object pattern provides a default, do-nothing implementation of an interface that represents the absence of a real object, eliminating the need for null checks throughout the codebase. Instead of returning `null` when an object is not found, you return a null object (e.g., `NoOpLogger`, `EmptyList`, `GuestUser`). The calling code treats it like any other implementation and the null object silently does nothing or returns safe defaults, making the code free of `if (obj != null)` guards.

```java
public interface Logger {
    void info(String message);
    void error(String message, Throwable t);
}

public class ConsoleLogger implements Logger {
    @Override public void info(String msg) { System.out.println("INFO: " + msg); }
    @Override public void error(String msg, Throwable t) { System.err.println("ERROR: " + msg); t.printStackTrace(); }
}

// Null Object: do-nothing implementation
public class NoOpLogger implements Logger {
    public static final NoOpLogger INSTANCE = new NoOpLogger();
    private NoOpLogger() {}
    @Override public void info(String msg) { /* do nothing */ }
    @Override public void error(String msg, Throwable t) { /* do nothing */ }
}

public class OrderService {
    private final Logger logger;

    // Accept a logger; if none provided, use NoOp — never null
    public OrderService(Logger logger) {
        this.logger = Objects.requireNonNullElse(logger, NoOpLogger.INSTANCE);
    }

    public void placeOrder(Order order) {
        logger.info("Placing order: " + order.getId()); // no null check needed
        // ... business logic
        logger.info("Order placed successfully");
    }
}
```

---

### Q52. What is the Specification pattern and how does it let you combine business rules composably in Java?

**Model Answer:** The Specification pattern encapsulates a business rule as an object with a `isSatisfiedBy(T candidate)` method, allowing rules to be combined using boolean operators (`and`, `or`, `not`) without modifying the rule classes themselves. This makes complex query predicates readable, reusable, and testable in isolation. It aligns naturally with Java 8 `Predicate<T>` which already provides `and()`, `or()`, and `negate()` combinators; the pattern adds the semantic richness of named business rules.

```java
// Specification interface
public interface Specification<T> {
    boolean isSatisfiedBy(T candidate);

    default Specification<T> and(Specification<T> other) {
        return candidate -> this.isSatisfiedBy(candidate) && other.isSatisfiedBy(candidate);
    }

    default Specification<T> or(Specification<T> other) {
        return candidate -> this.isSatisfiedBy(candidate) || other.isSatisfiedBy(candidate);
    }

    default Specification<T> not() {
        return candidate -> !this.isSatisfiedBy(candidate);
    }
}

// Named business rule specifications
public class PremiumCustomerSpec implements Specification<Customer> {
    @Override public boolean isSatisfiedBy(Customer c) { return c.getTotalSpend() > 10_000; }
}

public class ActiveCustomerSpec implements Specification<Customer> {
    @Override public boolean isSatisfiedBy(Customer c) { return c.getLastOrderDate().isAfter(LocalDate.now().minusMonths(6)); }
}

public class VerifiedEmailSpec implements Specification<Customer> {
    @Override public boolean isSatisfiedBy(Customer c) { return c.isEmailVerified(); }
}

// Composing rules: readable, no if/else spaghetti
Specification<Customer> eligibleForPromotion =
    new PremiumCustomerSpec()
        .and(new ActiveCustomerSpec())
        .and(new VerifiedEmailSpec());

List<Customer> eligible = customers.stream()
    .filter(eligibleForPromotion::isSatisfiedBy)
    .collect(Collectors.toList());
```

---

### Q53. What is the Repository pattern and how does it abstract data access in Java?

**Model Answer:** The Repository pattern mediates between the domain layer and the data mapping layer, acting as an in-memory collection of domain objects — callers think in domain terms (find orders by customer, save an order) rather than in persistence terms (SQL, JPA queries). The interface is defined in the domain layer (applying DIP), and the concrete implementation lives in the infrastructure layer. This makes the domain model independent of the persistence technology, enables easy testing with in-memory implementations, and provides a single place to enforce data access rules.

```java
// Repository interface: defined in the domain layer, speaks domain language
public interface OrderRepository {
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomerId(CustomerId customerId);
    List<Order> findPendingOrdersOlderThan(Duration age);
    void save(Order order);
    void delete(OrderId id);
}

// In-memory implementation for tests
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<OrderId, Order> store = new ConcurrentHashMap<>();

    @Override public Optional<Order> findById(OrderId id) { return Optional.ofNullable(store.get(id)); }
    @Override public List<Order> findByCustomerId(CustomerId cid) {
        return store.values().stream().filter(o -> o.getCustomerId().equals(cid)).collect(Collectors.toList());
    }
    @Override public List<Order> findPendingOrdersOlderThan(Duration age) {
        Instant cutoff = Instant.now().minus(age);
        return store.values().stream()
            .filter(o -> o.getStatus() == OrderStatus.PENDING && o.getCreatedAt().isBefore(cutoff))
            .collect(Collectors.toList());
    }
    @Override public void save(Order order) { store.put(order.getId(), order); }
    @Override public void delete(OrderId id) { store.remove(id); }
}

// JPA implementation in infrastructure layer
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final EntityManager em;

    @Override
    public Optional<Order> findById(OrderId id) {
        return Optional.ofNullable(em.find(OrderEntity.class, id.getValue()))
            .map(OrderMapper::toDomain);
    }
    // ... other methods
}
```

---

### Q54. Why is Dependency Injection preferred over Service Locator?

**Model Answer:** Both patterns decouple a class from its dependency creation, but DI is transparent (dependencies are declared in the constructor, making them visible and explicit) while Service Locator is opaque (dependencies are requested imperatively via a global registry, hiding them from the class's public API). With DI, a class's dependencies are fully visible at compile time in the constructor signature — the compiler enforces them, IDEs can analyze them, and tests can easily inject mocks. With Service Locator, the class has a hidden coupling to the locator; you cannot tell what a class needs without reading its body; and tests require configuring the global locator before each test, causing test ordering dependencies.

```java
// Service Locator: hidden coupling, hard to test
public class OrderService {
    public void placeOrder(Order order) {
        OrderRepository repo = ServiceLocator.get(OrderRepository.class); // hidden dependency
        EmailService email = ServiceLocator.get(EmailService.class);      // hidden dependency
        repo.save(order);
        email.sendConfirmation(order);
    }
}
// To test: must configure ServiceLocator registry — test setup is far from the class

// Dependency Injection: explicit, testable, compiler-verified
public class OrderService {
    private final OrderRepository repo;
    private final EmailService email;

    public OrderService(OrderRepository repo, EmailService email) { // explicit at construction
        this.repo = repo;
        this.email = email;
    }

    public void placeOrder(Order order) {
        repo.save(order);
        email.sendConfirmation(order);
    }
}
// To test: just pass mocks to the constructor
OrderService svc = new OrderService(mockRepo, mockEmail);
```

---

### Q55. How do you implement AOP-style cross-cutting concerns using the Decorator/Interceptor chain pattern in Java?

**Model Answer:** An interceptor chain applies a sequence of cross-cutting concerns (logging, metrics, retry, circuit breaking, auth) around a core operation without modifying the core implementation, using the same composition mechanism as Decorator. Each interceptor implements the same interface as the target, wraps the next interceptor in the chain, and can execute logic before and after delegation. This is the mechanism behind Spring AOP proxies, gRPC interceptors, and OkHttp interceptors. The key advantage over annotation-based AOP is that the chain is visible, explicit, and testable without a proxy framework.

```java
// Core service interface
public interface ProductService {
    Product getProduct(String id);
}

// Core implementation
public class DefaultProductService implements ProductService {
    @Override public Product getProduct(String id) { /* database fetch */ return new Product(id); }
}

// Logging interceptor
public class LoggingInterceptor implements ProductService {
    private final ProductService next;
    public LoggingInterceptor(ProductService next) { this.next = next; }

    @Override
    public Product getProduct(String id) {
        System.out.println("CALL getProduct(" + id + ")");
        long t = System.currentTimeMillis();
        Product p = next.getProduct(id);
        System.out.printf("RETURN getProduct(%s) in %dms%n", id, System.currentTimeMillis() - t);
        return p;
    }
}

// Caching interceptor
public class CachingInterceptor implements ProductService {
    private final ProductService next;
    private final Map<String, Product> cache = new ConcurrentHashMap<>();

    public CachingInterceptor(ProductService next) { this.next = next; }

    @Override
    public Product getProduct(String id) {
        return cache.computeIfAbsent(id, k -> next.getProduct(k));
    }
}

// Retry interceptor
public class RetryInterceptor implements ProductService {
    private final ProductService next;
    private final int maxRetries;

    public RetryInterceptor(ProductService next, int maxRetries) { this.next = next; this.maxRetries = maxRetries; }

    @Override
    public Product getProduct(String id) {
        for (int attempt = 1; attempt <= maxRetries; attempt++) {
            try { return next.getProduct(id); }
            catch (RuntimeException e) {
                if (attempt == maxRetries) throw e;
                System.out.println("Retry " + attempt + " for " + id);
            }
        }
        throw new IllegalStateException("Unreachable");
    }
}

// Composing the chain: innermost to outermost
ProductService service = new LoggingInterceptor(
    new CachingInterceptor(
        new RetryInterceptor(
            new DefaultProductService(), 3
        )
    )
);
// Call flow: Logging -> Cache -> Retry -> Default
```

---
## Category 4: LLD Design Scenarios

### Q56. Design a Parking Lot System

**Problem Statement:** Design a parking lot that supports multiple floors, multiple vehicle types (motorcycle, car, truck), and tracks space availability.

**What to Discuss:**
1. Vehicle type hierarchy and how it determines space compatibility
2. Parking space types and the assignment algorithm (best-fit vs first-fit)
3. Ticket generation, entry/exit flow, and fee calculation
4. Concurrency: how to prevent double-allocation of spaces
5. Extensibility: adding new vehicle or space types without restructuring

**Key Classes/Interfaces:** `ParkingLot`, `ParkingFloor`, `ParkingSpace`, `Vehicle` (abstract), `Car`, `Motorcycle`, `Truck`, `ParkingTicket`, `FeeCalculator`, `EntryPanel`, `ExitPanel`

**Key Design Patterns:** Strategy (fee calculation), Factory (vehicle creation), Singleton (ParkingLot), Observer (space availability updates)

**Model Answer:** The core design centers on a `ParkingLot` singleton that holds `ParkingFloor` objects, each managing a list of `ParkingSpace` instances categorized by `SpaceType` (COMPACT, LARGE, MOTORCYCLE). When a vehicle arrives, the lot selects the nearest available compatible space using a `SpaceAssignmentStrategy`, marks it `OCCUPIED`, and issues a `ParkingTicket` with entry timestamp. On exit, the `FeeCalculator` strategy computes the charge based on duration and vehicle type, space is released back to `AVAILABLE`, and the ticket is closed.

```java
public enum VehicleType { MOTORCYCLE, CAR, TRUCK }
public enum SpaceType { MOTORCYCLE, COMPACT, LARGE }
public enum SpaceStatus { AVAILABLE, OCCUPIED, RESERVED, MAINTENANCE }

public abstract class Vehicle {
    protected final String licensePlate;
    protected final VehicleType type;
    public Vehicle(String licensePlate, VehicleType type) {
        this.licensePlate = licensePlate; this.type = type;
    }
    public VehicleType getType() { return type; }
    public String getLicensePlate() { return licensePlate; }
}

public class Car extends Vehicle {
    public Car(String plate) { super(plate, VehicleType.CAR); }
}

public class ParkingSpace {
    private final String spaceId;
    private final SpaceType spaceType;
    private volatile SpaceStatus status = SpaceStatus.AVAILABLE;
    private Vehicle parkedVehicle;

    public ParkingSpace(String spaceId, SpaceType spaceType) {
        this.spaceId = spaceId; this.spaceType = spaceType;
    }

    public synchronized boolean assignVehicle(Vehicle vehicle) {
        if (status != SpaceStatus.AVAILABLE) return false;
        this.parkedVehicle = vehicle;
        this.status = SpaceStatus.OCCUPIED;
        return true;
    }

    public synchronized void removeVehicle() {
        this.parkedVehicle = null;
        this.status = SpaceStatus.AVAILABLE;
    }

    public boolean isAvailable() { return status == SpaceStatus.AVAILABLE; }
    public SpaceType getSpaceType() { return spaceType; }
    public String getSpaceId() { return spaceId; }
}

public class ParkingTicket {
    private final String ticketId;
    private final Vehicle vehicle;
    private final ParkingSpace space;
    private final Instant entryTime;
    private Instant exitTime;
    private double fee;

    public ParkingTicket(String ticketId, Vehicle vehicle, ParkingSpace space) {
        this.ticketId = ticketId;
        this.vehicle = vehicle;
        this.space = space;
        this.entryTime = Instant.now();
    }

    public void closeTicket(double fee) { this.exitTime = Instant.now(); this.fee = fee; }
    public Instant getEntryTime() { return entryTime; }
    public Vehicle getVehicle() { return vehicle; }
    public ParkingSpace getSpace() { return space; }
}

public interface FeeCalculator {
    double calculate(ParkingTicket ticket);
}

public class HourlyFeeCalculator implements FeeCalculator {
    private static final double RATE_PER_HOUR = 3.0;
    @Override
    public double calculate(ParkingTicket ticket) {
        Duration duration = Duration.between(ticket.getEntryTime(), Instant.now());
        long hours = Math.max(1, (long) Math.ceil(duration.toMinutes() / 60.0));
        return hours * RATE_PER_HOUR;
    }
}
```

---

### Q57. Design an Elevator System

**Problem Statement:** Design an elevator control system for a building with N floors and M elevators, supporting floor requests from inside and outside elevators.

**What to Discuss:**
1. Request types: internal (floor selected inside car) vs external (up/down button on floor)
2. Scheduling algorithm: SCAN (elevator continues in current direction), LOOK, or FCFS
3. Elevator state machine: IDLE, MOVING_UP, MOVING_DOWN, DOORS_OPEN
4. How multiple elevators are dispatched to minimize wait time
5. Thread safety: elevator movement and request queue updates from multiple threads

**Key Classes/Interfaces:** `ElevatorSystem`, `Elevator`, `ElevatorState`, `Request` (internal/external), `Scheduler`, `Direction`, `Floor`

**Key Design Patterns:** State (elevator state machine), Strategy (scheduling algorithm), Observer (floor request notifications)

**Model Answer:** The system has a central `ElevatorSystem` that receives `ExternalRequest` (floor + direction) and dispatches to the optimal `Elevator` using a `Scheduler` strategy. Each `Elevator` maintains a sorted set of destination floors and a `Direction`, implementing the SCAN algorithm: it services all requests in the current direction before reversing. The elevator's lifecycle is governed by a state machine (`IDLE`, `MOVING_UP`, `MOVING_DOWN`, `DOORS_OPEN`) — transitions are driven by the movement loop, which runs in a dedicated thread per elevator.

```java
public enum Direction { UP, DOWN, IDLE }
public enum ElevatorStatus { IDLE, MOVING_UP, MOVING_DOWN, DOORS_OPEN }

public class ExternalRequest {
    private final int floor;
    private final Direction direction;
    public ExternalRequest(int floor, Direction direction) { this.floor = floor; this.direction = direction; }
    public int getFloor() { return floor; }
    public Direction getDirection() { return direction; }
}

public class Elevator implements Runnable {
    private final int id;
    private volatile int currentFloor = 1;
    private volatile ElevatorStatus status = ElevatorStatus.IDLE;
    private final TreeSet<Integer> upQueue = new TreeSet<>();   // floors above
    private final TreeSet<Integer> downQueue = new TreeSet<>(Comparator.reverseOrder()); // floors below

    public Elevator(int id) { this.id = id; }

    public synchronized void addDestination(int floor) {
        if (floor > currentFloor) upQueue.add(floor);
        else if (floor < currentFloor) downQueue.add(floor);
    }

    @Override
    public void run() {
        while (true) {
            if (status == ElevatorStatus.MOVING_UP && !upQueue.isEmpty()) {
                moveTo(upQueue.first());
                upQueue.remove(currentFloor);
            } else if (status == ElevatorStatus.MOVING_DOWN && !downQueue.isEmpty()) {
                moveTo(downQueue.first());
                downQueue.remove(currentFloor);
            } else if (!upQueue.isEmpty()) {
                status = ElevatorStatus.MOVING_UP;
            } else if (!downQueue.isEmpty()) {
                status = ElevatorStatus.MOVING_DOWN;
            } else {
                status = ElevatorStatus.IDLE;
            }
        }
    }

    private void moveTo(int floor) {
        System.out.printf("Elevator %d moving from %d to %d%n", id, currentFloor, floor);
        currentFloor = floor;
        status = ElevatorStatus.DOORS_OPEN;
        // simulate door open/close delay
    }

    public int getCurrentFloor() { return currentFloor; }
    public ElevatorStatus getStatus() { return status; }
}

public interface ElevatorScheduler {
    Elevator selectElevator(List<Elevator> elevators, ExternalRequest request);
}

public class NearestElevatorScheduler implements ElevatorScheduler {
    @Override
    public Elevator selectElevator(List<Elevator> elevators, ExternalRequest request) {
        return elevators.stream()
            .filter(e -> e.getStatus() == ElevatorStatus.IDLE)
            .min(Comparator.comparingInt(e -> Math.abs(e.getCurrentFloor() - request.getFloor())))
            .orElse(elevators.get(0)); // fallback to first elevator
    }
}
```

---

### Q58. Design a Library Management System

**Problem Statement:** Design a library system that manages books, members, lending, reservations, and overdue fines.

**What to Discuss:**
1. Book vs BookItem: a book title can have multiple physical copies
2. Member lifecycle: registration, borrowing limits, suspension for overdue items
3. Lending flow: search → reserve → check out → return → fine calculation
4. Reservation queue: if all copies are checked out, member joins a queue
5. Notification: member is notified when a reserved book becomes available

**Key Classes/Interfaces:** `Library`, `Book`, `BookItem`, `Member`, `Lending`, `Reservation`, `Fine`, `BookSearch`, `NotificationService`

**Key Design Patterns:** Observer (availability notification), Strategy (fine calculation), Repository (book and member data access), Factory (search criteria)

**Model Answer:** The key modeling insight is separating `Book` (the catalog entry with ISBN, title, author) from `BookItem` (a specific physical copy with a barcode, condition, and checkout status). A `Member` can borrow up to a configured limit of `BookItem`s simultaneously; each loan is represented by a `Lending` record with issue date and due date. When a book has no available `BookItem`, the member is added to a `ReservationQueue`; when a copy is returned, the first member in the queue is notified via `NotificationService` and the item is held for them for 48 hours.

```java
public enum BookStatus { AVAILABLE, CHECKED_OUT, RESERVED, LOST, DAMAGED }
public enum MemberStatus { ACTIVE, SUSPENDED, EXPIRED }

public class Book {
    private final String isbn;
    private final String title;
    private final String author;
    private final String subject;
    private final List<BookItem> items = new ArrayList<>();

    public Book(String isbn, String title, String author, String subject) {
        this.isbn = isbn; this.title = title; this.author = author; this.subject = subject;
    }

    public void addItem(BookItem item) { items.add(item); }
    public List<BookItem> getAvailableItems() {
        return items.stream().filter(i -> i.getStatus() == BookStatus.AVAILABLE).collect(Collectors.toList());
    }
    public String getIsbn() { return isbn; }
    public String getTitle() { return title; }
}

public class BookItem {
    private final String barcode;
    private final Book book;
    private volatile BookStatus status = BookStatus.AVAILABLE;
    private Member currentBorrower;

    public BookItem(String barcode, Book book) { this.barcode = barcode; this.book = book; }

    public synchronized boolean checkOut(Member member) {
        if (status != BookStatus.AVAILABLE) return false;
        this.status = BookStatus.CHECKED_OUT;
        this.currentBorrower = member;
        return true;
    }

    public synchronized void returnItem() {
        this.status = BookStatus.AVAILABLE;
        this.currentBorrower = null;
    }

    public BookStatus getStatus() { return status; }
    public String getBarcode() { return barcode; }
}

public class Lending {
    private final String lendingId;
    private final Member member;
    private final BookItem bookItem;
    private final LocalDate issueDate;
    private final LocalDate dueDate;
    private LocalDate returnDate;

    public Lending(Member member, BookItem bookItem, int loanDays) {
        this.lendingId = UUID.randomUUID().toString();
        this.member = member;
        this.bookItem = bookItem;
        this.issueDate = LocalDate.now();
        this.dueDate = issueDate.plusDays(loanDays);
    }

    public boolean isOverdue() { return returnDate == null && LocalDate.now().isAfter(dueDate); }
    public long daysOverdue() { return ChronoUnit.DAYS.between(dueDate, LocalDate.now()); }
    public void returnBook() { this.returnDate = LocalDate.now(); bookItem.returnItem(); }
}
```

---

### Q59. Design a Hotel Booking System

**Problem Statement:** Design a hotel room booking system that supports searching available rooms, making reservations, check-in/check-out, and cancellation with refund policies.

**What to Discuss:**
1. Room hierarchy: room types (SINGLE, DOUBLE, SUITE) and room instances
2. Availability search: date range query across room types
3. Booking lifecycle: RESERVED → CHECKED_IN → CHECKED_OUT / CANCELLED
4. Overbooking prevention: concurrent reservation requests for the same room
5. Refund policy: full refund before 48h, 50% within 48h, no refund on day of arrival

**Key Classes/Interfaces:** `Hotel`, `Room`, `RoomType`, `Booking`, `Guest`, `BookingService`, `RefundPolicy`, `PaymentService`

**Key Design Patterns:** Strategy (refund policy), State (booking lifecycle), Repository (room and booking persistence), Template Method (booking flow)

**Model Answer:** The availability search queries `RoomRepository` for rooms of the requested type where no `Booking` record overlaps the requested date range with status `RESERVED` or `CHECKED_IN`. Concurrent booking prevention uses optimistic locking at the database level (version column) or pessimistic locking on the room record for the critical assign-and-reserve window. The `RefundPolicy` strategy is selected based on cancellation timing and computes the refund amount independently of the booking logic, keeping the cancellation flow clean.

```java
public enum RoomType { SINGLE, DOUBLE, DELUXE, SUITE }
public enum BookingStatus { CONFIRMED, CHECKED_IN, CHECKED_OUT, CANCELLED }

public class Room {
    private final String roomNumber;
    private final RoomType type;
    private final double pricePerNight;
    private final int capacity;

    public Room(String roomNumber, RoomType type, double pricePerNight, int capacity) {
        this.roomNumber = roomNumber; this.type = type;
        this.pricePerNight = pricePerNight; this.capacity = capacity;
    }

    public String getRoomNumber() { return roomNumber; }
    public RoomType getType() { return type; }
    public double getPricePerNight() { return pricePerNight; }
}

public class Booking {
    private final String bookingId;
    private final Room room;
    private final Guest guest;
    private final LocalDate checkIn;
    private final LocalDate checkOut;
    private BookingStatus status;
    private double totalAmount;

    public Booking(Room room, Guest guest, LocalDate checkIn, LocalDate checkOut) {
        this.bookingId = UUID.randomUUID().toString();
        this.room = room; this.guest = guest;
        this.checkIn = checkIn; this.checkOut = checkOut;
        this.status = BookingStatus.CONFIRMED;
        long nights = ChronoUnit.DAYS.between(checkIn, checkOut);
        this.totalAmount = nights * room.getPricePerNight();
    }

    public boolean overlapsWith(LocalDate start, LocalDate end) {
        return !checkOut.isBefore(start) && !checkIn.isAfter(end);
    }

    public BookingStatus getStatus() { return status; }
    public void setStatus(BookingStatus status) { this.status = status; }
    public LocalDate getCheckIn() { return checkIn; }
    public double getTotalAmount() { return totalAmount; }
}

public interface RefundPolicy {
    double calculateRefund(Booking booking);
}

public class StandardRefundPolicy implements RefundPolicy {
    @Override
    public double calculateRefund(Booking booking) {
        long hoursUntilCheckIn = ChronoUnit.HOURS.between(LocalDateTime.now(),
            booking.getCheckIn().atStartOfDay());
        if (hoursUntilCheckIn >= 48) return booking.getTotalAmount();          // full refund
        if (hoursUntilCheckIn >= 24) return booking.getTotalAmount() * 0.5;   // 50% refund
        return 0.0;                                                             // no refund
    }
}
```

---

### Q60. Design a Food Delivery Order Management System

**Problem Statement:** Design the order management system for a food delivery platform (like Swiggy/DoorDash), handling order placement, restaurant confirmation, rider assignment, and delivery tracking.

**What to Discuss:**
1. Order lifecycle state machine: PLACED → CONFIRMED → PREPARING → READY → PICKED_UP → DELIVERED / CANCELLED
2. Restaurant and menu modeling: menus with items, pricing, availability
3. Rider assignment: selecting the nearest available rider
4. Real-time tracking: broadcasting order status updates to the customer
5. Cancellation window: order can only be cancelled before PREPARING state

**Key Classes/Interfaces:** `Order`, `OrderItem`, `Restaurant`, `MenuItem`, `Rider`, `Customer`, `OrderService`, `RiderAssignmentService`, `OrderTracker`

**Key Design Patterns:** State (order lifecycle), Observer (status broadcasting), Strategy (rider assignment), Command (order actions)

**Model Answer:** The `Order` class acts as the central aggregate, progressing through well-defined states that constrain which operations are valid at each step — cancellation is only permitted before `PREPARING`, rider assignment only after `CONFIRMED`. The `RiderAssignmentService` uses a location-based strategy to find the nearest `AVAILABLE` rider and atomically marks them `ASSIGNED` to prevent double-assignment. Status changes are broadcast to registered `OrderTracker` observers, decoupling the notification logic (push notification, WebSocket) from the order domain.

```java
public enum OrderStatus { PLACED, CONFIRMED, PREPARING, READY_FOR_PICKUP, PICKED_UP, DELIVERED, CANCELLED }

public class OrderItem {
    private final MenuItem menuItem;
    private final int quantity;
    private final double unitPrice;

    public OrderItem(MenuItem menuItem, int quantity) {
        this.menuItem = menuItem; this.quantity = quantity;
        this.unitPrice = menuItem.getPrice();
    }

    public double subtotal() { return unitPrice * quantity; }
}

public class Order {
    private final String orderId;
    private final Customer customer;
    private final Restaurant restaurant;
    private final List<OrderItem> items;
    private volatile OrderStatus status = OrderStatus.PLACED;
    private Rider assignedRider;
    private final Instant placedAt = Instant.now();
    private final List<OrderStatusListener> listeners = new CopyOnWriteArrayList<>();

    public Order(Customer customer, Restaurant restaurant, List<OrderItem> items) {
        this.orderId = UUID.randomUUID().toString();
        this.customer = customer; this.restaurant = restaurant; this.items = new ArrayList<>(items);
    }

    public double total() { return items.stream().mapToDouble(OrderItem::subtotal).sum(); }

    public synchronized void transitionTo(OrderStatus newStatus) {
        validateTransition(newStatus);
        this.status = newStatus;
        listeners.forEach(l -> l.onStatusChange(this, newStatus));
    }

    private void validateTransition(OrderStatus next) {
        switch (status) {
            case PLACED:    if (next != OrderStatus.CONFIRMED && next != OrderStatus.CANCELLED) throw new IllegalStateException("Invalid transition from PLACED to " + next); break;
            case CONFIRMED: if (next != OrderStatus.PREPARING && next != OrderStatus.CANCELLED) throw new IllegalStateException("Invalid transition from CONFIRMED to " + next); break;
            case PREPARING: if (next != OrderStatus.READY_FOR_PICKUP) throw new IllegalStateException("Cannot cancel after PREPARING"); break;
            default: break;
        }
    }

    public void addStatusListener(OrderStatusListener listener) { listeners.add(listener); }
    public OrderStatus getStatus() { return status; }
    public String getOrderId() { return orderId; }
}

public interface OrderStatusListener {
    void onStatusChange(Order order, OrderStatus newStatus);
}
```

---

### Q61. Design a Ride-Sharing Dispatch System (Uber-style)

**Problem Statement:** Design the core dispatch system for a ride-sharing platform: driver matching, trip lifecycle, fare calculation, and rating.

**What to Discuss:**
1. Driver and rider models, real-time location tracking
2. Matching algorithm: finding nearby available drivers within a radius
3. Trip lifecycle: REQUESTED → DRIVER_ASSIGNED → EN_ROUTE_TO_PICKUP → IN_PROGRESS → COMPLETED / CANCELLED
4. Fare calculation: base fare + per-km + per-minute + surge multiplier
5. Rating system: mutual rating after trip completion

**Key Classes/Interfaces:** `Trip`, `Driver`, `Rider`, `Location`, `DispatchService`, `MatchingStrategy`, `FareCalculator`, `RatingService`

**Key Design Patterns:** Strategy (matching algorithm, fare calculation), Observer (trip status updates), State (trip lifecycle)

**Model Answer:** The `DispatchService` uses a `MatchingStrategy` to query nearby `AVAILABLE` drivers within a configurable radius, ranks them by distance and rating, and sends a trip request — the first driver to accept is assigned and the trip moves to `DRIVER_ASSIGNED`. The `FareCalculator` strategy computes the fare using a base rate, distance, duration, and a `SurgeMultiplier` that is computed by a separate service based on demand/supply ratio in the area. Trip state transitions are validated by the `Trip` state machine to prevent invalid operations (e.g., completing a trip that was never started).

```java
public class Location {
    private final double latitude, longitude;
    public Location(double lat, double lon) { latitude = lat; longitude = lon; }

    public double distanceTo(Location other) {
        // Haversine formula approximation
        double dlat = Math.toRadians(other.latitude - latitude);
        double dlon = Math.toRadians(other.longitude - longitude);
        double a = Math.sin(dlat/2) * Math.sin(dlat/2)
                 + Math.cos(Math.toRadians(latitude)) * Math.cos(Math.toRadians(other.latitude))
                 * Math.sin(dlon/2) * Math.sin(dlon/2);
        return 6371 * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a)); // km
    }
}

public enum TripStatus { REQUESTED, DRIVER_ASSIGNED, EN_ROUTE_TO_PICKUP, IN_PROGRESS, COMPLETED, CANCELLED }
public enum DriverStatus { AVAILABLE, ON_TRIP, OFFLINE }

public class Driver {
    private final String driverId;
    private final String name;
    private volatile Location currentLocation;
    private volatile DriverStatus status = DriverStatus.OFFLINE;
    private double rating;

    public Driver(String id, String name) { driverId = id; this.name = name; }
    public void updateLocation(Location loc) { this.currentLocation = loc; }
    public boolean isAvailable() { return status == DriverStatus.AVAILABLE; }
    public Location getCurrentLocation() { return currentLocation; }
    public double getRating() { return rating; }
    public void setStatus(DriverStatus status) { this.status = status; }
}

public interface FareCalculator {
    double calculate(double distanceKm, long durationSeconds, double surgeMultiplier);
}

public class StandardFareCalculator implements FareCalculator {
    private static final double BASE_FARE = 2.50;
    private static final double PER_KM = 1.20;
    private static final double PER_MINUTE = 0.25;

    @Override
    public double calculate(double distKm, long durSecs, double surge) {
        double fare = BASE_FARE + (distKm * PER_KM) + (durSecs / 60.0 * PER_MINUTE);
        return fare * surge;
    }
}

public class Trip {
    private final String tripId;
    private final Rider rider;
    private Driver driver;
    private volatile TripStatus status = TripStatus.REQUESTED;
    private final Location pickupLocation;
    private final Location dropoffLocation;
    private Instant startTime, endTime;
    private double fare;

    public Trip(Rider rider, Location pickup, Location dropoff) {
        tripId = UUID.randomUUID().toString();
        this.rider = rider; pickupLocation = pickup; dropoffLocation = dropoff;
    }

    public synchronized void assignDriver(Driver d) {
        if (status != TripStatus.REQUESTED) throw new IllegalStateException("Trip already has a driver");
        this.driver = d;
        this.status = TripStatus.DRIVER_ASSIGNED;
        d.setStatus(DriverStatus.ON_TRIP);
    }

    public synchronized void start() {
        if (status != TripStatus.EN_ROUTE_TO_PICKUP) throw new IllegalStateException();
        this.status = TripStatus.IN_PROGRESS;
        this.startTime = Instant.now();
    }

    public synchronized void complete(double fare) {
        if (status != TripStatus.IN_PROGRESS) throw new IllegalStateException();
        this.status = TripStatus.COMPLETED;
        this.endTime = Instant.now();
        this.fare = fare;
        driver.setStatus(DriverStatus.AVAILABLE);
    }
}
```

---

### Q62. Design an ATM Machine

**Problem Statement:** Design the software for an ATM machine supporting card insertion, PIN verification, balance inquiry, cash withdrawal, deposit, and transfer.

**What to Discuss:**
1. ATM state machine: IDLE → CARD_INSERTED → PIN_ENTERED → TRANSACTION → DISPENSING_CASH → IDLE
2. Authentication: PIN verification with lockout after N failed attempts
3. Cash dispensing: available denomination tracking, greedy denomination selection
4. Transaction atomicity: debit account and dispense cash — partial failure handling
5. Session management: timeout and forced logout on inactivity

**Key Classes/Interfaces:** `ATM`, `AtmState`, `Card`, `Account`, `Transaction`, `CashDispenser`, `CardReader`, `BankService`

**Key Design Patterns:** State (ATM state machine), Command (transactions), Template Method (transaction flow)

**Model Answer:** The ATM is modeled as a state machine with `AtmState` implementations — `IdleState`, `CardInsertedState`, `PinEnteredState`, `TransactionState` — each handling the same set of input events differently. The `CashDispenser` uses a greedy algorithm over a sorted-descending denomination list to select the minimum number of bills. Withdrawal atomicity is handled by first verifying the account balance, then dispensing cash, then committing the debit — if dispensing fails, the debit is not committed; this is a "two-phase" approach since a real ATM cannot roll back dispensed cash but can redeposit it mechanically.

```java
public interface AtmState {
    void insertCard(ATM atm, Card card);
    void enterPin(ATM atm, int pin);
    void selectTransaction(ATM atm, TransactionType type);
    void enterAmount(ATM atm, double amount);
    void ejectCard(ATM atm);
}

public class IdleState implements AtmState {
    @Override
    public void insertCard(ATM atm, Card card) {
        System.out.println("Card inserted: " + card.getMaskedNumber());
        atm.setCurrentCard(card);
        atm.setState(new CardInsertedState());
    }
    @Override public void enterPin(ATM atm, int pin) { System.out.println("Please insert card first"); }
    @Override public void selectTransaction(ATM atm, TransactionType t) { System.out.println("Please insert card first"); }
    @Override public void enterAmount(ATM atm, double amount) { System.out.println("Please insert card first"); }
    @Override public void ejectCard(ATM atm) { System.out.println("No card inserted"); }
}

public class CashDispenser {
    private final Map<Integer, Integer> denominations = new TreeMap<>(Comparator.reverseOrder());

    public CashDispenser() {
        denominations.put(100, 50);
        denominations.put(50, 100);
        denominations.put(20, 200);
        denominations.put(10, 500);
    }

    public boolean canDispense(double amount) {
        int remaining = (int) amount;
        for (Map.Entry<Integer, Integer> entry : denominations.entrySet()) {
            int bills = Math.min(remaining / entry.getKey(), entry.getValue());
            remaining -= bills * entry.getKey();
        }
        return remaining == 0;
    }

    public Map<Integer, Integer> dispense(double amount) {
        Map<Integer, Integer> dispensed = new TreeMap<>();
        int remaining = (int) amount;
        for (Map.Entry<Integer, Integer> entry : denominations.entrySet()) {
            int bills = Math.min(remaining / entry.getKey(), entry.getValue());
            if (bills > 0) {
                dispensed.put(entry.getKey(), bills);
                denominations.put(entry.getKey(), entry.getValue() - bills);
                remaining -= bills * entry.getKey();
            }
        }
        if (remaining != 0) throw new IllegalStateException("Cannot dispense exact amount");
        return dispensed;
    }
}
```

---

### Q63. Design a Chess Game

**Problem Statement:** Design a two-player chess game including board representation, piece movement, turn management, check/checkmate detection, and game state.

**What to Discuss:**
1. Board and piece hierarchy: abstract Piece with concrete King, Queen, Rook, Bishop, Knight, Pawn
2. Move validation: each piece type has unique movement rules; check must be avoided
3. Special moves: castling, en passant, pawn promotion
4. Check and checkmate detection: after each move, verify if the opposing king is in check
5. Game state machine: IN_PROGRESS, CHECK, CHECKMATE, STALEMATE, DRAW

**Key Classes/Interfaces:** `ChessGame`, `Board`, `Cell`, `Piece` (abstract), `King`, `Queen`, `Rook`, `Bishop`, `Knight`, `Pawn`, `Move`, `Player`, `GameStatus`

**Key Design Patterns:** Template Method (Piece.canMove defines validation skeleton), Strategy (check detection), Command (Move for undo support)

**Model Answer:** The `Board` is an 8x8 grid of `Cell` objects, each containing an optional `Piece`. Every `Piece` subclass implements `getValidMoves(Board board)` returning legal destination cells, with the `King` additionally checking that no destination leaves it in check. After each move, the game evaluates whether the moving side has put the opponent in check; if the opponent has no valid moves while in check, it is checkmate; if no valid moves without being in check, it is stalemate. The `Move` command stores source, destination, and captured piece, enabling undo functionality.

```java
public enum PieceType { KING, QUEEN, ROOK, BISHOP, KNIGHT, PAWN }
public enum PieceColor { WHITE, BLACK }

public class Cell {
    private final int row, col;
    private Piece piece;

    public Cell(int row, int col) { this.row = row; this.col = col; }
    public boolean isEmpty() { return piece == null; }
    public Piece getPiece() { return piece; }
    public void setPiece(Piece piece) { this.piece = piece; }
    public int getRow() { return row; }
    public int getCol() { return col; }
}

public abstract class Piece {
    protected final PieceColor color;
    protected final PieceType type;
    protected boolean hasMoved = false;

    protected Piece(PieceColor color, PieceType type) { this.color = color; this.type = type; }

    public abstract List<Cell> getValidMoves(Board board, Cell currentCell);

    public PieceColor getColor() { return color; }
    public PieceType getType() { return type; }
}

public class Rook extends Piece {
    public Rook(PieceColor color) { super(color, PieceType.ROOK); }

    @Override
    public List<Cell> getValidMoves(Board board, Cell from) {
        List<Cell> moves = new ArrayList<>();
        int[][] directions = {{0,1},{0,-1},{1,0},{-1,0}};
        for (int[] dir : directions) {
            int r = from.getRow() + dir[0];
            int c = from.getCol() + dir[1];
            while (board.isValid(r, c)) {
                Cell cell = board.getCell(r, c);
                if (cell.isEmpty()) { moves.add(cell); }
                else {
                    if (cell.getPiece().getColor() != color) moves.add(cell); // can capture
                    break; // blocked
                }
                r += dir[0]; c += dir[1];
            }
        }
        return moves;
    }
}

public class Board {
    private final Cell[][] grid = new Cell[8][8];

    public Board() {
        for (int r = 0; r < 8; r++)
            for (int c = 0; c < 8; c++)
                grid[r][c] = new Cell(r, c);
    }

    public boolean isValid(int row, int col) { return row >= 0 && row < 8 && col >= 0 && col < 8; }
    public Cell getCell(int row, int col) { return grid[row][col]; }

    public boolean isKingInCheck(PieceColor color) {
        Cell kingCell = findKing(color);
        PieceColor opponent = (color == PieceColor.WHITE) ? PieceColor.BLACK : PieceColor.WHITE;
        for (int r = 0; r < 8; r++)
            for (int c = 0; c < 8; c++) {
                Cell cell = grid[r][c];
                if (!cell.isEmpty() && cell.getPiece().getColor() == opponent) {
                    if (cell.getPiece().getValidMoves(this, cell).contains(kingCell)) return true;
                }
            }
        return false;
    }

    private Cell findKing(PieceColor color) {
        for (int r = 0; r < 8; r++)
            for (int c = 0; c < 8; c++)
                if (!grid[r][c].isEmpty() && grid[r][c].getPiece().getType() == PieceType.KING
                    && grid[r][c].getPiece().getColor() == color) return grid[r][c];
        throw new IllegalStateException("King not found for " + color);
    }
}
```

---

### Q64. Design a Movie Ticket Booking System

**Problem Statement:** Design a system for booking movie tickets across multiple theaters, screens, and shows, with seat selection and payment.

**What to Discuss:**
1. Theater, screen, and show hierarchy with seat layout
2. Seat locking: temporary hold during payment to prevent double booking
3. Booking lifecycle: SEAT_SELECTED → PAYMENT_PENDING → CONFIRMED / EXPIRED
4. Pricing: base price + seat category premium + show time premium
5. Concurrency: multiple users selecting the same seat simultaneously

**Key Classes/Interfaces:** `Theater`, `Screen`, `Show`, `Seat`, `SeatCategory`, `Booking`, `Payment`, `BookingService`, `SeatLockService`

**Key Design Patterns:** State (booking lifecycle), Strategy (pricing), Repository, Template Method (booking flow)

**Model Answer:** The critical concurrency challenge is preventing two users from booking the same seat; this is solved by a `SeatLockService` that acquires a short-duration distributed lock (or database row lock) on the seat for the payment window (typically 10 minutes). The seat transitions from `AVAILABLE` → `LOCKED` (during payment) → `BOOKED` (on payment success) or back to `AVAILABLE` (on payment failure or lock expiry). The `PricingStrategy` computes the total by combining the base price for the screen/show with category premiums (PREMIUM vs REGULAR) and time-of-day multipliers.

```java
public enum SeatCategory { STANDARD, PREMIUM, VIP }
public enum SeatStatus { AVAILABLE, LOCKED, BOOKED, MAINTENANCE }

public class Seat {
    private final String seatId;
    private final int row;
    private final int col;
    private final SeatCategory category;
    private volatile SeatStatus status = SeatStatus.AVAILABLE;

    public Seat(String id, int row, int col, SeatCategory category) {
        seatId = id; this.row = row; this.col = col; this.category = category;
    }

    public synchronized boolean lock() {
        if (status == SeatStatus.AVAILABLE) { status = SeatStatus.LOCKED; return true; }
        return false;
    }

    public synchronized void book() {
        if (status != SeatStatus.LOCKED) throw new IllegalStateException("Seat must be locked before booking");
        status = SeatStatus.BOOKED;
    }

    public synchronized void release() { if (status == SeatStatus.LOCKED) status = SeatStatus.AVAILABLE; }
    public SeatStatus getStatus() { return status; }
    public SeatCategory getCategory() { return category; }
}

public class Show {
    private final String showId;
    private final Movie movie;
    private final Screen screen;
    private final LocalDateTime showTime;
    private final double basePrice;

    public Show(String id, Movie movie, Screen screen, LocalDateTime time, double basePrice) {
        showId = id; this.movie = movie; this.screen = screen; showTime = time; this.basePrice = basePrice;
    }

    public double getBasePrice() { return basePrice; }
    public Screen getScreen() { return screen; }
    public LocalDateTime getShowTime() { return showTime; }
}

public interface PricingStrategy {
    double computePrice(Show show, Seat seat);
}

public class CategoryBasedPricing implements PricingStrategy {
    @Override
    public double computePrice(Show show, Seat seat) {
        double price = show.getBasePrice();
        switch (seat.getCategory()) {
            case PREMIUM: price *= 1.5; break;
            case VIP: price *= 2.0; break;
            default: break;
        }
        // Evening shows 20% premium
        int hour = show.getShowTime().getHour();
        if (hour >= 18) price *= 1.2;
        return price;
    }
}
```

---

### Q65. Design a Vending Machine

**Problem Statement:** Design a vending machine that accepts coins/bills, dispenses products, and handles insufficient funds, out-of-stock, and change dispensing.

**What to Discuss:**
1. Product inventory management: tracking quantity per slot
2. Payment state machine: IDLE → HAS_MONEY → DISPENSING → RETURNING_CHANGE
3. Change calculation: using available coin denominations (similar to ATM)
4. Error handling: product out of stock, insufficient funds, exact change only mode
5. Admin operations: restock, collect cash, update prices

**Key Classes/Interfaces:** `VendingMachine`, `VendingMachineState`, `Product`, `Slot`, `CoinDispenser`, `Display`

**Key Design Patterns:** State (machine state machine), Strategy (change dispensing), Command (transactions)

**Model Answer:** The vending machine uses a State pattern with four states: `IdleState` (waiting for money), `HasMoneyState` (money inserted, product not yet selected), `DispensingState` (product dispensed, computing change), and `OutOfServiceState`. Each state handles events (insert coin, select product, cancel) and transitions appropriately. The `CoinDispenser` uses a greedy algorithm over sorted denominations to return change; if exact change cannot be made, the machine enters "exact change only" mode and displays a warning.

```java
public interface VendingMachineState {
    void insertMoney(VendingMachine machine, double amount);
    void selectProduct(VendingMachine machine, String productCode);
    void cancel(VendingMachine machine);
}

public class IdleState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine machine, double amount) {
        machine.addInsertedAmount(amount);
        machine.display("Inserted: $" + String.format("%.2f", machine.getInsertedAmount()));
        machine.setState(new HasMoneyState());
    }
    @Override public void selectProduct(VendingMachine machine, String code) { machine.display("Please insert money first"); }
    @Override public void cancel(VendingMachine machine) { machine.display("Nothing to cancel"); }
}

public class HasMoneyState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine machine, double amount) {
        machine.addInsertedAmount(amount);
        machine.display("Total: $" + String.format("%.2f", machine.getInsertedAmount()));
    }

    @Override
    public void selectProduct(VendingMachine machine, String code) {
        Slot slot = machine.getSlot(code);
        if (slot == null) { machine.display("Invalid product code"); return; }
        if (slot.isEmpty()) { machine.display("Product out of stock"); return; }
        if (machine.getInsertedAmount() < slot.getProduct().getPrice()) {
            machine.display("Insufficient funds. Need $" +
                String.format("%.2f", slot.getProduct().getPrice() - machine.getInsertedAmount()));
            return;
        }
        slot.dispense();
        double change = machine.getInsertedAmount() - slot.getProduct().getPrice();
        machine.display("Dispensing " + slot.getProduct().getName() + ". Change: $" + String.format("%.2f", change));
        machine.returnChange(change);
        machine.resetInsertedAmount();
        machine.setState(new IdleState());
    }

    @Override
    public void cancel(VendingMachine machine) {
        machine.display("Returning $" + String.format("%.2f", machine.getInsertedAmount()));
        machine.returnChange(machine.getInsertedAmount());
        machine.resetInsertedAmount();
        machine.setState(new IdleState());
    }
}

public class Slot {
    private final String code;
    private final Product product;
    private int quantity;

    public Slot(String code, Product product, int quantity) {
        this.code = code; this.product = product; this.quantity = quantity;
    }

    public boolean isEmpty() { return quantity == 0; }
    public void dispense() {
        if (isEmpty()) throw new IllegalStateException("Slot is empty");
        quantity--;
    }
    public void restock(int qty) { this.quantity += qty; }
    public Product getProduct() { return product; }
}
```

---

### Q66. Design a Multi-Channel Notification System

**Problem Statement:** Design a notification system that can send notifications via Email, SMS, Push Notification, and In-App channels, with user preferences and retry logic.

**What to Discuss:**
1. Notification types and channel abstraction
2. User preference management: which channels are enabled per notification type
3. Template rendering: populating notification templates with data
4. Retry logic with exponential backoff for delivery failures
5. Delivery tracking and audit log

**Key Classes/Interfaces:** `NotificationService`, `NotificationChannel`, `EmailChannel`, `SmsChannel`, `PushChannel`, `NotificationTemplate`, `UserPreference`, `DeliveryStatus`

**Key Design Patterns:** Strategy (channel selection), Observer (event-triggered notifications), Decorator (retry wrapper), Template Method (notification flow)

**Model Answer:** The `NotificationService` resolves the target channels by consulting `UserPreferenceService` for the notification type and user, renders the message from a `NotificationTemplate` populated with event data, and dispatches to each resolved `NotificationChannel`. Each channel delivery is wrapped in a `RetryDecorator` that implements exponential backoff (e.g., 1s, 2s, 4s, 8s) and records each attempt in the `NotificationAuditLog`. Channels are fully decoupled via the `NotificationChannel` interface, making it trivial to add WhatsApp or Slack channels without modifying the dispatch logic.

```java
public interface NotificationChannel {
    String getChannelType();
    DeliveryResult send(Notification notification, Recipient recipient);
}

public class EmailChannel implements NotificationChannel {
    private final EmailClient client;
    public EmailChannel(EmailClient client) { this.client = client; }

    @Override public String getChannelType() { return "EMAIL"; }

    @Override
    public DeliveryResult send(Notification notification, Recipient recipient) {
        try {
            client.send(recipient.getEmail(), notification.getSubject(), notification.getBody());
            return DeliveryResult.success();
        } catch (EmailException e) {
            return DeliveryResult.failure(e.getMessage());
        }
    }
}

public class RetryableChannel implements NotificationChannel {
    private final NotificationChannel delegate;
    private final int maxAttempts;
    private static final long BASE_DELAY_MS = 1000;

    public RetryableChannel(NotificationChannel delegate, int maxAttempts) {
        this.delegate = delegate; this.maxAttempts = maxAttempts;
    }

    @Override public String getChannelType() { return delegate.getChannelType(); }

    @Override
    public DeliveryResult send(Notification notification, Recipient recipient) {
        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            DeliveryResult result = delegate.send(notification, recipient);
            if (result.isSuccess()) return result;
            if (attempt < maxAttempts) {
                long delay = BASE_DELAY_MS * (1L << (attempt - 1)); // exponential backoff
                try { Thread.sleep(delay); } catch (InterruptedException e) { Thread.currentThread().interrupt(); break; }
            }
        }
        return DeliveryResult.failure("Max retries exceeded");
    }
}

public class NotificationService {
    private final Map<String, NotificationChannel> channels;
    private final UserPreferenceService preferenceService;
    private final TemplateRenderer templateRenderer;

    public void send(NotificationEvent event, User user) {
        List<String> enabledChannels = preferenceService.getEnabledChannels(user, event.getType());
        String renderedBody = templateRenderer.render(event.getTemplateId(), event.getData());

        for (String channelType : enabledChannels) {
            NotificationChannel channel = channels.get(channelType);
            if (channel != null) {
                Notification notification = new Notification(event.getSubject(), renderedBody);
                channel.send(notification, user.asRecipient());
            }
        }
    }
}
```

---

### Q67. Design a Rate Limiter

**Problem Statement:** Design a rate limiter that restricts a client to N requests per time window, supporting multiple algorithms and per-client configuration.

**What to Discuss:**
1. Algorithms: Token Bucket, Leaky Bucket, Fixed Window Counter, Sliding Window Log, Sliding Window Counter
2. Storage: in-memory (single node) vs Redis (distributed)
3. Per-client configuration: different limits for different API keys or users
4. Thread safety for concurrent requests from the same client
5. Response: 429 with Retry-After header, or queue the request

**Key Classes/Interfaces:** `RateLimiter`, `TokenBucketRateLimiter`, `SlidingWindowRateLimiter`, `RateLimitConfig`, `RateLimitResult`

**Key Design Patterns:** Strategy (algorithm selection), Decorator (layered rate limits), Template Method (rate limit flow)

**Model Answer:** The Token Bucket algorithm is generally preferred for its smooth burst handling: each client has a bucket with capacity N that refills at rate R tokens/second; a request consumes one token and is allowed if the bucket is non-empty, otherwise rejected with 429. The implementation must be thread-safe — use `AtomicLong` and compare-and-swap for the token count, or `synchronized` on the client bucket object. In distributed environments, the token bucket state is stored in Redis using Lua scripts for atomic check-and-decrement to prevent race conditions across nodes.

```java
public interface RateLimiter {
    RateLimitResult tryAcquire(String clientId);
}

public class TokenBucketRateLimiter implements RateLimiter {
    private final int capacity;
    private final double refillRatePerSecond;
    private final ConcurrentHashMap<String, TokenBucket> buckets = new ConcurrentHashMap<>();

    public TokenBucketRateLimiter(int capacity, double refillRatePerSecond) {
        this.capacity = capacity;
        this.refillRatePerSecond = refillRatePerSecond;
    }

    @Override
    public RateLimitResult tryAcquire(String clientId) {
        TokenBucket bucket = buckets.computeIfAbsent(clientId, k -> new TokenBucket(capacity, refillRatePerSecond));
        return bucket.tryConsume() ? RateLimitResult.allowed() : RateLimitResult.rejected(bucket.retryAfterMs());
    }

    private static class TokenBucket {
        private final int capacity;
        private final double refillRatePerSecond;
        private double tokens;
        private long lastRefillNanos;

        TokenBucket(int capacity, double rate) {
            this.capacity = capacity; this.refillRatePerSecond = rate;
            this.tokens = capacity; this.lastRefillNanos = System.nanoTime();
        }

        synchronized boolean tryConsume() {
            refill();
            if (tokens >= 1.0) { tokens -= 1.0; return true; }
            return false;
        }

        synchronized long retryAfterMs() {
            refill();
            double tokensNeeded = 1.0 - tokens;
            return (long) Math.ceil(tokensNeeded / refillRatePerSecond * 1000);
        }

        private void refill() {
            long now = System.nanoTime();
            double elapsed = (now - lastRefillNanos) / 1_000_000_000.0;
            tokens = Math.min(capacity, tokens + elapsed * refillRatePerSecond);
            lastRefillNanos = now;
        }
    }
}
```

---

### Q68. Design an In-Memory URL Shortener

**Problem Statement:** Design an in-memory URL shortener that generates short codes for long URLs, supports redirection, tracks click counts, and supports custom aliases.

**What to Discuss:**
1. Short code generation: Base62 encoding of a counter, or random alphanumeric
2. Collision handling for random codes
3. Custom alias support with validation
4. Click tracking: atomic counter per short URL
5. Expiration: optional TTL per URL

**Key Classes/Interfaces:** `UrlShortener`, `ShortUrl`, `CodeGenerator`, `UrlRepository`, `ClickTracker`

**Key Design Patterns:** Strategy (code generation), Repository (URL storage), Decorator (analytics tracking)

**Model Answer:** The cleanest approach uses a monotonic counter encoded in Base62 (characters 0-9, A-Z, a-z) — a 7-character code supports 62^7 approximately 3.5 trillion URLs. The counter is an `AtomicLong` for thread safety in a single-node in-memory scenario. Custom aliases go through the same `UrlRepository` but bypass code generation. Click counts use `AtomicLong` per short URL to allow concurrent increment without locks.

```java
public class Base62CodeGenerator {
    private static final String CHARS = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    private static final int BASE = CHARS.length();
    private final AtomicLong counter = new AtomicLong(1);

    public String nextCode() {
        return encode(counter.getAndIncrement());
    }

    private String encode(long n) {
        StringBuilder sb = new StringBuilder();
        while (n > 0) {
            sb.append(CHARS.charAt((int)(n % BASE)));
            n /= BASE;
        }
        return sb.reverse().toString();
    }
}

public class ShortUrl {
    private final String code;
    private final String originalUrl;
    private final String createdBy;
    private final Instant createdAt = Instant.now();
    private final Instant expiresAt;
    private final AtomicLong clickCount = new AtomicLong(0);

    public ShortUrl(String code, String originalUrl, String createdBy, Duration ttl) {
        this.code = code; this.originalUrl = originalUrl; this.createdBy = createdBy;
        this.expiresAt = ttl != null ? Instant.now().plus(ttl) : null;
    }

    public boolean isExpired() { return expiresAt != null && Instant.now().isAfter(expiresAt); }
    public void incrementClicks() { clickCount.incrementAndGet(); }
    public long getClickCount() { return clickCount.get(); }
    public String getOriginalUrl() { return originalUrl; }
    public String getCode() { return code; }
}

public class InMemoryUrlShortener {
    private final Map<String, ShortUrl> store = new ConcurrentHashMap<>();
    private final Base62CodeGenerator generator = new Base62CodeGenerator();

    public String shorten(String originalUrl, String customAlias, Duration ttl) {
        String code = customAlias != null ? customAlias : generator.nextCode();
        if (store.containsKey(code)) throw new IllegalArgumentException("Alias already taken: " + code);
        store.put(code, new ShortUrl(code, originalUrl, "anonymous", ttl));
        return "https://short.ly/" + code;
    }

    public String resolve(String code) {
        ShortUrl url = store.get(code);
        if (url == null) throw new NoSuchElementException("Code not found: " + code);
        if (url.isExpired()) { store.remove(code); throw new NoSuchElementException("Link has expired"); }
        url.incrementClicks();
        return url.getOriginalUrl();
    }
}
```

---

### Q69. Design a Log Aggregator

**Problem Statement:** Design a log aggregator that collects logs from multiple services, parses and enriches them, and supports querying by service, severity, and time range.

**What to Discuss:**
1. Log entry model: service, level, timestamp, message, correlation ID, metadata
2. Ingestion pipeline: receive → parse → enrich → store
3. Query model: filtering by service, level range, time range, full-text search
4. Storage: in-memory ring buffer (bounded) vs persistent store
5. Alerting: trigger alerts when error rate crosses a threshold

**Key Classes/Interfaces:** `LogEntry`, `LogLevel`, `LogIngester`, `LogParser`, `LogEnricher`, `LogRepository`, `LogQuery`, `LogQueryResult`

**Key Design Patterns:** Chain of Responsibility (ingestion pipeline), Strategy (storage backend), Observer (alert triggers), Builder (log query)

**Model Answer:** The ingestion pipeline is a Chain of Responsibility: each `LogProcessor` in the chain (parser, enricher, filter, storage writer) transforms and forwards the log entry. The `LogQuery` object is built using a Builder pattern to compose optional filters (service name, level range, time window, correlation ID) without parameter explosion. An `AlertRule` observer monitors the incoming log stream for error-rate thresholds using a sliding window counter and triggers alert callbacks when conditions are met.

```java
public enum LogLevel { TRACE(0), DEBUG(1), INFO(2), WARN(3), ERROR(4), FATAL(5);
    private final int value;
    LogLevel(int v) { value = v; }
    public int getValue() { return value; }
}

public class LogEntry {
    private final String entryId;
    private final String service;
    private final LogLevel level;
    private final Instant timestamp;
    private final String message;
    private final String correlationId;
    private final Map<String, String> metadata;

    public static Builder builder(String service, LogLevel level, String message) {
        return new Builder(service, level, message);
    }

    // ... constructor, getters

    public static class Builder {
        private final String service, message;
        private final LogLevel level;
        private String correlationId;
        private Map<String, String> metadata = new HashMap<>();
        private Instant timestamp = Instant.now();

        public Builder(String service, LogLevel level, String message) {
            this.service = service; this.level = level; this.message = message;
        }
        public Builder correlationId(String id) { correlationId = id; return this; }
        public Builder metadata(String key, String value) { metadata.put(key, value); return this; }
        public LogEntry build() { return new LogEntry(this); }
    }
}

public class LogQuery {
    private final String serviceFilter;
    private final LogLevel minLevel;
    private final Instant from;
    private final Instant to;
    private final String correlationId;

    private LogQuery(Builder b) {
        serviceFilter = b.serviceFilter; minLevel = b.minLevel;
        from = b.from; to = b.to; correlationId = b.correlationId;
    }

    public boolean matches(LogEntry entry) {
        if (serviceFilter != null && !entry.getService().equals(serviceFilter)) return false;
        if (minLevel != null && entry.getLevel().getValue() < minLevel.getValue()) return false;
        if (from != null && entry.getTimestamp().isBefore(from)) return false;
        if (to != null && entry.getTimestamp().isAfter(to)) return false;
        if (correlationId != null && !correlationId.equals(entry.getCorrelationId())) return false;
        return true;
    }

    public static class Builder {
        private String serviceFilter, correlationId;
        private LogLevel minLevel;
        private Instant from, to;
        public Builder service(String s) { serviceFilter = s; return this; }
        public Builder minLevel(LogLevel l) { minLevel = l; return this; }
        public Builder from(Instant f) { from = f; return this; }
        public Builder to(Instant t) { to = t; return this; }
        public Builder correlationId(String id) { correlationId = id; return this; }
        public LogQuery build() { return new LogQuery(this); }
    }
}
```

---

### Q70. Design a Task Scheduler

**Problem Statement:** Design a task scheduler that supports one-time delayed tasks, recurring tasks (cron-like), task cancellation, and priority-based execution.

**What to Discuss:**
1. Task representation: one-time vs recurring, priority, execution context
2. Scheduling data structure: priority queue ordered by next execution time
3. Worker thread pool: how many threads, what happens on task failure
4. Recurring task rescheduling: fixed-rate vs fixed-delay semantics
5. Cancellation: how to cancel a queued or in-progress task

**Key Classes/Interfaces:** `TaskScheduler`, `ScheduledTask`, `TaskDefinition`, `TaskExecution`, `CancellationToken`, `RecurrenceRule`

**Key Design Patterns:** Command (TaskDefinition), Strategy (recurrence rule), Template Method (task execution flow)

**Model Answer:** The scheduler uses a `PriorityBlockingQueue<ScheduledTask>` ordered by `nextExecutionTime`; a dispatcher thread polls the queue, waits until the head task is due, and submits it to a `ThreadPoolExecutor`. For recurring tasks, after execution completes (or fails), the task computes its next execution time from the `RecurrenceRule` and re-enqueues itself. Cancellation uses a `CancellationToken` (a shared `AtomicBoolean`) that the dispatcher checks before submission and the task checks at checkpoints during execution.

```java
public interface RecurrenceRule {
    Instant nextExecutionTime(Instant lastExecution);
    boolean hasMore(Instant lastExecution);
}

public class FixedRateRule implements RecurrenceRule {
    private final Duration period;
    private final int maxExecutions;
    private int executionCount = 0;

    public FixedRateRule(Duration period, int maxExecutions) {
        this.period = period; this.maxExecutions = maxExecutions;
    }

    @Override public Instant nextExecutionTime(Instant last) { return last.plus(period); }
    @Override public boolean hasMore(Instant last) { return ++executionCount < maxExecutions || maxExecutions == -1; }
}

public class ScheduledTask implements Comparable<ScheduledTask> {
    private final String taskId;
    private final Runnable job;
    private Instant nextExecutionTime;
    private final RecurrenceRule recurrenceRule;
    private final AtomicBoolean cancelled = new AtomicBoolean(false);

    public ScheduledTask(String id, Runnable job, Instant firstRun, RecurrenceRule rule) {
        taskId = id; this.job = job; nextExecutionTime = firstRun; recurrenceRule = rule;
    }

    public boolean isCancelled() { return cancelled.get(); }
    public void cancel() { cancelled.set(true); }
    public Instant getNextExecutionTime() { return nextExecutionTime; }
    public boolean reschedule() {
        if (!recurrenceRule.hasMore(nextExecutionTime)) return false;
        nextExecutionTime = recurrenceRule.nextExecutionTime(nextExecutionTime);
        return true;
    }

    @Override public int compareTo(ScheduledTask other) { return nextExecutionTime.compareTo(other.nextExecutionTime); }

    public void execute() { if (!cancelled.get()) job.run(); }
}
```

---

### Q71. Design a Cache with Eviction Policies (LRU and LFU)

**Problem Statement:** Design a generic cache with configurable capacity and eviction policies: Least Recently Used (LRU) and Least Frequently Used (LFU).

**What to Discuss:**
1. LRU data structure: HashMap + doubly linked list for O(1) get and put
2. LFU data structure: HashMap for key-to-value, HashMap for key-to-frequency, HashMap for frequency-to-LinkedHashSet of keys
3. Thread safety: read-heavy vs write-heavy workloads
4. TTL support per entry
5. Cache statistics: hit rate, miss rate, eviction count

**Key Classes/Interfaces:** `Cache<K,V>`, `LruCache<K,V>`, `LfuCache<K,V>`, `CacheEntry<K,V>`, `EvictionPolicy`

**Key Design Patterns:** Strategy (eviction policy), Decorator (statistics tracking, TTL)

**Model Answer:** LRU is implemented with a `LinkedHashMap` (access-ordered) in Java, overriding `removeEldestEntry()`, or from scratch with a `HashMap<K, Node>` and a doubly linked list where `get()` moves the accessed node to the head and `put()` evicts the tail. LFU requires three maps: `keyToValue`, `keyToFreq`, and `freqToKeys` (frequency to an insertion-ordered set of keys); on access, increment frequency and move the key to the next frequency bucket; maintain `minFreq` to know which bucket to evict from. LFU is O(1) for all operations.

```java
// LRU Cache using LinkedHashMap
public class LruCache<K, V> {
    private final int capacity;
    private final Map<K, V> cache;

    public LruCache(int capacity) {
        this.capacity = capacity;
        this.cache = Collections.synchronizedMap(new LinkedHashMap<K, V>(capacity, 0.75f, true) {
            @Override protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > capacity;
            }
        });
    }

    public V get(K key) { return cache.getOrDefault(key, null); }
    public void put(K key, V value) { cache.put(key, value); }
}

// LFU Cache from scratch -- O(1) all operations
public class LfuCache<K, V> {
    private final int capacity;
    private final Map<K, V> keyToValue = new HashMap<>();
    private final Map<K, Integer> keyToFreq = new HashMap<>();
    private final Map<Integer, LinkedHashSet<K>> freqToKeys = new HashMap<>();
    private int minFreq = 0;

    public LfuCache(int capacity) { this.capacity = capacity; }

    public V get(K key) {
        if (!keyToValue.containsKey(key)) return null;
        incrementFreq(key);
        return keyToValue.get(key);
    }

    public void put(K key, V value) {
        if (capacity <= 0) return;
        if (keyToValue.containsKey(key)) {
            keyToValue.put(key, value);
            incrementFreq(key);
            return;
        }
        if (keyToValue.size() >= capacity) evict();
        keyToValue.put(key, value);
        keyToFreq.put(key, 1);
        freqToKeys.computeIfAbsent(1, k -> new LinkedHashSet<>()).add(key);
        minFreq = 1;
    }

    private void incrementFreq(K key) {
        int freq = keyToFreq.get(key);
        keyToFreq.put(key, freq + 1);
        freqToKeys.get(freq).remove(key);
        if (freqToKeys.get(freq).isEmpty()) {
            freqToKeys.remove(freq);
            if (minFreq == freq) minFreq++;
        }
        freqToKeys.computeIfAbsent(freq + 1, k -> new LinkedHashSet<>()).add(key);
    }

    private void evict() {
        LinkedHashSet<K> keys = freqToKeys.get(minFreq);
        K evictKey = keys.iterator().next(); // least recently used among least frequent
        keys.remove(evictKey);
        if (keys.isEmpty()) freqToKeys.remove(minFreq);
        keyToValue.remove(evictKey);
        keyToFreq.remove(evictKey);
    }
}
```

---

### Q72. Design an Event Bus

**Problem Statement:** Design an in-process event bus that allows components to publish events and subscribe to specific event types, with synchronous and asynchronous delivery options.

**What to Discuss:**
1. Type-safe event subscription: subscribers register for specific event types
2. Synchronous vs asynchronous delivery
3. Error isolation: one subscriber failure should not block others
4. Wildcard / hierarchy subscriptions: subscribing to a base type receives subtypes too
5. Unsubscription and preventing memory leaks

**Key Classes/Interfaces:** `EventBus`, `Event`, `EventSubscriber<T>`, `EventPublisher`, `SubscriptionHandle`

**Key Design Patterns:** Observer, Mediator, Command

**Model Answer:** The event bus maps each event type to a list of subscribers using a `ConcurrentHashMap<Class<?>, CopyOnWriteArrayList<Subscriber>>`. On publish, the bus iterates the subscribers for the exact event class and all its superclasses (supporting hierarchy subscriptions). Each subscriber invocation is wrapped in a try-catch to isolate failures. Async delivery is handled by submitting each subscriber call to an `ExecutorService`, returning a `CompletableFuture` per delivery for the caller to optionally join.

```java
public interface EventSubscriber<T> {
    void onEvent(T event);
}

public class SubscriptionHandle {
    private final Class<?> eventType;
    private final EventSubscriber<?> subscriber;
    private final EventBus bus;

    SubscriptionHandle(Class<?> type, EventSubscriber<?> sub, EventBus bus) {
        eventType = type; subscriber = sub; this.bus = bus;
    }

    public void unsubscribe() { bus.unsubscribe(eventType, subscriber); }
}

public class EventBus {
    private final ConcurrentHashMap<Class<?>, CopyOnWriteArrayList<EventSubscriber<Object>>> registry
        = new ConcurrentHashMap<>();
    private final ExecutorService asyncExecutor = Executors.newCachedThreadPool();

    @SuppressWarnings("unchecked")
    public <T> SubscriptionHandle subscribe(Class<T> eventType, EventSubscriber<T> subscriber) {
        registry.computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>())
                .add((EventSubscriber<Object>) subscriber);
        return new SubscriptionHandle(eventType, subscriber, this);
    }

    void unsubscribe(Class<?> eventType, EventSubscriber<?> subscriber) {
        CopyOnWriteArrayList<?> subs = registry.get(eventType);
        if (subs != null) subs.remove(subscriber);
    }

    public void publish(Object event) {
        Class<?> type = event.getClass();
        while (type != null) {
            CopyOnWriteArrayList<EventSubscriber<Object>> subs = registry.get(type);
            if (subs != null) {
                for (EventSubscriber<Object> sub : subs) {
                    try { sub.onEvent(event); }
                    catch (Exception e) { System.err.println("Subscriber error: " + e.getMessage()); }
                }
            }
            type = type.getSuperclass(); // walk hierarchy for hierarchy subscriptions
        }
    }

    public void publishAsync(Object event) {
        asyncExecutor.submit(() -> publish(event));
    }
}
```

---

### Q73. Design an Inventory Management System

**Problem Statement:** Design an inventory system for a warehouse that tracks stock levels, supports reservations, handles restocking, and triggers low-stock alerts.

**What to Discuss:**
1. Product catalog vs inventory: product definition vs stock records
2. Reservation pattern: soft-reserve before order confirmation, commit on payment
3. Concurrent stock updates: preventing overselling
4. Warehouse location modeling: bins, shelves, aisles
5. Low-stock alerting: threshold-based observer pattern

**Key Classes/Interfaces:** `Inventory`, `InventoryItem`, `StockReservation`, `Product`, `Warehouse`, `RestockOrder`, `LowStockAlert`, `InventoryService`

**Key Design Patterns:** Observer (low-stock alerts), Strategy (reservation strategy), Repository (inventory persistence), Command (stock adjustments)

**Model Answer:** Each `InventoryItem` tracks `quantityOnHand`, `quantityReserved`, and `quantityAvailable = onHand - reserved`. A reservation atomically increments `quantityReserved` after checking `quantityAvailable >= requested`, preventing overselling without locking the entire table. When `quantityAvailable` drops below a threshold, the `LowStockObserver` is notified and triggers a restock workflow. Stock adjustments (receive, adjust, write-off) are recorded as `StockTransaction` commands for full audit trail.

```java
public class InventoryItem {
    private final String sku;
    private final Product product;
    private int quantityOnHand;
    private int quantityReserved;
    private final int lowStockThreshold;
    private final List<StockObserver> observers = new CopyOnWriteArrayList<>();

    public InventoryItem(String sku, Product product, int initial, int threshold) {
        this.sku = sku; this.product = product;
        quantityOnHand = initial; this.lowStockThreshold = threshold;
    }

    public int getQuantityAvailable() { return quantityOnHand - quantityReserved; }

    public synchronized boolean reserve(int qty, String reservationId) {
        if (getQuantityAvailable() < qty) return false;
        quantityReserved += qty;
        checkLowStock();
        return true;
    }

    public synchronized void commitReservation(int qty) {
        quantityReserved -= qty;
        quantityOnHand -= qty;
        checkLowStock();
    }

    public synchronized void cancelReservation(int qty) {
        quantityReserved -= qty;
    }

    public synchronized void receiveStock(int qty) {
        quantityOnHand += qty;
        notifyObservers(new StockUpdateEvent(sku, quantityOnHand, getQuantityAvailable()));
    }

    private void checkLowStock() {
        if (getQuantityAvailable() <= lowStockThreshold) {
            observers.forEach(o -> o.onLowStock(new LowStockEvent(sku, getQuantityAvailable(), lowStockThreshold)));
        }
    }

    public void addObserver(StockObserver observer) { observers.add(observer); }
    private void notifyObservers(StockUpdateEvent event) { observers.forEach(o -> o.onStockUpdate(event)); }
}
```

---

### Q74. Design a Pub-Sub System

**Problem Statement:** Design an in-process publish-subscribe system with named topics, durable subscriptions, message filtering, and at-least-once delivery semantics.

**What to Discuss:**
1. Topics and subscriptions: named topics, multiple subscribers per topic
2. Message durability: buffering undelivered messages for offline subscribers
3. Message filtering: subscribers can register a predicate to filter messages
4. Delivery guarantees: at-most-once vs at-least-once, acknowledgement model
5. Backpressure: what happens when a subscriber is slow

**Key Classes/Interfaces:** `PubSubBroker`, `Topic`, `Message`, `Publisher`, `Subscriber`, `Subscription`, `MessageFilter`

**Key Design Patterns:** Observer (core pub-sub), Strategy (message filtering, delivery policy), Command (message as command)

**Model Answer:** Each `Topic` maintains a list of `Subscription` objects, each with an optional `MessageFilter` predicate and a `BlockingQueue<Message>` as the delivery buffer. When a message is published, the broker evaluates each subscription's filter and enqueues matching messages; a per-subscription dispatcher thread dequeues and delivers to the subscriber, retrying on failure up to N times before moving to a dead-letter queue. At-least-once is achieved by not removing the message from the buffer until the subscriber sends an explicit acknowledgement.

```java
public class Message {
    private final String messageId;
    private final String topic;
    private final Object payload;
    private final Map<String, String> headers;
    private final Instant publishedAt = Instant.now();

    public Message(String topic, Object payload, Map<String, String> headers) {
        messageId = UUID.randomUUID().toString();
        this.topic = topic; this.payload = payload; this.headers = Collections.unmodifiableMap(new HashMap<>(headers));
    }
    public String getMessageId() { return messageId; }
    public Object getPayload() { return payload; }
    public Map<String, String> getHeaders() { return headers; }
}

public class Subscription {
    private final String subscriptionId;
    private final MessageSubscriber subscriber;
    private final Predicate<Message> filter;
    private final BlockingQueue<Message> buffer = new LinkedBlockingQueue<>(1000);

    public Subscription(String id, MessageSubscriber sub, Predicate<Message> filter) {
        subscriptionId = id; subscriber = sub; this.filter = filter;
    }

    public void deliver(Message message) {
        if (filter == null || filter.test(message)) {
            if (!buffer.offer(message)) { System.err.println("Buffer full for subscription " + subscriptionId); }
        }
    }

    public void startDispatching() {
        Thread dispatcher = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    Message msg = buffer.take();
                    try { subscriber.onMessage(msg); }
                    catch (Exception e) { System.err.println("Delivery failed: " + e.getMessage()); }
                } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            }
        });
        dispatcher.setDaemon(true);
        dispatcher.start();
    }
}

public class PubSubBroker {
    private final ConcurrentHashMap<String, List<Subscription>> topicSubscriptions = new ConcurrentHashMap<>();

    public void publish(String topicName, Object payload, Map<String, String> headers) {
        Message msg = new Message(topicName, payload, headers);
        List<Subscription> subs = topicSubscriptions.getOrDefault(topicName, Collections.emptyList());
        subs.forEach(s -> s.deliver(msg));
    }

    public Subscription subscribe(String topic, MessageSubscriber sub, Predicate<Message> filter) {
        Subscription subscription = new Subscription(UUID.randomUUID().toString(), sub, filter);
        topicSubscriptions.computeIfAbsent(topic, k -> new CopyOnWriteArrayList<>()).add(subscription);
        subscription.startDispatching();
        return subscription;
    }
}
```

---

### Q75. Design a Workflow Engine

**Problem Statement:** Design a workflow engine that executes a sequence of steps with conditional branching, parallel execution, error handling, and step retries.

**What to Discuss:**
1. Workflow definition: steps, transitions, conditions, parallel splits/joins
2. Workflow instance vs workflow definition: definition is the template, instance is the running execution
3. Step types: action step, decision step, parallel split, parallel join, wait step
4. Error handling: retry policy per step, compensation (rollback) actions
5. Persistence: checkpointing workflow state for durability

**Key Classes/Interfaces:** `WorkflowDefinition`, `WorkflowInstance`, `Step`, `ActionStep`, `DecisionStep`, `ParallelStep`, `WorkflowContext`, `WorkflowEngine`, `RetryPolicy`

**Key Design Patterns:** Command (Step as command), Strategy (retry policy), Composite (ParallelStep contains multiple steps), Chain of Responsibility (step execution chain), State (workflow instance state)

**Model Answer:** A `WorkflowDefinition` is an immutable graph of `Step` nodes connected by `Transition` objects (with optional `Condition` predicates for branching). A `WorkflowInstance` holds a reference to the definition, the current step pointer, a `WorkflowContext` (shared data map), and the execution status. The `WorkflowEngine` executes the current step, evaluates outgoing transitions to find the next step, and persists the instance state after each step completes — this provides the checkpointing needed to resume after a crash. `ParallelStep` uses a `CountDownLatch` or `CompletableFuture.allOf()` to wait for all parallel branches before proceeding.

```java
public interface Step {
    String getId();
    StepResult execute(WorkflowContext context);
    List<Transition> getOutgoingTransitions();
}

public class StepResult {
    private final boolean success;
    private final String outcomeLabel; // used to select transition
    private final Exception error;

    private StepResult(boolean success, String outcome, Exception error) {
        this.success = success; outcomeLabel = outcome; this.error = error;
    }

    public static StepResult success(String outcome) { return new StepResult(true, outcome, null); }
    public static StepResult failure(Exception e) { return new StepResult(false, null, e); }
    public boolean isSuccess() { return success; }
    public String getOutcomeLabel() { return outcomeLabel; }
}

public class Transition {
    private final String targetStepId;
    private final Predicate<WorkflowContext> condition;
    private final String outcomeLabel;

    public Transition(String targetStepId, Predicate<WorkflowContext> condition, String outcomeLabel) {
        this.targetStepId = targetStepId; this.condition = condition; this.outcomeLabel = outcomeLabel;
    }

    public boolean evaluate(WorkflowContext ctx) { return condition == null || condition.test(ctx); }
    public String getTargetStepId() { return targetStepId; }
    public String getOutcomeLabel() { return outcomeLabel; }
}

public class WorkflowContext {
    private final Map<String, Object> data = new ConcurrentHashMap<>();
    public void set(String key, Object value) { data.put(key, value); }
    public Object get(String key) { return data.get(key); }
    @SuppressWarnings("unchecked")
    public <T> T get(String key, Class<T> type) { return type.cast(data.get(key)); }
}

public class WorkflowEngine {
    private final Map<String, WorkflowDefinition> definitions = new HashMap<>();

    public WorkflowInstance start(String definitionId, Map<String, Object> initialData) {
        WorkflowDefinition def = definitions.get(definitionId);
        WorkflowContext ctx = new WorkflowContext();
        initialData.forEach(ctx::set);
        WorkflowInstance instance = new WorkflowInstance(UUID.randomUUID().toString(), def, ctx);
        execute(instance);
        return instance;
    }

    private void execute(WorkflowInstance instance) {
        while (!instance.isFinished()) {
            Step current = instance.getCurrentStep();
            StepResult result = executeWithRetry(current, instance.getContext(), current.getRetryPolicy());
            if (!result.isSuccess()) { instance.markFailed(result.getError()); return; }
            Step next = selectNextStep(current, result, instance.getContext(), instance.getDefinition());
            if (next == null) { instance.markCompleted(); return; }
            instance.advanceTo(next);
        }
    }

    private StepResult executeWithRetry(Step step, WorkflowContext ctx, RetryPolicy policy) {
        int attempts = 0;
        while (true) {
            StepResult result = step.execute(ctx);
            if (result.isSuccess() || !policy.shouldRetry(attempts++)) return result;
            try { Thread.sleep(policy.delayMs(attempts)); } catch (InterruptedException e) { Thread.currentThread().interrupt(); break; }
        }
        return StepResult.failure(new RuntimeException("Max retries exceeded for step: " + step.getId()));
    }

    private Step selectNextStep(Step current, StepResult result, WorkflowContext ctx, WorkflowDefinition def) {
        return current.getOutgoingTransitions().stream()
            .filter(t -> t.getOutcomeLabel().equals(result.getOutcomeLabel()) && t.evaluate(ctx))
            .findFirst()
            .map(t -> def.getStep(t.getTargetStepId()))
            .orElse(null);
    }
}
```

---
## Category 5: Code Design and Best Practices

### Q76. What are coupling and cohesion? How do you measure and improve them?

**Model Answer:** Cohesion is the degree to which the elements inside a module belong together — high cohesion means every method in a class contributes to a single, well-defined purpose. Coupling is the degree of interdependence between modules — low coupling means a module can be changed, replaced, or tested without requiring changes to other modules. You improve cohesion by applying SRP (split classes with multiple responsibilities) and improve coupling by introducing abstractions (interfaces), using dependency injection, and avoiding direct instantiation of collaborators. A practical heuristic: if you cannot describe what a class does in one sentence without using "and," it has low cohesion.

```java
// Low cohesion + high coupling: one class does everything, references concrete types
public class OrderProcessor {
    private MySQLDatabase db = new MySQLDatabase();       // high coupling
    private EmailSender email = new EmailSender();        // high coupling
    private PdfGenerator pdf = new PdfGenerator();        // high coupling
    // Three unrelated concerns in one class — low cohesion
    public void processOrder(Order o) { db.save(o); email.send(o); pdf.generate(o); }
}

// High cohesion + low coupling: focused class, depends on abstractions
public class OrderService {
    private final OrderRepository repository; // depends on interface — low coupling
    public OrderService(OrderRepository repo) { this.repository = repo; }
    public void placeOrder(Order order) { // single responsibility — high cohesion
        order.validate();
        repository.save(order);
    }
}
```

---

### Q77. When is NOT following the DRY principle actually the right call?

**Model Answer:** DRY (Don't Repeat Yourself) is about eliminating *accidental* duplication — two code paths that represent the same knowledge and must always change together. But when two code fragments look similar yet represent *different domain concepts that happen to have the same shape today*, forced DRY creates wrong abstraction: an artificial coupling that makes both harder to change independently when requirements diverge. The wrong abstraction is more expensive than duplication; premature extraction locks in an abstraction before the variation pattern is understood. A concrete signal that duplication is correct: two tests that both set up a 3-field object differently would be made *harder* to read by extracting a shared helper.

```java
// WRONG DRY: extract because they "look the same" — but they represent different domain rules
public class OrderValidator {
    // Order discount eligibility
    public boolean isEligible(Customer c) { return c.getAge() >= 18 && c.isVerified(); }
}

public class CreditService {
    // Credit approval eligibility — same shape but DIFFERENT rule meaning
    // DO NOT reuse isEligible — these rules will diverge independently
    public boolean isApproved(Customer c) { return c.getAge() >= 18 && c.isVerified(); }
}

// RIGHT DRY: extract because these represent the SAME knowledge
public class Money {
    // This formatting rule is the same knowledge in 3 places — extract it
    private static final NumberFormat FORMATTER = NumberFormat.getCurrencyInstance();
    public String display() { return FORMATTER.format(amount); }
    // Previously: 3 classes each had: new DecimalFormat("$#,##0.00").format(amount)
}
```

---

### Q78. How does the KISS principle apply to Java codebases, and how does over-engineering manifest?

**Model Answer:** KISS (Keep It Simple, Stupid) advocates solving the actual problem with the simplest design that works — no more complexity than necessary. In Java codebases, over-engineering manifests as: introducing design patterns where a simple method would suffice (a 3-class Strategy hierarchy for a two-branch if/else), generic abstractions with type parameters that make the code unreadable for one-off use cases, deep class hierarchies for a domain with two variants, and XML/annotation-based configuration for logic that is trivially expressed in code. The test for KISS: can a new team member understand this code in 5 minutes? If not, is the complexity *earning its keep* by solving a genuine variability problem?

```java
// Over-engineered: Strategy for a 2-branch check that will never change
public interface GreetingStrategy { String greet(String name); }
public class FormalGreeting implements GreetingStrategy { public String greet(String n) { return "Good day, " + n; } }
public class CasualGreeting implements GreetingStrategy { public String greet(String n) { return "Hey, " + n; } }
public class Greeter {
    private final GreetingStrategy strategy;
    // ...
}
// Just use a method:
public class Greeter {
    public String greet(String name, boolean formal) {
        return formal ? "Good day, " + name : "Hey, " + name;
    }
}

// Over-engineered: generic pipeline for a single use case
public class Pipeline<I, O> {
    private final List<Stage<?, ?>> stages = new ArrayList<>();
    public <T> Pipeline<I, T> addStage(Stage<O, T> stage) { ... }
    public O execute(I input) { ... }
}
// Just write three method calls in a service class — you can extract the pipeline later if a fourth use case emerges
```

---

### Q79. What is YAGNI and how do you apply it during design reviews?

**Model Answer:** YAGNI (You Aren't Gonna Need It) is the principle that you should not add functionality or abstraction until it is actually needed — not "might be needed someday." In design reviews, YAGNI challenges speculative generality: extension points, plugin architectures, strategy hierarchies, and configurable frameworks built for hypothetical future requirements that are not on the roadmap. The cost of over-abstraction is borne immediately (complexity, maintenance, testing) while the benefit is deferred (and may never materialize). The counter to "we might need this later" is: adding it later is cheap when you understand the actual requirement; adding it now is expensive because you're guessing the shape of a future requirement.

```java
// YAGNI violation: plugin architecture for a feature that has one known implementation
public interface ReportPlugin { void process(ReportData data); }
public class PluginRegistry { private List<ReportPlugin> plugins = new ArrayList<>(); ... }
public class ReportService { private final PluginRegistry registry; ... }
// Currently there is exactly ONE report type. This "extension point" adds 3 classes and zero value.

// YAGNI-compliant: just solve the problem you have
public class ReportService {
    public Report generate(ReportRequest request) {
        // direct implementation — if a second report type appears, refactor then
        return new PdfReport(request.getData());
    }
}
// When a second report type is actually required, introduce the Strategy — the real use case
// will reveal what the abstraction boundary should actually be.
```

---

### Q80. What is the Law of Demeter and when is it pragmatic to violate it?

**Model Answer:** The Law of Demeter (LoD) states that a method should only call methods on: itself, its direct fields, parameters passed to it, and objects it creates locally — never on objects returned by collaborators ("one dot" rule). It reduces coupling by preventing deep dependency chains: `customer.getAddress().getCity().getZipCode()` couples the caller to three intermediate types. Pragmatic violations are acceptable when the intermediate objects are simple value types with no behavior (data transfer objects, Java records), where the navigation is purely for data extraction and the intermediate types are stable. Applying LoD dogmatically to fluent builders and Java streams would make them impossible to use.

```java
// LoD violation: caller couples to Order -> Customer -> Address -> City
public void sendShippingLabel(Order order) {
    String zip = order.getCustomer().getAddress().getCity().getZipCode(); // 3 intermediate types
    labelPrinter.print(zip);
}

// LoD-compliant: Order provides what the caller needs
public class Order {
    public String getShippingZipCode() {
        return customer.getAddress().getCity().getZipCode(); // Order knows its own graph
    }
}
public void sendShippingLabel(Order order) {
    labelPrinter.print(order.getShippingZipCode()); // caller only talks to Order
}

// Pragmatic violation: Java Stream chain — acceptable because these are standard value transforms
List<String> names = users.stream()
    .filter(User::isActive)
    .map(User::getName)
    .sorted()
    .collect(Collectors.toList()); // "method chain" but all operations are on the stream itself
```

---

### Q81. What is the "Tell, Don't Ask" principle and how does it differ from the Law of Demeter?

**Model Answer:** "Tell, Don't Ask" says you should tell objects what to do rather than querying their state and making decisions for them externally — keep behavior with the data it operates on. Law of Demeter says don't reach through chains of objects to get to what you need. They are related but distinct: LoD is about *navigation depth*, Tell Don't Ask is about *where decisions live*. A violation of Tell Don't Ask looks like: `if (account.getBalance() > 0) { account.debit(amount); }` — the caller is querying internal state and making a decision that belongs inside `Account`. The fix: `account.debit(amount)` — `Account.debit()` checks the balance and throws if insufficient.

```java
// Violates Tell Don't Ask: caller inspects state and makes the decision
public void processOrder(Order order) {
    if (order.getStatus() == OrderStatus.PENDING) {    // asking
        if (order.getItems().size() > 0) {             // asking
            order.setStatus(OrderStatus.CONFIRMED);     // telling too late
        }
    }
}

// Tell Don't Ask: push decision into Order
public class Order {
    public void confirm() {
        if (status != OrderStatus.PENDING) throw new IllegalStateException("Only pending orders can be confirmed");
        if (items.isEmpty()) throw new IllegalStateException("Cannot confirm empty order");
        this.status = OrderStatus.CONFIRMED;
    }
}

public void processOrder(Order order) {
    order.confirm(); // tell — Order decides if it's valid
}
```

---

### Q82. What is Feature Envy and how do you refactor it using Move Method?

**Model Answer:** Feature Envy is a code smell where a method is more interested in the data of another class than in its own — it accesses multiple fields or methods of a foreign class extensively, suggesting the method belongs in that class. The refactoring is Move Method: move the envious method to the class it is most attracted to, passing `this` as a parameter if the method still needs some access back to the original class. Feature Envy often co-occurs with Anemic Domain Model — business logic scattered in service classes that obsessively access domain object fields.

```java
// Feature Envy: OrderPrinter is obsessed with Order's fields
public class OrderPrinter {
    public String formatOrder(Order order) {
        StringBuilder sb = new StringBuilder();
        sb.append("Order ID: ").append(order.getOrderId()).append("\n");
        sb.append("Customer: ").append(order.getCustomer().getName()).append("\n");
        for (LineItem item : order.getLineItems()) {
            sb.append(item.getProduct().getName()).append(" x").append(item.getQuantity())
              .append(" = $").append(item.getSubtotal()).append("\n");
        }
        sb.append("Total: $").append(order.getTotal());
        return sb.toString();
    }
}

// After Move Method: formatting logic belongs in Order (or a nested formatter)
public class Order {
    public String format() {
        StringBuilder sb = new StringBuilder();
        sb.append("Order ID: ").append(orderId).append("\n");
        sb.append("Customer: ").append(customer.getName()).append("\n");
        lineItems.forEach(item ->
            sb.append(item.getProduct().getName()).append(" x").append(item.getQuantity())
              .append(" = $").append(item.getSubtotal()).append("\n"));
        sb.append("Total: $").append(getTotal());
        return sb.toString();
    }
}
```

---

### Q83. What is a God Class, why does it form, and how do you break it apart?

**Model Answer:** A God Class is a single class that knows too much and does too much — it centralizes most of the system's logic and data, violating SRP severely. It forms through "convenience accretion": developers keep adding related logic to one large class because it already knows about everything, the class grows incrementally, and no single addition seems unjustifiable in isolation. Breaking it apart requires identifying "clusters of cohesion" within the class — groups of methods and fields that always change together and serve one purpose — and extracting each cluster into its own class, delegating from the original class initially to avoid breaking callers.

```java
// God Class symptom: UserManager does authentication, profile, billing, reporting, notifications
public class UserManager {
    // 50+ methods, 20+ fields spanning 5 different domains
    public void authenticate(String u, String p) { ... }
    public void updateProfile(User u, ProfileData d) { ... }
    public void chargeCard(User u, double amount) { ... }
    public void generateMonthlyReport(User u) { ... }
    public void sendWelcomeEmail(User u) { ... }
    // ...
}

// After decomposition: extract cohesive clusters
public class AuthenticationService { public AuthResult authenticate(String u, String p) { ... } }
public class UserProfileService { public void updateProfile(User u, ProfileData d) { ... } }
public class BillingService { public void chargeCard(User u, double amount) { ... } }
public class UserReportService { public Report generateMonthlyReport(User u) { ... } }
public class UserNotificationService { public void sendWelcomeEmail(User u) { ... } }

// UserManager can temporarily delegate during migration
public class UserManager {
    private final AuthenticationService auth;
    private final UserProfileService profile;
    // delegate all calls — callers don't need to change immediately
    public AuthResult authenticate(String u, String p) { return auth.authenticate(u, p); }
}
```

---

### Q84. What is the Anemic Domain Model and why is it an anti-pattern in DDD contexts?

**Model Answer:** An Anemic Domain Model is one where domain objects are mere data containers (all getters/setters, no behavior) and all business logic lives in service classes that manipulate those objects procedurally. It is an anti-pattern in Domain-Driven Design because it splits the *what* (data in the model) from the *how* (logic in the service), making it impossible to enforce invariants at the object level, difficult to reason about which service owns which rule, and prone to duplication of business logic across multiple services. The fix is to move domain behavior back into domain objects — balance validation in `BankAccount`, state transition logic in `Order`.

```java
// Anemic Domain Model: Order is a data bag, all logic in service
public class Order {
    private String orderId;
    private OrderStatus status;
    private List<LineItem> lineItems;
    // only getters and setters
}

public class OrderService {
    public void confirmOrder(Order order) {
        if (order.getStatus() != OrderStatus.PENDING) throw new IllegalStateException("...");
        if (order.getLineItems().isEmpty()) throw new IllegalStateException("...");
        order.setStatus(OrderStatus.CONFIRMED); // service manually sets fields
    }
}

// Rich Domain Model: Order owns its own behavior and invariants
public class Order {
    private OrderStatus status = OrderStatus.PENDING;
    private final List<LineItem> lineItems = new ArrayList<>();

    public void addItem(LineItem item) {
        if (status != OrderStatus.PENDING) throw new IllegalStateException("Cannot modify confirmed order");
        lineItems.add(item);
    }

    public void confirm() {
        if (lineItems.isEmpty()) throw new IllegalStateException("Cannot confirm empty order");
        if (status != OrderStatus.PENDING) throw new IllegalStateException("Order already confirmed");
        status = OrderStatus.CONFIRMED; // order enforces its own state transition
    }
}
```

---

### Q85. What is Primitive Obsession and how do you replace primitives with domain types in Java?

**Model Answer:** Primitive Obsession is using basic types (`String`, `int`, `double`) to represent domain concepts that have their own rules, validation, and behavior. A `String` `email` cannot enforce email format; a `double` `price` cannot ensure it is non-negative or carry currency information; an `int` `userId` can be accidentally passed where an `orderId` is expected. The fix is to introduce Value Objects (domain types): small, immutable classes that wrap the primitive, validate it, and provide domain-relevant behavior. Java records (Java 16+) make this almost zero boilerplate.

```java
// Primitive Obsession: String for email, double for money, raw int for IDs
public void createOrder(int customerId, String email, double price, String currency) { ... }
// customerId and orderId are both int — easily swapped by accident

// Value Objects replace primitives
public record CustomerId(long value) {
    public CustomerId { if (value <= 0) throw new IllegalArgumentException("CustomerId must be positive"); }
}

public record Email(String value) {
    private static final Pattern EMAIL_REGEX = Pattern.compile("^[^@]+@[^@]+\\.[^@]+$");
    public Email {
        Objects.requireNonNull(value);
        if (!EMAIL_REGEX.matcher(value).matches()) throw new IllegalArgumentException("Invalid email: " + value);
    }
}

public record Money(BigDecimal amount, Currency currency) {
    public Money {
        Objects.requireNonNull(amount); Objects.requireNonNull(currency);
        if (amount.compareTo(BigDecimal.ZERO) < 0) throw new IllegalArgumentException("Amount cannot be negative");
    }
    public Money add(Money other) {
        if (!currency.equals(other.currency)) throw new IllegalArgumentException("Currency mismatch");
        return new Money(amount.add(other.amount), currency);
    }
    public boolean isGreaterThan(Money other) { return amount.compareTo(other.amount) > 0; }
}

// Usage: type-safe, self-validating, expressive
public void createOrder(CustomerId customerId, Email contactEmail, Money price) { ... }
```

---

### Q86. What is Shotgun Surgery and how do you refactor to reduce it?

**Model Answer:** Shotgun Surgery is a code smell where a single logical change requires making many small modifications to many different classes — the inverse of divergent change. It signals that a responsibility is scattered across the codebase rather than concentrated in one place, making changes risky (easy to miss one site) and expensive. The refactoring approach is to inline and consolidate: identify all the scattered change sites, extract the common concept into one class, and have all the scattered locations delegate to it.

```java
// Shotgun Surgery: changing tax calculation requires updating 5 different classes
public class OrderService {
    double tax = order.getTotal() * 0.08; // tax logic duplicated here
}
public class InvoiceService {
    double tax = amount * 0.08;            // and here
}
public class CartService {
    double tax = subtotal * 0.08;          // and here
}
// Changing tax rate or logic requires finding all 3+ locations

// After consolidation: single point of change
public class TaxCalculator {
    private static final BigDecimal TAX_RATE = new BigDecimal("0.08");

    public Money calculate(Money subtotal) {
        return new Money(subtotal.amount().multiply(TAX_RATE), subtotal.currency());
    }
}

// All callers use TaxCalculator — changing tax logic is a one-class change
public class OrderService {
    private final TaxCalculator taxCalculator;
    public Money computeTax(Order order) { return taxCalculator.calculate(order.getSubtotal()); }
}
```

---

### Q87. What is Divergent Change, how does it contrast with Shotgun Surgery, and what does it signal architecturally?

**Model Answer:** Divergent Change is when a single class changes for many different reasons — multiple unrelated stimuli cause modifications to the same class. It is the direct symptom of SRP violation. Shotgun Surgery is the complement: one reason to change forces modifications across many classes. Divergent Change signals that a class has too many responsibilities and should be split; Shotgun Surgery signals that a responsibility is too scattered and should be consolidated. Architecturally, divergent change in a class indicates it sits at the intersection of multiple bounded contexts or subsystems — a refactoring to separate those contexts will resolve it.

```java
// Divergent Change: ReportGenerator changes for new report formats, new data sources,
// and new output channels — three different axes
public class ReportGenerator {
    public void generate(ReportType type, DataSource source, OutputChannel channel) {
        // Changed by: data team (source), report designers (format), ops team (channel)
        // Three unrelated reasons to change — three bounded contexts tangled
    }
}

// After decomposition: each class changes for one reason
public interface DataSourceAdapter { ReportData fetch(ReportRequest req); }
public interface ReportFormatter { RenderedReport format(ReportData data, ReportType type); }
public interface ReportDelivery { void deliver(RenderedReport report, OutputChannel channel); }

public class ReportService {
    // Orchestrates, but each piece changes independently
    public void generate(ReportRequest req) {
        ReportData data = dataSource.fetch(req);
        RenderedReport rendered = formatter.format(data, req.getType());
        delivery.deliver(rendered, req.getOutputChannel());
    }
}
```

---

### Q88. What is Command-Query Separation (CQS) and when is it appropriate to break it?

**Model Answer:** CQS, introduced by Bertrand Meyer, states that every method should be either a command (changes state, returns void) or a query (returns a result, has no side effects) — never both. This makes code easier to reason about because you know that reading a value never changes anything. It is appropriate to break CQS when the state change and the returned value are inherently atomic and inseparable: `stack.pop()` both removes an element and returns it; `queue.poll()` both removes and returns in one atomic operation; `AtomicInteger.getAndIncrement()` is both a read and a write. Breaking CQS is also acceptable for performance: re-querying a value that was computed during a mutation wastes work.

```java
// CQS-compliant: separate command and query
public class UserService {
    // Command: changes state, returns void
    public void activateUser(UserId userId) {
        User user = repository.findById(userId).orElseThrow();
        user.activate();
        repository.save(user);
    }

    // Query: no side effects, returns result
    public boolean isUserActive(UserId userId) {
        return repository.findById(userId).map(User::isActive).orElse(false);
    }
}

// CQS violation that IS justified: pop is inherently atomic
public class Stack<T> {
    private final Deque<T> data = new ArrayDeque<>();

    // This IS both command and query — separation would require two calls and a race condition
    public T pop() {
        if (data.isEmpty()) throw new EmptyStackException();
        return data.pop(); // justified CQS violation: atomicity requirement
    }
}
```

---

### Q89. How do you implement defensive programming in Java, including preconditions, assertions, and fail-fast?

**Model Answer:** Defensive programming means writing code that detects and fails on incorrect assumptions immediately, rather than silently propagating bad state. Preconditions validate method arguments at entry and throw `IllegalArgumentException` or `NullPointerException` if violated (use `Objects.requireNonNull`, `Preconditions.checkArgument` from Guava). Assertions (`assert`) check internal invariants that should never fail in correct code and are enabled with JVM flag `-ea`; they are not for production input validation. Fail-fast means the program crashes loudly at the site of the problem rather than continuing with corrupt state and failing with an obscure error far from the root cause.

```java
import java.util.Objects;

public class TransferService {
    public void transfer(BankAccount from, BankAccount to, Money amount) {
        // Preconditions: validate public API contract
        Objects.requireNonNull(from, "source account must not be null");
        Objects.requireNonNull(to, "destination account must not be null");
        Objects.requireNonNull(amount, "amount must not be null");
        if (amount.isZeroOrNegative()) throw new IllegalArgumentException("Transfer amount must be positive: " + amount);
        if (from.getId().equals(to.getId())) throw new IllegalArgumentException("Cannot transfer to the same account");

        Money balance = from.getBalance();
        if (balance.isLessThan(amount)) throw new InsufficientFundsException(from.getId(), amount, balance);

        from.debit(amount);
        to.credit(amount);

        // Internal assertion: invariant that should NEVER fail if logic above is correct
        assert from.getBalance().isNonNegative() : "Balance went negative after debit — logic error: " + from.getBalance();
        assert to.getBalance().isGreaterThanOrEqual(amount) : "Credit did not take effect — logic error";
    }
}

// Fail-fast in domain object
public class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        this.amount = Objects.requireNonNull(amount);
        this.currency = Objects.requireNonNull(currency);
        // Fail-fast: reject negative money at construction, not at first use
        if (amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Money amount cannot be negative: " + amount);
    }
}
```

---

### Q90. What is Design by Contract and how do you apply it in Java using documentation and annotations?

**Model Answer:** Design by Contract (DbC), from Eiffel, formalizes the relationship between a method and its callers via three clauses: preconditions (what the caller must guarantee before calling), postconditions (what the method guarantees upon return), and invariants (what properties are always true of the object's state). Java has no native DbC support, but you approximate it with: Javadoc `@throws` tags documenting precondition failures, `@param` documenting valid input ranges, explicit guard clauses that throw on violation, assertions for postconditions and invariants, and frameworks like Bean Validation annotations (`@NotNull`, `@Min`, `@Max`) for declarative constraint documentation.

```java
/**
 * Processes a payment for the given order.
 *
 * <p><b>Preconditions:</b>
 * <ul>
 *   <li>{@code order} must not be null</li>
 *   <li>{@code order.getStatus()} must be {@code CONFIRMED}</li>
 *   <li>{@code order.getTotal()} must be greater than zero</li>
 * </ul>
 *
 * <p><b>Postconditions:</b>
 * <ul>
 *   <li>Returns a non-null {@link PaymentResult} with {@code success == true} on success</li>
 *   <li>{@code order.getStatus()} will be {@code PAID} on success</li>
 * </ul>
 *
 * @throws IllegalArgumentException if order is null or not in CONFIRMED status
 * @throws InsufficientFundsException if the payment method has insufficient balance
 */
public PaymentResult processPayment(Order order) {
    // Precondition checks
    Objects.requireNonNull(order, "order must not be null");
    if (order.getStatus() != OrderStatus.CONFIRMED)
        throw new IllegalArgumentException("Order must be in CONFIRMED status, was: " + order.getStatus());
    if (!order.getTotal().isPositive())
        throw new IllegalArgumentException("Order total must be positive: " + order.getTotal());

    PaymentResult result = paymentGateway.charge(order.getCustomer().getPaymentMethod(), order.getTotal());

    if (result.isSuccess()) {
        order.markPaid(result.getTransactionId());
        // Postcondition assertion
        assert order.getStatus() == OrderStatus.PAID : "Postcondition violated: order not marked PAID after success";
    }

    return result;
}

// Bean Validation for declarative contract on DTOs
public class CreateOrderRequest {
    @NotNull(message = "customerId is required")
    private CustomerId customerId;

    @NotEmpty(message = "order must have at least one item")
    private List<@Valid OrderItemRequest> items;

    @NotNull @Positive(message = "total must be positive")
    private BigDecimal total;
}
```

---

## Category 6: Testing and Quality

### Q91. Describe the TDD Red-Green-Refactor cycle and how it influences class design.

**Model Answer:** The TDD cycle is: Red — write a failing test that describes the desired behavior (the test must fail for the right reason — not compile error, but assertion failure); Green — write the minimum production code to make the test pass, no more; Refactor — clean up the code (eliminate duplication, improve naming, apply patterns) while keeping all tests green. TDD influences design by forcing you to think about the public API before the implementation: if the test setup is verbose or requires complex mocking, that is immediate feedback that the class is hard to instantiate, has too many dependencies, or is doing too much — TDD makes design pain visible immediately.

```java
// RED: write the failing test first
@Test
void should_calculate_discounted_price_for_premium_customer() {
    Customer customer = new Customer("Alice", CustomerType.PREMIUM);
    Product product = new Product("Widget", new Money(new BigDecimal("100.00"), Currency.USD));
    PricingService pricing = new PricingService();

    Money discountedPrice = pricing.calculatePrice(product, customer);

    assertEquals(new Money(new BigDecimal("85.00"), Currency.USD), discountedPrice);
    // Test FAILS: PricingService doesn't exist yet — that's the "Red"
}

// GREEN: minimum code to pass
public class PricingService {
    public Money calculatePrice(Product product, Customer customer) {
        if (customer.getType() == CustomerType.PREMIUM) {
            return product.getPrice().multiply(new BigDecimal("0.85"));
        }
        return product.getPrice();
    }
}

// REFACTOR: extract discount lookup to be extensible (OCP)
public class PricingService {
    private final Map<CustomerType, BigDecimal> discounts = Map.of(
        CustomerType.REGULAR, BigDecimal.ONE,
        CustomerType.PREMIUM, new BigDecimal("0.85"),
        CustomerType.VIP, new BigDecimal("0.75")
    );

    public Money calculatePrice(Product product, Customer customer) {
        BigDecimal multiplier = discounts.getOrDefault(customer.getType(), BigDecimal.ONE);
        return product.getPrice().multiply(multiplier);
    }
}
```

---

### Q92. What are the differences between unit tests and integration tests, and when should you use each?

**Model Answer:** Unit tests verify a single class or function in isolation, with all external dependencies replaced by test doubles — they run in milliseconds, have no network or database I/O, and give precise failure diagnosis. Integration tests verify that multiple components work correctly together, exercising real infrastructure (database, message broker, external API) — they are slower, more expensive to set up, but catch problems that unit tests cannot: ORM mapping bugs, SQL constraint violations, serialization issues. The recommended split for most systems is 70% unit, 20% integration, 10% end-to-end — integration tests are not a substitute for unit tests because they cannot exhaust all branch combinations efficiently.

```java
// Unit test: isolated, fast, precise
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository orderRepository;
    @Mock EmailService emailService;
    @InjectMocks OrderService orderService;

    @Test
    void should_save_order_and_send_confirmation() {
        Order order = new Order(new CustomerId(1L), List.of(new LineItem("SKU-1", 2)));
        when(orderRepository.save(any())).thenReturn(order);

        orderService.placeOrder(order);

        verify(orderRepository).save(order);       // unit: verifies exact interaction
        verify(emailService).sendConfirmation(order);
    }
}

// Integration test: real repository, real database
@SpringBootTest
@Transactional
class OrderRepositoryIntegrationTest {
    @Autowired OrderRepository orderRepository;

    @Test
    void should_persist_and_retrieve_order() {
        Order order = new Order(new CustomerId(1L), List.of(new LineItem("SKU-1", 2)));
        Order saved = orderRepository.save(order);

        Optional<Order> found = orderRepository.findById(saved.getId());

        assertTrue(found.isPresent());
        assertEquals(1, found.get().getLineItems().size()); // verifies real JPA mapping
    }
}
```

---

### Q93. What are the different types of test doubles in Java (mock, stub, fake, spy) and when do you use each?

**Model Answer:** A stub returns hard-coded values for specific calls without verifying behavior — use it when a test needs controlled outputs from a dependency but does not care whether or how that dependency was called. A mock verifies interactions — it records which methods were called, with what arguments, and how many times; use it to verify that a side-effectful operation (send email, write to DB) was invoked correctly. A fake is a real, lightweight implementation of the dependency (e.g., `InMemoryOrderRepository`) that works for tests but is not production-grade; use it when the behavior logic of the dependency matters. A spy wraps a real object and allows selective stubbing and interaction verification; use it when you want mostly real behavior with a few overrides.

```java
// Stub: returns controlled values, no interaction verification
@Test
void stub_example() {
    OrderRepository stub = Mockito.mock(OrderRepository.class);
    when(stub.findById(any())).thenReturn(Optional.of(new Order("O-1")));
    // only cares about the return value, not how many times it was called
}

// Mock: verifies interaction
@Test
void mock_example() {
    EmailService mockEmail = Mockito.mock(EmailService.class);
    OrderService service = new OrderService(repository, mockEmail);

    service.placeOrder(order);

    verify(mockEmail, times(1)).sendConfirmation(order); // verifies side effect occurred
    verify(mockEmail, never()).sendCancellation(any());
}

// Fake: real behavior, in-memory
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<OrderId, Order> store = new HashMap<>();
    @Override public void save(Order o) { store.put(o.getId(), o); }
    @Override public Optional<Order> findById(OrderId id) { return Optional.ofNullable(store.get(id)); }
}

// Spy: real object with selective override
@Test
void spy_example() {
    EmailService realEmail = new EmailService(smtpConfig);
    EmailService spy = Mockito.spy(realEmail);
    doNothing().when(spy).sendConfirmation(any()); // override only this method

    service.placeOrder(order);

    verify(spy).sendConfirmation(order); // verify it was called
    // all other EmailService methods still use real implementation
}
```

---

### Q94. Why should you generally avoid testing private methods, and what does the urge to do so signal about design?

**Model Answer:** Private methods are implementation details — testing them directly ties tests to the class's internal structure, meaning a refactoring that leaves external behavior unchanged (e.g., splitting one private method into two) would break the tests unnecessarily. Private methods are exercised indirectly through the public API they support; if they are correct, the public method is correct. The urge to test a private method directly is a design signal: if the method is complex enough to need its own test, it is probably doing enough work to deserve its own class with a public interface. Extract it, make it public, and test it as a first-class unit.

```java
// Design smell: complex private method begging for its own class
public class InvoiceService {
    public Invoice generate(Order order) {
        List<LineItem> taxedItems = applyTaxRules(order.getItems()); // complex private
        return new Invoice(order, taxedItems);
    }

    private List<LineItem> applyTaxRules(List<LineItem> items) {
        // 40 lines of complex tax logic — clearly deserves its own class
    }
}

// After extraction: TaxCalculator is public, testable, reusable
public class TaxCalculator {
    public List<LineItem> applyRules(List<LineItem> items, TaxJurisdiction jurisdiction) {
        // formerly private 40-line method — now independently testable
    }
}

@Test
void tax_calculator_applies_vat_to_non_exempt_items() {
    TaxCalculator calculator = new TaxCalculator();
    List<LineItem> items = List.of(new LineItem("FOOD", BigDecimal.TEN, false)); // non-exempt

    List<LineItem> taxed = calculator.applyRules(items, TaxJurisdiction.EU_VAT);

    assertEquals(new BigDecimal("12.00"), taxed.get(0).getTotal()); // 20% VAT applied
}
```

---

### Q95. How do you effectively test code that uses the Strategy, Observer, or Decorator pattern?

**Model Answer:** For Strategy: test each strategy implementation in isolation, then test the context with a simple stub strategy to verify the context calls the strategy correctly. For Observer: test the subject verifies it notifies observers on state changes (inject a mock observer and verify `onEvent` is called); test each observer's `onEvent` handler independently with a variety of events. For Decorator: test each decorator in isolation by constructing it with a mock delegate and verifying it adds the intended behavior while delegating correctly; test that the behavior composes correctly when decorators are layered.

```java
// Testing Strategy: each implementation independently
@Test
void premium_discount_applies_15_percent() {
    DiscountStrategy strategy = new PremiumDiscount();
    assertEquals(85.0, strategy.apply(100.0), 0.001);
}

// Testing context with stub strategy
@Test
void discount_calculator_delegates_to_strategy() {
    DiscountStrategy mockStrategy = Mockito.mock(DiscountStrategy.class);
    when(mockStrategy.apply(100.0)).thenReturn(90.0);
    DiscountCalculator calc = new DiscountCalculator(Map.of(CustomerType.REGULAR, mockStrategy));

    double result = calc.calculate(CustomerType.REGULAR, 100.0);

    assertEquals(90.0, result);
    verify(mockStrategy).apply(100.0);
}

// Testing Observer: verify notification
@Test
void order_notifies_listeners_on_status_change() {
    Order order = new Order(customer, restaurant, items);
    OrderStatusListener mockListener = Mockito.mock(OrderStatusListener.class);
    order.addStatusListener(mockListener);

    order.transitionTo(OrderStatus.CONFIRMED);

    verify(mockListener).onStatusChange(order, OrderStatus.CONFIRMED);
}

// Testing Decorator: verify it adds behavior and delegates
@Test
void logging_repository_logs_and_delegates_find() {
    OrderRepository mockDelegate = Mockito.mock(OrderRepository.class);
    Order expectedOrder = new Order("O-1");
    when(mockDelegate.findById("O-1")).thenReturn(expectedOrder);

    LoggingOrderRepository logging = new LoggingOrderRepository(mockDelegate);
    Order result = logging.findById("O-1");

    assertEquals(expectedOrder, result);
    verify(mockDelegate).findById("O-1"); // delegation verified
    // Also verify logging side effect via log capture or output stream spy
}
```

---

### Q96. Why should tests not share state, and what are examples of isolation failures in Java?

**Model Answer:** Test state isolation ensures that each test's outcome depends only on its own setup and the code under test — not on which other tests ran before it or in what order. Shared state creates non-deterministic ("flaky") tests: a test passes when run alone but fails as part of the suite because a previous test left the shared object in an unexpected state. Common isolation failures in Java include: static mutable fields in the class under test modified by one test and observed by another; tests sharing a Spring `ApplicationContext` with a database and one test dirtying data; `Mockito.mock()` created in a `@BeforeAll` and not reset between tests; and file system state (temp files created but not cleaned up).

```java
// Isolation failure: static mutable counter shared across tests
public class OrderIdGenerator {
    private static int counter = 0;      // static mutable — shared across all tests
    public static String next() { return "O-" + ++counter; }
}

@Test void test_a() { assertEquals("O-1", OrderIdGenerator.next()); } // passes if run first
@Test void test_b() { assertEquals("O-1", OrderIdGenerator.next()); } // FAILS if run after test_a

// Fix: reset state in @BeforeEach, or use instance fields instead of static
@BeforeEach void reset() { OrderIdGenerator.reset(); } // or use instance-scoped generator

// Isolation failure: shared Mockito mock not reset
class OrderServiceTest {
    private static final EmailService mockEmail = Mockito.mock(EmailService.class); // shared!

    @Test void test_a() { service.placeOrder(order); verify(mockEmail, times(1)).send(any()); }
    @Test void test_b() { service.placeOrder(order); verify(mockEmail, times(1)).send(any()); }
    // test_b FAILS: mock was called once in test_a, now count is 2
}

// Fix: create new mock per test
@BeforeEach void setUp() {
    mockEmail = Mockito.mock(EmailService.class); // fresh mock each test
    service = new OrderService(mockRepository, mockEmail);
}
```

---

### Q97. What are equivalence partitioning and boundary value analysis, and how do you apply them to write complete unit tests?

**Model Answer:** Equivalence partitioning divides input space into groups (partitions) where all values in a group are expected to produce the same behavior — test one representative from each partition rather than exhaustively testing every value. Boundary value analysis tests at the edges of each partition, where off-by-one errors and edge conditions are most likely to lurk. For a function that accepts age 18-65 as valid: partitions are (age < 18), (18 <= age <= 65), (age > 65); boundary values are 17, 18, 65, 66. Together these two techniques give near-complete coverage with a minimal but targeted test set.

```java
// Function under test: validate that a discount applies to ages 18-65
public class AgeDiscountPolicy {
    public boolean isEligible(int age) {
        return age >= 18 && age <= 65;
    }
}

class AgeDiscountPolicyTest {
    AgeDiscountPolicy policy = new AgeDiscountPolicy();

    // Equivalence partition: below minimum (any value < 18 should be ineligible)
    @Test void age_below_minimum_is_not_eligible() { assertFalse(policy.isEligible(10)); }

    // Boundary value: just below minimum
    @Test void age_17_is_not_eligible() { assertFalse(policy.isEligible(17)); }

    // Boundary value: exactly at minimum
    @Test void age_18_is_eligible() { assertTrue(policy.isEligible(18)); }

    // Equivalence partition: within valid range
    @Test void age_in_valid_range_is_eligible() { assertTrue(policy.isEligible(40)); }

    // Boundary value: exactly at maximum
    @Test void age_65_is_eligible() { assertTrue(policy.isEligible(65)); }

    // Boundary value: just above maximum
    @Test void age_66_is_not_eligible() { assertFalse(policy.isEligible(66)); }

    // Equivalence partition: above maximum
    @Test void age_above_maximum_is_not_eligible() { assertFalse(policy.isEligible(80)); }

    // Edge cases
    @Test void age_zero_is_not_eligible() { assertFalse(policy.isEligible(0)); }
    @Test void negative_age_is_not_eligible() { assertFalse(policy.isEligible(-1)); }
}
```

---

### Q98. What is property-based testing and how do you implement it in Java using jqwik?

**Model Answer:** Property-based testing (PBT) specifies invariant properties that must hold for *all* inputs in a domain, then automatically generates hundreds of random inputs to try to falsify the property. Unlike example-based tests (which verify specific cases), PBT explores the input space systematically and finds edge cases the developer did not think of — it is particularly powerful for finding off-by-one errors, commutativity violations, and round-trip encoding bugs. When a failing case is found, the framework "shrinks" it to the minimal failing input, making debugging easier. In Java, `jqwik` is the leading PBT library, integrating with JUnit 5.

```java
import net.jqwik.api.*;
import net.jqwik.api.constraints.*;

class MoneyPropertyTest {

    // Property: adding two non-negative amounts never decreases the total
    @Property
    void adding_money_is_commutative(@ForAll @DoubleRange(min = 0, max = 10000) double a,
                                      @ForAll @DoubleRange(min = 0, max = 10000) double b) {
        Money moneyA = new Money(BigDecimal.valueOf(a), Currency.USD);
        Money moneyB = new Money(BigDecimal.valueOf(b), Currency.USD);

        Money sum1 = moneyA.add(moneyB);
        Money sum2 = moneyB.add(moneyA);

        assertEquals(sum1, sum2); // commutativity: a + b == b + a
    }

    // Property: round-trip serialization preserves value
    @Property
    void base62_encode_decode_is_identity(@ForAll @LongRange(min = 1, max = Long.MAX_VALUE) long value) {
        Base62CodeGenerator gen = new Base62CodeGenerator();
        String encoded = gen.encode(value);
        long decoded = gen.decode(encoded);
        assertEquals(value, decoded);
    }

    // Property: LRU cache never exceeds capacity
    @Property
    void lru_cache_never_exceeds_capacity(
            @ForAll @Size(min = 1, max = 100) List<@IntRange(min = 1, max = 50) Integer> keys) {
        LruCache<Integer, String> cache = new LruCache<>(10);
        for (int key : keys) {
            cache.put(key, "value-" + key);
            assertTrue(cache.size() <= 10, "Cache exceeded capacity after inserting key: " + key);
        }
    }
}
```

---

### Q99. What is mutation testing and what does it measure that line coverage does not? How does PITest work in Java?

**Model Answer:** Line coverage measures which lines were executed by tests; it says nothing about whether the tests would detect a bug if the code changed. Mutation testing introduces small code mutations (flip `>` to `>=`, change `+` to `-`, delete a method call) and checks whether the test suite fails — if a mutation survives (tests still pass), it means the tests are not sensitive enough to catch that class of bug. PITest (PIT) is the leading Java mutation testing tool: it instruments bytecode to generate mutants, runs the test suite against each mutant, and reports a mutation score (killed/total mutants). A project with 90% line coverage but 40% mutation score has tests that execute lines without asserting on their behavior.

```java
// Example: this test has 100% line coverage but 0% mutation score
public class Calculator {
    public int add(int a, int b) { return a + b; }
}

@Test
void add_runs_without_error() {
    Calculator calc = new Calculator();
    calc.add(2, 3); // executes the line but never asserts the return value!
    // PITest mutation: changes a + b to a - b -> test still passes -> mutation survives
}

// PITest-effective test: asserts the result
@Test
void add_returns_sum() {
    Calculator calc = new Calculator();
    assertEquals(5, calc.add(2, 3)); // mutant a - b returns -1, test fails -> mutant killed
    assertEquals(0, calc.add(-1, 1));
    assertEquals(0, calc.add(0, 0));
}

// PIT Maven configuration (pom.xml)
// <plugin>
//   <groupId>org.pitest</groupId>
//   <artifactId>pitest-maven</artifactId>
//   <version>1.15.0</version>
//   <configuration>
//     <targetClasses><param>com.example.*</param></targetClasses>
//     <targetTests><param>com.example.*Test</param></targetTests>
//     <mutationThreshold>80</mutationThreshold>  <!-- fail build below 80% -->
//   </configuration>
// </plugin>
```

---

### Q100. How does difficulty in testing reveal design problems?

**Model Answer:** Testability is a proxy for design quality because well-designed code — with single responsibilities, explicit dependencies, and no hidden global state — is naturally easy to test. The inverse is equally reliable: if a class is hard to test, something in the design is wrong. Specific test pain maps to specific design problems: needing to construct half the application to test one method signals violated SRP or missing DIP; needing `PowerMock` to mock a static method signals a global dependency or inappropriate static state; needing to set system clocks or environment variables signals non-injected infrastructure; needing to look inside private state to verify behavior signals an overly procedural (anemic) model that should be richer.

```java
// Test pain map:

// PAIN: need to create 10 objects to test 1 method
// CAUSE: the class under test has too many dependencies (SRP violation)
// FIX: split the class; inject only what's needed
new OrderService(
    new UserRepository(new DB(config)),
    new PaymentGateway(apiKey),
    new EmailService(smtp),
    new SmsService(twilioKey),
    new AuditLogger(db),
    new MetricsService(graphite)
); // This is screaming to be split

// PAIN: must set static variables before test
// CAUSE: static global state (singleton with mutable state)
// FIX: inject dependencies
GlobalConfig.setEnvironment("test"); // test modifies global — isolation violation
// Fix: inject Config object

// PAIN: can only verify behavior by querying DB after the call
// CAUSE: anemic model, no return value, no observable event
// FIX: have the method return a result or raise a domain event
void placeOrder(Order order) { /* saves to DB — only way to verify is query DB */ }
// Fix:
OrderId placeOrder(Order order) { ... return savedOrder.getId(); } // testable via return value

// PAIN: need to use reflection to check private field value
// CAUSE: the behavior is not expressed in the public API
// FIX: express the outcome as a public query
// Instead of: assertEquals("CONFIRMED", getPrivateField(order, "status"))
// Do:         assertEquals(OrderStatus.CONFIRMED, order.getStatus())
```

---
## Bonus: Senior/Staff Engineer Discussion Topics

### Q101. Technical Debt Decision Framework — How do you decide when to accrue versus pay down technical debt?

**Expected Depth for Senior Level:** Technical debt is a deliberate or accidental decision to take a shortcut today that will require remediation later — the analogy to financial debt is precise: like interest, deferred remediation makes the original shortcut increasingly expensive over time. The decision framework has two axes: probability that the code will change (high-churn code makes debt compound fast; stable code can carry more debt longer) and the current cost of the debt (does it slow down every feature in that area, or is it isolated to one rarely-touched module). Accrue debt intentionally only when the speed benefit is time-critical (MVP launch, competitive deadline) and you document it explicitly with a `// TECH-DEBT:` comment, a Jira ticket, and an agreed repayment window. Pay it down when it is actively slowing velocity (developers cite it in every planning session), when it is in a change-hotspot (more than 2-3 PRs per sprint touch the class), or when the risk of a production bug from the shortcut exceeds the cost of the refactor. Never accrue debt on security boundaries, data integrity paths, or observability — the blast radius of failure there is too high.

---

### Q102. API Versioning Strategy — Compare URI versioning, header versioning, and media type versioning.

**Expected Depth for Senior Level:** URI versioning (`/v1/orders`, `/v2/orders`) is the most visible and simplest to implement — routing is trivial, curl-testable without extra headers, and API gateways handle it out of the box; the downside is that it pollutes the URI space (URIs should identify resources, not API contract versions) and makes clients couple to a version number that forces them to migrate when they may not need to. Header versioning (`Api-Version: 2`) keeps URIs clean and allows version negotiation without breaking existing links, but is invisible in browser address bars, harder to test, and requires middleware to inspect headers for routing. Media type versioning (`Accept: application/vnd.company.order.v2+json`) is the most REST-correct approach — it represents different resource representations as distinct media types, aligns with HTTP content negotiation, and allows fine-grained per-field versioning — but it adds significant complexity to client code and API documentation. In practice, URI versioning dominates public APIs for its developer-experience simplicity; header versioning is common in internal microservices; media type versioning is rarely chosen outside large, highly REST-principled organizations. The most important thing is not which strategy you choose but that you maintain backward compatibility by additive changes (new fields, new optional parameters) before resorting to a new version at all.

---

### Q103. Domain Model Evolution — How do you evolve a domain model without breaking existing clients?

**Expected Depth for Senior Level:** Domain model evolution requires distinguishing additive changes (backward-compatible: new optional fields, new operations) from breaking changes (removed fields, changed semantics, renamed concepts) — additive changes can be deployed without a versioning increment if clients tolerate unknown fields (Postel's Law). For breaking changes, the Expand-Contract pattern (also called the parallel-change pattern) is the safest approach: first expand (add the new field/behavior alongside the old), then migrate all consumers to the new version, then contract (remove the old). In event-sourced systems, domain model evolution is particularly sensitive because past events are immutable — schema migration means adding upcasters (functions that transform old event schemas to the current one on read), not modifying stored events. Value objects should be designed with serialization forward-compatibility in mind: mark fields optional in serialization schemas, use schema registries (Avro, Protobuf) with compatibility enforcement, and never use Java's default serialization for domain objects that cross service boundaries. The most overlooked dimension is semantic drift: the same field name with a changed meaning is the hardest evolution to manage because it is invisible to type systems — document domain concepts in a ubiquitous language glossary and version the glossary alongside the code.

---

### Q104. Extensibility vs YAGNI Tension — How do you resolve this tension in a design review?

**Expected Depth for Senior Level:** The tension is real and the resolution depends on the cost asymmetry of the specific extensibility decision: if adding a Strategy pattern costs 2 hours now and removing an unneeded abstraction later costs 30 minutes, YAGNI wins; if adding a Strategy pattern now costs 2 hours but retrofitting it later (after the codebase has grown around the concrete type) costs 3 days and involves a risky refactoring, extensibility wins. A practical heuristic for design reviews is the "Rule of Three" (from Martin Fowler's Refactoring): the first time you need something, just do it inline; the second time, note the duplication; the third time, extract the abstraction — real variation patterns are visible by the third case. Another filter: extensibility for a known, identified, near-term requirement is legitimate; extensibility for a hypothetical, unscheduled requirement is speculative generality. In a design review, the challenge to speculative extensibility is "what is the concrete use case that requires this extension point today?" — if the answer is "we might need it for future payment types," the appropriate response is to document the current design, commit to refactoring when the second payment type arrives, and defer the abstraction until then.

---

### Q105. Team API Boundaries — How does Conway's Law shape your LLD, and what should you do about it?

**Expected Depth for Senior Level:** Conway's Law states that organizations design systems that mirror their communication structures — a team that owns both the order service and the payment service will produce an architecture with a tightly coupled boundary between them that reflects the ease of internal team communication. At the LLD level, this manifests as: classes in the same team's service share internal data structures that should be opaque to other services, interfaces are designed to serve the owning team's convenience rather than the consuming team's needs, and API contracts drift because the same team writes both sides. The proactive response is the "Inverse Conway Maneuver" (coined by Thoughtworks): intentionally design the team structure to match the desired architecture, not the other way around. In LLD reviews, ask: "if this interface were consumed by a team that cannot contact us directly, would it be clear and sufficient?" — if the answer requires a hallway conversation, the interface is leaking internal design concerns and should be refactored into a richer, more self-describing contract.

---

### Q106. Conway's Law in Practice — How do you exploit or counteract it intentionally?

**Expected Depth for Senior Level:** Exploiting Conway's Law intentionally means aligning team boundaries with desired module/service boundaries before starting a major refactoring — if you want a clean separation between the catalog and inventory domains, form a catalog team and an inventory team with a formal interface between them before writing the code, because the team boundary will naturally produce the API boundary. Counteracting it when team boundaries cannot change requires explicit architectural governance: API review boards, consumer-driven contract tests (Pact), and interface ownership documentation that forces a team to think of their module as a public API regardless of organizational proximity. A specific LLD pattern that counteracts Conway's Law is the Anti-Corruption Layer (from DDD): a translation layer between two domains owned by different teams that prevents either team's internal model from contaminating the other, even when both teams are under pressure to share internal types for expediency. Distributed teams that share a monorepo are particularly at risk — the ease of importing across module boundaries makes it tempting to bypass intended API contracts, so module-level access enforcement (JPMS modules in Java 9+, or ArchUnit rules in CI) is needed to make Conway's Law work for rather than against the architecture.

---

### Q107. Strangler Fig Pattern — How do you apply it at the class/module level, not just the service level?

**Expected Depth for Senior Level:** The Strangler Fig pattern (Martin Fowler) is conventionally described at the service level: build a new service alongside the legacy one, route an increasing percentage of traffic to the new service, and strangle the old one when migration is complete. At the class and module level, the same principle applies: introduce a new interface alongside the old implementation, create a new implementation of that interface, add a routing/delegation layer that dispatches to old or new based on a feature flag or configuration, and gradually migrate callers to use the new interface directly before deleting the old one. This is especially valuable for replacing a God Class: create a new `OrderService` interface with cleaner responsibilities, implement it fully, introduce a `DelegatingOrderService` that wraps both old and new and can be configured to route each operation, migrate one operation at a time, and eventually delete the old class. The key discipline is to never let both implementations diverge in behavior — the new one must be a behavioral superset of the old one during the transition period, meaning comprehensive regression tests against the common interface are essential before the migration begins.

---

### Q108. Event-Driven vs Request-Driven Design — How does the choice affect your LLD class structure?

**Expected Depth for Senior Level:** Request-driven design produces synchronous, layered class structures: controllers call services call repositories, the call stack is the unit of composition, and each operation has a clear request/response contract; this is natural for CRUD operations where the caller needs an immediate result. Event-driven design produces reactive, decoupled class structures: publishers emit events to a bus or queue without knowing who will handle them, subscribers react independently, and the system is composed of independent event handlers rather than a call stack; this is natural for long-running processes, eventual consistency workflows, and systems where multiple downstream actions must happen in response to one trigger. At the LLD level, event-driven designs introduce explicit `Event` value objects as the communication contract between classes, `EventHandler` interfaces that decouple producers from consumers, and `EventStore`/`Outbox` patterns to ensure events are not lost if a handler fails. The key LLD tradeoff is that event-driven designs are harder to trace end-to-end (no single call stack to follow in a debugger) and require explicit correlation IDs to reconstruct what happened — these must be designed in from the start, not bolted on later.

---

### Q109. Eventual Consistency in Domain Models — What does it mean at the object-model level?

**Expected Depth for Senior Level:** Eventual consistency at the object model level means that not all parts of the system have the same view of the domain state at the same instant — a `Customer`'s credit score in the billing context may lag behind an update made in the identity context by seconds or minutes. At the LLD level, this means domain objects should not assume that related aggregates are always in their most current state: cross-aggregate state is accessed via read models (denormalized projections optimized for query), not by traversing live aggregate references. Invariants that span multiple aggregates cannot be enforced atomically without distributed locking (which is expensive and fragile); instead, they are enforced with sagas (a sequence of compensating transactions) or process managers that coordinate eventual convergence. In Java class design, the practical implication is that domain events are first-class objects in the model: when `Order.confirm()` runs, it does not directly call `InventoryService.reserve()` — it publishes an `OrderConfirmedEvent` to the domain event bus, and the inventory service reacts asynchronously, emitting `InventoryReservedEvent` or `InsufficientStockEvent` in return. Classes in an eventually consistent system are smaller and more focused (each handles one event type), but the overall system behavior is harder to reason about without explicit saga/workflow documentation.

---

### Q110. Architect vs Engineer Role in LLD — When should a Senior Engineer push back on architectural decisions?

**Expected Depth for Senior Level:** A Senior Engineer is the closest to the implementation details and often sees consequences of architectural decisions that architects and staff engineers further from the code do not — this positional advantage creates both the right and the responsibility to push back when an architectural constraint makes a class design unmaintainable, untestable, or unsafe. Legitimate pushback scenarios include: a prescribed pattern that forces widespread duplication (e.g., a "no shared libraries" microservices rule that causes 20 services to copy-paste the same authentication logic); a data model decision that prevents enforcing a critical domain invariant at the class level (e.g., a shared-database architecture that allows any service to write directly to another service's tables); and a technology choice that makes testing prohibitively expensive (e.g., mandatory use of a vendor framework that requires a full application boot for every unit test). The pushback should be evidence-based and solution-oriented: not "this is wrong" but "here is the specific LLD consequence of this decision, here is an alternative that meets the architectural goal with a better implementation property, and here is the cost comparison." Escalating through the right channels (RFC process, architecture review board, technical leadership) rather than unilaterally deviating is critical for trust and for ensuring the decision is revisited at the right level rather than individually worked around, which creates inconsistency across the codebase.
