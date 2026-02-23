# Software Requirements Specification (SRS) -- AI-Agents Platform
> Version: 1.0 | Last Updated: 2026-02-18 | Status: Draft
> Classification: Internal | Author: AIDD System

---

## 1. Introduction

### 1.1 Purpose
This Software Requirements Specification defines the functional and non-functional requirements for the AI-Agents Platform, a multi-agent orchestration system hosting 1,500 specialised AI agents. It serves as the authoritative reference for development, testing, and acceptance criteria.

### 1.2 Scope
The SRS covers all platform components: the React frontend (Agent Marketplace), FastAPI orchestration engine, 1,500 agent microservices, supporting infrastructure services, and all integration points.

### 1.3 Definitions

| Term | Definition |
|------|-----------|
| Agent | A standalone FastAPI microservice that performs a specific AI-driven task |
| Workflow | An ordered sequence of agent executions |
| Orchestration Engine | Central FastAPI service managing agents, workflows, auth, and analytics |
| Agent Definition | YAML/JSON file specifying an agent's identity, prompt, and schema |
| Framework Adapter | Module enabling agent execution under LangGraph, CrewAI, or AutoGen |

---

## 2. System Overview

The AI-Agents Platform comprises:
- **1,500 agent microservices** across 29 business categories
- **Orchestration engine** for agent management, workflow execution, and analytics
- **React frontend** for agent discovery, workflow management, and analytics
- **Infrastructure services**: PostgreSQL, Redis, Redpanda, OPA, Vault, Consul
- **Monitoring stack**: Prometheus, Grafana, Loki, AlertManager

---

## 3. Functional Requirements

### 3.1 Authentication and Authorisation

| ID | Requirement | Priority |
|----|------------|----------|
| SRS-AUTH-001 | System shall support user registration with username and password | P0 |
| SRS-AUTH-002 | System shall authenticate users via username/password and return a JWT token | P0 |
| SRS-AUTH-003 | JWT tokens shall use HS256 algorithm with configurable expiry | P0 |
| SRS-AUTH-004 | System shall support API key authentication for programmatic access | P0 |
| SRS-AUTH-005 | System shall maintain a Redis-based token blacklist for logout/revocation | P0 |
| SRS-AUTH-006 | System shall enforce OPA policies for agent-level access control | P0 |
| SRS-AUTH-007 | Passwords shall be hashed using bcrypt with a minimum of 12 salt rounds | P0 |
| SRS-AUTH-008 | System shall reject requests with expired, invalid, or blacklisted tokens | P0 |

### 3.2 Agent Management

| ID | Requirement | Priority |
|----|------------|----------|
| SRS-AGT-001 | System shall load agent definitions from JSON files at startup | P0 |
| SRS-AGT-002 | System shall support loading agent definitions from YAML files | P1 |
| SRS-AGT-003 | Each agent shall expose POST /api/v1/execute for task execution | P0 |
| SRS-AGT-004 | Each agent shall expose GET /health for liveness/readiness checks | P0 |
| SRS-AGT-005 | Each agent shall expose GET /metrics in Prometheus exposition format | P0 |
| SRS-AGT-006 | Each agent shall expose GET / returning agent info and description | P1 |
| SRS-AGT-007 | Agent input shall be validated via Pydantic models before LLM invocation | P0 |
| SRS-AGT-008 | Agent output shall pass through safety guardrails before returning to user | P0 |
| SRS-AGT-009 | System shall support agent search by name, description, and category | P0 |
| SRS-AGT-010 | System shall support pagination for agent listing (default limit: 50) | P1 |
| SRS-AGT-011 | Agents shall be assigned ports using formula: 8200 + (agent_id % 800) | P0 |

### 3.3 Workflow Management

| ID | Requirement | Priority |
|----|------------|----------|
| SRS-WF-001 | System shall support creating workflows with a name and list of tasks | P0 |
| SRS-WF-002 | Each workflow task shall reference an agent ID and provide input data | P0 |
| SRS-WF-003 | System shall assign a UUID to each workflow | P0 |
| SRS-WF-004 | System shall track workflow status: pending, running, completed, failed | P0 |
| SRS-WF-005 | System shall execute workflow tasks sequentially via Celery workers | P0 |
| SRS-WF-006 | System shall store workflow results in PostgreSQL as a JSON array | P0 |
| SRS-WF-007 | System shall publish workflow events to Redpanda | P1 |
| SRS-WF-008 | System shall support listing workflows by user | P0 |
| SRS-WF-009 | System shall support workflow re-execution | P1 |
| SRS-WF-010 | System shall support parallel task execution within workflows | P2 |

### 3.4 Multi-Framework Support

| ID | Requirement | Priority |
|----|------------|----------|
| SRS-MF-001 | System shall support executing agents under the custom (default) framework | P0 |
| SRS-MF-002 | System shall support executing agents under LangGraph | P1 |
| SRS-MF-003 | System shall support executing agents under CrewAI | P1 |
| SRS-MF-004 | System shall support executing agents under AutoGen | P2 |
| SRS-MF-005 | Framework selection shall be configurable per workflow step | P1 |

### 3.5 Frontend (Agent Marketplace)

| ID | Requirement | Priority |
|----|------------|----------|
| SRS-FE-001 | Frontend shall display agents grouped by 29 categories | P0 |
| SRS-FE-002 | Frontend shall provide keyword search across agent name and description | P0 |
| SRS-FE-003 | Frontend shall display agent detail pages with description and schemas | P0 |
| SRS-FE-004 | Frontend shall provide login and registration forms | P0 |
| SRS-FE-005 | Frontend shall persist authentication state across page refreshes | P0 |
| SRS-FE-006 | Frontend shall display a dashboard with active agents and workflow status | P1 |
| SRS-FE-007 | Frontend shall display an analytics page with usage statistics | P1 |
| SRS-FE-008 | Frontend shall use VITE_API_URL environment variable for API base URL | P0 |

### 3.6 Analytics and Reporting

| ID | Requirement | Priority |
|----|------------|----------|
| SRS-AN-001 | System shall track total agent executions by agent and status | P0 |
| SRS-AN-002 | System shall track execution latency per agent | P0 |
| SRS-AN-003 | System shall provide aggregated analytics via AnalyticsManager | P1 |
| SRS-AN-004 | System shall export Prometheus metrics for external dashboarding | P0 |
| SRS-AN-005 | System shall support log aggregation via Loki | P1 |

---

## 4. Non-Functional Requirements

### 4.1 Performance

| ID | Requirement | Target |
|----|------------|--------|
| SRS-PERF-001 | Agent execution latency (P95) | < 2 seconds (excl. LLM time) |
| SRS-PERF-002 | Agent execution latency (P99) | < 5 seconds (excl. LLM time) |
| SRS-PERF-003 | Concurrent agent executions | 50+ per orchestration node |
| SRS-PERF-004 | API gateway throughput | 1,000+ requests per second |
| SRS-PERF-005 | Agent cold start time | < 5 seconds |
| SRS-PERF-006 | Workflow execution (3-step) P95 | < 15 seconds |

### 4.2 Availability

| ID | Requirement | Target |
|----|------------|--------|
| SRS-AVAIL-001 | Platform uptime | 99.5% |
| SRS-AVAIL-002 | Recovery Time Objective | < 30 minutes |
| SRS-AVAIL-003 | Recovery Point Objective | < 5 minutes |
| SRS-AVAIL-004 | Zero-downtime deployments | Rolling updates via Kubernetes |

### 4.3 Scalability

| ID | Requirement | Target |
|----|------------|--------|
| SRS-SCALE-001 | Horizontal scaling for orchestration engine | Kubernetes HPA |
| SRS-SCALE-002 | Independent scaling per agent | Kubernetes HPA per deployment |
| SRS-SCALE-003 | Celery worker horizontal scaling | Based on queue depth |
| SRS-SCALE-004 | Database read scaling | PostgreSQL read replicas |

### 4.4 Security

| ID | Requirement | Target |
|----|------------|--------|
| SRS-SEC-001 | All API endpoints require authentication | JWT or API key |
| SRS-SEC-002 | Secrets stored in HashiCorp Vault | No plaintext secrets in code |
| SRS-SEC-003 | TLS for all inter-service communication | In production |
| SRS-SEC-004 | Network Policies for Kubernetes | Namespace-level isolation |
| SRS-SEC-005 | Automated vulnerability scanning | CI/CD pipeline |
| SRS-SEC-006 | Input sanitisation on all endpoints | Pydantic validation |

### 4.5 Maintainability

| ID | Requirement | Target |
|----|------------|--------|
| SRS-MAINT-001 | Standardised agent API contract | All agents follow same endpoints |
| SRS-MAINT-002 | Agent generation from definitions | YAML/JSON to code pipeline |
| SRS-MAINT-003 | Infrastructure as Code | Docker Compose + K8s manifests |
| SRS-MAINT-004 | Comprehensive documentation | Docs covering all components |

---

## 5. Interface Requirements

### 5.1 External Interfaces

| Interface | Protocol | Direction | Data Format |
|-----------|----------|-----------|-------------|
| Anthropic Claude API | HTTPS | Outbound | JSON |
| HubSpot CRM | HTTPS | Bidirectional | JSON |
| SMTP Email | SMTP/TLS | Outbound | MIME |
| Calendar APIs | HTTPS | Bidirectional | JSON/iCal |

### 5.2 Internal Interfaces

| Interface | Protocol | Data Format |
|-----------|----------|-------------|
| Frontend to Orchestration | HTTP/REST | JSON |
| Orchestration to Agents | HTTP/REST | JSON |
| Orchestration to PostgreSQL | TCP (async SQLAlchemy) | SQL |
| Orchestration to Redis | TCP (Redis protocol) | Key-Value |
| Orchestration to Redpanda | TCP (Kafka protocol) | JSON |
| Orchestration to OPA | HTTP/REST | JSON |
| Prometheus to All Services | HTTP | Prometheus exposition |
| Promtail to Loki | HTTP | JSON logs |

---

## 6. Data Requirements

### 6.1 Data Entities

| Entity | Storage | Estimated Volume |
|--------|---------|-----------------|
| Users | PostgreSQL | 1,000s |
| Workflows | PostgreSQL | 10,000s |
| Agent Definitions | Filesystem (YAML/JSON) | 1,500 |
| Execution Events | Redpanda | 100,000s/day |
| Metrics | Prometheus TSDB | Millions of data points |
| Logs | Loki | GBs/day |
| Secrets | Vault | 100s |
| Configuration | Consul | 100s of KV pairs |

### 6.2 Data Retention

| Data Type | Retention Period | Basis |
|-----------|-----------------|-------|
| User accounts | Indefinite | Account lifecycle |
| Workflow records | 90 days | Configurable |
| Execution events | 7 days (Redpanda) | Topic retention |
| Metrics | 30 days | Prometheus TSDB retention |
| Logs | 30 days | Loki retention |
| Audit logs | 1 year | Compliance requirement |

---

## 7. Software Prerequisites

| Component | Minimum Version | Purpose |
|-----------|----------------|---------|
| Python | 3.9+ (recommended 3.11) | Backend runtime |
| Node.js | 18+ | Frontend build |
| Docker | 24.0+ | Containerisation |
| Docker Compose | 2.0+ | Local development |
| Kubernetes | 1.27+ | Production orchestration |
| PostgreSQL | 15 | Primary database |
| Redis | 7 | Cache and broker |
