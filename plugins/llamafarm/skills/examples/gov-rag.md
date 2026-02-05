# gov_rag Example

Government and municipal document processing for large, complex PDFs.

## Overview

**Purpose:** Process city ordinances, planning documents, and municipal codes

**Documents:** Large government documents (often 100+ pages)

**Features:**
- Large document handling
- Hierarchy/outline extraction
- Geospatial entity extraction
- Table processing

**Best for:** Municipal planning, legal research, policy analysis

## Complete Configuration

```yaml
version: v1
name: government-docs-rag
namespace: default

runtime:
  default_model: default
  models:
    - name: default
      description: Government document analyst
      provider: universal
      model: unsloth/Qwen3-4B-GGUF:Q4_K_M
      base_url: http://127.0.0.1:11540/v1
      default: true
      model_api_parameters:
        temperature: 0.3
        max_tokens: 4096

    - name: fast
      description: Quick responses for simple queries
      provider: universal
      model: unsloth/Qwen3-1.7B-GGUF:Q4_K_M
      base_url: http://127.0.0.1:11540/v1
      model_api_parameters:
        temperature: 0.5
        max_tokens: 1024

prompts:
  - name: default
    messages:
      - role: system
        content: |
          You are a municipal policy analyst specializing in:
          - City ordinances and codes
          - Zoning regulations
          - Planning documents
          - Government procedures

          When answering:
          1. Cite specific ordinance numbers and sections
          2. Reference page numbers when available
          3. Note effective dates and amendments
          4. Identify geographic areas affected

          Use precise legal terminology when discussing regulations.
          Clarify any ambiguities in the regulatory language.

rag:
  default_database: gov_db
  databases:
    - name: gov_db
      type: ChromaStore
      config:
        collection_name: government_documents
        distance_function: cosine
        persist_directory: ./data/gov_db
      embedding_strategies:
        - name: default_embeddings
          type: UniversalEmbedder
          config:
            model: nomic-ai/nomic-embed-text-v2-moe
            dimension: 768
            batch_size: 8  # Smaller for memory efficiency
      retrieval_strategies:
        - name: semantic_search
          type: BasicSimilarityStrategy
          config:
            top_k: 15  # More context for legal docs
            score_threshold: 0.2
          default: true
        - name: hybrid_search
          type: HybridUniversalStrategy
          config:
            strategies:
              - type: BasicSimilarityStrategy
                config:
                  top_k: 10
                weight: 0.6
            combination_method: weighted_average
            final_k: 15
      default_embedding_strategy: default_embeddings
      default_retrieval_strategy: semantic_search

  data_processing_strategies:
    - name: large_doc_processor
      description: Process large government PDFs with hierarchy extraction
      parsers:
        - type: PDFParser_LlamaIndex
          file_include_patterns:
            - "*.pdf"
          priority: 100
          config:
            chunk_size: 1100
            chunk_overlap: 160
            chunk_strategy: semantic
            extract_tables: true
            extract_metadata: true
            preserve_layout: true
        - type: PDFParser_PyPDF2
          file_include_patterns:
            - "*.pdf"
          priority: 50
          config:
            chunk_size: 900
            chunk_overlap: 140
            chunk_strategy: paragraphs
      extractors:
        - type: HeadingExtractor
          config:
            max_level: 6
            include_hierarchy: true
            extract_outline: true  # PDF bookmarks
        - type: EntityExtractor
          config:
            entity_types:
              - GPE    # Cities, counties, districts
              - FAC    # Buildings, facilities
              - LOC    # Geographic locations
              - ORG    # Departments, agencies
              - DATE   # Effective dates
              - MONEY  # Fees, budgets
            use_fallback: true
        - type: TableExtractor
          config:
            extract_headers: true
            preserve_structure: true
            min_rows: 2
        - type: ContentStatisticsExtractor
          config:
            include_structure: true
            include_readability: true

datasets:
  - name: ordinances
    database: gov_db
    data_processing_strategy: large_doc_processor
  - name: planning_docs
    database: gov_db
    data_processing_strategy: large_doc_processor
  - name: municipal_codes
    database: gov_db
    data_processing_strategy: large_doc_processor
```

## Key Configuration Choices

### Chunk Size: 1100 / Overlap: 160

Large government documents have:
- Legal definitions spanning paragraphs
- Cross-references to other sections
- Complex regulatory language

**14.5% overlap** ensures section boundaries are captured in multiple chunks.

### Hierarchy Extraction

Government documents are deeply structured:

```
Chapter 12: Zoning
  Article 12.1: General Provisions
    Section 12.1.1: Purpose
    Section 12.1.2: Definitions
  Article 12.2: Residential Districts
    Section 12.2.1: R-1 Single Family
    Section 12.2.2: R-2 Multi-Family
```

HeadingExtractor preserves this hierarchy in metadata for:
- Contextual search
- Section navigation
- Citation generation

### Geospatial Entities

Government docs reference many locations:

| Entity Type | Examples |
|-------------|----------|
| GPE | "City of Springfield", "District 4" |
| FAC | "City Hall", "Memorial Park" |
| LOC | "Main Street corridor", "downtown area" |

### Table Extraction

Common tables in municipal documents:
- Zoning use matrices
- Fee schedules
- Setback requirements
- Parking requirements

## Workflow

### Setup

```bash
# Scaffold example
/example gov_rag

cd gov_rag
```

### Process Large Documents

```bash
# Start services
lf start

# Create datasets
lf datasets create -s large_doc_processor -b gov_db ordinances
lf datasets create -s large_doc_processor -b gov_db planning_docs

# Upload documents (may take time for large files)
lf datasets upload ordinances ./files/ordinances/*.pdf
lf datasets upload planning_docs ./files/planning/*.pdf

# Process - this may take several minutes for large documents
lf datasets process ordinances
lf datasets process planning_docs

# Monitor progress
lf datasets status ordinances
```

### Query Examples

**Zoning questions:**
```bash
lf chat "What are the setback requirements for R-1 residential zones?"
```

**Fee lookups:**
```bash
lf chat "What are the building permit fees for new construction?"
```

**Location-specific:**
```bash
lf rag query --filter "entity_gpe=District 4" "parking requirements"
```

## Sample Queries

### Zoning Research

```
Q: What uses are permitted in the C-2 Commercial zone?

A: According to the Zoning Ordinance, Chapter 4, Section 4.2.3
(C-2 General Commercial District), the following uses are permitted
by right:

Permitted Uses:
- Retail sales and services
- Restaurants and cafes
- Professional offices
- Personal services (salons, dry cleaners)
- Financial institutions

Conditional Uses (require special permit):
- Drive-through facilities
- Outdoor dining
- Entertainment venues

[Source: ordinances/zoning_code.pdf, Page 47-52, Table 4-2]
```

### Fee Schedule Lookup

```
Q: What are the fees for a variance application?

A: The Development Services fee schedule (Ordinance 2024-15,
effective July 1, 2024) lists the following variance fees:

| Application Type | Fee |
|-----------------|-----|
| Minor Variance | $250 |
| Major Variance | $500 |
| Commercial Variance | $750 |
| Sign Variance | $150 |

Additional fees may apply:
- Public hearing notice: $75
- Environmental review: $200-500

[Source: ordinances/fee_schedule.pdf, Page 12]
```

## Handling Large Documents

### Memory Management

For documents over 50MB:

```yaml
embedding_strategies:
  - name: default_embeddings
    config:
      batch_size: 4  # Reduce from default 8
```

### Processing Time

Large documents may take 5-15 minutes to process:

```bash
# Check progress
lf datasets status ordinances

# View processing logs
/logs --service rag
```

### Incremental Updates

For frequently updated codes:

```bash
# Add new documents without reprocessing existing
lf datasets upload ordinances ./new_amendments/*.pdf
lf datasets process ordinances --incremental
```

## Customization

### For Building Codes

Add pattern extraction for code citations:

```yaml
extractors:
  - type: PatternExtractor
    config:
      custom_patterns:
        - name: ibc_reference
          pattern: "IBC\\s+\\d{4}.*?Section\\s+[\\d.]+"
          description: "International Building Code citations"
        - name: ordinance_number
          pattern: "Ord\\.?\\s*(?:No\\.?)?\\s*\\d{4}-\\d+"
          description: "Ordinance numbers"
```

### For Meeting Minutes

Adjust for conversational format:

```yaml
parsers:
  - type: MarkdownParser_LlamaIndex
    config:
      chunk_size: 800
      chunk_strategy: paragraphs
```

### For Parcel Data

Add tabular extraction focus:

```yaml
extractors:
  - type: TableExtractor
    config:
      extract_all: true
      table_format: markdown
```

## Troubleshooting

### Out of Memory

For very large documents (100+ MB):
1. Split PDF into sections
2. Process sections sequentially
3. Use Universal Runtime for embeddings

### Slow Retrieval

With thousands of chunks:
1. Add metadata filters
2. Use hybrid search for precision
3. Consider Qdrant for larger scale

### Missing Sections

If document structure isn't captured:
1. Check if PDF has bookmarks (`extract_outline: true`)
2. Verify heading levels are consistent
3. Add custom heading patterns if needed
