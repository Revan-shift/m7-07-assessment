# =============================================================================
# Multi-Stage Dockerfile — Recommendation Service
# =============================================================================
# WHY MULTI-STAGE?
# Build-time dependencies (gcc, cmake, etc.) must never appear in the final
# image. Multi-stage builds produce a final layer that contains only runtime
# artifacts, reducing image size by ~40 % and shrinking the attack surface.
# =============================================================================

# ---------------------------------------------------------------
# STAGE 1: builder
# Purpose: compile Python dependencies into wheels.
# This stage is NOT shipped in the final image.
# ---------------------------------------------------------------
FROM python:3.11-slim AS builder

# Build toolchain — only needed in this stage.
# gcc / g++ are required because faiss-cpu contains C++ extensions.
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build

# Copy requirements first to exploit Docker layer caching.
# If requirements.txt is unchanged, pip wheel is not re-executed —
# saving ~3 minutes per build.
COPY requirements.txt .

# Compile into wheels; do not install yet.
# --no-deps: each package declares its own deps; avoids conflicts.
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /wheels -r requirements.txt

# ---------------------------------------------------------------
# STAGE 2: final
# Purpose: minimal, secure, production-ready runtime image.
# Only this stage is pushed to ECR and run in Kubernetes.
# ---------------------------------------------------------------
FROM python:3.11-slim AS final

# OCI labels: enables search, filtering, and audit in the image registry.
# X-Model-Version value (ARG MODEL_VERSION) corresponds to the
# com.company.model.version label, which is cross-checked by the
# ModelVersionMismatch alert in monitoring/alerts.yaml.
ARG MODEL_VERSION
ARG GIT_SHA
LABEL org.opencontainers.image.source="https://github.com/company/recommendation-service" \
      org.opencontainers.image.revision="${GIT_SHA}" \
      com.company.model.version="${MODEL_VERSION}" \
      com.company.component="recommendation-service"

# Runtime system libraries only — no build tools.
# libgomp1: required by faiss-cpu for OpenMP parallel computation.
RUN apt-get update && apt-get install -y --no-install-recommends \
    libgomp1 \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# SECURITY: never run as root.
# A root process inside a container has full host access if a container
# escape occurs. A dedicated non-root user limits the blast radius.
RUN groupadd --system --gid 1001 appuser \
    && useradd --system --uid 1001 --gid appuser --no-create-home appuser

WORKDIR /app

# Pull compiled wheels from the builder stage — no build toolchain needed.
COPY --from=builder /wheels /wheels

# Install from local wheels (no network access required in this step).
RUN pip install --no-cache-dir --no-index --find-links=/wheels /wheels/*.whl \
    && rm -rf /wheels

# Application source code and configuration.
COPY --chown=appuser:appuser src/    /app/src/
COPY --chown=appuser:appuser config/ /app/config/

# MODEL ARTIFACT — baked into the image per ADR-0002.
# --chown ensures appuser can read but not write the model files.
# Contents of model/:
#   weights.pt         ~280 MB  (Two-Tower PyTorch weights)
#   faiss_index.bin    ~120 MB  (pre-computed item embeddings, IVF1024)
# Total: ~400 MB — consistent with the size estimate in container/README.md
COPY --chown=appuser:appuser model/ /app/model/

# Drop to non-root user for all subsequent operations.
USER appuser

# The service listens on port 8000 (HTTP).
EXPOSE 8000

# HEALTHCHECK
# Kubernetes liveness and readiness probes call this endpoint.
# --start-period=15s: probe does not start until 15 s after container launch,
# giving the model (~8 s) and FAISS index time to load fully.
# Without this, a pod could be marked Ready before serving is available.
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Entrypoint: Gunicorn managing Uvicorn worker processes.
# WHY GUNICORN + UVICORN?
#   Uvicorn handles async FastAPI efficiently.
#   Gunicorn manages multiple worker processes and can restart a crashed worker.
# --workers 2: 2 workers per pod on 4 vCPU (other 2 vCPUs serve FAISS ANN).
# --worker-class uvicorn.workers.UvicornWorker: async support.
# --timeout 30: requests exceeding 30 s are killed (prevents stuck workers).
CMD ["gunicorn", "src.main:app", \
     "--workers", "2", \
     "--worker-class", "uvicorn.workers.UvicornWorker", \
     "--bind", "0.0.0.0:8000", \
     "--timeout", "30", \
     "--graceful-timeout", "10", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
