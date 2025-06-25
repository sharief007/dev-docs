---
title: 'Token Bucket Algorithm'
weight: 1
type: docs
toc: true
prev: 
next: 
params:
  editURL:
---

The **Token Bucket** algorithm is used to control the rate of requests, allowing bursts up to a certain limit while maintaining a steady rate over time.

#### **How It Works**:
1. **Token Bucket Setup**:
   - The bucket has a **predefined capacity** (e.g., 4 tokens).
   - Tokens are added to the bucket at a **preset rate** (e.g., 2 tokens per second).
   - Once the bucket is full, additional tokens overflow.

2. **Request Handling**:
   - Each request consumes **1 token**.
   - If enough tokens are available, 1 token is removed for each request, and the request proceeds.
   - If there are not enough tokens, the request is **dropped**.

![Token Bucket](/dev-docs/rate-limiting/token-bucket.png)

#### **Key Parameters**:
- **Bucket Size**: Maximum number of tokens the bucket can hold.
- **Refill Rate**: Number of tokens added to the bucket per second.

#### **Use Case Examples**:

- **Per API Endpoint**: Different buckets are needed for each API endpoint with varying request limits (e.g., 1 post per second, 150 friends per day).
- **Per IP Address**: Each IP address could have its own bucket for rate limiting.
- **Global Bucket**: A shared bucket could be used if there’s a global rate limit, like 10,000 requests per second for all users.

By adjusting the bucket size and refill rate, the system can control both burst capacity and steady traffic flow.

### Sample Implementation

```python
import time

class TokenBucket:
    def __init__(self, bucket_size, refill_rate):
        """
        :param bucket_size: Maximum capacity of the token bucket.
        :param refill_rate: Number of tokens added per second.
        """
        self.bucket_size = bucket_size
        self.refill_rate = refill_rate

        self.tokens = bucket_size  # Start with the bucket full
        self.last_refill_time = time.time()

    def refill(self):
        """
        Refill the token bucket based on the refill rate.
        This is called periodically to ensure tokens are added over time.
        """
        current_time = time.time()
        elapsed_time = current_time - self.last_refill_time
        
        # Calculate how many tokens to add based on the elapsed time
        tokens_to_add = elapsed_time * self.refill_rate
        if tokens_to_add > 0:
            self.tokens = min(self.bucket_size, self.tokens + tokens_to_add)
            self.last_refill_time = current_time

    def request(self):
        """
        Process a request. If a token is available, consume one and allow the request.
        If no tokens are available, deny the request.
        
        :return: True if request is allowed, False if denied.
        """
        self.refill()
        
        if self.tokens >= 1:
            self.tokens -= 1  # Consume a token
            return True  # Request allowed
        else:
            return False  # Request denied due to lack of tokens

# Example usage
if __name__ == "__main__":
    # Create a token bucket with capacity of 4 tokens, refilling at 2 tokens per second
    bucket = TokenBucket(bucket_size=4, refill_rate=2)

    # Simulate requests
    for i in range(10):
        if bucket.request():
            print(f"Request {i + 1}: Allowed")
        else:
            print(f"Request {i + 1}: Denied")
        
        time.sleep(0.5)  # Wait for 0.5 seconds between requests

```