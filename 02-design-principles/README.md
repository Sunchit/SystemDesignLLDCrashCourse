# Module 02: Design Principles

> **Target Audience:** Software engineers with 2-8 years of experience preparing for Senior Engineer roles.
> **Prerequisite:** Module 01 — SOLID Principles

---

## Why Design Principles Matter

SOLID gives you rules for classes. Design principles give you judgment for the spaces in between — the decisions SOLID doesn't cover and the instincts that distinguish a Senior Engineer from a Mid-Level one.

A senior engineer doesn't just write code that works. They write code that:
- Other engineers can read and modify without fear
- Can be tested in isolation
- Does not collapse under changing requirements
- Does not carry unnecessary weight from speculative features

This module builds that judgment systematically.

---

## Module Structure

| File | Topic | What You Will Learn |
|------|--------|---------------------|
| `01-dry-kiss-yagni.md` | DRY, KISS, YAGNI | The three foundational meta-principles that govern design decisions at every level |
| `02-composition-vs-inheritance.md` | Composition vs Inheritance | Why GoF said "favor composition," when inheritance is still right, and how to decide |
| `03-coupling-cohesion.md` | Coupling & Cohesion | The two forces that determine how well a system ages — measuring and optimizing them |
| `04-dependency-injection.md` | Dependency Injection | IoC, the three injection types, manual DI, Spring DI, testability |
| `05-code-smells-and-refactoring.md` | Code Smells & Refactoring | A complete catalog of what bad code looks like and the techniques to fix it |

---

## The Central Idea

Every principle in this module is a different lens on the same underlying truth:

> **Good design minimizes the cognitive load required to understand, change, and test the system.**

- **DRY** minimizes the cognitive load of tracking where truth lives
- **KISS** minimizes the cognitive load of understanding a solution
- **YAGNI** minimizes the cognitive load of working around unused abstractions
- **Composition over Inheritance** minimizes the cognitive load of predicting side effects from changes
- **Low Coupling** minimizes the cognitive load of understanding what a change affects
- **High Cohesion** minimizes the cognitive load of finding where a concept lives
- **DI** minimizes the cognitive load of wiring and testing components
- **Refactoring** is the practice of continuously reducing that cognitive load

---

## Key Concepts at a Glance

### DRY — Don't Repeat Yourself
Every piece of knowledge must have a **single, unambiguous, authoritative representation** in the system. Duplication of logic means two places to fix bugs and two places for them to diverge.

### KISS — Keep It Simple, Stupid
Prefer the simplest solution that correctly solves the problem. Simple is not the same as simplistic. Complex solutions are a liability.

### YAGNI — You Aren't Gonna Need It
Do not build what you do not need right now. Unused code has a maintenance cost with zero return. Extensibility should be earned, not speculated.

### Composition over Inheritance
Build flexible behavior by composing objects rather than by extending class hierarchies. Hierarchies are rigid; composition is malleable.

### Low Coupling, High Cohesion
Classes should know as little as possible about each other (low coupling). A class should have one clear, focused responsibility (high cohesion). These two forces together produce modular, maintainable systems.

### Dependency Injection
A class should not create its own dependencies. Dependencies should be handed in from outside. This enables testing, flexibility, and separation of construction from use.

### Code Smells
Symptoms that indicate deeper design problems. Not always bugs, but always technical debt. Recognizing smells is a prerequisite to refactoring.

### Refactoring
Improving the internal structure of code without changing observable behavior. A disciplined, incremental practice that keeps codebases healthy over time.

---

## How to Study This Module

1. **Read the concept** — understand the principle and why it exists
2. **Study the bad example** — recognize what violates it
3. **Study the good example** — see what following it looks like
4. **Answer the interview questions** — force yourself to articulate the reasoning
5. **Apply to the exercises** — build the muscle memory

---

## Interview Readiness

By the end of this module, you should be able to:

- Explain any of these principles clearly in 60 seconds without hesitation
- Give a concrete Java example for each one when asked
- Identify violations of these principles in code shown to you during a review round
- Make a principled argument for a design decision using these principles
- Know when to bend or break each principle and why

---

## How This Connects to the Rest of the Course

```
Module 01: SOLID          <- The rules for individual classes
Module 02: Design Principles  <- The judgment for systems of classes      (YOU ARE HERE)
Module 03: UML Modeling   <- The language for communicating design
Module 04: Design Patterns <- Proven solutions that embody these principles
Module 05: Design Thinking <- Applying all of the above in senior-level decisions
```

Design principles are the bridge between SOLID (class-level) and design patterns (system-level). You cannot use design patterns correctly without internalizing these principles first.

---

## Prerequisites Checklist

Before starting this module, confirm you can answer these without looking anything up:

- [ ] What is the Single Responsibility Principle?
- [ ] What problem does the Open/Closed Principle solve?
- [ ] What is the Liskov Substitution Principle in plain English?
- [ ] What distinguishes an interface from an abstract class in Java?
- [ ] What is a dependency in software?

If any of these are uncertain, revisit Module 01 first.
