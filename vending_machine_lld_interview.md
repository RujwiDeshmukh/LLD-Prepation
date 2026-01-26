# Low-Level Design Interview: Vending Machine

> A complete walkthrough of an LLD interview demonstrating the **State Pattern**, suitable for Amazon SDE-2 (L5) level interviews.

---

## 🎯 Problem Statement

**Interviewer:** Let's design a **Vending Machine** — the kind you see at airports and malls. Users insert money, select a product, and collect their item along with any change.

What clarifying questions do you have?

---

## 📋 Requirement Gathering

### Candidate's Questions

**Candidate:** I have several clarifying questions:

1. **Products**: What types of products? I've seen cans, chips, bottles, chocolates. Size might be a factor.

2. **Payment Flow**: User inserts money → sees available products → selects one → confirms → gets product + change?

3. **Cancel Operation**: Can the user change their mind after inserting money? We'd need to return their money.

4. **Edge Cases**:
   - What if no product is affordable with inserted amount?
   - What if selected product is out of stock?
   - What if user selects a product above their balance?

**Interviewer:** Great questions! Let me clarify:

| Question | Answer |
|----------|--------|
| **Products** | Any products — cans, chips, bottles. Each has price and quantity |
| **Payment** | Cash only. User inserts coins/notes |
| **Cancel** | ✅ Yes, user can cancel and get refund |
| **Change** | ✅ Return extra money after purchase |
| **Out of Stock** | Don't show unavailable products, but handle gracefully |

---

### Candidate's Smart Design Decision

**Candidate:** I'll only show products that are:
1. In stock (quantity > 0)
2. Affordable (price ≤ inserted amount)

This prevents invalid selections at the UI level. But I'll still handle edge cases in code defensively.

**Interviewer:** Excellent defensive thinking!

---

## 🔄 Identifying the State Pattern

**Interviewer:** You mentioned "reset the state of machine" — that's the key insight!

**Candidate:** Yes! The machine behaves differently depending on its state. The same action has different effects:

| Action | In IDLE State | In HAS_MONEY State |
|--------|---------------|-------------------|
| Select Product | ❌ "Insert money first" | ✅ Process selection |
| Cancel | ❌ "Nothing to cancel" | ✅ Refund money |

This is a perfect use case for the **State Pattern**!

---

## 📊 State Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│    ┌──────────┐      insert        ┌───────────┐                    │
│    │   IDLE   │ ─────────────────► │ HAS_MONEY │◄───┐               │
│    └────▲─────┘                    └─────┬─────┘    │               │
│         │                                │          │ insert more   │
│         │ cancel                  select │          │               │
│         │                                ▼          │               │
│         │                    ┌──────────────────┐   │               │
│         │◄───────────────────│ PRODUCT_SELECTED │───┘               │
│         │      cancel        │                  │ (change selection)│
│         │                    └────────┬─────────┘                   │
│         │                             │ confirm                     │
│         │                             ▼                             │
│         │                    ┌──────────────────┐                   │
│         │                    │    DISPENSING    │                   │
│         │                    └────────┬─────────┘                   │
│         │                             │                             │
│         │                             ▼                             │
│         │                    ┌──────────────────┐                   │
│         │                    │  RETURN_CHANGE   │                   │
│         │                    └────────┬─────────┘                   │
│         │                             │                             │
│         └─────────────────────────────┘                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📝 Final Requirements

| Feature | In Scope |
|---------|----------|
| Insert money (coins/notes) | ✅ |
| Show available products | ✅ |
| Select product by code | ✅ |
| Confirm selection | ✅ |
| Cancel and refund | ✅ |
| Dispense product | ✅ |
| Return change | ✅ |

---

## 🏗️ Core Entities

| Type | Classes |
|------|---------|
| **Domain Entities** | Product, VendingMachine |
| **State Classes** | VendingMachineState (abstract), IdleState, HasMoneyState, etc. |

**Candidate:** State classes are design constructs, not domain entities. I list domain entities first, then add behavioral classes when implementing the pattern.

**Interviewer:** Good distinction!

---

## 💻 Class Implementations

### Product Class

```python
class Product:
    """Represents a product in the vending machine."""
    
    def __init__(self, code: str, name: str, price: float, quantity: int):
        self.code = code        # e.g., "A1", "B2"
        self.name = name        # e.g., "Coca Cola"
        self.price = price      # e.g., 40.0
        self.quantity = quantity
    
    def is_available(self) -> bool:
        return self.quantity > 0
    
    def dispense(self) -> bool:
        """Reduce quantity by 1. Returns False if out of stock."""
        if self.quantity > 0:
            self.quantity -= 1
            return True
        return False
```

---

### State Pattern — Abstract State

```python
from abc import ABC

class VendingMachineState(ABC):
    """
    Abstract base class for vending machine states.
    Each state implements only the valid operations.
    Invalid operations raise errors (default behavior).
    """
    
    def __init__(self, machine: 'VendingMachine'):
        self.machine = machine
    
    def insert_coin(self, amount: float):
        raise InvalidOperationError(
            f"Cannot insert coin in {self.__class__.__name__}"
        )
    
    def select_product(self, code: str):
        raise InvalidOperationError(
            f"Cannot select product in {self.__class__.__name__}"
        )
    
    def confirm(self):
        raise InvalidOperationError(
            f"Cannot confirm in {self.__class__.__name__}"
        )
    
    def cancel(self):
        raise InvalidOperationError(
            f"Cannot cancel in {self.__class__.__name__}"
        )
    
    def dispense(self):
        raise InvalidOperationError(
            f"Cannot dispense in {self.__class__.__name__}"
        )


class InvalidOperationError(Exception):
    """Raised when an operation is invalid for current state."""
    pass
```

---

### Concrete States

#### IdleState

```python
class IdleState(VendingMachineState):
    """
    Initial state — waiting for user to insert money.
    Valid operations: insert_coin
    """
    
    def insert_coin(self, amount: float):
        if amount <= 0:
            print("Invalid amount")
            return
        
        self.machine.balance += amount
        print(f"Inserted: ₹{amount}. Balance: ₹{self.machine.balance}")
        
        # Show available products
        self._show_available_products()
        
        # Transition to HasMoney state
        self.machine.set_state(HasMoneyState(self.machine))
    
    def cancel(self):
        print("Nothing to cancel. Machine is idle.")
    
    def _show_available_products(self):
        print("\n--- Available Products ---")
        for product in self.machine.products:
            if product.is_available() and product.price <= self.machine.balance:
                print(f"  [{product.code}] {product.name} - ₹{product.price}")
        print("--------------------------\n")
```

#### HasMoneyState

```python
class HasMoneyState(VendingMachineState):
    """
    Money has been inserted — waiting for product selection.
    Valid operations: insert_coin, select_product, cancel
    """
    
    def insert_coin(self, amount: float):
        if amount <= 0:
            print("Invalid amount")
            return
        
        self.machine.balance += amount
        print(f"Added: ₹{amount}. Balance: ₹{self.machine.balance}")
        self._show_available_products()
        # Stay in same state
    
    def select_product(self, code: str):
        product = self.machine.get_product(code)
        
        if not product:
            print(f"Product {code} not found")
            return
        
        if not product.is_available():
            print(f"{product.name} is out of stock")
            return
        
        if product.price > self.machine.balance:
            print(f"Insufficient balance. Need ₹{product.price}, have ₹{self.machine.balance}")
            return
        
        # Valid selection
        self.machine.selected_product = product
        print(f"Selected: {product.name} (₹{product.price})")
        print("Press CONFIRM to purchase or CANCEL to abort")
        
        # Transition to ProductSelected state
        self.machine.set_state(ProductSelectedState(self.machine))
    
    def cancel(self):
        self._refund()
        self.machine.set_state(IdleState(self.machine))
    
    def _refund(self):
        print(f"Refunding: ₹{self.machine.balance}")
        self.machine.balance = 0
    
    def _show_available_products(self):
        print("\n--- Available Products ---")
        for product in self.machine.products:
            if product.is_available() and product.price <= self.machine.balance:
                print(f"  [{product.code}] {product.name} - ₹{product.price}")
        print("--------------------------\n")
```

#### ProductSelectedState

```python
class ProductSelectedState(VendingMachineState):
    """
    Product has been selected — waiting for confirmation.
    Valid operations: confirm, select_product (change), cancel
    """
    
    def confirm(self):
        product = self.machine.selected_product
        
        # Final validation
        if not product.is_available():
            print("Sorry, product just went out of stock!")
            self.machine.selected_product = None
            self.machine.set_state(HasMoneyState(self.machine))
            return
        
        print(f"Confirmed: {product.name}")
        
        # Transition to Dispensing
        self.machine.set_state(DispensingState(self.machine))
        
        # Auto-trigger dispensing
        self.machine.dispense()
    
    def select_product(self, code: str):
        # Allow changing selection
        product = self.machine.get_product(code)
        
        if product and product.is_available() and product.price <= self.machine.balance:
            self.machine.selected_product = product
            print(f"Changed selection to: {product.name}")
        else:
            print("Invalid selection")
    
    def cancel(self):
        self.machine.selected_product = None
        print(f"Selection cancelled. Refunding ₹{self.machine.balance}")
        self.machine.balance = 0
        self.machine.set_state(IdleState(self.machine))
```

#### DispensingState

```python
class DispensingState(VendingMachineState):
    """
    Dispensing the product.
    No user operations allowed — system is busy.
    """
    
    def dispense(self):
        product = self.machine.selected_product
        
        # Dispense product
        if product.dispense():
            print(f"🎉 Dispensing: {product.name}")
            
            # Deduct price from balance
            self.machine.balance -= product.price
            
            # Clear selection
            self.machine.selected_product = None
            
            # Transition to ReturnChange
            self.machine.set_state(ReturnChangeState(self.machine))
            
            # Auto-trigger change return
            self.machine.current_state.return_change()
        else:
            print("Dispensing failed!")
            self.machine.set_state(HasMoneyState(self.machine))
```

#### ReturnChangeState

```python
class ReturnChangeState(VendingMachineState):
    """
    Returning change to user.
    Transitions back to Idle after completion.
    """
    
    def return_change(self):
        if self.machine.balance > 0:
            print(f"💰 Returning change: ₹{self.machine.balance}")
            self.machine.balance = 0
        else:
            print("No change to return")
        
        print("Thank you! Machine is ready for next customer.\n")
        
        # Back to Idle
        self.machine.set_state(IdleState(self.machine))
```

---

### VendingMachine (Context)

```python
class VendingMachine:
    """
    The vending machine — context for the State Pattern.
    Delegates all operations to current state.
    """
    
    def __init__(self):
        self.products: List[Product] = []
        self.balance: float = 0.0
        self.selected_product: Product = None
        self.current_state: VendingMachineState = IdleState(self)
    
    # State management
    def set_state(self, state: VendingMachineState):
        print(f"[State: {state.__class__.__name__}]")
        self.current_state = state
    
    # Product management
    def add_product(self, product: Product):
        self.products.append(product)
    
    def get_product(self, code: str) -> Product:
        for product in self.products:
            if product.code == code:
                return product
        return None
    
    # Delegate operations to current state
    def insert_coin(self, amount: float):
        self.current_state.insert_coin(amount)
    
    def select_product(self, code: str):
        self.current_state.select_product(code)
    
    def confirm(self):
        self.current_state.confirm()
    
    def cancel(self):
        self.current_state.cancel()
    
    def dispense(self):
        self.current_state.dispense()
```

---

## 🔄 Complete Flow Walkthrough

### Scenario: User buys Coke with ₹50

```python
# Setup
machine = VendingMachine()
machine.add_product(Product("A1", "Coca Cola", 40, 5))
machine.add_product(Product("A2", "Pepsi", 35, 3))
machine.add_product(Product("B1", "Chips", 25, 10))

# User actions
machine.insert_coin(50)
machine.select_product("A1")
machine.confirm()
```

### Output

```
Inserted: ₹50. Balance: ₹50

--- Available Products ---
  [A1] Coca Cola - ₹40
  [A2] Pepsi - ₹35
  [B1] Chips - ₹25
--------------------------

[State: HasMoneyState]
Selected: Coca Cola (₹40)
Press CONFIRM to purchase or CANCEL to abort
[State: ProductSelectedState]
Confirmed: Coca Cola
[State: DispensingState]
🎉 Dispensing: Coca Cola
[State: ReturnChangeState]
💰 Returning change: ₹10
Thank you! Machine is ready for next customer.

[State: IdleState]
```

---

## 🎯 State Pattern Benefits

| Benefit | How It Helps |
|---------|--------------|
| **Clean Code** | No messy if-else chains checking state |
| **Single Responsibility** | Each state handles its own logic |
| **Open for Extension** | Add new states without modifying existing |
| **Easy to Debug** | State transitions are explicit and logged |

---

### Without State Pattern (Messy!) ❌

```python
def insert_coin(self, amount):
    if self.state == "IDLE":
        self.balance += amount
        self.state = "HAS_MONEY"
    elif self.state == "HAS_MONEY":
        self.balance += amount
    elif self.state == "DISPENSING":
        raise Error("Cannot insert while dispensing")
    # ... grows with every new state!
```

### With State Pattern (Clean!) ✅

```python
def insert_coin(self, amount):
    self.current_state.insert_coin(amount)
    # State handles the logic internally!
```

---

## 🏛️ Class Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  VendingMachine (Context)                                           │
│       │                                                             │
│       ├── products: List[Product]                                   │
│       ├── balance: float                                            │
│       ├── selected_product: Product                                 │
│       └── current_state: VendingMachineState ◄──────────────┐       │
│                                                              │       │
│                                                              │       │
│  VendingMachineState (Abstract)                              │       │
│       │                                                      │       │
│       ├── IdleState ─────────────────────────────────────────┤       │
│       ├── HasMoneyState ─────────────────────────────────────┤       │
│       ├── ProductSelectedState ──────────────────────────────┤       │
│       ├── DispensingState ───────────────────────────────────┤       │
│       └── ReturnChangeState ─────────────────────────────────┘       │
│                                                                     │
│                                                                     │
│  Product                                                            │
│       ├── code: str                                                 │
│       ├── name: str                                                 │
│       ├── price: float                                              │
│       └── quantity: int                                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🧪 Edge Cases Handled

| Edge Case | How Handled |
|-----------|-------------|
| Insert ₹0 or negative | Validation in insert_coin |
| Select unavailable product | Check quantity, show error |
| Select unaffordable product | Check balance, show error |
| Cancel at any point | Each state implements cancel appropriately |
| Product goes out of stock between select and confirm | Re-validate in confirm() |
| Invalid operation for state | Base class raises InvalidOperationError |

---

## 🔒 Thread Safety Note

**Interviewer:** What about concurrent users?

**Candidate:** A vending machine serves one user at a time — there's physical blocking. But if this were a software-only system (like a checkout process), I'd add:

```python
import threading

class VendingMachine:
    def __init__(self):
        self._lock = threading.Lock()
        # ...
    
    def insert_coin(self, amount):
        with self._lock:
            self.current_state.insert_coin(amount)
```

---

## 🎨 Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **State** | VendingMachineState hierarchy | Behavior changes based on state |
| **Singleton** | Could be applied to VendingMachine | One machine instance |

### Why NOT Strategy Pattern Here?

**Strategy** = Multiple algorithms for the **same operation** (e.g., different pricing strategies)

**State** = **Different operations available** based on current state

In vending machine, the **available actions change** depending on state — that's State Pattern!

---

## ✅ Interview Takeaways

1. **State Pattern** is ideal when object behavior depends on its state
2. Each state is a **separate class** — Single Responsibility Principle
3. **Transitions are explicit** — easy to trace and debug
4. **Default error handling** in base class keeps code DRY
5. **Context delegates** to current state — clean public API
6. Always **walk through a complete flow** to validate your design

---

## 🆚 State Pattern vs Strategy Pattern

| Aspect | State Pattern | Strategy Pattern |
|--------|--------------|------------------|
| **Purpose** | Behavior changes based on internal state | Algorithm can be swapped |
| **Transitions** | States change automatically | Strategy set externally |
| **Knows about context** | Yes, states often trigger transitions | Usually no |
| **Example** | Vending Machine, Order Status | Pricing, Sorting, Routing |

---

*This article was created as part of LLD interview practice for Amazon SDE-2 (L5) level.*
