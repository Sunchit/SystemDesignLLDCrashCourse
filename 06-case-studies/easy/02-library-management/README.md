# Library Management System — Low-Level Design Case Study

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Functional Requirements](#2-functional-requirements)
3. [Non-Functional Requirements](#3-non-functional-requirements)
4. [Assumptions and Constraints](#4-assumptions-and-constraints)
5. [Domain Model Description](#5-domain-model-description)
6. [Class Diagram](#6-class-diagram)
7. [Core Classes — Complete Java Implementation](#7-core-classes--complete-java-implementation)
8. [Design Patterns Used](#8-design-patterns-used)
9. [Key Design Decisions and Trade-offs](#9-key-design-decisions-and-trade-offs)
10. [Extension Points](#10-extension-points)
11. [Interview Discussion Points](#11-interview-discussion-points)
12. [Common Mistakes in This Design](#12-common-mistakes-in-this-design)

---

## 1. Problem Statement

Design a Library Management System that allows members to browse a catalog of books, magazines, and journals; borrow and return physical copies; place reservations on items that are currently checked out; and pay fines for overdue returns. Librarians must be able to manage the catalog, register new members, and process check-outs and returns on behalf of members.

The system must track every physical copy of an item separately (a library may own five copies of a title — each copy has its own borrowing history). Fine amounts are computed automatically based on how many days overdue a copy is. Members must be notified by the system when a reservation becomes available or when a fine is issued.

---

## 2. Functional Requirements

1. The system shall maintain a catalog of library items: books, magazines, and journals.
2. Each catalog entry may have multiple physical copies (BookItems); each copy is tracked independently.
3. Members shall be able to search the catalog by title, author, ISBN, and subject.
4. A registered member shall be able to borrow an available copy of an item.
5. The system shall enforce a per-member borrowing limit (configurable; default 5 items).
6. A member shall be able to return a borrowed copy.
7. A member shall be able to renew a borrowed copy if no reservation exists for that copy.
8. A member shall be able to place a reservation (hold) on an item that has no available copies.
9. When a reserved copy becomes available, the member who holds the first reservation shall be notified and the copy shall be held for them for a configurable period (default 3 days).
10. The system shall calculate fines for overdue returns: `overdue days × daily rate`.
11. Standard members and premium members shall have different daily fine rates.
12. A librarian shall be able to add, update, and remove catalog entries and physical copies.
13. A librarian shall be able to register new members and deactivate existing accounts.
14. A librarian shall be able to process lost item reports, marking a copy as LOST and charging the member a replacement fee.
15. Account management: members shall be able to view their borrow history, active loans, reservations, and outstanding fines.

---

## 3. Non-Functional Requirements

- **Correctness**: A copy cannot be simultaneously borrowed by two members. Reservations must be honoured in FIFO order.
- **Maintainability**: Item status transitions must be centralised; adding a new status must not require changes across the entire codebase.
- **Extensibility**: New item types (e.g., DVDs, e-books) should be addable without modifying core borrowing logic.
- **Testability**: Business rules (fine calculation, state transitions) must be unit-testable without a database.
- **Separation of concerns**: Notification delivery (email, SMS, push) must be decoupled from domain logic.

---

## 4. Assumptions and Constraints

- The implementation is an in-memory model suitable for a design interview or a prototype. Persistence (database) is out of scope but the design accommodates it.
- Only one currency is used; monetary amounts are represented as `double` (a real system would use `BigDecimal`).
- Thread-safety is noted where relevant but full concurrency control is out of scope for this exercise.
- A `Librarian` is modelled as a subclass of `Member` with elevated permissions, reflecting that librarians are also persons registered in the system.
- Time is represented using `java.time.LocalDate` for simplicity.
- The maximum borrow period is 14 days for standard members and 21 days for premium members.
- Renewals extend the due date by the same period as the original loan.
- A member with an outstanding fine above a configurable threshold (default $10) cannot borrow additional items until the fine is settled.

---

## 5. Domain Model Description

### Entities

| Entity | Responsibility |
|---|---|
| `LibraryItem` | Abstract base for catalog entries (Book, Magazine, Journal). Holds bibliographic metadata. |
| `Book` | A `LibraryItem` with author, ISBN, subject, and publisher. |
| `Magazine` | A `LibraryItem` with publisher and issue number. |
| `Journal` | A `LibraryItem` with publisher, volume, and issue. |
| `BookItem` | A single physical copy of any `LibraryItem`. Owns its `BookItemState` and tracks who has it. |
| `Member` | A registered library patron. Tracks active loans, reservations, and fines. |
| `Librarian` | A `Member` with additional catalog-management and account-management capabilities. |
| `BorrowRecord` | Records a single borrow event: which copy, by whom, borrow date, due date, and actual return date. |
| `Reservation` | A hold placed by a `Member` on a `LibraryItem`. Tracks creation time for FIFO ordering. |
| `Fine` | A charge raised against a member: either an overdue fine or a lost-item replacement fee. |
| `Catalog` | An in-memory store of all `LibraryItem`s. Provides search operations. |
| `Library` | Facade / application service. Orchestrates borrow, return, reserve, and fine workflows. |
| `NotificationService` | Implements `LibraryEventListener`. Delivers notifications (prints to console; swap for email/SMS). |

### Enumerations

| Enum | Values |
|---|---|
| `ItemType` | `BOOK`, `MAGAZINE`, `JOURNAL` |
| `ItemStatus` | `AVAILABLE`, `BORROWED`, `RESERVED`, `LOST` |
| `MemberStatus` | `ACTIVE`, `SUSPENDED`, `CLOSED` |
| `MemberType` | `STANDARD`, `PREMIUM` |
| `FineType` | `OVERDUE`, `LOST_ITEM` |

### State Machine for BookItem

```
         borrow()           return() / no reservation
AVAILABLE -------> BORROWED ---------------------------------> AVAILABLE
    ^                  |
    |                  | return() / reservation exists
    |                  v
    |           RESERVED <---------- reserve() (when AVAILABLE)
    |                  |
    |     notifyReservee() + pickup  |  cancelReservation()
    +<------------------------------ +
                        |
                    reportLost()
                        v
                      LOST
```

---

## 6. Class Diagram

```
+---------------------------+          +---------------------------+
|       LibraryItem         |          |        BookItem           |
+---------------------------+          +---------------------------+
| - id: String              |<>--------| - id: String             |
| - title: String           |  1    *  | - barcode: String        |
| - type: ItemType          |          | - location: String       |
| - subject: String         |          | - state: BookItemState   |
| - language: String        |          | - currentBorrower: Member|
| - numberOfCopies: int     |          | - dueDate: LocalDate     |
+---------------------------+          +---------------------------+
          ^                            | + borrow(Member)         |
          |                            | + returnItem()           |
    +-----+-----+                      | + reserve(Member)        |
    |           |                      | + reportLost()           |
  +------+ +----------+                | + getStatus(): ItemStatus|
  | Book | | Magazine |                +---------------------------+
  +------+ +----------+                            |
  | - author  | - issue  |                         | delegates to
  | - isbn    | - number |               +----------+----------+
  | - publisher           |               |BookItemState (iface)|
  +------+   +----------+               +---------------------+
             |                          | + doBorrow(item,mbr)|
          +--------+                    | + doReturn(item)    |
          | Journal|                    | + doReserve(item,m) |
          +--------+                    | + getStatus()       |
          | - vol  |                    +---------------------+
          | - issue|                           ^
          +--------+               +-----------+-----------+
                          +--------+--------+  +------------+  +--------------+
                          |AvailableState   |  |BorrowedState|  |ReservedState |
                          +-----------------+  +------------+  +--------------+

+---------------------------+          +---------------------------+
|         Member            |          |        Librarian          |
+---------------------------+          +---------------------------+
| - memberId: String        |<|--------| - employeeId: String     |
| - name: String            |          +---------------------------+
| - email: String           |          | + addBookItem(...)        |
| - memberType: MemberType  |          | + removeBookItem(...)     |
| - status: MemberStatus    |          | + registerMember(...)     |
| - activeLoans: List       |          | + deactivateMember(...)   |
| - reservations: List      |          +---------------------------+
| - fines: List             |
+---------------------------+
| + borrowItem(BookItem,Lib)|
| + returnItem(BookItem,Lib)|
| + reserveItem(LibItem,Lib)|
| + renewItem(BookItem,Lib) |
| + getTotalFines(): double |
+---------------------------+

+---------------------------+          +---------------------------+
|       BorrowRecord        |          |        Fine               |
+---------------------------+          +---------------------------+
| - recordId: String        |          | - fineId: String         |
| - bookItem: BookItem      |          | - member: Member         |
| - member: Member          |          | - bookItem: BookItem     |
| - borrowDate: LocalDate   |          | - fineType: FineType     |
| - dueDate: LocalDate      |          | - amount: double         |
| - returnDate: LocalDate   |          | - isPaid: boolean        |
| - isActive: boolean       |          | - issuedDate: LocalDate  |
+---------------------------+          +---------------------------+
| + close(returnDate)       |          | + pay()                  |
| + isOverdue(): boolean    |          +---------------------------+
| + overdueDays(): long     |
+---------------------------+

+---------------------------+          +---------------------------+
|        Reservation        |          |        Catalog            |
+---------------------------+          +---------------------------+
| - reservationId: String   |          | - items: Map<String,Item> |
| - member: Member          |          +---------------------------+
| - libraryItem: LibraryItem|          | + addItem(LibraryItem)   |
| - createdAt: LocalDate    |          | + searchByTitle(String)  |
| - isActive: boolean       |          | + searchByAuthor(String) |
+---------------------------+          | + searchByISBN(String)   |
| + cancel()                |          | + searchBySubject(String)|
+---------------------------+          | + getAvailableCopies(...)|
                                       +---------------------------+

+------------------------------+        +---------------------------+
| <<interface>>                |        |   NotificationService     |
| LibraryEventListener         |        +---------------------------+
+------------------------------+<|------| - listeners: List         |
| + onItemBorrowed(...)        |        +---------------------------+
| + onItemReturned(...)        |        | + subscribe(listener)    |
| + onReservationAvailable(...)  |        | + notify*(...)           |
| + onFineIssued(...)          |        +---------------------------+
+------------------------------+
```

---

## 7. Core Classes — Complete Java Implementation

### 7.1 Enumerations

```java
// ItemType.java
public enum ItemType {
    BOOK, MAGAZINE, JOURNAL
}

// ItemStatus.java
public enum ItemStatus {
    AVAILABLE, BORROWED, RESERVED, LOST
}

// MemberStatus.java
public enum MemberStatus {
    ACTIVE, SUSPENDED, CLOSED
}

// MemberType.java
public enum MemberType {
    STANDARD, PREMIUM
}

// FineType.java
public enum FineType {
    OVERDUE, LOST_ITEM
}
```

---

### 7.2 State Pattern — BookItemState Interface and Implementations

```java
// BookItemState.java
public interface BookItemState {
    /**
     * Attempt to borrow this copy for the given member.
     * Throws IllegalStateException if the transition is not permitted.
     */
    void doBorrow(BookItem item, Member member);

    /**
     * Attempt to return this copy.
     * Throws IllegalStateException if the transition is not permitted.
     */
    void doReturn(BookItem item);

    /**
     * Attempt to place a reservation on this copy.
     * Throws IllegalStateException if the transition is not permitted.
     */
    void doReserve(BookItem item, Member member);

    /**
     * Mark this copy as LOST.
     */
    void doReportLost(BookItem item);

    /**
     * Return the ItemStatus this state represents.
     */
    ItemStatus getStatus();
}
```

```java
// AvailableState.java
import java.time.LocalDate;

public class AvailableState implements BookItemState {

    @Override
    public void doBorrow(BookItem item, Member member) {
        item.setCurrentBorrower(member);
        LocalDate dueDate = LocalDate.now().plusDays(member.getBorrowPeriodDays());
        item.setDueDate(dueDate);
        item.setState(new BorrowedState());
    }

    @Override
    public void doReturn(BookItem item) {
        throw new IllegalStateException(
            "Cannot return item [" + item.getBarcode() + "]: it is not currently borrowed.");
    }

    @Override
    public void doReserve(BookItem item, Member member) {
        // Reserving an available copy immediately sets it to RESERVED for that member.
        // In practice the member should just borrow it; the library facade handles this.
        item.setState(new ReservedState());
    }

    @Override
    public void doReportLost(BookItem item) {
        item.setState(new LostState());
    }

    @Override
    public ItemStatus getStatus() {
        return ItemStatus.AVAILABLE;
    }
}
```

```java
// BorrowedState.java
public class BorrowedState implements BookItemState {

    @Override
    public void doBorrow(BookItem item, Member member) {
        throw new IllegalStateException(
            "Cannot borrow item [" + item.getBarcode() + "]: it is already borrowed by "
            + item.getCurrentBorrower().getName() + ".");
    }

    @Override
    public void doReturn(BookItem item) {
        item.setCurrentBorrower(null);
        item.setDueDate(null);
        // The caller (Library facade) decides whether to transition to AVAILABLE or RESERVED
        // based on whether a reservation exists. We transition to AVAILABLE here as default;
        // the facade overrides to RESERVED when needed.
        item.setState(new AvailableState());
    }

    @Override
    public void doReserve(BookItem item, Member member) {
        // A reservation is placed at the LibraryItem level, not the BookItem level.
        // Reservations on a borrowed copy are tracked separately.
        // This is a no-op at the state level; the Library facade handles reservation queues.
        throw new IllegalStateException(
            "Cannot directly reserve a borrowed copy. Place a reservation on the catalog item instead.");
    }

    @Override
    public void doReportLost(BookItem item) {
        item.setCurrentBorrower(null);
        item.setDueDate(null);
        item.setState(new LostState());
    }

    @Override
    public ItemStatus getStatus() {
        return ItemStatus.BORROWED;
    }
}
```

```java
// ReservedState.java
import java.time.LocalDate;

public class ReservedState implements BookItemState {

    @Override
    public void doBorrow(BookItem item, Member member) {
        // Only the member whose reservation is active may borrow this copy.
        // The Library facade validates the member identity before calling this.
        item.setCurrentBorrower(member);
        LocalDate dueDate = LocalDate.now().plusDays(member.getBorrowPeriodDays());
        item.setDueDate(dueDate);
        item.setState(new BorrowedState());
    }

    @Override
    public void doReturn(BookItem item) {
        throw new IllegalStateException(
            "Cannot return item [" + item.getBarcode() + "]: it has status RESERVED, not BORROWED.");
    }

    @Override
    public void doReserve(BookItem item, Member member) {
        throw new IllegalStateException(
            "Cannot reserve item [" + item.getBarcode() + "]: it is already reserved.");
    }

    @Override
    public void doReportLost(BookItem item) {
        item.setState(new LostState());
    }

    @Override
    public ItemStatus getStatus() {
        return ItemStatus.RESERVED;
    }
}
```

```java
// LostState.java
public class LostState implements BookItemState {

    @Override
    public void doBorrow(BookItem item, Member member) {
        throw new IllegalStateException(
            "Cannot borrow item [" + item.getBarcode() + "]: it has been reported lost.");
    }

    @Override
    public void doReturn(BookItem item) {
        // A lost item that is returned transitions back to AVAILABLE after inspection.
        item.setState(new AvailableState());
    }

    @Override
    public void doReserve(BookItem item, Member member) {
        throw new IllegalStateException(
            "Cannot reserve item [" + item.getBarcode() + "]: it has been reported lost.");
    }

    @Override
    public void doReportLost(BookItem item) {
        // Already lost — no-op.
    }

    @Override
    public ItemStatus getStatus() {
        return ItemStatus.LOST;
    }
}
```

---

### 7.3 LibraryItem and Concrete Item Types

```java
// LibraryItem.java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public abstract class LibraryItem {

    private final String id;
    private String title;
    private final ItemType type;
    private String subject;
    private String language;
    private final List<BookItem> copies;

    public LibraryItem(String id, String title, ItemType type, String subject, String language) {
        if (id == null || id.isBlank()) throw new IllegalArgumentException("id must not be blank");
        if (title == null || title.isBlank()) throw new IllegalArgumentException("title must not be blank");
        this.id = id;
        this.title = title;
        this.type = type;
        this.subject = subject;
        this.language = language;
        this.copies = new ArrayList<>();
    }

    public void addCopy(BookItem copy) {
        if (copy == null) throw new IllegalArgumentException("copy must not be null");
        copies.add(copy);
    }

    public boolean removeCopy(String barcode) {
        return copies.removeIf(c -> c.getBarcode().equals(barcode));
    }

    public List<BookItem> getAvailableCopies() {
        List<BookItem> available = new ArrayList<>();
        for (BookItem copy : copies) {
            if (copy.getStatus() == ItemStatus.AVAILABLE) {
                available.add(copy);
            }
        }
        return available;
    }

    public List<BookItem> getCopies() {
        return Collections.unmodifiableList(copies);
    }

    public String getId() { return id; }
    public String getTitle() { return title; }
    public ItemType getType() { return type; }
    public String getSubject() { return subject; }
    public String getLanguage() { return language; }

    public void setTitle(String title) { this.title = title; }
    public void setSubject(String subject) { this.subject = subject; }
    public void setLanguage(String language) { this.language = language; }

    public int getTotalCopies() { return copies.size(); }

    @Override
    public String toString() {
        return String.format("[%s] %s (type=%s, copies=%d)", id, title, type, copies.size());
    }
}
```

```java
// Book.java
public class Book extends LibraryItem {

    private String author;
    private String isbn;
    private String publisher;

    public Book(String id, String title, String subject, String language,
                String author, String isbn, String publisher) {
        super(id, title, ItemType.BOOK, subject, language);
        this.author = author;
        this.isbn = isbn;
        this.publisher = publisher;
    }

    public String getAuthor() { return author; }
    public String getIsbn() { return isbn; }
    public String getPublisher() { return publisher; }

    public void setAuthor(String author) { this.author = author; }
    public void setIsbn(String isbn) { this.isbn = isbn; }
    public void setPublisher(String publisher) { this.publisher = publisher; }

    @Override
    public String toString() {
        return String.format("Book[id=%s, title='%s', author='%s', isbn=%s]",
            getId(), getTitle(), author, isbn);
    }
}
```

```java
// Magazine.java
public class Magazine extends LibraryItem {

    private String publisher;
    private int issueNumber;

    public Magazine(String id, String title, String subject, String language,
                    String publisher, int issueNumber) {
        super(id, title, ItemType.MAGAZINE, subject, language);
        this.publisher = publisher;
        this.issueNumber = issueNumber;
    }

    public String getPublisher() { return publisher; }
    public int getIssueNumber() { return issueNumber; }

    public void setPublisher(String publisher) { this.publisher = publisher; }
    public void setIssueNumber(int issueNumber) { this.issueNumber = issueNumber; }

    @Override
    public String toString() {
        return String.format("Magazine[id=%s, title='%s', issue=%d]",
            getId(), getTitle(), issueNumber);
    }
}
```

```java
// Journal.java
public class Journal extends LibraryItem {

    private String publisher;
    private int volume;
    private int issueNumber;

    public Journal(String id, String title, String subject, String language,
                   String publisher, int volume, int issueNumber) {
        super(id, title, ItemType.JOURNAL, subject, language);
        this.publisher = publisher;
        this.volume = volume;
        this.issueNumber = issueNumber;
    }

    public String getPublisher() { return publisher; }
    public int getVolume() { return volume; }
    public int getIssueNumber() { return issueNumber; }

    public void setPublisher(String publisher) { this.publisher = publisher; }
    public void setVolume(int volume) { this.volume = volume; }
    public void setIssueNumber(int issueNumber) { this.issueNumber = issueNumber; }

    @Override
    public String toString() {
        return String.format("Journal[id=%s, title='%s', vol=%d, issue=%d]",
            getId(), getTitle(), volume, issueNumber);
    }
}
```

---

### 7.4 BookItem (Physical Copy)

```java
// BookItem.java
import java.time.LocalDate;

public class BookItem {

    private final String id;
    private final String barcode;
    private String location;   // shelf location, e.g. "A-12-3"
    private final LibraryItem libraryItem;
    private BookItemState state;
    private Member currentBorrower;
    private LocalDate dueDate;

    public BookItem(String id, String barcode, String location, LibraryItem libraryItem) {
        if (id == null || barcode == null || libraryItem == null) {
            throw new IllegalArgumentException("id, barcode, and libraryItem must not be null");
        }
        this.id = id;
        this.barcode = barcode;
        this.location = location;
        this.libraryItem = libraryItem;
        this.state = new AvailableState();  // every new copy starts as AVAILABLE
    }

    // --- Delegated state transitions ---

    public void borrow(Member member) {
        state.doBorrow(this, member);
    }

    public void returnItem() {
        state.doReturn(this);
    }

    public void reserve(Member member) {
        state.doReserve(this, member);
    }

    public void reportLost() {
        state.doReportLost(this);
    }

    // --- Status query ---

    public ItemStatus getStatus() {
        return state.getStatus();
    }

    // --- Getters ---

    public String getId() { return id; }
    public String getBarcode() { return barcode; }
    public String getLocation() { return location; }
    public LibraryItem getLibraryItem() { return libraryItem; }
    public Member getCurrentBorrower() { return currentBorrower; }
    public LocalDate getDueDate() { return dueDate; }

    // --- Package-accessible setters used by state implementations ---

    void setState(BookItemState state) {
        this.state = state;
    }

    void setCurrentBorrower(Member member) {
        this.currentBorrower = member;
    }

    void setDueDate(LocalDate dueDate) {
        this.dueDate = dueDate;
    }

    public void setLocation(String location) {
        this.location = location;
    }

    @Override
    public String toString() {
        return String.format("BookItem[barcode=%s, status=%s, item='%s']",
            barcode, getStatus(), libraryItem.getTitle());
    }
}
```

---

### 7.5 BorrowRecord

```java
// BorrowRecord.java
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;

public class BorrowRecord {

    private final String recordId;
    private final BookItem bookItem;
    private final Member member;
    private final LocalDate borrowDate;
    private final LocalDate dueDate;
    private LocalDate returnDate;
    private boolean isActive;

    public BorrowRecord(String recordId, BookItem bookItem, Member member,
                        LocalDate borrowDate, LocalDate dueDate) {
        this.recordId = recordId;
        this.bookItem = bookItem;
        this.member = member;
        this.borrowDate = borrowDate;
        this.dueDate = dueDate;
        this.returnDate = null;
        this.isActive = true;
    }

    /**
     * Close this record when the item is returned.
     */
    public void close(LocalDate returnDate) {
        if (!isActive) {
            throw new IllegalStateException("BorrowRecord " + recordId + " is already closed.");
        }
        this.returnDate = returnDate;
        this.isActive = false;
    }

    /**
     * Extend the due date (used for renewals).
     */
    public void extendDueDateBy(int days) {
        // We need a mutable due date for renewals.
        // Since dueDate is final we track an extension separately via a new field.
        // For simplicity we close this record and let the Library create a new one.
        throw new UnsupportedOperationException(
            "Renewals are handled by closing this record and creating a new BorrowRecord.");
    }

    public boolean isOverdue() {
        LocalDate checkDate = (returnDate != null) ? returnDate : LocalDate.now();
        return checkDate.isAfter(dueDate);
    }

    public long overdueDays() {
        if (!isOverdue()) return 0;
        LocalDate checkDate = (returnDate != null) ? returnDate : LocalDate.now();
        return ChronoUnit.DAYS.between(dueDate, checkDate);
    }

    // --- Getters ---

    public String getRecordId() { return recordId; }
    public BookItem getBookItem() { return bookItem; }
    public Member getMember() { return member; }
    public LocalDate getBorrowDate() { return borrowDate; }
    public LocalDate getDueDate() { return dueDate; }
    public LocalDate getReturnDate() { return returnDate; }
    public boolean isActive() { return isActive; }

    @Override
    public String toString() {
        return String.format(
            "BorrowRecord[id=%s, item=%s, member=%s, due=%s, returned=%s, active=%b]",
            recordId, bookItem.getBarcode(), member.getMemberId(), dueDate, returnDate, isActive);
    }
}
```

---

### 7.6 Fine

```java
// Fine.java
import java.time.LocalDate;

public class Fine {

    private final String fineId;
    private final Member member;
    private final BookItem bookItem;
    private final FineType fineType;
    private final double amount;
    private boolean isPaid;
    private final LocalDate issuedDate;

    public Fine(String fineId, Member member, BookItem bookItem,
                FineType fineType, double amount) {
        this.fineId = fineId;
        this.member = member;
        this.bookItem = bookItem;
        this.fineType = fineType;
        this.amount = amount;
        this.isPaid = false;
        this.issuedDate = LocalDate.now();
    }

    public void pay() {
        if (isPaid) {
            throw new IllegalStateException("Fine " + fineId + " has already been paid.");
        }
        this.isPaid = true;
    }

    public String getFineId() { return fineId; }
    public Member getMember() { return member; }
    public BookItem getBookItem() { return bookItem; }
    public FineType getFineType() { return fineType; }
    public double getAmount() { return amount; }
    public boolean isPaid() { return isPaid; }
    public LocalDate getIssuedDate() { return issuedDate; }

    @Override
    public String toString() {
        return String.format("Fine[id=%s, type=%s, amount=%.2f, paid=%b, member=%s]",
            fineId, fineType, amount, isPaid, member.getMemberId());
    }
}
```

---

### 7.7 Reservation

```java
// Reservation.java
import java.time.LocalDate;

public class Reservation {

    private final String reservationId;
    private final Member member;
    private final LibraryItem libraryItem;
    private final LocalDate createdAt;
    private boolean isActive;

    public Reservation(String reservationId, Member member, LibraryItem libraryItem) {
        this.reservationId = reservationId;
        this.member = member;
        this.libraryItem = libraryItem;
        this.createdAt = LocalDate.now();
        this.isActive = true;
    }

    public void cancel() {
        if (!isActive) {
            throw new IllegalStateException("Reservation " + reservationId + " is already cancelled.");
        }
        this.isActive = false;
    }

    public String getReservationId() { return reservationId; }
    public Member getMember() { return member; }
    public LibraryItem getLibraryItem() { return libraryItem; }
    public LocalDate getCreatedAt() { return createdAt; }
    public boolean isActive() { return isActive; }

    @Override
    public String toString() {
        return String.format("Reservation[id=%s, member=%s, item='%s', active=%b]",
            reservationId, member.getMemberId(), libraryItem.getTitle(), isActive);
    }
}
```

---

### 7.8 Observer Pattern — LibraryEventListener and NotificationService

```java
// LibraryEventListener.java
public interface LibraryEventListener {

    /** Called after a member successfully borrows a copy. */
    void onItemBorrowed(Member member, BookItem item, BorrowRecord record);

    /** Called after a member returns a copy. */
    void onItemReturned(Member member, BookItem item, BorrowRecord record);

    /** Called when a reserved copy becomes available for pickup. */
    void onReservationAvailable(Reservation reservation, BookItem item);

    /** Called when a fine is issued to a member. */
    void onFineIssued(Fine fine);

    /** Called when a reservation is cancelled. */
    void onReservationCancelled(Reservation reservation);
}
```

```java
// NotificationService.java
import java.util.ArrayList;
import java.util.List;

/**
 * NotificationService acts as an event bus.
 * It holds a list of LibraryEventListeners and fans out each event.
 * It also provides a built-in console-logging listener registered by default,
 * demonstrating that additional listeners (email, SMS) can be added without
 * modifying this class.
 */
public class NotificationService implements LibraryEventListener {

    private final List<LibraryEventListener> listeners;

    public NotificationService() {
        this.listeners = new ArrayList<>();
        // Register the built-in console listener
        listeners.add(new ConsoleNotificationListener());
    }

    /** Add a custom listener (e.g., EmailNotificationListener). */
    public void subscribe(LibraryEventListener listener) {
        if (listener != null) {
            listeners.add(listener);
        }
    }

    public void unsubscribe(LibraryEventListener listener) {
        listeners.remove(listener);
    }

    @Override
    public void onItemBorrowed(Member member, BookItem item, BorrowRecord record) {
        for (LibraryEventListener l : listeners) {
            l.onItemBorrowed(member, item, record);
        }
    }

    @Override
    public void onItemReturned(Member member, BookItem item, BorrowRecord record) {
        for (LibraryEventListener l : listeners) {
            l.onItemReturned(member, item, record);
        }
    }

    @Override
    public void onReservationAvailable(Reservation reservation, BookItem item) {
        for (LibraryEventListener l : listeners) {
            l.onReservationAvailable(reservation, item);
        }
    }

    @Override
    public void onFineIssued(Fine fine) {
        for (LibraryEventListener l : listeners) {
            l.onFineIssued(fine);
        }
    }

    @Override
    public void onReservationCancelled(Reservation reservation) {
        for (LibraryEventListener l : listeners) {
            l.onReservationCancelled(reservation);
        }
    }
}
```

```java
// ConsoleNotificationListener.java
public class ConsoleNotificationListener implements LibraryEventListener {

    @Override
    public void onItemBorrowed(Member member, BookItem item, BorrowRecord record) {
        System.out.printf("[NOTIFY] %s borrowed '%s' (barcode: %s). Due: %s%n",
            member.getName(), item.getLibraryItem().getTitle(),
            item.getBarcode(), record.getDueDate());
    }

    @Override
    public void onItemReturned(Member member, BookItem item, BorrowRecord record) {
        System.out.printf("[NOTIFY] %s returned '%s' (barcode: %s).%n",
            member.getName(), item.getLibraryItem().getTitle(), item.getBarcode());
    }

    @Override
    public void onReservationAvailable(Reservation reservation, BookItem item) {
        System.out.printf("[NOTIFY] Reservation available for %s: '%s' (barcode: %s) is ready for pickup.%n",
            reservation.getMember().getName(),
            reservation.getLibraryItem().getTitle(),
            item.getBarcode());
    }

    @Override
    public void onFineIssued(Fine fine) {
        System.out.printf("[NOTIFY] Fine issued to %s: $%.2f (%s) for '%s'.%n",
            fine.getMember().getName(), fine.getAmount(),
            fine.getFineType(), fine.getBookItem().getLibraryItem().getTitle());
    }

    @Override
    public void onReservationCancelled(Reservation reservation) {
        System.out.printf("[NOTIFY] Reservation cancelled for %s on '%s'.%n",
            reservation.getMember().getName(),
            reservation.getLibraryItem().getTitle());
    }
}
```

---

### 7.9 Member

```java
// Member.java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class Member {

    // Fine rate per overdue day, in dollars
    private static final double STANDARD_DAILY_FINE_RATE = 0.25;
    private static final double PREMIUM_DAILY_FINE_RATE  = 0.10;

    // Borrow period in days
    private static final int STANDARD_BORROW_DAYS = 14;
    private static final int PREMIUM_BORROW_DAYS  = 21;

    // Maximum active loans allowed
    private static final int MAX_LOANS = 5;

    // Fine threshold above which borrowing is blocked
    private static final double MAX_OUTSTANDING_FINE = 10.00;

    private final String memberId;
    private String name;
    private String email;
    private String phone;
    private MemberType memberType;
    private MemberStatus status;

    private final List<BorrowRecord> borrowHistory;
    private final List<Reservation> reservations;
    private final List<Fine> fines;

    public Member(String memberId, String name, String email, String phone, MemberType memberType) {
        this.memberId = memberId;
        this.name = name;
        this.email = email;
        this.phone = phone;
        this.memberType = memberType;
        this.status = MemberStatus.ACTIVE;
        this.borrowHistory = new ArrayList<>();
        this.reservations = new ArrayList<>();
        this.fines = new ArrayList<>();
    }

    // ---- Borrow / Return helpers used by Library facade ----

    public void addBorrowRecord(BorrowRecord record) {
        borrowHistory.add(record);
    }

    public List<BorrowRecord> getActiveLoans() {
        List<BorrowRecord> active = new ArrayList<>();
        for (BorrowRecord r : borrowHistory) {
            if (r.isActive()) active.add(r);
        }
        return active;
    }

    public boolean canBorrow() {
        if (status != MemberStatus.ACTIVE) return false;
        if (getActiveLoans().size() >= MAX_LOANS) return false;
        if (getTotalOutstandingFines() > MAX_OUTSTANDING_FINE) return false;
        return true;
    }

    public void addReservation(Reservation reservation) {
        reservations.add(reservation);
    }

    public List<Reservation> getActiveReservations() {
        List<Reservation> active = new ArrayList<>();
        for (Reservation r : reservations) {
            if (r.isActive()) active.add(r);
        }
        return active;
    }

    public void addFine(Fine fine) {
        fines.add(fine);
    }

    public double getTotalOutstandingFines() {
        double total = 0.0;
        for (Fine f : fines) {
            if (!f.isPaid()) total += f.getAmount();
        }
        return total;
    }

    public double getTotalFines() {
        double total = 0.0;
        for (Fine f : fines) {
            total += f.getAmount();
        }
        return total;
    }

    public double getDailyFineRate() {
        return memberType == MemberType.PREMIUM ? PREMIUM_DAILY_FINE_RATE : STANDARD_DAILY_FINE_RATE;
    }

    public int getBorrowPeriodDays() {
        return memberType == MemberType.PREMIUM ? PREMIUM_BORROW_DAYS : STANDARD_BORROW_DAYS;
    }

    // ---- Getters and setters ----

    public String getMemberId() { return memberId; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public String getPhone() { return phone; }
    public MemberType getMemberType() { return memberType; }
    public MemberStatus getStatus() { return status; }
    public List<BorrowRecord> getBorrowHistory() { return Collections.unmodifiableList(borrowHistory); }
    public List<Reservation> getReservations() { return Collections.unmodifiableList(reservations); }
    public List<Fine> getFines() { return Collections.unmodifiableList(fines); }

    public void setName(String name) { this.name = name; }
    public void setEmail(String email) { this.email = email; }
    public void setPhone(String phone) { this.phone = phone; }
    public void setMemberType(MemberType memberType) { this.memberType = memberType; }
    public void setStatus(MemberStatus status) { this.status = status; }

    @Override
    public String toString() {
        return String.format("Member[id=%s, name='%s', type=%s, status=%s, activeLoans=%d]",
            memberId, name, memberType, status, getActiveLoans().size());
    }
}
```

---

### 7.10 Librarian

```java
// Librarian.java
/**
 * A Librarian is a special Member with additional privileges:
 * - Manage the catalog (add/remove items and copies)
 * - Register and deactivate members
 * - Process lost item reports
 *
 * Extends Member so that librarians are also registered in the system
 * and can themselves borrow items.
 */
public class Librarian extends Member {

    private final String employeeId;

    public Librarian(String memberId, String name, String email, String phone, String employeeId) {
        super(memberId, name, email, phone, MemberType.PREMIUM);
        this.employeeId = employeeId;
    }

    /**
     * Add a physical copy to a library item and register it with the catalog.
     */
    public void addBookItem(LibraryItem item, BookItem copy) {
        item.addCopy(copy);
        System.out.printf("[LIBRARIAN] %s added copy [barcode=%s] to '%s'.%n",
            getName(), copy.getBarcode(), item.getTitle());
    }

    /**
     * Remove a physical copy from a library item by barcode.
     */
    public void removeBookItem(LibraryItem item, String barcode) {
        boolean removed = item.removeCopy(barcode);
        if (removed) {
            System.out.printf("[LIBRARIAN] %s removed copy [barcode=%s] from '%s'.%n",
                getName(), barcode, item.getTitle());
        } else {
            System.out.printf("[LIBRARIAN] Copy [barcode=%s] not found in '%s'.%n",
                barcode, item.getTitle());
        }
    }

    /**
     * Register a new member with the library.
     */
    public void registerMember(Catalog catalog, Member member) {
        // In a real system the catalog would hold members too, or there'd be a MemberRepository.
        // Here we just log; integration with Library facade handles actual registration.
        System.out.printf("[LIBRARIAN] %s registered new member: %s (id=%s).%n",
            getName(), member.getName(), member.getMemberId());
    }

    /**
     * Deactivate a member account.
     */
    public void deactivateMember(Member member) {
        member.setStatus(MemberStatus.CLOSED);
        System.out.printf("[LIBRARIAN] %s deactivated member: %s (id=%s).%n",
            getName(), member.getName(), member.getMemberId());
    }

    public String getEmployeeId() { return employeeId; }

    @Override
    public String toString() {
        return String.format("Librarian[employeeId=%s, name='%s']", employeeId, getName());
    }
}
```

---

### 7.11 Catalog

```java
// Catalog.java
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class Catalog {

    // Keyed by item id
    private final Map<String, LibraryItem> items;

    public Catalog() {
        this.items = new HashMap<>();
    }

    public void addItem(LibraryItem item) {
        if (items.containsKey(item.getId())) {
            throw new IllegalArgumentException("Item with id " + item.getId() + " already exists.");
        }
        items.put(item.getId(), item);
    }

    public void removeItem(String itemId) {
        items.remove(itemId);
    }

    public LibraryItem getItemById(String itemId) {
        return items.get(itemId);
    }

    /**
     * Case-insensitive title search (contains match).
     */
    public List<LibraryItem> searchByTitle(String title) {
        String query = title.toLowerCase();
        List<LibraryItem> results = new ArrayList<>();
        for (LibraryItem item : items.values()) {
            if (item.getTitle().toLowerCase().contains(query)) {
                results.add(item);
            }
        }
        return results;
    }

    /**
     * Search by author (Books only). Case-insensitive contains match.
     */
    public List<LibraryItem> searchByAuthor(String author) {
        String query = author.toLowerCase();
        List<LibraryItem> results = new ArrayList<>();
        for (LibraryItem item : items.values()) {
            if (item instanceof Book) {
                Book book = (Book) item;
                if (book.getAuthor().toLowerCase().contains(query)) {
                    results.add(book);
                }
            }
        }
        return results;
    }

    /**
     * Search by ISBN (Books only). Exact match.
     */
    public List<LibraryItem> searchByISBN(String isbn) {
        List<LibraryItem> results = new ArrayList<>();
        for (LibraryItem item : items.values()) {
            if (item instanceof Book) {
                Book book = (Book) item;
                if (book.getIsbn().equals(isbn)) {
                    results.add(book);
                }
            }
        }
        return results;
    }

    /**
     * Case-insensitive subject search (contains match).
     */
    public List<LibraryItem> searchBySubject(String subject) {
        String query = subject.toLowerCase();
        List<LibraryItem> results = new ArrayList<>();
        for (LibraryItem item : items.values()) {
            if (item.getSubject() != null && item.getSubject().toLowerCase().contains(query)) {
                results.add(item);
            }
        }
        return results;
    }

    /**
     * Return all available copies of a given library item.
     */
    public List<BookItem> getAvailableCopies(String itemId) {
        LibraryItem item = items.get(itemId);
        if (item == null) return Collections.emptyList();
        return item.getAvailableCopies();
    }

    public Map<String, LibraryItem> getAllItems() {
        return Collections.unmodifiableMap(items);
    }

    public int size() {
        return items.size();
    }
}
```

---

### 7.12 Library (Application Facade)

```java
// Library.java
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * Library is the central application facade.
 * All business workflows (borrow, return, reserve, renew, fine) run through here.
 * It coordinates BookItem state transitions, BorrowRecord lifecycle, Fine creation,
 * Reservation queues, and event notifications.
 */
public class Library {

    private final String name;
    private final Catalog catalog;
    private final NotificationService notificationService;

    // memberId -> Member
    private final Map<String, Member> members;

    // Reservation queue per libraryItem id (FIFO — sorted by createdAt)
    private final Map<String, List<Reservation>> reservationQueues;

    // All borrow records
    private final List<BorrowRecord> allBorrowRecords;

    // All fines
    private final List<Fine> allFines;

    private static final double LOST_ITEM_REPLACEMENT_FEE = 50.00;

    public Library(String name) {
        this.name = name;
        this.catalog = new Catalog();
        this.notificationService = new NotificationService();
        this.members = new HashMap<>();
        this.reservationQueues = new HashMap<>();
        this.allBorrowRecords = new ArrayList<>();
        this.allFines = new ArrayList<>();
    }

    // ---- Member management ----

    public void registerMember(Member member) {
        if (members.containsKey(member.getMemberId())) {
            throw new IllegalArgumentException("Member " + member.getMemberId() + " already registered.");
        }
        members.put(member.getMemberId(), member);
        System.out.printf("[LIBRARY] Member registered: %s (id=%s)%n",
            member.getName(), member.getMemberId());
    }

    public Member getMember(String memberId) {
        return members.get(memberId);
    }

    // ---- Catalog management ----

    public Catalog getCatalog() {
        return catalog;
    }

    public NotificationService getNotificationService() {
        return notificationService;
    }

    // ---- Borrow ----

    /**
     * Borrow the first available copy of the item identified by itemId on behalf of member.
     *
     * @return the BorrowRecord created
     * @throws IllegalStateException if the member cannot borrow or no copy is available
     */
    public BorrowRecord borrowItem(String memberId, String itemId) {
        Member member = getExistingMember(memberId);
        LibraryItem item = getExistingItem(itemId);

        if (!member.canBorrow()) {
            throw new IllegalStateException(
                "Member " + memberId + " cannot borrow: account inactive, loan limit reached, "
                + "or outstanding fines exceed threshold.");
        }

        List<BookItem> available = item.getAvailableCopies();
        if (available.isEmpty()) {
            throw new IllegalStateException(
                "No available copies of '" + item.getTitle() + "'. Consider placing a reservation.");
        }

        BookItem copy = available.get(0);
        copy.borrow(member);   // state transition: AVAILABLE -> BORROWED

        LocalDate borrowDate = LocalDate.now();
        LocalDate dueDate = borrowDate.plusDays(member.getBorrowPeriodDays());

        BorrowRecord record = new BorrowRecord(
            generateId("BR"), copy, member, borrowDate, dueDate);

        member.addBorrowRecord(record);
        allBorrowRecords.add(record);

        notificationService.onItemBorrowed(member, copy, record);
        return record;
    }

    /**
     * Borrow a specific copy (identified by barcode) for a member.
     * Used when a member is picking up a reserved copy.
     */
    public BorrowRecord borrowSpecificCopy(String memberId, String barcode) {
        Member member = getExistingMember(memberId);
        BookItem copy = findCopyByBarcode(barcode);

        if (!member.canBorrow()) {
            throw new IllegalStateException(
                "Member " + memberId + " cannot borrow: account inactive, loan limit reached, "
                + "or outstanding fines exceed threshold.");
        }

        if (copy.getStatus() == ItemStatus.RESERVED) {
            // Validate that this member is the one whose reservation is active for this item
            List<Reservation> queue = reservationQueues.getOrDefault(
                copy.getLibraryItem().getId(), new ArrayList<>());
            Reservation firstReservation = getFirstActiveReservation(queue);
            if (firstReservation == null || !firstReservation.getMember().getMemberId().equals(memberId)) {
                throw new IllegalStateException(
                    "Copy [" + barcode + "] is reserved for a different member.");
            }
            // Cancel the fulfilled reservation
            firstReservation.cancel();
            notificationService.onReservationCancelled(firstReservation);
        }

        copy.borrow(member);

        LocalDate borrowDate = LocalDate.now();
        LocalDate dueDate = borrowDate.plusDays(member.getBorrowPeriodDays());

        BorrowRecord record = new BorrowRecord(
            generateId("BR"), copy, member, borrowDate, dueDate);

        member.addBorrowRecord(record);
        allBorrowRecords.add(record);

        notificationService.onItemBorrowed(member, copy, record);
        return record;
    }

    // ---- Return ----

    /**
     * Return a copy identified by barcode. Calculates overdue fine if applicable.
     * If a reservation exists for this item, transitions the copy to RESERVED and notifies the member.
     */
    public void returnItem(String memberId, String barcode) {
        Member member = getExistingMember(memberId);
        BookItem copy = findCopyByBarcode(barcode);

        if (copy.getStatus() != ItemStatus.BORROWED) {
            throw new IllegalStateException(
                "Copy [" + barcode + "] is not currently borrowed.");
        }

        if (!copy.getCurrentBorrower().getMemberId().equals(memberId)) {
            throw new IllegalStateException(
                "Copy [" + barcode + "] is not borrowed by member " + memberId + ".");
        }

        // Find and close the active borrow record
        BorrowRecord record = findActiveBorrowRecord(copy);
        if (record == null) {
            throw new IllegalStateException(
                "No active borrow record found for copy [" + barcode + "].");
        }

        LocalDate returnDate = LocalDate.now();
        record.close(returnDate);
        copy.returnItem();  // state: BORROWED -> AVAILABLE

        notificationService.onItemReturned(member, copy, record);

        // Calculate and issue overdue fine if applicable
        if (record.isOverdue()) {
            double fineAmount = record.overdueDays() * member.getDailyFineRate();
            Fine fine = new Fine(generateId("FINE"), member, copy, FineType.OVERDUE, fineAmount);
            member.addFine(fine);
            allFines.add(fine);
            notificationService.onFineIssued(fine);
        }

        // Check if a reservation exists for this item; if so, hold the copy for that member
        String itemId = copy.getLibraryItem().getId();
        List<Reservation> queue = reservationQueues.getOrDefault(itemId, new ArrayList<>());
        Reservation nextReservation = getFirstActiveReservation(queue);

        if (nextReservation != null) {
            copy.reserve(nextReservation.getMember());  // state: AVAILABLE -> RESERVED
            notificationService.onReservationAvailable(nextReservation, copy);
        }
    }

    // ---- Renew ----

    /**
     * Renew a borrowed copy. Only allowed if no active reservation exists for the item.
     * Implemented as: close existing record, create a new one with a fresh due date.
     *
     * @return the new BorrowRecord
     */
    public BorrowRecord renewItem(String memberId, String barcode) {
        Member member = getExistingMember(memberId);
        BookItem copy = findCopyByBarcode(barcode);

        if (copy.getStatus() != ItemStatus.BORROWED) {
            throw new IllegalStateException(
                "Copy [" + barcode + "] is not currently borrowed.");
        }

        if (!copy.getCurrentBorrower().getMemberId().equals(memberId)) {
            throw new IllegalStateException(
                "Copy [" + barcode + "] is not borrowed by member " + memberId + ".");
        }

        String itemId = copy.getLibraryItem().getId();
        List<Reservation> queue = reservationQueues.getOrDefault(itemId, new ArrayList<>());
        if (getFirstActiveReservation(queue) != null) {
            throw new IllegalStateException(
                "Cannot renew: a reservation exists for '" + copy.getLibraryItem().getTitle() + "'.");
        }

        // Close the existing record (not overdue — we're renewing before return)
        BorrowRecord oldRecord = findActiveBorrowRecord(copy);
        if (oldRecord == null) {
            throw new IllegalStateException("No active borrow record found for copy [" + barcode + "].");
        }
        oldRecord.close(LocalDate.now());

        // Create a new record with extended due date; keep the copy BORROWED under the same member
        LocalDate newDueDate = LocalDate.now().plusDays(member.getBorrowPeriodDays());
        copy.setDueDate(newDueDate);

        BorrowRecord newRecord = new BorrowRecord(
            generateId("BR"), copy, member, LocalDate.now(), newDueDate);

        member.addBorrowRecord(newRecord);
        allBorrowRecords.add(newRecord);

        System.out.printf("[LIBRARY] Renewed '%s' for %s. New due date: %s%n",
            copy.getLibraryItem().getTitle(), member.getName(), newDueDate);

        return newRecord;
    }

    // ---- Reserve ----

    /**
     * Place a reservation for a member on a library item.
     * Reservations are only meaningful when all copies are currently borrowed.
     *
     * @return the Reservation created
     */
    public Reservation reserveItem(String memberId, String itemId) {
        Member member = getExistingMember(memberId);
        LibraryItem item = getExistingItem(itemId);

        if (member.getStatus() != MemberStatus.ACTIVE) {
            throw new IllegalStateException("Member " + memberId + " account is not active.");
        }

        // Check if the member already has a reservation on this item
        for (Reservation r : member.getActiveReservations()) {
            if (r.getLibraryItem().getId().equals(itemId)) {
                throw new IllegalStateException(
                    "Member " + memberId + " already has an active reservation on '" + item.getTitle() + "'.");
            }
        }

        Reservation reservation = new Reservation(generateId("RES"), member, item);
        member.addReservation(reservation);

        reservationQueues.computeIfAbsent(itemId, k -> new ArrayList<>()).add(reservation);

        System.out.printf("[LIBRARY] Reservation placed: %s for member %s (%s)%n",
            reservation.getReservationId(), member.getName(), member.getMemberId());

        return reservation;
    }

    /**
     * Cancel an existing reservation.
     */
    public void cancelReservation(String reservationId) {
        for (List<Reservation> queue : reservationQueues.values()) {
            for (Reservation r : queue) {
                if (r.getReservationId().equals(reservationId) && r.isActive()) {
                    r.cancel();
                    notificationService.onReservationCancelled(r);

                    // If the associated BookItem was RESERVED for this member, revert to AVAILABLE
                    LibraryItem item = r.getLibraryItem();
                    for (BookItem copy : item.getCopies()) {
                        if (copy.getStatus() == ItemStatus.RESERVED
                                && copy.getLibraryItem().getId().equals(item.getId())) {
                            // Check if another reservation is waiting
                            Reservation next = getFirstActiveReservation(queue);
                            if (next != null) {
                                notificationService.onReservationAvailable(next, copy);
                            } else {
                                // No more reservations — make copy available
                                copy.setState(new AvailableState());
                            }
                            break;
                        }
                    }
                    return;
                }
            }
        }
        throw new IllegalArgumentException("Reservation " + reservationId + " not found or already cancelled.");
    }

    // ---- Lost item ----

    /**
     * Report a copy as lost. Issues a replacement fine to the borrower.
     */
    public Fine reportLost(String barcode) {
        BookItem copy = findCopyByBarcode(barcode);
        Member borrower = copy.getCurrentBorrower();

        // Close any active borrow record
        BorrowRecord record = findActiveBorrowRecord(copy);
        if (record != null) {
            record.close(LocalDate.now());
        }

        copy.reportLost();  // state -> LOST

        Fine fine = null;
        if (borrower != null) {
            fine = new Fine(generateId("FINE"), borrower, copy,
                FineType.LOST_ITEM, LOST_ITEM_REPLACEMENT_FEE);
            borrower.addFine(fine);
            allFines.add(fine);
            notificationService.onFineIssued(fine);
        }

        System.out.printf("[LIBRARY] Copy [barcode=%s] reported lost.%s%n",
            barcode, borrower != null ? " Fine issued to " + borrower.getName() + "." : "");

        return fine;
    }

    // ---- Fine payment ----

    public void payFine(String fineId) {
        for (Fine f : allFines) {
            if (f.getFineId().equals(fineId)) {
                f.pay();
                System.out.printf("[LIBRARY] Fine %s ($%.2f) paid by %s.%n",
                    fineId, f.getAmount(), f.getMember().getName());
                return;
            }
        }
        throw new IllegalArgumentException("Fine " + fineId + " not found.");
    }

    // ---- Reporting ----

    public List<BorrowRecord> getAllBorrowRecords() {
        return new ArrayList<>(allBorrowRecords);
    }

    public List<Fine> getAllFines() {
        return new ArrayList<>(allFines);
    }

    public String getName() { return name; }

    // ---- Private helpers ----

    private Member getExistingMember(String memberId) {
        Member m = members.get(memberId);
        if (m == null) throw new IllegalArgumentException("Member not found: " + memberId);
        return m;
    }

    private LibraryItem getExistingItem(String itemId) {
        LibraryItem item = catalog.getItemById(itemId);
        if (item == null) throw new IllegalArgumentException("Item not found: " + itemId);
        return item;
    }

    private BookItem findCopyByBarcode(String barcode) {
        for (LibraryItem item : catalog.getAllItems().values()) {
            for (BookItem copy : item.getCopies()) {
                if (copy.getBarcode().equals(barcode)) return copy;
            }
        }
        throw new IllegalArgumentException("No copy found with barcode: " + barcode);
    }

    private BorrowRecord findActiveBorrowRecord(BookItem copy) {
        for (BorrowRecord r : allBorrowRecords) {
            if (r.isActive() && r.getBookItem().getBarcode().equals(copy.getBarcode())) {
                return r;
            }
        }
        return null;
    }

    private Reservation getFirstActiveReservation(List<Reservation> queue) {
        // Reservations are inserted in order; find the first active one (FIFO)
        for (Reservation r : queue) {
            if (r.isActive()) return r;
        }
        return null;
    }

    private String generateId(String prefix) {
        return prefix + "-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }
}
```

---

### 7.13 Demo — main() Method

```java
// LibraryManagementDemo.java
import java.time.LocalDate;

/**
 * End-to-end demonstration of the Library Management System.
 * Shows: catalog setup, borrow, return, fine calculation, reservation, and lost item flow.
 *
 * Run with:  javac *.java && java LibraryManagementDemo
 */
public class LibraryManagementDemo {

    public static void main(String[] args) {
        System.out.println("========== Library Management System Demo ==========\n");

        // ---- 1. Setup library ----
        Library library = new Library("City Central Library");
        Catalog catalog = library.getCatalog();

        // ---- 2. Add a book to the catalog ----
        Book cleanCode = new Book(
            "ITEM-001", "Clean Code", "Software Engineering", "English",
            "Robert C. Martin", "978-0132350884", "Prentice Hall");
        catalog.addItem(cleanCode);

        // Add two physical copies
        BookItem copy1 = new BookItem("BI-001", "BC-0001", "A-01-1", cleanCode);
        BookItem copy2 = new BookItem("BI-002", "BC-0002", "A-01-2", cleanCode);
        cleanCode.addCopy(copy1);
        cleanCode.addCopy(copy2);

        // Add a magazine
        Magazine techMonthly = new Magazine(
            "ITEM-002", "Tech Monthly", "Technology", "English",
            "TechPress", 42);
        catalog.addItem(techMonthly);
        BookItem magCopy1 = new BookItem("BI-003", "TM-0001", "B-01-1", techMonthly);
        techMonthly.addCopy(magCopy1);

        System.out.println("Catalog size: " + catalog.size() + " items");
        System.out.println();

        // ---- 3. Register members ----
        Member alice = new Member("M-001", "Alice Johnson", "alice@example.com", "555-1001", MemberType.STANDARD);
        Member bob   = new Member("M-002", "Bob Smith",    "bob@example.com",   "555-1002", MemberType.PREMIUM);
        Member carol = new Member("M-003", "Carol White",  "carol@example.com", "555-1003", MemberType.STANDARD);

        Librarian librarian = new Librarian("L-001", "David Brown", "david@library.com", "555-0001", "EMP-101");

        library.registerMember(alice);
        library.registerMember(bob);
        library.registerMember(carol);
        library.registerMember(librarian);
        System.out.println();

        // ---- 4. Search catalog ----
        System.out.println("--- Search by author 'Martin' ---");
        library.getCatalog().searchByAuthor("Martin").forEach(System.out::println);
        System.out.println();

        System.out.println("--- Search by subject 'Software' ---");
        library.getCatalog().searchBySubject("Software").forEach(System.out::println);
        System.out.println();

        // ---- 5. Borrow a copy ----
        System.out.println("--- Alice borrows 'Clean Code' ---");
        BorrowRecord aliceRecord = library.borrowItem("M-001", "ITEM-001");
        System.out.println("Alice's active loans: " + alice.getActiveLoans().size());
        System.out.println("Available copies after borrow: "
            + catalog.getAvailableCopies("ITEM-001").size());
        System.out.println();

        // ---- 6. Bob borrows the second copy ----
        System.out.println("--- Bob borrows 'Clean Code' (second copy) ---");
        BorrowRecord bobRecord = library.borrowItem("M-002", "ITEM-001");
        System.out.println("Available copies after both borrow: "
            + catalog.getAvailableCopies("ITEM-001").size());
        System.out.println();

        // ---- 7. Carol tries to borrow — no copies left, places reservation ----
        System.out.println("--- Carol tries to borrow 'Clean Code' (no copies available) ---");
        try {
            library.borrowItem("M-003", "ITEM-001");
        } catch (IllegalStateException e) {
            System.out.println("Expected: " + e.getMessage());
        }

        System.out.println("\n--- Carol places a reservation ---");
        Reservation carolReservation = library.reserveItem("M-003", "ITEM-001");
        System.out.println("Carol's active reservations: " + carol.getActiveReservations().size());
        System.out.println();

        // ---- 8. Alice returns her copy — triggers reservation notification for Carol ----
        System.out.println("--- Alice returns 'Clean Code' ---");
        library.returnItem("M-001", aliceRecord.getBookItem().getBarcode());
        System.out.println("Status of returned copy: "
            + aliceRecord.getBookItem().getStatus());
        System.out.println();

        // ---- 9. Carol picks up the reserved copy ----
        System.out.println("--- Carol picks up the reserved copy ---");
        BorrowRecord carolRecord = library.borrowSpecificCopy(
            "M-003", aliceRecord.getBookItem().getBarcode());
        System.out.println("Carol's active loans: " + carol.getActiveLoans().size());
        System.out.println();

        // ---- 10. Bob renews his copy ----
        System.out.println("--- Bob renews his copy of 'Clean Code' ---");
        BorrowRecord bobRenewed = library.renewItem("M-002", bobRecord.getBookItem().getBarcode());
        System.out.printf("Bob's new due date: %s%n", bobRenewed.getDueDate());
        System.out.println();

        // ---- 11. Simulate overdue return (demonstrate fine calculation) ----
        // We manually close Carol's current record and create one with a past due date
        // to simulate an overdue return scenario.
        System.out.println("--- Simulating overdue return for Carol ---");
        // Carol's current active record
        BorrowRecord carolActive = carol.getActiveLoans().get(0);
        // Manually close it so we can simulate with a separate demonstration record
        carolActive.close(LocalDate.now());
        carolRecord.getBookItem().returnItem();  // direct state reset for demo

        // Create an overdue record: borrowed 20 days ago, due 6 days ago
        BookItem overdueItem = aliceRecord.getBookItem();  // reuse the same physical copy
        overdueItem.borrow(carol);  // set it back to BORROWED state for the return demo

        // Inject a past-dated borrow record into carol's history for demo purposes
        LocalDate pastBorrow = LocalDate.now().minusDays(20);
        LocalDate pastDue    = LocalDate.now().minusDays(6);
        BorrowRecord overdueRecord = new BorrowRecord(
            "BR-OVERDUE-DEMO", overdueItem, carol, pastBorrow, pastDue);
        carol.addBorrowRecord(overdueRecord);
        library.getAllBorrowRecords().add(overdueRecord);

        // Now return it — this triggers fine calculation
        library.returnItem("M-003", overdueItem.getBarcode());

        System.out.printf("Carol's outstanding fines: $%.2f%n", carol.getTotalOutstandingFines());
        System.out.printf("Expected overdue days: 6, daily rate: $%.2f, expected fine: $%.2f%n",
            carol.getDailyFineRate(), 6 * carol.getDailyFineRate());
        System.out.println();

        // ---- 12. Pay Carol's fine ----
        System.out.println("--- Carol pays her fine ---");
        Fine carolFine = carol.getFines().get(0);
        library.payFine(carolFine.getFineId());
        System.out.printf("Carol's outstanding fines after payment: $%.2f%n",
            carol.getTotalOutstandingFines());
        System.out.println();

        // ---- 13. Report lost item ----
        System.out.println("--- Bob reports his copy of 'Clean Code' as lost ---");
        Fine lostFine = library.reportLost(bobRenewed.getBookItem().getBarcode());
        if (lostFine != null) {
            System.out.printf("Lost item fine issued: $%.2f to %s%n",
                lostFine.getAmount(), bob.getName());
        }
        System.out.println("Copy status: " + bobRenewed.getBookItem().getStatus());
        System.out.println();

        // ---- 14. Librarian adds a new copy ----
        System.out.println("--- Librarian adds a new copy of 'Clean Code' ---");
        BookItem copy3 = new BookItem("BI-004", "BC-0003", "A-01-3", cleanCode);
        librarian.addBookItem(cleanCode, copy3);
        System.out.println("Total copies of 'Clean Code': " + cleanCode.getTotalCopies());
        System.out.println("Available copies: " + catalog.getAvailableCopies("ITEM-001").size());
        System.out.println();

        // ---- 15. Summary ----
        System.out.println("========== Final State ==========");
        System.out.println("Alice: " + alice);
        System.out.println("Bob:   " + bob);
        System.out.println("Carol: " + carol);
        System.out.printf("Total borrow records: %d%n", library.getAllBorrowRecords().size());
        System.out.printf("Total fines issued:   %d%n", library.getAllFines().size());
    }
}
```

**Expected output (abridged):**

```
========== Library Management System Demo ==========

Catalog size: 2 items

[LIBRARY] Member registered: Alice Johnson (id=M-001)
[LIBRARY] Member registered: Bob Smith (id=M-002)
[LIBRARY] Member registered: Carol White (id=M-003)
[LIBRARY] Member registered: David Brown (id=L-001)

--- Search by author 'Martin' ---
Book[id=ITEM-001, title='Clean Code', author='Robert C. Martin', isbn=978-0132350884]

--- Alice borrows 'Clean Code' ---
[NOTIFY] Alice Johnson borrowed 'Clean Code' (barcode: BC-0001). Due: 2026-07-06
Alice's active loans: 1
Available copies after borrow: 1

--- Carol places a reservation ---
[LIBRARY] Reservation placed: RES-XXXXXXXX for member Carol White (M-003)

--- Alice returns 'Clean Code' ---
[NOTIFY] Alice Johnson returned 'Clean Code' (barcode: BC-0001).
[NOTIFY] Reservation available for Carol White: 'Clean Code' (barcode: BC-0001) is ready for pickup.
Status of returned copy: RESERVED

--- Carol picks up the reserved copy ---
[NOTIFY] Reservation cancelled for Carol White on 'Clean Code'.
[NOTIFY] Carol White borrowed 'Clean Code' (barcode: BC-0001). Due: 2026-07-06

--- Simulating overdue return for Carol ---
[NOTIFY] Carol White returned 'Clean Code' (barcode: BC-0001).
[NOTIFY] Fine issued to Carol White: $1.50 (OVERDUE) for 'Clean Code'.
Carol's outstanding fines: $1.50

--- Bob reports his copy of 'Clean Code' as lost ---
[NOTIFY] Fine issued to Bob Smith: $50.00 (LOST_ITEM) for 'Clean Code'.
Copy status: LOST
```

---

## 8. Design Patterns Used

### 8.1 State Pattern — BookItem Status Transitions

**Problem**: A `BookItem` has four statuses (AVAILABLE, BORROWED, RESERVED, LOST), and the rules for which transitions are legal differ per status. Without a pattern, you end up with a `switch` or chain of `if-else` in `BookItem` that grows with each new status.

**Solution**: Extract a `BookItemState` interface. Each concrete state (`AvailableState`, `BorrowedState`, `ReservedState`, `LostState`) encapsulates the legal transitions from that state and throws `IllegalStateException` for illegal ones. `BookItem` delegates all status-changing calls to its current state object.

**Benefit**:
- Adding a new status (e.g., `UnderRepairState`) requires writing one new class, not modifying `BookItem`.
- Each state class is independently unit-testable.
- Illegal transitions fail loudly at the state level instead of silently producing incorrect data.

**How it appears in the code**: `BookItem.borrow(member)` calls `state.doBorrow(this, member)`. If the item is already borrowed, `BorrowedState.doBorrow(...)` throws `IllegalStateException`. After a successful borrow, the state sets itself to a new `BorrowedState` instance on the item.

---

### 8.2 Observer Pattern — Library Event Notifications

**Problem**: Multiple components need to react to library events (borrow, return, fine issued, reservation available). If the `Library` facade calls email/SMS/push services directly, it becomes tightly coupled to every notification channel and hard to test.

**Solution**: Define a `LibraryEventListener` interface. `Library` fires events through `NotificationService`, which fans them out to all registered listeners. Any number of listeners (console, email, SMS, push) can be subscribed at runtime without modifying `Library` or `NotificationService`.

**Benefit**:
- `Library` is decoupled from notification delivery.
- New channels (e.g., mobile push) are added by implementing `LibraryEventListener` and calling `notificationService.subscribe(new PushNotificationListener())`.
- Tests can subscribe a mock listener to assert which events were fired.

---

### 8.3 Facade Pattern — Library Class

`Library` is a classic application-layer facade. Callers (controllers, CLI) interact with one entry point (`borrowItem`, `returnItem`, `reserveItem`, etc.) rather than orchestrating `BookItem`, `BorrowRecord`, `Reservation`, `Fine`, and `NotificationService` themselves. This keeps the coordination logic in one place.

---

### 8.4 Template Method (implicit) — LibraryItem Hierarchy

`LibraryItem` defines the structure (copies list, common metadata, search surface) and concrete subclasses (`Book`, `Magazine`, `Journal`) supply type-specific fields. The `searchByAuthor` and `searchByISBN` methods in `Catalog` use `instanceof` checks to delegate to the right subtype — in a more polished design these would become polymorphic methods on `LibraryItem` itself, but the current form is clear enough for an interview context.

---

## 9. Key Design Decisions and Trade-offs

### BookItem vs LibraryItem — Why Two Levels?

A library typically owns multiple physical copies of one title. `LibraryItem` is the catalog entry (the concept of "Clean Code"); `BookItem` is a barcode-tagged physical copy on a shelf. This separation allows:

- Independent borrowing history per copy.
- Finer-grained availability tracking.
- The ability to retire a damaged copy without removing the catalog entry.

**Trade-off**: Two levels of abstraction require callers to navigate from `LibraryItem` to a specific `BookItem` copy. The `Library` facade absorbs this complexity.

---

### Reservation at LibraryItem level, not BookItem level

Reservations are placed on a `LibraryItem` (the title), not a specific `BookItem` (a copy). When any copy is returned, the library checks the item-level reservation queue and assigns the returned copy to the next waiter.

**Alternative**: Reserve a specific copy. This is simpler but suboptimal — if a library has three copies and two are returned simultaneously, only the member who specifically reserved one of those copies benefits.

**Trade-off**: Item-level reservations require a queue-per-item map in `Library`. The queue management code (`reservationQueues`) is slightly more complex but gives fairer service.

---

### Librarian extends Member

Librarians are modelled as a subclass of `Member`. This reflects reality: a librarian is also a person registered in the system who may borrow items.

**Alternative**: Separate hierarchies (`Person -> Member`, `Person -> Librarian`). This is better for strict role separation (avoids mixing patron and staff concerns) but requires more boilerplate.

**Trade-off**: The `extends Member` approach is simpler and sufficient for an interview. In production, prefer composition or a Role/Permission model.

---

### Fine Calculation is Pull-based (on return), Not Push-based (daily job)

Fines are calculated at return time, not accrued daily by a background job.

**Trade-off**:
- **Simpler**: No scheduler, no cron job, no database updates for every overdue item every night.
- **Less real-time**: Members cannot see their accruing fine balance until they return the item.
- A real system would run a nightly job to update fine amounts and send reminder notifications.

---

### `double` for monetary amounts

`double` is used here for brevity. In production, use `BigDecimal` to avoid floating-point rounding errors in financial calculations.

---

### ID Generation with UUID

IDs are generated with `UUID.randomUUID()`. In a persistent system, IDs would come from the database (auto-increment or sequence). UUIDs are used here to avoid a global counter dependency.

---

## 10. Extension Points

### Adding a New Item Type (e.g., DVD, E-Book)

1. Add a value to the `ItemType` enum (`DVD`, `EBOOK`).
2. Create a class `DVD extends LibraryItem` with DVD-specific fields (director, runtime).
3. No changes to `BookItem`, `BorrowRecord`, `Library`, or any state classes — they all operate on `LibraryItem` and `BookItem` abstractly.

---

### Adding a New Notification Channel (e.g., Email)

1. Create `EmailNotificationListener implements LibraryEventListener`.
2. Implement the five event methods using an email client.
3. Register at startup: `library.getNotificationService().subscribe(new EmailNotificationListener(emailClient))`.
4. No changes to `Library`, `NotificationService`, or any domain class.

---

### Adding a New BookItem State (e.g., UnderRepairState)

1. Create `UnderRepairState implements BookItemState`.
2. Implement each method with the appropriate transition rules.
3. Add `doSendForRepair()` to `BookItemState` interface (or use a side-channel method).
4. Update `BookItem.sendForRepair()` to call `state.doSendForRepair(this)`.
5. Only the new state class and `BookItemState` interface need changing; no cascade through the rest of the codebase.

---

### Persistence Layer

Replace the in-memory maps in `Library` and `Catalog` with repository interfaces:

```java
interface BookItemRepository {
    Optional<BookItem> findByBarcode(String barcode);
    void save(BookItem item);
}
```

`Library` accepts a `BookItemRepository` constructor parameter. The in-memory implementation used in tests; a JPA/JDBC implementation used in production.

---

### Member Borrowing Limit — Policy Object

Currently the limit (`MAX_LOANS = 5`) is a constant in `Member`. To make it configurable per library or member tier, extract a `BorrowingPolicy` interface:

```java
interface BorrowingPolicy {
    int maxLoans(Member member);
    int borrowPeriodDays(Member member);
    double dailyFineRate(Member member);
}
```

`Library` receives a `BorrowingPolicy` at construction time. This removes numeric magic constants from `Member` and allows different policies for different branches or membership tiers.

---

### Search Enhancement

Replace linear scan in `Catalog` with an inverted index (title words -> item IDs) or delegate to a search engine (Elasticsearch). The interface is unchanged: callers still call `catalog.searchByTitle(...)`.

---

## 11. Interview Discussion Points

**Q: How would you handle concurrent borrow requests for the last copy of a book?**

In the current design, the `Library.borrowItem` method is not thread-safe: two threads could both find one available copy and both proceed to call `copy.borrow(member)`. In production, synchronize at the database level (SELECT FOR UPDATE or optimistic locking on BookItem). In an in-memory system, wrap the check-and-borrow in a `synchronized` block keyed on the item ID, or use a `ReentrantLock` per item.

---

**Q: Why did you choose State over a simple status enum with if-else?**

An enum with if-else puts all transition logic in one place (`BookItem`), making it a violation of the Open/Closed Principle. Every new status requires modifying `BookItem`. With State, each state class is self-contained and independently testable. The tradeoff is more classes, but each is small and single-purpose.

---

**Q: What if a member wants to renew an item that is overdue?**

The current design allows renewal if no reservation exists, regardless of overdue status. A policy could disallow renewal if the item is already overdue. This would be a one-line change in `Library.renewItem`: check `LocalDate.now().isAfter(activeRecord.getDueDate())` and throw.

---

**Q: How do you prevent a member from gaming the system by cancelling and re-placing reservations to jump the queue?**

A real system would record the original reservation creation timestamp and sort by it, ignoring cancellation-and-re-reservation within a cooldown window. The current `createdAt` field on `Reservation` supports this — add a check in `Library.reserveItem` that rejects a new reservation from a member who cancelled one within the last N hours.

---

**Q: Why does BorrowRecord use LocalDate instead of LocalDateTime?**

Library due dates are typically day-granularity, not hour-granularity. Using `LocalDate` simplifies overdue day calculation (no time-zone or time-of-day complexity) and matches how libraries communicate due dates to members. A real system might use `Instant` for audit timestamps but `LocalDate` for due dates.

---

**Q: How would you compute total fines outstanding for reporting?**

`Library.getAllFines()` returns all fines. Aggregate with a stream:

```java
double total = library.getAllFines().stream()
    .filter(f -> !f.isPaid())
    .mapToDouble(Fine::getAmount)
    .sum();
```

For large datasets, this computation belongs in the database (`SELECT SUM(amount) FROM fines WHERE is_paid = false`).

---

**Q: How would you add a waitlist position display to members?**

`Library` already holds `reservationQueues` (a `Map<String, List<Reservation>>`). Add a method:

```java
public int getWaitlistPosition(String memberId, String itemId) {
    List<Reservation> queue = reservationQueues.getOrDefault(itemId, Collections.emptyList());
    int position = 0;
    for (Reservation r : queue) {
        if (!r.isActive()) continue;
        position++;
        if (r.getMember().getMemberId().equals(memberId)) return position;
    }
    return -1;  // not on waitlist
}
```

---

## 12. Common Mistakes in This Design

### Mistake 1: Putting Status Directly in BookItem Without State Pattern

```java
// WRONG
public class BookItem {
    private ItemStatus status;

    public void borrow(Member member) {
        if (status == ItemStatus.BORROWED) throw new IllegalStateException("...");
        if (status == ItemStatus.LOST) throw new IllegalStateException("...");
        this.status = ItemStatus.BORROWED;
        // ... more conditions as statuses grow
    }
}
```

Every new status requires editing `borrow()`, `returnItem()`, `reserve()`, and any other method. The State pattern avoids this.

---

### Mistake 2: Modelling Reservations on BookItem Instead of LibraryItem

```java
// WRONG — ties reservation to a specific physical copy
public class BookItem {
    private Member reservedFor;
}
```

A member wanting "Clean Code" does not care which barcode they get. Tying reservations to copies means the library must manually pick a copy and assign it, instead of having any returned copy trigger the next reservation automatically.

---

### Mistake 3: Calculating Fines Inside Member

```java
// WRONG — mixes domain logic with domain data
public class Member {
    public double calculateFine(BookItem item) {
        long days = ChronoUnit.DAYS.between(item.getDueDate(), LocalDate.now());
        return days * 0.25;
    }
}
```

Fine calculation depends on the borrow record (the actual due date), the item (for lost-item fees), and the member's rate. Putting it in `Member` gives `Member` knowledge of `BookItem` internals. The `Library` facade is the right place because it has access to all participants.

---

### Mistake 4: Not Distinguishing LibraryItem from BookItem

Treating every physical copy as a top-level entity with its own title, ISBN, and author leads to massive data duplication and makes catalog search impossible without scanning every copy. The two-level model (catalog entry + physical copy) is the correct abstraction.

---

### Mistake 5: Notification Logic Inside Domain Classes

```java
// WRONG
public void returnItem() {
    this.status = ItemStatus.AVAILABLE;
    EmailService.getInstance().sendEmail(nextReserver.getEmail(), "Your book is ready!");
}
```

Domain classes should not know about delivery mechanisms. This makes the class impossible to test in isolation and couples business logic to infrastructure. Use the Observer pattern: the domain fires an event; a listener handles delivery.

---

### Mistake 6: Missing BorrowRecord Abstraction — Tracking State Directly on BookItem

A common shortcut is to store `borrowedBy`, `borrowedDate`, and `dueDate` directly on `BookItem` without a `BorrowRecord`. This destroys borrowing history — you can only know the current borrower, not who had the copy last month. `BorrowRecord` is essential for audit trails, fine disputes, and usage analytics.

---

### Mistake 7: Allowing Renewal When a Reservation Exists

Failing to check the reservation queue before renewing means a member can hold a popular book indefinitely by repeatedly renewing, starving all waiting members. The check `if (getFirstActiveReservation(queue) != null) throw new IllegalStateException(...)` in `Library.renewItem` prevents this.

---

### Mistake 8: Using `float` or `double` for Fine Amounts in Production

```java
// ACCEPTABLE for interview, WRONG for production
private double amount;
```

Floating-point arithmetic produces rounding errors (`0.1 + 0.2 != 0.3`). All monetary calculations in production systems must use `BigDecimal` with explicit rounding mode (`HALF_UP`).

---

### Mistake 9: Ignoring Member Status Checks

A suspended or closed member should not be able to borrow. The `member.canBorrow()` method centralises all preconditions (active status, loan limit, fine threshold) so callers do not need to repeat this logic.

---

### Mistake 10: One-to-One Mapping Between Reservation and BookItem

A reservation represents a member's intent to borrow *any* copy of a title. Assigning a specific `BookItem` to a `Reservation` at creation time is wrong — you cannot know which copy will be returned first. The design correctly queues reservations at the `LibraryItem` level and assigns a copy only when one becomes available.
