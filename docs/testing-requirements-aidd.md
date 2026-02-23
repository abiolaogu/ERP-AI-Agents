# Testing Requirements (AIDD) -- AI-Agents Platform
> Version: 1.0 | Last Updated: 2026-02-18 | Status: Draft
> Classification: Internal | Author: AIDD System

---

## 1. Purpose

This document defines the testing requirements for the AI-Agents Platform, covering all test levels, test types, tooling, environments, and quality gates. It is aligned with the AIDD (AI-Driven Development) pipeline and establishes the testing strategy for a system comprising 1,500 AI agent microservices, a central orchestration engine, and supporting infrastructure.

---

## 2. Testing Scope

### 2.1 In Scope

| Component | Test Level | Notes |
|-----------|-----------|-------|
| Orchestration Engine (FastAPI) | Unit, Integration, E2E | Core platform logic |
| AgentManager | Unit, Integration | Agent loading, routing, health checking |
| WorkflowManager | Unit, Integration | Workflow CRUD and execution |
| Auth Module | Unit, Integration | JWT, API keys, OPA integration |
| Agent Microservices (1,500) | Unit, Contract, Smoke | Generated agents |
| React Frontend | Unit, E2E | Agent Marketplace, Dashboard, Analytics |
| Multi-Framework Adapters | Unit, Integration | LangGraph, CrewAI, AutoGen |
| OPA Policies | Unit | Rego policy validation |
| Infrastructure | Integration, Smoke | Docker Compose, Kubernetes |
| API Endpoints | Contract, Integration | REST API compliance |

### 2.2 Out of Scope

- Testing Anthropic Claude API reliability (external dependency)
- Load testing the LLM provider's infrastructure
- Testing third-party managed services (AWS RDS, managed Redis)

---

## 3. Test Levels

### 3.1 Unit Testing

**Purpose**: Validate individual functions and methods in isolation.

**Scope**:
- Orchestration engine managers (AgentManager, WorkflowManager, AnalyticsManager)
- Auth functions (token creation, verification, password hashing)
- Agent framework classes (BaseAgent, EnhancedAgent, AgentTeam)
- Multi-framework adapters
- Frontend React components

**Requirements**:
| ID | Requirement |
|----|------------|
| TR-UNIT-001 | All public methods in orchestration engine managers must have unit tests |
| TR-UNIT-002 | Auth module must have 100% branch coverage for token validation |
| TR-UNIT-003 | Agent framework base classes must have unit tests for all methods |
| TR-UNIT-004 | Frontend components must have snapshot and interaction tests |
| TR-UNIT-005 | OPA Rego policies must have unit tests covering allow and deny paths |
| TR-UNIT-006 | Minimum code coverage target: 80% for orchestration engine |
| TR-UNIT-007 | Unit tests must not require external services (mock all dependencies) |
| TR-UNIT-008 | Unit tests must complete within 60 seconds total |

**Tooling**:
- Python: pytest, pytest-asyncio, pytest-cov, unittest.mock
- JavaScript/TypeScript: Vitest, React Testing Library
- OPA: `opa test`

### 3.2 Integration Testing

**Purpose**: Validate interactions between components and with real infrastructure.

**Scope**:
- Orchestration engine with PostgreSQL
- Orchestration engine with Redis
- Orchestration engine with agent microservices
- Celery workers with Redis broker
- Redpanda event publishing and consumption
- OPA policy evaluation from orchestration engine
- Frontend with orchestration engine API

**Requirements**:
| ID | Requirement |
|----|------------|
| TR-INT-001 | Auth flow must be tested end-to-end: register, login, token usage, logout |
| TR-INT-002 | Agent execution must be tested through the orchestration engine to a real agent |
| TR-INT-003 | Workflow creation and execution must be tested with Celery and real agents |
| TR-INT-004 | Database operations must be tested against a real PostgreSQL instance |
| TR-INT-005 | Redis operations must be tested against a real Redis instance |
| TR-INT-006 | Redpanda event publishing must be verified with a real Redpanda instance |
| TR-INT-007 | OPA policy evaluation must be tested against a running OPA instance |
| TR-INT-008 | Integration tests must use a dedicated test database (not production data) |

**Tooling**:
- pytest with testcontainers (or Docker Compose test environment)
- httpx.AsyncClient for API testing

### 3.3 End-to-End (E2E) Testing

**Purpose**: Validate complete user journeys through the full stack.

**Requirements**:
| ID | Requirement |
|----|------------|
| TR-E2E-001 | User registration, login, agent browsing, and execution must be tested end-to-end |
| TR-E2E-002 | Workflow creation, execution, and result retrieval must be tested end-to-end |
| TR-E2E-003 | Frontend flows must be tested via browser automation |
| TR-E2E-004 | E2E tests must run against a fully deployed stack (Docker Compose) |
| TR-E2E-005 | E2E test suite must complete within 10 minutes |

**Tooling**:
- `scripts/run_e2e_test.py` for API-level E2E
- Playwright or Cypress for frontend E2E (recommended)

### 3.4 Contract Testing

**Purpose**: Ensure agents conform to the standard API contract.

**Requirements**:
| ID | Requirement |
|----|------------|
| TR-CON-001 | Every generated agent must expose POST /api/v1/execute |
| TR-CON-002 | Every generated agent must expose GET /health returning {"status": "healthy"} |
| TR-CON-003 | Every generated agent must expose GET /metrics in Prometheus format |
| TR-CON-004 | Every generated agent must expose GET / returning agent info |
| TR-CON-005 | Contract tests must be runnable against any agent by parameterising the URL |

### 3.5 Load Testing

**Purpose**: Validate platform performance under expected and peak loads.

**Requirements**:
| ID | Requirement |
|----|------------|
| TR-LOAD-001 | Platform must handle 50 concurrent agent executions without errors |
| TR-LOAD-002 | P95 agent execution latency must remain below 5 seconds under load |
| TR-LOAD-003 | Orchestration engine must handle 1,000 RPS for listing endpoints |
| TR-LOAD-004 | Workflow execution must complete 3-step workflows within 15 seconds under load |
| TR-LOAD-005 | Load tests must simulate realistic usage patterns (mixed read/write) |

**Tooling**:
- Locust (`testing/locustfile.py`)
- Target: orchestration engine endpoints

### 3.6 Security Testing

**Purpose**: Validate security controls and identify vulnerabilities.

**Requirements**:
| ID | Requirement |
|----|------------|
| TR-SEC-001 | All API endpoints must reject unauthenticated requests (except /auth/register, /auth/login) |
| TR-SEC-002 | JWT tokens with tampered signatures must be rejected |
| TR-SEC-003 | SQL injection attempts must be blocked by Pydantic validation |
| TR-SEC-004 | XSS payloads in agent inputs must be sanitised |
| TR-SEC-005 | Dependency vulnerability scan must report zero critical/high vulnerabilities |
| TR-SEC-006 | Docker images must pass Trivy security scan |
| TR-SEC-007 | Kubernetes manifests must pass kubesec security scan |
| TR-SEC-008 | OPA policies must deny access for unauthorised users |

**Tooling**:
- `scripts/run_security_scans.sh`
- pip-audit / safety for Python dependencies
- trivy for Docker images
- kubesec for Kubernetes manifests

---

## 4. Test Environments

| Environment | Purpose | Infrastructure |
|-------------|---------|---------------|
| Local | Developer testing | Docker Compose on developer machine |
| CI | Automated testing on PR | GitHub Actions runners with Docker |
| Staging | Pre-production validation | Kubernetes cluster (dedicated namespace) |
| Production | Smoke tests post-deployment | Production Kubernetes cluster |

---

## 5. Test Data Requirements

| Data Type | Source | Management |
|-----------|--------|------------|
| User accounts | Test fixtures | Created and torn down per test session |
| Agent definitions | Repository (agents/definitions/) | Static test fixtures |
| Workflow definitions | Test fixtures | Created per test case |
| LLM responses | Mocked (unit/integration) or real (E2E) | Mock fixtures for deterministic tests |

---

## 6. Quality Gates

### 6.1 PR Merge Gate

| Gate | Threshold | Blocking |
|------|-----------|----------|
| Unit test pass rate | 100% | Yes |
| Code coverage (orchestration engine) | >= 80% | Yes |
| Integration test pass rate | 100% | Yes |
| Linting (Python + TypeScript) | Zero errors | Yes |
| Type checking (mypy + tsc) | Zero errors | Yes |
| Security scan (dependencies) | Zero critical/high | Yes |
| Docker build | Successful | Yes |

### 6.2 Release Gate

| Gate | Threshold | Blocking |
|------|-----------|----------|
| All PR gates pass | Yes | Yes |
| E2E test pass rate | 100% | Yes |
| Load test (50 concurrent) | Zero errors, P95 < 5s | Yes |
| Security scan (images + manifests) | Zero critical/high | Yes |
| Contract tests (sample of 50 agents) | 100% | Yes |
| Manual QA sign-off | Approved | Yes |

---

## 7. Test Automation Strategy

### 7.1 CI/CD Integration

```
PR Created/Updated:
  1. Lint (flake8/ruff + eslint) ............ ~30s
  2. Type Check (mypy + tsc) ................ ~30s
  3. Unit Tests (pytest + vitest) ........... ~60s
  4. Security Scan (pip-audit + trivy) ...... ~120s
  5. Docker Build (orchestration-engine) .... ~120s
  Total: ~6 minutes

Merge to Main:
  6. Integration Tests (Docker Compose) ..... ~300s
  7. E2E Tests (full stack) ................. ~300s
  8. Contract Tests (agent sample) .......... ~120s
  9. Deploy to Staging ...................... ~180s
  Total: ~15 minutes

Release Tag:
  10. Load Tests (Locust) .................. ~600s
  11. Full Security Scan ................... ~300s
  12. Manual QA Verification ............... Manual
  13. Deploy to Production ................. ~180s
```

### 7.2 Test Reporting

| Report | Format | Destination |
|--------|--------|-------------|
| Unit test results | JUnit XML | GitHub Actions summary |
| Coverage report | HTML + Cobertura XML | GitHub Actions artifact |
| Load test results | HTML (Locust report) | GitHub Actions artifact |
| Security scan results | SARIF | GitHub Security tab |

---

## 8. Defect Classification

| Severity | Definition | SLA |
|----------|-----------|-----|
| Critical | Platform outage, data loss, security breach | Fix within 4 hours |
| High | Major feature broken, significant performance degradation | Fix within 24 hours |
| Medium | Minor feature issue, workaround available | Fix within 1 sprint |
| Low | Cosmetic issue, documentation error | Fix when convenient |

---

## 9. Test Ownership

| Test Type | Owner | Reviewer |
|-----------|-------|----------|
| Unit tests | Feature developer | PR reviewer |
| Integration tests | Platform team | QA engineer |
| E2E tests | QA team | Platform team |
| Load tests | SRE team | Platform team |
| Security tests | Security team | Platform team |
| Contract tests | Platform team | QA engineer |
| OPA policy tests | Security team | Platform team |
