# STOV GUI Tree — Loading and Refresh Performance Handover

**Companion to** [`global-edit-performance-handover.md`](./global-edit-performance-handover.md),
which covers the bulk-update and Atoll-sync work (Phases 1–13). That document's
**Phase 7** describes the viewport-limited refresh as *"design agreed, not wired"*.
It is now wired, and the work went considerably further than the original design.
This document supersedes Phase 7 and open items 6, 7 and 9 of that document.

**Branch:** `C:/pratik/Java/branches/STOV9102RC1_CommScope_Repeater_CR_And_SPL`
**Scope:** everything the user experiences as the GUI hanging — initial workspace
load, expanding a cell group or cell compound, scrolling a large tree, and the
refresh that follows a global edit. §4.8 is the exception: a correctness defect in
the refresh diff rather than a performance one, documented here because it was
introduced by the scroll-lag fix in §4.3.

---

## Contents

This was one file until it passed 1,100 lines. It is now an index plus eight
parts. **Section numbers are unchanged** — a cross-reference to §4.3 or §7 means
the same thing it always did, it just lives in a different file now. Use the
table to find which.

| Part | Sections | What is in it |
|---|---|---|
| [1. Measuring](./gui-tree-performance/01-measuring.md) | §1 | How to read the profiler, `v$session`, JFR and the round-trip counters — and the three ways they mislead you |
| [2. System context](./gui-tree-performance/02-architecture.md) | §2 | The loading path, the refresh path, the costs that are invisible in the source, and the global framework lock |
| [3. Refresh and scroll](./gui-tree-performance/03-refresh-and-scroll.md) | §4.1–4.10 | Viewport-limited refresh, scroll lag, the `findTreeNode` discovery, row prefetch, the duplicate-node defect, EDT database reads |
| [4. Progressive loading](./gui-tree-performance/04-progressive-loading.md) | §4.11–4.18 | Streaming wide groups, the producer/consumer queue, its four bugs, the loading wash, and the `DynamicTreeListener` split |
| [5. Look and feel](./gui-tree-performance/05-look-and-feel.md) | §11 | FlatLaf: whether it is safe, how it is wired, and the three adjustments it needed |
| [6. Changes and trade-offs](./gui-tree-performance/06-changes.md) | §5–6 | Every file touched, and what was knowingly given up |
| [7. Traps](./gui-tree-performance/07-traps.md) | §7 | Twenty-three things that will mislead the next person |
| [8. Open items](./gui-tree-performance/08-open-items.md) | §8 | What is left, in priority order, including what was ruled out |
| [9. Verification](./gui-tree-performance/09-verification.md) | §9–10 | The checklist, and the order to pick this up in |

**Start here if you are new to it:** part 1 (§1.1 in particular — the profiler's
number formatting has cost more time than any bug in this document), then the
timeline below, then part 8.

---

## 3. Measurement timeline

| Metric | Before | After | Note |
|---|---|---|---|
| Initial load, EDT presentation task | **13.298s** (100% CPU) | **1.791s** | §4.5 |
| — of which `TreeUtils.findTreeNode` | 13.163s (99.6%) | — | |
| — of which `synchNode` + Swing events | **103ms** (0.8%) | — | the real work |
| Initial load, worker `prepareForSync` | 3.114s | removed on full path | §4.4 |
| Worker `findTreeNode` | 1.280s | 10.333s* | *different scenario; §4.6 pending |
| Oracle `DB time` (whole session, cumulative) | **1.18s** | — | database is not the bottleneck |
| Oracle `hard parse elapsed time` | **0.07s** | — | rules out parse cost entirely |
| Global edit, 34K cells | 7m50s | — | from the other document |

The two numbers worth internalising: the database contributed **1.18 seconds** to a
session that produced twenty-second hangs, and the tree's own update work was **103
milliseconds** of a 13-second EDT task. Everything expensive was incidental.
