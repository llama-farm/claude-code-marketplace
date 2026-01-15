# PDF Documents Pattern

For regulatory, legal, FDA, contracts, policies, and general PDF analysis.

## Model Configuration

```yaml
runtime:
  models:
    - name: default
      description: "PDF document analysis"
      provider: ollama
      model: llama3.1:8b
      base_url: http://localhost:11434
      default: true
      prompt_format: unstructured
      model_api_parameters:
        temperature: 0.2    # Lower for precision
```

## System Prompt

```yaml
prompts:
  - name: default
    messages:
      - role: system
        content: |
          You are an expert document analysis assistant.
          Always cite specific document sections and identifiers when referencing information.
          If information cannot be found in the provided context, state that clearly.
```

## Data Processing Strategy

```yaml
data_processing_strategies:
  - name: pdf_processor
    description: "Parser tuned for PDF documents"
    parsers:
      # Primary parser - handles complex PDFs
      - type: PDFParser_LlamaIndex
        file_include_patterns: ["*.pdf", "*.PDF"]
        priority: 100
        config:
          chunk_size: 1200
          chunk_overlap: 150
          chunk_strategy: semantic
          extract_metadata: true
          extract_tables: true
      # Fallback parser - simpler but reliable
      - type: PDFParser_PyPDF2
        file_include_patterns: ["*.pdf", "*.PDF"]
        priority: 50
        config:
          chunk_size: 1000
          chunk_overlap: 120
          chunk_strategy: paragraphs
          extract_metadata: true
    extractors:
      - type: EntityExtractor
        file_include_patterns: ["*"]
        priority: 100
        config:
          entity_types: [ORG, DATE, PERSON, PRODUCT]
          use_fallback: true
          min_entity_length: 2
      - type: ContentStatisticsExtractor
        file_include_patterns: ["*"]
        priority: 90
        config:
          include_readability: true
          include_structure: true
```

## Retrieval Configuration

```yaml
retrieval_strategies:
  - name: document_search
    type: BasicSimilarityStrategy
    config:
      distance_metric: cosine
      top_k: 8              # More results for citation support
    default: true
```

## Variations

### FDA/Regulatory Focus
Add to entity_types: `[ORG, DATE, PERSON, PRODUCT, LAW, MONEY]`

### Legal/Contracts
- Increase chunk_size to 1500 for longer clauses
- Add TableExtractor for contract tables

### Financial Documents
- Add MONEY, PERCENT to entity_types
- Add DateTimeExtractor for dates
