# Low-Level Design Interview: Pizza Ordering System

> A complete walkthrough demonstrating the **Decorator Pattern** — when to use it, why it fits, and how it differs from other patterns.

---

## 🎯 Problem Statement

**Interviewer:** Design a **Pizza Ordering System** where customers can build custom pizzas with various toppings and options. We need to calculate the final price.

---

## 🤔 Why Decorator Pattern?

Before diving into code, let's understand **why** Decorator is the right choice here.

### The Problem We're Solving

A customer orders:
- Base: Margherita Pizza (₹200)
- Add: Extra Cheese (₹40)
- Add: Olives (₹30)
- Add: Mushrooms (₹35)
- Add: Extra Cheese again (₹40)

**Total: ₹345**

### Why NOT Other Approaches?

#### ❌ Inheritance Approach

```python
class MargheritaWithCheeseAndOlives(Pizza): ...
class MargheritaWithCheeseAndMushrooms(Pizza): ...
class MargheritaWithDoubleCheeseAndOlives(Pizza): ...
```

**Problem:** Class explosion! With 10 toppings, you'd need 2^10 = 1024 classes!

#### ❌ Simple Composition

```python
class Pizza:
    def __init__(self):
        self.toppings = []
    
    def add_topping(self, topping):
        self.toppings.append(topping)
    
    def get_cost(self):
        return base_cost + sum(t.cost for t in self.toppings)
```

**Problem:** This works, but:
- All toppings must have same interface
- Can't have complex per-topping behavior
- Harder to extend with new topping types

#### ✅ Decorator Pattern

```python
pizza = MargheritaPizza()           # ₹200
pizza = ExtraCheese(pizza)          # +₹40
pizza = Olives(pizza)               # +₹30
pizza = ExtraCheese(pizza)          # +₹40 (double cheese!)

print(pizza.get_cost())             # ₹310
```

**Why it works:**
- Each decorator wraps the previous object
- Same interface throughout
- Can stack same decorator multiple times
- Easy to add new decorators
- No class explosion

---

## 🎨 Decorator Pattern Explained

### Core Concept

> **Decorator Pattern** attaches additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing.

### Visual Representation

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Without Decorator (Inheritance Hell)                          │
│   ────────────────────────────────────                          │
│                                                                 │
│   Pizza                                                         │
│     ├── Margherita                                              │
│     │     ├── MargheritaWithCheese                              │
│     │     │     ├── MargheritaWithCheeseAndOlives               │
│     │     │     └── MargheritaWithCheeseAndMushrooms            │
│     │     └── MargheritaWithOlives                              │
│     │           └── ...                                         │
│     └── ... (explosion!)                                        │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   With Decorator (Clean!)                                       │
│   ───────────────────────                                       │
│                                                                 │
│   Pizza (Component)                                             │
│     ├── MargheritaPizza (Concrete Component)                    │
│     ├── Farmhouse (Concrete Component)                          │
│     └── PizzaDecorator (wraps Pizza)                            │
│           ├── ExtraCheese                                       │
│           ├── Olives                                            │
│           ├── Mushrooms                                         │
│           └── Jalapenos                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### How Wrapping Works

```
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│   pizza = MargheritaPizza()                                   │
│                                                               │
│   ┌─────────────────┐                                         │
│   │  MargheritaPizza │  cost: ₹200                            │
│   └─────────────────┘                                         │
│                                                               │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│   pizza = ExtraCheese(pizza)                                  │
│                                                               │
│   ┌─────────────────────────────────────┐                     │
│   │  ExtraCheese                        │  +₹40               │
│   │    ┌─────────────────┐              │                     │
│   │    │  MargheritaPizza │  cost: ₹200 │                     │
│   │    └─────────────────┘              │                     │
│   └─────────────────────────────────────┘                     │
│                                                               │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│   pizza = Olives(pizza)                                       │
│                                                               │
│   ┌─────────────────────────────────────────────────────┐     │
│   │  Olives                                             │ +₹30│
│   │    ┌─────────────────────────────────────┐          │     │
│   │    │  ExtraCheese                        │  +₹40    │     │
│   │    │    ┌─────────────────┐              │          │     │
│   │    │    │  MargheritaPizza │  cost: ₹200 │          │     │
│   │    │    └─────────────────┘              │          │     │
│   │    └─────────────────────────────────────┘          │     │
│   └─────────────────────────────────────────────────────┘     │
│                                                               │
│   pizza.get_cost() → 30 + 40 + 200 = ₹270                     │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

---

## 💻 Complete Implementation

### Step 1: Component Interface (Pizza)

```python
from abc import ABC, abstractmethod

class Pizza(ABC):
    """
    Component interface.
    Both concrete pizzas and decorators implement this.
    """
    
    @abstractmethod
    def get_cost(self) -> float:
        pass
    
    @abstractmethod
    def get_description(self) -> str:
        pass
```

---

### Step 2: Concrete Components (Base Pizzas)

```python
class MargheritaPizza(Pizza):
    """A basic Margherita pizza."""
    
    def get_cost(self) -> float:
        return 200
    
    def get_description(self) -> str:
        return "Margherita"


class FarmhousePizza(Pizza):
    """Farmhouse pizza with veggies."""
    
    def get_cost(self) -> float:
        return 300
    
    def get_description(self) -> str:
        return "Farmhouse"


class PepperoniPizza(Pizza):
    """Pepperoni pizza."""
    
    def get_cost(self) -> float:
        return 350
    
    def get_description(self) -> str:
        return "Pepperoni"
```

---

### Step 3: Decorator Base Class

```python
class PizzaDecorator(Pizza):
    """
    Decorator base class.
    Wraps a Pizza and delegates to it.
    """
    
    def __init__(self, pizza: Pizza):
        self.pizza = pizza  # The wrapped component
    
    @abstractmethod
    def get_cost(self) -> float:
        pass
    
    @abstractmethod
    def get_description(self) -> str:
        pass
```

---

### Step 4: Concrete Decorators (Toppings)

```python
class ExtraCheese(PizzaDecorator):
    """Adds extra cheese to any pizza."""
    
    def get_cost(self) -> float:
        return 40 + self.pizza.get_cost()  # Add to wrapped pizza's cost
    
    def get_description(self) -> str:
        return self.pizza.get_description() + " + Extra Cheese"


class Olives(PizzaDecorator):
    """Adds olives to any pizza."""
    
    def get_cost(self) -> float:
        return 30 + self.pizza.get_cost()
    
    def get_description(self) -> str:
        return self.pizza.get_description() + " + Olives"


class Mushrooms(PizzaDecorator):
    """Adds mushrooms to any pizza."""
    
    def get_cost(self) -> float:
        return 35 + self.pizza.get_cost()
    
    def get_description(self) -> str:
        return self.pizza.get_description() + " + Mushrooms"


class Jalapenos(PizzaDecorator):
    """Adds jalapenos to any pizza."""
    
    def get_cost(self) -> float:
        return 25 + self.pizza.get_cost()
    
    def get_description(self) -> str:
        return self.pizza.get_description() + " + Jalapenos"


class ThinCrust(PizzaDecorator):
    """Changes to thin crust (premium option)."""
    
    def get_cost(self) -> float:
        return 50 + self.pizza.get_cost()
    
    def get_description(self) -> str:
        return self.pizza.get_description() + " (Thin Crust)"
```

---

## 🔄 Usage Examples

### Example 1: Simple Order

```python
pizza = MargheritaPizza()
pizza = ExtraCheese(pizza)

print(pizza.get_description())  # "Margherita + Extra Cheese"
print(pizza.get_cost())         # 240
```

### Example 2: Loaded Pizza

```python
pizza = FarmhousePizza()        # ₹300
pizza = ExtraCheese(pizza)      # +₹40
pizza = Olives(pizza)           # +₹30
pizza = Mushrooms(pizza)        # +₹35
pizza = ThinCrust(pizza)        # +₹50

print(pizza.get_description())  
# "Farmhouse + Extra Cheese + Olives + Mushrooms (Thin Crust)"

print(pizza.get_cost())         # ₹455
```

### Example 3: Double Cheese

```python
pizza = MargheritaPizza()       # ₹200
pizza = ExtraCheese(pizza)      # +₹40
pizza = ExtraCheese(pizza)      # +₹40 (again!)

print(pizza.get_description())  
# "Margherita + Extra Cheese + Extra Cheese"

print(pizza.get_cost())         # ₹280
```

---

## 🔄 How get_cost() Works (Recursion)

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   pizza = Olives(ExtraCheese(MargheritaPizza()))                │
│                                                                 │
│   pizza.get_cost() is called:                                   │
│                                                                 │
│   Olives.get_cost()                                             │
│       │                                                         │
│       └── return 30 + self.pizza.get_cost()                     │
│                           │                                     │
│                           └── ExtraCheese.get_cost()            │
│                                   │                             │
│                                   └── return 40 + self.pizza.get_cost()
│                                                   │             │
│                                                   └── Margherita.get_cost()
│                                                           │     │
│                                                           └── return 200
│                                                                 │
│   Result: 30 + 40 + 200 = 270                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🏛️ Class Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   <<interface>>                                                     │
│   Pizza                                                             │
│   ─────────────────────────                                         │
│   + get_cost(): float                                               │
│   + get_description(): str                                          │
│            ▲                                                        │
│            │                                                        │
│   ┌────────┴────────┐                                               │
│   │                 │                                               │
│   │     ┌───────────┴───────────┐                                   │
│   │     │                       │                                   │
│   │  Concrete Pizzas      PizzaDecorator                            │
│   │  ──────────────       ───────────────                           │
│   │  MargheritaPizza      - pizza: Pizza  ◄─── wraps a Pizza        │
│   │  FarmhousePizza       + get_cost()                              │
│   │  PepperoniPizza       + get_description()                       │
│   │                              ▲                                  │
│   │                              │                                  │
│   │               ┌──────────────┼──────────────┐                   │
│   │               │              │              │                   │
│   │          ExtraCheese     Olives      Mushrooms                  │
│   │                                                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🆚 Decorator vs Other Patterns

| Pattern | When to Use | Example |
|---------|-------------|---------|
| **Decorator** | Add features dynamically at runtime | Pizza toppings |
| **Strategy** | Swap entire algorithm | Pricing strategies |
| **Builder** | Complex construction with many steps | Building a meal |
| **Composite** | Tree structure, same operations on whole/parts | File system |

### Decorator vs Strategy

| Aspect | Decorator | Strategy |
|--------|-----------|----------|
| **Purpose** | Add behavior | Swap behavior |
| **Stacking** | ✅ Multiple decorators | ❌ One strategy at a time |
| **Example** | Pizza + Cheese + Olives | HourlyPricing vs FlatPricing |

### Decorator vs Inheritance

| Aspect | Decorator | Inheritance |
|--------|-----------|-------------|
| **When** | Runtime | Compile time |
| **Flexibility** | High | Low |
| **Class count** | O(n) | O(2^n) |
| **Same decorator twice** | ✅ Yes | ❌ No |

---

## 🎯 When to Use Decorator

Use Decorator when:

| Condition | Example |
|-----------|---------|
| Need to add features dynamically | Pizza toppings |
| Want to stack features | Multiple toppings |
| Same feature can be added multiple times | Double cheese |
| Avoid class explosion from inheritance | 10 toppings → 1024 classes |
| Features should be composable | Any combination of toppings |

---

## 🧪 Real-World Use Cases

| Domain | Base Object | Decorators |
|--------|-------------|------------|
| **Food** | Pizza, Coffee, Burger | Toppings, add-ons |
| **I/O Streams** | FileStream | BufferedStream, EncryptedStream |
| **UI** | Window | ScrollBar, Border, Shadow |
| **Logging** | BaseLogger | TimestampDecorator, FileDecorator |

### Java I/O Example

```java
InputStream file = new FileInputStream("data.txt");
InputStream buffered = new BufferedInputStream(file);
InputStream zipped = new GZIPInputStream(buffered);

// Each wraps the previous!
```

---

## ✅ Interview Takeaways

1. **Decorator = wrapping** — each decorator wraps the previous object
2. **Same interface** — decorators and components share the same interface
3. **Recursive methods** — `get_cost()` calls `pizza.get_cost()`
4. **Stackable** — can apply same decorator multiple times
5. **Open-Closed** — add new decorators without modifying existing code
6. **Avoids class explosion** — alternative to deep inheritance

---

## 📝 Quick Reference

```python
# Pattern structure
class Component(ABC):
    @abstractmethod
    def operation(self): pass

class ConcreteComponent(Component):
    def operation(self):
        return "base"

class Decorator(Component):
    def __init__(self, component: Component):
        self.component = component
    
    def operation(self):
        return f"decorated({self.component.operation()})"
```

---

*This article was created as part of LLD interview practice for Amazon SDE-2 (L5) level.*
