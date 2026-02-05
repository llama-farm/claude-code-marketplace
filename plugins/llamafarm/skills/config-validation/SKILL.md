---
description: Schema-aware guidance for validating LlamaFarm configurations. Activates for llamafarm.yaml validation, config errors, schema reference.
---

# LlamaFarm Config Validation Skill

Schema-aware guidance for validating LlamaFarm configurations.

## Command Entry Point

```
/llamafarm:validate                          # Validate ./llamafarm.yaml
/llamafarm:validate path/to/llamafarm.yaml   # Validate specific file
```

## When to Load

Activate this skill when:
- Validating `llamafarm.yaml` files
- User encounters config errors
- Editing LlamaFarm configuration
- Running `/llamafarm:validate` command

## Validation Workflow

1. **Find config file** - Locate `llamafarm.yaml` in current or specified path
2. **Run validation** - Check against LlamaFarm schema
3. **Interpret results** - Parse validation output
4. **Present findings** - Show errors with fixes

## Step 1: Find Config File

```bash
# Check current directory
ls llamafarm.yaml

# Or check specified path
ls path/to/llamafarm.yaml
```

If not found, prompt user for location.

## Step 2: Run Validation

### Validation Commands

```bash
# Using LlamaFarm CLI (preferred)
lf projects validate --config llamafarm.yaml

# Using Python validator (alternative)
cd /path/to/llamafarm/config
uv run python validate_config.py llamafarm.yaml --verbose
```

**Exit codes:** `0` = valid, `1` = invalid, `2` = error

## Step 3: Interpret Results

**Success output:**
```
Validation Results
==================

Status: VALID

Your configuration is ready to use.

Next steps:
  lf start
```

**Error output:**
```
Validation Results
==================

Status: INVALID

Errors found: 3

1. [runtime.models.0.provider] Invalid provider "anthropic"
   Valid providers: universal, ollama, openai, lemonade
   Fix: Change provider to one of the valid options

2. [rag.databases.0.name] Invalid name "Main-DB"
   Must match pattern: ^[a-z][a-z0-9_]*$
   Fix: Rename to "main_db"

3. [datasets.0.data_processing_strategy] Reference "markdown_proc" not found
   Available strategies: default_processor
   Fix: Update reference to match defined strategy name
```

## Step 4: Present Results

For each error, provide:
1. **Location** in the YAML (dotted path)
2. **What's wrong** (clear description)
3. **How to fix** (specific action)

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
- `universal` - LlamaFarm Universal Runtime
- `openai` - OpenAI API or compatible endpoints
- `ollama` - Ollama local models
- `lemonade` - Lemonade runtime with NPU/GPU acceleration

### Core Naming Patterns

| Field | Pattern | Example |
|-------|---------|---------|
| Prompt names | `^[a-z][a-z0-9_]*$` | `default`, `system_prompt` |
| Database names | `^[a-z][a-z0-9_]*$` | `main_db`, `vectors` |
| Dataset names | `^[a-zA-Z0-9_-]+$` | `research`, `fda-letters` |

## Common Validation Errors and Fixes

### Missing Required Fields

```
Error: Missing required field "version"
Fix: Add "version: v1" at the top of your config
```

```
Error: Missing required field "runtime.models"
Fix: Add at least one model configuration:
  runtime:
    models:
      - name: default
        provider: universal
        model: unsloth/Qwen3-4B-GGUF:Q4_K_M
        default: true
```

### Invalid Provider

```
Error: Invalid provider "anthropic"
Valid: universal, ollama, openai, lemonade
Fix: Change to a supported provider. For Anthropic models,
     use provider "openai" with the Anthropic-compatible endpoint.
```

### Parser Name Pattern

```
Error: Parser type "my-custom-parser" is not recognized
Valid parsers:
  - PDFParser_LlamaIndex
  - MarkdownParser_LlamaIndex
  - TextParser_LlamaIndex
  - CodeParser_LlamaIndex
  - HTMLParser_LlamaIndex
Fix: Use one of the recognized parser types
```

### Duplicate Dataset Names

```
Error: Duplicate dataset name "research"
Fix: Each dataset must have a unique name. Rename one of the
     duplicate datasets.
```

### Invalid Prompt Reference

```
Error: Prompt reference "chat_prompt" not found in prompts list
Defined prompts: default, analysis
Fix: Either add a prompt named "chat_prompt" to the prompts
     section, or update the reference to use an existing prompt.
```

### Database Name Pattern

```
Error: Database name "My-Database" does not match required pattern
Pattern: ^[a-z][a-z0-9_]*$
Fix: Use lowercase letters, numbers, and underscores only.
     Must start with a letter. Example: "my_database"
```

## Progressive Disclosure

For detailed references, load these files as needed:

- **parsers.md** - Parser types, patterns, and configurations
- **extractors.md** - Extractor types and configurations
- **strategies.md** - Retrieval and embedding strategies
- **cross-references.md** - Dataset→strategy→database validation
