# Low-Level Design Interview: Notification System

> A complete walkthrough of an LLD interview demonstrating the **Observer Pattern** combined with **Strategy Pattern**, suitable for Amazon SDE-2 (L5) level interviews.

---

## 🎯 Problem Statement

**Interviewer:** Let's design a **Notification System** — like YouTube subscriptions or stock alerts. When an event occurs, multiple subscribers need to be notified through their preferred channels (Email, SMS, Push, etc.).

What clarifying questions do you have?

---

## 📋 Requirement Gathering

### Candidate's Questions

**Candidate:** I have several questions:

1. **Event Trigger**: Do I need to design how events are generated, or can I assume they come from an API/queue?

2. **Subscribers**: Users subscribe to topics and want notifications when events occur?

3. **Notification Channels**: Can be WhatsApp, Email, SMS, InApp, Push — different users may prefer different channels?

4. **User Preferences**:
   - Can users configure which channels they want?
   - Can they change preferences later?
   - Can they unsubscribe?

**Interviewer:** Great questions! Let me clarify:

| Question | Answer |
|----------|--------|
| **Event Trigger** | Assume events come in — focus on notification logic |
| **Subscribers** | Yes, users subscribe to topics |
| **Channels** | Multiple: Email, SMS, Push, WhatsApp |
| **Preferences** | ✅ Configure at subscription, ✅ Change anytime, ✅ Unsubscribe |

---

### Candidate's Design Insight

**Candidate:** I'll only show products that are:
1. I'll filter available products by affordability — this prevents invalid selections

I see this as classic **Observer Pattern** (pub-sub), and for multiple channels, I can use **Strategy Pattern** too!

**Interviewer:** Excellent! You've identified both patterns upfront. Let's proceed!

---

## 🔄 The Observer Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   PUBLISHER (Subject)              SUBSCRIBERS (Observers)      │
│   ───────────────────              ───────────────────────      │
│                                                                 │
│   Topic: "Sports"   ──────────────►  User1 (Email + Push)       │
│        │                                                        │
│        ├───────────────────────────►  User2 (SMS only)          │
│        │                                                        │
│        └───────────────────────────►  User3 (All channels)      │
│                                                                 │
│   When event occurs, ALL subscribers are notified               │
│   Each via THEIR preferred channels                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📝 Final Requirements

| Feature | In Scope |
|---------|----------|
| Subscribe to topics | ✅ |
| Configure notification channels | ✅ |
| Unsubscribe from topics | ✅ |
| Change channel preferences | ✅ |
| Notify all subscribers on event | ✅ |
| Multiple notification channels | ✅ |

---

## 🏗️ Core Entities

| Entity | Purpose | Pattern Role |
|--------|---------|--------------|
| **Topic** | What users subscribe to | Subject (Observable) |
| **User** | Person who subscribes | Observer |
| **NotificationChannel** | How to send (Email, SMS, etc.) | Strategy |
| **NotificationService** | Orchestrator (optional) | — |

---

## 💻 Class Implementations

### Topic (Subject)

```python
from typing import Set

class Topic:
    """
    Subject in Observer Pattern.
    Users subscribe to topics and get notified when events occur.
    """
    
    def __init__(self, name: str):
        self.name = name
        self.subscribers: Set[User] = set()
    
    def subscribe(self, user: 'User'):
        """Add a subscriber to this topic."""
        self.subscribers.add(user)
        print(f"✅ {user.name} subscribed to '{self.name}'")
    
    def unsubscribe(self, user: 'User'):
        """Remove a subscriber from this topic."""
        self.subscribers.discard(user)
        print(f"❌ {user.name} unsubscribed from '{self.name}'")
    
    def notify(self, message: str):
        """
        Notify all subscribers about an event.
        Each subscriber receives notification via their preferred channels.
        """
        print(f"\n📢 [{self.name}] Broadcasting: {message}")
        print("-" * 50)
        
        for subscriber in self.subscribers:
            subscriber.receive_notification(self.name, message)
        
        print("-" * 50 + "\n")
```

---

### NotificationChannel (Strategy)

```python
from abc import ABC, abstractmethod

class NotificationChannel(ABC):
    """
    Strategy interface for notification delivery.
    Each channel implements its own send logic.
    """
    
    @abstractmethod
    def send(self, user: 'User', topic: str, message: str):
        """Send notification to user via this channel."""
        pass


class EmailChannel(NotificationChannel):
    """Send notifications via Email."""
    
    def send(self, user, topic, message):
        if user.email:
            print(f"  📧 Email to {user.email}: [{topic}] {message}")
        else:
            print(f"  ⚠️ Email skipped: {user.name} has no email")


class SMSChannel(NotificationChannel):
    """Send notifications via SMS."""
    
    def send(self, user, topic, message):
        if user.phone:
            print(f"  📱 SMS to {user.phone}: [{topic}] {message}")
        else:
            print(f"  ⚠️ SMS skipped: {user.name} has no phone")


class PushChannel(NotificationChannel):
    """Send push notifications to mobile app."""
    
    def send(self, user, topic, message):
        if user.device_id:
            print(f"  🔔 Push to {user.device_id}: [{topic}] {message}")
        else:
            print(f"  ⚠️ Push skipped: {user.name} has no device")


class WhatsAppChannel(NotificationChannel):
    """Send notifications via WhatsApp."""
    
    def send(self, user, topic, message):
        if user.phone:
            print(f"  💬 WhatsApp to {user.phone}: [{topic}] {message}")
        else:
            print(f"  ⚠️ WhatsApp skipped: {user.name} has no phone")
```

---

### User (Observer)

```python
from typing import List

class User:
    """
    Observer in Observer Pattern.
    Each user has preferred notification channels.
    """
    
    def __init__(
        self, 
        user_id: str, 
        name: str, 
        email: str = None, 
        phone: str = None,
        device_id: str = None
    ):
        self.user_id = user_id
        self.name = name
        self.email = email
        self.phone = phone
        self.device_id = device_id
        self.channels: List[NotificationChannel] = []
    
    def set_preferences(self, channels: List[NotificationChannel]):
        """Set notification channel preferences."""
        self.channels = channels
        channel_names = [c.__class__.__name__ for c in channels]
        print(f"⚙️ {self.name} preferences: {channel_names}")
    
    def add_channel(self, channel: NotificationChannel):
        """Add a notification channel."""
        self.channels.append(channel)
    
    def remove_channel(self, channel_type: type):
        """Remove a notification channel by type."""
        self.channels = [c for c in self.channels if not isinstance(c, channel_type)]
    
    def receive_notification(self, topic: str, message: str):
        """
        Called by Topic when an event occurs.
        Sends notification via all preferred channels.
        """
        if not self.channels:
            print(f"  ⚠️ {self.name} has no channels configured")
            return
        
        for channel in self.channels:
            channel.send(self, topic, message)
    
    def __hash__(self):
        return hash(self.user_id)
    
    def __eq__(self, other):
        return isinstance(other, User) and self.user_id == other.user_id
```

---

### NotificationService (Orchestrator)

```python
class NotificationService:
    """
    Orchestrator that manages topics and provides a simple API.
    Could be a Singleton in production.
    """
    
    def __init__(self):
        self.topics: Dict[str, Topic] = {}
    
    def create_topic(self, name: str) -> Topic:
        """Create a new topic."""
        if name not in self.topics:
            self.topics[name] = Topic(name)
        return self.topics[name]
    
    def get_topic(self, name: str) -> Topic:
        """Get existing topic."""
        return self.topics.get(name)
    
    def subscribe_user(self, user: User, topic_name: str):
        """Subscribe user to a topic."""
        topic = self.get_topic(topic_name)
        if topic:
            topic.subscribe(user)
    
    def publish(self, topic_name: str, message: str):
        """Publish event to a topic."""
        topic = self.get_topic(topic_name)
        if topic:
            topic.notify(message)
```

---

## 🔄 Complete Flow Walkthrough

### Setup and Usage

```python
# Create notification service
service = NotificationService()

# Create topics
sports = service.create_topic("Sports")
tech = service.create_topic("Tech")
stocks = service.create_topic("StockAlerts")

# Create users
user1 = User("u1", "Vinay", email="vinay@email.com", phone="9999999999")
user2 = User("u2", "Raj", email="raj@email.com", device_id="device_123")
user3 = User("u3", "Priya", phone="8888888888")

# Set preferences
user1.set_preferences([EmailChannel(), SMSChannel()])
user2.set_preferences([EmailChannel(), PushChannel()])
user3.set_preferences([WhatsAppChannel()])

# Subscribe to topics
sports.subscribe(user1)
sports.subscribe(user2)
tech.subscribe(user2)
stocks.subscribe(user3)

# Events occur!
service.publish("Sports", "🏏 India wins the World Cup!")
service.publish("Stocks", "📈 AAPL up 5%!")
```

### Output

```
⚙️ Vinay preferences: ['EmailChannel', 'SMSChannel']
⚙️ Raj preferences: ['EmailChannel', 'PushChannel']
⚙️ Priya preferences: ['WhatsAppChannel']
✅ Vinay subscribed to 'Sports'
✅ Raj subscribed to 'Sports'
✅ Raj subscribed to 'Tech'
✅ Priya subscribed to 'StockAlerts'

📢 [Sports] Broadcasting: 🏏 India wins the World Cup!
--------------------------------------------------
  📧 Email to vinay@email.com: [Sports] 🏏 India wins the World Cup!
  📱 SMS to 9999999999: [Sports] 🏏 India wins the World Cup!
  📧 Email to raj@email.com: [Sports] 🏏 India wins the World Cup!
  🔔 Push to device_123: [Sports] 🏏 India wins the World Cup!
--------------------------------------------------

📢 [StockAlerts] Broadcasting: 📈 AAPL up 5%!
--------------------------------------------------
  💬 WhatsApp to 8888888888: [StockAlerts] 📈 AAPL up 5%!
--------------------------------------------------
```

---

## 🏛️ Class Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  NotificationService (Orchestrator)                                 │
│       │                                                             │
│       └── topics: Dict[str, Topic]                                  │
│                       │                                             │
│                       ▼                                             │
│  Topic (Subject)                                                    │
│       │                                                             │
│       ├── name: str                                                 │
│       ├── subscribers: Set[User]                                    │
│       ├── subscribe(user)                                           │
│       ├── unsubscribe(user)                                         │
│       └── notify(message)                                           │
│                   │                                                 │
│                   ▼                                                 │
│  User (Observer)                                                    │
│       │                                                             │
│       ├── user_id, name, email, phone                               │
│       ├── channels: List[NotificationChannel]                       │
│       ├── set_preferences(channels)                                 │
│       └── receive_notification(topic, message)                      │
│                   │                                                 │
│                   ▼                                                 │
│  NotificationChannel (Strategy - Abstract)                          │
│       │                                                             │
│       ├── EmailChannel                                              │
│       ├── SMSChannel                                                │
│       ├── PushChannel                                               │
│       └── WhatsAppChannel                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🎨 Design Patterns Used

| Pattern | Where | Why |
|---------|-------|-----|
| **Observer** | Topic → User | Notify multiple subscribers when event occurs |
| **Strategy** | NotificationChannel variants | Different send mechanisms, same interface |

### Why Two Patterns?

- **Observer** handles the **who** — which users to notify
- **Strategy** handles the **how** — which channels to use

They complement each other beautifully!

---

## 🧪 Edge Cases Handled

| Edge Case | How Handled |
|-----------|-------------|
| User has no channels | Warning message, skip notification |
| User missing email/phone | Channel checks and skips gracefully |
| User subscribes twice | Set prevents duplicates |
| Unsubscribe non-existent user | `discard()` handles silently |
| Topic has no subscribers | Loop doesn't execute |

---

## 🚀 Production Enhancements

In a real system, consider:

| Enhancement | Why |
|-------------|-----|
| **Async sending** | Don't block main thread |
| **Retry mechanism** | Handle transient failures |
| **Notification queue** | Buffer for high volume |
| **Notification object** | Track status, timestamp, audit |
| **Rate limiting** | Prevent spam |
| **Batching** | Group notifications for efficiency |

---

## 🆚 Observer vs Pub-Sub

| Aspect | Observer Pattern | Pub-Sub |
|--------|-----------------|---------|
| **Coupling** | Subject knows observers | Decoupled via broker |
| **Complexity** | Simpler | More infrastructure |
| **Use Case** | In-process events | Distributed systems |
| **Example** | This implementation | Kafka, RabbitMQ |

For LLD interviews, Observer is typically sufficient. Mention Pub-Sub for system design!

---

## ✅ Interview Takeaways

1. **Observer Pattern** = Subject notifies all Observers
2. **Strategy Pattern** can complement Observer for delivery mechanisms
3. **Set for subscribers** prevents duplicates
4. Each Observer decides **how** to process the notification
5. Keep Observer interface simple (`receive_notification`)
6. Allow dynamic preference changes

---

*This article was created as part of LLD interview practice for Amazon SDE-2 (L5) level.*
