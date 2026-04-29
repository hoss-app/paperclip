---
name: intellova-qa
description: >
  QA gate. Run by Head of Trust & Safety / Eval Engineer / Reality Checker
  before a piece of work ships. Includes API contract tests, E2E (Playwright),
  performance, accessibility, and the LLM-eval suite. Triggered after code review
  passes, before /ship.
---

# QA Gate

Run after code review passes, before ship. Verifies the work meets its
`success_tests` empirically, not by assertion.

## The 5 QA modes

Each PR/issue triggers a subset based on what changed:

| Mode | When | Owner | Tool |
|---|---|---|---|
| `api` | API surface changed | API Tester (codex_local) | Vitest + supertest |
| `e2e` | UI changed | Evidence Collector (claude_local) | Playwright MCP |
| `perf` | Hot path or migration | Performance Engineer (codex_local) | k6 / autocannon |
| `a11y` | UI changed | Accessibility Auditor (claude_local) | axe-core |
| `eval` | Agent prompt or LLM behaviour changed | Eval Engineer (claude_local) | Eval framework |

The Code Reviewer marks which modes apply on the PR. QA dispatcher fires only
those.

## Per-mode minimum bar

### `api`
- Every changed/added endpoint has a contract test asserting request/response shape
- Tests cover happy path + 1+ failure cases
- All tests green
- No new endpoint without an entry in `intellova-api-contracts`

### `e2e`
- For each changed user-facing flow, a Playwright test that:
  1. Logs in (or stubs auth as appropriate)
  2. Navigates to the affected screen
  3. Performs the user action
  4. Asserts the expected DOM/URL state
- Screenshots captured at start, action, and end (Evidence Collector)
- Tests pass in headless Chrome at 1280×720

### `perf`
- p50, p95, p99 measured against a 1000-row test fixture
- p95 < target (target stated in the issue's `success_tests`)
- No regression > 10% from baseline (last shipped)
- Memory: no leak over 60s sustained load

### `a11y`
- axe-core run on changed pages: 0 critical violations
- Tab navigation: every interactive element reachable
- Screen-reader labels: present on all form fields
- Colour contrast: WCAG AA on all text

### `eval`
- For prompt changes: regression suite of N test cases passes at ≥ baseline rate
- For new agent behaviour: at least 5 hand-written test cases, ≥ 4 pass
- For tool/MCP changes: tool-call success rate measured, no regression
- All evaluations stored as `document_revisions` with full prompts + outputs

## Reality Checker (final gate)

After mode-specific QA passes, the Reality Checker runs:

1. Re-reads the issue's `success_tests`
2. Asks: did each test ACTUALLY pass with EVIDENCE attached?
3. Checks: is the evidence (screenshots, test output, eval results) genuine or fabricated?
4. Default position: skeptical. "Looks like it works" is not enough.

The Reality Checker can REJECT a green QA run if evidence is weak. They have
veto power.

## Comment shape

```
[QA]  <pass | fail | partial>

modes run: e2e, a11y
e2e:   ✓ 4/4 tests pass — evidence: <screenshots URL>
a11y:  ✓ axe 0 critical, contrast ✓, tab nav ✓

reality-checker: ✓ evidence reviewed, success_tests #1, #2, #3 met

Verdict: pass. Cleared for /ship.
```

## What QA NEVER does

- Approves on assertion ("should work")
- Skips a mode that the Code Reviewer flagged
- Lets a Reality Checker veto be overridden by anyone but CTO
- Edits the issue's success_tests (those are immutable in the packet)
- Runs partial tests and passes — partial = fail

## What QA does NOT cover (out of scope)

- Code style → covered by `intellova-review`
- Security policy → covered by `intellova-security-baseline`
- Production deploy → covered by ship gate

## TL;DR

> Mode-based: api/e2e/perf/a11y/eval. Each has a minimum bar. Reality Checker
> has veto. Evidence required, not assertion. Partial = fail.
