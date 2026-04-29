---
name: intellova-tech-stack
description: >
  The actual stack Intellova runs on: repos, languages, AWS services,
  third-party integrations, key URLs. Loaded by every engineering agent so
  recommendations align with what we already use. Update when stack changes.
---

# Intellova Tech Stack — current state

## Repositories

| Repo | Purpose | Owner |
|---|---|---|
| `hoss-app/paperclip` | Fork of paperclipai/paperclip — agent orchestrator we run | CTO |
| `hoss-app/in-aws` | CloudFormation IaC + Lambdas + AWS skills + cloudflare-dns skill | Head of Cloud Operations |
| `hoss-app/in-app` | Customer portal Next.js app (intellova-client-portal) | Head of Platform Engineering |
| `intellova-website` | Marketing site (static) | Head of Design |

## Languages / runtimes

- **TypeScript** everywhere (Node.js, Next.js)
- **Python 3.10+** for AWS Lambdas (in-aws repo)
- **SQL (Postgres)** for migrations
- Avoid: Go, Rust, Ruby, PHP. Don't introduce new languages without CTO + Head of Platform approval.

## AWS

- **Account**: `014420964816`
- **Region**: `ap-southeast-2` (Sydney)
- **CLI profile**: `intellova`
- Services in use:
  - **EC2**: paperclip control plane (`i-0ad22c54d771dd6dc`, t4g.medium), portal-prod, portal-dev
  - **RDS Postgres 16.12**: `intellova` instance (shared by intellova app + paperclip db)
  - **Bedrock AgentCore**: agent runtime workloads
  - **Lambda**: `intellova-db-migrate` (DB ops via API), inchoice-pipeline ETL
  - **Step Functions**: ETL orchestration
  - **S3**: `intellova-public` (logos, assets), `intellova-clients` (customer data)
  - **ECR**: `paperclip-server`, `intellova-client-portal`, `inchoice-etl`
  - **Secrets Manager**: namespace `intellova/...` (rds, paperclip, agentcore, cognito, ses, stripe)
  - **SES**: transactional email
  - **CloudWatch**: logs + metrics + budgets
  - **ElastiCache Redis**: portal session/throttling

## DNS / TLS

- **Cloudflare** manages `intellova.com.au`
- All public hostnames go through **Cloudflare Tunnels** (cloudflared on EC2)
- No public web ports on any EC2 — zero inbound rules
- Cert management is automatic via Cloudflare

## Auth

- Customer portal: AWS Cognito (separate prod / staging pools)
- Paperclip control plane: Better Auth (built-in, email + password)
- Agent-to-API: bearer tokens stored in `agent_api_keys` table, hashed at rest

## Payments

- Stripe (test mode currently). Secret in `intellova/stripe/test`.

## CI / CD

- GitHub Actions only. No CircleCI, no Jenkins.
- Workflow auth: IAM user `github-actions-paperclip` (this repo), `github-actions-in-app` (portal repo).
- Deploys: `workflow_dispatch` only on master/main. No auto-deploy.

## Conventions (briefly — see `intellova-conventions` for full)

- ARM64 builds for everything (t4g instances are Graviton).
- Docker images: `linux/arm64` only.
- TLS everywhere; `?sslmode=require` on Postgres connection strings.
- Secrets via Secrets Manager — NEVER in env files committed to git.
- Terraform: not used. CloudFormation only.

## What NOT to introduce

| Tech | Why not |
|---|---|
| Mobile (iOS/Android) | Not on roadmap |
| Kubernetes | Overkill for current scale |
| Multi-cloud | Single AWS account is fine |
| GraphQL | REST is sufficient |
| Ruby / Go / Rust / Java | Stack already TS + Python |
| Self-hosted DBs (other than RDS) | Use managed services |
| New ad-hoc S3 buckets | Use existing scoped buckets |

If you think a new dependency is needed, **file a work_request** with type `scope_question`. Don't `npm install` in your packet path.

## Repos already used by agents

Each Paperclip agent's working directory is `/app` inside the container. To
edit code in other repos, the agent goes through the GitHub flow:

- Read public repos via raw URL or gh API.
- For private edits, file a work_request → Head of Platform decides whether to clone + branch + PR.
- Direct git pushes from agents: forbidden. PRs only, via the `git-workflow-master` agent.

## TL;DR

> TypeScript on Next.js, Python Lambdas, Postgres, AWS in ap-southeast-2,
> Cloudflare Tunnels for ingress, GitHub Actions for CI. ARM64 everywhere.
> Don't add new languages or new clouds.
