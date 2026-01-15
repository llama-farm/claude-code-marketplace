# Cross-Reference Validation

LlamaFarm configs have several cross-references that must be valid.

## Dataset References

Each dataset must reference existing strategy and database:

```yaml
datasets:
  - name: research
    data_processing_strategy: pdf_processor  # Must exist in rag.data_processing_strategies
    database: main_db                        # Must exist in rag.databases
```

**Validation error example:**
```
Dataset 'research' references non-existent strategy 'pdf_processor'.
Available strategies: universal_processor, markdown_processor
```

## Model → Prompt References

Models can reference prompt sets:

```yaml
runtime:
  models:
    - name: default
      prompts: [system_prompt, rag_context]  # Each must exist in prompts section

prompts:
  - name: system_prompt
    messages: [...]
  - name: rag_context
    messages: [...]
```

**Validation error example:**
```
Model 'default' references non-existent prompt set 'custom_prompt'.
Available prompt sets: system_prompt, rag_context
```

## Database Strategy References

Databases must specify valid default strategies:

```yaml
databases:
  - name: main_db
    default_embedding_strategy: default_embeddings  # Must exist in embedding_strategies
    default_retrieval_strategy: basic_search        # Must exist in retrieval_strategies
    embedding_strategies:
      - name: default_embeddings
        ...
    retrieval_strategies:
      - name: basic_search
        ...
```

## Uniqueness Constraints

### Prompt Names (case-sensitive)
```
Duplicate prompt set names found: default, default
Each prompt set must have a unique name.
```

### Dataset Names (case-insensitive)
```
Duplicate dataset names found: 'Research', 'research'
Each dataset must have a unique name (case-insensitive).
```

### Database Names
```
Duplicate database names found: main_db
Each database must have a unique name.
```

## Common Validation Errors

| Error | Cause | Fix |
|-------|-------|-----|
| "references non-existent strategy" | Typo in strategy name | Check data_processing_strategies names |
| "references non-existent database" | Typo in database name | Check rag.databases names |
| "references non-existent prompt" | Typo in prompt name | Check prompts[].name values |
| "Duplicate names found" | Same name used twice | Rename one of the duplicates |
