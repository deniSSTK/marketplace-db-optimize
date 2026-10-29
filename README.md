# PostgreSQL High Load

A Go pet project exploring PostgreSQL under high load. Using a marketplace order service, I explore indexing, partitioning, sharding, concurrent access, and database operations.

The idea is to tackle concrete problems within a shared domain and validate solutions through measurements. Each experiment includes a reproducible workload, correctness checks, before-and-after results, and an explanation of the tradeoffs.

> **Status:** the project is in the design phase. This README describes the planned architecture and experiments. Implementation, run instructions, and benchmark results will be added as they become available.

## Domain

The service models a small part of a marketplace:

- Sellers and buyers.
- Orders and status changes.
- Order event history.
- Seller sales statistics.
- Background jobs for order processing.

Business logic is limited to what the experiments need. Operations must still preserve data correctness under concurrent requests, retries, and failures.

The dataset will include sellers of different sizes, uneven activity, and a long history of orders. This makes it possible to explore both typical workloads and cases where a popular seller receives a large share of requests.

## Architecture

A single backend exposes the same API across different storage configurations. The load generator runs the same user scenarios against each configuration.

```text
Data and load generator
            │
            ▼
        Go backend
            │
        Order logic
            │
        Data access
            │
            ├── Single PostgreSQL instance: baseline schema
            ├── Single PostgreSQL instance: optimization under test
            └── Multiple PostgreSQL instances: sharding by seller
```

Configurations run separately. The API contract stays the same across experiments, with execution behavior selected by configuration rather than separate `/slow` and `/fast` endpoints.

Changes are evaluated individually. Adding an index, rewriting a query, and introducing partitioning should not be bundled into a single comparison that obscures what caused the result.

### Stack

| Tool | Purpose |
| --- | --- |
| Go | Backend, data generation, and load scenarios |
| PostgreSQL | Data storage and query execution |
| pgx/v5 | PostgreSQL driver and connection pools |
| sqlc | Type-safe Go code generation from SQL |
| SQL migrations | Schema, index, and partition changes |
| Docker Compose | Running local environment configurations |
| PgBouncer | Connection pooling experiments |
| pg_stat_statements | SQL execution statistics |
| Prometheus and Grafana | Metrics collection and monitoring |

### SQL and sqlc

Queries are written by hand in `.sql` files. `sqlc` uses the schema and queries to generate Go structs and data access methods. This keeps SQL explicit and makes it possible to connect experiment results to a specific query and its execution plan.

`pgx` handles statement preparation at runtime according to connection settings. Warm-up before measurements is a separate part of the load scenario.

Shard routing lives in the application: it selects the appropriate connection pool before calling a generated method.

## API

Proposed operations:

```http
POST  /api/v1/orders
GET   /api/v1/orders/{order_id}
PATCH /api/v1/orders/{order_id}/status
GET   /api/v1/orders/{order_id}/events
GET   /api/v1/sellers/{seller_id}/orders
GET   /api/v1/sellers/{seller_id}/stats?from=...&to=...
```

Fetching an order by ID in sharded mode requires locating its shard. The lookup mechanism is part of the routing design; any additional query or shard fan-out must be included in measurements.

### Token-based pagination

Order and event lists use cursor pagination with an opaque continuation token:

```http
GET /api/v1/sellers/42/orders?limit=50
GET /api/v1/sellers/42/orders?limit=50&page_token=...
```

The response contains records and a `next_page_token` when another page is available. Clients pass the token back without interpreting its contents.

The token represents a position in a deterministic ordering, such as `(created_at, id)`, and is bound to the query parameters. The server validates its format, version, and compatibility with the request. The mechanism for protecting tokens against tampering will be chosen during implementation.

The experiment compares deep OFFSET pagination with cursor-based retrieval. Checks cover identical timestamps, records inserted between page requests, and updates to existing data. A cursor alone does not provide a snapshot of the entire result set as it existed at the first request.

## Experiments

### 1. Indexes and query plans

Scenarios include seller order history, filtering orders by status, events within a time range, and metadata searches.

- Composite B-tree indexes and column order.
- Partial indexes for frequently accessed subsets of data.
- B-tree versus BRIN on chronological data with different physical ordering.
- Expression indexes and GIN for suitable JSONB queries.
- The impact of indexes on inserts, updates, and storage usage.

Each case records the query, `EXPLAIN (ANALYZE, BUFFERS)`, and measurements under load. The baseline is documented explicitly: missing indexes should not be the sole basis for demonstrating a speedup.

### 2. Partitioning event history

Event history is split into time-based partitions. Experiments cover queries over short and long time ranges, along with history maintenance.

- Partition pruning compared with a regular indexed table.
- The impact of partition count and size.
- Automatic creation of future partitions.
- Late-arriving events and writes outside the prepared range.
- Dropping or detaching old partitions according to a retention policy.
- Uniqueness constraints and key selection in a partitioned table.

The goal is to identify when partitioning benefits queries or maintenance and measure its overhead.

### 3. Sharding by seller

Orders are distributed across independent PostgreSQL instances by `seller_id`. Related order data is kept on the same shard where possible.

- Shard key selection and query routing.
- Data and workload distribution across shards.
- A popular seller overloading a single shard.
- Buyer history and other queries spanning multiple shards.
- Timeouts, unavailable shards, and policies for partial results.
- The limits of local transactions and uniqueness across shards.

Events within each shard can also be partitioned by time. Sharding and partitioning are evaluated separately before being combined.

Multiple containers on one machine allow routing and failure behavior to be tested. Measurements in this setup do not establish scalability across independent nodes: shared CPU, disk, and memory must be considered when interpreting results.

### 4. Moving a seller between shards

Scenario: a large seller needs to move to another shard while the service continues running.

- Copying existing data and transferring changes made during the copy.
- Switching routing with explicit behavior for in-flight requests.
- Verifying the completeness and correctness of transferred data.
- Resuming after failure and defining rollback conditions.
- Safely removing the old copy after the move is complete.

The report records the impact on latency and availability, including any write pause. Zero downtime is not assumed in advance.

### 5. Concurrent updates

Multiple requests update the same order or compete for related resources.

- Explicit business invariants and validation of allowed status transitions.
- Comparing locking with record version checks.
- Deadlocks, lock acquisition order, and bounded transaction retries.
- Idempotency when clients retry requests.
- Workloads targeting separate records versus one hot record.

A successful experiment demonstrates both performance and preservation of invariants.

### 6. Background jobs in PostgreSQL

Multiple workers process jobs related to orders.

- Claiming jobs with `FOR UPDATE SKIP LOCKED`.
- Short claim transactions and processing state tracking.
- Recovering jobs after a worker crashes.
- Bounded retries, backoff, and recording terminal failures.
- Idempotent processing when jobs run more than once.

Checks also cover the boundary between committing a database result and producing an external side effect. Claiming a job alone does not guarantee that an external operation executes exactly once.

### 7. Aggregations and sales statistics

Aggregation summarizes many records into totals. For example, a seller's orders for a month can be used to calculate completed order counts, total sales, and average order value, grouped by day.

Three approaches are compared for the same statistics:

| Approach | How it works | What is measured |
| --- | --- | --- |
| Querying source orders | Runs `GROUP BY`, `COUNT`, `SUM`, and other calculations on request | Report latency and load on the primary database |
| Materialized view | Stores query results in the database and refreshes them periodically | Read performance, refresh cost, and data staleness |
| Aggregate table | Updates stored totals as order changes are processed | Maintenance cost, update lag, and correctness |

Refreshing a materialized view and incrementally updating an aggregate table are treated as distinct mechanisms.

Checks cover duplicate event processing, order cancellation, late changes, and rebuilding totals from source data. For distributed data, experiments also cover combining statistics from multiple shards.

Each approach defines an acceptable freshness level. A faster response containing stale statistics cannot be treated as equivalent to an up-to-date calculation without documenting that tradeoff.

### 8. Connection pools and replicas

- Application pooling and connections through PgBouncer.
- Connection limits, pool wait time, and overload behavior.
- Compatibility of prepared statement settings with the chosen PgBouncer mode.
- Reading history and reports from a replica.
- Replication lag and the case where a newly created order is not yet visible.
- Read-after-write policy and behavior when a replica is unavailable.

### 9. Migrations and maintenance under load

- Creating indexes on a running service and observing locks.
- Gradually backfilling a new column with support for resuming after interruption.
- Application and schema compatibility during changes.
- Autovacuum, statistics freshness, and table bloat from updates and deletes.
- Partition maintenance and the effect of background operations on user queries.

## Measurement methodology

Each experiment includes:

1. A problem statement, hypothesis, and correctness criteria.
2. Schema, queries, configuration, and dataset characteristics.
3. A workload description, including request distribution and concurrency.
4. Baseline measurements.
5. One change under evaluation.
6. Repeated measurements under comparable conditions.
7. Findings, limitations, and the cost of the improvement.

### Metrics

- Throughput: completed operations per second.
- Latency: p50, p95, and p99, including connection wait time.
- Errors, timeouts, and retries.
- CPU, memory, and disk I/O.
- Lock waits and deadlocks.
- Query statistics and execution plans.
- Table and index sizes.
- Replication or aggregate update lag, where applicable.

Runs record component versions, container resources, PostgreSQL settings, dataset size and random seed, warm-up and measurement durations, and repetition count. The initial data state is restored or explicitly accounted for when the workload modifies it.

Warm and cold scenarios are documented separately. Restarting the application is not treated as clearing PostgreSQL or operating system caches.

The load generator must track the target request rate, the actual load achieved, and its own limits so that a slowdown in the generator is not mistaken for a service improvement.

Speedups are evaluated alongside resource usage, write cost, result freshness, and operational complexity. No improvement is a valid experiment outcome.

## When an example is complete

An example is complete when it can be reproduced, verified, and explained:

- Startup, data preparation, and environment cleanup are documented.
- Necessary timeouts, limits, and error handling are implemented.
- Correctness is checked under concurrency and relevant failures.
- Recovery procedures and operational limitations are described.
- Results are saved together with measurement conditions.

These properties determine how closely an example reflects production concerns. A local setup alone does not establish readiness to operate with real data.

## Planned repository contents

- Backend with a shared API and data routing.
- SQL queries, sqlc configuration, and generated Go code.
- Database migrations and experiment configurations.
- Docker Compose configurations.
- A reproducible data and load generator.
- Monitoring configuration and dashboards.
- An experiment catalog with instructions, query plans, and reports.

Run commands, the actual directory structure, and measured results will be added as the corresponding components are implemented.
