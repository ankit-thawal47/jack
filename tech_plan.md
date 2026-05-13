# Jack Technical Plan

## Summary

Jack is a multi-tenant D2C AI Workforce OS. It is a modular monolith: one codebase with separately runnable API, worker, FastMCP, agent, and frontend processes.

The v0 uses Python for backend, agents, and MCP; TypeScript React for the Ops Cockpit; Postgres with RLS for tenant isolation; seeded SaaS connectors; Anthropic Claude models through Claude Agent SDK for multi-agent orchestration; and FastMCP as the agent tool boundary.

The system must satisfy the five build requirements:

- 3+ proper connectors behind one abstraction
- universal normalized data model with provenance
- chat layer with tool use and cited numerical claims
- at least one autonomous AI employee with run logs and reasoning
- scalability layer for 10,000 merchants

## Architecture Goals

- Ship a working v0 that proves Jack as a D2C AI workforce operating layer.
- Keep the implementation modular enough to split into services later.
- Make tenant isolation a database-enforced property through Postgres RLS.
- Make agent behavior debuggable through run logs, tool logs, and citations.
- Keep calculations deterministic in backend services; use Anthropic Claude for synthesis, ranking, and explanation.
- Treat Postgres plus deterministic metrics as the source of numerical truth. Anthropic models interpret trusted facts; they do not invent metrics.
- Avoid real external writes in v0. Agents create proposed recommendations only.

## System Architecture

```text
Seeded SaaS Connectors
  Shopify / Meta Ads / Support / Finance
        ↓
Sync + Normalize Pipeline
        ↓
Postgres
  source_records + normalized D2C schema + metrics
        ↓
FastMCP Server
  safe tools over trusted data
        ↓
Anthropic Claude Models
        ↓
Claude Agent SDK
  coordinator + specialist agents
        ↓
Postgres
  agent runs + tool calls + recommendations + citations
        ↓
FastAPI Backend
        ↓
React Ops Cockpit
  Action Queue + Chat + Agent Runs
```

Runtime shape:

- `api`: FastAPI HTTP server for the frontend and external app calls.
- `worker`: background sync and scheduled agent runs.
- `mcp`: FastMCP server exposing agent tools.
- `agent`: Claude Agent SDK runner using Anthropic Claude models for coordinator and specialist agents.
- `frontend`: TypeScript React Ops Cockpit.
- `postgres`: source of truth.
- `redis`: lightweight queue/cache for local worker orchestration.

Local development should use Docker Compose for Postgres, Redis, backend API, worker, MCP server, agent runner, and frontend.

## Module Boundaries

### API

The API module owns HTTP routes for:

- tenant bootstrap and current merchant context
- connector sync triggers and sync status
- recommendations and action queue lifecycle
- chat sessions and messages
- agent run history
- citation/source row lookup

### Database

The DB module owns:

- SQLAlchemy or SQLModel models
- Alembic migrations
- tenant context helpers
- RLS policy helpers
- repository/session utilities

Every request, job, MCP tool call, and agent run must execute with an explicit tenant context.

### Connectors

The connectors module owns:

- shared connector protocol
- Shopify connector
- Meta Ads connector
- Support connector shaped like Gorgias or Zendesk
- Finance connector shaped like QuickBooks or Xero

Connectors are ingestion adapters only. They are not the agent tool layer.

### Sync

The sync module owns:

- connector execution
- sync cursors
- raw source record persistence
- normalization into universal D2C tables
- sync run status and errors

Raw source records should be written before normalized rows so provenance is always retained.

### Metrics

The metrics module owns deterministic calculations:

- sales velocity
- days of inventory cover
- stockout risk
- ad waste
- contribution margin
- refund rate
- support complaint rate
- support quality signals

Anthropic Claude should not calculate core business metrics from scratch.

### MCP

The MCP module owns one FastMCP server with namespaced tools over normalized data, metrics, recommendations, and citations.

MCP tools must be tenant-scoped and log tool calls.

### Agents

The agents module owns Claude Agent SDK definitions backed by Anthropic Claude models:

- Coordinator Agent
- Inventory Agent
- Ad Waste Agent
- Margin Agent
- Support Quality Agent

Agents use MCP tools. They do not query raw SaaS APIs directly.

### Recommendations

The recommendations module owns:

- action queue records
- recommendation status transitions
- expected impact calculations
- approval/dismissal metadata

V0 status values:

- `proposed`
- `approved`
- `dismissed`
- `needs_review`

### Citations

The citations module owns:

- source-row citation creation
- metric-observation citation creation
- numerical-claim validation
- citation rendering for chat and recommendations

Uncited numerical claims must be blocked or rewritten before user display.

### Observability

The observability module owns:

- structured logs
- agent run logs
- agent run steps
- tool call logs
- optional OpenTelemetry export

The product-facing Agent Runs UI reads from Postgres audit tables.

### Frontend

The frontend module owns the Ops Cockpit:

- Action Queue
- Chat
- Agent Runs

The UI should be simple, operational, and focused on evidence-backed actions.

## Data Model

Use Postgres as the v0 database.

Internally use **tenant**. In UI and product copy use **merchant**.

Core tables:

- `tenants`
- `tenant_users`
- `connector_accounts`
- `sync_runs`
- `source_records`
- `products`
- `skus`
- `orders`
- `order_lines`
- `inventory_snapshots`
- `ad_campaigns`
- `ad_daily_metrics`
- `support_tickets`
- `refunds`
- `finance_entries`
- `metric_observations`
- `agent_runs`
- `agent_run_steps`
- `tool_calls`
- `recommendations`
- `recommendation_citations`

Nearly every business table must include:

```text
tenant_id uuid not null references tenants(id)
```

Normalized rows should retain provenance through:

```text
source_record_id uuid references source_records(id)
source text
external_id text
synced_at timestamptz
```

`source_records` should store:

```text
id
tenant_id
connector_account_id
source
entity_type
external_id
raw_payload jsonb
fetched_at
synced_at
content_hash
```

Tenant-scoped unique constraints:

```text
(tenant_id, source, external_id)
(tenant_id, source, entity_type, external_id)
```

Important indexes:

```text
(tenant_id, created_at)
(tenant_id, source, external_id)
(tenant_id, sku_id, observed_at)
(tenant_id, agent_name, started_at)
(tenant_id, status, created_at)
```

## RLS Design

Tenant isolation is enforced with Postgres Row Level Security.

Application code should still scope queries by tenant, but RLS is the hard boundary.

Example policy shape:

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_orders
ON orders
USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

Every API request, sync job, MCP tool call, and agent run should set tenant context inside a transaction:

```sql
SET LOCAL app.current_tenant_id = '<tenant-id>';
```

Testing must prove tenant A cannot read tenant B data even if application code forgets a tenant filter.

## Connector Design

Implement four seeded/API-shaped connectors behind one shared abstraction.

Shared protocol:

```text
Connector
  source_name
  sync(tenant_id, since)
  list_records(tenant_id, entity_type, since)
  normalize(raw_record)
```

### Shopify

Owns commerce truth:

- products
- SKUs/variants
- orders
- order lines
- inventory
- refunds

### Meta Ads

Owns acquisition and spend:

- campaigns
- ad sets or ads if needed
- daily spend
- attributed conversions/revenue if seeded

### Support

Modeled as Gorgias or Zendesk:

- support tickets
- complaint categories
- SKU/product references
- refund/return reasons
- priority or sentiment if seeded

### Finance

Modeled as QuickBooks or Xero:

- COGS
- fees
- expenses
- product-level or order-level finance entries

Each connector writes raw records first, then normalized records. Source-specific mapping stays in connectors. Cross-source business logic stays in metrics.

## FastMCP Tool Design

Use one FastMCP server for v0.

Tools:

- `get_sku_metrics`
- `find_inventory_risks`
- `find_ad_waste`
- `find_margin_leaks`
- `find_support_quality_issues`
- `create_recommendation`
- `list_recommendations`
- `get_citations`
- `get_source_rows`

Tool requirements:

- require tenant context
- require or create run context when called by agents
- return structured data, not prose
- include citation IDs or source row IDs for numerical outputs
- log inputs, output summary, source row IDs, duration, status, and errors

## Agent Design

Use Claude Agent SDK with Anthropic Claude models for orchestration.

Agents:

- **Coordinator Agent**: ranks, dedupes, and resolves conflicts between specialist findings.
- **Inventory Agent**: watches stock, sales velocity, days of cover, stockout risk, and overstock risk.
- **Ad Waste Agent**: watches campaign spend against inventory, refunds, contribution margin, and support signals.
- **Margin Agent**: watches revenue, COGS, refunds, fees, ad spend, and contribution margin.
- **Support Quality Agent**: watches ticket volume, complaint themes, refund reasons, and SKU-level quality signals.

Agent rules:

- agents use MCP tools only
- agents do not call raw SaaS APIs
- agents do not perform external writes in v0
- agents create proposed recommendations
- every recommendation must include evidence, citations, confidence, expected impact, and caveats
- every run must persist trigger, steps, tools used, rows inspected, calculations, final output, and failures

## Chat And Citation Contract

Chat uses Anthropic Claude through Claude Agent SDK with FastMCP tools.

Rules:

- deterministic metrics are computed in backend code
- Anthropic Claude explains, synthesizes, and ranks
- numerical claims require citations
- citations must resolve to metric observations or source rows
- uncited numerical claims are blocked or rewritten before display
- writes create internal recommendations only

Citation examples:

```text
"SKU-RED-01 has 6.2 days of cover [cit:metric_observations:abc123]."
"Campaign Prospecting-01 spent Rs. 42,500 this week [cit:ad_daily_metrics:def456]."
```

## Observability And Debugging

Use three levels of observability.

### Product Run Logs

Required Postgres audit tables:

- `agent_runs`
- `agent_run_steps`
- `tool_calls`
- `recommendations`
- `recommendation_citations`

These power the Agent Runs UI.

### MCP Tool Logging

Every MCP tool logs:

- tool name
- tenant ID
- agent name
- run ID
- input JSON
- output summary
- source row IDs
- duration
- error

### OpenTelemetry

If time permits, enable Claude Agent SDK OpenTelemetry export to Langfuse, Jaeger, Grafana Tempo, or another OTEL-compatible backend.

Do not depend on hidden model reasoning traces. Store observable reasoning artifacts: plan, tool calls, calculations, evidence, recommendation, confidence, and caveats.

## Scalability Plan

Jack should work for one merchant today and have a path to 10,000 tenants.

Built now:

- shared Postgres database with RLS
- tenant-scoped indexes
- tenant-scoped unique constraints
- async worker for sync and agent runs
- incremental sync cursors
- source record retention for replay/backfill
- deterministic metric observations
- FastMCP tool boundary for safe agent data access
- action queue instead of direct external writes
- agent/tool audit logs

What breaks first:

- SaaS API rate limits
- OAuth refresh failures
- sync scheduling across many tenants
- high-volume order and ad metric tables
- agent run cost and latency
- noisy or duplicate recommendations
- weak support-ticket-to-SKU matching

Future mitigations:

- per-source and per-tenant rate limits
- queue partitioning by tenant/source
- table partitioning by time/source
- materialized metric tables
- model tiering and deterministic prefilters
- per-tenant agent schedules
- warehouse or OLAP layer for larger merchants
- human approval workflows for real external writes

## Testing And Acceptance Criteria

Connector tests:

- all four connectors implement the shared abstraction
- raw source records are persisted
- normalized rows preserve provenance

RLS tests:

- tenant A cannot read tenant B rows
- cross-tenant reads fail even without explicit tenant filters
- missing tenant context fails closed

Metric tests:

- inventory risk calculations are deterministic
- ad waste calculations combine spend, stock, refund, and margin signals
- margin leak calculations include COGS, refunds, fees, and ad spend
- support quality calculations tie complaints/refunds to SKUs

MCP tests:

- tools are tenant-scoped
- tools return structured outputs
- numerical outputs include citation IDs or source row IDs
- tool calls are logged

Agent tests:

- at least one autonomous run creates a recommendation
- run logs include trigger, tool calls, evidence, confidence, and caveats
- recommendation has citations
- agents do not perform external writes

Chat tests:

- cited numerical claims pass validation
- uncited numerical claims are blocked or rewritten
- chat can answer cross-tool questions from seeded data

Frontend smoke tests:

- Action Queue renders seeded recommendations
- Chat renders cited answers
- Agent Runs renders run steps and tool calls

## Build Order

1. Create backend project structure and Docker Compose.
2. Add Postgres migrations for tenants, source records, normalized D2C schema, recommendations, citations, agent runs, and tool calls.
3. Add RLS helpers and tenant isolation tests.
4. Implement seeded connector abstraction and four connectors.
5. Implement sync pipeline and seed data load.
6. Implement deterministic metric services.
7. Implement FastMCP tools over metrics, recommendations, citations, and source rows.
8. Implement Anthropic Claude agents through Claude Agent SDK and coordinator.
9. Implement citation validator for chat and recommendations.
10. Implement FastAPI routes for cockpit data, chat, sync, and agent runs.
11. Implement React Ops Cockpit.
12. Write README sections for architecture, connector choices, schema, chat citations, agents, scale, eval honesty, hours, and next week.

## Assumptions

- Product name is Jack.
- Internal language uses tenant; UI/product language uses merchant.
- V0 uses seeded/API-shaped connectors, not live OAuth integrations.
- V0 uses Postgres, not SQLite.
- V0 uses Docker Compose for local development.
- V0 uses one FastMCP server with namespaced tools.
- V0 uses Anthropic Claude models through Claude Agent SDK for agent orchestration.
- V0 uses a modular monolith first, with service split points documented.
- Agents propose actions only; no real external action execution.
