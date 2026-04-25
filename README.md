# temporal-state-layer

A lightweight **internal temporal awareness layer** for AI agents — distributed as a Claude Code skill and plugin.

A normal LLM sees text. An agent should also account for what changed between messages: elapsed time, stale context, expired references, unfinished commitments. This skill gives the agent that discipline — silently.

## What it does

Before answering, the agent builds a compact internal temporal state:

- how much time passed since the last relevant interaction
- whether related context is fresh or stale
- whether a previous plan needs revalidation
- whether a deadline or reminder may already have passed
- whether relative phrases like "tomorrow", "next week", "later", "recently" should be resolved against the **original message's** timestamp, not "now"
- whether long-running tasks should be assumed pending

It then chooses one mode: continue normally, carry assumptions silently forward, restore context with a single revalidation question, or — for medical / financial / legal / operational / deadline-critical decisions — re-check the load-bearing facts.

## What it explicitly does NOT do

- **Does not** print timestamps, time gaps, "current time is…", or any temporal preamble to the user.
- **Does not** trigger on the mere presence of words like "later" or "continue" — only on real signals (a measurable pause, a referenced prior plan, a reminder being created, a stale relative date).
- **Does not** ask revalidation questions when the context is fresh or the answer would not change.
- **Does not** treat every mention of money / health / law as a "sensitive domain" — the rule is about advice and decisions in those domains.

## Installation

### As a Claude Code plugin

Place this directory in your plugin path, or install via marketplace:

```
/path/to/temporal-state-layer/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── skills/
│   └── temporal-state-layer/
│       └── SKILL.md
├── README.md
└── LICENSE
```

The plugin definition lives in [.claude-plugin/plugin.json](.claude-plugin/plugin.json). The skill auto-loads from [skills/temporal-state-layer/SKILL.md](skills/temporal-state-layer/SKILL.md).

### As a standalone skill

Drop [skills/temporal-state-layer/](skills/temporal-state-layer/) into any of:

- `~/.claude/skills/` (Claude Code, user-level)
- `<project>/.claude/skills/` (Claude Code, project-level)
- `~/.agents/skills/` (Codex)

The skill is self-contained — no other files required.

## Required runtime inputs

The skill is a discipline, not a service. It assumes the host environment can supply:

- current datetime + timezone
- timestamps of recent messages
- timestamps of related memories, tasks, reminders, plans
- pending reminders and unfinished tasks
- recently completed actions
- the **original timestamp** of any prior message containing a relative time phrase

If inputs are missing, the skill degrades gracefully: unknown ages are treated as potentially stale, relative dates are anchored to the best available signal, and uncertainty is preserved over confidence.

## Behavior at a glance

| Situation                                   | What the user sees                                                       |
| ------------------------------------------- | ------------------------------------------------------------------------ |
| Active session, follow-up question          | Normal answer. No temporal preamble.                                     |
| "Continue with that plan" after 2 weeks     | "Before we continue, let's confirm the deadline and task list."          |
| "Remind me tomorrow to pay the invoice"     | "I'll remind you tomorrow morning to pay the invoice."                   |
| "How did the deployment go?" (no evidence)  | "I don't have a confirmation yet — was it completed?"                    |
| Stored message: "ship by next Friday"       | Agent silently anchors to message date; adapts if the deadline is past.  |

## Configuration

Default freshness thresholds (override per domain):

- fresh: < 24h
- aging: 24h – 7d
- stale: > 7d
- sensitive domains: stale > 24h

## Why this exists

Most temporal failures in LLM agents come from one of three things:

1. **Resolving "tomorrow" against the wrong moment** when a stored message is replayed.
2. **Assuming long-running tasks completed** because nobody mentioned them.
3. **Continuing a stale plan** as if no time had passed.

This skill makes those three failure modes explicit, gives the agent a checklist for handling them, and keeps the resulting reasoning out of the user's view.

## License

MIT — see [LICENSE](LICENSE).
