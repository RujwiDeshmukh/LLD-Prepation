# Low-Level Design Interview: Movie Ticket Booking System

> The **MOST frequently asked** LLD interview problem — covering concurrent seat booking, Redis locking, and payment handling.

---

## 🎯 Problem Statement

**Interviewer:** Design a **Movie Ticket Booking System** like BookMyShow or PVR where users can browse movies, select seats, and book tickets.

---

## 📋 Requirements

### Functional Requirements

| Feature | Description |
|---------|-------------|
| Browse movies by city | Show currently playing movies |
| View showtimes | Movies at different theaters and times |
| Seat selection | Visual seat map with availability |
| **Concurrent booking** | Handle multiple users booking same seat |
| Payment window | 10 minutes to complete payment |
| Booking history | My Bookings section |

### Non-Functional Requirements

| Requirement | Description |
|-------------|-------------|
| **High consistency** | No double booking allowed |
| Low latency | Seat selection should be fast |
| Handle spikes | New movie release traffic |

---

## 🔥 Famous Interview Questions

These questions are asked in **90%** of movie booking interviews:

| Question | Why It's Asked |
|----------|----------------|
| How to prevent double booking? | Tests concurrency understanding |
| What if two users select same seat simultaneously? | Tests atomicity |
| How to auto-release seats after timeout? | Tests TTL/scheduling |
| What if payment succeeds but confirm fails? | Tests failure handling |

---

## 🔄 User Flow

```
User → Select City → Browse Movies → Select Movie
    → Select Theater + Showtime → Select Seats
    → Block Seats (10 min timer starts) → Make Payment
    → Confirm Booking → Seats BOOKED
    
If payment fails/times out:
    → Seats auto-released → Available for others
```

---

## 📡 API Contracts

### 1. Get Movies by City

```
GET /api/movies?city=Pune
```

**Response:**
```json
{
    "movies": [
        {
            "id": "MOV_001",
            "name": "Inception",
            "language": "English",
            "formats": ["2D", "IMAX"],
            "duration_mins": 148,
            "genre": "Sci-Fi",
            "rating": 8.8
        }
    ]
}
```

### 2. Get Shows for Movie

```
GET /api/shows?movie_id=MOV_001&city=Pune&date=2024-01-26
```

**Response:**
```json
{
    "shows": [
        {
            "show_id": "SH_001",
            "cinema": {
                "id": "CIN_01",
                "name": "PVR Phoenix Mall"
            },
            "screen": "Audi 1",
            "time": "10:30 AM",
            "format": "IMAX",
            "available_seats": 120,
            "price_range": {"min": 150, "max": 500}
        }
    ]
}
```

### 3. Get Seat Layout

```
GET /api/shows/{show_id}/seats
```

**Response:**
```json
{
    "show_id": "SH_001",
    "screen": "Audi 1",
    "layout": [
        {
            "row": "A",
            "seat_type": "RECLINER",
            "price": 500,
            "seats": [
                {"id": "A1", "status": "AVAILABLE"},
                {"id": "A2", "status": "BOOKED"},
                {"id": "A3", "status": "BLOCKED"}
            ]
        },
        {
            "row": "B",
            "seat_type": "PREMIUM",
            "price": 350,
            "seats": [...]
        }
    ]
}
```

### 4. Block Seats

```
POST /api/bookings/block
```

**Request:**
```json
{
    "show_id": "SH_001",
    "seat_ids": ["A1", "A3", "A4"],
    "user_id": "USER_123"
}
```

**Response:**
```json
{
    "booking_id": "BK_001",
    "status": "BLOCKED",
    "seats": ["A1", "A3", "A4"],
    "total_amount": 1500,
    "expires_at": "2024-01-26T12:10:00Z",
    "time_remaining_seconds": 600
}
```

### 5. Confirm Booking

```
POST /api/bookings/confirm
```

**Request:**
```json
{
    "booking_id": "BK_001",
    "payment_id": "PAY_789"
}
```

**Response:**
```json
{
    "booking_id": "BK_001",
    "status": "CONFIRMED",
    "ticket_id": "TKT_001",
    "seats": ["A1", "A3", "A4"],
    "show_details": {...},
    "qr_code": "..."
}
```

---

## 🏗️ Core Entities

### Enums

```python
from enum import Enum

class SeatType(Enum):
    REGULAR = "regular"
    PREMIUM = "premium"
    RECLINER = "recliner"

class SeatStatus(Enum):
    AVAILABLE = "available"
    BLOCKED = "blocked"
    BOOKED = "booked"

class BookingStatus(Enum):
    BLOCKED = "blocked"       # Seats held, awaiting payment
    CONFIRMED = "confirmed"   # Payment done
    CANCELLED = "cancelled"
    EXPIRED = "expired"       # Payment timeout

class MovieFormat(Enum):
    TWO_D = "2D"
    THREE_D = "3D"
    IMAX = "IMAX"
```

### Entity Classes

```python
class City:
    id: str
    name: str
    cinemas: List[Cinema]

class Movie:
    id: str
    name: str
    language: str
    duration_mins: int
    genres: List[str]
    formats: List[MovieFormat]

class Cinema:
    id: str
    name: str
    city_id: str
    address: str
    screens: List[Screen]

class Screen:
    id: str
    cinema_id: str
    name: str
    total_seats: int
    seat_layout: List[Seat]

class Show:
    """Ties Movie + Cinema + Screen + Time together."""
    id: str
    movie_id: str
    cinema_id: str
    screen_id: str
    show_date: date
    show_time: time
    format: MovieFormat

class Seat:
    id: str
    screen_id: str
    row: str
    column: int
    seat_type: SeatType
    price: float

class ShowSeat:
    """Status of a seat for a specific show."""
    show_id: str
    seat_id: str
    status: SeatStatus
    blocked_by: str  # user_id if blocked
    blocked_at: datetime

class Booking:
    id: str
    user_id: str
    show_id: str
    seat_ids: List[str]
    status: BookingStatus
    total_amount: float
    blocked_at: datetime
    confirmed_at: datetime
    payment_id: str
```

---

## 🔒 The Critical Problem: Concurrent Seat Booking

### ❌ Bad Approach 1: No Locking

```python
def block_seat(seat_id, user_id):
    seat = db.get_seat(seat_id)
    if seat.status == AVAILABLE:  # Check
        seat.status = BLOCKED      # Update
        seat.blocked_by = user_id
        db.save(seat)
        return Success
    return Failed
```

**Problem:** Race condition! Two users can both see AVAILABLE and both block.

```
User A: Check → AVAILABLE ✓
User B: Check → AVAILABLE ✓  ← Same moment!
User A: Update → BLOCKED
User B: Update → BLOCKED     ← Overwrites User A!
```

---

### ❌ Bad Approach 2: Database Lock (Pessimistic)

```python
def block_seat(seat_id, user_id):
    db.execute("SELECT * FROM seats WHERE id = ? FOR UPDATE", seat_id)
    # ... proceed
```

**Problem:** 
- Database-level locking doesn't scale
- Holds connections during entire booking flow
- Deadlocks with multiple seats

---

### ⚠️ Okay Approach: Database Optimistic Locking

```python
def block_seat(seat_id, user_id):
    result = db.execute("""
        UPDATE show_seats 
        SET status = 'BLOCKED', blocked_by = ?, blocked_at = NOW()
        WHERE seat_id = ? AND show_id = ? AND status = 'AVAILABLE'
    """, user_id, seat_id, show_id)
    
    if result.rows_affected == 1:
        return Success
    return Failed("Seat not available")
```

**Better!** But still hits database for every blocking request.

---

### ✅ Good Approach: Redis SETNX

```python
import redis

def block_seats(show_id: str, seat_ids: List[str], user_id: str) -> bool:
    redis_client = redis.Redis()
    
    blocked_seats = []
    try:
        for seat_id in seat_ids:
            key = f"seat:{show_id}:{seat_id}"
            
            # SETNX is atomic - only one user succeeds
            success = redis_client.set(
                key, 
                user_id,
                nx=True,   # Only set if NOT exists
                ex=600     # Expire in 10 minutes (auto-release!)
            )
            
            if not success:
                # Seat already taken, rollback
                raise SeatUnavailableError(seat_id)
            
            blocked_seats.append(seat_id)
        
        return True
        
    except SeatUnavailableError as e:
        # Rollback: release seats we blocked
        for seat_id in blocked_seats:
            redis_client.delete(f"seat:{show_id}:{seat_id}")
        raise e
```

**Why this works:**
- `SETNX` is atomic — only one user wins
- TTL auto-releases seats after 10 min
- No database load during blocking
- Scales horizontally

---

### ✅ Best Approach: Redis + Distributed Lock (Multiple Seats)

```python
def block_seats_atomic(show_id: str, seat_ids: List[str], user_id: str):
    """Block multiple seats atomically using Lua script."""
    
    lua_script = """
    -- Check all seats first
    for i, seat_id in ipairs(KEYS) do
        if redis.call('EXISTS', seat_id) == 1 then
            return {0, seat_id}  -- Seat already taken
        end
    end
    
    -- Block all seats
    for i, seat_id in ipairs(KEYS) do
        redis.call('SETEX', seat_id, ARGV[1], ARGV[2])
    end
    
    return {1, 'success'}
    """
    
    keys = [f"seat:{show_id}:{seat_id}" for seat_id in seat_ids]
    result = redis_client.eval(lua_script, len(keys), *keys, 600, user_id)
    
    if result[0] == 1:
        return Success("All seats blocked")
    else:
        return Failed(f"Seat {result[1]} not available")
```

**Why Lua script?**
- Entire script runs atomically on Redis server
- No partial blocking possible
- All-or-nothing semantics

---

## 🔄 Seat Status State Machine

```
┌───────────┐     block()     ┌───────────┐    confirm()   ┌──────────┐
│ AVAILABLE │ ───────────────► │  BLOCKED  │ ──────────────► │  BOOKED  │
└───────────┘                  └───────────┘                └──────────┘
      ▲                              │
      │                              │ TTL expires (10 min)
      │                              │ or cancel()
      └──────────────────────────────┘
              auto-release
```

---

## ⏰ Payment Timeout Handling

### Automatic Release with Redis TTL

```python
# When blocking, set 10-minute TTL
redis.setex(f"seat:{show_id}:{seat_id}", 600, user_id)

# After 10 minutes, key auto-deletes
# No cron job needed for release!
```

### Sync with Database

```python
def confirm_booking(booking_id: str, payment_id: str):
    booking = db.get_booking(booking_id)
    
    # Verify seats still blocked for this user
    for seat_id in booking.seat_ids:
        key = f"seat:{booking.show_id}:{seat_id}"
        blocked_by = redis.get(key)
        
        if blocked_by != booking.user_id:
            # Seats expired or taken by someone else
            refund(payment_id)
            booking.status = EXPIRED
            return Failed("Seats expired. Refund initiated.")
    
    # All good - confirm booking
    for seat_id in booking.seat_ids:
        # Update database
        db.execute("""
            UPDATE show_seats SET status = 'BOOKED' 
            WHERE show_id = ? AND seat_id = ?
        """, booking.show_id, seat_id)
        
        # Delete from Redis (no longer needed)
        redis.delete(f"seat:{booking.show_id}:{seat_id}")
    
    booking.status = CONFIRMED
    booking.confirmed_at = datetime.now()
    booking.payment_id = payment_id
    
    return Success("Booking confirmed!")
```

---

## 💥 Edge Case: Payment Success, Confirm Fails

**Scenario:**
1. User pays ₹500 ✓
2. Network fails during confirm API call
3. Seats expire after 10 min
4. User lost money

**Solution: Reconciliation Job**

```python
# Cron job running every 5 minutes
def reconcile_orphaned_payments():
    orphaned = db.query("""
        SELECT b.*, p.status as payment_status
        FROM bookings b
        JOIN payments p ON b.id = p.booking_id
        WHERE b.status = 'BLOCKED'
        AND b.blocked_at < NOW() - INTERVAL 10 MINUTE
        AND p.status = 'SUCCESS'
    """)
    
    for booking in orphaned:
        # Check if seats still available
        can_confirm = check_seats_available(booking)
        
        if can_confirm:
            # Confirm the booking
            confirm_booking_force(booking)
        else:
            # Seats taken, refund
            initiate_refund(booking.payment_id)
            booking.status = EXPIRED
```

---

## 🗄️ Database Schema

```sql
CREATE TABLE cities (
    id VARCHAR(36) PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE cinemas (
    id VARCHAR(36) PRIMARY KEY,
    city_id VARCHAR(36) REFERENCES cities(id),
    name VARCHAR(200) NOT NULL,
    address TEXT
);

CREATE TABLE screens (
    id VARCHAR(36) PRIMARY KEY,
    cinema_id VARCHAR(36) REFERENCES cinemas(id),
    name VARCHAR(50),
    total_seats INT
);

CREATE TABLE seats (
    id VARCHAR(10) PRIMARY KEY,
    screen_id VARCHAR(36) REFERENCES screens(id),
    row CHAR(1),
    col INT,
    seat_type ENUM('REGULAR', 'PREMIUM', 'RECLINER'),
    base_price DECIMAL(10,2)
);

CREATE TABLE movies (
    id VARCHAR(36) PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    language VARCHAR(50),
    duration_mins INT,
    release_date DATE
);

CREATE TABLE shows (
    id VARCHAR(36) PRIMARY KEY,
    movie_id VARCHAR(36) REFERENCES movies(id),
    screen_id VARCHAR(36) REFERENCES screens(id),
    show_date DATE,
    show_time TIME,
    format ENUM('2D', '3D', 'IMAX'),
    INDEX idx_movie_date (movie_id, show_date)
);

CREATE TABLE show_seats (
    show_id VARCHAR(36) REFERENCES shows(id),
    seat_id VARCHAR(10) REFERENCES seats(id),
    status ENUM('AVAILABLE', 'BLOCKED', 'BOOKED') DEFAULT 'AVAILABLE',
    blocked_by VARCHAR(36),
    blocked_at TIMESTAMP,
    PRIMARY KEY (show_id, seat_id),
    INDEX idx_status (show_id, status)
);

CREATE TABLE bookings (
    id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(36),
    show_id VARCHAR(36) REFERENCES shows(id),
    status ENUM('BLOCKED', 'CONFIRMED', 'CANCELLED', 'EXPIRED'),
    total_amount DECIMAL(10,2),
    blocked_at TIMESTAMP,
    confirmed_at TIMESTAMP,
    payment_id VARCHAR(36)
);

CREATE TABLE booking_seats (
    booking_id VARCHAR(36) REFERENCES bookings(id),
    seat_id VARCHAR(10),
    price DECIMAL(10,2),
    PRIMARY KEY (booking_id, seat_id)
);
```

---

## 🏛️ Class Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  BookingService (Singleton)                                     │
│       ├── block_seats() ── uses Redis SETNX                     │
│       ├── confirm_booking() ── sync DB + delete Redis           │
│       ├── cancel_booking()                                      │
│       └── get_seat_layout()                                     │
│                                                                 │
│  Movie ─────► Show ◄───── Screen ◄───── Cinema ◄───── City     │
│                │                            │                   │
│                ▼                            ▼                   │
│           ShowSeat ◄───────────────────── Seat                  │
│           (status)                     (layout)                 │
│                                                                 │
│  Booking                                                        │
│       ├── user_id, show_id                                      │
│       ├── seats: List[Seat]                                     │
│       ├── status (BLOCKED → CONFIRMED)                          │
│       └── payment_id                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## ✅ Interview Summary

| Question | Answer |
|----------|--------|
| How to prevent double booking? | Redis SETNX (atomic set-if-not-exists) |
| How to release seats after timeout? | Redis TTL (auto-expire in 10 min) |
| Multiple seats atomically? | Lua script or distributed lock |
| Payment success but confirm fails? | Reconciliation cron job |
| Database vs Redis? | Redis for blocking, DB for confirmed state |

---

## 📊 Comparison Table

| Approach | Pros | Cons |
|----------|------|------|
| No locking | Simple | Race conditions |
| DB pessimistic lock | Strong consistency | Doesn't scale, deadlocks |
| DB optimistic lock | Better than pessimistic | Still hits DB |
| **Redis SETNX** | Fast, atomic, TTL | Need Redis |
| **Redis + Lua** | Atomic multi-seat | Complex script |

---

## 🎯 Patterns Used

| Pattern | Where |
|---------|-------|
| **Singleton** | BookingService |
| **State** | Seat status transitions |
| **Strategy** | Pricing per seat type |

---

*This article was created as part of LLD interview practice for Amazon SDE-2 (L5) level.*
