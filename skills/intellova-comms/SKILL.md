---
name: intellova-comms
description: >
  How agents inside Intellova communicate. There is NO direct agent-to-agent
  message channel. Everything flows through Paperclip's data layer: issues,
  comments, approvals, work_requests, activity_log. Use when you need to ask
  another agent for anything, escalate, request approval, or coordinate.
---

# Intellova Communication Patterns

There is no Slack, no DM, no chat. **All agent-to-agent communication uses
Paperclip primitives.** Everything is async, durable, and audited.

## The five primitives

| Primitive | When to use |
|---|---|
| **Issue comment** (`POST /issues/:id/comments`) | Talking inside an open issue's thread. Most common. |
| **Approval request** (`POST /approvals`) | You need a Head / CTO / CFO / CEO sign-off before proceeding. |
| **Work request** (`POST /work_requests`) | You're an IC and need adjacent work done — see `intellova-scope-discipline`. |
| **Issue assignment change** (`PATCH /issues/:id`) | Hand off ownership. Always comment why before reassigning. |
| **Issue create** (`POST /issues`) | Heads/CTO/COO only, under an approved parent. ICs cannot. |

## How to ask someone for help

You're working on issue X. You need information from another agent.

1. **Don't DM.** Don't add to your packet without permission.
2. Comment on issue X: "@<agent-role-name> — I need <specific thing> to satisfy success_test #N."
3. If they don't reply in their next heartbeat, escalate to your Head via comment.
4. If you need them to do *work* (not just answer), file a `work_request` via your Head — your Head decides whether to assign that agent.

## How to escalate

Three escalation tiers:

| Severity | Channel | Example |
|---|---|---|
| Blocker (you can't proceed) | Comment + assign issue to your Head | "Blocked: DB schema change needed" |
| Disagreement (1 round failed) | Comment + assign issue to CTO | "Backend and Data disagree on storage approach, ADR needed" |
| Risk / out-of-band | `POST /approvals` with type=`escalation` | Spend overrun, security smell, scope creep request |

Never escalate by creating a new issue (you can't anyway as an IC). The
escalation must live on the *original* issue's thread for audit.

## How to request human (USER) input

Three things bubble to the human:

1. **Approvals on `three_gate` risk_class issues** — auto-routed to USER.
2. **Routine items in the Daily Brief** — COO routine aggregates these.
3. **Direct ask** — comment with "@user" tag in an issue assigned to CTO/COO/CEO. Their next heartbeat surfaces it via Daily Brief.

Do **not** spam the user with non-blocking questions. Batch them into a single
clear ask, post it in your issue, and let the digest pick it up.

## What every comment should look like

Every comment that records a decision or an action follows this shape:

```
[INTENT]
What you are about to do, in one sentence.

[CONTEXT]
Why, in two sentences max. Reference specific success_test or packet line if relevant.

[ACTION]
The actual change / call / decision (this part may be the API call result).

[RESULT]
What happened. Pass / fail / blocked / partial.
```

Skip sections that are obvious. Be terse. The activity_log captures the rest.

## What gets activity-logged automatically

You don't need to manually log:

- API calls you make (logged by Paperclip middleware).
- File reads/writes (logged by the workspace runtime).
- Tool invocations (logged by your adapter).
- Heartbeat start/end + exit code (logged by the dispatcher).

You **do** need to comment when:

- You make a non-obvious choice (which approach, which file, which trade-off).
- You hit something unexpected (error, scope question, blocker).
- You finish a success_test (mark it done in the comment).
- You hand off work.

## Naming and tagging

- Reference agents by **role**, not name: `@head-of-engineering`, not `@aisha-the-engineer`. Roles persist; names don't.
- Reference issues by `identifier` (e.g., `INTL-42`), not internal UUID.
- Reference packets by `id` (UUID is fine; they're immutable).

## What this skill does NOT cover

- The actual content of an issue/comment (your persona + role pack governs that).
- Triage rules (see `intellova-issue-validation`).
- Approval rules (see `intellova-escalation`).
- Goals and scope_contracts (see `intellova-scope-discipline`).

## TL;DR

> Talk via comments on issues. Ask for help via @role. Need adjacent work →
> work_request. Need a decision → approval. Blocked → escalate to Head. No DMs.
