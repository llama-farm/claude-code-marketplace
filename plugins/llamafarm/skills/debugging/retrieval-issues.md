# Retrieval Issues

Troubleshoot poor RAG results, missing information, and relevance problems.

## Common Retrieval Issues

### No Results Returned

**Symptoms:**
- RAG queries return empty
- Chat says "I don't have information about that"
- `top_k` chunks but 0 returned

**Diagnostics:**

```bash
# Check database has content
lf rag health

# Test direct query
lf rag query --database main_db --top-k 10 "test"
```

**Solutions:**

1. **Database is empty:**
```bash
# Process datasets first
lf datasets process <dataset>

# Verify chunks exist
lf rag health
```

2. **Wrong database queried:**
```bash
# Check which database has content
lf rag databases

# Query correct database
lf rag query --database <correct-db> "query"
```

3. **Score threshold too high:**
```yaml
# Lower threshold
retrieval_strategies:
  - name: basic_search
    config:
      score_threshold: 0.1  # Lower from default
```

---

### Irrelevant Results

**Symptoms:**
- Results don't match query
- Context confuses the model
- Answers are off-topic

**Diagnostics:**

```bash
# Inspect returned chunks
lf rag query --database main_db "your query" --verbose

# Check chunk content quality
lf rag query --database main_db "your query" | jq '.chunks[0].content'
```

**Solutions:**

1. **Chunk size too large:**
```yaml
# Smaller, more focused chunks
parsers:
  - type: PDFParser_LlamaIndex
    config:
      chunk_size: 400  # Smaller
      chunk_overlap: 50
```

2. **Poor embedding quality:**
```yaml
# Use better embedder
embedding_strategies:
  - name: better_embeddings
    type: OpenAIEmbedder
    config:
      model: text-embedding-3-large
```

3. **Need hybrid search:**
```yaml
# Combine semantic + keyword
retrieval_strategies:
  - name: hybrid_search
    type: HybridUniversalStrategy
    config:
      strategies:
        - type: BasicSimilarityStrategy
          weight: 0.6
      final_k: 10
```

---

### Missing Information

**Symptoms:**
- Specific facts not found
- Document exists but not retrieved
- Partial information only

**Diagnostics:**

```bash
# Search for specific terms
lf rag query --database main_db "exact phrase from document"

# Check if document was indexed
lf datasets list-files <dataset>
```

**Solutions:**

1. **Document not processed:**
```bash
# Check file status
lf datasets status <dataset>

# Reprocess if needed
lf datasets process <dataset>
```

2. **Information split across chunks:**
```yaml
# Increase overlap
parsers:
  - type: PDFParser_LlamaIndex
    config:
      chunk_overlap: 200  # More overlap
```

3. **Increase top_k:**
```bash
# Retrieve more chunks
lf chat --top-k 20 "query"
```

---

### Duplicate Results

**Symptoms:**
- Same content returned multiple times
- Repetitive context
- Wasted context window

**Diagnostics:**

```bash
# Check for duplicates
lf rag query --database main_db "query" | jq '.chunks[].content' | sort | uniq -d
```

**Solutions:**

1. **Too much overlap:**
```yaml
# Reduce overlap
parsers:
  - type: PDFParser_LlamaIndex
    config:
      chunk_overlap: 50  # Less overlap
```

2. **Duplicate files uploaded:**
```bash
# Check for duplicates
lf datasets list-files <dataset>

# Remove duplicates
lf datasets delete-file <dataset> duplicate.pdf
```

3. **Enable deduplication:**
```yaml
retrieval_strategies:
  - name: deduplicated_search
    type: BasicSimilarityStrategy
    config:
      deduplicate: true
      similarity_threshold: 0.95
```

---

### Slow Retrieval

**Symptoms:**
- Queries take seconds
- Chat response delayed
- Timeout errors

**Diagnostics:**

```bash
# Time a query
time lf rag query --database main_db "test"

# Check database size
lf rag health
```

**Solutions:**

1. **Too many chunks:**
```yaml
# Reduce top_k
retrieval_strategies:
  - name: basic_search
    config:
      top_k: 5  # Fewer chunks
```

2. **Large database:**
```yaml
# Use Qdrant for large scale
databases:
  - name: main_db
    type: QdrantStore
    config:
      url: http://localhost:6333
```

3. **Expensive strategy:**
```yaml
# Use simpler strategy
retrieval_strategies:
  - name: fast_search
    type: BasicSimilarityStrategy  # Not reranking
```

---

## Tuning Retrieval

### Chunk Size Guidelines

| Document Type | Chunk Size | Overlap |
|---------------|------------|---------|
| Short docs (<10 pages) | 300-500 | 30-50 |
| Long docs (10-50 pages) | 500-800 | 50-100 |
| Very long docs (50+) | 800-1200 | 100-150 |
| Code | 400-600 | 50-100 |
| Legal/regulatory | 1000-1500 | 150-200 |

### Strategy Selection

| Use Case | Strategy | Why |
|----------|----------|-----|
| General Q&A | BasicSimilarityStrategy | Fast, accurate |
| Keyword-heavy | HybridUniversalStrategy | Semantic + keyword |
| High precision | CrossEncoderRerankedStrategy | Reranking improves relevance |
| Filtered data | MetadataFilteredStrategy | Filter by attributes |

### Score Threshold

| Threshold | Effect |
|-----------|--------|
| 0.0 | Return everything (may be noisy) |
| 0.2-0.3 | Balanced (recommended default) |
| 0.5+ | High precision (may miss relevant) |
| 0.7+ | Very strict (few results) |

---

## Testing Retrieval

### Direct Query Test

```bash
# Test without model
lf rag query --database main_db --top-k 5 "your query"
```

### Relevance Check

```python
# Python script to test relevance
import requests

def test_retrieval(query, expected_terms):
    response = requests.post(
        "http://localhost:8000/v1/projects/default/my-project/rag/query",
        json={"query": query, "top_k": 10}
    )
    chunks = response.json()["chunks"]

    found = []
    for chunk in chunks:
        for term in expected_terms:
            if term.lower() in chunk["content"].lower():
                found.append(term)

    return {
        "query": query,
        "expected": expected_terms,
        "found": list(set(found)),
        "missing": [t for t in expected_terms if t not in found]
    }

# Test
result = test_retrieval(
    "neural network architecture",
    ["layer", "activation", "weights"]
)
print(result)
```

### A/B Testing Strategies

```bash
# Test basic search
lf rag query --database main_db --strategy basic_search "query"

# Test hybrid search
lf rag query --database main_db --strategy hybrid_search "query"

# Compare results
```

---

## Recovery Procedures

### Rebuild Embeddings

```bash
# Clear and reprocess
lf datasets clear <dataset>
lf datasets process <dataset>
```

### Change Embedding Model

```yaml
# Update embedder in config
embedding_strategies:
  - name: better_embeddings
    type: OpenAIEmbedder
    config:
      model: text-embedding-3-large
```

Then reprocess:
```bash
lf datasets clear <dataset>
lf datasets process <dataset>
```
