# Interface Segregation Principle (ISP)

> "Clients should not be forced to depend upon interfaces that they do not use."
> — Robert C. Martin, *Agile Software Development: Principles, Patterns, and Practices*

ISP is the fourth of the five SOLID principles. It operates at the boundary between the definition of an interface and the code that depends on it. Violating ISP does not always produce a compile error — it often produces something more insidious: tight coupling, forced stub methods, and fragile recompilation chains that grow silently until a team is afraid to touch a core interface.

---

## Table of Contents

1. [Understanding the Definition](#1-understanding-the-definition)
2. [What "Depend Upon" Actually Means](#2-what-depend-upon-actually-means)
3. [The Fat Interface Problem](#3-the-fat-interface-problem)
4. [Role Interfaces vs Header Interfaces](#4-role-interfaces-vs-header-interfaces)
5. [Before and After: The Multi-Function Device](#5-before-and-after-the-multi-function-device)
6. [ISP and Composition](#6-isp-and-composition)
7. [The Stale Dependency Problem and Recompilation Cascades](#7-the-stale-dependency-problem-and-recompilation-cascades)
8. [ISP in Java: Default Methods as a Migration Tool](#8-isp-in-java-default-methods-as-a-migration-tool)
9. [ISP and Cohesion: The Parallel with SRP](#9-isp-and-cohesion-the-parallel-with-srp)
10. [ISP in Real Systems](#10-isp-in-real-systems)
11. [How ISP and SRP Reinforce Each Other](#11-how-isp-and-srp-reinforce-each-other)
12. [Interview Questions and Model Answers](#12-interview-questions-and-model-answers)
13. [Assignment: Split the UserService Fat Interface](#13-assignment-split-the-userservice-fat-interface)

---

## 1. Understanding the Definition

Robert Martin's original formulation is precise: "clients should not be forced to depend upon interfaces that they do not use."

Every word carries weight.

**"Clients"** refers to the code that uses an interface — the callers, the service classes, the test doubles, the modules that import the type. Not the implementors. The principle is written from the perspective of the consumer, not the producer.

**"Forced"** means there is no escape. The dependency is structural. The client cannot call one method on an interface without taking on a compile-time or link-time dependency on all the others. If the interface declares ten methods and you only ever call two, you are still forced to depend on all ten — the bytecode reference to the interface type drags in the whole contract.

**"Depend upon interfaces they do not use"** means the client's source file, class file, or deployment unit carries a reference to methods and contracts that are irrelevant to its purpose. This is the coupling ISP targets.

A useful restatement: **split your interfaces so that no implementor is ever forced to write a method it cannot implement, and no caller is ever forced to import methods it will never invoke.**

---

## 2. What "Depend Upon" Actually Means

Dependency in Java is not just a runtime concept. It has three distinct manifestations, and ISP addresses all three.

### 2.1 Compilation Dependency

When a class implements an interface, it must provide a concrete body for every method declared in that interface. If a method is irrelevant to the implementing class, the class is nonetheless forced to write something — even if that something is an empty body or an exception throw. The compiler enforces this.

```java
// The interface declares five methods
public interface IMachine {
    void print(Document doc);
    void scan(Document doc);
    void fax(Document doc);
    void copy(Document doc);
    void staple(Document doc);
}

// The implementing class has a compilation dependency on all five methods.
// It cannot compile without providing bodies for scan, fax, copy, and staple,
// even though it has no ability to perform those operations.
public class OldFashionedPrinter implements IMachine {
    @Override
    public void print(Document doc) {
        // real implementation
    }

    @Override
    public void scan(Document doc) {
        throw new UnsupportedOperationException("This printer cannot scan");
    }

    @Override
    public void fax(Document doc) {
        throw new UnsupportedOperationException("This printer cannot fax");
    }

    @Override
    public void copy(Document doc) {
        throw new UnsupportedOperationException("This printer cannot copy");
    }

    @Override
    public void staple(Document doc) {
        throw new UnsupportedOperationException("This printer cannot staple");
    }
}
```

The `OldFashionedPrinter.java` source file has a compile-time dependency on the concepts of scanning, faxing, copying, and stapling — none of which it implements. Any change to the signature of `scan()` forces a recompile of `OldFashionedPrinter`, even though scanning is irrelevant to it.

### 2.2 Coupling

When a module holds a reference to a fat interface, it is coupled to every method in that interface. If any method's signature changes — even a method the module never calls — the module must be updated or recompiled. This is indirect coupling mediated by the interface boundary, and it is exactly what ISP is designed to prevent.

### 2.3 Forced Stub Methods

The most visible symptom of an ISP violation is the forced stub: a method that exists only because the interface requires it, contributing nothing to the implementing class's purpose. There are three common stub patterns, in ascending order of harm:

```java
// Pattern 1: Empty body — silently does nothing, callers get no feedback
@Override
public void scan(Document doc) { }

// Pattern 2: UnsupportedOperationException — fails at runtime, not compile time
// Callers cannot know this will fail without reading the source or catching exceptions
@Override
public void scan(Document doc) {
    throw new UnsupportedOperationException("scan not supported");
}

// Pattern 3: Return null / default value — misleads callers into thinking
// a valid result was produced
@Override
public Document scan() {
    return null; // caller has no way to distinguish "null result" from "not supported"
}
```

All three patterns are design failures. They mean the contract encoded in the interface is a lie: the interface promises behavior that the implementor cannot deliver.

---

## 3. The Fat Interface Problem

A fat interface is an interface that has grown to cover too many concerns. Fat interfaces arise from a common design impulse: "let me put everything this component can do into one place." The result is an interface that is comprehensive but unusable without dragging in concerns that most clients do not need.

### How Fat Interfaces Grow

Fat interfaces rarely start fat. They accumulate methods over time as features are added. The cycle typically looks like this:

1. An interface is defined with three sensible methods.
2. A new feature requires a fourth method. Since the interface is already used everywhere, the new method is added to it.
3. Implementors that do not support the new method add a stub.
4. Clients that do not need the new method are nonetheless recompiled because the interface changed.
5. The cycle repeats for a fifth method, a sixth, and so on.

After two years, an interface that started with three methods has twenty-two, and every implementor has eight to fourteen stubs, and every client is coupled to the entire surface whether it uses it or not.

### Symptoms of a Fat Interface

The following patterns are reliable indicators of a fat interface violation:

1. **Stub methods in implementors.** If two or more concrete classes that implement the same interface each throw `UnsupportedOperationException` on different subsets of methods, the interface is fat.

2. **Unrelated method groups.** If you can partition the interface's methods into two groups where no single caller ever needs both groups, the interface should be two interfaces.

3. **Changelog correlation.** If the interface's git log shows commits from five different teams or five different feature areas, it is serving too many masters.

4. **Test doubles that stub half the interface.** When a test for `PrintingService` must implement `scan()`, `fax()`, `copy()`, and `staple()` as empty stubs just to pass a `IMachine` reference, the interface is demanding irrelevant overhead from every consumer, including tests.

```java
// Test class forced to implement irrelevant methods
public class FakeMachine implements IMachine {
    private final List<Document> printedDocs = new ArrayList<>();

    @Override
    public void print(Document doc) {
        printedDocs.add(doc); // the only method the test actually cares about
    }

    // Every one of these is forced noise — the test for PrintingService
    // has no interest in scanning or faxing
    @Override public void scan(Document doc) { }
    @Override public void fax(Document doc)  { }
    @Override public void copy(Document doc) { }
    @Override public void staple(Document doc) { }

    public List<Document> getPrintedDocs() {
        return Collections.unmodifiableList(printedDocs);
    }
}
```

The test file for `PrintingService` has become a maintenance liability. Every time `IMachine` grows a new method, `FakeMachine` must be updated — even when the new method is irrelevant to printing.

---

## 4. Role Interfaces vs Header Interfaces

Martin Fowler introduced the distinction between role interfaces and header interfaces. It is the clearest conceptual tool for applying ISP correctly.

### Header Interface

A header interface is an interface that mirrors the public API of a single concrete class. It contains one method for every public method on the class. The name often ends in `Impl`: the concrete class is `UserServiceImpl`, the interface is `IUserService` and contains exactly the methods `UserServiceImpl` provides.

This pattern is a cargo-cult abstraction. It creates the syntactic form of an interface without the semantic value. Because the interface is defined from the implementor's perspective — "what can this class do?" — rather than from the client's perspective — "what does this caller need?" — it forces every client to depend on the full surface.

```java
// Header interface — mirrors every public method of UserServiceImpl
// This is an anti-pattern
public interface IUserService {
    User findById(long id);
    User findByEmail(String email);
    void save(User user);
    void delete(long id);
    void updatePassword(long userId, String newPassword);
    void sendPasswordResetEmail(String email);
    boolean verifyEmail(String token);
    List<User> findByRole(Role role);
    void assignRole(long userId, Role role);
    void revokeRole(long userId, Role role);
    UserProfile getProfile(long userId);
    void updateProfile(long userId, UserProfile profile);
    void setNotificationPreferences(long userId, NotificationPreferences prefs);
    NotificationPreferences getNotificationPreferences(long userId);
    void lockAccount(long userId);
    void unlockAccount(long userId);
    AuditLog getAuditLog(long userId);
}
```

A `ReportingService` that needs only `findByRole()` is forced to depend on `sendPasswordResetEmail()`, `lockAccount()`, and `getAuditLog()`. These methods have nothing to do with reporting. The header interface has created a false coupling.

### Role Interface

A role interface is defined from the client's point of view. The question asked is not "what does the implementing class do?" but "what does this particular caller need?" The interface is named for the role it plays in a specific interaction.

```java
// Role interfaces — each named from the client's perspective

// What the reporting module needs
public interface UserQueryPort {
    User findById(long id);
    User findByEmail(String email);
    List<User> findByRole(Role role);
}

// What the authentication module needs
public interface UserCredentialPort {
    void updatePassword(long userId, String newPassword);
    void sendPasswordResetEmail(String email);
    boolean verifyEmail(String token);
}

// What the admin module needs
public interface UserAdminPort {
    void lockAccount(long userId);
    void unlockAccount(long userId);
    void assignRole(long userId, Role role);
    void revokeRole(long userId, Role role);
    AuditLog getAuditLog(long userId);
}

// What the profile module needs
public interface UserProfilePort {
    UserProfile getProfile(long userId);
    void updateProfile(long userId, UserProfile profile);
}

// What the notification module needs
public interface UserNotificationPort {
    void setNotificationPreferences(long userId, NotificationPreferences prefs);
    NotificationPreferences getNotificationPreferences(long userId);
}
```

The concrete `UserServiceImpl` implements all five interfaces. Each caller depends only on the interface relevant to its role. `ReportingService` receives a `UserQueryPort` and has zero coupling to password reset, account locking, or notification preferences.

The key shift: **interfaces are owned by their clients, not their implementors.**

---

## 5. Before and After: The Multi-Function Device

This is the canonical example for ISP. It is concrete enough to reason about but abstract enough to generalize.

### The Domain

Modern offices use multi-function devices (MFDs) that can print, scan, fax, copy, and staple. Basic offices may have only a simple printer. The system must model both, and client code must be able to drive each device independently.

---

### BEFORE: Fat Interface

```java
// Document.java
public class Document {
    private final String content;
    private final String name;

    public Document(String name, String content) {
        this.name = name;
        this.content = content;
    }

    public String getName()    { return name; }
    public String getContent() { return content; }
}
```

```java
// IMachine.java — the fat interface
public interface IMachine {
    void print(Document doc);
    void scan(Document doc);
    void fax(Document doc);
    void copy(Document doc);
    void staple(Document doc);
}
```

```java
// MultiFunctionDevice.java — legitimate implementor; uses all methods
public class MultiFunctionDevice implements IMachine {

    @Override
    public void print(Document doc) {
        System.out.println("[MFD] Printing: " + doc.getName());
    }

    @Override
    public void scan(Document doc) {
        System.out.println("[MFD] Scanning: " + doc.getName());
    }

    @Override
    public void fax(Document doc) {
        System.out.println("[MFD] Faxing: " + doc.getName());
    }

    @Override
    public void copy(Document doc) {
        System.out.println("[MFD] Copying: " + doc.getName());
    }

    @Override
    public void staple(Document doc) {
        System.out.println("[MFD] Stapling: " + doc.getName());
    }
}
```

```java
// OldFashionedPrinter.java — ISP violation; implements IMachine but supports only print()
public class OldFashionedPrinter implements IMachine {

    @Override
    public void print(Document doc) {
        System.out.println("[Printer] Printing: " + doc.getName());
    }

    // Forced stubs — this printer has no scanning hardware
    @Override
    public void scan(Document doc) {
        throw new UnsupportedOperationException("OldFashionedPrinter cannot scan");
    }

    @Override
    public void fax(Document doc) {
        throw new UnsupportedOperationException("OldFashionedPrinter cannot fax");
    }

    @Override
    public void copy(Document doc) {
        throw new UnsupportedOperationException("OldFashionedPrinter cannot copy");
    }

    @Override
    public void staple(Document doc) {
        throw new UnsupportedOperationException("OldFashionedPrinter cannot staple");
    }
}
```

```java
// PrintingService.java — client that only needs printing
// But it is forced to depend on the full IMachine interface
public class PrintingService {
    private final IMachine machine;

    public PrintingService(IMachine machine) {
        this.machine = machine;
    }

    public void printBatch(List<Document> documents) {
        // This service only ever calls print() — yet it depends on
        // the scan, fax, copy, and staple contracts too
        for (Document doc : documents) {
            machine.print(doc);
        }
    }
}
```

```java
// Main.java — runtime failure is invisible until it happens
public class Main {
    public static void main(String[] args) {
        Document report = new Document("Q4 Report", "...");

        IMachine mfd     = new MultiFunctionDevice();
        IMachine printer = new OldFashionedPrinter();

        PrintingService svc = new PrintingService(printer);
        svc.printBatch(List.of(report)); // works fine

        // But if something calls scan() on OldFashionedPrinter:
        printer.scan(report); // UnsupportedOperationException at runtime
    }
}
```

**Problems with this design:**

1. `OldFashionedPrinter` has four methods that are lies — they promise behavior the class cannot deliver.
2. `PrintingService` carries a compile-time and runtime dependency on scanning, faxing, copying, and stapling — none of which it ever uses.
3. If the `scan()` signature changes (e.g., to return a `ScannedImage`), `OldFashionedPrinter` must be updated even though it cannot scan.
4. Tests for `PrintingService` must provide stubs for four irrelevant methods.
5. The LSP is violated: `OldFashionedPrinter` cannot be substituted for `IMachine` in contexts that call `scan()`.

---

### AFTER: Segregated Interfaces

```java
// Document.java — unchanged
public class Document {
    private final String content;
    private final String name;

    public Document(String name, String content) {
        this.name = name;
        this.content = content;
    }

    public String getName()    { return name; }
    public String getContent() { return content; }
}
```

```java
// IPrinter.java — role interface for printing clients
public interface IPrinter {
    void print(Document doc);
}
```

```java
// IScanner.java — role interface for scanning clients
public interface IScanner {
    void scan(Document doc);
}
```

```java
// IFax.java — role interface for fax clients
public interface IFax {
    void fax(Document doc);
}
```

```java
// ICopier.java — role interface for copy clients
public interface ICopier {
    void copy(Document doc);
}
```

```java
// IStapler.java — role interface for staple clients
public interface IStapler {
    void staple(Document doc);
}
```

```java
// SimplePrinter.java — implements only what it can actually do
public class SimplePrinter implements IPrinter {

    @Override
    public void print(Document doc) {
        System.out.println("[SimplePrinter] Printing: " + doc.getName());
    }
    // No stubs. No UnsupportedOperationException. No lies.
}
```

```java
// MultiFunctionDevice.java — implements all interfaces it supports
public class MultiFunctionDevice implements IPrinter, IScanner, IFax, ICopier, IStapler {

    @Override
    public void print(Document doc) {
        System.out.println("[MFD] Printing: " + doc.getName());
    }

    @Override
    public void scan(Document doc) {
        System.out.println("[MFD] Scanning: " + doc.getName());
    }

    @Override
    public void fax(Document doc) {
        System.out.println("[MFD] Faxing: " + doc.getName());
    }

    @Override
    public void copy(Document doc) {
        System.out.println("[MFD] Copying: " + doc.getName());
    }

    @Override
    public void staple(Document doc) {
        System.out.println("[MFD] Stapling: " + doc.getName());
    }
}
```

```java
// PrintingService.java — depends only on IPrinter
// Completely isolated from scanning, faxing, copying, stapling
public class PrintingService {
    private final IPrinter printer;

    public PrintingService(IPrinter printer) {
        this.printer = printer;
    }

    public void printBatch(List<Document> documents) {
        for (Document doc : documents) {
            printer.print(doc);
        }
    }
}
```

```java
// ScanningService.java — depends only on IScanner
public class ScanningService {
    private final IScanner scanner;

    public ScanningService(IScanner scanner) {
        this.scanner = scanner;
    }

    public void scanAll(List<Document> documents) {
        for (Document doc : documents) {
            scanner.scan(doc);
        }
    }
}
```

```java
// DocumentWorkflowService.java — a service that needs print AND scan
// Depends on both role interfaces independently; no forced coupling to fax/copy/staple
public class DocumentWorkflowService {
    private final IPrinter printer;
    private final IScanner scanner;

    public DocumentWorkflowService(IPrinter printer, IScanner scanner) {
        this.printer = printer;
        this.scanner = scanner;
    }

    public void scanThenPrint(Document doc) {
        scanner.scan(doc);
        printer.print(doc);
    }
}
```

```java
// Main.java — wiring the segregated design
public class Main {
    public static void main(String[] args) {
        Document report  = new Document("Q4 Report", "...");
        Document contract = new Document("Service Contract", "...");

        // SimplePrinter is a valid IPrinter — type system enforces this
        IPrinter simplePrinter = new SimplePrinter();

        // MFD can play any role
        MultiFunctionDevice mfd = new MultiFunctionDevice();

        // PrintingService receives an IPrinter — works with either device
        PrintingService printSvc = new PrintingService(simplePrinter);
        printSvc.printBatch(List.of(report, contract));

        // ScanningService receives an IScanner — MFD is the only candidate
        // SimplePrinter cannot be passed here — the type system prevents it
        ScanningService scanSvc = new ScanningService(mfd);
        scanSvc.scanAll(List.of(report));

        // Workflow service — MFD satisfies both interfaces
        DocumentWorkflowService workflow =
            new DocumentWorkflowService(mfd, mfd);
        workflow.scanThenPrint(contract);
    }
}
```

**What changed and why it matters:**

| Before (fat interface) | After (segregated interfaces) |
|---|---|
| `OldFashionedPrinter` has 4 stub methods | `SimplePrinter` has zero stubs |
| `PrintingService` depends on `IMachine` (5 methods) | `PrintingService` depends on `IPrinter` (1 method) |
| Passing a `SimplePrinter` where `IScanner` is needed compiles | Passing a `SimplePrinter` where `IScanner` is needed is a compile error |
| Changing `scan()` signature forces recompile of `PrintingService` | Changing `scan()` signature does not touch `PrintingService` |
| Test double for `PrintingService` needs 4 irrelevant stubs | Test double for `PrintingService` implements 1 method |

The **type system now encodes the capabilities of each device**. It is impossible to write code that calls `scan()` on a `SimplePrinter` because `SimplePrinter` does not implement `IScanner`. The compiler catches capability mismatches at build time, not at runtime.

---

## 6. ISP and Composition

ISP naturally leads to composition. When interfaces are small and focused, a class can implement multiple interfaces and thereby compose multiple capabilities. This is composition at the type level.

```java
// A fax-printer combination device — not an MFD, but more than a simple printer
// Composed by implementing exactly the capabilities it has
public class FaxPrinter implements IPrinter, IFax {

    @Override
    public void print(Document doc) {
        System.out.println("[FaxPrinter] Printing: " + doc.getName());
    }

    @Override
    public void fax(Document doc) {
        System.out.println("[FaxPrinter] Faxing: " + doc.getName());
    }
    // No scan, copy, or staple — because this device does not have those capabilities.
    // No stubs. No lies.
}
```

Contrast this with inheritance. If `FaxPrinter` had extended a base class `AbstractMachine` that provided default (empty or throwing) implementations of all five operations, the inheritance hierarchy would have encoded a false capability claim at the class level. Composition through ISP-compliant interfaces avoids this.

The relationship between ISP and composition is directional: **ISP does not require composition, but ISP-compliant design makes composition the natural outcome.** When you define a role interface per capability, you get a vocabulary of capabilities that classes compose freely, without inheriting unwanted behavior.

### Delegating to Composed Parts

ISP also integrates cleanly with the delegation pattern, where a class implements multiple interfaces by delegating each to a dedicated internal component:

```java
// A high-end MFD that delegates each capability to specialized hardware components
public class EnterpriseMultiFunctionDevice
        implements IPrinter, IScanner, IFax, ICopier, IStapler {

    private final LaserPrintEngine   printEngine;
    private final FlatbedScanModule  scanModule;
    private final FaxModem           faxModem;
    private final DuplexCopyEngine   copyEngine;
    private final ElectricStapler    stapler;

    public EnterpriseMultiFunctionDevice(
            LaserPrintEngine printEngine,
            FlatbedScanModule scanModule,
            FaxModem faxModem,
            DuplexCopyEngine copyEngine,
            ElectricStapler stapler) {
        this.printEngine = printEngine;
        this.scanModule  = scanModule;
        this.faxModem    = faxModem;
        this.copyEngine  = copyEngine;
        this.stapler     = stapler;
    }

    @Override
    public void print(Document doc)  { printEngine.execute(doc); }

    @Override
    public void scan(Document doc)   { scanModule.capture(doc); }

    @Override
    public void fax(Document doc)    { faxModem.transmit(doc); }

    @Override
    public void copy(Document doc)   { copyEngine.duplicate(doc); }

    @Override
    public void staple(Document doc) { stapler.bind(doc); }
}
```

Each capability is fully implemented by a specialized component. The `EnterpriseMultiFunctionDevice` is an assembler of roles, not an implementor of everything. The segregated interfaces made this design obvious.

---

## 7. The Stale Dependency Problem and Recompilation Cascades

The practical cost of ISP violations in large systems is not just messy code — it is build time, deployment coupling, and risk amplification.

### The Stale Dependency Problem

When a fat interface is shared across many modules, any change to the interface touches every module that imports it, even if those modules have no interest in the changed method.

```
IMachine (fat interface)
    |
    +-- PrintingService (uses print() only)
    +-- ScanningService (uses scan() only)
    +-- FaxService      (uses fax() only)
    +-- AuditService    (uses all methods for logging)
    +-- ReportService   (uses scan() only for document input)
    +-- ArchiveService  (uses print() and copy() for archival)
```

If the signature of `fax()` changes to include a destination phone number:

```java
// Before
void fax(Document doc);

// After
void fax(Document doc, String destinationNumber);
```

Every class in the dependency graph must be recompiled: `PrintingService`, `ScanningService`, `AuditService`, `ReportService`, `ArchiveService` — even though none of them call `fax()`. In a large codebase, this can mean hundreds of files recompiled, all tests re-run, and the entire module deployed just because a fax signature changed.

### With Segregated Interfaces

```
IPrinter         IScanner         IFax
    |                |               |
PrintingService  ScanningService  FaxService
ArchiveService   ReportService
```

When `IFax.fax()` signature changes, only `FaxService` and any concrete class implementing `IFax` must change. `PrintingService`, `ScanningService`, `ArchiveService`, and `ReportService` are completely unaffected. Their interface contracts did not change. They do not recompile. They do not redeploy.

At scale — in a microservices architecture where shared interface libraries are published as JAR artifacts — the difference between a fat interface and segregated interfaces is the difference between a breaking change that forces all consumers to update simultaneously and a breaking change that is invisible to unrelated consumers.

### The Risk Amplification Effect

Beyond recompilation, fat interfaces amplify the risk of change. When you modify a method signature on a fat interface, you must audit every implementor and every caller of that interface to ensure correctness. The blast radius is the entire interface. With segregated interfaces, the blast radius is exactly the set of classes that depend on the changed method — nothing more.

---

## 8. ISP in Java: Default Methods as a Migration Tool

Java 8 introduced default methods in interfaces, which provide a way to add new methods to an existing interface without breaking all existing implementors. This is a migration tool — it has legitimate uses, but it is frequently misapplied as a way to avoid interface segregation.

### Legitimate Use: Adding a Method Without Breaking Implementors

```java
public interface IScanner {
    void scan(Document doc);

    // Added in v2 — provides a convenience method with a default implementation
    // Existing implementors do not need to override this
    default void scanAll(List<Document> documents) {
        for (Document doc : documents) {
            scan(doc);
        }
    }
}
```

This is legitimate: the default implementation is a genuine convenience built on the existing contract. Implementors that want a more efficient batch scan can override it; implementors that don't can accept the default.

### Misuse: Using Default Methods to Avoid Splitting a Fat Interface

```java
// Anti-pattern: using default methods to paper over an ISP violation
public interface IMachine {
    void print(Document doc);

    // "Added convenience" — but OldFashionedPrinter still doesn't support scanning
    // The default implementation is a meaningless no-op
    default void scan(Document doc) {
        // do nothing — "free" method for implementations that can't scan
    }

    default void fax(Document doc) {
        throw new UnsupportedOperationException("fax not supported by default");
    }

    default void copy(Document doc) { }

    default void staple(Document doc) { }
}
```

This does not fix the ISP violation — it hides it. `OldFashionedPrinter` still depends on the `scan`, `fax`, `copy`, and `staple` contracts. The default implementations mask the stubs without resolving the fundamental problem: `PrintingService` is still coupled to the full surface of `IMachine`, and adding a new method to `IMachine` still forces a recompile of every implementor and caller.

### The Test for Legitimate Default Method Usage

A default method is appropriate when:

1. The default implementation is non-trivial and correct for most implementors.
2. The method belongs semantically to the same contract as the existing methods.
3. Implementors can choose to override for performance or correctness reasons.

A default method is being misused when:

1. The default body is empty or throws an exception.
2. The method is being added to avoid splitting an interface that has grown too large.
3. More than one or two implementors will want to override it to provide genuine behavior.

The presence of default methods in an interface with many methods that throw `UnsupportedOperationException` by default is a strong signal that the interface should be split.

---

## 9. ISP and Cohesion: The Parallel with SRP

ISP at the interface level is structurally identical to SRP at the class level. Understanding this parallel gives you a single mental model for both principles.

### SRP: A Class Should Have One Reason to Change

SRP says that a class should have one cohesive responsibility — one reason why the business would ask you to modify it. A class that handles both persistence and formatting and notification will change when the database changes, when the email template changes, and when the message queue changes. Three reasons to change means three axes of coupling.

### ISP: An Interface Should Have One Client Role

ISP says that an interface should serve one cohesive client role. An interface that has methods for querying, updating, administering, and notifying will force every caller to depend on all four concerns. Four concerns in one interface means four axes of coupling between implementors and callers.

### The Cohesion Test for Interfaces

A well-cohesive interface passes this test: **every method in the interface is used by every client that depends on the interface.** If you can find even one method that no particular client ever calls, the interface has at least two distinct cohesion groups and should be split.

```java
// Low cohesion interface — can identify two distinct caller groups
public interface IOrderService {
    // Group A: customer-facing operations
    Order placeOrder(Cart cart, Customer customer);
    OrderStatus getStatus(String orderId);
    void cancelOrder(String orderId);

    // Group B: operations team administration
    List<Order> findStuckOrders();
    void forceComplete(String orderId);
    void rerouteToWarehouse(String orderId, String warehouseId);
    Map<String, Integer> getDailyOrderVolume(LocalDate date);
}
```

No customer-facing service will ever call `findStuckOrders()` or `rerouteToWarehouse()`. No operations tool will typically call `placeOrder()`. This interface has two distinct cohesion groups:

```java
// High cohesion — each interface serves exactly one caller role

public interface IOrderPlacementService {
    Order placeOrder(Cart cart, Customer customer);
    OrderStatus getStatus(String orderId);
    void cancelOrder(String orderId);
}

public interface IOrderAdministrationService {
    List<Order> findStuckOrders();
    void forceComplete(String orderId);
    void rerouteToWarehouse(String orderId, String warehouseId);
    Map<String, Integer> getDailyOrderVolume(LocalDate date);
}
```

### The Cohesion Spectrum

| Interface cohesion level | Characteristics | ISP assessment |
|---|---|---|
| Coincidental | Methods grouped only because they are in the same class | Strong ISP violation |
| Logical | Methods grouped by general category (all "user" operations) | Likely ISP violation |
| Temporal | Methods called at the same time, not for the same reason | ISP violation |
| Functional | Methods that together accomplish a single well-defined function | ISP compliant |
| Sequential | Methods where output of one feeds input of another, for one purpose | ISP compliant |

Aim for functional or sequential cohesion in every interface.

---

## 10. ISP in Real Systems

ISP is not an academic concern. The most maintainable production systems you encounter will have interfaces that are narrow, role-specific, and named after the client's intent.

### Repository Interfaces: Read vs Write

The repository pattern is the most common place where ISP is applied or violated in enterprise Java.

```java
// Fat repository — all callers depend on all operations
// A read-only reporting service is forced to depend on save() and delete()
public interface OrderRepository {
    Optional<Order> findById(String orderId);
    List<Order> findByCustomerId(long customerId);
    List<Order> findByStatus(OrderStatus status);
    List<Order> findAll(Pageable pageable);
    Order save(Order order);
    void delete(String orderId);
    void deleteAll(List<String> orderIds);
    long countByStatus(OrderStatus status);
}
```

```java
// Segregated repository interfaces

public interface OrderReadRepository {
    Optional<Order> findById(String orderId);
    List<Order> findByCustomerId(long customerId);
    List<Order> findByStatus(OrderStatus status);
    List<Order> findAll(Pageable pageable);
    long countByStatus(OrderStatus status);
}

public interface OrderWriteRepository {
    Order save(Order order);
    void delete(String orderId);
    void deleteAll(List<String> orderIds);
}
```

```java
// The concrete repository implements both
public class JpaOrderRepository implements OrderReadRepository, OrderWriteRepository {

    private final OrderJpaRepository jpa; // Spring Data JPA generated repository

    public JpaOrderRepository(OrderJpaRepository jpa) {
        this.jpa = jpa;
    }

    @Override
    public Optional<Order> findById(String orderId) {
        return jpa.findById(orderId).map(OrderMapper::toDomain);
    }

    @Override
    public List<Order> findByCustomerId(long customerId) {
        return jpa.findByCustomerId(customerId).stream()
            .map(OrderMapper::toDomain)
            .collect(Collectors.toList());
    }

    @Override
    public List<Order> findByStatus(OrderStatus status) {
        return jpa.findByStatus(status).stream()
            .map(OrderMapper::toDomain)
            .collect(Collectors.toList());
    }

    @Override
    public List<Order> findAll(Pageable pageable) {
        return jpa.findAll(pageable).stream()
            .map(OrderMapper::toDomain)
            .collect(Collectors.toList());
    }

    @Override
    public long countByStatus(OrderStatus status) {
        return jpa.countByStatus(status);
    }

    @Override
    public Order save(Order order) {
        OrderEntity entity = OrderMapper.toEntity(order);
        return OrderMapper.toDomain(jpa.save(entity));
    }

    @Override
    public void delete(String orderId) {
        jpa.deleteById(orderId);
    }

    @Override
    public void deleteAll(List<String> orderIds) {
        jpa.deleteAllById(orderIds);
    }
}
```

```java
// Reporting service — depends only on reads
// Cannot accidentally call save() or delete() — compile-time protection
public class OrderReportingService {
    private final OrderReadRepository repository;

    public OrderReportingService(OrderReadRepository repository) {
        this.repository = repository;
    }

    public OrderSummaryReport generateSummary(OrderStatus status) {
        long count = repository.countByStatus(status);
        List<Order> orders = repository.findByStatus(status);
        return new OrderSummaryReport(status, count, orders);
    }
}
```

```java
// Order placement service — depends only on writes
public class OrderPlacementService {
    private final OrderWriteRepository repository;
    private final OrderReadRepository  readRepository;

    public OrderPlacementService(OrderWriteRepository repository,
                                 OrderReadRepository readRepository) {
        this.repository     = repository;
        this.readRepository = readRepository;
    }

    public Order placeOrder(Cart cart, Customer customer) {
        Order order = Order.from(cart, customer);
        return repository.save(order);
    }
}
```

The CQRS (Command Query Responsibility Segregation) pattern is, at its core, an application of ISP at the service and repository layer. Every CQRS implementation segregates read and write interfaces.

### Service Interfaces Split by Consumer

In a microservices architecture, a service often has different API surfaces exposed to different consumers. ISP applies at the service interface level:

```java
// Internal service interface — full capability surface for internal callers
public interface PaymentServiceInternal {
    PaymentResult charge(PaymentRequest request);
    void refund(String paymentId, Money amount);
    void partialRefund(String paymentId, Money amount);
    PaymentStatus getStatus(String paymentId);
    List<Payment> getHistory(long customerId, DateRange range);
    void flagForFraudReview(String paymentId, String reason);
    FraudScore computeFraudScore(PaymentRequest request);
    void applyFee(String paymentId, Fee fee);
}

// External webhook interface — only what the payment gateway callback needs
public interface PaymentGatewayCallbackHandler {
    void onPaymentConfirmed(String gatewayPaymentId, Money amount);
    void onPaymentFailed(String gatewayPaymentId, String reason);
    void onRefundProcessed(String gatewayRefundId, Money amount);
}

// Customer-facing interface — only what the frontend API needs
public interface PaymentServiceCustomerFacing {
    PaymentResult charge(PaymentRequest request);
    PaymentStatus getStatus(String paymentId);
    List<Payment> getHistory(long customerId, DateRange range);
}
```

The concrete `PaymentServiceImpl` implements all three interfaces. The payment gateway callback handler receives a `PaymentGatewayCallbackHandler` — it cannot accidentally call `flagForFraudReview()` or `applyFee()`, and it has no coupling to those methods.

---

## 11. How ISP and SRP Reinforce Each Other

ISP and SRP address coupling from different angles, but they are mutually reinforcing. Applying one correctly tends to push you toward the other.

### SRP Drives ISP

When SRP is applied correctly, each class has a single well-defined responsibility. A class with a single responsibility tends to have a focused interface. When clients depend on that focused interface, they depend only on methods relevant to the one responsibility — ISP is satisfied as a consequence.

Conversely, when SRP is violated — when a class has many responsibilities — its interface tends to grow fat. Each responsibility adds methods, and soon the interface serves many different client roles simultaneously. Splitting the class (SRP) naturally suggests splitting the interface (ISP).

### ISP Drives SRP

When ISP is applied correctly and interfaces are split by client role, each interface defines a specific responsibility boundary. A concrete class that implements multiple narrow interfaces is, in effect, offering multiple distinct capabilities. If those capabilities grow complex, SRP suggests extracting each capability into its own class. The ISP-driven interface split is the diagnostic tool that reveals the SRP violation.

### The Practical Relationship

```
ISP violation discovered:
    OldFashionedPrinter implements IMachine
    but throws for scan, fax, copy, staple
         |
         v
ISP fix: split IMachine into IPrinter, IScanner, IFax, ICopier, IStapler
         |
         v
ISP fix reveals SRP question:
    Should MultiFunctionDevice be ONE class implementing five interfaces?
    Or should it be an assembler that delegates to five specialized components?
         |
         v
SRP applied: each hardware capability becomes its own class
    PrintEngine, ScanModule, FaxModem, CopyEngine, StaplerUnit
    MultiFunctionDevice delegates to them
```

The two principles are not the same — SRP is about classes, ISP is about interfaces — but they share the same underlying metric: **cohesion**. High cohesion in a class (SRP) and high cohesion in an interface (ISP) both minimize coupling and maximize the ability to change one thing without touching everything else.

---

## 12. Interview Questions and Model Answers

### Q1. What is the Interface Segregation Principle in your own words?

**Model answer:**

ISP says that you should not force a client to depend on methods it does not use. In practice, this means breaking large interfaces into smaller, role-specific ones so that each caller only sees the slice of behavior it actually needs. The canonical example is a multi-function device: a simple printer should not be forced to implement scan, fax, and copy just because a fat `IMachine` interface declares them. Instead, `IPrinter`, `IScanner`, and `IFax` should be separate, and a simple printer implements only `IPrinter`.

The principle is written from the client's perspective, not the implementor's. The question to ask is not "what can this class do?" but "what does this caller need?" Interfaces should be owned by the clients that use them.

---

### Q2. What is a "fat interface" and why is it harmful?

**Model answer:**

A fat interface is an interface with too many methods, typically because it has grown to cover multiple distinct client roles or because it was defined to mirror all public methods of a single implementing class (a header interface).

The harms are concrete:

1. **Forced stub methods.** Implementors that cannot support some methods must write empty bodies or throw `UnsupportedOperationException`, making the contract a lie and turning compile-time safety into runtime guesswork.

2. **Unnecessary coupling.** A `PrintingService` that depends on `IMachine` is coupled to `scan()`, `fax()`, `copy()`, and `staple()` — methods it never calls. Any change to those method signatures forces a recompile of `PrintingService`.

3. **Recompilation cascades.** In modular systems, changing one method on a fat interface can force recompilation and redeployment of dozens of modules that have no interest in the changed method.

4. **Brittle test doubles.** Every test that creates a mock or fake of the fat interface must provide bodies for all irrelevant methods, making tests harder to write and more fragile to interface changes.

---

### Q3. What is the difference between a role interface and a header interface?

**Model answer:**

A header interface is defined from the implementor's perspective: one interface per class, mirroring all public methods of that class. The question it answers is "what can this class do?" Header interfaces create the syntactic form of abstraction without the semantic benefit — every client still depends on the full surface.

A role interface is defined from the client's perspective: one interface per client role or use case. The question it answers is "what does this caller need?" Role interfaces are narrow, cohesive, and named after their purpose from the consumer's point of view — `UserQueryPort`, `IPrinter`, `OrderReadRepository`.

The key insight is that **interfaces are owned by their clients, not their implementors.** When you define interfaces from the client's perspective, you get role interfaces naturally. When you define interfaces from the implementor's perspective, you get header interfaces.

---

### Q4. Can an ISP violation also be an LSP violation? Explain.

**Model answer:**

Yes, and they co-occur regularly. The classic example is `OldFashionedPrinter implements IMachine` where `OldFashionedPrinter` throws `UnsupportedOperationException` on `scan()`.

LSP says that a subtype must be substitutable for its supertype without altering the correctness of the program. If code receives an `IMachine` and calls `scan()`, it expects the operation to succeed. `OldFashionedPrinter` breaks this expectation by throwing an exception — that is an LSP violation.

ISP says that implementors should not be forced to implement methods they cannot support. `OldFashionedPrinter` being forced to implement `scan()` is an ISP violation.

The ISP violation is the root cause; the LSP violation is the symptom. When you fix the ISP violation by defining `IScanner` separately and having `OldFashionedPrinter` not implement it, the LSP violation disappears: there is now no way to pass `OldFashionedPrinter` where an `IScanner` is expected, because the type system prevents it.

This is why ISP and LSP are often described as complementary: fixing ISP structurally prevents LSP violations caused by capability mismatches.

---

### Q5. How do Java 8 default methods relate to ISP? Are they a solution?

**Model answer:**

Default methods in Java 8+ allow adding new methods to an interface with a default implementation, so existing implementors do not need to be updated. They are a useful evolution mechanism, not an ISP solution.

A default method is appropriate when: the default implementation is correct and non-trivial for most implementors, and the method belongs to the same semantic contract as the existing interface.

Default methods are a misuse of ISP when: the default body is empty or throws an exception (effectively a hidden forced stub), or they are added to avoid splitting a fat interface. Adding empty defaults to `IMachine` for `scan()`, `fax()`, and `copy()` does not fix the ISP violation — it merely hides the forced stub. `PrintingService` is still coupled to the scanning, faxing, and copying contracts. Changing those method signatures still forces recompilation. Test doubles still implicitly depend on the full interface shape.

The correct ISP fix is structural: split the interface into role interfaces. Default methods are a migration aid for stable APIs, not a substitute for interface design.

---

### Q6. In a microservices system, how would you apply ISP to a shared library?

**Model answer:**

In a shared library scenario, a fat shared interface is particularly dangerous because a signature change forces all consuming services to update their dependencies simultaneously, creating a coordinated release requirement — the "distributed monolith" problem.

Applying ISP means defining the interfaces from each consuming service's perspective. If `ServiceA` needs order read operations and `ServiceB` needs order write operations, publish two interfaces in the shared library: `OrderReadPort` and `OrderWritePort`. Neither service depends on the other's interface.

Further, consider publishing separate Maven/Gradle artifacts per interface group. If `OrderReadPort` and `OrderWritePort` are in the same artifact, a change to `OrderWritePort` still forces `ServiceA` to pick up a new dependency version even if the read interface did not change. ISP applied at the artifact level means: one artifact per client role, or at least grouped such that a change to one role's interface does not force updates to unrelated consumers.

This is the principle behind API versioning and interface stability contracts in service meshes: narrow, role-specific interfaces evolve independently.

---

### Q7. A colleague says: "ISP just means use lots of small interfaces. That's over-engineering." How do you respond?

**Model answer:**

The concern is legitimate — excessive interface proliferation for its own sake is over-engineering. But ISP is not a prescription to create interfaces for every possible grouping of methods. It is a prescription to split interfaces when doing so removes coupling that would otherwise force changes on unrelated code.

The test is concrete: do different callers of this interface use different subsets of its methods? If every caller uses every method, the interface is already cohesive and should not be split. If you can identify two callers where one never calls a method the other always calls, you have evidence of a fat interface.

The cost of unnecessary splitting is real: more types, more files, more cognitive overhead. The cost of a fat interface is also real: recompilation cascades, forced stubs, LSP violations, and brittle tests. ISP says to make the trade-off in favor of splitting when there is a concrete caller who is harmed by the coupling. It does not say to split preemptively on the basis of hypothetical future callers.

---

### Q8. How does ISP affect testability?

**Model answer:**

ISP significantly improves testability by reducing the size of test doubles. When a test for `PrintingService` needs to provide a mock or stub of `IMachine`, it must implement all five methods — even if only `print()` is relevant to the test. This creates noise and maintenance burden.

With `IPrinter` as a role interface, the test double implements exactly one method. The test is smaller, more focused, and immune to changes in the scanning, faxing, and copying contracts.

More importantly, ISP makes dependency injection natural. Narrow interfaces are easy to inject because the consumer's constructor makes its dependencies explicit and minimal. A constructor that takes `IPrinter printer, IScanner scanner` clearly communicates what the class needs. A constructor that takes `IMachine machine` obscures which capabilities are actually used.

In test-driven development, ISP-compliant interfaces emerge naturally: when you write the test first, you inject only the capabilities the class under test needs, which drives you toward narrow interfaces.

---

### Q9. What is the relationship between ISP and the Command Query Responsibility Segregation (CQRS) pattern?

**Model answer:**

CQRS is, in substantial part, an application of ISP at the service and data access layer. The core of CQRS is the separation of read (query) and write (command) operations into distinct models, interfaces, and often distinct classes.

This maps directly to ISP's prescription: the read-side consumers (reporting services, query handlers, UI data fetchers) should not be forced to depend on write operations (`save()`, `delete()`, `update()`). The write-side consumers (command handlers, workflow orchestrators) should not depend on complex query operations.

The `OrderReadRepository` / `OrderWriteRepository` split shown earlier is CQRS applied at the repository layer via ISP. At the service layer, separate `IOrderQueryService` and `IOrderCommandService` interfaces serve the same role. CQRS frameworks like Axon Framework enforce this separation architecturally, but the underlying principle is ISP: clients depend only on the operations they need.

---

### Q10. How would you identify an ISP violation in a code review?

**Model answer:**

I look for five signals, in order of reliability:

1. **Stub methods with `UnsupportedOperationException`.** Any `@Override` method that throws `UnsupportedOperationException` or contains only an empty body is an immediate flag. The question is: is this method declared in an interface? If yes, the interface is making a promise the class cannot keep.

2. **Client code that uses only a small subset of interface methods.** I look at classes that receive an interface as a constructor or method parameter and check which methods they actually call. If a class receives `IOrderService` but only calls `getStatus()`, the interface is doing more than this client needs.

3. **Test doubles with many irrelevant stubs.** In test files, classes that implement an interface by providing empty or throwing bodies for most methods are a dead giveaway. The test code reveals what the production code actually needs versus what the interface forces it to provide.

4. **Interface evolution history.** An interface whose git log shows commits from multiple unrelated features or teams is likely serving multiple masters and should be examined for segregation.

5. **Interface names that are nouns or broad verbs.** `IService`, `IManager`, `IHandler` are warning signs. Role interfaces tend to have names like `OrderQueryPort`, `IPrinter`, `NotificationSender` — names that describe a specific action from a specific client's perspective.

When I find one of these signals, I look for the set of callers, map which methods each caller uses, and identify the natural partition into role-specific interfaces.

---

## 13. Assignment: Split the UserService Fat Interface

### The Problem

The following `UserService` interface has grown over two years to cover four distinct concerns: authentication, profile management, notification preferences, and admin operations. It is a textbook fat interface.

```java
// UserService.java — the fat interface
// Do NOT modify this file. Read it and design the split.
public interface UserService {

    // --- Authentication operations ---
    User authenticate(String email, String password) throws AuthenticationException;
    void changePassword(long userId, String oldPassword, String newPassword);
    void sendPasswordResetEmail(String email);
    boolean resetPassword(String resetToken, String newPassword);
    boolean verifyEmailAddress(String verificationToken);
    void invalidateAllSessions(long userId);
    boolean isSessionValid(String sessionToken);

    // --- Profile management operations ---
    UserProfile getProfile(long userId);
    void updateProfile(long userId, ProfileUpdateRequest request);
    void uploadProfilePhoto(long userId, byte[] photoBytes, String mimeType);
    void deleteProfilePhoto(long userId);
    String getProfilePhotoUrl(long userId);
    User findById(long userId);
    User findByEmail(String email);
    List<User> searchUsers(UserSearchCriteria criteria);

    // --- Notification preference operations ---
    NotificationPreferences getNotificationPreferences(long userId);
    void updateNotificationPreferences(long userId, NotificationPreferences prefs);
    void subscribeToNewsletter(long userId, String newsletterCode);
    void unsubscribeFromNewsletter(long userId, String newsletterCode);
    List<String> getActiveSubscriptions(long userId);

    // --- Admin operations ---
    void lockAccount(long userId, String reason);
    void unlockAccount(long userId);
    void assignRole(long userId, Role role);
    void revokeRole(long userId, Role role);
    List<Role> getRoles(long userId);
    void deleteUser(long userId);
    UserAuditLog getAuditLog(long userId);
    List<User> findUsersLockedSince(LocalDateTime since);
    void bulkAssignRole(List<Long> userIds, Role role);
}
```

### Supporting Types (Already Implemented — Do Not Change)

```java
public class User { /* id, email, name, roles, accountStatus */ }
public class UserProfile { /* bio, displayName, location, website */ }
public class ProfileUpdateRequest { /* fields the user can update */ }
public class NotificationPreferences { /* email, sms, push flags and frequencies */ }
public class UserAuditLog { /* list of audit entries with timestamp and actor */ }
public class UserSearchCriteria { /* filter fields: role, status, dateRange */ }
public enum Role { USER, MODERATOR, ADMIN, SUPER_ADMIN }
public class AuthenticationException extends Exception { /* ... */ }
```

### What You Must Produce

**Part 1: Interface Segregation**

Define four or more focused interfaces that together cover the complete surface of `UserService`. Each interface must:

- Be named from the perspective of the client that uses it (role interface, not header interface).
- Contain only methods that belong to a single coherent concern.
- Have a Javadoc comment explaining which system component is the intended primary client.

**Part 2: Concrete Implementation**

Write `UserServiceImpl` that implements all your interfaces. You do not need to write real logic — stubs with `System.out.println` or `// TODO: implement` are fine. What matters is that:

- The class declaration shows exactly which interfaces it implements.
- There are zero `UnsupportedOperationException` throws.
- Every method declared in the original `UserService` is accounted for.

**Part 3: Client Classes**

Write three client classes that consume your interfaces:

1. `LoginController` — the REST controller handling login, logout, and password reset. Inject only the interface it needs.
2. `NotificationPreferenceService` — the service that manages a user's notification subscriptions and preferences. Inject only the interface it needs.
3. `UserAdminController` — the internal admin console controller. Inject only the interface it needs.

Each client class must:

- Receive its dependencies via constructor injection.
- Declare constructor parameters using only the narrow interface type, not `UserServiceImpl` directly.
- Show at least two method calls to demonstrate the dependency is being used.

**Part 4: Reflection (Written)**

Write a short paragraph (150-250 words) answering: Which methods were hardest to assign to a single interface, and why? Were there any methods that could plausibly belong to two different interfaces? How did you resolve that ambiguity?

---

### Evaluation Criteria

| Criterion | Weight | Description |
|---|---|---|
| Interface cohesion | 30% | Each interface covers exactly one client role; no method is in the wrong interface |
| Naming clarity | 15% | Interface names communicate the client role, not the implementor's capabilities |
| Zero stubs | 15% | `UserServiceImpl` has no empty bodies or `UnsupportedOperationException` throws |
| Client correctness | 20% | Each client injects the narrowest interface that satisfies its needs |
| Coverage | 10% | Every method from the original `UserService` appears in exactly one new interface |
| Written reflection | 10% | Demonstrates reasoning about boundary decisions, not just a list of choices |

---

### Hints (Read Only After a Genuine Attempt)

<details>
<summary>Hint 1: Interface count</summary>

Four interfaces is the most natural split: one for authentication, one for profile management, one for notification preferences, one for admin. Some engineers prefer five by further splitting profile management into read (query) and write (mutation) operations. Both are defensible — the key question is whether you have a concrete client that needs reads but not writes.

</details>

<details>
<summary>Hint 2: Ambiguous methods</summary>

`findById` and `findByEmail` belong naturally to profile management (the profile module needs to look up users), but admin operations also need to look up users. Consider whether a `UserQueryPort` is the right shared interface, or whether these methods should exist in the profile interface and the admin controller should also depend on the profile interface. The former is more ISP-pure; the latter may be more practical.

</details>

<details>
<summary>Hint 3: Client injection</summary>

`LoginController` needs authentication operations. `NotificationPreferenceService` needs notification preference operations. `UserAdminController` is the broadest — it may legitimately need two interfaces (admin operations and user query), and that is fine. A class can depend on more than one narrow interface.

</details>

---

## Summary

| Concept | Key takeaway |
|---|---|
| Definition | Clients must not be forced to depend on interface methods they do not use |
| Fat interface | One large interface forcing all implementors and callers to depend on the full surface |
| Header interface | Defined from the implementor's perspective — an anti-pattern |
| Role interface | Defined from the client's perspective — the ISP-compliant pattern |
| Forced stubs | Empty bodies or `UnsupportedOperationException` are signals of ISP and LSP violations |
| Recompilation cascade | Changing one method on a fat interface forces recompile of all unrelated dependents |
| Default methods | A migration tool, not an ISP solution; misuse hides violations without fixing them |
| ISP and SRP | Same underlying metric (cohesion) applied at different levels (interfaces vs classes) |
| ISP and composition | Small interfaces naturally encourage implementing classes to compose multiple capabilities |
| Repository example | `ReadRepository` / `WriteRepository` is ISP applied to the data access layer |
| CQRS connection | CQRS is ISP applied architecturally to the service and data model layer |

---

*Module: SOLID Principles | Lesson 4 of 5 | Next: Dependency Inversion Principle (DIP)*
