---
title: Command
type: docs
sidebar:
  open: true
toc: true
params:
  editURL: 
---

The Command design pattern is a behavioral design pattern that turns a request into a stand-alone object that contains all information about the request. This transformation allows you to parameterize methods with different requests, delay or queue a request's execution, and support undoable operations.

### Key Concepts

1. **Command**: An interface that declares a method for executing a command.
2. **Concrete Command**: Implements the Command interface and defines the binding between a receiver object and an action.
3. **Receiver**: The object that performs the actual action when the command's execute method is called.
4. **Invoker**: Asks the command to carry out the request.
5. **Client**: Creates a ConcreteCommand object and sets its receiver.

### Benefits

- **Decoupling**: Decouples the object that invokes the operation from the one that knows how to perform it.
- **Extensibility**: New commands can be added without changing existing code.
- **Support for Undo/Redo**: Commands can store state for undoable operations.

### Example in Python

Let's consider an example where we implement a simple text editor with commands for writing text, undoing the last operation, and redoing an undone operation.

```python
from abc import ABC, abstractmethod

# Command Interface
class Command(ABC):
    @abstractmethod
    def execute(self):
        pass

    @abstractmethod
    def undo(self):
        pass

# Concrete Command
class WriteCommand(Command):
    def __init__(self, receiver, text):
        self.receiver = receiver
        self.text = text

    def execute(self):
        self.receiver.write(self.text)

    def undo(self):
        self.receiver.erase(self.text)

# Receiver
class TextEditor:
    def __init__(self):
        self.content = ""

    def write(self, text):
        self.content += text

    def erase(self, text):
        self.content = self.content[:-len(text)]

    def __str__(self):
        return self.content

# Invoker
class TextEditorInvoker:
    def __init__(self):
        self.history = []
        self.undo_stack = []

    def execute_command(self, command):
        command.execute()
        self.history.append(command)
        self.undo_stack = []

    def undo(self):
        if self.history:
            command = self.history.pop()
            command.undo()
            self.undo_stack.append(command)

    def redo(self):
        if self.undo_stack:
            command = self.undo_stack.pop()
            command.execute()
            self.history.append(command)

# Client code
if __name__ == "__main__":
    editor = TextEditor()
    invoker = TextEditorInvoker()

    command1 = WriteCommand(editor, "Hello ")
    command2 = WriteCommand(editor, "World!")

    invoker.execute_command(command1)
    invoker.execute_command(command2)
    
    print(f"After writing: {editor}")

    invoker.undo()
    print(f"After undo: {editor}")

    invoker.redo()
    print(f"After redo: {editor}")
```

### Benefits in Practice

- **Decoupling**: The client code does not depend on the details of the command's execution.
- **Undo/Redo Support**: The ability to undo and redo commands adds flexibility and control.
- **Extensibility**: New commands can be easily added without changing the invoker or the client code.

### Another Example: Remote Control

Let's consider another example where we implement a remote control for a home automation system.

```python
from abc import ABC, abstractmethod

# Command Interface
class Command(ABC):
    @abstractmethod
    def execute(self):
        pass

    @abstractmethod
    def undo(self):
        pass

# Concrete Commands
class LightOnCommand(Command):
    def __init__(self, light):
        self.light = light

    def execute(self):
        self.light.on()

    def undo(self):
        self.light.off()

class LightOffCommand(Command):
    def __init__(self, light):
        self.light = light

    def execute(self):
        self.light.off()

    def undo(self):
        self.light.on()

class StereoOnCommand(Command):
    def __init__(self, stereo):
        self.stereo = stereo

    def execute(self):
        self.stereo.on()

    def undo(self):
        self.stereo.off()

class StereoOffCommand(Command):
    def __init__(self, stereo):
        self.stereo = stereo

    def execute(self):
        self.stereo.off()

    def undo(self):
        self.stereo.on()

# Receiver
class Light:
    def on(self):
        print("The light is on")

    def off(self):
        print("The light is off")

class Stereo:
    def on(self):
        print("The stereo is on")

    def off(self):
        print("The stereo is off")

# Invoker
class RemoteControl:
    def __init__(self):
        self.command_slots = {}
        self.history = []

    def set_command(self, slot, command):
        self.command_slots[slot] = command

    def press_button(self, slot):
        command = self.command_slots.get(slot)
        if command:
            command.execute()
            self.history.append(command)

    def press_undo(self):
        if self.history:
            command = self.history.pop()
            command.undo()

# Client code
if __name__ == "__main__":
    remote = RemoteControl()
    
    living_room_light = Light()
    stereo = Stereo()
    
    light_on = LightOnCommand(living_room_light)
    light_off = LightOffCommand(living_room_light)
    stereo_on = StereoOnCommand(stereo)
    stereo_off = StereoOffCommand(stereo)
    
    remote.set_command(1, light_on)
    remote.set_command(2, light_off)
    remote.set_command(3, stereo_on)
    remote.set_command(4, stereo_off)
    
    remote.press_button(1)
    remote.press_button(3)
    
    remote.press_undo()
    remote.press_button(2)
    remote.press_undo()
```

The Command design pattern is highly effective for encapsulating requests, decoupling invokers and receivers, supporting undo/redo functionality, and making the system more flexible and maintainable.