**English** · [繁體中文](design-notes.zh-Hant.md) · [简体中文](design-notes.zh-Hans.md)

# Knowledge Engine — Design Notes: Non-Obvious Trade-offs

> This document goes deep into the **knowledge engine** that underpins the whole system — the low-level expansion of the design ideas (the system layer) sketched in the README — with **illustrative** code snippets (all aliased, not real code).
> The point here is to record the design judgment as it stood — **why I chose this way** — rather than to serve as an operations manual.

---

## 1. Why Not RAG: pulling "structure" forward to write-time

The most obvious approach is to reach for RAG (vector retrieval): clean → chunk → embed → vector store → similarity retrieval. I tried exactly that on a predecessor system — a RAG pipeline for long-form text — but moving it onto game-development material turned out to be a poor fit, and I traced the trouble to three root causes:

| Ailment | RAG's problem | This system's fix |
|---------|---------------|-------------------|
| **Structure flattened** | A structured spec gets sliced into a few-hundred-word chunks and embedded into one blur, so you can't ask "which settings a given patch actually changed" | The LLM preserves structure **at curation time**, writing it into a readable wiki topic page |
| **Can't tell the final decision apart** | Once a discussion thread is chunked, the overturned interim opinions and the final decision read alike, and the LLM can't tell which one is authoritative | A knowledge page carries an explicit `status` (in discussion / final decision); the final decision is a flagged, authoritative record |
| **Source / decision is guesswork** | "Who decided this, and where does the asset live" can only be dredged up by vector similarity — unreliable | Curation writes source / time / authority into structured Frontmatter fields, read directly at query time |

The core trade-off: **take the heavy work of "structuring" and pull it forward from query-time to write-time**. You pay an extra cost once, at curation, to write knowledge into pages that are clearly structured, clearly authoritative, and traceable; from then on every retrieval is fast and accurate, no longer riding on the luck of vector similarity.

---

## 2. Single-File, Three-Layer Architecture: refusing "one copy for humans, one for AI"

Early in the project, I too considered maintaining two documents — one for humans to read, one for the AI to consume. But this approach always ends the same way: the two gradually fall out of sync, and eventually both lose their credibility.
So instead, I chose to layer everything the three readers need into a single file:

```markdown
---
# Machine layer: for AI recall and indexing
features: [Controller input/cancel-key stack-top detection]
system: UI
authority: tier1-codereview
sources: [raw/codereview/2026-XX-XX.md]
aliases: [controller, cancel key, GameOver]
---

## Feature: GameOver cancel key now detects whether the stack-top UI is still showing   ← Reading layer: for humans, no code mixed in
The cancel key's handling is changed to detect whether the stack-top UI is in fact still showing. [^cancel]

[^cancel]: `Foo/NewUIGameOver.cs:120-140` (illustrative).   ← Reference layer: human review + AI provenance
```

- The AI only needs to read the Frontmatter to complete semantic recall — no need to swallow the whole page.
- The game designer only needs to read the conversational bullets in the middle.
- When you need to trace the actual code change, follow the Footnote straight to the exact line.

One body of content, three perspectives, never forking.

---

## 3. Authority Arbitration: converging contradictions by "source credibility," not "time"

My first arbitration rule was "latest date overrides the old" — until it bit me:
a ticket spec from two months earlier overwrote a fact that had just been extracted from the real code that very day.
After that, I changed the first principle of arbitration to **source tier**, not date:

```
Tier1 = Code Review (reads the real code)   ← highest, represents fact
Tier2 = ticket spec (human-written intent)  ← lower, may be unimplemented / already changed

When the same Feature appears from multiple sources:
  ├ a Tier1 exists → higher Tier overrides lower (ignore date), the overridden entry is marked "corrected by implementation"
  ├ same Tier      → only then compare source dates
  └ code and ticket contradict with no Tier1 to adjudicate → move into review_queue to await human /resolve
```

Payoff: the knowledge base gains the ability to **de-contradict itself**, letting us concentrate human effort on the few core conflicts that genuinely need judgment.

### Hard rule: the LLM never infers

What makes it safe to automate arbitration is keeping **the judgment in the hands of rules, not in the LLM's guesswork**.
The authority tiers (Tier1 > Tier2), the final-decision `status`, and the overwrite conditions are all set explicitly by humans and rules; the LLM is only responsible for "executing per the rules" and "writing the structured content correctly" — it **never infers which source is more authoritative.**
This hard rule keeps the whole curation outcome predictable and auditable — if something goes wrong, it's the rule that's wrong, not the model "feeling differently" that day.

---

## 4. Cross-Tool Integration: a three-layer split forced out of me by Windows Symlink permissions

The goal was "one instruction source feeds all three: Claude Code / Codex / OpenCode."
I first reached for the most obvious choice, Symlinks — and the install script stalled on an administrator-privilege request the moment it ran on Windows. For a tool that's meant to be installed by other people, that's unacceptable.
Forced out by this pothole, I split the shared parts into three layers, by "the mechanism each layer needs":

```text
Instruction-file layer:  AGENTS.md as the single source of truth
                         CLAUDE.md imports it with a single line @AGENTS.md   (@import needs no elevation)

Command layer:           .claude/commands  ──junction──▶  .agent/commands
                         (Junction is a directory link, no administrator privileges to create; only Symlink needs them)

Compatibility layer:     Codex / OpenCode don't expand @import
                         → the install script writes the spec content "inline" into their respective instruction files
```

Key insight: **Junction ≠ Symlink.** Creating a Junction directory link needs no elevation, which happens to sidestep this pothole.

---

## 5. INDEX Drift-Proofing: turning the "routing table" into a derived artifact

The AI's semantic recall depends on an INDEX routing table. A hand-written index inevitably rots, and once it rots, recall starts quietly going wrong.
The solution is to **make INDEX impossible to hand-write** — it's a derived artifact of each knowledge page's Frontmatter:

```bash
# Regenerate INDEX from the Frontmatter of all topic pages
python scripts/build_index.py

# CI: if INDEX is inconsistent with its sources, return exit 1 and block the PR
python scripts/build_index.py --check
```

Regeneration triggers: page change / `/curate` wrap-up / manual trigger / CI `--check`.
INDEX is always identical to the real state of the knowledge pages, eliminating "the document and the index tell different stories."

---

## 6. Incremental Dedup: gatekeeping with content_hash

When curation reruns daily, redoing unchanged content in bulk is both time-consuming and pollutes the diff.
The Manifest records each source's hash, and skips anything unchanged:

```text
for each raw source:
    h = sha(content)
    if manifest[source] == h:  skip          # unchanged, no re-curation
    else:                      re-curate + update manifest[source] = h
```

Curation thus turns incremental, processing only the parts that genuinely changed.

---

## Appendix: A "Don't Build It" Decision

I had originally planned to automatically convert the game-designer-side Excel configs into AI-readable documents, and had actually started building it.
But **testing showed an information-conversion rate of only about 30%** — the output was too noisy, and the maintenance cost outweighed the benefit. I decided to cut the whole thing,
keeping only the high-signal-to-noise "code-side" knowledge.

In an Agent system, **controlling the signal-to-noise ratio of what enters the knowledge base** often matters more than wiring up one more source.
