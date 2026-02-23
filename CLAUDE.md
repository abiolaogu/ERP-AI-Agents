# CLAUDE.md -- AI-Agents Platform

This file provides context for Claude Code (or any LLM-powered coding assistant) when working
inside this repository.

---

## Project Overview

**AI-Agents** is an enterprise multi-agent AI platform containing 1,500 production-ready
specialised agents across 29 business categories. Each agent is a standalone FastAPI
microservice backed by Anthropic Claude 3.5 Sonnet. A central orchestration engine
dispatches workflows, and a React/TypeScript frontend serves as the Agent Marketplace.

---

## Repository Layout

```
AI-Agents/
  agents/definitions/          # 1,500 YAML + JSON agent definitions (29 category subdirs)
  config-management/           # Consul-based centralised config service
  docs/                        # All documentation (guides, project_docs, training, videos, design)
  examples/                    # 30 hand-crafted reference agents (Python + Go)
  experimental/                # Prototype backends (backend.py, aiops_engine.py, pipeline.py)
  generated-agents/            # 1,500 generated agents (app.py, Dockerfile, requirements.txt, README.md each)
  infrastructure/              # K8s manifests, Docker Compose, monitoring, deploy scripts
  installation/                # One-command install scripts
  k8s/                         # Additional Kubernetes manifests (HPA, Redis, orchestration-engine)
  monitoring/                  # Prometheus config (prometheus.yml)
  packages/                    # Shared Python packages:
    agent_framework/           #   BaseAgent, EnhancedAgent, AgentTeam, AgentLoader
    integration_framework/     #   BaseConnector, CredentialManager
    multi_framework/           #   FrameworkOrchestrator (LangGraph, CrewAI, AutoGen adapters)
    performance/               #   CacheManager
    vector_memory/             #   Qdrant integration
  policies/                    # OPA Rego policies + roles data
  scripts/                     # Agent generation, expansion, testing, security scan scripts
  security/                    # Vault config, K8s network/pod policies, API gateway auth middleware
  services/                    # Core microservices:
    orchestration_engine/      #   FastAPI app, AgentManager, WorkflowManager, AnalyticsManager,
                               #   Celery worker, Redpanda manager, auth, database, schemas
    seo_agent/                 #   Legacy hand-crafted agent
    lead_scoring_agent/        #   Legacy hand-crafted agent
    crm_data_entry_agent/      #   Legacy hand-crafted agent (HubSpot connector)
    meeting_scheduling_agent/  #   Legacy hand-crafted agent
    proposal_generation_agent/ #   Legacy hand-crafted agent
    email_campaign_agent/      #   Legacy hand-crafted agent
    social_media_agent/        #   Legacy hand-crafted agent
    brand_voice_consistency_agent/
    competitor_analysis_agent/
  testing/                     # Pytest config, Locust load tests, test framework
  tests/                       # Unit tests (agent_manager, workflow_manager), e2e tests
  tools/generators/            # agent_generator_v2.py, yaml_catalog_generator.py
  web/                         # React TypeScript frontend (Vite):
    src/pages/                 #   AgentMarketplace, Dashboard, Analytics, Login, Register
    src/components/            #   Layout
    src/context/               #   AuthContext
    src/services/              #   authService
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend Language | Python 3.11 |
| Backend Framework | FastAPI (async, Pydantic v2) |
| LLM Provider | Anthropic Claude 3.5 Sonnet (`claude-3-5-sonnet-20241022`) |
| LLM SDK | `anthropic` Python SDK |
| Multi-Framework | LangGraph, CrewAI, AutoGen adapters |
| Task Queue | Celery (Redis broker) |
| Event Streaming | Redpanda (Kafka-compatible) |
| Database | PostgreSQL 15 (async via SQLAlchemy) |
| Cache | Redis 7 |
| Policy Engine | Open Policy Agent (OPA) |
| Config Store | Consul |
| Secrets | HashiCorp Vault |
| Frontend | React 18 + TypeScript + Vite |
| Containerisation | Docker (multi-stage builds) |
| Orchestration | Kubernetes (manifests + HPA) |
| Monitoring | Prometheus + Grafana + Loki + Promtail + AlertManager |
| Auth | JWT (HS256) + API key + Redis token blacklist |
| Testing | pytest + Locust |

---

## Common Commands

```bash
# Start the full local stack
docker compose up -d

# Run unit tests
cd tests && python -m pytest -v

# Run e2e tests (requires running stack)
python scripts/run_e2e_test.py

# Generate K8s manifests for all agents
python infrastructure/scripts/generate_k8s_manifests.py --environment=production

# Build a single generated agent
cd generated-agents/<agent-name> && docker build -t <agent-name>:latest .

# Run security scans
./scripts/run_security_scans.sh

# Generate new agent definitions
python scripts/generate_agent_definitions.py

# Generate agent implementations from YAML
python tools/generators/agent_generator_v2.py
```

---

## Key Architecture Decisions

1. **One agent = one microservice**: Each of the 1,500 agents is a self-contained FastAPI app
   with its own Dockerfile, enabling independent scaling and deployment.

2. **Orchestration via workflows**: The `WorkflowManager` creates multi-step workflows that
   chain agent calls sequentially. Celery handles async dispatch; Redpanda provides event
   streaming.

3. **Multi-framework adapters**: The `FrameworkOrchestrator` pattern allows the same agent
   definition to run under LangGraph (complex state machines), CrewAI (role-based teams),
   AutoGen (conversational), or the custom framework.

4. **OPA policy engine**: Agent access control is defined in Rego policies under `policies/`.
   The OPA sidecar evaluates permissions at runtime.

5. **Agent definitions as data**: YAML/JSON definitions in `agents/definitions/` are the
   source of truth. Code is generated from these definitions via `tools/generators/`.

---

## Environment Variables

The following must be set (see `.env.example`):

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | Claude API access |
| `DATABASE_URL` | PostgreSQL connection string |
| `REDIS_URL` | Redis connection string |
| `OPA_URL` | Open Policy Agent endpoint |
| `SECRET_KEY` | App-level secret |
| `JWT_SECRET` | JWT signing secret |

---

## Known Issues

- The `AgentManager` only loads `.json` definitions; `.yaml` definitions in subdirectories
  are not loaded automatically.
- `AgentMarketplace.tsx` hardcodes `API_BASE_URL` to port 5000 instead of using the Vite
  env variable (`VITE_API_URL`).
- Port 8081 is claimed by both `redpanda` and `redis-commander` in `docker-compose.yml`.
- Generated agent `app.py` files contain `CLAUDE_API_KEY = "your-api-key-here"` placeholder
  strings instead of reading from environment.
- No Alembic migrations; schema is created via `create_all()` at startup.
- CORS is set to `allow_origins=["*"]` in the orchestration engine.

---

## Conventions

- **Agent naming**: `{category}_{agent}_{number}_{id}` (e.g. `healthcare_wellness_agent_1_351`).
- **Port assignment**: `8200 + (agent_id_number % 800)`, range 8200-9000.
- **API contract**: Every agent exposes `POST /api/v1/execute`, `GET /health`, `GET /metrics`,
  `GET /`.
- **Metrics**: Prometheus counters (`agent_requests_total`) and histograms
  (`agent_processing_seconds`) per agent.
- **Branches**: Feature branches off `main`. No `develop` branch observed.
- **Python style**: Type hints encouraged. Pydantic for request/response models. Async preferred.
- **Config**: Environment variables for secrets. Consul for dynamic runtime config. OPA for
  policy.
