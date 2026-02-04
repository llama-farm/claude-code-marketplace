# OCR (Optical Character Recognition)

## Endpoint

```http
POST /v1/ocr
Content-Type: multipart/form-data
```

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | file | required | Image or document file to process |
| `backend` | string | `surya` | OCR backend to use |
| `language` | string | `en` | Language code or comma-separated list (e.g., `en,fr`) |
| `options.dpi` | int | `300` | Resolution for PDF rasterization |
| `options.preprocess` | boolean | `true` | Apply automatic image preprocessing |
| `options.output_format` | string | `text` | Output format: `text`, `structured`, `hocr` |
| `options.detect_tables` | boolean | `false` | Extract tables as structured data |
| `options.page_range` | string | `all` | Pages to process for PDFs (e.g., `1-5`, `1,3,7`) |

## Supported File Types

| Format | Extensions | Notes |
|--------|------------|-------|
| PNG | `.png` | Best for screenshots, diagrams |
| JPEG | `.jpg`, `.jpeg` | Good for photos, scanned docs |
| TIFF | `.tiff`, `.tif` | Multi-page support, lossless |
| PDF | `.pdf` | Rasterized per-page, respects `page_range` |
| BMP | `.bmp` | Uncompressed bitmap |
| WebP | `.webp` | Modern web format |

## Backend Comparison

| Backend | Accuracy (Printed) | Accuracy (Handwritten) | Speed (pages/sec) | Memory | GPU Support | Best For |
|---------|-------------------|----------------------|-------------------|--------|-------------|----------|
| `surya` | 98% | 85% | 2-3 | ~2 GB | Yes | General purpose, highest quality |
| `easyocr` | 95% | 75% | 3-5 | ~1.5 GB | Yes | Wide language support |
| `paddleocr` | 96% | 70% | 5-8 | ~1 GB | Yes | CJK, structured documents |
| `tesseract` | 90% | 50% | 8-12 | ~200 MB | No | Simple docs, low resource |

### Choosing a Backend

- **Best overall quality:** `surya` -- highest accuracy across document types
- **CJK documents:** `paddleocr` -- optimized for Chinese, Japanese, Korean
- **Resource constrained:** `tesseract` -- lowest memory, no GPU needed
- **Multi-language:** `easyocr` -- broadest language coverage with good accuracy

## Basic Usage

### Single Image

```bash
curl -X POST http://localhost:14345/v1/ocr \
  -F "file=@receipt.png" \
  -F "backend=surya"
```

### PDF with Page Range

```bash
curl -X POST http://localhost:14345/v1/ocr \
  -F "file=@contract.pdf" \
  -F "backend=surya" \
  -F "options.page_range=1-10" \
  -F "options.dpi=300"
```

### Multi-Language Document

```bash
curl -X POST http://localhost:14345/v1/ocr \
  -F "file=@bilingual_doc.png" \
  -F "backend=easyocr" \
  -F "language=en,fr"
```

### Structured Output with Tables

```bash
curl -X POST http://localhost:14345/v1/ocr \
  -F "file=@invoice.png" \
  -F "backend=surya" \
  -F "options.output_format=structured" \
  -F "options.detect_tables=true"
```

## Response Format

### Text Output (`output_format=text`)

```json
{
  "text": "INVOICE #12345\nDate: 2024-01-15\nTotal: $1,250.00",
  "pages": 1,
  "backend": "surya",
  "processing_time_ms": 1250
}
```

### Structured Output (`output_format=structured`)

```json
{
  "pages": [
    {
      "page_number": 1,
      "blocks": [
        {
          "text": "INVOICE #12345",
          "confidence": 0.98,
          "bounding_box": {
            "x": 50,
            "y": 30,
            "width": 200,
            "height": 25
          },
          "type": "text"
        },
        {
          "text": [["Item", "Qty", "Price"], ["Widget A", "10", "$125.00"]],
          "confidence": 0.95,
          "bounding_box": {
            "x": 50,
            "y": 200,
            "width": 500,
            "height": 150
          },
          "type": "table"
        }
      ]
    }
  ],
  "backend": "surya",
  "processing_time_ms": 2100
}
```

## Image Preprocessing Tips

Good preprocessing significantly improves OCR accuracy.

### DPI and Resolution

| Document Type | Recommended DPI | Notes |
|--------------|----------------|-------|
| Standard printed text | 300 | Default, works for most docs |
| Fine print / footnotes | 400-600 | Higher DPI for small text |
| Large headings only | 150-200 | Lower DPI is sufficient |
| Handwritten text | 300-400 | Higher helps with recognition |

### Preprocessing Options

When `options.preprocess=true` (default), LlamaFarm automatically applies:
- **Deskewing:** Corrects rotated scans (up to 15 degrees)
- **Noise reduction:** Removes speckles and artifacts
- **Contrast enhancement:** Improves text/background separation
- **Binarization:** Converts to black/white for cleaner recognition

For difficult documents, preprocess externally before upload:
- Ensure minimum 300 DPI
- Convert to grayscale if color is not needed
- Crop to content area to avoid border noise
- Straighten skewed scans manually for angles beyond 15 degrees

## Batch OCR

### Processing Multiple Files with Python

```python
import requests
from pathlib import Path

def batch_ocr(file_paths, backend="surya"):
    results = []
    for path in file_paths:
        with open(path, "rb") as f:
            response = requests.post(
                "http://localhost:14345/v1/ocr",
                files={"file": f},
                data={"backend": backend}
            )
            results.append({
                "file": str(path),
                "result": response.json()
            })
    return results

# Process all PNGs in a directory
files = list(Path("./scanned_docs").glob("*.png"))
results = batch_ocr(files)

for r in results:
    print(f"{r['file']}: {r['result']['text'][:100]}...")
```

### Directory Scanning Pipeline

```python
import requests
from pathlib import Path

SUPPORTED_EXTENSIONS = {".png", ".jpg", ".jpeg", ".tiff", ".tif", ".pdf", ".bmp", ".webp"}

def scan_directory(directory, backend="surya", recursive=True):
    dir_path = Path(directory)
    pattern = "**/*" if recursive else "*"

    files = [
        f for f in dir_path.glob(pattern)
        if f.suffix.lower() in SUPPORTED_EXTENSIONS
    ]

    print(f"Found {len(files)} files to process")

    results = {}
    for i, file_path in enumerate(files):
        print(f"Processing [{i+1}/{len(files)}]: {file_path.name}")
        with open(file_path, "rb") as f:
            response = requests.post(
                "http://localhost:14345/v1/ocr",
                files={"file": f},
                data={"backend": backend}
            )
        if response.status_code == 200:
            results[str(file_path)] = response.json()
        else:
            results[str(file_path)] = {"error": response.json()}

    return results
```

## Error Handling

| Error | HTTP Status | Cause | Resolution |
|-------|-------------|-------|------------|
| `unsupported_format` | 400 | File type not supported | Use a supported file format |
| `file_corrupted` | 400 | Cannot read or decode file | Re-export or re-scan the document |
| `backend_not_installed` | 503 | Requested backend not available | Install the backend or use a different one |
| `file_too_large` | 413 | File exceeds size limit (default 50 MB) | Reduce file size or split into parts |
| `processing_timeout` | 504 | OCR took too long | Use a faster backend or reduce page count |

**Example error response:**

```json
{
  "error": {
    "code": "backend_not_installed",
    "message": "OCR backend 'paddleocr' is not installed. Available backends: surya, tesseract",
    "details": {
      "requested_backend": "paddleocr",
      "available_backends": ["surya", "tesseract"]
    }
  }
}
```
