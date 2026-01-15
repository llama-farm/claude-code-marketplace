# Retrieval and Embedding Strategies Reference

## Retrieval Strategies

| Type | Purpose | When to Use |
|------|---------|-------------|
| `BasicSimilarityStrategy` | Simple vector search | Default, single query |
| `MetadataFilteredStrategy` | Filtered search | Filtering by metadata |
| `MultiQueryStrategy` | Query variations | Complex questions |
| `HybridUniversalStrategy` | Combined strategies | Mixed document types |
| `CrossEncoderRerankedStrategy` | Reranked results | High precision needed |
| `MultiTurnRAGStrategy` | Query decomposition | Multi-part queries |
| `RerankedStrategy` | Basic reranking | Improve relevance |

## Retrieval Strategy Configuration

### BasicSimilarityStrategy
```yaml
retrieval_strategies:
  - name: basic_search
    type: BasicSimilarityStrategy
    config:
      distance_metric: cosine    # cosine, euclidean, dot
      top_k: 10
      score_threshold: 0.5       # optional
    default: true
```

### HybridUniversalStrategy
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
      combination_method: weighted_average
      final_k: 10
    default: true
```

### CrossEncoderRerankedStrategy
```yaml
retrieval_strategies:
  - name: reranked_search
    type: CrossEncoderRerankedStrategy
    config:
      model_name: reranker       # references runtime.models
      initial_k: 30
      final_k: 10
      base_strategy: BasicSimilarityStrategy
```

## Embedding Strategies

| Type | Purpose | When to Use |
|------|---------|-------------|
| `UniversalEmbedder` | LlamaFarm Universal Runtime | Local, recommended |
| `OllamaEmbedder` | Ollama embeddings | When using Ollama |
| `OpenAIEmbedder` | OpenAI embeddings | Cloud, high quality |
| `HuggingFaceEmbedder` | HuggingFace models | Custom models |
| `SentenceTransformerEmbedder` | Sentence Transformers | Local, lightweight |

## Embedding Strategy Configuration

### UniversalEmbedder (Recommended)
```yaml
embedding_strategies:
  - name: default_embeddings
    type: UniversalEmbedder
    priority: 0
    config:
      model: nomic-ai/nomic-embed-text-v2-moe
      dimension: 768
      batch_size: 16
      timeout: 60
      auto_pull: true
```

### OllamaEmbedder
```yaml
embedding_strategies:
  - name: ollama_embeddings
    type: OllamaEmbedder
    config:
      model: nomic-embed-text
      base_url: http://localhost:11434
      dimension: 768
```

## Database Types

| Type | Purpose | When to Use |
|------|---------|-------------|
| `ChromaStore` | Local vector database | Development, single machine |
| `QdrantStore` | Distributed vector database | Production, scaling |

### ChromaStore Configuration
```yaml
databases:
  - name: main_db
    type: ChromaStore
    config:
      collection_name: documents
      distance_function: cosine
      persist_directory: ./data/main_db
      port: 8000
```
