# Module 01: SOLID Principles

**Target audience:** Software engineers with 2-8 years of experience preparing for Senior Engineer roles and interviews.

**Prerequisites:** Module 00 complete. You can comfortably write a class diagram, explain each OOP pillar with precision, and apply composition over inheritance.

**Duration:** 1.5 weeks (Weeks 2-3) | ~15 hours

---

## Table of Contents

1. [What SOLID Is — and What It Is Not](#1-what-solid-is--and-what-it-is-not)
2. [Historical Context](#2-historical-context)
3. [The Cost of Ignorance: A Class That Violates All Five Principles](#3-the-cost-of-ignorance-a-class-that-violates-all-five-principles)
4. [The Five Principles — Core Summaries](#4-the-five-principles--core-summaries)
   - 4.1 [Single Responsibility Principle (SRP)](#41-single-responsibility-principle-srp)
   - 4.2 [Open/Closed Principle (OCP)](#42-openclosed-principle-ocp)
   - 4.3 [Liskov Substitution Principle (LSP)](#43-liskov-substitution-principle-lsp)
   - 4.4 [Interface Segregation Principle (ISP)](#44-interface-segregation-principle-isp)
   - 4.5 [Dependency Inversion Principle (DIP)](#45-dependency-inversion-principle-dip)
5. [How the Five Principles Interrelate](#5-how-the-five-principles-interrelate)
6. [Common Interview Traps](#6-common-interview-traps)
7. [Navigation: Principle Files](#7-navigation-principle-files)
8. [How to Study This Module](#8-how-to-study-this-module)
9. [Prerequisites Checklist](#9-prerequisites-checklist)

---

## 1. What SOLID Is — and What It Is Not

SOLID is a set of five object-oriented design principles that describe properties a class or module should have to remain understandable, changeable, and testable as a system grows. The acronym was coined by Robert C. Martin but the individual principles were assembled over a decade of published articles and lectures, each addressing a specific, concrete failure mode that appears repeatedly in large codebases.

The principles are not rules. They are not a checklist. They are not a certification. They are **design heuristics** — informed by thousands of engineering hours of observing what makes software expensive to change and what makes it cheap to change.

Violating these principles does not cause a runtime error. The compiler will not warn you. What violations cause is a slower, more painful, more error-prone future for every engineer who touches the code after you. The consequences are:

| Principle Violated | Real-World Consequence |
|---|---|
| SRP | A change to the invoice formatting logic breaks the payment calculation. A change to the database schema requires modifying business rules. Bug fixes in one area introduce bugs in an unrelated area. |
| OCP | Adding a new payment method requires modifying the core `OrderProcessor` class, retesting all existing payment paths, and re-deploying the entire service. |
| LSP | Code that passed all tests with `List` breaks silently when replaced with a `ReadOnlyList` subtype that throws on `add()`. Defensive `instanceof` checks proliferate throughout the codebase. |
| ISP | A class that implements a 12-method interface is forced to implement 8 methods it does not need, littering the codebase with `throw new UnsupportedOperationException()`. Every caller of that interface has access to methods it should not know about. |
| DIP | The `UserService` creates a `MySQLUserRepository` internally. You cannot unit test `UserService` without a live database. You cannot swap the data store without modifying business logic. |

These are not hypothetical edge cases. Every engineer with more than a few years of experience has been burned by at least three of these. The purpose of this module is to give you a precise mental model for each failure mode so you can recognize it in code you did not write, name it in a code review, and fix it without over-engineering.

---

## 2. Historical Context

Understanding where SOLID came from matters because it explains what problems the principles were designed to solve, and therefore where they apply and where they do not.

**Robert C. Martin** ("Uncle Bob") is an American software engineer and author whose work from the 1990s and 2000s shaped a generation of thinking about object-oriented design. The individual principles did not arrive together. They emerged separately:

- **SRP** was first articulated by Martin in the early 1990s as a direct response to the god-class problem — classes that accumulated behavior from multiple domains until they became impossible to maintain.
- **OCP** was originally stated by **Bertrand Meyer** in his 1988 book *Object-Oriented Software Construction*, before Martin popularized it in the OO community. Meyer was arguing against the practice of modifying library code directly.
- **LSP** was defined by **Barbara Liskov** in her 1987 keynote address "Data Abstraction and Hierarchy" and formalized in her 1994 paper with Jeannette Wing. It is the only principle in SOLID with a rigorous mathematical foundation — it derives from behavioral subtyping theory.
- **ISP** emerged from Martin's consulting work with Xerox in the early 1990s. The Xerox printer software had a single `Job` interface with dozens of methods; every class that needed any printing behavior had to implement all of them. The interface had become a dependency bottleneck.
- **DIP** was articulated by Martin in the mid-1990s as a generalization of techniques like dependency injection and inversion of control, which were becoming common in framework design.

Martin assembled these five principles, named the acronym, and published them in their most influential form in his 2002 book **"Agile Software Development: Principles, Patterns, and Practices."** The book argued that these principles were the foundation of agile-compatible object-oriented design — designs that could absorb change without requiring rewrites.

The principles are products of their era. They were developed in the context of statically typed, class-based OOP languages (C++, Java, Smalltalk). They assume a certain style of design — many small classes, interfaces as contracts, dependency injection. They should be understood in that context rather than applied as universal laws across all programming paradigms.

---

## 3. The Cost of Ignorance: A Class That Violates All Five Principles

Before studying each principle in isolation, it is instructive to see what code looks like when a developer ignores all five simultaneously. This is not a contrived example — it is a fair representation of code that gets written in real systems under deadline pressure, without design discipline.

```java
// A class that violates all five SOLID principles simultaneously.
// This is a cautionary example, not a template.

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.util.List;

public class OrderManager {

    // DIP violation: directly constructs a concrete database dependency.
    // Cannot be tested without a live MySQL database.
    // Cannot swap to PostgreSQL or an in-memory DB without modifying this class.
    private Connection connection;

    public OrderManager() {
        try {
            this.connection = DriverManager.getConnection(
                "jdbc:mysql://localhost:3306/orders", "root", "password"
            );
        } catch (Exception e) {
            throw new RuntimeException("DB connection failed", e);
        }
    }

    // SRP violation: this method calculates price, applies tax, saves to DB,
    // sends an email, and generates a PDF receipt.
    // Four completely separate reasons to change, all in one method.
    // A change to the email template requires touching the same class as a
    // change to the tax calculation formula.
    public void processOrder(String customerId, List<String> itemIds, String paymentType) {

        // Business logic: calculate total
        double total = 0.0;
        for (String itemId : itemIds) {
            // Hardcoded pricing logic — SRP and OCP violation
            if (itemId.startsWith("BOOK")) {
                total += 15.0;
            } else if (itemId.startsWith("ELEC")) {
                total += 299.0;
            } else if (itemId.startsWith("CLOTH")) {
                total += 49.0;
            }
            // OCP violation: every new product category requires modifying this method.
            // There is no way to add a new category type without opening this class.
        }

        // Tax logic also embedded here — SRP violation
        double tax = total * 0.18;
        double finalAmount = total + tax;

        // Payment processing — SRP violation, and another OCP violation
        if (paymentType.equals("CREDIT_CARD")) {
            System.out.println("Charging credit card: " + finalAmount);
        } else if (paymentType.equals("UPI")) {
            System.out.println("Processing UPI payment: " + finalAmount);
        } else if (paymentType.equals("WALLET")) {
            System.out.println("Deducting from wallet: " + finalAmount);
        }
        // OCP violation: adding "BANK_TRANSFER" requires modifying this block.

        // Persistence logic — SRP violation, DIP violation (raw JDBC in business code)
        try {
            PreparedStatement stmt = connection.prepareStatement(
                "INSERT INTO orders (customer_id, total) VALUES (?, ?)"
            );
            stmt.setString(1, customerId);
            stmt.setDouble(2, finalAmount);
            stmt.executeUpdate();
        } catch (Exception e) {
            throw new RuntimeException("Failed to save order", e);
        }

        // Notification logic — SRP violation
        System.out.println("Sending email to customer " + customerId);
        System.out.println("Receipt:\n" + generateReceipt(customerId, finalAmount));
    }

    // ISP violation: this interface (if extracted) would force any implementor
    // to provide both report generation and order management. These are
    // completely different capabilities that should not be bundled.
    // The method below belongs to a separate reporting concern.
    public String generateReceipt(String customerId, double amount) {
        return "Order for " + customerId + " | Amount: " + amount;
    }

    // ISP violation: admin reporting functionality bundled into the same class
    // as order processing. A component that only needs to generate reports
    // is forced to depend on (and know about) order processing logic.
    public void generateMonthlyReport() {
        System.out.println("Generating monthly order report...");
    }

    // LSP violation: if a subclass overrides processOrder() and throws an
    // exception for certain payment types (e.g., a "FreeTrialOrderManager"
    // that refuses paid payment methods), callers using the base type
    // will break. The contract of the base class is silently violated.
    // This design has no behavioral contract — there is nothing to enforce
    // correct subtype behavior.
}
```

After studying this module, you will be able to articulate precisely which lines violate which principle, explain the specific failure mode each violation introduces, and produce a refactored design that eliminates all five violations. The refactored version will have more classes and more files — and that is correct. The additional structure pays for itself the first time a requirement changes.

---

## 4. The Five Principles — Core Summaries

Each principle has a dedicated file with deep-dive theory, multiple code examples, refactoring walkthroughs, and interview questions. The summaries below give you the core intuition and the most important non-obvious point for each principle.

---

### 4.1 Single Responsibility Principle (SRP)

**The principle:** A class should have one, and only one, reason to change.

The word "responsibility" is misleading on first encounter. It does not mean "does one thing." A class that processes orders does one thing — it processes orders. But that one activity involves calculating prices, applying taxes, saving to a database, sending notifications, and generating receipts. Those are five different reasons to change, driven by five different stakeholders (pricing team, tax authority, DBA, marketing team, legal team). SRP is about **change axes**, not about task count.

The correct mental model: for every class you write, ask "who are the stakeholders who would ask me to change this class?" If the answer is "the DBA and the marketing team and the tax authority," the class has too many responsibilities.

**The failure mode SRP prevents:** The ripple effect. A change required by one stakeholder inadvertently breaks behavior that another stakeholder cares about, because both lived in the same class. This is one of the most common sources of regression bugs in production systems.

**The non-obvious point:** SRP applies at every level of the architecture — class, module, service. A microservice that handles user authentication, payment processing, and email delivery violates SRP at the service level. The principle scales.

---

### 4.2 Open/Closed Principle (OCP)

**The principle:** Software entities (classes, modules, functions) should be open for extension but closed for modification.

"Open for extension" means you can add new behavior to the system. "Closed for modification" means you can add that behavior without changing existing, tested, deployed code. The mechanism that makes this possible is **depending on abstractions rather than concretions**. If your `OrderProcessor` depends on a `PaymentStrategy` interface rather than a `CreditCardPayment` class, adding a new payment method means writing a new implementation of `PaymentStrategy` — no modification of `OrderProcessor` required.

The direct consequence is that the risk of adding new features is lower. When you do not modify existing code, you do not break existing tests. When you do not break existing tests, you do not introduce regressions. When you do not introduce regressions, deployment is less frightening.

**The failure mode OCP prevents:** The modification spiral. A feature request arrives, you open a core class to add an `if` branch, the change touches a function with 15 callers, three of them break, you fix those, two more break, and two weeks later you have touched 12 files for what should have been a 2-hour task.

**The non-obvious point:** OCP exists in creative tension with YAGNI (You Ain't Gonna Need It). Building extension points for behavior that may never change is gold-plating. The skill is identifying the axes of change that are genuinely likely — where the business changes frequently — and making those axes cheap to extend, while not over-abstracting axes that are stable.

---

### 4.3 Liskov Substitution Principle (LSP)

**The principle:** If `S` is a subtype of `T`, then objects of type `T` may be replaced with objects of type `S` without altering any of the desirable properties of the program.

LSP is the most technically rigorous of the five principles. Barbara Liskov derived it from behavioral subtyping theory: a subtype must honor the behavioral contract of its supertype, not just its method signatures. A subclass that overrides a method in a way that violates the calling code's assumptions about that method is an LSP violation, even if it compiles and runs without throwing an exception.

The classic violation: `Square` extends `Rectangle`. A `Rectangle` has independent width and height. A `Square` enforces width == height. Code that holds a `Rectangle` reference and calls `setWidth(5); setHeight(10);` then asserts `area() == 50` will silently get `area() == 100` when handed a `Square`. No exception, no compilation error, wrong answer. The subtype cannot be substituted for the supertype without altering the program's correctness.

LSP violations manifest as: `instanceof` checks before calling methods, `UnsupportedOperationException` in overridden methods, surprising behavior when polymorphism is used, and subtle bugs that only appear when a subtype is introduced.

**The failure mode LSP prevents:** Polymorphism becoming unreliable. If you cannot trust that a subtype behaves like its supertype, you are forced to write defensive code throughout the system — checking what type you actually have, guarding against exceptions that should not be thrown, adding special cases. The entire benefit of polymorphism is destroyed.

**The non-obvious point:** LSP is fundamentally about **behavioral contracts**, not method signatures. An interface specifies what methods exist; LSP specifies what those methods must guarantee in terms of preconditions, postconditions, and invariants. Design by Contract (Bertrand Meyer's concept) is the formal framework that underlies LSP.

---

### 4.4 Interface Segregation Principle (ISP)

**The principle:** Clients should not be forced to depend on methods they do not use.

When an interface is too broad — containing methods for multiple distinct capabilities — every class that implements it must provide implementations for all methods, even irrelevant ones. Every class that depends on the interface sees all methods, even the ones it should not call. The interface becomes a coupling point: a change to any method in it forces a recompile (and possibly a change) in every implementor and every caller, regardless of whether they use that specific method.

The fix is to split large interfaces into smaller, focused ones — one per cohesive capability. A `Printer` interface for printing, a `Scanner` interface for scanning, a `Fax` interface for faxing, rather than a single `MultiFunctionDevice` interface that bundles all three. A client that only needs to print depends only on `Printer` and is insulated from changes to scanning and faxing logic.

**The failure mode ISP prevents:** Fat interfaces acting as coupling hubs. In a system with a 20-method interface, a change to method 17 triggers recompilation across 40 classes, most of which do not call method 17. The blast radius of a change is determined by what is in the interface, not what any individual class actually uses.

**The non-obvious point:** ISP is the interface-level analog of SRP. SRP governs what goes in a class; ISP governs what goes in an interface. They address the same underlying problem — excessive coupling through over-bundled responsibilities — at different levels of abstraction. If a class has too many responsibilities (SRP violation), the interface it implements or exposes will often be too broad (ISP violation).

---

### 4.5 Dependency Inversion Principle (DIP)

**The principle:** High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details. Details should depend on abstractions.

The word "inversion" refers to a reversal of the traditional dependency direction. In naive procedural thinking, high-level policy code (business logic) calls low-level mechanism code (database access, email sending, HTTP clients). The high-level module depends on the low-level module. DIP inverts this: both the high-level module and the low-level module depend on an **interface** that lives at the high-level layer. The high-level module defines the contract it needs; the low-level module implements it.

The practical consequence is that business logic can be written and tested in complete isolation from infrastructure. An `OrderService` depends on an `OrderRepository` interface. The `MySQLOrderRepository` implements that interface. In tests, you inject a `FakeOrderRepository`. The business logic is never coupled to MySQL, and you can run your entire test suite without a database.

DIP is the foundation of dependency injection (DI) frameworks like Spring, Guice, and Dagger. It is also the design principle underlying the Ports and Adapters (Hexagonal Architecture) pattern.

**The failure mode DIP prevents:** Untestable business logic and locked-in infrastructure. When a `UserService` creates a `new MySQLUserRepository()` internally, you cannot unit test `UserService` without a MySQL instance. You cannot swap the database without modifying the service. You cannot mock the repository to simulate error conditions. The business logic and the infrastructure are fused, and changing one requires touching the other.

**The non-obvious point:** DIP does not say "always use dependency injection." It says the direction of the dependency should point toward abstractions, and the abstractions should be owned by the high-level layer. In many cases, a factory method or a service locator can achieve the same inversion. Dependency injection frameworks are the most common mechanism in Java, but they are a tool, not the principle itself.

---

## 5. How the Five Principles Interrelate

The five principles are not independent. They are five different facets of the same underlying goal: **managing dependencies so that change in one part of a system does not unexpectedly break another part**. Violating one principle typically creates conditions that make it easy to violate others. Applying one principle often naturally leads to applying others.

**SRP + ISP:** SRP says a class should have one reason to change. ISP says an interface should contain only the methods a client needs. If a class violates SRP (too many responsibilities), it will typically expose an interface that violates ISP (too many methods for any single client). When you fix a SRP violation by decomposing a class, you often automatically fix an ISP violation by splitting the interface.

**OCP + DIP:** OCP says you should be able to add new behavior without modifying existing code. The mechanism that makes this possible is depending on abstractions (DIP). You cannot have OCP without DIP. If `OrderProcessor` directly constructs `CreditCardPayment`, you must modify `OrderProcessor` to add `UPIPayment`. If `OrderProcessor` depends on a `PaymentStrategy` interface (DIP), adding `UPIPayment` is a new class — no modification required (OCP honored).

**LSP + OCP:** LSP is the constraint that makes OCP safe. OCP says "extend without modifying." That extension is done through subtyping — writing new subclasses or new implementations of an interface. If the new subtype violates LSP (does not honor the behavioral contract of the base type), the extension breaks existing code. LSP is the correctness guarantee that makes extension safe.

**DIP + LSP:** DIP says depend on abstractions. LSP says the concrete types that fulfill those abstractions must be substitutable. DIP creates the architecture (abstractions everywhere); LSP ensures the architecture is safe to use (every concrete type honors the contract). Together they form the foundation for reliable polymorphism at scale.

**ISP + DIP:** When DIP is applied, the abstractions (interfaces) that high-level modules depend on should be narrow and focused. If those interfaces are too broad (ISP violation), the high-level module is still coupled to more than it needs, even though it depends on an abstraction. ISP ensures that the abstractions created by DIP are precisely scoped.

A useful mental model: **SRP and ISP govern the shape of your components** (each class and interface does one cohesive thing). **OCP and DIP govern the direction of dependencies** (point toward abstractions, add new behavior by adding new code). **LSP governs the correctness of the component graph** (every concrete type can be trusted to honor its contract).

---

## 6. Common Interview Traps

Senior engineer interviews routinely probe SOLID because it is easy to distinguish between candidates who memorize definitions and candidates who have actually applied the principles in production. The following are traps that appear frequently.

---

**Trap 1: Reciting definitions without consequences**

A weak answer defines each principle in isolation. A strong answer connects each principle to the specific failure mode it prevents and gives an example of a production consequence. If you can only say "SRP means a class should do one thing," you will not differentiate yourself from every other candidate who read the same tutorial.

Required depth: What breaks when SRP is violated? What kind of bug does an OCP violation produce? What test failure would indicate an LSP violation?

---

**Trap 2: "SOLID is always right"**

SOLID principles exist in tension with other principles — particularly YAGNI, DRY, and simplicity. A good interviewer will present a scenario where applying a principle costs more than it saves. For example: "You have a small script that runs once a month. Should you apply DIP and inject all dependencies?"

The correct answer acknowledges the tradeoff. DIP adds indirection and complexity. For a script with no tests and no requirement to swap implementations, the added complexity is not justified. SOLID is not free — it trades some immediacy for long-term flexibility, and that trade is not always worth making.

---

**Trap 3: Confusing mechanism with principle**

Many engineers confuse the principle with its most common implementation. DIP is not "use Spring's @Autowired." OCP is not "use the Strategy pattern." These are mechanisms that apply the principle. The principle is the goal; the mechanism is one tool for achieving it. An interviewer who asks "how do you apply OCP?" is looking for an answer about the design goal, not a list of patterns.

---

**Trap 4: Applying SRP at the wrong granularity**

A common over-application: "I have a method with three responsibilities, so I should split it into three classes." SRP applies at the class and module level. A method within a class can have a few steps without violating SRP, as long as those steps all belong to the same cohesive responsibility. Taking SRP to the extreme produces systems with hundreds of single-method classes, which are just as difficult to maintain as god classes — they scatter cohesive logic across the codebase.

---

**Trap 5: Not knowing the Rectangle/Square example for LSP**

The Rectangle extends Square (or Square extends Rectangle) example is the canonical LSP illustration. Every senior engineer interview that touches LSP will either use this example or a structural analog to it. If you cannot explain why it violates LSP, what the actual failure is, and how to fix it (using composition or separate types), you will not pass the principle section of the interview.

---

**Trap 6: Conflating ISP with "small interfaces"**

ISP does not say "all interfaces should be small." It says "clients should not be forced to depend on methods they do not use." An interface can be broad if all its clients actually use all its methods. The violation occurs when clients are forced to implement or depend on methods irrelevant to them. Size is a symptom; forced irrelevant dependency is the actual violation.

---

**Trap 7: Treating the principles as sequential or prioritized**

SOLID is an acronym, not a priority order. The principles are not applied sequentially. In practice, you think about all five simultaneously during design. Framing one as "more important" than another is a false choice — they address different dimensions of the same problem.

---

## 7. Navigation: Principle Files

Each principle has a dedicated file with full theory, Java code examples, before/after refactoring walkthroughs, common violations to recognize, and targeted interview questions.

| File | Principle | Core Question It Answers |
|---|---|---|
| [01-single-responsibility.md](./01-single-responsibility.md) | Single Responsibility Principle | How do I decompose a class that does too many things? |
| [02-open-closed.md](./02-open-closed.md) | Open/Closed Principle | How do I add new behavior without modifying existing code? |
| [03-liskov-substitution.md](./03-liskov-substitution.md) | Liskov Substitution Principle | How do I design class hierarchies that can be used polymorphically without surprises? |
| [04-interface-segregation.md](./04-interface-segregation.md) | Interface Segregation Principle | How do I design interfaces that don't impose irrelevant obligations on implementors? |
| [05-dependency-inversion.md](./05-dependency-inversion.md) | Dependency Inversion Principle | How do I write business logic that can be tested without infrastructure? |

---

## 8. How to Study This Module

**Recommended sequence:** Read this README first, study the principles in file order (SRP through DIP), then do the assignments.

The sequence is intentional:

1. **Start with SRP** because it is the most concrete and immediately recognizable. Most engineers have written or maintained god classes. SRP gives you a vocabulary to describe the problem and a technique to fix it. It also introduces the concept of "reasons to change," which recurs across all five principles.

2. **Study OCP second** because it introduces the abstraction-based design mode that underpins the remaining three principles. OCP is where you start thinking in terms of interfaces and extension points rather than concrete types and branching logic.

3. **Study LSP third** because it adds the correctness constraint on top of the abstraction framework introduced by OCP. You need to understand what "depend on abstractions" means (OCP/DIP) before you can appreciate why "the abstractions must have reliable behavioral contracts" (LSP) matters.

4. **Study ISP fourth** because by this point you are already thinking in terms of interfaces and abstractions. ISP refines that thinking: not just "use an interface" but "use a precisely scoped interface."

5. **Study DIP last** because it is the most architectural of the five principles and the one that ties all others together. After understanding the other four, DIP feels like the natural conclusion: if high-level modules depend on well-scoped (ISP), reliably behaving (LSP), extension-friendly (OCP) abstractions, and those abstractions are correctly owned at the right layer (DIP), the system as a whole becomes genuinely maintainable.

**After reading each file:**
- Identify one class in a codebase you work on or have access to that violates the principle.
- Write a brief description of what would break if the violation were amplified — if the class grew to 10x its current size with the same structure.
- Sketch a refactored version without writing the code. Can you describe the class structure in words?
- Then write the code.

**Do the assignments** at the end of this module. The assignments are structured to force application of the principles under time pressure, which is the condition you will face in interviews. Reading without applying produces knowledge that evaporates under interview pressure.

**Engage with the "Common Violations" section in each file** before looking at the refactored examples. Try to identify what is wrong before being told. This builds the recognition muscle that actually matters in interviews.

**Cross-reference the principles.** After studying all five, return to the `OrderManager` example in this README and label every line that violates a principle. If you can do this without looking at the files again, you have internalized the material.

---

## 9. Prerequisites Checklist

This module assumes you have completed Module 00 and can demonstrate the following with confidence. If any item is uncertain, return to Module 00 before proceeding.

**OOP Pillars**
- [ ] You can explain encapsulation as invariant protection, not just "use private fields."
- [ ] You can define abstraction as hiding implementation complexity behind a stable interface — and give an example of a leaky abstraction.
- [ ] You can explain the fragile base class problem and why it is a risk of inheritance.
- [ ] You can distinguish compile-time polymorphism (overloading) from runtime polymorphism (overriding) and explain when each applies.

**Class vs. Abstract Class vs. Interface**
- [ ] You can articulate when to use an interface versus an abstract class with a concrete decision rule, not a vague preference.
- [ ] You understand that an interface defines a contract with no implementation bias, and that default methods in Java 8+ blur this but do not eliminate the distinction.

**Composition vs. Inheritance**
- [ ] You default to composition and can explain why without looking at notes.
- [ ] You can name at least two scenarios where inheritance is still the right choice.

**Object Modeling**
- [ ] Given a 200-word domain description, you can produce a class diagram with 5+ entities, labeled relationships, and primary methods in under 15 minutes.

**Clean Code**
- [ ] You name classes with nouns and methods with verb-noun pairs consistently.
- [ ] You keep methods at a single level of abstraction.
- [ ] Your code comments explain why, not what.

If you can check every box with genuine confidence, proceed to the principle files.

---

*This module is part of the LLD & Object-Oriented Design Mastery course targeting software engineers with 2-8 years of experience preparing for Senior Engineer roles.*
