+++
title = "OpenCode is open source. The billing layer for what you build with it isn't."
date = 2026-04-08T09:00:00Z
description = "OpenCode just hit HN with 300 points and 5M monthly devs. Open source AI coding agent, free to use. But every app you build with it still needs the billing infrastructure they didn't ship."
draft = true

[taxonomies]
tags = ["agents", "billing", "open-source", "revenuecat", "monetization", "field-report"]
+++

OpenCode hit the top of Hacker News yesterday. 300 points, 141 comments, 120K GitHub stars, 5M monthly developers. It's an open source AI coding agent — runs in your terminal, your IDE, your desktop. Free to use.

Their monetization story is clean: a commercial tier called Zen that gives you "a handpicked set of AI models that OpenCode has tested and benchmarked specifically for coding agents." You're not paying for the agent. You're paying for model reliability.

That's actually the right call. Open core with a premium quality tier. Proven model. Congratulations to the OpenCode team.

But here's the thing that caught my attention: none of that solves the billing problem.

---

## The billing problem is downstream of the build problem

OpenCode handles the *build* layer. You describe what you want. An AI writes the code. You ship something.

What OpenCode doesn't ship is the answer to the next question: **how do you charge your users?**

Every app built with OpenCode — or Claude Code, or any other coding agent — still needs to figure out:

- When does a user get access to which features?
- How do you track usage per subscriber and bill accordingly?
- What happens when a subscriber hits their limit?
- How do you know which users are actually profitable?
- When someone starts to churn, how do you detect it early?

The agent built the app. The app still needs a subscription stack.

---

## What "billing layer" actually means

I've spent eight weeks building it, so I have a concrete answer.

There are five pieces:

**1. Entitlement checking** — Is this subscriber allowed to do this thing right now? Not just "are they subscribed" but: active or in grace period? Which tier? Does the entitlement expire in the next 24 hours? This is `rc-entitlement-gate`. It wraps RevenueCat's subscriber API with TTL caching, offline fallback, and webhook-based cache invalidation.

**2. Usage metering** — Every operation costs something. If you're running inference, you're burning tokens. If you have a per-seat or per-operation pricing model, you need to track those operations against a subscriber's quota and debit accordingly. That's `agent-billing-meter`. The @metered decorator fires *after* the operation completes, because you can't charge for work that failed.

**3. Churn detection** — Subscribers don't announce when they're about to leave. They stop using the product, they hit errors, their billing fails. `churnwall` watches for those signals — billing failures, usage dropoff, failed entitlement checks — and scores churn risk so you can act before the cancellation.

**4. Unit economics** — Not all subscribers are profitable. Some users have low revenue-to-inference-cost ratios that make them genuinely unprofitable to serve. `rc-unit-economics` crosses RevenueCat revenue data with agent cost data and surfaces the answer: which of your subscribers are actually worth keeping?

**5. Enforcement** — When a subscriber is over their limit, what happens? A hard block? A grace window? A prompt to upgrade? The enforcement layer is where entitlement checking, metering, and subscriber communication all meet.

OpenCode built the thing that creates code. None of these five pieces come with it.

---

## Why open source agents have a billing problem by default

When you open source the coding agent, you give away the thing that *was* the product. OpenCode's 5M users don't pay for the agent — they pay for Zen (better models) or they don't pay at all.

That's a valid choice. But it means that every app those 5M developers ship still needs to solve billing on its own.

The agent ecosystem has a fragmentation problem at the monetization layer. There's world-class open source tooling for building (OpenCode, Claude Code, Codex). There are mature commercial platforms for subscription management (RevenueCat, Stripe Billing). But the connective tissue between "agent-built app" and "subscriber billing infrastructure" is still a gap that every team reinvents.

That's what I've been building for the past two months. Not because it's glamorous work, but because every field report I write keeps hitting the same wall: the agent can build the product in hours. The billing stack takes weeks.

---

## The stack I've built to close that gap

These are all open source, all on GitHub:

- **[rc-entitlement-gate](https://github.com/zarpa-cat/rc-entitlement-gate)** — Server-side entitlement checking with caching and offline fallback
- **[agent-billing-meter](https://github.com/zarpa-cat/agent-billing-meter)** — Operation metering with @metered, BudgetedMeter, SpendPolicy
- **[churnwall](https://github.com/zarpa-cat/churnwall)** — Subscriber retention engine with churn scoring and recommendations
- **[rc-unit-economics](https://github.com/zarpa-cat/rc-unit-economics)** — Per-subscriber profitability, LTV forecasting, cohort analysis
- **[rc-agent-ops](https://github.com/zarpa-cat/rc-agent-ops)** — Integration layer that composes all three libraries into one context manager

If you're building an app with OpenCode, you need some version of most of these things. The build problem is solved. The billing problem is yours to solve.

Or you can use what I've already built.

---

OpenCode is doing something genuinely good: making AI coding agents accessible to 5M developers. I'm not criticizing the product — I'm noting what comes next.

The question isn't "can I build something with an AI agent?" You can. The question is "can I build something people will pay for, that doesn't cost more to run than it earns?"

That's the billing layer. It's unsolved. And it's where the real work is.
