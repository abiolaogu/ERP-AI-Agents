# Figma & Make Design Prompts -- AI-Agents Platform
> Version: 1.0 | Last Updated: 2026-02-18 | Status: Draft
> Classification: Internal | Author: AIDD System

---

## 1. Purpose

This document provides structured prompts for generating UI/UX designs in Figma and automation workflows in Make (formerly Integromat) for the AI-Agents Platform. Designers and automation engineers can use these prompts to rapidly prototype interfaces and build no-code integrations.

---

## 2. Figma Design Prompts

### 2.1 Agent Marketplace -- Main Page

**Prompt**:
```
Design a modern, enterprise-grade AI agent marketplace page with the following elements:

Layout:
- Top navigation bar with logo "AI-Agents", search bar (centered, prominent),
  and user avatar/menu (right-aligned)
- Left sidebar with 29 category filters (collapsible, with icons per category):
  Marketing, Sales, Finance, HR, Legal, Healthcare, IT/DevOps, Operations,
  Customer Service, Data Analytics, and 19 more
- Main content area: Grid of agent cards (3 columns on desktop, responsive)
- Each agent card shows: Agent name, category badge, 2-line description,
  "Execute" button (primary), "Details" button (secondary)
- Pagination or infinite scroll at bottom

Style:
- Color palette: Deep navy (#1a1a2e) primary, electric blue (#4361ee) accent,
  white (#ffffff) cards, light grey (#f8f9fa) background
- Typography: Inter or Satoshi font family, clean and professional
- Border radius: 12px for cards, 8px for buttons
- Subtle shadows on cards (elevation: 2)
- Dark mode variant also required

Responsive breakpoints: 1440px, 1024px, 768px, 375px
```

### 2.2 Agent Marketplace -- Agent Detail Page

**Prompt**:
```
Design an agent detail page for an AI agent marketplace with these sections:

Header:
- Breadcrumb: Marketplace > Marketing > Content Agent
- Agent name (large, bold), category badge, version number
- Status indicator (green dot = healthy, red = unavailable)
- "Execute Agent" primary CTA button (large)

Content sections:
1. Description: Full agent description (2-3 paragraphs) with capability tags
2. Input Schema: Interactive form showing required and optional input fields,
   with field types, descriptions, and example values
3. Output Schema: Visual representation of what the agent returns
4. Usage Examples: 2-3 example input/output pairs in code blocks
5. Performance: Average response time, success rate, total executions (mini charts)
6. Related Agents: Horizontal scrollable row of similar agent cards

Style: Consistent with marketplace main page. Clean whitespace, clear hierarchy.
```

### 2.3 Dashboard Page

**Prompt**:
```
Design a user dashboard for an AI agent platform:

Top section:
- Welcome message with user name
- Quick stats row: Total Executions (today), Active Workflows, Favourite Agents, Success Rate
- Each stat in a card with icon, number, and trend arrow (up/down)

Main content (2-column layout):
Left column (60%):
- "Recent Executions" table: Agent name, Input summary, Status (badge), Time, Duration
- Pagination, 10 rows per page
- Click row to expand and see full result

Right column (40%):
- "Active Workflows" list: Workflow name, status (running/pending/completed),
  progress bar showing completed steps / total steps
- "Quick Execute" widget: Dropdown to select a favourite agent, input field, Run button

Bottom section:
- "Recommended Agents" carousel based on user's usage patterns

Style: Dashboard aesthetic with data visualisation. Use chart.js or recharts style for
mini-charts. Consistent with marketplace colour palette.
```

### 2.4 Workflow Builder

**Prompt**:
```
Design a visual workflow builder for chaining AI agents:

Canvas area (center, 70% width):
- Drag-and-drop canvas with grid background
- Workflow steps as connected cards (node-graph style)
- Each step card shows: Step number, Agent name, Status icon, Mini input preview
- Connection lines between steps with directional arrows
- "+" button between steps to insert new step
- Start node (green) and End node (red)

Right panel (30% width):
- When a step is selected: Full configuration form
  - Agent selector (searchable dropdown)
  - Input configuration (form fields based on agent's input schema)
  - Data mapping: "Use output from Step N" toggle per field
  - Advanced: Retry count, Timeout, Framework selector

Top bar:
- Workflow name (editable inline)
- Save button, Execute button, Delete button
- Status badge (Draft / Saved / Running / Completed)
- Execution history dropdown

Bottom bar:
- Minimap of the full workflow
- Zoom controls (+/-/fit)

Style: Clean, technical aesthetic similar to n8n or Make.com. Light background,
coloured step cards based on agent category.
```

### 2.5 Analytics Page

**Prompt**:
```
Design an analytics dashboard for an AI agent platform:

Filter bar (top):
- Date range picker (Last 7 days, 30 days, 90 days, Custom)
- Category filter dropdown (All, Marketing, Sales, etc.)
- Agent filter (searchable dropdown)

Metrics row:
- 4 metric cards: Total Executions, Average Latency, Success Rate, Active Users
- Each with number, percentage change, and sparkline chart

Charts section (2x2 grid):
1. Line chart: "Executions Over Time" (daily, with area fill)
2. Bar chart: "Top 10 Agents by Usage" (horizontal bars)
3. Pie/Donut chart: "Executions by Category" (29 categories, top 10 shown)
4. Heatmap: "Usage by Day of Week and Hour" (GitHub contribution style)

Table section:
- Detailed execution log: Timestamp, User, Agent, Category, Status, Duration, Tokens Used
- Sortable columns, searchable, exportable to CSV

Style: Data-rich but not cluttered. Use Recharts or D3-style visualisations.
Consistent colour coding per category throughout.
```

### 2.6 Login and Registration Pages

**Prompt**:
```
Design login and registration pages for an enterprise AI platform:

Layout:
- Split screen: Left 50% illustration/branding, Right 50% form
- Left side: Abstract AI/network illustration with brand colours,
  company logo, tagline "1,500 AI Agents. One Platform."
- Right side: Centered form card on light background

Login form:
- "Welcome Back" heading
- Username input field (with user icon)
- Password input field (with eye toggle for visibility)
- "Remember me" checkbox
- "Login" primary button (full width)
- "Don't have an account? Register" link below

Registration form:
- "Create Account" heading
- Username input field
- Password input field (with strength indicator bar)
- Confirm password field
- "Create Account" primary button
- "Already have an account? Login" link

Style: Enterprise professional. Subtle animations on form focus.
Error states: Red border with inline error message below field.
Loading state: Button shows spinner during API call.
```

### 2.7 Mobile Responsive Design

**Prompt**:
```
Design mobile-responsive variants (375px width) for the AI-Agents Platform:

1. Marketplace: Single column card layout, hamburger menu for categories,
   sticky search bar at top
2. Agent Detail: Stacked sections, full-width execute button (sticky bottom)
3. Dashboard: Stacked stat cards (2x2), scrollable execution list
4. Login/Register: Full-screen form (no split layout), logo above form
5. Bottom navigation bar: Home, Marketplace, Workflows, Analytics, Profile

Ensure touch targets are minimum 44x44px. Use bottom sheet modals for
filters and dropdowns instead of dropdown menus.
```

---

## 3. Make (Integromat) Automation Prompts

### 3.1 Agent Execution Monitoring Workflow

**Prompt**:
```
Create a Make.com scenario that monitors AI agent execution failures:

Trigger: Webhook (receives events from Redpanda consumer)
Filter: event_type == "agent.executed" AND status == "error"

Actions:
1. Parse JSON payload: Extract agent_name, error_message, user_id, timestamp
2. Log to Google Sheets: Append row to "Agent Error Log" spreadsheet
3. Send Slack notification to #agent-alerts channel:
   "Agent {agent_name} failed at {timestamp}. Error: {error_message}"
4. If error_count for this agent > 5 in last hour:
   Send PagerDuty incident: "Agent {agent_name} has high failure rate"
5. Send email summary to platform-ops@company.com

Schedule: Real-time (webhook-driven)
```

### 3.2 Daily Analytics Report Workflow

**Prompt**:
```
Create a Make.com scenario that generates a daily analytics report:

Trigger: Schedule - Daily at 8:00 AM UTC

Actions:
1. HTTP Request: GET /analytics/daily from orchestration engine API
   Headers: Authorization: Bearer {api_key}
2. Parse response: Extract total_executions, top_agents, error_rate, avg_latency
3. Generate formatted report using Text Aggregator:
   "AI-Agents Daily Report - {date}
    Total Executions: {total}
    Top Agent: {top_agent_name} ({top_agent_count} runs)
    Error Rate: {error_rate}%
    Avg Latency: {avg_latency}ms"
4. Send to Slack #daily-metrics channel
5. Send as email to stakeholders distribution list
6. If error_rate > 5%: Flag and send to #urgent-alerts

Error handling: Retry 3 times with 60-second delays. On final failure,
send error notification to platform-ops.
```

### 3.3 New User Onboarding Workflow

**Prompt**:
```
Create a Make.com scenario for automated user onboarding:

Trigger: Webhook (receives user.registered event)

Actions:
1. Parse JSON: Extract username, user_id, registration_timestamp
2. Wait 5 minutes (allow account setup)
3. Send welcome email via SendGrid:
   Subject: "Welcome to AI-Agents Platform"
   Body: HTML template with:
   - Getting started guide link
   - Top 5 recommended agents for their department
   - Link to training manual
   - Support contact information
4. Create Slack DM to user (if Slack integration enabled):
   "Welcome! Here are 3 agents to try first..."
5. Log onboarding event to CRM (HubSpot)
6. After 3 days: Send follow-up email with usage tips
7. After 7 days: Check if user has executed any agents
   If no: Send re-engagement email with specific use cases

Error handling: Log failures to error tracking system.
```

### 3.4 Workflow Completion Notification

**Prompt**:
```
Create a Make.com scenario that notifies users when workflows complete:

Trigger: Webhook (receives workflow.completed or workflow.failed events)

Actions for workflow.completed:
1. Parse JSON: Extract workflow_id, workflow_name, user_id, results_summary
2. Lookup user email from user_id
3. Send email: "Your workflow '{workflow_name}' completed successfully"
   Include: Summary of results, link to full results in dashboard
4. Post to user's Slack DM (if configured)

Actions for workflow.failed:
1. Parse JSON: Extract workflow_id, workflow_name, user_id, failed_step, error
2. Lookup user email
3. Send email: "Your workflow '{workflow_name}' failed at step {failed_step}"
   Include: Error details, troubleshooting suggestions, retry link
4. Post to #workflow-alerts Slack channel
5. If workflow has failed 3+ times: Create Jira ticket for investigation

Error handling: Retry 2 times. Log all notification failures.
```

### 3.5 Agent Health Monitoring Workflow

**Prompt**:
```
Create a Make.com scenario that performs periodic health checks on agents:

Trigger: Schedule - Every 5 minutes

Actions:
1. HTTP Request: GET /agents/list from orchestration engine (get all agent endpoints)
2. Iterator: For each agent in the list
3. HTTP Request: GET /health on each agent's endpoint
   Timeout: 5 seconds
4. Filter: Status != "healthy" OR request failed
5. For unhealthy agents:
   a. Log to Google Sheets: Agent name, status, timestamp, error
   b. If agent was healthy last check and now unhealthy:
      Send Slack alert: "Agent {name} is now UNHEALTHY"
   c. If agent has been unhealthy for 3+ consecutive checks:
      Send PagerDuty incident
6. Aggregate results: Count healthy vs unhealthy
7. Update status dashboard (Google Sheets or custom endpoint)

Rate limiting: Max 50 health checks per minute to avoid overloading.
```

---

## 4. Design System Tokens

For consistency across Figma designs, use these tokens:

### 4.1 Colours

| Token | Value | Usage |
|-------|-------|-------|
| `--color-primary` | #1a1a2e | Navigation, headers |
| `--color-accent` | #4361ee | Buttons, links, highlights |
| `--color-success` | #2ecc71 | Success states, healthy status |
| `--color-warning` | #f39c12 | Warning states, pending |
| `--color-error` | #e74c3c | Error states, failures |
| `--color-bg-primary` | #ffffff | Card backgrounds |
| `--color-bg-secondary` | #f8f9fa | Page background |
| `--color-text-primary` | #212529 | Body text |
| `--color-text-secondary` | #6c757d | Secondary text |

### 4.2 Typography

| Token | Value |
|-------|-------|
| `--font-family` | Inter, system-ui, sans-serif |
| `--font-size-h1` | 32px / 2rem |
| `--font-size-h2` | 24px / 1.5rem |
| `--font-size-h3` | 20px / 1.25rem |
| `--font-size-body` | 16px / 1rem |
| `--font-size-small` | 14px / 0.875rem |
| `--font-size-caption` | 12px / 0.75rem |

### 4.3 Spacing

| Token | Value |
|-------|-------|
| `--space-xs` | 4px |
| `--space-sm` | 8px |
| `--space-md` | 16px |
| `--space-lg` | 24px |
| `--space-xl` | 32px |
| `--space-2xl` | 48px |

### 4.4 Component Tokens

| Token | Value |
|-------|-------|
| `--border-radius-sm` | 4px |
| `--border-radius-md` | 8px |
| `--border-radius-lg` | 12px |
| `--border-radius-full` | 9999px |
| `--shadow-sm` | 0 1px 3px rgba(0,0,0,0.1) |
| `--shadow-md` | 0 4px 6px rgba(0,0,0,0.1) |
| `--shadow-lg` | 0 10px 15px rgba(0,0,0,0.1) |

---

## 5. Icon Requirements

The platform requires icons for:

| Category | Icons Needed |
|----------|-------------|
| Navigation | Home, Search, Marketplace, Dashboard, Analytics, Settings, Logout |
| Agent categories | 29 category icons (Marketing, Sales, Finance, HR, Legal, etc.) |
| Status | Healthy, Unhealthy, Pending, Running, Completed, Failed |
| Actions | Execute, Edit, Delete, Copy, Download, Share, Refresh |
| Data types | Text, Number, Array, Object, Boolean, Date |

Recommended icon libraries: Lucide, Heroicons, or Phosphor (consistent line weight).
