---
title: Search Framework
type: docs
sidebar:
  open: true
toc: true
params:
  editURL: 
---

### Abstract Storage

```python
from abc import ABC, abstractmethod
from typing import Any, Iterable, Optional

class Entity:
    __slots__ = ['id']
    
    def __init__(self):
        self.id: Optional[int] = None


class Storage(ABC):
    @abstractmethod
    def save(self, record: Entity) -> int:
        pass

    @abstractmethod
    def read(self, id: int) -> Entity:
        pass

    @abstractmethod
    def update(self, id: int, record: Entity) -> Entity:
        pass

    @abstractmethod
    def delete(self, id: int) -> bool:
        pass

    @abstractmethod
    def get_all(self) -> Iterable[Entity]:
        pass


class Indexer(ABC):
    @abstractmethod
    def create_index(self, attr_name: str):
        pass

    @abstractmethod
    def update_index(self, record: Entity):
        pass

    @abstractmethod
    def remove_from_index(self, record: Entity):
        pass

    @abstractmethod
    def get_by_index(self, attr_name: str, attr_value: Any) -> Iterable[Entity]:
        pass
```

### Concrete Storage Implementation

```python
import itertools
from threading import RLock
from typing import Any, Iterable, Optional
from weakref import WeakValueDictionary

from search.persistence.storage import Indexer, Storage, Entity

class InMemoryIndexer(Indexer):
    def __init__(self, *attr_names: str):
        self._indexes = {}  # A dictionary to store attribute-based indexes
        self._lock = RLock()

        for attr_name in attr_names:
            self.create_index(attr_name)

    def create_index(self, attr_name: str):
        """Create an index for a given attribute name if it doesn't exist."""
        with self._lock:
            if attr_name not in self._indexes:
                self._indexes[attr_name] = {}

    def update_index(self, record: Entity):
        """Update the indexes when a record is added or updated."""
        with self._lock:
            for attr_name, index in self._indexes.items():
                attr_value = getattr(record, attr_name, None)
                if attr_value is not None:
                    if attr_value not in index:
                        index[attr_value] = set()
                    index[attr_value].add(record.id)

    def remove_from_index(self, record: Entity):
        """Remove a record from indexes when it is deleted."""
        with self._lock:
            for attr_name, index in self._indexes.items():
                attr_value = getattr(record, attr_name, None)
                if attr_value in index:
                    index[attr_value].discard(record.id)

    def get_by_index(self, attr_name: str, attr_value: Any) -> Iterable[Entity]:
        """Retrieve records efficiently by using an index."""
        with self._lock:
            if attr_name not in self._indexes or attr_value not in self._indexes[attr_name]:
                return iter([])  # Return an empty Iterable if the attribute is not indexed or not found
            return self._indexes[attr_name][attr_value]


class InMemoryStorage(Storage):
    def __init__(self):
        self._container = WeakValueDictionary()
        self._lock = RLock()
        self._id_generator = itertools.count(start=1) 

    def save(self, record: Entity) -> int:
        with self._lock:
            record.id = next(self._id_generator)
            self._container[record.id] = record

        return record.id
    
    def read(self, id: int) -> Optional[Entity]:
        with self._lock:
            if id not in self._container:
                raise ValueError(f"Record with id {id} not found")
            return self._container[id]
    
    def update(self, id: int, record: Entity) -> Entity:
        with self._lock:
            if id not in self._container:
                raise ValueError(f"Cannot update. Record with id {id} not found")
            self._container[id] = record
            return record
    
    def delete(self, id: int) -> bool:
        with self._lock:
            return self._container.pop(id, None) is not None
    
    def get_all(self) -> Iterable[Entity]:
        with self._lock:
            return iter(self._container.values())


class StorageWithIndex(InMemoryStorage, InMemoryIndexer):
    def __init__(self, *attr_names: str):
        for attr_name in attr_names:
            self.create_index(attr_name)

    def save(self, record: Entity) -> int:
        record_id = super().save(record)
        self.update_index(record)
        return record_id

    def update(self, id: int, record: Entity) -> bool:
        success = super().update(id, record)
        if success:
            self.update_index(record)
        return success

    def delete(self, id: int) -> bool:
        try:
            record = super().read(id)
            if not super().delete(id):
                return False
            self.remove_from_index(record)
        except ValueError:
            return False
        return True
```

### Query Utils

```python
from abc import ABC, abstractmethod
from typing import Any
from enum import Enum, auto

from search.persistence.storage import Entity

class Operator(Enum):
    EQ = auto()
    GT = auto()
    GTE = auto()
    LT = auto()
    LTE = auto()

    def apply(self, op1: Any, op2: Any) -> bool:
        match self:
            case Operator.EQ:
                return op1 == op2
            case Operator.GT:
                return op1 > op2
            case Operator.GTE:
                return op1 >= op2
            case Operator.LT:
                return op1 < op2
            case Operator.LTE:
                return op1 <= op2
            case _:
                raise ValueError(f"Unsupported operator: {self}")


class Query(ABC):
    @abstractmethod
    def evaluate(self, record: Entity) -> bool:
        pass


class Predicate(Query):
    def __init__(self, name: str, value: Any, operator: Operator):
        self.attr_name = name
        self.attr_value = value
        self.operator = operator
    
    def evaluate(self, record: Entity) -> bool:
        record_value = getattr(record, self.attr_name, None)
        if record_value is None:
            return False
        return self.operator.apply(record_value, self.attr_value)


class CompositeQuery(Query):
    def __init__(self, *predicates: Predicate):
        self.predicates = predicates


class And(CompositeQuery):
    def evaluate(self, record: Entity) -> bool:
        return all(predicate.evaluate(record) for predicate in self.predicates)


class Or(CompositeQuery):
    def evaluate(self, record: Entity) -> bool:
        return any(predicate.evaluate(record) for predicate in self.predicates)
```


### Search Engine

```python
import itertools
from typing import Any, Callable, Iterable
from search.persistence.storage import Entity, Storage
from search.query.criteria import Query


class Paginator:
    def __init__(self, records: Iterable[Entity], page_size: int):
        self.records = records
        self.page_size = page_size

    def get_page(self, page_number: int) -> Iterable[Entity]:
        start = (page_number - 1) * self.page_size
        end = start + self.page_size
        return itertools.islice(self.records, start, end)


class SearchEngine:
    def __init__(self, storage: Storage):
        self.storage = storage

    def filter(self, query: Query) -> Iterable[Entity]:
        return filter(lambda rec: query.evaluate(rec), self.storage.get_all())

    # def indexed_filter(self, attr_name: str, attr_value: Any) -> Iterable[Entity]:
    #     return self.storage.get_by_index(attr_name, attr_value)

    def sort(self, records: Iterable[Entity], key: Callable[[Entity], Any], reverse: bool = False) -> Iterable[Entity]:
        return sorted(records, key=key, reverse=reverse)

    def page(self, records: Iterable[Entity], page_size: int, page_no: int) -> Iterable[Entity]:
        paginator = Paginator(records, page_size)
        return paginator.get_page(page_no)
```