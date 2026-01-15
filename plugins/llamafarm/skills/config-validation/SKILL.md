---
description: Schema-aware guidance for validating LlamaFarm configurations. Activates for llamafarm.yaml validation, config errors, schema reference.
---

# LlamaFarm Config Validation Skill

Schema-aware guidance for validating LlamaFarm configurations.

## When to Load

Activate this skill when:
- Validating `llamafarm.yaml` files
- User encounters config errors
- Editing LlamaFarm configuration
- Running `/validate` command

## Core Schema Reference

### Required Top-Level Fields

```yaml
version: v1              # Required, must be "v1"
name: my-project         # Required, project identifier
namespace: default       # Required, organization/team grouping
runtime:                 # Required, model configuration
  models: []
```

### Provider Enum

Valid values for `runtime.models[].provider`:
- `openai` - OpenAI API or compatible endpoints
- `ollama` - Ollama local models
- `lemonade` - Lemonade runtime with NPU/GPU acceleration
- `universal` - LlamaFarm Universal Runtime

### Core Naming Patterns

| Field | Pattern | Example |
|-------|---------|---------|
| Prompt names | `^[a-z][a-z0-9_]*$` | `default`, `system_prompt` |
| Database names | `^[a-z][a-z0-9_]*$` | `main_db`, `vectors` |
| Dataset names | `^[a-zA-Z0-9_-]+$` | `research`, `fda-letters` |

### Validation Commands

```bash
# Using LlamaFarm CLI
lf projects validate --config llamafarm.yaml

# Using Python validator
cd /path/to/llamafarm/config
uv run python validate_config.py llamafarm.yaml --verbose
```

**Exit codes:** `0` = valid, `1` = invalid, `2` = error

## Progressive Disclosure

For detailed references, load these files as needed:

- **parsers.md** - Parser types, patterns, and configurations
- **extractors.md** - Extractor types and configurations
- **strategies.md** - Retrieval and embedding strategies
- **cross-references.md** - Dataset→strategy→database validation
