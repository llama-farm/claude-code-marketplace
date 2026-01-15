# Embedding Models

## Available Embedders

### UniversalEmbedder (Recommended)
Uses LlamaFarm Universal Runtime with sentence-transformers.

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

**Popular models:**
| Model | Dimensions | Speed | Quality |
|-------|------------|-------|---------|
| nomic-ai/nomic-embed-text-v2-moe | 768 | Medium | High |
| sentence-transformers/all-MiniLM-L6-v2 | 384 | Fast | Good |
| sentence-transformers/all-mpnet-base-v2 | 768 | Medium | High |
| BAAI/bge-base-en-v1.5 | 768 | Medium | High |

### OllamaEmbedder
Uses Ollama's embedding endpoint.

```yaml
embedding_strategies:
  - name: ollama_embeddings
    type: OllamaEmbedder
    config:
      model: nomic-embed-text
      base_url: http://localhost:11434
      dimension: 768
```

### OpenAIEmbedder
Uses OpenAI's embedding API.

```yaml
embedding_strategies:
  - name: openai_embeddings
    type: OpenAIEmbedder
    config:
      model: text-embedding-3-small
      api_key: ${OPENAI_API_KEY}
      dimension: 1536
```

**OpenAI models:**
| Model | Dimensions | Cost |
|-------|------------|------|
| text-embedding-3-small | 1536 | Low |
| text-embedding-3-large | 3072 | Medium |
| text-embedding-ada-002 | 1536 | Low |

### SentenceTransformerEmbedder
Direct sentence-transformers integration.

```yaml
embedding_strategies:
  - name: st_embeddings
    type: SentenceTransformerEmbedder
    config:
      model_name: all-MiniLM-L6-v2
      device: cuda  # or cpu
```

## Performance Tuning

### Batch Size
```yaml
config:
  batch_size: 32    # Increase for GPU
  batch_size: 8     # Decrease for CPU/memory constraints
```

### Timeout
```yaml
config:
  timeout: 120      # Increase for large batches
```

### GPU Usage
```yaml
config:
  device: cuda      # Use GPU
  device: cpu       # CPU only
```

## Choosing an Embedder

| Scenario | Recommended Embedder |
|----------|---------------------|
| Local development | UniversalEmbedder |
| Using Ollama already | OllamaEmbedder |
| Production/cloud | OpenAIEmbedder |
| Custom models | SentenceTransformerEmbedder |

## Dimension Matching

**Important:** Embedding dimension must match database configuration.

```yaml
# Embedder
embedding_strategies:
  - name: default_embeddings
    config:
      dimension: 768    # Must match

# Database (if explicitly configured)
databases:
  - name: main_db
    config:
      vector_size: 768  # Must match
```
