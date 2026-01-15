---
description: Deployment guide for LlamaFarm. Production setup with Docker, Kubernetes, and cloud platforms.
---

# LlamaFarm Deployment Skill

Guide for deploying LlamaFarm to production environments.

## When to Load

Load this skill when the user:
- Wants to deploy LlamaFarm to production
- Needs Docker or Kubernetes setup
- Asks about scaling or high availability
- Needs to configure environment variables

## Deployment Options

| Option | Best For | Complexity |
|--------|----------|------------|
| Local | Development, testing | Low |
| Docker Compose | Single server production | Medium |
| Kubernetes | Multi-node, scaling | High |
| Cloud managed | Enterprise, SaaS | Medium |

## Quick Reference

### Docker Compose

```bash
# Start all services
docker compose up -d

# Check status
docker compose ps

# View logs
docker compose logs -f
```

### Kubernetes

```bash
# Deploy
kubectl apply -f k8s/

# Check status
kubectl get pods -l app=llamafarm

# View logs
kubectl logs -l app=llamafarm-server
```

## Progressive Disclosure

For detailed deployment guides:
- `docker-compose.md` - Docker setup and configuration
- `kubernetes.md` - K8s deployment patterns
- `environment.md` - Environment variables and secrets

## Architecture Overview

### Components

```
┌─────────────────┐     ┌─────────────────┐
│   Load Balancer │     │   Ingress       │
└────────┬────────┘     └────────┬────────┘
         │                       │
┌────────▼────────┐     ┌────────▼────────┐
│  LlamaFarm API  │────▶│    Celery       │
│   (FastAPI)     │     │   (Worker)      │
└────────┬────────┘     └────────┬────────┘
         │                       │
┌────────▼────────┐     ┌────────▼────────┐
│   Vector DB     │     │   Model Server  │
│ (Chroma/Qdrant) │     │ (Ollama/vLLM)   │
└─────────────────┘     └─────────────────┘
```

### Service Ports

| Service | Port | Purpose |
|---------|------|---------|
| API Server | 8000 | REST API, MCP |
| Ollama | 11434 | Local LLM |
| Universal Runtime | 11540 | ML inference |
| Qdrant | 6333 | Vector DB |
| Redis | 6379 | Task queue |

## Production Checklist

### Security

- [ ] Configure authentication
- [ ] Use HTTPS/TLS
- [ ] Secure API keys in secrets
- [ ] Restrict network access
- [ ] Enable audit logging

### Reliability

- [ ] Set up health checks
- [ ] Configure auto-restart
- [ ] Enable persistent storage
- [ ] Set resource limits
- [ ] Configure backups

### Monitoring

- [ ] Set up log aggregation
- [ ] Configure metrics/alerts
- [ ] Monitor resource usage
- [ ] Track error rates

### Performance

- [ ] Tune memory limits
- [ ] Configure connection pools
- [ ] Enable caching
- [ ] Optimize chunk sizes
