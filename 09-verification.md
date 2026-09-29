# Verification checklist and resume order

Part of the [STOV GUI tree performance handover](../gui-tree-performance-handover.md).

---

## 9. Verification checklist

- [ ] Round-trip delta across one large expansion, before and after prefetch (§1.4).
- [ ] **Re-record and rerun `JfrEdt` after `WARM_LABELS`** — the EDT socket group
      under `StovTreeRenderer.getTreeCellRendererComponent` should be gone entirely,
      not merely smaller (§4.9). If any remain, a row is reaching the renderer without
      having been through `warmLabels`, and the entry point needs finding.
- [ ] **Confirm the enhancer ran after the `.jdo` change** (§4.10) before concluding
      anything about it. Then check that `getPlanningStatus` no longer appears as a
      lazy load in the worker profile — the field should be populated by the group
      query. If it still fetches, the fetch group is not the one the query used.
- [ ] **Cell-group expansion is faster, not merely differently paced** (§4.10). Time
      it end to end from click to painted rows; the wait cursor being shorter is the
      signal, since the work is no longer being relocated but removed.
- [ ] Icons on cell rows are still **correct** after §4.10, not just faster. The
      change alters when `planningStatus` is loaded, never its value, so a difference
      here means something else reads the field expecting it unloaded.
- [ ] For a large cell and alternative group, record **click-to-first-row** and
      **click-to-last-row** separately (§4.11, §4.15). First paint should follow
      `createChildren` plus ~1 s of warming (300 rows max), not every row.
- [ ] **Workspace root shows children** after load, not an empty tree (§4.12).
- [ ] **No `ArrayIndexOutOfBoundsException`** when expanding or scrolling a wide
      group past row 300 (§4.13).
- [ ] **Tab switch does not restart** an in-progress progressive load (§4.14).
- [ ] During progressive loading, collapse/re-expand the group, select another row,
      refresh it, change the filter and delete an affected object. No cancelled
      generation may append afterwards; a re-expanded target must restart from its
      loading placeholder with no duplicates.
- [ ] For cell-compound and cell-group trees, first paint follows the representation
      query plus wrapping the first queued batches (50, then 250), **not** wrapping
      all 49k nodes (§4.16). Default trees still load atomically.
- [ ] Representation rows show identifier/count/name without extra Oracle round
      trips on `getCellCompound()` / `getGroup()`.
- [ ] Producer queue does not insert after collapse/cancel.
- [ ] **One create adds exactly one row — and adds it at all** (§4.8). Both failure
      directions matter: the duplicate that was fixed, and the missing row that the
      once-per-batch guard could cause if a node ever legitimately needed two syncs.
      Check a sector, a cell and a group.
- [ ] After a single sector create, `stoszn` holds one new row, not two (§4.8). This
      separates the GUI duplicate from a dialog that creates two sectors.
- [ ] **No `CellGroup` or `AlternativeGroup` ever appears below a compound node.**
      If a group refresh must update something in the compound tree, §4.6 silently
      stops finding it.
- [ ] The `ClassCastException` path in the old `canDescendantRepresentObject` never
      actually fired in production (§6).
- [ ] Labels and counts still update after a global edit, both under and over the
      50-cell threshold.
- [ ] A tree on a background tab refreshes when its tab is selected.
- [ ] Selection survives a scroll-triggered refresh.
- [ ] `-Xmx768m` still suffices with prefetch at 1000 on the largest real workspace.
- [ ] **The wash always comes off** (§4.17). Check all four exits: a group larger
      than 50 rows, a group smaller than 50, a load cancelled by collapsing, and a
      load cancelled by a filter change. A tree left permanently greyed is the
      failure to look for.
- [ ] **Two trees filling at once** — the root and an expanded group — do not
      uncover each other early (§4.17).
- [ ] The wash tracks the tree when the window is resized and the split moved, and
      does **not** cover open dialogs, menus or combo popups (§4.17).
- [ ] **The `DynamicTreeListener` split changed no behaviour** (§4.18). It has not
      been compiled or run. Re-check the progressive scenarios above after building.
- [ ] With `stov.laf = FLATLAF_INTELLIJ`: login and change-password fields show
      full descenders, and no dialog has grown a tall empty band (§11.3).
- [ ] With `stov.laf = SUN_WINDOWS`: every dialog looks exactly as it did before
      the FlatLaf work, and no `UIManager` override leaks into it (§11.5).
- [ ] Deleting `flatlaf-3.7.2.jar` from the classpath falls back to `SUN_WINDOWS`
      with a logged error rather than failing to start (§11.1).

---

## 10. Suggested resume order

0. Close out §4.8 first: create one sector, one cell and one group, confirm each
   adds exactly one row.
1. ~~Wire `loading.gif`~~ — **done**, as a translucent grey cover rather than a GIF
   (§4.17). What is still missing is feedback during the ~5 s representation query
   *before* the first batch, since the cover only goes up once a load starts.
2. **Verify producer/consumer** on cell-compound and cell-group only (§4.16, §9).
3. **Page the representation queries** — still the floor for time-to-first-row.
4. **Defer non-visible tree root loads.**
5. Build the `findTreeNode` identity map.
6. Wire the interaction-time staleness check.
7. Viewport-ahead warming if expanding a group of cells still lags on scroll.

Note that the largest remaining number in the *other* document — the empty-pass skip
in `AbstractGlobalEditBuldUpdate` — is unrelated to the tree and still outstanding.
