# Chunking Strategies

## Strategy Types

### Semantic Chunking
**Best for:** Complex documents, multi-topic content
**How it works:** Uses ML to detect semantic boundaries

```yaml
config:
  chunk_strategy: semantic
  chunk_size: 1000
  chunk_overlap: 150
```

### Paragraph Chunking
**Best for:** Well-structured documents, articles
**How it works:** Splits on paragraph boundaries

```yaml
config:
  chunk_strategy: paragraphs
  chunk_size: 800
  chunk_overlap: 100
```

### Heading-based Chunking
**Best for:** Documentation, markdown files
**How it works:** Preserves section structure

```yaml
config:
  chunk_strategy: headings
  chunk_size: 600
```

### Sentence Chunking
**Best for:** Dense technical content
**How it works:** Splits on sentence boundaries

```yaml
config:
  chunk_strategy: sentences
  chunk_size: 500
  chunk_overlap: 50
```

### Code Chunking
**Best for:** Source code files
**How it works:** Preserves function/class boundaries

```yaml
config:
  chunk_strategy: code
  chunk_size: 600
  preserve_code_structure: true
```

### Row-based Chunking (Data)
**Best for:** CSV, tabular data
**How it works:** Groups rows together

```yaml
config:
  chunk_strategy: rows
  chunk_size: 1000
```

## Chunk Size Guidelines

| Document Type | Recommended Size | Overlap | Reasoning |
|--------------|-----------------|---------|-----------|
| Legal/Regulatory | 1000-1200 | 150 | Preserve legal context |
| Technical Docs | 600-800 | 80-100 | Balance precision/context |
| Code | 500-600 | 50 | Keep functions together |
| Notes/Markdown | 400-500 | 40-50 | Smaller sections |
| Large PDFs | 900-1100 | 140-160 | Handle dense content |

## Trade-offs

### Larger Chunks
- **Pros:** More context, better understanding
- **Cons:** Less precise retrieval, slower processing

### Smaller Chunks
- **Pros:** More precise retrieval, faster
- **Cons:** May lose context, need higher top_k

### Overlap
- **Purpose:** Maintain context across boundaries
- **Typical:** 10-15% of chunk_size
- **Higher overlap:** Better context, more redundancy
- **Lower overlap:** Less redundancy, potential context loss

## Configuration Examples

### PDF with Tables
```yaml
config:
  chunk_size: 1200
  chunk_overlap: 150
  chunk_strategy: semantic
  extract_tables: true       # Keep tables as separate chunks
```

### Code Repository
```yaml
config:
  chunk_size: 600
  chunk_overlap: 50
  chunk_strategy: code
  preserve_code_structure: true
  detect_language: true
```

### Dense Technical Manual
```yaml
config:
  chunk_size: 1100
  chunk_overlap: 160
  chunk_strategy: paragraphs
```
