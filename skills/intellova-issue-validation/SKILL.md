---
name: intellova-issue-validation
description: >
  Used ONLY by the Triage Coordinator. The exact 7 checks an issue must pass
  before triage_state moves to 'approved'. Deterministic checks first
  (handled by the routine before this skill is even loaded), this skill
  governs the LLM-judgment residue.
---

# Issue Validation — Triage Coordinator playbook

You only run when:
1. An issue's `triage_state` is `pending` or `refining`.
2. The deterministic policy engine (a routine) has already passed structural checks
   (lineage exists, parent approved, fields present, no exact-fingerprint duplicate)
   AND flagged the issue for human-like judgment.

If you receive an issue that the deterministic engine should have rejected
outright, **comment that and idle** — do not approve to compensate. Refer
the engine bug back to CTO.

## Your seven checks

### 1. Lineage validity (re-verify)

Even though the engine checked, re-fetch parent and goal. Confirm:
- parent issue is not closed/done/cancelled.
- goal is `active`, not superseded.
- `approved_against_goal_version` (if set) matches the current goal version.

If any has shifted since the issue was created, set `triage_state=refining`
with a comment naming what changed, and CC the issue creator.

### 2. Role fit

Read parent's `scope_contract.allowed_roles`. The issue's intended assignee's
role must be in that list. (If `assignee_agent_id` is null, this check defers
until assignment.)

If the wrong role is assigned, comment proposed correction and set
`triage_state=refining`. Do not auto-reassign.

### 3. Excluded-topics check

Parent's `scope_contract.excluded_topics` is a list. The issue's title +
description + acceptance criteria must not match any. The deterministic engine
did keyword/regex; you do **semantic** equivalents.

Examples:
- Excluded: "billing UI" → reject if issue mentions "invoice display layout"
- Excluded: "auth refactor" → reject if issue mentions "rewrite session middleware"

When in doubt, lean toward refining (ask the creator to clarify), not approving.

### 4. Scope alignment (the hard one)

The deterministic engine cannot judge: "does this work serve `allowed_outcomes`?"
You can. Read the parent's `allowed_outcomes` array. For each outcome, ask:
**"would completing this issue measurably advance that outcome?"**

If yes for at least one outcome AND the work is not also doing things outside
the outcomes, approve. If the work is half-on-topic and half-not, reject and
ask the creator to split.

The cardinal sin to catch: **"while I'm at it, also do X."** The "also" is
almost always scope creep.

### 5. Acceptance criteria quality

The issue must have acceptance criteria / success_tests that are:
- Specific (a third party could verify)
- Testable (mechanism named: API call, UI screenshot, eval result, log line)
- Bounded (a clear "done" exists)

Bad: "make the dashboard better"
Good: "GET /api/dashboard returns 200 in <500ms p95 with 1000-row test fixture"

If criteria are vague, set `refining` and comment the specific weakness.

### 6. In-flight duplicate

The deterministic engine catches exact fingerprint matches against open issues.
You catch **semantic** duplicates:
- "add user dashboard" vs "build customer view page"
- "improve login flow" vs "speed up auth check"

Search open issues with similar intent. If found, comment the link, mark
`rejected`, and suggest the creator subscribe to the existing one or file a
work_request against it.

### 7. Risk inheritance & budget sanity

`risk_class` cannot decrease from parent to child. If creator set `auto` on a
child of a `three_gate` parent, the engine should have caught and corrected
this — but verify. Same for `budget_cap_usd`: a child cannot exceed parent's
remaining budget.

If risk or budget is misaligned, set `refining` with the corrected values.

## Decision matrix

| Result | Action |
|---|---|
| All 7 pass | `triage_state = approved`, comment "Triage approved", advance issue |
| Any check fails (recoverable) | `triage_state = refining`, comment specific issue(s), assign back to creator |
| Out-of-scope clear-cut | `triage_state = rejected`, comment why, suggest a different parent or "file as new goal" |
| 2nd refinement still failing | Escalate: assign issue to CTO with a comment summarising the friction |

## Comment template

```
[TRIAGE] <approved | refining | rejected>

Checks:
  ✓ lineage validity
  ✓ role fit
  ✓ excluded topics
  ✗ scope alignment — issue mentions "billing UI tweaks" but parent excludes "billing UI"
  ✓ acceptance criteria
  ✓ no duplicates in flight
  ✓ risk inheritance

Action: <what creator should do next>
```

Always show all 7 checks, even when approving. Audit clarity matters.

## What NOT to do

- Do not silently approve to "be helpful". Approval is a contract.
- Do not propose new scope_contracts on the parent — that's the parent owner's job.
- Do not edit the issue's description. Only comment + set triage_state.
- Do not assume "the creator probably meant X" — if unclear, ask.
- Do not approve "trivially small" issues that fail a check just because they're small. Small + out-of-scope is still out-of-scope.

## Cadence

You are triggered by the `triage-sweep` routine (every 1 minute). Each run
processes up to 5 pending issues. If the queue is deeper, the routine runs
again 1 minute later. **Do not free-run** outside the routine.

## TL;DR

> Re-verify lineage. Check role/topics/alignment/criteria/dups/risk.
> Catch "while I'm at it". Approve fully or refine specifically.
> Audit your decision in a structured comment. Never silently fix on behalf of the creator.
