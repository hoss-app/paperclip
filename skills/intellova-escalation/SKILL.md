---
name: intellova-escalation
description: >
  What gets escalated to whom, and what does NOT. Defines the risk-class system
  (auto / one-gate / three-gate), what each tier requires, and which actions
  always require human (USER) sign-off regardless of class. Loaded by every
  agent. Pairs with intellova-scope-discipline.
---

# Intellova Escalation Policy

Three risk classes determine the gates a piece of work must pass before
shipping. Every issue is tagged at creation; the class can only go UP, never
down (Codex's race-fix). At dispatch, the *effective* risk_class is computed
as `max(self, all ancestors)`.

## The three classes

### `auto` — ship if green

For changes the company has decided are safe to do without human review:

- Documentation updates
- Internal refactors that don't change public behaviour
- Adding tests, evals, or dashboards
- Non-prod infra changes ≤ $5
- Cleanup, dead-code removal, dependency bumps with passing CI

Required gates:
- `triage_state == approved`
- `/review` passes
- `/qa` passes
- spend ≤ packet's `budget_cap_usd`

**No human approval. No Head approval.** Ships automatically.

### `one_gate` — Head approves

Default for normal engineering work:

- Routine bugfixes within an approved goal
- Scoped feature implementation (Head already approved the plan)
- Schema changes approved by Head of Data + Head of Security
- Performance improvements
- New endpoint that doesn't expose new data

Required gates:
- everything in `auto`
- + `Head` (your reporting head) sign-off via approval row

### `three_gate` — full review chain

For irreversible / high-blast-radius changes:

- New customer-facing features
- Auth, secrets, billing, PII paths
- Production deploys
- Database migrations affecting customer data
- Any spend > $25 in single action
- New AWS resources (EC2, IAM roles, KMS keys)
- New Cloudflare records
- Any change to the deploy pipeline

Required gates:
- everything in `one_gate`
- + Head of Trust & Safety review (auto-flagged on PII/auth/secrets/billing)
- + spec approval (USER)
- + plan approval (USER)
- + demo sign-off (USER)

## The hard always-escalate list (regardless of class)

These ALWAYS go to USER, no matter what risk_class is set:

1. Hiring any agent.
2. Spending > $25 in a single action.
3. Any change touching: auth, secrets, PII, billing, IAM.
4. Any production deploy.
5. Two Heads disagreeing >1 round.
6. Any agent run failing 3 times in a row on the same issue.
7. Any rogue-run watcher alert.
8. Any goal scope_contract amendment.

## The never-escalate list

Don't bug the human about:

- Routine PRs that pass `/review` + `/qa` and are tagged `auto`.
- Within-team coordination.
- Schema migrations approved by Data + Security.
- Doc updates.
- Skill updates (auto-versioned; user reviews monthly).
- Cleanup work that doesn't change behaviour.

## The structured ask

When you need an approval, file an `approvals` row, NOT a comment:

```
{
  "issue_id": <issue id>,
  "type": "spec | plan | demo | spend | hire | escalation | other",
  "asker_agent_id": <self>,
  "approver_role": "head | cto | cfo | coo | ceo | user",
  "summary": "1 sentence",
  "context_link": "<Paperclip URL>",
  "options_if_relevant": ["option a", "option b"],
  "default_if_silence": "wait"  # NEVER default to "proceed" on three_gate
}
```

If the default is "wait", you idle until resolved or 24h timeout.

## Approval timeouts

- `one_gate` Head approval: 24h timeout. After that, escalate to CTO.
- `three_gate` USER approval: 72h timeout. After that, post a single nudge comment, then close issue as `blocked-on-user`. Do NOT auto-proceed.

## The "second-opinion" pattern (encouraged)

For three_gate work, the Head can voluntarily request a second-opinion from a
peer Head before sending up to CTO. This is good practice. It's not required
but reduces escalations.

## What COO + CFO see automatically

These are NOT escalations; they're streams the COO/CFO are subscribed to:

- COO: every issue assignment change, every blocker comment, every approval timeout.
- CFO: every cost_event, every spend > $5, every budget_incident.

They aggregate these into the Daily Brief. You don't need to copy them on
individual events.

## TL;DR

> auto = ship green. one_gate = Head signs. three_gate = USER signs spec, plan, demo.
> Always escalate hires, spend>$25, auth/secrets/PII/billing/deploys.
> Never escalate routine green PRs or within-team coordination.
