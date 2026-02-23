# Developer User Manual -- AI-Agents Platform
> Version: 1.0 | Last Updated: 2026-02-18 | Status: Draft
> Classification: Internal | Author: AIDD System

---

## 1. Introduction

This manual guides developers through building, customising, testing, and deploying AI agents on the AI-Agents Platform. It covers the development environment setup, agent creation workflow, API integration patterns, and best practices for extending the platform.

---

## 2. Development Environment Setup

### 2.1 Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| Python | 3.11+ | Backend development |
| Node.js | 18+ | Frontend development |
| Docker | 24.0+ | Containerisation |
| Docker Compose | 2.0+ | Local stack management |
| Git | 2.0+ | Version control |
| IDE | VS Code (recommended) | Development environment |

### 2.2 Repository Setup

```bash
# Clone the repository
git clone <repository-url> AI-Agents
cd AI-Agents

# Create Python virtual environment
python -m venv venv
source venv/bin/activate  # macOS/Linux
# venv\Scripts\activate   # Windows

# Install orchestration engine dependencies
cd services/orchestration_engine
pip install -r requirements.txt

# Install shared packages
pip install -e packages/agent_framework
pip install -e packages/multi_framework
pip install -e packages/integration_framework
pip install -e packages/vector_memory

# Install frontend dependencies
cd ../../web
npm install
```

### 2.3 Environment Configuration

Copy the example environment file and configure:

```bash
cp .env.example .env
```

Required variables:
```
ANTHROPIC_API_KEY=sk-ant-your-key-here
DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5432/ai_agents
REDIS_URL=redis://localhost:6379/0
OPA_URL=http://localhost:8181
SECRET_KEY=your-secret-key
JWT_SECRET=your-jwt-secret
```

### 2.4 Start Local Stack

```bash
# Start infrastructure services
docker compose up -d postgres redis redpanda opa

# Start orchestration engine (with hot-reload)
cd services/orchestration_engine
uvicorn orchestration_engine.main:app --reload --port 8000

# Start frontend dev server (separate terminal)
cd web
npm run dev
```

---

## 3. Creating a New Agent

### 3.1 Step 1: Write the Agent Definition

Create a YAML file in `agents/definitions/{category}/`:

```yaml
# agents/definitions/marketing/content_optimizer.yaml
name: marketing_content_optimizer_1_1501
category: marketing
description: "Analyses and optimises marketing content for engagement, readability, and SEO."
version: "1.0.0"
author: "Your Name"

system_prompt: |
  You are a marketing content optimisation specialist. Analyse the provided
  content and return specific, actionable improvements for:
  - Engagement: Hook strength, call-to-action effectiveness
  - Readability: Sentence structure, vocabulary level, flow
  - SEO: Keyword density, meta description, heading structure

  Return your analysis in a structured format with scores (1-10) and
  specific suggestions for each category.

input_schema:
  type: object
  required: [content]
  properties:
    content:
      type: string
      description: "The marketing content to optimise"
    target_audience:
      type: string
      description: "Intended audience for the content"
      default: "general"
    target_keywords:
      type: array
      items: { type: string }
      description: "SEO keywords to optimise for"

output_schema:
  type: object
  properties:
    engagement_score: { type: integer, minimum: 1, maximum: 10 }
    readability_score: { type: integer, minimum: 1, maximum: 10 }
    seo_score: { type: integer, minimum: 1, maximum: 10 }
    suggestions: { type: array, items: { type: string } }
    optimised_content: { type: string }

tags:
  - marketing
  - content
  - seo
  - optimisation
```

### 3.2 Step 2: Generate the Agent

```bash
python tools/generators/agent_generator_v2.py \
  --definition agents/definitions/marketing/content_optimizer.yaml \
  --output generated-agents/
```

This produces:
```
generated-agents/marketing_content_optimizer_1_1501/
  app.py
  Dockerfile
  requirements.txt
  README.md
```

### 3.3 Step 3: Customise the Agent (Optional)

For agents needing custom logic beyond the generated template, edit `app.py`:

```python
# Add custom pre-processing
@app.post("/api/v1/execute")
async def execute(request: ExecuteRequest):
    # Custom input transformation
    processed_input = preprocess(request.input)

    # Call LLM
    result = await call_claude(processed_input)

    # Custom post-processing
    structured_result = parse_scores(result)

    return ExecuteResponse(
        agent_name=AGENT_NAME,
        result=structured_result,
        execution_time=elapsed,
        metadata={"version": "1.0.0"}
    )
```

### 3.4 Step 4: Test Locally

```bash
# Build and run the agent
cd generated-agents/marketing_content_optimizer_1_1501/
docker build -t content-optimizer:latest .
docker run -p 8200:8200 -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY content-optimizer:latest

# Test the health endpoint
curl http://localhost:8200/health

# Test execution
curl -X POST http://localhost:8200/api/v1/execute \
  -H "Content-Type: application/json" \
  -d '{
    "input": "Check out our amazing new product that will revolutionize your workflow!",
    "context": {"target_audience": "developers", "target_keywords": ["workflow", "automation"]}
  }'
```

### 3.5 Step 5: Write Tests

Create tests in the `tests/` directory:

```python
# tests/test_content_optimizer.py
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_health():
    async with AsyncClient(base_url="http://localhost:8200") as client:
        response = await client.get("/health")
        assert response.status_code == 200
        assert response.json()["status"] == "healthy"

@pytest.mark.asyncio
async def test_execute():
    async with AsyncClient(base_url="http://localhost:8200") as client:
        response = await client.post("/api/v1/execute", json={
            "input": "Test content for optimization",
            "context": {"target_audience": "general"}
        })
        assert response.status_code == 200
        assert "result" in response.json()
```

Run tests:
```bash
cd tests && python -m pytest -v
```

---

## 4. Working with the Orchestration Engine API

### 4.1 Authentication

```python
import httpx

BASE_URL = "http://localhost:8000"

# Register
response = httpx.post(f"{BASE_URL}/auth/register", json={
    "username": "developer",
    "password": "secure_password"
})

# Login
response = httpx.post(f"{BASE_URL}/auth/login", json={
    "username": "developer",
    "password": "secure_password"
})
token = response.json()["access_token"]

# Use token for authenticated requests
headers = {"Authorization": f"Bearer {token}"}
```

### 4.2 Agent Execution via Orchestration Engine

```python
# List available agents
agents = httpx.get(f"{BASE_URL}/agents/list", headers=headers).json()

# Execute an agent
result = httpx.post(
    f"{BASE_URL}/agents/execute",
    headers=headers,
    json={
        "agent_id": "marketing_content_optimizer_1_1501",
        "input": {"content": "Your marketing text here..."}
    }
)
```

### 4.3 Workflow Creation and Execution

```python
# Create a workflow
workflow = httpx.post(
    f"{BASE_URL}/workflows/create",
    headers=headers,
    json={
        "name": "Content Pipeline",
        "tasks": [
            {"agent": "marketing_content_agent_1_101", "input": {"topic": "AI"}},
            {"agent": "marketing_content_optimizer_1_1501", "input": {}}
        ]
    }
).json()

# Execute the workflow
httpx.post(
    f"{BASE_URL}/workflows/{workflow['workflow_id']}/execute",
    headers=headers
)

# Check status
status = httpx.get(
    f"{BASE_URL}/workflows/{workflow['workflow_id']}",
    headers=headers
).json()
```

---

## 5. Using the Agent Framework

### 5.1 BaseAgent

```python
from agent_framework import BaseAgent

class MyAgent(BaseAgent):
    def __init__(self):
        super().__init__(
            name="my_custom_agent",
            description="A custom agent with special logic"
        )

    async def execute(self, input_data: dict) -> dict:
        # Your custom logic here
        result = await self.call_llm(
            system_prompt="You are a helpful assistant.",
            user_input=input_data.get("query", "")
        )
        return {"result": result}
```

### 5.2 EnhancedAgent (with Memory)

```python
from agent_framework import EnhancedAgent

class MemoryAgent(EnhancedAgent):
    def __init__(self):
        super().__init__(
            name="memory_agent",
            description="Agent with conversation memory",
            memory_enabled=True
        )

    async def execute(self, input_data: dict) -> dict:
        # Retrieve relevant context from vector memory
        context = await self.recall(input_data["query"])

        result = await self.call_llm(
            system_prompt=f"Context: {context}\n\nYou are a helpful assistant.",
            user_input=input_data["query"]
        )

        # Store interaction for future recall
        await self.memorise(input_data["query"], result)

        return {"result": result}
```

### 5.3 AgentTeam

```python
from agent_framework import AgentTeam

team = AgentTeam(
    name="content_team",
    agents=[writer_agent, editor_agent, seo_agent]
)

# Execute team workflow
result = await team.execute({
    "task": "Create an optimised blog post about AI automation",
    "coordination": "sequential"  # or "parallel", "delegated"
})
```

---

## 6. Multi-Framework Development

### 6.1 LangGraph Integration

```python
from multi_framework import FrameworkOrchestrator

orchestrator = FrameworkOrchestrator(framework="langgraph")

# Define a stateful graph
graph = orchestrator.create_graph(
    nodes=[
        {"name": "research", "agent": "research_agent_1_501"},
        {"name": "write", "agent": "content_agent_1_101"},
        {"name": "review", "agent": "review_agent_1_601"}
    ],
    edges=[
        ("research", "write"),
        ("write", "review"),
        ("review", "write", "needs_revision"),  # conditional edge
        ("review", "END", "approved")
    ]
)

result = await orchestrator.execute(graph, {"topic": "AI in healthcare"})
```

### 6.2 CrewAI Integration

```python
orchestrator = FrameworkOrchestrator(framework="crewai")

crew = orchestrator.create_crew(
    agents=[
        {"role": "researcher", "agent": "research_agent_1_501"},
        {"role": "writer", "agent": "content_agent_1_101"},
        {"role": "editor", "agent": "review_agent_1_601"}
    ],
    task="Create a comprehensive report on AI automation trends"
)

result = await orchestrator.execute(crew)
```

---

## 7. Building External Integrations

### 7.1 Creating a Connector

```python
from integration_framework import BaseConnector, CredentialManager

class SlackConnector(BaseConnector):
    def __init__(self):
        super().__init__(name="slack")
        self.cred_manager = CredentialManager(vault_path="secret/integrations/slack")

    async def connect(self):
        creds = await self.cred_manager.get_credentials()
        self.client = SlackClient(token=creds["bot_token"])

    async def send_message(self, channel: str, message: str):
        return await self.client.chat_postMessage(channel=channel, text=message)

    async def disconnect(self):
        self.client = None
```

---

## 8. Testing Guide

### 8.1 Unit Tests
```bash
cd tests && python -m pytest test_agent_manager.py -v
```

### 8.2 Integration Tests
```bash
# Requires running stack
cd tests && python -m pytest test_e2e.py -v
```

### 8.3 Load Tests
```bash
cd testing && locust -f locustfile.py --host http://localhost:8000
```

---

## 9. CI/CD Pipeline

### 9.1 PR Workflow
On every pull request, GitHub Actions runs:
1. **Lint**: Python linting (flake8/ruff) and TypeScript linting (eslint)
2. **Type Check**: mypy for Python, tsc for TypeScript
3. **Unit Tests**: pytest with coverage report
4. **Security Scan**: dependency vulnerability scanning
5. **Docker Build**: Verify image builds successfully

### 9.2 Deployment Workflow
On merge to `main`:
1. Build Docker images for changed services
2. Push to container registry
3. Deploy to staging environment
4. Run integration tests against staging
5. Promote to production (manual approval or tagged release)

---

## 10. Common Patterns and Best Practices

1. **Always validate inputs**: Use Pydantic models for all request/response schemas
2. **Never hardcode secrets**: Read from environment variables or Vault
3. **Use async/await**: All I/O operations should be async
4. **Add Prometheus metrics**: Every agent should export request count and latency
5. **Write tests first**: Create tests before or alongside agent implementation
6. **Follow naming conventions**: `{category}_{agent}_{number}_{id}` for agent names
7. **Document your agents**: Include clear descriptions in agent definitions
8. **Handle errors gracefully**: Return structured error responses with error codes
9. **Use type hints**: All Python code should have type annotations
10. **Keep agents focused**: Each agent should do one thing well
