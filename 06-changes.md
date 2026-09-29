# Every change made, and the trade-offs accepted

Part of the [STOV GUI tree performance handover](../gui-tree-performance-handover.md).

---

## 5. Every change made

| File | Change |
|---|---|
| `presentation/widget/tree/DynamicTreeListener.java` | Viewport `ChangeListener` with debounce timer; `HierarchyListener` for hidden tabs; `ensureViewportListener` moved out of the conditional; `visibleAdapters` → `visibleNodes(boolean includeParents)` returning a `LinkedHashSet`; `adaptersOf` helper; `RefreshData` carries `scopedNodes` and `limitTo` stores both sets; `RECURSIVE_EXPANDED` iterates `scopedNodes` in both tasks; detached-node guard; `isRootScoped`; `walksWholeTree` scope split; `scrollRefreshInFlight` + `ScrollRefreshFinished`; `prepareLeafForSync` and its `isLimited()` guard; `synchNode` respects `isPrepared`; explicit root inclusion on the full path; `addAdapter` excludes null-object adapters; `RefreshData.synched` + `claimForSync` and the once-per-batch guard at the top of `synchNode`; `synchNode` writes the re-read children back via `putOldChildren` (§4.8); progressive path only for `CellCompoundTree`/`CellGroupTree` workspace and cell-group nodes (§4.16); owns cancellation/restart generations; root treated as always expanded; `refresh()` no longer cancels in-flight loads; `hasActiveProgressiveLoad` skips `synchNode` for loading nodes (§4.14) |
| `presentation/widget/tree/CreateTreeNodeWorkerTask.java` | `WARM_LABELS` and `warmLabels` (§4.9); progressive mode starts producer from `getCachedChildren` iterator and `produceAhead(warmLabels)` (§4.16); skips representation adapters so `getObject()` does not hydrate `CellCompound`/`CellGroup` |
| `presentation/widget/tree/ProgressiveTreeLoad.java` | Bounded producer queue (`QUEUE_CAPACITY` 3, batches 50/250); iterator wrapping instead of `createChildren` of the full list; optional `Consumer` warmer on each produced slice (§4.16) |
| `presentation/widget/tree/ShowProgressiveTreeNodesTask.java` | EDT insertion of one warmed or cold slice; active-token/attachment/expansion/model-count guards; `nodeStructureChanged` then `expandPath` for invisible root (§4.12); batched `nodesWereInserted` (§4.11) |
| `presentation/widget/tree/FilteringTreeModelWrapper.java` | `extractChildrenInfo` uses `TreeMap` for ascending indices and a position map instead of `HashSet` + `indexOf` (§4.13) |
| `presentation/widget/tree/AbstractStovTree.java` | Filter/model changes cancel, reset and restart active progressive loads through `DynamicTreeListener` (§4.11); `setRootVisible(false)` in constructor (§4.12) |
| `src/jdo/.../{GSM,UMTS,LTE,NewRadio5G}{Plain,Repeater}CellImpl.jdo` and `RemoteUnitRepeaterCellImpl.jdo` | `alternativeRedundancies.planningStatus` added to the `alternativeRedundancies` fetch group in all nine, so `IconMap` no longer triggers a lazy load per row (§4.10) |
| `presentation/widget/tree/adapter/DefaultTreeNodeAdapter.java` | `guiVersion` as a per-instance `volatile int`; `markRefreshed(int)`; `isStale()`; seeded at the current epoch in the constructor |
| `presentation/widget/tree/adapter/AbstractTreeNodeAdapter.java` | Null-safe `hashCode()` and `equals()` |
| `presentation/widget/tree/TreeUtils.java` | `findRecursiveChildTreeNode` takes a `debug` flag, resolved once per search; three `Log.debug` call sites guarded |
| `presentation/widget/tree/cellcompound/CellCompoundNodeAdapter.java` | `canDescendantRepresentObject` rewritten to resolve the presentation object and its string once; early return for `DomainObjectGroup` and `DomainObjectGroupTreeRepresentation` |
| `presentation/widget/dialog/.../GlobalEditCloseWorkerTask.java` | `REFRESH_VISIBLE_ON_CLOSE`; epoch bump instead of a whole-tree walk; `RefreshCacheTask` ordered **before** `RefreshView` |
| `common/service/persistence/LocalIBOPersistenceManagerFactoryBean.java` | `rowPrefetch` field, getter/setter, `applyRowPrefetch`, `java.util.Properties` import |
| `src/conf/wiring/subcontexts/persistence.xml` | `rowPrefetch` property on `persistenceManagerFactory` |
| `build.xml` | `oracle.jdbc.defaultRowPrefetch=1000` sysproperty on the `run` target |
| `presentation/widget/tree/TreeLoadingOverlay.java` | **New.** Translucent grey cover over one tree's scroll pane while a progressive load is below the row threshold; lives in the layered pane at `PALETTE_LAYER` and swallows mouse, motion and wheel input (§4.17) |
| `presentation/widget/tree/ProgressiveLoadController.java` | **New.** Owns the live-load map, the overlay claim set and the whole progressive lifecycle, moved out of `DynamicTreeListener` (§4.18) |
| `presentation/widget/tree/RefreshData.java` | **New.** The former nested `RefreshData`, moved out unchanged (§4.18) |
| `presentation/lookandfeel/LookAndFeelTransformer.java` | Six `FLATLAF_*` class-name constants (§11.2) |
| `presentation/lookandfeel/StovLookAndFeelTransformer.java` | FlatLaf entries in `lafMap`; `init()` restructured around a `setLookAndFeel` helper that also catches `LinkageError` and falls back to `SUN_WINDOWS`; `applySystemDefaultFont` before install, `applyFlatLafBorders` + `applyFlatLafMetrics` after (§11.3) |
| `build.properties`, `build.xml`, `.classpath` | `flatlaf-3.7.2.jar` added to the Ant properties, the `app.jar` filelist and the Eclipse classpath — all three are required (§11.2) |
| `src/conf`, `src/envs/base`, `src/envs/dist` `.../application.properties` | `stov.laf = FLATLAF_INTELLIJ`, with the available values listed in the comment above it |

---

## 6. Trade-offs accepted

- **Off-screen nodes stay stale by design.** That is the whole point, and its
  correctness depends on the scroll listener plus an interaction-time check. **The
  interaction check is not yet wired** — see open item 6. Until it is, a user can
  act on a phantom row that was deleted elsewhere.
- **Scroll refresh does not re-read parents.** Structural changes to a visible
  parent are therefore not picked up until a full refresh. Accepted because
  re-reading a cell group means re-querying its entire child list.
- **Coalescing costs one debounce interval.** A scroll arriving during an in-flight
  refresh restarts the timer, so the final state can lag by ~200ms. Far better than
  queueing agendas behind a global lock.
- **Leaves are not pre-warmed on a full refresh.** They lazy-load on the EDT during
  paint, as before — about thirty rows. Accepted; the alternative was N forced
  refreshes (§4.4).
- **Row prefetch costs heap.** The driver buffers that many rows per open statement,
  and the client runs at `-Xmx768m` with a tree already holding ~54,000
  `TreeStateNode`s. If memory gets tight, 200 still removes ~95% of the round trips.
- **`canDescendantRepresentObject` no longer throws.** The old code, when the target
  was not a `ContextIdentifiableDomainObject` but the node's name contained `XL` or
  `XI`, reached an unguarded cast and threw `ClassCastException`. It now returns
  false. A crash becoming a sane answer is an improvement, but it *is* a semantic
  change.
- **Progressive loading warms only the first 300 rows.** Rows beyond that are
  inserted without name/icon resolution and lazy-load on scroll (~3 ms/row when
  painted). Accepted; warming all rows for a 49,510-child group would take ~160 s
  (§4.15).
- **Four trees populate on every workspace change**, regardless of which tab is
  visible. Each has its own `DynamicTreeListener` on `Aspects.WORKSPACE`. Deferred
  tab loading would be the fix (§4.15).
- **The tree is unusable for the first ~50 rows.** The wash is a deliberate block,
  not a progress indicator: it trades a second of enforced waiting for not having
  the user click a row that is about to be pushed down the list (§4.17).
- **FlatLaf is tuned rather than the dialogs.** The clipped fields and the tall
  validation band come from hardcoded pixel heights and `dlu` rows in the forms.
  Those are left alone and FlatLaf's own metrics are trimmed instead, because
  `stov.laf` can be switched back to a Swing look and feel at any time (§11.4).
