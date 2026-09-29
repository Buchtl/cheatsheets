Ah, das ändert die Diagnose deutlich. Wenn du **keinen Zugriff auf den Spring-Code** hast, kannst du die Ursache trotzdem ziemlich gut eingrenzen – direkt auf PostgreSQL-Seite.

Der wichtigste Punkt ist:

> Wenn derselbe `COUNT` in deiner PostgreSQL-Session ~1 Sekunde braucht, bedeutet das noch nicht automatisch, dass die Spring-App den gleichen Query nur 1 Sekunde lang ausführt.

### 1. Schau während der langsamen Anfrage in `pg_stat_activity`

Starte das:

```sql
SELECT
    pid,
    application_name,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS duration,
    left(query, 500) AS query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_start;
```

Dann löst du den langsamen Request aus.

Interessant ist insbesondere:

```text
wait_event_type
wait_event
duration
```

Wenn du beispielsweise siehst:

```text
wait_event_type | Lock
```

wartet die App auf einen Lock.

Wenn du siehst:

```text
wait_event_type | Client
```

kann es etwas anderes sein – beispielsweise dass PostgreSQL bereits fertig ist bzw. auf den Client wartet.

### 2. Prüfe, ob die App überhaupt denselben SQL ausführt

In `pg_stat_activity` solltest du während des Requests das SQL der Spring-App sehen.

Wenn möglich, vergleiche:

```sql
SELECT count(*)
FROM ...
WHERE ...;
```

mit dem Query, den die App tatsächlich ausführt.

Gerade bei Hibernate/JPA kann das SQL erheblich komplexer sein als das, was man zunächst erwartet.

---

### 3. PostgreSQL kann die Laufzeit pro Query protokollieren

Falls du Zugriff auf die PostgreSQL-Konfiguration hast, ist `log_min_duration_statement` sehr hilfreich.

Zum Beispiel temporär:

```sql
ALTER SYSTEM SET log_min_duration_statement = 1000;
```

Danach:

```sql
SELECT pg_reload_conf();
```

Damit werden Statements geloggt, die länger als 1 Sekunde laufen.

Dann Request ausführen und im PostgreSQL-Log schauen.

Wenn dort steht:

```text
duration: 1023 ms
statement: SELECT count(*) ...
```

aber dein HTTP-Request dauert 20 Sekunden, dann ist ziemlich eindeutig:

**PostgreSQL braucht ~1 Sekunde, die restlichen ~19 Sekunden passieren außerhalb der DB.**

Wenn PostgreSQL dagegen loggt:

```text
duration: 19872 ms
```

dann ist es tatsächlich ein DB-seitiges Problem.

---

### 4. Besonders interessant: `application_name`

Wenn die Spring-App eine eigene `application_name` setzt, kannst du sie direkt identifizieren:

```sql
SELECT
    application_name,
    count(*)
FROM pg_stat_activity
GROUP BY application_name;
```

Dann beispielsweise:

```text
application_name | count
-----------------+------
my-spring-app    | 12
psql             | 1
```

Du kannst dann gezielt schauen:

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE application_name = 'my-spring-app';
```

---

### Mein Verdacht

Bei deinem beschriebenen Verhalten:

**PostgreSQL direkt: ~1 s**
**Spring HTTP Request: ~20 s**

würde ich aktuell **nicht davon ausgehen, dass dein Index das Problem ist**.

Es könnte beispielsweise sein:

```text
HTTP Request
    │
    ├── Connection aus Pool bekommen       15 s
    │
    ├── COUNT in PostgreSQL                 1 s
    │
    └── Rest                                 4 s
```

oder:

```text
HTTP Request
    │
    ├── andere DB-Abfragen                   8 s
    ├── COUNT                                1 s
    ├── weitere Verarbeitung                10 s
    └── Response                             1 s
```

Da du keinen App-Zugriff hast, ist **`pg_stat_activity` während eines tatsächlich laufenden 20-Sekunden-Requests** wahrscheinlich der beste nächste Schritt.

Wenn du möchtest, kann ich dir auch ein **einziges SQL-Skript geben, das du während des Requests laufen lässt und das dir möglichst eindeutig zeigt, ob die Spring-App gerade auf Lock, Connection/DB, I/O oder tatsächlich auf dem `COUNT` hängt.**















# #####################
Right — that makes the workload much clearer.

You have **52 separate `COUNT` requests**, but the Spring application appears to have a concurrency limit of **8**, so the database sees roughly:

```text
52 total queries
        │
        ▼
┌───────────────────────┐
│ Spring concurrency 8  │
└───────────────────────┘
        │
        ▼
 PostgreSQL
 ┌────┬────┬────┬────┬────┬────┬────┬────┐
 │ Q1 │ Q2 │ Q3 │ Q4 │ Q5 │ Q6 │ Q7 │ Q8 │
 └────┴────┴────┴────┴────┴────┴────┴────┴──
        │
        ▼
      next 8
        │
        ▼
      next 8
        ...
```

So **8 is the important concurrency number**, not 52.

### For your 70-core machine

This makes me lean toward:

```conf
max_worker_processes = 64
max_parallel_workers = 56
max_parallel_workers_per_gather = 8
parallel_leader_participation = on
```

Why `56`?

You have up to 8 database queries concurrently. If each expensive query wants 8 parallel workers:

```text
8 × 8 = 64 workers
```

But you don't really want to consume every CPU with parallel workers. Having a global pool of ~56 gives PostgreSQL some breathing room.

The actual execution might look like:

```text
70 CPU cores
│
├── PostgreSQL parallel workers     ~56
├── query leaders / other PG work   ~8
├── Spring application               ?
└── OS / VM / filesystem             ?
```

The exact split isn't fixed, of course.

### But there's an even better configuration strategy

Because **only 8 queries can be active**, you don't necessarily want PostgreSQL to give every query the maximum number of workers.

I'd test:

```text
                    max_parallel_workers_per_gather
                               │
                 ┌─────────────┼─────────────┐
                 ▼             ▼             ▼
                4              8             12
```

and measure your **52-query batch completion time**, not just the runtime of the single large query.

That's important.

For example, imagine:

```text
4 workers/query → individual query = 5 s
8 workers/query → individual query = 3.5 s
12 workers/query → individual query = 3.0 s
```

You might find that **8 workers gives the best overall batch throughput**, because 8 concurrent queries × 8 workers already gives PostgreSQL plenty of CPU utilization.

Whereas 12 could create unnecessary contention.

---

## One thing I'd check in Spring

Since you observed exactly 8, I wouldn't assume PostgreSQL is responsible for that limit.

There are several common places where Spring applications end up with an effective concurrency of 8:

* HikariCP `maximumPoolSize`
* `@Async` executor
* `ThreadPoolTaskExecutor`
* Reactor/WebFlux scheduler
* application-level semaphore
* HTTP client connection pool
* custom executor
* JDBC connection pool

If Hikari is configured as:

```properties
spring.datasource.hikari.maximum-pool-size=8
```

then you've effectively got:

```text
Spring
   │
   ├── DB connection 1 ── Query
   ├── DB connection 2 ── Query
   ├── ...
   └── DB connection 8 ── Query
```

and query #9 simply waits for a connection.

That could be intentional and perfectly reasonable.

---

### One important distinction

You said:

> 52 requests

If you mean **52 SQL queries**, then the above applies.

If you mean **52 HTTP requests**, where each HTTP request causes one or more SQL queries, then the concurrency picture could be different.

But based on what you've described, I'd model your workload as:

```text
52 COUNT queries
       ↓
Spring executes 8 simultaneously
       ↓
PostgreSQL sees max ~8 active query sessions
       ↓
each large query can use parallel workers
```

With the 70-core upgrade, that's a **much more sensible workload for PostgreSQL** than 52 simultaneous queries.

And the fact that your large query goes from **25 s → 3–4 s with 8 cores** is a very strong indication that parallel query is already doing useful work.

If you can give me the **RAM of the planned 70-core VM** and whether storage is **NVMe/local SSD/network storage**, I can give you a concrete PostgreSQL 17 `postgresql.conf` tuned around **70 CPU / 8 concurrent queries / 52-query batch** rather than just worker settings.


# ------------
With **70 CPU cores + 64 GB RAM + PostgreSQL 17 + max ~8 concurrent queries**, I'd configure it fairly aggressively on CPU but **not** overly aggressively on memory.

The important caveat: your storage type matters a lot for the final tuning. Assuming reasonably fast SSD/NVMe storage, I'd start around this:

```conf
# ============================================================
# CONNECTIONS
# ============================================================

max_connections = 100


# ============================================================
# PARALLEL QUERY
# ============================================================

max_worker_processes = 64
max_parallel_workers = 56
max_parallel_workers_per_gather = 8
parallel_leader_participation = on


# ============================================================
# MEMORY
# ============================================================

shared_buffers = 16GB
effective_cache_size = 44GB

work_mem = 32MB
maintenance_work_mem = 1GB


# ============================================================
# WAL / CHECKPOINTS
# ============================================================

checkpoint_timeout = 15min
checkpoint_completion_target = 0.9

wal_buffers = 16MB
min_wal_size = 2GB
max_wal_size = 8GB


# ============================================================
# PLANNER
# ============================================================

random_page_cost = 1.1
effective_io_concurrency = 200


# ============================================================
# PARALLEL PLANNER
# ============================================================

min_parallel_table_scan_size = 8MB
min_parallel_index_scan_size = 512kB
```

But I would **not blindly copy all of those settings into production**. Some depend heavily on your storage, workload, and whether PostgreSQL is sharing the machine with the Spring application.

### The most important settings for your particular workload

I'd start with:

```conf
max_worker_processes = 64
max_parallel_workers = 56
max_parallel_workers_per_gather = 8
```

The reasoning is:

```text
70 CPUs
│
├── up to ~56 parallel workers
│
├── ~8 query leaders
│
└── remaining CPU
      ├── OS
      ├── Spring
      └── PostgreSQL background processes
```

And your maximum application concurrency is only 8.

So you're not designing for 52 simultaneous database queries; you're designing for **8 concurrent queries processing a 52-query batch**.

---

## 64 GB RAM is where I'd be conservative

I would **not** do something like:

```conf
shared_buffers = 32GB
work_mem = 256MB
```

That can become dangerous with parallel queries.

Remember that `work_mem` is not a global pool. It can be consumed by **each operation in each backend**, and parallel workers can multiply memory consumption.

For example, conceptually:

```text
8 queries
 ×
8 parallel workers
 ×
32 MB work_mem
=
2 GB
```

And that's only one memory-consuming operation per worker. A complex plan can have several such operations.

That's why I'd start with:

```conf
shared_buffers = 16GB
work_mem = 32MB
```

and increase `work_mem` only if `EXPLAIN ANALYZE` shows sorts/hashes spilling to disk.

---

# One thing I would change depending on the Spring application

If PostgreSQL and Spring are **on the same 70-core VM**, don't actually think of the machine as:

> 64 cores PostgreSQL + 6 cores Spring

Linux doesn't statically divide CPU that way.

Instead, PostgreSQL's workers and Spring's JVM compete for the same CPU scheduler.

That's another reason I like:

```conf
max_parallel_workers = 56
```

rather than:

```conf
max_parallel_workers = 64
```

You don't want PostgreSQL trying to occupy every CPU whenever the 8 expensive queries arrive.

---

# Your biggest potential optimization is still the array query

This is the part I'd investigate before spending too much time tuning PostgreSQL.

You have:

```text
13,000,000 rows
       │
       ├── integer[] datasource IDs
       │
       └── user has up to ~60 datasource IDs
```

If the query is essentially doing 60 separate tests:

```sql
WHERE
    123 = ANY(datasources)
 OR 456 = ANY(datasources)
 OR 789 = ANY(datasources)
 OR ...
```

I'd really want to see whether it can instead become an array overlap:

```sql
WHERE datasources && ARRAY[123,456,789,...]
```

with a GIN index.

For example:

```sql
CREATE INDEX CONCURRENTLY idx_big_table_datasources
ON big_table
USING gin (datasources);
```

**Don't create that exact index without checking the actual query first**, though. The operator used by the query matters.

If that optimization works for your query, you could potentially reduce the CPU work by an enormous amount.

Your current result:

```text
1 CPU  → ~25 sec
8 CPU  → ~3–4 sec
```

already tells us that the query is CPU-parallelizable. But an appropriate index could potentially mean **not having to inspect 13M rows in the first place**.

---

# One more thing: test the entire 52-query operation

This is what I'd benchmark after moving to 70 cores.

Don't just measure:

```text
single big query
```

Measure:

```text
52 queries
8 concurrent
total wall-clock time
```

For example:

```text
Configuration A
4 parallel workers/query
→ total batch: ???

Configuration B
8 parallel workers/query
→ total batch: ???

Configuration C
12 parallel workers/query
→ total batch: ???
```

You may discover that `8` is the sweet spot even if `12` makes an individual query slightly faster.

That's because your actual objective is probably:

> **How quickly can the application complete all 52 counts?**

rather than:

> How quickly can PostgreSQL execute one count?

---

## One correction I'd make to my earlier advice

With **70 CPUs and 64 GB RAM**, I would **not automatically set `max_worker_processes = 64` just because you have 64 CPUs available to PostgreSQL**.

`max_worker_processes` is also used by other PostgreSQL background/extension workers. It needs to be large enough to accommodate them as well as parallel workers.

So a reasonable initial setup is:

```conf
max_worker_processes = 64
max_parallel_workers = 56
max_parallel_workers_per_gather = 8
```

Then watch:

```sql
SELECT *
FROM pg_stat_activity;
```

and especially the actual execution plan:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

If you can bring me that `EXPLAIN (ANALYZE, BUFFERS)` output from work—even with table/column names anonymized—I can tell you **whether I'd use 4, 8, 12, or 16 parallel workers for this specific query**, and whether the GIN/index approach is likely to beat throwing more CPU at it.


# ----------------
Sure. For the **32-core / 64 GB RAM** machine, with your workload of **up to 8 concurrent queries**, I'd start with this:

```conf
# ============================================================
# CONNECTIONS
# ============================================================

max_connections = 100


# ============================================================
# PARALLEL QUERY
# ============================================================

max_worker_processes = 32
max_parallel_workers = 24
max_parallel_workers_per_gather = 4
parallel_leader_participation = on


# ============================================================
# MEMORY
# ============================================================

shared_buffers = 16GB
effective_cache_size = 44GB

work_mem = 32MB
maintenance_work_mem = 1GB


# ============================================================
# WAL / CHECKPOINTS
# ============================================================

checkpoint_timeout = 15min
checkpoint_completion_target = 0.9

wal_buffers = 16MB
min_wal_size = 2GB
max_wal_size = 8GB


# ============================================================
# PLANNER / STORAGE
# ============================================================

random_page_cost = 1.1
effective_io_concurrency = 200


# ============================================================
# PARALLEL PLANNER
# ============================================================

min_parallel_table_scan_size = 8MB
min_parallel_index_scan_size = 512kB
```

### The important differences from the 70-core version

| Setting                           | 32 cores | 70 cores |
| --------------------------------- | -------: | -------: |
| `max_worker_processes`            |   **32** |   **64** |
| `max_parallel_workers`            |   **24** |   **56** |
| `max_parallel_workers_per_gather` |    **4** |    **8** |
| `shared_buffers`                  |    16 GB |    16 GB |
| `effective_cache_size`            |    44 GB |    44 GB |
| `work_mem`                        |    32 MB |    32 MB |
| `maintenance_work_mem`            |     1 GB |     1 GB |

The **64 GB RAM configuration doesn't really need to change just because you're going from 32 → 70 CPUs**. CPU and memory sizing are relatively independent here.

### Why only 4 workers per query on 32 cores?

You have:

```text
8 concurrent queries
×
4 parallel workers
=
32 workers
```

So that's already enough to potentially occupy the whole CPU.

And remember that the query leader participates in the parallel work by default, so the actual CPU demand can be somewhat higher.

Your observed result of:

```text
1 core  → ~25 seconds
8 cores → ~3–4 seconds
```

makes me think **4 workers is worth testing against 8**, though.

I'd benchmark:

```conf
max_parallel_workers_per_gather = 4
```

versus:

```conf
max_parallel_workers_per_gather = 8
```

with:

```conf
max_parallel_workers = 24
```

The latter doesn't mean every query gets 8 workers. It gives PostgreSQL a **24-worker global pool**.

For your workload, I'd probably start with **4**, then move to **8 if the 52-query batch benefits**.

### One thing I would *not* change

I wouldn't increase `shared_buffers` simply because you have 64 GB RAM. **16 GB is a reasonable starting point**, especially because your Spring JVM is also sharing the machine.

And I'd be particularly cautious about `work_mem`: with 8 concurrent queries and parallel workers, a seemingly innocent `256MB` can turn into many GB of potential memory consumption.

Finally, the settings such as `random_page_cost = 1.1` and `effective_io_concurrency = 200` assume **fast SSD/NVMe storage**. If you're on network/block storage, I'd revisit those two.

# -----------------------------------------------------
Absolutely. For your **16-core / 64 GB RAM dev machine**, I'd use a more conservative parallel setup, especially since the dev machine presumably isn't dedicated exclusively to PostgreSQL.

```conf id="x7k2qp"
# ============================================================
# CONNECTIONS
# ============================================================

max_connections = 50


# ============================================================
# PARALLEL QUERY
# ============================================================

max_worker_processes = 16
max_parallel_workers = 12
max_parallel_workers_per_gather = 4
parallel_leader_participation = on


# ============================================================
# MEMORY
# ============================================================

shared_buffers = 16GB
effective_cache_size = 44GB

work_mem = 32MB
maintenance_work_mem = 1GB


# ============================================================
# WAL / CHECKPOINTS
# ============================================================

checkpoint_timeout = 15min
checkpoint_completion_target = 0.9

wal_buffers = 16MB
min_wal_size = 1GB
max_wal_size = 4GB


# ============================================================
# PLANNER / STORAGE
# ============================================================

random_page_cost = 1.1
effective_io_concurrency = 200


# ============================================================
# PARALLEL PLANNER
# ============================================================

min_parallel_table_scan_size = 8MB
min_parallel_index_scan_size = 512kB
```

### The three configurations together

| Setting                           | Dev: 16 CPU | Prod: 32 CPU | Large: 70 CPU |
| --------------------------------- | ----------: | -----------: | ------------: |
| RAM                               |       64 GB |        64 GB |         64 GB |
| `max_worker_processes`            |          16 |           32 |            64 |
| `max_parallel_workers`            |          12 |           24 |            56 |
| `max_parallel_workers_per_gather` |           4 |            4 |             8 |
| `shared_buffers`                  |       16 GB |        16 GB |         16 GB |
| `effective_cache_size`            |       44 GB |        44 GB |         44 GB |
| `work_mem`                        |       32 MB |        32 MB |         32 MB |
| `maintenance_work_mem`            |        1 GB |         1 GB |          1 GB |

For the **16-core dev machine**, `max_parallel_workers_per_gather = 4` is a good starting point. Your big query should still get meaningful parallelism, while leaving CPU available for your IDE, application, Docker, etc.

One thing I'd specifically do on the dev machine is test the big query with:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

and compare `max_parallel_workers_per_gather = 2`, `4`, and `8`. Since your production workload is highly parallel, that experiment on the 16-core machine could give you a useful indication of how well the query scales with CPU.

**And I'd keep the `integer[]` / GIN-index investigation high on the list**. If that query can be changed from dozens of `OR` membership tests into an indexable array-overlap operation, it could make all three machines considerably faster.

# ---- more aggressive
Yes. If the **16-core / 64 GB machine is basically dedicated to PostgreSQL while you're benchmarking**, I'd tune it much more aggressively.

I'd use this as a **database-focused development/benchmark profile**:

```conf id="q8m3zf"
# ============================================================
# CONNECTIONS
# ============================================================

max_connections = 50


# ============================================================
# PARALLEL QUERY
# ============================================================

max_worker_processes = 16
max_parallel_workers = 16
max_parallel_workers_per_gather = 8
parallel_leader_participation = on


# ============================================================
# MEMORY
# ============================================================

shared_buffers = 20GB
effective_cache_size = 52GB

work_mem = 128MB
maintenance_work_mem = 2GB


# ============================================================
# WAL / CHECKPOINTS
# ============================================================

checkpoint_timeout = 15min
checkpoint_completion_target = 0.9

wal_buffers = 64MB
min_wal_size = 2GB
max_wal_size = 8GB


# ============================================================
# PLANNER / STORAGE
# ============================================================

random_page_cost = 1.1
seq_page_cost = 1.0
effective_io_concurrency = 200


# ============================================================
# PARALLEL PLANNER
# ============================================================

min_parallel_table_scan_size = 1MB
min_parallel_index_scan_size = 256kB


# ============================================================
# STATISTICS
# ============================================================

default_statistics_target = 500
```

### The important aggressive changes

Compared with the conservative dev configuration:

**Parallelism:**

```text
max_parallel_workers_per_gather = 8
max_parallel_workers = 16
```

You can therefore let a single big query consume a substantial portion of the machine.

Given your measured:

```text
1 CPU  → ~25 sec
8 CPU  → ~3–4 sec
```

this is particularly useful for your testing.

You can also experiment with:

```conf
max_parallel_workers_per_gather = 12
```

and:

```conf
max_parallel_workers_per_gather = 16
```

to find where your query stops scaling.

---

### Memory

I'd use:

```conf
shared_buffers = 20GB
effective_cache_size = 52GB
work_mem = 128MB
```

**with one warning:** `work_mem = 128MB` is intentionally aggressive.

It isn't "128 MB for PostgreSQL." It is roughly a budget **per memory-consuming operation per backend/worker**.

So don't leave this setting at 128 MB if you're going to use the machine for hundreds of concurrent connections.

For your controlled benchmarking environment, however, it's useful.

For example, if you have:

```text
1 query
  ├── leader
  ├── worker
  ├── worker
  ├── worker
  └── worker
```

and the plan contains multiple hash/sort operations, memory consumption can multiply.

If you see the machine getting close to its 64 GB limit, I'd immediately drop it back to:

```conf
work_mem = 64MB
```

or even 32 MB.

---

## I'd actually use two modes

Since this is your playground machine, I'd keep a **"normal dev"** and **"benchmark mode"**.

### Normal dev

```conf
shared_buffers = 16GB
work_mem = 32MB

max_parallel_workers = 12
max_parallel_workers_per_gather = 4
```

### Database benchmark mode

```conf
shared_buffers = 20GB
work_mem = 128MB

max_parallel_workers = 16
max_parallel_workers_per_gather = 8
```

Then you can restart PostgreSQL and switch profiles when you want to investigate query performance.

---

## And there's one setting I'd particularly experiment with

Because your query is **extremely parallel-friendly**, try:

```conf
max_parallel_workers_per_gather = 2
```

then:

```text
4
8
12
16
```

and run the same:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...
```

For each run, record:

```text
Workers Planned
Workers Launched
Execution Time
Buffers: shared hit
Buffers: shared read
```

You'll probably get a curve something like:

```text
                 execution time
                       │
25 sec ──●            │
        │ \           │
        │  \          │
10 sec ─┤   ●         │
        │    \        │
 5 sec ─┤      ●      │
        │       \ ●   │
 3 sec ─┤         ● ● │
        └────────────────────
          1  2  4  8  12 16
             workers
```

The interesting point is where the curve **flattens**. That's the number I'd use in production rather than blindly basing it on CPU count.

And because your original query involves the `integer[]` datasource access check, I'd make **GIN/index experiments** part of the same benchmark. If you can get the query from work later, that's probably where we can make the biggest improvement.
