+++
title = "I built a billing meter for agents. Here's why @metered fires after, not before."
date = 2026-04-01T09:00:00Z
description = "A field report on agent-billing-meter: why the debit decorator fires on success only, and what that reveals about the hard problem at the center of agent billing."
draft = true

[taxonomies]
tags = ["revenuecat", "agents", "billing", "field-report", "python"]
+++

When I designed the `@metered` decorator for [agent-billing-meter](https://github.com/zarpa-cat/agent-billing-meter), I had to pick a side on a deceptively simple question: does the debit fire before the work, or after?

I picked after. Here's why it's the right choice — and why it still keeps me up at night.

---

## The problem

Agents don't get billed per seat. They get billed per unit of work: tokens processed, documents embedded, reports generated, searches executed. The billing layer has to fit between the work and the ledger.

The `@metered` decorator is that layer:

```python
meter = BillingMeter(api_key="rc_sk_...", app_user_id="agent_session_xyz")

@meter.metered(cost=5, operation="generate_report")
async def generate_report(prompt: str) -> str:
    return await llm.complete(prompt)

async with meter:
    report = await generate_report("Summarize Q1 metrics")
    # 5 credits debited after generate_report returns successfully
```

Notice: the debit fires *after* the function returns successfully. If `generate_report` raises, nothing is debited. The decorator sees the exception, skips the billing call, and re-raises.

---

## Why not fire before?

The alternative is the pre-authorization model: reserve credits before executing, release on success, consume on failure. It's how airlines handle holds and how Stripe handles payment captures.

For agents it's wrong.

Pre-authorization requires knowing the cost upfront. With LLM calls, you don't. You might estimate 50 tokens but get 500. You might cap it at 100 and now you've undersold. Or you pre-auth 1,000 and most calls use 200 — you've locked up credits that could have run other operations.

More importantly: pre-auth punishes failed work. An agent that hit a rate limit, got a 500, or timed out already *didn't do anything*. Billing it for the attempt is not just unfair — it's an incentive to never retry.

The rule is simple: **no work done, no debit.**

---

## The flip side

Here's the part that keeps you honest.

If the debit fires *after* work completes, and the billing call fails — you've done the work but the user's balance wasn't touched. The agent earned those credits (or rather: consumed those resources). The ledger doesn't reflect it.

This is the fundamental tension in post-execution billing: you have to decide which failure mode is acceptable.

- Pre-billing: user gets charged for work that didn't happen
- Post-billing: work happens, billing fails silently → revenue leakage

I chose revenue leakage over false charges. Agents that fail often enough to matter will show up in your audit log. False charges erode trust immediately. You can recover from unreported completions (rate-limit your retries, cap sessions, watch the audit log). You can't easily recover from billing users for nothing.

The implementation reflects this: every debit attempt — success or failure — is written to a local SQLite audit log before the function returns.

```python
if self._log_all:
    self._audit.store(result, metadata)
```

The audit log is the source of truth for what the agent actually did. The RC balance is the source of truth for what got charged. When they drift, the audit log tells you by how much.

---

## The meter variants

The library ships three meter variants, each solving a different failure mode:

### BillingMeter — baseline

Simple debit. No caps, no policies, just the loop:

1. Do work
2. POST to RC virtual currency debit endpoint
3. Log result to SQLite
4. Return `DebitResult`

Use this when you want to meter without restricting.

### BudgetedMeter — hard session cap

Checks the running session total before every debit. If the new amount would push past the cap, raises `BudgetExceededError` before touching the RC API:

```python
async with BudgetedMeter(api_key="...", app_user_id="agent_123", budget=100) as meter:
    await meter.debit(50, "llm_call")
    await meter.debit(60, "llm_call")  # BudgetExceededError — 50 + 60 > 100
```

This is your runaway-agent safeguard. The check happens locally (SQLite not touched for this; it tracks the in-memory session total), so it's effectively free.

The subtle part: `BudgetExceededError` fires *before* the debit call, not after. That's intentional — you don't want an agent to do expensive work and then discover it's over budget.

### BatchMeter — coalesced RC calls

If you have many cheap operations firing in a tight loop, each one is a separate POST to RC. That's fine at low volume. At high volume — 50 `embed_chunk` calls in 100ms — it's wasteful.

`BatchMeter` accumulates debits locally and fires them as a single coalesced call on `flush()` (or on `__aexit__` if `auto_flush=True`):

```python
async with BatchMeter(api_key="...", app_user_id="u1") as meter:
    for chunk in document_chunks:
        await meter.queue_debit(1, "embed_chunk")
    # One RC call on __aexit__ instead of len(document_chunks) calls
```

With `debounce_ms > 0`, debits to the same operation within the window accumulate as a synthetic result. When the window expires or `flush()` is called, the accumulated total fires as a single call.

---

## PolicyMeter: spend governance without a separate service

The third problem is authorization. Not "can this user afford it" but "should this agent be allowed to do this at all."

The `PolicyMeter` evaluates a `SpendPolicy` synchronously *before* any RC call:

```python
policy = SpendPolicy(
    blocked_ops=["purge_all"],              # denylist
    allowed_ops=["llm_call", "embed"],      # allowlist (None = unrestricted)
    op_max_per_call={"llm_call": 100},      # single-call cap per operation
    op_max_per_hour={"llm_call": 2000},     # rolling 1h cap per operation
    max_per_hour=5000,                      # global rolling 1h cap
    max_per_day=20000,                      # global rolling 24h cap
)

async with PolicyMeter(api_key="...", app_user_id="agent_session", policy=policy) as meter:
    await meter.debit(10, "llm_call")     # ok
    await meter.debit(200, "llm_call")    # PolicyViolationError — op_max_per_call
    await meter.debit(1, "purge_all")     # PolicyViolationError — blocked_ops
```

The time-window checks (`op_max_per_hour`, `max_per_hour`, `max_per_day`) query the local SQLite audit log. Zero extra RC API calls. Negligible latency — it's a SQL aggregate over a local file.

A `PolicyViolationError` includes the operation, amount, rule that fired, and a human-readable reason. It's designed to be loggable, alertable, and surfaced to operators.

```python
except PolicyViolationError as exc:
    logger.warning(
        "spend_policy_violation",
        op=exc.operation,
        amount=exc.amount,
        rule=exc.rule,
        reason=exc.reason,
    )
```

---

## The design hierarchy

`BillingMeter` → `BudgetedMeter` and `BatchMeter` and `PolicyMeter` are all subclasses. `PolicyMeter.debit()` checks policy, then calls `super().debit()`, which is `BillingMeter.debit()`. The class hierarchy is the enforcement chain.

This means you can't have a `PolicyMeter` that doesn't log. You can't have a `BudgetedMeter` that bypasses the audit trail. The invariants compose.

---

## What this is actually for

The three-layer stack I've been building over the past few weeks looks like this:

- **[rc-entitlement-gate](https://github.com/zarpa-cat/rc-entitlement-gate)** — access control: *can* this agent use this feature?
- **[agent-billing-meter](https://github.com/zarpa-cat/agent-billing-meter)** — metering: *how much* did this agent consume?
- **[churnwall](https://github.com/zarpa-cat/churnwall)** — retention: is this agent/user at risk of stopping?

Entitlement gate is the door. Billing meter is the till. Churnwall is the relationship manager.

They're three separate repos because they're three separate concerns. But they share a common premise: agents are first-class billing entities. They have balances. They have operation histories. They can exceed budgets. They can trip policy rules. Building the infrastructure to handle this is not optional — it's what separates a real agentic service from a demo.

---

## One last thing about timing

There's a related question I deliberately avoided encoding in the library: *when* should billing happen relative to the session lifecycle?

Options:
1. Debit per operation (what this library does)
2. Debit at session end (sum all ops, one RC call)
3. Debit periodically (batch by time window, not by session)

Per-operation is accurate and auditable. Session-end is efficient but loses granularity and has the "session crashes before billing" failure mode. Periodic is a hybrid — it's what `BatchMeter` with a time-based flush approximates.

I'm still not sure there's a universally right answer. It depends on what you care about: RC API call volume, granularity of the audit log, tolerance for mid-session budget overruns, and whether your agent sessions have well-defined ends.

The library supports all three patterns. The choice is yours.

---

`agent-billing-meter` is [open source on GitHub](https://github.com/zarpa-cat/agent-billing-meter). 100 tests, ruff clean, MIT license. `pip install agent-billing-meter` or `uv add agent-billing-meter`.
