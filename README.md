# 📚 Low-Level Design (LLD) Interview Preparation

> A comprehensive collection of **14 LLD problems** with complete implementations, design patterns, and interview tips — prepared for **Amazon SDE-2 (L5)** level interviews.

---

## 🎯 Quick Reference: Design Patterns

| Pattern | When to Use | Problems |
|---------|-------------|----------|
| **Strategy** | Multiple algorithms, interchangeable | Parking, Splitwise, Stock, Locker, Food Delivery |
| **State** | Object behavior changes with state | Vending Machine, Locker, Food Delivery |
| **Observer** | Notify multiple subscribers | Notification, Stock Trading, Logging, Food Delivery |
| **Decorator** | Add features dynamically | Pizza Ordering |
| **Command** | Undo/redo operations | Text Editor |
| **Chain of Responsibility** | Pass request through handlers | Coupon System, Logging |
| **Singleton** | Single instance | Most orchestrators |
| **Builder** | Complex object construction | Splitwise |
| **Factory** | Create objects without specifying class | Various |

---

## 📁 Problem Index

### 1. [Parking Lot](parking_lot_lld_interview.md)
**Patterns:** Strategy, Singleton  
**Key Concepts:** Vehicle hierarchy, floor management, pricing strategies

### 2. [Vending Machine](vending_machine_lld_interview.md)
**Patterns:** State  
**Key Concepts:** State machine, item dispensing, payment handling

### 3. [Notification System](notification_system_lld_interview.md)
**Patterns:** Observer, Strategy  
**Key Concepts:** Topic-based pub/sub, multiple channels (Email, SMS, Push)

### 4. [Pizza Ordering](pizza_ordering_lld_interview.md)
**Patterns:** Decorator  
**Key Concepts:** Dynamic toppings, price calculation, customization

### 5. [Text Editor](text_editor_lld_interview.md)
**Patterns:** Command  
**Key Concepts:** Undo/redo, command history, text operations

### 6. [Splitwise](splitwise_lld_interview.md)
**Patterns:** Strategy, Builder, Singleton  
**Key Concepts:** Expense splitting (equal, exact, percentage), debt simplification

### 7. [Coupon System](coupon_system_lld.md)
**Patterns:** Strategy, Chain of Responsibility  
**Key Concepts:** Multiple discount types, validation rules, stacking

### 8. [Rate Limiter](rate_limiter_lld.md)
**Patterns:** Strategy  
**Key Concepts:** 5 algorithms (Fixed Window, Sliding Log, Sliding Counter, Token Bucket, Leaky Bucket)

### 9. [Car Rental](car_rental_system_lld.md)
**Patterns:** Strategy  
**Key Concepts:** Booking flow, availability check, per-user locking, late fees

### 10. [Stock Trading](stock_trading_platform_lld.md)
**Patterns:** Strategy, Observer  
**Key Concepts:** Market/Limit orders, price subscription, portfolio management

### 11. [Movie Booking](movie_booking_system_lld.md)
**Patterns:** State  
**Key Concepts:** ⭐ Redis SETNX for seat locking, TTL for timeout, concurrent booking prevention

### 12. [Amazon Locker](amazon_locker_system_lld.md)
**Patterns:** Strategy, State  
**Key Concepts:** Locker assignment, code generation, expiry handling

### 13. [Logging Framework](logging_framework_lld.md)
**Patterns:** Chain of Responsibility, Observer, Strategy  
**Key Concepts:** Log levels, multiple handlers, formatters, both CoR and Observer approaches

### 14. [Food Delivery](food_delivery_system_lld.md)
**Patterns:** Strategy, State, Observer  
**Key Concepts:** Order flow, delivery partner assignment, real-time tracking

---

## 🔥 Most Asked Interview Questions

| Problem | Famous Question |
|---------|-----------------|
| **Movie Booking** | How to prevent double booking of seats? |
| **Rate Limiter** | Token Bucket vs Leaky Bucket? |
| **Parking Lot** | How to find nearest available spot? |
| **Stock Trading** | How to handle LIMIT orders? |
| **Food Delivery** | How to assign delivery partner? |
| **Logging** | Chain of Responsibility vs Observer? |

---

## 📋 Interview Structure (45 mins)

```
1. Clarifying Questions        (5 min)
2. Requirements (F & NF)       (5 min)
3. Use Cases / User Flow       (5 min)
4. API Contracts               (5 min)
5. Core Entities               (5 min)
6. Database Design             (5 min)
7. Class Diagram               (5 min)
8. Code Implementation         (10 min)
```

---

## ✅ Checklist Before Interview

- [ ] Know all 9 design patterns and when to use each
- [ ] Practice explaining trade-offs (e.g., CoR vs Observer)
- [ ] Remember to mention thread safety and locking
- [ ] Always define API contracts with request/response
- [ ] Discuss database schema for persistence
- [ ] Mention edge cases proactively

---

## 🎯 Pattern Quick Lookup

### Need to add behaviors dynamically?
→ **Decorator** (Pizza toppings)

### Need undo/redo?
→ **Command** (Text Editor)

### Object changes behavior based on state?
→ **State** (Vending Machine, Order Status)

### Multiple interchangeable algorithms?
→ **Strategy** (Pricing, Assignment, Splitting)

### Notify multiple subscribers?
→ **Observer** (Notifications, Tracking)

### Pass through chain of handlers?
→ **Chain of Responsibility** (Log levels, Coupon validation)

---

## 📝 Notes

- All examples use **Python** for clarity
- Each article includes **complete working code**
- Database schemas provided for persistence
- API contracts with request/response JSON
- Thread safety considerations included

---

*Created during LLD interview preparation sessions — Jan 2026*
