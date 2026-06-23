# Curriculum: LLD & Object-Oriented Design Mastery

> Complete syllabus, learning objectives, assessment rubrics, and certification criteria.

---

## Section 1: Course Overview

### Course Identity

| Field | Value |
|---|---|
| Title | Low-Level Design (LLD) & Object-Oriented Design Mastery |
| Target Audience | Software engineers (2-8 YOE) preparing for Senior roles |
| Duration — Full Track | 12 weeks (2-3 hours/day, weekdays + weekend assignments) |
| Duration — Intensive Track | 4 weeks (3-4 hours/day) |
| Total Estimated Hours | ~200 hours (full) / ~80 hours (intensive) |
| Language | Java (concepts apply to all OOP languages) |
| Format | Self-paced with structured milestones |

---

### Learning Objectives

Upon completing this course, you will be able to:

1. Apply all four OOP pillars (Encapsulation, Abstraction, Inheritance, Polymorphism) with production-level judgment, not just syntactic correctness
2. Distinguish between classes, interfaces, and abstract classes and choose appropriately for any design problem
3. Apply composition over inheritance as a default and articulate exactly when inheritance is justified
4. Detect all five SOLID principle violations in existing code and refactor to eliminate them
5. Design systems that are open for extension without modification (OCP) using strategy and template patterns
6. Write interfaces that enforce behavioral contracts suitable for Liskov Substitution
7. Apply DRY to eliminate knowledge duplication, not just textual duplication
8. Identify and reduce inappropriate coupling between components
9. Maximize cohesion within a single class or module
10. Read and draw Class, Sequence, Activity, and State UML diagrams at interview speed
11. Recognize which of the 23 GoF patterns applies to a given problem before opening a computer
12. Implement Creational patterns (Singleton, Factory Method, Abstract Factory, Builder, Prototype) with correct trade-off reasoning
13. Implement Structural patterns (Adapter, Decorator, Facade, Composite, Proxy, Bridge, Flyweight) and distinguish overlapping patterns
14. Implement Behavioral patterns (Strategy, Observer, Command, Chain of Responsibility, State, Template Method, Iterator, Mediator, Visitor, Memento) and explain when each earns its complexity
15. Decompose a natural-language requirement into entities, relationships, and behaviors within 10 minutes
16. Design extensible class hierarchies for all 17 case studies in the course, independently, in under 45 minutes
17. Complete a machine coding round problem producing clean, working, well-structured code within 90 minutes
18. Design systems for testability using dependency injection, interfaces, and seams
19. Write unit tests using Arrange-Act-Assert that validate behavior, not implementation
20. Produce a complete design artifact set: class diagram, sequence diagram, decision log, and implementation for a capstone project
21. Articulate design decisions under interview pressure, including tradeoffs and alternatives considered
22. Score 3+ on at least 40/50 items in the Interview Readiness Checklist

---

### Skills Acquired

**Design Skills**
- Object modeling from requirements
- Class hierarchy construction
- Interface design
- Responsibility assignment
- Abstraction selection
- Extensibility planning

**Pattern Skills**
- Pattern recognition from problem descriptions
- Pattern selection justification
- Pattern combination (composing multiple patterns)
- Pattern anti-pattern identification

**Communication Skills**
- UML diagram production and reading
- Design decision documentation
- Verbal design walkthrough under interview conditions
- Trade-off articulation

**Engineering Quality Skills**
- Testability design
- Code review literacy
- Technical debt identification
- Refactoring for design improvement

---

### Career Outcomes

Engineers who complete this course and internalize its framework consistently report:

- Passing LLD rounds at FAANG and tier-1 product companies
- Receiving Senior Engineer offers after previously failing at the design round
- Improved code review contributions — giving and receiving more substantive feedback
- Faster onboarding to new codebases due to pattern recognition
- Greater confidence when tasked with designing new services or subsystems

---

## Section 2: Module-by-Module Syllabus

---

### Module 00 — Foundation: OOP & Clean Code Fundamentals

**Duration**: 2 weeks (Weeks 1-2) | ~20 hours  
**Location**: [`00-foundation/`](./00-foundation/)

**Prerequisites**: Basic OOP syntax in any language. You can write a class and create an object.

#### Learning Objectives

1. Explain and demonstrate each OOP pillar beyond syntactic level — with behavioral and design implications
2. Choose between class, abstract class, and interface based on the design context
3. Apply composition over inheritance and justify the choice in code reviews
4. Design access modifiers to minimize public surface area
5. Write self-documenting code using Clean Code naming and function design principles
6. Identify and eliminate primitive obsession and anemic domain models
7. Model a real-world domain (3-5 entities) from a requirements paragraph without guidance

#### Topics Covered

**OOP Pillars Deep Dive**
- Encapsulation: what it really means (behavior bundling, not just getters/setters), invariant protection, information hiding
- Abstraction: hiding implementation complexity behind stable interfaces; abstraction levels; leaky abstractions
- Inheritance: IS-A relationships, code reuse vs. behavioral specialization, fragile base class problem
- Polymorphism: compile-time (overloading) vs. runtime (overriding), dynamic dispatch, substitutability

**Class, Abstract Class, Interface — Decision Framework**
- Interface: when you need a contract with no implementation bias
- Abstract class: when you have shared implementation and a template
- Concrete class: when you have a complete, standalone entity
- Default methods in interfaces and the impact on the decision
- Multiple inheritance of type vs. implementation

**Composition vs. Inheritance**
- Why "favor composition over inheritance" is in the Gang of Four book
- The fragile base class problem with concrete examples
- Composition patterns: has-a relationships, delegation
- When inheritance is still the right call (true IS-A, closed hierarchies, frameworks)
- Mixin pattern as a middle ground

**Access Modifiers & Information Hiding**
- Public vs. private vs. protected vs. package-private
- Designing the minimum viable public API for a class
- Builder pattern as a way to control complex construction exposure
- Immutability: final fields, defensive copying, value objects

**Clean Code Fundamentals**
- Naming: classes (noun), methods (verb + noun), booleans (is/has/can), variables (meaningful not abbreviated)
- Method design: single level of abstraction, command-query separation, small methods
- Comment philosophy: comments explain *why*, code explains *what*
- Code organization: field ordering, method ordering, related methods near each other

**Object Modeling from Requirements**
- Nouns → candidate classes
- Verbs → methods and responsibilities
- Adjectives → attributes
- Relationships: association, aggregation, composition, dependency
- Drawing the first draft class diagram

#### Assignments

1. **Anemic Model Refactoring**: Given an anemic `Order` class with all logic in a service, refactor it to move behavior into the domain model
2. **Composition Exercise**: Replace an inheritance hierarchy (`Animal → Dog → GuideDog`) with composition without losing any behavior
3. **Object Modeling Sprint**: Given a 200-word description of a library system, produce a class diagram with at least 6 entities, all relationships labeled, and all primary methods identified
4. **Clean Code Rewrite**: Given a 100-line class with code smell violations, rewrite it applying Clean Code principles; write a brief paragraph justifying each change

#### Assessment Criteria

| Criterion | Description | Weight |
|---|---|---|
| Pillars Demonstration | Can explain each pillar with a novel (non-example) scenario | 25% |
| Composition Decision | Correctly identifies composition vs. inheritance in 3/3 scenarios | 25% |
| Object Modeling | Produces a class diagram with correct relationships for a new domain | 30% |
| Clean Code | Rewrite scores < 3 violations on a peer review using a provided checklist | 20% |

#### Recommended Resources

- *Clean Code* — Robert C. Martin (Chapters 1-6, 9)
- *Head First Object-Oriented Analysis and Design* — McLaughlin, Pollice, West (Chapters 1-4)
- *Effective Java* — Joshua Bloch (Items 15-25 on classes and interfaces)

---

### Module 01 — SOLID Principles

**Duration**: 1.5 weeks (Weeks 2-3) | ~15 hours  
**Location**: [`01-solid-principles/`](./01-solid-principles/)

**Prerequisites**: Module 00 complete. You can comfortably model a domain in a class diagram.

#### Learning Objectives

1. State each SOLID principle precisely — not as a slogan but as a testable constraint
2. Identify all five violations in unfamiliar code you have not written
3. Refactor code that violates a principle to comply, without over-engineering
4. Explain the motivation behind each principle — the failure mode it prevents
5. Recognize that the principles are correlated: a violation of one often implies a violation of another
6. Apply SOLID principles during initial design (not just as refactoring tools)
7. Articulate when a principle is being intentionally violated for pragmatic reasons

#### Topics Covered

**Single Responsibility Principle (SRP)**
- Definition: a class should have one reason to change (not "one thing")
- Identifying stakeholders and change axes
- Common SRP violations: classes that do persistence + business logic + formatting
- Separation techniques: service layer extraction, value object extraction, utility delegation
- Granularity calibration: SRP does not mean "one method per class"
- God class anti-pattern and decomposition strategy

**Open/Closed Principle (OCP)**
- Definition: open for extension, closed for modification
- The underlying mechanism: depend on abstractions, not concretions
- Strategy pattern as the canonical OCP enabler
- Distinguishing good extension points from over-engineering (YAGNI tension)
- When to plan for extension vs. when to refactor when the time comes
- Plugin architectures and how OCP scales to module-level design

**Liskov Substitution Principle (LSP)**
- Definition: subtypes must be substitutable for their base types without altering program correctness
- Behavioral contracts (not just method signatures)
- Pre-condition strengthening and post-condition weakening violations
- The classic Rectangle/Square violation explained in depth
- Design by Contract basics: invariants, pre/post conditions
- Detecting LSP violations: methods that throw on subtype, instanceof checks, special-case subtype behavior

**Interface Segregation Principle (ISP)**
- Definition: no client should depend on methods it does not use
- Fat interface anti-pattern and its causes
- Role interfaces vs. header interfaces
- Interface splitting strategies: by client, by cohesion, by role
- ISP in Java: default methods, interface inheritance
- When to combine and when to split

**Dependency Inversion Principle (DIP)**
- Definition: high-level modules should not depend on low-level modules; both should depend on abstractions
- The "inversion" explained: who owns the interface
- DIP vs. Dependency Injection (DI implements DIP; DIP does not require DI)
- Constructor injection, method injection, property injection
- Inversion of Control containers: what they solve, when they're overkill
- Testing as the payoff: mocking high-level behavior requires DIP compliance

#### Assignments (located in [`01-solid-principles/assignments/`](./01-solid-principles/assignments/))

1. **Violation Hunt**: Given a 300-line e-commerce order processing class, identify and document every SOLID violation (at least one of each type is present)
2. **SRP Refactor**: Decompose a monolithic `ReportGenerator` class that handles data fetching, formatting, and emailing into compliant components
3. **OCP Design**: Design a payment processing system that supports Credit Card, PayPal, and Crypto without modifying the core payment engine for each new provider
4. **LSP Analysis**: Given four subtype pairs, identify which violate LSP and explain the precise contract violation
5. **DIP Transformation**: Refactor a `UserService` that directly instantiates `MySQLUserRepository` to depend on an abstraction; write a test that uses a mock repository

#### Assessment Criteria

| Criterion | Description | Weight |
|---|---|---|
| Violation Identification | Correctly identifies 90%+ of violations in the hunt exercise | 30% |
| Refactoring Quality | Refactored code has zero identifiable violations, is not over-engineered | 30% |
| LSP Precision | Can state the contract violation (not just "it feels wrong") | 20% |
| DIP + Testability | Refactored code can be unit-tested without modifying implementation | 20% |

#### Recommended Resources

- *Clean Architecture* — Robert C. Martin (Part III: Design Principles)
- *Agile Software Development* — Robert C. Martin (Chapters 7-11)
- SOLID principles articles by Mark Seemann (blog.ploeh.dk)

---

### Module 02 — Design Principles: DRY, KISS, YAGNI, Cohesion, Coupling, DI

**Duration**: 1.5 weeks (Weeks 3-4) | ~15 hours  
**Location**: [`02-design-principles/`](./02-design-principles/)

**Prerequisites**: Module 01. Solid understanding of SOLID.

#### Learning Objectives

1. Distinguish knowledge duplication (true DRY violation) from textual duplication (acceptable repetition)
2. Resist the urge to build for imagined future requirements (YAGNI) while still designing for known extensibility
3. Measure cohesion and coupling conceptually and identify when they are out of balance
4. Apply the Law of Demeter and explain what it prevents
5. Implement Dependency Injection in three forms and choose between them appropriately
6. Apply Tell, Don't Ask to eliminate decision-making spread across callers
7. Combine all principles in a holistic design review

#### Topics Covered

**DRY — Don't Repeat Yourself**
- The precise definition: "Every piece of knowledge must have a single, unambiguous, authoritative representation"
- Knowledge duplication vs. code duplication (two similar `for` loops may not be a DRY violation)
- The hidden duplication: business rules in multiple layers (validation in UI + service + DB)
- DRY and abstraction: extracting the right abstraction vs. forced unification
- When NOT to apply DRY: the wrong abstraction is worse than duplication (AHA principle — Avoid Hasty Abstractions)

**KISS — Keep It Simple, Stupid**
- Complexity as a design failure mode
- Accidental vs. essential complexity
- Over-engineering detection: abstractions with one implementor, factory for a single type, patterns where conditionals would suffice
- Code review heuristic: "Is this complexity justified by a real requirement?"

**YAGNI — You Aren't Gonna Need It**
- The cost of speculative generality: maintenance burden, cognitive overhead, delayed shipping
- YAGNI vs. OCP: designing extension points is not YAGNI if the extension is known
- Practical heuristic: "Is this requirement confirmed, or am I guessing?"
- Refactoring-friendly code as the alternative to speculative design

**High Cohesion**
- Functional cohesion: elements that work together to accomplish a single well-defined task
- Types of low cohesion: coincidental, logical, temporal, procedural cohesion (and why they're problems)
- Measuring cohesion: LCOM (Lack of Cohesion of Methods) metric conceptually
- Refactoring for cohesion: class splitting, method extraction

**Low Coupling**
- Types of coupling: content, common, control, stamp, data coupling — ranked from worst to best
- Coupling indicators: import count, method parameter count, direct field access across classes
- Reducing coupling: interfaces, events, dependency injection, mediators
- The coupling-cohesion balance: you cannot maximize both simultaneously — the design tradeoff

**Law of Demeter (Principle of Least Knowledge)**
- "Only talk to your immediate friends"
- Method chaining violations: `customer.getAddress().getCity().getCountry()`
- Tell, Don't Ask as the positive formulation
- Feature Envy code smell as a LoD violation signal
- When fluent interfaces are acceptable exceptions

**Dependency Injection**
- Constructor injection: preferred, enables immutability, makes dependencies explicit
- Method injection: for optional or operation-scoped dependencies
- Property injection: for framework-managed wiring (Spring, etc.) — and when to avoid it
- DI containers: what they solve (object graph construction), what they don't solve (design)
- Manual DI (Pure DI) for testability without a container

**Tell, Don't Ask**
- The anti-pattern: `if (account.getBalance() > amount) account.withdraw(amount)` — the caller makes the decision
- The fix: `account.withdraw(amount)` — the object enforces its own invariants
- Relationship to encapsulation: Tell, Don't Ask enforces behavioral encapsulation
- Where it breaks down: query methods are sometimes necessary (reporting, UI rendering)

#### Assignments (located in [`02-design-principles/assignments/`](./02-design-principles/assignments/))

1. **DRY Audit**: Given a multi-layer web application skeleton, find all knowledge duplication across layers and propose a canonical home for each business rule
2. **Coupling Reduction**: Refactor a tightly-coupled order fulfillment module to use interfaces and events; measure coupling before and after by counting cross-class references
3. **LoD Fix**: Given code with 5 LoD violations (identified), refactor each using Tell, Don't Ask
4. **DI Wiring**: Build a `NotificationService` with pluggable email, SMS, and push providers; wire it manually using constructor injection; write tests for each combination

#### Assessment Criteria

| Criterion | Description | Weight |
|---|---|---|
| DRY vs. Duplication | Correctly distinguishes knowledge vs. code duplication in 5/5 scenarios | 25% |
| Coupling Reduction | Measurably reduces inter-class references without over-abstracting | 30% |
| Tell Don't Ask | Correctly applies behavioral encapsulation in all LoD fixes | 25% |
| DI Implementation | DI wiring is complete, testable, and uses the appropriate injection form | 20% |

#### Recommended Resources

- *The Pragmatic Programmer* — Hunt & Thomas (DRY, KISS, Orthogonality chapters)
- *Growing Object-Oriented Software, Guided by Tests* — Freeman & Pryce
- *Working Effectively with Legacy Code* — Feathers (coupling and seam identification)

---

### Module 03 — UML & Design Modeling

**Duration**: 1 week (Week 4) | ~10 hours  
**Location**: [`03-uml-modeling/`](./03-uml-modeling/)

**Prerequisites**: Modules 00-02. You should be able to design a simple class hierarchy before learning to diagram it.

#### Learning Objectives

1. Read a class diagram and extract all relationship types, multiplicity, and visibility
2. Draw a class diagram for a 6-10 entity system within 15 minutes
3. Read a sequence diagram and trace a complete request-response flow
4. Draw a sequence diagram for a described use case workflow
5. Use activity and state diagrams where appropriate without defaulting to them for everything
6. Adopt an interview-speed shorthand notation that is expressive without being formally complete

#### Topics Covered

**Class Diagrams**
- Notation: class box, attributes (visibility, type), methods (visibility, parameters, return type)
- Relationships: association (uses), aggregation (has-a, shared lifetime), composition (has-a, owned lifetime), dependency (uses temporarily), inheritance (is-a), realization (implements)
- Multiplicity: 1, 0..1, *, 1..*, n..m
- Directionality: unidirectional vs. bidirectional associations
- Abstract classes and interfaces in UML
- Notes and constraints
- Practical shorthand: what to omit in interviews without losing communication

**Sequence Diagrams**
- Participants and lifelines
- Synchronous vs. asynchronous messages
- Return messages
- Self-calls (loops and recursion)
- Combined fragments: alt (if/else), opt (optional), loop, par (parallel)
- Activation boxes: showing when an object is executing
- Creating and destroying objects within a sequence

**Activity Diagrams**
- When to use: business processes, algorithm flows, multi-actor workflows
- Actions, decisions, forks (parallel), joins
- Swimlanes for multi-actor flows
- Start/end nodes

**State Machine Diagrams**
- When to use: objects with lifecycle (Order, Ticket, Connection)
- States, transitions, events, guards, actions
- Entry/exit actions
- Composite states (nested state machines)

**Use Case Diagrams**
- Actors, use cases, system boundary
- Include and extend relationships
- When use case diagrams add value (requirements communication) vs. when they don't (detailed design)

**UML for Interviews**
- The interview-speed workflow: start with nouns (entities), draw boxes, add relationships, add methods last
- What to say while you draw (verbal walkthrough technique)
- Common mistakes: over-specifying too early, forgetting multiplicities, missing interfaces
- Tools: paper/whiteboard notation vs. digital tools (draw.io, PlantUML syntax basics)

#### Exercises (located in [`03-uml-modeling/exercises/`](./03-uml-modeling/exercises/))

1. **Class Diagram Reading**: Given 3 class diagrams, answer 5 questions each about relationships and design intent
2. **Sequence Diagram Drawing**: Draw sequence diagrams for "User places an order" and "ATM cash withdrawal" flows
3. **Full System Diagram**: Given the Library Management System requirements from Module 00, produce a complete class diagram and two sequence diagrams (borrow book, return book)
4. **Speed Draw**: In 10 minutes, draw a class diagram for a Hotel Booking system from verbal description only

#### Assessment Criteria

| Criterion | Description | Weight |
|---|---|---|
| Relationship Accuracy | All relationship types, directions, and multiplicities correct | 40% |
| Sequence Completeness | Sequence diagrams capture all participants and message flows | 30% |
| Speed | Complete class diagram for a 6-entity system in < 15 minutes | 20% |
| Readability | Diagram is legible and organized without crossing lines | 10% |

---

### Module 04 — Design Patterns: Creational

**Duration**: Part of Week 5 | ~8 hours  
**Location**: [`04-design-patterns/creational/`](./04-design-patterns/creational/)

**Prerequisites**: Modules 00-03. Strong understanding of interfaces and composition.

#### Learning Objectives

1. Identify the object creation problem each creational pattern solves
2. Implement each of the five creational patterns from scratch without reference
3. Distinguish Factory Method from Abstract Factory and choose correctly
4. Implement a thread-safe Singleton and explain why naive implementations break
5. Apply Builder to eliminate telescoping constructors and improve API usability
6. Combine patterns: Builder + Factory, Prototype + Registry

#### Topics Covered

**Singleton**
- Intent: ensure a class has only one instance and provide global access
- Naive implementation and thread-safety problem
- Double-checked locking and the volatile keyword requirement
- Initialization-on-demand holder idiom (preferred in Java)
- Enum singleton (effective Java approach)
- Singleton vs. static class
- Why Singletons are often a design smell: global state, testability problems, hidden dependencies
- When Singletons are justified: truly global unique resources (configuration, registry, connection pool)

**Factory Method**
- Intent: define an interface for creating an object, but let subclasses decide which class to instantiate
- The creator/product class hierarchy
- Factory Method vs. simple factory (simple factory is not a GoF pattern)
- Factory Method as OCP enabler: adding new products without modifying creator
- Parameterized factory methods
- Real-world use: Java Collections.iterator(), Spring BeanFactory

**Abstract Factory**
- Intent: provide an interface for creating families of related or dependent objects without specifying concrete classes
- The product family concept
- Abstract Factory vs. Factory Method: one creates a single product hierarchy; the other creates a family
- Switching product families at runtime
- Real-world use: UI toolkit factories (Button/Checkbox/TextField for Windows vs. Mac)
- Extensibility limitation: adding a new product type requires changing the abstract factory interface

**Builder**
- Intent: separate the construction of a complex object from its representation
- Telescoping constructor anti-pattern and why it's unworkable beyond 4+ parameters
- Director + ConcreteBuilder + Product structure
- Fluent Builder (method chaining) — modern preferred form
- Immutable objects with Builder (all validation at `build()` time)
- Lombok `@Builder` and when code generation is appropriate
- When Builder is overkill (< 4 parameters, no optional fields)

**Prototype**
- Intent: specify the kinds of objects to create using a prototypical instance and create new objects by copying this prototype
- Shallow vs. deep copy and when each is appropriate
- Java's `Cloneable` and its problems; better alternatives (copy constructors, serialization)
- Prototype Registry: a map of pre-configured prototypes
- Real-world use: object creation when initialization is expensive

#### Assignments

1. Implement a thread-safe `ConfigurationManager` Singleton using the holder idiom; demonstrate it is the same instance across 3 threads
2. Build a `ShapeFactory` using Factory Method that supports Circle, Rectangle, Triangle; add a fourth shape without modifying the factory
3. Design an Abstract Factory for a cross-platform UI toolkit (Windows/Mac) with Button, TextField, and Checkbox products
4. Implement an immutable `Pizza` builder with required (size, crust) and optional (toppings) parameters; validate at `build()`
5. Build a Prototype Registry for pre-configured `DatabaseConnection` objects and demonstrate that clones are independent

#### Assessment Criteria

| Criterion | Description | Weight |
|---|---|---|
| Correct Intent | Implementation solves the right problem (not just syntactically correct) | 30% |
| OCP Compliance | Adding new products/variants does not require modifying existing classes | 30% |
| Thread Safety | Singleton is demonstrably thread-safe under concurrent access | 20% |
| Testability | Code is testable without modification (constructors injected, not hardcoded) | 20% |

---

### Module 04 — Design Patterns: Structural

**Duration**: Weeks 5-6 | ~10 hours  
**Location**: [`04-design-patterns/structural/`](./04-design-patterns/structural/)

**Prerequisites**: Creational patterns complete. Strong grasp of interfaces and delegation.

#### Learning Objectives

1. Apply Adapter to integrate incompatible interfaces without modifying either
2. Use Decorator to add behavior dynamically without inheritance explosion
3. Distinguish Decorator from Proxy and from Inheritance — critical interview question
4. Apply Facade to simplify a complex subsystem for a specific client
5. Use Composite to build tree structures that clients treat uniformly
6. Distinguish Flyweight from regular object sharing and know when memory optimization matters
7. Know the Bridge pattern and distinguish it from Adapter

#### Topics Covered

**Adapter**
- Intent: convert the interface of a class into another interface clients expect
- Object Adapter (composition-based, preferred) vs. Class Adapter (inheritance-based, Java-limited)
- Two-way adapters
- Adapter vs. Facade: Adapter translates one interface; Facade simplifies many interfaces
- Real-world use: Java InputStreamReader (adapts InputStream to Reader)

**Decorator**
- Intent: attach additional responsibilities to an object dynamically; an alternative to subclassing for extending functionality
- The wrapping mechanism: Decorator holds a reference to the component
- Stacking decorators: multiple behaviors, configurable at runtime
- When Decorator beats inheritance: when you have m behaviors that can combine in any of 2^m ways
- Java I/O streams as the canonical Decorator example
- Decorator vs. Proxy: Decorator adds behavior; Proxy controls access

**Facade**
- Intent: provide a unified interface to a set of interfaces in a subsystem
- Simplifying complex subsystems for specific client needs
- Facade does not seal the subsystem — clients can still access it directly
- Multiple Facades for different client types
- Real-world use: Spring's `JdbcTemplate` (facades over JDBC API)

**Composite**
- Intent: compose objects into tree structures to represent part-whole hierarchies; treat individual objects and compositions uniformly
- The Component/Leaf/Composite structure
- Uniformity of interface: client code does not need to distinguish leaves from branches
- Real-world use: file system, GUI widget trees, organizational hierarchies
- Composite vs. Decorator: Composite manages children; Decorator adds behavior

**Proxy**
- Intent: provide a surrogate or placeholder for another object to control access to it
- Types of proxies:
  - Virtual Proxy: defers expensive object creation (lazy initialization)
  - Protection Proxy: access control
  - Remote Proxy: local representative for a remote object (RMI, gRPC stubs)
  - Caching Proxy: caches results of expensive operations
  - Smart Reference: reference counting, locking
- Dynamic proxies in Java (java.lang.reflect.Proxy)
- Proxy vs. Decorator: Proxy controls access; Decorator adds behavior

**Bridge**
- Intent: decouple an abstraction from its implementation so that the two can vary independently
- The double hierarchy problem: without Bridge, m abstractions x n implementations = m*n classes
- With Bridge: m + n classes
- Real-world use: JDBC Driver API (abstraction = Statement; implementation = MySQLDriver, OracleDriver)
- Bridge vs. Adapter: Bridge is designed upfront; Adapter makes incompatible interfaces work after the fact

**Flyweight**
- Intent: use sharing to support large numbers of fine-grained objects efficiently
- Intrinsic state (shared, stored in Flyweight) vs. Extrinsic state (unique, passed by client)
- Flyweight Factory: manages the shared pool
- Real-world use: Java String pool, Integer cache (-128 to 127), character rendering in text editors
- When to use: millions of similar objects with significant memory footprint

#### Assignments

1. Build an Adapter that makes a third-party `XmlLogger` implement your application's `Logger` interface
2. Implement a Decorator chain for a `FileReader` that adds: BufferedReading, GzipDecompression, AESDecryption — in any configurable order
3. Design a Composite for a file system (File and Directory) where both respond to `getSize()` and `list()`
4. Implement a Virtual Proxy for a `HighResolutionImage` that loads from disk only when `display()` is first called
5. Design a Bridge for a `Shape` hierarchy (Circle, Rectangle) and `DrawingAPI` implementations (SVG, Canvas)

---

### Module 04 — Design Patterns: Behavioral

**Duration**: Weeks 6-7 | ~12 hours  
**Location**: [`04-design-patterns/behavioral/`](./04-design-patterns/behavioral/)

**Prerequisites**: Creational and Structural patterns. The behavioral patterns are the largest group and the most commonly misidentified.

#### Learning Objectives

1. Apply Strategy to eliminate conditional dispatch and enable algorithm swapping
2. Implement Observer/Event with proper decoupling and avoid memory leak pitfalls
3. Use Command to encapsulate operations, enabling undo/redo and queue-based execution
4. Build a Chain of Responsibility pipeline with configurable handler ordering
5. Model object lifecycle with State to eliminate state-checking conditionals
6. Apply Template Method to share algorithm structure while delegating variation
7. Distinguish Iterator, Mediator, Visitor, and Memento from structurally similar patterns

#### Topics Covered

**Strategy**
- Intent: define a family of algorithms, encapsulate each one, and make them interchangeable
- Eliminating switch/if-else on type for behavioral dispatch
- Passing strategy via constructor (preferred) vs. setter
- Composing strategies: strategy + factory for selection
- Real-world use: `java.util.Comparator`, sorting algorithms, pricing strategies, validation rules

**Observer**
- Intent: define a one-to-many dependency so that when one object changes state, all its dependents are notified automatically
- Push vs. Pull notification models
- Java's `java.util.Observable` (deprecated) and why; event listener pattern instead
- Memory leak pitfall: observers not unregistered (use WeakReferences or explicit unsubscribe)
- Observer vs. Event Bus: Observer is direct coupling; Event Bus is fully decoupled
- Real-world use: GUI event handling, stock ticker updates, MVC model notifications

**Command**
- Intent: encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations
- Command as a first-class object: store, queue, serialize
- Undo/Redo with Command history
- Macro commands (Composite + Command)
- Transaction log using Command queue
- Real-world use: text editor operations, UI button actions, job queues, database transactions

**Chain of Responsibility**
- Intent: avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request
- Handler chain: each handler either handles or passes along
- Configuring chains dynamically vs. statically
- Request filtering pipelines (web middleware, servlet filters)
- Chain of Responsibility vs. Command: CoR routes a request; Command encapsulates it
- Real-world use: Java servlet filters, middleware pipelines, logging frameworks

**State**
- Intent: allow an object to alter its behavior when its internal state changes; the object will appear to change its class
- State as a replacement for state-conditional code (`if (state == OPEN) ... else if (state == CLOSED) ...`)
- The Context holds state; delegates to State objects
- State transitions: managed by Context vs. managed by State objects
- Real-world use: Order lifecycle, TCP connection states, traffic light, vending machine

**Template Method**
- Intent: define the skeleton of an algorithm in a base class, deferring some steps to subclasses
- Hollywood Principle: "Don't call us, we'll call you" — the framework calls your hooks
- Abstract methods (required override) vs. hook methods (optional override)
- Template Method vs. Strategy: Template Method uses inheritance; Strategy uses composition
- Real-world use: `java.util.AbstractList`, Spring's `JdbcTemplate`, data processing pipelines

**Iterator**
- Intent: provide a way to access the elements of an aggregate object sequentially without exposing its underlying representation
- External vs. Internal iterators
- Java's `Iterable` and `Iterator` interfaces
- Fail-fast vs. fail-safe iterators
- Custom iterators for non-standard data structures (trees, graphs)

**Mediator**
- Intent: define an object that encapsulates how a set of objects interact, promoting loose coupling
- The N-to-N dependency problem without Mediator
- With Mediator: each colleague only knows the Mediator
- Mediator vs. Observer: Mediator has bidirectional awareness; Observer is one-directional
- Real-world use: chat room, air traffic control, form interaction management

**Visitor**
- Intent: represent an operation to be performed on elements of an object structure; lets you define a new operation without changing the classes of the elements
- Double dispatch mechanism
- The Open/Closed tradeoff: adding operations is easy (new Visitor); adding elements is hard (change all Visitors)
- When Visitor is appropriate vs. overkill
- Real-world use: AST traversal in compilers, DOM transformation, tax calculation over varied product types

**Memento**
- Intent: without violating encapsulation, capture and externalize an object's internal state so that the object can be restored to this state later
- Originator, Memento, Caretaker roles
- Wide vs. Narrow Memento interface
- Memento and undo history
- Real-world use: text editor undo, game save states, transaction rollback

#### Assignments

1. Refactor an `OrderProcessor` that has nested `if (paymentType == ...)` blocks using Strategy
2. Build an event system for a stock market simulator: stocks publish price changes; multiple subscribers (logger, alert, portfolio manager) react
3. Implement a Command-based text editor supporting: type character, delete character, undo, redo
4. Build an HTTP request filter chain with: AuthFilter, RateLimitFilter, LoggingFilter — configurable order
5. Model a vending machine using State pattern: idle, item-selected, payment-received, dispensing states
6. Design a tax calculation Visitor over a product hierarchy (Electronics, Food, Clothing) where tax rules differ by category

---

### Module 05 — Senior Engineer Design Thinking

**Duration**: 1.5 weeks (Weeks 7-8) | ~15 hours  
**Location**: [`05-senior-engineer-design-thinking/`](./05-senior-engineer-design-thinking/)

**Prerequisites**: All design patterns. This module synthesizes everything.

#### Learning Objectives

1. Decompose an ambiguous requirement into concrete entities, behaviors, and constraints within 10 minutes
2. Identify which patterns are applicable to a given design problem before writing any code
3. Evaluate a design for extensibility: which changes are easy, which require widespread modification
4. Apply abstraction at the right level — neither too early nor too late
5. Articulate design decisions with explicit tradeoff reasoning, not just "I chose X"
6. Recognize common anti-patterns and course-correct before they are embedded
7. Conduct a design review as a senior engineer: structured, constructive, technically precise

#### Topics Covered

**Requirement Decomposition**
- The 5-step decomposition process: Actors → Use Cases → Entities → Behaviors → Constraints
- Extracting nouns (entities), verbs (behaviors), and adjectives (attributes)
- Asking the right clarifying questions: scope, scale, concurrency, consistency requirements
- Separating functional requirements from non-functional requirements
- Drawing the first rough class diagram from verbal requirements in real time

**Extensibility Planning**
- Identifying dimensions of variation (what will change?)
- Protecting stable core from volatile periphery
- Extension points: interfaces, abstract methods, hooks, events
- The "change without modification" test for each component
- Case study: designing a notification system to support 5 channels now and unknown channels later

**Abstraction Selection**
- The abstraction ladder: concrete → type → role → capability
- Abstracting too early: the wrong abstraction forces bad designs downstream
- Abstracting too late: everything is coupled to concretions
- The AHA principle: wait for the second use case before abstracting
- Naming abstractions: what makes a good interface name (role-based, not implementation-based)

**Change Isolation Strategies**
- Package/module boundaries as change firewalls
- Anti-Corruption Layer pattern from Domain-Driven Design
- Adapter as a change isolation tool (insulate from third-party API changes)
- Feature flags as runtime isolation
- Versioned interfaces for backward-compatible evolution

**Design Tradeoffs**
- Flexibility vs. simplicity: every abstraction has a cost
- Inheritance vs. composition: when the hierarchy is small and stable vs. when it will grow
- Synchronous vs. asynchronous: latency vs. decoupling
- Consistency vs. availability in object design (not just distributed systems)
- How senior engineers communicate tradeoffs: "I chose X over Y because Z, and I accept the cost of W"

**API Design Philosophy**
- Designing interfaces that clients want to use
- The Principle of Least Surprise
- Fluent API design and when it helps vs. hurts readability
- Versioning strategy for interfaces used by external clients
- The "delete the implementation, keep the interface" test for abstraction quality

**The Design Review Mindset**
- What senior engineers look for: responsibility assignment, coupling, extensibility, testability
- How to give a design review: praise + question + suggest (not just criticize)
- Self-review checklist: before you present a design to the team
- Red flags that prompt deeper investigation: deep inheritance, many-argument constructors, static dependencies, god objects

---

### Module 06 — Real-World Case Studies

**Duration**: 3 weeks (Weeks 8-10) | ~35 hours  
**Location**: [`06-case-studies/`](./06-case-studies/)

**Prerequisites**: All prior modules. Case studies are the application layer of the entire course.

#### Learning Objectives

1. Apply end-to-end design process independently for a new problem statement
2. Produce class diagrams, sequence diagrams, and implementation for each case study
3. Justify pattern choices explicitly (not just "I used Strategy here")
4. Recognize which case studies appear most frequently in interviews and prioritize accordingly
5. Design for extension: each case study includes extension problems to be solved without modifying the base design

#### Case Studies by Difficulty

**Easy** (Week 8)
- `01-parking-lot/` — Classic. Spot types, vehicle types, ticketing, payment. Tests: Singleton (lot), Strategy (pricing), Factory (spot allocation)
- `02-library-management/` — Book catalog, membership, borrowing, fines. Tests: Composite (catalog), Observer (due date notification)
- `03-vending-machine/` — Item catalog, payment, dispense, change. Tests: State (machine states), Strategy (payment methods)

**Medium** (Week 9)
- `01-atm-system/` — Card, PIN, withdrawal, balance, transfer. Tests: State, Command (transaction log), Chain of Responsibility (auth pipeline)
- `02-hotel-booking/` — Rooms, reservations, pricing, check-in/check-out. Tests: Strategy (pricing), Factory (room type), Observer (availability)
- `03-food-delivery/` — Restaurants, menus, orders, delivery tracking. Tests: Builder (order), Observer (tracking), Strategy (delivery assignment)
- `04-movie-ticket-booking/` — Shows, seats, reservations, payment. Tests: Composite (seat sections), Decorator (pricing add-ons), Proxy (seat lock during booking)
- `05-inventory-management/` — Products, stock levels, reorder, supplier. Tests: Observer (low stock alerts), Chain of Responsibility (approval workflow)

**Advanced** (Week 10)
- `01-ride-sharing/` — Drivers, riders, trip matching, pricing, live tracking. Tests: Strategy (matching, pricing), Observer (location updates), State (trip lifecycle)
- `02-ecommerce-platform/` — Products, cart, checkout, orders, fulfillment. Tests: Builder (order), Decorator (discounts/tax), Chain of Responsibility (fraud checks)
- `03-splitwise/` — Groups, expenses, splits, settlements. Tests: Strategy (split algorithms), Observer (balance updates)
- `04-notification-framework/` — Multi-channel (email/SMS/push), template, routing, retry. Tests: Strategy (channel), Template Method (rendering), Chain of Responsibility (retry)
- `05-payment-gateway/` — Multi-provider, fraud detection, retry, idempotency. Tests: Adapter (provider integration), Chain of Responsibility (fraud pipeline), Command (transaction)
- `06-rate-limiter/` — Token bucket, sliding window, per-key limits. Tests: Strategy (algorithm), Decorator (limit stacking), Proxy (transparent rate limiting)
- `07-logging-framework/` — Log levels, handlers, formatters, appenders. Tests: Chain of Responsibility (level filtering), Decorator (formatting), Strategy (output)
- `08-cache-system/` — LRU/LFU/TTL eviction, put/get/evict, multi-level. Tests: Strategy (eviction), Decorator (multi-level), Observer (eviction events)
- `09-workflow-engine/` — DAG of tasks, conditional branching, retry, parallel execution. Tests: Composite (task graph), Strategy (execution), Command (task), State (workflow state)

#### Case Study Assessment

Each case study is self-assessed using:
- [ ] Class diagram complete with all entities and relationships
- [ ] At least 2 sequence diagrams (happy path + error path)
- [ ] Implementation compiles and passes basic scenarios
- [ ] Pattern choices explicitly documented with rationale
- [ ] At least one extension problem attempted

---

### Module 07 — Machine Coding Round Preparation

**Duration**: 2 weeks (Weeks 10-11) | ~20 hours  
**Location**: [`07-machine-coding/`](./07-machine-coding/)

**Prerequisites**: All case studies. Machine coding is applied case study skill under time pressure.

#### Learning Objectives

1. Complete a machine coding problem producing working code in under 90 minutes
2. Apply the 10-minute design phase before writing any code
3. Produce clean, readable code that demonstrates design thinking, not just correctness
4. Manage scope under time pressure: implement core flow first, extensions later
5. Communicate intent through naming and structure, not comments alone

#### Topics Covered

**The Machine Coding Round Format**
- Typical duration: 60-90 minutes
- Evaluation criteria: working code, design quality, code readability, test coverage
- What interviewers actually read: entry points, class names, method names, test names

**The 10-Minute Design Protocol**
- Step 1 (2 min): Read problem statement. Identify core entities.
- Step 2 (3 min): List use cases. Identify the happy path flow.
- Step 3 (3 min): Rough class sketch. Identify patterns if any.
- Step 4 (2 min): Scope decision. What is core? What is extension?

**Implementation Strategy**
- Start from the domain model, not the entry point
- Write interfaces before implementations
- Implement the happy path end-to-end first
- Add error handling and edge cases only after core flow works
- Test as you go, not at the end

**Common Traps**
- Premature optimization
- Implementing features that weren't asked for (YAGNI)
- Skipping the design phase and coding immediately
- Overengineering with unnecessary patterns
- Not testing (even basic print-based validation)

#### Questions (in [`07-machine-coding/questions/`](./07-machine-coding/questions/))

Timed problem statements include:
- Snake and Ladder game
- Car rental system
- In-memory key-value store with TTL
- Task management system with priority
- Social media feed (follow, post, timeline)
- Tic-tac-toe with configurable board
- Elevator controller
- Online auction system

#### Evaluation Rubric

| Dimension | 5 (Excellent) | 3 (Adequate) | 1 (Poor) |
|---|---|---|---|
| Correctness | All cases pass | Core flow works, edge cases fail | Core flow broken |
| Design | Patterns used appropriately, SOLID compliant | Minor violations, mostly sound | God class, no abstraction |
| Readability | Self-documenting, clean naming | Readable with effort | Requires explanation |
| Tests | Meaningful tests for core scenarios | Happy path only | No tests |
| Scope | Core complete + 1 extension | Core complete | Core incomplete |

---

### Module 08 — Testing & Quality Engineering

**Duration**: 1 week (Week 11) | ~10 hours  
**Location**: [`08-testing-quality/`](./08-testing-quality/)

**Prerequisites**: All prior modules.

#### Learning Objectives

1. Design classes and systems so they are testable without modification
2. Write unit tests using Arrange-Act-Assert with meaningful test names
3. Choose what to mock and what to test with real collaborators
4. Apply TDD's Red-Green-Refactor cycle to a design problem
5. Measure and reason about test coverage appropriately

#### Topics Covered

**Designing for Testability**
- Dependency injection as the primary testability enabler
- Avoiding untestable patterns: static methods, singletons with state, `new` in methods
- Seams: points where you can substitute behavior for testing
- The Humble Object pattern: keeping untestable dependencies at the boundary

**Unit Testing Principles**
- Arrange-Act-Assert: structure every test
- Test naming: `methodName_stateUnderTest_expectedBehavior` convention
- Test isolation: one logical assertion per test
- The FIRST properties: Fast, Independent, Repeatable, Self-validating, Timely
- Boundary value analysis and equivalence partitioning

**Mocking and Stubbing**
- Stubs: provide pre-programmed responses (for queries)
- Mocks: verify interactions (for commands)
- When to mock: external systems, slow operations, non-deterministic behavior
- When NOT to mock: simple value objects, pure functions, domain logic

**Test-Driven Development for Design**
- TDD's design benefit: writing tests first forces interface clarity
- Red-Green-Refactor applied to a design problem
- TDD and the single responsibility: hard-to-test code is often poorly designed

---

### Module 09 — Capstone Projects

**Duration**: 1 week (Week 12) | ~20 hours  
**Location**: [`09-capstone-projects/`](./09-capstone-projects/)

**Prerequisites**: Everything. Capstone is summative.

#### Learning Objectives

1. Apply the full design process independently for a portfolio-quality problem
2. Produce all design artifacts: class diagram, sequence diagrams, decision log, working code, tests
3. Demonstrate extensibility by successfully implementing a planned extension without modifying core
4. Discuss and defend all design decisions with tradeoff reasoning

#### Assessment

Full rubric in Section 3 of this document. Capstone projects must meet all five criteria to be considered complete.

---

### Module 10 — Interview Preparation

**Duration**: Ongoing | ~10 hours structured + continuous practice  
**Location**: [`10-interview-preparation/`](./10-interview-preparation/)

#### Topics Covered

- Interview format and evaluation criteria decoded
- The verbal design walkthrough: how to present while drawing
- Handling ambiguous requirements in an interview setting
- Recovering from design mistakes mid-interview
- Behavioral questions about design decisions
- 100+ question bank with difficulty ratings
- Mock interview scripts and evaluation forms

---

### Module 11 — Artifacts

**Location**: [`11-artifacts/`](./11-artifacts/)

Reference materials to use throughout the course and in interviews:
- Pattern quick-reference cards (problem → pattern mapping)
- SOLID principles violation checklist
- Design principles checklist
- UML notation reference
- Interview formula cards (per-problem templates)
- Flashcards for pattern recognition

---

## Section 3: Assessment Strategy

### Assignment Rubric (General)

All assignments are evaluated on five dimensions:

| Dimension | Description |
|---|---|
| **Correctness** | Does the design/code solve the stated problem? Does it handle the given scenarios? |
| **Principle Compliance** | Are SOLID and design principles applied correctly? Are violations absent? |
| **Pattern Appropriateness** | Are patterns used because they solve a real problem, not for their own sake? |
| **Extensibility** | Can the design accommodate a described extension without major modification? |
| **Clarity** | Is the code readable, well-named, and self-documenting? |

Passing threshold: 3/5 or above on all dimensions, and 4/5+ on at least 3.

---

### Code Review Criteria

During peer code reviews (and self-reviews), evaluate:

**Responsibility**
- Does each class have a clear, singular responsibility?
- Is responsibility assignment obvious from names alone?

**Coupling**
- How many classes does this class depend on?
- Are dependencies on abstractions or concretions?

**Cohesion**
- Do all methods in this class operate on the same data?
- Would removing any method break the class's coherence?

**Extensibility**
- If I need to add a new type/behavior, where does the code change?
- Is that change localized or spread across the codebase?

**Testability**
- Can I test this class without a database, network, or file system?
- Are dependencies injectable?

---

### Machine Coding Evaluation Framework

Used in Module 07 and mock interviews:

**Before submission, self-check**:
- [ ] Does the code compile without errors?
- [ ] Does the core happy path work end-to-end?
- [ ] Are class names meaningful nouns?
- [ ] Are method names meaningful verb-noun pairs?
- [ ] Is any single class over 200 lines? (If so, consider splitting)
- [ ] Is there at least one test for the primary use case?
- [ ] Are there any obvious null pointer risks?
- [ ] Did I use at least one design pattern where appropriate?

**Interviewer evaluation dimensions** (scored 1-5 each):
- Correctness of output
- Design quality (responsibility, coupling)
- Code readability
- Test presence and meaningfulness
- Completeness (core flow + handling failures)

---

### Capstone Project Evaluation

| Criterion | Requirement | Failing Condition |
|---|---|---|
| Class Diagram | All entities with relationships, multiplicity, visibility | Missing entities or relationships |
| Sequence Diagrams | Minimum 3: happy path, error path, one complex flow | Fewer than 2 or missing a flow |
| Decision Log | 5+ design decisions with alternatives considered and tradeoffs stated | Decisions without rationale |
| Implementation | Compiles, core scenarios work, demonstrates patterns in use | Does not compile; no patterns identifiable |
| Tests | Unit tests for core behaviors (not just happy path) | No tests or only print-statement verification |
| Extension | At least one described extension implemented without modifying core classes | Extension requires modifying core |

---

## Section 4: Learning Outcomes per Level

### After Week 4 — Foundation Level

You should be able to:
- Design a class hierarchy for a simple system (Parking Lot, Library) independently
- Apply all 5 SOLID principles and identify violations in code you read
- Apply DRY, KISS, YAGNI, and coupling/cohesion principles
- Draw a class diagram and a sequence diagram for a described flow
- Write clean, readable OOP code with appropriate method and class granularity

You are NOT yet expected to:
- Select design patterns confidently
- Design complex systems with many interacting components
- Complete machine coding rounds without preparation

---

### After Week 8 — Intermediate Level

You should be able to:
- Recognize which of the 23 GoF patterns applies to a problem
- Implement any pattern from scratch given a problem statement
- Design systems for all Easy and Medium case studies independently
- Complete machine coding rounds for moderate-complexity problems in 90 minutes
- Conduct a structured design review using the five criteria dimensions

You are NOT yet expected to:
- Design Advanced case studies independently without reference
- Handle highly ambiguous requirements with confidence
- Produce portfolio-quality design artifacts in under 45 minutes

---

### After Week 12 — Advanced/Interview-Ready Level

You should be able to:
- Design any of the 17 case studies independently in under 45 minutes with class diagram + core sequence diagrams
- Complete machine coding rounds for complex problems in under 90 minutes
- Explain every design decision with explicit tradeoff reasoning
- Produce a complete design artifact set for a capstone project
- Score 3+ on at least 40 of 50 items in the Interview Readiness Checklist
- Pass LLD rounds at Senior Engineer level at product companies

---

## Section 5: Certification Criteria

### What You Must Demonstrate

Course completion ("certification") requires demonstrated mastery across five dimensions:

**1. Foundations** (Modules 00-02)
- All module assignments complete with passing rubric scores
- Module 03 UML exercises complete

**2. Patterns** (Module 04)
- All three pattern sub-modules complete with working implementations
- Pattern identification drill: given 10 code snippets, correctly identify pattern in 8/10

**3. Case Studies** (Module 06)
- All Easy and Medium case studies: complete with class diagram + implementation
- At least 3 Advanced case studies: complete with full artifact set (class diagram + 2 sequence diagrams + decision log + implementation)

**4. Machine Coding** (Module 07)
- 3 timed machine coding problems completed in < 90 minutes each
- All three score 3+ on all five evaluation dimensions

**5. Capstone** (Module 09)
- One capstone project meeting all six capstone evaluation criteria

---

### Passing Thresholds

| Module | Minimum Score |
|---|---|
| 00-02 Assignments | 3/5 on all dimensions, 4/5 on 3+ dimensions |
| 03 UML Exercises | 80% relationship accuracy |
| 04 Pattern Implementations | Correct intent + OCP compliance + testability |
| 06 Case Studies | Class diagram + implementation for all required studies |
| 07 Machine Coding | 3/5 minimum on all dimensions for all 3 problems |
| 09 Capstone | All 6 capstone criteria met |
| 10 Interview Checklist | 40/50 items scored 3 or above |

---

### Portfolio Requirements

Upon completion, your portfolio should contain:

1. **Module implementations**: Working code for all pattern implementations (at minimum: 2 creational, 3 structural, 4 behavioral)
2. **Case study artifacts**: Full artifact sets for 3+ case studies (class diagram + sequence diagrams + decision log + code)
3. **Machine coding solutions**: 3 annotated machine coding solutions with time stamps
4. **Capstone project**: Complete design artifact set for a portfolio-quality system
5. **Interview Readiness Checklist**: Self-rated with honest gap identification and remediation notes

A complete portfolio demonstrates to interviewers (and yourself) that you have moved from "knowing about" design to "being able to" design.
