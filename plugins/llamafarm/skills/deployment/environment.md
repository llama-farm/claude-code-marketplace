# Environment Configuration

Configure LlamaFarm with environment variables and secrets management.

## Environment Variables

### Core Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `LF_SERVER_HOST` | `0.0.0.0` | Server bind host |
| `LF_SERVER_PORT` | `8000` | Server port |
| `LF_LOG_LEVEL` | `INFO` | Logging level |
| `LF_DATA_DIR` | `.llamafarm` | Data directory |
| `LF_VERSION_REF` | `latest` | Version reference |

### Service URLs

| Variable | Default | Description |
|----------|---------|-------------|
| `OLLAMA_HOST` | `http://localhost:11434` | Ollama server URL |
| `QDRANT_URL` | `http://localhost:6333` | Qdrant server URL |
| `REDIS_URL` | `redis://localhost:6379` | Redis URL |
| `CELERY_BROKER_URL` | `redis://localhost:6379/0` | Celery broker |

### API Keys

| Variable | Description |
|----------|-------------|
| `OPENAI_API_KEY` | OpenAI API key |
| `ANTHROPIC_API_KEY` | Anthropic API key |
| `LLAMAFARM_API_KEY` | LlamaFarm authentication key |

### Authentication

| Variable | Default | Description |
|----------|---------|-------------|
| `LLAMAFARM_AUTH_ENABLED` | `false` | Enable API authentication |
| `LLAMAFARM_API_KEY` | none | Required API key |

---

## Configuration File Substitution

Use `${env:VARIABLE}` in `llamafarm.yaml`:

```yaml
runtime:
  models:
    - name: default
      provider: openai
      api_key: ${env:OPENAI_API_KEY}

mcp:
  servers:
    - name: api
      transport: http
      base_url: ${env:API_BASE_URL}
      headers:
        Authorization: Bearer ${env:API_TOKEN}
```

---

## Local Development

### .env File

```bash
# .env (never commit!)
OPENAI_API_KEY=sk-your-key-here
ANTHROPIC_API_KEY=sk-ant-your-key
OLLAMA_HOST=http://localhost:11434

# Optional
LF_LOG_LEVEL=DEBUG
LF_SERVER_PORT=8000
```

### Using direnv

```bash
# .envrc
export OPENAI_API_KEY="sk-your-key"
export OLLAMA_HOST="http://localhost:11434"
```

### Shell Export

```bash
export OPENAI_API_KEY="sk-your-key"
export OLLAMA_HOST="http://localhost:11434"
lf start
```

---

## Docker Deployment

### docker-compose.yml

```yaml
services:
  llamafarm:
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - OLLAMA_HOST=http://ollama:11434
      - CELERY_BROKER_URL=redis://redis:6379/0
```

### .env for Docker

```bash
# .env (alongside docker-compose.yml)
OPENAI_API_KEY=sk-your-key
LLAMAFARM_API_KEY=secure-random-string
```

---

## Kubernetes Secrets

### Create Secret

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: llamafarm-secrets
  namespace: llamafarm
type: Opaque
stringData:
  OPENAI_API_KEY: sk-your-key-here
  ANTHROPIC_API_KEY: sk-ant-your-key
  LLAMAFARM_API_KEY: secure-api-key
```

### Reference in Deployment

```yaml
spec:
  containers:
    - name: llamafarm
      envFrom:
        - secretRef:
            name: llamafarm-secrets
```

### External Secrets (AWS/GCP/Vault)

```yaml
# Using External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: llamafarm-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: llamafarm-secrets
  data:
    - secretKey: OPENAI_API_KEY
      remoteRef:
        key: production/llamafarm
        property: openai_api_key
```

---

## Security Best Practices

### 1. Never Commit Secrets

```bash
# .gitignore
.env
.env.local
*.pem
*-key.json
```

### 2. Use Environment-Specific Files

```bash
.env.development   # Local dev
.env.staging       # Staging
.env.production    # Production (never commit)
```

### 3. Rotate Keys Regularly

```bash
# Generate new API key
openssl rand -hex 32

# Update in secrets manager
aws secretsmanager update-secret ...
```

### 4. Limit Scope

```yaml
# Per-environment configuration
runtime:
  models:
    - name: default
      provider: ollama
      # Local model for dev - no API key needed
```

### 5. Audit Access

Log API key usage:
```yaml
environment:
  - LF_AUDIT_ENABLED=true
  - LF_AUDIT_LOG_PATH=/var/log/llamafarm/audit.log
```

---

## Cloud Provider Secrets

### AWS Secrets Manager

```bash
# Create secret
aws secretsmanager create-secret \
  --name llamafarm/production \
  --secret-string '{"OPENAI_API_KEY":"sk-xxx","LLAMAFARM_API_KEY":"xxx"}'

# Retrieve
aws secretsmanager get-secret-value \
  --secret-id llamafarm/production \
  --query SecretString --output text
```

### Google Secret Manager

```bash
# Create secret
echo -n "sk-your-key" | gcloud secrets create openai-api-key --data-file=-

# Retrieve
gcloud secrets versions access latest --secret=openai-api-key
```

### Azure Key Vault

```bash
# Create secret
az keyvault secret set \
  --vault-name llamafarm-vault \
  --name openai-api-key \
  --value "sk-your-key"

# Retrieve
az keyvault secret show \
  --vault-name llamafarm-vault \
  --name openai-api-key \
  --query value -o tsv
```

### HashiCorp Vault

```bash
# Write secret
vault kv put secret/llamafarm \
  OPENAI_API_KEY="sk-xxx" \
  LLAMAFARM_API_KEY="xxx"

# Read secret
vault kv get -field=OPENAI_API_KEY secret/llamafarm
```

---

## Debugging Environment Issues

### Check Current Environment

```bash
# Print all LF variables
env | grep -E "^(LF_|OLLAMA|OPENAI|CELERY)"

# Check specific variable
echo $OPENAI_API_KEY | head -c 10
```

### Test API Key

```bash
# Test OpenAI key
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY"

# Test Ollama connection
curl $OLLAMA_HOST/api/tags
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "Invalid API key" | Wrong/missing key | Check variable is set |
| "Connection refused" | Wrong URL | Check host/port |
| "Authentication failed" | Key expired | Rotate key |
| "Variable not found" | Not exported | Use `export VAR=value` |
