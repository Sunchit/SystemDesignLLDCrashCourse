# Decorator Design Pattern

> **Category:** Structural | **Difficulty:** Medium | **GoF Pattern:** Yes

---

## Table of Contents

1. [Intent](#1-intent)
2. [When to Use / When NOT to Use](#2-when-to-use--when-not-to-use)
3. [UML Class Diagram](#3-uml-class-diagram)
4. [Complete Java Implementations](#4-complete-java-implementations)
   - [4.1 Coffee Shop Example](#41-coffee-shop-example)
   - [4.2 Java I/O Streams Replication](#42-java-io-streams-replication)
   - [4.3 HTTP Middleware / Filter Chain](#43-http-middleware--filter-chain)
5. [Decorator Chain and Ordering](#5-decorator-chain-and-ordering)
6. [Real-World Analogy](#6-real-world-analogy)
7. [Real-World Java Ecosystem Examples](#7-real-world-java-ecosystem-examples)
8. [Decorator vs Inheritance](#8-decorator-vs-inheritance)
9. [Decorator vs Strategy Pattern](#9-decorator-vs-strategy-pattern)
10. [Trade-offs](#10-trade-offs)
11. [Common Pitfalls and Gotchas](#11-common-pitfalls-and-gotchas)
12. [Interview Discussion Points](#12-interview-discussion-points)
13. [Interview Questions and Model Answers](#13-interview-questions-and-model-answers)

---

## 1. Intent

The **Decorator** pattern attaches additional responsibilities to an object **dynamically** and **transparently** — without modifying the object's class and without requiring knowledge from the object's clients.

### Core Problem Solved

Consider a text editor that must support bold, italic, underlined, and colored text. If you use inheritance:

- `BoldText extends Text`
- `ItalicText extends Text`
- `BoldItalicText extends Text`
- `BoldItalicUnderlinedText extends Text`
- ...and so on

With just 4 formatting options, you need **2^4 = 16 subclasses** to cover every combination. This is the **combinatorial explosion** problem. Decorator solves this by composing behavior at runtime instead of baking it into the class hierarchy at compile time.

### Open/Closed Principle

The Decorator pattern is one of the clearest embodiments of the **Open/Closed Principle** (OCP), the second letter of SOLID:

> *"Software entities should be open for extension, but closed for modification."*

With Decorator:
- **Open for extension:** New behaviors are added by writing new decorator classes.
- **Closed for modification:** Existing concrete components and decorators are never changed to add new behavior.

You extend the system by wrapping objects in new layers, not by touching what is already working and tested.

### Formal Definition (GoF)

> "Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality."
> — *Design Patterns: Elements of Reusable Object-Oriented Software*, Gang of Four

### Key Structural Insight

The decorator **wraps** the component it decorates and **conforms to the same interface**. This conformance is the secret: any code expecting a `Component` will work identically with a raw `ConcreteComponent` or with a `ConcreteComponent` wrapped in ten layers of decorators. The wrapping is completely transparent to the caller.

---

## 2. When to Use / When NOT to Use

### When to Use

| Situation | Reasoning |
|---|---|
| Behavior must be added at runtime, not compile time | Inheritance is a compile-time decision; Decorator is a runtime decision |
| You need many independent optional features that can be mixed and matched | Each feature becomes a decorator; clients compose them freely |
| Subclassing would cause a combinatorial explosion | N features = 2^N subclasses with inheritance; N features = N decorator classes with Decorator |
| You cannot modify the class being extended (third-party code, legacy code) | Wrap the object you cannot touch |
| You need to add behavior to some instances of a class, not all | Wrap only the instances that need extra behavior |
| Responsibilities should be added and removed at runtime | Stack and unstack decorators dynamically |

### When NOT to Use

| Situation | Reasoning |
|---|---|
| The component interface is very large | Every decorator must implement every method; this becomes a maintenance burden |
| Order of decoration does not matter and only one "layer" is ever added | Simple subclassing is clearer and less code |
| You need to introspect the decorator chain | Peeling back layers of decoration to inspect inner components is awkward and fragile |
| The feature you're adding is always present (not optional) | Bake it into the component directly; decoration is overhead for a mandatory feature |
| Type identity matters to clients | A decorated `Coffee` is not `instanceof Espresso`; if clients rely on concrete types via `instanceof`, Decorator breaks that |
| Performance is critical in a tight loop | Each decorator adds an extra method dispatch; many layers add up |

---

## 3. UML Class Diagram

### Generic Pattern Structure

```
         ┌─────────────────────────────────────────────────────────┐
         │                   <<interface>>                          │
         │                    Component                             │
         │─────────────────────────────────────────────────────────│
         │  + operation() : ResultType                              │
         └─────────────────────────────────────────────────────────┘
                    ▲                          ▲
                    │ implements               │ implements
                    │                          │
    ┌───────────────────────────┐   ┌──────────────────────────────────┐
    │     ConcreteComponent     │   │         Decorator                │
    │───────────────────────────│   │──────────────────────────────────│
    │  + operation() : Result   │   │  - wrappee: Component            │
    └───────────────────────────┘   │──────────────────────────────────│
                                    │  + Decorator(c: Component)       │
                                    │  + operation() : ResultType      │
                                    │    → delegates to wrappee        │
                                    └──────────────────────────────────┘
                                                   ▲
                                    ┌──────────────┴────────────────┐
                                    │                               │
                         ┌──────────────────┐           ┌──────────────────┐
                         │ConcreteDecoratorA│           │ConcreteDecoratorB│
                         │──────────────────│           │──────────────────│
                         │+ operation()     │           │+ operation()     │
                         │  super.op() +    │           │  addedBehavior()+│
                         │  extraBehavior() │           │  super.op()      │
                         └──────────────────┘           └──────────────────┘
```

### Coffee Shop Specific Diagram

```
         ┌──────────────────────────────────────┐
         │           <<interface>>              │
         │              Beverage                │
         │──────────────────────────────────────│
         │  + getDescription() : String         │
         │  + cost() : double                   │
         └──────────────────────────────────────┘
                    ▲                  ▲
         ┌──────────┘                  └──────────────────────────┐
         │                                                         │
┌────────────────────┐             ┌───────────────────────────────────────┐
│     Espresso       │             │        CondimentDecorator             │
│────────────────────│             │───────────────────────────────────────│
│ cost(): 1.99       │             │  # beverage: Beverage                 │
│ getDescription()   │             │───────────────────────────────────────│
│   "Espresso"       │             │  + CondimentDecorator(b: Beverage)    │
└────────────────────┘             │  + getDescription(): String           │
                                   │  + cost(): double                     │
┌────────────────────┐             └───────────────────────────────────────┘
│    HouseBlend      │                              ▲
│────────────────────│              ┌───────────────┼───────────────┐
│ cost(): 0.89       │              │               │               │
│ getDescription()   │        ┌─────────┐    ┌──────────┐    ┌──────────┐
│  "House Blend"     │        │  Milk   │    │  Mocha   │    │  Whip    │
└────────────────────┘        │─────────│    │──────────│    │──────────│
                              │+0.25    │    │+0.20     │    │+0.10     │
                              └─────────┘    └──────────┘    └──────────┘
```

---

## 4. Complete Java Implementations

### 4.1 Coffee Shop Example

This is the canonical teaching example from *Head First Design Patterns*. It is extended here with proper Javadoc, generics-aware patterns, and a main driver that illustrates chaining.

```java
package patterns.structural.decorator.coffee;

/**
 * Component interface that both concrete beverages and all condiment
 * decorators must implement. Keeping this interface narrow (two methods)
 * minimises the burden on every decorator implementation.
 */
public interface Beverage {

    /**
     * Returns a human-readable description of this beverage including
     * all condiments added to it.
     *
     * @return non-null description string
     */
    String getDescription();

    /**
     * Calculates the total cost of this beverage including all condiments.
     *
     * @return cost in USD, always &gt;= 0
     */
    double cost();
}
```

```java
package patterns.structural.decorator.coffee;

/**
 * ConcreteComponent: A single shot of espresso. No condiments,
 * just pure concentrated coffee.
 */
public class Espresso implements Beverage {

    @Override
    public String getDescription() {
        return "Espresso";
    }

    @Override
    public double cost() {
        return 1.99;
    }
}
```

```java
package patterns.structural.decorator.coffee;

/**
 * ConcreteComponent: House blend coffee — the budget-friendly base beverage.
 */
public class HouseBlend implements Beverage {

    @Override
    public String getDescription() {
        return "House Blend Coffee";
    }

    @Override
    public double cost() {
        return 0.89;
    }
}
```

```java
package patterns.structural.decorator.coffee;

/**
 * ConcreteComponent: Dark roast coffee — bold, full-bodied.
 */
public class DarkRoast implements Beverage {

    @Override
    public String getDescription() {
        return "Dark Roast Coffee";
    }

    @Override
    public double cost() {
        return 0.99;
    }
}
```

```java
package patterns.structural.decorator.coffee;

/**
 * Abstract Decorator base class. This class is the heart of the pattern:
 * <ul>
 *   <li>It implements {@link Beverage} so it can stand in anywhere a
 *       {@code Beverage} is expected (Liskov Substitution Principle).</li>
 *   <li>It holds a reference to the {@code Beverage} it wraps.</li>
 *   <li>It delegates both methods to the wrapped beverage by default,
 *       allowing concrete decorators to selectively override just the
 *       behaviour they need to change.</li>
 * </ul>
 *
 * <p>Note: This class is {@code abstract} to prevent direct instantiation
 * — you always want a concrete decorator, not a bare wrapper.</p>
 */
public abstract class CondimentDecorator implements Beverage {

    /** The beverage being decorated. Visible to subclasses for chaining. */
    protected final Beverage beverage;

    /**
     * Constructs a new condiment decorator wrapping the given beverage.
     *
     * @param beverage the beverage to wrap; must not be {@code null}
     * @throws IllegalArgumentException if beverage is null
     */
    protected CondimentDecorator(Beverage beverage) {
        if (beverage == null) {
            throw new IllegalArgumentException("Cannot decorate a null beverage");
        }
        this.beverage = beverage;
    }

    /**
     * Default delegation — subclasses typically call {@code beverage.getDescription()}
     * and append their own name.
     */
    @Override
    public String getDescription() {
        return beverage.getDescription();
    }
}
```

```java
package patterns.structural.decorator.coffee;

/**
 * ConcreteDecorator: Adds steamed milk to the beverage.
 * Milk adds $0.25 to the total cost.
 */
public class Milk extends CondimentDecorator {

    private static final double MILK_COST = 0.25;

    public Milk(Beverage beverage) {
        super(beverage);
    }

    @Override
    public String getDescription() {
        return beverage.getDescription() + ", Milk";
    }

    @Override
    public double cost() {
        return MILK_COST + beverage.cost();
    }
}
```

```java
package patterns.structural.decorator.coffee;

/**
 * ConcreteDecorator: Adds chocolate mocha syrup to the beverage.
 * Mocha adds $0.20 to the total cost.
 */
public class Mocha extends CondimentDecorator {

    private static final double MOCHA_COST = 0.20;

    public Mocha(Beverage beverage) {
        super(beverage);
    }

    @Override
    public String getDescription() {
        return beverage.getDescription() + ", Mocha";
    }

    @Override
    public double cost() {
        return MOCHA_COST + beverage.cost();
    }
}
```

```java
package patterns.structural.decorator.coffee;

/**
 * ConcreteDecorator: Adds whipped cream on top of the beverage.
 * Whip adds $0.10 to the total cost.
 */
public class Whip extends CondimentDecorator {

    private static final double WHIP_COST = 0.10;

    public Whip(Beverage beverage) {
        super(beverage);
    }

    @Override
    public String getDescription() {
        return beverage.getDescription() + ", Whip";
    }

    @Override
    public double cost() {
        return WHIP_COST + beverage.cost();
    }
}
```

```java
package patterns.structural.decorator.coffee;

/**
 * ConcreteDecorator: Adds soy milk as an alternative to dairy milk.
 * Soy adds $0.15 to the total cost.
 */
public class Soy extends CondimentDecorator {

    private static final double SOY_COST = 0.15;

    public Soy(Beverage beverage) {
        super(beverage);
    }

    @Override
    public String getDescription() {
        return beverage.getDescription() + ", Soy";
    }

    @Override
    public double cost() {
        return SOY_COST + beverage.cost();
    }
}
```

```java
package patterns.structural.decorator.coffee;

/**
 * Driver class demonstrating the Decorator pattern with the coffee shop
 * domain. Notice that the client code uses only the {@link Beverage}
 * interface — it is completely unaware of how many layers of decoration
 * are present.
 */
public class CoffeeShopDemo {

    public static void main(String[] args) {

        // Order 1: Plain espresso, no condiments
        Beverage espresso = new Espresso();
        printOrder(espresso);
        // Output: Espresso — $1.99

        // Order 2: Dark roast with double mocha and whip
        Beverage darkRoast = new DarkRoast();
        darkRoast = new Mocha(darkRoast);
        darkRoast = new Mocha(darkRoast); // double mocha!
        darkRoast = new Whip(darkRoast);
        printOrder(darkRoast);
        // Output: Dark Roast Coffee, Mocha, Mocha, Whip — $1.49

        // Order 3: House blend with soy, mocha, and whip
        Beverage houseBlend = new HouseBlend();
        houseBlend = new Soy(houseBlend);
        houseBlend = new Mocha(houseBlend);
        houseBlend = new Whip(houseBlend);
        printOrder(houseBlend);
        // Output: House Blend Coffee, Soy, Mocha, Whip — $1.34

        // Order 4: Espresso with milk — minimalist
        Beverage latteBase = new Milk(new Espresso());
        printOrder(latteBase);
        // Output: Espresso, Milk — $2.24
    }

    private static void printOrder(Beverage beverage) {
        System.out.printf("%-50s $%.2f%n",
                beverage.getDescription(),
                beverage.cost());
    }
}
```

**Expected Output:**
```
Espresso                                           $1.99
Dark Roast Coffee, Mocha, Mocha, Whip              $1.49
House Blend Coffee, Soy, Mocha, Whip               $1.34
Espresso, Milk                                     $2.24
```

---

### 4.2 Java I/O Streams Replication

Java's I/O library is the most famous real-world application of the Decorator pattern. The implementation below replicates the architectural skeleton of `java.io.*` streams to make the pattern explicit.

```java
package patterns.structural.decorator.io;

import java.io.IOException;

/**
 * Component interface — mirrors {@code java.io.InputStream}.
 * All stream types (raw sources and decorators alike) implement this.
 */
public interface DataStream {

    /**
     * Reads a single byte from the stream.
     *
     * @return the byte value (0-255), or -1 on end-of-stream
     * @throws IOException if an I/O error occurs
     */
    int read() throws IOException;

    /**
     * Reads up to {@code len} bytes into the given buffer.
     *
     * @param buffer destination array
     * @param offset starting index in {@code buffer}
     * @param len    maximum bytes to read
     * @return number of bytes actually read, or -1 on end-of-stream
     * @throws IOException if an I/O error occurs
     */
    int read(byte[] buffer, int offset, int len) throws IOException;

    /**
     * Closes this stream and releases any associated resources.
     *
     * @throws IOException if an I/O error occurs during close
     */
    void close() throws IOException;
}
```

```java
package patterns.structural.decorator.io;

import java.io.IOException;

/**
 * ConcreteComponent: A stream backed by an in-memory byte array.
 * Equivalent to {@code java.io.ByteArrayInputStream}.
 */
public class ByteArrayDataStream implements DataStream {

    private final byte[] data;
    private int position;

    /**
     * Creates a stream over the given byte array.
     *
     * @param data the source bytes; defensively copied
     */
    public ByteArrayDataStream(byte[] data) {
        this.data = data.clone();
        this.position = 0;
    }

    @Override
    public int read() throws IOException {
        if (position >= data.length) {
            return -1;
        }
        return data[position++] & 0xFF;
    }

    @Override
    public int read(byte[] buffer, int offset, int len) throws IOException {
        if (position >= data.length) {
            return -1;
        }
        int bytesToRead = Math.min(len, data.length - position);
        System.arraycopy(data, position, buffer, offset, bytesToRead);
        position += bytesToRead;
        return bytesToRead;
    }

    @Override
    public void close() throws IOException {
        // No resources to release for in-memory stream
    }
}
```

```java
package patterns.structural.decorator.io;

import java.io.IOException;

/**
 * Abstract Decorator base for stream decorators.
 * Mirrors the role of {@code java.io.FilterInputStream} in the JDK.
 *
 * <p>All method calls are delegated to the wrapped stream by default.
 * Concrete decorators override only the methods they need to augment.</p>
 */
public abstract class DataStreamDecorator implements DataStream {

    /** The wrapped stream. Subclasses may access this for delegation. */
    protected final DataStream wrappedStream;

    /**
     * @param stream the stream to decorate; must not be null
     */
    protected DataStreamDecorator(DataStream stream) {
        if (stream == null) {
            throw new IllegalArgumentException("Wrapped stream must not be null");
        }
        this.wrappedStream = stream;
    }

    @Override
    public int read() throws IOException {
        return wrappedStream.read();
    }

    @Override
    public int read(byte[] buffer, int offset, int len) throws IOException {
        return wrappedStream.read(buffer, offset, len);
    }

    @Override
    public void close() throws IOException {
        wrappedStream.close();
    }
}
```

```java
package patterns.structural.decorator.io;

import java.io.IOException;

/**
 * ConcreteDecorator: Adds buffering to any {@link DataStream}.
 * Equivalent to {@code java.io.BufferedInputStream}.
 *
 * <p>Instead of making a system call for every byte read, this decorator
 * reads a large chunk into an internal buffer and serves subsequent reads
 * from that buffer. This dramatically reduces I/O overhead when the
 * underlying stream is slow (e.g., a file or network socket).</p>
 */
public class BufferedDataStream extends DataStreamDecorator {

    private static final int DEFAULT_BUFFER_SIZE = 8192;

    private final byte[] buffer;
    private int bufferPosition;
    private int bufferLimit;

    public BufferedDataStream(DataStream stream) {
        this(stream, DEFAULT_BUFFER_SIZE);
    }

    public BufferedDataStream(DataStream stream, int bufferSize) {
        super(stream);
        this.buffer = new byte[bufferSize];
        this.bufferPosition = 0;
        this.bufferLimit = 0;
    }

    @Override
    public int read() throws IOException {
        if (bufferPosition >= bufferLimit) {
            fillBuffer();
            if (bufferLimit == -1) {
                return -1;
            }
        }
        return buffer[bufferPosition++] & 0xFF;
    }

    @Override
    public int read(byte[] dest, int offset, int len) throws IOException {
        if (bufferPosition >= bufferLimit) {
            fillBuffer();
            if (bufferLimit == -1) {
                return -1;
            }
        }
        int bytesAvailable = bufferLimit - bufferPosition;
        int bytesToCopy = Math.min(len, bytesAvailable);
        System.arraycopy(buffer, bufferPosition, dest, offset, bytesToCopy);
        bufferPosition += bytesToCopy;
        return bytesToCopy;
    }

    private void fillBuffer() throws IOException {
        bufferPosition = 0;
        bufferLimit = wrappedStream.read(buffer, 0, buffer.length);
    }
}
```

```java
package patterns.structural.decorator.io;

import java.io.IOException;
import java.nio.charset.StandardCharsets;

/**
 * ConcreteDecorator: Adds metrics / tracing to any {@link DataStream}.
 * Counts bytes read and logs them — analogous to a monitoring wrapper
 * you might use in a production system to track I/O throughput.
 *
 * <p>This decorator is entirely transparent: it does not alter the data;
 * it only observes the traffic passing through.</p>
 */
public class MetricsDataStream extends DataStreamDecorator {

    private final String streamName;
    private long totalBytesRead;

    public MetricsDataStream(DataStream stream, String streamName) {
        super(stream);
        this.streamName = streamName;
        this.totalBytesRead = 0;
    }

    @Override
    public int read() throws IOException {
        int result = wrappedStream.read();
        if (result != -1) {
            totalBytesRead++;
        }
        return result;
    }

    @Override
    public int read(byte[] buffer, int offset, int len) throws IOException {
        int bytesRead = wrappedStream.read(buffer, offset, len);
        if (bytesRead > 0) {
            totalBytesRead += bytesRead;
        }
        return bytesRead;
    }

    @Override
    public void close() throws IOException {
        System.out.printf("[Metrics] Stream '%s' closed. Total bytes read: %d%n",
                streamName, totalBytesRead);
        wrappedStream.close();
    }

    public long getTotalBytesRead() {
        return totalBytesRead;
    }
}
```

```java
package patterns.structural.decorator.io;

import java.io.IOException;

/**
 * Demonstrates the I/O stream decorator chain.
 *
 * <p>The composition pattern here directly mirrors how Java's own I/O library
 * works:</p>
 * <pre>
 *   new BufferedInputStream(new FileInputStream("file.txt"))
 * </pre>
 * <p>becomes in our model:</p>
 * <pre>
 *   new BufferedDataStream(new ByteArrayDataStream(bytes))
 * </pre>
 */
public class IOStreamDemo {

    public static void main(String[] args) throws IOException {
        String content = "Hello, Decorator Pattern! This is a simulated I/O stream.";
        byte[] bytes = content.getBytes();

        // Layer 1: raw source
        DataStream raw = new ByteArrayDataStream(bytes);

        // Layer 2: add buffering for efficiency
        DataStream buffered = new BufferedDataStream(raw, 16);

        // Layer 3: add metrics/tracing — outermost layer
        DataStream monitored = new MetricsDataStream(buffered, "demo-stream");

        // Client code only knows about DataStream — completely unaware of layers
        System.out.println("Reading stream content:");
        int b;
        StringBuilder sb = new StringBuilder();
        while ((b = monitored.read()) != -1) {
            sb.append((char) b);
        }

        System.out.println(sb);
        monitored.close(); // triggers metrics report
    }
}
```

---

### 4.3 HTTP Middleware / Filter Chain Example

This example models a production-style HTTP middleware chain — the same architectural pattern used by Spring's `OncePerRequestFilter`, Jakarta Servlet filters, and frameworks like Express.js and ASP.NET Core middleware.

```java
package patterns.structural.decorator.http;

import java.util.HashMap;
import java.util.Map;

/**
 * Represents an HTTP request flowing through the middleware chain.
 */
public final class HttpRequest {

    private final String method;
    private final String path;
    private final Map<String, String> headers;
    private final String body;

    public HttpRequest(String method, String path, Map<String, String> headers, String body) {
        this.method = method;
        this.path = path;
        this.headers = new HashMap<>(headers);
        this.body = body;
    }

    public String getMethod() { return method; }
    public String getPath() { return path; }
    public String getHeader(String name) { return headers.getOrDefault(name, null); }
    public Map<String, String> getHeaders() { return Map.copyOf(headers); }
    public String getBody() { return body; }

    @Override
    public String toString() {
        return method + " " + path;
    }
}
```

```java
package patterns.structural.decorator.http;

/**
 * Represents an HTTP response produced by a handler or middleware.
 */
public final class HttpResponse {

    private final int statusCode;
    private final String body;
    private final String contentType;

    public HttpResponse(int statusCode, String body, String contentType) {
        this.statusCode = statusCode;
        this.body = body;
        this.contentType = contentType;
    }

    /** Convenience constructor for plain-text responses. */
    public static HttpResponse ok(String body) {
        return new HttpResponse(200, body, "text/plain");
    }

    public static HttpResponse unauthorized(String message) {
        return new HttpResponse(401, message, "text/plain");
    }

    public static HttpResponse tooManyRequests() {
        return new HttpResponse(429, "Rate limit exceeded. Try again later.", "text/plain");
    }

    public static HttpResponse internalError(String message) {
        return new HttpResponse(500, message, "text/plain");
    }

    public int getStatusCode() { return statusCode; }
    public String getBody() { return body; }
    public String getContentType() { return contentType; }

    @Override
    public String toString() {
        return "HTTP " + statusCode + " [" + contentType + "]: " + body;
    }
}
```

```java
package patterns.structural.decorator.http;

/**
 * Component interface: the contract that every HTTP handler must satisfy.
 * Both the real business handler and every middleware decorator implement this.
 *
 * <p>This is structurally equivalent to {@code javax.servlet.Filter} combined
 * with {@code HttpServlet} — both respond to HTTP requests.</p>
 */
@FunctionalInterface
public interface HttpHandler {

    /**
     * Handles the given HTTP request and returns a response.
     *
     * @param request the incoming HTTP request; never null
     * @return the HTTP response; never null
     */
    HttpResponse handle(HttpRequest request);
}
```

```java
package patterns.structural.decorator.http;

/**
 * ConcreteComponent: The actual business logic handler.
 * In a real application this would delegate to a service layer.
 */
public class UserProfileHandler implements HttpHandler {

    @Override
    public HttpResponse handle(HttpRequest request) {
        String path = request.getPath();

        if (path.startsWith("/api/users/")) {
            String userId = path.substring("/api/users/".length());
            String responseBody = String.format(
                    "{\"id\":\"%s\",\"name\":\"John Doe\",\"email\":\"john@example.com\"}",
                    userId
            );
            return new HttpResponse(200, responseBody, "application/json");
        }

        return new HttpResponse(404, "Not found", "text/plain");
    }
}
```

```java
package patterns.structural.decorator.http;

/**
 * Abstract base decorator for HTTP middleware.
 * Holds a reference to the next handler in the chain and delegates by default.
 */
public abstract class HttpMiddleware implements HttpHandler {

    protected final HttpHandler next;

    protected HttpMiddleware(HttpHandler next) {
        if (next == null) {
            throw new IllegalArgumentException("Next handler in chain must not be null");
        }
        this.next = next;
    }

    /**
     * Default pass-through: subclasses intercept before and/or after this call.
     */
    @Override
    public HttpResponse handle(HttpRequest request) {
        return next.handle(request);
    }
}
```

```java
package patterns.structural.decorator.http;

import java.time.Instant;
import java.util.UUID;

/**
 * ConcreteDecorator: Logging middleware.
 *
 * <p>Intercepts every request, generates a correlation ID, logs the request
 * before delegating downstream, and logs the response after. This is the
 * same pattern used by Spring's {@code CommonsRequestLoggingFilter} and
 * Logback's MDC-based request logging.</p>
 */
public class LoggingMiddleware extends HttpMiddleware {

    public LoggingMiddleware(HttpHandler next) {
        super(next);
    }

    @Override
    public HttpResponse handle(HttpRequest request) {
        String correlationId = UUID.randomUUID().toString().substring(0, 8);
        long startMs = System.currentTimeMillis();

        System.out.printf("[%s] [LOG] --> %s %s%n",
                correlationId,
                request.getMethod(),
                request.getPath());

        HttpResponse response;
        try {
            response = next.handle(request);
        } catch (Exception e) {
            long elapsed = System.currentTimeMillis() - startMs;
            System.out.printf("[%s] [LOG] <-- ERROR after %dms: %s%n",
                    correlationId, elapsed, e.getMessage());
            return HttpResponse.internalError("Internal server error");
        }

        long elapsed = System.currentTimeMillis() - startMs;
        System.out.printf("[%s] [LOG] <-- HTTP %d after %dms%n",
                correlationId,
                response.getStatusCode(),
                elapsed);

        return response;
    }
}
```

```java
package patterns.structural.decorator.http;

/**
 * ConcreteDecorator: Authentication middleware.
 *
 * <p>Validates the {@code Authorization} header before allowing the request
 * to proceed downstream. If no valid token is present, short-circuits the
 * chain and returns 401 Unauthorized immediately — the business handler
 * is never invoked.</p>
 *
 * <p>This mirrors Spring Security's {@code BasicAuthenticationFilter} or
 * a JWT bearer-token filter.</p>
 */
public class AuthenticationMiddleware extends HttpMiddleware {

    private static final String VALID_TOKEN = "Bearer secret-token-42";

    public AuthenticationMiddleware(HttpHandler next) {
        super(next);
    }

    @Override
    public HttpResponse handle(HttpRequest request) {
        String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !VALID_TOKEN.equals(authHeader)) {
            System.out.println("[AUTH] Request rejected: missing or invalid token");
            return HttpResponse.unauthorized("Authentication required");
        }

        System.out.println("[AUTH] Token validated successfully");
        return next.handle(request);
    }
}
```

```java
package patterns.structural.decorator.http;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;
import java.time.Instant;

/**
 * ConcreteDecorator: Rate-limiting middleware.
 *
 * <p>Enforces a per-client request-per-minute cap using a simple in-memory
 * token bucket strategy. In production this counter would live in Redis
 * (using Redisson or Lettuce) to work across multiple instances.</p>
 *
 * <p>This pattern is used by Spring Cloud Gateway's
 * {@code RequestRateLimiterGatewayFilter} and AWS API Gateway throttling.</p>
 */
public class RateLimitingMiddleware extends HttpMiddleware {

    private static final int MAX_REQUESTS_PER_WINDOW = 5;
    private static final long WINDOW_MS = 60_000L; // 1 minute

    private final Map<String, AtomicInteger> requestCounts = new ConcurrentHashMap<>();
    private volatile long windowStart = System.currentTimeMillis();

    public RateLimitingMiddleware(HttpHandler next) {
        super(next);
    }

    @Override
    public HttpResponse handle(HttpRequest request) {
        String clientId = resolveClientId(request);

        resetWindowIfExpired();

        AtomicInteger count = requestCounts.computeIfAbsent(clientId,
                k -> new AtomicInteger(0));

        int currentCount = count.incrementAndGet();

        if (currentCount > MAX_REQUESTS_PER_WINDOW) {
            System.out.printf("[RATE] Client '%s' exceeded limit (%d/%d)%n",
                    clientId, currentCount, MAX_REQUESTS_PER_WINDOW);
            return HttpResponse.tooManyRequests();
        }

        System.out.printf("[RATE] Client '%s': request %d/%d in current window%n",
                clientId, currentCount, MAX_REQUESTS_PER_WINDOW);

        return next.handle(request);
    }

    private String resolveClientId(HttpRequest request) {
        // In production: extract from JWT sub claim, API key, or IP address
        String apiKey = request.getHeader("X-API-Key");
        return apiKey != null ? apiKey : "anonymous";
    }

    private synchronized void resetWindowIfExpired() {
        long now = System.currentTimeMillis();
        if (now - windowStart >= WINDOW_MS) {
            requestCounts.clear();
            windowStart = now;
        }
    }
}
```

```java
package patterns.structural.decorator.http;

import java.util.Map;

/**
 * Driver demonstrating the HTTP middleware chain.
 *
 * <p>The chain is assembled inside-out — innermost handler first, then
 * wrapped by each middleware layer. The outermost decorator is what the
 * client calls. This assembly is typically done in a DI container
 * (e.g., Spring's {@code FilterRegistrationBean}).</p>
 *
 * <p>Chain order (outermost to innermost):</p>
 * <pre>
 *   Logging → RateLimiting → Authentication → UserProfileHandler
 * </pre>
 */
public class HttpMiddlewareDemo {

    public static void main(String[] args) {

        // 1. Build the chain: innermost first
        HttpHandler chain = new LoggingMiddleware(
                new RateLimitingMiddleware(
                        new AuthenticationMiddleware(
                                new UserProfileHandler()
                        )
                )
        );

        System.out.println("=== Request 1: Valid request with auth token ===");
        HttpRequest request1 = new HttpRequest(
                "GET",
                "/api/users/123",
                Map.of("Authorization", "Bearer secret-token-42",
                       "X-API-Key", "client-A"),
                ""
        );
        HttpResponse response1 = chain.handle(request1);
        System.out.println("Response: " + response1);
        System.out.println();

        System.out.println("=== Request 2: Missing auth token ===");
        HttpRequest request2 = new HttpRequest(
                "GET",
                "/api/users/456",
                Map.of("X-API-Key", "client-B"),
                ""
        );
        HttpResponse response2 = chain.handle(request2);
        System.out.println("Response: " + response2);
        System.out.println();

        System.out.println("=== Requests 3-8: Exceeding rate limit ===");
        for (int i = 3; i <= 8; i++) {
            HttpRequest req = new HttpRequest(
                    "GET",
                    "/api/users/" + i,
                    Map.of("Authorization", "Bearer secret-token-42",
                           "X-API-Key", "client-C"),
                    ""
            );
            HttpResponse resp = chain.handle(req);
            System.out.println("Request " + i + " response: " + resp);
        }
    }
}
```

---

## 5. Decorator Chain and Ordering Matters

One of the most important — and most commonly misunderstood — aspects of the Decorator pattern is that **the order in which decorators are stacked determines the behavior of the chain**.

### Analogy

Think of a chain of people in an assembly line. The person at the front of the line receives the raw product, does their work, and passes it on. The order in which each worker is positioned completely changes the final product.

### Order Changes Behavior: Rate Limiting Before vs. After Authentication

Consider these two different chain assemblies:

**Chain A (Rate-limit BEFORE auth):**
```java
// Rate limiter sees ALL requests, including unauthenticated ones.
// An attacker can exhaust your rate limit with garbage requests.
HttpHandler chainA = new LoggingMiddleware(
    new RateLimitingMiddleware(        // outer
        new AuthenticationMiddleware( // inner
            new UserProfileHandler()
        )
    )
);
```

**Chain B (Auth BEFORE rate-limit):**
```java
// Only authenticated requests reach the rate limiter.
// Anonymous attackers get 401 immediately without consuming rate-limit quota.
HttpHandler chainB = new LoggingMiddleware(
    new AuthenticationMiddleware(     // outer
        new RateLimitingMiddleware(   // inner
            new UserProfileHandler()
        )
    )
);
```

Both chains compile. Both chains run. But Chain B is **semantically correct** from a security standpoint and Chain A creates a denial-of-service vulnerability.

### Order Changes Semantics: Mocha vs. Whip Example

In the coffee shop:

```java
// Chain 1: Whip on top of Mocha
Beverage b1 = new Whip(new Mocha(new Espresso()));
// Description: "Espresso, Mocha, Whip"

// Chain 2: Mocha on top of Whip
Beverage b2 = new Mocha(new Whip(new Espresso()));
// Description: "Espresso, Whip, Mocha"
```

Both cost the same ($2.29), but the descriptions differ. In a physical coffee shop, the order matters — you pour whip before or after adding mocha syrup, and the result is a different drink.

### Java I/O Ordering

```java
// CORRECT: Buffer wraps the network stream; reads are buffered before decompression
DataStream correct = new GZIPDataStream(new BufferedDataStream(networkStream));

// WRONG: Buffering compressed data has little benefit; you want to buffer
// the raw network bytes, then decompress from that buffer
DataStream wrong = new BufferedDataStream(new GZIPDataStream(networkStream));
```

The rule of thumb: **order your decorators so that the most expensive or most restrictive operation is applied as early as possible to short-circuit work.**

---

## 6. Real-World Analogy

### The Christmas Tree Ornament Model

Imagine a bare Christmas tree. The tree is your `ConcreteComponent`. You can decorate it:

1. Add lights → `LightsDecorator(tree)`
2. Add tinsel on top of the lit tree → `TinselDecorator(lit_tree)`
3. Add ornaments on top of that → `OrnamentsDecorator(tinsel_tree)`
4. Add a star on top → `StarDecorator(ornaments_tree)`

At every stage, the result is still a Christmas tree. Each decorator adds behavior (lights blink, tinsel shimmers, ornaments hang) without changing the underlying structure. You can add, remove, or rearrange decorations independently. The tree does not know it is decorated. The decorator does not know how the tree was built.

### The Pipeline in a Coffee House

When you order a custom coffee drink, the barista:
1. Pulls a shot of espresso (base component)
2. Adds steamed milk (decorator)
3. Adds vanilla syrup (decorator)
4. Adds whipped cream (decorator)
5. Adds caramel drizzle (decorator)

Each step is optional, independent, and composable. You do not need to train a specialized barista for every possible combination. The same set of condiment "decorators" applies to any base drink.

---

## 7. Real-World Java Ecosystem Examples

### 7.1 `java.io` Package — The Textbook Example

The entire `java.io` stream hierarchy is a Decorator implementation:

```
InputStream                     (Component interface)
├── FileInputStream             (ConcreteComponent)
├── ByteArrayInputStream        (ConcreteComponent)
├── FilterInputStream           (Abstract Decorator)
│   ├── BufferedInputStream     (ConcreteDecorator — adds buffering)
│   ├── DataInputStream         (ConcreteDecorator — adds typed reads)
│   ├── PushbackInputStream     (ConcreteDecorator — adds unread)
│   └── CheckedInputStream      (ConcreteDecorator — adds checksumming)
└── PipedInputStream            (ConcreteComponent)
```

Real usage:
```java
// Read a gzip-compressed binary file with buffering and checksum validation
DataInputStream dis = new DataInputStream(
    new BufferedInputStream(
        new GZIPInputStream(
            new CheckedInputStream(
                new FileInputStream("data.bin.gz"),
                new CRC32()
            )
        )
    )
);
int magicNumber = dis.readInt(); // reads 4 bytes, interpreting as int
```

### 7.2 `java.io.BufferedReader` and `BufferedWriter`

```java
// Text stream equivalent
BufferedReader reader = new BufferedReader(
    new InputStreamReader(
        new FileInputStream("data.txt"),
        StandardCharsets.UTF_8
    )
);
// BufferedReader decorates InputStreamReader which decorates FileInputStream
```

`BufferedReader.readLine()` is extra behavior added by the decorator; `InputStreamReader` only knows about single characters.

### 7.3 `javax.servlet` Filter Chain

The Servlet API's filter chain is a direct application of Decorator. Each `Filter` wraps the next, forming a chain that terminates at the `Servlet`:

```java
// From web.xml or @WebFilter annotation — assembled by the servlet container
// Filter1 → Filter2 → Filter3 → MyServlet

public class LoggingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        // Pre-processing (before delegation)
        long start = System.currentTimeMillis();

        // Delegate to next in chain (the "next" decorator)
        chain.doFilter(request, response);

        // Post-processing (after delegation)
        long elapsed = System.currentTimeMillis() - start;
        System.out.println("Request processed in " + elapsed + "ms");
    }
}
```

Note: The Servlet API uses a `FilterChain` object rather than constructor injection to pass the reference to the next handler — a variation of the pattern sometimes called the **Chain of Responsibility** hybrid, but the structural intent is Decorator.

### 7.4 Spring Security Filter Chain

Spring Security registers a chain of filters before the dispatcher servlet. The security filter chain is configured as:

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .sessionManagement(session ->
            session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/public/**").permitAll()
            .anyRequest().authenticated())
        .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
    return http.build();
}
```

Internally, Spring assembles a `FilterChainProxy` that wraps `SecurityFilterChain` instances. Each security concern (CSRF, session management, JWT validation, authorization) is a separate filter/decorator.

### 7.5 Project Lombok `@Delegate`

Lombok's `@Delegate` annotation generates delegation boilerplate — effectively creating a decorator where you only override the methods you want to change:

```java
public class InstrumentedList<T> implements List<T> {
    @Delegate
    private final List<T> delegate = new ArrayList<>();

    @Override
    public boolean add(T element) {
        System.out.println("Adding: " + element); // extra behavior
        return delegate.add(element);              // delegation
    }
    // All other List methods are generated by @Delegate
}
```

---

## 8. Decorator vs Inheritance

This is one of the most important design trade-offs in object-oriented design, and it comes up frequently in senior engineering interviews.

### The Combinatorial Explosion Problem

Suppose you have a `TextWidget` that needs to support:
- Border (yes/no)
- Scroll (yes/no)
- Shadow (yes/no)

**With inheritance:** You need every combination:
```
TextWidget
├── BorderedTextWidget
├── ScrollableTextWidget
├── ShadowTextWidget
├── BorderedScrollableTextWidget
├── BorderedShadowTextWidget
├── ScrollableShadowTextWidget
└── BorderedScrollableShadowTextWidget
```

That is **2^3 = 8 classes** for 3 boolean features. With 10 features, you need 1,024 classes.

**With Decorator:** You need:
```
Widget (interface)
TextWidget (concrete)
BorderDecorator
ScrollDecorator
ShadowDecorator
```

That is **4 classes** for 3 features. With 10 features, you need 11 classes. The composition handles all 1,024 combinations.

### Static vs. Dynamic Extension

| Dimension | Inheritance | Decorator |
|---|---|---|
| When is behavior added? | Compile time (static) | Runtime (dynamic) |
| Can you change behavior of a live object? | No | Yes (re-wrap it) |
| Type relationship | Subclass IS-A Superclass | Decorator IS-A Component (via interface) |
| Original class modified? | Derived class may override | Original class never touched |
| Violates OCP? | Adding new behavior requires new subclass (fine) or modifying existing (bad) | Never modifies existing classes |
| Supports multiple behavior combinations? | Poor (combinatorial explosion) | Excellent |
| Runtime introspection | `instanceof` works reliably | `instanceof` only identifies outermost decorator |

### Code Example Contrasting Both

```java
// INHERITANCE approach — works but doesn't scale
public class BufferedFileReader extends FileReader {
    // adds buffering, but can only buffer FileReaders
}
public class BufferedStringReader extends StringReader {
    // adds buffering, but can only buffer StringReaders
    // Same buffering logic duplicated!
}

// DECORATOR approach — scales to any Reader
public class BufferedReaderDecorator extends FilterReader {
    // adds buffering to ANY Reader — FileReader, StringReader, NetworkReader
    public BufferedReaderDecorator(Reader in) { super(in); }
}
```

The Decorator approach keeps buffering DRY — written once, reusable with any compatible type.

### When Inheritance Is Still the Right Choice

- The extended behavior is **always** present (not optional): just put it in the base class.
- You need to **override** behavior, not just add to it: Decorator adds; it doesn't replace.
- The class hierarchy is shallow and combinations are few: a `PremiumUser extends User` is perfectly fine.
- You need the subclass to be recognized by `instanceof` checks in client code.

---

## 9. Decorator vs Strategy Pattern

Both Decorator and Strategy are often confused because both involve composition and both change the behavior of an object. The differences are subtle but architecturally important.

### Structural Comparison

```
DECORATOR                              STRATEGY
─────────────────────────────────      ───────────────────────────────
Component                              Context
    ▲                                      - strategy: Strategy
    │ implements                            + setStrategy(s)
    │                                       + executeOperation()
ConcreteComponent          vs                  → delegates to strategy
    ▲
    │ implements                        Strategy (interface)
    │                                       ▲
Decorator                              ConcreteStrategyA
    - wrappee: Component               ConcreteStrategyB
    ▲
ConcreteDecoratorA
ConcreteDecoratorB
```

### Intent Differences

| Dimension | Decorator | Strategy |
|---|---|---|
| Primary goal | **Add** responsibilities/features to an object | **Swap** an algorithm or behavior |
| Relationship to component | IS-A Component (same interface) | HAS-A Strategy (different interface) |
| Number of active behaviors | Multiple decorators stack (additive) | One strategy active at a time (exclusive) |
| Transparency to client | Completely transparent — client sees same type | Not transparent — client knows it has a context |
| Knows about the wrapped object? | Yes — always has a reference to `wrappee` | No — strategy knows nothing about context |
| Direction of delegation | Decorator calls `wrappee.operation()` and adds behavior around it | Context calls `strategy.execute()` and uses the result |

### Example Illustrating the Difference

**Strategy** — different sorting algorithms for the same data structure:
```java
// Context: the list
// Strategy: the sort algorithm — mutually exclusive alternatives
list.setStrategy(new QuickSortStrategy());
list.sort(); // QuickSort runs

list.setStrategy(new MergeSortStrategy());
list.sort(); // MergeSort runs — QuickSort replaced, not added
```

**Decorator** — adding behavior to sorting without changing the algorithm:
```java
// Each decorator ADDS a capability — they stack, not replace
SortEngine engine = new LoggingDecorator(
    new TimingDecorator(
        new QuickSortEngine()
    )
);
engine.sort(data); // QuickSort runs, timing is measured, logging happens
```

### A Subtler Point: Both Can Be Used Together

A sorting engine could use **Strategy** to choose the algorithm AND **Decorator** to add cross-cutting concerns (logging, timing, caching) around whichever strategy is chosen. These patterns are complementary, not competing.

---

## 10. Trade-offs

No pattern is universally good. The Decorator pattern makes deliberate trade-offs.

### Advantages

**Flexibility over inheritance:**
Behavior is composed at runtime from small, focused classes. New combinations require no new classes — just new compositions.

**Single Responsibility Principle adherence:**
Each decorator does exactly one thing. `LoggingMiddleware` only logs. `AuthenticationMiddleware` only authenticates. This makes each decorator easy to test in isolation.

**Open/Closed Principle adherence:**
Adding a new behavior (a new decorator class) never requires modifying existing code.

**Reversibility:**
Because decoration happens at construction time, different application contexts can compose different chains. A test environment might omit the rate-limiting decorator. A production environment adds it. No `if (ENV == TEST)` branches needed.

**Fine-grained control:**
You can decorate individual object instances, not entire classes. Two `Espresso` objects can have different condiment stacks.

### Disadvantages

**Identity and equality problems:**
A decorated object is not the same instance as its inner component. Code that checks `instanceof ConcreteComponent` or relies on `equals()`/`hashCode()` of the concrete type will fail:
```java
Beverage espresso = new Espresso();
Beverage decorated = new Milk(new Mocha(espresso));
System.out.println(decorated instanceof Espresso); // false — this will surprise callers
```

**Debugging difficulty:**
A stack trace through a deep decorator chain looks like:
```
LoggingMiddleware.handle()
  RateLimitingMiddleware.handle()
    AuthenticationMiddleware.handle()
      UserProfileHandler.handle()
```
This is manageable with good naming, but it adds cognitive overhead compared to a single method call.

**Interface bloat burden:**
Every decorator must implement every method of the component interface. If the interface has 20 methods and you add a decorator that only cares about one, you still write 19 pass-through methods (or extend an abstract base decorator). This is mitigated by using an abstract base class with default delegation, but it is still boilerplate.

**Order sensitivity is implicit:**
The correct ordering of decorators is not encoded in the type system. Nothing prevents a developer from accidentally wrapping authentication outside of logging, and the compiler will not complain. This is a design-time discipline problem, not a compile-time safety problem.

**Not suitable for method interception on specific methods:**
If you want to decorate only *one* method of a five-method interface and pass through the rest, you still need to implement all five. AOP (Aspect-Oriented Programming) is a better tool for surgical method-level interception.

**Object proliferation:**
Each decoration creates a new object. In hot paths processing millions of requests per second, this can matter. Profiling is required before concluding it is a bottleneck, but the overhead is real.

---

## 11. Common Pitfalls and Gotchas

### Pitfall 1: Forgetting to Delegate

The most common bug when writing decorators: overriding a method and forgetting to call the wrapped component.

```java
// WRONG: This decorator swallows the cost instead of adding to it
public class Milk extends CondimentDecorator {
    @Override
    public double cost() {
        return 0.25; // BUG: forgot to add beverage.cost()
    }
}

// CORRECT:
public class Milk extends CondimentDecorator {
    @Override
    public double cost() {
        return 0.25 + beverage.cost(); // always delegate
    }
}
```

### Pitfall 2: Wrapping a Null Component

Always validate the wrapped component in the constructor. Null pointer exceptions deep inside a chain are difficult to diagnose.

```java
protected CondimentDecorator(Beverage beverage) {
    Objects.requireNonNull(beverage, "Cannot decorate a null beverage");
    this.beverage = beverage;
}
```

### Pitfall 3: Using `instanceof` on Decorated Objects

```java
Beverage b = new Milk(new Espresso());
if (b instanceof Espresso) { // FALSE — b is a Milk instance
    // this code never executes
}
```

If you find yourself doing `instanceof` checks through a decorator chain, it is a sign that the component interface needs a richer API, or that you should use Visitor pattern instead.

### Pitfall 4: Mutable State in Decorators

If a decorator holds mutable state, sharing the decorator across threads or requests is dangerous:

```java
// DANGEROUS: requestCount is shared mutable state
public class RateLimitingMiddleware extends HttpMiddleware {
    private int requestCount = 0; // not thread-safe as a plain int
    // ...
}
// Use AtomicInteger or ConcurrentHashMap instead
```

### Pitfall 5: Deep Chains Causing Stack Overflows

In theory, decorator chains can be arbitrarily deep. In practice, each layer adds a stack frame. Thousands of layers would cause `StackOverflowError`. This is rarely a real problem, but be aware when generating decorator chains programmatically (e.g., from configuration).

### Pitfall 6: Assuming Order Is Irrelevant

Developers new to the pattern often treat decorator ordering as cosmetic. It is not. Always document the required ordering and enforce it in a factory or builder if possible:

```java
// Builder that enforces correct middleware ordering
public class MiddlewareChainBuilder {
    private HttpHandler handler;

    public MiddlewareChainBuilder(HttpHandler businessHandler) {
        this.handler = businessHandler;
    }

    // Each method wraps the current chain
    public MiddlewareChainBuilder withAuthentication() {
        this.handler = new AuthenticationMiddleware(handler);
        return this;
    }

    public MiddlewareChainBuilder withRateLimiting() {
        this.handler = new RateLimitingMiddleware(handler);
        return this;
    }

    public MiddlewareChainBuilder withLogging() {
        this.handler = new LoggingMiddleware(handler);
        return this;
    }

    // build() returns the fully assembled chain
    public HttpHandler build() {
        return handler;
    }
}

// Usage — order is now enforced by the builder API
HttpHandler chain = new MiddlewareChainBuilder(new UserProfileHandler())
    .withAuthentication()    // applied first (innermost after business handler)
    .withRateLimiting()      // applied second
    .withLogging()           // applied last (outermost)
    .build();
```

---

## 12. Interview Discussion Points

### Discussion Point 1: "How does Decorator differ from Inheritance?"

The key insight: inheritance is **compile-time** and **class-level**; Decorator is **runtime** and **instance-level**. With inheritance, every instance of a subclass has the behavior. With Decorator, you choose at instantiation time which specific object gets the extra behavior.

### Discussion Point 2: "Is Java I/O a good design?"

A famous critique of `java.io` is that, while it correctly applies Decorator, the result is **verbose and error-prone** for users who don't understand the pattern. Most developers don't know why they need to nest constructors. This is a usability failure, not a pattern failure — the pattern is correctly applied, but the abstraction is too low-level for most use cases. `java.nio.file.Files` (Java 7+) fixed the usability issue with factory methods that hide the decorator chain.

### Discussion Point 3: "Can you decorate an already-decorated object?"

Yes, and this is the whole point. A decorator does not need to know how many layers are below it. `new Whip(new Mocha(new Mocha(new Espresso())))` is three layers deep, and each layer only talks to the interface it was handed.

### Discussion Point 4: "When would you choose AOP over Decorator?"

AOP (via Spring's `@Aspect` or bytecode manipulation) is better when:
- You need to intercept a method that is not part of a known interface.
- The class you want to decorate is created by a third-party framework and cannot be easily wrapped.
- The decoration needs to apply to all instances of a type automatically, without explicit construction.
- The decoration applies to many unrelated types (cross-cutting concerns like security and logging applied to every service class in the application).

Decorator is better when:
- The wrapping is explicit and intentional.
- Different instances need different decorations.
- You want the decoration to be visible at the call site.

### Discussion Point 5: "How does Decorator relate to the Chain of Responsibility pattern?"

Both patterns pass a request through a chain of handlers. The difference is in intent:
- **Chain of Responsibility**: each handler decides whether to handle the request itself or pass it to the next. Only one handler typically processes the request.
- **Decorator**: every handler in the chain processes the request (adding behavior before/after delegation). All layers participate.

The `FilterChain` in Java Servlets is a hybrid: all filters run (Decorator-style), but any filter can short-circuit the chain by not calling `chain.doFilter()` (Chain of Responsibility-style).

---

## 13. Interview Questions and Model Answers

---

### Q1: What is the Decorator pattern and what problem does it solve?

**Model Answer:**

The Decorator pattern is a structural design pattern that dynamically adds responsibilities to an object by wrapping it in a decorator object that shares the same interface. The pattern solves two related problems.

The first problem is **combinatorial explosion from inheritance**. If you have a base class with N optional features, inheritance requires 2^N subclasses to cover every combination. With Decorator, each feature is one class, and combinations are achieved by nesting instances at runtime — N features require N decorator classes plus the base, for a total of N+1 classes regardless of how many combinations exist.

The second problem is **closed-for-modification extensibility**. When a class is finalized, third-party, or simply tested and stable, you should not modify it to add new behavior. The Decorator pattern lets you wrap the object with new behavior without touching the original class, respecting the Open/Closed Principle.

The key structural requirement is that the decorator must implement the **same interface** as the component it wraps. This transparency is what makes the pattern work — clients that receive a `Beverage` do not know whether it is a raw `Espresso` or an `Espresso` wrapped in ten layers of condiment decorators.

---

### Q2: Explain how Java's `java.io` package uses the Decorator pattern.

**Model Answer:**

`java.io` is the most widely cited real-world example of the Decorator pattern in the Java standard library. The component interface is `InputStream`. Concrete components — the raw data sources — are classes like `FileInputStream`, `ByteArrayInputStream`, and `SocketInputStream`. These classes know how to read bytes from a specific source but offer no additional capabilities.

The abstract decorator class is `FilterInputStream`. It implements `InputStream` and holds a reference to another `InputStream` (the `in` field). By default, all its methods delegate to the wrapped stream. This makes writing concrete decorators easy: you only override the methods where you want to add or change behavior.

Concrete decorators include:
- `BufferedInputStream` — adds an in-memory buffer to reduce system call frequency
- `DataInputStream` — adds typed reads (`readInt()`, `readDouble()`, etc.)
- `GZIPInputStream` (in `java.util.zip`) — transparently decompresses gzip data
- `CheckedInputStream` — computes a checksum as data passes through
- `PushbackInputStream` — allows bytes to be "un-read" back into the stream

The decorator chain is assembled by nesting constructors:
```java
DataInputStream dis = new DataInputStream(
    new BufferedInputStream(new FileInputStream("data.bin"))
);
```

This composition reads a file (`FileInputStream`), buffers reads in 8KB chunks (`BufferedInputStream`), and interprets the bytes as typed primitives (`DataInputStream`). Each layer adds one capability; none of them modifies the others.

A common criticism is that the nesting syntax is verbose and requires developers to understand the pattern to use the API correctly. Java's `Files.newBufferedReader()` factory method in `java.nio.file` addresses this by hiding the decoration behind a simpler API.

---

### Q3: How does the ordering of decorators affect behavior? Give a concrete example.

**Model Answer:**

Decorator ordering is critical and is one of the most frequently misunderstood aspects of the pattern. Because each decorator wraps the next one, the outer decorator runs first on the way in and last on the way out. This means ordering is not merely cosmetic — it changes the semantics of the entire chain.

Consider an HTTP middleware chain with logging, authentication, and rate limiting. If you order it as:
```
Logging → RateLimiting → Authentication → Handler
```

Then unauthenticated requests pass through the rate limiter before being rejected by the authentication layer. An attacker sending thousands of unauthenticated requests can exhaust the rate limit quota for legitimate users — a denial-of-service vulnerability.

If you instead order it as:
```
Logging → Authentication → RateLimiting → Handler
```

Unauthenticated requests are rejected at the authentication layer and never reach the rate limiter. Only authenticated clients consume rate-limit quota, which is the correct behavior.

Both chains are type-correct Java. The compiler gives you no warning. The test suite might not catch it. This is why a **factory or builder that enforces the correct ordering** is often essential in production systems:

```java
public HttpHandler buildProductionChain(HttpHandler handler) {
    // Applied inside-out: logging is outermost, handler is innermost
    return new LoggingMiddleware(
        new AuthenticationMiddleware(
            new RateLimitingMiddleware(handler)
        )
    );
}
```

Another example from Java I/O: `new GZIPInputStream(new BufferedInputStream(stream))` decompresses compressed data while using a buffer — correct. `new BufferedInputStream(new GZIPInputStream(stream))` buffers the decompressed output, which wastes memory on data that will be discarded — semantically valid but suboptimal.

The lesson: always document the intended ordering of your decorator hierarchy and consider enforcing it through a factory, builder, or DI container configuration.

---

### Q4: What is the difference between the Decorator and Strategy patterns? They both use composition — how do you tell them apart?

**Model Answer:**

Decorator and Strategy are frequently confused because both use object composition and both change behavior without subclassing. The distinction lies in **intent** and **structural relationship**.

**Intent:** Decorator *adds* responsibilities to an existing object. Strategy *replaces* a core algorithm or behavior with an alternative. Decorator is additive and transparent; Strategy is substitutive and explicit.

**Structural relationship:** A Decorator implements the **same interface** as the component it wraps. This is the critical design choice — it makes decoration transparent to the client. A Strategy is a separate interface consumed by a Context. The Context's interface is different from the Strategy's interface.

**Stacking behavior:** Decorators stack — you can have ten decorators around one component, and all of them participate in every operation. Strategies are typically exclusive — a context uses one strategy at a time (though it can switch between them).

**Awareness:** A Decorator always holds a reference to the thing it wraps and delegates to it. The Decorator's job includes calling `wrappee.operation()` at some point. A Strategy is an isolated algorithm; it does not know what Context it serves.

A concrete illustration: if you have a list and you want to add timing and logging around sort operations regardless of how sorting is implemented, that is Decorator — you wrap the sort call and add cross-cutting behavior. If you want to choose between quick sort and merge sort based on data size, that is Strategy — you select one algorithm to use, not add behavior around it.

In practice, the patterns can coexist. A `SortEngine` might use a Strategy to determine which algorithm to run, while being wrapped in a `TimingDecorator` and a `CachingDecorator` that add behavior around whichever strategy is active.

---

### Q5: How would you implement a Decorator that is thread-safe?

**Model Answer:**

Thread safety in decorators requires careful consideration of two sources of risk: shared mutable state within a decorator, and thread safety of the wrapped component.

**Shared mutable state in the decorator itself** is the most common issue. Consider a rate-limiting decorator that counts requests. A naive implementation using a plain `int` counter is not thread-safe:

```java
// NOT thread-safe
public class RateLimitingMiddleware extends HttpMiddleware {
    private int count = 0; // data race on concurrent requests

    @Override
    public HttpResponse handle(HttpRequest request) {
        count++; // not atomic
        if (count > MAX) return HttpResponse.tooManyRequests();
        return next.handle(request);
    }
}
```

The fix is to use atomic primitives or concurrent data structures:

```java
// Thread-safe
public class RateLimitingMiddleware extends HttpMiddleware {
    private final AtomicInteger count = new AtomicInteger(0);
    private final ConcurrentHashMap<String, AtomicInteger> perClientCount =
        new ConcurrentHashMap<>();

    @Override
    public HttpResponse handle(HttpRequest request) {
        String clientId = resolveClientId(request);
        AtomicInteger clientCount = perClientCount.computeIfAbsent(
            clientId, k -> new AtomicInteger(0));

        if (clientCount.incrementAndGet() > MAX_PER_CLIENT) {
            return HttpResponse.tooManyRequests();
        }
        return next.handle(request);
    }
}
```

**Thread safety of the wrapped component** is a separate concern. A decorator cannot make a non-thread-safe component thread-safe simply by wrapping it. If the wrapped component has thread-safety issues, you must either fix the component or introduce synchronization in the decorator at a level that prevents concurrent access to the component. This typically means `synchronized` blocks around calls to `next.handle(request)`, which becomes a serialization bottleneck.

The best approach is to design components to be **effectively immutable** or **stateless** so that thread safety is trivially guaranteed. Stateless handlers (where all state lives in the request/response objects passed on the stack) are intrinsically thread-safe.

---

### Q6: What are the limitations of the Decorator pattern, and when would you not use it?

**Model Answer:**

The Decorator pattern has several genuine limitations that make it the wrong choice in certain situations.

**Large component interfaces create maintenance burden.** Every decorator must implement every method of the component interface. If you have an interface with 20 methods and you write 10 decorators, you write 200 method implementations — even if most are trivial pass-throughs. An abstract base decorator mitigates this (you only implement pass-throughs once), but the boilerplate remains. If the interface is broad, prefer AOP or a Proxy pattern.

**Type identity is broken.** A decorated object is not an instance of its inner component's class. Code that uses `instanceof` to make decisions will behave incorrectly when handed a decorated object. This is a real problem when integrating with frameworks or libraries that perform type checks. If type identity matters, consider Proxy (which can return the exact type expected via reflection) or redesign to avoid type checks.

**Order sensitivity is invisible.** Incorrect ordering of decorators does not produce a compile error or even a runtime exception — it produces logically incorrect behavior. If the correct ordering is not obvious and enforced (by a factory or builder), it is a reliability hazard.

**Introspection is difficult.** Peeling back a decorator chain to inspect or modify an inner component requires iterative unwrapping with careful type casting. Libraries like Jackson (when deserializing) or Spring (when finding target beans behind proxies) have had repeated bugs related to peeling through decorator/proxy layers.

**Performance overhead is real but usually minor.** Each decorator adds one virtual method dispatch per operation. For most applications this is negligible. For extremely tight loops running billions of times per second, it can matter. Measure before optimizing.

**When not to use:** When the feature being added is always present (subclass or modify the base class directly). When clients depend on the concrete type for correctness. When the interface is so large that every decorator becomes a wall of boilerplate. When you need to intercept specific methods rather than wrapping all of them (use AOP). When the number of combinations is small and fixed (simple inheritance is clearer).

---

### Q7: Can you explain how Spring Security's filter chain relates to the Decorator pattern?

**Model Answer:**

Spring Security assembles a chain of `jakarta.servlet.Filter` implementations that execute before the application's `DispatcherServlet`. While the Servlet API's `FilterChain` is technically a variant of Chain of Responsibility (because any filter can short-circuit the chain by not calling `chain.doFilter()`), from an architectural perspective it is a Decorator chain applied to HTTP request/response processing.

Each filter in the chain is a decorator that wraps the remaining filters. A request entering the chain flows through:

```
SecurityContextPersistenceFilter         (loads security context from session)
    → UsernamePasswordAuthenticationFilter (validates credentials)
    → BasicAuthenticationFilter          (validates Basic auth header)
    → JwtAuthenticationFilter            (custom — validates JWT token)
    → ExceptionTranslationFilter         (maps security exceptions to HTTP codes)
    → FilterSecurityInterceptor          (evaluates access control rules)
        → DispatcherServlet              (the actual application)
```

Each filter adds one security concern. They are individually configurable, reorderable (with care), and independently testable. Adding a new security concern — say, IP allowlisting — means writing one new `Filter` implementation and registering it at the appropriate position in the chain. No existing filter is modified.

Spring Security's `HttpSecurity` DSL is a builder that assembles this chain in the correct order:

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf(AbstractHttpConfigurer::disable)
        .addFilterBefore(new JwtAuthFilter(), UsernamePasswordAuthenticationFilter.class)
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
    return http.build();
}
```

`addFilterBefore` is the API for controlling decorator ordering. This is the pattern problem that decorator ordering solves — Spring provides explicit ordering APIs because order is critical for security correctness.

The key difference from a pure Decorator implementation: `FilterChain.doFilter()` is an explicit method call that forwards to the next handler, whereas in the canonical Decorator pattern the forwarding happens through the wrapped field. The behavioral result is identical; the mechanism differs.

---

### Q8: Write a Decorator that adds caching to any service method. What are the design considerations?

**Model Answer:**

A caching decorator is a real-world use case that illustrates several important considerations beyond just implementing the pattern.

```java
package patterns.structural.decorator.cache;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Function;

/**
 * A generic caching decorator for any service that maps a key to a value.
 * This demonstrates Decorator applied to a generic interface.
 *
 * @param <K> the type of the key (cache key)
 * @param <V> the type of the value (cached result)
 */
public interface DataService<K, V> {
    /**
     * Retrieves the value for the given key.
     *
     * @param key the lookup key; must not be null
     * @return the value associated with the key, or null if not found
     */
    V get(K key);
}

/**
 * ConcreteComponent: Simulates a slow database or external API call.
 */
class RemoteDataService implements DataService<String, String> {
    @Override
    public String get(String key) {
        System.out.println("[DB] Executing slow query for key: " + key);
        // Simulate network latency
        try { Thread.sleep(100); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        return "value-for-" + key;
    }
}

/**
 * ConcreteDecorator: Adds transparent in-memory caching.
 *
 * Design considerations addressed:
 * 1. Cache key construction — use the input key directly (K must implement equals/hashCode correctly)
 * 2. Null values — ConcurrentHashMap does not allow null values; use a sentinel
 * 3. Cache invalidation — TTL-based expiry (omitted for brevity; use Caffeine in production)
 * 4. Thread safety — ConcurrentHashMap with computeIfAbsent ensures at-most-once execution per key
 * 5. Cache stampede — computeIfAbsent prevents duplicate computation under load
 */
class CachingDataService<K, V> implements DataService<K, V> {

    private final DataService<K, V> delegate;
    private final Map<K, V> cache;
    private final int maxSize;

    public CachingDataService(DataService<K, V> delegate, int maxSize) {
        if (delegate == null) throw new IllegalArgumentException("Delegate must not be null");
        this.delegate = delegate;
        this.cache = new ConcurrentHashMap<>();
        this.maxSize = maxSize;
    }

    @Override
    public V get(K key) {
        if (key == null) throw new IllegalArgumentException("Cache key must not be null");

        // computeIfAbsent is atomic — only one thread executes the loader for a given key
        V cached = cache.get(key);
        if (cached != null) {
            System.out.println("[CACHE] Hit for key: " + key);
            return cached;
        }

        System.out.println("[CACHE] Miss for key: " + key);

        // Simple size eviction — in production use Caffeine's maximumSize
        if (cache.size() >= maxSize) {
            K oldest = cache.keySet().iterator().next();
            cache.remove(oldest);
        }

        V value = delegate.get(key);
        if (value != null) {
            cache.put(key, value);
        }
        return value;
    }

    public int cacheSize() { return cache.size(); }

    public void invalidate(K key) { cache.remove(key); }

    public void invalidateAll() { cache.clear(); }
}

class CachingDemo {
    public static void main(String[] args) {
        DataService<String, String> service = new CachingDataService<>(
            new RemoteDataService(), 100
        );

        // First call — cache miss, hits DB
        System.out.println(service.get("user:42"));

        // Second call — cache hit, no DB
        System.out.println(service.get("user:42"));

        // Third call, different key — cache miss
        System.out.println(service.get("user:99"));
    }
}
```

**Key design considerations for a production caching decorator:**

1. **Cache key semantics**: The key type `K` must have correct `equals()` and `hashCode()` implementations. For complex composite keys, use a dedicated key class or a string representation.

2. **Null values**: Decide whether to cache null results (representing "not found"). The implementation above does not cache nulls, which means a missing key always hits the backing service. This can be correct (if entries are created dynamically) or a performance problem (if the same missing key is looked up frequently). Use a `NullValue` sentinel to cache absence.

3. **Cache invalidation**: The hardest problem in distributed systems. Options range from TTL-based expiry (simple, stale data possible) to event-driven invalidation (complex, consistent). In production, use a battle-tested library like Caffeine or Redis rather than a homegrown `ConcurrentHashMap`.

4. **Cache stampede (thundering herd)**: When a popular key expires, many threads may simultaneously miss the cache and all call the backing service. `computeIfAbsent` in `ConcurrentHashMap` is not a complete solution because the computation runs under contention. Caffeine's `AsyncLoadingCache` solves this properly with a per-key lock.

5. **Observability**: A production caching decorator should expose hit ratio, miss count, eviction count, and load time metrics via Micrometer or JMX.

---

*End of Decorator Design Pattern Guide*

---

**Related Patterns:**
- [Composite](../composite/03-composite.md) — also uses recursive composition; the difference is that Composite represents part-whole hierarchies while Decorator adds behavior
- [Proxy](../proxy/04-proxy.md) — same structure as Decorator but different intent (access control, lazy loading, remote access vs. behavior addition)
- [Chain of Responsibility](../../behavioral/chain-of-responsibility.md) — passes requests through a chain; unlike Decorator, only one handler processes the request
- [Strategy](../../behavioral/strategy.md) — swaps algorithms; unlike Decorator, does not stack behaviors and is not transparent to the caller
