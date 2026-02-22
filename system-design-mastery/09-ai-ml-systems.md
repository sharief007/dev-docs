# L8: AI & ML Systems

> The deepest and fastest-moving domain. From classical ML to LLM infrastructure and distributed training.

---

## 1. Mathematical Prerequisites

### Linear Algebra (Review)
- Matrix multiplication: C = AB, shape (m×n)(n×p) → (m×p)
- Dot product as similarity measure (cos similarity = dot product of unit vectors)
- **Eigenvalue decomposition**: Av = λv → A = QΛQ⁻¹
- **SVD**: A = UΣVᵀ — used in PCA, recommendation systems, NLP
- Gradient of matrix operations: ∂(Wx)/∂W = xᵀ, ∂(Wx)/∂x = Wᵀ

### Calculus for ML
- Partial derivatives, gradient vectors, Hessian matrix
- **Chain rule**: ∂L/∂x = (∂L/∂y)(∂y/∂x) — foundation of backpropagation
- Jacobian matrix: J_{ij} = ∂f_i/∂x_j
- Taylor expansion: approximating functions locally
- Lagrange multipliers: optimization with constraints (used in SVM)

### Optimization
- **Gradient Descent**: θ = θ - α∇L (full dataset)
- **SGD**: θ = θ - α∇L(x_i, y_i) (one sample, noisy but fast)
- **Mini-batch SGD**: θ = θ - α∇L(batch) (practical default)
- **Momentum**: v = βv - α∇L, θ = θ + v (accelerates convergence, reduces oscillation)
- **Adam**: adaptive learning rates per parameter (most popular optimizer)
  - m = β₁m + (1-β₁)∇L (first moment, momentum)
  - v = β₂v + (1-β₂)(∇L)² (second moment, variance)
  - θ = θ - α × m̂/√v̂ (bias-corrected)
- **AdamW**: Adam + decoupled weight decay (standard for LLM training)
- **L-BFGS**: quasi-Newton method, good for smaller models
- **Learning rate scheduling**: step decay, exponential decay, cosine annealing, warmup

### Probability for ML
- Bayes' theorem in ML: P(model|data) ∝ P(data|model)P(model) — MLE vs MAP
- **MLE**: maximize P(data|θ) — gives point estimate
- **Cross-entropy loss** = negative log-likelihood for classification
- **KL divergence**: D_KL(P||Q) = Σ P(x) log(P(x)/Q(x)) — used in VAEs, RLHF
- **Expectation**: E[f(X)] = ∫ f(x)p(x)dx

---

## 2. Classical Machine Learning

### Supervised Learning

**Linear Regression**:
- Output: ŷ = Wᵀx + b
- Loss: MSE = (1/n)Σ(y_i - ŷ_i)²
- Closed form: W = (XᵀX)⁻¹Xᵀy
- Regularization: L2 (Ridge), L1 (Lasso), ElasticNet

**Logistic Regression**:
- Binary: ŷ = σ(Wᵀx + b), σ = 1/(1+e^-z)
- Loss: Binary Cross-entropy = -Σ[y log(ŷ) + (1-y) log(1-ŷ)]
- Multi-class: Softmax output, Categorical Cross-entropy

**Decision Trees**:
- Split on feature that maximizes information gain (ID3) or Gini impurity (CART)
- Pros: interpretable, no feature scaling needed, handles missing values
- Cons: overfits easily, unstable (small data change → different tree)

**Ensemble Methods**:
- **Random Forest**: bagging + random feature subsets → decorrelated trees
- **Gradient Boosting**: fit new trees to residuals of previous ensemble
  - **XGBoost**: regularized, fast, handles sparse data, most popular in Kaggle
  - **LightGBM**: leaf-wise growth, histogram binning → faster than XGBoost
  - **CatBoost**: handles categorical features natively, ordered boosting

**SVMs** (Support Vector Machines):
- Find maximum-margin hyperplane
- **Kernel trick**: implicitly map to higher-dimensional space (RBF, polynomial)
- Dual formulation: only support vectors matter (sparse solution)
- Not scalable to very large datasets (use SGD approximations instead)

**k-NN** (k-Nearest Neighbors):
- Lazy learning — no training, O(n) query time
- Distance metrics: Euclidean, Manhattan, cosine, Hamming
- Good for small datasets, multi-label, complex decision boundaries

### Unsupervised Learning

**k-Means Clustering**:
- Initialize k centroids, assign points to nearest, update centroids, repeat
- k-means++: smart initialization for better convergence
- Choosing k: elbow method, silhouette score

**DBSCAN** (Density-Based):
- Finds clusters of arbitrary shape
- Handles noise (outliers labeled as noise)
- Parameters: ε (radius), minPts (minimum neighbors)

**Hierarchical Clustering**:
- Agglomerative (bottom-up): start with N clusters, merge closest
- Dendrogram: tree of merges, cut at desired level

**PCA** (Principal Component Analysis):
- Find directions of maximum variance via eigendecomposition of covariance matrix
- Or via SVD: X = UΣVᵀ, principal components = columns of V
- Reduces dimensionality while preserving maximum variance

**t-SNE** (t-distributed Stochastic Neighbor Embedding):
- Non-linear dimensionality reduction for visualization
- Preserves local structure (nearby points in high-D → nearby in 2D)
- Not for training features (stochastic, no inverse transform)

**UMAP** (Uniform Manifold Approximation):
- Faster than t-SNE, preserves more global structure
- Better for large datasets
- Can be used for feature extraction (unlike t-SNE)

### Evaluation Metrics

**Classification**:
- Accuracy: TP+TN / total (misleading for imbalanced)
- Precision: TP / (TP + FP) — "of predicted positives, how many are correct?"
- Recall: TP / (TP + FN) — "of actual positives, how many did we find?"
- F1 = 2 × (Precision × Recall) / (Precision + Recall)
- AUC-ROC: area under ROC curve (TPR vs FPR at different thresholds)
- PR-AUC: area under Precision-Recall curve (better for imbalanced)

**Regression**: MAE, MSE, RMSE, R²

**Ranking**: NDCG, MRR, MAP (for recommendation systems)

**Bias-Variance Tradeoff**:
- High bias → underfitting (model too simple)
- High variance → overfitting (model memorizes training data)
- Regularization reduces variance (L1, L2, dropout, early stopping)
- More data reduces variance
- Bagging reduces variance, boosting reduces bias

---

## 3. Deep Learning

### Neural Network Fundamentals

**Forward Pass**:
```
z = Wx + b
a = activation(z)
```

**Activation Functions**:
- **ReLU**: max(0, x) — most common, avoids vanishing gradient
- **GELU**: x × Φ(x) — used in transformers (smoother than ReLU)
- **Swish/SiLU**: x × σ(x) — used in recent models
- **Sigmoid**: 1/(1+e^-x) — output layer for binary classification
- **Tanh**: (e^x - e^-x)/(e^x + e^-x) — centered at 0, better than sigmoid for hidden layers
- **Softmax**: exp(x_i)/Σexp(x_j) — multi-class output

**Backpropagation**:
- Chain rule through computational graph
- Automatic differentiation: forward pass builds computation graph, backward pass traverses it
- Frameworks: PyTorch (dynamic graph), JAX (functional, XLA compilation), TensorFlow (static/dynamic)

**Weight Initialization**:
- **Xavier (Glorot)**: W ~ N(0, 1/fan_in) — for tanh/sigmoid
- **Kaiming (He)**: W ~ N(0, 2/fan_in) — for ReLU

**Normalization**:
- **Batch Normalization**: normalize per feature across batch — accelerates training, depends on batch size
- **Layer Normalization**: normalize per sample across features — independent of batch size (used in transformers)
- **RMS Norm**: simpler LN without mean subtraction — used in LLaMA, Llama 2

### Architectures

**CNNs** (Convolutional Neural Networks):
- **Convolution**: shared weights slide across input (parameter efficiency)
- **Receptive field**: how much of input a neuron "sees"
- **Pooling**: spatial downsampling (MaxPool, AvgPool)
- **ResNet**: skip connections prevent vanishing gradient in deep networks
- **EfficientNet**: neural architecture search, compound scaling (depth+width+resolution)

**Transformers** (Vaswani et al., "Attention Is All You Need", 2017):

**Self-Attention**:
```
Q = XW_Q, K = XW_K, V = XW_V
Attention(Q,K,V) = softmax(QKᵀ/√d_k) V
```
- Q, K, V: query, key, value projections of input
- Dot product of Q and K: similarity between tokens
- Softmax: attention weights (sum to 1)
- Weighted sum of V: output

**Multi-Head Attention**:
- Multiple attention heads, each learning different relationships
- Concatenate outputs, project: `MultiHead = Concat(head_1...head_h) W_O`
- Different heads capture: local vs global, syntactic vs semantic

**Positional Encoding**:
- **Sinusoidal (original)**: fixed, generalizes to unseen lengths
- **Learned**: position embeddings trained end-to-end
- **RoPE** (Rotary Position Embedding): rotate Q/K vectors by position angle — relative positions encoded in dot products, used in LLaMA, GPT-NeoX
- **ALiBi** (Attention with Linear Biases): add linear bias to attention scores by distance — better length generalization

**FFN (Feed-Forward Network)**:
- Two linear layers with activation: FFN(x) = max(0, xW₁+b₁)W₂+b₂
- Typically 4× hidden dimension
- **SwiGLU**: gated variant used in LLaMA — better than vanilla FFN

**Transformer Variants**:
- **Encoder-only (BERT)**: bidirectional attention, for classification, NER, embeddings
- **Decoder-only (GPT)**: causal (unidirectional) attention, for generation
- **Encoder-Decoder (T5, BART)**: encoder processes input, decoder generates output

**Mixture of Experts (MoE)**:
- N expert FFN layers, router selects top-K for each token
- Sparse computation: each token only activates K experts
- Models: Mixtral (8×7B, 2 active), GPT-4 (rumored 8 experts), Switch Transformer
- Benefits: huge model capacity without proportional compute increase

**Graph Neural Networks (GNNs)**:
- **GCN** (Graph Convolutional Network): aggregate neighbor features
- **GraphSAGE**: sample and aggregate with learned aggregation
- **GAT** (Graph Attention Network): attention weights on edges
- Applications: drug discovery, social networks, fraud detection, knowledge graphs

### Diffusion Models

- Add Gaussian noise iteratively (forward process), learn to denoise (reverse process)
- **DDPM** (Denoising Diffusion Probabilistic Models): original, slow sampling
- **DDIM**: deterministic, 10-50× faster sampling via larger steps
- **Stable Diffusion**: latent diffusion (U-Net in latent space), CLIP conditioning
- Applications: image generation, text-to-image, audio generation, video generation

---

## 4. Large Language Models (LLMs)

### Architecture Details

**KV Cache**:
- During autoregressive generation, recompute K and V for each new token — quadratic cost
- Cache K and V for all previous tokens, only compute for new token
- Memory: KV_size = 2 × num_layers × seq_len × hidden_dim × bytes_per_float
- KV cache for LLaMA 70B (4096 ctx): ~20GB

**Attention Complexity**:
- Full self-attention: O(n²) time and space (n = sequence length)
- **FlashAttention** (Dao et al.): O(n) memory by recomputing attention in tiles
  - Avoids materializing full n×n attention matrix
  - 2-4× faster than standard attention, same output
  - **FlashAttention-2**: further optimized for A100/H100 hardware
- **FlashAttention-3**: specific to H100 FP8 hardware
- **Alternatives to full attention**: sparse (BigBird, Longformer), linear (Performer, Linformer)
- **Ring Attention**: distribute long-sequence attention across GPUs

**Context Length Extension**:
- RoPE base extension: increase θ in RoPE formula (allows longer contexts with some degradation)
- **YaRN** (Yet another RoPE extensioN): scale-based interpolation, better long-context performance
- **LongRoPE**: sliding window + mixed RoPE, extends to 2M tokens

**Tokenization**:
- **BPE (Byte Pair Encoding)**: GPT-2, GPT-4, LLaMA (SentencePiece-BPE)
  - Start with characters, iteratively merge most frequent pairs
- **WordPiece**: BERT — like BPE but merges maximize language model likelihood
- **SentencePiece**: language-agnostic BPE (handles spaces as special character)
- Typical vocabulary: 32K–128K tokens
- Unknown characters: byte-level BPE handles any Unicode

**Scaling Laws**:
- **Kaplan et al. (2020)**: loss ∝ N^-0.076 (model size), C^-0.05 (compute)
- **Chinchilla (2022)**: optimal ratio is ~20 tokens/parameter — GPT-3 was undertrained
  - For N params, use ~20N tokens of training data
  - LLaMA 65B: trained on 1.4T tokens (much more than Chinchilla optimal — intentionally overtrained for efficient inference)

### LLM Pre-training

**Next-Token Prediction (Causal LM)**:
- Input: token sequence [t₁, t₂, ..., tₙ]
- Target: shifted sequence [t₂, t₃, ..., tₙ₊₁]
- Loss: cross-entropy averaged over all tokens

**Data Pipeline**:
- Sources: Common Crawl (internet), books, code (GitHub), Wikipedia, academic papers
- **Deduplication**: exact (hash), near-exact (MinHash LSH), semantic (embedding similarity)
- **Quality filtering**: classifier to remove low-quality web text
- **Domain mixing**: adjust ratios of different data sources
- Data order matters: curriculum learning, random shuffling

**Training Stability**:
- Loss spikes: common in large model training (reduce LR, skip batch)
- Gradient norms: monitor for explosions
- **Gradient clipping**: clip gradient norm to max value (e.g., 1.0)
- Loss curves: watch for divergence, plateaus

### LLM Post-Training

**SFT (Supervised Fine-Tuning)**:
- Train on (instruction, response) pairs
- Often small high-quality dataset (1K–100K examples)
- Learning rate much lower than pre-training

**RLHF (Reinforcement Learning from Human Feedback)**:
1. **Collect preference data**: human raters rank model responses
2. **Train reward model**: predict human preference from completions
3. **PPO optimization**: optimize policy to maximize reward model score, constrained by KL divergence from SFT policy

**DPO (Direct Preference Optimization)**:
- Skip reward model — directly optimize policy on preference pairs
- Loss: -log(σ(β log π(y_w/x)/π_ref(y_w/x) - β log π(y_l/x)/π_ref(y_l/x)))
- Much simpler than RLHF, often competitive or better
- Variants: IPO, KTO, SimPO

**RLAIF** (RL from AI Feedback):
- Use another LLM (Claude, GPT-4) as preference rater instead of humans
- Constitutional AI (Anthropic): AI provides critique + revision + ratings

### Parameter-Efficient Fine-Tuning (PEFT)

**LoRA (Low-Rank Adaptation)**:
- Don't update original weights W ∈ ℝ^(d×k)
- Add trainable low-rank matrices: ΔW = BA where B∈ℝ^(d×r), A∈ℝ^(r×k), r<<min(d,k)
- At inference: merge W + BA (no inference overhead)
- Typical rank: r=8–64

**QLoRA**:
- Quantize base model to 4-bit (NF4), keep LoRA adapters in BF16
- Fine-tune 65B model on single 48GB GPU
- Double quantization: quantize the quantization constants too

**Prefix Tuning / Prompt Tuning**:
- Add trainable "soft tokens" to input
- Only these embeddings are updated
- Fewer parameters than LoRA but less effective

### Quantization

**Why quantize**: LLaMA 70B in BF16 = 140GB; in 4-bit = ~35GB

**Post-Training Quantization (PTQ)**:
- **INT8**: 8-bit integer weights (bitsandbytes)
- **INT4**: 4-bit integer (GPTQ, GGUF)
- **NF4** (Normal Float 4): 4-bit with data-type quantization matching weight distributions
- **GPTQ**: one-shot weight quantization via OBS (Optimal Brain Surgeon), layer-by-layer
- **AWQ** (Activation-aware Weight Quantization): protect salient weights from large activation

**Formats**:
- **GGUF** (previously GGML): llama.cpp native format, CPU/GPU inference
- **GPTQ**: GPU-optimized 4-bit
- **AWQ**: fast, high accuracy 4-bit

**Quantization-Aware Training (QAT)**: train with simulated quantization (more accurate but expensive)

**FP8**: 8-bit floating point (two formats: E4M3, E5M2), supported on H100 — new standard for training

### LLM Inference Optimization

**Continuous Batching**:
- Classic: batch all requests, wait for slowest to finish
- Continuous: as one sequence finishes, add new one immediately
- Increases GPU utilization dramatically
- Implemented by: vLLM, TGI, TensorRT-LLM

**Paged Attention (vLLM)**:
- KV cache memory fragmentation problem: variable-length sequences waste memory
- Solution: KV cache stored in fixed-size blocks (pages), allocated on demand
- Like virtual memory for KV cache
- Enables continuous batching + handles variable lengths efficiently

**Speculative Decoding**:
- Small draft model proposes N tokens
- Large model verifies all N tokens in one forward pass
- Accept if large model agrees, reject and regenerate if not
- Speedup: 2-3× for typical text (more for long sequences)

**Tensor Parallelism at Inference**:
- Split model across multiple GPUs (e.g., LLaMA 70B across 4×A100 80GB)
- Attention heads split across GPUs
- All-reduce between GPUs at each layer
- NCCL for communication

**Flash Decoding**: parallelize attention across sequence length during decoding (for long contexts)

---

## 5. Distributed Training

### Data Parallelism

**DDP (Distributed Data Parallel)**:
- Each GPU has full model copy, different data batch
- **AllReduce** gradients after each step: sum gradients across all GPUs
- `torch.nn.parallel.DistributedDataParallel`
- Efficient for models that fit on single GPU

**FSDP (Fully Sharded Data Parallel)**:
- Shard model parameters across GPUs (not just gradients)
- Each GPU only stores 1/N of parameters
- Gather parameters needed for forward/backward, discard after
- Enables training models too large for single GPU
- Tradeoff: more communication than DDP

### Model Parallelism

**Tensor Parallelism**:
- Split individual layer (e.g., attention heads, MLP dimensions) across GPUs
- All-reduce within each layer
- Tight coupling: GPUs must synchronize at every layer
- **Megatron-LM**: pioneered tensor parallelism for transformer training

**Pipeline Parallelism**:
- Different GPUs handle different layers (pipeline stages)
- Micro-batches flow through pipeline stages
- **Bubble problem**: GPUs idle waiting for micro-batches
- **GPipe**: batch of micro-batches, flush at end (large memory)
- **PipeDream (1F1B)**: interleave forward and backward passes to reduce bubble

### 3D Parallelism

Combine all three:
```
- Data parallelism: N data replicas
- Pipeline parallelism: M pipeline stages per replica
- Tensor parallelism: K tensor shards per stage
Total GPUs = N × M × K
```
Used to train GPT-3/4 scale models

### ZeRO (Zero Redundancy Optimizer — DeepSpeed)

**ZeRO-1**: Shard optimizer states across data parallel workers
**ZeRO-2**: + Shard gradients
**ZeRO-3**: + Shard parameters (= FSDP)
**ZeRO-Infinity**: Offload to CPU memory or NVMe storage

### Communication Collectives

- **AllReduce**: sum values across all GPUs, result on all GPUs (used for gradient sync)
- **AllGather**: gather different data from all GPUs, result on all GPUs (FSDP forward pass)
- **ReduceScatter**: reduce and scatter result across GPUs (FSDP backward pass)
- **Broadcast**: send data from one GPU to all others

**NCCL (NVIDIA Collective Communications Library)**:
- Optimized for NVIDIA GPU cluster communication
- Uses NVLink (within node) and InfiniBand (across nodes)
- Ring AllReduce: optimal bandwidth for large messages
- Tree AllReduce: optimal latency for small messages

### GPU Cluster Networking
- **NVLink**: 600 GB/s (NVLink 4.0) per GPU — within node
- **NVSwitch**: enables any-to-any GPU communication within node
- **InfiniBand**: 400 Gb/s (HDR200) to 800 Gb/s (NDR) — across nodes
- **RoCE** (RDMA over Converged Ethernet): InfiniBand semantics over Ethernet

---

## 6. MLOps & ML Infrastructure

### Experiment Tracking

- **MLflow**: open-source, experiments, parameters, metrics, artifacts, model registry
- **Weights & Biases (W&B)**: richer UI, sweeps (hyperparameter search), collaboration
- **Comet**: similar to W&B

### Model Registry

- Versioned model storage with metadata
- States: staging → production → archived
- Artifact storage: S3, MLflow artifact store
- Model lineage: track which data + code produced the model

### Model Serving

**Inference Servers**:
- **Triton Inference Server** (NVIDIA): multi-framework, batching, GPU optimization
- **TorchServe**: PyTorch-native serving
- **TensorFlow Serving**: TF-native
- **BentoML**: package models as services, deploy anywhere
- **vLLM**: LLM-specialized, paged attention, continuous batching
- **Text Generation Inference** (TGI): Hugging Face, production LLM serving

**Batch vs Online Inference**:
- **Batch**: process large datasets offline (daily scores, pre-computed embeddings)
- **Online**: real-time, low-latency (<100ms)
- **Near-real-time**: streaming inference (Kafka → model → Kafka)

**Model A/B Testing**:
- Shadow mode: new model receives traffic, doesn't serve responses
- Canary: 5% of traffic to new model, monitor metrics
- Traffic splitting: use feature flags or load balancer

### Model Monitoring

**Data Drift**: input distribution shifts from training distribution
- **PSI (Population Stability Index)**: compare feature distributions
- **KS test** (Kolmogorov-Smirnov): compare distributions statistically
- **Jensen-Shannon divergence**: symmetric KL divergence

**Concept Drift**: relationship between features and labels changes
- Monitor prediction distribution
- Monitor label distribution (if labels available with delay)
- Monitor downstream business metrics

**Model Performance Degradation**:
- Log predictions, collect ground truth with delay
- Compute metrics on a rolling window
- Alert when metrics degrade beyond threshold

---

## 7. Vector Databases & RAG

### Approximate Nearest Neighbor (ANN) Algorithms

**HNSW (Hierarchical Navigable Small World)**:
- Multi-layer graph: top layers have long-range connections, bottom layer has all points
- Search: start at top layer, greedily navigate to nearest, descend to next layer
- **Build**: O(n log n), **Search**: O(log n)
- High recall, in-memory, no disk support natively
- Hyperparameters: M (connections per node), efConstruction (build quality), ef (search quality)

**IVF (Inverted File Index)**:
- K-means cluster vectors into C centroids
- Assign each vector to nearest centroid
- Search: find nprobe nearest centroids, search only those clusters
- Faster than HNSW for large datasets, lower recall
- **IVF-PQ**: apply Product Quantization within each cluster for memory savings

**Product Quantization (PQ)**:
- Split each vector into M sub-vectors
- Quantize each sub-vector to k* centroids (codebook)
- Represent each vector as M small integers
- Compression: 100× memory reduction, fast approximate distance computation

### RAG Pipeline (Retrieval-Augmented Generation)

**Naive RAG**:
```
1. User query
2. Embed query → dense vector
3. Search vector DB → top-K chunks
4. Stuff chunks into LLM context
5. LLM generates answer
```

**Advanced RAG**:

*Query-side improvements*:
- **Query rewriting**: LLM rewrites ambiguous queries
- **HyDE** (Hypothetical Document Embeddings): generate hypothetical answer, use as query
- **Query expansion**: add synonyms, related terms (keyword extraction)
- **Multi-query retrieval**: generate N query variants, merge results

*Retrieval improvements*:
- **Hybrid search**: combine BM25 (lexical) + dense (semantic) via Reciprocal Rank Fusion
- **Re-ranking**: apply cross-encoder to top-K results, reorder by relevance
- **Contextual compression**: extract only relevant parts of retrieved chunks
- **Parent-child chunks**: retrieve parent chunk for context around small matched chunk

*Chunking strategies*:
- **Fixed-size**: N tokens with overlap — simple, widely used
- **Semantic chunking**: split at natural boundaries (sentences, paragraphs)
- **Document-aware**: respect document structure (headers, sections)
- **Hierarchical**: multi-level chunks for multi-hop questions

**Re-rankers (Cross-Encoders)**:
- Input: (query, passage) pair → relevance score
- Much more accurate than bi-encoder retrieval but expensive O(n)
- Use as 2nd stage: retrieve top-100 with HNSW, re-rank to top-10
- Models: BGE-Reranker, Cohere Rerank API

---

## 8. Recommendation Systems

### Collaborative Filtering

**Memory-based**: compute similarity between users or items
- User-User CF: users with similar history have similar preferences
- Item-Item CF: items liked by the same users are similar
- Scalability: O(n²) users — needs dimensionality reduction

**Model-based (Matrix Factorization)**:
- Decompose user-item matrix: R ≈ UVᵀ
- **ALS** (Alternating Least Squares): hold U fixed, solve for V; alternate
- Good for explicit feedback (ratings)
- **BPR** (Bayesian Personalized Ranking): optimize pairwise ranking for implicit feedback

### Two-Tower Model (Modern Industry Standard)

```
Query Tower (user)         Item Tower (item)
   User features              Item features
       ↓                           ↓
   User embedding             Item embedding
          ↘                 ↙
           Dot product → Score
```

- Train offline: maximize score for positive user-item pairs
- Inference: embed all items offline, store in vector DB; embed user at query time, ANN search
- Used by: YouTube, TikTok, Pinterest, Spotify

### Sequential Recommendation

- Model user's evolving preferences based on interaction sequence
- **SASRec** (Self-Attentive Sequential Recommendation): transformer-based, next-item prediction
- **BERT4Rec**: bidirectional transformer, masked item prediction
- Better than matrix factorization for session-based recommendations

### Retrieval + Ranking Pipeline

```
Candidate Generation: millions of items → hundreds
    ↓ (fast ANN search, two-tower)
Pre-ranking: filter to top-500
    ↓ (simple features, fast model)
Ranking: top-500 → top-50
    ↓ (complex features, deep model)
Re-ranking: top-50 → final list
    ↓ (business rules, diversity, freshness)
```

### Exploration vs Exploitation

**Multi-Armed Bandits**:
- **ε-greedy**: exploit best arm with prob 1-ε, explore random with prob ε
- **UCB** (Upper Confidence Bound): select arm with highest UCB(arm) = mean + sqrt(2 log t / n_arm)
- **Thompson Sampling**: sample from posterior for each arm, select highest — Bayesian optimal

**Contextual Bandits**: action depends on context (user, item features)
**LinUCB**: linear model for expected reward, UCB for exploration
