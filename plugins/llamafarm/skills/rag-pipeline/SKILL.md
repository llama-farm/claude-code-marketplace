---
description: Deep guidance for RAG pipelines. Activates for chunking, embeddings, retrieval, vector stores, RAG optimization.
---

# LlamaFarm RAG Pipeline Skill

Deep guidance for configuring RAG (Retrieval-Augmented Generation) pipelines.

## When to Load

Activate this skill when:
- User asks about RAG configuration
- User needs help with chunking, embeddings, or retrieval
- User is troubleshooting RAG quality issues
- User wants to optimize RAG performance

## RAG Pipeline Overview

```
Files → Parser → Chunks → Embedder → Vector Store → Retriever → LLM
         ↓
      Extractors → Metadata
```

## Quick Reference

### Chunk Size Guidelines

| Document Type | Chunk Size | Overlap |
|--------------|------------|---------|
| Legal/Regulatory | 1000-1200 | 150 |
| Technical Docs | 600-800 | 80-100 |
| Code | 500-600 | 50 |
| Notes/Markdown | 400-500 | 40-50 |
| Large PDFs | 900-1100 | 140-160 |

### Top-K Guidelines

| Use Case | Top-K |
|----------|-------|
| Quick Q&A | 3-5 |
| Research | 8-10 |
| Legal/Compliance | 10-15 |
| Code Search | 5-8 |

## Progressive Disclosure

Load these files for detailed guidance:

- **chunking.md** - Chunking strategies and configuration
- **embeddings.md** - Embedding model selection and setup
- **retrieval.md** - Retrieval strategy comparison
- **troubleshooting.md** - Diagnosing and fixing RAG issues
