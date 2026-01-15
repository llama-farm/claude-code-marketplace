# Extractor Reference

## Available Extractor Types

| Type | Purpose | Common Config |
|------|---------|---------------|
| `ContentStatisticsExtractor` | Word count, readability | `include_readability: true` |
| `EntityExtractor` | Named entities | `entity_types: [ORG, DATE, PERSON]` |
| `KeywordExtractor` | Keyword extraction | `algorithm: yake, max_keywords: 10` |
| `HeadingExtractor` | Document structure | `max_level: 6, include_hierarchy: true` |
| `LinkExtractor` | URLs, emails | `extract_urls: true` |
| `TableExtractor` | Table data from PDFs | `output_format: dict` |
| `DateTimeExtractor` | Date/time parsing | `fuzzy_parsing: true` |
| `PatternExtractor` | Custom regex patterns | `predefined_patterns: [email, url]` |
| `SummaryExtractor` | Text summarization | `summary_sentences: 3` |
| `RAKEExtractor` | RAKE keyword extraction | `max_phrases: 10` |
| `TFIDFExtractor` | TF-IDF keywords | `max_features: 100` |
| `YAKEExtractor` | YAKE keyword extraction | `language: en` |
| `PathExtractor` | File path extraction | `include_directories: true` |

## Extractor Configuration

```yaml
extractors:
  - type: EntityExtractor
    file_include_patterns: ["*"]     # Apply to all files
    priority: 100
    config:
      entity_types:
        - ORG
        - DATE
        - PERSON
        - PRODUCT
        - GPE        # Geo-political entity
        - FAC        # Facility
        - LOC        # Location
        - MONEY
        - PERCENT
      use_fallback: true
      min_entity_length: 2
```

## Entity Types for EntityExtractor

| Type | Description | Example |
|------|-------------|---------|
| `ORG` | Organizations | "FDA", "Microsoft" |
| `PERSON` | People | "John Smith" |
| `DATE` | Dates | "January 2024" |
| `PRODUCT` | Products | "iPhone 15" |
| `GPE` | Countries, cities, states | "California" |
| `FAC` | Facilities | "LAX Airport" |
| `LOC` | Non-GPE locations | "Mount Everest" |
| `MONEY` | Monetary values | "$1,000,000" |
| `PERCENT` | Percentages | "15%" |
| `LAW` | Legal documents | "Section 8" |

## PatternExtractor Custom Patterns

```yaml
- type: PatternExtractor
  config:
    predefined_patterns:
      - email
      - url
      - ip_address
      - version
    custom_patterns:
      - name: function_def
        pattern: "def\\s+\\w+\\s*\\("
        description: "Python function definitions"
      - name: class_def
        pattern: "class\\s+\\w+[\\(\\:]"
        description: "Class definitions"
```
