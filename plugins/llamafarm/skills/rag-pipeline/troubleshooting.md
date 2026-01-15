# RAG Troubleshooting

## Common Issues

### Low Quality Results

**Symptoms:** Irrelevant or wrong answers despite correct documents.

**Fixes:**
1. **Increase top_k** - Cast a wider net
   ```yaml
   config:
     top_k: 15  # Up from 5
   ```

2. **Reduce chunk_size** - More precise chunks
   ```yaml
   config:
     chunk_size: 600  # Down from 1000
   ```

3. **Add reranking** - Better relevance scoring
   ```yaml
   retrieval_strategies:
     - type: CrossEncoderRerankedStrategy
       config:
         initial_k: 30
         final_k: 10
   ```

4. **Check embedding model** - Use higher quality model
   ```yaml
   config:
     model: sentence-transformers/all-mpnet-base-v2
   ```

### Missing Information

**Symptoms:** "I don't have information about X" when it exists.

**Fixes:**
1. **Check parser** - Verify file type is being parsed
   ```bash
   lf datasets list  # Check file counts
   ```

2. **Increase chunk_overlap** - Better context preservation
   ```yaml
   config:
     chunk_overlap: 200  # Up from 100
   ```

3. **Add extractors** - Capture metadata for filtering
   ```yaml
   extractors:
     - type: EntityExtractor
     - type: KeywordExtractor
   ```

4. **Use MultiQueryStrategy** - Broader search
   ```yaml
   retrieval_strategies:
     - type: MultiQueryStrategy
       config:
         num_queries: 5
   ```

### Slow Performance

**Symptoms:** Long response times, timeouts.

**Fixes:**
1. **Reduce top_k** - Fewer results to process
   ```yaml
   config:
     top_k: 5
   ```

2. **Increase chunk_size** - Fewer chunks to embed
   ```yaml
   config:
     chunk_size: 1200
   ```

3. **Increase batch_size** (if GPU available)
   ```yaml
   config:
     batch_size: 32
   ```

4. **Use lighter embedding model**
   ```yaml
   config:
     model: sentence-transformers/all-MiniLM-L6-v2
   ```

### Duplicate/Redundant Results

**Symptoms:** Same information appearing multiple times.

**Fixes:**
1. **Use HybridUniversalStrategy** - Built-in deduplication
   ```yaml
   retrieval_strategies:
     - type: HybridUniversalStrategy
   ```

2. **Reduce chunk_overlap**
   ```yaml
   config:
     chunk_overlap: 50  # Down from 150
   ```

3. **Increase similarity threshold** for dedup
   ```yaml
   config:
     dedup_similarity_threshold: 0.9
   ```

### Dataset Processing Stuck

**Symptoms:** `lf datasets process` never completes.

**Fixes:**
1. **Check RAG service**
   ```bash
   lf services status
   ```

2. **Check logs**
   ```bash
   lf services logs --service rag
   ```

3. **Restart services**
   ```bash
   lf services stop
   lf start
   ```

4. **Process smaller batches**
   ```bash
   lf datasets upload data batch1/*
   lf datasets process data
   lf datasets upload data batch2/*
   lf datasets process data
   ```

## Diagnostic Commands

```bash
# Check service health
lf services status

# View RAG logs
lf services logs --service rag

# Test retrieval directly
lf rag query --database main_db --top-k 10 --include-score "test query"

# Check dataset stats
lf rag stats

# List datasets with details
lf datasets list
```

## Service Dependencies

| Service | Purpose | Impact if Down |
|---------|---------|----------------|
| server | API gateway | All operations fail |
| rag | Celery worker | Dataset processing fails |
| universal-runtime | Embeddings, inference | Embedding fails |
| chroma | Vector store | Queries fail |

## Log Locations

```bash
# Service logs
~/.llamafarm/logs/server.log
~/.llamafarm/logs/rag.log
~/.llamafarm/logs/universal-runtime.log

# Or via CLI
lf services logs
```

## Quick Health Check

```bash
# 1. Services running?
lf services status

# 2. Config valid?
lf projects validate

# 3. Datasets exist?
lf datasets list

# 4. RAG working?
lf rag query --database main_db --top-k 3 "test"
```
