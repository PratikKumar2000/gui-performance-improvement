# Progressive loading of wide groups

Part of the [STOV GUI tree performance handover](../gui-tree-performance-handover.md).

---

### 4.11 Progressive batches for wide groups

`WARM_LABELS` fixed the EDT database reads but introduced an all-or-nothing
presentation barrier: `ShowTreeNodesTask` received no children until every row was
warmed. Wide `Workspace`, `CellGroup` and `AlternativeGroup` expansions now use a
progressive path; all other nodes, deep expansion and `expandPath` retain the
original atomic path.

**Architecture.**

| Piece | Role |
|---|---|
| `ProgressiveTreeLoad` | Generation token and state holder: sorted child list, slice boundaries, displayed count, cancellation |
| `CreateTreeNodeWorkerTask(ProgressiveTreeLoad)` | Worker half: one `createChildren`, then warm-and-slice per batch |
| `ShowProgressiveTreeNodesTask` | EDT half: insert one slice, schedule the next agenda |
| `DynamicTreeListener` | Routes candidates, owns `progressiveLoads` map, cancel/restart lifecycle |

The first worker still calls `fac.createChildren` **once** per load — the adapter and
DAO currently return the complete sorted child list. It warms the first 50 rows; an
EDT task replaces the loading placeholder with that slice. Further agendas warm and
append 250 rows at a time until `WARM_LIMIT` (300 rows), then insert **unwarmed**
slices of 2,000 rows. Each continuation is a separate framework agenda: the global
fair write lock is released between slices. The progressive path does not raise the
main-frame wait cursor or glass pane.

The holder itself is the generation token. `DynamicTreeListener` keeps one active
`ProgressiveTreeLoad` per target node; an old worker can finish its query, but its
presentation task cannot publish unless that exact holder is still current. Collapse
resets a partial node to its loading placeholder. Filter/model changes cancel and
restart active loads. A presentation task also checks attachment, expansion and the
expected child count before inserting.

The source list is already sorted by `AbstractTreeNodeFactory.createChildren`, so
contiguous slices may be appended. The first slice uses `nodeStructureChanged` to
replace the placeholder. Later slices use one `nodesWereInserted` event for their
index range rather than remeasuring the entire accumulated subtree.

**Batch sizes (current).**

| Phase | Rows | Warming |
|---|---|---|
| First batch | 50 | yes |
| Batches while `nextIndex < 300` | 250 | yes |
| Remaining batches | 2,000 | no — renderer resolves on scroll |

**Important limits.**

- This improves the delay caused by warming all rows before first paint. It cannot
  show anything before `fac.createChildren` returns. Profiling (§4.15) showed that
  call alone can take 5+ seconds for 49,510 children.
- Rows past `WARM_LIMIT` are inserted without `getCachedName()`/`getCachedIcon()`.
  The renderer pays ~3 ms per row when scrolled into view. Acceptable trade-off;
  warming all rows would take ~160 s for a 49,510-row group.

---

### 4.12 Empty tree on workspace load (commit 15765)

**Symptom.** After the first progressive-load commit, the tree showed no nodes at
all — only the loading placeholder, or nothing visible.

**Cause.** `isProgressiveCandidate` initially included `Workspace`, so the invisible
root populated by `ChangeWorkspace` went through `startProgressiveLoad` instead of
the atomic `ShowTreeNodesTask` path. `ShowProgressiveTreeNodesTask` then checked
`tree.isExpanded(path)` on the root. `AbstractStovTree` calls `setRootVisible(false)`
in its constructor (line 75), so JTree does not report the hidden root as expanded.
The guard treated that as "user collapsed this node", cancelled the load, and nothing
was ever inserted.

**Fix.**

1. Re-enabled `Workspace` as a progressive candidate (the user wanted progressive
   loading for the workspace root too).
2. Treat the model root as always expanded in `ShowProgressiveTreeNodesTask` and
   `restartProgressiveLoads`: `expanded = isRoot || tree.isExpanded(...)`.
3. After the first batch's `nodeStructureChanged` on an invisible root, call
   `tree.expandPath(new TreePath(target.getPath()))`. `nodeStructureChanged` drops
   expansion state; without re-expanding, later `nodesWereInserted` batches land in a
   collapsed hidden root and stay invisible. The existing refresh path does the same
   at `DynamicTreeListener` line 883 (`if (node.isRoot()) getTree().expandPath(p)`).

**Where `setRootVisible(false)` is set:** `AbstractStovTree` constructor, line 75.
The root row is never drawn; only its children are visible. Swing still tracks an
expanded/collapsed flag for that invisible node.

---

### 4.13 `ArrayIndexOutOfBoundsException` on progressive insert (512 > 300)

**Symptom.** Expanding a wide group or scrolling during load threw
`java.lang.ArrayIndexOutOfBoundsException: 512 > 300` (or `256 > 230`) in
`VariableHeightLayoutCache.createNodeAt`, stack through
`FilteringTreeModelWrapper.fireTreeNodesInserted` and
`ShowProgressiveTreeNodesTask.runMe`.

**Cause.** `FilteringTreeModelWrapper.extractChildrenInfo` collected filtered child
indices in a `HashSet<Integer>` and called `toArray()`. For `Integer`, hash equals
value, so iteration order is bucket order — ascending only while all values stay below
the set's capacity. Inserting 250 children at positions 300–549 into a parent that
already had 300 children produced indices like `512, 513, …, 549, 300, 301, …`.
Swing requires `childIndices` ascending and aligned with `children[]`; the layout
cache then tried to create row 512 in a parent whose cache held 300 rows.

This was a **pre-existing bug** in the filter wrapper, latent until progressive
loading became the first code path to call `nodesWereInserted` with a large slice at
a high index. Refresh only ever inserted one or two nodes at low positions.

**Fix.** Replaced the `HashSet` with a `TreeMap<Integer, Object>` keyed by filtered
index, producing ascending aligned arrays. Also replaced the per-child `indexOf` scan
with a one-pass position map — the old code was O(batch × parentSize) per event.

---

### 4.14 Tab switch restarting progressive loads

**Symptom.** Clicking the cell compound or cell group tab restarted the progressive
load from the beginning — tree reset to placeholder, full reload.

**Cause.** `TreeViewFrame.TreeViewChangeTask` queues a `RefreshView` for **every**
cell group with a planning status when the cell group tab is selected. The original
`refresh()` implementation called `cancelAllProgressiveLoads(true)` at the start and
`RestartProgressiveLoadsTask` at the end, resetting every in-flight load to a
placeholder and restarting from batch zero.

**Fix.**

- Removed cancel-and-restart from `refresh()`. In-flight progressive loads are left
  running.
- Added `hasActiveProgressiveLoad(node)` guard at the top of `synchNode`: nodes with
  an active load are skipped by the refresh diff. Safe because the load re-reads
  through the factory; unsafe to diff underneath because batches are contiguous
  slices and `ShowProgressiveTreeNodesTask` checks `childCount == displayedCount`.
- Filter/model changes (`progressiveTreeModelChanged`) still cancel and restart —
  a filter change genuinely invalidates the captured child list.

**Not fixed (separate cost).** Tab switch still queues one refresh task per planning-
status cell group. That was pre-existing and scales with group count regardless of
which tab is visible.

---

### 4.15 Profiling: where the time actually goes

Timing logs added to `CreateTreeNodeWorkerTask` (debug level; use ERROR in dev if INFO
is filtered). Measured on a real workspace:

| Load | Children | `createChildren` | First warm batch |
|---|---|---|---|
| Cell compound root ("Standortplanung") | 49,510 | **5,070 ms** | 85 ms (50 rows) |
| Cell group root ("Standortplanung") | 4,207 | **1,595 ms** | 225 ms (50 rows) |
| Default alternatives ("Standardalternativen") | 69 | **720 ms** | — |
| Later warm batches | 250 rows each | — | **635–3,090 ms** (~3 ms/row) |

**Conclusions.**

1. **`createChildren` is the floor for time-to-first-row.** For the compound tree it
   is 5 s before any batch can start. That call runs `CellGroupDAOImpl.findGroupCells`,
   which issues **nine separate JDO queries** (one per concrete cell class) and
   materialises the full result. Batching cannot help until this is paged or merged.
2. **Per-row warming dominates total time after first paint.** At ~3 ms/row, warming
   49,510 rows is ~160 s of worker and database time. That is why `WARM_LIMIT = 300`
   and `COLD_BATCH_SIZE = 2000` were introduced: warm only what the user will see
   immediately; insert the rest as cheap structural nodes.
3. **Three trees load on every workspace change.** `TreeViewFrame` has four trees
   (`cellCompoundTree`, `groupTree`, `defaultCellTree`, `defaultAlternativeTree`).
   Each has its own `DynamicTreeListener` registered for `Aspects.WORKSPACE`, so one
   workspace change starts four root loads. The three `createChildren` log lines with
   different child counts (49,510 / 4,207 / 69) are three different trees, not the
   same node loaded twice. Within one load, `initialize()` throws if called twice.

**Duplicate log lines.** `src/conf/wiring/log4j.xml` attaches the STOV appender to
both `de.vodafone.rnp.stov` and `<root>` with default `additivity=true`, so every
event is written twice. Fix: `additivity="false"` on the named loggers. Same for
`de.vodafone.rnp.common`.

---

### 4.16 Producer/consumer queue (commit 15767)

Supersedes the wrap-all-then-slice behaviour in §4.11 for the two large trees.

**Scope.** Only `CellCompoundTree` and `CellGroupTree`, and only when the expanded
node is a `Workspace` or a `CellGroup`. Default-cell, default-alternative and
alternative-group expansions stay on the original atomic `createChildren` +
`ShowTreeNodesTask` agenda.

**Why.** `AbstractTreeNodeFactory.createChildren` wrapped every adapter as a
`DefaultMutableTreeNode` plus a loading placeholder before the EDT saw the first
row. For 49,510 compounds that allocation sat on top of the ~5 s JDO query. The
query still has to finish (`getCachedChildren()` is a full `find()`), but wrapping
and insert now overlap.

**Producer.** `CreateTreeNodeWorkerTask` in progressive mode:

1. Once: `object.getCachedChildren()` then `ProgressiveTreeLoad.startProducer(iterator)`.
2. `produceAhead(Consumer)` wraps up to `QUEUE_CAPACITY` (3) batches of 50 then 250
   and offers them on an in-memory queue. The consumer callback is `warmLabels`.
3. Representation adapters (`CellCompoundNodeAdapter`,
   `DomainObjectGroupNodeAdapter`) are skipped in `warmLabels` — labels are scalars
   already on the tree-representation row. Calling `getObject()` would lazy-load the
   real `CellCompound` / `CellGroup`. Other node types still resolve name/icon on the
   worker via `getPresentationObject()`.

**Consumer.** `ShowProgressiveTreeNodesTask` takes one batch per agenda, inserts,
then schedules the next produce+show pair if `hasMore()` (`queue` not empty or
iterator not exhausted). Empty queue with more to produce just reschedules the
worker.

**Generation token.** Unchanged: collapse / filter still cancel the
`ProgressiveTreeLoad` instance so a stale producer cannot publish.

**Honest limit.** First paint still waits for the full representation query. The
queue only overlaps **node wrapping + optional warm** with **EDT insert**. Cutting
the ~5 s floor needs a paged/iterator DAO.

**`loading.gif`.** Intended as the visible wait *before* that first query returns
(the hidden root currently shows only
`PlaceHolderIsLoadingNodeAdapter`, whose `getIcon()` is `null` and whose text is
`tree.isloading`). The GIF is **not** in this branch's `src` tree yet and is **not**
referenced from Java. Wiring it is the next presentation step: load it through the
existing `GuiUtils.createImageIcon` / `StovIcon` resource convention and return it
from the placeholder adapter (and/or overlay the tree viewport until `displayedCount > 0`).

*Superseded by §4.17: the overlay route was taken and the GIF was dropped.*

---

### 4.17 The loading wash

**What it is for.** A progressive load puts its first rows on screen within a
second or so, long before the group is complete. That is worse than an empty
tree: a handful of rows invites a click on the wrong one, and the selection is
then fought over by the rows arriving behind it. The tree is covered until it
holds enough rows to be worth using.

**Threshold.** `ProgressiveLoadController.OVERLAY_ROW_THRESHOLD = 50`.
`ShowProgressiveTreeNodesTask` calls `notifyProgressiveRowsDisplayed(load)` after
every batch; the cover is dropped the first time `getDisplayedCount()` reaches
50. A group smaller than 50 never reaches the threshold, so
`completeProgressiveLoad` drops the cover too — and so does
`cancelProgressiveLoad`, or a cancelled load would leave the tree covered
permanently.

**Claims, not a flag.** The root and a group can fill at the same time. Each
load holds a claim in `veiledLoads`, and the cover is only removed once the set
empties, so one finishing load cannot uncover another's tree.

**Where it lives.** `TreeLoadingOverlay` is added to the **root pane's layered
pane** at `PALETTE_LAYER`, sized to the tree's enclosing `JScrollPane`. Three
consequences, all deliberate:

- Not the frame's glass pane, which is owned by `CursorSelectorWindow` for the
  wait cursor. Taking it would have fought that, and would have blocked the tabs
  and detail panel, which stay usable while a tree fills.
- `PALETTE_LAYER` is above the content but below menus and popups, so an open
  dialog or combo list is not covered.
- A `ComponentListener` on the scroll pane keeps the bounds aligned when the
  window is resized or the split moved.

Input is swallowed by registering empty mouse, motion and wheel listeners: the
event stops at the topmost component that has one, so nothing needs to be
consumed explicitly.

**Appearance.** A flat translucent light grey, `new Color(217, 217, 217, 140)`,
painted over the whole area with `setOpaque(false)` so the tree stays legible
underneath. Two earlier attempts were discarded: an animated `loading.gif` on a
white veil, and a blur produced by downscaling a `BufferedImage` snapshot and
drawing it back up (later with a `ConvolveOp` box blur and a refresh `Timer`).
Both looked wrong — the blur in particular showed visible pixel blocks — and the
snapshot version also had to re-capture on a timer to avoid showing stale
content. The alpha constant is the only knob: raise it to hide more.

### 4.18 Splitting `DynamicTreeListener`

`DynamicTreeListener` had reached **1871 lines** and was by a wide margin the
largest file in the package. Two pieces were lifted out, chosen because they had
narrow interfaces and no access to the listener's own fields beyond `tree` and
`container`:

| New file | Lines | What moved |
|---|---|---|
| `ProgressiveLoadController` | ~285 | the `progressiveLoads` map, the `veiledLoads` claim set, `OVERLAY_ROW_THRESHOLD`, and every progressive method from `isProgressiveCandidate` through `progressiveTreeModelChanged` |
| `RefreshData` | ~135 | the former `private static final class RefreshData`, unchanged |

`DynamicTreeListener` is now 1553 lines. It keeps two thin delegating methods,
`isProgressiveCandidate` (still `protected`, so subclasses can opt out) and
`progressiveTreeModelChanged` (called by `AbstractStovTree`), and reaches the
controller directly for the other three call sites — `synchNode`, `setSource`
and `startWorker`.

`ShowProgressiveTreeNodesTask` now takes a `ProgressiveLoadController` instead of
a `DynamicTreeListener`, which is the only signature change outside the two new
files. `RefreshData` needed no modifier changes at all: its methods were already
package-private, and the refresh tasks that use it are still inner classes of
`DynamicTreeListener` in the same package.

This is a pure move — no behaviour was changed, and it has not been compiled or
run. It follows the `RefreshController` / `FilterController` pattern already
present in the package.
