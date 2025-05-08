---
title: Blocking Queue
type: docs
sidebar:
  open: true
toc: true
params:
  editURL: 
---

{{< tabs items="python" >}}
    {{< tab >}}
    ```python

from threading import Lock, Condition
from typing import Generic, TypeVar, List

T = TypeVar("T")  # Generic type for items in the queue

class BlockingQueue(Generic[T]):
    """
    A thread-safe blocking queue implementation.
    """

    def __init__(self, max_size: int):
        """
        Initialize the blocking queue.

        :param max_size: Maximum size of the queue. If 0, the queue is unbounded.
        """
        self._queue: List[T] = []
        self._max_size = max_size
        self._lock = Lock()
        self._not_empty = Condition(self._lock)
        self._not_full = Condition(self._lock)

    def enqueue(self, item: T) -> None:
        """
        Add an item to the queue. Blocks if the queue is full.

        :param item: The item to add to the queue.
        """
        with self._not_full:
            while self._max_size > 0 and len(self._queue) >= self._max_size:
                self._not_full.wait()  # Wait until space is available
            self._queue.append(item)
            self._not_empty.notify()  # Notify threads waiting to dequeue

    def dequeue(self) -> T:
        """
        Remove and return an item from the queue. Blocks if the queue is empty.

        :return: The item removed from the queue.
        """
        with self._not_empty:
            while not self._queue:
                self._not_empty.wait()  # Wait until an item is available
            item = self._queue.pop(0)
            self._not_full.notify()  # Notify threads waiting to enqueue
            return item

    def size(self) -> int:
        """
        Get the current size of the queue.

        :return: The number of items in the queue.
        """
        with self._lock:
            return len(self._queue)

    def is_empty(self) -> bool:
        """
        Check if the queue is empty.

        :return: True if the queue is empty, False otherwise.
        """
        with self._lock:
            return not self._queue

    def is_full(self) -> bool:
        """
        Check if the queue is full.

        :return: True if the queue is full, False otherwise.
        """
        with self._lock:
            return self._max_size > 0 and len(self._queue) >= self._max_size

    ```
    {{</ tab >}}
{{</ tabs >}}