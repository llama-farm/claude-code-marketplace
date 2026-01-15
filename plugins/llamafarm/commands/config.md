---
description: Generate or modify LlamaFarm configuration from natural language description
---

# /config - LlamaFarm Configuration Generator

Generate or modify LlamaFarm configurations from natural language descriptions.

## Usage

```
/config <description of project or changes>
```

### Examples

```
/config I want to chat with FDA correspondence PDFs using llama3.1
/config Add a second database for archived documents with hybrid search
/config Change the chunking strategy to semantic with 1000 char chunks
/config I have markdown docs and want keyword extraction
/config Switch from Ollama to Universal Runtime
/config Add entity extraction for organizations and dates
```

## What This Command Does

1. **Parses user intent** - Understands what kind of project or change is needed
2. **Maps to example patterns** - Uses internalized LlamaFarm examples
3. **Generates valid config** - Creates schema-compliant YAML
4. **Validates output** - Runs `/validate` internally
5. **Provides CLI guidance** - Shows any required CLI commands

## Implementation

When the user runs `/config <description>`, follow these steps:

### Step 1: Analyze Intent

Determine if this is:
- **New project** - Generate complete config from scratch
- **Modification** - Change existing `llamafarm.yaml`

Keywords that indicate new project:
- "I want to...", "Create a...", "Build a...", "New project..."

Keywords that indicate modification:
- "Add...", "Change...", "Switch...", "Remove...", "Update..."

### Step 2: Load the config-generation Skill

Reference the skill for:
- Use case patterns (PDF, Markdown, Mixed, Code)
- File type → parser mapping
- Extractor recommendations
- CLI operations

### Step 3: Map Description to Pattern

| User Says | Pattern to Use |
|-----------|----------------|
| PDF, documents, regulatory, FDA, legal | PDF Documents pattern |
| markdown, notes, docs, README | Markdown/Text Notes pattern |
| mixed, various, multiple formats | Mixed Formats pattern |
| code, source, technical, logs | Code/Technical pattern |
| large, complex, ordinance | Large Complex PDFs pattern |

### Step 4: Generate Config

For **new projects**, generate complete config:

```yaml
version: v1
name: <derived-from-description>
namespace: default

runtime:
  models:
    - name: default
      description: "<description-based>"
      provider: <ollama|universal|openai>
      model: <appropriate-model>
      base_url: <appropriate-url>
      default: true
      prompt_format: unstructured

prompts:
  - name: default
    messages:
      - role: system
        content: |
          <context-appropriate-system-prompt>

rag:
  default_database: main_db
  databases:
    - name: main_db
      type: ChromaStore
      config:
        collection_name: documents
        distance_function: cosine
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
    - name: <strategy-name>
      description: "<description>"
      parsers:
        <appropriate-parsers-based-on-file-types>
      extractors:
        <appropriate-extractors-based-on-use-case>

datasets:
  - name: <dataset-name>
    database: main_db
    data_processing_strategy: <strategy-name>
```

For **modifications**, read existing config, make targeted changes, preserve structure.

### Step 5: Validate Generated Config

Always run validation before presenting:

```bash
# Save to temp file and validate
lf projects validate --config /tmp/generated-config.yaml
```

If validation fails, fix the issues and regenerate.

### Step 6: Present Results

```
Generated LlamaFarm Configuration
=================================

[YAML config here]

---
Next Steps:

1. Save this as `llamafarm.yaml` in your project directory
2. Run: lf start
3. Create your dataset: lf datasets create -s <strategy> -b main_db <dataset-name>
4. Upload files: lf datasets upload <dataset-name> ./path/to/files/*
5. Process: lf datasets process <dataset-name>
6. Chat: lf chat "Your question here"
```

## Use Case Patterns

### PDF Documents (FDA/Regulatory/Legal)

```yaml
runtime:
  models:
    - name: default
      provider: ollama
      model: llama3.1:8b
      model_api_parameters:
        temperature: 0.2

prompts:
  - name: default
    messages:
      - role: system
        content: |
          You are an expert assistant. Always cite specific document sections.
          If information cannot be found in the context, state that clearly.

rag:
  data_processing_strategies:
    - name: pdf_processor
      parsers:
        - type: PDFParser_LlamaIndex
          file_include_patterns: ["*.pdf", "*.PDF"]
          priority: 100
          config:
            chunk_size: 1200
            chunk_overlap: 150
            chunk_strategy: semantic
            extract_metadata: true
            extract_tables: true
        - type: PDFParser_PyPDF2
          file_include_patterns: ["*.pdf", "*.PDF"]
          priority: 50
          config:
            chunk_size: 1000
            chunk_overlap: 120
            chunk_strategy: paragraphs
      extractors:
        - type: EntityExtractor
          config:
            entity_types: [ORG, DATE, PERSON, PRODUCT]
            use_fallback: true
        - type: ContentStatisticsExtractor
          config:
            include_readability: true
```

### Markdown/Text Notes

```yaml
rag:
  data_processing_strategies:
    - name: markdown_processor
      parsers:
        - type: MarkdownParser_LlamaIndex
          file_include_patterns: ["*.md", "*.markdown", "README*"]
          priority: 100
          config:
            chunk_size: 400
            chunk_overlap: 40
            chunk_strategy: headings
        - type: TextParser_Python
          file_include_patterns: ["*.txt"]
          priority: 90
          config:
            chunk_size: 400
            chunk_overlap: 40
      extractors:
        - type: HeadingExtractor
          config:
            max_level: 6
            include_hierarchy: true
        - type: LinkExtractor
          config:
            extract_urls: true
        - type: ContentStatisticsExtractor
          config:
            include_readability: true
```

### Mixed Formats

```yaml
rag:
  databases:
    - name: mixed_db
      retrieval_strategies:
        - name: hybrid_search
          type: HybridUniversalStrategy
          config:
            strategies:
              - type: BasicSimilarityStrategy
                config:
                  top_k: 8
                weight: 0.6
            combination_method: weighted_average
            final_k: 10
          default: true

  data_processing_strategies:
    - name: universal_processor
      parsers:
        - type: PDFParser_LlamaIndex
          file_include_patterns: ["*.pdf"]
          priority: 100
        - type: MarkdownParser_LlamaIndex
          file_include_patterns: ["*.md"]
          priority: 100
        - type: DocxParser_LlamaIndex
          file_include_patterns: ["*.docx"]
          priority: 100
        - type: CSVParser_Pandas
          file_include_patterns: ["*.csv"]
          priority: 100
        - type: TextParser_Python
          file_include_patterns: ["*.txt", "*.py", "*.html"]
          priority: 50
```

### Code/Technical

```yaml
rag:
  data_processing_strategies:
    - name: code_processor
      parsers:
        - type: TextParser_LlamaIndex
          file_include_patterns: ["*.py", "*.js", "*.ts", "*.go", "*.java"]
          priority: 100
          config:
            chunk_size: 600
            chunk_strategy: code
            preserve_code_structure: true
            detect_language: true
      extractors:
        - type: PatternExtractor
          config:
            predefined_patterns: [email, url, version]
            custom_patterns:
              - name: function_def
                pattern: "def\\s+\\w+\\s*\\("
                description: "Python function definitions"
              - name: class_def
                pattern: "class\\s+\\w+[\\(\\:]"
                description: "Class definitions"
        - type: LinkExtractor
          config:
            extract_urls: true
```

### Large Complex PDFs

```yaml
rag:
  data_processing_strategies:
    - name: large_pdf_processor
      parsers:
        - type: PDFParser_LlamaIndex
          file_include_patterns: ["*.pdf"]
          priority: 100
          config:
            chunk_size: 1100
            chunk_overlap: 160
            chunk_strategy: semantic
            extract_tables: true
        - type: PDFParser_PyPDF2
          file_include_patterns: ["*.pdf"]
          priority: 50
          config:
            chunk_size: 900
            chunk_overlap: 140
      extractors:
        - type: HeadingExtractor
          config:
            max_level: 4
            include_hierarchy: true
            extract_outline: true
        - type: EntityExtractor
          config:
            entity_types: [ORG, DATE, FAC, GPE, LOC]
            use_fallback: true
        - type: ContentStatisticsExtractor
          config:
            include_structure: true
```

## CLI Operations

When config changes require CLI operations, provide the commands:

### Initialize New Project
```bash
lf init my-project
cd my-project
# Then save the generated llamafarm.yaml here
```

### Create Dataset
```bash
lf datasets create -s <strategy-name> -b <database-name> <dataset-name>
```

### Upload and Process Files
```bash
lf datasets upload <dataset-name> ./path/to/files/*
lf datasets process <dataset-name>
```

### Verify Setup
```bash
lf services status
lf datasets list
lf rag query --database <db> --top-k 5 "test query"
```

## Skills to Load

- **config-generation**: For detailed patterns and file type mappings
- **config-validation**: For schema reference and validation
- **rag-pipeline**: For advanced RAG configuration guidance
