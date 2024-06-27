---
title: Iterator
type: docs
sidebar:
  open: true
toc: true
params:
  editURL: 
---

The Iterator design pattern is a behavioral design pattern that provides a way to access the elements of an aggregate object sequentially without exposing its underlying representation. This pattern is particularly useful for collections like lists, trees, and other complex data structures.

```python
from typing import Any, Optional
from abc import ABC, abstractmethod

# Node class for the binary tree
class Node:
    def __init__(self, value: Any):
        self.value = value
        self.left: Optional[Node] = None
        self.right: Optional[Node] = None

# Iterator Interface
class Iterator(ABC):
    @abstractmethod
    def __next__(self) -> Any:
        pass

    @abstractmethod
    def __iter__(self) -> 'Iterator':
        pass

# Concrete Iterator for in-order traversal
class InOrderIterator(Iterator):
    def __init__(self, root: Optional[Node]):
        self.stack = []
        self._push_left(root)

    def _push_left(self, node: Optional[Node]):
        while node:
            self.stack.append(node)
            node = node.left

    def __next__(self) -> Any:
        if not self.stack:
            raise StopIteration

        node = self.stack.pop()
        value = node.value
        if node.right:
            self._push_left(node.right)

        return value

    def __iter__(self) -> 'InOrderIterator':
        return self

# Aggregate Interface
class Aggregate(ABC):
    @abstractmethod
    def create_iterator(self) -> Iterator:
        pass

# Concrete Aggregate for the binary tree
class BinaryTree(Aggregate):
    def __init__(self, root: Optional[Node]):
        self.root = root

    def create_iterator(self) -> Iterator:
        return InOrderIterator(self.root)

# Client code
if __name__ == "__main__":
    # Create a binary tree
    root = Node(4)
    root.left = Node(2)
    root.right = Node(6)
    root.left.left = Node(1)
    root.left.right = Node(3)
    root.right.left = Node(5)
    root.right.right = Node(7)

    tree = BinaryTree(root)
    iterator = tree.create_iterator()

    for value in iterator:
        print(value)

```