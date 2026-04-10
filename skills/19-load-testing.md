# Skill 19: Load Testing with k6

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 1-2 hours
Depends On: 03, 05

## Input Contract
- Running API server with endpoints: `/api/v1/health`, `/api/v1/auth/signup`, `/api/v1/auth/login`, `/api/v1/images`, `/api/v1/images/generate`
- PostgreSQL and Redis running (local or Docker)
- k6 installed (`brew install k6` on macOS, or download from https://k6.io)
- Ability to create test users that won't conflict with production data

## Output Contract
- `tests/load/basic.js` — Baseline load test with 5 endpoints and staged load profile
- `tests/load/auth-flow.js` — Realistic user journey load test
- k6 thresholds configured: p95 < 500ms, <1% failure rate
- reproducible load tests runnable via `npm run test:load` and `npm run test:load:auth`
- Results summary printed to console with pass/fail thresholds

## Files to Create
- `tests/load/basic.js` — 5-endpoint baseline load test
- `tests/load/auth-flow.js` — Realistic user journey with signup→login→list→generate flow

## Steps

### Step 1: Install k6

```bash
# macOS
brew install k6

# Linux
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E391D3
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6
```

Verify installation:
```bash
k6 version
```

### Step 2: Create `tests/load/basic.js`

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const healthLatency = new Trend('health_latency');
const signupLatency = new Trend('signup_latency');
const loginLatency = new Trend('login_latency');
const listImagesLatency = new Trend('list_images_latency');
const generateLatency = new Trend('generate_latency');

// Configuration
const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';
const API_PREFIX = '/api/v1';

// Staged load: ramp up → sustain → ramp down
// 20 users → 50 users → sustain → ramp down to 0
export const options = {
  stages: [
    { duration: '30s', target: 20 },   // Ramp up to 20 users
    { duration: '1m', target: 20 },      // Sustain 20 users
    { duration: '30s', target: 50 },    // Ramp up to 50 users
    { duration: '1m', target: 50 },      // Sustain 50 users
    { duration: '30s', target: 20 },     // Scale back to 20 users
    { duration: '30s', target: 0 },      // Ramp down to 0
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],       // 95% of requests < 500ms
    http_req_failed: ['rate<0.01'],         // Less than 1% failure rate
    errors: ['rate<0.01'],                   // Custom error rate < 1%
    health_latency: ['p(95)<200'],          // Health endpoint p95 < 200ms
    signup_latency: ['p(95)<1000'],         // Signup p95 < 1000ms
    login_latency: ['p(95)<500'],           // Login p95 < 500ms
    list_images_latency: ['p(95)<500'],      // List images p95 < 500ms
    generate_latency: ['p(95)<2000'],       // Generate p95 < 2000ms
  },
  noConnectionReuse: false,
};

function generateUniqueEmail() {
  return `loadtest_${__VU}_${__ITER}_${Date.now()}@example.com`;
}

export default function () {
  // --- Test 1: Health endpoint ---
  {
    const res = http.get(`${BASE_URL}${API_PREFIX}/health`);
    const passed = check(res, {
      'health status is 200': (r) => r.status === 200,
      'health has status field': (r) => r.json('status') !== undefined,
    });
    errorRate.add(!passed);
    healthLatency.add(res.timings.duration);

    if (!passed) {
      console.error(`Health check failed: status=${res.status} body=${res.body}`);
    }
  }

  sleep(0.5);

  // --- Test 2: Signup ---
  let authToken = '';
  {
    const email = generateUniqueEmail();
    const password = 'LoadTest123!';
    const payload = JSON.stringify({ email, password });
    const params = {
      headers: { 'Content-Type': 'application/json' },
    };

    const res = http.post(`${BASE_URL}${API_PREFIX}/auth/signup`, payload, params);
    const passed = check(res, {
      'signup status is 201': (r) => r.status === 201,
      'signup returns token': (r) => r.json('token') !== undefined,
    });
    errorRate.add(!passed);
    signupLatency.add(res.timings.duration);

    if (passed) {
      authToken = res.json('token');
    } else {
      console.error(`Signup failed: status=${res.status} body=${res.body}`);
    }
  }

  sleep(0.5);

  // --- Test 3: Login ---
  {
    const email = generateUniqueEmail();

    // First signup to have a user to log in with
    const signupPayload = JSON.stringify({ email, password: 'LoadTest123!' });
    const signupParams = { headers: { 'Content-Type': 'application/json' } };
    http.post(`${BASE_URL}${API_PREFIX}/auth/signup`, signupPayload, signupParams);

    // Now login
    const loginPayload = JSON.stringify({ email, password: 'LoadTest123!' });
    const res = http.post(`${BASE_URL}${API_PREFIX}/auth/login`, loginPayload, signupParams);
    const passed = check(res, {
      'login status is 200': (r) => r.status === 200,
      'login returns token': (r) => r.json('token') !== undefined,
    });
    errorRate.add(!passed);
    loginLatency.add(res.timings.duration);

    if (passed) {
      authToken = res.json('token');
    } else {
      console.error(`Login failed: status=${res.status} body=${res.body}`);
    }
  }

  sleep(0.5);

  // --- Test 4: List images (authenticated) ---
  if (authToken) {
    const params = {
      headers: {
        Authorization: `Bearer ${authToken}`,
      },
    };

    const res = http.get(`${BASE_URL}${API_PREFIX}/images?page=1&limit=10`, params);
    const passed = check(res, {
      'list images status is 200': (r) => r.status === 200,
      'list images returns data array': (r) => {
        const body = r.json();
        return Array.isArray(body.data) || Array.isArray(body);
      },
    });
    errorRate.add(!passed);
    listImagesLatency.add(res.timings.duration);

    if (!passed) {
      console.error(`List images failed: status=${res.status} body=${res.body}`);
    }
  }

  sleep(0.5);

  // --- Test 5: Generate image (authenticated) ---
  if (authToken) {
    const payload = JSON.stringify({
      prompt: `A beautiful landscape number ${__ITER}`,
      width: 256,
      height: 256,
    });
    const params = {
      headers: {
        Authorization: `Bearer ${authToken}`,
        'Content-Type': 'application/json',
      },
    };

    const res = http.post(`${BASE_URL}${API_PREFIX}/images/generate`, payload, params);
    const passed = check(res, {
      'generate status is 202 or 200': (r) => r.status === 202 || r.status === 200,
      'generate returns id': (r) => r.json('id') !== undefined,
    });
    errorRate.add(!passed);
    generateLatency.add(res.timings.duration);

    if (!passed) {
      console.error(`Generate failed: status=${res.status} body=${res.body}`);
    }
  }

  sleep(1);
}
```

### Step 3: Create `tests/load/auth-flow.js`

This test simulates a realistic user journey: signup → login → list → upload → generate → view result.

```javascript
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend } from 'k6/metrics';
import { randomString } from 'https://jslib.k6.io/k6-utils/1.4.0/index.js';

// Custom metrics
const userJourneyFailRate = new Rate('user_journey_failed');
const totalJourneyDuration = new Trend('total_journey_duration', true);

// Configuration
const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';
const API_PREFIX = '/api/v1';

// Softer thresholds for realistic flows (includes sequential calls)
export const options = {
  stages: [
    { duration: '20s', target: 10 },   // Warm up
    { duration: '40s', target: 30 },    // Ramp up
    { duration: '1m', target: 30 },     // Sustain
    { duration: '20s', target: 0 },     // Cool down
  ],
  thresholds: {
    http_req_duration: ['p(95)<1000'],     // 95% < 1000ms (realistic flow)
    http_req_failed: ['rate<0.02'],        // < 2% failure rate
    user_journey_failed: ['rate<0.05'],    // < 5% of complete journeys fail
  },
};

function generateEmail() {
  return `user_${randomString(8)}_${Date.now()}@loadtest.com`;
}

export default function () {
  const journeyStart = Date.now();
  let journeyFailed = false;
  const email = generateEmail();
  const password = 'SecureP@ss123!';

  // --- Step 1: Sign up ---
  let authToken = '';
  group('Signup', () => {
    const payload = JSON.stringify({ email, password });
    const params = { headers: { 'Content-Type': 'application/json' } };

    const res = http.post(`${BASE_URL}${API_PREFIX}/auth/signup`, payload, params);

    if (!check(res, {
      'signup succeeded': (r) => r.status === 201,
    })) {
      console.error(`Signup failed for ${email}: status=${res.status} body=${res.body}`);
      journeyFailed = true;
      return;
    }

    authToken = res.json('token');
  });

  if (journeyFailed) {
    userJourneyFailRate.add(1);
    totalJourneyDuration.add(Date.now() - journeyStart);
    return;
  }

  sleep(Math.random() * 2 + 1); // Random think time 1-3s

  // --- Step 2: List images (empty initially) ---
  group('List Images (initial)', () => {
    const params = {
      headers: { Authorization: `Bearer ${authToken}` },
    };

    const res = http.get(`${BASE_URL}${API_PREFIX}/images`, params);

    check(res, {
      'list images succeeded': (r) => r.status === 200,
    });
  });

  sleep(Math.random() * 1 + 0.5); // Think time 0.5-1.5s

  // --- Step 3: Generate an AI image ---
  let generatedImageId = '';
  group('Generate Image', () => {
    const prompts = [
      'A serene mountain landscape at golden hour',
      'An underwater coral reef with tropical fish',
      'A futuristic city skyline at night with neon lights',
      'A cozy cabin in the woods during autumn',
      'A vast desert with sand dunes under a clear sky',
    ];
    const prompt = prompts[__ITER % prompts.length];

    const payload = JSON.stringify({ prompt, width: 256, height: 256 });
    const params = {
      headers: {
        Authorization: `Bearer ${authToken}`,
        'Content-Type': 'application/json',
      },
    };

    const res = http.post(`${BASE_URL}${API_PREFIX}/images/generate`, payload, params);

    if (check(res, {
      'generate succeeded': (r) => r.status === 202 || r.status === 200,
    })) {
      generatedImageId = res.json('id');
    } else {
      console.error(`Generate failed: status=${res.status}`);
    }
  });

  sleep(Math.random() * 3 + 2); // Wait for processing 2-5s

  // --- Step 4: List images (should contain the generated image) ---
  group('List Images (after generation)', () => {
    const params = {
      headers: { Authorization: `Bearer ${authToken}` },
    };

    const res = http.get(`${BASE_URL}${API_PREFIX}/images?limit=10`, params);

    check(res, {
      'list images succeeded': (r) => r.status === 200,
    });
  });

  sleep(Math.random() * 1 + 0.5);

  // --- Step 5: Get specific image (if ID was returned) ---
  if (generatedImageId) {
    group('Get Image Detail', () => {
      const params = {
        headers: { Authorization: `Bearer ${authToken}` },
      };

      const res = http.get(`${BASE_URL}${API_PREFIX}/images/${generatedImageId}`, params);

      check(res, {
        'get image succeeded': (r) => r.status === 200,
        'image has status': (r) => r.json('status') !== undefined,
      });
    });
  }

  sleep(Math.random() * 1 + 0.5);

  // --- Step 6: Login again (tests token refresh / re-auth) ---
  group('Login (re-auth)', () => {
    const payload = JSON.stringify({ email, password });
    const params = { headers: { 'Content-Type': 'application/json' } };

    const res = http.post(`${BASE_URL}${API_PREFIX}/auth/login`, payload, params);

    check(res, {
      're-login succeeded': (r) => r.status === 200,
      'got new token': (r) => r.json('token') !== undefined,
    });
  });

  // Record journey metrics
  userJourneyFailRate.add(journeyFailed ? 1 : 0);
  totalJourneyDuration.add(Date.now() - journeyStart);
}
```

### Step 4: Add npm Scripts

Add to `package.json`:

```json
{
  "scripts": {
    "test:load": "k6 run tests/load/basic.js",
    "test:load:auth": "k6 run tests/load/auth-flow.js",
    "test:load:ci": "k6 run --duration=30s --vus=10 tests/load/basic.js",
    "test:load:smoke": "k6 run --duration=10s --vus=5 tests/load/basic.js"
  }
}
```

### Step 5: Create `.k6ignore` or Add to `.gitignore`

Load tests create test data (users, images). Add cleanup guidance:

```bash
# Add to .gitignore if not present
tests/load/results/
```

### Step 6: Create a Smoke Test Helper

Create `tests/load/README.md` with usage instructions:

```
# Load Tests

## Prerequisites
- k6 installed (https://k6.io/docs/get-started/installation/)
- API server running locally or set BASE_URL

## Quick Tests
- Smoke test (5 users, 10s):   npm run test:load:smoke
- CI test (10 users, 30s):     npm run test:load:ci
- Full basic load test:         npm run test:load
- Auth flow load test:          npm run test:load:auth

## Custom Base URL
BASE_URL=https://api.example.com k6 run tests/load/basic.js

## Reading Results
- p95 latency should be < 500ms for basic endpoints
- Error rate should be < 1%
- Look at the `total_journey_duration` trend for auth-flow realism

## Cleanup
Load tests create users with @loadtest.com/@example.com emails.
Clean test data from the database after runs.
```

## Verification

1. **Run the smoke test (fast, minimal load):**
   ```bash
   npm run test:load:smoke
   ```
   Expected: All checks pass, thresholds met, execution completes in ~10 seconds.

2. **Run the full basic load test:**
   ```bash
   npm run test:load
   ```
   Expected: Staged load (20→50→20→0 users), all thresholds pass:
   - `http_req_duration p(95) < 500ms`
   - `http_req_failed rate < 1%`
   - `errors rate < 1%`
   - Individual endpoint thresholds met

3. **Run the auth-flow test:**
   ```bash
   npm run test:load:auth
   ```
   Expected: Realistic user journeys complete, `user_journey_failed` rate < 5%.

4. **Check custom metrics in output:**
   k6 output should include custom trends:
   - `health_latency`
   - `signup_latency`
   - `login_latency`
   - `list_images_latency`
   - `generate_latency`
   - `total_journey_duration`

5. **Run against a different base URL:**
   ```bash
   BASE_URL=http://staging.example.com npm run test:load:smoke
   ```

6. **Verify threshold failure detection:**
   Temporarily lower a threshold to an impossible value (e.g., `p(95)<1`), run the test, and verify k6 exits with a non-zero exit code.

## Rollback

1. Delete load test files:
   ```bash
   rm -rf tests/load/
   ```

2. Remove npm scripts from `package.json`:
   ```json
   Remove: "test:load", "test:load:auth", "test:load:ci", "test:load:smoke"
   ```

3. Remove k6 (optional):
   ```bash
   # macOS
   brew uninstall k6
   ```

4. Clean up test data created by load tests:
   ```sql
   DELETE FROM users WHERE email LIKE '%@loadtest.com' OR email LIKE '%@example.com';
   DELETE FROM images WHERE user_id NOT IN (SELECT id FROM users);
   ```

## ADR-019: k6 for Load Testing with Staged Profiles

**Decision:** Use k6 with staged load profiles and custom thresholds for load testing, separating baseline endpoint tests (`basic.js`) from realistic user journeys (`auth-flow.js`).

**Reason:** k6 provides a JavaScript-based scripting interface familiar to web developers, supports custom metrics and thresholds for pass/fail criteria, and provides detailed output including percentiles. Staged load profiles (20→50→0 VUs) simulate realistic traffic patterns with warm-up, peak, and cool-down phases rather than sudden spikes. Separating baseline and journey tests allows running lightweight checks independently.

**Consequences:**
- Two test files to maintain — endpoint metrics in `basic.js`, journey metrics in `auth-flow.js`
- Load tests create real data (users, images) — may need database cleanup
- k6 must be installed on developer machines and CI runners
- Thresholds are enforced — CI will fail if performance degrades below thresholds
- Custom metrics (`health_latency`, `signup_latency`, etc.) require explicit trending in k6

**Alternatives Considered:**
- **Artillery:** YAML-based config, simpler to start but less flexible for custom metrics and realistic flows
- **JMeter:** Java-based, powerful but heavy, poor developer experience for JavaScript teams
- **Locust:** Python-based, requires context-switching from TypeScript project
- **Autocannon:** Node.js-based, fast but lacks staged profiles and custom threshold assertions