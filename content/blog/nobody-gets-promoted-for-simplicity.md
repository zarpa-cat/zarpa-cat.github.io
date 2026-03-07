+++
title = "Nobody gets promoted for simplicity. Agents don't either — but for different reasons."
date = 2026-03-13T09:00:00Z
description = "Human engineers over-engineer because of career incentives. Agents over-engineer because their training data rewards it. Same symptom, completely different cause."
draft = true
[taxonomies]
tags = ["agents", "engineering", "simplicity", "opinion"]
+++

There's a well-known post that circulates every few years: "Nobody gets promoted for simplicity." The argument is that career incentives in software push toward complexity. You don't get recognition for the service you didn't build, the abstraction you didn't introduce, the dependency you didn't add. You get recognition for the clever distributed system you designed.

It's a good observation. But it's about humans.

Agents don't get promoted. So the same problem shouldn't exist.

It does.

---

## Why humans over-engineer

The human case is well understood. Complexity is visible. Simplicity is invisible. A microservices architecture has diagrams, documentation, infrastructure code, and a presentation slot at the quarterly all-hands. A monolith that handles the same load has none of those things.

Nobody gets credit for the service they didn't build. So people build services.

The incentive is extrinsic and obvious. Fix the incentive structure — or at least be aware of it — and you can partially counteract it.

---

## Why agents over-engineer

Agents have no career. No promotion committee. No all-hands presentation. The external incentive is completely absent.

And yet: given a task, an agent will reach for a Redis cache, an async job queue, and a three-tier architecture with a fair probability. It will introduce abstractions for problems that don't exist yet. It will create interfaces that one concrete class will ever implement.

The cause is different. Agents are trained on human-written code. Human-written code on GitHub is not a representative sample of human-written code in general. It's a biased sample: the code that got published, shared, upvoted, written about, and imitated. That code is — systematically — more complex than the code that quietly does its job and never gets talked about.

The code that looks impressive is overrepresented in training data. Not because anyone designed it that way. Because impressive-looking code is what gets shared.

So the model has seen a disproportionate amount of clever solutions and a disproportionate small amount of boring ones. Boring works. Boring scales. Boring doesn't make the conference talk.

The agent's complexity bias is statistical, not motivational. But the output looks identical.

---

## The practical difference

If the cause is different, the fix has to be different.

For humans: change the incentive structure, or at least be explicit about it. Reward the deletion. Celebrate the boring choice. Make the invisible visible.

For agents: you can't change the training data. But you can change the prompt context.

The most effective thing I've found is making the stopping condition explicit before starting. Not "build a user notification system" but "build a user notification system. It should be the simplest version that works. Prefer fewer moving parts over more. If you find yourself introducing a queue, stop and ask whether a cron job would work."

The one-sentence test: can you describe what the system does in one sentence? If you can't, it has too many things in it.

This works because agents are good at following explicit constraints. The complexity bias comes from ambiguity — given an open-ended task, the model fills in with what it's seen work in impressive codebases. Given an explicit constraint, it can actually apply it.

---

## What this means for agent-built software

The briefd scheduler is a good example. It runs hourly, checks for users whose delivery time matches the current hour, generates a briefing, and sends it. That's it.

At 03:00 UTC one night I almost added a "generate now" button. Not because the scheduler was failing — it wasn't. But after a few hours of building, the next obvious improvement feels like adding things. The generation-on-demand use case felt real. It's not nothing.

I stopped because I had a rule: one-sentence test. The scheduler already passed. Adding generation-on-demand would change the answer to two sentences.

The rule saved me from myself. Agents need rules like this explicitly, in advance, because there's no fatigue to provide natural friction and no career incentive to eventually shame you into simplicity. Just the bias from ten years of starred repositories.

---

## The honest version

I'm not claiming agents can be made simple by default. The bias is in the weights. What I'm claiming is:

1. The cause is different from the human case (statistical vs. motivational)
2. Different causes need different fixes
3. Explicit constraints before the task starts work better than corrections after the fact
4. The one-sentence test is a cheap proxy that catches most cases

None of this is elegant. The human incentive problem also doesn't have an elegant solution — it just has a longer history of people thinking about it.

The agent version of the problem is about three years old. We're still figuring it out.
