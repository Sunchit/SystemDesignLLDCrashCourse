# Capstone Project 1: Online Library Platform

---

## Project Overview

A full-featured digital library platform supporting ebook, audiobook, and physical book borrowing with membership tiers, waitlists, fines, and recommendations.

This capstone integrates every LLD concept covered in the course:
- **Enumerations** for domain constants (BookFormat, BookStatus, MembershipTier)
- **Strategy Pattern** for fine calculation (FineCalculator)
- **Factory Pattern** for policy and calculator instantiation
- **Observer Pattern** for waitlist notifications (WaitlistObserver)
- **Interface segregation** across BorrowingPolicy, RecommendationEngine, WaitlistObserver
- **Domain modeling** with rich entities (Book, Member, BorrowingRecord)
- **Service layer** orchestrating domain objects (LibraryService, LibraryAdmin)

---

## Requirements

### Functional Requirements

1. **Book Catalog** — manage books with multiple formats: Ebook, Audiobook, Physical.
2. **Member Management** — support subscription tiers:
   - **Free**: borrow up to 2 books per active period, 14-day loan window.
   - **Premium**: unlimited borrows, 21-day loan window.
3. **Borrowing System** — enforce per-tier limits; track borrow dates and due dates.
4. **Waitlist Management** — when a book is unavailable, members can join a FIFO waitlist; notify the first member when the book becomes available.
5. **Fine System**:
   - Physical books overdue: **$0.50 per day** after due date.
   - Ebook / Audiobook: **no fine** (digital license auto-expires).
6. **Recommendation Engine** — interface with a default implementation that recommends books by the same author; falls back to any available book.
7. **Admin Operations** — add book, remove book, view overdue borrowing report.

### Non-Functional Requirements

- **Open/Closed Principle**: new book formats must not require modifying existing calculator or policy classes.
- **Strategy Pattern**: fine calculation strategy is injected at runtime via FineCalculatorFactory.
- **Observer Pattern**: waitlist notifications are decoupled from waitlist state management.
- **Single Responsibility**: each class has one reason to change.
- **Dependency Inversion**: LibraryService depends on interfaces, not concrete implementations.

---

## Domain Model

### Entities

**Book** is the central catalog entity. It carries an ISBN (unique identifier), title, author name, the BookFormat it belongs to (EBOOK, AUDIOBOOK, PHYSICAL), and a current BookStatus (AVAILABLE, BORROWED, RESERVED). The status is mutable — it changes as members borrow and return books.

**Member** represents a registered library user. Each member has a unique memberId, a display name, a MembershipTier (FREE or PREMIUM), a list of currently borrowed book ISBNs, and an accumulated fine balance. The fine balance increases when overdue physical books are returned; the member can pay down the balance explicitly.

**BorrowingRecord** is the transactional log of a single borrow event. It links a member to a book via their identifiers, records the borrow date and computed due date (borrow date + policy duration), and optionally stores a return date when the book comes back. It exposes helpers to check whether the record is overdue and to calculate days overdue.

### Value Objects / Supporting Types

**WaitlistEntry** captures a single member's position in the queue for a book: memberId, bookIsbn, and the timestamp of the enqueue action (used to ensure deterministic FIFO ordering).

### Enumerations

**BookFormat** — EBOOK, AUDIOBOOK, PHYSICAL.
**BookStatus** — AVAILABLE, BORROWED, RESERVED.
**MembershipTier** — FREE, PREMIUM.

### Service / Policy Objects

**BorrowingPolicy** — per-tier rules: whether a member can borrow (based on current borrow count), the maximum number of books, and the loan duration in days.

**FineCalculator** — computes the fine owed given the number of overdue days. Physical books use $0.50/day; digital formats use zero.

**WaitlistManager** — owns a per-book Queue of WaitlistEntry objects and a list of WaitlistObserver listeners. When a book transitions back to AVAILABLE, it dequeues the next entry and fires all observers.

**RecommendationEngine** — given a member, returns a list of recommended books from the catalog.

**LibraryService** — orchestrates borrowing, returning, reservations, fine payment, and member registration. Delegates fine calculation and policy enforcement to strategy objects.

**LibraryAdmin** — administrative surface: adding and removing books, and generating overdue reports.

---

## Class Diagram (Text Format)

```
+------------------+       +------------------+       +------------------+
|   <<enum>>       |       |   <<enum>>       |       |   <<enum>>       |
|   BookFormat     |       |   BookStatus     |       | MembershipTier   |
+------------------+       +------------------+       +------------------+
| EBOOK            |       | AVAILABLE        |       | FREE             |
| AUDIOBOOK        |       | BORROWED         |       | PREMIUM          |
| PHYSICAL         |       | RESERVED         |       +------------------+
+------------------+       +------------------+

+-----------------------------+         +-----------------------------+
|           Book              |         |           Member            |
+-----------------------------+         +-----------------------------+
| - isbn: String              |         | - memberId: String          |
| - title: String             |         | - name: String              |
| - author: String            |         | - tier: MembershipTier      |
| - format: BookFormat        |         | - borrowedBooks: List<String>|
| - status: BookStatus        |         | - totalFines: double        |
+-----------------------------+         +-----------------------------+
| + getters / setters         |         | + getters                   |
| + equals / hashCode         |         | + addBorrowedBook()         |
| + toString                  |         | + removeBorrowedBook()      |
+-----------------------------+         | + addFine()                 |
                                        | + payFine()                 |
                                        | + toString                  |
                                        +-----------------------------+

+--------------------------------------------+
|            BorrowingRecord                 |
+--------------------------------------------+
| - recordId: UUID                           |
| - bookIsbn: String                         |
| - memberId: String                         |
| - borrowDate: LocalDate                    |
| - dueDate: LocalDate                       |
| - returnDate: LocalDate (nullable)         |
| - returned: boolean                        |
+--------------------------------------------+
| + getters                                  |
| + markReturned(returnDate)                 |
| + isOverdue(): boolean                     |
| + getDaysOverdue(): long                   |
| + toString                                 |
+--------------------------------------------+

+-----------------------------+       +-------------------------------+
|   <<interface>>             |       |   <<interface>>               |
|   FineCalculator            |       |   BorrowingPolicy             |
+-----------------------------+       +-------------------------------+
| + calculateFine(days): double|      | + canBorrow(member): boolean  |
+-----------------------------+       | + getBorrowLimit(): int        |
        ^           ^                 | + getBorrowDurationDays(): int |
        |           |                 +-------------------------------+
        |           |                         ^             ^
+---------------+ +---------------+           |             |
| PhysicalBook  | |   NoFine      |  +------------------+ +------------------+
| FineCalculator| | Calculator    |  | FreeMemberPolicy | |PremiumMemberPolicy|
+---------------+ +---------------+  +------------------+ +------------------+

+----------------------------+     +--------------------------------+
| FineCalculatorFactory      |     | BorrowingPolicyFactory         |
+----------------------------+     +--------------------------------+
| + getCalculator(format)    |     | + getPolicy(tier): Policy      |
+----------------------------+     +--------------------------------+

+-------------------------------+
|        WaitlistEntry          |
+-------------------------------+
| - memberId: String            |
| - bookIsbn: String            |
| - timestamp: LocalDateTime    |
+-------------------------------+
| + getters                     |
+-------------------------------+

+------------------------------+       +--------------------------------+
|  <<interface>>               |       |   WaitlistManager              |
|  WaitlistObserver            |       +--------------------------------+
+------------------------------+       | - waitlists: Map<String,Queue> |
| + onBookAvailable(memberId,  |       | - observers: List<Observer>    |
|   bookIsbn)                  |       +--------------------------------+
+------------------------------+       | + addToWaitlist()              |
        ^                              | + removeFromWaitlist()         |
        |                              | + notifyNext()                 |
+--------------------------------+     | + addObserver()                |
| MemberNotificationObserver     |     | + hasWaiters()                 |
+--------------------------------+     | + getQueueSize()               |
| + onBookAvailable(...)         |     +--------------------------------+
+--------------------------------+

+------------------------------+       +-----------------------------+
| <<interface>>                |       |  SimpleCatalogRecommendation|
| RecommendationEngine         |       |  Engine                     |
+------------------------------+       +-----------------------------+
| + recommend(member, catalog, |       | + recommend(...)            |
|   limit): List<Book>         |       +-----------------------------+
+------------------------------+

+-----------------------------+     +-----------------------------+
|       BookCatalog           |     |      LibraryService         |
+-----------------------------+     +-----------------------------+
| - books: Map<String,Book>   |     | - catalog: BookCatalog      |
+-----------------------------+     | - members: Map<String,Member|
| + addBook()                 |     | - borrowings: List<Record>  |
| + removeBook()              |     | - waitlistManager           |
| + getBook()                 |     | - recommendationEngine      |
| + searchByTitle()           |     +-----------------------------+
| + searchByAuthor()          |     | + borrowBook()              |
| + searchByFormat()          |     | + returnBook()              |
| + getAllAvailable()          |     | + reserveBook()             |
| + getAllBooks()              |     | + cancelReservation()       |
+-----------------------------+     | + payFine()                 |
                                    | + registerMember()          |
+-----------------------------+     | + getMember()               |
|      LibraryAdmin           |     | + getActiveBorrowings()     |
+-----------------------------+     | + getOverdueRecords()       |
| - service: LibraryService   |     +-----------------------------+
| - catalog: BookCatalog      |
+-----------------------------+
| + addBook()                 |
| + removeBook()              |
| + generateOverdueReport()   |
+-----------------------------+
```

---

## Design Patterns Applied

### 1. Strategy Pattern — Fine Calculation

**Intent**: Define a family of algorithms (fine rules), encapsulate each one, and make them interchangeable.

**Where used**: `FineCalculator` interface with `PhysicalBookFineCalculator` and `NoFineCalculator` implementations. `FineCalculatorFactory` selects the correct strategy at runtime based on `BookFormat`. `LibraryService.returnBook()` calls the factory to get the right calculator without any `if/else` on format.

**Benefit**: Adding a new format (e.g., MAGAZINE with $0.25/day) requires only a new class and one line in the factory — zero changes to `LibraryService`.

### 2. Observer Pattern — Waitlist Notifications

**Intent**: Define a one-to-many dependency so that when one object changes state, all its dependents are notified automatically.

**Where used**: `WaitlistObserver` interface; `MemberNotificationObserver` is the concrete observer. `WaitlistManager` holds the observer list and fires `onBookAvailable(memberId, bookIsbn)` when a book becomes free. `LibraryService.returnBook()` calls `waitlistManager.notifyNext()` after a book status is reset.

**Benefit**: New notification channels (email, SMS, push notification) are added by implementing `WaitlistObserver` and registering with `WaitlistManager` — no changes to `LibraryService` or `WaitlistManager`.

### 3. Factory Method Pattern — Policy and Calculator Creation

**Where used**: `FineCalculatorFactory.getCalculator(BookFormat)` and `BorrowingPolicyFactory.getPolicy(MembershipTier)`.

**Benefit**: Centralises the creation logic. Service code never references concrete classes.

### 4. Template / Policy Object Pattern — Borrowing Rules

**Where used**: `BorrowingPolicy` interface with `FreeMemberPolicy` and `PremiumMemberPolicy`. `LibraryService` asks the policy whether a member can borrow and for how many days.

---

## Complete Java Implementation

### BookFormat.java

```java
package library;

/**
 * Represents the physical or digital format of a book in the library catalog.
 * New formats can be added here without modifying any existing business logic class,
 * satisfying the Open/Closed Principle when combined with FineCalculatorFactory
 * and BorrowingPolicyFactory.
 */
public enum BookFormat {
    EBOOK,
    AUDIOBOOK,
    PHYSICAL
}
```

### BookStatus.java

```java
package library;

/**
 * Represents the current availability status of a single book copy.
 *
 * AVAILABLE  - the book can be borrowed immediately.
 * BORROWED   - the book is currently checked out by a member.
 * RESERVED   - the book has been reserved for pickup (future extension).
 */
public enum BookStatus {
    AVAILABLE,
    BORROWED,
    RESERVED
}
```

### MembershipTier.java

```java
package library;

/**
 * Defines the membership subscription tiers available to library members.
 *
 * FREE     - limited to 2 concurrent borrows, 14-day loan period.
 * PREMIUM  - unlimited concurrent borrows, 21-day loan period.
 */
public enum MembershipTier {
    FREE,
    PREMIUM
}
```

### Book.java

```java
package library;

import java.util.Objects;

/**
 * Represents a single book in the library catalog.
 *
 * Invariants:
 *  - isbn must be non-null and unique across the catalog.
 *  - status transitions: AVAILABLE -> BORROWED -> AVAILABLE (or RESERVED).
 */
public class Book {

    private final String isbn;
    private String title;
    private String author;
    private final BookFormat format;
    private BookStatus status;

    /**
     * Constructs a new Book. Initial status is always AVAILABLE.
     *
     * @param isbn   unique International Standard Book Number
     * @param title  display title of the book
     * @param author full author name
     * @param format the format of this book (EBOOK, AUDIOBOOK, PHYSICAL)
     */
    public Book(String isbn, String title, String author, BookFormat format) {
        if (isbn == null || isbn.isBlank()) {
            throw new IllegalArgumentException("ISBN must not be null or blank.");
        }
        if (title == null || title.isBlank()) {
            throw new IllegalArgumentException("Title must not be null or blank.");
        }
        if (author == null || author.isBlank()) {
            throw new IllegalArgumentException("Author must not be null or blank.");
        }
        if (format == null) {
            throw new IllegalArgumentException("BookFormat must not be null.");
        }
        this.isbn = isbn;
        this.title = title;
        this.author = author;
        this.format = format;
        this.status = BookStatus.AVAILABLE;
    }

    // -------------------------------------------------------------------------
    // Getters
    // -------------------------------------------------------------------------

    public String getIsbn() {
        return isbn;
    }

    public String getTitle() {
        return title;
    }

    public String getAuthor() {
        return author;
    }

    public BookFormat getFormat() {
        return format;
    }

    public BookStatus getStatus() {
        return status;
    }

    // -------------------------------------------------------------------------
    // Setters
    // -------------------------------------------------------------------------

    public void setTitle(String title) {
        if (title == null || title.isBlank()) {
            throw new IllegalArgumentException("Title must not be null or blank.");
        }
        this.title = title;
    }

    public void setAuthor(String author) {
        if (author == null || author.isBlank()) {
            throw new IllegalArgumentException("Author must not be null or blank.");
        }
        this.author = author;
    }

    public void setStatus(BookStatus status) {
        if (status == null) {
            throw new IllegalArgumentException("BookStatus must not be null.");
        }
        this.status = status;
    }

    // -------------------------------------------------------------------------
    // Object overrides
    // -------------------------------------------------------------------------

    /**
     * Two books are equal if and only if their ISBNs are equal.
     * ISBN is the natural key of a book.
     */
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Book book = (Book) o;
        return Objects.equals(isbn, book.isbn);
    }

    @Override
    public int hashCode() {
        return Objects.hash(isbn);
    }

    @Override
    public String toString() {
        return "Book{" +
                "isbn='" + isbn + '\'' +
                ", title='" + title + '\'' +
                ", author='" + author + '\'' +
                ", format=" + format +
                ", status=" + status +
                '}';
    }
}
```

### Member.java

```java
package library;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;

/**
 * Represents a registered library member.
 *
 * A member tracks:
 *  - their current set of borrowed book ISBNs
 *  - their accumulated fine balance
 *  - their membership tier (determines borrow limits)
 */
public class Member {

    private final String memberId;
    private String name;
    private MembershipTier tier;
    private final List<String> borrowedBooks;
    private double totalFines;

    /**
     * Constructs a new Member with zero fines and an empty borrow list.
     *
     * @param memberId unique identifier for this member
     * @param name     display name
     * @param tier     membership tier (FREE or PREMIUM)
     */
    public Member(String memberId, String name, MembershipTier tier) {
        if (memberId == null || memberId.isBlank()) {
            throw new IllegalArgumentException("Member ID must not be null or blank.");
        }
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Member name must not be null or blank.");
        }
        if (tier == null) {
            throw new IllegalArgumentException("MembershipTier must not be null.");
        }
        this.memberId = memberId;
        this.name = name;
        this.tier = tier;
        this.borrowedBooks = new ArrayList<>();
        this.totalFines = 0.0;
    }

    // -------------------------------------------------------------------------
    // Getters
    // -------------------------------------------------------------------------

    public String getMemberId() {
        return memberId;
    }

    public String getName() {
        return name;
    }

    public MembershipTier getTier() {
        return tier;
    }

    /**
     * Returns an unmodifiable view of currently borrowed book ISBNs.
     */
    public List<String> getBorrowedBooks() {
        return Collections.unmodifiableList(borrowedBooks);
    }

    public double getTotalFines() {
        return totalFines;
    }

    // -------------------------------------------------------------------------
    // Setters
    // -------------------------------------------------------------------------

    public void setName(String name) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name must not be null or blank.");
        }
        this.name = name;
    }

    public void setTier(MembershipTier tier) {
        if (tier == null) {
            throw new IllegalArgumentException("MembershipTier must not be null.");
        }
        this.tier = tier;
    }

    // -------------------------------------------------------------------------
    // Domain behaviour
    // -------------------------------------------------------------------------

    /**
     * Records that this member has borrowed a book.
     *
     * @param isbn the ISBN of the borrowed book
     */
    public void addBorrowedBook(String isbn) {
        if (isbn == null || isbn.isBlank()) {
            throw new IllegalArgumentException("ISBN must not be null or blank.");
        }
        if (!borrowedBooks.contains(isbn)) {
            borrowedBooks.add(isbn);
        }
    }

    /**
     * Removes the given ISBN from this member's active borrow list.
     * Called when the member returns a book.
     *
     * @param isbn the ISBN of the returned book
     * @return true if the book was found and removed, false otherwise
     */
    public boolean removeBorrowedBook(String isbn) {
        return borrowedBooks.remove(isbn);
    }

    /**
     * Adds a fine amount to the member's balance.
     *
     * @param amount the fine amount in dollars; must be non-negative
     */
    public void addFine(double amount) {
        if (amount < 0) {
            throw new IllegalArgumentException("Fine amount must be non-negative.");
        }
        this.totalFines += amount;
    }

    /**
     * Pays down the member's fine balance by the given amount.
     * If the payment exceeds the balance the balance is zeroed.
     *
     * @param amount amount to pay; must be non-negative
     * @return the actual amount deducted (may be less than amount if balance was lower)
     */
    public double payFine(double amount) {
        if (amount < 0) {
            throw new IllegalArgumentException("Payment amount must be non-negative.");
        }
        double deducted = Math.min(amount, totalFines);
        totalFines -= deducted;
        return deducted;
    }

    /**
     * Returns the number of books the member currently has borrowed.
     */
    public int getCurrentBorrowCount() {
        return borrowedBooks.size();
    }

    // -------------------------------------------------------------------------
    // Object overrides
    // -------------------------------------------------------------------------

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Member member = (Member) o;
        return Objects.equals(memberId, member.memberId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(memberId);
    }

    @Override
    public String toString() {
        return "Member{" +
                "memberId='" + memberId + '\'' +
                ", name='" + name + '\'' +
                ", tier=" + tier +
                ", borrowedBooks=" + borrowedBooks +
                ", totalFines=" + String.format("%.2f", totalFines) +
                '}';
    }
}
```

### BorrowingRecord.java

```java
package library;

import java.time.LocalDate;
import java.time.temporal.ChronoUnit;
import java.util.Objects;
import java.util.UUID;

/**
 * Immutable transactional record of a single borrow event.
 *
 * Once created only the return state can change (via markReturned).
 * All date comparisons use LocalDate.now() at call time — no caching —
 * so isOverdue() reflects the real-time state.
 */
public class BorrowingRecord {

    private final UUID recordId;
    private final String bookIsbn;
    private final String memberId;
    private final LocalDate borrowDate;
    private final LocalDate dueDate;
    private LocalDate returnDate;
    private boolean returned;

    /**
     * Creates a new open borrowing record.
     *
     * @param bookIsbn   ISBN of the borrowed book
     * @param memberId   ID of the borrowing member
     * @param borrowDate the date the book was borrowed
     * @param dueDate    the date by which the book must be returned
     */
    public BorrowingRecord(String bookIsbn, String memberId,
                           LocalDate borrowDate, LocalDate dueDate) {
        if (bookIsbn == null || bookIsbn.isBlank()) {
            throw new IllegalArgumentException("Book ISBN must not be null or blank.");
        }
        if (memberId == null || memberId.isBlank()) {
            throw new IllegalArgumentException("Member ID must not be null or blank.");
        }
        if (borrowDate == null) {
            throw new IllegalArgumentException("Borrow date must not be null.");
        }
        if (dueDate == null) {
            throw new IllegalArgumentException("Due date must not be null.");
        }
        if (dueDate.isBefore(borrowDate)) {
            throw new IllegalArgumentException("Due date must not be before borrow date.");
        }
        this.recordId = UUID.randomUUID();
        this.bookIsbn = bookIsbn;
        this.memberId = memberId;
        this.borrowDate = borrowDate;
        this.dueDate = dueDate;
        this.returnDate = null;
        this.returned = false;
    }

    // -------------------------------------------------------------------------
    // Getters
    // -------------------------------------------------------------------------

    public UUID getRecordId() {
        return recordId;
    }

    public String getBookIsbn() {
        return bookIsbn;
    }

    public String getMemberId() {
        return memberId;
    }

    public LocalDate getBorrowDate() {
        return borrowDate;
    }

    public LocalDate getDueDate() {
        return dueDate;
    }

    /**
     * Returns the return date, or null if the book has not yet been returned.
     */
    public LocalDate getReturnDate() {
        return returnDate;
    }

    public boolean isReturned() {
        return returned;
    }

    // -------------------------------------------------------------------------
    // Domain behaviour
    // -------------------------------------------------------------------------

    /**
     * Marks this record as returned on the given date.
     *
     * @param returnDate the actual return date
     * @throws IllegalStateException    if already returned
     * @throws IllegalArgumentException if returnDate is before borrowDate
     */
    public void markReturned(LocalDate returnDate) {
        if (returned) {
            throw new IllegalStateException("Book has already been returned.");
        }
        if (returnDate == null) {
            throw new IllegalArgumentException("Return date must not be null.");
        }
        if (returnDate.isBefore(borrowDate)) {
            throw new IllegalArgumentException("Return date must not be before borrow date.");
        }
        this.returnDate = returnDate;
        this.returned = true;
    }

    /**
     * Checks whether this borrowing is currently overdue.
     * If already returned, compares the actual return date to the due date.
     * If still open, compares today's date to the due date.
     *
     * @return true if the book is (or was returned) overdue
     */
    public boolean isOverdue() {
        LocalDate effectiveDate = returned ? returnDate : LocalDate.now();
        return effectiveDate.isAfter(dueDate);
    }

    /**
     * Returns the number of days overdue.
     * Returns 0 if not overdue.
     * Uses the return date if returned, otherwise today.
     *
     * @return non-negative count of overdue days
     */
    public long getDaysOverdue() {
        if (!isOverdue()) {
            return 0L;
        }
        LocalDate effectiveDate = returned ? returnDate : LocalDate.now();
        return ChronoUnit.DAYS.between(dueDate, effectiveDate);
    }

    // -------------------------------------------------------------------------
    // Object overrides
    // -------------------------------------------------------------------------

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        BorrowingRecord that = (BorrowingRecord) o;
        return Objects.equals(recordId, that.recordId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(recordId);
    }

    @Override
    public String toString() {
        return "BorrowingRecord{" +
                "recordId=" + recordId +
                ", bookIsbn='" + bookIsbn + '\'' +
                ", memberId='" + memberId + '\'' +
                ", borrowDate=" + borrowDate +
                ", dueDate=" + dueDate +
                ", returnDate=" + returnDate +
                ", returned=" + returned +
                ", overdue=" + isOverdue() +
                ", daysOverdue=" + getDaysOverdue() +
                '}';
    }
}
```

### FineCalculator.java

```java
package library;

/**
 * Strategy interface for computing fines on overdue borrowings.
 *
 * Each implementation encapsulates the fine rule for a particular book format.
 * The caller (LibraryService) selects the correct implementation via
 * FineCalculatorFactory and never contains format-specific branching logic.
 *
 * SRP  : each implementation has exactly one reason to change (its fine rule).
 * OCP  : new formats are supported by adding a new implementation + factory entry.
 * DIP  : LibraryService depends on this interface, not on concrete calculators.
 */
public interface FineCalculator {

    /**
     * Calculates the total fine for the given number of overdue days.
     *
     * @param daysOverdue number of days past the due date; guaranteed >= 0
     * @return fine amount in dollars; must be >= 0
     */
    double calculateFine(long daysOverdue);
}
```

### PhysicalBookFineCalculator.java

```java
package library;

/**
 * Fine calculator for physical books.
 *
 * Rule: $0.50 per day overdue.
 * Physical books occupy real shelf space and their absence inconveniences
 * other members, so a per-day penalty incentivises timely returns.
 */
public class PhysicalBookFineCalculator implements FineCalculator {

    /** Fine rate in dollars per overdue day. */
    private static final double FINE_RATE_PER_DAY = 0.50;

    /**
     * Calculates the fine as: daysOverdue * $0.50.
     *
     * @param daysOverdue number of days past the due date
     * @return total fine in dollars
     */
    @Override
    public double calculateFine(long daysOverdue) {
        if (daysOverdue <= 0) {
            return 0.0;
        }
        return daysOverdue * FINE_RATE_PER_DAY;
    }
}
```

### NoFineCalculator.java

```java
package library;

/**
 * Fine calculator for digital book formats (EBOOK, AUDIOBOOK).
 *
 * Rule: no fine is ever charged.
 * Digital licenses are soft-expired server-side, so there is no physical
 * item at risk. Returning "late" has no real cost to the library.
 *
 * This is the Null Object implementation of FineCalculator, eliminating
 * null checks in LibraryService.
 */
public class NoFineCalculator implements FineCalculator {

    /**
     * Always returns 0.0 regardless of overdue days.
     *
     * @param daysOverdue number of days past the due date (ignored)
     * @return always 0.0
     */
    @Override
    public double calculateFine(long daysOverdue) {
        return 0.0;
    }
}
```

### FineCalculatorFactory.java

```java
package library;

/**
 * Static factory that maps a BookFormat to the appropriate FineCalculator strategy.
 *
 * Adding a new format requires:
 *  1. Adding the constant to BookFormat enum.
 *  2. Creating a new FineCalculator implementation.
 *  3. Adding one case here.
 * No other class needs to change — OCP is preserved.
 */
public class FineCalculatorFactory {

    // Private constructor prevents instantiation.
    private FineCalculatorFactory() {
        throw new UnsupportedOperationException("FineCalculatorFactory is a utility class.");
    }

    /**
     * Returns the correct FineCalculator for the given book format.
     *
     * @param format the format of the book being returned
     * @return an appropriate FineCalculator; never null
     * @throws IllegalArgumentException if format is null or unrecognised
     */
    public static FineCalculator getCalculator(BookFormat format) {
        if (format == null) {
            throw new IllegalArgumentException("BookFormat must not be null.");
        }
        switch (format) {
            case PHYSICAL:
                return new PhysicalBookFineCalculator();
            case EBOOK:
                return new NoFineCalculator();
            case AUDIOBOOK:
                return new NoFineCalculator();
            default:
                throw new IllegalArgumentException("No FineCalculator defined for format: " + format);
        }
    }
}
```

### BorrowingPolicy.java

```java
package library;

/**
 * Strategy interface encapsulating the borrowing rules for a membership tier.
 *
 * Each tier implementation determines:
 *  - whether a given member is allowed to borrow another book right now
 *  - the maximum number of books the tier allows concurrently
 *  - how many days the loan period lasts
 *
 * LSP: both implementations are substitutable wherever BorrowingPolicy is used.
 */
public interface BorrowingPolicy {

    /**
     * Determines whether the given member is permitted to borrow one more book.
     *
     * @param member the member attempting to borrow
     * @return true if the member has not yet reached their borrow limit
     */
    boolean canBorrow(Member member);

    /**
     * Returns the maximum number of books this tier may borrow concurrently.
     *
     * @return borrow limit; Integer.MAX_VALUE means effectively unlimited
     */
    int getBorrowLimit();

    /**
     * Returns the number of days from the borrow date until the due date.
     *
     * @return loan duration in days
     */
    int getBorrowDurationDays();
}
```

### FreeMemberPolicy.java

```java
package library;

/**
 * Borrowing policy for FREE-tier members.
 *
 * Rules:
 *  - Maximum 2 books borrowed concurrently.
 *  - Loan duration: 14 days.
 */
public class FreeMemberPolicy implements BorrowingPolicy {

    private static final int BORROW_LIMIT = 2;
    private static final int BORROW_DURATION_DAYS = 14;

    /**
     * A FREE member can borrow if they currently hold fewer than 2 books.
     *
     * @param member the member attempting to borrow
     * @return true if the member's current borrow count is below the limit
     */
    @Override
    public boolean canBorrow(Member member) {
        if (member == null) {
            throw new IllegalArgumentException("Member must not be null.");
        }
        return member.getCurrentBorrowCount() < BORROW_LIMIT;
    }

    /**
     * @return 2
     */
    @Override
    public int getBorrowLimit() {
        return BORROW_LIMIT;
    }

    /**
     * @return 14
     */
    @Override
    public int getBorrowDurationDays() {
        return BORROW_DURATION_DAYS;
    }
}
```

### PremiumMemberPolicy.java

```java
package library;

/**
 * Borrowing policy for PREMIUM-tier members.
 *
 * Rules:
 *  - Unlimited concurrent borrows (Integer.MAX_VALUE as sentinel).
 *  - Loan duration: 21 days.
 */
public class PremiumMemberPolicy implements BorrowingPolicy {

    private static final int BORROW_LIMIT = Integer.MAX_VALUE;
    private static final int BORROW_DURATION_DAYS = 21;

    /**
     * A PREMIUM member can always borrow another book.
     *
     * @param member the member attempting to borrow (must not be null)
     * @return always true for PREMIUM members
     */
    @Override
    public boolean canBorrow(Member member) {
        if (member == null) {
            throw new IllegalArgumentException("Member must not be null.");
        }
        // PREMIUM members have no hard cap on concurrent borrows.
        return true;
    }

    /**
     * @return Integer.MAX_VALUE (unlimited)
     */
    @Override
    public int getBorrowLimit() {
        return BORROW_LIMIT;
    }

    /**
     * @return 21
     */
    @Override
    public int getBorrowDurationDays() {
        return BORROW_DURATION_DAYS;
    }
}
```

### BorrowingPolicyFactory.java

```java
package library;

/**
 * Static factory that maps a MembershipTier to the appropriate BorrowingPolicy.
 *
 * Follows the same OCP pattern as FineCalculatorFactory: adding a new tier
 * requires a new BorrowingPolicy implementation and one additional case here.
 */
public class BorrowingPolicyFactory {

    // Private constructor prevents instantiation.
    private BorrowingPolicyFactory() {
        throw new UnsupportedOperationException("BorrowingPolicyFactory is a utility class.");
    }

    /**
     * Returns the BorrowingPolicy for the given membership tier.
     *
     * @param tier the member's current subscription tier
     * @return an appropriate BorrowingPolicy; never null
     * @throws IllegalArgumentException if tier is null or unrecognised
     */
    public static BorrowingPolicy getPolicy(MembershipTier tier) {
        if (tier == null) {
            throw new IllegalArgumentException("MembershipTier must not be null.");
        }
        switch (tier) {
            case FREE:
                return new FreeMemberPolicy();
            case PREMIUM:
                return new PremiumMemberPolicy();
            default:
                throw new IllegalArgumentException("No BorrowingPolicy defined for tier: " + tier);
        }
    }
}
```

### WaitlistEntry.java

```java
package library;

import java.time.LocalDateTime;
import java.util.Objects;

/**
 * Represents a single member's position in the waitlist for a specific book.
 *
 * Entries are ordered by timestamp (FIFO): the member who joined earliest
 * is notified first when the book becomes available.
 *
 * This is a value-object: once created its fields never change.
 */
public class WaitlistEntry {

    private final String memberId;
    private final String bookIsbn;
    private final LocalDateTime timestamp;

    /**
     * Creates a new waitlist entry timestamped at the moment of construction.
     *
     * @param memberId  ID of the waiting member
     * @param bookIsbn  ISBN of the book being waited for
     */
    public WaitlistEntry(String memberId, String bookIsbn) {
        if (memberId == null || memberId.isBlank()) {
            throw new IllegalArgumentException("Member ID must not be null or blank.");
        }
        if (bookIsbn == null || bookIsbn.isBlank()) {
            throw new IllegalArgumentException("Book ISBN must not be null or blank.");
        }
        this.memberId = memberId;
        this.bookIsbn = bookIsbn;
        this.timestamp = LocalDateTime.now();
    }

    /**
     * Creates a waitlist entry with a specific timestamp (useful for testing).
     *
     * @param memberId  ID of the waiting member
     * @param bookIsbn  ISBN of the book being waited for
     * @param timestamp the enqueue time
     */
    public WaitlistEntry(String memberId, String bookIsbn, LocalDateTime timestamp) {
        if (memberId == null || memberId.isBlank()) {
            throw new IllegalArgumentException("Member ID must not be null or blank.");
        }
        if (bookIsbn == null || bookIsbn.isBlank()) {
            throw new IllegalArgumentException("Book ISBN must not be null or blank.");
        }
        if (timestamp == null) {
            throw new IllegalArgumentException("Timestamp must not be null.");
        }
        this.memberId = memberId;
        this.bookIsbn = bookIsbn;
        this.timestamp = timestamp;
    }

    public String getMemberId() {
        return memberId;
    }

    public String getBookIsbn() {
        return bookIsbn;
    }

    public LocalDateTime getTimestamp() {
        return timestamp;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        WaitlistEntry that = (WaitlistEntry) o;
        return Objects.equals(memberId, that.memberId) &&
                Objects.equals(bookIsbn, that.bookIsbn);
    }

    @Override
    public int hashCode() {
        return Objects.hash(memberId, bookIsbn);
    }

    @Override
    public String toString() {
        return "WaitlistEntry{" +
                "memberId='" + memberId + '\'' +
                ", bookIsbn='" + bookIsbn + '\'' +
                ", timestamp=" + timestamp +
                '}';
    }
}
```

### WaitlistObserver.java

```java
package library;

/**
 * Observer interface for waitlist availability events.
 *
 * Implementations receive a notification whenever the next member in a
 * book's waitlist should be told the book is now available.
 *
 * New notification channels (email, SMS, push) are added by implementing
 * this interface and registering with WaitlistManager — zero changes to
 * existing code (OCP).
 */
public interface WaitlistObserver {

    /**
     * Called when a book becomes available and the given member is next in line.
     *
     * @param memberId  ID of the member who should be notified
     * @param bookIsbn  ISBN of the book that is now available
     */
    void onBookAvailable(String memberId, String bookIsbn);
}
```

### MemberNotificationObserver.java

```java
package library;

/**
 * Concrete observer that notifies a member via console output.
 *
 * In a production system this would be replaced by (or supplemented with)
 * an email/SMS/push-notification implementation — all without touching
 * WaitlistManager or LibraryService.
 */
public class MemberNotificationObserver implements WaitlistObserver {

    /**
     * Prints a notification message to standard output.
     *
     * @param memberId  ID of the member being notified
     * @param bookIsbn  ISBN of the now-available book
     */
    @Override
    public void onBookAvailable(String memberId, String bookIsbn) {
        System.out.println("[NOTIFICATION] Member " + memberId +
                ": The book with ISBN " + bookIsbn +
                " is now available for borrowing.");
    }
}
```

### WaitlistManager.java

```java
package library;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.LinkedList;
import java.util.List;
import java.util.Map;
import java.util.Queue;

/**
 * Manages per-book waitlists and fires WaitlistObserver notifications.
 *
 * Internal structure:
 *  - waitlists: Map<bookIsbn, Queue<WaitlistEntry>>
 *    FIFO queue; the entry at the head is the next to be notified.
 *  - observers: List<WaitlistObserver>
 *    All registered observers are notified on every availability event.
 *
 * This class is the Subject in the Observer pattern.
 */
public class WaitlistManager {

    private final Map<String, Queue<WaitlistEntry>> waitlists;
    private final List<WaitlistObserver> observers;

    /**
     * Constructs a WaitlistManager with no waitlists or observers.
     */
    public WaitlistManager() {
        this.waitlists = new HashMap<>();
        this.observers = new ArrayList<>();
    }

    // -------------------------------------------------------------------------
    // Observer registration
    // -------------------------------------------------------------------------

    /**
     * Registers an observer to receive availability notifications.
     *
     * @param observer the observer to add; must not be null
     */
    public void addObserver(WaitlistObserver observer) {
        if (observer == null) {
            throw new IllegalArgumentException("Observer must not be null.");
        }
        if (!observers.contains(observer)) {
            observers.add(observer);
        }
    }

    /**
     * Removes a previously registered observer.
     *
     * @param observer the observer to remove
     * @return true if the observer was found and removed
     */
    public boolean removeObserver(WaitlistObserver observer) {
        return observers.remove(observer);
    }

    // -------------------------------------------------------------------------
    // Waitlist operations
    // -------------------------------------------------------------------------

    /**
     * Adds a member to the waitlist for the specified book.
     * If the member is already on the waitlist for this book, the call is ignored.
     *
     * @param memberId the ID of the member joining the waitlist
     * @param bookIsbn the ISBN of the book being waited for
     */
    public void addToWaitlist(String memberId, String bookIsbn) {
        if (memberId == null || memberId.isBlank()) {
            throw new IllegalArgumentException("Member ID must not be null or blank.");
        }
        if (bookIsbn == null || bookIsbn.isBlank()) {
            throw new IllegalArgumentException("Book ISBN must not be null or blank.");
        }

        waitlists.putIfAbsent(bookIsbn, new LinkedList<>());
        Queue<WaitlistEntry> queue = waitlists.get(bookIsbn);

        // Prevent duplicate entries for the same member-book pair.
        boolean alreadyWaiting = queue.stream()
                .anyMatch(entry -> entry.getMemberId().equals(memberId));
        if (!alreadyWaiting) {
            queue.add(new WaitlistEntry(memberId, bookIsbn));
        }
    }

    /**
     * Removes a member from the waitlist for the specified book.
     * Useful when a member cancels their reservation request.
     *
     * @param memberId the ID of the member to remove
     * @param bookIsbn the ISBN of the book
     * @return true if the member was found and removed from the waitlist
     */
    public boolean removeFromWaitlist(String memberId, String bookIsbn) {
        if (memberId == null || memberId.isBlank()) {
            throw new IllegalArgumentException("Member ID must not be null or blank.");
        }
        if (bookIsbn == null || bookIsbn.isBlank()) {
            throw new IllegalArgumentException("Book ISBN must not be null or blank.");
        }

        Queue<WaitlistEntry> queue = waitlists.get(bookIsbn);
        if (queue == null || queue.isEmpty()) {
            return false;
        }
        return queue.removeIf(entry -> entry.getMemberId().equals(memberId));
    }

    /**
     * Dequeues the next waiting member for the given book and notifies all
     * registered observers.
     *
     * Should be called by LibraryService.returnBook() after the book status
     * is reset to AVAILABLE.
     *
     * @param bookIsbn the ISBN of the book that has become available
     */
    public void notifyNext(String bookIsbn) {
        if (bookIsbn == null || bookIsbn.isBlank()) {
            throw new IllegalArgumentException("Book ISBN must not be null or blank.");
        }

        Queue<WaitlistEntry> queue = waitlists.get(bookIsbn);
        if (queue == null || queue.isEmpty()) {
            return; // No one is waiting; nothing to do.
        }

        WaitlistEntry next = queue.poll();
        for (WaitlistObserver observer : observers) {
            observer.onBookAvailable(next.getMemberId(), next.getBookIsbn());
        }
    }

    /**
     * Returns true if at least one member is waiting for the given book.
     *
     * @param bookIsbn the ISBN to check
     * @return true if the waitlist is non-empty
     */
    public boolean hasWaiters(String bookIsbn) {
        if (bookIsbn == null || bookIsbn.isBlank()) {
            return false;
        }
        Queue<WaitlistEntry> queue = waitlists.get(bookIsbn);
        return queue != null && !queue.isEmpty();
    }

    /**
     * Returns the number of members waiting for the given book.
     *
     * @param bookIsbn the ISBN to check
     * @return waitlist size; 0 if no waitlist exists for this book
     */
    public int getQueueSize(String bookIsbn) {
        if (bookIsbn == null || bookIsbn.isBlank()) {
            return 0;
        }
        Queue<WaitlistEntry> queue = waitlists.get(bookIsbn);
        return queue == null ? 0 : queue.size();
    }

    /**
     * Returns the memberId of the member at the front of the waitlist
     * without removing them, or null if the waitlist is empty.
     *
     * @param bookIsbn the ISBN to check
     * @return memberId of the next waiter, or null
     */
    public String peekNext(String bookIsbn) {
        if (bookIsbn == null || bookIsbn.isBlank()) {
            return null;
        }
        Queue<WaitlistEntry> queue = waitlists.get(bookIsbn);
        if (queue == null || queue.isEmpty()) {
            return null;
        }
        return queue.peek().getMemberId();
    }

    /**
     * Clears all waitlist entries for a given book.
     * Called when a book is permanently removed from the catalog.
     *
     * @param bookIsbn the ISBN of the book whose waitlist should be cleared
     */
    public void clearWaitlist(String bookIsbn) {
        if (bookIsbn == null || bookIsbn.isBlank()) {
            return;
        }
        waitlists.remove(bookIsbn);
    }
}
```

### RecommendationEngine.java

```java
package library;

import java.util.List;

/**
 * Strategy interface for book recommendation algorithms.
 *
 * Implementations use different signals (same author, genre, popularity,
 * collaborative filtering, etc.) to suggest books to a member.
 *
 * The interface depends only on the stable domain types Member and Book,
 * ensuring implementations are portable and independently testable.
 */
public interface RecommendationEngine {

    /**
     * Returns a list of recommended books for the given member.
     *
     * @param member  the member requesting recommendations; must not be null
     * @param catalog the book catalog to draw recommendations from; must not be null
     * @param limit   maximum number of recommendations to return; must be > 0
     * @return list of recommended books, possibly empty, never null;
     *         size is at most {@code limit}
     */
    List<Book> recommend(Member member, BookCatalog catalog, int limit);
}
```

### SimpleCatalogRecommendationEngine.java

```java
package library;

import java.util.ArrayList;
import java.util.List;

/**
 * Simple recommendation engine that suggests books by the same author as
 * any book currently borrowed by the member.
 *
 * Algorithm:
 *  1. Collect the authors of all books the member currently has borrowed.
 *  2. Find AVAILABLE books in the catalog written by those authors
 *     that the member does not already have borrowed.
 *  3. If fewer than {@code limit} results found, fill remaining slots with
 *     any other AVAILABLE books the member does not have.
 *  4. Return at most {@code limit} recommendations.
 *
 * This is intentionally simple. A production engine would use user history,
 * ratings, genre tags, collaborative filtering, etc.
 */
public class SimpleCatalogRecommendationEngine implements RecommendationEngine {

    /**
     * Recommends books using same-author affinity with available-book fallback.
     *
     * @param member  the member requesting recommendations
     * @param catalog the book catalog
     * @param limit   maximum number of results
     * @return up to {@code limit} recommended books
     */
    @Override
    public List<Book> recommend(Member member, BookCatalog catalog, int limit) {
        if (member == null) {
            throw new IllegalArgumentException("Member must not be null.");
        }
        if (catalog == null) {
            throw new IllegalArgumentException("Catalog must not be null.");
        }
        if (limit <= 0) {
            throw new IllegalArgumentException("Limit must be positive.");
        }

        List<String> borrowedIsbns = member.getBorrowedBooks();
        List<Book> recommendations = new ArrayList<>();

        // Step 1: collect authors of currently borrowed books.
        List<String> borrowedAuthors = new ArrayList<>();
        for (String isbn : borrowedIsbns) {
            Book borrowedBook = catalog.getBook(isbn);
            if (borrowedBook != null) {
                String author = borrowedBook.getAuthor();
                if (!borrowedAuthors.contains(author)) {
                    borrowedAuthors.add(author);
                }
            }
        }

        // Step 2: find available books by the same authors.
        if (!borrowedAuthors.isEmpty()) {
            for (String author : borrowedAuthors) {
                List<Book> byAuthor = catalog.searchByAuthor(author);
                for (Book book : byAuthor) {
                    if (book.getStatus() == BookStatus.AVAILABLE
                            && !borrowedIsbns.contains(book.getIsbn())
                            && !recommendations.contains(book)) {
                        recommendations.add(book);
                        if (recommendations.size() >= limit) {
                            return recommendations;
                        }
                    }
                }
            }
        }

        // Step 3: fall back to any available book not already borrowed.
        if (recommendations.size() < limit) {
            List<Book> allAvailable = catalog.getAllAvailable();
            for (Book book : allAvailable) {
                if (!borrowedIsbns.contains(book.getIsbn())
                        && !recommendations.contains(book)) {
                    recommendations.add(book);
                    if (recommendations.size() >= limit) {
                        break;
                    }
                }
            }
        }

        return recommendations;
    }
}
```

### BookCatalog.java

```java
package library;

import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * In-memory catalog of all books managed by the library.
 *
 * Primary key: ISBN (String).
 * All retrieval methods return defensive copies or unmodifiable views
 * to protect internal state.
 */
public class BookCatalog {

    private final Map<String, Book> books;

    /**
     * Constructs an empty BookCatalog.
     */
    public BookCatalog() {
        this.books = new HashMap<>();
    }

    // -------------------------------------------------------------------------
    // Mutation
    // -------------------------------------------------------------------------

    /**
     * Adds a book to the catalog.
     *
     * @param book the book to add; must not be null
     * @throws IllegalArgumentException if book is null or ISBN already exists
     */
    public void addBook(Book book) {
        if (book == null) {
            throw new IllegalArgumentException("Book must not be null.");
        }
        if (books.containsKey(book.getIsbn())) {
            throw new IllegalArgumentException(
                    "A book with ISBN " + book.getIsbn() + " already exists in the catalog.");
        }
        books.put(book.getIsbn(), book);
    }

    /**
     * Removes a book from the catalog by ISBN.
     *
     * @param isbn the ISBN of the book to remove
     * @return the removed Book, or null if not found
     */
    public Book removeBook(String isbn) {
        if (isbn == null || isbn.isBlank()) {
            throw new IllegalArgumentException("ISBN must not be null or blank.");
        }
        return books.remove(isbn);
    }

    // -------------------------------------------------------------------------
    // Retrieval
    // -------------------------------------------------------------------------

    /**
     * Retrieves a book by ISBN.
     *
     * @param isbn the ISBN to look up
     * @return the Book, or null if not found
     */
    public Book getBook(String isbn) {
        if (isbn == null || isbn.isBlank()) {
            return null;
        }
        return books.get(isbn);
    }

    /**
     * Returns all books whose title contains the given substring (case-insensitive).
     *
     * @param titleSubstring the substring to search for
     * @return list of matching books; never null
     */
    public List<Book> searchByTitle(String titleSubstring) {
        if (titleSubstring == null || titleSubstring.isBlank()) {
            return new ArrayList<>(books.values());
        }
        String lowerQuery = titleSubstring.toLowerCase();
        List<Book> results = new ArrayList<>();
        for (Book book : books.values()) {
            if (book.getTitle().toLowerCase().contains(lowerQuery)) {
                results.add(book);
            }
        }
        return results;
    }

    /**
     * Returns all books whose author name contains the given substring (case-insensitive).
     *
     * @param authorSubstring the substring to search for
     * @return list of matching books; never null
     */
    public List<Book> searchByAuthor(String authorSubstring) {
        if (authorSubstring == null || authorSubstring.isBlank()) {
            return new ArrayList<>(books.values());
        }
        String lowerQuery = authorSubstring.toLowerCase();
        List<Book> results = new ArrayList<>();
        for (Book book : books.values()) {
            if (book.getAuthor().toLowerCase().contains(lowerQuery)) {
                results.add(book);
            }
        }
        return results;
    }

    /**
     * Returns all books of the specified format.
     *
     * @param format the format to filter by; must not be null
     * @return list of matching books; never null
     */
    public List<Book> searchByFormat(BookFormat format) {
        if (format == null) {
            throw new IllegalArgumentException("BookFormat must not be null.");
        }
        List<Book> results = new ArrayList<>();
        for (Book book : books.values()) {
            if (book.getFormat() == format) {
                results.add(book);
            }
        }
        return results;
    }

    /**
     * Returns all books currently in AVAILABLE status.
     *
     * @return list of available books; never null
     */
    public List<Book> getAllAvailable() {
        List<Book> results = new ArrayList<>();
        for (Book book : books.values()) {
            if (book.getStatus() == BookStatus.AVAILABLE) {
                results.add(book);
            }
        }
        return results;
    }

    /**
     * Returns all books in the catalog regardless of status.
     *
     * @return unmodifiable view of all books; never null
     */
    public List<Book> getAllBooks() {
        return Collections.unmodifiableList(new ArrayList<>(books.values()));
    }

    /**
     * Returns the total number of books in the catalog.
     *
     * @return catalog size
     */
    public int size() {
        return books.size();
    }

    /**
     * Returns true if the catalog contains a book with the given ISBN.
     *
     * @param isbn the ISBN to check
     * @return true if present
     */
    public boolean contains(String isbn) {
        if (isbn == null || isbn.isBlank()) {
            return false;
        }
        return books.containsKey(isbn);
    }
}
```

### LibraryService.java

```java
package library;

import java.time.LocalDate;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * Core application service orchestrating all library operations.
 *
 * Responsibilities:
 *  - Member registration and retrieval.
 *  - Borrowing: policy enforcement, record creation, status transitions.
 *  - Returning: fine calculation, fine application, waitlist notification.
 *  - Reservations: status transitions for RESERVED books.
 *  - Fine payment: delegate to Member domain object.
 *  - Query: active borrowings, overdue records.
 *
 * Dependencies are injected via the constructor (DIP). LibraryService
 * references only interfaces (BorrowingPolicy via factory, FineCalculator
 * via factory, RecommendationEngine, WaitlistManager).
 */
public class LibraryService {

    private final BookCatalog catalog;
    private final Map<String, Member> members;
    private final List<BorrowingRecord> borrowings;
    private final WaitlistManager waitlistManager;
    private final RecommendationEngine recommendationEngine;

    /**
     * Constructs a LibraryService with all required collaborators.
     *
     * @param catalog               the book catalog
     * @param waitlistManager       the waitlist manager (observer subject)
     * @param recommendationEngine  the recommendation strategy
     */
    public LibraryService(BookCatalog catalog,
                          WaitlistManager waitlistManager,
                          RecommendationEngine recommendationEngine) {
        if (catalog == null) {
            throw new IllegalArgumentException("BookCatalog must not be null.");
        }
        if (waitlistManager == null) {
            throw new IllegalArgumentException("WaitlistManager must not be null.");
        }
        if (recommendationEngine == null) {
            throw new IllegalArgumentException("RecommendationEngine must not be null.");
        }
        this.catalog = catalog;
        this.members = new HashMap<>();
        this.borrowings = new ArrayList<>();
        this.waitlistManager = waitlistManager;
        this.recommendationEngine = recommendationEngine;
    }

    // -------------------------------------------------------------------------
    // Member management
    // -------------------------------------------------------------------------

    /**
     * Registers a new member with the library.
     *
     * @param member the member to register; must not be null
     * @throws IllegalArgumentException if a member with the same ID already exists
     */
    public void registerMember(Member member) {
        if (member == null) {
            throw new IllegalArgumentException("Member must not be null.");
        }
        if (members.containsKey(member.getMemberId())) {
            throw new IllegalArgumentException(
                    "A member with ID " + member.getMemberId() + " is already registered.");
        }
        members.put(member.getMemberId(), member);
    }

    /**
     * Retrieves a registered member by ID.
     *
     * @param memberId the member's unique identifier
     * @return the Member, or null if not found
     */
    public Member getMember(String memberId) {
        if (memberId == null || memberId.isBlank()) {
            return null;
        }
        return members.get(memberId);
    }

    // -------------------------------------------------------------------------
    // Borrowing
    // -------------------------------------------------------------------------

    /**
     * Borrows a book on behalf of a member.
     *
     * Pre-conditions:
     *  - Member exists and is registered.
     *  - Book exists in the catalog.
     *  - Book status is AVAILABLE.
     *  - Member's tier policy allows one more borrow.
     *
     * Post-conditions:
     *  - Book status is set to BORROWED.
     *  - Member's borrowedBooks list includes the ISBN.
     *  - A BorrowingRecord is created and stored.
     *
     * @param memberId the borrowing member's ID
     * @param bookIsbn the ISBN of the book to borrow
     * @return the created BorrowingRecord
     * @throws IllegalArgumentException if member or book not found
     * @throws IllegalStateException    if the book is not available or policy blocks the borrow
     */
    public BorrowingRecord borrowBook(String memberId, String bookIsbn) {
        Member member = requireMember(memberId);
        Book book = requireBook(bookIsbn);

        if (book.getStatus() != BookStatus.AVAILABLE) {
            throw new IllegalStateException(
                    "Book " + bookIsbn + " is not available. Current status: " + book.getStatus());
        }

        BorrowingPolicy policy = BorrowingPolicyFactory.getPolicy(member.getTier());
        if (!policy.canBorrow(member)) {
            throw new IllegalStateException(
                    "Member " + memberId + " has reached their borrow limit of "
                            + policy.getBorrowLimit() + " books.");
        }

        LocalDate borrowDate = LocalDate.now();
        LocalDate dueDate = borrowDate.plusDays(policy.getBorrowDurationDays());

        BorrowingRecord record = new BorrowingRecord(bookIsbn, memberId, borrowDate, dueDate);
        borrowings.add(record);

        book.setStatus(BookStatus.BORROWED);
        member.addBorrowedBook(bookIsbn);

        System.out.println("[BORROW] Member " + member.getName() + " borrowed '"
                + book.getTitle() + "'. Due: " + dueDate);

        return record;
    }

    /**
     * Returns a borrowed book and applies any applicable fine.
     *
     * Pre-conditions:
     *  - A non-returned BorrowingRecord exists for this member-book pair.
     *
     * Post-conditions:
     *  - The BorrowingRecord is marked returned with today's date.
     *  - Book status is reset to AVAILABLE.
     *  - Member's borrowedBooks no longer contains the ISBN.
     *  - If overdue and format is PHYSICAL, the fine is added to member's balance.
     *  - If waiters exist, waitlistManager.notifyNext() is called.
     *
     * @param memberId the returning member's ID
     * @param bookIsbn the ISBN of the book being returned
     * @return the fine amount charged (0.0 if no fine applies)
     * @throws IllegalArgumentException if member or book not found
     * @throws IllegalStateException    if no active borrowing record is found
     */
    public double returnBook(String memberId, String bookIsbn) {
        Member member = requireMember(memberId);
        Book book = requireBook(bookIsbn);

        BorrowingRecord record = findActiveBorrowing(memberId, bookIsbn);
        if (record == null) {
            throw new IllegalStateException(
                    "No active borrowing record found for member " + memberId
                            + " and book " + bookIsbn);
        }

        LocalDate returnDate = LocalDate.now();
        record.markReturned(returnDate);

        book.setStatus(BookStatus.AVAILABLE);
        member.removeBorrowedBook(bookIsbn);

        // Calculate and apply fine if overdue.
        double fine = 0.0;
        if (record.isOverdue()) {
            FineCalculator calculator = FineCalculatorFactory.getCalculator(book.getFormat());
            fine = calculator.calculateFine(record.getDaysOverdue());
            if (fine > 0.0) {
                member.addFine(fine);
                System.out.println("[FINE] Member " + member.getName()
                        + " charged $" + String.format("%.2f", fine)
                        + " for returning '" + book.getTitle()
                        + "' " + record.getDaysOverdue() + " day(s) late.");
            }
        }

        System.out.println("[RETURN] Member " + member.getName()
                + " returned '" + book.getTitle() + "' on " + returnDate + ".");

        // Notify the next person in the waitlist if any.
        if (waitlistManager.hasWaiters(bookIsbn)) {
            waitlistManager.notifyNext(bookIsbn);
        }

        return fine;
    }

    // -------------------------------------------------------------------------
    // Reservations
    // -------------------------------------------------------------------------

    /**
     * Reserves a book that is currently AVAILABLE for a member.
     * This is distinct from the waitlist — it transitions the book to RESERVED.
     *
     * @param memberId the member making the reservation
     * @param bookIsbn the ISBN of the book to reserve
     * @throws IllegalArgumentException if member or book not found
     * @throws IllegalStateException    if the book is not currently AVAILABLE
     */
    public void reserveBook(String memberId, String bookIsbn) {
        Member member = requireMember(memberId);
        Book book = requireBook(bookIsbn);

        if (book.getStatus() != BookStatus.AVAILABLE) {
            throw new IllegalStateException(
                    "Book " + bookIsbn + " cannot be reserved. Current status: " + book.getStatus());
        }

        book.setStatus(BookStatus.RESERVED);
        System.out.println("[RESERVE] Member " + member.getName()
                + " reserved '" + book.getTitle() + "'.");
    }

    /**
     * Cancels an existing reservation, returning the book to AVAILABLE.
     *
     * @param memberId the member cancelling the reservation
     * @param bookIsbn the ISBN of the reserved book
     * @throws IllegalArgumentException if member or book not found
     * @throws IllegalStateException    if the book is not in RESERVED status
     */
    public void cancelReservation(String memberId, String bookIsbn) {
        Member member = requireMember(memberId);
        Book book = requireBook(bookIsbn);

        if (book.getStatus() != BookStatus.RESERVED) {
            throw new IllegalStateException(
                    "Book " + bookIsbn + " is not currently reserved. Status: " + book.getStatus());
        }

        book.setStatus(BookStatus.AVAILABLE);
        System.out.println("[CANCEL RESERVE] Member " + member.getName()
                + " cancelled reservation for '" + book.getTitle() + "'.");

        // Notify waitlist now that the book is available again.
        if (waitlistManager.hasWaiters(bookIsbn)) {
            waitlistManager.notifyNext(bookIsbn);
        }
    }

    // -------------------------------------------------------------------------
    // Fine payment
    // -------------------------------------------------------------------------

    /**
     * Processes a fine payment for a member.
     *
     * @param memberId the ID of the paying member
     * @param amount   the amount to pay; must be non-negative
     * @return the amount actually deducted from the member's fine balance
     * @throws IllegalArgumentException if member not found
     */
    public double payFine(String memberId, double amount) {
        Member member = requireMember(memberId);
        double deducted = member.payFine(amount);
        System.out.println("[FINE PAYMENT] Member " + member.getName()
                + " paid $" + String.format("%.2f", deducted)
                + ". Remaining balance: $" + String.format("%.2f", member.getTotalFines()));
        return deducted;
    }

    // -------------------------------------------------------------------------
    // Queries
    // -------------------------------------------------------------------------

    /**
     * Returns all BorrowingRecords that have not yet been returned.
     *
     * @return list of active (open) borrowings; never null
     */
    public List<BorrowingRecord> getActiveBorrowings() {
        List<BorrowingRecord> active = new ArrayList<>();
        for (BorrowingRecord record : borrowings) {
            if (!record.isReturned()) {
                active.add(record);
            }
        }
        return active;
    }

    /**
     * Returns all BorrowingRecords that are currently overdue (not yet returned
     * and past their due date).
     *
     * @return list of overdue records; never null
     */
    public List<BorrowingRecord> getOverdueRecords() {
        List<BorrowingRecord> overdue = new ArrayList<>();
        for (BorrowingRecord record : borrowings) {
            if (!record.isReturned() && record.isOverdue()) {
                overdue.add(record);
            }
        }
        return overdue;
    }

    /**
     * Returns all borrowing history (returned and active) for a given member.
     *
     * @param memberId the member's ID
     * @return list of all borrowing records for the member; never null
     */
    public List<BorrowingRecord> getBorrowingHistory(String memberId) {
        List<BorrowingRecord> history = new ArrayList<>();
        for (BorrowingRecord record : borrowings) {
            if (record.getMemberId().equals(memberId)) {
                history.add(record);
            }
        }
        return history;
    }

    /**
     * Returns recommendations for the given member.
     *
     * @param memberId the member's ID
     * @param limit    maximum number of recommendations
     * @return list of recommended books
     */
    public List<Book> getRecommendations(String memberId, int limit) {
        Member member = requireMember(memberId);
        return recommendationEngine.recommend(member, catalog, limit);
    }

    // -------------------------------------------------------------------------
    // Private helpers
    // -------------------------------------------------------------------------

    private Member requireMember(String memberId) {
        if (memberId == null || memberId.isBlank()) {
            throw new IllegalArgumentException("Member ID must not be null or blank.");
        }
        Member member = members.get(memberId);
        if (member == null) {
            throw new IllegalArgumentException("Member not found: " + memberId);
        }
        return member;
    }

    private Book requireBook(String bookIsbn) {
        if (bookIsbn == null || bookIsbn.isBlank()) {
            throw new IllegalArgumentException("Book ISBN must not be null or blank.");
        }
        Book book = catalog.getBook(bookIsbn);
        if (book == null) {
            throw new IllegalArgumentException("Book not found in catalog: " + bookIsbn);
        }
        return book;
    }

    private BorrowingRecord findActiveBorrowing(String memberId, String bookIsbn) {
        for (BorrowingRecord record : borrowings) {
            if (!record.isReturned()
                    && record.getMemberId().equals(memberId)
                    && record.getBookIsbn().equals(bookIsbn)) {
                return record;
            }
        }
        return null;
    }
}
```

### LibraryAdmin.java

```java
package library;

import java.util.List;

/**
 * Administrative facade for library management operations.
 *
 * Exposes a restricted set of operations available only to library staff:
 *  - Adding new books to the catalog.
 *  - Removing books from the catalog.
 *  - Generating overdue borrowing reports.
 *
 * This class follows SRP: it exists solely for administrative concerns
 * and delegates all domain logic to LibraryService and BookCatalog.
 */
public class LibraryAdmin {

    private final LibraryService service;
    private final BookCatalog catalog;
    private final WaitlistManager waitlistManager;

    /**
     * Constructs a LibraryAdmin with the shared service and catalog.
     *
     * @param service         the library's core service
     * @param catalog         the library's book catalog
     * @param waitlistManager the waitlist manager (needed to clean up on removal)
     */
    public LibraryAdmin(LibraryService service,
                        BookCatalog catalog,
                        WaitlistManager waitlistManager) {
        if (service == null) {
            throw new IllegalArgumentException("LibraryService must not be null.");
        }
        if (catalog == null) {
            throw new IllegalArgumentException("BookCatalog must not be null.");
        }
        if (waitlistManager == null) {
            throw new IllegalArgumentException("WaitlistManager must not be null.");
        }
        this.service = service;
        this.catalog = catalog;
        this.waitlistManager = waitlistManager;
    }

    /**
     * Adds a new book to the library catalog.
     *
     * @param book the book to add; must not be null and its ISBN must be unique
     */
    public void addBook(Book book) {
        if (book == null) {
            throw new IllegalArgumentException("Book must not be null.");
        }
        catalog.addBook(book);
        System.out.println("[ADMIN] Added book: " + book.getTitle()
                + " (ISBN: " + book.getIsbn() + ", Format: " + book.getFormat() + ")");
    }

    /**
     * Removes a book from the library catalog by ISBN.
     *
     * If the book is currently borrowed or reserved, the removal still proceeds —
     * callers should verify status before calling this in production.
     * The waitlist for the book is also cleared.
     *
     * @param isbn the ISBN of the book to remove
     * @return the removed Book, or null if no book with that ISBN existed
     */
    public Book removeBook(String isbn) {
        if (isbn == null || isbn.isBlank()) {
            throw new IllegalArgumentException("ISBN must not be null or blank.");
        }
        Book removed = catalog.removeBook(isbn);
        if (removed != null) {
            waitlistManager.clearWaitlist(isbn);
            System.out.println("[ADMIN] Removed book: " + removed.getTitle()
                    + " (ISBN: " + isbn + ")");
        } else {
            System.out.println("[ADMIN] No book found with ISBN: " + isbn);
        }
        return removed;
    }

    /**
     * Prints a formatted overdue report to standard output.
     *
     * The report lists every currently borrowed book that is past its due date,
     * including the member ID, book ISBN, due date, and days overdue.
     */
    public void generateOverdueReport() {
        List<BorrowingRecord> overdueRecords = service.getOverdueRecords();

        System.out.println("========================================");
        System.out.println("         OVERDUE BOOKS REPORT           ");
        System.out.println("========================================");

        if (overdueRecords.isEmpty()) {
            System.out.println("  No overdue books at this time.");
        } else {
            System.out.printf("  %-12s %-15s %-12s %-10s%n",
                    "MemberId", "Book ISBN", "Due Date", "Days Late");
            System.out.println("  --------------------------------------------------");
            for (BorrowingRecord record : overdueRecords) {
                System.out.printf("  %-12s %-15s %-12s %-10d%n",
                        record.getMemberId(),
                        record.getBookIsbn(),
                        record.getDueDate().toString(),
                        record.getDaysOverdue());
            }
        }
        System.out.println("========================================");
        System.out.println("  Total overdue: " + overdueRecords.size());
        System.out.println("========================================");
    }
}
```

### LibraryDemo.java

```java
package library;

import java.time.LocalDate;

/**
 * Runnable demonstration of the Online Library Platform.
 *
 * Scenario:
 *  1. Admin sets up the catalog with 6 books (ebook, audiobook, physical).
 *  2. Two members are registered: Alice (FREE) and Bob (PREMIUM).
 *  3. Alice borrows up to her FREE limit and is blocked on a third borrow.
 *  4. Bob borrows multiple books with no restriction.
 *  5. A physical book is simulated as overdue; fine is applied on return.
 *  6. A book is fully borrowed; Charlie joins the waitlist; on return,
 *     Charlie is notified.
 *  7. Recommendations are generated for Alice.
 *  8. Admin generates the overdue report.
 *  9. Admin removes a book and verifies catalog size.
 */
public class LibraryDemo {

    public static void main(String[] args) {

        System.out.println("=================================================");
        System.out.println("       ONLINE LIBRARY PLATFORM — DEMO            ");
        System.out.println("=================================================");
        System.out.println();

        // ---------------------------------------------------------------
        // 1. Infrastructure setup
        // ---------------------------------------------------------------
        BookCatalog catalog = new BookCatalog();
        WaitlistManager waitlistManager = new WaitlistManager();
        RecommendationEngine recommendationEngine = new SimpleCatalogRecommendationEngine();

        // Register the console notification observer.
        waitlistManager.addObserver(new MemberNotificationObserver());

        LibraryService service = new LibraryService(catalog, waitlistManager, recommendationEngine);
        LibraryAdmin admin = new LibraryAdmin(service, catalog, waitlistManager);

        // ---------------------------------------------------------------
        // 2. Admin adds books to the catalog
        // ---------------------------------------------------------------
        System.out.println("--- Admin: Populating Catalog ---");
        Book cleanCode = new Book("ISBN-001", "Clean Code", "Robert C. Martin", BookFormat.PHYSICAL);
        Book cleanArch = new Book("ISBN-002", "Clean Architecture", "Robert C. Martin", BookFormat.PHYSICAL);
        Book dddEbook = new Book("ISBN-003", "Domain-Driven Design", "Eric Evans", BookFormat.EBOOK);
        Book dddAudio = new Book("ISBN-004", "Domain-Driven Design (Audio)", "Eric Evans", BookFormat.AUDIOBOOK);
        Book effectiveJava = new Book("ISBN-005", "Effective Java", "Joshua Bloch", BookFormat.EBOOK);
        Book designPatterns = new Book("ISBN-006", "Design Patterns", "Gang of Four", BookFormat.PHYSICAL);

        admin.addBook(cleanCode);
        admin.addBook(cleanArch);
        admin.addBook(dddEbook);
        admin.addBook(dddAudio);
        admin.addBook(effectiveJava);
        admin.addBook(designPatterns);

        System.out.println("Catalog size: " + catalog.size());
        System.out.println();

        // ---------------------------------------------------------------
        // 3. Register members
        // ---------------------------------------------------------------
        System.out.println("--- Registering Members ---");
        Member alice = new Member("M001", "Alice", MembershipTier.FREE);
        Member bob = new Member("M002", "Bob", MembershipTier.PREMIUM);
        Member charlie = new Member("M003", "Charlie", MembershipTier.FREE);

        service.registerMember(alice);
        service.registerMember(bob);
        service.registerMember(charlie);

        System.out.println("Registered: " + alice);
        System.out.println("Registered: " + bob);
        System.out.println("Registered: " + charlie);
        System.out.println();

        // ---------------------------------------------------------------
        // 4. Alice (FREE) borrows 2 books — should succeed
        // ---------------------------------------------------------------
        System.out.println("--- Alice (FREE) borrows up to her limit ---");
        BorrowingRecord aliceRecord1 = service.borrowBook("M001", "ISBN-001");
        BorrowingRecord aliceRecord2 = service.borrowBook("M001", "ISBN-003");
        System.out.println("Alice's current borrows: " + alice.getBorrowedBooks());
        System.out.println();

        // ---------------------------------------------------------------
        // 5. Alice tries to borrow a 3rd book — should be blocked
        // ---------------------------------------------------------------
        System.out.println("--- Alice tries to borrow a 3rd book (expect failure) ---");
        try {
            service.borrowBook("M001", "ISBN-005");
            System.out.println("ERROR: Should not have reached this line.");
        } catch (IllegalStateException e) {
            System.out.println("[EXPECTED ERROR] " + e.getMessage());
        }
        System.out.println();

        // ---------------------------------------------------------------
        // 6. Bob (PREMIUM) borrows multiple books with no restriction
        // ---------------------------------------------------------------
        System.out.println("--- Bob (PREMIUM) borrows multiple books ---");
        service.borrowBook("M002", "ISBN-002");
        service.borrowBook("M002", "ISBN-004");
        service.borrowBook("M002", "ISBN-005");
        System.out.println("Bob's current borrows: " + bob.getBorrowedBooks());
        System.out.println();

        // ---------------------------------------------------------------
        // 7. Simulate overdue return for Alice's physical book
        //    We manually patch the due date to simulate overdue.
        //    In the real system this is tested with mocked dates.
        // ---------------------------------------------------------------
        System.out.println("--- Simulating overdue return for Alice (Clean Code) ---");
        // The due date on aliceRecord1 is borrowDate + 14 days. We simulate
        // it being 5 days overdue by returning after 19 days. Since we can't
        // advance the clock in this demo, we note the behaviour here and test
        // it directly by constructing a record with a past due date.

        // Construct a backdated borrowing record manually to demonstrate fine calc.
        LocalDate pastBorrowDate = LocalDate.now().minusDays(20);
        LocalDate pastDueDate = LocalDate.now().minusDays(6);  // 6 days overdue
        BorrowingRecord overdueSimRecord =
                new BorrowingRecord("ISBN-006", "M001", pastBorrowDate, pastDueDate);

        System.out.println("Is overdue: " + overdueSimRecord.isOverdue());
        System.out.println("Days overdue: " + overdueSimRecord.getDaysOverdue());

        FineCalculator physicalCalc = FineCalculatorFactory.getCalculator(BookFormat.PHYSICAL);
        double simulatedFine = physicalCalc.calculateFine(overdueSimRecord.getDaysOverdue());
        System.out.println("Simulated fine for physical book overdue by "
                + overdueSimRecord.getDaysOverdue() + " days: $"
                + String.format("%.2f", simulatedFine));

        FineCalculator ebookCalc = FineCalculatorFactory.getCalculator(BookFormat.EBOOK);
        double ebookFine = ebookCalc.calculateFine(overdueSimRecord.getDaysOverdue());
        System.out.println("Fine for ebook (same days overdue): $"
                + String.format("%.2f", ebookFine));
        System.out.println();

        // ---------------------------------------------------------------
        // 8. Alice returns Clean Code (no fine in demo since not actually overdue)
        // ---------------------------------------------------------------
        System.out.println("--- Alice returns Clean Code ---");
        double fine = service.returnBook("M001", "ISBN-001");
        System.out.println("Fine charged: $" + String.format("%.2f", fine));
        System.out.println("Alice's borrows after return: " + alice.getBorrowedBooks());
        System.out.println("ISBN-001 status: " + catalog.getBook("ISBN-001").getStatus());
        System.out.println();

        // ---------------------------------------------------------------
        // 9. Waitlist scenario
        //    Bob borrowed ISBN-002 (Clean Architecture). Charlie joins waitlist.
        //    Bob returns the book. Charlie gets notified.
        // ---------------------------------------------------------------
        System.out.println("--- Waitlist Scenario: Charlie waits for Clean Architecture ---");
        System.out.println("ISBN-002 status: " + catalog.getBook("ISBN-002").getStatus());
        waitlistManager.addToWaitlist("M003", "ISBN-002");
        System.out.println("Waitlist size for ISBN-002: "
                + waitlistManager.getQueueSize("ISBN-002"));

        // Also add Alice to the waitlist (second in line).
        waitlistManager.addToWaitlist("M001", "ISBN-002");
        System.out.println("Waitlist size after Alice joins: "
                + waitlistManager.getQueueSize("ISBN-002"));
        System.out.println("Next in line: " + waitlistManager.peekNext("ISBN-002"));

        System.out.println();
        System.out.println("Bob returns Clean Architecture...");
        service.returnBook("M002", "ISBN-002");
        System.out.println("Waitlist size after notification: "
                + waitlistManager.getQueueSize("ISBN-002"));
        System.out.println();

        // ---------------------------------------------------------------
        // 10. Fine payment
        // ---------------------------------------------------------------
        System.out.println("--- Fine Payment Demo ---");
        // Manually add a fine to Alice to demonstrate the payment flow.
        alice.addFine(5.50);
        System.out.println("Alice's fine balance before payment: $"
                + String.format("%.2f", alice.getTotalFines()));
        service.payFine("M001", 3.00);
        System.out.println("Alice's fine balance after $3.00 payment: $"
                + String.format("%.2f", alice.getTotalFines()));
        service.payFine("M001", 100.00); // Overpayment — clamps to balance.
        System.out.println("Alice's fine balance after overpayment: $"
                + String.format("%.2f", alice.getTotalFines()));
        System.out.println();

        // ---------------------------------------------------------------
        // 11. Recommendations for Alice
        // ---------------------------------------------------------------
        System.out.println("--- Recommendations for Alice ---");
        // Alice currently has ISBN-003 (DDD Ebook by Eric Evans) borrowed.
        System.out.println("Alice currently has: " + alice.getBorrowedBooks());
        java.util.List<Book> recommendations = service.getRecommendations("M001", 3);
        System.out.println("Recommendations (" + recommendations.size() + "):");
        for (Book rec : recommendations) {
            System.out.println("  -> " + rec.getTitle() + " by " + rec.getAuthor()
                    + " [" + rec.getFormat() + "] - " + rec.getStatus());
        }
        System.out.println();

        // ---------------------------------------------------------------
        // 12. Catalog search demo
        // ---------------------------------------------------------------
        System.out.println("--- Catalog Search Demo ---");
        System.out.println("Search by author 'Eric Evans':");
        for (Book b : catalog.searchByAuthor("Eric Evans")) {
            System.out.println("  " + b.getTitle() + " [" + b.getFormat() + "] - " + b.getStatus());
        }

        System.out.println("Search by format PHYSICAL:");
        for (Book b : catalog.searchByFormat(BookFormat.PHYSICAL)) {
            System.out.println("  " + b.getTitle() + " [" + b.getFormat() + "] - " + b.getStatus());
        }

        System.out.println("Search by title 'design':");
        for (Book b : catalog.searchByTitle("design")) {
            System.out.println("  " + b.getTitle() + " [" + b.getFormat() + "] - " + b.getStatus());
        }
        System.out.println();

        // ---------------------------------------------------------------
        // 13. Active borrowings list
        // ---------------------------------------------------------------
        System.out.println("--- Active Borrowings ---");
        java.util.List<BorrowingRecord> active = service.getActiveBorrowings();
        System.out.println("Currently active borrowings: " + active.size());
        for (BorrowingRecord r : active) {
            System.out.println("  MemberId=" + r.getMemberId()
                    + " ISBN=" + r.getBookIsbn()
                    + " DueDate=" + r.getDueDate()
                    + " Overdue=" + r.isOverdue());
        }
        System.out.println();

        // ---------------------------------------------------------------
        // 14. Overdue report (no genuinely overdue records in real-time demo)
        // ---------------------------------------------------------------
        System.out.println("--- Admin: Overdue Report ---");
        admin.generateOverdueReport();
        System.out.println();

        // ---------------------------------------------------------------
        // 15. Admin removes a book
        // ---------------------------------------------------------------
        System.out.println("--- Admin: Remove a Book ---");
        System.out.println("Catalog size before removal: " + catalog.size());
        admin.removeBook("ISBN-006");
        System.out.println("Catalog size after removal: " + catalog.size());
        System.out.println();

        // ---------------------------------------------------------------
        // 16. Reservation demo
        // ---------------------------------------------------------------
        System.out.println("--- Reservation Demo ---");
        System.out.println("ISBN-005 (Effective Java) status: "
                + catalog.getBook("ISBN-005").getStatus());
        service.reserveBook("M002", "ISBN-005");
        System.out.println("ISBN-005 status after reserve: "
                + catalog.getBook("ISBN-005").getStatus());
        service.cancelReservation("M002", "ISBN-005");
        System.out.println("ISBN-005 status after cancel: "
                + catalog.getBook("ISBN-005").getStatus());
        System.out.println();

        // ---------------------------------------------------------------
        // 17. Wrap-up summary
        // ---------------------------------------------------------------
        System.out.println("=================================================");
        System.out.println("                  FINAL STATE                    ");
        System.out.println("=================================================");
        System.out.println("Alice : " + alice);
        System.out.println("Bob   : " + bob);
        System.out.println("Charlie: " + charlie);
        System.out.println();
        System.out.println("All books in catalog:");
        for (Book b : catalog.getAllBooks()) {
            System.out.println("  " + b);
        }
        System.out.println("=================================================");
        System.out.println("            DEMO COMPLETE                        ");
        System.out.println("=================================================");
    }
}
```

---

## Unit Tests Design

The following describes each test class, the test method names, and the assertion logic. These are designed for JUnit 5 with standard assertions.

### BookCatalogTest

```
Tests for: BookCatalog
```

| Test Method | Purpose | Key Assertions |
|---|---|---|
| `addBook_shouldStoreBookByIsbn` | Adding a valid book makes it retrievable by ISBN | `assertNotNull(catalog.getBook("ISBN-001"))`, `assertEquals(1, catalog.size())` |
| `addBook_shouldThrow_whenDuplicateIsbn` | Adding a book with an existing ISBN throws | `assertThrows(IllegalArgumentException.class, ...)` |
| `removeBook_shouldReturnRemovedBook` | Removing by ISBN returns the book and reduces size | `assertNotNull(removed)`, `assertEquals(0, catalog.size())` |
| `getBook_shouldReturnNull_whenNotFound` | Querying an unknown ISBN returns null | `assertNull(catalog.getBook("UNKNOWN"))` |
| `searchByAuthor_shouldReturnMatchingBooks` | Case-insensitive partial author match | `assertEquals(2, results.size())`, all results have expected author |
| `searchByFormat_shouldFilterByFormat` | Only books of requested format returned | all results have `BookFormat.PHYSICAL` |
| `getAllAvailable_shouldExcludeBorrowedBooks` | Books with BORROWED status not returned | result size decreases after a book's status is set to BORROWED |

### BorrowingPolicyTest

```
Tests for: FreeMemberPolicy, PremiumMemberPolicy, BorrowingPolicyFactory
```

| Test Method | Purpose | Key Assertions |
|---|---|---|
| `freeMemberPolicy_canBorrow_whenUnderLimit` | Member with 1 borrow can borrow again | `assertTrue(policy.canBorrow(memberWith1Book))` |
| `freeMemberPolicy_cannotBorrow_whenAtLimit` | Member with 2 borrows is blocked | `assertFalse(policy.canBorrow(memberWith2Books))` |
| `freeMemberPolicy_borrowDuration_is14Days` | Duration accessor returns 14 | `assertEquals(14, policy.getBorrowDurationDays())` |
| `premiumMemberPolicy_canAlwaysBorrow` | Premium members never blocked | `assertTrue(policy.canBorrow(memberWithManyBooks))` |
| `premiumMemberPolicy_borrowDuration_is21Days` | Duration accessor returns 21 | `assertEquals(21, policy.getBorrowDurationDays())` |
| `factory_returnsFreeMemberPolicy_forFreeTier` | Factory returns correct type | `assertInstanceOf(FreeMemberPolicy.class, BorrowingPolicyFactory.getPolicy(MembershipTier.FREE))` |
| `factory_returnsPremiumMemberPolicy_forPremiumTier` | Factory returns correct type | `assertInstanceOf(PremiumMemberPolicy.class, BorrowingPolicyFactory.getPolicy(MembershipTier.PREMIUM))` |

### FineCalculatorTest

```
Tests for: PhysicalBookFineCalculator, NoFineCalculator, FineCalculatorFactory
```

| Test Method | Purpose | Key Assertions |
|---|---|---|
| `physicalCalculator_returnsZero_whenNotOverdue` | Zero days overdue yields $0.00 | `assertEquals(0.0, calc.calculateFine(0), 0.001)` |
| `physicalCalculator_returnsCorrectFine_for5Days` | 5 days × $0.50 = $2.50 | `assertEquals(2.50, calc.calculateFine(5), 0.001)` |
| `physicalCalculator_returnsCorrectFine_for10Days` | 10 days × $0.50 = $5.00 | `assertEquals(5.00, calc.calculateFine(10), 0.001)` |
| `noFineCalculator_alwaysReturnsZero` | Digital format always $0.00 | `assertEquals(0.0, calc.calculateFine(100), 0.001)` |
| `factory_returnsPhysicalCalculator_forPhysicalFormat` | Factory maps PHYSICAL correctly | `assertInstanceOf(PhysicalBookFineCalculator.class, FineCalculatorFactory.getCalculator(BookFormat.PHYSICAL))` |
| `factory_returnsNoFineCalculator_forEbook` | Factory maps EBOOK correctly | `assertInstanceOf(NoFineCalculator.class, FineCalculatorFactory.getCalculator(BookFormat.EBOOK))` |
| `factory_returnsNoFineCalculator_forAudiobook` | Factory maps AUDIOBOOK correctly | `assertInstanceOf(NoFineCalculator.class, FineCalculatorFactory.getCalculator(BookFormat.AUDIOBOOK))` |

### WaitlistManagerTest

```
Tests for: WaitlistManager, MemberNotificationObserver (via mock/capture)
```

| Test Method | Purpose | Key Assertions |
|---|---|---|
| `addToWaitlist_shouldIncreaseQueueSize` | Adding one entry increases queue by 1 | `assertEquals(1, manager.getQueueSize("ISBN-001"))` |
| `addToWaitlist_shouldIgnoreDuplicateMember` | Same member added twice — size stays 1 | `assertEquals(1, manager.getQueueSize("ISBN-001"))` |
| `removeFromWaitlist_shouldDecreaseQueueSize` | Removing an entry decreases queue | `assertEquals(0, manager.getQueueSize("ISBN-001"))` |
| `notifyNext_shouldDequeueFirstEntry_andFireObserver` | FIFO: first member added is first notified; observer called with correct args | observer's `onBookAvailable` called with first member's ID; queue size decreases by 1 |
| `hasWaiters_returnsFalse_whenQueueIsEmpty` | Empty queue returns false | `assertFalse(manager.hasWaiters("ISBN-001"))` |
| `clearWaitlist_shouldRemoveAllEntries` | Clear empties the waitlist completely | `assertEquals(0, manager.getQueueSize("ISBN-001"))` |

### LibraryServiceTest

```
Tests for: LibraryService (integration of catalog, service, policy, fine, waitlist)
```

| Test Method | Purpose | Key Assertions |
|---|---|---|
| `registerMember_shouldMakeMemberRetrievable` | Registered member found by ID | `assertNotNull(service.getMember("M001"))` |
| `borrowBook_shouldTransitionBookToBorrowed` | After borrow, book status is BORROWED | `assertEquals(BookStatus.BORROWED, catalog.getBook("ISBN-001").getStatus())` |
| `borrowBook_shouldAddIsbnToMemberBorrows` | Borrow list updated | `assertTrue(member.getBorrowedBooks().contains("ISBN-001"))` |
| `borrowBook_shouldThrow_whenBookNotAvailable` | Borrowing BORROWED book throws | `assertThrows(IllegalStateException.class, ...)` |
| `borrowBook_shouldThrow_whenFreeMemberAtLimit` | Third borrow for FREE member throws | `assertThrows(IllegalStateException.class, ...)` |
| `returnBook_shouldTransitionBookToAvailable` | After return, book is AVAILABLE | `assertEquals(BookStatus.AVAILABLE, catalog.getBook("ISBN-001").getStatus())` |
| `returnBook_shouldApplyFine_forOverduePhysicalBook` | Fine applied when backdated due date passed | `assertTrue(member.getTotalFines() > 0)` |
| `returnBook_shouldNotApplyFine_forOverdueEbook` | No fine for digital overdue | `assertEquals(0.0, member.getTotalFines(), 0.001)` |
| `payFine_shouldReduceMemberBalance` | Payment reduces totalFines | balance decreases by paid amount |
| `getOverdueRecords_shouldReturnOnlyOverdueOpenRecords` | Only overdue non-returned records returned | all records in result are overdue and not returned |

---

## Extension Exercise: Adding Magazine Format

This exercise demonstrates the Open/Closed Principle in action. You must extend the system to support a MAGAZINE format with a $0.25/day overdue fine and a 7-day loan period — without modifying any existing class except the two factory switch statements and the enum.

### Step 1: Update BookFormat enum

```java
package library;

public enum BookFormat {
    EBOOK,
    AUDIOBOOK,
    PHYSICAL,
    MAGAZINE   // <-- new constant added
}
```

### Step 2: Implement MagazineFineCalculator

```java
package library;

/**
 * Fine calculator for magazine format.
 *
 * Rule: $0.25 per day overdue.
 * Magazines are time-sensitive publications; a lower fine than physical
 * books reflects their lower individual value.
 */
public class MagazineFineCalculator implements FineCalculator {

    private static final double FINE_RATE_PER_DAY = 0.25;

    /**
     * Calculates the fine as: daysOverdue * $0.25.
     *
     * @param daysOverdue number of days past the due date
     * @return total fine in dollars
     */
    @Override
    public double calculateFine(long daysOverdue) {
        if (daysOverdue <= 0) {
            return 0.0;
        }
        return daysOverdue * FINE_RATE_PER_DAY;
    }
}
```

### Step 3: Update FineCalculatorFactory

```java
package library;

public class FineCalculatorFactory {

    private FineCalculatorFactory() {
        throw new UnsupportedOperationException("FineCalculatorFactory is a utility class.");
    }

    public static FineCalculator getCalculator(BookFormat format) {
        if (format == null) {
            throw new IllegalArgumentException("BookFormat must not be null.");
        }
        switch (format) {
            case PHYSICAL:
                return new PhysicalBookFineCalculator();
            case EBOOK:
                return new NoFineCalculator();
            case AUDIOBOOK:
                return new NoFineCalculator();
            case MAGAZINE:
                return new MagazineFineCalculator();   // <-- added case
            default:
                throw new IllegalArgumentException("No FineCalculator defined for format: " + format);
        }
    }
}
```

### Step 4: Implement MagazineBorrowingPolicy

```java
package library;

/**
 * Borrowing policy for MAGAZINE format items.
 *
 * Rules:
 *  - FREE members: limited to 3 concurrent magazines (separate from book limit).
 *    For simplicity this demo uses the same member borrow count.
 *  - PREMIUM members: unlimited.
 *  - Loan duration: 7 days (magazines are weekly publications).
 */
public class MagazineBorrowingPolicy implements BorrowingPolicy {

    private static final int BORROW_LIMIT = 3;
    private static final int BORROW_DURATION_DAYS = 7;

    /**
     * A member can borrow a magazine if they hold fewer than 3 items.
     *
     * @param member the member attempting to borrow
     * @return true if under the magazine borrow limit
     */
    @Override
    public boolean canBorrow(Member member) {
        if (member == null) {
            throw new IllegalArgumentException("Member must not be null.");
        }
        return member.getCurrentBorrowCount() < BORROW_LIMIT;
    }

    /**
     * @return 3
     */
    @Override
    public int getBorrowLimit() {
        return BORROW_LIMIT;
    }

    /**
     * @return 7
     */
    @Override
    public int getBorrowDurationDays() {
        return BORROW_DURATION_DAYS;
    }
}
```

### Step 5: Update BorrowingPolicyFactory

> Note: In this extension the borrowing policy is still tier-based, not format-based.
> If magazine items require a distinct policy regardless of tier, the factory signature
> should be extended. The example below shows a format-aware overload for future flexibility.

```java
package library;

public class BorrowingPolicyFactory {

    private BorrowingPolicyFactory() {
        throw new UnsupportedOperationException("BorrowingPolicyFactory is a utility class.");
    }

    /**
     * Returns the tier-based borrowing policy (existing behaviour unchanged).
     */
    public static BorrowingPolicy getPolicy(MembershipTier tier) {
        if (tier == null) {
            throw new IllegalArgumentException("MembershipTier must not be null.");
        }
        switch (tier) {
            case FREE:
                return new FreeMemberPolicy();
            case PREMIUM:
                return new PremiumMemberPolicy();
            default:
                throw new IllegalArgumentException("No BorrowingPolicy defined for tier: " + tier);
        }
    }

    /**
     * Returns a format-specific policy override when one exists;
     * falls back to the tier-based policy.
     *
     * @param tier   the member's subscription tier
     * @param format the book format being borrowed
     * @return the most specific applicable BorrowingPolicy
     */
    public static BorrowingPolicy getPolicy(MembershipTier tier, BookFormat format) {
        if (format == null) {
            return getPolicy(tier);
        }
        switch (format) {
            case MAGAZINE:
                return new MagazineBorrowingPolicy();   // <-- format-specific override
            default:
                return getPolicy(tier);
        }
    }
}
```

### OCP Compliance Explanation

The Open/Closed Principle states: **software entities should be open for extension but closed for modification**.

| Change made | Class modified? | Verdict |
|---|---|---|
| `BookFormat.MAGAZINE` constant added | `BookFormat.java` — enum constant added | **Acceptable** — enum constants are extension points by design; no business logic changed |
| `MagazineFineCalculator` created | New file | **Extension only** — `FineCalculator` interface unchanged |
| `FineCalculatorFactory` — one `case` added | One line added to switch | **Minimal, isolated** — adding a mapping entry is the intended use of a factory; no algorithmic change |
| `MagazineBorrowingPolicy` created | New file | **Extension only** — `BorrowingPolicy` interface unchanged |
| `BorrowingPolicyFactory` — overloaded method added | New overload added; original untouched | **Backward compatible extension** |
| `LibraryService` | **Not modified** | OCP satisfied |
| `BorrowingRecord` | **Not modified** | OCP satisfied |
| `Member` | **Not modified** | OCP satisfied |
| `BookCatalog` | **Not modified** | OCP satisfied |

The two factory switch statements are the only "modification" — and they represent the minimum necessary seam. All consumer code (LibraryService, LibraryAdmin, demo) is completely untouched.

---

## SOLID Principles Checklist

### S — Single Responsibility Principle

> A class should have only one reason to change.

| Class | Single Responsibility | Reason it Changes |
|---|---|---|
| `Book` | Represents a book entity with its data | Book domain model changes (new field, format rule) |
| `Member` | Represents a library member and their state | Member domain model changes (new field, tier rules) |
| `BorrowingRecord` | Records and queries a single borrow transaction | Borrow transaction lifecycle changes |
| `PhysicalBookFineCalculator` | Computes fines for physical books | Fine rate for physical books changes |
| `NoFineCalculator` | Returns zero fine for digital books | Digital fine policy changes |
| `WaitlistManager` | Manages per-book queues and fires observer events | Waitlist data structure or notification trigger logic changes |
| `LibraryAdmin` | Administrative operations (add, remove, report) | Admin-facing requirements change |
| `LibraryService` | Orchestrates borrowing/return/reservation workflow | Core borrowing workflow changes |
| `BookCatalog` | CRUD and search over the in-memory book map | Catalog storage strategy changes |

SRP is **violated** when a class has multiple reasons to change. Example of a violation to avoid: putting fine calculation logic directly inside `LibraryService.returnBook()` would mean changing `LibraryService` for both workflow changes AND fine rule changes.

### O — Open/Closed Principle

> Open for extension, closed for modification.

| Extension point | Interface/Abstract class | Implementations |
|---|---|---|
| Fine calculation rules | `FineCalculator` | `PhysicalBookFineCalculator`, `NoFineCalculator`, `MagazineFineCalculator` |
| Borrowing rules | `BorrowingPolicy` | `FreeMemberPolicy`, `PremiumMemberPolicy`, `MagazineBorrowingPolicy` |
| Notification channels | `WaitlistObserver` | `MemberNotificationObserver`, (future: EmailObserver, SmsObserver) |
| Recommendation algorithms | `RecommendationEngine` | `SimpleCatalogRecommendationEngine`, (future: MLRecommendationEngine) |

`LibraryService` never needs to change when a new book format is added — the factory + interface combination absorbs the extension.

### L — Liskov Substitution Principle

> Subtypes must be substitutable for their base types.

| Interface | Implementations | LSP compliance |
|---|---|---|
| `FineCalculator` | Both `PhysicalBookFineCalculator` and `NoFineCalculator` accept a `long` days parameter and return a `double` >= 0 | **Compliant** — no surprise behaviour; postconditions are preserved |
| `BorrowingPolicy` | Both `FreeMemberPolicy` and `PremiumMemberPolicy` implement `canBorrow(Member)` and return boolean without side effects | **Compliant** — `canBorrow` has no side effects; return type contract preserved |
| `WaitlistObserver` | `MemberNotificationObserver` calls `onBookAvailable` with the args passed in — no exceptions, no state mutation of caller | **Compliant** |
| `BorrowingPolicy.canBorrow` and `PremiumMemberPolicy` | `PremiumMemberPolicy.canBorrow` always returns `true`; it does NOT strengthen the precondition (it doesn't require extra state beyond `member != null`) | **Compliant** |

A common LSP violation to watch for: if a future `RestrictedPolicy.canBorrow` threw an exception for members with unpaid fines without that being declared in the interface contract, it would violate LSP (strengthening preconditions).

### I — Interface Segregation Principle

> Clients should not be forced to depend on interfaces they do not use.

| Interface | Width | Reason it is appropriately narrow |
|---|---|---|
| `FineCalculator` | 1 method: `calculateFine(long)` | Only fine-computing clients need this |
| `BorrowingPolicy` | 3 methods: `canBorrow`, `getBorrowLimit`, `getBorrowDurationDays` | These 3 are cohesive: all describe the same policy dimension |
| `WaitlistObserver` | 1 method: `onBookAvailable` | Event listener; no other concern |
| `RecommendationEngine` | 1 method: `recommend` | Single entry point; implementation details hidden |

No client is forced to implement methods it doesn't need. If `BorrowingPolicy` had included `calculateFine()`, that would be an ISP violation.

### D — Dependency Inversion Principle

> High-level modules should not depend on low-level modules. Both should depend on abstractions.

| High-level module | Depends on abstraction | Not on concrete class |
|---|---|---|
| `LibraryService` | `FineCalculator` (via factory) | Not `PhysicalBookFineCalculator` |
| `LibraryService` | `BorrowingPolicy` (via factory) | Not `FreeMemberPolicy` |
| `LibraryService` | `RecommendationEngine` interface | Not `SimpleCatalogRecommendationEngine` |
| `LibraryService` | `WaitlistManager` (injected) | Concrete but treated as a collaborator; could be behind interface |
| `LibraryAdmin` | `LibraryService` (injected via constructor) | Not instantiated internally |
| `WaitlistManager` | `WaitlistObserver` interface | Not `MemberNotificationObserver` |

The constructor injection pattern in `LibraryService` and `LibraryAdmin` allows test doubles (mocks/stubs) to replace any collaborator, confirming DIP compliance.

---

*End of Capstone Project 1: Online Library Platform*
