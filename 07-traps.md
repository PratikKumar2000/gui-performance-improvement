# Traps for whoever picks this up

Part of the [STOV GUI tree performance handover](../gui-tree-performance-handover.md).

---

## 7. Traps for whoever picks this up

1. **Profiler locale.** `19.912 ms` is nineteen seconds (§1.1).
2. **Profiler frame aggregation.** A frame's children are the union across all
   invocations along that path, not one call's behaviour. This is what made the
   first pruning guard a half-fix (§4.6).
3. **`INACTIVE` sessions are not idle operations** (§1.2).
4. **Confirm the SID.** IntelliBO holds several sessions; the tree's is not
   necessarily the one you found first (§1.4).
5. **`jdoProperties` silently drops unknown keys.** Anything not in
   `newPersistenceManagerFactory`'s whitelist is ignored without a warning.
6. **Spring 1.2.5 and 2.5 are both in `lib`.** The override is
   `newPersistenceManagerFactory(Map)`, which only exists in 2.5; against 1.2.5 the
   hook is `(Properties)` and your override would silently not be called, skipping
   every IntelliBO setting including the prefetch. `build.properties` pins
   `jar.spring = spring-2.5/spring.jar`, and the `@Override` wouldn't compile
   otherwise, so this is a packaging risk rather than a live bug.
7. **`cachelessPersistenceManagerFactory` is dead configuration** — defined in
   `persistence.xml`, referenced nowhere. Its name also reads backwards: it sets
   `useNTXCache` to `true`, and that flag lands on
   `getSharedCacheConfiguration().setEnabled(...)`. The live factory sets it
   `false`, so the shared cache is **off** on the PMF actually in use. Whether
   that's deliberate is unknown.
8. **`taskAgenda.addNext` inserts at the *beginning*.** Ordering reads backwards.
   This is why `RefreshCacheTask` had to be queued after `RefreshView` in source to
   run before it.
9. **A getter on a persistent object is not free** (§2.3). Don't assume a `getX()`
   in a per-node loop is free because it looks like a field read.
10. **`invalidate(true)` hits the database twice** — read plus commit — and three
    times for a group node, because the label embeds a cell count that
    `refreshPersistentObject` marks dirty.
11. **A wrong pruning predicate fails silently.** `canDescendantRepresentObject`
    returning false too eagerly means the node is never found and the refresh
    quietly does nothing — the same failure shape as the empty-`objectId` bug in the
    other document. Never widen it on inference alone.
12. **`FilteringTreeModelWrapper.invalidate()` is a no-op** in this build. An
    approach that depends on it will appear to work and do nothing.
13. **Trees on hidden tabs fire no viewport events.** Anything viewport-driven needs
    the `HierarchyListener`.
14. **The global framework lock serialises everything** (§2.4). A slow EDT task
    starves unrelated workers for its full duration, which is why symptoms present
    as "the whole application hangs".
15. **`AbstractTreeNodeAdapter` breaks the `equals`/`hashCode` contract.**
    `hashCode()` derives from `getObject()`; `equals()` compares
    `getPresentationObject()`. For a plain `DefaultTreeNodeAdapter` those are the
    same object and nothing breaks — which is why sectors behave. But
    `CellCompoundNodeAdapter` and the group adapters return a *tree representation*
    from `getPresentationObject()`, so two of their adapters can be `equals` while
    hashing differently. Never key a     hash-based collection on an adapter; key it on
    the node, which compares by identity. See open item 13 — this sits directly under
    the refresh diff.
16. **A getter that is in a fetch group is free; the one next to it may not be.**
    `IconMap` reads four fields off the same object and only one of them cost a round
    trip (§4.10). Reading the getter tells you nothing — check the `.jdo` fetch group
    the *query* asked for, which for the tree is `alternativeRedundancies`.
17. **The wait cursor appearing is not evidence of new slowness.** It is set inside
    `CreateTreeNodeWorkerTask.runMe`, and the EDT can only paint it while it is free.
    Work moved off the EDT becomes *visible* as a wait cursor where the same work on
    the EDT was an invisible freeze (§4.10).
18. **Warming on the worker is a barrier.** `warmLabels` resolves every child before
    the tree is given any of them, so partial results stop appearing. Progressive
    loading (§4.11) and `WARM_LIMIT` (§4.15) address this for wide groups.
19. **An invisible root is never `isExpanded`.** `setRootVisible(false)` means
    `JTree.isExpanded(rootPath)` is false even when children are showing. Any guard
    on expansion must treat `node == model.getRoot()` as expanded, and re-call
    `expandPath` after `nodeStructureChanged` on the root (§4.12).
20. **`FilteringTreeModelWrapper.extractChildrenInfo` must return ascending indices.**
    A `HashSet` of `Integer` iterates in bucket order, not value order. Progressive
    `nodesWereInserted` at high indices exposed this as `ArrayIndexOutOfBoundsException`
    in `VariableHeightLayoutCache` (§4.13).
21. **`refresh()` must not cancel in-flight progressive loads.** Tab switches queue
    many refresh events; cancelling resets every load to a placeholder. Skip
    `synchNode` for nodes with an active load instead (§4.14).
22. **Duplicate log lines are a log4j config issue, not duplicate work.**
    `additivity=true` on named loggers that share the root's STOV appender writes
    every event twice (§4.15).
23. **Three `createChildren` log lines on workspace change are three trees**, not one
    node loaded three times. Check the child counts — they differ per tree (§4.15).
