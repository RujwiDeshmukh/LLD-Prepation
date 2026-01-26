# Low-Level Design Interview: Stock Trading Platform

> A complete LLD walkthrough for designing a stock trading platform like Zerodha, Groww, or Robinhood — demonstrating **Strategy** and **Observer** patterns.

---

## 🎯 Problem Statement

**Interviewer:** Design a **Stock Trading Platform** where users can buy and sell stocks, view their portfolio, and place different types of orders.

---

## 📋 Requirements

### Functional Requirements

| Feature | Description |
|---------|-------------|
| Account Management | Create account, add funds, verify KYC |
| Browse Stocks | Search stocks, view prices |
| Place Orders | Market and Limit orders |
| Portfolio | View holdings, P&L |
| Transaction History | Order history with status |

### Non-Functional Requirements

| Requirement | Description |
|-------------|-------------|
| Concurrency | Handle simultaneous orders from same user |
| Data Consistency | Balance and holdings must be accurate |
| Low Latency | Order execution should be fast |
| Persistence | Data must survive restarts |

---

## 🔄 User Flow

```
1. User creates account → KYC verification
2. User adds funds to account
3. User searches stocks → Views prices
4. User places order (MARKET or LIMIT)
   - MARKET: Executes immediately
   - LIMIT: Waits for target price
5. Order executes → Holdings + balance updated
6. User views portfolio and P&L
7. User views transaction history
```

---

## 📡 API Contracts

### 1. Search Stocks

```
GET /api/stocks?query=RELIANCE
```

**Response:**
```json
{
    "stocks": [
        {
            "symbol": "RELIANCE",
            "name": "Reliance Industries Ltd",
            "current_price": 2450.50,
            "change": 25.30,
            "change_percent": 1.04
        }
    ]
}
```

### 2. Get Stock Details

```
GET /api/stocks/RELIANCE
```

**Response:**
```json
{
    "symbol": "RELIANCE",
    "name": "Reliance Industries Ltd",
    "current_price": 2450.50,
    "open_price": 2420.00,
    "high_price": 2465.00,
    "low_price": 2410.00,
    "volume": 5000000,
    "market_cap": 1650000000000
}
```

### 3. Place Order

```
POST /api/orders
```

**Request:**
```json
{
    "stock_symbol": "RELIANCE",
    "order_type": "LIMIT",
    "side": "BUY",
    "quantity": 10,
    "limit_price": 2400.00
}
```

**Response:**
```json
{
    "order_id": "ORD_20240126_001",
    "status": "PENDING",
    "message": "Limit order placed successfully",
    "stock_symbol": "RELIANCE",
    "quantity": 10,
    "limit_price": 2400.00,
    "created_at": "2024-01-26T10:30:00Z"
}
```

### 4. Get Portfolio

```
GET /api/portfolio
```

**Response:**
```json
{
    "account_balance": 50000.00,
    "invested_value": 100000.00,
    "current_value": 115000.00,
    "total_pnl": 15000.00,
    "pnl_percent": 15.0,
    "holdings": [
        {
            "symbol": "RELIANCE",
            "name": "Reliance Industries",
            "quantity": 10,
            "avg_buy_price": 2400.00,
            "current_price": 2500.00,
            "invested_value": 24000.00,
            "current_value": 25000.00,
            "pnl": 1000.00,
            "pnl_percent": 4.17
        }
    ]
}
```

### 5. Get Order History

```
GET /api/orders?status=ALL
```

**Response:**
```json
{
    "orders": [
        {
            "order_id": "ORD_001",
            "stock_symbol": "RELIANCE",
            "order_type": "MARKET",
            "side": "BUY",
            "quantity": 10,
            "limit_price": null,
            "executed_price": 2450.50,
            "status": "EXECUTED",
            "created_at": "2024-01-26T10:00:00Z",
            "executed_at": "2024-01-26T10:00:01Z"
        }
    ]
}
```

---

## 🏗️ Core Entities

### Enums

```python
from enum import Enum

class OrderType(Enum):
    MARKET = "MARKET"
    LIMIT = "LIMIT"

class OrderSide(Enum):
    BUY = "BUY"
    SELL = "SELL"

class OrderStatus(Enum):
    PENDING = "PENDING"
    EXECUTED = "EXECUTED"
    CANCELLED = "CANCELLED"
    EXPIRED = "EXPIRED"
```

### User

```python
class User:
    def __init__(self, name: str, email: str, pan: str):
        self.id = str(uuid.uuid4())
        self.name = name
        self.email = email
        self.pan = pan
        self.balance = 0.0
        self.is_verified = False
        self.created_at = datetime.now()
    
    def add_funds(self, amount: float):
        self.balance += amount
    
    def withdraw_funds(self, amount: float) -> bool:
        if self.balance >= amount:
            self.balance -= amount
            return True
        return False
```

### Stock

```python
class Stock:
    def __init__(self, symbol: str, name: str, current_price: float):
        self.symbol = symbol
        self.name = name
        self.current_price = current_price
        self.high_price = current_price
        self.low_price = current_price
        self.observers = []  # For Observer pattern
    
    def update_price(self, new_price: float):
        self.current_price = new_price
        self.high_price = max(self.high_price, new_price)
        self.low_price = min(self.low_price, new_price)
        self._notify_observers()
    
    def add_observer(self, observer):
        self.observers.append(observer)
    
    def remove_observer(self, observer):
        self.observers.remove(observer)
    
    def _notify_observers(self):
        for observer in self.observers:
            observer.on_price_update(self)
```

### Order

```python
class Order:
    def __init__(self, user: User, stock: Stock, order_type: OrderType,
                 side: OrderSide, quantity: int, limit_price: float = None):
        self.id = f"ORD_{datetime.now().strftime('%Y%m%d')}_{uuid.uuid4().hex[:6]}"
        self.user = user
        self.stock = stock
        self.order_type = order_type
        self.side = side
        self.quantity = quantity
        self.limit_price = limit_price
        self.executed_price = None
        self.status = OrderStatus.PENDING
        self.created_at = datetime.now()
        self.executed_at = None
```

### Holding

```python
class Holding:
    def __init__(self, user: User, stock: Stock, quantity: int, avg_buy_price: float):
        self.user = user
        self.stock = stock
        self.quantity = quantity
        self.avg_buy_price = avg_buy_price
    
    def add_shares(self, quantity: int, price: float):
        """Update holding when buying more of same stock."""
        total_cost = (self.avg_buy_price * self.quantity) + (price * quantity)
        self.quantity += quantity
        self.avg_buy_price = total_cost / self.quantity
    
    def remove_shares(self, quantity: int) -> bool:
        if self.quantity >= quantity:
            self.quantity -= quantity
            return True
        return False
    
    def get_current_value(self) -> float:
        return self.quantity * self.stock.current_price
    
    def get_pnl(self) -> float:
        return self.get_current_value() - (self.quantity * self.avg_buy_price)
```

---

## 🎨 Design Patterns

### Strategy Pattern for Order Execution

```python
from abc import ABC, abstractmethod

class OrderStrategy(ABC):
    @abstractmethod
    def execute(self, order: Order, trading_service: 'TradingService') -> bool:
        pass

class MarketOrderStrategy(OrderStrategy):
    """Execute immediately at current market price."""
    
    def execute(self, order: Order, trading_service: 'TradingService') -> bool:
        current_price = order.stock.current_price
        return trading_service.execute_order(order, current_price)

class LimitOrderStrategy(OrderStrategy):
    """Subscribe to price updates and execute when target reached."""
    
    def execute(self, order: Order, trading_service: 'TradingService') -> bool:
        # Check if limit price already satisfied
        current_price = order.stock.current_price
        
        if order.side == OrderSide.BUY and current_price <= order.limit_price:
            return trading_service.execute_order(order, current_price)
        elif order.side == OrderSide.SELL and current_price >= order.limit_price:
            return trading_service.execute_order(order, current_price)
        
        # Subscribe to price updates (Observer pattern)
        order.stock.add_observer(LimitOrderObserver(order, trading_service))
        return True  # Order placed, waiting for execution
```

### Observer Pattern for Limit Orders

```python
class LimitOrderObserver:
    """Watches stock price and triggers order when limit reached."""
    
    def __init__(self, order: Order, trading_service: 'TradingService'):
        self.order = order
        self.trading_service = trading_service
    
    def on_price_update(self, stock: Stock):
        if self.order.status != OrderStatus.PENDING:
            stock.remove_observer(self)
            return
        
        current_price = stock.current_price
        should_execute = False
        
        if self.order.side == OrderSide.BUY and current_price <= self.order.limit_price:
            should_execute = True
        elif self.order.side == OrderSide.SELL and current_price >= self.order.limit_price:
            should_execute = True
        
        if should_execute:
            self.trading_service.execute_order(self.order, current_price)
            stock.remove_observer(self)
```

---

## 💻 Trading Service

```python
import threading
from typing import Dict, List, Optional

class TradingService:
    def __init__(self):
        self.users: Dict[str, User] = {}
        self.stocks: Dict[str, Stock] = {}
        self.orders: List[Order] = []
        self.holdings: Dict[str, Dict[str, Holding]] = {}  # user_id -> {symbol -> Holding}
        
        # Per-user locks for thread safety
        self.user_locks: Dict[str, threading.Lock] = {}
        self.global_lock = threading.Lock()
        
        # Strategy mapping
        self.strategies = {
            OrderType.MARKET: MarketOrderStrategy(),
            OrderType.LIMIT: LimitOrderStrategy()
        }
    
    def _get_user_lock(self, user_id: str) -> threading.Lock:
        with self.global_lock:
            if user_id not in self.user_locks:
                self.user_locks[user_id] = threading.Lock()
            return self.user_locks[user_id]
    
    # ─────────────────────────────────────────
    # Place Order
    # ─────────────────────────────────────────
    
    def place_order(self, user: User, stock_symbol: str, order_type: OrderType,
                    side: OrderSide, quantity: int, limit_price: float = None) -> Order:
        
        stock = self.stocks.get(stock_symbol)
        if not stock:
            raise ValueError(f"Stock {stock_symbol} not found")
        
        order = Order(user, stock, order_type, side, quantity, limit_price)
        self.orders.append(order)
        
        # Use strategy pattern
        strategy = self.strategies[order_type]
        strategy.execute(order, self)
        
        return order
    
    # ─────────────────────────────────────────
    # Execute Order (with locking)
    # ─────────────────────────────────────────
    
    def execute_order(self, order: Order, execution_price: float) -> bool:
        with self._get_user_lock(order.user.id):
            if order.side == OrderSide.BUY:
                return self._execute_buy(order, execution_price)
            else:
                return self._execute_sell(order, execution_price)
    
    def _execute_buy(self, order: Order, price: float) -> bool:
        total_cost = order.quantity * price
        
        # Validate balance
        if order.user.balance < total_cost:
            order.status = OrderStatus.CANCELLED
            return False
        
        # Deduct balance
        order.user.balance -= total_cost
        
        # Update or create holding
        user_holdings = self.holdings.setdefault(order.user.id, {})
        
        if order.stock.symbol in user_holdings:
            holding = user_holdings[order.stock.symbol]
            holding.add_shares(order.quantity, price)
        else:
            user_holdings[order.stock.symbol] = Holding(
                order.user, order.stock, order.quantity, price
            )
        
        # Update order
        order.executed_price = price
        order.executed_at = datetime.now()
        order.status = OrderStatus.EXECUTED
        
        return True
    
    def _execute_sell(self, order: Order, price: float) -> bool:
        user_holdings = self.holdings.get(order.user.id, {})
        holding = user_holdings.get(order.stock.symbol)
        
        # Validate holdings
        if not holding or holding.quantity < order.quantity:
            order.status = OrderStatus.CANCELLED
            return False
        
        # Update holding
        holding.remove_shares(order.quantity)
        
        # Remove holding if empty
        if holding.quantity == 0:
            del user_holdings[order.stock.symbol]
        
        # Add to balance
        total_value = order.quantity * price
        order.user.balance += total_value
        
        # Update order
        order.executed_price = price
        order.executed_at = datetime.now()
        order.status = OrderStatus.EXECUTED
        
        return True
    
    # ─────────────────────────────────────────
    # Portfolio
    # ─────────────────────────────────────────
    
    def get_portfolio(self, user: User) -> dict:
        user_holdings = self.holdings.get(user.id, {})
        
        holdings_data = []
        total_invested = 0
        total_current = 0
        
        for symbol, holding in user_holdings.items():
            invested = holding.quantity * holding.avg_buy_price
            current = holding.get_current_value()
            pnl = holding.get_pnl()
            
            holdings_data.append({
                'symbol': symbol,
                'quantity': holding.quantity,
                'avg_buy_price': holding.avg_buy_price,
                'current_price': holding.stock.current_price,
                'invested_value': invested,
                'current_value': current,
                'pnl': pnl,
                'pnl_percent': (pnl / invested * 100) if invested > 0 else 0
            })
            
            total_invested += invested
            total_current += current
        
        return {
            'account_balance': user.balance,
            'invested_value': total_invested,
            'current_value': total_current,
            'total_pnl': total_current - total_invested,
            'pnl_percent': ((total_current - total_invested) / total_invested * 100) if total_invested > 0 else 0,
            'holdings': holdings_data
        }
```

---

## 🔄 Complete Flow Example

```python
# Setup
service = TradingService()

# Add stocks
reliance = Stock("RELIANCE", "Reliance Industries", 2450.00)
tcs = Stock("TCS", "Tata Consultancy Services", 3800.00)
service.stocks["RELIANCE"] = reliance
service.stocks["TCS"] = tcs

# Create user and add funds
user = User("Vinay", "vinay@email.com", "ABCDE1234F")
user.add_funds(100000)
service.users[user.id] = user

# Place market order
order1 = service.place_order(
    user, "RELIANCE", OrderType.MARKET, 
    OrderSide.BUY, quantity=10
)
# Executes immediately at 2450, costs 24500

# Place limit order
order2 = service.place_order(
    user, "TCS", OrderType.LIMIT,
    OrderSide.BUY, quantity=5, limit_price=3700
)
# Waits until TCS price drops to 3700

# Simulate price drop
tcs.update_price(3700)  # Triggers limit order execution

# Check portfolio
portfolio = service.get_portfolio(user)
print(f"Balance: {portfolio['account_balance']}")
print(f"Holdings: {portfolio['holdings']}")
```

---

## 🗄️ Database Schema

```sql
CREATE TABLE users (
    id VARCHAR(36) PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    pan VARCHAR(10) UNIQUE NOT NULL,
    balance DECIMAL(15,2) DEFAULT 0,
    is_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE stocks (
    symbol VARCHAR(20) PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    current_price DECIMAL(12,2),
    high_price DECIMAL(12,2),
    low_price DECIMAL(12,2),
    volume BIGINT,
    market_cap DECIMAL(20,2)
);

CREATE TABLE orders (
    id VARCHAR(50) PRIMARY KEY,
    user_id VARCHAR(36) REFERENCES users(id),
    stock_symbol VARCHAR(20) REFERENCES stocks(symbol),
    order_type ENUM('MARKET', 'LIMIT') NOT NULL,
    side ENUM('BUY', 'SELL') NOT NULL,
    quantity INT NOT NULL,
    limit_price DECIMAL(12,2),
    executed_price DECIMAL(12,2),
    status ENUM('PENDING', 'EXECUTED', 'CANCELLED', 'EXPIRED') DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    executed_at TIMESTAMP,
    INDEX idx_user_orders (user_id, created_at),
    INDEX idx_status (status)
);

CREATE TABLE holdings (
    user_id VARCHAR(36) REFERENCES users(id),
    stock_symbol VARCHAR(20) REFERENCES stocks(symbol),
    quantity INT NOT NULL,
    avg_buy_price DECIMAL(12,2) NOT NULL,
    PRIMARY KEY (user_id, stock_symbol)
);

-- Trigger to update holding on order execution
-- (In practice, this would be in application logic)
```

---

## 🏛️ Class Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  TradingService                                                 │
│       ├── users, stocks, orders, holdings                       │
│       ├── place_order()                                         │
│       ├── execute_order() ── with user lock                     │
│       └── get_portfolio()                                       │
│                                                                 │
│  OrderStrategy (Abstract)          Stock                        │
│       ├── MarketOrderStrategy           ├── symbol, price       │
│       └── LimitOrderStrategy            └── observers (Observer)│
│                                                                 │
│  Order                             Holding                      │
│       ├── type, side, quantity          ├── quantity            │
│       ├── limit_price                   └── avg_buy_price       │
│       └── status                                                │
│                                                                 │
│  LimitOrderObserver                                             │
│       └── on_price_update() ── triggers execute when limit hit  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## ✅ Key Takeaways

| Concept | Implementation |
|---------|----------------|
| **Strategy Pattern** | Different execution for MARKET vs LIMIT orders |
| **Observer Pattern** | LIMIT orders subscribe to price changes |
| **Per-user Locking** | Prevent concurrent order issues |
| **Avg Price Calculation** | Weighted average for additional buys |
| **Partial Sell Handling** | Reduce quantity, don't delete holding |
| **API Contracts** | Complete request/response structures |

---

## 🔄 Order Flow Summary

```
MARKET ORDER:
place_order() → MarketOrderStrategy → execute_order() immediately

LIMIT ORDER:
place_order() → LimitOrderStrategy → Subscribe to stock price
    │
    └── When price reaches limit:
        on_price_update() → execute_order()
```

---

*This article was created as part of LLD interview practice for Amazon SDE-2 (L5) level.*
