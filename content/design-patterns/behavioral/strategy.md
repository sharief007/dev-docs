---
title: Strategy
type: docs
sidebar:
  open: true
toc: true
params:
  editURL: 
---

The Strategy design pattern is a behavioral design pattern that defines a family of algorithms, encapsulates each one, and makes them interchangeable. The strategy pattern lets the algorithm vary independently from the clients that use it.

### Key Concepts

1. **Strategy Interface**: Defines a common interface for all supported algorithms. Context uses this interface to call the algorithm defined by a concrete strategy.
2. **Concrete Strategies**: Implement the strategy interface and contain the actual algorithm.
3. **Context**: Maintains a reference to a strategy object and uses this object to execute the algorithm.

### Benefits

- **Encapsulation**: Each algorithm is encapsulated in a separate class, making the code easier to understand and maintain.
- **Flexibility**: Algorithms can be switched at runtime, allowing dynamic behavior.
- **Decoupling**: The client code is decoupled from the specific algorithm implementations, adhering to the Open/Closed Principle.

### Example in Python

Let's consider an example of a payment processing system where we might use different payment methods such as Credit Card, PayPal, and Bitcoin.

```python
from abc import ABC, abstractmethod

# Strategy Interface
class PaymentStrategy(ABC):
    @abstractmethod
    def pay(self, amount):
        pass

# Concrete Strategies
class CreditCardPayment(PaymentStrategy):
    def __init__(self, card_number, card_expiry, cvv):
        self.card_number = card_number
        self.card_expiry = card_expiry
        self.cvv = cvv

    def pay(self, amount):
        print(f"Paid {amount} using Credit Card ending in {self.card_number[-4:]}")

class PayPalPayment(PaymentStrategy):
    def __init__(self, email):
        self.email = email

    def pay(self, amount):
        print(f"Paid {amount} using PayPal account {self.email}")

class BitcoinPayment(PaymentStrategy):
    def __init__(self, wallet_address):
        self.wallet_address = wallet_address

    def pay(self, amount):
        print(f"Paid {amount} using Bitcoin wallet {self.wallet_address}")

# Context
class ShoppingCart:
    def __init__(self):
        self.items = []
        self.total = 0

    def add_item(self, item, price):
        self.items.append((item, price))
        self.total += price

    def set_payment_strategy(self, strategy):
        self.payment_strategy = strategy

    def checkout(self):
        self.payment_strategy.pay(self.total)

# Client code
if __name__ == "__main__":
    cart = ShoppingCart()
    cart.add_item("Laptop", 1200)
    cart.add_item("Phone", 800)

    # Paying with credit card
    credit_card = CreditCardPayment("1234-5678-9876-5432", "12/25", "123")
    cart.set_payment_strategy(credit_card)
    cart.checkout()

    # Paying with PayPal
    paypal = PayPalPayment("user@example.com")
    cart.set_payment_strategy(paypal)
    cart.checkout()

    # Paying with Bitcoin
    bitcoin = BitcoinPayment("1FfmbHfnpaZjKFvyi1okTjJJusN455paPH")
    cart.set_payment_strategy(bitcoin)
    cart.checkout()
```

### Explanation

1. **Strategy Interface**: `PaymentStrategy` defines a common interface for all payment methods with the `pay` method.
2. **Concrete Strategies**:
    - `CreditCardPayment`: Implements payment using a credit card.
    - `PayPalPayment`: Implements payment using PayPal.
    - `BitcoinPayment`: Implements payment using Bitcoin.
3. **Context**: `ShoppingCart` maintains a reference to a `PaymentStrategy` object. The `set_payment_strategy` method allows changing the strategy at runtime. The `checkout` method uses the strategy to process the payment.
4. **Client Code**: Demonstrates adding items to the shopping cart and checking out using different payment strategies.

### Another Example: Sorting Algorithms

Here is another example where the strategy pattern is used to implement different sorting algorithms.

```python
from abc import ABC, abstractmethod

# Strategy Interface
class SortStrategy(ABC):
    @abstractmethod
    def sort(self, data):
        pass

# Concrete Strategies
class BubbleSort(SortStrategy):
    def sort(self, data):
        n = len(data)
        for i in range(n):
            for j in range(0, n-i-1):
                if data[j] > data[j+1]:
                    data[j], data[j+1] = data[j+1], data[j]
        return data

class QuickSort(SortStrategy):
    def sort(self, data):
        if len(data) <= 1:
            return data
        pivot = data[len(data) // 2]
        left = [x for x in data if x < pivot]
        middle = [x for x in data if x == pivot]
        right = [x for x in data if x > pivot]
        return self.sort(left) + middle + self.sort(right)

class MergeSort(SortStrategy):
    def sort(self, data):
        if len(data) <= 1:
            return data

        def merge(left, right):
            result = []
            i = j = 0

            while i < len(left) and j < len(right):
                if left[i] < right[j]:
                    result.append(left[i])
                    i += 1
                else:
                    result.append(right[j])
                    j += 1

            result.extend(left[i:])
            result.extend(right[j:])
            return result

        middle = len(data) // 2
        left = self.sort(data[:middle])
        right = self.sort(data[middle:])
        return merge(left, right)

# Context
class SortContext:
    def __init__(self, strategy: SortStrategy):
        self.strategy = strategy

    def set_strategy(self, strategy: SortStrategy):
        self.strategy = strategy

    def sort(self, data):
        return self.strategy.sort(data)

# Client code
if __name__ == "__main__":
    data = [5, 2, 9, 1, 5, 6]

    context = SortContext(BubbleSort())
    print("BubbleSort:", context.sort(data.copy()))

    context.set_strategy(QuickSort())
    print("QuickSort:", context.sort(data.copy()))

    context.set_strategy(MergeSort())
    print("MergeSort:", context.sort(data.copy()))
```

### Explanation

1. **Strategy Interface**: `SortStrategy` defines a common interface for all sorting algorithms with the `sort` method.
2. **Concrete Strategies**:
    - `BubbleSort`: Implements bubble sort.
    - `QuickSort`: Implements quick sort.
    - `MergeSort`: Implements merge sort.
3. **Context**: `SortContext` maintains a reference to a `SortStrategy` object. The `set_strategy` method allows changing the strategy at runtime. The `sort` method uses the strategy to sort the data.
4. **Client Code**: Demonstrates sorting data using different sorting algorithms.

The Strategy pattern is very flexible and can be applied in various scenarios where multiple algorithms or behaviors need to be interchangeable at runtime. It promotes the Open/Closed Principle by allowing the addition of new strategies without modifying existing code.