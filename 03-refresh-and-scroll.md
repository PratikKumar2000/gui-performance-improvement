# Refresh, scroll and the findTreeNode discovery

Part of the [STOV GUI tree performance handover](../gui-tree-performance-handover.md).

---

### 4.1 Viewport-limited refresh, wired (supersedes Phase 7)

Implemented per the three corrections in the other document's Phase 7:

- `DefaultTreeNodeAdapter` keeps a **static `dbVersion`** (global epoch) and a
  per-instance **`volatile int guiVersion`**, with `markRefreshed(int)` and
  `isStale()`. The constructor seeds `guiVersion = dbVersion.get()` so a
  freshly-read node is not born stale.
- The epoch is captured **before** the read and stamped **after**, so a concurrent
  bump is not lost.
- `GlobalEditCloseWorkerTask` stops walking the whole tree and bumps the epoch
  instead — O(1) rather than a 106k-node walk.
- A restartable ~200ms `javax.swing.Timer` coalesces viewport events.

Two mechanics that had to be discovered rather than designed:

- `ensureViewportListener()` was originally called inside the
  `walksWholeTree(events)` branch, so the scroll listener never attached unless a
  whole-tree refresh happened first. Moved to every `refresh()` invocation.
- **Trees on hidden tabs never fire viewport events.** A `HierarchyListener` on
  `SHOWING_CHANGED` restarts the timer when the tree becomes visible, otherwise a
  background tab stays stale indefinitely.

`RECURSIVE_EXPANDED` handling was split by scope. Root-scoped events (source `null`,
a `Workspace`, or equal to the root — see `isRootScoped`) bump the epoch and take
the viewport-limited path. Narrow-scoped events, which name a specific domain
object, are processed in full as before, because a structural change in an
off-screen subtree must still be applied.

### 4.2 The refresh button

Changed to use the viewport-limited path rather than `RefreshController`. Going
through the controller notifies every listener and calls `evictAll()`, which is
correct for an explicit full refresh but pointless per scroll.

The viewport path also deliberately passes `null` as the event source rather than
the root's own object: a non-null source makes the presentation task skip
`FilteringTreeModelWrapper.invalidate()`, and re-filtering on every scroll would
undo the point. (An earlier attempt to pass `rootObject()` was reverted after
finding `FilteringTreeModelWrapper.invalidate()` is a no-op in this build.)

### 4.3 Scroll lag

Three distinct causes, fixed together.

1. **Both refresh tasks walked the entire realized tree** — a breadth-first
   enumeration over 50,000+ nodes per scroll stop, the presentation half of it on
   the EDT. `RefreshData` now carries `scopedNodes` (the visible
   `DefaultMutableTreeNode`s captured on the EDT) alongside `inScope`, and both
   tasks iterate that instead when the pass is limited. O(N) becomes
   O(visible_nodes). A guard skips nodes detached between capture and presentation
   (`!scoped.isRoot() && scoped.getParent() == null`).
2. **Refresh agendas piled up**, each taking the global write lock in turn. A
   `scrollRefreshInFlight` flag plus a `ScrollRefreshFinished` presentation task
   (registered via `addNewGuiFrameworkTasksWithFinally`, so it clears even on
   failure) means a scroll during an in-flight refresh merely restarts the timer.
3. **Visible leaf rows lazy-loaded during paint on the EDT.** `prepareForSync` only
   handled parents with loaded children, so cell rows were invalidated on the EDT by
   `synchNode` and then read one round trip per row by the renderer. Added
   `prepareLeafForSync`, and made `synchNode` check `!refreshData.isPrepared(...)`
   before re-invalidating so it doesn't discard the pre-warmed cache.

Also: `visibleNodes` took an `includeParents` flag. The scroll path passes `false`,
so a large cell group's entire child list is no longer re-queried on scroll; the
full path passes `true` and explicitly adds the root, which the old breadth-first
walk had included implicitly.

**A latent NPE surfaced here.** Once `isWorthPreparing` began hashing every node,
`PlaceHolderIsLoadingNodeAdapter` — which passes `null` to `super()` — reached
`AbstractTreeNodeAdapter.hashCode()` and dereferenced it. Fixed in three places:
`hashCode()` returns 0 for a null object, `equals()` compares
`getPresentationObject()` null-safely, and `addAdapter` refuses to put adapters with
a null domain object into the scope set at all.

### 4.4 A regression I introduced, and its fix

`prepareLeafForSync` called `invalidate(true)`, which per §2.3 is a read plus a
commit per row. On a **full** refresh `isLimited()` is false, so `isWorthPreparing`
waves every node through and *every realized leaf* paid it — where previously leaves
weren't prepared at all and the renderer lazily read only the ~30 rows it painted.
That converted about thirty EDT reads into N forced refreshes at two round trips
each, and showed up as the 3.114s `prepareForSync` frame.

Guarded: `prepareLeafForSync` returns immediately unless the pass is limited. This
restores the previous behaviour on the whole-tree path and keeps the pre-warm for
the scroll path it was written for.

### 4.5 Initial load: the `findTreeNode` discovery

Profiling initial load produced the most surprising result of the engagement.

The worker ran 19.912s but used only 4.394s of CPU; **15.518s was blocked in
`SwingUtilities.invokeAndWait`** at zero CPU, waiting for the EDT. The EDT showed
13.298s at 100% CPU, essentially all inside `TaskRunnableAdapter.run`.

Expanded, that resolved to `RefreshTreePresentationTask.runMe` spending **13.163s
inside `TreeUtils.findTreeNode`** — and `synchNode`, `nodeChanged`,
`fireTreeNodesChanged`, everything that actually updates the display, totalling
**103ms**. The task spent 99.6% of its time *finding* the node and 0.8% updating it.

Two causes, roughly even:

**~5.9 seconds was debug logging that was never printed** — `StringBuilder.append`
3.978s plus `Log.debug` frames at 964ms and 965ms.
`TreeUtils.findRecursiveChildTreeNode` built `"Suche nach: " + domainObject` and
`"Untersuche: " + tna.getPresentationObject()` for every node it walked. Because the
string and the `LogRecord` are constructed *before* `Log.debug` is called, the level
check inside comes too late to save anything. The first also calls `toString()` on a
persistent object per node. `Log.isDebugEnabled(Class)` already existed; it is now
resolved once per search and threaded through the recursion as a parameter.

**6.523 seconds was `canDescendantRepresentObject`**, which called
`getPresentationObject()` up to four times and `po.toString()` twice per node — the
second time only to test a different two-character substring of the same string.
Collapsed to one each.

Result: EDT presentation time **13.298s → 1.791s**, and the logging frames vanished
from the profile.

### 4.6 The compound tree searches for objects it cannot contain

Re-profiling showed the remaining cost had moved to the worker: 10.333s inside
`canDescendantRepresentObject`, whose children named the participants —
`CellGroupImpl.getContextIdentifier` 4.228s, `CellGroupTreeRepresentationImpl.getIdentifier`
2.539s, `CellCompoundTreeRepresentationImpl.getContextIdentifier` 1.406s,
`AbstractPersistentObject.toString` 628ms, `CellGroupImpl.getIdentifier` 566ms.

The object being searched for is a **cell group**. The nodes being asked about it
are **cell compounds**. A cell group refresh event is dispatched to the cell
compound tree's listener, which then walks its entire tree asking whether a cell
group might live below a cell compound — a question whose answer is structurally
always no. `CellCompoundNodeFactory` builds children from workspaces, compounds,
candidates, alternatives, sectors and cells. Ten seconds of identifier comparisons
to establish nothing.

`CellCompoundNodeAdapter.canDescendantRepresentObject` now returns false for such
targets. **This took two passes:** the first guard tested only
`instanceof DomainObjectGroup`, worth about 4.8s of the 10.3s. Refresh events name a
group either way round, and `DomainObjectGroupTreeRepresentation` extends
`TreeFilterRelevantNode` and `ContextIdentifiableDomainObject` — it *has* a group
via `getGroup()` rather than being one, so it slipped through. The guard now
excludes both forms.

Verified safe: `CellCompoundNodeAdapter` has no subclasses, and
`DomainObjectGroupNodeAdapter` is a sibling (both extend
`DefaultTreeFilterRelevantNodeAdapter`), so the early return is reachable only from
compound nodes and cannot stop the group tree from finding its own groups.

### 4.7 JDBC row prefetch

The Oracle thin driver defaults to **ten rows per round trip**, so reading a group of
34,000 cells costs thousands of server round trips. Driver is `ojdbc6-11.2.0`.

Applied at two levels, deliberately:

1. **System property** `oracle.jdbc.defaultRowPrefetch=1000`, added as a
   `<sysproperty>` to the `run` target in `build.xml`. On the client machine the
   equivalent goes in the shortcut target, the `.bat`, or the JNLP's
   `java-vm-args`. Zero code, and `ojdbc6` does honour it.
2. **Connection properties on the PMF**, because the property is best-effort. `javap`
   on `ibo.jar` shows `PersistenceManagerFactoryImpl.setConnectionProperties(Properties)`
   — the bag handed to the driver when it opens a connection.
   `LocalIBOPersistenceManagerFactoryBean` gained a Spring-injected `rowPrefetch`
   (default 1000, exposed in `persistence.xml`) and an `applyRowPrefetch` that merges
   into that bag **non-destructively**, keeping anything already present and not
   overwriting an explicit setting.

Two dead ends ruled out along the way, both of which look correct and aren't:

- **Adding it to `jdoProperties` in the XML does nothing.**
  `newPersistenceManagerFactory` translates a fixed whitelist of keys into explicit
  setter calls and silently drops everything else. `<prop key="defaultRowPrefetch">`
  would have been ignored without a word.
- **Setting it on the connection from `IboJdoDialect.getJdbcConnection` would not
  reach the tree.** That method calls `rm.getExclusiveNTXJDBCResource(...)`, a
  dedicated non-transactional connection used by Global Edit's bulk SQL and the
  sync. Tree reads go through JDO queries on IntelliBO's own connections.

---

### 4.8 The same object appearing twice after a create

**Symptom.** Creating one sector (reproduced with GSM 900) put **two identical rows**
under the alternative. Both carried the same identifier, and the tree's own
deduplication did not collapse them.

**Why deduplication did not save it.** Easy to assume a `HashSet` is missing
somewhere. It isn't: the child list is a `TreeSet` ordered by
`AlphabeticalTreeNodeComparator`, which for a `ContextIdentifiableDomainObject`
compares `getIdentifier()`, so two adapters for one sector *do* collapse inside a
single read. The duplicate is not two entries in one child list — it is **two child
nodes inserted into the tree model** by two passes over the same diff. Nothing
re-checks the model for a node that already represents the object;
`TreeUtils.addChild` inserts what it is given.

**Mechanism.** `synchNode` computes `added = newChildren − oldChildren` and inserts
`added`. `oldChildren` is a snapshot taken by the worker in `prepareForSync` *before*
it invalidates and re-reads. The diff is therefore **relative to a point in time**,
and applying it twice inserts everything in it twice. A batch reaches one node more
than once easily, because `RefreshController` hands both tasks the whole event list
and a create emits one event per affected object — the object, its parent, its
alternative groups, its cell groups — several of which can resolve to the same node.
The second pass then runs like this:

| Step | Pass 1 | Pass 2 |
|---|---|---|
| `prepareForSync` | snapshot taken, `invalidate(true)`, `markPrepared` | returns at the `isPrepared` check — **snapshot left at its pre-change value** |
| `synchNode` old children | pre-change snapshot | the *same* pre-change snapshot |
| `synchNode` invalidate | skipped (worker did it) | skipped, `isPrepared` still true |
| `synchNode` new children | re-read, contains the new sector | same re-read, still contains it |
| `added` | the new sector → inserted | the new sector **again** → inserted again |

Plain GSM create (`CreateSectorWorkerTask`) builds `refreshNodes` as the new sector
plus `RefreshNodes.getRefreshNodes(newSector)` — parent alternative, its groups,
cell groups — and `generateRefreshEvents` emits one `MODIFIED` event per entry.
The new sector's own event never inserts the row: `findTreeNode` returns `null`
because that node does not exist yet. The row only appears when the parent is
synched and the new sector is in `added`.

```mermaid
sequenceDiagram
  participant Create as CreateSectorWorkerTask
  participant RC as RefreshController
  participant W as RefreshTreeWorkerTask
  participant EDT as RefreshTreePresentationTask
  participant Tree as DefaultTreeModel

  Create->>RC: events = newSector + parent + groups
  Note over RC: one agenda, one RefreshData

  RC->>W: prepareForSync(alternative)
  W->>W: snapshot oldChildren = {A,B}
  W->>W: invalidate + re-read = {A,B,newSector}
  W->>W: markPrepared

  RC->>EDT: event newSector
  EDT->>EDT: findTreeNode = null, skip

  RC->>EDT: event parent alternative
  EDT->>Tree: synchNode pass 1
  Note over EDT,Tree: added = {newSector}<br/>insert first row<br/>without fix: snapshot still {A,B}

  RC->>EDT: another event resolving to same node
  EDT->>Tree: synchNode pass 2
  Note over EDT,Tree: still old={A,B} new={A,B,newSector}<br/>added = {newSector} again<br/>second identical row
```

**This is a regression I introduced, and the second one in this document after
§4.4.** The original `synchNode` read its old children inline and called
`invalidate(true)` itself on every pass. That was slow — the read landed on the EDT,
which is exactly what §4.3 set out to remove — but it was *idempotent*: a second pass
re-invalidated, re-read, and diffed fresh against fresh, so `added` came out empty
and nothing was inserted twice. Moving the read to the worker and guarding it with
`isPrepared` removed the redundant work and, with it, the accidental idempotence.
Each guard is right in isolation; together they make the second pass diff stale
against fresh.

The general lesson for this code: **a diff against a captured snapshot is only safe
if the pass that applies it runs exactly once.** Any future caching of pre-change
state needs the same claim discipline.

**Two fixes, both in `DynamicTreeListener`.**

1. `synchNode` writes the re-read children back as the snapshot once it has applied
   the diff, so `oldChildren` describes what the tree now shows rather than what it
   showed before the change.
2. `RefreshData.claimForSync(node)` claims each node once per batch, and `synchNode`
   returns immediately on a second attempt. This is the actual safety net: it holds
   for every event type and however the node was reached twice.

`claimForSync` is keyed on the **node**, not on its adapter. `DefaultMutableTreeNode`
does not override `equals`, so the set compares by identity and can be trusted. An
adapter-keyed set could not be — see trap 15.

**`RefreshData` is per batch**, constructed fresh in `refresh()` (line 439) and in
`refreshVisibleStaleNodes` (line 718), so the claim set cannot suppress a later
refresh. Worth re-checking if anyone ever hoists it to a field.

**Two wrong turns, recorded so they are not repeated.**

- *It is not the RECURSIVE_EXPANDED subtree walk.* That was the first explanation —
  a create fires an event for the alternative and another for its candidate, whose
  subtree contains the alternative, so the BFS visits it twice. Real, but the wrong
  branch: `CreateSectorWorkerTask` reaches `generateRefreshEvents` for a plain
  sector, which emits **MODIFIED**, and MODIFIED syncs one resolved node with no
  walk. `generateRefreshEventsNoSelection` (RECURSIVE_EXPANDED) is the other branch.
  The mechanism above is type-agnostic, which is why the fix covers both.
- *`isWorthPreparing` does not return false for a MODIFIED batch.* Tempting, since
  the `RefreshData` is empty at that point, but scope is a **nullable** set where
  null means unrestricted: `isLimited()` is `inScope != null`, and `inScope` is only
  assigned by `limitTo`, which `refresh()` calls only when `walksWholeTree(events)`
  holds — all events RECURSIVE_EXPANDED and root-scoped. A MODIFIED batch never gets
  there, so `isWorthPreparing` returns true on its first statement and everything
  below it is unreachable. The early exit that *can* fire is
  `hasLoadedChildren(node) || node.isRoot()` at line 1273, which sends a collapsed
  node to `prepareLeafForSync` and out again on the `isLimited()` guard. That one is
  harmless, because `synchNode` applies the identical test and takes its own `else`
  branch, which does not touch children.

**Also note:** the new object's *own* event always resolves to `null` in
`findTreeNode`, because its node does not exist yet. Both tasks skip it. A created
object reaches the tree only as a child added by its parent's sync — so if a create
ever shows *no* new row, look at the parent's event, not the object's.

**Not established.** Whether the reported case was one sector row rendered twice or
two rows written by `CreateSectorWorkerTask`. The fix addresses the former. If two
rows are in `stoszn`, that is the dialog's **Number** field and the master-map loop,
and it is a separate problem. One `SELECT` on `stoszn` after a single create settles
it and is worth doing before closing this out.

---

### 4.9 Proof of Oracle reads on the EDT, and where they come from

The first hard evidence rather than inference. `jfr-1788356828773.jfr`, analysed with
`_analysis/JfrEdt.java` (built for this, since JDK 8 has no `jfr` CLI).

**What the recording shows on `AWT-EventQueue-0`:**

| Event | Count | Total | Max |
|---|---|---|---|
| Socket Read | 81 | 9,289 ms | 324 ms |
| Socket Write | 5 | 342 ms | 102 ms |
| Java Thread Park | 2,389 | 161,732 ms | 4,050 ms |

Read the park row carefully and then ignore it: 2,269 of those events are
`EventQueue.getNextEvent`, which is the EDT **idle** and waiting for work. Of the
rest, 90 are a nested pump under `WaitDispatchSupport` and 29 are `Dialog.show` — a
modal dialog open, not a stall. Only the socket rows are the EDT actually blocked.

**Every EDT read goes to Oracle.** Port 1521 on `195.233.84.103`. The much larger
socket totals on `RMI TCP Connection(6/7/8)` go to `195.233.84.20:59362` and are the
RMI transport, which is a different thing and not on the EDT.

**Every EDT read has the same shape** — 86 of 86:

```
SocketInputStream.read → oracle.net → T4CPreparedStatement.executeForDescribe
  → IntelliBO JDOInstanceManager.preLoadField        <- lazy field load
  → <a domain getter>
  → IconMap.getIcon / getIconForGroup
  → <factory>.getIcon
  → DefaultTreeNodeAdapter.getIcon
  → AbstractTreeNodeAdapter.getCachedIcon            <- first-touch cache miss
  → StovTreeRenderer.getTreeCellRendererComponent
  → BasicTreeUI$NodeDimensionsHandler.getNodeDimensions
  → VariableHeightLayoutCache$TreeStateNode.updatePreferredSize / expand
  → VariableHeightLayoutCache.treeStructureChanged
  → DefaultTreeModel.nodeStructureChanged
  → ShowTreeNodesTask.runMe:143
```

Read bottom-up, that is the whole mechanism. `ShowTreeNodesTask` ends with
`nodeStructureChanged`; Swing's layout cache responds by measuring the rows; measuring
a row calls the renderer; the renderer asks for the icon; `getCachedIcon` is a
**first-touch** cache that `createChildren` never populates, so it misses; `getIcon`
runs `IconMap`, which is not a field read but business logic that reaches unloaded
JDO fields; each one is a query. On the EDT, one row at a time.

**It is the load path, not the refresh path.** Worth stating plainly, because the
symptom looks like refresh: `ShowTreeNodesTask` is expansion and selection, and it
does not go through `prepareForSync` at all. That is why the §4.3 leaf pre-warm,
which fixed exactly this for scroll, left it untouched — different entry point.

**The costly getters, by EDT time:**

| Getter reached from `IconMap` | Events | EDT time |
|---|---|---|
| `LTEAlternativeRedundanciesImpl.getPlanningStatus` (cell rows) | 46 | ~4,740 ms |
| `AlternativeImpl.getSectors` — `gsm`, `lte`, `newRadio5G` | 20 | ~3,000 ms |
| `LTEPlainSectorImpl.getMasterUnitAlternative` via `BusinessUtilities.isRepeater` | 3 | 277 ms |
| `CellCompoundImpl.getParent`/`getVersion` via `isDefaultWorkspace` | 1 | 280 ms |

Two things stand out. `getSectors` loads whole **collections** per alternative row
just to choose an icon. And roughly half of all these events are not the query at all
but `IntelliBOJDBCConnectionWrapper.commit` immediately after it — so each lazy field
load costs **two** round trips, matching the read-plus-commit pattern already noted
for `invalidate(true)` in trap 10.

**These 86 events are the tail, not the total.** JFR's socket threshold is 20 ms, so
every round trip faster than that is invisible. 9.3 s is a floor on EDT socket time,
not a measurement of it.

**Fix applied.** `CreateTreeNodeWorkerTask.warmLabels` resolves `getCachedName()` and
`getCachedIcon()` for each new child on the **worker thread**, right after
`createChildren` and before `ShowTreeNodesTask` fires `nodeStructureChanged`. Behind
`WARM_LABELS`. Placeholders are skipped — they hold no domain object.

This relocates the work rather than reducing it: the same rows are resolved, just off
the EDT. That is the point — the EDT stays able to paint and accept input — but it
does mean the wait cursor is up for a comparable time, and for a 34,000-cell group it
is 34,000 resolutions either way. Reducing the count needs either a batched prefetch
of the fields `IconMap` touches (open item 7 generalised) or a cheaper icon rule, and
neither is contained enough to bundle with this.

---

### 4.10 Removing the icon reads instead of relocating them

§4.9 moved the per-row hydration off the EDT. This removes most of it.

**What §4.9 changed about the symptom.** Before `WARM_LABELS`, rows appeared in
bunches: the renderer hydrated each row as it painted it, so the first screenful
showed up early and the rest filled in. After it, nothing appears until the last
child is resolved, because `warmLabels` runs to completion before
`taskCtxt.setResult(children)` hands anything to `ShowTreeNodesTask`. The total work
was unchanged; only the point of first paint moved to the end.

The wait cursor also became visible for the first time, which reads like a
regression and isn't. It is set and cleared inside `CreateTreeNodeWorkerTask.runMe`,
so `warmLabels` sits between the two calls. Previously the same work ran later,
inside `ShowTreeNodesTask` on the EDT — and the agenda holds the EDT with
`SwingUtilities.invokeAndWait`, so Swing could not paint a cursor at all. A visible
wait cursor is what a blocked EDT looks like once it is no longer blocked.

**Root cause of the reads themselves.** `IconMap.getIconForGroup` reads, in order,
`getAlternativeRedundancies()`, `getSearchCriterion()` and then
`getPlanningStatus()`. The group query in `CellGroupDAOImpl.findGroupCells` asks for
fetch group `alternativeRedundancies`, which listed `name`, `status`,
`searchCriterion` and `vendorType` — but not `planningStatus`. So the first three
calls are satisfied from the query and the fourth is a lazy load: one select plus one
commit per row, matching the read-plus-commit pattern in trap 10. That is the 46
events and ~4,740 ms attributed to `getPlanningStatus` in §4.9, and the ~115 selects
and ~167 commits seen in the NPS snapshot for one cell group.

**Fix applied.** `alternativeRedundancies.planningStatus` added to the
`alternativeRedundancies` fetch group in all nine cell mappings — GSM, UMTS, LTE and
5G in plain and repeater form, plus `RemoteUnitRepeaterCellImpl`. It joins a
one-to-one association the group already fetches, so this is one more column on an
existing join rather than a new one.

Both callers pick it up without further change: `CellGroupDAOImpl.findGroupCells` and
`AlternativeGroupDAOImpl.findGroupCells` request the group by name.

**Points worth knowing before touching this again.**

- **`rnpTempFlag` was deliberately left out.** `IconMap` reads it too, but it is
  declared `primary-key="true"` in every `*AlternativeRedundanciesImpl.jdo`, so it
  arrives with the identity. Add it only if a profile still shows a fetch for it.
- **Each cell class has its own redundancies class.** The nine cell mappings resolve
  to nine different `*AlternativeRedundanciesImpl` classes —
  `RemoteUnitRepeaterCellImpl` to `RepeaterAlternativeRedundanciesImpl`, for example.
  All nine declare `planningStatus` as a direct mapping onto `alt_planning_status`,
  which is why one field name works across all of them. Verify this again if a new
  cell type is added.
- **These are `.jdo` mappings, so they need the enhancer.** A plain class recompile
  will not pick the change up, and the symptom of skipping it is simply that nothing
  gets faster.
- **`warmLabels` stays.** It is now a safety net rather than the mechanism: with the
  field in the fetch group, `getPlanningStatus()` should hit populated state. Leaving
  it in place keeps any row that still needs work off the EDT.

**What this does not cover.** Alternative rows, not cell rows, reach
`AlternativeImpl.getSectors` from `IconMap` — 20 events, ~3,000 ms in §4.9 — and that
loads whole collections rather than one field, so a fetch group does not fix it. Open
item 5 remains open for that half.
