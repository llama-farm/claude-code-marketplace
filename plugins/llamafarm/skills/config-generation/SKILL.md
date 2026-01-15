---
description: Generate LlamaFarm configurations from natural language. Activates for new projects, config creation, PDF/markdown/code patterns.
---

# LlamaFarm Config Generation Skill

Generate LlamaFarm configurations from natural language descriptions using internalized patterns.

## When to Load

Activate this skill when:
- User runs `/config` command
- User asks to create a new LlamaFarm project
- User wants to modify existing configuration
- User describes what they want to build

## Generation Workflow

1. **Parse intent** - New project or modification?
2. **Identify use case** - PDF, markdown, mixed, code?
3. **Select pattern** - Load appropriate pattern file
4. **Generate config** - Apply pattern to user needs
5. **Validate** - Run `/validate` on output
6. **Present** - Show config with CLI next steps

## Intent Detection

**New project indicators:**
- "I want to...", "Create...", "Build...", "New project..."
- "Set up...", "Make a...", "Start a..."

**Modification indicators:**
- "Add...", "Change...", "Update...", "Remove..."
- "Switch...", "Modify...", "Enable..."

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
- **cli-reference.md** - CLI operations guide

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
