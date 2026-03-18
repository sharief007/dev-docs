---
title: 'Leaking Bucket Algorithm'
weight: 2
type: docs
toc: true
prev: 
next: 
params:
  editURL:
---

The **Leaking Bucket** algorithm is used to control the rate of requests with a fixed processing rate. It ensures that requests are handled at a constant rate, regardless of traffic spikes. It works similarly to the Token Bucket but differs in how requests are processed.

#### **How It Works**:
1. **Request Handling**:
   - When a request arrives, the system checks if the **queue** (bucket) is full.
   - If the queue is **not full**, the request is added to the queue.
   - If the queue is **full**, the request is **dropped**.

2. **Request Processing**:
   - Requests are pulled from the queue and processed at a **fixed rate** (outflow rate).

#### **Key Parameters**:
- **Bucket Size**: The maximum capacity of the queue, which holds the requests to be processed at a fixed rate.
- **Outflow Rate**: The rate at which requests are processed from the queue (e.g., 1 request per second).

#### **Pros**:
- **Memory Efficient**: The fixed queue size ensures that memory usage is controlled.
- **Fixed Outflow Rate**: Suitable for systems requiring a consistent, predictable processing rate.

#### **Cons**:
- **Traffic Bursts**: If there is a sudden burst of traffic, the queue may fill up quickly, causing **older requests** to be dropped, while **newer requests** are rate-limited.
- **Tuning Difficulty**: Properly tuning the **bucket size** and **outflow rate** can be challenging to balance different use cases.


#### **Use Case Examples**:

| **Use Case**            | **Bucket**        | **Bucket Size**                       | **Leak Rate**                         |
|-------------------------|-------------------|---------------------------------------|---------------------------------------|
| **Per API**             | Separate buckets for each API | Defines burst size for each API | Fixed rate of requests processed for each API |
| **Per User/IP Address** | Separate buckets for each user/IP | Defines burst size for each user/IP | Fixed rate of requests processed for each user/IP |
| **Global**              | Single global bucket | Defines burst size for all traffic | Fixed rate of requests processed globally |

### Sample Implementation

```python
import time
from collections import deque

class LeakyBucket:
    def __init__(self, bucket_size, leak_rate):
        """
        Initializes the leaky bucket.
        
        :param bucket_size: Maximum capacity of the bucket (queue size).
        :param leak_rate: Number of requests processed per second (leak rate).
        """
        self.bucket_size = bucket_size
        self.leak_rate = leak_rate
        self.tokens = 0  # Current tokens in the bucket
        self.last_refill_time = time.time()
        self.queue = deque()  # Queue to hold incoming requests
    
    def refill(self):
        """
        Refill the bucket based on the leak rate.
        The bucket leaks at a fixed rate over time.
        """
        current_time = time.time()
        elapsed_time = current_time - self.last_refill_time
        
        # Calculate how many tokens to leak based on elapsed time
        tokens_to_leak = int(elapsed_time * self.leak_rate)
        
        if tokens_to_leak > 0:
            # Leak the tokens
            self.tokens = max(0, self.tokens - tokens_to_leak)
            self.last_refill_time = current_time
    
    def handle_request(self):
        """
        Process a request. If the bucket has space, the request is allowed.
        If the bucket is full, the request is denied.
        """
        self.refill()

        if len(self.queue) < self.bucket_size:
            # Add request to the queue (i.e., in the bucket)
            self.queue.append(time.time())
            return True  # Request allowed
        else:
            # Drop the request as the bucket is full
            return False


class RateLimiter:
    def __init__(self, bucket_size, leak_rate, mode='global'):
        """
        Initializes the rate limiter.
        
        :param bucket_size: Maximum capacity of the bucket (queue size).
        :param leak_rate: Number of requests processed per second.
        :param mode: Mode of rate limiting ('global', 'per_user', 'per_api').
        """
        self.bucket_size = bucket_size
        self.leak_rate = leak_rate
        self.mode = mode
        self.buckets = {}

    def get_bucket_key(self, identifier):
        """
        Generate the key for each rate-limited bucket based on the mode.
        
        :param identifier: Unique identifier (API, user, IP).
        """
        if self.mode == 'global':
            return 'global'
        elif self.mode == 'per_api':
            return identifier  # API endpoint as identifier
        elif self.mode == 'per_user':
            return identifier  # User or IP address as identifier
        else:
            raise ValueError("Unknown mode for rate limiter")

    def handle_request(self, identifier):
        """
        Handle an incoming request. Returns True if allowed, False if denied.
        
        :param identifier: Unique identifier (API, user, IP).
        """
        bucket_key = self.get_bucket_key(identifier)

        if bucket_key not in self.buckets:
            # Create a new bucket if it doesn't exist
            self.buckets[bucket_key] = LeakyBucket(self.bucket_size, self.leak_rate)

        bucket = self.buckets[bucket_key]
        return bucket.handle_request()


# Example usage
if __name__ == "__main__":
    # Create a global rate limiter with bucket size of 5 and leak rate of 1 request per second
    limiter = RateLimiter(bucket_size=5, leak_rate=1, mode='global')

    # Simulate requests from multiple sources
    for i in range(10):
        if limiter.handle_request('global'):
            print(f"Request {i + 1}: Allowed")
        else:
            print(f"Request {i + 1}: Denied")
        time.sleep(0.5)  # Wait for 0.5 seconds between requests

    # Per User example with different identifiers (IP or User ID)
    user_limiter = RateLimiter(bucket_size=3, leak_rate=1, mode='per_user')
    users = ['user1', 'user2', 'user1', 'user3', 'user2']

    for user in users:
        if user_limiter.handle_request(user):
            print(f"User '{user}': Request Allowed")
        else:
            print(f"User '{user}': Request Denied")
        time.sleep(0.5)  # Wait for 0.5 seconds between requests

    # Per API example
    api_limiter = RateLimiter(bucket_size=4, leak_rate=1, mode='per_api')
    apis = ['GET /home', 'POST /login', 'GET /home', 'POST /login', 'GET /home']

    for api in apis:
        if api_limiter.handle_request(api):
            print(f"API '{api}': Request Allowed")
        else:
            print(f"API '{api}': Request Denied")
        time.sleep(0.5)  # Wait for 0.5 seconds between requests

```