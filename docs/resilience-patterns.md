# Resilience Patterns

Numen AI implements four complementary patterns: three guarantee no message is permanently lost, and a fourth guarantees a human-paused conversation never stays silenced indefinitely. This document describes each pattern with exact parameters and explains how they interact.

---

## The Guarantee

> A message that enters the platform either gets a response delivered to the user, or ends up in the Dead Letter Queue where it can be manually inspected and replayed. There is no silent discard. A conversation that a human takes over is never left unattended for more than a bounded window.

This is enforced by the combination of: retry with backoff for transient failures, circuit breaker to stop amplifying load on a broken downstream, DLQ as the terminal catch for anything that exhausts retries or hits a permanent error, and auto-resume as the self-healing backstop for forgotten human handoffs.

---

## Pattern 1: Exponential Backoff Retry Queue

### What it handles
Transient failures: network timeouts, OpenAI API 429 (rate limit), temporary n8n Cloud hiccups, transient DataTable write failures. These are errors where retrying after a delay is likely to succeed.

### How it works

`module-error-recovery` is the platform's global Error Trigger. Any workflow execution that fails unhandled routes through it. The classification logic:

```
if error is retryable (timeout / 5xx / 429 / network):
    → write to retry_queue with attempt_count=1 + next_retry_at = now + 2min
else (4xx client errors, 402 spend cap, 403 unauthorized, 422 validation):
    → write directly to failed_messages (DLQ)
```

`module-retry-engine` runs on a **5-minute schedule** and processes `retry_queue`:

```
for each pending entry where next_retry_at <= now:
    attempt_count = entry.attempt_count

    if attempt_count == 1: next_retry_at = now + 2 min   (retry 1)
    if attempt_count == 2: next_retry_at = now + 10 min  (retry 2)
    if attempt_count == 3: next_retry_at = now + 30 min  (retry 3)

    call core-router with original payload
    
    if success:
        → remove from retry_queue
        → log RETRY_SUCCESS in recovery_log
    
    if fail and attempt_count < 3:
        → update attempt_count++ in retry_queue
    
    if fail and attempt_count == 3:
        → move to DLQ (failed_messages)
        → log RETRY_EXHAUSTED in recovery_log
```

### Parameters

| Parameter | Value |
|---|---|
| Max attempts | 3 |
| Retry 1 delay | 2 minutes |
| Retry 2 delay | 10 minutes |
| Retry 3 delay | 30 minutes |
| Check frequency | Every 5 minutes |
| Retry payload | Full original UMM (channel, user, message, tenantId) |

### Why exponential backoff?

A 429 from OpenAI during a traffic spike won't resolve in 5 seconds. A 2-minute initial delay gives the upstream time to recover without flooding it with retries. The 30-minute final delay catches scenarios where the issue takes longer (provider degradation, network partition) without holding the message in limbo indefinitely.

---

## Pattern 2: Circuit Breaker

### What it handles
Prevents cascading failure when `core-router` itself is broken. Without a circuit breaker, every incoming message would trigger a retry storm — amplifying load on a system that's already failing.

### How it works

State is persisted in `platform_security` DataTable under key `circuit_core_router`. The breaker has three states: **CLOSED** (normal), **OPEN** (failing fast), **HALF-OPEN** (testing recovery).

```
CLOSED → OPEN:
    5 consecutive failures detected by module-error-recovery
    → set circuit_core_router.state = 'open'
    → set opened_at = now
    → send Telegram alert to admin

OPEN behavior:
    module-retry-engine reads circuit state before processing retry_queue
    if circuit is OPEN → skip all retries, return immediately
    (messages stay in retry_queue, not retried until circuit closes)

OPEN → HALF-OPEN:
    After 10 minutes (configurable), circuit moves to HALF-OPEN
    → allow one probe request through

HALF-OPEN → CLOSED (recovery):
    If probe succeeds → set state = 'closed', reset failure count
    → send Telegram alert: circuit recovered

HALF-OPEN → OPEN (still failing):
    If probe fails → set state = 'open', reset timer
```

### Parameters

| Parameter | Value |
|---|---|
| Failure threshold | 5 consecutive failures |
| Recovery probe interval | 10 minutes |
| State storage | platform_security DataTable (persistent across executions) |
| Admin notification | Telegram on OPEN and on CLOSED (recovery) |

### Why persistent state?

n8n workflows are stateless between executions. Storing circuit state in a DataTable means the breaker survives workflow restarts, n8n Cloud deployments, and even if the retry engine itself crashes mid-execution. The state is the ground truth, not in-memory counters.

---

## Pattern 3: Dead Letter Queue (DLQ)

### What it handles
Messages that either (a) hit a non-retryable error immediately (402, 403, 422) or (b) exhausted all 3 retry attempts without success. These messages cannot be auto-recovered and require human review.

### Structure

Every DLQ entry in `failed_messages` contains:

```json
{
  "id": "dlq_uuid",
  "tenantId": "tenant_xxx",
  "channel": "whatsapp",
  "originalPayload": { ...full UMM... },
  "errorCode": "OPENAI_TIMEOUT",
  "errorMessage": "...",
  "httpStatus": 504,
  "attempts": 3,
  "firstFailedAt": "2026-06-24T22:03:05Z",
  "lastFailedAt": "2026-06-24T22:33:05Z",
  "status": "pending",
  "retryable": false
}
```

### Operations (via module-dlq workflow)

| Action | Behavior |
|---|---|
| `list` | Returns all pending DLQ entries, optionally filtered by tenantId or channel |
| `get` | Returns a single entry with full payload for inspection |
| `replay` | Re-injects the original UMM into core-router. If success: marks entry `replayed`. If fail: increments a `replay_attempts` counter, logs to `recovery_log`. |
| `discard` | Marks entry `discarded` with a reason. Logged to `recovery_log`. |

### Error routing rules

| Error type | Routing |
|---|---|
| HTTP 402 (Spend Cap) | → DLQ immediately (non-retryable by design) |
| HTTP 403 (Unauthorized) | → DLQ immediately |
| HTTP 422 (Validation error) | → DLQ immediately |
| HTTP 5xx (server errors) | → Retry queue |
| Network timeout | → Retry queue |
| OpenAI 429 | → Retry queue |
| Retry exhausted (3 attempts) | → DLQ |

---

## Pattern 4: Auto-Resume (Self-Healing for Human-in-the-Loop)

### What it handles

A different kind of "failure": a human operator pauses the agent to handle a conversation personally, then forgets to hand it back. Unlike the other three patterns, nothing here is broken — the agent is behaving exactly as told — but an unattended lead is functionally the same outcome as a lost message if nobody resumes the bot.

### How it works

```
Every 30 minutes:
    read all sessions where humanPaused = true
    for each: if (now - pausedAt) >= 4 hours:
        → resume the agent (humanPaused = false)
        → notify the dashboard in real time
```

Same philosophy as the circuit breaker's HALF-OPEN auto-recovery: the system doesn't wait indefinitely for a human decision it can safely make itself after a bounded timeout.

### Parameters

| Parameter | Value |
|---|---|
| Scan interval | Every 30 minutes |
| Resume threshold | 4 hours since paused |
| Scope | Per-conversation, not per-tenant — one paused lead never blocks others |

### Why 4 hours?

Long enough that a genuine mid-conversation handoff (operator stepping away for lunch, a longer negotiation) isn't interrupted. Short enough that a forgotten pause doesn't silence the agent for an entire lead overnight. The threshold is a single constant in the scheduler — trivial to tune per the operator's actual response patterns once there's production data to look at.

---

## How the Three Message-Delivery Patterns Work Together

(Auto-resume operates on a separate axis — a human-paused conversation, not a failed message — so it isn't part of the delivery flow below; see Pattern 4 above.)

```
Incoming message
      │
      ▼
core-router
      │
  ┌───┴────────────────┐
  │ success            │ failure
  │                    ▼
  │             module-error-recovery
  │                    │
  │         ┌──────────┴──────────┐
  │         │ retryable?          │ non-retryable?
  │         ▼                     ▼
  │    retry_queue              DLQ (final)
  │         │
  │         ▼
  │    module-retry-engine (every 5 min)
  │         │
  │    check circuit breaker
  │         │
  │    ┌────┴──────────────────┐
  │    │ OPEN: skip            │ CLOSED: attempt retry
  │    │ (wait for recovery)   │
  │    │                       │
  │    │              ┌────────┴──────────┐
  │    │              │ success           │ fail (< 3)
  │    │              ▼                   ▼
  │    │          delivered           update attempt_count
  │    │                               schedule next retry
  │    │
  │    │                     fail (attempt 3)
  │    │                           ▼
  │    │                         DLQ (final)
  ▼
user receives response
```

**Failure scenarios and what happens:**

| Scenario | Outcome |
|---|---|
| OpenAI is slow (429) | Retries at 2/10/30 min. Likely succeeds on retry 1. |
| n8n Cloud hiccup | Circuit breaker stays CLOSED (single failure). Retries normally. |
| Core-router down for 20 min | Circuit opens at 5 failures. Retry queue pauses. Messages preserved. When router recovers, circuit closes and replay begins. |
| Tenant over budget | HTTP 402 → DLQ immediately. User gets spend cap message. No retry. |
| Persistent provider outage | All 3 retries fail over ~40 min. Message lands in DLQ. Manual replay possible when provider recovers. |

No message is silently discarded. Every failure is audited. Every DLQ entry can be replayed by an operator.
