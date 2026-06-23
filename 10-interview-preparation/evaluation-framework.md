# LLD Interview Evaluation Framework

---

## Table of Contents

1. [What Interviewers Actually Evaluate](#section-1-what-interviewers-actually-evaluate)
2. [Scoring Rubric (1–5 Scale)](#section-2-scoring-rubric-15-scale)
3. [Common Failure Modes](#section-3-common-failure-modes)
4. [What Senior Engineers Do Differently](#section-4-what-senior-engineers-do-differently)
5. [Mock Interview Format](#section-5-mock-interview-format)

---

## SECTION 1: What Interviewers Actually Evaluate

### 1.1 The Hidden Rubric Most Candidates Don't Know

Every LLD interview has a visible goal — "design a parking lot," "model a notification system," "build the class structure for a ride-sharing app." But running in parallel, invisible to most candidates, is a second evaluation that determines whether an offer is made. Interviewers use an internal rubric they almost never share, and the vast majority of candidates prepare only for the surface layer of that rubric — the explicit layer — while leaving the more decisive implicit layer entirely unaddressed.

The explicit layer asks: does the design work? Can the candidate produce a class diagram and some code that models the problem? This layer is table stakes. Getting past it confirms that the candidate is not a complete beginner. But it does not, by itself, earn a hire at the Senior level. The interviewer is not just checking whether the code compiles or whether the design is technically valid. They are watching the candidate's process, the structure of their thought, and the quality of their decision-making in real time. The question they are actually answering is: "Would I trust this person to own a complex, evolving codebase, work through ambiguity without supervision, and make design decisions I can stand behind six months from now?"

The implicit layer of the rubric is concerned with the meta-skill beneath the technical one: can this person decompose a fuzzy, real-world problem — the kind that arrives without a complete specification, with conflicting stakeholder needs and time pressure — into a clean, maintainable, and extensible object model? Can they do that without being told how? Can they catch their own mistakes? Can they adjust course when new information arrives mid-design?

Most candidates only optimize for the explicit layer. They practice designing parking lots and vending machines and elevator systems until they can produce a serviceable class diagram quickly. What they do not practice is the implicit layer: how to handle a problem statement that is deliberately underspecified, how to verbalize a trade-off rather than just making it silently, how to recognize that a design decision made five minutes ago now conflicts with a new requirement and course-correct without derailing the session.

The implicit layer includes at least four sub-dimensions that are rarely discussed but consistently scored:

First, **how the candidate handles ambiguity**. Does the candidate ask questions that reveal domain understanding, or do they either plunge forward on incorrect assumptions or ask questions about infrastructure that have no bearing on the class design? A Senior engineer's clarifying questions are diagnostic — each question, if answered differently, would produce a meaningfully different class structure. This signals that the candidate understands which decisions are architecture-affecting and which are cosmetic.

Second, **whether the candidate drives the conversation or reacts to prompts**. A candidate who waits for the interviewer to say "now add a new payment method" before introducing an abstraction for payment processing has not demonstrated design foresight — they have demonstrated the ability to follow instructions. A candidate who introduces the payment processing abstraction unprompted, while saying "I'm doing this because I can already see that we'll have multiple payment methods, and I want that variation to be additive rather than surgical," has demonstrated precisely the kind of proactive thinking that Senior-level ownership requires.

Third, **whether design decisions are justified or merely made**. Many candidates draw a class diagram in silence and then describe it after the fact. The interviewer sees the output but not the reasoning. A candidate who names two concrete classes, then says "I'm going to extract an interface here rather than using one of these as the base class, because these two implementations are siblings — neither one is the canonical form — and I want the caller to know only the contract," has shown the interviewer their decision-making process. This is the difference between a designer and a transcriptionist.

Fourth, **the quality of recovery when something goes wrong**. Every non-trivial LLD design will hit a moment of friction mid-session — a requirement that conflicts with a design decision made earlier, a follow-up question that exposes a gap, a realization that a class is doing too much. How the candidate handles this moment is one of the highest-signal events in the entire interview. A candidate who goes silent, or who begins quietly modifying the design without acknowledging the issue, loses points. A candidate who says "I see the problem — I've modeled the fare calculation as a method on Ride, but what you've just described means the calculation algorithm can vary independently of the ride lifecycle. Let me pull that out into a separate interface" has demonstrated self-awareness, problem decomposition, and the willingness to course-correct under pressure — all simultaneously.

Understanding this dual-layer structure changes the nature of interview preparation. The preparation question shifts from "how do I design parking lots quickly?" to "how do I develop the habits of thought that Senior engineers apply to every design problem, regardless of the specific domain?"

### 1.2 What "Thinking Out Loud" Really Means to an Interviewer

"Think out loud" is the most commonly given and most commonly misunderstood piece of interview advice in the engineering industry. Every interviewer says it. Almost no candidate knows what it actually means.

The two most common misinterpretations produce two distinct failure modes, and both of them damage scores.

**Failure mode (a): the candidate goes silent and produces output.** The interviewer watches a candidate type, or draw, for five to eight minutes without speaking. At the end of the silence, a class diagram or a block of code appears. The output may be technically correct. But the interviewer has learned nothing about how it was produced — what alternatives were considered, what constraints shaped the structure, what the candidate is uncertain about. The interviewer cannot evaluate what they cannot observe, and the process is precisely what is being evaluated at the Senior level. This failure mode is particularly common among strong coders who have internalized the habit of silent concentration from competitive programming or production coding. In an interview, that habit works against them.

**Failure mode (b): the candidate narrates every action without reasoning.** The candidate says "now I'm creating a class called Ride. It has a field called rideId of type String. Now I'm adding a field called passengerId of type String. Now I'm adding a method called requestRide." The interviewer hears a description of what they can already read. This is not thinking out loud — it is transcription narration. It produces no signal because it reveals nothing that the code does not already show. Worse, it can actively obscure the candidate's intelligence if the underlying design choices are sophisticated but the narration is confined to surface description.

What interviewers actually want from "thinking out loud" is **reasoning narration** — the externalization of the decision-making process that produces the design, not the description of the design itself.

The difference is not subtle. Here are three concrete contrasts:

**Noise narration:** "I'm creating a class called `Ride` and a class called `Trip`."
**Reasoning narration:** "I'm separating `Ride` from `Trip` because the lifecycle of a booking request is distinct from the lifecycle of an active trip. A `Ride` represents the customer's intent and can be cancelled before a driver accepts it. A `Trip` only comes into existence once both parties are committed and movement has begun. Collapsing these into one class would mean a single object has two radically different state machines, which makes both the cancellation logic and the billing logic harder to reason about."

**Noise narration:** "I'm making `PaymentProcessor` an interface."
**Reasoning narration:** "I'm making `PaymentProcessor` an interface rather than an abstract class because I don't want to impose any shared implementation. The implementations — credit card, UPI, wallet — have completely different execution paths. What they share is a contract: they take a `PaymentRequest` and return a `PaymentResult`. The interface captures exactly that contract without imposing any implementation inheritance."

**Noise narration:** "I'm adding a `VehicleType` enum to `ParkingSpot`."
**Reasoning narration:** "I'm adding `VehicleType` to `ParkingSpot` rather than making `MotorcycleSpot` and `CarSpot` as separate subclasses, because the only thing that varies by vehicle type in this system is the size constraint on the spot. That is a data-level variation, not a behavioral one. Using an enum and a size attribute captures the variation with far less structural complexity than a subclass hierarchy would."

In each case, the reasoning narration tells the interviewer something that the code alone does not — it reveals that the candidate considered alternatives, has a principled reason for the choice, and understands the design consequences of the decision. That information is what the interviewer needs to score the implicit layer.

One additional dimension of thinking out loud that is particularly powerful at the Senior level: **surfacing trade-offs in real time**. When a candidate reaches a genuine fork in the road — two valid design choices that optimize for different properties — the ideal behavior is to name both, state the trade-off explicitly, and then choose one with a stated reason. "I could model this as a Template Method pattern using an abstract base class, or I could use Strategy with composition. Template Method gives me cleaner lifecycle control but forces my implementations into an inheritance hierarchy, which becomes awkward if an implementation needs to vary two behaviors independently. Strategy via composition is slightly more boilerplate but gives each implementation full independence. Given that we've already identified three payment methods and the interviewer mentioned a fourth coming, I'll use Strategy." This is not a prepared speech — it is the kind of in-the-moment reasoning that Senior engineers practice daily. Demonstrating it in an interview is one of the highest-signal events possible.

### 1.3 Seniority Level Expectations

LLD interviews are not graded on an absolute scale — they are graded relative to the level being interviewed. The same design that earns a strong hire for a Mid-level candidate might result in a no-hire for a Senior candidate being evaluated at a higher bar. Understanding where the bar sits for each level allows you to calibrate your effort and not leave Senior-level signals off the table.

---

**Junior Engineer (0–2 Years Experience)**

*What they are expected to do:* At the Junior level, the interviewer is looking for basic object-orientation — the ability to identify the main entities in a problem statement, assign fields and methods to them, and create simple class structures that are at least recognizable as modeling the problem domain. The code should follow basic naming conventions, have sensible field names, and demonstrate some awareness of encapsulation (fields not all public, at least some methods rather than purely public data structures). A Junior candidate does not need to demonstrate interfaces or design patterns to pass — they need to demonstrate that they can think in objects.

*What is forgiven:* Missing design patterns entirely is acceptable. Missing edge cases is acceptable if the happy path is sound. Some redundancy between classes is forgiven if the overall structure is reasonable. Not being able to articulate why a design decision was made is forgiven as long as the decision itself is defensible.

*What causes a fail:* A Junior candidate fails when there is no object-orientation whatsoever — when the candidate produces purely procedural thinking: a single class with all fields and all behavior, or a set of classes with no methods (pure data bags that are manipulated entirely from the outside). A fail also occurs when the candidate cannot define a single interface or abstract relationship of any kind, even when the problem makes one obviously necessary. Not being able to name any entity from the problem domain, or using names like `Helper`, `Manager`, `Util` without qualification for every class, signals that the candidate has not yet internalized object modeling as a design tool.

---

**Mid-Level Engineer (2–5 Years Experience)**

*What they are expected to do:* Mid-level candidates should demonstrate interface and abstract class usage — not just as syntax, but as a design tool. They should be able to recognize at least one or two common patterns in the problem (Strategy, Factory, and Observer are all fair game at this level) and apply them correctly, even if the justification is not perfectly articulated. They should be able to talk about cohesion and coupling in their own words — not necessarily using the exact terms, but demonstrating that they think about which responsibilities belong together and which should be kept separate. Their code should have no magic numbers, should use consistent naming, and should be readable without explanation.

*What is forgiven:* Imperfect pattern fit is forgiven — applying a pattern that is 80% correct for the problem is better than not applying any patterns. Some over-engineering is forgiven if the candidate can recognize it when asked. Missing thread safety concerns is generally forgiven unless the problem explicitly involves concurrency.

*What causes a fail:* Absence of interfaces in a design that has obvious variation points fails a Mid-level candidate. The inability to justify design decisions when asked — "I just thought this made sense" as the answer to "why did you use inheritance here?" — signals that the design was not deliberately thought through. The most common Mid-level fail is the God class: a single class with 15 methods that handles booking, matching, pricing, notification, payment, and rating all in one place. It signals that the candidate has not internalized single responsibility as a working principle, only as a vocabulary item.

---

**Senior Engineer (5–8 Years Experience)**

*What they are expected to do:* A Senior candidate is expected to drive the requirements phase — not wait to be asked clarifying questions but proactively identify the ambiguities that would affect the class design and surface them immediately. They use composition over inheritance correctly: inheritance is reserved for genuine is-a relationships that satisfy the Liskov Substitution Principle; composition is the default for behavioral variation. They apply patterns only where they solve real problems, not from a checklist. When asked "why Strategy here?" the answer should be tied to a specific design constraint, not to a pattern description. They handle state machines explicitly — if an entity has a lifecycle (Order goes from PLACED to CONFIRMED to SHIPPED to DELIVERED to RETURNED), the transitions are modeled as an enum or a state machine with guarded transitions, not as a free-form string field. They demonstrate extensibility unprompted: they introduce an interface before being asked "but what if there's a second payment method?" because they already anticipated the variation.

*What is forgiven:* Not covering every edge case spontaneously is forgiven as long as the candidate responds well when probed — they should be able to add edge case handling quickly and correctly when the interviewer asks "what happens when X fails?" Minor imperfections in pattern implementation under time pressure are forgiven.

*What causes a fail:* Jumping straight to code without a requirements phase is a Senior-level fail. It signals that the candidate cannot operate at a design level before an implementation level — which is precisely the skill being evaluated. Inability to discuss alternatives — "I didn't consider any other approach" — is a fail. A design that cannot accommodate new requirements without a full rewrite when the interviewer introduces one standard extension scenario (new payment method, new vehicle type, new notification channel) is a fail. These are not corner cases — they are the baseline scenarios that any Senior engineer should design for habitually.

---

**Staff Engineer (8+ Years Experience)**

*What they are expected to do:* Staff candidates are expected to question the problem framing before accepting it. If the problem statement has an implicit assumption that is worth surfacing ("you've described this as a single-tenant system — is that a constraint or a default assumption?"), the Staff candidate surfaces it. They identify cross-cutting concerns proactively — audit trails, observability hooks (where would you add a metric? Where does a log line need to appear?), rate limiting surfaces, and failure mode handling at design time. They think about who will consume this system: "If this is an internal service used by three different teams, what happens if Team A's use case changes and Team B's does not? How does the interface versioning work?" They design for multiple consumers without being asked. They discuss operational concerns: configuration externalization, graceful degradation, what happens when a dependency is down.

*What is forgiven:* Staff candidates may whiteboard only without implementing deep code, and that is acceptable if the reasoning is clearly sound and the design is complete at the structural level. Asking fewer clarifying questions is acceptable if the design choices implicitly encode the right assumptions and the candidate can explain them when probed.

*What causes a fail:* No systems thinking is the primary Staff-level fail. If the candidate treats the LLD interview as a purely academic coding exercise — produces a clean class diagram, writes some methods, calls it done — without any evidence of thinking about team ownership, operational concerns, or failure modes, that is a fail at the Staff level. Solving only the stated problem without addressing the broader system context that any experienced engineer would recognize is present is another fail. A Staff candidate who cannot say "here's what would break in production and here's how the design mitigates it" has not yet demonstrated Staff-level judgment.

### 1.4 The 8 Dimensions Interviewers Score

Every LLD interview is evaluated across eight distinct dimensions. Individual interviewers may not use these exact names in their scorecards, but every piece of feedback written after an LLD interview maps to one or more of these axes. Knowing what each dimension tests allows you to practice each one deliberately, identify your specific gaps, and distribute your preparation effort where it will have the highest return.

---

**Dimension 1: Requirements Clarification**

This dimension evaluates the discipline of asking the right questions before committing to a design. It is not about asking as many questions as possible — excessive questioning signals indecision or an inability to operate under ambiguity. It is specifically about asking questions whose answers would change the class structure, the entity relationships, or the identified variation points in a meaningful way.

A strong performance on this dimension demonstrates that the candidate understands which design decisions are downstream of which requirements. When they ask "can a driver have multiple active vehicles, or only one at a time?", they are implicitly showing that they know this answer affects whether the `Driver`-to-`Vehicle` relationship is one-to-one or one-to-many, which in turn affects the assignment algorithm and the availability logic. The question is design-diagnostic, not curiosity-driven.

---

**Dimension 2: Abstraction Quality**

This dimension evaluates the ability to identify the right level at which to hide information. An abstraction is not just an interface placed over a class — it is a deliberate decision to expose a behavioral contract while hiding the details of fulfillment. A `MySQLOrderRepository` exposed directly to an `OrderService` is not an abstraction; it is a renamed class. An `OrderRepository` interface that defines `save`, `findById`, and `findByCustomerId` — which could be backed by MySQL, Postgres, DynamoDB, or an in-memory map — is a genuine abstraction.

Strong abstraction quality means identifying the seams between components — the boundaries across which the most likely variations will occur — and placing interfaces at those seams. It also means knowing when not to abstract: a helper method that converts a date format is not a variation point and does not need an interface.

---

**Dimension 3: Class Design and Relationships**

This dimension evaluates how entities are structured and how they relate to one another. It covers the choice between inheritance and composition: is-a relationships (a `SavingsAccount` is a `BankAccount`, and Liskov's principle is satisfied) versus has-a relationships (a `Driver` has a `Vehicle`, and the vehicle exists independently of the driver). It covers relationship directionality — which direction an association points, and whether bidirectional associations are genuinely needed or introduce unnecessary coupling. It covers aggregation versus composition at the lifecycle level: a `Ride` has `Waypoints` via composition (waypoints do not exist without the ride) but has a `Driver` via aggregation (the driver exists before and after the ride). It covers field and method visibility: fields are private by default, methods are public only when they form the external contract.

---

**Dimension 4: Pattern Selection Appropriateness**

This dimension evaluates whether design patterns are applied because they fit the problem, not because the candidate knows them. A pattern is appropriate when it solves a structural problem that is genuinely present in the design. A pattern is inappropriate when the problem it is meant to solve does not exist yet, or when the problem is simpler than the pattern's solution requires.

This dimension specifically penalizes two opposite failure modes: not using any patterns when the problem clearly calls for them (a payment system with multiple payment methods and no Strategy abstraction) and over-applying patterns to problems that do not need them (using a Factory for a class with no variant implementations, or an Observer for a one-to-one event notification with no subscription management needed).

---

**Dimension 5: Code Quality**

This dimension evaluates the craft-level properties of the code produced during the session. Strong code quality means that every identifier — class name, method name, variable name, constant name — reveals its intent to a reader who has never seen the codebase before. It means methods do exactly one thing, are short enough to be understood in a single read, and have names that match the behavior they contain. It means constants are named and placed where they are semantically accessible. It means access modifiers are chosen deliberately. It means exception handling distinguishes between recoverable and unrecoverable conditions rather than swallowing all exceptions silently. At the Senior level, code quality in an interview should be close to what the candidate would commit to a production codebase on a well-run team.

---

**Dimension 6: Extensibility Demonstration**

This dimension evaluates the ability to show that the design can accommodate new requirements without structural surgery. The Open/Closed Principle — software entities should be open for extension but closed for modification — is the organizing concept here, but this dimension goes beyond naming the principle. It requires demonstrating it concretely: when the interviewer says "now add a loyalty discount to this system," a candidate who scores a 5 on this dimension shows which new class to add, names the interface it implements, and explains that the existing `OrderService` and `CheckoutController` do not change. The change is additive, not surgical.

This is one of the highest-differentiation dimensions between Mid-level and Senior candidates. Mid-level candidates solve today's problem. Senior candidates solve today's problem in a way that does not make tomorrow's problem harder.

---

**Dimension 7: Edge Case Handling**

This dimension evaluates the systematic identification and handling of scenarios that deviate from the happy path. The bar is not exhaustive coverage of every possible failure scenario — that would require hours, not 45 minutes. The bar is demonstrating a systematic approach to asking "what can go wrong here?" and handling the most important cases explicitly in code.

Strong edge case handling includes proactively identifying null inputs at public method boundaries, guarding against invalid state transitions (a booking that is already `CANCELLED` cannot be `CONFIRMED`), recognizing concurrent access scenarios (two users booking the last available seat simultaneously), and thinking through partial failure recovery (what is the system's state if the payment charge succeeds but the order confirmation write to the database fails?).

---

**Dimension 8: Communication Clarity**

This dimension evaluates the ability to make the design legible to the interviewer in real time, throughout the session. It is distinct from presentation skill — the interviewer is not looking for a polished talk. They are looking for a running explanation that keeps them oriented at all times, so they can follow the reasoning and give feedback without confusion.

Strong communication clarity means that every new entity introduced during the session is immediately explained in one sentence. It means the vocabulary used verbally is consistent with the vocabulary in the code — if the class is called `RideMatcher`, it is called `RideMatcher` in conversation, not "the matching thing" or "the algorithm." It means trade-offs are called out when they are made, not reconstructed after the fact when the interviewer asks. And it means that the interviewer never has to ask "what does this class do?" because that question was answered the moment the class was introduced.

---

## SECTION 2: Scoring Rubric (1–5 Scale)

The following rubric defines what each score level looks like for each of the eight dimensions. Scores 1, 3, and 5 are defined as anchors; scores 2 and 4 represent performance between those anchors. Use this rubric for honest self-evaluation after mock interviews. The descriptions are behavioral and specific — they describe observable actions and outputs, not personality traits or general competence levels.

---

### Dimension 1: Requirements Clarification

**Score 1 — Does Not Demonstrate**

The candidate dives directly into design the moment the problem statement ends. No clarifying questions are asked. When the interviewer later introduces a follow-up requirement that a standard domain question would have uncovered — "what if a vehicle spans two parking spaces?" — the candidate's design requires partial reconstruction. When asked "what did you assume about X?", the candidate either cannot articulate the assumption or reveals an assumption that was clearly wrong in context. The design is optimized for a problem that was never confirmed. The interviewer scores this dimension at 1 because there is no evidence that the candidate distinguishes between "what was stated" and "what I need to know before designing."

**Score 3 — Adequately Demonstrates**

The candidate asks one or two questions before starting, but the questions are surface-level or misdirected. Common Score 3 questions include "Is this for mobile or web?" (irrelevant to class structure), "Should I worry about the database?" (a system design concern, not an LLD concern), and "How many users will this support?" (scale does not affect the class model unless concurrency is explicitly in scope). The questions reduce some ambiguity but leave the structural design questions unasked — does a driver have one vehicle or multiple? Can a ride be shared by multiple passengers? Can a parking space accommodate different vehicle sizes? These answers would change the entity model, and they are not surfaced. The candidate occasionally proceeds on an unspoken assumption.

**Score 5 — Strongly Demonstrates**

The candidate asks four to six targeted questions before picking up a marker or opening the IDE. Each question, if answered differently, would produce a meaningfully different class structure. After asking, the candidate explicitly states their scope: "I'll design for: single building, three vehicle types — motorcycle, car, bus — flat pricing, real-time occupancy tracking for the system admin. I'm treating VIP zones and dynamic pricing as out of scope for this session, though I'll note where they could plug in."

For a parking lot problem, Score 5 questions include: "How many vehicle types do we need to support, and do different types require different space sizes?" (affects `VehicleType` enum and `SpotAssignmentStrategy`), "Is pricing flat-rate by duration or does it vary by vehicle type, zone, or time of day?" (determines whether `PricingPolicy` needs to be an interface), "Who are the actors — is there an attendant role or is it fully automated?" (affects whether an `Attendant` entity exists and what operations they own), "Can a single ticket be used for multiple entry and exit cycles, or is each entry a fresh ticket?" (determines whether `Ticket` needs a state machine or a simple timestamp), and "Do we need to track occupancy in real time for an admin dashboard, or just for entry/exit decisions?" (affects whether an `OccupancyMonitor` is a separate concern).

---

### Dimension 2: Abstraction Quality

**Score 1 — Does Not Demonstrate**

The candidate's design contains no interfaces. All class dependencies are on concrete types. The client classes construct their own dependencies using `new`. The design is not only untestable — it is structurally monolithic.

```java
// Score 1 — No abstraction
public class CreditCardPayment {
    public void pay(double amount) {
        // calls credit card API directly
    }
}

public class UpiPayment {
    public void pay(double amount) {
        // same signature, no shared abstraction
        // duplicated structure, no polymorphism possible
    }
}

public class CheckoutService {
    private CreditCardPayment creditCard = new CreditCardPayment();
    private UpiPayment upi = new UpiPayment();

    public void processPayment(String method, double amount) {
        if (method.equals("CREDIT_CARD")) {
            creditCard.pay(amount);
        } else if (method.equals("UPI")) {
            upi.pay(amount);
        }
        // adding a third payment method requires modifying this class
    }
}
```

The consequence is clear and serious: adding any new payment method requires opening `CheckoutService` and modifying it. Every addition increases the risk of breaking existing payment paths.

**Score 3 — Adequately Demonstrates**

The candidate introduces interfaces at one or two seams but not all of them. The most commonly abstracted seam at this score level is storage — the repository layer is behind an interface. But the notification mechanism, the payment processor, or the pricing logic remains as a concrete class dependency. Alternatively, the candidate introduces interfaces but names them in implementation-referential terms (`MySQLOrderRepositoryInterface`) rather than contract-first terms (`OrderRepository`), revealing that the abstraction is mechanical rather than conceptual.

**Score 5 — Strongly Demonstrates**

Every significant collaborator is hidden behind an interface that represents a behavioral contract named from the caller's perspective. The interfaces are narrow — they capture exactly the capability the caller needs, nothing more. Dependencies are injected, not constructed. The candidate can explain what each abstraction hides and why that hidden knowledge is volatile.

```java
public interface PaymentProcessor {
    PaymentResult process(PaymentRequest request);
    boolean supports(PaymentMethod method);
}

public class CreditCardProcessor implements PaymentProcessor {
    @Override
    public PaymentResult process(PaymentRequest request) {
        // credit card-specific charge logic
        return PaymentResult.success(generateTransactionId());
    }

    @Override
    public boolean supports(PaymentMethod method) {
        return method == PaymentMethod.CREDIT_CARD;
    }
}

public class UpiProcessor implements PaymentProcessor {
    @Override
    public PaymentResult process(PaymentRequest request) {
        // UPI-specific payment initiation logic
        return PaymentResult.success(generateTransactionId());
    }

    @Override
    public boolean supports(PaymentMethod method) {
        return method == PaymentMethod.UPI;
    }
}

public class CheckoutService {
    private final List<PaymentProcessor> processors;

    public CheckoutService(List<PaymentProcessor> processors) {
        this.processors = processors;
    }

    public PaymentResult processPayment(PaymentRequest request) {
        return processors.stream()
            .filter(p -> p.supports(request.getMethod()))
            .findFirst()
            .orElseThrow(() -> new UnsupportedPaymentMethodException(
                request.getMethod()))
            .process(request);
    }
}
```

The candidate explains: "I've extracted `PaymentProcessor` as an interface with `supports` and `process` because those are the only two things `CheckoutService` needs to know. It does not need to know which processor is in use, how the charge is executed, or what API is called. Adding a new payment method means adding a new class and registering it — `CheckoutService` is not touched."

---

### Dimension 3: Class Design and Relationships

**Score 1 — Does Not Demonstrate**

The candidate uses inheritance as the default relationship for everything. Behavioral variation is modeled as type variation. Composition is absent. The design quickly produces impossible hierarchies because Java does not support multiple inheritance — `PremiumEmailUser` cannot extend both `PremiumUser` and `EmailUser`. Fields are made public for convenience. Methods are grouped arbitrarily: some classes have 15 methods spanning unrelated concerns while others are empty data containers with no behavior.

The most common Score 1 pattern is deep inheritance for what should be compositional variation:

```java
// Score 1 — Inheritance abuse
public class User { }
public class PremiumUser extends User { }
public class NotificationUser extends User { }
public class PremiumNotificationUser extends ??? // impossible in Java
```

The design collapses as soon as two behavioral dimensions need to co-exist in the same object.

**Score 3 — Adequately Demonstrates**

The candidate gets the main entities right and avoids the most egregious inheritance mistakes. Composition is used in at least one place where it is clearly the right choice. However, relationships are not fully thought through: bidirectional associations are created by habit without considering whether both directions are needed, leading to circular dependencies that make testing harder. Some classes are still too large. Visibility is inconsistently applied — some fields are public, some private, with no discernible guiding principle.

**Score 5 — Strongly Demonstrates**

Each class is designed with a single named responsibility. Composition is the default; inheritance is reserved for genuine is-a relationships where the subclass satisfies the Liskov Substitution Principle. The candidate makes conscious, articulated choices between aggregation and composition at the lifecycle level.

```java
// Score 5 — Conscious relationship design
public interface SchedulingPolicy {
    boolean isReadyToDispatch(RideRequest request);
    LocalDateTime getScheduledTime(RideRequest request);
}

public class ImmediateSchedulingPolicy implements SchedulingPolicy {
    @Override
    public boolean isReadyToDispatch(RideRequest request) {
        return true; // always ready immediately
    }

    @Override
    public LocalDateTime getScheduledTime(RideRequest request) {
        return LocalDateTime.now();
    }
}

public class AdvanceSchedulingPolicy implements SchedulingPolicy {
    @Override
    public boolean isReadyToDispatch(RideRequest request) {
        return LocalDateTime.now().isAfter(
            request.getRequestedPickupTime().minusMinutes(10));
    }

    @Override
    public LocalDateTime getScheduledTime(RideRequest request) {
        return request.getRequestedPickupTime();
    }
}

public class RideRequest {
    private final String requestId;
    private final String passengerId;
    private final Location pickupLocation;
    private final Location dropLocation;
    private final SchedulingPolicy schedulingPolicy; // composed, not inherited

    // RideRequest does not extend ScheduledRideRequest or ImmediateRideRequest
    // Variation is in the policy, not the type
}
```

The candidate articulates: "I'm composing `SchedulingPolicy` because a ride request is always a ride request — the scheduling behavior is a property of how it is processed, not of what it fundamentally is. If I used inheritance, adding a third scheduling type would require a new subclass, and a ride that needs to switch scheduling behavior at runtime would be impossible to model. Composition lets me swap the policy without changing the request object."

---

### Dimension 4: Pattern Selection Appropriateness

**Score 1 — Does Not Demonstrate**

The candidate either uses no patterns (designing everything procedurally with direct method calls and switch statements) or names patterns without implementing them. Saying "I would use the Factory pattern here" and then constructing objects with `new` throughout the code is a Score 1 moment. Applying a pattern reflexively — "it has multiple types, therefore Factory" — when a factory adds no value because the caller already knows the type is also Score 1 for pattern judgment.

**Score 3 — Adequately Demonstrates**

The candidate applies one or two patterns correctly and can describe their purpose. However, selection is sometimes mechanical — the problem mentions "multiple notification types" and the candidate applies Observer without first establishing whether there is actually a publish-subscribe relationship or just a direct one-to-one call that happens to be configured at runtime. The pattern fits approximately but the candidate cannot explain why this problem specifically required it rather than a simpler alternative.

**Score 5 — Strongly Demonstrates**

Patterns emerge from the requirements, not from a checklist. The candidate identifies the structural problem first, then designs a solution, and the result happens to be a known pattern — which may or may not be named explicitly. When a name is used, it is used accurately and the implementation is complete. The candidate can explain the "why this pattern, not that one" choice for the specific problem.

```java
// Problem: fare calculation algorithm must change independently of trip lifecycle
// Score 5 — Strategy applied because the problem demands it

public interface FareStrategy {
    Money calculate(TripDuration duration, TripDistance distance);
}

public class StandardFareStrategy implements FareStrategy {
    private static final Money BASE_FARE = Money.of(2.00, Currency.USD);
    private static final double PER_MINUTE_RATE = 0.25;
    private static final double PER_KM_RATE = 1.50;

    @Override
    public Money calculate(TripDuration duration, TripDistance distance) {
        double variable = duration.inMinutes() * PER_MINUTE_RATE
            + distance.inKilometres() * PER_KM_RATE;
        return BASE_FARE.add(Money.of(variable, Currency.USD));
    }
}

public class SurgeFareStrategy implements FareStrategy {
    private final FareStrategy baseStrategy;
    private final double surgeMultiplier;

    public SurgeFareStrategy(FareStrategy baseStrategy, double surgeMultiplier) {
        this.baseStrategy = baseStrategy;
        this.surgeMultiplier = surgeMultiplier;
    }

    @Override
    public Money calculate(TripDuration duration, TripDistance distance) {
        return baseStrategy.calculate(duration, distance).multiply(surgeMultiplier);
    }
}

public class Trip {
    private final String tripId;
    private final String driverId;
    private final String passengerId;
    private final FareStrategy fareStrategy; // injected, not hardcoded

    public Money calculateFare(TripDuration duration, TripDistance distance) {
        return fareStrategy.calculate(duration, distance);
    }
}
```

The candidate explains: "I'm using Strategy here because fare calculation rules change independently of the trip lifecycle. The product team will want to experiment with pricing algorithms — flat rates, per-kilometer rates, surge multipliers, subscription discounts. If I embed the calculation in `Trip`, every pricing experiment requires touching the core trip entity and re-testing everything that depends on it. With `FareStrategy` as a separate interface, a new pricing algorithm is a new class. The `Trip` entity is never touched. I can even compose strategies — `SurgeFareStrategy` decorates any base strategy, which makes surge a cross-cutting modifier rather than a separate calculation branch."

---

### Dimension 5: Code Quality

**Score 1 — Does Not Demonstrate**

Magic numbers pervade the code. Single-letter variable names appear outside of loop counters. Method names are generic verbs without domain context. Classes are named `Manager`, `Handler`, or `Processor` without qualification. Exception handling swallows errors silently. Fields are public. No constants are defined.

```java
// Score 1 — Poor code quality
public double calc(int t, int d) {
    return t * 1.5 + d * 0.25 + 2.0;
    // t = minutes? seconds? What is 1.5? What is 0.25? What is 2.0?
    // A reader has no idea what this computes
}
```

The cost of this code is disproportionate to the time saved writing it. Every reader, including the candidate themselves in six months, must reverse-engineer the meaning of every numeric literal and single-letter variable.

**Score 3 — Adequately Demonstrates**

Names are generally meaningful but occasionally vague. Magic numbers are mostly eliminated. Exception handling acknowledges errors but does not distinguish between recoverable situations (transient network failure → retry) and unrecoverable ones (invalid input → throw immediately). Methods are sometimes too long — a method that validates input, calculates a result, and persists it to storage is doing three things that should be separated. Constants are defined locally when they belong in a shared location.

**Score 5 — Strongly Demonstrates**

Code reads as if written by a careful author for a long-lived codebase. Every name is the most intention-revealing name available in the domain vocabulary. Constants are named and placed where they are most semantically accessible. Methods are short, cohesive, and do exactly what their name says. Access modifiers are intentional. Exception handling makes the distinction between recoverable and unrecoverable explicit.

```java
public class FareCalculator {
    private static final Money BASE_FARE = Money.of(2.00, Currency.USD);
    private static final double PER_MINUTE_RATE = 1.50;
    private static final double PER_KM_RATE = 0.25;

    public Money calculate(TripDuration duration, TripDistance distance) {
        Objects.requireNonNull(duration, "TripDuration must not be null");
        Objects.requireNonNull(distance, "TripDistance must not be null");

        double variableFare = duration.inMinutes() * PER_MINUTE_RATE
            + distance.inKilometres() * PER_KM_RATE;

        return BASE_FARE.add(Money.of(variableFare, Currency.USD));
    }
}
```

Every element in this code communicates purpose: the constants are named for what they represent (not `1.50` but `PER_MINUTE_RATE`), the method signature uses domain-typed parameters (not `int` and `double` but `TripDuration` and `TripDistance`), the null guards are specific about which argument is invalid, and the calculation is readable as prose.

---

### Dimension 6: Extensibility Demonstration

**Score 1 — Does Not Demonstrate**

The design is fully closed to extension. Adding a new vehicle type to a parking lot system requires modifying the core `ParkingLot` class. Adding a new notification channel requires adding a new branch to a switch statement. When the interviewer asks "how would you add scheduled rides to this design?", the candidate looks at their code and says "I'd add a boolean `isScheduled` field to `Ride` and some conditional logic in the dispatcher." This is not extension — it is modification that makes the class harder to understand and more fragile over time.

**Score 3 — Adequately Demonstrates**

The candidate has made one extension point explicit (usually the most obvious one — payment processing or notification channel) but has not applied extensibility thinking across the whole design. When a second extension scenario is introduced, the candidate realizes their design requires modification and proposes an in-place refactor. The one extension point they did identify is correctly implemented but not fully walked through — they show what to implement but do not explain what stays unchanged.

**Score 5 — Strongly Demonstrates**

Extension points are introduced proactively, before the interviewer asks for them. The candidate can walk through an extension scenario and name explicitly: which new class gets created, which existing interface it implements, and which existing classes require zero modification. The demonstration is code-level, not abstract.

```java
// The extension scenario: "Add surge pricing during peak hours"
// Score 5 — Extension is additive, nothing existing changes

// Already in the design:
public interface FareStrategy {
    Money calculate(TripDuration duration, TripDistance distance);
}

// New class — the only artifact required:
public class PeakHourFareStrategy implements FareStrategy {
    private static final double PEAK_MULTIPLIER = 1.8;
    private final FareStrategy baseStrategy;
    private final PeakHourChecker peakHourChecker;

    public PeakHourFareStrategy(FareStrategy baseStrategy,
                                 PeakHourChecker peakHourChecker) {
        this.baseStrategy = baseStrategy;
        this.peakHourChecker = peakHourChecker;
    }

    @Override
    public Money calculate(TripDuration duration, TripDistance distance) {
        Money baseFare = baseStrategy.calculate(duration, distance);
        return peakHourChecker.isPeakHour(LocalTime.now())
            ? baseFare.multiply(PEAK_MULTIPLIER)
            : baseFare;
    }
}
```

The candidate walks through: "To add peak-hour pricing, I implement `FareStrategy` in `PeakHourFareStrategy`. It decorates the existing standard strategy. The `Trip` class does not change. `TripService` does not change. `FareCalculator` does not change. The only decision is wiring: when should the `PeakHourFareStrategy` be injected instead of the standard one? That is a configuration decision, not a class structure decision. The design handles the extension with one new class and one configuration change."

---

### Dimension 7: Edge Case Handling

**Score 1 — Does Not Demonstrate**

Only the happy path exists. When the interviewer asks "what happens if the payment fails after the ride completes?", the candidate looks at the code and has no answer. Null inputs are not guarded. State machine violations are not protected — calling `completeRide()` on a ride that has already been completed raises no exception and produces undefined behavior. No concurrent access scenarios are identified or handled.

**Score 3 — Adequately Demonstrates**

The candidate handles the most obvious failure scenario — the one that appears in most textbook descriptions of the problem — but misses the subtler ones. Null checks appear at the top-level entry point but not at internal boundaries. The candidate acknowledges concurrency as a concern when the interviewer prompts "what about concurrent booking?" but cannot connect the acknowledgment to a specific design change. Edge cases are mentioned verbally but not implemented: "I would handle the case where the driver is unavailable" without showing any code that represents that handling.

**Score 5 — Strongly Demonstrates**

Edge cases are identified proactively, before being asked. The candidate systematically thinks through the failure modes at each boundary and handles them explicitly in code. Return types use `Optional<T>` where a result may legitimately not exist. Custom exceptions carry descriptive messages that include the relevant context (which entity was not found, what the current state was, what the expected state was). Guard clauses appear at method entry points. Cascading failures are thought through: "What if the payment succeeds but writing the trip record to the database fails? Now the customer was charged for a trip that doesn't exist in the system. Let me show how the design handles that rollback."

```java
public class RideBookingService {

    private final DriverRepository driverRepository;
    private final RideRepository rideRepository;
    private final RideEventPublisher eventPublisher;

    public RideConfirmation bookRide(RideRequest request) {
        Objects.requireNonNull(request, "RideRequest must not be null");
        Objects.requireNonNull(request.getPassengerId(),
            "Passenger ID must not be null");
        Objects.requireNonNull(request.getPickupLocation(),
            "Pickup location must not be null");

        // Guard: cannot book a ride that has already been processed
        if (request.getStatus() != RideRequestStatus.PENDING) {
            throw new InvalidRideRequestStateException(
                request.getRequestId(),
                request.getStatus(),
                RideRequestStatus.PENDING);
        }

        // May legitimately return empty if no drivers are available
        Optional<Driver> availableDriver = driverRepository
            .findNearestAvailable(request.getPickupLocation());

        if (availableDriver.isEmpty()) {
            request.markNoDriverAvailable();
            rideRepository.save(request);
            throw new NoDriverAvailableException(request.getPickupLocation());
        }

        Driver driver = availableDriver.get();
        // Optimistic lock: driver may have been assigned between find and assign
        try {
            driver.assignToRide(request.getRequestId());
            driverRepository.save(driver);
        } catch (DriverAlreadyAssignedException ex) {
            // Retry once with a different driver, then fail
            throw new BookingConflictException(request.getRequestId(), ex);
        }

        Ride ride = Ride.create(request, driver);
        rideRepository.save(ride);
        eventPublisher.publish(new RideBookedEvent(ride));

        return RideConfirmation.from(ride);
    }
}
```

The candidate explains each guard: "The null checks at entry prevent NPEs deep in the call stack where the context is lost. The status guard prevents double-booking if the same request is submitted twice. The `Optional` return on driver availability forces the caller to handle the no-driver case explicitly — it cannot accidentally proceed as if a driver was found. The `DriverAlreadyAssignedException` handles the race condition where two booking requests compete for the same driver."

---

### Dimension 8: Communication Clarity

**Score 1 — Does Not Demonstrate**

The interviewer must ask "what are you doing?" or "what does this class do?" multiple times during the session. The candidate codes in silence for stretches of five minutes or more. Class names appear in the code without being introduced verbally. The reasoning behind design decisions is never stated — the candidate makes choices but does not explain them. When asked "why did you design it this way?", the response is a re-description of the design rather than a justification.

**Score 3 — Adequately Demonstrates**

The candidate narrates most of what they are doing but not always why. "I'm adding a `RideMatcher` interface" — the interviewer hears the name, but not why it is an interface rather than a class, and not what it enables. Communication is mostly continuous but breaks down during complex implementation sections. Terminology is mostly consistent but slips occasionally (the interface is called `RideMatcher` in code and "the dispatcher" in speech). The candidate can answer direct questions about the design but does not volunteer explanations proactively when making non-obvious decisions.

**Score 5 — Strongly Demonstrates**

The interviewer is always oriented. Every new entity is introduced in one sentence before the code for it is written. Trade-offs are called out as they are made, not reconstructed afterward. The vocabulary used in speech matches the vocabulary in code with no drift. The candidate checks for comprehension at natural pause points: "Does that relationship make sense before I move to the matching logic?" and self-corrects when they notice a gap: "I realize I haven't explained why `Location` is a value object rather than an entity — let me cover that because it matters for how it is shared across the design."

The candidate speaks in a structured unit for every significant decision: state the intent, implement, reflect. "I'm going to model `TripState` as an enum with explicit transition guards rather than a plain `String` field. Here is the enum — `REQUESTED`, `DRIVER_ASSIGNED`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED` — and here is the transition guard method: `transitionTo` throws `InvalidStateTransitionException` if the requested transition is not in the allowed set. The benefit is that invalid state transitions fail at the point where they are attempted, with a clear error message, rather than silently producing corrupted state downstream."

---

## SECTION 3: Common Failure Modes

### 3.1 "Jumping to Code" Failure

This is the single most common and most damaging mistake made by candidates in LLD interviews. It occurs more frequently at the Senior level than at the Junior level, which makes it more penalizing — the Senior candidate should know better, and the interviewer knows that they should know better.

The mechanics of the failure are well-understood. The candidate hears the problem statement, feels the clock start, and experiences the anxiety of an open whiteboard or empty editor. Writing code feels productive. Sitting in silence asking questions feels like wasting time. So within sixty seconds of the problem statement ending, the candidate opens their editor and writes:

```java
public class Ride { }
public class Driver { }
public class Rider { }
```

These three classes may be correct. But the candidate has chosen them before establishing whether rides are on-demand or pre-scheduled, whether drivers can reject requests or must accept them, whether the pricing model needs to be modeled explicitly, whether a ride can be shared by multiple passengers, or whether the geographic matching algorithm is in scope. Every one of these answers could add or remove entire classes from the model. The candidate has not solved the wrong problem — yet. But they have committed to a structure before knowing what the problem is.

The cost accumulates throughout the session. When the interviewer follows up with "what if a passenger can schedule a ride 24 hours in advance?", the candidate realizes that `Ride` has no scheduling model. They begin patching: "I'll add an `isScheduled` boolean... and a `scheduledTime` field... and I'll add conditional logic in the dispatcher." Each patch makes the class harder to extend. By minute 35, the design is a patchwork of conditionals on a class that was never intended to model scheduled rides.

More importantly, this failure costs the candidate an entire scoring dimension. Requirements clarification cannot receive any score above 1 if the candidate never asks any requirements questions. The dimension simply is not demonstrated. This is a guaranteed low score on one of the eight evaluated dimensions, regardless of how well the rest of the session goes.

The correct behavior is simple in description and harder in practice: the first eight to ten minutes of the session must be requirements gathering and entity identification. No Java code should appear during this phase. The verbal marker that signals this correctly: "I want to spend a few minutes making sure I understand the domain before I start modeling. The requirements I establish here will shape the class structure more than the implementation details will."

A practical technique to enforce this on yourself: during practice, place a physical timer in front of you and refuse to write any code until it reaches the eight-minute mark. The discipline of that constraint trains the instinct to pause before building.

### 3.2 "God Class" Failure

The God class is the most common structural failure in LLD interviews, and it is distinct from the "jumping to code" failure in an important way: the candidate may have spent time on requirements and identified the right entities, but has then collapsed all the behavior into one or two central classes.

The pattern is consistent across problem domains. The candidate identifies the main noun in the problem statement — `ParkingLot`, `RideService`, `OrderManager`, `HotelSystem` — and assigns all behavior to it.

```java
// Score 1 — God class
public class RideService {
    // booking
    public Ride bookRide(RideRequest request) { ... }
    // driver matching
    public Driver findNearestDriver(Location location) { ... }
    // fare calculation
    public double calculateFare(Ride ride) { ... }
    // payment
    public void processPayment(String rideId, PaymentMethod method) { ... }
    // notification
    public void notifyDriver(Driver driver, Ride ride) { ... }
    public void notifyRider(Rider rider, Ride ride) { ... }
    // rating
    public void submitRating(String rideId, int rating, String comment) { ... }
    // reporting
    public RideStats getDailyStats(LocalDate date) { ... }
}
```

This single class handles booking, matching, pricing, payment, notification, rating, and reporting. It has at least seven distinct reasons to change: if the matching algorithm changes, if the pricing model changes, if the notification channel changes, if the payment provider changes, if the rating system changes, if the reporting requirements change, or if the booking flow changes. It violates Single Responsibility six ways simultaneously.

The root cause is almost always the same: the candidate is thinking procedurally. They are asking "what does this system do?" and producing a list of functions. They are not asking "what are the responsibilities in this system, and which object owns each one?" The shift from the former question to the latter is the shift from procedural thinking to object-oriented thinking.

The correctly decomposed design for a ride-sharing system separates these concerns:

- `RideBookingService` — orchestrates the booking flow (request validation, driver assignment trigger, ride creation)
- `DriverMatchingService` — contains the algorithm for finding and ranking available drivers
- `FareCalculationService` (or better: `FareStrategy` interface with implementations) — contains fare calculation logic
- `PaymentProcessor` interface — abstracts payment execution
- `NotificationService` interface — abstracts notification delivery
- `RatingService` — handles rating submission, storage, and aggregation
- `RideReportingService` — aggregates operational statistics

Each class has one reason to change. Adding a new pricing algorithm touches only `FareStrategy` implementations. Adding a new notification channel touches only `NotificationChannel` implementations. The `RideBookingService` orchestrates the flow but delegates all domain-specific logic to the appropriate specialist.

Testing becomes tractable: each class can be tested independently with mocked dependencies. The God class, by contrast, requires testing all concerns simultaneously, making tests large, brittle, and slow.

### 3.3 "Missing Abstraction" Failure

The missing abstraction failure is the failure to see the abstract category that unifies a set of concrete implementations. The candidate correctly identifies that there will be multiple payment methods and correctly implements each one as a separate class. But the classes are parallel silos — each one is implemented independently, without a shared interface or base class that captures what they have in common.

```java
// Score 1 — Parallel concrete implementations, no abstraction
public class CreditCardPaymentService {
    public boolean processPayment(Order order, CreditCardDetails card) {
        // validate card
        // charge via Stripe API
        // save transaction
        // send confirmation email
        return true;
    }
}

public class DebitCardPaymentService {
    public boolean processPayment(Order order, DebitCardDetails card) {
        // validate card (same structure)
        // charge via different API
        // save transaction (same structure)
        // send confirmation email (same structure)
        return true;
    }
}

public class WalletPaymentService {
    public boolean processPayment(Order order, WalletDetails wallet) {
        // validate balance (different validation)
        // deduct from wallet
        // save transaction (same structure again)
        // send confirmation email (same structure again)
        return true;
    }
}
```

Three classes with nearly identical structure. The retry logic is duplicated across all three. The transaction persistence is duplicated across all three. The confirmation email is duplicated across all three. When the retry policy changes, it must change in three places. When a new payment method is added, the entire structure is duplicated again. When the `OrderService` needs to invoke payment processing, it must know about all three classes and contain the conditional logic to choose between them.

The missing abstraction failure is particularly penalized at the Senior level because the pattern it requires — an interface with variant implementations — is one of the most basic tools in the object-oriented design toolbox. An experienced engineer sees three parallel concrete classes with similar structure and immediately asks "where is the interface?" If they do not ask that question, they are applying less rigor than their experience should guarantee.

The rule for identifying missing abstractions: any time two or more concrete classes have the same method signature with different implementations, an interface is missing. The rule for identifying what should go in an abstract base class versus an interface: if there is shared lifecycle or shared infrastructure that all implementations need, an abstract base class (or a template method) captures it; if implementations are fully independent, a pure interface suffices.

### 3.4 "Premature Optimization" Failure

This failure mode appears most commonly among candidates with distributed systems backgrounds who import production architecture concerns into an LLD session before the core domain model is established. The problem says "design a task scheduler." The candidate immediately begins designing a distributed job queue with worker pools, dead-letter queues, and Zookeeper-based leader election. The interviewer says "assume single machine for this session." The candidate pivots but has already signaled that their primary thinking lens is infrastructure and scale, not domain modeling and behavioral design.

The three specific costs of premature optimization in an interview are distinct and cumulative:

**Cost 1: Complexity without justification.** The candidate introduces mechanisms whose purpose is unclear because the problem that required them was never established. When the interviewer asks "why do you need three worker pools?", the candidate cannot give a requirements-driven answer. This signals that the complexity was imported from prior experience, not derived from the current problem.

**Cost 2: Time consumed on secondary concerns.** Ten minutes spent designing a caching layer around a repository is ten minutes not spent on the behavioral model — the state machine for the task lifecycle, the retry policy abstraction, the failure notification mechanism. The interviewer is scoring class design and behavioral correctness; the caching architecture is not on the rubric at all for this level.

**Cost 3: The primary layer is incomplete.** Candidates who spend heavily on infrastructure often have a domain model that is underdeveloped. The `Task` class has no state machine. The `Scheduler` has no policy abstraction. The executor has no interface. The core behavioral design — which is what LLD interviews evaluate — is half-finished because the session time was consumed by concerns that belong in a different kind of interview.

The correct ordering is: model correctness first, then extensibility, then performance. In a 45-minute LLD interview, performance discussion should happen only after the class diagram is complete and the core method has been implemented. When it does appear, one sentence is usually sufficient: "This design runs on a single thread. If we needed to handle high-concurrency task execution, I'd replace the in-process queue with a persistent store and add worker pool management — but the `TaskExecutor` interface I've defined means that transition doesn't change the task model or the scheduling policy."

### 3.5 "Interview Paralysis" Failure

Interview paralysis is the failure mode where forward momentum stops entirely. It manifests at two moments: at the beginning of the session (the blank whiteboard problem) and mid-session when the candidate realizes that the design they have been building has a structural problem.

Mid-session paralysis is more damaging than opening paralysis, because the candidate was progressing and then visibly regressed. A 45-second silence mid-session reads to the interviewer not as deep thinking but as a loss of confidence. The candidate was building, and then stopped. The interviewer does not know whether the candidate identified a real problem and is processing it, or whether they are simply stuck. The ambiguity resolves unfavorably in the interviewer's mind when it persists past 30 seconds without narration.

Mid-interview paralysis is triggered almost universally by the same scenario: the candidate discovers an inconsistency in their design. They realize that the `Order` class cannot cleanly model both digital and physical products within the same state machine. Or they realize that the `Notification` class they designed has no way to handle retry logic without restructuring. The freeze happens because pivoting feels like admitting failure, and the candidate is uncertain how to frame the correction.

The recovery protocol is straightforward and effective. The moment a design problem is identified, speak it aloud immediately. Do not sit with it silently. The verbal acknowledgment of the problem is itself a Senior-level signal — it demonstrates self-monitoring and intellectual honesty.

The recovery statement follows a structure: identify the problem, name it precisely, state the two options, choose one, and continue. "I'm noticing an issue. I've put the fare calculation directly on `Trip`, but what you've just described means the algorithm needs to vary independently of the trip — different pricing strategies for different markets. If I keep it on `Trip`, every pricing change requires touching the trip entity, which is exactly the coupling I was trying to avoid. I have two options: extract a `FareStrategy` interface and inject it, or use a decorator chain. I'll go with `FareStrategy` because it's simpler for what we've described, and I can add the decorator pattern later if we need composable policies. Let me restructure this quickly."

The interviewer hears: problem identified, precisely named, two options evaluated, decision made with stated reason, and the candidate is already moving. That is four positive signals from what began as a negative event.

For opening paralysis, a structured anchor breaks the freeze: "Let me start by listing the main entities I can see in this problem. I'll do a quick pass and then ask questions about the ones whose behavior is ambiguous." Speaking the entities out loud — even if some turn out to be wrong — provides something to react to and breaks the silence.

If a pattern name is genuinely unknown, describe the behavior. "I want to be able to swap in different fare calculation algorithms without touching the `Ride` class — I want that to be configurable" is a perfectly valid statement. The interviewer will often respond "that's the Strategy pattern" — accept it naturally: "Yes, exactly, Strategy — let me show how I'd structure it." Pattern names are vocabulary, not knowledge. The knowledge is the ability to identify and solve the structural problem that patterns address.

### 3.6 "Telling Not Showing" Failure

The telling-not-showing failure is the most insidious of the common failure modes because candidates who fall into it often leave the interview feeling that it went well. They talked for 45 minutes. They mentioned the Observer pattern, the Strategy pattern, and event-driven architecture. They discussed extensibility and decoupling. But nothing was ever written. No interface was defined. No class diagram was drawn. No method signature was committed to paper or screen.

The interviewer is left with nothing to evaluate. Verbal descriptions of design intentions are not verifiable. A candidate who says "I would use the Strategy pattern for fare calculation" has not demonstrated that they know how to implement Strategy. A candidate who writes:

```java
public interface FareStrategy {
    Money calculate(TripDuration duration, TripDistance distance);
}
```

...and then shows two implementations with an injection point in `Trip` has demonstrated it. The difference is not about time or effort — writing those three lines takes fifteen seconds. The difference is about the discipline of committing verbal intentions to concrete artifacts.

Interviewers are trained to watch for this. "I would use..." is a yellow flag. "I would have used..." is a red flag. "Here is how I'm using it" — followed by code — is the only statement that earns points on the pattern and abstraction dimensions.

The rule is simple: every verbal claim about a design decision must be backed by a concrete artifact. "I'll use an interface here" must be followed by the interface definition, with at least one method signature. "This is a state machine" must be followed by the enum with the state values and at least the transition guard. "I'm going to separate these concerns" must be followed by the two separate classes or interfaces. Without the artifact, the claim is unscored.

### 3.7 Five Concrete Mistakes in the First Five Minutes

The first five minutes of an LLD interview establish the trajectory for the entire session. The following five mistakes, if made in the opening minutes, create negative impressions that are difficult to reverse even if the rest of the session is strong.

**Mistake 1: Not identifying the actors before identifying the entities.**

For any non-trivial system, there are multiple actors — roles that interact with the system in different ways and may have different permissions, different workflows, and different data ownership. In a parking lot: there is a driver, an attendant, and an admin. Each actor maps to a different set of behaviors. A candidate who begins listing entities (`Spot`, `Ticket`, `Vehicle`) without first identifying actors will frequently omit entities that exist only to support a specific actor's workflow, and will model shared entities incorrectly because they assumed a single user perspective.

The first question to answer, before touching a class diagram: "Who uses this system, and what can each role do that the others cannot?"

**Mistake 2: Starting with utility or infrastructure classes.**

Beginning with a `DatabaseHelper`, a `ConfigManager`, or a `Utils` class signals that the candidate thinks in infrastructure layers before domain layers. These classes are not domain entities — they are implementation conveniences that emerge after the domain model is stable. A candidate who starts here has their design hierarchy inverted. The interviewer sees this immediately: starting with infrastructure before the domain model exists is the same as building plumbing before deciding what rooms the building will have.

Always start with domain entities and their behavioral contracts. Infrastructure classes are introduced when the domain model is stable and the specific infrastructure needs are known.

**Mistake 3: Writing getters and setters as the first code.**

When the first lines of code produced are `getOrderId()`, `setOrderId()`, `getCustomerId()`, `setCustomerId()`, the interviewer sees a Java boilerplate reflex, not a design thought. Getters and setters are not design — they are access patterns that emerge from a design. Writing them first signals that the candidate is thinking about how to expose data rather than about what the class is responsible for doing. Worse, exposing all fields via getters and setters before thinking through the behavioral contract often means the class ends up as a data container with no real behavior — an anemic domain model.

Design the behaviors first. What does this class do? What are its primary methods? What domain operations does it enable? The access patterns follow from the behavioral design, not the other way around.

**Mistake 4: Using "Manager" or "Handler" as class name suffixes without justification.**

Class names that end in `Manager` or `Handler` are the classic signals of unclear responsibility allocation. What does a `RideManager` manage? Everything related to rides? That is an entire domain, not a class. What does a `PaymentHandler` handle? Every payment scenario? That is likely a God class waiting to happen.

These suffixes are acceptable when they are specific: `DriverAssignmentManager` (manages the specific concern of assigning drivers to rides) or `RetryHandler` (handles the specific concern of retry logic for a given operation). They are red flags when they are generic: `OrderManager`, `UserHandler`, `SystemProcessor`. Generic suffixes signal that the candidate has not yet decided what the class is specifically responsible for.

If you cannot name a class without using `Manager` or `Handler`, ask: "What is the single most specific responsibility of this class?" Then name the class after that responsibility.

**Mistake 5: Confusing system design concerns with low-level design concerns.**

LLD interviews evaluate class structure, behavioral design, and object relationships. System design interviews evaluate service topology, data storage strategies, network protocols, and scaling approaches. These are different interview formats evaluating different skills.

A candidate who, in an LLD interview, spends the first five minutes asking about database partitioning, cache invalidation strategies, or API gateway configuration has misidentified the level of abstraction being evaluated. These are valid concerns in a system design session, but in an LLD session they consume time that should be spent on class design and produce output that the interviewer is not scoring.

The heuristic for distinguishing: LLD concerns live in `.java` files. System design concerns live in architecture diagrams and infrastructure configuration. If the decision you are making would not appear in any Java class, it is probably a system design concern and does not belong in this session.

---

## SECTION 4: What Senior Engineers Do Differently

### 4.1 The First 3 Questions Every Senior Asks Before Touching a Class Diagram

Senior engineers delay the class diagram longer than any other level of engineer — and they delay it more deliberately than junior engineers do (who delay from uncertainty). The Senior's delay is purposeful. They are gathering three categories of information without which any class design is premature.

---

**Question 1: "What are the actors, and what can each actor do?"**

This question drives entity identification and role separation more powerfully than any amount of domain analysis. Actors reveal the use cases, and use cases reveal the behaviors, and behaviors determine what methods exist and which objects own them.

For ride-sharing: the Rider requests rides, views ride history, and pays. The Driver accepts and declines rides, marks arrival and completion, and views earnings. The Admin manages the platform, overrides states, and accesses reporting. Each actor maps to a distinct set of operations. Those operations map to method signatures on domain entities and service classes.

Without identifying actors first, the candidate is likely to model the system from one perspective (usually the Rider's) and then discover mid-session that the Driver's workflow requires entities that are not in the current model. By asking this question first, the candidate ensures that the entity model is comprehensive from the start.

This question also reveals permission boundaries. An Admin can cancel any ride. A Rider can cancel only their own rides within a time window. A Driver cannot cancel — only decline a request before accepting. These distinctions produce guard clauses, role checks, and state transition rules that belong in the class design.

**Question 2: "What is the most likely way this system will need to change?"**

This question drives abstraction placement. The answer directly reveals which components need to be behind interfaces and which can be concrete.

If the answer is "we'll add more payment methods frequently," the `PaymentProcessor` interface is the most critical abstraction in the design. Every payment method must be behind it, and the selection mechanism must be designed for additive extension.

If the answer is "the pricing model will change frequently — marketing experiments with fares," then `FareStrategy` is the critical interface. The calculation algorithm must be completely decoupled from the trip lifecycle, injectable, and replaceable without touching any existing class.

If the answer is "we'll add new notification channels — currently SMS, but email and push are coming," then `NotificationChannel` is the critical interface, and the dispatcher must be designed to handle a list of channels rather than a hardcoded set.

This question prevents the common mistake of abstracting the wrong things. Many candidates create interface hierarchies around storage (repositories) because repositories are the canonical example in textbooks, while leaving the genuinely volatile parts of the system (pricing, notifications, matching algorithms) as concrete classes.

**Question 3: "What must never break?"**

This question drives invariant design and state machine specification. The answer reveals the constraints that must be enforced not just by convention but by the class structure itself.

For ride-sharing: a trip must never be billed twice (idempotency requirement → the payment step must check for an existing transaction before processing). A driver must never be assigned two simultaneous rides (exclusivity constraint → the driver assignment step must use a locking mechanism or a state check that prevents double assignment). A completed ride must not be cancellable (lifecycle constraint → the `cancelRide` method must check the current state and throw `InvalidStateTransitionException` if the ride is already in `COMPLETED` state).

These constraints are not nice-to-haves. They represent the correctness guarantees of the system. A design that cannot enforce them is incomplete regardless of how clean the architecture is. By asking this question before designing, the Senior candidate ensures that the invariants are structural — encoded in the class design itself — rather than documentary (written in comments or assumed by callers).

### 4.2 The "Extensibility Question"

Senior engineers habitually ask, for every concrete class they write: "What is the single most likely variation of this class that I will need in the next six months?" This is not designing for all future requirements — that way lies over-engineering. It is asking a specific, bounded question about the cost of one concrete change that is foreseeable given the domain context.

The answer sorts every component into two categories:

**Easy to extend:** The component is behind an interface. The interface has well-defined input and output types. Adding a second version of the component means implementing the interface and registering the new implementation. No existing code changes. The cost of extension is one new class.

**Hard to extend:** The component selects behavior based on type checks (`instanceof`), string comparisons, or `switch` statements. Adding a second version means opening the existing class and modifying its conditional logic, which risks breaking the existing behavior.

For every component in the "hard to extend" category, the Senior engineer asks a follow-up: "Is the stability of this component justified?" Sometimes the answer is yes — a component that genuinely will never need a second implementation does not need an interface. The engineer makes this choice consciously and can defend it. They are not failing to abstract; they are choosing not to abstract with clear rationale.

Here are three concrete examples of how this habit produces better designs in practice:

**FareCalculator as a concrete class.** The Senior asks: "What if we add surge pricing?" The answer is non-trivial — surge pricing is a different algorithm, not a parameterization of the existing one. The cost of the interface is one interface file and one implementing class — trivial. The cost of not having the interface when surge pricing arrives is a structural refactor of a class that may be referenced across the codebase. Verdict: extract `FareStrategy`.

**NotificationSender as a concrete class that sends push notifications.** The Senior asks: "What if we add email?" The implementation is completely different (push uses a device token; email uses an SMTP relay). The cost of not abstracting: two parallel code paths that share no structure. Verdict: extract `NotificationChannel`.

**DriverMatcher as a concrete class that finds the nearest driver.** The Senior asks: "What if we want priority matching for premium users?" This is a genuinely different algorithm — it may score drivers differently, have a different radius, or pull from a different driver pool. Verdict: extract `MatchingStrategy`.

The habit is mechanical enough to apply in real time during an interview: after writing each concrete class, pause for five seconds and ask the question. If the answer reveals a non-trivial variation, extract the interface immediately rather than waiting to be asked.

### 4.3 How They Handle Ambiguity

Senior engineers do not treat ambiguity as an obstacle to design. They treat it as design information — the presence of ambiguity tells them something important about the volatility of a component. A component whose requirements are ambiguous is likely to be a component whose implementation will change. That means it needs an abstraction today.

Four concrete techniques for handling ambiguity in an LLD interview:

**Technique 1: State the assumption explicitly and continue.**

When the interviewer says "assume whatever is reasonable," a Junior candidate asks more questions (which reads as insecurity) or silently assumes something (which is invisible to the interviewer and therefore unscored). A Senior candidate states the assumption out loud in one sentence, encodes its implication in the design, and continues.

"I'll assume a driver can only have one active vehicle at a time. I'm noting this as an assumption — if that changes, the relationship between `Driver` and `Vehicle` becomes one-to-many, and I'd need to add an `ActiveVehicle` concept. For now, `Driver` holds a single `Vehicle` reference."

This is complete. The assumption is visible. The design implication of the assumption is explained. The escape hatch is described. The session continues.

**Technique 2: Design for the more general case when the incremental cost is low.**

When two designs are equally simple but one is more general, use the more general one. If a one-to-one relationship and a one-to-many relationship are both plausible and the cost of using a `List` instead of a single reference is near zero, use the `List`. The upside (correct when assumptions change) greatly exceeds the downside (trivially more complex when assumptions hold).

**Technique 3: Defer the edge case explicitly without ignoring it.**

When ambiguity touches a concern that cannot be fully resolved during the design session, defer it with a visible marker — a comment, a note on the whiteboard, a verbal statement: "I'm noting that the concurrent booking case needs either a distributed lock or optimistic concurrency control with a retry. I'll design the interface for it now but won't implement the distributed coordination mechanism — that belongs in the infrastructure layer. The `BookingService` contracts I've defined here are compatible with both approaches."

The deferral is explicit and documented. When the interviewer asks about it later, the candidate can point to the deferral note and explain it. This is categorically different from simply forgetting — it demonstrates that the concern was identified, evaluated, and consciously deferred rather than missed.

**Technique 4: Apply the "what if?" probe before finalizing.**

Before calling any major design decision final, ask "what if?" once. "What if there were ten payment methods instead of three — does this design still work?" "What if rides could be shared by multiple passengers — what breaks?" "What if the fare changes mid-ride because of traffic?" These probes are the same ones the interviewer is about to ask. Running them internally first prepares the response and sometimes surfaces a design problem before the interviewer finds it.

### 4.4 How They Introduce Patterns Without Naming Them Awkwardly

The anti-pattern in pattern introduction is announcing the pattern name as if it were a password: "I'm going to use the Strategy pattern here." This can feel performative. If the implementation is incorrect, the explicit name makes the mistake more visible. If the implementation is correct but the pattern was not the best fit, the name invites scrutiny of the choice. And if the name is stated before the problem is explained, it looks like recall rather than reasoning.

Senior engineers typically introduce patterns by describing the problem they are solving, then implementing the solution. The pattern name, if mentioned at all, comes at the end — as an acknowledgment of the known name for what was just designed.

**Weak approach (pattern-first):** "I'm going to use the Observer pattern here. I'll create a subject and attach observers to it."

**Strong approach (problem-first):** "When a ride is completed, three different subsystems need to react: billing needs to calculate and charge the fare, the rating system needs to prompt both driver and rider, and analytics needs to log the completion event. I don't want `RideService` to know about all three of these consumers directly — that creates a dependency that breaks every time a new consumer is added. So instead of calling them directly, I'll model completion as an event: `RideCompletedEvent`. Any component that cares about ride completion registers as a listener. `RideService` publishes the event and is done. This decouples the publisher from the consumers completely — which is the Observer pattern, and here's the interface structure..."

The problem statement justifies the pattern. The implementation follows naturally from the problem. The pattern name comes after the implementation, as recognition rather than announcement. The interviewer hears reasoning first and nomenclature second, which is the correct ordering.

If the pattern name is genuinely unknown during the interview, describing the behavior is completely acceptable and often more impressive: "I want something where I can swap in different fare calculation algorithms without touching the `Ride` class — the algorithm should be configurable at injection time." The interviewer will frequently respond "that's Strategy" and the candidate can accept it naturally.

### 4.5 The "So What" Habit

The single most differentiating communication habit at the Senior level is connecting every design decision to its consequence. Every design choice — every interface, every composition over inheritance choice, every state machine, every value object — has a "so what": a reason it matters to the business, to the engineering team, or to the system's correctness.

Without the "so what," a design decision sounds like a preference. "I made `FareStrategy` an interface" describes a choice. "I made `FareStrategy` an interface so that adding surge pricing next quarter requires implementing one class and registering it, without touching `Trip`, `TripService`, or `CheckoutController`" justifies the choice and names its consequences.

The following table shows five representative design decisions and the "so what" that transforms each from a description into a justification:

| Design Decision | So What |
|---|---|
| `FareStrategy` is an interface with implementations | Adding a new pricing algorithm requires one new class. `Trip`, `TripService`, and `CheckoutController` are never touched. Pricing experiments can be deployed independently without regression risk on the booking flow. |
| `TripState` is an enum with explicit transition guards | Invalid state transitions throw immediately at the point of violation with a descriptive error, rather than silently corrupting state. State bugs are surfaced during development and testing, not in production. |
| `RideEventPublisher` decouples ride events from consumers | Notification, analytics, and billing all react to ride events without being coupled to each other or to `RideService`. Adding a new event consumer requires implementing one interface and registering it. `RideService` is never modified. |
| `Location` is an immutable value object | It can be safely shared across threads, passed across service boundaries, and used as a map key without synchronization. Defensive copying is unnecessary. |
| `RideRequest` is a separate entity from `Ride` | A request can be cancelled before a `Ride` exists — the cancellation logic is independent of ride lifecycle logic. A request that receives no driver response has a distinct state (`NO_DRIVER_FOUND`) that does not map to any valid `Ride` state. The two lifecycles are modeled correctly. |

The habit works at every decision scale. The smallest design choices deserve a "so what": "I extracted `TicketNumber` as a value object — so what? — so that invalid ticket formats cannot be constructed, and any method that accepts a `TicketNumber` is guaranteed to receive a valid one. The compiler enforces the invariant rather than a runtime check."

Interviewers who hear this consistently throughout a session register: this candidate thinks about the team, the product roadmap, and engineering correctness — not just the local design problem.

### 4.6 Verbal Phrases That Signal Senior-Level Thinking

The following phrases, when used naturally as an expression of underlying reasoning (not recited mechanically), consistently register as Senior-level signals in LLD interviews. Each phrase is paired with the signal it sends to the interviewer.

| Phrase | Signal |
|---|---|
| "Before I start drawing, let me understand the actors and what each one can do..." | Requirements-first discipline; the candidate knows that actors define use cases, which define behaviors, which drive the class structure |
| "I'm separating X from Y because their lifecycles are different" | Domain modeling sophistication; the candidate models real-world lifecycle boundaries rather than grouping by convenience |
| "I'll use composition here because Vehicle can exist without a Driver" | Conscious application of aggregation vs. composition based on lifecycle semantics, not habit |
| "I'm extracting this interface because I can already see two implementations — and the third is probably coming" | Abstraction driven by observed variation, not by pattern-memorization reflex |
| "Let me note this as an assumption — if it changes, here's specifically where it would affect the design" | Handles ambiguity with transparency; documents the blast radius of each assumption |
| "This entity has a state machine — let me define the valid transitions explicitly before I implement anything that changes state" | Recognizes state-heavy entities and enforces lifecycle constraints at the design level |
| "I want to make sure adding a new payment method requires zero changes to existing classes" | Open/Closed Principle applied as a working constraint, not a vocabulary item |
| "Let me think about what could go wrong in this path before we finalize it" | Proactive edge case identification; the candidate is their own QA |
| "The 'so what' here is that billing and ride matching are now completely decoupled — you can change either one without touching the other" | Business value orientation; decisions are connected to organizational impact |
| "I'd swap this out with a proper event bus in production — for now, a simple observer implementation suffices for the scope we've agreed on" | Pragmatism with production awareness; the candidate knows the gap between interview design and production architecture and articulates it honestly |
| "The one reason this class would need to change is X — everything else it does supports that one responsibility" | Single Responsibility Principle as a live design tool, not a definition to recite |
| "If you asked me to add Y six months from now, I'd need to touch only this one component — let me show you why everything else stays stable" | Extensibility thinking with concrete demonstration, not abstract claims |

---

## SECTION 5: Mock Interview Format

### 5.1 The 45-Minute Self-Mock Interview Structure

Self-mock interviews are the most underused preparation tool at the Senior level. Most candidates study LLD by reading solutions, watching explanations, and reviewing class diagrams. These are passive learning modes. They produce recognition — the ability to understand a design when it is presented — without producing generation — the ability to produce a strong design under time pressure, from a blank page, while narrating reasoning out loud.

The following minute-by-minute structure transforms passive preparation into active deliberate practice. Run this structure with a timer, a voice recorder, and a problem you have not previously designed.

---

**Phase 1: Requirements (0–8 minutes)**

*Minutes 0–3:* Read the problem statement once, deliberately. Set it down and write from memory: who are the actors? What can each one do? What is the primary noun in the problem (the system's central entity)? What is the most obvious ambiguity in the problem statement? Do not type — write by hand. The act of writing by hand slows the cognitive pace and reduces the impulse to start coding immediately.

*Minutes 3–6:* Formulate your four to six clarifying questions. Each question must pass this test: if the answer were different, would the class structure change? If yes, it is a design-affecting question. If no, it is a scale or infrastructure question that belongs in a different kind of interview. Write the questions, then rank them by impact. The highest-impact question — the one whose wrong answer would require the most redesign — should be asked first.

*Minutes 6–8:* State your scope explicitly and write it down. "I will design for: [list of included features]. I am treating [list of excluded features] as out of scope. My key assumptions are: [list of stated assumptions]." This is the contract you will hold yourself to for the rest of the session. Having it written prevents scope creep and provides a reference point when the extension scenario is introduced in Phase 5.

---

**Phase 2: Core Entities (8–14 minutes)**

*Minutes 8–11:* List the five to eight core domain entities. Do not model them yet — just list them by name and write one sentence of responsibility for each. The one-sentence test is important: if you cannot write one sentence describing what an entity is responsible for, the entity is not yet well-defined. Do not proceed with an entity until its responsibility is clear in one sentence.

*Minutes 11–14:* Draw the relationships between entities. For each pair of entities that interact, identify the relationship type: is-a (inheritance — use sparingly), has-a (composition or aggregation), uses (association), creates (factory relationship), or observes (observer relationship). Mark which relationships are the primary axes of variation — these are the relationships that will need interfaces.

At the end of Phase 2, you should have: a list of five to eight entities with one-sentence responsibilities each, a relationship map between them, and a clear identification of which entities need interfaces and which do not.

---

**Phase 3: Class Design (14–24 minutes)**

*Minutes 14–18:* Define the primary interfaces. For each relationship marked as a variation axis in Phase 2, write the interface with full method signatures. Pay attention to the types: use domain-typed parameters (`TripDuration`, `TripDistance`) rather than primitive types (`long`, `double`) where the domain distinction matters. Pay attention to the return types: use `Optional<T>` where a result may legitimately be absent. Use custom exception types rather than generic `RuntimeException` where failure has specific domain meaning.

*Minutes 18–24:* Define the concrete classes. For each class: write the constructor with its injected dependencies (no `new` inside constructors except for value objects). Write the field declarations with types and visibility modifiers (fields private by default). Write the method signatures with parameter types and return types. Do not implement methods yet — this phase is about structure and contracts, not logic.

By the end of Phase 3, you should have: all interfaces defined with full method signatures, all concrete classes defined with constructors, fields, and method signatures, and the dependency graph between classes visible in the constructor signatures.

---

**Phase 4: Code a Core Component (24–36 minutes)**

*Minutes 24–26:* Choose the component to implement. If the interviewer has specified one, use it. Otherwise, choose the component that exercises the most relationships and contains the most interesting behavioral logic. Before writing the first line of implementation, narrate out loud what you are about to implement and what makes it interesting: "I'm going to implement `RideBookingService.bookRide()` because it orchestrates the interaction between passenger request validation, driver matching, ride creation, and event publishing — it's the core transaction in this system."

*Minutes 26–36:* Implement the chosen component. Apply code quality standards throughout: name everything with intention, use no magic numbers or strings, keep each method short and cohesive, add null guards at public method boundaries, use typed exceptions with descriptive messages. As you implement, narrate the reasoning for each non-obvious decision. If you discover an edge case mid-implementation, call it out: "I'm noticing that if the driver assignment succeeds but the ride persistence fails, we have an inconsistent state. Let me add handling for that path."

---

**Phase 5: Extend the Design (36–42 minutes)**

*Minutes 36–38:* Receive the extension requirement (or formulate one yourself if self-directed). Before responding, ask one clarifying question about the extension — this demonstrates that the extension phase has the same requirements discipline as the initial design phase. Write the extension scenario in the scope section you established in Phase 1.

*Minutes 38–42:* Walk through the extension. Be explicit and specific: name the new class, name the interface it implements, name the existing classes that are not modified, and briefly show the key method signature. The goal of this phase is not to implement the extension fully — it is to demonstrate that the extension is additive rather than surgical. If any existing class requires modification to support the extension, name it explicitly and explain why — this is not automatically a failure if there is a genuine reason, but it should be acknowledged rather than glossed over.

---

**Phase 6: Debrief (42–45 minutes)**

In three sentences, summarize the design to the imagined interviewer: what the system does, how the primary abstraction enables extensibility, and what the most important design decision was. Then call out one trade-off you made and explain both sides of it. Then name one thing you would do differently with more time — and be specific: not "I would add more features" but "I would implement the concurrent booking protection more rigorously — the optimistic lock approach I described works in theory but I didn't show the retry mechanism, which is where most of the complexity lives."

Stop the recording.

### 5.2 Recording and Self-Review Checklist

After completing the self-mock, wait at least one hour before reviewing the recording. Reviewing immediately is too cognitively close — you will fill in gaps from memory rather than from the recording. The recording shows what you actually said, not what you intended to say or thought you said.

Use the following 20-item checklist during review. Mark each item Y (clearly and fully demonstrated), P (partially demonstrated — visible attempt but incomplete or unclear), or N (not demonstrated — absent entirely).

---

1. Asked at least 4 clarifying questions before touching the design
2. Stated scope explicitly before starting the class diagram (what is in scope, what is out of scope, what are the key assumptions)
3. Listed all core actors before listing domain entities
4. Every entity introduced was given a one-sentence responsibility statement at the moment of introduction
5. At least one interface was defined before any concrete class was written
6. No class name ends in "Manager" or "Handler" without a specific, single-responsibility justification
7. All fields have declared types — no fields introduced as just a name without a type
8. No magic numbers appear anywhere in the implemented code
9. Every method has an intention-revealing name that communicates its domain purpose without requiring reading the implementation
10. At least one design pattern was applied with a stated justification for why that pattern fit the problem
11. The extension scenario in Phase 5 was handled additively — the demonstration showed new classes without modifying existing ones
12. At least two edge cases were identified proactively, before the interviewer or the self-review prompted them
13. Identified edge cases are handled in code, not just acknowledged verbally (Optional return types, guard clauses, typed exceptions, or explicit state checks)
14. Every major design decision was connected to a consequence — the "so what" was stated
15. Communication was continuous throughout — no silence longer than 20 seconds without narration
16. When a design problem was discovered mid-session, a recovery statement was used rather than silent modification
17. The interviewer (imagined or real) was never in a position of not knowing what entity was being discussed or why it existed
18. Composition was used at least once where inheritance would have been the convenient but incorrect choice
19. At least one entity's state machine or lifecycle was modeled explicitly — an enum with state values, or explicit transition guards in method bodies
20. The Phase 6 debrief identified a genuine trade-off with both sides stated, not a generic "I would add more features with more time"

---

**Scoring key:**

**18–20 Y:** Strong Senior-level performance. The design process is fluent and the habits are internalized. Focus remaining practice on breadth — design systems in unfamiliar domains to stress-test the habits against novel vocabulary.

**14–17 Y:** Solid performance with clear strengths and identified gaps. Note the specific P and N items — these are the dimensions to target in the next two to three focused practice sessions before the next full mock.

**Below 14 Y:** One or more foundational habits are not yet consistent. Return to Section 3 (Common Failure Modes) and identify which failure mode pattern your session exhibited. Run a focused drill on that specific failure mode before returning to full mocks. Full mocks before the foundational habits are consistent produce diminishing returns.

### 5.3 Peer Review Guide

Self-review has a fundamental limitation: you cannot hear your own clarity gaps. When you narrate a design decision, you hear your intended meaning. A reviewer hears only what you actually said. The difference between these two is exactly the communication gap that interviewers observe.

Use the following guide to brief your peer reviewer before the session. A reviewer who knows what to watch for provides ten times more actionable feedback than one watching generally.

---

**What to observe during the interview:**

*On requirements:* Did the candidate ask domain-behavioral questions — questions whose answers would change which classes to create — or did they ask infrastructure questions that do not affect the class structure? Did they state their scope and assumptions explicitly before starting the diagram, or did the design begin without a stated constraint set?

*On entity introduction:* Every time a new entity appeared in the design, did the candidate explain its responsibility in one sentence at the moment of introduction? If the reviewer had to ask "what does this class do?", the communication score for that entity is low.

*On interfaces:* Do the interfaces in the design look like they are solving a real variation problem — there is an obvious second implementation that would be needed — or do they look like decoration on a concrete class that has no real variants? The test: if you cannot name two distinct implementations of an interface, the interface may not be solving a real problem.

*On the extension scenario:* When the extension requirement was introduced, how long did the candidate hesitate before responding? Did the response require modifying existing classes, or was it additive? Was the candidate able to name specifically which files would change and which would not?

*On silences:* Note any silence longer than 30 seconds with a timestamp and what the candidate was working on at that moment. These are the moments where narration broke down. Review them for a pattern — does the candidate go silent during implementation specifically? During complexity? During edge case handling?

---

**What to ask during debrief:**

These questions probe the same dimensions that real interviewers use. They are designed to reveal whether the candidate's verbal justifications are backed by genuine understanding or by post-hoc rationalization:

- "Why did you choose composition over inheritance for the relationship between X and Y? What would have broken if you used inheritance?"
- "What happens if the system receives two simultaneous booking requests for the same driver? Walk me through the execution path and where the conflict is detected."
- "How would you add a loyalty discount to this design right now? Which classes change, which new classes appear?"
- "If this system needed to handle ten times the request volume, what is the first thing in the design that would break?"
- "What is the one decision you made that you are least confident about, and why?"

---

**What to write in your debrief note:**

The reviewer's written feedback should be structured around four specific items:

1. The two strongest moments in the design process — specific timestamps and a description of what made each moment strong. "At minute 12, the candidate noticed that the fare calculation would need to vary independently of the trip lifecycle and immediately extracted `FareStrategy` without being asked — that's a Senior-level instinct and it was completely unprompted."

2. The one moment where recovery was handled well, if any. "At minute 28, the candidate realized their state machine was missing the `DRIVER_EN_ROUTE` state. They named the gap immediately, explained what failing to model it would cause, and added it with the correct transitions in under two minutes."

3. The one moment that the candidate should have caught themselves. "The `NotificationService` remained as a concrete class throughout the session, even after the candidate said 'we'll need to add email alongside push.' The interface was never extracted despite the explicit verbal acknowledgment of multiple implementations."

4. An overall score across the 8 dimensions using the 1–5 rubric from Section 2. The dimension-by-dimension score is far more actionable than a holistic impression.

### 5.4 The Five-Days-Before-Interview Sharpening Plan

The five days before an interview are not for learning new material. Learning new patterns, studying unfamiliar system types, or reading design articles in the week before an interview is low-value preparation. The material will not have been practiced enough to be reliable under pressure, and it may introduce uncertainty where confidence previously existed.

The five-day plan is for sharpening existing competencies, removing cognitive friction, and building the automatic recall of habits that will activate in the first critical minutes of the session.

---

**Day 5 (Five days before): Full mock on a new problem, followed by a failure mode audit.**

Run one complete 45-minute self-mock on a problem you have not previously designed. After the mock, do not immediately review the recording. Instead, audit the session from memory against the Section 3 failure modes. Which failure modes appeared, even partially? Write them down. These are your specific targets for the next four days.

Do not study new patterns today. The goal of Day 5 is diagnosis, not acquisition.

---

**Day 4 (Four days before): Targeted dimension drilling on the two weakest dimensions from Day 5.**

Take the two dimensions that scored lowest in your Day 5 mock. Run two focused 20-minute drills, one per dimension. The drill is not a full mock — it is an isolated exercise on one dimension only.

If the weak dimension is **requirements clarification:** choose five different systems (vending machine, library, hotel booking, ride-sharing, food delivery) and spend 30 minutes generating four to six design-affecting clarifying questions for each one. Do not design anything. The entire exercise is requirement question generation. At the end, review: are your questions domain-behavioral or infrastructure-oriented?

If the weak dimension is **abstraction quality:** take a design you completed previously and systematically identify every concrete class dependency that an interface could replace. For each one, write the interface, name it from the caller's perspective, and write the method signatures. Explain aloud why that specific dependency benefits from abstraction.

If the weak dimension is **extensibility demonstration:** take a design you completed previously and walk through three hypothetical extension scenarios. For each one, trace the exact code change: which new class is created, which interface it implements, and which existing classes are not modified. Practice articulating this walkthrough out loud until it is fluent.

---

**Day 3 (Three days before): Full mock on a new system, recording and scoring.**

Run a full 45-minute self-mock on a system type you have not previously designed in this preparation cycle. Apply the full structure from Section 5.1. Record the session.

After the mock, score it against the 20-item checklist from Section 5.2. Pay particular attention to the two dimensions you targeted on Day 4 — did the drill translate to improvement in the full session context? If yes, those dimensions are improving. If no, the drill was insufficient and one more targeted day is warranted before Day 1.

---

**Day 2 (Two days before): Comparative recording review and targeted phrase practice.**

Pull up the recordings from Days 5 and 3 back-to-back. For each recording, identify the three specific moments where you would have scored higher — where you almost demonstrated something well but fell short. Write these moments down with the timestamp and a one-sentence description of what was missing.

Then practice those exact moments verbally, out loud, until the reasoning narration is clean and fluent. Not the system — the specific moment. If the moment was "I introduced the `NotificationChannel` interface without explaining what it enables," practice that introduction until you can deliver it in two sentences that name the interface, state why it is an interface rather than a class, and connect it to the extension scenario it enables.

Also review the "so what" table from Section 4.5 and the verbal phrases from Section 4.6. For each phrase, say one concrete example of how you would use it for a system you have recently designed. This is recall activation, not memorization.

---

**Day 1 (One day before): No new systems. Mental rehearsal and rest.**

No new coding today. The subconscious is still processing the week's practice. New material introduced today will not have time to consolidate and will introduce uncertainty rather than confidence.

In the morning, write out by hand — not typed, handwritten — the three questions from Section 4.1 and the 8 dimensions from Section 1.4. Writing activates memory differently than reading. Do it once.

In the afternoon, review the five first-five-minutes mistakes from Section 3.7. For each mistake, write one sentence in this format: "Instead of [mistake], I will [specific correct behavior]." This creates an explicit if-then that activates under stress more reliably than general advice.

In the evening, do not study. Do not run a mock. Sleep is not a luxury — it is the mechanism by which the week's practice is consolidated into accessible memory.

**Interview morning:** Fifteen minutes before the interview begins, write on a piece of paper: the 3 pre-design questions from Section 4.1 and the 8 dimension names from Section 1.4. Use this as a silent mental checklist at the start of the session. Before asking the first clarifying question, glance at the paper. Before writing the first class, glance at the paper. This costs fifteen seconds and activates the framework that was practiced all week.

---

## Closing

LLD interviews test something more fundamental than knowledge of design patterns or familiarity with Java syntax. They test the ability to think like a Senior engineer — to decompose ambiguous real-world problems into clean object models, to anticipate variation before it arrives, to communicate design reasoning with precision, and to recover from design errors with intellectual honesty rather than panic. These are not interview skills. They are engineering skills that show up in interviews.

The framework in this document is not a checklist to memorize before a session. It is a model of how experienced engineers naturally approach design problems — the questions they ask, the habits they apply, the ways they think about abstraction, lifecycle, and extensibility. Reading it once produces recognition. Applying it through repeated, deliberate, self-reviewed practice produces internalization.

The path to a consistent Senior-level performance in LLD interviews is straightforward: design many different systems from scratch, under time pressure, while narrating reasoning out loud, and then review the recordings honestly against the eight dimensions. The systems that go well will show you what the habits look like when they are operating correctly. The systems that go poorly will show you precisely which habits have not yet been internalized — and precisely where the next round of deliberate practice should focus.

There is no shortcut from recognition to internalization. But the distance is shorter than most candidates assume, and the direction is clear.
