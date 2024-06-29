---
title: Visitor
type: docs
sidebar:
  open: true
toc: true
params:
  editURL: 
---

The Visitor design pattern is a behavioral design pattern that allows you to add further operations to objects without modifying them. This pattern follows the open/closed principle, allowing you to add new functionality to classes without changing their structure. It involves four main components:

1. **Visitor**: An interface or abstract class for all visitors. It declares a visit method for each type of element that can be visited.
2. **Concrete Visitor**: Classes that implement the Visitor interface. They provide the implementation for the visit methods.
3. **Element**: An interface or abstract class for the elements that can be visited. It declares an accept method that takes a visitor as an argument.
4. **Concrete Element**: Classes that implement the Element interface. They provide the implementation for the accept method.

### When to Use

- When you need to perform many unrelated operations on objects without modifying their classes.
- When the object structure is fixed, and new operations need to be added frequently.
- When extending functionality by adding new methods to existing classes would be impractical.

### Example Implementation in Python

Let's create an example where we have a set of different shapes (Circle and Rectangle), and we want to perform various operations on them, such as calculating the area and drawing them, without modifying the shape classes.

```python
from abc import ABC, abstractmethod
from math import pi

# Visitor interface
class Visitor(ABC):
    @abstractmethod
    def visit_circle(self, circle):
        pass

    @abstractmethod
    def visit_rectangle(self, rectangle):
        pass

# Concrete Visitor for calculating the area
class AreaVisitor(Visitor):
    def visit_circle(self, circle):
        return pi * (circle.radius ** 2)

    def visit_rectangle(self, rectangle):
        return rectangle.width * rectangle.height

# Concrete Visitor for drawing shapes
class DrawVisitor(Visitor):
    def visit_circle(self, circle):
        print(f"Drawing a circle with radius {circle.radius}")

    def visit_rectangle(self, rectangle):
        print(f"Drawing a rectangle with width {rectangle.width} and height {rectangle.height}")

# Element interface
class Shape(ABC):
    @abstractmethod
    def accept(self, visitor):
        pass

# Concrete Element for Circle
class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius

    def accept(self, visitor):
        return visitor.visit_circle(self)

# Concrete Element for Rectangle
class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def accept(self, visitor):
        return visitor.visit_rectangle(self)

# Client code
if __name__ == "__main__":
    shapes = [
        Circle(5),
        Rectangle(3, 4)
    ]

    area_visitor = AreaVisitor()
    draw_visitor = DrawVisitor()

    for shape in shapes:
        area = shape.accept(area_visitor)
        print(f"The area is {area}")
        shape.accept(draw_visitor)
```

### 1. File System Operations
In a file system, you have different types of files (e.g., text files, images, videos) and you may want to perform various operations on these files, such as compression, encryption, or displaying properties. The Visitor pattern allows you to add these operations without modifying the file classes.

```python
from abc import ABC, abstractmethod

# Visitor interface
class FileVisitor(ABC):
    @abstractmethod
    def visit_text_file(self, text_file):
        pass

    @abstractmethod
    def visit_image_file(self, image_file):
        pass

    @abstractmethod
    def visit_video_file(self, video_file):
        pass

# Concrete Visitor for displaying properties
class PropertiesVisitor(FileVisitor):
    def visit_text_file(self, text_file):
        print(f"Text file: {text_file.name}, Size: {text_file.size} KB")

    def visit_image_file(self, image_file):
        print(f"Image file: {image_file.name}, Resolution: {image_file.resolution}")

    def visit_video_file(self, video_file):
        print(f"Video file: {video_file.name}, Duration: {video_file.duration} minutes")

# File interface
class File(ABC):
    @abstractmethod
    def accept(self, visitor):
        pass

# Concrete Elements
class TextFile(File):
    def __init__(self, name, size):
        self.name = name
        self.size = size

    def accept(self, visitor):
        visitor.visit_text_file(self)

class ImageFile(File):
    def __init__(self, name, resolution):
        self.name = name
        self.resolution = resolution

    def accept(self, visitor):
        visitor.visit_image_file(self)

class VideoFile(File):
    def __init__(self, name, duration):
        self.name = name
        self.duration = duration

    def accept(self, visitor):
        visitor.visit_video_file(self)

# Client code
if __name__ == "__main__":
    files = [
        TextFile("Document.txt", 120),
        ImageFile("Picture.jpg", "1920x1080"),
        VideoFile("Movie.mp4", 120)
    ]

    properties_visitor = PropertiesVisitor()

    for file in files:
        file.accept(properties_visitor)
```

### 2. Shopping Cart with Different Discount Strategies
In a shopping cart system, you might have different types of items like electronics, groceries, and clothing, each of which might have different discount strategies. The Visitor pattern allows you to apply these discounts without modifying the item classes.

```python
from abc import ABC, abstractmethod

# Visitor interface
class DiscountVisitor(ABC):
    @abstractmethod
    def visit_electronics(self, electronics):
        pass

    @abstractmethod
    def visit_groceries(self, groceries):
        pass

    @abstractmethod
    def visit_clothing(self, clothing):
        pass

# Concrete Visitor for calculating discounts
class PercentageDiscountVisitor(DiscountVisitor):
    def visit_electronics(self, electronics):
        return electronics.price * 0.9  # 10% discount

    def visit_groceries(self, groceries):
        return groceries.price * 0.95  # 5% discount

    def visit_clothing(self, clothing):
        return clothing.price * 0.8  # 20% discount

# Item interface
class Item(ABC):
    @abstractmethod
    def accept(self, visitor):
        pass

# Concrete Elements
class Electronics(Item):
    def __init__(self, name, price):
        self.name = name
        self.price = price

    def accept(self, visitor):
        return visitor.visit_electronics(self)

class Groceries(Item):
    def __init__(self, name, price):
        self.name = name
        self.price = price

    def accept(self, visitor):
        return visitor.visit_groceries(self)

class Clothing(Item):
    def __init__(self, name, price):
        self.name = name
        self.price = price

    def accept(self, visitor):
        return visitor.visit_clothing(self)

# Client code
if __name__ == "__main__":
    cart = [
        Electronics("Laptop", 1000),
        Groceries("Apple", 2),
        Clothing("Jacket", 100)
    ]

    discount_visitor = PercentageDiscountVisitor()

    for item in cart:
        discounted_price = item.accept(discount_visitor)
        print(f"Discounted price for {item.name}: {discounted_price}")
```

### 3. Syntax Tree Evaluation
In a compiler or interpreter, you often work with abstract syntax trees (ASTs) representing different language constructs. The Visitor pattern can be used to traverse and evaluate these trees.

```python
from abc import ABC, abstractmethod

# Visitor interface
class NodeVisitor(ABC):
    @abstractmethod
    def visit_number_node(self, number_node):
        pass

    @abstractmethod
    def visit_add_node(self, add_node):
        pass

    @abstractmethod
    def visit_subtract_node(self, subtract_node):
        pass

# Concrete Visitor for evaluating the expression
class Evaluator(NodeVisitor):
    def visit_number_node(self, number_node):
        return number_node.value

    def visit_add_node(self, add_node):
        return add_node.left.accept(self) + add_node.right.accept(self)

    def visit_subtract_node(self, subtract_node):
        return subtract_node.left.accept(self) - subtract_node.right.accept(self)

# Node interface
class Node(ABC):
    @abstractmethod
    def accept(self, visitor):
        pass

# Concrete Elements
class NumberNode(Node):
    def __init__(self, value):
        self.value = value

    def accept(self, visitor):
        return visitor.visit_number_node(self)

class AddNode(Node):
    def __init__(self, left, right):
        self.left = left
        self.right = right

    def accept(self, visitor):
        return visitor.visit_add_node(self)

class SubtractNode(Node):
    def __init__(self, left, right):
        self.left = left
        self.right = right

    def accept(self, visitor):
        return visitor.visit_subtract_node(self)

# Client code
if __name__ == "__main__":
    # Constructing an AST for the expression (5 + 3) - 2
    tree = SubtractNode(
        AddNode(
            NumberNode(5),
            NumberNode(3)
        ),
        NumberNode(2)
    )

    evaluator = Evaluator()
    result = tree.accept(evaluator)
    print(f"The result of the expression is: {result}")
```

### Explanation of Examples

1. **File System Operations**: Different file types (text, image, video) can have various operations performed on them (displaying properties). The Visitor pattern allows adding these operations without modifying the file classes.
2. **Shopping Cart with Different Discount Strategies**: Different item types (electronics, groceries, clothing) can have different discount strategies. The Visitor pattern allows applying these strategies without modifying the item classes.
3. **Syntax Tree Evaluation**: Different nodes in an abstract syntax tree (number, add, subtract) can be evaluated. The Visitor pattern allows adding new operations on the syntax tree nodes without modifying their structure.

### Benefits

- **Extensibility**: New operations can be added without changing the object structure.
- **Single Responsibility Principle**: Different operations are separated into different classes.
- **Open/Closed Principle**: Classes are open for extension but closed for modification.

### Drawbacks

- **Complexity**: Adding a new type of element requires adding a new visit method to the Visitor interface and all its concrete implementations.
- **Coupling**: The pattern introduces tight coupling between the Element and Visitor hierarchies.

The Visitor design pattern is particularly useful in scenarios where you need to perform various unrelated operations on a set of objects with a stable structure, making it a powerful tool for extending functionality in a clean and maintainable way.
