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