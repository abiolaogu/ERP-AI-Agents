# Use Cases -- AI-Agents Platform
> Version: 1.0 | Last Updated: 2026-02-18 | Status: Draft
> Classification: Internal | Author: AIDD System

---

## 1. Introduction

This document catalogues the primary use cases for the AI-Agents Platform, organised by actor type. Each use case describes the goal, preconditions, main flow, alternative flows, and postconditions. The platform supports 1,500 agents across 29 categories; this document focuses on the platform-level use cases rather than individual agent capabilities.

---

## 2. Actors

| Actor | Description |
|-------|-------------|
| **Business User** | Non-technical user who discovers and executes agents via the Marketplace |
| **Developer** | Technical user who creates, customises, and integrates agents |
| **Administrator** | IT staff who manages users, policies, infrastructure, and monitoring |
| **System (Orchestration Engine)** | Automated system that dispatches workflows and manages agent lifecycle |
| **External System** | Third-party services that interact with agents via API (CRM, email, calendars) |

---

## 3. Use Case Catalogue

### UC-001: Browse Agent Marketplace

| Field | Detail |
|-------|--------|
| **Actor** | Business User |
| **Goal** | Discover available agents by browsing categories |
| **Precondition** | User is authenticated and on the Marketplace page |
| **Main Flow** | 1. User navigates to Agent Marketplace. 2. System displays 29 category tiles. 3. User selects a category (e.g., Marketing). 4. System displays agents in that category with name, description, and rating. 5. User scrolls through paginated results. |
| **Alternative Flow** | User uses search bar to find agents by keyword instead of browsing categories. |
| **Postcondition** | User has viewed available agents and their descriptions. |

### UC-002: Search for an Agent

| Field | Detail |
|-------|--------|
| **Actor** | Business User |
| **Goal** | Find a specific agent by keyword search |
| **Precondition** | User is authenticated |
| **Main Flow** | 1. User enters search term in the Marketplace search bar. 2. System queries agent definitions matching name, description, or tags. 3. System displays matching agents ranked by relevance. 4. User reviews results and selects an agent. |
| **Alternative Flow** | No results found: System displays "No agents found" with suggestions. |
| **Postcondition** | User has located the desired agent. |

### UC-003: Execute a Single Agent

| Field | Detail |
|-------|--------|
| **Actor** | Business User |
| **Goal** | Run an agent with specific input and receive results |
| **Precondition** | User is authenticated; agent is available and healthy |
| **Main Flow** | 1. User selects an agent from Marketplace or Dashboard. 2. System displays the agent's input form (based on Pydantic schema). 3. User provides input data. 4. User clicks "Execute". 5. System validates input, routes request to the agent microservice. 6. Agent invokes Claude 3.5 Sonnet with domain-specific prompt. 7. System returns structured results to the user. |
| **Alternative Flow** | a) Input validation fails: System displays field-level error messages. b) Agent is unavailable: System returns 503 with retry suggestion. c) LLM timeout: System returns 504 with option to retry. |
| **Postcondition** | User has received the agent's output. Execution is logged. |

### UC-004: Create a Multi-Step Workflow

| Field | Detail |
|-------|--------|
| **Actor** | Business User |
| **Goal** | Chain multiple agents into an automated workflow |
| **Precondition** | User is authenticated; at least 2 agents are available |
| **Main Flow** | 1. User navigates to Workflow Builder. 2. User provides a workflow name. 3. User adds steps, selecting an agent and defining input for each. 4. User maps outputs from previous steps to inputs of subsequent steps. 5. User saves the workflow. 6. System persists the workflow in PostgreSQL with status "pending". |
| **Alternative Flow** | User references a non-existent agent: System displays validation error. |
| **Postcondition** | Workflow is saved and ready for execution. |

### UC-005: Execute a Workflow

| Field | Detail |
|-------|--------|
| **Actor** | Business User |
| **Goal** | Run a previously created workflow |
| **Precondition** | Workflow exists with status "pending" or "completed" (re-run) |
| **Main Flow** | 1. User selects a workflow from the Dashboard. 2. User clicks "Execute". 3. System dispatches the workflow to Celery. 4. Celery worker executes each step sequentially. 5. System publishes events to Redpanda at each step. 6. User sees real-time status updates. 7. System marks workflow as "completed" when all steps finish. |
| **Alternative Flow** | Step N fails: System marks workflow as "failed", logs error, notifies user. |
| **Postcondition** | Workflow results are stored in PostgreSQL. Events are in Redpanda. |

### UC-006: Register a New Account

| Field | Detail |
|-------|--------|
| **Actor** | Business User / Developer |
| **Goal** | Create a platform account |
| **Precondition** | User does not have an existing account |
| **Main Flow** | 1. User navigates to /register. 2. User enters username and password. 3. System validates input (username uniqueness, password strength). 4. System hashes password with bcrypt and stores in PostgreSQL. 5. System returns success message. 6. User redirected to login page. |
| **Alternative Flow** | Username already exists: System returns 409 conflict error. |
| **Postcondition** | User account is created in the database. |

### UC-007: Authenticate and Obtain Token

| Field | Detail |
|-------|--------|
| **Actor** | Business User / Developer |
| **Goal** | Log in and receive a JWT access token |
| **Precondition** | User has a registered account |
| **Main Flow** | 1. User submits username and password to /auth/login. 2. System verifies credentials against PostgreSQL. 3. System generates JWT with user claims. 4. System returns JWT token. 5. Frontend stores token in AuthContext and localStorage. |
| **Alternative Flow** | Invalid credentials: System returns 401. |
| **Postcondition** | User has a valid JWT for subsequent API calls. |

### UC-008: Create a Custom Agent Definition

| Field | Detail |
|-------|--------|
| **Actor** | Developer |
| **Goal** | Define a new AI agent with custom capabilities |
| **Precondition** | Developer has access to the repository and YAML/JSON schema |
| **Main Flow** | 1. Developer creates a YAML/JSON file in agents/definitions/{category}/. 2. Developer specifies: name, description, category, system prompt, input schema, output schema, tools. 3. Developer runs agent_generator_v2.py to generate the FastAPI implementation. 4. Developer tests locally via Docker. 5. Developer submits PR for review. |
| **Alternative Flow** | Schema validation fails: Generator reports errors, developer fixes definition. |
| **Postcondition** | New agent is generated, tested, and ready for deployment. |

### UC-009: Deploy an Agent to Kubernetes

| Field | Detail |
|-------|--------|
| **Actor** | Administrator / Developer |
| **Goal** | Deploy an agent microservice to the production Kubernetes cluster |
| **Precondition** | Agent Docker image is built and pushed to registry |
| **Main Flow** | 1. Admin generates K8s manifests via generate_k8s_manifests.py. 2. Admin reviews Deployment, Service, and HPA manifests. 3. Admin applies manifests: kubectl apply -f. 4. Kubernetes creates pods and services. 5. Admin verifies agent health via /health endpoint. 6. Admin updates agent definition in orchestration engine. |
| **Alternative Flow** | Pod fails to start: Admin checks logs, fixes issues, re-deploys. |
| **Postcondition** | Agent is running in Kubernetes with health checks passing. |

### UC-010: Monitor Platform Health

| Field | Detail |
|-------|--------|
| **Actor** | Administrator |
| **Goal** | View platform health metrics and identify issues |
| **Precondition** | Monitoring stack (Prometheus, Grafana, Loki) is running |
| **Main Flow** | 1. Admin opens Grafana dashboard. 2. Admin reviews panels: agent request rates, error rates, latency percentiles. 3. Admin checks Loki for recent error logs. 4. Admin reviews AlertManager for active alerts. 5. Admin investigates any anomalies. |
| **Alternative Flow** | Alert fires: Admin follows runbook for the specific alert. |
| **Postcondition** | Admin has assessed platform health and taken action if needed. |

### UC-011: Manage OPA Policies

| Field | Detail |
|-------|--------|
| **Actor** | Administrator |
| **Goal** | Update access control policies for agent execution |
| **Precondition** | Admin has access to policies/ directory and OPA |
| **Main Flow** | 1. Admin edits Rego policy file in policies/. 2. Admin writes OPA unit tests. 3. Admin submits PR for security team review. 4. After approval, policy is deployed to OPA. 5. OPA evaluates new policy for subsequent requests. |
| **Alternative Flow** | Policy syntax error: OPA rejects the update, rollback to previous policy. |
| **Postcondition** | Updated access control policies are enforced. |

### UC-012: Integrate Agent with External System

| Field | Detail |
|-------|--------|
| **Actor** | Developer |
| **Goal** | Connect an agent to an external system (e.g., HubSpot CRM) |
| **Precondition** | Integration framework is available; external system credentials are in Vault |
| **Main Flow** | 1. Developer extends BaseConnector for the target system. 2. Developer implements authentication and API methods. 3. Developer registers credentials in Vault via CredentialManager. 4. Developer modifies agent definition to include the connector. 5. Agent uses connector during execution to read/write external data. |
| **Alternative Flow** | External API unavailable: Agent returns partial result with error details. |
| **Postcondition** | Agent can read from and write to the external system. |

### UC-013: View Analytics Dashboard

| Field | Detail |
|-------|--------|
| **Actor** | Business User / Administrator |
| **Goal** | Review platform usage statistics and performance metrics |
| **Precondition** | User is authenticated; analytics data is available |
| **Main Flow** | 1. User navigates to Analytics page. 2. System queries AnalyticsManager for aggregated metrics. 3. System displays: total executions, top agents, execution trends, error rates. 4. User filters by date range, category, or agent. 5. User exports data if needed. |
| **Alternative Flow** | No data for selected period: System displays empty state with message. |
| **Postcondition** | User has reviewed platform usage analytics. |

### UC-014: API-Based Agent Execution

| Field | Detail |
|-------|--------|
| **Actor** | External System |
| **Goal** | Programmatically execute an agent via API |
| **Precondition** | External system has a valid API key |
| **Main Flow** | 1. External system sends POST to /api/v1/agents/{id}/execute with API key header. 2. System validates API key. 3. System evaluates OPA policy for the API key's permissions. 4. System routes request to the target agent. 5. Agent executes and returns results. 6. System returns JSON response to external system. |
| **Alternative Flow** | Invalid API key: System returns 401. Policy denied: System returns 403. |
| **Postcondition** | External system has received agent execution results. |

---

## 4. Use Case Traceability Matrix

| Use Case | Business Requirement | Feature Requirement |
|----------|---------------------|-------------------|
| UC-001 | BR-002 Self-Service Discovery | FR-001, FR-002 |
| UC-002 | BR-002 Self-Service Discovery | FR-002 |
| UC-003 | BR-001 Agent Catalogue | FR-010, FR-011, FR-014 |
| UC-004 | BR-003 Workflow Automation | FR-020, FR-021 |
| UC-005 | BR-003 Workflow Automation | FR-022, FR-023, FR-024 |
| UC-006 | BR-004 Enterprise Security | FR-040 |
| UC-007 | BR-004 Enterprise Security | FR-040, FR-041 |
| UC-008 | BR-008 Developer Extensibility | -- |
| UC-009 | BR-006 Scalable Infrastructure | -- |
| UC-010 | BR-005 Operational Visibility | FR-052, FR-053, FR-054 |
| UC-011 | BR-004 Enterprise Security | FR-042 |
| UC-012 | BR-008 Developer Extensibility | -- |
| UC-013 | BR-005 Operational Visibility | FR-050, FR-051 |
| UC-014 | BR-008 Developer Extensibility | FR-041, FR-042 |

---

## 5. Use Case Diagram (Text Representation)

```
                    +---------------------------+
                    |    AI-Agents Platform      |
                    +---------------------------+
                    |                           |
  Business User --->| UC-001 Browse Marketplace |
  Business User --->| UC-002 Search Agents      |
  Business User --->| UC-003 Execute Agent      |
  Business User --->| UC-004 Create Workflow    |
  Business User --->| UC-005 Execute Workflow   |
  Business User --->| UC-006 Register Account   |
  Business User --->| UC-007 Authenticate       |
  Business User --->| UC-013 View Analytics     |
                    |                           |
  Developer ------->| UC-008 Create Agent Def   |
  Developer ------->| UC-009 Deploy Agent       |
  Developer ------->| UC-012 Integrate External |
                    |                           |
  Administrator --->| UC-010 Monitor Health     |
  Administrator --->| UC-011 Manage Policies    |
                    |                           |
  External System ->| UC-014 API Execution      |
                    +---------------------------+
```
