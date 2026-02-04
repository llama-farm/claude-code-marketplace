# LlamaFarm Claude Code Plugin

Claude Code plugin for building LlamaFarm RAG and ML applications. Provides configuration generation, service management, anomaly detection, text classification, OCR, and more through slash commands and deep reference skills.

## Installation

```bash
git clone https://github.com/llama-farm/claude-code-marketplace.git
cp -r claude-code-marketplace/plugins/llamafarm ~/.claude/plugins/
```

Or use the plugin directory directly:

```bash
claude --plugin-dir ./plugins/llamafarm
```

## Skills

### Task Skills (user-invocable)

| Skill | Description |
|-------|-------------|
| `/llamafarm:start` | Start LlamaFarm services (server, RAG worker, runtime) |
| `/llamafarm:stop` | Stop services gracefully |
| `/llamafarm:status` | Check service health and component status |
| `/llamafarm:logs` | View and filter service logs |
| `/llamafarm:ml-status` | Check ML services health (anomaly, classify, buffers) |
| `/llamafarm:config` | Generate or modify configuration from natural language |
| `/llamafarm:validate` | Validate configuration against schema |
| `/llamafarm:example` | Browse and scaffold from example projects |
| `/llamafarm:anomaly` | Anomaly detection (fit, detect, stream) |
| `/llamafarm:classify` | Text classification (zero-shot, train, predict) |
| `/llamafarm:ocr` | Extract text from images and scanned documents |

### Reference Skills (loaded by agent/other skills)

| Skill | Description |
|-------|-------------|
| `config-validation` | Schema reference and validation patterns |
| `config-generation` | Use-case patterns, multi-model setup |
| `rag-pipeline` | Chunking, embeddings, retrieval strategies |
| `examples` | Example project documentation |
| `api-integration` | REST API programming, OpenAI compatibility |
| `mcp-development` | Build MCP servers, inline tools |
| `debugging` | Troubleshooting services, ingestion, retrieval |
| `deployment` | Docker Compose, Kubernetes configuration |
| `anomaly-detection` | Batch anomaly detection backends and tuning |
| `streaming-anomaly` | Real-time streaming anomaly detection |
| `text-classification` | Zero-shot and custom SetFit classification |
| `polars-buffers` | Feature engineering with sliding windows |
| `ml-nlp` | OCR, NER, and reranking capabilities |
| `ml-use-cases` | End-to-end ML patterns (IoT, fraud, document intelligence) |

## MCP Server

The plugin bundles an MCP server configuration that connects to running LlamaFarm services, providing tools for project management, chat, RAG queries, and dataset operations.

## Requirements

- LlamaFarm CLI (`lf`) installed
- LlamaFarm project directory with `llamafarm.yaml`
- For chat: Ollama or configured model provider
- For ML features: Universal Runtime (`lf runtime start`)

## Quick Start

```bash
# Generate a configuration
/llamafarm:config I want to chat with my markdown documentation

# Validate and start
/llamafarm:validate
/llamafarm:start

# Create and process a dataset
lf datasets create -s markdown_processor -b main_db docs
lf datasets upload docs ./docs/*.md
lf datasets process docs

# Chat with your documents
lf chat "What are the main topics in my documentation?"
```

## Contributing

This plugin works with [LlamaFarm](https://github.com/llama-farm/llamafarm).

- Plugin issues: File in this repository
- LlamaFarm issues: [llama-farm/llamafarm](https://github.com/llama-farm/llamafarm/issues)
- Discord: [LlamaFarm Community](https://discord.gg/RrAUXTCVNF)

## License

Apache 2.0 - Same as LlamaFarm
