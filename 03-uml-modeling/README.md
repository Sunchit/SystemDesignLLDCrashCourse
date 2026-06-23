# Module 03: UML Modeling

> **Target Audience:** Software engineers with 2-8 years of experience preparing for Senior Engineer roles.
> **Prerequisite:** Module 02 — Design Principles

---

## Why UML Matters for Senior Engineers

UML (Unified Modeling Language) is the shared vocabulary of software design. When a senior engineer sits down with a whiteboard — in a design review, an architecture meeting, or an interview — they communicate using diagrams. Engineers who cannot draw diagrams clearly lose the ability to communicate at the level their role demands.

UML is not about producing perfect formal specifications. It is about communicating design intent efficiently. In an interview, a clean, hand-drawn class diagram drawn in 5 minutes shows more design competence than 30 minutes of verbal explanation.

---

## What UML Is (and Is Not)

**UML is:**
- A standardized notation for communicating software design
- A tool for thinking through design before writing code
- A lingua franca for technical discussions across teams
- A lightweight way to document key design decisions

**UML is not:**
- A complete formal specification language
- A replacement for code
- Something you must draw perfectly — approximate is usually sufficient
- Something that requires a tool — pen and paper (or whiteboard) is fine

---

## Module Structure

| File | Diagram Type | What You Will Learn |
|------|-------------|---------------------|
| `01-class-diagrams.md` | Class Diagrams | Static structure: classes, relationships, multiplicity |
| `02-sequence-diagrams.md` | Sequence Diagrams | Dynamic behavior: object interactions over time |
| `03-state-and-activity-diagrams.md` | State & Activity Diagrams | Lifecycle modeling and process flows |

---

## The Three Diagram Types You Actually Use

Of UML's 14 diagram types, three account for the vast majority of practical use:

### Class Diagrams — "What exists"
Show the static structure: classes, their attributes, methods, and the relationships between them. Used to communicate domain model design, API contracts, and architectural layers.

### Sequence Diagrams — "What happens"
Show how objects interact over time to fulfill a use case. Used to communicate call flows, API interactions, and protocol design.

### State Diagrams — "What an object goes through"
Show the lifecycle of an entity: what states it can be in, what events trigger transitions, what actions occur. Used for workflow modeling, protocol design, and order lifecycle.

---

## How to Draw UML in an Interview

In a technical interview (on a whiteboard or shared screen), the goal is clarity and speed — not perfection.

### For Class Diagrams
```
Use text notation:

+------------------+
|   ClassName      |   <- class name in top box
+------------------+
| - field: Type    |   <- attributes (- private, + public, # protected)
+------------------+
| + method(): Type |   <- methods
+------------------+

Relationships:
  -------->   Association
  -------<>   Aggregation
  -------<#>  Composition
  -------|>   Inheritance (open arrowhead)
  ------->    Dependency (dashed arrow)
  --------|>  Realization/implements (dashed + open arrowhead)
```

### For Sequence Diagrams
```
List actors/objects across the top.
Use vertical lines for lifelines.
Use horizontal arrows for messages.
Use numbered steps for clarity.
```

### For State Diagrams
```
[State Name] --> [Next State] : trigger / action
Use rectangles for states, arrows for transitions.
```

---

## Key Relationships at a Glance

| Relationship | Meaning | Java Equivalent |
|-------------|---------|-----------------|
| Association | A uses B | Field of type B |
| Aggregation | A has B (B can exist without A) | Field of type B, B created externally |
| Composition | A owns B (B cannot exist without A) | Field of type B, B created by A |
| Inheritance | A is-a B | `class A extends B` |
| Realization | A implements B | `class A implements B` |
| Dependency | A uses B temporarily | B appears in method parameter or local variable |

---

## Interview Readiness Checklist

By the end of this module, you should be able to:

- [ ] Draw a class diagram with correct notation for any LLD problem
- [ ] Show all relationship types (association, aggregation, composition, inheritance, realization)
- [ ] Draw a sequence diagram for a multi-step flow (login, ATM withdrawal, order placement)
- [ ] Draw a state diagram for an entity with a defined lifecycle (order, ATM session)
- [ ] Explain the difference between aggregation and composition verbally
- [ ] Quickly sketch any of these in under 5 minutes

---

## Common Diagram Anti-Patterns

**Too detailed:** Diagrams with every getter, setter, and private field. Class diagrams should show the design, not transcribe the code.

**No relationships:** A class diagram with isolated boxes shows nothing about design. Relationships are the point.

**Wrong arrow direction:** Association arrows point from the class that holds the reference to the class it references. Inheritance arrows point from child to parent.

**Mixing levels:** A class diagram that mixes infrastructure classes (JDBC, HttpClient) with domain classes. Keep levels consistent.

**Over-normalized:** Splitting trivially into 20 classes when 5 would communicate the design. Diagrams should communicate, not impress.

---

## How This Connects to the Rest of the Course

```
Module 01: SOLID              <- Design rules
Module 02: Design Principles  <- Design judgment
Module 03: UML Modeling       <- Design communication   (YOU ARE HERE)
Module 04: Design Patterns    <- Pattern recognition (all patterns shown with class diagrams)
Module 06: Case Studies       <- Full design walkthroughs using UML
Module 07: Machine Coding     <- Code from design
```

Every design pattern in Module 04 will be shown with a class diagram. Every case study in Module 06 will start with a class diagram and sequence diagram. The UML skills you build here are the foundation for everything that follows.
