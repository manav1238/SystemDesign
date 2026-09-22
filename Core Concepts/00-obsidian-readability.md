# Obsidian Readability — Current Format vs What We Can Do

A decision doc for formatting the 61 concept files so they render well in Obsidian.

---

## 1. What We Have Today (Current Format)

**Plain, portable GitHub-style markdown.** No Obsidian-specific syntax anywhere.

| Property | Current state |
|----------|---------------|
| Concept files | 61 (sections 01–15; 04 & 13+ still `Planned`) |
| File structure | Identical 21-section template in every file |
| Section headings | `## 1.` … `## 21.` — same titles across all 61 files |
| Diagrams | Mermaid code blocks — 61/61 files (Obsidian renders these natively) |
| Tables | Standard GFM tables (terminology, trade-offs, failure modes) |
| Internal links | 325 markdown links `[Text](path.md)`; **0** Obsidian wikilinks |
| YAML frontmatter | 0 files have it (only `00-index.md` has a small header) |
| Tags (`#topic`) | 0 |
| Obsidian callouts (`> [!note]`) | 0 |
| Checkboxes / embeds / properties | 0 |

**The 21-section template (every file):**

| # | Section | Kind | Reader value |
|---|---------|------|--------------|
| 1 | One-Line Definition | prose | skim |
| 2 | Why Do We Need It? | prose | skim |
| 3 | Simple Intuition | prose/analogy | read |
| 4 | What Happens Without It? | prose | skim |
| 5 | Core Idea | lists | read |
| 6 | Important Terminology | table | reference |
| 7 | Basic Architecture | Mermaid | view |
| 8 | Request / Data Flow | numbered list | read |
| 9 | Practical Example | prose | read |
| 10 | Scaling | lists | read |
| 11 | Reliability & Failure Scenarios | table | reference |
| 12 | Consistency and Correctness | prose | advanced |
| 13 | Performance | prose | advanced |
| 14 | Security | prose | advanced |
| 15 | Trade-Offs | table | read |
| 16 | Common Mistakes | bullets | read |
| 17 | HLD vs LLD Boundary | prose | interview |
| 18 | Interview Questions | bullets (B/I/A) | self-test |
| 19 | Interview Memory Summary | bullets | revision |
| 20 | Related Concepts | links | navigate |
| 21 | References | prose | verify |

**Takeaway:** the content is solid and consistent; the *presentation* is vanilla markdown that Obsidian renders as flat text — nothing is visually prioritized, nothing is tracked in the graph, and nothing is queryable.

---

## 2. What Obsidian Can Do for This Content

### 2.1 Callouts — visually prioritize key sections

> [!info] One-Line Definition — section 1 → an info box at the top of every file

```markdown
> [!info] One-Line Definition
> The shard key is the column(s) a system uses to decide which shard owns a row.
```

> [!abstract] Interview Memory Summary — section 19 → a foldable recap box

```markdown
> [!abstract]- Interview Memory Summary
> - **30-second:** Evaluate a key against the four criteria…
> - **Trap:** "Shard by user_id" for a system whose core query is…
```

Other candidates: section 16 (Common Mistakes → `> [!warning]`), section 9 (Practical Example → `> [!example]`), section 21 (References → `> [!cite]`).

### 2.2 YAML frontmatter — properties, graph coloring, queries

```markdown
---
title: Shard Key
aliases: [Shard Key, Partition Key]
tags: [hld, database, sharding]
status: learning
priority: must-know
section: 09
related: ["[[sharding]]", "[[consistent-hashing]]"]
---
```

Unlocks:
- **Properties panel** at the top of each note (status, priority, section).
- **Graph view**: color nodes by `section` or `status`, filter to `priority: must-know`.
- **Search/Dataview**: e.g., a dynamic page listing all `must-know` files not yet `interview-ready`.

### 2.3 Wikilinks — backlinks + a real knowledge graph

Convert `[Sharding](sharding.md)` → `[[sharding]]` (or `[[sharding|Sharding]]`).

Unlocks the **backlinks panel** and makes the graph view meaningful — "load balancing" shows every concept that references it. This is where Obsidian's value over plain markdown is strongest.

Trades off portability: `[[wikilink]]` doesn't render on GitHub (needs the Obsidian URI/plugin to resolve). Files stay readable as text either way.

### 2.4 Tags — cluster the vault

`#hld`, `#database`, `#kafka`, `#security`, `#reliability`, `#interview-ready` etc. Give instant cross-file browsing in the tag pane and second filter axis in Dataview.

### 2.5 Foldable / collapsible content

`> [!abstract]- heading` fold the revision box; `> [!todo]-` for the interview-question checklist. Reduces scroll noise on long notes.

### 2.6 Embeds

`![[file]]` embed a diagram note inside another; useful later for a single "system design review" note that pulls in related concepts.

---

## 3. Before / After — One Section, Current vs Obsidian

**Current (renders flat):**

```markdown
## 1. One-Line Definition
The shard key is the column(s) a system uses to decide which shard owns a row.
```

**Obsidian-native (renders as colored box):**

```markdown
> [!info] One-Line Definition
> The shard key is the column(s) a system uses to decide which shard owns a row.
```

**Current (memory section buried mid-file):**

```markdown
## 19. Interview Memory Summary
- **Five key points:** …
- **30-second:** …
```

**Obsidian-native (foldable recap at bottom):**

```markdown
> [!abstract]- Interview Memory Summary
> **Five key points:** …
> **30-second:** …
```

**Current (plain link):** `[Sharding](sharding.md)`
**Obsidian-native:** `[[sharding|Sharding]]` → backlink + graph edge.

---

## 4. What Stays the Same

- **Mermaid diagrams** — already native to Obsidian (only had to fix broken syntax, done).
- **Tables** — Obsidian renders GFM tables natively; no change needed.
- **Numbered 21-section structure** — keep; it's consistent and aids scanning.
- **Plain markdown** still works everywhere; Obsidian simply enhances it.

---

## 5. Proposed Levels (choose ONE)

| Level | Changes | Cost / Risk |
|-------|---------|-------------|
| **L1 — Callouts + frontmatter** | Add YAML (title, tags, status, priority) to all 61; wrap sections 1 & 19 in callouts; add tags. Keep markdown links. | Low. Fully portable (markdown still valid). Universally renders fine even outside Obsidian (callouts are just blockquotes in plain markdown). |
| **L2 — L1 + wikilinks** | Also convert the 325 internal links to `[[wikilinks]]`. | Medium. Backlinks + graph light up; files stop rendering links on GitHub. |
| **L3 — Full restyle** | L2 + callouts for more sections (16, 9, 21), per-file breadcrumbs, badge system, foldable interview checklist. | Higher. Biggest visual change; most editing churn. |

---

## 6. Decision Point

Go with **L1** for instant readability at low risk. Add **L2** if you live in Obsidian and want the graph/backlinks. Skip L3 unless you want heavy visual styling.

After you pick, I'll: (1) transform one sample file first for your approval, then (2) batch-apply to all 61 via script, and (3) update `00-index.md` / `progress.md` accordingly.