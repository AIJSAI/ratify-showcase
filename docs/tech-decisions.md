# Technical Decisions: Ratify

This document contains excerpts from the project's decision log.

---

## D-006: Split Architecture (FastAPI + Next.js)

**Status**: Accepted
**Context**: A Vercel-only Next.js monolith was the default modern starting point, but several constraints made it the wrong fit for a compliance platform. Vercel functions have an 800-second duration ceiling, which is insufficient for batch jobs that can run for many minutes. Vercel has no persistent worker model for background work and webhook queues. The Vercel filesystem is read-only, which complicates report generation. There is no built-in API gateway for rate limiting, auth middleware, or API versioning at the level a production API needs.

**Decision**: Split the system: FastAPI (Python) backend in a container plus Next.js frontend on Vercel.

- **FastAPI** carries the compliance engine and background workers. Standard Docker containers, portable across hosts.
- **Vercel / Next.js** carries the dashboard surface and edge-cached server components.
- **Communication** is HTTPS REST with idempotency keys and workspace-scoped auth tokens.

**Consequences**:
- Two deployment targets to operate instead of one, accepted in exchange for long-running workers, persistent queues and Python's AI libraries.
- Python's full set of AI libraries (the Anthropic SDK, Pydantic, FastAPI dependencies) available on the backend without shoehorning into Node.js.
- Frontend stays on Vercel for what Vercel is good at (fast SSR, edge caching).
- Easy to relocate either side later without rewriting the other.

---

## D-007: PostgreSQL on Supabase

**Status**: Accepted
**Context**: A compliance product needs row-level separation between workspaces and point-in-time audit reconstruction. Building these from scratch on a custom-managed database would have absorbed months of foundational work before any product value.

**Decision**: PostgreSQL on Supabase as the system of record.

- Row-Level Security policies enforce workspace isolation at the database level (not just at the application layer).
- Built-in primitives (Auth, `pg_cron`) cover several adjacent needs.

**Consequences**:
- Vendor concentration on Supabase, mitigated because the underlying database is standard PostgreSQL, portable to RDS, Cloud SQL, or self-hosted; Supabase Auth and the row-level security policies keyed on its token claims are the parts that would need rework.
- Workspace isolation lives in policies keyed on the workspace claim in each user's token, and a CI check confirms that row-level security stays enabled on every workspace table.

---

## D-009: AI Never in the Critical Path

**Status**: Accepted
**Context**: Compliance decisions must be deterministic, reproducible and auditable, and fast enough to gate an order in real time (the target is under 100 ms). LLM latency (1-5 seconds typical) is incompatible with real-time order gating. LLM hallucination risk is incompatible with tax calculations. An auditor must be able to reproduce the answer: the same order under the same rules must produce the same answer on every check.

**Decision**: Compliance checks and tax calculations are deterministic rules engine operations. AI sits *next to* the critical path, never inside it.

- **Real-Time Order Compliance Gate**: pure rules engine. Returns allowed or denied with a structured reason code.
- **Tax Calculation**: pure rules engine.
- **AI workloads**: natural-language compliance questions and other unstructured work. None of these block an order or a filing.

**Consequences**:
- An AI provider outage degrades assistive features (the NL assistant goes down) but never breaks the core product.
- Compliance answers are reproducible: the same inputs always produce the same outputs.
- Every compliance decision has a deterministic audit trail with a citation to the underlying rule, not a model output.
- AI takes the unstructured work, while bounded rule evaluation stays deterministic.
