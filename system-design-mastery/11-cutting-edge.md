# L10: Cutting Edge

> The frontier of system design. These topics are actively evolving — understanding the fundamentals lets you adapt as they mature.

---

## 1. Blockchain & Web3

### Distributed Ledger Fundamentals

- **Blockchain**: linked list of blocks, each block contains transactions + hash of previous block
- **Hash chaining**: changing any block invalidates all subsequent blocks (tamper-evident)
- **Immutability**: once confirmed, data cannot be altered
- **Decentralization**: no single authority controls the ledger

### Consensus Mechanisms

**Proof of Work (PoW)**:
- Miners compete to find nonce such that hash(block) < target (CPU-intensive puzzle)
- Difficulty adjusts to maintain ~10 min block time (Bitcoin)
- **51% attack**: control >50% of hash rate → can rewrite history
- Energy-intensive: Bitcoin uses ~130 TWh/year
- Used by: Bitcoin

**Proof of Stake (PoS)**:
- Validators stake ETH/tokens as collateral; chosen to propose blocks by stake weight
- **Slashing**: validators that act dishonestly lose their stake
- Much more energy-efficient than PoW
- **Finality**: Ethereum uses Casper FFG for economic finality
- Used by: Ethereum (post-Merge), Cardano, Solana

**Delegated PoS (DPoS)**:
- Token holders vote for a small set of delegates (21 in EOS)
- Faster finality, more centralized
- Used by: EOS, TRON

**Proof of Authority (PoA)**:
- Known, trusted validators (identity-based, not stake-based)
- Permissioned blockchains: Hyperledger Besu, private Ethereum networks
- No Sybil protection via economics (relies on identity vetting)

### Ethereum Architecture

- **EVM (Ethereum Virtual Machine)**: stack-based VM, runs smart contracts
- **Gas**: computational cost of EVM operations — prevents DoS
- **Solidity**: main smart contract language
- **Accounts**: EOA (Externally Owned Account) and Contract Account
- **State trie**: Patricia Merkle trie stores all account balances + contract storage
- **EIP (Ethereum Improvement Proposal)**: standards process

**EIP-1559 (London fork)**:
- Base fee: burned per block, adjusts based on congestion
- Priority fee (tip): goes to validator
- Reduces ETH supply over time (deflationary mechanism)

### Smart Contracts and DeFi

- **Smart contract**: code deployed on blockchain, executes deterministically
- **ERC-20**: fungible token standard (USDC, UNI, LINK)
- **ERC-721**: NFT standard (non-fungible tokens)
- **DeFi**: financial services without intermediaries
  - **AMM (Automated Market Maker)**: Uniswap, Curve — x×y=k invariant for token swaps
  - **Lending**: Aave, Compound — over-collateralized loans
  - **Yield farming**: earn rewards by providing liquidity

### Layer 2 Scaling Solutions

**Problem**: Ethereum mainnet handles ~15 TPS — insufficient for global adoption

**Rollups** (main solution):
- Execute transactions off-chain, post data/proof to Ethereum L1
- L1 Ethereum = data availability + security + settlement

**Optimistic Rollups** (Optimism, Arbitrum):
- Assume transactions valid by default
- 7-day fraud challenge window (dispute period)
- If fraud proof submitted, invalid batch rolled back
- Simple EVM compatibility

**ZK-Rollups** (zkSync, StarkNet, Polygon zkEVM):
- Generate cryptographic validity proof for every batch
- No fraud period — immediate finality on L1
- More complex, but faster and more secure
- ZK-EVMs: increasingly EVM-compatible

**State Channels** (Lightning Network for Bitcoin):
- Open channel by locking funds on-chain
- Exchange signed state updates off-chain
- Close channel by posting final state on-chain
- Best for: high-frequency micropayments between same parties

### IPFS (InterPlanetary File System)

- Content-addressed storage: `ipfs://{CID}` where CID = hash of content
- Files split into chunks, each chunk has CID
- **DHT (Distributed Hash Table)**: Kademlia-based, find who has a CID
- **Pinning**: explicitly request a node to keep your data
- **Filecoin**: incentive layer for IPFS storage (miners get paid to store)

### MEV (Maximal Extractable Value)

- Value extractable by block producers by reordering/inserting/censoring transactions
- Examples: front-running DEX trades, sandwich attacks, liquidation bots
- **Flashbots**: system for transparent MEV extraction (MEV-Boost in Ethereum PoS)

---

## 2. Federated Learning

### What It Solves

Traditional ML: send all private data to central server → privacy concerns, legal compliance (GDPR, HIPAA)

Federated Learning: model trained collaboratively without centralizing data

### Federated Averaging (FedAvg)

```
Global model → send to N devices
Each device: train on local data for E epochs
Each device: send model updates (gradients or weights) to server
Server: aggregate updates (weighted average by data size)
Repeat for T rounds
```

### Challenges

**Statistical heterogeneity**: devices have non-IID (non-identically distributed) data
- Device 1: only sees cats, Device 2: only sees dogs → global model diverges
- **FedProx**: adds proximal term to penalize deviation from global model

**Systems heterogeneity**: devices have different compute, memory, connectivity
- Stragglers: slow devices delay global aggregation
- **Partial participation**: only wait for fraction of devices per round

**Communication efficiency**: model updates are expensive to send
- Gradient compression: sparsification, quantization of updates
- Delta weights: send only changed weights

**Privacy leakage**: even gradients can leak training data information
- **Secure Aggregation**: server sees only sum of updates (via MPC), not individual gradients
- **Differential Privacy**: add noise to gradients before sending

### Differential Privacy in Federated Learning

- **DP-SGD**: clip gradients to norm C, add Gaussian noise N(0, σ²) to clipped gradients
- Trade-off: privacy budget ε vs model utility
- **Per-sample gradient clipping**: clip each sample's gradient separately before aggregation

### On-Device Inference

- **TensorFlow Lite**: Google's mobile/edge inference framework
- **CoreML**: Apple's framework (Metal GPU acceleration, ANE for neural engine)
- **ONNX Runtime**: cross-platform inference, works on mobile/edge/server
- **PyTorch Mobile**: PyTorch models on iOS/Android

**Model compression for edge**:
- Quantization: INT8 weights (4× memory reduction, 2–4× faster)
- Pruning: remove unimportant weights (sparse models)
- Knowledge distillation: train small "student" model to mimic large "teacher"
- Neural Architecture Search (NAS): AutoML to find efficient architectures

---

## 3. Quantum Computing (Implications for Systems)

### Quantum Computing Basics

- **Qubit**: quantum bit, can be in superposition of 0 and 1 simultaneously
- **Superposition**: qubit = α|0⟩ + β|1⟩ (probability amplitudes, |α|² + |β|² = 1)
- **Entanglement**: two qubits whose states are correlated regardless of distance
- **Quantum gates**: operations on qubits (Hadamard, CNOT, Toffoli)
- **Decoherence**: qubits lose quantum state due to environmental noise (main engineering challenge)
- **Quantum error correction**: required for fault-tolerant quantum computing (requires ~1000 physical qubits per logical qubit)

### Quantum Algorithms That Matter

**Shor's Algorithm (1994)**:
- Factors large integers in polynomial time: O((log N)³)
- Classical: no polynomial-time factoring algorithm known
- **Impact**: breaks RSA, ECC, Diffie-Hellman
- Threat: "harvest now, decrypt later" — adversaries store encrypted traffic now, decrypt when quantum computers exist
- Timeline: ~10–15 years for cryptographically relevant quantum computers (estimates vary)

**Grover's Algorithm**:
- Searches unsorted database of N items in O(√N) instead of O(N)
- Quadratic speedup (not exponential)
- **Impact on symmetric crypto**: doubles effective key length requirement
  - AES-128: was 128-bit security → 64-bit against Grover (insufficient)
  - AES-256: was 256-bit security → 128-bit against Grover (still secure)
  - SHA-256: 256-bit collision resistance → 128-bit against Grover (acceptable)
- **Mitigation**: already done — use AES-256, SHA-256+

### Post-Quantum Cryptography (Already Covered in Security)

- NIST 2024 standards: ML-KEM, ML-DSA, SLH-DSA
- Hybrid deployments: classical + PQC for transition
- **TLS 1.3 + Kyber**: already deployed by Cloudflare, Google

### Quantum Networking

- **Quantum key distribution (QKD)**: use quantum mechanics to distribute cryptographic keys
  - Any eavesdropping disturbs quantum states (detectable)
  - **BB84 protocol**: encode bits in photon polarization
- **Quantum repeaters**: extend QKD range (quantum entanglement limited by fiber loss)
- **Quantum internet**: long-term vision, decades away

---

## 4. AI Systems at the Frontier

### LLM Agents

**ReAct Pattern** (Reasoning + Acting):
- Interleave thought (reasoning) with action (tool call)
- Thought → Action → Observation → Thought → ...
- More interpretable than end-to-end neural approaches

**Tool Use / Function Calling**:
- LLM generates structured JSON describing tool to call + parameters
- Host executes tool, returns result to LLM context
- Tools: web search, code execution, database queries, APIs

**Planning Strategies**:
- **Chain-of-Thought (CoT)**: prompt model to "think step by step"
- **Tree of Thoughts (ToT)**: explore multiple reasoning paths, backtrack
- **Graph of Thoughts (GoT)**: non-linear reasoning structure

**Memory Systems**:
- **Working memory**: in-context window
- **Episodic memory**: vector DB of past interactions
- **Semantic memory**: knowledge base, RAG
- **Procedural memory**: fine-tuned behaviors, tool definitions

**Multi-Agent Systems**:
- Orchestrator → calls specialized sub-agents
- Agents can verify each other's work
- Debates: multiple agents argue positions, judge decides
- Used in: AutoGPT, CrewAI, LangGraph, Claude's agentic capabilities

### Multi-Modal Models

- **Vision-Language Models (VLMs)**: understand images + text
  - **CLIP** (Contrastive Language-Image Pre-training): embed images and text in shared space
  - **GPT-4V, Claude 3, Gemini**: native multi-modal
  - Architecture: visual encoder (ViT) + cross-attention into language model
- **Audio-Language Models**: speech understanding, music, audio generation
- **Video Models**: temporal understanding (Sora-style)
- **Robotics foundation models**: RT-2, OpenVLA — visual + language + action

### World Models

- Internal model of how the world works and evolves
- Predict next state given current state + action
- Used in: model-based reinforcement learning, robotics planning
- **JEPA** (Joint Embedding Predictive Architecture — Yann LeCun): predict in embedding space (not pixel space)
- **Dreamer**: learn compact world model, plan in latent space

### AI Hardware

**NVIDIA H100**:
- 80GB HBM3 memory, 3.35 TB/s memory bandwidth
- Transformer Engine: FP8 mixed precision training
- NVLink 4.0: 900 GB/s bidirectional per GPU
- Used for: LLM training + inference

**Google TPUs (Tensor Processing Units)**:
- Custom ASIC for neural network training
- TPU v4: 275 TFLOPS BF16, 32 GB HBM
- **TPU Pods**: thousands of TPUs connected via 3D torus network (IPU: 3000 TB/s)
- Used to train: PaLM 2, Gemini, BERT original

**Groq LPU (Language Processing Unit)**:
- Deterministic execution, no memory bottleneck
- Fast inference (no batching needed, 500+ tokens/sec)
- Not for training, specialized for inference

**Cerebras WSE (Wafer Scale Engine)**:
- Largest chip ever built (462,000 mm², vs 826mm² for H100)
- 40GB on-chip SRAM (no HBM bottleneck)
- Excellent for small-batch training

### Neuromorphic Computing

- Inspired by biological neurons (spiking neural networks)
- **Intel Loihi**: neuromorphic chip (2017, 2021)
- **IBM TrueNorth**: 4096 cores, 1M neurons, 256M synapses
- Event-driven: compute only when input changes (very energy-efficient)
- Current state: research, not production-ready for most applications

---

## 5. Edge Computing

### Edge AI Inference

**Why edge**:
- Latency: no round trip to cloud
- Privacy: data never leaves device
- Bandwidth: process locally, send only results
- Availability: works without internet

**Deployment pipeline**:
```
Train cloud model → Compress (quantize, prune, distill) → Export (ONNX, CoreML, TFLite) → Deploy to device
```

### WebAssembly (WASM) for Edge

- Portable binary instruction format for stack-based VM
- Near-native performance in browser and on servers
- **WASM at edge**: Cloudflare Workers, Fastly Compute@Edge
- **WASI** (WebAssembly System Interface): access OS resources portably
- **Component Model**: modularity for WASM (wasmtime, jco)

**Why WASM for edge computing**:
- Smaller than containers (instant startup)
- Language-agnostic (compile any language to WASM)
- Sandboxed by default (capability-based security)
- Platform-independent

### 5G and Mobile Edge Computing

- **Network slicing**: dedicated virtual network segments with guaranteed QoS
- **MEC (Multi-access Edge Computing)**: move compute to edge of 5G network
  - Ultra-low latency (<1ms for some use cases)
  - Applications: autonomous vehicles, AR/VR, real-time gaming
- **URLLC (Ultra-Reliable Low Latency Communications)**: 5G slice for critical applications
- **Network function virtualization (NFV)**: run telco functions as software on commodity hardware

---

## 6. Emerging Database Paradigms

### HTAP (Hybrid Transactional/Analytical Processing)

- Combine OLTP and OLAP in one system
- No ETL needed: analytics on fresh transactional data
- **TiDB** (PingCAP): HTAP — TiKV (row storage, OLTP) + TiFlash (columnar, OLAP)
- **SingleStore** (formerly MemSQL): in-memory HTAP
- **Google AlloyDB**: PostgreSQL-compatible with columnar acceleration

### Serverless Databases

**PlanetScale**:
- MySQL-compatible, horizontal sharding via Vitess
- **Non-blocking schema changes**: deploy schema changes without locking
- Branching: database branches like Git branches (dev, preview, main)
- Serverless connection pooling

**Neon**:
- Serverless PostgreSQL, compute and storage separated
- Scale to zero: pause compute when idle
- Instant branching: fork database for testing

**CockroachDB Serverless**:
- Auto-scales compute, charges by RU (Request Units)
- Multi-region, geo-partitioned

### Graph + Vector + Document Convergence

- **Multi-model databases**: one DB for multiple data models
- **ArangoDB**: graph + document + key-value in one
- **Neo4j + vector search**: graph traversal + semantic similarity search
- **Weaviate**: vector search + graph relationships
- Trend: specialized databases adding capabilities of others

### Real-Time Analytics

**Apache Pinot**:
- Sub-second OLAP on live streaming data
- Used by: LinkedIn, Uber, Stripe, Stripe
- Star-tree indexes for complex aggregations

**Apache Druid**:
- Real-time ingestion + historical queries
- Used by: Lyft, Airbnb, eBay, Walmart

**ClickHouse**:
- Column-oriented OLAP, extremely fast aggregations
- Written in C++, vectorized execution
- Used by: Cloudflare, Contentsquare, Deutsche Telekom

---

## 7. Real-Time AI Systems

### Streaming Inference

- Run ML models on streaming data (Kafka/Flink → model → Kafka)
- Challenges: model initialization, state management, latency requirements
- Patterns: embed models in Flink operators, external model server with async calls

### Online Learning

- Model updates continuously as new data arrives (no batch retraining)
- **SGD with streaming data**: update model on each example
- **River** (Python library): online ML algorithms
- Challenges: concept drift, catastrophic forgetting
- Used for: real-time ad CTR prediction, fraud detection

### Continual Learning

- Learn new tasks without forgetting old tasks (**catastrophic forgetting** problem)
- **Elastic Weight Consolidation (EWC)**: penalize changing important weights
- **Progressive Neural Networks**: new columns for new tasks, frozen old columns
- **Replay**: mix new data with samples from old tasks (experience replay)
- Critical for: long-running AI agents, production models that need retraining

### Concept Drift Adaptation

- Distribution shift detected → trigger retraining or fine-tuning
- Adaptive models: weigh recent examples more heavily
- **ADWIN (Adaptive Windowing)**: statistical test for drift detection
- **Page-Hinkley test**: sequential test for mean shift

---

## Looking Ahead: Key Trends to Watch

| Trend | Where It's Heading |
|-------|-------------------|
| LLM Agents | Autonomous software engineering, scientific discovery |
| Post-Quantum Crypto | NIST standards deployed in TLS/SSH by 2030 |
| Federated Learning | Privacy-preserving AI in healthcare, finance |
| Edge AI | Inference on devices with <10W power budgets |
| WASM | Server-side WASM replacing containers for serverless |
| Vector + Graph DBs | Converge into unified knowledge graph stores |
| Neuromorphic | 10+ years from practical deployment |
| Quantum Computing | Harvest-now-decrypt-later already a real threat; RSA not safe after ~2030 |
