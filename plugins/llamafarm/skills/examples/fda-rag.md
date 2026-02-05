# fda_rag Example

FDA regulatory document analysis with entity extraction and citation tracking.

## Overview

**Purpose:** Analyze FDA correspondence, warning letters, and regulatory documents

**Documents:** FDA warning letters and regulatory correspondence (PDFs)

**Features:**
- PDF parsing with table extraction
- Entity extraction (organizations, dates, products)
- Semantic chunking for legal text
- Citation-aware responses

**Best for:** Regulatory analysis, legal document review, compliance work

## Complete Configuration

```yaml
version: v1
name: fda-regulatory-analysis
namespace: default

runtime:
  default_model: default
  models:
    - name: default
      description: Regulatory document analysis assistant
      provider: universal
      model: unsloth/Qwen3-4B-GGUF:Q4_K_M
      base_url: http://127.0.0.1:11540/v1
      default: true
      model_api_parameters:
        temperature: 0.2  # Lower for factual accuracy
        max_tokens: 4096  # Longer for detailed analysis

prompts:
  - name: default
    messages:
      - role: system
        content: |
          You are an expert FDA regulatory analyst. Your role is to:

          1. Analyze FDA correspondence and warning letters accurately
          2. Identify key regulatory citations and references
          3. Extract important dates, organizations, and products
          4. Provide precise answers with specific document citations

          When responding:
          - Always cite the specific document and section
          - Use exact quotes when referencing regulatory language
          - Note any uncertainties or ambiguities
          - Highlight compliance deadlines and requirements

          If information is not in the provided context, state that clearly
          rather than speculating.

rag:
  default_database: fda_db
  databases:
    - name: fda_db
      type: ChromaStore
      config:
        collection_name: fda_documents
        distance_function: cosine
        persist_directory: ./data/fda_db
      embedding_strategies:
        - name: regulatory_embeddings
          type: UniversalEmbedder
          config:
            model: nomic-ai/nomic-embed-text-v2-moe
            dimension: 768
            batch_size: 16  # Smaller for large PDFs
      retrieval_strategies:
        - name: semantic_search
          type: BasicSimilarityStrategy
          config:
            top_k: 10
            score_threshold: 0.25
          default: true
        - name: entity_filtered
          type: MetadataFilteredStrategy
          config:
            top_k: 15
            filters:
              - field: entity_type
                operator: in
                values: [ORG, PRODUCT]
      default_embedding_strategy: regulatory_embeddings
      default_retrieval_strategy: semantic_search

  data_processing_strategies:
    - name: fda_processor
      description: Process FDA regulatory PDFs with entity extraction
      parsers:
        - type: PDFParser_LlamaIndex
          file_include_patterns:
            - "*.pdf"
            - "*.PDF"
          priority: 100
          config:
            chunk_size: 1200
            chunk_overlap: 150
            chunk_strategy: semantic
            extract_metadata: true
            extract_tables: true
            preserve_layout: true
        - type: PDFParser_PyPDF2
          file_include_patterns:
            - "*.pdf"
            - "*.PDF"
          priority: 50
          config:
            chunk_size: 1000
            chunk_overlap: 120
            chunk_strategy: paragraphs
      extractors:
        - type: EntityExtractor
          config:
            entity_types:
              - ORG      # FDA, companies
              - DATE     # Deadlines, issue dates
              - PERSON   # Officials, contacts
              - PRODUCT  # Drugs, devices
              - LAW      # Regulations cited
              - MONEY    # Fines, amounts
            use_fallback: true
            min_confidence: 0.7
        - type: ContentStatisticsExtractor
          config:
            include_readability: true
            include_structure: true
        - type: DateTimeExtractor
          config:
            extract_relative: true
            normalize_dates: true

datasets:
  - name: warning_letters
    database: fda_db
    data_processing_strategy: fda_processor
  - name: correspondence
    database: fda_db
    data_processing_strategy: fda_processor
```

## Key Configuration Choices

### Chunk Size: 1200 / Overlap: 150

Legal and regulatory documents contain:
- Long, complex sentences
- Cross-references between sections
- Context-dependent terminology

Larger chunks preserve context; 12.5% overlap ensures continuity.

### Semantic Chunking

FDA documents have natural section boundaries:
- Violation summaries
- Regulatory citations
- Required actions
- Response deadlines

Semantic chunking respects these boundaries.

### Entity Extraction

Critical entities in FDA documents:

| Entity | Examples |
|--------|----------|
| ORG | FDA, company names, divisions |
| DATE | Issue dates, response deadlines |
| PERSON | FDA officials, company contacts |
| PRODUCT | Drug names, device identifiers |
| LAW | 21 CFR citations, FD&C Act |
| MONEY | Penalties, consent decree amounts |

### Parser Fallback

PDFParser_LlamaIndex handles complex layouts but may fail on:
- Scanned documents
- Unusual PDF structures
- Protected files

PyPDF2 fallback ensures processing continues.

## Workflow

### Setup

```bash
# Ensure dependencies
pip install llama-index pypdf2

# Scaffold example
/example fda_rag

cd fda_rag
```

### Process Documents

```bash
# Start services
lf start

# Create datasets
lf datasets create -s fda_processor -b fda_db warning_letters
lf datasets create -s fda_processor -b fda_db correspondence

# Upload PDFs
lf datasets upload warning_letters ./files/warning_letters/*.pdf
lf datasets upload correspondence ./files/correspondence/*.pdf

# Process
lf datasets process warning_letters
lf datasets process correspondence
```

### Query Examples

**Compliance deadlines:**
```bash
lf chat "What are the response deadlines in the most recent warning letters?"
```

**Regulatory citations:**
```bash
lf chat "Which CFR sections are cited in the Acme Corp warning letter?"
```

**Entity-filtered search:**
```bash
lf rag query --strategy entity_filtered --filter "entity_type=ORG" \
  "manufacturing violations"
```

## Sample Queries

### Finding Violations

```
Q: What GMP violations were cited in warning letters from 2024?

A: Based on the warning letters from 2024, the following GMP violations
were cited:

1. ABC Pharmaceuticals (Warning Letter WL-2024-0142, dated January 15, 2024):
   - 21 CFR 211.68(a): Failure to calibrate equipment
   - 21 CFR 211.160(b): Failure to establish sampling procedures
   [Source: warning_letters/abc_pharma_wl.pdf, Section II]

2. XYZ Medical Devices (Warning Letter WL-2024-0156, dated February 3, 2024):
   - 21 CFR 820.30(d): Design input requirements not met
   [Source: warning_letters/xyz_devices_wl.pdf, Section III]
```

### Tracking Deadlines

```
Q: What is the response deadline for Acme Corp?

A: According to the warning letter dated March 10, 2024, Acme Corporation
must respond within 15 working days of receipt. The letter states:

"You should respond in writing within fifteen (15) working days of receipt
of this letter."

Based on standard FDA practices, assuming receipt within 3 business days,
the response would be due approximately April 1, 2024.

[Source: warning_letters/acme_corp_wl.pdf, Page 5]
```

## Customization

### For 483 Observations

Add observations dataset with focused extraction:

```yaml
datasets:
  - name: form_483s
    database: fda_db
    data_processing_strategy: fda_processor
```

### For Consent Decrees

Add MONEY entity extraction and longer chunks:

```yaml
extractors:
  - type: EntityExtractor
    config:
      entity_types: [ORG, MONEY, DATE]
```

### For Import Alerts

Adjust for tabular data:

```yaml
extractors:
  - type: TableExtractor
    config:
      extract_headers: true
      preserve_structure: true
```

## Troubleshooting

### Poor PDF Extraction

If text is garbled or missing:
1. Check if PDF is scanned (use OCR)
2. Try PyPDF2 fallback only
3. Pre-process with `pdftotext`

### Missing Entities

If entities aren't extracted:
1. Lower `min_confidence` to 0.5
2. Add custom patterns for regulatory terms
3. Check chunk size isn't too small

### Slow Processing

Large regulatory documents can be slow:
1. Reduce `batch_size` for embeddings
2. Process fewer files at once
3. Use Universal Runtime for faster embeddings
