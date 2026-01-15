# Retrieval Strategies

## Strategy Comparison

| Strategy | Use Case | Complexity | Quality |
|----------|----------|------------|---------|
| BasicSimilarityStrategy | Default, simple queries | Low | Good |
| MetadataFilteredStrategy | Filtered search | Low | Good |
| MultiQueryStrategy | Complex questions | Medium | Better |
| HybridUniversalStrategy | Mixed documents | Medium | Better |
| CrossEncoderRerankedStrategy | High precision | High | Best |
| MultiTurnRAGStrategy | Multi-part queries | High | Best |

## BasicSimilarityStrategy

Simple vector similarity search. Good default choice.

```yaml
retrieval_strategies:
  - name: basic_search
    type: BasicSimilarityStrategy
    config:
      distance_metric: cosine    # cosine, euclidean, dot
      top_k: 10
      score_threshold: 0.5       # Optional minimum score
    default: true
```

**When to use:** Most scenarios, simple Q&A, starting point

## MetadataFilteredStrategy

Filter results by metadata before or after retrieval.

```yaml
retrieval_strategies:
  - name: filtered_search
    type: MetadataFilteredStrategy
    config:
      top_k: 10
      filter_mode: pre       # pre or post
      filters:
        document_type: "legal"
        year: 2024
```

**When to use:** When you need to scope results by metadata

## MultiQueryStrategy

Generates multiple query variations for better recall.

```yaml
retrieval_strategies:
  - name: multi_query
    type: MultiQueryStrategy
    config:
      num_queries: 3
      top_k: 10
      aggregation_method: weighted  # weighted, max, mean, reciprocal_rank
```

**When to use:** Complex questions, ambiguous queries

## HybridUniversalStrategy

Combines multiple strategies with weighted fusion.

```yaml
retrieval_strategies:
  - name: hybrid_search
    type: HybridUniversalStrategy
    config:
      strategies:
        - type: BasicSimilarityStrategy
          config:
            top_k: 8
          weight: 0.6
        - type: MetadataFilteredStrategy
          config:
            top_k: 8
          weight: 0.4
      combination_method: weighted_average  # or rank_fusion
      final_k: 10
    default: true
```

**When to use:** Mixed document types, diverse queries

## CrossEncoderRerankedStrategy

Two-stage retrieval with cross-encoder reranking for highest precision.

```yaml
retrieval_strategies:
  - name: reranked_search
    type: CrossEncoderRerankedStrategy
    config:
      model_name: reranker       # Must reference runtime.models
      initial_k: 30              # Cast wide net
      final_k: 10                # Keep top after reranking
      base_strategy: BasicSimilarityStrategy
      relevance_threshold: 0.3
```

**When to use:** High-stakes queries, legal/compliance

## MultiTurnRAGStrategy

Decomposes complex queries into sub-queries.

```yaml
retrieval_strategies:
  - name: multi_turn
    type: MultiTurnRAGStrategy
    config:
      model_name: default
      max_sub_queries: 3
      complexity_threshold: 50
      sub_query_top_k: 10
      final_top_k: 10
      dedup_similarity_threshold: 0.95
```

**When to use:** Multi-part questions, research queries

## Top-K Guidelines

| Use Case | Top-K | Reasoning |
|----------|-------|-----------|
| Quick Q&A | 3-5 | Fast, focused |
| Research | 8-10 | Comprehensive |
| Legal/Compliance | 10-15 | Maximum coverage |
| Code Search | 5-8 | Balance precision/recall |
| With Reranking | 20-30 → 5-10 | Wide then filter |

## Distance Metrics

| Metric | Description | When to Use |
|--------|-------------|-------------|
| cosine | Angle between vectors | Most common, text |
| euclidean | Absolute distance | Dense embeddings |
| dot | Dot product | Normalized vectors |
