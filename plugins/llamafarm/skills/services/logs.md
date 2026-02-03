# /llamafarm:logs - View Service Logs

View and filter logs from LlamaFarm services for debugging and monitoring.

## Usage

```
/llamafarm:logs                     # Recent logs from all services
/llamafarm:logs --service server    # Logs from specific service
/llamafarm:logs --follow            # Stream logs in real-time
/llamafarm:logs --errors            # Only show errors
/llamafarm:logs --since 1h          # Logs from last hour
```

## What This Command Does

1. **Retrieves logs** - From server, RAG worker, and other components
2. **Filters output** - By service, level, time range
3. **Formats for reading** - Highlights errors, timestamps
4. **Identifies issues** - Common error patterns

## Implementation

When the user runs `/llamafarm:logs`, follow these steps:

### Step 1: Determine Log Source

LlamaFarm logs are typically in:
- Server: stdout/stderr when running in foreground
- Celery: Worker logs
- File: `~/.llamafarm/logs/` (if configured)

```bash
# Check if services are running and get log location
lf services logs --help
```

### Step 2: Retrieve Logs

```bash
# All recent logs
lf services logs

# Specific service
lf services logs --service server
lf services logs --service rag

# Follow mode (real-time)
lf services logs --follow
```

### Step 3: Filter and Format

For `--errors` flag:
```bash
lf services logs | grep -E "(ERROR|CRITICAL|Exception|Traceback)"
```

For `--since` flag:
```bash
lf services logs --since "1 hour ago"
```

### Step 4: Present Results

```
LlamaFarm Logs (last 50 lines)
==============================

[2024-01-14 10:23:45] INFO  [server] Started on http://localhost:14345
[2024-01-14 10:23:46] INFO  [celery] Worker ready, consuming from queue
[2024-01-14 10:25:12] INFO  [rag] Processing dataset: research_papers
[2024-01-14 10:25:15] INFO  [rag] Parsed 5 documents, 127 chunks created
[2024-01-14 10:25:18] INFO  [rag] Embedding batch 1/3...
[2024-01-14 10:25:22] INFO  [rag] Embedding batch 2/3...
[2024-01-14 10:25:26] INFO  [rag] Embedding batch 3/3...
[2024-01-14 10:25:27] INFO  [rag] Ingestion complete: 127 chunks indexed
[2024-01-14 10:30:01] INFO  [chat] Request from session abc123
[2024-01-14 10:30:02] INFO  [rag] Query: "neural scaling laws"
[2024-01-14 10:30:02] INFO  [rag] Retrieved 10 chunks (0.15s)
[2024-01-14 10:30:05] INFO  [chat] Response generated (3.2s)
```

## Service-Specific Logs

### Server Logs

```
/llamafarm:logs --service server
```

Shows:
- HTTP requests and responses
- API errors
- Configuration loading
- Model interactions

### RAG Worker Logs

```
/llamafarm:logs --service rag
```

Shows:
- Document parsing progress
- Chunking details
- Embedding generation
- Vector store operations
- Retrieval queries

### Celery Logs

```
/llamafarm:logs --service celery
```

Shows:
- Task queue status
- Worker health
- Task execution times
- Queue backlogs

## Error Analysis

When errors are found, provide context:

```
Error Found in Logs
===================

[2024-01-14 10:35:12] ERROR [rag] Failed to parse document: contract.pdf
Traceback:
  PDFParser_LlamaIndex: Unable to extract text from scanned PDF

Analysis:
  This PDF appears to be a scanned image without text layer.

Suggested fixes:
1. Use OCR-enabled parser:
   parsers:
     - type: PDFParser_LlamaIndex
       config:
         use_ocr: true

2. Or use dedicated OCR pipeline:
   See /llamafarm:config for OCR-enabled configuration
```

## Common Log Patterns

### Successful Ingestion

```
[INFO] Processing dataset: my_dataset
[INFO] Parsed 10 documents, 450 chunks created
[INFO] Embedding complete: 450 vectors
[INFO] Indexed to main_db
```

### RAG Query

```
[INFO] Query: "search terms"
[INFO] Strategy: BasicSimilarityStrategy
[INFO] Retrieved 10 chunks in 0.12s
[INFO] Top score: 0.87
```

### Model Interaction

```
[INFO] Chat request: model=default, rag=true
[INFO] Context: 5 chunks, 2,340 tokens
[INFO] Response: 512 tokens in 2.1s
```

### Common Errors

| Pattern | Meaning | Fix |
|---------|---------|-----|
| `Connection refused :11434` | Ollama not running | Start Ollama: `ollama serve` |
| `Model not found` | Model not pulled | Pull model: `ollama pull llama3.1:8b` |
| `ChromaDB error` | Vector store issue | Check disk space, permissions |
| `Task timeout` | Slow processing | Increase timeout or reduce batch size |

## Log Levels

| Level | When Used |
|-------|-----------|
| `DEBUG` | Detailed internal operations |
| `INFO` | Normal operations, progress |
| `WARNING` | Non-critical issues |
| `ERROR` | Failures that need attention |
| `CRITICAL` | System-breaking issues |

Filter by level:
```
/llamafarm:logs --level ERROR    # Errors and above
/llamafarm:logs --level WARNING  # Warnings and above
```

## Related

- `status.md` - Overall health check
- `start.md` - Start services
- `stop.md` - Stop services
