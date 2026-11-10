Got it. You don’t care about roles — you just have specific `SELECT COUNT(*)` queries that are too slow with only 4 workers.

On a 32-core machine that’s completely fine. Just raise the limit.

### Simple fix (global)

In `postgresql.conf`:

```ini
max_parallel_workers_per_gather = 16
max_parallel_workers = 32
max_worker_processes = 48
```

Then restart (or at least reload + new connections).

With 16 workers a big `COUNT(*)` can use up to 16 cores for the sequential scan + aggregate. That’s usually the sweet spot on 32 cores — going all the way to 32 rarely helps more and can hurt other queries.

### Even better for just those tables

You can force more workers only on the big tables without changing the global setting:

```sql
ALTER TABLE your_big_table SET (parallel_workers = 16);
```

Then even with `max_parallel_workers_per_gather = 4` globally, that table will still get up to 16 workers when the planner decides to parallelize.

### Quick test right now (no restart)

```sql
SET max_parallel_workers_per_gather = 16;
EXPLAIN (ANALYZE, BUFFERS)
SELECT COUNT(*) FROM your_big_table;
```

You’ll immediately see whether 16 workers actually helps that query.

---

**Bottom line:** 4 is conservative for a pure heavy `COUNT(*)` on a 32-core box. Raise it to 8–16 (globally or just on the table). That’s not excessive.


With a primary + identical replica and **pgpool-II** doing 50/50 read split, you have more headroom on each node for heavy reads. That changes the parallel-worker advice.

### Updated recommendation

On **both** nodes (primary and replica):

```ini
max_parallel_workers_per_gather = 12
max_parallel_workers = 32
max_worker_processes = 48
```

- 12 is a good practical number for large `COUNT(*)` / sequential scans on 32 cores when reads are already load-balanced.
- Going to 16 is also fine if those counts are the main pain and concurrency is not extreme.
- Still avoid 24–32. Even with pgpool, multiple heavy queries can land on the same node, and each worker can burn `work_mem`.

### Why this is safer now

- ~50% of the read load goes to the replica.
- The primary still does all writes, but the big read queries are spread out.
- You have two machines with the same specs, so saturating one with parallel workers is less catastrophic.

### Quick test on either node

```sql
SET max_parallel_workers_per_gather = 12;
EXPLAIN (ANALYZE, BUFFERS)
SELECT COUNT(*) FROM your_big_table;
```

Compare with 4 and with 16. Pick the number where you still see good scaling but don’t lose the rest of the machine.

### Other notes for this setup

- Keep the rest of the memory and I/O settings I gave earlier the same on both nodes.
- Make sure pgpool-II is actually load-balancing the `SELECT COUNT` statements (some versions/settings treat certain queries differently).
- Watch `work_mem` carefully — with 12 workers a single query can use a lot of RAM if it does sorts/hashes.

So yes: with the replica + pgpool, **12** (or 16) is a reasonable global setting. 4 was too conservative for your case.

Use `RESET` to remove the table-level override so it falls back to the global setting:

```sql
ALTER TABLE your_big_table RESET (parallel_workers);
```

That clears the custom `parallel_workers` value. The table will then use whatever `max_parallel_workers_per_gather` (and the planner) decide, same as any other table without an explicit setting.

You can confirm it’s gone with:

```sql
SELECT reloptions FROM pg_class WHERE relname = 'your_big_table';
```

It should no longer show `parallel_workers=...`.



Yes — that’s exactly the right move.

### Do this

1. Lower the global limit so the 50 lighter counts don’t each grab many workers:

```sql
-- in postgresql.conf or via ALTER SYSTEM
max_parallel_workers_per_gather = 2
-- or even 1 if you want to be strict
```

2. Force more workers only on the heavy table:

```sql
ALTER TABLE your_very_heavy_table SET (parallel_workers = 16);
```

Now:
- The heavy table can use up to 16 workers when a query runs on it.
- All other tables stay limited to the low global value (2 or 1).
- The lighter counts stop stealing so many CPUs from the heavy one.

### Notes

- The table setting is an upper limit / hint. The planner still decides whether to parallelize and how many workers to actually use (it won’t exceed `max_parallel_workers_per_gather` *and* the table’s `parallel_workers`, and it also respects the global `max_parallel_workers` pool).
- You still need enough free slots in `max_parallel_workers` and `max_worker_processes` (keep those high, e.g. 32 and 48).
- After changing the global setting, new connections pick it up (or restart if you want to be sure).

This is the cleanest way to solve “50 counts at once, one of them is much heavier”.