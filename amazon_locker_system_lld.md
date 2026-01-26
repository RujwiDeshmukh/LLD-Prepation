# Low-Level Design Interview: Amazon Locker System

> Amazon-specific LLD interview question demonstrating **Strategy** and **State** patterns.

---

## 🎯 Problem Statement

**Interviewer:** Design an **Amazon Locker System** — self-service kiosks where customers pick up or return packages.

---

## 📋 Requirements

### Functional

| Feature | Description |
|---------|-------------|
| Assign locker to package | Based on package size |
| Generate pickup code | 6-digit code, sent to customer |
| Customer pickup | Enter code → locker opens |
| Code expiration | 3 days to pickup |
| Return packages | Customer can return via locker |

### Constraints

| Constraint | Value |
|------------|-------|
| Locker sizes | S, M, L, XL |
| Code validity | 3 days |
| 1 package per locker | Yes |

---

## 🔄 User Flow

### Delivery Flow
```
Amazon ships package → Delivery person arrives at locker location
    → System assigns best-fit locker → Delivery person places package
    → System generates code → Customer receives code via email/SMS
```

### Pickup Flow
```
Customer at locker → Enters 6-digit code → System validates
    → Locker opens → Customer takes package → Locker becomes available
```

---

## 📡 API Contracts

### 1. Assign Locker

```
POST /api/locker/assign
```

**Request:**
```json
{
    "package_id": "PKG_001",
    "package_size": "MEDIUM",
    "location_id": "LOC_MALL_01"
}
```

**Response:**
```json
{
    "status": "SUCCESS",
    "locker_id": "L_15",
    "position": {"row": 2, "col": 5},
    "message": "Place package in Locker 15 (Row 2, Col 5)"
}
```

### 2. Pickup Package

```
POST /api/locker/pickup
```

**Request:**
```json
{
    "code": "847291",
    "location_id": "LOC_MALL_01"
}
```

**Response:**
```json
{
    "status": "SUCCESS",
    "locker_id": "L_15",
    "message": "Locker 15 is open. Please collect your package."
}
```

### 3. Initiate Return

```
POST /api/locker/return
```

**Request:**
```json
{
    "order_id": "ORD_001",
    "location_id": "LOC_MALL_01"
}
```

**Response:**
```json
{
    "code": "293847",
    "locker_id": "L_08",
    "expires_at": "2024-01-26T18:00:00Z",
    "message": "Place return item in Locker 8"
}
```

---

## 🏗️ Core Entities

### Enums

```python
from enum import Enum

class LockerSize(Enum):
    SMALL = 1
    MEDIUM = 2
    LARGE = 3
    XL = 4

class LockerStatus(Enum):
    AVAILABLE = "available"
    RESERVED = "reserved"      # Assigned, package not yet delivered
    OCCUPIED = "occupied"      # Package inside
    OUT_OF_SERVICE = "out_of_service"
```

### Entity Classes

```python
class LockerLocation:
    """Physical kiosk (e.g., at mall)."""
    id: str
    name: str
    address: str
    lockers: List[Locker]

class Locker:
    """Individual compartment."""
    id: str
    location_id: str
    size: LockerSize
    row: int
    col: int
    status: LockerStatus
    package: Package
    access_code: str
    code_expires_at: datetime

class Package:
    id: str
    order_id: str
    user_id: str
    size: LockerSize
    delivery_date: datetime

class AccessCode:
    code: str
    locker_id: str
    package_id: str
    user_id: str
    generated_at: datetime
    expires_at: datetime
    is_used: bool
```

---

## 🎨 Strategy Pattern: Locker Assignment

```python
from abc import ABC, abstractmethod

class AssignmentStrategy(ABC):
    @abstractmethod
    def assign(self, package_size: LockerSize, 
               available_lockers: List[Locker]) -> Optional[Locker]:
        pass


class ExactFitStrategy(AssignmentStrategy):
    """Assign locker of exact same size."""
    
    def assign(self, package_size, available_lockers):
        for locker in available_lockers:
            if locker.size == package_size:
                return locker
        return None


class BestFitStrategy(AssignmentStrategy):
    """Assign smallest locker that fits package."""
    
    def assign(self, package_size, available_lockers):
        # Sort by size ascending
        candidates = [l for l in available_lockers 
                      if l.size.value >= package_size.value]
        candidates.sort(key=lambda l: l.size.value)
        
        return candidates[0] if candidates else None


class FirstAvailableStrategy(AssignmentStrategy):
    """Assign first available locker that fits."""
    
    def assign(self, package_size, available_lockers):
        for locker in available_lockers:
            if locker.size.value >= package_size.value:
                return locker
        return None
```

---

## 🎨 State Pattern: Locker Status

```python
class LockerState(ABC):
    @abstractmethod
    def assign(self, locker: 'Locker', package: Package) -> bool:
        pass
    
    @abstractmethod
    def deliver(self, locker: 'Locker') -> bool:
        pass
    
    @abstractmethod
    def pickup(self, locker: 'Locker', code: str) -> bool:
        pass
    
    @abstractmethod
    def expire(self, locker: 'Locker'):
        pass


class AvailableState(LockerState):
    def assign(self, locker, package):
        locker.package = package
        locker.set_state(ReservedState())
        return True
    
    def deliver(self, locker):
        raise InvalidOperationError("No package assigned")
    
    def pickup(self, locker, code):
        raise InvalidOperationError("Locker is empty")
    
    def expire(self, locker):
        pass  # Nothing to expire


class ReservedState(LockerState):
    """Package assigned, awaiting delivery."""
    
    def assign(self, locker, package):
        raise InvalidOperationError("Already reserved")
    
    def deliver(self, locker):
        locker.access_code = generate_code()
        locker.code_expires_at = datetime.now() + timedelta(days=3)
        locker.set_state(OccupiedState())
        notify_customer(locker.package.user_id, locker.access_code)
        return True
    
    def pickup(self, locker, code):
        raise InvalidOperationError("Package not delivered yet")
    
    def expire(self, locker):
        locker.package = None
        locker.set_state(AvailableState())


class OccupiedState(LockerState):
    """Package inside, waiting for pickup."""
    
    def assign(self, locker, package):
        raise InvalidOperationError("Locker occupied")
    
    def deliver(self, locker):
        raise InvalidOperationError("Already delivered")
    
    def pickup(self, locker, code):
        if locker.access_code != code:
            raise InvalidCodeError("Wrong code")
        if datetime.now() > locker.code_expires_at:
            raise ExpiredCodeError("Code expired")
        
        locker.open_door()
        locker.package = None
        locker.access_code = None
        locker.set_state(AvailableState())
        return True
    
    def expire(self, locker):
        locker.package.status = RETURN_TO_WAREHOUSE
        locker.package = None
        locker.access_code = None
        locker.set_state(AvailableState())


class Locker:
    def __init__(self, id: str, size: LockerSize, row: int, col: int):
        self.id = id
        self.size = size
        self.row = row
        self.col = col
        self.package = None
        self.access_code = None
        self.state = AvailableState()
    
    def set_state(self, state: LockerState):
        self.state = state
    
    def assign(self, package):
        return self.state.assign(self, package)
    
    def deliver(self):
        return self.state.deliver(self)
    
    def pickup(self, code):
        return self.state.pickup(self, code)
```

### State Transitions

```
┌───────────┐   assign()   ┌───────────┐  deliver()   ┌──────────┐
│ AVAILABLE │ ────────────► │ RESERVED  │ ───────────► │ OCCUPIED │
└───────────┘              └───────────┘              └──────────┘
      ▲                          │                          │
      │        expire()          │                          │ pickup()
      │◄─────────────────────────┘                          │
      │                                                     │
      │◄────────────────────────────────────────────────────┘
                              expire() (3 days)
```

---

## 💻 Locker System Service

```python
import threading
from datetime import datetime, timedelta

class LockerSystem:
    _instance = None
    _lock = threading.Lock()
    
    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance
    
    def __init__(self):
        self.locations: Dict[str, LockerLocation] = {}
        self.code_mapping: Dict[str, Locker] = {}
        self.assignment_strategy = BestFitStrategy()
        self.location_locks: Dict[str, threading.Lock] = {}
    
    def _get_location_lock(self, location_id: str):
        if location_id not in self.location_locks:
            self.location_locks[location_id] = threading.Lock()
        return self.location_locks[location_id]
    
    def assign_locker(self, package: Package, location_id: str) -> AssignResult:
        """Assign a locker to package (called by delivery person)."""
        with self._get_location_lock(location_id):
            location = self.locations.get(location_id)
            if not location:
                return AssignResult(False, "Location not found")
            
            available = [l for l in location.lockers 
                         if isinstance(l.state, AvailableState)]
            
            locker = self.assignment_strategy.assign(package.size, available)
            if not locker:
                return AssignResult(False, "No locker available")
            
            locker.assign(package)
            return AssignResult(True, locker.id, locker.row, locker.col)
    
    def deliver_package(self, locker_id: str) -> DeliverResult:
        """Confirm delivery and generate code (called by delivery person)."""
        locker = self.get_locker(locker_id)
        locker.deliver()
        
        self.code_mapping[locker.access_code] = locker
        return DeliverResult(True, locker.access_code)
    
    def pickup(self, code: str, location_id: str) -> PickupResult:
        """Customer pickup (called by kiosk)."""
        if code not in self.code_mapping:
            return PickupResult(False, "Invalid code")
        
        locker = self.code_mapping[code]
        if locker.location_id != location_id:
            return PickupResult(False, "Wrong location")
        
        try:
            locker.pickup(code)
            del self.code_mapping[code]
            return PickupResult(True, locker.id, "Locker opened!")
        except InvalidCodeError:
            return PickupResult(False, "Invalid code")
        except ExpiredCodeError:
            return PickupResult(False, "Code expired")
```

---

## ⏰ Expiry Handler (Cron Job)

```python
def handle_expired_packages():
    """Run every hour to handle expired packages."""
    current_time = datetime.now()
    
    for location in locker_system.locations.values():
        for locker in location.lockers:
            if isinstance(locker.state, OccupiedState):
                if current_time > locker.code_expires_at:
                    locker.expire()
                    # Notify warehouse for return pickup
                    schedule_return_pickup(locker.package)
```

---

## 🗄️ Database Schema

```sql
CREATE TABLE locker_locations (
    id VARCHAR(36) PRIMARY KEY,
    name VARCHAR(200),
    address TEXT,
    city VARCHAR(100)
);

CREATE TABLE lockers (
    id VARCHAR(36) PRIMARY KEY,
    location_id VARCHAR(36) REFERENCES locker_locations(id),
    size ENUM('SMALL', 'MEDIUM', 'LARGE', 'XL'),
    row_num INT,
    col_num INT,
    status ENUM('AVAILABLE', 'RESERVED', 'OCCUPIED', 'OUT_OF_SERVICE')
);

CREATE TABLE packages (
    id VARCHAR(36) PRIMARY KEY,
    order_id VARCHAR(36),
    user_id VARCHAR(36),
    size ENUM('SMALL', 'MEDIUM', 'LARGE', 'XL'),
    status ENUM('IN_TRANSIT', 'IN_LOCKER', 'PICKED_UP', 'RETURNED')
);

CREATE TABLE access_codes (
    code VARCHAR(10) PRIMARY KEY,
    locker_id VARCHAR(36) REFERENCES lockers(id),
    package_id VARCHAR(36) REFERENCES packages(id),
    user_id VARCHAR(36),
    generated_at TIMESTAMP,
    expires_at TIMESTAMP,
    used_at TIMESTAMP,
    is_used BOOLEAN DEFAULT FALSE
);
```

---

## 🏛️ Class Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  LockerSystem (Singleton)                                    │
│      ├── locations: Dict[str, LockerLocation]                │
│      ├── assignment_strategy: AssignmentStrategy             │
│      ├── assign_locker()                                     │
│      ├── deliver_package()                                   │
│      └── pickup()                                            │
│                                                              │
│  AssignmentStrategy (Abstract)     LockerState (Abstract)    │
│      ├── ExactFitStrategy              ├── AvailableState    │
│      ├── BestFitStrategy               ├── ReservedState     │
│      └── FirstAvailableStrategy        └── OccupiedState     │
│                                                              │
│  LockerLocation ──────► Locker ◄───── LockerState            │
│                          ├── size                            │
│                          ├── package                         │
│                          └── access_code                     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## ✅ Key Takeaways

| Pattern | Usage |
|---------|-------|
| **Strategy** | Locker assignment (ExactFit, BestFit) |
| **State** | Locker status transitions |
| **Singleton** | LockerSystem |

| Concept | Implementation |
|---------|----------------|
| **Concurrent assignment** | Per-location locking |
| **Code expiry** | Cron job + State.expire() |
| **Size matching** | Package size ≤ Locker size |

---

*This article was created as part of LLD interview practice for Amazon SDE-2 (L5) level.*
