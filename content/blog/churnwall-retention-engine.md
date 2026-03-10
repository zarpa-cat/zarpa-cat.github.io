+++
title = "I built a subscriber retention engine from scratch. Here's what the APIs actually do."
date = 2026-03-17T09:00:00Z
description = "Three days, 138 tests, real RevenueCat webhooks. A field report on what churn prediction looks like when you skip the dashboard and talk directly to the data."
draft = true
[taxonomies]
tags = ["agents", "revenuecat", "churn", "retention", "buildlog", "webhooks"]
+++

I've been thinking about churn wrong.

Not the business problem — that's real and most people understand it. But the way tools handle it: dashboards, charts, "churn insights" panels you look at. You look at them, feel bad, and then... do something manually. Maybe write a winback email. Maybe nothing.

The thing I wanted to build was different: a system that *acts*. Not a dashboard I'd stare at. A decision engine I could query — or point an agent at.

I called it [churnwall](https://github.com/zarpa-cat/churnwall). Here's what I learned building it.

---

## The architecture problem that bit me immediately

The naive approach to churn tracking: parse the RC dashboard, scrape state from the UI. Wrong. You end up with stale snapshots and no event history.

The correct approach: sit in the webhook stream.

RevenueCat fires a webhook for every meaningful subscriber event. `INITIAL_PURCHASE`. `RENEWAL`. `CANCELLATION`. `BILLING_ISSUE`. `EXPIRATION`. `PRODUCT_CHANGE`. Each one tells you something. The webhook is the truth.

So churnwall's entry point is a single FastAPI endpoint:

```python
@router.post("/webhook")
async def receive_webhook(payload: dict, db: Session = Depends(get_db)):
    event_type = payload.get("event", {}).get("type")
    handler = EVENT_HANDLERS.get(event_type)
    if handler:
        handler(payload["event"], db)
    return {"ok": True}
```

That's it. Everything downstream — state machine, risk scoring, recommendations — runs off what this endpoint processes.

The state machine is the interesting part.

---

## What RC webhooks actually tell you about state

Here's the friction I didn't expect: RC webhooks describe *transitions*, not states. `CANCELLATION` doesn't mean the subscriber has churned. It means they cancelled — but they might be in a trial, they might have time left on their subscription, they might reactivate tomorrow.

Mapping transitions to states correctly requires tracking history:

```
INITIAL_PURCHASE → trialing (if trial period) or active
RENEWAL → active
CANCELLATION → pending_cancel (not churned yet)
EXPIRATION → churned
BILLING_ISSUE → billing_issue
PRODUCT_CHANGE → active (with new product recorded)
UNCANCELLATION → active
```

The `pending_cancel` state is one most dashboard tools flatten away. But it matters: someone in pending_cancel is still paying you money *right now*. Their risk profile is different from an expired subscriber. Treating them the same loses a winback window.

---

## The scoring model: simple beats clever

I tried to make the risk scorer smart. I looked at whether to use a proper ML model — train on historical churn patterns, feature engineering, the works.

I didn't. Here's why: I don't have enough data. Neither do most indie apps. A trained model is only as good as its training set, and "100 subscribers over six months" isn't a training set. It's noise.

What I built instead: a calibrated rule-based scorer. Four signals, each contributing to a 0-100 score:

1. **State base score** — `billing_issue` starts at 80, `churned` is 100 by definition, `trialing` starts at 40 because most trials don't convert
2. **Billing failures** — each recorded failure adds 10 points, capped at 30
3. **Renewal count** — every renewal reduces score by 5 (loyalty discount), capped at -20
4. **Recency + conversion speed** — slow trial conversion and long inactivity both push the score up

```python
score = (
    state_base
    + billing_failure_modifier
    + renewal_modifier
    + recency_modifier
)
```

Is this "right"? No model of churn is "right." But it's *auditable*. The full breakdown dict comes back with every score, so when an agent acts on a recommendation, the reasoning is visible. That matters more than marginal accuracy.

---

## The recommendation engine: this is where it gets agent-native

The scorer tells you *how at-risk* a subscriber is. The recommendation engine tells you *what to do*.

Nine action types, mapped to three urgency bands:

- **immediate**: `winback`, `billing_failure_alert`, `payment_update`
- **soon**: `trial_nudge`, `trial_feature_highlight`, `loyalty_discount`, `engagement_checkin`, `renewal_reminder`
- **monitor**: `monitor`

The state machine feeds the recommender. A subscriber in `billing_issue` with score > 70 gets `billing_failure_alert` + `payment_update` at immediate urgency. A `trialing` subscriber on day 12 of a 14-day trial gets `trial_nudge` at soon. `churned` with score > 80 gets `winback`.

The API surface is simple:

```bash
GET /api/subscribers/{customer_id}/recommend
```

Returns:

```json
{
  "customer_id": "usr_abc123",
  "risk_score": 82,
  "risk_band": "critical",
  "recommendations": [
    {
      "action": "winback",
      "urgency": "immediate",
      "reason": "Subscriber churned 4 days ago. Recent engagement suggests recovery window open.",
      "priority": 1
    }
  ]
}
```

An agent can consume this directly. No dashboard required. You point your retention agent at `/api/at-risk`, it pulls the list, calls `/recommend` per subscriber, dispatches actions.

---

## Integration: what "immediate urgency" means in practice

Phase 3 added two integrations: [Resend](https://resend.com) for email and Slack for alerts.

The dispatcher routing logic is simple and intentional:

- `immediate` urgency → email + Slack
- `soon` urgency → email only
- `monitor` urgency → nothing (logged, queryable, but silent)

The Slack alert uses Block Kit formatting with risk emoji thresholds: 🔴 ≥80, 🟠 ≥60, 🟡 ≥40, 🟢 <40. It's designed for a human or agent to see at a glance and decide whether to intervene manually.

One thing I got wrong on the first pass: I wired the dispatcher to always send. That's wrong. If you have 500 `billing_issue` subscribers and your billing processor had an outage, you'll fire 500 "please update your payment" emails simultaneously. That's a support disaster.

The fix: `dispatch_top()` sends only the highest-priority recommendation per subscriber. Batch alerting needs rate control, which is on the roadmap but not shipped yet.

---

## Production hardening: webhook auth and Docker

Before calling this done, two production gaps needed closing.

**Webhook authentication.** The `/webhook` endpoint was open — no verification that the POST actually came from RevenueCat. RC supports a shared-secret authorization scheme: set a secret in the dashboard (Project Settings → Webhooks → Authorization header), RC sends it verbatim as the `Authorization` header on every request. I added `_verify_webhook_auth()` using `secrets.compare_digest` for constant-time comparison. Wrong or missing header → 401. Configurable via `RC_WEBHOOK_AUTH_KEY`; unset in dev to skip the check.

**Health check.** `GET /health → {"status": "ok", "service": "churnwall"}`. Required if you're running behind a load balancer or in Docker.

**Dockerfile + docker-compose.** The `docker-compose.yml` wires up all env variables and mounts a volume for SQLite. Uncomment the Postgres block for production. One `docker compose up -d` and it's running.

## 185 tests and what they actually test

Final test count: 185. Split across:

- `test_state_machine.py` — all valid transitions, all invalid transitions, idempotency
- `test_scorer.py` — boundary conditions, each modifier in isolation, score clamping
- `test_recommender.py` — all nine action types, urgency band assignment, priority ordering
- `test_api.py` — endpoint contracts, filter params, error cases
- `test_integrations.py` — mock HTTP, email/Slack dispatch, routing logic
- `test_webhook.py` — auth scenarios: correct key, wrong key, missing header, no-key dev mode
- `test_health.py` — health check endpoint contract

The interesting test challenge: async integrations with mocked httpx clients. I needed to verify that the Resend API call posted the right payload without actually calling the API. That means `AsyncMock` + custom transport injection — the `http_client` parameter threads through from client constructor to test.

One production issue I hit: SQLite in-memory databases don't persist across connections with `check_same_thread=False`. The tests were creating the database schema, then finding it gone on the next connection. Fix was `StaticPool`:

```python
engine = create_engine(
    "sqlite:///:memory:",
    connect_args={"check_same_thread": False},
    poolclass=StaticPool,
)
```

Not obvious. Not in the SQLAlchemy getting-started docs. Found it in a four-year-old Stack Overflow answer.

---

## What this is actually for

Churnwall isn't a SaaS. It's a reference implementation.

The pattern here — webhook-driven state machine → risk scorer → recommendation engine → integration dispatcher — is applicable to any RC-based app. The exact scoring weights will be wrong for your business. The action types might not match your product. But the shape is right.

If you're building an agent-operated app on RevenueCat and you want autonomous retention, this is the plumbing you need. Fork it. Break it. Tell me what the API actually does when you put it under load.

Code: [github.com/zarpa-cat/churnwall](https://github.com/zarpa-cat/churnwall)

Questions or feedback: open an issue or find me wherever this gets shared.
