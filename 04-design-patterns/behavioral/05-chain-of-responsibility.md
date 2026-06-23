# Chain of Responsibility Design Pattern

> **Category:** Behavioral | **GoF Pattern #14** | **Difficulty:** Intermediate–Advanced

---

## Table of Contents

1. [Intent and Problem Statement](#1-intent-and-problem-statement)
2. [When to Use / When NOT to Use](#2-when-to-use--when-not-to-use)
3. [UML Class Diagram](#3-uml-class-diagram)
4. [Implementation 1 — Support Ticket Escalation System](#4-implementation-1--support-ticket-escalation-system)
5. [Implementation 2 — HTTP Middleware / Filter Chain](#5-implementation-2--http-middleware--filter-chain)
6. [Breaking the Chain vs Passing Forward](#6-breaking-the-chain-vs-passing-forward)
7. [Functional Chain with Java 8+](#7-functional-chain-with-java-8)
8. [Real-World Framework Examples](#8-real-world-framework-examples)
9. [Trade-offs](#9-trade-offs)
10. [Common Interview Discussion Points](#10-common-interview-discussion-points)
11. [Interview Questions and Model Answers](#11-interview-questions-and-model-answers)
12. [Comparison with Related Patterns](#12-comparison-with-related-patterns)

---

## 1. Intent and Problem Statement

### Intent

Chain of Responsibility lets you pass a request along a **dynamic chain of handlers** where each handler decides either to process the request or to forward it to the next handler in the chain. The sender of a request is **decoupled from the concrete receiver** — it does not know which handler will ultimately handle it, or even whether it will be handled at all.

### The Core Problem

Consider a customer-support system. An incoming ticket might be handled by:

- A Level-1 agent if it is a simple password reset
- A Level-2 engineer if it requires product knowledge
- A Level-3 specialist if it is a data-corruption bug
- Management if it involves an SLA breach worth millions of dollars

Without Chain of Responsibility the calling code must contain a giant `if-else` / `switch` tree that inspects ticket priority and dispatches to the right handler. This violates the **Open/Closed Principle** — every time a new support tier is added the dispatch logic must be changed. It also creates tight coupling between the sender and every possible receiver.

### Forces the Pattern Resolves

| Force | How CoR Resolves It |
|---|---|
| Multiple objects may handle a request; the handler is not known statically | Handler is selected at runtime by walking the chain |
| You want to issue a request without specifying the receiver explicitly | Sender only knows the first handler |
| You need to configure the set of objects that can handle a request dynamically | Assemble the chain at startup or runtime |
| You need to add/remove handlers without changing the sender | New handlers implement the interface and are inserted into the chain |

### Canonical Definition (GoF)

> "Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along the chain until an object handles it."

---

## 2. When to Use / When NOT to Use

### Use When

- **More than one object may handle a request** and the handler is not known a priori.
- **Handlers and their order are configured externally** (e.g., from a configuration file or dependency injection).
- **A request must pass through a series of processing steps**, each of which may or may not act on it (middleware, filters, interceptors).
- **You need to enforce a priority/escalation hierarchy** — lower-priority handlers get first crack, escalating upward only when they cannot satisfy the request.
- **You want to open the system for extension** (add new handlers) **without modifying existing code** (OCP).
- **Processing order matters** and may change independently of business logic.

### Do NOT Use When

- **Exactly one handler will always handle the request** — use Strategy or simple polymorphism instead.
- **Guaranteed handling is required** — a pure chain may drop a request if no handler claims it; ensure you have a terminal catch-all handler or check for unhandled requests explicitly.
- **The chain is very long and performance is critical** — traversal overhead grows linearly with chain length. Consider caching or a dispatch map.
- **Debugging is already difficult** — a chain obscures which handler fired and why. Add extensive logging when the chain is non-trivial.
- **Handlers share a lot of state** — if handlers need to coordinate state, Chain of Responsibility is the wrong tool; use a Command queue or workflow engine.
- **You need parallel handling** — the pattern is inherently sequential. For fan-out use the Observer or Mediator patterns.

---

## 3. UML Class Diagram

```
          Client
            |
            | uses
            v
    +------------------+
    |   <<interface>>  |
    |     Handler      |
    +------------------+
    | +setNext(h)      |
    | +handle(request) |
    +------------------+
             ^
             |  extends
    +------------------+
    | AbstractHandler  |          (optional convenience base)
    +------------------+
    | - next: Handler  |
    +------------------+
    | +setNext(h)      |   <<returns this for fluent chaining>>
    | +handle(request) |   <<calls next.handle() if not null>>
    +------------------+
      ^        ^        ^
      |        |        |
 +--------+ +--------+ +--------+
 |Handler1| |Handler2| |Handler3|
 +--------+ +--------+ +--------+
 |handle()| |handle()| |handle()|
 +--------+ +--------+ +--------+

  Assembled chain at runtime:
  client → Handler1 → Handler2 → Handler3 → null
```

**Key relationships:**

- `Handler` interface declares `handle(request)` and `setNext(handler)`.
- `AbstractHandler` holds a reference to the **next** handler and provides a default forwarding implementation.
- Concrete handlers extend `AbstractHandler`, handle what they can, and call `super.handle(request)` (or `next.handle(request)`) for the rest.
- The **Client** builds the chain and sends the request to the first handler only.

---

## 4. Implementation 1 — Support Ticket Escalation System

This example models a realistic enterprise support system with four tiers. Each tier can resolve tickets up to its authority level; anything beyond is escalated automatically.

### Domain Model

```java
// SupportTicket.java
package com.course.cor.support;

import java.time.Instant;
import java.util.UUID;

/**
 * Represents a customer-support ticket with a priority level.
 * Priority drives which handler in the chain claims it.
 */
public class SupportTicket {

    public enum Priority {
        LOW(1),       // password resets, general FAQ
        MEDIUM(2),    // product configuration, billing queries
        HIGH(3),      // service outages, data loss
        CRITICAL(4);  // SLA breach, legal exposure

        private final int level;

        Priority(int level) { this.level = level; }

        public int getLevel() { return level; }
    }

    private final String id;
    private final String customerName;
    private final String description;
    private final Priority priority;
    private final Instant createdAt;

    public SupportTicket(String customerName, String description, Priority priority) {
        this.id           = UUID.randomUUID().toString().substring(0, 8).toUpperCase();
        this.customerName = customerName;
        this.description  = description;
        this.priority     = priority;
        this.createdAt    = Instant.now();
    }

    public String getId()           { return id; }
    public String getCustomerName() { return customerName; }
    public String getDescription()  { return description; }
    public Priority getPriority()   { return priority; }
    public Instant getCreatedAt()   { return createdAt; }

    @Override
    public String toString() {
        return String.format("[Ticket #%s | %s | %s | \"%s\"]",
                id, customerName, priority, description);
    }
}
```

```java
// SupportHandler.java
package com.course.cor.support;

/**
 * Handler interface for the support escalation chain.
 * Uses fluent setNext() so chain assembly reads naturally:
 *   level1.setNext(level2).setNext(level3).setNext(management)
 */
public interface SupportHandler {

    /**
     * Set the next handler in the chain.
     * Returns the next handler so calls can be chained fluently.
     */
    SupportHandler setNext(SupportHandler next);

    /**
     * Process or escalate the ticket.
     */
    void handle(SupportTicket ticket);
}
```

```java
// AbstractSupportHandler.java
package com.course.cor.support;

/**
 * Convenience base class.  Concrete handlers extend this and only
 * need to override handle() — forwarding is provided here.
 */
public abstract class AbstractSupportHandler implements SupportHandler {

    private SupportHandler next;

    @Override
    public SupportHandler setNext(SupportHandler next) {
        this.next = next;
        return next;                // enables fluent chaining
    }

    /**
     * Subclasses call this to forward to the next handler.
     * If there is no next handler the ticket is silently dropped
     * (callers should ensure a terminal catch-all exists).
     */
    protected void escalate(SupportTicket ticket) {
        if (next != null) {
            next.handle(ticket);
        } else {
            System.out.printf("[CHAIN] No further handler found for %s — ticket unhandled!%n",
                    ticket);
        }
    }

    /** Convenience accessor for subclasses that need to inspect the chain. */
    protected boolean hasNext() {
        return next != null;
    }
}
```

```java
// Level1SupportHandler.java
package com.course.cor.support;

/**
 * First-line support — handles LOW priority tickets only.
 * Anything higher is escalated to Level 2.
 */
public class Level1SupportHandler extends AbstractSupportHandler {

    private static final String TIER = "Level-1 (Front-line)";

    @Override
    public void handle(SupportTicket ticket) {
        if (ticket.getPriority() == SupportTicket.Priority.LOW) {
            System.out.printf("[%s] Handling %s%n", TIER, ticket);
            System.out.printf("[%s]   -> Resolution: Sent automated FAQ response to %s%n%n",
                    TIER, ticket.getCustomerName());
        } else {
            System.out.printf("[%s] Cannot handle %s (priority too high) — escalating...%n%n",
                    TIER, ticket);
            escalate(ticket);
        }
    }
}
```

```java
// Level2SupportHandler.java
package com.course.cor.support;

/**
 * Second-tier support — engineers with product knowledge.
 * Handles LOW and MEDIUM priority tickets.
 */
public class Level2SupportHandler extends AbstractSupportHandler {

    private static final String TIER = "Level-2 (Technical Support)";

    @Override
    public void handle(SupportTicket ticket) {
        int level = ticket.getPriority().getLevel();
        if (level <= SupportTicket.Priority.MEDIUM.getLevel()) {
            System.out.printf("[%s] Handling %s%n", TIER, ticket);
            System.out.printf("[%s]   -> Resolution: Opened remote session with %s; issue resolved%n%n",
                    TIER, ticket.getCustomerName());
        } else {
            System.out.printf("[%s] Escalating %s beyond technical support...%n%n",
                    TIER, ticket);
            escalate(ticket);
        }
    }
}
```

```java
// Level3SupportHandler.java
package com.course.cor.support;

/**
 * Third-tier support — senior engineers and subject-matter experts.
 * Handles LOW, MEDIUM, and HIGH priority tickets.
 */
public class Level3SupportHandler extends AbstractSupportHandler {

    private static final String TIER = "Level-3 (Senior Engineering)";

    @Override
    public void handle(SupportTicket ticket) {
        int level = ticket.getPriority().getLevel();
        if (level <= SupportTicket.Priority.HIGH.getLevel()) {
            System.out.printf("[%s] Handling %s%n", TIER, ticket);
            System.out.printf("[%s]   -> Resolution: Root-cause analysis complete; "
                    + "hotfix deployed for %s%n%n", TIER, ticket.getCustomerName());
        } else {
            System.out.printf("[%s] CRITICAL ticket — escalating to Management...%n%n", TIER);
            escalate(ticket);
        }
    }
}
```

```java
// ManagementSupportHandler.java
package com.course.cor.support;

/**
 * Terminal handler — Management handles all remaining tickets,
 * especially CRITICAL ones involving SLA / legal exposure.
 * This handler never escalates further.
 */
public class ManagementSupportHandler extends AbstractSupportHandler {

    private static final String TIER = "Management (Executive Escalation)";

    @Override
    public void handle(SupportTicket ticket) {
        System.out.printf("[%s] Taking ownership of %s%n", TIER, ticket);
        System.out.printf("[%s]   -> Resolution: Account Manager called; "
                + "compensation credit issued; engineering post-mortem scheduled for %s%n%n",
                TIER, ticket.getCustomerName());
    }
}
```

```java
// SupportChainDemo.java
package com.course.cor.support;

/**
 * Assembles the support chain and demonstrates all priority levels.
 *
 * Chain assembly (fluent style):
 *   level1 -> level2 -> level3 -> management
 */
public class SupportChainDemo {

    public static void main(String[] args) {

        // ---- Build the chain ----
        SupportHandler level1     = new Level1SupportHandler();
        SupportHandler level2     = new Level2SupportHandler();
        SupportHandler level3     = new Level3SupportHandler();
        SupportHandler management = new ManagementSupportHandler();

        // Fluent chain assembly: each setNext() returns the argument,
        // so we can chain calls left-to-right.
        level1.setNext(level2).setNext(level3).setNext(management);

        // ---- Submit tickets ----
        SupportTicket[] tickets = {
            new SupportTicket("Alice",   "Cannot log into my account",          SupportTicket.Priority.LOW),
            new SupportTicket("Bob",     "Integration API returning 500 errors", SupportTicket.Priority.MEDIUM),
            new SupportTicket("Carol",   "Database corruption on production",    SupportTicket.Priority.HIGH),
            new SupportTicket("Acme Inc","Complete service outage — SLA breach", SupportTicket.Priority.CRITICAL),
        };

        System.out.println("=== Support Ticket Escalation Demo ===\n");

        for (SupportTicket ticket : tickets) {
            System.out.println("--- Submitting: " + ticket + " ---");
            level1.handle(ticket);   // always enter at the head of the chain
        }
    }
}
```

**Expected Output:**

```
=== Support Ticket Escalation Demo ===

--- Submitting: [Ticket #A1B2C3D4 | Alice | LOW | "Cannot log into my account"] ---
[Level-1 (Front-line)] Handling [Ticket #A1B2C3D4 | Alice | LOW | "Cannot log into my account"]
[Level-1 (Front-line)]   -> Resolution: Sent automated FAQ response to Alice

--- Submitting: [Ticket #... | Bob | MEDIUM | "Integration API returning 500 errors"] ---
[Level-1 (Front-line)] Cannot handle ... (priority too high) — escalating...
[Level-2 (Technical Support)] Handling ...
[Level-2 (Technical Support)]   -> Resolution: Opened remote session with Bob; issue resolved

--- Submitting: ... HIGH ...
[Level-1 ...] Cannot handle ... escalating...
[Level-2 ...] Escalating ... beyond technical support...
[Level-3 (Senior Engineering)] Handling ...
[Level-3 ...] Root-cause analysis complete; hotfix deployed for Carol

--- Submitting: ... CRITICAL ...
[Level-1 ...] escalating...
[Level-2 ...] Escalating...
[Level-3 ...] CRITICAL ticket — escalating to Management...
[Management (Executive Escalation)] Taking ownership of ...
[Management ...] Resolution: Account Manager called; compensation credit issued; ...
```

---

## 5. Implementation 2 — HTTP Middleware / Filter Chain

HTTP middleware is the canonical real-world application of Chain of Responsibility. Each filter processes part of the request (authentication, rate-limiting, logging, validation, caching) and then passes control to the next filter or to the actual route handler.

### Design Notes

- Unlike the support example where only **one** handler claims the request, HTTP filters **all** process the request sequentially — none "breaks" the chain unless something fails (e.g., auth failure returns 401 immediately).
- This models the **pure pipeline** variant of CoR, as opposed to the **exclusive-claim** variant above.

```java
// HttpRequest.java
package com.course.cor.http;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

/** Simplified HTTP request model. */
public class HttpRequest {

    private final String method;
    private final String path;
    private final Map<String, String> headers;
    private final String body;

    public HttpRequest(String method, String path,
                       Map<String, String> headers, String body) {
        this.method  = method;
        this.path    = path;
        this.headers = Collections.unmodifiableMap(new HashMap<>(headers));
        this.body    = body;
    }

    public String getMethod()  { return method; }
    public String getPath()    { return path; }
    public String getBody()    { return body; }

    public String getHeader(String name) {
        return headers.getOrDefault(name, null);
    }

    public boolean hasHeader(String name) {
        return headers.containsKey(name);
    }

    @Override
    public String toString() {
        return method + " " + path;
    }
}
```

```java
// HttpResponse.java
package com.course.cor.http;

/** Simplified HTTP response model. */
public class HttpResponse {

    private int statusCode;
    private String body;
    private boolean committed;

    public HttpResponse() {
        this.statusCode = 200;
        this.body       = "";
        this.committed  = false;
    }

    public void setStatus(int code)  { this.statusCode = code; }
    public void setBody(String body) { this.body = body; }

    /**
     * Commit the response — once committed, no further processing occurs.
     * This is how a filter "breaks" the chain (e.g., on auth failure).
     */
    public void commit() { this.committed = true; }

    public boolean isCommitted() { return committed; }
    public int getStatusCode()   { return statusCode; }
    public String getBody()      { return body; }

    @Override
    public String toString() {
        return "HTTP " + statusCode + " | " + body;
    }
}
```

```java
// HttpFilter.java
package com.course.cor.http;

/**
 * Filter (Handler) interface.
 * doFilter() must call chain.doFilter() to pass control forward.
 * If it does NOT call chain.doFilter() the chain is broken.
 */
public interface HttpFilter {

    /** Human-readable name for logging. */
    String name();

    /**
     * Process the request/response.
     * @param request  incoming HTTP request
     * @param response mutable HTTP response
     * @param chain    the remaining filter chain
     */
    void doFilter(HttpRequest request, HttpResponse response, FilterChain chain);
}
```

```java
// FilterChain.java
package com.course.cor.http;

import java.util.ArrayList;
import java.util.List;

/**
 * Holds the ordered list of filters and tracks the current position.
 * This is the "chain" object passed to each filter's doFilter().
 *
 * Pattern note: FilterChain is a separate object (not embedded in Handler)
 * because the same chain may be traversed by multiple concurrent requests.
 * Creating a new FilterChain per request provides thread safety.
 */
public class FilterChain {

    private final List<HttpFilter> filters;
    private int index = 0;

    private FilterChain(List<HttpFilter> filters) {
        this.filters = List.copyOf(filters);
    }

    /**
     * Factory: create a chain from an ordered list of filters.
     */
    public static FilterChain of(List<HttpFilter> filters) {
        return new FilterChain(filters);
    }

    /**
     * Advance the chain to the next filter, or invoke the terminal
     * "route handler" if all filters have been executed.
     */
    public void doFilter(HttpRequest request, HttpResponse response) {
        if (response.isCommitted()) {
            return;  // a filter already wrote the response; stop processing
        }
        if (index < filters.size()) {
            HttpFilter current = filters.get(index++);
            System.out.printf("  [%s] -> processing %s%n", current.name(), request);
            current.doFilter(request, response, this);
        } else {
            // Terminal: simulate actual route handler
            routeHandler(request, response);
        }
    }

    private void routeHandler(HttpRequest request, HttpResponse response) {
        System.out.printf("  [RouteHandler] handling %s%n", request);
        response.setStatus(200);
        response.setBody("{\"result\": \"Hello from route handler!\"}");
    }
}
```

```java
// AuthenticationFilter.java
package com.course.cor.http;

/**
 * Validates the Authorization header.
 * BREAKS the chain with 401 if no valid token is present.
 */
public class AuthenticationFilter implements HttpFilter {

    private static final String VALID_TOKEN = "Bearer secret-token-123";

    @Override
    public String name() { return "AuthenticationFilter"; }

    @Override
    public void doFilter(HttpRequest request, HttpResponse response, FilterChain chain) {
        String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.equals(VALID_TOKEN)) {
            System.out.println("  [AuthenticationFilter] REJECTED — invalid or missing token");
            response.setStatus(401);
            response.setBody("{\"error\": \"Unauthorized\"}");
            response.commit();   // do NOT forward — chain broken
            return;
        }
        System.out.println("  [AuthenticationFilter] PASSED — token valid");
        chain.doFilter(request, response);  // forward to next filter
    }
}
```

```java
// RateLimitFilter.java
package com.course.cor.http;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Simple in-memory rate limiter (max 3 requests per client IP per window).
 * BREAKS the chain with 429 if the limit is exceeded.
 *
 * Production note: a real implementation would use a sliding window and
 * a distributed store (Redis); this illustrates the CoR hook.
 */
public class RateLimitFilter implements HttpFilter {

    private static final int MAX_REQUESTS = 3;
    private final ConcurrentHashMap<String, AtomicInteger> counters = new ConcurrentHashMap<>();

    @Override
    public String name() { return "RateLimitFilter"; }

    @Override
    public void doFilter(HttpRequest request, HttpResponse response, FilterChain chain) {
        String clientIp = request.getHeader("X-Client-IP");
        if (clientIp == null) clientIp = "unknown";

        AtomicInteger count = counters.computeIfAbsent(clientIp, k -> new AtomicInteger(0));
        int current = count.incrementAndGet();

        if (current > MAX_REQUESTS) {
            System.out.printf("  [RateLimitFilter] REJECTED — IP %s exceeded limit (%d/%d)%n",
                    clientIp, current, MAX_REQUESTS);
            response.setStatus(429);
            response.setBody("{\"error\": \"Too Many Requests\"}");
            response.commit();
            return;
        }
        System.out.printf("  [RateLimitFilter] PASSED — IP %s request %d/%d%n",
                clientIp, current, MAX_REQUESTS);
        chain.doFilter(request, response);
    }
}
```

```java
// LoggingFilter.java
package com.course.cor.http;

import java.time.Instant;

/**
 * Logs request entry and response exit times.
 * ALWAYS passes forward — illustrates a pure-pipeline handler.
 */
public class LoggingFilter implements HttpFilter {

    @Override
    public String name() { return "LoggingFilter"; }

    @Override
    public void doFilter(HttpRequest request, HttpResponse response, FilterChain chain) {
        long start = System.currentTimeMillis();
        System.out.printf("  [LoggingFilter] START  %s at %s%n", request, Instant.now());

        chain.doFilter(request, response);   // always forward

        long elapsed = System.currentTimeMillis() - start;
        System.out.printf("  [LoggingFilter] END    %s -> HTTP %d (%dms)%n",
                request, response.getStatusCode(), elapsed);
    }
}
```

```java
// ValidationFilter.java
package com.course.cor.http;

/**
 * Validates that POST/PUT requests supply a Content-Type header
 * and a non-empty body.
 * BREAKS the chain with 400 on validation failure.
 */
public class ValidationFilter implements HttpFilter {

    @Override
    public String name() { return "ValidationFilter"; }

    @Override
    public void doFilter(HttpRequest request, HttpResponse response, FilterChain chain) {
        String method = request.getMethod();
        if ("POST".equalsIgnoreCase(method) || "PUT".equalsIgnoreCase(method)) {
            String contentType = request.getHeader("Content-Type");
            if (contentType == null || !contentType.contains("application/json")) {
                System.out.println("  [ValidationFilter] REJECTED — missing Content-Type: application/json");
                response.setStatus(400);
                response.setBody("{\"error\": \"Content-Type must be application/json\"}");
                response.commit();
                return;
            }
            if (request.getBody() == null || request.getBody().isBlank()) {
                System.out.println("  [ValidationFilter] REJECTED — empty request body");
                response.setStatus(400);
                response.setBody("{\"error\": \"Request body must not be empty\"}");
                response.commit();
                return;
            }
        }
        System.out.println("  [ValidationFilter] PASSED");
        chain.doFilter(request, response);
    }
}
```

```java
// CachingFilter.java
package com.course.cor.http;

import java.util.HashMap;
import java.util.Map;

/**
 * Simple response cache for GET requests.
 * If a cached response exists the chain is SHORT-CIRCUITED
 * (no further filters or the route handler execute).
 * On a cache miss the request passes forward, and the response
 * is cached before returning to the caller.
 */
public class CachingFilter implements HttpFilter {

    private final Map<String, String> cache = new HashMap<>();

    @Override
    public String name() { return "CachingFilter"; }

    @Override
    public void doFilter(HttpRequest request, HttpResponse response, FilterChain chain) {
        if (!"GET".equalsIgnoreCase(request.getMethod())) {
            chain.doFilter(request, response);
            return;
        }

        String cacheKey = request.getPath();
        if (cache.containsKey(cacheKey)) {
            System.out.printf("  [CachingFilter] CACHE HIT for %s — skipping downstream%n", cacheKey);
            response.setStatus(200);
            response.setBody(cache.get(cacheKey));
            response.commit();
            return;
        }

        System.out.printf("  [CachingFilter] CACHE MISS for %s — forwarding%n", cacheKey);
        chain.doFilter(request, response);

        // Store successful responses in cache
        if (response.getStatusCode() == 200) {
            cache.put(cacheKey, response.getBody());
            System.out.printf("  [CachingFilter] Cached response for %s%n", cacheKey);
        }
    }
}
```

```java
// HttpFilterChainDemo.java
package com.course.cor.http;

import java.util.List;
import java.util.Map;

/**
 * Assembles and exercises the HTTP filter chain.
 *
 * Filter order:
 *   LoggingFilter -> AuthenticationFilter -> RateLimitFilter
 *       -> ValidationFilter -> CachingFilter -> RouteHandler
 *
 * Logging wraps everything so latency is measured end-to-end.
 */
public class HttpFilterChainDemo {

    private static final List<HttpFilter> FILTERS = List.of(
            new LoggingFilter(),
            new AuthenticationFilter(),
            new RateLimitFilter(),
            new ValidationFilter(),
            new CachingFilter()
    );

    public static void main(String[] args) {

        System.out.println("=== HTTP Filter Chain Demo ===\n");

        // Scenario 1: Missing auth header
        System.out.println("--- Scenario 1: No Authorization header ---");
        sendRequest("GET", "/api/users",
                Map.of("X-Client-IP", "10.0.0.1"),
                null);

        // Scenario 2: Valid GET request (cache miss)
        System.out.println("--- Scenario 2: Valid GET (cache miss) ---");
        sendRequest("GET", "/api/users",
                Map.of("Authorization", "Bearer secret-token-123",
                       "X-Client-IP",   "10.0.0.2"),
                null);

        // Scenario 3: Same GET (cache hit — CachingFilter short-circuits)
        System.out.println("--- Scenario 3: Valid GET (cache hit) ---");
        sendRequest("GET", "/api/users",
                Map.of("Authorization", "Bearer secret-token-123",
                       "X-Client-IP",   "10.0.0.3"),
                null);

        // Scenario 4: POST with missing Content-Type
        System.out.println("--- Scenario 4: POST — missing Content-Type ---");
        sendRequest("POST", "/api/users",
                Map.of("Authorization", "Bearer secret-token-123",
                       "X-Client-IP",   "10.0.0.4"),
                "{\"name\":\"Alice\"}");

        // Scenario 5: Valid POST
        System.out.println("--- Scenario 5: Valid POST ---");
        sendRequest("POST", "/api/users",
                Map.of("Authorization",  "Bearer secret-token-123",
                       "X-Client-IP",    "10.0.0.5",
                       "Content-Type",   "application/json"),
                "{\"name\":\"Bob\"}");
    }

    private static void sendRequest(String method, String path,
                                    Map<String, String> headers, String body) {
        HttpRequest  request  = new HttpRequest(method, path, headers, body);
        HttpResponse response = new HttpResponse();
        FilterChain  chain    = FilterChain.of(FILTERS);
        chain.doFilter(request, response);
        System.out.println("  [Result] " + response + "\n");
    }
}
```

---

## 6. Breaking the Chain vs Passing Forward

Understanding when a handler should **stop** the chain versus **continue** it is the most important design decision in CoR.

### Style A — Exclusive Claim (handler breaks the chain by NOT calling next)

```java
// HandlerA.java  — EXCLUSIVE claim style
package com.course.cor.styles;

public class HandlerA extends AbstractHandler<Integer, String> {

    @Override
    public String handle(Integer request) {
        if (request < 10) {
            // We own this request — process it and STOP.
            return "HandlerA handled: " + request;
        }
        // We cannot handle it — pass forward.
        return passForward(request);
    }
}
```

```java
// AbstractHandler.java
package com.course.cor.styles;

public abstract class AbstractHandler<REQ, RESP> {

    private AbstractHandler<REQ, RESP> next;

    public AbstractHandler<REQ, RESP> setNext(AbstractHandler<REQ, RESP> next) {
        this.next = next;
        return next;
    }

    protected RESP passForward(REQ request) {
        if (next != null) {
            return next.handle(request);
        }
        return null;  // unhandled — callers should null-check
    }

    public abstract RESP handle(REQ request);
}
```

```java
// HandlerB.java
package com.course.cor.styles;

public class HandlerB extends AbstractHandler<Integer, String> {

    @Override
    public String handle(Integer request) {
        if (request < 100) {
            return "HandlerB handled: " + request;
        }
        return passForward(request);
    }
}
```

```java
// CatchAllHandler.java  — terminal handler; always claims the request
package com.course.cor.styles;

public class CatchAllHandler extends AbstractHandler<Integer, String> {

    @Override
    public String handle(Integer request) {
        return "CatchAllHandler handled: " + request + " (fallback)";
    }
}
```

```java
// ExclusiveClaimDemo.java
package com.course.cor.styles;

public class ExclusiveClaimDemo {
    public static void main(String[] args) {
        HandlerA a = new HandlerA();
        HandlerB b = new HandlerB();
        CatchAllHandler catchAll = new CatchAllHandler();

        a.setNext(b).setNext(catchAll);

        System.out.println(a.handle(5));    // HandlerA handled: 5
        System.out.println(a.handle(50));   // HandlerB handled: 50
        System.out.println(a.handle(500));  // CatchAllHandler handled: 500 (fallback)
    }
}
```

### Style B — Pure Pipeline (all handlers execute; some may short-circuit on error)

```java
// PipelineHandler.java
package com.course.cor.styles;

/**
 * In the pipeline style, the "chain" is explicit and passed to each handler.
 * Every handler runs unless one deliberately stops propagation.
 */
@FunctionalInterface
public interface PipelineHandler<CTX> {
    /**
     * @param context  shared mutable context object
     * @param next     call next.handle(context) to continue; omit to stop
     */
    void handle(CTX context, Runnable next);
}
```

```java
// PipelineContext.java
package com.course.cor.styles;

import java.util.ArrayList;
import java.util.List;

/** Accumulated context that all pipeline stages can read/write. */
public class PipelineContext {
    private final List<String> log = new ArrayList<>();
    private boolean aborted = false;

    public void addLog(String msg)   { log.add(msg); }
    public void abort()              { aborted = true; }
    public boolean isAborted()       { return aborted; }
    public List<String> getLog()     { return List.copyOf(log); }
}
```

```java
// Pipeline.java
package com.course.cor.styles;

import java.util.ArrayList;
import java.util.List;

/**
 * Executes a list of PipelineHandlers in order.
 * Any handler can stop propagation by NOT calling next.run().
 */
public class Pipeline<CTX> {

    private final List<PipelineHandler<CTX>> handlers = new ArrayList<>();

    public Pipeline<CTX> addHandler(PipelineHandler<CTX> handler) {
        handlers.add(handler);
        return this;
    }

    public void execute(CTX context) {
        run(context, 0);
    }

    private void run(CTX context, int index) {
        if (index >= handlers.size()) return;
        PipelineHandler<CTX> current = handlers.get(index);
        current.handle(context, () -> run(context, index + 1));
    }
}
```

```java
// PipelineDemo.java
package com.course.cor.styles;

public class PipelineDemo {

    public static void main(String[] args) {

        // Handler that always passes forward
        PipelineHandler<PipelineContext> step1 = (ctx, next) -> {
            ctx.addLog("Step1: preprocessing");
            next.run();  // always continue
            ctx.addLog("Step1: postprocessing");
        };

        // Handler that conditionally aborts
        PipelineHandler<PipelineContext> step2 = (ctx, next) -> {
            ctx.addLog("Step2: validation");
            boolean valid = true; // change to false to see abort behaviour
            if (!valid) {
                ctx.abort();
                ctx.addLog("Step2: ABORTED — did not call next");
                return;
            }
            next.run();
        };

        PipelineHandler<PipelineContext> step3 = (ctx, next) -> {
            ctx.addLog("Step3: business logic");
            next.run();
        };

        Pipeline<PipelineContext> pipeline = new Pipeline<PipelineContext>()
                .addHandler(step1)
                .addHandler(step2)
                .addHandler(step3);

        PipelineContext ctx = new PipelineContext();
        pipeline.execute(ctx);
        System.out.println("Execution log: " + ctx.getLog());
        // Output: [Step1: preprocessing, Step2: validation,
        //          Step3: business logic, Step1: postprocessing]
    }
}
```

---

## 7. Functional Chain with Java 8+

Java 8 lambdas and `Function` composition let you build a chain without defining handler classes, which is idiomatic in modern Java.

```java
// FunctionalChain.java
package com.course.cor.functional;

import java.util.function.Function;
import java.util.function.Predicate;
import java.util.function.UnaryOperator;

/**
 * Demonstrates three Java 8+ approaches to Chain of Responsibility.
 */
public class FunctionalChain {

    // ---------------------------------------------------------------
    // Approach 1: Function.andThen() — pure pipeline
    // Each function transforms the input; all run sequentially.
    // ---------------------------------------------------------------

    static String processWithAndThen(String input) {
        Function<String, String> trim        = String::trim;
        Function<String, String> toLowerCase = String::toLowerCase;
        Function<String, String> addPrefix   = s -> "processed:" + s;

        Function<String, String> pipeline = trim
                .andThen(toLowerCase)
                .andThen(addPrefix);

        return pipeline.apply(input);
    }

    // ---------------------------------------------------------------
    // Approach 2: Recursive lambda — exclusive-claim style
    // Build a single Function that encodes handler logic inline.
    // ---------------------------------------------------------------

    @FunctionalInterface
    interface Handler<T> {
        /** @return result, or null if this handler cannot process the input */
        T handle(T input);
    }

    static <T> Handler<T> chain(Handler<T>... handlers) {
        return input -> {
            for (Handler<T> h : handlers) {
                T result = h.handle(input);
                if (result != null) return result;
            }
            return null;  // unhandled
        };
    }

    // ---------------------------------------------------------------
    // Approach 3: Builder DSL with Predicate guard
    // Reads most naturally for business-rule chains.
    // ---------------------------------------------------------------

    static class ChainBuilder<REQ, RESP> {

        private Function<REQ, RESP> composed = req -> null;  // default: unhandled

        /**
         * Add a handler: if predicate matches, apply function; otherwise skip.
         */
        public ChainBuilder<REQ, RESP> when(Predicate<REQ> predicate,
                                            Function<REQ, RESP> handler) {
            Function<REQ, RESP> previous = this.composed;
            this.composed = req -> {
                RESP existing = previous.apply(req);
                if (existing != null) return existing;   // already handled
                if (predicate.test(req)) return handler.apply(req);
                return null;
            };
            return this;
        }

        public Function<REQ, RESP> build() {
            return composed;
        }
    }

    // ---------------------------------------------------------------
    // Demo
    // ---------------------------------------------------------------

    public static void main(String[] args) {

        // Approach 1
        System.out.println("Approach 1: " + processWithAndThen("  HELLO WORLD  "));
        // Output: processed:hello world

        // Approach 2
        @SuppressWarnings("unchecked")
        Handler<Integer> discountChain = chain(
            n -> (n > 1000) ? (int)(n * 0.70) : null,   // 30% for large orders
            n -> (n > 500)  ? (int)(n * 0.85) : null,   // 15% for medium orders
            n -> (int)(n * 0.95)                          // 5% default
        );
        System.out.println("Approach 2 (order=1500): " + discountChain.handle(1500)); // 1050
        System.out.println("Approach 2 (order=600):  " + discountChain.handle(600));  // 510
        System.out.println("Approach 2 (order=100):  " + discountChain.handle(100));  // 95

        // Approach 3
        Function<String, String> router = new ChainBuilder<String, String>()
                .when(s -> s.startsWith("/admin"),  s -> "AdminController.handle(" + s + ")")
                .when(s -> s.startsWith("/api"),    s -> "ApiController.handle(" + s + ")")
                .when(s -> s.startsWith("/static"), s -> "StaticFileServer.serve(" + s + ")")
                .when(s -> true,                    s -> "NotFoundController.handle(" + s + ")")
                .build();

        System.out.println("Approach 3: " + router.apply("/api/users"));
        System.out.println("Approach 3: " + router.apply("/admin/config"));
        System.out.println("Approach 3: " + router.apply("/unknown"));
    }
}
```

---

## 8. Real-World Framework Examples

### 8.1 Java Servlet Filters (javax.servlet / jakarta.servlet)

The Servlet specification is the quintessential CoR implementation in the Java ecosystem.

```java
// ExampleServletFilter.java  — mirrors real javax.servlet.Filter
package com.course.cor.realworld;

/**
 * This is the ACTUAL javax.servlet.Filter interface signature.
 * Included here for study; in a real project add the servlet-api dependency.
 *
 * The FilterChain passed to doFilter() is exactly the CoR "next" reference.
 */
/*
public interface Filter {
    void init(FilterConfig filterConfig) throws ServletException;
    void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException;
    void destroy();
}
*/

/**
 * A real CORS filter implemented using javax.servlet.Filter.
 * Demonstrates how the pattern is used in production servlet containers.
 */
public class CorsFilter /* implements javax.servlet.Filter */ {

    // In a real application annotate with @WebFilter("/*") or register in web.xml

    public void doFilter(Object request, Object response, Object chain) throws Exception {
        // Cast to HttpServletRequest/HttpServletResponse in real code
        // httpResponse.setHeader("Access-Control-Allow-Origin", "*");
        // httpResponse.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE");
        // chain.doFilter(request, response);  // MUST call to continue chain
        System.out.println("[CorsFilter] Adding CORS headers");
        System.out.println("[CorsFilter] Forwarding to next filter");
        // chain.doFilter(request, response);
    }
}
```

```
Servlet container assembles the chain from web.xml / @WebFilter declarations:
  Request
    |
    v
  Filter1 (CORS)
    |
    v
  Filter2 (Authentication)
    |
    v
  Filter3 (Logging)
    |
    v
  Servlet (DispatcherServlet / route handler)
    |
    v (response travels back UP the chain)
  Filter3 (Logging — post-processing)
    |
    v
  Filter2 (Authentication — no post-processing)
    |
    v
  Filter1 (CORS — no post-processing)
    |
    v
  Client
```

### 8.2 Spring Security Filter Chain

Spring Security implements a `FilterChainProxy` that contains a list of `SecurityFilterChain` objects. Each chain is a CoR chain of `Filter` implementations:

```
SecurityContextPersistenceFilter
  -> UsernamePasswordAuthenticationFilter
  -> BasicAuthenticationFilter
  -> ExceptionTranslationFilter
  -> FilterSecurityInterceptor (authorization)
```

Key class: `org.springframework.security.web.FilterChainProxy`

```java
// SpringSecurityChainSketch.java
package com.course.cor.realworld;

/**
 * Conceptual sketch showing how Spring Security configures its filter chain
 * using the modern SecurityFilterChain DSL (Spring Security 5.7+).
 *
 * The actual filters are CoR handlers; Spring assembles the chain from the
 * HttpSecurity builder and delegates each request through them in order.
 */
public class SpringSecurityChainSketch {

    /*
    @Configuration
    @EnableWebSecurity
    public class SecurityConfig {

        @Bean
        public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
            http
                // Each method configures a filter added to the CoR chain
                .csrf(csrf -> csrf.disable())
                .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
                .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/public/**").permitAll()
                    .anyRequest().authenticated()
                )
                .addFilterBefore(new JwtAuthFilter(), UsernamePasswordAuthenticationFilter.class);

            return http.build();  // Assembles the CoR chain
        }
    }
    */
}
```

### 8.3 Log4j / SLF4J — Level-Based Filtering

Logging frameworks use a CoR-like pattern where each logger checks whether it should handle a log event before passing it to its parent (appender chain):

```java
// LogLevelChainSketch.java
package com.course.cor.realworld;

/**
 * Simplified model of how Log4j's level-based filtering works.
 * Each Logger has a level threshold; if the event level is below the
 * threshold, the event is dropped (chain broken).
 * If an event passes the threshold it propagates to parent loggers
 * (additivity) — a CoR traversal up the logger hierarchy.
 */
public class LogLevelChainSketch {

    enum Level { TRACE(0), DEBUG(1), INFO(2), WARN(3), ERROR(4), FATAL(5);
        final int value;
        Level(int v) { this.value = v; }
    }

    static class Logger {
        private final String name;
        private final Level threshold;
        private Logger parent;  // CoR "next"

        Logger(String name, Level threshold) {
            this.name      = name;
            this.threshold = threshold;
        }

        Logger withParent(Logger parent) { this.parent = parent; return this; }

        void log(Level eventLevel, String message) {
            if (eventLevel.value < threshold.value) {
                return;  // Chain broken — below this logger's threshold
            }
            System.out.printf("[%s/%s] %s%n", name, eventLevel, message);
            if (parent != null) {
                parent.log(eventLevel, message); // Additivity — propagate up
            }
        }
    }

    public static void main(String[] args) {
        Logger root         = new Logger("root",            Level.WARN);
        Logger appLogger    = new Logger("com.myapp",       Level.INFO).withParent(root);
        Logger serviceLog   = new Logger("com.myapp.svc",  Level.DEBUG).withParent(appLogger);

        serviceLog.log(Level.DEBUG, "Entering method");   // DEBUG >= DEBUG: passes serviceLog, stopped at appLogger (INFO threshold)
        serviceLog.log(Level.INFO,  "Request received");  // INFO >= DEBUG: passes serviceLog and appLogger, stopped at root (WARN)
        serviceLog.log(Level.ERROR, "Unhandled exception");// ERROR >= all: propagates all the way to root
    }
}
```

### 8.4 OkHttp Interceptors

OkHttp's `Interceptor` interface is a textbook Chain of Responsibility implementation:

```java
// OkHttpInterceptorSketch.java
package com.course.cor.realworld;

/**
 * OkHttp's Interceptor is identical in structure to our HttpFilter:
 *
 *   interface Interceptor {
 *       Response intercept(Chain chain) throws IOException;
 *   }
 *
 *   interface Chain {
 *       Request request();
 *       Response proceed(Request request) throws IOException;
 *   }
 *
 * Calling chain.proceed(request) is exactly "pass to the next handler."
 * NOT calling it breaks the chain and returns a synthetic response.
 *
 * Example use: authentication + logging + caching interceptors stacked
 * via OkHttpClient.Builder.addInterceptor() / addNetworkInterceptor().
 */
public class OkHttpInterceptorSketch {

    /*
    // Application interceptor (outermost layer)
    Interceptor loggingInterceptor = chain -> {
        Request request = chain.request();
        long start = System.nanoTime();
        Response response = chain.proceed(request);   // forward
        long elapsed = (System.nanoTime() - start) / 1_000_000;
        System.out.printf("%-20s %dms%n", request.url(), elapsed);
        return response;
    };

    OkHttpClient client = new OkHttpClient.Builder()
        .addInterceptor(loggingInterceptor)
        .addInterceptor(authInterceptor)
        .addNetworkInterceptor(cacheInterceptor)
        .build();
    */
}
```

---

## 9. Trade-offs

### Pros

| Benefit | Explanation |
|---|---|
| **Single Responsibility** | Each handler does exactly one thing. Business logic is not tangled with routing/dispatch logic. |
| **Open/Closed Principle** | Add new handlers without touching existing handlers or the sender. The chain is the extension point. |
| **Configurable handling order** | Chain can be assembled at runtime from configuration, dependency injection, or feature flags. |
| **Decouples sender from receivers** | The sender only knows the head of the chain; it never depends on concrete handler types. |
| **Flexible per-request chains** | Create different chains for different client types, tenants, or environments. |
| **Natural escalation modeling** | Maps directly to tiered support, approval workflows, permission hierarchies. |

### Cons

| Drawback | Explanation |
|---|---|
| **No guaranteed handling** | A request may fall off the end of the chain unprocessed. Must have a catch-all or explicit checks. |
| **Debugging difficulty** | Tracing which handler acted on a request requires logging at every step; silent failures are hard to diagnose. |
| **Runtime overhead** | O(n) traversal per request where n is chain length. Rarely a bottleneck, but consider caching for hot paths. |
| **Order sensitivity** | Handler order can be critical and is not enforced by the pattern itself — wrong order causes subtle bugs. |
| **Implicit coupling through shared state** | If handlers read/write shared mutable context, coupling reappears in disguise. |
| **Hard to observe** | Unlike a `switch` statement, the chain's flow is not statically visible; requires good documentation and logging. |

---

## 10. Common Interview Discussion Points

### Point 1 — The "Pure Pipeline" vs "Exclusive Claim" distinction

Most candidates describe only the exclusive-claim variant (one handler wins). Be prepared to discuss both:

- **Exclusive claim** (support escalation): exactly one handler processes the request.
- **Pure pipeline** (HTTP filters, logging): all handlers process the request in sequence; any may short-circuit on error.

Real systems often combine both: filters in a pipeline, but each filter may internally claim or reject the request.

### Point 2 — Chain assembly is separate from handler logic

Chain of Responsibility separates **who builds the chain** (configuration/factory) from **what each handler does** (business logic). This is a key design principle. In Spring Security the chain is built by `HttpSecurity`; the filters themselves know nothing about their siblings.

### Point 3 — The "unhandled request" problem

A well-designed chain always includes a terminal catch-all handler. Failing to do so means requests can be silently dropped — a production defect. The catch-all can:
- Throw an exception (fail fast).
- Return a default response.
- Log and no-op (for optional processing).

### Point 4 — Chain of Responsibility vs Decorator

Both patterns chain objects and each object wraps the next. The difference:
- **Decorator** adds behavior transparently; the client does not know decoration is happening. All decorators execute.
- **CoR** routes to exactly one handler (exclusive-claim) or allows short-circuiting. The sender is aware it is sending to a chain.

### Point 5 — Dynamic chain reconfiguration

In microservices, the filter chain can be reconfigured at runtime using feature flags. For example, a `CanaryFilter` can be toggled into the chain for 5% of traffic, then 50%, then 100%, without deploying new code.

### Point 6 — Thread safety of shared chains

In servlet containers, a single `FilterChain` object is **shared across concurrent requests**. Filters must therefore be stateless or use thread-local/scoped state. This is why `FilterChain.doFilter()` in our implementation creates a new `FilterChain` per request.

---

## 11. Interview Questions and Model Answers

---

### Q1: What is the Chain of Responsibility pattern and what problem does it solve?

**Model Answer:**

Chain of Responsibility is a behavioral design pattern in which a request is passed through a sequence of handler objects. Each handler decides independently whether to process the request or forward it to the next handler in the chain. The sender of the request is completely decoupled from its receivers — it only knows the head of the chain.

The core problem it solves is **dynamic, ordered dispatch with an unknown or changing set of receivers**. Without it, the sender would need a giant `if-else` or `switch` block that couples it to every possible handler and violates Open/Closed. With it, new handlers can be added by inserting them into the chain without touching the sender or any other handler.

Classic examples: servlet filter chains, logging level hierarchies, support ticket escalation, approval workflows, middleware pipelines.

---

### Q2: What are the two variants of Chain of Responsibility and when do you use each?

**Model Answer:**

**Variant 1 — Exclusive Claim:** Exactly one handler processes the request. A handler either processes it and stops, or passes it to the next. Used when a request is conceptually "owned" by one handler — support escalation tiers, approval authority levels, error recovery (try handlers one by one until one succeeds).

**Variant 2 — Pure Pipeline:** All handlers execute sequentially, each adding behavior, and any one may short-circuit the rest on a specific condition (e.g., auth failure). Used when every handler must contribute — HTTP middleware, aspect-oriented cross-cutting concerns (logging + security + caching + routing), request/response transformation pipelines.

In practice, many systems blend both: the outer layer is a pipeline (all filters run), but individual filters may internally apply exclusive-claim logic (auth filter either passes or breaks the chain with 401).

---

### Q3: How does Chain of Responsibility differ from Strategy and from Command?

**Model Answer:**

**vs Strategy:**
- Strategy selects **one algorithm** from a set at configuration time; the algorithm is injected and always used for that object's lifetime.
- CoR also selects a handler, but the selection is **dynamic and runtime** — the chain is traversed to find the right handler. Strategy has one handler; CoR has many candidates.

**vs Command:**
- Command encapsulates **what to execute** as an object (parameterizes requests, supports undo/redo, queuing). The invoker knows the command explicitly and calls it directly.
- CoR decides **which object executes** — the sender does not know who will handle the request. Command is about encapsulating an action; CoR is about routing a request to its handler.

Summary: Strategy = one algorithm selected at config time; Command = encapsulated action + known receiver; CoR = dynamic routing through a chain of potential handlers.

---

### Q4: How do servlet filters implement Chain of Responsibility? Walk through the flow.

**Model Answer:**

In the Servlet specification, `javax.servlet.Filter` is the Handler interface. `FilterChain` is the "next" reference. The container assembles the chain from `web.xml` or `@WebFilter` annotations.

Flow for a single request:

1. Container calls `filter1.doFilter(request, response, chain)`.
2. Filter1 does pre-processing (e.g., adds CORS headers).
3. Filter1 calls `chain.doFilter(request, response)` — this advances the index and calls `filter2.doFilter(...)`.
4. Filter2 does authentication: if token is invalid, it writes 401 to the response and **returns without calling chain.doFilter()**, breaking the chain.
5. If auth passes, Filter2 calls `chain.doFilter(...)` which calls Filter3, and so on until the Servlet's `service()` is reached.
6. On the way back up (after the Servlet returns), control returns to Filter3's code after its `chain.doFilter()` call, then Filter2's, then Filter1's — allowing post-processing (e.g., Filter1 measures total elapsed time).

This two-phase traversal (pre-processing going down, post-processing going up) is unique to the servlet filter chain and is not present in the basic GoF description. It resembles a stack of AOP around-advice.

---

### Q5: What happens if no handler in the chain processes the request? How do you handle this?

**Model Answer:**

In the basic GoF pattern, if no handler claims the request it falls off the end of the chain silently. This is a common production defect — a request disappears with no error and no log.

There are three standard approaches:

1. **Terminal catch-all handler:** Always add a final handler that either provides a default response, throws an appropriate exception (e.g., `UnsupportedOperationException` or HTTP 404/500), or logs a structured error. This is mandatory in most real systems.

2. **Chain-level check after traversal:** After calling `head.handle(request)`, the caller checks a result flag or return value. If null or unhandled, the caller itself takes corrective action.

3. **Assertion in the abstract base class:** The `AbstractHandler.escalate()` method logs a warning and throws an `IllegalStateException` when there is no next handler and the current handler could not process the request.

Best practice: combine approach 1 (always have a catch-all) with approach 3 (fail loudly in development mode). Log every unhandled request in production with full context so the missing handler is easy to diagnose.

---

### Q6: How do you make a Chain of Responsibility thread-safe?

**Model Answer:**

The key insight is to separate **chain structure** (the linked list of handlers) from **per-request state**.

**Handler objects should be stateless:** If handlers are stateless (no instance fields that vary per request), the same handler instances can be shared safely across threads. Authentication filters, logging filters, and validation filters should all be designed stateless.

**Per-request state in the context object:** Any state that varies per request (user identity, request ID, timing data) belongs in the request/context object, not in the handler. Pass the context object into `handle()` / `doFilter()`.

**FilterChain must NOT be shared:** The `FilterChain` object holds the current traversal index. If one `FilterChain` instance is shared across concurrent requests, the index becomes a race condition. Create a **new FilterChain per request** — in our implementation `FilterChain.of(FILTERS)` does this.

**Stateful handlers use scoped containers:** If a handler genuinely needs per-request state (e.g., a transaction), use `ThreadLocal`, Spring's `RequestScope`, or inject a factory that creates request-scoped objects.

**Immutable chain structure:** Assemble the chain at startup and never modify it afterward. Dynamic chain changes require proper synchronization (e.g., `volatile`, `ReadWriteLock`, or rebuilding the chain atomically).

---

### Q7: How would you implement an approval workflow (purchase orders requiring different authority levels) using Chain of Responsibility?

**Model Answer:**

A purchase order approval workflow is a classic CoR use case with an **exclusive-claim** variant.

**Domain model:**
```java
class PurchaseOrder {
    double amount;
    String department;
    String description;
}

interface Approver {
    Approver setNext(Approver next);
    void approve(PurchaseOrder order);
}
```

**Handler hierarchy:**
- `TeamLeadApprover` — approves orders up to $1,000
- `ManagerApprover` — approves orders $1,000–$10,000
- `DirectorApprover` — approves orders $10,000–$100,000
- `CFOApprover` — approves all remaining orders (terminal)

**Chain assembly in Spring:**
```java
@Bean
Approver approvalChain(TeamLeadApprover tl, ManagerApprover mgr,
                        DirectorApprover dir, CFOApprover cfo) {
    tl.setNext(mgr).setNext(dir).setNext(cfo);
    return tl;
}
```

**Extensions to discuss in the interview:**
- **Parallel approval** for amounts requiring two signatories — use Observer or fork-join alongside CoR.
- **Timeout escalation** — if the handler does not respond within SLA, auto-escalate to the next tier.
- **Audit trail** — each handler appends to an `ApprovalHistory` list before passing forward.
- **Dynamic chain** — authority levels stored in a database and the chain is rebuilt on configuration changes, without code deployment.

---

### Q8: Compare Chain of Responsibility with Decorator. When would you choose one over the other?

**Model Answer:**

Both patterns link objects in a chain where each object has a reference to the next. The structural similarity is intentional — they are complementary tools.

**Decorator:**
- Adds behavior **transparently** to a component without the client knowing.
- **All decorators always execute** — there is no "break" mechanism.
- Decorators wrap a **core component** (leaf) which provides the primary behavior.
- The client sees a uniform interface; it cannot distinguish the decorated object from the undecorated one.
- Use case: `BufferedInputStream(new FileInputStream(...))` — buffering is transparent.

**Chain of Responsibility:**
- **Routes** a request to the appropriate handler — the handler selection is the point, not decoration.
- Handlers can **break the chain** — zero or one handler claims the request (exclusive-claim) or any handler can short-circuit (pipeline).
- There is no "core component" — the chain itself decides who handles the request.
- The client knows it is dealing with a chain and intentionally sends requests to it.
- Use case: authentication filter chain — if auth fails, nothing else executes.

**Choosing:**
- Use **Decorator** when: you need to add behavior to an object transparently, all wrappers always execute, and behavior is additive (streams, I/O, caching wrappers).
- Use **CoR** when: you need to route a request to the appropriate handler, handlers may reject requests, or the set of handlers changes at runtime (middleware, escalation, approval workflows).

A common real-world combination: Spring AOP uses Decorator for cross-cutting advice; Spring Security uses CoR for its filter chain. Both are present in the same application, doing different things.

---

## 12. Comparison with Related Patterns

### Chain of Responsibility vs Command

```
┌────────────────────────┬────────────────────────────┬────────────────────────────┐
│ Dimension              │ Chain of Responsibility     │ Command                    │
├────────────────────────┼────────────────────────────┼────────────────────────────┤
│ Core concern           │ WHO handles the request     │ WHAT to execute            │
│ Sender knowledge       │ Sender knows only the chain │ Invoker knows the command  │
│ Receiver               │ Dynamic; discovered at RT   │ Encapsulated in command obj│
│ Undo/Redo support      │ Not built-in                │ First-class feature        │
│ Queuing/scheduling     │ Not built-in                │ First-class feature        │
│ Number of handlers     │ Zero or more may act        │ Exactly one receiver       │
│ Use case               │ Middleware, escalation      │ Undo history, job queues   │
└────────────────────────┴────────────────────────────┴────────────────────────────┘
```

**When mixed:** A command queue where each command is dispatched through a CoR chain before execution — the queue provides ordering and scheduling (Command), the chain provides routing (CoR).

### Chain of Responsibility vs Decorator

```
┌────────────────────────┬────────────────────────────┬────────────────────────────┐
│ Dimension              │ Chain of Responsibility     │ Decorator                  │
├────────────────────────┼────────────────────────────┼────────────────────────────┤
│ Intent                 │ Route request to handler    │ Add behavior transparently │
│ All handlers execute?  │ No — can break the chain    │ Yes — all wrappers run     │
│ Core component         │ Absent (chain IS the logic) │ Required (wrapped object)  │
│ Client awareness       │ Client knows it's a chain   │ Client is unaware          │
│ Handler relationship   │ Peers — any can claim       │ Wrappers — outer wraps inner│
│ Short-circuit          │ Yes (on failure / claim)    │ No                         │
│ Example                │ Servlet filters             │ BufferedInputStream        │
└────────────────────────┴────────────────────────────┴────────────────────────────┘
```

### Chain of Responsibility vs Strategy

```
┌────────────────────────┬────────────────────────────┬────────────────────────────┐
│ Dimension              │ Chain of Responsibility     │ Strategy                   │
├────────────────────────┼────────────────────────────┼────────────────────────────┤
│ Handler selection      │ Runtime traversal of chain  │ Static injection / config  │
│ Number of algorithms   │ Many candidates, one (or 0) │ One algorithm at a time    │
│ Sender knowledge       │ Knows only chain head       │ Knows the strategy         │
│ Changeability          │ Chain can change at runtime │ Strategy can be swapped    │
│ Fallback behavior      │ Natural (next in chain)     │ Requires explicit default  │
│ Use case               │ Escalation, middleware      │ Sort algorithms, validators│
└────────────────────────┴────────────────────────────┴────────────────────────────┘
```

### Quick Reference: Pattern Selection

```
Request dispatch problem?
    |
    ├─ One algorithm, selected at config time? ──────────────────> Strategy
    |
    ├─ Encapsulate what to execute (undo/queue)? ────────────────> Command
    |
    ├─ Add behavior to object transparently? ────────────────────> Decorator
    |
    ├─ Fan-out to many listeners? ───────────────────────────────> Observer
    |
    └─ Route through ordered handlers, any can claim or pass? ──> Chain of Responsibility
           |
           ├─ Exactly one handler claims? ── Exclusive-claim CoR (escalation, routing)
           └─ All handlers contribute?    ── Pure-pipeline CoR  (middleware, filters)
```

---

*End of Chain of Responsibility — Module 04 / Behavioral Patterns / 05*
