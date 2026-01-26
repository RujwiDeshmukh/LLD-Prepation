# Low-Level Design: Rate Limiter

> Complete guide covering all rate limiting algorithms, implementations, and API integration patterns.

---

## 🎯 What is Rate Limiting?

> Control how many requests a client can make in a given time window.

**Use Cases:**
- Prevent API abuse
- Protect against DDoS attacks
- Fair usage among users
- Cost control (paid APIs)
- Prevent brute force attacks

---

## 📊 Rate Limiting Algorithms

| Algorithm | Memory | Accuracy | Burst Handling |
|-----------|--------|----------|----------------|
| Fixed Window | O(1) | Low | ❌ Edge burst |
| Sliding Window Log | O(n) | High | ✅ |
| Sliding Window Counter | O(1) | Medium | ✅ |
| **Token Bucket** | O(1) | High | ✅ Controlled |
| Leaky Bucket | O(n) | High | ❌ No burst |

---

## 💻 Algorithm Implementations

### 1. Fixed Window Counter

**Concept:** Divide time into fixed windows (e.g., 1-minute), count requests per window.

```python
import time

class FixedWindowRateLimiter:
    """
    Simple rate limiting using fixed time windows.
    Example: 100 requests per minute = window_size=60, max_requests=100
    """
    
    def __init__(self, max_requests: int, window_size_seconds: int):
        self.max_requests = max_requests
        self.window_size = window_size_seconds
        self.requests = {}  # user_id -> (window_start, count)
    
    def allow_request(self, user_id: str) -> bool:
        current_time = time.time()
        window_start = int(current_time // self.window_size) * self.window_size
        
        if user_id not in self.requests:
            self.requests[user_id] = (window_start, 1)
            return True
        
        user_window, count = self.requests[user_id]
        
        if user_window != window_start:
            # New window, reset
            self.requests[user_id] = (window_start, 1)
            return True
        
        if count >= self.max_requests:
            return False
        
        self.requests[user_id] = (window_start, count + 1)
        return True
```

| Pros | Cons |
|------|------|
| ✅ Very simple | ❌ Edge burst problem |
| ✅ O(1) memory per user | ❌ Not smooth |
| ✅ Fast | |

**Edge Burst Problem:**
```
Window 1 (00:00-00:59)    Window 2 (01:00-01:59)
              |---100 reqs---|
                00:59 - 01:01 = 200 requests in 2 seconds!
```

---

### 2. Sliding Window Log

**Concept:** Store timestamp of every request, count requests in last N seconds.

```python
import time
from collections import deque

class SlidingWindowLogRateLimiter:
    """
    Accurate rate limiting by tracking all request timestamps.
    Higher memory usage but no edge burst problem.
    """
    
    def __init__(self, max_requests: int, window_size_seconds: int):
        self.max_requests = max_requests
        self.window_size = window_size_seconds
        self.requests = {}  # user_id -> deque of timestamps
    
    def allow_request(self, user_id: str) -> bool:
        current_time = time.time()
        cutoff_time = current_time - self.window_size
        
        if user_id not in self.requests:
            self.requests[user_id] = deque()
        
        user_requests = self.requests[user_id]
        
        # Remove expired timestamps
        while user_requests and user_requests[0] < cutoff_time:
            user_requests.popleft()
        
        if len(user_requests) >= self.max_requests:
            return False
        
        user_requests.append(current_time)
        return True
```

| Pros | Cons |
|------|------|
| ✅ Very accurate | ❌ High memory O(n) |
| ✅ No edge burst | ❌ O(n) cleanup |

---

### 3. Sliding Window Counter

**Concept:** Hybrid of fixed and sliding window. Uses weighted average.

```python
import time

class SlidingWindowCounterRateLimiter:
    """
    Balances accuracy and memory by using weighted counts.
    Approximates sliding window without storing all timestamps.
    """
    
    def __init__(self, max_requests: int, window_size_seconds: int):
        self.max_requests = max_requests
        self.window_size = window_size_seconds
        self.requests = {}  # user_id -> {prev_count, curr_count, curr_window}
    
    def allow_request(self, user_id: str) -> bool:
        current_time = time.time()
        current_window = int(current_time // self.window_size)
        window_progress = (current_time % self.window_size) / self.window_size
        
        if user_id not in self.requests:
            self.requests[user_id] = {
                'prev_count': 0,
                'curr_count': 0,
                'curr_window': current_window
            }
        
        data = self.requests[user_id]
        
        # Slide windows if needed
        if data['curr_window'] < current_window:
            data['prev_count'] = data['curr_count']
            data['curr_count'] = 0
            data['curr_window'] = current_window
        
        # Weighted count: prev_weight decreases as we move through current window
        prev_weight = 1 - window_progress
        weighted_count = data['prev_count'] * prev_weight + data['curr_count']
        
        if weighted_count >= self.max_requests:
            return False
        
        data['curr_count'] += 1
        return True
```

| Pros | Cons |
|------|------|
| ✅ Low memory O(1) | ❌ Approximate |
| ✅ Smooth limiting | ❌ Slightly complex |
| ✅ No edge burst | |

---

### 4. Token Bucket ⭐ (Most Popular)

**Concept:** Bucket holds tokens. Tokens added at constant rate. Each request consumes a token.

```python
import time
import threading

class TokenBucketRateLimiter:
    """
    Industry standard rate limiter used by AWS, Stripe, etc.
    Allows controlled bursts while maintaining average rate.
    
    capacity: Maximum tokens in bucket (burst size)
    refill_rate: Tokens added per second (sustained rate)
    """
    
    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.buckets = {}  # user_id -> {'tokens': float, 'last_refill': float}
        self.lock = threading.Lock()
    
    def allow_request(self, user_id: str, tokens_needed: int = 1) -> bool:
        with self.lock:
            current_time = time.time()
            
            if user_id not in self.buckets:
                self.buckets[user_id] = {
                    'tokens': self.capacity,
                    'last_refill': current_time
                }
            
            bucket = self.buckets[user_id]
            
            # Refill tokens based on time passed
            time_passed = current_time - bucket['last_refill']
            tokens_to_add = time_passed * self.refill_rate
            bucket['tokens'] = min(self.capacity, bucket['tokens'] + tokens_to_add)
            bucket['last_refill'] = current_time
            
            if bucket['tokens'] < tokens_needed:
                return False
            
            bucket['tokens'] -= tokens_needed
            return True
    
    def get_remaining_tokens(self, user_id: str) -> int:
        if user_id in self.buckets:
            return int(self.buckets[user_id]['tokens'])
        return self.capacity
```

**Visual:**
```
Bucket (capacity=10, refill_rate=2/sec)
━━━━━━━━━━━━━━━━━━━━━━━━
│ 🪙🪙🪙🪙🪙🪙🪙🪙🪙🪙 │  Full = 10 tokens
━━━━━━━━━━━━━━━━━━━━━━━━

Request comes → consumes 1 token
━━━━━━━━━━━━━━━━━━━━━━━━
│ 🪙🪙🪙🪙🪙🪙🪙🪙🪙   │  9 tokens
━━━━━━━━━━━━━━━━━━━━━━━━

After 1 second → 2 tokens added
━━━━━━━━━━━━━━━━━━━━━━━━
│ 🪙🪙🪙🪙🪙🪙🪙🪙🪙🪙 │  Full again
━━━━━━━━━━━━━━━━━━━━━━━━
```

| Pros | Cons |
|------|------|
| ✅ Allows controlled bursts | ❌ Slightly complex |
| ✅ Smooth long-term rate | |
| ✅ Variable cost per request | |
| ✅ Industry standard | |

---

### 5. Leaky Bucket

**Concept:** Requests enter a queue, processed at constant rate. Queue overflow = rejected.

```python
import time
from collections import deque

class LeakyBucketRateLimiter:
    """
    Requests are processed at a constant rate.
    Good for smoothing bursty traffic into steady stream.
    
    capacity: Maximum queue size
    leak_rate: Requests processed per second
    """
    
    def __init__(self, capacity: int, leak_rate: float):
        self.capacity = capacity
        self.leak_rate = leak_rate
        self.buckets = {}  # user_id -> {'queue': deque, 'last_leak': float}
    
    def allow_request(self, user_id: str) -> bool:
        current_time = time.time()
        
        if user_id not in self.buckets:
            self.buckets[user_id] = {
                'queue': deque(),
                'last_leak': current_time
            }
        
        bucket = self.buckets[user_id]
        
        # Leak (process) requests
        time_passed = current_time - bucket['last_leak']
        requests_to_leak = int(time_passed * self.leak_rate)
        
        for _ in range(min(requests_to_leak, len(bucket['queue']))):
            bucket['queue'].popleft()
        
        bucket['last_leak'] = current_time
        
        # Try to add new request
        if len(bucket['queue']) >= self.capacity:
            return False
        
        bucket['queue'].append(current_time)
        return True
```

| Pros | Cons |
|------|------|
| ✅ Constant output rate | ❌ No bursts allowed |
| ✅ Smooths traffic | ❌ May add latency |

---

## 🎨 Strategy Pattern for Flexibility

```python
from abc import ABC, abstractmethod

class RateLimiter(ABC):
    @abstractmethod
    def allow_request(self, user_id: str) -> bool:
        pass
    
    @abstractmethod
    def get_wait_time(self, user_id: str) -> float:
        pass

class RateLimiterService:
    def __init__(self, strategy: RateLimiter):
        self.strategy = strategy
    
    def check_rate_limit(self, user_id: str) -> dict:
        allowed = self.strategy.allow_request(user_id)
        return {
            'allowed': allowed,
            'wait_time': 0 if allowed else self.strategy.get_wait_time(user_id)
        }

# Easily swap algorithms
service = RateLimiterService(TokenBucketRateLimiter(100, 10))
# service = RateLimiterService(SlidingWindowCounterRateLimiter(100, 60))
```

---

## 🔌 API Integration

### Middleware Approach (Flask)

```python
from flask import Flask, request, jsonify, g
from functools import wraps

app = Flask(__name__)
limiter = TokenBucketRateLimiter(capacity=100, refill_rate=10)

def get_user_id():
    """Identify user by API key, session, or IP."""
    if 'X-API-Key' in request.headers:
        return f"api:{request.headers['X-API-Key']}"
    return f"ip:{request.remote_addr}"

@app.before_request
def check_rate_limit():
    user_id = get_user_id()
    
    if not limiter.allow_request(user_id):
        return jsonify({
            "error": "Rate limit exceeded",
            "message": "Too many requests. Please slow down.",
            "retry_after": 60
        }), 429

@app.after_request
def add_rate_limit_headers(response):
    user_id = get_user_id()
    remaining = limiter.get_remaining_tokens(user_id)
    
    response.headers['X-RateLimit-Limit'] = '100'
    response.headers['X-RateLimit-Remaining'] = str(remaining)
    return response
```

---

### Per-Endpoint Limits

```python
class EndpointRateLimiter:
    """Different rate limits for different endpoints."""
    
    LIMITS = {
        '/api/login': (5, 1),        # 5 requests/sec (prevent brute force)
        '/api/search': (100, 50),    # 100 requests, 50/sec
        '/api/upload': (10, 1),      # 10 requests/sec (heavy operation)
        'default': (60, 30)          # 60 requests, 30/sec
    }
    
    def __init__(self):
        self.limiters = {}
    
    def allow_request(self, user_id: str, endpoint: str) -> bool:
        key = (user_id, endpoint)
        
        if key not in self.limiters:
            capacity, rate = self.LIMITS.get(endpoint, self.LIMITS['default'])
            self.limiters[key] = TokenBucketRateLimiter(capacity, rate)
        
        return self.limiters[key].allow_request(user_id)
```

---

### Tiered Rate Limits (Free vs Premium)

```python
from enum import Enum

class UserTier(Enum):
    FREE = "free"
    BASIC = "basic"
    PREMIUM = "premium"
    ENTERPRISE = "enterprise"

class TieredRateLimiter:
    TIER_LIMITS = {
        UserTier.FREE:       {'capacity': 100,    'rate': 10},
        UserTier.BASIC:      {'capacity': 1000,   'rate': 100},
        UserTier.PREMIUM:    {'capacity': 10000,  'rate': 500},
        UserTier.ENTERPRISE: {'capacity': 100000, 'rate': 5000}
    }
    
    def __init__(self):
        self.limiters = {}
    
    def allow_request(self, user_id: str, tier: UserTier) -> bool:
        if user_id not in self.limiters:
            config = self.TIER_LIMITS[tier]
            self.limiters[user_id] = TokenBucketRateLimiter(
                config['capacity'], config['rate']
            )
        return self.limiters[user_id].allow_request(user_id)
```

---

### Distributed Rate Limiting (Redis)

For multi-server deployments:

```python
import redis
import time

class RedisTokenBucketRateLimiter:
    """
    Distributed rate limiter using Redis.
    Uses Lua script for atomic operations.
    """
    
    def __init__(self, redis_client: redis.Redis, capacity: int, refill_rate: float):
        self.redis = redis_client
        self.capacity = capacity
        self.refill_rate = refill_rate
        
        # Lua script for atomic token bucket
        self.lua_script = """
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local current_time = tonumber(ARGV[3])
        local tokens_needed = tonumber(ARGV[4])
        
        local data = redis.call('HMGET', key, 'tokens', 'last_refill')
        local tokens = tonumber(data[1]) or capacity
        local last_refill = tonumber(data[2]) or current_time
        
        -- Refill tokens
        local time_passed = current_time - last_refill
        local tokens_to_add = time_passed * refill_rate
        tokens = math.min(capacity, tokens + tokens_to_add)
        
        -- Check if enough tokens
        if tokens < tokens_needed then
            redis.call('HMSET', key, 'tokens', tokens, 'last_refill', current_time)
            redis.call('EXPIRE', key, 3600)
            return {0, tokens}
        end
        
        -- Consume tokens
        tokens = tokens - tokens_needed
        redis.call('HMSET', key, 'tokens', tokens, 'last_refill', current_time)
        redis.call('EXPIRE', key, 3600)
        return {1, tokens}
        """
    
    def allow_request(self, user_id: str, tokens_needed: int = 1) -> tuple:
        key = f"rate_limit:{user_id}"
        current_time = time.time()
        
        result = self.redis.eval(
            self.lua_script, 1, key,
            self.capacity, self.refill_rate, current_time, tokens_needed
        )
        
        allowed = result[0] == 1
        remaining = int(result[1])
        return allowed, remaining
```

---

## 📋 HTTP Response Headers

Always inform clients about rate limits:

```python
class RateLimitResponse:
    @staticmethod
    def success(data, remaining: int, limit: int, reset_time: int):
        response = jsonify(data)
        response.headers['X-RateLimit-Limit'] = str(limit)
        response.headers['X-RateLimit-Remaining'] = str(remaining)
        response.headers['X-RateLimit-Reset'] = str(reset_time)
        return response
    
    @staticmethod
    def rate_limited(retry_after: int):
        response = jsonify({
            "error": "Too Many Requests",
            "message": "Rate limit exceeded. Please slow down.",
            "retry_after": retry_after
        })
        response.status_code = 429
        response.headers['Retry-After'] = str(retry_after)
        return response
```

---

## 🏛️ Class Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  RateLimiter (Abstract)                                         │
│       ├── allow_request(user_id) -> bool                        │
│       └── get_wait_time(user_id) -> float                       │
│            ▲                                                    │
│            │                                                    │
│   ┌────────┼────────┬──────────────┬──────────────┐             │
│   │        │        │              │              │             │
│   Fixed   Sliding  Sliding     Token         Leaky             │
│   Window  Log      Counter     Bucket        Bucket            │
│                                                                 │
│  RateLimiterMiddleware                                          │
│       ├── limiter: RateLimiter                                  │
│       ├── get_user_id() -> str                                  │
│       └── check_rate_limit()                                    │
│                                                                 │
│  TieredRateLimiter                                              │
│       └── TIER_LIMITS: Dict[UserTier, config]                   │
│                                                                 │
│  RedisTokenBucketRateLimiter                                    │
│       └── Distributed rate limiting with Redis                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🎯 Algorithm Selection Guide

| Scenario | Best Algorithm | Why |
|----------|----------------|-----|
| Simple, low traffic | Fixed Window | Easy to implement |
| Need accuracy, have memory | Sliding Log | Most accurate |
| Production API | **Token Bucket** | Burst + steady rate |
| Smooth output needed | Leaky Bucket | Constant rate |
| Balance of all | Sliding Counter | Low memory, good accuracy |
| Multi-server | Redis + Token Bucket | Distributed state |

---

## ✅ Interview Recommendations

1. **Start with requirements:**
   - Requests per second/minute?
   - Per user or global?
   - Single server or distributed?

2. **Discuss trade-offs:**
   - Memory vs accuracy
   - Burst handling
   - Implementation complexity

3. **Implement Token Bucket** — most practical choice

4. **Mention production concerns:**
   - Redis for distributed systems
   - Response headers for client awareness
   - Graceful degradation

---

*This article was created as part of LLD learning for system design interviews.*
