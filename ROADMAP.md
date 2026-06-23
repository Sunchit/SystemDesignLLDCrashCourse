# Learning Roadmap: LLD & Object-Oriented Design Mastery

> Week-by-week plans, daily schedules, skill progression maps, and the interview readiness checklist.

---

## Section 1: 12-Week Full Learning Plan

> **Commitment**: 2-3 hours/day on weekdays. Weekend: 4-6 hours for assignments and case studies.
> **Total**: ~200 hours over 12 weeks.

---

### Week 1 — OOP Foundations, Part I

**Theme**: Moving from OOP syntax to OOP judgment.

| Day | Topics | Time | Deliverable |
|---|---|---|---|
| Monday | OOP Pillars: Encapsulation deep dive — invariant protection, information hiding beyond getters/setters | 2.5h | Refactor a `BankAccount` class to enforce its invariants through method behavior, not setter exposure |
| Tuesday | OOP Pillars: Abstraction — what to hide, what to expose; the leaky abstraction problem | 2h | Write `Shape` with a stable interface hiding all rendering details |
| Wednesday | OOP Pillars: Inheritance — IS-A vs. HAS-A, fragile base class, when it earns its cost | 2.5h | Identify 3 inheritance abuses in a given codebase and document why each is fragile |
| Thursday | OOP Pillars: Polymorphism — dynamic dispatch, substitutability, overloading vs. overriding | 2h | Implement a polymorphic `Animal.speak()` hierarchy; add a new animal without touching existing code |
| Friday | Class vs. Abstract Class vs. Interface — decision framework; default methods; multiple inheritance of type | 2.5h | Given 6 design scenarios, choose the right construct and justify in writing |

**Weekend Assignment**: Object modeling sprint. Given a 300-word food delivery app description, produce: (1) a candidate entity list with attributes, (2) a full class diagram with relationships and multiplicities, (3) a brief rationale paragraph for each major design decision. Do not reference any course material — this is a calibration baseline.

**Week 1 Milestone**: You can look at a requirements paragraph and produce a rough class diagram in under 20 minutes.

---

### Week 2 — OOP Foundations, Part II + SOLID Introduction

**Theme**: Composition, Clean Code, and the first two SOLID principles.

| Day | Topics | Time | Deliverable |
|---|---|---|---|
| Monday | Composition vs. Inheritance — the "favor composition" rule, delegation, mixin pattern | 2.5h | Replace a 3-level inheritance hierarchy with composition; verify behavior is identical |
| Tuesday | Access modifiers, immutability, value objects, defensive copying | 2h | Design an immutable `Money` value object with correct equality and defensive copy in all constructors |
| Wednesday | Clean Code: naming (classes, methods, booleans, variables), function design, single level of abstraction | 2.5h | Rewrite a 100-line procedural `ReportGenerator` using Clean Code principles; document each change |
| Thursday | SOLID — Single Responsibility Principle: one reason to change, stakeholder-axis thinking | 2h | Given a god-class `UserManager` (handles DB, email, logging, validation), decompose into SRP-compliant components |
| Friday | SOLID — Open/Closed Principle: open for extension, closed for modification; plugin architecture concept | 2.5h | Design a tax calculator that supports 5 tax regimes without modification when adding a sixth |

**Weekend Assignment**: Module 00 Assignments 1-3 (Anemic Model Refactoring, Composition Exercise, Object Modeling Sprint). Grade yourself using the rubric in [`CURRICULUM.md`](./CURRICULUM.md). Note where you scored below 3/5.

**Week 2 Milestone**: You can decompose a class by responsibility and explain *why* each responsibility belongs in a separate class. You can state SRP and OCP precisely and identify violations.

---

### Week 3 — SOLID Principles (Completion) + Design Principles

**Theme**: LSP, ISP, DIP, then DRY, KISS, YAGNI, Cohesion, Coupling.

| Day | Topics | Time | Deliverable |
|---|---|---|---|
| Monday | SOLID — Liskov Substitution Principle: behavioral subtyping, contract design, Rectangle/Square problem | 2.5h | Given 4 subtype pairs, identify which violate LSP; for each violation, redesign to eliminate it |
| Tuesday | SOLID — Interface Segregation Principle: fat interfaces, role interfaces, splitting strategies | 2h | Decompose a `Printer` interface (`print`, `scan`, `fax`, `email`) into role interfaces; rewire two client classes |
| Wednesday | SOLID — Dependency Inversion Principle: inversion of ownership, DIP vs. DI, testing payoff | 2.5h | Refactor a `NotificationService` that instantiates `SMTPEmailSender` directly; make it testable with a mock sender |
| Thursday | DRY and KISS: knowledge vs. code duplication; essential vs. accidental complexity; wrong abstraction cost | 2h | DRY Audit assignment: find knowledge duplication in a multi-layer skeleton; propose canonical locations |
| Friday | YAGNI, Cohesion, Coupling: speculative generality cost; LCOM; coupling types; coupling reduction techniques | 2.5h | Coupling Reduction assignment: refactor a 5-class module with measured before/after coupling |

**Weekend Assignment**: SOLID Violation Hunt (Module 01 Assignment 1) — find all violations in the 300-line `OrderProcessor` class. Then complete Module 01 Assignment 5 (DIP Transformation). Write tests that prove the refactored code is testable.

**Week 3 Milestone**: You can read unfamiliar code and identify SOLID violations with precision. You can distinguish knowledge duplication from code duplication.

---

### Week 4 — Design Principles (Completion) + UML Modeling

**Theme**: Finishing design principles, then full UML fluency.

| Day | Topics | Time | Deliverable |
|---|---|---|---|
| Monday | Law of Demeter, Tell Don't Ask, Dependency Injection (all three forms) | 2.5h | LoD Fix assignment: refactor 5 LoD violations; DI Wiring: build pluggable NotificationService |
| Tuesday | UML Class Diagrams: notation, all relationship types, multiplicity, visibility | 2h | Draw class diagrams for 3 given domain descriptions; check against answer key |
| Wednesday | UML Sequence Diagrams: participants, message types, combined fragments (alt, opt, loop) | 2.5h | Draw sequence diagrams for "User places order" and "ATM withdrawal" |
| Thursday | UML Activity and State diagrams: when to use each, notation | 2h | Draw a state machine for an `Order` (pending → confirmed → shipped → delivered → returned) |
| Friday | UML speed practice: complete class diagram for a verbal description in < 15 minutes | 2.5h | Speed Draw: Hotel Booking System in 10 minutes; compare with reference |

**Weekend Assignment**: Full UML artifact set for Library Management System — class diagram + 2 sequence diagrams (borrow book, return book) + state diagram (book loan lifecycle). This is also your first timed self-assessment: set a 45-minute timer and design without reference.

**Week 4 Milestone (Foundation Checkpoint)**: You have earned the Foundation Badge. You can design a simple system, apply all design principles, and produce class and sequence diagrams. This is the minimum bar for interview-relevant design skill.

---

### Week 5 — Design Patterns: Creational + Structural (Part I)

**Theme**: The five creational patterns and the first four structural patterns.

| Day | Topics | Time | Deliverable |
|---|---|---|---|
| Monday | Singleton (thread-safe), Factory Method (intent, when to use) | 2.5h | Thread-safe ConfigurationManager Singleton + ShapeFactory with Factory Method |
| Tuesday | Abstract Factory (product families), Builder (telescoping constructor elimination) | 2h | Cross-platform UI toolkit Abstract Factory; immutable Pizza Builder |
| Wednesday | Prototype (shallow vs. deep copy, registry), creational pattern comparison | 2.5h | DatabaseConnection Prototype Registry; write a comparison table: which creational pattern for which problem |
| Thursday | Adapter (object vs. class), Decorator (wrapping, behavior stacking) | 2h | XmlLogger Adapter; 3-layer FileReader Decorator chain (Buffer + Gzip + AES) |
| Friday | Facade (subsystem simplification), Composite (tree uniformity) | 2.5h | JDBC-wrapping Facade; FileSystem Composite with `getSize()` and `list()` |

**Weekend Assignment**: Parking Lot System (Easy Case Study 01). Complete independently: class diagram, sequence diagram for "vehicle enters and parks", implementation, at least one identified pattern with rationale. Set a 45-minute design timer before implementation.

**Week 5 Milestone**: You can implement any creational pattern from a problem statement. You can apply Adapter, Decorator, Facade, and Composite correctly.

---

### Week 6 — Design Patterns: Structural (Completion) + Behavioral (Part I)

**Theme**: Proxy, Bridge, Flyweight, then Strategy, Observer, Command.

| Day | Topics | Time | Deliverable |
|---|---|---|---|
| Monday | Proxy (virtual, protection, caching, remote), Proxy vs. Decorator distinction | 2.5h | Virtual Proxy for HighResolutionImage; write a 200-word "Proxy vs. Decorator" comparison |
| Tuesday | Bridge (double hierarchy problem, designed upfront), Flyweight (intrinsic/extrinsic state) | 2h | Shape + DrawingAPI Bridge; Character flyweight for text editor |
| Wednesday | Strategy: interchangeable algorithms, eliminating conditional dispatch | 2.5h | OrderProcessor Strategy refactor (replace nested if/else); sorting comparator examples |
| Thursday | Observer: push vs. pull, memory leak pitfall, Observer vs. Event Bus | 2h | Stock market simulator with Logger, AlertManager, PortfolioManager observers |
| Friday | Command: encapsulating requests, undo/redo, queue execution | 2.5h | Text editor with type/delete/undo/redo using Command history |

**Weekend Assignment**: Library Management System (Easy Case Study 02) + Vending Machine (Easy Case Study 03). Both independently. Class diagram + implementation + identified patterns.

**Week 6 Milestone**: You have completed all structural patterns. You can implement Strategy, Observer, and Command from a problem description.

---

### Week 7 — Design Patterns: Behavioral (Completion) + Senior Thinking

**Theme**: Finishing the behavioral patterns, then the synthesis module.

| Day | Topics | Time | Deliverable |
|---|---|---|---|
| Monday | Chain of Responsibility (filter pipelines), State (behavior by state) | 2.5h | HTTP filter chain (Auth + RateLimit + Logging); Vending Machine State pattern |
| Tuesday | Template Method (algorithm skeleton), Iterator (external vs. internal) | 2h | Data import pipeline with Template Method; custom tree Iterator |
| Wednesday | Mediator (N-to-N dependency reduction), Visitor (double dispatch, add operations without changing elements) | 2.5h | Chat room Mediator; Tax calculation Visitor over product hierarchy |
| Thursday | Memento (snapshot and restore), pattern comparison and recognition drill | 2h | Text editor Memento for undo; Pattern Recognition drill: identify patterns in 10 code snippets |
| Friday | Senior Engineer Design Thinking: requirement decomposition, extensibility planning | 2.5h | Given a notification system spec, produce: entity list, extension points, initial class hierarchy |

**Weekend Assignment**: ATM System (Medium Case Study 01) — full artifact set: class diagram, 2 sequence diagrams, decision log with patterns identified and justified, implementation.

**Week 7 Milestone (Pattern Navigator Badge)**: You have completed all 23 GoF patterns. You can identify the applicable pattern for a described problem in under 5 minutes.

---

### Week 8 — Senior Design Thinking + Medium Case Studies

**Theme**: Design judgment and application.

| Day | Topics | Time | Deliverable |
|---|---|---|---|
| Monday | Abstraction selection, change isolation, API design philosophy | 2.5h | Review your ATM design — identify 3 places where abstraction level is wrong; correct them |
| Tuesday | Design tradeoffs: flexibility vs. simplicity, inheritance vs. composition at system scale | 2h | Write a 300-word design tradeoff analysis for a given scenario |
| Wednesday | Hotel Booking System (Medium Case Study 02) — design phase (45 min timer) | 2.5h | Class diagram + sequence diagrams + decision log |
| Thursday | Hotel Booking System — implementation | 2h | Working implementation with core scenarios covered |
| Friday | Food Delivery System (Medium Case Study 03) — design phase | 2.5h | Class diagram + sequence diagrams |

**Weekend Assignment**: Movie Ticket Booking (Medium 04) + Inventory Management (Medium 05). Both complete with full artifacts.

**Week 8 Milestone**: You can independently design a medium-complexity system in under 45 minutes. You can articulate tradeoffs explicitly.

---

### Week 9 — Advanced Case Studies, Part I

**Theme**: The hard problems. Ride-sharing, E-commerce, Splitwise.

| Day | Topics | Time | Deliverable |
|---|---|---|---|
| Monday | Ride-Sharing System design (45 min timer) | 2.5h | Class diagram + 3 sequence diagrams (request trip, driver matching, trip completion) |
| Tuesday | Ride-Sharing implementation — core: rider/driver/trip entities, matching logic | 2.5h | Working implementation: trip request, match, complete |
| Wednesday | E-commerce Platform design (45 min timer) | 2.5h | Class diagram + sequence diagrams (add to cart, checkout, payment, order confirmation) |
| Thursday | Splitwise / Expense Sharing design + implementation | 2.5h | Group, expense, split algorithms (equal/exact/percent/share), settlement calculation |
| Friday | Notification Framework design — multi-channel, template, routing | 2.5h | Class diagram + extensibility analysis: how to add a new channel without modification |

**Weekend Assignment**: Payment Gateway (Advanced 05) — design + implementation. Focus on: multi-provider adapter, fraud detection chain, idempotency handling, retry logic.

---

### Week 10 — Advanced Case Studies, Part II + Machine Coding Introduction

**Theme**: Logging framework, cache, workflow engine, then machine coding orientation.

| Day | Topics | Time | Deliverable |
|---|---|---|---|
| Monday | Rate Limiter design: token bucket + sliding window algorithms | 2.5h | Design + implementation; plug in a new algorithm (fixed window) via Strategy without modification |
| Tuesday | Logging Framework design: levels, handlers, formatters, appenders | 2h | Class diagram; implement Chain of Responsibility for level filtering |
| Wednesday | Cache System design: LRU + LFU + TTL, eviction, multi-level | 2.5h | LRU cache implementation; plug in LFU via Strategy |
| Thursday | Workflow Engine design: DAG, conditional branching, parallel execution | 2h | Task graph design; Composite for task groups; State for workflow lifecycle |
| Friday | Machine Coding orientation: format, evaluation, the 10-minute design protocol | 2.5h | Read all machine coding problem statements; practice the 10-minute design phase on 2 problems without implementing |

**Weekend Assignment**: First timed machine coding attempt. Snake and Ladder game. Set a 90-minute timer. Submit whatever state the code is in when the timer ends. Self-evaluate against the rubric.

**Week 10 Milestone (Case Study Practitioner Badge)**: All 17 case studies complete with required artifacts. You have a body of design work to reference and review.

---

### Week 11 — Machine Coding (Intensive) + Testing & Quality

**Theme**: Execution speed and quality engineering.

| Day | Topics | Time | Deliverable |
|---|---|---|---|
| Monday | Machine coding timed practice: Car Rental System (90 min timer) | 2.5h | Submit + self-evaluate; read reference solution; note gaps |
| Tuesday | Machine coding timed practice: In-memory key-value store with TTL (90 min timer) | 2.5h | Submit + self-evaluate; identify the one design decision that most hurt your score |
| Wednesday | Testing: designing for testability, seams, Humble Object pattern | 2h | Refactor your Car Rental solution to remove all testability barriers (static deps, `new` in methods) |
| Thursday | Unit testing: AAA, naming conventions, FIRST properties, what to mock | 2.5h | Write 10 unit tests for your key-value store solution: naming, isolation, meaningful assertions |
| Friday | TDD for design: Red-Green-Refactor on a design problem; test coverage philosophy | 2h | Implement a new machine coding problem (Task Manager with Priority) TDD-first |

**Weekend Assignment**: Machine coding timed practice x2. Choose 2 problems from the remaining list. 90-minute timer each. Full rubric self-evaluation for both.

**Week 11 Milestone (Machine Coder Badge)**: You have completed 5+ timed machine coding problems. Your average score is 3+ on all dimensions.

---

### Week 12 — Capstone + Interview Final Preparation

**Theme**: Portfolio completion and interview readiness.

| Day | Topics | Time | Deliverable |
|---|---|---|---|
| Monday | Capstone project selection + 45-minute design phase | 3h | Class diagram + sequence diagrams + extension plan |
| Tuesday | Capstone implementation — core entities and primary use cases | 3h | Working core flow |
| Wednesday | Capstone implementation — error handling, extensions, tests | 3h | Extensions implemented without modifying core; unit tests |
| Thursday | Capstone decision log + artifact finalization | 2h | Decision log with 5+ entries; all artifacts assembled |
| Friday | Interview Readiness Checklist — complete all 50 items; identify gaps | 2h | Scored checklist; gap remediation plan |
| Weekend | Mock interview session + final review | 4h | One full mock interview (45-min design + Q&A); gap closure on lowest-scored checklist items |

**Week 12 Milestone (Senior Designator Badge + Course Completion)**: All certification criteria met. Portfolio complete. Interview Readiness Checklist scored. You are ready.

---

## Section 2: 4-Week Intensive Interview Prep Plan

> **For engineers with < 1 month to a Senior Engineer interview.**
> **Commitment**: 3-4 hours/day on weekdays, 6-8 hours on weekends.
> **Total**: ~80 hours over 4 weeks.

This plan is ruthlessly prioritized. You will cover the highest-signal topics and skip or skim the rest. You can return to gaps after your interview cycle.

---

### Week 1 — Foundations + SOLID + Top Design Principles

**Goal**: Solid enough on principles to not lose points. Move fast.

| Day | Topics | Hours | Notes |
|---|---|---|---|
| Monday | OOP Pillars summary (2h) + Composition vs. Inheritance (1h) | 3h | Skim Module 00. Do NOT spend more than half a day here if you have OOP experience. |
| Tuesday | SOLID — SRP + OCP (all of SRP in depth; OCP at interface level) | 3.5h | Assignment: decompose a god class |
| Wednesday | SOLID — LSP + ISP + DIP | 3.5h | Focus on DIP: 90% of testability problems trace here |
| Thursday | DRY + Cohesion/Coupling + Law of Demeter + DI | 3h | DI wiring exercise: constructor injection, mock in tests |
| Friday | UML speed practice: class diagrams (2h); sequence diagrams (1.5h) | 3.5h | Speed Draw exercise. You need to diagram in < 15 min |

**Weekend (6-8h)**: All three Easy case studies — Parking Lot, Library, Vending Machine. Each one: 20-minute independent design (on paper), then compare with reference, then implement core in 45 minutes. This is the template you'll use for every case study.

---

### Week 2 — Design Patterns (High-Priority Subset)

**Goal**: Know the 12 most interview-relevant patterns cold. Recognize, implement, justify.

**Priority order** (most to least interview-common):

| Priority | Pattern | Week 2 Day |
|---|---|---|
| 1 | Strategy | Monday |
| 2 | Observer | Monday |
| 3 | Factory Method | Tuesday |
| 4 | Builder | Tuesday |
| 5 | Singleton | Tuesday |
| 6 | Adapter | Wednesday |
| 7 | Decorator | Wednesday |
| 8 | Facade | Wednesday |
| 9 | Proxy | Thursday |
| 10 | Command | Thursday |
| 11 | Chain of Responsibility | Friday |
| 12 | State | Friday |

| Day | Topics | Hours |
|---|---|---|
| Monday | Strategy (in depth) + Observer (in depth) | 3.5h |
| Tuesday | Factory Method + Builder + Singleton (thread-safe) | 3.5h |
| Wednesday | Adapter + Decorator + Facade | 3h |
| Thursday | Proxy + Command (undo/redo) | 3h |
| Friday | Chain of Responsibility + State + Pattern recognition drill (identify pattern from 10 descriptions) | 3.5h |

**Secondary patterns** (read concepts, skip full implementation):
- Abstract Factory, Composite, Template Method, Mediator

**Skip on first pass**: Prototype, Bridge, Flyweight, Visitor, Memento, Iterator

**Weekend (7-8h)**: Medium case studies — ATM System + Hotel Booking + Movie Ticket Booking. Full design (30-min timer) + implementation for each. Focus: can you identify and justify at least 2 patterns per case study?

---

### Week 3 — Advanced Case Studies + Machine Coding

**Goal**: Cover the most interview-common advanced designs. Begin machine coding execution.

**Must-do advanced case studies** (ranked by interview frequency):

| Priority | Case Study | Day |
|---|---|---|
| 1 | Ride-Sharing System | Monday |
| 2 | Notification Framework | Tuesday |
| 3 | Payment Gateway | Tuesday |
| 4 | Rate Limiter | Wednesday |
| 5 | Cache System (LRU) | Wednesday |
| 6 | Logging Framework | Thursday |
| 7 | Splitwise | Thursday |

| Day | Topics | Hours |
|---|---|---|
| Monday | Ride-Sharing: full design (45-min timer) + implementation of core | 4h |
| Tuesday | Notification Framework (2h) + Payment Gateway design (2h) | 4h |
| Wednesday | Rate Limiter (2h) + Cache System LRU (2h) | 4h |
| Thursday | Logging Framework (1.5h) + Splitwise (2.5h) | 4h |
| Friday | Machine coding: Snake and Ladder (90-min timer) + self-evaluate (30 min) | 3h |

**Weekend (8h)**: Machine coding practice x3. Car Rental (90 min) + In-memory KV store (90 min) + Task Manager (90 min). Full self-evaluation after each. Read reference solutions. Identify your consistent weak spot (correctness? design? readability?). Fix it next week.

---

### Week 4 — Mock Interviews + Gap Closure

**Goal**: Simulate the interview. Identify and close the gaps that matter most.

| Day | Topics | Hours |
|---|---|---|
| Monday | Mock interview #1: design round (pick any unseen advanced case study, 45-min timer, walk through it verbally as if presenting) | 3h |
| Tuesday | Gap closure from Mock #1: whatever you struggled with most | 3h |
| Wednesday | Mock interview #2: design round (a different advanced case study) + machine coding (90-min timer) | 4h |
| Thursday | UML speed review (30 min) + Interview checklist self-assessment (1h) + weak area practice | 3h |
| Friday | Final mock + cheat sheet review + rest | 2h |

**Weekend**: Light review only. Skim cheat sheets. Sleep well.

---

### 4-Week Priority Checklist

Before your interview, confirm you can do these without reference:

**Foundations (must be solid)**
- [ ] Explain composition vs. inheritance with a concrete example
- [ ] Identify an SRP violation and propose the decomposition
- [ ] Explain DIP and show how it enables testability

**Patterns (must be solid)**
- [ ] Draw Strategy pattern class diagram from memory
- [ ] Explain Observer and the memory leak risk
- [ ] Explain Factory Method vs. Abstract Factory
- [ ] Implement a fluent Builder
- [ ] Explain Adapter with a real use case
- [ ] Explain Decorator vs. Proxy distinction clearly

**Case Studies (must be solid)**
- [ ] Design Parking Lot in 20 minutes without reference
- [ ] Design Ride-Sharing at a high level in 30 minutes
- [ ] Explain LRU Cache data structures and eviction policy

**Machine Coding (must be solid)**
- [ ] Complete a moderate problem in under 90 minutes with clean code
- [ ] Apply the 10-minute design protocol before coding

---

## Section 3: Skill Progression Map

### Progression Levels

```
Beginner → Intermediate → Advanced → Interview-Ready
```

---

### Beginner (Start of Course)

**Characteristics**: You can write OOP code syntactically but design decisions feel arbitrary.

**Typical behaviors**:
- Writes logic in service classes rather than domain objects
- Uses inheritance because it reduces code repetition, not because of IS-A
- Does not distinguish between interfaces and abstract classes by design intent
- Cannot explain why a design is "good" beyond "it works"

**Skills to unlock to reach Intermediate**:
- [ ] Apply SRP to decompose a monolithic class
- [ ] Choose interface vs. abstract class for 5 different scenarios correctly
- [ ] Replace an inheritance hierarchy with composition
- [ ] Draw a class diagram with correct relationship types
- [ ] Identify a DRY violation and extract the correct abstraction

---

### Intermediate (After Week 4-5)

**Characteristics**: You understand design principles and can apply them intentionally. Patterns are known but selection feels uncertain.

**Typical behaviors**:
- Applies SOLID principles when reviewing code
- Can design a simple system (< 6 entities) in under 20 minutes
- Knows pattern names and their definitions
- Struggles to select patterns without a hint in the problem description
- Machine coding produces working code with some design debt

**Skills to unlock to reach Advanced**:
- [ ] Select the right pattern for a described problem without prompting
- [ ] Implement any of the 12 priority patterns from memory
- [ ] Design a medium case study in under 30 minutes
- [ ] Explain a design decision with explicit tradeoff reasoning (not just "I chose X")
- [ ] Identify extension points in your design and confirm they hold when a new variant is added

---

### Advanced (After Week 8-9)

**Characteristics**: You design confidently, apply patterns appropriately, and can articulate every choice.

**Typical behaviors**:
- Approaches a new requirement with a clear decomposition process
- Selects patterns based on the problem's variation axis, not just familiarity
- Produces class diagrams with all relationships and multiplicities in < 20 minutes
- Can design all 17 case studies (with some time pressure)
- Machine coding produces clean code with appropriate design in 90 minutes

**Skills to unlock to reach Interview-Ready**:
- [ ] Design any advanced case study independently in under 45 minutes
- [ ] Complete 3 machine coding problems under 90 minutes with score 3+ across all dimensions
- [ ] Produce a capstone with full artifact set
- [ ] Walk through a design verbally while drawing, maintaining clarity
- [ ] Score 40/50 on Interview Readiness Checklist

---

### Interview-Ready (After Week 12)

**Characteristics**: You can enter any LLD interview and perform at a Senior Engineer level.

**Markers**:
- You have a reliable design process you can explain
- You draw and talk simultaneously without losing coherence
- You ask clarifying questions naturally before designing
- You propose tradeoffs without being asked
- You have a portfolio of 5+ case study designs to reference mentally
- You are not rattled by follow-up questions like "what if X changes?" or "how would you test this?"

---

### Self-Assessment Checklist (by progression level)

Rate yourself 1-5 on each skill:
- 1 = I cannot do this yet
- 2 = I understand the concept but cannot apply it
- 3 = I can apply it with some effort and reference
- 4 = I can apply it independently under normal conditions
- 5 = I can apply it under interview pressure and explain it clearly

| Skill | Self-Rating |
|---|---|
| Decompose a requirements paragraph into entities and behaviors | /5 |
| Draw a class diagram with all relationship types for a 6-entity system | /5 |
| Draw a sequence diagram for a multi-step use case | /5 |
| Apply SRP to decompose a god class | /5 |
| Apply OCP to design an extension point | /5 |
| Identify an LSP violation precisely | /5 |
| Apply DIP and make code testable with mocks | /5 |
| Distinguish knowledge duplication from textual duplication | /5 |
| Measure coupling and propose a reduction | /5 |
| Implement Singleton (thread-safe) | /5 |
| Implement Factory Method | /5 |
| Implement Builder (fluent, immutable) | /5 |
| Implement Adapter | /5 |
| Implement Decorator and stack multiple decorators | /5 |
| Implement Proxy (virtual) | /5 |
| Implement Strategy and eliminate a conditional dispatch | /5 |
| Implement Observer with unsubscribe | /5 |
| Implement Command with undo | /5 |
| Implement Chain of Responsibility | /5 |
| Implement State with lifecycle transitions | /5 |
| Design Parking Lot independently in 20 minutes | /5 |
| Design ATM System independently in 30 minutes | /5 |
| Design Ride-Sharing at class level in 45 minutes | /5 |
| Design LRU Cache with eviction strategy | /5 |
| Design a Notification Framework that supports new channels | /5 |
| Complete a machine coding problem in 90 minutes | /5 |
| Explain a design decision with explicit tradeoff reasoning | /5 |
| Recover from a wrong design direction mid-interview | /5 |

**Score interpretation**:
- Sum < 60: Beginner. Follow the 12-week plan.
- Sum 60-90: Intermediate. Accelerate through patterns and case studies.
- Sum 90-115: Advanced. Machine coding and mock interview focus.
- Sum 115+: Interview-Ready. Mock interview validation and gap closure only.

---

## Section 4: Interview Readiness Checklist

> Rate each item 1-5 (1 = cannot do, 5 = can do under interview pressure).
> **Target**: 40+ items scored 3 or above.
> **Gaps**: Any item scored 1-2 is a study priority.

---

### Category A: Object-Oriented Foundations (Items 1-10)

| # | Item | Rating |
|---|---|---|
| 1 | State the four OOP pillars and give a non-trivial example of each | /5 |
| 2 | Explain why "getter/setter on every field" violates encapsulation | /5 |
| 3 | Choose between interface, abstract class, and concrete class for a described design context | /5 |
| 4 | Replace an inheritance hierarchy with composition and preserve all behavior | /5 |
| 5 | Explain the fragile base class problem with a concrete example | /5 |
| 6 | Design an immutable value object with correct equality | /5 |
| 7 | Name a class, method, and boolean variable following Clean Code conventions for a described domain | /5 |
| 8 | Decompose a natural-language requirements paragraph into entities, attributes, and behaviors | /5 |
| 9 | Identify anemic domain model anti-pattern and refactor domain behavior into the model | /5 |
| 10 | Draw a class diagram for a described domain in under 15 minutes | /5 |

---

### Category B: SOLID Principles (Items 11-20)

| # | Item | Rating |
|---|---|---|
| 11 | State SRP precisely (one reason to change, not "one thing") and give the stakeholder-axis explanation | /5 |
| 12 | Decompose a god class by identifying each distinct change axis | /5 |
| 13 | State OCP and identify the concrete mechanism that enables it (abstractions, not wishful thinking) | /5 |
| 14 | Design an extension point so a new variant can be added without modifying existing code | /5 |
| 15 | Identify an LSP violation by specifying which behavioral contract is broken | /5 |
| 16 | Redesign a violated LSP hierarchy to restore substitutability | /5 |
| 17 | Decompose a fat interface into role interfaces appropriate for each client | /5 |
| 18 | State DIP and distinguish it from Dependency Injection | /5 |
| 19 | Refactor a class that instantiates its dependencies to use constructor injection | /5 |
| 20 | Demonstrate that a DIP-compliant class is testable with a mock collaborator | /5 |

---

### Category C: Design Principles (Items 21-25)

| # | Item | Rating |
|---|---|---|
| 21 | Distinguish knowledge duplication (DRY violation) from textual duplication (not always a violation) | /5 |
| 22 | Identify a Law of Demeter violation and refactor using Tell, Don't Ask | /5 |
| 23 | Given a class with low cohesion, identify the split and justify it | /5 |
| 24 | Measure coupling between two components qualitatively and propose a reduction | /5 |
| 25 | Identify a YAGNI violation: a feature built for a speculated requirement | /5 |

---

### Category D: Design Patterns — Creational & Structural (Items 26-35)

| # | Item | Rating |
|---|---|---|
| 26 | Implement a thread-safe Singleton and explain why double-checked locking needs `volatile` | /5 |
| 27 | Implement Factory Method and add a new product without modifying the creator | /5 |
| 28 | Distinguish Factory Method from Abstract Factory with a concrete use case | /5 |
| 29 | Implement a fluent, immutable Builder with validation at `build()` | /5 |
| 30 | Apply Adapter using composition (object adapter) to integrate an incompatible interface | /5 |
| 31 | Implement Decorator and stack three decorators; verify order matters | /5 |
| 32 | Explain Decorator vs. Proxy: what is the intent difference? | /5 |
| 33 | Apply Facade to simplify a 4-class subsystem to a single client call | /5 |
| 34 | Design a Composite for a tree structure where leaves and branches share a common interface | /5 |
| 35 | Implement a Virtual Proxy that defers expensive initialization to first use | /5 |

---

### Category E: Design Patterns — Behavioral (Items 36-45)

| # | Item | Rating |
|---|---|---|
| 36 | Apply Strategy to eliminate a switch or nested if/else on type | /5 |
| 37 | Implement Observer with both subscribe and unsubscribe; explain the memory leak risk | /5 |
| 38 | Implement Command with undo and redo; explain how command queues enable transaction logs | /5 |
| 39 | Build a Chain of Responsibility pipeline and configure handler order at runtime | /5 |
| 40 | Model an object lifecycle (Order, Ticket, Connection) using State pattern | /5 |
| 41 | Apply Template Method to share algorithm structure with subclass-overridable steps | /5 |
| 42 | Distinguish Template Method (inheritance-based) from Strategy (composition-based) for the same problem | /5 |
| 43 | Identify when Visitor is appropriate (stable structure, varying operations) vs. overkill | /5 |
| 44 | Implement Memento with Caretaker; handle a redo stack | /5 |
| 45 | Identify which pattern applies from a 100-word problem description without being told there is a pattern | /5 |

---

### Category F: Case Studies & System Design (Items 46-50)

| # | Item | Rating |
|---|---|---|
| 46 | Design a Parking Lot system independently: class diagram + sequence diagrams + 2 patterns identified | /5 |
| 47 | Design a Ride-Sharing system at class level: driver matching, trip lifecycle, pricing — in under 45 minutes | /5 |
| 48 | Design an LRU Cache: data structures, eviction, `get`/`put` operations — implement from memory | /5 |
| 49 | Design a Notification Framework that supports new channels via extension, not modification | /5 |
| 50 | Articulate 3 design decisions from a case study with: alternative considered, reason for choice, cost accepted | /5 |

---

### Score Summary Table

| Category | Items | Your Score | Max Score |
|---|---|---|---|
| A: OOP Foundations | 1-10 | /50 | 50 |
| B: SOLID Principles | 11-20 | /50 | 50 |
| C: Design Principles | 21-25 | /25 | 25 |
| D: Creational & Structural Patterns | 26-35 | /50 | 50 |
| E: Behavioral Patterns | 36-45 | /50 | 50 |
| F: Case Studies | 46-50 | /25 | 25 |
| **Total** | **1-50** | **/250** | **250** |

**Interpretation**:
- 200+ (80%+): Interview-Ready. Mock interviews and final polish only.
- 150-199 (60-79%): Advanced. 2-3 more weeks of targeted practice.
- 100-149 (40-59%): Intermediate. Full pattern + case study pass required.
- < 100 (< 40%): Beginner-Intermediate. Complete the 12-week plan.

---

### Gap Identification Guidance

After completing the checklist, run this analysis:

**Step 1**: List all items scored 1 or 2. These are your hard gaps.

**Step 2**: Group gaps by category:
- Mostly Category A-B → You need more time in Modules 00-01. Re-read and re-do the assignments before advancing to patterns.
- Mostly Category D-E → You have the principles but not the patterns. Intensive pattern week.
- Mostly Category F → You know the material but can't apply it under pressure. Case study repetition with a timer.

**Step 3**: For each hard gap item, identify the module and assignment that covers it directly. Schedule a focused 2-hour session.

**Step 4**: Re-score the checklist after 1 week of gap-focused study. Confirm you've moved from 1-2 to 3+.

---

### Interview Day Protocol

When you enter the LLD interview:

1. **First 2 minutes**: Ask clarifying questions. Scope, scale, constraints, what to focus on. Do not skip this — it signals senior thinking.
2. **Next 8 minutes**: Design phase. Nouns → entities. Verbs → behaviors. Draw boxes before lines. Add relationships after entities are stable.
3. **Next 5 minutes**: Identify patterns. Name them. "I'll use Strategy here because pricing varies by tier." Interviewers want to hear you name patterns correctly.
4. **Next 10-20 minutes**: Walk through the design. Explain one flow end-to-end using the sequence diagram.
5. **Ongoing**: When asked "what if X changes?" — answer with the extension point: "Because I have this interface, adding X just means a new implementation. The core doesn't change."
6. **Final 5 minutes**: Tradeoffs you considered. What you'd improve with more time. This is where Senior engineers separate from Mid-level.

---

*Navigation: [README.md](./README.md) | [CURRICULUM.md](./CURRICULUM.md) | [00-foundation/](./00-foundation/)*
