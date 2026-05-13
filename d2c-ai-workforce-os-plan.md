# Jack: D2C AI Workforce OS Plan

## Product Thesis

Jack is a multi-agent operating layer for D2C brands.

D2C founders and operators run their business across many SaaS tools. Cross-tool questions often require stitching exports across commerce, ads, support, inventory, and finance. The product should act like a team of AI employees that monitor this operating layer, reason over normalized data, and propose concrete ops-saving or profit-saving actions with citations back to source rows.

The v0 should be an **Ops Cockpit**: dashboard, chat, and agent run logs as first-class surfaces.

## Product Shape

Jack is not a generic chatbot and not a collection of disconnected agents. It is a shared data and action platform with specialist AI employees on top.

Core components:

- Seeded SaaS connectors for D2C operating data
- Universal D2C data model
- Row-level provenance and citations
- FastMCP server exposing safe tools over normalized data
- Claude Agent SDK multi-agent system
- Multi-tenant Postgres database with RLS
- Ops Cockpit frontend with action queue, chat, and run logs

## User Experience

The v0 UX should be an **Ops Cockpit** with three main sections.

### Action Queue

Prioritized recommendations from agents:

- action
- expected rupee impact
- agent/source
- confidence
- status: proposed, approved, dismissed, needs review
- citations

### Chat

Operators can ask cross-tool questions such as:

- Why is profit down this week?
- Which SKUs should I reorder?
- Where is ad spend being wasted?
- Show products with rising complaints and falling margin.

The chat layer must use tools over normalized data. Every numerical claim must carry citations back to source rows. Uncited numbers should not survive to the user.

### Agent Runs

Transparent run logs:

- trigger
- agent
- tools used
- rows inspected
- calculations performed
- reasoning summary
- recommendation created
- failure modes and caveats

## Connectors

The v0 should implement four connectors behind one shared abstraction.

### Shopify Connector

Commerce source of truth:

- orders
- order lines
- products
- variants/SKUs
- inventory
- refunds

### Meta Ads Connector

Acquisition and spend signal:

- campaigns
- ad sets or ads if needed
- daily spend
- attributed revenue/conversions if seeded

### Support Connector

Customer pain and quality signal. This can be modeled as Gorgias or Zendesk:

- tickets
- complaint categories
- product/SKU references
- refund/return reasons
- sentiment or priority if seeded

### Finance Connector

Profitability signal. This can be modeled as QuickBooks or Xero:

- COGS
- fees
- expenses
- product-level or order-level finance entries

### Connector Abstraction

Connectors should be backend ingestion adapters, not agent tools directly.

Suggested interface:

```text
Connector
  source_name
  sync(tenant_id, since)
  list_records(tenant_id, entity_type, since)
  normalize(raw_record)
```

Each connector should produce raw source records and normalized records. The raw payload should be retained for provenance and auditability.

## Data Architecture

Use **Postgres** for the v0.

The database should be multi-tenant. Internally, use the word **tenant**. In product/UI copy, use **merchant**.

Core tables:

- tenants
- tenant_users
- connector_accounts
- source_records
- products
- skus or variants
- orders
- order_lines
- inventory_snapshots
- ad_campaigns
- ad_daily_metrics
- support_tickets
- refunds
- finance_entries
- metric_observations
- agent_runs
- agent_run_steps
- tool_calls
- recommendations
- recommendation_citations

Nearly every business table should include:

```text
tenant_id uuid not null references tenants(id)
```

Every normalized row should retain provenance back to its source record:

```text
source_record_id
source
external_id
raw_payload
synced_at
```

## Multi-Tenancy and RLS

Tenant isolation should be enforced with Postgres Row Level Security.

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

Important indexes:

- `(tenant_id, created_at)`
- `(tenant_id, source, external_id)`
- `(tenant_id, sku_id, observed_at)`
- `(tenant_id, agent_name, started_at)`

Unique constraints should be tenant-scoped, for example:

```text
(tenant_id, source, external_id)
```

## Agents and MCP

Use **Claude Agent SDK** as the agent runtime/orchestrator.

Use **FastMCP** to expose controlled tools to agents.

Do not make agents responsible for raw SaaS ingestion. Agents should query normalized data and metric tools.

Architecture:

```text
Claude Agent SDK
  Coordinator Agent
  Inventory Agent
  Ad Waste Agent
  Margin Agent
  Support Quality Agent
        ↓ uses
FastMCP Server
  get_sku_metrics
  find_inventory_risks
  find_ad_waste
  find_margin_leaks
  find_support_quality_issues
  create_recommendation
  get_citations
        ↓ calls
Backend Services
  metric engine
  recommendation service
  citation service
  normalized repositories
        ↓ reads/writes
Postgres
```

Separately:

```text
Seeded SaaS Connectors
  ShopifyConnector
  MetaAdsConnector
  SupportConnector
  FinanceConnector
        ↓
Sync + Normalize Pipeline
        ↓
Postgres
```

## Multi-Agent System

The system should include specialist agents plus a coordinator.

### Inventory Agent

Watches:

- stock on hand
- sales velocity
- days of cover
- stockout risk
- overstock risk

Proposes:

- reorder action
- discount action
- hold/pause growth action if inventory is constrained

### Ad Waste Agent

Watches:

- ad spend
- campaign performance
- inventory constraints
- refund/support issues
- contribution margin

Proposes:

- pause or reduce spend
- investigate poor-performing campaigns
- avoid scaling campaigns linked to weak margin or low stock

### Margin Agent

Watches:

- revenue
- COGS
- ad spend
- refunds
- fees
- contribution margin

Proposes:

- pricing review
- margin investigation
- ad spend adjustment
- product profitability review

### Support Quality Agent

Watches:

- support ticket volume
- product complaint themes
- refund reasons
- SKU-level quality signals

Proposes:

- quality investigation
- stop scaling problematic SKUs
- update product page or support SOP

### Coordinator Agent

Combines findings into one ranked action queue.

Responsibilities:

- dedupe recommendations
- resolve conflicts between agents
- rank by expected rupee impact and urgency
- attach citations and caveats
- produce operator-facing summary

## Chat Layer

The chat layer should use Claude Agent SDK with tool use through FastMCP.

Rules:

- Calculations should be deterministic backend code.
- LLMs should explain, rank, and synthesize.
- Numerical claims must cite source rows or metric observations.
- A citation validator should block or strip uncited numerical claims.
- Writes should create internal recommendations or action proposals only.
- No external sends, refunds, campaign changes, or purchase orders in v0.

## Observability and Debugging

Use three levels of observability.

### Claude Agent SDK Telemetry

Claude Agent SDK supports OpenTelemetry export for traces, metrics, and logs.

Useful signals:

- agent interaction spans
- model request latency
- token usage
- tool calls
- tool execution failures
- nested subagent traces
- session IDs

For dev/demo, route telemetry to Langfuse, Grafana Tempo, Jaeger, or another OTEL-compatible backend if time permits.

### Product-Level Run Logs

Store product-facing audit logs in Postgres:

- agent_runs
- agent_run_steps
- tool_calls
- recommendations
- recommendation_citations
- agent_evaluations

This is what the Ops Cockpit should display.

### FastMCP Tool Logging

Every MCP tool should log:

- tool_name
- tenant_id
- agent_name
- run_id
- input_json
- output_summary
- source_row_ids
- duration_ms
- error

This helps debug whether an incorrect recommendation came from bad data, bad deterministic calculations, or bad model interpretation.

## Scalability Layer

The v0 should work for one merchant but be designed for 10,000 tenants.

Built-in scale choices:

- tenant-scoped Postgres schema
- RLS tenant isolation
- tenant-scoped indexes
- unique constraints scoped by tenant
- async sync jobs
- source records retained for replay/backfills
- normalized tables for deterministic metrics
- agent run logs for auditability
- FastMCP tool boundary for safe agent access
- action queue instead of direct external writes

What breaks first at scale:

- SaaS API rate limits and OAuth refresh failures
- ingestion job scheduling across many tenants
- agent run cost and latency
- metric query performance on large order/ad tables
- noisy recommendations without ranking/deduping
- support ticket/product matching quality

Future mitigations:

- job queue with per-source and per-tenant rate limits
- incremental sync cursors
- table partitioning by time/source for high-volume data
- materialized metric tables
- per-tenant agent schedules
- model tiering and deterministic prefilters
- approval workflows for external writes
- warehouse or OLAP layer for larger merchants

## Requirement Mapping

### Requirement 1: At least 3 proper connectors

Planned answer: 4 connectors.

- Shopify
- Meta Ads
- Support: Gorgias/Zendesk shaped
- Finance: QuickBooks/Xero shaped

All behind one shared connector abstraction.

### Requirement 2: Universal data model with provenance

Planned answer: normalized D2C schema in Postgres with source_records and source_record_id references on normalized rows.

Every row should be traceable to:

- source system
- external id
- raw payload
- sync timestamp
- connector account
- tenant

### Requirement 3: Chat layer with citations

Planned answer: Claude Agent SDK chat using FastMCP tools over normalized data.

Citation contract:

- numerical claims require citations
- citations point to source rows or metric observations
- uncited numbers are blocked or removed

### Requirement 4: Autonomous agent

Planned answer: multi-agent system.

At minimum, one full autonomous workflow should run end-to-end and create recommendations with logs. The target system includes Inventory, Ad Waste, Margin, Support Quality, and Coordinator agents.

Agents propose actions only. They do not send messages, change ad budgets, create purchase orders, or issue refunds in v0.

### Requirement 5: Scalability layer

Planned answer: Postgres multi-tenant architecture with RLS, async ingestion, scoped indexes, tool/run logs, and README discussion of what breaks first at 10,000 merchants.

## Recommended Implementation Stack

Backend and data:

- Python
- FastAPI
- Postgres
- SQLAlchemy or SQLModel
- Alembic

Agents and tools:

- Claude Agent SDK
- FastMCP
- OpenTelemetry if time permits

Frontend:

- TypeScript React, likely Vite or Next.js
- Simple Ops Cockpit UI

## Build Priority

Smallest credible working v0:

1. Postgres schema with tenants, RLS-ready tables, source records, normalized entities, agent runs, recommendations, citations.
2. Four seeded connectors behind one abstraction.
3. Sync pipeline that loads seeded data into normalized tables.
4. Deterministic metric services for inventory risk, ad waste, margin leaks, and support quality issues.
5. FastMCP server exposing namespaced tools over those services.
6. Claude Agent SDK coordinator and at least one fully working specialist agent.
7. Ops Cockpit with action queue, chat, and agent runs.
8. README explaining choices, scale path, eval honesty, hours spent, and another-week roadmap.

## Current Decision Log

- Product name: Jack.
- Scope: any D2C brand, not a niche-only merchant type.
- Primary value area: ops/profit.
- Product form: multi-agent system.
- UX: Ops Cockpit.
- Connectors: 4 seeded/API-shaped connectors.
- Database: Postgres.
- Tenancy: tenant internally, merchant in UI.
- Isolation: Postgres RLS.
- Agent runtime: Claude Agent SDK.
- Agent tools: FastMCP.
- Observability: Postgres run logs first, OpenTelemetry if time permits.
