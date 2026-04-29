---
name: intellova-execution-packet
description: >
  How Heads create the Execution Packet that workers actually run from. The
  packet is an immutable per-issue context document that REPLACES "load all
  the org lore" — workers see only the packet plus three global skills.
  Loaded by all Heads (and read-only by ICs to understand what they're getting).
---

# Execution Packet — the immutable per-issue context

When an issue moves from `triage_state=approved` to ready-to-execute, the
issue's reporting Head produces an **Execution Packet**. It is stored as a
`document_revisions` row linked to the issue (immutable once created; new
revisions are new documents).

The worker agent's run loads:
1. `intellova-scope-discipline` (global)
2. `intellova-comms` (global)
3. `intellova-escalation` (global)
4. **The Execution Packet** (per-issue)
5. The worker's role pack (per-role, e.g., `intellova-role-backend-engineer`)

That's it. **No company-wide skill preload.** No design system loaded into
the DBA's context. No brand guidelines loaded into the security engineer's
context. The packet is the bounded universe.

## Packet fields

```jsonc
{
  "issue_id":         "<UUID>",
  "issue_identifier": "INTL-42",
  "goal_id":          "<UUID — the approved goal at the top of lineage>",
  "goal_version":     1,                 // for race-condition lineage check
  "parent_chain":     ["INTL-7","INTL-23","INTL-42"], // for context
  "title":            "<terse>",
  "goal":             "<one sentence: what success looks like>",

  "allowed_actions": {
    "files":   ["server/src/api/dashboard/**", "server/src/db/queries/dashboard.ts"],
    "endpoints": ["GET /api/dashboard", "POST /api/dashboard/refresh"],
    "tools":   ["Read","Write","Edit","Bash:test","Bash:lint","Playwright"],
    "external": []                       // outbound API domains the worker may hit
  },

  "forbidden_actions": {
    "files":   ["server/src/auth/**", "server/src/billing/**", ".github/workflows/**"],
    "tools":   ["Bash:rm","Bash:git push","Bash:aws"],
    "external": ["any-non-listed-domain"]
  },

  "budget_cap_usd": 5,
  "max_runtime_minutes": 30,

  "dependencies": [
    { "blocks_until": "INTL-41", "kind": "issue_done" }
  ],

  "success_tests": [
    "GET /api/dashboard returns 200 in <500ms p95",
    "Test fixture with 1000 rows loads correctly",
    "Lint + type-check pass",
    "API contract test in tests/api/dashboard.spec.ts passes"
  ],

  "escalation_triggers": [
    "any test in success_tests fails after 2 retries → comment + assign to head",
    "spend approaches budget_cap_usd × 0.8 → STOP, comment, idle",
    "any forbidden_action attempted (caught at workspace runtime) → STOP, escalate"
  ],

  "risk_class": "one_gate",
  "approver_role_for_ship": "head_of_platform_engineering",

  "context_links": [
    { "kind": "doc",   "id": "<UUID>", "label": "intellova-api-contracts excerpt" },
    { "kind": "issue", "id": "INTL-23", "label": "parent feature spec" }
  ],

  "created_by": "<head agent id>",
  "created_at": "2026-04-29T14:00:00Z",
  "immutable": true
}
```

## Authoring guidance for Heads

**`goal`**: one sentence, in the form "<system> does <X> when <Y>". If you
need two sentences, the issue is too big — split it before generating a packet.

**`allowed_actions.files`**: be specific. Globs are OK but lean restrictive.
"`server/src/**`" is too broad; the worker will treat the whole server as fair
game.

**`forbidden_actions.files`**: always include `auth/`, `secrets/`,
`billing/`, `iam/`, `.github/workflows/` unless the issue is *explicitly* about
those (in which case the issue is `three_gate` and got USER signoff).

**`success_tests`**: must be machine-checkable. Three is usually right. Five
means the issue is doing too much.

**`escalation_triggers`**: include the budget-80% rule by default. Add issue-
specific ones (e.g., "if the migration locks the table > 30s, abort").

**`max_runtime_minutes`**: set conservatively. If the worker hits this and
hasn't satisfied success_tests, it pauses and escalates. Default: 30 min.

## Anti-patterns to refuse

- **The "kitchen sink" packet**: 17 success_tests, broad file globs, no
  forbidden list. Refuse to generate.
- **The "trust the agent" packet**: vague goal, no concrete tests. Refuse.
- **The "borrow scope from parent"**: copying parent's allowed_actions
  wholesale. Each child packet must have *narrower* scope than its parent.
- **The "budget is whatever"**: never leave `budget_cap_usd` unset.
- **The "I'll add tests later" packet**: no success_tests, no packet.

## How a worker uses the packet

When the worker's heartbeat starts:

1. Load packet from `document_revisions`.
2. Verify `goal_version` still matches current goal (if not, idle and
   comment — goal was amended; wait for re-triage).
3. Walk `dependencies`; if any blocker isn't done, idle.
4. Set workspace constraints to `allowed_actions.files` (workspace runtime
   enforces this — write outside the glob fails).
5. Track spend; trip on `escalation_triggers.budget`.
6. Run work; verify each `success_tests` item; comment as it passes.
7. On all-pass: comment final report + advance issue status.
8. On any escalation_trigger: STOP, comment, set issue to blocked, assign to
   the issue's reporting head.

## How packet revisions work

Packets are immutable. If a Head needs to change scope mid-flight (e.g., a new
forbidden_action discovered):

1. Worker pauses (Head closes original issue or marks blocked).
2. Head creates new issue (or sub-issue) with new packet.
3. New packet has new ID, new immutable contents.
4. Original packet stays for audit.

Never edit a packet. The whole point is replay-ability.

## Storing the packet in Paperclip

```sql
INSERT INTO documents (id, company_id, kind, title, ...)
  VALUES (gen_random_uuid(), <company>, 'execution_packet', '<issue identifier>', ...);

INSERT INTO document_revisions (document_id, version, body_json, ...)
  VALUES (<doc_id>, 1, <packet JSON>, ...);

UPDATE issues SET execution_state = jsonb_set(
  execution_state,
  '{packet_document_id}',
  '"<doc_id>"'
) WHERE id = <issue_id>;
```

The worker reads `issues.execution_state.packet_document_id` and fetches
the latest revision.

## TL;DR

> Heads write packets when issues are approved. Workers run from the packet only.
> Specific allowed/forbidden globs. Concrete success_tests. Hard budget. Immutable.
> No editing. Mid-flight scope change = new issue + new packet.
