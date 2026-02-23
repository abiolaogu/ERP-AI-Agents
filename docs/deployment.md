# Deployment Guide -- AI-Agents Platform
> Version: 1.0 | Last Updated: 2026-02-18 | Status: Draft
> Classification: Internal | Author: AIDD System

---

## 1. Overview

This guide covers deployment procedures for the AI-Agents Platform across three environments: local development (Docker Compose), staging/production (Kubernetes), and GPU-enabled deployments (RunPod). It includes prerequisites, step-by-step instructions, configuration management, and post-deployment verification.

---

## 2. Prerequisites

### 2.1 Software Requirements

| Tool | Version | Required For |
|------|---------|-------------|
| Docker | 24.0+ | All environments |
| Docker Compose | 2.0+ | Local development |
| kubectl | 1.27+ | Kubernetes deployments |
| Helm | 3.0+ | Kubernetes (Helm charts) |
| Python | 3.11+ | Build and scripting |
| Node.js | 18+ | Frontend build |
| Git | 2.0+ | Source code |

### 2.2 Infrastructure Requirements

| Environment | Compute | Storage | Network |
|-------------|---------|---------|---------|
| Development | 4 CPU, 8 GB RAM | 20 GB SSD | Broadband |
| Staging | 8 CPU, 16 GB RAM | 100 GB NVMe | 1 Gbps |
| Production | 16+ CPU, 32+ GB RAM | 200+ GB NVMe | 1 Gbps |

### 2.3 Required Credentials

| Credential | Purpose | Storage |
|-----------|---------|---------|
| ANTHROPIC_API_KEY | Claude 3.5 Sonnet access | Vault / .env |
| DATABASE_URL | PostgreSQL connection | Vault / .env |
| REDIS_URL | Redis connection | Vault / .env |
| JWT_SECRET | Token signing | Vault / .env |
| SECRET_KEY | Application secret | Vault / .env |
| OPA_URL | Policy engine endpoint | .env / ConfigMap |
| Container registry credentials | Image push/pull | Docker config / K8s secret |

---

## 3. Local Development Deployment (Docker Compose)

### 3.1 Quick Start

```bash
# Clone repository
git clone <repository-url> AI-Agents
cd AI-Agents

# Create environment file
cp .env.example .env
# Edit .env with your ANTHROPIC_API_KEY and other credentials

# Start all services
docker compose up -d

# Verify services
docker compose ps
```

### 3.2 Service Inventory

| Service | Port | Health Check |
|---------|------|-------------|
| orchestration-engine | 8000 | `curl http://localhost:8000/health` |
| postgres | 5432 | `pg_isready -h localhost -p 5432` |
| redis | 6379 | `redis-cli ping` |
| redpanda | 9092 | `rpk cluster health` |
| opa | 8181 | `curl http://localhost:8181/health` |
| frontend | 3000 | `curl http://localhost:3000` |

### 3.3 Starting Individual Agents

```bash
# Build a specific agent
cd generated-agents/marketing_content_agent_1_101
docker build -t marketing-content-agent:latest .

# Run with environment variables
docker run -d \
  --name marketing-content-agent \
  -p 8200:8200 \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  --network ai-agents_default \
  marketing-content-agent:latest

# Verify
curl http://localhost:8200/health
```

### 3.4 Stopping and Cleanup

```bash
# Stop services (preserve data)
docker compose down

# Stop and remove volumes (WARNING: destroys all data)
docker compose down -v

# Remove all agent containers
docker rm -f $(docker ps -a -q --filter "ancestor=*agent*")
```

---

## 4. Kubernetes Deployment

### 4.1 Cluster Preparation

```bash
# Verify cluster access
kubectl cluster-info
kubectl get nodes

# Create namespace
kubectl create namespace ai-agents

# Create secrets
kubectl create secret generic platform-secrets \
  --from-literal=ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  --from-literal=DATABASE_URL=$DATABASE_URL \
  --from-literal=REDIS_URL=$REDIS_URL \
  --from-literal=JWT_SECRET=$JWT_SECRET \
  --from-literal=SECRET_KEY=$SECRET_KEY \
  -n ai-agents
```

### 4.2 Deploy Infrastructure Services

```bash
# PostgreSQL
kubectl apply -f infrastructure/k8s/postgres/ -n ai-agents

# Redis
kubectl apply -f k8s/redis/ -n ai-agents

# Redpanda
kubectl apply -f infrastructure/k8s/redpanda/ -n ai-agents

# OPA
kubectl apply -f infrastructure/k8s/opa/ -n ai-agents

# Consul (configuration management)
kubectl apply -f infrastructure/k8s/consul/ -n ai-agents

# Verify infrastructure pods
kubectl get pods -n ai-agents
```

### 4.3 Deploy Orchestration Engine

```bash
# Build and push image
docker build -t registry.example.com/orchestration-engine:1.0.0 \
  -f services/orchestration_engine/Dockerfile .
docker push registry.example.com/orchestration-engine:1.0.0

# Apply deployment
kubectl apply -f k8s/orchestration-engine/ -n ai-agents

# Verify
kubectl get deployment orchestration-engine -n ai-agents
kubectl get pods -l app=orchestration-engine -n ai-agents
```

### 4.4 Deploy Agents

```bash
# Generate K8s manifests for all agents
python infrastructure/scripts/generate_k8s_manifests.py \
  --environment=production

# Deploy a batch of agents
kubectl apply -f infrastructure/k8s/agents/ -n ai-agents

# Or deploy a specific agent
kubectl apply -f infrastructure/k8s/agents/marketing_content_agent_1_101.yaml -n ai-agents

# Verify agent pods
kubectl get pods -l tier=agent -n ai-agents
```

### 4.5 Deploy Monitoring Stack

```bash
# Prometheus
kubectl apply -f infrastructure/k8s/monitoring/prometheus/ -n monitoring

# Grafana
kubectl apply -f infrastructure/k8s/monitoring/grafana/ -n monitoring

# Loki + Promtail
kubectl apply -f infrastructure/k8s/monitoring/loki/ -n monitoring

# AlertManager
kubectl apply -f infrastructure/k8s/monitoring/alertmanager/ -n monitoring

# Verify
kubectl get pods -n monitoring
```

### 4.6 Deploy Frontend

```bash
# Build frontend
cd web
npm run build

# Build and push Docker image
docker build -t registry.example.com/ai-agents-frontend:1.0.0 .
docker push registry.example.com/ai-agents-frontend:1.0.0

# Apply deployment
kubectl apply -f infrastructure/k8s/frontend/ -n ai-agents
```

### 4.7 Configure Ingress

```yaml
# infrastructure/k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ai-agents-ingress
  namespace: ai-agents
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - agents.your-domain.com
        - api.agents.your-domain.com
      secretName: ai-agents-tls
  rules:
    - host: agents.your-domain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
    - host: api.agents.your-domain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: orchestration-engine
                port:
                  number: 8000
```

```bash
kubectl apply -f infrastructure/k8s/ingress.yaml -n ai-agents
```

### 4.8 Configure HPA

```bash
# Orchestration engine auto-scaling
kubectl autoscale deployment orchestration-engine \
  --min=2 --max=10 --cpu-percent=70 -n ai-agents

# High-demand agent auto-scaling
kubectl autoscale deployment marketing-content-agent-1-101 \
  --min=1 --max=5 --cpu-percent=70 -n ai-agents

# Verify HPA
kubectl get hpa -n ai-agents
```

---

## 5. Helm Deployment (Alternative)

```bash
# Add Helm chart repository (if published)
helm repo add ai-agents https://charts.your-domain.com

# Install with custom values
helm install ai-agents ai-agents/ai-agents-platform \
  --namespace ai-agents \
  --create-namespace \
  -f values-production.yaml

# Upgrade
helm upgrade ai-agents ai-agents/ai-agents-platform \
  --namespace ai-agents \
  -f values-production.yaml

# Rollback
helm rollback ai-agents 1 --namespace ai-agents
```

---

## 6. Network Policies

```bash
# Apply network policies for namespace isolation
kubectl apply -f security/k8s/network-policies/ -n ai-agents
```

Key policies:
- Agents can only receive traffic from orchestration-engine
- Orchestration-engine can access PostgreSQL, Redis, Redpanda, OPA
- Monitoring namespace can scrape metrics from all pods
- No direct external access to agent pods

---

## 7. Post-Deployment Verification

### 7.1 Health Check Script

```bash
#!/bin/bash
echo "=== AI-Agents Platform Post-Deployment Verification ==="

# Orchestration Engine
echo -n "Orchestration Engine: "
curl -s http://api.agents.your-domain.com/health | jq -r .status

# PostgreSQL
echo -n "PostgreSQL: "
kubectl exec -n ai-agents deploy/orchestration-engine -- \
  python -c "import asyncio; from database import engine; print('OK')"

# Redis
echo -n "Redis: "
kubectl exec -n ai-agents deploy/orchestration-engine -- \
  python -c "import redis; r=redis.from_url('$REDIS_URL'); print(r.ping())"

# Sample Agent Health
echo -n "Sample Agent: "
curl -s http://api.agents.your-domain.com/agents/health/marketing_content_agent_1_101

# Auth Flow
echo "Testing auth flow..."
TOKEN=$(curl -s -X POST http://api.agents.your-domain.com/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test123"}' | jq -r .access_token)
echo "JWT obtained: ${TOKEN:0:20}..."

echo "=== Verification Complete ==="
```

### 7.2 Smoke Tests

```bash
# Run automated smoke tests
python scripts/run_e2e_test.py --environment production

# Expected: All critical paths pass
# - User registration and login
# - Agent listing and search
# - Single agent execution
# - Workflow creation and execution
```

---

## 8. Rollback Procedures

### 8.1 Kubernetes Rollback

```bash
# Check deployment history
kubectl rollout history deployment/orchestration-engine -n ai-agents

# Rollback to previous version
kubectl rollout undo deployment/orchestration-engine -n ai-agents

# Rollback to specific revision
kubectl rollout undo deployment/orchestration-engine --to-revision=2 -n ai-agents

# Verify rollback
kubectl rollout status deployment/orchestration-engine -n ai-agents
```

### 8.2 Database Rollback

```bash
# Restore from backup
psql -U postgres -h $DB_HOST ai_agents < backup_YYYYMMDD.sql
```

---

## 9. Monitoring Post-Deployment

| Check | Tool | Frequency |
|-------|------|-----------|
| Pod health | `kubectl get pods -n ai-agents` | Every 5 minutes (automated) |
| Error rate | Grafana dashboard | Continuous |
| Latency | Prometheus / Grafana | Continuous |
| Resource usage | `kubectl top pods -n ai-agents` | Every 15 minutes |
| Log errors | Loki / Grafana | Continuous |
| Certificate expiry | cert-manager / manual | Weekly |

---

## 10. Environment-Specific Configuration

| Setting | Development | Staging | Production |
|---------|-------------|---------|------------|
| Replicas (orchestration) | 1 | 2 | 3+ |
| Replicas (agents) | 1 | 1 | 1-5 (HPA) |
| Database | Local PostgreSQL | Managed DBaaS | Managed DBaaS (HA) |
| Redis | Local | Managed | Managed (cluster) |
| CORS origins | `*` | Staging URL | Production URLs only |
| TLS | Self-signed / none | Let's Encrypt | Production CA |
| Log level | DEBUG | INFO | INFO |
| Monitoring | Optional | Full stack | Full stack + alerting |
| Vault | Local dev mode | Standard | HA cluster |

---

## 11. Troubleshooting Deployment Issues

| Issue | Diagnosis | Resolution |
|-------|-----------|------------|
| Pod CrashLoopBackOff | `kubectl logs <pod> -n ai-agents` | Fix application error, check env vars |
| ImagePullBackOff | `kubectl describe pod <pod>` | Verify image name/tag, registry credentials |
| Service unreachable | `kubectl get svc -n ai-agents` | Verify service selector matches pod labels |
| Database connection refused | Check DATABASE_URL, pg_isready | Verify PostgreSQL pod is running, check network policy |
| Redis connection timeout | Check REDIS_URL | Verify Redis pod is running, check network policy |
| OPA policy errors | `curl http://opa:8181/v1/policies` | Reload policies, check Rego syntax |
| High memory usage | `kubectl top pods` | Increase resource limits, check for memory leaks |
| Disk pressure | `kubectl describe node` | Clean up old images, expand PersistentVolumes |
