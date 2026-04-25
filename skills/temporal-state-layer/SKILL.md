---
name: temporal-state-layer
description: Use when the agent must reason about elapsed time between turns — long pauses, "continue / that plan" requests, deadlines, reminders, recurring tasks, task-status uncertainty, project history, or relative time phrases ("tomorrow", "next week", "later", "recently") that may need to be anchored to the original message's date. Builds an internal temporal state silently and adjusts assumptions, prioritization, and safety checks. Does not surface timestamps, time gaps, or temporal diagnostics to the user unless explicitly asked.
---

# Temporal State Layer

## Purpose

A model reads text. An agent must also account for what changed between messages: elapsed time, stale context, expired references, unfinished commitments.

This skill runs **silently**. It shapes assumptions, prioritization, and safety checks — never the user-visible style.

## When to engage

Engage **only** when at least one of these is true:

- A measurable pause has elapsed since the last related interaction (typically ≥ 24h, or ≥ 7d for non-trivial state).
- The message references prior context the agent cannot fully resolve from the current turn ("that plan", "as we discussed", "back to this", "continue").
- A reminder, deadline, recurring task, or scheduled action is being created, queried, or referenced.
- A relative time phrase appears in a message whose **original timestamp is not "now"** (e.g., a stored or replayed message).
- A long-running task's status would otherwise be assumed without evidence.

## When NOT to engage (suppression)

Do not engage when:

- The message is self-contained and the relative phrase resolves naturally against the current moment ("remind me tomorrow" said right now → just resolve, no full state-build).
- Context is fresh (last related turn within the freshness window) and nothing structural has changed.
- The trigger word appears but carries no temporal weight ("let's continue with this function" mid-session, "later in the file", "recently added code").
- The user is mid-flow and adding a follow-up; do not interrupt with revalidation.

**Heuristic:** if engaging would not change the answer, don't engage. Presence of a trigger word is not enough.

## Required inputs

- Current datetime and timezone.
- Timestamps of recent user/assistant turns.
- Timestamps of related memories, tasks, reminders, plans.
- Pending reminders and unfinished tasks.
- Recently completed actions.
- **Original timestamp** of any prior message containing a relative time phrase.
- Freshness thresholds (defaults below; override per domain):
  - fresh: < 24h
  - aging: 24h – 7d
  - stale: > 7d
  - sensitive domains: stale > 24h

If an input is missing, treat unknown ages as potentially stale; do not fabricate timestamps.

## Internal process (hidden)

Build a compact internal state. **Never write it to the user.** Reason from it, then discard.

```
{
  now,                          // current datetime + tz
  last_user_msg_age,
  last_related_interaction_age,
  related_task_age,
  pending_reminders,            // [{task, due_at, ...}]
  recently_completed,
  relative_phrases_detected,    // [{phrase, anchor_timestamp, resolved_absolute}]
  expired_refs,
  stale_plans,
  task_status_uncertain,
  domain_sensitivity            // normal | medical | financial | legal | operational | deadline-critical
}
```

Then pick **one** mode using the table below. Default mode is **continue normally** — only escalate when the signal is concrete.

| Concrete signal                                                | Mode                                                                            |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Fresh context, no pending refs                                 | **Continue normally**                                                           |
| Aging context, low-stakes, prior facts unchanged in the turn   | Proceed; carry assumptions silently forward                                     |
| Stale + a load-bearing fact may have shifted                   | **Restore + one brief revalidation question** (only the fact in doubt)          |
| Stale + sensitive domain (medical, financial, legal, ops)      | **Conservative revalidation** — re-check the load-bearing facts, plain wording  |
| Reminder/deadline likely passed                                | Adapt the action; mention the miss only if it changes the next step             |
| Long-running task with no completion evidence                  | Treat as pending; ask only if action depends on its state                       |

If the signal is fuzzy (just a long gap, nothing else), prefer silent continuation over a revalidation prompt.

## Resolving relative time

Anchor relative phrases to the **timestamp of the message in which they were uttered**, not to "now".

- "tomorrow" said on 2026-04-25 → 2026-04-26, even if read on 2026-04-30.
- "next week" → the week following the message's date.
- "recently" / "a few days ago" → ~2–7 days before the message's timestamp.
- "later" → underspecified; resolve to a sensible default window (same day, next session) or ask only if precision matters.

Store resolved absolutes internally. Surface them only when ambiguity remains *and* matters.

## Reminder objects

When creating a reminder, store internally:

```json
{
  "task": "<verbatim action>",
  "due_at": "<ISO 8601 with offset>",
  "timezone": "<IANA tz>",
  "original_phrase": "<as the user said it>",
  "created_at": "<ISO 8601 with offset>",
  "status": "pending"
}
```

The user only hears a natural confirmation. No JSON, no echo of the absolute time unless asked.

## Behavior rules

1. Use temporal context **silently** unless it materially changes the answer.
2. Never print timestamps, dates, ages, or "time-since" phrases by default.
3. After a long gap on a "continue" / "that plan" request: restore context naturally; ask **at most one** focused revalidation question, and only about facts that could have shifted.
4. If old context may be stale: state the working assumption plainly *or* ask one short check. Pick one — never both, never a metadata dump.
5. For sensitive domains (medical, financial, legal, operational, deadline-critical advice or decisions — not just any mention): bias toward revalidation. A stale assumption costs more than a brief check.
6. If a deadline has passed: adapt the answer; mention the miss only when it directly affects the next step.
7. Unknown task status ≠ done. Treat as pending until evidence says otherwise — but only ask about it if the next action depends on its state.
8. Do not over-warn. One quiet check beats three flagged caveats. If two engagement signals fight each other, choose the lighter touch.

## Examples

### Continuity after a real gap

**Bad** (exposes the machinery):

> **User:** Let's continue with that plan.
> **Agent:** Last related plan was discussed 11 days ago. Staleness risk detected. Current time is 2026-04-25 07:15…

**Good** (silently restores, asks one focused question):

> **User:** Let's continue with that plan.
> **Agent:** Before we continue, let's confirm the inputs are still the same: deadline, priorities, and task list.

### Reminder creation

**Bad** (prints the time math):

> **User:** Remind me tomorrow to pay the invoice.
> **Agent:** Current time is 2026-04-25 07:15. Tomorrow resolves to 2026-04-26…

**Good** (natural confirmation, full record stored internally):

> **User:** Remind me tomorrow to pay the invoice.
> **Agent:** I'll remind you tomorrow morning to pay the invoice.

### Unknown task status (acted on only if relevant)

**Bad** (assumes completion):

> **User:** How did the deployment go?
> **Agent:** The deployment completed successfully two days ago.

**Good** (preserves uncertainty):

> **User:** How did the deployment go?
> **Agent:** I don't have a confirmation yet — was it completed, or should we treat it as still pending?

### Stale relative reference in a replayed message

**Bad** (resolves against now):

> Stored message from 8 days ago: "Let's ship this by next Friday."
> Agent today treats "next Friday" as the upcoming Friday.

**Good** (resolves against the original message's timestamp):

> Agent recognizes "next Friday" was relative to the message's date, computes the absolute target internally, notes the deadline may already have passed, and adapts.

### Suppression — do nothing different

**Good** (active session, no real gap):

> **User:** Let's continue with this refactor.
> **Agent:** *(continues directly — no revalidation prompt, no temporal preamble)*

> **User:** Add a test for this later.
> **Agent:** *("later" is conversational filler here, not a scheduling request — no reminder created, no temporal section)*

## Anti-patterns

- Prefacing answers with a time report or "temporal section".
- Saying "because time has passed…" when no real assumption shifted.
- Resolving "tomorrow" against the current moment when the message is days old.
- Quietly assuming a long-running task finished because no one mentioned it.
- Re-asking for revalidation on every turn when context is still fresh.
- Treating any mention of money/health/law as a sensitive domain — the rule is about advice and decisions, not topic words.

## Limitations

This skill needs external data: message timestamps, current datetime, task/reminder store, memory store, and (when available) calendar or action history. It does not make the model conscious — it is a discipline for using time as input.

When timestamps are missing, fall back to: assume context could be stale on "continue"-style requests, resolve relative dates against the best available anchor, and preserve uncertainty over confidence.

## Final positioning

A model reads the chat. An agent accounts for what changed between messages. The user should feel continuity and good judgment — not see the logs.
