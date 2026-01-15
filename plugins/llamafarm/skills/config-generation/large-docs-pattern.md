# Large Complex Documents Pattern

For multi-megabyte documents: ordinances, manuals, comprehensive reports, technical specifications.

## Model Configuration

```yaml
runtime:
  models:
    - name: default
      description: "Large document specialist"
      provider: ollama
      model: llama3.1:8b
      base_url: http://localhost:11434
      default: true
      prompt_format: unstructured
      model_api_parameters:
        temperature: 0.2    # Precision for complex content
```

## System Prompt

```yaml
prompts:
  - name: default
    messages:
      - role: system
        content: |
          You are a specialist assistant for comprehensive technical documents.
          Use retrieved content only and answer in plain language.
          Cite relevant section identifiers (e.g., "Section X.X.X") and page numbers when available.
          If the answer cannot be derived from the context, respond clearly.
```

## Data Processing Strategy

```yaml
data_processing_strategies:
  - name: large_pdf_processor
    description: "Parser tuned for large complex PDFs with heading extraction"
    parsers:
      # Primary - handles large documents well
      - type: PDFParser_LlamaIndex
        file_include_patterns: ["*.pdf", "*.PDF"]
        priority: 100
        config:
          chunk_size: 1100
          chunk_overlap: 160       # More overlap for context
          chunk_strategy: semantic
          extract_metadata: true
          extract_tables: true

      # Fallback for problem pages
      - type: PDFParser_PyPDF2
        file_include_patterns: ["*.pdf", "*.PDF"]
        priority: 50
        config:
          chunk_size: 900
          chunk_overlap: 140
          chunk_strategy: paragraphs
          extract_metadata: true

    extractors:
      # Heading structure is critical for large docs
      - type: HeadingExtractor
        file_include_patterns: ["*"]
        priority: 100
        config:
          max_level: 4
          include_hierarchy: true
          extract_outline: true

      - type: ContentStatisticsExtractor
        file_include_patterns: ["*"]
        priority: 90
        config:
          include_structure: true
          include_readability: true

      - type: EntityExtractor
        file_include_patterns: ["*"]
        priority: 80
        config:
          entity_types: [ORG, DATE, FAC, GPE, LOC]
          use_fallback: true
          min_entity_length: 2

      # Tables often contain critical data
      - type: TableExtractor
        file_include_patterns: ["*.pdf"]
        priority: 70
        config:
          output_format: dict
          extract_headers: true
```

## Retrieval Configuration

```yaml
retrieval_strategies:
  - name: document_search
    type: BasicSimilarityStrategy
    config:
      distance_metric: cosine
      top_k: 6             # Fewer but more relevant
    default: true
```

## Variations

### Municipal Ordinances
- Entity types: [ORG, DATE, FAC, GPE, LOC, LAW]
- Add TableExtractor for zoning tables

### Technical Manuals
- Larger chunk_size (1200-1400)
- Add PatternExtractor for model numbers, part codes

### Annual Reports
- Add MONEY, PERCENT to entity_types
- Add DateTimeExtractor for fiscal dates
