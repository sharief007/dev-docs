---
title: Template
type: docs
sidebar:
  open: true
toc: true
params:
  editURL: 
---

The Template Method design pattern is a behavioral design pattern that defines the skeleton of an algorithm in a method, deferring some steps to subclasses. This pattern lets subclasses redefine certain steps of an algorithm without changing the algorithm's structure.

### Key Concepts

1. **Abstract Class**: Defines the template method and declares abstract operations that subclasses should implement.
2. **Concrete Classes**: Implement the abstract operations defined by the abstract class.

### Example in Python

Let's consider an example where we define a skeleton for cooking a dish. Different dishes will have their own specific steps, but the overall structure (prepare, cook, and serve) remains the same.

```python
from abc import ABC, abstractmethod

class Cooking(ABC):
    def cook(self):
        self.prepare_ingredients()
        self.cook_ingredients()
        self.serve()

    @abstractmethod
    def prepare_ingredients(self):
        pass

    @abstractmethod
    def cook_ingredients(self):
        pass

    @abstractmethod
    def serve(self):
        pass

class Pasta(Cooking):
    def prepare_ingredients(self):
        print("Preparing pasta, sauce, and cheese.")

    def cook_ingredients(self):
        print("Boiling pasta and cooking sauce.")

    def serve(self):
        print("Serving pasta with sauce and cheese.")

class Steak(Cooking):
    def prepare_ingredients(self):
        print("Preparing steak, salt, and pepper.")

    def cook_ingredients(self):
        print("Grilling steak.")

    def serve(self):
        print("Serving steak with a side of vegetables.")

# Client code
if __name__ == "__main__":
    print("Cooking Pasta:")
    pasta = Pasta()
    pasta.cook()

    print("\nCooking Steak:")
    steak = Steak()
    steak.cook()
```

### Another Example: Data Processing

Let's look at another example where the Template Method pattern is used for data processing. Different data sources (CSV and JSON) will have their own ways of reading and processing data, but the overall structure remains the same.

```python
from abc import ABC, abstractmethod
import json
import csv

class DataProcessor(ABC):
    def process(self):
        data = self.read_data()
        self.process_data(data)
        self.save_data(data)

    @abstractmethod
    def read_data(self):
        pass

    @abstractmethod
    def process_data(self, data):
        pass

    @abstractmethod
    def save_data(self, data):
        pass

class CSVDataProcessor(DataProcessor):
    def read_data(self):
        with open('data.csv', mode='r') as file:
            reader = csv.reader(file)
            data = [row for row in reader]
        return data

    def process_data(self, data):
        for row in data:
            row.append("processed")
        print("Processing CSV data.")

    def save_data(self, data):
        with open('processed_data.csv', mode='w', newline='') as file:
            writer = csv.writer(file)
            writer.writerows(data)
        print("Saving processed CSV data.")

class JSONDataProcessor(DataProcessor):
    def read_data(self):
        with open('data.json', mode='r') as file:
            data = json.load(file)
        return data

    def process_data(self, data):
        for item in data:
            item['status'] = "processed"
        print("Processing JSON data.")

    def save_data(self, data):
        with open('processed_data.json', mode='w') as file:
            json.dump(data, file, indent=4)
        print("Saving processed JSON data.")

# Client code
if __name__ == "__main__":
    print("Processing CSV Data:")
    csv_processor = CSVDataProcessor()
    csv_processor.process()

    print("\nProcessing JSON Data:")
    json_processor = JSONDataProcessor()
    json_processor.process()
```

### Benefits in Practice

- **Reuse and Extension**: Common steps are implemented in the abstract class, promoting reuse. New steps can be added or existing ones changed in concrete classes without affecting the overall algorithm.
- **Maintenance**: Changes to the algorithm's structure are made in one place (the abstract class), simplifying maintenance.
- **Consistent Processing**: Ensures a consistent processing structure across different concrete implementations.

The Template Method pattern is useful whenever you have an algorithm with a fixed structure but varying steps. It promotes code reuse, consistency, and flexibility, making it easier to maintain and extend.