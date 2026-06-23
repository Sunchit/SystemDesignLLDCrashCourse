# OOP & Design Principles — Flashcard Set

> 50 cards for active recall practice. Work through one section at a time, mark cards you miss, and revisit them in the next session.

---

## Section 1: OOP Fundamentals (Cards 1–12)

---
**Q1:** What is encapsulation, and what problem does it solve?

**A:** Encapsulation is the practice of bundling data (fields) and the operations that act on that data (methods) inside a single unit, and restricting direct access to internal state. It solves the problem of uncontrolled mutation: external code can only change state through well-defined methods, making invariants easier to enforce and internal representation free to change without breaking callers.

---

---
**Q2:** What is the difference between abstraction and encapsulation?

**A:** Abstraction is about *what* an object does — exposing only the essential interface and hiding complexity. Encapsulation is about *how* it is protected — hiding internal state behind access controls. Abstraction operates at the design level (interfaces, abstract classes); encapsulation operates at the implementation level (private fields, getters/setters). You can have one without the other, but they work best together.

---

---
**Q3:** What are the four types of inheritance, and which does Java support directly?

**A:**
- **Single inheritance** — one parent class (Java supports this for classes)
- **Multilevel inheritance** — A extends B extends C (Java supports this)
- **Hierarchical inheritance** — multiple subclasses share one parent (Java supports)
- **Multiple inheritance** — one class extends multiple parents (Java supports this only through interfaces, not concrete classes, to avoid the diamond problem)

---

---
**Q4:** What is polymorphism? Distinguish between compile-time and runtime polymorphism.

**A:** Polymorphism means "many forms" — a single interface can refer to objects of different types. **Compile-time (static) polymorphism** is achieved through method overloading: the correct method is selected at compile time based on argument types. **Runtime (dynamic) polymorphism** is achieved through method overriding: the JVM dispatches to the correct subclass implementation at runtime via a virtual method table.

---

---
**Q5:** What is the Liskov Substitution Principle as it relates to inheritance? (Brief answer — the full card is in Section 2.)

**A:** LSP states that any subclass instance must be usable wherever a superclass instance is expected, without breaking the program's correctness. In practice, subclasses must honour the contracts (preconditions, postconditions, invariants) of the parent. Violating LSP is a sign that inheritance is being misused and composition should be considered instead.

---

---
**Q6:** What is an abstract class, and when should you prefer it over an interface?

**A:** An abstract class is a class that cannot be instantiated and may contain both abstract methods (no body) and concrete methods (with body). Prefer an abstract class when: (1) you want to share implementation code among related classes, (2) you need non-public members (protected fields, package-private methods), or (3) you want to provide a default implementation that subclasses can selectively override. Use an interface when you want to define a pure contract that unrelated classes can implement.

---

---
**Q7:** What is an interface, and how did Java 8 change its capabilities?

**A:** An interface is a pure contract — a set of method signatures that implementing classes promise to fulfil. Before Java 8, all methods were implicitly public and abstract. Java 8 introduced **default methods** (concrete implementations in interfaces) and **static methods**, enabling backward-compatible API evolution without breaking existing implementations. Java 9 added private methods to share logic between default methods.

---

---
**Q8:** What is method overriding, and what are its rules in Java?

**A:** Method overriding is redefining an inherited method in a subclass to provide a different implementation. Rules: the method signature (name + parameters) must match exactly; the return type must be the same or a covariant subtype; access modifier cannot be more restrictive; checked exceptions cannot be broader. The `@Override` annotation is strongly recommended to catch mistakes at compile time.

---

---
**Q9:** What is the diamond problem, and how does Java resolve it?

**A:** The diamond problem occurs in multiple inheritance when a class inherits the same method from two different parents that themselves share a common ancestor — creating ambiguity about which version to use. Java avoids it for classes by disallowing multiple class inheritance. For interfaces with conflicting default methods, Java requires the implementing class to explicitly override the method and resolve the conflict.

---

---
**Q10:** What is a constructor, and what happens if you do not define one?

**A:** A constructor is a special method that initialises a new object's state when it is created. If you define no constructor, the Java compiler inserts a public no-arg constructor. If you define any constructor with parameters, the compiler does NOT insert a no-arg constructor — callers must use one of your defined constructors. Subclasses automatically call the parent's no-arg constructor unless you explicitly call `super(...)`.

---

---
**Q11:** What is the difference between `is-a` and `has-a` relationships?

**A:** An **is-a** relationship represents inheritance: a `Dog` *is-a* `Animal`. It implies that the subclass can be substituted anywhere the parent is expected. A **has-a** relationship represents composition: a `Car` *has-a* `Engine`. Use is-a when the subclass genuinely is a specialisation of the parent; use has-a (composition) when you want to reuse behaviour without the tight coupling of inheritance. Prefer has-a when in doubt.

---

---
**Q12:** What is the difference between early binding and late binding?

**A:** **Early (static) binding** resolves the method call at compile time based on the declared type of the reference. Method overloading and calls to `private`, `static`, or `final` methods are early-bound. **Late (dynamic) binding** resolves the method call at runtime based on the actual object type. Overridden instance methods are late-bound through the JVM's virtual dispatch mechanism, enabling runtime polymorphism.

---

---

## Section 2: SOLID Principles (Cards 13–22)

---
**Q13:** What does the Single Responsibility Principle (SRP) state, and how do you detect a violation?

**A:** SRP states that a class should have only one reason to change — i.e., it should serve one actor or stakeholder. Violations show up as: classes with many unrelated methods, classes that grow without bound, and tests that must mock many unrelated dependencies. A useful heuristic: if you struggle to name a class without using "And" or "Manager", it probably has more than one responsibility.

---

---
**Q14:** What does the Open/Closed Principle (OCP) state? Give a concrete example.

**A:** OCP states that a software entity should be open for extension but closed for modification. You should be able to add new behaviour without changing existing, tested code. **Example:** a payment processor that uses a `switch` on payment type must be modified every time a new type is added (violates OCP). Replacing it with a `PaymentStrategy` interface lets you add new strategies without touching the processor — it is extended, not modified.

---

---
**Q15:** What does the Liskov Substitution Principle (LSP) state? Give an example of a violation.

**A:** LSP states that objects of a subclass must be substitutable for objects of the superclass without altering the correctness of the program. **Classic violation:** `Square extends Rectangle`. Setting width on a `Square` changes both sides, but a function written for `Rectangle` expects width and height to be independent — substituting a `Square` breaks it. The fix is to model them as separate, unrelated classes.

---

---
**Q16:** What does the Interface Segregation Principle (ISP) state? What smell does it prevent?

**A:** ISP states that no client should be forced to depend on methods it does not use. Fat interfaces — those with many unrelated methods — force implementing classes to stub out methods with empty bodies or throw `UnsupportedOperationException`, which is both deceptive and fragile. The fix is to split fat interfaces into smaller, role-specific interfaces so each client depends only on what it actually needs.

---

---
**Q17:** What does the Dependency Inversion Principle (DIP) state? How is it different from dependency injection?

**A:** DIP states that (1) high-level modules should not depend on low-level modules — both should depend on abstractions, and (2) abstractions should not depend on details — details should depend on abstractions. **Dependency Injection (DI)** is a technique (often a framework feature) for supplying dependencies from outside a class. DI is a common *mechanism* for achieving DIP, but DIP is the *principle* — you can satisfy DIP without a DI framework.

---

---
**Q18:** How do SOLID principles relate to each other? Which two are most often violated together?

**A:** The principles reinforce each other: SRP encourages small, focused classes; OCP and DIP both push toward programming to interfaces; ISP keeps interfaces focused; LSP enforces correct subtype contracts. SRP and ISP are most often violated together — a class that does too many things frequently also implements a fat interface. Cleaning up the interface (ISP) often naturally leads to splitting the class (SRP).

---

---
**Q19:** A class reads from a database, transforms data, and writes to a CSV file — all in one method. Which SOLID principles does this violate?

**A:**
- **SRP** — three distinct responsibilities (read, transform, write) means three reasons to change.
- **OCP** — adding a new output format requires modifying the existing method.
- **DIP** — the method depends directly on concrete database and CSV implementations, not on abstractions.

Refactor by extracting a `DataSource` interface, a `DataTransformer`, and a `DataWriter` interface, each with their own implementations.

---

---
**Q20:** What is the difference between programming to an interface and programming to an implementation?

**A:** Programming to an **interface** means your variable types, method parameters, and return types use abstract types (interfaces or abstract classes). The actual implementation is determined at runtime or injected externally. Programming to an **implementation** means you depend on concrete classes directly. The former is far more flexible: you can swap implementations (for testing, for new requirements) without changing consuming code.

---

---
**Q21:** When is it acceptable to violate OCP?

**A:** OCP is a guideline, not an absolute law. It is acceptable to modify existing code when: (1) the change is small and the abstraction cost of extending would be higher than the change itself, (2) you are fixing a genuine bug, (3) the code has no callers yet (you cannot know the right abstraction until the use cases stabilise). The key heuristic: protect code that is *proven to be a point of variation*; do not add abstractions speculatively.

---

---
**Q22:** What is the "fragile base class" problem, and which SOLID principle addresses it?

**A:** The fragile base class problem occurs when changes to a superclass unexpectedly break subclasses. Because subclasses depend on internal behaviour of the superclass (not just its public contract), an internal refactor of the parent can silently change subclass behaviour. **LSP** addresses this by demanding that subclasses honour the parent's contract. **DIP** reduces exposure by having both depend on stable abstractions rather than concrete implementations.

---

---

## Section 3: Design Principles — DRY / KISS / YAGNI / etc. (Cards 23–30)

---
**Q23:** What does DRY stand for, and what is the real definition (beyond "no copy-paste")?

**A:** DRY stands for **Don't Repeat Yourself**. The precise definition from *The Pragmatic Programmer* is: "Every piece of *knowledge* must have a single, unambiguous, authoritative representation within a system." It is broader than avoiding code duplication — it also applies to duplicated business rules, schema definitions, and documentation. Two pieces of code that look similar but represent *different business concepts* are not a DRY violation.

---

---
**Q24:** What is the danger of applying DRY too aggressively?

**A:** Over-applying DRY leads to **wrong abstractions** — merging things that happen to look similar but that diverge in the future. The abstraction then grows with conditionals to handle each caller's special case, becoming harder to understand than the original duplication. A useful rule: wait until code is duplicated three times before extracting an abstraction (the "Rule of Three"). The cost of the wrong abstraction is higher than the cost of duplication.

---

---
**Q25:** What does KISS mean in software design?

**A:** KISS stands for **Keep It Simple, Stupid** (or "Keep It Simple and Straightforward"). It is the principle that systems work best when kept as simple as possible. Complexity should be added only when it is genuinely required by the problem. Simple code is easier to read, test, debug, and hand off. When two solutions solve the same problem, prefer the simpler one even if it feels less clever.

---

---
**Q26:** What does YAGNI mean, and when is it most important to apply it?

**A:** YAGNI stands for **You Aren't Gonna Need It**. It means: do not add functionality or abstractions until they are actually required. It is most important during early development and when the domain is still being explored, because speculative code adds complexity, maintenance burden, and dead weight. YAGNI pairs naturally with OCP: add the extension point when a second concrete use case exists, not before.

---

---
**Q27:** What is the "Law of Demeter" (Principle of Least Knowledge)?

**A:** The Law of Demeter states that a method should only call methods on: (1) the object itself, (2) objects passed as parameters, (3) objects it creates, (4) its direct fields. It should not "reach through" objects to call methods on deeply nested objects (e.g., `order.getCustomer().getAddress().getCity()`). Violations create tight coupling between classes that have no direct relationship. The fix is often to add a delegation method at the intermediate level.

---

---
**Q28:** What is the Principle of Least Astonishment?

**A:** A component should behave in a way that most users would expect, given its name and context. A method called `getUser()` should not delete records. A method called `isEmpty()` should not throw exceptions. Violations cause subtle bugs because callers rely on implicit assumptions. Apply it by: naming things accurately, matching behaviour to conventions in the surrounding codebase, and documenting any unavoidable surprises prominently.

---

---
**Q29:** What is "Separation of Concerns," and how does it manifest in layered architecture?

**A:** Separation of Concerns (SoC) is the idea that a program should be decomposed into distinct sections, each addressing a single concern. In layered architecture: the **presentation layer** handles UI/API parsing; the **business/service layer** handles domain logic; the **data layer** handles persistence. Each layer depends only on the layer below it. SoC reduces cognitive load, enables independent testing of each layer, and lets teams change one layer without touching others.

---

---
**Q30:** What is "Tell, Don't Ask"?

**A:** "Tell, Don't Ask" means you should tell objects what to do rather than querying their state and making decisions externally. **Bad (ask):** `if (order.getStatus() == PENDING) { order.setStatus(APPROVED); }`. **Good (tell):** `order.approve()`. The object owns its own state transitions and encapsulates the logic. Asking violates encapsulation by leaking state and business rules into the caller; telling keeps logic where the data lives.

---

---

## Section 4: Coupling, Cohesion & Dependencies (Cards 31–38)

---
**Q31:** Define coupling. What are the types of coupling from worst to best?

**A:** Coupling measures how much one module depends on another. From worst to best:
1. **Content coupling** — one module directly accesses or modifies another's internals
2. **Common coupling** — both modules share global state
3. **Control coupling** — one module passes flags that control the other's behaviour
4. **Stamp/Data coupling** — one module passes a data structure, and the other uses only part of it
5. **Data coupling** — modules communicate only through simple parameters (ideal)
6. **Message coupling** — modules communicate only through messages/events (loosest)

---

---
**Q32:** Define cohesion. What are high cohesion and low cohesion?

**A:** Cohesion measures how closely the responsibilities within a single module are related. **High cohesion** means all elements of a class work toward the same well-defined purpose — the class is focused and its methods share data and logic. **Low cohesion** means the class has unrelated methods that change for different reasons. Aim for high cohesion and low coupling together; they are complementary goals that lead to maintainable, reusable modules.

---

---
**Q33:** What is the difference between tight and loose coupling? Give a code-level example of each.

**A:** **Tight coupling:** `OrderService` creates a `new MySQLOrderRepository()` inside its constructor. Any change to the repository forces a change in the service; testing requires a real database. **Loose coupling:** `OrderService` accepts an `OrderRepository` interface in its constructor. The concrete implementation is injected externally. The service can be tested with a mock, and the database can be swapped without touching the service.

---

---
**Q34:** What is "connascence," and why is it a more nuanced model than simple coupling?

**A:** Connascence (from Meilir Page-Jones) describes the *type* of coupling between two elements: if changing one requires changing the other, they are connascent. Types range from weak (Connascence of Name — sharing a method name) to strong (Connascence of Algorithm — sharing an algorithm that must stay in sync). It is more nuanced than binary coupling because it lets you rank refactoring priorities: reduce the strongest forms of connascence first.

---

---
**Q35:** What is the Stable Dependencies Principle?

**A:** The Stable Dependencies Principle states that modules should depend in the direction of stability — depend on things that are less likely to change. A frequently-changing (unstable) module should not be depended upon by stable modules. In practice: your domain model should be stable and depended upon by many; your infrastructure details (HTTP adapters, DB drivers) should be unstable and depended upon by few. Reversing this relationship makes the system fragile.

---

---
**Q36:** What is "dependency injection," and what are its three forms?

**A:** Dependency injection means providing a class's dependencies from the outside rather than letting it create them. Three forms:
1. **Constructor injection** — dependencies passed in constructor (preferred: makes dependencies explicit and class immutable)
2. **Setter injection** — dependencies set via setters (useful for optional dependencies)
3. **Field/interface injection** — framework injects directly into fields via annotations (convenient but hides dependencies and makes testing harder)

---

---
**Q37:** What is an "Acyclic Dependencies Principle" violation, and why is it harmful?

**A:** The Acyclic Dependencies Principle states that the dependency graph of packages/modules must have no cycles. A cycle means packages cannot be compiled, tested, or released independently — you must build them together, and a change in any one can force a rebuild of all. Cycles are broken by extracting a shared abstraction into a new lower-level package that both sides depend on, or by inverting one dependency with an interface.

---

---
**Q38:** When should you prefer composition over inheritance?

**A:** Prefer composition when:
- The relationship is not a true is-a (Liskov) relationship
- You need to combine behaviours from multiple sources (Java does not support multiple class inheritance)
- The superclass is not designed for extension (no `protected` hooks, `final` methods)
- You want the flexibility to change behaviour at runtime by swapping composed objects
- You want to avoid fragile base class issues

Use inheritance when the subclass is genuinely a specialisation of the parent and the inheritance hierarchy is shallow and stable.

---

---

## Section 5: Code Smells & Refactoring (Cards 39–50)

---
**Q39:** What is a "God Class," and what refactoring techniques address it?

**A:** A God Class knows too much and does too much — it accumulates responsibilities over time until it becomes the hub of the entire system. It violates SRP and high cohesion. Refactoring techniques: **Extract Class** to pull cohesive groups of fields and methods into new focused classes; **Move Method** to relocate behaviour to the class whose data it operates on; **Introduce Parameter Object** to reduce the class's responsibility for data aggregation.

---

---
**Q40:** What is "Feature Envy," and what does it indicate?

**A:** Feature Envy is a smell where a method in one class is more interested in the data of another class than its own. It typically calls many getters on another object to compute something. It indicates that the logic belongs in the other class, not the current one. Refactoring: **Move Method** to the class whose data the method uses, or **Extract Method** and then move the extracted method.

---

---
**Q41:** What is "Primitive Obsession," and why is it a smell?

**A:** Primitive Obsession is the overuse of primitive types (String, int, boolean) to represent domain concepts. Examples: using a `String` for a phone number, an `int` for money, or a `boolean` for state. It scatters validation and behaviour across the codebase. Refactoring: introduce **Value Objects** (e.g., `PhoneNumber`, `Money`, `OrderStatus`) that encapsulate validation, formatting, and business rules in one place.

---

---
**Q42:** What is "Shotgun Surgery," and how does it differ from Divergent Change?

**A:** **Shotgun Surgery**: one type of change requires modifying many different classes. Indicates that a single responsibility is scattered across the codebase. **Divergent Change**: one class is modified for many different reasons. Indicates too many responsibilities in one place. They are inverses: Divergent Change is a cohesion problem within a class; Shotgun Surgery is a cohesion problem across many classes. Both are fixed by consolidating related code into focused classes.

---

---
**Q43:** What is the "Data Clumps" smell, and what is the refactoring?

**A:** Data Clumps occur when the same group of data items appears together repeatedly across methods and classes (e.g., `firstName`, `lastName`, `email` always passed together). The data wants to be a class. Refactoring: **Introduce Parameter Object** or **Extract Class** to group the data into a new type. This opens up a home for behaviour that operates on that data and reduces method parameter lists.

---

---
**Q44:** What is "Long Method," and what are the refactoring techniques?

**A:** A long method is one that tries to do too much, making it hard to read and test. No strict line limit applies, but if you cannot fit the method on one screen or must scroll to follow a single train of thought, it is too long. Refactoring techniques: **Extract Method** to pull coherent blocks into named helper methods; **Replace Temp with Query** to eliminate intermediate variables; **Decompose Conditional** to extract the condition and branches into named methods.

---

---
**Q45:** What is "Dead Code," and why is it harmful?

**A:** Dead code is code that is never executed — unreachable branches, unused methods, commented-out blocks, unused parameters. It is harmful because it: adds cognitive load for readers who try to understand its purpose, may cause confusion about whether it is intentional, accumulates over time, and can harbour latent bugs if accidentally reactivated. Refactoring: delete it. Version control preserves history if it is ever needed again.

---

---
**Q46:** What is the "Switch Statement" smell? When is it genuinely a smell vs. acceptable?

**A:** A `switch` on a type field becomes a smell when it is duplicated in multiple places across the codebase — adding a new case requires modifying every switch. It is a signal to use **polymorphism**: create subclasses or strategy objects, each implementing the behaviour for their type. It is *not* a smell when the switch is isolated in one place (e.g., a factory method) or when the set of cases is genuinely closed and unlikely to change (e.g., days of the week).

---

---
**Q47:** What is "Speculative Generality," and how does it relate to YAGNI?

**A:** Speculative Generality is the smell of adding hooks, abstractions, or parameters for use cases that do not yet exist, under the assumption that they will be needed later. It directly violates YAGNI. The result is extra complexity (abstract classes with a single subclass, parameters that are always `null`, methods that are never called) that future maintainers must work around or understand. The fix: remove the unused generality; it can always be added back when a real use case emerges.

---

---
**Q48:** What are the "Extract Method" and "Inline Method" refactorings? When do you use each?

**A:** **Extract Method:** take a code fragment, give it a descriptive name, and replace the original with a call to the new method. Use when a block of code needs a comment to explain it, is duplicated, or is too long. **Inline Method:** the inverse — replace a trivially-simple method call with its body. Use when the method's name is no more informative than its body, or when excessive small methods make code harder to follow. The goal is always readability and appropriate level of abstraction.

---

---
**Q49:** What is "Replace Conditional with Polymorphism," and when should you apply it?

**A:** This refactoring replaces a conditional (if/switch) that branches on an object's type or state with polymorphic dispatch. Steps: create an abstract base class or interface with the branching behaviour as an abstract method; create a subclass for each branch; move branch code into the corresponding subclass. Apply it when the same conditional appears in multiple methods, when adding new cases requires modifying many places, or when the type-based logic grows complex. Do not apply it for simple, stable conditionals.

---

---
**Q50:** What is "Introduce Null Object," and how does it reduce conditional complexity?

**A:** Instead of returning `null` and requiring every caller to null-check, return a **Null Object** — an object that implements the expected interface but does nothing (or returns safe defaults). For example, a `NullCustomer` that returns `""` for `getName()` and `0.0` for `getDiscount()`. This eliminates scattered null-guard code and follows the "Tell, Don't Ask" principle. Be cautious: Null Object makes sense when "do nothing" is a valid, meaningful default — not when null signals an error that callers must handle explicitly.

---

---

## Study Tips

1. **Use spaced repetition.** On your first pass, mark every card you answered correctly on the first try. On subsequent sessions, focus on the marked-miss cards. Revisit the full deck after one week, then two weeks, then a month.

2. **Self-grade honestly.** Before flipping to the answer, write or say your answer out loud. Partial credit is fine but distinguish between "I knew it" and "I recognised it" — recognition is weaker than free recall.

3. **Apply cards to real code.** After each section, spend 10 minutes scanning a codebase you know for one example of each concept covered. Connecting abstract definitions to concrete code locks them in far more effectively than reading alone.

4. **Pair cards with design pattern flashcards.** Many smells (God Class, Switch Statement, Feature Envy) map directly to patterns that solve them (Strategy, Visitor, Move Method). Cross-referencing the two decks builds the "problem → solution" reflex that distinguishes experienced engineers.
