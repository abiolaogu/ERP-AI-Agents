# Low-Level Design (LLD) -- AI-Agents Platform
> Version: 1.0 | Last Updated: 2026-02-18 | Status: Draft
> Classification: Internal | Author: AIDD System

---

## 1. Purpose

This Low-Level Design document details the internal structure of each component in the AI-Agents Platform, including class diagrams, method signatures, data structures, algorithms, and database access patterns. It serves as the primary reference for developers implementing or modifying the system.

---

## 2. Orchestration Engine -- Detailed Design

### 2.1 Module Structure

```
services/orchestration_engine/orchestration_engine/
  main.py              # FastAPI app initialisation, route registration
  agent_manager.py     # Agent loading, routing, execution
  workflow_manager.py  # Workflow CRUD and execution
  analytics_manager.py # Metrics aggregation and reporting
  auth.py              # JWT and API key authentication
  database.py          # SQLAlchemy models and async engine
  schemas.py           # Pydantic request/response schemas
  celery_worker.py     # Celery app configuration and tasks
  redpanda_manager.py  # Kafka producer/consumer for Redpanda
```

### 2.2 AgentManager Class

```python
class AgentManager:
    """
    Manages agent lifecycle: loading definitions, routing execution
    requests, and monitoring agent health.
    """

    def __init__(self, definitions_path: str):
        self.agents: Dict[str, AgentDefinition] = {}
        self.definitions_path = definitions_path

    def load_agents(self) -> int:
        """
        Scan definitions_path for .json files and load into memory.
        Returns: Number of agents loaded.
        Note: Currently only loads JSON; YAML support planned.
        """

    def get_agent(self, agent_id: str) -> Optional[AgentDefinition]:
        """Return agent definition by ID or None if not found."""

    def list_agents(
        self, category: Optional[str] = None,
        search: Optional[str] = None,
        limit: int = 50, offset: int = 0
    ) -> List[AgentDefinition]:
        """List agents with optional category filter and search."""

    async def execute_agent(
        self, agent_id: str, input_data: dict
    ) -> AgentResponse:
        """
        Route execution request to the target agent microservice.
        1. Resolve agent's host:port from definition
        2. POST input_data to /api/v1/execute
        3. Return structured AgentResponse
        """

    async def check_health(self, agent_id: str) -> HealthStatus:
        """GET /health on the target agent and return status."""
```

### 2.3 WorkflowManager Class

```python
class WorkflowManager:
    """
    Manages workflow CRUD operations and orchestrates multi-step
    agent executions via Celery.
    """

    def __init__(self, db_session: AsyncSession, agent_manager: AgentManager):
        self.db = db_session
        self.agent_manager = agent_manager

    async def create_workflow(
        self, name: str, tasks: List[TaskDefinition], user_id: int
    ) -> Workflow:
        """
        Persist a new workflow in PostgreSQL.
        1. Generate UUID for workflow_id
        2. Serialise tasks to JSON
        3. INSERT into workflows table with status='pending'
        4. Return Workflow object
        """

    async def execute_workflow(self, workflow_id: str) -> None:
        """
        Dispatch workflow to Celery for async execution.
        1. Fetch workflow from DB
        2. Validate all referenced agents exist
        3. Update status to 'running'
        4. Submit Celery task: execute_workflow_task.delay(workflow_id)
        5. Publish 'workflow.started' event to Redpanda
        """

    async def get_workflow(self, workflow_id: str) -> Optional[Workflow]:
        """Fetch workflow by ID from PostgreSQL."""

    async def list_workflows(self, user_id: int) -> List[Workflow]:
        """List all workflows for a user, ordered by created_at desc."""

    async def update_status(
        self, workflow_id: str, status: str, results: Optional[List] = None
    ) -> None:
        """Update workflow status and optionally append results."""
```

### 2.4 Auth Module

```python
# JWT Configuration
JWT_SECRET: str          # From environment variable
JWT_ALGORITHM: str = "HS256"
JWT_EXPIRY_MINUTES: int = 60

def create_access_token(user_id: int, username: str) -> str:
    """
    Create JWT with claims: sub=user_id, username, exp, iat.
    Returns: Encoded JWT string.
    """

def verify_token(token: str) -> TokenPayload:
    """
    Decode and validate JWT.
    1. Decode with JWT_SECRET and HS256
    2. Check expiry
    3. Check Redis blacklist
    4. Return TokenPayload(user_id, username)
    Raises: HTTPException 401 if invalid/expired/blacklisted.
    """

def hash_password(password: str) -> str:
    """bcrypt hash with salt rounds=12."""

def verify_password(password: str, password_hash: str) -> bool:
    """bcrypt verify."""
```

---

## 3. Agent Microservice -- Detailed Design

### 3.1 Standard Agent Structure (app.py)

```python
from fastapi import FastAPI
from pydantic import BaseModel
from anthropic import Anthropic
from prometheus_client import Counter, Histogram, generate_latest

app = FastAPI(title="{agent_name}")

# Prometheus metrics
REQUEST_COUNT = Counter(
    "agent_requests_total",
    "Total requests",
    ["agent_name", "status"]
)
PROCESSING_TIME = Histogram(
    "agent_processing_seconds",
    "Processing time in seconds",
    ["agent_name"]
)

# Anthropic client
client = Anthropic(api_key=os.environ.get("ANTHROPIC_API_KEY"))

class ExecuteRequest(BaseModel):
    input: str
    context: Optional[dict] = None

class ExecuteResponse(BaseModel):
    agent_name: str
    result: str
    execution_time: float
    metadata: dict

@app.post("/api/v1/execute")
async def execute(request: ExecuteRequest) -> ExecuteResponse:
    """
    1. Validate input via Pydantic
    2. Construct system prompt from agent definition
    3. Call Anthropic Claude API with system prompt + user input
    4. Extract and structure response
    5. Record Prometheus metrics
    6. Return ExecuteResponse
    """

@app.get("/health")
async def health():
    return {"status": "healthy", "agent": AGENT_NAME}

@app.get("/metrics")
async def metrics():
    return Response(generate_latest(), media_type="text/plain")
```

### 3.2 LLM Call Pattern

```python
async def call_llm(system_prompt: str, user_input: str) -> str:
    """
    Invoke Anthropic Claude 3.5 Sonnet.

    Parameters:
        system_prompt: Domain-specific instructions for the agent
        user_input: User's request/query

    Returns:
        str: Claude's response text

    Implementation:
        1. Construct messages array: [{"role": "user", "content": user_input}]
        2. Call client.messages.create(
               model="claude-3-5-sonnet-20241022",
               max_tokens=4096,
               system=system_prompt,
               messages=messages
           )
        3. Extract response.content[0].text
        4. Apply output filtering (safety guardrails)
        5. Return filtered text
    """
```

### 3.3 Port Assignment Algorithm

```python
def calculate_port(agent_id_number: int) -> int:
    """
    Deterministic port assignment for agent microservices.
    Range: 8200-9000 (800 ports)

    Formula: 8200 + (agent_id_number % 800)

    Note: In production (Kubernetes), ports are abstracted
    by Services and the formula applies to container ports only.
    """
    return 8200 + (agent_id_number % 800)
```

---

## 4. Data Access Layer

### 4.1 SQLAlchemy Models

```python
from sqlalchemy import Column, Integer, String, DateTime, ForeignKey
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, autoincrement=True)
    username = Column(String(255), unique=True, nullable=False)
    password_hash = Column(String(255), nullable=False)
    created_at = Column(DateTime, server_default=func.now())

class Workflow(Base):
    __tablename__ = "workflows"
    id = Column(String(36), primary_key=True)  # UUID
    name = Column(String(255), nullable=False)
    tasks = Column(Text, nullable=False)        # JSON serialised
    status = Column(String(50), default="pending")
    results = Column(Text, default="[]")        # JSON serialised
    user_id = Column(Integer, ForeignKey("users.id"))
    created_at = Column(DateTime, server_default=func.now())
```

### 4.2 Database Connection

```python
DATABASE_URL = os.environ.get("DATABASE_URL")
engine = create_async_engine(DATABASE_URL, pool_size=20, max_overflow=10)

async def get_session() -> AsyncSession:
    async with AsyncSession(engine) as session:
        yield session

async def init_db():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
```

### 4.3 Query Patterns

| Operation | Query | Index Used |
|-----------|-------|------------|
| User lookup by username | `SELECT * FROM users WHERE username = ?` | `users_username_key` (unique) |
| Workflow by ID | `SELECT * FROM workflows WHERE id = ?` | Primary key |
| User's workflows | `SELECT * FROM workflows WHERE user_id = ? ORDER BY created_at DESC` | `ix_workflows_user_id` (recommended) |
| Running workflows | `SELECT * FROM workflows WHERE status = 'running'` | Sequential scan (add index) |

---

## 5. Celery Task Definitions

```python
from celery import Celery

celery_app = Celery(
    "ai_agents",
    broker=os.environ.get("REDIS_URL", "redis://localhost:6379/0"),
    backend=os.environ.get("REDIS_URL", "redis://localhost:6379/0")
)

celery_app.conf.update(
    task_serializer="json",
    result_serializer="json",
    accept_content=["json"],
    timezone="UTC",
    task_track_started=True,
    task_acks_late=True,
    worker_prefetch_multiplier=1
)

@celery_app.task(bind=True, max_retries=3, default_retry_delay=30)
def execute_workflow_task(self, workflow_id: str):
    """
    Execute all steps in a workflow sequentially.

    Retry policy: Up to 3 retries with 30-second delay.
    On permanent failure: Mark workflow as 'failed'.
    """
```

---

## 6. Redpanda Event Schema

### 6.1 Event Envelope

```python
@dataclass
class Event:
    event_id: str          # UUID
    event_type: str        # e.g., "agent.executed"
    timestamp: datetime    # UTC ISO 8601
    source: str            # Service name
    data: dict             # Event-specific payload
    metadata: dict         # Tracing, correlation IDs
```

### 6.2 Event Types

| Event Type | Producer | Data Fields |
|-----------|----------|-------------|
| `agent.executed` | Agent via Orchestration | agent_id, user_id, duration, status |
| `workflow.started` | WorkflowManager | workflow_id, user_id, task_count |
| `workflow.completed` | Celery Worker | workflow_id, duration, results_summary |
| `workflow.failed` | Celery Worker | workflow_id, failed_step, error_message |
| `user.registered` | Auth Module | user_id, username |
| `user.logged_in` | Auth Module | user_id, ip_address |

---

## 7. Frontend Component Design

### 7.1 React Component Hierarchy

```
App
  AuthProvider (context)
    Layout
      Header (navigation)
      Routes
        /login       -> LoginPage
        /register    -> RegisterPage
        /marketplace -> AgentMarketplace
        /dashboard   -> Dashboard
        /analytics   -> AnalyticsPage
```

### 7.2 State Management

| State | Scope | Storage | Update Trigger |
|-------|-------|---------|---------------|
| Auth token | Global | AuthContext + localStorage | Login/logout |
| Agent list | Page | AgentMarketplace local state | API fetch on mount |
| Search filters | Page | AgentMarketplace local state | User interaction |
| Workflow status | Page | Dashboard local state | Polling / WebSocket |
| Analytics data | Page | AnalyticsPage local state | API fetch on mount |

### 7.3 API Service Layer

```typescript
// services/authService.ts
const API_BASE_URL = import.meta.env.VITE_API_URL || "http://localhost:8000";

export const authService = {
  login: (username: string, password: string) => Promise<AuthResponse>,
  register: (username: string, password: string) => Promise<void>,
  logout: () => void,
};

// services/agentService.ts
export const agentService = {
  listAgents: (category?: string, search?: string) => Promise<Agent[]>,
  executeAgent: (agentId: string, input: object) => Promise<AgentResponse>,
  getAgentHealth: (agentId: string) => Promise<HealthStatus>,
};

// services/workflowService.ts
export const workflowService = {
  createWorkflow: (name: string, tasks: Task[]) => Promise<Workflow>,
  executeWorkflow: (workflowId: string) => Promise<void>,
  getWorkflowStatus: (workflowId: string) => Promise<Workflow>,
  listWorkflows: () => Promise<Workflow[]>,
};
```

---

## 8. Error Handling Design

### 8.1 Error Code Registry

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `AGENT_NOT_FOUND` | 404 | Requested agent ID does not exist |
| `AGENT_UNAVAILABLE` | 503 | Agent service is not responding |
| `AGENT_EXECUTION_FAILED` | 500 | Agent returned an error during execution |
| `WORKFLOW_NOT_FOUND` | 404 | Requested workflow ID does not exist |
| `WORKFLOW_ALREADY_RUNNING` | 409 | Workflow is already in `running` state |
| `AUTH_INVALID_CREDENTIALS` | 401 | Username or password incorrect |
| `AUTH_TOKEN_EXPIRED` | 401 | JWT token has expired |
| `AUTH_TOKEN_BLACKLISTED` | 401 | JWT token has been revoked |
| `POLICY_DENIED` | 403 | OPA policy denied the request |
| `VALIDATION_ERROR` | 422 | Request body failed Pydantic validation |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests from this user/IP |
| `LLM_API_ERROR` | 502 | Anthropic API returned an error |
| `LLM_API_TIMEOUT` | 504 | Anthropic API call timed out |

### 8.2 Error Response Format

```json
{
  "error": {
    "code": "AGENT_NOT_FOUND",
    "message": "Agent 'marketing_content_agent_1_999' not found",
    "details": {},
    "request_id": "req_abc123",
    "timestamp": "2026-02-18T10:30:00Z"
  }
}
```

---

## 9. Security Implementation Details

### 9.1 Password Storage
- Algorithm: bcrypt
- Salt rounds: 12
- Library: `passlib[bcrypt]`

### 9.2 JWT Token Structure

```json
{
  "sub": 42,
  "username": "john.doe",
  "iat": 1708243800,
  "exp": 1708247400,
  "jti": "unique-token-id"
}
```

### 9.3 OPA Policy Evaluation

```python
async def evaluate_policy(user: TokenPayload, agent_id: str, action: str) -> bool:
    """
    Call OPA at http://opa:8181/v1/data/agents/allow
    Input: {"user": user_dict, "agent": agent_id, "action": action}
    Returns: True if policy allows, False otherwise.
    """
```

---

## 10. Performance Considerations

| Component | Bottleneck | Optimisation |
|-----------|-----------|-------------|
| Agent execution | LLM API latency (1-5s) | Response caching, streaming responses |
| Workflow execution | Sequential step execution | Planned: parallel step execution |
| Agent loading | 1,500 definitions at startup | Lazy loading, in-memory cache |
| Database queries | Workflow status polling | Add indexes, consider WebSocket push |
| Frontend rendering | Large agent list (1,500) | Virtual scrolling, pagination |
