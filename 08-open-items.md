# Open items, prioritised

Part of the [STOV GUI tree performance handover](../gui-tree-performance-handover.md).

---

## 8. Open items, prioritised

### High

1. **Page `findGroupCells`.** Profiling (§4.15) shows `createChildren` is 5 s for
   49,510 children before any batch can start. `CellGroupDAOImpl.findGroupCells`
   runs nine JDO queries and materialises the full list. Progressive loading cannot
   beat this floor without DAO-level paging or a merged query. This is now the
   largest remaining win for time-to-first-row.
2. **Defer root load until tab is shown.** Four trees each load their root on every
   workspace change (§4.15). Only the visible tab's tree needs to populate
   immediately; the others can wait for `TreeViewFrame` tab selection or
   `HierarchyListener.SHOWING_CHANGED`.
3. **Measure after §4.6.** The compound-tree pruning guard is unquantified.
   Re-profile initial load and expansion.
4. **Index `findTreeNode`.** Pruning makes the search cheaper; an identity map from
   domain object to `DefaultMutableTreeNode` deletes it. Still outstanding.
5. **The cell group tree has no pruning at all.** See original item 3.
6. **Verify row prefetch took effect** via the round-trip delta in §1.4.
7. **Make the icon cheap on alternative rows.** Half-addressed in §4.10.
8. **Viewport-ahead warming for cold rows.** Rows past `WARM_LIMIT` lazy-load on
   scroll (~3 ms/row). Hook the existing viewport listener to warm the next screenful
   on the worker thread, so scroll stays smooth without warming all 49,000 rows.

### Medium

6. **Wire the interaction-time staleness check** (Phase 7 correction 3):
   `MyTreeSelectionListener.valueChanged` should verify staleness, so a user cannot
   act on a row deleted elsewhere. This is a **correctness** gap, not a performance
   one, and it is the outstanding half of the viewport design.
7. **Batch the leaf refresh.** Even limited to the viewport, ~30 leaves cost ~60
   round trips per scroll stop. `PersistenceHelper.refreshAll(o, collection)` exists
   and issues one `pm.refreshAll(col)`. Requires restructuring `prepareForSync` to
   collect objects before refreshing.
8. **The group label's cell count.** Refreshing a group node re-runs
   `wms.getCellCount(this)`. Consider not marking the count dirty on refresh, or
   folding it into the same read.
9. **First-touch pre-warm.** First expansion of a compound or a group is slow, later
   ones are fast. `JDOLifecycleManager.setUp()` has a commented-out
   `initializeAllPersistentCapableClasses()` whose body no longer exists — it can't
   have worked there, since `setUp()` runs before credentials are set. The hook
   belongs in `login()`, immediately before `isInitilized = InitState.FINISHED`, warming
   with binds that match nothing. Check whether `login()` runs on the EDT first.
   Oracle is ruled out as the cause: `hard parse elapsed time` was 0.07s.
10. **Trim the `nodesChanged` storm** in `synchNode` — fire only for indices whose
    cached label or icon changed, and drop the redundant trailing `nodeChanged`.
    Carried over from the other document. Note that after §4.5 the Swing event cost
    measured only ~99ms, so this is now much lower value than it looked.
11. **Drop the eager placeholder.** `AbstractTreeNodeFactory.createChildren`
    allocates a second `DefaultMutableTreeNode` per child purely so the row looks
    expandable — 68,000 nodes for a 34,000-cell group. Cells are never expanded.
    Needs care: `hasLoadedChildren` keys off the placeholder's presence.
12. **Deferred hydration.** `CreateTreeNodeWorkerTask` spent 848ms, of which
    `DomainObjectGroupNodeAdapter.getObject` was 783ms and the actual cell query
    only 64.4ms. Fetching identity plus label columns and hydrating the full domain
    object on selection would keep the model structure identical. Confined to
    `CellGroupFactory.createChildren` and the adapter's `getName`/`getIcon`.
13. **Fix the adapter `equals`/`hashCode` contract** (trap 15). Latent, but it sits
    directly under the refresh diff: `DiffUtils.removedObjects`/`addedObjects`
    operate on `HashSet<TreeNodeAdapter>`, and `RefreshData.oldChildren` is a
    `HashMap` keyed by adapter. For any adapter whose `getPresentationObject()`
    differs from `getObject()` — the compound and group adapters — a lookup can miss
    and an addition or removal can be computed wrongly. It did **not** cause the
    §4.8 duplicate, and sectors are unaffected, so this is prevention rather than a
    known defect. The fix is to derive both from the same accessor; check every
    override of `getPresentationObject()` before changing which one wins.

### Not viable

14. **True windowed loading** (`getChild(i)` on demand) was considered and rejected
    as non-contained. `getChildCount()` must be truthful — scrollbar extent, row
    indices and `getPathForRow` all derive from it, including the viewport code —
    so windowing needs a custom `TreeModel`, and `synchNode`'s diff,
    `TreeUtils.findTreeNode`, selection restoration and the scoped refresh all
    assume materialised `DefaultMutableTreeNode` children.
