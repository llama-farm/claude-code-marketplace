---
description: OCR, Named Entity Recognition, and reranking. Extract text from images, identify entities in text, and rerank search results for relevance.
---

# ML NLP Skill

Guide for OCR, NER, and reranking capabilities in LlamaFarm.

## When to Load

Load this skill when the user:
- Wants to extract text from images or PDFs (OCR)
- Asks about named entity recognition (NER)
- Needs to rerank search results
- Wants to identify people, organizations, or locations in text
- Asks about document processing pipelines
- Needs to improve RAG retrieval quality with reranking

## API Overview

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/ocr` | POST | Extract text from images/documents |
| `/v1/ner` | POST | Extract named entities from text |
| `/v1/rerank` | POST | Rerank documents by relevance to query |

## OCR Backend Selection

| Backend | Speed | Accuracy | Languages | Best For |
|---------|-------|----------|-----------|----------|
| `surya` | Medium | Very High | 90+ | General purpose, best quality |
| `easyocr` | Medium | High | 80+ | Good balance, wide language support |
| `paddleocr` | Fast | High | 80+ | CJK languages, structured docs |
| `tesseract` | Fast | Moderate | 100+ | Simple documents, legacy systems |

## NER Models

| Model | Speed | Entities | Best For |
|-------|-------|----------|----------|
| `dslim/bert-base-NER` | Fast | PER, ORG, LOC, MISC | General English NER |
| `Jean-Baptiste/camembert-ner` | Fast | PER, ORG, LOC, MISC | French NER |
| `flair/ner-english-large` | Medium | PER, ORG, LOC, MISC | Higher accuracy English |

## Reranking Overview

Reranking improves search quality by re-scoring retrieved results with a cross-encoder model. Typical pattern:

```
RAG retrieve top-50 → Rerank → Return top-10
```

This significantly improves precision over embedding-only retrieval.

## Quick Start

### OCR

```bash
curl -X POST http://localhost:14345/v1/ocr \
  -F "file=@document.png" \
  -F "backend=surya"
```

### NER

```bash
curl -X POST http://localhost:14345/v1/ner \
  -H "Content-Type: application/json" \
  -d '{
    "text": "John Smith works at Google in Mountain View.",
    "model": "dslim/bert-base-NER"
  }'
```

### Rerank

```bash
curl -X POST http://localhost:14345/v1/rerank \
  -H "Content-Type: application/json" \
  -d '{
    "query": "How to configure authentication?",
    "documents": [
      "Authentication is configured via the auth section...",
      "The logging system supports multiple outputs...",
      "Set up OAuth by adding provider credentials..."
    ],
    "top_k": 2
  }'
```

## Progressive Disclosure

For detailed guidance:
- **ocr.md** - Backend comparison, file types, preprocessing, batch OCR
- **ner.md** - Entity types, model selection, confidence thresholds
- **reranking.md** - RAG integration, model selection, performance tuning
