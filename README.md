# Visual Learning Guides

A collection of interactive, single-file HTML visual references — built for deep study and quick review. No build step, no dependencies.

## Live Demo

View the index page at:
`https://drodriguez74.github.io/visual-learning-guides/`

Or jump directly to any guide:

| Guide | URL |
|-------|-----|
| **Index** | `https://drodriguez74.github.io/visual-learning-guides/` |
| **Context Engineering** | `https://drodriguez74.github.io/visual-learning-guides/context-engineering.html` |
| **HCI Fusion Model** | `https://drodriguez74.github.io/visual-learning-guides/hci-fusion-model.html` |
| **Vector Stores & Semantic Search** | `https://drodriguez74.github.io/visual-learning-guides/vector-stores.html` |
| **Lean Six Sigma** | `https://drodriguez74.github.io/visual-learning-guides/lean-six-sigma-guide.html` |
| **The Mainframe** | `https://drodriguez74.github.io/visual-learning-guides/mainframe-explainer.html` |
| **AI Harness Explainer** | `https://drodriguez74.github.io/visual-learning-guides/ai-harness-explainer.html` |
| **The Ralph Wiggum Loop** | `https://drodriguez74.github.io/visual-learning-guides/ralph-wiggum-loop.html` |

## Guides

### 01 · Context Engineering
`context-engineering.html` — Dark sci-fi aesthetic (Syne + IBM Plex Mono)

| Section | Contents |
|---------|----------|
| **Context Window** | Animated token bar showing SYS / MEM / TOOLS / HISTORY / RAG / USER segments |
| **Six Pillars** | System prompt, Memory, Tools, History, RAG, User query — with watermark numbers |
| **Pipeline** | 5-step context assembly flow: Receive → Fetch → Construct → Infer → Post-process |
| **Good vs. Bad** | Side-by-side comparison of poor vs. strong context engineering |

---

### 02 · HCI Fusion Model
`hci-fusion-model.html` — Dark animated maximalist (Bebas Neue + Outfit + JetBrains Mono)

| Section | Contents |
|---------|----------|
| **Hero** | SVG Venn diagram with animated orbiting dots and pulsing fusion zone |
| **Three Pillars** | Human (cognitive), Machine (computational), Interface (mediation) |
| **Fusion Loop** | Animated canvas loop: INTENT → SIGNAL → FUSION → OUTPUT → ADAPT → MEMORY |
| **Interaction Taxonomy** | 5 layers: Perception → Interpretation → Synthesis → Presentation → Learning |
| **Micro-Cycle** | 5-beat SVG arc diagram with curved bezier paths |
| **Cognitive Load** | Animated bar charts — what AI offloads vs. what humans retain |

---

### 03 · Vector Stores & Semantic Search
`vector-stores.html` — Dark editorial / newspaper (DM Serif Display + DM Mono)

| Section | Contents |
|---------|----------|
| **Embedding** | Word → model → vector diagram with 3 clickable newspaper-clipping example cards |
| **Vector Space** | Interactive canvas: 6 semantic clusters with hover tooltips and query dot |
| **Semantic Search** | Live demo with preset query chips and cosine similarity score bars |
| **Distance Metrics** | Cosine, Euclidean, Dot Product with inline geometric SVG diagrams |
| **RAG Pipeline** | 5-stage horizontal strip: Chunk → Embed → Store → Query → Generate |

---

### 04 · Lean Six Sigma
`lean-six-sigma-guide.html` — Dark (Bebas Neue + DM Sans + JetBrains Mono)

| Tab | Contents |
|-----|----------|
| **Overview** | Lean vs Six Sigma comparison table |
| **Lean Principles** | 5 core principles with key tools per phase |
| **8 Wastes** | TIMWOODS visual reference |
| **Six Sigma** | Sigma levels (1σ–6σ DPMO chart), Cp/Cpk, CTQ, variation types |
| **DMAIC** | Full Define → Measure → Analyze → Improve → Control breakdown |
| **Key Tools** | VSM, 5S, Kanban, Poka-Yoke, A3, Heijunka, SIPOC, Fishbone, Pareto, Control Charts, FMEA, 5 Whys, Gage R&R, DOE |
| **Belt Levels** | 3D flip cards: White → Yellow → Green → Black → Master Black Belt |
| **Interview Prep** | 11 Q&As with expandable answers + built-in flashcard quiz mode |

---

### 05 · The Mainframe
`mainframe-explainer.html` — Dark Retro Terminal (Orbitron + Share Tech Mono + VT323) · Green / Amber phosphor toggle

| Section | Contents |
|---------|----------|
| **System Architecture** | Hardware stack diagram — LPAR, z/OS, DASD, channel subsystem — with phosphor-glow styling |
| **Programming Languages** | COBOL, PL/I, Assembler, Java, Python on z/OS — history, use cases, and comparison |
| **Applications** | Industries still running mainframe workloads: banking, insurance, retail, government |
| **Middleware Ecosystem** | CICS, IMS, MQ, DB2, VSAM — roles and relationships |
| **Processing Pipeline** | JCL job lifecycle: Submit → Queue → Initiate → Execute → Output → Purge |
| **Relevance** | Why mainframes still run 68% of global transaction processing in 2024 |

---

### 06 · AI Harness Explainer
`ai-harness-explainer.html` — Dark (Syne + JetBrains Mono)

| Section | Contents |
|---------|----------|
| **What Is a Harness?** | Definition, analogy, and 3-zone anatomy diagram (Human Intent / Harness Bridge / AI Model + Context + Output) |
| **Anatomy — Layer by Layer** | 6-layer stack: System Prompt, Agent Persona, Skill/Workflow, Project Context, Tool Schemas, Conversation History |
| **Signal Flow** | Interactive 5-step animated walkthrough of a single `/review` invocation |
| **Three Harness Patterns** | CLI Harness, IDE Plugin Harness, Pipeline Harness — with real-world examples |
| **With vs. Without** | 6-row comparison table across Context, Consistency, Tool Access, Constraints, Portability, Multi-Provider |

---

### 07 · The Ralph Wiggum Loop
`ralph-wiggum-loop.html` — Dark · Hand-drawn / Notebook (Permanent Marker + Caveat + Special Elite)

| Section | Contents |
|---------|----------|
| **The Core Idea** | What the Ralph Wiggum Loop is and why it works — fresh context as a feature, not a bug |
| **The Loop — Visualised** | Animated diagram of the full agentic cycle |
| **The Context Window Secret** | Context rot vs. clean slate each iteration — why resetting wins |
| **What Happens Each Iteration** | 6-step breakdown: Load → Pick → Implement → Test → Commit → Repeat |
| **Why "Dumb" Wins** | Counter-intuitive argument for simplicity over elaborate orchestration |
| **Ralph Grows Up** | Gas Town — parallel multi-agent loops, Kubernetes for agents |

---

## Usage

No dependencies. No build step. Open the index:

```bash
open index.html
```

Or any guide directly:

```bash
open context-engineering.html
open hci-fusion-model.html
open vector-stores.html
open lean-six-sigma-guide.html
```

Or serve locally:

```bash
npx serve .
# then visit http://localhost:3000
```

Most guides support **light and dark mode** via a toggle in the top-right corner (default: dark). The Mainframe guide uses a **Green / Amber phosphor mode** toggle instead — switching between two retro terminal palettes that preserve the CRT aesthetic.

## Design

Each guide is a distinct aesthetic but shares a set of conventions:

- Single self-contained HTML file — CSS, JS, and fonts bundled inline or via Google Fonts CDN
- Dark mode default with light/dark toggle (Mainframe: Green / Amber phosphor toggle)
- Scroll-triggered reveals via `IntersectionObserver`
- Interactive elements: canvas visualizations, clickable cards, animated diagrams
- No frameworks, no dependencies, no build step

## Background

Built as personal study references for technical interviews and concept review. Covers AI systems (context engineering, vector stores), interface design (HCI Fusion), and process improvement (Lean Six Sigma).
