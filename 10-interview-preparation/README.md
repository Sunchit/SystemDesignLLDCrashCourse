# Module 10 — Interview Preparation

> The gap between knowing design and performing design under pressure is not a knowledge gap. It is a practice gap. This module closes it.

---

## Table of Contents

1. [Module Overview](#1-module-overview)
2. [The 4 Phases of LLD Interview Preparation](#2-the-4-phases-of-lld-interview-preparation)
   - [Phase 1: Concept Review](#phase-1-concept-review)
   - [Phase 2: Pattern Practice](#phase-2-pattern-practice)
   - [Phase 3: Mock Interviews](#phase-3-mock-interviews)
   - [Phase 4: Self-Evaluation](#phase-4-self-evaluation)
3. [How LLD Interviews Differ from System Design Interviews](#3-how-lld-interviews-differ-from-system-design-interviews)
4. [Time Investment Guide](#4-time-investment-guide)
5. [How This Module Maps to the Rest of the Course](#5-how-this-module-maps-to-the-rest-of-the-course)
6. [Quick-Start Checklist for 2-Week Preparation](#6-quick-start-checklist-for-2-week-preparation)

---

## 1. Module Overview

### Purpose

This module is the final preparation layer before you walk into a Senior Software Engineer interview that includes a Low-Level Design (LLD) or Object-Oriented Design (OOD) component. Every module that came before it built your design knowledge. This module converts that knowledge into interview performance.

There is an important distinction to hold here. A candidate who has internalized SOLID principles, studied fifteen design patterns, and read through case studies can still perform poorly in an LLD interview. The reason is almost never a lack of knowledge. It is one or more of the following: not knowing how to open a blank problem, losing the thread under time pressure, producing a design that is internally inconsistent without noticing, failing to communicate reasoning in real time, or not knowing what the interviewer is actually scoring.

This module directly targets each of those failure modes.

### What This Module Contains

```
10-interview-preparation/
├── README.md                          <- You are here
├── mock-interviews/                   <- Structured mock session formats
│   ├── session-format.md
│   ├── self-evaluation-rubric.md
│   └── worked-examples/
├── assignments/                       <- Timed practice problems
│   ├── easy/
│   ├── medium/
│   └── hard/
├── behavioral/                        <- Design decision justification practice
│   ├── design-decision-questions.md
│   └── tradeoff-articulation-guide.md
└── question-bank/                     <- 100+ categorized LLD questions
    ├── by-domain.md
    ├── by-pattern.md
    └── by-difficulty.md
```

### How to Navigate This Module

This README is the primary navigation document. Read it in full before engaging with any sub-directory. It provides the preparation strategy; the sub-directories provide the material to execute that strategy.

If you are short on time, read [Section 6: Quick-Start Checklist](#6-quick-start-checklist-for-2-week-preparation) immediately after this overview and return to the full module after your interview.

If you have 4 to 6 weeks, read this document top to bottom and follow the [Time Investment Guide](#4-time-investment-guide).

---

## 2. The 4 Phases of LLD Interview Preparation

Effective LLD interview preparation follows a specific sequence. Candidates who skip phases or work through them in the wrong order consistently underperform relative to their knowledge level. The phases are not stages to complete and forget — they are a cycle. Most candidates will complete one full loop in 4 to 6 weeks and one abbreviated loop in the final week before the interview.

```
Phase 1: Concept Review
        |
        v
Phase 2: Pattern Practice
        |
        v
Phase 3: Mock Interviews
        |
        v
Phase 4: Self-Evaluation
        |
        v (loop back to Phase 1 for identified weak spots)
Phase 1: Targeted Concept Review
```

---

### Phase 1: Concept Review

**Objective**: Ensure that your foundational knowledge is accurate, current in your working memory, and retrievable under pressure — not just passively familiar.

#### What to Review

The concept review phase covers four distinct knowledge areas. Each area has a different review strategy.

**OOP Fundamentals**

The following constructs must be articulable in real time without hesitation. "Articulable" means you can explain the concept, give a Java example, state when you would choose it, and state when you would not.

| Construct | What You Must Be Able to Do |
|---|---|
| Encapsulation | Explain it beyond "private fields." Explain it as controlling the contract of state mutation. |
| Abstraction | Distinguish interface-as-abstraction from class-as-abstraction. Know when each applies. |
| Inheritance | State the Liskov Substitution Principle from memory and explain when inheritance violates it. |
| Polymorphism | Demonstrate both compile-time and runtime polymorphism in Java. Know why runtime polymorphism is the more design-relevant form. |
| Composition | Build a non-trivial example that uses composition where a naive engineer would use inheritance. |
| Interface vs. Abstract Class | Give the decision rule. Know why Java 8 default methods blurred — but did not eliminate — the distinction. |

A common failure in Phase 1 is reviewing these concepts only at the definitional level. Interviewers do not ask for definitions. They put you in a situation where the correct choice requires these concepts and observe whether you make that choice. Review through application, not recitation.

**SOLID Principles**

For each principle, you should be able to:

1. State the principle in one sentence without jargon
2. Show a concrete Java violation and explain why it is a violation
3. Refactor the violation to a compliant design and explain the improvement
4. Name a real-world scenario (not a textbook example) where the principle applies

The principles most commonly exercised in LLD interviews, in order of frequency:

1. **Single Responsibility Principle** — the interviewer gives you a class that does too much, or gives you a problem where the naive solution would produce a class that does too much. You are expected to identify the separate responsibilities and split them.

2. **Open/Closed Principle** — most design pattern application questions are, at their root, OCP questions. "How would you add a new type of notification?" is an OCP question.

3. **Dependency Inversion Principle** — every meaningful use of interfaces in your design should have a DIP rationale.

4. **Liskov Substitution Principle** — inheritance hierarchies that appear in your design will be implicitly evaluated against LSP. Violations show up as code that checks `instanceof` or casts down the hierarchy.

5. **Interface Segregation Principle** — fat interfaces are a common beginner mistake in LLD. ISP violations signal inexperience faster than almost any other mistake.

**UML and Design Notation**

Interview-speed UML is different from specification-level UML. You are not producing ISO-compliant diagrams. You are producing diagrams fast enough that your hand does not fall behind your thinking, and clearly enough that an interviewer can follow your reasoning in real time.

For class diagrams, you must be able to draw:

- Class boxes with `+`, `-`, `#` visibility notation
- Inheritance arrows (solid line, hollow triangle)
- Interface implementation arrows (dashed line, hollow triangle)
- Association, aggregation, and composition with correct line terminations
- Multiplicity notation (`1`, `*`, `0..1`, `1..*`)

For sequence diagrams, you must be able to draw:

- Lifelines and activation boxes
- Synchronous and asynchronous message arrows
- Return arrows
- Loops and conditionals (even if you use an informal shorthand)

The target speed is: a class diagram with 6 to 8 classes in under 8 minutes. Time yourself. Most candidates are slower than they think.

**Core Design Patterns**

The following patterns appear in LLD interviews at high frequency. They should be at recall speed — meaning you can sketch the structure and name a concrete application in under 60 seconds.

| Priority | Pattern | Why It Appears |
|---|---|---|
| High | Strategy | Any "interchangeable algorithms" requirement — payment methods, sorting strategies, pricing rules |
| High | Observer | Any event/notification requirement — order status updates, UI events, audit logging |
| High | Factory Method / Abstract Factory | Any "create objects without specifying exact class" requirement |
| High | Builder | Complex object construction — order objects, config objects, query builders |
| High | Decorator | Any "add behavior dynamically" requirement — middleware, logging wrappers, caching layers |
| High | Singleton | Discussed even when not used — interviewers often ask why you did not use it |
| High | Command | Undo/redo, task queuing, request logging |
| Medium | Adapter | Integrating incompatible interfaces — legacy system integration, third-party wrappers |
| Medium | Proxy | Access control, lazy loading, caching |
| Medium | Chain of Responsibility | Request pipelines, middleware chains, approval workflows |
| Medium | State | Any entity with a defined lifecycle — order state machine, vending machine, ATM |
| Medium | Template Method | Algorithms with a fixed skeleton but variable steps |
| Medium | Composite | Tree structures — file systems, organization hierarchies, UI component trees |
| Lower | Facade | Large subsystem simplification |
| Lower | Flyweight | Large numbers of fine-grained objects — rarely the primary point of an LLD interview |
| Lower | Memento | Undo/redo when full object state must be captured |
| Lower | Visitor | Operations over object structures — rarely asked directly |

"Recall speed" is not the same as "memorized code." The goal is to immediately recognize when a pattern applies, sketch its structure, and begin writing the relevant classes. You should not need to reconstruct the pattern from first principles under interview conditions.

#### Phase 1 Review Strategy

Do not read passively. For each concept, write a small Java class or snippet from scratch without referring to notes. If you cannot do it from scratch, your recall is not at interview speed — it is at open-book speed, which is insufficient.

Allocate review sessions by area:

| Area | Recommended Review Time |
|---|---|
| OOP Fundamentals | 3–4 hours |
| SOLID Principles (all 5) | 5–6 hours |
| UML Speed Drawing | 2–3 hours |
| High-Priority Patterns (7) | 6–8 hours |
| Medium-Priority Patterns (5) | 4–5 hours |

---

### Phase 2: Pattern Practice

**Objective**: Develop the ability to recognize which patterns apply to a given problem under time pressure, and to begin applying them fluently without deliberate effort.

Pattern recognition is a separate skill from pattern knowledge. You can know the Strategy pattern in detail and still fail to reach for it when a problem is phrased as "the system needs to support multiple discount calculation methods depending on user tier." Phase 2 builds the pattern recognition reflex.

#### The Recognition Problem

Most candidates study patterns through canonical examples: Strategy with sorting algorithms, Observer with event listeners. These canonical examples create a narrow recognition template. When an interview problem uses a different domain or different vocabulary, the recognition fails.

The solution is pattern practice across multiple domains. For each high-priority pattern, you should practice applying it to at least three different problem domains.

**Strategy Pattern — Cross-Domain Practice**

The core recognition signal for Strategy: "the algorithm or behavior needs to vary independently of the class that uses it."

| Domain | Problem Statement Phrasing |
|---|---|
| E-commerce | "The pricing calculation needs to differ for premium, standard, and trial users." |
| Notifications | "We need to support email, SMS, and push notification channels, and the channel used may change per user." |
| File processing | "The compression algorithm should be configurable at runtime." |
| Logging | "Log output format (JSON, CSV, plain text) should be selectable per environment." |
| Authentication | "The system needs to support OAuth, SAML, and password-based authentication." |

```java
// Strategy interface — the variation point is extracted behind a contract
public interface PricingStrategy {
    Money calculate(Order order, Customer customer);
}

// Each algorithm is encapsulated in its own class
public class PremiumPricingStrategy implements PricingStrategy {
    @Override
    public Money calculate(Order order, Customer customer) {
        return order.getSubtotal().multiply(0.80); // 20% discount
    }
}

public class StandardPricingStrategy implements PricingStrategy {
    @Override
    public Money calculate(Order order, Customer customer) {
        return order.getSubtotal();
    }
}

// The context delegates to the strategy — it does not know which one
public class OrderPricingService {
    private final PricingStrategy pricingStrategy;

    public OrderPricingService(PricingStrategy pricingStrategy) {
        this.pricingStrategy = pricingStrategy;
    }

    public Money calculateFinalPrice(Order order, Customer customer) {
        return pricingStrategy.calculate(order, customer);
    }
}
```

The key interview insight: the context class (`OrderPricingService`) satisfies OCP — you can add a new pricing strategy without modifying it. This is the argument you make to the interviewer.

**Observer Pattern — Cross-Domain Practice**

The core recognition signal for Observer: "when X changes, multiple other things need to react, and the set of things that react may grow or change."

| Domain | Problem Statement Phrasing |
|---|---|
| Order management | "When an order is placed, inventory must be decremented, a confirmation email sent, and analytics updated." |
| Stock trading | "Multiple displays need to update when a stock price changes." |
| User management | "When a user is deactivated, their active sessions must be terminated and their team must be notified." |
| Ride-sharing | "When a driver accepts a ride, the passenger, the billing system, and the tracking system all need to be notified." |

```java
// The event being published
public class OrderPlacedEvent {
    private final Order order;
    private final Instant timestamp;

    public OrderPlacedEvent(Order order) {
        this.order = order;
        this.timestamp = Instant.now();
    }

    public Order getOrder() { return order; }
    public Instant getTimestamp() { return timestamp; }
}

// Observer contract
public interface OrderEventListener {
    void onOrderPlaced(OrderPlacedEvent event);
}

// Subject — owns the listener list and publishes events
public class OrderService {
    private final List<OrderEventListener> listeners = new ArrayList<>();

    public void addListener(OrderEventListener listener) {
        listeners.add(listener);
    }

    public Order placeOrder(Cart cart, Customer customer) {
        Order order = createOrder(cart, customer);
        persist(order);
        OrderPlacedEvent event = new OrderPlacedEvent(order);
        listeners.forEach(l -> l.onOrderPlaced(event));
        return order;
    }

    private Order createOrder(Cart cart, Customer customer) { /* ... */ return null; }
    private void persist(Order order) { /* ... */ }
}

// Each observer handles one concern — SRP is maintained
public class InventoryReservationListener implements OrderEventListener {
    private final InventoryService inventoryService;

    public InventoryReservationListener(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }

    @Override
    public void onOrderPlaced(OrderPlacedEvent event) {
        event.getOrder().getLineItems()
             .forEach(item -> inventoryService.reserve(item.getProductId(), item.getQuantity()));
    }
}

public class ConfirmationEmailListener implements OrderEventListener {
    private final EmailService emailService;

    public ConfirmationEmailListener(EmailService emailService) {
        this.emailService = emailService;
    }

    @Override
    public void onOrderPlaced(OrderPlacedEvent event) {
        emailService.sendOrderConfirmation(event.getOrder());
    }
}
```

The interview insight: new reactions to order placement are addable without touching `OrderService`. The subject and observers are decoupled through the event interface.

#### The Pattern Application Drill

For each of the high-priority patterns, practice the following drill three times per pattern, using different domains each time:

1. Read a 3-to-5-sentence problem description.
2. Identify which pattern or patterns apply within 60 seconds.
3. Sketch the class structure (on paper or whiteboard) in under 5 minutes.
4. Write the core classes from scratch in under 15 minutes.
5. State the OCP or SOLID rationale for the pattern choice in two sentences.

The 60-second identification step is the most important. It simulates the mental shift you need to make in an interview immediately after the problem is stated.

#### Compound Pattern Recognition

Senior-level LLD problems typically require multiple patterns working together. The ability to recognize compound pattern usage is what distinguishes a Senior-bar response from a mid-level one.

Common combinations:

| Combination | Typical Problem Signal |
|---|---|
| Factory + Strategy | "Create different processor types based on input, and each processor runs a different algorithm." |
| Observer + Command | "When an event occurs, queue operations that can be retried or rolled back." |
| Builder + Factory | "Construct complex objects whose type depends on configuration." |
| Decorator + Strategy | "Add behaviors to processing pipeline steps, where each step is interchangeable." |
| State + Observer | "When the entity transitions to a new state, notify dependent components." |
| Composite + Visitor | "Traverse a tree structure and perform different operations at different nodes." |

Practice recognizing at least five of these combinations before Phase 3.

---

### Phase 3: Mock Interviews

**Objective**: Simulate real interview conditions accurately enough that the actual interview feels familiar, not foreign. Develop the discipline of communicating design decisions in real time while designing.

#### Why Mock Interviews Are Non-Negotiable

There is a capability gap that no amount of solo study closes: the gap between knowing a design and producing a design on demand while talking. In a real LLD interview, you are simultaneously:

- Processing the problem statement
- Asking clarifying questions
- Sketching a class hierarchy
- Speaking your reasoning aloud
- Listening to the interviewer's follow-up questions
- Adjusting your design in response

Each of these is a manageable skill individually. Combined under time pressure with a stranger evaluating you, they create interference that surprises almost every candidate the first time. Mock interviews are how you convert that interference from paralyzing to manageable.

#### The Mock Interview Format

Each mock session should follow this structure precisely. Deviating from it reduces the fidelity of the practice.

**Pre-session setup (5 minutes)**

- Set a timer for 45 minutes total.
- Have paper, a whiteboard, or a drawing tool ready.
- If practicing with another person, the interviewer should have the problem statement but the candidate should not.
- If practicing alone, use a problem you have not seen before. Do not use a problem you have solved previously.

**Requirements gathering (5–8 minutes)**

The interviewer presents the problem in one or two sentences. The candidate asks clarifying questions. The discipline here is asking questions that are both necessary for design and demonstrating structured thinking.

Good clarifying questions:
- "Should this support concurrent access, or is single-user scope sufficient for this problem?"
- "When you say 'notifications,' should I model the delivery mechanism, or treat that as an external dependency?"
- "Are there any read-heavy versus write-heavy assumptions I should design for?"
- "How many discount types exist today, and is extensibility to new types a requirement?"

Poor clarifying questions:
- "What should I use — an interface or an abstract class?" (Design decision, not a requirements question)
- "Should I use the Strategy pattern here?" (The pattern choice is yours to make)
- "Can you give me an example?" (Ask specific, targeted questions, not open requests for more information)

**Core design (20–25 minutes)**

Produce a class hierarchy, draw the key class relationships, and write the core classes in code. The sequence:

1. Identify the primary entities from the requirements. (3 minutes)
2. Identify the relationships between entities. (3 minutes)
3. Draw the class diagram. (5 minutes)
4. Identify the variation points and apply appropriate patterns. (3 minutes)
5. Write the core classes in code — interfaces, key concrete classes, the most important method signatures. (10–12 minutes)

Throughout this phase, speak every decision aloud: "I'm making `PaymentProcessor` an interface rather than an abstract class because the different processors don't share any implementation — they only share a contract." This narration is part of what the interviewer is evaluating.

**Extension discussion (5–8 minutes)**

The interviewer introduces one or two extension requirements: "How would you add a loyalty points system?" or "How would you extend this to support international currencies?" The candidate explains, and optionally sketches, how the existing design accommodates the extension.

A Senior-bar response demonstrates that the extension was anticipated in the original design (even if not implemented), and that the existing class boundaries either already support it or require minimal, localized change.

**Debrief (5 minutes)**

If practicing with a partner, the interviewer gives feedback using the self-evaluation rubric. If practicing alone, immediately score yourself using the rubric before looking at any reference material.

#### Self-Evaluation Rubric

Score each dimension from 1 to 4. A score of 3 on all dimensions is the Senior bar at most companies. A score of 4 on the majority of dimensions is the Senior+ or Staff bar.

| Dimension | 1 — Below Bar | 2 — Approaching Bar | 3 — At Bar | 4 — Above Bar |
|---|---|---|---|---|
| **Requirements Clarity** | Did not ask clarifying questions, or asked questions that revealed confusion | Asked some clarifying questions but missed important scope decisions | Identified all critical scope ambiguities and resolved them efficiently | Identified non-obvious ambiguities; asked questions that revealed design expertise |
| **Entity Identification** | Missed key entities or introduced irrelevant entities | Found primary entities but missed secondary ones or included noise | Correctly identified all necessary entities and their primary attributes | Identified the correct entities and made explicit tradeoffs about which attributes belong in the domain vs. the infrastructure layer |
| **Class Hierarchy Design** | Classes have multiple responsibilities; hierarchy is unclear or inconsistent | Single responsibilities are present but some classes are over-coupled | Each class has a single, clear responsibility; hierarchy is coherent and traversable | The hierarchy demonstrates forward-looking extensibility; variation points are explicitly named |
| **Pattern Application** | No patterns applied, or patterns applied incorrectly | One appropriate pattern applied; others missed | Patterns applied where they earn their complexity; no over-engineering | Patterns are applied with explicit SOLID justification; compound patterns are handled cleanly |
| **Communication** | Silent during design; decisions are not explained | Some decisions explained after the fact | Design decisions are narrated in real time and are accurate | Narration includes proactive tradeoff discussion; alternatives are named and dismissed with reasons |
| **Extension Handling** | Extension breaks the existing design; major restructuring needed | Extension is possible but requires modification of existing classes | Extension is additive — new classes without modifying existing ones | Extension was evidently anticipated in the original design structure |
| **Code Quality** | Code does not compile or is missing critical pieces | Code compiles but uses poor naming or unclear structure | Code is clean, well-named, and demonstrates Java idioms correctly | Code is production-quality: generic types used appropriately, immutability considered, null handling explicit |
| **UML Accuracy** | Diagram is absent or incorrect | Diagram present but uses incorrect relationship notation | Diagram correctly represents the design with standard notation | Diagram uses sequence diagrams or activity diagrams in addition to class diagrams where they add clarity |

A score of 2 in any dimension signals a specific gap to address in Phase 4. Do not average your scores and treat that as your overall performance — individual dimension gaps are more actionable than an average.

#### Recommended Mock Interview Schedule

| Week | Sessions | Format |
|---|---|---|
| Week 3 | 2 sessions | Easy problems; self-evaluation only |
| Week 4 | 3 sessions | Medium problems; at least 1 with a partner |
| Week 5 | 4 sessions | Medium and hard problems; at least 2 with a partner |
| Week 6 (pre-interview) | 2–3 sessions | Simulate interview conditions precisely; no notes |

---

### Phase 4: Self-Evaluation

**Objective**: Accurately identify your current weakest dimensions, target preparation resources specifically at those dimensions, and calibrate your self-assessment to the Senior bar rather than to an abstract personal standard.

#### The Calibration Problem

Most candidates are poor at self-assessment in LLD for one of two reasons:

1. They evaluate their designs against their own intuition, which was formed during the same period their habits were established. If they have been writing procedural code disguised as OOP for three years, their intuition tells them that procedural-OOP is "good enough."

2. They know the rules well enough to evaluate their own code in slow, deliberate review, but they do not know where their performance degrades under time pressure. Speed surfaces different weaknesses than careful review.

The self-evaluation rubric in Phase 3 is designed to address the first problem. The second problem is addressed by ensuring that mock interviews are conducted under realistic time pressure.

#### Mapping Rubric Scores to Preparation Actions

| Dimension with Score of 2 | Root Cause | Targeted Action |
|---|---|---|
| Requirements Clarity | Have not practiced structured question-asking | Study the `behavioral/design-decision-questions.md` file. Practice asking requirements questions on five problems without designing them. |
| Entity Identification | Weak domain modeling skill | Return to `06-case-studies/` and, before reading the solution, independently list all entities. Compare against the case study. |
| Class Hierarchy Design | SRP is not yet reflexive | Return to `01-solid-principles/` SRP material. Complete all violation-detection exercises. |
| Pattern Application | Pattern recognition reflex is not developed | Complete Phase 2 pattern recognition drills for the failing patterns. |
| Communication | Unused to narrating design in real time | Practice solo — talk aloud while designing even when no one is listening. Record yourself and review. |
| Extension Handling | OCP is not yet applied proactively | Study the OCP material in `01-solid-principles/`. Review the extension discussion sections of three advanced case studies. |
| Code Quality | Java fluency gaps | Study Java interfaces, generics, and Optional handling independently. These are prerequisites, not LLD skills. |
| UML Accuracy | Insufficient diagram practice | Use the UML exercises in `03-uml-modeling/exercises/` and time yourself. |

#### Calibrating to the Senior Bar

The Senior bar is not what you think it is if you have not conducted interviews yourself. Here is what distinguishes a hire from a no-hire at the Senior level, stated plainly:

**A hire demonstrates**:
- A design that can be explained to a colleague in five minutes and understood
- Classes that have one reason to change and one reason to exist
- Patterns applied because the problem called for them, with an explanation of what problem the pattern solves
- Extension handling that is additive, not surgical
- Code that could go into a PR without a significant rewrite

**A no-hire typically demonstrates**:
- God classes that own all the business logic
- Inheritance hierarchies that use inheritance for code reuse rather than behavioral subtyping
- Patterns applied because the candidate "knows" them, not because the problem requires them
- A design that breaks the moment a single extension requirement is introduced
- Code with ambiguous method names, `boolean` parameters that switch behavior, and null returns from methods that should signal absence through the type system

The most common Senior no-hire that candidates are surprised by is the over-engineered design: a candidate who introduces three patterns where one would suffice, creates abstractions for things that will never have a second implementation, and adds extension points for scenarios that were never mentioned. This reads as insecurity, not sophistication. A Senior engineer's instinct is toward the simplest design that correctly handles the requirements — and that can be cleanly extended if requirements change. YAGNI applies even in interviews.

---

## 3. How LLD Interviews Differ from System Design Interviews

This distinction matters because the preparation, the deliverables, the evaluation criteria, and the failure modes are fundamentally different. Candidates who have strong System Design preparation and assume it transfers to LLD interviews consistently underperform.

### Scope: What Level of the Stack Are You Designing?

| Dimension | LLD Interview | System Design (HLD) Interview |
|---|---|---|
| Unit of design | Classes, interfaces, and their relationships | Services, databases, message queues, and their relationships |
| Granularity | Method signatures, class responsibilities, object interactions | API contracts, data stores, network topology, replication strategy |
| Code expectation | Write real code — method bodies, not just signatures | No code expected; may write pseudo-schemas |
| Primary artifact | Class diagram + working code | Architecture diagram |
| Pattern vocabulary | GoF design patterns, OOP principles | Distributed systems patterns (CQRS, event sourcing, saga, etc.) |
| Scaling concern | Extensibility, testability, changeability | Throughput, latency, availability, consistency |
| Evaluation lens | Is the design clean, extensible, and well-reasoned? | Is the architecture scalable, resilient, and operationally sound? |

A critical mistake is treating an LLD interview as a scaled-down system design interview. They are not on the same dimension. An LLD interview is not "design a small system" — it is "design the class structure of a system." You can spend the entire 45 minutes on a single service's internal design without discussing load balancing, databases, or microservice boundaries.

### What Interviewers Are Scoring

#### In a System Design Interview, the interviewer scores:

- Requirement scoping — can you narrow "design Twitter" to something tractable?
- Capacity estimation — can you reason about scale?
- Component selection — SQL vs. NoSQL, sync vs. async, monolith vs. microservices
- Trade-off articulation — why did you choose eventual consistency here?
- Back-of-envelope arithmetic — can you validate your choices numerically?

#### In an LLD Interview, the interviewer scores:

- OOP correctness — are your abstractions cohesive?
- SOLID compliance — does your design survive the standard refactoring critiques?
- Pattern recognition — did you identify the variation points and apply the right patterns?
- Extension readiness — can new requirements be accommodated without surgery?
- Code quality — would this code pass a Senior Engineer's code review?
- Communication — can you explain your reasoning in real time, or only after the fact?

Notably absent from the LLD scoring criteria: performance, scalability, deployment topology, and database design. These are not relevant to the LLD round.

### Time Constraints and Expected Output

A typical LLD interview at a Senior Engineer level runs 45 minutes. Here is how that time is expected to distribute:

| Segment | Duration | Expected Output |
|---|---|---|
| Problem presentation and clarification | 5–8 minutes | A shared, written list of confirmed requirements and explicitly out-of-scope decisions |
| Entity and relationship identification | 5–7 minutes | A rough entity list with primary relationships noted |
| Class hierarchy design | 10–12 minutes | A class diagram with at minimum: classes, interfaces, key attributes, and relationship arrows |
| Core implementation | 12–15 minutes | Working code for 3 to 5 core classes, including interface definitions and at least one concrete implementation per interface |
| Extension and follow-up discussion | 5–8 minutes | Verbal explanation of how the design handles 1–2 extension scenarios, potentially with a diagram sketch |

The implication is that you have approximately 12 minutes to write code. That is not enough time to write all the code for a system. It is enough time to write the code that demonstrates your design decisions. Prioritize:

1. Interface and abstract class definitions (these show your abstraction strategy)
2. The most pattern-critical concrete class (this shows you can implement what you designed)
3. The class that ties others together — the context class, the service class, the factory

Do not prioritize:
- Utility methods
- Getter/setter boilerplate (you can note "standard getters" and move on)
- Error handling in detail (acknowledge it exists and move on unless it is the focus of the question)
- Persistence layer code

### Expected Deliverables vs. HLD Deliverables

| Deliverable | LLD Interview | HLD Interview |
|---|---|---|
| Primary diagram | Class diagram with UML notation | Architecture diagram with service boxes and arrows |
| Secondary diagram | Sequence diagram for key interactions | Data flow diagram or deployment diagram |
| Code | Required — write actual Java (or chosen language) | Not required — may write pseudo-schema or API spec |
| Design decisions | Stated as OOP principle arguments ("I used Strategy here because the algorithm needs to vary independently of the client, which satisfies OCP") | Stated as system tradeoff arguments ("I chose eventual consistency here because the alternative requires distributed transactions") |
| Scalability discussion | Code-level — "this design scales by adding new Strategy implementations" | Infrastructure-level — "this service scales horizontally behind a load balancer" |

### Depth vs. Breadth

LLD interviews reward depth over breadth. A candidate who designs three classes with clear responsibilities, explicit interfaces, correct pattern application, and clean code is a stronger hire signal than a candidate who sketches fifteen classes at a surface level.

System design interviews reward breadth over depth. Covering all major system components at an appropriate level of abstraction is the baseline; going deep on a selected few is the evidence of seniority.

This difference has direct preparation implications. In LLD preparation, practice finishing a design rather than covering all the cases. In system design preparation, practice covering the full landscape at a reasonable depth.

### The Typical 45-Minute LLD Interview — Minute-by-Minute

The following is a realistic breakdown of how top-tier interviewers structure a 45-minute LLD session and what they are observing at each stage.

**Minutes 1–2: Problem statement**

The interviewer presents the problem. Candidates who immediately start designing have already made a mistake. The correct behavior is to listen fully, note key nouns (entities) and verbs (behaviors), and pause before speaking.

What the interviewer observes: Does the candidate listen fully, or do they interrupt with premature design assumptions?

**Minutes 3–8: Clarifying questions**

The candidate drives this phase. Good candidates ask 3 to 5 targeted questions that reduce ambiguity about scope, scale, and extensibility requirements. They do not ask questions that should be answered by design judgment.

What the interviewer observes: Are the questions evidence-based (the candidate identified a genuine ambiguity) or defensive (the candidate is delaying because they do not know how to start)?

**Minutes 9–15: Entity and relationship identification**

The candidate identifies the primary entities, their attributes, and their relationships. This should be done aloud and on paper. "I see three main entities: `ParkingLot`, `ParkingSpot`, and `Vehicle`. A `ParkingLot` contains multiple `ParkingSpot`s (composition, because spots do not exist outside a lot). A `Vehicle` is associated with at most one `ParkingSpot` at any time."

What the interviewer observes: Is the entity list correct and complete? Are relationships categorized correctly (association vs. aggregation vs. composition vs. inheritance)? Are responsibilities placed on the correct entities?

**Minutes 16–28: Class hierarchy and pattern application**

The candidate draws the class diagram and identifies variation points. For each variation point, a pattern should be applied and explained. The interviewer may ask questions during this phase — answer them directly and briefly, then return to the design.

What the interviewer observes: Does the candidate identify the variation points (or does the design bake in concrete implementations everywhere)? Are patterns applied where they solve a real extensibility problem? Is the design internally consistent?

**Minutes 29–41: Core code**

The candidate writes code for the most design-critical classes. The code should be compilable Java (or close to it). The candidate should call out tradeoffs in the code: "I'm using `Optional<ParkingSpot>` here as the return type of `findAvailableSpot` to make the absence case explicit at the type level rather than returning null."

What the interviewer observes: Does the code match the class diagram? Are the Java constructs used correctly (generics, interfaces, access modifiers)? Does the code reflect the SOLID principles the candidate claimed to apply?

**Minutes 42–45: Extension and closing**

The interviewer introduces an extension: "How would you extend this to support electric vehicle charging spots?" The candidate explains — and if time permits, sketches — the extension. The correct answer describes additive change: a new subclass of `ParkingSpot`, a new `VehicleType`, a new implementation of a strategy — not a change to existing classes.

What the interviewer observes: Does the design survive the extension? Was the candidate's OCP claim accurate or aspirational?

---

## 4. Time Investment Guide

### Weekly Plan: 4-Week Preparation

For candidates with 4 weeks of preparation time. Total investment: approximately 10–12 hours per week, 40–50 hours total.

| Week | Focus | Hours | Key Activities |
|---|---|---|---|
| Week 1 | Concept Review | 10–12 hours | Phase 1 in full: OOP review, SOLID review, UML speed drawing, high-priority patterns |
| Week 2 | Pattern Practice + Case Studies | 10–12 hours | Phase 2 drills; 5 case studies (3 easy, 2 medium); begin question bank |
| Week 3 | Advanced Case Studies + Early Mocks | 11–13 hours | 4 advanced case studies; 2 timed mock interviews (solo); machine coding rounds |
| Week 4 | Mock Intensity + Gap Closure | 10–12 hours | 4–5 mock interviews (2–3 with partner); targeted review of low rubric scores; final cheat sheet run |

**Week 1 — Concept Review**

| Day | Activity | Hours |
|---|---|---|
| Day 1 | OOP fundamentals review — write each construct from scratch | 2 |
| Day 2 | SOLID review — SRP, OCP, LSP with Java examples from scratch | 2.5 |
| Day 3 | SOLID review — ISP, DIP + combined violations | 2 |
| Day 4 | UML speed drawing — class diagrams, sequence diagrams, timed practice | 2 |
| Day 5 | High-priority patterns: Strategy, Observer, Factory Method | 2.5 |
| Day 6–7 | High-priority patterns: Builder, Decorator, Singleton, Command | 3 |

**Week 2 — Pattern Practice + Case Studies**

| Day | Activity | Hours |
|---|---|---|
| Day 1 | Medium-priority patterns: Adapter, Proxy, Chain of Responsibility, State | 2.5 |
| Day 2 | Pattern recognition drill: 10 problem descriptions, identify pattern within 60 seconds each | 2 |
| Day 3 | Easy case studies: Parking Lot (independent design then review) | 2 |
| Day 4 | Easy case studies: Library Management, Vending Machine | 2 |
| Day 5 | Medium case study: ATM System (independent design with timer) | 2 |
| Day 6 | Medium case study: Movie Ticket Booking | 2 |
| Day 7 | Question bank review: 20 questions by domain | 1.5 |

**Week 3 — Advanced Case Studies + Early Mocks**

| Day | Activity | Hours |
|---|---|---|
| Day 1 | Advanced case study: Notification Framework | 2 |
| Day 2 | Advanced case study: Payment Gateway | 2 |
| Day 3 | Advanced case study: Rate Limiter | 2 |
| Day 4 | Advanced case study: Ride-Sharing System | 2.5 |
| Day 5 | Mock Interview 1 — Easy problem, solo, full 45 minutes, immediate rubric scoring | 1.5 |
| Day 6 | Machine coding: 1 timed problem from `07-machine-coding/questions/` | 2 |
| Day 7 | Review mock session rubric scores; identify gaps; study gap materials | 1.5 |

**Week 4 — Mock Intensity + Gap Closure**

| Day | Activity | Hours |
|---|---|---|
| Day 1 | Targeted review of lowest-scoring rubric dimensions from Week 3 | 2 |
| Day 2 | Mock Interview 2 — Medium problem, with partner | 1.5 |
| Day 3 | Mock Interview 3 — Medium problem, solo | 1.5 |
| Day 4 | Machine coding: 1 timed problem under 90 minutes | 2 |
| Day 5 | Mock Interview 4 — Hard problem, with partner | 1.5 |
| Day 6 | Review all cheat sheets and pattern cards from `11-artifacts/` | 1.5 |
| Day 7 | Mock Interview 5 — Simulate exact interview conditions: no notes, timer, complete rubric | 1.5 |

---

### Weekly Plan: 6-Week Preparation

For candidates who prefer deeper coverage or are targeting Staff-level interviews. Total investment: approximately 8–10 hours per week, 50–60 hours total.

| Week | Focus | Hours | Key Activities |
|---|---|---|---|
| Week 1 | Deep OOP and SOLID Review | 8–10 | All of Phase 1 OOP and SOLID at violation-detection level |
| Week 2 | UML Mastery + Full Pattern Coverage | 8–10 | All patterns including lower-priority ones; UML sequence diagrams |
| Week 3 | Case Studies — Easy and Medium | 8–10 | All 8 easy and medium case studies with independent design before review |
| Week 4 | Advanced Case Studies + Senior Design Thinking | 9–11 | All advanced case studies; Module 05 topics: domain modeling, API design |
| Week 5 | Mock Interviews + Machine Coding | 9–11 | 5 mocks (3 with partner); 3 machine coding rounds |
| Week 6 | Gap Closure + Final Mock Intensive | 8–10 | Targeted review; 4 final mocks; behavioral and decision-justification practice |

**Hours-Per-Week Breakdown by Activity Type**

| Activity Type | 4-Week Plan (weekly avg) | 6-Week Plan (weekly avg) |
|---|---|---|
| Concept review and reading | 3 hours | 3 hours |
| Coding practice (solo) | 3 hours | 2.5 hours |
| Case study work | 2.5 hours | 2.5 hours |
| Mock interviews | 1.5 hours | 1.5 hours |
| Rubric review and gap analysis | 0.5 hours | 0.5 hours |

---

## 5. How This Module Maps to the Rest of the Course

Every module in this course contributes to interview performance. This section maps each module to the specific interview competencies it builds.

### Module 01 — SOLID Principles (`01-solid-principles/`)

**Interview relevance: Critical**

SOLID principles are the most-evaluated dimension in Senior LLD interviews. They are the vocabulary through which you explain design decisions. When you say "I'm making this an interface rather than a class because the calling code should depend on the abstraction, not the implementation — that's DIP," you are demonstrating exactly what interviewers are looking for.

The specific contributions of Module 01:

- The **violation detection exercises** train you to see SRP and OCP violations in real time, which is the skill that catches mistakes in your own interview designs before the interviewer does.
- The **before/after refactoring examples** provide the vocabulary for explaining design improvements under pressure.
- The **combined violations** exercises train you to handle the most common interview trap: a design that simultaneously violates SRP, OCP, and DIP, which is the default output of candidates who have not internalized these principles.

**Module 01 sections most relevant to interview preparation**:
- SRP violation detection — apply during Phase 4 self-evaluation to identify SRP gaps
- OCP before/after examples — use as templates for the Extension Handling dimension
- DIP examples — every interface in your interview design should have a DIP justification

### Module 02 — Design Principles (`02-design-principles/`)

**Interview relevance: High**

Module 02 builds the judgment that makes your designs readable and defensible.

- **DRY** prevents you from producing redundant class hierarchies, which is a common mistake when designing under pressure.
- **KISS** prevents over-engineering — the single most common Senior-level failure mode in LLD interviews.
- **YAGNI** prevents you from adding extension points for requirements that were not stated, which reads as noise rather than sophistication.
- **Coupling and cohesion** are the lens through which you evaluate your own class hierarchy during design.
- **Dependency Injection** is not just a Spring concept — it is the mechanism by which your designs satisfy DIP and enable testability. Interviewers who ask "how would you test this?" are asking about DI.
- **Code smells** — the `05-code-smells-and-refactoring.md` material trains you to recognize when your own design is going wrong before the interviewer points it out.

**Most interview-relevant sections**:
- `02-design-principles/03-coupling-cohesion.md` — for self-evaluating class responsibility boundaries
- `02-design-principles/04-dependency-injection.md` — for constructor injection patterns in your interview code

### Module 03 — UML Modeling (`03-uml-modeling/`)

**Interview relevance: High**

The class diagram you produce in the first 20 minutes of an LLD interview is the central communication artifact. A class diagram that uses incorrect notation, misrepresents relationships, or omits key entities will cause the interviewer to question whether your design is as sound as you claim.

Specific Module 03 contributions:
- **Class diagram notation** — the `exercises/` directory provides timed drawing practice. The target is accurate class diagrams in under 8 minutes.
- **Sequence diagrams** — for the extension discussion phase of the interview, a sequence diagram showing how a new feature integrates with existing classes is often more efficient than verbal explanation.
- **UML for interviews** — the interview-speed shorthand covered in Module 03 is specifically calibrated to the constraints of an interview whiteboard.

**Most interview-relevant sections**:
- `03-uml-modeling/exercises/` — complete all timed exercises in Phase 1
- Sequence diagrams section — practice drawing message flows for Observer and Command patterns

### Module 04 — Design Patterns (`04-design-patterns/`)

**Interview relevance: Critical**

Design patterns are the tools. Phase 2 of this preparation guide is built entirely on the material in Module 04. The key distinction is that Module 04 teaches you what each pattern is and how to implement it; this module teaches you when to reach for each pattern and how to justify the choice under pressure.

The cross-reference between Module 04 and interview preparation:

| Module 04 Section | Interview Use |
|---|---|
| `04-design-patterns/creational/` | Factory and Builder patterns appear in almost every interview involving object creation with variability. Review Factory Method and Builder most thoroughly. |
| `04-design-patterns/behavioral/` | Strategy, Observer, Command, State, and Chain of Responsibility cover the majority of LLD interview variation points. These are Phase 2's core. |
| `04-design-patterns/structural/` | Adapter, Decorator, and Proxy cover the most common structural interview scenarios. Composite appears in tree-structure problems. |

**Most interview-relevant sections**:
- `04-design-patterns/behavioral/` — all five high-priority patterns, in depth
- `04-design-patterns/creational/` — Factory Method, Abstract Factory, Builder
- The "when NOT to use" sections for each pattern (avoiding over-engineering is as important as pattern application)

### Module 05 — Senior Engineer Design Thinking (`05-senior-engineer-design-thinking/`)

**Interview relevance: High — for the Senior and above distinction**

Module 05 is what separates a candidate who performs at the Senior bar from one who performs at the Senior+ or Staff bar. The concepts in this module — domain modeling, API design philosophy, error handling strategy, technical debt reasoning — are what interviewers are probing when they ask follow-up questions that go beyond the surface design.

Specific contributions to interview performance:

- **`01-extensibility-and-scalability.md`**: Directly maps to the Extension Handling rubric dimension. Candidates who have internalized this file produce designs that anticipate the interviewer's extension question.
- **`02-api-design.md`**: The method signatures in your interview code are API design decisions. Making illegal states unrepresentable, using the type system to enforce contracts, Command/Query Separation — these details distinguish Senior code from mid-level code.
- **`03-domain-modeling.md`**: The entity identification phase of the interview is a domain modeling exercise. Candidates who can articulate the difference between entities and value objects, and who can identify aggregate boundaries, demonstrate a level of domain sophistication that most candidates lack.
- **`04-error-handling-and-resilience.md`**: Interviewers frequently ask "what happens when this fails?" Candidates with weak error handling backgrounds either add boilerplate try-catch blocks or say "I'd handle errors in a real implementation." Candidates who have studied this module explain their error contract, distinguish recoverable from non-recoverable errors, and show where validation occurs.

**Most interview-relevant sections**:
- The opening section on the mid-level to senior mental model shift — reread this the day before your interview
- `02-api-design.md` — apply to every method signature you write in your interview code
- `03-domain-modeling.md` exercises — practice entity/value object identification on novel domains

### Module 06 — Real-World Case Studies (`06-case-studies/`)

**Interview relevance: Critical for volume and fluency**

The case studies in Module 06 are the primary vehicle for building design fluency. Each case study is a full end-to-end LLD problem of the type that appears in Senior interviews.

| Sub-module | Interview Use | Preparation Priority |
|---|---|---|
| `06-case-studies/easy/` (Parking Lot, Library, Vending Machine) | Warm-up designs. Use these to build the entity identification reflex. | Phase 1 and 2 |
| `06-case-studies/medium/` (ATM, Hotel Booking, Movie Ticket Booking, Food Delivery, Inventory) | Core difficulty level for most Senior interviews. Master these before the hard ones. | Phase 2 and 3 |
| `06-case-studies/advanced/` (Ride-Sharing, Splitwise, Notification Framework, Payment Gateway, Rate Limiter, Cache System, Workflow Engine, E-commerce, Logging Framework) | Appear in Staff-level interviews and in Senior interviews at top-tier companies. Use the ones most similar to your target company's domain. | Phase 3 |

The correct way to use case studies for interview preparation is to design independently before looking at the solution. Spend at least 30 minutes on the independent design. The gap between your design and the reference design is the most precise signal of where your gaps are.

### Module 07 — Machine Coding Round Preparation (`07-machine-coding/`)

**Interview relevance: High for companies that run separate machine coding rounds**

Many companies separate the LLD round into two parts: a design round (class diagrams, discussion) and a machine coding round (implement a working system in 60–90 minutes). Module 07 is preparation for the second.

The machine coding round evaluates different capabilities than the discussion-based LLD round:

| Capability | Discussion LLD Round | Machine Coding Round |
|---|---|---|
| Correctness | Class hierarchy and pattern correctness | Working code that passes test cases |
| Speed | Design a system in 45 minutes | Implement a working system in 90 minutes |
| Coverage | Design depth on key components | Working breadth — all requirements covered |
| Communication | Verbal narration of design decisions | Code self-evidence — naming and structure communicate intent |

The `07-machine-coding/questions/` directory contains problems at easy, medium, and hard difficulty. For interview preparation:

- Treat every machine coding problem as a timed exercise. Set a timer for 90 minutes. Stop when it goes off.
- Review your own code against the rubric before looking at the solution.
- The most common machine coding failure is spending too long on design and not leaving enough time for implementation. The inverse of the discussion round failure.

**Most interview-relevant content**:
- The evaluation rubric in `07-machine-coding/` — know what is being scored
- Medium-difficulty questions — the majority of machine coding rounds are at this level
- The annotated solutions — read these to see what "complete" looks like under time pressure

### Module 08 — Testing and Quality Engineering (`08-testing-quality/`)

**Interview relevance: Medium — but increasingly common at Senior level**

The testing round is less common than the design round, but many companies now ask "how would you test this?" as a standard follow-up in the LLD interview. Candidates who have studied Module 08 can answer this question fluently; candidates who have not give vague answers about "writing unit tests."

Specific interview applications:

- **Designing for testability** — the question "is this design testable?" should be answerable for your interview code. If it is not (static calls, global state, constructors with logic), the interviewer will probe it.
- **Dependency injection as a seam** — your use of constructor injection in interview code is justified both by DIP and by testability. Being able to give both reasons is a Senior signal.
- **Test naming as specification** — if the interviewer asks you to write a test, write it with a behavior-based name (`givenExpiredCreditCard_whenPaymentAttempted_thenPaymentDeclinedException`) not an implementation-based name (`testProcessPaymentFail`).
- **Mocking strategy** — knowing when to mock, what to mock, and what the risk of over-mocking is demonstrates depth that most candidates lack.

---

## 6. Quick-Start Checklist for 2-Week Preparation

This checklist is for engineers with 2 weeks or less before a Senior LLD interview. It is a ruthless prioritization of the minimum sufficient preparation path. Every item on this list is actionable today.

The checklist is organized into two weeks. Within each week, items are ordered by priority. Complete each item before moving to the next.

---

### Week 1: Knowledge Foundation

**Day 1 — OOP and SOLID Core (3 hours)**

- [ ] Write a Java example from scratch demonstrating each OOP pillar (encapsulation, abstraction, inheritance, polymorphism) — do not look up syntax
- [ ] Write the definition of each SOLID principle in one sentence without notes
- [ ] For SRP: write a Java class that violates it, then refactor it
- [ ] For OCP: write a Java example that closes for modification but opens for extension via an interface
- [ ] For DIP: write a Java class whose dependency is injected through the constructor via an interface

**Day 2 — SOLID Completion + UML Basics (3 hours)**

- [ ] For LSP: identify a Java inheritance hierarchy that violates behavioral subtyping and explain why
- [ ] For ISP: write an example of a fat interface and split it into role interfaces
- [ ] Draw a UML class diagram with at least 6 classes, using correct notation for inheritance, implementation, association, composition, and multiplicity — without looking up notation
- [ ] Time yourself drawing a class diagram: target is under 8 minutes

**Day 3 — High-Priority Patterns: Strategy, Observer, Builder (2.5 hours)**

- [ ] Write Strategy from scratch for a payment processing problem (no notes)
- [ ] Write Observer from scratch for an order notification problem (no notes)
- [ ] Write Builder from scratch for a complex order object (no notes)
- [ ] For each pattern: state which SOLID principle it satisfies and why in one sentence

**Day 4 — High-Priority Patterns: Factory, Decorator, Command (2.5 hours)**

- [ ] Write Factory Method from scratch for an alert creation problem
- [ ] Write Decorator from scratch for a logging wrapper problem
- [ ] Write Command from scratch for an undo-redo problem
- [ ] Practice pattern recognition: read 6 problem descriptions, identify the pattern within 60 seconds

**Day 5 — Medium-Priority Patterns + Case Study 1 (3 hours)**

- [ ] Study Adapter, Proxy, Chain of Responsibility, State — sketch each structure from scratch
- [ ] Complete the Parking Lot case study: design independently for 30 minutes, then compare against the reference in `06-case-studies/easy/01-parking-lot/`
- [ ] Identify which patterns you applied that match the reference and which you missed

**Day 6 — Case Studies 2 and 3 (3 hours)**

- [ ] Complete ATM System case study independently (30 minutes), then compare
- [ ] Complete Hotel Booking or Movie Ticket Booking case study independently (30 minutes), then compare
- [ ] For each case study, write down: which entities you got right, which patterns you missed, and why

**Day 7 — First Mock Interview (1.5 hours)**

- [ ] Select a problem from `10-interview-preparation/assignments/medium/` that you have not seen before
- [ ] Set a 45-minute timer and conduct a solo mock interview — speak aloud throughout
- [ ] Immediately score yourself on all 8 rubric dimensions before reviewing any reference material
- [ ] Write down the 2 lowest-scoring dimensions — these are your Week 2 priorities

---

### Week 2: Performance and Gap Closure

**Day 8 — Address Lowest-Scoring Rubric Dimension from Day 7 (2 hours)**

- [ ] If Entity Identification scored low: do the entity extraction exercise for Splitwise or Ride-Sharing independently. Compare against `06-case-studies/advanced/`
- [ ] If Pattern Application scored low: redo the pattern recognition drill with 10 new problem descriptions
- [ ] If Extension Handling scored low: study OCP material in `01-solid-principles/` and review extension sections of 2 advanced case studies
- [ ] If Code Quality scored low: write 3 interfaces and their implementations with production-quality naming and method signatures

**Day 9 — Advanced Case Study (2 hours)**

- [ ] Select the advanced case study most likely to appear in your target company's domain:
  - For e-commerce / marketplace companies: `06-case-studies/advanced/02-ecommerce-platform/`
  - For mobility / logistics companies: `06-case-studies/advanced/01-ride-sharing/`
  - For fintech companies: `06-case-studies/advanced/05-payment-gateway/`
  - For platform / infrastructure companies: `06-case-studies/advanced/08-cache-system/` or `06-case-studies/advanced/06-rate-limiter/`
- [ ] Design independently for 40 minutes, then compare against reference
- [ ] Write a 5-sentence summary of the design decisions that most surprised you in the reference

**Day 10 — Machine Coding Round (2 hours)**

- [ ] Select one problem from `07-machine-coding/questions/`
- [ ] Set a 90-minute timer and implement a working solution
- [ ] Score your solution against the machine coding rubric in `07-machine-coding/`

**Day 11 — Mock Interview with Partner (1.5 hours)**

- [ ] Conduct a full 45-minute mock interview with a peer who acts as the interviewer
- [ ] The interviewer should ask at least one extension question and one "how would you test this?" question
- [ ] Have the interviewer score you on the rubric while you are designing, not after
- [ ] Debrief for 15 minutes: which decisions did the interviewer question, and were your answers satisfactory?

**Day 12 — Second Lowest-Scoring Dimension + Behavioral Practice (2 hours)**

- [ ] Address the second-lowest rubric dimension from Day 7 using the action table in Phase 4
- [ ] Study `10-interview-preparation/behavioral/design-decision-questions.md`
- [ ] Practice answering 5 design decision justification questions aloud — "Why did you use composition here instead of inheritance?" — without stopping to think more than 10 seconds

**Day 13 — Final Mock Interview (1.5 hours)**

- [ ] Select a medium or hard problem from `10-interview-preparation/assignments/` that you have not seen before
- [ ] Conduct under exact interview conditions: no notes, no reference material, timer set, speaking aloud throughout
- [ ] Score yourself immediately after
- [ ] If any dimension scores below 3, do one final targeted review of the relevant module

**Day 14 — Light Review and Preparation (1 hour)**

- [ ] Review cheat sheets in `11-artifacts/cheat-sheets/` — do not study new material the day before the interview
- [ ] Review pattern quick reference in `11-artifacts/` — refresh, do not relearn
- [ ] Reread the opening section of `05-senior-engineer-design-thinking/README.md` — the mid-level to senior mental model shift is the right frame to carry into the interview

---

### Mandatory Pre-Interview Checks

Complete these within 48 hours of your interview, regardless of how much preparation time you had.

- [ ] Can you state all 5 SOLID principles from memory in under 60 seconds?
- [ ] Can you draw a class diagram with 6 classes and correct notation in under 8 minutes?
- [ ] Can you sketch the Strategy pattern structure in under 2 minutes without reference?
- [ ] Can you sketch the Observer pattern structure in under 2 minutes without reference?
- [ ] Can you name and describe 3 patterns that satisfy OCP and explain how each does so?
- [ ] Can you describe the difference between an LLD interview and a System Design interview in 3 sentences?
- [ ] Do you know what you will say in the first 60 seconds of the interview after the problem is presented?

If any of these checks fail, spend one focused hour on that item only. Do not spread the last 48 hours thin.

---

### What Not to Do in the Final 2 Days

- Do not study new patterns you have not practiced before. Unfamiliar patterns under pressure produce worse output than no pattern at all.
- Do not work through new case studies. You will not have time to internalize the design before the interview.
- Do not over-prepare on topics you are already strong in at the cost of not resting. Cognitive performance in a timed interview is significantly affected by sleep.
- Do not memorize code. Interviewers detect memorized code immediately — it is too clean, too fast, and disconnected from your own reasoning. Code that you derive in the interview, even if slightly imperfect, is far more credible.

---

*The interview is not a test of what you know. It is a test of what you can do, under pressure, while explaining why. Those are practiced skills, not knowledge states. The work in this module is the practice.*
