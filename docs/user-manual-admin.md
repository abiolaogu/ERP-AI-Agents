# Administrator User Manual -- AI-Agents Platform
> Version: 1.0 | Last Updated: 2026-02-18 | Status: Draft
> Classification: Internal | Author: AIDD System

---

## 1. Introduction

This manual provides system administrators with comprehensive guidance for managing the AI-Agents Platform. It covers user management, agent lifecycle administration, infrastructure operations, security policy management, monitoring, and troubleshooting.

---

## 2. Prerequisites

Before administering the platform, ensure you have:
- **Admin account** with elevated privileges on the AI-Agents Platform
- **kubectl** access to the Kubernetes cluster (production)
- **Docker** and Docker Compose installed (development/staging)
- **SSH access** to infrastructure nodes (if applicable)
- Access to **Grafana**, **Prometheus**, and **Vault** dashboards
- Familiarity with Linux command line, Docker, and Kubernetes basics

---

## 3. Platform Startup and Shutdown

### 3.1 Local Development Stack

**Start all services**:
```bash
cd AI-Agents/
docker compose up -d
```

**Verify services are running**:
```bash
docker compose ps
```

Expected services: orchestration-engine, postgres, redis, redpanda, opa, frontend

**Stop all services**:
```bash
docker compose down
```

**Stop and remove volumes** (WARNING: destroys data):
```bash
docker compose down -v
```

### 3.2 Production (Kubernetes)

**Check cluster status**:
```bash
kubectl get nodes
kubectl get pods -n ai-agents
```

**Deploy/update platform**:
```bash
kubectl apply -f infrastructure/k8s/ -n ai-agents
```

**Scale orchestration engine**:
```bash
kubectl scale deployment orchestration-engine --replicas=3 -n ai-agents
```

---

## 4. User Management

### 4.1 Viewing Users
Currently, user management is performed via direct database queries:

```sql
-- List all users
SELECT id, username, created_at FROM users ORDER BY created_at DESC;

-- Find a specific user
SELECT * FROM users WHERE username = 'john.doe';
```

### 4.2 Creating Admin Users
Admin users are created through the standard registration flow, then elevated:

1. Register via `POST /auth/register`
2. Update role in OPA roles data:

```json
// policies/roles.json
{
  "users": {
    "john.doe": {
      "roles": ["admin", "developer"]
    }
  }
}
```

3. Reload OPA policies:
```bash
curl -X PUT http://opa:8181/v1/data/roles --data-binary @policies/roles.json
```

### 4.3 Deactivating Users
To immediately revoke a user's access:

1. Blacklist their current JWT token in Redis:
```bash
redis-cli SET "blacklist:<token_jti>" "revoked" EX 3600
```

2. Remove user from OPA roles data
3. Optionally disable the account in PostgreSQL

### 4.4 Password Resets
Currently handled via direct database update (admin-initiated):

```python
from passlib.hash import bcrypt
new_hash = bcrypt.hash("new_password", rounds=12)
# UPDATE users SET password_hash = '<new_hash>' WHERE username = '<username>';
```

---

## 5. Agent Lifecycle Management

### 5.1 Viewing Agent Status

**List all loaded agents** (via API):
```bash
curl -H "Authorization: Bearer <admin_token>" \
  http://localhost:8000/agents/list
```

**Check individual agent health**:
```bash
curl http://localhost:8200/health
```

**View agent metrics**:
```bash
curl http://localhost:8200/metrics
```

### 5.2 Deploying a New Agent

1. Generate the agent from its definition:
```bash
python tools/generators/agent_generator_v2.py \
  --definition agents/definitions/marketing/content_agent.yaml
```

2. Build the Docker image:
```bash
cd generated-agents/marketing_content_agent_1_101/
docker build -t marketing_content_agent_1_101:latest .
```

3. For Kubernetes deployment, generate and apply manifests:
```bash
python infrastructure/scripts/generate_k8s_manifests.py \
  --agent marketing_content_agent_1_101 \
  --environment production
kubectl apply -f infrastructure/k8s/agents/marketing_content_agent_1_101.yaml
```

4. Verify the agent is healthy:
```bash
kubectl get pods -l app=marketing-content-agent-1-101 -n ai-agents
```

### 5.3 Scaling an Agent

```bash
# Kubernetes HPA (auto-scaling)
kubectl autoscale deployment marketing-content-agent-1-101 \
  --min=1 --max=5 --cpu-percent=70 -n ai-agents

# Manual scaling
kubectl scale deployment marketing-content-agent-1-101 \
  --replicas=3 -n ai-agents
```

### 5.4 Retiring an Agent

1. Remove from agent definitions
2. Scale deployment to 0: `kubectl scale deployment <agent> --replicas=0`
3. Remove Kubernetes resources: `kubectl delete -f <agent-manifest>.yaml`
4. Update OPA policies to deny access
5. Archive agent code and documentation

---

## 6. Security Administration

### 6.1 OPA Policy Management

**View current policies**:
```bash
curl http://opa:8181/v1/policies
```

**Update a policy**:
```bash
curl -X PUT http://opa:8181/v1/policies/agents \
  --data-binary @policies/agent_access.rego
```

**Test a policy**:
```bash
curl -X POST http://opa:8181/v1/data/agents/allow \
  -d '{"input": {"user": "john.doe", "agent": "marketing_content_agent_1_101", "action": "execute"}}'
```

### 6.2 Vault Secrets Management

**List secrets engines**:
```bash
vault secrets list
```

**Store an API key**:
```bash
vault kv put secret/agents/anthropic api_key="sk-ant-..."
```

**Rotate a secret**:
```bash
vault kv put secret/agents/anthropic api_key="sk-ant-new-key-..."
```

**Audit secret access**:
```bash
vault audit list
vault audit enable file file_path=/var/log/vault_audit.log
```

### 6.3 Network Policy Review

Review Kubernetes Network Policies:
```bash
kubectl get networkpolicies -n ai-agents
kubectl describe networkpolicy <policy-name> -n ai-agents
```

### 6.4 Security Scanning

Run the security scan suite:
```bash
./scripts/run_security_scans.sh
```

This runs:
- Dependency vulnerability scanning (pip-audit / safety)
- Docker image scanning (trivy)
- Kubernetes manifest scanning (kubesec)
- OWASP dependency check

---

## 7. Monitoring and Alerting

### 7.1 Grafana Dashboards

Access Grafana at `http://<grafana-host>:3000` (default credentials: admin/admin).

**Key dashboards**:
- **Platform Overview**: Total requests, error rate, active agents, workflow status
- **Agent Performance**: Per-agent latency, throughput, and error rates
- **Infrastructure**: CPU, memory, disk, network for all pods
- **Workflow Analytics**: Workflow success/failure rates and duration

### 7.2 Prometheus Queries

Useful PromQL queries for administration:

```promql
# Total agent requests in last hour
sum(increase(agent_requests_total[1h]))

# Agent error rate
sum(rate(agent_requests_total{status="error"}[5m])) /
sum(rate(agent_requests_total[5m]))

# P95 agent latency
histogram_quantile(0.95, rate(agent_processing_seconds_bucket[5m]))

# Active workflows
count(workflow_status == "running")
```

### 7.3 Alert Management

**View active alerts**:
```bash
curl http://alertmanager:9093/api/v2/alerts
```

**Silence an alert** (during maintenance):
```bash
curl -X POST http://alertmanager:9093/api/v2/silences \
  -d '{
    "matchers": [{"name": "alertname", "value": "AgentErrorRateHigh"}],
    "startsAt": "2026-02-18T10:00:00Z",
    "endsAt": "2026-02-18T12:00:00Z",
    "comment": "Planned maintenance"
  }'
```

### 7.4 Log Investigation

**Query recent errors via Loki/Grafana**:
```
{app="orchestration-engine"} |= "error" | json
```

**Query agent-specific logs**:
```
{app="marketing-content-agent-1-101"} | json | level="ERROR"
```

---

## 8. Configuration Management

### 8.1 Consul KV Store

**View all configuration**:
```bash
consul kv get -recurse ai-agents/
```

**Update a configuration value**:
```bash
consul kv put ai-agents/config/max_concurrent_workflows 50
```

**Watch for configuration changes**:
```bash
consul watch -type=keyprefix -prefix=ai-agents/config/ <handler-script>
```

### 8.2 Environment Variables

Key environment variables (set in `.env` or Kubernetes ConfigMaps):

| Variable | Purpose | Default |
|----------|---------|---------|
| `ANTHROPIC_API_KEY` | Claude API access | (required) |
| `DATABASE_URL` | PostgreSQL connection | `postgresql://...` |
| `REDIS_URL` | Redis connection | `redis://localhost:6379/0` |
| `OPA_URL` | OPA endpoint | `http://opa:8181` |
| `SECRET_KEY` | App-level secret | (required) |
| `JWT_SECRET` | JWT signing secret | (required) |
| `JWT_EXPIRY_MINUTES` | Token expiry | `60` |

---

## 9. Backup and Recovery

### 9.1 Database Backup

```bash
# Full PostgreSQL dump
pg_dump -U postgres ai_agents > backup_$(date +%Y%m%d).sql

# Restore from backup
psql -U postgres ai_agents < backup_20260218.sql
```

### 9.2 Redpanda Topic Backup

```bash
# Export topic data
rpk topic consume agent.executed --offset start --num 1000 > events_backup.json
```

### 9.3 Vault Backup

```bash
# Snapshot Vault data
vault operator raft snapshot save vault_snapshot_$(date +%Y%m%d).snap
```

---

## 10. Troubleshooting Quick Reference

| Symptom | Check | Resolution |
|---------|-------|------------|
| Agent returns 503 | `kubectl get pods -l app=<agent>` | Restart pod or check logs |
| High latency | Grafana agent latency dashboard | Scale agent or check Anthropic API status |
| Workflow stuck in "running" | `SELECT * FROM workflows WHERE status='running'` | Check Celery worker logs, restart worker |
| Auth failures | Check Redis blacklist, JWT expiry | Verify JWT_SECRET consistency, clear blacklist |
| OPA denying requests | `curl http://opa:8181/v1/data/agents/allow` with test input | Review and update Rego policies |
| Redpanda lag | `rpk group describe ai-agents-consumer` | Scale consumers or check Redpanda health |
| Database connection errors | `kubectl logs <orchestration-engine-pod>` | Verify DATABASE_URL, check PostgreSQL health |
