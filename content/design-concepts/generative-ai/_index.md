---
title: Generative AI
weight: 16
type: docs
toc: false
---

Learn how a language-model feature becomes a system you can operate: generation, retrieval, evaluation, safety, and serving. This track assumes basic APIs and databases, not prior ML mathematics.

**New to generative AI?** Follow the cards in order. One fictional ExampleCo policy assistant connects the explanations, diagrams, and examples. Full-time employees in Region A receive 20 annual leave days. Each of the 13 concepts ends with **20 tricky questions and expandable answers**: try answering before opening them.

The same planning corpus is used wherever a number is about ExampleCo: 100,000 documents, 50 chunks each, five million passages, and a 30 questions/s peak. Smaller drills are labeled as hypotheticals.

## Foundations

{{< cards >}}
  {{< card link="generative-ai-fundamentals" title="Generative AI & LLM Fundamentals" subtitle="Generation vs classification, training vs inference, transformer intuition, and why fluent answers can be wrong" >}}
  {{< card link="tokens-context-and-sampling" title="Tokens, Context Windows & Sampling" subtitle="Worked token budgets, conversation history, truncation, temperature, and top-p without confusing randomness with correctness" >}}
  {{< card link="prompt-design-and-structured-outputs" title="Prompt Design & Structured Outputs" subtitle="Clear tasks, few-shot examples, response schemas, business validation, and source-backed output contracts" >}}
  {{< card link="embeddings-and-semantic-search" title="Embeddings & Semantic Search" subtitle="Vectors, cosine similarity, exact-search examples, model compatibility, and semantic vs keyword matching" >}}
  {{< card link="vector-search-and-databases" title="Vector Search & Vector Databases" subtitle="Exact vs approximate search, HNSW knobs, filtered recall, storage arithmetic, and when not to add a vector database" >}}
{{< /cards >}}

## Grounded Answers, Evaluation & Safety

{{< cards >}}
  {{< card link="retrieval-augmented-generation" title="Retrieval-Augmented Generation (RAG)" subtitle="From manually supplied evidence to retrieval, checked citations, missing evidence, and separate retrieval/generation failures" >}}
  {{< card link="rag-ingestion-and-retrieval" title="RAG Ingestion, Retrieval Quality & Freshness" subtitle="Extraction, chunking, BM25 plus vectors, cross-encoder reranking, idempotent jobs, atomic activation, and revocation" >}}
  {{< card link="prompting-vs-rag-vs-fine-tuning" title="Prompting vs RAG vs Fine-Tuning" subtitle="Match the solution to the failure, understand LoRA, avoid evaluation leakage, and compare complete lifecycle costs" >}}
  {{< card link="generative-ai-evaluation" title="Generative AI Evaluation & Observability" subtitle="Reviewed cases, recall and precision, groundedness, judge limitations, safe traces, and release gates" >}}
  {{< card link="generative-ai-safety" title="GenAI Safety, Prompt Injection & Data Privacy" subtitle="Untrusted content, enforceable permissions, data minimization, safe outputs, provider settings, and resource limits" >}}
{{< /cards >}}

## Tools & Production Trade-offs

{{< cards >}}
  {{< card link="tool-calling-and-agents" title="Tool Calling, Workflows & AI Agents" subtitle="Model proposals vs execution, bounded loops, real approval, idempotency, persistent state, and what MCP does not solve" >}}
  {{< card link="llm-serving-latency-and-cost" title="LLM Serving, Latency & Cost" subtitle="Prefill vs decoding, streaming, admission-cap quotas, concurrency math, hypothetical pricing, and honest outages" >}}
  {{< card link="multimodal-generative-ai" title="Multimodal Generative AI" subtitle="Optional extension: scanned policies, OCR, image input, speech errors, provenance, privacy, and modality-specific costs" >}}
{{< /cards >}}

## Put the Concepts Together

The four-page [RAG Document Assistant case study]({{% relref "/system-design/rag-document-assistant" %}}) covers requirements, capacity estimates, API/data contracts, iterative architecture, resiliency, and **30 additional questions**. It starts read-only and links back to these concepts instead of requiring an agent framework.

Recommendation systems, feature stores, and ML platforms stay in [AI / ML Systems]({{% relref "/design-concepts/ml" %}}).
