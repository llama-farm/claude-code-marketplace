---
description: Validate llamafarm.yaml configuration against schema
---

# /llamafarm:validate - LlamaFarm Configuration Validation

Validate a LlamaFarm configuration file against the schema and custom validators.

## Usage

```
/llamafarm:validate                      # Validates ./llamafarm.yaml
/llamafarm:validate path/to/config.yaml  # Validates specific file
```

## What This Command Does

1. **Locates the config file** - Uses provided path or finds `llamafarm.yaml` in current directory
2. **Validates against JSON Schema** - Checks types, required fields, enum values, patterns
3. **Runs custom validators** - Checks unique names, cross-references, naming patterns
4. **Reports errors with actionable fixes** - Provides specific guidance on how to fix issues

## Implementation

When the user runs `/llamafarm:validate`, follow these steps:

### Step 1: Find the Config File

```bash
# Check if file exists
if [[ -f "$CONFIG_PATH" ]]; then
  echo "Validating: $CONFIG_PATH"
else
  echo "Error: Config file not found at $CONFIG_PATH"
  exit 1
fi
```

### Step 2: Run Validation

Try the `lf` CLI first (if installed), then fall back to Python validator:

```bash
# Option 1: Use lf CLI (preferred if installed)
lf projects validate --config "$CONFIG_PATH"

# Option 2: Fall back to Python validator
# Run from LlamaFarm config directory or with uv
cd /path/to/llamafarm/config && uv run python validate_config.py "$CONFIG_PATH" --verbose
```

### Step 3: Interpret Results

**Exit codes:**
- `0` = Valid configuration
- `1` = Invalid configuration (schema or custom validation error)
- `2` = File not found or system error

### Step 4: Present Results

**For valid configs:**
```
✓ Valid: llamafarm.yaml
  Version: v1
  Name: my-project
  Namespace: default
  Models: 2 configured
  Datasets: 1 defined
  RAG: 1 databases, 1 strategies
```

**For invalid configs:**
```
✗ Invalid: llamafarm.yaml
  Schema error: runtime.models[0].provider must be one of: openai, ollama, lemonade, universal

  Fix: Change provider from "gpt" to "openai"
```

## Common Validation Errors and Fixes

### Missing Required Fields
```
Error: 'runtime' is a required property
Fix: Add a runtime section with at least one model configuration
```

### Invalid Provider
```
Error: 'gpt' is not a valid provider
Fix: Use one of: openai, ollama, lemonade, universal
```

### Invalid Parser Name Pattern
```
Error: Parser type 'MyParser' does not match pattern
Fix: Parser names must match: ^[A-Za-z][A-Za-z0-9_]*Parser[A-Za-z0-9_]*$
     Example: PDFParser_LlamaIndex, TextParser_Python
```

### Duplicate Dataset Names
```
Error: Duplicate dataset names found: 'research'
Fix: Each dataset must have a unique name (case-insensitive)
```

### Invalid Prompt Reference
```
Error: Model 'default' references non-existent prompt set 'custom_prompt'
Fix: Define a prompt with name 'custom_prompt' in the prompts section,
     or use one of: default, system
```

### Invalid Database Name Pattern
```
Error: Database name 'My-Database' contains invalid characters
Fix: Database names must match: ^[a-z][a-z0-9_]*$
     Use lowercase letters, numbers, and underscores only
```

## Skills to Load

- **config-validation**: For detailed schema reference and validation patterns
