# Mixed Formats Pattern

For projects with various document types: PDFs, Word docs, markdown, HTML, CSV, code.

## Model Configuration

```yaml
runtime:
  models:
    - name: default
      description: "Multi-format document analysis"
      provider: ollama
      model: llama3.1:8b
      base_url: http://localhost:11434
      default: true
      prompt_format: unstructured
```

## System Prompt

```yaml
prompts:
  - name: default
    messages:
      - role: system
        content: |
          You are a research assistant analyzing project documentation.
          Consider information from multiple document types.
          Cite sources and file names when referencing specific information.
```

## Retrieval Configuration (Hybrid)

```yaml
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
```

## Data Processing Strategy

```yaml
data_processing_strategies:
  - name: universal_processor
    description: "Handles PDFs, Word docs, CSVs, Markdown, and text files"
    parsers:
      # PDF files
      - type: PDFParser_LlamaIndex
        file_include_patterns: ["*.pdf", "*.PDF"]
        priority: 100
        config:
          chunk_size: 1000
          chunk_overlap: 120
          extract_metadata: true

      # Word documents
      - type: DocxParser_LlamaIndex
        file_include_patterns: ["*.docx", "*.DOCX"]
        priority: 100
        config:
          chunk_size: 800
          chunk_overlap: 80
          extract_metadata: true

      # Markdown
      - type: MarkdownParser_LlamaIndex
        file_include_patterns: ["*.md", "*.markdown"]
        priority: 100
        config:
          chunk_size: 600
          chunk_overlap: 60

      # CSV/Excel
      - type: CSVParser_Pandas
        file_include_patterns: ["*.csv", "*.CSV", "*.tsv"]
        priority: 100
        config:
          chunk_size: 1000
          chunk_strategy: rows

      - type: ExcelParser_Pandas
        file_include_patterns: ["*.xlsx", "*.xls"]
        priority: 100
        config:
          chunk_size: 500
          sheets: null  # All sheets

      # Text/HTML/Code (fallback)
      - type: TextParser_Python
        file_include_patterns: ["*.txt", "*.html", "*.py", "*.js"]
        priority: 50
        config:
          chunk_size: 600
          chunk_overlap: 60

    extractors:
      - type: ContentStatisticsExtractor
        file_include_patterns: ["*"]
        priority: 100
        config:
          include_readability: true
          include_structure: true
      - type: EntityExtractor
        file_include_patterns: ["*"]
        priority: 90
        config:
          entity_types: [ORG, PERSON, DATE, PRODUCT]
          use_fallback: true
```

## Variations

### Project Documentation
- Add HeadingExtractor for markdown
- Add LinkExtractor for references

### Research Repository
- Add KeywordExtractor
- Increase top_k for comprehensive search

### Business Documents
- Add TableExtractor for spreadsheet data
- Add DateTimeExtractor for dates
