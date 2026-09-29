# System context: loading, refresh and the framework lock

Part of the [STOV GUI tree performance handover](../gui-tree-performance-handover.md).

---

## 2. System context

### 2.1 The loading path (expanding a node)

| Step | Where | Thread |
|---|---|---|
| 1 | `DynamicTreeListener.MyTreeExpansionListener.treeExpanded` sees the `PlaceHolderIsLoadingNodeAdapter` child and calls `startWorker` | EDT |
| 2 | `startWorker` raises a glass pane (`DisableGlassPaneTaskPresenter` as the finally task) and queues two tasks | EDT |
| 3 | `CreateTreeNodeWorkerTask.runMe` → `factory.createChildren(adapter)` | worker |
| 4 | `AbstractTreeNodeFactory.createChildren` → `adapter.getCachedChildren()` (the DB read, via `CellGroupNodeFactory$CellGroupFactory.createChildren`), then allocates **two** `DefaultMutableTreeNode`s per child — the child and its own "is loading" placeholder | worker |
| 5 | `ShowTreeNodesTask.runMe` inserts every child and fires one `nodeStructureChanged` | EDT |

Step 5 is what makes `VariableHeightLayoutCache` build a row entry per cell — the
~54,000 `TreeStateNode`s seen in the heap dump.

### 2.2 The refresh path

`DynamicTreeListener.refresh` splits into `RefreshTreeWorkerTask` (off-EDT:
`prepareForSync`, which does `invalidate(true)`, re-reads children, pre-warms name
and icon) and `RefreshTreePresentationTask` (on-EDT model mutation), passing state
through a `RefreshData` holder.

Refresh after a global edit is queued by `GlobalEditCloseWorkerTask` on **dialog
close**, not on Update. Under 50 cells it uses targeted events; at 50 or more it
bumps the global epoch and queues `RefreshCacheTask`.

### 2.3 Costs that are not obvious from reading the source

These three shaped almost every decision below.

- **A getter on a persistent object is not free.** IntelliBO enhances field access,
  so every read routes through the instance manager to check whether the field is
  loaded. `CellGroupImpl.getContextIdentifier()` measured **4.228 seconds of pure
  CPU** in one trace. That is the cache *hit* path. Any per-node loop touching one
  is expensive by construction.
- **`invalidate(true)` is a database operation.** It calls
  `AbstractPersistentObject.refreshPersistentObject()` → `PersistenceHelper.refresh`
  → `pm.refresh(o)`, and IntelliBO releases the connection afterwards, which
  **commits**. One trace: `loadFields` 456ms, of which `PhysicalConnection.commit`
  is 278ms sitting in a socket read. So it costs a read round trip *and* a commit
  round trip, per object.
- **Group nodes pay a third round trip.** `DomainObjectGroupNodeAdapter.getName()`
  embeds the cell count in the label, and `CellGroupImpl.refreshPersistentObject()`
  sets `cellCountDirty = true`. So `invalidate(true)` followed by `getCachedName()`
  re-runs `determineCellCount()` → `wms.getCellCount(this)`.

### 2.4 The global framework lock

`ApplicationContainer.addNewGuiFrameworkTasks` acquires a global **fair write lock**
(`SWING_FRAMEWORK_LOCK`) for the whole duration of an agenda. Consequences seen
directly in the traces:

- `pool-1-thread-5`: 17.907s alive, **50ms of CPU**, 17.853s blocked on
  `ReentrantReadWriteLock$WriteLock.lock`.
- `pool-1-thread-3`: 16.327s alive, 848ms of CPU, **15.479s (94.8%)** blocked on the
  same lock.

So one slow EDT task doesn't just freeze the tree — it starves every other worker in
the application for as long as it runs. This is why "the whole GUI hangs" rather
than just the tree, and why coalescing scroll refreshes (§4.3) mattered.

