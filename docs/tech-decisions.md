# Technical Decisions: Ratify

This document contains excerpts from the project's decision log.

---

## D-006: Split Architecture (FastAPI + Next.js)

**Status**: Accepted
**Context**: A Vercel-only Next.js monolith was the default modern starting point, but several constraints made it the wrong fit for a compliance platform. Vercel functions have an 800-second duration ceiling, which is insufficient for state tax filing batch jobs that can run for many minutes. Vercel has no persistent worker model for background regulatory monitoring and webhook queues. The Vercel filesystem is read-only, which complicates report generation. There is no built-in API gateway for rate limiting, auth middleware, or API versioning at the level enterprise customers expect.

**Decision**: Split the system: FastAPI (Python) backend on Railway plus Next.js frontend on Vercel.

- **Railway / FastAPI** carries the compliance engine, AI extraction pipeline, background workers, and integrations (Commerce7, ShipStation, FedEx). Standard Docker containers, no vendor lock-in.
- **Vercel / Next.js** carries the dashboard surface, marketing pages, and edge-cached server components.
- **Communication** is HTTPS REST with idempotency keys and tenant-scoped auth tokens.

**Consequences**:
- Two deployment targets to operate instead of one, an accepted trade-off for the flexibility gains.
- Full Python AI ecosystem (LiteLLM, Pydantic, FastAPI dependencies) available on the backend without shoehorning into Node.js.
- Frontend stays on Vercel for what Vercel is good at (fast SSR, edge caching).
- Easy to relocate either side later (Railway → AWS App Runner, Vercel → any static host) without rewriting the other.

---

## D-007: PostgreSQL on Supabase

**Status**: Accepted
**Context**: A compliance product needs row-level multi-tenancy, vector search for RAG over regulatory documents, point-in-time audit reconstruction, and a path to SOC 2 attestation. Building these from scratch on a custom-managed database would have absorbed months of foundational work before any product value.

**Decision**: PostgreSQL on Supabase as the system of record.

- Row-Level Security policies enforce tenant isolation at the database level (not just at the application layer).
- `pgvector` extension powers the regulatory RAG corpus with HNSW indexes, so no separate vector database to operate.
- Temporal tables support point-in-time audit reconstruction for filings and compliance gate decisions.
- SOC 2 attestation on Supabase Team plan removes a category of enterprise procurement blockers.
- Built-in primitives (Auth, real-time subscriptions, `pg_cron`) cover several adjacent needs.

**Consequences**:
- Vendor concentration on Supabase, mitigated by the fact that the underlying database is standard PostgreSQL, portable to RDS, Cloud SQL, or self-hosted with minimal changes.
- Scales to 64 cores / 256 GB RAM on Supabase before self-hosting becomes a forced move.
- HNSW indexes return candidates *before* WHERE filters, so tenant filtering must happen *inside* the search query, handled in the hybrid search RPC.

---

## D-008: LiteLLM for Model-Agnostic AI

**Status**: Accepted
**Context**: Direct SDK calls to a single LLM provider create two problems: vendor lock-in (model upgrades require code changes) and zero AI fault tolerance (provider outage means AI features go down). For a product that runs daily compliance workflows, neither is acceptable.

**Decision**: All AI inference flows through a LiteLLM proxy.

- **100+ providers** behind a single OpenAI-compatible interface, so never locked into a single vendor.
- **Hard budget caps** per tenant and globally, enforced at proxy level before the request hits a provider.
- **Automatic fallback routing**: if the primary provider returns 429 or 5xx, the proxy fails over to the next in the chain (Claude → GPT → Gemini, configured per tier).
- **Per-request cost tracking**: every LLM call has known cost attribution to a tenant.
- **Semantic caching**: duplicate prompts return cached responses, cutting cost on the assistive paths.

**Consequences**:
- Added dependency on LiteLLM, mitigated by the project being open source and self-hostable if the hosted layer ever becomes a constraint.
- Better LLMs improve the product with zero code changes: just update the provider config.
- AI provider outages no longer cause AI feature downtime; they cause a graceful degradation through the fallback chain.

---

## D-009: AI Never in the Critical Path

**Status**: Accepted
**Context**: Compliance decisions must be 100% reliable, fast (<100ms), and auditable. LLM latency (1-5 seconds typical) is incompatible with real-time order gating. LLM hallucination risk is incompatible with tax calculations. Regulatory requirements demand deterministic, reproducible results: the same order under the same rules must produce the same answer on every check.

**Decision**: Compliance checks and tax calculations are deterministic rules engine operations. AI sits *next to* the critical path, never inside it.

- **Real-Time Order Compliance Gate**: pure rules engine. Looks up jurisdiction rules from Postgres, returns allowed/denied with a structured reason code.
- **Tax Calculation**: pure rules engine. Walks tax-rule lookup tables for jurisdiction × product type × channel.
- **AI workloads**: natural-language compliance questions, regulatory document extraction, expansion advice, audit report synthesis. None of these block an order or a filing.

**Consequences**:
- An AI provider outage degrades assistive features (the NL assistant goes down) but never breaks the core product.
- Compliance answers are reproducible: the same inputs always produce the same outputs.
- Every compliance decision has a deterministic audit trail with a citation to the underlying rule, not a model output.
- AI is positioned where it excels: ambiguous, unstructured, generative work. Compliance is positioned where determinism excels: bounded rule evaluation.

---

## D-010: Jurisdiction-Agnostic Data Model

**Status**: Accepted
**Context**: A naive design models tax and compliance rules as `state_rules`, which works for the US domestic case but forces a refactor the moment counties (dry counties, local ordinances), cities, territories (Puerto Rico, Guam, USVI), or international jurisdictions enter the picture.

**Decision**: Use `jurisdiction_rules` (not `state_rules`) with a `jurisdiction_type` ENUM and an ISO-style `jurisdiction_code` column.

- `jurisdiction_type` supports `state`, `county`, `city`, `country`, `territory`.
- `jurisdiction_code` carries the canonical identifier (e.g., `US-CA`, `US-CA-LOS_ANGELES`, `FR`).
- Same schema serves US states today and international jurisdictions later with zero migrations.
- Cannabis-style municipality opt-in/opt-out structures are handled by the existing model.

**Consequences**:
- Slightly more verbose than `state_code VARCHAR(2)` would have been, an acceptable cost for the long-tail flexibility.
- Queries are jurisdiction-shaped, not state-shaped, a one-time learning curve.
- Schema decision made once at the start of the project; refactoring later would have rippled across every rule lookup.

---

## D-049: Two-Pass Citations Extraction

**Status**: Accepted
**Context**: A compliance product is only useful if its rules are correct, and "correct" means traceable to a verbatim source. Naive single-pass LLM extraction produces plausible-looking JSON with no audit trail: there is no way to retrace which span of the source document produced which extracted value, no way to defend a conclusion in a compliance review, and no way to ground a regression test.

**Decision**: Two-pass extraction using Anthropic's Citations API plus Structured Outputs.

- **Pass 1** locates citation spans in the source document: exact verbatim quotes anchored to character offsets.
- **Pass 2** validates extracted values against the schema *and* against the cited spans returned by pass 1.
- **Ephemeral prompt caching** with 5-minute TTL and 0.10x read-rate billing cuts cost ~10x for repeated source documents (state DOR pages are large and re-fetched frequently).
- **Model tiering**: Claude Sonnet 4.6 for primary extraction quality, Claude Haiku 4.5 for cheap-path workloads where the document is small or pre-screened.

**Consequences**:
- Every extracted compliance value carries a verbatim citation back to the source document: defensible in audit, replayable in CI.
- Golden eval fixtures freeze the cited spans, not just the output JSON, so prompt drift surfaces immediately as a citation mismatch.
- Higher per-call latency and cost than a single-pass extract, offset by prompt caching and Haiku-tier routing on cheap paths.
- Wave 3 used this pattern to land judge-grade HIGH-confidence wine-DTC extractors across all 51 US jurisdictions.
