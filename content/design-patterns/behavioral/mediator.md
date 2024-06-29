---
title: Mediator
type: docs
sidebar:
  open: true
toc: true
params:
  editURL: 
---

The Mediator design pattern is a behavioral design pattern that encapsulates how a set of objects interact. It promotes loose coupling by preventing objects from referring to each other explicitly and allows their interaction to be varied independently. Instead of having each object communicate directly with each other, they communicate through a mediator.

### Key Concepts

1. **Mediator Interface**: Defines the interface for communication between Colleague objects.
2. **Concrete Mediator**: Implements the Mediator interface and coordinates communication between Colleague objects.
3. **Colleague Interface**: Defines an interface for communication with other Colleagues through the Mediator.
4. **Concrete Colleague**: Implements the Colleague interface and interacts with other Colleagues through the Mediator.

### When to Use

- When a set of objects communicate in complex but well-defined ways.
- When you want to re-use an object in a different context without changing its existing behavior.
- When you want to make a complex system easier to understand, maintain, and extend.

### Example Implementation in Python

Let's consider a chat room example where multiple users can send messages to each other through a central chat mediator.

```python
from abc import ABC, abstractmethod

# Mediator interface
class ChatMediator(ABC):
    @abstractmethod
    def show_message(self, user, message):
        pass

# Concrete Mediator
class ChatRoom(ChatMediator):
    def show_message(self, user, message):
        print(f"[{user.name}]: {message}")

# Colleague interface
class User(ABC):
    def __init__(self, name, mediator):
        self.name = name
        self.mediator = mediator

    @abstractmethod
    def send(self, message):
        pass

    @abstractmethod
    def receive(self, message):
        pass

# Concrete Colleague
class ConcreteUser(User):
    def send(self, message):
        self.mediator.show_message(self, message)

    def receive(self, message):
        print(f"{self.name} received: {message}")

# Client code
if __name__ == "__main__":
    mediator = ChatRoom()

    user1 = ConcreteUser("Alice", mediator)
    user2 = ConcreteUser("Bob", mediator)

    user1.send("Hi Bob!")
    user2.send("Hello Alice!")
```

### Benefits

- **Reduced Coupling**: Colleague objects are decoupled from each other. They interact through the mediator, which reduces direct dependencies between them.
- **Improved Maintainability**: Adding new communication paths or modifying existing ones becomes easier by only changing the mediator.
- **Enhanced Flexibility**: Communication between objects can be changed at runtime by changing the mediator.

### Drawbacks

- **Complexity**: The mediator can become complex if it needs to handle many different interactions.
- **Potential Performance Issues**: The mediator can become a bottleneck if it handles a large number of interactions or complex logic.

### Real-World Example Use Cases

1. **Air Traffic Control System**: The mediator (air traffic controller) coordinates the interactions between multiple airplanes, preventing direct communication and collisions.
2. **UI Frameworks**: A dialog box acting as a mediator coordinates interactions between various UI components like buttons, text fields, and labels.
3. **Message Broker Systems**: A message broker (mediator) coordinates message exchanges between various services in a microservices architecture.

### Extended Example in Python

Consider an extended example where we implement a simplified air traffic control system using the Mediator pattern.

```python
from abc import ABC, abstractmethod

# Mediator interface
class ATCMediator(ABC):
    @abstractmethod
    def register_aircraft(self, aircraft):
        pass

    @abstractmethod
    def broadcast_message(self, sender, message):
        pass

# Concrete Mediator
class AirTrafficControl(ATCMediator):
    def __init__(self):
        self.aircrafts = []

    def register_aircraft(self, aircraft):
        self.aircrafts.append(aircraft)

    def broadcast_message(self, sender, message):
        for aircraft in self.aircrafts:
            if aircraft != sender:
                aircraft.receive_message(sender, message)

# Colleague interface
class Aircraft(ABC):
    def __init__(self, call_sign, mediator):
        self.call_sign = call_sign
        self.mediator = mediator
        self.mediator.register_aircraft(self)

    @abstractmethod
    def send_message(self, message):
        pass

    @abstractmethod
    def receive_message(self, sender, message):
        pass

# Concrete Colleague
class ConcreteAircraft(Aircraft):
    def send_message(self, message):
        print(f"{self.call_sign} sending message: {message}")
        self.mediator.broadcast_message(self, message)

    def receive_message(self, sender, message):
        print(f"{self.call_sign} received message from {sender.call_sign}: {message}")

# Client code
if __name__ == "__main__":
    mediator = AirTrafficControl()

    aircraft1 = ConcreteAircraft("Flight 101", mediator)
    aircraft2 = ConcreteAircraft("Flight 202", mediator)
    aircraft3 = ConcreteAircraft("Flight 303", mediator)

    aircraft1.send_message("Requesting landing clearance.")
    aircraft2.send_message("Acknowledged.")
    aircraft3.send_message("Requesting takeoff clearance.")
```