# Architecture Justification & Trade-off Analysis

## Chosen Pattern: Synchronous Two-Tower Serving + External Feature Store

### What Were the Alternatives?

Three classical architectures exist for recommendation systems:

| Pattern | Latency | Complexity | A/B Support | Decision |
|---|---|---|---|---|
| **Real-time inference** (chosen) | Low (online) | Medium | Strong | ✅ Chosen |
| **Precomputed batch** | Very low | Low | Weak | ❌ Rejected |
| **Two-stage (retrieval + ranking)** | Medium | High | Strong | ⚠️ Future |

### Why Precomputed Batch Was Rejected

In a precomputed approach, recommendations are generated nightly, written to Redis/DB, and
inference is a simple key lookup. Advantage: p95 < 5 ms.

Reason for rejection: A/B testing becomes impractical. Serving two models simultaneously to
different user segments requires either two separate tables (2× storage) or complex routing
logic. Additionally, real-time behavioral signals (last 1 hour of activity) cannot be reflected
in the recommendation — a meaningful conversion loss in retail.

### Why Two-Stage Is Deferred

Two-stage: first retrieve 1,000 candidates via ANN, then re-rank to top-10 with a more
precise cross-encoder model. This is unnecessarily complex for an MVP. It requires two separate
models, two services, and two latency budgets. The current 120 ms budget is comfortably met
with a simple Two-Tower model. The ranker can be added in a later iteration.

---

## Feature Store Decision: Redis vs Alternatives

| Option | Latency | Freshness | Cost | Decision |
|---|---|---|---|---|
| **Redis Cluster** | <5 ms | Hourly | Medium | ✅ |
| DynamoDB | 5–15 ms | Hourly | Low | ❌ |
| Feast (online store) | <5 ms | Config-dependent | High | ❌ |
| PostgreSQL | 20–50 ms | Real-time | Low | ❌ |

Redis is chosen because:
1. We have only 5 ms of Feature Store budget — PostgreSQL destroys this budget.
2. DynamoDB cold-read latency can reach 30 ms at p99.
3. Feast adds infrastructure complexity that is unnecessary for MVP.

---

## Sync vs Async Endpoint Decision

The scenario specifies p95 = 120 ms. At this latency level, an async callback pattern makes
no sense because:
- Would the client open a WebSocket and wait? → More complex, higher latency.
- Would the client poll? → Higher latency, more requests.

We use a **sync endpoint** as the primary serving path.

The **batch endpoint** (POST /v1/recommend/batch) is provided separately for marketing
campaigns and nightly precompute jobs — a distinct use case.

The **async endpoint** (POST /v1/recommend/async) is included for completeness of the API
contract (required by the OpenAPI spec). It would handle arbitrarily large user sets. In
practice fewer than 0.1 % of traffic will reach this endpoint.

---

## Kubernetes vs ECS vs Serverless

| Platform | Cold Start | Scale Speed | Cost | Decision |
|---|---|---|---|---|
| **Kubernetes (EKS)** | None (warm) | 60–90 s | Medium | ✅ |
| ECS Fargate | None (warm) | 90–120 s | Medium+ | ❌ |
| Lambda/Serverless | 2–8 s | Instant | Variable | ❌ |

Lambda rejected because:
- 400 MB model artifact exceeds Lambda's 250 MB unzipped limit.
- Cold-start latency of 2–8 s violates the 120 ms p95 budget.
- At 800 RPS, provisioned concurrency cost exceeds EKS.

ECS rejected because:
- Kubernetes HPA, PDB, and service mesh ecosystem are more capable.
- No migration cost if the team already runs Kubernetes.
