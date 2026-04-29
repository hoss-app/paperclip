---
name: intellova-plan-review
description: >
  Used by Heads when reviewing a plan from a worker agent. The structured gate
  before the worker is allowed to execute. Pairs with the Execution Packet —
  this skill is what the Head runs to decide whether to authorise packet
  generation. Triggered by the agent on any /plan-eng-review-style ask.
---

# Plan Review (Head-side gate)

When a worker agent posts a plan on an issue and asks for approval, you (the
Head) run this skill before authorising packet generation.

## What you must check

For every plan you review, confirm ALL of:

### 1. Goal alignment
- Plan's stated goal matches the issue's `goal` field exactly.
- Plan does not silently expand to adjacent goals.

### 2. Scope contract conformance
- Every step in the plan touches files within the parent's `scope_contract.allowed_actions.files`.
- No step touches anything in `scope_contract.forbidden_actions`.
- No step adds dependencies to domains not in `allowed_actions.external`.

### 3. Decomposition quality
- Plan has 3-7 concrete steps. Fewer = under-thought; more = should be split.
- Each step has a clear "done" condition.
- Dependencies between steps are explicit.

### 4. Test coverage
- Plan names the success_tests it will satisfy (must reference packet's `success_tests` array by index).
- For new code paths: a unit test step exists.
- For UI changes: a Playwright E2E step exists.
- For schema changes: a migration verification step exists.

### 5. Risk classification
- Plan's `risk_class` is appropriate (auto / one_gate / three_gate).
- If touching auth/secrets/billing/PII/IAM/deploy → must be `three_gate` regardless.
- If risk is higher than the parent's class, escalate (computed live, not stored).

### 6. Budget & runtime
- Plan estimates < `packet.budget_cap_usd × 0.7` (leave headroom).
- Plan estimates < `packet.max_runtime_minutes × 0.7` per step.

### 7. Rollback / safety
- For three_gate work: plan names rollback steps.
- For one_gate work: plan names "what to do if a success_test fails".
- For auto: not required.

## Decision tree

```
All 7 pass:
  → Comment: "Plan approved. Proceeding to packet generation."
  → Generate Execution Packet (see intellova-execution-packet)
  → Move issue triage_state → approved (if not already)
  → Worker can begin

Any fails (recoverable):
  → Comment: list specific failures
  → Set issue to blocked-on-author with reassignment back to worker
  → Worker iterates, resubmits

Any fails (clear-cut out of scope):
  → Reject the issue (triage_state=rejected)
  → Comment why, suggest alternative parent or new-goal-needed
```

## Comment template

```
[PLAN REVIEW]  <approved | rework | rejected>

Checks:
  ✓ goal alignment
  ✓ scope contract conformance
  ✗ decomposition quality — step 4 ("misc cleanup") is not concrete
  ✓ test coverage
  ✓ risk classification (one_gate)
  ✓ budget & runtime
  ⚠ rollback — please add a step for "what if migration locks > 30s"

Action: rework step 4 and add migration timeout handling, then resubmit.
```

## Anti-patterns to flag

- **"And while we're at it"** — the cardinal sin. Reject.
- **"This is similar to issue X, so I'll just copy"** — no, write a fresh plan grounded in THIS packet.
- **"Tests can come later"** — no. Plan tests in the same PR.
- **"Quick fix"** — there is no such thing in three_gate.
- **"I'll figure it out as I go"** — that's the failure mode of the previous CTO. Reject.

## What this skill is NOT

- This is not the gstack `/plan-eng-review` (yet — that comes in a later session via image rebuild).
- This is the minimum-viable Intellova version. It captures the essential checks but lacks gstack's interactive prompts.

## TL;DR

> 7 checks, structured comment, approved/rework/rejected. No "while we're at it". No "tests later". No "quick fix" on three_gate.
