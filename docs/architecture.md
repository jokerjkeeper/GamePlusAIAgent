**English** · [繁體中文](architecture.zh-Hant.md) · [简体中文](architecture.zh-Hans.md)

# System Architecture (De-identified)

> This document is an architecture snapshot of the AI agent system for a commercial Unity mobile-game project, with sensitive details removed.
> The whole solution is formed by three collaborating systems: **Game Source (the game client)**, **Game Agent (the knowledge-agent system, the subject this repo showcases)**, and **Task (a self-built ticketing system)**.

---

## ① The Three Systems & Responsibility Boundaries

| System | Role | Core Responsibility | Boundary & Decoupling |
|--------|------|---------------------|------------------------|
| **Game Source** (game client) | Data source | Produces daily code reviews (Tier1 facts) and specs; receives the deployed agent configuration | A read-only deployment target — it only feeds material, taking no part in curation |
| **Game Agent** (knowledge-agent system) | The system's brain | Curates data into a wiki knowledge base and performs requirement-feasibility analysis; maintains the agent config master template and deploys it with one command | Lives outside the game client; portable to the next project |
| **Task** (self-built ticketing system) | Collaboration hub | Persists and tracks tickets; loops completed tickets back to the Agent | Decoupled from the other two, interacting only via REST; contains no LLM itself |

```mermaid
flowchart LR
  classDef src fill:#ecfdf5,stroke:#10b981,color:#065f46;
  classDef agent fill:#eef2ff,stroke:#6366f1,color:#3730a3;
  classDef task fill:#fef3c7,stroke:#d97706,color:#92400e;

  PLAN["Game designer / producer"]
  SRC["Game Source (game client)<br/>Daily code review (Tier1) + specs"]:::src
  AGT["Game Agent (knowledge-agent system)<br/>Curation engine + wiki knowledge base<br/>Feasibility analysis · config master template"]:::agent
  TASK["Task (self-built ticketing system)<br/>Ticket persistence (REST + SQLite)"]:::task

  PLAN -->|"plain-language request"| AGT
  SRC -->|"import material (Tier1)"| AGT
  AGT -->|"feasibility analysis → file &amp; schedule tickets"| TASK
  TASK -->|"Tier2 specs / completed-ticket loop-back"| AGT
  AGT -.->|"one-command deploy of config template"| SRC
```

---

## ② Data-Flow Pipeline (produce → curate → requirement analysis → loop-back)

```mermaid
flowchart TD
  classDef up fill:#ecfdf5,stroke:#10b981,color:#065f46;
  classDef agent fill:#eef2ff,stroke:#6366f1,color:#3730a3;
  classDef plan stroke-dasharray:5 5,stroke:#d97706,fill:#fef3c7,color:#92400e;

  C["Daily git commit"]:::up --> CR["Daily code review (upstream skill)"]:::up
  CR -->|"(a) engineering retrospective report"| DOCS["Engineering report + fix list<br/>(upstream's own use, not into the knowledge base)"]:::up
  CR -->|"(b) knowledge material<br/>code facts + file:line, no evaluation"| RSRC["upstream codereview artifacts"]:::up
  WP["Task ticketing system"] -->|"spec / completed-ticket export"| TIK["raw/tickets"]:::plan
  RSRC -->|"import"| RAW["raw/codereview"]:::agent
  RAW --> CUR["/curate cleaning engine"]:::agent
  TIK --> CUR
  CUR --> STATE["(.state)<br/>manifest incremental · features arbitration · review_queue"]:::agent
  CUR --> WIKI["wiki/ topic pages + source pages"]:::agent
  WIKI -.->|"frontmatter"| BI["build_index.py"]:::agent
  BI --> IDX["INDEX (semantic routing table)"]:::agent
  IDX --> ST["/stopic semantic · multi-page recall"]:::agent
  WIKI --> ASK["/ask QA"]:::agent
  ST --> REQ["Game-designer requirement → which .cs gets touched<br/>→ feasibility / plan / create ticket"]:::agent
  ASK --> REQ
  REQ -->|"create / loop back"| WP
  CFG["config.json"]:::agent -.->|"paths / sources / tier"| CUR
  CFG -.-> BI
  CFG -.-> ST
```

**Responsibility split across the two ends**

| End | Role |
|-----|------|
| Game client · codereview artifacts | **code data**: what the code did + `file:line`, one file per day |
| Game client · engineering report + fix list | **bug / risk / evaluation**: upstream's own use, not into the knowledge base |
| Knowledge base · `wiki/` | **knowledge base**: split into "feature / code" layers, for the AI to load on demand for requirement analysis |

---

## ③ Directory Structure and Responsibilities

```text
agent/                            ← wiki startup directory = the knowledge base itself (Game Agent)
├ CLAUDE.md                       @AGENTS.md (Claude Code entry point)
├ AGENTS.md                       cross-tool instruction index → @.agent/spec/*
├ .agent/
│  ├ config.json                  ★paths / sources / authority tiers; every engine reads this file (no hardcoding)
│  ├ commands/                    engine commands: curate / ask / resolve / stopic
│  └ spec/  git.md  wiki.md       canonical specs (commit format / file split·feature)
├ .claude/
│  └ commands ──junction──▶ .agent/commands   (no elevation, lets Claude Code recognize the commands)
├ scripts/
│  └ build_index.py               ★INDEX auto-regeneration (from each topic page's frontmatter)
├ raw/        codereview/  tickets/             ingestion inbox
├ .state/     manifest / features / review_queue   incremental · arbitration · queue
└ wiki/       topic-*.md  source/  INDEX.md       curated knowledge pages + routing table
```

> In addition, Game Agent maintains an **agent config master template** (instructions / commands / specs / configuration), deployed to Game Source with one command by an installer; this is the key infrastructure behind "cross-project portability."

---

## ④ Cross-Tool Loading / Control Chain (cc · codex · opencode, all three covered)

```mermaid
flowchart LR
  classDef warn stroke-dasharray:4 4,stroke:#d97706,fill:#fef3c7,color:#92400e;
  CM["CLAUDE.md"] -->|"@AGENTS.md"| AG["AGENTS.md"]
  AG -->|"@import"| SPEC[".agent/spec/<br/>git.md · wiki.md"]
  AG -.->|"don't expand @ → installer inlines"| INL["codex / opencode<br/>instruction files (inlined spec)"]:::warn
  CC[".claude/commands"] -->|"junction (no admin)"| ACMD[".agent/commands/*.md"]
  CFG["config.json"] -.->|"paths / sources / tier"| ENG["engine commands<br/>curate · ask · resolve · stopic · build_index"]
```

**Each layer uses the right mechanism**

| Layer | Mechanism | Why |
|-------|-----------|-----|
| Instruction file | `AGENTS.md` is the single source of truth, `CLAUDE.md` `@import` | `@import` needs no administrator privileges; symlink requires elevation on Windows |
| Command | Windows junction | directory link, creating it needs no administrator privileges (only symlink does) |
| Compatibility | installer inlines the spec | codex / opencode don't expand `@import` |

---

## ⑤ Authority-Arbitration State Machine (.state core)

```mermaid
flowchart TD
  X["Same feature appears from multiple sources"] --> D1{"Is there a Tier1<br/>Code Review?"}
  D1 -->|"Yes"| OV["Higher tier overrides lower (regardless of date)<br/>overridden entry marked 'corrected by implementation / superseded by ticket'"]
  D1 -->|"No · same tier"| DT["Compare source dates"]
  CF["Code and ticket contradict<br/>with no Tier1 to adjudicate"] --> Q1["review_queue : conflict"]
  UV["Ticket spec lacks code corroboration<br/>needs implementation confirmation"] --> Q2["review_queue : unverified"]
  Q1 --> RS["/resolve human handling"]
  Q2 --> RS
```

- **Tiering**: `Tier1 Code Review (reads real code)` ＞ `Tier2 ticket spec`.
- **Hard rule**: authority tiers and finalized decisions are set by humans or rules; the **LLM never infers** on its own — it only executes.
- **Incremental**: relies on the manifest's `content_hash`; if nothing changed, skip — no repeated curation.

---

## ⑥ INDEX Maintenance Chain (drift-proof)

```mermaid
flowchart LR
  FM["topic-*.md frontmatter<br/>aliases · features · summary · authority"] -->|"build_index.py"| IDX["INDEX (semantic routing table)"]
  IDX -->|"LLM semantic interpretation"| ST["/stopic multi-page recall"]
  ST --> EX["'Skill Simulator' →<br/>editor + TestRunner + battlefield + GM"]
  TRG["regeneration triggers: page change / /curate wrap-up / manual / --check drift-proof"] -.-> FM
```

INDEX **must not be hand-written**: it is regenerated from each page's frontmatter, and CI `--check` returns `exit 1` the moment it goes stale.
