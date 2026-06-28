# Architecture Decisions

This document explains the key architectural choices in Numen AI and the reasoning behind each. Written to answer the questions a senior architect would ask.

---

## 1. Why the Gateway Pattern (core-router as single entry point)?

**The problem:** Six channels (WhatsApp, Telegram, Web, Email, Facebook, Instagram) each have different security models, payload formats, and delivery guarantees. Without a gateway, each adapter would need to implement auth, rate limiting, tenant resolution, and spend cap logic independently — guaranteeing drift and inconsistency over time.

**The decision:** All channels normalize to a Universal Message Model and call a single `core-router` webhook. Every security check, rate limit, tenant gate, and spend cap lives in one place.

**What this buys:**
- A security fix to rate limiting applies everywhere simultaneously
- A new channel only needs to implement UMM normalization — it inherits all platform guarantees for free
- A bug in spend cap enforcement is fixed once, not six times
- Observability is trivial: one workflow processes all traffic, so one execution trace shows the full picture

**The tradeoff:** `core-router` is a single point of failure. Mitigated by n8n Cloud's managed uptime, the circuit breaker that cuts off cascading retries when it's down, and the retry queue that replays once it recovers.

---

## 2. Why a Universal Message Model instead of per-channel logic?

**The problem:** WhatsApp sends `{messages[0].text.body, from, wamid}`. Telegram sends `{message.text, message.from.id, message.message_id}`. Web Chat sends `{chatInput, sessionId}`. If the agent engine knows about these formats, adding a new channel requires touching core logic.

**The decision:** Each adapter translates its native format to a canonical UMM before calling the router:

```json
{
  "tenantId": "tenant_lindermann",
  "channel": "whatsapp",
  "user": { "id": "5491112345678", "name": "Demo User" },
  "message": { "text": "...", "type": "text", "id": "wamid.xxx" },
  "security": { "signatureValid": true, "mode": "shadow" },
  "timestamp": "2026-06-24T22:03:05Z"
}
```

**What this buys:**
- `core-agent-engine` is channel-agnostic. It processes UMM and returns a text response. The adapter handles delivery.
- Session memory key is `{tenantId}_{channel}_{userId}` — works identically across all channels
- Analytics and logging are channel-aware without channel-specific code in core

**The tradeoff:** Adapters must implement UMM mapping correctly. A malformed UMM breaks the request. Mitigated by: input validation in `core-router`, structured adapter testing before activation, and deduplication by message ID (each adapter tracks its native ID to prevent duplicate processing on webhook retries).

---

## 3. Why tenantId-based isolation instead of separate n8n instances?

**The problem:** Multi-tenancy can be implemented as separate workflow stacks per customer (simple, but operationally expensive) or as shared infrastructure with tenant-level data isolation (operationally efficient, but requires discipline).

**The decision:** One shared stack, 22 DataTables partitioned by `tenantId`. Every DataTable query filters by `tenantId`. Every module call passes `tenantId` through to the dispatcher, which validates it against `tenant_modules` before routing.

**What this buys:**
- Adding a new tenant is a single API call to `tenant-provisioner` (writes to `tenant_registry`, `tenant_config`, `tenant_knowledge_base`, `tenant_modules`, `audit_logs`)
- Ops changes (cost aggregator, backup, security maintenance) run once and cover all tenants
- Billing export generates per-tenant cost and margin in a single scheduled execution

**The security boundary:** `module-dispatcher-v2` performs a two-check authorization on every module call: (1) is the `tenantId` valid? (2) is the `moduleId` enabled for this tenant? A 403 is returned and logged if either check fails. The audit trail in `audit_logs` records all access attempts with severity tagging.

**The tradeoff:** A bug in a DataTable filter could leak cross-tenant data. This was discovered once in production (a `config.tenant.tenantId` path error caused all requests to be processed as `tenant_demo`). Fixed by adding tenantId validation as an early assertion in the agent engine and by the dispatcher's explicit authorization check — defense in depth.

**Scalability ceiling:** DataTables are scanned with full reads for some operations. This scales linearly with tenant count and becomes a bottleneck around 100+ tenants. Sprint E plan: migrate to Postgres with indexed queries for `tenant_usage`, `conversation_sessions`, and `retry_queue`.

---

## 4. Why fire-and-forget telemetry?

**The problem:** Every message generates logging, analytics, and cost tracking needs. If these block the response path, they add latency on every request. If they fail, they either break the user's conversation or require complex retry logic on the hot path.

**The decision:** `telemetry-sink` is called as a fire-and-forget Execute Workflow (`waitForSubWorkflow: false`). The agent engine does not wait for it. It consolidates three writes (execution_logs, analytics_events, token_usage) into a single sub-execution.

**What this buys:**
- Zero latency impact on the response path — the user gets their answer while telemetry writes happen asynchronously
- A telemetry failure never breaks a conversation
- The consolidation from 3 separate sub-executions to 1 reduced total executions per message from 13 to 6–7 (−46%)

**The tradeoff:** If `telemetry-sink` fails, metrics are lost silently. There is no alert on telemetry failure today. Consequence: cost tracking may undercount temporarily; execution logs may have gaps. Acceptable pre-scale, but should be instrumented with a failure alert before operating at significant volume.

---

## 5. Why n8n Cloud instead of a traditional microservices backend?

**The honest answer:** Speed-to-production for a solo builder. n8n provides managed scheduling, webhook infrastructure, retry mechanics, credential storage, and a visual debugging tool (execution trace) that would take months to build from scratch.

**The architectural answer:** The platform is not bound to n8n by design. The `core-router` is a webhook — it could be fronted by API Gateway. DataTables are a managed key-value store — they could be replaced by Postgres without changing module logic. The UMM is a schema, not a framework. If the platform outgrows n8n, the migration path is incremental: replace DataTables with Postgres (Sprint E), optionally self-host n8n in queue mode, and eventually extract hot-path modules to Lambda or Cloud Run. The workflows document the business logic in a portable way.

**What was consciously traded:** Native code performance and full observability control, in exchange for operational simplicity and a 10× faster development cycle at early stage.
