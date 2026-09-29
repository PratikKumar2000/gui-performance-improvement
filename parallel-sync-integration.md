# Integrating the parallel Atoll sync

How to move a caller from a loop of `PSTO_MASTER_BF` calls to a single
`PSTO_MASTER_PARALLEL_BF` call, and what the measurements say about when that is
worth doing.

## What was built

| Object | Purpose |
| --- | --- |
| `STO_OBJ_ID_TAB` | Collection type callers pass ids in |
| `PSTO_SYNC_WORK` | Work queue and per-id result channel, keyed by `RUN_ID` |
| `SEQ_PSTO_SYNC_RUN` | Run identifier |
| `FSTO_PSTO_PARENT_ID` | Resolves an object's immediate sync parent |
| `PSTO_MASTER_PARALLEL_WRK` | Worker; runs one chunk range in its own session |
| `PSTO_MASTER_PARALLEL_BF` | Orchestrator; takes `STO_OBJ_ID_TAB` |
| `PSTO_MASTER_PARALLEL_BF_STR` | Same, taking a delimited `CLOB` |

Source: `stov_atoll_sync_utility/metadata/09_create_parallel_sync_api.sql` then
`stov_atoll_sync_utility/SyncRegression/PSTO_MASTER_PARALLEL_API.sql`. Deploy in
that order, into each schema owning a caller (`STOSM_O`, `STO1M_O`, `STO2M_O`,
`STOCM_O`). Both scripts derive the application grantee from the schema name and
are safe to re-run.

## Measurements that shape the design

All taken on this database. The no-op figures use `V_VER_NR = -1`, which
`PSTO_MASTER_BF` short-circuits at line 30, so they isolate overhead from work.

| What | Cost |
| --- | --- |
| One no-op `PSTO_MASTER_BF` call | 40 microseconds |
| One real 10-char cell | ~80 ms |
| One real 9-char site | ~260 ms |
| Starting the parallel slaves | ~3,000 ms, fixed |

Two conclusions follow.

**Batching round trips is not the win.** Per-call overhead is four orders of
magnitude below the work, so a server-side loop that merely removes JDBC round
trips would save almost nothing. Concurrency is the only real lever.

**Going parallel has a fixed price, so small batches must not pay it.** The ~3 s
slave startup is independent of both the workload and the chunk count: one id
took 3,030 ms and 400 ids took 3,050 ms. Unguarded, this would turn an ordinary
single-object save from ~80 ms into ~3 s. `PSTO_MASTER_PARALLEL_BF` therefore
compares

```
serial   = N * per_object
parallel = startup + N * per_object / level
```

and only goes parallel above the break-even point,
`N = startup / (per_object * (1 - 1/level))` — about 50 cells at level 4. Below
it, the same worker runs in-session. Callers do not need to guess: passing one id
is now measured at 0 ms of orchestration overhead.

So the wrapper is safe as a blanket replacement. A caller that sometimes syncs
one object and sometimes 8,000 gets serial behaviour for the former and
concurrency for the latter without branching.

## Transaction semantics — read before adopting

`PSTO_MASTER_BF` does **not** commit on success. Its only three `COMMIT`s (lines
148, 236 and 244) sit on failure paths beside `INSERT INTO SYNC_FAILED`. The
Atoll sync is therefore transactional with its caller today.

Workers are separate sessions and cannot see a caller's uncommitted rows, so
`PSTO_MASTER_PARALLEL_BF` **commits the caller's transaction** before
dispatching. Consequences, accepted deliberately:

- The Atoll sync stops being transactional with the STOV change. A caller that
  fails afterwards leaves Atoll updated, so retries must be idempotent.
- PL/SQL callers that currently run a whole cursor loop as one transaction become
  per-batch commits.

## The GUI does commit its batch today

Worth recording, because [global-edit-performance-handover.md](global-edit-performance-handover.md)
suggested a possible lost-update bug here and there isn't one.

`AtollDatabaseDAOImpl.syncStovToAtoll(Connection, Integer, List<String>, String)`
deliberately neither commits nor closes. The commit happens one level up:
`AbstractDialogWorkerTask.syncStovToAtoll` calls `releaseSyncConnection()` in its
`finally`, which calls `commitConnection()`, which does `con.commit()` and then
`con.close(); con = null;`. So the batch is committed exactly once.

Only the comment is wrong. `AtollDatabaseDAOImpl` lines 117-118 say "No commit
here: psto_master_bf commits internally", and it does not — the code is correct
for the wrong stated reason.

Two real defects do remain in the **single-id** overload (lines 49-86), still used
by `AbstractWorkspaceWorkerTask` line 258:

- It calls `con.commit()` and `con.close()` in its `finally` without being able to
  null the task's `con` field, so the task keeps a handle on a closed connection.
- `cb.close()` at line 77 is not null-guarded, so a failure in `prepareCall`
  raises `NullPointerException` from the `finally` and masks the original error.

Both disappear if that overload is retired in favour of the wrapper.

## Recommended JDBC integration: the CLOB entry point

Prefer `PSTO_MASTER_PARALLEL_BF_STR`. Binding a `STO_OBJ_ID_TAB` requires an
`OracleConnection` to call `createOracleArray`, and the STOV client's connection
arrives through JDO's `CustomConnectionHandler`, so it would have to be unwrapped
first. A `CLOB` needs none of that, and the GUI's `objectIdBuilder` already
produces a comma-separated string.

```java
private static final int PARALLEL_LEVEL = 4;

public String syncStovToAtoll(Connection con,
                              Integer versionNo,
                              List<String> ids,
                              String operation) throws DataAccessException
{
  if (ids == null || ids.isEmpty())
  {
    return "Success";
  }

  final String joined = String.join(",", ids);

  CallableStatement cb = null;
  try
  {
    cb = con.prepareCall("{call psto_master_parallel_bf_str(?,?,?,?,?,?,?,?,?)}");
    cb.setInt(1, versionNo);
    // setCharacterStream rather than setString: a global edit can exceed the
    // 32k limit that setString has against a CLOB parameter.
    cb.setCharacterStream(2, new StringReader(joined), joined.length());
    cb.setString(3, operation);
    cb.setString(4, ",");
    cb.setInt(5, PARALLEL_LEVEL);
    cb.registerOutParameter(6, java.sql.Types.NUMERIC);  // run id
    cb.registerOutParameter(7, java.sql.Types.NUMERIC);  // failed count
    cb.registerOutParameter(8, java.sql.Types.NUMERIC);  // rcode
    cb.registerOutParameter(9, java.sql.Types.VARCHAR);  // message

    cb.execute();

    long runId = cb.getLong(6);
    int failed = cb.getInt(7);
    String message = cb.getString(9);

    if (failed > 0)
    {
      logger.error(new LogRecord("Atoll sync run " + runId + ": " + message));
    }
    return message;
  }
  catch (Exception e)
  {
    logger.error(new LogRecord(e.getMessage()));
    return "Error Occurs while Sync to Atoll while operation of " + operation;
  }
  finally
  {
    if (cb != null)
    {
      try
      {
        cb.close();
      }
      catch (SQLException e)
      {
        logger.error(new LogRecord(e.getMessage()));
      }
    }
  }
}
```

Callers keep working unchanged: `P_MESSAGE` is the literal `Success` on a clean
run, so the existing `result.contains("Success")` test still holds, and no failure
message contains that substring.

Keep the connection lifecycle exactly as it is. The wrapper commits internally, so
`releaseSyncConnection()` has nothing left to publish, but calling it still closes
the connection and shuts down the `CustomConnectionHandler`.

The `#`, `@` and `,` splitting in `AbstractDialogWorkerTask` must stay, because a
`#`-encoded payload carries **two different operations** and each needs its own
call. The wrapper takes one trigger mode per call by design.

## Alternative: the collection type

If a caller already holds an `OracleConnection`, `PSTO_MASTER_PARALLEL_BF` takes
the array directly and skips the string round trip.

```java
OracleConnection oc = con.unwrap(OracleConnection.class);
Array ids = oc.createOracleArray("STO_OBJ_ID_TAB", objectIds.toArray());

CallableStatement cb =
    con.prepareCall("{call psto_master_parallel_bf(?,?,?,?,?,?,?,?)}");
cb.setInt(1, versionNo);
cb.setArray(2, ids);
cb.setString(3, operation);
cb.setInt(4, PARALLEL_LEVEL);
// ... four OUT parameters as above
```

`createOracleArray` resolves the name against the *calling* user, which is why
`09_create_parallel_sync_api.sql` grants `EXECUTE` on the type and creates a
synonym for `STOV_HOTLINE_*`. `PSTO_GLOBAL_EDIT` needed the same treatment. If
`unwrap` returns the handler's delegate rather than an `OracleConnection`, fall
back to the CLOB entry point rather than reaching into JDO internals.

## Surfacing per-id failures

`PSTO_MASTER_BF` has a single `V_RCODE` / `V_MESSAGE` OUT pair, so a caller
syncing N objects could only ever see one outcome — and in the old per-id loop a
failure on id 3 of 8,000 was overwritten by a success on the last one.
`PSTO_SYNC_WORK` keeps every id:

```sql
SELECT src_obj_id, obj_id, status, rcode, message
  FROM PSTO_SYNC_WORK
 WHERE run_id = :run_id
   AND status <> 'Success'
 ORDER BY src_obj_id;
```

Every supplied id has a row keyed by `SRC_OBJ_ID`, so nothing is silently
dropped. Statuses:

| Status | Meaning |
| --- | --- |
| `Success` | `PSTO_MASTER_BF` returned rcode 0 |
| `Failed` | Non-zero rcode, or the call raised |
| `Skipped` | Length matches no dispatch branch, or the id contains CR/LF |
| `Pending` / `Running` | Only after a run died mid-flight |

`Skipped` is reported through `P_RCODE`, not swallowed, because
`PSTO_MASTER_BF` raises for an undispatchable id today and hiding that would let
objects silently miss Atoll.

Ids are also normalized and folded: XB TRX tracker keys `<ZEL_ID>_<suffix>` fold
onto their 10-char cell, since syncing the cell covers all four TRGs via
`PATOG_STOSEKG_BF`'s own `STOTRGG` loop. Only the first of a fold group drives a
call; the others take its outcome. Verified: four supplied ids collapsed to one
call with all four rows reporting the result.

## Verification status

Exercised against `STOSM_O`:

- Every object compiles `VALID`.
- Invalid trigger mode returns rcode 999 without committing or starting jobs.
- Empty input returns `Success`.
- Undispatchable ids are reported, not dropped.
- XB tracker keys fold 4:1 and inherit the carrier's outcome.
- The `CLOB` entry point tokenizes, drops blanks and de-duplicates.
- 400 ids across 119 locations produce 16 balanced buckets (18-43 rows each) and
  400 successes with nothing left pending.
- Threshold routing: 1 id and 40 ids run serially; 400 go parallel.

## Concurrency and scaling results on ATOSM_O

Real Atoll writes on `ver_nr = 0` have now been exercised. Harnesses:

| Script | What it covers | Mutates Atoll |
| --- | --- | --- |
| `PSTO_PARALLEL_STRESS_TEST.sql` | folding, hierarchy order, cross-technology bucketing, malformed input, chunk balance | no |
| `PSTO_PARALLEL_SCALING_TEST.sql` | per-object cost and speedup, 10-20,000 ids, levels 1-8 | no (Update only) |
| `PSTO_PARALLEL_DELETE_TEST.sql` | cross-chunk separation contention in Delete mode | yes, snapshot/restore |
| `PSTO_PARALLEL_CONCURRENT_TEST.sql` | two overlapping runs: same ids, disjoint ids at shared sites, and concurrent Delete | Update no, Delete snapshot/restore |
| `PSTO_PARALLEL_SECTOR_FOLD_TEST.sql` | XB tracker-key vs GSM sector discrimination, and serial replay of every parallel failure | yes, Update only |
| `PSTO_PARALLEL_TEST_SNAPSHOT.sql` | snapshot and restore tooling | restores |
| `PSTO_PARALLEL_LOCK_MONITOR.sql` | per-run diagnostics, wait events if `V$` is granted | no |

Two capabilities were added to make this testable: `P_DRY_RUN`, which stages and
buckets a batch without calling `PSTO_MASTER_BF`, and `PSTO_SYNC_WORK.SESSION_ID`,
which is the only reliable evidence a run fanned out. Overlapping timestamps give
false positives at ~1 ms per object, and `USER_PARALLEL_EXECUTE_CHUNKS` is dropped
with the task, so neither can answer that question after the fact.

### Contention: no deadlock, and concurrency changes nothing

150 GSM cells drawn from the 1,708 that participate in a cross-KDT separation,
deleted serially and then at level 8, restoring in between:

| Measure | Serial | Level 8 |
| --- | --- | --- |
| chunks / concurrent sessions | 1 | 32 / 6 |
| separations split across chunks | n/a | 154 |
| Atoll rows destroyed by the cascade | 982 | 982 |
| objects failed | 26 | 26 |
| failures introduced by concurrency | n/a | **0** |

The failure sets are identical, which is the only sound way to attribute a
failure to concurrency. Row-lock contention on `GSEPARATIONS` therefore resolves
by waiting rather than by deadlocking.

An earlier version of this table also claimed "`ORA-00060` deadlocks: 0". That row
has been removed because it was measuring a condition that cannot arise. See
"The deadlock retry could never fire" below.

Deadlock prevention, against the standard checklist: the chunk key is a logical
key (KDT), not `ROWID`; `PSTO_SYNC_WORK` is `INITRANS 16`; there are **no foreign
keys at all** on the Atoll tables, so "index your foreign keys" does not apply
here, and referential integrity is entirely trigger-maintained; there are no
bitmap indexes on the affected tables.

### Scaling: parallelism only pays above ~3,000 objects

Update mode, LTE cells, `ver_nr = 0`:

| ids | serial | level 8 | speedup | sessions |
| --- | --- | --- | --- | --- |
| 100 | 110 ms | 3,050 ms | 0.04x | 1 (stayed serial) |
| 500 | 460 ms | 3,140 ms | 0.15x | 5 |
| 2,000 | 1,860 ms | 3,440 ms | 0.54x | 7 |
| 5,000 | 5,910 ms | 5,060 ms | 1.17x | 7 |
| 20,000 | 32,940 ms | 23,990 ms | 1.37x | 7 |

The measured serial cost is **0.92-1.65 ms per cell, not the 80 ms** the
threshold originally assumed. `C_MS_PER_OBJ` has been corrected from 80 to 1.2,
which moves the break-even from 50 objects to ~2,858 at level 8. Effect of the
fix, same 2,000-cell batch at level 8: **3,440 ms before, 1,750 ms after**,
because it now correctly stays serial.

**Adopt this for bulk interfaces only.** Even at 20,000 objects and 7 concurrent
sessions the speedup is 1.37x, not 7x: at ~1 ms per object the work is dominated
by a shared resource rather than CPU, so extra sessions mostly add contention. A
GUI save is one object and gains nothing.

### Two concurrent runs: the one case bucketing cannot cover

Bucketing guarantees that one location is touched by one chunk, but it partitions
*within* a run. Two callers get two `RUN_ID`s, two sets of buckets and two sets of
sessions, so nothing separates them. This is the shape of a GUI save overlapping a
scheduled `PSTO_DAILY_SYNC`.

Run A is called inline so it genuinely fans out; run B is a one-off scheduler job.
That asymmetry is deliberate and matches production: the wrapper detects
`BG_JOB_ID` and falls back to serial inside a job, because
`DBMS_PARALLEL_EXECUTE` cannot nest. Anything the scheduler drives is therefore
serial, so the realistic worst case is one fanned-out interactive run overlapping
one serial scheduled run. Harness: `PSTO_PARALLEL_CONCURRENT_TEST.sql`.

| | Update, same ids | Update, disjoint ids, shared sites | Delete, same ids |
| --- | --- | --- | --- |
| pool | 4,000 GSM cells | 6,000 | 8,000 |
| A / B | 3,000 @ level 8 inline / 2,500 as a job | 3,000 odd / 3,000 even | 6,000 @ level 8 / 6,000 serial |
| A sessions, chunks | 8, 32 | 8, 32 | 8, 32 |
| ids in common | 1,500 | **0** | 4,000 |
| site rows in common | all of them | **1,785 of 2,183** | all of them |
| seconds genuinely overlapping | 6.5 | 7.4 | **8.5 of A's 8.6** |
| objects failed | 0 | 0 | 0 |
| collisions healed by the retry | 0 | 0 | 0 |
| duplicate site rows | 0 | 0 | 0 |
| Atoll rows lost after restore | 0 | 0 | 0 (39,543 destroyed, all restored) |

The middle column is the interesting one for the site race: because a GSM cell id
carries its site in characters 1..5, taking alternate positions of an id-ordered
pool gives two batches that name no id in common but share 82% of their site rows.
That is the check-then-act on `SITES` in its pure form, and it came through clean.

The Delete column answers a separate question: with 4,000 ids named by both runs,
whichever run arrives second finds the transmitter already gone. That is tolerated
— zero objects failed — and eight sessions firing `TD_GTRANSMITTERS` against
overlapping rows alongside a serial run produced no failure either.

Two sizing traps worth knowing, both of which produce a false pass. A Delete costs
only ~1.4 ms per GSM cell, so an initial 120-cell attempt finished in 150 ms,
which was **before job B had even started** — overlap −2.9 s. And the job start
offset matters: at three seconds a 3,000-object Delete still finished first. The
harness therefore asserts a positive overlap explicitly rather than assuming one,
and uses a one-second offset.

It came through clean because the site rows already existed — measured, 0 of
242,272 GSM cells currently lack their Atoll site row. The race needs the row to
be *absent*, so the exposure is confined to genuinely new sites created by two
callers at once. Provoking it artificially was rejected as not worth the risk:
`TD_SITES` cascades into `UTRANSMITTERS`, `XGTRANSMITTERS`, `XGRELAYLINKS` and
`SITESLISTS`, and `TD_GTRANSMITTERS` into a dozen more, so deleting site rows to
open the window would put far more at risk than the finding is worth.

### The deadlock retry could never fire

Found while assessing the above, and now fixed. The worker's retry was written as:

```sql
e_deadlock EXCEPTION;
PRAGMA EXCEPTION_INIT(e_deadlock, -60);
...
EXCEPTION WHEN e_deadlock THEN ...
```

`PSTO_MASTER_BF` ends in `WHEN OTHERS THEN RAISE_APPLICATION_ERROR(-20002, ...)`,
so every error — deadlocks included — reaches the worker as `ORA-20002`. The
branch was unreachable. Confirmed empirically: all 185 failures recorded in
`PSTO_SYNC_WORK` carry rcode `-20002` and no other value, and none contains
`ORA-00060` anywhere in its message.

Matching on the message text instead is not dependable either, because each level
of the BF call tree re-wraps with `SUBSTR(SQLERRM, 1, 200)`. Real messages show
four to five levels of nesting and visible mid-word truncation:

```
ORA-20002: :DATABASE ERROR <-20002-ORA-20002: :DATABASE ERROR <-20002- ...
  ... <:-20005-ORA-20005: RECORD DOES NOT EXIST IN xgcarriers. CANNOT MODIF>IN ...
```

The retry is now unconditional — one attempt, on any error. It costs one resync of
one object and covers the whole family of transient cross-run collisions rather
than deadlocks alone, including the `SITES` race above: after the loser rolls back,
the winner's row exists, so `V_TEMP_SITE` is 1 on the retry and the object
succeeds. `PSTO_SYNC_WORK.ATTEMPTS` records how many attempts each object took, so
collisions absorbed by the retry are now visible instead of silent.

Verified: re-running the concurrent shared-site pair after the change still gives
8 sessions, zero failures and `ATTEMPTS = 1` throughout; and four ids that fail
deterministically (`ORA-20005` on `xgcarriers`) now record `ATTEMPTS = 2` and still
end `Failed`, so the loop retries and still terminates.

The caveat carried over from the deadlock-only version still applies: a failed
attempt rolls back only its current statement and `PSTO_MASTER_BF` may already
have committed on an error path, so a retry can re-run over partially applied
work. That is safe only because the BFs guard their inserts with a `COUNT(1)`
existence check. An unconditional `INSERT` or a counter update anywhere in that
call tree would double-apply.

### The XB fold silently swallowed every GSM sector

Fixed in the wrapper on 2026-08-31. **If you pull a build of
`PSTO_MASTER_PARALLEL_BF` dated earlier than that, do not pass sector ids to it.**

The tracker-key fold normalized any id that was longer than 10 characters and had
net type `XB` down to `SUBSTR(id, 1, 10)`. A 17-character GSM sector id satisfies
both conditions, so sectors were folded as if they were tracker keys — and the
first 10 characters of a sector are not its cell:

| id | first 10 chars | what it actually is |
| --- | --- | --- |
| `BXB0001aaS001_018` (sector) | `BXB0001aaS` | not a cell id |
| `BXB0001aaA` (its real cell) | — | resolved only via `STOZELG.SEKG_SEK_ID` |

Position 10 is `S` on all 254,768 sectors but a per-cell letter on the cell, so
**0 of 244,266 cells equal their sector truncated to 10 characters**. Because all
four sectors of a site share the same first 10 characters, they also folded 4:1
onto that single bogus id: three rows marked `Pruned`, one `PSTO_MASTER_BF` call
against a non-existent cell, and the sectors never synced. No error surfaced —
the fold is a normal, successful code path, so this was silent data loss.

Blast radius was `XB` only, 244,266 of 692,549 sectors. LTE/5G (`XL`, `XI`, `XV`,
`XJ`) and GSM repeaters (`XF`) never matched the `XB` test, which is why earlier
dry runs still showed level-2 sectors classifying correctly and the bug survived
the whole test campaign.

The fix adds a third condition — an underscore at position 11. A tracker key is
`<10-char ZEL_ID>_<TXT_ID>`, so it always has one there; a sector is
`<9-char alt>S<nnn>_<nnn>` and always has a digit. Measured: 0 of 254,768 sectors
have `_` at position 11, and every tracker key does. Do not relax this back to a
length test.

The general rule, for any future caller: **a sector's cell is not derivable from
its id.** The first 9 characters agree because they are the site alternative, but
nothing beyond that is computable. Use `STOZELG.SEKG_SEK_ID`, which is what the
BFs do.

## Pre-existing defects found, not fixed here

Both reproduce with a plain single-session `PSTO_MASTER_BF` call and are
unrelated to parallelism, so they are reported rather than changed unilaterally.

**1. `PATOG_STOSTOG_BF` fails whenever a separation partner is not released to
Atoll.** Line 42 is `SELECT 1 INTO V_TEMP FROM DUAL WHERE <both cells released>`
with no `NO_DATA_FOUND` handler, so a false condition raises `ORA-01403`, which
the outer `WHEN OTHERS` turns into `ORA-20002`. At `ver_nr = 0`, 2,372 of 11,016
`STOSTOG` rows have such a partner, affecting **250 GSM cells that cannot be
synced at all today**. Reproduced by
`PSTO_MASTER_BF(0,'BXB1111apB','Update',...)`. Fix would be to count instead of
select, or handle `NO_DATA_FOUND` and leave `V_TEMP` at 0.

**2. Re-syncing cannot repair a bulk delete of separations.** `PSTO_MASTER_BF`
walks `STOSTOG` on `ZEL_ID_1` only, and no row is stored in both directions, but
`TD_GTRANSMITTERS` deletes on both `TX_ID` and `TX_ID_OTHER`. So deleting a cell
destroys rows owned by other cells, and re-syncing never brings them back.
Measured: of 982 rows destroyed, re-syncing all 288 affected objects left **112
unrecoverable** (108 `GSEPARATIONS`, 4 `GTRGS`); only the row snapshot restored
them. This is a production data-loss risk in bulk delete plus repair, whether or
not the delete ran in parallel.

**3. Four LTE cells raise `ORA-20005: RECORD DOES NOT EXIST IN xgcarriers`** from
`PSTO_STOZELL_BF` in the 20,000-cell batch, serially as well as in parallel.

**4. Some GSM cells raise `ORA-20005: RECORD DOES NOT EXIST IN "gfreqdomains"`**
from `PSTO_STOZELG_BF` — the 2G counterpart of defect 3. Three of the 400 cells
sampled in `PSTO_PARALLEL_SECTOR_FOLD_TEST` hit it, and all three fail identically
under a single-session `PSTO_MASTER_BF` call. Because the cell BF is also what
syncs a cell's TRGs, all four tracker keys folded onto such a cell inherit the
failure, which is correct reporting but means one missing reference row can
account for four ids in a failure list.

A batch drawn from arbitrary production data will contain objects affected by
these, so **do not treat a non-zero failure count from a bulk run as a parallelism
problem**. The check that distinguishes the two is a serial replay of the failing
targets; `PSTO_PARALLEL_SECTOR_FOLD_TEST` T5 does exactly that and is the pattern
to copy.

## Still unverified

- ITL pressure attributed by wait event, and `INITRANS` on the Atoll tables. The
  `INITRANS 16` rebuild of `PSTO_SYNC_WORK` is written into
  `metadata/09_create_parallel_sync_api.sql` but **not yet applied**, and no
  `enq: TX - allocate ITL entry` measurement exists, because `STOSM_O` gets
  `ORA-00942` on `V$SESSION`, `V$LOCK` and `V$ACTIVE_SESSION_HISTORY`. Section G
  of `PSTO_PARALLEL_LOCK_MONITOR.sql` is ready for a session that has them.
- Worker kill and sweep recovery, nested scheduler-job serial fallback, and MR/RR
  serial-tail ordering under load.
- The `SITES` check-then-act race with the site row genuinely absent. Characterised
  and mitigated by the unconditional retry, but not reproduced, because opening the
  window means deleting site rows and `TD_SITES` cascades too far to be worth it.
  It would be reproducible cheaply on a schema that can be restored wholesale.
- Two concurrent runs where **both** fan out. Not reachable from a single script:
  the scheduler forces serial execution, so a second parallel run needs a second
  interactive session. The tested asymmetric case (8 sessions against 1) is the
  production shape.
- The neighbour write path. `PATOG/PATOL/PATOU_STO_NEIGH_SYNC` are live code but
  the neighbour tables are empty, so it has never executed. Note also that
  `TIA_XGCELLSLTE` is `DISABLED` in `ATOSM_O`, so results here do not transfer to
  schemas where it is enabled.
