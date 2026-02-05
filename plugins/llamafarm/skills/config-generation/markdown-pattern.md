# Markdown/Text Notes Pattern

For documentation, README files, wiki pages, engineering notes, and text files.

## Model Configuration

```yaml
runtime:
  models:
    - name: default
      description: "Documentation assistant"
      provider: universal
      model: unsloth/Qwen3-4B-GGUF:Q4_K_M
      base_url: http://127.0.0.1:11540/v1
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
          You are a concise technical documentation assistant.
          Cite file names and section headings when referencing information.
          Provide accurate, focused answers.
```

## Data Processing Strategy

```yaml
data_processing_strategies:
  - name: markdown_processor
    description: "Processor for markdown and text documentation"
    parsers:
      - type: MarkdownParser_LlamaIndex
        file_include_patterns: ["*.md", "*.markdown", "*.mdown", "README*"]
        priority: 100
        config:
          chunk_size: 400
          chunk_overlap: 40
          chunk_strategy: headings
          extract_metadata: true
      - type: TextParser_LlamaIndex
        file_include_patterns: ["*.txt", "*.text"]
        priority: 90
        config:
          chunk_size: 400
          chunk_overlap: 40
          chunk_strategy: sentences
    extractors:
      - type: HeadingExtractor
        file_include_patterns: ["*.md", "*.markdown", "README*"]
        priority: 100
        config:
          max_level: 6
          include_hierarchy: true
          extract_outline: true
      - type: LinkExtractor
        file_include_patterns: ["*.md", "*.markdown", "*.html"]
        priority: 90
        config:
          extract_urls: true
          extract_emails: true
      - type: ContentStatisticsExtractor
        file_include_patterns: ["*"]
        priority: 80
        config:
          include_readability: true
```

## Retrieval Configuration

```yaml
retrieval_strategies:
  - name: doc_search
    type: BasicSimilarityStrategy
    config:
      distance_metric: cosine
      top_k: 5
    default: true
```

## Variations

### API Documentation
- Add KeywordExtractor for technical terms
- Consider smaller chunks (300-400)

### Wiki/Knowledge Base
- Increase top_k to 8-10
- Add EntityExtractor for people/orgs mentioned

### Engineering Notes
- Keep chunks small (400)
- Focus on ContentStatisticsExtractor
