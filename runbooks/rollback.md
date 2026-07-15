# Rollback Runbook — Recommendation Service

**Purpose**: enable any on-call engineer to revert a bad deployment within 10 minutes
at any hour, without needing deep knowledge of the system.

---

## When to Roll Back

Initiate rollback if **any one** of the following conditions is true:

| Alert | Threshold | Defined in |
|---|---|---|
| `AvailabilityBudgetBurnFast` | Error rate > 1.44 % for 2 min | `monitoring/alerts.yaml` |
| `LatencyBudgetBurnFast` | p95 > 120 ms for 2 min | `monitoring/alerts.yaml` |
| `ModelVersionMismatch` | More than one version active for > 10 min | `monitoring/alerts.yaml` |
| `FeatureStoreDown` | Redis unreachable for > 1 min | `monitoring/alerts.yaml` |
| Manual decision | ML Lead or Engineering Manager judgment | — |

**When in doubt, roll back. Rollback is cheap. A prolonged incident is not.**

---

## Rollback Steps

### Step 1 — Identify the active and target versions (2 min)

```bash
# What is currently running?
kubectl get deployment recommendation-service-stable \
  -n recommendation-prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

kubectl get deployment recommendation-service-canary \
  -n recommendation-prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# List recent rollout history to find the last known-good version
kubectl rollout history deployment/recommendation-service-stable \
  -n recommendation-prod
```

☐ Active (bad) version recorded: `______________`
☐ Target (last known-good) version identified: `______________`

---

### Step 2 — Cut traffic to the old stable version (1 min)

**If a canary is active, remove it from the traffic split first:**

```bash
# Send 100 % of traffic to the stable (old) deployment
kubectl patch virtualservice recommendation-vs \
  -n recommendation-prod \
  --type=merge \
  -p '{"spec":{"http":[{"route":[{"destination":{"host":"recommendation-stable"},"weight":100}]}]}}'
```

☐ Traffic is 100 % on stable
☐ Grafana shows error rate starting to decrease

---

### Step 3 — Roll back the stable deployment (2 min)

```bash
# Option A: undo to the previous revision (fastest)
kubectl rollout undo deployment/recommendation-service-stable \
  -n recommendation-prod

# Option B: pin to a specific known-good immutable tag (more precise)
kubectl set image deployment/recommendation-service-stable \
  app=123456789.dkr.ecr.eu-west-1.amazonaws.com/recommendation-service:v1.4.2-git-abc1234 \
  -n recommendation-prod

# Wait for rollout to complete
kubectl rollout status deployment/recommendation-service-stable \
  -n recommendation-prod \
  --timeout=120s
```

☐ Rollback command executed
☐ Rollout completed successfully
☐ All pods show the previous image tag

---

### Step 4 — Verify recovery (3 min)

```bash
# Health check
curl -f https://api.company.com/health | jq .

# Confirm X-Model-Version matches the reverted version
# (ModelVersionMismatch alert should resolve within 1 minute)
curl -s -I -X POST https://api.company.com/v1/recommend \
  -H "Authorization: Bearer $PROD_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"user_id":"usr-smoke-test","k":1}' \
  | grep -i "x-model-version"

# Check live error rate from Prometheus
curl -s "http://prometheus:9090/api/v1/query" \
  --data-urlencode 'query=rate(http_requests_total{service="recommendation-service",status_code=~"5.."}[2m])/rate(http_requests_total{service="recommendation-service"}[2m])' \
  | jq '.data.result[0].value[1]'
```

☐ `/health` returns `{"status":"ok"}`
☐ `X-Model-Version` matches the rollback target
☐ Error rate < 0.1 % (slos.yaml availability objective)
☐ Alerts resolved in Alertmanager

---

### Step 5 — Communicate (2 min)

```bash
curl -X POST $SLACK_WEBHOOK_URL \
  -H 'Content-type: application/json' \
  -d '{
    "text": "🔄 Recommendation Service rollback complete.\nReverted to: <previous version>\nReason: <brief description>\nNext step: post-mortem to be scheduled"
  }'
```

☐ Slack notification sent to #ml-platform-incidents
☐ Incident ticket opened (e.g. JIRA REC-XXXX)

---

### Step 6 — Reset the canary deployment (1 min)

```bash
# Revert the canary deployment to the same known-good image
kubectl set image deployment/recommendation-service-canary \
  app=123456789.dkr.ecr.eu-west-1.amazonaws.com/recommendation-service:v1.4.2-git-abc1234 \
  -n recommendation-prod
kubectl rollout status deployment/recommendation-service-canary \
  -n recommendation-prod --timeout=60s
```

☐ Canary deployment also reverted

---

## After Rollback — Follow-up Actions (Business Hours)

- [ ] Root-cause analysis: identify which change caused the incident.
- [ ] Tag the bad image in ECR with `status=quarantined`.
- [ ] Downgrade the model in MLflow registry from `Production` back to `Staging`.
- [ ] Add a CI/CD block rule to prevent this image from being re-deployed.
- [ ] Complete a post-mortem within 5 business days (required by `serving/slos.yaml` error budget policy).
- [ ] When a fixed model version is ready, re-run the full lifecycle from `lifecycle/lifecycle.md`.

---

## Quick Reference — Minimum Commands

```bash
# 1. Cut traffic to stable
kubectl patch virtualservice recommendation-vs -n recommendation-prod \
  --type=merge -p '{"spec":{"http":[{"route":[{"destination":{"host":"recommendation-stable"},"weight":100}]}]}}'

# 2. Roll back stable
kubectl rollout undo deployment/recommendation-service-stable -n recommendation-prod

# 3. Confirm rollout finished
kubectl rollout status deployment/recommendation-service-stable -n recommendation-prod

# 4. Verify health
curl -f https://api.company.com/health
```

**Total time: ~10 minutes. If the problem persists after rollback, escalate to the Platform team.**
