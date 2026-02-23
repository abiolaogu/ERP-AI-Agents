# Workflows Guide -- AI-Agents Platform
> Version: 1.0 | Last Updated: 2026-02-18 | Status: Draft
> Classification: Internal | Author: AIDD System

---

## 1. Overview

Workflows are the core mechanism for composing multiple AI agents into automated multi-step business processes. The AI-Agents Platform's `WorkflowManager` enables users to define, execute, and monitor sequences of agent invocations where each step's output can feed into the next step's input.

---

## 2. Workflow Concepts

### 2.1 Terminology

| Term | Definition |
|------|-----------|
| **Workflow** | A named sequence of tasks that execute agents in a defined order |
| **Task** | A single step within a workflow, representing one agent execution |
| **Step Input** | The JSON payload sent to the agent for a given task |
| **Step Output** | The JSON response from the agent after execution |
| **Workflow Status** | One of: `pending`, `running`, `completed`, `failed` |
| **Dispatch** | The act of sending a workflow's tasks to Celery for execution |

### 2.2 Data Model

```sql
CREATE TABLE workflows (
    id          VARCHAR(36) PRIMARY KEY,   -- UUID
    name        VARCHAR(255) NOT NULL,
    tasks       TEXT NOT NULL,             -- JSON array of task definitions
    status      VARCHAR(50) DEFAULT 'pending',
    results     TEXT DEFAULT '[]',         -- JSON array of step results
    user_id     INTEGER REFERENCES users(id),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 3. Workflow Lifecycle

### 3.1 State Machine

```
  [Create]
     |
     v
  PENDING ----[Execute]----> RUNNING ----[All steps pass]----> COMPLETED
                               |
                               +----[Any step fails]--------> FAILED
```

### 3.2 Lifecycle Stages

**Stage 1: Definition**
- User creates a workflow via `POST /workflows/create`
- Provides a name and a list of tasks (agent names and inputs)
- System assigns a UUID and stores the workflow in PostgreSQL

**Stage 2: Dispatch**
- User triggers execution via `POST /workflows/{id}/execute`
- WorkflowManager sends each task to a Celery worker
- Redpanda event published: `workflow.started`

**Stage 3: Execution**
- Celery workers process tasks sequentially
- Each task invokes the target agent's `/api/v1/execute` endpoint
- Agent responses are collected and stored in the workflow's `results` field
- Redpanda events published per step: `agent.executed`

**Stage 4: Completion**
- All tasks completed successfully: status set to `completed`
- Any task fails: status set to `failed`, remaining tasks skipped
- Redpanda event published: `workflow.completed` or `workflow.failed`

---

## 4. API Reference

### 4.1 Create Workflow

```http
POST /workflows/create
Authorization: Bearer <jwt_token>
Content-Type: application/json

{
  "name": "Content Marketing Pipeline",
  "tasks": [
    {
      "agent": "marketing_content_agent_1_101",
      "input": {"topic": "AI in healthcare", "format": "blog_post"}
    },
    {
      "agent": "marketing_seo_agent_1_102",
      "input": {"content": "{{previous_output}}", "target_keywords": ["AI healthcare"]}
    },
    {
      "agent": "marketing_social_media_agent_1_103",
      "input": {"content": "{{previous_output}}", "platforms": ["twitter", "linkedin"]}
    }
  ]
}
```

**Response (201)**:
```json
{
  "workflow_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "Content Marketing Pipeline",
  "status": "pending",
  "task_count": 3
}
```

### 4.2 Execute Workflow

```http
POST /workflows/a1b2c3d4-e5f6-7890-abcd-ef1234567890/execute
Authorization: Bearer <jwt_token>
```

**Response (202)**:
```json
{
  "workflow_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "running",
  "message": "Workflow execution started"
}
```

### 4.3 Get Workflow Status

```http
GET /workflows/a1b2c3d4-e5f6-7890-abcd-ef1234567890
Authorization: Bearer <jwt_token>
```

**Response (200)**:
```json
{
  "workflow_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "Content Marketing Pipeline",
  "status": "completed",
  "tasks": [
    {"agent": "marketing_content_agent_1_101", "status": "completed"},
    {"agent": "marketing_seo_agent_1_102", "status": "completed"},
    {"agent": "marketing_social_media_agent_1_103", "status": "completed"}
  ],
  "results": [
    {"step": 1, "output": "..."},
    {"step": 2, "output": "..."},
    {"step": 3, "output": "..."}
  ]
}
```

### 4.4 List Workflows

```http
GET /workflows/list
Authorization: Bearer <jwt_token>
```

---

## 5. Workflow Patterns

### 5.1 Sequential Pipeline
Tasks execute one after another. Output from step N is available as input to step N+1.

```
Agent A --> Agent B --> Agent C --> Result
```

**Use Case**: Content creation pipeline (draft, optimise, distribute).

### 5.2 Fan-Out / Fan-In (Planned)
A single input is processed by multiple agents in parallel, and results are aggregated.

```
            +--> Agent B --+
Agent A --> +--> Agent C --+--> Agent D (Aggregator)
            +--> Agent D --+
```

**Use Case**: Multi-perspective analysis (financial, legal, technical review of a document).

### 5.3 Conditional Branching (Planned)
Workflow paths depend on the output of a previous step.

```
Agent A --> [Condition] --> Agent B (if true)
                       --> Agent C (if false)
```

**Use Case**: Lead qualification (high-value leads get a different treatment than low-value).

### 5.4 Iterative Loop (Planned)
A step repeats until a condition is met.

```
Agent A --> [Check Quality] --> Agent A (if below threshold)
                            --> Next Step (if above threshold)
```

**Use Case**: Content refinement (revise until quality score > 0.8).

---

## 6. Example Workflows

### 6.1 Sales Lead Processing

| Step | Agent | Action |
|------|-------|--------|
| 1 | `sales_lead_scoring_agent_1_201` | Score incoming lead based on firmographic data |
| 2 | `sales_crm_agent_1_202` | Create or update CRM record with lead score |
| 3 | `sales_proposal_agent_1_203` | Generate personalised outreach proposal |
| 4 | `marketing_email_agent_1_104` | Draft and schedule follow-up email |

### 6.2 IT Incident Response

| Step | Agent | Action |
|------|-------|--------|
| 1 | `it_support_triage_agent_1_301` | Classify incident severity and category |
| 2 | `it_monitoring_agent_1_302` | Pull relevant metrics and logs |
| 3 | `it_diagnosis_agent_1_303` | Analyse data and suggest root cause |
| 4 | `it_remediation_agent_1_304` | Generate remediation playbook |

### 6.3 Legal Contract Review

| Step | Agent | Action |
|------|-------|--------|
| 1 | `legal_contract_extraction_agent_1_401` | Extract key terms and clauses |
| 2 | `legal_risk_assessment_agent_1_402` | Identify risk areas and flag concerns |
| 3 | `legal_compliance_agent_1_403` | Check against regulatory requirements |
| 4 | `legal_summary_agent_1_404` | Generate executive summary with recommendations |

---

## 7. Workflow Engine Internals

### 7.1 Celery Task Dispatch

```python
# Simplified workflow execution flow
@celery_app.task
def execute_workflow_task(workflow_id: str):
    workflow = get_workflow(workflow_id)
    update_status(workflow_id, "running")

    results = []
    for task in workflow.tasks:
        try:
            response = execute_agent(task.agent, task.input)
            results.append({"step": len(results) + 1, "output": response})
        except Exception as e:
            update_status(workflow_id, "failed")
            publish_event("workflow.failed", workflow_id)
            raise

    update_results(workflow_id, results)
    update_status(workflow_id, "completed")
    publish_event("workflow.completed", workflow_id)
```

### 7.2 Redpanda Event Schema

```json
{
  "event_type": "workflow.completed",
  "workflow_id": "a1b2c3d4-...",
  "user_id": 42,
  "timestamp": "2026-02-18T10:30:00Z",
  "metadata": {
    "task_count": 4,
    "duration_seconds": 12.5,
    "agents_used": ["agent_1", "agent_2", "agent_3", "agent_4"]
  }
}
```

---

## 8. Error Handling

| Error Scenario | Behaviour | Recovery |
|----------------|-----------|----------|
| Agent unavailable | Task fails, workflow marked `failed` | Retry workflow after agent recovery |
| LLM API timeout | Task fails with timeout error | Automatic retry (configurable) |
| Invalid agent input | Task fails with validation error | Fix input and re-execute workflow |
| Celery worker crash | Task remains in queue | Worker restart picks up pending tasks |
| Redpanda unavailable | Events not published (non-blocking) | Events replayed after recovery |

---

## 9. Monitoring Workflows

### 9.1 Metrics
- `workflow_executions_total` -- Counter of workflow executions by status
- `workflow_duration_seconds` -- Histogram of workflow execution times
- `workflow_step_duration_seconds` -- Histogram of individual step times
- `workflow_active_count` -- Gauge of currently running workflows

### 9.2 Grafana Dashboard
- Workflow throughput over time
- Success/failure rate trends
- Average workflow duration by workflow name
- Top failing agents across workflows

### 9.3 Alerts
- `WorkflowFailureRateHigh` -- > 10% failure rate over 15 minutes
- `WorkflowDurationAnomaly` -- P99 latency > 2x baseline
- `WorkflowQueueBacklog` -- Pending workflows > 50 for > 5 minutes

---

## 10. Best Practices

1. **Keep workflows focused**: Each workflow should accomplish a single business objective with 2-6 steps.
2. **Design for failure**: Assume any step can fail. Use descriptive names so failures are easy to diagnose.
3. **Validate inputs**: Ensure each step's expected input format matches the previous step's output format.
4. **Monitor execution times**: Set up alerts for workflows that exceed expected duration.
5. **Use meaningful names**: Workflow names should describe the business process, not the technical implementation.
6. **Test incrementally**: Test each agent independently before composing into a workflow.
7. **Document data flow**: For each workflow, document what data flows between steps and what transformations occur.
