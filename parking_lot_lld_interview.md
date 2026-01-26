# Low-Level Design Interview: Parking Lot System

> A complete walkthrough of an LLD interview for Amazon SDE-2 (L5) level, demonstrating requirement gathering, OOP design, design patterns, and production considerations.

---

## 🎯 Problem Statement

**Interviewer:** Let's design a **Parking Lot System** — the kind you see at malls or airports. Vehicles come in, park in an available spot, and pay when they leave.

Before we dive into classes and code, I'd like to understand how you think about this problem. What clarifying questions do you have?

---

## 📋 Requirement Gathering

### Candidate's Questions

**Candidate:** I have a few clarifying questions:
1. What types of vehicles can we take in this problem?
2. Can the parking lot have different floors, or is it single floor?
3. For features, I'm thinking: park vehicle, unpark vehicle, payment, get vehicle position. Can we just calculate payment amount and keep actual payment processing out of scope?

**Interviewer:** Great questions! Let me clarify:

| Question | Answer |
|----------|--------|
| **Vehicle Types** | 3 types: Motorcycle, Car, Bus/Truck |
| **Floors** | Multi-floor, configurable number of floors |
| **Payment Scope** | Yes, just calculate the amount — payment gateway is out of scope |
| **Core Features** | ✅ Park, ✅ Unpark, ✅ Calculate payment, ✅ Get vehicle position |

---

### Candidate's Design Decisions

**Candidate:** For additional considerations:

1. **Different spots for different vehicles** — Small spots for motorcycles, Medium for cars, Large for buses
2. **Spot allocation** — I'll use **Strategy Pattern** here. We can implement "First Available" strategy first and easily swap it later
3. **Pricing** — Also using **Strategy Pattern**. Different vehicles have different hourly rates

**Interviewer:** Excellent! You're proactively identifying patterns. That's exactly what we look for at SDE-2 level.

---

## 📝 Final Requirements Summary

| Feature | Details |
|---------|---------|
| **Vehicles** | Motorcycle, Car, Bus |
| **Spots** | Small (motorcycle), Medium (car), Large (bus) |
| **Floors** | Multiple floors |
| **Spot Allocation** | Strategy Pattern — First Available (default) |
| **Pricing** | Strategy Pattern — Hourly pricing with per-vehicle rates |
| **Payment** | Calculate amount only |

---

## 🏗️ Core Entities Identification

**Candidate:** Based on the requirements, here are my core entities:

| Type | Classes |
|------|---------|
| **Domain Entities** | Vehicle, Spot, Floor, Ticket, ParkingSystem |
| **Enums** | VehicleType, SpotType |
| **Strategies** | SpotAllocationStrategy, PricingStrategy |

**Interviewer:** Good list! Why enums instead of separate classes for vehicle types?

**Candidate:** Since motorcycles, cars, and buses don't have different *behavior* in our system — just different *types* — enums are simpler. If they had different methods, I'd use classes.

**Interviewer:** Perfect reasoning! Know when NOT to overcomplicate.

---

## 💻 Class Designs

### Enums

```python
from enum import Enum

class VehicleType(Enum):
    MOTORCYCLE = "motorcycle"
    CAR = "car"
    BUS = "bus"

class SpotType(Enum):
    SMALL = "small"      # Fits motorcycle
    MEDIUM = "medium"    # Fits car
    LARGE = "large"      # Fits bus
```

---

### Vehicle Class

```python
class Vehicle:
    vehicle_id: str
    vehicle_type: VehicleType
    license_plate: str
```

Simple and clean — no need for inheritance since behavior is identical.

---

### Spot Class

```python
import threading

class Spot:
    spot_id: str
    spot_type: SpotType
    is_available: bool
    vehicle: Vehicle = None
    floor_number: int
    lock: threading.Lock  # For concurrency
    
    def get_availability(self) -> bool:
        return self.is_available
    
    def park_vehicle(self, vehicle: Vehicle) -> bool:
        """
        Parks a vehicle in this spot.
        Returns True if successful, False if spot taken or incompatible.
        Thread-safe implementation.
        """
        with self.lock:
            if not self.is_available:
                return False
            
            if not self._is_compatible(vehicle):
                return False
            
            self.vehicle = vehicle
            self.is_available = False
            return True
    
    def unpark_vehicle(self) -> Vehicle:
        """Removes and returns the parked vehicle."""
        with self.lock:
            vehicle = self.vehicle
            self.vehicle = None
            self.is_available = True
            return vehicle
    
    def _is_compatible(self, vehicle: Vehicle) -> bool:
        """Check if vehicle type matches spot type."""
        compatibility = {
            VehicleType.MOTORCYCLE: SpotType.SMALL,
            VehicleType.CAR: SpotType.MEDIUM,
            VehicleType.BUS: SpotType.LARGE
        }
        return compatibility[vehicle.vehicle_type] == self.spot_type
```

**Interviewer:** I see you've added threading.Lock. Why?

**Candidate:** For concurrency! If two cars try to park at the same spot simultaneously, we need atomic operations. Each spot has its own lock for fine-grained locking — this way, parking at different spots doesn't block each other.

---

### Floor Class

```python
class Floor:
    floor_id: str
    floor_number: int
    spots: List[Spot]
    
    def get_available_spots_by_type(self, spot_type: SpotType) -> List[Spot]:
        return [
            spot for spot in self.spots 
            if spot.is_available and spot.spot_type == spot_type
        ]
```

---

### Ticket Class

```python
from datetime import datetime

class Ticket:
    ticket_id: str
    vehicle: Vehicle
    spot: Spot
    entry_time: datetime
    exit_time: datetime = None
    amount: float = None
    
    def calculate_duration_hours(self) -> int:
        """Returns parking duration in hours (rounded up)."""
        if not self.exit_time:
            self.exit_time = datetime.now()
        
        duration = self.exit_time - self.entry_time
        hours = duration.total_seconds() / 3600
        return max(1, int(hours) + (1 if hours % 1 > 0 else 0))  # Round up
```

---

## 🎨 Design Patterns

### Strategy Pattern 1: Spot Allocation

```python
from abc import ABC, abstractmethod

class SpotAllocationStrategy(ABC):
    @abstractmethod
    def find_spot(self, floors: List[Floor], vehicle: Vehicle) -> Spot:
        """Find an available spot for the vehicle. Returns None if not found."""
        pass


class FirstAvailableStrategy(SpotAllocationStrategy):
    """Returns the first available compatible spot."""
    
    def find_spot(self, floors: List[Floor], vehicle: Vehicle) -> Spot:
        required_type = self._get_required_spot_type(vehicle)
        
        for floor in floors:
            for spot in floor.spots:
                if spot.is_available and spot.spot_type == required_type:
                    return spot
        return None
    
    def _get_required_spot_type(self, vehicle: Vehicle) -> SpotType:
        mapping = {
            VehicleType.MOTORCYCLE: SpotType.SMALL,
            VehicleType.CAR: SpotType.MEDIUM,
            VehicleType.BUS: SpotType.LARGE
        }
        return mapping[vehicle.vehicle_type]


class NearestToEntranceStrategy(SpotAllocationStrategy):
    """Returns the spot nearest to the entrance (lower floor, lower spot number)."""
    
    def find_spot(self, floors: List[Floor], vehicle: Vehicle) -> Spot:
        required_type = self._get_required_spot_type(vehicle)
        
        # Floors are assumed to be ordered, floor 1 is nearest to entrance
        for floor in sorted(floors, key=lambda f: f.floor_number):
            available = [s for s in floor.spots 
                        if s.is_available and s.spot_type == required_type]
            if available:
                return available[0]
        return None
```

**Interviewer:** Why Strategy Pattern here?

**Candidate:** Because the algorithm for finding a spot can vary — first available, nearest to entrance, nearest to exit, load-balanced across floors, etc. With Strategy Pattern, we can swap algorithms without modifying ParkingSystem. It's open for extension, closed for modification.

---

### Strategy Pattern 2: Pricing

```python
class PricingStrategy(ABC):
    @abstractmethod
    def calculate_charges(self, ticket: Ticket) -> float:
        """Calculate parking charges based on vehicle type and duration."""
        pass


class HourlyPricingStrategy(PricingStrategy):
    """Different hourly rates per vehicle type."""
    
    RATES = {
        VehicleType.MOTORCYCLE: 10,  # $10/hour
        VehicleType.CAR: 20,         # $20/hour
        VehicleType.BUS: 50          # $50/hour
    }
    
    def calculate_charges(self, ticket: Ticket) -> float:
        hours = ticket.calculate_duration_hours()
        rate = self.RATES[ticket.vehicle.vehicle_type]
        return hours * rate


class WeekendPricingStrategy(PricingStrategy):
    """50% extra on weekends."""
    
    BASE_RATES = {
        VehicleType.MOTORCYCLE: 10,
        VehicleType.CAR: 20,
        VehicleType.BUS: 50
    }
    
    def calculate_charges(self, ticket: Ticket) -> float:
        hours = ticket.calculate_duration_hours()
        rate = self.BASE_RATES[ticket.vehicle.vehicle_type]
        base_charge = hours * rate
        
        # Check if it's weekend
        if ticket.entry_time.weekday() >= 5:  # Saturday=5, Sunday=6
            return base_charge * 1.5
        return base_charge
```

---

## 🎛️ Orchestrator: ParkingSystem (Singleton)

```python
class ParkingSystem:
    _instance = None
    
    def __new__(cls, *args, **kwargs):
        if not cls._instance:
            cls._instance = super().__new__(cls)
        return cls._instance
    
    def __init__(
        self,
        floors: List[Floor],
        spot_strategy: SpotAllocationStrategy,
        pricing_strategy: PricingStrategy
    ):
        self.floors = floors
        self.spot_strategy = spot_strategy
        self.pricing_strategy = pricing_strategy
        self.active_tickets: Dict[str, Ticket] = {}  # ticket_id -> Ticket
    
    def park(self, vehicle: Vehicle) -> Ticket:
        """
        Park a vehicle and return a ticket.
        Returns None if no spot available.
        """
        # Find spot using strategy
        spot = self.spot_strategy.find_spot(self.floors, vehicle)
        
        if not spot:
            return None  # Parking full
        
        # Attempt to park (thread-safe)
        if not spot.park_vehicle(vehicle):
            # Race condition: spot was taken between find and park
            # Could retry with a loop, but keeping simple for now
            return None
        
        # Create ticket
        ticket = Ticket(
            ticket_id=self._generate_ticket_id(),
            vehicle=vehicle,
            spot=spot,
            entry_time=datetime.now()
        )
        
        self.active_tickets[ticket.ticket_id] = ticket
        return ticket
    
    def unpark(self, ticket_id: str) -> float:
        """
        Unpark a vehicle and return the charges.
        Returns -1 if ticket not found.
        """
        if ticket_id not in self.active_tickets:
            return -1
        
        ticket = self.active_tickets[ticket_id]
        
        # Calculate charges
        ticket.exit_time = datetime.now()
        ticket.amount = self.pricing_strategy.calculate_charges(ticket)
        
        # Free the spot
        ticket.spot.unpark_vehicle()
        
        # Remove from active tickets
        del self.active_tickets[ticket_id]
        
        return ticket.amount
    
    def get_vehicle_position(self, ticket_id: str) -> str:
        """Returns the location of a parked vehicle."""
        if ticket_id not in self.active_tickets:
            return "Vehicle not found"
        
        ticket = self.active_tickets[ticket_id]
        spot = ticket.spot
        return f"Floor {spot.floor_number}, Spot {spot.spot_id}"
    
    def _generate_ticket_id(self) -> str:
        import uuid
        return str(uuid.uuid4())[:8].upper()
```

---

## 🔄 Flow Walkthrough

### Entry Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        CAR ENTERS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Vehicle created (type=CAR, license="ABC123")                │
│                    │                                            │
│                    ▼                                            │
│  2. ParkingSystem.park(vehicle)                                 │
│                    │                                            │
│                    ▼                                            │
│  3. SpotAllocationStrategy.find_spot(floors, vehicle)           │
│                    │                                            │
│         ┌─────────┴─────────┐                                   │
│         ▼                   ▼                                   │
│     Spot found          No spot                                 │
│         │                   │                                   │
│         ▼                   ▼                                   │
│  4. spot.park_vehicle()   return None                           │
│         │                                                       │
│         ▼                                                       │
│  5. Create Ticket (entry_time = now)                            │
│         │                                                       │
│         ▼                                                       │
│  6. Store in active_tickets dict                                │
│         │                                                       │
│         ▼                                                       │
│  7. Return Ticket                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Exit Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        CAR EXITS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ParkingSystem.unpark(ticket_id)                             │
│                    │                                            │
│                    ▼                                            │
│  2. Find ticket from active_tickets (O(1))                      │
│                    │                                            │
│                    ▼                                            │
│  3. PricingStrategy.calculate_charges(ticket)                   │
│                    │                                            │
│                    ▼                                            │
│  4. ticket.exit_time = now                                      │
│     ticket.amount = calculated_charges                          │
│                    │                                            │
│                    ▼                                            │
│  5. spot.unpark_vehicle()                                       │
│                    │                                            │
│                    ▼                                            │
│  6. Remove from active_tickets                                  │
│                    │                                            │
│                    ▼                                            │
│  7. Return charges                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔒 Concurrency Handling

**Interviewer:** What happens if two cars try to park at the same spot?

**Candidate:** Each spot has its own lock. The `park_vehicle` method is thread-safe:

```python
def park_vehicle(self, vehicle: Vehicle) -> bool:
    with self.lock:  # Atomic block
        if not self.is_available:
            return False
        self.vehicle = vehicle
        self.is_available = False
        return True
```

**Interviewer:** If two cars park at different spots, do they block each other?

**Candidate:** No! Because each spot has its own lock instance. `spot1.lock` and `spot2.lock` are different locks. This is **fine-grained locking** — maximum parallelism.

| Thread 1 | Thread 2 | Blocked? |
|----------|----------|----------|
| `spot1.park(car1)` | `spot2.park(car2)` | ❌ No |
| `spot1.park(car1)` | `spot1.park(car2)` | ✅ Yes |

---

## 🏛️ Class Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  ParkingSystem (Singleton)                                          │
│       │                                                             │
│       ├── floors: List[Floor]                                       │
│       │       └── Floor                                             │
│       │            └── spots: List[Spot]                            │
│       │                    └── Spot ◄─── Vehicle                    │
│       │                                                             │
│       ├── spot_strategy: SpotAllocationStrategy                     │
│       │       ├── FirstAvailableStrategy                            │
│       │       └── NearestToEntranceStrategy                         │
│       │                                                             │
│       ├── pricing_strategy: PricingStrategy                         │
│       │       ├── HourlyPricingStrategy                             │
│       │       └── WeekendPricingStrategy                            │
│       │                                                             │
│       └── active_tickets: Dict[str, Ticket]                         │
│                               └── Ticket ◄─── Vehicle + Spot        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🎯 Design Patterns Summary

| Pattern | Where Used | Why |
|---------|------------|-----|
| **Strategy** | SpotAllocationStrategy | Multiple algorithms for finding spots, swappable |
| **Strategy** | PricingStrategy | Different pricing models, swappable |
| **Singleton** | ParkingSystem | One orchestrator for entire system |

### Why NOT Factory Pattern?

**Candidate:** Since we used enums for VehicleType and SpotType instead of separate classes, Factory pattern would be overkill. Factory shines when you need to create different concrete classes dynamically.

**Interviewer:** Excellent judgment! Knowing when NOT to use a pattern is just as important.

---

## 🧪 Edge Cases to Consider

| Edge Case | How to Handle |
|-----------|---------------|
| Parking lot full | Return None/error from park() |
| Invalid ticket ID | Return -1/error from unpark() |
| Same vehicle parks twice | Validate before parking |
| Vehicle type doesn't match spot | Compatibility check in Spot |
| Concurrent parking at same spot | Fine-grained locking |

---

## 📊 Time & Space Complexity

| Operation | Time | Space |
|-----------|------|-------|
| park() | O(F × S) worst case | O(1) |
| unpark() | O(1) | O(1) |
| get_vehicle_position() | O(1) | O(1) |

Where F = number of floors, S = spots per floor

---

## 🚀 Extensibility

This design easily accommodates:

1. **New vehicle types** — Add to enum, add spot compatibility
2. **New allocation strategies** — Implement SpotAllocationStrategy
3. **New pricing models** — Implement PricingStrategy
4. **New features** — Reservations, EV charging spots, etc.

---

## ✅ Interview Takeaways

1. **Clarify requirements** before designing
2. **Use enums** when types differ only in data, not behavior
3. **Apply patterns purposefully** — Strategy for algorithms, Singleton for orchestrators
4. **Consider concurrency** — Fine-grained locking for parallelism
5. **Design for extensibility** — Open-Closed Principle
6. **Know when NOT to use patterns** — Avoid overengineering

---

*This article was created as part of LLD interview practice for Amazon SDE-2 (L5) level.*
