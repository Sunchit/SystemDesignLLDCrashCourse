# Template Method Pattern

## Table of Contents
1. [Intent and Problem It Solves](#intent-and-problem-it-solves)
2. [Hook Methods](#hook-methods)
3. [When to Use / When NOT to Use](#when-to-use--when-not-to-use)
4. [UML Class Diagram](#uml-class-diagram)
5. [Hollywood Principle](#hollywood-principle)
6. [Implementation 1: Report Generation System](#implementation-1-report-generation-system)
7. [Implementation 2: Game Turn Sequence](#implementation-2-game-turn-sequence)
8. [Implementation 3: Data Miner](#implementation-3-data-miner)
9. [Template Method vs Strategy Pattern](#template-method-vs-strategy-pattern)
10. [Real-World Framework Examples](#real-world-framework-examples)
11. [Trade-offs](#trade-offs)
12. [Common Interview Discussion Points](#common-interview-discussion-points)
13. [Interview Questions and Answers](#interview-questions-and-answers)
14. [Comparison with Related Patterns](#comparison-with-related-patterns)

---

## Intent and Problem It Solves

### Definition

The **Template Method** pattern defines the skeleton of an algorithm in a base class, deferring some steps to subclasses. It lets subclasses redefine certain steps of an algorithm without changing the algorithm's overall structure.

**GoF Definition:**
> "Define the skeleton of an algorithm in an operation, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm's structure."

### The Core Problem

Consider building a data mining application that processes reports from different sources (PDF, CSV, Excel). Each source requires:

1. Opening the file
2. Extracting raw data
3. Parsing the data
4. Analyzing the data
5. Generating a report
6. Closing the file

Steps 3, 4, and 5 are nearly identical across all mining classes — only steps 1, 2, and 6 differ based on file type.

**Without Template Method — the duplication nightmare:**

```java
// BAD: Massive duplication across concrete classes
class PDFDataMiner {
    public void mine() {
        openPDF();         // unique
        extractPDF();      // unique
        parseData();       // DUPLICATED
        analyzeData();     // DUPLICATED
        generateReport();  // DUPLICATED
        closePDF();        // unique
    }
}

class CSVDataMiner {
    public void mine() {
        openCSV();         // unique
        extractCSV();      // unique
        parseData();       // DUPLICATED — same code as above
        analyzeData();     // DUPLICATED — same code as above
        generateReport();  // DUPLICATED — same code as above
        closeCSV();        // unique
    }
}
```

Problems with this approach:
- **Code duplication**: Common steps are copied everywhere
- **Maintenance risk**: A bug fix to `analyzeData()` must be applied in N places
- **Invariant violation**: Nothing prevents a subclass from reordering or skipping critical steps
- **No algorithmic contract**: Any subclass can implement `mine()` however it wants

### The Template Method Solution

Move the invariant parts of the algorithm into the base class and make only the variable parts abstract (or hook) methods:

```java
// GOOD: Algorithm skeleton locked in base class
abstract class DataMiner {
    // The template method — final so subclasses cannot reorder steps
    public final void mine() {
        openFile();        // abstract — subclass provides
        extractData();     // abstract — subclass provides
        parseData();       // concrete — base class owns
        analyzeData();     // concrete — base class owns
        generateReport();  // concrete — base class owns
        closeFile();       // abstract — subclass provides
    }

    protected abstract void openFile();
    protected abstract void extractData();
    protected abstract void closeFile();

    private void parseData()    { /* shared logic */ }
    private void analyzeData()  { /* shared logic */ }
    private void generateReport() { /* shared logic */ }
}
```

The algorithm's **control flow** is now owned by the base class. Subclasses cannot accidentally skip `analyzeData()` or run steps out of order.

### Key Insight: Inversion of Control at the Algorithm Level

Normal OOP: you call library code.
Template Method: the library calls your code (through abstract method overrides). This is the Hollywood Principle in action — covered in detail below.

---

## Hook Methods

### What Are Hook Methods?

A **hook method** is a method declared in the abstract base class that has a **default (often empty) implementation**. Subclasses *may* override hooks to inject behavior at specific points in the algorithm, but they are not required to.

Contrast with **abstract methods**, which subclasses *must* override.

```java
abstract class AbstractReport {

    // Template method
    public final void generate() {
        loadData();
        processData();
        if (shouldAddHeader()) {  // <-- hook used as conditional
            writeHeader();
        }
        writeBody();
        if (shouldAddFooter()) {  // <-- hook used as conditional
            writeFooter();
        }
        onComplete();             // <-- hook for post-processing
        output();
    }

    // Abstract methods — MUST be implemented
    protected abstract void loadData();
    protected abstract void writeBody();
    protected abstract void output();

    // Hook methods — MAY be overridden
    protected boolean shouldAddHeader() {
        return true; // default: yes
    }

    protected boolean shouldAddFooter() {
        return true; // default: yes
    }

    protected void writeHeader() {
        // default: do nothing
    }

    protected void writeFooter() {
        // default: do nothing
    }

    protected void onComplete() {
        // default: do nothing — lifecycle hook
    }

    // Concrete method — must NOT be overridden
    private void processData() {
        System.out.println("Processing data...");
    }
}
```

### Types of Hook Methods

**1. Conditional hooks (boolean return)**
Used to let subclasses toggle optional algorithm steps:
```java
protected boolean shouldCacheResult() { return false; }
protected boolean requiresAuthentication() { return true; }
```

**2. Lifecycle / event hooks (void)**
Notification points where subclasses can inject side effects:
```java
protected void beforeProcess() {}
protected void afterProcess() {}
protected void onError(Exception e) {}
```

**3. Provider hooks (non-void, non-boolean)**
Subclasses supply configuration values the algorithm needs:
```java
protected String getDelimiter() { return ","; }
protected int getPageSize() { return 100; }
protected Comparator<Row> getSortOrder() { return Comparator.naturalOrder(); }
```

### When to Use Hook Methods

Use a hook when:
- The step is **optional** in the algorithm (some subclasses need it, others do not)
- You want to provide a **sensible default** but allow customization
- You need **lifecycle notifications** (before/after events)
- The variation is a **configuration value**, not a full behavioral change
- You want to avoid forcing every subclass to implement a no-op override

Use an abstract method (not a hook) when:
- The step is **mandatory** — every subclass must provide a real implementation
- There is **no meaningful default** — a no-op would be wrong
- Forgetting to implement it should be a **compile-time error**

### Hooks vs Abstract Methods — Decision Table

| Criterion | Abstract Method | Hook Method |
|-----------|----------------|-------------|
| Implementation required? | Yes (compile error otherwise) | No (default provided) |
| Step is optional? | No | Yes |
| Has a meaningful default? | No | Yes |
| Used for lifecycle events? | Rarely | Commonly |
| Signals contract obligation? | Strong | Weak |

---

## When to Use / When NOT to Use

### When to Use

1. **Multiple classes share the same algorithm skeleton** but differ in one or more steps. You have recognized the common structure and want to eliminate duplication.

2. **You control the algorithm invariant** and want to prevent subclasses from changing it. Making the template method `final` enforces this.

3. **Framework design**: You are building an extensible framework where users plug in specific behavior. The framework controls the flow; users supply the variation points. (JUnit, Spring, Servlet API all do this.)

4. **Step variations are few and well-defined**. You know exactly which steps vary and which do not. If nearly everything varies, consider Strategy instead.

5. **Inheritance is already the right relationship**. The subclass truly *is a kind of* the base class, not just something that performs a similar task.

6. **You need compile-time enforcement** that certain steps are implemented (abstract methods give you this).

### When NOT to Use

1. **The algorithm varies too much across subclasses.** If most steps need overriding, the base class adds no value. Use Strategy or composition instead.

2. **You need to change algorithms at runtime.** Template Method uses inheritance — the algorithm is fixed at compile time. Strategy swaps algorithms dynamically.

3. **The inheritance hierarchy would become deep or wide.** Many concrete classes × many slight variations = combinatorial explosion. Prefer composition.

4. **Subclasses need to call super in complex ways.** If correct behavior requires `super.step()` calls in the right order, you have an invisible coupling that is easy to break.

5. **You are not building a framework or library.** In application code (not library code), Template Method often indicates premature abstraction. Wait until you have three concrete examples before extracting the template.

6. **The "common" steps actually differ subtly in each subclass.** You will end up with many hooks to accommodate all the variations, making the base class complicated.

---

## UML Class Diagram

```
┌──────────────────────────────────────────────────────┐
│             <<abstract>>                             │
│             AbstractClass                            │
├──────────────────────────────────────────────────────┤
│ + templateMethod() : void    ← final, defines flow  │
│ # primitiveOperation1() : void  ← abstract          │
│ # primitiveOperation2() : void  ← abstract          │
│ # hook1() : void             ← concrete (empty)     │
│ # hook2() : boolean          ← concrete (default)   │
│ - concreteOperation() : void ← private, invariant   │
└──────────────────────────┬───────────────────────────┘
                           │
             ┌─────────────┴─────────────┐
             │                           │
┌────────────▼───────────┐  ┌────────────▼───────────┐
│      ConcreteClassA    │  │      ConcreteClassB    │
├────────────────────────┤  ├────────────────────────┤
│ # primitiveOperation1()│  │ # primitiveOperation1()│
│ # primitiveOperation2()│  │ # primitiveOperation2()│
│ # hook1()  ← overrides │  │   (uses default hook1) │
└────────────────────────┘  └────────────────────────┘
```

### Participation Roles

| Role | Responsibility |
|------|---------------|
| `AbstractClass` | Defines `templateMethod()` and declares abstract/hook primitives |
| `templateMethod()` | The invariant algorithm skeleton — typically `final` |
| `primitiveOperation` | Abstract steps — subclass must override |
| `hook` | Optional steps — subclass may override |
| `ConcreteClass` | Implements abstract primitives; may override hooks |

### Sequence Diagram

```
Client          ConcreteClassA         AbstractClass
  │                   │                      │
  │──templateMethod()─►                      │
  │                   │──────────────────────►
  │                   │   concreteOperation() │
  │                   │◄──────────────────────
  │                   │◄─primitiveOperation1()│
  │                   │──[implementation]─────►
  │                   │◄──────────────────────
  │                   │◄─hook1()──────────────│  (if overridden)
  │                   │──[override]───────────►
  │                   │◄─primitiveOperation2()│
  │                   │──[implementation]─────►
  │                   │◄──────────────────────
  │◄──────────────────│                      │
```

---

## Hollywood Principle

### "Don't Call Us, We'll Call You"

The Hollywood Principle is an inversion of control (IoC) guideline that states:

> **High-level components call low-level components, not the other way around.**

In traditional procedural or "call up the hierarchy" programming, a low-level component explicitly calls utilities, frameworks, and high-level code. The low-level code is in charge.

The Hollywood Principle inverts this: **the high-level component (the framework or base class) defines when and how low-level code (the subclass or plugin) gets called.** The low-level component just registers itself and waits.

### Template Method as Hollywood Principle in Action

```java
abstract class AbstractReport {

    // HIGH-LEVEL COMPONENT controls the algorithm
    public final void generate() {
        loadData();         // <-- "We'll call you (loadData) when we need you"
        processData();
        formatOutput();     // <-- "We'll call you (formatOutput) when we need you"
        writeToDestination(); // <-- "We'll call you when we need you"
    }

    // LOW-LEVEL COMPONENTS just provide implementations
    // They do NOT call generate() themselves — they wait to be called
    protected abstract void loadData();
    protected abstract void formatOutput();
    protected abstract void writeToDestination();
}

class PDFReport extends AbstractReport {

    @Override
    protected void loadData() {
        // PDFReport does NOT control when this runs — AbstractReport does
        System.out.println("Loading data from database for PDF...");
    }

    @Override
    protected void formatOutput() {
        System.out.println("Formatting as PDF layout...");
    }

    @Override
    protected void writeToDestination() {
        System.out.println("Writing PDF bytes to output stream...");
    }
}
```

`PDFReport` never calls `generate()`. It never decides the order. It simply says "here is my implementation of `loadData()`" and the abstract class calls it at the right time.

### Why This Matters

**Without Hollywood Principle (bad — low-level controls flow):**
```java
class PDFReport {
    public void generate() {
        // PDFReport controls everything — coupling to the algorithm
        loadPDFData();
        new DataProcessor().process(data);  // calls upward
        new PDFFormatter().format(data);    // calls upward
        new FileWriter().write(output);     // calls upward
    }
}
```

Problems:
- `PDFReport` is coupled to `DataProcessor`, `PDFFormatter`, `FileWriter`
- Changing the algorithm order requires changing every concrete class
- The invariant (steps must run in order) is not enforced anywhere

**With Hollywood Principle (good — high-level controls flow):**
- The base class is the single authority on algorithm order
- Subclasses are called (not calling) — they are passive participants
- Adding a new subclass never risks breaking the algorithm structure
- The invariant is enforced by `final` on the template method

### Hollywood Principle Beyond Template Method

This principle appears throughout software design:

| Context | High-Level (calls) | Low-Level (called) |
|---------|-------------------|-------------------|
| Template Method | Abstract base class | Concrete subclass methods |
| Event systems | Event dispatcher | Event listeners/handlers |
| Dependency Injection | DI container | Components/services |
| Callbacks/Lambdas | Framework | Your callback function |
| Observer pattern | Subject/Observable | Observer implementations |
| Servlet API | `HttpServlet.service()` | Your `doGet()`/`doPost()` |

### Dependency Inversion vs Hollywood Principle

These are related but distinct:

- **Hollywood Principle**: About *who controls the flow of execution*. High-level code calls low-level code, not the reverse.
- **Dependency Inversion Principle (DIP)**: About *what you depend on*. Depend on abstractions, not concretions.

Template Method satisfies both: the abstract class (high-level policy) defines the flow and depends only on its own abstract method signatures (abstractions), while concrete subclasses (low-level details) depend on the abstract class but are called by it, not the other way.

---

## Implementation 1: Report Generation System

This implementation covers a production-quality report generation system where `PDFReport`, `HTMLReport`, `CSVReport`, and `ExcelReport` all extend `AbstractReport`. The template defines: data loading, processing, formatting, and output steps.

### Domain Model

```java
// ReportData.java
package com.course.patterns.templatemethod.report;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;

public class ReportData {

    private final String reportTitle;
    private final LocalDateTime generatedAt;
    private final List<String> headers;
    private final List<List<Object>> rows;
    private final Map<String, Object> metadata;

    public ReportData(String reportTitle,
                      List<String> headers,
                      List<List<Object>> rows,
                      Map<String, Object> metadata) {
        this.reportTitle = reportTitle;
        this.generatedAt = LocalDateTime.now();
        this.headers = Collections.unmodifiableList(new ArrayList<>(headers));
        this.rows = Collections.unmodifiableList(new ArrayList<>(rows));
        this.metadata = Collections.unmodifiableMap(metadata);
    }

    public String getReportTitle()         { return reportTitle; }
    public LocalDateTime getGeneratedAt()  { return generatedAt; }
    public List<String> getHeaders()       { return headers; }
    public List<List<Object>> getRows()    { return rows; }
    public Map<String, Object> getMetadata() { return metadata; }
    public int getRowCount()               { return rows.size(); }
    public int getColumnCount()            { return headers.size(); }

    @Override
    public String toString() {
        return String.format("ReportData[title='%s', rows=%d, cols=%d]",
                reportTitle, rows.size(), headers.size());
    }
}
```

```java
// ReportConfig.java
package com.course.patterns.templatemethod.report;

public class ReportConfig {

    private final String dataSourceUrl;
    private final String outputPath;
    private final String author;
    private final boolean includeMetadata;
    private final boolean applyFilters;

    private ReportConfig(Builder builder) {
        this.dataSourceUrl   = builder.dataSourceUrl;
        this.outputPath      = builder.outputPath;
        this.author          = builder.author;
        this.includeMetadata = builder.includeMetadata;
        this.applyFilters    = builder.applyFilters;
    }

    public String getDataSourceUrl()  { return dataSourceUrl; }
    public String getOutputPath()     { return outputPath; }
    public String getAuthor()         { return author; }
    public boolean isIncludeMetadata() { return includeMetadata; }
    public boolean isApplyFilters()   { return applyFilters; }

    public static Builder builder() { return new Builder(); }

    public static class Builder {
        private String dataSourceUrl = "jdbc:default";
        private String outputPath    = "/tmp/reports";
        private String author        = "System";
        private boolean includeMetadata = true;
        private boolean applyFilters    = false;

        public Builder dataSourceUrl(String url)       { this.dataSourceUrl = url; return this; }
        public Builder outputPath(String path)         { this.outputPath = path; return this; }
        public Builder author(String author)           { this.author = author; return this; }
        public Builder includeMetadata(boolean flag)   { this.includeMetadata = flag; return this; }
        public Builder applyFilters(boolean flag)      { this.applyFilters = flag; return this; }
        public ReportConfig build()                    { return new ReportConfig(this); }
    }
}
```

```java
// ReportException.java
package com.course.patterns.templatemethod.report;

public class ReportException extends RuntimeException {

    public enum Phase {
        DATA_LOADING, DATA_PROCESSING, FORMATTING, OUTPUT, VALIDATION
    }

    private final Phase phase;

    public ReportException(String message, Phase phase) {
        super(message);
        this.phase = phase;
    }

    public ReportException(String message, Phase phase, Throwable cause) {
        super(message, cause);
        this.phase = phase;
    }

    public Phase getPhase() { return phase; }

    @Override
    public String toString() {
        return String.format("ReportException[phase=%s, message=%s]", phase, getMessage());
    }
}
```

### Abstract Base Class

```java
// AbstractReport.java
package com.course.patterns.templatemethod.report;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.logging.Logger;

/**
 * Template Method pattern: defines the skeleton of the report generation algorithm.
 *
 * Algorithm steps (in order):
 *   1. validate()           — hook (optional validation before loading)
 *   2. loadData()           — abstract (subclass fetches raw data)
 *   3. preProcess()         — hook (optional data enrichment/filtering)
 *   4. processData()        — concrete (shared data processing logic)
 *   5. format()             — abstract (subclass formats for its target type)
 *   6. addWatermark()       — hook (optional watermark injection)
 *   7. output()             — abstract (subclass writes to its destination)
 *   8. onSuccess()          — hook (lifecycle: called after successful generation)
 *
 * The template method is declared final to prevent subclasses from
 * altering the algorithm's control flow.
 */
public abstract class AbstractReport {

    protected static final Logger logger = Logger.getLogger(AbstractReport.class.getName());

    protected final ReportConfig config;
    protected ReportData rawData;
    protected ReportData processedData;
    protected byte[] formattedOutput;

    private final List<String> auditLog = new ArrayList<>();

    protected AbstractReport(ReportConfig config) {
        if (config == null) {
            throw new IllegalArgumentException("ReportConfig must not be null");
        }
        this.config = config;
    }

    // =========================================================================
    // TEMPLATE METHOD — final: algorithm order is invariant
    // =========================================================================

    public final ReportResult generate() {
        Instant start = Instant.now();
        auditLog.clear();

        try {
            log("Starting report generation: " + getReportName());

            // Step 1: Optional pre-flight validation (hook)
            log("Step 1: Validation");
            validate();

            // Step 2: Load data — subclass responsibility
            log("Step 2: Loading data from " + config.getDataSourceUrl());
            this.rawData = loadData();
            if (rawData == null) {
                throw new ReportException(
                        "loadData() returned null", ReportException.Phase.DATA_LOADING);
            }
            log("Loaded " + rawData.getRowCount() + " rows");

            // Step 3: Optional pre-processing hook
            if (config.isApplyFilters()) {
                log("Step 3: Pre-processing (filters enabled)");
                preProcess();
            }

            // Step 4: Shared processing logic — base class owns this
            log("Step 4: Processing data");
            this.processedData = processData(rawData);

            // Step 5: Format — subclass responsibility
            log("Step 5: Formatting as " + getFormat());
            this.formattedOutput = format(processedData);
            if (formattedOutput == null || formattedOutput.length == 0) {
                throw new ReportException(
                        "format() returned empty output", ReportException.Phase.FORMATTING);
            }

            // Step 6: Optional watermark hook
            if (shouldAddWatermark()) {
                log("Step 6: Adding watermark");
                formattedOutput = addWatermark(formattedOutput);
            }

            // Step 7: Write output — subclass responsibility
            log("Step 7: Writing output to " + config.getOutputPath());
            String outputLocation = output(formattedOutput);

            // Step 8: Success lifecycle hook
            onSuccess(outputLocation);

            Duration elapsed = Duration.between(start, Instant.now());
            log("Report generation complete in " + elapsed.toMillis() + "ms");

            return ReportResult.success(
                    getReportName(),
                    outputLocation,
                    formattedOutput.length,
                    processedData.getRowCount(),
                    elapsed,
                    List.copyOf(auditLog));

        } catch (ReportException e) {
            onError(e);
            log("Report generation failed at phase " + e.getPhase() + ": " + e.getMessage());
            return ReportResult.failure(getReportName(), e, List.copyOf(auditLog));
        } catch (Exception e) {
            ReportException wrapped = new ReportException(
                    "Unexpected error: " + e.getMessage(),
                    ReportException.Phase.DATA_PROCESSING, e);
            onError(wrapped);
            return ReportResult.failure(getReportName(), wrapped, List.copyOf(auditLog));
        }
    }

    // =========================================================================
    // ABSTRACT METHODS — subclasses MUST implement
    // =========================================================================

    /**
     * Load raw data from the data source.
     * Subclasses connect to their specific source (DB, file, API, etc.).
     */
    protected abstract ReportData loadData();

    /**
     * Format the processed data into the target output format (PDF bytes, HTML string, etc.).
     */
    protected abstract byte[] format(ReportData data);

    /**
     * Write formatted output to the destination and return the output location URI.
     */
    protected abstract String output(byte[] formattedBytes);

    /**
     * Human-readable name of this report type.
     */
    public abstract String getReportName();

    /**
     * The output format identifier (PDF, HTML, CSV, XLSX).
     */
    protected abstract String getFormat();

    // =========================================================================
    // CONCRETE METHOD — invariant processing logic, not overridable
    // =========================================================================

    /**
     * Shared data processing: sort, aggregate, apply business rules.
     * This logic is identical for all report types — it lives here.
     */
    private ReportData processData(ReportData data) {
        logger.fine("Processing " + data.getRowCount() + " rows");

        // Sort rows by first column value (lexicographic for simplicity)
        List<List<Object>> sorted = new ArrayList<>(data.getRows());
        sorted.sort((a, b) -> {
            if (a.isEmpty() || b.isEmpty()) return 0;
            return String.valueOf(a.get(0)).compareTo(String.valueOf(b.get(0)));
        });

        // Apply shared business rules: remove rows where first cell is null/empty
        List<List<Object>> filtered = sorted.stream()
                .filter(row -> !row.isEmpty() && row.get(0) != null
                        && !String.valueOf(row.get(0)).isBlank())
                .toList();

        return new ReportData(
                data.getReportTitle(),
                data.getHeaders(),
                filtered,
                data.getMetadata());
    }

    // =========================================================================
    // HOOK METHODS — subclasses MAY override
    // =========================================================================

    /**
     * Hook: Pre-flight validation before data loading.
     * Default: no-op. Override to add custom checks.
     * Throw ReportException if validation fails.
     */
    protected void validate() {
        // default: no validation
    }

    /**
     * Hook: Optional data enrichment or additional filtering before processData().
     * Default: no-op. Override to add joins, lookups, or pre-filtering.
     */
    protected void preProcess() {
        // default: no pre-processing
    }

    /**
     * Hook: Whether to add a watermark to the formatted output.
     * Default: false.
     */
    protected boolean shouldAddWatermark() {
        return false;
    }

    /**
     * Hook: Add a watermark to the formatted bytes.
     * Called only if shouldAddWatermark() returns true.
     * Default: returns bytes unchanged.
     */
    protected byte[] addWatermark(byte[] formattedBytes) {
        return formattedBytes; // default: no watermark
    }

    /**
     * Hook: Called after successful report generation.
     * Default: logs the output location.
     */
    protected void onSuccess(String outputLocation) {
        logger.info("Report saved to: " + outputLocation);
    }

    /**
     * Hook: Called when an error occurs during generation.
     * Default: logs the error.
     */
    protected void onError(ReportException e) {
        logger.severe("Report failed: " + e);
    }

    // =========================================================================
    // UTILITY
    // =========================================================================

    protected void log(String message) {
        auditLog.add("[" + java.time.LocalTime.now() + "] " + message);
        logger.fine(message);
    }

    protected ReportConfig getConfig() {
        return config;
    }
}
```

```java
// ReportResult.java
package com.course.patterns.templatemethod.report;

import java.time.Duration;
import java.util.List;

public class ReportResult {

    public enum Status { SUCCESS, FAILURE }

    private final String reportName;
    private final Status status;
    private final String outputLocation;
    private final long outputSizeBytes;
    private final int rowCount;
    private final Duration elapsed;
    private final ReportException error;
    private final List<String> auditLog;

    private ReportResult(String reportName, Status status, String outputLocation,
                         long outputSizeBytes, int rowCount, Duration elapsed,
                         ReportException error, List<String> auditLog) {
        this.reportName      = reportName;
        this.status          = status;
        this.outputLocation  = outputLocation;
        this.outputSizeBytes = outputSizeBytes;
        this.rowCount        = rowCount;
        this.elapsed         = elapsed;
        this.error           = error;
        this.auditLog        = auditLog;
    }

    public static ReportResult success(String reportName, String outputLocation,
                                       long sizeBytes, int rowCount,
                                       Duration elapsed, List<String> auditLog) {
        return new ReportResult(reportName, Status.SUCCESS, outputLocation,
                sizeBytes, rowCount, elapsed, null, auditLog);
    }

    public static ReportResult failure(String reportName, ReportException error,
                                       List<String> auditLog) {
        return new ReportResult(reportName, Status.FAILURE, null, 0, 0,
                Duration.ZERO, error, auditLog);
    }

    public boolean isSuccess()          { return status == Status.SUCCESS; }
    public String getReportName()       { return reportName; }
    public Status getStatus()           { return status; }
    public String getOutputLocation()   { return outputLocation; }
    public long getOutputSizeBytes()    { return outputSizeBytes; }
    public int getRowCount()            { return rowCount; }
    public Duration getElapsed()        { return elapsed; }
    public ReportException getError()   { return error; }
    public List<String> getAuditLog()   { return auditLog; }

    @Override
    public String toString() {
        if (isSuccess()) {
            return String.format("ReportResult[%s, SUCCESS, rows=%d, size=%d bytes, time=%dms, output=%s]",
                    reportName, rowCount, outputSizeBytes, elapsed.toMillis(), outputLocation);
        } else {
            return String.format("ReportResult[%s, FAILURE, error=%s]", reportName, error);
        }
    }
}
```

### Concrete Implementations

```java
// PDFReport.java
package com.course.patterns.templatemethod.report;

import java.nio.charset.StandardCharsets;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * Generates a PDF report. Connects to a relational database.
 * Overrides the watermark hook to add "CONFIDENTIAL" to all PDFs.
 */
public class PDFReport extends AbstractReport {

    private static final String WATERMARK_TEXT = "CONFIDENTIAL";
    private final boolean confidential;

    public PDFReport(ReportConfig config, boolean confidential) {
        super(config);
        this.confidential = confidential;
    }

    @Override
    public String getReportName() {
        return "PDF Sales Report";
    }

    @Override
    protected String getFormat() {
        return "PDF";
    }

    // -------------------------------------------------------------------------
    // Abstract method implementations
    // -------------------------------------------------------------------------

    @Override
    protected ReportData loadData() {
        log("PDFReport: connecting to relational database at " + config.getDataSourceUrl());
        // In production: use JDBC. Here we simulate realistic data.

        List<String> headers = List.of("Region", "Product", "Q1 Sales", "Q2 Sales", "Total");

        List<List<Object>> rows = List.of(
                List.of("North America", "Widget Pro",  125_000, 132_000, 257_000),
                List.of("Europe",        "Widget Pro",   98_000,  105_000, 203_000),
                List.of("Asia Pacific",  "Widget Lite",  67_000,   72_000, 139_000),
                List.of("",              "Ghost Row",         0,        0,       0),  // will be filtered
                List.of("Latin America", "Widget Pro",   34_000,   39_000,  73_000),
                List.of("Middle East",   "Widget Lite",  21_000,   25_000,  46_000)
        );

        Map<String, Object> metadata = new HashMap<>();
        metadata.put("source", "sales_db");
        metadata.put("query_time_ms", 42);
        metadata.put("author", config.getAuthor());

        return new ReportData("Sales Performance Report", headers, rows, metadata);
    }

    @Override
    protected byte[] format(ReportData data) {
        log("PDFReport: rendering PDF layout with iText (simulated)");

        // Production: use iText, Apache PDFBox, or Jasper Reports
        // Here we produce a structured text representation of the PDF content
        StringBuilder pdfContent = new StringBuilder();
        pdfContent.append("%PDF-1.4 (simulated)\n");
        pdfContent.append("Title: ").append(data.getReportTitle()).append("\n");
        pdfContent.append("Generated: ").append(data.getGeneratedAt()).append("\n");
        pdfContent.append("Author: ").append(data.getMetadata().get("author")).append("\n");
        pdfContent.append("=".repeat(60)).append("\n");

        // Headers
        pdfContent.append(String.join(" | ", data.getHeaders())).append("\n");
        pdfContent.append("-".repeat(60)).append("\n");

        // Rows
        for (List<Object> row : data.getRows()) {
            StringBuilder rowBuilder = new StringBuilder();
            for (int i = 0; i < row.size(); i++) {
                if (i > 0) rowBuilder.append(" | ");
                rowBuilder.append(String.valueOf(row.get(i)));
            }
            pdfContent.append(rowBuilder).append("\n");
        }

        pdfContent.append("=".repeat(60)).append("\n");
        pdfContent.append("Total rows: ").append(data.getRowCount()).append("\n");

        return pdfContent.toString().getBytes(StandardCharsets.UTF_8);
    }

    @Override
    protected String output(byte[] formattedBytes) {
        String path = config.getOutputPath() + "/sales_report.pdf";
        log("PDFReport: writing " + formattedBytes.length + " bytes to " + path);
        // Production: Files.write(Paths.get(path), formattedBytes);
        return path;
    }

    // -------------------------------------------------------------------------
    // Hook overrides
    // -------------------------------------------------------------------------

    @Override
    protected void validate() {
        if (config.getDataSourceUrl() == null || config.getDataSourceUrl().isBlank()) {
            throw new ReportException(
                    "PDFReport requires a valid data source URL",
                    ReportException.Phase.VALIDATION);
        }
        log("PDFReport: validation passed");
    }

    @Override
    protected boolean shouldAddWatermark() {
        return confidential;
    }

    @Override
    protected byte[] addWatermark(byte[] formattedBytes) {
        log("PDFReport: applying CONFIDENTIAL watermark");
        // Production: use iText to stamp watermark on each PDF page
        String existing = new String(formattedBytes, StandardCharsets.UTF_8);
        String watermarked = "*** " + WATERMARK_TEXT + " ***\n" + existing
                + "\n*** " + WATERMARK_TEXT + " ***";
        return watermarked.getBytes(StandardCharsets.UTF_8);
    }

    @Override
    protected void onSuccess(String outputLocation) {
        super.onSuccess(outputLocation); // call base hook
        log("PDFReport: sending email notification to " + config.getAuthor());
        // Production: emailService.send(config.getAuthor(), "Report ready: " + outputLocation);
    }
}
```

```java
// HTMLReport.java
package com.course.patterns.templatemethod.report;

import java.nio.charset.StandardCharsets;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * Generates an HTML report from a REST API data source.
 * Applies CSS styling and embeds chart placeholders.
 */
public class HTMLReport extends AbstractReport {

    private final String cssTheme;

    public HTMLReport(ReportConfig config, String cssTheme) {
        super(config);
        this.cssTheme = cssTheme != null ? cssTheme : "default";
    }

    @Override
    public String getReportName() {
        return "HTML Dashboard Report";
    }

    @Override
    protected String getFormat() {
        return "HTML";
    }

    @Override
    protected ReportData loadData() {
        log("HTMLReport: fetching JSON data from REST API at " + config.getDataSourceUrl());
        // Production: use HttpClient, parse JSON with Jackson

        List<String> headers = List.of("Month", "Revenue", "Expenses", "Net Profit", "Margin%");

        List<List<Object>> rows = List.of(
                List.of("January",  480_000, 310_000, 170_000, "35.4%"),
                List.of("February", 520_000, 330_000, 190_000, "36.5%"),
                List.of("March",    610_000, 375_000, 235_000, "38.5%"),
                List.of("April",    575_000, 360_000, 215_000, "37.4%"),
                List.of("May",      680_000, 410_000, 270_000, "39.7%"),
                List.of("June",     720_000, 430_000, 290_000, "40.3%")
        );

        Map<String, Object> metadata = new HashMap<>();
        metadata.put("source", "analytics_api");
        metadata.put("fiscal_year", "2024");

        return new ReportData("Monthly P&L Dashboard", headers, rows, metadata);
    }

    @Override
    protected byte[] format(ReportData data) {
        log("HTMLReport: generating responsive HTML with " + cssTheme + " theme");

        StringBuilder html = new StringBuilder();
        html.append("<!DOCTYPE html>\n<html lang=\"en\">\n<head>\n");
        html.append("  <meta charset=\"UTF-8\">\n");
        html.append("  <meta name=\"viewport\" content=\"width=device-width, initial-scale=1\">\n");
        html.append("  <title>").append(data.getReportTitle()).append("</title>\n");
        html.append("  <style>\n");
        html.append(getCssForTheme());
        html.append("  </style>\n</head>\n<body>\n");

        html.append("  <div class=\"report-container\">\n");
        html.append("    <h1 class=\"report-title\">").append(data.getReportTitle()).append("</h1>\n");
        html.append("    <p class=\"report-meta\">Generated: ").append(data.getGeneratedAt()).append("</p>\n");
        html.append("    <p class=\"report-meta\">Fiscal Year: ")
                .append(data.getMetadata().get("fiscal_year")).append("</p>\n");

        // Chart placeholder (production: embed Chart.js or D3)
        html.append("    <div class=\"chart-placeholder\">[Revenue Trend Chart]</div>\n");

        // Data table
        html.append("    <table class=\"data-table\">\n      <thead>\n        <tr>\n");
        for (String header : data.getHeaders()) {
            html.append("          <th>").append(escapeHtml(header)).append("</th>\n");
        }
        html.append("        </tr>\n      </thead>\n      <tbody>\n");

        for (int i = 0; i < data.getRows().size(); i++) {
            String rowClass = i % 2 == 0 ? "row-even" : "row-odd";
            html.append("        <tr class=\"").append(rowClass).append("\">\n");
            for (Object cell : data.getRows().get(i)) {
                html.append("          <td>").append(escapeHtml(String.valueOf(cell))).append("</td>\n");
            }
            html.append("        </tr>\n");
        }

        html.append("      </tbody>\n    </table>\n");
        html.append("    <p class=\"row-count\">Total rows: ").append(data.getRowCount()).append("</p>\n");
        html.append("  </div>\n</body>\n</html>");

        return html.toString().getBytes(StandardCharsets.UTF_8);
    }

    @Override
    protected String output(byte[] formattedBytes) {
        String path = config.getOutputPath() + "/dashboard.html";
        log("HTMLReport: writing " + formattedBytes.length + " bytes to " + path);
        // Production: Files.write(Paths.get(path), formattedBytes);
        return path;
    }

    // Hook: HTML reports do not need watermarks
    @Override
    protected boolean shouldAddWatermark() {
        return false;
    }

    // Hook: pre-process can add metadata enrichment
    @Override
    protected void preProcess() {
        log("HTMLReport: enriching data with computed summary row");
        // Production: compute totals, add summary rows, join lookup tables
    }

    private String getCssForTheme() {
        return switch (cssTheme) {
            case "dark" -> """
                    body { background: #1a1a2e; color: #eee; font-family: Arial, sans-serif; }
                    .data-table { width: 100%; border-collapse: collapse; }
                    th { background: #16213e; padding: 10px; }
                    .row-even { background: #0f3460; }
                    .row-odd  { background: #1a1a2e; }
                    .chart-placeholder { height: 200px; background: #16213e; margin: 20px 0;
                                         display:flex; align-items:center; justify-content:center; }
                    """;
            case "corporate" -> """
                    body { background: #f5f5f5; font-family: 'Helvetica', sans-serif; }
                    .data-table { width: 100%; border-collapse: collapse; border: 1px solid #ccc; }
                    th { background: #003087; color: white; padding: 10px; }
                    .row-even { background: #ffffff; }
                    .row-odd  { background: #e8f0fe; }
                    .chart-placeholder { height: 200px; background: #e8f0fe; margin: 20px 0; }
                    """;
            default -> """
                    body { font-family: sans-serif; margin: 20px; }
                    .data-table { width: 100%; border-collapse: collapse; }
                    th { background: #4CAF50; color: white; padding: 8px; }
                    .row-even { background: #f2f2f2; }
                    .row-odd  { background: #ffffff; }
                    .chart-placeholder { height: 200px; background: #f9f9f9; margin: 20px 0; }
                    """;
        };
    }

    private String escapeHtml(String text) {
        return text.replace("&", "&amp;")
                   .replace("<", "&lt;")
                   .replace(">", "&gt;")
                   .replace("\"", "&quot;");
    }
}
```

```java
// CSVReport.java
package com.course.patterns.templatemethod.report;

import java.nio.charset.StandardCharsets;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * Generates a CSV report from a file-based data source.
 * CSV reports are plain data exports — no watermark, no header/footer beyond the column row.
 */
public class CSVReport extends AbstractReport {

    private final char delimiter;
    private final boolean includeHeaderRow;

    public CSVReport(ReportConfig config, char delimiter, boolean includeHeaderRow) {
        super(config);
        this.delimiter       = delimiter;
        this.includeHeaderRow = includeHeaderRow;
    }

    @Override
    public String getReportName() {
        return "CSV Data Export";
    }

    @Override
    protected String getFormat() {
        return "CSV";
    }

    @Override
    protected ReportData loadData() {
        log("CSVReport: reading source CSV file from " + config.getDataSourceUrl());
        // Production: parse the source CSV using Apache Commons CSV or OpenCSV

        List<String> headers = List.of(
                "customer_id", "name", "email", "signup_date",
                "total_orders", "lifetime_value", "segment");

        List<List<Object>> rows = List.of(
                List.of("C001", "Alice Martin",   "alice@example.com",  "2021-03-15", 24, 4_560.00, "Premium"),
                List.of("C002", "Bob Johnson",    "bob@example.com",    "2022-07-22", 11, 1_890.00, "Standard"),
                List.of("C003", "Carol Williams", "carol@example.com",  "2020-11-08", 47, 9_230.00, "VIP"),
                List.of("C004", "David Lee",      "david@example.com",  "2023-01-30",  5,   780.00, "Standard"),
                List.of("C005", "Emma Davis",     "emma@example.com",   "2019-06-14", 83, 18_400.00, "VIP")
        );

        Map<String, Object> metadata = new HashMap<>();
        metadata.put("source_file", "customers_2024.csv");
        metadata.put("encoding", "UTF-8");

        return new ReportData("Customer Data Export", headers, rows, metadata);
    }

    @Override
    protected byte[] format(ReportData data) {
        log("CSVReport: formatting with delimiter='" + delimiter + "'");

        StringBuilder csv = new StringBuilder();

        // Optional BOM for Excel compatibility with UTF-8
        // csv.append('﻿');  // uncomment if target is Excel on Windows

        if (includeHeaderRow) {
            csv.append(formatRow(data.getHeaders().stream()
                    .map(Object.class::cast)
                    .toList()));
            csv.append("\n");
        }

        for (List<Object> row : data.getRows()) {
            csv.append(formatRow(row)).append("\n");
        }

        return csv.toString().getBytes(StandardCharsets.UTF_8);
    }

    @Override
    protected String output(byte[] formattedBytes) {
        String path = config.getOutputPath() + "/customers_export.csv";
        log("CSVReport: writing " + formattedBytes.length + " bytes to " + path);
        // Production: Files.write(Paths.get(path), formattedBytes);
        return path;
    }

    // Hook: CSV reports do not use watermarks
    @Override
    protected boolean shouldAddWatermark() {
        return false;
    }

    private String formatRow(List<Object> cells) {
        StringBuilder row = new StringBuilder();
        for (int i = 0; i < cells.size(); i++) {
            if (i > 0) row.append(delimiter);
            row.append(escapeCsvCell(String.valueOf(cells.get(i))));
        }
        return row.toString();
    }

    private String escapeCsvCell(String value) {
        // RFC 4180: if the value contains delimiter, newline, or double-quote, wrap in quotes
        if (value.contains(String.valueOf(delimiter))
                || value.contains("\"")
                || value.contains("\n")
                || value.contains("\r")) {
            return "\"" + value.replace("\"", "\"\"") + "\"";
        }
        return value;
    }
}
```

```java
// ExcelReport.java
package com.course.patterns.templatemethod.report;

import java.nio.charset.StandardCharsets;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * Generates an Excel-compatible report (OOXML/XLSX format simulated).
 * In production this would use Apache POI.
 * Demonstrates a more complex hook pattern: the report fetches from multiple sources
 * and uses the preProcess hook to merge them.
 */
public class ExcelReport extends AbstractReport {

    private final boolean includeCharts;
    private final boolean enableConditionalFormatting;

    public ExcelReport(ReportConfig config, boolean includeCharts,
                       boolean enableConditionalFormatting) {
        super(config);
        this.includeCharts = includeCharts;
        this.enableConditionalFormatting = enableConditionalFormatting;
    }

    @Override
    public String getReportName() {
        return "Excel Multi-Sheet Report";
    }

    @Override
    protected String getFormat() {
        return "XLSX";
    }

    @Override
    protected ReportData loadData() {
        log("ExcelReport: loading financial data from data warehouse");
        // Production: query Redshift / BigQuery / Snowflake via JDBC

        List<String> headers = List.of(
                "Department", "Budget", "Actual", "Variance", "Variance%", "Status");

        List<List<Object>> rows = List.of(
                List.of("Engineering",  500_000, 487_000,  13_000, "2.6%",  "ON TRACK"),
                List.of("Marketing",    300_000, 341_000, -41_000, "-13.7%", "OVER BUDGET"),
                List.of("Sales",        750_000, 720_000,  30_000, "4.0%",  "ON TRACK"),
                List.of("Operations",   250_000, 248_000,   2_000, "0.8%",  "ON TRACK"),
                List.of("HR",           150_000, 155_000,  -5_000, "-3.3%", "SLIGHTLY OVER"),
                List.of("Finance",      120_000, 118_000,   2_000, "1.7%",  "ON TRACK")
        );

        Map<String, Object> metadata = new HashMap<>();
        metadata.put("source", "data_warehouse");
        metadata.put("fiscal_quarter", "Q2-2024");
        metadata.put("currency", "USD");

        return new ReportData("Budget vs Actual Analysis", headers, rows, metadata);
    }

    @Override
    protected byte[] format(ReportData data) {
        log("ExcelReport: building XLSX workbook (Apache POI simulated)");
        // Production: use Apache POI XSSFWorkbook
        // - Create workbook, add sheets
        // - Apply cell styles, fonts, colors
        // - Add data validation, formulas
        // - If includeCharts: add bar chart for variances
        // - If enableConditionalFormatting: highlight OVER BUDGET rows in red

        StringBuilder xlsx = new StringBuilder();
        xlsx.append("[XLSX Workbook: ").append(data.getReportTitle()).append("]\n");
        xlsx.append("Sheet 1: Summary\n");
        xlsx.append("Fiscal Quarter: ").append(data.getMetadata().get("fiscal_quarter")).append("\n");
        xlsx.append("Currency: ").append(data.getMetadata().get("currency")).append("\n\n");

        // Headers with formatting markers
        for (String header : data.getHeaders()) {
            xlsx.append("[BOLD]").append(header).append("\t");
        }
        xlsx.append("\n");

        // Data rows with conditional formatting
        for (List<Object> row : data.getRows()) {
            String status = String.valueOf(row.get(5));
            String prefix = status.contains("OVER") ? "[RED_BG]" : "";
            for (Object cell : row) {
                xlsx.append(prefix).append(cell).append("\t");
            }
            xlsx.append("\n");
        }

        if (includeCharts) {
            xlsx.append("\nSheet 2: [VARIANCE BAR CHART]\n");
        }

        if (enableConditionalFormatting) {
            xlsx.append("\n[Conditional Formatting Rules Applied: OVER BUDGET = Red]\n");
        }

        xlsx.append("\nSheet 3: Raw Data\n");
        xlsx.append("[Raw data rows: ").append(data.getRowCount()).append("]\n");

        return xlsx.toString().getBytes(StandardCharsets.UTF_8);
    }

    @Override
    protected String output(byte[] formattedBytes) {
        String path = config.getOutputPath() + "/budget_analysis.xlsx";
        log("ExcelReport: writing " + formattedBytes.length + " bytes to " + path);
        // Production: workbook.write(new FileOutputStream(path));
        return path;
    }

    // Hook: Excel reports with financial data get a DRAFT watermark
    @Override
    protected boolean shouldAddWatermark() {
        return true;
    }

    @Override
    protected byte[] addWatermark(byte[] formattedBytes) {
        log("ExcelReport: stamping DRAFT watermark on all sheets");
        String content = new String(formattedBytes, StandardCharsets.UTF_8);
        return ("[WATERMARK: DRAFT - For Internal Review Only]\n" + content)
                .getBytes(StandardCharsets.UTF_8);
    }

    // Hook: Use preProcess to merge supplementary data
    @Override
    protected void preProcess() {
        log("ExcelReport: merging actuals with prior year comparatives from secondary source");
        // Production: JOIN the loaded data with a second query result
    }

    @Override
    protected void validate() {
        if (config.getOutputPath() == null || config.getOutputPath().isBlank()) {
            throw new ReportException(
                    "Excel report requires a valid output path",
                    ReportException.Phase.VALIDATION);
        }
        log("ExcelReport: validation passed");
    }

    @Override
    protected void onSuccess(String outputLocation) {
        super.onSuccess(outputLocation);
        log("ExcelReport: notifying finance team via Slack webhook");
        // Production: slackClient.postMessage("#finance", "New budget report: " + outputLocation);
    }
}
```

### Client Code and Test Driver

```java
// ReportGenerationDemo.java
package com.course.patterns.templatemethod.report;

import java.util.List;

public class ReportGenerationDemo {

    public static void main(String[] args) {
        System.out.println("=".repeat(70));
        System.out.println("Template Method Pattern — Report Generation System Demo");
        System.out.println("=".repeat(70));

        // 1. PDF Report — confidential, uses watermark hook
        ReportConfig pdfConfig = ReportConfig.builder()
                .dataSourceUrl("jdbc:postgresql://sales-db:5432/salesdb")
                .outputPath("/reports/2024/Q2")
                .author("finance-team@company.com")
                .build();

        AbstractReport pdfReport = new PDFReport(pdfConfig, true);
        ReportResult pdfResult = pdfReport.generate();
        printResult(pdfResult);

        // 2. HTML Report — dark theme dashboard
        ReportConfig htmlConfig = ReportConfig.builder()
                .dataSourceUrl("https://analytics-api.internal/v2/monthly")
                .outputPath("/reports/web")
                .author("analytics@company.com")
                .applyFilters(true)
                .build();

        AbstractReport htmlReport = new HTMLReport(htmlConfig, "dark");
        ReportResult htmlResult = htmlReport.generate();
        printResult(htmlResult);

        // 3. CSV Report — tab-separated with header row
        ReportConfig csvConfig = ReportConfig.builder()
                .dataSourceUrl("/data/customers_2024.csv")
                .outputPath("/exports")
                .author("data-engineering@company.com")
                .build();

        AbstractReport csvReport = new CSVReport(csvConfig, '\t', true);
        ReportResult csvResult = csvReport.generate();
        printResult(csvResult);

        // 4. Excel Report — with charts and conditional formatting
        ReportConfig excelConfig = ReportConfig.builder()
                .dataSourceUrl("jdbc:redshift://dw.internal:5439/analytics")
                .outputPath("/reports/finance")
                .author("cfo@company.com")
                .applyFilters(true)
                .build();

        AbstractReport excelReport = new ExcelReport(excelConfig, true, true);
        ReportResult excelResult = excelReport.generate();
        printResult(excelResult);

        // Summary
        System.out.println("\n" + "=".repeat(70));
        System.out.println("All 4 reports used the SAME algorithm skeleton:");
        System.out.println("  validate → loadData → preProcess → processData → format → addWatermark → output");
        System.out.println("Only the VARIABLE STEPS differed across subclasses.");
        System.out.println("=".repeat(70));
    }

    private static void printResult(ReportResult result) {
        System.out.println("\n--- " + result.getReportName() + " ---");
        System.out.println(result);
        System.out.println("Audit trail:");
        result.getAuditLog().forEach(entry -> System.out.println("  " + entry));
    }
}
```

---

## Implementation 2: Game Turn Sequence

```java
// AbstractGame.java
package com.course.patterns.templatemethod.game;

/**
 * Template Method pattern for a two-player board game turn sequence.
 *
 * Algorithm skeleton:
 *   1. initialize()     — abstract: set up board and game state
 *   2. Loop until isOver():
 *      a. printBoard()  — hook: display current state (optional for headless mode)
 *      b. makeMove()    — abstract: current player makes a move
 *      c. onTurnEnd()   — hook: post-turn logic (update scores, check combos, etc.)
 *   3. printWinner()    — hook: announce result
 */
public abstract class AbstractGame {

    protected int numberOfPlayers;
    protected int currentPlayer;
    protected int totalTurns;

    // =========================================================================
    // TEMPLATE METHOD
    // =========================================================================

    public final void playGame() {
        System.out.println("[" + getGameName() + "] Initializing game...");
        initialize();
        currentPlayer = firstPlayer();
        totalTurns    = 0;

        System.out.println("[" + getGameName() + "] Game started. Players: " + numberOfPlayers);

        while (!isOver()) {
            totalTurns++;

            if (shouldPrintBoard()) {
                printBoard();
            }

            System.out.println("[" + getGameName() + "] Turn " + totalTurns
                    + " — Player " + (currentPlayer + 1) + " moves");
            makeMove(currentPlayer);
            onTurnEnd(currentPlayer);
            currentPlayer = nextPlayer();
        }

        if (shouldPrintBoard()) {
            printBoard(); // show final board state
        }

        printWinner();
        onGameEnd();

        System.out.println("[" + getGameName() + "] Game over after " + totalTurns + " turns.");
    }

    // =========================================================================
    // ABSTRACT METHODS
    // =========================================================================

    /** Set up the board, pieces, player count, and any initial state. */
    protected abstract void initialize();

    /** Current player makes a move. Mutate game state accordingly. */
    protected abstract void makeMove(int player);

    /** Return true when the game has ended (win, draw, or no more moves). */
    protected abstract boolean isOver();

    /** Human-readable game name. */
    public abstract String getGameName();

    // =========================================================================
    // HOOK METHODS
    // =========================================================================

    /** Which player goes first. Default: player 0. */
    protected int firstPlayer() {
        return 0;
    }

    /** Determine the next player. Default: round-robin. */
    protected int nextPlayer() {
        return (currentPlayer + 1) % numberOfPlayers;
    }

    /** Whether to print the board on each turn. Default: true. */
    protected boolean shouldPrintBoard() {
        return true;
    }

    /** Display the current board state. Default: no-op. */
    protected void printBoard() {
        // default: no-op — subclasses override for visual output
    }

    /** Called after each turn ends. Default: no-op. */
    protected void onTurnEnd(int playerWhoMoved) {
        // default: no-op
    }

    /** Announce the winner or draw. Default: generic message. */
    protected void printWinner() {
        System.out.println("[" + getGameName() + "] Game complete!");
    }

    /** Called once when the game fully ends. Default: no-op. */
    protected void onGameEnd() {
        // default: no-op
    }
}
```

```java
// ChessGame.java
package com.course.patterns.templatemethod.game;

import java.util.Random;

/**
 * Chess implementation. Simplified: pieces move randomly, game ends after 10 moves
 * or when a king is captured. Demonstrates complex state tracking in makeMove().
 */
public class ChessGame extends AbstractGame {

    private static final String[][] INITIAL_BOARD = {
            {"R", "N", "B", "Q", "K", "B", "N", "R"},
            {"P", "P", "P", "P", "P", "P", "P", "P"},
            {" ", " ", " ", " ", " ", " ", " ", " "},
            {" ", " ", " ", " ", " ", " ", " ", " "},
            {" ", " ", " ", " ", " ", " ", " ", " "},
            {" ", " ", " ", " ", " ", " ", " ", " "},
            {"p", "p", "p", "p", "p", "p", "p", "p"},
            {"r", "n", "b", "q", "k", "b", "n", "r"}
    };

    private String[][] board;
    private boolean kingCaptured;
    private int capturedKingByPlayer;
    private final Random random;
    private int moveCount;
    private static final int MAX_MOVES = 6; // shortened for demo

    public ChessGame() {
        this.random = new Random(42); // seeded for reproducibility
    }

    @Override
    public String getGameName() {
        return "Chess";
    }

    @Override
    protected void initialize() {
        numberOfPlayers = 2;
        board = new String[8][8];
        for (int i = 0; i < 8; i++) {
            board[i] = INITIAL_BOARD[i].clone();
        }
        kingCaptured         = false;
        capturedKingByPlayer = -1;
        moveCount            = 0;
        System.out.println("Chess: board initialized. White (Player 1) vs Black (Player 2).");
    }

    @Override
    protected void makeMove(int player) {
        moveCount++;
        // Simplified: randomly move a piece to a random square
        int fromRow = random.nextInt(8);
        int fromCol = random.nextInt(8);
        int toRow   = random.nextInt(8);
        int toCol   = random.nextInt(8);

        String piece = board[fromRow][fromCol];

        // Check if destination has opponent's king (simplified capture logic)
        String target = board[toRow][toCol];
        boolean capturesKing = (player == 0 && "k".equals(target))
                            || (player == 1 && "K".equals(target));

        if (capturesKing) {
            kingCaptured         = true;
            capturedKingByPlayer = player;
            System.out.printf("Chess: Player %d captures the KING at [%d,%d]!%n",
                    player + 1, toRow, toCol);
        }

        board[toRow][toCol]     = piece;
        board[fromRow][fromCol] = " ";

        System.out.printf("Chess: Player %d moves '%s' from [%d,%d] to [%d,%d]%n",
                player + 1, piece, fromRow, fromCol, toRow, toCol);
    }

    @Override
    protected boolean isOver() {
        return kingCaptured || moveCount >= MAX_MOVES;
    }

    @Override
    protected void printBoard() {
        System.out.println("  a b c d e f g h");
        for (int i = 0; i < 8; i++) {
            System.out.print((8 - i) + " ");
            for (int j = 0; j < 8; j++) {
                System.out.print(board[i][j] + " ");
            }
            System.out.println(8 - i);
        }
        System.out.println("  a b c d e f g h");
    }

    @Override
    protected boolean shouldPrintBoard() {
        return true; // chess always shows the board
    }

    @Override
    protected void onTurnEnd(int playerWhoMoved) {
        // Chess: check for check condition (simplified)
        System.out.println("Chess: evaluating board for check/checkmate...");
    }

    @Override
    protected void printWinner() {
        if (kingCaptured) {
            System.out.printf("Chess: CHECKMATE! Player %d wins!%n", capturedKingByPlayer + 1);
        } else {
            System.out.println("Chess: Draw by move limit.");
        }
    }

    @Override
    protected void onGameEnd() {
        System.out.println("Chess: saving game record to PGN format...");
        // Production: persist game history for replay
    }
}
```

```java
// CheckersGame.java
package com.course.patterns.templatemethod.game;

import java.util.Random;

/**
 * Checkers implementation. 8x8 board, pieces captured when jumped.
 * Demonstrates how a completely different game uses the SAME algorithm skeleton.
 */
public class CheckersGame extends AbstractGame {

    private int[] piecesOnBoard; // piecesOnBoard[0] = Player 1, piecesOnBoard[1] = Player 2
    private int turnLimit;
    private int moveCount;
    private final Random random;

    public CheckersGame(int turnLimit) {
        this.turnLimit = turnLimit;
        this.random    = new Random(7);
    }

    @Override
    public String getGameName() {
        return "Checkers";
    }

    @Override
    protected void initialize() {
        numberOfPlayers = 2;
        piecesOnBoard   = new int[]{12, 12}; // standard checkers: 12 pieces each
        moveCount       = 0;
        System.out.println("Checkers: board initialized. Each player has 12 pieces.");
    }

    @Override
    protected void makeMove(int player) {
        moveCount++;
        int opponent = 1 - player;

        // Simulate: 30% chance of capturing an opponent piece
        boolean captures = random.nextInt(10) < 3;

        if (captures && piecesOnBoard[opponent] > 0) {
            piecesOnBoard[opponent]--;
            System.out.printf("Checkers: Player %d jumps and captures! "
                    + "Player %d has %d pieces remaining.%n",
                    player + 1, opponent + 1, piecesOnBoard[opponent]);
        } else {
            System.out.printf("Checkers: Player %d makes a regular move.%n", player + 1);
        }
    }

    @Override
    protected boolean isOver() {
        return piecesOnBoard[0] == 0
            || piecesOnBoard[1] == 0
            || moveCount >= turnLimit;
    }

    // Hook: Checkers in headless mode does not print the board every turn
    @Override
    protected boolean shouldPrintBoard() {
        return false;
    }

    // Hook: after each turn, print piece counts
    @Override
    protected void onTurnEnd(int playerWhoMoved) {
        System.out.printf("  [Pieces: P1=%d, P2=%d]%n",
                piecesOnBoard[0], piecesOnBoard[1]);
    }

    @Override
    protected void printWinner() {
        if (piecesOnBoard[0] == 0) {
            System.out.println("Checkers: Player 2 wins! Player 1 has no pieces left.");
        } else if (piecesOnBoard[1] == 0) {
            System.out.println("Checkers: Player 1 wins! Player 2 has no pieces left.");
        } else {
            // Determine winner by piece count
            if (piecesOnBoard[0] > piecesOnBoard[1]) {
                System.out.printf("Checkers: Player 1 wins on piece count! (%d vs %d)%n",
                        piecesOnBoard[0], piecesOnBoard[1]);
            } else if (piecesOnBoard[1] > piecesOnBoard[0]) {
                System.out.printf("Checkers: Player 2 wins on piece count! (%d vs %d)%n",
                        piecesOnBoard[1], piecesOnBoard[0]);
            } else {
                System.out.println("Checkers: Draw! Both players have equal pieces.");
            }
        }
    }
}
```

```java
// GameDemo.java
package com.course.patterns.templatemethod.game;

public class GameDemo {

    public static void main(String[] args) {
        System.out.println("=".repeat(60));
        System.out.println("Template Method Pattern — Game Turn Sequence Demo");
        System.out.println("=".repeat(60));

        System.out.println("\n>>> CHESS <<<");
        AbstractGame chess = new ChessGame();
        chess.playGame();

        System.out.println("\n>>> CHECKERS <<<");
        AbstractGame checkers = new CheckersGame(8);
        checkers.playGame();

        System.out.println("\n" + "=".repeat(60));
        System.out.println("Both games used the SAME playGame() skeleton:");
        System.out.println("  initialize -> [printBoard -> makeMove -> onTurnEnd]* -> printWinner");
        System.out.println("=".repeat(60));
    }
}
```

---

## Implementation 3: Data Miner

This example is the "classic" GoF illustration — extracting data from heterogeneous sources.

```java
// DataMiner.java
package com.course.patterns.templatemethod.dataminer;

import java.util.List;
import java.util.Map;

/**
 * Template Method: defines the invariant data mining algorithm.
 * Variable steps (opening, extracting, closing) are delegated to subclasses.
 */
public abstract class DataMiner {

    // =========================================================================
    // TEMPLATE METHOD
    // =========================================================================

    public final MiningResult mine(String sourcePath) {
        System.out.println("[" + getSourceType() + "] Starting mining: " + sourcePath);

        openSource(sourcePath);

        RawData rawData     = extractData();
        ParsedData parsed   = parseData(rawData);       // concrete — shared
        Analysis analysis   = analyzeData(parsed);      // concrete — shared
        String report       = generateReport(analysis); // concrete — shared

        closeSource();

        System.out.println("[" + getSourceType() + "] Mining complete.");
        return new MiningResult(getSourceType(), sourcePath, analysis, report);
    }

    // =========================================================================
    // ABSTRACT METHODS
    // =========================================================================

    protected abstract void openSource(String path);
    protected abstract RawData extractData();
    protected abstract void closeSource();
    public abstract String getSourceType();

    // =========================================================================
    // CONCRETE (SHARED) METHODS — invariant logic
    // =========================================================================

    private ParsedData parseData(RawData raw) {
        System.out.println("[DataMiner] Parsing " + raw.lines().size() + " raw lines...");
        // Shared parsing: split lines, trim whitespace, remove empty lines
        List<String[]> parsedRows = raw.lines().stream()
                .filter(line -> line != null && !line.isBlank())
                .map(line -> line.trim().split("[,\t|]+"))
                .toList();
        return new ParsedData(parsedRows, raw.encoding());
    }

    private Analysis analyzeData(ParsedData parsed) {
        System.out.println("[DataMiner] Analyzing " + parsed.rows().size() + " parsed rows...");

        int totalRows   = parsed.rows().size();
        int totalCells  = parsed.rows().stream().mapToInt(r -> r.length).sum();
        double avgCols  = totalRows == 0 ? 0 : (double) totalCells / totalRows;

        // Find the most frequent value across all cells
        Map<String, Long> frequency = parsed.rows().stream()
                .flatMap(row -> java.util.Arrays.stream(row))
                .collect(java.util.stream.Collectors.groupingBy(
                        s -> s.trim(), java.util.stream.Collectors.counting()));

        String mostFrequent = frequency.entrySet().stream()
                .max(Map.Entry.comparingByValue())
                .map(Map.Entry::getKey)
                .orElse("N/A");

        return new Analysis(totalRows, avgCols, mostFrequent, frequency.size());
    }

    private String generateReport(Analysis analysis) {
        System.out.println("[DataMiner] Generating standardized report...");
        return String.format("""
                === Data Mining Report ===
                Total rows:        %d
                Avg columns/row:   %.2f
                Unique values:     %d
                Most frequent:     '%s'
                ==========================
                """,
                analysis.totalRows(), analysis.avgColumnsPerRow(),
                analysis.uniqueValueCount(), analysis.mostFrequentValue());
    }

    // =========================================================================
    // DATA RECORDS
    // =========================================================================

    public record RawData(List<String> lines, String encoding) {}
    public record ParsedData(List<String[]> rows, String encoding) {}
    public record Analysis(int totalRows, double avgColumnsPerRow,
                           String mostFrequentValue, int uniqueValueCount) {}
    public record MiningResult(String sourceType, String sourcePath,
                               Analysis analysis, String report) {}
}
```

```java
// PDFDataMiner.java
package com.course.patterns.templatemethod.dataminer;

import java.util.List;

public class PDFDataMiner extends DataMiner {

    private String pdfText; // extracted text content

    @Override
    public String getSourceType() { return "PDF"; }

    @Override
    protected void openSource(String path) {
        System.out.println("[PDF] Opening PDF file: " + path);
        // Production: PDDocument doc = PDDocument.load(new File(path));
    }

    @Override
    protected RawData extractData() {
        System.out.println("[PDF] Extracting text via PDFBox...");
        // Production: PDFTextStripper stripper = new PDFTextStripper(); ...
        // Simulate extracted text
        List<String> lines = List.of(
                "Invoice Number, Date, Amount, Status",
                "INV-001, 2024-01-15, 5000.00, Paid",
                "INV-002, 2024-01-22, 3200.00, Pending",
                "",                              // blank line — will be filtered
                "INV-003, 2024-02-03, 8750.00, Paid",
                "INV-004, 2024-02-14, 1100.00, Overdue"
        );
        return new RawData(lines, "UTF-8");
    }

    @Override
    protected void closeSource() {
        System.out.println("[PDF] Closing PDF document and releasing memory.");
        // Production: doc.close();
    }
}
```

```java
// CSVDataMiner.java
package com.course.patterns.templatemethod.dataminer;

import java.util.List;

public class CSVDataMiner extends DataMiner {

    @Override
    public String getSourceType() { return "CSV"; }

    @Override
    protected void openSource(String path) {
        System.out.println("[CSV] Opening CSV file with OpenCSV: " + path);
        // Production: CSVReader reader = new CSVReaderBuilder(new FileReader(path)).build();
    }

    @Override
    protected RawData extractData() {
        System.out.println("[CSV] Reading CSV rows...");
        List<String> lines = List.of(
                "employee_id|name|department|salary|years",
                "E001|Alice|Engineering|95000|5",
                "E002|Bob|Marketing|72000|3",
                "E003|Carol|Engineering|110000|8",
                "E004|David|Sales|68000|2",
                "E005|Emma|Engineering|105000|7"
        );
        return new RawData(lines, "UTF-8");
    }

    @Override
    protected void closeSource() {
        System.out.println("[CSV] Closing CSV reader.");
        // Production: reader.close();
    }
}
```

```java
// DatabaseDataMiner.java
package com.course.patterns.templatemethod.dataminer;

import java.util.ArrayList;
import java.util.List;

public class DatabaseDataMiner extends DataMiner {

    private final String jdbcUrl;

    public DatabaseDataMiner(String jdbcUrl) {
        this.jdbcUrl = jdbcUrl;
    }

    @Override
    public String getSourceType() { return "Database"; }

    @Override
    protected void openSource(String query) {
        System.out.println("[DB] Connecting to: " + jdbcUrl);
        System.out.println("[DB] Executing query: " + query);
        // Production: connection = DriverManager.getConnection(jdbcUrl, user, pass);
        //             statement  = connection.prepareStatement(query);
    }

    @Override
    protected RawData extractData() {
        System.out.println("[DB] Fetching ResultSet rows...");
        // Production: ResultSet rs = statement.executeQuery(); stream rows
        List<String> rows = new ArrayList<>();
        rows.add("product_id,product_name,category,price,stock");
        rows.add("P001,Widget Pro,Electronics,299.99,150");
        rows.add("P002,Widget Lite,Electronics,149.99,320");
        rows.add("P003,Desk Organizer,Office,49.99,800");
        rows.add("P004,Ergonomic Chair,Furniture,599.99,45");
        rows.add("P005,Laptop Stand,Electronics,79.99,200");
        return new RawData(rows, "UTF-8");
    }

    @Override
    protected void closeSource() {
        System.out.println("[DB] Closing ResultSet, Statement, and Connection.");
        // Production: rs.close(); statement.close(); connection.close();
    }
}
```

```java
// DataMinerDemo.java
package com.course.patterns.templatemethod.dataminer;

public class DataMinerDemo {

    public static void main(String[] args) {
        System.out.println("=".repeat(60));
        System.out.println("Template Method — Data Miner Demo");
        System.out.println("=".repeat(60));

        DataMiner pdfMiner = new PDFDataMiner();
        DataMiner.MiningResult pdfResult = pdfMiner.mine("/data/invoices_q1.pdf");
        System.out.println(pdfResult.report());

        DataMiner csvMiner = new CSVDataMiner();
        DataMiner.MiningResult csvResult = csvMiner.mine("/data/employees.csv");
        System.out.println(csvResult.report());

        DataMiner dbMiner = new DatabaseDataMiner("jdbc:postgresql://prod-db:5432/inventory");
        DataMiner.MiningResult dbResult = dbMiner.mine("SELECT * FROM products WHERE active = true");
        System.out.println(dbResult.report());

        System.out.println("=".repeat(60));
        System.out.println("All three miners shared: parseData(), analyzeData(), generateReport()");
        System.out.println("Each differed in: openSource(), extractData(), closeSource()");
    }
}
```

---

## Template Method vs Strategy Pattern

This is one of the most important comparisons in OOP design and a frequent interview topic.

### Core Distinction

| Dimension | Template Method | Strategy |
|-----------|----------------|----------|
| Mechanism | **Inheritance** | **Composition** |
| Variation | Subclassing — override steps | Encapsulate algorithm — swap objects |
| Algorithm selection | Compile-time (fixed by class) | Runtime (set via setter or constructor) |
| Who controls flow | Base class | Context class (delegates entirely) |
| Step granularity | Individual steps vary | Entire algorithm varies |
| Relationship | "is-a" (subtype) | "has-a" (delegate) |

### Template Method

```java
// Template Method: base class owns the skeleton, subclasses override parts
abstract class Sorter {
    public final void sort(int[] data) {
        preSort(data);        // hook
        doSort(data);         // abstract — you provide the algorithm
        postSort(data);       // hook
    }
    protected abstract void doSort(int[] data);
    protected void preSort(int[] data) {}
    protected void postSort(int[] data) {}
}

class QuickSorter extends Sorter {
    @Override
    protected void doSort(int[] data) {
        // QuickSort implementation
        System.out.println("QuickSort: " + data.length + " elements");
    }
}

class MergeSorter extends Sorter {
    @Override
    protected void doSort(int[] data) {
        // MergeSort implementation
        System.out.println("MergeSort: " + data.length + " elements");
    }
}

// Usage:
Sorter sorter = new QuickSorter();
sorter.sort(data);  // cannot change algorithm at runtime
```

### Strategy

```java
// Strategy: context holds a reference to a strategy object
interface SortStrategy {
    void sort(int[] data);
}

class QuickSortStrategy implements SortStrategy {
    @Override
    public void sort(int[] data) {
        System.out.println("QuickSort strategy: " + data.length + " elements");
    }
}

class MergeSortStrategy implements SortStrategy {
    @Override
    public void sort(int[] data) {
        System.out.println("MergeSort strategy: " + data.length + " elements");
    }
}

class DataProcessor {
    private SortStrategy sortStrategy;

    public DataProcessor(SortStrategy strategy) {
        this.sortStrategy = strategy;
    }

    public void setStrategy(SortStrategy strategy) {
        this.sortStrategy = strategy;
    }

    public void process(int[] data) {
        preProcess(data);
        sortStrategy.sort(data);  // delegates ENTIRE sorting to strategy
        postProcess(data);
    }

    private void preProcess(int[] data)  { System.out.println("Pre-processing..."); }
    private void postProcess(int[] data) { System.out.println("Post-processing..."); }
}

// Usage:
DataProcessor processor = new DataProcessor(new QuickSortStrategy());
processor.process(data);
processor.setStrategy(new MergeSortStrategy()); // swap at runtime!
processor.process(data);
```

### Deep Trade-off Analysis

**1. Coupling**

- *Template Method*: The concrete class is **tightly coupled** to the abstract base class. It inherits all state, all non-overridable methods, and all future changes to the base class. This is the "fragile base class" problem.
- *Strategy*: The context class is **loosely coupled** to the strategy interface. You can change strategies without touching the context. Strategy classes are independently testable.

**2. Granularity of variation**

- *Template Method*: Shines when only **a few specific steps** vary. The algorithm's overall structure is stable.
- *Strategy*: Better when the **entire behavior** varies, or when you want to compose multiple behaviors independently.

**3. State sharing**

- *Template Method*: Subclasses have direct access to the base class's `protected` state (fields, helper methods). Easy to share data between steps.
- *Strategy*: The context must **pass data to the strategy explicitly** via method parameters or a context object. More explicit but more verbose.

**4. Runtime flexibility**

- *Template Method*: **None**. The algorithm variant is fixed at object creation time (determined by which subclass you instantiate).
- *Strategy*: **Full**. You can swap strategies at runtime, A/B test different algorithms, or configure from external configuration.

**5. Code reuse**

- *Template Method*: Reuse through inheritance — common steps live in the base class automatically.
- *Strategy*: Reuse requires composition — you can share strategies across multiple contexts.

**6. Testing**

- *Template Method*: Harder to test individual steps in isolation. Testing a subclass requires the full template to run.
- *Strategy*: Each strategy is independently testable via its interface.

### When to Choose Which

Choose **Template Method** when:
- The algorithm skeleton is stable and well-understood
- Only a few steps vary, and variations are few
- You want compile-time enforcement that subclasses implement required steps
- Shared state between steps is significant (and you do not want to pass it around)
- You are building a framework where users extend by subclassing

Choose **Strategy** when:
- The entire algorithm or multiple independent behaviors vary
- You need to swap algorithms at runtime
- You want to combine behaviors (multiple strategies on one context)
- Testing strategies in isolation is important
- You want to avoid inheritance hierarchies

### The Inheritance vs Composition Debate

> "Favor composition over inheritance" — GoF, *Design Patterns*

Template Method is the pattern most often cited when discussing the limits of inheritance:

1. **Java does not support multiple inheritance**: If `PDFReport` already extends `BaseDocument`, it cannot also extend `AbstractReport`.

2. **Inheritance is a compile-time coupling**: You cannot change what you inherit. Composition lets you wire behaviors at runtime via injection.

3. **Template Method can be refactored to Strategy**: Extract each abstract method into its own strategy interface. This is a common evolutionary path as requirements grow.

```java
// Evolved: Template Method → Strategy (when runtime flexibility is needed)
class ReportGenerator {
    private final DataLoader loader;
    private final DataFormatter formatter;
    private final OutputWriter writer;

    public ReportGenerator(DataLoader loader, DataFormatter formatter, OutputWriter writer) {
        this.loader    = loader;
        this.formatter = formatter;
        this.writer    = writer;
    }

    public ReportResult generate() {
        ReportData data      = loader.load();
        byte[] formatted     = formatter.format(data);
        String location      = writer.write(formatted);
        return new ReportResult(location);
    }
}

// Now you can mix and match at runtime:
new ReportGenerator(
    new DatabaseLoader(config),
    new PDFFormatter(),
    new S3Writer(bucket)
);
```

This is more flexible but loses the advantages of Template Method: the invariant algorithm structure is no longer enforced, and shared state must be passed between components.

---

## Real-World Framework Examples

### 1. Java's AbstractList and AbstractMap

`java.util.AbstractList` is a textbook Template Method implementation. It provides default implementations of almost all `List` methods in terms of two abstract primitives:

```java
// From JDK source (simplified)
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {

    // Abstract primitives — you MUST implement
    public abstract E get(int index);
    public abstract int size();

    // Template methods built on top of those primitives
    public Iterator<E> iterator() { /* uses get() and size() */ }
    public ListIterator<E> listIterator(int index) { /* uses get() and size() */ }
    public List<E> subList(int from, int to) { /* uses get() and size() */ }
    public int indexOf(Object o) { /* uses get() and size() */ }
    public int lastIndexOf(Object o) { /* uses get() and size() */ }
    public boolean contains(Object o) { return indexOf(o) >= 0; }

    // Hook: override for mutation support
    public E set(int index, E element) {
        throw new UnsupportedOperationException();
    }
    public void add(int index, E element) {
        throw new UnsupportedOperationException();
    }
    public E remove(int index) {
        throw new UnsupportedOperationException();
    }
}

// Your read-only list — just implement two methods:
class RangeList extends AbstractList<Integer> {
    private final int start, end;
    public RangeList(int start, int end) { this.start = start; this.end = end; }

    @Override public Integer get(int index) { return start + index; }
    @Override public int size() { return end - start; }

    // iterator(), contains(), indexOf(), subList() all WORK FOR FREE
}
```

This is Template Method eliminating enormous code duplication across all Java collection implementations.

### 2. HttpServlet.service()

```java
// javax.servlet.http.HttpServlet (simplified)
public abstract class HttpServlet extends GenericServlet {

    // TEMPLATE METHOD — dispatches to the right handler
    protected void service(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        String method = req.getMethod();
        switch (method) {
            case "GET"    -> doGet(req, resp);
            case "POST"   -> doPost(req, resp);
            case "PUT"    -> doPut(req, resp);
            case "DELETE" -> doDelete(req, resp);
            case "HEAD"   -> doHead(req, resp);
            default       -> resp.sendError(HttpServletResponse.SC_NOT_IMPLEMENTED);
        }
    }

    // HOOK METHODS — default implementations send "405 Method Not Allowed"
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        resp.sendError(HttpServletResponse.SC_METHOD_NOT_ALLOWED);
    }

    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        resp.sendError(HttpServletResponse.SC_METHOD_NOT_ALLOWED);
    }
    // doHead, doPut, doDelete similarly
}

// Your servlet — just override what you support:
public class OrderServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        // HttpServlet calls this when GET arrives — Hollywood Principle
        String orderId = req.getParameter("id");
        resp.getWriter().println("Order: " + orderId);
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        // Called by HttpServlet on POST — you do NOT call service() yourself
        resp.getWriter().println("Order created");
    }
}
```

`HttpServlet.service()` is the template method. You never override `service()` — the Servlet container calls it, and it calls your `doGet()`/`doPost()`. This is the Hollywood Principle applied to HTTP request handling.

### 3. JUnit 5 Lifecycle Methods

JUnit's test lifecycle is a Template Method implemented by the framework:

```java
// JUnit 5 internally orchestrates this sequence (simplified):
class JUnit5TestRunner {

    // TEMPLATE METHOD — called by the JUnit engine for each test class
    void runTestClass(Class<?> testClass) throws Exception {
        Object instance = testClass.getDeclaredConstructor().newInstance();

        // @BeforeAll hooks (static)
        invokeStaticMethods(testClass, BeforeAll.class);

        for (Method testMethod : getTestMethods(testClass)) {
            // @BeforeEach hooks
            invokeMethods(instance, BeforeEach.class);

            // @Test — the actual test
            testMethod.invoke(instance);

            // @AfterEach hooks
            invokeMethods(instance, AfterEach.class);
        }

        // @AfterAll hooks (static)
        invokeStaticMethods(testClass, AfterAll.class);
    }
}

// Your test class is the "subclass" providing variable steps:
class OrderServiceTest {

    private OrderService orderService;
    private TestDatabase testDb;

    @BeforeAll
    static void startDatabase() {
        // JUnit calls this once before any test — hook
    }

    @BeforeEach
    void setUp() {
        testDb       = new TestDatabase();
        orderService = new OrderService(testDb);
        // JUnit calls this before each test — hook
    }

    @Test
    void shouldCreateOrder() {
        // The actual test — JUnit calls this
        Order order = orderService.create("ITEM-001", 5);
        assertNotNull(order.getId());
    }

    @AfterEach
    void tearDown() {
        testDb.clear();
        // JUnit calls this after each test — hook
    }

    @AfterAll
    static void stopDatabase() {
        // JUnit calls this once after all tests — hook
    }
}
```

You never call `runTestClass()`. JUnit calls your methods at the right points. Classic Hollywood Principle via Template Method.

### 4. Spring's JdbcTemplate

```java
// Spring's JdbcTemplate uses Template Method internally
// query() is a template method that manages connection lifecycle
public <T> List<T> query(String sql, RowMapper<T> rowMapper) {
    return execute(new QueryStatementCallback<>(sql, rowMapper));
}

// execute() is the actual template method:
private <T> T execute(StatementCallback<T> action) {
    Connection con = DataSourceUtils.getConnection(dataSource); // step 1
    Statement stmt = null;
    try {
        stmt = con.createStatement();  // step 2
        T result = action.doInStatement(stmt); // step 3 — YOUR code (RowMapper)
        handleWarnings(stmt);          // step 4
        return result;
    } catch (SQLException ex) {
        JdbcUtils.closeStatement(stmt);
        stmt = null;
        DataSourceUtils.releaseConnection(con, dataSource);
        throw translateException("StatementCallback", sql, ex);
    } finally {
        JdbcUtils.closeStatement(stmt);      // step 5
        DataSourceUtils.releaseConnection(con, dataSource); // step 6
    }
}

// Usage: you provide ONLY the RowMapper (the variable step)
List<Customer> customers = jdbcTemplate.query(
    "SELECT id, name, email FROM customers",
    (rs, rowNum) -> new Customer(
        rs.getLong("id"),
        rs.getString("name"),
        rs.getString("email")
    )
);
```

Note: Spring's `JdbcTemplate` achieves Template Method via *callback objects* (lambda / functional interface) rather than inheritance. This is a hybrid approach: the structure of Template Method (invariant algorithm in `execute()`, variable step via `action`) but implemented with composition rather than subclassing. It is more flexible and avoids the inheritance problems.

---

## Trade-offs

### Advantages

**1. Eliminates code duplication**
The invariant parts of an algorithm live in exactly one place. Bug fixes and improvements automatically benefit all subclasses.

**2. Enforces algorithm structure**
Declaring the template method `final` means no subclass can accidentally reorder, skip, or duplicate steps. The invariant is guaranteed by the type system.

**3. Hollywood Principle / IoC**
Framework code controls the flow; user code just provides the variable parts. This is a clean separation of concerns — frameworks are built on this principle.

**4. Open/Closed Principle compliance**
You can extend behavior (new subclass) without modifying the existing algorithm (base class).

**5. Compile-time safety**
Abstract methods give you compile-time enforcement that all required steps are implemented. Miss one and it does not compile.

**6. Simplified subclass implementation**
Subclasses only implement what they need to change. All the shared logic is inherited for free.

### Disadvantages

**1. Fragile base class problem**
Changes to the abstract class can break all subclasses silently. If you add a concrete method to the base class or change a `protected` method's signature, every subclass is affected.

**2. Liskov Substitution Principle risk**
Subclasses that override hooks in unexpected ways can violate LSP. If `shouldAddWatermark()` is supposed to be a lightweight flag check but a subclass makes it trigger a network call, the template method's behavior changes in unexpected ways.

**3. Limited runtime flexibility**
The algorithm variant is fixed at compile time. You cannot change it without instantiating a different subclass.

**4. Inheritance limitations**
Java does not support multiple inheritance. A class that must extend two different abstract classes cannot. This is a hard constraint.

**5. Inverted control is hard to understand**
New developers reading a subclass may not immediately understand that its methods are called by the base class, not by client code. The call chain is implicit.

**6. Proliferation of classes**
Each distinct variant requires a new subclass. If you have N sources × M formats, you need N × M subclasses (or careful use of hooks to reduce this).

**7. Protected members create coupling**
`protected` fields and methods in the base class are part of its API in the sense that subclasses depend on them. Changing them is a breaking change even though they are not `public`.

---

## Common Interview Discussion Points

### 1. The Inheritance vs Composition Debate

This is the most common discussion point when Template Method comes up in interviews.

**The standard argument against Template Method:**

> "Template Method uses inheritance, and inheritance is a compile-time coupling. Prefer composition over inheritance. Use Strategy instead."

**The nuanced response:**

Template Method is appropriate when:
- The relationship truly is "is-a" (a `PDFReport` truly IS a kind of `AbstractReport`)
- The algorithm skeleton is stable — unlikely to change
- The concrete class needs access to protected base class state (avoids passing it as parameters)
- You want compile-time enforcement of required steps (abstract methods > interfaces for this)
- You are building a framework (Spring, JUnit, Servlet API all use it correctly)

Strategy is better when:
- You need runtime algorithm swapping
- You want to compose multiple independent behaviors
- You are building application code (not framework code)
- The inheritance chain would grow deep

**Evolutionary path**: Start with Template Method when the structure is clear. Refactor toward composition (Strategy) when you see: (1) the base class becoming a "grab bag" of loosely related code, (2) many hooks being added to accommodate edge cases, or (3) a need for runtime flexibility.

### 2. Why Mark the Template Method `final`?

The template method should almost always be `final` for these reasons:

1. **Invariant protection**: The whole point is that the algorithm structure cannot change. Making it `final` enforces this by the compiler.
2. **Prevent accidental override**: A subclass author might override the template method thinking they are overriding a step, breaking the pattern entirely.
3. **LSP compliance**: If a subclass could override `generate()`, the contract that "calling `generate()` runs steps in order A, B, C, D" would be broken.

The only legitimate reason to not mark it `final` is if you want a subclass to have the ability to add pre/post logic around the algorithm — but this is usually better achieved with `beforeGenerate()` and `afterGenerate()` hooks.

### 3. Hook Method Design

**The "should vs must" principle**: If a step is truly optional or has a reasonable default, make it a hook. If it is mandatory and has no universal default, make it abstract.

**Naming conventions for hooks**:
- `should*()` — boolean hooks: `shouldAddWatermark()`, `shouldCache()`
- `before*()` / `after*()` — lifecycle hooks: `beforeProcess()`, `afterFormat()`
- `get*()` — configuration hooks: `getDelimiter()`, `getPageSize()`
- `on*()` — event hooks: `onSuccess()`, `onError()`

### 4. Template Method in Java 8+ (Default Methods)

Java 8 introduced default methods in interfaces, which allow a partial form of Template Method without inheritance:

```java
interface Reportable {

    // Template method in an interface
    default ReportResult generate() {
        ReportData data        = loadData();          // abstract
        byte[] formatted       = format(data);        // abstract
        String location        = output(formatted);   // abstract
        return new ReportResult(location);
    }

    // Abstract steps (interface methods are implicitly abstract)
    ReportData loadData();
    byte[] format(ReportData data);
    String output(byte[] bytes);
}
```

**Key difference from class-based Template Method**:
- No shared mutable state (no `protected` fields)
- No `final` on default methods — any implementing class can override `generate()`
- Multiple interface implementation is allowed (solves the single-inheritance problem)
- Cannot call `super.generate()` to extend (only to delegate)

This is a lightweight alternative, but it lacks the invariant enforcement of `final` and the ability to share state through `protected` fields.

---

## Interview Questions and Answers

### Q1: What is the Template Method pattern and what problem does it solve?

**Model Answer:**

The Template Method pattern defines the skeleton of an algorithm in a base class method, declaring some steps as abstract (subclasses must implement) and others as hooks (subclasses may optionally override). The template method itself is typically declared `final` to prevent subclasses from altering the algorithm's overall control flow.

**Problem it solves**: When multiple classes share the same algorithmic structure but differ in specific steps, you end up with either (a) massive code duplication if you implement the entire algorithm in each class, or (b) no enforcement of the algorithm's structure if you just use an interface.

Template Method resolves this by centralizing the invariant parts of the algorithm in the base class while creating well-defined extension points (abstract methods and hooks) for the parts that legitimately differ.

Real example: `HttpServlet.service()` dispatches to `doGet()`, `doPost()`, etc. Every servlet shares the dispatch algorithm; each servlet provides its own request handling.

---

### Q2: What is the Hollywood Principle and how does Template Method embody it?

**Model Answer:**

The Hollywood Principle states: "Don't call us, we'll call you." It is an inversion of control (IoC) guideline where high-level components (frameworks, base classes) control when and how low-level components (subclasses, plugins) are invoked.

Template Method embodies this because:
- The **base class (high-level)** defines the algorithm and controls the sequence of calls
- The **subclass (low-level)** provides implementations but never calls the template method itself
- The subclass "registers" behavior by overriding methods and then **waits to be called** by the base class

Without the Hollywood Principle (traditional):
```java
class PDFReport {
    void generate() {
        new DataLoader().load();   // calls upward — low-level controls flow
        new PDFFormatter().format();
        new FileWriter().write();
    }
}
```

With Hollywood Principle (Template Method):
```java
abstract class AbstractReport {
    final void generate() {     // HIGH-LEVEL controls flow
        loadData();             // calls DOWN to subclass
        format();               // calls DOWN to subclass
        output();               // calls DOWN to subclass
    }
}

class PDFReport extends AbstractReport {
    void loadData() { /* waits to be called, does not initiate */ }
    void format()   { /* waits to be called */ }
    void output()   { /* waits to be called */ }
}
```

This matters because it makes the invariant (steps run in this order) enforced by the type system rather than by convention.

---

### Q3: What is the difference between abstract methods and hook methods? When do you use each?

**Model Answer:**

**Abstract methods** are methods declared without implementation in the base class. Subclasses **must** provide an implementation, enforced at compile time. Use them for steps that are:
- Mandatory (every variant must do something here)
- Have no universal sensible default
- Represent the core "variable part" of the algorithm

**Hook methods** are methods with a default implementation in the base class (often a no-op or a simple boolean return). Subclasses **may** override them but are not required to. Use them for steps that are:
- Optional (some subclasses need them, others do not)
- Have a sensible default (e.g., "no pre-processing by default")
- Represent lifecycle events or configuration values
- Would force subclasses to write no-op implementations if abstract

Decision rule: "If forgetting to implement this method in a subclass would produce a bug or meaningless behavior, make it abstract. If the default behavior is valid for most cases, make it a hook."

Example:
```java
abstract class AbstractReport {
    protected abstract void loadData();   // mandatory — no universal default
    protected abstract byte[] format();   // mandatory — completely different per format

    protected void validate() {}          // hook — most subclasses don't need custom validation
    protected boolean shouldCache() { return false; } // hook — reasonable default
    protected void onComplete() {}        // hook — lifecycle, optional notification
}
```

---

### Q4: Template Method uses inheritance, which many consider a design smell. When is it justified?

**Model Answer:**

Inheritance is appropriate — not a smell — when the relationship is a true "is-a" relationship and the shared structure is stable. Template Method is justified when:

1. **You are building a framework, not application code.** Frameworks like Spring, JUnit, and the Servlet API use Template Method extensively and correctly. The framework owns the algorithm; users extend it. This is the canonical use case.

2. **The algorithm skeleton is stable.** If the sequence of steps rarely changes, inheritance is low-risk. The cost of inheritance (tight coupling) is worth paying when the base class API is stable.

3. **Most concrete classes genuinely share state and helper methods.** If subclasses need access to `protected` fields and utilities in the base class, inheritance naturally provides this without requiring explicit parameter passing.

4. **You want compile-time enforcement.** Abstract methods give you a compile error if a subclass forgets to implement a required step. This is stronger than interface default methods (which can be accidentally overridden in the template method) or documentation.

5. **Subclasses truly ARE subtypes.** `PDFReport` is a type of report. `ChessGame` is a type of game. The is-a relationship holds semantically, not just structurally.

When to switch to Strategy (composition):
- You need runtime algorithm swapping
- The inheritance hierarchy is growing wide or deep
- The base class is accumulating unrelated hooks to accommodate edge cases
- Subclasses need to extend multiple abstract classes (Java limitation)

The mature answer is: Template Method and Strategy are not mutually exclusive. Use Template Method for the outer structure, and use Strategy for individual steps that need extreme flexibility. Spring's `JdbcTemplate` does exactly this — it has a template method (`execute()`) that delegates the variable step to a `StatementCallback` (a Strategy).

---

### Q5: How does Template Method relate to the Open/Closed Principle?

**Model Answer:**

Template Method is a direct implementation of the Open/Closed Principle (OCP):

> "Software entities should be open for extension but closed for modification."

**Closed for modification**: The template method is `final`. The algorithm's structure — the sequence of steps, the invariant logic — cannot be changed without modifying the base class. Existing behavior is protected.

**Open for extension**: New report types, new game implementations, new data miners can be added by creating a new subclass. The base class does not need to change. The extension point is the abstract methods and hooks.

```java
// Closed (final = cannot modify structure):
public final void generate() {
    loadData();
    processData(); // concrete, shared
    format();
    output();
}

// Open (new variants via new subclasses):
class ExcelReport extends AbstractReport { /* adds Excel support without touching AbstractReport */ }
class ParquetReport extends AbstractReport { /* adds Parquet support without touching AbstractReport */ }
```

The caveat: if you need to add a new step to the algorithm (e.g., add a "compress" step between `format()` and `output()`), you must modify the base class. This is the inherent tension — the template method itself is not open for extension in terms of step addition. This is often acceptable because the algorithm skeleton is usually more stable than the implementations of individual steps.

---

### Q6: How would you refactor Template Method to Strategy, and why might you do that?

**Model Answer:**

**Trigger for refactoring**: You would refactor when you observe one or more of:
- The need to change the algorithm at runtime
- The inheritance hierarchy becoming unmanageable (too many subclasses)
- Multiple concrete classes differ in more than one step (combinatorial explosion)
- Subclasses are in a different module/jar and cannot extend the base class
- You want to unit test steps in isolation

**The Refactoring:**

Before (Template Method):
```java
abstract class DataExporter {
    public final void export(Dataset data) {
        connect();        // abstract
        transform(data);  // abstract
        write();          // abstract
        disconnect();     // abstract
    }
}
class JdbcExporter extends DataExporter { /* ... */ }
class S3Exporter   extends DataExporter { /* ... */ }
```

After (Strategy — step 1: extract each abstract method into an interface):
```java
interface Connector       { void connect(); void disconnect(); }
interface Transformer<T>  { T transform(Dataset data); }
interface Writer<T>       { void write(T data); }

class DataExporter {
    private final Connector connector;
    private final Transformer<byte[]> transformer;
    private final Writer<byte[]> writer;

    public DataExporter(Connector connector,
                        Transformer<byte[]> transformer,
                        Writer<byte[]> writer) {
        this.connector   = connector;
        this.transformer = transformer;
        this.writer      = writer;
    }

    public void export(Dataset data) {
        connector.connect();
        byte[] transformed = transformer.transform(data);
        writer.write(transformed);
        connector.disconnect();
    }
}

// Now mix and match:
DataExporter exporter = new DataExporter(
    new JdbcConnector(url),
    new CSVTransformer(),
    new LocalFileWriter(path)
);
// Or at runtime:
exporter = new DataExporter(
    new S3Connector(bucket),
    new ParquetTransformer(),
    new CloudWriter(endpoint)
);
```

**What you gain**: Runtime flexibility, independent testability, no inheritance constraint.
**What you lose**: The invariant algorithm structure (`final` template method), compile-time enforcement of required steps, easy state sharing between steps.

---

### Q7: Explain the Template Method pattern as it appears in Java's AbstractList.

**Model Answer:**

`java.util.AbstractList<E>` is a textbook Template Method implementation. It provides a skeletal implementation of the `List` interface built on exactly two abstract primitive operations:

```java
public abstract E get(int index);
public abstract int size();
```

Everything else — `iterator()`, `listIterator()`, `contains()`, `indexOf()`, `lastIndexOf()`, `subList()`, `toArray()`, etc. — is implemented in `AbstractList` using only `get()` and `size()`. These are the template methods built on the abstract primitives.

Hook methods for mutation:
- `set(int index, E element)` — hook: throws `UnsupportedOperationException` by default
- `add(int index, E element)` — hook: throws `UnsupportedOperationException` by default
- `remove(int index)` — hook: throws `UnsupportedOperationException` by default

Override `get()` and `size()` → you get a fully functional read-only list (iterator, contains, indexOf, toArray all work for free).
Override `set()` additionally → you get a mutable list.
Override `add()` and `remove()` additionally → you get a fully mutable, resizable list.

This is why `ArrayList`, `LinkedList`, `Arrays.asList()` result, and `Collections.unmodifiableList()` result all work with the same `List` interface — they all extend from this abstract skeleton and only implement what they need to vary.

The benefit: if Sun/Oracle discovers a performance improvement in `indexOf()`, all implementations (ArrayList, LinkedList, custom lists) benefit automatically from the fix in `AbstractList`.

---

### Q8: What is the "fragile base class" problem and how does it apply to Template Method?

**Model Answer:**

The fragile base class problem occurs when seemingly safe changes to a base class break subclasses, often in ways that are not detectable at compile time.

**How it manifests in Template Method:**

Scenario: You have `AbstractReport` with a template method and 20 subclasses deployed in production.

```java
abstract class AbstractReport {
    public final void generate() {
        loadData();
        processData(); // currently: sort by first column
        format();
        output();
    }
}
```

You change `processData()` to sort by a different column because a new requirement says "sort by date column". This is a private method, so it seems safe. But now all 20 subclasses produce output in a different order. The subclasses did not change, but their behavior changed.

**More insidious scenario**: You add a new step to the template method:

```java
public final void generate() {
    loadData();
    processData();
    compress(); // NEW STEP — added for storage efficiency
    format();
    output();
}

protected void compress() {
    // default: gzip compression
    formattedOutput = gzip(formattedOutput);
}
```

All existing subclasses now get gzip compression. If their downstream consumers did not expect compressed output, they silently break. The compiler gives no warning.

**Mitigation strategies:**

1. **Comprehensive tests for all subclasses**: Catch behavioral changes before deployment.
2. **Stable base class API**: Treat `protected` methods as public API once subclasses exist.
3. **Semantic versioning**: If you must change the base class in a breaking way, bump the major version.
4. **Prefer hooks with no-op defaults for new steps**: Instead of adding to the main flow, add an `afterFormat()` hook that defaults to a no-op, so existing subclasses are unaffected.
5. **Consider composition over inheritance**: Strategy pattern avoids this entirely because the base class does not exist.

The fragile base class problem is the primary reason the GoF recommendation to "favor composition over inheritance" was made. Template Method is not wrong — it is the right tool in the right context — but you must be deliberate about base class stability when subclasses are in external code or widespread use.

---

## Comparison with Related Patterns

### Template Method vs Factory Method

Both use inheritance and the Hollywood Principle.

| | Template Method | Factory Method |
|---|---|---|
| Purpose | Define algorithm skeleton | Defer object creation to subclass |
| What varies | Steps of an algorithm | Which object gets created |
| Template method | `generate()`, `mine()`, `playGame()` | `createProduct()` |
| Variable step | `loadData()`, `format()`, `output()` | The factory method itself |

Factory Method can be seen as a degenerate Template Method where the algorithm has only one variable step: creating an object.

### Template Method vs Strategy

Covered in depth above. Key summary:
- Template Method: **inheritance**, compile-time, parts of algorithm vary
- Strategy: **composition**, runtime, entire algorithm varies

### Template Method vs Builder

Builder assembles complex objects step by step. Template Method defines a step-by-step algorithm.

| | Template Method | Builder |
|---|---|---|
| Goal | Execute an algorithm | Construct an object |
| Output | Side effects (report written, game played) | A complex object |
| Pattern | Behavioral | Creational |
| Variable part | How each step is performed | Which optional parts to include |

### Template Method vs Iterator

Iterator abstracts traversal (how to move through a collection). Template Method abstracts an algorithm (how to execute a sequence of steps).

Both use the Hollywood Principle: the Iterator's `hasNext()`/`next()` are called by the consumer; the Template Method's abstract steps are called by the base class.

### Template Method in the Context of Behavioral Patterns

| Pattern | Core intent | Key mechanism |
|---------|-------------|---------------|
| Template Method | Invariant algorithm skeleton | Inheritance, abstract steps |
| Strategy | Interchangeable algorithms | Composition, interface delegation |
| Command | Encapsulate a request as an object | Object that carries an action |
| Observer | Notify many dependents of a state change | Event + listener registration |
| State | Behavior changes with object's state | State object composition |
| Iterator | Traverse a collection without exposing structure | Encapsulated cursor |

Template Method sits at the foundation of the behavioral patterns — it is one of the simplest structural ideas (override a method in a subclass) elevated to a named, intentional pattern by recognizing that an algorithm skeleton in the base class with variable steps in subclasses is a recurring, powerful design.

---

*Course: System Design & LLD Crash Course | Module: Design Patterns — Behavioral | Pattern 6 of 11*
