# Numen AI — Multi-Tenant AI Agent Platform

![Status](https://img.shields.io/badge/status-LIVE%20%E2%80%93%20Production-brightgreen)
![Platform](https://img.shields.io/badge/platform-n8n%20Cloud-orange)
![AI](https://img.shields.io/badge/AI-GPT--5.6%20Luna-blue)
![Tenants](https://img.shields.io/badge/tenants-multi--tenant-purple)

**A production AI SaaS platform serving real clients via WhatsApp, Telegram, Instagram, Messenger and Web Chat — engineered with Gateway pattern, multi-tenant isolation, full FinOps control, and enterprise-grade resilience. Not a tutorial. Not a prototype.**

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                        INBOUND CHANNELS                            │
│                                                                    │
│   WhatsApp    Telegram   Web Chat   Instagram   Messenger   Gmail* │
│   (text+audio) (text)    (embedded)  (Meta app in review) (paused) │
└───────────┬────────────────┬───────────────┬──────────────────────┘
            │                │               │
            ▼                ▼               ▼
┌────────────────────────────────────────────────────────────────────┐
│              CHANNEL ADAPTERS  ─  Universal Message Model (UMM)    │
│   Normalize every channel to: { tenantId, channel, user,           │
│   message, security, timestamp }  ─  signed with X-Hub-Signature   │
└──────────────────────────────┬─────────────────────────────────────┘
                               │
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│                         CORE ROUTER                                │
│   1. Auth        X-Platform-Key validation                         │
│   2. Sanitize    input truncation + injection stripping            │
│   3. Rate Limit  20 req/min/user · 30/session · 100/tenant         │
│   4. Tenant Gate tenant_registry lookup + status check            │
│   5. Spend Cap   HTTP 402 (non-retryable) when budget exceeded     │
└──────────────────────────────┬─────────────────────────────────────┘
                               │
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│                    CORE AGENT ENGINE v6                            │
│                                                                    │
│  ┌──────────────────┐   ┌─────────────────┐   ┌────────────────┐  │
│  │  Session Memory  │   │   GPT-5.6 Core  │   │    Tenant      │  │
│  │  ~50 msg window  │   │   Tool Calling  │   │  Config Loader │  │
│  │  per user/channel│   │   + RAG context │   │  (modules,     │  │
│  └──────────────────┘   └────────┬────────┘   │   prompt, KB)  │  │
│                                  │            └────────────────┘  │
└──────────────────────────────────┼─────────────────────────────────┘
                                   │  execute_module tool
                                   ▼
┌────────────────────────────────────────────────────────────────────┐
│              MODULE DISPATCHER v2  (tenant-isolated)               │
│   Validates tenantId + moduleId against tenant_modules before      │
│   routing. Returns HTTP 403 SECURITY_MODULE_FORBIDDEN on           │
│   unauthorized access. workflowId resolved at runtime from         │
│   module_registry DataTable — zero coupling.                       │
└───┬───────────────┬───────────────────┬────────────────────────────┘
    │               │                   │
    ▼               ▼                   ▼
┌────────┐   ┌──────────────┐   ┌────────────┐
│ RAG v3 │   │ Calendar v3  │   │  Leads v2  │
│ KB     │   │ Google Cal.  │   │ G. Sheets  │
│ +cache │   │ real booking │   │ real CRM   │
└────────┘   └──────────────┘   └────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────────────┐
│               TELEMETRY SINK  (fire-and-forget)                    │
│   Single sub-execution consolidating: execution_logs +             │
│   analytics_events + token_usage + cost calculation                │
│   Does NOT block the response path if it fails.                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## What This Platform Does

| Capability | Detail |
|---|---|
| **AI Agent** | GPT-5.6 Luna (Responses API, tool calling) with RAG over business KB and ~50-message session memory per user per channel. Model overridable per tenant via config — zero redeploy. |
| **Multi-tenant** | Full tenant isolation by `tenantId` across 22+ DataTables. Onboarding in a single API call. |
| **Omnichannel** | WhatsApp (text + audio with Whisper transcription), Telegram, Web Chat — live. Instagram DM and Messenger — webhooks verified, pending Meta app review. Gmail — ready. |
| **RAG** | Pluggable backend per tenant: Google Sheets KB + in-memory vector store (default), or Supabase pgvector with hybrid search (vector ⊕ Postgres FTS fused via RRF), LLM query rewriting and reranking |
| **Self-service KB Center** | Clients edit their own knowledge base from the dashboard: draft/publish editorial flow on Supabase with RLS, PDF/DOCX import, and a website crawler (SSRF guards, robots.txt, quotas, checksums) that keeps the KB in sync with the client's site |
| **Business modules** | Calendar booking (Google Calendar), lead capture (Google Sheets), reminders (Telegram daily 8AM) |
| **Latency** | ~12 s E2E with RAG. RAG: 4.3 s cold / 1.7 s cached. *Measured on GPT-4.1; not re-measured since the GPT-5.6 migration.* |
| **Executions/msg** | 6–7 (direct) / 9–10 (with module) — down from 13 after Sprint C optimization |
| **Audio** | WhatsApp voice messages → Whisper API → text → agent, transparent to the user |
| **Live supervision** | Client-facing dashboard streams every stage of the agent's work (received → thinking → tool call → answered) via Supabase Realtime, without exposing model chain-of-thought |
| **Human-in-the-loop** | Pause the agent on a single conversation from the dashboard — zero AI cost while paused, reply manually (WhatsApp), auto-resume after 4h if the operator forgets |

---

## Resilience: No Message Is Ever Permanently Lost

Three patterns work together to guarantee delivery:

### 1. Exponential Backoff Retry Queue
- Retryable failures (network timeouts, transient AI errors) go to `retry_queue`
- Retry schedule: **2 min → 10 min → 30 min** (max 3 attempts)
- After 3 failures → promoted to DLQ for manual review

### 2. Dead Letter Queue (DLQ)
- Catches: non-retryable errors (HTTP 402, 403, 422) + retry-exhausted messages
- Every entry logged with: `tenantId`, `channel`, `error_code`, `payload`, `timestamp`, `attempts`
- Manual replay via `module-dlq` workflow: replay action re-injects into `core-router` with full context

### 3. Circuit Breaker
- Opens after **5 consecutive core-router failures** (state persisted in `platform_security` DataTable)
- When open: rejects new messages immediately (fast-fail) instead of hammering a broken downstream
- Auto-recovery: next successful execution closes the breaker
- Admin alert via Telegram on state change (open/closed)

Together: transient errors retry automatically, permanent errors are captured for manual action, and cascading failure is cut off at the source.

---

## FinOps: Full Cost Visibility and Enforcement

| Metric | Value |
|---|---|
| **Idle execution reduction** | −90% (9,330 → 936 executions/month) |
| **Spend Cap enforcement** | HTTP 402 (non-retryable) — adapter never retries this |
| **Usage alert** | Telegram notification at 80% of monthly budget (1×/day, no spam) |
| **Cost per conversation** | ~$0.06 (GPT-4.1, 7 messages) / ~$0.012 (GPT-4o-mini) — *pre-migration figures* |
| **Gross margin** | >90% at ~200 conversations/month per tenant |
| **Spend cap lag** | ≤6h (cost-aggregator runs every 6h) |
| **Billing export** | Automated monthly: MRR, cost/tenant, margin, ROI → Google Sheets |

**How it works:** Every AI call writes to `token_usage` (chars/4 estimation). `module-cost-aggregator` runs every 6h and upserts `tenant_usage` rollup. `core-router` checks `tenant_usage` against the plan's spend cap before every request — if exceeded, returns HTTP 402 immediately, which the channel adapter marks as non-retryable.

---

## Human-in-the-Loop: Supervision Without Losing Control

An AI agent handling real leads needs an escape hatch. Three pieces work together, all authenticated with the operator's own Supabase JWT — never the platform's internal shared secret:

- **Live dashboard** — every message's lifecycle (received, thinking, tool call, answered) streams to the operator's browser in real time via Supabase Realtime. No chain-of-thought is ever exposed, only operational stage.
- **Pause gate, checked before the LLM call, not after** — `core-router` checks a `humanPaused` flag right after loading the session, *before* building the prompt or invoking the model. A paused conversation costs **zero AI tokens**: the message is saved to session memory (so context isn't lost) and the operator's channel adapter receives a static acknowledgment instead of a model response — same response contract as a normal answer, so the WhatsApp/Telegram/Web Chat adapters needed **no changes** to support pausing.
- **Auto-resume** — a scheduled workflow reactivates any conversation paused for more than 4 hours, so an operator forgetting to unpause never permanently silences the bot for a lead.

Manual reply (operator types the answer, agent stays silent) currently ships for WhatsApp — same JWT-authenticated pattern, reusing the channel's existing send credential.

---

## Advanced RAG in Production: the Huberman Tenant

The multi-tenant RAG layer is exercised beyond the commercial use case by a dedicated tenant running a coaching bot over 400+ episodes of podcast content: dual-layer ingestion (episode summaries + 17,899 transcript chunks cut on the show's real chapter timestamps), hybrid retrieval (pgvector cosine ⊕ Postgres full-text fused with Reciprocal Rank Fusion), LLM query rewriting and reranking, YouTube deep-links to the exact minute of the source, and a quantitative eval harness (golden set, Recall@8 / MRR) — all sharing this platform's router, agent engine, and module dispatcher with zero changes for other tenants.

**→ Dedicated repo with architecture, evals, and two production debugging case studies: [huberman-rag-bot](https://github.com/lindermannn/huberman-rag-bot)**

---

## Workflow Inventory (70 total, 50 active in production)

| Category | Workflow | Role |
|---|---|---|
| **Core hot-path** | `core-router` | Single entry point: auth, rate limit, tenant gate, spend cap |
| | `core-agent-engine-v6` | GPT-5.6 Luna agent: Responses API, tool calling, RAG context injection, session memory |
| | `core-config-loader` | Loads tenant config, modules, KB location per request |
| | `core-memory-manager` | Session lifecycle: get/create/update with DataTable upsert |
| | `core-logger` | Structured logging to `execution_logs` |
| | `core-error-handler` | Error classification: retryable vs non-retryable, HTTP status mapping |
| **Business modules** | `module-rag-v3` | RAG: KB lookup via Google Sheets + embedding cache |
| | `module-calendar-v3` | Google Calendar: real booking, cancellation, availability check |
| | `module-leads-v2` | Lead capture: name/email/phone extraction → Google Sheets |
| | `module-dispatcher-v2` | Tenant-isolated routing: validates auth before executing any module |
| | `module-reminders` | Daily 8AM: tomorrow's appointments via Telegram |
| | `module-registry` | CRUD for module_registry DataTable |
| **Channel adapters** | `channel-whatsapp-adapter-v2` | WhatsApp: X-Hub-Signature-256, dedup by wamid, audio routing |
| | `audio-transcriber` | WhatsApp audio → Whisper → text (sub-workflow) |
| | `channel-telegram-adapter` | Telegram Trigger → UMM → core-router → reply |
| | `channel-webchat` | n8n hosted chat → UMM → core-router |
| | `channel-facebook-adapter-v2` | Facebook Messenger: security hardening, dedup by mid |
| | `channel-instagram-adapter-v2` | Instagram DM: security hardening, dedup by mid |
| | `channel-email-adapter-v2` | Email: dedup by Message-Id (active, pending mailbox) |
| **Resilience** | `module-error-recovery` | Error Trigger → classify → retry_queue or DLQ |
| | `module-retry-engine` | Schedule 5min: process retry_queue with backoff + circuit breaker |
| | `module-dlq` | DLQ CRUD: ingest / list / get / replay / discard |
| **Observability** | `telemetry-sink` | Single fire-and-forget: logs + analytics + cost in one execution |
| | `analytics-collector` | Business events ingestion (lead_captured, booking_created, etc.) |
| | `analytics-engine` | Event normalization + GPT cost estimation → analytics_events |
| | `Audit Logger` | Security audit trail → audit_logs + Telegram on critical events |
| **Security** | `Tenant Isolation Validator` | tenant_registry + tenant_modules authorization check |
| | `Rate Limiter` | Bucket-key counter: 20/min user, 30/session, 100/tenant |
| | `security-maintenance` | Daily 04:00: purges processed_messages >7d, rate_limit_counters >24h |
| **FinOps & billing** | `billing-export` | Monthly: MRR, cost, margin, ROI per tenant → Google Sheets |
| | `module-cost-tracker` | Per-call: token estimation → pricing lookup → token_usage insert |
| | `module-cost-aggregator` | Every 6h: aggregates token_usage → tenant_usage UPSERT |
| **Operations** | `tenant-provisioner` | Full tenant onboarding in 1 API call |
| | `tenant-manager` | Admin CRUD: create/list/get/update/set_status |
| | `tenant-module-manager` | Per-tenant module entitlements: activate/deactivate |
| | `ops-set-tenant-plan` | Hot plan change: free/starter/pro/enterprise without restart |
| | `ops-pricing-api` | Public pricing endpoint — single source of truth consumed by the commercial website |
| | `kb-builder` | AI-assisted KB ingestion via web form (GPT-4.1 structures raw text) |
| **Knowledge Center** | `ops-kb-command-api` | Authenticated API: KB drafts, answer preview, PDF/DOCX import |
| | `ops-kb-website-api` + `kb-website-crawler` | Website discovery and sync with SSRF guards, robots.txt, quotas, checksums |
| | `kb-source-sync-scheduler` | Scheduled re-sync of crawled sources |
| | `ops-kb-migrate-sheets-to-drafts` | One-shot migration: legacy Sheets KB → editorial drafts |
| | `ops-sync-tenant-profile` | Tenant profile sync between dashboard and platform config |
| **Human-in-the-loop** | `ops-toggle-human-pause` | Browser-facing webhook (Supabase JWT auth): pause/resume the agent for one conversation |
| | `ops-send-manual-reply` | Browser-facing webhook: operator sends a manual reply (WhatsApp) while paused |
| | `ops-auto-resume-scheduler` | Every 30 min: reactivates conversations paused for over 4h |
| **Backup** | `backup-exporter` | Nightly 03:30: 14 critical DataTables → Google Sheets snapshot |
| | `backup-exporter-table` | Sub-workflow: one table per execution to isolate memory |

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Orchestration** | n8n Cloud (workflow automation, 70 workflows) |
| **AI / LLM** | OpenAI GPT-5.6 Luna via Responses API (per-tenant agent core, overridable per tenant), text-embedding-3-small (RAG), Whisper (audio) |
| **Messaging** | WhatsApp Business Cloud API, Telegram Bot API, Meta Graph API (Instagram DM, Messenger) |
| **Data** | n8n DataTables (22 tables), Google Sheets (KB + billing + backup), Supabase pgvector (advanced RAG backend) |
| **Live supervision** | Supabase (Postgres + Realtime), row-level security scoped by tenant JWT |
| **Calendar** | Google Calendar API (real booking and cancellation) |
| **Web** | Next.js 16 on Vercel (commercial site + client dashboard + lead capture) |
| **Security** | X-Platform-Key (internal auth, centralized in a DataTable — zero hardcoded secrets after the July 2026 hardening pass across 9 workflows), X-Hub-Signature-256 (Meta webhooks), Supabase JWT (browser-facing actions), rate limiting |

---

## Channel Status

| Channel | Status | Notes |
|---|---|---|
| WhatsApp Business | **LIVE** (test-mode token) | Text + audio (Whisper). Real phone number. Migration from Meta's 24h test token to a permanent System User token in progress — incident of 2026-07-14 documented and runbooked. |
| Telegram | **LIVE** | Text. Admin alerts also routed here. |
| Web Chat | **LIVE** | Embedded via n8n hosted chat. |
| Gmail / Email | **Built — Paused** | Adapter built and tested E2E. Paused: needs dedicated mailbox to avoid answering personal email. |
| Facebook Messenger | **Connected** | Page linked, webhook verified, security hardening active. Pending Meta App Review (business verification) for public rollout. |
| Instagram DM | **Connected** | Business account linked, webhook verified, security hardening active. Pending Meta App Review. |

---

## Commercial Platform

Live SaaS offering 5 plans (Lite → Agency) with per-company pricing, done-for-you onboarding, and guaranteed spend cap:

**→ https://numen-ai.cl/

---

## See Also

- [Architecture Decisions](docs/architecture.md) — why Gateway pattern, UMM, tenantId isolation, fire-and-forget telemetry, JWT-scoped human intervention
- [Resilience Patterns](docs/resilience-patterns.md) — DLQ, Circuit Breaker, Exponential Backoff, auto-resume — with exact parameters
- [FinOps Strategy](docs/finops.md) — full cost control: spend cap, metering, idle reduction, margin model, human-in-the-loop cost profile

---

> Platform designed and built using **Claude Code + MCP** as the execution layer, with architectural design, failure mode analysis, and iterative optimization done independently.
