# Technical Writeup -- AI-Agents Platform
> Version: 1.0 | Last Updated: 2026-02-18 | Status: Draft
> Classification: Internal | Author: AIDD System

---

## 1. Abstract

The AI-Agents Platform is a cloud-native, microservices-based system designed to host, orchestrate, and manage 1,500 specialised AI agents across 29 enterprise business categories. Each agent is an autonomous FastAPI microservice backed by Anthropic Claude 3.5 Sonnet, exposing a standardised API contract. A central orchestration engine provides workflow management, authentication, analytics, and event-driven communication. This writeup details the technical design, implementation strategies, and engineering decisions that underpin the platform.

---

## 2. Problem Statement

Enterprises face significant barriers to adopting AI automation at scale:

1. **Fragmentation**: Each AI use case requires a separate integration, leading to inconsistent implementations.
2. **Development Cost**: Building, testing, and deploying custom AI agents from scratch takes months per use case.
3. **Governance Gap**: Ad-hoc AI deployments lack centralised security, audit trails, and policy enforcement.
4. **Scaling Difficulty**: Moving from proof-of-concept to production-grade AI services requires substantial infrastructure investment.

The AI-Agents Platform addresses these challenges by providing a unified, governed, and scalable platform for AI agent operations.

---

## 3. Architecture Overview

The system is layered into four tiers:

**Presentation Tier**: A React 18 / TypeScript frontend built with Vite, providing the Agent Marketplace, Dashboard, and Analytics pages. Authentication state is managed via React Context with JWT tokens stored in localStorage.

**Orchestration Tier**: A FastAPI-based orchestration engine that serves as the platform's control plane. It comprises:
- `AgentManager` for agent discovery, routing, and health monitoring
- `WorkflowManager` for multi-step workflow lifecycle management
- `AnalyticsManager` for usage metrics aggregation
- `Auth` module for JWT issuance, API key validation, and token blacklisting
- Celery workers for async task dispatch via Redis broker
- Redpanda manager for event streaming

**Agent Tier**: 1,500 independently deployable FastAPI microservices, each wrapping a domain-specific system prompt around Anthropic Claude 3.5 Sonnet. Agents are generated from YAML/JSON definitions using `agent_generator_v2.py`, ensuring consistency across the catalogue.

**Infrastructure Tier**: PostgreSQL 15 for persistent state, Redis 7 for caching and task brokering, Redpanda for event streaming, OPA for policy enforcement, Vault for secrets management, Consul for dynamic configuration, and a full observability stack (Prometheus, Grafana, Loki, AlertManager).

---

## 4. Agent Design and Generation

### 4.1 Definition-Driven Architecture

The platform follows a "definitions as data" philosophy. Each agent is defined in a YAML or JSON file specifying:

```yaml
name: marketing_content_agent_1_101
category: marketing
description: "Generates marketing content for blogs, social media, and email campaigns"
system_prompt: |
  You are a marketing content specialist. Generate high-quality,
  engaging content based on the user's requirements...
input_schema:
  type: object
  properties:
    topic: { type: string }
    format: { type: string, enum: [blog_post, social_media, email] }
    tone: { type: string, default: professional }
output_schema:
  type: object
  properties:
    content: { type: string }
    word_count: { type: integer }
    suggested_headlines: { type: array, items: { type: string } }
```

### 4.2 Code Generation Pipeline

The `agent_generator_v2.py` tool reads each definition and generates:
1. `app.py` -- FastAPI application with standard endpoints
2. `Dockerfile` -- Multi-stage build based on `python:3.11-slim`
3. `requirements.txt` -- Pinned dependencies (fastapi, uvicorn, anthropic, prometheus-client)
4. `README.md` -- Agent documentation

This approach ensures all 1,500 agents share identical structure, API contracts, and operational characteristics.

### 4.3 LLM Integration Pattern

Each agent follows a consistent pattern for Claude invocation:

1. **Input Validation**: Pydantic model validates the request body
2. **Prompt Construction**: System prompt from definition + user input as human message
3. **API Call**: `client.messages.create()` with `claude-3-5-sonnet-20241022`
4. **Output Extraction**: Parse `response.content[0].text`
5. **Safety Filtering**: Apply output guardrails (PII detection, content filtering)
6. **Metrics Recording**: Increment Prometheus counters, observe latency histogram

---

## 5. Orchestration Engine Deep Dive

### 5.1 Request Lifecycle

```
HTTP Request
    |
    v
FastAPI Middleware (CORS, logging)
    |
    v
Auth Middleware (JWT/API key validation)
    |
    v
OPA Policy Check (via sidecar)
    |
    v
Route Handler (AgentManager / WorkflowManager / AnalyticsManager)
    |
    v
Business Logic
    |
    v
Database / Agent Service / Celery
    |
    v
HTTP Response
```

### 5.2 Async Architecture

The orchestration engine is fully async, leveraging:
- FastAPI's async route handlers (`async def`)
- SQLAlchemy async engine with `AsyncSession`
- `httpx.AsyncClient` for agent-to-agent HTTP calls
- Celery for background task processing (sync workers with async dispatch)

### 5.3 Event-Driven Communication

Redpanda provides Kafka-compatible event streaming for:
- **Audit Trail**: Every agent execution is published as an event
- **Workflow Coordination**: Workflow lifecycle events enable monitoring and alerting
- **Analytics Pipeline**: Events are consumed for real-time analytics aggregation
- **Integration**: External systems can subscribe to agent events

---

## 6. Multi-Framework Orchestration

The `FrameworkOrchestrator` in `packages/multi_framework/` enables agents to run under different orchestration paradigms:

| Framework | Pattern | Best For |
|-----------|---------|----------|
| Custom (Default) | Direct HTTP call to agent microservice | Simple, single-agent tasks |
| LangGraph | Stateful graph with nodes and edges | Complex workflows with branching and loops |
| CrewAI | Role-based agent teams with delegation | Collaborative tasks requiring multiple perspectives |
| AutoGen | Multi-turn conversational agents | Interactive, dialogue-based problem solving |

The adapter pattern abstracts framework-specific details, allowing the same agent definition to be executed under any supported framework by specifying the framework at workflow creation time.

---

## 7. Security Architecture

### 7.1 Authentication Layer
- **JWT Tokens**: HS256-signed tokens with configurable expiry (default 60 minutes)
- **API Keys**: For machine-to-machine authentication (stored hashed in PostgreSQL)
- **Token Blacklist**: Redis set for immediate token revocation on logout

### 7.2 Authorisation Layer
- **OPA Sidecar**: Evaluates Rego policies per request
- **Policy Structure**: Policies define which users/roles can execute which agents
- **Role Data**: `policies/roles.json` maps users to roles (admin, developer, viewer)

### 7.3 Data Protection
- **In Transit**: TLS for all inter-service communication in production
- **At Rest**: PostgreSQL native encryption, Vault for secrets
- **Input Sanitisation**: Pydantic validation prevents injection attacks
- **Output Filtering**: Safety guardrails filter PII and harmful content from LLM responses

### 7.4 Infrastructure Security
- **Network Policies**: Kubernetes NetworkPolicy restricts pod-to-pod traffic
- **Pod Security**: Security contexts enforce non-root, read-only filesystems
- **Image Security**: Vulnerability scanning in CI/CD pipeline
- **Secrets Rotation**: Vault dynamic secrets with short TTLs

---

## 8. Observability Stack

### 8.1 Metrics (Prometheus)
Every agent and the orchestration engine export Prometheus metrics:
- `agent_requests_total{agent_name, status}` -- Request counter
- `agent_processing_seconds{agent_name}` -- Latency histogram
- `workflow_executions_total{status}` -- Workflow counter
- `workflow_duration_seconds` -- Workflow latency histogram

### 8.2 Logging (Loki + Promtail)
Structured JSON logs from all services are collected by Promtail and stored in Loki:
- Request/response logging (with sensitive data redacted)
- Error and exception logging with stack traces
- Workflow step execution logging

### 8.3 Dashboards (Grafana)
Pre-built dashboards for:
- Platform Overview: Total requests, active agents, error rate
- Agent Performance: Per-agent latency, throughput, error rates
- Workflow Analytics: Workflow success rate, duration distribution
- Infrastructure: CPU, memory, disk, network per service

### 8.4 Alerting (AlertManager)
Alert rules for critical conditions:
- Agent error rate > 5% over 5 minutes
- P99 latency > 10 seconds
- Workflow failure rate > 10% over 15 minutes
- Disk usage > 85%
- Memory usage > 90%

---

## 9. Deployment and Operations

### 9.1 Local Development
- `docker compose up -d` starts the full stack
- Hot-reload for orchestration engine (uvicorn --reload)
- Frontend dev server via Vite (`npm run dev`)

### 9.2 Production (Kubernetes)
- Manifests in `infrastructure/` and `k8s/`
- Horizontal Pod Autoscaler for orchestration engine and high-traffic agents
- Rolling deployments with readiness probes
- Helm charts for parameterised deployments

### 9.3 CI/CD (GitHub Actions)
- On PR: lint, type check, unit tests, security scan
- On merge to main: build Docker images, push to registry, deploy to staging
- On tag: promote staging to production

---

## 10. Performance Characteristics

| Metric | Target | Current Estimate |
|--------|--------|-----------------|
| Agent cold start | < 5 seconds | ~3 seconds (Docker container start) |
| Agent execution (simple) | P95 < 2 seconds | ~1.5 seconds (LLM-dominated) |
| Agent execution (complex) | P95 < 5 seconds | ~3-4 seconds |
| Workflow execution (3-step) | P95 < 10 seconds | ~6-8 seconds |
| Concurrent agent capacity | 50+ | Depends on cluster size |
| API gateway throughput | 1,000 RPS | FastAPI with uvicorn workers |

---

## 11. Known Limitations and Future Work

| Limitation | Impact | Planned Resolution |
|-----------|--------|-------------------|
| Sequential-only workflows | Cannot parallelise independent steps | DAG-based workflow engine |
| Single LLM provider | Vendor dependency on Anthropic | Multi-provider abstraction layer |
| No database migrations | Schema changes are destructive | Alembic migration framework |
| JSON-only agent loading | YAML definitions not utilised | YAML support in AgentManager |
| Hardcoded frontend API URL | Breaks in non-default environments | Environment variable configuration |
| No multi-tenancy | Single organisation only | Tenant isolation with namespace-per-tenant |
| No agent versioning | Cannot roll back agent changes | Semantic versioning for agent definitions |

---

## 12. Conclusion

The AI-Agents Platform demonstrates a scalable approach to enterprise AI automation through its microservices architecture, definition-driven agent generation, and comprehensive orchestration capabilities. The platform's standardised API contract, multi-framework support, and robust security model position it for enterprise adoption. Key areas for future development include DAG-based workflows, multi-provider LLM support, and multi-tenancy.
