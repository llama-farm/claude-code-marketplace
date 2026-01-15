---
description: Example projects and patterns for LlamaFarm. Reference for scaffolding and learning from working examples.
---

# LlamaFarm Examples Skill

Reference for LlamaFarm example projects and their patterns.

## When to Load

Load this skill when:
- User asks about example projects
- User wants to scaffold from examples
- User needs working configuration references
- Learning LlamaFarm patterns

## Example Catalog

| Example | Use Case | Complexity |
|---------|----------|------------|
| `quick_rag` | Getting started, minimal setup | Beginner |
| `fda_rag` | Regulatory PDF analysis | Intermediate |
| `gov_rag` | Large government documents | Intermediate |
| `ocr_and_document` | Scanned document OCR | Advanced |
| `rag_pipeline` | Multi-database, hybrid search | Advanced |

## Progressive Disclosure

For specific example patterns, load:
- `quick-rag.md` - Minimal RAG setup
- `fda-rag.md` - FDA regulatory analysis
- `gov-rag.md` - Government document processing

## Quick Reference

### Scaffolding Commands

```bash
# List examples
lf examples list

# Import example
lf examples import quick_rag --name my-project

# Show example config
cat examples/quick_rag/llamafarm.yaml
```

### Example Locations

Examples are located in the LlamaFarm repository:
```
llamafarm/examples/
├── quick_rag/
│   ├── llamafarm.yaml
│   ├── manifest.yaml
│   ├── README.md
│   └── files/
├── fda_rag/
├── gov_rag/
├── ocr_and_document/
└── rag_pipeline/
```

### Common Patterns Across Examples

**All examples include:**
- Complete `llamafarm.yaml` configuration
- Sample documents in `files/` directory
- `manifest.yaml` for import metadata
- `README.md` with usage instructions
- `run_example.sh` for interactive walkthrough

**Standard workflow:**
1. Initialize/import example
2. Start services
3. Create dataset
4. Upload and process files
5. Query with chat

## Learning Path

### Beginner

1. **Start with `quick_rag`**
   - Minimal configuration
   - 2 markdown files
   - Basic similarity search
   - Ollama integration

2. **Learn the workflow:**
   ```bash
   lf start
   lf datasets create -s markdown_processor -b main_db research
   lf datasets upload research ./files/*
   lf datasets process research
   lf chat "What is the document about?"
   ```

### Intermediate

3. **Try `fda_rag` for PDFs**
   - PDF parsing with fallbacks
   - Entity extraction
   - Semantic chunking
   - Larger documents

4. **Explore `gov_rag` for scale**
   - Large document handling
   - Hierarchy extraction
   - Geospatial entities

### Advanced

5. **Study `rag_pipeline` for optimization**
   - Multiple databases
   - Hybrid search
   - Reranking
   - Strategy composition

6. **Explore `ocr_and_document` for vision**
   - OCR pipelines
   - Image processing
   - Document extraction

## Key Configuration Patterns

### Minimal Valid Config (from quick_rag)

```yaml
version: v1
name: my-project
namespace: default

runtime:
  models:
    - name: default
      provider: ollama
      model: llama3.1:8b
      default: true

prompts:
  - name: default
    messages:
      - role: system
        content: You are a helpful assistant.

rag:
  default_database: main_db
  databases:
    - name: main_db
      type: ChromaStore
      embedding_strategies:
        - name: default_embeddings
          type: UniversalEmbedder
      retrieval_strategies:
        - name: basic_search
          type: BasicSimilarityStrategy
          default: true

  data_processing_strategies:
    - name: default
      parsers:
        - type: MarkdownParser_LlamaIndex
          file_include_patterns: ["*.md"]

datasets:
  - name: docs
    database: main_db
    data_processing_strategy: default
```

### PDF Analysis (from fda_rag)

Key additions:
- `PDFParser_LlamaIndex` with semantic chunking
- `EntityExtractor` for ORG, DATE, PERSON
- Larger chunk sizes (1200/150)
- Citation-aware prompts

### Large Documents (from gov_rag)

Key additions:
- `HeadingExtractor` with hierarchy
- `TableExtractor` for structured data
- Multiple parser fallbacks
- Increased chunk overlap

### Advanced RAG (from rag_pipeline)

Key additions:
- Multiple databases with routing
- `HybridUniversalStrategy`
- `CrossEncoderRerankedStrategy`
- Metadata filtering
