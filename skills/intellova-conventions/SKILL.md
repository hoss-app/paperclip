---
name: intellova-conventions
description: >
  Code style, PR rules, branch naming, commit format, review etiquette,
  and "do not do this" patterns for Intellova. Loaded by every engineering
  agent. Pairs with intellova-tech-stack.
---

# Intellova Engineering Conventions

## Branches

- `main` / `master` — protected, never push directly. PRs only.
- Feature branches: `feat/<issue-id>-short-slug` (e.g., `feat/INTL-42-dashboard`)
- Bugfix: `fix/<issue-id>-short-slug`
- Chore (deps, docs, formatting): `chore/<short-slug>`
- Long-lived branches that aren't a PR: forbidden.

## Commits

- Imperative tense: "add X", not "added X" or "adding X".
- Conventional commit prefix: `feat(scope): ...`, `fix(scope): ...`, `chore(scope): ...`, `docs(scope): ...`, `test(scope): ...`.
- One change per commit. Squash before merge if you racked up exploration commits.
- Reference the issue ID at the end: "feat(api): add user dashboard endpoint (INTL-42)".

## PRs

- Title = the same as the issue title (don't restate).
- Body MUST include: linked issue, what changed, why, how it was tested, risks.
- Keep PRs small. Target < 400 LOC of net change. Above that, split.
- One PR per issue — don't bundle.
- Self-review your own diff before requesting review.

## Code review

- `claude_local` Code Reviewer agent runs `/review` on every PR before human eyes.
- Required checks: type check, lint, tests, the gstack `/review` skill.
- Blocking comment styles: `[blocker]` (must fix), `[nit]` (style), `[question]` (clarification), `[future]` (out of scope, file as separate issue).
- Reviewer never pushes commits to the PR; comments only. Author addresses, marks resolved.

## TypeScript

- Strict mode on. `noImplicitAny`, `strictNullChecks`.
- Don't `any` your way around type errors. If genuinely needed, comment why.
- `unknown` over `any` when the type is truly variable.
- Functional > class-based unless framework requires.
- Result types over throwing for expected errors. Throw only for unexpected.

## Python (Lambdas)

- 3.10+. Type hints on every function signature.
- `black` for formatting, `flake8` for lint, `mypy` for type-check.
- `structlog` for JSON-structured logging — never `print`.
- Dependencies pinned in `requirements.txt`. No transitive surprises.

## SQL

- Migrations: idempotent (use `IF NOT EXISTS` / `IF EXISTS`).
- Constraints: prefer CHECK over triggers when sufficient.
- Indexes: prove necessity with EXPLAIN before adding. Don't pre-optimize.
- Naming: `snake_case`. Plural table names. Singular column names.
- Foreign keys: always with explicit `ON DELETE` clause (`CASCADE`, `SET NULL`, or `NO ACTION` per intent — never default-implicit).

## Errors & logging

- No silent failures. Every error gets logged with structured context.
- No `console.log` in committed code. Use the project logger.
- Errors that the user might see: localized, actionable.
- Errors internal: include stack + correlation id (issue id, run id, packet id).

## Tests

- Vitest for TS unit, pytest for Python, Playwright for E2E.
- Coverage isn't a target on its own. Test the contract, not the implementation.
- New endpoint → API contract test (via Head of Quality's API Tester).
- New UI flow → Playwright E2E (via Evidence Collector).
- Every fix → a regression test.

## What NEVER goes in code

- Hard-coded secrets, API keys, passwords, customer IDs.
- IP addresses (use hostnames + DNS).
- Personal info / PII in test data.
- Customer-specific logic in shared modules — use config or strategy pattern.
- `// TODO` without an issue link. If it's worth a TODO, file the issue.
- Comments saying "this is a hack" — fix it or document properly.

## What NEVER ships without explicit approval

- Database migration that drops or renames columns.
- Anything in `auth/`, `secrets/`, `iam/`, `billing/` paths.
- Anything that adds an outbound network call to a domain not already in use.
- New Cloudflare records.
- New AWS resources (any service).
- Anything that touches the deploy workflow itself.

These trigger `three_gate` automatically — see `intellova-escalation`.

## Documentation

- README at the top of every package, updated when behaviour changes.
- ADRs (Architecture Decision Records) for non-trivial architectural choices.
  Live in `docs/adr/NNNN-title.md`. Use the agency-agents Software Architect ADR template.
- Skill files live in `skills/<slug>/SKILL.md` — same shape every time.
- Inline docs only when the WHY is non-obvious. The code already explains the WHAT.

## Style nits we enforce

- Unix line endings.
- Trailing newlines on files.
- No tabs. 2 spaces (TS), 4 spaces (Python).
- Lines under 100 chars.
- Use the project's prettier/black config. Don't override.

## TL;DR

> Branch per issue. Conventional commits with issue id. Small PRs. Strict types. No silent errors.
> Migrations idempotent. Anything touching auth/secrets/billing/iam/deploy = three_gate.
> No secrets in code. No TODO without an issue.
