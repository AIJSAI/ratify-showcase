# Ratify: Alcohol Shipping Compliance Engine

> A personal project: a compliance engine I built that tells alcohol producers where they can legally ship across all 51 US jurisdictions, and cites the statute or refuses to answer.

---

**This repository holds engineering notes for Ratify. The implementation is private.**

[Portfolio case study](https://jamesshehan.dev/projects/ratify)

Other showcases: [Connect](https://github.com/AIJSAI/connect-voice-ai-showcase) · [Vinny](https://github.com/AIJSAI/vinny-showcase) · [Hive](https://github.com/AIJSAI/hive-showcase) · [On the Lees](https://github.com/AIJSAI/on-the-lees-showcase) · [Backyard](https://github.com/AIJSAI/backyard-showcase) · [The Chief](https://github.com/AIJSAI/operating-partner-showcase)

---

## Problem

Selling beverage alcohol in the US means complying with federal rules and the rules of 50 states plus DC:

- **50 states + DC**, each with different licenses, tax rates, volume limits, product restrictions, filing frequencies, and dry communities
- **Two distribution channels** (direct-to-consumer shipping, and three-tier wholesale through distributors and retailers), each with its own rule set
- **Constant regulatory change** across 51 jurisdictions
- **Scattered sources**: the rules live across 50+ government sites, so producers often fall back to spreadsheets and manual research

The same rules reach wineries, breweries, cideries and distilleries.

## Architecture

Ratify is a **split-backend application**: a long-running FastAPI service carries the compliance engine and background workers; a Next.js dashboard on Vercel handles the user-facing surface; Supabase Postgres holds each workspace's data, with Row-Level Security walling off each workspace. Compliance rulings come from the deterministic rules engine; Claude answers plain-English questions and cites the rule, or refuses.

| Component | Function |
|-----------|----------|
| **FastAPI service** | Long-running compliance jobs, no serverless duration limits, Python's full set of AI libraries |
| **Next.js 16 on Vercel** | Server Components, edge rendering, fast dashboard surface |
| **Rules Engine** | Deterministic compliance checks and tax calculations, no LLM in the critical path |
| **Spend caps** | Daily per-workspace and global caps on AI assistant spend, checked before each paid call |
| **Supabase Postgres + RLS** | Each workspace's data kept separate, audit trail |

## Tech Stack

| Technology | Role | Why This Choice |
|-----------|------|-----------------|
| FastAPI | Backend API | 800s+ batch jobs, persistent workers, no Vercel duration ceiling, portable Docker |
| Next.js 16 on Vercel | Frontend | App Router, Server Components, edge caching for the dashboard |
| Supabase (Postgres) | Database | Row-Level Security keeps each workspace's data separate, `pg_cron` for scheduled database jobs, audit trail |
| Anthropic Claude | LLM tier | Plain-English questions and other unstructured work, next to the rules engine and never inside it |
| Sentry | Observability | PII-scrubbed exception tracking, integrated with FastAPI middleware |
| GitHub Actions | CI/CD | 30+ workflows including CI, nightly smoke tests and security scans |

## Technical Challenges & Solutions

### 1. Compliance Must Be Deterministic, But Regulatory Text Is Unstructured

**Challenge**: Compliance checks must be reliable, fast (<100ms), and auditable. LLM latency (1-5 seconds) is too slow for real-time order gating, and hallucination risk rules it out for tax calculations. But the underlying regulatory source material is unstructured text scattered across 50+ government sites.

**Solution**: Strict separation of concerns (D-009). The deterministic rules engine handles every compliance check and tax calculation in-line: order gating, tax math, license expiration. AI sits *next to* the critical path, not inside it. The model handles the unstructured work, such as natural-language compliance questions. An AI provider outage degrades the assistive features but never breaks the core product.

### 2. Keeping Each Workspace's Data Separate

**Challenge**: Each workspace's data must stay separate, in the database and in the AI path. A filter in application code can be missed on one query, and a shared prompt cache could carry one workspace's context into another workspace's answer.

**Solution**: Row-Level Security at the database layer (D-007). Every workspace-scoped table carries a `tenant_id` column with a policy that compares it with the workspace claim in the signed-in user's token, and a CI check confirms that row-level security stays enabled on every workspace table. In the assistant, the workspace id sits at the start of the system prompt, ahead of the cache breakpoint, so a cached prompt prefix is never shared across workspaces.

## Key Decisions

Each decision is numbered in the project's decision log (D-###); excerpts are in [docs/tech-decisions.md](docs/tech-decisions.md).

| Record | Decision | Rationale |
|---------|--------|-----------|
| D-006 | Split Architecture (FastAPI + Next.js, not monolith) | Vercel function duration limits and read-only filesystem are incompatible with multi-minute batch jobs and report generation |
| D-007 | PostgreSQL on Supabase | Row-Level Security keeps each workspace's data separate in the database; standard Postgres stays portable |
| D-009 | AI never in the critical path | Compliance and tax calculations are deterministic rules; AI handles plain-English questions and other unstructured work |

## Results

- **37 of 37** sampled high-confidence answers verified against independently located primary sources, zero fabricated citations
- **788+ test files, 130+ database migrations, 30+ CI workflows**
- **Daily per-workspace and global spending caps on the AI assistant**, checked before each paid call, with Sentry PII-scrubbed observability

---

**Built by [James Shehan](https://jamesshehan.dev)**
