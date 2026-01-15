# LlamaFarm CLI Reference

## Project Initialization

```bash
# Create new project
lf init my-project
cd my-project

# Initialize in current directory
lf init .
```

## Service Management

```bash
# Start all services
lf start

# Check service status
lf services status

# View logs
lf services logs
lf services logs --service server
lf services logs --service rag

# Stop services
lf services stop
```

## Dataset Management

```bash
# Create dataset (requires strategy and database in config)
lf datasets create -s <strategy-name> -b <database-name> <dataset-name>

# List datasets
lf datasets list

# Upload files to dataset
lf datasets upload <dataset-name> ./path/to/files/*
lf datasets upload <dataset-name> ./docs/*.pdf ./reports/*.pdf

# Process dataset (embed into vector store)
lf datasets process <dataset-name>

# Delete dataset
lf datasets delete <dataset-name>
```

## RAG Operations

```bash
# Query RAG directly
lf rag query --database <db-name> --top-k 5 "Your question"

# With metadata and scores
lf rag query --database <db-name> --top-k 5 --include-metadata --include-score "query"

# Check RAG stats
lf rag stats
```

## Chat

```bash
# Chat with RAG (uses default database)
lf chat "Your question here"

# Specify database
lf chat --database <db-name> "Your question"

# Specify model
lf chat --model powerful "Complex question"

# Chat without RAG
lf chat --no-rag "Question without context"

# Specify retrieval strategy
lf chat --retrieval-strategy filtered "query"

# Control top-k
lf chat --rag-top-k 10 "query"
```

## Model Management

```bash
# List configured models
lf models list

# Pull model (Ollama)
lf models pull llama3.1:8b
```

## Configuration

```bash
# Validate configuration
lf projects validate --config llamafarm.yaml

# List projects
lf projects list
```

## Common Workflows

### New Project Setup
```bash
lf init my-project
cd my-project
# Edit llamafarm.yaml
lf start
lf datasets create -s pdf_processor -b main_db documents
lf datasets upload documents ./files/*.pdf
lf datasets process documents
lf chat "What is in these documents?"
```

### Add More Files
```bash
lf datasets upload documents ./more-files/*
lf datasets process documents
```

### Query Different Database
```bash
lf chat --database archive_db "Historical question"
```

### Debug RAG Results
```bash
lf rag query --database main_db --top-k 10 --include-score "test query"
```

## Environment Variables

```bash
# OpenAI API key
export OPENAI_API_KEY=sk-...

# Custom server URL
export LLAMAFARM_SERVER_URL=http://localhost:8080

# Verbose logging
export LLAMAFARM_LOG_LEVEL=debug
```

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Error (invalid input, failed operation) |
| 2 | Service not running |
