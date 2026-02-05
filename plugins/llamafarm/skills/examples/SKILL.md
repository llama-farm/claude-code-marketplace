---
description: Example projects and patterns for LlamaFarm. Reference for scaffolding and learning from working examples.
---

# LlamaFarm Examples Skill

Reference for LlamaFarm example projects and their patterns.

## Command Entry Point

```
/llamafarm:example                       # List all available examples
/llamafarm:example quick_rag             # Scaffold from quick_rag example
/llamafarm:example fda_rag --name myproj # Scaffold with custom name
/llamafarm:example gov_rag --show        # Show example config without scaffolding
```

## When to Load

Load this skill when:
- User runs `/llamafarm:example` command
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

## Implementation

### Step 1: List Available Examples (no arguments)

```
Available LlamaFarm Examples
============================

quick_rag
  Minimal RAG setup for getting started quickly.
  Documents: 2 markdown files about AI/ML
  Features: Basic similarity search, Universal Runtime integration
  Best for: Learning LlamaFarm basics

fda_rag
  FDA regulatory document analysis.
  Documents: FDA correspondence letters (PDFs)
  Features: Entity extraction, semantic chunking, citation tracking
  Best for: Regulatory/legal document analysis

gov_rag
  Government and municipal document processing.
  Documents: City ordinances, planning documents
  Features: Large document handling, hierarchy extraction
  Best for: Large complex PDFs, municipal docs

ocr_and_document
  Document OCR and image text extraction.
  Documents: Scanned PDFs, images with text
  Features: OCR pipeline, form parsing, document extraction
  Best for: Scanned documents, image-based PDFs

rag_pipeline
  Advanced RAG configuration showcase.
  Documents: Mixed format technical docs
  Features: Multiple databases, reranking, hybrid search
  Best for: Learning advanced RAG patterns

Usage:
  /llamafarm:example <name>           # Scaffold from example
  /llamafarm:example <name> --show    # Preview config only
```

### Step 2: Show Example (--show flag)

When user runs `/llamafarm:example quick_rag --show`:

```bash
# Read the example's llamafarm.yaml
cat ~/workspace/pivot/llamafarm/examples/quick_rag/llamafarm.yaml
```

Present the configuration with annotations explaining key choices.

### Step 3: Scaffold Project

When user runs `/llamafarm:example quick_rag`:

```bash
# Import example using CLI
lf examples import quick_rag --name quick_rag

# Or manually copy files
mkdir -p ./quick_rag
cp -r ~/workspace/pivot/llamafarm/examples/quick_rag/* ./quick_rag/
cd ./quick_rag
```

Report results:

```
Project Scaffolded: quick_rag
=============================

Created in: ./quick_rag/

Files:
  llamafarm.yaml    - Project configuration
  files/            - Sample documents
  README.md         - Example documentation

Next Steps:

1. Navigate to project:
   cd quick_rag

2. Start services:
   lf start

3. Create and process dataset:
   lf datasets create -s markdown_processor -b main_db research
   lf datasets upload research ./files/*
   lf datasets process research

4. Chat with your documents:
   lf chat "What are neural scaling laws?"

Requirements:
  - Universal Runtime running on port 11540
```

### Step 4: Custom Project Name

When user runs `/llamafarm:example fda_rag --name my_legal_project`:

1. Copy example files to `./my_legal_project/`
2. Update `name` field in `llamafarm.yaml`
3. Report new project location

## Progressive Disclosure

For specific example patterns, load:
- `quick-rag.md` - Minimal RAG setup
- `fda-rag.md` - FDA regulatory analysis
- `gov-rag.md` - Government document processing

## Scaffolding Commands

```bash
# List examples
lf examples list

# Import example
lf examples import quick_rag --name my-project

# Show example config
cat examples/quick_rag/llamafarm.yaml
```

## Example Locations

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
   - Universal Runtime integration

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

## Example Details

### quick_rag

**Purpose:** Minimal viable RAG for learning

**Configuration highlights:**
- Single Universal Runtime model (unsloth/Qwen3-4B-GGUF:Q4_K_M)
- ChromaStore vector database
- Basic similarity retrieval
- Markdown parsing with heading extraction

**Sample files:**
- `neural_scaling.md` - AI research notes
- `engineering_practices.md` - Software engineering tips

**Requirements:**
- Universal Runtime installed and running
- Model auto-downloaded on first use

### fda_rag

**Purpose:** Regulatory document analysis

**Configuration highlights:**
- PDF parsing with entity extraction
- Semantic chunking (1200 chars / 150 overlap)
- Organizations, dates, products extraction
- Citation-aware system prompt

**Sample files:**
- Multiple FDA warning letters (PDF)
- Regulatory correspondence

**Requirements:**
- Universal Runtime or OpenAI for chat
- PDF dependencies (PyPDF2, LlamaIndex)

### gov_rag

**Purpose:** Municipal and government documents

**Configuration highlights:**
- Large document handling
- Hierarchy extraction (outline/headings)
- Geospatial entity extraction (GPE, FAC, LOC)
- Table extraction

**Sample files:**
- City ordinances
- Planning documents
- Municipal codes

**Requirements:**
- Sufficient disk space for large vectors
- Consider Universal Runtime for faster embeddings

### ocr_and_document

**Purpose:** Scanned document processing

**Configuration highlights:**
- OCR pipeline integration
- Image text extraction
- Form field parsing
- Multi-model with vision capability

**Features:**
- Surya OCR
- PaddleOCR
- Document extraction models

**Requirements:**
- Universal Runtime for OCR models
- Additional ML dependencies

### rag_pipeline

**Purpose:** Advanced RAG showcase

**Configuration highlights:**
- Multiple vector databases
- Hybrid search (semantic + keyword)
- Cross-encoder reranking
- Metadata filtering

**Demonstrates:**
- Database routing
- Strategy composition
- Performance optimization

**Requirements:**
- Multiple model endpoints
- Larger resource allocation

## Key Configuration Patterns

### Minimal Valid Config (from quick_rag)

```yaml
version: v1
name: my-project
namespace: default

runtime:
  models:
    - name: default
      provider: universal
      model: unsloth/Qwen3-4B-GGUF:Q4_K_M
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
