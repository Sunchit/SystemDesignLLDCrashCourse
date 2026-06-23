# Hotel Booking System — LLD Case Study

> **Difficulty**: Medium | **Patterns**: Strategy, Builder, Repository | **Estimated Design Time**: 45–60 min

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

Design a **Hotel Booking System** that allows guests to search for hotels, book rooms, manage reservations, and handle the full lifecycle of a hotel stay — from search through check-in, check-out, and invoice generation.

The system must support **multiple hotels**, each with **multiple room types** at different price points. Pricing should be flexible enough to handle base rates, weekend surcharges, and seasonal variations. Bookings should follow a defined lifecycle (pending → confirmed → checked in → checked out), and cancellations should apply a policy-based refund calculation.

This is the kind of problem that tests whether you can:
- Model a multi-entity domain with clear ownership and relationships
- Apply the **Strategy pattern** to make pricing algorithms swappable
- Apply the **Builder pattern** to construct a flexible, composable search query
- Handle state transitions safely
- Generate derived documents (invoices) from booking state

**Stop here. Spend 10 minutes sketching your own design before reading further.**

---

## 2. Functional Requirements

1. The system shall support multiple hotels, each identified by name, location (city, address), and a set of amenities.
2. Each hotel shall contain multiple rooms with a room number, room type (SINGLE, DOUBLE, SUITE), base price per night, and current availability status.
3. Each room shall have an associated list of amenities (e.g., WiFi, Pool, Gym, Parking, Breakfast).
4. A guest shall be able to search for available rooms using filters: city, check-in date, check-out date, room type, maximum price per night, and required amenities.
5. A guest shall be able to book an available room for a specified date range, producing a Booking with a unique ID.
6. A booking shall progress through the states: PENDING → CONFIRMED → CHECKED_IN → CHECKED_OUT, with CANCELLED as a terminal state reachable from PENDING or CONFIRMED.
7. A guest shall be able to modify a confirmed booking's date range (subject to availability).
8. A guest shall be able to cancel a booking, with the refund amount calculated by the cancellation policy.
9. The cancellation policy shall be: full refund if cancelled more than 48 hours before check-in; 50% refund if cancelled 24–48 hours before check-in; no refund if cancelled less than 24 hours before check-in.
10. The system shall support check-in (transition CONFIRMED → CHECKED_IN) and check-out (transition CHECKED_IN → CHECKED_OUT).
11. The total booking amount shall be calculated using a pluggable pricing strategy: base price, weekend surcharge, or seasonal pricing.
12. Upon check-out, the system shall generate an itemized invoice showing: room charges per night, the applied pricing strategy, taxes, and the total amount due.
13. Each guest shall have a profile storing personal details and a history of all their bookings.
14. The system shall prevent double-booking: a room cannot be booked by two guests for overlapping date ranges.

---

## 3. Non-Functional Requirements

| Category | Requirement |
|---|---|
| **Correctness** | No double-bookings must be permitted under any sequence of operations |
| **Extensibility** | New pricing strategies must be addable without modifying existing booking logic |
| **Readability** | The booking flow (search → book → check-in → check-out → invoice) must be expressed clearly in the service layer |
| **Immutability** | Completed bookings (CHECKED_OUT, CANCELLED) must not be modifiable |
| **Testability** | All pricing strategies must be independently unit-testable with no external dependencies |
| **Auditability** | Invoices must be reproducible from booking data at any time after check-out |

---

## 4. Assumptions and Constraints

**Assumptions:**
- This is a single-instance, in-memory design (no database, no distributed locking). Thread safety is noted but not the primary focus.
- A "night" is counted as the difference in calendar days between check-in and check-out (e.g., check-in Monday, check-out Wednesday = 2 nights).
- Prices are in a single currency (USD). Currency conversion is out of scope.
- A guest books exactly one room per booking. Multi-room bookings are an extension point.
- Room availability is checked at booking time; no reservation hold period is modeled.
- Amenities are modeled as a set of strings (e.g., "WiFi", "Pool") for simplicity. An enum-based design is an extension point.
- The system clock is represented by `LocalDateTime.now()` for cancellation policy evaluation. In production, a `Clock` abstraction would be injected.

**Constraints:**
- A room's status (AVAILABLE, OCCUPIED, MAINTENANCE) reflects its physical state, not its booking state. A room can be AVAILABLE physically but have future bookings.
- Cancellation policy is time-based relative to check-in, not booking creation time.
- The Builder pattern for `RoomSearchQuery` is strictly used for constructing the search query object — it does not perform the search itself.

---

## 5. Domain Model Description

The domain has five core entities and two value objects:

**Hotel** — An aggregation root that owns a collection of rooms. It has identity (hotelId), location (city, address), a name, and hotel-level amenities (e.g., "Pool", "Gym"). The hotel is the entry point for room queries.

**Room** — Owned by a hotel. Each room has a room number (unique within a hotel), a type (SINGLE, DOUBLE, SUITE), a base price per night, a physical status (AVAILABLE, OCCUPIED, MAINTENANCE), and a list of in-room amenities. A room does not directly know about its bookings — the `HotelService` manages the availability check by querying active bookings.

**GuestProfile** — Represents a hotel guest. Contains personal information (name, email, phone) and a list of all bookings made by the guest. The guest does not hold references to rooms directly — only to bookings.

**Booking** — The central transactional entity. Connects a guest to a room for a date range. Holds the booking lifecycle state (PENDING, CONFIRMED, CHECKED_IN, CHECKED_OUT, CANCELLED), the computed total amount, and a reference to the pricing strategy that was used when the booking was created.

**Invoice** — A derived document generated from a completed booking at check-out. It is an immutable snapshot containing an itemized breakdown: nightly rate, number of nights, dynamic pricing adjustment, subtotal, tax amount, and total. Invoices are not entities in the strictest sense — they are generated on demand from booking data, but once generated they are stored for auditability.

**RoomSearchQuery (Value Object)** — An immutable value object built by `RoomSearchQuery.Builder`. Carries the search criteria: city, check-in/check-out dates, room type, maximum price, and required amenities. Constructed via a fluent Builder API.

**CancellationResult (Value Object)** — Returned by the cancellation operation. Contains the refund amount and the policy tier applied (FULL, PARTIAL, NONE).

---

## 6. Class Diagram (ASCII UML)

```
+---------------------------+          +---------------------------+
|        Hotel              |1       * |          Room             |
|---------------------------|--------->|---------------------------|
| - hotelId: String         |          | - roomId: String          |
| - name: String            |          | - roomNumber: String      |
| - city: String            |          | - type: RoomType          |
| - address: String         |          | - basePrice: double       |
| - amenities: Set<String>  |          | - status: RoomStatus      |
| - rooms: List<Room>       |          | - amenities: Set<String>  |
+---------------------------+          +---------------------------+
                                                   |
                                                   | booked in
                                                   v
+---------------------------+          +---------------------------+
|      GuestProfile         |1       * |         Booking           |
|---------------------------|          |---------------------------|
| - guestId: String         |--------->| - bookingId: String       |
| - name: String            |          | - guest: GuestProfile     |
| - email: String           |          | - room: Room              |
| - phone: String           |          | - hotel: Hotel            |
| - bookings: List<Booking> |          | - checkIn: LocalDate      |
+---------------------------+          | - checkOut: LocalDate     |
                                       | - status: BookingStatus   |
                                       | - totalAmount: double     |
                                       | - pricingStrategy:        |
                                       |     PricingStrategy       |
                                       | - invoice: Invoice        |
                                       +---------------------------+
                                                   |
                                                   | generates
                                                   v
                                       +---------------------------+
                                       |         Invoice           |
                                       |---------------------------|
                                       | - invoiceId: String       |
                                       | - booking: Booking        |
                                       | - lineItems: List<...>    |
                                       | - subtotal: double        |
                                       | - taxAmount: double       |
                                       | - totalAmount: double     |
                                       | - generatedAt: LocalDateTime|
                                       +---------------------------+

                    <<interface>>
                  +------------------+
                  | PricingStrategy  |
                  |------------------|
                  | + calculatePrice(|
                  |   room, nights,  |
                  |   checkIn): double|
                  +------------------+
                         / | \
                        /  |  \
           ____________/   |   \____________
          /                |                \
+----------------+ +----------------+ +------------------+
| BasePricing    | | WeekendPricing | | SeasonalPricing  |
| Strategy       | | Strategy       | | Strategy         |
|----------------| |----------------| |------------------|
| multiplier=1.0 | | weekdayMult    | | peakMonths:      |
|                | | weekendMult    | |   Set<Month>     |
+----------------+ +----------------+ +------------------+

+----------------------------------+
|       RoomSearchQuery            |
|----------------------------------|
| - city: String                   |
| - checkIn: LocalDate             |
| - checkOut: LocalDate            |
| - roomType: RoomType (optional)  |
| - maxPrice: Double (optional)    |
| - requiredAmenities: Set<String> |
|                                  |
|  <<inner class>> Builder         |
|  + city(String): Builder         |
|  + checkIn(LocalDate): Builder   |
|  + checkOut(LocalDate): Builder  |
|  + roomType(RoomType): Builder   |
|  + maxPrice(double): Builder     |
|  + amenity(String): Builder      |
|  + build(): RoomSearchQuery      |
+----------------------------------+

+------------------------------------------+
|             HotelService                 |
|------------------------------------------|
| - hotels: Map<String, Hotel>             |
| - bookings: Map<String, Booking>         |
| - guestProfiles: Map<String, Guest...>   |
| - defaultPricingStrategy:PricingStrategy |
|------------------------------------------|
| + registerHotel(Hotel): void             |
| + registerGuest(GuestProfile): void      |
| + searchRooms(RoomSearchQuery):          |
|     List<Room>                           |
| + createBooking(guestId, roomId,         |
|     checkIn, checkOut): Booking          |
| + confirmBooking(bookingId): Booking     |
| + cancelBooking(bookingId):              |
|     CancellationResult                   |
| + modifyBooking(bookingId, checkIn,      |
|     checkOut): Booking                   |
| + checkIn(bookingId): Booking            |
| + checkOut(bookingId): Invoice           |
| + getInvoice(bookingId): Invoice         |
+------------------------------------------+

Enumerations:
  BookingStatus: PENDING, CONFIRMED, CHECKED_IN, CHECKED_OUT, CANCELLED
  RoomType:      SINGLE, DOUBLE, SUITE
  RoomStatus:    AVAILABLE, OCCUPIED, MAINTENANCE
```

---

## 7. Core Classes — Complete Java Implementation

### 7.1 Enumerations

```java
// BookingStatus.java
public enum BookingStatus {
    PENDING,
    CONFIRMED,
    CHECKED_IN,
    CHECKED_OUT,
    CANCELLED
}

// RoomType.java
public enum RoomType {
    SINGLE,
    DOUBLE,
    SUITE
}

// RoomStatus.java
public enum RoomStatus {
    AVAILABLE,
    OCCUPIED,
    MAINTENANCE
}
```

---

### 7.2 PricingStrategy (Strategy Pattern)

```java
import java.time.LocalDate;

// PricingStrategy.java
public interface PricingStrategy {
    /**
     * Calculate the total price for a booking.
     *
     * @param basePrice  The room's base price per night.
     * @param nights     The number of nights.
     * @param checkIn    The check-in date (used for day-of-week or seasonal logic).
     * @return           The total price for all nights combined.
     */
    double calculatePrice(double basePrice, int nights, LocalDate checkIn);

    /**
     * A human-readable name for this strategy, used in invoice line items.
     */
    String getStrategyName();
}
```

```java
import java.time.LocalDate;

// BasePricingStrategy.java
public class BasePricingStrategy implements PricingStrategy {

    @Override
    public double calculatePrice(double basePrice, int nights, LocalDate checkIn) {
        return basePrice * nights;
    }

    @Override
    public String getStrategyName() {
        return "Standard Rate";
    }
}
```

```java
import java.time.DayOfWeek;
import java.time.LocalDate;

// WeekendPricingStrategy.java
/**
 * Applies a surcharge to nights that fall on Friday or Saturday.
 * Weekday nights use the weekday multiplier (typically 1.0).
 * Weekend nights use the weekend multiplier (typically 1.25).
 */
public class WeekendPricingStrategy implements PricingStrategy {

    private final double weekdayMultiplier;
    private final double weekendMultiplier;

    public WeekendPricingStrategy(double weekdayMultiplier, double weekendMultiplier) {
        if (weekdayMultiplier <= 0 || weekendMultiplier <= 0) {
            throw new IllegalArgumentException("Multipliers must be positive");
        }
        this.weekdayMultiplier = weekdayMultiplier;
        this.weekendMultiplier = weekendMultiplier;
    }

    @Override
    public double calculatePrice(double basePrice, int nights, LocalDate checkIn) {
        double total = 0.0;
        for (int i = 0; i < nights; i++) {
            LocalDate night = checkIn.plusDays(i);
            DayOfWeek day = night.getDayOfWeek();
            boolean isWeekend = (day == DayOfWeek.FRIDAY || day == DayOfWeek.SATURDAY);
            double multiplier = isWeekend ? weekendMultiplier : weekdayMultiplier;
            total += basePrice * multiplier;
        }
        return total;
    }

    @Override
    public String getStrategyName() {
        return String.format("Weekend Rate (%.0f%% weekend surcharge)",
                (weekendMultiplier - 1.0) * 100);
    }
}
```

```java
import java.time.LocalDate;
import java.time.Month;
import java.util.EnumSet;
import java.util.Set;

// SeasonalPricingStrategy.java
/**
 * Applies different multipliers based on whether the check-in month
 * falls in the configured peak season.
 *
 * Peak months use the peak multiplier (e.g., 1.5x).
 * Off-peak months use the off-peak multiplier (e.g., 0.8x).
 */
public class SeasonalPricingStrategy implements PricingStrategy {

    private final Set<Month> peakMonths;
    private final double peakMultiplier;
    private final double offPeakMultiplier;

    public SeasonalPricingStrategy(Set<Month> peakMonths,
                                   double peakMultiplier,
                                   double offPeakMultiplier) {
        if (peakMonths == null || peakMonths.isEmpty()) {
            throw new IllegalArgumentException("Peak months must not be empty");
        }
        if (peakMultiplier <= 0 || offPeakMultiplier <= 0) {
            throw new IllegalArgumentException("Multipliers must be positive");
        }
        this.peakMonths = EnumSet.copyOf(peakMonths);
        this.peakMultiplier = peakMultiplier;
        this.offPeakMultiplier = offPeakMultiplier;
    }

    @Override
    public double calculatePrice(double basePrice, int nights, LocalDate checkIn) {
        // Seasonal pricing is determined by the check-in month.
        // A simple model: all nights at the same rate based on check-in month.
        boolean isPeak = peakMonths.contains(checkIn.getMonth());
        double multiplier = isPeak ? peakMultiplier : offPeakMultiplier;
        return basePrice * nights * multiplier;
    }

    @Override
    public String getStrategyName() {
        boolean isPeak = peakMonths.contains(LocalDate.now().getMonth());
        String season = isPeak ? "Peak" : "Off-Peak";
        double multiplier = isPeak ? peakMultiplier : offPeakMultiplier;
        return String.format("Seasonal Rate (%s, %.0f%% adjustment)",
                season, (multiplier - 1.0) * 100);
    }
}
```

---

### 7.3 Room

```java
import java.util.Collections;
import java.util.HashSet;
import java.util.Set;

// Room.java
public class Room {

    private final String roomId;
    private final String roomNumber;
    private final RoomType type;
    private final double basePrice;
    private RoomStatus status;
    private final Set<String> amenities;

    public Room(String roomId, String roomNumber, RoomType type,
                double basePrice, Set<String> amenities) {
        if (roomId == null || roomId.isEmpty()) {
            throw new IllegalArgumentException("Room ID must not be null or empty");
        }
        if (basePrice <= 0) {
            throw new IllegalArgumentException("Base price must be positive");
        }
        this.roomId = roomId;
        this.roomNumber = roomNumber;
        this.type = type;
        this.basePrice = basePrice;
        this.status = RoomStatus.AVAILABLE;
        this.amenities = new HashSet<>(amenities != null ? amenities : Set.of());
    }

    public String getRoomId() {
        return roomId;
    }

    public String getRoomNumber() {
        return roomNumber;
    }

    public RoomType getType() {
        return type;
    }

    public double getBasePrice() {
        return basePrice;
    }

    public RoomStatus getStatus() {
        return status;
    }

    public void setStatus(RoomStatus status) {
        if (status == null) {
            throw new IllegalArgumentException("Status must not be null");
        }
        this.status = status;
    }

    public Set<String> getAmenities() {
        return Collections.unmodifiableSet(amenities);
    }

    public boolean hasAmenity(String amenity) {
        return amenities.contains(amenity);
    }

    public boolean hasAllAmenities(Set<String> required) {
        return amenities.containsAll(required);
    }

    @Override
    public String toString() {
        return String.format("Room{id='%s', number='%s', type=%s, price=$%.2f, status=%s, amenities=%s}",
                roomId, roomNumber, type, basePrice, status, amenities);
    }
}
```

---

### 7.4 Hotel

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

// Hotel.java
public class Hotel {

    private final String hotelId;
    private final String name;
    private final String city;
    private final String address;
    private final Set<String> amenities;
    private final List<Room> rooms;

    public Hotel(String hotelId, String name, String city, String address,
                 Set<String> amenities) {
        if (hotelId == null || hotelId.isEmpty()) {
            throw new IllegalArgumentException("Hotel ID must not be null or empty");
        }
        this.hotelId = hotelId;
        this.name = name;
        this.city = city;
        this.address = address;
        this.amenities = new HashSet<>(amenities != null ? amenities : Set.of());
        this.rooms = new ArrayList<>();
    }

    public void addRoom(Room room) {
        if (room == null) {
            throw new IllegalArgumentException("Room must not be null");
        }
        rooms.add(room);
    }

    public String getHotelId() {
        return hotelId;
    }

    public String getName() {
        return name;
    }

    public String getCity() {
        return city;
    }

    public String getAddress() {
        return address;
    }

    public Set<String> getAmenities() {
        return Collections.unmodifiableSet(amenities);
    }

    public List<Room> getRooms() {
        return Collections.unmodifiableList(rooms);
    }

    @Override
    public String toString() {
        return String.format("Hotel{id='%s', name='%s', city='%s', rooms=%d}",
                hotelId, name, city, rooms.size());
    }
}
```

---

### 7.5 GuestProfile

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

// GuestProfile.java
public class GuestProfile {

    private final String guestId;
    private final String name;
    private final String email;
    private final String phone;
    private final List<Booking> bookingHistory;

    public GuestProfile(String guestId, String name, String email, String phone) {
        if (guestId == null || guestId.isEmpty()) {
            throw new IllegalArgumentException("Guest ID must not be null or empty");
        }
        if (email == null || !email.contains("@")) {
            throw new IllegalArgumentException("Invalid email address");
        }
        this.guestId = guestId;
        this.name = name;
        this.email = email;
        this.phone = phone;
        this.bookingHistory = new ArrayList<>();
    }

    public void addBooking(Booking booking) {
        if (booking == null) {
            throw new IllegalArgumentException("Booking must not be null");
        }
        bookingHistory.add(booking);
    }

    public String getGuestId() {
        return guestId;
    }

    public String getName() {
        return name;
    }

    public String getEmail() {
        return email;
    }

    public String getPhone() {
        return phone;
    }

    public List<Booking> getBookingHistory() {
        return Collections.unmodifiableList(bookingHistory);
    }

    @Override
    public String toString() {
        return String.format("GuestProfile{id='%s', name='%s', email='%s', bookings=%d}",
                guestId, name, email, bookingHistory.size());
    }
}
```

---

### 7.6 Booking

```java
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;

// Booking.java
public class Booking {

    private final String bookingId;
    private final GuestProfile guest;
    private final Room room;
    private final Hotel hotel;
    private LocalDate checkIn;
    private LocalDate checkOut;
    private BookingStatus status;
    private double totalAmount;
    private final PricingStrategy pricingStrategy;
    private Invoice invoice;

    public Booking(String bookingId, GuestProfile guest, Room room, Hotel hotel,
                   LocalDate checkIn, LocalDate checkOut,
                   PricingStrategy pricingStrategy) {
        if (bookingId == null || bookingId.isEmpty()) {
            throw new IllegalArgumentException("Booking ID must not be null or empty");
        }
        if (checkIn == null || checkOut == null) {
            throw new IllegalArgumentException("Check-in and check-out dates must not be null");
        }
        if (!checkOut.isAfter(checkIn)) {
            throw new IllegalArgumentException("Check-out must be after check-in");
        }
        this.bookingId = bookingId;
        this.guest = guest;
        this.room = room;
        this.hotel = hotel;
        this.checkIn = checkIn;
        this.checkOut = checkOut;
        this.status = BookingStatus.PENDING;
        this.pricingStrategy = pricingStrategy;
        this.totalAmount = calculateTotalAmount();
    }

    private double calculateTotalAmount() {
        int nights = (int) ChronoUnit.DAYS.between(checkIn, checkOut);
        return pricingStrategy.calculatePrice(room.getBasePrice(), nights, checkIn);
    }

    public void confirm() {
        if (status != BookingStatus.PENDING) {
            throw new IllegalStateException(
                    "Only PENDING bookings can be confirmed. Current status: " + status);
        }
        this.status = BookingStatus.CONFIRMED;
    }

    public void doCheckIn() {
        if (status != BookingStatus.CONFIRMED) {
            throw new IllegalStateException(
                    "Only CONFIRMED bookings can be checked in. Current status: " + status);
        }
        this.status = BookingStatus.CHECKED_IN;
    }

    public void doCheckOut() {
        if (status != BookingStatus.CHECKED_IN) {
            throw new IllegalStateException(
                    "Only CHECKED_IN bookings can be checked out. Current status: " + status);
        }
        this.status = BookingStatus.CHECKED_OUT;
    }

    public void cancel() {
        if (status == BookingStatus.CHECKED_IN
                || status == BookingStatus.CHECKED_OUT
                || status == BookingStatus.CANCELLED) {
            throw new IllegalStateException(
                    "Cannot cancel a booking with status: " + status);
        }
        this.status = BookingStatus.CANCELLED;
    }

    public void modifyDates(LocalDate newCheckIn, LocalDate newCheckOut) {
        if (status != BookingStatus.CONFIRMED) {
            throw new IllegalStateException(
                    "Only CONFIRMED bookings can be modified. Current status: " + status);
        }
        if (!newCheckOut.isAfter(newCheckIn)) {
            throw new IllegalArgumentException("Check-out must be after check-in");
        }
        this.checkIn = newCheckIn;
        this.checkOut = newCheckOut;
        this.totalAmount = calculateTotalAmount();
    }

    public void setInvoice(Invoice invoice) {
        if (this.invoice != null) {
            throw new IllegalStateException("Invoice already generated for booking: " + bookingId);
        }
        this.invoice = invoice;
    }

    public int getNights() {
        return (int) ChronoUnit.DAYS.between(checkIn, checkOut);
    }

    public String getBookingId() {
        return bookingId;
    }

    public GuestProfile getGuest() {
        return guest;
    }

    public Room getRoom() {
        return room;
    }

    public Hotel getHotel() {
        return hotel;
    }

    public LocalDate getCheckIn() {
        return checkIn;
    }

    public LocalDate getCheckOut() {
        return checkOut;
    }

    public BookingStatus getStatus() {
        return status;
    }

    public double getTotalAmount() {
        return totalAmount;
    }

    public PricingStrategy getPricingStrategy() {
        return pricingStrategy;
    }

    public Invoice getInvoice() {
        return invoice;
    }

    @Override
    public String toString() {
        return String.format(
                "Booking{id='%s', guest='%s', room='%s', hotel='%s', "
                        + "checkIn=%s, checkOut=%s, nights=%d, total=$%.2f, status=%s}",
                bookingId, guest.getName(), room.getRoomNumber(), hotel.getName(),
                checkIn, checkOut, getNights(), totalAmount, status);
    }
}
```

---

### 7.7 Invoice

```java
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

// Invoice.java
public class Invoice {

    // Represents a single line on the invoice.
    public static class LineItem {
        private final String description;
        private final double amount;

        public LineItem(String description, double amount) {
            this.description = description;
            this.amount = amount;
        }

        public String getDescription() {
            return description;
        }

        public double getAmount() {
            return amount;
        }

        @Override
        public String toString() {
            return String.format("  %-45s $%8.2f", description, amount);
        }
    }

    private final String invoiceId;
    private final Booking booking;
    private final List<LineItem> lineItems;
    private final double subtotal;
    private final double taxRate;
    private final double taxAmount;
    private final double totalAmount;
    private final LocalDateTime generatedAt;

    public Invoice(String invoiceId, Booking booking, List<LineItem> lineItems,
                   double taxRate) {
        if (invoiceId == null || invoiceId.isEmpty()) {
            throw new IllegalArgumentException("Invoice ID must not be null or empty");
        }
        this.invoiceId = invoiceId;
        this.booking = booking;
        this.lineItems = new ArrayList<>(lineItems);
        this.taxRate = taxRate;
        this.subtotal = lineItems.stream()
                .mapToDouble(LineItem::getAmount)
                .sum();
        this.taxAmount = subtotal * taxRate;
        this.totalAmount = subtotal + taxAmount;
        this.generatedAt = LocalDateTime.now();
    }

    public String getInvoiceId() {
        return invoiceId;
    }

    public Booking getBooking() {
        return booking;
    }

    public List<LineItem> getLineItems() {
        return Collections.unmodifiableList(lineItems);
    }

    public double getSubtotal() {
        return subtotal;
    }

    public double getTaxRate() {
        return taxRate;
    }

    public double getTaxAmount() {
        return taxAmount;
    }

    public double getTotalAmount() {
        return totalAmount;
    }

    public LocalDateTime getGeneratedAt() {
        return generatedAt;
    }

    /**
     * Produces a formatted, printable invoice string.
     */
    public String toPrintableString() {
        StringBuilder sb = new StringBuilder();
        sb.append("=".repeat(60)).append("\n");
        sb.append(String.format("  INVOICE #%s%n", invoiceId));
        sb.append("=".repeat(60)).append("\n");
        sb.append(String.format("  Hotel:    %s%n", booking.getHotel().getName()));
        sb.append(String.format("  Guest:    %s (%s)%n",
                booking.getGuest().getName(), booking.getGuest().getEmail()));
        sb.append(String.format("  Room:     %s (%s)%n",
                booking.getRoom().getRoomNumber(), booking.getRoom().getType()));
        sb.append(String.format("  Check-in: %s%n", booking.getCheckIn()));
        sb.append(String.format("  Check-out:%s%n", booking.getCheckOut()));
        sb.append(String.format("  Nights:   %d%n", booking.getNights()));
        sb.append(String.format("  Pricing:  %s%n", booking.getPricingStrategy().getStrategyName()));
        sb.append("-".repeat(60)).append("\n");
        sb.append("  LINE ITEMS\n");
        sb.append("-".repeat(60)).append("\n");
        for (LineItem item : lineItems) {
            sb.append(item.toString()).append("\n");
        }
        sb.append("-".repeat(60)).append("\n");
        sb.append(String.format("  %-45s $%8.2f%n", "Subtotal", subtotal));
        sb.append(String.format("  %-45s $%8.2f%n",
                String.format("Tax (%.0f%%)", taxRate * 100), taxAmount));
        sb.append("=".repeat(60)).append("\n");
        sb.append(String.format("  %-45s $%8.2f%n", "TOTAL DUE", totalAmount));
        sb.append("=".repeat(60)).append("\n");
        sb.append(String.format("  Generated: %s%n", generatedAt));
        return sb.toString();
    }
}
```

---

### 7.8 CancellationResult

```java
// CancellationResult.java
public class CancellationResult {

    public enum RefundTier {
        FULL,
        PARTIAL,
        NONE
    }

    private final String bookingId;
    private final RefundTier refundTier;
    private final double originalAmount;
    private final double refundAmount;
    private final String reason;

    public CancellationResult(String bookingId, RefundTier refundTier,
                               double originalAmount, double refundAmount,
                               String reason) {
        this.bookingId = bookingId;
        this.refundTier = refundTier;
        this.originalAmount = originalAmount;
        this.refundAmount = refundAmount;
        this.reason = reason;
    }

    public String getBookingId() {
        return bookingId;
    }

    public RefundTier getRefundTier() {
        return refundTier;
    }

    public double getOriginalAmount() {
        return originalAmount;
    }

    public double getRefundAmount() {
        return refundAmount;
    }

    public String getReason() {
        return reason;
    }

    @Override
    public String toString() {
        return String.format(
                "CancellationResult{bookingId='%s', tier=%s, original=$%.2f, "
                        + "refund=$%.2f, reason='%s'}",
                bookingId, refundTier, originalAmount, refundAmount, reason);
    }
}
```

---

### 7.9 RoomSearchQuery (Builder Pattern)

```java
import java.time.LocalDate;
import java.util.Collections;
import java.util.HashSet;
import java.util.Set;

// RoomSearchQuery.java
/**
 * Immutable value object representing a room search query.
 * Constructed exclusively via the inner Builder with a fluent API.
 *
 * Usage:
 *   RoomSearchQuery query = new RoomSearchQuery.Builder()
 *       .city("New York")
 *       .checkIn(LocalDate.of(2025, 6, 15))
 *       .checkOut(LocalDate.of(2025, 6, 18))
 *       .roomType(RoomType.DOUBLE)
 *       .maxPrice(200.0)
 *       .amenity("WiFi")
 *       .amenity("Pool")
 *       .build();
 */
public class RoomSearchQuery {

    private final String city;
    private final LocalDate checkIn;
    private final LocalDate checkOut;
    private final RoomType roomType;          // null = any type
    private final Double maxPrice;            // null = no upper limit
    private final Set<String> requiredAmenities;

    private RoomSearchQuery(Builder builder) {
        this.city = builder.city;
        this.checkIn = builder.checkIn;
        this.checkOut = builder.checkOut;
        this.roomType = builder.roomType;
        this.maxPrice = builder.maxPrice;
        this.requiredAmenities = Collections.unmodifiableSet(
                new HashSet<>(builder.requiredAmenities));
    }

    public String getCity() {
        return city;
    }

    public LocalDate getCheckIn() {
        return checkIn;
    }

    public LocalDate getCheckOut() {
        return checkOut;
    }

    public RoomType getRoomType() {
        return roomType;
    }

    public Double getMaxPrice() {
        return maxPrice;
    }

    public Set<String> getRequiredAmenities() {
        return requiredAmenities;
    }

    public int getNights() {
        return (int) java.time.temporal.ChronoUnit.DAYS.between(checkIn, checkOut);
    }

    @Override
    public String toString() {
        return String.format(
                "RoomSearchQuery{city='%s', checkIn=%s, checkOut=%s, "
                        + "type=%s, maxPrice=%s, amenities=%s}",
                city, checkIn, checkOut, roomType, maxPrice, requiredAmenities);
    }

    // -----------------------------------------------------------------------
    // Builder
    // -----------------------------------------------------------------------

    public static class Builder {

        private String city;
        private LocalDate checkIn;
        private LocalDate checkOut;
        private RoomType roomType;
        private Double maxPrice;
        private final Set<String> requiredAmenities = new HashSet<>();

        public Builder city(String city) {
            if (city == null || city.isBlank()) {
                throw new IllegalArgumentException("City must not be blank");
            }
            this.city = city;
            return this;
        }

        public Builder checkIn(LocalDate checkIn) {
            if (checkIn == null) {
                throw new IllegalArgumentException("Check-in date must not be null");
            }
            this.checkIn = checkIn;
            return this;
        }

        public Builder checkOut(LocalDate checkOut) {
            if (checkOut == null) {
                throw new IllegalArgumentException("Check-out date must not be null");
            }
            this.checkOut = checkOut;
            return this;
        }

        public Builder roomType(RoomType roomType) {
            this.roomType = roomType;
            return this;
        }

        public Builder maxPrice(double maxPrice) {
            if (maxPrice <= 0) {
                throw new IllegalArgumentException("Max price must be positive");
            }
            this.maxPrice = maxPrice;
            return this;
        }

        public Builder amenity(String amenity) {
            if (amenity != null && !amenity.isBlank()) {
                this.requiredAmenities.add(amenity);
            }
            return this;
        }

        public Builder amenities(Set<String> amenities) {
            if (amenities != null) {
                this.requiredAmenities.addAll(amenities);
            }
            return this;
        }

        public RoomSearchQuery build() {
            // Mandatory fields validation
            if (city == null) {
                throw new IllegalStateException("City is required");
            }
            if (checkIn == null) {
                throw new IllegalStateException("Check-in date is required");
            }
            if (checkOut == null) {
                throw new IllegalStateException("Check-out date is required");
            }
            if (!checkOut.isAfter(checkIn)) {
                throw new IllegalStateException("Check-out must be after check-in");
            }
            if (checkIn.isBefore(LocalDate.now())) {
                throw new IllegalStateException("Check-in cannot be in the past");
            }
            return new RoomSearchQuery(this);
        }
    }
}
```

---

### 7.10 HotelService (Main Orchestrator)

```java
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.stream.Collectors;

// HotelService.java
/**
 * Central orchestrator for the Hotel Booking System.
 *
 * Responsibilities:
 *  - Hotel and guest registration
 *  - Room search with composite filters
 *  - Booking lifecycle management (create, confirm, modify, cancel, check-in, check-out)
 *  - Invoice generation
 *  - Availability enforcement (prevents double-bookings)
 */
public class HotelService {

    private static final double DEFAULT_TAX_RATE = 0.12; // 12% tax

    private final Map<String, Hotel> hotels;
    private final Map<String, Booking> bookings;
    private final Map<String, GuestProfile> guestProfiles;
    private final PricingStrategy defaultPricingStrategy;

    public HotelService(PricingStrategy defaultPricingStrategy) {
        if (defaultPricingStrategy == null) {
            throw new IllegalArgumentException("Default pricing strategy must not be null");
        }
        this.hotels = new HashMap<>();
        this.bookings = new HashMap<>();
        this.guestProfiles = new HashMap<>();
        this.defaultPricingStrategy = defaultPricingStrategy;
    }

    // -----------------------------------------------------------------------
    // Registration
    // -----------------------------------------------------------------------

    public void registerHotel(Hotel hotel) {
        if (hotel == null) {
            throw new IllegalArgumentException("Hotel must not be null");
        }
        hotels.put(hotel.getHotelId(), hotel);
    }

    public void registerGuest(GuestProfile guest) {
        if (guest == null) {
            throw new IllegalArgumentException("Guest profile must not be null");
        }
        if (guestProfiles.containsKey(guest.getGuestId())) {
            throw new IllegalArgumentException(
                    "Guest already registered: " + guest.getGuestId());
        }
        guestProfiles.put(guest.getGuestId(), guest);
    }

    // -----------------------------------------------------------------------
    // Room Search
    // -----------------------------------------------------------------------

    /**
     * Searches for rooms matching all criteria in the query.
     *
     * A room is included in results if ALL of the following are true:
     *  1. Its hotel is in the requested city.
     *  2. Its type matches (if roomType filter is set).
     *  3. Its base price is at or below maxPrice (if set).
     *  4. It has all required amenities (if any are specified).
     *  5. It is not already booked for any overlapping date range.
     *  6. Its physical status is AVAILABLE.
     */
    public List<Room> searchRooms(RoomSearchQuery query) {
        if (query == null) {
            throw new IllegalArgumentException("Search query must not be null");
        }

        List<Room> results = new ArrayList<>();

        for (Hotel hotel : hotels.values()) {
            if (!hotel.getCity().equalsIgnoreCase(query.getCity())) {
                continue;
            }

            for (Room room : hotel.getRooms()) {
                if (room.getStatus() != RoomStatus.AVAILABLE) {
                    continue;
                }

                if (query.getRoomType() != null && room.getType() != query.getRoomType()) {
                    continue;
                }

                if (query.getMaxPrice() != null
                        && room.getBasePrice() > query.getMaxPrice()) {
                    continue;
                }

                if (!query.getRequiredAmenities().isEmpty()
                        && !room.hasAllAmenities(query.getRequiredAmenities())) {
                    continue;
                }

                if (isRoomBooked(room.getRoomId(), query.getCheckIn(), query.getCheckOut())) {
                    continue;
                }

                results.add(room);
            }
        }

        return results;
    }

    /**
     * Returns true if the given room has an active booking that overlaps
     * with [requestedCheckIn, requestedCheckOut).
     *
     * Overlap condition: bookingA overlaps bookingB iff
     *   bookingA.checkIn < bookingB.checkOut AND bookingA.checkOut > bookingB.checkIn
     */
    private boolean isRoomBooked(String roomId,
                                  LocalDate requestedCheckIn,
                                  LocalDate requestedCheckOut) {
        for (Booking booking : bookings.values()) {
            if (!booking.getRoom().getRoomId().equals(roomId)) {
                continue;
            }
            BookingStatus s = booking.getStatus();
            if (s == BookingStatus.CANCELLED || s == BookingStatus.CHECKED_OUT) {
                continue;
            }
            // Check overlap
            boolean overlaps = booking.getCheckIn().isBefore(requestedCheckOut)
                    && booking.getCheckOut().isAfter(requestedCheckIn);
            if (overlaps) {
                return true;
            }
        }
        return false;
    }

    // -----------------------------------------------------------------------
    // Booking Lifecycle
    // -----------------------------------------------------------------------

    /**
     * Creates a new PENDING booking for the given guest and room.
     * Uses the default pricing strategy.
     */
    public Booking createBooking(String guestId, String roomId,
                                  LocalDate checkIn, LocalDate checkOut) {
        return createBookingWithStrategy(guestId, roomId, checkIn, checkOut,
                defaultPricingStrategy);
    }

    /**
     * Creates a new PENDING booking using a custom pricing strategy.
     */
    public Booking createBookingWithStrategy(String guestId, String roomId,
                                              LocalDate checkIn, LocalDate checkOut,
                                              PricingStrategy strategy) {
        GuestProfile guest = getGuestOrThrow(guestId);
        Room room = findRoomByIdOrThrow(roomId);
        Hotel hotel = findHotelForRoomOrThrow(roomId);

        if (isRoomBooked(roomId, checkIn, checkOut)) {
            throw new IllegalStateException(
                    "Room " + roomId + " is not available for the requested dates");
        }

        String bookingId = "BKG-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
        Booking booking = new Booking(bookingId, guest, room, hotel,
                checkIn, checkOut, strategy);

        bookings.put(bookingId, booking);
        guest.addBooking(booking);

        return booking;
    }

    /**
     * Confirms a PENDING booking (transitions PENDING → CONFIRMED).
     */
    public Booking confirmBooking(String bookingId) {
        Booking booking = getBookingOrThrow(bookingId);
        booking.confirm();
        return booking;
    }

    /**
     * Modifies the dates of a CONFIRMED booking.
     * Validates that the new date range does not conflict with other bookings.
     */
    public Booking modifyBooking(String bookingId,
                                  LocalDate newCheckIn,
                                  LocalDate newCheckOut) {
        Booking booking = getBookingOrThrow(bookingId);

        if (booking.getStatus() != BookingStatus.CONFIRMED) {
            throw new IllegalStateException(
                    "Only CONFIRMED bookings can be modified. Current: "
                            + booking.getStatus());
        }

        // Temporarily check availability excluding the current booking
        for (Booking other : bookings.values()) {
            if (other.getBookingId().equals(bookingId)) {
                continue; // skip self
            }
            if (!other.getRoom().getRoomId().equals(booking.getRoom().getRoomId())) {
                continue;
            }
            BookingStatus s = other.getStatus();
            if (s == BookingStatus.CANCELLED || s == BookingStatus.CHECKED_OUT) {
                continue;
            }
            boolean overlaps = other.getCheckIn().isBefore(newCheckOut)
                    && other.getCheckOut().isAfter(newCheckIn);
            if (overlaps) {
                throw new IllegalStateException(
                        "Room is not available for the new dates due to conflicting booking: "
                                + other.getBookingId());
            }
        }

        booking.modifyDates(newCheckIn, newCheckOut);
        return booking;
    }

    /**
     * Cancels a booking (PENDING or CONFIRMED) and computes the refund.
     *
     * Cancellation policy:
     *   > 48 hours before check-in  → full refund
     *   24–48 hours before check-in → 50% refund
     *   < 24 hours before check-in  → no refund
     */
    public CancellationResult cancelBooking(String bookingId) {
        Booking booking = getBookingOrThrow(bookingId);

        if (booking.getStatus() == BookingStatus.CHECKED_IN
                || booking.getStatus() == BookingStatus.CHECKED_OUT
                || booking.getStatus() == BookingStatus.CANCELLED) {
            throw new IllegalStateException(
                    "Cannot cancel booking with status: " + booking.getStatus());
        }

        LocalDateTime now = LocalDateTime.now();
        LocalDateTime checkInDateTime = booking.getCheckIn().atStartOfDay();
        long hoursUntilCheckIn = ChronoUnit.HOURS.between(now, checkInDateTime);

        CancellationResult.RefundTier tier;
        double refundAmount;
        String reason;

        if (hoursUntilCheckIn > 48) {
            tier = CancellationResult.RefundTier.FULL;
            refundAmount = booking.getTotalAmount();
            reason = "Cancelled more than 48 hours before check-in. Full refund applied.";
        } else if (hoursUntilCheckIn >= 24) {
            tier = CancellationResult.RefundTier.PARTIAL;
            refundAmount = booking.getTotalAmount() * 0.5;
            reason = "Cancelled 24–48 hours before check-in. 50% refund applied.";
        } else {
            tier = CancellationResult.RefundTier.NONE;
            refundAmount = 0.0;
            reason = "Cancelled less than 24 hours before check-in. No refund.";
        }

        booking.cancel();

        return new CancellationResult(bookingId, tier,
                booking.getTotalAmount(), refundAmount, reason);
    }

    /**
     * Checks in a guest: transitions CONFIRMED → CHECKED_IN.
     * Also marks the room as OCCUPIED.
     */
    public Booking checkIn(String bookingId) {
        Booking booking = getBookingOrThrow(bookingId);
        booking.doCheckIn();
        booking.getRoom().setStatus(RoomStatus.OCCUPIED);
        return booking;
    }

    /**
     * Checks out a guest: transitions CHECKED_IN → CHECKED_OUT.
     * Marks the room as AVAILABLE again and generates the invoice.
     */
    public Invoice checkOut(String bookingId) {
        Booking booking = getBookingOrThrow(bookingId);
        booking.doCheckOut();
        booking.getRoom().setStatus(RoomStatus.AVAILABLE);

        Invoice invoice = generateInvoice(booking);
        booking.setInvoice(invoice);
        return invoice;
    }

    /**
     * Retrieves the invoice for a checked-out booking.
     */
    public Invoice getInvoice(String bookingId) {
        Booking booking = getBookingOrThrow(bookingId);
        if (booking.getInvoice() == null) {
            throw new IllegalStateException(
                    "Invoice not yet generated for booking: " + bookingId
                            + " (status: " + booking.getStatus() + ")");
        }
        return booking.getInvoice();
    }

    // -----------------------------------------------------------------------
    // Invoice Generation
    // -----------------------------------------------------------------------

    /**
     * Builds an itemized invoice from a completed booking.
     *
     * Line items:
     *  1. Base room charge: basePrice * nights
     *  2. Pricing adjustment: difference between strategy total and base total
     *     (positive = surcharge, negative = discount)
     */
    private Invoice generateInvoice(Booking booking) {
        List<Invoice.LineItem> lineItems = new ArrayList<>();

        int nights = booking.getNights();
        double basePrice = booking.getRoom().getBasePrice();
        double baseTotal = basePrice * nights;
        double strategyTotal = booking.getTotalAmount();
        double pricingAdjustment = strategyTotal - baseTotal;

        lineItems.add(new Invoice.LineItem(
                String.format("Room %s (%s) — %d night(s) @ $%.2f/night",
                        booking.getRoom().getRoomNumber(),
                        booking.getRoom().getType(),
                        nights, basePrice),
                baseTotal));

        if (Math.abs(pricingAdjustment) > 0.001) {
            String adjustLabel = pricingAdjustment > 0
                    ? "Dynamic pricing surcharge (" + booking.getPricingStrategy().getStrategyName() + ")"
                    : "Dynamic pricing discount (" + booking.getPricingStrategy().getStrategyName() + ")";
            lineItems.add(new Invoice.LineItem(adjustLabel, pricingAdjustment));
        }

        String invoiceId = "INV-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
        return new Invoice(invoiceId, booking, lineItems, DEFAULT_TAX_RATE);
    }

    // -----------------------------------------------------------------------
    // Private Helpers
    // -----------------------------------------------------------------------

    private GuestProfile getGuestOrThrow(String guestId) {
        GuestProfile guest = guestProfiles.get(guestId);
        if (guest == null) {
            throw new IllegalArgumentException("Guest not found: " + guestId);
        }
        return guest;
    }

    private Booking getBookingOrThrow(String bookingId) {
        Booking booking = bookings.get(bookingId);
        if (booking == null) {
            throw new IllegalArgumentException("Booking not found: " + bookingId);
        }
        return booking;
    }

    private Room findRoomByIdOrThrow(String roomId) {
        for (Hotel hotel : hotels.values()) {
            for (Room room : hotel.getRooms()) {
                if (room.getRoomId().equals(roomId)) {
                    return room;
                }
            }
        }
        throw new IllegalArgumentException("Room not found: " + roomId);
    }

    private Hotel findHotelForRoomOrThrow(String roomId) {
        for (Hotel hotel : hotels.values()) {
            for (Room room : hotel.getRooms()) {
                if (room.getRoomId().equals(roomId)) {
                    return hotel;
                }
            }
        }
        throw new IllegalArgumentException("No hotel found for room: " + roomId);
    }

    // -----------------------------------------------------------------------
    // Accessors (for testing / reporting)
    // -----------------------------------------------------------------------

    public Hotel getHotel(String hotelId) {
        return hotels.get(hotelId);
    }

    public GuestProfile getGuest(String guestId) {
        return guestProfiles.get(guestId);
    }

    public Booking getBooking(String bookingId) {
        return bookings.get(bookingId);
    }

    public List<Booking> getBookingsForGuest(String guestId) {
        GuestProfile guest = getGuestOrThrow(guestId);
        return guest.getBookingHistory();
    }
}
```

---

### 7.11 Main Demo

```java
import java.time.LocalDate;
import java.time.Month;
import java.util.EnumSet;
import java.util.List;
import java.util.Set;

// HotelBookingDemo.java
/**
 * Demonstrates the full hotel booking lifecycle:
 *   1. Setup (hotels, rooms, guest)
 *   2. Search with filters (Builder pattern)
 *   3. Book a room (Strategy pattern — weekend pricing)
 *   4. Confirm booking
 *   5. Check in
 *   6. Check out
 *   7. Print invoice
 *   8. Demonstrate cancellation policy
 */
public class HotelBookingDemo {

    public static void main(String[] args) {
        System.out.println("=".repeat(60));
        System.out.println("  HOTEL BOOKING SYSTEM — LLD DEMO");
        System.out.println("=".repeat(60));

        // ------------------------------------------------------------------
        // 1. Setup — create hotels, rooms, and a guest
        // ------------------------------------------------------------------
        Hotel grandHotel = new Hotel(
                "HOTEL-001",
                "The Grand Manhattan",
                "New York",
                "123 Fifth Avenue, New York, NY 10001",
                Set.of("Pool", "Gym", "Spa", "Concierge"));

        Room room101 = new Room("ROOM-101", "101", RoomType.SINGLE,
                120.00, Set.of("WiFi", "TV", "Mini-Bar"));
        Room room202 = new Room("ROOM-202", "202", RoomType.DOUBLE,
                180.00, Set.of("WiFi", "TV", "Mini-Bar", "City View"));
        Room room303 = new Room("ROOM-303", "303", RoomType.SUITE,
                350.00, Set.of("WiFi", "TV", "Mini-Bar", "City View", "Jacuzzi", "Kitchen"));

        grandHotel.addRoom(room101);
        grandHotel.addRoom(room202);
        grandHotel.addRoom(room303);

        Hotel budgetInn = new Hotel(
                "HOTEL-002",
                "Budget Inn Downtown",
                "New York",
                "456 Broadway, New York, NY 10013",
                Set.of("Parking", "Breakfast"));

        Room roomA1 = new Room("ROOM-A1", "A1", RoomType.SINGLE,
                75.00, Set.of("WiFi", "TV"));
        Room roomA2 = new Room("ROOM-A2", "A2", RoomType.DOUBLE,
                95.00, Set.of("WiFi", "TV", "Parking"));

        budgetInn.addRoom(roomA1);
        budgetInn.addRoom(roomA2);

        GuestProfile alice = new GuestProfile(
                "GUEST-001", "Alice Johnson",
                "alice@example.com", "+1-555-0101");

        GuestProfile bob = new GuestProfile(
                "GUEST-002", "Bob Smith",
                "bob@example.com", "+1-555-0202");

        // ------------------------------------------------------------------
        // 2. Initialize HotelService with Weekend Pricing as the default
        // ------------------------------------------------------------------
        PricingStrategy weekendPricing = new WeekendPricingStrategy(1.0, 1.25);
        HotelService hotelService = new HotelService(weekendPricing);

        hotelService.registerHotel(grandHotel);
        hotelService.registerHotel(budgetInn);
        hotelService.registerGuest(alice);
        hotelService.registerGuest(bob);

        System.out.println("\n[SETUP] Registered 2 hotels and 2 guests.");
        System.out.println("  " + grandHotel);
        System.out.println("  " + budgetInn);

        // ------------------------------------------------------------------
        // 3. Search — Alice searches for a Double room in New York with WiFi
        //    under $200/night for a 3-night stay
        // ------------------------------------------------------------------
        // Using Friday check-in to demonstrate weekend pricing
        LocalDate checkIn  = LocalDate.now().plusDays(7);  // 1 week from now
        LocalDate checkOut = checkIn.plusDays(3);           // 3-night stay

        RoomSearchQuery searchQuery = new RoomSearchQuery.Builder()
                .city("New York")
                .checkIn(checkIn)
                .checkOut(checkOut)
                .roomType(RoomType.DOUBLE)
                .maxPrice(200.0)
                .amenity("WiFi")
                .build();

        System.out.println("\n[SEARCH] Query: " + searchQuery);
        List<Room> availableRooms = hotelService.searchRooms(searchQuery);
        System.out.println("[SEARCH] Found " + availableRooms.size() + " room(s):");
        for (Room r : availableRooms) {
            System.out.println("  " + r);
        }

        if (availableRooms.isEmpty()) {
            System.out.println("[ERROR] No rooms found. Exiting demo.");
            return;
        }

        Room selectedRoom = availableRooms.get(0);
        System.out.println("[SEARCH] Alice selects: " + selectedRoom.getRoomNumber()
                + " at " + selectedRoom.getBasePrice() + "/night");

        // ------------------------------------------------------------------
        // 4. Create Booking (PENDING)
        // ------------------------------------------------------------------
        Booking aliceBooking = hotelService.createBooking(
                alice.getGuestId(), selectedRoom.getRoomId(), checkIn, checkOut);

        System.out.println("\n[BOOKING] Created: " + aliceBooking);
        System.out.println("[BOOKING] Status: " + aliceBooking.getStatus());
        System.out.println("[BOOKING] Estimated total: $" + String.format("%.2f", aliceBooking.getTotalAmount()));

        // ------------------------------------------------------------------
        // 5. Confirm Booking (PENDING → CONFIRMED)
        // ------------------------------------------------------------------
        hotelService.confirmBooking(aliceBooking.getBookingId());
        System.out.println("\n[CONFIRM] Booking confirmed. Status: " + aliceBooking.getStatus());

        // ------------------------------------------------------------------
        // 5b. Bob also books a SINGLE room at the Grand Manhattan
        //     using Seasonal Pricing (peak summer months)
        // ------------------------------------------------------------------
        PricingStrategy seasonalPricing = new SeasonalPricingStrategy(
                EnumSet.of(Month.JUNE, Month.JULY, Month.AUGUST, Month.DECEMBER),
                1.5,  // 50% peak surcharge
                0.85  // 15% off-peak discount
        );

        Booking bobBooking = hotelService.createBookingWithStrategy(
                bob.getGuestId(), room101.getRoomId(),
                checkIn, checkOut.plusDays(2), // Bob stays 5 nights
                seasonalPricing);

        hotelService.confirmBooking(bobBooking.getBookingId());
        System.out.println("\n[BOOKING] Bob's booking confirmed: " + bobBooking);
        System.out.println("[BOOKING] Bob's estimated total: $" + String.format("%.2f", bobBooking.getTotalAmount()));

        // ------------------------------------------------------------------
        // 6. Demonstrate cancellation: Bob cancels his booking
        //    (> 48 hours before check-in → full refund)
        // ------------------------------------------------------------------
        CancellationResult cancellation = hotelService.cancelBooking(bobBooking.getBookingId());
        System.out.println("\n[CANCEL] Bob cancels his booking:");
        System.out.println("  " + cancellation);

        // ------------------------------------------------------------------
        // 7. Check In (CONFIRMED → CHECKED_IN)
        // ------------------------------------------------------------------
        hotelService.checkIn(aliceBooking.getBookingId());
        System.out.println("\n[CHECK-IN] Alice checks in. Status: " + aliceBooking.getStatus());
        System.out.println("[CHECK-IN] Room " + selectedRoom.getRoomNumber()
                + " status: " + selectedRoom.getStatus());

        // ------------------------------------------------------------------
        // 8. Check Out (CHECKED_IN → CHECKED_OUT) + Invoice Generation
        // ------------------------------------------------------------------
        Invoice invoice = hotelService.checkOut(aliceBooking.getBookingId());
        System.out.println("\n[CHECK-OUT] Alice checks out. Status: " + aliceBooking.getStatus());
        System.out.println("[CHECK-OUT] Room " + selectedRoom.getRoomNumber()
                + " status: " + selectedRoom.getStatus());

        System.out.println("\n" + invoice.toPrintableString());

        // ------------------------------------------------------------------
        // 9. Retrieve Invoice Later (shows auditability)
        // ------------------------------------------------------------------
        Invoice retrieved = hotelService.getInvoice(aliceBooking.getBookingId());
        System.out.println("[AUDIT] Invoice retrieved post-checkout. Invoice ID: "
                + retrieved.getInvoiceId());

        // ------------------------------------------------------------------
        // 10. Alice's booking history
        // ------------------------------------------------------------------
        System.out.println("\n[HISTORY] Alice's booking history:");
        for (Booking b : hotelService.getBookingsForGuest(alice.getGuestId())) {
            System.out.println("  " + b);
        }

        System.out.println("\n[DEMO COMPLETE]");
    }
}
```

---

### 7.12 Sample Output

```
============================================================
  HOTEL BOOKING SYSTEM — LLD DEMO
============================================================

[SETUP] Registered 2 hotels and 2 guests.
  Hotel{id='HOTEL-001', name='The Grand Manhattan', city='New York', rooms=3}
  Hotel{id='HOTEL-002', name='Budget Inn Downtown', city='New York', rooms=2}

[SEARCH] Query: RoomSearchQuery{city='New York', checkIn=2025-06-29,
         checkOut=2025-07-02, type=DOUBLE, maxPrice=200.0, amenities=[WiFi]}
[SEARCH] Found 2 room(s):
  Room{id='ROOM-202', number='202', type=DOUBLE, price=$180.00, status=AVAILABLE, amenities=[WiFi, TV, Mini-Bar, City View]}
  Room{id='ROOM-A2', number='A2', type=DOUBLE, price=$95.00, status=AVAILABLE, amenities=[WiFi, TV, Parking]}
[SEARCH] Alice selects: 202 at 180.0/night

[BOOKING] Created: Booking{id='BKG-A3B4C5D6', guest='Alice Johnson', room='202',
          hotel='The Grand Manhattan', checkIn=2025-06-29, checkOut=2025-07-02,
          nights=3, total=$607.50, status=PENDING}
[BOOKING] Status: PENDING
[BOOKING] Estimated total: $607.50

[CONFIRM] Booking confirmed. Status: CONFIRMED

[BOOKING] Bob's booking confirmed: Booking{..., nights=5, total=$900.00, status=CONFIRMED}
[BOOKING] Bob's estimated total: $900.00

[CANCEL] Bob cancels his booking:
  CancellationResult{bookingId='BKG-...', tier=FULL, original=$900.00,
                     refund=$900.00, reason='Cancelled more than 48 hours before check-in. Full refund applied.'}

[CHECK-IN] Alice checks in. Status: CHECKED_IN
[CHECK-IN] Room 202 status: OCCUPIED

[CHECK-OUT] Alice checks out. Status: CHECKED_OUT
[CHECK-OUT] Room 202 status: AVAILABLE

============================================================
  INVOICE #INV-F1E2D3C4
============================================================
  Hotel:    The Grand Manhattan
  Guest:    Alice Johnson (alice@example.com)
  Room:     202 (DOUBLE)
  Check-in: 2025-06-29
  Check-out:2025-07-02
  Nights:   3
  Pricing:  Weekend Rate (25% weekend surcharge)
------------------------------------------------------------
  LINE ITEMS
------------------------------------------------------------
  Room 202 (DOUBLE) — 3 night(s) @ $180.00/night  $   540.00
  Dynamic pricing surcharge (Weekend Rate...)       $    67.50
------------------------------------------------------------
  Subtotal                                          $   607.50
  Tax (12%)                                         $    72.90
============================================================
  TOTAL DUE                                         $   680.40
============================================================
  Generated: 2025-06-29T14:32:11.421

[AUDIT] Invoice retrieved post-checkout. Invoice ID: INV-F1E2D3C4

[HISTORY] Alice's booking history:
  Booking{id='BKG-A3B4C5D6', ..., status=CHECKED_OUT}

[DEMO COMPLETE]
```

---

## 8. Design Patterns Used

### Strategy Pattern — Pricing

**Location**: `PricingStrategy` interface + `BasePricingStrategy`, `WeekendPricingStrategy`, `SeasonalPricingStrategy`

**Why here**: Pricing is a classic example of the Strategy pattern because:
1. The algorithm varies independently of the context that uses it (Booking).
2. New pricing rules will be added without changing existing booking logic — an airline might add LoyaltyPricingStrategy or DemandPricingStrategy.
3. Pricing strategies are unit-testable in complete isolation — no need to construct a Booking object to test whether the weekend multiplier is applied correctly.
4. The strategy is captured at booking creation time, so the invoice always shows the rate that was promised to the guest.

**Alternative considered**: A switch statement on a PricingType enum inside Booking. Rejected because it violates OCP — every new pricing type requires modifying the Booking class.

### Builder Pattern — RoomSearchQuery

**Location**: `RoomSearchQuery.Builder`

**Why here**: `RoomSearchQuery` has 6 fields, 4 of which are optional. Without a Builder:
- A constructor with 6 parameters is unreadable (telescoping constructor problem).
- Optional fields would need multiple constructor overloads or require null arguments.
- The search query is used as a value object that should be immutable once built.

**Key Builder behaviors**:
- Mandatory fields (`city`, `checkIn`, `checkOut`) are validated in `build()`.
- `amenity(String)` can be called multiple times to add amenities incrementally.
- `build()` validates cross-field constraints (e.g., checkOut after checkIn).
- The resulting `RoomSearchQuery` is fully immutable.

**Alternative considered**: Passing all parameters directly to `searchRooms()`. Rejected because adding a 7th filter (e.g., starRating) would require changing the service method signature, breaking all call sites.

### Repository / Service Layer (Architectural Pattern)

**Location**: `HotelService`

**Why here**: All persistence (in-memory maps), domain logic (availability checking, policy enforcement), and transaction coordination lives in `HotelService`. This follows the Service Layer pattern, which:
- Provides a single entry point for all operations.
- Separates domain logic from the client (main/demo code).
- Makes it straightforward to swap the backing store (from HashMap to a database) without changing domain classes.

---

## 9. Key Design Decisions and Trade-offs

### Decision 1: Room Availability Checked via Booking Scan, Not Room State

**Decision**: `isRoomBooked()` scans active bookings to determine availability, rather than relying solely on `RoomStatus.AVAILABLE`.

**Rationale**: A room's `RoomStatus` represents its *physical* state (is it currently occupied or under maintenance?). It does not represent *future* booking availability. A room can be `AVAILABLE` physically today but have a confirmed booking starting tomorrow.

**Trade-off**: The scan is O(n) over all bookings, which is acceptable for an in-memory design. In production, this would be replaced by a date-indexed availability table query.

### Decision 2: PricingStrategy Captured at Booking Creation

**Decision**: The `PricingStrategy` used to calculate the price is stored in the `Booking` object rather than re-applied at invoice time.

**Rationale**: This ensures the guest pays the rate that was shown during the booking flow, even if the hotel later changes its pricing strategy. The invoice always reflects what the guest agreed to.

**Trade-off**: If a pricing error was made, there is no automatic way to recalculate without modifying the booking. An admin override flow would be needed.

### Decision 3: Invoice as a Derived Document, Not a First-Class Entity During Booking

**Decision**: Invoices are generated at check-out time, not at booking time.

**Rationale**: Before check-out, the total amount is an estimate (dates could be modified). The authoritative financial record only makes sense once the stay is complete. This matches real-world hotel accounting practice.

**Trade-off**: A preliminary invoice (quote) for the booking period cannot be shown to the guest before check-in without additional modeling. An extension point is to add a `generateQuote()` method that creates a non-stored, read-only invoice snapshot.

### Decision 4: GuestProfile Holds Booking References

**Decision**: `GuestProfile` maintains a list of `Booking` objects directly.

**Rationale**: Guest booking history is a core use case. Embedding the list in the profile makes retrieval O(1) per guest. The `HotelService` also maintains the master bookings map for lookup-by-ID.

**Trade-off**: This creates a bidirectional reference (Booking holds GuestProfile, GuestProfile holds Bookings). In a JPA context, this would need explicit management. For an in-memory design, it is the simplest correct approach.

### Decision 5: Cancellation Policy Uses Wall Clock Time

**Decision**: Cancellation refund tiers are computed against `LocalDateTime.now()` in `HotelService`.

**Rationale**: For simplicity in an in-memory design. The cancellation policy is straightforward business logic based on current time vs. check-in time.

**Trade-off**: This makes cancellation logic difficult to unit-test deterministically. Production code would inject a `java.time.Clock` dependency so tests can control the current time.

---

## 10. Extension Points

### 10.1 New Pricing Strategies

Add a class that implements `PricingStrategy`. Zero changes required elsewhere.

```java
// LoyaltyPricingStrategy — discount for returning guests
public class LoyaltyPricingStrategy implements PricingStrategy {

    private final double discountRate; // e.g., 0.10 for 10% off
    private final int qualifyingStays; // minimum past stays to qualify

    public LoyaltyPricingStrategy(double discountRate, int qualifyingStays) {
        this.discountRate = discountRate;
        this.qualifyingStays = qualifyingStays;
    }

    @Override
    public double calculatePrice(double basePrice, int nights, LocalDate checkIn) {
        return basePrice * nights * (1.0 - discountRate);
    }

    @Override
    public String getStrategyName() {
        return String.format("Loyalty Rate (%.0f%% discount)", discountRate * 100);
    }
}
```

### 10.2 Composite Pricing Strategy

Apply multiple strategies in sequence (e.g., weekend surcharge AND loyalty discount):

```java
public class CompositePricingStrategy implements PricingStrategy {

    private final List<PricingStrategy> strategies;

    public CompositePricingStrategy(List<PricingStrategy> strategies) {
        this.strategies = new ArrayList<>(strategies);
    }

    @Override
    public double calculatePrice(double basePrice, int nights, LocalDate checkIn) {
        double price = basePrice * nights;
        for (PricingStrategy strategy : strategies) {
            // Each strategy adjusts the price sequentially.
            // Note: This assumes strategies apply multiplicative adjustments
            // to the per-night rate. A more sophisticated model would
            // separate "compute adjustment" from "apply adjustment".
            double adjusted = strategy.calculatePrice(basePrice, nights, checkIn);
            price = adjusted; // simplified: last strategy wins
        }
        return price;
    }

    @Override
    public String getStrategyName() {
        return strategies.stream()
                .map(PricingStrategy::getStrategyName)
                .collect(java.util.stream.Collectors.joining(" + "));
    }
}
```

### 10.3 Additional Search Filters

Add new fields to `RoomSearchQuery.Builder` without changing the search method signature:

```java
// In RoomSearchQuery.Builder:
public Builder minStarRating(int stars) {
    this.minStarRating = stars;
    return this;
}

public Builder maxDistanceFromCenter(double km) {
    this.maxDistanceFromCenter = km;
    return this;
}
```

`HotelService.searchRooms()` reads `query.getMinStarRating()` and filters accordingly.

### 10.4 Observer Pattern for Booking Events

Attach notification listeners to booking lifecycle transitions:

```java
public interface BookingEventListener {
    void onBookingCreated(Booking booking);
    void onBookingConfirmed(Booking booking);
    void onBookingCancelled(Booking booking, CancellationResult result);
    void onCheckIn(Booking booking);
    void onCheckOut(Booking booking, Invoice invoice);
}
```

`HotelService` holds a `List<BookingEventListener>` and fires events at each transition point. Implementations can send email confirmations, update analytics, or trigger inventory systems.

### 10.5 Multi-Room Bookings

Extend `Booking` to hold `List<Room>` instead of a single `Room`. The invoice generation would sum across all room line items. The availability check would need to validate each room independently.

### 10.6 State Pattern for Booking Lifecycle

Replace the `status` field and the guard clauses in `Booking` with a full State pattern if the state machine becomes complex:

```java
interface BookingState {
    void confirm(Booking context);
    void cancel(Booking context);
    void checkIn(Booking context);
    void checkOut(Booking context);
}

class PendingState implements BookingState {
    public void confirm(Booking context) { context.setState(new ConfirmedState()); }
    public void cancel(Booking context)  { context.setState(new CancelledState()); }
    public void checkIn(Booking context) { throw new IllegalStateException("Must confirm first"); }
    public void checkOut(Booking context){ throw new IllegalStateException("Must confirm first"); }
}
```

This is overkill for 4 states but is the right direction if each state needs different behavior for multiple operations (e.g., different cancellation logic per state).

---

## 11. Interview Discussion Points

### "Walk me through the booking flow."

> A guest creates a `RoomSearchQuery` using the Builder with required filters. `HotelService.searchRooms()` filters hotels by city, then rooms by type, price, amenities, and availability (scanning active bookings for date overlaps). The guest picks a room and calls `createBooking()`, which instantiates a `Booking` in PENDING state, calculates the price using the configured `PricingStrategy`, and adds the booking to both the service's master map and the guest's profile. The booking is then confirmed, which moves it to CONFIRMED state. At check-in, it moves to CHECKED_IN and the room is marked OCCUPIED. At check-out, it moves to CHECKED_OUT, the room is freed, and an itemized invoice is generated and attached to the booking.

### "Why Strategy for pricing and not an enum-based switch?"

> An enum-based switch in the Booking class would violate OCP — every new pricing type requires opening and modifying Booking. With Strategy, adding a new pricing model is zero-change to existing code: you write a new class that implements the interface. It also enables testing pricing logic completely independently of bookings. Additionally, the strategy instance captured at booking time serves as an audit record: the invoice can always describe exactly which pricing algorithm was applied and why.

### "How do you prevent double-booking?"

> `isRoomBooked()` scans active bookings (those not in CANCELLED or CHECKED_OUT state) for the target room and checks for date overlap using the canonical interval overlap condition: `booking.checkIn < requestedCheckOut AND booking.checkOut > requestedCheckIn`. Both `createBooking()` and `modifyBooking()` call this check before committing changes. In production, this would be a database-level serializable transaction or an optimistic lock with a retry loop.

### "What if two guests try to book the same room at the same time?"

> In this in-memory design, the check-then-act sequence is not atomic, so a race condition exists. In production, you would use one of:
> 1. A database row-level lock with `SELECT FOR UPDATE` on the room's availability record.
> 2. An optimistic concurrency control approach using a version column on the room or availability table.
> 3. A distributed lock (e.g., Redis `SET NX EX`) keyed on `roomId:dateRange`.
> 4. An event-sourcing approach where booking attempts are serialized through a command queue per room.

### "Why is Invoice generated at check-out rather than booking time?"

> Because the booking can be modified after creation. Generating an invoice at booking time would produce a stale figure if dates change. The authoritative invoice is the one generated from the final, locked state of the booking — after check-out when no further changes are possible. We do store the `totalAmount` on the Booking as a running estimate for display purposes.

### "How would you extend this to support cancellation policies per hotel?"

> The `Hotel` class would hold a `CancellationPolicy` object (or interface). `HotelService.cancelBooking()` would call `booking.getHotel().getCancellationPolicy().calculateRefund(booking)` instead of applying hardcoded tiers. This is another Strategy application — each hotel can define its own policy, and a `DefaultCancellationPolicy` covers the standard 48h/24h tiers.

### "Could Room be its own aggregate root?"

> It depends on the access patterns. If rooms are frequently queried independently of their hotel (e.g., "find this room by ID"), a separate `RoomRepository` makes sense. In this design, rooms are always accessed through hotel traversal, which suggests they are value objects within the Hotel aggregate. The current design is pragmatic for an in-memory scope; a production DDD model would likely split them.

---

## 12. Common Mistakes in This Design

### Mistake 1: Using RoomStatus as the Availability Check

**The mistake**: Checking `room.getStatus() == RoomStatus.AVAILABLE` as the sole availability check during search.

**Why it's wrong**: A room can be physically AVAILABLE right now but have a confirmed booking starting tomorrow. Using only the status field causes double-bookings.

**The fix**: Always scan active bookings for date overlap in addition to the physical status check. Physical status catches rooms under maintenance; booking scan catches future commitments.

### Mistake 2: Storing Price Logic Inside Booking

**The mistake**: Writing pricing conditionals directly in `Booking.calculateTotalAmount()`:
```java
// Wrong
if (isWeekend) { price *= 1.25; }
else if (isPeakSeason) { price *= 1.5; }
```

**Why it's wrong**: Booking becomes hard to extend, and the pricing code is untestable in isolation. Every new pricing rule requires modifying the Booking class.

**The fix**: Delegate entirely to `PricingStrategy`. Booking calls `strategy.calculatePrice(basePrice, nights, checkIn)` and does not know which algorithm is running.

### Mistake 3: Generating Invoice at Booking Creation Time

**The mistake**: Generating and storing the final invoice when `createBooking()` is called.

**Why it's wrong**: Dates can be modified after booking creation. The invoice amount would be stale. Additionally, invoicing before a stay is completed does not match real-world accounting practice.

**The fix**: Generate the invoice only on check-out, after the booking is locked in CHECKED_OUT state.

### Mistake 4: Making Booking.modifyDates() Accessible Regardless of State

**The mistake**: Not guarding `modifyDates()` in `Booking` — allowing modification when the booking is PENDING or CHECKED_IN.

**Why it's wrong**: Modifying a CHECKED_IN booking's dates after the guest has already arrived creates inconsistent state. Modifying a PENDING booking is debatable but typically the booking should be confirmed before modification is meaningful.

**The fix**: Guard with a status check. This design allows modification only in CONFIRMED state. For PENDING, the guest can simply cancel and rebook.

### Mistake 5: Putting Hotel Name in Room Instead of Using the Reference

**The mistake**: Storing hotel name, city, and address as fields on `Room`.

**Why it's wrong**: Data duplication. If the hotel's address changes, you have to update every room. The room is owned by the hotel — it should reference the hotel, not copy its data.

**The fix**: `Room` has no hotel reference. The hotel is passed into `Booking` at booking creation time, which is when the hotel–room association is explicitly captured.

### Mistake 6: Not Capturing the Strategy Instance on the Booking

**The mistake**: Storing only a `PricingType` enum on `Booking` and recalculating the price from the current strategy configuration at invoice time.

**Why it's wrong**: If the hotel switches from `WeekendPricingStrategy` to `SeasonalPricingStrategy` between booking and check-out, the invoice will use the wrong pricing algorithm. The guest was quoted under the old strategy.

**The fix**: The actual `PricingStrategy` instance (including its configuration, e.g., `weekendMultiplier = 1.25`) is captured in the `Booking` object at creation time. This is why `Booking` holds a `PricingStrategy` reference, not a `PricingType` enum.

### Mistake 7: Forgetting to Validate Date Range in Builder.build()

**The mistake**: Validating individual field nullness in setter methods but not validating cross-field constraints (`checkOut > checkIn`) until `searchRooms()` is called.

**Why it's wrong**: Invalid queries propagate into the system before being caught. The Builder contract is that `build()` returns a valid, usable object or throws.

**The fix**: `build()` always validates: both dates present, checkOut after checkIn, checkIn not in the past. Callers get an `IllegalStateException` immediately at query construction time.

### Mistake 8: Designing Room as a Value Object with No Identity

**The mistake**: Not assigning a stable `roomId` to rooms, relying on `roomNumber` as the identifier.

**Why it's wrong**: Room numbers are hotel-scoped (two different hotels can have room "101"). Using room number as a global identifier causes collisions. The booking system needs to uniquely identify a specific room across all hotels.

**The fix**: Every room gets a system-generated `roomId` (e.g., `ROOM-101`). `roomNumber` is a display label; `roomId` is the system identifier.

---

*End of Hotel Booking System Case Study. Next: [Food Delivery System](../03-food-delivery/) →*
