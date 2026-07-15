# Capacity Plan — Recommendation Service

## Target Parameters

These figures are the single source of truth for both this document and `serving/slos.yaml`.
Any discrepancy between the two files is a bug.

| Parameter | Value | Source |
|---|---|---|
| Peak RPS | 800 req/s | Scenario definition |
| p95 latency budget | 120 ms | Scenario definition |
| p99 latency target | 200 ms | Buffered above SLO |
| Availability SLO | 99.9 % | slos.yaml |

---

## Single-Pod Performance Profile

**Hardware**: AWS c6i.xlarge (4 vCPU, 8 GB RAM) — $0.17/hr

How much traffic can one pod handle?

```
Model inference (p50):    ~35 ms   (User Tower forward pass + FAISS ANN top-50)
Feature Store lookup (p50): ~5 ms  (Redis, same region)
Serialization + network:  ~10 ms
────────────────────────────────────
Single request (p50):     ~50 ms

Workers per pod: 2 (Gunicorn)
Each worker handles: 1 request at a time (synchronous path)

Theoretical pod throughput at p50:
  2 workers × (1000 ms / 50 ms) = 40 req/s per pod

At p95 (tail latency):
  inference: ~70 ms, lookup: ~10 ms, other: ~20 ms → ~100 ms total
  2 workers × (1000 ms / 100 ms) = 20 req/s per pod
```

---

## Replica Math

**Goal**: serve 800 RPS with p95 ≤ 120 ms.

```
Pessimistic estimate (p95 throughput):
  800 RPS ÷ 20 req/s/pod = 40 pods

Optimistic estimate (p50 throughput):
  800 RPS ÷ 40 req/s/pod = 20 pods

Mid-point estimate (p75 throughput, ~70 ms):
  2 workers × (1000/70) ≈ 28 req/s
  800 ÷ 28 ≈ 29 pods

With 1.4× safety factor:
  29 × 1.4 ≈ 41 → rounded to 40 pods at full peak load
```

We do not run 40 pods continuously — HPA handles scaling.

### HPA Configuration

```yaml
minReplicas: 12
# Rationale: off-peak traffic is ~30 % of peak (240 RPS).
# 240 ÷ 28 ≈ 9 pods + buffer = 12 pods minimum.
# Fewer than 12 risks SLO breach during morning ramp-up
# before HPA has time to react (60–90 s scale-up lag).

maxReplicas: 50
# Headroom above the 40-pod peak estimate for unexpected spikes.

targetCPUUtilizationPercentage: 70
# At 70 % CPU, there is enough headroom to absorb in-flight requests
# while new pods start up.

scaleUpStabilizationWindowSeconds: 0
# React immediately to sudden load spikes (e.g. push notification).

scaleDownStabilizationWindowSeconds: 300
# Wait 5 minutes before removing pods to avoid flapping.
```

### Memory Budget Per Pod

```
Base Python runtime:        ~150 MB
FastAPI + dependencies:      ~80 MB
PyTorch Two-Tower weights:  ~280 MB
FAISS IVF index (in RAM):   ~256 MB  (500 K × 128 dim × 4 B)
OS + overhead:               ~50 MB
────────────────────────────────────
Total per worker:           ~816 MB

2 Gunicorn workers × ~900 MB = ~1.8 GB
OS page sharing reduces shared FAISS to ~350 MB → effective total ~1.4 GB
Pod memory limit: 8 GB → 17.5 % utilization. Comfortable buffer.
```

---

## Hardware Selection: CPU vs GPU

**Decision: CPU (c6i.xlarge)**

| Hardware | Type | Inference (ms) | $/hr | 12-pod monthly |
|---|---|---|---|---|
| c6i.xlarge | CPU 4vCPU/8GB | ~35 ms p50 | $0.170 | ~$1,469 |
| g4dn.xlarge | GPU T4 | ~8 ms p50 | $0.526 | ~$4,545 |
| g5.xlarge | GPU A10G | ~4 ms p50 | $1.006 | ~$8,692 |

**Why GPU is unnecessary:**
p50 CPU inference is 35 ms, which consumes only 29 % of the 120 ms budget. Switching to GPU
(~8 ms) would shift the bottleneck from model inference to network + Feature Store, which
together contribute ~20 ms and cannot be reduced further without architecture changes. GPU
delivers no SLO benefit here and costs 3–5× more. The savings are better spent on the
Feature Store or offline pipeline.

---

## Monthly Cost Estimate

```
Inference cluster (steady state, 12 pods):
  12 × c6i.xlarge × $0.170/hr × 730 hr = $1,489/mo

HPA scale-out (average +8 pods during peak hours ~6 hr/day):
  8 × $0.170 × (6 hr × 30 days) = $244/mo

Redis Cluster (3× r6g.large @ $0.146/hr):
  3 × $0.146 × 730 = $320/mo

Application Load Balancer:
  ~$100/mo

ECR image storage + data transfer:
  ~$30/mo

────────────────────────────────────────
Total monthly estimate: ~$2,183/mo
Conservative (includes sustained peak):  ~$3,000/mo
```

---

## Spike Behavior Analysis

800 RPS peaks arrive suddenly following push notifications:

```
t=0 s:    800 RPS spike begins
t=0–60s:  12 pods carry ~67 req/s each → CPU ~85 % → HPA triggered
t=60–90s: 8 new pods starting (8 s model load + scheduling)
t=90s+:   20 pods → ~40 req/s each → CPU ~55 % → SLO restored
```

The 60–90 s window may produce some p95 > 120 ms responses. This is acceptable because:
1. It is transient, not sustained.
2. SLOs are measured over 30-day windows; a 90 s spike does not exhaust the error budget.
3. If notification schedules are known in advance, proactive pre-scaling eliminates the gap.
