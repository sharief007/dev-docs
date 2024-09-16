---
title: Distributed Unique ID Generator
type: docs
sidebar:
  open: true
toc: true
params:
  editURL: 
---

### Object-Oriented Design for Distributed ID Generation System (like Twitter's Snowflake ID)

The goal is to design a **Distributed ID Generation System** that can generate globally unique IDs across multiple services and datacenters without coordination. This system should be highly available, fault-tolerant, and able to generate IDs at high throughput. **Twitter's Snowflake ID** is a popular example, but we will generalize its concept.

### Requirements:
1. **Uniqueness**: The ID should be globally unique.
2. **Scalability**: The system should scale across multiple servers and datacenters.
3. **High throughput**: The system should be able to generate a large number of IDs per second.
4. **Time-ordering**: IDs should be roughly ordered based on the time they were generated.
5. **Fault-tolerance**: The system should be able to tolerate failures of individual servers.

### ID Structure:
We will structure the ID as a 64-bit integer, which consists of:
- **Timestamp** (in milliseconds): 41 bits (enough for ~69 years from the start time)
- **Datacenter ID**: 5 bits (32 datacenters)
- **Machine ID (or Worker ID)**: 5 bits (32 machines per datacenter)
- **Sequence number**: 12 bits (4096 IDs per millisecond per machine)

### Example ID Structure:
```
| 41 bits (timestamp) | 5 bits (datacenter ID) | 5 bits (machine ID) | 12 bits (sequence) |
```

### Object-Oriented Design:

#### 1. **IDGenerator Class**:
This is the main class responsible for generating unique IDs. It combines the timestamp, datacenter ID, machine ID, and sequence to form a unique ID.

```python
import time
from threading import Lock

class IDGenerator:
    def __init__(self, datacenter_id: int, machine_id: int, epoch: int):
        self.datacenter_id = datacenter_id
        self.machine_id = machine_id
        self.epoch = epoch  # Custom epoch, in milliseconds
        
        # Bit allocations
        self.timestamp_bits = 41
        self.datacenter_bits = 5
        self.machine_bits = 5
        self.sequence_bits = 12
        
        # Maximum values
        self.max_datacenter_id = (1 << self.datacenter_bits) - 1
        self.max_machine_id = (1 << self.machine_bits) - 1
        self.max_sequence = (1 << self.sequence_bits) - 1
        
        # Shifts for each component
        self.datacenter_shift = self.sequence_bits + self.machine_bits
        self.machine_shift = self.sequence_bits
        self.timestamp_shift = self.sequence_bits + self.machine_bits + self.datacenter_bits
        
        # State
        self.sequence = 0
        self.last_timestamp = -1
        self.lock = Lock()

        # Validation
        if not (0 <= datacenter_id <= self.max_datacenter_id):
            raise ValueError(f"Datacenter ID must be between 0 and {self.max_datacenter_id}")
        if not (0 <= machine_id <= self.max_machine_id):
            raise ValueError(f"Machine ID must be between 0 and {self.max_machine_id}")

    def _current_timestamp(self):
        return int(time.time() * 1000)  # Current time in milliseconds

    def _wait_for_next_millis(self, last_timestamp):
        # Wait for the next millisecond to avoid duplicate timestamps
        timestamp = self._current_timestamp()
        while timestamp <= last_timestamp:
            timestamp = self._current_timestamp()
        return timestamp

    def generate_id(self) -> int:
        with self.lock:
            timestamp = self._current_timestamp()
            
            if timestamp < self.last_timestamp:
                raise Exception("Clock moved backwards. Unable to generate ID.")
            
            if timestamp == self.last_timestamp:
                # Increment the sequence number
                self.sequence = (self.sequence + 1) & self.max_sequence
                if self.sequence == 0:
                    # Sequence overflow, wait for the next millisecond
                    timestamp = self._wait_for_next_millis(self.last_timestamp)
            else:
                # Reset sequence for a new millisecond
                self.sequence = 0
            
            self.last_timestamp = timestamp

            # Generate the ID
            id_value = ((timestamp - self.epoch) << self.timestamp_shift) | \
                       (self.datacenter_id << self.datacenter_shift) | \
                       (self.machine_id << self.machine_shift) | \
                       self.sequence
            return id_value
```

#### 2. **Datacenter Class**:
Represents a datacenter that manages multiple workers (machines). Each datacenter is assigned a unique ID.

```python
class Datacenter:
    def __init__(self, datacenter_id: int, workers: int, epoch: int):
        self.datacenter_id = datacenter_id
        self.workers = [IDGenerator(datacenter_id, machine_id, epoch) for machine_id in range(workers)]

    def get_worker(self, machine_id: int) -> IDGenerator:
        if machine_id < 0 or machine_id >= len(self.workers):
            raise ValueError("Invalid machine ID")
        return self.workers[machine_id]
```

#### 3. **System Class**:
This class manages multiple datacenters and facilitates the distribution of ID generators to various clients.

```python
class IDSystem:
    def __init__(self, datacenters: int, workers_per_datacenter: int, epoch: int):
        self.datacenters = [Datacenter(datacenter_id, workers_per_datacenter, epoch) for datacenter_id in range(datacenters)]

    def get_generator(self, datacenter_id: int, machine_id: int) -> IDGenerator:
        if datacenter_id < 0 or datacenter_id >= len(self.datacenters):
            raise ValueError("Invalid datacenter ID")
        return self.datacenters[datacenter_id].get_worker(machine_id)
```

### Key Classes and Their Responsibilities:
1. **`IDGenerator`**: Generates a globally unique ID based on the timestamp, datacenter ID, machine ID, and sequence number. It ensures that no two IDs generated by the same machine in the same millisecond are the same.
2. **`Datacenter`**: Represents a datacenter containing multiple machines. It manages the ID generation on those machines.
3. **`IDSystem`**: Represents the entire distributed ID generation system. It manages multiple datacenters and facilitates the retrieval of ID generators for different machines.

### Design Details:

- **Thread Safety**: The `generate_id` method in the `IDGenerator` class is thread-safe using a lock to ensure no race conditions when multiple threads try to generate an ID concurrently.
- **Fault-tolerance**: Since each machine independently generates IDs, even if some machines fail, others continue to function without coordination.
- **Scalability**: This design scales horizontally by adding more datacenters and machines. The system supports up to 32 datacenters and 32 machines per datacenter.
- **Time-ordering**: The 41-bit timestamp ensures that IDs are roughly time-ordered.

### Usage Example:

```python
if __name__ == "__main__":
    epoch = 1609459200000  # Custom epoch (Jan 1, 2021, 00:00:00 UTC)
    
    # Initialize a system with 3 datacenters, each having 5 machines
    system = IDSystem(datacenters=3, workers_per_datacenter=5, epoch=epoch)

    # Get the ID generator for datacenter 1, machine 2
    id_generator = system.get_generator(datacenter_id=1, machine_id=2)

    # Generate an ID
    unique_id = id_generator.generate_id()
    print(f"Generated ID: {unique_id}")
```

### Further Considerations:
1. **Clock synchronization**: If clocks across machines are not synchronized, we may need mechanisms to detect and handle clock drifts (e.g., retries or system-wide time synchronization like NTP).
2. **Scaling limits**: With 5 bits for datacenter and machine IDs, the system is limited to 32 datacenters and 32 machines per datacenter. If more machines/datacenters are required, the bit allocation can be adjusted (e.g., reducing sequence bits).

This design is based on **Snowflake's** principles but is generalized for broader use cases.