# Docker Compose Deployment

Deploy LlamaFarm with Docker Compose for single-server production.

## Quick Start

### 1. Create docker-compose.yml

```yaml
version: '3.8'

services:
  llamafarm:
    image: llamafarm/llamafarm:latest
    ports:
      - "14345:14345"
    volumes:
      - ./llamafarm.yaml:/app/llamafarm.yaml:ro
      - ./data:/app/data
      - llamafarm-cache:/app/.llamafarm
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - UNIVERSAL_RUNTIME_URL=http://universal-runtime:11540/v1
    depends_on:
      - universal-runtime
      - redis
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:14345/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  worker:
    image: llamafarm/llamafarm:latest
    command: celery -A llamafarm.worker worker --loglevel=info
    volumes:
      - ./llamafarm.yaml:/app/llamafarm.yaml:ro
      - ./data:/app/data
      - llamafarm-cache:/app/.llamafarm
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - UNIVERSAL_RUNTIME_URL=http://universal-runtime:11540/v1
    depends_on:
      - redis
      - universal-runtime

  universal-runtime:
    image: llamafarm/universal-runtime:latest
    ports:
      - "11540:11540"
    volumes:
      - universal-models:/root/.cache/huggingface
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
    volumes:
      - qdrant-data:/qdrant/storage
    environment:
      - QDRANT__SERVICE__GRPC_PORT=6334

volumes:
  llamafarm-cache:
  universal-models:
  redis-data:
  qdrant-data:
```

### 2. Create .env file

```bash
# .env
OPENAI_API_KEY=sk-your-key-here
LLAMAFARM_API_KEY=your-api-key
```

### 3. Start Services

```bash
# Start all services
docker compose up -d

# Check status
docker compose ps

# View logs
docker compose logs -f llamafarm
```

---

## Configuration

### Environment Variables

```yaml
services:
  llamafarm:
    environment:
      # API Keys
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}

      # Service URLs
      - UNIVERSAL_RUNTIME_URL=http://universal-runtime:11540/v1
      - QDRANT_URL=http://qdrant:6333
      - CELERY_BROKER_URL=redis://redis:6379/0

      # Server settings
      - LF_SERVER_HOST=0.0.0.0
      - LF_SERVER_PORT=8000
      - LF_LOG_LEVEL=INFO

      # Security
      - LLAMAFARM_API_KEY=${LLAMAFARM_API_KEY}
```

### Resource Limits

```yaml
services:
  llamafarm:
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: '2'
        reservations:
          memory: 2G

  worker:
    deploy:
      resources:
        limits:
          memory: 8G
          cpus: '4'
        reservations:
          memory: 4G

  universal-runtime:
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

### Persistent Storage

```yaml
volumes:
  # LlamaFarm data
  llamafarm-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/data/llamafarm

  # Universal Runtime models (can be large)
  universal-models:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/data/universal-runtime

  # Vector database
  qdrant-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/data/qdrant
```

---

## Production Setup

### With HTTPS (Traefik)

```yaml
services:
  traefik:
    image: traefik:v2.10
    command:
      - --api.insecure=true
      - --providers.docker=true
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.letsencrypt.acme.tlschallenge=true
      - --certificatesresolvers.letsencrypt.acme.email=your@email.com
      - --certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - letsencrypt:/letsencrypt

  llamafarm:
    labels:
      - traefik.enable=true
      - traefik.http.routers.llamafarm.rule=Host(`api.yourdomain.com`)
      - traefik.http.routers.llamafarm.entrypoints=websecure
      - traefik.http.routers.llamafarm.tls.certresolver=letsencrypt
```

### With Authentication

```yaml
services:
  llamafarm:
    environment:
      - LLAMAFARM_AUTH_ENABLED=true
      - LLAMAFARM_API_KEY=${LLAMAFARM_API_KEY}
```

### With Monitoring

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    volumes:
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
```

---

## Scaling

### Multiple Workers

```yaml
services:
  worker:
    deploy:
      replicas: 3
```

### Scale Command

```bash
# Scale workers
docker compose up -d --scale worker=3

# Check scaling
docker compose ps
```

---

## Maintenance

### Updates

```bash
# Pull latest images
docker compose pull

# Restart with new images
docker compose up -d

# Remove old images
docker image prune -f
```

### Backups

```bash
# Backup volumes
docker run --rm -v llamafarm-data:/data -v $(pwd):/backup \
  alpine tar czf /backup/llamafarm-backup.tar.gz /data

# Backup Qdrant
docker run --rm -v qdrant-data:/data -v $(pwd):/backup \
  alpine tar czf /backup/qdrant-backup.tar.gz /data
```

### Logs

```bash
# View all logs
docker compose logs

# Follow specific service
docker compose logs -f llamafarm

# Last 100 lines
docker compose logs --tail=100 worker
```

---

## Troubleshooting

### Service Won't Start

```bash
# Check logs
docker compose logs llamafarm

# Check health
docker compose ps

# Restart service
docker compose restart llamafarm
```

### Out of Memory

```bash
# Check memory usage
docker stats

# Increase limits in docker-compose.yml
# Then restart
docker compose up -d
```

### Network Issues

```bash
# Check network
docker network ls
docker network inspect llamafarm_default

# Recreate network
docker compose down
docker compose up -d
```
