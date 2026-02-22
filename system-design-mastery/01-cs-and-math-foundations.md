# L0: CS & Math Foundations

> Everything system design builds on. If gaps exist here, they will compound at every higher level.

---

## 1. Discrete Mathematics

### Logic & Proof Techniques
- Propositional logic, predicate logic
- Direct proof, proof by contradiction, proof by induction
- Strong induction (used in algorithm correctness proofs)

### Set Theory
- Sets, multisets, relations, functions
- Equivalence relations and partitions
- Cardinality (finite, countably infinite, uncountably infinite)

### Combinatorics
- Permutations and combinations
- Pigeonhole principle (critical for hash collision reasoning)
- Inclusion-exclusion principle
- Generating functions (used in algorithm analysis)

### Graph Theory
- Directed and undirected graphs, multigraphs, hypergraphs
- Trees, DAGs (directed acyclic graphs)
- Connectivity (strongly connected components, biconnected components)
- Planarity, graph coloring
- Network flow (max-flow min-cut theorem — used in load analysis)
- Eulerian and Hamiltonian paths/circuits
- Bipartite graphs (used in matching problems, scheduling)

### Information Theory
- Entropy: H(X) = -Σ p(x) log₂ p(x)
- Shannon's source coding theorem (compression limits)
- Channel capacity (Shannon-Hartley theorem)
- Mutual information, KL divergence (used heavily in ML)
- Minimum description length

---

## 2. Linear Algebra

> Essential for ML, computer graphics, recommendation systems, and PCA.

### Core Concepts
- Vectors and vector spaces, span, basis, linear independence
- Matrix operations (addition, multiplication, transpose, inverse)
- Dot product and its geometric interpretation
- Cross product (3D geometry)
- Norms (L1, L2, L∞, Frobenius)

### Matrix Decompositions
- LU decomposition (Gaussian elimination)
- QR decomposition (numerically stable solving)
- Eigenvalue decomposition: A = QΛQ⁻¹
- Singular Value Decomposition (SVD): A = UΣVᵀ
  - Used in: PCA, recommender systems, image compression, LSA
- Cholesky decomposition (positive definite matrices)
- Moore-Penrose pseudoinverse

### Applications to ML/Systems
- Principal Component Analysis (PCA) via SVD
- Dimensionality reduction
- Page rank as eigenvalue problem
- Attention mechanism as matrix products
- Transformations in computer graphics

---

## 3. Probability & Statistics

### Probability Theory
- Sample spaces, events, probability axioms
- Conditional probability: P(A|B) = P(A∩B)/P(B)
- Bayes' theorem: P(A|B) = P(B|A)P(A) / P(B)
- Independence vs. correlation
- Law of total probability, law of total expectation

### Random Variables & Distributions
- Discrete: Bernoulli, Binomial, Poisson, Geometric
- Continuous: Uniform, Gaussian (Normal), Exponential, Beta, Dirichlet
- Central Limit Theorem (CLT) — underpins A/B testing
- Moment generating functions, characteristic functions

### Statistical Inference
- Maximum Likelihood Estimation (MLE)
- Maximum A Posteriori (MAP) — Bayesian alternative
- Confidence intervals
- Hypothesis testing (t-test, chi-square test, KS test)
- p-values and their misuse
- Power of a test, Type I / Type II errors
- Multiple comparisons correction (Bonferroni, FDR)

### A/B Testing Methodology
- Randomization and control groups
- Sample size calculation
- Sequential testing (early stopping)
- Multi-variate testing (factorial designs)
- Novelty effects and network effects
- Metric selection (guardrail vs primary metrics)

---

## 4. Computational Complexity

### Asymptotic Analysis
- Big-O, Big-Ω, Big-Θ notations
- Common complexity classes: O(1), O(log n), O(n), O(n log n), O(n²), O(2ⁿ)
- Best/average/worst case analysis
- Amortized analysis (aggregate, accounting, potential methods)
  - Used in: dynamic arrays, union-find, splay trees

### Complexity Classes
- P (polynomial time)
- NP (nondeterministic polynomial time)
- NP-complete problems (3-SAT, Traveling Salesman, Knapsack)
- NP-hard (optimization versions)
- PSPACE, EXPTIME
- Why this matters: knowing intractability prevents over-engineering

### Space vs Time Tradeoffs
- Hash tables: O(n) space for O(1) time
- Precomputation / memoization
- Bloom filters: space for approximate membership
- Index structures: space for query speed

---

## 5. Data Structures

> Know not just the API but the internals and implementation tradeoffs.

### Linear Structures
- **Arrays**: cache-friendly, O(1) random access, O(n) insert/delete
- **Dynamic arrays**: amortized O(1) append (Java ArrayList, C++ vector)
- **Linked lists**: O(1) insert/delete with pointer, poor cache locality
  - Singly, doubly, circular, XOR linked list
- **Stacks**: LIFO — call stacks, DFS, expression parsing
- **Queues**: FIFO — BFS, scheduling
- **Deques** (double-ended queues): sliding window maximum in O(n)

### Trees
- **Binary Search Trees (BST)**: O(h) operations, can degrade to O(n)
- **AVL Trees**: height-balanced BST, O(log n) guaranteed
- **Red-Black Trees**: looser balance, faster inserts (used in Linux kernel, Java TreeMap)
- **B-Trees**: multi-way, disk-optimized, O(log_B n) I/Os
  - Used in: almost every RDBMS index
- **B+ Trees**: all data in leaves, linked leaves for range scans
  - Used in: InnoDB, PostgreSQL B-tree indexes
- **Segment Trees**: range queries with lazy propagation, O(log n)
- **Fenwick Trees (BIT)**: prefix sums, O(log n) update/query, simpler than segment tree
- **K-D Trees**: multi-dimensional spatial search, nearest neighbor
- **Interval Trees**: overlapping interval queries

### Heaps
- **Binary Heap**: min/max, O(log n) insert/extract, O(n) build
- **Fibonacci Heap**: O(1) amortized decrease-key, used in Dijkstra's optimal
- **Pairing Heap**: simpler Fibonacci alternative

### Hash-Based Structures
- **Hash Tables**: O(1) average — chaining vs open addressing
  - Load factor and rehashing
  - Robin Hood hashing (reduces probe variance)
  - Cuckoo hashing (worst-case O(1) lookup)
  - Linear probing (cache-friendly but clustering)
- **Bloom Filter**: probabilistic membership, no false negatives
  - Space: ~10 bits per element for 1% FPR
  - Used in: Cassandra, HBase, Chrome, CDNs
- **Counting Bloom Filter**: supports deletes
- **Cuckoo Filter**: better FPR than Bloom, supports deletes
- **HyperLogLog**: approximate distinct count, ~1.5% error with 1.5KB
  - Used in: Redis PFCOUNT, Google BigQuery, analytics
- **Count-Min Sketch**: approximate frequency counting
  - Used in: network monitoring, heavy hitter detection

### Advanced Structures
- **Trie (Prefix Tree)**: string prefix operations, O(m) where m=key length
- **Compressed Trie (Patricia Trie, Radix Tree)**: space-efficient trie
- **Suffix Array**: all suffixes sorted, O(n log n) build, O(m log n) search
- **Suffix Tree**: O(n) build with Ukkonen's, O(m) search — most powerful string structure
- **Aho-Corasick Automaton**: multi-pattern string matching in O(n + m + k)
- **Skip List**: probabilistic sorted structure, O(log n) average, used in Redis Sorted Sets
- **Disjoint Set Union (Union-Find)**: with path compression + union by rank = near-O(1)
  - Used in: Kruskal's MST, connected components, social network clusters
- **Rope**: string with fast concatenation and split (used in text editors)

---

## 6. Algorithms

### Sorting
- **Comparison sorts**: Quicksort (O(n log n) avg, O(n²) worst), Mergesort (O(n log n) stable), Heapsort (O(n log n) in-place)
- **Non-comparison sorts**: Counting sort O(n+k), Radix sort O(nk), Bucket sort O(n)
- **Timsort**: Python/Java default, adaptive mergesort, O(n) best case
- External sorting (merge sort with limited memory): used in database query execution

### Searching
- Binary search (and its variants: first/last occurrence, floor/ceil)
- Interpolation search (better than binary for uniform distributions)
- Exponential search (for unbounded arrays)
- Ternary search (unimodal functions)

### Graph Algorithms
- **Traversal**: BFS (shortest path unweighted), DFS (topological sort, SCC)
- **Shortest Paths**:
  - Dijkstra's: O((V+E) log V) with binary heap, non-negative weights
  - Bellman-Ford: O(VE), handles negative weights, detects negative cycles
  - Floyd-Warshall: O(V³) all-pairs
  - A*: heuristic-guided Dijkstra (used in maps, game AI)
  - Contraction Hierarchies (CH): used in GPS navigation for speed
- **Minimum Spanning Tree**: Kruskal's (sort edges), Prim's (grow from vertex)
- **Topological Sort**: DFS-based, Kahn's algorithm (BFS-based)
- **Strongly Connected Components**: Kosaraju's, Tarjan's
- **Biconnected Components and Articulation Points**: network reliability
- **Max Flow / Min Cut**: Ford-Fulkerson, Edmonds-Karp, Dinic's
  - Used in: network capacity, bipartite matching, image segmentation
- **Bipartite Matching**: Hungarian algorithm, Hopcroft-Karp

### Dynamic Programming
- Memoization vs tabulation
- Classic problems: LCS, LIS, Edit Distance, Knapsack, Matrix Chain
- DP on trees, DP on DAGs
- Bitmask DP (exponential state with bitmasks)
- DP optimization: convex hull trick, divide and conquer optimization

### String Algorithms
- **KMP (Knuth-Morris-Pratt)**: O(n+m) single pattern matching using failure function
- **Rabin-Karp**: rolling hash, O(n+m) average, good for multiple patterns
- **Z-algorithm**: O(n+m) pattern matching via Z-array
- **Boyer-Moore**: O(n/m) best case (used in grep, text editors)
- **Aho-Corasick**: O(n + m + k) multi-pattern (n=text, m=patterns total, k=matches)
- **Ukkonen's Algorithm**: O(n) suffix tree construction — most important string algorithm
- **Suffix Array with LCP**: O(n log n) or O(n) construction, space-efficient suffix tree alternative
- **Manacher's Algorithm**: O(n) longest palindromic substring

---

## 7. Computer Architecture

### CPU Internals
- **Instruction pipeline**: fetch → decode → execute → memory → writeback
- **Branch prediction**: static, dynamic (2-bit counters), TAGE predictors
- **Out-of-order execution** (Tomasulo's algorithm, register renaming)
- **Superscalar and VLIW**: multiple execution units per cycle
- **SIMD/SSE/AVX**: vectorized operations (8–16 floats in one instruction)
- **Spectre/Meltdown**: what speculative execution vulnerabilities reveal about CPU design

### Memory Hierarchy
```
Registers:    ~0.3 ns,    ~1 KB
L1 Cache:     ~1 ns,      32–64 KB (per core)
L2 Cache:     ~4 ns,      256 KB – 1 MB (per core)
L3 Cache:     ~10–40 ns,  8–32 MB (shared)
DRAM:         ~100 ns,    GBs
NVMe SSD:     ~100 μs,    TBs
HDD:          ~10 ms,     TBs
```
- Cache lines (64 bytes), spatial locality, temporal locality
- False sharing (performance killer in multi-threaded code)
- TLB (Translation Lookaside Buffer) and page walks

### Cache Coherence
- MESI protocol (Modified, Exclusive, Shared, Invalid)
- Snooping vs directory-based protocols
- Memory barriers and acquire-release semantics
- Happens-before relationships in CPU memory model

### NUMA Architecture
- Non-Uniform Memory Access: local vs remote memory
- NUMA nodes, NUMA-aware allocation
- Affects: database performance, kernel scheduling, Java GC tuning

### GPU Architecture
- CUDA cores vs CPU cores (thousands vs tens)
- SIMT (Single Instruction Multiple Threads)
- Warp scheduling, occupancy
- Memory hierarchy: registers → shared memory → L1/L2 → HBM
- PCIe bandwidth bottleneck (solved by NVLink)
- Tensor Cores (matrix multiply acceleration) — used in deep learning

### Virtual Memory
- Pages (4KB default), huge pages (2MB), transparent huge pages
- Page tables, multi-level page tables (x86-64: 4 levels)
- Page faults: demand paging
- Copy-on-write (fork() optimization)
- Memory-mapped files (mmap) — used in databases like MongoDB

---

## 8. Operating System Internals

### Process Management
- Process vs thread vs goroutine/green thread
- PCB (Process Control Block)
- fork(), exec(), wait() system calls
- Zombie and orphan processes
- Copy-on-write fork semantics
- Context switching cost (~1–10 μs for OS thread, ~100 ns for green thread)

### Thread Models
- User-space threads (N:1) — fast context switch, no parallelism
- Kernel threads (1:1) — true parallelism, heavier context switch
- Hybrid (M:N) — Go, Erlang
- Fiber/coroutine/async models — cooperative scheduling

### CPU Scheduling
- Preemptive vs cooperative scheduling
- CFS (Completely Fair Scheduler) — Linux default, uses red-black tree
- Real-time scheduling (SCHED_FIFO, SCHED_RR)
- Multilevel feedback queues
- Gang scheduling for parallel workloads (GPU clusters)
- Work stealing (Go's scheduler, Java ForkJoinPool)

### Synchronization Primitives
- **Mutex**: mutual exclusion, sleeping wait
- **Spinlock**: busy-wait, efficient for short critical sections
- **Semaphore**: counting, signaling between threads
- **Condition Variable**: wait for condition with associated mutex
- **Read-Write Lock**: concurrent readers, exclusive writers
- **Monitor**: synchronized block + condition variables (Java)
- **Atomic operations**: CAS (Compare-And-Swap), fetch-and-add, memory ordering
- Lock-free and wait-free data structures

### Deadlock
- Four conditions (mutual exclusion, hold-and-wait, no preemption, circular wait)
- Prevention, avoidance (Banker's algorithm), detection, recovery
- Deadlock in distributed systems (distributed deadlock detection)

### Memory Management
- **malloc internals**: free lists, slab allocator, jemalloc, tcmalloc
- **Garbage collection**: mark-sweep, generational, concurrent GC (Go, JVM G1, ZGC)
- **Memory pressure**: OOM killer, swap, cgroups memory limits

### File System Internals
- VFS (Virtual File System) layer
- inode structure (permissions, size, block pointers, timestamps)
- Hard links vs symbolic links
- Journaling (ext4): write-ahead log prevents corruption on crash
- Copy-on-write (Btrfs, ZFS): snapshots for free
- Log-structured file systems: optimized for sequential writes
- Page cache and dirty page writeback
- fsync() and fdatasync() — guaranteeing persistence

### Asynchronous I/O
- Blocking I/O → non-blocking I/O → I/O multiplexing → async I/O
- select() / poll() — O(n) per call, limited fds
- epoll (Linux) — O(1) per event, edge vs level triggered
- kqueue (BSD/macOS)
- io_uring (Linux 5.1+) — zero-copy, submission/completion queues, no syscall overhead
  - Used in: RocksDB, PostgreSQL 14+, high-performance networking
- Comparison: Nginx uses epoll; io_uring is the future

### Networking in OS
- Socket buffers (send buffer, receive buffer)
- TCP offloading (TSO, GRO, LRO)
- DPDK (Data Plane Development Kit) — bypass kernel networking
- Zero-copy sendfile() — used in Kafka, Nginx
- Interrupt coalescing (NAPI) — batching interrupts for high-throughput
