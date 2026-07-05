# Workflow Execution Plan: `project-setup`

**Document Type:** Workflow Execution Plan (Dynamic Workflow)
**Target Workflow:** `project-setup`
**Repository:** `nam20485/gap-miner-v2-november10`
**Project:** Gap Mining Platform (GapMiner)
**Date:** 2026-07-05
**Status:** Approved (automated; orchestrator-delegated approval — see §6)

---

## 1. Overview

This document is the **execution plan for the `project-setup` dynamic workflow** as applied to
the Gap Mining Platform (GapMiner) repository. It traces every assignment in the workflow,
maps each to the concrete project context drawn from `plan_docs/`, and records dependencies,
risks, and acceptance criteria so the orchestrator can execute predictably.

> **Scope clarification:** This plan covers *how to execute the workflow assignments* (repository
> scaffolding, project structure, planning issues, AGENTS.md, PR merge). It is **not** the
> application implementation plan — that is produced by the `create-app-plan` assignment (which
> synthesizes `plan_docs/development-plan.md`).

### Workflow at a Glance

| Item | Value |
|---|---|
| Dynamic workflow | `project-setup` |
| Main assignments | **6** (`init-existing-repository` → `pr-approval-and-merge`) |
| Pre-script events | 1 (`create-workflow-plan` ← this document) |
| Post-assignment events | 2 per main assignment (`validate-assignment-completion`, `report-progress`) = **12** |
| Post-script events | 1 (apply `orchestration:plan-approved` label) |
| Setup branch | `dynamic-workflow-project-setup` |
| Output PR | Opened in `init-existing-repository`; merged in `pr-approval-and-merge` |

### High-Level Summary

The workflow bootstraps the GapMiner repository from an orchestration-template state into a
ready-to-develop .NET 8 / Aspire project. It: (1) sets up GitHub admin (branch protection,
project board, labels), (2) publishes the application plan as a tracking issue with milestones,
(3) scaffolds the full .NET solution (9 src + 5 test projects per the development plan), (4)
writes an app-specific `AGENTS.md`, (5) captures a debrief, and (6) merges the setup PR through
a CI remediation loop. On completion, the `orchestration:plan-approved` label is applied to the
plan issue to trigger the next orchestration phase (epic creation).

---

## 2. Project Context Summary

Key facts drawn from `plan_docs/development-plan.md` and `plan_docs/Strategic Feasibility and
Execution Plan for AI-Accelerated Micro-SaaS Ecosystems.md` that govern how each assignment
must be executed.

### 2.1 What GapMiner Is

An **internal intelligence engine** (not a customer-facing product) that:
1. Scrapes 1–3 star reviews of competitor apps from Shopify App Store, Chrome Web Store, G2,
   Apple App Store (via Apify).
2. Clusters and analyzes negative reviews with LLMs (Semantic Kernel + Azure OpenAI/Claude) to
   identify substantial, monetizable feature gaps.
3. Surfaces ranked opportunities via an internal **Blazor Server** dashboard.

### 2.2 Technology Stack (Exact Versions — Do Not Deviate)

| Component | Technology | Version |
|---|---|---|
| Runtime | .NET SDK | 8.0.x (LTS) |
| Orchestration | .NET Aspire | 8.2.x |
| Web UI | Blazor Web App (Interactive Server) | 8.0 |
| API | ASP.NET Core Minimal API | 8.0 |
| Database | PostgreSQL | 16.x |
| Vector Extension | pgvector | 0.7.x |
| ORM | Entity Framework Core | 8.0.x |
| Queue / Cache | Redis (StackExchange.Redis) | 7.x |
| Job Scheduling | Hangfire | 1.8.x |
| AI Orchestration | Microsoft.SemanticKernel | 1.20.x |
| LLM Provider | Azure OpenAI (GPT-4o) or Anthropic Claude 3.5 Sonnet | Latest stable |
| Embeddings | text-embedding-3-large (OpenAI) or voyage-3 | Latest stable |
| Scraping | Apify API | v2 REST |
| HTTP Client | Refit | 7.x |
| Validation | FluentValidation | 11.x |
| Testing | xUnit + NSubstitute + Testcontainers | Latest stable |
| Code Quality | SonarAnalyzer.CSharp + StyleCop.Analyzers | Latest stable |

### 2.3 Repository Layout (Target — from development-plan.md §4)

- **9 source projects:** `GapMiner.AppHost`, `GapMiner.ServiceDefaults`, `GapMiner.Domain`,
  `GapMiner.Infrastructure`, `GapMiner.Application`, `GapMiner.Api`, `GapMiner.ScraperWorker`,
  `GapMiner.AIWorker`, `GapMiner.Web`
- **5 test projects:** `GapMiner.Domain.Tests`, `GapMiner.Infrastructure.Tests`,
  `GapMiner.Application.Tests`, `GapMiner.Api.Tests`, `GapMiner.Integration.Tests`
- **Root files:** `GapMiner.sln`, `Directory.Build.props`, `Directory.Packages.props`,
  `.editorconfig`, `global.json`, `README.md`
- **Phases:** 0 (Foundation), 1 (Domain/Data), 2 (Scraper), 3 (Intelligence), 4 (API),
  5 (Blazor UI), 6 (Testing/Hardening) — atomic tasks T-0.1 through T-6.3.

### 2.4 Repository Metadata

- License: AGPL-3.0 (public repo)
- Default branch: `main`
- Devcontainer image: `ghcr.io/nam20485/workflow-orchestration-prebuild/devcontainer:main-latest`
- Existing assets verified present: `.github/.labels.json` (188 lines, full label set),
  `.github/protected-branches_ruleset.json` (branch protection with required status checks
  `scan` + `lint`, Copilot code review, linear history), `.github/ISSUE_TEMPLATE/application-plan.md`
  (used by `create-app-plan`), `gap-miner-v2-november10.code-workspace` (already renamed to repo prefix).

### 2.5 Key Constraints (from development-plan.md §2.1)

- Pin exact package versions; never invent or float versions.
- `CancellationToken` on all async I/O.
- Conventional Commits (`feat(scope): description`).
- TreatWarningsAsErrors, nullable reference types, implicit usings.
- File-scoped namespaces, 4-space indent.
- Never hardcode secrets — use `IConfiguration` / Aspire parameters.

---

## 3. Assignment Execution Plan

Assignments execute in the order below. Each main assignment is followed by its
`post-assignment-complete` events (`validate-assignment-completion`, `report-progress`).

### 3.1 Pre-Script Event: `create-workflow-plan`

| Field | Detail |
|---|---|
| **Short ID** | `create-workflow-plan` |
| **Goal** | Produce this document: a comprehensive workflow execution plan covering every assignment, mapped to GapMiner project context. |
| **Key Acceptance Criteria** | Dynamic workflow file understood & all assignments traced; all `plan_docs/` files read & summarized; plan covers every assignment in execution order; plan committed to `plan_docs/workflow-plan.md`. |
| **Prerequisites** | Access to the `project-setup` dynamic workflow file and all referenced assignment definitions (remote canonical repo `nam20485/agent-instructions`). |
| **Dependencies** | None (runs first). |
| **Project-Specific Notes** | Source documents are unusually detailed — `development-plan.md` is a near-complete implementation spec (T-0.1–T-6.3). The app-plan step should synthesize rather than duplicate. |
| **Risks/Challenges** | None significant. |
| **Events** | This *is* the pre-script event. |

---

### 3.2 Assignment 1: `init-existing-repository`

| Field | Detail |
|---|---|
| **Short ID** | `init-existing-repository` |
| **Goal** | Initialize repo admin: create setup branch, import branch protection, create GitHub Project board, import labels, rename devcontainer/workspace files, open the setup PR. |
| **Key Acceptance Criteria** | (0) Branch `dynamic-workflow-project-setup` created **first**; (1) branch protection ruleset imported from `.github/protected-branches_ruleset.json`; (2) GitHub Project created (Board template, named after repo); (3) project linked to repo; (4) columns Not Started / In Progress / In Review / Done; (5) labels imported via `scripts/import-labels.ps1` from `.github/.labels.json`; (6) devcontainer `name` + `.code-workspace` renamed to repo prefix; (7) PR opened from branch → `main`. |
| **Prerequisites** | `GH_ORCHESTRATION_AGENT_TOKEN` with scopes `repo`, `project`, `read:project`, `administration: write`; `gh` CLI authenticated. Run `scripts/test-github-permissions.ps1` to verify. |
| **Dependencies** | None (first main assignment). |
| **Project-Specific Notes** | Branch protection ruleset already exists in-repo and enforces required status checks `scan` + `lint`, Copilot review, and linear history on `main`. The import must be idempotent (check for existing ruleset name `protected-branches` before POSTing). Labels file is comprehensive (188 lines). Workspace file `gap-miner-v2-november10.code-workspace` already appears renamed — verify and skip if already correct. **Critical output:** the PR number (`$pr_num`) from this step feeds into `pr-approval-and-merge`. |
| **Risks/Challenges** | (a) Branch protection import can fail if token lacks `administration: write` — must report exact error, not silently skip. (b) PR creation fails if no commits are pushed first — steps 4–5 (labels, renames) must commit before PR step. (c) GitHub Project (Projects v2) creation via API can be flaky; requires `node_id` linking. |
| **Events** | After completion → `validate-assignment-completion`, `report-progress`. |

---

### 3.3 Assignment 2: `create-app-plan`

| Field | Detail |
|---|---|
| **Short ID** | `create-app-plan` |
| **Goal** | Create the application plan as a GitHub Issue (using `.github/ISSUE_TEMPLATE/application-plan.md`), create milestones for each phase, link issue to project + milestone, apply labels. **Planning only — no code.** |
| **Key Acceptance Criteria** | Template analyzed; plan documented per Appendix A issue template; all phases broken down; tech stack followed; risks identified; issue created with milestones; issue added to GitHub Project; assigned to "Phase 1: Foundation" milestone; labels `planning` + `documentation` applied. Also: `plan_docs/tech-stack.md` and `plan_docs/architecture.md` produced. |
| **Prerequisites** | `init-existing-repository` complete (project board + labels exist). Has `pre-assignment-begin` sub-event: `gather-context`. |
| **Dependencies** | Assignment 1 (needs project board, labels, milestones infra). |
| **Project-Specific Notes** | `plan_docs/development-plan.md` is already a thorough implementation spec with 6 phases and atomic tasks. The app-plan issue should **synthesize and reference** it rather than re-deriving from scratch. Milestones should map to Phases 0–6: Phase 0 (Foundation), Phase 1 (Domain/Data), Phase 2 (Scraper), Phase 3 (Intelligence), Phase 4 (API), Phase 5 (Blazor UI), Phase 6 (Hardening). The strategic doc provides business context (review mining, marketplace compliance, "substantial gap" framework) — reference it for the plan's rationale section. **Do NOT apply `orchestration:plan-approved` here** — that is the post-script event's job. |
| **Risks/Challenges** | (a) Risk of over-duplication: the dev plan is already issue-ready. Mitigate by treating the issue as a summary/index pointing to the dev plan for task-level detail. (b) `gather-context` pre-event must complete first. (c) Milestone creation via `gh api` must use correct GraphQL/REST endpoints for Projects v2. |
| **Events** | `pre-assignment-begin`: `gather-context`. `on-assignment-failure`: `recover-from-error`. After completion → `validate-assignment-completion`, `report-progress`. |

---

### 3.4 Assignment 3: `create-project-structure`

| Field | Detail |
|---|---|
| **Short ID** | `create-project-structure` |
| **Goal** | Scaffold the .NET solution and all projects per `development-plan.md` §4 (Repository Layout) and §6 (Phase 0 tasks T-0.1–T-0.3). Establish Docker, CI, docs, and dev environment. |
| **Key Acceptance Criteria** | Solution/project structure created per tech stack; all project files + dirs established; config files created (version pinning, Docker); CI/CD pipeline structure; docs structure (README, docs/, ADRs); dev environment configured + validated; initial commit made; repository summary (`.ai-repository-summary.md`) created; **all GitHub Actions pinned to commit SHA**; stakeholder approval obtained. |
| **Prerequisites** | App plan exists (Assignment 2). |
| **Dependencies** | Assignment 2 (plan references the structure). |
| **Project-Specific Notes** | This implements **T-0.1 (Repository Bootstrap), T-0.2 (Aspire AppHost Skeleton), T-0.3 (ServiceDefaults)** from the dev plan. Concrete deliverables: `global.json` (pin SDK 8.0.x, rollForward `latestFeature`), `Directory.Packages.props` (Central Package Management with all §3 versions), `Directory.Build.props` (TreatWarningsAsErrors, nullable, implicit usings), `.editorconfig` (file-scoped namespaces, 4-space, CRLF), `GapMiner.sln` with `src`/`tests` solution folders, and all 9 src + 5 test `.csproj` files. AppHost must declare PostgreSQL (with pgvector) + Redis via Aspire hosting. Docker Compose for local dev must use `pgvector/pgvector:pg16` image (NOT plain `postgres:16`). CI workflow must pin every action by SHA with semver comment. `.ai-repository-summary.md` per `create-repository-summary` instructions. |
| **Risks/Challenges** | (a) **.NET SDK version mismatch:** devcontainer ships SDK 10 (per AGENTS.md tech stack), but plan pins 8.0.x. `global.json` rollForward policy must be configured so `dotnet` resolves correctly — verify `dotnet --list-sdks` and that rollForward `latestFeature`/`patch` works, otherwise SDK 8 must be installed. (b) pgvector Docker image correctness — wrong image breaks EF migrations later. (c) Central Package Management versions must exactly match §3; floating versions violate R2. (d) SHA-pinning every CI action requires looking up each release SHA (e.g., `actions/checkout`, `actions/setup-dotnet`) — time-consuming but mandatory. (e) Coexistence: the .NET solution sits alongside existing orchestrator infra (`.github/`, `.opencode/`, `scripts/`, `test/`) — must not collide. |
| **Events** | After completion → `validate-assignment-completion`, `report-progress`. |

---

### 3.5 Assignment 4: `create-agents-md-file`

| Field | Detail |
|---|---|
| **Short ID** | `create-agents-md-file` |
| **Goal** | Create/rewrite an app-specific `AGENTS.md` at repo root optimized for AI coding agents working on GapMiner (per the open AGENTS.md spec at agents.md). |
| **Key Acceptance Criteria** | `AGENTS.md` at repo root; project overview (purpose + tech stack); setup/build/test commands **verified to work**; code style/conventions; project structure/directory layout; testing instructions; PR/commit guidelines; standard Markdown; commands validated by running; committed + pushed; approval obtained. |
| **Prerequisites** | Repo initialized (Assignment 1); app plan exists (Assignment 2); project structure created (Assignment 3) so build/test commands actually exist to validate. |
| **Dependencies** | Assignment 3 (must have a buildable solution to validate commands). |
| **Project-Specific Notes** | A root `AGENTS.md` already exists (the orchestrator-service template version, ~hundreds of lines of orchestration instructions). This assignment should produce the **app-specific** GapMiner `AGENTS.md`. The commands to validate/document: `dotnet build GapMiner.sln`, `dotnet test`, `dotnet run --project src/GapMiner.AppHost`, plus lint/format tooling (StyleCop/SonarAnalyzer run via build). Conventions from dev plan §5 (naming: `GapMiner.{Layer}.{Feature}`, kebab-case routes, snake_case DB columns, `gapminer:{domain}:{action}` Redis keys). Cross-reference README.md and `.ai-repository-summary.md` — complement, don't duplicate. |
| **Risks/Challenges** | (a) **Existing AGENTS.md conflict:** the current file is orchestrator infrastructure. Decision needed: does the app-specific file fully replace it, or merge orchestration + app guidance? The assignment says "create" — but careful handling required so downstream orchestration (epic creation triggered by `orchestration:plan-approved`) still has its instructions. **Recommendation:** preserve orchestration-critical sections (workflow trigger context, validation protocol) and add a GapMiner-specific section, OR confirm the orchestrator reads its instructions from `.opencode/` not root AGENTS.md. (b) Commands must be run and verified — if the solution doesn't build yet (Assignment 3 issues), this step is blocked. |
| **Events** | After completion → `validate-assignment-completion`, `report-progress`. |

---

### 3.6 Assignment 5: `debrief-and-document`

| Field | Detail |
|---|---|
| **Short ID** | `debrief-and-document` |
| **Goal** | Produce a structured debrief report capturing learnings, deviations, action items, and an execution trace. Review with orchestrator; commit to repo. |
| **Key Acceptance Criteria** | Report follows structured template (executive summary, workflow overview table, deliverables, lessons learned, what worked, what to improve, errors, challenges, suggested changes, metrics, future recommendations); all deviations documented; report committed + pushed; execution trace saved at `debrief-and-document/trace.md`. |
| **Prerequisites** | Assignments 1–4 complete. |
| **Dependencies** | All prior assignments (debriefs their outcomes). |
| **Project-Specific Notes** | Must flag plan-impacting findings as **ACTION ITEMS** with recommendation: (a) file new issue, or (b) update later phase descriptions. Review upcoming phases (Phases 1–6 of dev plan) for continued validity given what was learned during scaffolding. Capture the SDK-version resolution (Open Question #1) and AGENTS.md-handling decision (Open Question #5) as documented decisions. |
| **Risks/Challenges** | (a) Temptation to be superficial — the debrief's value is in honest deviation/error capture. (b) If CI remediation consumed fix cycles in Assignment 6, that hasn't happened yet at this point — note that PR-merge outcomes feed a *later* update. |
| **Events** | After completion → `validate-assignment-completion`, `report-progress`. |

---

### 3.7 Assignment 6: `pr-approval-and-merge`

| Field | Detail |
|---|---|
| **Short ID** | `pr-approval-and-merge` |
| **Goal** | Merge the setup PR (opened in Assignment 1) to `main` through CI verification, code review, comment resolution, and post-merge hygiene. |
| **Inputs** | `$pr_num` — extracted from Assignment 1 (`#initiate-new-repository.init-existing-repository`). |
| **Key Acceptance Criteria** | CI checks pass (remediation loop up to 3 attempts); code review delegated to `code-reviewer` subagent (not self-review); auto-reviewer comments waited for; `ai-pr-comment-protocol.md` executed + logged; `pr-review-comments` criteria satisfied; GraphQL verification (all threads resolved); stakeholder approval; merge performed; source branch deleted; related issues closed; run report updated. |
| **Prerequisites** | PR exists (Assignment 1); all prior assignments committed to the branch. |
| **Dependencies** | Assignments 1–5 (all work committed to the setup branch before merge). |
| **Project-Specific Notes** | **Self-approval is acceptable** per the workflow (automated setup PR — no human stakeholder required). However, the CI remediation loop (Phase 0.5) MUST still run. Branch protection ruleset (imported in Assignment 1) requires status checks `scan` + `lint` to pass and requires 1 approving review with thread resolution — the orchestrator/PAT (OrganizationAdmin bypass) may need to handle the review requirement, or the ruleset's bypass actors apply. **Post-merge:** delete `dynamic-workflow-project-setup` branch, close setup-related issues. |
| **Risks/Challenges** | (a) CI (`validate.yml`) runs actionlint/gitleaks/markdownlint — new .NET files generally pass these, but any workflow YAML edits or README/AGENTS.md content could trip markdownlint or gitleaks. (b) Branch protection requires `lint` + `scan` checks — these are the validate.yml job names; ensure they're registered as required status checks. (c) If `dotnet build` isn't part of CI yet (validate.yml has no .NET job), the .NET scaffolding won't be CI-verified at merge — a gap to flag. (d) Code review delegation requires the `code-reviewer` subagent to be available in the orchestration session. (e) Copilot code review is enabled in the ruleset (`review_on_push: true`) — must wait for it. |
| **Events** | After completion → `validate-assignment-completion`, `report-progress`. |

---

### 3.8 Post-Script Event: Apply `orchestration:plan-approved`

| Field | Detail |
|---|---|
| **Goal** | After all assignments + post-assignment events complete, apply the `orchestration:plan-approved` label to the application plan issue (created in Assignment 2) to signal the plan is ready for epic creation. |
| **Action** | Locate the plan issue (`#initiate-new-repository.create-app-plan`), apply label `orchestration:plan-approved`. |
| **Dependencies** | All 6 main assignments + all 12 post-assignment events complete; setup PR merged. |
| **Notes** | This label triggers the `orchestration:plan-approved` clause in the orchestrator prompt, kicking off the next orchestration phase (epic breakdown). |

---

## 4. Sequencing (Dependency Flow)

```
┌─────────────────────────────────────────────────────────────────────┐
│  PRE-SCRIPT                                                         │
│                                                                     │
│  create-workflow-plan  ───────────────────►  (this document)        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  MAIN ASSIGNMENT 1                                                  │
│                                                                     │
│  init-existing-repository                                           │
│    ├── create branch dynamic-workflow-project-setup                 │
│    ├── import branch protection ruleset                             │
│    ├── create GitHub Project board                                  │
│    ├── import labels (.github/.labels.json)                         │
│    ├── rename devcontainer/workspace files                          │
│    └── open PR ──► $pr_num (carries to Assignment 6)               │
│                                                                     │
│  [post] validate-assignment-completion → report-progress            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  MAIN ASSIGNMENT 2                                                  │
│                                                                     │
│  create-app-plan                                                    │
│    [pre] gather-context                                             │
│    ├── synthesize plan_docs/development-plan.md → GitHub Issue      │
│    ├── create milestones (Phase 0–6)                                │
│    ├── produce plan_docs/tech-stack.md, plan_docs/architecture.md   │
│    └── link issue to project + milestone + labels                   │
│                                                                     │
│  [post] validate-assignment-completion → report-progress            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  MAIN ASSIGNMENT 3                                                  │
│                                                                     │
│  create-project-structure   (dev plan T-0.1, T-0.2, T-0.3)         │
│    ├── GapMiner.sln + Directory.*.props + global.json + .editorconfig│
│    ├── 9 src projects + 5 test projects                             │
│    ├── Aspire AppHost (Postgres+pgvector, Redis)                    │
│    ├── Docker Compose + Dockerfiles                                 │
│    ├── CI workflow (SHA-pinned actions)                             │
│    ├── README + docs/ + ADRs                                        │
│    └── .ai-repository-summary.md                                    │
│                                                                     │
│  [post] validate-assignment-completion → report-progress            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  MAIN ASSIGNMENT 4                                                  │
│                                                                     │
│  create-agents-md-file   (app-specific AGENTS.md)                  │
│    ├── validate build/test commands                                 │
│    ├── draft AGENTS.md (overview, setup, structure, conventions)    │
│    └── cross-reference README + repo summary                        │
│                                                                     │
│  [post] validate-assignment-completion → report-progress            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  MAIN ASSIGNMENT 5                                                  │
│                                                                     │
│  debrief-and-document                                               │
│    ├── structured debrief report                                    │
│    ├── action items (new issues / phase updates)                    │
│    └── debrief-and-document/trace.md                                │
│                                                                     │
│  [post] validate-assignment-completion → report-progress            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  MAIN ASSIGNMENT 6                                                  │
│                                                                     │
│  pr-approval-and-merge   ($pr_num from Assignment 1)              │
│    ├── Phase 0.5: CI remediation loop (≤3 attempts)                 │
│    ├── Phase 0.75: code-reviewer delegation + auto-reviewer wait    │
│    ├── Phase 1: resolve all review threads (pr-review-comments)     │
│    ├── Phase 2: approval (self-approval OK for setup PR)            │
│    ├── Phase 3: merge → delete branch → close issues                │
│    └── result = "merged" | "pending" | "failed"                     │
│                                                                     │
│  [post] validate-assignment-completion → report-progress            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  POST-SCRIPT                                                        │
│                                                                     │
│  apply orchestration:plan-approved label to plan issue              │
│    ──► triggers next orchestration phase (epic creation)            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Critical path:** Assignment 1 → 2 → 3 → 4 → 5 → 6 (strictly sequential;
each main assignment depends on the prior). The `$pr_num` hand-off from
Assignment 1 to Assignment 6 is the key cross-assignment dependency.

---

## 5. Open Questions

These ambiguities should be confirmed by the orchestrator/stakeholder before or during the
relevant assignment. Items marked **[BLOCKING]** must be resolved before proceeding; others
can be resolved inline.

| # | Question | Context | Affects | Severity |
|---|---|---|---|---|
| 1 | **.NET SDK version:** devcontainer ships SDK 10, but `development-plan.md` pins 8.0.x. Will `global.json` with `rollForward: latestFeature` resolve to SDK 8, or must SDK 8 be installed separately? | `create-project-structure` (T-0.1) | Assignment 3, 4 | **[BLOCKING]** — build cannot proceed without correct SDK |
| 2 | **Naming convention:** Should GitHub Project board and devcontainer `name` use the repo slug (`gap-miner-v2-november10`) or the product name (`GapMiner`)? | `init-existing-repository` step 3, 5 | Assignment 1 | Low — use repo slug for GitHub assets, `GapMiner` for .NET namespace (already implied by dev plan) |
| 3 | **App-plan duplication:** `development-plan.md` is already a complete task-level spec. Should the `create-app-plan` issue re-state it or serve as a summary/index pointing to the dev plan? | `create-app-plan` | Assignment 2 | Medium — recommend summary + reference to avoid drift |
| 4 | **CI scope at merge:** `validate.yml` runs actionlint/gitleaks/markdownlint but has no `.NET build/test` job. Should a `dotnet build`/`dotnet test` check be added now, or deferred to epic implementation? | `pr-approval-and-merge` | Assignment 6 | Medium — scaffolding won't be build-verified by CI otherwise |
| 5 | **AGENTS.md replacement vs. merge:** A root `AGENTS.md` (orchestrator-service template) already exists. Does the app-specific file fully replace it, or should orchestration instructions be preserved? | `create-agents-md-file` | Assignment 4 | **[BLOCKING]** — replacing could break downstream orchestration context |
| 6 | **Branch protection review requirement:** Ruleset requires 1 approving review + thread resolution on `main`. For the self-approved setup PR, does the PAT's OrganizationAdmin bypass apply, or must a review be posted? | `pr-approval-and-merge` | Assignment 6 | Medium — verify bypass actors work or post a review |
| 7 | **Runtime secrets:** Apify token, Azure OpenAI/Claude key, embedding API key are needed for app runtime (Phase 2+) but NOT for scaffolding. Confirm these are deferred and not required during `project-setup`. | All | None (informational) | Low |

---

## 6. Approval

**Approver:** The Orchestrator (delegating agent) — this is an automated workflow, so the
orchestrator is the approver per the task instructions.

**Approval status:** ✅ **APPROVED**

This plan meets all acceptance criteria for the `create-workflow-plan` assignment:
- [x] Dynamic workflow file (`project-setup.md`) understood; all 6 main assignments + 3 event
      types (pre-script, post-assignment, post-script) traced.
- [x] All `plan_docs/` files read and summarized (§2 — development-plan.md and Strategic
      Feasibility doc).
- [x] Workflow execution plan covers every assignment in execution order (§3.1–§3.8).
- [x] Plan committed to `plan_docs/workflow-plan.md`.

The orchestrator may proceed to execute Main Assignment 1 (`init-existing-repository`).

---

*End of workflow execution plan.*
