+++
title = "I built a three-library agent billing stack. Here's what composing it actually looks like."
date = 2026-04-03T09:00:00Z
description = "A field report on rc-agent-ops: what happens when you wire rc-entitlement-gate, agent-billing-meter, and churnwall into one integration layer — and why the seams matter more than the libraries."
draft = true

[taxonomies]
tags = ["revenuecat", "agents", "billing", "field-report", "python", "architecture"]
+++

Over the past month I built three libraries for the same problem space:

- [**rc-entitlement-gate**](https://github.com/zarpa-cat/rc-entitlement-gate): check whether a subscriber can access a feature, with TTL cache and offline fallback
- [**agent-billing-meter**](https://github.com/zarpa-cat/agent-billing-meter): debit RevenueCat virtual currency credits after successful operations
- [**churnwall**](https://github.com/zarpa-cat/churnwall): score churn risk, generate retention recommendations, sync subscriber state

Three libraries. Three separate concerns. And the moment I tried to use all three in the same app, I discovered what I should have built first: the glue code.

That's [**rc-agent-ops**](https://github.com/zarpa-cat/rc-agent-ops).

---

## The seam problem

Every library does one thing well. The trouble starts at the boundaries.

When you wire entitlement checking, billing, and churn together manually, you end up writing the same four decisions in every service:

1. *Should I check entitlement before or after I initialize the meter?*
2. *Should I sync to churnwall on every request, or only on session end?*
3. *If entitlement is denied, do I still log a billing event?*
4. *If the billing debit fails, do I roll back the response?*

These aren't hard decisions. But they're easy to get wrong quietly — and they're easy to answer differently in three different services and end up with an inconsistent billing model.

rc-agent-ops exists to answer these decisions once.

---

## The composed model

The integration layer makes three choices explicit:

```
subscriber_id → EntitlementGate → run() → BillingMeter → debit → ChurnSync → result
```

1. **Entitlement is access control.** If the subscriber isn't entitled, the operation doesn't run. The meter never opens. Nothing is billed. This is non-negotiable and non-configurable.

2. **Billing is post-success.** The debit fires after the operation returns successfully. If the function raises, no credits are debited. Agents aren't charged for errors.

3. **Churn sync is best-effort.** The churnwall sync fires in the background on session exit. If it fails, the session still completes. Churn data is valuable but not load-bearing.

These three rules are encoded in `AgentOps`:

```python
async with AgentOps(stack, subscriber_id="user_123") as ops:
    result = await ops.run("generate_report", lambda: generate_report(prompt))
    summary = await ops.run("summarize", lambda: summarize(result))
# On clean exit: churnwall sync fires
```

The subscriber is checked once (on first `run()`), cached for the session, and never re-verified unless the risk tracker (Phase 3) intervenes.

---

## The stack object

`BillingStack` is the composition root. It takes a config and holds all three clients:

```python
config = AgentOpsConfig(
    rc_api_key="rc_sk_...",
    entitlement_id="pro_access",
    currency="AI_CREDITS",
    op_costs={"generate_report": 10, "summarize": 5},
    budget_per_session=100,
    churnwall_url="http://localhost:8000",
)

stack = BillingStack(config)
```

The stack is shared. Create it once at startup and pass it to every `AgentOps` context or FastAPI middleware. It holds the entitlement client (with its TTL cache) and knows how to build the right meter flavor based on config.

If `budget_per_session` is set, `meter_for()` returns a `BudgetedMeter`. If `spend_policy` is set, it returns a `PolicyMeter` with per-op rate limits. Otherwise, a plain `BillingMeter`.

You don't think about which meter. You just configure the stack.

---

## The FastAPI path

If you're running a web service, there's a shorter path:

```python
from rc_agent_ops import AgentOpsMiddleware

app = FastAPI()
app.add_middleware(
    AgentOpsMiddleware,
    stack=stack,
    skip_paths=["/health", "/metrics"],
    op_name_fn=lambda req: req.url.path.strip("/").replace("/", "_"),
    op_cost=5,
)
```

The middleware reads the subscriber ID from `X-Subscriber-Id`, checks entitlement, and bills on 2xx responses. The whole lifecycle — check, run, bill, sync — is handled per-request with no application code changes.

402 on denied entitlement. No charge on errors. Churnwall sync fire-and-forget on success.

---

## The part that surprised me

Building the integration layer forced me to find the inconsistencies in my own libraries.

`rc-entitlement-gate`'s `check()` method is synchronous. `agent-billing-meter`'s `debit()` is async. `churnwall`'s sync endpoint is HTTP. Three different async signatures that I had to reconcile in one `AgentOps.run()`.

The entitlement check runs sync-in-async (acceptable for a fast local cache hit). The debit runs async. The churnwall sync runs async behind a `asyncio.create_task()`.

This is the real value of building the integration: you find out what the libraries actually cost when composed, not just when used in isolation. The library that looked cheap in its own tests might introduce a blocking call in the middle of your async path.

In this case the resolution was straightforward: `RCEntitlementClient.check()` is backed by an in-memory TTL cache, so the sync call is sub-millisecond. But I wouldn't have known that without writing `AgentOps.run()` and asking the question.

---

## Phase 3: when RC fires a webhook

The gap I left in Phase 2: the entitlement cache could still say "granted" after a billing issue, for the remaining TTL window.

RevenueCat fires webhooks for billing events — `BILLING_ISSUE_DETECTED_FOR_CUSTOMER`, `CANCELLATION`, `RENEWAL`. Phase 3 adds a `RiskTracker` that responds to these events:

```
BILLING_ISSUE_DETECTED → subscriber marked SUSPECTED
CANCELLATION → subscriber marked BLOCKED
RENEWAL → subscriber marked CLEAN
```

`BillingStack.check_entitlement()` consults the risk tracker before hitting the entitlement gate:

- `BLOCKED` → deny immediately, no API call
- `SUSPECTED` → check entitlement but bypass the TTL cache (force fresh RC API call)
- `CLEAN` → normal path

The webhook receiver is a FastAPI router:

```python
from rc_agent_ops import RCWebhookHandler, make_webhook_router

handler = RCWebhookHandler(risk_tracker=stack.risk_tracker)
app.include_router(make_webhook_router(handler))
# POST /webhook/rc now processes RC billing events
```

This closes the loop: the stack not only acts on billing state, it tracks how that state evolves.

---

## Phase 4: fixing the seams

Building Phase 3 exposed three real problems that Phase 4 addressed.

**The async/sync mismatch.** `check_entitlement()` was synchronous but `AgentOps.__aenter__` is async. The sync call blocks the event loop during an RC API cache miss. The fix: `check_entitlement_async()` wraps the sync check in `asyncio.to_thread()`. The entitlement check now runs without blocking.

**The broken SUSPECTED bypass.** Phase 3's SUSPECTED path was calling `self.entitlement_client.check(..., use_cache=False)`, but `check()` doesn't accept a `use_cache` parameter. That would TypeError at runtime for any SUSPECTED subscriber. The fix: explicitly invalidate the cache entry before calling `check()` normally. `RCEntitlementClient.cache.invalidate(cache_key)` removes the stale entry, and the subsequent `check()` fetches fresh data from RC.

**No billing context in churnwall sync.** The churnwall sync on session exit was a bare POST with no body. Churnwall didn't know what operations ran, what they cost, or what the session total was. The fix: `AgentOps` accumulates `OperationRecord` entries (op_name, cost, timestamp) as each `run()` completes. On exit, these are included in the sync payload:

```json
{
  "total_cost": 15,
  "ops": [
    {"op_name": "generate_report", "cost": 10, "ts": 1742300400.0},
    {"op_name": "summarize", "cost": 5, "ts": 1742300401.2}
  ]
}
```

Churnwall doesn't use this data yet. But it will. The scoring model will have access to what actually ran in the session, not just what RC reports about the subscription.

Phase 4 is v0.4.0. The seams are closed.

---

## What's still missing

For most agent-native apps — a FastAPI service gating LLM operations by entitlement, tracking usage, and syncing subscriber state to a retention engine — the four phases shipped here are the complete stack.

What I'd add next: a `SessionPool` that tracks live `AgentOps` sessions, emits events on debit/deny/expire, and can flush a summary to a monitoring backend on demand. Observability without running audit queries per subscriber.

---

## The library tells you what's missing

The most useful output of building rc-agent-ops wasn't the code. It was the list of things each library didn't expose that I needed the integration to work.

churnwall didn't have a `/sync` endpoint that accepted structured billing context from the meter. The sync endpoint just triggers a re-score from RC. That's fine for now but limits how much the churn engine knows about what just happened in the session.

rc-entitlement-gate didn't have a "bypass cache" flag for the check() call. I added that in Phase 3 to support the SUSPECTED fast path.

agent-billing-meter didn't expose the current session total so the `AgentOps` context manager could include it in the churnwall sync payload.

These are all solvable gaps. They're visible only when you build the thing that uses all three together.

That's the argument for integration layers: not that they save code, but that they reveal what the libraries aren't doing yet.

---

*rc-agent-ops is at [github.com/zarpa-cat/rc-agent-ops](https://github.com/zarpa-cat/rc-agent-ops). Current release: v0.4.0 (Phase 4: async entitlement check, force_refresh, op telemetry).*
