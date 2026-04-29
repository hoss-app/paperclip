---
name: intellova-review
description: >
  Code review gate. Used by Code Reviewer agents before a PR can ship.
  Structured checklist that maps to intellova-conventions and intellova-security-baseline.
  Triggered on every PR by the Code Reviewer; called manually for ad-hoc review.
---

# Code Review Gate

Run on every PR before merge, regardless of risk_class. Code Reviewer agent
posts the review comment; ICs do not self-approve.

## Required checks

### 1. Conventions
- Branch name format: `feat/INTL-N-slug` or `fix/INTL-N-slug` etc.
- Commit messages: conventional + issue id reference
- Files only in packet's `allowed_actions.files` glob
- No new dependencies without prior work_request approval
- Code style: prettier/black clean, no `any` without justification

### 2. Tests
- New behaviour has a test
- Bugfix has a regression test
- All tests pass locally (CI confirms)
- Coverage didn't drop on changed paths
- E2E (if UI) or contract (if API) tests added/updated

### 3. Security
- No hard-coded secrets or PII
- No new outbound calls to undisclosed domains
- No `eval`, `exec`, dynamic SQL
- All user inputs validated
- Auth/secrets paths untouched (or three_gate approval present)

### 4. Observability
- Errors logged with context (issue id, run id)
- No `console.log` left in
- Metrics/traces added for new code paths
- User-facing errors are localized + actionable

### 5. Documentation
- README updated if behaviour changes
- ADR filed if architectural choice (in `docs/adr/`)
- Inline comment for non-obvious WHY only

### 6. Compatibility
- Migration is backward-compatible (or marked breaking with rollback plan)
- API change is additive (or versioned)
- No silent broken contracts with consumers

### 7. Scope
- The diff is what the issue + packet asked for, no more
- "While I was here" cleanup is its own PR or rejected

## Comment shape

```
[REVIEW]  <approve | request-changes | reject>

✓ conventions
✓ tests
⚠ security — line 42 logs `req.body` which may contain PII; use `redact()`
✓ observability
✓ documentation
✓ compatibility
✗ scope — files modified outside packet's allowed_actions.files:
   - server/src/utils/random-helper.ts (not in scope)

Action: redact PII in line 42; remove unrelated change to random-helper.ts; resubmit.
```

## Severity tags inside comments

- `[blocker]` — must fix to merge
- `[nit]` — style preference, fix at author's discretion
- `[question]` — clarification needed, may not block
- `[future]` — out of scope for this PR; suggest filing as separate issue

Authors must respond to every `[blocker]` and either resolve or push back with reasoning. Reviewers don't approve until blockers are resolved.

## What never blocks a review

- Personal style preferences without a project rule
- Optimisations not measurably needed
- Scope expansion suggestions ("you should also...")
- Naming bikeshedding when the existing name follows convention

## What always blocks a review

- Any of the 7 required checks failing
- Tests not run or failing
- Touching files outside packet
- New secrets in code
- Dropping tests "to ship faster"
- Auth/secrets/PII path changes without explicit three_gate approval

## TL;DR

> 7 checks. Structured comment. Severity-tagged. Never approve with a blocker open. Never block on style.
