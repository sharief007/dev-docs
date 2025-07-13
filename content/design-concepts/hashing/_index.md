---
title: 'Hashing'
weight: 0
type: docs
toc: true
sidebar:
  open: true
prev: 
next: 
params:
  editURL: 
---


Hashing is a **deterministic transformation** of input data (key) into a fixed-size output (usually an integer or hex value). This is done through a **hash function**. The result, called a **hash code** or **digest**, ideally should:
- Be quick to compute
- Minimize collisions (different inputs mapping to same output)
- Be evenly distributed across the output space

Mathematically, if we have a hash function `h(x)`, then:
$$
h: X \to Y \quad \text{where } X \text{ is the set of inputs, and } Y \text{ is the range of hash values}
$$

For a hash table of size `m`, we often do:
$$
\text{index} = h(x) \mod m
$$

This lets us map the hash value to one of `m` buckets efficiently.


## 🧠 Popular Non-Cryptographic Hash Functions

These are typically used for data structures, databases, and indexing—not for security.

| Hash Function | Description | Use Case |
|---------------|-------------|----------|
| **MurmurHash** | High performance, non-cryptographic | Databases, bloom filters |
| **FNV (Fowler–Noll–Vo)** | Simple and fast | Hash tables in compilers |
| **CityHash / FarmHash** | Designed by Google for speed | Efficient large string hashing |
| **xxHash** | Insanely fast, great distribution | Real-time systems |


## 🔐 Cryptographic Hash Functions

These are designed with **security** in mind. Unlike standard hashes, cryptographic hashes must satisfy:

1. **Pre-image Resistance**: Given `H(x)`, it should be computationally hard to find `x`.
2. **Second Pre-image Resistance**: Given `x1`, hard to find `x2 != x1` such that `H(x1) = H(x2)`.
3. **Collision Resistance**: It's hard to find any two distinct inputs that hash to the same output.

### Popular Cryptographic Hashes:

| Hash Function | Output Size | Notes |
|---------------|-------------|-------|
| **SHA-256** | 256 bits | Standard for many security protocols |
| **SHA-3** | 224/256/384/512 bits | Latest secure hash family |
| **BLAKE3** | Variable | Fast and secure alternative to SHA-2 |

These are used in **password storage**, **digital signatures**, **blockchains**, etc.


## 🔍 Standard Hashing vs Cryptographic Hashing

| Feature              | Standard Hashing      | Cryptographic Hashing   |
|---------------------|------------------------|--------------------------|
| Purpose             | Fast data lookup       | Data integrity & security |
| Speed               | Very fast              | Slower (but secure)      |
| Collision Resistance| Low priority           | Critical                 |
| Reversibility       | Not a concern          | Must be irreversible     |
| Example Use Cases   | Caching, indexing      | Passwords, digital signatures |

**Do we need both?** Heck yes.

- You **don’t** want to hash your database keys with SHA-256—it’s overkill and slow.
- You **must** hash passwords with cryptographic hash functions or attackers could reverse them.

