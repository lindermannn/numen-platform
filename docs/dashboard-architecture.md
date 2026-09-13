# Dashboard Architecture

The agent backend documented in [`architecture.md`](./architecture.md) is one half of the platform. The other half is what a tenant's team actually looks at: a Next.js dashboard where an operator watches the agent work in real time, takes over a conversation, and manages the business data the agent draws on — catalog, inventory, orders, agenda, knowledge base.

This document covers that half: how it's built, and the decisions behind it.

## Stack

| Layer | Choice |
|---|---|
| Framework | Next.js 16, App Router, React 19, TypeScript strict |
| Styling | Tailwind CSS v4 + shadcn/ui (`@base-ui/react`) |
| Data | Supabase — Postgres, Realtime, Row-Level Security |
| Deploy | Vercel, `output: standalone` |

## 1. Why the dashboard talks to Supabase directly, not through a backend API

The dashboard's Supabase client is created with the **anon key** — the same public key any browser could use — not a privileged service-role key. There is no bespoke API layer in front of the database for tenant data: every query the dashboard makes is filtered by Postgres Row-Level Security, keyed on the tenant claim in the operator's JWT.

That means the isolation guarantee lives in one place — the RLS policies — instead of being re-implemented in every endpoint an API layer would need. An operator's browser session can only ever see rows for their own tenant, enforced by the database itself, regardless of what the frontend code does or doesn't check. The tradeoff: any new tenant-scoped table needs its RLS policy written and tested *before* the UI can safely query it — there's no application-layer fallback if a policy is missing or wrong.

## 2. Live supervision as an event log, not a polled status field

An operator doesn't just see "the agent is working" — they see the actual pipeline stage streamed live: message received → analyzed → a tool was called (knowledge base, calendar, stock, payment link, contact capture) → response sent. Each of those is a row appended to an event log (`conversation_events`, scoped by `tenant_id`), not a status field that gets overwritten.

That distinction matters for two reasons. First, Supabase Realtime subscribes to inserts on that table, so the dashboard updates by *receiving new events*, not by polling a mutable row — there's no race between "read the status" and "the status just changed." Second, an append-only log is itself an audit trail: what the agent did during a conversation is reconstructable after the fact, which a single overwritten status column would have discarded.

The stages are deliberately operational, never the model's reasoning — an operator sees *that* a tool was called, not the chain-of-thought that led there. Full detail on why is in `architecture.md`'s human-in-the-loop section.

## 3. One dashboard, several tenant-facing modules

The dashboard isn't one page — it's a set of modules a tenant's team uses day to day: agenda, catalog, inventory, orders, a self-service knowledge-base editor, and analytics. Each is its own route under `/dashboard`, each queries only what its own RLS policies expose. This is what makes the platform a SaaS product rather than a chatbot demo with a status page bolted on — the AI agent and the tools an operator uses to run their business are the same system, not two things wired together after the fact.

## 4. Design system: isolated on purpose

The premium tier's design tokens live in their own stylesheet, imported only where that surface is rendered — deliberately not merged into the global stylesheet. A palette change for one tier can't silently bleed into the rest of the product, and the reverse is also true: the base dashboard's styling can evolve without anyone needing to check whether it broke the premium surface. [`examples/design-tokens.css`](../examples/design-tokens.css) in this repo is the actual token file, unmodified.

## What's not here

The dashboard's business logic — order handling, inventory, catalog publishing, payment configuration per tenant — is real product code with real client data models, and stays in the private implementation. What's documented here is the architecture: how the pieces fit, and why, at the level of detail a technical reader needs to evaluate the work — the same standard the rest of this repo holds itself to.
