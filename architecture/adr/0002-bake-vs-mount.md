# ADR 0002 — Model Artifact: Bake Into Image vs Runtime Mount

**Date**: 2026-06-07
**Status**: Accepted
**Deciders**: ML Platform Lead, Platform Engineering

---

## Context

The inference service is packaged as a Docker image. The model artifact (~400 MB: Two-Tower
weights + FAISS index) must be co-located with the serving code at runtime. Two primary
strategies exist.

---

## Decision

**The model artifact is baked into the Docker image** (via `COPY` in the Dockerfile).

---

## Options Compared

| Criterion | Bake-in | Runtime Mount (S3 / PVC) |
|---|---|---|
| Deployment atomicity | ✅ Image = complete unit | ❌ Image + mount are separate |
| Rollback speed | ✅ One `kubectl set image` command | ❌ Must revert mount separately |
| Pod startup time | ✅ ~8 s (model already present) | ❌ ~40–60 s (S3 download) |
| Image size | ❌ ~700 MB (base + model) | ✅ ~300 MB (code only) |
| Model update requires new image | ❌ Yes | ✅ No |
| Audit trail | ✅ Image tag = exact model version | ⚠️ Requires separate tracking |

---

## Reasoning

### 1. Rollback speed is critical for on-call reliability

When a bad model is deployed, the on-call engineer must revert with a single command:

```bash
kubectl set image deployment/recommendation-service \
  app=recommendation-service:v1.23.0-git-abc1234
```

This atomically reverts both code and model to the previous state. With runtime mounting:

```bash
# Step 1: roll back the Deployment
kubectl rollout undo deployment/recommendation-service
# Step 2: manually update the S3 model pointer
aws s3 cp s3://models/v1.23/model.pkl s3://models/current/model.pkl
# Step 3: restart pods to re-fetch
kubectl rollout restart deployment/recommendation-service
```

Three steps at 3 AM — the error surface is unacceptably wide.

### 2. Pod startup time affects the HPA scale-up window

HPA responds to a traffic spike in 60–90 s. If each new pod takes 60 s to download the model
before becoming ready, the scale-up window extends to 120–150 s — meaning existing pods carry
overload for longer and latency SLOs are at risk. With baked-in models, startup is ~8 s.

### 3. Atomic deployment simplifies the audit trail

Every CI/CD build produces one image tagged with both the semantic version and the git SHA:
`v<semver>-git-<sha>`. This single tag encodes the code version, model version, and
configuration simultaneously. Determining which model was live at any point in time requires
only a registry lookup — no separate logs to cross-reference.

---

## Why Runtime Mount Was Rejected Here

Runtime mount is preferable when:
- The model is very large (>2 GB) and image registry cost is a concern.
- The model changes multiple times per day — image build latency would become a bottleneck.
- Multiple code versions must run against different model versions simultaneously.

None of these apply here:
- Model is ~400 MB — manageable.
- Model is updated at most daily — build latency is not a bottleneck.
- Each model version is validated end-to-end before promotion — no mixed-version serving needed.

---

## Consequences

- `Dockerfile`: `COPY --chown=appuser:appuser model/ /app/model/`
- Image size: ~700 MB (python:3.11-slim ~200 MB + deps ~100 MB + model ~400 MB)
- Registry: AWS ECR; each image tagged `git-<sha>` and `v<semver>-git-<sha>`
- CI/CD: when a model is promoted in the registry, a new image build is triggered automatically
  (see `cicd/.github/workflows/deploy-model.yml`)
- Rollback: `kubectl set image` + previous semver tag — one command, one human action
