# Measuring: how to read the numbers

Part of the [STOV GUI tree performance handover](../gui-tree-performance-handover.md).

---

## 1. Read this first: how to measure

Most of the time on this problem was lost to **misreading measurements**, not to
writing code. Three traps, all of which cost real hours.

### 1.1 The profiler uses European number formatting

`19.912 ms` is **19,912 ms — nineteen seconds.** `0,0 ms` is zero. The period is a
thousands separator and the comma is the decimal point.

Getting this backwards inverts every conclusion: a twenty-second hang reads as
twenty milliseconds and you go looking for the problem somewhere else. Sanity-check
against the wall clock before interpreting anything.

Also note the profiler **aggregates every invocation of a frame along one call
path**, so a frame's children are the *union* of everything called across all those
invocations — not what any single call did. This produced a half-correct fix in
§4.6: two sibling frames appeared that one invocation could never have produced,
because the search target differed between calls.

### 1.2 `v$session.STATUS` does not mean what it looks like

`INACTIVE` means *"not inside a database call at this instant"* — the server process
has returned the last call and is parked on `SQL*Net message from client`. It says
nothing about whether the session is connected, holding a transaction, or
mid-operation. With chatty round trips a session flips state thousands of times a
second, so a single sample reads `INACTIVE` most of the time by luck of timing.

Cumulative counters (`v$sesstat`, parse counts, round trips) keep climbing
regardless — they record what *has* happened, not what *is* happening. Seeing
`INACTIVE` alongside a rising parse count is not a contradiction.

Check `LAST_CALL_ET` next to the status: for an `INACTIVE` session it distinguishes
"idle for 20 seconds" from "idle for 2 milliseconds between round trips".

### 1.3 JFR's socket threshold hides most of the traffic

Socket events are only recorded above **20 ms**, so a recording showing a handful of
EDT reads is showing the tail of a much larger population. Use it to find the *call
site* — the stacks are exact — and use the round-trip counters in §1.4 for volume.
`_analysis/JfrEdt.java` extracts every EDT socket, file, park and wait event with
full stacks; JDK 8 has no `jfr` CLI, so it parses via `oracle.jrockit.jfr.parser`
from `jre/lib/jfr.jar`. Note that `Socket Write` carries `bytesWritten`, not
`bytesRead`, and asking for the wrong field throws rather than returning null.

### 1.4 Quantify, don't sample

The measurement that actually settled where the time goes:

```sql
SELECT stat_name, ROUND(value/1e6, 2) AS secs
  FROM v$sess_time_model
 WHERE sid = <sid>
   AND stat_name IN ('DB time', 'DB CPU',
                     'sql execute elapsed time',
                     'parse time elapsed',
                     'hard parse elapsed time');
```

Wall clock minus the `DB time` delta is time spent **outside** the database — client
CPU plus network latency, neither of which Oracle counts.

**First confirm you have the right session.** IntelliBO holds several: the
transactional one plus the exclusive non-transactional resource that
`IboJdoDialect.getJdbcConnection` hands out. It is easy to land on a session only
ever used for bulk SQL.

```sql
SELECT s.sid, s.serial#, s.status, s.last_call_et,
       s.prev_sql_id, q.sql_text
  FROM v$session s
  LEFT JOIN v$sql q ON q.sql_id = s.prev_sql_id
 WHERE s.username IN ('STOV_HOTLINE_SM', 'STOSM_O')
 ORDER BY s.logon_time;
```

If `prev_sql_id` resolves to the cell-children query, that's the tree's session.

Round trips, which double as the check that row prefetch took effect:

```sql
SELECT sn.name, st.value
  FROM v$sesstat st JOIN v$statname sn ON sn.statistic# = st.statistic#
 WHERE st.sid = <sid>
   AND sn.name = 'SQL*Net roundtrips to/from client';
```

