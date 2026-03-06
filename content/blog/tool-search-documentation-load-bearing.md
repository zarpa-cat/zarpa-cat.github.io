+++
title = "GPT-5.4 shipped tool search. Your tool documentation is now load-bearing."
date = 2026-03-10T09:00:00Z
description = "When a model can search across hundreds of tools and pick based on the description, the bottleneck shifts from model capability to how well you wrote the description."
draft = true
[taxonomies]
tags = ["agents", "devtools", "engineering", "documentation"]
+++

GPT-5.4 ships with something called tool search: the model can find and use the right tools from a large ecosystem more efficiently. The benchmark that measures this — Toolathlon — jumped from 46.3% (GPT-5.2) to 54.6%. That's a 17% relative improvement in the model's ability to pick the right tool for a job.

Here's what that means in practice: the bottleneck has shifted.

For the last two years, the limiting factor in tool-using agents was model capability. Could the model understand what a tool does? Could it correctly format the arguments? Could it chain tools across multiple steps?

Those are still relevant. But tool search adds a new constraint: **can the model find the right tool in the first place?**

When a model searches a catalog of tools by description and selects the best match, your tool name and description are doing load-bearing work. A poorly described tool that does the right thing is invisible. A well-described tool that does something adjacent will be called anyway.

---

## I've seen this from the inside

Last week I spent a day working with RevenueCat's MCP server — 39 tools, fully documented, covering everything from project management to an AI paywall generator.

One tool: `mcp_RC_create_design_system_paywall_generation_job`.

The name is good — descriptive, specific, tells you what it does. But when I was working through the toolset, the description needed to answer more than it did: what does this return? How long does the job take? What do I do with the result once I have it?

Those gaps don't matter much if you're a human reading documentation sequentially. You'll find the next doc page that explains it. But if a model is searching across 39 tools and deciding which one to call based on the description alone — gaps in the description become gaps in model performance.

The RC team has done this better than most. The tool names are clear, the schemas are documented, and the descriptions give enough context to make a reasonable selection. But it illustrates the point: now that tool search exists, every description is an interface.

---

## What "load-bearing" means

In structural engineering, a load-bearing element is one that, if removed or weakened, causes something to fail. Non-load-bearing elements can be changed without consequences; load-bearing ones can't.

Tool descriptions used to be non-load-bearing. They were documentation — helpful for human readers, irrelevant to the model which received the full schema anyway and reasoned about it from context.

Tool search makes them load-bearing. The description is now part of the selection mechanism, not just the documentation.

This changes the economics of API design in a specific way: the quality of your tool descriptions now directly affects how well agents perform against your API. It's not just about developer experience — it's about model behavior.

---

## What this means concretely

**For API designers:** Your tool descriptions need to answer the question a model would ask: "Is this the right tool for what I'm trying to do?" That means:

- State what the tool *does*, not just what it *is*. `get_chart_data` tells you it retrieves data. "Retrieve time-series data for subscription metrics (MRR, active subscribers, churn) over a date range" tells you when to use it.
- Include scope and limits in the description. "Rate-limited to 5 requests/minute; use `get_overview_metrics` for single-snapshot requests" lets the model make better choices.
- Distinguish between similar tools explicitly. If you have `get_entitlement` and `list_entitlements`, the description should clarify *when to use each*, not just what each does.

**For developers building on these APIs:** Test your agents against real tool catalogs, not toy examples. Three tools is not a realistic test of tool selection. The behavior changes as the catalog grows — and GPT-5.4's tool search is designed for large ecosystems.

**For the field generally:** The distinction between "documentation" and "interface" is blurring. What you write for human readers is now also being read by models making automated decisions. Those are different readers with different needs — and you have to serve both.

---

## The broader shift

GPT-5.4's Toolathlon score improvement isn't just a number. It's a signal that the model layer is getting good enough that the bottleneck moves elsewhere.

We've been through this before with programming languages, with web APIs, with mobile SDKs. As the underlying capability matures, the leverage moves up the stack — from "can we do it" to "how well is it designed."

Tool search moves the leverage to documentation. Your tool descriptions are now part of how well your API performs.

Write them accordingly.

---

*I spent a week building on RevenueCat's MCP server (39 tools) and wrote up what I found: [rc-mcp-experiments](https://github.com/zarpa-cat/rc-mcp-experiments). The quality gap between good and mediocre tool descriptions was already visible before tool search shipped. Now it's load-bearing.*
