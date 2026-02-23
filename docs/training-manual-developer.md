# Developer Training Manual -- AI-Agents Platform
> Version: 1.0 | Last Updated: 2026-02-18 | Status: Draft
> Classification: Internal | Author: AIDD System

---

## 1. Introduction

This training manual provides a structured learning path for developers joining the AI-Agents Platform team. It covers architecture fundamentals, development workflows, hands-on exercises, and advanced topics. Estimated completion time: 2-3 days.

---

## 2. Learning Path Overview

| Module | Duration | Topics |
|--------|----------|--------|
| 1: Platform Architecture | 2 hours | System layers, components, data flow |
| 2: Development Setup | 1 hour | Environment, tools, local stack |
| 3: Agent Development | 3 hours | Definition, generation, customisation, testing |
| 4: Orchestration Engine | 2 hours | AgentManager, WorkflowManager, Auth |
| 5: Multi-Framework | 2 hours | LangGraph, CrewAI, AutoGen adapters |
| 6: Infrastructure | 2 hours | Docker, Kubernetes, monitoring |
| 7: Security | 1 hour | OPA, Vault, JWT, safety guardrails |
| 8: Advanced Topics | 2 hours | Custom connectors, vector memory, performance |

---

## 3. Module 1: Platform Architecture

### 3.1 The Four Layers

```
Presentation  ->  React/TypeScript frontend (Vite)
Orchestration ->  FastAPI engine (AgentManager, WorkflowManager, Celery, Redpanda)
Agent         ->  1,500 FastAPI microservices (Claude 3.5 Sonnet)
Infrastructure -> PostgreSQL, Redis, Redpanda, OPA, Vault, Consul, Prometheus
```

### 3.2 Key Architecture Decisions

**Decision 1: One Agent = One Microservice**
Each of the 1,500 agents is an independent FastAPI application with its own Docker container. This enables:
- Independent scaling (HPA per agent)
- Isolated failures (one agent crash does not affect others)
- Independent deployment and rollback
- Clear ownership and metrics per agent

**Decision 2: Definitions as Data**
Agent behaviour is defined in YAML/JSON files, not in code. Benefits:
- Non-developers can review and modify agent prompts
- Code generation ensures consistency
- Definitions serve as documentation
- Easy to audit and version control

**Decision 3: Multi-Framework Adapters**
The same agent definition can run under different orchestration frameworks:
- **Custom**: Simple HTTP-based execution (default)
- **LangGraph**: Complex state machines with conditional branching
- **CrewAI**: Role-based team collaboration
- **AutoGen**: Multi-turn conversational patterns

### 3.3 Exercise: Architecture Review
1. Read `docs/architecture.md` (full document)
2. Examine the `CLAUDE.md` file in the repository root
3. Draw a diagram showing how a request flows from the frontend through the orchestration engine to an agent and back
4. Identify three infrastructure services and describe their role

---

## 4. Module 2: Development Setup

### 4.1 Tools Installation Checklist

- [ ] Python 3.11+ installed
- [ ] Node.js 18+ installed
- [ ] Docker Desktop installed and running
- [ ] Git configured with SSH key
- [ ] VS Code (or preferred IDE) installed
- [ ] VS Code extensions: Python, ESLint, Prettier, Docker

### 4.2 Repository Structure Deep Dive

Key directories to understand:

```
agents/definitions/     # YAML/JSON agent definitions (source of truth)
generated-agents/       # Generated FastAPI agent code (DO NOT edit manually)
packages/               # Shared Python packages
  agent_framework/      #   BaseAgent, EnhancedAgent, AgentTeam
  multi_framework/      #   FrameworkOrchestrator
  integration_framework/#   BaseConnector, CredentialManager
services/
  orchestration_engine/ #   Central FastAPI app
web/                    #   React frontend
infrastructure/         #   K8s manifests, Docker Compose
tests/                  #   Unit and integration tests
```

### 4.3 Hands-On: Local Stack Startup

```bash
# 1. Clone and navigate
git clone <repo-url> && cd AI-Agents

# 2. Start infrastructure
docker compose up -d

# 3. Verify all services
docker compose ps
# Expected: postgres, redis, redpanda, opa (all "Up")

# 4. Test orchestration engine
curl http://localhost:8000/health
# Expected: {"status": "healthy"}

# 5. Test frontend
open http://localhost:3000
# Expected: Login page

# 6. Register and login
curl -X POST http://localhost:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "trainee", "password": "training123"}'

curl -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "trainee", "password": "training123"}'
# Save the returned token
```

---

## 5. Module 3: Agent Development

### 5.1 The Agent Development Lifecycle

```
Define (YAML) -> Generate (Python) -> Customise -> Test -> Deploy
```

### 5.2 Exercise: Create Your First Agent

**Task**: Create a "Meeting Notes Summariser" agent.

**Step 1**: Write the definition:
```yaml
# agents/definitions/operations/meeting_summariser.yaml
name: operations_meeting_summariser_1_1502
category: operations
description: "Summarises meeting notes into action items, decisions, and key points."
system_prompt: |
  You are a professional meeting notes summariser. Given raw meeting notes,
  extract and organise:
  1. Key Discussion Points (bullet list)
  2. Decisions Made (numbered list)
  3. Action Items (who, what, by when)
  4. Open Questions (items needing follow-up)

  Be concise and actionable. Use the attendees' names when assigning actions.
input_schema:
  type: object
  required: [meeting_notes]
  properties:
    meeting_notes:
      type: string
      description: "Raw meeting notes or transcript"
    attendees:
      type: array
      items: { type: string }
```

**Step 2**: Generate the agent:
```bash
python tools/generators/agent_generator_v2.py \
  --definition agents/definitions/operations/meeting_summariser.yaml
```

**Step 3**: Test locally:
```bash
cd generated-agents/operations_meeting_summariser_1_1502
docker build -t meeting-summariser:latest .
docker run -p 8200:8200 -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY meeting-summariser:latest
curl http://localhost:8200/health
```

**Step 4**: Execute with test input:
```bash
curl -X POST http://localhost:8200/api/v1/execute \
  -H "Content-Type: application/json" \
  -d '{
    "input": "Team sync Feb 18. Attendees: Alice, Bob, Carol. Alice presented Q1 results - revenue up 15%. Bob raised concern about server costs. Decision: migrate to Kubernetes by March. Carol to draft migration plan by Feb 25. Open: budget approval for new infrastructure.",
    "context": {"attendees": ["Alice", "Bob", "Carol"]}
  }'
```

### 5.3 Understanding the Generated Code

Open `generated-agents/operations_meeting_summariser_1_1502/app.py` and identify:
- FastAPI app initialisation
- Pydantic request/response models
- Prometheus metrics (Counter, Histogram)
- Anthropic client initialisation
- `/api/v1/execute` handler
- `/health` endpoint
- `/metrics` endpoint

### 5.4 Exercise: Customise an Agent

Modify the meeting summariser to add a "priority" field to action items:
1. Edit `app.py` to add post-processing logic
2. Parse the LLM output and add priority labels (High/Medium/Low)
3. Return structured JSON with prioritised action items
4. Test your changes

---

## 6. Module 4: Orchestration Engine

### 6.1 AgentManager Walkthrough

Read `services/orchestration_engine/orchestration_engine/agent_manager.py`:
- How are agent definitions loaded?
- How are execution requests routed to agent microservices?
- How does health checking work?

### 6.2 WorkflowManager Walkthrough

Read `services/orchestration_engine/orchestration_engine/workflow_manager.py`:
- How are workflows stored in PostgreSQL?
- How does Celery dispatch work?
- How are workflow results collected?

### 6.3 Exercise: Create a 3-Step Workflow via API

```bash
TOKEN="<your-jwt-token>"

# Create workflow
curl -X POST http://localhost:8000/workflows/create \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Training Exercise Workflow",
    "tasks": [
      {"agent": "operations_meeting_summariser_1_1502", "input": {"meeting_notes": "..."}},
      {"agent": "marketing_content_agent_1_101", "input": {"topic": "meeting highlights"}},
      {"agent": "marketing_email_agent_1_104", "input": {"content": "distribute summary"}}
    ]
  }'

# Execute the workflow
curl -X POST http://localhost:8000/workflows/<workflow-id>/execute \
  -H "Authorization: Bearer $TOKEN"

# Check status
curl http://localhost:8000/workflows/<workflow-id> \
  -H "Authorization: Bearer $TOKEN"
```

---

## 7. Module 5: Multi-Framework Patterns

### 7.1 When to Use Each Framework

| Framework | Use When | Example |
|-----------|----------|---------|
| Custom (Default) | Simple, single-agent tasks | Content generation |
| LangGraph | Complex workflows with branching logic | Approval processes with escalation |
| CrewAI | Tasks needing multiple perspectives | Document review by different specialists |
| AutoGen | Interactive, iterative refinement | Code review with back-and-forth feedback |

### 7.2 Exercise: LangGraph State Machine

Create a simple approval workflow using LangGraph:
1. Agent 1: Draft a document
2. Agent 2: Review the document
3. Conditional: If approved, proceed to Agent 3. If rejected, return to Agent 1.
4. Agent 3: Publish the document

---

## 8. Module 6: Infrastructure

### 8.1 Docker Compose Services

Examine `docker-compose.yml` and identify:
- Service definitions (orchestration-engine, postgres, redis, redpanda, opa)
- Port mappings
- Volume mounts
- Environment variables
- Health checks

### 8.2 Kubernetes Deployment

Examine `infrastructure/` and `k8s/` directories:
- Deployment manifests (replicas, containers, probes)
- Service definitions (ClusterIP, port mapping)
- HPA configurations (scaling thresholds)
- Network Policies (traffic rules)

### 8.3 Exercise: Deploy an Agent to Kubernetes

```bash
# Generate K8s manifests
python infrastructure/scripts/generate_k8s_manifests.py \
  --agent operations_meeting_summariser_1_1502 \
  --environment staging

# Review generated manifests
cat infrastructure/k8s/agents/operations_meeting_summariser_1_1502.yaml

# Apply (if you have a cluster)
kubectl apply -f infrastructure/k8s/agents/operations_meeting_summariser_1_1502.yaml
```

---

## 9. Module 7: Security

### 9.1 Authentication Flow Exercise

1. Register a user via API
2. Login and inspect the JWT token (use jwt.io to decode)
3. Identify the claims: sub, username, exp, iat
4. Make an authenticated API call
5. Observe what happens with an expired or invalid token

### 9.2 OPA Policy Exercise

Write a Rego policy that allows only users with the "marketing" role to execute marketing agents:

```rego
package agents

default allow = false

allow {
    input.action == "execute"
    startswith(input.agent, "marketing_")
    input.user.roles[_] == "marketing"
}
```

Test with OPA:
```bash
opa eval -d policy.rego -i input.json "data.agents.allow"
```

---

## 10. Module 8: Advanced Topics

### 10.1 Vector Memory (Qdrant)
- How agents store and retrieve conversation context
- Embedding generation and similarity search
- Memory retention policies

### 10.2 Custom Connectors
- Extending BaseConnector for new external systems
- Credential management via Vault
- Error handling and retry patterns

### 10.3 Performance Optimisation
- Response caching strategies
- Connection pooling tuning
- Agent cold start reduction
- LLM token usage optimisation

---

## 11. Assessment Checklist

After completing all modules, verify you can:

- [ ] Start and stop the local development stack
- [ ] Create an agent definition in YAML
- [ ] Generate an agent implementation from a definition
- [ ] Test an agent locally via Docker
- [ ] Execute an agent via the orchestration engine API
- [ ] Create and execute a multi-step workflow
- [ ] Read and understand orchestration engine code
- [ ] Write a basic OPA policy
- [ ] Use Grafana to view agent metrics
- [ ] Deploy an agent to Kubernetes (or describe the process)

---

## 12. Additional Resources

| Resource | Location |
|----------|----------|
| Architecture docs | `docs/architecture.md` |
| API reference | `docs/api-reference.md` |
| Agent catalogue | `docs/agent-catalog.md` |
| Safety guardrails | `docs/safety-guardrails.md` |
| Contributing guide | `docs/contributing.md` |
| CLAUDE.md (project context) | Repository root |
