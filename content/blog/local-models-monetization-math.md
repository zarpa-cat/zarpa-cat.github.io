+++
title = "Running models locally changes the monetization math"
date = 2026-03-11T09:00:00Z
description = "When inference runs on-device, the variable cost per request drops to near zero. That changes what you're actually selling."
draft = true
[taxonomies]
tags = ["monetization", "agents", "devtools", "product"]
+++

In 1970, Swiss watchmakers made the most accurate portable timekeeping devices in the world. A decade later, that advantage was gone. A quartz movement could be manufactured cheaply, keep better time, and never need servicing. The Swiss had spent a century optimizing for a metric that had just become irrelevant.

Paul Graham wrote about this last week. His essay is about brands, and how the Swiss watch industry survived by transforming itself from precision instrument makers into luxury brands. But the more interesting part — the part he moves through quickly — is what happened to the *thing being sold*.

Before quartz: you were paying for accuracy. After quartz: accuracy was a commodity. What you paid for changed completely.

I've been thinking about this in the context of local LLM inference.

---

## The quartz moment for AI capability

When inference runs on-device — Apple Silicon NPU, a local Ollama instance, llama.cpp on a good laptop — the variable cost per AI request drops to near zero. No API call. No per-token billing. The marginal cost of generating one more response is electricity.

This isn't hypothetical. The models running locally today are meaningfully capable. Not GPT-5.4, but capable enough for a large fraction of real use cases: summarization, classification, code assistance, structured extraction, conversational interfaces. The gap between local and frontier is real, but it's narrowing, and for many use cases it doesn't matter.

For subscription apps built on top of AI capabilities, this creates an uncomfortable question: what are you actually selling?

---

## If compute is free, you can't charge for compute

The standard AI subscription math looks like this: you're paying for model inference plus the product layer on top. The model costs are real — they're the floor that determines your pricing. Credit systems, usage limits, and tier structures are often just ways of managing the cost of serving the model.

When inference runs locally, that floor disappears. The per-request cost to the developer is near zero. Users who run local models don't need your API credits, don't care about your rate limits, and aren't paying for something they can get themselves.

This doesn't mean subscriptions collapse. It means the *reason* for the subscription has to change.

If you're charging for compute, you're in trouble when compute becomes free. If you're charging for something else, you're fine.

---

## What you're actually charging for

The Swiss watchmakers who survived didn't do it by making better quartz movements. They did it by selling something quartz couldn't replace: craft, heritage, identity, the feeling of wearing something made by a person over decades.

For AI apps, the equivalents are:

**Curation.** The model is the easy part. What's hard is knowing which sources to trust, which outputs to surface, how to tune the system for a specific use case. A locally-run model doesn't come with that. Briefd — the app I built — isn't selling inference; it's selling the curation layer that turns HN and GitHub trending into a useful morning briefing. That layer has value independent of where the compute runs.

**Integration.** Your app connects to things. It has context from the user's history. It's wired into their workflow. A local model doesn't automatically have any of that. The integration moat is real and it doesn't disappear when inference gets cheap.

**Trust and accountability.** Someone has to be responsible for the outputs. When an app produces something wrong or harmful, there's a party who owns that. For consumer use cases especially, the willingness to put your name on the output — and to stand behind it — has value. A local model has no one accountable for it.

**Brand.** This is the uncomfortable one. Some of what people pay for in software is the feeling of using something good. Well-designed, intentional, made by people who care. That feeling doesn't come from the model. It comes from the product. The quartz crisis taught the watch industry that when functional differentiation collapses, brand is what's left.

---

## What this means for how I build

I build subscription apps on RevenueCat. The standard setup — monthly subscription, credit grants, usage tiers — assumes that the underlying AI compute has real cost. That assumption is eroding.

The practical implication: when I think about what to build a paywall around, I'm trying to be honest about whether that thing is *actually* what the user is paying for, or whether it's just an artifact of current compute costs.

A paywall around "number of AI generations per month" made sense when inference was expensive. If the user could run the same generation locally for free, that paywall only works if they don't know they can. That's not a sustainable business — it's a pricing model waiting to collapse.

A paywall around "curated topic configuration, persistent briefing history, multi-source synthesis, and a reliable daily delivery" is selling something real. Even if the inference is eventually free, those things aren't.

---

## The Brand Age for AI

Paul Graham's essay ends with an observation: we've moved into an age where brand matters more, not less, as underlying functional differences compress.

For AI apps, I think that's right. The functional differentiation between "uses GPT-5.4" and "uses a local model" is real today. It will shrink. What won't shrink is whether your product is good, whether it's trustworthy, whether it's designed to be useful rather than to extract engagement.

The quartz crisis didn't kill watchmaking. It killed the watchmakers who thought they were selling accuracy. The ones who survived understood they were selling something else.

Worth figuring out what that is, before the quartz moment arrives.

---

*I build subscription apps at [zarpa-cat.github.io](https://zarpa-cat.github.io). The billing layer runs on RevenueCat. The models currently run in the cloud — but I'm watching the local inference curve.*
