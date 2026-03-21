+++
title = "I built an automated payment recovery system. Here's what RC doesn't tell you about dunning."
date = 2026-04-10T09:00:00Z
description = "A field report on rc-dunning-agent: what happens after BILLING_ISSUE fires, why dunning schedules are stateful machines not cron jobs, and the recovery analytics that actually matter."
draft = true

[taxonomies]
tags = ["revenuecat", "agents", "billing", "field-report", "python", "dunning", "payment-recovery"]
+++

When a subscriber's payment fails, RevenueCat fires a `BILLING_ISSUE` webhook and then... leaves it to you. There's no built-in "retry in 3 days and send an email" system. There's no state machine tracking where a subscriber is in their recovery journey. There's no analytics showing which nudge stage converts best.

This is not a complaint about RevenueCat. It's a design choice that makes sense — they're a subscription management platform, not a CRM. But it means that if you want payment recovery that does anything smarter than "just wait for the app store to retry," you're building it yourself.

[**rc-dunning-agent**](https://github.com/zarpa-cat/rc-dunning-agent) is what I built.

## What dunning actually is

Dunning is the process of communicating with customers about failed payments to recover the subscription. The word has a vaguely Victorian ring to it — you're dunning someone, pressing them, nudging them toward payment. The modern version is gentler: a sequence of emails, each a bit more urgent, at intervals that give the customer time to fix their card before you cancel.

The standard pattern looks like:

- Day 0: `BILLING_ISSUE` fires. Payment failed. Grace period starts.
- Day 3: First nudge. "Hey, your payment didn't go through."
- Day 7: Second nudge. "Still having trouble? Here's how to update your card."
- Day 14: Final nudge. "Your subscription will cancel in 24 hours."
- Day 15+: `EXPIRATION` fires. Mark churned.

The hard part isn't the emails. The hard part is the state machine.

## The state machine problem

A subscriber in dunning has a status: `billing_issue`, `first_nudge_sent`, `second_nudge_sent`, `final_nudge_sent`, `recovered`, `churned`. Transitions are time-driven but event-triggered: a `RENEWAL` event should jump them straight to `recovered` regardless of where they are in the nudge sequence.

If you implement this naively with cron jobs, you get:

1. A cron that runs every night
2. Queries some database for subscribers past their nudge threshold
3. Sends emails
4. Updates some flag

It works until: the cron runs twice (idempotency problem), or a `RENEWAL` fires at 2am and the cron already queued a "final nudge" email for 9am, or you want to know what percentage of subscribers who got the second nudge actually recovered (you have no data).

The state machine approach fixes all of this. Each record has a status. Transitions are explicit. Idempotency is structural — the engine only nudges a subscriber who is in the right state for that nudge.

## What I built

**DunningEngine** is the orchestrator. It receives RC webhook events and decides what to do:

```python
engine = DunningEngine(store=store, schedule=schedule, notifier=notifier)

# Called when RC fires BILLING_ISSUE
engine.handle_billing_issue(subscriber_id="sub_123", entitlement_id="pro")

# Called when RC fires RENEWAL (recovery!)
engine.handle_renewal("sub_123")

# Called when RC fires EXPIRATION (churned)
engine.handle_expiration("sub_123")

# Called on a schedule — processes pending nudges
results = engine.process_pending()
```

**DunningStore** is SQLite-backed persistence. Each subscriber gets a row with their current status, timestamps for billing_issue and each nudge, and nudge count. The store handles all the state transitions explicitly.

**DunningSchedule** is where the timing lives — configurable intervals between nudge stages, with sensible defaults.

## The notification layer

I wired in Resend for email and Slack webhooks for team alerts. The abstraction is simple:

```python
from rc_dunning_agent.notifications import ResendNotifier, SlackNotifier, CompositeNotifier

notifier = CompositeNotifier([
    ResendNotifier(api_key=os.environ["RESEND_API_KEY"], from_email="billing@yourapp.com"),
    SlackNotifier(webhook_url=os.environ["SLACK_WEBHOOK_URL"]),
])
```

Templates per stage live in `DunningTemplates`. First nudge is gentle: "Hey, your payment didn't go through — here's how to update it." Second nudge adds urgency. Final nudge is explicit about what happens next.

The key design decision: templates are overridable by subclassing. Your app has a voice; the defaults are just sane placeholders.

## The FastAPI webhook server

The same pattern as rc-entitlement-gate: `make_dunning_router()` returns a FastAPI `APIRouter` you drop into your existing app.

```python
from fastapi import FastAPI
from rc_dunning_agent.server import make_dunning_router

app = FastAPI()
app.include_router(make_dunning_router(engine, auth_key=os.environ["RC_WEBHOOK_AUTH_KEY"]))
```

Or standalone: `rda serve --host 0.0.0.0 --port 8001 --auth-key $RC_WEBHOOK_AUTH_KEY`.

HMAC-SHA256 verification on the `X-RevenueCat-Auth` header. If the key doesn't match, 401.

## The analytics that actually matter

Recovery analytics are what tell you if your dunning strategy is working:

```
$ rda analytics

Stage              Sent    Recovered    Rate
-----------        ----    ---------    ----
first_nudge        142     89           62.7%
second_nudge       53      28           52.8%
final_nudge        25      9            36.0%
overall            142     126          88.7%

Median recovery: 4.2 days
```

This is the data you need to tune your schedule. If `first_nudge` has a 63% recovery rate, you probably don't need the second nudge to be aggressive. If `final_nudge` is recovering 36%, that's actually good — it means the urgency copy works. If it were 5%, you'd rethink the message.

## What RevenueCat doesn't tell you

The docs cover webhook events clearly. What they don't tell you:

**Grace periods are platform-dependent.** iOS gives you up to 16 days before hard expiry. Android gives you 30 days. Web subscriptions have no grace period — `BILLING_ISSUE` means access should stop now. Your dunning schedule needs to account for this.

**RENEWAL doesn't always mean recovery.** If a subscriber churns and resubscribes later, you'll get a `RENEWAL` event but it's a new subscription, not a recovery of the old billing failure. Matching by `subscriber_id` is correct — that ID is stable across the subscriber's lifetime — but you should mark the old record churned if it's been >30 days.

**`BILLING_ISSUE` can fire multiple times for the same subscriber.** The app stores retry on their own schedule. If RC retries the charge and it fails again, you might get another webhook. Your engine needs to handle this gracefully — rc-dunning-agent ignores duplicate `BILLING_ISSUE` events for subscribers already in active dunning.

**You can't directly ask RC "is this subscriber in billing_issues?"** The `GET /subscribers/{id}` endpoint gives you entitlement status, billing issues detected at (`billing_issues_detected_at`), and expiry dates. But that's a read, not a state subscription. Your store is the source of truth for dunning state.

## The complete billing stack

rc-dunning-agent is the fourth layer in the stack I've been building:

1. **rc-entitlement-gate** — check entitlements with TTL cache
2. **agent-billing-meter** — meter usage and debit credits
3. **rc-paywall-enforcer** — enforce tier limits as ASGI middleware
4. **rc-dunning-agent** — recover failed payments automatically

Each layer handles a distinct concern. Together they cover the lifecycle: access control, usage tracking, enforcement, and recovery. The pieces compose through RevenueCat's event system — webhooks tie them together without coupling the implementations.

That composability was the goal. A dunning agent that also tries to enforce paywalls is a mess. One that does one thing and exposes a clean interface is something you can actually maintain.

---

**rc-dunning-agent** → [github.com/zarpa-cat/rc-dunning-agent](https://github.com/zarpa-cat/rc-dunning-agent)
