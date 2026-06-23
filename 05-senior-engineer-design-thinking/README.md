# Module 05 — Senior Engineer Design Thinking

> Knowing patterns is table stakes. Knowing which problem you are actually solving — and what you are trading away to solve it — is what separates a senior engineer from a good mid-level one.

---

## Introduction

Most mid-level engineers can implement a feature. A strong mid-level engineer can do it cleanly, using the right data structures and following team conventions. What changes at the senior level is not the ability to write code — it is the ability to make decisions that age well.

The gap between mid-level and senior is not primarily about technical knowledge. It is about a shift in what questions you ask before you write a single line.

A mid-level engineer asks: "How do I implement this?"

A senior engineer asks: "Why does this need to exist? What changes in six months? Who else touches this code? What does a caller need to know to use this safely? What breaks if I get this wrong?"

This distinction shows up in concrete, observable habits:

**A mid-level engineer abstracts when they see repetition. A senior engineer abstracts when they anticipate divergence.** The mid-level engineer reaches for an interface because two things look similar today. The senior engineer reaches for an interface because they have reasoned that those two things will need to change independently tomorrow. This is not a philosophical difference — it produces fundamentally different class hierarchies.

**A mid-level engineer designs for the feature. A senior engineer designs for the feature's successor.** The question is never "does this work for the current requirement?" The question is "does this survive the next three requirements?" OCP (Open/Closed Principle) is not just a rule to follow — it is a prediction about where change will enter the system.

**A mid-level engineer treats error handling as an afterthought. A senior engineer treats the failure path as the first path.** Every external dependency, every I/O operation, every user input is a source of failure. Senior engineers design the contract for failure before they design the happy path.

**A mid-level engineer produces a solution. A senior engineer documents a decision.** Code is not self-documenting. The what is in the code. The why is in the structure, the comments, the PR description, and sometimes the naming. Code that cannot explain its own decisions will be modified incorrectly.

**A mid-level engineer evaluates designs by their structure. A senior engineer evaluates designs by the cost of the decisions they lock in.** Choosing an abstraction, a data model, or a module boundary is not free — it constrains future choices. Senior engineers price those constraints before committing.

This module is about internalizing that shift. Each file targets a specific dimension of senior-level design thinking, translating it from principle into practice.

---

## The Senior Engineer Mental Model

Before engaging with the individual files, hold this overarching frame in mind. It is the lens through which every topic in this module should be read.

### Designing for the Engineer Two Years From Now

Every design decision you make today will be read, modified, and debugged by someone — possibly yourself — with no memory of the original context. The primary constraint on good design is not "does this solve the problem?" but "can a competent engineer, unfamiliar with my reasoning, work safely with this code?"

This reframes many decisions. A slightly more verbose API that makes invalid usage a compile error is better than a concise one that fails at runtime. A class with an obvious single responsibility is better than one with a clever, ambiguous abstraction. Naming that encodes intent is better than naming that encodes implementation.

### Designing for Change

Requirements change. Business priorities shift. The domain you understood six months ago will be reframed. A design that cannot accommodate change without structural surgery is not a complete design — it is a liability.

Designing for change does not mean speculative generalization. It means identifying the dimensions along which your domain is likely to evolve and making sure change along those dimensions is cheap. It means separating the stable core from the volatile periphery. It means making the boundaries between modules explicit so that change can be localized.

### The Cost of Decisions

Every design choice has a cost. Choosing inheritance over composition makes certain extensions easy and others painful. Introducing an abstraction adds indirection that must be navigated by every future reader. Coupling two modules makes them change together. Choosing a particular data representation constrains every algorithm that will touch it.

Senior engineers do not try to make every decision costless — they make the cost visible and verify that the benefit justifies it. This is not analysis paralysis; it is professional rigor.

### Tradeoffs Over Rules

SOLID principles, design patterns, and the other tools you have studied in this course are not rules. They are tradeoffs dressed as rules. Every principle is an argument that one cost is generally worth paying. But "generally" is not "always."

A senior engineer's job is to understand the argument behind the rule well enough to know when the rule does not apply. Adding an abstraction layer that has exactly one current implementation, no anticipated second implementation, and a team that will be confused by the indirection is a violation of good engineering — even if it satisfies OCP on paper.

The vocabulary of principles matters because it gives you a shared language for discussing tradeoffs with precision. But the destination is judgment, not compliance.

---

## Prerequisites

This module assumes you have completed or are fluent in the content of Modules 00 through 04:

- **Module 00 — Foundation**: Classes, interfaces, abstract classes, composition vs. inheritance, information hiding, object modeling from requirements
- **Module 01 — SOLID Principles**: All five principles at the level of violation detection and before/after refactoring — not just definitional recall
- **Module 02 — Design Principles**: DRY, KISS, YAGNI, cohesion, coupling, Law of Demeter, Dependency Injection
- **Module 03 — UML Modeling**: Class diagrams (reading and drawing), sequence diagrams, enough to represent designs visually under time pressure
- **Module 04 — Design Patterns**: Creational, Structural, and Behavioral patterns — when to apply them, not just how to implement them

If you skipped any of those modules, you can still read this one — but you will find the discussions of specific patterns and principles assume fluency, not just familiarity.

Additionally, this module assumes you have some professional software engineering experience. The examples and mental models here are grounded in what happens on real engineering teams. If you have never maintained production code through a requirement change, some of this will be abstract until you have that experience.

---

## What You Will Learn

The six files in this module cover six distinct dimensions of senior-level design thinking. Below is a concrete breakdown of what each file delivers.

### 1. Extensibility and Scalability at the Code Level

- How to identify the dimensions of change in a domain and encode that identification into your class hierarchy
- The distinction between accidental coupling (two things that share code for convenience) and intentional coupling (two things that must change together)
- Practical techniques: plugin points via Strategy, extension via Decorator, variation isolation via Abstract Factory
- When to defer an abstraction vs. when to introduce it upfront — the two-implementation rule and its limits
- How to use package and module boundaries to enforce extensibility contracts without runtime overhead
- Code-level scalability patterns: immutable value objects, stateless service objects, separation of computation from I/O

### 2. API Design for Senior Engineers

- The contract model of API design: what a caller is promised and what they are not
- Designing APIs that fail loudly and early rather than silently and late — the principle of making illegal states unrepresentable
- Fluent interfaces and builder patterns for complex configuration — when they help and when they hide complexity
- Versioning strategy at the API level: backward-compatible evolution, deprecation patterns, client shielding
- The difference between an API designed for the implementer's convenience and one designed for the caller's safety
- Command/Query Separation (CQS) as an API design principle
- Common API design mistakes senior engineers reject during code review: side effects on getters, mutation through queries, boolean flag parameters, output parameters, exceptions as control flow

### 3. Domain Modeling

- How to extract the domain model from requirements: identifying entities, value objects, aggregates, and domain services
- The distinction between an entity (identity matters) and a value object (content matters) — and why conflating them causes bugs
- Aggregate boundaries: what belongs inside an aggregate, what belongs outside, and how to enforce the boundary
- Anemic vs. rich domain models: when logic belongs in the domain object vs. in a service
- Ubiquitous language: why the names in your code should match the names in the business domain, and what happens when they do not
- Domain events as a modeling tool for state transitions that the rest of the system cares about
- Practical domain modeling exercises on realistic problem statements (e-commerce, booking, payments)

### 4. Error Handling and Resilience Patterns

- The two categories of errors: errors the caller can handle and errors the caller cannot — and why mixing them is a design failure
- Checked vs. unchecked exceptions: a design decision, not a style preference
- Result types and the Either pattern as an alternative to exception-based error signaling
- Designing error contracts: what error information a caller legitimately needs, and what is implementation detail
- Resilience patterns at the code level: retry with backoff, circuit breaker, bulkhead, timeout — implementation and when each applies
- Idempotency as a design property: how to make operations safe to retry
- Defensive programming vs. fail-fast: when to validate eagerly and when to trust your collaborators
- Error propagation strategy: how errors should flow across layer boundaries

### 5. Testing Strategy and Testability

- Designing code that is easy to test before the tests exist — testability as a design quality, not a testing concern
- The testing pyramid in the context of object-oriented design: what each layer should verify and what it should not
- Dependency injection as a seam mechanism: making collaborators replaceable in tests without production overhead
- When to use mocks vs. stubs vs. fakes vs. real implementations — the costs and signals of each choice
- Test naming as specification: how a test's name should describe a behavior, not an implementation
- Testing domain logic in isolation: the value of keeping domain objects free of framework and infrastructure dependencies
- Code smells that signal testability problems: static coupling, hidden dependencies, constructors with logic, global state
- Property-based testing as a tool for finding edge cases in domain invariants

### 6. Technical Debt and Refactoring Strategy

- A precise definition of technical debt: the deliberate deferral of a better design with the intent to repay it, vs. accumulated carelessness
- How to identify technical debt: coupling metrics, change failure rate, cognitive load signals
- The strangler fig pattern and other incremental refactoring strategies for legacy code
- How to build a business case for refactoring: framing debt in terms of delivery velocity and defect rate, not code aesthetics
- Refactoring vs. rewriting: when each is appropriate and how to evaluate the risk
- The refactoring toolkit: extract method/class, introduce parameter object, replace conditional with polymorphism, push logic to domain objects
- Making refactoring safe: characterization tests, seam identification, strangling the old design before removing it
- How senior engineers advocate for refactoring without stalling delivery

---

## Module Structure

| File | Topic | Estimated Reading Time |
|---|---|---|
| `01-extensibility-and-scalability.md` | Extensibility and Scalability at the Code Level | 35-40 minutes |
| `02-api-design.md` | API Design for Senior Engineers | 30-35 minutes |
| `03-domain-modeling.md` | Domain Modeling | 40-45 minutes |
| `04-error-handling-and-resilience.md` | Error Handling and Resilience Patterns | 30-35 minutes |
| `05-testing-strategy-and-testability.md` | Testing Strategy and Testability | 30-35 minutes |
| `06-technical-debt-and-refactoring.md` | Technical Debt and Refactoring Strategy | 25-30 minutes |

Total estimated reading time: approximately 3 to 3.5 hours. This does not include time for exercises, which add another 2 to 3 hours depending on how deeply you engage with them.

---

## How to Use This Module

### Suggested Reading Order

The files are ordered intentionally. Read them in sequence on the first pass.

1. Start with **Extensibility and Scalability** because it establishes the foundational concern: designing for the next change. This frames everything else.
2. **API Design** follows naturally — once you understand what you are designing for, you can design the interface through which others interact with it.
3. **Domain Modeling** is the heart of object-oriented design at the senior level. It is placed third because it builds on both the extensibility mindset and the API contract mindset.
4. **Error Handling and Resilience** is a domain that most engineers under-invest in. Reading it after domain modeling is deliberate — a rich domain model makes error contracts much clearer to reason about.
5. **Testing Strategy** comes fifth because testability is a consequence of good design decisions, not a parallel concern. By this point you should be able to evaluate your designs against a testing lens.
6. **Technical Debt and Refactoring** is last because it presupposes familiarity with all the prior concepts — you cannot identify or fix debt without knowing what "better" looks like.

On subsequent passes, treat each file independently and return to specific sections when a real engineering problem makes them relevant. Senior design thinking is absorbed through application, not memorization.

### How to Approach the Exercises

Each file contains exercises ranging from analysis tasks (evaluate this design and identify problems) to design tasks (design a solution from a given requirement) to refactoring tasks (take this code and improve it).

Do not skip the exercises. Reading about design thinking and practicing design thinking are different activities. The exercises are calibrated to the level of a real senior engineering interview or design review.

For analysis exercises: write your analysis before reading the provided analysis. The comparison between what you noticed and what was identified is the learning.

For design exercises: spend at least 15 minutes on a problem before looking at the reference design. Your first attempt will surface your current instincts. The gap between your design and the reference is where the learning lives.

For refactoring exercises: make the changes in a real editor if possible. Refactoring is a physical skill as much as a cognitive one, and the friction of actually moving code around builds the muscle memory that matters in a machine coding round.

### Pairing With Other Modules

This module is most effective when paired with the advanced case studies in Module 06. After reading a file in this module, look for the corresponding dimension in the case study designs:

- After `01-extensibility-and-scalability.md`: examine how the Notification Framework and Workflow Engine case studies handle extensibility
- After `02-api-design.md`: examine the Payment Gateway API contracts
- After `03-domain-modeling.md`: examine the Ride-Sharing and Splitwise domain models
- After `04-error-handling-and-resilience.md`: examine the Payment Gateway and ATM error paths
- After `05-testing-strategy-and-testability.md`: read the test suites accompanying the Medium case studies and evaluate their design
- After `06-technical-debt-and-refactoring.md`: attempt the extension problems in the Easy and Medium case studies as refactoring exercises on designs you did not write

---

## A Note on Scope

This module deliberately does not cover distributed systems, high-availability infrastructure, database design, or capacity planning. Those belong to the high-level system design domain.

This module is about code-level design: the decisions you make within a service, a library, or a bounded component that determine whether the code can be safely understood, safely extended, and safely changed over time. That scope is distinct from infrastructure-level decisions, and it is the scope that determines whether you write production-quality code vs. code that merely runs.

Senior engineers at top-tier companies are expected to be fluent in both domains. This course builds the code-level half. The distributed systems half requires a separate curriculum.

---

*When you are ready, begin with [`01-extensibility-and-scalability.md`](./01-extensibility-and-scalability.md).*
