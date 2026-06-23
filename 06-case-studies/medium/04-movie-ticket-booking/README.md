# LLD Case Study: Movie Ticket Booking System (BookMyShow)

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Functional Requirements](#2-functional-requirements)
3. [Non-Functional Requirements](#3-non-functional-requirements)
4. [Assumptions and Constraints](#4-assumptions-and-constraints)
5. [Domain Model Description](#5-domain-model-description)
6. [Class Diagram (ASCII UML)](#6-class-diagram-ascii-uml)
7. [Core Classes — Complete Java Implementation](#7-core-classes--complete-java-implementation)
8. [Design Patterns Used](#8-design-patterns-used)
9. [Key Design Decisions and Trade-offs](#9-key-design-decisions-and-trade-offs)
10. [Extension Points](#10-extension-points)
11. [Interview Discussion Points](#11-interview-discussion-points)
12. [Common Mistakes in This Design](#12-common-mistakes-in-this-design)

---

## 1. Problem Statement

Design a **Movie Ticket Booking System** similar to BookMyShow.

Users browse movies currently showing across multiple theaters, select a show (specific movie at a specific theater on a specific date/time), choose seats from a seat map, pay for their selection, and receive a booking confirmation. The system must support seat categories (Normal, Premium, VIP) with differential pricing, handle concurrent users attempting to book the same seats simultaneously without allowing double-booking, and enforce cancellation refund rules.

The critical challenge is **concurrency**: during peak times, thousands of users may simultaneously attempt to book seats for a popular show. The system must prevent two users from successfully booking the same seat.

---

## 2. Functional Requirements

1. Users can search for movies currently playing in a city.
2. Users can view all theaters showing a specific movie and the available show times.
3. Users can view the seat map for a specific show, with each seat's category and availability status.
4. Users can select one or more available seats for a show.
5. The system temporarily **locks** selected seats for up to 10 minutes while the user completes payment, preventing other users from selecting the same seats.
6. Users can complete payment for their locked seats; on success the seats are marked BOOKED.
7. If payment fails or the lock expires, locked seats are released back to AVAILABLE.
8. The system generates a unique booking confirmation with a booking ID upon successful payment.
9. Users can cancel a booking subject to refund rules:
   - More than 24 hours before show: **100% refund**
   - Between 12 and 24 hours before show: **50% refund**
   - Less than 12 hours before show: **no refund**
10. The system prevents double-booking under concurrent access (two users cannot book the same seat).
11. Multiple shows can run per screen per day (non-overlapping time slots).
12. Pricing varies by seat category and may increase during peak hours (weekends, evenings).

---

## 3. Non-Functional Requirements

- **Consistency**: A seat must never be booked by two users simultaneously (strong consistency for seat state).
- **Availability**: The system should remain available even under high load; degraded seat availability (showing a seat as locked when it was just released) is acceptable for a few seconds.
- **Low Latency**: Seat map reads should be fast (< 200 ms). Booking operations may take up to 2 seconds.
- **Scalability**: The system should scale to millions of bookings per day.
- **Durability**: Confirmed bookings must be persisted and not lost.
- **Fault Tolerance**: Expired seat locks must be released automatically (background job or TTL mechanism).

---

## 4. Assumptions and Constraints

- A **seat** belongs to exactly one screen. Each screen has a fixed, pre-configured seat layout.
- A **show** maps exactly one movie to one screen at one date-time. Shows do not overlap on the same screen.
- Seat locking is managed in-memory with a `ScheduledExecutorService` for this design; in production this would be backed by Redis with TTL.
- Payment processing is abstracted as a `PaymentService` interface — actual integration (Stripe, Razorpay) is outside scope.
- A user is identified by a `userId` (String); authentication/authorization is outside scope.
- Currency is a single denomination (INR by default) with prices stored as `double`.
- The in-memory `ConcurrentHashMap` for seats simulates a database row with a `version` field for optimistic locking demonstration.
- This design is single-JVM. In a distributed system, distributed locking (Redis Redlock) or database-level pessimistic/optimistic locking would replace the `ReentrantLock` shown here.

---

## 5. Domain Model Description

### Core Entities

| Entity | Responsibility |
|---|---|
| **Movie** | Immutable metadata: title, genre, duration, rating |
| **Theater** | Physical venue with a name, location, and one or more screens |
| **Screen** | A single auditorium inside a theater; owns the physical seat layout |
| **Seat** | An individual seat on a screen with a fixed row/column, category, and a mutable status |
| **Show** | An event: a movie playing on a screen at a specific date/time; manages per-show seat state |
| **Booking** | Records a confirmed or cancelled transaction: user, show, seats, amount, status |
| **Payment** | Represents a payment attempt tied to a booking |
| **User** | A person who makes bookings |

### Key Relationships

- A **Theater** has many **Screens**.
- A **Screen** has many **Seats** (the physical layout, immutable once configured).
- A **Show** references one **Movie** and one **Screen**; it maintains its own seat-status map copied from the screen layout at show creation time — so status changes for one show do not bleed into another show on the same screen.
- A **Booking** references one **Show** and one or more **Seats**.
- A **Payment** references exactly one **Booking**.

### Concurrency Model

`Show.bookSeats()` is the **critical section**. Each `Show` owns a `ReentrantLock`. When a user attempts to lock seats:

1. Acquire the show's `ReentrantLock`.
2. Verify each seat's status is still `AVAILABLE` and version matches (optimistic locking check).
3. Transition seats to `LOCKED` and increment their version.
4. Release the lock.
5. Schedule an unlock task for 10 minutes later.

If two threads enter simultaneously, one will acquire the lock first and the second will find the seats already `LOCKED`, causing it to fail gracefully.

---

## 6. Class Diagram (ASCII UML)

```
+------------------+       +-------------------+       +------------------+
|      Movie       |       |      Theater      |       |      User        |
+------------------+       +-------------------+       +------------------+
| - movieId:String |       | - theaterId:String|       | - userId:String  |
| - title:String   |       | - name:String     |       | - name:String    |
| - genre:String   |       | - location:String |       | - email:String   |
| - durationMins:  |       | - screens:List    |       | - phone:String   |
|     int          |       +-------------------+       +------------------+
| - rating:double  |               |
+------------------+               | 1..*
                                   v
                           +-------------------+
                           |      Screen       |
                           +-------------------+
                           | - screenId:String |
                           | - screenName:String|
                           | - totalRows:int   |
                           | - seatsPerRow:int |
                           | - seats:List<Seat>|
                           +-------------------+
                                   |
                                   | (seat layout)
                                   v
+------------------+       +-------------------+
|   SeatCategory   |       |       Seat        |
+------------------+       +-------------------+
| NORMAL           |<------| - seatId:String   |
| PREMIUM          |       | - row:int         |
| VIP              |       | - column:int      |
+------------------+       | - category:       |
                           |   SeatCategory    |
+------------------+       | - version:int     |
|   SeatStatus     |       +-------------------+
+------------------+
| AVAILABLE        |
| LOCKED           |
| BOOKED           |
+------------------+

           Show (per-show seat state, not screen state)
+-------------------------------------------------------------+
|                          Show                               |
+-------------------------------------------------------------+
| - showId:String                                             |
| - movie:Movie                                               |
| - screen:Screen                                             |
| - showTime:LocalDateTime                                    |
| - showSeatMap:Map<String,ShowSeat>  (seatId -> ShowSeat)    |
| - lock:ReentrantLock                                        |
+-------------------------------------------------------------+
| + bookSeats(seatIds, userId):List<ShowSeat>                 |
| + releaseSeats(seatIds)                                     |
| + confirmSeats(seatIds)                                     |
| + getAvailableSeats():List<ShowSeat>                        |
+-------------------------------------------------------------+

+-------------------------------------------------------------+
|                         ShowSeat                            |
+-------------------------------------------------------------+
| - showSeatId:String                                         |
| - seat:Seat  (immutable seat metadata)                      |
| - status:SeatStatus                                         |
| - lockedByUserId:String                                     |
| - lockExpiresAt:LocalDateTime                               |
| - version:int                                               |
+-------------------------------------------------------------+

+-------------------------------------------------------------+
|                         Booking                             |
+-------------------------------------------------------------+
| - bookingId:String                                          |
| - user:User                                                 |
| - show:Show                                                 |
| - showSeats:List<ShowSeat>                                  |
| - totalAmount:double                                        |
| - status:BookingStatus                                      |
| - createdAt:LocalDateTime                                   |
| - payment:Payment                                           |
+-------------------------------------------------------------+

+-------------------------------------------------------------+
|                         Payment                             |
+-------------------------------------------------------------+
| - paymentId:String                                          |
| - bookingId:String                                          |
| - amount:double                                             |
| - status:PaymentStatus                                      |
| - transactionRef:String                                     |
| - paidAt:LocalDateTime                                      |
+-------------------------------------------------------------+

<<interface>>                     <<interface>>
PricingStrategy                   PaymentService
+------------------------+        +------------------------+
| + calculatePrice(      |        | + processPayment(      |
|   ShowSeat):double     |        |   Booking):Payment     |
+------------------------+        | + processRefund(       |
         ^                        |   Payment,             |
         |                        |   double):boolean      |
   +-----+-----+                  +------------------------+
   |           |
   v           v
CategoryPricing  PeakHourPricing
Strategy         Strategy
```

---

## 7. Core Classes — Complete Java Implementation

### 7.1 Enums

```java
// SeatCategory.java
public enum SeatCategory {
    NORMAL,
    PREMIUM,
    VIP
}
```

```java
// SeatStatus.java
public enum SeatStatus {
    AVAILABLE,
    LOCKED,
    BOOKED
}
```

```java
// BookingStatus.java
public enum BookingStatus {
    PENDING,       // seats locked, awaiting payment
    CONFIRMED,     // payment successful
    CANCELLED,     // user cancelled
    EXPIRED        // lock timed out before payment
}
```

```java
// PaymentStatus.java
public enum PaymentStatus {
    PENDING,
    SUCCESS,
    FAILED,
    REFUNDED
}
```

---

### 7.2 Movie

```java
// Movie.java
import java.util.UUID;

public class Movie {

    private final String movieId;
    private final String title;
    private final String genre;
    private final int durationMinutes;
    private final double rating;       // e.g. 8.5 out of 10
    private final String language;

    public Movie(String title, String genre, int durationMinutes, double rating, String language) {
        this.movieId = UUID.randomUUID().toString();
        this.title = title;
        this.genre = genre;
        this.durationMinutes = durationMinutes;
        this.rating = rating;
        this.language = language;
    }

    public String getMovieId()        { return movieId; }
    public String getTitle()          { return title; }
    public String getGenre()          { return genre; }
    public int getDurationMinutes()   { return durationMinutes; }
    public double getRating()         { return rating; }
    public String getLanguage()       { return language; }

    @Override
    public String toString() {
        return String.format("Movie[%s, %s, %d min, %.1f]", title, genre, durationMinutes, rating);
    }
}
```

---

### 7.3 Seat

```java
// Seat.java
import java.util.UUID;

/**
 * Represents a physical seat on a Screen.
 * Immutable except for the version field, which tracks optimistic locking.
 * The mutable per-show state lives in ShowSeat, not here.
 */
public class Seat {

    private final String seatId;
    private final int row;
    private final int column;
    private final SeatCategory category;
    // Version is on Seat for the physical record; per-show version is on ShowSeat.
    private volatile int version;

    public Seat(int row, int column, SeatCategory category) {
        this.seatId  = UUID.randomUUID().toString();
        this.row     = row;
        this.column  = column;
        this.category = category;
        this.version = 0;
    }

    public String getSeatId()         { return seatId; }
    public int getRow()               { return row; }
    public int getColumn()            { return column; }
    public SeatCategory getCategory() { return category; }
    public int getVersion()           { return version; }
    public void incrementVersion()    { this.version++; }

    /** Human-readable label like "R3-C5" */
    public String getLabel() {
        return "R" + row + "-C" + column;
    }

    @Override
    public String toString() {
        return String.format("Seat[%s, row=%d, col=%d, %s]", seatId.substring(0, 8), row, column, category);
    }
}
```

---

### 7.4 ShowSeat

```java
// ShowSeat.java
import java.time.LocalDateTime;
import java.util.UUID;

/**
 * Per-show view of a seat.  Each Show gets its own ShowSeat instances so that
 * status changes for one show do not affect other shows on the same screen.
 *
 * The version field enables optimistic-locking detection:
 *   - Before locking, a caller records the current version.
 *   - After acquiring the show lock, it checks the version is unchanged.
 *   - If someone else modified the seat between the read and the lock acquisition,
 *     the version will differ and the operation is rejected.
 */
public class ShowSeat {

    private final String showSeatId;
    private final Seat seat;              // immutable physical seat
    private SeatStatus status;
    private String lockedByUserId;
    private LocalDateTime lockExpiresAt;
    private int version;                  // incremented on every state transition

    public ShowSeat(Seat seat) {
        this.showSeatId = UUID.randomUUID().toString();
        this.seat       = seat;
        this.status     = SeatStatus.AVAILABLE;
        this.version    = 0;
    }

    // -----------------------------------------------------------------------
    // State transitions — called only from within Show's critical section
    // -----------------------------------------------------------------------

    public void lock(String userId, LocalDateTime expiresAt) {
        this.status         = SeatStatus.LOCKED;
        this.lockedByUserId = userId;
        this.lockExpiresAt  = expiresAt;
        this.version++;
    }

    public void release() {
        this.status         = SeatStatus.AVAILABLE;
        this.lockedByUserId = null;
        this.lockExpiresAt  = null;
        this.version++;
    }

    public void book() {
        this.status         = SeatStatus.BOOKED;
        this.lockedByUserId = null;
        this.lockExpiresAt  = null;
        this.version++;
    }

    // -----------------------------------------------------------------------
    // Accessors
    // -----------------------------------------------------------------------

    public String getShowSeatId()            { return showSeatId; }
    public Seat getSeat()                    { return seat; }
    public SeatStatus getStatus()            { return status; }
    public String getLockedByUserId()        { return lockedByUserId; }
    public LocalDateTime getLockExpiresAt()  { return lockExpiresAt; }
    public int getVersion()                  { return version; }

    public boolean isAvailable() {
        return status == SeatStatus.AVAILABLE;
    }

    public boolean isLocked() {
        return status == SeatStatus.LOCKED;
    }

    public boolean isLockedByUser(String userId) {
        return status == SeatStatus.LOCKED && userId.equals(lockedByUserId);
    }

    public boolean isLockExpired() {
        return status == SeatStatus.LOCKED
                && lockExpiresAt != null
                && LocalDateTime.now().isAfter(lockExpiresAt);
    }

    @Override
    public String toString() {
        return String.format("ShowSeat[%s, %s, %s, v%d]",
                seat.getLabel(), seat.getCategory(), status, version);
    }
}
```

---

### 7.5 Screen

```java
// Screen.java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.UUID;

public class Screen {

    private final String screenId;
    private final String screenName;
    private final int totalRows;
    private final int seatsPerRow;
    private final List<Seat> seats;   // immutable after construction

    public Screen(String screenName, int totalRows, int seatsPerRow) {
        this.screenId    = UUID.randomUUID().toString();
        this.screenName  = screenName;
        this.totalRows   = totalRows;
        this.seatsPerRow = seatsPerRow;
        this.seats       = new ArrayList<>();
        initializeSeats();
    }

    /**
     * Builds the seat layout.
     * Convention:
     *   Rows 1..2               -> VIP
     *   Rows 3..(totalRows/2)   -> PREMIUM
     *   Remaining rows          -> NORMAL
     */
    private void initializeSeats() {
        int premiumEndRow = totalRows / 2;
        for (int row = 1; row <= totalRows; row++) {
            for (int col = 1; col <= seatsPerRow; col++) {
                SeatCategory category;
                if (row <= 2) {
                    category = SeatCategory.VIP;
                } else if (row <= premiumEndRow) {
                    category = SeatCategory.PREMIUM;
                } else {
                    category = SeatCategory.NORMAL;
                }
                seats.add(new Seat(row, col, category));
            }
        }
    }

    public String getScreenId()          { return screenId; }
    public String getScreenName()        { return screenName; }
    public int getTotalRows()            { return totalRows; }
    public int getSeatsPerRow()          { return seatsPerRow; }
    public List<Seat> getSeats()         { return Collections.unmodifiableList(seats); }
    public int getTotalCapacity()        { return totalRows * seatsPerRow; }

    @Override
    public String toString() {
        return String.format("Screen[%s, %d rows x %d cols = %d seats]",
                screenName, totalRows, seatsPerRow, getTotalCapacity());
    }
}
```

---

### 7.6 Theater

```java
// Theater.java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.UUID;

public class Theater {

    private final String theaterId;
    private final String name;
    private final String location;
    private final List<Screen> screens;

    public Theater(String name, String location) {
        this.theaterId = UUID.randomUUID().toString();
        this.name      = name;
        this.location  = location;
        this.screens   = new ArrayList<>();
    }

    public void addScreen(Screen screen) {
        screens.add(screen);
    }

    public String getTheaterId()          { return theaterId; }
    public String getName()               { return name; }
    public String getLocation()           { return location; }
    public List<Screen> getScreens()      { return Collections.unmodifiableList(screens); }

    @Override
    public String toString() {
        return String.format("Theater[%s, %s, %d screens]", name, location, screens.size());
    }
}
```

---

### 7.7 User

```java
// User.java
import java.util.UUID;

public class User {

    private final String userId;
    private final String name;
    private final String email;
    private final String phone;

    public User(String name, String email, String phone) {
        this.userId = UUID.randomUUID().toString();
        this.name   = name;
        this.email  = email;
        this.phone  = phone;
    }

    public String getUserId() { return userId; }
    public String getName()   { return name; }
    public String getEmail()  { return email; }
    public String getPhone()  { return phone; }

    @Override
    public String toString() {
        return String.format("User[%s, %s]", name, email);
    }
}
```

---

### 7.8 Pricing Strategy

```java
// PricingStrategy.java

/**
 * Strategy interface for calculating the price of a single ShowSeat.
 * Implementations can be chained (decorator) to apply multiple adjustments.
 */
public interface PricingStrategy {
    double calculatePrice(ShowSeat showSeat);
}
```

```java
// CategoryPricingStrategy.java

/**
 * Base pricing driven purely by seat category.
 * This is always the first strategy in the chain.
 */
public class CategoryPricingStrategy implements PricingStrategy {

    private final double normalPrice;
    private final double premiumPrice;
    private final double vipPrice;

    public CategoryPricingStrategy(double normalPrice, double premiumPrice, double vipPrice) {
        this.normalPrice  = normalPrice;
        this.premiumPrice = premiumPrice;
        this.vipPrice     = vipPrice;
    }

    @Override
    public double calculatePrice(ShowSeat showSeat) {
        SeatCategory category = showSeat.getSeat().getCategory();
        switch (category) {
            case VIP:     return vipPrice;
            case PREMIUM: return premiumPrice;
            case NORMAL:  return normalPrice;
            default:
                throw new IllegalArgumentException("Unknown seat category: " + category);
        }
    }
}
```

```java
// PeakHourPricingStrategy.java
import java.time.DayOfWeek;
import java.time.LocalDateTime;

/**
 * Decorator that wraps another PricingStrategy and applies a peak-hour
 * surcharge multiplier on weekends or evening shows (after 6 PM).
 *
 * Pattern: Decorator (also can be seen as Chain of Responsibility).
 */
public class PeakHourPricingStrategy implements PricingStrategy {

    private final PricingStrategy baseStrategy;
    private final double peakMultiplier;
    private final LocalDateTime showTime;

    public PeakHourPricingStrategy(PricingStrategy baseStrategy,
                                   double peakMultiplier,
                                   LocalDateTime showTime) {
        this.baseStrategy    = baseStrategy;
        this.peakMultiplier  = peakMultiplier;
        this.showTime        = showTime;
    }

    @Override
    public double calculatePrice(ShowSeat showSeat) {
        double basePrice = baseStrategy.calculatePrice(showSeat);
        if (isPeakTime()) {
            return basePrice * peakMultiplier;
        }
        return basePrice;
    }

    private boolean isPeakTime() {
        DayOfWeek day  = showTime.getDayOfWeek();
        int hour        = showTime.getHour();
        boolean isWeekend = (day == DayOfWeek.SATURDAY || day == DayOfWeek.SUNDAY);
        boolean isEvening = (hour >= 18);   // 6 PM or later
        return isWeekend || isEvening;
    }
}
```

---

### 7.9 Show

```java
// Show.java
import java.time.LocalDateTime;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReentrantLock;
import java.util.stream.Collectors;

/**
 * Represents a single screening event: one movie on one screen at one time.
 *
 * CONCURRENCY DESIGN:
 * ------------------
 * Each Show owns a ReentrantLock.  All seat state transitions go through this
 * lock so that no two threads can concurrently modify seat status.
 *
 * Optimistic locking is simulated via ShowSeat.version:
 *   - A caller reads the version before deciding to book.
 *   - Inside the critical section the version is re-checked.
 *   - If they differ, someone else modified the seat in between — fail fast.
 *
 * This mimics a database UPDATE ... WHERE version = :expected which is the
 * standard optimistic-locking pattern used by JPA / Hibernate.
 */
public class Show {

    private final String showId;
    private final Movie movie;
    private final Screen screen;
    private final LocalDateTime showTime;
    private final PricingStrategy pricingStrategy;

    // seatId -> ShowSeat (per-show mutable seat state)
    private final Map<String, ShowSeat> showSeatMap;

    // One lock per show — only one thread mutates seat state at a time
    private final ReentrantLock lock;

    public Show(Movie movie, Screen screen, LocalDateTime showTime,
                PricingStrategy pricingStrategy) {
        this.showId          = UUID.randomUUID().toString();
        this.movie           = movie;
        this.screen          = screen;
        this.showTime        = showTime;
        this.pricingStrategy = pricingStrategy;
        this.showSeatMap     = new ConcurrentHashMap<>();
        this.lock            = new ReentrantLock(true); // fair lock: FIFO ordering

        // Create a ShowSeat for every physical seat on this screen
        for (Seat seat : screen.getSeats()) {
            showSeatMap.put(seat.getSeatId(), new ShowSeat(seat));
        }
    }

    // -----------------------------------------------------------------------
    // Seat Locking — THE CRITICAL SECTION
    // -----------------------------------------------------------------------

    /**
     * Attempts to lock the given seats for the specified user.
     *
     * Steps:
     *  1. Acquire the ReentrantLock (blocking — threads queue up).
     *  2. For each seat, verify:
     *       a. The ShowSeat exists (defensive).
     *       b. Its status is AVAILABLE (not locked/booked by someone else).
     *       c. Its version matches the caller's expected version (optimistic check).
     *  3. If all checks pass, transition all seats to LOCKED atomically.
     *  4. Release the lock.
     *
     * @param seatIds         list of seatIds the user wants to book
     * @param userId          the user attempting the lock
     * @param expectedVersions map of seatId -> version the caller read before calling
     * @param lockDurationMins how many minutes to hold the lock
     * @return list of ShowSeats that were successfully locked
     * @throws SeatNotAvailableException if any seat is unavailable or version mismatch
     */
    public List<ShowSeat> lockSeats(List<String> seatIds,
                                    String userId,
                                    Map<String, Integer> expectedVersions,
                                    int lockDurationMins) {
        lock.lock();
        try {
            LocalDateTime expiresAt = LocalDateTime.now().plusMinutes(lockDurationMins);

            // --- Phase 1: Validate all seats atomically ---
            List<ShowSeat> toBeBooked = new ArrayList<>();
            for (String seatId : seatIds) {
                ShowSeat showSeat = showSeatMap.get(seatId);
                if (showSeat == null) {
                    throw new SeatNotAvailableException("Seat not found: " + seatId);
                }

                // Expire stale locks before checking availability
                if (showSeat.isLockExpired()) {
                    showSeat.release();
                }

                if (!showSeat.isAvailable()) {
                    throw new SeatNotAvailableException(
                            "Seat " + showSeat.getSeat().getLabel() +
                            " is already " + showSeat.getStatus() + ".");
                }

                // Optimistic locking check: version must match what caller read
                int expected = expectedVersions.getOrDefault(seatId, 0);
                if (showSeat.getVersion() != expected) {
                    throw new SeatNotAvailableException(
                            "Concurrent modification detected for seat " +
                            showSeat.getSeat().getLabel() +
                            ". Please refresh and try again.");
                }

                toBeBooked.add(showSeat);
            }

            // --- Phase 2: Apply lock transition atomically ---
            for (ShowSeat showSeat : toBeBooked) {
                showSeat.lock(userId, expiresAt);
            }

            System.out.printf("[Show %s] Seats locked by %s until %s%n",
                    showId.substring(0, 8), userId, expiresAt);
            return Collections.unmodifiableList(toBeBooked);

        } finally {
            lock.unlock();
        }
    }

    /**
     * Releases locked seats back to AVAILABLE.
     * Called on payment failure, booking cancellation, or lock expiry.
     */
    public void releaseSeats(List<String> seatIds) {
        lock.lock();
        try {
            for (String seatId : seatIds) {
                ShowSeat showSeat = showSeatMap.get(seatId);
                if (showSeat != null && showSeat.isLocked()) {
                    showSeat.release();
                    System.out.printf("[Show %s] Seat %s released%n",
                            showId.substring(0, 8), showSeat.getSeat().getLabel());
                }
            }
        } finally {
            lock.unlock();
        }
    }

    /**
     * Confirms seats (transitions LOCKED -> BOOKED).
     * Called only after payment succeeds.
     */
    public void confirmSeats(List<String> seatIds, String userId) {
        lock.lock();
        try {
            for (String seatId : seatIds) {
                ShowSeat showSeat = showSeatMap.get(seatId);
                if (showSeat == null) {
                    throw new IllegalStateException("Seat not found: " + seatId);
                }
                if (!showSeat.isLockedByUser(userId)) {
                    throw new IllegalStateException(
                            "Seat " + showSeat.getSeat().getLabel() +
                            " is not locked by user " + userId);
                }
                showSeat.book();
            }
            System.out.printf("[Show %s] Seats confirmed (BOOKED) for user %s%n",
                    showId.substring(0, 8), userId);
        } finally {
            lock.unlock();
        }
    }

    // -----------------------------------------------------------------------
    // Queries
    // -----------------------------------------------------------------------

    public List<ShowSeat> getAvailableSeats() {
        return showSeatMap.values().stream()
                .filter(ShowSeat::isAvailable)
                .collect(Collectors.toList());
    }

    public List<ShowSeat> getAllShowSeats() {
        return Collections.unmodifiableList(new ArrayList<>(showSeatMap.values()));
    }

    public ShowSeat getShowSeat(String seatId) {
        return showSeatMap.get(seatId);
    }

    public double getPriceForSeat(ShowSeat showSeat) {
        return pricingStrategy.calculatePrice(showSeat);
    }

    // -----------------------------------------------------------------------
    // Accessors
    // -----------------------------------------------------------------------

    public String getShowId()            { return showId; }
    public Movie getMovie()              { return movie; }
    public Screen getScreen()            { return screen; }
    public LocalDateTime getShowTime()   { return showTime; }

    @Override
    public String toString() {
        return String.format("Show[%s at %s on %s]",
                movie.getTitle(), screen.getScreenName(), showTime);
    }
}
```

---

### 7.10 SeatNotAvailableException

```java
// SeatNotAvailableException.java

public class SeatNotAvailableException extends RuntimeException {

    public SeatNotAvailableException(String message) {
        super(message);
    }

    public SeatNotAvailableException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

---

### 7.11 Payment and PaymentService

```java
// Payment.java
import java.time.LocalDateTime;
import java.util.UUID;

public class Payment {

    private final String paymentId;
    private final String bookingId;
    private final double amount;
    private PaymentStatus status;
    private String transactionRef;
    private LocalDateTime paidAt;
    private double refundAmount;

    public Payment(String bookingId, double amount) {
        this.paymentId  = UUID.randomUUID().toString();
        this.bookingId  = bookingId;
        this.amount     = amount;
        this.status     = PaymentStatus.PENDING;
        this.refundAmount = 0.0;
    }

    public void markSuccess(String transactionRef) {
        this.status         = PaymentStatus.SUCCESS;
        this.transactionRef = transactionRef;
        this.paidAt         = LocalDateTime.now();
    }

    public void markFailed() {
        this.status = PaymentStatus.FAILED;
    }

    public void markRefunded(double refundAmount) {
        this.refundAmount = refundAmount;
        this.status       = PaymentStatus.REFUNDED;
    }

    public String getPaymentId()      { return paymentId; }
    public String getBookingId()      { return bookingId; }
    public double getAmount()         { return amount; }
    public PaymentStatus getStatus()  { return status; }
    public String getTransactionRef() { return transactionRef; }
    public LocalDateTime getPaidAt()  { return paidAt; }
    public double getRefundAmount()   { return refundAmount; }

    @Override
    public String toString() {
        return String.format("Payment[id=%s, amount=%.2f, status=%s]",
                paymentId.substring(0, 8), amount, status);
    }
}
```

```java
// PaymentService.java

/**
 * Abstraction over actual payment gateway integration.
 */
public interface PaymentService {
    /**
     * Processes a payment for the booking.
     * Returns the Payment object with status set to SUCCESS or FAILED.
     */
    Payment processPayment(Booking booking);

    /**
     * Issues a refund for a previous payment.
     * @param payment     the original payment
     * @param refundAmount the amount to refund (may be partial)
     * @return true if refund was issued successfully
     */
    boolean processRefund(Payment payment, double refundAmount);
}
```

```java
// MockPaymentService.java
import java.util.UUID;

/**
 * Simulated payment service for demonstration.
 * Always succeeds in tests; can be configured to fail for negative-path testing.
 */
public class MockPaymentService implements PaymentService {

    private final boolean shouldSucceed;

    public MockPaymentService(boolean shouldSucceed) {
        this.shouldSucceed = shouldSucceed;
    }

    @Override
    public Payment processPayment(Booking booking) {
        Payment payment = new Payment(booking.getBookingId(), booking.getTotalAmount());
        if (shouldSucceed) {
            String txRef = "TXN-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
            payment.markSuccess(txRef);
            System.out.printf("[Payment] Success: %.2f for booking %s | ref=%s%n",
                    booking.getTotalAmount(), booking.getBookingId().substring(0, 8), txRef);
        } else {
            payment.markFailed();
            System.out.printf("[Payment] Failed for booking %s%n",
                    booking.getBookingId().substring(0, 8));
        }
        return payment;
    }

    @Override
    public boolean processRefund(Payment payment, double refundAmount) {
        payment.markRefunded(refundAmount);
        System.out.printf("[Payment] Refund of %.2f issued for payment %s%n",
                refundAmount, payment.getPaymentId().substring(0, 8));
        return true;
    }
}
```

---

### 7.12 Booking

```java
// Booking.java
import java.time.LocalDateTime;
import java.util.Collections;
import java.util.List;
import java.util.UUID;

public class Booking {

    private final String bookingId;
    private final User user;
    private final Show show;
    private final List<ShowSeat> showSeats;
    private final double totalAmount;
    private BookingStatus status;
    private final LocalDateTime createdAt;
    private Payment payment;

    public Booking(User user, Show show, List<ShowSeat> showSeats, double totalAmount) {
        this.bookingId   = UUID.randomUUID().toString();
        this.user        = user;
        this.show        = show;
        this.showSeats   = Collections.unmodifiableList(showSeats);
        this.totalAmount = totalAmount;
        this.status      = BookingStatus.PENDING;
        this.createdAt   = LocalDateTime.now();
    }

    // -----------------------------------------------------------------------
    // State transitions
    // -----------------------------------------------------------------------

    public void confirm(Payment payment) {
        this.payment = payment;
        this.status  = BookingStatus.CONFIRMED;
    }

    public void cancel() {
        this.status = BookingStatus.CANCELLED;
    }

    public void expire() {
        this.status = BookingStatus.EXPIRED;
    }

    // -----------------------------------------------------------------------
    // Accessors
    // -----------------------------------------------------------------------

    public String getBookingId()           { return bookingId; }
    public User getUser()                  { return user; }
    public Show getShow()                  { return show; }
    public List<ShowSeat> getShowSeats()   { return showSeats; }
    public double getTotalAmount()         { return totalAmount; }
    public BookingStatus getStatus()       { return status; }
    public LocalDateTime getCreatedAt()    { return createdAt; }
    public Payment getPayment()            { return payment; }

    public List<String> getSeatIds() {
        return showSeats.stream()
                .map(ss -> ss.getSeat().getSeatId())
                .collect(java.util.stream.Collectors.toList());
    }

    @Override
    public String toString() {
        return String.format("Booking[id=%s, user=%s, show=%s, seats=%d, amount=%.2f, status=%s]",
                bookingId.substring(0, 8), user.getName(), show.getMovie().getTitle(),
                showSeats.size(), totalAmount, status);
    }
}
```

---

### 7.13 BookingService

```java
// BookingService.java
import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.Collectors;

/**
 * Orchestrates the full booking lifecycle:
 *   1. Lock seats (with 10-min expiry)
 *   2. Calculate price
 *   3. Process payment
 *   4. Confirm or release seats
 *   5. Handle cancellation with refund rules
 *
 * The ScheduledExecutorService automatically releases seats whose lock
 * has expired (simulating what Redis TTL would do in production).
 */
public class BookingService {

    private static final int LOCK_DURATION_MINUTES = 10;

    // bookingId -> Booking
    private final Map<String, Booking> bookingStore = new ConcurrentHashMap<>();

    private final PaymentService paymentService;
    private final ScheduledExecutorService scheduler;

    public BookingService(PaymentService paymentService) {
        this.paymentService = paymentService;
        // Single-thread scheduler is enough: lock-release tasks are cheap
        this.scheduler = Executors.newScheduledThreadPool(2);
    }

    // -----------------------------------------------------------------------
    // Step 1 + 2: Initiate booking (lock seats)
    // -----------------------------------------------------------------------

    /**
     * Initiates a booking attempt:
     *   - Reads current versions of requested seats (snapshot).
     *   - Calls Show.lockSeats() inside the show's ReentrantLock.
     *   - Creates a PENDING Booking record.
     *   - Schedules automatic seat release after LOCK_DURATION_MINUTES.
     *
     * @param user    the user initiating the booking
     * @param show    the show being booked
     * @param seatIds list of seatIds the user selected
     * @return a PENDING Booking (user must call confirmBooking() next)
     * @throws SeatNotAvailableException if any seat is unavailable
     */
    public Booking initiateBooking(User user, Show show, List<String> seatIds) {
        // Capture version snapshot BEFORE acquiring the show lock
        // This is the "optimistic read" part of optimistic locking
        Map<String, Integer> expectedVersions = new HashMap<>();
        for (String seatId : seatIds) {
            ShowSeat showSeat = show.getShowSeat(seatId);
            if (showSeat == null) {
                throw new SeatNotAvailableException("Seat not found: " + seatId);
            }
            expectedVersions.put(seatId, showSeat.getVersion());
        }

        // Lock seats inside the show's critical section
        List<ShowSeat> lockedSeats = show.lockSeats(
                seatIds, user.getUserId(), expectedVersions, LOCK_DURATION_MINUTES);

        // Calculate total amount
        double totalAmount = lockedSeats.stream()
                .mapToDouble(show::getPriceForSeat)
                .sum();

        Booking booking = new Booking(user, show, lockedSeats, totalAmount);
        bookingStore.put(booking.getBookingId(), booking);

        System.out.printf("[BookingService] Booking %s INITIATED for %s | %d seat(s) | %.2f%n",
                booking.getBookingId().substring(0, 8), user.getName(),
                lockedSeats.size(), totalAmount);

        // Schedule automatic seat release if user doesn't pay in time
        scheduleAutoRelease(booking, show);

        return booking;
    }

    // -----------------------------------------------------------------------
    // Step 3: Confirm booking (process payment)
    // -----------------------------------------------------------------------

    /**
     * Processes payment and transitions booking from PENDING to CONFIRMED.
     * If payment fails, locked seats are immediately released.
     *
     * @param bookingId the booking to confirm
     * @return the confirmed Booking
     * @throws IllegalArgumentException if booking not found
     * @throws IllegalStateException    if booking is not in PENDING state
     */
    public Booking confirmBooking(String bookingId) {
        Booking booking = getBookingOrThrow(bookingId);

        if (booking.getStatus() != BookingStatus.PENDING) {
            throw new IllegalStateException(
                    "Booking " + bookingId.substring(0, 8) + " is not in PENDING state. " +
                    "Current status: " + booking.getStatus());
        }

        Payment payment = paymentService.processPayment(booking);

        if (payment.getStatus() == PaymentStatus.SUCCESS) {
            // Transition seats: LOCKED -> BOOKED
            booking.getShow().confirmSeats(booking.getSeatIds(), booking.getUser().getUserId());
            booking.confirm(payment);
            System.out.printf("[BookingService] Booking %s CONFIRMED%n",
                    bookingId.substring(0, 8));
        } else {
            // Payment failed — release seats immediately
            booking.getShow().releaseSeats(booking.getSeatIds());
            booking.expire();
            System.out.printf("[BookingService] Booking %s EXPIRED (payment failed)%n",
                    bookingId.substring(0, 8));
        }

        return booking;
    }

    // -----------------------------------------------------------------------
    // Step 4: Cancel booking (with refund rules)
    // -----------------------------------------------------------------------

    /**
     * Cancels a confirmed booking and issues a refund based on how far
     * in advance the cancellation is made relative to show time:
     *
     *   > 24 hours  : 100% refund
     *   12–24 hours : 50%  refund
     *   < 12 hours  : 0%   refund (no refund)
     *
     * @param bookingId the booking to cancel
     * @return the cancelled Booking
     */
    public Booking cancelBooking(String bookingId) {
        Booking booking = getBookingOrThrow(bookingId);

        if (booking.getStatus() != BookingStatus.CONFIRMED) {
            throw new IllegalStateException(
                    "Only CONFIRMED bookings can be cancelled. " +
                    "Current status: " + booking.getStatus());
        }

        LocalDateTime showTime = booking.getShow().getShowTime();
        LocalDateTime now      = LocalDateTime.now();
        long hoursUntilShow    = ChronoUnit.HOURS.between(now, showTime);

        double refundPercentage;
        if (hoursUntilShow > 24) {
            refundPercentage = 1.0;   // 100%
        } else if (hoursUntilShow >= 12) {
            refundPercentage = 0.5;   // 50%
        } else {
            refundPercentage = 0.0;   // no refund
        }

        double refundAmount = booking.getTotalAmount() * refundPercentage;

        System.out.printf("[BookingService] Cancelling booking %s | hours until show: %d | " +
                "refund: %.0f%% = %.2f%n",
                bookingId.substring(0, 8), hoursUntilShow,
                refundPercentage * 100, refundAmount);

        // Release seats back to AVAILABLE
        // (seats were BOOKED; we return them to AVAILABLE so others can buy)
        booking.getShow().releaseSeats(booking.getSeatIds());
        booking.cancel();

        // Process refund if applicable
        if (refundAmount > 0 && booking.getPayment() != null) {
            paymentService.processRefund(booking.getPayment(), refundAmount);
        }

        return booking;
    }

    // -----------------------------------------------------------------------
    // Helpers
    // -----------------------------------------------------------------------

    /**
     * Schedules a background task to release seats if the user
     * doesn't complete payment within the lock window.
     */
    private void scheduleAutoRelease(Booking booking, Show show) {
        scheduler.schedule(() -> {
            Booking current = bookingStore.get(booking.getBookingId());
            if (current != null && current.getStatus() == BookingStatus.PENDING) {
                show.releaseSeats(booking.getSeatIds());
                current.expire();
                System.out.printf("[BookingService] Auto-released seats for expired booking %s%n",
                        booking.getBookingId().substring(0, 8));
            }
        }, LOCK_DURATION_MINUTES, TimeUnit.MINUTES);
    }

    private Booking getBookingOrThrow(String bookingId) {
        Booking booking = bookingStore.get(bookingId);
        if (booking == null) {
            throw new IllegalArgumentException("Booking not found: " + bookingId);
        }
        return booking;
    }

    public Booking getBooking(String bookingId) {
        return bookingStore.get(bookingId);
    }

    public Collection<Booking> getAllBookings() {
        return Collections.unmodifiableCollection(bookingStore.values());
    }

    public void shutdown() {
        scheduler.shutdown();
    }
}
```

---

### 7.14 MovieTicketBookingSystem (Facade)

```java
// MovieTicketBookingSystem.java
import java.time.LocalDateTime;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

/**
 * Facade / entry point for the booking system.
 * Maintains registries for movies, theaters, and shows.
 * Delegates complex booking logic to BookingService.
 */
public class MovieTicketBookingSystem {

    // Registries
    private final Map<String, Movie>   movies   = new ConcurrentHashMap<>();
    private final Map<String, Theater> theaters = new ConcurrentHashMap<>();
    private final Map<String, Show>    shows    = new ConcurrentHashMap<>();

    private final BookingService bookingService;

    public MovieTicketBookingSystem(PaymentService paymentService) {
        this.bookingService = new BookingService(paymentService);
    }

    // -----------------------------------------------------------------------
    // Admin operations (catalog management)
    // -----------------------------------------------------------------------

    public void addMovie(Movie movie) {
        movies.put(movie.getMovieId(), movie);
        System.out.println("[System] Added movie: " + movie);
    }

    public void addTheater(Theater theater) {
        theaters.put(theater.getTheaterId(), theater);
        System.out.println("[System] Added theater: " + theater);
    }

    public Show scheduleShow(Movie movie, Screen screen, LocalDateTime showTime,
                              PricingStrategy pricingStrategy) {
        Show show = new Show(movie, screen, showTime, pricingStrategy);
        shows.put(show.getShowId(), show);
        System.out.println("[System] Scheduled: " + show);
        return show;
    }

    // -----------------------------------------------------------------------
    // User-facing search operations
    // -----------------------------------------------------------------------

    /** Returns all shows for a given movie, sorted by showTime. */
    public List<Show> getShowsForMovie(String movieId) {
        return shows.values().stream()
                .filter(s -> s.getMovie().getMovieId().equals(movieId))
                .sorted(Comparator.comparing(Show::getShowTime))
                .collect(Collectors.toList());
    }

    /** Returns all available seats for a specific show. */
    public List<ShowSeat> getAvailableSeats(String showId) {
        Show show = shows.get(showId);
        if (show == null) {
            throw new IllegalArgumentException("Show not found: " + showId);
        }
        return show.getAvailableSeats();
    }

    // -----------------------------------------------------------------------
    // Booking operations (delegated to BookingService)
    // -----------------------------------------------------------------------

    public Booking initiateBooking(User user, String showId, List<String> seatIds) {
        Show show = shows.get(showId);
        if (show == null) {
            throw new IllegalArgumentException("Show not found: " + showId);
        }
        return bookingService.initiateBooking(user, show, seatIds);
    }

    public Booking confirmBooking(String bookingId) {
        return bookingService.confirmBooking(bookingId);
    }

    public Booking cancelBooking(String bookingId) {
        return bookingService.cancelBooking(bookingId);
    }

    public Booking getBooking(String bookingId) {
        return bookingService.getBooking(bookingId);
    }

    public void shutdown() {
        bookingService.shutdown();
    }
}
```

---

### 7.15 Main — Concurrent Booking Demo

```java
// Main.java
import java.time.LocalDateTime;
import java.util.*;
import java.util.concurrent.*;

/**
 * Demonstrates the booking system with a concurrent booking scenario:
 *   - Two threads attempt to book the SAME seat simultaneously.
 *   - Only one should succeed; the other must receive a SeatNotAvailableException.
 */
public class Main {

    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Movie Ticket Booking System Demo ===\n");

        // ----------------------------------------------------------------
        // 1. Setup: Create movies, theater, screens, shows
        // ----------------------------------------------------------------

        MovieTicketBookingSystem system =
                new MovieTicketBookingSystem(new MockPaymentService(true));

        Movie movie = new Movie("Inception", "Sci-Fi Thriller", 148, 8.8, "English");
        system.addMovie(movie);

        Theater theater = new Theater("PVR Cinemas", "Connaught Place, Delhi");
        Screen screen   = new Screen("Screen 1", 10, 12);  // 10 rows x 12 cols = 120 seats
        theater.addScreen(screen);
        system.addTheater(theater);

        // Pricing: Normal=200, Premium=350, VIP=500; 1.3x on peak hours
        LocalDateTime showTime = LocalDateTime.now().plusDays(2).withHour(21).withMinute(0);
        PricingStrategy basePricing    = new CategoryPricingStrategy(200.0, 350.0, 500.0);
        PricingStrategy peakPricing    = new PeakHourPricingStrategy(basePricing, 1.3, showTime);

        Show show = system.scheduleShow(movie, screen, showTime, peakPricing);

        // ----------------------------------------------------------------
        // 2. Display available seats (first 5 for brevity)
        // ----------------------------------------------------------------

        List<ShowSeat> available = system.getAvailableSeats(show.getShowId());
        System.out.println("\n[Available Seats - first 5]");
        available.stream().limit(5).forEach(ss ->
                System.out.printf("  %s | %s | Price: %.2f%n",
                        ss.getSeat().getLabel(),
                        ss.getSeat().getCategory(),
                        show.getPriceForSeat(ss)));

        // ----------------------------------------------------------------
        // 3. Happy-path booking (User A books 2 seats)
        // ----------------------------------------------------------------

        System.out.println("\n--- Happy Path: User A books 2 seats ---");

        User userA = new User("Alice", "alice@example.com", "+91-9900000001");
        User userB = new User("Bob",   "bob@example.com",   "+91-9900000002");

        // Pick the first 2 available seatIds
        String seatId1 = available.get(0).getSeat().getSeatId();
        String seatId2 = available.get(1).getSeat().getSeatId();

        Booking bookingA = system.initiateBooking(
                userA, show.getShowId(), Arrays.asList(seatId1, seatId2));

        Booking confirmedA = system.confirmBooking(bookingA.getBookingId());
        System.out.println("[Result] " + confirmedA);
        System.out.println("[Payment] " + confirmedA.getPayment());

        // ----------------------------------------------------------------
        // 4. Concurrent booking: Two threads try to book the SAME seat
        // ----------------------------------------------------------------

        System.out.println("\n--- Concurrency Test: Two threads race for the same seat ---");

        // Pick a fresh available seat
        List<ShowSeat> stillAvailable = system.getAvailableSeats(show.getShowId());
        String racedSeatId = stillAvailable.get(0).getSeat().getSeatId();

        User userC = new User("Carol", "carol@example.com", "+91-9900000003");
        User userD = new User("Dave",  "dave@example.com",  "+91-9900000004");

        // Use a CountDownLatch to make both threads start at exactly the same time
        CountDownLatch startGate = new CountDownLatch(1);
        CountDownLatch endGate   = new CountDownLatch(2);

        // Shared results
        String[] results = new String[2];

        Thread thread1 = new Thread(() -> {
            try {
                startGate.await();  // wait for both threads to be ready
                Booking b = system.initiateBooking(
                        userC, show.getShowId(), Collections.singletonList(racedSeatId));
                system.confirmBooking(b.getBookingId());
                results[0] = "Thread-1 (Carol): SUCCESS — Booking " + b.getBookingId().substring(0, 8);
            } catch (SeatNotAvailableException e) {
                results[0] = "Thread-1 (Carol): FAILED — " + e.getMessage();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                results[0] = "Thread-1 (Carol): INTERRUPTED";
            } finally {
                endGate.countDown();
            }
        }, "BookingThread-Carol");

        Thread thread2 = new Thread(() -> {
            try {
                startGate.await();  // wait for both threads to be ready
                Booking b = system.initiateBooking(
                        userD, show.getShowId(), Collections.singletonList(racedSeatId));
                system.confirmBooking(b.getBookingId());
                results[1] = "Thread-2 (Dave):  SUCCESS — Booking " + b.getBookingId().substring(0, 8);
            } catch (SeatNotAvailableException e) {
                results[1] = "Thread-2 (Dave):  FAILED — " + e.getMessage();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                results[1] = "Thread-2 (Dave):  INTERRUPTED";
            } finally {
                endGate.countDown();
            }
        }, "BookingThread-Dave");

        thread1.start();
        thread2.start();

        // Release both threads simultaneously
        startGate.countDown();

        // Wait for both threads to finish (max 5 seconds)
        endGate.await(5, TimeUnit.SECONDS);

        System.out.println("\n[Concurrency Results]");
        System.out.println("  " + results[0]);
        System.out.println("  " + results[1]);
        System.out.println("  => Exactly ONE thread should succeed. No double-booking occurred.");

        // ----------------------------------------------------------------
        // 5. Cancellation demo
        // ----------------------------------------------------------------

        System.out.println("\n--- Cancellation Demo ---");
        // confirmedA was booked 2 days in the future => > 24 hours => 100% refund
        Booking cancelledA = system.cancelBooking(confirmedA.getBookingId());
        System.out.println("[Result] Booking status: " + cancelledA.getStatus());
        if (cancelledA.getPayment() != null) {
            System.out.printf("[Result] Refunded: %.2f (status: %s)%n",
                    cancelledA.getPayment().getRefundAmount(),
                    cancelledA.getPayment().getStatus());
        }

        // ----------------------------------------------------------------
        // 6. Optimistic locking demonstration
        // ----------------------------------------------------------------

        System.out.println("\n--- Optimistic Locking Demo ---");
        List<ShowSeat> freshSeats = system.getAvailableSeats(show.getShowId());
        if (!freshSeats.isEmpty()) {
            ShowSeat targetSeat = freshSeats.get(0);
            int staleVersion    = targetSeat.getVersion();  // read version = 0

            // Simulate: someone else modifies the seat between our read and our lock attempt
            // We do this by booking it first with another user...
            User userE = new User("Eve", "eve@example.com", "+91-9900000005");
            Booking eBooking = system.initiateBooking(
                    userE, show.getShowId(),
                    Collections.singletonList(targetSeat.getSeat().getSeatId()));
            // ...now the version has incremented to 1

            // Now try with the stale version
            User userF = new User("Frank", "frank@example.com", "+91-9900000006");
            Map<String, Integer> staleVersionMap = new HashMap<>();
            staleVersionMap.put(targetSeat.getSeat().getSeatId(), staleVersion);

            try {
                show.lockSeats(
                        Collections.singletonList(targetSeat.getSeat().getSeatId()),
                        userF.getUserId(),
                        staleVersionMap,
                        10);
                System.out.println("[OptimisticLock] Frank succeeded (unexpected!)");
            } catch (SeatNotAvailableException e) {
                System.out.println("[OptimisticLock] Frank rejected: " + e.getMessage());
                System.out.println("[OptimisticLock] Stale version detected — no silent overwrite occurred.");
            }
        }

        // Cleanup
        system.shutdown();
        System.out.println("\n=== Demo Complete ===");
    }
}
```

---

## 8. Design Patterns Used

### 8.1 Strategy Pattern — Pricing

**Where**: `PricingStrategy` interface, `CategoryPricingStrategy`, `PeakHourPricingStrategy`.

**Why**: Pricing rules change frequently (weekends, holidays, early-bird discounts, loyalty discounts). The Strategy pattern lets us swap or compose pricing rules without modifying `Show` or `BookingService`. Adding a "Tuesday discount" is a new class, not a change to existing code (Open/Closed Principle).

**Alternative considered**: Simple if-else chain inside `Show`. Rejected because it puts unrelated business logic in the domain entity and makes testing harder.

### 8.2 Decorator Pattern — Pricing Chain

**Where**: `PeakHourPricingStrategy` wraps any `PricingStrategy`.

**Why**: Allows composing multiple pricing modifiers. You can stack a `PeakHourPricingStrategy` on top of a `CategoryPricingStrategy`, then add a `LoyaltyDiscountStrategy` on top of that — each decorator is unaware of others. This is more flexible than a single class with multiple flags.

### 8.3 Facade Pattern — `MovieTicketBookingSystem`

**Where**: `MovieTicketBookingSystem` is the single entry point for all client operations.

**Why**: Clients (controllers, CLI, tests) do not need to know about `BookingService`, `Show`, `PricingStrategy`, or their interactions. The facade simplifies the API surface and acts as the composition root.

### 8.4 Repository / Registry Pattern — in-memory maps

**Where**: `movies`, `theaters`, `shows` maps inside `MovieTicketBookingSystem`; `bookingStore` inside `BookingService`.

**Why**: Encapsulates data access behind a simple key lookup. In production, these maps are replaced by database DAO/Repository classes without changing the service logic.

### 8.5 Template Method (implicit) — Booking Lifecycle

**Where**: The sequence `initiateBooking` → `confirmBooking` → (optionally) `cancelBooking` in `BookingService` forms a fixed template with hook points.

### 8.6 Optimistic Locking (pattern, not GoF)

**Where**: `ShowSeat.version` + the version-check inside `Show.lockSeats()`.

**Why**: Avoids holding a database lock between the "read seat state" and "write seat state" operations. The show's `ReentrantLock` is held only for the duration of the in-memory state transition (microseconds), not during payment processing (seconds). This maximizes throughput.

---

## 9. Key Design Decisions and Trade-offs

### Decision 1: Per-Show Seat State vs. Per-Screen Seat State

**Choice**: Each `Show` maintains its own `Map<String, ShowSeat>`.

**Why**: If seat status were stored on `Seat` directly, a cancellation for the 9 PM show would incorrectly affect the 6 PM show on the same screen. Per-show state isolation is fundamental correctness.

**Trade-off**: Higher memory usage (N shows × M seats objects). Acceptable given typical show counts (< 10 shows/screen/day).

### Decision 2: ReentrantLock per Show vs. Global Lock

**Choice**: Each `Show` has its own `ReentrantLock`.

**Why**: A global lock would serialize all booking attempts across all shows in the entire system — catastrophic for throughput. A per-show lock means shows compete only within themselves. Two people booking seats for different shows never contend.

**Trade-off**: More lock objects, but the contention surface is minimized.

### Decision 3: Fair Lock (`new ReentrantLock(true)`)

**Choice**: Use a fair `ReentrantLock`.

**Why**: Under heavy load, an unfair lock can starve some threads indefinitely (thread starvation). With a fair lock, threads acquire in FIFO order. Users who clicked "book" first get priority.

**Trade-off**: Fair locks have slightly lower throughput than unfair locks due to ordering overhead. The UX improvement (first-come-first-served) justifies this.

### Decision 4: Separate Lock Phase from Confirmation Phase

**Choice**: `initiateBooking()` locks seats; `confirmBooking()` pays and confirms.

**Why**: Payment takes 1–3 seconds (network call). We must not hold the `ReentrantLock` during payment — that would block all other users trying to book the same show. Instead, we use the LOCKED seat status as the reservation mechanism and release the lock immediately after the seat state transition.

**Trade-off**: Between lock acquisition and payment completion, the seat is `LOCKED` (not `AVAILABLE`). Other users see it as unavailable. This is correct behavior — the seat is reserved during checkout.

### Decision 5: Optimistic Locking via Version Field

**Choice**: `ShowSeat.version` is checked before locking; mismatches cause immediate failure.

**Why**: Without this, a race could occur even with the `ReentrantLock`:
1. Thread A reads seat version=0 and decides to book.
2. Thread B acquires the lock, books the seat (version becomes 1), and releases the lock.
3. Thread A acquires the lock and overwrites Thread B's booking (because A only checked `isAvailable()` — which is now false, so A fails too). In practice, the AVAILABLE check prevents overwrite, but the version check provides an extra correctness layer and mirrors real database optimistic locking exactly.

**Trade-off**: Version checks add one comparison per seat per booking operation. Negligible.

### Decision 6: 10-Minute Seat Lock with Auto-Release

**Choice**: `ScheduledExecutorService` releases expired locks.

**Why**: Users frequently abandon carts (network issues, changed mind). Without auto-release, abandoned locks permanently prevent others from booking those seats.

**Trade-off (in-memory)**: If the JVM crashes, scheduled tasks are lost and seats remain "LOCKED" in the database. Production solution: use Redis with TTL — when the key expires, seats are released.

---

## 10. Extension Points

### 10.1 Add Notification Service

Add a `NotificationService` interface injected into `BookingService`:
```java
public interface NotificationService {
    void sendBookingConfirmation(Booking booking);
    void sendCancellationConfirmation(Booking booking, double refundAmount);
    void sendPaymentFailure(Booking booking);
}
```
Implement with `EmailNotificationService`, `SMSNotificationService` using the same interface. Call from `confirmBooking()` and `cancelBooking()`.

### 10.2 Add Group/Bulk Bookings

`initiateBooking()` already accepts a `List<String> seatIds` — it naturally supports multi-seat selection. For group discounts, add a `GroupDiscountStrategy` decorator to `PricingStrategy` that applies a percentage discount when `seatCount >= threshold`.

### 10.3 Add Loyalty / Coupon Discounts

Create a `CouponStrategy` implementing `PricingStrategy` that looks up a coupon code and applies a fixed or percentage discount. Chain it:
```java
PricingStrategy pricing = new CouponStrategy(
        new PeakHourPricingStrategy(
                new CategoryPricingStrategy(200, 350, 500), 1.3, showTime),
        "FIRST50", 0.5);
```

### 10.4 Add Waitlist

When a fully-booked show has a cancellation, notify waitlisted users. Add a `WaitlistService` that stores `(userId, showId)` pairs and subscribes to seat-release events from `BookingService`.

### 10.5 Distributed Lock for Multi-Node Deployment

Replace `ReentrantLock` in `Show` with a `DistributedLockManager` interface:
```java
public interface DistributedLockManager {
    boolean tryLock(String lockKey, long timeoutMs);
    void unlock(String lockKey);
}
```
Implement with Redis Redlock or ZooKeeper. `Show.lockSeats()` calls `distributedLock.tryLock("show:" + showId, 500)` instead of `lock.lock()`.

### 10.6 Event Sourcing for Booking State

Replace `bookingStore` with an event log. Each booking state transition (`SeatLocked`, `PaymentProcessed`, `BookingConfirmed`, `BookingCancelled`) is appended as an immutable event. The current state is derived by replaying events. This gives full audit trail and enables time-travel debugging.

### 10.7 Seat Recommendation Engine

Add a `SeatRecommendationService` that returns the "best available" seats based on user preferences (front row, center, near exit). Operates purely on `getAvailableSeats()` output — does not touch locking logic.

---

## 11. Interview Discussion Points

### Q1: "How do you prevent double-booking?"

**Answer**: Two layers:

1. **ReentrantLock per Show**: `Show.lockSeats()` acquires an exclusive lock before reading or writing seat status. Since only one thread holds the lock at a time, it is physically impossible for two threads to pass the `isAvailable()` check simultaneously for the same seat.

2. **Optimistic locking via version**: Even before the lock is acquired, each caller captures the seat's current version number. Inside the critical section, the version is re-verified. If it changed (meaning another thread snuck in), the request fails fast with a clear error. This mirrors the database-level `UPDATE seats SET status='LOCKED' WHERE seat_id=? AND version=? AND status='AVAILABLE'` pattern used in production.

### Q2: "What happens if 1000 users try to book the same seat simultaneously?"

**Answer**: The fair `ReentrantLock` queues all 1000 threads. The first one acquires the lock, verifies the seat is AVAILABLE, locks it, and releases the lock. The remaining 999 each acquire the lock in turn, find the seat is now LOCKED, and receive a `SeatNotAvailableException`. They are redirected to choose other seats. The lock is held for microseconds per thread, so the queue drains very quickly. Throughput is high even under extreme contention.

### Q3: "How is this different from a database-level solution?"

| In-Memory (this design) | Database-level |
|---|---|
| `ReentrantLock` per `Show` object | `SELECT ... FOR UPDATE` (pessimistic) or `UPDATE ... WHERE version=?` (optimistic) |
| Seat version in `ShowSeat.version` field | `version` column in `show_seats` table |
| Auto-release via `ScheduledExecutorService` | Redis TTL or scheduled SQL job |
| Single JVM only | Works across all JVM instances |

In production, the `Show` object is not held in memory long-term; the database is the source of truth. The in-memory lock is a per-request transaction-boundary mechanism.

### Q4: "Why not use `synchronized` instead of `ReentrantLock`?"

**Answer**: `synchronized` is simpler but less flexible:
- Cannot interrupt a waiting thread (`lock.lockInterruptibly()` can).
- Cannot implement fairness (FIFO ordering).
- Cannot try-lock with a timeout (`lock.tryLock(500, TimeUnit.MILLISECONDS)`).
- Cannot have separate condition variables per lock (needed for advanced patterns).

For a booking system, `tryLock` with timeout is particularly valuable: if a show is under heavy contention, a thread should fail fast rather than waiting indefinitely, improving user experience.

### Q5: "How would you scale this to handle 1 million concurrent users?"

**Answer**:
1. **Horizontal scaling**: Deploy multiple JVM instances behind a load balancer. Seat locking must move from `ReentrantLock` (JVM-local) to Redis Redlock (distributed). Each booking request is routed to any instance; the distributed lock ensures only one wins.
2. **Sharding**: Shard shows by `showId` across multiple Redis nodes or database shards. All requests for a given show hit the same shard, reducing cross-node contention.
3. **Read/write separation**: Seat availability reads (popular on the browsing page) go to read replicas; locking writes go to the primary.
4. **Seat lock queue**: Instead of threads contending on a lock, use a per-show queue backed by Kafka. Each seat-lock request is a message; a single consumer per show processes them sequentially — naturally serialized without explicit locking.

### Q6: "What's the impact of the 10-minute lock on user experience?"

**Answer**: It's a deliberate trade-off:
- **Too short** (< 3 min): Users who take time filling payment details lose their seats mid-checkout. Frustrating.
- **Too long** (> 15 min): Seats are withheld from other buyers for too long; popular shows appear sold out when they aren't.
- **10 minutes** matches industry standard (BookMyShow, Ticketmaster). The auto-release mechanism ensures abandoned locks don't permanently block seats.

An improvement: show a countdown timer in the UI and warn the user at 2 minutes remaining.

### Q7: "How does the cancellation refund policy work?"

**Answer**: It's a simple time-based rule applied at cancellation time:
```
hoursUntilShow = showTime - now (in hours)

if hoursUntilShow > 24  -> refund 100%
if 12 <= hoursUntilShow <= 24 -> refund 50%
if hoursUntilShow < 12  -> refund 0%
```
This is business logic in `BookingService.cancelBooking()`. To make it configurable (different rules per movie or theater), extract the refund policy into a `RefundPolicy` interface (another Strategy application).

---

## 12. Common Mistakes in This Design

### Mistake 1: Storing Seat Status Directly on `Seat` (Not on `ShowSeat`)

**What goes wrong**: If you put `SeatStatus` on the `Seat` class, then when Show A at 3 PM is cancelled and seats are released, they become AVAILABLE for Show B at 6 PM even if Show B's seats were already booked. The seat state is global but should be per-show.

**Correct approach**: `Seat` is immutable (just layout metadata). `ShowSeat` holds the per-show mutable state.

---

### Mistake 2: Holding the Lock During Payment Processing

**What goes wrong**:
```java
// WRONG — lock held across a 3-second payment call
lock.lock();
try {
    validateAndLockSeats();
    paymentService.processPayment(booking);  // 3 seconds!
    confirmSeats();
} finally {
    lock.unlock();
}
```
This serializes all booking attempts for the entire duration of payment. 1000 users each waiting 3 seconds = 3000 seconds of cumulative waiting time. The system grinds to a halt.

**Correct approach**: Lock, transition seats to LOCKED, unlock. Then process payment (outside the lock). Then lock again briefly to transition LOCKED -> BOOKED (or release if payment failed).

---

### Mistake 3: Not Handling Lock Expiry

**What goes wrong**: User starts checkout, then their browser crashes. Their `LOCKED` seats are never released. Nobody can buy those seats until the show is over.

**Correct approach**: Schedule auto-release after `LOCK_DURATION_MINUTES`. In production, use Redis TTL on the lock key.

---

### Mistake 4: Version Check After Lock Acquisition Without Re-Reading

**What goes wrong**:
```java
// WRONG — version captured outside lock is checked inside lock, but seat
// status not re-read inside the lock before comparing
lock.lock();
try {
    if (oldVersion == showSeat.getVersion()) {  // still stale check
        showSeat.lock(...);  // may still be wrong if status was LOCKED
    }
}
```
Even with the version check, you must still verify `isAvailable()` — the version tells you the seat was modified, but not *how* it was modified. Always check both: version match AND status == AVAILABLE.

---

### Mistake 5: Using a Single Global Lock

**What goes wrong**: `synchronized(this)` on the `MovieTicketBookingSystem` or a single `ReentrantLock` for all shows. Every booking request across every show blocks on the same lock. A show with 1000 concurrent users blocks a show with 1 user.

**Correct approach**: Lock granularity must be per-Show. Shows are independent; their seats do not overlap.

---

### Mistake 6: Forgetting to Release Seats When Payment Fails

**What goes wrong**:
```java
Payment payment = paymentService.processPayment(booking);
if (payment.getStatus() == PaymentStatus.SUCCESS) {
    show.confirmSeats(seatIds, userId);
    booking.confirm(payment);
}
// WRONG: payment failed path does nothing — seats remain LOCKED forever
```

**Correct approach**: The `else` branch must explicitly call `show.releaseSeats()`. The auto-release scheduler is a safety net, not the primary release mechanism.

---

### Mistake 7: Calculating Refund Amount After Applying It

**What goes wrong**: Calculating `refundAmount` based on `booking.getTotalAmount()` after the booking status has been changed to CANCELLED and payment already modified. The sequence matters — calculate first, then mutate.

**Correct approach**: Always determine the refund amount before changing any state. Then apply it.

---

### Mistake 8: Not Making the Show's Lock Fair

**What goes wrong**: An unfair `ReentrantLock` (the default) can cause **thread starvation**: under high contention, the same thread may repeatedly re-acquire the lock because it is already scheduled on the CPU, while other threads wait indefinitely.

**Correct approach**: `new ReentrantLock(true)` — fair lock guarantees FIFO ordering and prevents starvation at a small throughput cost.

---

### Mistake 9: Sharing the Same Pricing Strategy Instance Across Shows Without Thread Safety

**What goes wrong**: If a `PricingStrategy` implementation stores mutable state (e.g., a dynamic price cache), sharing one instance across shows creates a data race.

**Correct approach**: Either make pricing strategies stateless (pure functions of their inputs), or create a new instance per show.

---

### Mistake 10: Catching `InterruptedException` and Swallowing It

**What goes wrong**:
```java
try {
    startGate.await();
} catch (InterruptedException e) {
    // swallowed — thread doesn't know it was interrupted
}
```
If a thread is interrupted (e.g., system shutdown), swallowing the exception means the thread continues working instead of stopping cleanly.

**Correct approach**: Always restore the interrupt flag: `Thread.currentThread().interrupt()` after catching `InterruptedException`, or re-throw it.

---

*End of LLD Case Study: Movie Ticket Booking System*
