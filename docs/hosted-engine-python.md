# Hosted Engine Python

This page documents the local Python surface for the hosted engine.

It is a different package boundary from the public late-interaction SDK:

- use `import kayak` for the open local retrieval SDK
- use `import kayak_engine` for hosted-engine snapshot reuse and local serving
  helpers

Use this page when:

- your data already lives in a hosted-engine service root
- one published snapshot stays fixed across many searches
- you want to reuse that snapshot from Python instead of reloading it per query
- you want one explicit local scheduler for many same-process callers

## One Pinned Snapshot

Use `prepare_exact_search_session(...)` when one caller owns the query loop and
you want to reuse one published snapshot directly.

```python
from kayak_engine import prepare_exact_search_session

session = prepare_exact_search_session(
    service_root="./.state/kayak-engine",
    collection_id="news",
    tenant_id="tenant-a",
    namespace_id="search",
    snapshot_id="snapshot-0001",
)

response = session.search(
    {
        "query_model_name": "colbertv2",
        "query": [[1.0, 0.0], [0.0, 1.0]],
        "final_k": 5,
    }
)
```

Why this exists:

- the snapshot is prepared once
- repeated exact search reuses the same pinned snapshot state
- the request still stays explicit about query vectors, snapshot id, and `final_k`

## Explicit Scoring Knobs

Use `ExactScoringOptions` when you want the exact scoring kernel settings to be
visible in Python instead of relying on hidden defaults in application code.

```python
from kayak_engine import ExactScoringOptions

scoring = ExactScoringOptions(
    enable_parallel_scoring=True,
    enable_dim128_fast_path=True,
    enable_parallel_work_item_oversubscription=False,
    parallel_work_item_count_override=4,
)

response = session.search(request, scoring=scoring)
```

## Many Same-Process Callers

Use `prepare_exact_search_scheduler(...)` when many callers in the same Python
process need one shared same-snapshot exact-search surface.

```python
from kayak_engine import (
    PreparedExactSearchSchedulerConfig,
    prepare_exact_search_scheduler,
)

scheduler = prepare_exact_search_scheduler(
    service_root="./.state/kayak-engine",
    collection_id="news",
    tenant_id="tenant-a",
    namespace_id="search",
    snapshot_id="snapshot-0001",
    config=PreparedExactSearchSchedulerConfig(
        worker_count=4,
        max_batch_size=32,
        max_batch_wait_ms=2,
    ),
)

response = scheduler.search(
    {
        "query_model_name": "colbertv2",
        "query": [[1.0, 0.0], [0.0, 1.0]],
        "final_k": 5,
    }
)
```

The scheduler keeps policy explicit through:

- `worker_count`
- `max_batch_size`
- `max_batch_wait_ms`
- `scoring`

Use:

- smaller `max_batch_wait_ms` for lower tail latency
- larger `max_batch_size` when throughput matters more than single-request latency
- explicit `worker_count` because the right value depends on hardware and query shape

## Concurrent Submitters

Use `submit(...)` when the caller wants a future immediately and plans to wait
later.

```python
future_a = scheduler.submit(request_a)
future_b = scheduler.submit(request_b)

response_a = future_a.result()
response_b = future_b.result()
```

The scheduler coalesces near-simultaneous requests into explicit batches inside
one local worker process.

That process boundary is deliberate.

Reason:

- the verified threaded Python-to-Mojo exact-search path was not stable in the
  current environment
- the process-backed scheduler preserves the same explicit Python contract while
  keeping the actual Mojo execution on a worker-process main thread

## Stats

Use `scheduler.stats()` when you want to inspect what the scheduler is actually
doing.

```python
stats = scheduler.stats()

print(stats.executed_batch_count)
print(stats.average_batch_size)
print(stats.average_queue_wait_ms)
print(stats.average_batch_execution_ms)
```

Current stats include:

- submitted request count
- completed request count
- failed request count
- executed batch count
- last batch size
- maximum observed batch size
- maximum observed queue depth
- total queue wait time
- total batch execution time

## Close The Scheduler

The scheduler owns a worker process, so close it explicitly when you are done.

```python
with prepare_exact_search_scheduler(...) as scheduler:
    response = scheduler.search(request)
```

Or:

```python
scheduler = prepare_exact_search_scheduler(...)
try:
    response = scheduler.search(request)
finally:
    scheduler.close()
```

## What This Is Not

This page does **not** describe the public `kayak` SDK surface from:

```python
import kayak
```

It also does **not** mean the hosted HTTP server is now concurrent.

Current boundary:

- `kayak_engine.prepare_exact_search_session(...)` is the local pinned-snapshot path
- `kayak_engine.prepare_exact_search_scheduler(...)` is the local same-process
  multi-caller path
- `python -m kayak_engine.server ...` is still the current hosted HTTP transport
