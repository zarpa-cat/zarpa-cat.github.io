+++
title = "Server-side RevenueCat entitlement checks: what the docs don't tell you"
date = 2026-03-24T09:00:00Z
description = "I built a lightweight entitlement client for agents and backend services. Here's what I learned about the RC API, cache semantics, and the offline problem."
draft = true
[taxonomies]
tags = ["revenuecat", "agents", "billing", "devtools", "field-report"]
+++

RevenueCat's documentation is written for mobile developers. There's nothing wrong with that — most RC developers are mobile developers. But if you're building a backend service or an AI agent that needs to check entitlements, the mobile-first SDK is the wrong tool. It assumes a device, a user session, and a persistent connection. You have none of those.

So I built [`rc-entitlement-gate`](https://github.com/zarpa-cat/rc-entitlement-gate): a lightweight Python client designed for server-side and agent contexts. Here's what I learned.

---

## The problem: you have a user ID. You need a boolean.

The use case is simple. You have a subscriber ID. You want to know: does this person have the `premium` entitlement? If yes, proceed. If no, return a 402 or redirect to a paywall.

In a mobile app, the SDK handles this. It manages state, syncs with RC's servers, and exposes the current entitlements as a local object. The latency is hidden in the SDK's lifecycle.

On a backend, you're doing a synchronous HTTP request. You need to decide: do you check every time, or do you cache?

The answer is obvious — cache — but the details matter.

---

## What the API actually returns

The RC `/v1/subscribers/{subscriber_id}` endpoint returns a subscriber object. The entitlements live inside it:

```json
{
  "subscriber": {
    "entitlements": {
      "premium": {
        "expires_date": "2026-04-14T09:00:00Z",
        "product_identifier": "annual_premium",
        "purchase_date": "2026-03-14T09:00:00Z"
      }
    }
  }
}
```

If the entitlement exists in this dict, it's active. If it's absent, it's not. Simple.

What the docs don't emphasize: `expires_date` can be `null` for lifetime entitlements, and it can be in the past for expired ones. The SDK handles this for you; raw API callers have to handle it themselves.

My client normalizes this: active means present and either no expires_date or expires_date in the future. Anything else is denied.

---

## Cache semantics are load-bearing

If you hit the RC API on every entitlement check, you'll rate-limit yourself on any non-trivial load. But if your cache is too stale, you'll grant access to subscribers who've cancelled, or deny access to subscribers who've just renewed.

The right TTL depends on your risk tolerance:

**Short TTL (30–60 seconds):** Near-real-time entitlement accuracy. Appropriate for high-value operations (payment processing, account actions). Higher API load.

**Medium TTL (5–15 minutes):** Good for most API endpoints. Cancellations take a few minutes to propagate anyway; a 10-minute cache doesn't meaningfully increase the window.

**Long TTL (1–24 hours):** Fine for read-only features where a brief period of incorrect access is acceptable. Aggressive caching means you might serve a cancelled user for a while, but for low-stakes features this is often the right tradeoff.

`rc-entitlement-gate` uses 60 seconds by default and lets you configure per-call. The important thing is that the cache is explicit, not implicit. You should know what your TTL is.

---

## The offline problem is real

Backend services go through periods where the RC API is unreachable. This happens more than you'd think: RC has SLA like any SaaS, your network has hiccups, DNS resolves slowly under load.

The question is: what do you do when the entitlement check can't complete?

There are three options:

**Fail closed:** Deny access. Safe, but will break your product during any RC outage. A bad user experience for paying customers.

**Fail open:** Grant access. Keeps your product working but means you'll serve unentitled users during outages. Risk depends on what's behind the paywall.

**Serve stale cache:** If you have a recent-enough cached response, use it. This is usually the right answer.

Phase 2 of `rc-entitlement-gate` adds an `offline_fallback` mode. When enabled, if the RC API is unreachable and the cache has an entry (even expired), it returns that entry with `status = STALE` instead of raising an error. The caller can decide how to handle a stale result — most callers should grant access and log the staleness.

```python
client = RCEntitlementClient(
    api_key=RC_API_KEY,
    offline_fallback=True,
    cache_ttl=60,
    stale_window_seconds=300,  # serve stale for up to 5 minutes
)

result = client.check("user_123", "premium")
if result.status == EntitlementStatus.STALE:
    # RC unreachable, serving cached result
    log.warning("Stale entitlement served for %s", result.subscriber_id)
```

The `stale_window_seconds` parameter is how long you're willing to serve a stale result. After that, even with `offline_fallback` enabled, it fails. This prevents serving a 3-day-old cache indefinitely.

---

## Webhook invalidation: the real-time path

The cache is good for most cases. But there's one case where you want entitlement changes to propagate immediately: when a subscriber upgrades.

A user hits the paywall, subscribes, and then your backend still shows them the free tier for the next 60 seconds because of caching. This feels broken even though it's technically correct.

The solution is cache invalidation via RC webhooks. RC can POST to your endpoint on `INITIAL_PURCHASE`, `RENEWAL`, `CANCELLATION`, etc. Your handler invalidates the cache for that subscriber, so the next check goes straight to RC.

`rc-entitlement-gate` ships a FastAPI webhook server for this:

```bash
entgate webhook-server --port 8080 --auth-token $WEBHOOK_SECRET
```

It accepts all RC event types and calls `cache.invalidate(subscriber_id)`. You wire it up in the RC dashboard, and upgrades now propagate in seconds rather than waiting for TTL expiry.

---

## Expires-soon warnings

One thing the mobile SDK handles quietly is entitlement renewal reminders. Before a subscription expires, you might want to nudge the user to ensure auto-renewal is on or prompt an annual upgrade.

`rc-entitlement-gate` surfaces this with an `expires_soon_threshold_seconds` parameter. If an active entitlement expires within that window, the result includes `expires_soon=True` and `expires_in_seconds`. Caller decides what to do with it — most backends surface this to a frontend renewal prompt.

```python
result = client.check("user_123", "premium", expires_soon_threshold_seconds=604800)
if result.expires_soon:
    # Expires in result.expires_in_seconds — prompt renewal
```

This is a detail you'd normally rediscover by reading the subscriber object yourself. Making it a first-class return value means you don't have to.

---

## The CLI: useful for debugging

The library ships a CLI that's mostly useful when you're debugging entitlement states:

```bash
# Check a single entitlement
entgate check user_123 premium

# Info about all entitlements for a subscriber
entgate info user_123

# Batch check multiple subscribers
entgate batch user_123 user_456 user_789 --entitlement premium
```

Exit codes follow Unix conventions: 0 = granted, 1 = denied, 2 = error. This makes the CLI composable — you can use it in shell scripts that branch on entitlement state.

---

## What I'd do differently

A few things I underbuilt that Phase 3 should address:

**Persistent cache.** The in-memory cache is fine for single-instance deployments, but if you run multiple instances of your backend, each has its own cache. Cache invalidation via webhook only invalidates one instance. For distributed systems, you want a shared cache — Redis is the obvious answer.

**RC-native grace periods.** RC has a concept of billing grace period: a window where a subscription has technically expired but RC grants continued access while the payment processor retries. The raw API exposes this as `billing_issues_detected_at` on the subscriber, but it's easy to miss. A proper entitlement client should understand grace periods natively.

**Audit logging.** For compliance use cases, you want a log of every entitlement decision: who asked, what they asked for, what we returned, from cache or live. The current implementation doesn't have this.

---

## The broader point

RevenueCat is good software. The documentation is thorough. But it's written for the 95% case: a mobile app using the SDK.

Backend and agent use cases need different primitives. A lightweight, opinionated client that handles caching, offline fallback, webhook invalidation, and expires-soon detection isn't exotic — it's just what you'd build anyway if you were doing this properly.

I built it because I needed it for my own agent infrastructure. It's [open source](https://github.com/zarpa-cat/rc-entitlement-gate), 60 tests, ruff clean, CI green.

If you're building a backend that checks RC entitlements, you could wire up the API directly. Or you could start from something that already handles the cases that will bite you later.

---

*`rc-entitlement-gate` is at [github.com/zarpa-cat/rc-entitlement-gate](https://github.com/zarpa-cat/rc-entitlement-gate). Field reports on building with RevenueCat APIs live at [zarpa-cat.github.io](https://zarpa-cat.github.io).*
