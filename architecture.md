# Architecture: Real-Time Fraud Scoring (Scenario A)

```mermaid
flowchart TD
    subgraph Online["🟢 Online (request path, &lt;80ms budget)"]
        A[Payments App] -->|transaction request| B[API Gateway / Load Balancer]
        B -->|enriched request| C[Fraud Scoring Service]
        C -->|account_id lookup| D[(Feature Store\nRedis / DynamoDB)]
        C -->|device_id lookup| D
        C -->|score request| E[Model Serving\nclaude-fraud-v2, ONNX]
        E -->|risk_score 0–1| C
        C -->|block / allow / step-up| B
        B -->|decision + score| A
    end

    subgraph Batch["🔵 Batch / Offline (background)"]
        F[(Data Warehouse\nS3 + Athena)] -->|labeled txn history| G[Training Pipeline\nSpark + SageMaker]
        G -->|trained artifact| H[Model Registry\nMLflow]
        H -->|promote candidate| E
        I[Feature Engineering\nSpark job, hourly] -->|account features\nbatch write| D
    end

    subgraph Feedback["🔴 Feedback Loop"]
        C -->|decision log + score| J[Event Bus\nKinesis]
        J -->|raw events| F
        J -->|realtime signal| K[Monitoring\nCloudWatch + Evidently]
        K -->|drift alert| G
        L[Chargeback Feed\nexternal] -->|ground truth labels| F
    end
```

## Serving Boundary

| Layer | Pattern | Latency |
|---|---|---|
| Fraud Scoring Service + Model Serving + Feature Store | **Online** | &lt;80ms p95 |
| Feature Engineering, Training Pipeline | **Batch** | hourly / weekly |
| Decision Log → Warehouse → Retraining | **Async feedback** | hours–days |

## Component Glossary

| Component | Role |
|---|---|
| **API Gateway** | Auth, rate-limiting, routes to scoring service |
| **Fraud Scoring Service** | Orchestrates feature lookup + model call + decision logic |
| **Feature Store (Redis/DynamoDB)** | Low-latency key-value store for account history & device fingerprint |
| **Model Serving (ONNX)** | Stateless inference pod; ONNX runtime for &lt;10ms inference |
| **Model Registry (MLflow)** | Tracks versions, promotes candidates to production |
| **Event Bus (Kinesis)** | Decouples decision logging from downstream consumers |
| **Monitoring (Evidently)** | Tracks score distribution drift, triggers retraining alert |
| **Training Pipeline** | Full retrain on labeled transactions + chargeback ground truth |
