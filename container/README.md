# Container Image Plan — Recommendation Service

## Bake-in vs Runtime Mount Decision

**Decision: model artifact is baked into the Docker image.**

The full rationale is in `architecture/adr/0002-bake-vs-mount.md`. Short summary:
rollback is a single command, pod startup is 8 s, and the audit trail is automatic.

---

## Base Image Selection

`python:3.11-slim` is chosen.

| Alternative | Size | Why Rejected |
|---|---|---|
| `python:3.11` (full) | ~900 MB | Contains unnecessary build tools and compilers |
| `python:3.11-slim` ✅ | ~125 MB | Minimal; pip install works correctly |
| `python:3.11-alpine` | ~50 MB | musl libc — binary incompatibility with faiss-cpu wheels |
| `gcr.io/distroless/python3` | ~50 MB | No pip; build complexity too high |

Alpine is rejected because faiss-cpu PyPI wheels are linked against glibc, which Alpine
replaces with musl libc — a direct ABI incompatibility.

---

## Image Size Estimate

| Layer | Size | Notes |
|---|---|---|
| python:3.11-slim base | ~125 MB | OS + Python runtime |
| Runtime system libs | ~5 MB | libgomp1, curl |
| Python deps (wheels) | ~95 MB | torch-cpu, faiss-cpu, fastapi, gunicorn, redis, prometheus-client |
| Application code | ~2 MB | src/, config/ |
| Model weights (PT) | ~280 MB | Two-Tower PyTorch weights |
| FAISS index | ~120 MB | 500 K items × 128 dim × 4 B ≈ 256 MB raw; ~120 MB after IVF compression |
| **Total (uncompressed)** | **~627 MB** | ECR push ~400 MB (gzip) |

**Hard limit: ≤ 700 MB** — enforced by the CI/CD pipeline (see `deploy-model.yml`).

---

## Key Dependencies (requirements.txt)

```
fastapi==0.111.0          # async web framework
uvicorn[standard]==0.29.0 # ASGI server
gunicorn==22.0.0          # process manager
torch==2.3.0+cpu          # PyTorch CPU-only — GPU not required (ADR-0001)
faiss-cpu==1.8.0          # ANN search
redis==5.0.4              # Feature Store client
prometheus-client==0.20.0 # metrics endpoint
pydantic==2.7.0           # request/response validation
structlog==24.1.0         # structured JSON logging
```

---

## Security Scanning

Every image build in CI/CD triggers two scans before the image is pushed to ECR:
1. **Hadolint** — Dockerfile linting for best-practice violations.
2. **Trivy** — CVE scan; CRITICAL or HIGH vulnerabilities fail the build.

See the `security-scan` job in `cicd/.github/workflows/deploy-model.yml`.

---

## Runtime Behavior

- **Startup**: model load + FAISS index into RAM ≈ 8 s
- **Memory per pod**: base ~500 MB + FAISS index ~256 MB + inference buffer ≈ **~900 MB**
  (11.25 % of the 8 GB pod limit — comfortable buffer)
- **Worker memory**: 2 Gunicorn workers × ~900 MB = ~1.8 GB active; OS page sharing
  reduces the FAISS portion to ~350 MB shared in practice
- **Model version propagation**: the `MODEL_VERSION` build arg flows into the
  `com.company.model.version` OCI label and into the `X-Model-Version` response header
  (cross-checked by the `ModelVersionMismatch` alert in `monitoring/alerts.yaml`)

---

## Image Tagging Convention

```
# Immutable tag — used for rollback
123456789.dkr.ecr.eu-west-1.amazonaws.com/recommendation-service:v1.5.0-git-a3f9c12

# Mutable tags — tracked by Kubernetes deployments
123456789.dkr.ecr.eu-west-1.amazonaws.com/recommendation-service:stable
123456789.dkr.ecr.eu-west-1.amazonaws.com/recommendation-service:canary
```

`stable` and `canary` are mutable; deployments reference them.
`v<semver>-git-<sha>` is immutable; rollback operations reference this tag directly.
Tag format matches `lifecycle/model-registry.yaml` `image_tag_format` field.
