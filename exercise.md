# 🧪 k6 Hands-On Practice — REST API Load Testing

A complete hands-on scenario that covers all core k6 concepts: project setup, HTTP testing, response handling, checks, lifecycle phases, modular scripts, environment variables, scenarios, custom metrics, thresholds, and output reporting — all in one real-world REST API load testing use case.

---

## 📖 Scenario

You are a backend engineer at **TokoKita** — a contact management platform. Before going live, you need to load test the REST API to ensure it can handle real traffic. You will progressively build the load test — from a single GET request all the way to a multi-scenario test with thresholds, custom metrics, and real-time output.

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

  // Step 1 — Register
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
  const registerResponse = http.post('http://localhost:3000/api/users',
    JSON.stringify({
      username: `user-${uniqueId}`,
      password: 'rahasia',
      name: 'Dzaru Rizky Fathan Fortuna'
    }), {
      headers: { 'Accept': 'application/json', 'Content-Type': 'application/json' }
    }
  );

  const checkRegister = check(registerResponse, {
    'register status is 200':         (r) => r.status === 200,
    'register username not null':     (r) => r.json().data.username !== null,
  });

  if (!checkRegister) {
    fail(`Register failed for user-${uniqueId}`);
  }

  // Login
  const loginResponse = http.post('http://localhost:3000/api/users/login',
    JSON.stringify({ username: `user-${uniqueId}`, password: 'rahasia' }), {
      headers: { 'Accept': 'application/json', 'Content-Type': 'application/json' }
    }
  );

  const checkLogin = check(loginResponse, {
    'login status is 200':    (r) => r.status === 200,
    'login token must exist': (r) => r.json().data.token !== null,
  });

  if (!checkLogin) {
    fail(`Login failed for user-${uniqueId}`);
  }

  const token = loginResponse.json().data.token;

  // Create contact
  const contactResponse = http.post('http://localhost:3000/api/contacts',
    JSON.stringify({
      first_name: 'Dzaru',
      last_name: 'Fortuna',
      email: `dzaru-${uniqueId}@example.com`,
      phone: '081217147620'
    }), {
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
        'Authorization': token
      }
    }
  );

  const checkContact = check(contactResponse, {
    'create contact status is 200':  (r) => r.status === 200,
    'contact id must exist':         (r) => r.json().data.id !== null,
  });

  if (!checkContact) {
    fail(`Create contact failed for user-${uniqueId}`);
  }

  const contactId = contactResponse.json().data.id;

  // Create address
  const addressResponse = http.post(
    `http://localhost:3000/api/contacts/${contactId}/addresses`,
    JSON.stringify({
      street: 'Jl. Merdeka No. 10',
      city: 'Surabaya',
      province: 'Jawa Timur',
      country: 'Indonesia',
      postal_code: '60111'
    }), {
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
        'Authorization': token
      }
    }
  );

  check(addressResponse, {
    'create address status is 200': (r) => r.status === 200,
    'address country is Indonesia': (r) => r.json().data.country === 'Indonesia',
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

When running with multiple VUs, you need per-VU identity to login as a pre-seeded user. Use `k6/execution`.

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

  const loginResponse = http.post('http://localhost:3000/api/users/login',
    JSON.stringify({ username: username, password: 'rahasia' }), {
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

  // Search contacts using the logged-in user's token
  const searchResponse = http.get(
    'http://localhost:3000/api/contacts?page=1&size=10', {
      headers: { 'Accept': 'application/json', 'Authorization': token }
    }
  );

  check(searchResponse, {
    'search contacts status is 200': (r) => r.status === 200,
  });

  console.log(`VU ${exec.vu.idInInstance} logged in as ${username}`);
}
```

> ❓ **What do you observe?** Each VU logs in as a different user (testuser1, testuser2, ... testuser5). This is useful when you've pre-seeded test accounts and want each VU to use its own session independently.

---

## 7. Test Lifecycle — setup() & teardown()

Use `setup()` to prepare test data once before all VUs start, and `teardown()` to log cleanup info after.

Create `src/lifecycle.js`:

```javascript
import http from 'k6/http';
import { check, fail } from 'k6';
import exec from 'k6/execution';

export const options = {
  vus: 5,
  duration: '15s',
};

// Runs once before all VUs — prepares contact seed data
export function setup() {
  const contacts = [];
  for (let i = 0; i < 5; i++) {
    contacts.push({
      first_name: `Contact`,
      last_name: `${i}`,
      email: `contact${i}@example.com`,
      phone: `0812000000${i}`
    });
  }

  console.log(`Setup complete — prepared ${contacts.length} contacts`);
  return contacts;
}

// Runs continuously per VU — uses data from setup()
export default function (contacts) {
  const username = `testuser${exec.vu.idInInstance}`;

  const loginResponse = http.post('http://localhost:3000/api/users/login',
    JSON.stringify({ username: username, password: 'rahasia' }), {
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

  // Each VU creates all contacts from setup data
  for (const contact of contacts) {
    const response = http.post('http://localhost:3000/api/contacts',
      JSON.stringify(contact), {
        headers: {
          'Accept': 'application/json',
          'Content-Type': 'application/json',
          'Authorization': token
        }
      }
    );

    check(response, {
      'create contact status is 200': (r) => r.status === 200,
    });
  }
}

// Runs once after all VUs finish
export function teardown(contacts) {
  console.log(`Teardown — test finished, ${contacts.length} contact types were used`);
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

export function updateUser(token, body) {
  return http.patch(`${BASE_URL}/api/users/current`, JSON.stringify(body), {
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json',
      'Authorization': token
    }
  });
}

export function logoutUser(token) {
  return http.del(`${BASE_URL}/api/users/logout`, null, {
    headers: { 'Accept': 'application/json', 'Authorization': token }
  });
}
```

`src/helper/contact.js`:

```javascript
import http from 'k6/http';

const BASE_URL = 'http://localhost:3000';

export function createContact(token, body) {
  return http.post(`${BASE_URL}/api/contacts`, JSON.stringify(body), {
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json',
      'Authorization': token
    }
  });
}

export function getContact(token, contactId) {
  return http.get(`${BASE_URL}/api/contacts/${contactId}`, {
    headers: { 'Accept': 'application/json', 'Authorization': token }
  });
}

export function searchContacts(token, params = '') {
  return http.get(`${BASE_URL}/api/contacts?${params}`, {
    headers: { 'Accept': 'application/json', 'Authorization': token }
  });
}

export function updateContact(token, contactId, body) {
  return http.put(`${BASE_URL}/api/contacts/${contactId}`, JSON.stringify(body), {
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json',
      'Authorization': token
    }
  });
}

export function deleteContact(token, contactId) {
  return http.del(`${BASE_URL}/api/contacts/${contactId}`, null, {
    headers: { 'Accept': 'application/json', 'Authorization': token }
  });
}
```

`src/helper/address.js`:

```javascript
import http from 'k6/http';

const BASE_URL = 'http://localhost:3000';

export function createAddress(token, contactId, body) {
  return http.post(
    `${BASE_URL}/api/contacts/${contactId}/addresses`,
    JSON.stringify(body), {
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
        'Authorization': token
      }
    }
  );
}

export function getAddress(token, contactId, addressId) {
  return http.get(
    `${BASE_URL}/api/contacts/${contactId}/addresses/${addressId}`, {
      headers: { 'Accept': 'application/json', 'Authorization': token }
    }
  );
}

export function listAddresses(token, contactId) {
  return http.get(
    `${BASE_URL}/api/contacts/${contactId}/addresses`, {
      headers: { 'Accept': 'application/json', 'Authorization': token }
    }
  );
}

export function updateAddress(token, contactId, addressId, body) {
  return http.put(
    `${BASE_URL}/api/contacts/${contactId}/addresses/${addressId}`,
    JSON.stringify(body), {
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
        'Authorization': token
      }
    }
  );
}

export function deleteAddress(token, contactId, addressId) {
  return http.del(
    `${BASE_URL}/api/contacts/${contactId}/addresses/${addressId}`,
    null, {
      headers: { 'Accept': 'application/json', 'Authorization': token }
    }
  );
}
```

### 8b. Main Script Using Modules

`src/main.js`:

```javascript
import { check, fail } from 'k6';
import exec from 'k6/execution';

import { loginUser } from './helper/user.js';
import { createContact, searchContacts } from './helper/contact.js';
import { createAddress, listAddresses } from './helper/address.js';

export const options = {
  vus: 10,
  duration: '20s',
  summaryTrendStats: ['avg', 'min', 'med', 'max', 'p(90)', 'p(95)', 'p(99)']
};

export function setup() {
  const totalContacts = Number(__ENV.TOTAL_CONTACTS) || 5;
  const contacts = [];
  for (let i = 0; i < totalContacts; i++) {
    contacts.push({
      first_name: 'Contact',
      last_name: `${i}`,
      email: `contact${i}@example.com`,
      phone: `0812000000${i}`
    });
  }
  return contacts;
}

function getToken() {
  const username = `testuser${exec.vu.idInInstance}`;
  const loginResponse = loginUser({ username, password: 'rahasia' });

  const checkLogin = check(loginResponse, {
    'login status is 200': (r) => r.status === 200,
    'token must exist':    (r) => r.json().data.token !== null,
  });

  if (!checkLogin) fail(`Login failed for ${username}`);

  return loginResponse.json().data.token;
}

export default function (contacts) {
  const token = getToken();

  for (const contact of contacts) {
    const createResponse = createContact(token, contact);

    const checkCreate = check(createResponse, {
      'create contact status is 200': (r) => r.status === 200,
      'contact id must exist':        (r) => r.json().data.id !== null,
    });

    if (!checkCreate) continue;

    const contactId = createResponse.json().data.id;

    // Create an address for each contact
    const addressResponse = createAddress(token, contactId, {
      street: 'Jl. Merdeka No. 10',
      city: 'Surabaya',
      province: 'Jawa Timur',
      country: 'Indonesia',
      postal_code: '60111'
    });

    check(addressResponse, {
      'create address status is 200': (r) => r.status === 200,
    });
  }

  // Search all contacts with pagination
  const searchResponse = searchContacts(token, 'page=1&size=10');
  check(searchResponse, {
    'search contacts status is 200':  (r) => r.status === 200,
    'paging data must exist':         (r) => r.json().paging !== null,
  });
}

export function teardown(contacts) {
  console.log(`Test complete — used ${contacts.length} contact types`);
}
```

Run it:

```bash
k6 run src/main.js
```

---

## 9. Environment Variables

Avoid hardcoding values like base URL and contact count. Use environment variables instead.

Set variables in your shell:

```bash
export BASE_URL=http://localhost:3000
export TOTAL_CONTACTS=20
```

Update `src/helper/user.js`, `src/helper/contact.js`, and `src/helper/address.js` to use `__ENV`:

```javascript
// in each helper file, replace hardcoded URL with:
const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';
```

Update `setup()` in `src/main.js`:

```javascript
export function setup() {
  const totalContacts = Number(__ENV.TOTAL_CONTACTS) || 5;
  const contacts = [];
  for (let i = 0; i < totalContacts; i++) {
    contacts.push({
      first_name: 'Contact',
      last_name: `${i}`,
      email: `contact${i}@example.com`,
      phone: `0812000000${i}`
    });
  }
  return contacts;
}
```

Run with environment variables:

```bash
TOTAL_CONTACTS=20 BASE_URL=http://localhost:3000 k6 run src/main.js
```

> 💡 This makes your scripts portable across environments — local, staging, and production — without changing any code.

---

## 10. Scenarios & Executors

Instead of one test at a time, run multiple scenarios simultaneously — each simulating a different user behavior.

Update `src/main.js`:

```javascript
import { Counter } from 'k6/metrics';
import { check, fail } from 'k6';
import exec from 'k6/execution';

import { registerUser, loginUser } from './helper/user.js';
import { createContact, searchContacts, deleteContact } from './helper/contact.js';
import { createAddress, listAddresses } from './helper/address.js';

const registerSuccess = new Counter('register_success');
const registerError   = new Counter('register_error');
const contactSuccess  = new Counter('contact_create_success');
const contactError    = new Counter('contact_create_error');

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

    // Scenario 2: Constant 5 VUs searching contacts for 20s
    searchContacts: {
      exec: 'searchContactsScenario',
      executor: 'constant-vus',
      vus: 5,
      duration: '20s',
    },

    // Scenario 3: Ramp up contact creation load
    createContacts: {
      exec: 'createContactsScenario',
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
    password: 'rahasia',
    name: 'Dzaru Rizky Fathan Fortuna'
  });

  response.status === 200 ? registerSuccess.add(1) : registerError.add(1);
}

export function searchContactsScenario() {
  const username = `testuser${exec.vu.idInInstance}`;
  const loginResponse = loginUser({ username, password: 'rahasia' });

  if (loginResponse.status !== 200) return;

  const token = loginResponse.json().data.token;
  const searchResponse = searchContacts(token, 'page=1&size=10');

  check(searchResponse, {
    'search contacts status is 200': (r) => r.status === 200,
    'paging data must exist':        (r) => r.json().paging !== null,
  });
}

export function createContactsScenario() {
  const username = `testuser${exec.vu.idInInstance}`;
  const loginResponse = loginUser({ username, password: 'rahasia' });

  if (loginResponse.status !== 200) return;

  const token = loginResponse.json().data.token;
  const uniqueId = new Date().getTime();

  const response = createContact(token, {
    first_name: 'Load',
    last_name: `Test-${uniqueId}`,
    email: `load-${uniqueId}@example.com`,
    phone: '081200000000'
  });

  response.status === 200 ? contactSuccess.add(1) : contactError.add(1);
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

1. **Add a `DELETE /api/contacts/:id` scenario** using the `per-vu-iterations` executor — each VU creates then immediately deletes 5 contacts. Use a `Counter` to track successful deletes.

2. **Simulate a traffic spike on contact search** using `ramping-arrival-rate` — ramp from 10 iterations/sec to 100 iterations/sec over 30 seconds on `GET /api/contacts`. Observe at what point the `http_req_duration` p(95) starts degrading.

3. **Add a threshold for the contact creation success rate** — at least 95% of create contact requests must succeed. Then intentionally break it by sending an invalid email format and verify k6 exits with a non-zero code.

4. **Use `uuidv4` from the k6 JS library** (`https://jslib.k6.io/k6-utils/1.4.0/index.js`) instead of `new Date().getTime()` for generating unique usernames in the registration scenario.

5. **Export results to both CSV and JSON simultaneously** in a single run, then open the CSV and verify the `http_req_duration` values match the terminal summary.

6. **Move `BASE_URL` and `TOTAL_CONTACTS`** to a config object in a new `src/config.js` module, and import it in all helper files — no more `__ENV` calls scattered across files.

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
| Capturing response ID for chained requests | Step 4 — contactId → address |
| check() — non-blocking validation | Step 5 |
| fail() — stop iteration on error | Step 5 |
| Execution context (exec.vu.idInInstance) | Step 6 |
| setup() — prepare data once | Step 7 |
| teardown() — cleanup after test | Step 7 |
| Modular scripts (helper modules) | Step 8 |
| PUT request | Step 8 — updateContact, updateAddress |
| DELETE request | Step 8 — deleteContact, deleteAddress |
| PATCH request | Step 8 — updateUser |
| Query params in GET | Step 8 — searchContacts |
| Environment variables (__ENV) | Step 9 |
| shared-iterations executor | Step 10 |
| constant-vus executor | Step 10 |
| ramping-vus executor | Step 10 |
| Counter custom metric | Step 10, 11 |
| Thresholds — count, rate, percentile | Step 11 |
| Summary export to JSON | Step 12a |
| Real-time output to CSV & JSON | Step 12b, 12c |
| Web dashboard | Step 12d |
