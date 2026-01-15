# Performance Optimization

Optimize LlamaFarm for speed, memory efficiency, and scalability.

## Performance Bottlenecks

### Common Issues

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| Slow chat | Large context | Reduce top_k |
| Slow embedding | CPU embedding | Use GPU/batch |
| High memory | Large chunks | Reduce chunk size |
| Slow queries | Large database | Add filtering |

---

## Chat Performance

### Reduce Context Size

```yaml
# Fewer chunks = faster
retrieval_strategies:
  - name: basic_search
    config:
      top_k: 5  # Down from 10

# Smaller chunks = less tokens
parsers:
  - type: PDFParser_LlamaIndex
    config:
      chunk_size: 400  # Down from 800
```

### Use Faster Models

```yaml
runtime:
  models:
    # Fast model for simple queries
    - name: fast
      provider: ollama
      model: llama3.2:1b  # Smaller = faster
      model_api_parameters:
        max_tokens: 512

    # Default for complex queries
    - name: default
      provider: ollama
      model: llama3.1:8b
```

### Enable Streaming

```python
# Stream responses for perceived speed
response = requests.post(
    "http://localhost:8000/v1/chat/completions",
    json={"messages": [...], "stream": True},
    stream=True
)
```

---

## Embedding Performance

### Batch Processing

```yaml
embedding_strategies:
  - name: default_embeddings
    type: UniversalEmbedder
    config:
      batch_size: 64  # Larger batches
```

### Use GPU

```bash
# Ensure CUDA is available
python -c "import torch; print(torch.cuda.is_available())"

# Configure for GPU
CUDA_VISIBLE_DEVICES=0 lf start
```

### Faster Embedding Model

```yaml
# Fast local embeddings
embedding_strategies:
  - name: fast_embeddings
    type: OllamaEmbedder
    config:
      model: nomic-embed-text  # Fast

# Or use Universal Runtime
embedding_strategies:
  - name: universal_embeddings
    type: UniversalEmbedder
    config:
      model: nomic-ai/nomic-embed-text-v2-moe
```

---

## Database Performance

### Index Optimization

For large databases (100k+ chunks):

```yaml
# Use Qdrant for scale
databases:
  - name: main_db
    type: QdrantStore
    config:
      url: http://localhost:6333
      collection_name: documents
      vector_params:
        size: 768
        distance: Cosine
```

### Metadata Filtering

Pre-filter before vector search:

```yaml
retrieval_strategies:
  - name: filtered_search
    type: MetadataFilteredStrategy
    config:
      pre_filter:
        source: {"$contains": ".pdf"}
```

### Limit Search Scope

```python
# Filter to specific dataset
results = query_rag(
    "query",
    filters={"dataset": "research_papers"}
)
```

---

## Memory Optimization

### Reduce Chunk Size

```yaml
parsers:
  - type: PDFParser_LlamaIndex
    config:
      chunk_size: 400  # Smaller chunks
      chunk_overlap: 40
```

### Limit Concurrent Processing

```bash
# Fewer workers = less memory
CELERY_WORKERS=1 lf start
```

### Process in Batches

```bash
# Upload and process in batches
for batch in batch1 batch2 batch3; do
    lf datasets upload <dataset> ./$batch/*
    lf datasets process <dataset>
    # Wait for completion
done
```

### Reduce Extractors

```yaml
# Fewer extractors = less memory
extractors:
  - type: ContentStatisticsExtractor  # Lightweight
  # Remove expensive ones like EntityExtractor
```

---

## Ingestion Performance

### Parallel Processing

```yaml
# Process multiple files concurrently
data_processing_strategies:
  - name: parallel_processor
    config:
      max_workers: 4
```

### Skip Expensive Extraction

```yaml
# Minimal extractors for speed
extractors:
  - type: ContentStatisticsExtractor
  # Skip: EntityExtractor, SummaryExtractor
```

### Optimize Chunk Strategy

| Strategy | Speed | Quality |
|----------|-------|---------|
| `fixed` | Fastest | Lower |
| `paragraphs` | Fast | Good |
| `semantic` | Slow | Best |
| `headings` | Medium | Good |

---

## Network Performance

### Local vs Remote

| Setup | Latency | Best For |
|-------|---------|----------|
| Local Ollama | ~50ms | Development |
| Local vLLM | ~30ms | Production |
| Remote OpenAI | ~500ms | Quality |

### Connection Pooling

```python
# Reuse HTTP sessions
session = requests.Session()
adapter = HTTPAdapter(pool_connections=10, pool_maxsize=10)
session.mount('http://', adapter)
```

### Async Requests

```python
import asyncio
import httpx

async def parallel_queries(queries):
    async with httpx.AsyncClient() as client:
        tasks = [
            client.post(url, json={"query": q})
            for q in queries
        ]
        return await asyncio.gather(*tasks)
```

---

## Monitoring Performance

### Response Time Tracking

```python
import time

start = time.time()
response = chat("query")
duration = time.time() - start
print(f"Response time: {duration:.2f}s")
```

### Resource Monitoring

```bash
# CPU and memory
top -l 1 | head -20

# GPU utilization (if using)
nvidia-smi

# Disk I/O
iostat -x 1
```

### Log Analysis

```bash
# Find slow operations
lf services logs | grep -E "took [0-9]+\.[0-9]+ seconds"

# Find errors
lf services logs | grep -E "ERROR|timeout"
```

---

## Benchmarks

### Chat Latency Targets

| Model Size | Target | Acceptable |
|------------|--------|------------|
| 1B params | <1s | <2s |
| 8B params | <3s | <5s |
| 70B params | <10s | <20s |

### Embedding Throughput

| Model | Chunks/sec | Notes |
|-------|------------|-------|
| nomic-embed-text | 50-100 | Ollama |
| text-embedding-3-small | 20-50 | OpenAI |
| Universal Runtime | 100-200 | Local GPU |

### Query Latency

| Database Size | Target | Acceptable |
|---------------|--------|------------|
| <10k chunks | <100ms | <200ms |
| 10k-100k chunks | <200ms | <500ms |
| 100k+ chunks | <500ms | <1s |

---

## Scaling Recommendations

### Small (Dev/Test)

```yaml
# Single model, local everything
runtime:
  models:
    - name: default
      provider: ollama
      model: llama3.2:1b

databases:
  - type: ChromaStore
```

### Medium (Team/Production)

```yaml
# Multiple models, optimized settings
runtime:
  models:
    - name: fast
      provider: ollama
      model: llama3.2:1b
    - name: default
      provider: ollama
      model: llama3.1:8b

databases:
  - type: QdrantStore
```

### Large (Enterprise)

```yaml
# Distributed, high-performance
runtime:
  models:
    - name: default
      provider: openai
      model: gpt-4-turbo

databases:
  - type: QdrantStore
    config:
      url: http://qdrant-cluster:6333
      replication_factor: 3
```
