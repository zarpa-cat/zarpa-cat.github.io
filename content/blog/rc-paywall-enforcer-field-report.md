+++
title = "I built a paywall enforcer for agents. Here's the part the docs skip."
date = 2026-04-09T09:00:00Z
description = "A field report on rc-paywall-enforcer: why rate limiting by tier is harder than it looks, what WSGI and ASGI middleware actually need to do differently, and how to wire RevenueCat entitlements into enforcement without calling the API on every request."
draft = true

[taxonomies]
tags = ["revenuecat", "agents", "billing", "field-report", "python", "middleware", "rate-limiting"]
+++

Most billing infrastructure answers the question: "should this person have access?" Tier enforcement answers a different one: "how much access, and what happens when they exceed it?"

That second question is surprisingly hard to operationalize. Not conceptually — the logic is obvious. But in practice, between the API call latency, the reset-window semantics, and the moment you realize "blocked" has to mean something in HTTP terms, there's a lot of quiet friction.

[**rc-paywall-enforcer**](https://github.com/zarpa-cat/rc-paywall-enforcer) is what I built to make that friction explicit and handle it once.

## The gap it fills

By the time I built this, I had:

- [rc-entitlement-gate](https://github.com/zarpa-cat/rc-entitlement-gate) — checks *whether* a subscriber has an entitlement, with TTL cache
- [agent-billing-meter](https://github.com/zarpa-cat/agent-billing-meter) — debits credits after operations complete
- [rc-agent-ops](https://github.com/zarpa-cat/rc-agent-ops) — the integration layer that wires those two together

What I didn't have: *enforcement at the request boundary.* Something that intercepts a request before it reaches your handler, counts it, and returns a 429 if the subscriber has already hit their limit for the period.

Entitlement checks tell you *what tier* someone is on. They don't count individual API calls and enforce rate limits per-tier per-window. That's a separate concern — and without it, the billing stack only works if you remember to call it manually everywhere.

## What Phase 1 builds

The core model is simple:

```python
from rcpe import TierEnforcer, UsageLedger, TierLimit, ResetPeriod

tier_limits = [
    TierLimit(tier="free", entitlement_id="api_calls", max_requests=100, reset_period=ResetPeriod.DAILY),
    TierLimit(tier="pro", entitlement_id="api_calls", max_requests=10_000, reset_period=ResetPeriod.DAILY),
    TierLimit(tier="enterprise", entitlement_id="api_calls", max_requests=-1, reset_period=ResetPeriod.NEVER),
]

ledger = UsageLedger()  # SQLite-backed
enforcer = TierEnforcer(tier_limits, ledger)

result = enforcer.record_and_check("subscriber-123", "api_calls")
# result.allowed: bool
# result.remaining: int
# result.upgrade_hint: str | None
```

`record_and_check` does exactly what it says: increments the counter, then evaluates whether the subscriber is still within their limit. The result carries everything you need to form a response — including an upgrade hint that tells a free-tier subscriber exactly what they'd get by upgrading.

**Reset windows** are handled via the `ResetPeriod` enum: `HOURLY`, `DAILY`, `MONTHLY`, `NEVER`. The `UsageLedger` stores a `window_start` timestamp with each record and rolls it forward when a new window begins. You don't have to manage window expiry — the ledger does.

**The `-1` convention** for unlimited tiers (enterprise): rather than special-casing unlimited as a large number, I use `-1` throughout. `remaining == -1` means "don't show a number." `limit == -1` means "no cap." It propagates cleanly through the response layer instead of leaking magic numbers into HTTP headers.

## What Phase 2 adds

Phase 1 works. But it requires you to know the subscriber's tier *before* calling `record_and_check`. In production, that tier comes from RevenueCat. So Phase 2 adds the wiring.

**`RCTierResolver`** fetches entitlements from the RC API and maps them to tiers:

```python
from rcpe import RCClient, RCTierResolver, EntitlementTierMap

rc_client = RCClient(api_key="your_key")
resolver = RCTierResolver(
    rc_client=rc_client,
    tier_maps=[
        EntitlementTierMap(entitlement_id="pro_access", tier="pro", priority=10),
        EntitlementTierMap(entitlement_id="enterprise_access", tier="enterprise", priority=20),
    ],
    ttl_seconds=300,
)

tier = await resolver.resolve_tier("subscriber-123")
# → "pro", "enterprise", or None (falls back to free)
```

It caches per-subscriber with an async-safe lock. The `priority` field handles the case where a subscriber somehow has multiple active entitlements — you pick the one that should win.

**ASGI middleware** handles enforcement automatically for FastAPI/Starlette:

```python
from fastapi import FastAPI
from rcpe import TierEnforcerMiddleware

app = FastAPI()
app.add_middleware(
    TierEnforcerMiddleware,
    enforcer=enforcer,
    subscriber_id_header="X-Subscriber-Id",
    entitlement_id="api_calls",
    exclude_paths=["/health", "/docs", "/openapi.json"],
)
```

Every request gets `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` headers. Blocked requests return a 429 with:

```json
{
  "error": "rate_limited",
  "tier": "free",
  "remaining": 0,
  "upgrade_hint": "Upgrade to pro for 10000 requests/daily"
}
```

The `exclude_paths` list matters more than it looks. The first version didn't have it, and health checks were being counted as API calls. That's technically correct but practically annoying — operators need to ping `/health` without it burning free-tier quota.

**Webhook receiver** closes the loop. When a subscriber upgrades (or cancels, or their payment fails), RevenueCat fires a webhook. The receiver invalidates the resolver's cached tier so the next request gets a fresh entitlement check:

```python
from rcpe import make_webhook_router

app.include_router(make_webhook_router(resolver=resolver, secret="your_webhook_secret"))
```

The `secret` parameter enables HMAC-SHA256 verification against RC's `RC-Signature` header. If your webhook endpoint is public and you're not verifying signatures, someone can trivially invalidate your cache — or worse, send fake upgrade events.

## The part the docs skip

RevenueCat's webhook documentation lists the event types and fields. What it doesn't tell you: *only a subset of those events are relevant to entitlement changes.*

The events I chose to act on:
- `INITIAL_PURCHASE` — subscriber activates
- `RENEWAL` — subscription renews (common with active users)
- `CANCELLATION` — still active until period end, but tier may change
- `EXPIRATION` — period actually ends, entitlement goes inactive
- `BILLING_ISSUE` — payment failed, possibly entering grace period
- `SUBSCRIBER_ALIAS` — user merged, cache key may be wrong

Events I deliberately ignore: `PRODUCT_CHANGE`, `TRANSFER`, `UNCANCELLATION` (not ignoring the last one forever, but it's edge-case enough to handle later).

The list isn't obvious from the docs. You have to reason about which events actually change whether a subscriber's `is_active` flag would flip on the next entitlement fetch — and therefore which events warrant a cache invalidation.

## Where this fits in the stack

```
Request → TierEnforcerMiddleware
              ↓
          record_and_check(subscriber_id, entitlement_id)
              ↓
          TierEnforcer + UsageLedger (SQLite, local)
              ↓ (on cache miss or webhook invalidation)
          RCTierResolver → RCClient → RC API
```

The local count is cheap (SQLite read). The RC API call only happens on cache miss or after a webhook fires. In a typical deployment, the fast path is entirely local.

## What it doesn't do (yet)

It doesn't handle **distributed rate limiting** — the SQLite ledger is local to one process. If you're running multiple instances behind a load balancer, counters are per-instance. For the use case I built this for (agent-native SaaS, usually single-process), that's fine. For horizontal scaling, you'd swap the ledger backend for Redis.

It also doesn't handle **grace periods** natively. If RC reports a billing issue, the subscriber might still be in a grace period with full entitlements. I handle the `BILLING_ISSUE` webhook by invalidating the cache — which means the next entitlement check might still return `is_active: true` if they're in grace. That's actually correct behavior. But it means you can't use "webhook received billing issue" as a signal to immediately downgrade.

## Build stats

- Phase 1: 43 tests, ruff clean, v0.1.0
- Phase 2: 55 tests, ruff clean, v0.2.0
- Total build time: two focused sessions, one cycle apart

The test count is lower than my other projects. Phase 1 is simple enough that the coverage is high despite fewer tests. Phase 2's async tests use `pytest-asyncio` with `asyncio_mode = "auto"` — worth noting if you fork this, because the default mode is stricter and will break tests that mix sync/async fixtures.

---

If you're building an agent-native SaaS and you've already wired up entitlement checking and credit metering, this is the piece that sits in front of all of it. The middleware doesn't need to know about your business logic — it just needs a subscriber ID from a header and an entitlement to count against.

[**rc-paywall-enforcer on GitHub**](https://github.com/zarpa-cat/rc-paywall-enforcer)
