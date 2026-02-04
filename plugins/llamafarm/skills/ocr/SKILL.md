---
description: Extract text from images and scanned documents using OCR
disable-model-invocation: true
allowed-tools:
  - Bash
argument-hint: "[file_path] [--backend <name>] [--lang <codes>]"
---

# /llamafarm:ocr - Optical Character Recognition

Extract text from images, scanned PDFs, and documents using configurable OCR backends.

## Usage

```
/llamafarm:ocr                      # Extract text from a file (prompts for path)
/llamafarm:ocr <file_path>          # Extract text from specified file
/llamafarm:ocr --backend surya      # Use a specific OCR backend
/llamafarm:ocr --lang en,de         # Specify languages for extraction
```

## What This Command Does

1. **Locates the file** — verifies the file exists and is a supported format
2. **Selects a backend** — recommends the best OCR engine for the file type
3. **Extracts text** — runs OCR and returns structured text output
4. **Presents results** — extracted text with confidence and layout info
5. **Suggests next steps** — downstream tasks like classification or indexing

## Implementation

When the user runs `/llamafarm:ocr`, follow these steps:

### Step 1: Gather Input

Ask the user:
1. What file do you want to extract text from? (file path)
2. What language(s) is the document in? (default: English)
3. Do you have a backend preference? (default: surya)

### Step 2: Verify File Exists

```bash
ls -la "<file_path>"
file "<file_path>"
```

Supported formats: `.png`, `.jpg`, `.jpeg`, `.tiff`, `.bmp`, `.pdf` (scanned), `.webp`

### Step 3: Select Backend

| Backend | Best For | Languages | Speed |
|---------|----------|-----------|-------|
| `surya` | General documents, high accuracy | 90+ languages | Medium |
| `easyocr` | Scene text, handwriting | 80+ languages | Medium |
| `paddleocr` | CJK text, complex layouts | 80+ languages | Fast |
| `tesseract` | Simple documents, legacy | 100+ languages | Fast |

Default recommendation: `surya` for most documents.

### Step 4: Execute OCR

```bash
curl -X POST http://localhost:14345/v1/ocr \
  -H "Content-Type: application/json" \
  -d '{
    "file_path": "<absolute_file_path>",
    "backend": "surya",
    "languages": ["en"]
  }'
```

For multi-page PDFs:

```bash
curl -X POST http://localhost:14345/v1/ocr \
  -H "Content-Type: application/json" \
  -d '{
    "file_path": "<absolute_file_path>",
    "backend": "surya",
    "languages": ["en"],
    "pages": "all"
  }'
```

### Step 5: Present Results

```
OCR Extraction Results
======================

File: invoice_scan.pdf
Backend: surya
Pages: 3
Language: en

Extracted Text (Page 1):
-----------------------
ACME Corporation
Invoice #12345
Date: 2024-01-15
...

Confidence: 94.2%
Characters extracted: 2,847
Processing time: 1.3s
```

Show:
- Extracted text per page
- Overall confidence score
- Character count and processing time
- Any warnings (low confidence regions, skipped pages)

### Step 6: Suggest Next Steps

Based on the extracted content, suggest:

```
Next Steps
==========

1. Classify this document:
   /llamafarm:classify
   Use the extracted text with labels like "invoice", "receipt", "contract"

2. Extract entities (names, dates, amounts):
   Use NER on the extracted text via the ML runtime

3. Index in RAG for search:
   Upload the extracted text to a LlamaFarm dataset for retrieval

4. Run anomaly detection on extracted values:
   /llamafarm:anomaly detect
```

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| `404 Not Found` | File path does not exist | Verify the file path is correct and absolute |
| `422 Unprocessable Entity` | Unsupported file format | Check supported formats listed above |
| `503 Service Unavailable` | ML runtime not running | Run `/llamafarm:start --runtime` |
| `400 Bad Request` | Invalid backend name | Use one of: surya, easyocr, paddleocr, tesseract |
| `500 Internal Server Error` | Backend processing failure | Try a different backend or check file integrity |

## Skills to Load

- `ml-nlp` - NLP pipeline reference for post-OCR processing

## Related Commands

- `/llamafarm:classify` - Classify extracted text
- `/llamafarm:ml-status` - Check ML service health
