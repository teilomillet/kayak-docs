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
- you want one explicit local runtime for many same-process callers

## Choose The Hosted Python Shape

=== "One Caller Owns The Query Loop"

    Use:

    - `prepare_exact_search_session(...)`

    Choose this when one caller reuses one pinned snapshot directly.

=== "Many Callers Share One Fixed Snapshot"

    Use:

    - `prepare_exact_search_runtime(...)`
    - `concurrency_lane_count` to raise same-snapshot concurrent capacity

    Choose this when one Python process needs one explicit multi-caller runtime.

=== "I Need Futures And Batching"

    Use:

    - `runtime.submit(...)`
    - `runtime.stats()`

    Choose this when callers submit work independently and you want explicit
    queue and batch behavior.

=== "I Actually Want The Public SDK"

    This is the wrong page.

    Use:

    - [Usage Patterns](usage-patterns.md)
    - [API Reference](api.md)

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

Use `prepare_exact_search_runtime(...)` when many callers in the same Python
process need one shared same-snapshot exact-search surface.

```python
from kayak_engine import (
    PreparedExactSearchRuntimeConfig,
    prepare_exact_search_runtime,
)

runtime = prepare_exact_search_runtime(
    service_root="./.state/kayak-engine",
    collection_id="news",
    tenant_id="tenant-a",
    namespace_id="search",
    snapshot_id="snapshot-0001",
    config=PreparedExactSearchRuntimeConfig(
        concurrency_lane_count=2,
        worker_count=4,
        max_batch_size=32,
        max_batch_wait_ms=2,
    ),
)

response = runtime.search(
    {
        "query_model_name": "colbertv2",
        "query": [[1.0, 0.0], [0.0, 1.0]],
        "final_k": 5,
    }
)
```

The runtime keeps policy explicit through:

- `execution_backend`
- `concurrency_lane_count`
- `worker_count`
- `max_batch_size`
- `max_batch_wait_ms`
- `scoring`

Current verified backend support:

- only `execution_backend="process"` is supported today

Reason:

- the runtime contract should stay stable even if the execution mechanism
  changes later
- the threaded Python-to-Mojo exact-search path was not stable in current local
  evidence

Current lane meaning:

- under the verified `process` backend, each concurrency lane is one worker
  process with its own prepared snapshot state

Tradeoff:

- larger `concurrency_lane_count` can increase same-snapshot concurrent search
  capacity
- each additional lane also increases prepared-snapshot memory use and worker
  startup cost

Use:

- `concurrency_lane_count` to choose how many independent runtime lanes you
  want
- smaller `max_batch_wait_ms` for lower tail latency
- larger `max_batch_size` when throughput matters more than single-request latency
- explicit `worker_count` to choose the Mojo batch-kernel parallelism inside one
  lane

## Concurrent Submitters

Use `submit(...)` when the caller wants a future immediately and plans to wait
later.

```python
future_a = runtime.submit(request_a)
future_b = runtime.submit(request_b)

response_a = future_a.result()
response_b = future_b.result()
```

The runtime coalesces near-simultaneous requests into explicit batches inside
one local worker process.

That process boundary is deliberate.

Reason:

- the verified threaded Python-to-Mojo exact-search path was not stable in the
  current environment
- the process-backed runtime preserves the same explicit Python contract while
  keeping the actual Mojo execution on a worker-process main thread

## Stats

Use `runtime.stats()` when you want to inspect what the runtime is actually
doing.

```python
stats = runtime.stats()

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

## Close The Runtime

The runtime owns a worker process, so close it explicitly when you are done.

```python
with prepare_exact_search_runtime(...) as runtime:
    response = runtime.search(request)
```

Or:

```python
runtime = prepare_exact_search_runtime(...)
try:
    response = runtime.search(request)
finally:
    runtime.close()
```

Compatibility note:

- `PreparedExactSearchSchedulerConfig`
- `PreparedExactSearchScheduler`
- `prepare_exact_search_scheduler(...)`

still exist as aliases for earlier code, but `runtime` is the canonical naming.

## What This Is Not

This page does **not** describe the public `kayak` SDK surface from:

```python
import kayak
```

It also does **not** mean the hosted HTTP server is now concurrent.

Current boundary:

- `kayak_engine.prepare_exact_search_session(...)` is the local pinned-snapshot path
- `kayak_engine.prepare_exact_search_runtime(...)` is the canonical local
  same-process multi-caller path
- `kayak_engine.prepare_exact_search_scheduler(...)` remains a compatibility
  alias
- `python -m kayak_engine.server ...` is still the current hosted HTTP transport

## Read Next

- [Usage Patterns](usage-patterns.md) when you want the public `import kayak` path
- [API Reference](api.md) when you want the open SDK surface instead of hosted snapshot reuse
- [Storage + Search](storage-and-search.md) when your data lives in an external database instead of a hosted snapshot
