# Module 09 — Capstone Projects

> **You have studied the principles. You have worked through the case studies. Now you prove you can design independently.**

---

## Overview

### What Capstone Projects Are

Capstone projects are open-ended, portfolio-ready design problems that require you to demonstrate mastery — not comprehension. There are no guided walkthroughs here. There is no reference solution to compare yourself against midway through. You receive a problem brief, a set of constraints, and a deliverables checklist. The design is entirely yours.

Each capstone project is evaluated against an explicit rubric covering seven dimensions: domain modeling, abstraction quality, SOLID compliance, code quality, extensibility, artifact completeness, and tradeoff awareness. The rubric is the same one a senior engineer or engineering manager would use to evaluate a design document before a code review. It is not generous, and it should not be — the goal is to produce work you would genuinely be comfortable defending in a technical interview or presenting to a tech lead.

Completing a capstone project means delivering a full artifact set: class diagrams, sequence diagrams, a decision log, a working implementation, and a written tradeoff discussion. Partial submissions — code without diagrams, or diagrams without implementation — do not count as complete. This is by design. In a real interview or a real job, you are expected to communicate your design, not just build it.

### How Capstones Differ from Module 06 Case Studies

The case studies in Module 06 are guided learning exercises. They provide a problem statement, then walk you through requirements extraction, a reference class hierarchy, sequence diagrams, a decision log, and a full implementation. Working through a case study, you are learning *how senior engineers think through this category of problem*. The solution is always available. The experience is closer to studying a worked example than solving an unseen problem.

Capstone projects are fundamentally different in four ways:

**No guided solution.** You will not find a reference implementation to compare your work against. You may consult the case studies and patterns from earlier modules as inspiration, but the design decisions are yours to make and yours to defend.

**Full artifact delivery is required.** Case studies demonstrate artifacts one section at a time. Capstone projects require you to produce the complete artifact set from scratch. You cannot claim completion without all deliverables in place.

**Open-ended requirements.** Case study requirements are pre-extracted and organized for you. Capstone project requirements are written as you might receive them in a real interview — as a scenario with embedded ambiguity. Part of the exercise is identifying what needs clarification and what you can resolve with a stated assumption.

**Designed for defense.** When you complete a capstone project, the final step is writing a design review response — a document where you anticipate the questions a skeptical tech lead would ask about your choices and write your answers down. This practice is described in the section on the Defend-Your-Design Rule. It mirrors the real interview experience and forces you to pressure-test your own reasoning.

### The Philosophy

The capstone projects are built on one premise: **you learn to design by designing under real conditions, not by reading about design**. 

Real design conditions mean real constraints. You have incomplete information and must state assumptions. You have multiple viable approaches and must pick one and commit to it. You have competing concerns — extensibility versus simplicity, consistency versus performance, correctness versus speed of delivery — and you must make tradeoffs explicitly, not by accident.

The goal is not to produce a perfect design. There is no such thing. The goal is to produce a *defensible* design: one where every significant choice was deliberate, every tradeoff was acknowledged, and every interface boundary was drawn with a reason. When you finish a capstone project, you should be able to look a senior engineer in the eye and explain why the system is structured the way it is, what you gave up, and what you would change if requirements shifted.

That is what Senior Engineer–level design looks like. That is what this module is for.

---

## The Five Capstone Projects

---

### Project 1 — Document Collaboration System

**Complexity**: `Intermediate`

---

#### Scenario

Your team is building the document engine for an internal productivity platform. The product manager has scoped an initial version that allows multiple users to work on documents concurrently, maintains a full version history, and enforces role-based permissions so that not every collaborator can do everything. The system will eventually be integrated into a larger suite, so the design must be clean and extensible, but for now it is a standalone document service.

You are the sole engineer on this design. You have one week before you need to present it to the tech lead.

---

#### Core Requirements

**Functional Requirements**

1. Users can create, open, edit, and delete documents.
2. A document has a title, body content, and metadata (created time, last modified time, owner).
3. Multiple users can be collaborators on a document. Each collaborator has one of three roles: `OWNER`, `EDITOR`, or `VIEWER`.
4. Only `OWNER` and `EDITOR` roles can perform edits. `VIEWER` can only read.
5. Only the `OWNER` can delete the document or change collaborator roles.
6. Every edit operation is recorded as a versioned snapshot. The system can restore a document to any previous version.
7. Users can undo the last N operations on a document within an active session.
8. Redo must work after an undo (standard undo/redo stack semantics).
9. A conflict occurs when two users attempt to modify the same region of the document simultaneously. The system must have an explicit strategy for resolving conflicts — you must choose one and document why.

**Non-Functional Requirements**

1. The document model must be independent of any particular storage implementation. Business logic must not reference file systems, databases, or network calls directly.
2. The access control logic must be centralized — not duplicated across every operation.
3. Adding a new document operation type (e.g., insert image, add comment) must not require modifying existing operation handling code.
4. The undo/redo mechanism must support any operation that implements a reversible contract — it must not be hardcoded to specific operation types.

---

#### Required Deliverables

1. **Class diagram** — covering all entities (Document, User, Collaborator, Role, Operation, Version, ConflictResolver), their relationships, cardinalities, and interface boundaries. Every interface and abstract class must be present.
2. **Sequence diagram: Concurrent Edit with Conflict Resolution** — showing two users editing simultaneously, the conflict detection trigger, and the resolution path.
3. **Sequence diagram: Undo / Redo flow** — showing the full operation stack interaction, including what happens when undo is called at the beginning of the stack.
4. **Decision log** — at minimum covering: (a) your chosen conflict resolution strategy and why, (b) how you structured the Command pattern for operations, (c) the access control enforcement point, and (d) how versioning is stored.
5. **Working implementation** in Java — covering all functional requirements. The implementation must demonstrate the Command pattern (undo/redo), role-based access control, and version management.
6. **Unit tests** — covering: role-enforcement (all role/permission combinations), undo/redo stack behavior (including edge cases), and version restoration.

---

#### Stretch Goals

1. **Operational Transformation lite**: Instead of a simple last-write-wins conflict strategy, implement a simplified operational transformation that adjusts the position of incoming edits based on concurrent edits already applied. You do not need a production-grade OT algorithm — a two-user positional adjustment is sufficient. Document the invariants it maintains and the cases it does not handle.
2. **Document templates**: Allow documents to be created from a template, where the template defines the initial content and permitted operation types. A "meeting notes" template, for example, might restrict body-level deletions but allow section-level additions. Show how this extends the existing design without modifying the Document or Operation classes.
3. **Branching versions**: Extend versioning to support branches — a collaborator can fork the document at a given version, make changes independently, and request a merge back. Design the merge conflict resolution interface.

---

#### Key Design Challenges

- **The Command pattern scope**: The undo/redo mechanism must be genuinely operation-agnostic. The trap most engineers fall into is building a switch statement inside the undo dispatcher that knows about every operation type. A clean design means the undo/redo stack never inspects the type of the operation — it only calls a reversed execution contract.
- **Access control enforcement point**: Where does the permission check live? If it lives inside each operation class, you have SRP violations and duplicated logic everywhere. If it lives in a single dispatcher or service, you need to think carefully about how operation types communicate their required permission level without the dispatcher hardcoding a map.
- **Version storage**: Storing a full copy of the document body on every edit is simple but expensive at scale. Storing deltas (diffs) is efficient but complicates restoration. You must choose a strategy and explicitly state what you gave up.
- **Conflict scope**: Defining what constitutes a "conflict" is harder than it looks. Character-level, line-level, and section-level are all valid units of conflict scope, and each produces different resolution semantics. You must define your scope explicitly and defend it.

---

### Project 2 — Multi-Tenant Task Management Platform

**Complexity**: `Senior`

---

#### Scenario

You are designing the core domain model for a B2B task management SaaS product — think Jira or Linear, but starting from scratch. The platform serves multiple organizations (tenants). Each organization has isolated data: their workspaces, projects, and tasks must never be visible to users of another organization, and the platform's internal logic must enforce this boundary structurally, not just by convention.

The product team has given you a requirements document from a recent discovery sprint. The engineering lead wants to see a full class-level design before any implementation begins. You have three days.

---

#### Core Requirements

**Functional Requirements**

1. The platform supports multiple organizations (tenants). Each tenant is isolated: no user of Tenant A can access data belonging to Tenant B.
2. Within a tenant, there are one or more workspaces. A workspace is the top-level organizational unit (analogous to a Jira project space).
3. Within a workspace, there are projects. Within a project, there are tasks.
4. Tasks can have subtasks (one level of nesting is sufficient — a subtask cannot itself have subtasks).
5. Tasks have: title, description, status (`TODO`, `IN_PROGRESS`, `IN_REVIEW`, `DONE`, `CANCELLED`), priority (`LOW`, `MEDIUM`, `HIGH`, `CRITICAL`), assignee (a user within the same workspace), due date, and labels (free-form tags).
6. Each workspace can define custom fields that apply to all tasks in that workspace. Custom fields have a name, type (`TEXT`, `NUMBER`, `DATE`, `ENUM`), and for `ENUM` type, a defined list of allowed values.
7. Every state-changing operation on a task (created, assigned, status changed, due date changed, custom field set, comment added) generates an activity event. The activity feed shows the history of all events on a task in chronological order.
8. Notification rules: users can subscribe to notification triggers (e.g., "notify me when any task in project X is assigned to me" or "notify me when a task I'm watching changes status to DONE"). Notification delivery is out of scope for this design — the design must only model the trigger and subscriber structure.
9. Users have workspace-level roles: `WORKSPACE_ADMIN`, `MEMBER`, `GUEST`. Guests can only view tasks assigned to them. Members can create and edit tasks. Admins can manage workspace configuration including custom fields.

**Non-Functional Requirements**

1. Multi-tenancy isolation must be enforced at the model level — the design must make it structurally difficult (not just conventionally discouraged) to return data belonging to a different tenant.
2. Adding a new task event type must not require modifying existing subscribers or the activity feed logger.
3. Adding a new notification trigger type must not require modifying the existing rule evaluation engine.
4. Custom fields must be extensible without modifying the Task entity's core structure.
5. The activity logging system and the notification system must operate independently — a failure in notification delivery must not affect activity logging, and vice versa.

---

#### Required Deliverables

1. **Class diagram** — covering all entities (Tenant, Workspace, Project, Task, Subtask, User, WorkspaceRole, Label, CustomFieldDefinition, CustomFieldValue, ActivityEvent, NotificationRule, NotificationSubscriber), their relationships, and interface boundaries. The diagram must show how multi-tenancy isolation is structurally represented.
2. **Sequence diagram: Task status change with activity logging and notification dispatch** — showing the full event propagation path from the status change call through activity event creation through notification subscriber evaluation.
3. **Sequence diagram: Custom field addition to a workspace** — showing how a new custom field definition propagates to existing tasks.
4. **Decision log** — at minimum covering: (a) multi-tenancy isolation strategy, (b) the event system design (Observer? event bus? direct dispatch?), (c) how custom fields are stored without polluting the Task schema, and (d) how notification rules are evaluated without a hardcoded rule engine.
5. **Working implementation** in Java — covering all functional requirements. The implementation must demonstrate: tenant isolation, the observer-driven event system, extensible entity model with custom fields, and role-based access control.
6. **Unit tests** — covering: tenant isolation enforcement (verify that cross-tenant data access is structurally prevented), event propagation (verify all subscribers receive events), custom field validation (verify ENUM constraints), and role enforcement.

---

#### Stretch Goals

1. **Board view with column ordering**: Add the concept of a Board within a Project — tasks are organized into columns, each column mapping to a status or a custom grouping. Column order is configurable per board. The task's status and its board column position must stay consistent. Model the relationship and show the invariant.
2. **Task dependencies**: Allow tasks to declare dependencies on other tasks (`BLOCKS`, `IS_BLOCKED_BY`, `RELATES_TO`). Add validation that prevents circular blocking dependencies. Show the data model and the validation algorithm in the decision log.
3. **Workspace-level audit log**: Beyond per-task activity feeds, maintain a workspace-level audit log of all administrative actions (user invited, custom field added, project archived). Design this as a separate concern from the task activity feed, reusing the event infrastructure but with a distinct persistence path.

---

#### Key Design Challenges

- **Structural multi-tenancy**: The naive implementation passes a `tenantId` as a parameter to every query and hopes that developers remember to filter. A well-designed system makes it structurally impossible to load a task without first resolving the tenant context. Think about how the object graph itself enforces this: a Task is only reachable by traversing through a Project, which is only reachable through a Workspace, which belongs to a Tenant.
- **Observer fan-out**: A single task status change might trigger multiple independent concerns: activity logging, notification dispatch, webhook delivery (stretch), analytics (stretch). These must not be coupled. The design challenge is wiring up multiple subscribers without creating a monolithic dispatcher.
- **Custom field storage**: Task has a fixed schema. Custom fields are defined per-workspace, dynamically. Storing them as a map on the Task entity is the obvious choice, but it has implications for querying, validation, and type safety. You must design the CustomFieldValue model to enforce type constraints at the domain level, not just in the UI.
- **Notification rule expressiveness vs. complexity**: A general-purpose rule engine is complex. A hardcoded set of supported trigger types is simple but not extensible. Your design must find a principled middle ground and state where the extensibility boundary lives.

---

### Project 3 — Real-Time Auction Engine

**Complexity**: `Senior`

---

#### Scenario

You have been hired by a marketplace startup that needs an auction engine as a core component. The business has validated three auction formats with initial customers and will likely add more. The CTO has explicitly said: "The auction type logic cannot be hardcoded — we'll add new formats every quarter." The engineering team is small, and your class-level design will be the canonical reference for this component for the next 18 months.

You are tasked with designing the auction engine from scratch. The system handles item listings, bidding, auction lifecycle management, and result determination. Assume the frontend and persistence layers are handled by other teams — your design covers the domain model and business logic only.

---

#### Core Requirements

**Functional Requirements**

1. Sellers can list items for auction. A listing includes: item description, starting price, reserve price (optional, minimum price to sell), auction type, and schedule (start time, end time).
2. The auction lifecycle follows these states: `DRAFT` → `SCHEDULED` → `ACTIVE` → `ENDING_SOON` (final N minutes, configurable) → `CLOSED`. A `CANCELLED` state is reachable from `DRAFT` and `SCHEDULED` only.
3. Three auction types must be supported:
   - **English Auction**: Open ascending bid. Highest bid at close wins. Bidders can see the current highest bid.
   - **Dutch Auction**: Price starts high and decreases on a schedule. First bidder to accept the current price wins. Auction closes immediately on first accepted bid.
   - **Sealed-Bid Auction**: All bids are submitted privately. No bidder can see others' bids. At close, the highest bid wins (or the highest above reserve, if set).
4. **Proxy bidding** applies to English auctions: a bidder sets a maximum bid, and the system automatically places the minimum competitive bid on their behalf as others bid, up to their maximum. If two proxy bids compete, the earlier-submitted proxy wins ties.
5. Bid validation: a bid must be rejected if: (a) the auction is not in `ACTIVE` or `ENDING_SOON` state, (b) the bid amount is below the current minimum increment, (c) the bidder is the seller, or (d) the bidder has been suspended from this auction.
6. When an auction closes, the system determines the winner according to the auction type's rules and checks the reserve price. If the reserve price was not met, the auction closes with no winner (item not sold).
7. An auction result includes: winner (if any), winning bid amount, whether reserve was met, and a complete bid history.

**Non-Functional Requirements**

1. Adding a new auction type must require creating a new class — it must not require modifying the auction lifecycle, bid validation, or result determination logic in existing classes.
2. State transitions must be enforced by the model, not by conditional logic scattered across service methods. An invalid transition (e.g., `ACTIVE` → `DRAFT`) must be structurally prevented.
3. The bid placement logic must be safe under concurrent access. Two valid bids submitted simultaneously must not both succeed if only one can be valid (e.g., Dutch auction — only the first accepted bid should win).
4. The proxy bidding engine must be independent of the bid submission path and not leak into the English auction type implementation in a way that prevents its reuse or replacement.

---

#### Required Deliverables

1. **Class diagram** — covering all entities (AuctionListing, Item, Bid, BidResult, AuctionState, AuctionType strategy hierarchy, AuctionLifecycle state machine, BidValidator, ProxyBiddingEngine, AuctionResult), their relationships, and interface boundaries. The Strategy pattern application for auction types must be explicit.
2. **State machine diagram** — showing all auction lifecycle states, valid transitions, trigger conditions for each transition, and guard conditions.
3. **Sequence diagram: English auction with proxy bidding** — showing a full scenario: auction goes active, two proxy bids are placed, a manual bid from a third bidder is placed, proxy engine responds, auction enters ENDING_SOON, auction closes with winner determination.
4. **Sequence diagram: Sealed-bid auction close** — showing the result determination path when the reserve price is not met.
5. **Decision log** — at minimum covering: (a) how the Strategy pattern is applied to auction types and what the interface contract is, (b) how state transitions are enforced (state machine vs. conditional checks), (c) concurrency strategy for bid placement, and (d) where proxy bidding logic lives and why.
6. **Working implementation** in Java — covering all functional requirements. The implementation must demonstrate: the Strategy pattern for auction types, a state machine for auction lifecycle, bid validation, and proxy bidding.
7. **Unit tests** — covering: all bid rejection scenarios (invalid state, below increment, seller bidding), proxy bid auto-increment behavior, Dutch auction first-bid-wins, sealed-bid result determination with and without reserve, and state transition enforcement.

---

#### Stretch Goals

1. **Vickrey Auction (Second-Price Sealed Bid)**: Add a fourth auction type where the winner pays the second-highest bid amount, not their own. The winner determination logic changes; the bid collection and sealing logic does not. Implement this as a new strategy with no changes to existing auction type classes.
2. **Auction extension on last-minute bid**: In English auctions, if a bid is placed within the final 2 minutes, automatically extend the `ENDING_SOON` window by N minutes (configurable). This is an anti-sniping measure. Design this as an optional policy that can be attached to an auction at configuration time, not hardcoded into the English auction type.
3. **Bid retraction**: Allow a bidder to retract a bid under specific conditions (e.g., in sealed-bid auctions before closing, or in English auctions only if they were outbid and the retraction is within 5 minutes of placing the bid). Model this as an explicit operation with its own state and validation rules.

---

#### Key Design Challenges

- **State machine design**: The auction lifecycle states are straightforward to list. Making the transitions structurally enforced — so that code cannot accidentally call `close()` on a `DRAFT` auction — requires committing to a state machine pattern. The State pattern is the canonical choice, but it adds object count. An explicit transition map with guard evaluation is leaner. You must choose and justify.
- **Strategy vs. type hierarchy**: English, Dutch, and Sealed-Bid auctions differ in: how bids are validated, how the current price is determined, when the auction closes, and how the winner is determined. Some of these are lifecycle concerns (the AuctionLifecycle manages them), and some are type-specific (the AuctionType strategy handles them). Partitioning responsibilities between the lifecycle and the type strategy cleanly is the central design challenge of this project.
- **Proxy bidding isolation**: Proxy bidding is a feature of English auctions, but it should not live inside the EnglishAuctionType class. If it does, it becomes untestable in isolation, impossible to reuse, and entangled with every English auction behavior. The design challenge is making the ProxyBiddingEngine a collaborator that the lifecycle can invoke, with a clear interface contract.
- **Concurrency in bid placement**: A Dutch auction has a "first accepted bid wins" rule. Two simultaneous requests to accept the current price must be serialized. The design must show where the concurrency boundary is (what is the scope of the lock or compare-and-swap operation) without making the entire auction object a single coarse-grained lock.

---

### Project 4 — Pluggable Data Pipeline Framework

**Complexity**: `Staff`

---

#### Scenario

A platform engineering team is building an internal data pipeline framework for use across multiple product teams. Product teams need to define multi-step data processing pipelines: ingest data from a source, transform it through a sequence of processing stages, and deliver it to a sink. Pipelines must be composable (stages can be reused across multiple pipelines), observable (each stage emits metrics), and fault-tolerant (a stage failure must not silently lose data).

You have been brought in as the Staff Engineer leading this design. The framework must be opinionated enough to provide strong guarantees (error handling, metrics, serialization), but flexible enough that product teams can plug in custom processors without modifying the framework.

This is a framework design problem, not an application design problem. Your primary customer is the developer who will use this framework to build a pipeline, not the end user whose data flows through it.

---

#### Core Requirements

**Functional Requirements**

1. A pipeline is a directed acyclic graph (DAG) of processing stages. Each stage takes an input payload, processes it, and emits one or more output payloads.
2. Stages must be individually composable: a given stage can appear in multiple pipelines without modification.
3. The framework must support fan-out: a single stage can emit output to multiple downstream stages simultaneously.
4. The framework must support fan-in: a stage can receive inputs from multiple upstream stages.
5. If a stage fails to process a payload (throws an exception), the payload must be routed to a dead-letter queue associated with that stage. The pipeline must continue processing other payloads. The dead-letter queue must be inspectable and re-driveable (a failed payload can be resubmitted to the failed stage).
6. Each stage must emit the following metrics automatically, without the stage author needing to add instrumentation code: input record count, output record count, error count, and processing latency (min/max/avg over a configurable window).
7. The framework must support pluggable serialization: payloads can be serialized as JSON, Avro, or a custom format. The serialization format is configured per-pipeline, not per-stage.
8. The pipeline execution model supports at minimum a synchronous (sequential) execution mode. An asynchronous execution mode is a stretch goal.

**Non-Functional Requirements**

1. Adding a new processor stage must require only implementing a single interface — the framework must not require the new stage author to interact with metrics, error handling, dead-letter routing, or serialization directly.
2. Adding a new serialization format must require only providing a new serializer implementation — it must not require any changes to stage code or pipeline configuration beyond specifying the new format.
3. Adding a new metrics collection backend (e.g., switching from in-memory counters to a Prometheus reporter) must not require changes to any stage or pipeline code.
4. The DAG structure must be validated before pipeline execution begins: cycles must be detected and reported as configuration errors, not runtime failures.
5. The framework must be testable: a pipeline must be executable in a test environment without external dependencies (no real queues, no real serializers required for unit tests).

---

#### Required Deliverables

1. **Class diagram** — covering all framework entities (Pipeline, PipelineStage, Processor interface, StageConnection, DAGValidator, DeadLetterQueue, StageMetrics, MetricsCollector, SerializationFormat, Serializer, PipelineContext, ExecutionResult), their relationships, and interface boundaries. The Visitor pattern for metrics collection and the Chain of Responsibility for error handling must be explicit.
2. **Sequence diagram: Successful multi-stage pipeline execution** — showing payload flow from source through three stages to sink, with metrics collection at each stage.
3. **Sequence diagram: Stage failure with dead-letter routing and re-drive** — showing a stage throwing an exception, dead-letter queue capture, inspection, and re-submission to the stage.
4. **Sequence diagram: Fan-out execution** — showing a single stage emitting to two downstream stages and both processing independently.
5. **Decision log** — at minimum covering: (a) why Visitor is used for metrics and what the alternative was, (b) how Chain of Responsibility is applied to error handling and where in the chain the dead-letter queue sits, (c) DAG validation strategy (how cycles are detected), (d) how the framework isolates stage authors from cross-cutting concerns, and (e) the chosen execution model and its tradeoffs.
6. **Working implementation** in Java — covering all functional requirements. The implementation must demonstrate: the Visitor pattern for metrics collection, the Chain of Responsibility for error handling, DAG validation, dead-letter queue, and pluggable serialization.
7. **Unit tests** — covering: DAG cycle detection, dead-letter routing on stage failure, metrics accumulation across stage invocations, and serializer pluggability (verify that swapping serializers changes wire format without breaking stage logic).

---

#### Stretch Goals

1. **Asynchronous execution mode**: Add a parallel execution mode where independent stages in the DAG execute concurrently. Dependent stages (those with a data dependency from an upstream fan-in) wait for all upstream stages to complete before executing. Show how the execution model is abstracted behind a strategy so that synchronous and asynchronous execution are interchangeable at the pipeline level.
2. **Pipeline versioning and schema evolution**: Pipeline configurations change over time. Allow a pipeline to declare a version, and allow stages to declare which payload schema version they accept. When a payload serialized with version N arrives at a stage expecting version N+1, an optional migration transformer is applied. Design the migration interface and show how it integrates with the serialization layer.
3. **Conditional routing**: Add support for content-based routing — a stage can emit a payload to different downstream stages based on the payload's content. Design a RoutingCondition interface and show how it integrates with fan-out without modifying the Stage class.

---

#### Key Design Challenges

- **The framework vs. application boundary**: This is the highest-order design challenge. The framework must provide strong guarantees (error handling, metrics, dead-letters) without requiring stage authors to know those guarantees exist. This means the framework must intercept every stage execution and apply its concerns as cross-cutting decoration. The Decorator or Template Method pattern applied at the framework layer — not the stage author's layer — is the standard approach. Getting this partition right is the core of this project.
- **Visitor for metrics**: Metrics collection is a cross-cutting concern that needs to observe the internals of each stage invocation without coupling to any specific stage type. The Visitor pattern solves this cleanly by separating the observation logic from the stage logic. The challenge is defining the visitable interface so that the framework's metrics visitor can observe what it needs without the stage author exposing implementation details.
- **DAG validation before execution**: Detecting cycles in a directed graph before execution begins requires a topological sort or depth-first traversal with back-edge detection. The more interesting challenge is designing the pipeline configuration API so that it is easy to build valid DAGs and hard to express cycles — validation is a safety net, not the first line of defense.
- **Dead-letter queue contract**: What contract does the dead-letter queue expose? It must accept failed payloads. It must allow inspection. It must allow re-drive. But it must not depend on any specific stage type or payload type. Designing a type-safe dead-letter queue that works across all payload types without losing type information is the specific challenge here.

---

### Project 5 — Financial Transaction Ledger

**Complexity**: `Staff`

---

#### Scenario

You are the Staff Engineer on a fintech platform building its core accounting engine. The product is a multi-currency expense management system for small businesses. At its heart is a financial ledger — the source of truth for all money movement in the system. Every transaction, every balance, every reconciliation flows through this component.

The CTO has given you explicit domain constraints that are non-negotiable: (1) all recorded transactions are immutable — they cannot be modified or deleted after commit; (2) the ledger must implement double-entry bookkeeping; (3) every balance must be derivable from the transaction history — there are no cached balances that can become inconsistent with the ledger. You are responsible for designing the domain model and business logic of this engine.

This is a domain model purity problem. The challenge is not algorithmic complexity — it is designing a model where the invariants are enforced structurally, not by hope.

---

#### Core Requirements

**Functional Requirements**

1. The ledger supports multiple accounts. Each account has: account ID, name, type (`ASSET`, `LIABILITY`, `EQUITY`, `REVENUE`, `EXPENSE`), currency, and owner (a reference to an entity — user, team, or organization — that owns the account).
2. Transactions follow double-entry bookkeeping: every transaction consists of two or more line items (journal entries), and the sum of debits must equal the sum of credits across all line items in the transaction.
3. Each line item records: account reference, amount, currency, direction (`DEBIT` or `CREDIT`), and an optional memo.
4. A transaction has: transaction ID, timestamp, description, status (`PENDING`, `POSTED`, `VOIDED`), and a reference to the journal entry set (the line items).
5. Once a transaction reaches `POSTED` status, it cannot be modified in any field. Attempted modification must throw a domain exception, not silently fail.
6. Voiding a posted transaction does not delete or modify the original transaction. It creates a new reversal transaction with the opposite debit/credit directions. The original transaction remains in the ledger with a reference to its reversal.
7. Balance calculation: the current balance of an account is computed by summing all posted transactions affecting that account. There is no separately stored balance field — balance is always derived.
8. The system supports multi-currency accounts. Exchange rate snapshots are recorded at the time of a cross-currency transaction. Balance reporting in a reference currency uses the exchange rate snapshots stored with each transaction — it does not re-fetch live rates.
9. Reconciliation: for a given account and time range, the system can produce a reconciliation report showing: opening balance, all transactions in the range, closing balance, and any discrepancies if an external balance is provided for comparison.
10. The system generates a Journal Entry report: a chronological list of all debit and credit entries for a given account, analogous to a bank statement.

**Non-Functional Requirements**

1. The domain model must enforce the double-entry constraint at the object model level — it must be impossible to persist a transaction that is not balanced without the constraint being explicitly bypassed.
2. All monetary amounts must be represented as value objects with currency — a raw `double` or `BigDecimal` without currency context must never appear as a method parameter or return type in the domain layer.
3. Immutability after commit must be enforced by the domain model, not by the persistence layer. The domain object itself must refuse modification after it reaches `POSTED` state.
4. Exchange rate snapshots must be stored with the transaction, not in a separately queryable table that can diverge. Historical balance reports must be deterministic regardless of when they are run.
5. The balance calculation must be pure: given the same set of posted transactions, it must always return the same balance with no external dependencies.

---

#### Required Deliverables

1. **Class diagram** — covering all entities (Account, AccountType, Transaction, TransactionStatus, JournalEntry, LineItem, Money value object, Currency, ExchangeRate, ExchangeRateSnapshot, LedgerService, BalanceCalculator, ReconciliationReport), their relationships, and interface boundaries. Value objects and immutability must be explicitly shown (e.g., with stereotypes or notes on the diagram).
2. **Sequence diagram: Complete transaction lifecycle** — showing transaction creation in PENDING state, validation of double-entry balance, posting, and the immutability lock that prevents subsequent modification.
3. **Sequence diagram: Transaction voiding and reversal** — showing the original transaction, the reversal transaction creation with opposite entries, and how both appear in the journal entry report.
4. **Sequence diagram: Multi-currency balance report** — showing how exchange rate snapshots stored on transactions are used to compute a reference-currency balance without external rate fetching.
5. **Decision log** — at minimum covering: (a) how the double-entry invariant is enforced at the model level, (b) the Money value object design and why primitive types are excluded, (c) the immutability-after-commit mechanism, (d) how exchange rate snapshots are associated with transactions, and (e) the balance derivation strategy (derived from history vs. cached — explain why cached balance is rejected for this domain).
6. **Working implementation** in Java — covering all functional requirements. The implementation must demonstrate: Money value object with currency, immutable Transaction post-commit, double-entry validation, balance derivation, and reversal transaction creation.
7. **Unit tests** — covering: double-entry balance validation (both valid and unbalanced transactions), post-commit immutability enforcement (verify exception on modification attempt), balance derivation correctness (known transaction set, expected balance), multi-currency balance report using rate snapshots, and reconciliation report accuracy.

---

#### Stretch Goals

1. **Audit trail with cryptographic chaining**: Extend the immutable audit trail so that each posted transaction contains a hash of the previous transaction in the account's history. This creates a tamper-evident chain: any modification to a historical transaction invalidates all subsequent hashes. Design the hash computation (what fields are included), the chain verification algorithm, and how the chain handles multi-account transactions.
2. **Period locking**: Allow the system to close a financial period (e.g., a month). Once a period is closed, no new transactions can be posted with a timestamp within that period. Existing posted transactions in the period become doubly immutable — they cannot even be voided. The only way to correct a closed-period transaction is to post an adjusting entry in the current open period. Model the period boundary as a first-class domain concept.
3. **Chart of accounts hierarchy**: Extend accounts to support a hierarchical chart of accounts (parent-child relationships). A parent account's balance is the sum of all its children. Balance reports can be generated at any level of the hierarchy. Design the recursive balance computation and show how it handles multi-currency children under a parent reporting in a reference currency.

---

#### Key Design Challenges

- **Domain model purity and invariant enforcement**: This project is fundamentally about building a domain model that cannot be put into an invalid state. The double-entry constraint must be checked before a transaction can be marked as anything other than `PENDING`. The `POSTED` immutability must be structural — meaning the Transaction class itself must refuse setters and modification methods after posting, not depend on callers checking the status field before calling. This requires careful constructor design, factory methods, and state-machine-controlled mutability.
- **Money as a value object**: A monetary amount without a currency is meaningless in a multi-currency system. The `Money` class must be a true value object: immutable, with value equality, and with currency-aware arithmetic (two `Money` objects in different currencies cannot be added without an explicit currency conversion). Every method signature in the domain layer that accepts or returns a monetary amount must use `Money`, not `BigDecimal`.
- **Immutability under the JVM**: Java objects are mutable by default. Enforcing immutability after commit requires more than setting a boolean flag. Consider: final fields for post-commit data, package-private setters used only by the factory/constructor, or a two-phase object construction pattern where the mutable builder produces an immutable posted transaction. Each approach has tradeoffs; you must choose and justify.
- **Balance derivation performance**: Deriving balance from full transaction history is correct and simple, but it scales poorly for an account with millions of transactions. The design must acknowledge this tradeoff and provide a path to checkpointing (a known-correct balance at a known point in history, beyond which only new transactions need to be summed) without violating the no-cached-balance constraint for the initial design.

---

## How to Approach Capstone Projects

### Recommended Order

Work through the five projects in the order they are listed. The ordering is deliberate.

**Projects 1 and 2** share structural DNA with the case studies you completed in Module 06. Project 1 (Document Collaboration) extends the Command pattern and role-based access control concepts from the Notification Framework and Workflow Engine case studies. Project 2 (Task Management) extends the multi-entity modeling and observer-based event propagation from the Ride-Sharing and Inventory Management case studies. Starting here lets you apply familiar patterns in a more open-ended context before the scaffold is fully removed.

**Projects 3** introduces the State pattern applied at the lifecycle level — a pattern that appears in several case studies but is now the central design concern rather than a supporting element. Project 3 also introduces concurrency as a first-class design constraint. These are Senior-level concerns that require a higher level of deliberate attention than the Intermediate projects.

**Projects 4 and 5** are Staff-level problems. Project 4 requires thinking like a framework author rather than an application developer — the primary design goal is protecting the caller from complexity, not modeling a business domain. Project 5 requires domain model discipline of a kind that is rarely needed in application code but is essential in regulated or financial domains. Both projects require synthesizing patterns from across the entire course simultaneously.

Doing Project 5 before Project 3, for example, risks building the ledger as a simple CRUD model because you haven't yet internalized state machine design. Doing Project 4 before Project 2 risks missing the framework abstraction opportunities because you haven't practiced distinguishing application logic from cross-cutting concerns.

### Time Investment Per Project

These estimates assume focused, uninterrupted work and include both design time (producing all diagrams and the decision log) and implementation time (working Java code with tests).

| Project | Design (hours) | Implementation (hours) | Total |
|---|---|---|---|
| Project 1 — Document Collaboration | 3–4 | 5–7 | **8–11 hours** |
| Project 2 — Task Management Platform | 4–5 | 7–9 | **11–14 hours** |
| Project 3 — Auction Engine | 5–6 | 7–9 | **12–15 hours** |
| Project 4 — Data Pipeline Framework | 6–8 | 9–12 | **15–20 hours** |
| Project 5 — Financial Ledger | 6–8 | 9–12 | **15–20 hours** |
| **Total** | | | **61–80 hours** |

Design time includes: reading the brief thoroughly, writing clarifying questions and assumptions, producing the class diagram, producing the sequence diagrams, and writing the decision log. Do not collapse this into a 30-minute sketch — the decision log alone should take 60–90 minutes per project.

**If you are time-constrained**: Complete Project 1 in its entirety (all deliverables, all tests). This gives you one portfolio-ready artifact. Then spend 50% of remaining time on Project 3 (carry it through design and partial implementation — the state machine and Strategy pattern components are the high-value sections). The combination of a complete Intermediate project and a partially completed Senior project with thorough design artifacts is more valuable in a portfolio than five superficially completed projects.

### The Design-First Rule

Begin every capstone project by producing the complete design artifacts before writing a single line of implementation code. This is not a stylistic preference — it is a discipline that separates senior-level design practice from coding-with-intentions.

When you write implementation code before completing the class diagram, you make dozens of micro-decisions about class structure, interface boundaries, and responsibility assignment without conscious deliberation. These decisions calcify into the implementation and become expensive to reverse. By the time you notice that your AccessController ended up with ten methods because you never decided what it was actually responsible for, you have already built a large portion of the system on top of it.

The design phase forces three disciplines that implementation suppresses: (1) **naming discipline** — you cannot draw a class in a diagram without committing to its name and its responsibilities; (2) **interface discipline** — drawing relationships between classes forces you to define the contracts between them before you know how you will implement either side; (3) **scope discipline** — a complete class diagram forces you to identify all entities in the system before you are invested in any particular implementation path.

The sequence diagrams are equally important. They reveal coupling that class diagrams hide. A class diagram might show that `AuctionService` collaborates with `ProxyBiddingEngine`, but only a sequence diagram reveals that `AuctionService` calls `ProxyBiddingEngine.recalculate()` three times in a single bid submission — a coupling problem that would be invisible until the code is written.

Complete the class diagram. Complete the sequence diagrams. Write the decision log. Then write the code.

### The Defend-Your-Design Rule

After completing all deliverables for a capstone project, add one final artifact to your submission: a **Design Review Response document**.

This document is structured as a set of anticipated questions from a skeptical tech lead, paired with your answers. Write at minimum 8 questions per project. The questions should be adversarial — not "why did you use the Strategy pattern?" but "why didn't you just use a switch statement and add a new case for each auction type?" Good question-answer pairs expose the specific tradeoffs you made.

Sample question prompts to get you started (do not copy these verbatim — write questions specific to your design choices):

- "You chose [X pattern] here. What does that pattern cost, and why is the benefit worth it?"
- "If a new requirement arrives that requires [specific change], how many classes do you need to touch and which ones?"
- "Your [class/interface/method] does [X]. What happens if [edge case Y]?"
- "You chose to enforce [invariant Z] here rather than in [location W]. Why not the other way?"
- "If this system needed to scale to 10x the expected load, which design decisions would you revisit first?"

Writing the answers forces you to confront the cases where your design doesn't hold up cleanly. If you cannot answer a question about your own design, that is a design gap — go fix it before you claim the project is complete. This process simulates the design review and interview experience so precisely that engineers who practice it consistently report dramatically improved confidence when defending designs under real interview pressure.

---

## What "Complete" Means — The Deliverables Checklist

### Universal Deliverables (Required for All Five Projects)

The following artifacts are required for every capstone project. Each item states what "done" looks like — not the artifact name alone.

- [ ] **Class Diagram** — All entities present; all relationships drawn (association, aggregation, composition, dependency, realization, generalization); cardinalities marked on every relationship; all interfaces and abstract classes distinguished from concrete classes; all field names and key method signatures visible at appropriate zoom level. A diagram that shows boxes and lines with no method signatures or cardinalities is not complete.

- [ ] **Sequence Diagram(s)** — Each key workflow covered by at minimum one sequence diagram (project-specific diagrams listed in each brief above); all participating objects shown; every message labeled with the method name and key parameters; return messages shown where they carry meaningful data; alternative frames (alt/opt) used where conditional paths exist.

- [ ] **Decision Log** — Minimum four decisions documented per project; each decision follows the ADR (Architecture Decision Record) format: context (what problem you faced), options considered (at minimum two alternatives), decision (what you chose), and consequences (what you gained and what you gave up). A decision log that only records what you chose without recording what you rejected is not a decision log — it is a description.

- [ ] **Working Implementation** — Java source code compiles without errors; all functional requirements from the project brief are covered by the implementation; no placeholder methods with TODO comments in submitted work; package structure reflects domain boundaries (not a single flat package); no business logic in main or test classes.

- [ ] **Unit Tests** — Tests written in JUnit or equivalent; each test follows Arrange-Act-Assert structure; test method names describe the scenario, not the implementation (`shouldRejectBidOnClosedAuction` not `testBid`); all edge cases listed in the project brief are covered by at least one test; tests pass without modification.

- [ ] **Design Review Response** — Minimum 8 adversarial question-answer pairs per the Defend-Your-Design Rule described above; questions are specific to this submission's design choices (not generic); answers are complete (they include the tradeoff acknowledged, not just the justification for the choice made).

- [ ] **README (project-level)** — Project directory contains a README that states: the project name, the design patterns applied (with one sentence on where each is applied), the key design decisions summarized in three to five bullet points, known limitations, and instructions for running the code and tests.

### Project-Specific Additional Notes

**Project 1 — Document Collaboration System**
- The class diagram must show the Command interface hierarchy with at least three concrete operation types, demonstrating that the undo/redo stack is operation-agnostic.
- The access control enforcement point must be explicitly shown in the sequence diagram — the diagram must make clear that permission checking happens in one place, not distributed across each operation type.
- The decision log must include a section on the version storage strategy comparing full-snapshot vs. delta approaches with explicit tradeoffs stated.

**Project 2 — Multi-Tenant Task Management Platform**
- The class diagram must show how multi-tenancy isolation is represented structurally (the object traversal path from Tenant to Task must be traceable on the diagram).
- The event system sequence diagram must show at least two independent subscribers receiving the same event, demonstrating that they are decoupled.
- The decision log must address the custom field storage design — specifically, how type validation (particularly ENUM value constraints) is enforced at the domain model level.

**Project 3 — Real-Time Auction Engine**
- A complete state machine diagram for the auction lifecycle is required in addition to the class and sequence diagrams. This is not optional.
- The class diagram must show the Strategy pattern interface for auction types with the English, Dutch, and Sealed-Bid concrete strategies, and must show that the lifecycle management code depends only on the interface.
- The decision log must address the concurrency strategy for bid placement in the Dutch auction — specifically, what is the scope of atomic operation and why.

**Project 4 — Pluggable Data Pipeline Framework**
- The class diagram must explicitly show the Visitor pattern structure: the Visitable interface on PipelineStage, the Visitor interface for MetricsCollector, and at least one concrete visitor implementation.
- The sequence diagram for dead-letter routing must show the full re-drive path, not just the capture path.
- The decision log must address the DAG validation algorithm — specifically, how cycle detection is implemented (DFS with coloring, Kahn's algorithm, or other) and the time complexity.

**Project 5 — Financial Transaction Ledger**
- The class diagram must use stereotypes or notes to distinguish value objects (Money, Currency, ExchangeRateSnapshot) from entities (Account, Transaction) — this distinction is a graded criterion.
- The immutability mechanism must be shown explicitly in the class diagram (final fields, absence of setters, or structural notes) — it cannot be left implicit.
- The decision log must address the balance derivation strategy tradeoff between correctness (full recalculation from history) and performance (checkpointing), and must state the conditions under which a checkpointing approach would be introduced.

---

## Evaluation Rubric

The following rubric is used to evaluate capstone submissions. Each of the seven dimensions is scored on a 1/3/5 scale. Total maximum score is 35 points.

Mentors and peer reviewers should score based on observable evidence in the submitted artifacts — not on the candidate's stated intentions.

---

### Scoring Scale

| Score | Meaning |
|---|---|
| 5 | Exemplary — demonstrates senior to staff-level mastery of this dimension |
| 3 | Meets Standard — demonstrates competent, solid work expected of a senior candidate |
| 1 | Poor — demonstrates fundamental gaps that would be visible in an interview setting |

Scores of 2 and 4 are permitted for "between" assessments.

---

### Dimension 1: Domain Modeling

*Have the right entities been identified? Are they at the right level of granularity?*

| Score | Observable Criteria |
|---|---|
| **5** | All domain concepts from the requirements are modeled as explicit entities, value objects, or enums. No important concept is represented implicitly (as a string field or a magic integer). Granularity is appropriate — neither over-split (a separate class for something that is just an attribute) nor under-split (two distinct concepts collapsed into one entity). Relationships reflect real-world domain semantics (an Account owns many Transactions, not the other way around). |
| **3** | All major entities are present. One or two supporting concepts are missing or represented as primitives where a class would be more appropriate (e.g., `String currency` instead of a `Currency` value object). Granularity is mostly appropriate with at most one obvious misjudgment. |
| **1** | Key domain concepts are missing (e.g., no Transaction entity in the ledger, no Bid entity in the auction engine). Or the model is a single God Entity that accumulates all fields. Or relationships are inverted or missing entirely. |

---

### Dimension 2: Abstraction Quality

*Are interfaces defined at the right level? Is the right pattern applied?*

| Score | Observable Criteria |
|---|---|
| **5** | Interfaces are defined at exactly the variation points where they earn their cost. Each interface has a coherent single purpose. The pattern applied is the most appropriate one for the variation point — not the most familiar or the most impressive. The interface contract (method signatures) is minimal (ISP-compliant) and enables the stated extensibility goals. Abstract classes are used where default behavior is shared and interfaces where it is not. |
| **3** | Interfaces exist at the major variation points. One or two abstraction boundaries are drawn slightly too wide (an interface with six methods where two distinct two-method interfaces would be cleaner) or slightly too narrow (an interface created for what is actually a single concrete implementation). Pattern selection is justifiable even if not optimal. |
| **1** | Abstractions are missing at the major variation points (e.g., a strategy pattern is needed but a conditional chain is used instead). Or abstractions exist but are drawn at the wrong level (interfaces on implementation details, not on behavioral contracts). Or patterns are applied where there is no genuine variation point (e.g., Factory for a class with a single constructor and a single implementation). |

---

### Dimension 3: SOLID Compliance

*Which SOLID principles are demonstrated? Which are violated?*

| Score | Observable Criteria |
|---|---|
| **5** | All five principles are visibly applied. SRP: no class has more than one reason to change — each class's responsibility can be described in one sentence. OCP: adding the required stretch goal extension does not require modifying existing classes. LSP: all implementations of any interface honor the interface's behavioral contract without surprising callers. ISP: no class implements an interface method by throwing UnsupportedOperationException or returning null as a signal. DIP: all high-level modules depend on abstractions, not on concrete classes directly. |
| **3** | Four of five principles are clearly applied. One principle is partially applied (e.g., DIP is applied everywhere except for one direct instantiation in a service class). No principle is actively violated in a way that would cause a maintainability problem. |
| **1** | One or more SOLID principles are visibly violated in ways that would be noticed in a real code review: God classes with three or more distinct responsibilities; hardcoded conditionals for what should be a strategy; subclass that throws on a method that the interface contract guarantees will work; interface with ten methods when the callers use at most two. |

---

### Dimension 4: Code Quality

*Is the implementation readable, well-named, and cohesive?*

| Score | Observable Criteria |
|---|---|
| **5** | Class and method names are precise and domain-specific — a reader unfamiliar with the codebase can infer what each class and method does from its name alone. Methods are short (under 20 lines as a strong preference), focused, and do one thing. No method has more than three parameters; complex parameter groups are encapsulated in objects. No magic strings or magic numbers. Cohesion is high: all methods in a class operate on the same state. No dead code, no commented-out blocks, no TODO comments in submitted work. |
| **3** | Names are mostly clear with one or two generic names (e.g., `Manager`, `Handler`, `Data`) that could be more specific. Most methods are focused; one or two are longer than they should be. Magic strings are confined to constants. No dead code in core paths. |
| **1** | Class names are generic and not domain-specific (`Manager`, `Util`, `Helper` as primary names). Methods are long (40+ lines) and do multiple things. Primitive types used throughout where value objects would be appropriate. Magic strings scattered through business logic. Code structure makes it hard to trace a single workflow without jumping through many indirections that do not add clarity. |

---

### Dimension 5: Extensibility

*How hard is it to add the listed stretch goals?*

| Score | Observable Criteria |
|---|---|
| **5** | Adding any of the project's three stretch goals requires only: (a) creating a new class/implementation that honors an existing interface, and/or (b) registering the new implementation in a configuration or factory location. No existing class needs modification. The stretch goal implementation is isolated and does not require understanding the whole system. The Extensibility Test (see dedicated section below) passes on all five test scenarios. |
| **3** | Adding stretch goals requires modifying at most two existing classes. The modifications are confined to extension points (adding a new case to a registration map, adding a new implementation to a strategy list). No modification to core business logic classes is required. Two or three of the five Extensibility Test scenarios pass cleanly. |
| **1** | Adding any stretch goal requires modifying three or more existing classes including core business logic classes (e.g., adding a new auction type requires modifying `AuctionService`, `BidValidator`, and `AuctionResultDeterminer` because all three have conditionals on auction type). The design is closed for extension in the areas the project explicitly identified as variation points. |

---

### Dimension 6: Design Artifact Completeness

*Are the diagrams accurate, and is the decision log substantive?*

| Score | Observable Criteria |
|---|---|
| **5** | Class diagram covers all entities with correct relationships, cardinalities, and interface boundaries. Sequence diagrams trace the complete message flow with no missing objects, no missing return messages where data flows back, and correct use of alt/opt/loop frames. Decision log entries follow ADR format with explicitly stated alternatives and consequences. The artifacts are internally consistent (a class in the sequence diagram exists in the class diagram). |
| **3** | Class diagram is mostly complete; one or two supporting entities are missing. Sequence diagrams cover the happy path clearly; error paths are present but may be less detailed. Decision log entries state what was chosen and why, but do not always record alternatives considered. Minor inconsistencies between class diagram and implementation (an interface in the diagram is missing a method that exists in the code). |
| **1** | Class diagram is incomplete (fewer than 60% of domain entities present, or relationships are present but unlabeled and without cardinalities). Sequence diagrams are missing for one or more required workflows. Decision log contains only a list of pattern names used with no rationale. Artifacts contradict each other or contradict the implementation. |

---

### Dimension 7: Tradeoff Awareness

*Does the candidate acknowledge what they gave up?*

| Score | Observable Criteria |
|---|---|
| **5** | Every significant design decision in the decision log acknowledges the tradeoff made. The candidate can articulate what they gave up, under what conditions their choice becomes problematic, and what would trigger a reconsideration of the choice. The Design Review Response contains adversarial questions that expose real limitations of the design, with honest answers that do not paper over the weakness. |
| **3** | Most decisions acknowledge at least one alternative. The language of tradeoffs is present ("this approach is simpler to implement but more expensive to extend"). One or two decisions are stated as obvious choices without acknowledging the alternative. The Design Review Response is present and covers most of the obvious pushback questions. |
| **1** | The decision log reads as a list of justifications without alternatives ("I chose the Strategy pattern because it is the best pattern for this"). No decision acknowledges a downside. The Design Review Response is absent, superficial (fewer than 4 questions), or consists only of softball questions the candidate can easily answer. |

---

### Score Interpretation

| Total Score | Interpretation |
|---|---|
| **32–35** | Portfolio-ready. This submission is of the quality you would expect from a strong Senior Engineer or Staff Engineer candidate. Submit it in GitHub, link it in your resume, and reference it in interviews without hesitation. |
| **28–31** | Strong pass. Demonstrates clear Senior-level competence with minor gaps. Address the lowest-scoring dimension before submitting publicly. One revision cycle brings this to portfolio-ready. |
| **21–27** | Competent pass. The fundamental design is sound, but there are gaps in either artifact quality, extensibility, or tradeoff awareness that would be noticeable to a discerning interviewer. Requires targeted revision of the weakest two dimensions. |
| **14–20** | Requires significant revision. The core design concepts are present but the execution — either in the artifacts, the code, or the tradeoff reasoning — has gaps significant enough that a senior engineer reviewing this in an interview context would have substantial concerns. Revisit the weakest areas in earlier modules before resubmitting. |
| **7–13** | Foundational gaps present. The submission demonstrates incomplete internalization of the design concepts from Modules 01–05. Return to the case studies, complete at least three independently (without referring to the solutions), and then re-approach the capstone. |

---

## Portfolio Submission Guidelines — Structuring a GitHub Repository

A capstone project is only a portfolio asset if it is presented as one. A repository with 200 uncommitted files, a blank README, and a package named `com.example.test` communicates the opposite of what you intend.

This section describes how to structure a public GitHub repository as a professional LLD portfolio.

### Recommended Repository Structure

```
lld-portfolio/
├── README.md                          ← Root portfolio README (see below)
├── .gitignore                         ← Standard Java/.gitignore
├── 01-document-collaboration/
│   ├── README.md                      ← Project-level README
│   ├── docs/
│   │   ├── class-diagram.png          ← Or .svg / .puml source
│   │   ├── sequence-concurrent-edit.png
│   │   ├── sequence-undo-redo.png
│   │   └── decision-log.md
│   └── src/
│       ├── main/
│       │   └── java/
│       │       └── com/yourname/docsystem/
│       │           ├── domain/        ← Entities and value objects
│       │           ├── service/       ← Application/domain services
│       │           ├── command/       ← Command pattern implementations
│       │           ├── access/        ← Access control logic
│       │           └── version/       ← Versioning subsystem
│       └── test/
│           └── java/
│               └── com/yourname/docsystem/
│                   ├── domain/
│                   ├── service/
│                   └── command/
├── 02-task-management-platform/
│   ├── README.md
│   ├── docs/
│   │   ├── class-diagram.png
│   │   ├── sequence-task-status-change.png
│   │   ├── sequence-custom-field-addition.png
│   │   └── decision-log.md
│   └── src/
│       └── main/java/com/yourname/taskplatform/
│           ├── domain/
│           │   ├── tenant/
│           │   ├── workspace/
│           │   ├── project/
│           │   └── task/
│           ├── event/
│           ├── notification/
│           └── service/
├── 03-auction-engine/
│   ├── README.md
│   ├── docs/
│   │   ├── class-diagram.png
│   │   ├── state-machine-auction-lifecycle.png
│   │   ├── sequence-english-proxy-bidding.png
│   │   ├── sequence-sealed-bid-close.png
│   │   └── decision-log.md
│   └── src/
│       └── main/java/com/yourname/auction/
│           ├── domain/
│           ├── lifecycle/
│           ├── strategy/
│           ├── bidding/
│           └── service/
├── 04-pipeline-framework/
│   ├── README.md
│   ├── docs/
│   │   ├── class-diagram.png
│   │   ├── sequence-pipeline-execution.png
│   │   ├── sequence-dead-letter-redrive.png
│   │   ├── sequence-fanout.png
│   │   └── decision-log.md
│   └── src/
│       └── main/java/com/yourname/pipeline/
│           ├── core/
│           ├── processor/
│           ├── error/
│           ├── metrics/
│           └── serialization/
└── 05-financial-ledger/
    ├── README.md
    ├── docs/
    │   ├── class-diagram.png
    │   ├── sequence-transaction-lifecycle.png
    │   ├── sequence-voiding-reversal.png
    │   ├── sequence-multicurrency-balance.png
    │   └── decision-log.md
    └── src/
        └── main/java/com/yourname/ledger/
            ├── domain/
            │   ├── account/
            │   ├── transaction/
            │   └── money/
            ├── service/
            └── reporting/
```

### What Goes in Each Location

**Root `README.md`**: This is the landing page for anyone who navigates to your repository. It must communicate, in under 60 seconds of reading, that you are a Senior-level designer. It should contain: (1) a one-paragraph framing statement about what this portfolio demonstrates, (2) a summary table of all five projects (see format below), and (3) a brief note on the course and methodology.

Root README summary table format:

| Project | Complexity | Key Patterns Demonstrated | Status |
|---|---|---|---|
| Document Collaboration System | Intermediate | Command (undo/redo), Role-Based Access Control, Versioning | Complete |
| Multi-Tenant Task Management | Senior | Observer (event system), Multi-tenancy isolation, Extensible entity model | Complete |
| Real-Time Auction Engine | Senior | Strategy (auction types), State Machine (lifecycle), Proxy bidding | Complete |
| Pluggable Data Pipeline | Staff | Visitor (metrics), Chain of Responsibility (error handling), DAG validation | Complete |
| Financial Transaction Ledger | Staff | Value Objects (Money), Immutability, Double-entry domain invariants | Complete |

**Per-project `README.md`**: This is what a hiring manager or interviewer reads after clicking into a project folder. It must contain: (1) a two-to-three sentence problem statement in plain English (not the brief from this course), (2) the design patterns applied and specifically where each pattern appears in the codebase (package and class name), (3) the three to five key design decisions made with one-sentence rationale for each, (4) known limitations and what you would do differently at scale, (5) instructions for running the code (`mvn test` or equivalent), and (6) a thumbnail or inline image of the class diagram.

**`docs/decision-log.md`**: Use the ADR (Architecture Decision Record) format for each decision. Each entry should be under 400 words and follow this structure: title, date, status (Accepted), context, decision, consequences (positive and negative). Number your ADRs sequentially: `ADR-001`, `ADR-002`, etc.

**`src/main/java/com/yourname/[project]/`**: Use a domain-driven package structure — not a layer-driven one. `com.yourname.auction.strategy` is better than `com.yourname.service.auction` because it groups by domain concept, not by technical layer. This signals mature package design to a reviewer.

**Diagram files**: Store diagrams as image exports (`.png` or `.svg`) alongside the source file if you used a diagram tool (`.puml` for PlantUML, `.drawio` for draw.io). Both source and export allows reviewers to inspect the source and also see a rendered version without tooling.

### Naming Conventions

- **Packages**: lowercase, domain-driven, no abbreviations (`com.yourname.auction.bidding`, not `com.yourname.auctn.bid`)
- **Classes**: UpperCamelCase, descriptive, domain-specific nouns and noun phrases (`ProxyBiddingEngine`, not `BidHelper`)
- **Interfaces**: No `I` prefix (prefer `AuctionStrategy` over `IAuctionStrategy`); use role-describing nouns where applicable (`Processor`, `Serializer`, `Validator`)
- **Test classes**: `[ClassUnderTest]Test.java` (e.g., `AuctionLifecycleTest.java`)
- **Diagram files**: `[diagram-type]-[subject].png` (e.g., `sequence-proxy-bidding.png`, `state-machine-auction-lifecycle.png`)
- **ADR files**: Either inline in `decision-log.md` or individual files named `adr-001-conflict-resolution-strategy.md`

### How to Present the Repository in an Interview

When asked "do you have any design examples you can share?", have the URL ready. Open the repository before the interview and navigate to one of the project READMEs so you are not starting from the repository root.

Your opening statement when referencing the portfolio should take this form: "I have a portfolio of five LLD capstone projects on GitHub. The one most relevant to what you described is [project name] — I designed it to specifically address [the core challenge that maps to the interview problem]. The key design decision I'm most proud of in that one is [decision], because [brief rationale]."

Then navigate directly to the class diagram (not the source code). Hiring managers and interviewers read diagrams faster than code. Lead with the diagram, use it to orient them on the structure, then point to specific packages in the source to ground the conversation.

Do not open the test directory first. Do not apologize for limitations. Do not say "this was just an exercise" — it was a complete design that you built and can defend.

### Git Commit Hygiene

A portfolio repository is read by people who are evaluating your engineering judgment. Your commit history is part of that evaluation.

**Write meaningful commit messages.** Each commit message should describe *why* the change was made, not just what changed. Prefer `Enforce double-entry balance constraint in Transaction factory method` over `update Transaction.java`.

**Commit in logical increments.** A single commit that adds 1,200 lines says nothing about your design process. Commit the domain model first, then the service layer, then tests, then the documentation artifacts. A commit history that tells the story of your design process is itself a portfolio artifact.

**Do not commit generated files, IDE config files, or build artifacts.** A `.gitignore` that includes `target/`, `.idea/`, `*.class`, and `.DS_Store` is the first thing a reviewer checks. Its absence suggests inexperience with collaborative development.

**Tag completed projects.** When you finish a project and are satisfied with the submission quality, create a git tag: `git tag v1.0-project-1`. This makes it easy to reference a known-good state and signals that you treat your portfolio as a maintained artifact, not an abandoned exercise.

---

## The Extensibility Test

Run this test after completing every capstone project. It is a structured self-evaluation that takes approximately 30 minutes. It asks you to apply five specific changes to your design and evaluate whether each change is isolated or invasive.

For each scenario, answer the test question by tracing through your actual design. Then evaluate against the pass/fail criteria.

---

### Test Scenario 1: Adding a New Core Type

**Test question**: Add a new variation of the project's central entity. For the Auction Engine, add a new auction type (e.g., an All-Pay Auction where all bidders pay their bid, not just the winner). For the Pipeline Framework, add a new stage type with enrichment-and-filter semantics. For the Financial Ledger, add a new account type for off-balance-sheet items.

**Pass**: Adding the new type requires creating exactly one new class (the implementation) plus any data/configuration entries (e.g., a new enum constant, a new factory case). No existing class requires modification to its logic. The test suite for existing types continues to pass without modification.

**Fail**: Adding the new type requires modifying any of the following: the service class that orchestrates behavior, the validator, the lifecycle manager, or any conditional chain (`if instanceOf`, `switch (type)`) that lists all existing types. If you find yourself opening more than two existing files to make this change, your design has a closed extension point.

---

### Test Scenario 2: Replacing the Storage Layer

**Test question**: Replace your current data storage mechanism with a completely different one. For example, replace the in-memory collections (Maps, Lists) used to store entities with a hypothetical database-backed repository. Specifically: which classes do you need to modify, and which classes can remain completely unchanged?

**Pass**: All business logic classes (services, domain entities, validators, pattern implementations) require zero modification. The only changes are in a thin persistence/repository layer that the business logic classes depend on through interfaces. You can demonstrate this by showing that every class in the `domain/` and `service/` packages has no import of any persistence mechanism.

**Fail**: Business logic methods directly reference `HashMap`, `ArrayList`, or other in-memory storage primitives in a way that is not hidden behind an interface. Switching storage requires touching business logic classes. Or there are no repository interfaces — the service directly manages collections.

---

### Test Scenario 3: Adding a Cross-Cutting Concern

**Test question**: Add structured logging to every operation in the system — specifically, log the method name, input parameters, and execution time for every public method in your service layer. You must do this without modifying any service method body.

**Pass**: Your design has a seam — either through an explicit Decorator pattern on service interfaces, a proxy mechanism, or a template method that wraps all service operations — that allows adding logging without touching service method implementations. You can point to the specific class or interface where the concern would be inserted.

**Fail**: Service methods are concrete implementations with no wrapping seam. To add logging to 10 service methods, you must add 10 log statements to 10 different methods. There is no single extension point. This indicates that cross-cutting concerns are not isolated in your design.

---

### Test Scenario 4: Adding a New Event or Notification Type

**Test question**: Add a new event type that did not exist before. For the Task Management Platform, add a `TASK_ARCHIVED` event. For the Document Collaboration System, add a `COLLABORATOR_REMOVED` event. For the Auction Engine, add an `AUCTION_EXTENDED` event.

**Pass**: Adding the new event requires: (1) defining the new event class or enum constant, (2) publishing it at the triggering point. No existing subscriber or event handler requires modification to accommodate the new event type — they simply ignore it. If new subscribers are needed for the new event, they are added without modification to existing subscribers.

**Fail**: Your event system has a central dispatcher with a conditional chain that enumerates all event types (`if event instanceof TaskAssigned` ... `else if event instanceof TaskCompleted` ...). Adding a new event type requires modifying this dispatcher. Or your Observer implementation uses event type checking (`if (event.getType() == EventType.TASK_ASSIGNED)`) inside the subscriber logic rather than type-specific subscription.

---

### Test Scenario 5: Swapping an Algorithm

**Test question**: Replace one algorithm with a different one that satisfies the same interface. For the Auction Engine, replace the Last-Write-Wins conflict policy with an operational transformation strategy. For the Financial Ledger, replace the full-history balance computation with a checkpointed computation. For the Pipeline Framework, replace the topological sort algorithm for DAG validation with an alternative cycle detection algorithm.

**Pass**: The algorithm lives behind an interface. Replacing it means creating a new implementation class and changing the injection point (constructor parameter, factory, or configuration). The rest of the system is unaffected — no callers know which concrete implementation is in use. You can demonstrate this by showing the interface and pointing to where the concrete implementation is injected.

**Fail**: The algorithm is embedded directly in the service or business logic class as a private method, not extracted behind an interface. To replace it, you must modify the class that uses it — which also means re-testing everything that class does. Or the algorithm is hardcoded with a conditional chain that tests for specific algorithm identifiers rather than delegating to a pluggable implementation.

---

### Interpreting Extensibility Test Results

| Scenarios Passed | Interpretation |
|---|---|
| 5 of 5 | Your design is genuinely extensible at all tested dimensions. Submit with confidence. |
| 3–4 of 5 | Good extensibility with specific gaps. Review the failing scenarios, identify the missing abstraction, and refactor before submitting. These are exactly the gaps that senior interviewers probe for. |
| 1–2 of 5 | The design's extensibility is largely coincidental rather than structural. Revisit Module 02 (OCP, DIP) and Module 04 (Strategy, Observer, Decorator) with specific attention to where variation points are identified and how they are abstracted. Return to the capstone after that review. |
| 0 of 5 | The design treats all variation points as fixed requirements. No extensibility is planned. This is a fundamental design-thinking gap, not a code quality issue. Complete additional case studies from Module 06 independently (without reading the solutions) to build the pattern-recognition muscle for variation points. |

---

## Common Mistakes in Capstone Submissions — Five Antipatterns to Avoid

These are the five most frequently observed antipatterns in capstone submissions from engineers who have completed the course material but have not internalized the design-thinking discipline. Each antipattern is recognizable, diagnosable, and fixable — but only if you know what to look for.

---

### Antipattern 1: The God Dispatcher

**What it looks like**: One class — often named `NotificationService`, `AuctionService`, `TaskProcessor`, or `EventHandler` — accumulates every orchestration responsibility. In the Document Collaboration System, it checks the user's role, validates the operation, applies the edit, records the version, updates the undo stack, checks for conflicts, and dispatches notifications. In the Auction Engine, it validates the bid, applies proxy bidding logic, determines if the auction state should transition, calculates the new minimum increment, and records the bid history. The class has 8–15 public methods and 300–500 lines of implementation code. Every new requirement makes this class larger.

**Why it happens**: When engineers start from the service layer (writing the service class first without a completed class diagram), they naturally add each new requirement as another method in the service. The service becomes the "place where things happen," and over time it becomes the place where *all* things happen. The design-first discipline exists specifically to prevent this.

**How to fix it**: Return to the class diagram. For each method in the God class, ask: "What is the single concept that makes this method belong here?" If you cannot answer, the method belongs somewhere else. Extract a `BidValidator` from the bid validation logic. Extract a `ProxyBiddingEngine` from the proxy bidding logic. Extract a `AuctionStateTransitionEngine` from the state transition logic. The `AuctionService` becomes a thin orchestrator that delegates to these focused collaborators. Each collaborator is testable in isolation.

---

### Antipattern 2: Primitive Obsession in the Domain Model

**What it looks like**: The domain model uses `String` for every identifier (`userId`, `auctionId`, `transactionId`), `String` for every categorized value (`status`, `currency`, `role`, `type`), `double` or `BigDecimal` for monetary amounts, and `long` for timestamps. The `Transaction` class has the signature: `void addLineItem(String accountId, BigDecimal amount, String currency, String direction)`. None of these values validate themselves. A caller can pass `"CREDI"` for direction or `-1.0` for amount and the compiler accepts it.

**Why it happens**: Primitive types are the path of least resistance in Java. Creating a `Money` value object or an `AccountId` class feels like overhead, especially when the design is being rushed. The engineer rationalizes that "the caller will validate inputs" — but this distributes validation responsibility across every call site, making it impossible to guarantee that invalid state cannot enter the domain.

**How to fix it**: Introduce value objects for all domain-meaningful primitives. `Money` encapsulates amount and currency with currency-aware equality and arithmetic. `AccountId` wraps a string with validation of the required format. `TransactionStatus` and `AuctionState` become enums with explicit valid transitions. `Direction` becomes an enum with `DEBIT` and `CREDIT` constants. Each value object validates its own construction — an invalid `Money` cannot be instantiated. The result is a domain model where invalid state is not a runtime error but a compile-time impossibility.

---

### Antipattern 3: Pattern Forcing

**What it looks like**: Patterns are applied to every class regardless of whether there is a genuine variation point. The Pipeline Framework uses Factory Method to create `Stage` objects even though there are only two stage types and neither is extensible. The Document Collaboration System uses Abstract Factory for `DocumentFactory` even though all documents are of the same type and the factory adds zero flexibility. The Financial Ledger uses a Decorator chain for `BalanceCalculator` even though there is exactly one calculation algorithm and no plans to add decoration. The decision log says: "I used the Strategy pattern for balance calculation because it allows different algorithms to be swapped in." But there is only one algorithm, and the interface was never designed to support a second one.

**Why it happens**: Engineers learning design patterns develop pattern recognition faster than pattern application judgment. They learn to spot variation points correctly but then apply patterns to variation points that exist only theoretically ("someday we might need multiple implementations"). The YAGNI principle from Module 02 exists to counteract this impulse.

**How to fix it**: Before applying any pattern, answer three questions: (1) Is there an existing variation point — something that currently varies, or a stated requirement that this must vary? (2) What is the concrete second implementation that the pattern would enable? (3) If I remove the pattern and inline the logic, how many classes would I need to modify when the variation arrives? If the answers are "no existing variation," "I don't know," and "just one," the pattern is premature. Use the simplest structure that satisfies the current requirements, and add the pattern when the second implementation actually arrives.

---

### Antipattern 4: Missing the Invariants

**What it looks like**: The design allows the domain model to enter states that the problem statement explicitly prohibits. In the Auction Engine, the `placeBid(Bid bid)` method does not check if the auction is in a valid state for bidding — a caller can invoke it on a `CLOSED` auction and the method processes the bid. In the Financial Ledger, the `Transaction.addLineItem()` method can be called after the transaction has been posted — there is no enforcement of the post-commit immutability. In the Task Management Platform, a task can be assigned to a user who is not a member of the workspace — there is no validation that the assignee belongs to the same tenant hierarchy. The tests for happy paths pass, but the domain invariants are either untested or unenforced.

**Why it happens**: Invariant enforcement is not visible in the happy path. Engineers who test only expected inputs never discover that invalid inputs are accepted. Domain invariants feel like "validation" — something that belongs in the controller or the service layer, not the domain model. This is the Anemic Domain Model antipattern applied to invariant enforcement.

**How to fix it**: Each domain class is responsible for enforcing its own invariants. `AuctionListing.placeBid()` checks the auction state before accepting the bid and throws `AuctionNotActiveException` if the state is invalid — not because a caller remembered to check, but because the method itself enforces the precondition. `Transaction.post()` flips an internal flag and every subsequent mutation attempt throws `TransactionAlreadyPostedException`. `Task.assignTo(User user)` verifies that the user is a member of the task's workspace before accepting the assignment. Write a test for every invariant violation scenario. If the test fails because the domain does not throw an exception, the invariant is not enforced.

---

### Antipattern 5: Synchronous Coupling Masquerading as Design

**What it looks like**: Every operation in the system is a direct method call chain with no seam for decoupling. A task status change calls the `ActivityLogger.log()` method directly, then calls `NotificationDispatcher.dispatch()` directly, then calls `WebhookService.fire()` directly. The `TaskService.updateStatus()` method has a chain of seven direct calls to seven different collaborators in sequence. The implementation compiles and the tests pass, but every collaborator is tightly coupled to the `TaskService`. Adding a new collaborator requires modifying `TaskService`. A slow `WebhookService` blocks the status update. A failing `NotificationDispatcher` causes the status update to fail even though the status was successfully changed. The design log says "I used the Observer pattern" — but the implementation uses direct method calls, not a subscription mechanism.

**Why it happens**: Direct method calls are the simplest way to make something happen, and they work correctly for unit tests where collaborators are mocked. The coupling is invisible in a test environment. The impact of the coupling only becomes apparent when thinking about failure isolation, performance, and extensibility under change — considerations that require deliberate design thinking, not just implementation.

**How to fix it**: Introduce an event publishing seam between the primary operation (status change) and the secondary effects (logging, notification, webhook). `TaskService.updateStatus()` publishes a `TaskStatusChangedEvent` to an `EventBus` or `EventPublisher`. `ActivityLogger`, `NotificationDispatcher`, and `WebhookService` are subscribers to that event. They register themselves with the event bus — `TaskService` does not know they exist. Adding a new secondary effect is a new subscriber, not a modification to `TaskService`. A failing subscriber is isolated. A slow subscriber can be made asynchronous without touching any other component. The key test: if removing `NotificationDispatcher` from the system requires zero changes to `TaskService`, your seam is correct.

---

## How These Projects Map to Real Interview Questions

Understanding which companies ask which categories of LLD problems — and what they are actually testing — transforms preparation from generic practice into targeted readiness.

---

### Project 1 — Document Collaboration System

**Companies that ask similar problems**: Google (Docs/Drive infrastructure teams), Microsoft (Office 365 / SharePoint engineering), Notion, Confluence (Atlassian), Quip (Salesforce), Dropbox Paper.

**Real interview variant phrasing**: "Design a collaborative document editor that supports real-time editing. Multiple users can edit the same document simultaneously. How do you handle conflicts?" (Google L4/L5 LLD round). "Design a version-controlled document system with undo/redo support." (Microsoft SDE2/Senior).

**What the capstone demonstrates directly**: Command pattern for undo/redo demonstrates knowledge of encapsulating operations as first-class objects — a direct signal that you understand behavioral pattern application, not just recognition. Role-based access control enforcement demonstrates SRP and DIP applied to a security concern. Version management demonstrates an understanding of immutable audit trails.

**Company-specific preparation tip**: Google interviews for this domain typically probe concurrency explicitly — they want to know what happens when two users type at the same character position simultaneously. They are testing whether you can articulate a conflict resolution strategy with clear semantics (last-write-wins vs. OT vs. CRDTs) and defend its tradeoffs. Have your answer to "what does your system do when two users edit the same line at the same time?" prepared in two sentences.

---

### Project 2 — Multi-Tenant Task Management Platform

**Companies that ask similar problems**: Atlassian (Jira), Linear, Asana, Monday.com, Salesforce (multi-tenant SaaS infrastructure), Zendesk.

**Real interview variant phrasing**: "Design a project management system like Jira." (Atlassian Senior/Staff LLD). "Design a multi-tenant SaaS platform with role-based access. How do you ensure tenant data isolation?" (Salesforce / enterprise SaaS companies). "Design a notification system that triggers on task events." (Asana, Monday.com).

**What the capstone demonstrates directly**: Multi-tenancy isolation design demonstrates Senior-level thinking about data boundaries and security-by-structure rather than security-by-convention. Observer-driven event system demonstrates understanding of decoupled, extensible event propagation. Custom fields without schema pollution demonstrates extensible entity modeling.

**Company-specific preparation tip**: Atlassian interviews frequently include a discussion of how the system behaves "at Jira scale" — they know their product and they want to see if you have thought about the implications of having 50,000 tasks in a single project. Prepare an answer for: "If a project has 100,000 tasks, how does your activity feed implementation perform?" This tests whether your design has separated the concern of data completeness from the concern of query performance.

---

### Project 3 — Real-Time Auction Engine

**Companies that ask similar problems**: eBay (core business), Amazon (third-party seller auctions), Uber (dynamic pricing shares DNA with auction mechanics), DoorDash and Instacart (bidding for delivery slots), ad-tech companies (RTB — real-time bidding for ad inventory).

**Real interview variant phrasing**: "Design an auction system for eBay. Focus on the bid placement logic and how you handle concurrent bids." (eBay Staff+ interview). "Design a real-time bidding system for programmatic advertising." (Google, Meta, The Trade Desk — ad-tech track). "Design a food delivery demand-pricing system." (DoorDash Senior/Staff).

**What the capstone demonstrates directly**: State machine design for auction lifecycle demonstrates that you can model complex object lifecycles without conditional soup. Strategy pattern for auction types demonstrates OCP applied to a real business problem where new types are a stated business requirement. Concurrency on bid placement demonstrates awareness of race conditions in domain logic, not just in infrastructure code.

**Company-specific preparation tip**: eBay and ad-tech RTB interviews always ask about throughput constraints at some point. You will be asked: "Your auction engine receives 10,000 bids per second. What breaks first?" You are not expected to have a distributed systems answer — the LLD round is about class-level design — but you are expected to identify where the concurrency bottleneck is in your design (the `placeBid()` method's critical section) and articulate whether your locking strategy is per-auction or system-wide.

---

### Project 4 — Pluggable Data Pipeline Framework

**Companies that ask similar problems**: Uber (data platform), LinkedIn (Kafka ecosystem and data infrastructure), Stripe (payment processing pipelines), Airbnb (data engineering platform), Databricks, Confluent, and any company with a platform engineering or developer experience team.

**Real interview variant phrasing**: "Design a data processing pipeline framework. Teams should be able to plug in custom processing stages without modifying the framework." (Uber, LinkedIn Staff-level interview). "Design an event processing system with pluggable handlers, error handling, and observability." (Stripe Platform team). "Design a workflow execution engine where steps can be composed dynamically." (Airbnb).

**What the capstone demonstrates directly**: Framework design thinking (protecting the caller from complexity) demonstrates Staff-level design maturity — the ability to design for multiple consumers, not just a single application. Visitor pattern for metrics demonstrates a sophisticated cross-cutting concern solution. DAG validation demonstrates algorithmic design integrated into the class model.

**Company-specific preparation tip**: Framework design interviews at Staff level assess whether you can articulate the API contract from the framework consumer's perspective before you explain the framework's internals. The question "what does a developer write to create a new stage?" should produce a 3-line answer. If your answer starts with "well, first they need to understand the lifecycle..." the abstraction is leaking. Prepare a 3-minute "hello world" walkthrough of your framework API that you can deliver without diagrams.

---

### Project 5 — Financial Transaction Ledger

**Companies that ask similar problems**: Stripe (core payments), Square (payment processing and merchant finance), Coinbase (crypto transaction ledger), Brex / Ramp (corporate cards and expense management), Plaid (financial data), Robinhood (trade ledger), any fintech company.

**Real interview variant phrasing**: "Design a financial ledger for a payments platform. Transactions must be immutable and the balance must always be derivable from history." (Stripe Senior/Staff interview). "Design a double-entry bookkeeping system. How do you ensure consistency?" (Brex, Ramp Staff interview). "Design a transaction history system that supports multi-currency balances and is tamper-evident." (Coinbase Staff interview).

**What the capstone demonstrates directly**: Money value object design demonstrates the rarest and most valued skill in fintech interviews: domain model purity. Immutability-after-commit demonstrates understanding of domain invariant enforcement as a structural concern, not a runtime check. Double-entry validation demonstrates that you can translate a real accounting rule into a class-level constraint.

**Company-specific preparation tip**: Stripe interviews, particularly for Staff level, are known for drilling on what happens in the failure case. "What happens if the database write of a POSTED transaction succeeds but the application crashes before the caller receives the response?" is a real Stripe LLD question. You are not expected to design a distributed two-phase commit — but you are expected to explain why the Transaction ID is idempotency-safe and how the ledger handles a duplicate POST attempt for the same transaction ID.

---

### Competency Coverage Map

The following table shows which key competencies are demonstrated by which capstone projects. Use this to identify whether your completed projects provide sufficient coverage for the roles you are targeting.

| Competency | P1 Doc | P2 Tasks | P3 Auction | P4 Pipeline | P5 Ledger |
|---|:---:|:---:|:---:|:---:|:---:|
| State machine design | | | **Primary** | Supporting | Supporting |
| Concurrency awareness | Supporting | | **Primary** | Supporting | |
| Extensibility via Strategy | Supporting | | **Primary** | Supporting | |
| Observer / event-driven design | Supporting | **Primary** | | Supporting | |
| Multi-tenancy isolation | | **Primary** | | | |
| Domain model purity (value objects, invariants) | Supporting | | | | **Primary** |
| Command pattern (undo/redo / operation history) | **Primary** | Supporting | | | |
| Framework design (cross-cutting concerns) | | | | **Primary** | |
| Role-based access control | **Primary** | Supporting | | | |
| Audit / immutability / tamper-evidence | Supporting | | | | **Primary** |
| Visitor pattern | | | | **Primary** | |
| Chain of Responsibility | | | Supporting | **Primary** | |
| Composite / DAG structures | | Supporting | | **Primary** | |
| Double-entry / financial domain | | | | | **Primary** |
| Multi-currency arithmetic | | | | | **Primary** |
| Pluggable serialization / format independence | | | | **Primary** | |

A candidate who completes all five projects can demonstrate every competency in the table. A candidate who completes Projects 1, 3, and 5 demonstrates a strong complementary set covering command-pattern history, state-machine lifecycle, concurrency, domain model purity, and financial domain modeling — a combination that maps well to fintech, marketplace, and infrastructure engineering roles.

---

*This module is the culmination of the course. The effort you invest here — in designing carefully, in writing honest decision logs, in running the extensibility test rather than skipping it — is precisely what separates candidates who interview well from candidates who interview at a Senior level.*

*Return to [`ROADMAP.md`](../ROADMAP.md) for the completion criteria, or proceed to [`10-interview-preparation/`](../10-interview-preparation/) when your capstone submissions are complete.*
