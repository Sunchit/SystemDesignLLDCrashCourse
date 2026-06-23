# Single Responsibility Principle (SRP)

## Table of Contents

1. [Definition and Mental Models](#1-definition-and-mental-models)
2. [The Reason to Change Mental Model](#2-the-reason-to-change-mental-model)
3. [Identifying SRP Violations](#3-identifying-srp-violations)
4. [Before and After Refactoring: The Employee God Class](#4-before-and-after-refactoring-the-employee-god-class)
5. [Common Anti-Patterns](#5-common-anti-patterns)
6. [When SRP Can Be Over-Applied](#6-when-srp-can-be-over-applied)
7. [The Cohesion Connection](#7-the-cohesion-connection)
8. [Interview Questions and Model Answers](#8-interview-questions-and-model-answers)
9. [Assignment: Refactor the OrderProcessor](#9-assignment-refactor-the-orderprocessor)

---

## 1. Definition and Mental Models

### Robert Martin's Original Definition

> "A class should have only one reason to change."
>
> -- Robert C. Martin, *Agile Software Development: Principles, Patterns, and Practices* (2002)

This formulation is deceptively simple. The critical word is "reason." Martin was not saying a class should do only one thing in a mechanical sense -- a `UserService` may expose a dozen methods. He was saying those methods should all change for the same underlying reason. If you can imagine two distinct forces that could independently drive you to open this file and modify it, the class has more than one responsibility.

### The Modern Reformulation: The Actor Model

Martin himself clarified and sharpened the definition in *Clean Architecture* (2017):

> "A module should be responsible to one, and only one, actor."

An **actor** is a group of users or stakeholders who have a common set of needs and who interact with the system from the same perspective. They are the source of change requests. If two different people or teams could independently ask you to modify the same class for completely unrelated reasons, that class is serving two actors -- and that is the violation.

This reformulation is more actionable because it forces you to think sociologically rather than technically. Instead of asking "what does this code do?", you ask "who cares about this code, and why would they ever want it changed?"

### Putting Both Definitions Together

The two definitions are complementary:

| Lens | Question to ask |
|---|---|
| "Reason to change" | If requirement X changes, does that force me to change this class? Now if requirement Y changes (unrelated to X), does that also force me to change this class? |
| "Actor / stakeholder" | Is the CFO making a payroll change going to modify the same file as the DBA optimizing queries? |

If the answer to either question reveals two independent forces, the class violates SRP.

---

## 2. The Reason to Change Mental Model

### What Counts as a "Reason"

A "reason to change" is a category of requirement change traceable to a specific stakeholder group. It is not a line of code, not a method, not a feature -- it is the *origin* of the pressure to change.

Examples of distinct reasons:

- The finance team decides to change how overtime is calculated. (Payroll logic)
- The DBA team decides to switch from JDBC to JPA. (Persistence strategy)
- The HR team decides payslips need a new column for tax deductions. (Report format)
- The security team mandates all database calls go through a new audit layer. (Cross-cutting infrastructure)

Each of these is a separate "reason." If all four of them could force you to modify a single class, that class has four responsibilities, not one.

### How to Identify Stakeholders

Stakeholders are not always people. They are *roles* or *systems* that care about the behavior of a module. A useful categorization:

1. **Business domain experts**: product managers, finance, HR, legal. They own business rules.
2. **Technical operations**: DBAs, platform engineers, SREs. They own infrastructure, performance, and data access strategies.
3. **External consumers**: other services, clients, third-party integrators. They own the API contract.
4. **Presentation layer owners**: frontend engineers, report designers. They own output format.

### The Actor Model in Practice

When you read a class, ask: "If this class were a person's job description, how many managers would this person report to?"

A class with `calculatePay()`, `save()`, and `generatePayslip()` reports to three different managers:
- The finance director (for pay calculation rules)
- The DBA / infra team (for persistence)
- The HR director (for report format)

That is a sign of a violation. The moment any one of those managers sends a change request, the developer must open the same file that the other two managers also depend on. This creates accidental coupling, merge conflicts, and regression risk.

---

## 3. Identifying SRP Violations

### Code Smell 1: The God Class

A God class knows too much and does too much. It has hundreds of lines, imports from half the application, and its name often ends in `Manager`, `Handler`, `Processor`, or `Service` but covers multiple unrelated domains.

**Warning signs:**
- More than 300-500 lines in a single class (context-dependent, but a reliable heuristic)
- The class name is vague or generic (`DataManager`, `AppController`, `SystemHelper`)
- Methods in the class have nothing logically in common

### Code Smell 2: Methods with Nothing in Common

Open the class and list every method. Now ask: could you draw a clear line grouping them into two or more clusters where methods within a cluster are closely related but methods across clusters are unrelated? If yes, those clusters are separate responsibilities.

```java
class Employee {
    // Cluster 1 -- business logic
    Money calculatePay() { ... }
    EmployeeType getEmployeeType() { ... }

    // Cluster 2 -- persistence
    void save() { ... }
    Employee findById(long id) { ... }

    // Cluster 3 -- reporting
    void generatePayslip() { ... }
    void sendReportToHR() { ... }
}
```

Three distinct clusters. Three responsibilities. SRP violated.

### Code Smell 3: Large Import Lists

If a class imports from `java.sql.*`, `java.io.*`, a PDF library, an email library, and your domain package, it is almost certainly doing too much. Each import family represents a different abstraction layer or external concern.

### Code Smell 4: Tests That Need Many Mocks

When writing a unit test for a single method, if you need to set up mocks for a database connection, an email client, a file system, and a payment gateway -- all because they are instance variables on the same class -- the class has too many dependencies, which is a direct consequence of too many responsibilities.

```java
// This test setup signals a problem
@Test
void testCalculatePay() {
    DataSource mockDataSource = mock(DataSource.class);
    EmailClient mockEmail = mock(EmailClient.class);
    PDFGenerator mockPDF = mock(PDFGenerator.class);
    PaymentGateway mockPayment = mock(PaymentGateway.class);

    Employee employee = new Employee(mockDataSource, mockEmail, mockPDF, mockPayment);
    // ... we only wanted to test pay calculation
}
```

### Code Smell 5: Frequent, Unrelated Change History

Check the git log for a class. If the commit messages alternate between "fixed payroll calculation", "updated DB connection pool", and "changed payslip template", three different actors are driving changes to the same file. That is the SRP violation made visible over time.

### Code Smell 6: Conditional Branching on Type for Unrelated Behavior

```java
void process(Employee e) {
    if (e.getType() == CONTRACTOR) {
        // one business logic path
    } else {
        // different path
    }

    if (dbType == MYSQL) {
        // one persistence path
    } else {
        // different path
    }
}
```

When you see conditions branching on unrelated axes (business type and infrastructure type), multiple responsibilities are colliding inside a single method.

---

## 4. Before and After Refactoring: The Employee God Class

### The Problem Statement

We have an `Employee` class that a real development team might write early in a project when moving fast. It handles three distinct concerns:

1. **Business logic**: calculating pay and determining employee type
2. **Database persistence**: saving and retrieving employees
3. **Report generation**: generating payslips and notifying HR

### BEFORE: The God Class

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.Properties;
import javax.mail.Message;
import javax.mail.MessagingException;
import javax.mail.Session;
import javax.mail.Transport;
import javax.mail.internet.InternetAddress;
import javax.mail.internet.MimeMessage;

/**
 * Violates SRP: this class is responsible to three different actors.
 *   1. Finance team      -> calculatePay(), getEmployeeType()
 *   2. DBA / Infra team  -> save(), findById()
 *   3. HR team           -> generatePayslip(), sendReportToHR()
 *
 * Any change by any of these three teams requires modifying this single file,
 * creating merge conflicts, regression risk, and tight coupling.
 */
public class Employee {

    private long id;
    private String name;
    private String email;
    private EmployeeType type;
    private BigDecimal baseSalary;
    private int hoursWorked;
    private LocalDate hireDate;

    // Database configuration baked into the class
    private static final String DB_URL = "jdbc:mysql://localhost:3306/hrdb";
    private static final String DB_USER = "root";
    private static final String DB_PASS = "secret";

    // Email configuration baked into the class
    private static final String SMTP_HOST = "smtp.company.com";
    private static final String HR_EMAIL = "hr@company.com";

    public enum EmployeeType {
        FULL_TIME, PART_TIME, CONTRACTOR
    }

    public Employee(long id, String name, String email,
                    EmployeeType type, BigDecimal baseSalary, int hoursWorked) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.type = type;
        this.baseSalary = baseSalary;
        this.hoursWorked = hoursWorked;
        this.hireDate = LocalDate.now();
    }

    // -------------------------------------------------------------------------
    // CONCERN 1: Business Logic
    // Owner: Finance team
    // Reason to change: tax rules, overtime policies, contractor rate changes
    // -------------------------------------------------------------------------

    public BigDecimal calculatePay() {
        switch (type) {
            case FULL_TIME:
                return calculateFullTimePay();
            case PART_TIME:
                return calculatePartTimePay();
            case CONTRACTOR:
                return calculateContractorPay();
            default:
                throw new IllegalStateException("Unknown employee type: " + type);
        }
    }

    private BigDecimal calculateFullTimePay() {
        // Full-time: base salary + overtime for hours beyond 40
        if (hoursWorked > 40) {
            BigDecimal overtimeHours = BigDecimal.valueOf(hoursWorked - 40);
            BigDecimal overtimeRate = baseSalary.divide(BigDecimal.valueOf(160))
                                                .multiply(BigDecimal.valueOf(1.5));
            return baseSalary.add(overtimeHours.multiply(overtimeRate));
        }
        return baseSalary;
    }

    private BigDecimal calculatePartTimePay() {
        // Part-time: hourly rate x hours worked
        BigDecimal hourlyRate = baseSalary.divide(BigDecimal.valueOf(160));
        return hourlyRate.multiply(BigDecimal.valueOf(hoursWorked));
    }

    private BigDecimal calculateContractorPay() {
        // Contractor: flat daily rate x days worked (hours / 8)
        BigDecimal dailyRate = baseSalary;
        BigDecimal daysWorked = BigDecimal.valueOf(hoursWorked).divide(BigDecimal.valueOf(8));
        return dailyRate.multiply(daysWorked);
    }

    public EmployeeType getEmployeeType() {
        return type;
    }

    // -------------------------------------------------------------------------
    // CONCERN 2: Database Persistence
    // Owner: DBA / Infrastructure team
    // Reason to change: schema changes, switch from JDBC to JPA, query tuning
    // -------------------------------------------------------------------------

    public void save() {
        String sql = "INSERT INTO employees (id, name, email, type, base_salary, hours_worked, hire_date) "
                   + "VALUES (?, ?, ?, ?, ?, ?, ?) "
                   + "ON DUPLICATE KEY UPDATE name=VALUES(name), email=VALUES(email), "
                   + "type=VALUES(type), base_salary=VALUES(base_salary), hours_worked=VALUES(hours_worked)";

        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASS);
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setLong(1, id);
            stmt.setString(2, name);
            stmt.setString(3, email);
            stmt.setString(4, type.name());
            stmt.setBigDecimal(5, baseSalary);
            stmt.setInt(6, hoursWorked);
            stmt.setObject(7, hireDate);
            stmt.executeUpdate();

        } catch (SQLException e) {
            throw new RuntimeException("Failed to save employee: " + id, e);
        }
    }

    public static Employee findById(long id) {
        String sql = "SELECT id, name, email, type, base_salary, hours_worked FROM employees WHERE id = ?";

        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASS);
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setLong(1, id);
            ResultSet rs = stmt.executeQuery();

            if (rs.next()) {
                return new Employee(
                    rs.getLong("id"),
                    rs.getString("name"),
                    rs.getString("email"),
                    EmployeeType.valueOf(rs.getString("type")),
                    rs.getBigDecimal("base_salary"),
                    rs.getInt("hours_worked")
                );
            }
            return null;

        } catch (SQLException e) {
            throw new RuntimeException("Failed to find employee: " + id, e);
        }
    }

    // -------------------------------------------------------------------------
    // CONCERN 3: Report Generation
    // Owner: HR team
    // Reason to change: payslip format changes, new fields, different recipients
    // -------------------------------------------------------------------------

    public String generatePayslip() {
        BigDecimal pay = calculatePay();
        StringBuilder sb = new StringBuilder();
        sb.append("===== PAYSLIP =====\n");
        sb.append("Employee: ").append(name).append("\n");
        sb.append("ID: ").append(id).append("\n");
        sb.append("Type: ").append(type).append("\n");
        sb.append("Hours Worked: ").append(hoursWorked).append("\n");
        sb.append("Gross Pay: $").append(pay).append("\n");
        sb.append("Period: ").append(LocalDate.now().getMonth()).append(" ").append(LocalDate.now().getYear()).append("\n");
        sb.append("===================\n");
        return sb.toString();
    }

    public void sendReportToHR() {
        String payslip = generatePayslip();

        Properties props = new Properties();
        props.put("mail.smtp.host", SMTP_HOST);
        Session session = Session.getDefaultInstance(props);

        try {
            MimeMessage message = new MimeMessage(session);
            message.setFrom(new InternetAddress(email));
            message.addRecipient(Message.RecipientType.TO, new InternetAddress(HR_EMAIL));
            message.setSubject("Payslip for " + name);
            message.setText(payslip);
            Transport.send(message);
        } catch (MessagingException e) {
            throw new RuntimeException("Failed to send payslip for employee: " + id, e);
        }
    }

    // Getters
    public long getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public BigDecimal getBaseSalary() { return baseSalary; }
    public int getHoursWorked() { return hoursWorked; }
    public LocalDate getHireDate() { return hireDate; }
}
```

**Problems with this design:**

1. The finance team changes overtime rules -> we open `Employee.java` -> risk of breaking persistence or report logic.
2. The DBA migrates from MySQL to PostgreSQL -> we open `Employee.java` -> risk of breaking business logic.
3. HR wants a new field on the payslip -> we open `Employee.java` -> risk of breaking pay calculation.
4. Unit testing `calculatePay()` requires a database to be available (or heavy mocking of static `DriverManager`).
5. `findById()` is a static method on a domain object -- the domain object is doing ORM work.
6. SMTP credentials and DB credentials live in the same class as tax logic.

---

### AFTER: Applying SRP

We decompose the God class into four focused units:

1. `Employee` -- pure data carrier (the domain entity)
2. `EmployeeService` -- business logic (the finance team's concern)
3. `EmployeeRepository` -- persistence (the DBA's concern)
4. `PayrollReporter` -- report generation (the HR team's concern)

We also introduce interfaces so that each piece is testable and replaceable independently.

---

#### The Domain Entity (Pure Data Carrier)

```java
import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.Objects;

/**
 * Pure domain entity. Contains only data and identity.
 * No persistence logic. No business calculations. No formatting.
 *
 * Reason to change: the Employee domain model itself changes --
 * e.g., adding a new field like 'department' or 'location'.
 * Only one actor: the domain/product team.
 */
public class Employee {

    public enum Type {
        FULL_TIME, PART_TIME, CONTRACTOR
    }

    private final long id;
    private final String name;
    private final String email;
    private final Type type;
    private final BigDecimal baseSalary;
    private final int hoursWorked;
    private final LocalDate hireDate;

    public Employee(long id, String name, String email,
                    Type type, BigDecimal baseSalary, int hoursWorked) {
        this.id = id;
        this.name = Objects.requireNonNull(name, "name must not be null");
        this.email = Objects.requireNonNull(email, "email must not be null");
        this.type = Objects.requireNonNull(type, "type must not be null");
        this.baseSalary = Objects.requireNonNull(baseSalary, "baseSalary must not be null");
        this.hoursWorked = hoursWorked;
        this.hireDate = LocalDate.now();
    }

    public long getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public Type getType() { return type; }
    public BigDecimal getBaseSalary() { return baseSalary; }
    public int getHoursWorked() { return hoursWorked; }
    public LocalDate getHireDate() { return hireDate; }
}
```

---

#### The Business Logic Layer

```java
import java.math.BigDecimal;
import java.math.RoundingMode;

/**
 * Interface for employee business logic.
 * Segregates the contract from the implementation,
 * allowing alternative implementations (e.g., for testing).
 */
public interface EmployeeService {
    BigDecimal calculatePay(Employee employee);
}
```

```java
import java.math.BigDecimal;
import java.math.RoundingMode;

/**
 * Implements payroll calculation rules.
 *
 * Reason to change: payroll policy changes -- overtime rules, tax logic,
 * contractor rate adjustments, part-time hour caps.
 *
 * Single actor: Finance / Payroll team.
 *
 * This class is now trivially unit-testable with no mocking required.
 */
public class DefaultEmployeeService implements EmployeeService {

    private static final int STANDARD_MONTHLY_HOURS = 160;
    private static final int FULL_TIME_WEEKLY_HOURS = 40;
    private static final int HOURS_PER_DAY = 8;
    private static final BigDecimal OVERTIME_MULTIPLIER = new BigDecimal("1.5");

    @Override
    public BigDecimal calculatePay(Employee employee) {
        switch (employee.getType()) {
            case FULL_TIME:  return calculateFullTimePay(employee);
            case PART_TIME:  return calculatePartTimePay(employee);
            case CONTRACTOR: return calculateContractorPay(employee);
            default:
                throw new IllegalArgumentException(
                    "Unsupported employee type: " + employee.getType());
        }
    }

    private BigDecimal calculateFullTimePay(Employee employee) {
        int hoursWorked = employee.getHoursWorked();
        BigDecimal baseSalary = employee.getBaseSalary();

        if (hoursWorked <= FULL_TIME_WEEKLY_HOURS) {
            return baseSalary;
        }

        BigDecimal hourlyRate = baseSalary
            .divide(BigDecimal.valueOf(STANDARD_MONTHLY_HOURS), 4, RoundingMode.HALF_UP);
        BigDecimal overtimeHours = BigDecimal.valueOf(hoursWorked - FULL_TIME_WEEKLY_HOURS);
        BigDecimal overtimePay = overtimeHours.multiply(hourlyRate).multiply(OVERTIME_MULTIPLIER);

        return baseSalary.add(overtimePay).setScale(2, RoundingMode.HALF_UP);
    }

    private BigDecimal calculatePartTimePay(Employee employee) {
        BigDecimal hourlyRate = employee.getBaseSalary()
            .divide(BigDecimal.valueOf(STANDARD_MONTHLY_HOURS), 4, RoundingMode.HALF_UP);
        return hourlyRate
            .multiply(BigDecimal.valueOf(employee.getHoursWorked()))
            .setScale(2, RoundingMode.HALF_UP);
    }

    private BigDecimal calculateContractorPay(Employee employee) {
        BigDecimal daysWorked = BigDecimal.valueOf(employee.getHoursWorked())
            .divide(BigDecimal.valueOf(HOURS_PER_DAY), 4, RoundingMode.HALF_UP);
        return employee.getBaseSalary()
            .multiply(daysWorked)
            .setScale(2, RoundingMode.HALF_UP);
    }
}
```

---

#### The Persistence Layer

```java
import java.util.Optional;

/**
 * Repository contract for Employee persistence.
 * Callers depend on this interface, not on any concrete data access technology.
 */
public interface EmployeeRepository {
    void save(Employee employee);
    Optional<Employee> findById(long id);
    void delete(long id);
}
```

```java
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.Optional;

/**
 * JDBC implementation of EmployeeRepository.
 *
 * Reason to change: SQL schema changes, migration to JPA, connection pool tuning,
 * switching database vendor, adding read replicas.
 *
 * Single actor: DBA / Infrastructure / Platform team.
 *
 * Notice: zero business logic, zero formatting, zero email sending.
 * A JpaEmployeeRepository could replace this without touching any other class.
 */
public class JdbcEmployeeRepository implements EmployeeRepository {

    private static final String INSERT_SQL =
        "INSERT INTO employees (id, name, email, type, base_salary, hours_worked, hire_date) "
      + "VALUES (?, ?, ?, ?, ?, ?, ?) "
      + "ON DUPLICATE KEY UPDATE name=VALUES(name), email=VALUES(email), "
      + "type=VALUES(type), base_salary=VALUES(base_salary), hours_worked=VALUES(hours_worked)";

    private static final String SELECT_BY_ID_SQL =
        "SELECT id, name, email, type, base_salary, hours_worked FROM employees WHERE id = ?";

    private static final String DELETE_SQL =
        "DELETE FROM employees WHERE id = ?";

    private final DataSource dataSource;

    public JdbcEmployeeRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public void save(Employee employee) {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(INSERT_SQL)) {

            stmt.setLong(1, employee.getId());
            stmt.setString(2, employee.getName());
            stmt.setString(3, employee.getEmail());
            stmt.setString(4, employee.getType().name());
            stmt.setBigDecimal(5, employee.getBaseSalary());
            stmt.setInt(6, employee.getHoursWorked());
            stmt.setObject(7, employee.getHireDate());
            stmt.executeUpdate();

        } catch (SQLException e) {
            throw new EmployeeRepositoryException(
                "Failed to save employee with id: " + employee.getId(), e);
        }
    }

    @Override
    public Optional<Employee> findById(long id) {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(SELECT_BY_ID_SQL)) {

            stmt.setLong(1, id);
            ResultSet rs = stmt.executeQuery();

            if (rs.next()) {
                return Optional.of(mapRow(rs));
            }
            return Optional.empty();

        } catch (SQLException e) {
            throw new EmployeeRepositoryException(
                "Failed to find employee with id: " + id, e);
        }
    }

    @Override
    public void delete(long id) {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(DELETE_SQL)) {

            stmt.setLong(1, id);
            stmt.executeUpdate();

        } catch (SQLException e) {
            throw new EmployeeRepositoryException(
                "Failed to delete employee with id: " + id, e);
        }
    }

    private Employee mapRow(ResultSet rs) throws SQLException {
        return new Employee(
            rs.getLong("id"),
            rs.getString("name"),
            rs.getString("email"),
            Employee.Type.valueOf(rs.getString("type")),
            rs.getBigDecimal("base_salary"),
            rs.getInt("hours_worked")
        );
    }
}
```

```java
/**
 * Typed exception for repository failures.
 * Isolates persistence exceptions from domain and reporting exceptions.
 */
public class EmployeeRepositoryException extends RuntimeException {
    public EmployeeRepositoryException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

---

#### The Reporting Layer

```java
/**
 * Contract for payroll reporting.
 */
public interface PayrollReporter {
    String generatePayslip(Employee employee);
    void sendReportToHR(Employee employee);
}
```

```java
import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.Properties;
import javax.mail.Message;
import javax.mail.MessagingException;
import javax.mail.Session;
import javax.mail.Transport;
import javax.mail.internet.InternetAddress;
import javax.mail.internet.MimeMessage;

/**
 * Generates and distributes employee payslip reports.
 *
 * Reason to change: payslip format redesign, new fields required by HR,
 * change in email delivery provider, addition of PDF output.
 *
 * Single actor: HR team / Payroll Operations team.
 *
 * This class delegates pay calculation to EmployeeService rather than
 * re-implementing it. It knows how to format and send; it does not know
 * how to calculate.
 */
public class EmailPayrollReporter implements PayrollReporter {

    private final EmployeeService employeeService;
    private final String smtpHost;
    private final String hrEmail;

    public EmailPayrollReporter(EmployeeService employeeService,
                                 String smtpHost,
                                 String hrEmail) {
        this.employeeService = employeeService;
        this.smtpHost = smtpHost;
        this.hrEmail = hrEmail;
    }

    @Override
    public String generatePayslip(Employee employee) {
        BigDecimal pay = employeeService.calculatePay(employee);
        LocalDate now = LocalDate.now();

        return "===== PAYSLIP =====\n"
             + "Employee : " + employee.getName() + "\n"
             + "ID       : " + employee.getId() + "\n"
             + "Type     : " + employee.getType() + "\n"
             + "Hours    : " + employee.getHoursWorked() + "\n"
             + "Gross Pay: $" + pay + "\n"
             + "Period   : " + now.getMonth() + " " + now.getYear() + "\n"
             + "===================\n";
    }

    @Override
    public void sendReportToHR(Employee employee) {
        String payslip = generatePayslip(employee);

        Properties props = new Properties();
        props.put("mail.smtp.host", smtpHost);
        Session session = Session.getDefaultInstance(props);

        try {
            MimeMessage message = new MimeMessage(session);
            message.setFrom(new InternetAddress(employee.getEmail()));
            message.addRecipient(Message.RecipientType.TO, new InternetAddress(hrEmail));
            message.setSubject("Payslip for " + employee.getName());
            message.setText(payslip);
            Transport.send(message);
        } catch (MessagingException e) {
            throw new PayrollReportException(
                "Failed to send payslip for employee: " + employee.getId(), e);
        }
    }
}
```

```java
public class PayrollReportException extends RuntimeException {
    public PayrollReportException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

---

#### How They Are Wired Together

```java
import javax.sql.DataSource;
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

/**
 * Composition root: the one place that wires all dependencies together.
 * In a Spring application this would be a @Configuration class.
 * Here it is shown as plain Java for clarity.
 *
 * Notice that no business class knows about how it is assembled.
 * Each class receives its dependencies through its constructor (dependency injection).
 */
public class ApplicationConfig {

    public static void main(String[] args) {
        // Build infrastructure
        DataSource dataSource = buildDataSource();

        // Build each focused component independently
        EmployeeRepository repository = new JdbcEmployeeRepository(dataSource);
        EmployeeService service = new DefaultEmployeeService();
        PayrollReporter reporter = new EmailPayrollReporter(
            service,
            "smtp.company.com",
            "hr@company.com"
        );

        // Use them
        Employee emp = new Employee(
            42L, "Jane Smith", "jane@company.com",
            Employee.Type.FULL_TIME,
            new java.math.BigDecimal("5000.00"),
            45  // 5 hours overtime
        );

        repository.save(emp);

        java.math.BigDecimal pay = service.calculatePay(emp);
        System.out.println("Calculated pay: $" + pay);

        String payslip = reporter.generatePayslip(emp);
        System.out.println(payslip);

        reporter.sendReportToHR(emp);
    }

    private static DataSource buildDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/hrdb");
        config.setUsername("appuser");
        config.setPassword("secret");
        return new HikariDataSource(config);
    }
}
```

---

#### Unit Test: Pure Business Logic, Zero Mocks

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import static org.junit.jupiter.api.Assertions.*;

/**
 * Because DefaultEmployeeService has no I/O dependencies, these tests
 * run in microseconds with zero mocking infrastructure.
 *
 * Before the refactoring, this was impossible without mocking DriverManager.
 */
class DefaultEmployeeServiceTest {

    private DefaultEmployeeService service;

    @BeforeEach
    void setUp() {
        service = new DefaultEmployeeService();
    }

    @Test
    void fullTimeEmployeeWithNoOvertimeReceivesBaseSalary() {
        Employee emp = new Employee(1L, "Alice", "alice@co.com",
            Employee.Type.FULL_TIME, new BigDecimal("4000.00"), 40);

        BigDecimal pay = service.calculatePay(emp);

        assertEquals(new BigDecimal("4000.00"), pay);
    }

    @Test
    void fullTimeEmployeeWithFiveOvertimeHoursReceivesOvertimePremium() {
        Employee emp = new Employee(2L, "Bob", "bob@co.com",
            Employee.Type.FULL_TIME, new BigDecimal("4000.00"), 45);

        BigDecimal pay = service.calculatePay(emp);

        // Hourly rate = 4000 / 160 = $25.00
        // Overtime rate = 25 * 1.5 = $37.50
        // Overtime pay = 5 * 37.50 = $187.50
        // Total = 4000 + 187.50 = $4187.50
        assertEquals(new BigDecimal("4187.50"), pay);
    }

    @Test
    void contractorPayIsBasedOnDailyRateTimesDaysWorked() {
        Employee emp = new Employee(3L, "Carol", "carol@co.com",
            Employee.Type.CONTRACTOR, new BigDecimal("500.00"), 16); // 2 days

        BigDecimal pay = service.calculatePay(emp);

        assertEquals(new BigDecimal("1000.00"), pay);
    }
}
```

### Summary of What Changed

| Concern | Before | After |
|---|---|---|
| Business logic | Buried in `Employee` | `DefaultEmployeeService` (testable, no I/O) |
| Persistence | Static JDBC in `Employee` | `JdbcEmployeeRepository` behind `EmployeeRepository` interface |
| Reporting | Mixed with domain methods | `EmailPayrollReporter` behind `PayrollReporter` interface |
| Domain entity | Fat class with I/O | Lean `Employee` with only data and identity |
| Testability | Requires DB + SMTP to test pay logic | Pure unit test, no mocks needed |
| Replaceability | Cannot swap DB without touching pay logic | Swap `JdbcEmployeeRepository` for `JpaEmployeeRepository` with zero impact on `EmployeeService` |

---

## 5. Common Anti-Patterns

### Anti-Pattern 1: The God Class

Already shown in the `Employee` example above. The defining characteristic is that the class is the center of gravity for multiple unrelated concerns.

Short example of a different God class:

```java
// Violates SRP: UserManager is responsible to 4+ different actors
public class UserManager {

    // Authentication concern
    public boolean login(String username, String password) { ... }
    public void logout(String sessionId) { ... }
    public String generateJwt(User user) { ... }

    // Profile management concern
    public void updateProfile(long userId, ProfileDto dto) { ... }
    public void uploadAvatar(long userId, byte[] imageData) { ... }

    // Notification concern
    public void sendWelcomeEmail(User user) { ... }
    public void sendPasswordResetEmail(User user) { ... }

    // Billing concern
    public void chargeUser(long userId, BigDecimal amount) { ... }
    public List<Invoice> getInvoices(long userId) { ... }
}
```

The fix is to decompose into `AuthenticationService`, `UserProfileService`, `NotificationService`, and `BillingService`.

---

### Anti-Pattern 2: The Bloated Service

This is a subtler violation. The class name ends in `Service`, which implies it is a focused domain service, but it has grown to absorb tangential concerns:

```java
// Appears to be a focused order service but has absorbed unrelated concerns
public class OrderService {

    // Legitimate: core order business logic
    public Order createOrder(OrderRequest request) { ... }
    public void cancelOrder(long orderId) { ... }
    public Order getOrder(long orderId) { ... }

    // Violation: inventory is a separate bounded context
    public boolean checkStockAvailability(long productId, int quantity) { ... }
    public void reserveStock(long productId, int quantity) { ... }

    // Violation: payment is a separate bounded context
    public PaymentResult processPayment(PaymentDetails details) { ... }
    public void issueRefund(long orderId) { ... }

    // Violation: notification is a separate concern
    public void sendOrderConfirmationEmail(Order order) { ... }
    public void sendShippingNotification(Order order, String trackingId) { ... }
}
```

---

### Anti-Pattern 3: Mixed Abstraction Levels

A class mixes high-level domain logic with low-level technical details. The two levels change for different reasons.

```java
public class ReportGenerator {

    // High-level domain: what the report contains
    public Report buildMonthlyReport(YearMonth period) {
        List<Transaction> transactions = fetchTransactions(period);
        BigDecimal total = transactions.stream()
            .map(Transaction::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        return new Report(period, transactions, total);
    }

    // Low-level technical: how to connect and query
    // This should live in a repository, not here
    private List<Transaction> fetchTransactions(YearMonth period) {
        try (Connection conn = DriverManager.getConnection("jdbc:mysql://...")) {
            PreparedStatement stmt = conn.prepareStatement(
                "SELECT * FROM transactions WHERE YEAR(date)=? AND MONTH(date)=?");
            stmt.setInt(1, period.getYear());
            stmt.setInt(2, period.getMonthValue());
            // ... map results
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
        return null;
    }
}
```

The low-level SQL is a separate concern owned by the data team. When the schema changes, `ReportGenerator` should not need to change.

---

## 6. When SRP Can Be Over-Applied

SRP is a tool for managing change, not a rule that maximizes the number of classes. Over-applying it produces its own problems: excessive indirection, shallow classes that add no value, and cognitive overhead from jumping through many files to trace a simple operation.

### Guideline 1: Do Not Split What Is Cohesively One Thing

If all methods in a class operate on the same data, transform it in related ways, and would logically be changed together by the same team, they belong together even if there are many of them.

```java
// These methods all deal with the same data in related ways.
// Splitting them apart would not help -- they change for the same reason.
public class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money add(Money other) { ... }
    public Money subtract(Money other) { ... }
    public Money multiply(BigDecimal factor) { ... }
    public boolean isGreaterThan(Money other) { ... }
    public Money convertTo(Currency target, ExchangeRate rate) { ... }
}
```

`Money` has many methods but a single responsibility: value arithmetic for monetary amounts. Splitting it into `MoneyAdder`, `MoneySubtractor`, `MoneyComparator` would be absurd.

### Guideline 2: YAGNI -- Do Not Pre-emptively Split for Hypothetical Actors

If right now, one team owns a class and there is no concrete indication that a second actor will ever exist, do not split it. The cost of introducing premature abstraction is real: more files, more interfaces, more cognitive load, and harder onboarding. Wait for the second actor to appear before splitting.

### Guideline 3: The Cost of Abstraction Must Be Justified

Every interface, every repository, every additional class is a unit of indirection. In a small application or a script, the overhead of strict SRP may outweigh the benefit. Apply SRP aggressively in:
- Long-lived production systems with multiple teams
- Code that will be tested in isolation
- Code that is likely to change independently

Apply it lightly in:
- Prototypes and proof-of-concept code
- Small utilities and scripts
- Code behind a single-implementation interface where the interface serves no concrete purpose yet

### Guideline 4: Cohesion Should Guide, Not Granularity

The goal is high cohesion. If splitting a class produces two classes where each class still feels like it belongs together internally, the split was correct. If splitting produces two classes where each class feels arbitrary, you have over-applied SRP.

---

## 7. The Cohesion Connection

### What Is Cohesion?

Cohesion is the degree to which the elements inside a module belong together. A highly cohesive class has methods that all operate on the same data, serve the same purpose, and change for the same reason. A loosely cohesive class has methods that are only loosely related and could be separated without losing meaning.

### SRP Maximizes Cohesion

SRP is effectively the prescription for achieving high cohesion. When a class has exactly one responsibility -- one actor, one reason to change -- all its methods and fields necessarily relate to that single purpose. There is no room for unrelated methods to creep in.

Conversely, measuring cohesion is a way to detect SRP violations before they become obvious:

- **Functional cohesion (ideal)**: all parts contribute to a single, well-defined task. This is what SRP produces.
- **Logical cohesion (warning sign)**: methods are grouped because they are the same *kind* of operation, not because they serve the same purpose. Example: a `Utils` class.
- **Coincidental cohesion (violation)**: elements are grouped arbitrarily. Example: a God class that grew by accumulation.

### Practical Cohesion Check

After writing a class, ask: "If I remove one method from this class and put it in a new class, does the remaining class still make complete sense, and does the new class also make complete sense?" If yes, the original class had low cohesion and likely violated SRP. If the separation feels violent and unnatural, cohesion was high and the class was correct.

### LCOM: Lack of Cohesion of Methods (Conceptual)

LCOM is a formal metric. In simplified terms: if you draw a graph where methods are nodes and two methods share an edge if they access the same instance variable, a highly cohesive class has one connected component. A class with two connected components is practically two classes in one file, which is an SRP violation.

You do not need to calculate LCOM by hand, but the mental model is useful: if you can draw a clear boundary through the class's field-access graph, the class should be split.

---

## 8. Interview Questions and Model Answers

---

### Q1: What is the Single Responsibility Principle in your own words? How does the actor model definition differ from the original definition?

**Model Answer:**

SRP states that a class should have exactly one reason to change. In practical terms, this means only one stakeholder group -- one actor -- should be able to demand changes to a given class.

The original "reason to change" framing focuses on the code: you look at a class and ask whether two independent types of requirements could force you to modify it. The actor model framing focuses on people and organizational structure: you ask whether two different teams or stakeholders could independently send you a change request for this class.

The actor model is more actionable in practice because you can map it directly to your organization. If the finance team and the DBA team can both send change requests that touch `Employee.java`, that class serves two actors and violates SRP. The fix is to split along the actor boundary.

---

### Q2: Give a concrete example of an SRP violation and explain what the real-world consequences are.

**Model Answer:**

Consider an `Employee` class that contains `calculatePay()`, `save()` to a database, and `generatePayslip()`. Three teams own these concerns: finance, infrastructure, and HR.

Real-world consequences:

First, **merge conflicts**: if the finance team is updating overtime rules at the same time the DBA team is optimizing the SQL query, they both open the same file and create a conflict that requires human resolution.

Second, **regression risk**: when the DBA modifies the SQL in `save()`, the `calculatePay()` method is right there in the same file. A careless edit or a bad merge can accidentally break pay calculation -- a very sensitive function.

Third, **testability**: to unit-test `calculatePay()`, you must have a working database connection because `save()` is in the same class and the class may initialize DB connections in its constructor.

Fourth, **redeployment coupling**: if you use module-level deployment, changing the payslip format forces a redeployment of the entire module, including the risk to payroll calculation -- even though only the report format changed.

---

### Q3: How do you identify an SRP violation during a code review?

**Model Answer:**

I look for several signals:

1. **Method clustering**: I mentally group the methods by what data they touch and what concern they serve. If I can draw a clear boundary between two clusters, that is a violation.

2. **Import diversity**: If the import list spans JDBC, an email library, a PDF library, and the domain package, the class is almost certainly doing too much.

3. **Divergent change and shotgun surgery patterns**: Divergent change means "this class gets changed for many different reasons." Shotgun surgery means "this change requires editing many classes." SRP violations cause divergent change on the violating class.

4. **Test setup complexity**: If a unit test for one method requires mocking five unrelated collaborators, those collaborators should not all belong to the same class.

5. **Git log analysis**: If the commit history shows alternating messages from different functional areas ("fix payroll calculation" vs. "update email template" vs. "optimize query"), different actors are driving changes to the same file.

6. **Class name vagueness**: Names like `Manager`, `Handler`, `Helper`, `Processor` with no qualifying domain noun are red flags.

---

### Q4: Is it possible to violate SRP at the method level, not just the class level?

**Model Answer:**

Yes. SRP applies at every granularity of design -- methods, classes, modules, and services.

A method violates SRP when it does more than one thing. The heuristic is: if you cannot describe a method in one sentence without using "and" or "then", it probably has more than one responsibility.

```java
// Violates SRP at the method level
public void processAndSaveAndNotify(Order order) {
    // validates the order
    if (order.getItems().isEmpty()) throw new IllegalArgumentException("No items");

    // persists it
    orderRepository.save(order);

    // sends a notification
    emailService.sendConfirmation(order.getCustomerEmail(), order);
}
```

This method should be three methods: `validate(Order)`, `save(Order)`, and `notifyCustomer(Order)`, coordinated by an orchestrating method that calls them in sequence. This makes each step independently testable and independently changeable.

---

### Q5: How does SRP relate to the Open/Closed Principle and the Dependency Inversion Principle?

**Model Answer:**

The SOLID principles reinforce each other. SRP is often a prerequisite for the others to work properly.

SRP enables OCP: a class with a single responsibility is easier to extend without modification because its boundary is clear. You know exactly what you are adding to and what you are leaving alone. A God class is almost impossible to extend safely because you cannot be sure what you will break.

SRP enables DIP: once you have decomposed a God class into focused classes, you can extract each focused class into an interface. Callers then depend on the abstract interface, not the concrete implementation. In the `Employee` example, `EmployeeRepository` the interface emerged naturally after the persistence concern was isolated. Before the split, there was no clear boundary to extract an interface around.

In summary: SRP creates the clean units that OCP can protect and DIP can invert.

---

### Q6: Can a class have many methods and still comply with SRP?

**Model Answer:**

Yes, absolutely. The number of methods is not the measure of SRP compliance. Cohesion is. A class can have twenty methods and a single responsibility if all twenty methods serve the same purpose and would change for the same reason.

A `UserQueryService` could have `findById`, `findByEmail`, `findByUsername`, `findAllActive`, `findAllByDepartment`, `searchByName`, and many more query methods. All of them serve one concern -- querying user data -- and all of them would be changed by the same actor: the team responsible for user data access.

Conversely, a class with only three methods can violate SRP if those three methods serve three different actors, as in the minimal `Employee` example:

```java
class Employee {
    void calculatePay() { ... }   // finance team
    void save() { ... }           // DBA team
    void sendPayslip() { ... }    // HR team
}
```

Three methods, three actors: a clear violation.

---

### Q7: How do you handle the trade-off between SRP and the pragmatic cost of creating too many small classes?

**Model Answer:**

The trade-off is real. I approach it by asking whether the split is motivated by a concrete, present actor or by a hypothetical future one.

If two concerns are currently owned by the same team and change together in practice, I do not split them. The YAGNI principle applies: you are not going to need that abstraction until the second actor appears.

When the second actor does appear -- when I see that two different teams are making independent changes to the same class -- I treat that as the signal to refactor. The refactoring is motivated by a real cost that the violation is already creating, not by theory.

I also consider the lifetime and scale of the system. In a small internal tool used by one team, strict SRP adds overhead without proportional benefit. In a large, long-lived production system with multiple teams, the cost of violations compounds over years: merge conflicts, regressions, inability to deploy changes independently. The payoff from SRP discipline is much higher there.

The goal is not to maximize the number of classes. The goal is to minimize the coupling between concerns that change independently. If two things always change together, they belong together.

---

### Q8: Walk me through how you would refactor a God class in a live production system.

**Model Answer:**

Refactoring a God class in production requires a disciplined, incremental approach because a big-bang rewrite risks introducing bugs across many unrelated behaviors simultaneously.

**Step 1: Characterize with tests.** Before changing anything, write integration or characterization tests that capture the current behavior of every public method. These tests are your safety net.

**Step 2: Identify the responsibilities.** Group the methods into clusters by actor. Draw the boundary explicitly.

**Step 3: Extract one responsibility at a time.** Start with the easiest or most valuable split. Create a new class. Move methods into it. Make the God class delegate to the new class. This keeps the God class as the public API temporarily (no callers break).

```java
// Intermediate step: God class delegates instead of implementing
public class Employee {
    private final EmployeePayCalculator payCalculator = new EmployeePayCalculator();

    public BigDecimal calculatePay() {
        return payCalculator.calculate(this); // delegates
    }

    // persistence and reporting still here for now
}
```

**Step 4: Update callers incrementally.** Identify callers that only use the extracted concern and point them directly at the new class.

**Step 5: Repeat for each responsibility.** Once all callers of the first extracted class are updated, remove the delegation from the God class. Then extract the next responsibility.

**Step 6: Delete the God class (or rename it).** Once all responsibilities are extracted, the God class shell can be deleted or renamed to reflect whatever single thing it still does.

Running tests at each step ensures no regressions are introduced.

---

### Q9: How does SRP apply to microservices, not just classes?

**Model Answer:**

The actor model scales directly to microservice architecture. At the service level, SRP says a microservice should be responsible to one bounded context or one actor in the business domain.

A common violation is the "mega-service": a service that grew to own user management, order processing, inventory, and notifications because it was convenient to keep adding endpoints. Each of those domains has a different team and a different rate of change. Coupling them in one service means:

- Deploying a notification change requires redeploying the user management code.
- The inventory team's database schema is in the same migrations as the order team's schema.
- An outage in the payment processing logic brings down user profile reads.

The fix is the same as at the class level: decompose by actor. The team that owns orders owns the order service. The team that owns inventory owns the inventory service.

The heuristic from Amazon's Jeff Bezos -- the two-pizza team rule -- is a proxy for the actor model: if a service needs more than a two-pizza team to understand and operate it, it is serving too many actors.

---

### Q10: How would you explain SRP to a junior developer in one minute?

**Model Answer:**

Imagine your class is an employee's job description. Now imagine that employee has three bosses: the finance director, the DBA, and the HR manager. Each boss can independently walk up to that employee and ask them to change how they work.

That is a problem. When the finance director asks for a change, the employee has to edit the same document that the other two bosses also rely on. Any mistake could affect their work too.

SRP says: give each employee only one boss. The finance director's logic goes in one class. The DBA's logic goes in another. The HR manager's logic goes in a third. Now when one boss sends a change request, only that one class changes. Nothing else is at risk.

In code, "boss" means "actor" or "stakeholder." The question to ask for every class you write is: "How many different kinds of people could ask me to change this?" If the answer is more than one, split the class.

---

## 9. Assignment: Refactor the OrderProcessor

### The Bloated Class

Study the following `OrderProcessor` class carefully. It was written by a developer working under time pressure and has accumulated responsibilities from multiple teams.

```java
import java.math.BigDecimal;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Properties;
import java.util.UUID;
import javax.mail.Message;
import javax.mail.MessagingException;
import javax.mail.Session;
import javax.mail.Transport;
import javax.mail.internet.InternetAddress;
import javax.mail.internet.MimeMessage;

public class OrderProcessor {

    private static final String DB_URL = "jdbc:mysql://localhost:3306/shop";
    private static final String DB_USER = "root";
    private static final String DB_PASS = "secret";
    private static final String SMTP_HOST = "smtp.company.com";
    private static final String PAYMENT_GATEWAY_URL = "https://pay.gateway.com/api/charge";

    // ---- Order validation ----

    public boolean validateOrder(Order order) {
        if (order == null) return false;
        if (order.getCustomerId() == null) return false;
        if (order.getItems() == null || order.getItems().isEmpty()) return false;
        for (OrderItem item : order.getItems()) {
            if (item.getQuantity() <= 0) return false;
            if (item.getUnitPrice().compareTo(BigDecimal.ZERO) <= 0) return false;
        }
        BigDecimal total = order.getItems().stream()
            .map(i -> i.getUnitPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        if (total.compareTo(new BigDecimal("0.01")) < 0) return false;
        return true;
    }

    // ---- Inventory checking ----

    public boolean checkInventory(Order order) {
        String sql = "SELECT stock_quantity FROM products WHERE product_id = ?";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASS)) {
            for (OrderItem item : order.getItems()) {
                PreparedStatement stmt = conn.prepareStatement(sql);
                stmt.setString(1, item.getProductId());
                var rs = stmt.executeQuery();
                if (!rs.next() || rs.getInt("stock_quantity") < item.getQuantity()) {
                    return false;
                }
            }
            return true;
        } catch (SQLException e) {
            throw new RuntimeException("Inventory check failed", e);
        }
    }

    public void reserveInventory(Order order) {
        String sql = "UPDATE products SET stock_quantity = stock_quantity - ? WHERE product_id = ?";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASS)) {
            for (OrderItem item : order.getItems()) {
                PreparedStatement stmt = conn.prepareStatement(sql);
                stmt.setInt(1, item.getQuantity());
                stmt.setString(2, item.getProductId());
                stmt.executeUpdate();
            }
        } catch (SQLException e) {
            throw new RuntimeException("Inventory reservation failed", e);
        }
    }

    // ---- Payment processing ----

    public PaymentResult processPayment(Order order, PaymentDetails paymentDetails) {
        BigDecimal total = order.getItems().stream()
            .map(i -> i.getUnitPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        // Simplified: in reality this would call the payment gateway via HTTP
        String transactionId = UUID.randomUUID().toString();
        boolean success = total.compareTo(new BigDecimal("10000.00")) < 0; // fake limit

        if (success) {
            logPaymentToDb(order.getOrderId(), transactionId, total);
        }

        return new PaymentResult(success, transactionId, total);
    }

    private void logPaymentToDb(String orderId, String transactionId, BigDecimal amount) {
        String sql = "INSERT INTO payments (order_id, transaction_id, amount, created_at) VALUES (?, ?, ?, ?)";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASS);
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            stmt.setString(1, orderId);
            stmt.setString(2, transactionId);
            stmt.setBigDecimal(3, amount);
            stmt.setObject(4, LocalDateTime.now());
            stmt.executeUpdate();
        } catch (SQLException e) {
            throw new RuntimeException("Failed to log payment", e);
        }
    }

    // ---- Email notification ----

    public void sendOrderConfirmation(Order order, String customerEmail) {
        String subject = "Order Confirmation - " + order.getOrderId();
        StringBuilder body = new StringBuilder();
        body.append("Dear Customer,\n\n");
        body.append("Your order ").append(order.getOrderId()).append(" has been confirmed.\n\n");
        body.append("Items:\n");
        for (OrderItem item : order.getItems()) {
            body.append("  - ").append(item.getProductId())
                .append(" x").append(item.getQuantity())
                .append(" @ $").append(item.getUnitPrice()).append("\n");
        }
        body.append("\nThank you for your purchase!\n");

        sendEmail(customerEmail, subject, body.toString());
    }

    public void sendPaymentFailureNotification(Order order, String customerEmail) {
        sendEmail(
            customerEmail,
            "Payment Failed - " + order.getOrderId(),
            "Your payment for order " + order.getOrderId() + " could not be processed. Please try again."
        );
    }

    private void sendEmail(String to, String subject, String body) {
        Properties props = new Properties();
        props.put("mail.smtp.host", SMTP_HOST);
        Session session = Session.getDefaultInstance(props);
        try {
            MimeMessage message = new MimeMessage(session);
            message.setFrom(new InternetAddress("orders@company.com"));
            message.addRecipient(Message.RecipientType.TO, new InternetAddress(to));
            message.setSubject(subject);
            message.setText(body);
            Transport.send(message);
        } catch (MessagingException e) {
            throw new RuntimeException("Failed to send email to: " + to, e);
        }
    }

    // ---- Audit logging ----

    public void auditLog(String orderId, String action, String performedBy) {
        String sql = "INSERT INTO audit_log (order_id, action, performed_by, timestamp) VALUES (?, ?, ?, ?)";
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASS);
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            stmt.setString(1, orderId);
            stmt.setString(2, action);
            stmt.setString(3, performedBy);
            stmt.setObject(4, LocalDateTime.now());
            stmt.executeUpdate();
        } catch (SQLException e) {
            throw new RuntimeException("Failed to write audit log for order: " + orderId, e);
        }
    }

    public List<AuditEntry> getAuditHistory(String orderId) {
        // ... query audit_log table and return list
        return List.of();
    }

    // ---- Supporting types (would normally be in separate files) ----

    public static class Order {
        private String orderId;
        private String customerId;
        private List<OrderItem> items;
        public String getOrderId() { return orderId; }
        public String getCustomerId() { return customerId; }
        public List<OrderItem> getItems() { return items; }
    }

    public static class OrderItem {
        private String productId;
        private int quantity;
        private BigDecimal unitPrice;
        public String getProductId() { return productId; }
        public int getQuantity() { return quantity; }
        public BigDecimal getUnitPrice() { return unitPrice; }
    }

    public static class PaymentDetails {
        private String cardToken;
        private String billingAddress;
        public String getCardToken() { return cardToken; }
    }

    public static class PaymentResult {
        private final boolean success;
        private final String transactionId;
        private final BigDecimal amount;
        public PaymentResult(boolean success, String transactionId, BigDecimal amount) {
            this.success = success;
            this.transactionId = transactionId;
            this.amount = amount;
        }
        public boolean isSuccess() { return success; }
        public String getTransactionId() { return transactionId; }
        public BigDecimal getAmount() { return amount; }
    }

    public static class AuditEntry {
        private String orderId;
        private String action;
        private String performedBy;
        private LocalDateTime timestamp;
    }
}
```

---

### Your Tasks

**Task 1: Identify the violations.**

For each responsibility in `OrderProcessor`, answer:
1. What is the responsibility?
2. Who is the actor (which team or stakeholder owns this concern)?
3. What would force a change to this code that is independent of the other responsibilities?

Write your analysis in a comment block or a separate document before writing any code.

**Task 2: Define the class structure.**

Propose the new set of classes and interfaces. For each proposed class, state:
- Its name
- Its single responsibility
- Its actor
- The interface it implements (if any)

**Task 3: Implement the refactored design.**

Write the full Java implementation. Requirements:
- Extract each responsibility into a focused class with a single actor.
- Define interfaces for each extracted class so callers depend on abstractions.
- The `Order`, `OrderItem`, `PaymentDetails`, `PaymentResult`, and `AuditEntry` types should become standalone domain classes (not nested).
- Inject all dependencies through constructors. No static field configuration baked into classes.
- Write at least two unit tests for `OrderValidator.validate()` that require zero mocks.
- Write a composition root (a `main` method or a `Config` class) that wires all components together.

**Task 4: Reflect.**

Answer the following questions in comments at the top of your composition root:
1. Which responsibility was most tempting to leave in `OrderProcessor`? Why?
2. How many files did you end up creating? Does that feel like too many? Why or why not?
3. If the payment gateway vendor changes, which files need to change in your new design? In the original design?
4. If the audit log needs to write to a file instead of a database, which files need to change in your new design? In the original design?

---

### Expected Outcome Structure

You should arrive at approximately the following set of classes (exact names may vary):

| Class | Interface | Responsibility | Actor |
|---|---|---|---|
| `Order` | -- | Domain entity | Domain / product team |
| `OrderItem` | -- | Domain entity | Domain / product team |
| `PaymentDetails` | -- | Value object | Domain / product team |
| `PaymentResult` | -- | Value object | Domain / product team |
| `AuditEntry` | -- | Value object | Compliance team |
| `OrderValidator` | `OrderValidationService` | Business rules validation | Product / Business rules team |
| `JdbcInventoryRepository` | `InventoryRepository` | Inventory data access | DBA / Inventory team |
| `PaymentGatewayClient` | `PaymentService` | Payment processing | Payments team |
| `SmtpNotificationService` | `NotificationService` | Customer notifications | CX / Notifications team |
| `JdbcAuditRepository` | `AuditService` | Audit trail persistence | Compliance / Security team |
| `OrderOrchestrator` | -- | Coordinates the full order flow | Engineering / Product team |

`OrderOrchestrator` is the class that replaces the original `OrderProcessor` at the orchestration level: it calls `OrderValidationService`, `InventoryRepository`, `PaymentService`, `NotificationService`, and `AuditService` in the correct sequence, but it implements none of their logic itself.

---

*End of Module: Single Responsibility Principle*
