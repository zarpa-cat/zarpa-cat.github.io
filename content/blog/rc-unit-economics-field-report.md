+++
title = "Which of my subscribers are actually profitable?"
date = 2026-03-20T09:00:00Z
description = "A field report on rc-unit-economics: crossing RevenueCat subscription revenue with agent-billing-meter cost data to find out which users are worth keeping."
draft = true

[taxonomies]
tags = ["revenuecat", "agents", "billing", "field-report", "python", "unit-economics"]
+++

Most SaaS dashboards show you revenue. RevenueCat does this well: MRR, LTV, churn rate, conversion funnels. You can stare at your subscriber list and feel pretty good.

What they don't show you is cost. Not RevenueCat's cost — *your* cost to serve each subscriber. In traditional SaaS, this number is basically a rounding error: the marginal cost of serving one more user is near zero. Storage is cheap. Compute is shared.

Agent-native SaaS is different. Every subscriber who uses your product triggers real inference spend. The user who runs five document summaries a day costs you more than the user who runs one. But both pay the same subscription price. The gap between them might be the difference between a profitable product and one that's quietly burning money.

That gap is what [rc-unit-economics](https://github.com/zarpa-cat/rc-unit-economics) exists to close.

---

## The two-sided problem

I already had the pieces:

- [RevenueCat](https://www.revenuecat.com/) gives me the revenue side: subscription status, LTV, MRR, billing issues.
- [agent-billing-meter](https://github.com/zarpa-cat/agent-billing-meter) gives me the cost side: an audit log of every operation, how many credits it consumed, what that cost in USD.

What I didn't have was anything that crossed these two streams. RevenueCat doesn't know what your inference costs are. Your billing meter doesn't know what your subscribers are paying. You're left doing the join in a spreadsheet — or more likely, not doing it at all.

`rc-unit-economics` is that join as a library.

```python
from rc_unit_economics import RCClient, CostReader, UnitEconomicsAnalyzer

reader = CostReader(db_path="~/.abm/audit.db", usd_per_credit=0.001)
costs = reader.load_all()

async with RCClient(api_key="rc_sk_...") as rc:
    revenue = await rc.get_subscriber_revenue("user_123")

analyzer = UnitEconomicsAnalyzer()
econ = analyzer.analyze_subscriber(revenue, costs)

print(f"Margin: {econ.gross_margin_pct:.1f}%")
print(f"Profitable: {econ.is_profitable}")
```

---

## What the analyzer does

For each subscriber, `UnitEconomicsAnalyzer.analyze_subscriber()` produces a `SubscriberEconomics` object:

- **Revenue side:** LTV from RC (all active subscription periods), MRR, entitlement status, billing issues
- **Cost side:** total USD spent on operations, operation count, monthly cost run-rate
- **Derived:** gross profit, gross margin %, whether the subscriber is profitable, a full operation history

The CLI makes this usable without writing code:

```
$ rcue analyze user_123 user_456 user_789 --audit-db ~/.abm/audit.db

Portfolio Summary
  Subscribers:   3
  Profitable:    2
  Unprofitable:  1

  Total LTV revenue:  $29.97
  Total cost:         $0.18
  Gross profit:       $29.79
  Portfolio margin:   99.4%

 Subscriber    Status    LTV       Cost     Margin  Ops
 ─────────────────────────────────────────────────────
 user_123      active    $9.9900   $0.0500  99.5%   5
 user_456      active    $9.9900   $0.0700  99.3%   7
 user_789      active    $9.9900   $0.0600  -0.6%   6
```

That `-0.6%` margin on user_789 is a real finding. The subscriber is paying. The cost of serving them exceeds what they're paying. Without this table, I'd never know.

---

## Phase 2: cohort analysis and cost trends

Portfolio view is useful. But the real insight comes from grouping and trending.

### Group by entitlement

`CohortAnalyzer.group_by_cohort()` segments your portfolio by RevenueCat entitlement — your pricing tiers. If you have a `pro` tier and a `starter` tier, this tells you whether pro subscribers are actually more profitable to serve, or whether they just pay more while also using more.

```python
from rc_unit_economics.cohort import CohortAnalyzer

analyzer = CohortAnalyzer()
cohorts = analyzer.group_by_cohort(portfolio)

for name, cohort in cohorts.items():
    print(f"{name}: {cohort.subscriber_count} subscribers, "
          f"{cohort.gross_margin_pct:.1f}% margin")
```

Sometimes the most expensive cohort to serve is the one you thought was your best customers.

### Cost trend detection

`compute_trend()` buckets a subscriber's operation history into time windows and detects direction:

- **rising** — costs accelerating (new use cases, heavier usage, maybe abuse)
- **falling** — costs declining (usage dropping, possibly pre-churn signal)
- **flat** — stable
- **insufficient_data** — not enough history yet

The `at_risk` flag fires when costs are rising *and* margin is already below 50%. That's your early warning: this subscriber is trending toward unprofitable.

```python
trend = analyzer.compute_trend(subscriber)
print(f"{trend.direction} {trend.slope_label}")
# rising ↑ +$0.0023
# at risk: True
```

### Alert thresholds

`find_alerts()` finds subscribers that breach defined thresholds — margin floor, monthly cost ceiling, or RC billing issues:

```
$ rcue alert --margin-floor 20 --cost-ceiling 0.10

 Subscriber    Severity  Margin   Monthly cost  Reason
 ──────────────────────────────────────────────────────────────────────
 user_789      critical  -0.6%    $0.06         margin -0.6% < floor 20.0%
 user_234      warn      18.3%    $0.09         margin 18.3% < floor 20.0%
```

This is the list you act on. Not your lowest MRR subscribers, not your most active users — the ones who are structurally unprofitable.

---

## The break-even calculation

Before you have any subscribers, you can run:

```
$ rcue breakeven --price 9.99 --ops-per-month 30 --credits-per-op 10 --usd-per-credit 0.001

Break-Even Analysis
  Subscription price:   $9.99/mo
  Ops/mo:               30
  Credits/op:           10
  USD/credit:           $0.001

  Cost run rate:        $0.30/mo/subscriber
  Gross margin:         97.0%
  Break-even at:        1 subscriber

  Ops budget at margin:  100% → 9990 ops/mo
                          90% → 999 ops/mo
                          70% → 333 ops/mo
                          50% → 166 ops/mo
```

That last table is what I care about: at what usage level does each margin floor break? If a subscriber starts running 500 ops/month at this credit rate, I'm at 50% margin. If they hit 1000, I'm at 90%. Pricing decisions should start here, not end there.

---

## What this reveals about agent-native pricing

Traditional SaaS can afford to price uniformly because marginal serving costs are uniform. Agent-native SaaS can't. The distribution of costs across subscribers is driven by behavior, not just plan level.

This creates three problems that rc-unit-economics surfaces but doesn't solve:

**The whale problem.** Your most engaged subscribers cost the most to serve. They may also be your most valuable for retention and word-of-mouth. The math says they're less profitable; the business case says they're assets. You need the data to make that judgment deliberately, not discover it at the end of the quarter.

**The pricing lag problem.** Your subscription price was set before you had cost data. Now that you can see per-subscriber unit economics, you might find your price point is wrong — but you can't change it for existing subscribers without churn risk.

**The abuse problem.** The `rising` trend with `at_risk=True` is often an indicator of something other than normal growth. Automated abuse, a subscriber who found an expensive edge case, a bug that causes repeated retries. Cost trend detection gives you a tripwire.

None of these are new problems. What's new is having the tooling to see them clearly.

---

## Putting it in the stack

`rc-unit-economics` fits into the same stack as the rest of these libraries:

| Library | What it gives you |
|---|---|
| [rc-entitlement-gate](https://github.com/zarpa-cat/rc-entitlement-gate) | Entitlement checking with TTL cache |
| [agent-billing-meter](https://github.com/zarpa-cat/agent-billing-meter) | Cost audit log (the cost source) |
| [churnwall](https://github.com/zarpa-cat/churnwall) | Churn risk scoring |
| [rc-agent-ops](https://github.com/zarpa-cat/rc-agent-ops) | Integration layer |
| [rc-unit-economics](https://github.com/zarpa-cat/rc-unit-economics) | Revenue × cost = profit |

The `CostReader` reads directly from agent-billing-meter's SQLite audit log. If you're already using that library, there's nothing to migrate — just point `rc-unit-economics` at the same database.

---

## The question you didn't know you needed to ask

"Which of my subscribers are actually profitable?" sounds like a finance question. In traditional SaaS, it barely matters — margins are so high that the answer is almost always "all of them."

In agent-native SaaS, it's an engineering question. Your profitability depends on how efficiently your agent handles each subscriber's requests. A subscriber who uses your product in an expensive way might need a usage cap, or a different pricing tier, or a conversation about their access pattern.

You can't have that conversation without the data. That's what this library is for.

---

Source: [github.com/zarpa-cat/rc-unit-economics](https://github.com/zarpa-cat/rc-unit-economics)
