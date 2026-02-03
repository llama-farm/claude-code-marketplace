---
description: Generate LlamaFarm configurations from natural language. Activates for new projects, config creation, PDF/markdown/code patterns.
---

# LlamaFarm Config Generation Skill

Generate LlamaFarm configurations from natural language descriptions using internalized patterns.

## Command Entry Point

```
/llamafarm:config "I want to analyze FDA regulatory PDFs"
/llamafarm:config "Add OCR support to my existing project"
/llamafarm:config "Create a multi-model setup with Ollama and OpenAI"
```

## When to Load

Activate this skill when:
- User runs `/llamafarm:config` command
- User asks to create a new LlamaFarm project
- User wants to modify existing configuration
- User describes what they want to build

## Generation Workflow

1. **Parse intent** - New project or modification?
2. **Identify use case** - PDF, markdown, mixed, code?
3. **Select pattern** - Load appropriate pattern file
4. **Generate config** - Apply pattern to user needs
5. **Validate** - Run `lf projects validate --config llamafarm.yaml`
6. **Present** - Show config with CLI next steps

## Intent Detection

**New project indicators:**
- "I want to...", "Create...", "Build...", "New project..."
- "Set up...", "Make a...", "Start a..."

**Modification indicators:**
- "Add...", "Change...", "Update...", "Remove..."
- "Switch...", "Modify...", "Enable..."

If modifying, read the existing `llamafarm.yaml` first.

## Use Case Detection

| Keywords | Use Case Pattern |
|----------|-----------------|
| PDF, documents, regulatory, FDA, legal | `pdf-pattern.md` |
| markdown, notes, docs, README, wiki | `markdown-pattern.md` |
| mixed, various, multiple formats | `mixed-pattern.md` |
| code, source, technical, codebase | `code-pattern.md` |
| large, complex, ordinance, manual | `large-docs-pattern.md` |

## Progressive Disclosure

Load these files based on the use case:

- **pdf-pattern.md** - PDF document configurations
- **markdown-pattern.md** - Markdown/text notes configurations
- **mixed-pattern.md** - Mixed format configurations
- **code-pattern.md** - Code analysis configurations
- **large-docs-pattern.md** - Large complex document configurations
- **models.md** - Model configuration templates

## Minimal Valid Config

Every generated config must include:

```yaml
version: v1
name: <project-name>
namespace: default

runtime:
  models:
    - name: default
      provider: <ollama|universal|openai>
      model: <model-id>
      default: true

prompts:
  - name: default
    messages:
      - role: system
        content: |
          <appropriate-system-prompt>

rag:
  databases:
    - name: main_db
      type: ChromaStore
      config:
        collection_name: documents
        persist_directory: ./data/main_db
      embedding_strategies:
        - name: default_embeddings
          type: UniversalEmbedder
          config:
            model: nomic-ai/nomic-embed-text-v2-moe
            dimension: 768
      retrieval_strategies:
        - name: basic_search
          type: BasicSimilarityStrategy
          config:
            top_k: 10
          default: true
      default_embedding_strategy: default_embeddings
      default_retrieval_strategy: basic_search

  data_processing_strategies:
    - name: default_processor
      parsers:
        - type: <appropriate-parser>
          file_include_patterns: ["<patterns>"]
          priority: 100
      extractors:
        - type: ContentStatisticsExtractor

datasets:
  - name: default_dataset
    database: main_db
    data_processing_strategy: default_processor
```

## Presenting Results

After generating a config, show it with annotations and next steps:

```
Configuration Generated: <project-name>
=======================================

File: llamafarm.yaml

Key choices:
  - Provider: <provider> (<reason>)
  - Parser: <parser> (<reason>)
  - Extractors: <list> (<reason>)

Next steps:
  1. Review the generated llamafarm.yaml
  2. lf start
  3. lf datasets create -s default_processor -b main_db my_dataset
  4. lf datasets upload my_dataset ./path/to/files/*
  5. lf datasets process my_dataset
  6. lf chat "Ask a question"
```

## Post-Generation CLI Workflow

```bash
# Validate
lf projects validate --config llamafarm.yaml

# Start services
lf start

# Create dataset
lf datasets create -s <strategy> -b <database> <dataset-name>

# Upload files
lf datasets upload <dataset-name> ./path/to/files/*

# Process
lf datasets process <dataset-name>

# Chat
lf chat "Your question here"
```
