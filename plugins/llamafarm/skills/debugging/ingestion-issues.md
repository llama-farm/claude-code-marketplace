# Ingestion Issues

Troubleshoot document processing and dataset ingestion failures.

## Common Ingestion Issues

### Processing Stuck

**Symptoms:**
- `lf datasets process` hangs
- Progress stays at 0%
- No new chunks in database

**Diagnostics:**

```bash
# Check dataset status
lf datasets status <dataset-name>

# Check worker status
lf services status | grep worker

# Check RAG logs
lf services logs --service rag
```

**Solutions:**

1. **Worker not running:**
```bash
# Restart services
lf services stop
lf start
```

2. **Large files blocking queue:**
```bash
# Process smaller batches
lf datasets upload <dataset> ./small_batch/*
lf datasets process <dataset>
# Repeat for each batch
```

3. **Parser error on file:**
```bash
# Check logs for file-specific errors
lf services logs --service rag | grep -i error

# Remove problematic file
lf datasets delete-file <dataset> <filename>
lf datasets process <dataset>
```

---

### Parser Failures

**Symptoms:**
- "Failed to parse" errors
- Specific file types fail
- Empty chunks created

**Diagnostics:**

```bash
# Check logs for parser errors
lf services logs --service rag | grep -i parser

# Verify file type
file /path/to/document.pdf
```

**Solutions:**

1. **PDF is scanned image:**
```yaml
# Enable OCR
parsers:
  - type: PDFParser_LlamaIndex
    config:
      use_ocr: true
```

2. **Encrypted/protected PDF:**
```bash
# Remove protection first
qpdf --decrypt protected.pdf unlocked.pdf
```

3. **Unsupported file type:**
```yaml
# Add appropriate parser
parsers:
  - type: TextParser_Python
    file_include_patterns: ["*.xyz"]
```

4. **Corrupted file:**
```bash
# Verify file integrity
file document.pdf
# "PDF document" = OK
# "data" = corrupted

# Remove corrupted file
lf datasets delete-file <dataset> corrupted.pdf
```

---

### Empty Chunks

**Symptoms:**
- Files processed but 0 chunks
- Database shows no documents
- RAG queries return nothing

**Diagnostics:**

```bash
# Check chunk count
lf rag health

# Check dataset files
lf datasets list-files <dataset>
```

**Solutions:**

1. **File pattern mismatch:**
```yaml
# Check file_include_patterns match your files
parsers:
  - type: PDFParser_LlamaIndex
    file_include_patterns:
      - "*.pdf"
      - "*.PDF"  # Case-sensitive!
```

2. **Chunk size too large:**
```yaml
# Reduce chunk size
parsers:
  - type: PDFParser_LlamaIndex
    config:
      chunk_size: 500  # Smaller chunks
```

3. **No text content:**
```bash
# Verify file has extractable text
pdftotext document.pdf - | head
```

---

### Slow Processing

**Symptoms:**
- Processing takes hours
- Memory usage very high
- System becomes unresponsive

**Diagnostics:**

```bash
# Check system resources
top -l 1 | head -20
free -h

# Check file sizes
du -sh ./data/datasets/<dataset>/
```

**Solutions:**

1. **Large files:**
```yaml
# Reduce batch size
embedding_strategies:
  - name: default_embeddings
    config:
      batch_size: 8  # Smaller batches
```

2. **Too many extractors:**
```yaml
# Use fewer extractors
extractors:
  - type: ContentStatisticsExtractor  # Keep minimal
  # Remove expensive ones like EntityExtractor
```

3. **Insufficient memory:**
```bash
# Process in smaller batches
lf datasets upload <dataset> ./batch1/*
lf datasets process <dataset>
# Wait for completion, then:
lf datasets upload <dataset> ./batch2/*
lf datasets process <dataset>
```

---

### Duplicate Chunks

**Symptoms:**
- Same content appears multiple times
- RAG returns repetitive results
- Database larger than expected

**Diagnostics:**

```bash
# Check chunk count vs file count
lf rag health
lf datasets list-files <dataset>
```

**Solutions:**

1. **Re-processed without clearing:**
```bash
# Clear and reprocess
lf datasets clear <dataset>
lf datasets process <dataset>
```

2. **Overlapping file patterns:**
```yaml
# Make patterns exclusive
parsers:
  - type: PDFParser_LlamaIndex
    file_include_patterns: ["*.pdf"]
    file_exclude_patterns: ["*copy*.pdf", "*backup*"]
```

---

## File-Specific Issues

### PDF Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| No text | Scanned image | Enable OCR |
| Garbled text | Encoding | Try different parser |
| Missing pages | Protected | Decrypt first |
| Tables broken | Layout | Enable table extraction |

### Markdown Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| Code blocks merged | Wrong chunking | Use `code` strategy |
| Headings lost | No extractor | Add HeadingExtractor |
| Links broken | No extraction | Add LinkExtractor |

### CSV Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| Wrong columns | Encoding | Specify encoding |
| Empty rows | Header mismatch | Check delimiter |
| Too many chunks | Row-per-chunk | Use larger chunk size |

---

## Recovery Procedures

### Reprocess Single File

```bash
# Delete and re-upload
lf datasets delete-file <dataset> problem.pdf
lf datasets upload <dataset> ./fixed/problem.pdf
lf datasets process <dataset>
```

### Reprocess All Files

```bash
# Clear all chunks
lf datasets clear <dataset>

# Reprocess
lf datasets process <dataset>
```

### Start Fresh

```bash
# Delete dataset completely
lf datasets delete <dataset> --include-vectors

# Recreate
lf datasets create -s <strategy> -b <database> <dataset>
lf datasets upload <dataset> ./files/*
lf datasets process <dataset>
```

---

## Monitoring Progress

### Real-Time Status

```bash
# Watch progress
watch -n 5 'lf datasets status <dataset>'
```

### Log Streaming

```bash
# Stream RAG logs
lf services logs --service rag --follow
```

### Progress Indicators

Look for these log messages:
```
[INFO] Processing file: document.pdf
[INFO] Created 45 chunks
[INFO] Embedding batch 1/3...
[INFO] Indexed to main_db
```
