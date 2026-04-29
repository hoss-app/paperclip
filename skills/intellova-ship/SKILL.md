---
name: intellova-ship
description: >
  Ship gate. Final checklist before deploying or merging. Run by the Head
  authorising release (or USER for three_gate). Requires green from
  intellova-review and intellova-qa.
---

# Ship Gate

This is the LAST gate before the change is live. Run by:

- For `auto`: the Code Reviewer agent automatically (after review + qa green)
- For `one_gate`: the issue's reporting Head
- For `three_gate`: the USER (after Head ack)

## Prerequisites (block ship if any false)

- Code review: PASS
- QA: PASS for all flagged modes
- Reality Checker: not vetoed (if mode included)
- All `success_tests` from packet have evidence attached
- Spend tracker: under packet's `budget_cap_usd`
- All approvals required by `risk_class`: present
- Lineage: still reaches an active goal at current version

If ANY are false: don't ship. Comment what's missing. Wait.

## Ship checklist

### 1. Final confirm
- Issue `status` is `ready_to_ship` (not `done` yet — that's after ship)
- All blockers resolved
- No new comments since last review (if so, re-confirm)

### 2. Pre-deploy
- For code: PR is green, branch is up-to-date with main, no merge conflicts
- For infra: CFN changeset reviewed and applied to staging first (if applicable)
- For DB: migration tested on a copy of the database

### 3. Execute deploy
- Trigger via the appropriate path:
  - Code → merge PR → GH Actions auto-deploys (if configured) OR `workflow_dispatch`
  - DB migration → `intellova-db-migrate` Lambda
  - Cloudflare → CF API or dashboard
  - AWS resources → CFN deploy
- Capture the deploy artifact ID (PR sha, CFN stack id, run id)

### 4. Post-deploy verify
- Smoke test: the most basic "is it alive" check
- Health endpoint: 200
- For UI: a Playwright "load + click main thing" smoke
- For API: a contract test against production
- Cloudflared still active, tunnel healthy

### 5. Mark complete
- Update issue `status` → `done`
- Comment: "Shipped. Deploy ID: <id>. Smoke: PASS. Effective at <timestamp>."
- Activity log entry recorded
- Notify Analytics Reporter if customer-facing change (for the digest)

### 6. Watch for 1 hour
- Set a 1-hour follow-up routine
- If error rates spike, alerts fire, customer reports come in: STOP, file an incident issue, page Incident Response Commander

## Rollback (when post-deploy verify fails)

For three_gate, the plan included rollback steps. Run them.

For one_gate without explicit rollback:
- Revert the PR (if code)
- Roll back the CFN stack to prior version
- Roll back the migration (if reversible)
- Comment what failed, where, and what's the next step

If you cannot rollback safely, escalate IMMEDIATELY to Incident Response
Commander + USER.

## What ship does NOT do

- Re-run code review (already done)
- Re-run QA (already done — but smoke test is fresh)
- Decide if the change was a good idea (that was approval)
- Skip approvals because "it's a small change" (no — `auto` tier exists exactly for that)

## Comment shape

```
[SHIP]  <success | rollback | held>

prereqs: ✓ review ✓ qa ✓ reality-checker ✓ approvals ✓ lineage
deploy:  PR #142 merged to main, GH Actions run #25052403246
verify:  ✓ /api/health 200, ✓ smoke playwright pass
status:  done — effective 2026-04-29T16:42:00Z

watching for 1h, will alert on regression.
```

## TL;DR

> Final gate. Only ships if everything green. Smoke test post-deploy.
> Watch 1h. Have rollback ready. Three_gate = USER signs.
