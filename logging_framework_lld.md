# Low-Level Design Interview: Logging Framework

> Comprehensive guide covering **TWO approaches** — Chain of Responsibility vs Observer pattern.

---

## 🎯 Problem Statement

**Interviewer:** Design a **Logging Framework** like Log4j, SLF4J, or Python's logging module where developers can log messages at different levels and send them to various destinations.

---

## 📋 Requirements

### Functional

| Feature | Description |
|---------|-------------|
| Log levels | DEBUG, INFO, WARN, ERROR, FATAL |
| Multiple handlers | Console, File, Database |
| Level filtering | ERROR handler ignores DEBUG |
| Multiple formatters | Simple text, JSON |
| Configurable | Add/remove handlers at runtime |

### Non-Functional

| Requirement | Description |
|-------------|-------------|
| Thread-safe | Multiple threads logging simultaneously |
| Singleton | One logger per name |
| Extensible | Easy to add new handlers/formatters |

---

## 🔄 Log Level Priority

```
DEBUG (1) < INFO (2) < WARN (3) < ERROR (4) < FATAL (5)

If handler.level = WARN:
    - Handles: WARN, ERROR, FATAL
    - Ignores: DEBUG, INFO
```

---

## 🏗️ Common Entities (Both Approaches)

```python
from enum import Enum
from datetime import datetime
from abc import ABC, abstractmethod
import threading
import json

# ─────────────────────────────────────────────────────────────────
# Log Level Enum
# ─────────────────────────────────────────────────────────────────
class LogLevel(Enum):
    DEBUG = 1
    INFO = 2
    WARN = 3
    ERROR = 4
    FATAL = 5

# ─────────────────────────────────────────────────────────────────
# Log Message
# ─────────────────────────────────────────────────────────────────
class LogMessage:
    def __init__(self, level: LogLevel, message: str, logger_name: str = "root"):
        self.level = level
        self.message = message
        self.logger_name = logger_name
        self.timestamp = datetime.now()
        self.thread_name = threading.current_thread().name

# ─────────────────────────────────────────────────────────────────
# Formatter (Strategy Pattern) — Same for both approaches
# ─────────────────────────────────────────────────────────────────
class Formatter(ABC):
    @abstractmethod
    def format(self, msg: LogMessage) -> str:
        pass

class SimpleFormatter(Formatter):
    def format(self, msg: LogMessage) -> str:
        return f"[{msg.timestamp}] [{msg.level.name}] [{msg.logger_name}] - {msg.message}"

class JSONFormatter(Formatter):
    def format(self, msg: LogMessage) -> str:
        return json.dumps({
            "timestamp": str(msg.timestamp),
            "level": msg.level.name,
            "logger": msg.logger_name,
            "message": msg.message,
            "thread": msg.thread_name
        })

class ColoredFormatter(Formatter):
    COLORS = {
        LogLevel.DEBUG: "\033[36m",   # Cyan
        LogLevel.INFO: "\033[32m",    # Green
        LogLevel.WARN: "\033[33m",    # Yellow
        LogLevel.ERROR: "\033[31m",   # Red
        LogLevel.FATAL: "\033[35m",   # Magenta
    }
    RESET = "\033[0m"
    
    def format(self, msg: LogMessage) -> str:
        color = self.COLORS.get(msg.level, "")
        return f"{color}[{msg.level.name}]{self.RESET} {msg.message}"
```

---

# Approach 1: Chain of Responsibility

## Concept

Each handler is a **node in a chain**. Message flows through the entire chain, and each handler decides whether to process based on log level.

```
Message → DEBUG Handler → INFO Handler → ERROR Handler → FATAL Handler
              ↓              ↓               ↓              ↓
           Console         File          Database        Slack
```

## Complete Implementation

```python
# ─────────────────────────────────────────────────────────────────
# Chain of Responsibility Handler
# ─────────────────────────────────────────────────────────────────
class ChainHandler(ABC):
    def __init__(self, level: LogLevel, formatter: Formatter):
        self.level = level
        self.formatter = formatter
        self.next_handler: 'ChainHandler' = None
        self.lock = threading.Lock()
    
    def set_next(self, handler: 'ChainHandler') -> 'ChainHandler':
        """Chain handlers together."""
        self.next_handler = handler
        return handler  # For fluent chaining
    
    def handle(self, msg: LogMessage):
        """Process message if level matches, then pass to next."""
        # Check if this handler should process
        if msg.level.value >= self.level.value:
            formatted = self.formatter.format(msg)
            self.write(formatted)
        
        # Pass to next handler in chain
        if self.next_handler:
            self.next_handler.handle(msg)
    
    @abstractmethod
    def write(self, formatted_message: str):
        pass


class ConsoleChainHandler(ChainHandler):
    """Writes logs to console/stdout."""
    
    def write(self, msg: str):
        with self.lock:
            print(msg)


class FileChainHandler(ChainHandler):
    """Writes logs to a file."""
    
    def __init__(self, level: LogLevel, formatter: Formatter, file_path: str):
        super().__init__(level, formatter)
        self.file_path = file_path
    
    def write(self, msg: str):
        with self.lock:
            with open(self.file_path, 'a') as f:
                f.write(msg + '\n')


class DatabaseChainHandler(ChainHandler):
    """Writes logs to database."""
    
    def __init__(self, level: LogLevel, formatter: Formatter, db_connection):
        super().__init__(level, formatter)
        self.db = db_connection
    
    def write(self, msg: str):
        with self.lock:
            # self.db.execute("INSERT INTO logs VALUES (?)", msg)
            print(f"[DB INSERT] {msg}")


class SlackChainHandler(ChainHandler):
    """Sends critical logs to Slack."""
    
    def __init__(self, level: LogLevel, formatter: Formatter, webhook_url: str):
        super().__init__(level, formatter)
        self.webhook_url = webhook_url
    
    def write(self, msg: str):
        # requests.post(self.webhook_url, json={"text": msg})
        print(f"[SLACK ALERT] {msg}")


# ─────────────────────────────────────────────────────────────────
# Logger with Chain of Responsibility
# ─────────────────────────────────────────────────────────────────
class ChainLogger:
    _instances = {}
    _lock = threading.Lock()
    
    def __init__(self, name: str):
        self.name = name
        self.min_level = LogLevel.DEBUG
        self.chain_head: ChainHandler = None
    
    @classmethod
    def get_logger(cls, name: str = "root") -> 'ChainLogger':
        with cls._lock:
            if name not in cls._instances:
                cls._instances[name] = ChainLogger(name)
            return cls._instances[name]
    
    def set_chain(self, head: ChainHandler):
        """Set the head of the handler chain."""
        self.chain_head = head
    
    def set_level(self, level: LogLevel):
        """Set minimum log level for this logger."""
        self.min_level = level
    
    def log(self, level: LogLevel, message: str):
        if level.value < self.min_level.value:
            return  # Ignore messages below threshold
        
        msg = LogMessage(level, message, self.name)
        
        if self.chain_head:
            self.chain_head.handle(msg)
    
    def debug(self, message: str):
        self.log(LogLevel.DEBUG, message)
    
    def info(self, message: str):
        self.log(LogLevel.INFO, message)
    
    def warn(self, message: str):
        self.log(LogLevel.WARN, message)
    
    def error(self, message: str):
        self.log(LogLevel.ERROR, message)
    
    def fatal(self, message: str):
        self.log(LogLevel.FATAL, message)


# ─────────────────────────────────────────────────────────────────
# Usage: Chain of Responsibility
# ─────────────────────────────────────────────────────────────────
def setup_chain_logger():
    # Create handlers with different levels and destinations
    console_handler = ConsoleChainHandler(LogLevel.DEBUG, ColoredFormatter())
    file_handler = FileChainHandler(LogLevel.INFO, SimpleFormatter(), "app.log")
    db_handler = DatabaseChainHandler(LogLevel.ERROR, JSONFormatter(), None)
    slack_handler = SlackChainHandler(LogLevel.FATAL, SimpleFormatter(), "https://slack.webhook")
    
    # Build the chain: DEBUG → INFO → ERROR → FATAL
    console_handler.set_next(file_handler).set_next(db_handler).set_next(slack_handler)
    
    # Configure logger
    logger = ChainLogger.get_logger("MyApp")
    logger.set_chain(console_handler)
    
    return logger

# Test
logger = setup_chain_logger()
logger.debug("Starting application")    # → Console only
logger.info("User logged in")           # → Console + File
logger.error("Database connection lost") # → Console + File + DB
logger.fatal("System crash!")           # → Console + File + DB + Slack
```

## Chain Flow Diagram

```
logger.error("DB connection lost")
         │
         ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ ConsoleHandler  │───►│  FileHandler    │───►│ DatabaseHandler │───►│  SlackHandler   │
│ level: DEBUG    │    │ level: INFO     │    │ level: ERROR    │    │ level: FATAL    │
│                 │    │                 │    │                 │    │                 │
│ ERROR >= DEBUG  │    │ ERROR >= INFO   │    │ ERROR >= ERROR  │    │ ERROR < FATAL   │
│ ✓ WRITE         │    │ ✓ WRITE         │    │ ✓ WRITE         │    │ ✗ SKIP          │
└─────────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

---

# Approach 2: Observer Pattern

## Concept

Logger maintains a **list of handlers** (observers). When a message is logged, **all handlers are notified** and each decides independently whether to process.

```
                    ┌─── ConsoleHandler
                    │
Message → Logger ───┼─── FileHandler
                    │
                    └─── DatabaseHandler
```

## Complete Implementation

```python
from typing import List

# ─────────────────────────────────────────────────────────────────
# Observer Pattern Handler
# ─────────────────────────────────────────────────────────────────
class ObserverHandler(ABC):
    def __init__(self, level: LogLevel, formatter: Formatter):
        self.level = level
        self.formatter = formatter
        self.lock = threading.Lock()
    
    def handle(self, msg: LogMessage):
        """Process message if level matches."""
        if msg.level.value >= self.level.value:
            formatted = self.formatter.format(msg)
            self.write(formatted)
    
    @abstractmethod
    def write(self, formatted_message: str):
        pass


class ConsoleObserverHandler(ObserverHandler):
    def write(self, msg: str):
        with self.lock:
            print(msg)


class FileObserverHandler(ObserverHandler):
    def __init__(self, level: LogLevel, formatter: Formatter, file_path: str):
        super().__init__(level, formatter)
        self.file_path = file_path
    
    def write(self, msg: str):
        with self.lock:
            with open(self.file_path, 'a') as f:
                f.write(msg + '\n')


class DatabaseObserverHandler(ObserverHandler):
    def __init__(self, level: LogLevel, formatter: Formatter, db_connection):
        super().__init__(level, formatter)
        self.db = db_connection
    
    def write(self, msg: str):
        with self.lock:
            print(f"[DB INSERT] {msg}")


class SlackObserverHandler(ObserverHandler):
    def __init__(self, level: LogLevel, formatter: Formatter, webhook_url: str):
        super().__init__(level, formatter)
        self.webhook_url = webhook_url
    
    def write(self, msg: str):
        print(f"[SLACK ALERT] {msg}")


# ─────────────────────────────────────────────────────────────────
# Logger with Observer Pattern
# ─────────────────────────────────────────────────────────────────
class ObserverLogger:
    _instances = {}
    _lock = threading.Lock()
    
    def __init__(self, name: str):
        self.name = name
        self.min_level = LogLevel.DEBUG
        self.handlers: List[ObserverHandler] = []
    
    @classmethod
    def get_logger(cls, name: str = "root") -> 'ObserverLogger':
        with cls._lock:
            if name not in cls._instances:
                cls._instances[name] = ObserverLogger(name)
            return cls._instances[name]
    
    def add_handler(self, handler: ObserverHandler):
        """Add a handler (observer) to the logger."""
        self.handlers.append(handler)
    
    def remove_handler(self, handler: ObserverHandler):
        """Remove a handler from the logger."""
        self.handlers.remove(handler)
    
    def set_level(self, level: LogLevel):
        """Set minimum log level for this logger."""
        self.min_level = level
    
    def log(self, level: LogLevel, message: str):
        if level.value < self.min_level.value:
            return
        
        msg = LogMessage(level, message, self.name)
        
        # Notify all handlers (Observer pattern)
        for handler in self.handlers:
            handler.handle(msg)
    
    def debug(self, message: str):
        self.log(LogLevel.DEBUG, message)
    
    def info(self, message: str):
        self.log(LogLevel.INFO, message)
    
    def warn(self, message: str):
        self.log(LogLevel.WARN, message)
    
    def error(self, message: str):
        self.log(LogLevel.ERROR, message)
    
    def fatal(self, message: str):
        self.log(LogLevel.FATAL, message)


# ─────────────────────────────────────────────────────────────────
# Usage: Observer Pattern
# ─────────────────────────────────────────────────────────────────
def setup_observer_logger():
    logger = ObserverLogger.get_logger("MyApp")
    
    # Add handlers independently (order doesn't matter)
    logger.add_handler(ConsoleObserverHandler(LogLevel.DEBUG, ColoredFormatter()))
    logger.add_handler(FileObserverHandler(LogLevel.INFO, SimpleFormatter(), "app.log"))
    logger.add_handler(DatabaseObserverHandler(LogLevel.ERROR, JSONFormatter(), None))
    logger.add_handler(SlackObserverHandler(LogLevel.FATAL, SimpleFormatter(), "https://slack"))
    
    return logger

# Test
logger = setup_observer_logger()
logger.debug("Starting application")    # → Console only
logger.info("User logged in")           # → Console + File
logger.error("Database connection lost") # → Console + File + DB
logger.fatal("System crash!")           # → Console + File + DB + Slack
```

## Observer Flow Diagram

```
logger.error("DB connection lost")
         │
         ▼
┌─────────────────────────────────────┐
│       Notify ALL handlers           │
└─────────────────────────────────────┘
         │
    ┌────┼────┬────────┬────────┐
    ▼    ▼    ▼        ▼        ▼
┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
│Console│ │ File  │ │  DB   │ │ Slack │
│ DEBUG │ │ INFO  │ │ ERROR │ │ FATAL │
│       │ │       │ │       │ │       │
│ERROR>=│ │ERROR>=│ │ERROR>=│ │ERROR< │
│DEBUG  │ │INFO   │ │ERROR  │ │FATAL  │
│✓ WRITE│ │✓ WRITE│ │✓ WRITE│ │✗ SKIP │
└───────┘ └───────┘ └───────┘ └───────┘
```

---

# Comparison

## Code Differences

| Aspect | Chain of Responsibility | Observer |
|--------|------------------------|----------|
| Handler connection | `handler.set_next(next)` | `logger.add_handler(h)` |
| Handler knows next? | ✅ Yes (`self.next_handler`) | ❌ No |
| Logger structure | Has `chain_head` | Has `handlers[]` list |
| Adding handler | Rebuild chain | Just append |

## Pros and Cons

| Aspect | Chain of Responsibility | Observer |
|--------|------------------------|----------|
| **Adding handler** | ❌ Complex (rebuild chain) | ✅ Easy (`add_handler`) |
| **Removing handler** | ❌ Complex (relink) | ✅ Easy (`remove_handler`) |
| **Order control** | ✅ Explicit order | ❌ No order guarantee |
| **Stop processing** | ✅ Can stop chain | ❌ All handlers run |
| **Handler coupling** | ❌ Knows next handler | ✅ Independent |
| **Classic logging** | ✅ Log4j style | ✅ Python logging |

## When to Use

| Use Chain of Responsibility | Use Observer |
|----------------------------|--------------|
| Order of processing matters | Handlers are independent |
| Need to stop at certain level | All matching handlers should run |
| Classic hierarchical logging | Flexible, dynamic configuration |
| Handlers have natural sequence | Runtime add/remove handlers |

---

# Output Comparison

Both produce **identical outputs** for this configuration:

```
logger.debug("Starting app")    → Console
logger.info("User logged in")   → Console + File  
logger.error("DB failed")       → Console + File + DB
logger.fatal("Crash!")          → Console + File + DB + Slack
```

The difference is in **extensibility and configuration**, not output.

---

# Class Diagrams

## Chain of Responsibility

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ChainLogger (Singleton)                                        │
│      ├── name, min_level                                        │
│      ├── chain_head: ChainHandler                               │
│      └── log(level, message)                                    │
│                 │                                               │
│                 ▼                                               │
│  ChainHandler (Abstract)                                        │
│      ├── level, formatter, next_handler                         │
│      ├── handle(msg) → process → pass to next                   │
│      └── write(msg) [abstract]                                  │
│           ▲                                                     │
│           │                                                     │
│   ┌───────┴───────┬────────────┬────────────┐                   │
│   │               │            │            │                   │
│ Console        File       Database       Slack                  │
│ Handler       Handler     Handler       Handler                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Observer

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ObserverLogger (Singleton)                                     │
│      ├── name, min_level                                        │
│      ├── handlers: List[ObserverHandler]                        │
│      ├── add_handler(h), remove_handler(h)                      │
│      └── log(level, message) → notify all handlers              │
│                                                                 │
│  ObserverHandler (Abstract)                                     │
│      ├── level, formatter                                       │
│      ├── handle(msg) → check level → write                      │
│      └── write(msg) [abstract]                                  │
│           ▲                                                     │
│           │                                                     │
│   ┌───────┴───────┬────────────┬────────────┐                   │
│   │               │            │            │                   │
│ Console        File       Database       Slack                  │
│ Handler       Handler     Handler       Handler                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# Design Patterns Summary

| Pattern | Where | Both Approaches? |
|---------|-------|------------------|
| **Singleton** | Logger instance | ✅ Yes |
| **Strategy** | Formatter (Simple, JSON) | ✅ Yes |
| **Chain of Responsibility** | Handler chain | Approach 1 only |
| **Observer** | Handler notification | Approach 2 only |
| **Template Method** | Handler.handle() | ✅ Yes |

---

# Interview Tips

1. **Start with requirements** — Don't jump to patterns
2. **Both approaches are valid** — Pick one and justify
3. **Mention trade-offs** — Shows maturity
4. **Thread safety is critical** — Always mention locks
5. **Strategy for Formatter** — Easy pattern to include

---

*This article was created as part of LLD interview practice for Amazon SDE-2 (L5) level.*
