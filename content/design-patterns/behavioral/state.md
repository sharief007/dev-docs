---
title: State
type: docs
sidebar:
  open: true
toc: true
params:
  editURL: 
---

The State design pattern is a behavioral design pattern that allows an object to change its behavior when its internal state changes. The object will appear to change its class.

```python
from abc import ABC, abstractmethod

class State(ABC):
    @abstractmethod
    def insert_coin(self, context):
        pass

    @abstractmethod
    def eject_coin(self, context):
        pass

    @abstractmethod
    def dispense(self, context):
        pass

class NoCoinState(State):
    def insert_coin(self, context):
        print("Coin inserted.")
        context.state = HasCoinState()

    def eject_coin(self, context):
        print("No coin to eject.")

    def dispense(self, context):
        print("Insert coin first.")

class HasCoinState(State):
    def insert_coin(self, context):
        print("Coin already inserted.")

    def eject_coin(self, context):
        print("Coin ejected.")
        context.state = NoCoinState()

    def dispense(self, context):
        print("Item dispensed.")
        context.state = NoCoinState()

class VendingMachine:
    def __init__(self):
        self.state = NoCoinState()

    def insert_coin(self):
        self.state.insert_coin(self)

    def eject_coin(self):
        self.state.eject_coin(self)

    def dispense(self):
        self.state.dispense(self)

# Client code
if __name__ == "__main__":
    machine = VendingMachine()
    
    machine.insert_coin()
    machine.dispense()
    machine.eject_coin()
    
    machine.insert_coin()
    machine.eject_coin()
    machine.dispense()

```

The State pattern allows the objects (ex: VendingMachine) to change their behavior dynamically based on their current state without using complex conditionals. 