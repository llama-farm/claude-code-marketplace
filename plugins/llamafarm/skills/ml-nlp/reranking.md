# Reranking

## Endpoint

```http
POST /v1/rerank
Content-Type: application/json
```

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `query` | string | required | The search query to rerank against |
| `documents` | array | required | List of document strings to rerank |
| `top_k` | int | `10` | Number of top results to return |
| `model` | string | `cross-encoder/ms-marco-MiniLM-L-6-v2` | Reranking model to use |
| `return_documents` | boolean | `true` | Include document text in response |

## Basic Usage

```bash
curl -X POST http://localhost:14345/v1/rerank \
  -H "Content-Type: application/json" \
  -d '{
    "query": "How to configure authentication?",
    "documents": [
      "Authentication is configured via the auth section in llamafarm.yaml.",
      "The logging system supports multiple output formats and destinations.",
      "Set up OAuth by adding provider credentials to the auth config.",
      "Database connections are managed through the storage configuration.",
      "API keys can be rotated using the admin endpoint."
    ],
    "top_k": 3
  }'
```

## Response Format

```json
{
  "results": [
    {
      "index": 0,
      "document": "Authentication is configured via the auth section in llamafarm.yaml.",
      "score": 0.92
    },
    {
      "index": 2,
      "document": "Set up OAuth by adding provider credentials to the auth config.",
      "score": 0.87
    },
    {
      "index": 4,
      "document": "API keys can be rotated using the admin endpoint.",
      "score": 0.61
    }
  ],
  "model": "cross-encoder/ms-marco-MiniLM-L-6-v2",
  "processing_time_ms": 35
}
```

Results are sorted by score in descending order. The `index` field refers to the position in the original `documents` array.

## Model Selection

| Model | Speed | Quality | Memory | Best For |
|-------|-------|---------|--------|----------|
| `cross-encoder/ms-marco-MiniLM-L-6-v2` | Fast (~15ms) | Good | ~80 MB | Low-latency production use |
| `BAAI/bge-reranker-base` | Medium (~30ms) | High | ~400 MB | Balanced quality and speed |
| `BAAI/bge-reranker-large` | Slow (~80ms) | Very High | ~1.2 GB | Maximum quality, research |
| `cross-encoder/ms-marco-TinyBERT-L-2-v2` | Very Fast (~5ms) | Moderate | ~20 MB | Ultra-low latency, edge deployment |

### Choosing a Model

- **Production, low latency:** `cross-encoder/ms-marco-MiniLM-L-6-v2` -- best speed/quality ratio
- **Quality-sensitive applications:** `BAAI/bge-reranker-base` -- noticeably better ranking
- **Research, maximum accuracy:** `BAAI/bge-reranker-large` -- best ranking quality
- **High throughput / edge:** `cross-encoder/ms-marco-TinyBERT-L-2-v2` -- minimal latency

## Integration with RAG Pipeline

Reranking is most valuable as a second stage after initial vector retrieval. The pattern is:

```
Vector search (top-50) → Rerank (top-10) → LLM context
```

### Python RAG + Rerank Pipeline

```python
import requests

def rag_with_rerank(query, project="my-project", initial_k=50, final_k=10):
    # Step 1: Retrieve candidates from vector store
    rag_response = requests.post(
        f"http://localhost:14345/v1/projects/default/{project}/rag/query",
        json={
            "query": query,
            "top_k": initial_k
        }
    ).json()

    documents = [chunk["content"] for chunk in rag_response["chunks"]]

    # Step 2: Rerank for relevance
    rerank_response = requests.post(
        "http://localhost:14345/v1/rerank",
        json={
            "query": query,
            "documents": documents,
            "top_k": final_k,
            "model": "BAAI/bge-reranker-base"
        }
    ).json()

    # Step 3: Build reranked context
    reranked_chunks = []
    for result in rerank_response["results"]:
        original_chunk = rag_response["chunks"][result["index"]]
        original_chunk["rerank_score"] = result["score"]
        reranked_chunks.append(original_chunk)

    return reranked_chunks
```

### Using Reranked Context in Chat

```python
import requests

def chat_with_reranked_context(query, project="my-project"):
    # Get reranked chunks
    chunks = rag_with_rerank(query, project, initial_k=50, final_k=5)

    # Build context string
    context = "\n\n---\n\n".join(
        f"[Source: {c['metadata'].get('source', 'unknown')}]\n{c['content']}"
        for c in chunks
    )

    # Chat with reranked context
    response = requests.post(
        f"http://localhost:14345/v1/projects/default/{project}/chat/completions",
        json={
            "messages": [
                {"role": "system", "content": f"Answer using this context:\n\n{context}"},
                {"role": "user", "content": query}
            ]
        }
    ).json()

    return response["choices"][0]["message"]["content"]
```

## Performance Characteristics

Reranking latency scales linearly with document count.

| Documents | MiniLM-L-6-v2 | bge-reranker-base | bge-reranker-large |
|-----------|---------------|-------------------|-------------------|
| 10 | ~15 ms | ~30 ms | ~80 ms |
| 25 | ~30 ms | ~65 ms | ~180 ms |
| 50 | ~55 ms | ~120 ms | ~350 ms |
| 100 | ~105 ms | ~230 ms | ~700 ms |
| 200 | ~210 ms | ~460 ms | ~1400 ms |

**Guidelines:**
- Keep document count under 100 for interactive use
- For large candidate sets, increase `initial_k` from vector search but cap at 200 for reranking
- Use the fastest model (`MiniLM-L-6-v2`) when reranking more than 50 documents

## When Reranking Helps Most

| Scenario | Impact | Reason |
|----------|--------|--------|
| Diverse retrieval results | High | Cross-encoder distinguishes relevant from tangentially related |
| Multi-topic queries | High | Embedding similarity may match wrong topic; reranker catches nuance |
| Short queries (1-3 words) | High | Embeddings lack context; reranker evaluates full query-document pairs |
| Long, specific queries | Medium | Embeddings already capture specificity, but reranker refines |
| Homogeneous corpus | Low | All documents are similarly relevant, little to differentiate |
| Exact keyword match needed | Low | Better handled by keyword/BM25 search |

## Score Interpretation

Reranking scores are not probabilities. They are model-specific relevance scores.

| Score Range | Interpretation | Action |
|-------------|---------------|--------|
| > 0.8 | Highly relevant | Include in context |
| 0.5 - 0.8 | Likely relevant | Include if context window allows |
| 0.2 - 0.5 | Marginally relevant | Include only if few high-confidence results |
| < 0.2 | Likely irrelevant | Exclude from context |

Scores can be negative for some models (especially cross-encoder models). Compare relative ordering rather than absolute values.

## Combining with Other Features

### Rerank then NER Pipeline

Extract entities from only the most relevant documents.

```python
import requests

def rerank_then_ner(query, documents):
    # Step 1: Rerank to find most relevant docs
    rerank_response = requests.post(
        "http://localhost:14345/v1/rerank",
        json={
            "query": query,
            "documents": documents,
            "top_k": 5
        }
    ).json()

    # Step 2: Extract entities from top results only
    entities_by_doc = []
    for result in rerank_response["results"]:
        ner_response = requests.post(
            "http://localhost:14345/v1/ner",
            json={
                "text": result["document"],
                "confidence_threshold": 0.6
            }
        ).json()

        entities_by_doc.append({
            "document_index": result["index"],
            "rerank_score": result["score"],
            "entities": ner_response["entities"]
        })

    return entities_by_doc
```

### Full Pipeline: RAG Retrieve, Rerank, NER, Classify

```python
import requests
from collections import Counter

def full_nlp_pipeline(query, project="my-project"):
    # Retrieve
    chunks = requests.post(
        f"http://localhost:14345/v1/projects/default/{project}/rag/query",
        json={"query": query, "top_k": 50}
    ).json()["chunks"]

    documents = [c["content"] for c in chunks]

    # Rerank
    reranked = requests.post(
        "http://localhost:14345/v1/rerank",
        json={"query": query, "documents": documents, "top_k": 10}
    ).json()["results"]

    # NER on top results
    all_entities = []
    for result in reranked:
        ner = requests.post(
            "http://localhost:14345/v1/ner",
            json={"text": result["document"], "confidence_threshold": 0.5}
        ).json()
        all_entities.extend(ner["entities"])

    # Summarize
    entity_counts = Counter(
        (e["type"], e["text"]) for e in all_entities
    )

    return {
        "top_documents": len(reranked),
        "total_entities": len(all_entities),
        "top_entities": [
            {"type": t, "text": txt, "count": count}
            for (t, txt), count in entity_counts.most_common(10)
        ]
    }
```

## Error Handling

| Error | HTTP Status | Cause | Resolution |
|-------|-------------|-------|------------|
| `invalid_request` | 400 | Missing `query` or `documents` | Provide both required fields |
| `empty_documents` | 400 | Empty documents array | Provide at least one document |
| `model_not_found` | 404 | Requested model not available | Use a supported model |
| `documents_too_many` | 400 | Exceeded max document count (default 1000) | Reduce document count or pre-filter |
| `processing_error` | 500 | Model inference failed | Check server logs, try a different model |

**Example error response:**

```json
{
  "error": {
    "code": "documents_too_many",
    "message": "Document count 1500 exceeds maximum of 1000. Pre-filter or reduce the candidate set.",
    "details": {
      "max_documents": 1000,
      "received_documents": 1500
    }
  }
}
```
