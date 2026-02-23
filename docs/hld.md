# High-Level Design (HLD) -- AI-Agents Platform
> Version: 1.0 | Last Updated: 2026-02-18 | Status: Draft
> Classification: Internal | Author: AIDD System

---

## 1. Purpose

This High-Level Design document provides a system-wide view of the AI-Agents Platform, describing the major subsystems, their responsibilities, interactions, and the rationale behind key architectural decisions. It serves as the bridge between business requirements and the detailed low-level design.

---

## 2. System Overview

The AI-Agents Platform is a distributed, cloud-native system comprising four primary subsystems:

1. **Presentation Subsystem** -- React-based web application for agent discovery and management
2. **Orchestration Subsystem** -- Central FastAPI engine for routing, workflow management, and analytics
3. **Agent Subsystem** -- 1,500 independent FastAPI microservices, each wrapping Anthropic Claude 3.5 Sonnet
4. **Infrastructure Subsystem** -- Databases, caches, messaging, policy engine, secrets, and monitoring

---

## 3. Subsystem Decomposition

### 3.1 Presentation Subsystem

```
+---------------------------------------------------------------+
|  React 18 + TypeScript + Vite                                  |
|                                                                 |
|  Pages:                                                         |
|  +------------------+  +------------+  +-----------+           |
|  | AgentMarketplace |  | Dashboard  |  | Analytics |           |
|  +------------------+  +------------+  +-----------+           |
|  +------------------+  +------------+                           |
|  | Login            |  | Register   |                           |
|  +------------------+  +------------+                           |
|                                                                 |
|  Shared:                                                        |
|  +------------------+  +------------+  +-----------+           |
|  | AuthContext       |  | Layout     |  | authService|          |
|  +------------------+  +------------+  +-----------+           |
+---------------------------------------------------------------+
```

**Responsibilities**:
- Agent browsing, searching, and filtering by category
- User authentication (login/register flows)
- Dashboard for user's active agents and workflow status
- Analytics visualisation for usage statistics

**Communication**: REST API calls to the Orchestration Subsystem via HTTP/JSON.

### 3.2 Orchestration Subsystem

```
+---------------------------------------------------------------+
|  FastAPI Orchestration Engine                                    |
|                                                                 |
|  Core Managers:                                                 |
|  +------------------+  +--------------------+                   |
|  | AgentManager     |  | WorkflowManager    |                   |
|  | - Load defs      |  | - Create workflows |                   |
|  | - Route requests |  | - Execute steps    |                   |
|  | - Health checks  |  | - Track status     |                   |
|  +------------------+  +--------------------+                   |
|                                                                 |
|  +------------------+  +--------------------+                   |
|  | AnalyticsManager |  | Auth Module        |                   |
|  | - Usage stats    |  | - JWT issue/verify |                   |
|  | - Performance    |  | - API key auth     |                   |
|  | - Reports        |  | - Token blacklist  |                   |
|  +------------------+  +--------------------+                   |
|                                                                 |
|  Workers:                                                       |
|  +------------------+  +--------------------+                   |
|  | Celery Workers   |  | Redpanda Manager   |                   |
|  | - Async tasks    |  | - Event publish    |                   |
|  | - Workflow steps |  | - Event consume    |                   |
|  +------------------+  +--------------------+                   |
+---------------------------------------------------------------+
```

**Responsibilities**:
- Central API gateway for all client requests
- Agent lifecycle management (discovery, routing, health monitoring)
- Workflow creation, execution, and state management
- User authentication and authorisation
- Analytics aggregation and reporting
- Async task dispatch via Celery
- Event streaming via Redpanda

### 3.3 Agent Subsystem

```
+---------------------------------------------------------------+
|  1,500 Agent Microservices                                      |
|                                                                 |
|  Each Agent:                                                    |
|  +---------------------------+                                  |
|  | FastAPI App (app.py)      |                                  |
|  | POST /api/v1/execute      |  <-- Standard API Contract      |
|  | GET  /health              |                                  |
|  | GET  /metrics             |                                  |
|  | GET  /                    |                                  |
|  +---------------------------+                                  |
|  | Anthropic Claude SDK      |  <-- LLM Integration            |
|  | Pydantic Models           |  <-- Input Validation           |
|  | Prometheus Client         |  <-- Metrics Export             |
|  +---------------------------+                                  |
|                                                                 |
|  Shared Packages:                                               |
|  +------------------+  +---------------------+                  |
|  | agent_framework  |  | multi_framework     |                  |
|  | - BaseAgent      |  | - LangGraph adapter |                  |
|  | - EnhancedAgent  |  | - CrewAI adapter    |                  |
|  | - AgentTeam      |  | - AutoGen adapter   |                  |
|  +------------------+  +---------------------+                  |
|  +------------------+  +---------------------+                  |
|  | integration_fw   |  | vector_memory       |                  |
|  | - BaseConnector  |  | - Qdrant client     |                  |
|  | - CredentialMgr  |  | - Embedding store   |                  |
|  +------------------+  +---------------------+                  |
+---------------------------------------------------------------+
```

**Responsibilities**:
- Execute AI-powered tasks using domain-specific prompts
- Validate inputs via Pydantic before invoking Claude
- Export Prometheus metrics for observability
- Health checking for liveness/readiness probes

### 3.4 Infrastructure Subsystem

```
+---------------------------------------------------------------+
|  Persistence            | Messaging          | Security         |
|  +------------------+   | +---------------+  | +-----------+   |
|  | PostgreSQL 15    |   | | Redpanda      |  | | OPA       |   |
|  | (primary store)  |   | | (events)      |  | | (policies)|   |
|  +------------------+   | +---------------+  | +-----------+   |
|  +------------------+   | +---------------+  | +-----------+   |
|  | Redis 7          |   | | Celery/Redis  |  | | Vault     |   |
|  | (cache/broker)   |   | | (task queue)  |  | | (secrets) |   |
|  +------------------+   | +---------------+  | +-----------+   |
|  +------------------+   |                    |                  |
|  | Qdrant           |   | Monitoring         | Config           |
|  | (vectors)        |   | +---------------+  | +-----------+   |
|  +------------------+   | | Prometheus    |  | | Consul    |   |
|                         | | Grafana       |  | | (config)  |   |
|                         | | Loki          |  | +-----------+   |
|                         | | AlertManager  |  |                  |
|                         | +---------------+  |                  |
+---------------------------------------------------------------+
```

---

## 4. Key Interaction Flows

### 4.1 Agent Execution Flow

```
User -> Frontend -> Orchestration Engine -> AgentManager
                                               |
                                               v
                                          Target Agent
                                               |
                                               v
                                        Anthropic Claude API
                                               |
                                               v
                                          Target Agent (response)
                                               |
                                               v
                                        Orchestration Engine
                                               |
                                               v
                                          Frontend -> User
```

### 4.2 Workflow Execution Flow

```
User -> Frontend -> Orchestration Engine -> WorkflowManager
                                               |
                                               v
                                          Celery Task Queue (Redis)
                                               |
                                               v
                                          Celery Worker
                                               |
                                      +--------+--------+
                                      v        v        v
                                  Agent 1  Agent 2  Agent N
                                      |        |        |
                                      v        v        v
                                  Results collected in PostgreSQL
                                               |
                                               v
                                          Redpanda Event
                                               |
                                               v
                                          User notification
```

### 4.3 Authentication Flow

```
User -> POST /auth/login -> Auth Module -> Verify credentials (PostgreSQL)
                                              |
                                              v
                                         Issue JWT token
                                              |
                                              v
                                         Return token to User
                                              |
User -> Request with Bearer token -> Auth Module -> Validate JWT
                                                       |
                                                       v
                                                  OPA policy check
                                                       |
                                                       v
                                                  Allow/Deny
```

---

## 5. Data Flow Overview

| Flow | Source | Destination | Protocol | Data |
|------|--------|-------------|----------|------|
| Agent execution request | Frontend | Orchestration Engine | HTTP/REST | JSON payload |
| Agent dispatch | Orchestration Engine | Agent Service | HTTP/REST | JSON payload |
| LLM invocation | Agent Service | Anthropic API | HTTPS | Prompt + system message |
| Task dispatch | WorkflowManager | Celery (Redis) | AMQP-like | Serialised task |
| Event streaming | Orchestration Engine | Redpanda | Kafka protocol | JSON event |
| Metrics scrape | Prometheus | All services | HTTP | Prometheus exposition format |
| Log aggregation | All services | Loki (via Promtail) | HTTP | Structured JSON logs |

---

## 6. Non-Functional Design Decisions

### 6.1 Performance
- Async FastAPI for non-blocking I/O across all services
- Redis caching for frequently accessed agent metadata
- Connection pooling for PostgreSQL (SQLAlchemy async engine)
- Agent response caching with configurable TTL

### 6.2 Scalability
- Each agent is independently deployable and scalable
- Kubernetes HPA scales orchestration engine and agents based on load
- Celery workers scale horizontally based on queue depth
- Redpanda partitions for parallel event processing

### 6.3 Reliability
- Health check endpoints for Kubernetes liveness/readiness probes
- Workflow status tracking for recovery from partial failures
- Redis-backed task persistence for Celery worker recovery
- Redpanda replication for event durability

### 6.4 Security
- JWT authentication with configurable expiry and Redis blacklist
- OPA sidecar for declarative policy enforcement
- Vault for secrets management with dynamic secret rotation
- Network Policies for Kubernetes namespace isolation
- TLS for all inter-service communication in production

---

## 7. Technology Selection Summary

| Concern | Technology | Alternative Considered | Rationale for Choice |
|---------|-----------|----------------------|---------------------|
| API framework | FastAPI | Flask, Django | Async, Pydantic, auto-docs |
| LLM | Claude 3.5 Sonnet | GPT-4, Llama | Reasoning quality, safety |
| Task queue | Celery + Redis | Dramatiq, Huey | Maturity, ecosystem |
| Event streaming | Redpanda | Apache Kafka | Lower resource footprint |
| Policy | OPA | Casbin, custom RBAC | Industry standard, Rego |
| Secrets | Vault | AWS Secrets Manager | Cloud-agnostic |
| Config | Consul | etcd, Spring Config | Service discovery + KV store |
| Monitoring | Prometheus + Grafana | Datadog, New Relic | Open source, cost-effective |

---

## 8. Deployment Topology

### 8.1 Development Environment
Single Docker Compose file running all services on one machine:
- Orchestration Engine (port 8000)
- PostgreSQL (port 5432)
- Redis (port 6379)
- Redpanda (port 9092)
- OPA (port 8181)
- Frontend (port 3000)
- Selected agents (ports 8200+)

### 8.2 Production Environment
Kubernetes cluster with:
- Dedicated namespace per concern (platform, agents, monitoring, security)
- Ingress controller with TLS termination
- HPA for auto-scaling
- PersistentVolumes for stateful services
- Network Policies for traffic control
- Helm charts for version-controlled deployments

---

## 9. Constraints and Limitations

| Constraint | Impact | Mitigation Path |
|-----------|--------|-----------------|
| Single LLM provider (Anthropic) | Vendor lock-in risk | Abstract LLM client for future multi-provider support |
| Sequential-only workflows | Limited to linear processes | Planned: DAG-based workflows with parallel execution |
| No Alembic migrations | Manual schema management | Planned: Add Alembic migration framework |
| CORS wildcard in development | Security gap if used in production | Restrict origins in production config |
| JSON-only agent loading | YAML definitions not loaded | Planned: Add YAML loader to AgentManager |
