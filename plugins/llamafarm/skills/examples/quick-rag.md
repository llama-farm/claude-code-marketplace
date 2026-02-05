# quick_rag Example

Minimal RAG setup for getting started with LlamaFarm quickly.

## Overview

**Purpose:** Learn LlamaFarm basics with the simplest possible configuration

**Documents:** 2 markdown files about AI/ML topics

**Features:**
- Basic similarity search
- Universal Runtime integration
- Markdown parsing
- Heading extraction

**Best for:** First-time LlamaFarm users

## Complete Configuration

```yaml
version: v1
name: quick-rag-example
namespace: default

runtime:
  default_model: default
  models:
    - name: default
      description: General purpose chat model
      provider: universal
      model: unsloth/Qwen3-4B-GGUF:Q4_K_M
      base_url: http://127.0.0.1:11540/v1
      default: true
      prompt_format: unstructured
      model_api_parameters:
        temperature: 0.7
        max_tokens: 2048

prompts:
  - name: default
    messages:
      - role: system
        content: |
          You are a helpful research assistant. Use the provided context
          to answer questions accurately. If the context doesn't contain
          relevant information, say so clearly.

          When citing information, reference the source document.

rag:
  default_database: main_db
  databases:
    - name: main_db
      type: ChromaStore
      config:
        collection_name: quick_rag_docs
        distance_function: cosine
        persist_directory: ./data/main_db
      embedding_strategies:
        - name: default_embeddings
          type: UniversalEmbedder
          config:
            model: nomic-ai/nomic-embed-text-v2-moe
            dimension: 768
            batch_size: 32
      retrieval_strategies:
        - name: basic_search
          type: BasicSimilarityStrategy
          config:
            top_k: 5
            score_threshold: 0.3
          default: true
      default_embedding_strategy: default_embeddings
      default_retrieval_strategy: basic_search

  data_processing_strategies:
    - name: markdown_processor
      description: Process markdown documentation files
      parsers:
        - type: MarkdownParser_LlamaIndex
          file_include_patterns:
            - "*.md"
            - "*.markdown"
          priority: 100
          config:
            chunk_size: 500
            chunk_overlap: 50
            chunk_strategy: headings
      extractors:
        - type: HeadingExtractor
          config:
            max_level: 4
            include_hierarchy: true
        - type: ContentStatisticsExtractor
          config:
            include_readability: true

datasets:
  - name: research
    database: main_db
    data_processing_strategy: markdown_processor
```

## Sample Files

### neural_scaling.md

```markdown
# Neural Scaling Laws

## Overview

Neural scaling laws describe how model performance improves with
increased compute, data, and model size.

## Key Findings

- Performance scales predictably with model size
- Larger models are more sample-efficient
- Compute optimal training exists at specific ratios

## Implications

Understanding scaling laws helps:
- Predict model capabilities
- Optimize training budgets
- Plan infrastructure needs
```

### engineering_practices.md

```markdown
# Software Engineering Best Practices

## Code Quality

- Write clear, readable code
- Use meaningful variable names
- Keep functions small and focused

## Testing

- Write unit tests for all logic
- Use integration tests for workflows
- Maintain high test coverage

## Documentation

- Document public APIs
- Keep README files updated
- Use inline comments sparingly
```

## Workflow

### Setup

```bash
# 1. Ensure Universal Runtime is running
# Start the universal runtime server on port 11540
# The model will be downloaded automatically on first use

# 2. Scaffold the example
/example quick_rag

# 3. Navigate to project
cd quick_rag
```

### Run

```bash
# Start LlamaFarm services
lf start

# Create dataset
lf datasets create -s markdown_processor -b main_db research

# Upload sample files
lf datasets upload research ./files/*

# Process (generate embeddings)
lf datasets process research

# Wait for processing to complete
lf datasets status research
```

### Query

```bash
# Chat with RAG context
lf chat "What are neural scaling laws?"

# Expected output:
# Neural scaling laws describe how model performance improves
# with increased compute, data, and model size. Key findings
# include that performance scales predictably with model size,
# and larger models are more sample-efficient...
# [Source: neural_scaling.md]
```

## Configuration Explained

### Why These Settings?

**Chunk size: 500**
- Small enough for markdown sections
- Large enough for context

**Chunk overlap: 50 (10%)**
- Preserves context at boundaries
- Minimal redundancy

**top_k: 5**
- Enough context for simple queries
- Fast retrieval

**score_threshold: 0.3**
- Filters low-relevance results
- Prevents noise in context

### Customization Points

**For more context:**
```yaml
retrieval_strategies:
  - name: basic_search
    config:
      top_k: 10  # More chunks
```

**For technical docs:**
```yaml
parsers:
  - type: MarkdownParser_LlamaIndex
    config:
      chunk_strategy: code  # Preserve code blocks
```

**For faster embedding:**
```yaml
embedding_strategies:
  - name: default_embeddings
    type: OllamaEmbedder
    config:
      model: nomic-embed-text
```

## Troubleshooting

### "Model not found"

```bash
# Verify Universal Runtime is running and the model is available
curl http://127.0.0.1:11540/v1/models
```

### "Connection refused"

```bash
# Start Universal Runtime
# Check it's running
curl http://127.0.0.1:11540/v1/models
```

### "No relevant chunks found"

- Verify files were uploaded: `lf datasets list-files research`
- Check processing completed: `lf datasets status research`
- Lower score_threshold to 0.1 for testing
