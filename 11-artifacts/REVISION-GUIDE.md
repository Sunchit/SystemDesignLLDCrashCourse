# Master Revision Guide — The Complete Pre-Interview Checklist

> **Audience:** Senior engineers preparing for LLD interviews at Google, Amazon, Meta, Adobe, and equivalent top-tier companies.  
> **How to use this guide:** Work through it linearly the week before your interview. The 7-day plan governs scheduling; sections 2–6 are reference material you return to repeatedly.

---

## Section 1: The 7-Day Revision Plan

**Total estimated hours: ~22–26 hours across 7 days**

---

### Day 7 (One week out) — SOLID + OOP Fundamentals
**Estimated time: 3–4 hours**

**What to study:**
- All five SOLID principles — state each one, identify a violation in code, then refactor it
- Core OOP concepts: encapsulation, abstraction, inheritance, polymorphism
- IS-A vs HAS-A relationships — when to inherit vs compose
- Method overloading vs overriding — understand the dispatch mechanisms
- Object composition: why it produces more flexible designs than deep inheritance hierarchies

**How to study:**
- Write each SOLID principle in one sentence from memory. If you hesitate, the understanding isn't solid.
- For each principle, write a 5–10 line Java/Python class that violates it, then refactor to comply.
- Draw a 3-level class hierarchy (e.g., `Vehicle → Car → ElectricCar`) and identify which relationships are IS-A and which should be HAS-A.

**Course files to use:**
- `01-oops/` — revisit all concept files
- `02-solid/` — one file per principle; work through each violation example
- `03-design-principles/` — composition over inheritance notes

**Goal:** By end of day, you can explain every SOLID principle with a concrete code example. No notes required.

---

### Day 6 — All 18 Design Patterns
**Estimated time: 5–6 hours**

**What to study:**
- All three categories: Creational (5–6), Structural (7), Behavioral (11, but focus on the 9 most common)
- For each pattern: intent, structure (key participants), a canonical real-world use case, and how to recognize it in an existing codebase

**How to study — 15–20 minutes per pattern:**
1. State the problem it solves (1 sentence)
2. Draw the minimal class diagram from memory
3. Write a 10–20 line code skeleton — just the structure, not full implementation
4. Name one place in a production system where this pattern appears (e.g., Observer in event buses, Decorator in Java I/O streams, Strategy in payment processing)

**Pacing:**
- Morning (2.5 hours): Creational + Structural (Singleton, Factory Method, Abstract Factory, Builder, Prototype, Object Pool, Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy)
- Afternoon (2.5 hours): Behavioral (Chain of Responsibility, Command, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method)

**Course files to use:**
- `04-design-patterns/creational/`
- `04-design-patterns/structural/`
- `04-design-patterns/behavioral/`

**Goal:** For every pattern, you can sketch the structure and name a use case in under 2 minutes.

---

### Day 5 — Case Studies from Scratch
**Estimated time: 4–5 hours**

**What to study:**
- Pick 2 case studies from: Library Management, Parking Lot, Hotel Booking, Ride Sharing, ATM
- Do each one completely from scratch — no notes, no reference implementations open

**What "from scratch" means:**
1. Read the problem statement only
2. Spend 3 minutes asking clarifying questions (write them down — interview habit)
3. Identify entities and relationships — draw a rough class diagram by hand or in a text editor (15 minutes)
4. Identify which design patterns apply and why — document your reasoning (5 minutes)
5. Write the core classes: interfaces, abstract classes, key concrete implementations (30–40 minutes)
6. Add at least one non-trivial feature: concurrency handling, extensibility hook, or edge case (10 minutes)

**What to aim for:**
- A working class structure that a senior engineer would not immediately reject
- At least 2 design patterns applied with explicit reasoning
- Identified trade-offs: "I used Strategy here instead of a switch statement because pricing rules change frequently"
- Handled at least one edge case: null inputs, concurrent access, invalid state transitions

**Goal:** Two complete case study designs produced independently. Compare them against reference implementations in `05-case-studies/` only after you finish.

---

### Day 4 — Mock Interview Simulation
**Estimated time: 3–4 hours**

**Format:**
- Set a timer for 45 minutes
- Choose a case study you have NOT done in the last 3 days (keep it fresh)
- Narrate out loud as you would to a real interviewer — all decisions, all trade-offs
- Record yourself if possible (voice memo is sufficient)

**What to simulate:**
- The first 2 minutes: Ask clarifying questions. Do not start designing immediately.
- Minutes 2–10: Identify core entities, draw class diagram
- Minutes 10–30: Write the core classes — the interviewer is watching you code
- Minutes 30–40: Discuss extensibility, add a feature, handle a curveball
- Minutes 40–45: Review your design — what would you change?

**How to evaluate yourself (post-simulation):**
After the timer stops, score yourself on each dimension (1–5):
- Did you ask good clarifying questions before designing?
- Was the class hierarchy clean and well-reasoned?
- Did you apply at least one design pattern correctly?
- Could you explain your trade-offs clearly?
- Did you finish within time?
- Did you handle a curveball gracefully?

Write down your two weakest areas. These become Day 3's focus.

**Goal:** One full mock interview completed. Self-assessment written. Weak areas identified.

---

### Day 3 — Address Weak Areas
**Estimated time: 3–4 hours**

**How to identify weak areas:**
Use your Day 4 self-assessment scores. Anything rated 3 or below is a weak area.

**Common weak areas and how to address them:**

| Weak Area | How to Address |
|---|---|
| Pattern selection (wrong pattern chosen) | Review the "when to use" section for each pattern. Practice: given a scenario, name the pattern in 30 seconds. |
| Class hierarchy design (too flat or too deep) | Review IS-A vs HAS-A. Redraw any case study with 3 different hierarchy structures and compare. |
| Clarifying questions (skipped or superficial) | Write a list of 10 universal LLD clarifying questions. Memorize them. |
| Code structure (messy or incomplete) | Time yourself: write a clean `interface` + `abstract class` + 2 concrete classes in under 10 minutes. |
| Explaining trade-offs (vague or absent) | For every design decision in a case study, write: "I chose X over Y because Z." Practice until it's automatic. |
| Concurrency (ignored entirely) | Add thread-safety to one case study: `synchronized`, `ReentrantLock`, or immutable objects. |

**Spend at least 2 hours on your weakest area, then 1 hour on the second weakest.**

**Goal:** Quantifiable improvement on the areas that failed in Day 4's mock.

---

### Day 2 — Quick Pattern Review + Machine Coding Practice
**Estimated time: 3–4 hours**

**Quick pattern review (1 hour):**
- For each of the 18 patterns, write the intent in one sentence from memory
- If you hesitate on any, spend 5 minutes reviewing that pattern only
- This is a speed drill — you should complete all 18 in under 60 minutes

**Machine coding practice (2–3 hours):**
Machine coding means writing working, compilable code (not pseudocode) within a fixed time limit — typically 60–90 minutes. This is distinct from whiteboard design.

**How to time yourself:**
- Choose a medium-complexity problem (e.g., Parking Lot, Library Management)
- Set a 75-minute timer
- Write actual code: classes, methods, basic logic — it must compile
- The bar is not production code; it is readable, structured, and complete enough to discuss

**What to focus on:**
- Class design speed: How fast can you produce a clean interface hierarchy?
- Method signatures first, implementations second — stub everything, then fill in
- Run through your mental checklist: single responsibility, correct abstractions, pattern applied

**Goal:** One complete machine coding submission in under 75 minutes. Code compiles. Design is defensible.

---

### Day 1 (Day before) — Light Review + Rest
**Estimated time: 1–2 hours maximum**

**What "light review" means:**
- Skim Section 5 of this guide (Last 30-Minute Review) — the full version
- Review your self-assessment checklist from Section 2 — look at any items still rated 3 or below
- Read through your Day 4 self-assessment notes — remind yourself what improved
- Do NOT write new code. Do NOT start a new case study.

**Why rest matters:**
Decision quality in an interview degrades significantly when you are fatigued. The interview requires fluid, real-time reasoning — not rote recall. Cognitive flexibility peaks after adequate sleep. Over-cramming on Day 1 fills your working memory with fragmented notes and increases anxiety without proportional benefit. The marginal return on the 10th hour of study the day before is negative.

**What to do instead:**
- Sleep 7–8 hours minimum
- Light walk, exercise, or activity that reduces cortisol
- Review once, then stop

**Goal:** You arrive at the interview rested, with a clear mental model of the material — not exhausted and second-guessing yourself.

---

**Total: 22–26 hours over 7 days**

| Day | Focus | Hours |
|---|---|---|
| Day 7 | SOLID + OOP fundamentals | 3–4 |
| Day 6 | All 18 design patterns | 5–6 |
| Day 5 | 2 case studies from scratch | 4–5 |
| Day 4 | Mock interview simulation | 3–4 |
| Day 3 | Address weak areas | 3–4 |
| Day 2 | Pattern review + machine coding | 3–4 |
| Day 1 | Light review + rest | 1–2 |

---

## Section 2: Revision Topics Self-Assessment Checklist

**Instructions:** Rate yourself honestly from 1–5 before and after your revision week.  
`1 = Can't recall` | `2 = Vague understanding` | `3 = Can explain` | `4 = Can apply` | `5 = Can teach`

---

### OOP Concepts

- [ ] **Encapsulation** — Can explain why hiding internal state behind a public interface reduces coupling, and demonstrate with a class that uses private fields + getters/setters correctly.  
  `Rating: _/5`

- [ ] **Abstraction** — Can distinguish between abstraction (hiding complexity) and encapsulation (hiding data), and implement it correctly using abstract classes and interfaces.  
  `Rating: _/5`

- [ ] **Inheritance** — Can describe the inheritance hierarchy mechanism, explain the fragile base class problem, and state when NOT to use inheritance.  
  `Rating: _/5`

- [ ] **Polymorphism** — Can implement runtime polymorphism via method overriding and compile-time polymorphism via overloading; can explain how the JVM resolves method calls at runtime.  
  `Rating: _/5`

- [ ] **Method overloading vs overriding** — Can state the exact rules for each (return type, signature, access modifier), explain why they are compile-time vs runtime mechanisms, and identify which is happening in a code snippet.  
  `Rating: _/5`

- [ ] **IS-A vs HAS-A relationships** — Can read a class diagram and correctly label every relationship as IS-A (inheritance) or HAS-A (composition/aggregation); can explain when HAS-A is preferable.  
  `Rating: _/5`

- [ ] **Covariance and contravariance** — Can explain covariance in return types and contravariance in parameter types; can identify how they relate to the Liskov Substitution Principle.  
  `Rating: _/5`

- [ ] **Object composition over inheritance** — Can take a class hierarchy with 3+ levels and refactor it to use composition; can articulate why this is more flexible at runtime.  
  `Rating: _/5`

---

### SOLID Principles

- [ ] **Single Responsibility — state it** — Can state the principle precisely: "A class should have one, and only one, reason to change" — and explain what "reason to change" means in terms of stakeholders/actors.  
  `Rating: _/5`

- [ ] **Single Responsibility — apply it** — Can take a God Class (e.g., an `OrderService` that validates, persists, and sends email) and split it into well-bounded classes with a single responsibility each.  
  `Rating: _/5`

- [ ] **Open/Closed Principle — state it** — Can state: "Software entities should be open for extension but closed for modification" — and explain the trade-off between stability and extensibility.  
  `Rating: _/5`

- [ ] **Open/Closed Principle — apply it** — Can demonstrate OCP using the Strategy pattern or abstract base class; can show how adding a new payment type (e.g., crypto) requires zero changes to existing classes.  
  `Rating: _/5`

- [ ] **Liskov Substitution Principle — state it** — Can state: "Subtypes must be substitutable for their base types without altering program correctness" — and give the classic Rectangle/Square violation example.  
  `Rating: _/5`

- [ ] **Liskov Substitution Principle — apply it** — Can detect an LSP violation in a hierarchy (method that throws `UnsupportedOperationException`, precondition strengthened in subclass) and refactor to fix it.  
  `Rating: _/5`

- [ ] **Interface Segregation Principle — state it** — Can state: "No client should be forced to depend on methods it does not use" — and explain how fat interfaces create unnecessary coupling.  
  `Rating: _/5`

- [ ] **Interface Segregation Principle — apply it** — Can take an interface with 8 methods and split it into 3 focused interfaces; can explain why this reduces the blast radius of changes.  
  `Rating: _/5`

- [ ] **Dependency Inversion Principle — state it** — Can state: "High-level modules should not depend on low-level modules; both should depend on abstractions" — and explain the directionality of the dependency.  
  `Rating: _/5`

- [ ] **Dependency Inversion Principle — apply it** — Can refactor a class that directly instantiates its dependencies (`new MySQLDatabase()`) to use constructor injection with an interface; can explain how this enables testing and swapping implementations.  
  `Rating: _/5`

---

### Design Patterns — Creational

- [ ] **Singleton** — Can implement a thread-safe Singleton (double-checked locking or enum), explain when Singleton is appropriate vs. an antipattern, and name the testability problem it creates.  
  `Rating: _/5`

- [ ] **Factory Method** — Can distinguish Factory Method (subclass decides what to create) from Simple Factory (not a GoF pattern); can implement a `DocumentFactory` where subclasses produce PDF vs Word documents.  
  `Rating: _/5`

- [ ] **Abstract Factory** — Can explain why Abstract Factory provides a family of related objects without specifying concrete classes; can implement a UI theme factory that produces consistent Button + Checkbox + TextField objects.  
  `Rating: _/5`

- [ ] **Builder** — Can implement a Builder for a complex object (e.g., `HttpRequest`) with mandatory and optional parameters, explain why it solves the telescoping constructor problem, and name when to use it over a plain constructor.  
  `Rating: _/5`

- [ ] **Prototype** — Can explain shallow vs deep copy, implement `clone()` correctly, and describe when Prototype is preferable to instantiation (expensive object creation, large number of similar objects).  
  `Rating: _/5`

- [ ] **Object Pool** — Can explain the Object Pool pattern (reuse a fixed set of expensive-to-create objects), implement a basic pool with `acquire()` and `release()`, and name a real-world use case (DB connection pools, thread pools).  
  `Rating: _/5`

---

### Design Patterns — Structural

- [ ] **Adapter** — Can distinguish class adapter (via inheritance) from object adapter (via composition), implement a legacy API adapter, and explain why object adapter is preferred in Java.  
  `Rating: _/5`

- [ ] **Bridge** — Can explain how Bridge separates abstraction from implementation along two independent dimensions, draw the two-hierarchy structure, and name a use case (e.g., `Shape` + `RenderingEngine`).  
  `Rating: _/5`

- [ ] **Composite** — Can implement the Component/Leaf/Composite structure, explain how it enables uniform treatment of individual objects and compositions, and name a canonical example (file system, UI component tree).  
  `Rating: _/5`

- [ ] **Decorator** — Can implement a decorator chain that adds behavior at runtime (e.g., `BufferedReader` wrapping `FileReader`), explain why it is preferable to subclassing for optional feature combinations.  
  `Rating: _/5`

- [ ] **Facade** — Can implement a Facade that simplifies a complex subsystem, explain the trade-off between hiding complexity and reducing transparency, and distinguish Facade from Adapter.  
  `Rating: _/5`

- [ ] **Flyweight** — Can explain the intrinsic vs extrinsic state split, implement a character glyph flyweight, and calculate the memory saving in a concrete example.  
  `Rating: _/5`

- [ ] **Proxy** — Can implement at least two proxy variants (virtual proxy for lazy loading, protection proxy for access control), explain how Proxy differs from Decorator in intent.  
  `Rating: _/5`

---

### Design Patterns — Behavioral

- [ ] **Chain of Responsibility** — Can implement a handler chain (e.g., approval workflow, middleware pipeline), explain how it decouples sender from receiver, and describe how to handle the case where no handler processes the request.  
  `Rating: _/5`

- [ ] **Command** — Can implement Command with `execute()` and `undo()`, explain how it enables queuing, logging, and undo/redo, and give a concrete use case (text editor operations, transaction queue).  
  `Rating: _/5`

- [ ] **Iterator** — Can implement a custom Iterator over a non-standard data structure, explain how it separates traversal logic from the collection, and state why this matters for encapsulation.  
  `Rating: _/5`

- [ ] **Mediator** — Can implement a Mediator that coordinates object interactions (e.g., air traffic control, chat room), explain how it reduces n-to-n dependencies to 1-to-n, and state when it crosses into God Object territory.  
  `Rating: _/5`

- [ ] **Memento** — Can implement Memento with Originator/Memento/Caretaker, explain how it captures state without violating encapsulation, and implement an undo stack.  
  `Rating: _/5`

- [ ] **Observer** — Can implement Observer (Subject/Observer interfaces, push vs pull model), explain the difference between synchronous and asynchronous notification, and describe the dangling reference problem.  
  `Rating: _/5`

- [ ] **State** — Can implement the State pattern for a stateful object (e.g., ATM, vending machine), explain how it eliminates conditional branching, and distinguish State from Strategy.  
  `Rating: _/5`

- [ ] **Strategy** — Can implement Strategy with interchangeable algorithms (e.g., sorting, payment processing), explain how it satisfies OCP, and decide when to use Strategy vs simple polymorphism.  
  `Rating: _/5`

- [ ] **Template Method** — Can implement Template Method in an abstract class with invariant steps and variable steps as abstract methods; explain the Hollywood Principle ("Don't call us, we'll call you").  
  `Rating: _/5`

---

### Case Studies

- [ ] **Library Management System** — Can design the full class hierarchy (Book, Member, Loan, Catalog, fine calculation) from scratch in 45 minutes; can apply Strategy for fine calculation and Observer for overdue notifications.  
  `Rating: _/5`

- [ ] **Parking Lot** — Can design multi-level parking with different vehicle types, pricing strategies, and spot assignment logic; can apply Strategy for pricing, Factory for spot assignment, and Observer for availability updates.  
  `Rating: _/5`

- [ ] **Hotel Booking System** — Can model rooms, reservations, pricing (seasonal, loyalty tier), and booking workflows; can apply Strategy for pricing, State for reservation status, and Builder for complex booking objects.  
  `Rating: _/5`

- [ ] **Ride Sharing (Uber/Lyft)** — Can model drivers, riders, trips, surge pricing, and matching; can apply Strategy for pricing and driver matching, Observer for trip status, and State for trip lifecycle.  
  `Rating: _/5`

- [ ] **ATM System** — Can model the full ATM flow: card reading, PIN validation, account operations, cash dispensing; can apply State for ATM status, Command for transactions, and Chain of Responsibility for transaction validation.  
  `Rating: _/5`

---

### Machine Coding Skills

- [ ] **Time management** — Can consistently produce a working class design in 45 minutes; knows which parts to implement first (interfaces and key classes) vs defer (edge case handlers, utility methods).  
  `Rating: _/5`

- [ ] **Class design speed** — Can produce a clean interface hierarchy (3–5 interfaces, 2–3 abstract classes, 4–6 concrete classes) in under 15 minutes for a medium-complexity problem.  
  `Rating: _/5`

- [ ] **Handling edge cases** — Proactively identifies and handles: null inputs, concurrent modification, invalid state transitions, boundary conditions — without being prompted by the interviewer.  
  `Rating: _/5`

- [ ] **Asking clarifying questions** — Asks targeted, high-value questions before designing: scale requirements, consistency vs availability trade-off, which features are in scope, whether extensibility is a priority.  
  `Rating: _/5`

- [ ] **Explaining trade-offs** — Articulates every major design decision as a trade-off ("I used X instead of Y because Z, with the downside that W"); does not present decisions as the only option.  
  `Rating: _/5`

---

### UML / Diagrams

- [ ] **Class notation** — Can draw class boxes correctly: class name, attributes (with visibility `+/-/#`), methods — and uses correct notation for abstract classes and interfaces.  
  `Rating: _/5`

- [ ] **Relationship types** — Can draw and distinguish association, aggregation, composition, dependency, inheritance (generalization), and realization (interface implementation) — correct arrowheads for each.  
  `Rating: _/5`

- [ ] **Reading a diagram** — Can look at an unfamiliar class diagram and explain: the responsibility of each class, the lifecycle implications of composition vs aggregation, the direction of dependencies.  
  `Rating: _/5`

- [ ] **Drawing quickly** — Can produce a complete class diagram for a 5-class system in under 8 minutes — interview speed, not publication quality.  
  `Rating: _/5`

- [ ] **Pattern recognition in diagrams** — Can look at a class diagram and identify which design pattern(s) it represents, citing the specific structural indicators (e.g., "this is Decorator because the concrete component and decorator both implement the same interface").  
  `Rating: _/5`

---

## Section 3: The Most Important Things to Know

### 3a. Top 10 Patterns by Interview Frequency

**1. Strategy**  
Use case: Interchangeable algorithms — payment methods, sorting, pricing engines, validation rules.  
Why interviewers love it: It is the most direct demonstration of OCP and DIP simultaneously. It separates what an algorithm does from who calls it. Every experienced interviewer has a pricing or sorting problem ready to hand.

**2. Observer**  
Use case: Event-driven notification — inventory alerts, order status updates, UI event handling, pub/sub systems.  
Why interviewers love it: It appears in almost every distributed and UI system. It tests whether candidates understand decoupling between publishers and subscribers, and whether they think about synchronous vs async delivery and memory leak risks.

**3. Factory Method / Abstract Factory**  
Use case: Object creation without specifying concrete types — document generators, UI component factories, database connection factories.  
Why interviewers love it: It tests whether candidates can separate creation logic from business logic. The moment a codebase has `if type == "PDF" return new PdfDocument()`, Factory Method is the answer. Abstract Factory shows up whenever consistent object families are required.

**4. Decorator**  
Use case: Dynamic behavior extension — I/O stream wrapping, middleware pipelines, feature flag layering, logging/tracing wrappers.  
Why interviewers love it: It directly challenges the instinct to solve every extension problem with subclassing. A candidate who reaches for Decorator demonstrates they understand the combinatorial explosion problem with inheritance.

**5. Singleton**  
Use case: Single shared instance — configuration manager, thread pool, connection pool, logger.  
Why interviewers love it: It is the most commonly misused pattern. They want to see that you can implement it thread-safely and that you know when NOT to use it (global mutable state, testability problems).

**6. State**  
Use case: Object behavior changes based on state — ATM, vending machine, order lifecycle, network connection, media player.  
Why interviewers love it: State machines appear in every system. Candidates who use a `switch` statement fail; candidates who apply the State pattern demonstrate that they can eliminate conditional branching and make transitions explicit and type-safe.

**7. Builder**  
Use case: Complex object construction with many optional parameters — HTTP request, query builder, configuration objects, complex domain entities.  
Why interviewers love it: It shows awareness of the telescoping constructor antipattern. Senior engineers use Builder naturally for objects with more than 3–4 parameters; junior engineers write a constructor with 8 parameters.

**8. Command**  
Use case: Encapsulating operations as objects — undo/redo, transaction queuing, task scheduling, macro recording.  
Why interviewers love it: It tests whether candidates understand that operations can be first-class objects. It comes up in any system that needs reversibility, auditability, or deferred execution.

**9. Composite**  
Use case: Tree structures with uniform treatment — file systems, UI component trees, organizational hierarchies, expression parsers.  
Why interviewers love it: It tests recursive structure design. Any time a problem has "a group of X behaves like a single X," the answer is Composite. Candidates who spot this quickly impress interviewers.

**10. Chain of Responsibility**  
Use case: Sequential processing pipeline — request validation, approval workflows, event propagation, middleware.  
Why interviewers love it: It appears in every web framework (middleware chains), every workflow engine (approval chains), and every validation system. It tests whether candidates can design open-ended processing pipelines without hardcoding the sequence.

---

### 3b. Top 5 Case Studies to Know Cold

**1. Parking Lot**
- Key design decisions:
  - Use Strategy for pricing (flat rate, hourly, premium) to allow adding new strategies without modifying the pricing engine
  - Use Factory Method for spot allocation (compact, large, handicapped, EV) so new spot types can be added cleanly
  - Use Observer to notify floor management when all spots on a level fill or free up
  - Model spot availability as State (Available, Occupied, Reserved, Maintenance) to make transitions explicit
- The class interviewers always ask about: `ParkingSpot` — they want to see whether you made it abstract with concrete subclasses, and how you model availability vs reservation.

**2. Library Management System**
- Key design decisions:
  - Use Strategy for fine calculation — daily fine, cap-based fine, member-tier-based fine — so rules can change independently
  - Use Observer to notify members about due dates, overdue items, and reservation availability
  - Model loan lifecycle as State (Checked Out, Overdue, Returned, Renewed) rather than boolean flags
  - Use a `Catalog` as a Facade over the book search subsystem (title, ISBN, author, genre searches)
- The class interviewers always ask about: `Loan` — they want to see how you model the relationship between Book, Member, dates, and fine calculation without cramming everything into one class.

**3. Hotel Booking System**
- Key design decisions:
  - Use Strategy for pricing (seasonal pricing, loyalty discounts, corporate rates) so the pricing engine is extensible
  - Use State for reservation status (Pending, Confirmed, Checked In, Checked Out, Cancelled, No-Show)
  - Use Builder for constructing complex booking objects with many optional attributes (meal plan, room type, add-ons)
  - Use Observer to notify housekeeping and inventory systems when booking status changes
- The class interviewers always ask about: `Room` — they want to see the hierarchy (Room → StandardRoom, DeluxeRoom, Suite), how you handle amenities without an explosion of subclasses, and how pricing attaches.

**4. Ride Sharing (Uber-style)**
- Key design decisions:
  - Use Strategy for driver matching (nearest driver, highest-rated driver, cheapest driver) so matching logic is swappable
  - Use Strategy for pricing (base fare, surge multiplier, subscription override) decoupled from the Trip class
  - Use State for trip lifecycle (Requested, Driver Assigned, Driver En Route, In Progress, Completed, Cancelled)
  - Use Observer for trip status notifications so Rider, Driver, and billing system all update independently
- The class interviewers always ask about: `Trip` — they want to see how you model start/end location, route, fare calculation, and status without making Trip a God Object.

**5. ATM System**
- Key design decisions:
  - Use State for ATM status (Idle, Card Inserted, PIN Entered, Transaction In Progress, Out Of Service) to make transitions type-safe and prevent illegal operations
  - Use Command for transactions (Withdrawal, Deposit, Transfer, Balance Inquiry) to enable logging, rollback, and audit trail
  - Use Chain of Responsibility for transaction validation (card validation → PIN validation → balance check → daily limit check)
  - Use Facade for the bank interface — the ATM should not know the details of how the bank processes transactions
- The class interviewers always ask about: `ATMState` — they want to see whether you implemented it as an interface with concrete state classes, and how state transitions are triggered and guarded.

---

### 3c. The 3 Things That Always Impress Interviewers

**1. You name the pattern and justify it in terms of the problem, not the pattern itself.**  
Junior engineers say: "I'll use Strategy here."  
Senior engineers say: "I'll use Strategy here because pricing rules change frequently and independently of the checkout flow — this way adding a new pricing tier requires zero changes to existing classes and the product team can configure it without touching checkout logic."  
The justification signals that you understand the pattern's purpose, not just its structure.

**2. You proactively raise the extensibility question before the interviewer does.**  
Before finishing your design, you pause and say: "Let me think about what's most likely to change in this system — and whether my design makes those changes cheap."  
Then you identify 1–2 concrete extension points and show that your design handles them without modification to existing classes. Interviewers plant extension requirements precisely to see if you anticipated them. When you already handled the change before they asked, it signals architectural maturity.

**3. You model invalid states as unrepresentable rather than guarded.**  
When designing state-heavy systems, junior engineers add `if (status != null && status.equals("ACTIVE"))` guards everywhere. Senior engineers make the invalid state structurally impossible — by using the State pattern, by making fields final, by using sealed class hierarchies where the type itself encodes the state.  
When you say "I've designed this so a Loan in Returned state cannot have a non-null return date checked out again — the type system prevents it," interviewers notice.

---

### 3d. The 3 Things That Always Fail Candidates

**1. Jumping to code before establishing a clear class design.**  
The single most common failure mode. A candidate starts writing method bodies before they have decided what the classes are. The interviewer watches the design unravel in real time as the candidate discovers they need a class that doesn't fit what they've already written.  
The fix: Always spend the first 10–15 minutes on entities, relationships, and interfaces — on paper or a blank file. No implementation code until the structure is agreed on.

**2. Producing a design that cannot be extended without modification.**  
A design where adding a new payment type requires editing `PaymentProcessor.java`, a design where adding a new vehicle type requires adding a new `if` branch in the spot allocation logic — these are OCP violations that interviewers specifically look for.  
The fix: Before presenting your design, ask yourself: "If they ask me to add one more X, can I do it by adding a new class?" If the answer is no for every major dimension of variability, your design is closed.

**3. Inability to explain what you would do differently given more time.**  
Interviewers end LLD rounds with: "What would you improve about your design?" Candidates who say "I think it's fine" or "I'd add more error handling" fail this. Senior engineers always have a list ready: thread safety for shared resources, persistence layer abstraction, event sourcing for auditability, richer value objects to eliminate primitive obsession.  
The fix: After every design you produce, write down 3 specific improvements you didn't have time for. Practice naming them fluently.

---

## Section 4: Mental Models to Internalize (15 items)

**1. "Depend on abstractions, not concretions."**  
Prevents: High-level business logic becoming brittle when low-level implementation details change.  
Example: `OrderService` depends on `PaymentGateway` (interface), not `StripePaymentGateway` (class) — swapping to PayPal requires no changes to order logic.

**2. "Favor composition over inheritance."**  
Prevents: Fragile class hierarchies where a change to a base class breaks multiple subclasses in unexpected ways.  
Example: `Logger` with pluggable `Formatter` and `Transport` objects supports more combinations than a `FileLogger extends Logger` hierarchy ever could.

**3. "Open for extension, closed for modification."**  
Prevents: Bug regression in tested, deployed code whenever new features are added.  
Example: Adding `CryptoPayment` means creating a new class, not editing `PaymentProcessor` — existing tests remain valid.

**4. "Program to an interface, not an implementation."**  
Prevents: Call sites becoming coupled to a specific class, making substitution and testing impossible.  
Example: Declare `List<Order> orders = new ArrayList<>()` but accept `List<Order>` in method parameters — callers pass any list implementation.

**5. "Encapsulate what varies."**  
Prevents: Volatility propagating through the system — when the thing that changes is isolated, everything else is stable.  
Example: Surge multiplier calculation is encapsulated in `SurgePricingStrategy` — changes to surge logic touch exactly one class.

**6. "Make illegal states unrepresentable."**  
Prevents: Defensive null checks and boolean guards scattered throughout the codebase.  
Example: A `Loan` that has been returned is represented as `ReturnedLoan` — it structurally cannot have an active due date because that field doesn't exist on the type.

**7. "The Law of Demeter: talk to friends, not strangers."**  
Prevents: Deep method chains that couple classes through intermediaries they shouldn't know about.  
Example: `order.getShippingAddress().getCity()` couples Order logic to Address internals — instead, `order.getShippingCity()` hides the traversal.

**8. "Tell, don't ask."**  
Prevents: Logic leaking out of objects into their callers — the caller queries state and makes decisions that the object should make itself.  
Example: Instead of `if (account.getBalance() >= amount) account.deduct(amount)`, call `account.debit(amount)` and let Account enforce its own invariants.

**9. "Single level of abstraction per method."**  
Prevents: Methods that mix high-level orchestration with low-level implementation details — making them hard to read, test, and change.  
Example: `processOrder()` calls `validatePayment()`, `reserveInventory()`, `sendConfirmation()` — none of these contain string manipulation or SQL.

**10. "Prefer small, focused interfaces over fat interfaces."**  
Prevents: Classes being forced to implement methods they don't need, creating dead code and misleading contracts.  
Example: `Printable` and `Scannable` separately are better than `AllInOne` — a printer that can't scan doesn't implement `scan()`.

**11. "High cohesion, low coupling."**  
Prevents: Classes that do too many unrelated things (low cohesion) and classes that are too dependent on each other (high coupling) — both make the system rigid.  
Example: `InvoiceGenerator` only generates invoices — it doesn't send emails or update the database; those are separate, loosely coupled services.

**12. "Don't repeat yourself (DRY)."**  
Prevents: Bug fixes that must be applied in 5 places because the same logic was duplicated.  
Example: Fine calculation logic lives in exactly one `FineCalculationStrategy` — not copied into `LoanService`, `OverdueNotificationService`, and the admin portal.

**13. "You aren't gonna need it (YAGNI)."**  
Prevents: Accidental complexity from over-engineering features that were never required.  
Example: Design the parking lot for the three vehicle types in scope — do not add a `SpaceshipParkingSpot` subclass because "we might need it someday."

**14. "Fail fast."**  
Prevents: Errors propagating deep into a system and causing corrupt state far from where the bad input entered.  
Example: Validate constructor arguments immediately and throw `IllegalArgumentException` — don't accept a null `userId` and fail 10 method calls later with a `NullPointerException`.

**15. "Separate what changes often from what changes rarely."**  
Prevents: Forced modifications to stable, well-tested code every time a business rule evolves.  
Example: The `Order` entity structure is stable; the discount rules change every quarter — put discount rules in a Strategy that can be swapped without touching `Order`.

---

## Section 5: Last 30-Minute Review

**This is a memory-jog for someone who has completed the course. Do not use this to learn anything new.**

---

**Minutes 0–5: SOLID in 60 seconds each**
- SRP: One reason to change. Who changes it?
- OCP: Add new class, don't edit existing.
- LSP: Subtype must be substitutable. No `UnsupportedOperationException`.
- ISP: No method a client doesn't use. Split fat interfaces.
- DIP: Both high and low level depend on abstraction. Constructor inject the interface.

---

**Minutes 5–12: The 18 patterns — one line each**

*Creational:*
- Singleton: One instance, globally accessible, thread-safe.
- Factory Method: Subclass decides what to instantiate.
- Abstract Factory: Family of related objects, no concrete types.
- Builder: Step-by-step construction of complex object.
- Prototype: Clone instead of construct. Shallow vs deep.
- Object Pool: Reuse expensive objects. Acquire/release.

*Structural:*
- Adapter: Convert incompatible interfaces. Object adapter preferred.
- Bridge: Separate abstraction and implementation. Two hierarchies.
- Composite: Tree structure, uniform treatment of leaf and node.
- Decorator: Wrap to add behavior. Implements same interface.
- Facade: Simple interface to complex subsystem.
- Flyweight: Share intrinsic state. Extrinsic state passed in.
- Proxy: Control access. Virtual, protection, remote.

*Behavioral:*
- Chain of Responsibility: Pass request along handler chain.
- Command: Encapsulate operation as object. Supports undo.
- Iterator: Traverse without exposing structure.
- Mediator: Centralize n-to-n communication.
- Memento: Capture and restore state without breaking encapsulation.
- Observer: Notify dependents on state change. Push vs pull.
- State: Change behavior when state changes. No switch.
- Strategy: Interchangeable algorithms. Swap at runtime.
- Template Method: Invariant skeleton, variable steps in subclasses.

---

**Minutes 12–18: Case study key decisions (one per system)**
- Parking Lot: Strategy for pricing, Factory for spot type, State for spot status.
- Library: Strategy for fines, Observer for notifications, State for loan status.
- Hotel: Builder for booking, State for reservation, Strategy for pricing.
- Ride Sharing: Strategy for matching + pricing, State for trip lifecycle, Observer for status events.
- ATM: State for ATM mode, Command for transactions, Chain of Responsibility for validation.

---

**Minutes 18–22: The 5 mental models to anchor everything**
1. Depend on abstractions, not concretions.
2. Encapsulate what varies.
3. Open for extension, closed for modification.
4. Favor composition over inheritance.
5. Make illegal states unrepresentable.

---

**Minutes 22–26: Interview process reminders**
- First 2 minutes: Ask clarifying questions. Do not skip this.
- Confirm scope: "Is X in scope? What about Y?"
- State your assumptions out loud.
- Name the pattern when you apply it and say why.
- When asked to add a feature: say "Here's how my current design handles that" before writing code.

---

**Minutes 26–30: The 3 things to avoid**
- Do not jump to implementation before the class design is settled.
- Do not present a closed design — always show at least one extension point.
- Do not say "I think it's fine" when asked what you'd improve. Have three answers ready.

**Stop here. You're ready.**

---

## Section 6: Post-Interview Debrief Template

Complete this within 30 minutes of finishing a practice interview. Honest answers compound into significant improvement over multiple sessions.

---

### What Went Well

1. Which part of the design came most naturally to you, and why do you think that was?
2. Which pattern did you apply most confidently, and was the application correct?
3. Did you ask clarifying questions before designing? If yes, were they the right questions?
4. Was there a moment where you made a good trade-off call and explained it clearly?
5. Did you finish within the time limit? What did you prioritize correctly?

---

### What Went Poorly

6. Where did the design feel uncertain or forced? What was the root cause — missing knowledge, or failure to apply known knowledge under pressure?
7. Did you violate any SOLID principle in your final design? Which one, and where?
8. Was there a pattern that would have improved your design that you didn't use? Which one, and why didn't you reach for it?
9. Did you produce any God Classes — classes with more than one responsibility?
10. Were there any `if/else` or `switch` chains in your design that a State or Strategy pattern should have replaced?

---

### Design Decisions

11. For your three most important design decisions, can you now articulate the trade-off clearly? Write the trade-off statement: "I chose X over Y because Z, with the downside that W."
12. Which part of your design is hardest to extend? What would you change to fix that?
13. Was your inheritance vs composition balance correct? Did you have any IS-A relationships that should have been HAS-A?
14. Did you design for the scale and constraints of the problem, or did you over-engineer or under-engineer?
15. If the interviewer asked you to add a new feature, how did your design handle it? Would adding it require modifying existing classes?

---

### Communication

16. Did you narrate your reasoning as you designed, or did you design silently and explain only at the end?
17. When you made a decision, did you state the alternatives you considered and why you rejected them?
18. Were there moments where you lost the thread of what you were explaining? What caused it?
19. Did you use correct vocabulary throughout (cohesion, coupling, encapsulation, the pattern names)?

---

### Time Management

20. Draw a rough timeline of the 45 minutes: how long on clarification, class design, coding, extensibility discussion. Was it well-proportioned?
21. Did you run out of time before finishing a core class? If yes, what should you have deferred?
22. Did you spend too long on a low-value part of the design (e.g., utility methods, getters/setters) at the expense of core structure?

---

### Action Items for Next Time

23. Name one specific concept to review before your next session (a pattern, a principle, a case study element).
24. Name one specific behavioral change to make in your next session (e.g., "ask three clarifying questions before drawing anything", "name the pattern and justify it every time I apply one").
25. On a scale of 1–10, how would a senior engineer at your target company rate this performance? What is the gap between your rating and 8/10, and what specifically closes it?
