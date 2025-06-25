---
title: 'Leaking Bucket Algorithm'
weight: 4
type: docs
toc: true
prev: 
next: 
params:
  editURL:
---

The **Sliding Window Log** algorithm is a rate-limiting mechanism that uses a log of timestamps to track and evaluate requests. Each incoming request's timestamp is recorded in a data structure, and only requests within the current window are considered.  
 
The Sliding Window Log algorithm is designed to address issues with **Fixed Window** rate limiting, such as request bursts at the boundary of time windows.

### Working:
1. **Track Timestamps**: For each request, store its timestamp in a list or queue.
2. **Filter Old Entries**: Remove timestamps that fall outside the current window (e.g., requests older than the window duration).
3. **Evaluate Request**: Allow or deny the request based on whether the number of requests within the window exceeds the defined limit.

### Pros:
- **Precise Limiting**: Tracks exact timestamps, offering fine-grained control over request rates.  
- **No Boundary Issues**: Avoids the sudden allowance of burst requests at window boundaries.  

### Cons:
- **Memory Usage**: Requires storing timestamps for all requests within the window, which can grow large for high traffic.  
- **Performance**: Filtering old timestamps can be computationally expensive.  

### Sample Algorithm Implementation in Python:

```python
from collections import deque
import time

class SlidingWindowLog:
    def __init__(self, max_requests, window_size):
        """
        Initialize the sliding window log rate limiter.
        :param max_requests: Maximum allowed requests within the time window.
        :param window_size: Time window size in seconds.
        """
        self.max_requests = max_requests
        self.window_size = window_size
        self.timestamps = deque()  # Deque to store timestamps of requests
    
    def allow_request(self):
        """
        Determines if a request is allowed.
        :return: True if the request is allowed, False otherwise.
        """
        current_time = time.time()
        
        # Remove timestamps outside the current window
        while self.timestamps and self.timestamps[0] < current_time - self.window_size:
            self.timestamps.popleft()
        
        if len(self.timestamps) < self.max_requests:
            # Allow request and add timestamp
            self.timestamps.append(current_time)
            return True

        # Deny request
        return False

# Example Usage
rate_limiter = SlidingWindowLog(max_requests=5, window_size=10)  # 5 requests per 10 seconds

# Simulate requests
for i in range(7):
    print(f"Request {i + 1}: {'Allowed' if rate_limiter.allow_request() else 'Denied'}")
    time.sleep(1)
```
