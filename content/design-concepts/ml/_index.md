---
title: AI / ML Systems
weight: 16
type: docs
toc: false
---

Learn how AI-backed systems work, from the first model call to a source-backed document assistant. The generative AI track assumes basic APIs and databases, not prior ML mathematics.

**New to generative AI?** Start with the foundations below and follow the cards in order. One fictional ExampleCo policy assistant connects the explanations, diagrams, and examples. Each of the 13 concepts ends with **20 tricky questions and expandable answers**: try answering before opening them.

## Generative AI: Foundations

{{< cards >}}
  {{< card link="generative-ai-fundamentals" title="Generative AI & LLM Fundamentals" subtitle="Generation vs classification, training vs inference, transformer intuition, and why fluent answers can be wrong" >}}
  {{< card link="tokens-context-and-sampling" title="Tokens, Context Windows & Sampling" subtitle="Worked token budgets, conversation history, truncation, temperature, and top-p without confusing randomness with correctness" >}}
  {{< card link="prompt-design-and-structured-outputs" title="Prompt Design & Structured Outputs" subtitle="Clear tasks, few-shot examples, response schemas, business validation, and source-backed output contracts" >}}
  {{< card link="embeddings-and-semantic-search" title="Embeddings & Semantic Search" subtitle="Vectors, cosine similarity, exact-search examples, model compatibility, and semantic vs keyword matching" >}}
  {{< card link="vector-search-and-databases" title="Vector Search & Vector Databases" subtitle="Exact vs approximate search, HNSW intuition, storage arithmetic, filtering, and database extension vs dedicated service" >}}
{{< /cards >}}

## Generative AI: Grounded Answers, Evaluation & Safety

{{< cards >}}
  {{< card link="retrieval-augmented-generation" title="Retrieval-Augmented Generation (RAG)" subtitle="From manually supplied evidence to retrieval, checked citations, missing evidence, and separate retrieval/generation failures" >}}
  {{< card link="rag-ingestion-and-retrieval" title="RAG Ingestion, Retrieval Quality & Freshness" subtitle="Extraction, chunking, hybrid search, reranking, idempotent jobs, atomic version activation, deletion, and revocation" >}}
  {{< card link="prompting-vs-rag-vs-fine-tuning" title="Prompting vs RAG vs Fine-Tuning" subtitle="Match the solution to the failure, understand LoRA, avoid evaluation leakage, and compare complete lifecycle costs" >}}
  {{< card link="generative-ai-evaluation" title="Generative AI Evaluation & Observability" subtitle="Reviewed cases, recall and precision, groundedness, judge limitations, safe traces, and release gates" >}}
  {{< card link="generative-ai-safety" title="GenAI Safety, Prompt Injection & Data Privacy" subtitle="Untrusted content, enforceable permissions, data minimization, safe outputs, provider settings, and resource limits" >}}
{{< /cards >}}

## Generative AI: Tools & Production Trade-offs

{{< cards >}}
  {{< card link="tool-calling-and-agents" title="Tool Calling, Workflows & AI Agents" subtitle="Model proposals vs execution, bounded loops, real approval, idempotency, persistent state, and what MCP does not solve" >}}
  {{< card link="llm-serving-latency-and-cost" title="LLM Serving, Latency & Cost" subtitle="Prefill vs decoding, streaming, quota and concurrency math, hypothetical pricing, caches, and honest outage behavior" >}}
  {{< card link="multimodal-generative-ai" title="Multimodal Generative AI" subtitle="Optional extension: scanned policies, OCR, image input, speech errors, provenance, privacy, and modality-specific costs" >}}
{{< /cards >}}

## Put the Concepts Together

The four-page [RAG Document Assistant case study]({{% relref "/system-design/rag-document-assistant" %}}) covers requirements, capacity estimates, API/data contracts, iterative architecture, resiliency, and **30 additional questions**. It starts read-only and links back to these concepts instead of requiring an agent framework.

## Classical ML & Shared Platforms

These existing topics remain useful for recommendation-focused and ML-platform roles.

{{< cards >}}
  {{< card link="recommendation-systems" title="Recommendation System Design" subtitle="Two-stage retrieval + ranking, ANN candidate generation, two-tower embeddings, gradient-boosted ranker, A/B testing, and cold start strategies" >}}
  {{< card link="feature-store" title="Feature Store Design" subtitle="Online vs offline store, point-in-time correctness, training-serving skew, feature freshness trade-offs, and Feast / Tecton / Vertex AI architecture" >}}
  {{< card link="ml-platform" title="ML Platform Design" subtitle="Training pipeline, model registry with lineage, online / batch / streaming serving, drift monitoring, and shadow / canary rollout" >}}
{{< /cards >}}
