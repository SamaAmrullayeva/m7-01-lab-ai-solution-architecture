# JUSTIFICATION: Real-Time Fraud Scoring

## Serving Pattern: Online (Synchronous Request-Response)

The scenario states a **p95 latency budget of 80 ms end-to-end** and a requirement that the decision (block / allow / step-up-auth) must reach the user *before they see confirmation*. That rules out batch — a pre-computed score would be stale by the time a new transaction arrives — and rules out streaming with async callback, because the payments app needs a synchronous answer to show the user. Online serving is the only pattern that closes the loop within a single HTTP request.

Streaming inference (e.g., Kafka → model consumer → response topic) would add at least one broker round-trip (~5–15 ms) plus consumer lag, blowing the budget before any model work happens. Batch pre-scoring is impossible for novel device/account combinations and would require a per-account re-score trigger anyway, collapsing back into online serving under load.

## Where Inference Runs: Cloud (Single-Region, Multi-AZ)

Inference runs in cloud, not at the edge. The fraud model requires two real-time feature lookups — account history and device fingerprint — that live in a managed feature store (Redis/DynamoDB). Shipping those feature stores to edge nodes would add significant operational cost, introduce cache coherence problems, and require a larger model footprint than edge hardware typically supports. A compact ONNX-exported model (~50–200 MB) running on cloud inference pods keeps the serving stack colocated with the feature store and the data warehouse, minimising lateral network hops.

The tradeoff: users in geographically distant regions pay a few extra milliseconds for the round trip to the nearest cloud region. At 300 TPS, horizontal scaling of inference pods is cheaper than the operational burden of edge deployment.

## Latency, Throughput, and Cost Targets

**Optimize for:** p95 latency ≤ 80 ms and throughput ≥ 300 TPS peak.

**Budget (accept worse):** infrastructure cost. The cost of a missed fraud event or a falsely blocked transaction far exceeds the cost of over-provisioned inference pods. We provision for 2× peak (600 TPS) to absorb traffic spikes without autoscale lag.

Latency breakdown target:
- API Gateway + network: ~5 ms
- Feature Store lookups (2× parallel): ~10 ms
- ONNX inference: ≤ 10 ms
- Decision logic + response serialization: ~5 ms
- **Total budget used: ~30 ms**, leaving 50 ms margin for p99 tail and cross-AZ variance.

## Fallback When the Model is Unavailable or Wrong

Two failure modes require distinct responses:

**Model service down (timeout / 5xx):** The Fraud Scoring Service falls back to a rules-based engine — a lightweight set of hard-coded signals (transaction amount > 3σ from account mean, new device + high-value txn, geographic anomaly). This adds ~2 ms and produces a coarser score. The fallback is always-on and shadowed in production, so it is never cold.

**Model returns a wrong score (silent error):** This is the harder problem. Chargebacks from external processors feed back into the data warehouse as ground-truth labels within 30–90 days. The monitoring layer (Evidently) tracks score distribution in real time; a shift in the score histogram triggers a drift alert to the training pipeline. Additionally, the step-up-auth action (MFA challenge) provides a low-cost hedge: for borderline scores (0.4–0.7), the system requests additional verification rather than hard-blocking or silently allowing, reducing the blast radius of model errors on both sides.
