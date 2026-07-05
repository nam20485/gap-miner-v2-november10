# GapMiner — Technology Stack

**Document Type:** Technology Stack Reference
**Project:** Gap Mining Platform (GapMiner)
**Source of truth:** `plan_docs/development-plan.md` §3 (Technology Stack) and T-0.1 (`Directory.Packages.props`)
**Version:** 1.0
**Date:** 2026-07-05
**Status:** Approved (planning phase)

---

## 1. Purpose

This document enumerates **every technology** used by GapMiner, the **exact version** that must be
pinned, and the **role** each plays in the system. It is the canonical reference for
`Directory.Packages.props` (Central Package Management) and `global.json`.

> **Rule (from development-plan.md R2):** *Never invent or float package versions.* Every version
> below is fixed. Central Package Management (`Directory.Packages.props`) is the single source of
> truth for NuGet versions; individual `.csproj` files MUST NOT set `Version` attributes.

---

## 2. Stack at a Glance

| Layer | Technology | Version | NuGet Package(s) |
|---|---|---|---|
| Runtime | .NET SDK | **8.0.x (LTS)** | — (`global.json`) |
| Distributed orchestration | .NET Aspire | **8.2.x** | `Aspire.Hosting.AppHost` 8.2.2 |
| Web UI | Blazor Web App (Interactive Server) | **8.0** | `Microsoft.AspNetCore.Components.WebAssembly.Server` (framework) |
| API | ASP.NET Core Minimal API | **8.0** | (framework) |
| Database | PostgreSQL | **16.x** | `Npgsql.EntityFrameworkCore.PostgreSQL` **8.0.10** |
| Vector search | pgvector | **0.7.x** | `Pgvector.EntityFrameworkCore` **0.2.0** |
| ORM | Entity Framework Core | **8.0.x** | `Microsoft.EntityFrameworkCore.*` 8.0.x |
| Queue / cache | Redis (StackExchange.Redis) | **7.x** | `StackExchange.Redis` 7.x |
| Job scheduling | Hangfire | **1.8.x** | `Hangfire.Core` **1.8.14** |
| AI orchestration | Microsoft.SemanticKernel | **1.20.x** | `Microsoft.SemanticKernel` **1.20.0** |
| LLM provider | Azure OpenAI (GPT-4o) **or** Anthropic Claude 3.5 Sonnet | Latest stable | (Semantic Kernel connectors) |
| Embeddings | `text-embedding-3-large` (OpenAI) or `voyage-3` | Latest stable | (Semantic Kernel connectors) |
| Scraping | Apify API | **v2 REST** | (Refit interface) |
| HTTP client | Refit | **7.x** | `Refit.HttpClientFactory` **7.2.1** |
| Validation | FluentValidation | **11.x** | `FluentValidation` **11.10.0** |
| Resilience | Polly | (bundled w/ Aspire / .NET 8) | `Microsoft.Extensions.Http.Resilience` |
| Testing | xUnit + NSubstitute + Testcontainers | Latest stable | `xunit`, `NSubstitute`, `Testcontainers.*` |
| Code quality | SonarAnalyzer.CSharp + StyleCop.Analyzers | Latest stable | analyzers (MSBuild) |
| Observability | Serilog + OpenTelemetry | Latest stable | `Serilog.*`, `OpenTelemetry.*` (via Aspire) |
| Charts (UI) | Blazor-ApexCharts **or** AntDesign.Charts | Latest stable | (chosen in Phase 5) |
| Component library (UI) | MudBlazor **or** Radzen.Blazor | Latest stable | (chosen in Phase 5) |

---

## 3. Pinned NuGet Versions (from T-0.1 `Directory.Packages.props`)

These are the authoritative pinned versions. They go verbatim into `Directory.Packages.props`.

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <!-- Aspire hosting & integrations -->
    <PackageVersion Include="Aspire.Hosting" Version="8.2.2" />
    <PackageVersion Include="Aspire.Hosting.PostgreSQL" Version="8.2.2" />
    <PackageVersion Include="Aspire.Hosting.Redis" Version="8.2.2" />

    <!-- AI -->
    <PackageVersion Include="Microsoft.SemanticKernel" Version="1.20.0" />

    <!-- Persistence -->
    <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.0.10" />
    <PackageVersion Include="Pgvector.EntityFrameworkCore" Version="0.2.0" />

    <!-- Jobs / queues -->
    <PackageVersion Include="Hangfire.Core" Version="1.8.14" />

    <!-- HTTP / scraping -->
    <PackageVersion Include="Refit.HttpClientFactory" Version="7.2.1" />

    <!-- Validation -->
    <PackageVersion Include="FluentValidation" Version="11.10.0" />
  </ItemGroup>
</Project>
```

> Additional packages (Serilog, OpenTelemetry, Testcontainers, analyzers, chart/component libs) are
> added in their respective phases but remain centrally managed. See §4 for their roles.

---

## 4. Technology Roles & Rationale

### 4.1 Runtime & Orchestration

| Technology | Version | Role |
|---|---|---|
| **.NET SDK 8.0.x (LTS)** | 8.0.x | Target framework for all projects. LTS channel for long-term support. Pinned via `global.json` with `rollForward: latestFeature`. |
| **.NET Aspire** | 8.2.x | Local orchestration of the distributed app (PostgreSQL, Redis, API, Web, ScraperWorker, AIWorker). Provides the dashboard, service discovery, and `WithReference()` connection-string wiring. Hosting integration packages bring Postgres + Redis resources. |

> **Note (Open Question #1):** The devcontainer ships SDK 10, but GapMiner targets 8.0.x.
> `global.json` with `rollForward: latestFeature` must resolve to an installed SDK 8 build; if SDK 8
> is absent it must be installed (the `create-project-structure` assignment owns this).

### 4.2 Web & API

| Technology | Version | Role |
|---|---|---|
| **Blazor Web App (Interactive Server)** | 8.0 | Internal dashboard. Server-side interactive rendering for live data (job queue polling, opportunity matrix) with no client download burden. |
| **ASP.NET Core Minimal API** | 8.0 | Lightweight HTTP gateway exposing `/api/v1/*` endpoints (targets, gaps, jobs) consumed by the Blazor UI and any future automation. |

### 4.3 Data Layer

| Technology | Version | Role |
|---|---|---|
| **PostgreSQL** | 16.x | Primary relational store for `CompetitorTarget`, `Review`, `FeatureGap`. Chosen for mature JSONB, indexing, and the pgvector extension. |
| **pgvector** | 0.7.x | PostgreSQL extension enabling vector similarity search (`<=>` cosine distance) over `Review.Embedding` for semantic clustering. Enabled via `HasPostgresExtension("vector")`; `Review.Embedding` mapped as `vector(3072)`. |
| **EF Core** | 8.0.x | ORM. `GapMinerDbContext` with `IEntityTypeConfiguration` per entity. Migrations (`InitialSchema`) applied at AppHost startup (dev only). Npgsql provider 8.0.10 pairs with the pgvector EF provider 0.2.0. |
| **Redis (StackExchange.Redis)** | 7.x | Job queue transport (`gapminer:{domain}:{action}` keys) and cache. Decouples the API/UI from the workers via a FIFO list abstraction (`IJobQueue<TCommand>`). |

### 4.4 Background Processing

| Technology | Version | Role |
|---|---|---|
| **Hangfire** | 1.8.x | Durable job scheduling and retry for `ScrapeReviewsJob`, `EmbedReviewsJob`, `AnalyzeGapsJob`. Provides job history surfaced on the Jobs page. |

### 4.5 AI / Intelligence

| Technology | Version | Role |
|---|---|---|
| **Microsoft.SemanticKernel** | 1.20.x | AI orchestration kernel. Registers `IChatCompletionService` (gap analysis) and embedding generation. Provider-switchable (Azure OpenAI vs Anthropic) via `IConfiguration["AI:Provider"]`. |
| **Azure OpenAI GPT-4o / Anthropic Claude 3.5 Sonnet** | Latest stable | Chat completion for the Map→Reduce gap analysis (MapReviewsPrompt, ReduceGapsPrompt). |
| **text-embedding-3-large / voyage-3** | Latest stable | 3072-dim embeddings stored on `Review.Embedding` for semantic clustering. |

### 4.6 Integration & Resilience

| Technology | Version | Role |
|---|---|---|
| **Refit** | 7.x | Strongly-typed REST client for the Apify v2 API (`IApifyClient`). Wired through `Refit.HttpClientFactory`. |
| **Apify API** | v2 REST | External scraping service. Actor runs fetch 1–3 star reviews from Shopify App Store, Chrome Web Store, G2, Apple App Store. |
| **Polly / `Microsoft.Extensions.Http.Resilience`** | bundled | Retry policy: 3 attempts on 429/5xx with exponential backoff (Apify + embedding/LLM calls). |
| **FluentValidation** | 11.x | Request validation at the API boundary with a consistent `ProblemDetails` error shape. |

### 4.7 Testing & Quality

| Technology | Version | Role |
|---|---|---|
| **xUnit** | Latest stable | Unit + integration test framework. |
| **NSubstitute** | Latest stable | Mocking library for services/LLM in unit and integration tests. |
| **Testcontainers** | Latest stable | Ephemeral PostgreSQL 16 + Redis 7 containers for repository and end-to-end tests. |
| **SonarAnalyzer.CSharp** | Latest stable | Static analysis gate (zero new violations). |
| **StyleCop.Analyzers** | Latest stable | Style enforcement (file-scoped namespaces, 4-space indent, CRLF). |

### 4.8 Observability & UI Extras

| Technology | Version | Role |
|---|---|---|
| **Serilog** | Latest stable | Structured (JSON) logging; scrubs `*Password*`, `*Token*`, `*Key*` properties. |
| **OpenTelemetry** | Latest stable (via Aspire) | Traces for scrape jobs, LLM calls, embedding calls; custom metrics (`gapminer.reviews.scraped`, `gapminer.gaps.identified`, `gapminer.llm.tokens.consumed`). Surfaced on the Aspire dashboard. |
| **MudBlazor / Radzen.Blazor** | Latest stable | Component library for the dashboard design system (chosen in Phase 5, T-5.1). |
| **Blazor-ApexCharts / AntDesign.Charts** | Latest stable | Opportunity Matrix scatter plot (Severity × Frequency) (T-5.4). |

---

## 5. Local Dev Infrastructure Images

| Service | Image | Why |
|---|---|---|
| PostgreSQL + pgvector | `pgvector/pgvector:pg16` | Postgres 16 **with the vector extension preinstalled** — plain `postgres:16` lacks pgvector and breaks EF migrations (`HasPostgresExtension("vector")`). |
| Redis | `redis:7` | Matches the StackExchange.Redis 7.x client. |

These are used by both Aspire (`AddPostgres`/`AddRedis`) and `docker-compose` for local dev.

---

## 6. Secrets & Configuration

| Secret | Source | Used by |
|---|---|---|
| `Apify:Token` | Aspire parameter / env var | `IApifyClient` auth handler (T-2.1) |
| `AI:Provider`, `AI:ApiKey`, `AI:Endpoint` | Aspire parameter / env var | Semantic Kernel bootstrap (T-3.1) |
| `ConnectionStrings:gapminer` | Aspire `WithReference(postgres)` | `GapMinerDbContext` |
| `ConnectionStrings:redis` | Aspire `WithReference(redis)` | `RedisJobQueue` |

> **Rule (R5):** *Never hardcode secrets.* All keys come from `IConfiguration` or Aspire secret
> parameters — never from `appsettings.json`.

---

## 7. Version Discipline

- **No floating versions.** Every `<PackageVersion>` is an exact, pinned number.
- **Central Package Management** is mandatory (`ManagePackageVersionsCentrally=true`).
- Adding a dependency = add one `<PackageVersion>` line here, reference it from the consuming
  `.csproj` with `<PackageReference Include="…" />` (no `Version`).
- SDK is pinned in `global.json`; `rollForward: latestFeature` allows patch/feature updates within
  8.0.x only.

---

*Cross-references: architecture.md (how these components fit together),
development-plan.md (task-by-task implementation), workflow-plan.md (how the build is sequenced).*
