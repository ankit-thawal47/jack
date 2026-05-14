# Jack: D2C AI Workforce OS — Strengthened Product + Tech Plan

## Product Thesis

Jack is a multi-agent operating layer for D2C brands.

D2C founders and operators run their business across many SaaS tools. Cross-tool questions require stitching exports across commerce, ads, support, inventory, and finance. The product acts like a team of AI employees that monitor this operating layer, reason over normalized data, and propose concrete ops-saving or profit-saving actions — with every numerical claim cited back to specific source rows.

The v0 is an **Ops Cockpit**: action queue, chat, and agent run logs as first-class surfaces.

---

## Product Shape

Jack is not a generic chatbot and not a collection of disconnected agents. It is a **shared data and action platform** with specialist AI employees on top.

Core components:

- Seeded SaaS connectors behind a shared typed abstraction
- Universal D2C data model with row-level provenance
- Concrete citation contract: all numbers traceable to source rows
- FastMCP server with typed tool schemas
- Claude Agent SDK multi-agent system with explicit agent design cards
- Multi-tenant Postgres database with RLS
- Ops Cockpit: action queue, chat, agent run logs

---

## User Experience

### Action Queue

Prioritized recommendations from agents:

- action description
- expected rupee impact
- agent/source
- confidence (0–1)
- status: `proposed | approved | dismissed | needs_review`
- citations (list of `[cit:{table}:{id}]` pointers)

### Chat

Operators ask cross-tool questions:

- Why is profit down this week?
- Which SKUs should I reorder?
- Where is ad spend being wasted?
- Show products with rising complaints and falling margin.

Every numerical claim in a chat answer carries a citation tag pointing to a source row or metric observation. Uncited numbers do not survive to the user — the architecture enforces this, not just a prompt instruction.

### Agent Runs

Transparent run logs:

- trigger (what fired the agent)
- agent name + run ID
- tools called + inputs + outputs
- rows inspected (`source_row_ids`)
- calculations performed
- reasoning summary
- recommendation created (or not, with reason)
- failure modes and caveats encountered during this run

---

## Connectors

### WHY These Four Connectors

| Connector | Why | Why not alternatives |
|---|---|---|
| **Shopify** | 80%+ of Indian D2C brands run on Shopify. It is the single source of truth for orders, revenue, inventory, refunds, and product catalog. No D2C AI system is credible without it. | BigCommerce, WooCommerce — market share too small in Indian D2C to justify v0 priority. |
| **Meta Ads** | Meta (Instagram + Facebook) is the dominant paid acquisition channel for Indian D2C brands — typically 40–70% of total ad budget. Without Meta spend data, ROAS and contribution margin calculations are incomplete. | Google Ads — better for search-intent, lower priority for cold-acquisition D2C. Klaviyo — retention channel, no spend signal relevant to margin analysis. |
| **Support (Gorgias-shaped)** | Support ticket volume is the earliest leading indicator of product quality issues — earlier than refunds, earlier than reviews. Gorgias is purpose-built for D2C with structured complaint categories that map cleanly to SKU-level signals. | Zendesk — valid alternative with same shape; connector abstraction makes swap trivial. WhatsApp — unstructured, no category tagging in v0. |
| **Finance (Xero/QuickBooks-shaped)** | COGS data is the only way to calculate real contribution margin. Without COGS, margin analysis is fictional (revenue analysis dressed as margin analysis). Xero and QuickBooks together cover ~70% of Indian D2C accounting setups. | Stripe — payment processor, not accounting; no COGS. Manual CSV — not a connector. |

### WHY These Agents (not others)

| Agent | Why primary |
|---|---|
| **Inventory Agent** | Stockouts are the single highest-impact ops failure for D2C. One day of stockout on a top SKU can lose ₹50k+ in revenue. Most concrete, most testable, data all in Shopify. First to build. |
| **Ad Waste Agent** | Scaling spend on a poorly stocked or low-margin SKU burns cash. The cross-signal insight (ads × inventory × margin) is uniquely enabled by the multi-connector architecture — unavailable in any single SaaS tool. |
| **Margin Agent** | Founders typically see revenue, not margin. Surfacing contribution margin degradation early prevents the "we were growing but losing money" trap. |
| **Support Quality Agent** | Complaint patterns predict future refunds and churn before they show up in revenue metrics. Stopping a scaling SKU before it accrues 1,000 negative reviews is worth more than the saved refunds. |

---

### Connector Abstraction

Connectors are backend ingestion adapters. Agents never call connectors directly — agents call MCP tools over normalized data.

**Typed Python interface:**

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Protocol
import uuid

@dataclass
class RawRecord:
    external_id: str
    entity_type: str       # "order", "campaign", "ticket", "finance_entry"
    raw_payload: dict      # full API response, retained for provenance
    fetched_at: datetime

@dataclass
class NormalizedRecord:
    id: uuid.UUID
    tenant_id: uuid.UUID
    source: str            # "shopify" | "meta_ads" | "support" | "finance"
    external_id: str
    entity_type: str
    normalized_data: dict  # source-agnostic fields
    source_record_id: uuid.UUID  # FK to source_records table
    synced_at: datetime

@dataclass
class SyncResult:
    records_fetched: int
    records_normalized: int
    records_skipped: int
    errors: list[str]
    cursor: str | None     # for incremental sync

class Connector(Protocol):
    source_name: str

    def sync(
        self,
        tenant_id: uuid.UUID,
        since: datetime | None
    ) -> SyncResult: ...

    def list_records(
        self,
        tenant_id: uuid.UUID,
        entity_type: str,
        since: datetime | None,
        limit: int = 100
    ) -> list[RawRecord]: ...

    def normalize(self, raw_record: RawRecord) -> NormalizedRecord: ...
```

All four connectors implement this `Protocol`. Swapping Shopify for BigCommerce requires only a new class that satisfies the same protocol — no changes elsewhere.

**What "seeded" means in v0:**

Each connector's `list_records()` reads from a realistic JSON fixture file instead of calling a live API. The fixture is generated from real Shopify/Meta Ads API shapes (same fields, same types, realistic Indian D2C data — SKU names, INR amounts, support ticket themes). The abstraction is real; the transport is a file reader for v0. OAuth flows and live API calls are the week-2 upgrade path.

---

## Data Architecture

### Postgres — Core Tables

```sql
-- Multi-tenancy foundation
tenants (id, name, created_at, plan)
tenant_users (id, tenant_id, user_id, role)
connector_accounts (id, tenant_id, source, credentials_encrypted, last_synced_at)

-- Raw source data (all connectors write here)
source_records (
    id uuid PRIMARY KEY,
    tenant_id uuid NOT NULL REFERENCES tenants(id),
    source text NOT NULL,           -- "shopify" | "meta_ads" | "support" | "finance"
    entity_type text NOT NULL,      -- "order" | "campaign" | "ticket" | etc.
    external_id text NOT NULL,
    raw_payload jsonb NOT NULL,     -- full API response, never modified
    connector_account_id uuid REFERENCES connector_accounts(id),
    synced_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, source, external_id)
)

-- Normalized business entities (provenance via source_record_id FK)
products (id, tenant_id, source_record_id, external_id, title, handle, synced_at)
skus (id, tenant_id, source_record_id, product_id, external_id, title, sku_code, price_inr, cost_inr, synced_at)
inventory_snapshots (id, tenant_id, source_record_id, sku_id, location_id, quantity, snapshot_at)
orders (id, tenant_id, source_record_id, external_id, status, total_inr, created_at)
order_lines (id, tenant_id, source_record_id, order_id, sku_id, quantity, unit_price_inr, line_total_inr)
refunds (id, tenant_id, source_record_id, order_id, reason, amount_inr, refunded_at)
ad_campaigns (id, tenant_id, source_record_id, external_id, name, status, objective)
ad_daily_metrics (id, tenant_id, source_record_id, campaign_id, date, spend_inr, impressions, clicks, conversions, attributed_revenue_inr)
support_tickets (id, tenant_id, source_record_id, external_id, subject, body, category, sentiment_score, sku_references text[], status, created_at)
finance_entries (id, tenant_id, source_record_id, entry_type, sku_id, order_id, amount_inr, period_start, period_end)

-- Computed metrics (with provenance chain to source rows)
metric_observations (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid NOT NULL REFERENCES tenants(id),
    metric_name text NOT NULL,         -- "days_of_cover" | "roas_7d" | "contribution_margin"
    subject_type text NOT NULL,        -- "sku" | "campaign" | "product"
    subject_id text NOT NULL,          -- sku.id, campaign.id, etc.
    value numeric NOT NULL,
    unit text,                         -- "days" | "inr" | "ratio" | "count"
    period_start timestamptz NOT NULL,
    period_end timestamptz NOT NULL,
    source_record_ids uuid[] NOT NULL, -- source_records.id[] used to compute this metric
    calculation_description text,      -- "sum(order_lines.quantity) / avg_daily_velocity(14d)"
    computed_at timestamptz NOT NULL DEFAULT now()
)

-- Agent system tables
agent_runs (id, tenant_id, agent_name, trigger_type, trigger_payload, status, started_at, finished_at, error)
agent_run_steps (id, run_id, step_type, tool_name, input_json, output_json, source_row_ids uuid[], duration_ms, created_at)
tool_calls (id, run_id, step_id, tool_name, tenant_id, agent_name, input_json, output_summary, source_row_ids uuid[], duration_ms, error)
recommendations (id, tenant_id, run_id, agent_name, recommendation_type, subject_type, subject_id, action_description, estimated_rupee_impact, confidence, status, caveats text[], created_at)
recommendation_citations (id, recommendation_id, table_name, record_id, field_name, value_cited)
```

### Provenance Rule

`raw_payload` lives **only** in `source_records`. Normalized rows carry a `source_record_id` FK — they do NOT duplicate `raw_payload`. This prevents storage bloat and ensures a single source of truth for the raw API response.

Citation chain: `recommendation_citations.record_id` → `metric_observations.id` → `metric_observations.source_record_ids[]` → `source_records.raw_payload`. Every cited number is fully traceable to the API response that generated it.

---

## Multi-Tenancy and RLS

### Policy Pattern

```sql
-- Applied to every business table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_orders ON orders
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

Every API request, sync job, MCP tool call, and agent run sets tenant context at transaction start:

```sql
BEGIN;
SET LOCAL app.current_tenant_id = '<tenant-id>';
-- ... all queries run here under RLS
COMMIT;
```

### PgBouncer Gotcha

**Critical:** PgBouncer in transaction mode (the default for SaaS) resets session state between transactions. `SET` (without `LOCAL`) leaks state across pooled connections. You **must** use `SET LOCAL` inside an explicit `BEGIN ... COMMIT` block — never at the session level. Verify this in every code path: API handlers, sync workers, MCP tool calls, and agent run launchers.

### Indexes

```sql
CREATE INDEX idx_orders_tenant_created ON orders (tenant_id, created_at DESC);
CREATE INDEX idx_source_records_tenant_source ON source_records (tenant_id, source, external_id);
CREATE INDEX idx_inventory_tenant_sku ON inventory_snapshots (tenant_id, sku_id, snapshot_at DESC);
CREATE INDEX idx_agent_runs_tenant_agent ON agent_runs (tenant_id, agent_name, started_at DESC);
CREATE INDEX idx_metric_obs_subject ON metric_observations (tenant_id, subject_type, subject_id, period_end DESC);
```

---

## Citation Contract

This is the hardest requirement. The mechanism is architectural — not a post-hoc string parser.

### Implementation Mechanism

**Rule: all numbers in chat answers come from MCP tool call outputs. The LLM is not permitted to generate numbers.**

1. **Tool output contract:** Every MCP tool returns a typed result that includes `source_row_ids: list[uuid]`. The metric engine stores every computed metric in `metric_observations` with its `source_record_ids`. Tools return pre-formatted citation tags alongside values:

    ```python
    # Tool output example
    {
        "sku_id": "abc123",
        "days_of_cover": 4.2,
        "days_of_cover_citation": "[cit:metric_observations:obs-uuid-001]",
        "current_stock": 84,
        "current_stock_citation": "[cit:inventory_snapshots:snap-uuid-007]",
        "source_row_ids": ["source-uuid-011", "source-uuid-012"]
    }
    ```

2. **Chat system prompt:** The LLM receives tool outputs formatted with citation tags already embedded. The system prompt includes: *"Every number you use in your answer must be taken verbatim from the tool output above, including its citation tag in brackets. Do not perform your own arithmetic. Do not use numbers from your training knowledge. If you cannot cite a number, say you don't have that data."*

3. **Citation validator:** After the LLM generates a response, a post-processing pass runs:
    - Regex: find all bare numeric tokens (e.g. `4.2`, `₹45,000`, `23%`) not followed by `[cit:...]`
    - If any found: re-prompt once with the specific uncited numbers flagged
    - If re-prompt still contains uncited numbers: return a structured error response to the user: `"I found data but cannot generate a fully cited answer. Please ask me to show you the raw metrics instead."`
    - Log every validation failure to `agent_run_steps` with `step_type = 'citation_validation_failure'`

4. **What this catches:** fabricated numbers from LLM training knowledge. **What this misses:** numbers expressed in word form ("forty-five thousand rupees"), or numbers that happen to match a cited value by coincidence. This is called out in Eval Honesty.

---

## Agent Design Cards

### WHY This Set of Agents

The four specialist agents map to the four leading causes of D2C profit erosion: stockouts (Inventory), ad waste (Ad Waste), margin compression (Margin), and quality failures (Support Quality). The Coordinator is the synthesis layer. This is not an arbitrary selection — these are the exact cross-tool insights that a D2C founder currently cannot get from any single SaaS tool.

---

### Inventory Agent

**Purpose:** Prevent stockouts and overstock before they cost money.

**Trigger:**
- `on_sync_complete(source='shopify')` — fires after each Shopify sync
- Nightly cron: `06:00 IST` (fallback if sync event is missed)

**Input (MCP tool call):**
```python
find_inventory_risks(
    tenant_id=...,
    days_cover_threshold=7,      # flag SKUs with < 7 days of cover
    overstock_threshold=60,       # flag SKUs with > 60 days of cover + declining velocity
    velocity_window_days=14,      # rolling window for velocity calculation
    limit=20                      # cap: top 20 by revenue at risk
)
```

**Decision Logic:**
```
days_of_cover = current_stock / avg_daily_velocity(14 days)
velocity_trend = linear_slope(daily_sales, 14 days)  # positive = growing

IF days_of_cover < 3:
    → URGENT_REORDER (high urgency)
ELIF days_of_cover < 7 AND velocity_trend >= 0:
    → REORDER (medium urgency)
ELIF days_of_cover > 60 AND velocity_trend < 0:
    → DISCOUNT (low urgency, overstock)
ELSE:
    → no action
```

**Action Object Schema:**
```python
@dataclass
class InventoryRecommendation:
    recommendation_type: Literal["URGENT_REORDER", "REORDER", "DISCOUNT"]
    sku_id: str
    sku_title: str
    current_stock: int
    avg_daily_velocity: float
    days_of_cover: float
    recommended_order_qty: int      # days_cover_target=30 * avg_daily_velocity
    estimated_rupee_impact: float   # days_at_risk * avg_daily_velocity * avg_selling_price_inr
    confidence: float               # 0.9 if velocity is stable, 0.6 if noisy
    source_row_ids: list[uuid.UUID]
    caveats: list[str]
```

**Failure Modes:**
1. **Shopify sync stale (>24h):** Agent checks `connector_accounts.last_synced_at`. If stale, logs warning to `agent_runs.error`, skips run, does not create recommendations.
2. **Missing SKU cost (no COGS):** `estimated_rupee_impact` is calculated using `skus.price_inr` as a proxy. Caveat appended: "Impact estimate uses selling price, not margin — COGS data unavailable."
3. **All SKUs flagged:** Cap recommendations at top 20 by `estimated_rupee_impact DESC`. Log how many were suppressed.
4. **Zero velocity SKU:** If `avg_daily_velocity == 0`, `days_of_cover = ∞`. SKU is excluded from reorder recommendations. Added to a separate "no recent sales" flag.
5. **Seasonal demand:** Agent assumes flat velocity (14-day rolling average). This is wrong for seasonal products. Known limitation — called out in every reorder caveat: "Velocity calculated from 14-day rolling average; does not account for seasonal demand patterns."

---

### Ad Waste Agent

**Purpose:** Stop burning ad budget on low-ROAS campaigns or SKUs that can't be fulfilled.

**Trigger:**
- Nightly cron: `06:30 IST` (after Meta Ads sync, 30min after Inventory Agent)

**Input (MCP tool call):**
```python
find_ad_waste(
    tenant_id=...,
    roas_threshold=2.0,           # flag campaigns with ROAS < 2.0
    spend_threshold_inr=1000.0,   # only flag campaigns spending > ₹1,000 over lookback
    lookback_days=7,
    check_inventory=True          # cross-reference with inventory status
)
```

**Decision Logic:**
```
FOR EACH campaign in ad_daily_metrics (last 7 days, spend > ₹1,000):
    roas = attributed_revenue_inr / spend_inr

    IF roas < 1.0:
        → URGENT_PAUSE (spending more than earning)
    ELIF roas < 2.0:
        IF campaign targets SKU with days_of_cover < 3:
            → PAUSE_INVENTORY_CONFLICT (would create stockout + refunds)
        ELSE:
            → INVESTIGATE_PERFORMANCE
    ELIF campaign targets SKU with days_of_cover < 3:
        → FLAG_INVENTORY_CONFLICT (good ROAS, bad timing)
```

**Action Object Schema:**
```python
@dataclass
class AdWasteRecommendation:
    recommendation_type: Literal["URGENT_PAUSE", "INVESTIGATE_PERFORMANCE", "PAUSE_INVENTORY_CONFLICT", "FLAG_INVENTORY_CONFLICT"]
    campaign_id: str
    campaign_name: str
    spend_7d_inr: float
    roas_7d: float
    attributed_revenue_7d_inr: float
    estimated_waste_inr: float        # spend_7d - (spend_7d * roas_threshold)
    linked_sku_id: str | None
    linked_sku_days_of_cover: float | None
    confidence: float
    source_row_ids: list[uuid.UUID]
    caveats: list[str]
```

**Failure Modes:**
1. **Attribution window mismatch:** Meta Ads reports on 7-day click / 1-day view attribution. ROAS in this system uses the attributed_revenue field from the fixture. Real Meta data may show different numbers due to view-through attribution. Caveat: "ROAS based on Meta-reported attribution; actual blended ROAS may differ."
2. **COGS unavailable:** Contribution margin check skipped. ROAS-only analysis is noted in caveat.
3. **New campaigns (<3 days old):** Excluded from waste analysis — insufficient data for reliable ROAS signal.
4. **Campaign targets multiple SKUs:** If a campaign targets a product group, the inventory conflict check uses the lowest days_of_cover across all linked SKUs. Caveat appended if multi-SKU.

---

### Margin Agent

**Purpose:** Surface contribution margin degradation before it becomes a crisis.

**Trigger:**
- Weekly cron: `Sunday 07:00 IST`
- On-demand: when Coordinator requests it for a specific SKU

**Input (MCP tool call):**
```python
find_margin_leaks(
    tenant_id=...,
    margin_threshold=0.30,     # flag SKUs with contribution margin < 30%
    lookback_weeks=4,
    include_trend=True         # flag SKUs declining >10% WoW for 2+ consecutive weeks
)
```

**Decision Logic:**
```
FOR EACH sku over last 4 weeks:
    revenue = sum(order_lines.line_total_inr)
    cogs = sum(finance_entries.amount_inr WHERE entry_type='cogs')
    ad_spend = sum(ad_daily_metrics.spend_inr WHERE campaign targets this sku)
    refunds = sum(refunds.amount_inr WHERE order line references this sku)
    contribution_margin = (revenue - cogs - ad_spend - refunds) / revenue

    IF contribution_margin < 0.30:
        → PRICING_REVIEW
    IF margin_trend is declining >10% for 2 consecutive weeks:
        → MARGIN_INVESTIGATION
    IF ad_spend / revenue > 0.50:
        → AD_SPEND_ADJUSTMENT
```

**Action Object Schema:**
```python
@dataclass
class MarginRecommendation:
    recommendation_type: Literal["PRICING_REVIEW", "MARGIN_INVESTIGATION", "AD_SPEND_ADJUSTMENT"]
    sku_id: str
    sku_title: str
    revenue_4w_inr: float
    cogs_4w_inr: float
    ad_spend_4w_inr: float
    refunds_4w_inr: float
    contribution_margin: float
    margin_trend: list[float]     # weekly CM for last 4 weeks
    confidence: float
    source_row_ids: list[uuid.UUID]
    caveats: list[str]
```

**Failure Modes:**
1. **Finance connector unavailable:** Falls back to `skus.cost_inr` (Shopify product cost field) as COGS approximation. Caveat: "COGS estimated from Shopify product cost field; finance connector not synced."
2. **Refund attribution:** Refunds are attributed to original order date (Shopify behavior). This may distort weekly margin for months with high delayed refunds.
3. **Ad spend → SKU attribution:** Ad campaigns may not be tagged to specific SKUs. When untagged, ad spend is distributed proportionally across all SKUs by revenue share. This is an approximation; caveat appended.

---

### Support Quality Agent

**Purpose:** Catch product quality issues before they become refund and churn problems.

**Trigger:**
- `on_sync_complete(source='support')` — fires after each support sync
- Daily cron: `07:30 IST` (fallback)

**Input (MCP tool call):**
```python
find_support_quality_issues(
    tenant_id=...,
    ticket_volume_threshold=5,   # flag SKUs with >= 5 complaints in period
    period_days=7,
    include_refund_correlation=True
)
```

**Decision Logic:**
```
FOR EACH sku referenced in support_tickets (last 7 days):
    complaint_count = count(tickets WHERE sku in ticket.sku_references)
    refund_count = count(refunds WHERE sku linked to refunded order, last 7 days)
    sentiment_avg = avg(ticket.sentiment_score) if available, else None

    IF complaint_count >= 5:
        → QUALITY_INVESTIGATION
    IF refund_count >= 3 AND complaint_theme includes "defective" | "broken" | "quality":
        → STOP_SCALING
    IF complaint_count >= 3 AND sku is currently in active ad campaign:
        → FLAG_SCALING_RISK (agent may be driving growth into a quality problem)
```

**Action Object Schema:**
```python
@dataclass
class SupportQualityRecommendation:
    recommendation_type: Literal["QUALITY_INVESTIGATION", "STOP_SCALING", "FLAG_SCALING_RISK"]
    sku_id: str
    sku_title: str
    complaint_count_7d: int
    top_complaint_themes: list[str]   # extracted from ticket subjects/bodies
    refund_count_7d: int
    sentiment_avg: float | None       # None if not available in v0
    active_campaign_ids: list[str]    # campaigns scaling this SKU
    confidence: float
    source_row_ids: list[uuid.UUID]
    caveats: list[str]
```

**Failure Modes:**
1. **Ticket → SKU matching is keyword-based:** `ticket.sku_references` is populated by matching product names/SKU codes against ticket subject + body text. Will produce false positives for ambiguous product names (e.g., "Red T-Shirt" matches any ticket mentioning "red" and "shirt"). Known limitation — caveat on every recommendation.
2. **Sentiment not implemented in v0:** `sentiment_score` defaults to `None`. Tickets are scored by escalation flag and priority level instead. Caveat appended.
3. **Support connector only tracks complaint categories if available in fixture:** The seeded data includes pre-categorized tickets. Real Gorgias/Zendesk data may require category extraction.

---

### Coordinator Agent

**Purpose:** Combine specialist findings into a single ranked action queue. Prevent conflicting or duplicate recommendations from confusing the operator.

**Trigger:**
- After any specialist agent run completes (`on_agent_run_complete`)
- Nightly cron: `08:00 IST` (full synthesis run)

**Decision Logic:**

```
1. DEDUP
   Key: (tenant_id, subject_type, subject_id, recommendation_type)
   Window: 24 hours
   Rule: if same key appears multiple times, keep highest confidence. Log suppressed duplicates.

2. CONFLICT DETECTION
   Conflict: REORDER(sku_id=X) + STOP_SCALING(sku_id=X)
   Rule: DO NOT auto-resolve conflicts. Mark both recommendations with status='needs_review'.
   Present both to operator with conflict summary: "Inventory agent recommends reorder, Support agent
   recommends stopping growth due to quality issues. Human decision required."
   Log conflict to agent_runs with step_type='coordinator_conflict'.

3. RANKING
   score = (estimated_rupee_impact * urgency_multiplier * confidence)
   urgency_multiplier: URGENT_* = 2.0, default = 1.0
   Sort DESC by score. Cap action queue at 10 visible recommendations.

4. OPERATOR SUMMARY
   LLM call: given ranked recommendations + conflict flags, produce a 3-sentence plain-language summary
   of the most important actions for the operator today.
   This summary is the ONLY LLM-generated text in the action queue — it contains no numbers,
   only references recommendations by their action_description.
```

**Failure Modes:**
1. **₹ impact estimates are rough:** Formula is `velocity × days_at_risk × avg_selling_price`. This is directional for ranking, not a forecast. Every recommendation marked: "Impact estimate is directional. Do not use as financial projection."
2. **Coordinator runs before all specialists finish:** If a specialist is still running, coordinator uses stale recommendations from previous runs. Logs which specialist runs were pending.
3. **All recommendations are conflicts:** If >50% of recommendations have conflicts, coordinator raises a `coordinator_alert` run step and summarizes the conflicts for operator review rather than attempting a ranked queue.

---

## FastMCP Tool Schemas

All tools are namespaced. Every tool returns `source_row_ids` for citation. Every tool runs inside a tenant-scoped transaction.

```python
# tools/inventory.py
@mcp.tool()
def find_inventory_risks(
    tenant_id: str,
    days_cover_threshold: int = 7,
    overstock_threshold: int = 60,
    velocity_window_days: int = 14,
    limit: int = 20
) -> list[InventoryRisk]:
    """
    Find SKUs at risk of stockout or overstock.
    Ranks by estimated revenue at risk (days_at_risk × velocity × price).
    Returns source_row_ids for every metric cited.
    Does NOT perform any calculations at call time — reads from metric_observations.
    """

@mcp.tool()
def get_sku_metrics(
    tenant_id: str,
    sku_id: str,
    metrics: list[str],     # ["inventory", "velocity", "margin", "complaints", "ad_spend"]
    period_days: int = 30
) -> SkuMetrics:
    """
    Get cross-source metrics for a specific SKU.
    Joins inventory, orders, finance, support, and ad data.
    Returns each metric with its own source_row_ids for granular citation.
    """

# tools/ads.py
@mcp.tool()
def find_ad_waste(
    tenant_id: str,
    roas_threshold: float = 2.0,
    spend_threshold_inr: float = 1000.0,
    lookback_days: int = 7,
    check_inventory: bool = True
) -> list[AdWaste]:
    """
    Find campaigns with ROAS below threshold and spend above threshold.
    Optionally cross-references inventory status for stockout conflict detection.
    Returns source_row_ids from ad_daily_metrics and inventory_snapshots.
    """

# tools/margin.py
@mcp.tool()
def find_margin_leaks(
    tenant_id: str,
    margin_threshold: float = 0.30,
    lookback_weeks: int = 4,
    include_trend: bool = True
) -> list[MarginLeak]:
    """
    Find SKUs with contribution margin below threshold or declining trend.
    Contribution margin = (revenue - cogs - ad_spend - refunds) / revenue.
    Falls back to Shopify cost field if finance connector unavailable; flags caveat.
    Returns source_row_ids from order_lines, finance_entries, ad_daily_metrics, refunds.
    """

# tools/support.py
@mcp.tool()
def find_support_quality_issues(
    tenant_id: str,
    ticket_volume_threshold: int = 5,
    period_days: int = 7,
    include_refund_correlation: bool = True
) -> list[SupportQualityIssue]:
    """
    Find SKUs with elevated complaint volume or refund patterns.
    Ticket→SKU matching uses keyword matching on product names.
    Returns source_row_ids from support_tickets and refunds.
    """

# tools/recommendations.py
@mcp.tool()
def create_recommendation(
    tenant_id: str,
    agent_name: str,
    run_id: str,
    recommendation_type: str,
    subject_type: str,                  # "sku" | "campaign" | "product"
    subject_id: str,
    action_description: str,
    estimated_rupee_impact: float,
    confidence: float,                  # 0.0–1.0
    source_row_ids: list[str],
    caveats: list[str]
) -> RecommendationResult:
    """
    Create a recommendation in the action queue. Writes to recommendations table.
    Returns recommendation_id for subsequent citation linking.
    """

@mcp.tool()
def get_citations(
    tenant_id: str,
    citation_ids: list[str]             # e.g. ["[cit:metric_observations:uuid]"]
) -> list[CitationDetail]:
    """
    Resolve citation tags to human-readable source detail.
    Returns table, record_id, field, value, and a link to the raw source record.
    Used by the chat layer to expand citations in UI.
    """
```

**Return types all include `source_row_ids: list[uuid]`.** The metric engine pre-computes all metrics and stores them in `metric_observations` with their `source_record_ids`. Tools read from `metric_observations`, not from raw tables, keeping agent response times fast and calculations deterministic.

---

## Scalability Layer

### What's Built for Scale in v0

- Tenant-scoped Postgres with RLS hard boundary
- `(tenant_id, ...)` leading composite indexes on all queried tables
- `SET LOCAL app.current_tenant_id` enforced in every transaction
- `source_records` table retains raw payloads for replay/backfill without re-fetching from APIs
- `metric_observations` pre-computes metrics on sync completion — agents read pre-computed values, not raw tables
- Async sync jobs (one per connector per tenant) — isolated failure domains
- Action queue instead of direct external writes — no irreversible agent actions in v0
- Unique constraint `(tenant_id, source, external_id)` prevents duplicate ingestion on re-sync

### Capacity Estimate

A merchant doing ₹1 crore/month (~₹33k/day) generates roughly:
- 100–300 orders/day → ~3,000–9,000 order_lines/month
- 30 Meta Ads campaigns → ~900 ad_daily_metric rows/month
- 200–500 support tickets/month

At 10,000 tenants, active tables at 2 years:
- `order_lines`: ~10,000 × 6,000 × 24 = ~1.4 billion rows
- `ad_daily_metrics`: ~10,000 × 900 × 24 = ~216 million rows

Without table partitioning, `order_lines` EXPLAIN queries start degrading past ~500M rows even with composite indexes. Partition by `(tenant_id, MONTH(created_at))` absorbs this. This is a week-4 migration — not needed in v0 but the schema should be partition-ready.

### What Breaks First at Scale

1. **SaaS API rate limits and OAuth token refresh:** Shopify allows ~2 req/sec per store. At 10k tenants with hourly syncs, a naive scheduler hits Shopify's rate limits immediately. Need: per-tenant rate limiter with token bucket, OAuth token refresh queue.

2. **Ingestion job scheduling:** A cron that fires all 10k tenants at 06:00 IST creates a thundering herd on the database and APIs. Need: staggered tenant schedules with jitter, or a job queue (Celery/RQ) with per-source concurrency limits.

3. **Agent run cost and latency:** Each full agent run (4 specialists + coordinator) costs approximately 8–15 Claude API calls. At $0.01/call average, that's ~$0.15/tenant/day = $1,500/day at 10k tenants. Need: model tiering (haiku for prefiltering, sonnet for synthesis), deterministic prefilters to skip agent runs when no metrics change.

4. **Metric query performance on large tables:** As `order_lines` grows past 100M rows, metric pre-computation queries slow. Need: materialized metric tables or incremental metric updates on sync completion (update only the delta, not full recompute).

5. **Noisy recommendations at scale:** At 10k tenants, the coordinator's queue could have thousands of recommendations. Without dedup and ranking calibration, operators stop trusting the system. Need: per-tenant recommendation dedup window, relevance scoring calibration, operator feedback loop.

---

## Eval Honesty

*What doesn't work in v0 — named before the evaluators find it.*

1. **Connectors use seeded fixtures, not live OAuth.** The connector abstraction is real and swappable. But in v0, Shopify reads from `fixtures/shopify_orders.json`, not from a live Shopify API. Real OAuth token management, webhook handling, and API rate limiting are week-2 work.

2. **Citation validator can be bypassed.** The validator catches uncited numbers in the LLM's text output by checking for bare numeric tokens. It does not catch: numbers expressed in word form ("forty-five thousand rupees"), or numbers that coincidentally match a cited value without being cited. A sufficiently adversarial query can produce uncited numbers that pass the check.

3. **Support ticket → SKU matching is keyword-based and noisy.** `ticket.sku_references` is populated by matching product name keywords against ticket text. False positive rate is high for common words ("red", "medium", "basic"). False negatives occur for indirect language ("the thing I ordered last week"). A production version requires semantic embedding-based matching.

4. **Days-of-cover assumes flat velocity.** The Inventory Agent calculates `days_of_cover = stock / avg_daily_velocity(14d)`. This is wrong for seasonal products, products with a recent viral spike, or new SKUs with <14 days of sales history. Every reorder recommendation includes this caveat explicitly.

5. **₹ impact estimates are directional, not financial projections.** The formula `days_at_risk × avg_velocity × avg_price_inr` is useful for ranking recommendations by relative importance. It is not a revenue forecast. It will be wrong for products with volatile demand.

6. **Coordinator conflict resolution is deferral, not resolution.** When Inventory says "reorder SKU X" and Support says "stop scaling SKU X," the coordinator presents both with a "Human decision required" flag. It does not attempt to synthesize a combined recommendation — that would require product-domain knowledge the agent doesn't have.

7. **RLS tested for application code, not adversarial query shapes.** RLS policies are configured and every application query sets tenant context. However, the policies have not been tested against all possible query shapes an agent might generate (e.g., subqueries, CTEs, lateral joins). A missed `SET LOCAL` in one code path is a cross-tenant data leak.

8. **Agent runs are not truly parallel in v0.** Specialist agents run sequentially in the implementation to simplify debugging. The architecture supports parallelism (each subagent gets isolated context via Claude Agent SDK), but the v0 scheduler is sequential. True parallel execution is the week-2 improvement.

9. **No customer table = no LTV analysis.** The schema doesn't model customers. Repeat purchase rate, LTV, and cohort analysis are not available. These are high-value D2C signals that require a customers + orders join and are planned for v1.

---

## README Draft Answers

*Written before coding so the "why" is documented while the decisions are fresh.*

**1. What you built — 5-line architecture summary**

Jack is a multi-agent AI operating layer for D2C brands. Four specialist AI agents (Inventory, Ad Waste, Margin, Support Quality) continuously monitor normalized data from Shopify, Meta Ads, support, and finance connectors. Every agent recommendation cites back to source rows. A FastMCP server exposes typed tools over a multi-tenant Postgres database with RLS. The Ops Cockpit surfaces a ranked action queue, a cited chat interface, and transparent agent run logs.

**2. Connectors — which 3 (shipping), why**

Shipping: Shopify (commerce source of truth for 80%+ of Indian D2C), Meta Ads (primary paid acquisition for D2C in India, 40–70% of ad spend), Gorgias-shaped support (earliest quality signal, SKU-level complaint categorization). Finance connector is the 4th, shipping if time permits.

Why not Google Ads: Meta dominates D2C cold acquisition in India. Why not Klaviyo: retention channel, not margin/ops signal. Why not Shipstation: fulfillment timing is downstream of the core ops decisions we're targeting.

**3. Schema — why this shape**

Every normalized row has a `source_record_id` FK to the raw API payload in `source_records`. The citation chain flows from any recommendation → `metric_observations.source_record_ids[]` → `source_records.raw_payload`. `metric_observations` is the pivot: pre-computed, deterministic, citeable. The LLM never calculates — it reads pre-computed metrics and cites them.

**4. Chat — tool schema exposed, how citation works**

Tools: `find_inventory_risks`, `find_ad_waste`, `find_margin_leaks`, `find_support_quality_issues`, `get_sku_metrics`, `create_recommendation`, `get_citations`. Every tool returns `source_row_ids`. The LLM receives tool outputs with pre-formatted citation tags and is constrained to only use numbers from those outputs. A post-processing validator checks for uncited numeric tokens. See tool schemas above for full typed signatures.

**5. Agent — what it does, why this one first**

The Inventory Agent fires on Shopify sync completion. It queries `metric_observations` for days_of_cover and velocity trend. If days_of_cover < 7 and velocity is not declining, it creates a REORDER recommendation with estimated rupee impact and cites the specific inventory snapshot and sales records used. Inventory was built first because stockouts are the highest-impact, most concrete, and most testable ops failure for D2C — the data all lives in Shopify alone, making it the best proof of concept.

**6. Scale — 1 → 10,000 merchants**

Built: RLS tenant isolation, composite tenant-scoped indexes, pre-computed metric_observations (agents don't hit raw tables), async sync jobs, action queue (no direct external writes), source_records retained for replay. What breaks first: SaaS API rate limits at ingestion scale, thundering herd at cron time, agent run cost at $0.15/tenant/day, metric query latency past 500M order_lines rows. Fixes: per-tenant rate limiter, staggered cron with jitter, model tiering (haiku prefilter), table partitioning by month.

**7. Eval — where it breaks**

See Eval Honesty section above. Short version: (1) fixtures not live API, (2) citation validator bypassable with word-form numbers, (3) ticket→SKU matching is noisy keyword matching, (4) days-of-cover assumes flat velocity, (5) ₹ impact estimates are not forecasts.

**8. Hours spent**

[Fill in at submission.]

**9. What you'd do with another week**

Real Shopify OAuth + webhook ingestion. Embedding-based ticket→SKU matching. True parallel agent execution. Customer table for LTV analysis. Per-tenant agent schedules with cost controls. Frontend polish (current UI is functional, not polished). Google Ads connector.

---

## Build Priority — Sunday Scope

### MUST SHIP (core requirements satisfied)

1. Postgres schema + Alembic migration (tenants, source_records, products, skus, orders, order_lines, inventory_snapshots, ad_campaigns, ad_daily_metrics, support_tickets, refunds, finance_entries, metric_observations, agent_runs, agent_run_steps, tool_calls, recommendations, recommendation_citations)
2. RLS policies on all business tables + `SET LOCAL` in all query paths
3. Three connectors (Shopify + Meta Ads + Support) with `NormalizedRecord` return type and fixture data
4. Sync pipeline: reads fixtures → writes `source_records` → writes normalized tables → computes `metric_observations`
5. Inventory Agent: end-to-end with trigger, decision logic, `InventoryRecommendation` output, failure modes, agent run log
6. FastMCP server: `find_inventory_risks`, `get_sku_metrics`, `create_recommendation`, `get_citations`
7. Chat layer: Claude Agent SDK + FastMCP tool use + citation contract (validator pass)
8. README: all 9 questions answered, eval honesty section, WHY documentation

### SHIP IF TIME

- Finance connector (4th connector, needed for real margin analysis)
- Ad Waste Agent (2nd agent — simpler than Margin/Support)
- Coordinator (ranking + dedup logic — can ship with just 1 agent if pressed)
- Ops Cockpit frontend (React — skip entirely if backend is not solid)

### STUB IN README

- Margin Agent — designed and documented here, not implemented in v0
- Support Quality Agent — designed and documented here, not implemented in v0
- Full coordinator conflict resolution — described, not built
- OpenTelemetry export — "planned, not shipped"

---

## Implementation Stack

Backend:
- Python 3.12+
- FastAPI (API layer + MCP host)
- Postgres 16 (with RLS)
- SQLModel (typed ORM, aligns with Pydantic)
- Alembic (schema migrations)

Agents + tools:
- Claude Agent SDK (agent runtime)
- FastMCP 3.x (tool server)

Frontend (if time permits):
- Next.js (App Router) — minimal UI for Ops Cockpit three-panel layout

Data:
- Fixture JSON files (realistic seeded data — Indian D2C brand, INR amounts, real SKU patterns)

---

## Decision Log

| Decision | Choice | Reason |
|---|---|---|
| Product name | Jack | —  |
| Scope | Any D2C brand | Not niche-specific; the connector set covers the universal D2C stack |
| Primary value | Ops/profit | Revenue is visible; margin and ops costs are not — that's the gap Jack fills |
| Product form | Multi-agent OS | Cross-tool insights require multiple specialized readers, not one generalist |
| UX | Ops Cockpit (action queue + chat + run logs) | Founders act on recommendations, not dashboards |
| Connectors | Shopify + Meta Ads + Support + Finance | See WHY table above |
| Database | Postgres | Multi-tenant RLS, JSONB for raw payloads, excellent tooling |
| Tenancy word | tenant (internal) / merchant (UI) | Avoids confusion in API code |
| Isolation | Postgres RLS + `SET LOCAL` | Database-enforced, not application-enforced |
| Agent runtime | Claude Agent SDK | Anthropic's own runtime; subagent isolation is a first-class primitive |
| Tool server | FastMCP 3.x | Decorator-based, auto-schema generation, OTEL built-in |
| Citation mechanism | Tool-output-only numbers + regex validator | Architectural constraint, not prompt instruction |
| Observability | Postgres run logs (primary) + OTEL (if time permits) | Run logs are the product surface; OTEL is the infra surface |
| Agent scope for Sunday | Inventory Agent (1 fully working) + stubs documented | Depth > breadth for evaluation; 5 half-working agents score lower than 1 complete one |
