# Parser Reference

## Parser Type Pattern

All parser types must match: `^[A-Za-z][A-Za-z0-9_]*Parser[A-Za-z0-9_]*$|^DirectoryParser$`

## Available Parser Types

### PDF Parsers
| Type | Description | Best For |
|------|-------------|----------|
| `PDFParser_LlamaIndex` | Advanced with fallback strategies | Complex PDFs, tables |
| `PDFParser_PyPDF2` | Simple, reliable | Basic text PDFs |

### Document Parsers
| Type | Description | Best For |
|------|-------------|----------|
| `DocxParser_LlamaIndex` | Advanced DOCX with table extraction | Word documents |
| `DocxParser_PythonDocx` | Basic python-docx | Simple Word docs |

### Markdown Parsers
| Type | Description | Best For |
|------|-------------|----------|
| `MarkdownParser_LlamaIndex` | Semantic chunking, heading-aware | Documentation |
| `MarkdownParser_Python` | Section-based chunking | Simple markdown |

### Data Parsers
| Type | Description | Best For |
|------|-------------|----------|
| `CSVParser_Pandas` | Advanced with statistics | Data analysis |
| `CSVParser_Python` | Simple row-based | Basic CSV |
| `ExcelParser_Pandas` | Multi-sheet support | Excel files |
| `ExcelParser_OpenPyXL` | Formula extraction | Excel with formulas |

### Text Parsers
| Type | Description | Best For |
|------|-------------|----------|
| `TextParser_LlamaIndex` | Semantic, code-aware | Code, logs |
| `TextParser_Python` | Sentence-based | Plain text |

## Parser Configuration

```yaml
parsers:
  - type: PDFParser_LlamaIndex
    file_include_patterns: ["*.pdf", "*.PDF"]
    priority: 100                    # Lower = tried first
    config:
      chunk_size: 1200
      chunk_overlap: 150
      chunk_strategy: semantic       # semantic, paragraphs, sentences
      extract_metadata: true
      extract_tables: true
```

## Priority and Fallback

Parsers are tried in priority order (lower first):

```yaml
parsers:
  - type: PDFParser_LlamaIndex
    priority: 100            # Primary
  - type: PDFParser_PyPDF2
    priority: 50             # Fallback
```
