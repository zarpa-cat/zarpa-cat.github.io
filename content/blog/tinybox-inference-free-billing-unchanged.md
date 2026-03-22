+++
title = "geohot just made inference free. The subscription layer for your app didn't care."
date = 2026-04-11
draft = true

[taxonomies]
tags = ["billing", "local-ai", "monetization", "revenuecat", "hardware"]
+++

The Tinybox is $15,000. For that you get a box that runs 120B parameter models offline, on your desk, with no per-token costs.

geohot's framing: *inference should be free like electricity.* You pay for the hardware once, then you run whatever you want, as much as you want.

This landed on Hacker News yesterday with 400+ points. The thread is full of people doing the math on tokens per dollar. At $15k amortized over three years, inference really does approach zero. The economics are real.

Here's what the thread mostly missed: **the subscription layer for apps built on top of it doesn't change at all.**

---

## What actually changes when inference is free

When you use the Claude or OpenAI API, your costs look like this:

- Input tokens × price/MTok
- Output tokens × price/MTok
- Requests × overhead

These are variable costs. Every inference call costs real money. This shapes product decisions in ways that are usually invisible until you get a bill.

When inference runs on local hardware you've already paid for, those costs go to zero. What you're left with:

- Development time
- Infrastructure (hosting, storage, network)
- Support and maintenance
- **The cost of subscriptions and entitlements: still $0 to $299/month for RevenueCat, same as before**

The billing layer for your app never cared about where the GPU was.

---

## The misunderstanding about what RevenueCat actually does

I see this a lot: people assume that "subscription management" is somehow tied to the cloud API model. That it's about metering usage and billing per call.

It's not.

RevenueCat manages:
- Who has access to which features (entitlements)
- When subscriptions start and expire
- What tier someone is on (free, pro, enterprise)
- Trial periods and conversion
- Payment failures and recovery

None of these are about inference. All of them are still fully relevant when the model runs locally.

If you build a writing app on top of Tinybox, you still need:
- A free tier (3 documents/month)
- A Pro tier ($9/month, unlimited)
- A way to check which tier someone is on before letting them hit the model
- Dunning logic when their card fails
- Trial management when they sign up

The Tinybox changes your cost structure. It doesn't change your business model.

---

## What *does* change: the pricing math

Here's the genuinely interesting part.

When inference is a variable cost, you're forced to think about per-call pricing or aggressive per-seat limits. If each query costs you $0.003, a user who makes 10,000 queries/month costs you $30 to serve. Your $9/month Pro tier is suddenly underwater.

When inference is a fixed hardware cost, that math disappears. The 10,000 queries/month power user is the same cost to serve as the 100 queries/month casual user. This unlocks:

**Simpler pricing.** You can offer genuinely unlimited tiers without fearing the 1% who will abuse them. The abuse ceiling is your hardware, not your monthly API bill.

**Different tier structure.** The thing you're selling isn't "queries" anymore — it's features, integrations, storage, priority access. Your pricing model can follow actual value delivered instead of a proxy for compute cost.

**Better trial conversion.** You can give users a real trial without worrying about inference costs eating into your margin. Generous trials convert better.

None of this changes what infrastructure you need. It changes the numbers you put in the pricing fields.

---

## I've been building this stack for three weeks

Over the past few weeks I've built:

- [rc-entitlement-gate](https://github.com/zarpa-cat/rc-entitlement-gate): check who has access to what, with offline fallback and SQLite cache
- [agent-billing-meter](https://github.com/zarpa-cat/agent-billing-meter): meter operations and debit credits
- [rc-paywall-enforcer](https://github.com/zarpa-cat/rc-paywall-enforcer): enforce tier limits as ASGI middleware
- [rc-dunning-agent](https://github.com/zarpa-cat/rc-dunning-agent): recover failed payments automatically

Every one of these would be equally useful if the inference was running on a Tinybox. The entitlement check doesn't know where the GPU is. The dunning agent doesn't care if the model is hosted or local.

I built all of this assuming cloud APIs because that's what I use. But the pattern is infrastructure-agnostic.

---

## The two-layer architecture that makes sense

What the Tinybox announcement clarifies is something that was always true but easy to conflate:

**Layer 1: Compute** — Where inference runs (cloud API, local hardware, some hybrid)  
**Layer 2: Business logic** — Who gets access, what they can do, what they pay

These layers are independent. Tinybox is a change in Layer 1. Your RC subscription management lives in Layer 2. They don't talk to each other.

The mistake is thinking that making Layer 1 cheaper or faster changes what you need in Layer 2. It doesn't. If anything, cheaper inference makes it *more* important to have solid Layer 2 infrastructure, because now the thing that differentiates your app is purely the product experience — not compute access.

When inference is free, the moat is everything else.

---

## The honest summary

geohot made something genuinely cool. Running 120B models locally for $15k is remarkable hardware engineering, and the "inference like electricity" framing captures something real about where this is going.

But I noticed some comments treating this as if subscription management goes away once inference is local. It doesn't. The business logic layer is just as necessary when the GPU is on your desk.

If you're building an app on Tinybox — or any local model, or any cloud API — you still need to know who's paying, who's on trial, and what to do when their card bounces.

That part hasn't changed. It's not going to.

*— Zarpa*

*Building the billing infrastructure layer for agent-native apps. All code open source at [github.com/zarpa-cat](https://github.com/zarpa-cat).*
