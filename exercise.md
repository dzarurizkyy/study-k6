# 🧪 k6 Hands-On Practice — REST API Load Testing

A complete hands-on scenario that covers all core k6 concepts: project setup, HTTP testing, response handling, checks, lifecycle phases, modular scripts, environment variables, scenarios, custom metrics, thresholds, and output reporting — all in one real-world REST API load testing use case.

---

## 📖 Scenario

You are a backend engineer at **TokoKita** — an e-commerce platform. Before going live, you need to load test the REST API to ensure it can handle real traffic. The API has these endpoints:

- `POST /api/users` — Register a new user
- `POST /api/users/login` — Login and get a token
- `GET /api/users/current` — Get current user profile (requires auth)
- `POST /api/products` — Create a product (requires auth)
- `GET /api/products` — Get all products (requires auth)

You will progressively build the load test — from a single GET request all the way to a multi-scenario test with thresholds, custom metrics, and real-time output.

---

## 🗂️ Table of Contents

1. [Setup & Verify](#1-setup--verify)
2. [Your First Script](#2-your-first-script)
3. [HTTP Testing — POST with JSON Body](#3-http-testing--post-with-json-body)
4. [Working with Responses](#4-working-with-responses)
5. [Test Validation — check() & fail()](#5-test-validation--check--fail)
6. [Execution Context](#6-execution-context)
7. [Test Lifecycle — setup() & teardown()](#7-test-lifecycle--setup--teardown)
8. [Modular Scripts](#8-modular-scripts)
9. [Environment Variables](#9-environment-variables)
10. [Scenarios & Executors](#10-scenarios--executors)
11. [Custom Metrics & Thresholds](#11-custom-metrics--thresholds)
12. [Output & Reporting](#12-output--reporting)
13. [Challenge Tasks](#-challenge-tasks)

---

## 1. Setup & Verify

### 1a. Install k6

```bash
brew install k6
```

Verify:

```bash
k6 --version
```

Expected output:

```
k6 v0.xx.x
```

### 1b. Create Project

```bash
mkdir tokokita-k6
cd tokokita-k6
npm init -y
npm install k6
npm install --save-dev @types/k6
```

Update `package.json`:

```json
{
  "name": "tokokita-k6",
  "version": "1.0.0",
  "type": "module"
}
```

### 1c. Create Project Structure

```
tokokita-k6/
├── src/
│   ├── main.js
│   └── helper/
│       ├── user.js
│       └── product.js
├── package.json
└── summary.json        ← generated after running tests
```

---

## 2. Your First Script

Start simple — just hit the health check endpoint and verify k6 is working.

Create `src/ping.js`:

```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  vus: 5,
  duration: '10s',
  summaryTrendStats: ['avg', 'min', 'med', 'max', 'p(90)', 'p(95)', 'p(99)']
};

export default function () {
  http.get('http://localhost:3000/ping');
  sleep(1);
}
```

Run it:

```bash
k6 run src/ping.js
```

Expected output (summary):

```
http_req_duration............: avg=12ms   min=5ms   med=10ms  max=45ms   p(90)=20ms  p(95)=30ms  p(99)=44ms
http_reqs....................: 50     4.8/s
vus..........................: 5      min=5 max=5
```

> ❓ **Notice** — 5 virtual users running for 10 seconds with `sleep(1)` means roughly 5 requests/second. Try removing `sleep(1)` and observe the difference.

### 2a. Try Stages (Ramp Up & Down)

Update `options` in `src/ping.js`:

```javascript
export const options = {
  stages: [
    { duration: '5s',  target: 10 },
    { duration: '10s', target: 10 },
    { duration: '5s',  target: 0  },
  ],
  summaryTrendStats: ['avg', 'min', 'med', 'max', 'p(90)', 'p(95)', 'p(99)']
};
```

Run again and watch the `vus` metric ramp up and back down in real time.

---

## 3. HTTP Testing — POST with JSON Body

Now test the register endpoint. Create `src/register.js`:

```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  vus: 10,
  duration: '15s',
};

export default function () {
  const uniqueId = new Date().getTime();

  const body = {
    username: `user-${uniqueId}`,
    password: 'secret',
    name: 'Dzaru Rizky Fathan Fortuna'
  };

  const response = http.post('http://localhost:3000/api/users', JSON.stringify(body), {
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json'
    }
  });

  console.log(`Register status: ${response.status}`);
  sleep(1);
}
```

Run it:

```bash
k6 run src/register.js
```

> ❓ **What do you observe?** With 10 VUs running for 15 seconds each with `sleep(1)`, roughly 150 unique users get registered. Every iteration uses a different `uniqueId` from `new Date().getTime()`.

---

## 4. Working with Responses

A common real-world flow: register → login → use token for authenticated requests.

Create `src/auth-flow.js`:

```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  vus: 5,
  duration: '15s',
};

export default function () {
  const uniqueId = new Date().getTime();

  // Step 1 — Register/products/
  const registerBody = {
    username: `user-${uniqueId}`,
    password: 'secret',
    name: 'Dzaru Rizky Fathan Fortuna'
  };

  http.post('http://localhost:3000/api/users', JSON.stringify(registerBody), {
    headers: { 'Accept': 'application/json', 'Content-Type': 'application/json' }
  });

  // Step 2 — Login and capture token
  const loginBody = {
    username: `user-${uniqueId}`,
    password: 'secret'
  };

  const loginResponse = http.post('http://localhost:3000/api/users/login', JSON.stringify(loginBody), {
    headers: { 'Accept': 'application/json', 'Content-Type': 'application/json' }
  });

  const token = loginResponse.json().data.token;

  // Step 3 — Access protected endpoint using token
  http.get('http://localhost:3000/api/users/current', {
    headers: {
      'Accept': 'application/json',
      'Authorization': token
    }
  });

  sleep(1);
}
```

Run it:

```bash
k6 run src/auth-flow.js
```

> ❓ **Notice** — `loginResponse.json()` parses the response body as JSON so you can chain into `.data.token`. This is how you pass auth context between requests in the same iteration.

---

## 5. Test Validation — check() & fail()

Without validation, k6 doesn't care if your API returns 500. Let's add proper validation.

Create `src/validated-flow.js`:

```javascript
import http from 'k6/http';
import { check, fail } from 'k6';
import { sleep } from 'k6';

export const options = {
  vus: 10,
  duration: '15s',
};

export default function () {
  const uniqueId = new Date().getTime();

  // Register
  const registerBody = {
    username: `user-${uniqueId}`,
    password: 'secret',
    name: 'Dzaru Rizky Fathan Fortuna'
  };

  const registerResponse = http.post('http://localhost:3000/api/users', JSON.stringify(registerBody), {
    headers: { 'Accept': 'application/json', 'Content-Type': 'application/json' }
  });

  const checkRegister = check(registerResponse, {
    'register status is 200':          (r) => r.status === 200,
    'register response data not null':  (r) => r.json().data !== null,
  });

  // If register fails, stop this iteration — no point continuing
  if (!checkRegister) {
    fail(`Register failed for user-${uniqueId}`);
  }

  // Login
  const loginBody = {
    username: `user-${uniqueId}`,
    password: 'secret'
  };

  const loginResponse = http.post('http://localhost:3000/api/users/login', JSON.stringify(loginBody), {
    headers: { 'Accept': 'application/json', 'Content-Type': 'application/json' }
  });

  const checkLogin = check(loginResponse, {
    'login status is 200':        (r) => r.status === 200,
    'login token must exist':     (r) => r.json().data.token !== null,
  });

  if (!checkLogin) {
    fail(`Login failed for user-${uniqueId}`);
  }

  const token = loginResponse.json().data.token;

  // Get current user
  const profileResponse = http.get('http://localhost:3000/api/users/current', {
    headers: { 'Accept': 'application/json', 'Authorization': token }
  });

  check(profileResponse, {
    'get profile status is 200': (r) => r.status === 200,
  });

  sleep(1);
}
```

Run it:

```bash
k6 run src/validated-flow.js
```

After the run, look for the `checks` section in the summary:

```
checks.........................: 97.50% ✓ 975  ✗ 25
```

> 💡 **check() vs fail()** — `check()` records pass/fail but never stops execution. `fail()` stops the current iteration immediately. Use them together: `check()` to validate, then `fail()` if the check fails and continuing makes no sense.

---

## 6. Execution Context

When running with multiple VUs, you need per-VU identity (e.g., to log in as a pre-seeded user). Use `k6/execution`.

Create `src/exec-context.js`:

```javascript
import http from 'k6/http';
import { check, fail } from 'k6';
import exec from 'k6/execution';

export const options = {
  vus: 5,
  duration: '10s',
};

export default function () {
  // Each VU gets a unique ID: 1, 2, 3, 4, 5
  const username = `testuser${exec.vu.idInInstance}`;

  const loginBody = { username: username, password: 'secret' };

  const loginResponse = http.post('http://localhost:3000/api/users/login', JSON.stringify(loginBody), {
    headers: { 'Accept': 'application/json', 'Content-Type': 'application/json' }
  });

  const checkLogin = check(loginResponse, {
    'login status is 200':    (r) => r.status === 200,
    'token must exist':       (r) => r.json().data.token !== null,
  });

  if (!checkLogin) {
    fail(`Login failed for ${username}`);
  }

  console.log(`VU ${exec.vu.idInInstance} logged in as ${username}`);
}
```

> ❓ **What do you observe?** Each VU logs in as a different user (testuser1, testuser2, ... testuser5). This is useful when you've pre-seeded test accounts and want each VU to use its own session.

---

## 7. Test Lifecycle — setup() & teardown()

Use `setup()` to prepare test data once before all VUs start, and `teardown()` to clean up after the test.

Create `src/lifecycle.js`:

```javascript
import http from 'k6/http';
import { check, fail } from 'k6';
import exec from 'k6/execution';

export const options = {
  vus: 5,
  duration: '15s',
};

// Runs once before all VUs — seeds product data
export function setup() {
  const products = [];
  for (let i = 0; i < 5; i++) {
    products.push({
      name: `Product ${i}`,
      price: (i + 1) * 10000,
      stock: 100
    });
  }

  console.log(`Setup complete — prepared ${products.length} products`);
  return products; // passed to default function as parameter
}

// Runs continuously per VU — uses data from setup()
export default function (products) {
  const username = `testuser${exec.vu.idInInstance}`;

  const loginResponse = http.post('http://localhost:3000/api/users/login',
    JSON.stringify({ username: username, password: 'secret' }), {
      headers: { 'Accept': 'application/json', 'Content-Type': 'application/json' }
    }
  );

  const checkLogin = check(loginResponse, {
    'login status is 200': (r) => r.status === 200,
    'token must exist':    (r) => r.json().data.token !== null,
  });

  if (!checkLogin) {
    fail(`Login failed for ${username}`);
  }

  const token = loginResponse.json().data.token;

  // Each VU creates all products using the token
  for (const product of products) {
    const response = http.post('http://localhost:3000/api/products', JSON.stringify(product), {
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
        'Authorization': token
      }
    });

    check(response, {
      'create product status is 200': (r) => r.status === 200,
    });
  }
}

// Runs once after all VUs finish
export function teardown(products) {
  console.log(`Teardown — test finished, ${products.length} product types were used`);
}
```

Run it:

```bash
k6 run src/lifecycle.js
```

> 💡 **Key insight** — `setup()` runs **once**, returns data, and that data is passed as a parameter to every VU's `default function`. This is the correct way to share read-only test data across VUs without re-computing it each iteration.

---

## 8. Modular Scripts

As scripts grow, split them into reusable helper modules.

### 8a. Create Helper Modules

`src/helper/user.js`:

```javascript
import http from 'k6/http';

const BASE_URL = 'http://localhost:3000';

export function registerUser(body) {
  return http.post(`${BASE_URL}/api/users`, JSON.stringify(body), {
    headers: { 'Accept': 'application/json', 'Content-Type': 'application/json' }
  });
}

export function loginUser(body) {
  return http.post(`${BASE_URL}/api/users/login`, JSON.stringify(body), {
    headers: { 'Accept': 'application/json', 'Content-Type': 'application/json' }
  });
}

export function getCurrentUser(token) {
  return http.get(`${BASE_URL}/api/users/current`, {
    headers: { 'Accept': 'application/json', 'Authorization': token }
  });
}
```

`src/helper/product.js`:

```javascript
import http from 'k6/http';

const BASE_URL = 'http://localhost:3000';

export function createProduct(token, product) {
  return http.post(`${BASE_URL}/api/products`, JSON.stringify(product), {
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json',
      'Authorization': token
    }
  });
}

export function getProducts(token) {
  return http.get(`${BASE_URL}/api/products`, {
    headers: { 'Accept': 'application/json', 'Authorization': token }
  });
}
```

### 8b. Main Script Using Modules

`src/main.js`:

```javascript
import { check, fail } from 'k6';
import exec from 'k6/execution';

import { loginUser } from './helper/user.js';
import { createProduct, getProducts } from './helper/product.js';

export const options = {
  vus: 10,
  duration: '20s',
  summaryTrendStats: ['avg', 'min', 'med', 'max', 'p(90)', 'p(95)', 'p(99)']
};

export function setup() {
  const totalProducts = Number(__ENV.TOTAL_PRODUCTS) || 5;
  const products = [];
  for (let i = 0; i < totalProducts; i++) {
    products.push({ name: `Product ${i}`, price: (i + 1) * 10000, stock: 100 });
  }
  return products;
}

function getToken() {
  const username = `testuser${exec.vu.idInInstance}`;
  const loginResponse = loginUser({ username, password: 'secret' });

  const checkLogin = check(loginResponse, {
    'login status is 200':  (r) => r.status === 200,
    'token must exist':     (r) => r.json().data.token !== null,
  });

  if (!checkLogin) fail(`Login failed for ${username}`);

  return loginResponse.json().data.token;
}

export default function (products) {
  const token = getToken();

  for (const product of products) {
    const createResponse = createProduct(token, product);
    check(createResponse, {
      'create product status is 200': (r) => r.status === 200,
    });
  }

  const listResponse = getProducts(token);
  check(listResponse, {
    'get products status is 200':   (r) => r.status === 200,
    'products list is not empty':   (r) => r.json().data.length > 0,
  });
}

export function teardown(products) {
  console.log(`Test complete — used ${products.length} product types`);
}
```

Run it:

```bash
k6 run src/main.js
```

---

## 9. Environment Variables

Avoid hardcoding values like base URL and product count. Use environment variables instead.

Set variables in your shell:

```bash
export BASE_URL=http://localhost:3000
export TOTAL_PRODUCTS=20
```

Update `src/helper/user.js` and `src/helper/product.js` to use `__ENV`:

```javascript
// src/helper/user.js
const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';
```

```javascript
// src/helper/product.js
const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';
```

Update `setup()` in `src/main.js`:

```javascript
export function setup() {
  const totalProducts = Number(__ENV.TOTAL_PRODUCTS) || 5;
  const products = [];
  for (let i = 0; i < totalProducts; i++) {
    products.push({ name: `Product ${i}`, price: (i + 1) * 10000, stock: 100 });
  }
  return products;
}
```

Run with environment variables:

```bash
TOTAL_PRODUCTS=20 BASE_URL=http://localhost:3000 k6 run src/main.js
```

> 💡 This makes your scripts portable across environments — local, staging, and production — without changing any code.

---

## 10. Scenarios & Executors

Instead of one test at a time, run multiple scenarios simultaneously — each simulating a different user behavior.

Update `src/main.js` options:

```javascript
import { Counter } from 'k6/metrics';
import { check, fail } from 'k6';
import exec from 'k6/execution';

import { registerUser, loginUser } from './helper/user.js';
import { createProduct, getProducts } from './helper/product.js';

const registerSuccess = new Counter('register_success');
const registerError   = new Counter('register_error');
const productSuccess  = new Counter('product_create_success');
const productError    = new Counter('product_create_error');

export const options = {
  scenarios: {

    // Scenario 1: 200 registrations shared across 10 VUs
    userRegistration: {
      exec: 'userRegistration',
      executor: 'shared-iterations',
      vus: 10,
      iterations: 200,
      maxDuration: '30s',
    },

    // Scenario 2: Constant 5 VUs browsing products for 20s
    browseProducts: {
      exec: 'browseProducts',
      executor: 'constant-vus',
      vus: 5,
      duration: '20s',
    },

    // Scenario 3: Ramp up product creation load
    createProducts: {
      exec: 'createProducts',
      executor: 'ramping-vus',
      stages: [
        { duration: '5s',  target: 5  },
        { duration: '10s', target: 10 },
        { duration: '5s',  target: 0  },
      ],
    },
  },
};

export function userRegistration() {
  const uniqueId = new Date().getTime();
  const response = registerUser({
    username: `user-${uniqueId}`,
    password: 'secret',
    name: 'Dzaru Rizky Fathan Fortuna'
  });

  response.status === 200 ? registerSuccess.add(1) : registerError.add(1);
}

export function browseProducts() {
  const username = `testuser${exec.vu.idInInstance}`;
  const loginResponse = loginUser({ username, password: 'secret' });

  if (loginResponse.status !== 200) return;

  const token = loginResponse.json().data.token;
  const listResponse = getProducts(token);

  check(listResponse, {
    'get products status is 200': (r) => r.status === 200,
  });
}

export function createProducts() {
  const username = `testuser${exec.vu.idInInstance}`;
  const loginResponse = loginUser({ username, password: 'secret' });

  if (loginResponse.status !== 200) return;

  const token = loginResponse.json().data.token;
  const response = createProduct(token, {
    name: `Product-${new Date().getTime()}`,
    price: 50000,
    stock: 10
  });

  response.status === 200 ? productSuccess.add(1) : productError.add(1);
}
```

Run it:

```bash
k6 run src/main.js
```

> ❓ **What do you observe?** All 3 scenarios run concurrently. The summary will show metrics broken down per scenario, giving you a realistic picture of mixed traffic load.

---

## 11. Custom Metrics & Thresholds

Add thresholds to define pass/fail criteria for your test.

Add thresholds to the `options` in `src/main.js`:

```javascript
export const options = {
  thresholds: {
    // At least 190 out of 200 registrations must succeed
    register_success: ['count>190'],
    // Fewer than 10 registration errors allowed
    register_error:   ['count<10'],
    // 95th percentile of all HTTP requests must be under 500ms
    http_req_duration: ['p(95)<500'],
    // Less than 1% of requests can fail
    http_req_failed:   ['rate<0.01'],
  },
  scenarios: {
    // ... same as above
  }
};
```

Run it and observe the result:

```
✓ register_success.........: count=198
✓ register_error...........: count=2
✓ http_req_duration........: p(95)=120ms
✓ http_req_failed..........: rate=0.001
```

If a threshold is violated, k6 exits with a non-zero status code — making it easy to fail CI/CD pipelines automatically.

> ❓ **Try this** — lower the threshold to `count>199` and run again. The test should now fail with `✗ register_success`.

---

## 12. Output & Reporting

### 12a. Export Summary to JSON

```bash
k6 run src/main.js --summary-export=summary.json
```

### 12b. Real-time Output to CSV

```bash
k6 run --out csv=test_results.csv src/main.js
```

### 12c. Real-time Output to JSON

```bash
k6 run --out json=test_results.json src/main.js
```

### 12d. Web Dashboard

Watch your test live in a browser:

```bash
export K6_WEB_DASHBOARD=true
k6 run src/main.js
```

Open your browser at `http://localhost:5665` while the test is running.

> 💡 The web dashboard shows VUs, request rate, response times, and check results in real time — much easier to monitor than reading the terminal during a long test.

---

## 🏆 Challenge Tasks

Once you've completed the practice above, try these on your own:

1. **Add a `DELETE /api/products/:id` scenario** using the `per-vu-iterations` executor — each VU deletes 5 products. Use a `Counter` to track successful deletes.

2. **Simulate a traffic spike** using `ramping-arrival-rate` — ramp from 10 iterations/sec to 100 iterations/sec over 30 seconds on the `GET /api/products` endpoint. Observe at what point the `http_req_duration` p(95) starts degrading.

3. **Add a threshold for the product creation success rate** — at least 95% of create product requests must succeed. Then intentionally break it by sending invalid data and verify k6 exits with a non-zero code.

4. **Use `uuidv4` from the k6 JS library** (`https://jslib.k6.io/k6-utils/1.4.0/index.js`) instead of `new Date().getTime()` for generating unique usernames in the registration scenario.

5. **Export results to both CSV and JSON simultaneously** in a single run, then open the CSV and verify the `http_req_duration` values match the terminal summary.

6. **Move `BASE_URL` and `TOTAL_PRODUCTS`** to a config object in a new `src/config.js` module, and import it in both helper files — no more `__ENV` calls scattered across files.

---

## ✅ Concepts Covered

| Concept | Where Practiced |
|---|---|
| Project setup & installation | Step 1 |
| options — vus, duration, stages | Step 2, 2a |
| summaryTrendStats | Step 2 |
| GET request | Step 2 |
| POST request with JSON body | Step 3 |
| Working with response & chaining requests | Step 4 |
| check() — non-blocking validation | Step 5 |
| fail() — stop iteration on error | Step 5 |
| Execution context (exec.vu.idInInstance) | Step 6 |
| setup() — prepare data once | Step 7 |
| teardown() — cleanup after test | Step 7 |
| Modular scripts (helper modules) | Step 8 |
| Environment variables (__ENV) | Step 9 |
| shared-iterations executor | Step 10 |
| constant-vus executor | Step 10 |
| ramping-vus executor | Step 10 |
| Counter custom metric | Step 10, 11 |
| Thresholds — count, rate, percentile | Step 11 |
| Summary export to JSON | Step 12a |
| Real-time output to CSV & JSON | Step 12b, 12c |
| Web dashboard | Step 12d |
