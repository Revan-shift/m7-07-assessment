# ADR 0001 — Two-Tower Embedding Model as Serving Architecture

**Date**: 2026-06-07
**Status**: Accepted
**Deciders**: ML Platform Lead, Recommendation Team

---

## Context

The system must serve 800 RPS at peak with p95 ≤ 120 ms end-to-end. A/B experiments run
continuously, meaning model version changes must not introduce latency regressions. Cold-start
users (no history) must always receive a response. We need to choose a model architecture that
satisfies these constraints at inference time.

---

## Decision

We adopt a **Two-Tower Embedding model**.

- **User Tower**: encodes the user's last-30-day history into a dense embedding vector at
  request time.
- **Item Tower**: produces a static embedding vector for every product (pre-computed nightly).
- **At inference time**: only the User Tower runs; its output is then searched against the
  pre-computed item index using Approximate Nearest Neighbor (FAISS IVF).

---

## Reasoning

### 1. Latency profile fits the budget

Item embeddings are recomputed nightly and loaded into an in-memory FAISS index inside each
serving pod. At inference time only two operations execute:

- User Tower forward pass: ~20 ms on CPU (128-dim embedding)
- ANN search against 500 K items (FAISS IVF top-50): ~15 ms

Total model inference ~35 ms — well within the 120 ms end-to-end budget.

### 2. Cold-start is handled natively

When a user has no history, the User Tower receives a zero vector or a population-median
vector. No separate cold-start model is needed — the existing architecture handles it
internally. A popularity-cache fallback is also retained as a secondary safety net.

### 3. A/B experiments are straightforward

A new model version runs in a separate Kubernetes deployment. The gateway routes 10 % of
traffic to the canary. Both models receive the same features for the same `user_id`, so
results are directly comparable.

---

## Rejected Alternatives

### Matrix Factorization (SVD, ALS)
- Latency: acceptable (~10 ms).
- **Problem**: zero embedding for cold-start users — no answer without a second model.
- **Problem**: new catalog items have no matrix entry ("item cold-start"). Critical for retail.

### Sequence Model (Transformer / BERT4Rec)
- Higher accuracy (encodes longer history).
- **Problem**: inference time ~200–400 ms on CPU — exceeds budget.
- **Problem**: model size 1–4 GB — pod startup time exceeds 30 s; GPU required (3–4× cost).

### Collaborative Filtering (KNN)
- Simple and interpretable.
- **Problem**: real-time KNN at 800 RPS does not scale. The full user-item matrix must reside
  in memory (~GBs) and be scanned per request.

---

## Consequences

- Item embeddings must be recomputed nightly and the FAISS index reloaded into serving pods.
- When the model version changes, the index changes too — a rolling deploy protocol is required.
- FAISS index memory per pod: 500 K items × 128 dim × 4 bytes ≈ **256 MB** — 3.2 % of the
  8 GB pod memory limit. Acceptable.

---

## Future Path

Migration to a two-stage architecture (retrieval + ranker) is possible:
- Two-Tower retrieves top-500 candidates.
- A separate cross-encoder re-ranks to top-10.
- This adds ~30–40 ms to the latency budget and would justify revisiting GPU inference.
