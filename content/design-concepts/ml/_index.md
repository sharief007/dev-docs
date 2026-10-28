---
title: AI / ML Systems
weight: 17
type: docs
toc: false
---

Classical machine-learning system design: recommendations, feature stores, and shared training and serving platforms. These pages assume APIs, storage, and batch jobs, not a language-model track.

Language models, retrieval, RAG, evaluation, safety, agents, and serving are in [Generative AI]({{% relref "/design-concepts/generative-ai" %}}). That section has its own sidebar order. Follow it from foundations rather than starting here.

## Classical ML & Shared Platforms

{{< cards >}}
  {{< card link="recommendation-systems" title="Recommendation System Design" subtitle="Two-stage retrieval + ranking, ANN candidate generation, two-tower embeddings, gradient-boosted ranker, A/B testing, and cold start strategies" >}}
  {{< card link="feature-store" title="Feature Store Design" subtitle="Online vs offline store, point-in-time correctness, training-serving skew, feature freshness trade-offs, and Feast / Tecton / Vertex AI architecture" >}}
  {{< card link="ml-platform" title="ML Platform Design" subtitle="Training pipeline, model registry with lineage, online / batch / streaming serving, drift monitoring, and shadow / canary rollout" >}}
{{< /cards >}}
