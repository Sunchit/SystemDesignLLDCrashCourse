# Singleton Pattern — Complete Industry-Grade Guide

> **Course:** LLD & OOP Design Patterns Crash Course
> **Category:** Creational Patterns
> **Difficulty:** Intermediate to Advanced
> **Language:** Java

---

## Table of Contents

1. [Intent and Problem It Solves](#1-intent-and-problem-it-solves)
2. [When to Use / When NOT to Use](#2-when-to-use--when-not-to-use)
3. [UML Class Diagram](#3-uml-class-diagram)
4. [Complete Java Implementations](#4-complete-java-implementations)
   - [4a. Eager Initialization](#4a-eager-initialization)
   - [4b. Lazy Initialization — Not Thread-Safe](#4b-lazy-initialization--not-thread-safe-showing-the-problem)
   - [4c. Thread-Safe — Synchronized Method](#4c-thread-safe--synchronized-method-showing-performance-cost)
   - [4d. Double-Checked Locking with volatile](#4d-double-checked-locking-with-volatile)
   - [4e. Initialization-on-Demand Holder (Bill Pugh)](#4e-initialization-on-demand-holder-idiom-bill-pugh-singleton)
   - [4f. Enum Singleton](#4f-enum-singleton-joshua-blochs-effective-java-recommendation)
5. [Real-World Analogy](#5-real-world-analogy)
6. [Real-World Examples in the Java Ecosystem](#6-real-world-examples-in-the-java-ecosystem)
7. [Singleton vs Static Class — Comparison Table](#7-singleton-vs-static-class--comparison-table)
8. [Testability Problems and Solutions](#8-testability-problems-and-solutions)
9. [Spring Singleton Scope vs GoF Singleton](#9-springs-singleton-scope-vs-gof-singleton)
10. [Anti-Pattern Debate](#10-anti-pattern-debate)
11. [Trade-offs — Balanced Pros and Cons](#11-trade-offs--balanced-pros-and-cons)
12. [Common Pitfalls](#12-common-pitfalls)
13. [Interview Questions with Model Answers](#13-interview-questions-with-model-answers)
14. [Comparison with Related Patterns](#14-comparison-with-related-patterns)
15. [Quick Reference Cheat Sheet](#15-quick-reference-cheat-sheet)

---

## 1. Intent and Problem It Solves

### Definition

The **Singleton** is a creational design pattern that ensures a class has **only one instance** throughout the lifetime of an application and provides a **global point of access** to that instance.

It was introduced in the seminal work *Design Patterns: Elements of Reusable Object-Oriented Software* (1994) by the Gang of Four (GoF): Gamma, Helm, Johnson, and Vlissides.

### The Core Problem

Consider a system that manages a shared, expensive resource — a database connection pool, a configuration manager, a thread pool, or a logging facility. Without Singleton, every time client code needs that resource, it would call `new`:

```java
// Without Singleton — problematic scenario
public class DatabaseConnectionPool {
    private final List<Connection> connections = new ArrayList<>();

    public DatabaseConnectionPool() {
        // Expensive: opens 10 real DB connections each time!
        for (int i = 0; i < 10; i++) {
            connections.add(openNewConnection());
        }
    }
}

// Somewhere in service layer A
DatabaseConnectionPool poolA = new DatabaseConnectionPool(); // 10 connections opened

// Somewhere in service layer B — completely separate object!
DatabaseConnectionPool poolB = new DatabaseConnectionPool(); // 10 MORE connections opened
```

This causes three critical problems:

1. **Resource wastage**: Multiple instances each hold their own costly resources (open sockets, file handles, memory buffers).
2. **Inconsistent state**: Each instance operates independently. Writes to `poolA` are invisible to `poolB`. Configuration changes in one instance don't propagate to another.
3. **No coordination**: Concurrent access becomes unmanageable — two pools might hand out the same underlying connection, causing data corruption.

### The Singleton Solution

Singleton solves this by:

1. **Making the constructor private** — prevents external code from calling `new`.
2. **Holding a static reference** to the single instance inside the class itself.
3. **Providing a static factory method** (`getInstance()`) that returns the same instance every time.

```java
// Client code with Singleton
DatabaseConnectionPool pool1 = DatabaseConnectionPool.getInstance();
DatabaseConnectionPool pool2 = DatabaseConnectionPool.getInstance();

System.out.println(pool1 == pool2); // true — same object in memory
```

### Two Responsibilities (and Why That Matters)

The GoF Singleton has two distinct responsibilities baked into one class:
- **Business logic**: The actual functionality (pooling connections, managing config, etc.)
- **Instance management**: Ensuring only one instance exists

This duality is why Singleton is controversial — it violates the Single Responsibility Principle. We will revisit this in the anti-pattern debate section.

---

## 2. When to Use / When NOT to Use

### When to Use Singleton

| Scenario | Reason |
|---|---|
| Centralized configuration store | All components must read the same config values. Multiple instances would diverge. |
| Logger / Audit trail | All log messages must flow through one writer to maintain ordering and avoid interleaving. |
| Connection pool manager | Expensive to create; must be shared and coordinated. |
| Cache manager | Cache coherence requires one authority. Two caches would cause stale reads. |
| Hardware interface driver | A printer driver or serial port handler can only have one owner at a time. |
| Thread pool executor | System-wide thread budget must be managed centrally. |
| Service registry / locator | One registry to look up services; duplicates would cause confusion. |

### When NOT to Use Singleton

| Scenario | Why It's Wrong |
|---|---|
| Domain entities (User, Order, Product) | These represent data — there are many of them. Singleton would destroy the domain model. |
| Stateless utility classes | Use static methods or dependency injection instead. No shared state means no need for a single instance. |
| When you need to test in isolation | Singleton's global state makes unit tests order-dependent and impossible to reset (see Section 8). |
| When instances need different configurations | A singleton allows only one configuration. If different parts of the app need different settings, use dependency injection. |
| In a multi-classloader environment | Each classloader creates its own "singleton", giving you multiple instances in the same JVM (e.g., OSGi, certain app servers). |
| When your class is likely to be subclassed | Private constructor prevents inheritance. Subclassing patterns cannot be applied. |

---

## 3. UML Class Diagram

```
+------------------------------------------+
|              <<Singleton>>               |
|              ConfigManager               |
+------------------------------------------+
| - instance: ConfigManager  {static}      |
| - properties: Map<String, String>        |
| - configFilePath: String                 |
+------------------------------------------+
| - ConfigManager()                        |   <-- private constructor
| + getInstance(): ConfigManager  {static} |   <-- global access point
| + getProperty(key: String): String       |
| + setProperty(key: String, val: String)  |
| + reload(): void                         |
+------------------------------------------+
          |
          |  uses (returns same instance every call)
          |
   +------+------+
   |             |
+--+--+       +--+--+
| ServiceA |   | ServiceB |
+----------+   +----------+
| - config: ConfigManager |  (both hold reference to the SAME object)
```

**Reading the diagram:**

- The dashed border with `<<Singleton>>` stereotype marks it as a Singleton.
- The `-instance: ConfigManager {static}` field is static — it lives on the class, not on any object.
- The `-ConfigManager()` constructor is private (shown by `-`), blocking `new`.
- `+getInstance()` is the only way in. Its `{static}` tag means no instance is needed to call it.
- Both `ServiceA` and `ServiceB` hold a reference to the same object in heap memory.

---

## 4. Complete Java Implementations

---

### 4a. Eager Initialization

The simplest and safest form. The instance is created at class-loading time by the JVM, before any thread can call `getInstance()`.

```java
package com.course.patterns.singleton;

/**
 * EagerSingleton — created at class-load time.
 *
 * PROS:
 *   - Thread-safe: JVM class loading is guaranteed to be thread-safe.
 *   - Simple: no null checks, no synchronization.
 *   - No lazy loading complexity.
 *
 * CONS:
 *   - Instance is created even if it's never used (wasted resources if instantiation is expensive).
 *   - Cannot pass constructor arguments (no way to configure at creation time without static config).
 *   - Cannot handle checked exceptions cleanly during static initialization.
 */
public final class EagerSingleton {

    // Created when the class is first loaded by the JVM.
    // 'final' ensures the reference can never be reassigned after initialization.
    private static final EagerSingleton INSTANCE = new EagerSingleton();

    // Private constructor blocks external instantiation.
    private EagerSingleton() {
        // Guard against reflection attacks (see Section 12 — Pitfalls).
        if (INSTANCE != null) {
            throw new IllegalStateException(
                "Use EagerSingleton.getInstance() — direct instantiation is forbidden."
            );
        }
        System.out.println("EagerSingleton: constructor called once at class load.");
    }

    /**
     * Returns the single, eagerly-created instance.
     * No synchronization needed — instance exists before any thread calls this.
     *
     * @return the singleton instance
     */
    public static EagerSingleton getInstance() {
        return INSTANCE;
    }

    // --- Business logic below ---

    public void doWork() {
        System.out.println("EagerSingleton.doWork() called on instance: " + this.hashCode());
    }

    // Usage demonstration
    public static void main(String[] args) {
        EagerSingleton s1 = EagerSingleton.getInstance();
        EagerSingleton s2 = EagerSingleton.getInstance();

        System.out.println("Same instance? " + (s1 == s2)); // true
        s1.doWork();
    }
}
```

**Key insight**: The JVM specification guarantees that static fields are initialized before any thread accesses them, and that class initialization is synchronized. This makes eager initialization inherently thread-safe — no `synchronized` keyword needed.

---

### 4b. Lazy Initialization — Not Thread-Safe (Showing the Problem)

This version creates the instance only when first requested, but is **not safe in a multithreaded environment**. Study this to understand the race condition.

```java
package com.course.patterns.singleton;

/**
 * LazyUnsafeSingleton — DO NOT USE IN PRODUCTION.
 *
 * Demonstrates the classic race condition that occurs without synchronization.
 *
 * RACE CONDITION SCENARIO:
 *   Thread A calls getInstance(). Checks: instance == null → true. Begins new LazyUnsafeSingleton().
 *   Thread B calls getInstance() before Thread A finishes. Checks: instance == null → STILL true!
 *   Thread B also creates a new instance.
 *   Result: TWO instances exist — Singleton contract is broken.
 */
public final class LazyUnsafeSingleton {

    // volatile NOT used here intentionally — to show the problem.
    private static LazyUnsafeSingleton instance;

    private LazyUnsafeSingleton() {
        System.out.println("LazyUnsafeSingleton constructor called by thread: "
            + Thread.currentThread().getName());
    }

    /**
     * NOT THREAD-SAFE.
     * The check-then-act (instance == null → create) is not atomic.
     * Two threads can simultaneously pass the null check and both create instances.
     */
    public static LazyUnsafeSingleton getInstance() {
        if (instance == null) {                        // (1) Thread A checks: null → true
            // ← Thread B can enter here before Thread A finishes (2)
            instance = new LazyUnsafeSingleton();      // (3) Both threads create separate objects
        }
        return instance;
    }

    /**
     * Demonstrates the race condition using real threads.
     * Run this multiple times — occasionally you'll see the constructor printed twice.
     */
    public static void main(String[] args) throws InterruptedException {
        Runnable task = () -> {
            LazyUnsafeSingleton s = LazyUnsafeSingleton.getInstance();
            System.out.println(Thread.currentThread().getName()
                + " got instance with hashCode: " + s.hashCode());
        };

        Thread t1 = new Thread(task, "Thread-1");
        Thread t2 = new Thread(task, "Thread-2");
        Thread t3 = new Thread(task, "Thread-3");

        // Start all threads near-simultaneously to maximize race condition probability.
        t1.start(); t2.start(); t3.start();
        t1.join();  t2.join();  t3.join();

        // If constructor prints more than once, the race condition manifested.
    }
}
```

**The race condition visualized:**

```
Time →
Thread A: [ check null: TRUE ]--[ creating instance ]---[ assign instance ]
Thread B:           [ check null: TRUE ]---[ creating instance ]---[ assign instance ]
                         ^
                    Gap where B slips through before A assigns
```

---

### 4c. Thread-Safe — Synchronized Method (Showing Performance Cost)

Adding `synchronized` to `getInstance()` fixes the race condition but introduces a serialization bottleneck.

```java
package com.course.patterns.singleton;

/**
 * SynchronizedSingleton — thread-safe but potentially slow under high contention.
 *
 * Every call to getInstance() acquires a lock on the class object,
 * even after the instance has already been created. In a high-throughput
 * system where getInstance() is called millions of times per second
 * (e.g., a logger), this mutex becomes a bottleneck.
 *
 * Benchmark insight: synchronized method can be 100x slower than
 * an unsynchronized read in a heavily contended scenario.
 */
public final class SynchronizedSingleton {

    private static SynchronizedSingleton instance;

    private SynchronizedSingleton() {
        System.out.println("SynchronizedSingleton: constructor called by "
            + Thread.currentThread().getName());
    }

    /**
     * Thread-safe via method-level synchronization.
     *
     * Only one thread can execute this method at a time.
     * After first creation, every subsequent caller still waits for the lock
     * even though instance != null — this is the unnecessary overhead.
     *
     * @return the singleton instance
     */
    public static synchronized SynchronizedSingleton getInstance() {
        if (instance == null) {
            instance = new SynchronizedSingleton();
        }
        return instance;
    }

    // --- Performance measurement demonstration ---

    public static void main(String[] args) throws InterruptedException {
        int threadCount = 100;
        int callsPerThread = 10_000;

        Thread[] threads = new Thread[threadCount];
        long startTime = System.nanoTime();

        for (int i = 0; i < threadCount; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < callsPerThread; j++) {
                    SynchronizedSingleton.getInstance(); // Lock acquired and released 1M times total
                }
            });
        }

        for (Thread t : threads) t.start();
        for (Thread t : threads) t.join();

        long elapsed = System.nanoTime() - startTime;
        System.out.printf("Total time for %d calls: %.2f ms%n",
            threadCount * callsPerThread, elapsed / 1_000_000.0);

        // Compare this with the DCL version (4d) to see the speedup.
    }
}
```

**Why synchronization on `getInstance()` is excessive**: After the first invocation, `instance` is never `null` again. Yet every thread still serializes through the lock to read a non-null reference — work that could be done with a simple volatile read.

---

### 4d. Double-Checked Locking with volatile

The industry-standard approach for lazy-initialized singletons that balances thread safety and performance.

```java
package com.course.patterns.singleton;

/**
 * DoubleCheckedLockingSingleton — the classic, efficient, thread-safe lazy singleton.
 *
 * The pattern:
 *   1. First check (outside synchronized): fast path for already-initialized case.
 *      Most calls after initialization pass through here with zero locking overhead.
 *   2. Synchronized block: entered only when instance MIGHT be null.
 *   3. Second check (inside synchronized): ensures only one thread actually creates.
 *
 * CRITICAL: The 'volatile' keyword is MANDATORY.
 *
 * Without volatile, the CPU or JVM may reorder instructions inside the constructor.
 * Object creation (instance = new DCLSingleton()) is NOT atomic — it involves:
 *   Step 1: Allocate memory for the object.
 *   Step 2: Invoke the constructor (initialize fields).
 *   Step 3: Assign the memory address to 'instance'.
 *
 * Without volatile, Step 3 can be reordered before Step 2.
 * A second thread seeing a non-null 'instance' might read a partially constructed object.
 * volatile prohibits this reordering — it establishes a happens-before relationship.
 *
 * DCL was BROKEN before Java 5. The Java Memory Model fix in JSR-133 (Java 5+)
 * combined with volatile makes it correct. Do not use on Java 1.4 or earlier.
 */
public final class DoubleCheckedLockingSingleton {

    // volatile ensures visibility and ordering guarantees across threads.
    private static volatile DoubleCheckedLockingSingleton instance;

    // Example state to show a non-trivial singleton
    private final String configFilePath;
    private volatile boolean initialized = false;

    private DoubleCheckedLockingSingleton() {
        // Simulate expensive initialization
        this.configFilePath = System.getProperty("app.config", "/etc/app/config.properties");
        this.initialized = true;
        System.out.println("DoubleCheckedLockingSingleton initialized by: "
            + Thread.currentThread().getName());
    }

    /**
     * Returns the singleton instance.
     * Thread-safe with minimal synchronization overhead.
     *
     * @return the singleton instance, never null
     */
    public static DoubleCheckedLockingSingleton getInstance() {
        // First check — no lock. Fast path for 99.999% of calls after initialization.
        if (instance == null) {
            // Lock acquired only when instance might be null.
            synchronized (DoubleCheckedLockingSingleton.class) {
                // Second check — inside lock. Only one thread creates.
                if (instance == null) {
                    instance = new DoubleCheckedLockingSingleton();
                }
            }
        }
        return instance;
    }

    public String getConfigFilePath() {
        return configFilePath;
    }

    public boolean isInitialized() {
        return initialized;
    }

    // --- Thread-safety demonstration ---

    public static void main(String[] args) throws InterruptedException {
        int threadCount = 50;
        Thread[] threads = new Thread[threadCount];

        for (int i = 0; i < threadCount; i++) {
            threads[i] = new Thread(() -> {
                DoubleCheckedLockingSingleton s = DoubleCheckedLockingSingleton.getInstance();
                System.out.println(Thread.currentThread().getName()
                    + " → hashCode: " + s.hashCode()
                    + " | initialized: " + s.isInitialized());
            }, "Worker-" + i);
        }

        for (Thread t : threads) t.start();
        for (Thread t : threads) t.join();

        System.out.println("All threads done. Constructor was called exactly once.");
    }
}
```

**Memory model guarantee**: `volatile` on `instance` means that when Thread B reads `instance != null`, it is guaranteed to also see all writes that Thread A made inside the constructor before assigning to `instance`. This is the *happens-before* relationship mandated by JSR-133.

---

### 4e. Initialization-on-Demand Holder Idiom (Bill Pugh Singleton)

Leverages the JVM's class-loading guarantees to achieve lazy, thread-safe initialization without any `volatile` or `synchronized` keywords. Widely considered the most elegant Java-specific approach.

```java
package com.course.patterns.singleton;

import java.util.Properties;
import java.io.InputStream;
import java.io.IOException;

/**
 * BillPughSingleton — Initialization-on-Demand Holder Idiom.
 *
 * How it works:
 *   The JVM loads a class only when it is first actively used.
 *   The inner static class 'SingletonHolder' is NOT loaded when BillPughSingleton
 *   is first loaded — it's loaded only when getInstance() is called and the
 *   JVM resolves the reference to SingletonHolder.INSTANCE.
 *
 *   The JVM specification (§12.4) guarantees:
 *     - A class initializer runs exactly once.
 *     - Class initialization is synchronized by the JVM.
 *     - All threads see the fully-initialized class after initialization.
 *
 *   This gives us:
 *     - Lazy initialization (holder class not loaded until needed).
 *     - Thread safety (JVM handles synchronization during class loading).
 *     - No performance overhead (no synchronized, no volatile on INSTANCE read).
 *
 * This is the RECOMMENDED approach for most production scenarios
 * where Enum Singleton is not applicable (e.g., need lazy init with complex setup).
 */
public final class BillPughSingleton {

    private final Properties appProperties;
    private final long creationTimeMs;

    private BillPughSingleton() {
        this.creationTimeMs = System.currentTimeMillis();
        this.appProperties = loadProperties();
        System.out.println("BillPughSingleton: heavy initialization done at " + creationTimeMs);
    }

    /**
     * Inner static holder class.
     * Not loaded until getInstance() is called for the first time.
     * JVM guarantees thread-safe, once-only initialization of static fields.
     */
    private static final class SingletonHolder {
        // This is initialized exactly once, by the class loader,
        // before any thread can access it via getInstance().
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();

        // Prevent instantiation of the holder itself.
        private SingletonHolder() {
            throw new UnsupportedOperationException("Holder class must not be instantiated.");
        }
    }

    /**
     * Triggers loading of SingletonHolder on first call.
     * Subsequent calls return the already-initialized INSTANCE with no overhead.
     *
     * @return the singleton instance, guaranteed non-null
     */
    public static BillPughSingleton getInstance() {
        return SingletonHolder.INSTANCE;
    }

    /**
     * Retrieves a configuration property.
     *
     * @param key the property key
     * @return the property value, or null if not found
     */
    public String getProperty(String key) {
        return appProperties.getProperty(key);
    }

    /**
     * Retrieves a configuration property with a fallback default.
     *
     * @param key          the property key
     * @param defaultValue returned if key is absent
     * @return the property value or defaultValue
     */
    public String getProperty(String key, String defaultValue) {
        return appProperties.getProperty(key, defaultValue);
    }

    public long getCreationTimeMs() {
        return creationTimeMs;
    }

    private Properties loadProperties() {
        Properties props = new Properties();
        // Try to load from classpath; fall back to defaults gracefully.
        try (InputStream is = getClass().getClassLoader()
                .getResourceAsStream("application.properties")) {
            if (is != null) {
                props.load(is);
            } else {
                System.out.println("Warning: application.properties not found on classpath. Using defaults.");
                props.setProperty("db.url", "jdbc:h2:mem:default");
                props.setProperty("app.env", "development");
            }
        } catch (IOException e) {
            System.err.println("Failed to load properties: " + e.getMessage());
            // Non-fatal: continue with empty/default properties.
        }
        return props;
    }

    public static void main(String[] args) throws InterruptedException {
        System.out.println("Main started — BillPughSingleton class loaded but holder NOT yet loaded.");

        // Simulate concurrent first-access
        Runnable task = () -> {
            BillPughSingleton config = BillPughSingleton.getInstance();
            System.out.println(Thread.currentThread().getName()
                + " | env=" + config.getProperty("app.env", "unknown")
                + " | instance=" + config.hashCode());
        };

        Thread t1 = new Thread(task, "Thread-Alpha");
        Thread t2 = new Thread(task, "Thread-Beta");
        t1.start(); t2.start();
        t1.join();  t2.join();

        // hashCode will be the same for both threads.
    }
}
```

**Why this is better than DCL**: No `volatile` read on every `getInstance()` call. The JVM's class-loading synchronization does the heavy lifting once, and then INSTANCE reads are plain, unsynchronized static field reads — the fastest possible access pattern.

---

### 4f. Enum Singleton (Joshua Bloch's Effective Java Recommendation)

The most concise and bulletproof approach. Handles serialization and reflection attacks automatically. From *Effective Java, 3rd Edition*, Item 3: "A single-element enum type is often the best way to implement a singleton."

```java
package com.course.patterns.singleton;

import java.io.Serializable;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.logging.Level;
import java.util.logging.Logger;

/**
 * EnumSingleton — the Joshua Bloch recommendation.
 *
 * Guarantees:
 *   1. THREAD SAFETY: Enum initialization is synchronized by the JVM.
 *   2. SERIALIZATION SAFETY: Enum serialization is handled by the JVM.
 *      Deserializing an enum always returns the existing enum constant —
 *      Java's serialization mechanism guarantees this via Enum.valueOf().
 *      You do NOT need to implement readResolve().
 *   3. REFLECTION SAFETY: The JVM blocks reflective instantiation of enums.
 *      Constructor.newInstance() on an enum constant throws IllegalArgumentException.
 *   4. SIMPLICITY: 3 lines of "ceremony" vs 20+ for DCL.
 *
 * LIMITATION:
 *   - Cannot be lazily initialized (enum constants are initialized at class-load time).
 *   - Cannot extend another class (enums implicitly extend java.lang.Enum).
 *   - May feel semantically odd to developers unfamiliar with the pattern.
 */
public enum EnumSingleton {

    INSTANCE; // The single element — this IS the singleton.

    // Instance state — enums can have fields and methods just like classes.
    private static final Logger LOGGER = Logger.getLogger(EnumSingleton.class.getName());
    private final AtomicInteger requestCount;
    private volatile String serviceEndpoint;

    /**
     * Enum constructor — called by the JVM once, thread-safely, when the enum is loaded.
     * Enum constructors are always package-private or private.
     */
    EnumSingleton() {
        this.requestCount = new AtomicInteger(0);
        this.serviceEndpoint = System.getProperty("service.endpoint", "https://api.example.com/v1");
        LOGGER.info("EnumSingleton initialized with endpoint: " + serviceEndpoint);
    }

    // --- Business Methods ---

    /**
     * Processes a request and increments the counter.
     *
     * @param requestId the unique identifier for this request
     * @return processed request count
     */
    public int processRequest(String requestId) {
        int count = requestCount.incrementAndGet();
        LOGGER.log(Level.FINE, "Processing request {0}, total count: {1}",
            new Object[]{requestId, count});
        return count;
    }

    /**
     * @return total number of requests processed since startup
     */
    public int getRequestCount() {
        return requestCount.get();
    }

    /**
     * @return the configured service endpoint
     */
    public String getServiceEndpoint() {
        return serviceEndpoint;
    }

    /**
     * Updates the service endpoint (e.g., during blue-green deployment).
     *
     * @param endpoint the new endpoint URL
     */
    public void setServiceEndpoint(String endpoint) {
        if (endpoint == null || endpoint.isBlank()) {
            throw new IllegalArgumentException("Endpoint must not be null or blank.");
        }
        this.serviceEndpoint = endpoint;
        LOGGER.info("Service endpoint updated to: " + endpoint);
    }

    /**
     * Demonstrates that reflection cannot break enum singletons.
     */
    public static void demonstrateReflectionSafety() {
        try {
            // Attempt to reflectively instantiate the enum
            java.lang.reflect.Constructor<EnumSingleton> constructor =
                EnumSingleton.class.getDeclaredConstructor(String.class, int.class);
            constructor.setAccessible(true);
            EnumSingleton reflectedInstance = constructor.newInstance("FAKE", 99);
            System.out.println("Reflection succeeded (should NOT happen): " + reflectedInstance);
        } catch (IllegalArgumentException e) {
            System.out.println("Reflection blocked correctly: " + e.getMessage());
            // Expected output: "Cannot reflectively create enum objects"
        } catch (Exception e) {
            System.out.println("Reflection blocked with: " + e.getClass().getSimpleName());
        }
    }

    public static void main(String[] args) {
        // Access via enum constant name
        EnumSingleton s1 = EnumSingleton.INSTANCE;
        EnumSingleton s2 = EnumSingleton.INSTANCE;

        System.out.println("Same instance? " + (s1 == s2));    // true
        System.out.println("Requests: " + s1.processRequest("REQ-001"));
        System.out.println("Requests: " + s2.processRequest("REQ-002")); // Counter shared
        System.out.println("Total: " + EnumSingleton.INSTANCE.getRequestCount()); // 2

        demonstrateReflectionSafety();
    }
}
```

**Why it is so robust**: The JVM was specifically designed to prevent duplicate enum constants. The Java Language Specification (JLS §8.9) states that it is a compile-time error to attempt to explicitly instantiate an enum, and the reflection API throws `IllegalArgumentException` if you try via reflection.

---

## 5. Real-World Analogy

### The Country's Central Bank

Think of a country's central bank (e.g., the Reserve Bank of India, or the Federal Reserve).

- There is **exactly one** central bank per country. The economy cannot function with two competing monetary authorities simultaneously printing currency and setting interest rates independently.
- You do not create a new central bank every time you want to know the interest rate — you go to **the** central bank.
- It maintains **shared state** — the total money supply, foreign exchange reserves, prime lending rate — that all commercial banks, businesses, and citizens must agree upon.
- It has a **global access point** — its official website, phone number, physical address. Everyone uses the same channel to interact with it.
- It is **expensive to create** — requires legislation, infrastructure, staff, international recognition. You cannot spin one up casually.

This mirrors exactly what Singleton does in software: one authoritative instance, global access, shared state, expensive initialization done once.

### Other Everyday Analogies

| Real World | Software Equivalent |
|---|---|
| Government (one per country) | Application-level config manager |
| President / Prime Minister | System-wide scheduler or thread pool |
| Phone book (one per city, in the old days) | Service registry |
| National power grid | Connection pool |
| DNA in a cell (one nucleus per cell) | Per-process runtime environment |

---

## 6. Real-World Examples in the Java Ecosystem

### 6.1 java.lang.Runtime

```java
package com.course.patterns.singleton.examples;

/**
 * java.lang.Runtime is the canonical JDK example of Singleton.
 * It represents the runtime environment of the running JVM process.
 *
 * The JDK implementation (simplified):
 *
 *   public class Runtime {
 *       private static final Runtime currentRuntime = new Runtime();  // eager
 *
 *       private Runtime() {}
 *
 *       public static Runtime getRuntime() {
 *           return currentRuntime;
 *       }
 *
 *       public long totalMemory()  { ... }
 *       public long freeMemory()   { ... }
 *       public int  availableProcessors() { ... }
 *       public void addShutdownHook(Thread hook) { ... }
 *   }
 */
public class RuntimeSingletonDemo {

    public static void main(String[] args) throws Exception {
        Runtime rt1 = Runtime.getRuntime();
        Runtime rt2 = Runtime.getRuntime();

        System.out.println("Same instance: " + (rt1 == rt2));     // true
        System.out.printf("Available processors: %d%n", rt1.availableProcessors());
        System.out.printf("Total JVM memory: %,d bytes%n", rt1.totalMemory());
        System.out.printf("Free JVM memory:  %,d bytes%n", rt1.freeMemory());
        System.out.printf("Max JVM memory:   %,d bytes%n", rt1.maxMemory());

        // Execute a process via the singleton Runtime
        Process process = rt1.exec(new String[]{"echo", "Hello from Runtime singleton"});
        System.out.println("Process exit code: " + process.waitFor());

        // Register a shutdown hook — all components share the same Runtime
        rt2.addShutdownHook(new Thread(() ->
            System.out.println("JVM shutting down — cleanup via Runtime singleton.")));
    }
}
```

### 6.2 java.awt.Desktop

```java
package com.course.patterns.singleton.examples;

import java.awt.Desktop;
import java.net.URI;

/**
 * java.awt.Desktop is another JDK Singleton.
 * Represents the system desktop, providing a way to launch associated applications
 * for files, URIs, and mail. Only one Desktop object can exist per application.
 *
 * Desktop.getDesktop() throws UnsupportedOperationException if the current
 * platform does not support it — demonstrating that Singletons can encode
 * platform-specific constraints.
 */
public class DesktopSingletonDemo {

    public static void main(String[] args) {
        if (!Desktop.isDesktopSupported()) {
            System.err.println("Desktop API not supported on this platform.");
            return;
        }

        Desktop desktop1 = Desktop.getDesktop();
        Desktop desktop2 = Desktop.getDesktop();

        System.out.println("Same Desktop instance: " + (desktop1 == desktop2)); // true

        try {
            // Open a URL in the system's default browser
            desktop1.browse(new URI("https://docs.oracle.com/en/java/"));
        } catch (Exception e) {
            System.err.println("Could not open browser: " + e.getMessage());
        }
    }
}
```

### 6.3 Spring Framework — Singleton-Scoped Beans

```java
package com.course.patterns.singleton.examples;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

/**
 * Spring's default bean scope is 'singleton' — one instance per ApplicationContext.
 *
 * Key difference from GoF Singleton:
 *   GoF Singleton = one per JVM ClassLoader.
 *   Spring Singleton = one per ApplicationContext (IoC container).
 *   You CAN have multiple ApplicationContexts in the same JVM,
 *   each with their own "singleton" bean instance.
 */

// Configuration class defining beans
@Configuration
class AppConfig {

    /**
     * This bean is singleton-scoped by default.
     * Spring creates exactly one instance per ApplicationContext.
     */
    @Bean
    // @Scope("singleton") — not needed, it's the default
    public DatabaseService databaseService() {
        return new DatabaseService("jdbc:postgresql://localhost:5432/mydb");
    }

    /**
     * For contrast: a prototype-scoped bean.
     * Spring creates a NEW instance every time it is requested.
     */
    @Bean
    @Scope("prototype")
    public RequestProcessor requestProcessor() {
        return new RequestProcessor();
    }
}

@Component
class DatabaseService {
    private final String jdbcUrl;

    public DatabaseService(String jdbcUrl) {
        this.jdbcUrl = jdbcUrl;
        System.out.println("DatabaseService created: " + this.hashCode());
    }

    public String getJdbcUrl() { return jdbcUrl; }
}

@Component
class RequestProcessor {
    public RequestProcessor() {
        System.out.println("RequestProcessor created: " + this.hashCode());
    }
    public void process(String request) {
        System.out.println("Processing: " + request + " on instance " + this.hashCode());
    }
}

@Service
class OrderService {
    private final DatabaseService dbService;

    @Autowired
    public OrderService(DatabaseService dbService) {
        this.dbService = dbService;
    }

    public void placeOrder(String orderId) {
        System.out.println("Placing order " + orderId + " via db: " + dbService.hashCode());
    }
}

@Service
class PaymentService {
    private final DatabaseService dbService;

    @Autowired
    public PaymentService(DatabaseService dbService) {
        this.dbService = dbService;
    }

    public void processPayment(String paymentId) {
        System.out.println("Processing payment " + paymentId + " via db: " + dbService.hashCode());
    }
}

public class SpringSingletonDemo {

    public static void main(String[] args) {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);

        // Both services receive the SAME DatabaseService instance (singleton scope)
        OrderService orderSvc   = ctx.getBean(OrderService.class);
        PaymentService paySvc   = ctx.getBean(PaymentService.class);

        // Retrieve DatabaseService directly twice — same instance
        DatabaseService db1 = ctx.getBean(DatabaseService.class);
        DatabaseService db2 = ctx.getBean(DatabaseService.class);
        System.out.println("DB singleton same instance: " + (db1 == db2)); // true

        // Retrieve prototype bean twice — different instances
        RequestProcessor rp1 = ctx.getBean(RequestProcessor.class);
        RequestProcessor rp2 = ctx.getBean(RequestProcessor.class);
        System.out.println("Processor prototype same instance: " + (rp1 == rp2)); // false

        orderSvc.placeOrder("ORD-100");
        paySvc.processPayment("PAY-200");

        ((AnnotationConfigApplicationContext) ctx).close();
    }
}
```

---

## 7. Singleton vs Static Class — Comparison Table

Both Singleton and static classes provide a single point of access. Understanding the differences is critical for interviews.

| Dimension | Singleton | Static Class |
|---|---|---|
| **OOP Compliance** | Full object — implements interfaces, participates in polymorphism | Not an object; static methods cannot implement interfaces |
| **Inheritance** | Can implement interfaces; limited inheritance (subclassing blocked by private constructor, but workarounds exist) | Cannot be inherited; `final` implicitly |
| **Lazy Initialization** | Yes — instance created on first `getInstance()` call | No — static initializer runs at class load time |
| **Serialization** | Can implement `Serializable` (with care); Enum handles it natively | Not serializable |
| **Testability** | Instance can be mocked via interfaces or dependency injection | Static methods cannot be mocked without bytecode tools (PowerMock) |
| **Reflection** | Reflection can break it (mitigated by Enum form) | Not applicable |
| **State management** | Instance fields hold state; controlled lifecycle | Static fields hold state; cleared only on class unloading |
| **Thread safety** | Must be explicitly designed (volatile, synchronized, holder, enum) | Static initializer is thread-safe; individual method calls may not be |
| **Memory** | One object on the heap | No heap object; methods and fields live in method area |
| **Overriding behavior** | Possible via interfaces and delegation | Impossible without changing the class |
| **IoC / DI compatibility** | Yes — DI containers manage singletons natively | No — static classes bypass the DI container |
| **Example** | `Runtime.getRuntime()` | `java.lang.Math` |

**Rule of thumb**: If you need state, polymorphism, testability, or lifecycle control — use Singleton (or better, DI-managed singleton). If you need a stateless collection of utility functions — use a static class.

---

## 8. Testability Problems and Solutions

### 8.1 The Problem — Global State Poisons Tests

```java
package com.course.patterns.singleton.testing;

/**
 * Demonstrates why Singleton makes testing painful.
 *
 * This is NOT a pattern to follow — it's the PROBLEM to recognize.
 */
public class GlobalStateProblems {

    // Bad singleton — tightly coupled global state
    public static class BadConfigSingleton {
        private static BadConfigSingleton instance;
        private String dbUrl = "jdbc:h2:mem:default";

        private BadConfigSingleton() {}

        public static BadConfigSingleton getInstance() {
            if (instance == null) instance = new BadConfigSingleton();
            return instance;
        }

        public String getDbUrl() { return dbUrl; }
        public void setDbUrl(String url) { this.dbUrl = url; }
    }

    // Service that uses the singleton directly — hard to test!
    public static class UserRepository {
        public String getConnectionString() {
            // Direct coupling to global singleton — cannot be substituted in tests
            return BadConfigSingleton.getInstance().getDbUrl();
        }
    }

    /*
     * Problems in JUnit:
     *
     * @Test
     * void testWithProductionDb() {
     *     BadConfigSingleton.getInstance().setDbUrl("jdbc:postgresql://prod:5432/db");
     *     // Test runs...
     * }
     *
     * @Test
     * void testWithTestDb() {
     *     // BAD: depends on test execution ORDER.
     *     // If testWithProductionDb ran first, the singleton still has the prod URL!
     *     BadConfigSingleton.getInstance().setDbUrl("jdbc:h2:mem:test");
     *     // ... test may pass or fail depending on execution order
     * }
     *
     * This is called "test pollution" — one test's side effects affect another's.
     */
}
```

### 8.2 Solution 1 — Program to an Interface

```java
package com.course.patterns.singleton.testing;

/**
 * Solution: Extract an interface, inject the dependency.
 * The singleton still exists, but it's no longer the direct dependency.
 */

// Step 1: Define the contract as an interface
interface ConfigProvider {
    String getDbUrl();
    String getProperty(String key, String defaultValue);
}

// Step 2: Real singleton implements the interface
class ProductionConfigProvider implements ConfigProvider {

    private static volatile ProductionConfigProvider instance;

    private ProductionConfigProvider() {
        // Real initialization logic
    }

    public static ProductionConfigProvider getInstance() {
        if (instance == null) {
            synchronized (ProductionConfigProvider.class) {
                if (instance == null) {
                    instance = new ProductionConfigProvider();
                }
            }
        }
        return instance;
    }

    @Override
    public String getDbUrl() {
        return System.getProperty("db.url", "jdbc:postgresql://localhost:5432/prod");
    }

    @Override
    public String getProperty(String key, String defaultValue) {
        return System.getProperty(key, defaultValue);
    }
}

// Step 3: Service depends on the interface, not the concrete singleton
class TestableUserRepository {

    private final ConfigProvider config;

    // Constructor injection — the golden standard
    public TestableUserRepository(ConfigProvider config) {
        this.config = config;
    }

    public String getConnectionString() {
        return config.getDbUrl();
    }

    public String getDataCenter() {
        return config.getProperty("datacenter", "us-west-2");
    }
}

// Step 4: Test with a stub — no singleton involved
class StubConfigProvider implements ConfigProvider {
    private final String dbUrl;

    public StubConfigProvider(String dbUrl) {
        this.dbUrl = dbUrl;
    }

    @Override
    public String getDbUrl() { return dbUrl; }

    @Override
    public String getProperty(String key, String defaultValue) { return defaultValue; }
}

/**
 * Now tests are isolated, order-independent, and fast.
 *
 * Equivalent JUnit 5 tests:
 *
 * class UserRepositoryTest {
 *
 *     @Test
 *     void testGetConnectionString_returnsInjectedUrl() {
 *         ConfigProvider stub = new StubConfigProvider("jdbc:h2:mem:test");
 *         TestableUserRepository repo = new TestableUserRepository(stub);
 *         assertEquals("jdbc:h2:mem:test", repo.getConnectionString());
 *     }
 *
 *     @Test
 *     void testGetConnectionString_withDifferentUrl() {
 *         ConfigProvider stub = new StubConfigProvider("jdbc:mysql://testhost/db");
 *         TestableUserRepository repo = new TestableUserRepository(stub);
 *         assertEquals("jdbc:mysql://testhost/db", repo.getConnectionString());
 *     }
 * }
 */
public class TestableDesignDemo {
    public static void main(String[] args) {
        // Production wiring
        TestableUserRepository prodRepo =
            new TestableUserRepository(ProductionConfigProvider.getInstance());

        // Test wiring — no global state involved
        TestableUserRepository testRepo =
            new TestableUserRepository(new StubConfigProvider("jdbc:h2:mem:unit_test"));

        System.out.println("Prod DB: " + prodRepo.getConnectionString());
        System.out.println("Test DB: " + testRepo.getConnectionString());
    }
}
```

### 8.3 Solution 2 — Registry Pattern for Reset

```java
package com.course.patterns.singleton.testing;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Supplier;

/**
 * Registry pattern: allows substituting or resetting singleton instances.
 * Useful when migrating legacy code that cannot immediately adopt full DI.
 *
 * @param <T> the type managed by this registry
 */
public class SingletonRegistry<T> {

    private static final Map<Class<?>, Object> registry = new ConcurrentHashMap<>();

    /**
     * Registers a supplier for the given type.
     * Used in production: call once at startup.
     * Used in tests: override with a stub supplier.
     *
     * @param type     the class key
     * @param supplier provides the instance on first access
     * @param <T>      the type
     */
    @SuppressWarnings("unchecked")
    public static <T> void register(Class<T> type, Supplier<T> supplier) {
        registry.put(type, supplier);
    }

    /**
     * Retrieves the registered instance for the given type,
     * creating it via the supplier if not yet created.
     *
     * @param type the class key
     * @param <T>  the type
     * @return the singleton instance
     * @throws IllegalStateException if no supplier is registered for the type
     */
    @SuppressWarnings("unchecked")
    public static <T> T get(Class<T> type) {
        Object entry = registry.get(type);
        if (entry == null) {
            throw new IllegalStateException("No registration found for type: " + type.getName()
                + ". Call SingletonRegistry.register() before use.");
        }
        if (entry instanceof Supplier) {
            T instance = ((Supplier<T>) entry).get();
            registry.put(type, instance); // Replace supplier with instance
            return instance;
        }
        return (T) entry;
    }

    /**
     * Clears all registrations — call in @AfterEach in tests to reset state.
     */
    public static void reset() {
        registry.clear();
    }
}
```

---

## 9. Spring's Singleton Scope vs GoF Singleton

This distinction is one of the most commonly misunderstood topics in Java interviews.

```
╔══════════════════════════════════════╦══════════════════════════════════════╗
║        GoF Singleton                 ║     Spring Singleton Scope           ║
╠══════════════════════════════════════╬══════════════════════════════════════╣
║ One instance per ClassLoader         ║ One instance per ApplicationContext  ║
╠══════════════════════════════════════╬══════════════════════════════════════╣
║ Constructor is private               ║ Constructor is public (Spring calls  ║
║ Cannot be injected                   ║ it); fully DI-compatible             ║
╠══════════════════════════════════════╬══════════════════════════════════════╣
║ Provides its own global access point ║ Spring container manages lifecycle;  ║
║ via static getInstance()             ║ access via @Autowired or getBean()   ║
╠══════════════════════════════════════╬══════════════════════════════════════╣
║ Singleton is enforced by the class   ║ Singleness is enforced by the        ║
║ itself (private constructor)         ║ container; class has no constraint   ║
╠══════════════════════════════════════╬══════════════════════════════════════╣
║ Two ClassLoaders = two instances     ║ Two ApplicationContexts = two        ║
║ (breaks singleton guarantee)         ║ instances (by design, not a bug)     ║
╠══════════════════════════════════════╬══════════════════════════════════════╣
║ Hard to test (tight coupling to      ║ Easily testable — mock the bean,     ║
║ static getInstance())                ║ override with @Primary or @TestBean  ║
╠══════════════════════════════════════╬══════════════════════════════════════╣
║ Example: Runtime.getRuntime()        ║ Example: @Service UserService        ║
╚══════════════════════════════════════╩══════════════════════════════════════╝
```

**Practical implication**: In a Spring Boot application, you rarely write GoF Singletons. Instead, you annotate classes with `@Component`, `@Service`, `@Repository`, or `@Bean` and let Spring manage the one-instance-per-context lifecycle. This gives you Singleton semantics with full testability and DI support.

```java
// Spring singleton — NOT a GoF Singleton.
// Multiple tests can create separate ApplicationContexts, each with their own instance.
@Service
public class AuditService {
    // No private constructor. No getInstance(). Spring handles it all.
    public void log(String event) { /* ... */ }
}
```

---

## 10. Anti-Pattern Debate

### Arguments FOR Singleton

1. **Guaranteed single instance**: Critical for hardware drivers, connection pools, or any resource where concurrent independent initialization would corrupt system state.
2. **Lazy initialization with expensive resources**: If initializing a resource costs 5 seconds and most runs don't need it, Singleton (Bill Pugh form) avoids that cost entirely.
3. **Well-understood**: Every Java developer recognizes the pattern. The getInstance() convention is universally understood.
4. **Appropriate for framework internals**: Spring itself uses Singletons internally (BeanFactory, ApplicationContext). Low-level frameworks need tight instance control that DI containers cannot easily provide.

### Arguments AGAINST Singleton (the Anti-Pattern Perspective)

1. **Violates Single Responsibility Principle**: The class manages both its own instantiation (a lifecycle concern) and its business logic. These are two separate reasons to change.
2. **Global mutable state**: A Singleton with mutable fields is effectively a global variable — the most reviled construct in software engineering. Global state makes programs hard to reason about, test, and parallelize.
3. **Hidden coupling**: When `ServiceA` calls `ConfigManager.getInstance()`, the dependency is invisible — it doesn't appear in `ServiceA`'s constructor or method signatures. Code review, refactoring, and dependency analysis are all harder.
4. **Breaks testability**: Test isolation requires the ability to substitute implementations. A self-managing singleton prevents this without bytecode manipulation.
5. **Classloader problems**: In OSGi environments, application servers (Tomcat redeployment), and some plugin frameworks, a Singleton can exist multiple times in the same JVM — one per classloader. This breaks the fundamental guarantee.
6. **Concurrency complexity**: Every non-trivial implementation requires careful thought about volatile, happens-before, and memory barriers. Getting it wrong is silent — there's no compile-time warning.
7. **Serialization pitfalls**: Non-enum Singletons can be cloned via deserialization (see Section 12).

### The Balanced Verdict

Singleton is not inherently evil — the evil lies in **misapplication**. The pattern is appropriate when:
- The resource genuinely must be unique (not just convenient to be unique).
- The resource is expensive to create and shared system-wide.
- You are writing framework code or low-level infrastructure.

It should be avoided when:
- You are writing business/domain logic.
- Testability is required.
- A DI container (Spring, Guice, CDI) is available to manage the lifecycle instead.

The modern consensus: **Prefer DI-managed singletons. Reserve GoF Singleton for cases where DI is unavailable or the resource must enforce its own uniqueness guarantee.**

---

## 11. Trade-offs — Balanced Pros and Cons

### Pros

| Pro | Explanation |
|---|---|
| Controlled access to shared resource | Centralizes management of resources that would be dangerous to duplicate |
| Reduced memory footprint | One instance vs N instances — significant savings for large objects |
| Global access | Any code in the system can reach the instance without passing it through layers |
| Lazy initialization possible | Bill Pugh and DCL forms defer expensive init until first use |
| Thread-safe options exist | Several well-tested, proven implementations (Enum, Bill Pugh) |
| Namespace-free | No need for global variables — the class itself serves as the namespace |

### Cons

| Con | Explanation |
|---|---|
| Global state | Shared mutable state introduces invisible coupling and hard-to-debug race conditions |
| Violates SRP | Two responsibilities: business logic + instance management |
| Difficult to mock | Static `getInstance()` calls cannot be intercepted by standard mock frameworks |
| Test order dependence | Shared state leaks between tests; one test's mutation affects the next |
| Classloader issues | Multiple classloaders in the same JVM = multiple "singleton" instances |
| Serialization vulnerabilities | Non-enum forms can be duplicated via Java serialization (requires extra code to prevent) |
| Reflection attacks | `Constructor.setAccessible(true)` can bypass private constructor (requires extra code) |
| Hides dependencies | Callers don't declare their need for the singleton; it's a hidden assumption |
| Concurrency complexity | Easy to get DCL wrong (pre-Java 5 broken, volatile mandatory) |

---

## 12. Common Pitfalls

### 12.1 Serialization Breaking Singleton

```java
package com.course.patterns.singleton.pitfalls;

import java.io.*;

/**
 * Demonstrates how Java serialization can break the Singleton guarantee,
 * and how to prevent it.
 */
public final class SerializableSingleton implements Serializable {

    // Required for stable serialization across versions
    private static final long serialVersionUID = 1L;

    private static final SerializableSingleton INSTANCE = new SerializableSingleton();
    private int counter = 0;

    private SerializableSingleton() {}

    public static SerializableSingleton getInstance() {
        return INSTANCE;
    }

    public int incrementAndGet() { return ++counter; }

    /**
     * THE FIX: readResolve() is called by the Java serialization mechanism
     * after deserializing an object. Returning INSTANCE here ensures that
     * the deserialized "object" is immediately replaced with the existing singleton.
     *
     * Without this method, deserialization creates a NEW object — a second instance —
     * breaking the Singleton guarantee.
     *
     * @return the existing singleton instance
     */
    protected Object readResolve() throws ObjectStreamException {
        // Discard the deserialized object; return the canonical instance.
        return INSTANCE;
    }

    public static void main(String[] args) throws Exception {
        SerializableSingleton original = SerializableSingleton.getInstance();
        original.incrementAndGet(); // counter = 1
        System.out.println("Original hashCode: " + original.hashCode());

        // Serialize to bytes
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        try (ObjectOutputStream oos = new ObjectOutputStream(baos)) {
            oos.writeObject(original);
        }

        // Deserialize back
        SerializableSingleton deserialized;
        try (ObjectInputStream ois = new ObjectInputStream(
                new ByteArrayInputStream(baos.toByteArray()))) {
            deserialized = (SerializableSingleton) ois.readObject();
        }

        System.out.println("Deserialized hashCode: " + deserialized.hashCode());
        System.out.println("Same instance? " + (original == deserialized)); // true WITH readResolve
        // Without readResolve: false — two instances!
    }
}
```

**Enum singletons are immune**: The JVM's serialization of enums uses `Enum.valueOf()` during deserialization, which always returns the existing constant. No `readResolve()` needed.

### 12.2 Reflection Attack

```java
package com.course.patterns.singleton.pitfalls;

import java.lang.reflect.Constructor;

/**
 * Demonstrates reflection breaking a naive singleton, and the defense.
 */
public final class ReflectionDefendedSingleton {

    private static final ReflectionDefendedSingleton INSTANCE = new ReflectionDefendedSingleton();

    private ReflectionDefendedSingleton() {
        // DEFENSE: Throw if the singleton already exists.
        // This guard only helps for eager singletons where INSTANCE is assigned before
        // any external code can call the constructor via reflection.
        if (INSTANCE != null) {
            throw new IllegalStateException(
                "Singleton already instantiated. Use getInstance(). Reflection is prohibited."
            );
        }
    }

    public static ReflectionDefendedSingleton getInstance() {
        return INSTANCE;
    }

    public static void main(String[] args) {
        ReflectionDefendedSingleton original = ReflectionDefendedSingleton.getInstance();
        System.out.println("Original: " + original.hashCode());

        try {
            Constructor<ReflectionDefendedSingleton> constructor =
                ReflectionDefendedSingleton.class.getDeclaredConstructor();
            constructor.setAccessible(true);
            ReflectionDefendedSingleton hacked = constructor.newInstance();
            System.out.println("Hacked instance created: " + hacked.hashCode()); // Should NOT reach here
        } catch (Exception e) {
            // Expected: IllegalStateException wrapped in InvocationTargetException
            System.out.println("Reflection attack blocked: " + e.getCause().getMessage());
        }

        // BEST PRACTICE: Use Enum Singleton — JVM itself blocks reflective instantiation.
    }
}
```

### 12.3 Classloader Issue

```java
package com.course.patterns.singleton.pitfalls;

/**
 * Classloader isolation can break Singleton guarantees.
 *
 * Scenario: An application server (e.g., Tomcat) hot-deploys a WAR.
 * On each deployment, a new ClassLoader is created for the webapp.
 * The Singleton's static field (and the single instance) is per-ClassLoader.
 * Two deployments → two ClassLoaders → two "singleton" instances.
 *
 * This is also true in OSGi bundles and certain plugin systems.
 *
 * MITIGATION:
 * 1. Place the Singleton in a shared parent ClassLoader (e.g., the server's lib directory).
 * 2. Use a JNDI registry to store the instance outside the webapp ClassLoader.
 * 3. Use the Enum form — same problem, but the JVM's enum constants are per-ClassLoader.
 * 4. Use an IoC container (Spring) that is loaded by the parent ClassLoader.
 *
 * The code below demonstrates the concept (actual ClassLoader manipulation
 * requires a more complex setup with separate JAR files and custom ClassLoaders):
 */
public class ClassLoaderSingletonPitfall {

    /**
     * To truly demonstrate this, you would need:
     *   ClassLoader cl1 = new URLClassLoader(urls, null); // no parent = isolated
     *   ClassLoader cl2 = new URLClassLoader(urls, null);
     *   Class<?> s1 = cl1.loadClass("com.example.MySingleton");
     *   Class<?> s2 = cl2.loadClass("com.example.MySingleton");
     *   s1 == s2         → false  (different Class objects)
     *   s1.getField("INSTANCE").get(null) == s2.getField("INSTANCE").get(null) → false
     *
     * Moral: Never assume singleton when you have multiple classloaders.
     * Spring's @Singleton scope is per-ApplicationContext, not per-JVM,
     * precisely because Spring acknowledges this classloader reality.
     */
    public static void main(String[] args) {
        System.out.println("Current ClassLoader: " + ClassLoaderSingletonPitfall.class.getClassLoader());
        System.out.println("Parent ClassLoader: " + ClassLoaderSingletonPitfall.class.getClassLoader().getParent());
        // In an app server, the hierarchy could be: bootstrap → server → shared → webapp
        // A Singleton in 'webapp' CL is isolated from other webapps.
    }
}
```

---

## 13. Interview Questions with Model Answers

---

### Q1: What is the Singleton pattern and why is it used?

**Model Answer:**

The Singleton pattern is a creational design pattern whose purpose is twofold: to ensure a class has only one instance throughout the application's lifetime, and to provide a global, well-known access point to that instance. It achieves this by making the constructor private (preventing external `new` calls), holding a static reference to the sole instance within the class itself, and exposing that instance through a public static factory method, typically named `getInstance()`.

The pattern is used when the cost or semantics of having multiple instances is prohibitive. A database connection pool is a textbook example: each instance maintains a set of expensive, open socket connections to the database server. If every service class created its own pool, the system would exhaust the database's maximum connection limit almost immediately. The pool must be a single, shared entity to coordinate connection lending and return.

Other legitimate use cases include configuration managers (where all components must agree on the same config values), logging frameworks (where all log entries must pass through one file writer to maintain ordering), thread pool executors (where the total thread budget is a system-wide constraint), and hardware interface drivers (where only one process may own a serial port or printer queue at a time).

---

### Q2: What are the different ways to implement Singleton in Java? Which is the best?

**Model Answer:**

There are six main implementation strategies, each with distinct trade-offs:

**Eager initialization** creates the instance as a static final field at class-load time. It is thread-safe by virtue of the JVM's class-loading guarantee, but instantiates the object even if it is never used — a problem when initialization is expensive or conditionally needed.

**Lazy initialization (unsafe)** defers creation to the first `getInstance()` call but has a race condition: two threads can simultaneously observe `instance == null` and both call `new`, creating two instances. This must never be used in a multithreaded context.

**Synchronized method** fixes the race condition by declaring `getInstance()` as `synchronized`, ensuring mutual exclusion. However, every call — including the millions of calls after the instance exists — pays the lock acquisition and release cost. Under high contention this can be a serious bottleneck.

**Double-checked locking (DCL) with volatile** solves the performance problem by checking `instance == null` once outside the lock (fast path for 99.999% of calls) and again inside the lock (ensuring only one creation). The `volatile` keyword is mandatory: without it, the CPU or JVM may reorder the steps of object construction, allowing a second thread to see a non-null but partially-initialized instance. This was actually broken before Java 5; the JSR-133 memory model fix makes it correct on Java 5+.

**Initialization-on-demand holder (Bill Pugh)** uses an inner static class to hold the instance. The JVM loads the inner class only when `getInstance()` first references it, providing lazy initialization. Since class initialization is synchronized by the JVM and runs exactly once, this achieves thread safety without any `synchronized` or `volatile` in application code. It is widely regarded as the most elegant Java-specific solution.

**Enum Singleton**, recommended by Joshua Bloch in *Effective Java*, uses a single-element enum. It is lazy (enum constants are initialized at class-load time of the enum class, which is lazy relative to the application), thread-safe (JVM class loading), serialization-safe (JVM deserialization of enums uses `Enum.valueOf()`, always returning the existing constant), and reflection-safe (the JVM blocks reflective construction of enum instances with `IllegalArgumentException`). Enum cannot extend another class, which is its main limitation.

**Best choice**: For most new code, Enum Singleton is the best choice when lazy initialization is not strictly required and the enum's limitations are acceptable. When lazy initialization is important or the enum form feels semantically forced, the Bill Pugh (Initialization-on-Demand Holder) idiom is the best alternative.

---

### Q3: Why is the volatile keyword necessary in Double-Checked Locking?

**Model Answer:**

Without `volatile`, DCL is broken due to the Java Memory Model's permission to reorder instructions within a thread, combined with how object construction actually works at the bytecode level.

When the JVM executes `instance = new MySingleton()`, this is not a single atomic operation. At the bytecode level, it involves three steps: (1) allocate a block of memory for the object, (2) invoke the constructor to initialize the fields within that memory, and (3) assign the memory address to the reference variable `instance`. The JVM and the CPU are permitted to reorder steps 2 and 3 — that is, they may assign the memory address to `instance` before the constructor has finished executing. This is a valid optimization in a single-threaded program, but it is catastrophic in a multithreaded one.

Imagine Thread A is inside the synchronized block creating the instance. It allocates memory and assigns the address to `instance` (step 3) before the constructor finishes (step 2). At this exact moment, Thread B enters `getInstance()`, performs the first check (`instance == null`), sees that `instance` is non-null (the address has been assigned), and returns the reference — to an incompletely initialized object. Thread B then proceeds to use an object whose fields may be in an undefined state. This is a silent bug: no exception is thrown, but the object behaves incorrectly or produces corrupt data.

The `volatile` keyword prevents this by establishing a *happens-before* relationship. The Java Memory Model guarantees that all writes made before a volatile write are visible to any thread that subsequently reads that volatile variable. By declaring `instance` as `volatile`, we guarantee that the assignment `instance = new MySingleton()` cannot be reordered: the constructor must fully complete (step 2) before the reference is visible to other threads (step 3). This makes DCL correct on Java 5 and later.

---

### Q4: How does Java serialization break the Singleton pattern, and how do you fix it?

**Model Answer:**

Java's object serialization mechanism, when deserializing an object, creates a brand new instance by bypassing the normal object creation path (including the constructor). It uses reflection and the internal `ObjectInputStream.readObject()` machinery to reconstruct the object's state from the byte stream. This means that if you serialize a Singleton instance and later deserialize it, you get a different object — a second instance — breaking the fundamental Singleton guarantee.

Consider a Singleton named `ConfigManager` with a `static final INSTANCE` field. When `ObjectInputStream` deserializes the byte stream, it creates a new `ConfigManager` object using memory allocation without calling the constructor. The `INSTANCE` field is not consulted during deserialization — the stream contains all the field values needed to reconstruct the object from scratch. The result is two `ConfigManager` objects in the JVM at the same time.

The standard fix for non-enum singletons is to implement the `readResolve()` method. When `ObjectInputStream` finds a `readResolve()` method on the deserialized class, it calls it and uses the returned object instead of the freshly deserialized one. By returning `INSTANCE` from `readResolve()`, you ensure that the serialized-and-deserialized "copy" is immediately discarded, and the canonical instance is returned. The JVM garbage-collects the temporarily created object.

```java
protected Object readResolve() throws ObjectStreamException {
    return INSTANCE; // Canonical instance replaces the deserialized copy
}
```

The Enum Singleton approach is immune to this problem entirely. The Java specification mandates that enum constants are deserialized using `Enum.valueOf(Class, String)`, which always returns the existing constant from the current JVM. No extra code is needed.

---

### Q5: How does Singleton affect testability and what are the solutions?

**Model Answer:**

Singleton introduces three specific testability problems: global state pollution, inability to substitute implementations, and hidden dependencies.

**Global state pollution** means that a mutable Singleton's state persists between test runs within the same JVM. If `TestA` modifies `ConfigManager.getInstance().setProperty("env", "staging")` and `TestB` assumes `env` defaults to `production`, then `TestB` may fail or pass based on which test ran first — a property known as test-order dependence. This makes tests non-deterministic and extremely difficult to debug.

**Inability to substitute** is the deeper problem. When a class calls `ConfigManager.getInstance()` directly, there is no seam to insert a test double (mock, stub, or fake). Standard mocking frameworks like Mockito operate on instances — you create a mock object and inject it. But you cannot inject it when the class fetches the instance itself via a static method call. The only workaround is bytecode manipulation tools like PowerMock, which intercept static method calls at the bytecode level — a fragile and heavyweight approach.

**Hidden dependencies** make test setup non-obvious. A test must know to initialize or clear the singleton before exercising the class under test, but this requirement is invisible from the class's public API.

The recommended solutions are:

1. **Extract an interface** from the Singleton and **inject it via the constructor** (or a setter). The class under test depends on `ConfigProvider` (interface), not `ConfigManager` (concrete singleton). Tests inject a `StubConfigProvider`.

2. **Use a DI container** (Spring, Guice). The container manages the singleton lifecycle, and tests can override beans using `@Primary`, `@TestBean`, or by providing a test application context.

3. **Registry with reset**: If migrating legacy code, a static registry can hold the "current" instance and provide a `reset()` method that tests call in `@BeforeEach` or `@AfterEach` to restore a known state.

The underlying principle is the Dependency Inversion Principle: depend on abstractions, not concretions. A class that depends on an interface can be tested with any implementation. A class that calls `getInstance()` is permanently coupled to one specific object.

---

### Q6: What is the difference between Singleton in GoF and Singleton scope in Spring?

**Model Answer:**

This is one of the most commonly confused topics in Java interviews. The two are related in intent — one instance shared globally — but differ fundamentally in mechanism and scope.

The GoF Singleton enforces instance uniqueness within the class itself. The private constructor and the static `INSTANCE` field inside the class ensure that no second instance can be created. This enforcement is per-ClassLoader: if two classloaders each load the same Singleton class, each classloader has its own `INSTANCE` field, resulting in two independent instances in the same JVM. The class manages its own lifecycle.

Spring's singleton scope means one instance per `ApplicationContext`. The class itself has no private constructor, no static field, no lifecycle-management code — it is a plain Java class (POJO). Spring's IoC container wraps it and ensures that every injection point requesting that bean type receives the same object from the container. The class has no knowledge that it is being managed as a singleton; Spring enforces this from the outside.

The consequences are significant. First, scope: GoF is per-ClassLoader; Spring is per-ApplicationContext. Two Spring contexts in the same JVM each have their own "singleton" bean — this is by design, not a bug, and is heavily used in testing where each test class gets its own Spring context. Second, testability: GoF singletons resist mocking; Spring beans can be overridden with `@Primary`, `@MockBean`, or by providing a different `@Bean` in a test configuration class. Third, injection: GoF singletons access themselves via `getInstance()`; Spring beans declare their dependencies through constructor parameters, making the dependencies explicit and visible.

In modern Spring Boot applications, you almost never write a GoF Singleton. The entire application is managed by the Spring container, and "singleton" semantics are achieved transparently through the DI framework.

---

### Q7: Can Singleton be broken using Reflection? How do you defend against it?

**Model Answer:**

Yes, reflection can break a naive Singleton implementation. Using `java.lang.reflect.Constructor`, an attacker can call `getDeclaredConstructor()` on the Singleton class, invoke `setAccessible(true)` to bypass Java's access control checks, and then call `newInstance()` to create a second instance — even when the constructor is declared `private`.

```java
Constructor<MySingleton> c = MySingleton.class.getDeclaredConstructor();
c.setAccessible(true);
MySingleton second = c.newInstance(); // Bypasses private!
```

There are two defensive strategies for non-enum singletons. The first is a guard in the constructor that throws if the instance already exists:

```java
private MySingleton() {
    if (INSTANCE != null) {
        throw new IllegalStateException("Use getInstance() — reflection is prohibited.");
    }
}
```

This works reliably only for **eager** singletons where `INSTANCE` is assigned before external code can execute. For lazy singletons, there is a window between class loading and the first `getInstance()` call during which the reflective attacker could succeed before `INSTANCE` is set.

The second, and by far superior, defense is to use the **Enum Singleton**. The JVM itself enforces that enum constants cannot be reflectively instantiated. When you call `Constructor.newInstance()` on an enum, the JVM throws `IllegalArgumentException: Cannot reflectively create enum objects`. This is baked into the JDK source of `java.lang.reflect.Constructor.newInstance()` — there is an explicit check for enum types. No application-level guard code is needed.

For new code where the Singleton pattern is genuinely needed and serialization/reflection safety are concerns, the Enum Singleton is the correct choice. Bloch's *Effective Java* makes this point emphatically in Item 3.

---

### Q8: Is Singleton an anti-pattern? Argue both sides.

**Model Answer:**

Calling Singleton an anti-pattern is an overstatement, but the criticism contains important truths that every engineer should understand.

**The case against Singleton**: The pattern introduces global mutable state, which is the software engineering equivalent of global variables — universally acknowledged as harmful because they couple unrelated components, make execution order significant, and undermine the principle of local reasoning. When you read a function, you should be able to understand its behavior from its inputs and outputs. A function that reads from a mutable Singleton has a hidden input — the Singleton's current state — that is invisible from its signature. This makes code harder to reason about, refactor, and test.

The SRP violation is also real: a class that self-manages its instantiation has two reasons to change — changes to business logic and changes to lifecycle-management policy.

The testability damage is severe in practice. In a large codebase where dozens of classes call `ConfigManager.getInstance()`, replacing the config implementation for a test requires either modifying the Singleton (risky) or using bytecode manipulation tools (fragile). Teams often discover this problem only after the codebase has grown to a size where refactoring is painful.

**The case for Singleton**: Some resources genuinely must be unique. A database connection pool, a global event bus, or a hardware port driver is not a matter of convenience — having two independent instances would cause data corruption, connection limit exhaustion, or physical device conflicts. In these cases, the Singleton pattern expresses a real-world constraint in code. The alternative — relying on developers to "not create a second one" — is far worse engineering.

Furthermore, framework and infrastructure code often operates at a level where DI containers are not available or appropriate. The JDK's `Runtime.getRuntime()` is a Singleton because the JVM runtime is genuinely singular — there is exactly one per process.

**The synthesis**: Singleton is appropriate for *genuinely* singular infrastructure resources and inappropriate for business logic. Modern best practice is to let a DI container (Spring, Guice) manage singleton semantics externally, keeping classes POJO-clean. When you find yourself writing GoF Singletons in business code, that is a code smell pointing toward missing DI infrastructure.

---

## 14. Comparison with Related Patterns

### 14.1 Singleton vs Factory Method Pattern

```
+--------------------+-----------------------------+----------------------------+
|                    |      Singleton              |     Factory Method         |
+--------------------+-----------------------------+----------------------------+
| Purpose            | Control instance count to 1 | Decouple object creation   |
|                    |                             | from its usage             |
+--------------------+-----------------------------+----------------------------+
| Creates            | Always returns the SAME     | May return NEW instances    |
|                    | instance                    | on each call               |
+--------------------+-----------------------------+----------------------------+
| Flexibility        | Fixed: one instance         | Flexible: subclasses choose|
|                    |                             | what to create             |
+--------------------+-----------------------------+----------------------------+
| Subclassing        | Blocked by private          | Enabled — key design point |
|                    | constructor                 |                            |
+--------------------+-----------------------------+----------------------------+
| Can be combined?   | Yes: a Singleton can be     | Yes: Factory can ensure    |
|                    | obtained via a Factory      | it produces a Singleton    |
+--------------------+-----------------------------+----------------------------+
```

```java
// Singleton + Factory combined: a registry of singletons by type
package com.course.patterns.singleton.comparison;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Supplier;

/**
 * A generic singleton factory: ensures one instance per type.
 * Combines Factory Method (parameterized creation) with Singleton (unique instance).
 */
public final class SingletonFactory {

    private static final Map<Class<?>, Object> instances = new ConcurrentHashMap<>();

    private SingletonFactory() {
        throw new UnsupportedOperationException("Utility class.");
    }

    /**
     * Returns the singleton instance of the given type.
     * Creates it via the supplier on first call; returns the cached instance on subsequent calls.
     *
     * @param <T>      the type
     * @param type     the class key
     * @param supplier creates the instance if absent
     * @return the singleton instance for the given type
     */
    @SuppressWarnings("unchecked")
    public static <T> T getInstance(Class<T> type, Supplier<T> supplier) {
        return (T) instances.computeIfAbsent(type, k -> supplier.get());
    }

    // Usage
    public static void main(String[] args) {
        // Each call to getInstance for the same type returns the same object
        Object s1 = SingletonFactory.getInstance(StringBuilder.class, StringBuilder::new);
        Object s2 = SingletonFactory.getInstance(StringBuilder.class, StringBuilder::new);
        System.out.println("Same StringBuilder? " + (s1 == s2)); // true
    }
}
```

### 14.2 Singleton vs Monostate Pattern

The **Monostate** pattern (also called Borg pattern) achieves shared behavior without restricting the number of instances. All instances share the same static state, so creating multiple objects is harmless — they all behave as one.

```java
package com.course.patterns.singleton.comparison;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Monostate Pattern — all instances share the same static state.
 *
 * Unlike Singleton:
 *   - Constructor is public — you CAN call new MonostateConfig()
 *   - Multiple instances can exist
 *   - All instances read/write the SAME underlying static data
 *   - Polymorphism works naturally — subclasses also share state
 *
 * Unlike Singleton:
 *   - Does NOT enforce a single instance at the class level
 *   - State is static; cannot be per-instance
 *   - Sharing is implicit and invisible to callers (arguably worse than Singleton)
 */
public class MonostateConfig {

    // All instances share this map — static state
    private static final Map<String, String> SHARED_STATE = new ConcurrentHashMap<>();

    // Public constructor — you can create many instances
    public MonostateConfig() {
        // No restriction on instantiation
    }

    public void setProperty(String key, String value) {
        SHARED_STATE.put(key, value); // Written to shared static state
    }

    public String getProperty(String key) {
        return SHARED_STATE.get(key); // Read from shared static state
    }

    public static void main(String[] args) {
        MonostateConfig c1 = new MonostateConfig();
        MonostateConfig c2 = new MonostateConfig();

        c1.setProperty("env", "production");
        System.out.println("c2 sees env: " + c2.getProperty("env")); // "production" — shared state!
        System.out.println("c1 == c2? " + (c1 == c2));               // false — different objects
    }
}

/**
 * Singleton vs Monostate — when to choose each:
 *
 * +------------------------+----------------------------------+----------------------------------+
 * |                        |         Singleton                |         Monostate                |
 * +------------------------+----------------------------------+----------------------------------+
 * | Number of instances    | One (enforced)                   | Many (but share state)           |
 * | Constructor            | Private                          | Public                           |
 * | Polymorphism           | Blocked (private constructor)    | Fully supported                  |
 * | Visibility of sharing  | Explicit (getInstance())         | Invisible (caller doesn't know)  |
 * | Testability            | Difficult                        | Also difficult (static state)    |
 * | Use case               | Resource must truly be unique    | Multiple instances need behavior |
 * |                        |                                  | coordination without             |
 * |                        |                                  | restricting instantiation        |
 * +------------------------+----------------------------------+----------------------------------+
 *
 * Monostate is rarely preferred over Singleton. It has all of Singleton's state-sharing
 * downsides (global state, test pollution) with none of the clarity (the sharing is
 * invisible to the caller). Prefer Singleton or DI-managed singletons over Monostate.
 */
```

---

## 15. Quick Reference Cheat Sheet

```
SINGLETON PATTERN — QUICK REFERENCE
=====================================

INTENT:
  Ensure one instance. Provide global access.

PICK YOUR IMPLEMENTATION:
  ┌─────────────────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
  │ Need                    │ Eager        │ Bill Pugh    │ DCL+volatile │ Enum         │
  ├─────────────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
  │ Simplest correct code   │ Yes          │              │              │ Best         │
  │ Lazy initialization     │ No           │ Best         │ Yes          │ No           │
  │ Serialization safe      │ + readResolve│ + readResolve│ + readResolve│ Yes (free)   │
  │ Reflection safe         │ + guard      │ + guard      │ + guard      │ Yes (free)   │
  │ Can extend a class      │ Yes          │ Yes          │ Yes          │ No           │
  │ Java 4 compatible       │ Yes          │ Yes          │ No (volatile)│ No (enum)    │
  └─────────────────────────┴──────────────┴──────────────┴──────────────┴──────────────┘

CRITICAL VOCABULARY:
  volatile        → prevents instruction reordering across threads
  happens-before  → JMM guarantee: all writes before X are visible after X
  readResolve()   → hook called by ObjectInputStream to replace deserialized object
  ClassLoader     → each CL has its own static fields → own "singleton"

TESTING CHECKLIST:
  □ Does the class accept a dependency via constructor? → Good
  □ Does the class call getInstance() internally?       → Test pain
  □ Is there an interface you can stub?                 → Extract one
  □ Does state persist between tests?                   → Add reset() / use DI

SPRING REMINDER:
  @Service / @Component = Spring singleton scope (per ApplicationContext)
  GoF Singleton = per ClassLoader
  Prefer Spring-managed beans over handwritten GoF singletons in app code.

KEY JDK EXAMPLES:
  java.lang.Runtime.getRuntime()          → eager singleton
  java.awt.Desktop.getDesktop()           → platform singleton with guard
  java.lang.System.getSecurityManager()  → nullable singleton
  java.util.logging.LogManager            → per-ClassLoader singleton
```

---

*End of Singleton Pattern Guide*

*Next in Series: [02 — Factory Method Pattern](02-factory-method.md)*
*Previous: [00 — Creational Patterns Overview](00-overview.md)*

---

> **Study Tip**: The three implementations you must be able to write from memory in an interview are: (1) Eager Initialization, (2) Double-Checked Locking with volatile, and (3) Enum Singleton. Be prepared to explain why volatile is necessary in DCL and why Enum is serialization/reflection safe.
