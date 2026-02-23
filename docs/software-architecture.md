# Software Architecture Document -- AI-Agents Platform
> Version: 1.0 | Last Updated: 2026-02-18 | Status: Draft
> Classification: Internal | Author: AIDD System

---

## 1. Introduction

This document describes the software architecture of the AI-Agents Platform, a cloud-native multi-agent orchestration system hosting 1,500 specialised AI agents. It complements the existing `architecture.md` by providing deeper detail on software design patterns, component interactions, and technology rationale.

---

## 2. Architectural Style

The AI-Agents Platform follows a **microservices architecture** with event-driven communication. Key architectural patterns include:

- **Service-per-Agent**: Each of the 1,500 agents is an independent FastAPI microservice with its own container, enabling isolated scaling and deployment.
- **Orchestrator Pattern**: A central orchestration engine coordinates multi-agent workflows, acting as the single entry point for workflow dispatch.
- **Event Sourcing (Partial)**: Redpanda streams capture agent execution events for downstream processing, analytics, and audit.
- **Sidecar Pattern**: OPA runs as a sidecar for policy evaluation at the orchestration layer.
- **Gateway Pattern**: The orchestration engine serves as the API gateway, routing requests to individual agents.

---

## 3. System Context

```
                    +-----------+
                    |  End User |
                    +-----+-----+
                          |
                          v
                 +--------+--------+
                 | React Frontend  |
                 | (Vite/TS)       |
                 +--------+--------+
                          |
                          v
              +-----------+-----------+
              | Orchestration Engine  |
              | (FastAPI)             |
              +---+------+------+----+
                  |      |      |
         +--------+  +---+---+  +--------+
         v           v       v           v
    +--------+  +--------+  +--------+  +--------+
    | Agent  |  | Agent  |  | Agent  |  | Agent  |
    | 1..N   |  | N+1..M |  | M+1..P |  | P+1..  |
    +--------+  +--------+  +--------+  +--------+
         |           |           |           |
         v           v           v           v
    +--------------------------------------------+
    |        Anthropic Claude 3.5 Sonnet         |
    +--------------------------------------------+
```

---

## 4. Component Architecture

### 4.1 Orchestration Engine (`services/orchestration_engine/`)

The central nervous system of the platform, responsible for:

| Component | Module | Responsibility |
|-----------|--------|----------------|
| AgentManager | `agent_manager.py` | Load agent definitions, route execution requests, health monitoring |
| WorkflowManager | `workflow_manager.py` | Create, execute, and track multi-step workflows |
| AnalyticsManager | `analytics_manager.py` | Aggregate execution metrics, generate usage reports |
| Auth Module | `auth.py` | JWT issuance/validation, API key management, token blacklist |
| Database Layer | `database.py` | SQLAlchemy async models, connection pooling |
| Celery Workers | `celery_worker.py` | Async task execution for workflow steps |
| Redpanda Manager | `redpanda_manager.py` | Event publishing and consumption |

### 4.2 Agent Microservices (`generated-agents/`)

Each agent follows an identical structure:

```
agent_name/
  app.py              # FastAPI application with /api/v1/execute, /health, /metrics
  Dockerfile          # Multi-stage build (python:3.11-slim)
  requirements.txt    # fastapi, uvicorn, anthropic, prometheus-client
  README.md           # Agent description and usage
```

**Standard API Contract**:
- `POST /api/v1/execute` -- Execute the agent with JSON input
- `GET /health` -- Health check returning `{"status": "healthy"}`
- `GET /metrics` -- Prometheus metrics (requests_total, processing_seconds)
- `GET /` -- Agent info and description

### 4.3 Agent Framework (`packages/agent_framework/`)

Shared base classes for agent implementations:

- **BaseAgent**: Minimal agent with execute method and health check
- **EnhancedAgent**: BaseAgent + memory, tool use, conversation history
- **AgentTeam**: Coordinates multiple agents in a team configuration
- **AgentLoader**: Discovers and loads agent definitions from YAML/JSON

### 4.4 Multi-Framework Layer (`packages/multi_framework/`)

The `FrameworkOrchestrator` enables the same agent definition to run under different orchestration frameworks:

| Framework | Use Case | Integration |
|-----------|----------|-------------|
| Custom (Default) | Simple sequential execution | Direct FastAPI calls |
| LangGraph | Complex state machines with branching | Graph-based agent workflows |
| CrewAI | Role-based team collaboration | Agent role assignment and delegation |
| AutoGen | Multi-turn conversational patterns | Conversational agent groups |

### 4.5 Frontend (`web/`)

React 18 + TypeScript application built with Vite:

| Page | Route | Purpose |
|------|-------|---------|
| AgentMarketplace | `/marketplace` | Browse, search, and activate agents |
| Dashboard | `/dashboard` | User workspace with active agents |
| Analytics | `/analytics` | Usage statistics and performance charts |
| Login/Register | `/login`, `/register` | Authentication |

---

## 5. Data Architecture

### 5.1 PostgreSQL 15 (Primary Store)
- **Users**: Authentication credentials and profiles
- **Workflows**: Workflow definitions, status, and results
- **Agent Definitions**: Agent metadata (supplementing file-based YAML/JSON)
- Connection pooling via SQLAlchemy async engine

### 5.2 Redis 7 (Cache and Broker)
- Celery task broker and result backend
- JWT token blacklist for logout/revocation
- Agent response caching (configurable TTL)
- Rate limiting counters

### 5.3 Redpanda (Event Streaming)
- Kafka-compatible event streaming
- Topics: `agent.executed`, `workflow.started`, `workflow.completed`, `workflow.failed`
- Used for async event processing and audit trail

### 5.4 Qdrant (Vector Memory)
- Optional vector database for agent memory
- Stores conversation embeddings for context retrieval
- Integrated via `packages/vector_memory/`

---

## 6. Integration Architecture

### 6.1 LLM Integration
- Provider: Anthropic Claude 3.5 Sonnet (`claude-3-5-sonnet-20241022`)
- SDK: `anthropic` Python package
- Pattern: Each agent constructs a system prompt from its definition, sends user input as the human message, and returns the assistant response
- Safety: Input validation before LLM call, output filtering after response

### 6.2 External System Integration
- `packages/integration_framework/` provides `BaseConnector` and `CredentialManager`
- Connectors for CRM (HubSpot), email, calendars, and custom APIs
- Credentials managed via Vault with dynamic secrets

### 6.3 Policy Integration
- OPA evaluates Rego policies at the orchestration layer
- Policy files in `policies/` define agent-level access control
- Roles data in `policies/roles.json` maps users to permissions

---

## 7. Cross-Cutting Concerns

### 7.1 Authentication and Authorisation
- JWT tokens (HS256) with configurable expiry
- API key authentication for machine-to-machine calls
- Redis-based token blacklist for immediate revocation
- OPA for fine-grained agent-level authorisation

### 7.2 Observability
- **Metrics**: Prometheus counters and histograms per agent
- **Logging**: Structured JSON logs aggregated via Loki + Promtail
- **Dashboards**: Grafana with pre-built dashboards for agent health, latency, and throughput
- **Alerting**: AlertManager with rules for error rate spikes, latency breaches, and resource exhaustion

### 7.3 Configuration Management
- Environment variables for secrets (via `.env` or Vault)
- Consul for dynamic runtime configuration (feature flags, agent settings)
- Agent definitions (YAML/JSON) as the source of truth for agent behaviour

### 7.4 Error Handling
- Structured error responses with error codes and messages
- Workflow-level retry policies (configurable per step)
- Dead letter queues for failed events
- Circuit breaker pattern for external API calls

---

## 8. Deployment Architecture

### 8.1 Local Development
- Docker Compose with all services (orchestration engine, PostgreSQL, Redis, Redpanda, OPA)
- Hot-reload for orchestration engine and frontend
- Individual agents run via `uvicorn` or Docker

### 8.2 Production (Kubernetes)
- Kubernetes manifests in `infrastructure/` and `k8s/`
- Horizontal Pod Autoscaler (HPA) for orchestration engine and high-demand agents
- Helm charts for repeatable deployments
- Network policies for service-to-service communication control

---

## 9. Technology Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Backend framework | FastAPI | Async support, Pydantic validation, auto-generated OpenAPI docs |
| LLM provider | Anthropic Claude 3.5 Sonnet | Strong reasoning, safety features, enterprise support |
| Task queue | Celery + Redis | Mature, well-understood, reliable async task processing |
| Event streaming | Redpanda | Kafka-compatible with lower resource footprint |
| Policy engine | OPA | Industry standard, declarative Rego policies |
| Secrets | Vault | Enterprise-grade secrets management with dynamic secrets |
| Frontend | React + TypeScript + Vite | Fast builds, type safety, large ecosystem |
| Database | PostgreSQL 15 | Reliable, async support via SQLAlchemy, rich query capabilities |

---

## 10. Known Technical Debt

| Item | Impact | Remediation |
|------|--------|-------------|
| No Alembic migrations | Schema changes require manual `create_all()` | Add Alembic migration framework |
| Hardcoded API URL in frontend | Breaks in non-default environments | Use `VITE_API_URL` env variable |
| API keys in generated agent code | Security vulnerability | Read from environment/Vault |
| CORS `allow_origins=["*"]` | Security risk in production | Restrict to known origins |
| Port conflict (8081) | Redpanda and Redis Commander conflict | Reassign Redis Commander port |
| AgentManager only loads JSON | YAML definitions ignored | Add YAML loading support |
