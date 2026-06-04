# QA Evergreen Ecosystem — Implementation Plan

**Organization:** Fiserv (Fortune 100)
**Approach:** One repo at a time · 50% coverage floor · AI-accelerated · Self-healing
**Stack:** TypeScript · Playwright · Cucumber-JS · Python (AI agents) · GitLab CI
**Prepared:** June 2025

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Prerequisites & Non-Negotiables](#3-prerequisites--non-negotiables)
4. [Team Structure & Ownership](#4-team-structure--ownership)
5. [Phase 1 — Foundation (Weeks 1–4)](#5-phase-1--foundation-weeks-14)
6. [Phase 2 — Pilot Repo (Weeks 5–10)](#6-phase-2--pilot-repo-weeks-510)
7. [Phase 3 — Scale to 3 Repos (Weeks 11–18)](#7-phase-3--scale-to-3-repos-weeks-1118)
8. [Phase 4 — Company Rollout (Weeks 19–36)](#8-phase-4--company-rollout-weeks-1936)
9. [The Breadcrumb System](#9-the-breadcrumb-system)
10. [The Generation Pipeline](#10-the-generation-pipeline)
11. [GitLab Enforcement Model](#11-gitlab-enforcement-model)
12. [Autonomy & Self-Healing](#12-autonomy--self-healing)
13. [Brownfield Onboarding Playbook](#13-brownfield-onboarding-playbook)
14. [KPIs & Success Metrics](#14-kpis--success-metrics)
15. [Risk Register](#15-risk-register)
16. [Definition of Done — Per Repo](#16-definition-of-done--per-repo)
17. [Tooling & Infrastructure](#17-tooling--infrastructure)
18. [Open Questions & Decisions Log](#18-open-questions--decisions-log)

---

## 1. Executive Summary

The QA Evergreen Ecosystem is a company-wide, AI-accelerated quality automation platform that transforms QA from a manual bottleneck into a self-sustaining, autonomous function embedded directly in the development lifecycle.

### The Problem

- Manual regression cycles consume 2–3 days per sprint across teams
- Test coverage is unmeasured, inconsistent, and undocumented
- Contractor turnover erases institutional QA knowledge
- Production defects are caught days after deployment, not minutes before merge
- There is no scalable, standardized QA practice across the portfolio

### The Solution

A five-layer ecosystem where:

- **Breadcrumb artifacts** capture the complete behavioral surface of every application (pages, roles, actions, API calls) as versioned YAML files committed to the codebase
- **AI agents** derive role-scoped user journeys from those breadcrumbs, prioritized by real Dynatrace session data
- **Generation agents** produce feature files, TypeScript/Playwright POM classes, Postman collections, and Pact contract tests automatically
- **A shared GitLab CI template** enforces the ecosystem across every repo with a single one-line include
- **Self-healing agents** keep tests current autonomously — AI fixes, AI reviews, human approves

### Success Criteria

| Horizon | Metric | Target |
|---|---|---|
| 90 days | Pilot repo coverage | 50% API + 50% UI |
| 6 months | Repos onboarded | 3–5 |
| 12 months | Repos onboarded | 10+ |
| 18 months | Portfolio coverage | 80%+ repos in ecosystem |
| Ongoing | Coverage floor | Never regresses below 50% |

---

## 2. Architecture Overview

The ecosystem is organized into five layers, each with a defined input, output, and owner. Layers communicate through versioned YAML artifacts — not runtime coupling.

```
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1: DETECTION                                             │
│  Extended workspace-analysis skill                              │
│  Input:  Source repos (frontend + backend)                      │
│  Output: .ai-sdlc/qa-profile.yaml                              │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│  LAYER 2: BREADCRUMB                                            │
│  breadcrumb_emitter.py — Journey artifact generator             │
│  Input:  qa-profile.yaml + Dynatrace RUM + OpenAPI spec        │
│  Output: .ai-sdlc/qa-breadcrumbs.yaml                         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│  LAYER 3: GENERATION                                            │
│  generator_agent.py — Test artifact factory                     │
│  Input:  qa-breadcrumbs.yaml                                   │
│  Output: Feature files, POM classes, Postman, Pact → QA repo   │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│  LAYER 4: ENFORCEMENT                                           │
│  qa-ecosystem.gitlab-ci.yml — Company-wide shared template      │
│  Input:  One-line include in any repo's .gitlab-ci.yml          │
│  Output: Pass / Block / Coverage report / Email notification    │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│  LAYER 5: AUTONOMY                                              │
│  Self-healing agents — Continuous maintenance                   │
│  Input:  Runtime failures, drift detection, coverage deltas     │
│  Output: Draft PRs with AI review → Human approval             │
└─────────────────────────────────────────────────────────────────┘
```

### Repository Structure

The ecosystem spans three repository types:

```
platform/qa-ecosystem          ← Shared GitLab CI template + scripts (GitLab Team owns)
platform/qa-toolkit            ← AI agents, breadcrumb emitter, generator (SDET Team owns)
{app-name}-qa                  ← Per-application QA repo (SDET Team owns, per app)
```

Source application repos (frontend + backend) are **read-only** from the ecosystem's perspective. The only modification to a source repo is adding one `include:` line to `.gitlab-ci.yml` and setting CI/CD variables.

---

## 3. Prerequisites & Non-Negotiables

These are hard requirements. If any are missing, the ecosystem cannot function for a given repo. The pipeline enforces all of them.

### Rule 1 — OpenAPI Spec (Hard Prerequisite)

Every Java backend **must** expose an OpenAPI 3.x spec before onboarding. If missing:

1. Add `springdoc-openapi-starter-webmvc-ui` to `pom.xml`
2. Run the app locally and verify `/v3/api-docs` is reachable
3. Commit the generated spec to `openapi/` in the repo
4. The `qa:openapi-check` pipeline job will verify this on every run

```xml
<!-- pom.xml addition -->
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.3.0</version>
</dependency>
```

**OpenAPI Annotation Quality Bar**

Auto-generated specs from unannotated Spring controllers produce schemas with missing descriptions, incorrect nullable flags, and `object` types that cause the generator to produce low-quality or incorrect test assertions. Before onboarding proceeds, the following minimum bar must be met:

- All endpoints have `@Operation(summary=)` annotations
- All request/response DTOs use typed Java classes (no raw `Object`, `Map<String,Object>`, or `JsonNode`)
- Nullable fields are explicitly marked (Spring validation annotations or OpenAPI `nullable: true`)

The `qa:openapi-check` job validates spec presence and this quality bar. A spec that passes presence but fails quality produces a `WARNING` in the pipeline output and creates a mandatory "OpenAPI annotation review" item in the onboarding Jira ticket. Onboarding cannot complete until this item is resolved.

### Rule 2 — Source Code Read Access

The `qa-toolkit` service account must have `read_repository` access to:

- Frontend React repo
- Backend Java/Spring repo
- Any shared component libraries used by the frontend

This is provisioned once per application, not per developer.

### Rule 3 — Dynatrace API Token

A read-only Dynatrace API token with `DataExport` and `ReadSyntheticData` scopes must be provisioned and stored as a GitLab CI/CD variable (`DYNATRACE_TOKEN`) in the application's GitLab group. If unavailable, set `QA_TELEMETRY_SOURCE=static` — the system falls back to static analysis, but journey prioritization will be less accurate.

### Rule 4 — GitLab Runner Compute

The following runner configurations are required:

| Job Type | Image | Minimum Runners |
|---|---|---|
| TypeScript tests (UI + API) | `mcr.microsoft.com/playwright:v1.44.0-jammy` | 6 concurrent |
| Python agents (AI + reporting) | `python:3.12-slim` | 2 concurrent |
| Regression (cron) | Both above | 8 concurrent |

### Rule 5 — Two Approvals Already Enforced

### Rule 6 — Secret Management

All tokens and API keys used by the ecosystem must be:

- Stored as **masked + protected** GitLab CI/CD variables at the **group level** (`platform/`), never at individual project level
- Named with the `QA_` prefix for namespace isolation
- Rotated every 90 days (a scheduled audit job in `platform/qa-ecosystem` alerts when tokens are approaching expiry)
- Never echoed to pipeline logs (GitLab's `masked` flag enforces this; the CI template explicitly uses `--no-verbose` flags on any command that could print token values)

The `qa-toolkit` service account uses purpose-scoped tokens:

| Scope | Access Level | Rationale |
|---|---|---|
| Source repos | `read_repository` only | No write, no API — git clone only |
| QA repos | `developer` role | Minimum to open draft MRs |
| GitLab API | `api` scope, `platform/` group only | Scoped to prevent lateral access |
| Jira | `write:jira-work` only | File tickets, no read of unrelated projects |

Confirmed: all MR merges require two human approvals. The ecosystem's QA coverage gate adds a third **non-human gate** — the pipeline must pass. This is enforced via GitLab branch protection rules (the `qa:coverage-gate` job must succeed before merge is permitted).

---

## 4. Team Structure & Ownership

### Core Team

| Role | Responsibility | Headcount |
|---|---|---|
| **QA Architect / Lead SDET** | Ecosystem design, AI agent quality, breadcrumb schema ownership | 1 |
| **Senior SDET** | QA repo development, POM library, step definition standards | 2 |
| **GitLab Engineer** | Shared CI template, runner config, branch protection rules | 1 (from dedicated GitLab team) |
| **DevOps Engineer** | Docker images, runner scaling, Dynatrace integration | 1 (part-time) |

### Extended Team (Per Repo Onboarding)

| Role | Responsibility |
|---|---|
| **App Dev Lead** | Approve one-line CI include, confirm OpenAPI spec, write feature files for new features (DoD) |
| **App QA Champion** | First reviewer of generated breadcrumbs and test artifacts for that app, ongoing test ownership |

### Ownership Matrix

| Artifact | Owner | Can Modify | Requires Approval From |
|---|---|---|---|
| `qa-ecosystem.gitlab-ci.yml` | GitLab Team | GitLab Team | QA Architect |
| `qa-profile.yaml` | Generated | No hand-editing | — |
| `qa-breadcrumbs.yaml` | Generated | SDET (corrections only) | QA Architect |
| Feature files (`*.feature`) | SDET / Dev | Anyone | 2 approvals (MR rule) |
| POM classes (`*Page.ts`) | SDET | SDET | 1 SDET approval |
| Step definitions (`*.steps.ts`) | SDET | SDET | 1 SDET approval |
| AI agent prompts | QA Architect | QA Architect | — |
| Coverage gate thresholds | GitLab Team | Per-repo CI variable | QA Architect |

---

## 5. Phase 1 — Foundation (Weeks 1–4)

**Goal:** Build the shared infrastructure before touching any application repo. Everything built here is reused for every future onboarding.

### Week 1 — Repository & Tooling Setup

**GitLab Team:**

- [ ] Create `platform/qa-ecosystem` repo with protected `main` branch
- [ ] Create `platform/qa-toolkit` repo with protected `main` branch
- [ ] Configure shared GitLab runners with Playwright and Python images
- [ ] Set up `platform/` GitLab group with shared CI/CD variables:
  - `QA_SCRIPTS` — path to shared Python scripts
  - `QA_ANTHROPIC_KEY` — Anthropic API key for AI agents
  - `QA_DEFAULT_COVERAGE_GATE` — `50`

**SDET Team:**

- [ ] Bootstrap `platform/qa-toolkit` with project structure:

```
qa-toolkit/
├── agents/
│   ├── breadcrumb_emitter.py      # Layer 2 — source → breadcrumbs
│   ├── journey_deriver.py         # Layer 2 — breadcrumbs → journeys
│   ├── generator_agent.py         # Layer 3 — breadcrumbs → test artifacts
│   ├── scope_detector.py          # Layer 4 — git diff → TEST_TAGS
│   ├── locator_healer.py          # Layer 5 — broken locator → patch PR
│   ├── drift_detector.py          # Layer 5 — breadcrumb diff → action
│   ├── coverage_audit.py          # Layer 5 — weekly gap analysis
│   ├── flakiness_quarantine.py    # Layer 5 — intermittent fail → @flaky
│   └── report_narrator.py         # Layer 4 — AI failure summary
├── parsers/
│   ├── react_route_parser.py      # Handles central / file-based / modular
│   ├── spring_role_extractor.py   # Reads @PreAuthorize + Redux auth slices
│   ├── openapi_mapper.py          # Maps OpenAPI → api_surface schema
│   └── dynatrace_prioritizer.py   # Fetches session data → priority scores
├── templates/
│   ├── feature.jinja2             # Gherkin feature file template
│   ├── steps.jinja2               # TypeScript step definition template
│   ├── page_object.jinja2         # Playwright POM class template
│   └── postman.jinja2             # Postman collection template
├── reporting/
│   ├── coverage_delta.py          # Δ coverage vs last run
│   ├── coverage_gate.py           # Pass/fail against threshold
│   └── email_sender.py            # jinja2 HTML + smtplib
├── schemas/
│   └── qa-breadcrumbs.schema.json # JSON Schema for breadcrumb validation
├── requirements.txt               # anthropic gitpython bs4 jinja2 pyyaml
└── tests/                         # Unit tests for the agents themselves
```

### Week 2 — Breadcrumb Emitter (Layer 2)

**Deliverables:**

- [ ] `react_route_parser.py` — detects routing strategy from `qa-profile.yaml` and parses accordingly. Supports three strategies:
  - `central_routes` — reads `src/routes.tsx` or `src/App.tsx`
  - `file_based` — traverses `src/pages/` directory structure (Next.js style)
  - `modular` — reads each feature module's route registration file
- [ ] `spring_role_extractor.py` — parses `@PreAuthorize` annotations from Spring controllers, maps to role identifiers. Cross-references Redux auth slice (`authSlice.ts`) for UI-side role guards
- [ ] `dynatrace_prioritizer.py` — queries Dynatrace Sessions API for page-level session counts over last 90 days. Returns priority map: `{route: "high"|"medium"|"low"}`
- [ ] `openapi_mapper.py` — reads OpenAPI YAML/JSON spec, extracts all endpoints grouped by tag/module with HTTP method and path
- [ ] `journey_deriver.py` — uses Anthropic Claude API to synthesize journey sequences from page graph + role permission map. Produces `journeys[]` entries in breadcrumb schema
- [ ] `breadcrumb_emitter.py` — orchestrates all parsers, writes `qa-breadcrumbs.yaml`, diffs against previous version and emits drift events

**Breadcrumb schema validation:** All generated breadcrumb files must validate against `qa-breadcrumbs.schema.json` before being committed. Invalid schema = pipeline fails.

### Week 3 — Generator Agent (Layer 3)

**Deliverables:**

- [ ] `feature.jinja2` template — produces valid Gherkin with `@role-{role}`, `@{module}`, `@{priority}` tags. Generates Background block for auth setup. Includes Scenario Outline with Examples table for parameterized cases
- [ ] `steps.jinja2` template — produces TypeScript step definitions with `async function(this: World)` pattern, correct `@cucumber/cucumber` imports, and TODO markers for SDET to complete
- [ ] `page_object.jinja2` template — produces Playwright `Page` class extending `BasePage`, with typed `Locator` properties using `getByRole`/`getByLabel`/`getByTestId`, and action methods
- [ ] `postman.jinja2` template — produces v2.1 Postman collection from OpenAPI spec with environment variables for base URL and auth token
- [ ] `generator_agent.py` — reads breadcrumbs, calls templates, calls Claude API for scenario wording and edge case suggestions, writes all artifacts to a staging directory, opens draft PR in target QA repo

### Week 4 — Shared CI Template (Layer 4)

**Deliverables:**

- [ ] `qa-ecosystem.gitlab-ci.yml` — complete shared template (see Section 11 for full spec)
- [ ] `check_openapi.py` — validates OpenAPI spec presence and parsability. Attempts auto-generation via springdoc if Spring Boot detected
- [ ] `check_breadcrumbs.py` — validates freshness: checks `openapi_hash` matches current spec, checks all React routes are represented, checks file is not older than last source commit
- [ ] `coverage_gate.py` — reads `allure-results/` JSON, computes coverage percentage, posts MR comment with delta, fails if below threshold
- [ ] Documentation: `README.md` in `platform/qa-ecosystem` with adoption instructions

---

## 6. Phase 2 — Pilot Repo (Weeks 5–10)

**Goal:** Prove the pattern on one production application. Achieve 50% coverage floor. Make the win visible and undeniable.

### Pilot Repo Selection Criteria

Select the pilot repo based on:

- React + Java/Spring Boot stack (confirmed)
- Has an OpenAPI spec or can generate one quickly
- Has an active Dynatrace application
- Dev team is willing to participate (designate a QA Champion)
- Moderate size — not the largest or most complex system in the portfolio
- Has clear role-based access (admin / user roles at minimum)

### Week 5 — Pilot Prerequisites

- [ ] Run `workspace-analysis` on both frontend and backend repos. Commit `qa-profile.yaml`
- [ ] Verify OpenAPI spec is present and accurate. Fix any missing endpoint documentation
- [ ] Provision Dynatrace API token for pilot application group
- [ ] Create `{pilot-app}-qa` repo. Bootstrap with Playwright + Cucumber-JS:

```bash
npm init -y
npm install --save-dev @playwright/test @cucumber/cucumber \
  allure-cucumberjs @cucumber/pretty-formatter \
  axios zod faker-js typescript ts-node \
  @types/node dotenv
npx playwright install chromium firefox webkit
```

- [ ] Confirm `platform/qa-ecosystem` include is accessible from pilot repo's GitLab group

### Week 6 — Breadcrumb Generation & Review

- [ ] Run `breadcrumb_emitter.py` against pilot source repos
- [ ] Review generated `qa-breadcrumbs.yaml` with SDET team:
  - Verify page inventory is complete (compare against app sitemap)
  - Verify roles map correctly (confirm with dev lead)
  - Review journey derivation — correct any illogical sequences
  - Confirm Dynatrace priority scores look reasonable
- [ ] Commit reviewed breadcrumbs to QA repo (not source repo)
- [ ] Document any corrections made — these feed back into emitter improvements

### Week 7 — Generate First Test Suite

- [ ] Run `generator_agent.py` against reviewed breadcrumbs
- [ ] AI code review runs automatically on all generated artifacts
- [ ] SDET reviews draft PR in QA repo:
  - Feature files: verify scenario wording is accurate and testable
  - POM classes: verify locators are correct (manual spot-check against app)
  - Step definitions: complete the TODO implementations
  - Postman collection: import and verify against live API
- [ ] Merge approved artifacts to QA repo `main`
- [ ] First manual test run to verify everything executes: `npx cucumber-js --tags "@smoke"`

### Week 8 — Pipeline Integration

- [ ] Add to pilot source repo `.gitlab-ci.yml`:

```yaml
include:
  - project: 'platform/qa-ecosystem'
    ref: main
    file: 'qa-ecosystem.gitlab-ci.yml'
```

- [ ] Set CI/CD variables in pilot repo:

```
QA_ECOSYSTEM_ENABLED    = true
QA_BROWNFIELD_MODE      = true      # soft mode during onboarding
QA_COVERAGE_GATE        = 50
QA_TELEMETRY_SOURCE     = dynatrace
QA_REPO_PATH            = /path/to/{pilot-app}-qa
QA_REPORT_EMAIL         = qa-team@fiserv.com,dev-lead@fiserv.com
DYNATRACE_TOKEN         = [provisioned token]
```

- [ ] Open a test MR and verify all QA pipeline jobs are visible and running
- [ ] Verify email report is delivered on first regression cron run
- [ ] Set up cron schedules in GitLab UI:
  - "Regression Tue" → `0 6 * * 2` → `main`
  - "Regression Thu" → `0 6 * * 4` → `main`

### Weeks 9–10 — Coverage Climb to 50% Floor

- [ ] Coverage audit agent runs — generates Jira tickets for uncovered modules
- [ ] SDET team backfills highest-priority gaps (guided by Dynatrace priority scores)
- [ ] AI generates draft feature files for uncovered journeys — SDET reviews and merges
- [ ] Validate self-healing: intentionally modify a locator in the app, observe healer detect and patch it
- [ ] Confirm coverage floor: `API ≥ 50%` and `UI ≥ 50%` in Allure report
- [ ] Set `QA_BROWNFIELD_MODE=false` — full enforcement now active
- [ ] **Pilot retrospective:** document what worked, what needed manual correction, estimated time savings
- [ ] Present results to leadership and next two candidate teams

---

## 7. Phase 3 — Scale to 3 Repos (Weeks 11–18)

**Goal:** Onboard two more repos in parallel. Validate that the pattern is repeatable without the founding team doing all the work.

### Weeks 11–12 — Repo Selection & Parallel Kickoff

- [ ] Select two additional repos using the same pilot criteria
- [ ] Designate a QA Champion on each app team
- [ ] QA Champions attend 2-hour onboarding workshop (created from pilot learnings):
  - What is a breadcrumb and why it matters
  - How to read and correct a generated feature file
  - Developer responsibility: write `.feature` for new features (DoD)
  - How to read the email report and act on it
- [ ] Run prerequisites check on both repos simultaneously

### Weeks 13–16 — Parallel Onboarding

Each repo follows the same 6-week brownfield playbook (Section 13). With two repos in flight:

- SDET A owns Repo 2 breadcrumb review and test approval
- SDET B owns Repo 3 breadcrumb review and test approval
- QA Architect reviews all AI agent outputs and breadcrumb corrections
- GitLab Engineer monitors runner load and scales if needed

### Weeks 17–18 — Ecosystem Improvements

After three repos are running, the pattern will have surfaced improvements:

- [ ] Refine `journey_deriver.py` prompt based on correction patterns from all three repos
- [ ] Add any new React routing patterns discovered to `react_route_parser.py`
- [ ] Update `qa-ecosystem.gitlab-ci.yml` to `v1.1.0` with any pipeline improvements
- [ ] Publish internal adoption guide: "How to onboard your repo in one sprint"
- [ ] Create self-service onboarding request process (GitLab issue template)

---

## 8. Phase 4 — Company Rollout (Weeks 19–36)

**Goal:** Scale to 10+ repos. Transition from founding team driving every onboarding to teams self-serving with light SDET support.

### Weeks 19–24 — Accelerated Onboarding (5 Repos)

- Target: 5 additional repos onboarded in 6 weeks
- Each onboarding takes 1 sprint with a trained QA Champion and the published guide
- SDET team provides office hours (2 hours/week per active onboarding) rather than hands-on ownership
- GitLab team monitors shared runner utilization and scales ahead of demand

### Weeks 25–36 — Self-Service Rollout

- Teams request onboarding via GitLab issue template
- QA Champions trained by existing QA Champions (train-the-trainer model)
- SDET team focuses on ecosystem improvements, not individual repo onboarding
- Monthly ecosystem health report: repos onboarded, average coverage, self-healing stats, ROI metrics

### Coverage Targets by Milestone

| Week | Repos in Ecosystem | Avg Coverage |
|---|---|---|
| 10 | 1 (pilot) | 50% |
| 18 | 3 | 55% |
| 24 | 8 | 60% |
| 36 | 15+ | 70%+ |

---

## 9. The Breadcrumb System

Breadcrumbs are the foundational artifact of the entire ecosystem. They are the machine-readable answer to: **"What does this application do, who can do it, and how important is it?"**

### What a Breadcrumb Captures

For every application module, the breadcrumb YAML records:

- **Pages** — every React route, the component that renders it, which roles can access it
- **Actions** — every meaningful user interaction on that page (filter, submit, navigate, export), the role required, and which API calls it triggers
- **Journeys** — named sequences of pages and actions that represent a coherent user goal, with role scope and priority level
- **API Surface** — every endpoint from the OpenAPI spec, grouped by module, with coverage status
- **Priority** — derived from Dynatrace session counts (or static heuristics as fallback)
- **Metadata** — generator version, generation timestamp, OpenAPI hash (for freshness detection)

### Freshness Enforcement

The breadcrumb file is considered stale when any of the following is true:

- The `openapi_hash` field does not match the current spec hash
- A React route exists in source that is not in the breadcrumb `pages[]` list
- The file was last generated before the most recent commit to `src/`

When stale, the `qa:breadcrumb-check` job fails with instructions to re-run the emitter. This ensures breadcrumbs never silently drift from the application.

### Breadcrumb Corrections

The emitter is AI-assisted but not perfect. SDETs may correct generated breadcrumbs. Corrections must be:

- Made in the breadcrumb file directly (it is a valid source file)
- Committed with a message starting with `fix(breadcrumbs):`
- Documented in the PR description so corrections feed back into emitter improvements

### Coverage Metric — Explicit Definition

"50% coverage" is used throughout this document. It means exactly:

- **UI Coverage %** = `(journeys with ≥1 passing automated scenario) ÷ (total journeys in breadcrumbs)`. This is journey coverage, not line coverage. A journey with 5 scenarios counts once — covered or not.
- **API Coverage %** = `(OpenAPI endpoints with ≥1 test that validates status code AND response schema via zod) ÷ (total endpoints in spec)`. A test that only checks `status 200` without schema validation is not counted. Pact consumer contracts only count once the provider runs `pact:verify` and the result is published to the Pact Broker — consumer-only contracts are flagged as `partial` in `coverage-manifest.json`.

These numbers come from `coverage_gate.py` reading `coverage-manifest.json` and `allure-results/`. They are the only numbers that count toward the 50% gate.

### Priority Levels and Their Effect

| Priority | Source | Coverage Target | Scenario Depth |
|---|---|---|---|
| `high` | Dynatrace top 25% by session count | 100% | All roles · Happy path · 3+ negatives · Edge cases |
| `medium` | Dynatrace 25–75th percentile | 60% | Primary roles · Happy path · 2 negatives |
| `low` | Dynatrace bottom 25% / static fallback | 20% | Smoke test only |
| `pilot` | Pages accessible only to `pilot` role | Isolated | Separate `@demo` suite |

---

## 10. The Generation Pipeline

The generator agent reads `qa-breadcrumbs.yaml` and produces the complete test artifact suite. All output goes to the target QA repo as a draft PR, never directly to `main`.

### Generated Artifacts Per Journey

For each journey entry in the breadcrumb file, the generator produces:

**1. Gherkin Feature File** (`features/{type}/{module}/{journey-id}.feature`)

- One file per journey
- Tags: `@role-{role}`, `@{module}`, `@{priority}`, `@smoke` or `@regression`
- Background block with authentication setup
- Primary scenario (happy path)
- Scenario Outlines for parameterized negative cases
- Separate scenarios for role-boundary checks (e.g., "user cannot access admin action")

**2. TypeScript Step Definition Scaffold** (`steps/{module}/{journey-id}.steps.ts`)

- Correct `@cucumber/cucumber` imports
- `async function(this: World)` pattern throughout
- All steps present with TODO markers for implementation
- Typed World context with page object references

**3. Playwright POM Class** (`pages/{ComponentName}Page.ts`)

- Extends `BasePage`
- One file per unique page component (deduplicated across journeys)
- Locators as typed `Locator` properties using semantic Playwright locators:
  - Preferred: `getByRole()`, `getByLabel()`, `getByPlaceholder()`
  - Acceptable: `getByTestId()` if `data-testid` attributes exist
  - Last resort: `locator('css')` — flagged with a comment for AI healer monitoring

**4. Postman Collection** (`postman/{module}-collection.json`)

- One collection per module
- All endpoints from OpenAPI spec for that module
- Environment variables for `{{base_url}}`, `{{auth_token}}`
- Pre-request scripts for auth token refresh
- Test assertions for status codes and response schema

**5. Pact Provider Verification (Required for API Coverage Credit)**

Generated consumer Pact contracts are not counted toward API coverage until the provider verifies them. The generation pipeline also produces:

- A `pact-provider-job.yml` snippet the Java backend team adds to their pipeline — a single copy-paste addition
- A `mvn pact:verify` call that publishes results to the shared Pact Broker (`platform/pact-broker`)
- A `can-i-deploy` check in the backend's pipeline that blocks merge if the provider verification fails

Until provider verification passes, endpoints are listed as `consumer-only: true` in `coverage-manifest.json` and do not count toward the 50% API floor. The coverage gate displays them as a separate `partial` bucket so gaps are visible without blocking onboarding.

**Context Window Management**

When a brownfield application has more than 150 journeys, the generator agent splits processing by module. Each Claude API call receives: (1) one module's breadcrumb entries, (2) the relevant OpenAPI endpoints for that module only, and (3) up to 3 existing feature files from that module as style examples. No single prompt exceeds 80k tokens. Module completion state is tracked in `coverage-manifest.json` so interrupted generation runs resume from the correct module rather than starting over.

**6. Coverage Manifest Update** (`.ai-sdlc/coverage-manifest.json`)

- Updated to reflect newly generated coverage
- Tracks: total journeys, covered journeys, coverage % per module and overall

### Developer-Written Tests (Greenfield)

For new features in greenfield repos, the developer is responsible for writing the `.feature` file as part of their Definition of Done. The generator scaffolds step definitions and POM from that feature file automatically. This means:

- Developer writes: `features/ui/{module}/{feature}.feature`
- Generator produces: matching `*.steps.ts` scaffold + POM class additions
- SDET reviews and completes implementations

This is enforced in the DoD — PRs without a corresponding `.feature` file for new user-facing features are blocked by the `qa:breadcrumb-check` job.

---

## 11. GitLab Enforcement Model

### Shared Template Structure

The `platform/qa-ecosystem` repo contains:

```
qa-ecosystem/
├── qa-ecosystem.gitlab-ci.yml     # The shared template (versioned)
├── CHANGELOG.md                   # Version history
├── scripts/
│   ├── check_openapi.py
│   ├── check_breadcrumbs.py
│   ├── scope_detector.py
│   ├── coverage_gate.py
│   ├── coverage_delta.py
│   └── email_sender.py
└── README.md                      # Adoption guide
```

### Pipeline Jobs Summary

| Job | Stage | Trigger | Blocks Merge | Image |
|---|---|---|---|---|
| `qa:openapi-check` | `.pre` | Always (if enabled) | Yes | `python:3.12-slim` |
| `qa:breadcrumb-check` | `.pre` | MR + schedule (if not brownfield) | Yes | `python:3.12-slim` |
| `qa:scope-detect` | `qa-analysis` | MR only | No | `python:3.12-slim` |
| `qa:run-feature-tests` | `qa-test` | MR only | Yes | `playwright:jammy` |
| `qa:regression` | `qa-test` | Schedule only | No | `playwright:jammy` |
| `qa:coverage-gate` | `qa-report` | Always (if enabled) | Yes | `python:3.12-slim` |
| `qa:report` | `qa-notify` | Always | No | `python:3.12-slim` |

### Feature Flag Variables

All behavior is controlled by per-repo CI/CD variables. No changes to the shared template are needed for per-repo customization.

| Variable | Default | Effect |
|---|---|---|
| `QA_ECOSYSTEM_ENABLED` | `false` | Master switch — must be explicitly set to `true` |
| `QA_BROWNFIELD_MODE` | `false` | Disables breadcrumb freshness hard gate (onboarding window only) |
| `QA_COVERAGE_GATE` | `50` | Minimum coverage % to pass the gate job |
| `QA_TELEMETRY_SOURCE` | `static` | Set to `dynatrace` to enable session-based prioritization |
| `QA_REPORT_EMAIL` | — | Comma-separated email list for regression reports |
| `QA_REPO_PATH` | — | Path to the application's QA repo on the runner |
| `QA_AUTOHEALER_AUTOMERGE` | `true` | Set to `false` to disable all auto-merging for regulated environments (SOX, PCI-DSS, HIPAA). All healer PRs then require explicit SDET approval regardless of confidence score. |

> **Empty `TEST_TAGS` guard:** If `scope_detector.py` finds no relevant changed modules (e.g., a docs-only commit or a new config file), `$TEST_TAGS` is set to `@smoke` rather than left empty. An empty `--tags` flag in Cucumber runs the entire suite, which is never the intent on an MR pipeline. The `scope_detector.py` script enforces this fallback explicitly.

### Branch Protection Integration

The `qa:coverage-gate` job is added to the list of **required status checks** on the `main` branch via GitLab branch protection rules. This means:

- The two required human approvals are unchanged
- The pipeline must also pass (including the coverage gate) before merge is available
- This is configured once per repo by the GitLab team during onboarding

---

## 12. Autonomy & Self-Healing

### Trust Levels by Change Type

| Event | System Action | Auto-Merge? | Approval Required |
|---|---|---|---|
| Locator fails (CSS/XPath stale) | Healer patches POM, opens draft PR, AI reviews | Yes — 24h if ≥ 90% confidence | SDET can reject |
| Component renamed | Regenerate POM + steps, open draft PR, AI reviews | No | SDET required |
| New page added | Generate feature + POM + steps, open draft PR, AI reviews | No | SDET required |
| Page removed | Flag tests as `@deprecated`, open draft PR | No | SDET required |
| New API endpoint | Generate API scenario, open draft PR, AI reviews | No | SDET required |
| Test flaky 3x | Auto-tag `@flaky`, move to quarantine suite, file Jira | Yes — immediate | No approval needed |
| Coverage drops below gate | Block pipeline, notify via email | N/A — hard gate | Fix coverage to unblock |
| Coverage audit gap found | Open Jira ticket with generated feature stubs | N/A | SDET picks up ticket |

### AI Code Review Format

Every draft PR opened by an autonomous agent includes an AI code review comment with:

```
### AI Code Review — Locator Healer

**Confidence:** 94%
**Change type:** Locator patch (low blast radius)
**Reason:** CSS selector `.btn-submit` no longer resolves on DisputeDetailPage.
  Found matching element via role attribute: `button[aria-label="Submit Resolution"]`

**Recommendation:** APPROVE — semantic locator is more resilient than the original CSS class.
  Consider adding `data-testid="submit-resolution"` to the component for future stability.

**Auto-merge scheduled:** If no objection, this PR will auto-merge in 24 hours.
  To cancel: add label `no-auto-merge` or close the PR.
```

For scenario-level changes, the AI review notes the business impact and explicitly states no auto-merge will occur.

### Confidence Score — Defined

The AI reviewer's confidence score is a weighted composite, not an arbitrary number:

| Weight | Factor | How Measured |
|---|---|---|
| 40% | Semantic locator equivalence | Does the new element have the same ARIA role + accessible name as the old one? |
| 30% | Post-patch execution outcome | Does the test pass when run against the live dev environment with the patched locator? |
| 30% | Blast radius classification | Locator-only change = 1.0 · Step logic change = 0.0 · Mixed = 0.5 |

**Threshold:** ≥ 90% composite score required for auto-merge eligibility. All scores are logged in `.ai-sdlc/locator-heal-log.json` with element fingerprints for audit and trend analysis.

### Circuit Breaker

If a single locator (identified by its `page + locator_id` key) is healed **3 or more times within a 30-day rolling window**, `locator_healer.py` stops auto-patching that locator and instead:
1. Opens a Jira escalation ticket referencing all 3 heal events and linking to the heal log
2. Tags the affected test `@locator-unstable` so it runs in the quarantine suite (non-blocking)
3. Notifies the SDET via the existing email report

Repeated healing at the same site is a signal that the component itself is unstable or uses an anti-pattern for locating — both require human judgment, not another automated patch.

### Regulated Environment Compliance

For environments subject to SOX, PCI-DSS, or HIPAA change management policies:

- Set `QA_AUTOHEALER_AUTOMERGE=false` at the **repo level** (not the job level — the shared template enforces this). All autonomous PRs require explicit SDET approval.
- The 24-hour auto-merge timer is automatically suppressed during declared GitLab deploy freezes (`$CI_DEPLOY_FREEZE` variable). No configuration needed — the healer checks this variable before scheduling auto-merge.
- All autonomous agent actions (PR creation, locator heal, flakiness tag, Jira ticket) are logged to a tamper-evident audit table in `platform/qa-toolkit` with actor, timestamp, confidence score, and outcome. This log satisfies change management audit requirements for locator-level changes.

### Flakiness Management

A test is quarantined when it fails intermittently in 3 or more pipeline runs within a rolling 14-day window. The flakiness quarantine agent:

1. Tags the test `@flaky` in the feature file (commits directly to a branch, opens PR)
2. Creates a Jira ticket with: test name, failure history, last 3 error messages, link to Allure reports
3. The test continues to run in the `@flaky` quarantine suite (separate job, non-blocking)
4. Once the Jira ticket is resolved and the test passes consistently for 5 runs, the `@flaky` tag is removed

---

## 13. Brownfield Onboarding Playbook

This playbook is executed for every existing production application being onboarded. It is designed to be completed in one 6-week sprint by one SDET with support from the app's QA Champion.

### Week 1 — Prerequisites

- [ ] Run `workspace-analysis` on frontend and backend repos. Review `qa-profile.yaml`
- [ ] Confirm OpenAPI spec exists. If not, add springdoc dependency and generate
- [ ] Provision Dynatrace token (or confirm `QA_TELEMETRY_SOURCE=static`)
- [ ] Create `{app-name}-qa` GitLab repo. Bootstrap with npm + Playwright + Cucumber-JS
- [ ] Confirm `platform/qa-ecosystem` is accessible from the app's GitLab group
- [ ] Schedule kickoff with app dev lead and QA Champion

### Week 2 — Breadcrumb Generation & Review

- [ ] Run `breadcrumb_emitter.py` — review output with QA Champion
- [ ] Verify page inventory: open the application, navigate to every section, confirm all routes are captured
- [ ] Verify role model: confirm with dev lead which roles exist and their permission boundaries
- [ ] Review top 10 journeys by Dynatrace priority — confirm they match real user workflows
- [ ] Commit reviewed breadcrumbs to QA repo

### Week 3 — Test Generation & SDET Review

- [ ] Run `generator_agent.py` — review draft PR
- [ ] Complete TODO implementations in step definitions (implement actual Playwright actions)
- [ ] Spot-check generated locators against live application — fix any that don't resolve
- [ ] Import Postman collection, run against dev environment, fix any failures
- [ ] Merge approved artifacts to QA repo `main`
- [ ] Run first smoke suite manually: `npx cucumber-js --tags "@smoke"`

### Week 4 — Pipeline Integration

- [ ] Add one-line include to source repo `.gitlab-ci.yml`
- [ ] Set all required CI/CD variables including `QA_BROWNFIELD_MODE=true`
- [ ] Set up cron schedules (Tue + Thu 06:00)
- [ ] Verify first pipeline run succeeds end-to-end
- [ ] Verify email report delivers correctly

### Weeks 5–6 — Coverage to 50% Floor

- [ ] Coverage audit agent generates gap tickets — SDET prioritizes and backfills
- [ ] AI generates draft feature files for high-priority uncovered journeys — SDET reviews
- [ ] First self-heal event observed and validated
- [ ] Coverage gate confirmed: `API ≥ 50%` and `UI ≥ 50%`
- [ ] Set `QA_BROWNFIELD_MODE=false`
- [ ] Retrospective held with dev team and QA Champion
- [ ] Lessons documented in `platform/qa-toolkit` issues for emitter improvements

### Important Notes

- The source application repo's `main` branch is **never modified** during onboarding except for the one-line CI include and CI/CD variable additions
- All generated test artifacts live in the QA repo, not the source repo
- `QA_BROWNFIELD_MODE=true` is the safety valve — it prevents hard gates from blocking the team while coverage is being established. It must be set to `false` before the onboarding is considered complete
- If Dynatrace data is unavailable for the pilot period, use `static` mode. The journeys will still be generated but prioritization will be based on route depth and component naming heuristics

---

## 14. KPIs & Success Metrics

All metrics are captured automatically from pipeline data and surfaced in:

- Allure reports (per-run)
- Email reports (per regression run — Tue + Thu)
- Monthly ecosystem health dashboard (executive-level)

### Per-Repo Metrics

| Metric | Description | Measurement | Target |
|---|---|---|---|
| API Coverage % | Endpoints with at least one automated test | `covered / total` from coverage-manifest.json | ≥ 50% floor, trending to 100% |
| UI Coverage % | User journeys with at least one automated scenario | `covered journeys / total journeys` | ≥ 50% floor, trending to 100% |
| Coverage Delta (Δ) | Week-over-week coverage change | Computed per regression run | Always ≥ 0 (never regresses) |
| Pass Rate | % of tests passing in latest run | `passed / total` (excluding quarantined) | ≥ 95% |
| Mean Time to Detect (MTTD) | Time from code push to defect detection | Pipeline duration for MR run | < 15 minutes |
| Flakiness Index | % of tests tagged @flaky | `flaky / total` | < 2% |
| Pipeline Duration — MR | Total time for feature-scoped MR run | CI duration log | < 15 minutes |
| Pipeline Duration — Regression | Total time for full regression run | CI duration log | < 25 minutes |

### Portfolio Metrics (Executive Dashboard)

| Metric | Description | Target |
|---|---|---|
| Repos in Ecosystem | Count of repos with `QA_ECOSYSTEM_ENABLED=true` and coverage ≥ 50% | +2 per sprint |
| Defect Escape Rate | % of production defects not caught by automation | < 5% by month 12 |
| Automation ROI | Hours of manual testing saved × $120/hour | Positive by month 4 |
| Self-Heal Events | Count of locator patches per month (shows AI is working) | Trending — no target |
| Tests Auto-Generated | Count of test artifacts created by AI agents vs by hand | Trending toward 80% AI-generated |

### ROI Calculation Model

```
Manual regression hours per sprint:          ~40 hours
Fully-loaded hourly rate (SDET):             $120/hour
Monthly manual testing cost:                 $9,600
Annual manual testing cost:                  $115,200

Ecosystem investment (one-time setup):       ~$60,000 (SDET time + tooling)
Ongoing cost (maintenance + scaling):        ~$2,000/month

Break-even point:                            Month 6
Year 1 net savings:                          ~$55,000 per repo in ecosystem
Year 2+ net savings per repo:                ~$115,000/year
```

---

## 15. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| OpenAPI specs are missing or inaccurate for many repos | High | High | Make spec generation a required first step. Provide springdoc setup guide. Block pipeline until resolved. |
| Dynatrace data is unavailable for some applications | Medium | Medium | `static` fallback mode built-in. Priority heuristics from component naming are good enough for initial coverage. |
| Generated breadcrumbs have significant inaccuracies | Medium | High | SDET review gate before any test generation runs. Corrections feed back into emitter improvements. |
| Developer resistance to writing `.feature` files as DoD | High | Medium | Make it easy — AI drafts the file, dev just reviews. Provide 10-minute workshop. The pipeline only blocks if the file is missing for new user-facing features. |
| GitLab runner capacity insufficient for scale | Medium | Medium | DevOps monitors runner utilization. Scale ahead of each onboarding wave. Playwright's speed means fewer runners needed than Selenium. |
| Self-healing creates incorrect patches | Low | High | 90% confidence threshold before auto-merge. 24-hour review window. SDET always has veto. Blast radius limited to locator-level changes only. |
| AI-generated scenarios miss business logic nuances | Medium | Medium | SDET review is mandatory for all generated feature files before merge. AI is a first draft, not a final product. |
| Contractor turnover erodes breadcrumb accuracy | Medium | High | Breadcrumbs are auto-regenerated on every significant source change. Human review is a checkpoint, not the primary maintenance mechanism. |
| Coverage gate blocks legitimate MRs during ramp-up | High | Medium | `QA_BROWNFIELD_MODE=true` during onboarding sprint. Gate only activates after SDET confirms 50% floor is achievable. |
| Ecosystem template changes break downstream repos | Low | High | Semantic versioning on `qa-ecosystem.gitlab-ci.yml`. Repos pin to a version tag, not `main`. Changes announced 2 weeks in advance via GitLab release notes. |
| Pact consumer contracts generated without provider verification inflate API coverage | High | Medium | `coverage-manifest.json` tracks `consumer-only: true` separately. Consumer-only endpoints are excluded from the 50% coverage gate. Provider job config is auto-generated and included in the QA onboarding PR for the backend team. |
| Auto-merged healer PRs violate SOX / PCI-DSS change management policy | Medium | High | `QA_AUTOHEALER_AUTOMERGE=false` disables all auto-merge. Required for regulated repos. Full audit log of all autonomous actions stored in `platform/qa-toolkit`. |
| OpenAPI specs are syntactically valid but semantically poor | High | Medium | `qa:openapi-check` enforces annotation quality bar (typed DTOs, `@Operation` annotations) before onboarding. Poor specs produce WARNING + mandatory Jira item before generation proceeds. |
| AI confidence scores are not independently validated after deployment | Medium | Medium | Confidence formula is documented (Section 12). False-positive rate reviewed monthly by QA Architect via `locator-heal-log.json` trend analysis. Threshold can be raised per-repo via `QA_HEAL_CONFIDENCE_THRESHOLD` CI variable. |
| Automated Jira ticket creation without triage owner creates noise and resentment | High | Medium | Coverage gap tickets are filed against the app team's board with a designated `QA-Champion` assignee set at onboarding time. Tickets include generated feature file stubs so the work is bounded. Weekly creation volume is capped at 5 tickets per repo — larger gaps are batched into one planning ticket. |

---

## 16. Definition of Done — Per Repo

A repo is considered fully onboarded into the QA Evergreen Ecosystem when **all** of the following are true:

- [ ] `workspace-analysis` skill has been run and `qa-profile.yaml` is committed
- [ ] `qa-breadcrumbs.yaml` is committed to the QA repo, reviewed by SDET, and passing schema validation
- [ ] QA repo (`{app}-qa`) is created with Playwright + Cucumber-JS bootstrap
- [ ] First test generation run completed — feature files, POM classes, and step definitions committed
- [ ] `platform/qa-ecosystem` include is active in source repo `.gitlab-ci.yml`
- [ ] All required CI/CD variables are set
- [ ] Cron schedules for Tuesday and Thursday regression are active
- [ ] Email report is delivering successfully to the distribution list
- [ ] `QA_BROWNFIELD_MODE=false` — full enforcement active (breadcrumb freshness gate is live)
- [ ] API coverage ≥ 50% confirmed in coverage-manifest.json
- [ ] UI coverage ≥ 50% confirmed in Allure report
- [ ] At least one self-healing locator event has been observed and validated
- [ ] Coverage gate is blocking pipeline (tested with a deliberate coverage regression)
- [ ] QA Champion on the app team has completed onboarding workshop
- [ ] Retrospective held — lessons documented in `platform/qa-toolkit` issues
- [ ] Next repo candidate has been nominated

---

## 17. Tooling & Infrastructure

### Language Stack Decision

| Layer | Language | Key Libraries | Rationale |
|---|---|---|---|
| Step Definitions | TypeScript | `@cucumber/cucumber`, `ts-node` | Existing team codebase. Strong typing catches POM errors at compile time. |
| POM Classes | TypeScript | `@playwright/test`, Locator API | Playwright is TypeScript-native. Semantic locators are more resilient than XPath. |
| API Test Clients | TypeScript | `axios`, `zod`, `faker-js` | Typed requests + schema validation in one ecosystem. |
| AI Agents | Python 3.12 | `anthropic`, `gitpython`, `bs4`, `jinja2` | Best AI/ML library ecosystem. All agent work is Python. |
| Reporting / Email | Python 3.12 | `jinja2`, `smtplib`, `allure-python` | 20-line email sender with HTML template. Simple and debuggable. |
| CI Glue | bash (minimal) | `echo`, `cp`, `if/else` only | Max 10 lines per job. Anything complex goes into Python. |
| ❌ Java / Selenium | Not used | — | Team is JS/TS. Playwright supersedes Selenium for new UI tests. |

### Docker Images

| Purpose | Image | Notes |
|---|---|---|
| UI + API tests | `mcr.microsoft.com/playwright:v1.44.0-jammy` | Node 20 + Chromium + Firefox + WebKit pre-installed |
| AI agents + reporting | `python:3.12-slim` | Minimal image. Dependencies installed per job from `requirements.txt` |
| Allure report generation | `python:3.12-slim` with `allure-commandline` | Installed via npm in the job |

### External Integrations

| Service | Purpose | Access Method |
|---|---|---|
| Dynatrace | Session-based journey prioritization | REST API — `DYNATRACE_TOKEN` CI variable |
| Splunk | Log-based failure correlation in AI reports | REST API — `SPLUNK_TOKEN` CI variable |
| Anthropic Claude API | AI agent intelligence (journey derivation, generation, review, narration) | `QA_ANTHROPIC_KEY` CI variable (group-level) |
| Jira | Auto-filing tickets for coverage gaps and flaky tests | REST API — `JIRA_TOKEN` CI variable |
| GitLab API | Opening draft PRs from autonomous agents | `CI_JOB_TOKEN` (built-in) |

---

## 18. Open Questions & Decisions Log

### Resolved Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Test framework language | TypeScript + Cucumber-JS | Existing team codebase. No migration cost. |
| UI test framework | Playwright (not Selenium) | Auto-wait, network interception, TypeScript-native, actively maintained. |
| API test library | axios + zod | Typed requests + schema validation in one ecosystem. |
| AI agent language | Python 3.12 | Best AI/ML library ecosystem for agent work. |
| CI/CD platform | GitLab (shared include template) | Dedicated GitLab team available. Two-approval MR rule already enforced. |
| Self-healing trust level | AI fixes → AI reviews → Human approves | Locator fixes auto-merge after 24h at ≥ 90% confidence. Scenario changes always require SDET. |
| Brownfield strategy | Feature-flagged parallel branch + separate QA repo | Zero disruption to source repo `main`. `QA_BROWNFIELD_MODE` controls gate enforcement. |
| Journey prioritization | Dynatrace-first, static-analysis fallback | Real telemetry beats assumption. Fallback ensures brownfield repos with no telemetry still work. |
| Coverage floor | 50% API + 50% UI | Proven PoC threshold. Undeniable enough to drive adoption. Path to 100% is automated. |

### Open Questions

| Question | Owner | Target Date |
|---|---|---|
| What is the Anthropic API rate limit for the volume of generation calls expected during a large brownfield scan? | QA Architect | Before Pilot Week 7 |
| Should the Postman collection be committed to the QA repo or the source repo? | QA Architect + Dev Lead | Before Pilot Week 7 |
| What Jira project should coverage gap tickets be filed against — the app's board or a central QA board? | QA Architect + PM | Before Pilot Week 9 |
| Should `qa-breadcrumbs.yaml` live in the source repo or the QA repo for greenfield projects? | QA Architect | Before Phase 3 |
| How do we handle applications with multiple frontends (e.g., admin portal + customer portal in same repo)? | QA Architect | Before Phase 3 |
| What is the process for deprecating the ecosystem from a repo that is being sunset? | GitLab Team | Before Phase 4 |

---

*Document maintained by QA Architect · Feedback via `platform/qa-toolkit` GitLab issues*
*Version 1.0 · June 2025*
