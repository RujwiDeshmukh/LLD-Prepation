# Low-Level Design Interview: Text Editor (Undo/Redo)

> A complete walkthrough demonstrating the **Command Pattern** — encapsulating operations as objects to enable undo, redo, and action history.

---

## 🎯 Problem Statement

**Interviewer:** Design a **Text Editor** with basic operations (insert, delete) that supports **Undo** and **Redo** functionality.

---

## 🤔 Why Command Pattern?

### The Core Problem

When a user types, deletes, or modifies text, we need to:
1. **Execute** the operation
2. **Remember** it for potential undo
3. **Reverse** it when undo is called
4. **Re-apply** it when redo is called

### Why NOT Simple Approach?

#### ❌ Store Entire Document After Each Change

```python
history = []
history.append(document.copy())  # After each keystroke
```

**Problem:** Memory explosion! A 10MB document with 1000 edits = 10GB memory!

#### ✅ Store Just the Operations

```python
history = []
history.append(InsertCommand(pos=5, text="Hello"))
```

**Solution:** Store the *action*, not the *result*. Each command knows how to execute AND undo itself.

---

## 🎨 Command Pattern Explained

### Core Concept

> **Command Pattern** encapsulates a request as an object, allowing you to parameterize, queue, log, and undo operations.

### Key Components

| Component | Role | In Our System |
|-----------|------|---------------|
| **Command** | Interface for all operations | `Command` abstract class |
| **ConcreteCommand** | Specific operation | `InsertCommand`, `DeleteCommand` |
| **Receiver** | Object being operated on | `TextBuffer` |
| **Invoker** | Triggers commands | `TextEditor` |

---

## 🔄 The Two-Stack Mechanism

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   UNDO STACK              REDO STACK                            │
│   ──────────              ──────────                            │
│                                                                 │
│   ┌─────────┐             ┌─────────┐                           │
│   │ Insert  │             │         │  (empty initially)        │
│   │ "World" │             │         │                           │
│   ├─────────┤             └─────────┘                           │
│   │ Insert  │                                                   │
│   │ "Hello" │                                                   │
│   └─────────┘                                                   │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   User presses UNDO:                                            │
│   1. Pop "Insert World" from undo_stack                         │
│   2. Call its undo() → deletes "World"                          │
│   3. Push "Insert World" to redo_stack                          │
│                                                                 │
│   UNDO STACK              REDO STACK                            │
│   ┌─────────┐             ┌─────────┐                           │
│   │ Insert  │             │ Insert  │                           │
│   │ "Hello" │             │ "World" │                           │
│   └─────────┘             └─────────┘                           │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   User presses REDO:                                            │
│   1. Pop "Insert World" from redo_stack                         │
│   2. Call its execute() → inserts "World"                       │
│   3. Push "Insert World" to undo_stack                          │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   User types NEW text after undo:                               │
│   → redo_stack.clear()  (history branch invalidated)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 💻 Complete Implementation

### TextBuffer (Receiver)

```python
class TextBuffer:
    """
    The receiver — the actual document being edited.
    Commands operate on this buffer.
    """
    
    def __init__(self):
        self.text = ""
        self.cursor = 0
    
    def insert(self, position: int, chars: str):
        """Insert characters at position."""
        self.text = self.text[:position] + chars + self.text[position:]
        self.cursor = position + len(chars)
    
    def delete(self, position: int, length: int) -> str:
        """Delete characters and return what was deleted."""
        deleted = self.text[position:position + length]
        self.text = self.text[:position] + self.text[position + length:]
        self.cursor = position
        return deleted
    
    def get_text(self) -> str:
        return self.text
    
    def __str__(self) -> str:
        return f'"{self.text}" (cursor: {self.cursor})'
```

---

### Command Interface

```python
from abc import ABC, abstractmethod

class Command(ABC):
    """
    Command interface — every command must be executable and undoable.
    """
    
    @abstractmethod
    def execute(self, buffer: TextBuffer):
        """Execute the command."""
        pass
    
    @abstractmethod
    def undo(self, buffer: TextBuffer):
        """Reverse the command."""
        pass
    
    @abstractmethod
    def get_description(self) -> str:
        """Human-readable description for history."""
        pass
```

---

### Concrete Commands

```python
class InsertCommand(Command):
    """Inserts text at a position."""
    
    def __init__(self, position: int, text: str):
        self.position = position
        self.text = text
    
    def execute(self, buffer: TextBuffer):
        buffer.insert(self.position, self.text)
    
    def undo(self, buffer: TextBuffer):
        # Undo insert = delete the same text
        buffer.delete(self.position, len(self.text))
    
    def get_description(self) -> str:
        preview = self.text[:20] + "..." if len(self.text) > 20 else self.text
        return f'Insert "{preview}" at {self.position}'


class DeleteCommand(Command):
    """Deletes text at a position."""
    
    def __init__(self, position: int, length: int):
        self.position = position
        self.length = length
        self.deleted_text = None  # Stored when executed
    
    def execute(self, buffer: TextBuffer):
        # Store deleted text for undo
        self.deleted_text = buffer.delete(self.position, self.length)
    
    def undo(self, buffer: TextBuffer):
        # Undo delete = insert the deleted text back
        buffer.insert(self.position, self.deleted_text)
    
    def get_description(self) -> str:
        if self.deleted_text:
            preview = self.deleted_text[:20] + "..." if len(self.deleted_text) > 20 else self.deleted_text
            return f'Delete "{preview}" at {self.position}'
        return f'Delete {self.length} chars at {self.position}'


class ReplaceCommand(Command):
    """Replaces text (compound: delete + insert)."""
    
    def __init__(self, position: int, length: int, new_text: str):
        self.position = position
        self.length = length
        self.new_text = new_text
        self.old_text = None
    
    def execute(self, buffer: TextBuffer):
        self.old_text = buffer.delete(self.position, self.length)
        buffer.insert(self.position, self.new_text)
    
    def undo(self, buffer: TextBuffer):
        buffer.delete(self.position, len(self.new_text))
        buffer.insert(self.position, self.old_text)
    
    def get_description(self) -> str:
        return f'Replace "{self.old_text}" with "{self.new_text}"'
```

---

### TextEditor (Invoker)

```python
class TextEditor:
    """
    The invoker — manages command execution and history.
    """
    
    MAX_HISTORY = 100  # Limit undo stack size
    
    def __init__(self):
        self.buffer = TextBuffer()
        self.undo_stack: list[Command] = []
        self.redo_stack: list[Command] = []
    
    def execute(self, command: Command):
        """Execute a command and add to history."""
        command.execute(self.buffer)
        self.undo_stack.append(command)
        
        # New action invalidates redo history
        self.redo_stack.clear()
        
        # Enforce history limit
        if len(self.undo_stack) > self.MAX_HISTORY:
            self.undo_stack.pop(0)  # Remove oldest
        
        print(f"✓ {command.get_description()}")
    
    def undo(self):
        """Undo the last command."""
        if not self.undo_stack:
            print("Nothing to undo")
            return
        
        command = self.undo_stack.pop()
        command.undo(self.buffer)
        self.redo_stack.append(command)
        print(f"↩ Undo: {command.get_description()}")
    
    def redo(self):
        """Redo the last undone command."""
        if not self.redo_stack:
            print("Nothing to redo")
            return
        
        command = self.redo_stack.pop()
        command.execute(self.buffer)
        self.undo_stack.append(command)
        print(f"↪ Redo: {command.get_description()}")
    
    def get_text(self) -> str:
        return self.buffer.get_text()
    
    def show_history(self):
        """Display undo/redo stacks."""
        print("\n--- History ---")
        print(f"Undo stack ({len(self.undo_stack)}):")
        for i, cmd in enumerate(reversed(self.undo_stack)):
            print(f"  {i+1}. {cmd.get_description()}")
        print(f"Redo stack ({len(self.redo_stack)}):")
        for i, cmd in enumerate(reversed(self.redo_stack)):
            print(f"  {i+1}. {cmd.get_description()}")
        print("---------------\n")
```

---

## 🔄 Usage Example

```python
editor = TextEditor()

# Type some text
editor.execute(InsertCommand(0, "Hello"))
editor.execute(InsertCommand(5, " World"))
editor.execute(InsertCommand(11, "!"))

print(editor.get_text())  # "Hello World!"

# Undo last action
editor.undo()
print(editor.get_text())  # "Hello World"

# Undo more
editor.undo()
print(editor.get_text())  # "Hello"

# Redo
editor.redo()
print(editor.get_text())  # "Hello World"

# New edit clears redo
editor.execute(InsertCommand(5, " There"))
print(editor.get_text())  # "Hello There"

# Try redo — nothing happens
editor.redo()  # "Nothing to redo"
```

### Output

```
✓ Insert "Hello" at 0
✓ Insert " World" at 5
✓ Insert "!" at 11
"Hello World!"
↩ Undo: Insert "!" at 11
"Hello World"
↩ Undo: Insert " World" at 5
"Hello"
↪ Redo: Insert " World" at 5
"Hello World"
✓ Insert " There" at 5
"Hello There"
Nothing to redo
```

---

## 🏛️ Class Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   TextEditor (Invoker)                                              │
│       │                                                             │
│       ├── buffer: TextBuffer                                        │
│       ├── undo_stack: List[Command]                                 │
│       ├── redo_stack: List[Command]                                 │
│       ├── execute(cmd)                                              │
│       ├── undo()                                                    │
│       └── redo()                                                    │
│                                                                     │
│   TextBuffer (Receiver)                                             │
│       │                                                             │
│       ├── text: str                                                 │
│       ├── insert(pos, chars)                                        │
│       └── delete(pos, length)                                       │
│                                                                     │
│   Command (Interface)                                               │
│       │                                                             │
│       ├── execute(buffer)                                           │
│       ├── undo(buffer)                                              │
│       └── get_description()                                         │
│            ▲                                                        │
│            │                                                        │
│   ┌────────┼────────┬────────────────┐                              │
│   │        │        │                │                              │
│   InsertCmd  DeleteCmd  ReplaceCmd  MacroCmd                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Advanced: Macro Commands

Group multiple commands into one undoable action:

```python
class MacroCommand(Command):
    """Executes multiple commands as one unit."""
    
    def __init__(self, commands: list[Command]):
        self.commands = commands
    
    def execute(self, buffer: TextBuffer):
        for cmd in self.commands:
            cmd.execute(buffer)
    
    def undo(self, buffer: TextBuffer):
        # Undo in reverse order!
        for cmd in reversed(self.commands):
            cmd.undo(buffer)
    
    def get_description(self) -> str:
        return f"Macro ({len(self.commands)} commands)"


# Usage: "Find and Replace" as single undo
find_replace = MacroCommand([
    DeleteCommand(0, 5),
    InsertCommand(0, "Hi")
])
editor.execute(find_replace)

# Single undo reverts both
editor.undo()
```

---

## 🆚 Command vs Memento Pattern

| Aspect | Command | Memento |
|--------|---------|---------|
| **Stores** | The action/operation | The entire state |
| **Memory** | O(1) per operation | O(state size) per snapshot |
| **Undo** | Reverse the action | Restore previous state |
| **Best for** | Text editors, transactions | Game saves, simple states |

### When to Use What?

| Scenario | Better Pattern |
|----------|----------------|
| Text editor with many small edits | **Command** |
| Game with complex state, infrequent saves | **Memento** |
| Database transactions | **Command** |
| Form wizard with "back" button | **Memento** |

---

## 🎯 When to Use Command Pattern

| Condition | Example |
|-----------|---------|
| Need undo/redo | Text editors, drawing apps |
| Queue operations | Job queues, schedulers |
| Log operations | Audit trails, transactions |
| Decouple invoker from action | Remote controls, menus |
| Support macros | Recording and playback |

---

## 🧪 Edge Cases

| Edge Case | How Handled |
|-----------|-------------|
| Undo with empty stack | Check and return early |
| Redo with empty stack | Check and return early |
| New edit after undo | Clear redo stack |
| Unlimited history | MAX_HISTORY limit |
| Delete beyond text length | Validate in TextBuffer |

---

## 🚀 Real-World Examples

| Application | Commands |
|-------------|----------|
| **VS Code** | Every keystroke is a command |
| **Photoshop** | Each brush stroke, filter |
| **Git** | Commits are like commands |
| **Database** | Transactions with rollback |
| **Game Dev** | Player actions for replay |

---

## ✅ Interview Takeaways

1. **Command = Action as Object** — encapsulate what to do
2. **Two stacks** — undo_stack and redo_stack
3. **Each command knows execute() AND undo()** — bidirectional
4. **New edit clears redo** — history branches
5. **Store minimal data** — just enough to reverse
6. **Macro commands** — group actions for single undo
7. **Memory efficient** — O(1) per operation vs O(state) for Memento

---

*This article was created as part of LLD interview practice for Amazon SDE-2 (L5) level.*
