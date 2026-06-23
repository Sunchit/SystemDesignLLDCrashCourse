# Builder Pattern

## Table of Contents

1. [Intent and the Problem It Solves](#1-intent-and-the-problem-it-solves)
2. [The Telescoping Constructor Anti-Pattern](#2-the-telescoping-constructor-anti-pattern)
3. [When to Use / When NOT to Use](#3-when-to-use--when-not-to-use)
4. [UML Class Diagrams](#4-uml-class-diagrams)
5. [Complete Java Implementations](#5-complete-java-implementations)
   - [HTTP Request Builder](#51-http-request-builder)
   - [Pizza Builder](#52-pizza-builder)
   - [Computer Configuration Builder](#53-computer-configuration-builder)
6. [Fluent API and Method Chaining](#6-fluent-api-and-method-chaining)
7. [Director Class](#7-director-class)
8. [Immutability Through Builder](#8-immutability-through-builder)
9. [Real-World Examples](#9-real-world-examples)
10. [Builder vs Factory - Decision Table](#10-builder-vs-factory---decision-table)
11. [Trade-offs](#11-trade-offs)
12. [Interview Questions and Answers](#12-interview-questions-and-answers)

---

## 1. Intent and the Problem It Solves

### Definition

The Builder pattern is a **creational design pattern** that separates the **construction** of a complex object from its **representation**, allowing the same construction process to produce different representations.

The canonical GoF definition (Gang of Four, *Design Patterns: Elements of Reusable Object-Oriented Software*, 1994):

> "Separate the construction of a complex object from its representation so that the same construction process can create different representations."

### Core Problem

When an object has many fields — some required, some optional — and you want the object to be immutable after construction, you end up with one of these painful situations:

- **Telescoping constructors**: multiple overloaded constructors, each adding one more parameter
- **JavaBeans pattern**: a no-arg constructor + many setters — but this makes the object mutable and leaves it in an inconsistent state mid-construction
- **Parameter confusion**: long parameter lists where callers make positional mistakes (e.g., swapping two `String` arguments silently)

The Builder pattern solves all three problems by introducing a companion `Builder` object that accumulates field assignments and then creates the target object in a single validated `build()` call.

---

## 2. The Telescoping Constructor Anti-Pattern

The telescoping constructor pattern occurs when you provide multiple overloaded constructors to handle different combinations of optional parameters. Each new optional field forces a combinatorial explosion of constructors.

### Anti-Pattern: NutritionFacts (classic example)

```java
// ANTI-PATTERN: Telescoping constructors
// Adding one new field forces duplicating every existing constructor
public class NutritionFacts {

    private final int servingSize;    // required
    private final int servings;       // required
    private final int calories;       // optional
    private final int fat;            // optional
    private final int sodium;         // optional
    private final int carbohydrate;   // optional
    private final int protein;        // optional
    private final int fiber;          // optional

    // 1. Only required fields
    public NutritionFacts(int servingSize, int servings) {
        this(servingSize, servings, 0);
    }

    // 2. Required + calories
    public NutritionFacts(int servingSize, int servings, int calories) {
        this(servingSize, servings, calories, 0);
    }

    // 3. Required + calories + fat
    public NutritionFacts(int servingSize, int servings, int calories, int fat) {
        this(servingSize, servings, calories, fat, 0);
    }

    // 4. Required + calories + fat + sodium
    public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium) {
        this(servingSize, servings, calories, fat, sodium, 0);
    }

    // 5. Required + calories + fat + sodium + carbohydrate
    public NutritionFacts(int servingSize, int servings, int calories, int fat,
                          int sodium, int carbohydrate) {
        this(servingSize, servings, calories, fat, sodium, carbohydrate, 0);
    }

    // 6. Required + calories + fat + sodium + carbohydrate + protein
    public NutritionFacts(int servingSize, int servings, int calories, int fat,
                          int sodium, int carbohydrate, int protein) {
        this(servingSize, servings, calories, fat, sodium, carbohydrate, protein, 0);
    }

    // 7. All fields
    public NutritionFacts(int servingSize, int servings, int calories, int fat,
                          int sodium, int carbohydrate, int protein, int fiber) {
        this.servingSize  = servingSize;
        this.servings     = servings;
        this.calories     = calories;
        this.fat          = fat;
        this.sodium       = sodium;
        this.carbohydrate = carbohydrate;
        this.protein      = protein;
        this.fiber        = fiber;
    }
}
```

**Problems with the above code:**

```java
// What do these positional integers mean? Try to tell at a glance:
NutritionFacts cocaCola = new NutritionFacts(240, 8, 100, 0, 35, 27, 0, 0);
//                                            ^    ^   ^   ^  ^   ^   ^  ^
//                          servingSize ------|    |   |   |  |   |   |  |
//                          servings ---------|    |   |   |  |   |   |  |
//                          calories ------------------|   |  |   |   |  |
//                          fat ----------------------------|  |   |   |  |
//                          sodium ----------------------------|   |   |  |
//                          carbohydrate ---------------------------|   |  |
//                          protein -----------------------------------|  |
//                          fiber ------------------------------------|
// If you swap fat=0 and sodium=35, the compiler says nothing. The object is silently wrong.
```

### Anti-Pattern: HttpRequest with Many Options

```java
// ANTI-PATTERN: 6+ constructor overloads on a real-world class
public class HttpRequest {

    private final String method;
    private final String url;
    private final Map<String, String> headers;
    private final String body;
    private final int timeoutMs;
    private final String authToken;
    private final boolean followRedirects;
    private final int retryCount;

    // Overload 1 — minimum viable request (GET with no extras)
    public HttpRequest(String method, String url) {
        this(method, url, Collections.emptyMap(), null, 5000, null, true, 0);
    }

    // Overload 2 — with custom headers
    public HttpRequest(String method, String url, Map<String, String> headers) {
        this(method, url, headers, null, 5000, null, true, 0);
    }

    // Overload 3 — with body (for POST/PUT)
    public HttpRequest(String method, String url, Map<String, String> headers, String body) {
        this(method, url, headers, body, 5000, null, true, 0);
    }

    // Overload 4 — with body and custom timeout
    public HttpRequest(String method, String url, Map<String, String> headers,
                       String body, int timeoutMs) {
        this(method, url, headers, body, timeoutMs, null, true, 0);
    }

    // Overload 5 — with auth token
    public HttpRequest(String method, String url, Map<String, String> headers,
                       String body, int timeoutMs, String authToken) {
        this(method, url, headers, body, timeoutMs, authToken, true, 0);
    }

    // Overload 6 — with redirect control
    public HttpRequest(String method, String url, Map<String, String> headers,
                       String body, int timeoutMs, String authToken, boolean followRedirects) {
        this(method, url, headers, body, timeoutMs, authToken, followRedirects, 0);
    }

    // Overload 7 — all fields (actual all-args constructor)
    public HttpRequest(String method, String url, Map<String, String> headers,
                       String body, int timeoutMs, String authToken,
                       boolean followRedirects, int retryCount) {
        this.method          = method;
        this.url             = url;
        this.headers         = Collections.unmodifiableMap(new HashMap<>(headers));
        this.body            = body;
        this.timeoutMs       = timeoutMs;
        this.authToken       = authToken;
        this.followRedirects = followRedirects;
        this.retryCount      = retryCount;
    }

    // Getters omitted for brevity...
}

// CALLER CODE — extremely hard to read and error-prone:
HttpRequest req = new HttpRequest("POST", "https://api.example.com/data",
        Map.of("Content-Type", "application/json"),
        "{\"key\":\"value\"}",
        3000,
        "Bearer eyJhbG...",
        true,
        2);
// ^ reader has no idea what 3000, true, and 2 mean without counting parameters
```

### The JavaBeans Alternative — Also Wrong

```java
// ALSO AN ANTI-PATTERN: mutable JavaBean
public class HttpRequest {
    private String method;    // mutable — bad for thread safety
    private String url;
    private String body;
    // setters expose mutability...

    public HttpRequest() {}   // object exists in invalid state (no URL, no method)

    public void setMethod(String method) { this.method = method; }
    public void setUrl(String url)       { this.url = url; }
    // If someone passes this object to another thread after setUrl()
    // but before setMethod(), the object is in an inconsistent state.
}
```

---

## 3. When to Use / When NOT to Use

### When TO Use the Builder Pattern

| Situation | Reason |
|-----------|--------|
| Object has **4 or more fields**, many optional | Avoids combinatorial constructor explosion |
| You want the product to be **immutable** | Builder accumulates state; `build()` creates a final object |
| Object construction involves **validation logic** | `build()` is the single place to run cross-field validations |
| The **same construction steps** produce different representations | Director + Builder combination serves this well |
| **Named parameters** matter for readability | `.timeout(3000).method("POST")` is self-documenting |
| You are building a **DSL** or fluent configuration API | Builder chains read like English sentences |
| The object requires **multi-step construction** | Builder accumulates steps; final object is only created once all steps complete |

### When NOT to Use the Builder Pattern

| Situation | Reason |
|-----------|--------|
| Object has **2-3 fields**, all required | Simple constructor is cleaner and less boilerplate |
| Object is a **simple value object** (POJO/record) | Java 14+ `record` types handle this with zero boilerplate |
| Construction is **trivial** with no optional fields | Builder adds overhead without benefit |
| You need **mutability** post-construction | Builder produces a snapshot; mutable objects use setters directly |
| The class is part of a **framework requiring no-arg constructors** | JPA entities, Jackson deserialization with default config |
| **Performance is critical** and object creation is in a hot loop | Builder allocates an extra object per construction |

### Rule of Thumb

> If you find yourself writing more than two constructor overloads OR if a caller reading your API cannot tell what positional arguments mean without an IDE tooltip — reach for a Builder.

---

## 4. UML Class Diagrams

### 4.1 Builder Pattern WITHOUT Director

This is the most common form used in practice (Joshua Bloch's *Effective Java*, Item 2).

```
┌─────────────────────────────────────┐
│          <<product>>                │
│           HttpRequest               │
├─────────────────────────────────────┤
│ - method    : String                │
│ - url       : String                │
│ - headers   : Map<String,String>    │
│ - body      : String                │
│ - timeoutMs : int                   │
│ - authToken : String                │
├─────────────────────────────────────┤
│ - HttpRequest(Builder)              │ ← private constructor
│ + getMethod()  : String             │
│ + getUrl()     : String             │
│ + getHeaders() : Map<String,String> │
│ + getBody()    : String             │
│ + getTimeoutMs(): int               │
│ + getAuthToken(): String            │
└────────────────────┬────────────────┘
                     │  creates
                     │◄────────────────────────────────────────┐
                     │                                         │
┌────────────────────▼────────────────────────────────────────┤
│         HttpRequest.Builder   (static inner class)          │
├─────────────────────────────────────────────────────────────┤
│ - method    : String  (required)                            │
│ - url       : String  (required)                            │
│ - headers   : Map<String,String> = new HashMap<>()          │
│ - body      : String  (optional)                            │
│ - timeoutMs : int = 5000  (optional, has default)           │
│ - authToken : String  (optional)                            │
├─────────────────────────────────────────────────────────────┤
│ + Builder(method: String, url: String)  ← required fields   │
│ + header(key: String, value: String) : Builder              │
│ + body(body: String)                  : Builder             │
│ + timeout(ms: int)                    : Builder             │
│ + authToken(token: String)            : Builder             │
│ + build()                             : HttpRequest         │ ← validates + creates
└─────────────────────────────────────────────────────────────┘

Legend:
  ─────   association
  ◄────   "creates" dependency (Builder creates HttpRequest)
```

### 4.2 Builder Pattern WITH Director (GoF canonical form)

The Director is useful when you want to encapsulate pre-defined construction sequences (e.g., "build a standard GET request", "build a POST with auth").

```
┌───────────────────────────────┐
│           Director            │
├───────────────────────────────┤
│ - builder : Builder           │◄──── receives at construction or via setter
├───────────────────────────────┤
│ + setBuilder(b: Builder)      │
│ + constructGetRequest()       │ ← calls builder steps in sequence
│ + constructAuthPostRequest()  │
└───────────────┬───────────────┘
                │ uses
                ▼
┌───────────────────────────────┐          ┌───────────────────────────────┐
│         <<interface>>         │          │      ConcreteBuilderA         │
│            Builder            │◄─────────┤  (JsonHttpRequestBuilder)     │
├───────────────────────────────┤ implements├───────────────────────────────┤
│ + setMethod(m: String)        │          │ - method  : String            │
│ + setUrl(u: String)           │          │ - url     : String            │
│ + setBody(b: String)          │          │ - body    : String            │
│ + setTimeout(ms: int)         │          ├───────────────────────────────┤
│ + setAuthToken(t: String)     │          │ + setMethod(...)              │
│ + getResult() : Product       │          │ + setUrl(...)                 │
└───────────────────────────────┘          │ + setBody(...)                │
                                           │ + setTimeout(...)             │
              ┌────────────────────────────┤ + setAuthToken(...)           │
              │ creates                    │ + getResult() : HttpRequest   │
              ▼                            └───────────────────────────────┘
┌───────────────────────────────┐
│      <<product>>              │
│       HttpRequest             │
├───────────────────────────────┤
│ - method    : String          │
│ - url       : String          │
│ - body      : String          │
│ - timeoutMs : int             │
│ - authToken : String          │
└───────────────────────────────┘

Legend:
  ◄──── implements
  ─────  association / uses
  ──►   creates
```

---

## 5. Complete Java Implementations

### 5.1 HTTP Request Builder

This is the most production-realistic example — the pattern you'll see in OkHttp, Retrofit, and virtually every HTTP client library.

```java
package com.course.patterns.builder;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

/**
 * Immutable HTTP request object built via inner static Builder.
 *
 * <p>Usage:
 * <pre>{@code
 * HttpRequest request = new HttpRequest.Builder("POST", "https://api.example.com/users")
 *     .header("Content-Type", "application/json")
 *     .header("Accept", "application/json")
 *     .body("{\"name\":\"Alice\"}")
 *     .timeout(3000)
 *     .authToken("Bearer eyJhbGciOiJIUzI1NiJ9...")
 *     .followRedirects(false)
 *     .retryCount(2)
 *     .build();
 * }</pre>
 */
public final class HttpRequest {

    // ── Fields: all final → object is fully immutable after build() ───────────
    private final String method;
    private final String url;
    private final Map<String, String> headers;     // unmodifiable view
    private final String body;                      // nullable — GET has no body
    private final int timeoutMs;
    private final String authToken;                 // nullable — not all requests need auth
    private final boolean followRedirects;
    private final int retryCount;

    /**
     * Private constructor: only the inner Builder may call this.
     * This is the key mechanism that enforces immutability — callers
     * cannot instantiate HttpRequest any other way.
     */
    private HttpRequest(Builder builder) {
        this.method          = builder.method;
        this.url             = builder.url;
        // Defensive copy + make unmodifiable so the internal map cannot be mutated
        this.headers         = Collections.unmodifiableMap(new HashMap<>(builder.headers));
        this.body            = builder.body;
        this.timeoutMs       = builder.timeoutMs;
        this.authToken       = builder.authToken;
        this.followRedirects = builder.followRedirects;
        this.retryCount      = builder.retryCount;
    }

    // ── Getters (no setters — immutable) ─────────────────────────────────────

    public String getMethod()          { return method; }
    public String getUrl()             { return url; }
    public Map<String, String> getHeaders() { return headers; }
    public String getBody()            { return body; }
    public int getTimeoutMs()          { return timeoutMs; }
    public String getAuthToken()       { return authToken; }
    public boolean isFollowRedirects() { return followRedirects; }
    public int getRetryCount()         { return retryCount; }

    /** Convenience: is this a POST, PUT, or PATCH that can carry a body? */
    public boolean hasBody() {
        return body != null && !body.isBlank();
    }

    @Override
    public String toString() {
        return "HttpRequest{" +
               "method='"         + method          + '\'' +
               ", url='"          + url             + '\'' +
               ", headers="       + headers                +
               ", body='"         + (body != null ? "[present]" : "[none]") + '\'' +
               ", timeoutMs="     + timeoutMs             +
               ", authToken='"    + (authToken != null ? "[redacted]" : "[none]") + '\'' +
               ", followRedirects=" + followRedirects      +
               ", retryCount="    + retryCount            +
               '}';
    }

    // ── Inner Static Builder ──────────────────────────────────────────────────

    public static final class Builder {

        // Required fields — passed to the Builder constructor
        private final String method;
        private final String url;

        // Optional fields — sensible defaults assigned here
        private final Map<String, String> headers = new HashMap<>();
        private String body          = null;
        private int    timeoutMs     = 5_000;    // 5 second default
        private String authToken     = null;
        private boolean followRedirects = true;
        private int    retryCount    = 0;

        /**
         * Builder constructor accepts only the required fields.
         * This ensures the product can never be in an invalid state
         * (missing method or URL) regardless of which optional setters are called.
         *
         * @param method HTTP method, e.g. "GET", "POST", "PUT", "DELETE"
         * @param url    fully-qualified URL including scheme, e.g. "https://api.example.com/v1/users"
         */
        public Builder(String method, String url) {
            // Validate right at entry — fail fast
            this.method = Objects.requireNonNull(method, "HTTP method must not be null")
                                 .toUpperCase().strip();
            this.url    = Objects.requireNonNull(url, "URL must not be null").strip();

            if (this.method.isBlank()) {
                throw new IllegalArgumentException("HTTP method must not be blank");
            }
            if (this.url.isBlank()) {
                throw new IllegalArgumentException("URL must not be blank");
            }
        }

        /**
         * Add a single HTTP header. Calling this multiple times accumulates headers.
         * Duplicate keys overwrite the previous value (last-write-wins).
         */
        public Builder header(String key, String value) {
            Objects.requireNonNull(key,   "Header key must not be null");
            Objects.requireNonNull(value, "Header value must not be null");
            this.headers.put(key.strip(), value.strip());
            return this; // return 'this' enables method chaining
        }

        /**
         * Add multiple headers at once from an existing map.
         */
        public Builder headers(Map<String, String> headers) {
            Objects.requireNonNull(headers, "Headers map must not be null");
            this.headers.putAll(headers);
            return this;
        }

        /**
         * Set the request body. Only meaningful for POST, PUT, PATCH.
         * For GET/DELETE this is silently ignored during validation.
         */
        public Builder body(String body) {
            this.body = body;
            return this;
        }

        /**
         * Set the connection + read timeout in milliseconds.
         * Must be positive.
         */
        public Builder timeout(int timeoutMs) {
            if (timeoutMs <= 0) {
                throw new IllegalArgumentException(
                    "Timeout must be positive, got: " + timeoutMs);
            }
            this.timeoutMs = timeoutMs;
            return this;
        }

        /**
         * Set the Authorization header token.
         * Caller should include scheme prefix: "Bearer &lt;token&gt;" or "Basic &lt;base64&gt;".
         */
        public Builder authToken(String authToken) {
            this.authToken = authToken;
            return this;
        }

        /**
         * Control whether 3xx redirects are followed automatically.
         */
        public Builder followRedirects(boolean followRedirects) {
            this.followRedirects = followRedirects;
            return this;
        }

        /**
         * Number of automatic retries on transient failure (5xx, timeout).
         * Must be non-negative. Zero means no retries.
         */
        public Builder retryCount(int retryCount) {
            if (retryCount < 0) {
                throw new IllegalArgumentException(
                    "Retry count must be non-negative, got: " + retryCount);
            }
            this.retryCount = retryCount;
            return this;
        }

        /**
         * Validate accumulated state and construct the immutable HttpRequest.
         *
         * <p>Cross-field validation rules:
         * <ul>
         *   <li>GET and HEAD requests must not have a body.</li>
         *   <li>POST and PUT requests are warned (not hard-failed) if body is absent.</li>
         *   <li>URL must start with http:// or https://.</li>
         * </ul>
         *
         * @return a new, fully-immutable {@link HttpRequest}
         * @throws IllegalStateException if validation fails
         */
        public HttpRequest build() {
            validate();
            return new HttpRequest(this);
        }

        private void validate() {
            // URL scheme validation
            if (!url.startsWith("http://") && !url.startsWith("https://")) {
                throw new IllegalStateException(
                    "URL must start with http:// or https://, got: " + url);
            }

            // Body presence check for safe methods
            if ((method.equals("GET") || method.equals("HEAD") || method.equals("DELETE"))
                    && body != null && !body.isBlank()) {
                throw new IllegalStateException(
                    method + " requests must not have a body. " +
                    "Use query parameters instead.");
            }

            // Warn for likely-wrong POST without body (not throwing — may be intentional)
            if ((method.equals("POST") || method.equals("PUT")) && (body == null || body.isBlank())) {
                System.err.println("[WARN] HttpRequest.Builder: " + method +
                    " request has no body. Was this intentional?");
            }

            // Auth token sanity: if set, it should look like a scheme prefix
            if (authToken != null && !authToken.isBlank()) {
                if (!authToken.startsWith("Bearer ") && !authToken.startsWith("Basic ")) {
                    throw new IllegalStateException(
                        "authToken should start with 'Bearer ' or 'Basic '. Got: " +
                        authToken.substring(0, Math.min(authToken.length(), 10)) + "...");
                }
            }
        }
    }

    // ── Demo main ─────────────────────────────────────────────────────────────

    public static void main(String[] args) {
        // 1. Simple GET
        HttpRequest getRequest = new HttpRequest.Builder("GET", "https://api.example.com/users")
            .header("Accept", "application/json")
            .timeout(2000)
            .build();
        System.out.println("GET: " + getRequest);

        // 2. Authenticated POST
        HttpRequest postRequest = new HttpRequest.Builder("POST", "https://api.example.com/users")
            .header("Content-Type", "application/json")
            .header("Accept", "application/json")
            .body("{\"name\":\"Alice\",\"email\":\"alice@example.com\"}")
            .timeout(5000)
            .authToken("Bearer eyJhbGciOiJIUzI1NiJ9.dGVzdA.abc123")
            .followRedirects(false)
            .retryCount(3)
            .build();
        System.out.println("POST: " + postRequest);

        // 3. Try to build an invalid request — runtime exception
        try {
            HttpRequest badRequest = new HttpRequest.Builder("GET", "https://api.example.com/data")
                .body("this should not be here")
                .build();
        } catch (IllegalStateException e) {
            System.out.println("Caught expected validation error: " + e.getMessage());
        }
    }
}
```

---

### 5.2 Pizza Builder

This example demonstrates the Builder with enums for constrained choices and a list for accumulating toppings.

```java
package com.course.patterns.builder;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;

/**
 * Immutable Pizza product built through a fluent Builder.
 *
 * <p>Usage:
 * <pre>{@code
 * Pizza pizza = new Pizza.Builder(Pizza.Size.LARGE, Pizza.Crust.THIN)
 *     .sauce(Pizza.Sauce.TOMATO)
 *     .topping("Mozzarella")
 *     .topping("Pepperoni")
 *     .topping("Mushrooms")
 *     .extraCheese(true)
 *     .stuffedCrust(false)
 *     .build();
 * }</pre>
 */
public final class Pizza {

    // ── Domain Enums ──────────────────────────────────────────────────────────

    public enum Size {
        SMALL("9-inch"),
        MEDIUM("12-inch"),
        LARGE("14-inch"),
        EXTRA_LARGE("18-inch");

        private final String inches;
        Size(String inches) { this.inches = inches; }
        public String getInches() { return inches; }
    }

    public enum Crust {
        THIN("Thin & Crispy"),
        REGULAR("Hand-Tossed"),
        THICK("Pan/Deep-Dish"),
        STUFFED("Cheese-Stuffed");

        private final String displayName;
        Crust(String displayName) { this.displayName = displayName; }
        public String getDisplayName() { return displayName; }
    }

    public enum Sauce {
        TOMATO, BBQ, ALFREDO, PESTO, NONE;
    }

    // ── Product Fields (all final) ────────────────────────────────────────────

    private final Size         size;
    private final Crust        crust;
    private final Sauce        sauce;
    private final List<String> toppings;       // unmodifiable after build
    private final boolean      extraCheese;
    private final boolean      stuffedCrust;
    private final String       specialInstructions;

    private Pizza(Builder builder) {
        this.size                = builder.size;
        this.crust               = builder.crust;
        this.sauce               = builder.sauce;
        // Defensive copy → caller cannot mutate our internal list
        this.toppings            = Collections.unmodifiableList(new ArrayList<>(builder.toppings));
        this.extraCheese         = builder.extraCheese;
        this.stuffedCrust        = builder.stuffedCrust;
        this.specialInstructions = builder.specialInstructions;
    }

    // ── Getters ───────────────────────────────────────────────────────────────

    public Size         getSize()               { return size; }
    public Crust        getCrust()              { return crust; }
    public Sauce        getSauce()              { return sauce; }
    public List<String> getToppings()           { return toppings; }
    public boolean      isExtraCheese()         { return extraCheese; }
    public boolean      isStuffedCrust()        { return stuffedCrust; }
    public String       getSpecialInstructions(){ return specialInstructions; }

    /** Derived property: compute base price by size */
    public double getBasePrice() {
        return switch (size) {
            case SMALL       -> 8.99;
            case MEDIUM      -> 11.99;
            case LARGE       -> 14.99;
            case EXTRA_LARGE -> 18.99;
        };
    }

    /** Compute total price including add-ons */
    public double getTotalPrice() {
        double price = getBasePrice();
        price += toppings.size() * 1.25;          // $1.25 per topping
        if (extraCheese)  price += 1.50;
        if (crust == Crust.STUFFED) price += 2.00;
        return Math.round(price * 100.0) / 100.0; // round to cents
    }

    @Override
    public String toString() {
        return String.format(
            "Pizza{size=%s(%s), crust=%s, sauce=%s, toppings=%s, " +
            "extraCheese=%b, stuffedCrust=%b, price=$%.2f, notes='%s'}",
            size, size.getInches(), crust.getDisplayName(),
            sauce, toppings, extraCheese, stuffedCrust,
            getTotalPrice(),
            specialInstructions != null ? specialInstructions : "none"
        );
    }

    // ── Inner Static Builder ──────────────────────────────────────────────────

    public static final class Builder {

        // Required
        private final Size  size;
        private final Crust crust;

        // Optional — with defaults
        private Sauce        sauce               = Sauce.TOMATO;
        private List<String> toppings            = new ArrayList<>();
        private boolean      extraCheese         = false;
        private boolean      stuffedCrust        = false;
        private String       specialInstructions = null;

        /** Required fields taken at construction time */
        public Builder(Size size, Crust crust) {
            this.size  = Objects.requireNonNull(size,  "Pizza size is required");
            this.crust = Objects.requireNonNull(crust, "Crust type is required");
        }

        public Builder sauce(Sauce sauce) {
            this.sauce = Objects.requireNonNull(sauce, "Sauce must not be null — use Sauce.NONE for no sauce");
            return this;
        }

        /**
         * Add a single topping. Can be called multiple times.
         * Duplicate toppings are allowed (e.g., extra pepperoni).
         */
        public Builder topping(String topping) {
            Objects.requireNonNull(topping, "Topping must not be null");
            if (topping.isBlank()) {
                throw new IllegalArgumentException("Topping name must not be blank");
            }
            this.toppings.add(topping.strip());
            return this;
        }

        /** Replace all current toppings with the provided list */
        public Builder toppings(List<String> toppings) {
            Objects.requireNonNull(toppings, "Toppings list must not be null");
            this.toppings = new ArrayList<>(toppings);
            return this;
        }

        public Builder extraCheese(boolean extraCheese) {
            this.extraCheese = extraCheese;
            return this;
        }

        public Builder stuffedCrust(boolean stuffedCrust) {
            this.stuffedCrust = stuffedCrust;
            return this;
        }

        public Builder specialInstructions(String instructions) {
            this.specialInstructions = instructions;
            return this;
        }

        public Pizza build() {
            validate();
            return new Pizza(this);
        }

        private void validate() {
            // Business rule: stuffed crust cannot be THIN (physically impossible)
            if (stuffedCrust && crust == Crust.THIN) {
                throw new IllegalStateException(
                    "Stuffed crust is not available with THIN crust. " +
                    "Choose REGULAR, THICK, or STUFFED crust.");
            }
            // Business rule: stuffed crust AND STUFFED crust enum are redundant
            if (crust == Crust.STUFFED && !stuffedCrust) {
                // Auto-correct: if they chose Crust.STUFFED, enable stuffedCrust flag
                this.stuffedCrust = true;
            }
            // Business rule: EXTRA_LARGE pizza must have at least one topping
            if (size == Size.EXTRA_LARGE && toppings.isEmpty()) {
                throw new IllegalStateException(
                    "Extra-large pizzas must have at least one topping.");
            }
            // Business rule: max 10 toppings
            if (toppings.size() > 10) {
                throw new IllegalStateException(
                    "Maximum 10 toppings allowed. Got: " + toppings.size());
            }
        }
    }

    // ── Demo ──────────────────────────────────────────────────────────────────

    public static void main(String[] args) {
        // Classic Margherita
        Pizza margherita = new Pizza.Builder(Size.MEDIUM, Crust.THIN)
            .sauce(Sauce.TOMATO)
            .topping("Fresh Mozzarella")
            .topping("Basil")
            .topping("Cherry Tomatoes")
            .extraCheese(false)
            .specialInstructions("Light on the basil, please")
            .build();
        System.out.println("Margherita: " + margherita);

        // BBQ Meat Feast
        Pizza meatFeast = new Pizza.Builder(Size.LARGE, Crust.THICK)
            .sauce(Sauce.BBQ)
            .topping("Pulled Pork")
            .topping("Crispy Bacon")
            .topping("Pepperoni")
            .topping("Smoked Chicken")
            .extraCheese(true)
            .build();
        System.out.println("Meat Feast: " + meatFeast);

        // Validation failure demo
        try {
            Pizza invalid = new Pizza.Builder(Size.LARGE, Crust.THIN)
                .stuffedCrust(true)   // THIN + stuffed = illegal
                .build();
        } catch (IllegalStateException e) {
            System.out.println("Caught: " + e.getMessage());
        }
    }
}
```

---

### 5.3 Computer Configuration Builder

This example explicitly highlights required vs optional fields, a complex product with meaningful defaults, and a usage pattern common in system configuration.

```java
package com.course.patterns.builder;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import java.util.Optional;

/**
 * Immutable Computer configuration object.
 *
 * <p>Required fields: CPU, RAM, primary storage, OS.
 * <p>Optional fields: GPU (workstation/gaming PCs only), secondary storage,
 * USB ports count, display ports count, WiFi capability, Bluetooth capability.
 *
 * <p>Usage:
 * <pre>{@code
 * // Developer workstation — no GPU needed
 * Computer devMachine = new Computer.Builder("Intel Core i9-14900K", 64, "2TB NVMe SSD", "Ubuntu 24.04")
 *     .secondaryStorage("4TB HDD")
 *     .usbPorts(6)
 *     .displayPorts(3)
 *     .wifi(true)
 *     .bluetooth(true)
 *     .build();
 *
 * // Gaming rig — GPU required
 * Computer gamingRig = new Computer.Builder("AMD Ryzen 9 7950X", 32, "1TB NVMe SSD", "Windows 11 Pro")
 *     .gpu("NVIDIA RTX 4090 24GB")
 *     .secondaryStorage("2TB HDD")
 *     .usbPorts(8)
 *     .build();
 * }</pre>
 */
public final class Computer {

    // ── Required Fields ───────────────────────────────────────────────────────
    private final String cpu;
    private final int    ramGb;
    private final String primaryStorage;
    private final String operatingSystem;

    // ── Optional Fields ───────────────────────────────────────────────────────
    private final String       gpu;               // null if integrated/no GPU
    private final String       secondaryStorage;  // null if single-drive config
    private final int          usbPorts;
    private final int          displayPorts;
    private final boolean      wifi;
    private final boolean      bluetooth;
    private final List<String> installedSoftware; // pre-installed apps

    private Computer(Builder builder) {
        // Required
        this.cpu              = builder.cpu;
        this.ramGb            = builder.ramGb;
        this.primaryStorage   = builder.primaryStorage;
        this.operatingSystem  = builder.operatingSystem;
        // Optional
        this.gpu              = builder.gpu;
        this.secondaryStorage = builder.secondaryStorage;
        this.usbPorts         = builder.usbPorts;
        this.displayPorts     = builder.displayPorts;
        this.wifi             = builder.wifi;
        this.bluetooth        = builder.bluetooth;
        this.installedSoftware = Collections.unmodifiableList(
            new ArrayList<>(builder.installedSoftware)
        );
    }

    // ── Getters ───────────────────────────────────────────────────────────────

    public String           getCpu()               { return cpu; }
    public int              getRamGb()             { return ramGb; }
    public String           getPrimaryStorage()    { return primaryStorage; }
    public String           getOperatingSystem()   { return operatingSystem; }

    /**
     * GPU is optional — returning Optional<String> signals this clearly to callers.
     * Forces callers to handle the absent case explicitly, avoiding NPEs.
     */
    public Optional<String> getGpu()              { return Optional.ofNullable(gpu); }

    /**
     * Secondary storage is optional — same Optional idiom.
     */
    public Optional<String> getSecondaryStorage() { return Optional.ofNullable(secondaryStorage); }

    public int              getUsbPorts()          { return usbPorts; }
    public int              getDisplayPorts()      { return displayPorts; }
    public boolean          hasWifi()              { return wifi; }
    public boolean          hasBluetooth()         { return bluetooth; }
    public List<String>     getInstalledSoftware() { return installedSoftware; }

    /** Derived: is this a gaming/workstation build? */
    public boolean isGamingOrWorkstationBuild() {
        return gpu != null;
    }

    /** Derived: total storage description */
    public String getTotalStorageDescription() {
        if (secondaryStorage == null) {
            return primaryStorage;
        }
        return primaryStorage + " + " + secondaryStorage;
    }

    @Override
    public String toString() {
        return "Computer{\n" +
               "  CPU:             " + cpu             + "\n" +
               "  RAM:             " + ramGb + "GB"   + "\n" +
               "  Primary Storage: " + primaryStorage  + "\n" +
               "  Secondary:       " + (secondaryStorage != null ? secondaryStorage : "N/A") + "\n" +
               "  GPU:             " + (gpu != null ? gpu : "Integrated") + "\n" +
               "  OS:              " + operatingSystem + "\n" +
               "  USB Ports:       " + usbPorts        + "\n" +
               "  Display Ports:   " + displayPorts    + "\n" +
               "  WiFi:            " + wifi            + "\n" +
               "  Bluetooth:       " + bluetooth       + "\n" +
               "  Software:        " + installedSoftware + "\n" +
               "}";
    }

    // ── Inner Static Builder ──────────────────────────────────────────────────

    public static final class Builder {

        // ── Required: passed to the Builder constructor ───────────────────────
        private final String cpu;
        private final int    ramGb;
        private final String primaryStorage;
        private final String operatingSystem;

        // ── Optional: sensible defaults ───────────────────────────────────────
        private String       gpu               = null;    // no dedicated GPU by default
        private String       secondaryStorage  = null;    // single-drive by default
        private int          usbPorts          = 4;       // modern minimum
        private int          displayPorts      = 1;       // single display by default
        private boolean      wifi              = true;    // almost always present
        private boolean      bluetooth         = true;    // almost always present
        private List<String> installedSoftware = new ArrayList<>();

        /**
         * All four required parameters must be provided at Builder construction time.
         * This makes the required/optional distinction explicit in the API.
         *
         * @param cpu             CPU model string, e.g. "Intel Core i9-14900K"
         * @param ramGb           RAM in gigabytes — must be a power of 2, minimum 4
         * @param primaryStorage  primary storage description, e.g. "1TB NVMe SSD"
         * @param operatingSystem OS name, e.g. "Windows 11 Pro", "Ubuntu 24.04", "macOS 15 Sequoia"
         */
        public Builder(String cpu, int ramGb, String primaryStorage, String operatingSystem) {
            this.cpu             = Objects.requireNonNull(cpu, "CPU is required").strip();
            this.primaryStorage  = Objects.requireNonNull(primaryStorage, "Primary storage is required").strip();
            this.operatingSystem = Objects.requireNonNull(operatingSystem, "Operating system is required").strip();

            if (ramGb < 4) {
                throw new IllegalArgumentException("RAM must be at least 4GB, got: " + ramGb);
            }
            if (!isPowerOfTwo(ramGb)) {
                throw new IllegalArgumentException(
                    "RAM must be a power of 2 (4, 8, 16, 32, 64, 128...), got: " + ramGb);
            }
            this.ramGb = ramGb;
        }

        /** Specify a dedicated GPU. Leave unset for integrated graphics. */
        public Builder gpu(String gpu) {
            Objects.requireNonNull(gpu, "GPU model must not be null — omit this method for integrated graphics");
            this.gpu = gpu.strip();
            return this;
        }

        /** Add a secondary storage device (HDD, additional SSD). */
        public Builder secondaryStorage(String secondaryStorage) {
            this.secondaryStorage = Objects.requireNonNull(secondaryStorage).strip();
            return this;
        }

        public Builder usbPorts(int count) {
            if (count < 0) throw new IllegalArgumentException("USB port count cannot be negative");
            this.usbPorts = count;
            return this;
        }

        public Builder displayPorts(int count) {
            if (count < 0) throw new IllegalArgumentException("Display port count cannot be negative");
            this.displayPorts = count;
            return this;
        }

        public Builder wifi(boolean wifi) {
            this.wifi = wifi;
            return this;
        }

        public Builder bluetooth(boolean bluetooth) {
            this.bluetooth = bluetooth;
            return this;
        }

        /** Pre-install a software package. Can be called multiple times. */
        public Builder preInstall(String softwareName) {
            Objects.requireNonNull(softwareName, "Software name must not be null");
            this.installedSoftware.add(softwareName.strip());
            return this;
        }

        public Computer build() {
            validate();
            return new Computer(this);
        }

        private void validate() {
            if (cpu.isBlank()) {
                throw new IllegalStateException("CPU model must not be blank");
            }
            if (primaryStorage.isBlank()) {
                throw new IllegalStateException("Primary storage description must not be blank");
            }
            if (operatingSystem.isBlank()) {
                throw new IllegalStateException("Operating system must not be blank");
            }
            // Business rule: high RAM without a GPU on a likely gaming build is suspicious
            // (this is a warning, not a hard error)
            if (ramGb >= 64 && gpu == null) {
                System.err.println("[WARN] Computer.Builder: 64GB+ RAM configured without a dedicated GPU. " +
                                   "Is this intentional for a server/workstation?");
            }
            // Business rule: more than 8 USB ports is unusual — flag it
            if (usbPorts > 8) {
                System.err.println("[WARN] Computer.Builder: " + usbPorts + " USB ports is unusually high.");
            }
        }

        private static boolean isPowerOfTwo(int n) {
            return n > 0 && (n & (n - 1)) == 0;
        }
    }

    // ── Demo ──────────────────────────────────────────────────────────────────

    public static void main(String[] args) {
        // Developer workstation
        Computer devMachine = new Computer.Builder(
                "Intel Core i9-14900K", 64, "2TB NVMe SSD", "Ubuntu 24.04 LTS")
            .secondaryStorage("4TB SATA HDD")
            .displayPorts(3)
            .usbPorts(6)
            .preInstall("IntelliJ IDEA")
            .preInstall("Docker Desktop")
            .preInstall("Git 2.43")
            .build();
        System.out.println("Developer Machine:\n" + devMachine);

        // Gaming rig
        Computer gamingRig = new Computer.Builder(
                "AMD Ryzen 9 7950X", 32, "1TB Gen5 NVMe SSD", "Windows 11 Pro")
            .gpu("NVIDIA RTX 4090 24GB GDDR6X")
            .secondaryStorage("2TB HDD")
            .displayPorts(4)
            .usbPorts(8)
            .preInstall("Steam")
            .preInstall("NVIDIA GeForce Experience")
            .build();
        System.out.println("Gaming Rig:\n" + gamingRig);

        // Check optional GPU usage with Optional
        gamingRig.getGpu().ifPresentOrElse(
            g -> System.out.println("GPU found: " + g),
            () -> System.out.println("No dedicated GPU")
        );

        devMachine.getGpu().ifPresentOrElse(
            g -> System.out.println("GPU found: " + g),
            () -> System.out.println("Dev machine: No dedicated GPU — using integrated")
        );
    }
}
```

---

## 6. Fluent API and Method Chaining

### What is Fluent API?

A **Fluent API** (coined by Martin Fowler and Eric Evans in 2005) is an interface design style where method calls read like natural language sentences. The key mechanism enabling it in Builder is that every setter method returns `this` — the Builder instance — allowing calls to be chained without intermediate variables.

### The Mechanics

```java
// WITHOUT fluent API — clunky, requires intermediate variable
HttpRequest.Builder builder = new HttpRequest.Builder("POST", "https://api.example.com");
builder.header("Content-Type", "application/json");
builder.body("{\"key\":\"value\"}");
builder.timeout(3000);
HttpRequest request = builder.build();

// WITH fluent API — reads left-to-right, top-to-bottom like a sentence
HttpRequest request = new HttpRequest.Builder("POST", "https://api.example.com")
    .header("Content-Type", "application/json")
    .body("{\"key\":\"value\"}") // "then set the body to..."
    .timeout(3000)               // "then set the timeout to..."
    .build();                    // "then build the request"
```

### How Each Setter Enables Chaining

```java
// Pattern: every setter modifies state then returns 'this'
public Builder timeout(int timeoutMs) {
    if (timeoutMs <= 0) {
        throw new IllegalArgumentException("Timeout must be positive");
    }
    this.timeoutMs = timeoutMs;
    return this;  // ← this single line enables chaining
}

// If the return type were void:
//   builder.timeout(3000);   — returns nothing, chain breaks
// Because it returns Builder:
//   builder.timeout(3000).retryCount(2).build();  — chain continues
```

### Fluent API Design Principles

```java
// PRINCIPLE 1: Name setters like verbs or prepositions, not JavaBean style
// BAD (JavaBean style — breaks the sentence):
builder.setMethod("POST").setUrl("...").setBody("...");
// GOOD (verb/preposition style — reads naturally):
builder.method("POST").url("...").body("...");

// PRINCIPLE 2: Accept the most natural type for the caller
// BAD:
builder.timeout(Duration.ofMillis(3000));    // verbose for common cases
// GOOD (accept both):
builder.timeout(3000)                         // int milliseconds
builder.timeout(Duration.ofSeconds(3))        // Duration overload

// PRINCIPLE 3: Provide additive methods for collection fields
// BAD: forces caller to build a list externally
builder.toppings(Arrays.asList("Cheese", "Pepperoni", "Mushrooms"));
// GOOD: additive methods accumulate naturally
builder.topping("Cheese").topping("Pepperoni").topping("Mushrooms");

// PRINCIPLE 4: Self-documenting via method names
// Compare:
new HttpRequest("POST", url, headers, body, 3000, token, true, 2);  // what is 'true'??
// vs:
new HttpRequest.Builder("POST", url).body(body).timeout(3000)
    .authToken(token).followRedirects(true).retryCount(2).build();
```

---

## 7. Director Class

### What is a Director?

In the canonical GoF Builder pattern, the **Director** class encapsulates the construction logic — the sequence in which to call Builder methods to produce a specific type of product. The Director knows the recipe; the Builder knows how to execute each step.

### When to Use a Director

Use a Director when:
- You have **multiple predefined configurations** of the same product type
- The **same sequence of steps** is needed in many places across the codebase
- You want to **decouple callers** from knowing the construction steps
- You are building a **configurable framework** where users provide a Builder and a Director runs a standard build pipeline

### Director Example

```java
package com.course.patterns.builder;

/**
 * Director encapsulates construction sequences for common HttpRequest configurations.
 * Clients use the Director to get standard request types without knowing the steps.
 */
public class HttpRequestDirector {

    private HttpRequest.Builder builder;

    public HttpRequestDirector(HttpRequest.Builder builder) {
        this.builder = builder;
    }

    public void setBuilder(HttpRequest.Builder builder) {
        this.builder = builder;
    }

    /**
     * Constructs a standard health-check GET request.
     * Callers don't need to know which headers/timeouts are standard.
     */
    public HttpRequest constructHealthCheckRequest(String baseUrl) {
        return new HttpRequest.Builder("GET", baseUrl + "/health")
            .header("Accept", "application/json")
            .header("X-Request-Source", "health-monitor")
            .timeout(1000)      // short timeout for health checks
            .followRedirects(false)
            .retryCount(0)      // health checks do not retry
            .build();
    }

    /**
     * Constructs a standard authenticated API POST request.
     * Applies the company-standard headers and timeout policies.
     */
    public HttpRequest constructAuthenticatedPost(String url, String body, String bearerToken) {
        return new HttpRequest.Builder("POST", url)
            .header("Content-Type", "application/json")
            .header("Accept", "application/json")
            .header("X-API-Version", "v2")
            .header("X-Correlation-ID", java.util.UUID.randomUUID().toString())
            .body(body)
            .authToken("Bearer " + bearerToken)
            .timeout(10_000)    // 10 second standard timeout for write operations
            .followRedirects(false)
            .retryCount(3)      // retry transient failures
            .build();
    }

    /**
     * Constructs a webhook notification POST (no auth, short timeout, no retry).
     */
    public HttpRequest constructWebhookPost(String webhookUrl, String payload) {
        return new HttpRequest.Builder("POST", webhookUrl)
            .header("Content-Type", "application/json")
            .header("X-Webhook-Source", "my-service")
            .body(payload)
            .timeout(3000)
            .followRedirects(false)
            .retryCount(1)  // try once more on transient failure
            .build();
    }
}

// ── Usage of Director ──────────────────────────────────────────────────────────

class DirectorUsageDemo {
    public static void main(String[] args) {
        HttpRequestDirector director = new HttpRequestDirector(null);

        // Standard health check
        HttpRequest healthCheck = director.constructHealthCheckRequest("https://api.example.com");
        System.out.println("Health check: " + healthCheck);

        // Authenticated POST
        HttpRequest apiPost = director.constructAuthenticatedPost(
            "https://api.example.com/v2/orders",
            "{\"product\":\"SKU-001\",\"qty\":5}",
            "eyJhbGciOiJIUzI1NiJ9..."
        );
        System.out.println("API POST: " + apiPost);
    }
}
```

### Pizza Director Example

```java
package com.course.patterns.builder;

/**
 * Director that knows standard pizza recipes.
 * Encapsulates the recipe steps so menu items are defined in one place.
 */
public class PizzaDirector {

    /** Build a Margherita according to the classic recipe */
    public Pizza buildClassicMargherita(Pizza.Size size) {
        return new Pizza.Builder(size, Pizza.Crust.THIN)
            .sauce(Pizza.Sauce.TOMATO)
            .topping("Fresh Mozzarella")
            .topping("San Marzano Tomatoes")
            .topping("Fresh Basil")
            .extraCheese(false)
            .specialInstructions("Add fresh basil after baking")
            .build();
    }

    /** Build a Four-Cheese pizza */
    public Pizza buildFourCheese(Pizza.Size size) {
        return new Pizza.Builder(size, Pizza.Crust.REGULAR)
            .sauce(Pizza.Sauce.TOMATO)
            .topping("Mozzarella")
            .topping("Parmesan")
            .topping("Gorgonzola")
            .topping("Fontina")
            .extraCheese(true)
            .build();
    }

    /** Build a Meat Feast */
    public Pizza buildMeatFeast(Pizza.Size size) {
        return new Pizza.Builder(size, Pizza.Crust.THICK)
            .sauce(Pizza.Sauce.BBQ)
            .topping("Pepperoni")
            .topping("Italian Sausage")
            .topping("Smoked Bacon")
            .topping("Pulled Pork")
            .topping("Ground Beef")
            .extraCheese(true)
            .build();
    }
}
```

### Director vs No-Director: Decision

- **Without Director**: client code chains the Builder directly. Good for **unique, one-off constructions** or when the product has only one standard form.
- **With Director**: client code calls `director.buildX()`. Good for **named configurations**, **menu items**, **environment profiles**, or anything where the same recipe appears in multiple places.

---

## 8. Immutability Through Builder

### Why Immutability Matters

Immutable objects are:
- **Inherently thread-safe** — no synchronization needed; multiple threads can read simultaneously
- **Safe to share** — you can pass an `HttpRequest` to any method or thread without defensive copies
- **Simple to reason about** — the object's state never changes; no "what was it set to last?" bugs
- **Safe as HashMap keys and Set members** — hashCode cannot change

### How Builder Enforces Immutability

The combination of three design decisions locks down immutability:

```java
public final class HttpRequest {      // (1) final class — no mutable subclass possible

    private final String method;      // (2) all fields are final
    private final Map<String, String> headers;

    private HttpRequest(Builder b) {  // (3) private constructor — only Builder can create
        this.method  = b.method;
        // (4) Defensive copy of mutable fields
        this.headers = Collections.unmodifiableMap(new HashMap<>(b.headers));
        //             ^^^                          ^^^
        //  wraps in unmodifiable view    defensive copy so builder's map changes don't affect us
    }

    // NO SETTERS — the class exposes only getters
    public String getMethod() { return method; }
    public Map<String, String> getHeaders() { return headers; } // returns unmodifiable view
}
```

### The Four Immutability Rules

```java
// RULE 1: Declare the class final
// Prevents: new MutableHttpRequest extends HttpRequest { /* adds setters */ }
public final class HttpRequest { ... }

// RULE 2: Declare all fields private final
// Prevents: any assignment after constructor completes
private final String method;

// RULE 3: No setters
// Not having setters means no external code can mutate state

// RULE 4: Defensive copy of mutable fields in constructor
// Without defensive copy:
this.headers = b.headers;  // WRONG — builder's map is shared; builder.header() after build() mutates us
// With defensive copy:
this.headers = Collections.unmodifiableMap(new HashMap<>(b.headers));  // CORRECT

// Also needed for collection getters:
public List<String> getToppings() {
    return toppings; // safe because toppings is already Collections.unmodifiableList(...)
    // alternatively: return Collections.unmodifiableList(new ArrayList<>(toppings));
}
```

### Thread Safety Proof

```java
// Once build() returns, no thread can mutate the HttpRequest.
// Multiple threads can read it concurrently with zero synchronization:
HttpRequest request = new HttpRequest.Builder("GET", "https://api.example.com")
    .timeout(3000)
    .build();

// These can run on 100 different threads simultaneously — perfectly safe:
ExecutorService pool = Executors.newFixedThreadPool(100);
for (int i = 0; i < 100; i++) {
    pool.submit(() -> {
        String method = request.getMethod(); // read-only, safe
        doSomething(method);
    });
}
```

---

## 9. Real-World Examples

### 9.1 Java's StringBuilder (Precursor to Builder Pattern)

`StringBuilder` is not the GoF Builder pattern — it builds a `String` (a product), but `StringBuilder` itself is mutable. It demonstrates the **accumulation principle** and fluent chaining, which informed the GoF Builder.

```java
String result = new StringBuilder()
    .append("Hello")
    .append(", ")
    .append("World")
    .append("!")
    .toString();  // ← analogous to build()

// StringBuilder shows:
// 1. Accumulation — append() accumulates state
// 2. Method chaining — each append() returns 'this'
// 3. Termination — toString() finalizes and returns the product
// It differs from GoF Builder: StringBuilder IS mutable (it's not the product)
```

### 9.2 Lombok @Builder Annotation

Lombok's `@Builder` eliminates the boilerplate of writing the inner Builder class:

```java
import lombok.Builder;
import lombok.Getter;
import lombok.NonNull;
import java.util.Map;

// Lombok generates the entire Builder inner class at compile time
@Getter
@Builder(toBuilder = true)  // toBuilder = true adds a toBuilder() method for copying with changes
public final class HttpRequest {

    @NonNull
    private final String method;

    @NonNull
    private final String url;

    @Builder.Default
    private final int timeoutMs = 5000;      // @Builder.Default sets a default value

    private final String body;               // null = absent
    private final String authToken;          // null = absent

    @Builder.Default
    private final boolean followRedirects = true;

    @Builder.Default
    private final int retryCount = 0;
}

// Generated usage (identical API to hand-written Builder):
HttpRequest req = HttpRequest.builder()
    .method("POST")
    .url("https://api.example.com/users")
    .body("{\"name\":\"Alice\"}")
    .timeoutMs(3000)
    .authToken("Bearer token123")
    .build();

// Copy and modify (toBuilder = true enables this):
HttpRequest retryReq = req.toBuilder()
    .retryCount(3)
    .build();  // new object; original is unchanged
```

**What Lombok generates behind the scenes:**

```java
// Lombok expands @Builder into approximately this (simplified):
public static final class HttpRequestBuilder {
    private String method;
    private String url;
    private int timeoutMs = 5000;
    private String body;
    private String authToken;
    private boolean followRedirects = true;
    private int retryCount = 0;

    public HttpRequestBuilder method(String method) { this.method = method; return this; }
    public HttpRequestBuilder url(String url)       { this.url = url;       return this; }
    // ... all other setters ...
    public HttpRequest build() {
        return new HttpRequest(method, url, timeoutMs, body, authToken, followRedirects, retryCount);
    }
}
public static HttpRequestBuilder builder() { return new HttpRequestBuilder(); }
```

### 9.3 OkHttp Request.Builder

OkHttp's `Request.Builder` is one of the cleanest real-world examples of this pattern:

```java
// OkHttp uses exactly the inner static Builder pattern
import okhttp3.*;

OkHttpClient client = new OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .addInterceptor(loggingInterceptor)
    .build();

Request request = new Request.Builder()
    .url("https://api.example.com/users")
    .addHeader("Authorization", "Bearer " + token)
    .addHeader("Content-Type", "application/json")
    .post(RequestBody.create(json, MediaType.get("application/json")))
    .build();

// OkHttp also uses Builder for Response, FormBody, MultipartBody...
FormBody form = new FormBody.Builder()
    .add("username", "alice")
    .add("password", "secret")
    .build();
```

### 9.4 Protocol Buffers (Protobuf) Builder

Google's Protocol Buffers generate a Builder for every message type — this is arguably the most widely-deployed Builder in production systems globally:

```java
// Generated from a .proto file — you write this:
// message UserRequest {
//   string user_id = 1;
//   string email = 2;
//   repeated string roles = 3;
//   int32 timeout_ms = 4;
// }

// Protobuf generates a Builder:
UserRequest request = UserRequest.newBuilder()
    .setUserId("usr-001")
    .setEmail("alice@example.com")
    .addRoles("ADMIN")
    .addRoles("USER")
    .setTimeoutMs(5000)
    .build();                    // returns immutable UserRequest

// Modification is copy-on-write:
UserRequest modified = request.toBuilder()
    .setEmail("alice2@example.com")
    .build();                    // returns a new immutable UserRequest
```

### 9.5 Java Stream.Builder

`Stream.Builder` is the GoF Builder for building a `Stream<T>` when you need to add elements dynamically (not from an existing collection):

```java
Stream.Builder<String> streamBuilder = Stream.builder();
streamBuilder.accept("alpha");
if (someCondition) {
    streamBuilder.accept("beta");
}
streamBuilder.accept("gamma");
Stream<String> stream = streamBuilder.build();
// After build(), the builder is "consumed" — calling accept() again throws IllegalStateException
stream.forEach(System.out::println);
```

### 9.6 Java ProcessBuilder

`ProcessBuilder` builds OS-level processes — another real JDK example:

```java
ProcessBuilder pb = new ProcessBuilder("git", "log", "--oneline", "-10")
    .directory(new File("/path/to/repo"))
    .redirectErrorStream(true);

pb.environment().put("GIT_TERMINAL_PROMPT", "0");

Process process = pb.start();   // analogous to build()
```

### 9.7 Spring's MockMvcRequestBuilders (Testing)

```java
// In Spring MVC tests — Builder pattern everywhere
mockMvc.perform(
    MockMvcRequestBuilders.post("/api/users")
        .contentType(MediaType.APPLICATION_JSON)
        .header("Authorization", "Bearer " + token)
        .content("{\"name\":\"Alice\"}")
)
.andExpect(status().isCreated())
.andExpect(jsonPath("$.id").exists());
```

---

## 10. Builder vs Factory - Decision Table

These two patterns are frequently confused. Use this table to choose the right one:

| Decision Factor | Builder | Factory (Factory Method / Abstract Factory) |
|-----------------|---------|---------------------------------------------|
| **Primary goal** | Construct a **complex object** step by step | **Encapsulate and delegate** object creation |
| **Number of fields** | Many (4+), often with optional fields | Few; often just a type discriminator |
| **Object complexity** | High — multiple fields, defaults, validation | Low to medium |
| **Product type** | **One type** of product (e.g., HttpRequest) | **Multiple related types** (e.g., WindowsButton vs MacButton) |
| **Construction steps** | Steps can be **ordered and optional** | Steps are **fixed and internal** to the factory |
| **Caller awareness** | Caller specifies what they want step by step | Caller says "give me an X" — factory decides how |
| **Immutability of product** | Natural — `build()` creates final immutable object | Possible but not inherent |
| **Subclass variation** | Via different Builder implementations | Via factory method override in subclasses |
| **Runtime type selection** | Less common | Core use case — type chosen at runtime |
| **Readability of creation** | High — fluent chain is self-documenting | Moderate — single method call, less context |
| **Validation** | Natural single point in `build()` | Must be scattered in factory or product constructor |

### Code-Level Comparison

```java
// USE BUILDER WHEN: many optional fields, readable config, immutable result
HttpRequest request = new HttpRequest.Builder("POST", url)
    .header("Content-Type", "application/json")
    .body(payload)
    .timeout(3000)
    .retryCount(2)
    .build();

// USE FACTORY METHOD WHEN: hiding which concrete type to create
// Caller just says "create me a shape" — factory picks the concrete type
public abstract class ShapeFactory {
    // Factory method — subclass decides the concrete type
    public abstract Shape createShape();

    public void draw() {
        Shape shape = createShape(); // delegates to subclass
        shape.render();
    }
}
class CircleFactory extends ShapeFactory {
    @Override public Shape createShape() { return new Circle(10); }
}

// USE ABSTRACT FACTORY WHEN: families of related objects
// e.g., UI toolkit — you want all widgets to match the OS theme
UIFactory factory = isWindows ? new WindowsUIFactory() : new MacOSUIFactory();
Button btn    = factory.createButton();    // WindowsButton or MacButton
TextField txt = factory.createTextField(); // WindowsTextField or MacTextField

// SUMMARY HEURISTIC:
// - "I need to configure a complex object with many parameters" → Builder
// - "I need to create one of several related types" → Factory Method / Abstract Factory
// - "I need to centralize and hide instantiation logic" → Factory
// - "I need the product to be immutable and thread-safe" → Builder
```

---

## 11. Trade-offs

### Advantages

| Advantage | Explanation |
|-----------|-------------|
| **Readable, self-documenting code** | `.timeout(3000).retryCount(2)` is immediately clear; `new Obj(3000, 2)` is not |
| **Eliminates telescoping constructors** | No more `new Obj(a, b, null, null, 5000, null, true, 0)` |
| **Single validation point** | All cross-field validation lives in `build()` — easy to find, test, and audit |
| **Enforces immutability** | Private constructor + final fields = thread-safe, shareable object |
| **Optional fields with defaults** | Builder can hold sensible defaults; callers only override what they need |
| **Safe multi-step construction** | Object exists in a consistent state only after `build()` — no half-constructed objects shared |
| **Easy to extend** | Adding a new optional field to the Builder is backward-compatible — existing call sites continue to compile |
| **Testability** | Builders are easy to construct in tests — no need to mock complex constructors |

### Disadvantages

| Disadvantage | Explanation |
|--------------|-------------|
| **Boilerplate (without Lombok)** | Hand-written Builder duplicates every field in two places (product + Builder) |
| **Extra heap allocation** | Each `build()` call creates both a Builder and a product object — doubles allocation in hot paths |
| **Partial construction is valid** | A Builder can exist without calling `build()` — this is rarely a problem but worth knowing |
| **Mutability during construction** | The Builder itself is mutable; sharing a Builder across threads before `build()` is unsafe |
| **Required fields can be forgotten** | If you put required fields as Builder setters (not constructor params), they can be missed — build() must check |
| **Can be overkill** | For simple 2-3 field objects, a record or plain constructor is cleaner |
| **No compile-time enforcement** | Unlike constructor parameters, omitting a Builder setter is not a compile error — it's a runtime error in `build()` |

### Mitigation Strategies

```java
// MITIGATION 1: Put required fields in the Builder constructor (not setters)
// → compile-time enforcement of required fields
new HttpRequest.Builder("POST", url)  // method and url are required — won't compile without them

// MITIGATION 2: Use Lombok @Builder to eliminate boilerplate
@Builder
public final class HttpRequest { ... }

// MITIGATION 3: Use @NonNull (Lombok) or Objects.requireNonNull() in setters
// for fail-fast validation before build()
public Builder body(String body) {
    this.body = Objects.requireNonNull(body, "body");
    return this;
}

// MITIGATION 4: Document defaults in Javadoc so callers know what they're getting
/**
 * @param timeoutMs connection + read timeout (default: 5000ms)
 */
public Builder timeout(int timeoutMs) { ... }
```

---

## 12. Interview Questions and Answers

### Q1: What is the Builder pattern and what problem does it solve?

**Answer:**

The Builder pattern is a creational design pattern that separates the construction of a complex object from its representation. It solves three distinct but related problems that arise when constructing objects with many fields.

The first problem is the **telescoping constructor anti-pattern**: when an object has N optional fields, you need up to 2^N constructor overloads to handle every combination, or you chain constructors (each calling the next with one more parameter). Either approach explodes in complexity and produces call sites where positional arguments are indistinguishable — `new Obj(240, 8, 100, 0, 35, 27)` gives no indication of what each integer means.

The second problem is the **JavaBeans alternative's mutability flaw**: a no-arg constructor plus setters lets the object exist in an inconsistent intermediate state (e.g., after `setUrl()` but before `setMethod()`). This makes the object unsafe to share between threads mid-construction and prevents immutability.

The third problem is **validation timing**: with constructors or setters, cross-field validation is scattered or impossible (you can only validate one field at a time). Builder concentrates all validation in the `build()` method, where all fields are available simultaneously.

The Builder pattern resolves all three by accumulating field assignments in a mutable companion `Builder` object, then creating the immutable product in a single `build()` call that validates cross-field constraints.

---

### Q2: How does Builder enforce immutability, and why does this matter?

**Answer:**

Builder enforces immutability through four cooperating mechanisms. First, the product class is declared `final` — no subclass can add mutable fields. Second, all fields in the product are declared `private final` — they can only be assigned in the constructor, never again. Third, the constructor is `private` — only the inner `Builder.build()` method can call it, eliminating any other path to creating a partially-initialized instance. Fourth, any mutable fields (collections, arrays, mutable objects) receive a **defensive copy** in the constructor, wrapped in an unmodifiable view: `Collections.unmodifiableMap(new HashMap<>(builder.headers))`.

Immutability matters enormously in concurrent systems. An immutable object can be freely shared between threads with zero synchronization overhead — reads are always safe because the state never changes. There is no "check-then-act" race condition. This is why the JDK's `String`, `Integer`, `BigDecimal`, and the standard record types are immutable. In microservice architectures, request/response DTOs that flow through thread pools, futures, and reactive streams benefit greatly from immutability — you never need to ask "was this object mutated after I received it?"

A subtle point: the `Builder` itself is mutable — that is by design, since it accumulates state during construction. But the product returned by `build()` is immutable. This is the fundamental contract: mutable accumulation (Builder) → single construction event → immutable result (Product).

---

### Q3: What is the difference between Builder and Director, and when do you need a Director?

**Answer:**

The `Builder` is responsible for knowing **how** to assemble each piece of the product — how to add a header, set a timeout, add a topping. The `Director` is responsible for knowing **which pieces** to assemble in **which sequence** to produce a specific named configuration. They are collaborators with different responsibilities.

You need a Director when you have multiple predefined configurations of the same product type that appear in more than one place in the codebase. For example, if you have a `PizzaDirector` with `buildMargherita()`, `buildMeatFeast()`, and `buildVeganOption()`, each of these encapsulates a standard recipe. Without a Director, every call site that wants a Margherita must know all the toppings, crust type, and sauce — duplicating the recipe everywhere. If the recipe changes (say, we switch from mozzarella to vegan cheese in the Margherita), you have one place to update instead of N.

In practice, the Director is **optional** and often omitted in the Effective Java style of Builder (where the Builder's constructor takes required fields directly). Directors are more commonly found in frameworks and configuration-heavy systems — for example, a `ReportDirector` that knows how to configure a PDF report versus an HTML report by calling the same `ReportBuilder` in different sequences. When you have only one type of product with no predefined recipes, using the Builder directly without a Director is cleaner.

---

### Q4: How does Builder differ from Factory Method and Abstract Factory?

**Answer:**

These three patterns all deal with object creation, but they address different problems and operate at different levels of abstraction.

**Factory Method** addresses the question "which concrete class should be instantiated?" It defines an interface for creating an object but lets subclasses decide which class to instantiate. The caller asks for "a Shape" and the factory decides whether to return a Circle or Rectangle. There are no configuration parameters — the factory decides everything internally.

**Abstract Factory** extends this to "families of related objects" — it produces suites of objects that belong together (e.g., a MacOSUIFactory produces MacButton, MacTextField, MacScrollbar, all visually consistent). Again, configuration parameters are not the focus; type selection and family consistency are.

**Builder** addresses the question "how do I construct a single complex object with many optional parameters?" The type of object being constructed is known — an `HttpRequest` is always an `HttpRequest`. What varies is the configuration. Builder gives callers fine-grained control over every field, with a fluent API that makes the configuration readable.

A practical heuristic: if you are choosing between "which type" to create, reach for Factory. If you are configuring "one type" with many options, reach for Builder. Factories hide type selection; Builders expose field configuration. They can also be combined — a `ComputerFactory` might internally use a `Computer.Builder` to assemble different preset configurations.

---

### Q5: What are the dangers of putting required fields as Builder setters instead of Builder constructor parameters?

**Answer:**

When required fields are expressed as Builder setters, calling `build()` without them produces a runtime exception rather than a compile-time error. This is a category shift from a hard, safe failure to a soft, deferred failure.

Consider `new HttpRequest.Builder().method("POST").url("...").build()` — if `method()` is a setter. Now consider a developer who writes `new HttpRequest.Builder().url("...").build()` — the compiler says nothing. The error surfaces only at runtime, possibly in production, when the `build()` method throws `IllegalStateException: method is required`. This kind of bug is hard to catch in reviews and may not be caught by tests that exercise the happy path.

The correct design, as demonstrated in `Effective Java` Item 2, is to put genuinely required fields in the `Builder` constructor: `new HttpRequest.Builder("POST", "https://...")`. Now omitting `method` or `url` is a compile error — it is impossible to create a Builder at all without them. The rule of thumb is: if the product can never be valid without a field, that field belongs in the Builder constructor. If the product has a meaningful default for a field (e.g., `timeout = 5000ms`), that field belongs as a Builder setter with a default.

A secondary danger: required-as-setter also means you lose the ability to make those fields `final` in the Builder, since they must be assigned after construction. This makes the Builder harder to reason about.

---

### Q6: How would you implement copy-and-modify semantics with Builder (the "withX" pattern)?

**Answer:**

Immutable objects built by Builders cannot be mutated after `build()`. This is usually desirable, but sometimes you need to take an existing object and produce a slightly modified copy — changing only one or two fields while keeping all others the same.

The clean solution is a `toBuilder()` method on the product that returns a pre-populated Builder:

```java
public final class HttpRequest {
    // ... fields ...

    /** Return a Builder pre-populated with all this request's current values. */
    public Builder toBuilder() {
        return new Builder(this.method, this.url)
            .headers(this.headers)
            .body(this.body)
            .timeout(this.timeoutMs)
            .authToken(this.authToken)
            .followRedirects(this.followRedirects)
            .retryCount(this.retryCount);
    }
}

// Usage:
HttpRequest original = new HttpRequest.Builder("GET", "https://api.example.com/users")
    .timeout(3000)
    .build();

// Copy with only the timeout changed:
HttpRequest slower = original.toBuilder()
    .timeout(10_000)
    .build();

// original is unchanged — slower is a new object with different timeout
```

This is exactly what Protobuf does with `message.toBuilder().setX(newValue).build()`, and what Lombok's `@Builder(toBuilder = true)` generates automatically. The pattern preserves immutability while enabling the ergonomics of mutation — callers see a clean API while the underlying model remains immutable.

This is particularly valuable in event sourcing and functional programming contexts, where you derive new state from old state by copying and adjusting, never mutating in place.

---

### Q7: When should you NOT use the Builder pattern?

**Answer:**

Builder should be avoided when it adds overhead without providing proportional benefit.

For **simple objects with 1-3 required fields and no optional fields**, a direct constructor is cleaner. `new Point(x, y)` is more readable than `new Point.Builder().x(5).y(10).build()`. Java 14+ records make this even cleaner: `record Point(int x, int y) {}`.

For **JPA entities and Jackson-deserialized DTOs**, frameworks typically require a no-arg constructor and mutable fields with setters. Forcing Builder onto these classes fights the framework.

For **performance-critical hot paths** where object creation happens millions of times per second, the double allocation (Builder + Product) can matter. In such cases, object pooling or direct field assignment may be warranted.

For **highly dynamic objects** where you do not know all fields at construction time and need to modify them after creation, Builder-enforced immutability is a hindrance rather than a help.

The broader principle: design patterns solve specific problems. Applying Builder just because an object "looks complex" without the actual pain of telescoping constructors or mutability issues is over-engineering. Always identify the problem first, then select the pattern that solves it.

---

### Q8: How does Lombok's @Builder compare to hand-written Builder, and what are its limitations?

**Answer:**

Lombok's `@Builder` is a compile-time annotation processor that generates the inner Builder class, the static `builder()` factory method, and optionally a `toBuilder()` method. It eliminates nearly all the boilerplate of hand-written Builders — you declare the fields once, and Lombok duplicates them in the Builder automatically. For well-understood cases, this is excellent: the API it generates is identical to a hand-written Builder, and the compiled bytecode is indistinguishable.

However, Lombok's `@Builder` has several limitations. First, it does not generate `@Builder.Default` values for primitive fields unless you explicitly annotate them, and the interplay between `@Builder.Default` and field initializers can be counterintuitive — the field initializer is ignored; you must use `@Builder.Default`. Second, Lombok cannot generate cross-field validation in `build()` without customization — you must add a `@Builder` with a custom `build()` method override, which partially defeats the simplicity advantage. Third, Lombok is a build-tool dependency that some teams avoid for transparency reasons — the generated code is invisible in source control, making it harder to understand what exactly gets compiled.

For production APIs where validation logic is non-trivial, many teams write the Builder manually (or use a record + a hand-written factory method) specifically so the `build()` validation is explicit, readable, and testable. Lombok and hand-written Builders are complementary choices: Lombok for simple value objects where the defaults are fine; hand-written for complex domain objects with significant validation or lifecycle concerns.

---

### Summary Reference Card

```
Builder Pattern at a Glance
═══════════════════════════════════════════════════════════════
PROBLEM:  Complex object with many optional fields
          → Telescoping constructors, JavaBeans mutability

SOLUTION: Mutable Builder accumulates state
          Immutable Product created in build()

STRUCTURE:
  Product       — final class, final fields, private constructor
  Builder       — inner static class, one setter per field,
                  each setter returns 'this', build() validates

FLUENCY:
  new Product.Builder(required1, required2)
      .optionalA(valueA)
      .optionalB(valueB)
      .build()

IMMUTABILITY RECIPE:
  1. final class
  2. private final fields
  3. private constructor(Builder)
  4. defensive copy of mutable fields
  5. no setters

DIRECTOR: optional; encapsulates named build recipes

WHEN TO USE:
  ✓ 4+ fields, many optional
  ✓ Immutability needed
  ✓ Cross-field validation
  ✓ Fluent/DSL API design

WHEN TO SKIP:
  ✗ 2-3 fields, all required → use record / direct constructor
  ✗ Framework needs no-arg + setters → use JavaBeans
  ✗ Extremely hot allocation path → profile first

REAL WORLD:
  OkHttp Request.Builder, Protobuf generated builders,
  Lombok @Builder, Java ProcessBuilder, Spring MockMvc
═══════════════════════════════════════════════════════════════
```
