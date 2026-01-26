# Low-Level Design Interview: Splitwise

> A complete walkthrough demonstrating **Strategy**, **Builder**, and **Singleton** patterns for an expense sharing system.

---

## 🎯 Problem Statement

**Interviewer:** Design **Splitwise** — an app that helps groups track shared expenses and figure out who owes whom.

---

## 📋 Requirements

| Feature | In Scope |
|---------|----------|
| Create users and groups | ✅ |
| Add expenses with different split types | ✅ |
| Equal, Exact, Percentage splits | ✅ |
| View balances (who owes whom) | ✅ |
| Simplify debts | ✅ |
| Single currency | ✅ |
| Multi-currency | ❌ |
| Notifications | ❌ |
| Payment integration | ❌ |

---

## 🏗️ Core Entities

| Entity | Purpose |
|--------|---------|
| **User** | Person who participates in expenses |
| **Group** | Collection of users sharing expenses |
| **Expense** | A shared cost (who paid, how to split) |
| **Split** | Individual share of an expense |
| **BalanceSheet** | Tracks what each user owes/is owed |
| **SplitStrategy** | Algorithm for dividing expense |
| **SplitwiseService** | Orchestrator (Singleton) |

---

## 🎨 Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Strategy** | SplitStrategy | Different ways to split (equal, exact, %) |
| **Builder** | ExpenseBuilder | Complex expense creation |
| **Singleton** | SplitwiseService | Single orchestrator instance |

---

## 💻 Complete Implementation

### User

```python
import uuid
from balance_sheet import BalanceSheet

class User:
    def __init__(self, name: str, email: str):
        self._id = str(uuid.uuid4())
        self._name = name
        self._email = email
        self._balance_sheet = BalanceSheet(self)
    
    def get_id(self) -> str:
        return self._id
    
    def get_name(self) -> str:
        return self._name
    
    def get_balance_sheet(self) -> 'BalanceSheet':
        return self._balance_sheet
```

---

### BalanceSheet

```python
import threading
from typing import Dict

class BalanceSheet:
    """Tracks what a user owes to or is owed by others."""
    
    def __init__(self, owner: 'User'):
        self._owner = owner
        self._balances: Dict['User', float] = {}
        self._lock = threading.Lock()
    
    def get_balances(self) -> Dict['User', float]:
        return self._balances
    
    def adjust_balance(self, other_user: 'User', amount: float):
        """
        Positive amount = other_user owes owner
        Negative amount = owner owes other_user
        """
        with self._lock:
            if self._owner == other_user:
                return
            
            if other_user in self._balances:
                self._balances[other_user] += amount
            else:
                self._balances[other_user] = amount
    
    def show_balances(self):
        print(f"--- Balance Sheet for {self._owner.get_name()} ---")
        if not self._balances:
            print("All settled up!")
            return
        
        for other_user, amount in self._balances.items():
            if amount > 0.01:
                print(f"{other_user.get_name()} owes {self._owner.get_name()} ${amount:.2f}")
            elif amount < -0.01:
                print(f"{self._owner.get_name()} owes {other_user.get_name()} ${-amount:.2f}")
```

---

### Split

```python
class Split:
    """Represents one person's share of an expense."""
    
    def __init__(self, user: 'User', amount: float):
        self._user = user
        self._amount = amount
    
    def get_user(self) -> 'User':
        return self._user
    
    def get_amount(self) -> float:
        return self._amount
```

---

### SplitStrategy (Strategy Pattern)

```python
from abc import ABC, abstractmethod
from typing import List, Optional

class SplitStrategy(ABC):
    @abstractmethod
    def calculate_splits(
        self, 
        total_amount: float, 
        paid_by: 'User', 
        participants: List['User'], 
        split_values: Optional[List[float]]
    ) -> List['Split']:
        pass


class EqualSplitStrategy(SplitStrategy):
    """Split equally among all participants."""
    
    def calculate_splits(self, total_amount, paid_by, participants, split_values):
        amount_per_person = total_amount / len(participants)
        return [Split(p, amount_per_person) for p in participants]


class ExactSplitStrategy(SplitStrategy):
    """Split by exact amounts specified."""
    
    def calculate_splits(self, total_amount, paid_by, participants, split_values):
        if len(participants) != len(split_values):
            raise ValueError("Participants and values count mismatch")
        if abs(sum(split_values) - total_amount) > 0.01:
            raise ValueError("Exact amounts must sum to total")
        
        return [Split(participants[i], split_values[i]) 
                for i in range(len(participants))]


class PercentageSplitStrategy(SplitStrategy):
    """Split by percentages."""
    
    def calculate_splits(self, total_amount, paid_by, participants, split_values):
        if len(participants) != len(split_values):
            raise ValueError("Participants and values count mismatch")
        if abs(sum(split_values) - 100.0) > 0.01:
            raise ValueError("Percentages must sum to 100")
        
        return [Split(participants[i], total_amount * split_values[i] / 100) 
                for i in range(len(participants))]
```

---

### Expense (Builder Pattern)

```python
from datetime import datetime
from typing import List, Optional

class Expense:
    def __init__(self, builder: 'ExpenseBuilder'):
        self._id = builder._id
        self._description = builder._description
        self._amount = builder._amount
        self._paid_by = builder._paid_by
        self._timestamp = datetime.now()
        
        # Use strategy to calculate splits
        self._splits = builder._split_strategy.calculate_splits(
            builder._amount, 
            builder._paid_by, 
            builder._participants, 
            builder._split_values
        )
    
    def get_amount(self) -> float:
        return self._amount
    
    def get_paid_by(self) -> 'User':
        return self._paid_by
    
    def get_splits(self) -> List['Split']:
        return self._splits
    
    def get_description(self) -> str:
        return self._description


class ExpenseBuilder:
    def __init__(self):
        self._id: Optional[str] = None
        self._description: Optional[str] = None
        self._amount: Optional[float] = None
        self._paid_by: Optional['User'] = None
        self._participants: Optional[List['User']] = None
        self._split_strategy: Optional['SplitStrategy'] = None
        self._split_values: Optional[List[float]] = None
    
    def set_id(self, expense_id: str) -> 'ExpenseBuilder':
        self._id = expense_id
        return self
    
    def set_description(self, description: str) -> 'ExpenseBuilder':
        self._description = description
        return self
    
    def set_amount(self, amount: float) -> 'ExpenseBuilder':
        self._amount = amount
        return self
    
    def set_paid_by(self, paid_by: 'User') -> 'ExpenseBuilder':
        self._paid_by = paid_by
        return self
    
    def set_participants(self, participants: List['User']) -> 'ExpenseBuilder':
        self._participants = participants
        return self
    
    def set_split_strategy(self, strategy: 'SplitStrategy') -> 'ExpenseBuilder':
        self._split_strategy = strategy
        return self
    
    def set_split_values(self, values: List[float]) -> 'ExpenseBuilder':
        self._split_values = values
        return self
    
    def build(self) -> 'Expense':
        if self._split_strategy is None:
            raise ValueError("Split strategy is required")
        return Expense(self)
```

---

### Group

```python
import uuid
from typing import List

class Group:
    def __init__(self, name: str, members: List['User']):
        self._id = str(uuid.uuid4())
        self._name = name
        self._members = members
    
    def get_id(self) -> str:
        return self._id
    
    def get_name(self) -> str:
        return self._name
    
    def get_members(self) -> List['User']:
        return self._members.copy()
```

---

### SplitwiseService (Singleton)

```python
import threading
from typing import Dict, List, Optional

class SplitwiseService:
    """Singleton orchestrator for the expense sharing system."""
    
    _instance = None
    _lock = threading.Lock()
    
    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
                    cls._instance._initialized = False
        return cls._instance
    
    def __init__(self):
        if not self._initialized:
            self._users: Dict[str, User] = {}
            self._groups: Dict[str, Group] = {}
            self._initialized = True
    
    @classmethod
    def get_instance(cls):
        return cls()
    
    def add_user(self, name: str, email: str) -> User:
        user = User(name, email)
        self._users[user.get_id()] = user
        return user
    
    def add_group(self, name: str, members: List[User]) -> Group:
        group = Group(name, members)
        self._groups[group.get_id()] = group
        return group
    
    def create_expense(self, builder: ExpenseBuilder):
        """Create expense and update all balances."""
        with self._lock:
            expense = builder.build()
            paid_by = expense.get_paid_by()
            
            for split in expense.get_splits():
                participant = split.get_user()
                amount = split.get_amount()
                
                if paid_by != participant:
                    # Participant owes paid_by
                    paid_by.get_balance_sheet().adjust_balance(participant, amount)
                    participant.get_balance_sheet().adjust_balance(paid_by, -amount)
    
    def settle_up(self, payer: User, payee: User, amount: float):
        """Record a settlement payment."""
        with self._lock:
            payee.get_balance_sheet().adjust_balance(payer, -amount)
            payer.get_balance_sheet().adjust_balance(payee, amount)
    
    def simplify_group_debts(self, group_id: str) -> List['Transaction']:
        """Minimize number of transactions within a group."""
        group = self._groups.get(group_id)
        if not group:
            raise ValueError("Group not found")
        
        # Calculate net balance for each member
        net_balances = {}
        members = group.get_members()
        
        for member in members:
            balance = 0
            for other, amount in member.get_balance_sheet().get_balances().items():
                if other in members:
                    balance += amount
            net_balances[member] = balance
        
        # Separate creditors and debtors
        creditors = [(u, b) for u, b in net_balances.items() if b > 0.01]
        debtors = [(u, b) for u, b in net_balances.items() if b < -0.01]
        
        creditors.sort(key=lambda x: x[1], reverse=True)
        debtors.sort(key=lambda x: x[1])
        
        # Match debtors to creditors
        transactions = []
        i = j = 0
        
        while i < len(creditors) and j < len(debtors):
            creditor, credit_amt = creditors[i]
            debtor, debt_amt = debtors[j]
            
            settle_amt = min(credit_amt, -debt_amt)
            transactions.append(Transaction(debtor, creditor, settle_amt))
            
            creditors[i] = (creditor, credit_amt - settle_amt)
            debtors[j] = (debtor, debt_amt + settle_amt)
            
            if abs(creditors[i][1]) < 0.01:
                i += 1
            if abs(debtors[j][1]) < 0.01:
                j += 1
        
        return transactions


class Transaction:
    def __init__(self, from_user: User, to_user: User, amount: float):
        self._from = from_user
        self._to = to_user
        self._amount = amount
    
    def __str__(self):
        return f"{self._from.get_name()} pays {self._to.get_name()} ${self._amount:.2f}"
```

---

## 🔄 Usage Example

```python
# Setup
service = SplitwiseService.get_instance()

alice = service.add_user("Alice", "alice@a.com")
bob = service.add_user("Bob", "bob@b.com")
charlie = service.add_user("Charlie", "charlie@c.com")

group = service.add_group("Friends Trip", [alice, bob, charlie])

# Equal Split: Alice pays $300 for dinner
service.create_expense(
    ExpenseBuilder()
    .set_description("Dinner")
    .set_amount(300)
    .set_paid_by(alice)
    .set_participants([alice, bob, charlie])
    .set_split_strategy(EqualSplitStrategy())
)

# Result: Bob owes Alice $100, Charlie owes Alice $100

# Exact Split: Bob pays $150 for movie
service.create_expense(
    ExpenseBuilder()
    .set_description("Movie")
    .set_amount(150)
    .set_paid_by(bob)
    .set_participants([alice, bob, charlie])
    .set_split_strategy(ExactSplitStrategy())
    .set_split_values([50, 50, 50])
)

# Simplify debts
transactions = service.simplify_group_debts(group.get_id())
for t in transactions:
    print(t)
```

---

## 🔄 Simplify Debts Algorithm

Before simplification:
```
A owes B: $50
B owes C: $50
```

After simplification:
```
A pays C: $50
```

**Algorithm:**
1. Calculate net balance for each person
2. Separate into creditors (+) and debtors (-)
3. Match debtors to creditors greedily
4. Minimize total transactions

---

## 🏛️ Class Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  SplitwiseService (Singleton)                                   │
│       ├── users: Dict[str, User]                                │
│       ├── groups: Dict[str, Group]                              │
│       ├── create_expense(builder)                               │
│       ├── settle_up(payer, payee, amount)                       │
│       └── simplify_group_debts(group_id)                        │
│                                                                 │
│  User                          Group                            │
│       ├── id, name, email           ├── id, name                │
│       └── balance_sheet             └── members: List[User]     │
│              │                                                  │
│              ▼                                                  │
│  BalanceSheet                                                   │
│       └── balances: Dict[User, float]                           │
│                                                                 │
│  Expense ◄── ExpenseBuilder                                     │
│       ├── amount, paid_by                                       │
│       ├── splits: List[Split]                                   │
│       └── uses SplitStrategy                                    │
│                                                                 │
│  SplitStrategy (Abstract)                                       │
│       ├── EqualSplitStrategy                                    │
│       ├── ExactSplitStrategy                                    │
│       └── PercentageSplitStrategy                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## ✅ Interview Takeaways

1. **Strategy Pattern** for different split algorithms
2. **Builder Pattern** for complex expense creation
3. **Singleton** for service layer
4. **Thread safety** with locks for concurrent access
5. **BalanceSheet per user** — clean separation
6. **Simplify debts** uses greedy matching algorithm
7. **Bidirectional balance updates** — if A owes B, update both sheets

---

*This article was created as part of LLD interview practice for Amazon SDE-2 (L5) level.*
