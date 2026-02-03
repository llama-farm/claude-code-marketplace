# RAG Query API

Programmatic access to LlamaFarm's vector database for semantic search and retrieval.

## Overview

The RAG API allows you to:
- Query vector databases directly
- Apply filters and strategies
- Retrieve document chunks with metadata
- Check database health and statistics

## Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/rag/query` | Query documents |
| GET | `/rag/health` | Database health |
| GET | `/rag/databases` | List databases |
| GET | `/rag/databases/{name}` | Database details |

All endpoints are under `/v1/projects/{namespace}/{project}/`.

---

## Query Documents

```http
POST /v1/projects/{namespace}/{project}/rag/query
Content-Type: application/json
```

### Basic Query

**Request:**
```json
{
  "query": "What are neural scaling laws?",
  "database": "main_db",
  "top_k": 10
}
```

**Response:**
```json
{
  "chunks": [
    {
      "id": "chunk-abc123",
      "content": "Neural scaling laws describe the relationship between model size, compute, and performance...",
      "score": 0.892,
      "metadata": {
        "source": "neural_scaling.md",
        "heading": "Introduction",
        "chunk_index": 3,
        "entities": ["neural scaling", "compute"],
        "statistics": {
          "word_count": 85,
          "readability_score": 42.5
        }
      }
    },
    {
      "id": "chunk-def456",
      "content": "Key findings from scaling research include...",
      "score": 0.845,
      "metadata": {
        "source": "ml_research.md",
        "heading": "Scaling Laws",
        "chunk_index": 7
      }
    }
  ],
  "query_time_ms": 45,
  "total_chunks_searched": 2340
}
```

### Python Examples

**Basic Query:**
```python
import requests

def query_rag(query, database="main_db", top_k=10, project="my-project"):
    response = requests.post(
        f"http://localhost:14345/v1/projects/default/{project}/rag/query",
        json={
            "query": query,
            "database": database,
            "top_k": top_k
        }
    )
    response.raise_for_status()
    return response.json()

# Simple query
results = query_rag("What are scaling laws?")
for chunk in results["chunks"]:
    print(f"Score: {chunk['score']:.2f} - {chunk['metadata']['source']}")
    print(f"  {chunk['content'][:100]}...")
```

**With Score Threshold:**
```python
results = query_rag(
    query="machine learning basics",
    top_k=20,
    score_threshold=0.5  # Only return chunks with score >= 0.5
)
```

---

## Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `query` | string | required | Search query text |
| `database` | string | default | Database to query |
| `strategy` | string | default | Retrieval strategy |
| `top_k` | int | 10 | Max chunks to return |
| `score_threshold` | float | 0.0 | Min relevance score |
| `filters` | object | null | Metadata filters |
| `include_metadata` | bool | true | Include chunk metadata |
| `include_embeddings` | bool | false | Include raw embeddings |

---

## Retrieval Strategies

Specify different strategies for different use cases:

### Basic Similarity

```python
results = query_rag(
    query="neural networks",
    strategy="basic_search"
)
```

### Metadata Filtered

```python
results = requests.post(
    f"{base_url}/rag/query",
    json={
        "query": "implementation details",
        "strategy": "metadata_filtered",
        "filters": {
            "source": {"$contains": "technical"}
        }
    }
).json()
```

### Hybrid Search

Combines semantic and keyword search:

```python
results = requests.post(
    f"{base_url}/rag/query",
    json={
        "query": "API authentication",
        "strategy": "hybrid_search"
    }
).json()
```

---

## Metadata Filters

Filter chunks by metadata fields:

### Exact Match

```json
{
  "filters": {
    "source": "research_paper.pdf"
  }
}
```

### Contains

```json
{
  "filters": {
    "source": {"$contains": "research"}
  }
}
```

### In List

```json
{
  "filters": {
    "entity_type": {"$in": ["ORG", "PERSON"]}
  }
}
```

### Numeric Comparison

```json
{
  "filters": {
    "page": {"$gte": 10, "$lte": 50}
  }
}
```

### Combined Filters

```json
{
  "filters": {
    "$and": [
      {"source": {"$contains": ".pdf"}},
      {"entity_ORG": {"$exists": true}}
    ]
  }
}
```

### Python Example with Filters

```python
def query_by_source(query, source_pattern, project="my-project"):
    response = requests.post(
        f"http://localhost:14345/v1/projects/default/{project}/rag/query",
        json={
            "query": query,
            "filters": {
                "source": {"$contains": source_pattern}
            },
            "top_k": 10
        }
    )
    return response.json()

# Find relevant chunks only from PDF files
results = query_by_source("compliance requirements", ".pdf")

# Find chunks from a specific document
results = query_by_source("introduction", "annual_report")
```

---

## Database Health

Check database status and statistics:

```http
GET /v1/projects/{namespace}/{project}/rag/health
```

**Response:**
```json
{
  "status": "healthy",
  "databases": [
    {
      "name": "main_db",
      "type": "ChromaStore",
      "status": "healthy",
      "stats": {
        "document_count": 150,
        "chunk_count": 3456,
        "total_size_mb": 125.4,
        "last_updated": "2024-01-14T12:00:00Z"
      }
    }
  ]
}
```

**Python Example:**
```python
def check_rag_health(project="my-project"):
    response = requests.get(
        f"http://localhost:14345/v1/projects/default/{project}/rag/health"
    )
    health = response.json()

    print(f"RAG Status: {health['status']}")
    for db in health["databases"]:
        print(f"  {db['name']}: {db['stats']['chunk_count']} chunks")

    return health

health = check_rag_health()
```

---

## List Databases

```http
GET /v1/projects/{namespace}/{project}/rag/databases
```

**Response:**
```json
{
  "databases": [
    {
      "name": "main_db",
      "type": "ChromaStore",
      "default": true,
      "embedding_strategy": "universal_embeddings",
      "retrieval_strategies": ["basic_search", "hybrid_search"]
    },
    {
      "name": "archive_db",
      "type": "ChromaStore",
      "default": false,
      "embedding_strategy": "universal_embeddings",
      "retrieval_strategies": ["basic_search"]
    }
  ]
}
```

---

## Advanced Usage

### Batch Queries

```python
def batch_query(queries, database="main_db", project="my-project"):
    results = []
    for query in queries:
        result = requests.post(
            f"http://localhost:14345/v1/projects/default/{project}/rag/query",
            json={"query": query, "database": database, "top_k": 5}
        ).json()
        results.append({
            "query": query,
            "top_chunk": result["chunks"][0] if result["chunks"] else None
        })
    return results

queries = [
    "What is machine learning?",
    "How do neural networks work?",
    "What are transformers?"
]

results = batch_query(queries)
```

### Query with Context Building

```python
def get_context_for_llm(query, max_tokens=2000, project="my-project"):
    """Retrieve chunks and format as context for LLM."""
    results = requests.post(
        f"http://localhost:14345/v1/projects/default/{project}/rag/query",
        json={"query": query, "top_k": 10}
    ).json()

    context_parts = []
    total_chars = 0
    max_chars = max_tokens * 4  # Rough estimate

    for chunk in results["chunks"]:
        if total_chars + len(chunk["content"]) > max_chars:
            break

        source = chunk["metadata"].get("source", "unknown")
        context_parts.append(f"[Source: {source}]\n{chunk['content']}")
        total_chars += len(chunk["content"])

    return "\n\n---\n\n".join(context_parts)

context = get_context_for_llm("neural scaling laws")
print(f"Context length: {len(context)} characters")
```

### Multi-Database Query

```python
def query_multiple_databases(query, databases, project="my-project"):
    """Query multiple databases and combine results."""
    all_chunks = []

    for db in databases:
        results = requests.post(
            f"http://localhost:14345/v1/projects/default/{project}/rag/query",
            json={"query": query, "database": db, "top_k": 5}
        ).json()

        for chunk in results["chunks"]:
            chunk["database"] = db
            all_chunks.append(chunk)

    # Sort by score across all databases
    all_chunks.sort(key=lambda x: x["score"], reverse=True)
    return all_chunks[:10]  # Top 10 overall

results = query_multiple_databases(
    "API documentation",
    ["main_db", "archive_db"]
)
```

---

## Error Handling

```python
def safe_query(query, project="my-project"):
    try:
        response = requests.post(
            f"http://localhost:14345/v1/projects/default/{project}/rag/query",
            json={"query": query},
            timeout=30
        )
        response.raise_for_status()
        return response.json()

    except requests.exceptions.HTTPError as e:
        if e.response.status_code == 404:
            return {"error": "Database not found"}
        error = e.response.json().get("error", {})
        return {"error": error.get("message", str(e))}

    except requests.exceptions.Timeout:
        return {"error": "Query timed out - database may be overloaded"}

    except Exception as e:
        return {"error": str(e)}
```

Common errors:
| Code | Cause | Solution |
|------|-------|----------|
| `database_not_found` | Invalid database name | Check database in config |
| `strategy_not_found` | Invalid strategy | Use configured strategies |
| `index_empty` | No documents indexed | Upload and process documents |
| `query_too_long` | Query exceeds limit | Shorten query text |
