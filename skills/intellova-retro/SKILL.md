---
name: intellova-retro
description: >
  Friday weekly retro. Auto-drafted by COO from activity_log + heartbeat_runs.
  Reality Checker presents shipped work; issues flagged for improvement become
  next-week's process backlog. Loaded by COO + Project Shepherd; viewed by USER.
---

# Weekly Retro — Friday cadence

Run automatically by COO at Fri 16:00 UTC via the `retro-draft` routine. The
Project Shepherd also reads this skill when participating.

## What goes in the retro

Five sections, each evidence-based — no opinions without data backing them:

### 1. Goal progress
- Each active Goal: % complete, on-track / at-risk / blocked
- Goals slipped: which, why, what's the new ETA
- Goals shipped: which, by whom, link to demo evidence

### 2. Throughput metrics
- Issues opened this week / closed this week / cycle-time p50, p95
- Time blocked-on-human vs blocked-on-other
- Approval SLA: time from request to decision (USER's, Heads')
- Concurrency utilisation: % of slots used at peak

### 3. Quality metrics
- PRs merged / total PRs reviewed
- Code review block rate (% PRs with `[blocker]`)
- QA pass rate (first attempt vs after rework)
- Reality Checker veto count
- Production incidents (number, severity, MTTR)
- Cost per shipped issue

### 4. Process notes
- Triage rejection reasons (top 3) — signals systemic issues
- Work requests rejected (top 3) — signals scope confusion
- Rogue-run watcher alerts — should always be 0
- Approvals timed out → escalated

### 5. Decisions for next week
- 1-3 process changes to try
- 1-3 risks to watch
- Any agent to pause / fire / hire
- Any skill to update
- Anything blocked on USER

## Comment shape (posted as a doc, not a comment)

The COO writes this to a `documents` row of kind `retro`, one per week,
indexed by `2026-W18` etc. Then drops a link in the daily digest.

```markdown
# Retro — Week 2026-W18

## TL;DR (USER-facing)
- Shipped: 3 issues across Customer Dashboard MVP (60% complete)
- Slipped: 1 issue (DB index optimisation, blocked on Head of Data approval Tue→Thu)
- Quality: 0 incidents, 1 Reality Checker veto (overridden after evidence added)
- Risk: approval SLA p95 was 26h — above the 24h target

## Goals
- ✅ Stabilise platform: 0 OOMs this week (was 2 last week)
- 🔄 Customer Dashboard MVP: 3/5 issues done
- ⏸ Doc the product: not started (deferred per USER)

## Throughput
- Opened 8, closed 6, in-flight 2
- Cycle time p50: 6h (active 4h, blocked 2h), p95: 28h (active 5h, blocked 23h)
- Approval SLA: p50 4h, p95 26h
- Concurrency utilisation: peak 3/4 slots

## Quality
- 6 PRs merged, 8 reviewed (2 sent back for rework)
- Block rate: 33% (2/6 had blockers, both auth-related — see process notes)
- QA pass first-attempt: 4/6
- Reality Checker vetoes: 1 (lacked screenshot evidence; overridden after evidence added)
- Incidents: 0
- Spend: $34, $5.67/issue avg

## Process notes
- Triage rejected 4 issues for "while I'm at it" creep — same Head twice
- Work requests rejected: 2 ("can I add caching?" — out of scope)
- Rogue-run alerts: 0 ✓
- Approvals timed out: 1 (CFO budget approval; CFO was paused)

## Decisions
- [ ] Tighten Head of Backend's persona on scope (assigned to CTO)
- [ ] Wake CFO on weekdays only — pause Sat/Sun (saves quota)
- [ ] Add a test that any new auth-touching PR auto-blocks (assigned to Head of Trust & Safety)

## Asks for USER
- Approval needed: amend Customer Dashboard MVP goal to include export-CSV (was excluded; now in demand)
- Decision needed: pause Brand Guardian until marketing site is funded?
```

## How retro is generated

The `retro-draft` routine runs every Friday at 16:00 UTC:

1. Queries `activity_log`, `heartbeat_runs`, `cost_events`, `approvals` for the week
2. Aggregates the metrics above
3. Writes draft to `documents` table with kind=`retro`, status=`draft`
4. Assigns to COO for review
5. COO heartbeat picks it up, edits/refines, sets status=`final`
6. COO posts link to USER via Daily Brief Monday

## What the COO MUST NOT do during retro

- Editorialise without data (every claim needs a metric or evidence link)
- Recommend hires (that's CTO + USER)
- Promise specific actions without owner assignments
- Hide bad numbers — show them, propose the fix
- Skip weeks (the routine runs every Friday; skip a week and it accumulates)

## TL;DR

> Auto-drafted Friday. 5 sections. Evidence required. COO finalises. Dropped in Mon Daily Brief. No editorialising without data.
