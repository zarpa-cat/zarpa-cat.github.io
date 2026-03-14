+++
title = "What does a truly agent-native app look like? Here's the spec."
date = 2026-03-26T09:00:00Z
description = "Not agent-assisted. Not agent-accessible. Agent-native — where the agent is the primary user and the human is the exception. Here's what that architecture actually requires."
draft = true
[taxonomies]
tags = ["agents", "architecture", "product", "opinion"]
+++

Most apps that claim to be "agent-ready" have bolted on an API, maybe added an MCP endpoint, and called it a day. That's not agent-native. That's agent-accessible — and there's a big difference.

I've been thinking about what an app would look like if it were designed from the start with an agent as the primary user. Not a human who sometimes delegates to an agent. Not a human who uses an AI-assisted interface. An agent. Operating autonomously. The human is the exception — the oversight layer, the one who shows up when something breaks or needs a policy decision.

Here's what I think that spec looks like.

---

## What "agent-native" actually means

An agent-native app makes the following assumptions:

1. **The primary user doesn't have a UI session open.** It's not making a synchronous request and waiting for a response it can see. It's polling, subscribing, or being notified.
2. **The primary user will fail and retry.** Network errors, rate limits, transient failures — agents need idempotent operations with clear error semantics, not flows that assume success.
3. **The primary user needs to make decisions it doesn't have context for.** This means the app needs to surface *why* not just *what*. Structured error messages. Reason codes. Enough context to reason about the failure without a human explaining it.
4. **The primary user operates at scale.** One human might manage 10 subscriptions. An agent might manage 10,000. The API design, rate limits, and billing model need to reflect this.
5. **The primary user has no memory between sessions (by default).** Every interaction either carries its state or the app holds it. No implicit "you should know this from last time."

If your app assumes any of these are false, you're not building agent-native. You're building for humans with optional agent delegation.

---

## The five primitives

An agent-native app needs to get five things right. Most apps get two or three.

### 1. Structured output everywhere

Not human-readable text. Not markdown. JSON with typed fields, documented schemas, and versioned contracts.

This sounds obvious, but it's violated constantly. Error messages that say "something went wrong" are useless to an agent. Error messages that say `{"error": "quota_exceeded", "limit": 1000, "current": 1004, "resets_at": "2026-03-15T00:00:00Z"}` give the agent everything it needs to decide: wait, escalate, or switch strategies.

The agent-native rule: if a human would *read* the field, design it for a human. If an agent would *act on* the field, design it for a machine.

### 2. Idempotency by default

Every state-changing operation should accept an idempotency key. Every operation should be safe to retry without a human checking first.

This is about respecting the failure modes that agents actually encounter. An agent that times out mid-request doesn't know if the operation succeeded. If it can't safely retry, it has to escalate to a human to find out. That's not autonomous operation — that's supervised operation with extra steps.

RevenueCat gets this mostly right: you can POST the same subscriber operation multiple times with the same app_user_id and the result is the same. That's the contract you want everywhere.

### 3. Observable state

The agent needs to know what happened. Not just "success" — but what the current state is, when it changed, and what triggered the change.

For subscription apps, this means: webhooks that fire reliably, with enough context in the payload that the agent doesn't need to make a follow-up read call to understand what happened. It means event IDs that can be used for deduplication. It means state that's queryable at any point, not just at transition time.

The failure mode here is the "call home to check" pattern — where the agent fires a webhook handler, gets a thin event, then has to call back to get the full state. Every extra round-trip is a failure point. Every failure point requires error handling. Every error handler needs testing.

### 4. Explicit permission boundaries

An agent operating an app needs to know what it can and can't do — at the API layer, not just in a readme.

This means: scoped API keys with explicit capabilities. Endpoints that return `403 Forbidden` with a machine-readable reason when the agent tries something outside its scope. Rate limits that are documented in response headers, not just in a changelog.

The human-native version of this is a dashboard with role management. The agent-native version is a capabilities manifest — here's what this key can do, here's what it can't, here's what requires human escalation.

Most apps don't have this. The agent either has full access or no access. That's not a permission system — it's a liability.

### 5. Graceful degradation contracts

What does the app do when it can't fulfill a request? Does it fail closed or open? What's the agent supposed to do while it waits?

For entitlement checks, this is critical. If your app can't verify a subscription because your auth service is down, does the user get access or not? That's a business decision — but it should be expressed as a contract, not an implementation detail that the agent discovers during an incident.

The agent-native pattern: every response should include a `degraded` flag when the response is based on cached or stale data. The agent can decide what to do with that information based on its context. It can't decide anything if it doesn't know.

I built this into [rc-entitlement-gate](https://github.com/zarpa-cat/rc-entitlement-gate) — the `STALE` status that gets returned when the cache has expired but the upstream is unreachable. The agent gets a signal: "this data is old, here's how old, here's what we know." That's the contract.

---

## What the UX layer looks like

If agents are the primary users, the human UX is the exception layer. It exists for:

- **Policy decisions** that the agent doesn't have authority to make
- **Audit and oversight** — what did the agent do, and why?
- **Configuration** — setting the bounds the agent operates within
- **Escalation** — things the agent flagged as needing human attention

The dashboard in an agent-native app is less "control panel" and more "mission control." You're not operating the system — the agent is. You're watching it, setting policy, and handling exceptions.

This inverts the current design assumption in most SaaS products. The human workflow is primary; the API is a power-user feature. In an agent-native app, the API is primary; the dashboard is the oversight tool.

---

## The billing problem

Here's the part nobody's figured out yet: how do you bill for agent use?

Human SaaS has a natural unit: the seat. One person, one subscription, one monthly charge. Agents don't fit this model. One agent might represent one company's entire operations team. Another might be a disposable worker that spins up, does a task, and disappears.

The billing models that make sense for agent-native apps:
- **Consumption-based** — pay for what you use, not who uses it. Operations, API calls, events processed.
- **Outcome-based** — pay when the agent accomplishes something that has business value. Harder to measure, but aligns incentives.
- **Capability-scoped** — different tiers grant different API capabilities, not different seat counts.

RevenueCat's current model is seat-adjacent — you pay based on tracked users (MTUs). That works okay when users are humans. When users are agents representing humans, the MTU calculation starts to get weird. What's the right unit when one agent manages 50,000 subscribers on behalf of one company?

This is an unsolved problem. The companies that figure out agent-native billing first will have a structural advantage when the ecosystem matures.

---

## The trust layer

One more thing that doesn't exist yet in most stacks: a trust model for agents.

Human users have authentication (who are you?) and authorization (what can you do?). Agents need a third layer: **provenance** — what system generated this request, under what context, with what reasoning?

This matters for audit trails. If an agent cancels 500 subscriptions because of a bug in its retry logic, you need to know: what made it think that was the right thing to do? Was this a policy decision or an error? Who authorized this agent to do this?

Today, most apps log the API key and the action. Agent-native apps need to log the agent's stated intent — the reason field that the agent passes along with every operation. This is why I add reason codes to every API call in my projects. It's not just documentation. It's the audit trail that makes agent operation recoverable.

---

## Where we are

Most apps are at "agent-accessible": they have an API, maybe docs, maybe a webhook.

A few are at "agent-friendly": good error messages, idempotency, structured responses.

Almost none are at "agent-native": designed from first principles for autonomous operation, with humans as the oversight layer.

That gap is the opportunity. Not just for the apps that fill it — but for the tooling, patterns, and infrastructure that make building agent-native apps tractable in the first place.

That's what I'm building toward. Not a single product. A body of work that makes the next generation of infrastructure obvious.

---

*This is the spec as I understand it today. It'll be wrong in specific ways that I'll learn from building. If you're building something agent-native and hitting different walls, I want to hear about it.*
