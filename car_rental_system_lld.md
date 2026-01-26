# Low-Level Design Interview: Car Rental System

> A complete LLD walkthrough for designing a car rental system like Zoomcar, Hertz, or Uber Rentals.

---

## 🎯 Problem Statement

**Interviewer:** Design a **Car Rental System** where customers can browse available cars, book them for a duration, pick up, return, and pay.

---

## 📋 Requirements

| Feature | In Scope |
|---------|----------|
| Browse available cars | ✅ |
| Book a car (date/time range) | ✅ |
| Pick up car | ✅ |
| Return car | ✅ |
| Cancel booking | ✅ |
| Different car types with pricing | ✅ |
| Late return penalty | ✅ |
| Hourly and daily rentals | ✅ |

| Feature | Out of Scope |
|---------|--------------|
| Multiple locations | ❌ |
| Damage/insurance | ❌ |
| Payment gateway | ❌ |
| Driver option | ❌ |

---

## 🏗️ Core Entities

| Entity | Purpose |
|--------|---------|
| **Car** | Vehicle available for rent |
| **User** | Customer renting a car |
| **Booking** | Reservation linking user, car, dates |
| **Payment** | Payment for a booking |
| **CarRentalService** | Orchestrator |

---

## 💻 Enums

```python
from enum import Enum

class CarType(Enum):
    HATCHBACK = "hatchback"
    SEDAN = "sedan"
    SUV = "suv"
    LUXURY = "luxury"

class CarStatus(Enum):
    AVAILABLE = "available"
    RENTED = "rented"
    MAINTENANCE = "maintenance"

class BookingStatus(Enum):
    CONFIRMED = "confirmed"   # Booked, not picked up
    ACTIVE = "active"         # Car picked up
    COMPLETED = "completed"   # Car returned
    CANCELLED = "cancelled"

class PaymentStatus(Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    REFUNDED = "refunded"
    FAILED = "failed"
```

---

## 💻 Pricing Strategy

```python
from abc import ABC, abstractmethod

class PricingStrategy(ABC):
    @abstractmethod
    def calculate_price(self, car_type: CarType, hours: int) -> float:
        pass

class StandardPricing(PricingStrategy):
    """Standard hourly/daily pricing."""
    
    RATES = {
        CarType.HATCHBACK: {'hourly': 50, 'daily': 500},
        CarType.SEDAN: {'hourly': 70, 'daily': 700},
        CarType.SUV: {'hourly': 100, 'daily': 1000},
        CarType.LUXURY: {'hourly': 200, 'daily': 2000},
    }
    
    def calculate_price(self, car_type: CarType, hours: int) -> float:
        rates = self.RATES[car_type]
        days = hours // 24
        remaining_hours = hours % 24
        
        # Daily rate is cheaper if > 10 hours
        if remaining_hours > 10:
            days += 1
            remaining_hours = 0
        
        return days * rates['daily'] + remaining_hours * rates['hourly']

class WeekendPricing(PricingStrategy):
    """Higher rates on weekends."""
    
    def calculate_price(self, car_type: CarType, hours: int) -> float:
        base = StandardPricing().calculate_price(car_type, hours)
        return base * 1.2  # 20% weekend surcharge
```

---

## 💻 Entity Classes

### Car

```python
import uuid

class Car:
    def __init__(self, name: str, model: str, car_type: CarType, 
                 license_plate: str, km_driven: int = 0):
        self.id = str(uuid.uuid4())
        self.name = name
        self.model = model
        self.car_type = car_type
        self.license_plate = license_plate
        self.km_driven = km_driven
        self.status = CarStatus.AVAILABLE
    
    def is_available(self) -> bool:
        return self.status == CarStatus.AVAILABLE
    
    def __eq__(self, other):
        return isinstance(other, Car) and self.id == other.id
    
    def __hash__(self):
        return hash(self.id)
```

### User

```python
class User:
    def __init__(self, name: str, email: str, phone: str, 
                 license_number: str, age: int):
        self.id = str(uuid.uuid4())
        self.name = name
        self.email = email
        self.phone = phone
        self.license_number = license_number
        self.age = age
    
    def is_eligible_to_rent(self) -> bool:
        return self.age >= 21 and self.license_number is not None
```

### Payment

```python
class Payment:
    def __init__(self, amount: float, user: User):
        self.id = str(uuid.uuid4())
        self.amount = amount
        self.user = user
        self.status = PaymentStatus.PENDING
        self.transaction_id = None
    
    def complete(self, transaction_id: str):
        self.status = PaymentStatus.COMPLETED
        self.transaction_id = transaction_id
    
    def refund(self):
        self.status = PaymentStatus.REFUNDED
```

### Booking

```python
from datetime import datetime

class Booking:
    def __init__(self, user: User, car: Car, 
                 start_time: datetime, end_time: datetime):
        self.id = str(uuid.uuid4())
        self.user = user
        self.car = car
        self.start_time = start_time
        self.end_time = end_time
        self.actual_return_time = None
        self.status = BookingStatus.CONFIRMED
        self.payment = None
        self.created_at = datetime.now()
    
    def get_duration_hours(self) -> int:
        delta = self.end_time - self.start_time
        return int(delta.total_seconds() / 3600)
    
    def is_late_return(self) -> bool:
        if self.actual_return_time:
            return self.actual_return_time > self.end_time
        return False
    
    def get_late_hours(self) -> int:
        if not self.is_late_return():
            return 0
        delta = self.actual_return_time - self.end_time
        return int(delta.total_seconds() / 3600) + 1  # Round up
```

---

## 💻 Car Rental Service

```python
import threading
from typing import List, Optional, Dict
from datetime import datetime

class BookingResult:
    def __init__(self, success: bool, message: str, booking: Booking = None):
        self.success = success
        self.message = message
        self.booking = booking

class CarRentalService:
    """Orchestrator for car rental operations with thread safety."""
    
    def __init__(self, pricing_strategy: PricingStrategy = None):
        self.cars: List[Car] = []
        self.users: Dict[str, User] = {}
        self.bookings: List[Booking] = []
        self.pricing = pricing_strategy or StandardPricing()
        
        # Fine-grained locks per car
        self.car_locks: Dict[str, threading.Lock] = {}
        self.global_lock = threading.Lock()
    
    def _get_car_lock(self, car_id: str) -> threading.Lock:
        with self.global_lock:
            if car_id not in self.car_locks:
                self.car_locks[car_id] = threading.Lock()
            return self.car_locks[car_id]
    
    # ─────────────────────────────────────────
    # Car Management
    # ─────────────────────────────────────────
    
    def add_car(self, car: Car):
        self.cars.append(car)
    
    def add_user(self, user: User):
        self.users[user.id] = user
    
    # ─────────────────────────────────────────
    # Browse Cars
    # ─────────────────────────────────────────
    
    def browse_available_cars(self, start_time: datetime, end_time: datetime, 
                               car_type: CarType = None) -> List[dict]:
        """Return available cars with pricing for given date range."""
        available = []
        hours = int((end_time - start_time).total_seconds() / 3600)
        
        for car in self.cars:
            if car_type and car.car_type != car_type:
                continue
            if self._is_car_available(car, start_time, end_time):
                price = self.pricing.calculate_price(car.car_type, hours)
                available.append({
                    'car': car,
                    'price': price,
                    'price_per_hour': price / hours if hours > 0 else 0
                })
        
        return available
    
    def _is_car_available(self, car: Car, start_time: datetime, 
                          end_time: datetime) -> bool:
        """Check if car is available for given time range."""
        if car.status == CarStatus.MAINTENANCE:
            return False
        
        for booking in self.bookings:
            if booking.car != car:
                continue
            if booking.status not in [BookingStatus.CONFIRMED, BookingStatus.ACTIVE]:
                continue
            
            # Check for time overlap
            if not (booking.end_time <= start_time or booking.start_time >= end_time):
                return False
        
        return True
    
    # ─────────────────────────────────────────
    # Book Car
    # ─────────────────────────────────────────
    
    def book_car(self, user: User, car: Car, 
                 start_time: datetime, end_time: datetime) -> BookingResult:
        """Book a car for given time range with thread safety."""
        
        # Validate user
        if not user.is_eligible_to_rent():
            return BookingResult(False, "User not eligible to rent")
        
        # Lock this specific car
        with self._get_car_lock(car.id):
            # Check availability (inside lock)
            if not self._is_car_available(car, start_time, end_time):
                return BookingResult(False, "Car not available for selected dates")
            
            # Create booking
            booking = Booking(user, car, start_time, end_time)
            
            # Calculate and create payment
            hours = booking.get_duration_hours()
            amount = self.pricing.calculate_price(car.car_type, hours)
            payment = Payment(amount, user)
            
            # Simulate payment processing
            payment.complete(f"TXN_{booking.id[:8]}")
            booking.payment = payment
            
            # Add to bookings
            self.bookings.append(booking)
            
            return BookingResult(True, f"Booking confirmed! Amount: ₹{amount}", booking)
    
    # ─────────────────────────────────────────
    # Pick Up Car
    # ─────────────────────────────────────────
    
    def pickup_car(self, booking_id: str) -> BookingResult:
        """Mark car as picked up."""
        booking = self._get_booking(booking_id)
        
        if not booking:
            return BookingResult(False, "Booking not found")
        
        with self._get_car_lock(booking.car.id):
            if booking.status != BookingStatus.CONFIRMED:
                return BookingResult(False, f"Invalid status: {booking.status}")
            
            if booking.payment.status != PaymentStatus.COMPLETED:
                return BookingResult(False, "Payment not completed")
            
            booking.status = BookingStatus.ACTIVE
            booking.car.status = CarStatus.RENTED
            
            return BookingResult(True, "Car picked up successfully", booking)
    
    # ─────────────────────────────────────────
    # Return Car
    # ─────────────────────────────────────────
    
    def return_car(self, booking_id: str) -> BookingResult:
        """Return a rented car, calculate late fees if any."""
        booking = self._get_booking(booking_id)
        
        if not booking:
            return BookingResult(False, "Booking not found")
        
        with self._get_car_lock(booking.car.id):
            if booking.status != BookingStatus.ACTIVE:
                return BookingResult(False, f"Invalid status: {booking.status}")
            
            booking.actual_return_time = datetime.now()
            booking.status = BookingStatus.COMPLETED
            booking.car.status = CarStatus.AVAILABLE
            
            # Calculate late fee
            late_fee = 0
            if booking.is_late_return():
                late_hours = booking.get_late_hours()
                hourly_rate = self.pricing.RATES[booking.car.car_type]['hourly']
                late_fee = late_hours * hourly_rate * 1.5  # 50% penalty
            
            message = "Car returned successfully"
            if late_fee > 0:
                message += f". Late fee: ₹{late_fee}"
            
            return BookingResult(True, message, booking)
    
    # ─────────────────────────────────────────
    # Cancel Booking
    # ─────────────────────────────────────────
    
    def cancel_booking(self, booking_id: str) -> BookingResult:
        """Cancel a booking and process refund."""
        booking = self._get_booking(booking_id)
        
        if not booking:
            return BookingResult(False, "Booking not found")
        
        with self._get_car_lock(booking.car.id):
            if booking.status == BookingStatus.ACTIVE:
                return BookingResult(False, "Cannot cancel active rental")
            
            if booking.status != BookingStatus.CONFIRMED:
                return BookingResult(False, f"Cannot cancel: {booking.status}")
            
            # Calculate refund based on cancellation policy
            hours_until_start = (booking.start_time - datetime.now()).total_seconds() / 3600
            
            if hours_until_start > 24:
                refund_percent = 100
            elif hours_until_start > 6:
                refund_percent = 50
            else:
                refund_percent = 0
            
            refund_amount = booking.payment.amount * refund_percent / 100
            
            booking.status = BookingStatus.CANCELLED
            if refund_percent > 0:
                booking.payment.refund()
            
            return BookingResult(
                True, 
                f"Booking cancelled. Refund: ₹{refund_amount} ({refund_percent}%)", 
                booking
            )
    
    # ─────────────────────────────────────────
    # Helper
    # ─────────────────────────────────────────
    
    def _get_booking(self, booking_id: str) -> Optional[Booking]:
        for booking in self.bookings:
            if booking.id == booking_id:
                return booking
        return None
```

---

## 🔄 Usage Example

```python
from datetime import datetime, timedelta

# Setup
service = CarRentalService()

# Add cars
car1 = Car("Maruti Swift", "2023", CarType.HATCHBACK, "MH01AB1234")
car2 = Car("Honda City", "2022", CarType.SEDAN, "MH01CD5678")
car3 = Car("Toyota Fortuner", "2023", CarType.SUV, "MH01EF9012")
service.add_car(car1)
service.add_car(car2)
service.add_car(car3)

# Add user
user = User("Vinay", "vinay@email.com", "9999999999", "DL1234567890", 25)
service.add_user(user)

# Browse cars
start = datetime.now() + timedelta(days=1)
end = start + timedelta(days=2)
available = service.browse_available_cars(start, end)
for item in available:
    print(f"{item['car'].name} - ₹{item['price']}")

# Book a car
result = service.book_car(user, car2, start, end)
print(result.message)

# Pickup
result = service.pickup_car(result.booking.id)
print(result.message)

# Return (simulate late return)
result = service.return_car(result.booking.id)
print(result.message)
```

---

## 🏛️ Class Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  CarRentalService                                               │
│       ├── cars: List[Car]                                       │
│       ├── bookings: List[Booking]                               │
│       ├── pricing: PricingStrategy                              │
│       ├── browse_available_cars()                               │
│       ├── book_car()                                            │
│       ├── pickup_car()                                          │
│       ├── return_car()                                          │
│       └── cancel_booking()                                      │
│                                                                 │
│  Car                           User                             │
│       ├── id, name, model           ├── id, name, email         │
│       ├── car_type: CarType         ├── license_number          │
│       └── status: CarStatus         └── age                     │
│                                                                 │
│  Booking                       Payment                          │
│       ├── user, car                 ├── amount                  │
│       ├── start_time, end_time      ├── status                  │
│       ├── status: BookingStatus     └── transaction_id          │
│       └── payment                                               │
│                                                                 │
│  PricingStrategy (Abstract)                                     │
│       ├── StandardPricing                                       │
│       └── WeekendPricing                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔒 Thread Safety

| Operation | Lock Type | Why |
|-----------|-----------|-----|
| **browse_cars** | None | Read-only |
| **book_car** | Per-car lock | Prevent double booking |
| **pickup_car** | Per-car lock | Update status atomically |
| **return_car** | Per-car lock | Update status + fee calculation |
| **cancel_booking** | Per-car lock | Update status + refund |

---

## ✅ Key Takeaways

1. **Status Enums** for Car and Booking — track state transitions
2. **Strategy Pattern** for pricing — easy to add seasonal/promo pricing
3. **Fine-grained locking** per car — better concurrency than global lock
4. **Availability check** uses date overlap logic
5. **Cancellation policy** based on time until booking
6. **Late fee calculation** with penalty multiplier

---

*This article was created as part of LLD interview practice for Amazon SDE-2 (L5) level.*
