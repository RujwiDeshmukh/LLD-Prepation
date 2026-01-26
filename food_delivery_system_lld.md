# Low-Level Design Interview: Food Delivery System

> Complete LLD for designing a food delivery platform like Swiggy, Zomato, or Uber Eats.

---

## 🎯 Problem Statement

**Interviewer:** Design a **Food Delivery System** where users can browse restaurants, order food, and get it delivered.

---

## 📋 Requirements

### Functional

| Feature | Description |
|---------|-------------|
| Browse restaurants | By location, cuisine, rating |
| View menu | See items with prices |
| Add to cart | Select items, quantity |
| Place order | Checkout with payment |
| Track order | Real-time status updates |
| Assign delivery partner | Based on proximity |
| Rate & review | After delivery |

### Non-Functional

| Requirement | Description |
|-------------|-------------|
| Handle concurrent orders | Multiple users ordering from same restaurant |
| Real-time tracking | Push updates to user |
| Low latency | Restaurant search should be fast |

---

## 🔄 Order Flow

```
User → Browse Restaurants → Select Restaurant → View Menu
    → Add Items to Cart → Apply Coupon (optional)
    → Place Order → Payment → Order Confirmed
    → Restaurant Accepts → Preparing → Ready for Pickup
    → Delivery Partner Assigned → Picked Up → On the Way
    → Delivered → User Rates
```

---

## 📡 API Contracts

### 1. Get Nearby Restaurants

```
GET /api/restaurants?lat=18.52&lng=73.85&radius=5km
```

**Response:**
```json
{
    "restaurants": [
        {
            "id": "REST_001",
            "name": "Domino's Pizza",
            "cuisine": ["Italian", "Fast Food"],
            "rating": 4.2,
            "avg_delivery_time": 35,
            "distance_km": 2.5,
            "is_open": true,
            "min_order": 200
        }
    ]
}
```

### 2. Get Restaurant Menu

```
GET /api/restaurants/{restaurant_id}/menu
```

**Response:**
```json
{
    "restaurant_id": "REST_001",
    "categories": [
        {
            "name": "Pizzas",
            "items": [
                {
                    "id": "ITEM_001",
                    "name": "Margherita Pizza",
                    "description": "Classic cheese pizza",
                    "price": 299,
                    "is_veg": true,
                    "is_available": true,
                    "customizations": [
                        {"name": "Extra Cheese", "price": 50},
                        {"name": "Jalapenos", "price": 30}
                    ]
                }
            ]
        }
    ]
}
```

### 3. Add to Cart

```
POST /api/cart/add
```

**Request:**
```json
{
    "restaurant_id": "REST_001",
    "item_id": "ITEM_001",
    "quantity": 2,
    "customizations": ["Extra Cheese"]
}
```

### 4. Place Order

```
POST /api/orders
```

**Request:**
```json
{
    "cart_id": "CART_001",
    "delivery_address": {
        "lat": 18.52,
        "lng": 73.85,
        "address_line": "123 Main St"
    },
    "payment_method": "UPI",
    "coupon_code": "FIRST50"
}
```

**Response:**
```json
{
    "order_id": "ORD_001",
    "status": "PLACED",
    "estimated_delivery": "2024-01-26T20:30:00Z",
    "bill": {
        "item_total": 598,
        "delivery_fee": 40,
        "taxes": 30,
        "discount": -50,
        "total": 618
    }
}
```

### 5. Track Order

```
GET /api/orders/{order_id}/track
```

**Response:**
```json
{
    "order_id": "ORD_001",
    "status": "ON_THE_WAY",
    "delivery_partner": {
        "name": "Rahul",
        "phone": "9999999999",
        "current_location": {"lat": 18.51, "lng": 73.84}
    },
    "eta_minutes": 10,
    "timeline": [
        {"status": "PLACED", "time": "19:45"},
        {"status": "ACCEPTED", "time": "19:47"},
        {"status": "PREPARING", "time": "19:48"},
        {"status": "PICKED_UP", "time": "20:10"},
        {"status": "ON_THE_WAY", "time": "20:12"}
    ]
}
```

---

## 🏗️ Core Entities

### Enums

```python
from enum import Enum

class OrderStatus(Enum):
    PLACED = "placed"
    ACCEPTED = "accepted"
    PREPARING = "preparing"
    READY = "ready"
    PICKED_UP = "picked_up"
    ON_THE_WAY = "on_the_way"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class PaymentStatus(Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    FAILED = "failed"
    REFUNDED = "refunded"

class DeliveryPartnerStatus(Enum):
    AVAILABLE = "available"
    BUSY = "busy"
    OFFLINE = "offline"
```

### Entity Classes

```python
from dataclasses import dataclass
from typing import List, Optional
from datetime import datetime
import uuid

@dataclass
class Location:
    latitude: float
    longitude: float
    address: str

@dataclass
class User:
    id: str
    name: str
    email: str
    phone: str
    addresses: List[Location]

@dataclass
class MenuItem:
    id: str
    name: str
    description: str
    price: float
    is_veg: bool
    is_available: bool
    category: str
    customizations: List[dict]

@dataclass
class Restaurant:
    id: str
    name: str
    location: Location
    cuisines: List[str]
    menu: List[MenuItem]
    rating: float
    is_open: bool
    min_order_amount: float
    avg_prep_time_mins: int

@dataclass
class CartItem:
    menu_item: MenuItem
    quantity: int
    customizations: List[str]
    
    def get_total(self) -> float:
        base = self.menu_item.price * self.quantity
        # Add customization prices
        return base

@dataclass
class Cart:
    id: str
    user: User
    restaurant: Restaurant
    items: List[CartItem]
    
    def get_subtotal(self) -> float:
        return sum(item.get_total() for item in self.items)

@dataclass
class DeliveryPartner:
    id: str
    name: str
    phone: str
    current_location: Location
    status: DeliveryPartnerStatus
    rating: float
    vehicle_type: str  # BIKE, SCOOTER

@dataclass
class Order:
    id: str
    user: User
    restaurant: Restaurant
    items: List[CartItem]
    delivery_address: Location
    status: OrderStatus
    delivery_partner: Optional[DeliveryPartner]
    payment_status: PaymentStatus
    
    item_total: float
    delivery_fee: float
    taxes: float
    discount: float
    total: float
    
    placed_at: datetime
    delivered_at: Optional[datetime]
    estimated_delivery: datetime
```

---

## 🎨 Design Patterns

### 1. Strategy Pattern: Delivery Partner Assignment

```python
from abc import ABC, abstractmethod

class DeliveryAssignmentStrategy(ABC):
    @abstractmethod
    def assign(self, order: Order, 
               available_partners: List[DeliveryPartner]) -> Optional[DeliveryPartner]:
        pass

class NearestPartnerStrategy(DeliveryAssignmentStrategy):
    """Assign the nearest available delivery partner."""
    
    def assign(self, order, available_partners):
        if not available_partners:
            return None
        
        restaurant_loc = order.restaurant.location
        
        # Sort by distance from restaurant
        partners = sorted(
            available_partners,
            key=lambda p: self._calculate_distance(p.current_location, restaurant_loc)
        )
        
        return partners[0] if partners else None
    
    def _calculate_distance(self, loc1: Location, loc2: Location) -> float:
        # Haversine formula (simplified)
        return ((loc1.latitude - loc2.latitude)**2 + 
                (loc1.longitude - loc2.longitude)**2) ** 0.5

class HighestRatedPartnerStrategy(DeliveryAssignmentStrategy):
    """Assign the highest-rated available partner."""
    
    def assign(self, order, available_partners):
        if not available_partners:
            return None
        
        partners = sorted(available_partners, key=lambda p: p.rating, reverse=True)
        return partners[0]

class LoadBalancedStrategy(DeliveryAssignmentStrategy):
    """Balance orders among available partners."""
    
    def __init__(self):
        self.partner_order_count = {}  # partner_id -> order count today
    
    def assign(self, order, available_partners):
        if not available_partners:
            return None
        
        # Choose partner with least orders today
        partners = sorted(
            available_partners,
            key=lambda p: self.partner_order_count.get(p.id, 0)
        )
        
        selected = partners[0]
        self.partner_order_count[selected.id] = self.partner_order_count.get(selected.id, 0) + 1
        return selected
```

---

### 2. State Pattern: Order Status

```python
class OrderState(ABC):
    @abstractmethod
    def next(self, order: 'OrderContext') -> bool:
        pass
    
    @abstractmethod
    def cancel(self, order: 'OrderContext') -> bool:
        pass

class PlacedState(OrderState):
    def next(self, order):
        order.set_state(AcceptedState())
        order.notify_observers("Order accepted by restaurant")
        return True
    
    def cancel(self, order):
        order.set_state(CancelledState())
        order.process_refund()
        return True

class AcceptedState(OrderState):
    def next(self, order):
        order.set_state(PreparingState())
        order.notify_observers("Restaurant is preparing your order")
        return True
    
    def cancel(self, order):
        order.set_state(CancelledState())
        order.process_refund()
        return True

class PreparingState(OrderState):
    def next(self, order):
        order.set_state(ReadyState())
        order.assign_delivery_partner()
        order.notify_observers("Order is ready for pickup")
        return True
    
    def cancel(self, order):
        return False  # Cannot cancel while preparing

class ReadyState(OrderState):
    def next(self, order):
        order.set_state(PickedUpState())
        order.notify_observers("Order picked up by delivery partner")
        return True
    
    def cancel(self, order):
        return False

class PickedUpState(OrderState):
    def next(self, order):
        order.set_state(OnTheWayState())
        order.notify_observers("Order is on the way")
        return True
    
    def cancel(self, order):
        return False

class OnTheWayState(OrderState):
    def next(self, order):
        order.set_state(DeliveredState())
        order.mark_delivered()
        order.notify_observers("Order delivered!")
        return True
    
    def cancel(self, order):
        return False

class DeliveredState(OrderState):
    def next(self, order):
        return False  # No next state
    
    def cancel(self, order):
        return False

class CancelledState(OrderState):
    def next(self, order):
        return False
    
    def cancel(self, order):
        return False

class OrderContext:
    def __init__(self, order: Order):
        self.order = order
        self.state = PlacedState()
        self.observers = []
    
    def set_state(self, state: OrderState):
        self.state = state
        self.order.status = self._get_status_enum()
    
    def _get_status_enum(self) -> OrderStatus:
        mapping = {
            PlacedState: OrderStatus.PLACED,
            AcceptedState: OrderStatus.ACCEPTED,
            PreparingState: OrderStatus.PREPARING,
            ReadyState: OrderStatus.READY,
            PickedUpState: OrderStatus.PICKED_UP,
            OnTheWayState: OrderStatus.ON_THE_WAY,
            DeliveredState: OrderStatus.DELIVERED,
            CancelledState: OrderStatus.CANCELLED,
        }
        return mapping.get(type(self.state), OrderStatus.PLACED)
    
    def next_status(self):
        return self.state.next(self)
    
    def cancel_order(self):
        return self.state.cancel(self)
```

---

### 3. Observer Pattern: Order Tracking

```python
class OrderObserver(ABC):
    @abstractmethod
    def update(self, order_id: str, message: str):
        pass

class CustomerNotifier(OrderObserver):
    def update(self, order_id: str, message: str):
        # Push notification to customer app
        print(f"📱 Customer notification: {message}")

class RestaurantNotifier(OrderObserver):
    def update(self, order_id: str, message: str):
        # Update restaurant dashboard
        print(f"🍕 Restaurant dashboard: {message}")

class DeliveryPartnerNotifier(OrderObserver):
    def update(self, order_id: str, message: str):
        # Notify delivery partner app
        print(f"🛵 Delivery partner: {message}")

class OrderContext:
    def __init__(self, order: Order):
        self.order = order
        self.state = PlacedState()
        self.observers: List[OrderObserver] = []
    
    def add_observer(self, observer: OrderObserver):
        self.observers.append(observer)
    
    def notify_observers(self, message: str):
        for observer in self.observers:
            observer.update(self.order.id, message)
```

---

## 💻 Food Delivery Service

```python
import threading
from typing import Dict, List

class FoodDeliveryService:
    _instance = None
    _lock = threading.Lock()
    
    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance
    
    def __init__(self):
        self.restaurants: Dict[str, Restaurant] = {}
        self.orders: Dict[str, OrderContext] = {}
        self.delivery_partners: Dict[str, DeliveryPartner] = {}
        self.carts: Dict[str, Cart] = {}
        
        self.assignment_strategy = NearestPartnerStrategy()
        self.order_lock = threading.Lock()
    
    # ─────────────────────────────────────────
    # Restaurant Operations
    # ─────────────────────────────────────────
    
    def get_nearby_restaurants(self, location: Location, 
                                radius_km: float) -> List[Restaurant]:
        nearby = []
        for restaurant in self.restaurants.values():
            if restaurant.is_open:
                distance = self._calculate_distance(location, restaurant.location)
                if distance <= radius_km:
                    nearby.append(restaurant)
        
        # Sort by rating
        return sorted(nearby, key=lambda r: r.rating, reverse=True)
    
    def get_restaurant_menu(self, restaurant_id: str) -> List[MenuItem]:
        restaurant = self.restaurants.get(restaurant_id)
        if restaurant:
            return [item for item in restaurant.menu if item.is_available]
        return []
    
    # ─────────────────────────────────────────
    # Cart Operations
    # ─────────────────────────────────────────
    
    def add_to_cart(self, user: User, restaurant_id: str, 
                    item_id: str, quantity: int) -> Cart:
        cart_id = f"CART_{user.id}"
        
        if cart_id not in self.carts:
            restaurant = self.restaurants[restaurant_id]
            self.carts[cart_id] = Cart(cart_id, user, restaurant, [])
        
        cart = self.carts[cart_id]
        
        # Validate same restaurant
        if cart.restaurant.id != restaurant_id:
            raise ValueError("Cannot add items from different restaurant")
        
        # Find menu item
        menu_item = next(
            (item for item in cart.restaurant.menu if item.id == item_id), 
            None
        )
        
        if menu_item:
            cart.items.append(CartItem(menu_item, quantity, []))
        
        return cart
    
    # ─────────────────────────────────────────
    # Order Operations
    # ─────────────────────────────────────────
    
    def place_order(self, cart_id: str, delivery_address: Location,
                    coupon_code: str = None) -> Order:
        with self.order_lock:
            cart = self.carts.get(cart_id)
            if not cart:
                raise ValueError("Cart not found")
            
            # Calculate bill
            item_total = cart.get_subtotal()
            delivery_fee = self._calculate_delivery_fee(
                cart.restaurant.location, delivery_address
            )
            taxes = item_total * 0.05  # 5% tax
            discount = self._apply_coupon(coupon_code, item_total)
            total = item_total + delivery_fee + taxes - discount
            
            # Create order
            order = Order(
                id=f"ORD_{uuid.uuid4().hex[:8]}",
                user=cart.user,
                restaurant=cart.restaurant,
                items=cart.items,
                delivery_address=delivery_address,
                status=OrderStatus.PLACED,
                delivery_partner=None,
                payment_status=PaymentStatus.COMPLETED,
                item_total=item_total,
                delivery_fee=delivery_fee,
                taxes=taxes,
                discount=discount,
                total=total,
                placed_at=datetime.now(),
                delivered_at=None,
                estimated_delivery=self._calculate_eta(cart.restaurant, delivery_address)
            )
            
            # Create order context with observers
            order_context = OrderContext(order)
            order_context.add_observer(CustomerNotifier())
            order_context.add_observer(RestaurantNotifier())
            
            self.orders[order.id] = order_context
            
            # Clear cart
            del self.carts[cart_id]
            
            return order
    
    def accept_order(self, order_id: str) -> bool:
        order_context = self.orders.get(order_id)
        if order_context:
            return order_context.next_status()  # PLACED → ACCEPTED
        return False
    
    def update_order_status(self, order_id: str) -> bool:
        order_context = self.orders.get(order_id)
        if order_context:
            return order_context.next_status()
        return False
    
    def assign_delivery_partner(self, order_id: str) -> Optional[DeliveryPartner]:
        order_context = self.orders.get(order_id)
        if not order_context:
            return None
        
        order = order_context.order
        
        # Get available partners
        available = [
            p for p in self.delivery_partners.values()
            if p.status == DeliveryPartnerStatus.AVAILABLE
        ]
        
        # Use strategy to assign
        partner = self.assignment_strategy.assign(order, available)
        
        if partner:
            partner.status = DeliveryPartnerStatus.BUSY
            order.delivery_partner = partner
            order_context.add_observer(DeliveryPartnerNotifier())
        
        return partner
    
    def cancel_order(self, order_id: str) -> bool:
        order_context = self.orders.get(order_id)
        if order_context:
            return order_context.cancel_order()
        return False
    
    # ─────────────────────────────────────────
    # Helpers
    # ─────────────────────────────────────────
    
    def _calculate_distance(self, loc1: Location, loc2: Location) -> float:
        return ((loc1.latitude - loc2.latitude)**2 + 
                (loc1.longitude - loc2.longitude)**2) ** 0.5 * 111  # Rough km conversion
    
    def _calculate_delivery_fee(self, restaurant_loc: Location, 
                                 delivery_loc: Location) -> float:
        distance = self._calculate_distance(restaurant_loc, delivery_loc)
        base_fee = 20
        per_km_fee = 5
        return base_fee + (distance * per_km_fee)
    
    def _apply_coupon(self, coupon_code: str, amount: float) -> float:
        # Simplified coupon logic
        coupons = {
            "FIRST50": 50,
            "FLAT100": 100,
        }
        return coupons.get(coupon_code, 0)
    
    def _calculate_eta(self, restaurant: Restaurant, 
                       delivery_loc: Location) -> datetime:
        prep_time = restaurant.avg_prep_time_mins
        distance = self._calculate_distance(restaurant.location, delivery_loc)
        travel_time = distance * 3  # ~3 mins per km
        
        total_mins = prep_time + travel_time + 10  # 10 min buffer
        return datetime.now() + timedelta(minutes=total_mins)
```

---

## 🔄 Order State Transitions

```
┌────────┐   accept   ┌──────────┐  start prep  ┌───────────┐
│ PLACED │ ─────────► │ ACCEPTED │ ───────────► │ PREPARING │
└────────┘            └──────────┘              └───────────┘
     │                                                │
     │ cancel                                  ready  │
     ▼                                                ▼
┌───────────┐                                   ┌───────┐
│ CANCELLED │                                   │ READY │
└───────────┘                                   └───────┘
                                                      │
     ┌──────────────────────────────────────────────┘
     │ picked up
     ▼
┌───────────┐   en route   ┌────────────┐  arrive  ┌───────────┐
│ PICKED_UP │ ───────────► │ ON_THE_WAY │ ───────► │ DELIVERED │
└───────────┘              └────────────┘          └───────────┘
```

---

## 🗄️ Database Schema

```sql
CREATE TABLE users (
    id VARCHAR(36) PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100) UNIQUE,
    phone VARCHAR(15)
);

CREATE TABLE restaurants (
    id VARCHAR(36) PRIMARY KEY,
    name VARCHAR(200),
    latitude DECIMAL(10, 8),
    longitude DECIMAL(11, 8),
    address TEXT,
    rating DECIMAL(2, 1),
    is_open BOOLEAN,
    min_order_amount DECIMAL(10, 2),
    avg_prep_time INT
);

CREATE TABLE menu_items (
    id VARCHAR(36) PRIMARY KEY,
    restaurant_id VARCHAR(36) REFERENCES restaurants(id),
    name VARCHAR(200),
    description TEXT,
    price DECIMAL(10, 2),
    category VARCHAR(50),
    is_veg BOOLEAN,
    is_available BOOLEAN
);

CREATE TABLE delivery_partners (
    id VARCHAR(36) PRIMARY KEY,
    name VARCHAR(100),
    phone VARCHAR(15),
    current_lat DECIMAL(10, 8),
    current_lng DECIMAL(11, 8),
    status ENUM('AVAILABLE', 'BUSY', 'OFFLINE'),
    rating DECIMAL(2, 1)
);

CREATE TABLE orders (
    id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(36) REFERENCES users(id),
    restaurant_id VARCHAR(36) REFERENCES restaurants(id),
    delivery_partner_id VARCHAR(36) REFERENCES delivery_partners(id),
    status ENUM('PLACED', 'ACCEPTED', 'PREPARING', 'READY', 
                'PICKED_UP', 'ON_THE_WAY', 'DELIVERED', 'CANCELLED'),
    delivery_address TEXT,
    delivery_lat DECIMAL(10, 8),
    delivery_lng DECIMAL(11, 8),
    item_total DECIMAL(10, 2),
    delivery_fee DECIMAL(10, 2),
    taxes DECIMAL(10, 2),
    discount DECIMAL(10, 2),
    total DECIMAL(10, 2),
    placed_at TIMESTAMP,
    delivered_at TIMESTAMP,
    estimated_delivery TIMESTAMP
);

CREATE TABLE order_items (
    order_id VARCHAR(36) REFERENCES orders(id),
    menu_item_id VARCHAR(36) REFERENCES menu_items(id),
    quantity INT,
    price DECIMAL(10, 2),
    PRIMARY KEY (order_id, menu_item_id)
);
```

---

## 🏛️ Class Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  FoodDeliveryService (Singleton)                                │
│      ├── restaurants, orders, delivery_partners                 │
│      ├── assignment_strategy: DeliveryAssignmentStrategy        │
│      ├── get_nearby_restaurants()                               │
│      ├── place_order()                                          │
│      └── assign_delivery_partner()                              │
│                                                                 │
│  DeliveryAssignmentStrategy (Abstract)                          │
│      ├── NearestPartnerStrategy                                 │
│      ├── HighestRatedPartnerStrategy                            │
│      └── LoadBalancedStrategy                                   │
│                                                                 │
│  OrderState (Abstract)          OrderObserver (Abstract)        │
│      ├── PlacedState                ├── CustomerNotifier        │
│      ├── AcceptedState              ├── RestaurantNotifier      │
│      ├── PreparingState             └── DeliveryPartnerNotifier │
│      ├── ReadyState                                             │
│      ├── PickedUpState                                          │
│      ├── OnTheWayState                                          │
│      └── DeliveredState                                         │
│                                                                 │
│  Restaurant ──── MenuItem                                       │
│       │                                                         │
│       └──── Order ──── OrderContext ──── OrderState             │
│                │                                                │
│                └──── DeliveryPartner                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## ✅ Key Takeaways

| Pattern | Usage |
|---------|-------|
| **Singleton** | FoodDeliveryService |
| **Strategy** | Delivery partner assignment |
| **State** | Order status transitions |
| **Observer** | Order tracking notifications |

| Concept | Implementation |
|---------|----------------|
| **Location-based search** | Calculate distance, filter by radius |
| **ETA calculation** | Prep time + travel time |
| **Pricing** | Item total + delivery fee + tax - discount |
| **Order locking** | Prevent concurrent order issues |

---

## 🎯 Interview Tips

1. **Start with entities** — User, Restaurant, Order, DeliveryPartner
2. **Order status is key** — State machine for transitions
3. **Partner assignment** — Strategy pattern for flexibility
4. **Real-time tracking** — Observer for notifications
5. **Concurrent orders** — Locking when placing orders

---

*This article was created as part of LLD interview practice for Amazon SDE-2 (L5) level.*
