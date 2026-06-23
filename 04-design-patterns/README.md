# Design Patterns — Comprehensive Reference Guide

> "Each pattern describes a problem which occurs over and over again in our environment, and then describes the core of the solution to that problem, in such a way that you can use this solution a million times over, without ever doing it the same way twice."
>
> — Christopher Alexander (quoted in GoF, 1994)

---

## Table of Contents

1. [What Are Design Patterns?](#1-what-are-design-patterns)
2. [The Gang of Four — Historical Context](#2-the-gang-of-four--historical-context)
3. [The Three Categories](#3-the-three-categories)
   - 3.1 [Creational Patterns (5)](#31-creational-patterns-5)
   - 3.2 [Structural Patterns (7)](#32-structural-patterns-7)
   - 3.3 [Behavioral Patterns (11)](#33-behavioral-patterns-11)
4. [Patterns vs Over-Engineering](#4-patterns-vs-over-engineering)
5. [Pattern Selection Guide](#5-pattern-selection-guide)
6. [Patterns and SOLID Principles](#6-patterns-and-solid-principles)
7. [How to Study Patterns Effectively](#7-how-to-study-patterns-effectively)
8. [Quick Reference — All 23 GoF Patterns](#8-quick-reference--all-23-gof-patterns)

---

## 1. What Are Design Patterns?

A **design pattern** is a general, reusable solution to a commonly occurring problem in a given context within software design. Patterns are not finished code you paste into a project — they are templates, blueprints, or descriptions of how to solve a problem that can be applied in many different situations.

### Key Characteristics

| Property | Explanation |
|---|---|
| **Name** | A single, memorable vocabulary term that lets engineers communicate precise ideas quickly |
| **Problem** | The situation or context in which the pattern applies |
| **Solution** | An abstract description of the elements that make up the design, their relationships, and responsibilities |
| **Consequences** | The trade-offs — time, space, flexibility, testability costs of applying the pattern |

### Why They Matter

1. **Shared vocabulary.** When a senior engineer says "use a Strategy here," every other engineer on the team immediately understands a class hierarchy is about to be introduced to isolate an algorithm. Without patterns, that conversation requires several minutes of whiteboard explanation.

2. **Proven solutions.** Patterns encode hard-won wisdom from hundreds of real production systems. You avoid rediscovering the same mistakes.

3. **Raise the level of design conversation.** Patterns allow you to discuss design at a higher level of abstraction than individual classes and methods.

4. **Documentation shorthand.** A comment like `// Decorator pattern — adds logging without touching the base class` communicates far more than a paragraph of prose.

5. **Interview and system design currency.** At every FAANG/MAANG system design or LLD interview, pattern knowledge is assumed. Not knowing them is a fast-track to a no-hire.

### What Patterns Are NOT

- They are **not algorithms** — algorithms solve computational problems. Patterns solve structural/collaboration problems.
- They are **not libraries or frameworks** — you cannot import a design pattern. You must implement the idea yourself.
- They are **not the only way** — a pattern is a *best practice*, not a mandate. The absence of a pattern is sometimes the right answer.

---

## 2. The Gang of Four — Historical Context

In **1994**, four computer scientists published the book that changed how the industry thinks about software design:

**"Design Patterns: Elements of Reusable Object-Oriented Software"**

Authors (the "Gang of Four"):
- **Erich Gamma** — later co-creator of Eclipse IDE, contributor to JUnit, Distinguished Engineer at Microsoft
- **Richard Helm** — researcher at IBM
- **Ralph Johnson** — professor at University of Illinois, Urbana-Champaign
- **John Vlissides** — researcher at IBM T.J. Watson Research Center (passed away 2005)

The book catalogued **23 patterns** across three categories. It remains one of the best-selling programming books ever written — often called simply "the GoF book."

### Why 1994 Was the Right Moment

By the early 1990s, object-oriented programming (OOP) with C++ and later Java was mainstream. Developers had the *tools* (classes, polymorphism, inheritance) but lacked a *shared language* for talking about how to combine those tools well. The GoF book provided exactly that — a vocabulary and a catalogue grounded in real, production-grade codebases.

### Beyond GoF

Since 1994, the pattern community has grown significantly:

- **POSA (Pattern-Oriented Software Architecture)** — architectural patterns for distributed systems
- **Enterprise Integration Patterns** — Gregor Hohpe and Bobby Woolf (2003), messaging patterns
- **Domain-Driven Design patterns** — Eric Evans (2003), Repository, Aggregate, Value Object, etc.
- **Concurrency patterns** — Active Object, Monitor Object, Half-Sync/Half-Async

This guide focuses on the original 23 GoF patterns because they are foundational to everything that followed.

---

## 3. The Three Categories

### 3.1 Creational Patterns (5)

**Core question:** How do we create objects?

Creational patterns abstract the instantiation process. They help make a system independent of how its objects are created, composed, and represented. They become important when a system needs to use objects whose creation must be decoupled from their use.

---

#### 3.1.1 Singleton

**Intent:** Ensure a class has only one instance, and provide a global point of access to it.

**Problem it solves:** You need exactly one object to coordinate actions across the system — a configuration registry, a connection pool manager, a logging service, or a thread pool.

**Structure:**
```
+------------------+
|    Singleton     |
+------------------+
| -instance: self  |
+------------------+
| -Singleton()     |
| +getInstance()   |
+------------------+
```

**Implementation (Java — thread-safe double-checked locking):**
```java
public class ConfigRegistry {
    // volatile ensures visibility across threads
    private static volatile ConfigRegistry instance;
    private final Map<String, String> config = new HashMap<>();

    private ConfigRegistry() {
        // private constructor prevents external instantiation
        loadFromFile();
    }

    public static ConfigRegistry getInstance() {
        if (instance == null) {                   // first check (no lock)
            synchronized (ConfigRegistry.class) {
                if (instance == null) {           // second check (with lock)
                    instance = new ConfigRegistry();
                }
            }
        }
        return instance;
    }

    public String get(String key) {
        return config.getOrDefault(key, "");
    }
}
```

**When to use:**
- Shared resource that must be controlled (database connection pool, thread pool)
- A single configuration object read across many classes
- A logging service

**Pitfalls:**
- Global state makes unit testing hard — consider dependency injection instead
- Singletons in multi-classloader environments (OSGi, app servers) can produce multiple instances
- Makes it harder to swap implementations

**Real-world examples:** `java.lang.Runtime.getRuntime()`, Spring's default bean scope, `Logger` instances

---

#### 3.1.2 Factory Method

**Intent:** Define an interface for creating an object, but let subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses.

**Problem it solves:** A framework needs to create objects but doesn't know what concrete class to instantiate because that detail belongs to the application using the framework.

**Structure:**
```
Creator (abstract)              Product (interface)
+------------------+            +------------------+
| +factoryMethod() |------>     | +operation()     |
| +anOperation()   |            +------------------+
+------------------+                    ^
         ^                              |
         |                    ConcreteProduct
ConcreteCreator             +------------------+
+------------------+        | +operation()     |
| +factoryMethod() |------> +------------------+
+------------------+
```

**Implementation (Java):**
```java
// Product interface
public interface Notification {
    void send(String message, String recipient);
}

// Concrete products
public class EmailNotification implements Notification {
    public void send(String message, String recipient) {
        System.out.println("Email to " + recipient + ": " + message);
    }
}

public class SMSNotification implements Notification {
    public void send(String message, String recipient) {
        System.out.println("SMS to " + recipient + ": " + message);
    }
}

// Creator (abstract)
public abstract class NotificationService {
    // The factory method
    protected abstract Notification createNotification();

    // Template method that uses the factory method
    public void notify(String message, String recipient) {
        Notification n = createNotification();
        n.send(message, recipient);
    }
}

// Concrete creators
public class EmailNotificationService extends NotificationService {
    protected Notification createNotification() {
        return new EmailNotification();
    }
}

public class SMSNotificationService extends NotificationService {
    protected Notification createNotification() {
        return new SMSNotification();
    }
}
```

**When to use:**
- A class cannot anticipate the class of objects it must create
- You want subclasses to specify the objects they create
- You want to encapsulate object creation logic to keep it in one place

**Pitfalls:**
- Can result in many parallel class hierarchies (one per product family)
- May be overkill when only a `new` keyword would suffice

**Real-world examples:** `java.util.Collections.synchronizedList()`, JDBC `DriverManager.getConnection()`, Spring's `BeanFactory`

---

#### 3.1.3 Abstract Factory

**Intent:** Provide an interface for creating families of related or dependent objects without specifying their concrete classes.

**Problem it solves:** A system must be independent of how its products are created, composed, and represented, AND it must work with multiple families of products. You never want to mix products from different families.

**Structure:**
```
AbstractFactory                     AbstractProductA
+----------------------+            +-----------------+
| +createProductA()    |            | +operationA()   |
| +createProductB()    |            +-----------------+
+----------------------+                    ^
         ^                     ConcreteProductA1 / A2
         |
ConcreteFactory1             AbstractProductB
+----------------------+     +-----------------+
| +createProductA()    |     | +operationB()   |
| +createProductB()    |     +-----------------+
+----------------------+              ^
                             ConcreteProductB1 / B2
```

**Implementation (Java — UI theme example):**
```java
// Abstract products
public interface Button { void render(); void onClick(); }
public interface TextField { void render(); String getValue(); }

// Abstract factory
public interface UIFactory {
    Button createButton();
    TextField createTextField();
}

// Concrete products — Light theme
public class LightButton implements Button {
    public void render() { System.out.println("Render light button"); }
    public void onClick() { System.out.println("Light button clicked"); }
}
public class LightTextField implements TextField {
    public void render() { System.out.println("Render light text field"); }
    public String getValue() { return "light-value"; }
}

// Concrete factory — Light theme
public class LightThemeFactory implements UIFactory {
    public Button createButton() { return new LightButton(); }
    public TextField createTextField() { return new LightTextField(); }
}

// Dark theme follows the same structure

// Client
public class Application {
    private final Button button;
    private final TextField textField;

    public Application(UIFactory factory) {
        button = factory.createButton();
        textField = factory.createTextField();
    }

    public void buildUI() {
        button.render();
        textField.render();
    }
}
```

**Difference from Factory Method:**
- Factory Method uses *inheritance* — subclasses decide what to create
- Abstract Factory uses *composition* — a factory object is passed in (dependency injection)
- Abstract Factory creates *families* of related objects; Factory Method creates *one* type

**When to use:**
- When a system must be independent of how its products are created
- When you need to enforce that a family of related objects is always used together
- Cross-platform UI toolkits, database drivers per vendor, cloud provider SDK abstraction

**Real-world examples:** `javax.xml.parsers.DocumentBuilderFactory`, Java AWT/Swing peer factories, Spring's `ApplicationContext`

---

#### 3.1.4 Builder

**Intent:** Separate the construction of a complex object from its representation so that the same construction process can create different representations.

**Problem it solves:** An object requires many steps or many optional parameters to construct. Using a constructor with 15 parameters (the "telescoping constructor antipattern") is unreadable and error-prone.

**Structure:**
```
Director                  Builder (interface)
+--------------+          +-------------------+
| +construct() |--------> | +buildPartA()     |
+--------------+          | +buildPartB()     |
                          | +getResult()      |
                          +-------------------+
                                   ^
                          ConcreteBuilder
                          +-------------------+
                          | +buildPartA()     |
                          | +buildPartB()     |
                          | +getResult()      |
                          +-------------------+
                                   |
                                Product
```

**Implementation (Java — HTTP Request Builder):**
```java
public class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final String body;
    private final int timeoutMs;
    private final boolean followRedirects;

    private HttpRequest(Builder builder) {
        this.url = builder.url;
        this.method = builder.method;
        this.headers = Collections.unmodifiableMap(builder.headers);
        this.body = builder.body;
        this.timeoutMs = builder.timeoutMs;
        this.followRedirects = builder.followRedirects;
    }

    public static class Builder {
        // Required
        private final String url;
        // Optional with defaults
        private String method = "GET";
        private Map<String, String> headers = new HashMap<>();
        private String body = null;
        private int timeoutMs = 5000;
        private boolean followRedirects = true;

        public Builder(String url) {
            this.url = url;
        }

        public Builder method(String method) { this.method = method; return this; }
        public Builder header(String key, String value) { headers.put(key, value); return this; }
        public Builder body(String body) { this.method = "POST"; this.body = body; return this; }
        public Builder timeoutMs(int ms) { this.timeoutMs = ms; return this; }
        public Builder noRedirects() { this.followRedirects = false; return this; }

        public HttpRequest build() {
            if (url == null || url.isBlank()) throw new IllegalStateException("URL is required");
            return new HttpRequest(this);
        }
    }
}

// Usage — reads like English
HttpRequest request = new HttpRequest.Builder("https://api.example.com/users")
    .method("POST")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token123")
    .body("{\"name\": \"Alice\"}")
    .timeoutMs(10000)
    .build();
```

**When to use:**
- Objects with many optional parameters
- Objects that require step-by-step construction
- When you want to ensure an object is fully configured before use (immutability)
- Producing different representations from the same construction process

**Real-world examples:** `StringBuilder`, `AlertDialog.Builder` (Android), `OkHttpClient.Builder`, `Stream.Builder`, Lombok's `@Builder`

---

#### 3.1.5 Prototype

**Intent:** Specify the kinds of objects to create using a prototypical instance, and create new objects by copying (cloning) this prototype.

**Problem it solves:** Object creation is expensive (database lookup, complex initialization), and you need many similar objects differing only in a few fields. Cloning the expensive object is faster than creating from scratch.

**Structure:**
```
Prototype (interface)
+-------------------+
| +clone(): self    |
+-------------------+
         ^
ConcretePrototype
+-------------------+
| -field1           |
| -field2           |
| +clone(): self    |
+-------------------+
```

**Implementation (Java):**
```java
public abstract class Shape implements Cloneable {
    protected String color;
    protected int x, y;

    public abstract double area();

    @Override
    public Shape clone() {
        try {
            return (Shape) super.clone(); // shallow copy via Object.clone()
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
}

public class Circle extends Shape {
    private double radius;

    public Circle(String color, int x, int y, double radius) {
        this.color = color; this.x = x; this.y = y; this.radius = radius;
    }

    // Deep clone if needed
    @Override
    public Circle clone() {
        return (Circle) super.clone(); // double is primitive, safe
    }

    public double area() { return Math.PI * radius * radius; }
}

// Prototype registry
public class ShapeRegistry {
    private final Map<String, Shape> prototypes = new HashMap<>();

    public void register(String key, Shape shape) {
        prototypes.put(key, shape);
    }

    public Shape get(String key) {
        Shape proto = prototypes.get(key);
        if (proto == null) throw new IllegalArgumentException("Unknown prototype: " + key);
        return proto.clone();
    }
}
```

**Shallow vs Deep Copy Warning:**
- **Shallow copy** copies primitive fields by value, object references by reference — the clone shares mutable objects with the original
- **Deep copy** recursively copies all objects — safe but more expensive
- Always determine which is appropriate for your use case

**When to use:**
- Object creation is expensive and similar objects are needed frequently
- You want to avoid subclassing a creator in the way Factory Method requires
- Game development (cloning enemy instances from a template)
- Document templates, configuration snapshots

**Real-world examples:** Java's `Cloneable` interface, JavaScript's `Object.create()`, Spring prototype scope beans

---

### 3.2 Structural Patterns (7)

**Core question:** How do we compose objects and classes into larger structures?

Structural patterns explain how to assemble objects and classes into larger structures while keeping these structures flexible and efficient.

---

#### 3.2.1 Adapter

**Intent:** Convert the interface of a class into another interface that clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces.

**Problem it solves:** You have existing code you cannot change (third-party library, legacy system), and its interface doesn't match what your system expects.

**Two variants:**
- **Object Adapter** — uses composition (preferred, more flexible)
- **Class Adapter** — uses multiple inheritance (Java doesn't support this directly)

**Implementation (Java — Object Adapter):**
```java
// Your system expects this interface
public interface PaymentProcessor {
    boolean processPayment(String userId, double amount, String currency);
}

// Third-party library you cannot modify
public class StripeClient {
    public StripeResponse charge(StripeChargeRequest request) {
        // Stripe-specific implementation
        return new StripeResponse(true, "ch_123");
    }
}

// Adapter bridges the two
public class StripePaymentAdapter implements PaymentProcessor {
    private final StripeClient stripeClient;

    public StripePaymentAdapter(StripeClient stripeClient) {
        this.stripeClient = stripeClient;
    }

    @Override
    public boolean processPayment(String userId, double amount, String currency) {
        StripeChargeRequest request = new StripeChargeRequest()
            .setUserId(userId)
            .setAmountCents((long)(amount * 100))   // Stripe uses cents
            .setCurrency(currency.toLowerCase());    // Stripe uses lowercase

        StripeResponse response = stripeClient.charge(request);
        return response.isSuccess();
    }
}

// Client code — only knows PaymentProcessor, not Stripe at all
public class OrderService {
    private final PaymentProcessor paymentProcessor;

    public OrderService(PaymentProcessor paymentProcessor) {
        this.paymentProcessor = paymentProcessor;
    }

    public void checkout(Order order) {
        boolean paid = paymentProcessor.processPayment(
            order.getUserId(), order.getTotal(), order.getCurrency()
        );
        if (paid) order.markComplete();
    }
}
```

**When to use:**
- Integrating a third-party library or legacy system whose interface differs from yours
- When you want to reuse existing classes but their interfaces don't match
- Migration: new interface adapter wraps old implementation

**Real-world examples:** `java.io.InputStreamReader` (adapts InputStream to Reader), Spring's `HandlerAdapter`, JDBC drivers

---

#### 3.2.2 Bridge

**Intent:** Decouple an abstraction from its implementation so that the two can vary independently.

**Problem it solves:** You have an abstraction (e.g., Shape) with multiple implementations (e.g., rasterization engines: OpenGL, DirectX). Without Bridge, you'd need classes for every combination: `OpenGLCircle`, `DirectXCircle`, `OpenGLSquare`, etc. — a combinatorial explosion.

**Structure:**
```
Abstraction                 Implementor (interface)
+-----------------+         +-------------------+
| -impl: Impl     |-------> | +operationImpl()  |
| +operation()    |         +-------------------+
+-----------------+                  ^
         ^                  ConcreteImplementorA
RefinedAbstraction          ConcreteImplementorB
+-----------------+
| +operation()    |
+-----------------+
```

**Implementation (Java):**
```java
// Implementor interface
public interface Renderer {
    void renderCircle(double x, double y, double radius);
    void renderRectangle(double x, double y, double width, double height);
}

// Concrete Implementors
public class VectorRenderer implements Renderer {
    public void renderCircle(double x, double y, double radius) {
        System.out.printf("Drawing vector circle at (%.1f,%.1f) r=%.1f%n", x, y, radius);
    }
    public void renderRectangle(double x, double y, double w, double h) {
        System.out.printf("Drawing vector rect at (%.1f,%.1f) %s x %s%n", x, y, w, h);
    }
}

public class RasterRenderer implements Renderer {
    public void renderCircle(double x, double y, double radius) {
        System.out.printf("Rasterizing circle at (%.1f,%.1f) r=%.1f%n", x, y, radius);
    }
    public void renderRectangle(double x, double y, double w, double h) {
        System.out.printf("Rasterizing rect at (%.1f,%.1f) %s x %s%n", x, y, w, h);
    }
}

// Abstraction
public abstract class Shape {
    protected final Renderer renderer;  // Bridge to implementation

    protected Shape(Renderer renderer) {
        this.renderer = renderer;
    }

    public abstract void draw();
    public abstract void resize(double factor);
}

// Refined Abstractions
public class Circle extends Shape {
    private double x, y, radius;

    public Circle(Renderer renderer, double x, double y, double radius) {
        super(renderer);
        this.x = x; this.y = y; this.radius = radius;
    }

    public void draw() { renderer.renderCircle(x, y, radius); }
    public void resize(double factor) { radius *= factor; }
}
```

**Key insight:** Bridge is about *preventing* an explosion of subclasses. If you have M abstractions and N implementations, without Bridge you need M*N classes. With Bridge you need M+N.

**When to use:**
- You want to avoid a permanent binding between abstraction and implementation
- Both abstractions and implementations should be extensible via subclassing
- Changes in implementation should not impact client code
- Cross-platform code: one abstraction, multiple platform implementations

**Real-world examples:** Java's JDBC (Connection/Statement are abstractions; MySQL/PostgreSQL drivers are implementations), `java.util.logging` handlers

---

#### 3.2.3 Composite

**Intent:** Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly.

**Problem it solves:** You have a tree structure (file system, DOM, organization chart, bill of materials) and want to treat leaf nodes and composite nodes through the same interface.

**Structure:**
```
Component (interface)
+------------------+
| +operation()     |
| +add(Component)  |
| +remove(Comp.)   |
| +getChild(int)   |
+------------------+
       ^        ^
      /          \
   Leaf        Composite
+----------+   +--------------+
|+operation|   |-children[]   |
+----------+   |+operation()  |
               |+add()        |
               |+remove()     |
               |+getChild()   |
               +--------------+
```

**Implementation (Java — File System):**
```java
public interface FileSystemEntry {
    String getName();
    long getSize();
    void print(String indent);
}

// Leaf
public class File implements FileSystemEntry {
    private final String name;
    private final long size;

    public File(String name, long size) {
        this.name = name; this.size = size;
    }

    public String getName() { return name; }
    public long getSize() { return size; }
    public void print(String indent) {
        System.out.println(indent + name + " (" + size + " bytes)");
    }
}

// Composite
public class Directory implements FileSystemEntry {
    private final String name;
    private final List<FileSystemEntry> children = new ArrayList<>();

    public Directory(String name) { this.name = name; }

    public void add(FileSystemEntry entry) { children.add(entry); }
    public void remove(FileSystemEntry entry) { children.remove(entry); }

    public String getName() { return name; }

    public long getSize() {
        // Recursively sums children — client doesn't need to know if it's a file or dir
        return children.stream().mapToLong(FileSystemEntry::getSize).sum();
    }

    public void print(String indent) {
        System.out.println(indent + name + "/");
        children.forEach(c -> c.print(indent + "  "));
    }
}

// Client treats files and directories identically
FileSystemEntry root = new Directory("root");
Directory src = new Directory("src");
src.add(new File("Main.java", 2048));
src.add(new File("Utils.java", 1024));
root.add(src);
root.add(new File("README.md", 512));

root.print("");       // prints the whole tree
System.out.println("Total: " + root.getSize() + " bytes");
```

**When to use:**
- You want clients to be able to ignore the difference between compositions and individual objects
- Tree structures: DOM, AST, organization charts, GUI component trees
- When operations need to work recursively across the whole structure

**Real-world examples:** Java AWT/Swing component hierarchy, HTML DOM, JSON/XML parsers, `java.awt.Container`

---

#### 3.2.4 Decorator

**Intent:** Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.

**Problem it solves:** You need to add behavior to individual objects at runtime, without affecting other objects of the same class. Subclassing for every combination of behaviors leads to class explosion.

**Structure:**
```
Component (interface)     ConcreteComponent
+------------------+      +------------------+
| +operation()     |      | +operation()     |
+------------------+      +------------------+
         ^
    Decorator (abstract)
    +------------------+
    | -wrapped: Comp.  |
    | +operation()     |  // calls wrapped.operation()
    +------------------+
             ^
    ConcreteDecoratorA   ConcreteDecoratorB
    +------------------+  +------------------+
    | +operation()     |  | +operation()     |
    | +addedBehavior() |  +------------------+
    +------------------+
```

**Implementation (Java — Data pipeline example):**
```java
public interface DataSource {
    void writeData(String data);
    String readData();
}

// ConcreteComponent
public class FileDataSource implements DataSource {
    private final String filename;

    public FileDataSource(String filename) { this.filename = filename; }

    public void writeData(String data) {
        System.out.println("Writing to " + filename + ": " + data);
    }

    public String readData() {
        System.out.println("Reading from " + filename);
        return "raw-data";
    }
}

// Base Decorator
public abstract class DataSourceDecorator implements DataSource {
    protected final DataSource wrappee;

    protected DataSourceDecorator(DataSource source) { this.wrappee = source; }

    public void writeData(String data) { wrappee.writeData(data); }
    public String readData() { return wrappee.readData(); }
}

// Concrete Decorators
public class EncryptionDecorator extends DataSourceDecorator {
    public EncryptionDecorator(DataSource source) { super(source); }

    public void writeData(String data) {
        wrappee.writeData(encrypt(data));
    }
    public String readData() {
        return decrypt(wrappee.readData());
    }
    private String encrypt(String data) { return "[ENCRYPTED:" + data + "]"; }
    private String decrypt(String data) { return data.replace("[ENCRYPTED:", "").replace("]", ""); }
}

public class CompressionDecorator extends DataSourceDecorator {
    public CompressionDecorator(DataSource source) { super(source); }

    public void writeData(String data) {
        wrappee.writeData(compress(data));
    }
    public String readData() {
        return decompress(wrappee.readData());
    }
    private String compress(String data) { return "[COMPRESSED:" + data + "]"; }
    private String decompress(String data) { return data.replace("[COMPRESSED:", "").replace("]", ""); }
}

// Composition at runtime — stacking decorators
DataSource source = new CompressionDecorator(
                        new EncryptionDecorator(
                            new FileDataSource("data.txt")
                        )
                    );
source.writeData("Hello World");
// Writes: [COMPRESSED:[ENCRYPTED:Hello World]]
```

**Decorator vs Inheritance:**
- Inheritance is static — decided at compile time
- Decorator is dynamic — responsibilities added at runtime
- Multiple decorators can be stacked in any order

**When to use:**
- You want to add responsibilities to individual objects without affecting others
- Extension by subclassing is impractical (too many combinations)
- Java I/O: `BufferedReader(new FileReader(...))` is a textbook Decorator

**Real-world examples:** `java.io.BufferedReader`, `java.io.DataInputStream`, Spring security filter chain, logging wrappers

---

#### 3.2.5 Facade

**Intent:** Provide a simplified interface to a complex subsystem.

**Problem it solves:** A subsystem has many classes with complex interdependencies. Client code is tightly coupled to subsystem internals and is difficult to use correctly.

**Structure:**
```
Client ---> Facade ---> SubsystemClass1
                  ---> SubsystemClass2
                  ---> SubsystemClass3
                  ---> SubsystemClass4
```

**Implementation (Java — Home Theater System):**
```java
// Complex subsystem classes
public class Amplifier { public void on() {} public void setVolume(int v) {} }
public class DVDPlayer { public void on() {} public void play(String movie) {} }
public class Projector { public void on() {} public void setInput(String input) {} }
public class TheaterLights { public void dim(int level) {} }
public class Screen { public void down() {} }
public class PopcornPopper { public void on() {} public void pop() {} }

// Facade — simplifies the experience
public class HomeTheaterFacade {
    private final Amplifier amp;
    private final DVDPlayer dvd;
    private final Projector projector;
    private final TheaterLights lights;
    private final Screen screen;
    private final PopcornPopper popper;

    public HomeTheaterFacade(Amplifier amp, DVDPlayer dvd, Projector projector,
                              TheaterLights lights, Screen screen, PopcornPopper popper) {
        this.amp = amp; this.dvd = dvd; this.projector = projector;
        this.lights = lights; this.screen = screen; this.popper = popper;
    }

    public void watchMovie(String movie) {
        System.out.println("Get ready to watch a movie...");
        popper.on();
        popper.pop();
        lights.dim(10);
        screen.down();
        projector.on();
        projector.setInput("DVD");
        amp.on();
        amp.setVolume(5);
        dvd.on();
        dvd.play(movie);
    }

    public void endMovie() {
        System.out.println("Shutting movie theater down...");
        // Turn everything off in the right order
    }
}

// Client — single method call instead of 10+ steps
HomeTheaterFacade theater = new HomeTheaterFacade(amp, dvd, projector, lights, screen, popper);
theater.watchMovie("Inception");
```

**Facade vs Adapter:**
- Facade *simplifies* an interface (wraps multiple classes behind one)
- Adapter *converts* one interface to another (wraps a single class)

**When to use:**
- Providing a simple interface to a complex body of code (layered architecture)
- Reducing dependencies between client code and complex subsystems
- Service layer in web applications hiding JPA/repository complexity

**Real-world examples:** `java.net.URL` (hides TCP/IP, HTTP protocol complexity), SLF4J (facade over Log4j, Logback, JUL), Spring's `JdbcTemplate`

---

#### 3.2.6 Flyweight

**Intent:** Use sharing to support large numbers of fine-grained objects efficiently.

**Problem it solves:** You need to create a very large number of objects (millions), and memory is a concern. Many of these objects share common state.

**Key concept — Intrinsic vs Extrinsic state:**
- **Intrinsic state:** Stored in the flyweight, shared across many objects (e.g., character glyph bitmap)
- **Extrinsic state:** Passed to flyweight methods by the client, unique per context (e.g., x,y position of the character on screen)

**Implementation (Java — Text Rendering):**
```java
// Flyweight — stores only intrinsic state
public class CharacterGlyph {
    private final char character;
    private final String fontFamily;
    private final int fontSize;
    // Expensive: pre-rendered bitmap, font metrics, kerning tables
    private final byte[] bitmap;

    public CharacterGlyph(char character, String fontFamily, int fontSize) {
        this.character = character;
        this.fontFamily = fontFamily;
        this.fontSize = fontSize;
        this.bitmap = loadBitmap(character, fontFamily, fontSize); // expensive
    }

    // Extrinsic state (x, y, color) is passed in — not stored
    public void render(int x, int y, String color) {
        System.out.printf("Rendering '%c' at (%d,%d) in %s%n", character, x, y, color);
    }

    private byte[] loadBitmap(char c, String font, int size) {
        return new byte[size * size * 4]; // RGBA bitmap
    }
}

// Flyweight Factory — manages the shared pool
public class GlyphFactory {
    private final Map<String, CharacterGlyph> cache = new HashMap<>();

    public CharacterGlyph getGlyph(char c, String font, int size) {
        String key = c + ":" + font + ":" + size;
        return cache.computeIfAbsent(key, k -> new CharacterGlyph(c, font, size));
    }

    public int getCacheSize() { return cache.size(); }
}

// Client — rendering 1 million characters, but only unique glyphs are in memory
GlyphFactory factory = new GlyphFactory();
String text = "Hello World ".repeat(100000);   // 1.2M chars
for (int i = 0; i < text.length(); i++) {
    CharacterGlyph glyph = factory.getGlyph(text.charAt(i), "Arial", 12);
    glyph.render(i * 8, 0, "black");           // extrinsic state passed per call
}
System.out.println("Unique glyphs in memory: " + factory.getCacheSize()); // ~12, not 1.2M
```

**When to use:**
- A very large number of objects are consuming too much memory
- Most object state can be made extrinsic
- Applications: game engines (trees, bullets, particles), text editors (character glyphs), network packet headers

**Real-world examples:** Java's `Integer.valueOf(-128 to 127)` cache, `String.intern()`, character glyph caches in text rendering engines

---

#### 3.2.7 Proxy

**Intent:** Provide a surrogate or placeholder for another object to control access to it.

**Problem it solves:** You need to control access to an object — for lazy initialization, access control, logging, caching, or remote access — without changing the real object.

**Types of Proxy:**
| Type | Purpose |
|---|---|
| **Virtual Proxy** | Lazy initialization — create expensive object on demand |
| **Protection Proxy** | Access control — check permissions before forwarding |
| **Remote Proxy** | Network transparency — hides that object is on another machine |
| **Cache Proxy** | Caching — stores results of expensive operations |
| **Logging Proxy** | Audit trail — logs all calls to the real object |

**Implementation (Java — Virtual Proxy with caching):**
```java
public interface ImageLoader {
    void display(int x, int y);
    int getWidth();
    int getHeight();
}

// RealSubject — expensive to create (loads from disk/network)
public class RealImage implements ImageLoader {
    private final String filename;
    private byte[] imageData;
    private int width, height;

    public RealImage(String filename) {
        this.filename = filename;
        loadFromDisk(); // expensive!
    }

    private void loadFromDisk() {
        System.out.println("Loading image from disk: " + filename);
        this.imageData = new byte[1024 * 1024 * 4]; // 4MB RGBA
        this.width = 1024;
        this.height = 1024;
    }

    public void display(int x, int y) {
        System.out.printf("Displaying %s at (%d,%d)%n", filename, x, y);
    }
    public int getWidth() { return width; }
    public int getHeight() { return height; }
}

// Proxy — same interface, controls access
public class ImageProxy implements ImageLoader {
    private final String filename;
    private RealImage realImage;  // null until first use

    public ImageProxy(String filename) {
        this.filename = filename;
        // Does NOT load the image yet — lazy!
    }

    private RealImage getRealImage() {
        if (realImage == null) {
            realImage = new RealImage(filename);  // created on first access
        }
        return realImage;
    }

    public void display(int x, int y) {
        getRealImage().display(x, y);  // load on demand
    }

    // Can return metadata without loading the full image
    public int getWidth() {
        if (realImage != null) return realImage.getWidth();
        return parseWidthFromHeader(filename);  // read just the header
    }

    public int getHeight() {
        if (realImage != null) return realImage.getHeight();
        return parseHeightFromHeader(filename);
    }

    private int parseWidthFromHeader(String file) { return 1024; }
    private int parseHeightFromHeader(String file) { return 1024; }
}

// Client — doesn't know it's dealing with a proxy
ImageLoader img = new ImageProxy("large-photo.jpg");
System.out.println("Width: " + img.getWidth());  // no disk load yet
img.display(0, 0);                                 // disk load happens HERE
img.display(100, 200);                             // no disk load — already loaded
```

**When to use:**
- Lazy initialization of expensive objects
- Access control / permission checks
- Caching results of expensive remote calls
- Java dynamic proxies / AOP (Spring's `@Transactional`, `@Cacheable` are proxy-based)

**Real-world examples:** Spring AOP proxies, `java.lang.reflect.Proxy`, Hibernate lazy-loading proxies, CDN as a network proxy

---

### 3.3 Behavioral Patterns (11)

**Core question:** How do objects communicate and distribute responsibility?

Behavioral patterns concern algorithms and the assignment of responsibilities between objects. They describe patterns of communication between objects.

---

#### 3.3.1 Chain of Responsibility

**Intent:** Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along the chain until an object handles it.

**Implementation (Java — HTTP middleware / request pipeline):**
```java
public abstract class RequestHandler {
    protected RequestHandler next;

    public RequestHandler setNext(RequestHandler next) {
        this.next = next;
        return next;  // fluent API for chaining
    }

    public abstract boolean handle(HttpRequest request);

    protected boolean passToNext(HttpRequest request) {
        if (next != null) return next.handle(request);
        return false; // no handler handled the request
    }
}

public class AuthenticationHandler extends RequestHandler {
    public boolean handle(HttpRequest request) {
        if (request.getHeader("Authorization") == null) {
            System.out.println("Auth failed: missing token");
            return false;
        }
        System.out.println("Auth passed");
        return passToNext(request);
    }
}

public class RateLimitHandler extends RequestHandler {
    private final int maxRpm;
    public RateLimitHandler(int maxRpm) { this.maxRpm = maxRpm; }

    public boolean handle(HttpRequest request) {
        if (isRateLimited(request.getClientIp())) {
            System.out.println("Rate limited: " + request.getClientIp());
            return false;
        }
        return passToNext(request);
    }
    private boolean isRateLimited(String ip) { return false; }
}

public class LoggingHandler extends RequestHandler {
    public boolean handle(HttpRequest request) {
        System.out.println("LOG: " + request.getMethod() + " " + request.getPath());
        return passToNext(request);
    }
}

// Building the chain
RequestHandler chain = new AuthenticationHandler();
chain.setNext(new RateLimitHandler(100))
     .setNext(new LoggingHandler());
```

**When to use:**
- More than one object may handle a request and the handler is not known at compile time
- You want to issue a request to one of several objects without specifying the receiver explicitly
- The set of objects that can handle a request should be specified dynamically

**Real-world examples:** Servlet filters, Spring security filter chain, Express.js/Node middleware, exception handling hierarchy

---

#### 3.3.2 Command

**Intent:** Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations.

**Key players:** Command (interface), ConcreteCommand, Invoker, Receiver

**Implementation (Java — Text Editor with Undo):**
```java
public interface Command {
    void execute();
    void undo();
}

public class TextEditor {
    private final StringBuilder text = new StringBuilder();

    public void insertText(int pos, String str) {
        text.insert(pos, str);
    }
    public void deleteText(int pos, int length) {
        text.delete(pos, pos + length);
    }
    public String getText() { return text.toString(); }
}

public class InsertTextCommand implements Command {
    private final TextEditor editor;
    private final int position;
    private final String text;

    public InsertTextCommand(TextEditor editor, int position, String text) {
        this.editor = editor; this.position = position; this.text = text;
    }

    public void execute() { editor.insertText(position, text); }
    public void undo() { editor.deleteText(position, text.length()); }
}

// Invoker — manages command history
public class CommandHistory {
    private final Deque<Command> history = new ArrayDeque<>();

    public void executeCommand(Command cmd) {
        cmd.execute();
        history.push(cmd);
    }

    public void undo() {
        if (!history.isEmpty()) history.pop().undo();
    }
}
```

**When to use:** Undo/redo, transactional operations, task queues, macro recording

**Real-world examples:** `java.lang.Runnable`, AWT Action, Java EE transactions, database stored procedures

---

#### 3.3.3 Iterator

**Intent:** Provide a way to access elements of a collection sequentially without exposing its underlying representation.

**Implementation (Java — Custom Tree Iterator):**
```java
public interface Iterator<T> {
    boolean hasNext();
    T next();
}

public class TreeNode<T> {
    T value;
    TreeNode<T> left, right;
}

// In-order iterator for BST — client doesn't need to know the tree structure
public class InOrderIterator<T> implements Iterator<T> {
    private final Deque<TreeNode<T>> stack = new ArrayDeque<>();

    public InOrderIterator(TreeNode<T> root) {
        pushLeft(root);
    }

    private void pushLeft(TreeNode<T> node) {
        while (node != null) { stack.push(node); node = node.left; }
    }

    public boolean hasNext() { return !stack.isEmpty(); }

    public T next() {
        TreeNode<T> node = stack.pop();
        pushLeft(node.right);
        return node.value;
    }
}
```

**When to use:**
- You want to provide a way to access an aggregate object's contents without exposing its internal representation
- You want to support multiple simultaneous traversals of aggregate objects
- You want to provide a uniform traversal interface over different aggregate structures

**Real-world examples:** Java's `java.util.Iterator`, Python's `__iter__/__next__`, C++ STL iterators, database cursors

---

#### 3.3.4 Mediator

**Intent:** Define an object that encapsulates how a set of objects interact. Mediator promotes loose coupling by keeping objects from referring to each other explicitly, and it lets you vary their interaction independently.

**Problem it solves:** When many objects communicate with each other in complex ways, the resulting interdependencies can be hard to maintain. Every object needs to know about every other object.

**Implementation (Java — Chat Room):**
```java
public interface ChatMediator {
    void sendMessage(String message, User sender);
    void addUser(User user);
}

public class ChatRoom implements ChatMediator {
    private final List<User> users = new ArrayList<>();

    public void addUser(User user) { users.add(user); }

    public void sendMessage(String message, User sender) {
        users.stream()
             .filter(u -> u != sender)
             .forEach(u -> u.receive(message, sender.getName()));
    }
}

public class User {
    private final String name;
    private final ChatMediator mediator;

    public User(String name, ChatMediator mediator) {
        this.name = name; this.mediator = mediator;
        mediator.addUser(this);
    }

    public void send(String message) { mediator.sendMessage(message, this); }
    public void receive(String message, String from) {
        System.out.println(name + " received from " + from + ": " + message);
    }
    public String getName() { return name; }
}
```

**Mediator vs Facade:**
- Facade is unidirectional — client talks to facade, facade talks to subsystem
- Mediator is bidirectional — colleagues communicate through the mediator, and the mediator coordinates them

**When to use:** When many objects communicate in complex ways, resulting in tight coupling. Air traffic control, chat systems, UI form component coordination, message brokers.

---

#### 3.3.5 Memento

**Intent:** Without violating encapsulation, capture and externalize an object's internal state so that the object can be restored to this state later.

**Implementation (Java — Editor State):**
```java
public class EditorMemento {
    private final String content;
    private final int cursorPosition;
    private final Instant savedAt;

    EditorMemento(String content, int cursorPosition) {
        this.content = content;
        this.cursorPosition = cursorPosition;
        this.savedAt = Instant.now();
    }

    // Package-private — only Editor can access internals
    String getContent() { return content; }
    int getCursorPosition() { return cursorPosition; }
    Instant getSavedAt() { return savedAt; }
}

public class Editor {
    private String content = "";
    private int cursorPosition = 0;

    public void type(String text) {
        content += text;
        cursorPosition += text.length();
    }

    // Creates snapshot — serializes state without exposing it
    public EditorMemento save() {
        return new EditorMemento(content, cursorPosition);
    }

    // Restores from snapshot
    public void restore(EditorMemento memento) {
        content = memento.getContent();
        cursorPosition = memento.getCursorPosition();
    }
}

// Caretaker — holds snapshots, doesn't inspect them
public class History {
    private final Deque<EditorMemento> snapshots = new ArrayDeque<>();

    public void push(EditorMemento memento) { snapshots.push(memento); }
    public EditorMemento pop() { return snapshots.isEmpty() ? null : snapshots.pop(); }
}
```

**When to use:**
- You need to provide an undo mechanism
- You want to snapshot and restore an object's state without exposing its internal structure
- Direct interface to getting the state would expose implementation details and break encapsulation

**Real-world examples:** Git commits, browser back/forward history, game save states, database transaction rollback (SAVEPOINT)

---

#### 3.3.6 Observer

**Intent:** Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.

**Also known as:** Publish-Subscribe, Event Listener, Reactive

**Implementation (Java — Generic Event System):**
```java
public interface EventListener<T> {
    void onEvent(T event);
}

public class EventBus<T> {
    private final List<EventListener<T>> listeners = new CopyOnWriteArrayList<>();

    public void subscribe(EventListener<T> listener) { listeners.add(listener); }
    public void unsubscribe(EventListener<T> listener) { listeners.remove(listener); }

    public void publish(T event) {
        listeners.forEach(l -> l.onEvent(event));
    }
}

// Usage
EventBus<OrderEvent> orderBus = new EventBus<>();
orderBus.subscribe(event -> emailService.sendConfirmation(event.getOrderId()));
orderBus.subscribe(event -> inventoryService.reserve(event.getItems()));
orderBus.subscribe(event -> analyticsService.track("order_placed", event));

orderBus.publish(new OrderEvent(order));
// All three handlers called automatically
```

**When to use:**
- When a change to one object requires changing others, and you don't know how many objects need to change
- When an object should be able to notify other objects without making assumptions about who those objects are
- Implementing distributed event handling systems

**Real-world examples:** `java.util.Observable`, Java Swing listeners, Spring ApplicationEventPublisher, RxJava, Kafka consumers, MVC (model notifies view)

---

#### 3.3.7 State

**Intent:** Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.

**Problem it solves:** Complex state machines with many conditionals scattered throughout the code (`if state == X ... else if state == Y ...`). Transitions become hard to track and test.

**Implementation (Java — Order state machine):**
```java
public interface OrderState {
    void pay(OrderContext ctx);
    void ship(OrderContext ctx);
    void deliver(OrderContext ctx);
    void cancel(OrderContext ctx);
    String getName();
}

public class PendingState implements OrderState {
    public void pay(OrderContext ctx) {
        System.out.println("Payment received");
        ctx.setState(new PaidState());
    }
    public void ship(OrderContext ctx) { throw new IllegalStateException("Must pay first"); }
    public void deliver(OrderContext ctx) { throw new IllegalStateException("Must pay first"); }
    public void cancel(OrderContext ctx) { ctx.setState(new CancelledState()); }
    public String getName() { return "PENDING"; }
}

public class PaidState implements OrderState {
    public void pay(OrderContext ctx) { throw new IllegalStateException("Already paid"); }
    public void ship(OrderContext ctx) {
        System.out.println("Order shipped");
        ctx.setState(new ShippedState());
    }
    public void deliver(OrderContext ctx) { throw new IllegalStateException("Must ship first"); }
    public void cancel(OrderContext ctx) {
        System.out.println("Refund issued");
        ctx.setState(new CancelledState());
    }
    public String getName() { return "PAID"; }
}

// Context — delegates behavior to current state
public class OrderContext {
    private OrderState state = new PendingState();
    private final String orderId;

    public OrderContext(String orderId) { this.orderId = orderId; }

    void setState(OrderState state) { this.state = state; }

    public void pay() { state.pay(this); }
    public void ship() { state.ship(this); }
    public void deliver() { state.deliver(this); }
    public void cancel() { state.cancel(this); }
    public String getStatus() { return state.getName(); }
}
```

**State vs Strategy:**
- State changes itself over the object's lifetime; strategies are typically set once
- State objects may know about and transition to each other; strategy objects are independent
- State controls *what* the object does; Strategy controls *how* a specific operation works

**Real-world examples:** TCP connection states (LISTEN, ESTABLISHED, CLOSE_WAIT), order status machine, traffic light controller, vending machine, game character states

---

#### 3.3.8 Strategy

**Intent:** Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

**Implementation (Java — Sorting with pluggable algorithms):**
```java
public interface SortStrategy<T extends Comparable<T>> {
    void sort(List<T> data);
    String getName();
}

public class QuickSortStrategy<T extends Comparable<T>> implements SortStrategy<T> {
    public void sort(List<T> data) { Collections.sort(data); } // simplified
    public String getName() { return "QuickSort"; }
}

public class BucketSortStrategy implements SortStrategy<Integer> {
    public void sort(List<Integer> data) { /* bucket sort implementation */ }
    public String getName() { return "BucketSort"; }
}

// Context
public class Sorter<T extends Comparable<T>> {
    private SortStrategy<T> strategy;

    public Sorter(SortStrategy<T> strategy) { this.strategy = strategy; }

    public void setStrategy(SortStrategy<T> strategy) { this.strategy = strategy; }

    public void sort(List<T> data) {
        System.out.println("Using " + strategy.getName());
        strategy.sort(data);
    }
}

// Runtime selection
Sorter<Integer> sorter = new Sorter<>(new QuickSortStrategy<>());
sorter.sort(smallList);

sorter.setStrategy(new BucketSortStrategy());
sorter.sort(largeIntegerList);
```

**When to use:**
- Many related classes differ only in their behavior
- You need different variants of an algorithm
- An algorithm uses data that clients shouldn't know about
- A class defines many behaviors as multiple conditional statements — move each branch into its own Strategy

**Real-world examples:** `java.util.Comparator`, Spring's `AuthenticationStrategy`, payment method selection, compression algorithm selection, routing algorithms

---

#### 3.3.9 Template Method

**Intent:** Define the skeleton of an algorithm in an operation, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm's structure.

**Implementation (Java — Data Parser):**
```java
public abstract class DataParser {
    // Template method — defines the algorithm skeleton (final = cannot override)
    public final ParseResult parse(String source) {
        String raw = readData(source);           // step 1
        String[] lines = splitLines(raw);        // step 2
        List<Record> records = processLines(lines);  // step 3 — abstract
        return validate(records);                // step 4
    }

    protected String readData(String source) {
        // default: read from file
        return readFile(source);
    }

    protected String[] splitLines(String data) {
        return data.split("\n");
    }

    // Subclasses MUST implement this
    protected abstract List<Record> processLines(String[] lines);

    // Hook — subclasses MAY override
    protected ParseResult validate(List<Record> records) {
        return new ParseResult(records, Collections.emptyList());
    }

    private String readFile(String path) { return "line1\nline2"; }
}

public class CsvParser extends DataParser {
    protected List<Record> processLines(String[] lines) {
        return Arrays.stream(lines)
                     .map(line -> new Record(line.split(",")))
                     .collect(Collectors.toList());
    }
}

public class JsonParser extends DataParser {
    protected List<Record> processLines(String[] lines) {
        // JSON-specific parsing
        return new ArrayList<>();
    }
}
```

**Template Method vs Strategy:**
- Template Method uses *inheritance* — behavior variation via subclassing; the base class controls the algorithm structure
- Strategy uses *composition* — behavior variation via delegation; the entire algorithm can be swapped at runtime

**When to use:**
- To implement the invariant parts of an algorithm once and leave subclasses to implement the behavior that can vary
- When common behavior among subclasses should be factored and localized in a common class to avoid code duplication
- To control subclass extensions (hooks let subclasses extend behavior at specific points)

**Real-world examples:** `java.io.InputStream.read()`, `AbstractList`, JUnit test lifecycle (`@BeforeEach`, test, `@AfterEach`), Spring's `JdbcTemplate.execute()`

---

#### 3.3.10 Visitor

**Intent:** Represent an operation to be performed on elements of an object structure. Visitor lets you define a new operation without changing the classes of the elements on which it operates.

**Problem it solves:** You have a stable class hierarchy (rarely changes) but frequently need to add new operations over it without modifying every class. Without Visitor, each new operation requires modifying every class in the hierarchy.

**Implementation (Java — AST Visitor):**
```java
// Element hierarchy (stable)
public interface ASTNode {
    <T> T accept(ASTVisitor<T> visitor);
}

public class NumberNode implements ASTNode {
    public final double value;
    public NumberNode(double value) { this.value = value; }
    public <T> T accept(ASTVisitor<T> visitor) { return visitor.visitNumber(this); }
}

public class BinaryOpNode implements ASTNode {
    public final String operator;
    public final ASTNode left, right;
    public BinaryOpNode(String op, ASTNode left, ASTNode right) {
        this.operator = op; this.left = left; this.right = right;
    }
    public <T> T accept(ASTVisitor<T> visitor) { return visitor.visitBinaryOp(this); }
}

// Visitor interface
public interface ASTVisitor<T> {
    T visitNumber(NumberNode node);
    T visitBinaryOp(BinaryOpNode node);
}

// Concrete visitors — new operations without touching AST nodes
public class EvaluatorVisitor implements ASTVisitor<Double> {
    public Double visitNumber(NumberNode node) { return node.value; }
    public Double visitBinaryOp(BinaryOpNode node) {
        double left = node.left.accept(this);
        double right = node.right.accept(this);
        return switch (node.operator) {
            case "+" -> left + right;
            case "-" -> left - right;
            case "*" -> left * right;
            case "/" -> left / right;
            default -> throw new IllegalArgumentException("Unknown op: " + node.operator);
        };
    }
}

public class PrettyPrinterVisitor implements ASTVisitor<String> {
    public String visitNumber(NumberNode node) { return String.valueOf(node.value); }
    public String visitBinaryOp(BinaryOpNode node) {
        return "(" + node.left.accept(this) + " " + node.operator + " " + node.right.accept(this) + ")";
    }
}

// Usage
ASTNode ast = new BinaryOpNode("+",
    new NumberNode(3),
    new BinaryOpNode("*", new NumberNode(4), new NumberNode(5))
);

System.out.println(ast.accept(new PrettyPrinterVisitor())); // (3.0 + (4.0 * 5.0))
System.out.println(ast.accept(new EvaluatorVisitor()));      // 23.0
```

**Important trade-off:** Visitor makes it easy to add new operations but hard to add new element types. Adding a new node type requires updating every existing Visitor. Use Visitor only when the element hierarchy is stable.

**Real-world examples:** Java compiler AST visitors, XML DOM traversal, static analysis tools (checkstyle, SpotBugs)

---

#### 3.3.11 Interpreter

**Intent:** Given a language, define a representation for its grammar along with an interpreter that uses the representation to interpret sentences in the language.

**Implementation (Java — Boolean Expression Evaluator):**
```java
public interface BooleanExpression {
    boolean interpret(Map<String, Boolean> context);
}

public class VariableExpression implements BooleanExpression {
    private final String name;
    public VariableExpression(String name) { this.name = name; }
    public boolean interpret(Map<String, Boolean> context) {
        return context.getOrDefault(name, false);
    }
}

public class AndExpression implements BooleanExpression {
    private final BooleanExpression left, right;
    public AndExpression(BooleanExpression left, BooleanExpression right) {
        this.left = left; this.right = right;
    }
    public boolean interpret(Map<String, Boolean> context) {
        return left.interpret(context) && right.interpret(context);
    }
}

public class OrExpression implements BooleanExpression {
    private final BooleanExpression left, right;
    public OrExpression(BooleanExpression left, BooleanExpression right) {
        this.left = left; this.right = right;
    }
    public boolean interpret(Map<String, Boolean> context) {
        return left.interpret(context) || right.interpret(context);
    }
}

// Usage: (A AND B) OR C
BooleanExpression expr = new OrExpression(
    new AndExpression(new VariableExpression("A"), new VariableExpression("B")),
    new VariableExpression("C")
);
Map<String, Boolean> ctx = Map.of("A", true, "B", false, "C", true);
System.out.println(expr.interpret(ctx)); // true
```

**Note:** Interpreter is rarely used in practice for large or complex grammars — use a parser generator (ANTLR) instead. It's most useful for small, specialized languages where the grammar is simple and performance is not critical.

**When to use:**
- When you need to interpret sentences in a simple language
- When the grammar is simple (complex grammars make the class hierarchy unmanageable)
- Efficiency is not a critical concern

**Real-world examples:** SQL query engines, regular expression engines, configuration DSLs, Spring EL (SpEL), feature flag rule evaluation

---

## 4. Patterns vs Over-Engineering

### The Most Important Warning in This Document

Design patterns are **tools**, not goals. Applying a pattern where it is not needed is **worse** than not knowing the pattern at all. It introduces:

- Unnecessary complexity
- More files and classes to navigate
- Harder debugging (more indirection layers)
- Slower onboarding for new team members
- The "astronaut architect" reputation

### Signs You Are Over-Engineering

1. **"We might need this later"** — YAGNI (You Ain't Gonna Need It). Patterns should solve problems you have today.
2. **The pattern introduces more code than it removes.** If you needed 5 lines without the pattern and 50 lines with it, reconsider.
3. **You cannot explain WHY the pattern is needed in one sentence.**
4. **You are applying a pattern to a class that has only one implementation and one caller.**
5. **Junior engineers cannot understand the code without reading the GoF book first.**

### Simple Decision Framework

```
Do I have a real problem right now?
     |
    YES
     |
Is the problem caused by one of the forces a pattern addresses
(e.g., class explosion, tight coupling, scattered algorithm, repeated creation)?
     |
    YES
     |
Does applying the pattern make the code more readable and maintainable,
or just more abstract?
     |
  MORE READABLE
     |
Apply the pattern.
```

### The Rule of Three

> "The first time you do something, you just do it.
> The second time you do something similar, you wince at the duplication, but you do the duplicate thing anyway.
> The third time you do something similar, you refactor."
>
> — Martin Fowler, "Refactoring"

Patterns often emerge through refactoring, not upfront design. Write the simple code first. When you feel pain (duplicated logic, cascading changes, impossible tests), that is when you reach for a pattern.

### Patterns vs Language Features in Modern Languages

Some GoF patterns address problems that modern language features solve more elegantly:

| Pattern | Modern Alternative |
|---|---|
| Strategy | Lambda / first-class function (`Comparator.comparing(...)`) |
| Command | Lambda / `Runnable` / `Callable` |
| Observer | Reactive streams (RxJava, Project Reactor), Java `Flow` API |
| Iterator | Language-native iteration (`for-each`, generators, streams) |
| Singleton | DI container (Spring `@Bean` singleton scope, Guice) |
| Template Method | Higher-order functions passed as parameters |

This does not mean the patterns are dead — it means you can implement them more concisely in modern languages.

---

## 5. Pattern Selection Guide

Use this table when you have a problem and need to identify the right pattern:

| Problem You Are Facing | Pattern to Consider | Category |
|---|---|---|
| Need exactly one instance of a class across the system | Singleton | Creational |
| Subclasses should decide which object to instantiate | Factory Method | Creational |
| Need to create families of related objects consistently | Abstract Factory | Creational |
| Object has many optional parameters; telescoping constructor antipattern | Builder | Creational |
| Object creation is expensive; need many similar objects | Prototype | Creational |
| Third-party / legacy interface doesn't match what you need | Adapter | Structural |
| Two hierarchies evolving independently; combinatorial class explosion | Bridge | Structural |
| Tree structure where leaf and composite must be treated uniformly | Composite | Structural |
| Add behavior to objects dynamically at runtime | Decorator | Structural |
| Complex subsystem needs a simpler unified interface | Facade | Structural |
| Very large number of fine-grained objects consuming too much memory | Flyweight | Structural |
| Control access to object (lazy init, access control, caching, logging) | Proxy | Structural |
| Request should pass through a chain of potential handlers | Chain of Responsibility | Behavioral |
| Need to queue, log, or undo operations | Command | Behavioral |
| Need to traverse a collection without exposing its structure | Iterator | Behavioral |
| Many objects communicate in complex ways; need to reduce coupling | Mediator | Behavioral |
| Need to snapshot and restore object state without violating encapsulation | Memento | Behavioral |
| One-to-many dependency; state change should notify dependents | Observer | Behavioral |
| Object's behavior changes based on its internal state | State | Behavioral |
| Family of algorithms; make them interchangeable at runtime | Strategy | Behavioral |
| Algorithm has invariant skeleton; steps vary across subclasses | Template Method | Behavioral |
| New operations on stable object structure without modifying it | Visitor | Behavioral |
| Small language or grammar that needs to be interpreted | Interpreter | Behavioral |

### Symptom-to-Pattern Diagnosis

```
SYMPTOM                                         LIKELY PATTERN(S)
---------------------------------------------------------------------------
"I need only one of X across the whole app"  -> Singleton
"My constructor has 10+ parameters"          -> Builder
"I need different types of X at runtime"     -> Factory Method / Abstract Factory
"These two hierarchies are too coupled"      -> Bridge
"These subsystem classes are too complex     -> Facade
  for clients to use directly"
"I need to add features without modifying    -> Decorator, Strategy, Observer
  existing classes"
"My if/else chain for state is getting huge" -> State
"My if/else for algorithm selection is huge" -> Strategy
"I need undo functionality"                  -> Command, Memento
"Changes in one object need to propagate"    -> Observer
"I have tree-structured data"                -> Composite
"I keep instantiating concrete classes       -> Factory Method, Abstract Factory
  directly in client code"
"I'm accessing a remote/expensive/guarded    -> Proxy
  object directly"
"I keep copying the same algorithm with      -> Template Method
  small variations"
"I need to add operations to a class         -> Visitor
  hierarchy I can't modify"
```

---

## 6. Patterns and SOLID Principles

SOLID and design patterns are deeply interrelated. Patterns are the *implementations* of SOLID principles in specific contexts.

### S — Single Responsibility Principle

> A class should have only one reason to change.

**Patterns that enforce SRP:**
- **Command** — encapsulates a single action, separating it from the invoker and receiver. Each command class has one reason to change: the action it represents.
- **Strategy** — isolates each algorithm into its own class. Adding a new algorithm does not change existing strategy classes.
- **Template Method** — separates the invariant algorithm structure (in the base class) from the variant steps (in subclasses). Each subclass has one responsibility: implementing its variation.
- **Facade** — separates subsystem coordination from business logic. The Facade class has one responsibility: providing a simplified API.
- **Iterator** — separates traversal logic from the collection class. The collection class is responsible for storing data; the iterator is responsible for traversal.

**Example of SRP violation that a pattern fixes:**
Without Command pattern, `OrderController` might handle HTTP parsing, payment processing, inventory updates, and email sending — five unrelated responsibilities, five reasons to change. With Command objects, each operation gets its own class.

---

### O — Open/Closed Principle

> Software entities should be open for extension, but closed for modification.

**Patterns that enforce OCP:**
- **Strategy** — add new algorithms without changing the context class. The context is open for extension (new strategies) but closed for modification.
- **Decorator** — add new behavior by wrapping, not modifying existing classes. The wrapped class never changes.
- **Observer** — add new subscribers without changing the publisher. The publisher is closed for modification.
- **Factory Method / Abstract Factory** — add new product types by adding new factory subclasses, without modifying the existing factory hierarchy.
- **Visitor** — add new operations over an existing hierarchy without modifying any element classes.
- **Chain of Responsibility** — add new handlers without changing existing ones or the client.

**Example:** A payment system using Strategy pattern lets you add `CryptoPaymentStrategy` without touching `PaymentContext`. Without Strategy, you'd add an `else if` branch directly in `processPayment()` — modifying existing, tested code.

---

### L — Liskov Substitution Principle

> Objects of a subtype must be substitutable for objects of their supertypes without altering the correctness of the program.

**Patterns that enforce LSP:**
- **Template Method** — subclasses override *steps*, not the algorithm contract. The base class guarantees the invariant.
- **Strategy** — all ConcreteStrategy classes must honor the `Strategy` interface contract exactly. The Context must not behave differently based on which concrete strategy it receives.
- **Composite** — Leaf and Composite must be genuinely interchangeable through the Component interface.
- **State** — all ConcreteState classes must handle all state transitions defined by the interface.

**LSP Violation to watch for in Composite:** When a Leaf is forced to implement `add()` and `remove()` from the Component interface but throws `UnsupportedOperationException`, it violates LSP. The fix: design the Component interface carefully, or use a two-interface design (safe Composite).

---

### I — Interface Segregation Principle

> Clients should not be forced to depend upon interfaces they do not use.

**Patterns that enforce ISP:**
- **Facade** — clients use only the simplified interface, not all subsystem interfaces. A subsystem with 20 methods is hidden behind a Facade with 3.
- **Proxy** — clients depend on the minimal `Subject` interface, not the `RealSubject`'s full surface area.
- **Adapter** — wraps a fat third-party interface behind the thin, targeted interface the client needs.
- **Iterator** — clients only need `hasNext()` and `next()`, not the full collection interface with all its mutation methods.

---

### D — Dependency Inversion Principle

> High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details. Details should depend on abstractions.

**Patterns that enforce DIP:**
- **Abstract Factory** — high-level code depends on `UIFactory` interface, not `WindowsUIFactory` or `MacUIFactory`.
- **Factory Method** — creator depends on the product interface, not concrete products.
- **Strategy** — context depends on `Strategy` interface, not concrete algorithms.
- **Observer** — subject depends on `Observer` interface, not concrete listeners.
- **Bridge** — abstraction depends on `Implementor` interface, not concrete implementations.
- **Proxy** — client depends on `Subject` interface, not `RealSubject`.

### Pattern to SOLID Principle Mapping

| Pattern | SRP | OCP | LSP | ISP | DIP |
|---|---|---|---|---|---|
| Singleton | - | - | - | - | - |
| Factory Method | - | YES | - | - | YES |
| Abstract Factory | - | YES | - | YES | YES |
| Builder | YES | - | - | - | - |
| Prototype | - | - | - | - | - |
| Adapter | - | - | - | YES | YES |
| Bridge | - | YES | - | - | YES |
| Composite | - | YES | YES | - | YES |
| Decorator | YES | YES | YES | - | YES |
| Facade | YES | - | - | YES | YES |
| Flyweight | - | - | - | - | - |
| Proxy | - | YES | YES | - | YES |
| Chain of Responsibility | YES | YES | - | - | - |
| Command | YES | YES | - | - | - |
| Iterator | YES | - | - | YES | YES |
| Mediator | YES | - | - | - | - |
| Memento | YES | - | - | - | - |
| Observer | YES | YES | - | - | YES |
| State | - | YES | YES | - | - |
| Strategy | YES | YES | YES | - | YES |
| Template Method | - | YES | YES | - | - |
| Visitor | YES | YES | - | - | - |
| Interpreter | - | YES | YES | - | - |

---

## 7. How to Study Patterns Effectively

### The Three-Phase Study Approach

**Phase 1: Problem first, pattern second (Weeks 1–2)**

Do NOT memorize patterns. For each pattern:
1. Read only the "Problem it solves" section
2. Try to design your own solution without looking at the pattern
3. Then compare your solution with the pattern
4. Ask: what does the pattern give you that your solution didn't?

**Phase 2: Implement every pattern from memory (Weeks 3–4)**

For each of the 23 patterns:
1. Write the core structure from memory (no notes)
2. Write a real-world example — your own, not the textbook one
3. Write one unit test that proves the pattern works
4. Write one example of where this pattern would be wrong to apply

**Phase 3: Pattern recognition in existing code (Weeks 5–6)**

Open a major open-source codebase (Spring Framework, Apache Commons, JDK source):
1. Find examples of each pattern in real production code
2. Note how the production version differs from the textbook version
3. Understand why the pattern was chosen in that specific context

### Find Patterns in Real Codebases

| Pattern | Where to find it |
|---|---|
| Singleton | `java.lang.Runtime.getRuntime()`, Spring beans |
| Factory Method | JDBC `DriverManager`, JUnit test lifecycle |
| Builder | `StringBuilder`, `OkHttpClient.Builder`, Protobuf |
| Decorator | `java.io.BufferedReader(new FileReader(...))` |
| Observer | Spring `ApplicationEventPublisher`, Swing listeners |
| Strategy | `java.util.Comparator` with `Collections.sort()` |
| Proxy | Spring `@Transactional` — enable and inspect the proxy class |
| Composite | `java.awt.Container` and its children |
| Chain of Responsibility | Spring Security filter chain |
| Template Method | `AbstractList`, Spring `JdbcTemplate` |

### Recommended Study Order

Study in this sequence — each pattern builds conceptual foundations for the next:

1. **Strategy** — easiest Behavioral; teaches delegation as a design tool
2. **Observer** — second most common in interviews; teaches publish-subscribe
3. **Factory Method** — understand before Abstract Factory
4. **Builder** — immediately useful; eliminates telescoping constructors
5. **Decorator** — teaches wrapping; prerequisite for understanding Java I/O
6. **Singleton** — know it well including thread safety and its pitfalls
7. **Adapter** — immediately practical; every integration project needs it
8. **Facade** — immediately practical; understand it as a layering tool
9. **Proxy** — prerequisite for understanding Spring AOP
10. **Abstract Factory** — now Factory Method is solid ground
11. **Composite** — tree structures appear in every domain
12. **Command** — undo/redo, task queues, the Command Bus pattern
13. **State** — replaces complex if/else state machines
14. **Template Method** — after Strategy; compare the two
15. **Chain of Responsibility** — middleware and filter pipelines
16. **Prototype** — object cloning, less common but appears in interviews
17. **Bridge** — hardest Structural; study after Adapter and Decorator
18. **Iterator** — Java makes this trivial; understand the concept
19. **Mediator** — deferred for later; understand Facade first
20. **Memento** — pairs well with Command for undo/redo
21. **Flyweight** — memory optimization; appears in game dev interviews
22. **Visitor** — hardest Behavioral; requires solid grasp of polymorphism
23. **Interpreter** — know it exists and why; rarely implement from scratch

### How to Answer Pattern Questions in Interviews

When asked "Which pattern would you use here?", structure your answer in five steps:

1. **State the problem** — "The problem here is that we have multiple notification channels, and we need to add new ones without changing the sender..."
2. **Name the pattern** — "I would use the Observer pattern..."
3. **Name the SOLID principles it upholds** — "This respects OCP because adding a new notification channel doesn't require modifying the publisher..."
4. **Describe the trade-offs** — "The downside is that if many observers subscribe, notification can become slow. We'd want to consider async notification for high-throughput systems..."
5. **State when you would NOT use it** — "If we only ever have one notification channel and that's unlikely to change, I'd just call it directly and avoid the indirection."

### Common Pattern Confusions to Resolve

Knowing how to distinguish these commonly confused pairs signals senior-level understanding:

| Confused Pair | The Key Distinguishing Difference |
|---|---|
| Strategy vs State | Strategy is set by the client and stays stable; State transitions internally during the object's lifetime |
| Decorator vs Proxy | Decorator adds new capabilities; Proxy controls access to existing capabilities |
| Facade vs Adapter | Facade simplifies (one facade, many subsystem classes); Adapter translates (one adapter, one adaptee) |
| Factory Method vs Abstract Factory | Factory Method creates one product type; Abstract Factory creates families of related products |
| Composite vs Decorator | Composite composes a tree (one-to-many children); Decorator wraps a single component |
| Observer vs Mediator | In Observer, the subject broadcasts and has no knowledge of what observers do; in Mediator, all communication goes through the mediator which can coordinate complex interactions |
| Template Method vs Strategy | Template Method uses inheritance (structure fixed in base class); Strategy uses composition (algorithm swappable at runtime) |
| Command vs Strategy | Command encapsulates a request with history (undo); Strategy encapsulates an algorithm selection (no history) |

---

## 8. Quick Reference — All 23 GoF Patterns

| # | Pattern | Category | Intent (One Line) | Complexity | Real-World Analog |
|---|---|---|---|---|---|
| 1 | **Singleton** | Creational | Ensure one instance, global access point | Low | Database connection pool manager |
| 2 | **Factory Method** | Creational | Let subclasses decide which class to instantiate | Medium | Plug-and-play object creation |
| 3 | **Abstract Factory** | Creational | Create families of related objects without specifying classes | Medium-High | Cross-platform UI toolkit |
| 4 | **Builder** | Creational | Separate complex object construction from its representation | Medium | SQL query builder, HTTP client config |
| 5 | **Prototype** | Creational | Create objects by cloning an existing instance | Low-Medium | Copy-on-write, game entity templates |
| 6 | **Adapter** | Structural | Convert one interface to another expected by clients | Low | Power plug adapter, JDBC drivers |
| 7 | **Bridge** | Structural | Decouple abstraction from implementation; both vary independently | Medium-High | Remote controls and devices |
| 8 | **Composite** | Structural | Treat individual objects and compositions uniformly | Medium | File system, HTML DOM, org charts |
| 9 | **Decorator** | Structural | Attach responsibilities to objects dynamically | Medium | Java I/O streams, middleware wrappers |
| 10 | **Facade** | Structural | Simplified interface to a complex subsystem | Low | SLF4J over logging backends |
| 11 | **Flyweight** | Structural | Share fine-grained objects to reduce memory | High | Character glyphs, game particles |
| 12 | **Proxy** | Structural | Surrogate for another object to control access | Medium | Spring AOP, lazy-loading, CDN |
| 13 | **Chain of Responsibility** | Behavioral | Pass request along chain until a handler handles it | Medium | HTTP middleware, exception handlers |
| 14 | **Command** | Behavioral | Encapsulate request as an object; supports undo/queue | Medium | GUI actions, task queues, macros |
| 15 | **Iterator** | Behavioral | Sequential access without exposing internal structure | Low | `for-each` loops, database cursors |
| 16 | **Mediator** | Behavioral | Centralize complex communication between objects | Medium | Chat room, air traffic control |
| 17 | **Memento** | Behavioral | Snapshot and restore object state without violating encapsulation | Medium | Undo history, Git commits |
| 18 | **Observer** | Behavioral | Notify dependents automatically when state changes | Low-Medium | Event systems, MVC, Kafka consumers |
| 19 | **State** | Behavioral | Alter behavior when internal state changes | Medium | Order status machine, TCP connection |
| 20 | **Strategy** | Behavioral | Interchangeable family of algorithms | Low | Sorting, payment methods, compression |
| 21 | **Template Method** | Behavioral | Define algorithm skeleton; defer steps to subclasses | Low | JUnit lifecycle, Spring JdbcTemplate |
| 22 | **Visitor** | Behavioral | New operation on stable hierarchy without modifying it | High | AST traversal, tax calculation engines |
| 23 | **Interpreter** | Behavioral | Grammar-based interpreter for a specialized language | High | SQL engines, regex, SpEL |

### Complexity Scale

- **Low:** Can be explained in under 5 minutes, straightforward to implement
- **Low-Medium:** Requires understanding of delegation or basic polymorphism
- **Medium:** Requires understanding of delegation, polymorphism, or class hierarchies — takes practice
- **Medium-High:** Multiple interacting roles; wrong application creates bugs
- **High:** Requires careful design upfront; wrong application creates hard-to-debug indirection

### Pattern Frequency in Industry Code

```
VERY COMMON (see weekly):
  Singleton, Factory Method, Builder, Observer, Strategy, Proxy, Decorator, Facade

COMMON (see monthly):
  Abstract Factory, Adapter, Composite, Command, Template Method, State

LESS COMMON (see quarterly or in specific domains):
  Chain of Responsibility, Iterator, Mediator, Memento, Bridge, Flyweight, Prototype

RARE (specialized use cases):
  Visitor, Interpreter
```

---

## Further Reading

- **Design Patterns: Elements of Reusable Object-Oriented Software** — Gamma, Helm, Johnson, Vlissides (1994)
  The original source. Chapter 1 is worth re-reading every year.
- **Head First Design Patterns** — Freeman, Freeman, Sierra, Bates (2004)
  Best introductory book. Extremely readable, highly recommended.
- **Refactoring: Improving the Design of Existing Code** — Martin Fowler (2018, 2nd edition)
  Patterns emerge through refactoring. Read this to understand the *when*.
- **Clean Code** — Robert C. Martin (2008)
  SOLID principles and why they matter before you apply patterns.
- **Patterns of Enterprise Application Architecture** — Martin Fowler (2002)
  Bridges GoF patterns to distributed system concerns.
- **refactoring.guru/design-patterns** — Free, excellent visual explanations of all 23 patterns with multi-language code examples.

---

*This document is part of the System Design and LLD Crash Course.*
*Directory: `04-design-patterns/`*
*Last updated: 2026-06-22*
