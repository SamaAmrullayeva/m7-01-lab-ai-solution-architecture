# ADR 0001: Serve Features from a Pre-Materialized Online Feature Store, Not Live DB Queries

## Context

The fraud scoring service needs account history and device fingerprint for every transaction. These features live in a data warehouse updated by an hourly Spark job. The scoring service must return a decision in under 80 ms p95 at 300 TPS peak, meaning feature retrieval must complete in roughly 10 ms.

## Decision

We will pre-materialize features into a low-latency key-value store (Redis or DynamoDB) on each hourly batch run. The scoring service reads from this store at inference time using two parallel key lookups (account_id, device_id). Feature freshness is accepted as up-to-1-hour stale. The feature engineering job is responsible for writing consistent, versioned snapshots; the serving layer is read-only.

## Alternatives Rejected

- **Live query against the data warehouse (Athena/Redshift):** Cold query latency is 200 ms–2 s, an order of magnitude over budget. Caching individual query results produces inconsistent feature values depending on cache hit rate, and cache misses under novel account IDs would breach the latency SLA on exactly the transactions most likely to be fraud.

- **Real-time feature computation inside the scoring service:** Computing features on-the-fly (e.g., aggregating the last 30 transactions at inference time) requires the scoring service to hold a DB connection, adds 50–150 ms of aggregation latency, and couples the serving and storage tiers. Under load it would saturate the transaction database.

- **Streaming feature updates via Kinesis → Redis (sub-minute freshness):** Would reduce feature staleness from ~1 hour to ~30 seconds, but requires a streaming feature pipeline that the team does not currently operate. The marginal fraud-detection improvement from fresher features does not justify the operational cost at this stage; this is a clear revisit candidate as volume grows.

## Consequences

- **Good:** Feature lookups are predictable O(1) reads with p99 < 5 ms; serving latency is decoupled from warehouse query performance.
- **Good:** Feature engineering logic lives in one place (the Spark job); the serving layer cannot drift from the training feature set.
- **Bad:** Features are up to 1 hour stale. A fraudster who successfully completes one transaction and immediately attempts a second within the same hour may not be caught by account-history features until the next batch write.
- **Bad:** We now operate a second data store (Redis/DynamoDB) whose consistency with the warehouse must be monitored.

## Revisit If

Transaction volume grows above ~2,000 TPS or the fraud team demonstrates that sub-minute feature freshness materially improves recall — at that point the streaming feature pipeline becomes cost-justified.
