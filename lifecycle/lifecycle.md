# Model Lifecycle — Personalized Recommendations

## End-to-End Lifecycle Diagram

```mermaid
flowchart TD
    A[Data Engineering\nSpark / dbt Pipeline] -->|feature table refreshed| B

    subgraph TRAIN["Training Phase"]
        B[Training Trigger\nCI/CD + cron] -->|dataset version| C[Model Training\nSageMaker Job]
        C -->|evaluation metrics| D{Gate P1\nOffline Metrics}
        D -->|NDCG@10 ≥ 0.42\nHR@10 ≥ 0.35| E[Register Model\nMLflow — Staging]
        D -->|fail| FAIL1[❌ Build fails\nSlack notification]
    end

    subgraph STAGING["Validation Phase"]
        E --> F[Integration Test\nCanary deployment 1%]
        F -->|p95 ≤ 120 ms\nerror rate ≤ 0.1%| G{Gate P2\nOnline Shadow}
        G -->|no A/B metric drift| H[Human Approval\nML Lead sign-off]
        G -->|fail| FAIL2[❌ Promotion blocked]
        H -->|approve| I[Registry: Staging → Production]
        H -->|reject| FAIL3[❌ Sent back for revision]
    end

    subgraph PROD["Production Phase"]
        I --> J[CI/CD Image Build\nbake model + code]
        J --> K[Canary Deploy\n10 % traffic]
        K -->|monitor 30 min| L{Burn-rate Alert?}
        L -->|none| M[Full Rollout\n100 % traffic]
        L -->|fired| N[Automatic Rollback\nprevious image]
        M --> O[Monitoring\nPrometheus + Grafana]
    end

    subgraph RETIRE["Retirement Phase"]
        O -->|model drift detected| P[Retrain trigger]
        P --> B
        M -->|90 days later| Q[Registry: Production → Archived]
    end
```

---

## Phase-by-Phase Explanation

### 1. Training Trigger

A new model is trained in two situations:
- **Cron**: every Monday at 02:00 UTC (fixed schedule).
- **Drift alert**: when Evidently AI detects feature distribution drift (PSI > 0.2 —
  defined in `monitoring/alerts.yaml`).

Weekly retraining is sufficient because retail user behavior changes slowly. Daily retraining
is not cost-justified by the volume of new data. Drift alerts handle flash-sale or seasonal
spikes that outpace the weekly cadence.

### 2. Offline Metric Gates (P1)

After training completes, the following checks run **automatically**:

```
NDCG@10         ≥ 0.42    # ranking quality
HR@10           ≥ 0.35    # top-10 hit rate
Coverage@1000   ≥ 0.60    # fraction of catalog surfaced
ColdStart_HR@10 ≥ 0.20    # hit rate for users with no history
```

Failure on any gate prevents registration in the model registry. Cold-start metrics are
tracked separately because aggregate metrics can mask poor cold-start performance when
history-rich users dominate the evaluation set.

### 3. Online Shadow / Canary Gate (P2)

A 1 % canary deployment runs for 30 minutes with real traffic:

```
p95 latency            ≤ 120 ms
error rate             ≤ 0.1 %
X-Model-Version header present
feature lookup miss rate ≤ 5 %
```

Why is online testing needed when offline metrics are available? Offline tests measure model
quality. Online tests catch: incorrect artifact loading, Redis connectivity issues,
serialization bugs, and real production latency (not lab estimates).

### 4. Human Approval Gate

The ML Lead (or a designated delegate) must sign off. This gate reviews:
- Offline + online metric results side by side
- The A/B test plan for the rollout
- Confirmation that a rollback plan is in place (`runbooks/rollback.md`)

This gate is not automated because the "is this model good enough?" judgment involves
business metrics (conversion, revenue, user satisfaction) that require human interpretation.

### 5. Canary Rollout to Production

Before 100 % rollout, 10 % of production traffic is sent to the new model for 30 minutes.
A burn-rate alert during this window triggers an automatic rollback. If no alert fires,
traffic is shifted to 100 %. The 30-minute window provides 6 alert evaluation cycles (alerts
use a 5-minute window — see `monitoring/alerts.yaml`), sufficient for statistical confidence.
