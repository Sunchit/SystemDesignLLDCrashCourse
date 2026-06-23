# SOLID Principles Cheat Sheet

One-page reference for all five principles. For each: definition → violation symptom → fix → memory hook → code example → interview one-liner.

---

## S — Single Responsibility Principle (SRP)

**Definition:** A class should have one, and only one, reason to change. Every responsibility is a potential axis of change; keeping them separate limits blast radius.

**Violation symptom:** A class whose name contains "And" or "Manager" that touches persistence, business logic, and formatting in the same file. Methods that have nothing in common sitting in the same class.

**The fix:** Extract each responsibility into its own class. If you can name two distinct business reasons a class could be modified, split it.

**Memory hook:** _"One class, one job. If the HR department and the IT department would both ask you to change this class, it violates SRP."_

```java
// BAD — UserService handles business logic AND email formatting AND DB write
class UserService {
    void registerUser(User u) { validate(u); sendWelcomeEmail(u); db.save(u); }
}

// GOOD — responsibilities separated
class UserRegistrationService { void register(User u) { validate(u); repo.save(u); } }
class EmailService            { void sendWelcome(User u) { mailer.send(u.email()); } }
```

**Interview one-liner:** "SRP says a class should have only one reason to change — one business concern, one owner."

---

## O — Open/Closed Principle (OCP)

**Definition:** Software entities should be open for extension but closed for modification. Add new behavior by adding new code, not by editing existing code.

**Violation symptom:** A `switch` or `if-else` chain on a type tag (`instanceof`, an enum, a string constant) that grows every time a new variant is added. Touching proven, tested code to support a new use case.

**The fix:** Introduce an abstraction (interface or abstract class). New variants implement or extend the abstraction without modifying existing implementations or the code that uses them.

**Memory hook:** _"OCP = 'Add, don't edit.' Think of a plugin system — you add a plugin, you don't rewrite the host."_

```java
// BAD — must edit this method every time a new shape is added
double area(Shape s) { if (s instanceof Circle) ... else if (s instanceof Rect) ... }

// GOOD — each shape owns its area calculation; no edits needed to add Triangle
interface Shape     { double area(); }
class Circle        implements Shape { public double area() { return PI * r * r; } }
class Triangle      implements Shape { public double area() { return 0.5 * b * h; } }
```

**Interview one-liner:** "OCP means you extend behavior through new classes, not through editing existing ones — protected by abstraction."

---

## L — Liskov Substitution Principle (LSP)

**Definition:** Objects of a subtype must be substitutable for objects of their supertype without altering the correctness of the program. A subclass must honor the contract of its parent.

**Violation symptom:** An overridden method that throws `UnsupportedOperationException`, weakens a precondition, strengthens a postcondition, or silently does nothing. A subclass that makes the parent's contract a lie.

**The fix:** Prefer composition over inheritance when the subtype cannot fully honor the parent's contract. If a subclass needs to restrict behavior, it should not inherit — it should implement a narrower interface.

**Memory hook:** _"LSP = 'No surprises.' If you hand me a subclass where I expected the parent, I should not notice."_

```java
// BAD — Square violates Rectangle's contract (setWidth changes height)
class Square extends Rectangle {
    void setWidth(int w)  { super.setWidth(w); super.setHeight(w); } // breaks caller
}

// GOOD — separate types, shared interface; no contract violation
interface Shape { double area(); }
class Rectangle implements Shape { ... }
class Square    implements Shape { ... }
```

**Interview one-liner:** "LSP says you must be able to swap a subclass for its parent and have the program behave correctly — no surprise exceptions or silent no-ops."

---

## I — Interface Segregation Principle (ISP)

**Definition:** Clients should not be forced to depend on interfaces they do not use. Prefer many small, focused interfaces over one fat interface.

**Violation symptom:** An interface with ten methods where every implementor leaves three of them as empty stubs or throwing `UnsupportedOperationException`. Classes coupled to methods they never call.

**The fix:** Break the fat interface into role-specific interfaces. Each client depends only on the slice it needs. Implementing classes may implement multiple slim interfaces.

**Memory hook:** _"ISP = 'Don't force me to sign a contract I won't honor.' Thin interfaces, focused contracts."_

```java
// BAD — Printer is forced to implement fax() which it doesn't support
interface MultiFunctionDevice { void print(); void scan(); void fax(); }
class Printer implements MultiFunctionDevice { public void fax() { throw new UnsupportedOperationException(); } }

// GOOD — clients depend only on what they need
interface Printable { void print(); }
interface Scannable { void scan(); }
class Printer       implements Printable { public void print() { ... } }
```

**Interview one-liner:** "ISP says split fat interfaces so no client is forced to depend on methods it doesn't use."

---

## D — Dependency Inversion Principle (DIP)

**Definition:** High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details; details should depend on abstractions.

**Violation symptom:** A service class directly instantiating a concrete repository or infrastructure class (`new MySQLRepository()`). Business logic that breaks when the database is swapped. Constructor calls hardwiring dependencies.

**The fix:** Depend on an interface, not a concrete class. Inject the dependency from outside (constructor injection, DI framework). The high-level module defines the interface it needs; the low-level module implements it.

**Memory hook:** _"DIP = 'Depend on the menu, not the chef.' The menu (interface) is stable; the chef (implementation) can be replaced."_

```java
// BAD — OrderService is hardwired to MySQL
class OrderService { private MySQLRepo repo = new MySQLRepo(); }

// GOOD — depends on abstraction; implementation injected from outside
interface OrderRepository { void save(Order o); }
class OrderService {
    OrderService(OrderRepository repo) { this.repo = repo; } // inject any impl
}
```

**Interview one-liner:** "DIP says depend on abstractions, not concretions — high-level policy should not know which database or library is behind the interface."

---

## SOLID at a Glance

| Principle | One-line definition | Key word |
|-----------|--------------------|--------------------|
| **S**RP | One class, one reason to change | Responsibility |
| **O**CP | Extend by adding, not editing | Extension |
| **L**SP | Subtypes must honor parent contracts | Substitutability |
| **I**SP | Many focused interfaces over one fat interface | Segregation |
| **D**IP | Depend on abstractions, not concretions | Inversion |

---

## Common SOLID Interview Questions

**Q1. Can you violate SRP without a class being "large"?**
Yes. A 10-line class can violate SRP if it handles two distinct business concerns — size is not the indicator; number of reasons to change is.

**Q2. Does OCP mean you never modify existing code?**
No. OCP applies to stable, proven code. Fixing bugs, refactoring internals of a class, and removing dead code are all legitimate. OCP guards against the anti-pattern of adding a new case by editing a working `if-else` chain.

**Q3. What is the classic LSP violation in Java's standard library?**
`java.util.Stack extends Vector`. Stack exposes `get(index)`, `add(index, element)`, and `remove(index)` through Vector — operations that violate Stack's LIFO contract. The design should have used composition, not inheritance.

**Q4. What is the difference between ISP and SRP?**
SRP is about classes — a class should do one thing. ISP is about interfaces — a client should not depend on interface methods it does not call. They often co-enforce each other: splitting a fat interface (ISP) tends to drive classes toward a single responsibility (SRP).

**Q5. Is Dependency Injection the same as DIP?**
No — DI is a mechanism; DIP is a principle. DI (constructor injection, setter injection, frameworks like Spring) is the most common way to implement DIP, but DIP can also be satisfied by factories or service locators. DIP is the goal; DI is one tool to achieve it.

---

## SOLID + Patterns Mapping

| Principle | Design Patterns that directly enforce it |
|-----------|------------------------------------------|
| SRP | **Facade** (hides multiple subsystems behind one surface), **Command** (encapsulates one operation per class) |
| OCP | **Strategy** (swap algorithms without touching the context), **Decorator** (extend behavior without modifying the component), **Template Method** (fixed skeleton, open steps) |
| LSP | **Factory Method** (creates objects through a common interface, substitutable products), **Abstract Factory** (families of substitutable objects) |
| ISP | **Adapter** (exposes only the interface the client needs), **Proxy** (thin wrapper presenting a minimal interface) |
| DIP | **Dependency Injection** (inject abstractions), **Repository** (business logic depends on a repo interface, not the DB), **Observer** (subject depends on observer interface, not concrete listeners) |
