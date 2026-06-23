# 11-Artifacts — Learning Artifacts Reference

This folder contains every reference artifact for the LLD course. These are not lecture notes — they are high-density, interview-ready references designed for review, active recall, and last-mile preparation. Use them alongside the weekly content, not instead of it.

---

## Artifact Inventory

| # | File | What It Is | Best Time to Use | Est. Read Time |
|---|------|-----------|-----------------|---------------|
| 1 | `cheat-sheets/solid-principles-cheat-sheet.md` | All 5 SOLID principles — definition, violation, fix, code example, interview one-liner | Week 1 review; night before interview | 8 min |
| 2 | `cheat-sheets/design-patterns-cheat-sheet.md` | All 18 GoF patterns — intent, structure, when to use, one-liner | Week 2 review; pattern deep-dive sessions | 15 min |
| 3 | `cheat-sheets/lld-interview-cheat-sheet.md` | Interview-day playbook — question types, approach template, traps to avoid | Morning of interview only | 10 min |
| 4 | `design-pattern-matrix.md` | Side-by-side comparison of all patterns across 8 dimensions | When two patterns feel interchangeable; thorough pattern study | 20 min |
| 5 | `flashcards/oop-flashcards.md` | 50 OOP flashcards covering encapsulation, polymorphism, abstraction, coupling | Daily active recall — start of each study session | 15–20 min |
| 6 | `flashcards/design-pattern-flashcards.md` | 54 pattern flashcards — intent, participants, tradeoffs | Daily active recall — after Week 2 begins | 15–20 min |
| 7 | `uml-templates/class-diagram-template.md` | UML notation reference, class diagram templates, relationship guide | Any time you practice drawing systems on paper or a whiteboard | 10 min |
| 8 | `REVISION-GUIDE.md` | Master pre-interview checklist with a structured 7-day countdown plan | One week before the interview; daily checkpoint | 12 min |

---

## How to Use This Folder

### Guiding principle
These artifacts are **output-side tools** — they consolidate what you have already learned. Do not open a cheat sheet to learn a concept for the first time. Learn it in the weekly content, then use these to compress and retain it.

### Suggested study sequence

```
Week 1 (OOP + SOLID)
  ├── Complete weekly lectures and exercises first
  ├── Day 5 review  →  solid-principles-cheat-sheet.md  (read + annotate)
  └── Days 5–7      →  oop-flashcards.md  (active recall, daily)

Week 2 (Design Patterns)
  ├── Complete weekly lectures and exercises first
  ├── Day 5 review  →  design-patterns-cheat-sheet.md  (read + annotate)
  ├── Day 6         →  design-pattern-matrix.md  (use when patterns blur together)
  └── Days 5–7      →  design-pattern-flashcards.md  (active recall, daily)

Any week (System Practice)
  └── Before each practice problem  →  uml-templates/class-diagram-template.md

Final 7 days before interview
  ├── Day 7         →  REVISION-GUIDE.md  (build your personal checklist)
  ├── Days 6–2      →  Flashcards + cheat sheets (daily rotation)
  └── Morning of    →  lld-interview-cheat-sheet.md  (once, then close it)
```

### Active recall rule
Flashcards only work if you **cover the answer before reading it**. For each card, say the answer aloud (or write it) before flipping. If you cannot answer without looking, mark it for a second pass.

### Cheat sheet annotation
Print or open the cheat sheets and **add your own examples** in the margins. Personal examples are 3x more memorable than the ones provided. The provided examples are a scaffold, not a ceiling.

---

## Artifact Descriptions

### `cheat-sheets/solid-principles-cheat-sheet.md`
A one-page-equivalent reference for all five SOLID principles. Each principle includes the violation symptom (so you can spot it in interview code reviews), the corrective pattern, and a tightly scoped Java example. Use the "interview one-liner" column when you need to explain a principle in a single breath.

### `cheat-sheets/design-patterns-cheat-sheet.md`
Covers all 18 GoF patterns organized by category (Creational, Structural, Behavioral). Each entry shows the pattern's intent, the participants, a canonical use case, and common confusions. The GoF count used here is 18 — Interpreter is excluded as it rarely appears in product LLD interviews.

### `cheat-sheets/lld-interview-cheat-sheet.md`
Interview-day only. Contains the universal LLD approach (requirements → entities → relationships → behaviors → code), patterns for common question archetypes (parking lot, elevator, chess, etc.), and a list of traps that trip up candidates who over-engineer or under-specify. Read it once in the morning; do not re-read it during the interview.

### `design-pattern-matrix.md`
A comparison matrix that evaluates every pattern against dimensions such as: complexity to implement, runtime flexibility, testability impact, and common confusion pairs. Use this when you keep conflating two patterns (e.g., Strategy vs. State, Decorator vs. Proxy) — the matrix makes the differences structural and concrete.

### `flashcards/oop-flashcards.md`
50 cards. Covers the four pillars, SOLID principles, coupling and cohesion, composition vs. inheritance, abstract classes vs. interfaces, and common OOP interview traps. Sorted from foundational to advanced. Do the full deck the first time; on subsequent passes, skip cards you answered correctly.

### `flashcards/design-pattern-flashcards.md`
54 cards — three cards per pattern (intent, code structure, tradeoff). This three-card structure builds three distinct memory traces for the same concept, which dramatically improves interview recall under pressure.

### `uml-templates/class-diagram-template.md`
UML class diagram notation reference (visibility modifiers, relationship arrows, multiplicity), plus ready-to-fill templates for common LLD archetypes: single class with dependencies, inheritance hierarchy, interface + implementations, and composite structure. Use it as a drawing guide before you start any design problem.

### `REVISION-GUIDE.md`
The master pre-interview checklist. Covers every topic in the course, organized into a 7-day countdown. Each day has a focus area, a list of concepts to verify, and a self-assessment prompt. The final day checklist covers logistics, energy management, and what to do in the first 5 minutes of the interview.

---

## Quick Tips

- **Do not cram.** These artifacts are for consolidation, not first-pass learning. If a cheat sheet entry is confusing, go back to the source material.
- **Prioritize flashcards over re-reading.** Reading feels productive; retrieval practice actually builds memory. Flashcards first, cheat sheets second.
- **Use the matrix when you are confused, not before.** The pattern matrix is a diagnostic tool. Pull it out when you mix up two patterns, not as an introduction to patterns.
- **The REVISION-GUIDE is a plan, not a reading assignment.** Work through the checklist — tick items off, note gaps, address gaps. Do not just read it.
