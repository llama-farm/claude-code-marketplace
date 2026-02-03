# LlamaFarm Claude Code Plugin

The comprehensive Claude Code plugin for building LlamaFarm RAG and agentic AI applications. This plugin provides intelligent configuration generation, validation, API integration, MCP development guidance, debugging, and deployment support.

## Features

- **MCP Server Integration** - Direct LlamaFarm tools in Claude Code
- **10 Deep Skills** - Services, config, validation, generation, RAG, examples, API, MCP, debugging, deployment
- **Context-Aware Agent** - Intelligent assistance for all LlamaFarm tasks

## Installation

```bash
# Add the marketplace
/plugin marketplace add llama-farm/claude-code-marketplace

# Install the plugin
/plugin install llamafarm
```

Or clone manually:
```bash
git clone https://github.com/llama-farm/claude-code-marketplace.git
cp -r claude-code-marketplace/plugins/llamafarm ~/.claude/plugins/
```

## Skills

### User-Invocable Skills

| Skill | Description |
|-------|-------------|
| `/llamafarm:services` | Start, stop, check status, and view logs for LlamaFarm services |
| `/llamafarm:config` | Generate, validate, and scaffold configurations from examples |

### Knowledge Skills

| Skill | Purpose |
|-------|---------|
| `config-validation` | Schema reference, required fields, patterns |
| `config-generation` | Use-case patterns, multi-model setup |
| `rag-pipeline` | Chunking, embeddings, retrieval strategies |
| `examples` | Example projects and scaffolding |
| `api-integration` | REST API programming, OpenAI compatibility |
| `mcp-development` | Build MCP servers, inline tools, access control |
| `debugging` | Service issues, ingestion, retrieval, performance |
| `deployment` | Docker Compose, Kubernetes, environment config |

### Usage Examples

```bash
# Generate a new configuration
/llamafarm:config I want to analyze FDA correspondence PDFs using llama3.1

# Scaffold from an example
/llamafarm:config scaffold fda_rag

# Start services and check status
/llamafarm:services start
/llamafarm:services status

# Validate your configuration
/llamafarm:config validate llamafarm.yaml
```

## MCP Server

The plugin bundles an MCP server configuration that connects to running LlamaFarm services. When LlamaFarm is running, Claude Code can:

- List and manage projects
- Chat with AI models (with RAG)
- Query vector databases
- Manage datasets

**Tools available:**
- `list_projects` - List all LlamaFarm projects
- `create_project` - Create new project
- `chat_completions` - AI chat with RAG
- `rag_query` - Query documents
- `list_datasets` - List datasets
- `process_dataset` - Trigger ingestion

## Agent

### lf-assistant

A context-aware assistant that activates for LlamaFarm tasks:

**Triggers:**
- LlamaFarm configuration questions
- llamafarm.yaml editing
- RAG pipeline setup
- API integration questions
- MCP development
- Troubleshooting
- Deployment questions

**Capabilities:**
- Configuration generation and validation
- Multi-model setup guidance
- API endpoint reference
- MCP server development
- Debugging assistance
- Deployment planning

## Use Case Patterns

| Use Case | Configuration | Skills |
|----------|---------------|----------|
| **PDF Analysis** | llama3.1, semantic chunking, EntityExtractor | `/llamafarm:config` (example fda_rag) |
| **Documentation** | Smaller chunks, HeadingExtractor, MarkdownParser | `/llamafarm:config` (example quick_rag) |
| **Large Documents** | Hierarchy extraction, larger overlap | `/llamafarm:config` (example gov_rag) |
| **Code Analysis** | Code-aware chunking, PatternExtractor | `/llamafarm:config` (generate) |
| **Mixed Formats** | HybridUniversalStrategy, multiple parsers | `/llamafarm:config` (generate) |

## Requirements

- LlamaFarm CLI (`lf`) installed
- LlamaFarm project directory with `llamafarm.yaml`
- For chat: Ollama or configured model provider
- For MCP: LlamaFarm services running (`lf start`)

## Quick Start

```bash
# 1. Generate a configuration
/llamafarm:config I want to chat with my markdown documentation

# 2. Validate the config
/llamafarm:config validate

# 3. Start services
/llamafarm:services start

# 4. Check everything is running
/llamafarm:services status

# 5. Create and process a dataset
lf datasets create -s markdown_processor -b main_db docs
lf datasets upload docs ./docs/*.md
lf datasets process docs

# 6. Chat with your documents
lf chat "What are the main topics in my documentation?"
```

## File Structure

```
plugins/llamafarm/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── mcp-servers/
│   └── llamafarm.json           # MCP server config
├── skills/
│   ├── services/                # Service management skill (4 files)
│   ├── config/                  # Configuration skill (3 files)
│   ├── config-validation/       # Schema reference (5 files)
│   ├── config-generation/       # Config patterns (9 files)
│   ├── rag-pipeline/            # RAG guidance (5 files)
│   ├── examples/                # Example projects (4 files)
│   ├── api-integration/         # REST API (5 files)
│   ├── mcp-development/         # MCP servers (5 files)
│   ├── debugging/               # Troubleshooting (5 files)
│   └── deployment/              # Docker/K8s (4 files)
├── agents/
│   └── lf-assistant.md          # Context-aware assistant
└── README.md
```

## Contributing

This plugin works with [LlamaFarm](https://github.com/llama-farm/llamafarm).

**For issues or improvements:**
- Plugin issues: File in this repository
- LlamaFarm issues: [llama-farm/llamafarm](https://github.com/llama-farm/llamafarm/issues)
- Discord: [LlamaFarm Community](https://discord.gg/RrAUXTCVNF)

## License

Apache 2.0 - Same as LlamaFarm
