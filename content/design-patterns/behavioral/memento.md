---
title: Memento
type: docs
sidebar:
  open: true
toc: true
params:
  editURL: 
---

The Memento design pattern is a behavioral design pattern that allows you to capture and externalize an object's internal state without violating its encapsulation so that the object can be restored to this state later. This is particularly useful for implementing undo or rollback functionality.

### Key Concepts

1. **Memento**: The Memento object stores the internal state of the originator object. It has two interfaces: a narrow interface for caretakers (which should not expose any state) and a wide interface for originators (which allows full access to the state).
2. **Originator**: The object whose state needs to be saved and restored. It creates a memento containing a snapshot of its current state and uses the memento to restore its state.
3. **Caretaker**: The caretaker is responsible for keeping the memento. It never operates on or examines the contents of the memento.

### When to Use

- To provide the ability to undo or rollback changes.
- To save and restore the state of an object without exposing its internal structure.
- When the encapsulation of the object needs to be preserved.

### Example Implementation in Python

Let's consider an example of a text editor where we can save the state of the text and undo changes.

```python
from typing import List

# Memento
class Memento:
    def __init__(self, state: str):
        self._state = state

    def get_state(self) -> str:
        return self._state

# Originator
class TextEditor:
    def __init__(self):
        self._state = ""

    def write(self, text: str):
        self._state += text

    def save(self) -> Memento:
        return Memento(self._state)

    def restore(self, memento: Memento):
        self._state = memento.get_state()

    def __str__(self):
        return self._state

# Caretaker
class EditorHistory:
    def __init__(self):
        self._history: List[Memento] = []

    def push(self, memento: Memento):
        self._history.append(memento)

    def pop(self) -> Memento:
        return self._history.pop()

# Client code
if __name__ == "__main__":
    editor = TextEditor()
    history = EditorHistory()

    editor.write("Hello, ")
    history.push(editor.save())

    editor.write("World!")
    history.push(editor.save())

    print(editor)  # Output: Hello, World!

    editor.restore(history.pop())
    print(editor)  # Output: Hello, 

    editor.restore(history.pop())
    print(editor)  # Output: 
```

### Benefits

- **Encapsulation Preserved**: The internal state of the originator is not exposed to the outside world.
- **Easy to Implement Undo**: Provides a straightforward way to implement undo functionality.
- **Simplifies Originator**: Keeps the originator's code simple by delegating state storage and restoration to the memento and caretaker.

### Drawbacks

- **Memory Overhead**: Storing mementos can consume a significant amount of memory if the state is large or changes frequently.
- **Complexity in Managing Mementos**: If not managed properly, the history of mementos can grow quickly, leading to potential performance issues.

### Real-World Use Cases

1. **Text Editors**: Implementing undo and redo functionality.
2. **Database Transactions**: Saving the state of a transaction to rollback if needed.
3. **Game Development**: Saving the state of the game at certain points to allow players to return to those points.
4. **Graphic Design Applications**: Keeping track of changes to allow users to undo and redo actions.

### Extended Example: Game State Management

Consider a game where the player's state (like position, health, and score) needs to be saved and restored.

```python
from typing import List

# Memento
class GameStateMemento:
    def __init__(self, state: dict):
        self._state = state.copy()

    def get_state(self) -> dict:
        return self._state

# Originator
class Game:
    def __init__(self):
        self._state = {
            "level": 1,
            "health": 100,
            "score": 0
        }

    def play(self, level: int, health: int, score: int):
        self._state["level"] = level
        self._state["health"] = health
        self._state["score"] = score

    def save(self) -> GameStateMemento:
        return GameStateMemento(self._state)

    def restore(self, memento: GameStateMemento):
        self._state = memento.get_state()

    def __str__(self):
        return str(self._state)

# Caretaker
class GameHistory:
    def __init__(self):
        self._history: List[GameStateMemento] = []

    def push(self, memento: GameStateMemento):
        self._history.append(memento)

    def pop(self) -> GameStateMemento:
        return self._history.pop()

# Client code
if __name__ == "__main__":
    game = Game()
    history = GameHistory()

    game.play(1, 100, 10)
    history.push(game.save())

    game.play(2, 80, 20)
    history.push(game.save())

    print(game)  # Output: {'level': 2, 'health': 80, 'score': 20}

    game.restore(history.pop())
    print(game)  # Output: {'level': 1, 'health': 100, 'score': 10}

    game.restore(history.pop())
    print(game)  # Output: {'level': 1, 'health': 100, 'score': 0}
```
