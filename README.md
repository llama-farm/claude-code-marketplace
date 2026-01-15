# LlamaFarm Claude Code Plugin

The comprehensive Claude Code plugin for building LlamaFarm RAG and agentic AI applications. This plugin provides intelligent configuration generation, validation, API integration, MCP development guidance, debugging, and deployment support.

## Features

- **MCP Server Integration** - Direct LlamaFarm tools in Claude Code
- **7 Slash Commands** - Service management, configuration, examples
- **8 Deep Skills** - Configuration, API, MCP, debugging, deployment
- **Context-Aware Agent** - Intelligent assistance for all LlamaFarm tasks

## Installation

```bash
# Add the marketplace
/plugin marketplace add llama-farm/claude-code-plugin

# Install the plugin
/plugin install llamafarm
```

Or clone manually:
```bash
git clone https://github.com/llama-farm/claude-code-plugin.git
cp -r claude-code-plugin/plugins/llamafarm ~/.claude/plugins/
```

## Commands

### Service Management

| Command | Description |
|---------|-------------|
| `/start` | Start LlamaFarm services |
| `/stop` | Stop services gracefully |
| `/status` | Check service health |
| `/logs` | View service logs |

### Configuration

| Command | Description |
|---------|-------------|
| `/config` | Generate config from natural language |
| `/validate` | Validate configuration against schema |
| `/example` | Browse and scaffold from examples |

### Usage Examples

```bash
# Generate a new configuration
/config I want to analyze FDA correspondence PDFs using llama3.1

# Scaffold from an example
/example fda_rag

# Start services and check status
/start
/status

# Validate your configuration
/validate llamafarm.yaml
```

## Skills

### Configuration Skills

| Skill | Purpose |
|-------|---------|
| `config-validation` | Schema reference, required fields, patterns |
| `config-generation` | Use-case patterns, multi-model setup |
| `rag-pipeline` | Chunking, embeddings, retrieval strategies |
| `examples` | Example projects and scaffolding |

### Development Skills

| Skill | Purpose |
|-------|---------|
| `api-integration` | REST API programming, OpenAI compatibility |
| `mcp-development` | Build MCP servers, inline tools, access control |

### Operations Skills

| Skill | Purpose |
|-------|---------|
| `debugging` | Service issues, ingestion, retrieval, performance |
| `deployment` | Docker Compose, Kubernetes, environment config |

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

| Use Case | Configuration | Commands |
|----------|---------------|----------|
| **PDF Analysis** | llama3.1, semantic chunking, EntityExtractor | `/example fda_rag` |
| **Documentation** | Smaller chunks, HeadingExtractor, MarkdownParser | `/example quick_rag` |
| **Large Documents** | Hierarchy extraction, larger overlap | `/example gov_rag` |
| **Code Analysis** | Code-aware chunking, PatternExtractor | `/config code analysis` |
| **Mixed Formats** | HybridUniversalStrategy, multiple parsers | `/config mixed documents` |

## Requirements

- LlamaFarm CLI (`lf`) installed
- LlamaFarm project directory with `llamafarm.yaml`
- For chat: Ollama or configured model provider
- For MCP: LlamaFarm services running (`lf start`)

## Quick Start

```bash
# 1. Generate a configuration
/config I want to chat with my markdown documentation

# 2. Validate the config
/validate

# 3. Start services
/start

# 4. Check everything is running
/status

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
├── commands/
│   ├── lf-config.md             # Config generation
│   ├── lf-validate.md           # Config validation
│   ├── lf-example.md            # Example scaffolding
│   ├── lf-start.md              # Start services
│   ├── lf-stop.md               # Stop services
│   ├── lf-status.md             # Health check
│   └── lf-logs.md               # View logs
├── skills/
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
