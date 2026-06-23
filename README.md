# Low-Level Design (LLD) & Object-Oriented Design Mastery

> **Design systems the way senior engineers think — not just code, but architecture from first principles.**

---

## What This Course Is

This is a structured, interview-grade curriculum for software engineers preparing for Senior Engineer roles at product companies (FAANG, unicorns, and mid-stage startups). It bridges the gap between "knowing OOP" and "designing production systems confidently" — the gap that causes most mid-level engineers to stumble at Senior interviews.

The course is not a collection of pattern definitions. It is a thinking framework. You will learn how to approach a blank whiteboard, ask the right clarifying questions, translate requirements into class hierarchies, apply patterns where they earn their complexity, and defend every design decision under pressure.

---

## Value Proposition

| Without This Course | With This Course |
|---|---|
| You know design patterns by name | You know *when* and *why* to apply each one |
| You can code but struggle to design | You decompose problems into clean class hierarchies in minutes |
| You memorize solutions | You derive solutions from first principles |
| You freeze at "design X from scratch" | You have a repeatable framework for any design problem |
| You write code that works | You write code that is extensible, testable, and maintainable |

---

## What You Will Learn

- **Object-Oriented Design Mastery**: Classes, interfaces, abstract classes, composition vs. inheritance — with production-level judgment on when to use what
- **SOLID Principles in Depth**: Not just definitions, but violation detection, refactoring techniques, and real-world tradeoffs
- **23 Gang of Four Design Patterns**: Creational, Structural, and Behavioral — each with canonical implementations, real-world use cases, and anti-pattern warnings
- **UML Fluency**: Class diagrams, sequence diagrams, activity diagrams — draw and read them at interview speed
- **Senior Engineer Design Thinking**: Requirement decomposition, extensibility planning, abstraction selection, change isolation strategies
- **17 Real-World Case Studies**: Parking Lot to Ride-Sharing — full end-to-end designs with class hierarchies, sequence diagrams, and decision logs
- **Machine Coding Round Preparation**: Timed practice, evaluation rubrics, and annotated solutions
- **Testing & Quality Engineering**: Unit testing strategy, testability design, mocking patterns
- **Capstone Projects**: Portfolio-ready, full-system designs demonstrating mastery
- **Interview Preparation**: Question banks, mock sessions, common traps, and evaluation criteria decoded

---

## Who This Is For

**Primary audience**: Software engineers with 2-8 years of experience preparing for:
- Senior Software Engineer interviews at product companies
- Promotions requiring design-level contributions
- Technical lead roles requiring system design ownership

**You will benefit if**:
- You can code fluently but feel shaky when asked to "design X from scratch"
- You've read about design patterns but don't know when to reach for them
- You pass coding rounds but struggle in the design round
- You want to write code that senior engineers actually respect during code review

**You may not need this if**:
- You are a complete programming beginner (this course assumes OOP literacy)
- You are preparing exclusively for system design (HLD/distributed systems) — though this course complements that preparation

**Prerequisites**: See the Prerequisites section below.

---

## Prerequisites

Before starting this course, you should be comfortable with:

- **One OOP language**: Java, Python, C++, or TypeScript. Code examples use Java; the principles are language-agnostic.
- **Basic OOP syntax**: You can write a class, define methods, create objects, and use inheritance at a syntactic level.
- **Basic data structures**: Arrays, lists, maps, sets, stacks, queues.
- **Coding problem-solving**: You can solve medium-difficulty coding problems. This is not a DSA course.

You do **not** need prior design pattern knowledge, UML experience, or previous exposure to SOLID principles. Those are taught from the ground up.

---

## Complete Table of Contents

### Module 00 — Foundation: OOP & Clean Code Fundamentals
> [`00-foundation/`](./00-foundation/)

| Topic | Description |
|---|---|
| OOP Pillars Deep Dive | Encapsulation, Abstraction, Inheritance, Polymorphism — beyond syntax |
| Classes vs Interfaces vs Abstract Classes | Decision framework for choosing the right construct |
| Composition vs Inheritance | When each is appropriate; the "favor composition" rule explained |
| Access Modifiers & Information Hiding | Designing for minimal exposure |
| Clean Code Fundamentals | Naming, function design, comment philosophy, code organization |
| Immutability & Value Objects | Designing for thread safety and predictability |
| Object Modeling from Requirements | Going from a requirements sentence to a class hierarchy |

---

### Module 01 — SOLID Principles
> [`01-solid-principles/`](./01-solid-principles/)

| Principle | Focus |
|---|---|
| Single Responsibility Principle | One reason to change; identifying responsibility boundaries |
| Open/Closed Principle | Extension without modification; plugin architectures |
| Liskov Substitution Principle | Behavioral subtyping; contract design |
| Interface Segregation Principle | Fat interfaces; role interfaces |
| Dependency Inversion Principle | Abstractions over concretions; inversion of control |

Includes: violation detection exercises, before/after refactoring examples, combined violations.

---

### Module 02 — Design Principles
> [`02-design-principles/`](./02-design-principles/)

| Principle | Focus |
|---|---|
| DRY (Don't Repeat Yourself) | Knowledge duplication vs. code duplication |
| KISS (Keep It Simple, Stupid) | Simplicity as an engineering value |
| YAGNI (You Aren't Gonna Need It) | Avoiding speculative generality |
| High Cohesion | Keeping related things together |
| Low Coupling | Reducing inter-component dependencies |
| Law of Demeter | "Don't talk to strangers" |
| Dependency Injection | Constructor, method, and property injection |
| Tell, Don't Ask | Behavior assignment philosophy |

---

### Module 03 — UML & Design Modeling
> [`03-uml-modeling/`](./03-uml-modeling/)

| Topic | Focus |
|---|---|
| Class Diagrams | Classes, relationships, multiplicity, visibility |
| Sequence Diagrams | Object interaction over time |
| Activity Diagrams | Business process flows |
| Use Case Diagrams | Actor-system interactions |
| State Machine Diagrams | Object lifecycle modeling |
| UML for Interviews | Speed-drawing technique, shorthand notation |

---

### Module 04 — Design Patterns
> [`04-design-patterns/`](./04-design-patterns/)

#### Creational Patterns — [`04-design-patterns/creational/`](./04-design-patterns/creational/)

| Pattern | Core Idea |
|---|---|
| Singleton | Controlled single instance |
| Factory Method | Subclass decides instantiation |
| Abstract Factory | Families of related objects |
| Builder | Step-by-step complex object construction |
| Prototype | Clone-based instantiation |

#### Structural Patterns — [`04-design-patterns/structural/`](./04-design-patterns/structural/)

| Pattern | Core Idea |
|---|---|
| Adapter | Interface translation |
| Decorator | Dynamic behavior addition |
| Facade | Simplified interface to subsystem |
| Composite | Tree structures of objects |
| Proxy | Controlled object access |
| Bridge | Abstraction/implementation decoupling |
| Flyweight | Shared fine-grained objects |

#### Behavioral Patterns — [`04-design-patterns/behavioral/`](./04-design-patterns/behavioral/)

| Pattern | Core Idea |
|---|---|
| Strategy | Interchangeable algorithms |
| Observer | Event notification |
| Command | Encapsulated requests |
| Chain of Responsibility | Request handler pipeline |
| State | Behavior by state |
| Template Method | Skeleton with overridable steps |
| Iterator | Sequential collection traversal |
| Mediator | Centralized component coordination |
| Visitor | Operations over object structures |
| Memento | State snapshot and restore |

---

### Module 05 — Senior Engineer Design Thinking
> [`05-senior-engineer-design-thinking/`](./05-senior-engineer-design-thinking/)

| Topic | Focus |
|---|---|
| Requirement Decomposition | Extracting nouns, verbs, constraints from requirements |
| Extensibility Planning | Designing for the next feature, not the current one |
| Abstraction Selection | When to abstract, when not to |
| Change Isolation | Protecting stable code from volatile code |
| Design Tradeoffs | Flexibility vs. simplicity, inheritance vs. composition |
| API Design Philosophy | Designing interfaces that clients want to use |
| The Design Review Mindset | How senior engineers evaluate designs |

---

### Module 06 — Real-World Case Studies
> [`06-case-studies/`](./06-case-studies/)

#### Easy — [`06-case-studies/easy/`](./06-case-studies/easy/)
- [Parking Lot System](./06-case-studies/easy/01-parking-lot/)
- [Library Management System](./06-case-studies/easy/02-library-management/)
- [Vending Machine](./06-case-studies/easy/03-vending-machine/)

#### Medium — [`06-case-studies/medium/`](./06-case-studies/medium/)
- [ATM System](./06-case-studies/medium/01-atm-system/)
- [Hotel Booking System](./06-case-studies/medium/02-hotel-booking/)
- [Food Delivery System](./06-case-studies/medium/03-food-delivery/)
- [Movie Ticket Booking](./06-case-studies/medium/04-movie-ticket-booking/)
- [Inventory Management System](./06-case-studies/medium/05-inventory-management/)

#### Advanced — [`06-case-studies/advanced/`](./06-case-studies/advanced/)
- [Ride-Sharing System (Uber/Ola)](./06-case-studies/advanced/01-ride-sharing/)
- [E-commerce Platform](./06-case-studies/advanced/02-ecommerce-platform/)
- [Splitwise / Expense Sharing](./06-case-studies/advanced/03-splitwise/)
- [Notification Framework](./06-case-studies/advanced/04-notification-framework/)
- [Payment Gateway](./06-case-studies/advanced/05-payment-gateway/)
- [Rate Limiter](./06-case-studies/advanced/06-rate-limiter/)
- [Logging Framework](./06-case-studies/advanced/07-logging-framework/)
- [Cache System](./06-case-studies/advanced/08-cache-system/)
- [Workflow Engine](./06-case-studies/advanced/09-workflow-engine/)

---

### Module 07 — Machine Coding Round Preparation
> [`07-machine-coding/`](./07-machine-coding/)

Timed, realistic machine coding problems with:
- Problem statements (as given in real interviews)
- Evaluation rubrics
- Annotated reference solutions
- Common mistakes and traps

---

### Module 08 — Testing & Quality Engineering
> [`08-testing-quality/`](./08-testing-quality/)

| Topic | Focus |
|---|---|
| Designing for Testability | DI, interfaces, and seams |
| Unit Testing Strategy | Arrange-Act-Assert, test naming, coverage philosophy |
| Mocking & Stubbing | When to mock, what to verify |
| Test-Driven Development | Red-Green-Refactor applied to design problems |
| Code Quality Metrics | Cyclomatic complexity, coupling metrics |

---

### Module 09 — Capstone Projects
> [`09-capstone-projects/`](./09-capstone-projects/)

Three portfolio-ready projects requiring full end-to-end design:
- Full class diagram with all relationships
- Sequence diagrams for key workflows
- Decision log explaining design choices
- Working implementation with tests
- Extensibility discussion

---

### Module 10 — Interview Preparation
> [`10-interview-preparation/`](./10-interview-preparation/)

| Resource | Description |
|---|---|
| Question Bank | 100+ LLD interview questions with difficulty ratings |
| Mock Interviews | [`10-interview-preparation/mock-interviews/`](./10-interview-preparation/mock-interviews/) |
| Common Traps | Anti-patterns interviewers look for |
| Evaluation Criteria Decoded | What interviewers actually score |
| Behavioral Questions | Design decision justification practice |

---

### Module 11 — Artifacts & Reference Materials
> [`11-artifacts/`](./11-artifacts/)

| Resource | Location |
|---|---|
| Cheat Sheets | [`11-artifacts/cheat-sheets/`](./11-artifacts/cheat-sheets/) |
| Flashcards | [`11-artifacts/flashcards/`](./11-artifacts/flashcards/) |
| UML Templates | [`11-artifacts/uml-templates/`](./11-artifacts/uml-templates/) |
| Pattern Quick Reference | Pattern → problem mapping |
| Interview Formula Cards | Per-problem design templates |

---

## How to Use This Course

### Option A: Structured Learning (Recommended for depth)
Follow the 12-week plan in [`ROADMAP.md`](./ROADMAP.md). Work module by module, complete all assignments before advancing, and do the capstone projects at the end.

### Option B: Interview-Accelerated (Recommended with < 4 weeks)
Follow the 4-week crash plan in [`ROADMAP.md`](./ROADMAP.md#4-week-intensive). Skip Module 03 (UML) on the first pass, go deep on Modules 01, 02, 04, 06, and 07. Return to gaps after your interview cycle.

### Option C: Reference Mode
If you already have design fundamentals, use this as a structured reference. Navigate to specific modules or case studies as needed. Use [`11-artifacts/cheat-sheets/`](./11-artifacts/cheat-sheets/) for quick review.

### Working Through Case Studies
Each case study follows this format:
1. **Problem statement** — Read it. Stop. Spend 10 minutes designing before looking further.
2. **Requirements extraction** — Compare your extracted requirements with the provided list.
3. **Class hierarchy** — Compare your design.
4. **Sequence diagrams** — Trace through the flows.
5. **Decision log** — Read the tradeoff rationale.
6. **Implementation** — Read and run the code.
7. **Extensions** — Try the extension problems independently.

---

## Quick Start: 4-Week Interview Crash Plan

For engineers with a Senior interview in under a month. This is the high-signal, minimum-sufficient path.

```
Week 1: Foundations + SOLID
  Mon-Tue  → 00-foundation (OOP pillars, composition vs inheritance)
  Wed-Thu  → 01-solid-principles (all 5 principles + violation exercises)
  Fri      → 02-design-principles (DRY, KISS, coupling/cohesion, DI)
  Weekend  → Easy case studies: Parking Lot, Library, Vending Machine

Week 2: Patterns (High-Priority Subset)
  Mon      → Strategy, Observer, Command (most interview-common behavioral)
  Tue      → Factory Method, Builder, Singleton (creational)
  Wed      → Adapter, Decorator, Facade, Proxy (structural)
  Thu      → Chain of Responsibility, State, Template Method
  Fri      → Pattern recognition drill — identify patterns in 10 code snippets
  Weekend  → Medium case studies: ATM, Hotel Booking, Movie Ticket

Week 3: Advanced Case Studies + Machine Coding
  Mon-Tue  → Ride-Sharing, Notification Framework, Payment Gateway
  Wed      → Rate Limiter, Cache System
  Thu      → Machine coding timed practice x2
  Fri      → Machine coding timed practice x2 + review
  Weekend  → Splitwise, Logging Framework

Week 4: Mock Interviews + Gap Closure
  Mon-Tue  → Mock interview sessions (use 10-interview-preparation)
  Wed      → Weak area deep dive based on mock feedback
  Thu      → UML speed practice (class diagrams in < 10 minutes)
  Fri      → Final mock + cheat sheet review
  Weekend  → Rest + light review of cheat sheets
```

---

## Deep Dive: 12-Week Full Learning Plan

For engineers investing in long-term design mastery. Full details in [`ROADMAP.md`](./ROADMAP.md).

| Phase | Weeks | Focus |
|---|---|---|
| Foundation | 1-2 | OOP mastery + Clean Code |
| Principles | 3-4 | SOLID + Design Principles + UML |
| Patterns | 5-7 | All 23 GoF patterns with implementations |
| Application | 8-10 | Senior thinking + all case studies |
| Mastery | 11-12 | Machine coding + Testing + Capstone |

---

## Milestone & Badge System

Progress through the course earns the following milestones:

| Badge | Earned By | Signifies |
|---|---|---|
| **Foundation Badge** | Complete Modules 00-02 + assignments | OOP and principles fluency |
| **Pattern Navigator** | Complete Module 04 (all 3 sub-modules) | Pattern recognition and application |
| **UML Fluent** | Complete Module 03 + draw 5 diagrams unaided | Diagram literacy |
| **Case Study Practitioner** | Complete all Easy + Medium case studies | Applied design capability |
| **Design Architect** | Complete all Advanced case studies | Production-level design ability |
| **Machine Coder** | Complete 5 timed machine coding problems under 90 min each | Interview execution speed |
| **Senior Designator** | Complete all modules + Capstone | Full course completion |

**Course Completion** = Foundation Badge + Pattern Navigator + Design Architect + Machine Coder + Capstone submitted.

See [`CURRICULUM.md`](./CURRICULUM.md) for certification criteria and rubrics.

---

## Certification Criteria

To consider this course "complete" (not just sampled), you must:

1. Complete all assignments in Modules 00-05 with passing rubric scores
2. Independently design and implement at least 5 case studies (including 2 advanced)
3. Complete 3 timed machine coding problems in under 90 minutes with clean output
4. Submit one Capstone Project with full artifacts (class diagram + sequence diagram + decision log + working code + tests)
5. Score yourself 3+ on at least 40 of 50 items in the Interview Readiness Checklist in [`ROADMAP.md`](./ROADMAP.md)

See [`CURRICULUM.md`](./CURRICULUM.md) for the full rubric and portfolio requirements.

---

## Repository Structure

```
SystemDesignLLDCrashCourse/
├── README.md                          ← You are here
├── CURRICULUM.md                      ← Full syllabus and learning objectives
├── ROADMAP.md                         ← Week-by-week learning plans
├── 00-foundation/                     ← OOP & Clean Code
├── 01-solid-principles/               ← SOLID with assignments
│   └── assignments/
├── 02-design-principles/              ← DRY, KISS, Coupling, DI
│   └── assignments/
├── 03-uml-modeling/                   ← UML with exercises
│   └── exercises/
├── 04-design-patterns/
│   ├── creational/
│   ├── structural/
│   └── behavioral/
├── 05-senior-engineer-design-thinking/
├── 06-case-studies/
│   ├── easy/
│   ├── medium/
│   └── advanced/
├── 07-machine-coding/
│   ├── questions/
│   └── solutions/
├── 08-testing-quality/
├── 09-capstone-projects/
├── 10-interview-preparation/
│   └── mock-interviews/
└── 11-artifacts/
    ├── cheat-sheets/
    ├── flashcards/
    └── uml-templates/
```

---

## A Note on Language

Code examples throughout this course are written in **Java**. The concepts are language-agnostic. If you work in Python, TypeScript, C++, or Go, every principle and pattern applies — only the syntax differs. The important thing is that you understand *why* a design works, not just how to transcribe it.

---

*Start with [`ROADMAP.md`](./ROADMAP.md) to pick your learning path, or dive directly into [`00-foundation/`](./00-foundation/) if you're ready to begin.*

---

## Connect with Me

If this series helps you ace an interview, ship a better design, or just survive a 3 AM on-call — I'd love to hear about it. Follow along, share your progress, and tag me when you land that offer.

| Platform | Handle | Why follow |
|----------|--------|------------|
| 💼 LinkedIn | [Sunchit Dudeja](https://www.linkedin.com/in/sunchitdudeja/) | Long-form posts, career insights, and series announcements |
| 📺 YouTube | [@CodeWithSunchitDudeja](https://www.youtube.com/@CodeWithSunchitDudeja) | Video walkthroughs of every post in this series |
| 📸 Instagram | [@sunchitdudeja](https://www.instagram.com/sunchitdudeja/) | Daily system design snippets and behind-the-scenes |
| 🐙 GitHub | [@Sunchit](https://github.com/Sunchit) | All the source code, diagrams, and companion series |
| 🎯 Topmate | [sunchit_dudeja](https://topmate.io/sunchit_dudeja/) | 1:1 mentorship, mock interviews, and career guidance |

---

## About the Author

**Sunchit Dudeja** — Engineering Leader · System Design Educator

Building this series to give every developer the foundation that took most architects years of painful incidents to develop.

Also writing the companion series **Learn to Think Like an Architect** — mental models and real-world architectural war stories.

Happy Learning! 🎉

> *"A developer makes the code work. An architect makes the system last."*
