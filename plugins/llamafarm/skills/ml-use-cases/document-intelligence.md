# Document Intelligence Pipeline

End-to-end pipeline for extracting structured knowledge from documents using OCR, NER, classification, and RAG indexing.

## Architecture Overview

```
Document (PDF/image)
    │
    ├─→ OCR (Surya) ─→ Raw text
    │
    ├─→ NER ─→ Entities (people, orgs, dates, amounts)
    │
    ├─→ Classification ─→ Document type (invoice, contract, report)
    │
    └─→ RAG Index ─→ Enriched chunks with entity metadata
                         │
                         └─→ Query with entity filters
```

**Why combine these features:**
- OCR provides the raw text from scanned or image-based documents.
- NER extracts structured entities that become searchable metadata.
- Classification routes documents to the correct processing logic.
- RAG indexes the enriched content for semantic retrieval.

Each stage adds a layer of structure. By the time a document reaches the RAG index, it carries its text, entities, and type label as queryable metadata.

---

## Step 1: OCR Extraction

Extract text from scanned documents or images using the Surya backend for high-quality multilingual OCR.

```bash
curl -X POST http://localhost:14345/v1/ml/ocr/extract \
  -H "Content-Type: application/json" \
  -d '{
    "source": "/path/to/document.pdf",
    "backend": "surya",
    "options": {
      "languages": ["en"],
      "detect_layout": true,
      "extract_tables": true
    }
  }'
```

**Response:**
```json
{
  "text": "INVOICE\n\nBill To: Acme Corporation\nDate: 2025-01-15\nInvoice #: INV-2025-0042\n\nDescription          Qty   Unit Price   Total\nConsulting Services    40   $150.00      $6,000.00\nTravel Expenses         1   $450.00        $450.00\n\nSubtotal: $6,450.00\nTax (8%): $516.00\nTotal Due: $6,966.00",
  "pages": 1,
  "confidence": 0.94,
  "tables": [
    {
      "page": 1,
      "headers": ["Description", "Qty", "Unit Price", "Total"],
      "rows": [
        ["Consulting Services", "40", "$150.00", "$6,000.00"],
        ["Travel Expenses", "1", "$450.00", "$450.00"]
      ]
    }
  ],
  "layout": {
    "blocks": [
      {"type": "title", "text": "INVOICE", "bbox": [100, 50, 300, 80]},
      {"type": "text", "text": "Bill To: Acme Corporation...", "bbox": [50, 100, 400, 200]},
      {"type": "table", "bbox": [50, 250, 550, 400]}
    ]
  }
}
```

**Backend options:**
| Backend | Speed | Quality | Best For |
|---------|-------|---------|----------|
| `surya` | Moderate | High | Production, multilingual |
| `tesseract` | Fast | Good | English-only, quick extraction |
| `easyocr` | Slow | Good | Many languages, handwriting |

Use `surya` for production pipelines. Use `tesseract` when speed matters and documents are clean English text.

---

## Step 2: NER Extraction

Extract named entities from the OCR text. Entities become searchable metadata in the RAG index.

```bash
curl -X POST http://localhost:14345/v1/ml/ner/extract \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Bill To: Acme Corporation\nDate: 2025-01-15\nInvoice #: INV-2025-0042\nTotal Due: $6,966.00",
    "entity_types": ["ORG", "DATE", "MONEY", "PERSON"]
  }'
```

**Response:**
```json
{
  "entities": [
    {
      "text": "Acme Corporation",
      "type": "ORG",
      "start": 9,
      "end": 25,
      "confidence": 0.97
    },
    {
      "text": "2025-01-15",
      "type": "DATE",
      "start": 32,
      "end": 42,
      "confidence": 0.99
    },
    {
      "text": "$6,966.00",
      "type": "MONEY",
      "start": 78,
      "end": 87,
      "confidence": 0.95
    }
  ]
}
```

**Supported entity types:**
| Type | Examples |
|------|----------|
| `PERSON` | John Smith, Dr. Jane Doe |
| `ORG` | Acme Corp, FDA, Department of Defense |
| `DATE` | 2025-01-15, January 15th, Q1 2025 |
| `MONEY` | $6,966.00, USD 1,000 |
| `LOCATION` | New York, 123 Main St |
| `REGULATION` | 21 CFR Part 11, GDPR Article 5 |

---

## Step 3: Document Classification

Classify the document type to route it through the appropriate processing logic.

```bash
curl -X POST http://localhost:14345/v1/ml/classify/text \
  -H "Content-Type: application/json" \
  -d '{
    "text": "INVOICE\nBill To: Acme Corporation\nDate: 2025-01-15\nInvoice #: INV-2025-0042\nTotal Due: $6,966.00",
    "labels": ["invoice", "contract", "report", "memo", "receipt"],
    "mode": "zero_shot"
  }'
```

**Response:**
```json
{
  "predictions": [
    {"label": "invoice", "score": 0.92},
    {"label": "receipt", "score": 0.05},
    {"label": "report", "score": 0.02},
    {"label": "contract", "score": 0.01},
    {"label": "memo", "score": 0.00}
  ],
  "top_label": "invoice",
  "top_score": 0.92
}
```

**Classification modes:**
| Mode | Description | When to Use |
|------|-------------|-------------|
| `zero_shot` | No training data required, uses label descriptions | Quick start, broad categories |
| `setfit` | Fine-tuned on your labeled examples | High accuracy, specific categories |

For production pipelines with known document types, train a SetFit model for better accuracy.

---

## Step 4: RAG Indexing with Enriched Metadata

Store the extracted text in the RAG index with entities and classification as metadata. This enables filtered retrieval by entity, document type, or date range.

```bash
# Create a dataset for enriched documents
curl -X POST http://localhost:14345/v1/projects/default/doc-intel/datasets \
  -H "Content-Type: application/json" \
  -d '{
    "name": "enriched_docs",
    "database": "main_db",
    "data_processing_strategy": "default_processor"
  }'
```

```bash
# Index with metadata from OCR + NER + classification
curl -X POST http://localhost:14345/v1/projects/default/doc-intel/rag/index \
  -H "Content-Type: application/json" \
  -d '{
    "documents": [
      {
        "content": "INVOICE\nBill To: Acme Corporation\nDate: 2025-01-15...",
        "metadata": {
          "source": "invoice_scan_042.pdf",
          "doc_type": "invoice",
          "doc_type_confidence": 0.92,
          "entities_org": ["Acme Corporation"],
          "entities_date": ["2025-01-15"],
          "entities_money": ["$6,966.00"],
          "ocr_confidence": 0.94,
          "page_count": 1
        }
      }
    ],
    "database": "main_db"
  }'
```

### Querying with Entity Filters

Once documents are indexed with entity metadata, use filters to narrow retrieval.

```bash
# Find all invoices from Acme Corporation
curl -X POST http://localhost:14345/v1/projects/default/doc-intel/rag/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "consulting services invoice",
    "database": "main_db",
    "top_k": 5,
    "filters": {
      "metadata.doc_type": "invoice",
      "metadata.entities_org": "Acme Corporation"
    }
  }'
```

```bash
# Find all documents mentioning dates in January 2025
curl -X POST http://localhost:14345/v1/projects/default/doc-intel/rag/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "January 2025 deliverables",
    "database": "main_db",
    "top_k": 10,
    "filters": {
      "metadata.entities_date": "2025-01-15"
    }
  }'
```

---

## Combining ML Features

The power of this pipeline comes from using each ML feature's output as input to the next stage. Here is how the pieces connect.

### NER Entities as RAG Metadata

Entities extracted by NER become metadata fields on RAG chunks. This turns unstructured text into a queryable knowledge base.

```python
# Extract entities, then attach to RAG document
entities = ner_extract(text)

metadata = {
    "entities_org": [e["text"] for e in entities if e["type"] == "ORG"],
    "entities_date": [e["text"] for e in entities if e["type"] == "DATE"],
    "entities_money": [e["text"] for e in entities if e["type"] == "MONEY"],
}
```

### Classification as Routing

Classification determines which processing path a document follows.

```python
doc_type = classify(text)["top_label"]

if doc_type == "invoice":
    # Extract line items, amounts, due dates
    process_invoice(text)
elif doc_type == "contract":
    # Extract parties, terms, obligations
    process_contract(text)
elif doc_type == "report":
    # Extract findings, recommendations
    process_report(text)
```

### OCR Confidence as Quality Gate

Low OCR confidence indicates poor scan quality. Route low-confidence documents for manual review.

```python
ocr_result = ocr_extract(document_path)

if ocr_result["confidence"] < 0.7:
    flag_for_manual_review(document_path)
else:
    proceed_with_pipeline(ocr_result["text"])
```

---

## Full Working Example

Complete script that processes a batch of documents through the entire pipeline.

```python
import requests
from pathlib import Path

BASE = "http://localhost:14345/v1"
ML_BASE = f"{BASE}/ml"
PROJECT = "default/doc-intel"


def ocr_extract(file_path):
    """Extract text from a document using OCR."""
    resp = requests.post(f"{ML_BASE}/ocr/extract", json={
        "source": str(file_path),
        "backend": "surya",
        "options": {
            "languages": ["en"],
            "detect_layout": True,
            "extract_tables": True,
        },
    })
    return resp.json()


def ner_extract(text):
    """Extract named entities from text."""
    resp = requests.post(f"{ML_BASE}/ner/extract", json={
        "text": text,
        "entity_types": ["ORG", "DATE", "MONEY", "PERSON"],
    })
    return resp.json()["entities"]


def classify(text):
    """Classify document type."""
    resp = requests.post(f"{ML_BASE}/classify/text", json={
        "text": text[:2000],  # First 2000 chars is enough for classification
        "labels": ["invoice", "contract", "report", "memo", "receipt"],
        "mode": "zero_shot",
    })
    return resp.json()


def index_document(content, metadata):
    """Index enriched document in RAG."""
    resp = requests.post(
        f"{BASE}/projects/{PROJECT}/rag/index",
        json={
            "documents": [
                {"content": content, "metadata": metadata}
            ],
            "database": "main_db",
        },
    )
    return resp.json()


def process_document(file_path):
    """Run a single document through the full pipeline."""
    print(f"\nProcessing: {file_path}")

    # Step 1: OCR
    ocr = ocr_extract(file_path)
    text = ocr["text"]
    print(f"  OCR: {len(text)} chars, confidence={ocr['confidence']:.2f}")

    if ocr["confidence"] < 0.7:
        print("  WARNING: Low OCR confidence, flagging for review")
        return {"status": "needs_review", "file": str(file_path)}

    # Step 2: NER
    entities = ner_extract(text)
    entity_summary = {}
    for entity in entities:
        entity_type = entity["type"].lower()
        key = f"entities_{entity_type}"
        entity_summary.setdefault(key, []).append(entity["text"])
    print(f"  NER: {len(entities)} entities found")

    # Step 3: Classification
    classification = classify(text)
    doc_type = classification["top_label"]
    doc_score = classification["top_score"]
    print(f"  Classification: {doc_type} ({doc_score:.2f})")

    # Step 4: RAG indexing with enriched metadata
    metadata = {
        "source": str(file_path),
        "doc_type": doc_type,
        "doc_type_confidence": doc_score,
        "ocr_confidence": ocr["confidence"],
        "page_count": ocr.get("pages", 1),
        **entity_summary,
    }

    result = index_document(text, metadata)
    print(f"  Indexed: {result}")

    return {
        "status": "processed",
        "file": str(file_path),
        "doc_type": doc_type,
        "entities": len(entities),
    }


def process_batch(directory):
    """Process all documents in a directory."""
    doc_dir = Path(directory)
    extensions = {".pdf", ".png", ".jpg", ".jpeg", ".tiff"}
    files = [f for f in doc_dir.iterdir() if f.suffix.lower() in extensions]

    print(f"Found {len(files)} documents to process")

    results = []
    for file_path in files:
        result = process_document(file_path)
        results.append(result)

    # Summary
    processed = sum(1 for r in results if r["status"] == "processed")
    review = sum(1 for r in results if r["status"] == "needs_review")
    print(f"\nComplete: {processed} processed, {review} need review")

    return results


def query_documents(question, doc_type=None, org=None):
    """Query the enriched document index."""
    filters = {}
    if doc_type:
        filters["metadata.doc_type"] = doc_type
    if org:
        filters["metadata.entities_org"] = org

    resp = requests.post(
        f"{BASE}/projects/{PROJECT}/rag/query",
        json={
            "query": question,
            "database": "main_db",
            "top_k": 5,
            "filters": filters if filters else None,
        },
    )
    return resp.json()


if __name__ == "__main__":
    # Process a batch of documents
    process_batch("/path/to/documents")

    # Query examples
    print("\n--- Query: all invoices ---")
    results = query_documents(
        "consulting services total amount",
        doc_type="invoice",
    )
    for chunk in results.get("chunks", []):
        print(f"  [{chunk['score']:.2f}] {chunk['content'][:100]}...")

    print("\n--- Query: Acme Corporation documents ---")
    results = query_documents(
        "contract terms and obligations",
        org="Acme Corporation",
    )
    for chunk in results.get("chunks", []):
        print(f"  [{chunk['score']:.2f}] {chunk['content'][:100]}...")
```

---

## Variations

### Invoice Processing

Focus on extracting line items, amounts, and payment terms.

```python
def process_invoice(text, entities):
    """Extract structured invoice data."""
    amounts = [e for e in entities if e["type"] == "MONEY"]
    dates = [e for e in entities if e["type"] == "DATE"]
    orgs = [e for e in entities if e["type"] == "ORG"]

    return {
        "vendor": orgs[0]["text"] if orgs else None,
        "date": dates[0]["text"] if dates else None,
        "amounts": [a["text"] for a in amounts],
        "total": amounts[-1]["text"] if amounts else None,
    }
```

### Contract Analysis

Focus on extracting parties, terms, and key dates.

```python
def process_contract(text, entities):
    """Extract structured contract data."""
    parties = [e for e in entities if e["type"] == "ORG"]
    dates = [e for e in entities if e["type"] == "DATE"]
    people = [e for e in entities if e["type"] == "PERSON"]

    return {
        "parties": [p["text"] for p in parties],
        "signatories": [p["text"] for p in people],
        "key_dates": [d["text"] for d in dates],
    }
```

### Compliance Review

Focus on identifying regulatory references and ensuring required sections exist.

```python
def process_compliance(text, entities):
    """Check compliance document for required elements."""
    regulations = [e for e in entities if e["type"] == "REGULATION"]

    # Check for required sections
    required = ["scope", "purpose", "procedures", "responsibilities"]
    text_lower = text.lower()
    found = [s for s in required if s in text_lower]
    missing = [s for s in required if s not in text_lower]

    return {
        "regulations_cited": [r["text"] for r in regulations],
        "required_sections_found": found,
        "required_sections_missing": missing,
        "is_complete": len(missing) == 0,
    }
```
