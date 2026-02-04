---
description: Browse and scaffold from LlamaFarm example projects
disable-model-invocation: true
allowed-tools:
  - Bash
  - Read
argument-hint: "[example-name] [--name <project-name>] [--show]"
---

# /llamafarm:example - Example Project Scaffolding

Browse available LlamaFarm examples and scaffold new projects from them.

## Usage

```
/llamafarm:example                       # List all available examples
/llamafarm:example quick_rag             # Scaffold from quick_rag example
/llamafarm:example fda_rag --name myproj # Scaffold with custom name
/llamafarm:example gov_rag --show        # Show example config without scaffolding
```

## What This Command Does

1. **Lists examples** - Shows all available example projects with descriptions
2. **Previews configs** - Displays example configurations before scaffolding
3. **Scaffolds projects** - Creates new project from example template
4. **Provides guidance** - Next steps for running the example

## Implementation

### Step 1: List Available Examples (no arguments)

```
Available LlamaFarm Examples
============================

quick_rag
  Minimal RAG setup for getting started quickly.
  Documents: 2 markdown files about AI/ML
  Features: Basic similarity search, Ollama integration
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

Present the configuration with annotations:

```
quick_rag Example Configuration
===============================

# This example demonstrates a minimal RAG setup

version: v1
name: quick-rag-example
namespace: default

runtime:
  models:
    - name: default
      provider: ollama
      model: llama3.1:8b        # Requires: ollama pull llama3.1:8b
      default: true

prompts:
  - name: default
    messages:
      - role: system
        content: |
          You are a helpful assistant. Use the provided context to answer questions.
          Always cite your sources.

rag:
  default_database: main_db
  databases:
    - name: main_db
      type: ChromaStore
      # ... full config ...

datasets:
  - name: research
    database: main_db
    data_processing_strategy: markdown_processor

---
Files included:
  - files/neural_scaling.md
  - files/engineering_practices.md

To scaffold: /llamafarm:example quick_rag
```

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
  - Ollama with llama3.1:8b: ollama pull llama3.1:8b
```

### Step 4: Custom Project Name

When user runs `/llamafarm:example fda_rag --name my_legal_project`:

1. Copy example files to `./my_legal_project/`
2. Update `name` field in `llamafarm.yaml`
3. Report new project location

## Example Details

### quick_rag

**Purpose:** Minimal viable RAG for learning

**Configuration highlights:**
- Single Ollama model (llama3.1:8b)
- ChromaStore vector database
- Basic similarity retrieval
- Markdown parsing with heading extraction

**Sample files:**
- `neural_scaling.md` - AI research notes
- `engineering_practices.md` - Software engineering tips

**Requirements:**
- Ollama installed and running
- llama3.1:8b model pulled

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
- Ollama or OpenAI for chat
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

## Skills to Load

- `examples` skill for detailed pattern documentation

## Related Commands

- `/llamafarm:config` - Generate custom configuration
- `/llamafarm:validate` - Validate configuration
- `/llamafarm:start` - Start services
