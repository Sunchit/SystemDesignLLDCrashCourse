# Technical Debt, Refactoring Strategy, and Technical Leadership

> This document is part of the Senior Software Engineer LLD Crash Course. It covers one of the most
> misunderstood and undervalued areas of software engineering: how to reason about, measure, and
> strategically manage technical debt, how to refactor safely and systematically, and what it means
> to lead a team technically through code reviews, design standards, and documentation culture.

---

## Table of Contents

- [Part 1: Technical Debt](#part-1-technical-debt)
  - [1. What Is Technical Debt?](#1-what-is-technical-debt)
  - [2. The Technical Debt Quadrant (Fowler)](#2-the-technical-debt-quadrant-fowler)
  - [3. Types of Technical Debt](#3-types-of-technical-debt)
  - [4. Measuring Technical Debt](#4-measuring-technical-debt)
  - [5. Debt Prioritization Strategy](#5-debt-prioritization-strategy)
- [Part 2: Refactoring](#part-2-refactoring)
  - [6. What Refactoring Is (and Is Not)](#6-what-refactoring-is-and-is-not)
  - [7. When to Refactor](#7-when-to-refactor)
  - [8. Refactoring Safely](#8-refactoring-safely)
  - [9. Catalog of Key Refactorings](#9-catalog-of-key-refactorings)
  - [10. Big Bang Rewrite vs Incremental Refactoring](#10-big-bang-rewrite-vs-incremental-refactoring)
  - [11. Strangler Fig Pattern](#11-strangler-fig-pattern)
  - [12. Branch by Abstraction](#12-branch-by-abstraction)
  - [13. Feature Toggles for Risky Refactors](#13-feature-toggles-for-risky-refactors)
- [Part 3: Code Reviews as Quality Gate](#part-3-code-reviews-as-quality-gate)
  - [14. What Senior Engineers Look for in Code Reviews](#14-what-senior-engineers-look-for-in-code-reviews)
  - [15. Code Review Checklist](#15-code-review-checklist)
  - [16. How to Give Effective Review Feedback](#16-how-to-give-effective-review-feedback)
  - [17. Architecture Review vs Code Review](#17-architecture-review-vs-code-review)
- [Part 4: Technical Leadership](#part-4-technical-leadership)
  - [18. Setting Design Standards for the Team](#18-setting-design-standards-for-the-team)
  - [19. Design Review Process](#19-design-review-process)
  - [20. Documentation Culture](#20-documentation-culture)
  - [21. Senior vs Mid-level Thinking](#21-senior-vs-mid-level-thinking)
  - [22. Closing: The Senior Engineer Mindset](#22-closing-the-senior-engineer-mindset)

---

# Part 1: Technical Debt

## 1. What Is Technical Debt?

Ward Cunningham coined the term "technical debt" in 1992, and it is one of the most frequently
cited and least faithfully understood metaphors in software engineering.

### Ward Cunningham's Original Metaphor (and What He Actually Meant)

Cunningham's original framing, presented at OOPSLA 1992, was this: shipping code before you fully
understand the domain is like taking out a loan. You get value now (shipping faster), but you owe
interest in the form of extra work every time you touch that code without first reconciling your
implementation with your improved understanding.

The key insight is that **the debt is incurred by mismatched understanding, not by writing messy
code**. Cunningham later clarified explicitly that he never intended the metaphor to excuse writing
bad code: "With borrowed money you can do something sooner than you might otherwise, but then until
you pay back that money you'll be paying interest... I thought borrowing trouble in the code base
was the same way."

The original debt metaphor describes the cost of not refactoring as your understanding of the
problem evolves. It is not a blank check to ship spaghetti code and call it "strategic debt."

### Debt Principal vs Interest: The Compounding Cost

Using the financial metaphor precisely:

- **Principal**: the additional implementation cost you would have incurred had you done it right
  the first time. For example, if a clean design would have taken 3 days instead of 1, the
  principal is 2 days.
- **Interest**: every time you touch the bad code, you pay interest. A poorly factored OrderService
  with 2,000 lines and zero tests might add 2 hours to every feature that touches it. If your team
  touches it 4 times a month, that's 8 engineer-hours per month in interest.

The dangerous property of technical debt is that, unlike financial debt, it often **compounds
exponentially rather than linearly**. A poorly abstracted data model forces every downstream service
to work around it. Those workarounds become their own debt. The codebase develops an immune system
that resists change.

### The Difference Between Debt and Legacy Code

These terms are often conflated but are distinct:

- **Technical debt** is a conscious or unconscious shortcut that creates future work. It is debt
  because it was once a choice (even if a bad one), and the principal can be paid down.
- **Legacy code** is code that is in production and working, but which is difficult to safely
  change. Michael Feathers' working definition is: "Legacy code is code without tests." It may not
  even be old code. A six-month-old service with no tests and no documentation is legacy code.

Not all legacy code is debt-laden, and not all debt is in legacy code. A brand-new system can be
written with massive design debt. A 15-year-old system can be well-tested, well-documented, and
easy to modify.

---

## 2. The Technical Debt Quadrant (Fowler)

Martin Fowler refined the technical debt concept in 2009 with his debt quadrant, which classifies
debt along two axes:

- **Reckless vs Prudent**: Was the decision made thoughtfully?
- **Deliberate vs Inadvertent**: Did the team know they were taking on debt?

```
                    RECKLESS              PRUDENT
                 +------------------+------------------+
  DELIBERATE     | "We don't have   | "We must ship    |
                 | time for design" | now and deal     |
                 |                  | with consequences"|
                 +------------------+------------------+
  INADVERTENT    | "What's          | "Now we know how |
                 | layering?"       | we should have   |
                 |                  | done it"         |
                 +------------------+------------------+
```

### Quadrant 1: Deliberate + Reckless

**"We don't have time for design."**

This is the most dangerous quadrant. The team knowingly skips design and does not acknowledge that
there will be consequences. The debt is incurred not from pressure or necessity but from
indifference or poor judgment.

**Java Example**: A team building an order management system is told the deadline is Friday. The
tech lead says "just put everything in the controller, we'll clean it up later." The result is a
1,500-line `OrderController` that talks directly to the database, contains business logic, sends
emails, and handles PDF generation. There was no plan to clean it up, and eighteen months later, no
one touches it without fear.

```java
// Deliberate + Reckless result: A controller that does everything
@RestController
public class OrderController {

    @Autowired
    private JdbcTemplate jdbc;

    @PostMapping("/orders")
    public ResponseEntity<String> createOrder(@RequestBody Map<String, Object> payload) {
        // Validation directly in controller
        if (payload.get("customerId") == null) {
            return ResponseEntity.badRequest().body("Missing customer");
        }

        // Business logic in controller
        double total = (double) payload.get("amount");
        if (total > 10000) {
            total = total * 0.9; // 10% discount for large orders
        }

        // Direct DB access in controller
        jdbc.update("INSERT INTO orders (customer_id, total) VALUES (?, ?)",
            payload.get("customerId"), total);

        // Email sending in controller
        // ... 200 more lines of mixed concerns
        return ResponseEntity.ok("Order created");
    }
}
```

This quadrant produces debt that almost never gets paid down because there was never an intention to
pay it. The "later" cleanup never arrives.

**Acceptability**: Not acceptable. This is the quadrant that destroys codebases over years.

### Quadrant 2: Deliberate + Prudent

**"We must ship now, and we know the consequences."**

This is the only responsible form of deliberate debt. The team explicitly acknowledges the tradeoff,
documents it, and plans to address it.

**Java Example**: A startup is building an MVP payment integration. The team knows that hardcoding
Stripe as the payment provider creates vendor lock-in debt. They make an explicit decision: ship
with a direct Stripe dependency, file a backlog ticket to introduce a `PaymentGateway` abstraction,
and document the constraint in a comment and an ADR.

```java
// Deliberate + Prudent: Technical debt is explicitly documented
/**
 * TECHNICAL DEBT (TD-2024-001):
 * This class is directly coupled to Stripe's API. We have intentionally skipped
 * the PaymentGateway abstraction to hit our Q1 launch deadline.
 *
 * When to address: Before adding a second payment provider.
 * Backlog ticket: PAY-441
 * Estimated effort: 2 days
 * Risk if not addressed: Cannot add PayPal, Adyen, or other providers without
 *   significant rework across the payment flow.
 */
public class StripePaymentService {

    private final Stripe stripe;

    public PaymentResult charge(Money amount, String stripeToken) {
        // Direct Stripe API call — no abstraction
        Charge charge = Charge.create(
            ChargeCreateParams.builder()
                .setAmount(amount.inCents())
                .setCurrency(amount.currency().toLowerCase())
                .setSource(stripeToken)
                .build()
        );
        return PaymentResult.success(charge.getId());
    }
}
```

**Acceptability**: Acceptable when genuinely necessary, but must be tracked and paid. The discipline
is in the documentation and follow-through, not in the decision to skip the abstraction.

### Quadrant 3: Inadvertent + Reckless

**"What's layering?"**

This quadrant represents debt caused by lack of skill or knowledge. The team did not know better.
The debt is not deliberate — they thought they were writing good code. This is the quadrant of
junior engineers left unsupervised, of teams with no code review culture, and of projects where
"it works" was the only quality standard.

**Java Example**: A junior developer builds a notification service. They do not know about
dependency injection, interface-based design, or separation of concerns. The result is a God class
that is impossible to test.

```java
// Inadvertent + Reckless: The developer didn't know about design patterns
public class NotificationManager {

    public void sendOrderConfirmation(String orderId) {
        // Direct instantiation — no DI, untestable
        MySQLConnection conn = new MySQLConnection("jdbc:mysql://prod-db:3306/orders",
            "root", "password123");
        ResultSet rs = conn.query("SELECT * FROM orders WHERE id = '" + orderId + "'");

        // Business logic mixed with infrastructure
        String customerEmail = rs.getString("customer_email");
        String subject = "Order #" + orderId + " confirmed";

        // Hardcoded SMTP — cannot mock in tests
        JavaMail.send("smtp.company.com", customerEmail, subject,
            "Your order has been placed.");

        // Also sends to Slack because "why not"
        HttpURLConnection slack = (HttpURLConnection) new URL(
            "https://hooks.slack.com/services/hardcoded-webhook").openConnection();
        // ...
    }
}
```

**Acceptability**: Not acceptable to leave in place, but requires a different response than Quadrant
1. The fix here is education, mentorship, and code review — not just cleanup sprints.

### Quadrant 4: Inadvertent + Prudent

**"Now we know how we should have done it."**

This is the healthy quadrant that Cunningham originally described. The team does their best, ships
the product, and through the act of working with the code in production, learns a better way to
structure it. This is not failure — this is how good engineering actually works.

**Java Example**: A team builds a reporting service with a `ReportGenerator` class. After six months
of feature work, they realize that the abstraction is wrong — what they actually have are three
different report types (financial, operational, compliance) with completely different lifecycle
requirements. The original design was not bad given what they knew; the problem domain revealed
itself over time.

```java
// Inadvertent + Prudent: We built this as one class, but the domain taught us otherwise

// What we originally wrote (not wrong at the time):
public class ReportGenerator {
    public Report generate(ReportType type, DateRange range) {
        // Grew to 800 lines as we added financial, operational, and compliance
        // reports — each with different rules, formats, and recipients
    }
}

// What we now know we need:
public interface ReportStrategy {
    Report generate(DateRange range);
    List<Recipient> getRecipients();
    ReportFormat format();
}

public class FinancialReportStrategy implements ReportStrategy { ... }
public class OperationalReportStrategy implements ReportStrategy { ... }
public class ComplianceReportStrategy implements ReportStrategy { ... }
```

**Acceptability**: Fully acceptable — this is normal and healthy. The correct response is to
refactor when you have the opportunity, not to feel shame about the original design.

---

## 3. Types of Technical Debt

### Design Debt

Design debt is the most expensive form because it affects everything built on top of it. It
manifests as:

- **Wrong abstractions**: An abstraction that seemed natural early on but does not fit the domain.
  For example, modeling `Invoice` and `Order` as the same `Transaction` class because they share
  some fields.
- **Missing abstractions**: Code that calls concrete implementations everywhere rather than
  programming to interfaces. A `UserService` that instantiates a `PostgresUserRepository` directly
  instead of depending on a `UserRepository` interface.
- **Leaky abstractions**: A `Repository` interface that exposes SQL-specific behavior through its
  method names (`findByJoinQuery`, `executeRawSql`).

Design debt is particularly dangerous because it is structural. You cannot fix it by renaming
variables or extracting methods. It requires rethinking the shape of the system.

### Code Debt

Code debt is the most visible form and what most people picture when they hear "technical debt":

- **Duplication**: The same business logic in three different places. When the rule changes, you
  find two of the three places. The third one silently produces incorrect results for a year.
- **Complexity**: Methods that are 300 lines long. Nested if-else structures 8 levels deep.
  Cyclomatic complexity of 40 in a single function.
- **Poor naming**: `data`, `temp`, `handler2`, `processStuff()`. Names that do not communicate
  intent force every reader to reconstruct the author's mental model from scratch.

### Test Debt

Test debt is often invisible until it becomes catastrophic:

- **Missing tests**: Code paths that have never been tested. The risk is silent until a regression
  hits production.
- **Brittle tests**: Tests that break when you change implementation details rather than behavior.
  Tests that depend on timing, file system state, or network calls.
- **Test debt compounds design debt**: Untested code is harder to refactor safely, which means
  design debt cannot be paid down without first paying test debt.

### Documentation Debt

Documentation debt is underestimated because it does not cause test failures or runtime errors:

- **Missing architecture documentation**: A new engineer cannot understand why the system is
  structured the way it is without reading months of commit history.
- **Stale documentation**: Documentation that was accurate two years ago and now actively
  misleads. Stale docs are often worse than no docs.
- **Missing decision logs**: No record of why a key technology choice was made. The team debates
  the same decision every six months because the reasoning is lost.

### Dependency Debt

- **Outdated libraries**: A service running on Java 8 and Spring Boot 1.5 when the rest of the
  organization has moved to Java 21 and Spring Boot 3.x. Upgrading requires months of compatibility
  work.
- **Security vulnerabilities**: Dependencies with known CVEs. Every month without an upgrade is
  another month of exposure.
- **Transitive dependency hell**: Version conflicts between transitive dependencies that nobody
  owns.

### Infrastructure Debt

Infrastructure debt is outside the primary scope of this document but deserves mention:
manually-provisioned servers, missing CI/CD pipelines, no infrastructure-as-code, hand-applied
database migrations, and absent disaster recovery plans. These create risk that eventually manifests
as reliability and security incidents.

---

## 4. Measuring Technical Debt

Measurement is important for two reasons: it helps you prioritize where to spend refactoring effort,
and it provides evidence when making the case to stakeholders for paying down debt. However,
measurement must be done carefully to avoid Goodhart's Law: when a measure becomes a target, it
ceases to be a good measure.

### Code Complexity Metrics

**Cyclomatic Complexity** (McCabe, 1976) counts the number of linearly independent paths through a
function. A method with no branches has a complexity of 1. Each `if`, `for`, `while`, `case`,
`catch`, `&&`, `||` adds 1.

```
Complexity  | Risk Level
------------|------------------
1-5         | Low, well-structured
6-10        | Moderate
11-20       | High, hard to test fully
21+         | Very high, refactoring candidate
```

A method with cyclomatic complexity of 25 requires at least 25 test cases to achieve full branch
coverage. In practice this means it usually has 2 test cases and large sections of untested code.

**Cognitive Complexity** is a more modern metric (SonarSource, 2018) that measures how hard code is
to understand, rather than just how many paths it has. It penalizes nesting more heavily and
recognizes that some structural patterns (like method extraction) reduce cognitive load even if they
do not reduce cyclomatic complexity.

```java
// Cyclomatic complexity: 4. Cognitive complexity: 9 (nesting penalty).
// The nested structure makes it harder to understand than the path count suggests.
public double calculateDiscount(Order order) {
    double discount = 0;
    if (order.isVip()) {                          // +1 cyclomatic, +1 cognitive
        if (order.getTotal() > 1000) {            // +1 cyclomatic, +2 cognitive (nested)
            if (order.isFirstOrder()) {            // +1 cyclomatic, +3 cognitive (nested)
                discount = 0.20;
            } else {
                discount = 0.15;
            }
        } else {
            discount = 0.10;
        }
    }
    return discount;
}
```

### Code Duplication (DRY Violations)

Duplication is measurable. Tools like PMD's CPD (Copy-Paste Detector) identify blocks of code that
appear in multiple places. A typical threshold is: blocks of 10+ tokens that appear 3+ times are
flagged. As a rule of thumb, duplication above 5% in a codebase is a significant maintenance
burden.

### Test Coverage as a Debt Proxy

Line coverage and branch coverage are debt proxies, not quality guarantees. A codebase with 80%
line coverage can still have critical, untested code paths. But very low coverage (below 40%) is
a reliable indicator that large sections of the codebase have never been exercised in isolation.

Treat coverage as a floor (below 60% branch coverage in production code is a red flag), not a
target (optimizing for 95% coverage leads to tautological tests that assert nothing meaningful).

### Code Churn + Defect Rate Correlation

This is one of the most useful debt metrics and one of the least used. **Code churn** measures how
frequently files change. When you overlay defect reports onto the change history, you can identify
the highest-risk files: those that both change frequently and have a high defect rate. These files
have a high technical debt tax because:

1. They change often (high interest payments per period).
2. They are prone to defects (the interest is painful when paid).

Many version control systems and static analysis tools can surface this data. A 2013 Microsoft
Research study of Windows Vista found that churn-based metrics outperformed traditional code quality
metrics in predicting where defects would be found.

### Tools Overview

| Tool | Primary Use | Language |
|------|-------------|----------|
| **SonarQube** | Comprehensive static analysis, technical debt estimation, security hotspots | Java, many others |
| **PMD** | Code quality rules, copy-paste detection (CPD), complexity analysis | Java |
| **SpotBugs** | Bug pattern detection (successor to FindBugs) | Java bytecode |
| **Checkstyle** | Coding style enforcement, formatting conventions | Java |
| **JaCoCo** | Test coverage measurement | Java |
| **ArchUnit** | Architectural constraint enforcement in unit tests | Java |

**The problem with treating metric thresholds as goals**: A team that sets "SonarQube technical debt
< 5 days" as a sprint goal will find ways to reduce the SonarQube estimate without improving
actual quality — renaming variables to satisfy naming rules, suppressing warnings with annotations,
writing test cases that cover lines without asserting anything. Metrics should inform decisions, not
drive them.

---

## 5. Debt Prioritization Strategy

### Not All Debt Needs to Be Paid: Opportunity Cost

Paying down debt consumes engineering time that could alternatively ship features, fix user-facing
bugs, or improve reliability. The decision to pay down a given piece of debt must be weighed against
the opportunity cost of not doing something else.

A module that is poorly designed but never changes and never breaks does not need to be refactored.
A module that is slightly messy but touched every sprint and responsible for 30% of production
incidents absolutely does.

### Prioritization Matrix

Evaluate each debt item across three dimensions:

```
Score = (Frequency of Change) x (Severity of Debt) x (Risk of Not Addressing)

Each dimension rated 1-3:

Frequency of Change:
  1 = Rarely touched (once a quarter or less)
  2 = Touched occasionally (monthly)
  3 = Touched frequently (weekly or more)

Severity of Debt:
  1 = Minor: readability issue, small duplication
  2 = Moderate: missing abstraction, moderate complexity
  3 = Severe: wrong abstraction, untestable, actively causing bugs

Risk:
  1 = Low: failure mode is recoverable, low impact
  2 = Medium: degraded user experience, operational overhead
  3 = High: data integrity risk, security risk, system instability

Score range: 1-27
Priority:
  18-27: Address in the next sprint
  9-17:  Address in the next quarter
  1-8:   Address opportunistically or not at all
```

### The Boy Scout Rule

The Boy Scouts of America have a rule: "Leave the campground cleaner than you found it." Applied to
code: whenever you touch a file for any reason, leave it slightly better than you found it. Rename
an unclear variable. Extract a 15-line block into a well-named method. Add a test for an untested
branch you noticed while reading.

This is the most sustainable form of debt paydown because it requires no dedicated budget. It
happens as part of normal feature work. Over twelve months, a team that consistently applies the
boy scout rule meaningfully improves the quality of the most frequently touched code — exactly the
code where improvement has the most impact.

### Debt Paydown: During Feature Work vs Dedicated Sprints

Both strategies are necessary at different scales:

- **During feature work** (boy scout rule): Effective for small-to-medium debt in frequently
  changed code. Sustainable and has zero political cost. Appropriate for the bottom 80% of debt
  items.
- **Dedicated refactoring sprints**: Required for large-scale, structural debt that cannot be
  addressed incrementally. Examples: replacing a data model, eliminating a God class that affects
  20 services, migrating from synchronous to asynchronous processing. These require dedicated time
  and focused attention.

A common cadence: one refactoring sprint for every four feature sprints, with the specific work
chosen based on the prioritization matrix.

### How to Make the Case to Non-Technical Stakeholders

The core mistake engineers make when advocating for debt work is using technical language with
business stakeholders. "We need to refactor the OrderService because it violates the Single
Responsibility Principle" is not persuasive to a product manager.

Translate to business terms:

- **Time**: "Every feature that touches the payment system takes us 40% longer than it should
  because the code is poorly structured. If we spend 2 weeks fixing this now, the next 10 features
  in that area will each take 2 days less. We will break even in 3 months."
- **Risk**: "The authentication module has 12% test coverage. The last three production incidents
  in that module were caused by undetected regressions. Improving the test coverage will reduce our
  incident rate."
- **Reliability**: "The current deployment process requires 6 manual steps and takes 90 minutes.
  Automating this will reduce our deployment time to 15 minutes and eliminate the risk of the 3
  manual errors we have had this quarter."

Always lead with the business consequence. The technical explanation is the supporting evidence.

---

# Part 2: Refactoring

## 6. What Refactoring Is (and Is Not)

### Fowler's Definition: Behavior-Preserving Transformation

Martin Fowler's definition from "Refactoring: Improving the Design of Existing Code" (1999, 2nd ed.
2018) is precise:

> Refactoring is a disciplined technique for restructuring an existing body of code, altering its
> internal structure without changing its external behavior.

The key phrase is **without changing its external behavior**. A refactoring is not complete if the
system behaves differently after it. The test suite must still pass. Users must not experience any
change in functionality.

### Refactoring is NOT Rewriting

Rewriting means replacing an implementation with a different one. A rewrite changes structure,
algorithm, technology, or all three. It may change behavior intentionally. It carries the risk that
the new implementation fails to reproduce the behavior of the old one (including the undocumented,
relied-upon behaviors that nobody told you about).

Refactoring operates on the existing code incrementally, in small verifiable steps.

### Refactoring is NOT Adding Features

Adding a feature while refactoring is the most common way to break both. The discipline of
refactoring requires that you put on one hat at a time:

- When refactoring: the tests must stay green, and no new behavior is added.
- When adding features: you may write new code, but you do not restructure existing code.

Kent Beck's formulation: "For each desired change, make the change easy (warning: this may be
hard), then make the easy change."

The first step is refactoring. The second step is the feature. They are separate commits.

### Why the Distinction Matters for Safety

Mixing refactoring and feature work makes bugs impossible to isolate. If a test fails after a
combined refactor+feature commit, you cannot determine whether the refactoring broke something or
the feature introduced a bug. If you separate them, a failing test after the refactoring commit
immediately tells you the refactoring changed behavior — you have not added any feature yet.

---

## 7. When to Refactor

### The Rule of Three (Duplication Trigger)

Don Knuth's original advice was to duplicate once and refactor on the third occurrence. The rule of
three:

1. First time: just do it.
2. Second time: note the duplication but continue.
3. Third time: refactor.

This prevents over-abstraction (abstracting things that happen to look similar but are not actually
the same concept) while still addressing genuine duplication.

### Pre-Feature Refactoring: "Make the Change Easy, Then Make the Easy Change"

Kent Beck's maxim describes the most effective moment to refactor: **before adding a feature**.
If you need to add a feature and the existing code makes it awkward, refactor the structure first
so that the feature becomes a clean, small addition, then add the feature.

This approach avoids the trap of jamming new features into an existing bad structure (which makes
both the feature and the structure worse). It also avoids the trap of doing a speculative
refactoring (refactoring for imagined future requirements that never arrive).

### Post-Red Refactoring: After Tests Pass, Clean Up

In Test-Driven Development, the cycle is: Red (write a failing test) -> Green (make it pass with
the simplest possible code) -> Refactor (clean up the code while keeping tests green).

The post-green refactoring step is where you eliminate the duplication, rename poorly chosen names,
and extract methods that emerged from the mechanical "make it pass" step. This is the natural
moment for small, targeted refactoring.

### The Refactoring Opportunity

Two other contexts reliably surface good refactoring opportunities:

- **Code review feedback**: When a reviewer says "this is hard to follow," that is a signal that
  the code needs structural improvement, not just a comment explaining it.
- **Bug investigation**: When you are debugging an issue, you develop a deep understanding of the
  code path involved. After the bug is fixed, you are in the best position to refactor that code
  because you understand it thoroughly. Bug-fix commits often warrant a companion refactoring
  commit.

---

## 8. Refactoring Safely

### Test Coverage is the Prerequisite

You cannot safely refactor code that has no tests. Without tests, you have no automated way to
confirm that the behavior is preserved. Every refactoring step is a leap of faith.

Before refactoring any module, ensure you have test coverage that exercises the observable behavior
of that module. This does not mean you need 100% line coverage — it means you need tests that
cover the cases that matter.

### Characterization Tests for Legacy Code (Michael Feathers Approach)

Michael Feathers' "Working Effectively with Legacy Code" (2004) introduces **characterization
tests**: tests that describe the current behavior of the system, whatever that behavior is. They
are not tests of what the system *should* do — they are tests of what it *does* do.

The process:
1. Pick a behavior you want to preserve.
2. Write a test that exercises that behavior.
3. Run it. If it passes, you have captured the current behavior.
4. If it fails in a surprising way, you have discovered an existing behavior you did not know about.
5. Adjust the test to match the observed behavior (you are characterizing, not specifying).

Characterization tests are your safety harness for refactoring legacy code.

### Small, Verifiable Steps

Each refactoring step should be:
- Small enough to complete in under an hour (ideally under 15 minutes).
- Followed by running the full test suite.
- Committed separately from other steps.

This creates a commit history where each commit represents a single, behavior-preserving
transformation. If something goes wrong, you can bisect the history to find the exact step that
broke behavior.

### Full Java Example: Characterization Tests Before Refactoring

Consider a legacy `OrderProcessor` class that has grown over years:

```java
// Legacy OrderProcessor — untested, mixed concerns, hard to understand
public class OrderProcessor {

    private Connection dbConnection;
    private Properties emailConfig;

    public OrderProcessor() {
        try {
            dbConnection = DriverManager.getConnection(
                "jdbc:mysql://prod-db/orders", "app_user", "secret");
            emailConfig = new Properties();
            emailConfig.load(new FileInputStream("/etc/app/email.properties"));
        } catch (Exception e) {
            throw new RuntimeException("Failed to initialize", e);
        }
    }

    public String process(int customerId, List<int[]> items) throws Exception {
        double total = 0;
        for (int[] item : items) {
            int productId = item[0];
            int qty = item[1];
            PreparedStatement ps = dbConnection.prepareStatement(
                "SELECT price FROM products WHERE id = ?");
            ps.setInt(1, productId);
            ResultSet rs = ps.executeQuery();
            if (rs.next()) {
                total += rs.getDouble("price") * qty;
            }
        }

        // Discount logic buried in here
        PreparedStatement cps = dbConnection.prepareStatement(
            "SELECT tier FROM customers WHERE id = ?");
        cps.setInt(1, customerId);
        ResultSet crs = cps.executeQuery();
        if (crs.next()) {
            String tier = crs.getString("tier");
            if ("GOLD".equals(tier)) {
                total = total * 0.85;
            } else if ("SILVER".equals(tier)) {
                total = total * 0.92;
            }
        }

        // Save order
        PreparedStatement ops = dbConnection.prepareStatement(
            "INSERT INTO orders (customer_id, total, status) VALUES (?, ?, 'PENDING')",
            Statement.RETURN_GENERATED_KEYS);
        ops.setInt(1, customerId);
        ops.setDouble(2, total);
        ops.executeUpdate();
        ResultSet keys = ops.getGeneratedKeys();
        keys.next();
        int orderId = keys.getInt(1);

        // Send email
        Session session = Session.getInstance(emailConfig);
        Message msg = new MimeMessage(session);
        msg.setFrom(new InternetAddress("orders@company.com"));
        // ... more email setup ...
        Transport.send(msg);

        return "ORDER-" + orderId;
    }
}
```

**Step 1: Write characterization tests against the observable behavior**

To test this, we first need to break the hard dependency on the database and email server. We
introduce a seam using subclassing (Feathers' "Extract and Override" technique):

```java
// Testable subclass that overrides the infrastructure dependencies
// This is NOT the final refactoring — it is a temporary seam for testing
class TestableOrderProcessor extends OrderProcessor {

    private Map<Integer, Double> productPrices = new HashMap<>();
    private Map<Integer, String> customerTiers = new HashMap<>();
    private List<SavedOrder> savedOrders = new ArrayList<>();
    private List<String> sentEmails = new ArrayList<>();

    public void addProduct(int id, double price) {
        productPrices.put(id, price);
    }

    public void setCustomerTier(int customerId, String tier) {
        customerTiers.put(customerId, tier);
    }

    @Override
    protected double lookupPrice(int productId) {
        return productPrices.getOrDefault(productId, 0.0);
    }

    @Override
    protected String lookupCustomerTier(int customerId) {
        return customerTiers.getOrDefault(customerId, "STANDARD");
    }

    @Override
    protected int saveOrder(int customerId, double total) {
        int id = savedOrders.size() + 1;
        savedOrders.add(new SavedOrder(id, customerId, total));
        return id;
    }

    @Override
    protected void sendConfirmationEmail(int orderId, int customerId, double total) {
        sentEmails.add("ORDER-" + orderId);
    }

    public List<SavedOrder> getSavedOrders() { return savedOrders; }
    public List<String> getSentEmails() { return sentEmails; }
}

// Characterization tests — these document CURRENT behavior, whatever it is
class OrderProcessorCharacterizationTest {

    private TestableOrderProcessor processor;

    @BeforeEach
    void setUp() {
        processor = new TestableOrderProcessor();
        processor.addProduct(1, 100.0);
        processor.addProduct(2, 50.0);
    }

    @Test
    void standardCustomerPaysFullPrice() throws Exception {
        processor.setCustomerTier(42, "STANDARD");
        String orderId = processor.process(42, List.of(new int[]{1, 2})); // 2x product 1
        assertThat(orderId).startsWith("ORDER-");
        assertThat(processor.getSavedOrders().get(0).getTotal()).isEqualTo(200.0);
    }

    @Test
    void goldCustomerGets15PercentDiscount() throws Exception {
        processor.setCustomerTier(42, "GOLD");
        processor.process(42, List.of(new int[]{1, 1})); // 1x product 1 = 100.0
        assertThat(processor.getSavedOrders().get(0).getTotal()).isEqualTo(85.0);
    }

    @Test
    void silverCustomerGets8PercentDiscount() throws Exception {
        processor.setCustomerTier(42, "SILVER");
        processor.process(42, List.of(new int[]{1, 1}));
        assertThat(processor.getSavedOrders().get(0).getTotal()).isEqualTo(92.0);
    }

    @Test
    void confirmationEmailIsSentAfterOrder() throws Exception {
        processor.setCustomerTier(42, "STANDARD");
        processor.process(42, List.of(new int[]{1, 1}));
        assertThat(processor.getSentEmails()).hasSize(1);
    }

    @Test
    void orderIdIsReturnedInExpectedFormat() throws Exception {
        processor.setCustomerTier(42, "STANDARD");
        String orderId = processor.process(42, List.of(new int[]{1, 1}));
        assertThat(orderId).matches("ORDER-\\d+");
    }
}
```

With these characterization tests in place, you can now refactor the `OrderProcessor` with
confidence. If you break any of the characterized behaviors, a test will fail immediately.

---

## 9. Catalog of Key Refactorings (with Java Examples)

### Extract Method

**When to apply**: A method is too long, or a block of code within a method needs a comment to
explain what it does (the comment should be the method name instead).

**Before**:
```java
public void processOrder(Order order) {
    // Validate order
    if (order.getItems().isEmpty()) {
        throw new IllegalArgumentException("Order must have at least one item");
    }
    if (order.getCustomerId() == null) {
        throw new IllegalArgumentException("Order must have a customer");
    }
    if (order.getShippingAddress() == null) {
        throw new IllegalArgumentException("Order must have a shipping address");
    }

    // Calculate total
    double total = 0;
    for (OrderItem item : order.getItems()) {
        total += item.getUnitPrice() * item.getQuantity();
    }
    if (total > 500) {
        total -= 25; // flat $25 discount for large orders
    }

    // Save and notify
    orderRepository.save(order.withTotal(total));
    eventPublisher.publish(new OrderPlacedEvent(order.getId(), total));
}
```

**After**:
```java
public void processOrder(Order order) {
    validateOrder(order);
    double total = calculateTotal(order);
    persistAndNotify(order, total);
}

private void validateOrder(Order order) {
    if (order.getItems().isEmpty()) {
        throw new IllegalArgumentException("Order must have at least one item");
    }
    if (order.getCustomerId() == null) {
        throw new IllegalArgumentException("Order must have a customer");
    }
    if (order.getShippingAddress() == null) {
        throw new IllegalArgumentException("Order must have a shipping address");
    }
}

private double calculateTotal(Order order) {
    double subtotal = order.getItems().stream()
        .mapToDouble(item -> item.getUnitPrice() * item.getQuantity())
        .sum();
    return applyBulkDiscount(subtotal);
}

private double applyBulkDiscount(double subtotal) {
    return subtotal > 500 ? subtotal - 25 : subtotal;
}

private void persistAndNotify(Order order, double total) {
    orderRepository.save(order.withTotal(total));
    eventPublisher.publish(new OrderPlacedEvent(order.getId(), total));
}
```

The refactored version reads as a table of contents. Each extracted method has a single
responsibility and a name that communicates intent without requiring you to read the body.

---

### Extract Class

**When to apply**: A class has too many responsibilities. Clue: the class name contains "And",
"Manager", or "Handler" and the class is large.

**Before**:
```java
public class Customer {
    private String id;
    private String firstName;
    private String lastName;
    private String email;

    // Address fields mixed into Customer
    private String street;
    private String city;
    private String state;
    private String postalCode;
    private String country;

    // Address methods mixed into Customer
    public String getFullAddress() {
        return street + ", " + city + ", " + state + " " + postalCode + ", " + country;
    }

    public boolean isInternational() {
        return !"US".equals(country);
    }

    public String formatForShipping() {
        return firstName + " " + lastName + "\n" + street + "\n"
            + city + ", " + state + " " + postalCode;
    }

    // Customer-specific methods
    public String getDisplayName() {
        return firstName + " " + lastName;
    }

    // getters and setters for all fields...
}
```

**After**:
```java
public class Address {
    private final String street;
    private final String city;
    private final String state;
    private final String postalCode;
    private final String country;

    public Address(String street, String city, String state,
                   String postalCode, String country) {
        this.street = street;
        this.city = city;
        this.state = state;
        this.postalCode = postalCode;
        this.country = country;
    }

    public String getFullAddress() {
        return street + ", " + city + ", " + state + " " + postalCode + ", " + country;
    }

    public boolean isInternational() {
        return !"US".equals(country);
    }

    public String formatForShipping(String recipientName) {
        return recipientName + "\n" + street + "\n" + city + ", " + state + " " + postalCode;
    }

    // getters...
}

public class Customer {
    private String id;
    private String firstName;
    private String lastName;
    private String email;
    private Address shippingAddress;

    public String getDisplayName() {
        return firstName + " " + lastName;
    }

    public String formatShippingLabel() {
        return shippingAddress.formatForShipping(getDisplayName());
    }

    // getters and setters...
}
```

`Address` is now its own coherent concept that can be tested independently, reused (billing
address, shipping address), and evolved separately from `Customer`.

---

### Replace Conditional with Polymorphism

**When to apply**: A switch statement or chain of if-elseif checks on a type field, and the same
pattern appears in multiple places.

**Before**:
```java
public class ShippingCalculator {

    public double calculate(Order order) {
        switch (order.getShippingType()) {
            case "STANDARD":
                return order.getWeight() * 0.50;
            case "EXPRESS":
                return order.getWeight() * 1.20 + 5.00;
            case "OVERNIGHT":
                return order.getWeight() * 2.50 + 15.00;
            case "FREE":
                return 0.0;
            default:
                throw new IllegalArgumentException(
                    "Unknown shipping type: " + order.getShippingType());
        }
    }

    public int getDeliveryDays(Order order) {
        switch (order.getShippingType()) {
            case "STANDARD":
                return 5;
            case "EXPRESS":
                return 2;
            case "OVERNIGHT":
                return 1;
            case "FREE":
                return 7;
            default:
                throw new IllegalArgumentException(
                    "Unknown shipping type: " + order.getShippingType());
        }
    }
}
```

**After**:
```java
public interface ShippingStrategy {
    double calculateCost(double weightKg);
    int getDeliveryDays();
}

public class StandardShipping implements ShippingStrategy {
    @Override
    public double calculateCost(double weightKg) { return weightKg * 0.50; }

    @Override
    public int getDeliveryDays() { return 5; }
}

public class ExpressShipping implements ShippingStrategy {
    @Override
    public double calculateCost(double weightKg) { return weightKg * 1.20 + 5.00; }

    @Override
    public int getDeliveryDays() { return 2; }
}

public class OvernightShipping implements ShippingStrategy {
    @Override
    public double calculateCost(double weightKg) { return weightKg * 2.50 + 15.00; }

    @Override
    public int getDeliveryDays() { return 1; }
}

public class FreeShipping implements ShippingStrategy {
    @Override
    public double calculateCost(double weightKg) { return 0.0; }

    @Override
    public int getDeliveryDays() { return 7; }
}

// Factory to map from string to strategy (the switch survives, but in one place)
public class ShippingStrategyFactory {
    private static final Map<String, ShippingStrategy> STRATEGIES = Map.of(
        "STANDARD", new StandardShipping(),
        "EXPRESS", new ExpressShipping(),
        "OVERNIGHT", new OvernightShipping(),
        "FREE", new FreeShipping()
    );

    public static ShippingStrategy forType(String type) {
        ShippingStrategy strategy = STRATEGIES.get(type);
        if (strategy == null) {
            throw new IllegalArgumentException("Unknown shipping type: " + type);
        }
        return strategy;
    }
}

// Usage — no switch statements, adding a new type requires a new class, not editing existing ones
public class ShippingCalculator {
    public double calculate(Order order) {
        return ShippingStrategyFactory.forType(order.getShippingType())
            .calculateCost(order.getWeight());
    }

    public int getDeliveryDays(Order order) {
        return ShippingStrategyFactory.forType(order.getShippingType())
            .getDeliveryDays();
    }
}
```

Adding a new shipping type now requires adding a new class. The Open/Closed Principle is satisfied.
Existing switch arms cannot be accidentally broken.

---

### Replace Primitive with Object (Primitive Obsession)

**When to apply**: A primitive type (String, int, double) carries domain meaning that is validated
and manipulated in many places. Classic examples: email addresses stored as String, currency stored
as double, order IDs stored as String.

**Before**:
```java
public class Order {
    private String orderId;       // "ORD-2024-000001" — format validated everywhere
    private String customerEmail; // validated everywhere with regex
    private double totalAmount;   // currency — floating point precision issues
    private String currency;      // "USD", "EUR" — no validation

    public void setCustomerEmail(String email) {
        if (!email.matches("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+$")) {
            throw new IllegalArgumentException("Invalid email: " + email);
        }
        this.customerEmail = email;
    }
    // ...validation logic duplicated wherever emails are used
}
```

**After**:
```java
public final class EmailAddress {
    private static final Pattern EMAIL_PATTERN =
        Pattern.compile("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+$");

    private final String value;

    public EmailAddress(String value) {
        if (value == null || !EMAIL_PATTERN.matcher(value).matches()) {
            throw new IllegalArgumentException("Invalid email address: " + value);
        }
        this.value = value.toLowerCase();
    }

    public String getValue() { return value; }

    @Override
    public String toString() { return value; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof EmailAddress)) return false;
        return value.equals(((EmailAddress) o).value);
    }

    @Override
    public int hashCode() { return value.hashCode(); }
}

public final class Money {
    private final long amountInCents; // avoid floating point
    private final Currency currency;

    public Money(long amountInCents, Currency currency) {
        if (amountInCents < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
        this.amountInCents = amountInCents;
        this.currency = Objects.requireNonNull(currency);
    }

    public static Money ofDollars(double dollars, Currency currency) {
        return new Money(Math.round(dollars * 100), currency);
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add different currencies");
        }
        return new Money(this.amountInCents + other.amountInCents, this.currency);
    }

    public double toDecimal() { return amountInCents / 100.0; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money)) return false;
        Money other = (Money) o;
        return amountInCents == other.amountInCents && currency.equals(other.currency);
    }
    // hashCode, toString...
}

// Now Order uses type-safe, self-validating value objects
public class Order {
    private final OrderId id;
    private final EmailAddress customerEmail;
    private Money total;
    // ...
}
```

Validation is centralized in the value object. The compiler enforces correct usage. You cannot
accidentally pass a phone number where an email is expected.

---

### Introduce Parameter Object

**When to apply**: A method takes many parameters that belong together, or the same set of
parameters is passed through multiple methods.

**Before**:
```java
public class ReportService {

    public Report generateSalesReport(
            LocalDate startDate,
            LocalDate endDate,
            String region,
            String productCategory,
            boolean includeReturns,
            boolean groupByMonth,
            ReportFormat format) {
        // ...
    }

    public Report generateInventoryReport(
            LocalDate startDate,
            LocalDate endDate,
            String region,
            String productCategory,
            boolean includeDiscontinued,
            ReportFormat format) {
        // ...
    }
}
```

**After**:
```java
public class DateRange {
    private final LocalDate start;
    private final LocalDate end;

    public DateRange(LocalDate start, LocalDate end) {
        if (start.isAfter(end)) {
            throw new IllegalArgumentException("Start date must be before end date");
        }
        this.start = start;
        this.end = end;
    }

    public LocalDate getStart() { return start; }
    public LocalDate getEnd() { return end; }
    public boolean contains(LocalDate date) {
        return !date.isBefore(start) && !date.isAfter(end);
    }
}

public class ReportFilter {
    private final DateRange dateRange;
    private final String region;
    private final String productCategory;
    private final ReportFormat format;

    // Builder pattern for optional parameters
    public static class Builder {
        private final DateRange dateRange;
        private String region = "ALL";
        private String productCategory = "ALL";
        private ReportFormat format = ReportFormat.PDF;

        public Builder(DateRange dateRange) {
            this.dateRange = dateRange;
        }

        public Builder region(String region) { this.region = region; return this; }
        public Builder category(String category) { this.productCategory = category; return this; }
        public Builder format(ReportFormat format) { this.format = format; return this; }
        public ReportFilter build() { return new ReportFilter(this); }
    }

    private ReportFilter(Builder b) {
        this.dateRange = b.dateRange;
        this.region = b.region;
        this.productCategory = b.productCategory;
        this.format = b.format;
    }

    // getters...
}

public class SalesReportOptions {
    private final ReportFilter filter;
    private final boolean includeReturns;
    private final boolean groupByMonth;
    // ...
}

public class ReportService {

    public Report generateSalesReport(SalesReportOptions options) { /* ... */ }

    public Report generateInventoryReport(ReportFilter filter, boolean includeDiscontinued) {
        /* ... */
    }
}
```

The parameter object also benefits from being reusable, testable in isolation, and extensible
without changing method signatures.

---

### Move Method

**When to apply**: A method in class A uses more data from class B than from class A. Feature Envy
(a method more interested in another class than its own) is the code smell.

**Before**:
```java
public class OrderItem {
    private int productId;
    private int quantity;
    private double unitPrice;

    // getters...
}

public class OrderSummaryService {

    // This method only uses OrderItem data — it belongs on OrderItem
    public double calculateItemTotal(OrderItem item) {
        double base = item.getUnitPrice() * item.getQuantity();
        if (item.getQuantity() >= 10) {
            return base * 0.95; // 5% bulk discount
        }
        return base;
    }
}
```

**After**:
```java
public class OrderItem {
    private int productId;
    private int quantity;
    private double unitPrice;

    // Moved here — the method belongs with the data it uses
    public double calculateTotal() {
        double base = unitPrice * quantity;
        return quantity >= 10 ? base * 0.95 : base;
    }

    // getters...
}

public class OrderSummaryService {
    // Now delegates to the object that owns the data
    public double calculateOrderTotal(List<OrderItem> items) {
        return items.stream().mapToDouble(OrderItem::calculateTotal).sum();
    }
}
```

---

### Remove Dead Code

**When to apply**: Code that is never called, commented-out code that has been in the codebase for
more than one sprint, and feature flags for features that shipped more than six months ago.

**Before**:
```java
public class PaymentService {

    public PaymentResult process(Payment payment) {
        return processWithNewFlow(payment);
    }

    // TODO: Remove after PAYMENT-123 is stable (ticket closed 8 months ago)
    private PaymentResult processWithOldFlow(Payment payment) {
        // 200 lines of old payment processing logic
        // ...
    }

    private PaymentResult processWithNewFlow(Payment payment) {
        // Current implementation
        // ...
    }

    // Commented out during refactoring — never re-enabled
    // private void validateCardNumber(String cardNumber) {
    //     if (cardNumber.length() != 16) throw new ...
    // }

    // DEPRECATED: Use process() instead
    @Deprecated
    public PaymentResult submitPayment(Payment payment) {
        return process(payment); // wrapper from old API — no callers remain
    }
}
```

**After**:
```java
public class PaymentService {

    public PaymentResult process(Payment payment) {
        // ... current implementation only
    }
}
```

Dead code is not free. It creates confusion ("should I be using processWithOldFlow or
processWithNewFlow?"), it is included in searches and static analysis, and it misleads new
engineers. Version control is the history — delete with confidence.

---

## 10. Big Bang Rewrite vs Incremental Refactoring

### Why Big Bang Rewrites Almost Always Fail

Joel Spolsky's 2000 essay "Things You Should Never Do, Part I" makes the case directly. His
argument: the old code, however ugly, encodes years of bug fixes, edge cases, and learned behaviors.
"All those weird lines of code that you look at and say 'what on earth is this for?' — those are
bug fixes."

A rewrite starts from scratch. It re-discovers all the same problems the original team solved,
but this time without the benefit of production data, user feedback, or the original engineers.
The rewrite takes 18 months instead of the promised 6. By the time it ships, it has a new class
of bugs and is missing behaviors the old system had.

The empirical data supports this. Notable failed rewrites: Netscape 6 (2000), Apple's Copland
project, the NHS IT Programme, and dozens of internal enterprise systems that are never publicly
reported.

### When Incremental Refactoring is the Right Choice (Almost Always)

Incremental refactoring:
- Keeps the system in production throughout the process.
- Allows you to validate each change against real traffic.
- Distributes the risk across many small steps.
- Preserves accumulated behavioral knowledge.
- Is interruptible — if business priorities change, you stop and the system still works.

The only situations where a full rewrite is genuinely the right choice:
- The technology stack is so obsolete that no modern tooling can run it (COBOL on 1970s hardware).
- The legal or regulatory environment requires a clean-room implementation.
- The system is so completely wrong that there is no salvageable behavior (extremely rare).

In 20 years of professional software engineering across most teams, one of these conditions is
almost never true. The appropriate choice is almost always incremental refactoring, possibly
accompanied by the Strangler Fig pattern.

### The Cost of Parallel Maintenance During a Rewrite

Even when a rewrite is attempted, teams often underestimate the cost of **running both systems
simultaneously**. During the transition period:
- Every new feature must be implemented in both systems.
- Both systems must be tested.
- Production incidents in the old system still require fixes.
- Data must be kept in sync between old and new systems.
- Operations teams must support two systems.

This parallel maintenance cost often equals or exceeds the cost of incremental refactoring. It is
not a free lunch.

---

## 11. Strangler Fig Pattern

### How to Incrementally Replace Legacy Systems

The Strangler Fig pattern (Martin Fowler, 2004) is named after the strangler fig tree, which grows
around a host tree, eventually replacing it while using it as a scaffold during growth.

Applied to software: a new system grows alongside the legacy system. Traffic is gradually routed
from the old system to the new one, endpoint by endpoint, feature by feature. When all traffic has
been migrated, the legacy system is shut down.

The pattern works because:
- The new system is built incrementally, with production validation at each step.
- The legacy system continues to serve traffic during the migration.
- Rollback is trivial at any point — just route traffic back to the old system.
- The migration can be paused and resumed without breaking anything.

### The Facade Layer That Intercepts Calls

The key architectural element is a routing facade that sits in front of both systems. All requests
go through the facade. The facade decides which system handles each request based on routing rules.

### Full Java Example: Strangling a Legacy OrderProcessor

**Step 1: The legacy system**

```java
// The legacy system we are replacing
public class LegacyOrderProcessor {

    public String createOrder(int customerId, List<OrderItem> items) throws Exception {
        // ... 500 lines of legacy code
        return "ORDER-" + generateId();
    }

    public OrderStatus getOrderStatus(String orderId) {
        // ... legacy status lookup
        return OrderStatus.UNKNOWN;
    }

    public void cancelOrder(String orderId) {
        // ... legacy cancellation
    }
}
```

**Step 2: The new system being built**

```java
// The new system — built with proper design
public class NewOrderService {

    private final OrderRepository orderRepository;
    private final PricingService pricingService;
    private final EventPublisher eventPublisher;

    public NewOrderService(OrderRepository orderRepository,
                           PricingService pricingService,
                           EventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.pricingService = pricingService;
        this.eventPublisher = eventPublisher;
    }

    public OrderId createOrder(CustomerId customerId, List<OrderItem> items) {
        Money total = pricingService.calculateTotal(items, customerId);
        Order order = Order.create(customerId, items, total);
        orderRepository.save(order);
        eventPublisher.publish(new OrderCreatedEvent(order.getId(), total));
        return order.getId();
    }

    public OrderStatus getOrderStatus(OrderId orderId) {
        return orderRepository.findById(orderId)
            .map(Order::getStatus)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }

    public void cancelOrder(OrderId orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        order.cancel();
        orderRepository.save(order);
        eventPublisher.publish(new OrderCancelledEvent(orderId));
    }
}
```

**Step 3: The strangler facade with routing logic**

```java
// The routing facade — this is what all callers use
// It routes to old or new system based on migration state
public class OrderServiceFacade {

    private final LegacyOrderProcessor legacyProcessor;
    private final NewOrderService newService;
    private final MigrationRouter router;
    private static final Logger log = LoggerFactory.getLogger(OrderServiceFacade.class);

    public OrderServiceFacade(LegacyOrderProcessor legacyProcessor,
                               NewOrderService newService,
                               MigrationRouter router) {
        this.legacyProcessor = legacyProcessor;
        this.newService = newService;
        this.router = router;
    }

    public String createOrder(int customerId, List<OrderItem> items) {
        if (router.useNewSystemForCreate()) {
            // New system path
            OrderId id = newService.createOrder(
                new CustomerId(customerId),
                items
            );
            log.info("Order created via new system: {}", id);
            return id.toString();
        } else {
            // Legacy path
            try {
                String orderId = legacyProcessor.createOrder(customerId, items);
                log.info("Order created via legacy system: {}", orderId);
                return orderId;
            } catch (Exception e) {
                throw new OrderProcessingException("Legacy order creation failed", e);
            }
        }
    }

    public OrderStatus getOrderStatus(String orderId) {
        if (router.useNewSystemForStatus(orderId)) {
            return newService.getOrderStatus(new OrderId(orderId));
        } else {
            return legacyProcessor.getOrderStatus(orderId);
        }
    }

    public void cancelOrder(String orderId) {
        if (router.useNewSystemForCancel(orderId)) {
            newService.cancelOrder(new OrderId(orderId));
        } else {
            legacyProcessor.cancelOrder(orderId);
        }
    }
}
```

**Step 4: The migration router with graduated rollout**

```java
// Controls which system handles which requests
// Routing rules can be changed via configuration without redeployment
public class MigrationRouter {

    private final FeatureFlags flags;

    public MigrationRouter(FeatureFlags flags) {
        this.flags = flags;
    }

    // Phase 1: Not started — all traffic goes to legacy
    // Phase 2: New system handles create for new orders
    // Phase 3: New system handles all create + status for new orders
    // Phase 4: New system handles everything — legacy decommissioned

    public boolean useNewSystemForCreate() {
        return flags.isEnabled("order-service-migration.create");
    }

    public boolean useNewSystemForStatus(String orderId) {
        // Only use new system for orders created in the new system
        // (identified by prefix or a lookup table)
        return flags.isEnabled("order-service-migration.status")
            && isNewSystemOrder(orderId);
    }

    public boolean useNewSystemForCancel(String orderId) {
        return flags.isEnabled("order-service-migration.cancel")
            && isNewSystemOrder(orderId);
    }

    private boolean isNewSystemOrder(String orderId) {
        // New system uses a UUID-based ID format
        // Legacy system uses "ORDER-NNNN" format
        return orderId.length() == 36 && orderId.contains("-");
    }
}
```

The migration proceeds endpoint by endpoint:

1. Deploy the facade routing all traffic to legacy. Validate in production.
2. Enable `order-service-migration.create` for 1% of traffic. Monitor error rates.
3. Ramp to 10%, 50%, 100%. Monitor at each step.
4. Enable status and cancel routing as those endpoints are validated.
5. Remove the legacy code path from the facade.
6. Decommission the legacy system.

---

## 12. Branch by Abstraction

### Safer Than Feature Branches for Long-Running Refactors

Long-running feature branches are painful to integrate. The longer a branch lives, the more it
diverges from the main branch, and the more painful the merge. This is particularly bad for
refactorings that touch many files.

Branch by Abstraction (Paul Hammant, 2007) allows a long-running refactoring to live on the main
branch without breaking it.

### The Four Steps

1. **Create an abstraction** over the code you intend to replace. All callers use the abstraction.
2. **Build a new implementation** behind the abstraction. Deploy with the old implementation still
   active.
3. **Switch the abstraction** to use the new implementation (via configuration or code change).
4. **Remove the old implementation** once the new one is validated.

### Full Java Example: Migrating from Sync HTTP Client to Async

**Scenario**: The system uses `OkHttpClient` synchronously. We need to migrate to `CompletableFuture`-based
async processing to handle higher throughput. This is a significant change that will touch many
call sites.

**Step 1: Create the abstraction. All callers switch to use it.**

```java
// The abstraction — represents our HTTP client capability
public interface HttpClientPort {
    /**
     * Execute an HTTP request and return the response body.
     * May be implemented synchronously or asynchronously.
     */
    CompletableFuture<String> get(String url, Map<String, String> headers);

    CompletableFuture<String> post(String url, String body, Map<String, String> headers);
}
```

**Step 2: Build the existing sync client behind the abstraction (preserving current behavior)**

```java
// Old implementation wrapped behind the new abstraction
// This runs the sync OkHttp call on a dedicated thread pool
public class SynchronousHttpClientAdapter implements HttpClientPort {

    private final OkHttpClient okHttpClient;
    private final Executor executor;

    public SynchronousHttpClientAdapter(OkHttpClient okHttpClient) {
        this.okHttpClient = okHttpClient;
        // Run sync calls on a bounded thread pool so they don't block event loop
        this.executor = Executors.newFixedThreadPool(50,
            new ThreadFactoryBuilder().setNameFormat("sync-http-%d").build());
    }

    @Override
    public CompletableFuture<String> get(String url, Map<String, String> headers) {
        return CompletableFuture.supplyAsync(() -> {
            Request.Builder builder = new Request.Builder().url(url);
            headers.forEach(builder::addHeader);
            try (Response response = okHttpClient.newCall(builder.build()).execute()) {
                if (!response.isSuccessful()) {
                    throw new HttpException(response.code(), url);
                }
                return response.body().string();
            } catch (IOException e) {
                throw new UncheckedIOException(e);
            }
        }, executor);
    }

    @Override
    public CompletableFuture<String> post(String url, String body,
                                          Map<String, String> headers) {
        return CompletableFuture.supplyAsync(() -> {
            RequestBody requestBody = RequestBody.create(body,
                MediaType.parse("application/json"));
            Request.Builder builder = new Request.Builder().url(url).post(requestBody);
            headers.forEach(builder::addHeader);
            try (Response response = okHttpClient.newCall(builder.build()).execute()) {
                if (!response.isSuccessful()) {
                    throw new HttpException(response.code(), url);
                }
                return response.body().string();
            } catch (IOException e) {
                throw new UncheckedIOException(e);
            }
        }, executor);
    }
}
```

**Step 3: Build the new async implementation**

```java
// New implementation using Java 11+ HttpClient with true async
public class AsyncHttpClientAdapter implements HttpClientPort {

    private final java.net.http.HttpClient httpClient;

    public AsyncHttpClientAdapter() {
        this.httpClient = java.net.http.HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(10))
            .executor(Executors.newVirtualThreadPerTaskExecutor()) // Java 21
            .build();
    }

    @Override
    public CompletableFuture<String> get(String url, Map<String, String> headers) {
        HttpRequest.Builder builder = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .GET();
        headers.forEach(builder::header);

        return httpClient.sendAsync(builder.build(), HttpResponse.BodyHandlers.ofString())
            .thenApply(response -> {
                if (response.statusCode() >= 400) {
                    throw new HttpException(response.statusCode(), url);
                }
                return response.body();
            });
    }

    @Override
    public CompletableFuture<String> post(String url, String body,
                                          Map<String, String> headers) {
        HttpRequest.Builder builder = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .POST(HttpRequest.BodyPublishers.ofString(body));
        headers.forEach(builder::header);
        builder.header("Content-Type", "application/json");

        return httpClient.sendAsync(builder.build(), HttpResponse.BodyHandlers.ofString())
            .thenApply(response -> {
                if (response.statusCode() >= 400) {
                    throw new HttpException(response.statusCode(), url);
                }
                return response.body();
            });
    }
}
```

**Step 4: Configuration controls which implementation is active**

```java
@Configuration
public class HttpClientConfiguration {

    @Value("${http.client.use-async:false}")
    private boolean useAsync;

    @Bean
    public HttpClientPort httpClientPort() {
        if (useAsync) {
            return new AsyncHttpClientAdapter();
        } else {
            return new SynchronousHttpClientAdapter(new OkHttpClient());
        }
    }
}
```

**Step 4 is the switch**: set `http.client.use-async=true` in configuration (via feature flag or
properties file). Roll back by setting it to `false`. When the async implementation is validated,
delete `SynchronousHttpClientAdapter` and remove the conditional.

---

## 13. Feature Toggles for Risky Refactors

### Using Flags to Gradually Roll Out Refactored Code

Feature toggles (also called feature flags) allow you to deploy code to production without exposing
it to all users immediately. For risky refactors, you can:

- Deploy both the old and new implementation.
- Route a percentage of traffic to the new implementation.
- Monitor error rates, latency, and business metrics.
- Increase the percentage as confidence grows.
- Roll back instantly by changing the flag value, without redeployment.

### A/B Testing Refactors

For refactors where correctness is uncertain, you can run both implementations simultaneously and
compare outputs. This is the **shadow mode** or **dark launch** pattern: the old implementation
handles the request, the new implementation runs in parallel, and the outputs are compared. Any
discrepancies are logged for investigation.

### Full Java Example: Toggling Between Old and New Payment Processing

```java
// Feature flag interface — can be backed by LaunchDarkly, ConfigCat, Spring Config, etc.
public interface FeatureFlags {
    boolean isEnabled(String flagName, String userId);
    boolean isEnabled(String flagName);
    int getRolloutPercentage(String flagName);
}

// Old payment processor (being replaced)
public class LegacyPaymentProcessor {

    public PaymentResult charge(String customerId, Money amount, String paymentToken) {
        // ... old Stripe integration with manual retry logic
        log.info("Processing payment via legacy processor for customer {}", customerId);
        return PaymentResult.success("legacy-charge-" + UUID.randomUUID());
    }

    public RefundResult refund(String chargeId, Money amount) {
        // ... old refund logic
        return RefundResult.success("legacy-refund-" + UUID.randomUUID());
    }
}

// New payment processor (replacement)
public class ModernPaymentProcessor {

    private final StripeClient stripeClient;
    private final RetryTemplate retryTemplate;

    public ModernPaymentProcessor(StripeClient stripeClient, RetryTemplate retryTemplate) {
        this.stripeClient = stripeClient;
        this.retryTemplate = retryTemplate;
    }

    public PaymentResult charge(String customerId, Money amount, String paymentToken) {
        log.info("Processing payment via modern processor for customer {}", customerId);
        return retryTemplate.execute(ctx ->
            stripeClient.charge(customerId, amount.inCents(), amount.getCurrencyCode(),
                paymentToken)
        );
    }

    public RefundResult refund(String chargeId, Money amount) {
        return stripeClient.refund(chargeId, amount.inCents());
    }
}

// The toggling facade
public class PaymentProcessorFacade {

    private static final Logger log = LoggerFactory.getLogger(PaymentProcessorFacade.class);
    private static final String PAYMENT_FLAG = "modern-payment-processor";

    private final LegacyPaymentProcessor legacyProcessor;
    private final ModernPaymentProcessor modernProcessor;
    private final FeatureFlags featureFlags;
    private final MetricsRegistry metrics;

    public PaymentProcessorFacade(LegacyPaymentProcessor legacyProcessor,
                                   ModernPaymentProcessor modernProcessor,
                                   FeatureFlags featureFlags,
                                   MetricsRegistry metrics) {
        this.legacyProcessor = legacyProcessor;
        this.modernProcessor = modernProcessor;
        this.featureFlags = featureFlags;
        this.metrics = metrics;
    }

    public PaymentResult charge(String customerId, Money amount, String paymentToken) {
        boolean useModern = featureFlags.isEnabled(PAYMENT_FLAG, customerId);
        String processorTag = useModern ? "modern" : "legacy";

        long start = System.currentTimeMillis();
        try {
            PaymentResult result = useModern
                ? modernProcessor.charge(customerId, amount, paymentToken)
                : legacyProcessor.charge(customerId, amount, paymentToken);

            metrics.increment("payment.charge.success", "processor", processorTag);
            return result;

        } catch (Exception e) {
            metrics.increment("payment.charge.error", "processor", processorTag);
            log.error("Payment charge failed via {} processor for customer {}",
                processorTag, customerId, e);

            // If modern processor fails, fall back to legacy for safety
            if (useModern) {
                log.warn("Falling back to legacy processor for customer {}", customerId);
                metrics.increment("payment.charge.fallback");
                return legacyProcessor.charge(customerId, amount, paymentToken);
            }
            throw e;

        } finally {
            metrics.recordTime("payment.charge.duration",
                System.currentTimeMillis() - start, "processor", processorTag);
        }
    }

    // Shadow mode: run both, compare, return legacy result
    public PaymentResult chargeWithShadowValidation(String customerId, Money amount,
                                                     String paymentToken) {
        // Run legacy (authoritative)
        PaymentResult legacyResult = legacyProcessor.charge(customerId, amount, paymentToken);

        // Run modern in shadow (non-authoritative, errors are swallowed)
        try {
            PaymentResult modernResult = modernProcessor.charge(customerId, amount, paymentToken);
            if (!resultsMatch(legacyResult, modernResult)) {
                log.warn("SHADOW: Payment result mismatch for customer {}. " +
                    "Legacy: {}, Modern: {}", customerId, legacyResult, modernResult);
                metrics.increment("payment.shadow.mismatch");
            } else {
                metrics.increment("payment.shadow.match");
            }
        } catch (Exception e) {
            log.warn("SHADOW: Modern processor error (non-impacting)", e);
            metrics.increment("payment.shadow.error");
        }

        return legacyResult; // Always return legacy result in shadow mode
    }

    private boolean resultsMatch(PaymentResult legacy, PaymentResult modern) {
        // Compare relevant fields (not the charge IDs, which will differ)
        return legacy.isSuccess() == modern.isSuccess()
            && legacy.getAmountCharged().equals(modern.getAmountCharged());
    }
}
```

**Migration phases:**
1. Shadow mode: both run, legacy is authoritative, discrepancies are logged.
2. 1% of customers use modern (A/B mode). Monitor metrics carefully.
3. Ramp to 10%, 25%, 50%, 100%.
4. Remove legacy processor and the toggle.

---

# Part 3: Code Reviews as Quality Gate

## 14. What Senior Engineers Look for in Code Reviews

Code review is one of the highest-leverage activities available to a senior engineer. A 30-minute
review that catches a design flaw before it is merged saves days of future refactoring and
potentially prevents production incidents.

Senior engineers review for the following, roughly in order of importance:

### Correctness: Does It Do What It Claims?

This is the baseline. Does the implementation match the specification? Are edge cases handled?
Are error conditions covered? A common failure mode is reviewing the happy path and missing the
edge cases that will cause production incidents: null inputs, empty collections, negative numbers,
concurrent access, network timeouts.

### Design: Is the Abstraction Right?

Correct code with a wrong abstraction is worse than incorrect code with a right abstraction,
because wrong abstractions compound. Questions to ask:
- Does the class/method do one thing?
- Is the naming accurate? Would a new engineer infer the right behavior from the name?
- Are the dependencies pointing in the right direction? (Details depending on abstractions, not
  abstractions depending on details.)
- Is this change making the system simpler or more complex?

### Testability: Can This Be Tested?

If a class cannot be unit tested without significant mocking infrastructure, it has a design
problem. Senior engineers look for:
- Hard dependencies on `new` inside business logic (should be injected).
- Static method calls to infrastructure (database, file system, network).
- Global state or singletons in business logic.
- Methods that do too many things to test one behavior at a time.

### Maintainability: Will the Next Engineer Understand This?

Code is written once and read many times. The most relevant reader is "you, six months from now"
or "a new team member who did not write this." Signals of poor maintainability:
- Methods longer than 30-40 lines.
- Variable names that require reading the body to understand.
- Logic that only makes sense if you know the history.
- Missing tests that document behavior.

### Performance: Any Obvious Bottlenecks?

Not micro-optimization — obvious problems:
- N+1 queries (loading a collection in a loop, each iteration triggering a database call).
- Loading large datasets into memory when a streaming approach is available.
- Missing database indexes on columns used in WHERE clauses.
- Synchronous blocking calls in a code path that should be non-blocking.

### Security: Any Obvious Vulnerabilities?

- SQL injection risks (any String concatenation in SQL).
- Unvalidated user input being used in file paths, commands, or redirects.
- Sensitive data being logged.
- Missing authorization checks.
- Insecure defaults (trusting external input, disabled SSL verification).

### Observability: Is There Enough Logging and Metrics?

- Are significant operations logged at appropriate levels?
- Are errors logged with enough context to diagnose them in production?
- Are there metrics for latency and error rate on new code paths?
- Is the log output going to be meaningful at 3am during an incident?

### Not: Style (That Is What Linters Are For)

A senior engineer does not spend review time on formatting, import order, brace placement, or
naming conventions that a linter can catch. These should be caught by automated tools (Checkstyle,
PMD, SpotBugs) and fail the CI build before human review. Human review time is too valuable to
spend on mechanical checks.

---

## 15. Code Review Checklist

A structured checklist for senior engineers reviewing Java code:

### Functionality

- [ ] Does the implementation match the stated requirements?
- [ ] Are all happy path cases covered?
- [ ] Are boundary conditions handled (empty lists, null, zero, max values)?
- [ ] Is concurrent access to shared state safe (synchronization, immutability, thread-local)?
- [ ] Are external API contracts honored (method signatures, return types, checked exceptions)?
- [ ] Are all error conditions handled or explicitly propagated?

### Error Handling

- [ ] Are exceptions caught at the appropriate level (not too early, not swallowed)?
- [ ] Do error messages contain enough context to diagnose the problem?
- [ ] Are checked exceptions declared when appropriate?
- [ ] Are resources closed properly (try-with-resources for Closeable)?
- [ ] Are timeouts set on external calls?
- [ ] Is the system left in a consistent state after an error?

### Security

- [ ] Is user input validated before use (format, range, whitelist)?
- [ ] Are parameterized queries used for all database operations?
- [ ] Is sensitive data (passwords, tokens, PII) absent from logs?
- [ ] Are authorization checks in place for all new endpoints?
- [ ] Is new serialization/deserialization from untrusted sources safe?
- [ ] Are any new dependencies checked for known CVEs?

### Performance

- [ ] Are there database queries inside loops (N+1 pattern)?
- [ ] Are large result sets paginated or streamed rather than loaded entirely?
- [ ] Are expensive operations cached where appropriate?
- [ ] Are synchronous blocking operations used where async would be required by the system's
  non-functional requirements?
- [ ] Are new database queries backed by appropriate indexes?

### Testing

- [ ] Do unit tests cover the happy path and key edge cases?
- [ ] Are the tests actually testing behavior rather than implementation details?
- [ ] Are the tests readable? Does the test name describe the scenario?
- [ ] Is test data realistic?
- [ ] Are there integration tests for new API endpoints?
- [ ] Would the tests catch a regression if someone broke the code?

### Documentation

- [ ] Are non-obvious decisions documented in comments?
- [ ] Is Javadoc present on public API methods?
- [ ] Are magic numbers extracted to named constants with explanation?
- [ ] Is any existing documentation updated to reflect the change?
- [ ] If a known limitation or tradeoff was accepted, is it noted?

### Design

- [ ] Does this class/method have a single responsibility?
- [ ] Are the dependencies correctly directed (no abstraction depending on detail)?
- [ ] Is there unnecessary duplication?
- [ ] Has a new abstraction been introduced when an existing one would serve?
- [ ] Are the class and method names accurate and descriptive?
- [ ] Is this change consistent with existing patterns in the codebase?

### Naming

- [ ] Do variable names reveal intent?
- [ ] Do method names describe what they do (not how)?
- [ ] Do class names reflect their responsibility?
- [ ] Are abbreviations avoided unless universally understood in context?

### Observability

- [ ] Are key operations logged at appropriate levels (INFO for business events, DEBUG for
  diagnostic)?
- [ ] Are errors logged with sufficient context (IDs, state, cause)?
- [ ] Are metrics emitted for new code paths that will be monitored?
- [ ] Is the log output free of sensitive data?

---

## 16. How to Give Effective Review Feedback

### The Four Levels of Review Feedback

Establish explicit levels so both reviewer and author understand the weight of each comment:

- **Nitpick (nit)**: A minor style or preference issue. The author can take it or leave it; it
  will not block the merge. Prefix with `nit:`.
- **Suggestion**: The reviewer has a recommendation that would improve the code but is not
  blocking. The author should consider it seriously but is not obligated to follow it. Prefix
  with `suggestion:`.
- **Concern**: The reviewer believes this is likely to cause problems — bugs, maintainability
  issues, performance degradation. It requires discussion before merging. Prefix with `concern:`.
- **Blocker**: The reviewer believes this must be addressed before merge. It is incorrect,
  insecure, or violates a hard constraint. Prefix with `blocker:`.

### Asking Questions vs Making Demands

Prefer asking questions over making demands, especially for anything below blocker level:

- Makes demands: "You need to use an interface here."
- Asks questions: "Would it be worth introducing an interface here to make this testable in
  isolation? I'm thinking of the scenario where we need to mock this in PaymentServiceTest."

The question form communicates the same information while respecting that the author may have
context the reviewer lacks. It invites dialogue rather than compliance.

### Praise What Deserves Praise

A code review that contains only criticism trains authors to view reviews as adversarial. When you
see something that is done particularly well — an elegant solution, a thorough test, a clear
abstraction — say so explicitly. This builds trust and signals what the team values.

### How to Handle Disagreements

When an author disagrees with a blocking comment:
1. Both parties should re-read the other's position carefully before responding.
2. If still in disagreement, bring in a third engineer to weigh in.
3. If still unresolved, escalate to the tech lead — but document the decision and reasoning.

Do not let disagreements stall a PR indefinitely. Make a decision, document it, and move forward.

### Full Examples: Poorly Worded vs Well-Worded Feedback

**Issue**: An engineer has written a method that queries the database inside a loop.

Poorly worded:
> "This is wrong. You're doing N+1 queries. Fix it."

This is accurate but provides no educational value, no explanation of why it is a problem, and no
guidance on how to fix it. It is also unnecessarily aggressive.

Well-worded:
> **blocker:** This loop calls `orderRepository.findById()` once per line item, which will produce
> N+1 database queries at runtime. For an order with 50 items, this is 51 queries where 1 would
> suffice. At scale this will cause significant database load and slow response times.
>
> Consider fetching all relevant records in a single query before the loop:
> ```java
> Set<Long> itemIds = lineItems.stream().map(LineItem::getItemId).collect(toSet());
> Map<Long, Product> products = productRepository.findAllByIdIn(itemIds)
>     .stream().collect(toMap(Product::getId, Function.identity()));
> ```
> Then use `products.get(item.getItemId())` inside the loop.

The well-worded version: explains the problem, gives the production impact, provides a concrete
suggestion, and communicates the severity clearly.

---

## 17. Architecture Review vs Code Review

### Architecture Review: Before Coding Starts

An architecture review evaluates the design of a solution before any code is written. It examines:
- The choice of components and their responsibilities.
- The communication patterns between components (sync vs async, REST vs events).
- The data model.
- The operational characteristics (scalability, fault tolerance, consistency guarantees).
- The impact on existing systems.

Architecture review is cheap because no code exists yet. Changing a design at this stage costs
hours. Changing it after implementation costs days or weeks.

Architecture reviews should be triggered by:
- New services or significant new components.
- Changes to data models that affect multiple services.
- Introduction of new technology or infrastructure.
- Changes to security boundaries.
- Work estimated at more than one sprint.

### Code Review: After Coding, for Implementation Quality

Code review evaluates the quality of an implementation against an already-approved design. It does
not re-debate the architecture. It verifies correctness, tests, naming, error handling, and
observability.

### Why Doing Architecture Review via Code Review Is Too Late

The most common failure mode in software teams is discovering architectural problems in code
review. "This approach won't scale" or "This creates a circular dependency between services" or
"This requires a shared database which we agreed to avoid" — these comments in a code review
represent wasted work. The engineer has already spent a week or two on an implementation that
needs to be rethought.

The engineering cost is not just the author's time. The reviewer's time, the integration test time,
the CI resources, the deployment to a staging environment — all of it is wasted.

The solution is to make architecture review a required step before significant work begins, with a
lightweight written artifact (a design doc or ADR) that can be reviewed asynchronously. The
investment is small relative to the cost of discovering architectural problems late.

---

# Part 4: Technical Leadership

## 18. Setting Design Standards for the Team

### Coding Conventions and Why They Matter

Coding conventions are not about aesthetics. They serve a concrete function: they reduce the
cognitive load required to read unfamiliar code. When every engineer on the team uses the same
patterns for error handling, the same structure for service classes, and the same approach to
logging, a reader can navigate any part of the codebase without first decoding each engineer's
personal style.

Conventions that matter at the team level:
- Exception hierarchy: when to use checked vs unchecked exceptions, and how to wrap them.
- Result representation: `Optional<T>`, throwing exceptions, or `Result<T, Error>` types.
- Service layer structure: constructor injection, how to handle transactions, logging format.
- Test structure: naming conventions (`givenX_whenY_thenZ`), setup patterns, fixture strategy.
- Logging format: what to include in log messages, how to include correlation IDs.

Conventions that do not matter enough to argue about (delegate to a linter):
- Braces on the same line vs next line.
- Import ordering.
- Maximum line length.

### Architecture Decision Records (ADRs)

An Architecture Decision Record is a short document that captures a significant architectural
decision: what was decided, why it was decided, and what alternatives were rejected. ADRs are
stored in the repository alongside the code they govern.

The value of ADRs:
- New team members can read the history of decisions and understand why the system is structured
  as it is.
- Teams stop relitigating the same decisions ("why do we use Result types instead of
  exceptions?").
- Engineers who join after a decision was made can evaluate whether the original reasoning still
  applies.

**ADR Template:**

```markdown
# ADR-{number}: {Title}

**Date**: {YYYY-MM-DD}
**Status**: {Proposed | Accepted | Deprecated | Superseded by ADR-XXX}
**Deciders**: {names or roles of engineers involved}

## Context

What is the problem or situation that requires a decision?
What are the constraints and forces at play?

## Decision

What was decided? State it clearly and unambiguously.

## Consequences

### Positive
What becomes easier or better as a result of this decision?

### Negative
What becomes harder or worse? What are the accepted tradeoffs?

### Neutral
What else changes as a result?

## Alternatives Considered

### Alternative 1: {name}
Why this was rejected.

### Alternative 2: {name}
Why this was rejected.

## References
Links to discussions, research, or related ADRs.
```

**Completed ADR Example: Use Result Type for Error Handling**

```markdown
# ADR-007: Use Result Type for Error Handling in Service Layer

**Date**: 2024-09-15
**Status**: Accepted
**Deciders**: Lead Engineers (Platform, Payments, Orders teams)

## Context

The service layer currently uses a mix of:
1. Checked exceptions (declared in method signatures with `throws`)
2. Unchecked exceptions (RuntimeException subclasses)
3. Optional<T> for not-found cases

This inconsistency creates two problems:
- Call sites cannot tell from the method signature whether an operation can fail
  and what kinds of failures are possible.
- Checked exceptions cause method signature pollution — every method in the call
  chain must declare or handle exceptions it cannot meaningfully handle.
- Error handling logic is scattered: sometimes in try/catch, sometimes in
  isPresent() checks, sometimes not at all.

Java does not have a native Result type, but the pattern (returning a type that
represents either a success value or an error) is well-established in functional
programming and available via third-party libraries.

## Decision

Service methods that can fail for business reasons (as opposed to programming
errors) will return a `Result<T, ServiceError>` type. We will use the
`dev.failsafe.Result` class from the Failsafe library, which is already in our
dependency tree for retry logic.

The convention:
- Use `Result<T, ServiceError>` when a method can fail for expected business
  reasons (payment declined, order not found, validation failure).
- Use unchecked exceptions for programming errors (null where prohibited,
  contract violations).
- Never use checked exceptions in the service layer.

ServiceError will be a sealed class hierarchy:
  - NotFoundError
  - ValidationError
  - ExternalServiceError
  - AuthorizationError

## Consequences

### Positive
- Method signatures are self-documenting: a caller can see what kinds of failures
  to handle without reading the body.
- No checked exception pollution through the call chain.
- Pattern matching on sealed class hierarchy provides exhaustiveness checking
  at the call site (Java 21 switch expressions).
- Testable: easy to verify that a method returns the expected error type.

### Negative
- Requires discipline: engineers may be tempted to throw exceptions anyway.
  Code review must enforce the convention.
- Third-party library dependency (minor; Failsafe is already present).
- Learning curve for engineers unfamiliar with Result types.
- Some boilerplate at call sites compared to try/catch.

### Neutral
- Existing service methods will be migrated opportunistically during feature work.
  No dedicated migration sprint.

## Alternatives Considered

### Alternative 1: Continue with mixed exceptions + Optional
Rejected because this is the status quo and we have identified the inconsistency
as a source of bugs (unhandled error cases) and maintainability issues.

### Alternative 2: Checked exceptions uniformly
Rejected because checked exceptions require every intermediate caller to declare
or handle exceptions, creating signature pollution and encouraging empty catch
blocks ("catch (ServiceException e) {}").

### Alternative 3: Build our own Result type
Rejected because Failsafe already provides a suitable implementation and adding
our own type increases maintenance burden.

## References
- Railway Oriented Programming (Scott Wlaschin)
- Failsafe library: https://failsafe.dev
- ADR-003: Retry and Circuit Breaker Strategy (Failsafe adoption)
```

### How to Get Buy-In from the Team

Design standards imposed top-down without consultation are resisted. Standards developed
collaboratively are owned. Practical approach:

1. **Identify the problem**: Show concrete examples of the inconsistency or the cost it is
   creating. Not "we should use Result types" but "look at these three incident post-mortems where
   swallowed exceptions caused data loss."
2. **Propose, do not mandate**: Circulate a draft ADR for discussion. Give the team time to comment.
3. **Acknowledge legitimate concerns**: If someone raises a valid objection, update the proposal.
4. **Accept that you will not get consensus**: Some engineers will disagree. After sufficient
   discussion, the tech lead makes a decision and documents it. The team follows it.
5. **Review in code review**: Standards are only effective if they are enforced in code review.
   Automation (linting, ArchUnit tests) is better than human enforcement where possible.

---

## 19. Design Review Process

### When to Require a Design Doc

Not every task needs a design doc. A design doc is appropriate when:
- The work will take more than one sprint.
- The work introduces a new service, database, or external dependency.
- The work changes a shared API that other teams depend on.
- The work has non-trivial security or data model implications.
- The approach is non-obvious and multiple reasonable solutions exist.

One-sprint tasks with a clear, obvious implementation do not need a design doc. The overhead of
writing and reviewing it exceeds the benefit.

### Design Doc Structure

A design doc does not need to be long. A 3-5 page document reviewed by 3-5 engineers is more
valuable than a 30-page document that nobody reads.

```
1. Background (1-2 paragraphs)
   - What is the problem being solved?
   - What are the current constraints?

2. Goals and Non-Goals
   - What does this change intend to accomplish?
   - What is explicitly out of scope?

3. Proposed Design (core of the document)
   - Architecture diagram if applicable
   - Component responsibilities
   - Data model changes
   - API contract (if applicable)
   - Error handling approach
   - Migration strategy (if replacing existing behavior)

4. Alternatives Considered
   - Why was each alternative rejected?

5. Open Questions
   - What decisions have not been made yet?

6. Risks and Mitigations

7. Test and Rollout Plan
```

### Async vs Sync Design Review

Async review (document shared for comments over 2-3 days) is appropriate for most design docs.
It allows reviewers to think carefully and respond without scheduling overhead.

Sync review (a meeting) is appropriate when:
- The design is complex enough that written comments will generate long back-and-forth threads.
- There are known disagreements to resolve.
- The design needs to be unblocked quickly.

The most effective pattern: share the design doc for async review first (24-48 hours), then hold
a 30-60 minute sync meeting to resolve open questions and make final decisions.

### How to Give Useful Design Feedback

Focus on:
- **Correctness**: Does the design solve the stated problem?
- **Scope creep**: Is the design solving problems that are not in scope?
- **Missing cases**: What happens when X fails? How is Y handled at scale?
- **Inconsistency**: Does this conflict with an existing pattern or decision?
- **Clarity**: Is the proposed design clear enough to implement from?

Avoid:
- Litigating technology choices that were settled in a prior ADR.
- Re-debating decisions that are clearly within the author's scope.
- Bikeshedding on names or structure that will be determined during implementation.

---

## 20. Documentation Culture

### What to Document and What Not To

The principle: document the **why**, not the **what**. The code already tells the reader what it
does. What the code cannot tell the reader is:
- Why this approach was chosen over obvious alternatives.
- What constraint or business rule this code is enforcing.
- What the performance characteristics are and why.
- What will break if this is changed in a seemingly benign way.

```java
// Poor documentation — restates the code
// Increments the counter by 1
counter++;

// Good documentation — explains the non-obvious constraint
// The retry counter must be incremented BEFORE the sleep to ensure that
// a timeout during sleep does not result in an extra retry attempt beyond
// the configured maximum. See PAYMENTS-2847 for the incident that led to this.
counter++;
Thread.sleep(backoffMs);
```

Documents not worth writing:
- Javadoc that says "Gets the name" on a method called `getName()`.
- READMEs that describe the code structure instead of how to run the service.
- Architecture diagrams that are not kept up to date (stale diagrams are actively harmful).

Documents worth writing:
- Runbooks: step-by-step guides for operational tasks (how to restart the service safely, how to
  roll back a migration, how to investigate a specific alarm).
- Decision logs (ADRs).
- API contracts (owned by the service team and kept up to date).
- Onboarding guides: what does a new engineer need to know to be productive in this codebase?

### Living Documentation: Tests as Documentation

A well-written test suite is the most reliable documentation in a codebase because it is
automatically verified to be accurate. Tests that fail to compile or pass are immediately obvious.
Stale documentation silently misleads.

A test named `orderWithGoldCustomerTierReceives15PercentDiscount` tells you, precisely and
verifiably, what the system does. A comment saying "gold customers get 15% discount" might still
be there after the discount was changed to 12%.

Write tests that serve as documentation:
- Test method names should be full sentences describing the scenario.
- Tests should cover the business rules, not just the code paths.
- Failure messages should explain what expectation was violated, not just which assertion failed.

```java
// Tests as documentation: each test is a specification of the system's behavior
class DiscountServiceTest {

    @Test
    void goldTierCustomersReceiveFifteenPercentDiscountOnAllOrders() { ... }

    @Test
    void silverTierCustomersReceiveEightPercentDiscountOnAllOrders() { ... }

    @Test
    void standardCustomersReceiveNoAutomaticDiscount() { ... }

    @Test
    void discountsDoNotApplyToTaxAndShippingFees() { ... }

    @Test
    void discountPercentageIsAppliedBeforeTaxCalculation() { ... }
}
```

### Decision Logs

A decision log is a lightweight, chronological record of decisions made during a project. Unlike
ADRs (which are architectural-scale decisions), a decision log captures smaller decisions: "we
decided to use UTC everywhere rather than storing timezone-aware timestamps," "we agreed to return
HTTP 422 for validation errors rather than 400."

Decision logs prevent the same discussion from happening five times over the life of a project.
They are typically maintained in a team wiki or as a section in the project's README.

### Runbooks

A runbook is an operational playbook: step-by-step instructions for handling a specific situation.
Good runbooks:
- Are written during development, not during an incident.
- Are tested periodically (actually follow the runbook and verify the steps work).
- Have a "last verified" date.
- Are linked from the monitoring alerts they are designed to address.

A runbook linked from an alert that fires at 2am is worth more than the most elegant code.

---

## 21. Senior vs Mid-level Thinking

| Dimension | Mid-level Engineer | Senior Engineer |
|-----------|-------------------|-----------------|
| **Relationship to technical debt** | Frustrated by it; wants to rewrite everything | Treats it as a managed liability; prioritizes by impact |
| **Response to a messy codebase** | "This needs a full rewrite" | "What is the highest-leverage thing I can improve before this next feature?" |
| **Refactoring approach** | Large, undivided refactoring that blocks feature work | Small, incremental steps that are individually shippable |
| **Test approach** | Writes tests after the code is working | Writes tests first (or alongside) as a design tool |
| **Code review focus** | Correctness and style | Design, testability, observability, and team patterns |
| **Review feedback style** | "This is wrong" | "This would cause N+1 queries at scale because...; consider this alternative:" |
| **Design documentation** | Skips design docs for "simple" changes | Writes a design doc when scope or uncertainty is high |
| **Response to a disagreement** | Insists on their approach or gives up | Seeks a third opinion; escalates constructively; documents the decision |
| **On architectural decisions** | Implements what is asked | Questions the approach before implementing; surfaces tradeoffs |
| **On coding standards** | Follows them when they agree | Follows them even when they disagree; advocates for changes through proper channels |
| **On technical debt paydown** | Wants a dedicated sprint to fix everything | Uses the boy scout rule; reserves dedicated sprints for structural debt only |
| **On feature toggles** | "Why do we need a flag? Just deploy it" | Defaults to gradual rollout for significant changes; toggle is the rollback plan |
| **On rewrites** | "Let's rewrite this in [new technology]" | "Let's strangle the old system incrementally while the new one is validated" |
| **On metrics** | "Our coverage is 85%, we're good" | "Coverage is a proxy. What behaviors are we not testing?" |
| **Bus factor awareness** | Becomes the only person who understands a system | Actively documents, reviews, and pair-programs to distribute knowledge |
| **Communication** | Communicates implementation details | Communicates design intent and tradeoffs |
| **On time pressure** | Skips tests and documentation under pressure | Explains explicitly what is being deferred and files the debt ticket |
| **On new technology** | Excited by novelty | Evaluates operational maturity, team expertise, and fit before adopting |
| **On legacy code** | Avoids it | Understands it, documents it, improves it incrementally |
| **Scope management** | Takes on whatever is asked | Pushes back on scope that creates unsustainable debt or risk |

---

## 22. Closing: The Senior Engineer Mindset

There is a particular failure mode that affects technically excellent engineers who have not yet
made the transition to senior engineering. They optimize for the elegance of their own code. They
are proud of clever solutions. They resist following conventions they consider suboptimal. They
hoard knowledge because being the expert feels valuable. This is not malice — it is a natural
consequence of the incentive structures that rewarded individual performance throughout their
career until this point.

The senior engineer operates differently, and the difference is not primarily technical.

A senior engineer optimizes for team velocity, not individual cleverness. The most elegant solution
that nobody else on the team can maintain or extend is not the right solution. The right solution
is the one that the whole team can understand, modify, and confidently ship. This sometimes means
choosing a slightly less clever approach because it uses a pattern the team already knows. The
constraint is not a compromise — it is a deliberate choice that maximizes the team's effective
output.

A senior engineer manages complexity as a first-class concern. Complexity is the primary enemy of
a software system's long-term health. Every abstraction, every dependency, every design choice
either reduces or adds to the system's total complexity. Senior engineers develop a strong negative
visceral reaction to unnecessary complexity — not because they cannot understand it, but because
they understand the long-term cost of it. They say no to clever features that add marginal value
at the cost of significant complexity. They say yes to simplification even when it feels like
admitting a past mistake.

A senior engineer thinks in systems and tradeoffs, not rules. "Always use interfaces" and "never
use exceptions" and "microservices are the right architecture" are not principles — they are
cargo-cultured heuristics. A senior engineer understands why these rules of thumb exist, and
therefore knows when they apply and when they do not. The decision is always: given this specific
context, these specific constraints, and this specific team, what is the right tradeoff? Answering
this question well requires understanding the domain, the operations, the team, and the history —
not just the technology.

A senior engineer communicates through code, documentation, and architecture. The code tells the
reader what the system does. The documentation tells the reader why. The architecture tells the
reader how the system fits together. A senior engineer considers all three as channels through
which they communicate with every engineer who will touch this code after them — which may be
dozens of people over the next decade. Writing code that speaks for itself, writing tests that
serve as specifications, and writing ADRs that explain the reasoning behind decisions are all forms
of communication that scale beyond the individual.

A senior engineer reduces the bus factor. The bus factor of a system is the number of people who
would need to be hit by a bus (or more charitably, leave the team) before the system becomes
unmaintainable. A bus factor of one is a liability. A senior engineer deliberately works to
increase the bus factor of the systems they own: through documentation, through code reviews that
transfer knowledge, through pair programming on critical paths, and through explicit cross-training.
They resist the comfortable role of "the only person who understands this service" because they
understand that this is a risk, not an asset.

A senior engineer says no to the right things. This is perhaps the most underrated skill. They say
no to features that solve hypothetical problems at the cost of real complexity. They say no to
architectural shortcuts that feel fast now and will be expensive for years. They say no to the
request to "just skip the tests this once." They say no by explaining the tradeoff clearly and
professionally, not by refusing to engage. They make the cost visible. Sometimes the team overrides
the no — and the senior engineer accepts that, implements the decision faithfully, and files the
debt ticket.

The through-line in all of this is a shift from a personal focus to a systemic one. The question
is no longer "is this code good?" but "does this code make the system healthier, the team more
effective, and the product more reliable?" That shift in perspective is what separates a technically
excellent engineer from a senior engineer.
