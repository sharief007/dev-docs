---
title: Observer
type: docs
sidebar:
  open: true
toc: true
params:
  editURL: 
---

The Observer design pattern is a behavioral design pattern that defines a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically. It is typically used to implement distributed event-handling systems.

### Key Concepts

1. **Subject**: The subject (or observable) maintains a list of its observers and notifies them of any state changes, usually by calling one of their methods.
2. **Observers**: Observers (or subscribers) register with the subject to receive updates when the subject changes state.

### Benefits

- **Decoupling**: The subject and observers are loosely coupled. The subject does not need to know the concrete class of the observer, only that the observer implements a certain interface.
- **Flexibility**: You can add or remove observers at runtime.
- **Scalability**: Multiple observers can subscribe to the same subject, and multiple subjects can be observed by a single observer.

### Example in Python

Let's implement a simple example of a stock market where multiple investors (observers) watch the stock (subject) for price changes.

```python
from abc import ABC, abstractmethod

class Observer(ABC):
    @abstractmethod
    def update(self, price):
        pass

class Subject(ABC):
    @abstractmethod
    def register_observer(self, observer):
        pass

    @abstractmethod
    def remove_observer(self, observer):
        pass

    @abstractmethod
    def notify_observers(self):
        pass

class Stock(Subject):
    def __init__(self):
        self._observers = []
        self._price = 0

    def register_observer(self, observer):
        self._observers.append(observer)

    def remove_observer(self, observer):
        self._observers.remove(observer)

    def notify_observers(self):
        for observer in self._observers:
            observer.update(self._price)

    def set_price(self, price):
        self._price = price
        self.notify_observers()

class Investor(Observer):
    def __init__(self, name):
        self._name = name

    def update(self, price):
        print(f"Investor {self._name} notified. New stock price: {price}")

# Client code
if __name__ == "__main__":
    stock = Stock()

    investor1 = Investor("Alice")
    investor2 = Investor("Bob")

    stock.register_observer(investor1)
    stock.register_observer(investor2)

    stock.set_price(100)
    stock.set_price(200)

    stock.remove_observer(investor1)

    stock.set_price(300)
```

### Another Example: Weather Station

Here is another example with a weather station that notifies various displays (observers) about weather changes.

```python
from abc import ABC, abstractmethod

class Observer(ABC):
    @abstractmethod
    def update(self, temperature, humidity, pressure):
        pass

class Subject(ABC):
    @abstractmethod
    def register_observer(self, observer):
        pass

    @abstractmethod
    def remove_observer(self, observer):
        pass

    @abstractmethod
    def notify_observers(self):
        pass

class WeatherData(Subject):
    def __init__(self):
        self._observers = []
        self._temperature = 0
        self._humidity = 0
        self._pressure = 0

    def register_observer(self, observer):
        self._observers.append(observer)

    def remove_observer(self, observer):
        self._observers.remove(observer)

    def notify_observers(self):
        for observer in self._observers:
            observer.update(self._temperature, self._humidity, self._pressure)

    def measurements_changed(self):
        self.notify_observers()

    def set_measurements(self, temperature, humidity, pressure):
        self._temperature = temperature
        self._humidity = humidity
        self._pressure = pressure
        self.measurements_changed()

class CurrentConditionsDisplay(Observer):
    def update(self, temperature, humidity, pressure):
        print(f"Current conditions: {temperature}F degrees, {humidity}% humidity, {pressure} pressure")

class StatisticsDisplay(Observer):
    def update(self, temperature, humidity, pressure):
        print(f"Statistics: Avg/Max/Min temperature = {temperature}/{temperature}/{temperature}")

class ForecastDisplay(Observer):
    def update(self, temperature, humidity, pressure):
        print(f"Forecast: More of the same")

# Client code
if __name__ == "__main__":
    weather_data = WeatherData()

    current_display = CurrentConditionsDisplay()
    statistics_display = StatisticsDisplay()
    forecast_display = ForecastDisplay()

    weather_data.register_observer(current_display)
    weather_data.register_observer(statistics_display)
    weather_data.register_observer(forecast_display)

    weather_data.set_measurements(80, 65, 30.4)
    weather_data.set_measurements(82, 70, 29.2)
    weather_data.set_measurements(78, 90, 29.2)
```

In both examples, the Observer pattern allows the subject to notify all registered observers of any changes in its state, making it easy to add or remove observers without changing the subject's code. This ensures loose coupling between the subject and its observers.