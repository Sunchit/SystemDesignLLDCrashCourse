# SOLID Principles - Comprehensive Assignment

**Course:** Low-Level Design and OOP for Senior Engineers  
**Target Audience:** Software Engineers with 2-8 years of experience preparing for Senior Engineer roles  
**Estimated Time:** 8-12 hours (can be split across multiple sessions)  
**Language:** Java

---

## How to Use This Assignment

This file is structured as a self-guided workshop. For each assignment:

1. **Read the provided code carefully.** The violation code is intentionally realistic - it looks like code you would find in a production codebase. Do not skim it.
2. **Identify the violations before reading the tasks.** Spend at least 10 minutes on your own analysis before looking at what the tasks ask. Write your observations in a separate document.
3. **Complete all tasks in order.** The tasks build on each other. Do not skip to the refactoring step without completing the identification step.
4. **Check your work against the rubric.** The rubric defines what "done" means. If you cannot score yourself on a criterion, you have not finished it.
5. **Compile and run your refactored code.** Every class you write must be compilable. Stubs are acceptable for external dependencies (database, email), but the structure must be sound.

**A note on the violation code:** The provided Java classes are complete and compilable. They represent a single class or hierarchy doing too much. Your job is to identify why this is a problem, articulate the problem precisely, and produce a better design. The explanation matters as much as the code.

---

## Assignment 1 - Single Responsibility Principle

### Background

The Single Responsibility Principle states that a class should have one, and only one, reason to change. "Reason to change" means an actor - a person or group of people who can demand a change to that class. If multiple different actors can demand changes to the same class, the class has multiple responsibilities and violates SRP.

### The Violation Code

Read the following class in its entirety before attempting any tasks.

```java
import java.sql.*;
import java.util.ArrayList;
import java.util.List;
import java.util.Properties;
import javax.mail.*;
import javax.mail.internet.*;

public class InvoiceManager {

    private static final String DB_URL = "jdbc:mysql://localhost:3306/billing";
    private static final String DB_USER = "root";
    private static final String DB_PASSWORD = "secret";
    private static final String SMTP_HOST = "smtp.company.com";
    private static final int SMTP_PORT = 587;

    // ----------------------------------------------------------------
    // INVOICE CREATION AND VALIDATION
    // ----------------------------------------------------------------

    public Invoice createInvoice(String clientId, List<LineItem> lineItems) {
        if (clientId == null || clientId.isBlank()) {
            throw new IllegalArgumentException("Client ID must not be blank");
        }
        if (lineItems == null || lineItems.isEmpty()) {
            throw new IllegalArgumentException("Invoice must have at least one line item");
        }
        for (LineItem item : lineItems) {
            if (item.getQuantity() <= 0) {
                throw new IllegalArgumentException("Line item quantity must be positive: " + item.getDescription());
            }
            if (item.getUnitPrice() < 0) {
                throw new IllegalArgumentException("Unit price cannot be negative: " + item.getDescription());
            }
        }
        double subtotal = lineItems.stream()
                .mapToDouble(i -> i.getQuantity() * i.getUnitPrice())
                .sum();
        double tax = calculateTax(subtotal, clientId);
        double total = subtotal + tax;

        Invoice invoice = new Invoice();
        invoice.setClientId(clientId);
        invoice.setLineItems(new ArrayList<>(lineItems));
        invoice.setSubtotal(subtotal);
        invoice.setTaxAmount(tax);
        invoice.setTotalAmount(total);
        invoice.setStatus("DRAFT");
        return invoice;
    }

    // ----------------------------------------------------------------
    // TAX CALCULATION
    // ----------------------------------------------------------------

    public double calculateTax(double subtotal, String clientId) {
        // Rule 1: Clients in the EU are subject to 20% VAT
        if (clientId.startsWith("EU-")) {
            return subtotal * 0.20;
        }
        // Rule 2: Clients in the US state of California are subject to 8.5% sales tax
        if (clientId.startsWith("US-CA-")) {
            return subtotal * 0.085;
        }
        // Rule 3: Clients in the US (non-CA) are subject to 6% federal tax
        if (clientId.startsWith("US-")) {
            return subtotal * 0.06;
        }
        // Rule 4: Government clients are tax-exempt
        if (clientId.startsWith("GOV-")) {
            return 0.0;
        }
        // Rule 5: Default international rate
        return subtotal * 0.15;
    }

    // ----------------------------------------------------------------
    // DATABASE OPERATIONS
    // ----------------------------------------------------------------

    public void saveInvoice(Invoice invoice) {
        String sql = "INSERT INTO invoices (client_id, subtotal, tax_amount, total_amount, status) VALUES (?, ?, ?, ?, ?)";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {

            stmt.setString(1, invoice.getClientId());
            stmt.setDouble(2, invoice.getSubtotal());
            stmt.setDouble(3, invoice.getTaxAmount());
            stmt.setDouble(4, invoice.getTotalAmount());
            stmt.setString(5, invoice.getStatus());
            stmt.executeUpdate();

            ResultSet keys = stmt.getGeneratedKeys();
            if (keys.next()) {
                invoice.setId(keys.getLong(1));
            }
        } catch (SQLException e) {
            throw new RuntimeException("Failed to save invoice", e);
        }
    }

    public void updateInvoiceStatus(long invoiceId, String newStatus) {
        String sql = "UPDATE invoices SET status = ? WHERE id = ?";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setString(1, newStatus);
            stmt.setLong(2, invoiceId);
            stmt.executeUpdate();
        } catch (SQLException e) {
            throw new RuntimeException("Failed to update invoice status", e);
        }
    }

    public Invoice findInvoiceById(long invoiceId) {
        String sql = "SELECT * FROM invoices WHERE id = ?";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setLong(1, invoiceId);
            ResultSet rs = stmt.executeQuery();
            if (rs.next()) {
                Invoice invoice = new Invoice();
                invoice.setId(rs.getLong("id"));
                invoice.setClientId(rs.getString("client_id"));
                invoice.setSubtotal(rs.getDouble("subtotal"));
                invoice.setTaxAmount(rs.getDouble("tax_amount"));
                invoice.setTotalAmount(rs.getDouble("total_amount"));
                invoice.setStatus(rs.getString("status"));
                return invoice;
            }
            return null;
        } catch (SQLException e) {
            throw new RuntimeException("Failed to find invoice", e);
        }
    }

    // ----------------------------------------------------------------
    // PDF GENERATION
    // ----------------------------------------------------------------

    public byte[] generateInvoicePdf(Invoice invoice) {
        StringBuilder sb = new StringBuilder();
        sb.append("=== INVOICE ===\n");
        sb.append("Invoice ID: ").append(invoice.getId()).append("\n");
        sb.append("Client: ").append(invoice.getClientId()).append("\n");
        sb.append("---\n");
        for (LineItem item : invoice.getLineItems()) {
            sb.append(String.format("%-30s x%d @ $%.2f = $%.2f%n",
                    item.getDescription(),
                    item.getQuantity(),
                    item.getUnitPrice(),
                    item.getQuantity() * item.getUnitPrice()));
        }
        sb.append("---\n");
        sb.append(String.format("Subtotal: $%.2f%n", invoice.getSubtotal()));
        sb.append(String.format("Tax:      $%.2f%n", invoice.getTaxAmount()));
        sb.append(String.format("TOTAL:    $%.2f%n", invoice.getTotalAmount()));
        // In a real implementation, this would use iText or Apache PDFBox
        // to produce an actual PDF binary. Here we return the text as bytes.
        return sb.toString().getBytes();
    }

    // ----------------------------------------------------------------
    // EMAIL SENDING
    // ----------------------------------------------------------------

    public void sendInvoiceToClient(Invoice invoice, String recipientEmail) {
        Properties props = new Properties();
        props.put("mail.smtp.host", SMTP_HOST);
        props.put("mail.smtp.port", String.valueOf(SMTP_PORT));
        props.put("mail.smtp.auth", "true");
        props.put("mail.smtp.starttls.enable", "true");

        Session session = Session.getInstance(props, new Authenticator() {
            protected PasswordAuthentication getPasswordAuthentication() {
                return new PasswordAuthentication("billing@company.com", "emailPassword123");
            }
        });

        try {
            byte[] pdfBytes = generateInvoicePdf(invoice);
            MimeMessage message = new MimeMessage(session);
            message.setFrom(new InternetAddress("billing@company.com"));
            message.setRecipient(Message.RecipientType.TO, new InternetAddress(recipientEmail));
            message.setSubject("Invoice #" + invoice.getId() + " from Company Inc.");

            MimeBodyPart textPart = new MimeBodyPart();
            textPart.setText("Please find your invoice attached. Total due: $" + invoice.getTotalAmount());

            MimeBodyPart attachmentPart = new MimeBodyPart();
            attachmentPart.setContent(pdfBytes, "application/pdf");
            attachmentPart.setFileName("invoice_" + invoice.getId() + ".pdf");

            Multipart multipart = new MimeMultipart();
            multipart.addBodyPart(textPart);
            multipart.addBodyPart(attachmentPart);
            message.setContent(multipart);

            Transport.send(message);
        } catch (MessagingException e) {
            throw new RuntimeException("Failed to send invoice email", e);
        }
    }
}
```

### Supporting Classes (provided for context, not to be refactored)

```java
public class Invoice {
    private long id;
    private String clientId;
    private List<LineItem> lineItems = new ArrayList<>();
    private double subtotal;
    private double taxAmount;
    private double totalAmount;
    private String status;

    // getters and setters omitted for brevity - assume standard JavaBeans style
    public long getId() { return id; }
    public void setId(long id) { this.id = id; }
    public String getClientId() { return clientId; }
    public void setClientId(String clientId) { this.clientId = clientId; }
    public List<LineItem> getLineItems() { return lineItems; }
    public void setLineItems(List<LineItem> lineItems) { this.lineItems = lineItems; }
    public double getSubtotal() { return subtotal; }
    public void setSubtotal(double subtotal) { this.subtotal = subtotal; }
    public double getTaxAmount() { return taxAmount; }
    public void setTaxAmount(double taxAmount) { this.taxAmount = taxAmount; }
    public double getTotalAmount() { return totalAmount; }
    public void setTotalAmount(double totalAmount) { this.totalAmount = totalAmount; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
}

public class LineItem {
    private String description;
    private int quantity;
    private double unitPrice;

    public LineItem(String description, int quantity, double unitPrice) {
        this.description = description;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
    }
    public String getDescription() { return description; }
    public int getQuantity() { return quantity; }
    public double getUnitPrice() { return unitPrice; }
}
```

---

### Task 1.1 - Identify All SRP Violations

For each responsibility you identify in `InvoiceManager`, answer the following questions in writing:

1. What is the responsibility?
2. Which actor (person or group) would demand a change to this responsibility independently of the other actors?
3. Give a concrete example of a change that actor would request that would have no relevance to the other actors.

You should identify at least five distinct responsibilities and five distinct actors.

**Hint:** Think about who in an organization would raise a ticket requesting a change to that code:
- Would the finance/accounting team care about PDF layout?
- Would the DBA care about tax calculation rules?
- Would the devops team care about business validation rules?

---

### Task 1.2 - Refactor into Focused Classes

Redesign `InvoiceManager` by splitting its responsibilities into separate classes. Your refactored design must:

1. Define an interface for each major responsibility (repository, PDF generation, email, tax calculation).
2. Implement each interface in a concrete class.
3. Create a new `InvoiceService` (or `InvoiceFacade`) class that coordinates the focused classes via constructor injection. This coordinator class is the only class that should depend on multiple interfaces.
4. Each class in your design must have exactly one reason to change.

**Expected interface and class names (you may deviate with justification):**

```
IInvoiceRepository         -> JdbcInvoiceRepository
IInvoicePdfGenerator       -> TextInvoicePdfGenerator
IInvoiceEmailSender        -> SmtpInvoiceEmailSender
ITaxCalculator             -> RegionBasedTaxCalculator
InvoiceValidator           (no interface needed - internal concern)
InvoiceService             (the coordinator)
```

Write the full implementation of each class. For infrastructure classes (JDBC, SMTP), you may keep the same implementation logic from the violation code - the goal is structural separation, not re-implementing the infrastructure.

**Show a usage example:**

```java
// In a main method or test, demonstrate how the refactored classes wire together
// and how InvoiceService is used.
```

---

### Task 1.3 - Rubric

| Criterion | Points | Description |
|---|---|---|
| Violation identification | 20 | Correctly identifies all five responsibilities and names the correct actor for each with a concrete change example |
| Interface design | 20 | Each interface models a single responsibility, has a meaningful name, and contains only methods related to that responsibility |
| Implementation correctness | 20 | Each concrete class fully implements its interface and contains only code relevant to its single responsibility |
| Coordinator class | 20 | `InvoiceService` depends only on interfaces (not concrete classes), receives dependencies via constructor, and contains no infrastructure code of its own |
| Compilation and usage | 20 | All classes compile, interfaces are used as types throughout, and the usage example demonstrates wiring |

**Total: 100 points**

---

## Assignment 2 - Open/Closed Principle

### Background

The Open/Closed Principle states that software entities should be open for extension but closed for modification. In practice, this means that adding new behavior should not require modifying existing, tested code. When a class must be modified every time a new variant of behavior is needed, it is violating OCP.

The most common OCP violation pattern is a large `switch` or `if/else-if` chain that dispatch on a type tag. Every new type requires cracking open the existing class and risking regressions.

### The Violation Code

```java
public class ShippingCostCalculator {

    // Shipping type constants
    public static final String STANDARD = "STANDARD";
    public static final String EXPRESS = "EXPRESS";
    public static final String OVERNIGHT = "OVERNIGHT";
    public static final String INTERNATIONAL = "INTERNATIONAL";
    public static final String DRONE = "DRONE";

    /**
     * Calculates the shipping cost for an order.
     *
     * @param shippingType   one of the type constants defined above
     * @param weightKg       the weight of the package in kilograms
     * @param distanceKm     the distance to the destination in kilometers
     * @param isFragile      whether the package requires fragile handling
     * @return               the total shipping cost in USD
     */
    public double calculateShippingCost(String shippingType, double weightKg,
                                        double distanceKm, boolean isFragile) {
        double cost;

        if (STANDARD.equals(shippingType)) {
            // Standard shipping: $2 base + $0.50 per kg + $0.01 per km
            cost = 2.0 + (weightKg * 0.50) + (distanceKm * 0.01);
            if (isFragile) {
                cost += 3.00; // fragile surcharge
            }

        } else if (EXPRESS.equals(shippingType)) {
            // Express shipping: $5 base + $1.00 per kg + $0.02 per km
            cost = 5.0 + (weightKg * 1.00) + (distanceKm * 0.02);
            if (isFragile) {
                cost += 5.00; // higher fragile surcharge for express
            }
            // Express has a minimum charge of $15
            if (cost < 15.0) {
                cost = 15.0;
            }

        } else if (OVERNIGHT.equals(shippingType)) {
            // Overnight shipping: $15 base + $2.00 per kg + $0.05 per km
            cost = 15.0 + (weightKg * 2.00) + (distanceKm * 0.05);
            if (isFragile) {
                cost += 8.00;
            }
            // Overnight has a minimum charge of $30
            if (cost < 30.0) {
                cost = 30.0;
            }

        } else if (INTERNATIONAL.equals(shippingType)) {
            // International shipping: $25 base + $3.00 per kg + $0.08 per km
            // Plus a 15% customs processing fee on the subtotal
            cost = 25.0 + (weightKg * 3.00) + (distanceKm * 0.08);
            cost = cost * 1.15; // customs fee
            if (isFragile) {
                cost += 12.00;
            }
            // International has a minimum charge of $50
            if (cost < 50.0) {
                cost = 50.0;
            }

        } else if (DRONE.equals(shippingType)) {
            // Drone delivery: $10 base + $0.80 per kg (distance not relevant for drones)
            // Drones cannot handle packages over 5kg
            if (weightKg > 5.0) {
                throw new IllegalArgumentException("Drone delivery is limited to packages under 5kg. Package weight: " + weightKg);
            }
            cost = 10.0 + (weightKg * 0.80);
            // Drones cannot handle fragile items - operator must choose different method
            if (isFragile) {
                throw new IllegalArgumentException("Drone delivery does not support fragile packages");
            }

        } else {
            throw new IllegalArgumentException("Unknown shipping type: " + shippingType);
        }

        return Math.round(cost * 100.0) / 100.0; // round to 2 decimal places
    }

    /**
     * Returns a human-readable description of the shipping type's SLA.
     */
    public String getShippingSla(String shippingType) {
        if (STANDARD.equals(shippingType)) {
            return "5-7 business days";
        } else if (EXPRESS.equals(shippingType)) {
            return "2-3 business days";
        } else if (OVERNIGHT.equals(shippingType)) {
            return "Next business day by 10:00 AM";
        } else if (INTERNATIONAL.equals(shippingType)) {
            return "10-21 business days (customs clearance may apply)";
        } else if (DRONE.equals(shippingType)) {
            return "Same day, within 4 hours";
        } else {
            throw new IllegalArgumentException("Unknown shipping type: " + shippingType);
        }
    }
}
```

---

### Task 2.1 - Identify the OCP Violation

Answer the following questions in writing:

1. Which part of `ShippingCostCalculator` violates OCP, and what is the precise mechanism of the violation?
2. Imagine the product team wants to add a `LOCKER` shipping type (drop-off at a pickup locker, flat rate of $3 plus $0.30 per kg, no distance component, no fragile support). List every line of code that must be modified in the existing class to add this type. How many distinct places did you count?
3. What is the risk of this approach in a production codebase where `ShippingCostCalculator` is used by 20 different services and has 150 unit tests? Explain concretely what can go wrong.
4. The string-based type system (`"STANDARD"`, `"EXPRESS"`, etc.) is a separate but related problem. Explain what it is and why it compounds the OCP violation.

---

### Task 2.2 - Refactor Using the Strategy Pattern

Redesign `ShippingCostCalculator` so that:

1. There is a `ShippingStrategy` interface with the necessary methods.
2. Each shipping type is implemented as its own class that implements `ShippingStrategy`.
3. The `ShippingCostCalculator` (or a renamed equivalent) depends only on the `ShippingStrategy` interface and does not contain any `if/else` or `switch` on shipping types.
4. Adding a new shipping type (`LOCKER`) requires **only** adding a new class - no modification of any existing class.

**Required interface design (implement in full):**

```java
public interface ShippingStrategy {
    double calculateCost(double weightKg, double distanceKm, boolean isFragile);
    String getSla();
    String getTypeName();
}
```

Write the full implementation of:
- `ShippingStrategy` interface
- `StandardShippingStrategy`
- `ExpressShippingStrategy`
- `OvernightShippingStrategy`
- `InternationalShippingStrategy`
- `DroneShippingStrategy`
- `LockerShippingStrategy` (the new type, to prove OCP is satisfied)
- The refactored `ShippingCostCalculator` class

---

### Task 2.3 - Bonus: Factory

Write a `ShippingStrategyFactory` class that:

1. Maintains a registry of available strategies (use a `Map<String, ShippingStrategy>`).
2. Provides a `getStrategy(String typeName)` method that returns the correct strategy.
3. Provides a `registerStrategy(ShippingStrategy strategy)` method that allows registering new strategies at runtime without modifying the factory.
4. Show how `LockerShippingStrategy` can be registered and used without modifying any existing class.

---

### Task 2.4 - Rubric

| Criterion | Points | Description |
|---|---|---|
| Violation identification | 20 | Correctly names the OCP violation, counts all modification points for adding a new type, explains production risk, and identifies the string-type problem |
| Strategy interface design | 20 | Interface is minimal and complete; all necessary methods are present; no leaking of implementation details into the interface |
| Strategy implementations | 20 | Each strategy class correctly implements its logic from the original; no cross-strategy logic appears in any single strategy |
| Refactored calculator | 20 | `ShippingCostCalculator` contains no conditional logic on type; depends only on the interface; closed for modification |
| Factory and extensibility | 20 | Factory uses a registry pattern; `registerStrategy` allows extension without modification; `LockerShippingStrategy` demo proves OCP is satisfied |

**Total: 100 points**

---

## Assignment 3 - Liskov Substitution Principle

### Background

The Liskov Substitution Principle states that if S is a subtype of T, then objects of type T may be replaced with objects of type S without altering any of the desirable properties of the program. More practically: a subclass must honor the behavioral contract established by its superclass. This means:

- **Preconditions** cannot be strengthened in a subtype (you cannot demand more from the caller than the base class demanded).
- **Postconditions** cannot be weakened in a subtype (you cannot deliver less to the caller than the base class promised).
- **Invariants** must be preserved (properties that hold for the base class must hold for all subtypes).
- A subtype must not throw exceptions that the base class contract does not allow.

### The Violation Code

```java
public class Vehicle {

    protected String make;
    protected String model;
    protected int year;
    protected boolean engineRunning;
    protected int speedKmh;

    public Vehicle(String make, String model, int year) {
        this.make = make;
        this.model = model;
        this.year = year;
        this.engineRunning = false;
        this.speedKmh = 0;
    }

    /**
     * Starts the engine.
     * Postcondition: engineRunning == true after this call.
     * Postcondition: speedKmh remains unchanged.
     */
    public void startEngine() {
        System.out.println("Starting engine for " + make + " " + model);
        this.engineRunning = true;
    }

    /**
     * Accelerates the vehicle.
     * Precondition: engineRunning must be true.
     * Precondition: targetSpeedKmh must be >= 0 and >= current speed.
     * Postcondition: speedKmh == targetSpeedKmh after this call.
     * Postcondition: engineRunning remains true.
     */
    public void accelerate(int targetSpeedKmh) {
        if (!engineRunning) {
            throw new IllegalStateException("Cannot accelerate: engine is not running");
        }
        if (targetSpeedKmh < 0) {
            throw new IllegalArgumentException("Target speed cannot be negative");
        }
        if (targetSpeedKmh < speedKmh) {
            throw new IllegalArgumentException("Use brake() to decelerate. accelerate() only increases speed.");
        }
        System.out.println(make + " " + model + " accelerating to " + targetSpeedKmh + " km/h");
        this.speedKmh = targetSpeedKmh;
    }

    /**
     * Refuels the vehicle.
     * Precondition: fuelLitres must be > 0.
     * Postcondition: vehicle's fuel level increases by fuelLitres.
     */
    public void refuel(double fuelLitres) {
        if (fuelLitres <= 0) {
            throw new IllegalArgumentException("Fuel amount must be positive");
        }
        System.out.println("Refuelling " + make + " " + model + " with " + fuelLitres + " litres");
        // In a real class, this would update a fuelLevel field
    }

    public String getMake() { return make; }
    public String getModel() { return model; }
    public int getYear() { return year; }
    public boolean isEngineRunning() { return engineRunning; }
    public int getSpeedKmh() { return speedKmh; }
}


public class ElectricCar extends Vehicle {

    private double batteryChargePercent;

    public ElectricCar(String make, String model, int year, double initialCharge) {
        super(make, model, year);
        this.batteryChargePercent = initialCharge;
    }

    /**
     * Electric cars do not have an engine. This method activates the drive system.
     * Note: the postcondition engineRunning == true is maintained to satisfy the superclass.
     */
    @Override
    public void startEngine() {
        System.out.println("Activating electric drive system for " + make + " " + model);
        this.engineRunning = true;
        // Side effect not present in base class: battery check
        if (batteryChargePercent < 5.0) {
            throw new IllegalStateException("Battery critically low. Charge before driving.");
        }
    }

    /**
     * LSP VIOLATION: Electric cars cannot be refuelled with fuel.
     * This throws an exception that the base class contract does not permit.
     */
    @Override
    public void refuel(double fuelLitres) {
        throw new UnsupportedOperationException(
            "Electric vehicles cannot be refuelled. Use charge() instead.");
    }

    /**
     * LSP VIOLATION: This override weakens the postcondition.
     * The base class guarantees speedKmh == targetSpeedKmh after the call.
     * This class may not reach targetSpeedKmh if battery is below 20%.
     */
    @Override
    public void accelerate(int targetSpeedKmh) {
        if (!engineRunning) {
            throw new IllegalStateException("Cannot accelerate: drive system is not active");
        }
        if (targetSpeedKmh < 0) {
            throw new IllegalArgumentException("Target speed cannot be negative");
        }
        if (targetSpeedKmh < speedKmh) {
            throw new IllegalArgumentException("Use brake() to decelerate.");
        }
        if (batteryChargePercent < 20.0) {
            // VIOLATION: Partially satisfies the postcondition - only goes to half the requested speed
            int reducedSpeed = targetSpeedKmh / 2;
            System.out.println("Low battery: limited acceleration to " + reducedSpeed + " km/h");
            this.speedKmh = reducedSpeed;
        } else {
            this.speedKmh = targetSpeedKmh;
        }
    }

    public void charge(double energyKwh) {
        if (energyKwh <= 0) {
            throw new IllegalArgumentException("Energy amount must be positive");
        }
        double chargeAdded = Math.min(energyKwh * 10, 100.0 - batteryChargePercent);
        this.batteryChargePercent += chargeAdded;
        System.out.println("Charged " + make + " " + model + ": battery now at " + batteryChargePercent + "%");
    }

    public double getBatteryChargePercent() { return batteryChargePercent; }
}


public class SportsCar extends Vehicle {

    private boolean turboBoostActive;

    public SportsCar(String make, String model, int year) {
        super(make, model, year);
        this.turboBoostActive = false;
    }

    /**
     * LSP VIOLATION: Strengthens the precondition.
     * The base class allows accelerating to any non-negative speed >= current speed.
     * This class adds an additional precondition: speed must be <= 300 km/h.
     * A caller operating against the Vehicle contract has no way to know this limit.
     */
    @Override
    public void accelerate(int targetSpeedKmh) {
        if (!engineRunning) {
            throw new IllegalStateException("Cannot accelerate: engine is not running");
        }
        if (targetSpeedKmh < 0) {
            throw new IllegalArgumentException("Target speed cannot be negative");
        }
        if (targetSpeedKmh < speedKmh) {
            throw new IllegalArgumentException("Use brake() to decelerate.");
        }
        // VIOLATION: Additional precondition not present in base class
        if (targetSpeedKmh > 300) {
            throw new IllegalArgumentException("SportsCar maximum speed is 300 km/h. Requested: " + targetSpeedKmh);
        }
        if (targetSpeedKmh > 200) {
            this.turboBoostActive = true;
            System.out.println("Turbo boost engaged!");
        }
        this.speedKmh = targetSpeedKmh;
    }

    public boolean isTurboBoostActive() { return turboBoostActive; }
}


// A client method that should work correctly for any Vehicle
public class FleetManager {

    public void performRoutineCheck(List<Vehicle> fleet) {
        for (Vehicle vehicle : fleet) {
            System.out.println("Checking: " + vehicle.getMake() + " " + vehicle.getModel());

            // Start all vehicles
            vehicle.startEngine();

            // Accelerate all vehicles to highway speed
            vehicle.accelerate(120); // BREAKS for ElectricCar if battery < 20%

            // Refuel all vehicles
            vehicle.refuel(40.0); // THROWS UnsupportedOperationException for ElectricCar

            System.out.println("Check complete for " + vehicle.getMake());
        }
    }
}
```

---

### Task 3.1 - Identify All LSP Violations

For each violation, provide a precise analysis using the pre/postcondition framework:

1. **ElectricCar.refuel()**: Identify the violation type (precondition strengthening, postcondition weakening, or invariant violation). Quote the base class contract. Explain what a caller depending on `Vehicle` would expect and what it actually gets.

2. **ElectricCar.startEngine()**: Identify the issue with the additional battery check. Is this a precondition strengthening, postcondition weakening, or both? Explain why this is subtle and why it matters.

3. **ElectricCar.accelerate()**: Identify the postcondition that is violated. Quote the postcondition from the base class. Show with a concrete example (specific battery % and target speed) how the caller's expectation is broken.

4. **SportsCar.accelerate()**: Identify the precondition that is strengthened. Show with a concrete example (specific target speed) how code that correctly uses `Vehicle` would fail when the `Vehicle` is actually a `SportsCar`.

5. **FleetManager.performRoutineCheck()**: Explain why this method is the "canary in the coal mine" for LSP. If the hierarchy were correct, what should this method be able to assume?

---

### Task 3.2 - Redesign the Hierarchy

Redesign the `Vehicle` hierarchy to eliminate all LSP violations. Your design must:

1. Keep `Vehicle` as a base class or abstract class with only behavior that all vehicles share unconditionally.
2. Use interfaces to model optional capabilities (`IFuelPowered`, `IElectricPowered`, or similar). Every method on an interface must be implementable by every class that implements the interface without exceptions or weakened postconditions.
3. Eliminate `UnsupportedOperationException` from all method implementations.
4. Ensure that `FleetManager.performRoutineCheck()` (or a redesigned equivalent) works correctly for all vehicle types using only the common base interface.

**Provide:**
- All interface definitions with full Javadoc contracts
- `Vehicle` abstract class (or interface)
- `ElectricCar` implementation
- `SportsCar` implementation
- Refactored `FleetManager` that does not break for any vehicle type

**Hint for the hierarchy design:**

Consider separating what all vehicles can do (move, stop, report status) from what only certain vehicles can do (refuel, charge). The `FleetManager.performRoutineCheck()` should only call methods on the common base. Type-specific operations should be handled with `instanceof` checks (acceptable in this case) or by extracting separate managers for fuel and electric vehicles.

---

### Task 3.3 - Rubric

| Criterion | Points | Description |
|---|---|---|
| Violation identification | 25 | All four method-level violations are correctly identified with the correct violation type (pre/postcondition/invariant) and concrete examples |
| FleetManager analysis | 15 | Correctly explains what assumptions FleetManager makes and which ones are broken by each subclass |
| Interface segregation in hierarchy | 20 | `IFuelPowered` and `IElectricPowered` (or equivalent) correctly model optional capabilities; no class implements a method it cannot fulfill |
| Refactored implementations | 25 | ElectricCar and SportsCar implementations honor the contracts of every interface/class they implement; no UnsupportedOperationException; postconditions are met |
| FleetManager correctness | 15 | Refactored FleetManager works for all vehicle types without exceptions or broken postconditions |

**Total: 100 points**

---

## Assignment 4 - Interface Segregation Principle

### Background

The Interface Segregation Principle states that no client should be forced to depend on methods it does not use. A "fat" interface - one that bundles together methods from multiple unrelated capabilities - forces implementing classes to provide dummy or throwing implementations for capabilities they do not have. This creates brittleness, misleads readers of the code, and makes the system harder to extend.

The key question to ask about any interface: "Can every class that might reasonably implement this interface provide a meaningful implementation of every method?" If the answer is no, the interface is too fat.

### The Violation Code

```java
public interface IAnimalBehavior {
    void eat();
    void sleep();
    void fly();
    void swim();
    void run();
    void climb();
    void makeSound();
    void hibernate();
    void layEggs();
}


public class Dog implements IAnimalBehavior {

    private String name;

    public Dog(String name) {
        this.name = name;
    }

    @Override
    public void eat() {
        System.out.println(name + " is eating dog food");
    }

    @Override
    public void sleep() {
        System.out.println(name + " is sleeping");
    }

    @Override
    public void fly() {
        // VIOLATION: Dogs cannot fly
        throw new UnsupportedOperationException("Dogs cannot fly");
    }

    @Override
    public void swim() {
        // Dogs can actually swim, so this is a valid implementation
        System.out.println(name + " is paddling in the water");
    }

    @Override
    public void run() {
        System.out.println(name + " is running");
    }

    @Override
    public void climb() {
        // VIOLATION: Dogs cannot climb trees
        throw new UnsupportedOperationException("Dogs cannot climb");
    }

    @Override
    public void makeSound() {
        System.out.println(name + " barks: Woof!");
    }

    @Override
    public void hibernate() {
        // VIOLATION: Dogs do not hibernate
        throw new UnsupportedOperationException("Dogs do not hibernate");
    }

    @Override
    public void layEggs() {
        // VIOLATION: Dogs are mammals, they do not lay eggs
        throw new UnsupportedOperationException("Dogs do not lay eggs");
    }
}


public class Eagle implements IAnimalBehavior {

    private String species;

    public Eagle(String species) {
        this.species = species;
    }

    @Override
    public void eat() {
        System.out.println(species + " eagle is hunting and eating");
    }

    @Override
    public void sleep() {
        System.out.println(species + " eagle is roosting");
    }

    @Override
    public void fly() {
        System.out.println(species + " eagle is soaring at high altitude");
    }

    @Override
    public void swim() {
        // VIOLATION: Eagles do not swim
        throw new UnsupportedOperationException("Eagles do not swim");
    }

    @Override
    public void run() {
        // VIOLATION: Eagles do not run - they hop or walk awkwardly
        throw new UnsupportedOperationException("Eagles do not run");
    }

    @Override
    public void climb() {
        System.out.println(species + " eagle is perching and climbing rocks");
    }

    @Override
    public void makeSound() {
        System.out.println(species + " eagle calls: Screech!");
    }

    @Override
    public void hibernate() {
        // VIOLATION: Eagles do not hibernate
        throw new UnsupportedOperationException("Eagles do not hibernate");
    }

    @Override
    public void layEggs() {
        System.out.println(species + " eagle is laying eggs in its nest");
    }
}


public class Bear implements IAnimalBehavior {

    private String species;

    public Bear(String species) {
        this.species = species;
    }

    @Override
    public void eat() {
        System.out.println(species + " bear is foraging for food");
    }

    @Override
    public void sleep() {
        System.out.println(species + " bear is sleeping");
    }

    @Override
    public void fly() {
        // VIOLATION: Bears cannot fly
        throw new UnsupportedOperationException("Bears cannot fly");
    }

    @Override
    public void swim() {
        System.out.println(species + " bear is swimming across the river");
    }

    @Override
    public void run() {
        System.out.println(species + " bear is running at 48 km/h");
    }

    @Override
    public void climb() {
        System.out.println(species + " bear is climbing a tree");
    }

    @Override
    public void makeSound() {
        System.out.println(species + " bear growls");
    }

    @Override
    public void hibernate() {
        System.out.println(species + " bear is entering torpor for winter");
    }

    @Override
    public void layEggs() {
        // VIOLATION: Bears are mammals, they do not lay eggs
        throw new UnsupportedOperationException("Bears do not lay eggs");
    }
}


// A client that iterates animals and calls various behaviors
public class AnimalSimulator {

    public void simulateDay(List<IAnimalBehavior> animals) {
        for (IAnimalBehavior animal : animals) {
            animal.eat();
            animal.sleep();
            animal.makeSound();
            // The following calls will crash at runtime for animals that do not support them:
            animal.fly();    // crashes for Dog and Bear
            animal.swim();   // crashes for Eagle
            animal.hibernate(); // crashes for Dog and Eagle
        }
    }
}
```

---

### Task 4.1 - Identify All ISP Violations

1. Create a matrix showing which animals support which behaviors. Use the following format:

```
Behavior      | Dog | Eagle | Bear
--------------|-----|-------|-----
eat()         |  Y  |   Y   |  Y
sleep()       |  Y  |   Y   |  Y
fly()         |  N  |   Y   |  N
swim()        |  Y  |   N   |  Y
run()         |  Y  |   N   |  Y
climb()       |  N  |   Y   |  Y
makeSound()   |  Y  |   Y   |  Y
hibernate()   |  N  |   N   |  Y
layEggs()     |  N  |   Y   |  N
```

2. For each behavior that is not universally supported, explain: which classes are forced to provide a throwing implementation, and what the downstream consequence is when `AnimalSimulator` calls that behavior on a list containing those animals.

3. Identify the two behaviors that all animals share (and therefore could legitimately remain in a common interface).

4. Explain why `UnsupportedOperationException` in an interface implementation is a code smell that signals an ISP violation. What is the alternative that ISP prescribes?

---

### Task 4.2 - Redesign into Focused Interfaces

Split `IAnimalBehavior` into focused role interfaces. Your redesign must:

1. Define a minimal common base interface (only behaviors all animals share unconditionally).
2. Define focused capability interfaces for locomotion types and special behaviors.
3. Implement `Dog`, `Eagle`, and `Bear` implementing only the interfaces that apply to them.
4. Redesign `AnimalSimulator` so that it never calls a behavior on an animal that does not support it - without using `instanceof` in the main simulation loop.

**Suggested interface breakdown (you may adjust with justification):**

```java
IAnimal       // eat(), sleep(), makeSound() - universal
ICanFly       // fly()
ICanSwim      // swim()
ICanRun       // run()
ICanClimb     // climb()
ICanHibernate // hibernate()
ILaysEggs     // layEggs()
```

Write the full definitions of all interfaces. Write the full implementations of `Dog`, `Eagle`, and `Bear`. Write the refactored `AnimalSimulator` demonstrating a simulation that is correct for all animal types.

**Hint for AnimalSimulator:** If you have separate lists - `List<ICanFly> fliers`, `List<ICanSwim> swimmers`, etc. - populated at construction time, you can call each behavior only on animals that implement the corresponding interface, without instanceof checks in the simulation loop.

---

### Task 4.3 - Rubric

| Criterion | Points | Description |
|---|---|---|
| Violation matrix and analysis | 20 | Matrix is complete and correct; each unsupported behavior correctly identifies which classes are forced to throw; downstream consequence explained |
| UnsupportedOperationException analysis | 15 | Correctly explains why throwing implementations signal ISP violations; explains what ISP prescribes as the alternative |
| Interface breakdown | 25 | Interfaces are focused (single responsibility); no interface contains behaviors not all implementors can support; naming is clear |
| Implementations | 25 | Dog, Eagle, and Bear implement exactly the right interfaces; no throwing implementations; all real behaviors are correctly coded |
| AnimalSimulator redesign | 15 | Simulation loop is type-safe; no instanceof in the main loop; all behaviors are called only on animals that support them |

**Total: 100 points**

---

## Assignment 5 - Dependency Inversion Principle

### Background

The Dependency Inversion Principle states:
1. High-level modules should not depend on low-level modules. Both should depend on abstractions.
2. Abstractions should not depend on details. Details (concrete implementations) should depend on abstractions.

In practice, this means that a business logic class (a high-level module) should not directly instantiate or name concrete infrastructure classes (low-level modules) such as database drivers, email services, or SMS providers. When it does, the business logic becomes entangled with infrastructure decisions, making it impossible to test in isolation and fragile to vendor changes.

The most common DIP violation is `new ConcreteClass()` inside a business logic class.

### The Violation Code

```java
public class MySQLUserRepository {

    private static final String DB_URL = "jdbc:mysql://prod-db.internal:3306/users";
    private static final String DB_USER = "app_user";
    private static final String DB_PASSWORD = "prodPassword456";

    public void save(User user) {
        String sql = "INSERT INTO users (username, email, password_hash, created_at) VALUES (?, ?, ?, NOW())";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            stmt.setString(1, user.getUsername());
            stmt.setString(2, user.getEmail());
            stmt.setString(3, user.getPasswordHash());
            stmt.executeUpdate();
            ResultSet keys = stmt.getGeneratedKeys();
            if (keys.next()) {
                user.setId(keys.getLong(1));
            }
        } catch (SQLException e) {
            throw new RuntimeException("Failed to save user", e);
        }
    }

    public boolean existsByEmail(String email) {
        String sql = "SELECT COUNT(*) FROM users WHERE email = ?";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            stmt.setString(1, email);
            ResultSet rs = stmt.executeQuery();
            return rs.next() && rs.getInt(1) > 0;
        } catch (SQLException e) {
            throw new RuntimeException("Failed to check email existence", e);
        }
    }
}


public class SendGridEmailService {

    private static final String SENDGRID_API_KEY = "SG.prod_api_key_hardcoded_here";

    public void sendWelcomeEmail(String recipientEmail, String username) {
        // In a real implementation, this would call the SendGrid HTTP API
        System.out.println("[SendGrid] Sending welcome email to: " + recipientEmail);
        System.out.println("[SendGrid] Subject: Welcome to our platform, " + username + "!");
        System.out.println("[SendGrid] API Key used: " + SENDGRID_API_KEY);
        // HTTP call to https://api.sendgrid.com/v3/mail/send would happen here
    }
}


public class TwilioSMSService {

    private static final String TWILIO_ACCOUNT_SID = "ACprod_account_sid_hardcoded";
    private static final String TWILIO_AUTH_TOKEN = "prod_auth_token_hardcoded";
    private static final String FROM_PHONE = "+15551234567";

    public void sendVerificationSMS(String phoneNumber, String verificationCode) {
        // In a real implementation, this would call the Twilio REST API
        System.out.println("[Twilio] Sending SMS to: " + phoneNumber);
        System.out.println("[Twilio] Verification code: " + verificationCode);
        System.out.println("[Twilio] Account SID: " + TWILIO_ACCOUNT_SID);
        // HTTP call to Twilio API would happen here
    }
}


public class UserRegistrationService {

    // DIP VIOLATION: Direct instantiation of concrete infrastructure classes
    private final MySQLUserRepository userRepository = new MySQLUserRepository();
    private final SendGridEmailService emailService = new SendGridEmailService();
    private final TwilioSMSService smsService = new TwilioSMSService();

    public void registerUser(String username, String email,
                             String rawPassword, String phoneNumber) {

        // Step 1: Validate input (business logic - correct to be here)
        if (username == null || username.isBlank()) {
            throw new IllegalArgumentException("Username must not be blank");
        }
        if (email == null || !email.contains("@")) {
            throw new IllegalArgumentException("Email address is not valid");
        }
        if (rawPassword == null || rawPassword.length() < 8) {
            throw new IllegalArgumentException("Password must be at least 8 characters");
        }

        // Step 2: Check for duplicate email (business logic - correct to be here)
        // DIP VIOLATION: calling a method on a concrete class
        if (userRepository.existsByEmail(email)) {
            throw new IllegalStateException("An account with this email already exists");
        }

        // Step 3: Hash the password (business logic - correct to be here)
        String passwordHash = hashPassword(rawPassword);

        // Step 4: Persist the user
        User user = new User(username, email, passwordHash);
        // DIP VIOLATION: calling save on a concrete class
        userRepository.save(user);

        // Step 5: Send welcome email
        // DIP VIOLATION: calling sendWelcomeEmail on a concrete class
        emailService.sendWelcomeEmail(email, username);

        // Step 6: Send verification SMS if phone provided
        if (phoneNumber != null && !phoneNumber.isBlank()) {
            String verificationCode = generateVerificationCode();
            // DIP VIOLATION: calling sendVerificationSMS on a concrete class
            smsService.sendVerificationSMS(phoneNumber, verificationCode);
        }

        System.out.println("User registered successfully: " + username);
    }

    private String hashPassword(String rawPassword) {
        // In production, use BCrypt. This is a placeholder.
        return "hashed_" + rawPassword.hashCode();
    }

    private String generateVerificationCode() {
        return String.valueOf((int)(Math.random() * 900000) + 100000);
    }
}


public class User {
    private long id;
    private String username;
    private String email;
    private String passwordHash;

    public User(String username, String email, String passwordHash) {
        this.username = username;
        this.email = email;
        this.passwordHash = passwordHash;
    }

    public long getId() { return id; }
    public void setId(long id) { this.id = id; }
    public String getUsername() { return username; }
    public String getEmail() { return email; }
    public String getPasswordHash() { return passwordHash; }
}
```

---

### Task 5.1 - Identify All DIP Violations

Answer the following in writing:

1. List every DIP violation in `UserRegistrationService` by line reference (e.g., "Line X: `new MySQLUserRepository()` directly instantiates a concrete class"). For each violation, explain:
   - What is the high-level module?
   - What is the low-level module?
   - What is the coupling problem?

2. What happens when the engineering team decides to migrate from MySQL to PostgreSQL? Walk through every file that must change. How does this affect `UserRegistrationService`?

3. What happens when the business team decides to migrate from SendGrid to AWS SES? Walk through every file that must change. How does this affect `UserRegistrationService`?

4. Why is it impossible to write a unit test for `UserRegistrationService` in its current form without a live MySQL database, a live SendGrid account, and a live Twilio account? What category of testing does this make impossible?

---

### Task 5.2 - Refactor Using Dependency Inversion

Refactor `UserRegistrationService` so that:

1. You define interfaces for all three infrastructure dependencies.
2. `UserRegistrationService` depends only on the interfaces (not concrete classes).
3. Dependencies are injected via constructor (constructor injection is preferred over setter injection for mandatory dependencies).
4. The concrete implementations (`MySQLUserRepository`, `SendGridEmailService`, `TwilioSMSService`) continue to exist but are no longer referenced by name inside `UserRegistrationService`.

**Required interface definitions:**

```java
public interface IUserRepository {
    void save(User user);
    boolean existsByEmail(String email);
}

public interface IEmailService {
    void sendWelcomeEmail(String recipientEmail, String username);
}

public interface ISmsService {
    void sendVerificationSMS(String phoneNumber, String verificationCode);
}
```

Write:
- The three interfaces above with proper Javadoc
- The refactored `UserRegistrationService` (full implementation)
- A brief comment showing how the concrete classes now implement the interfaces

---

### Task 5.3 - Write Unit Tests Using Mock Implementations

Write unit tests for the refactored `UserRegistrationService` using hand-written mock implementations (do not use Mockito or any mocking framework - write the mocks yourself). This demonstrates that DIP enables testing without infrastructure.

Write the following test helper classes:

```java
// A mock repository that stores users in memory
public class InMemoryUserRepository implements IUserRepository {
    // Store users in a List or Map
    // Track whether save() and existsByEmail() were called
    // Allow test setup: pre-populate with existing emails to test duplicate detection
}

// A mock email service that records what it was asked to send
public class FakeEmailService implements IEmailService {
    // Record the recipient email and username from each call
    // Provide assertions helpers (wasEmailSentTo, getLastRecipient, etc.)
}

// A mock SMS service that records what it was asked to send
public class FakeSmsService implements ISmsService {
    // Record the phone number and verification code from each call
    // Provide assertion helpers
}
```

Write the following test cases (plain JUnit 5, no Spring context):

1. `registerUser_withValidInputAndPhone_savesUserSendsEmailAndSms` - happy path with phone number
2. `registerUser_withValidInputNoPhone_savesUserSendsEmailButNotSms` - happy path without phone
3. `registerUser_withDuplicateEmail_throwsIllegalStateException` - duplicate email detection
4. `registerUser_withBlankUsername_throwsIllegalArgumentException` - validation
5. `registerUser_withInvalidEmail_throwsIllegalArgumentException` - validation
6. `registerUser_withShortPassword_throwsIllegalArgumentException` - validation

Each test must assert not only the exception or outcome but also that the mocks were called (or not called) with the expected arguments.

---

### Task 5.4 - Rubric

| Criterion | Points | Description |
|---|---|---|
| DIP violation identification | 20 | All four violations are identified precisely; MySQL/SendGrid migration impact is correctly traced; unit testing impossibility is correctly explained |
| Interface definitions | 15 | Interfaces are minimal, correctly typed, and have meaningful Javadoc that specifies contract (not implementation) |
| Refactored UserRegistrationService | 25 | No concrete infrastructure class is named; constructor injection is used; business logic is identical to original; compiles cleanly |
| Mock implementations | 20 | Mocks correctly implement the interfaces; state is tracked; assertion helpers are present and useful |
| Unit tests | 20 | All six test cases are present; each tests one scenario; mocks are used to verify calls; tests pass against the refactored service |

**Total: 100 points**

---

## Bonus Challenge - All Five Principles

### Background

Real legacy codebases rarely violate just one SOLID principle. They tend to accumulate violations across all five simultaneously because the violations reinforce each other: a God class with multiple responsibilities will naturally have a fat interface, will resist extension, will have subclasses that violate LSP, and will depend directly on infrastructure. This challenge gives you a single realistic God class and asks you to redesign it comprehensively.

**Estimated additional time:** 3-5 hours. This challenge is appropriate for engineers targeting Staff or Principal Engineer roles.

### The God Class

```java
import java.sql.*;
import java.util.*;

/**
 * ILibrarySystem - A fat interface that exposes every method of the God class.
 * This violates ISP because clients managing books have no business knowing about
 * fine calculation, and clients processing returns have no business knowing about
 * email sending.
 */
public interface ILibrarySystem {
    // Book catalog operations
    void addBook(String isbn, String title, String author, int totalCopies);
    Book findBook(String isbn);
    List<Book> searchBooks(String keyword);

    // Member management
    void registerMember(String memberId, String name, String email);
    Member findMember(String memberId);

    // Borrowing and returning
    void borrowBook(String memberId, String isbn);
    void returnBook(String memberId, String isbn);
    List<BorrowRecord> getActiveBorrowsForMember(String memberId);

    // Fine calculation
    double calculateFine(String memberId, String isbn);
    void payFine(String memberId, double amount);

    // Reporting
    String generateOverdueReport();
    String generatePopularBooksReport();
    String generateMemberActivityReport(String memberId);

    // Email notifications
    void sendOverdueNotification(String memberId);
    void sendWelcomeEmail(String memberId);

    // Database
    void persistBook(Book book);
    void persistMember(Member member);
    void persistBorrowRecord(BorrowRecord record);
}


/**
 * LibrarySystem - The God class. Violates SRP (7 responsibilities), OCP (hardcoded
 * fine rules and report formats), and DIP (direct DB access). Also exposes the fat
 * ILibrarySystem interface (ISP violation).
 */
public class LibrarySystem implements ILibrarySystem {

    private static final String DB_URL = "jdbc:postgresql://localhost:5432/library";
    private static final String DB_USER = "library_app";
    private static final String DB_PASSWORD = "libPass789";
    private static final double DAILY_FINE_RATE = 0.50;
    private static final int BORROW_PERIOD_DAYS = 14;

    // In-memory caches (mixed with business logic and infrastructure - SRP violation)
    private final Map<String, Book> bookCache = new HashMap<>();
    private final Map<String, Member> memberCache = new HashMap<>();

    // ----------------------------------------------------------------
    // BOOK CATALOG
    // ----------------------------------------------------------------

    @Override
    public void addBook(String isbn, String title, String author, int totalCopies) {
        if (isbn == null || isbn.isBlank()) throw new IllegalArgumentException("ISBN is required");
        if (totalCopies <= 0) throw new IllegalArgumentException("Total copies must be positive");
        Book book = new Book(isbn, title, author, totalCopies, totalCopies);
        bookCache.put(isbn, book);
        persistBook(book);
    }

    @Override
    public Book findBook(String isbn) {
        if (bookCache.containsKey(isbn)) return bookCache.get(isbn);
        String sql = "SELECT * FROM books WHERE isbn = ?";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            stmt.setString(1, isbn);
            ResultSet rs = stmt.executeQuery();
            if (rs.next()) {
                Book book = new Book(rs.getString("isbn"), rs.getString("title"),
                        rs.getString("author"), rs.getInt("total_copies"),
                        rs.getInt("available_copies"));
                bookCache.put(isbn, book);
                return book;
            }
        } catch (SQLException e) {
            throw new RuntimeException("DB error finding book", e);
        }
        return null;
    }

    @Override
    public List<Book> searchBooks(String keyword) {
        List<Book> results = new ArrayList<>();
        String sql = "SELECT * FROM books WHERE title ILIKE ? OR author ILIKE ?";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            stmt.setString(1, "%" + keyword + "%");
            stmt.setString(2, "%" + keyword + "%");
            ResultSet rs = stmt.executeQuery();
            while (rs.next()) {
                results.add(new Book(rs.getString("isbn"), rs.getString("title"),
                        rs.getString("author"), rs.getInt("total_copies"),
                        rs.getInt("available_copies")));
            }
        } catch (SQLException e) {
            throw new RuntimeException("DB error searching books", e);
        }
        return results;
    }

    // ----------------------------------------------------------------
    // MEMBER MANAGEMENT
    // ----------------------------------------------------------------

    @Override
    public void registerMember(String memberId, String name, String email) {
        if (memberId == null || memberId.isBlank()) throw new IllegalArgumentException("Member ID required");
        Member member = new Member(memberId, name, email, 0.0);
        memberCache.put(memberId, member);
        persistMember(member);
        sendWelcomeEmail(memberId); // SRP violation: member registration triggers email sending
    }

    @Override
    public Member findMember(String memberId) {
        if (memberCache.containsKey(memberId)) return memberCache.get(memberId);
        String sql = "SELECT * FROM members WHERE member_id = ?";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            stmt.setString(1, memberId);
            ResultSet rs = stmt.executeQuery();
            if (rs.next()) {
                return new Member(rs.getString("member_id"), rs.getString("name"),
                        rs.getString("email"), rs.getDouble("outstanding_fines"));
            }
        } catch (SQLException e) {
            throw new RuntimeException("DB error finding member", e);
        }
        return null;
    }

    // ----------------------------------------------------------------
    // BORROWING AND RETURNING
    // ----------------------------------------------------------------

    @Override
    public void borrowBook(String memberId, String isbn) {
        Member member = findMember(memberId);
        Book book = findBook(isbn);
        if (member == null) throw new IllegalArgumentException("Member not found: " + memberId);
        if (book == null) throw new IllegalArgumentException("Book not found: " + isbn);
        if (book.getAvailableCopies() <= 0) throw new IllegalStateException("No copies available for: " + isbn);
        if (member.getOutstandingFines() > 5.0) {
            throw new IllegalStateException("Member has outstanding fines over $5.00. Pay fines before borrowing.");
        }

        book.setAvailableCopies(book.getAvailableCopies() - 1);
        BorrowRecord record = new BorrowRecord(memberId, isbn, new Date(), null);
        persistBorrowRecord(record);
        persistBook(book);
    }

    @Override
    public void returnBook(String memberId, String isbn) {
        String sql = "SELECT * FROM borrow_records WHERE member_id = ? AND isbn = ? AND return_date IS NULL";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            stmt.setString(1, memberId);
            stmt.setString(2, isbn);
            ResultSet rs = stmt.executeQuery();
            if (!rs.next()) throw new IllegalStateException("No active borrow record found for this member/book");

            Date borrowDate = rs.getDate("borrow_date");
            Date today = new Date();
            long daysOverdue = Math.max(0, (today.getTime() - borrowDate.getTime()) / (1000 * 60 * 60 * 24) - BORROW_PERIOD_DAYS);

            if (daysOverdue > 0) {
                double fine = daysOverdue * DAILY_FINE_RATE;
                Member member = findMember(memberId);
                member.setOutstandingFines(member.getOutstandingFines() + fine);
                persistMember(member);
                // SRP violation: returnBook also handles fine calculation and member state update
            }

            String updateSql = "UPDATE borrow_records SET return_date = NOW() WHERE member_id = ? AND isbn = ? AND return_date IS NULL";
            try (PreparedStatement updateStmt = conn.prepareStatement(updateSql)) {
                updateStmt.setString(1, memberId);
                updateStmt.setString(2, isbn);
                updateStmt.executeUpdate();
            }

            Book book = findBook(isbn);
            book.setAvailableCopies(book.getAvailableCopies() + 1);
            persistBook(book);

        } catch (SQLException e) {
            throw new RuntimeException("DB error returning book", e);
        }
    }

    @Override
    public List<BorrowRecord> getActiveBorrowsForMember(String memberId) {
        List<BorrowRecord> records = new ArrayList<>();
        String sql = "SELECT * FROM borrow_records WHERE member_id = ? AND return_date IS NULL";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            stmt.setString(1, memberId);
            ResultSet rs = stmt.executeQuery();
            while (rs.next()) {
                records.add(new BorrowRecord(rs.getString("member_id"), rs.getString("isbn"),
                        rs.getDate("borrow_date"), rs.getDate("return_date")));
            }
        } catch (SQLException e) {
            throw new RuntimeException("DB error fetching borrow records", e);
        }
        return records;
    }

    // ----------------------------------------------------------------
    // FINE CALCULATION - OCP violation: adding a new fine rule requires
    // modifying this method and potentially returnBook() above
    // ----------------------------------------------------------------

    @Override
    public double calculateFine(String memberId, String isbn) {
        // Rule 1: Standard overdue fine
        // Rule 2: If book is a "rare" book (ISBN starts with RARE-), double the fine rate
        // Rule 3: If member is a "premium" member (ID starts with PREM-), fines are halved
        String sql = "SELECT borrow_date FROM borrow_records WHERE member_id = ? AND isbn = ? AND return_date IS NULL";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            stmt.setString(1, memberId);
            stmt.setString(2, isbn);
            ResultSet rs = stmt.executeQuery();
            if (!rs.next()) return 0.0;

            Date borrowDate = rs.getDate("borrow_date");
            Date today = new Date();
            long daysOverdue = Math.max(0, (today.getTime() - borrowDate.getTime()) / (1000 * 60 * 60 * 24) - BORROW_PERIOD_DAYS);

            double rate = DAILY_FINE_RATE;
            if (isbn.startsWith("RARE-")) rate *= 2.0;     // OCP violation: hardcoded rule
            if (memberId.startsWith("PREM-")) rate *= 0.5; // OCP violation: hardcoded rule

            return daysOverdue * rate;
        } catch (SQLException e) {
            throw new RuntimeException("DB error calculating fine", e);
        }
    }

    @Override
    public void payFine(String memberId, double amount) {
        Member member = findMember(memberId);
        if (member == null) throw new IllegalArgumentException("Member not found");
        double newBalance = Math.max(0, member.getOutstandingFines() - amount);
        member.setOutstandingFines(newBalance);
        persistMember(member);
    }

    // ----------------------------------------------------------------
    // REPORTING - OCP violation: adding a new report type requires
    // modifying this class. Format changes (CSV, JSON) also require modification.
    // ----------------------------------------------------------------

    @Override
    public String generateOverdueReport() {
        StringBuilder sb = new StringBuilder("=== OVERDUE BOOKS REPORT ===\n");
        String sql = "SELECT br.member_id, br.isbn, br.borrow_date, m.name, b.title " +
                     "FROM borrow_records br JOIN members m ON br.member_id = m.member_id " +
                     "JOIN books b ON br.isbn = b.isbn " +
                     "WHERE br.return_date IS NULL AND br.borrow_date < NOW() - INTERVAL '14 days'";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            ResultSet rs = stmt.executeQuery();
            while (rs.next()) {
                sb.append(String.format("Member: %-20s Book: %-30s Borrowed: %s%n",
                        rs.getString("name"), rs.getString("title"), rs.getDate("borrow_date")));
            }
        } catch (SQLException e) {
            throw new RuntimeException("DB error generating overdue report", e);
        }
        return sb.toString();
    }

    @Override
    public String generatePopularBooksReport() {
        StringBuilder sb = new StringBuilder("=== POPULAR BOOKS REPORT ===\n");
        String sql = "SELECT b.isbn, b.title, b.author, COUNT(br.isbn) AS borrow_count " +
                     "FROM books b LEFT JOIN borrow_records br ON b.isbn = br.isbn " +
                     "GROUP BY b.isbn, b.title, b.author ORDER BY borrow_count DESC LIMIT 10";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            ResultSet rs = stmt.executeQuery();
            int rank = 1;
            while (rs.next()) {
                sb.append(String.format("%2d. %-40s by %-20s (%d borrows)%n",
                        rank++, rs.getString("title"), rs.getString("author"), rs.getInt("borrow_count")));
            }
        } catch (SQLException e) {
            throw new RuntimeException("DB error generating popular books report", e);
        }
        return sb.toString();
    }

    @Override
    public String generateMemberActivityReport(String memberId) {
        Member member = findMember(memberId);
        if (member == null) throw new IllegalArgumentException("Member not found");
        StringBuilder sb = new StringBuilder("=== MEMBER ACTIVITY: " + member.getName() + " ===\n");
        List<BorrowRecord> records = getActiveBorrowsForMember(memberId);
        sb.append("Active borrows: ").append(records.size()).append("\n");
        sb.append("Outstanding fines: $").append(String.format("%.2f", member.getOutstandingFines())).append("\n");
        return sb.toString();
    }

    // ----------------------------------------------------------------
    // EMAIL NOTIFICATIONS - SRP violation: notification logic in library system
    // ----------------------------------------------------------------

    @Override
    public void sendOverdueNotification(String memberId) {
        Member member = findMember(memberId);
        if (member == null) return;
        // Direct SMTP-style email sending (DIP violation: hardcoded infrastructure)
        System.out.println("[EMAIL] To: " + member.getEmail());
        System.out.println("[EMAIL] Subject: Overdue Book Reminder");
        System.out.println("[EMAIL] Body: Dear " + member.getName() + ", you have overdue books. Please return them.");
    }

    @Override
    public void sendWelcomeEmail(String memberId) {
        Member member = findMember(memberId);
        if (member == null) return;
        System.out.println("[EMAIL] To: " + member.getEmail());
        System.out.println("[EMAIL] Subject: Welcome to the Library!");
        System.out.println("[EMAIL] Body: Dear " + member.getName() + ", your library account is active.");
    }

    // ----------------------------------------------------------------
    // DATABASE PERSISTENCE - DIP violation: direct JDBC in business class
    // ----------------------------------------------------------------

    @Override
    public void persistBook(Book book) {
        String sql = "INSERT INTO books (isbn, title, author, total_copies, available_copies) VALUES (?, ?, ?, ?, ?) " +
                     "ON CONFLICT (isbn) DO UPDATE SET available_copies = EXCLUDED.available_copies";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            stmt.setString(1, book.getIsbn());
            stmt.setString(2, book.getTitle());
            stmt.setString(3, book.getAuthor());
            stmt.setInt(4, book.getTotalCopies());
            stmt.setInt(5, book.getAvailableCopies());
            stmt.executeUpdate();
        } catch (SQLException e) {
            throw new RuntimeException("DB error persisting book", e);
        }
    }

    @Override
    public void persistMember(Member member) {
        String sql = "INSERT INTO members (member_id, name, email, outstanding_fines) VALUES (?, ?, ?, ?) " +
                     "ON CONFLICT (member_id) DO UPDATE SET outstanding_fines = EXCLUDED.outstanding_fines";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            stmt.setString(1, member.getMemberId());
            stmt.setString(2, member.getName());
            stmt.setString(3, member.getEmail());
            stmt.setDouble(4, member.getOutstandingFines());
            stmt.executeUpdate();
        } catch (SQLException e) {
            throw new RuntimeException("DB error persisting member", e);
        }
    }

    @Override
    public void persistBorrowRecord(BorrowRecord record) {
        String sql = "INSERT INTO borrow_records (member_id, isbn, borrow_date) VALUES (?, ?, ?)";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            stmt.setString(1, record.getMemberId());
            stmt.setString(2, record.getIsbn());
            stmt.setDate(3, new java.sql.Date(record.getBorrowDate().getTime()));
            stmt.executeUpdate();
        } catch (SQLException e) {
            throw new RuntimeException("DB error persisting borrow record", e);
        }
    }
}


/**
 * LibrarySystemV2 - Extends LibrarySystem to add "enhanced" features.
 * Contains LSP violations.
 */
public class LibrarySystemV2 extends LibrarySystem {

    private static final int PREMIUM_BORROW_PERIOD_DAYS = 30;

    /**
     * LSP VIOLATION: Strengthens precondition.
     * Base class allows any valid memberId and isbn.
     * This override rejects non-premium members, which the base class contract does not allow.
     */
    @Override
    public void borrowBook(String memberId, String isbn) {
        // VIOLATION: Additional precondition not in base class
        if (!memberId.startsWith("PREM-") && !memberId.startsWith("STD-")) {
            throw new IllegalArgumentException("LibrarySystemV2 only accepts PREM- or STD- member IDs");
        }
        super.borrowBook(memberId, isbn);
    }

    /**
     * LSP VIOLATION: Weakens postcondition.
     * The base class generateOverdueReport() always returns a non-null string.
     * This override may return null.
     */
    @Override
    public String generateOverdueReport() {
        // VIOLATION: Returns null when there are no overdue items,
        // but callers of the base type expect a non-null String
        String report = super.generateOverdueReport();
        if (report.equals("=== OVERDUE BOOKS REPORT ===\n")) {
            return null; // "no data" represented as null instead of an empty report
        }
        return report;
    }
}


// Supporting domain classes (not to be refactored - just context)
class Book {
    private String isbn, title, author;
    private int totalCopies, availableCopies;
    public Book(String isbn, String title, String author, int totalCopies, int availableCopies) {
        this.isbn = isbn; this.title = title; this.author = author;
        this.totalCopies = totalCopies; this.availableCopies = availableCopies;
    }
    public String getIsbn() { return isbn; }
    public String getTitle() { return title; }
    public String getAuthor() { return author; }
    public int getTotalCopies() { return totalCopies; }
    public int getAvailableCopies() { return availableCopies; }
    public void setAvailableCopies(int availableCopies) { this.availableCopies = availableCopies; }
}

class Member {
    private String memberId, name, email;
    private double outstandingFines;
    public Member(String memberId, String name, String email, double outstandingFines) {
        this.memberId = memberId; this.name = name;
        this.email = email; this.outstandingFines = outstandingFines;
    }
    public String getMemberId() { return memberId; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public double getOutstandingFines() { return outstandingFines; }
    public void setOutstandingFines(double outstandingFines) { this.outstandingFines = outstandingFines; }
}

class BorrowRecord {
    private String memberId, isbn;
    private Date borrowDate, returnDate;
    public BorrowRecord(String memberId, String isbn, Date borrowDate, Date returnDate) {
        this.memberId = memberId; this.isbn = isbn;
        this.borrowDate = borrowDate; this.returnDate = returnDate;
    }
    public String getMemberId() { return memberId; }
    public String getIsbn() { return isbn; }
    public Date getBorrowDate() { return borrowDate; }
    public Date getReturnDate() { return returnDate; }
}
```

---

### Bonus Task - Redesign the Entire System

Redesign `LibrarySystem` and `LibrarySystemV2` applying all five SOLID principles. Your solution must explicitly address each principle and explain how your design satisfies it.

**Deliverable 1: Class Diagram (ASCII Art)**

Produce an ASCII class diagram showing all interfaces, classes, and their relationships. Use the following conventions:

```
[Interface]       - square brackets for interfaces
(AbstractClass)   - round brackets for abstract classes
ConcreteClass     - no decoration for concrete classes
-->               - implements / extends
---->             - depends on (uses)
```

Example format (not the answer - just the format):

```
[IBookRepository] <-- JdbcBookRepository
[IBookRepository] <-- InMemoryBookRepository
BookService ----> [IBookRepository]
BookService ----> [INotificationService]
```

Your diagram should show all services, all repositories, all notification interfaces, the fine policy interface, the reporting interface, and the domain objects.

**Deliverable 2: All Interface Definitions**

Write the full Javadoc and method signatures for every interface in your design. Each interface should be focused (ISP), contract-specifying (not implementation-describing), and should not expose any method that not all implementors can fulfill.

Minimum expected interfaces (you may add more):
- `IBookRepository`
- `IMemberRepository`
- `IBorrowRecordRepository`
- `IFinePolicy`
- `INotificationService`
- `IReportGenerator`

**Deliverable 3: Key Class Implementations**

Write full implementations for:
- `BorrowingService` (the class responsible for borrowBook and returnBook - depends only on interfaces)
- `OverdueFinePolicy implements IFinePolicy` (the standard fine calculation)
- `PremiumMemberFinePolicy implements IFinePolicy` (the premium member adjusted fine - added without modifying OverdueFinePolicy, demonstrating OCP)
- One repository implementation of your choice

**Deliverable 4: SOLID Principle Mapping**

Write a section for each principle explaining:
1. Which violation in the original `LibrarySystem` corresponds to this principle
2. Which specific design decision in your refactored design addresses it
3. Which class or interface in your design is the evidence that this principle is now satisfied

---

## Self-Evaluation Rubric

### Per-Principle Quality Matrix

Use this matrix to evaluate your solutions before submitting.

| Principle | Excellent (90-100%) | Good (70-89%) | Needs Work (below 70%) |
|---|---|---|---|
| SRP | Each class has a single actor; changing the tax rules requires touching exactly one class; changing the PDF layout requires touching exactly one class; no class has methods from two different responsibilities | Most responsibilities are separated; one or two methods may be misplaced; coordinator class is mostly clean | God class broken into slightly smaller God classes; responsibilities still mixed; coordinator knows about implementation details |
| OCP | Adding a new shipping type (or fine policy, or report type) requires zero changes to existing classes; only a new file is created; tests for existing types do not need updating | Strategy pattern is in place but the factory or dispatcher still requires modification when a new type is added | if/else or switch still present; adding new type still requires modifying existing class |
| LSP | Every subclass can be used everywhere the base class is used; no UnsupportedOperationException; postconditions hold for all subtypes; base class contract is documented explicitly | Postconditions hold for most cases; one edge case where subclass behavior diverges from contract but does not throw | Subclasses throw exceptions for methods they cannot support; postconditions are silently violated; FleetManager/AnimalSimulator breaks at runtime |
| ISP | No class implements a method it does not meaningfully support; no UnsupportedOperationException in any implementation; interfaces have two to five methods maximum | Most interfaces are focused; one interface may have one extra method; one class may have one no-op implementation | Fat interfaces remain; multiple classes have throwing implementations; clients depend on methods they do not use |
| DIP | Business logic classes name zero concrete infrastructure classes; all dependencies injected via constructor; unit tests require zero infrastructure setup; swapping MySQL for PostgreSQL requires changing exactly one class | Most dependencies are injected; one or two places still use new ConcreteClass(); unit tests possible but require some stubbing | Direct instantiation remains in business logic; unit tests require live database or external services |

---

### Common Mistakes Checklist

Review your solution against each item below before considering it complete.

**SRP mistakes:**
- [ ] Breaking one God class into several slightly-smaller God classes (splitting is not the same as separating responsibilities)
- [ ] Putting the coordinator/facade logic in the same class as one of the implementations
- [ ] Defining an interface with methods from two different responsibilities
- [ ] Putting validation logic in the repository class

**OCP mistakes:**
- [ ] Using the Strategy pattern but keeping a switch statement in the factory to select the strategy
- [ ] Creating a Strategy interface but not making each strategy a self-contained class (leaking knowledge of other strategies)
- [ ] Making the calculator work but leaving getShippingSla() unreformed (if/else still present there)
- [ ] Forgetting to demonstrate that the new type (LockerShipping, new fine policy) was added with zero modifications to existing files

**LSP mistakes:**
- [ ] Fixing UnsupportedOperationException by making the method a no-op (silent no-op violates the postcondition just as badly as throwing)
- [ ] Designing a hierarchy where the base class has a method that only some subclasses can implement
- [ ] Using instanceof in the refactored FleetManager as a permanent solution (acceptable as a transition; not acceptable as the final design)
- [ ] Forgetting to document the contracts (preconditions, postconditions) in the refactored base class

**ISP mistakes:**
- [ ] Creating interfaces with too many methods "just in case" a future class might need them
- [ ] Merging ICanRun and ICanFly into a single ILocomotion interface when not all locomotion types share both methods
- [ ] Designing AnimalSimulator so that it still needs to check capabilities at runtime
- [ ] Forgetting that ISP applies to clients, not just implementors - a client that only needs ICanFly should not be forced to depend on the full IAnimal

**DIP mistakes:**
- [ ] Defining interfaces in the same package as the concrete implementations (the interface should be in the business layer, the implementation in the infrastructure layer)
- [ ] Using setter injection instead of constructor injection for mandatory dependencies
- [ ] Writing unit tests that test mock behavior (verifying that the mock returns what you told it to return) rather than testing the business logic
- [ ] Leaving hardcoded credentials in the concrete implementation classes (a separate but related problem - flag it)

---

### How to Know You Are Done

You are done with an assignment when you can answer "yes" to all of the following:

1. **Compilation:** Every class I wrote compiles without errors.
2. **Single change, single class:** For each type of change that was previously scattered across the God class, I can name exactly one class in my refactored design that would need to change.
3. **Zero throwing implementations:** No class in my design implements an interface method by throwing `UnsupportedOperationException` (or any other exception not declared in the interface contract).
4. **Constructor-injected dependencies:** No business logic class in my design contains `new ConcreteInfrastructureClass()`.
5. **Testable without infrastructure:** I can instantiate and test every business logic class using only in-memory fakes, with no database, no HTTP calls, and no filesystem access.
6. **Rubric score above 70%:** I have honestly scored myself against the rubric and achieved at least 70 out of 100 on each assignment.
7. **Explanation in words:** I can explain in plain English, without code, why the original design was wrong and what specific design decision I made to fix each violation.

---

*End of Assignment File*
