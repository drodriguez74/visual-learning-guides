# Visual Learning Guides

Single-file HTML visual explainers (self-contained CSS/JS, dark/light theme toggle, SVG diagrams) plus one markdown implementation plan. `index.html` is the gallery; update it and `README.md` whenever a guide is added. Files ending `.ignore.html` are drafts — never list them in the index.

## QA Evergreen ecosystem (the active workstream)

Four related documents describe an AI-accelerated QA platform for enterprise CMS-style financial workflow apps (money movement, fraud, disputes, chargebacks). Keep the public docs unbranded — no employer names, internal URLs, or internal system identifiers; use `example.com` in samples:

- `qa-evergreen-implementation-plan.md` — **source of truth** (v1.2). Sections 18–20 hold the decisions log, agent-browser integration, and journey discovery design.
- `qa-evergreen-ecosystem.html` — system-design explainer (10 rules, 5 layers). Mirrors the plan; keep them in sync when editing either.
- `qa-automation-roadmap.html` — single-repo 18-month view; defers to the ecosystem doc for enforcement.
- `qa-automation-gherkin-ai.html` — generic BDD teaching page; lightly references discovery.

The set has had **three multi-discipline hardening passes** (commits `ea7b8cb`, `5cf76a2`, `f67f796`) plus a discovery/CMS update. When asked to critique again, find NEW gaps — don't re-litigate settled invariants:

1. **Journeys are demonstrated, not imagined** — a journey enters `qa-breadcrumbs.yaml` only with a recorded agent-browser execution trace; the trace compiler freezes it into deterministic `.feature`/steps/POM with role+name locators (never session `@refs`). Undemonstrated journeys are excluded from the coverage denominator.
2. **`QA_TARGET_MODE`** (`review` | `dev-canary`) — an MR gate must test the MR's build; UI tests against shared dev are informational only.
3. **Rule 7** — role-scoped test accounts (agent-browser auth vault), synthetic-only data (PCI), per-pipeline entity namespacing.
4. **Drift-corroborated healing** — locator heals auto-merge only with a matching breadcrumb drift event AND a successful agent-browser intent replay; unexplained DOM changes stay red.
5. **Agent-browser runs in cron/async lanes only** — never the blocking MR path. Deterministic Playwright is the test of record; agents verify, heal, triage, discover.
6. **CMS apps: the config export is the spec** (`cms_config` parser strategy), with per-role crawls verifying the configured surface actually renders.
7. Anthropic key lives behind a platform gateway; data governance = zero-retention or Bedrock/Vertex routing.

## agent-browser CLI notes

`npm i -g agent-browser && agent-browser install` (Vercel Labs, Rust daemon + CDP, no Playwright dependency). Core loop: `open` → `snapshot -i` (a11y tree with `@eN` refs) → `click/fill @ref` → re-snapshot (refs go stale on any page change). The daemon's `default` session is shared machine-wide — always use `--session <name>` for isolation and close your own sessions when done. `chat` mode needs a Vercel AI Gateway key; for governed environments, prefer driving primitives from your own agent instead. Empirical findings from live testing (saucedemo, public sites) are recorded in plan §20.6.
