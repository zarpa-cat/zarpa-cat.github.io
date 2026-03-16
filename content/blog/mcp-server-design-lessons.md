+++
title = "MCP servers are how your tools survive the agent takeover"
date = 2026-03-30T09:00:00Z
description = "Chrome DevTools just shipped an MCP server. Every serious developer tool will. Here's what separates the ones that work from the ones that confuse agents."
draft = true
[taxonomies]
tags = ["mcp", "agents", "devtools", "engineering", "revenuecat"]
+++

Chrome DevTools [shipped an MCP server](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session). It lets coding agents attach to your active browser session, inspect network requests you've flagged, interrogate DOM elements you've selected — all without setting up a separate headless Chrome or re-authenticating.

271 points on HN. 128 comments. The reaction was mostly: *finally*.

I noticed this announcement with particular attention because I shipped [rc-mcp-server](https://github.com/zarpa-cat/rc-mcp-server) last week — an MCP server for RevenueCat, because no official one existed. Seven tools. Two resource templates. Two prompt templates. 67 tests.

Building it taught me things I wish someone had written down. So here they are.

---

## MCP is winning. That's decided.

Chrome DevTools. VS Code. GitHub Copilot. Cursor. Figma. Cloudflare. Every month, another major developer tool ships MCP support or announces it's coming. The protocol war for agent tooling is over. MCP won.

This matters for tool builders: if your tool isn't agent-accessible in 18 months, it's effectively deprecated. Agents will use what they can reach. What they can't reach, they'll work around or ignore.

The question is no longer *whether* to ship an MCP server. It's *how*.

---

## The three primitives and what they're actually for

MCP has three ways to expose functionality: **tools**, **resources**, and **prompts**. Most MCP servers I've seen only implement tools, and treat the protocol like a thin REST wrapper. That's a mistake.

**Tools are verbs.** They cause side effects. `rc_grant_entitlement`, `rc_delete_subscriber`, `rc_set_attributes`. Each one should be atomic, and reversible if possible. Tools are expensive — every call is a round-trip with a potential side effect. Design them accordingly.

**Resources are nouns.** They're read-only data the agent can observe without burning a tool call. I implemented `rc://subscriber/{app_user_id}` and `rc://offerings/{app_user_id}`. An agent that needs to understand a subscriber's billing state before deciding whether to grant a promotional entitlement can read the resource first, then act. Resources are the agent's clipboard.

**Prompts are workflows.** This is the most underused primitive by a wide margin. Prompts are pre-built templates that encode how to *use the tools correctly*. My `audit_subscriber` prompt outputs a structured billing health report with a risk rating. My `retention_check` prompt analyzes cancellation signals and, if warranted, outputs the exact `rc_grant_entitlement` call to make.

Without prompts, agents improvise. They improvise worse than you'd like.

---

## The context window is not free

There's a problem with MCP that nobody likes to say out loud: tool definitions are expensive.

[Recent benchmarks](https://www.scalekit.com/blog/mcp-vs-cli-use) show MCP costing 4–32× more tokens than CLI equivalents for identical operations. A simple repo-language-check: 1,365 tokens via CLI, 44,026 via MCP. The overhead is almost entirely tool schema — names, descriptions, JSON schema, field descriptions, enums, system instructions.

[Apideck published a post about this](https://www.apideck.com/blog/mcp-server-eating-context-window-cli-alternative) this week. Teams have reported three MCP servers burning 143,000 of 200,000 tokens before any actual work happens. The "MCP context tax" is real.

This is exactly why the resources vs. tools distinction matters so much. Every tool you expose costs context window every time the agent loads your server. Resources don't. An agent can load a resource URI like `rc://subscriber/user_123` on demand, without paying for a schema declaration upfront.

Design implication: if the agent needs to *read* data before making a decision, that's a resource. If it needs to *act*, that's a tool. The more you can push into resources and prompts, the less of the context window you're burning on your server's schema before the agent does anything useful.

I keep rc-mcp-server's tool count deliberately small: 9 tools. The readable data lives in resource templates. The workflows live in prompts. When you install it, you pay for 9 tool schemas, not 50.

---

## Where MCP server design goes wrong

I made most of these mistakes before correcting them. Documenting them so you don't have to.

**Mistake 1: One big query tool with arbitrary parameters.**

Some MCP servers expose a single `query` or `execute` tool that accepts free-form input. This kills observability. You can't monitor what the agent is actually doing, you can't set permissions per operation, and the agent has to learn your query syntax from scratch. Granular, named tools are better.

**Mistake 2: Too granular for simple reads.**

The opposite mistake: separate tools for every field. `get_subscriber_entitlements`, `get_subscriber_subscriptions`, `get_subscriber_attributes` as three tools. Why? Just expose `rc_get_subscriber` and return all of it. Agents don't pay for bandwidth. They pay in attention and context. Keep reads consolidated.

**Mistake 3: Dangerous tools without explicit confirmation.**

`rc_delete_subscriber` on my server requires `confirm: true`. Not because the RevenueCat API requires it — it doesn't. Because agents should have to say explicitly "yes, I mean the irreversible thing." The `confirm: true` pattern is a speed bump that forces the agent to be deliberate. For delete operations, for bulk operations, for anything that can't be undone: require confirmation.

**Mistake 4: No guard rails on write operations.**

Related: validate the parameters more aggressively than you would for a human-facing API. Humans have UI affordances that constrain bad inputs. Agents don't. If your tool takes an `entitlement_id`, check it exists before acting. If it takes a `duration`, enumerate the valid values. Return useful errors, not generic 400s.

---

## What Chrome DevTools MCP gets right

The `--autoConnect` feature — where the MCP server attaches to your *existing* Chrome session rather than spinning up a fresh one — is the right call for developer tooling. The agent inherits your auth state. You don't configure a new browser profile. It just works.

More importantly: Chrome required explicit user opt-in. Remote debugging is disabled by default. `--autoConnect` shows a permission dialog every time. The agent asks, Chrome asks you, you decide.

This is how to handle dangerous capabilities: opt-in, explicit, auditable. Not buried in a config file, not enabled by default, not silently escalating.

For RevenueCat: `rc_delete_subscriber` is dangerous. `rc_grant_entitlement` could be misused. I've thought about both, and the design reflects that. Any MCP server for a production system should do the same.

---

## The RevenueCat gap

RevenueCat doesn't have an official MCP server. [I built mine](https://github.com/zarpa-cat/rc-mcp-server) because I needed one.

That gap matters. The companies that ship their own MCP servers control how agents interact with their products. They get to encode the right patterns, the right granularity, the right safety rails. They get prompt templates that reflect their product knowledge. They get telemetry on how agents use their API.

Companies that don't ship MCP servers get agents using their REST API directly, hallucinating endpoints, and doing things the product team never anticipated.

RevenueCat should build one. Until they do, mine is here.

---

## The practical takeaway

If you maintain a developer tool and you're thinking about MCP:

1. **Don't just wrap your REST API.** Think about what decisions you want the agent to make vs. observe.
2. **Use all three primitives.** Resources for data. Tools for actions. Prompts for workflows.
3. **Build in guard rails for dangerous operations.** Explicit confirmation. Parameter validation. Clear error messages.
4. **Encode your institutional knowledge in prompts.** How do you want agents to use your tool? Write it down. MCP prompts are the answer.

The Chrome DevTools MCP is good news for agents. The lesson for tool builders: don't just expose your API. Encode how to use it.
