# Code Analysis Pattern

For source code, repositories, technical logs, and codebase analysis.

## Model Configuration

```yaml
runtime:
  models:
    - name: default
      description: "Code analysis assistant"
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
          You are a code analysis assistant.
          Reference specific files, functions, and line numbers when discussing code.
          Explain code patterns and suggest improvements when asked.
          Be precise about language-specific syntax and conventions.
```

## Data Processing Strategy

```yaml
data_processing_strategies:
  - name: code_processor
    description: "Parser for source code and technical files"
    parsers:
      # Python
      - type: TextParser_LlamaIndex
        file_include_patterns: ["*.py"]
        priority: 100
        config:
          chunk_size: 600
          chunk_overlap: 50
          chunk_strategy: code
          preserve_code_structure: true
          detect_language: true

      # JavaScript/TypeScript
      - type: TextParser_LlamaIndex
        file_include_patterns: ["*.js", "*.ts", "*.jsx", "*.tsx"]
        priority: 100
        config:
          chunk_size: 600
          chunk_overlap: 50
          chunk_strategy: code

      # Go
      - type: TextParser_LlamaIndex
        file_include_patterns: ["*.go"]
        priority: 100
        config:
          chunk_size: 600
          chunk_overlap: 50
          chunk_strategy: code

      # Other languages
      - type: TextParser_Python
        file_include_patterns: ["*.java", "*.cpp", "*.c", "*.h", "*.rs", "*.rb"]
        priority: 90
        config:
          chunk_size: 600
          chunk_overlap: 50

      # Config files
      - type: TextParser_Python
        file_include_patterns: ["*.json", "*.yaml", "*.yml", "*.toml", "*.xml"]
        priority: 80
        config:
          chunk_size: 500
          chunk_overlap: 40

      # Logs
      - type: TextParser_Python
        file_include_patterns: ["*.log"]
        priority: 70
        config:
          chunk_size: 800
          chunk_overlap: 100

    extractors:
      - type: PatternExtractor
        file_include_patterns: ["*.py", "*.js", "*.ts", "*.go", "*.java"]
        priority: 100
        config:
          predefined_patterns: [url, email, version]
          custom_patterns:
            - name: function_def
              pattern: "def\\s+\\w+\\s*\\("
              description: "Python function definitions"
            - name: class_def
              pattern: "class\\s+\\w+[\\(\\:]"
              description: "Class definitions"
            - name: import_stmt
              pattern: "^import\\s+|^from\\s+\\w+\\s+import"
              description: "Import statements"
            - name: go_func
              pattern: "func\\s+\\w+\\s*\\("
              description: "Go function definitions"
            - name: js_function
              pattern: "function\\s+\\w+\\s*\\(|const\\s+\\w+\\s*=\\s*\\("
              description: "JavaScript functions"

      - type: LinkExtractor
        file_include_patterns: ["*"]
        priority: 90
        config:
          extract_urls: true

      - type: ContentStatisticsExtractor
        file_include_patterns: ["*"]
        priority: 80
```

## Retrieval Configuration

```yaml
retrieval_strategies:
  - name: code_search
    type: BasicSimilarityStrategy
    config:
      distance_metric: cosine
      top_k: 8
    default: true
```

## Variations

### Python Focus
Add custom patterns for decorators, type hints

### API/Web Backend
Add patterns for routes, endpoints, middleware

### Data Science
Add patterns for pandas operations, model definitions
