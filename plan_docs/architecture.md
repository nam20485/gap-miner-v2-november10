# GapMiner — Architecture

**Document Type:** Architecture Overview
**Project:** Gap Mining Platform (GapMiner)
**Source of truth:** `plan_docs/development-plan.md` (§1 strategic context, §4 repository layout, §6–§12 phases, §16 parallel execution map)
**Version:** 1.0
**Date:** 2026-07-05
**Status:** Approved (planning phase)

---

## 1. Purpose

This document describes the **high-level architecture** of GapMiner: its components, how they
communicate, the end-to-end data flow, and the key design decisions and their rationale. It is a
synthesis of `development-plan.md` intended for orientation; the development plan remains the
authoritative task-level spec.

---

## 2. What GapMiner Is

GapMiner is an **internal intelligence engine** (not a customer-facing product). Its mission:

1. **Scrape** 1–3 star reviews of competitor apps from digital marketplaces (Shopify App Store,
   Chrome Web Store, G2, Apple App Store) via Apify.
2. **Cluster and analyze** negative reviews with LLMs to identify *substantial, monetizable feature
   gaps* (missing capabilities, not bug fixes).
3. **Surface** ranked opportunities on an internal Blazor dashboard for human strategic review.

It does **not** build, ship, or monetize micro-SaaS applications — that is a downstream activity.

---

## 3. High-Level Architecture

```
                          ┌─────────────────────────────────────────────┐
                          │            Aspire AppHost (orchestrator)     │
                          │   AddPostgres(pgvector) · AddRedis · projects│
                          └─────────────────────────────────────────────┘
                 service discovery / connection strings (WithReference)
        ┌──────────────────────┬─────────────────────┬──────────────────────┐
        ▼                      ▼                     ▼                      ▼
┌──────────────┐      ┌─────────────────┐   ┌──────────────────┐   ┌─────────────────┐
│  GapMiner.Api│      │ GapMiner.Web    │   │GapMiner.Scraper- │   │ GapMiner.AIWorker│
│  (Minimal API)│     │ (Blazor Server) │   │     Worker       │   │                  │
│  /api/v1/*   │◀─────│  dashboard      │   │ ScrapeReviewsJob │   │ EmbedReviewsJob  │
└──────┬───────┘      └─────────────────┘   │ Analyze enqueue  │   │ AnalyzeGapsJob   │
       │                                    └────────┬─────────┘   └────────┬─────────┘
       │                                             │                      │
       │              ┌──────────────────────────────┘                      │
       │              ▼                                                     │
       │      ┌───────────────────┐   enqueue commands    ┌─────────────────┘
       │      │  Redis Job Queue  │◀──────────────────────│  (enqueues itself
       │      │ IJobQueue<TCmd>   │   commands flow via   │   when more work)
       │      └─────────┬─────────┘   the queue, not HTTP │
       │                │                               │
       │      ┌─────────┴────────┐                      │
       │      │   External APIs  │                      │
       │      │  Apify v2 REST   │  (ScrapeReviewsJob)  │
       │      └──────────────────┘                      │
       │                                                ▼
       │            ┌──────────────────────────────────────────┐
       └────────────┤         PostgreSQL 16 + pgvector         │
                    │  CompetitorTarget · Review(+Embedding)   │
                    │  FeatureGap                              │
                    └──────────────────────────────────────────┘
                                     ▲
                                     │ embeddings + LLM calls
                            ┌────────┴─────────┐
                            │ Semantic Kernel  │  (EmbedReviewsJob, AnalyzeGapsJob)
                            │ Azure OpenAI /   │
                            │ Anthropic Claude │
                            └──────────────────┘
```

---

## 4. Components (from development-plan.md §4)

### 4.1 Orchestration

| Component | Responsibility |
|---|---|
| `GapMiner.AppHost` | Aspire orchestrator. Declares PostgreSQL (pgvector), Redis, and project references (Api, Web, ScraperWorker, AIWorker). Wires connection strings via `WithReference()`. Launches the Aspire dashboard. |
| `GapMiner.ServiceDefaults` | Shared Aspire service defaults: OpenTelemetry tracing/logging, health checks (`/health`, `/alive`). Consumed by every downstream project via `builder.AddServiceDefaults()`. |

### 4.2 Domain & Application

| Component | Responsibility |
|---|---|
| `GapMiner.Domain` | Pure domain entities & value objects — **no EF references**. `CompetitorTarget`, `Review` (incl. `Embedding float[]`), `FeatureGap`, `SeverityScore` value object, `MarketplaceKind` enum. |
| `GapMiner.Application` | Use cases / command handlers: `IngestTarget`, `RunScraper`, `AnalyzeReviews`, `GetFeatureGaps`. DTOs. Coordinates domain + infrastructure without owning I/O side effects. |

### 4.3 Infrastructure

| Component | Responsibility |
|---|---|
| `GapMiner.Infrastructure/Persistence` | `GapMinerDbContext`, per-entity `IEntityTypeConfiguration`, `Migrations/`, and `Repositories/` (`ICompetitorTargetRepository`, `IReviewRepository`, `IFeatureGapRepository`). pgvector enabled; idempotency indexes. |
| `GapMiner.Infrastructure/Queueing` | `IJobQueue<TCommand>` + `RedisJobQueue<TCommand>` (FIFO via `RPUSH`/`LPOP`, JSON-serialized). Decouples producers (API) from consumers (workers). |
| `GapMiner.Infrastructure/Scraping` | `IApifyClient` (Refit) + bearer-token handler + Polly retry. Typed DTOs for actor-run status and dataset items. |
| `GapMiner.Infrastructure/AI` | `IGapAnalyzer` + `SemanticKernelGapAnalyzer`, plus version-controlled prompt library (`Prompts/*.txt`). |

### 4.4 Hosts (entry points)

| Component | Responsibility |
|---|---|
| `GapMiner.Api` | Minimal API gateway. Endpoints under `/api/v1/*` (targets, gaps, jobs) + Swagger/OpenAPI + FluentValidation. |
| `GapMiner.ScraperWorker` | Background worker hosting `ScrapeReviewsJob` (Hangfire). |
| `GapMiner.AIWorker` | Background worker hosting `EmbedReviewsJob` and `AnalyzeGapsJob` (Hangfire). |
| `GapMiner.Web` | Blazor Web App (Interactive Server). Pages: Dashboard, Targets, Jobs, Opportunity Matrix. |

### 4.5 Tests

| Project | Scope |
|---|---|
| `GapMiner.Domain.Tests` | Value-object/entity invariants. |
| `GapMiner.Infrastructure.Tests` | Repository + queue tests with Testcontainers (Postgres, Redis). |
| `GapMiner.Application.Tests` | Command/query handler unit tests. |
| `GapMiner.Api.Tests` | Endpoint tests via `WebApplicationFactory`. |
| `GapMiner.Integration.Tests` | End-to-end (`FullPipeline_ShouldProduceGaps`) with Testcontainers + mocked LLM. |

---

## 5. End-to-End Data Flow

The core pipeline is **ingest → scrape → embed → cluster → analyze → surface**:

1. **Ingest** — User adds a competitor target via the API/UI. `POST /api/v1/targets` validates the
   URL (must match the marketplace pattern) and **enqueues** a `ScrapeTargetCommand` onto the Redis
   scrape queue. *(T-4.1, T-5.2)*

2. **Scrape** — `ScrapeReviewsJob` (ScraperWorker) dequeues the command, resolves the Apify actor
   via the Marketplace Actor Registry, triggers the actor run filtered to 1–3 star reviews, polls
   status, downloads the dataset, maps items to `Review` entities using the registry field map, and
   **bulk-inserts with idempotency** (`ON CONFLICT DO NOTHING` on the
   `(CompetitorTargetId, SourceReviewId)` unique index). On success it enqueues an
   `AnalyzeReviewsCommand`. *(T-2.1, T-2.2, T-2.3)*

3. **Embed** — `EmbedReviewsJob` (AIWorker) pulls batches of unembedded reviews (≤100 per API call),
   generates 3072-dim vectors, stores them in `Review.Embedding`, and **re-enqueues itself** while
   unembedded reviews remain. *(T-3.1, T-3.2)*

4. **Cluster** — Semantic clustering groups embedded reviews by cosine similarity using pgvector
   KNN + k-means, producing `ReviewCluster`s with representative reviews. *(T-3.3)*

5. **Analyze (Map→Reduce)** — `AnalyzeGapsJob` runs the two-stage LLM pipeline:
   - **Map:** per cluster, `MapReviewsPrompt.txt` extracts candidate gaps as strict JSON.
   - **Reduce:** candidates are deduplicated by title similarity (>0.8 cosine), then
     `ReduceGapsPrompt.txt` produces a final ranked list (≤10 gaps, severity desc), validated
     against schema before persistence as `FeatureGap` records. One retry on invalid output, then
     dead-letter. *(T-3.4)*

6. **Surface** — The Blazor dashboard reads ranked gaps via `/api/v1/gaps` (sortable by severity or
   frequency, filterable by target/marketplace) and renders the **Opportunity Matrix**
   (Severity × Frequency scatter). The Jobs page shows live Hangfire queue state. *(T-5.x)*

```
UI/API ──(enqueue)──▶ Redis ──▶ ScraperWorker ──(Apify)──▶ reviews
                                                       │
                                              (idempotent bulk insert)
                                                       ▼
                        AIWorker ◀──(enqueue)──  PostgreSQL + pgvector
                          │  embed                 ▲
                          │  cluster (pgvector)    │
                          │  map→reduce (LLM) ─────┘
                          ▼
                    FeatureGap records ──▶ /api/v1/gaps ──▶ Opportunity Matrix
```

---

## 6. Key Design Decisions

| # | Decision | Rationale |
|---|---|---|
| **D1** | **Aspire for local orchestration** | Single `dotnet run --project src/GapMiner.AppHost` brings up Postgres, Redis, and all services with wired connection strings and a live dashboard. Removes hand-rolled compose choreography during dev. |
| **D2** | **Separated workers (ScraperWorker vs AIWorker)** | Scraping (I/O-bound, Apify rate limits) and AI (LLM/embedding latency, token cost) have disjoint failure modes and scaling. Separate processes let them fail/retry independently. |
| **D3** | **Redis queue abstraction (`IJobQueue<TCommand>`)** | Decouples API producers from worker consumers and avoids tight coupling to Hangfire's storage. Commands are JSON; the queue is testable with Testcontainers. |
| **D4** | **pgvector for semantic clustering** | Keeps vectors co-located with relational data — clustering and ranked queries run in one store with `<=>` cosine distance. Avoids a separate vector DB. |
| **D5** | **Idempotency by construction** | Unique composite index on `Review(CompetitorTargetId, SourceReviewId)` + `ON CONFLICT DO NOTHING` means re-scraping a target never duplicates reviews; re-analysis produces no duplicate `FeatureGap`s. (Audit: T-6.3.) |
| **D6** | **Map→Reduce LLM analysis** | Maps within a single context window per cluster (≤8k tokens) to avoid truncation, then reduces/dedupes globally. Strict JSON schema + 1 retry + dead-letter guards against malformed LLM output. |
| **D7** | **Version-controlled prompt library** | Prompts live as `.txt` under `Infrastructure/AI/Prompts/` (not inline strings), loaded at runtime — reviewable, diffable, auditable. |
| **D8** | **Central Package Management + exact pinning** | `Directory.Packages.props` is the single source of truth; no floating versions (mitigates Semantic Kernel / EF API drift). `global.json` pins SDK 8.0.x. |
| **D9** | **Dead-letter queue + retry policy** | Every job has `MaxRetryAttempts` (default 3); failures land in a Redis dead-letter list for human review (T-6.3). No silent failures. |
| **D10** | **Secret hygiene** | All keys via `IConfiguration`/Aspire parameters; Serilog scrubs `*Password*`/`*Token*`/`*Key*` from logs. |

---

## 7. Quality Attributes

| Attribute | How it is met |
|---|---|
| **Observability** | ServiceDefaults adds OpenTelemetry traces (scrape, LLM, embedding calls) + Serilog JSON logs + custom metrics; all visible on the Aspire dashboard (T-6.2). |
| **Reliability** | Polly retries on Apify/LLM 429–5xx; idempotent inserts; dead-letter queue; Hangfire durable scheduling. |
| **Testability** | Clean layering (Domain → Application → Infrastructure → Hosts); LLM mocked via NSubstitute in integration tests; Testcontainers for Postgres/Redis. |
| **Security** | No hardcoded secrets; bearer-token handlers from config; AGPL-3.0; gitleaks in CI. |
| **Maintainability** | XML doc comments on all public methods; StyleCop + SonarAnalyzer gates; file-scoped namespaces; Conventional Commits. |

---

## 8. Module Boundaries & Dependencies

```
Domain ─────────────────────────────────────── (no dependencies)
   ▲
Application ──▶ Domain
   ▲
Infrastructure ──▶ Domain   (EF, Redis, Apify, Semantic Kernel)
   ▲
Hosts (Api, ScraperWorker, AIWorker, Web) ──▶ Application + Infrastructure + ServiceDefaults
```

- **Domain** has zero project references (pure).
- **Application** depends only on Domain.
- **Infrastructure** depends on Domain (never on Application).
- **Hosts** depend on Application + Infrastructure and compose the DI graph.
- This direction keeps the core independent of frameworks and I/O.

---

## 9. Known Risks (architectural)

| Risk | Mitigation (phase/task) |
|---|---|
| LLM output violates JSON schema | Strict schema validation + 1 retry + dead-letter (T-3.4). |
| Apify actor ID incorrect / hallucinated | Registry marks unknown actors `NeedsVerification`; runtime throws `MarketplaceNotSupportedException`; agent must verify against live Apify Store (T-2.2). |
| pgvector missing in container | Use `pgvector/pgvector:pg16` image; `HasPostgresExtension("vector")` (T-0.2). |
| Context-window overflow on large review sets | Map calls capped at ≤8,000 tokens (T-3.4). |
| Race condition on re-scrape | Unique composite index + `ON CONFLICT DO NOTHING` (T-1.2). |
| Semantic Kernel / EF API drift | Exact version pinning in `Directory.Packages.props` (T-0.1). |
| Secret leakage in logs | Serilog scrubs sensitive properties (T-6.2). |

(Full table in `development-plan.md` §14.)

---

## 10. Phase → Component Mapping

| Phase | Primary components delivered |
|---|---|
| 0 — Foundation | AppHost, ServiceDefaults, root config (`global.json`, `Directory.*.props`, `.editorconfig`) |
| 1 — Domain & Data | Domain entities, `GapMinerDbContext` + migrations, repositories, `IJobQueue`/`RedisJobQueue` |
| 2 — Scraper Pipeline | `IApifyClient`, Marketplace Actor Registry, `ScrapeReviewsJob` |
| 3 — Intelligence Pipeline | Semantic Kernel bootstrap, `EmbedReviewsJob`, semantic clustering, `AnalyzeGapsJob` |
| 4 — API Gateway | Minimal API endpoints (targets, gaps, jobs) + Swagger + validation |
| 5 — Blazor Dashboard | Layout/Nav, Targets, Jobs, Opportunity Matrix |
| 6 — Hardening | E2E integration test, observability, error/idempotency audit |

---

*Cross-references: tech-stack.md (exact versions), development-plan.md (task-level spec),
workflow-plan.md (build sequencing).*
