# Load Test Plan — Recommendation Service

## Strategy

Load testing is structured in three progressive phases. Each phase has a distinct purpose.
Tool: **Grafana k6** (JavaScript-based, runs natively in Kubernetes).

SLO thresholds used in all tests are sourced directly from `serving/slos.yaml`:
- p95 ≤ 120 ms
- Error rate ≤ 0.1 % (derived from 1 − 0.999 availability SLO)

---

## Phase 1: Baseline Test (40 min)

**Purpose**: confirm that the SLO is met under sustained peak load.

```javascript
// k6/baseline.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '5m',  target: 200 },   // ramp-up: 0 → 200 RPS
    { duration: '30m', target: 800 },   // sustained peak: 800 RPS
    { duration: '5m',  target: 0   },   // ramp-down
  ],
  thresholds: {
    // Thresholds mirror slos.yaml exactly.
    'http_req_duration{endpoint:sync}': ['p(95)<120'],   // p95 ≤ 120 ms
    'http_req_failed':                  ['rate<0.001'],  // error rate < 0.1 %
    'errors':                           ['rate<0.001'],
  },
};

const BASE_URL = __ENV.TARGET_URL || 'https://api.staging.company.com';

// User mix: 20 % cold-start, 80 % users with history.
// Reflects the real new-user registration rate in the business.
function generateUserId() {
  return Math.random() < 0.2
    ? `usr-new-${Math.floor(Math.random() * 1_000_000)}`   // cold-start
    : `usr-${Math.floor(Math.random() * 100_000)}`;         // warm user
}

export default function () {
  const payload = JSON.stringify({
    user_id: generateUserId(),
    k: 10,
    context: { surface: 'home_screen', session_id: `sess-${Date.now()}` },
  });

  const params = {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.API_TOKEN}`,
    },
    tags: { endpoint: 'sync' },
  };

  const res = http.post(`${BASE_URL}/v1/recommend`, payload, params);

  const ok = check(res, {
    'status 200':              (r) => r.status === 200,
    'has X-Model-Version':     (r) => r.headers['X-Model-Version'] !== undefined,
    'recommendations present': (r) => {
      try { return JSON.parse(r.body).recommendations.length > 0; }
      catch { return false; }
    },
  });

  errorRate.add(!ok);
  sleep(0.001); // 1 ms sleep allows a single VU to sustain ~1000 RPS
}
```

**Why ramp to 800 RPS?** That is the scenario's stated peak. The test validates that p95
stays within the 120 ms budget at this specific load level. If it fails here, the replica
count or pod sizing is wrong.

---

## Phase 2: Spike Test (20 min)

**Purpose**: observe HPA reaction time and measure latency degradation during an instant
load spike (simulating a push notification broadcast).

```javascript
// k6/spike.js
export const options = {
  stages: [
    { duration: '2m',  target: 100  },   // quiet period
    { duration: '30s', target: 1200 },   // INSTANT SPIKE — 800 RPS × 1.5×
    { duration: '5m',  target: 1200 },   // spike sustained
    { duration: '2m',  target: 100  },   // return to baseline
    { duration: '10m', target: 100  },   // observe recovery
  ],
  thresholds: {
    // During a spike, p99 ≤ 500 ms is acceptable.
    // The 30-day SLO window absorbs transient spikes.
    'http_req_duration{endpoint:sync}': ['p(99)<500'],
    'http_req_failed':                  ['rate<0.01'],  // 1 % tolerated during spike
  },
};
```

**Metrics to observe**:
- How many seconds does HPA take to respond?
- How high does p95 climb during the 60–90 s scale-up window?
- Does latency return to baseline within 5 minutes of spike end?

---

## Phase 3: Stress / Breaking-Point Test (45 min)

**Purpose**: find the maximum RPS at which the SLO is still met. Validates the 1.4× safety
factor built into the replica math in `serving/capacity-plan.md`.

```javascript
// k6/stress.js
export const options = {
  stages: [
    { duration: '5m', target: 500  },
    { duration: '5m', target: 800  },    // nominal peak
    { duration: '5m', target: 1000 },
    { duration: '5m', target: 1200 },
    { duration: '5m', target: 1500 },    // SLO breach expected around here
    { duration: '10m', target: 0   },    // recovery
  ],
  thresholds: {
    'http_req_failed': ['rate<0.10'],    // relaxed — this test is exploratory
  },
};
```

**Expected outcome**: p95 > 120 ms begins around 1,000–1,100 RPS. This confirms that the
40-pod HPA maximum provides ~35 % headroom above the 800 RPS nominal peak.

---

## Test Schedule

| Test | Trigger | Environment |
|---|---|---|
| Baseline | Every Monday 03:00 UTC (cron) | Staging |
| Spike | After every new model deploy | Staging |
| Stress | Quarterly | Staging |
| Baseline (canary) | During canary phase of production deploy | Production (10 %) |

---

## Pass / Fail Criteria

### Pass (deployment proceeds)
```
✅ p95 latency ≤ 120 ms         at 800 RPS for 30 minutes
✅ Error rate ≤ 0.1 %
✅ Feature store miss rate ≤ 5 %
✅ cold_start responses return for new user IDs
✅ X-Model-Version header present in every response
```

### Fail (deployment is blocked)
```
❌ p95 latency > 120 ms        → revisit capacity-plan.md replica count
❌ Error rate > 0.1 %          → investigate model or feature store issues
❌ Feature miss rate > 5 %     → check offline data pipeline lag
❌ Model version header absent → serving code defect
```

---

## Running the Tests

```bash
# Baseline against staging
k6 run \
  --env TARGET_URL=https://api.staging.company.com \
  --env API_TOKEN=$(vault kv get -field=token secret/k6-api) \
  --out influxdb=http://influxdb:8086/k6 \
  k6/baseline.js

# Results dashboard: https://grafana.internal/d/k6-results
```
