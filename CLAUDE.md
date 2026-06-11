# Visual Learning Guides

Single-file HTML visual explainers (self-contained CSS/JS, dark/light theme toggle, SVG diagrams) plus one markdown implementation plan. `index.html` is the gallery; update it and `README.md` whenever a guide is added. Files ending `.ignore.html` are drafts — never list them in the index.

## QA Evergreen ecosystem (the active workstream)

Four related documents describe an AI-accelerated QA platform for enterprise CMS-style financial workflow apps (money movement, fraud, disputes, chargebacks). Keep the public docs unbranded — no employer names, internal URLs, or internal system identifiers; use `example.com` in samples:

- `qa-evergreen-implementation-plan.md` — **source of truth** (v1.3). §21 is the reality check: nothing is built yet (no CI, no agents), Azure OpenAI provider decision, hostile-review backlog, and the POC scope. §18–20 hold the decisions log, agent-browser integration, and journey discovery design.
- `qa-evergreen-ecosystem.html` — system-design explainer (10 rules, 5 layers). Mirrors the plan; keep them in sync when editing either. **Still references Claude/Anthropic — pending a provider-sync pass to Azure OpenAI.**
- `qa-automation-roadmap.html` — single-repo 18-month view; defers to the ecosystem doc for enforcement. **Still references Claude/Anthropic — pending provider sync.**
- `qa-automation-gherkin-ai.html` — generic BDD teaching page; lightly references discovery.

**Current state (critical context):** this is target-state vision. As of v1.3 nothing is implemented — no `.gitlab-ci.yml`, no shared template, no cron/cleanup, and none of the AI agents exist. LLM provider is **Azure OpenAI (GPT-5.2)**, in-tenant (resolves most data-egress concerns). The real next step is the §21.5 proof-of-concept steel thread, with agents run locally/manually. Operational controls in §11–§19 only become risks once the layer that needs them is built.

The set has had **three multi-discipline hardening passes** (commits `ea7b8cb`, `5cf76a2`, `f67f796`), a discovery/CMS update, and a four-lens hostile review (GitLab/Security/app-lead/SRE) captured in §21.4. When asked to critique again, find NEW gaps — don't re-litigate settled invariants:

1. **Journeys are demonstrated, not imagined** — a journey enters `qa-breadcrumbs.yaml` only with a recorded agent-browser execution trace; the trace compiler freezes it into deterministic `.feature`/steps/POM with role+name locators (never session `@refs`). Undemonstrated journeys are excluded from the coverage denominator.
2. **`QA_TARGET_MODE`** (`review` | `dev-canary`) — an MR gate must test the MR's build; UI tests against shared dev are informational only.
3. **Rule 7** — role-scoped test accounts (agent-browser auth vault), synthetic-only data (PCI), per-pipeline entity namespacing.
4. **Drift-corroborated healing** — locator heals auto-merge only with a matching breadcrumb drift event AND a successful agent-browser intent replay; unexplained DOM changes stay red.
5. **Agent-browser runs in cron/async lanes only** — never the blocking MR path. Deterministic Playwright is the test of record; agents verify, heal, triage, discover.
6. **CMS apps: the config export is the spec** (`cms_config` parser strategy), with per-role crawls verifying the configured surface actually renders.
7. **Azure OpenAI (GPT-5.2)** behind the platform gateway (Azure AD auth, key in Key Vault, never a group GitLab var); in-tenant hosting resolves most data-egress concerns. Provider-agnostic via the gateway.
8. **Coverage denominator = demonstrated journeys only** (§9 authoritative; §20 conforms). Undemonstrated entries excluded until proven by a trace.

## agent-browser CLI notes

`npm i -g agent-browser && agent-browser install` (Vercel Labs, Rust daemon + CDP, no Playwright dependency). Core loop: `open` → `snapshot -i` (a11y tree with `@eN` refs) → `click/fill @ref` → re-snapshot (refs go stale on any page change). The daemon's `default` session is shared machine-wide — always use `--session <name>` for isolation and close your own sessions when done. `chat` mode needs a Vercel AI Gateway key; for governed environments, prefer driving primitives from your own agent (Azure OpenAI via the gateway) instead. Empirical findings from live testing (saucedemo, public sites) are recorded in plan §20.6.
