# Kubernetes Deployment

Deploy LlamaFarm on Kubernetes for scalability and high availability.

## Quick Start

### 1. Create Namespace

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: llamafarm
```

### 2. Create ConfigMap

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: llamafarm-config
  namespace: llamafarm
data:
  llamafarm.yaml: |
    version: v1
    name: production
    namespace: default

    runtime:
      default_model: default
      models:
        - name: default
          provider: universal
          model: unsloth/Qwen3-4B-GGUF:Q4_K_M
          base_url: http://universal-runtime:11540/v1

    rag:
      default_database: main_db
      databases:
        - name: main_db
          type: QdrantStore
          config:
            url: http://qdrant:6333
```

### 3. Create Secrets

```yaml
# secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: llamafarm-secrets
  namespace: llamafarm
type: Opaque
stringData:
  OPENAI_API_KEY: sk-your-key-here
  LLAMAFARM_API_KEY: your-api-key
```

### 4. Deploy Server

```yaml
# server-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llamafarm-server
  namespace: llamafarm
spec:
  replicas: 2
  selector:
    matchLabels:
      app: llamafarm
      component: server
  template:
    metadata:
      labels:
        app: llamafarm
        component: server
    spec:
      containers:
        - name: server
          image: llamafarm/llamafarm:latest
          ports:
            - containerPort: 8000
          envFrom:
            - secretRef:
                name: llamafarm-secrets
          env:
            - name: UNIVERSAL_RUNTIME_URL
              value: "http://universal-runtime:11540/v1"
            - name: CELERY_BROKER_URL
              value: "redis://redis:6379/0"
            - name: QDRANT_URL
              value: "http://qdrant:6333"
          volumeMounts:
            - name: config
              mountPath: /app/llamafarm.yaml
              subPath: llamafarm.yaml
            - name: data
              mountPath: /app/data
          resources:
            limits:
              memory: "4Gi"
              cpu: "2"
            requests:
              memory: "2Gi"
              cpu: "1"
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: config
          configMap:
            name: llamafarm-config
        - name: data
          persistentVolumeClaim:
            claimName: llamafarm-data
```

### 5. Create Service

```yaml
# server-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: llamafarm
  namespace: llamafarm
spec:
  selector:
    app: llamafarm
    component: server
  ports:
    - port: 8000
      targetPort: 8000
  type: ClusterIP
```

---

## Full Stack Deployment

### Worker Deployment

```yaml
# worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llamafarm-worker
  namespace: llamafarm
spec:
  replicas: 3
  selector:
    matchLabels:
      app: llamafarm
      component: worker
  template:
    metadata:
      labels:
        app: llamafarm
        component: worker
    spec:
      containers:
        - name: worker
          image: llamafarm/llamafarm:latest
          command: ["celery", "-A", "llamafarm.worker", "worker", "-l", "info"]
          envFrom:
            - secretRef:
                name: llamafarm-secrets
          env:
            - name: CELERY_BROKER_URL
              value: "redis://redis:6379/0"
            - name: UNIVERSAL_RUNTIME_URL
              value: "http://universal-runtime:11540/v1"
          volumeMounts:
            - name: config
              mountPath: /app/llamafarm.yaml
              subPath: llamafarm.yaml
            - name: data
              mountPath: /app/data
          resources:
            limits:
              memory: "8Gi"
              cpu: "4"
            requests:
              memory: "4Gi"
              cpu: "2"
      volumes:
        - name: config
          configMap:
            name: llamafarm-config
        - name: data
          persistentVolumeClaim:
            claimName: llamafarm-data
```

### Redis Deployment

```yaml
# redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: llamafarm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
          resources:
            limits:
              memory: "1Gi"
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: llamafarm
spec:
  selector:
    app: redis
  ports:
    - port: 6379
```

### Qdrant Deployment

```yaml
# qdrant.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: qdrant
  namespace: llamafarm
spec:
  serviceName: qdrant
  replicas: 1
  selector:
    matchLabels:
      app: qdrant
  template:
    metadata:
      labels:
        app: qdrant
    spec:
      containers:
        - name: qdrant
          image: qdrant/qdrant:latest
          ports:
            - containerPort: 6333
          volumeMounts:
            - name: storage
              mountPath: /qdrant/storage
          resources:
            limits:
              memory: "4Gi"
  volumeClaimTemplates:
    - metadata:
        name: storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 50Gi
---
apiVersion: v1
kind: Service
metadata:
  name: qdrant
  namespace: llamafarm
spec:
  selector:
    app: qdrant
  ports:
    - port: 6333
```

---

## Ingress

### NGINX Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: llamafarm
  namespace: llamafarm
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - api.yourdomain.com
      secretName: llamafarm-tls
  rules:
    - host: api.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: llamafarm
                port:
                  number: 8000
```

---

## Horizontal Pod Autoscaler

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: llamafarm-server
  namespace: llamafarm
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: llamafarm-server
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: llamafarm-worker
  namespace: llamafarm
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: llamafarm-worker
  minReplicas: 1
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 80
```

---

## Persistent Volumes

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: llamafarm-data
  namespace: llamafarm
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  storageClassName: standard
```

---

## Deploy Commands

```bash
# Apply all manifests
kubectl apply -f namespace.yaml
kubectl apply -f secrets.yaml
kubectl apply -f configmap.yaml
kubectl apply -f pvc.yaml
kubectl apply -f redis.yaml
kubectl apply -f qdrant.yaml
kubectl apply -f server-deployment.yaml
kubectl apply -f worker-deployment.yaml
kubectl apply -f server-service.yaml
kubectl apply -f ingress.yaml

# Check status
kubectl get all -n llamafarm

# View logs
kubectl logs -f deployment/llamafarm-server -n llamafarm

# Scale workers
kubectl scale deployment llamafarm-worker --replicas=5 -n llamafarm
```

---

## Troubleshooting

```bash
# Pod status
kubectl get pods -n llamafarm

# Describe problem pod
kubectl describe pod <pod-name> -n llamafarm

# Pod logs
kubectl logs <pod-name> -n llamafarm

# Execute into pod
kubectl exec -it <pod-name> -n llamafarm -- /bin/bash
```
