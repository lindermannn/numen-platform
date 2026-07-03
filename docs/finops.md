# FinOps Strategy

Cost control in a multi-tenant AI platform is not optional — a single runaway tenant or an accidental prompt loop can produce a surprise bill. This document describes Numen AI's full cost control stack: metering, enforcement, alerting, optimization, and billing.

---

## The Problem Space

Running GPT-4.1 at scale has two cost risks:

1. **Per-tenant overrun:** a tenant sends 10× expected volume in a single day. Without enforcement, the platform absorbs the cost with no way to recover it.
2. **Platform idle waste:** scheduled workflows (probes, analytics, backups) run regardless of whether there's any real traffic. At zero clients, these cost real money.

Both are solved differently and documented below.

---

## Layer 1: Per-Call Cost Metering

Every AI call (chat completion or embedding) flows through `module-cost-tracker`:

```
1. Receive { tenantId, model, inputChars, outputChars, type }
2. Load pricing from openai_cost DataTable:
   - gpt-4.1:              $2.00 / $8.00 per 1M tokens (input/output)
   - gpt-4o-mini:          $0.15 / $0.60 per 1M tokens
   - text-embedding-3-small: $0.02 per 1M tokens
3. Estimate tokens: chars / 4 (fast, ~10% error vs actual API count)
4. Insert row into token_usage:
   { tenantId, sessionId, model, inputTokens, outputTokens,
     estimatedCost, source: 'estimated', timestamp }
```

`token_usage` is the ledger. Every request has a record. Gaps indicate a telemetry failure (fire-and-forget), not missing data.

**Known limitation:** Token estimation via chars/4 is ~10% inaccurate vs OpenAI's actual count. Planned fix in Sprint E: use the `usage` field from the OpenAI API response for exact counts.

---

## Layer 2: 6-Hour Rollup + Spend Cap Enforcement

`module-cost-aggregator` runs on schedule: **00:10 / 06:10 / 12:10 / 18:10** (every 6 hours with 10-minute offset to avoid hour boundary collisions):

```
1. Read all token_usage rows for current month, group by tenantId
2. For each tenant:
   - sum estimatedCost → total_cost_usd
   - UPSERT into tenant_usage { tenantId, month, total_cost_usd, ... }
3. Check each tenant's total against plan spend cap:
   - free:       $1/month cap
   - starter:    $20/month cap
   - pro:        $90/month cap
   - enterprise: no hard cap
4. If total >= 80% of cap:
   - send Telegram alert to admin: "[ALERT] tenant_xxx at 83% of $20 budget"
   - rate-limited: 1 alert per tenant per day (no alert spam)
```

**core-router** checks `tenant_usage` on every incoming request:

```
if tenant.total_cost_usd >= plan.spend_cap:
    return HTTP 402 { error: "SPEND_CAP_EXCEEDED", retryable: false }
```

HTTP 402 is marked non-retryable in every channel adapter. The adapter delivers a spend cap message to the user and does not queue a retry. This prevents a runaway retry loop from doubling the cost of an already-over-budget tenant.

**Spend cap lag:** Maximum ≤6h (the aggregator interval). A tenant could technically exceed their cap intra-interval before the next rollup catches it. At current usage levels (~$0.06/conversation, 200 conversations/month = $12) this is a negligible risk. Sprint E plan: Postgres + real-time rollup for sub-minute enforcement.

---

## Layer 3: 80% Alert — Design Decisions

The alert fires at 80% rather than 100% for two reasons:

1. **Human latency:** a tenant at 80% of their $20 budget still has $4 left, enough time for the operator to contact them and offer an upgrade before the cap hits.
2. **No-spam guarantee:** the alert has a `last_alerted_at` timestamp per tenant. If the tenant stays between 80–99% for multiple aggregator cycles, only one alert fires per day. Without this, a tenant at 81% would get 4 Telegram messages per day.

---

## Layer 4: Idle Execution Reduction (−90%)

**Before optimization (pre-commercial, 0 real clients):**

| Workflow | Frequency | Executions/month |
|---|---|---|
| synthetic-probe | Every hour | 720 |
| module-retry-engine | Every 5 min | 8,640 |
| backup-exporter | Daily | 30 |
| analytics workflows | On demand | ~0 |
| **Total** | | **~9,390** |

**After optimization:**

| Workflow | Frequency | Executions/month |
|---|---|---|
| synthetic-probe | Unpublished (pre-launch) | 0 |
| module-retry-engine | Every hour | 720 |
| backup-exporter | Weekly | ~4 |
| module-cost-aggregator | Every 6h | 120 |
| security-maintenance | Daily | 30 |
| billing-export | Monthly | 1 |
| **Total idle** | | **~936** |

**Reduction: −90% (9,390 → 936 executions/month)**

All changes are reversible. `synthetic-probe` is ready to reactivate before the first paying client. `module-retry-engine` reverts to 5-minute schedule when there's real traffic to retry. The optimization buys low burn rate during the pre-commercial validation period.

---

## Layer 5: Monthly Billing Export

`billing-export` runs on the **1st of each month at 08:00** and can be triggered on-demand with a `{ month: "2026-06" }` parameter:

```
For each active tenant:
1. Read tenant_usage for the given month
2. Read plan pricing from billing_plans
3. Calculate:
   - revenue    = plan monthly price (CLP)
   - cost_usd   = total_cost_usd from tenant_usage
   - cost_clp   = cost_usd * fx_rate
   - margin     = (revenue - cost_clp) / revenue
   - roi        = (revenue - cost_clp) / cost_clp
   - leads      = count from analytics_events where eventType='lead_captured'
   - bookings   = count from analytics_events where eventType='booking_created'
4. Export to Google Sheets: billing-YYYY-MM tab
5. Return summary: { mrr, total_cost, gross_margin, tenants[] }
```

**Sample output at steady state (200 conversations/month, gpt-4.1):**

| Metric | Value |
|---|---|
| Cost per conversation | ~$0.06 USD |
| Monthly AI cost per tenant | ~$12 USD |
| Plan revenue (Esencial plan) | $149,000 CLP (~$160 USD) |
| Gross margin | >90% |
| ROI on AI spend | ~13× |

At 10 tenants on the Esencial plan: MRR ~$1,600 USD, AI cost ~$120 USD, gross margin >92%.

---

## Layer 6: Human-in-the-Loop Cost Profile

Pausing the agent for a conversation was evaluated for cost impact before shipping — a supervision feature that quietly increased execution volume would defeat its own purpose.

| Action | Execution cost | Trigger |
|---|---|---|
| Pause / resume toggle | 1 execution + 1 sub-execution (session write) | Operator click — human-paced, not per-message |
| Manual reply sent | 1 execution + 1 sub-execution | Operator click |
| Message received while paused | 1 execution + 1 sub-execution (session write) — **no engine, no dispatcher** | Per inbound message |
| Auto-resume scan | 1 execution every 30 min, regardless of whether anything is overdue | Schedule |

A paused conversation is **cheaper per message** than a normal one: 2 sub-executions instead of the 6–7 a GPT-4.1 answer with RAG normally costs, because the agent engine and module dispatcher are never invoked. The only new fixed cost is the auto-resume scheduler — 48 executions/day, in the same order of magnitude as `module-cost-aggregator`. No polling was added anywhere: the dashboard's live view rides on Supabase Realtime (push, not pull), and there is no workflow that periodically checks "is there anything to show."

---

## The Full Cost Control Stack at a Glance

```
Every AI call
      │
      ▼
module-cost-tracker (per-call metering → token_usage)
      │
      ▼ (every 6h)
module-cost-aggregator (rollup → tenant_usage + 80% alert)
      │
      ▼ (every request)
core-router: spend cap gate
      │
   over cap? → HTTP 402 (non-retryable, user notified)
   under cap? → proceed to agent
      │
      ▼ (monthly)
billing-export (MRR + margin + ROI → Google Sheets)
```

No tenant can run up an unbounded bill. No alert fires more than once per day. No idle workflows burn quota during the pre-commercial phase. Every dollar spent is tracked to the tenant and model that generated it.
