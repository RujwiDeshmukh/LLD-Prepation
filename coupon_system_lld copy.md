# Low-Level Design: Coupon/Promo Code System

> A complete tutorial for designing a coupon system like Zepto, Swiggy, or Instamart — using **Strategy** and **Chain of Responsibility** patterns.

---

## 🎯 Problem Overview

When a user applies a coupon code, the system must:
1. Validate the coupon (expired? usage limit? eligible user?)
2. Calculate the discount based on coupon type
3. Apply the discount to the cart

---

## 📦 Types of Coupons

| Type | Example | How It Works |
|------|---------|--------------|
| **Flat Discount** | ₹50 off | Subtract fixed amount |
| **Percentage Discount** | 20% off (max ₹100) | Calculate % of cart total |
| **BOGO** | Buy 1 Get 1 Free | Add free item to cart |
| **Free Delivery** | Free delivery | Waive delivery fee |
| **Cashback** | ₹100 cashback | Credit after order completion |

---

## 🎯 Coupon Rules/Constraints

| Rule | Example |
|------|---------|
| **Minimum Order Value** | Valid on orders above ₹500 |
| **Maximum Discount** | Up to ₹200 off |
| **Valid Date Range** | Valid from 1st Jan to 31st Jan |
| **Global Usage Limit** | First 1000 users only |
| **Per User Limit** | One time per user |
| **Category Restriction** | Only on groceries |
| **Product Restriction** | Only on specific items |
| **User Eligibility** | New users only / Prime members |
| **Payment Method** | Only on HDFC cards |

---

## 🎨 Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Strategy** | Discount calculation | Different coupon types calculate differently |
| **Chain of Responsibility** | Validation rules | Multiple rules checked in sequence |
| **Factory** | Coupon creation | Create different coupon types |

---

## 🏗️ Core Entities

```
Coupon              → The coupon with code, rules, discount type
DiscountStrategy    → How to calculate discount
ValidationRule      → One validation check
CouponService       → Orchestrator
Cart                → User's shopping cart
User                → The customer
```

---

## 💻 Complete Implementation

### Supporting Classes

```python
from typing import List, Dict

class User:
    def __init__(self, user_id: str, name: str, is_prime: bool = False):
        self.id = user_id
        self.name = name
        self.is_prime = is_prime
        self.order_count = 0

class CartItem:
    def __init__(self, name: str, price: float, quantity: int, category: str):
        self.name = name
        self.price = price
        self.quantity = quantity
        self.category = category

class Cart:
    def __init__(self):
        self.items: List[CartItem] = []
        self.delivery_fee = 40.0
    
    def add_item(self, item: CartItem):
        self.items.append(item)
    
    def get_subtotal(self) -> float:
        return sum(item.price * item.quantity for item in self.items)
    
    def get_total(self) -> float:
        return self.get_subtotal() + self.delivery_fee
    
    def get_delivery_fee(self) -> float:
        return self.delivery_fee
```

---

### Discount Strategy (Strategy Pattern)

```python
from abc import ABC, abstractmethod

class DiscountStrategy(ABC):
    """Base class for all discount calculation strategies."""
    
    @abstractmethod
    def calculate_discount(self, cart: Cart) -> float:
        pass
    
    @abstractmethod
    def get_description(self) -> str:
        pass


class FlatDiscountStrategy(DiscountStrategy):
    """Fixed amount discount: ₹50 off, ₹100 off, etc."""
    
    def __init__(self, amount: float):
        self.amount = amount
    
    def calculate_discount(self, cart: Cart) -> float:
        return min(self.amount, cart.get_subtotal())
    
    def get_description(self) -> str:
        return f"₹{self.amount} off"


class PercentageDiscountStrategy(DiscountStrategy):
    """Percentage discount with optional max cap."""
    
    def __init__(self, percentage: float, max_discount: float = None):
        self.percentage = percentage
        self.max_discount = max_discount
    
    def calculate_discount(self, cart: Cart) -> float:
        discount = cart.get_subtotal() * self.percentage / 100
        if self.max_discount:
            discount = min(discount, self.max_discount)
        return discount
    
    def get_description(self) -> str:
        desc = f"{self.percentage}% off"
        if self.max_discount:
            desc += f" (up to ₹{self.max_discount})"
        return desc


class FreeDeliveryStrategy(DiscountStrategy):
    """Waives the delivery fee."""
    
    def calculate_discount(self, cart: Cart) -> float:
        return cart.get_delivery_fee()
    
    def get_description(self) -> str:
        return "Free Delivery"


class BOGOStrategy(DiscountStrategy):
    """Buy One Get One Free on specific items."""
    
    def __init__(self, applicable_items: List[str]):
        self.applicable_items = applicable_items
    
    def calculate_discount(self, cart: Cart) -> float:
        discount = 0
        for item in cart.items:
            if item.name in self.applicable_items:
                free_items = item.quantity // 2
                discount += free_items * item.price
        return discount
    
    def get_description(self) -> str:
        return "Buy 1 Get 1 Free"
```

---

### Validation Result

```python
class ValidationResult:
    """Result of a validation check."""
    
    def __init__(self, is_valid: bool, message: str):
        self.is_valid = is_valid
        self.message = message
    
    @staticmethod
    def success():
        return ValidationResult(True, "Valid")
    
    @staticmethod
    def failure(message: str):
        return ValidationResult(False, message)
```

---

### Validation Rules (Chain of Responsibility)

```python
from datetime import datetime
from typing import Optional

class ValidationRule(ABC):
    """Base class for validation rules using Chain of Responsibility."""
    
    def __init__(self):
        self.next_rule: Optional['ValidationRule'] = None
    
    def set_next(self, rule: 'ValidationRule') -> 'ValidationRule':
        self.next_rule = rule
        return rule
    
    @abstractmethod
    def validate(self, coupon: 'Coupon', cart: Cart, user: User) -> ValidationResult:
        pass
    
    def check_next(self, coupon, cart, user) -> ValidationResult:
        if self.next_rule:
            return self.next_rule.validate(coupon, cart, user)
        return ValidationResult.success()


class DateValidationRule(ValidationRule):
    """Check if coupon is within valid date range."""
    
    def validate(self, coupon, cart, user):
        now = datetime.now()
        if now < coupon.start_date:
            return ValidationResult.failure("Coupon is not yet active")
        if now > coupon.end_date:
            return ValidationResult.failure("Coupon has expired")
        return self.check_next(coupon, cart, user)


class MinOrderValueRule(ValidationRule):
    """Check minimum order value requirement."""
    
    def __init__(self, min_value: float):
        super().__init__()
        self.min_value = min_value
    
    def validate(self, coupon, cart, user):
        if cart.get_subtotal() < self.min_value:
            return ValidationResult.failure(
                f"Add ₹{self.min_value - cart.get_subtotal():.0f} more to apply this coupon"
            )
        return self.check_next(coupon, cart, user)


class UsageLimitRule(ValidationRule):
    """Check global and per-user usage limits."""
    
    def validate(self, coupon, cart, user):
        # Global limit
        if coupon.max_uses and coupon.current_uses >= coupon.max_uses:
            return ValidationResult.failure("Coupon usage limit reached")
        
        # Per user limit
        user_uses = coupon.user_usage.get(user.id, 0)
        if user_uses >= coupon.max_uses_per_user:
            return ValidationResult.failure("You have already used this coupon")
        
        return self.check_next(coupon, cart, user)


class NewUserOnlyRule(ValidationRule):
    """Restrict coupon to new users only."""
    
    def validate(self, coupon, cart, user):
        if user.order_count > 0:
            return ValidationResult.failure("This coupon is for new users only")
        return self.check_next(coupon, cart, user)


class PrimeUserOnlyRule(ValidationRule):
    """Restrict coupon to Prime/Premium members."""
    
    def validate(self, coupon, cart, user):
        if not user.is_prime:
            return ValidationResult.failure("This coupon is for Prime members only")
        return self.check_next(coupon, cart, user)


class CategoryRule(ValidationRule):
    """Restrict coupon to specific categories."""
    
    def __init__(self, allowed_categories: List[str]):
        super().__init__()
        self.allowed_categories = [c.lower() for c in allowed_categories]
    
    def validate(self, coupon, cart, user):
        for item in cart.items:
            if item.category.lower() not in self.allowed_categories:
                return ValidationResult.failure(
                    f"Coupon not valid on {item.category} items"
                )
        return self.check_next(coupon, cart, user)


class PaymentMethodRule(ValidationRule):
    """Restrict coupon to specific payment methods."""
    
    def __init__(self, allowed_methods: List[str], selected_method: str):
        super().__init__()
        self.allowed_methods = [m.lower() for m in allowed_methods]
        self.selected_method = selected_method.lower()
    
    def validate(self, coupon, cart, user):
        if self.selected_method not in self.allowed_methods:
            return ValidationResult.failure(
                f"Coupon valid only on: {', '.join(self.allowed_methods)}"
            )
        return self.check_next(coupon, cart, user)
```

---

### Building the Validation Chain

```python
def build_validation_chain(rules: List[ValidationRule]) -> Optional[ValidationRule]:
    """Link rules into a chain and return the head."""
    if not rules:
        return None
    
    head = rules[0]
    current = head
    
    for rule in rules[1:]:
        current.set_next(rule)
        current = rule
    
    return head
```

---

### Coupon Class

```python
class Coupon:
    """Represents a coupon with its rules and discount strategy."""
    
    def __init__(
        self,
        code: str,
        discount_strategy: DiscountStrategy,
        rules: List[ValidationRule],
        start_date: datetime,
        end_date: datetime,
        max_uses: int = None,
        max_uses_per_user: int = 1,
        description: str = ""
    ):
        self.code = code.upper()
        self.discount_strategy = discount_strategy
        self.rules = rules
        self.start_date = start_date
        self.end_date = end_date
        self.max_uses = max_uses
        self.max_uses_per_user = max_uses_per_user
        self.description = description or discount_strategy.get_description()
        
        # Usage tracking
        self.current_uses = 0
        self.user_usage: Dict[str, int] = {}
    
    def record_usage(self, user: User):
        self.current_uses += 1
        self.user_usage[user.id] = self.user_usage.get(user.id, 0) + 1
```

---

### Coupon Service (Orchestrator)

```python
class ApplyResult:
    """Result of applying a coupon."""
    
    def __init__(self, success: bool, discount: float, message: str):
        self.success = success
        self.discount = discount
        self.message = message


class CouponService:
    """Manages coupon creation, validation, and application."""
    
    def __init__(self):
        self.coupons: Dict[str, Coupon] = {}
    
    def create_coupon(self, coupon: Coupon):
        self.coupons[coupon.code] = coupon
        print(f"✓ Created coupon: {coupon.code} - {coupon.description}")
    
    def get_coupon(self, code: str) -> Optional[Coupon]:
        return self.coupons.get(code.upper())
    
    def apply_coupon(self, code: str, cart: Cart, user: User) -> ApplyResult:
        code = code.upper()
        
        # Check if coupon exists
        if code not in self.coupons:
            return ApplyResult(False, 0, "Invalid coupon code")
        
        coupon = self.coupons[code]
        
        # Run validation chain
        chain = build_validation_chain(coupon.rules)
        if chain:
            result = chain.validate(coupon, cart, user)
            if not result.is_valid:
                return ApplyResult(False, 0, result.message)
        
        # Calculate discount
        discount = coupon.discount_strategy.calculate_discount(cart)
        
        # Record usage
        coupon.record_usage(user)
        
        return ApplyResult(
            True, 
            discount, 
            f"🎉 Coupon applied! You saved ₹{discount:.0f}"
        )
    
    def list_available_coupons(self, cart: Cart, user: User) -> List[Coupon]:
        """Return coupons that are valid for this cart and user."""
        available = []
        for coupon in self.coupons.values():
            chain = build_validation_chain(coupon.rules)
            if chain:
                result = chain.validate(coupon, cart, user)
                if result.is_valid:
                    available.append(coupon)
            else:
                available.append(coupon)
        return available
```

---

## 🔄 Usage Examples

### Example 1: New User Discount

```python
from datetime import datetime, timedelta

service = CouponService()

# Create: 20% off (max ₹100), min order ₹500, new users only
new_user_coupon = Coupon(
    code="NEWUSER20",
    discount_strategy=PercentageDiscountStrategy(20, max_discount=100),
    rules=[
        DateValidationRule(),
        MinOrderValueRule(500),
        UsageLimitRule(),
        NewUserOnlyRule()
    ],
    start_date=datetime.now() - timedelta(days=1),
    end_date=datetime.now() + timedelta(days=30),
    max_uses=1000,
    max_uses_per_user=1
)
service.create_coupon(new_user_coupon)

# Test
user = User("u1", "Alice")  # New user (order_count=0)
cart = Cart()
cart.add_item(CartItem("Milk", 60, 2, "Dairy"))
cart.add_item(CartItem("Bread", 40, 1, "Bakery"))
cart.add_item(CartItem("Eggs", 80, 2, "Dairy"))
cart.add_item(CartItem("Rice", 200, 1, "Grocery"))
cart.add_item(CartItem("Oil", 150, 1, "Grocery"))

print(f"Cart Total: ₹{cart.get_subtotal()}")
result = service.apply_coupon("NEWUSER20", cart, user)
print(result.message)
# Output: 🎉 Coupon applied! You saved ₹100
```

### Example 2: Free Delivery

```python
free_delivery_coupon = Coupon(
    code="FREEDELIVERY",
    discount_strategy=FreeDeliveryStrategy(),
    rules=[
        DateValidationRule(),
        MinOrderValueRule(199)
    ],
    start_date=datetime.now() - timedelta(days=1),
    end_date=datetime.now() + timedelta(days=30)
)
service.create_coupon(free_delivery_coupon)

result = service.apply_coupon("FREEDELIVERY", cart, user)
print(result.message)
# Output: 🎉 Coupon applied! You saved ₹40
```

### Example 3: BOGO

```python
bogo_coupon = Coupon(
    code="BOGO_MILK",
    discount_strategy=BOGOStrategy(["Milk"]),
    rules=[DateValidationRule()],
    start_date=datetime.now() - timedelta(days=1),
    end_date=datetime.now() + timedelta(days=7)
)
service.create_coupon(bogo_coupon)
```

---

## 🏛️ Class Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  CouponService                                                  │
│       ├── coupons: Dict[str, Coupon]                            │
│       ├── create_coupon(coupon)                                 │
│       ├── apply_coupon(code, cart, user)                        │
│       └── list_available_coupons(cart, user)                    │
│                                                                 │
│  Coupon                                                         │
│       ├── code, start_date, end_date                            │
│       ├── discount_strategy: DiscountStrategy                   │
│       ├── rules: List[ValidationRule]                           │
│       └── usage tracking                                        │
│                                                                 │
│  DiscountStrategy (Abstract)          ValidationRule (Abstract) │
│       ├── FlatDiscountStrategy              ├── DateValidation  │
│       ├── PercentageDiscountStrategy        ├── MinOrderValue   │
│       ├── FreeDeliveryStrategy              ├── UsageLimit      │
│       └── BOGOStrategy                      ├── NewUserOnly     │
│                                             ├── CategoryRule    │
│                                             └── PaymentMethod   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Validation Flow

```
User applies "NEWUSER20"
        │
        ▼
DateValidationRule ──► Pass ──► MinOrderValueRule ──► Pass ──► UsageLimitRule
        │                               │                           │
        ▼                               ▼                           ▼
    (expired?)                    (cart >= ₹500?)              (used before?)
        │                               │                           │
        ▼                               ▼                           ▼
       Pass ────────────────────► NewUserOnlyRule ──► Pass ──► Calculate Discount
                                        │
                                        ▼
                                  (order_count = 0?)
```

---

## ✅ Key Takeaways

1. **Strategy Pattern** for discount types — easy to add new types
2. **Chain of Responsibility** for rules — easy to add/remove validations
3. **Each rule is independent** — Single Responsibility Principle
4. **Validation chain is configurable** — different coupons have different rules
5. **Usage tracking** prevents abuse

---

## 📝 Possible Extensions

| Feature | Implementation |
|---------|----------------|
| Stackable coupons | Allow multiple coupons per order |
| Time-based (Happy Hour) | Add TimeOfDayRule |
| First order on app | Add AppFirstOrderRule |
| Referral coupons | Link to referrer User |
| Auto-apply best coupon | Compare all valid coupons |

---

*This article was created as part of LLD learning for e-commerce systems.*
