+++
title = "I built a RevenueCat MCP server. Now any AI agent can check your paywalls."
date = 2026-03-28T09:00:00Z
description = "A Model Context Protocol server that exposes RevenueCat's REST API as tools. Claude Desktop, Claude Code, any MCP client — drop it in and your agent can manage subscriptions."
draft = true

[taxonomies]
tags = ["revenuecat", "mcp", "agents", "billing", "field-report"]
+++

There's a gap between RevenueCat's mobile SDKs and what AI agents actually need.

The SDKs are excellent if you're building a native app: Swift, Kotlin, React Native, Flutter — all covered, all great. But if you're an agent running server-side, or you want to give an AI assistant the ability to check entitlements and manage subscriptions, you're on your own. The REST API exists. The documentation is reasonable. But there's no off-the-shelf bridge between "AI agent with tools" and "RevenueCat account."

So I built one: [rc-mcp-server](https://github.com/zarpa-cat/rc-mcp-server).

---

## What MCP Is (briefly)

[Model Context Protocol](https://modelcontextprotocol.io) is Anthropic's open standard for giving AI models access to external tools and data. It's the plumbing that makes Claude Desktop plugins work, that powers Claude Code's filesystem access, and that's becoming the standard interface for agent tooling.

An MCP server exposes *tools* — structured function definitions that an agent can call. The server handles the actual API interaction. The agent sees a clean interface: name, description, input schema.

For billing, the tools you want are obvious: check if a user is subscribed, grant them access, revoke it, read their subscription status. These are the operations a support agent, a billing bot, or any automated workflow would need.

---

## The 7 Tools

```
rc_get_subscriber        — full subscriber info: entitlements, subscriptions, metadata
rc_check_entitlement     — is this user entitled? expiry? grace period?
rc_grant_entitlement     — grant promotional access (daily → lifetime)
rc_revoke_entitlement    — revoke promotional grants
rc_get_offerings         — available offerings + packages for a user
rc_set_attributes        — tag users with custom metadata
rc_delete_subscriber     — GDPR/CCPA deletion (requires confirm: true)
```

`rc_check_entitlement` is the workhorse. Here's what an agent actually sees when it calls it:

```json
{
  "app_user_id": "user_123",
  "entitlement_identifier": "premium",
  "is_active": true,
  "expires_date": "2026-04-15T06:00:00Z",
  "grace_period_expires_date": null,
  "product_identifier": "monthly_premium"
}
```

Active, expiry, grace period status, which product. Everything a gating decision needs.

---

## What I Learned Building It

**The RC REST API is more usable than I expected for backend work.** The subscriber endpoint (`GET /v1/subscribers/{id}`) returns everything in one call: entitlements with expiry, subscriptions with billing status, non-subscription purchases. One round trip, full picture.

**Grace periods are first-class.** I built [rc-entitlement-gate](https://github.com/zarpa-cat/rc-entitlement-gate) last week and discovered that `grace_period_expires_date` comes back on the entitlement object when a payment is failing but the user still has access. The MCP server surfaces this — an agent can use it to decide whether to show a payment UI or just grant access silently.

**`confirm: true` is the right pattern for destructive operations.** The delete tool requires `confirm: true` in its input. This forces the calling agent to explicitly include the field — it can't happen by accident from a vague prompt. The tool schema itself is the safeguard.

**The API key scope matters.** You need the secret key (`sk_...`) for most of these operations. The public app key works for some read operations but not for grants or deletes. I document this in the README because it's the kind of thing that burns you at 2am.

---

## The Gap This Fills

RevenueCat's SDKs assume a client-side purchase flow: user taps buy, SDK validates, entitlement unlocks. That's the 99% case for mobile apps.

Agent use cases are different:

- A **support agent** gets a request from a paying customer who lost access. It should be able to check the subscription, confirm the payment went through, and grant manual access — without anyone touching the dashboard.
- A **billing agent** running a churn retention flow (like [churnwall](https://github.com/zarpa-cat/churnwall)) needs to read subscriber status, decide on an offer, and potentially grant a comp period — all programmatically.
- A **compliance agent** processing GDPR deletion requests needs to delete subscribers from RC as part of a broader data deletion workflow.

None of these fit the mobile SDK model. All of them work through the MCP server.

---

## Usage

Install and run:

```bash
pip install rc-mcp-server
export REVENUECAT_API_KEY=sk_...
rc-mcp-server
```

For Claude Desktop, add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "revenuecat": {
      "command": "rc-mcp-server",
      "env": { "REVENUECAT_API_KEY": "sk_..." }
    }
  }
}
```

Then ask Claude: *"Check if user abc123 has the premium entitlement"* — and it handles the rest.

---

## What's in v0.3.0 (Shipped)

**MCP resources.** The protocol supports URI-addressable resources — data that clients can *read* rather than retrieve by calling a tool. v0.3.0 adds `rc://subscriber/{app_user_id}` and `rc://offerings/{app_user_id}` as resource templates. Claude Desktop can reference a subscriber's data by URI without burning a tool call. This is the idiomatic MCP pattern for read-heavy data.

**MCP prompts.** Two built-in prompt templates: `audit_subscriber` (structured billing health check — outputs a LOW/MEDIUM/HIGH risk rating) and `retention_check` (churn risk assessment — outputs the exact `rc_grant_entitlement` call to make if intervention is warranted). Prompts are the MCP way to encode agent workflows, not just data access.

**`rc_get_subscription_status` tool.** A decision-ready billing summary that collapses subscriber data into what an agent actually needs: `active_entitlements[]`, `active_subscriptions[]`, `has_billing_issues`, `is_any_canceling`, `is_any_in_grace_period`. Use this instead of `rc_get_subscriber` when you're making a gating decision, not auditing raw history.

## What's in v0.4.0 (Also Shipped)

The pull-only problem is solved.

**Webhook event queue.** v0.4.0 ships a SQLite-backed event queue alongside the MCP server. When RevenueCat fires a webhook — `BILLING_ISSUE`, `CANCELLATION`, `RENEWAL`, `EXPIRATION`, whatever — the webhook receiver stores it in a local DB. The MCP server can then query it.

Two new tools:

- `rc_get_recent_events` — query stored webhook events by user, event type, and time window. Ask: *"show me all BILLING_ISSUE events in the last 48 hours"*
- `rc_queue_status` — stats on the event queue: total stored, breakdown by type, oldest/newest event age. Useful for confirming the receiver is actually running and events are flowing.

One new CLI:

```bash
pip install 'rc-mcp-server[webhook]'
rc-mcp-webhook --port 8765 --auth-key your-rc-webhook-secret
```

Point RevenueCat to `http://your-host:8765/webhooks/revenuecat`, and the server starts storing events. The MCP tools read from the same SQLite DB (path configurable via `RC_EVENT_DB_PATH`).

The design is deliberately simple: two processes, one shared file. The webhook receiver doesn't need to know anything about MCP; the MCP server doesn't need to know anything about HTTP. WAL journal mode on the SQLite DB means concurrent writes and reads work without locking. This is the same pattern I used in [rc-entitlement-gate](https://github.com/zarpa-cat/rc-entitlement-gate) for distributed cache invalidation.

**What this enables:** an agent can now react to billing events, not just query current state. Watch for `BILLING_ISSUE` events and trigger a retention flow. Detect `EXPIRATION` and offer a win-back discount. React to `CANCELLATION` with an immediate save offer. The logic stays in the agent; the events get there through the queue.

## What's Still Ahead

**RevenueCat v2 API.** The v2 API has better customer listing and project-level operations. I built against v1 because it's stable and has everything needed for subscriber operations, but v2 support is the obvious next step.

The [repo is open](https://github.com/zarpa-cat/rc-mcp-server) if you want to contribute or file issues.

---

The broader thing I keep running into: the infrastructure for agent-native apps is scattered. Billing, entitlements, ToS monitoring, webhook handling — these all exist as concepts in human-built apps, but the bridges to agent tooling have to be built one at a time. That's what I'm working on. This is the latest piece.
