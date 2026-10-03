# ⚡ k6 Load Testing

A practical reference guide for performance testing with k6 — covering installation, writing and running scripts, options, HTTP requests, validation with checks, the test lifecycle, modular scripts, environment variables, scenarios and executors, metrics, thresholds, and output & reporting, worked through hands-on against a REST API.

---

## 📋 Table of Contents

- [Introduction](#-introduction)
  - [What Is k6?](#what-is-k6)
  - [What k6 Does Not Do](#what-k6-does-not-do)
  - [JavaScript Limitations](#javascript-limitations)
- [Installation & Setup](#-installation--setup)
  - [System Requirements](#system-requirements)
  - [Installing k6](#installing-k6)
  - [Project Setup](#project-setup)
- [Writing Scripts](#-writing-scripts)
  - [Generating a Script](#generating-a-script)
  - [Script Structure](#script-structure)
- [Running Tests](#-running-tests)
- [Options](#-options)
  - [Basic Options](#basic-options)
  - [Stages](#stages)
- [HTTP Requests](#-http-requests)
  - [GET Request](#get-request)
  - [POST Request with JSON Body](#post-request-with-json-body)
- [Working with Responses](#-working-with-responses)
- [Test Validation](#-test-validation)
  - [fail()](#fail)
  - [check()](#check)
- [Execution Context](#-execution-context)
- [Test Lifecycle](#-test-lifecycle)
  - [Lifecycle Stages](#lifecycle-stages)
  - [Complete Lifecycle Example](#complete-lifecycle-example)
- [Modular Scripts](#-modular-scripts)
  - [Helper Modules](#helper-modules)
  - [Main Script](#main-script)
- [Environment Variables](#-environment-variables)
- [Scenarios](#-scenarios)
  - [Executors](#executors)
  - [Shared Iterations Example](#shared-iterations-example)
- [Metrics](#-metrics)
  - [Built-in Metrics](#built-in-metrics)
  - [Custom Metrics](#custom-metrics)
- [Thresholds](#-thresholds)
- [Output & Reporting](#-output--reporting)
  - [Summary Output](#summary-output)
  - [Summary Statistics](#summary-statistics)
  - [Real-time Output](#real-time-output)
  - [Third-party Services](#third-party-services)
  - [Web Dashboard](#web-dashboard)
- [JavaScript Libraries](#-javascript-libraries)
- [Test Execution Flow](#-test-execution-flow)
- [Quick Reference](#-quick-reference)
- [Best Practices](#-best-practices)

---

## 🎯 Introduction

### What Is k6?

- **k6** is a free, open-source **load testing tool** built for developers
- It helps you test application performance, find issues early, and make sure an application holds up under load
- **Terminal-based** — runs on any system, including servers without a GUI
- **Scripted in JavaScript** — test scenarios are plain JavaScript files
- **Built-in libraries** — ships with modules (`k6/http`, `k6/metrics`, `k6/execution`, …) that simplify writing scenarios
- **Easily extensible** — can be rebuilt with extensions for extra outputs and protocols

> Reference: [grafana.com/docs/k6](https://grafana.com/docs/k6/latest/)

### What k6 Does Not Do

| k6 is **not** | Why |
| --- | --- |
| A browser | It is terminal-based and does not render web pages |
| A Node.js program | Scripts are executed by k6 itself (written in Go), not by Node.js |
| Compatible with npm modules | Node packages cannot be imported directly |

### JavaScript Limitations

- k6 is built with **Golang**
- It uses the **Goja** library to execute JavaScript inside Go
- Scripts are JavaScript, but **only the features Goja supports** are available

> **Key Insight:** a k6 script *looks* like Node.js code but runs in a completely different runtime. Anything that depends on Node APIs (`fs`, `require`, npm packages) will not work — use k6's own modules instead.

---

## 📦 Installation & Setup

### System Requirements

| Requirement | Detail |
| --- | --- |
| Operating system | macOS, Linux, Windows |
| Memory | 2 GB RAM recommended |
| CPU | Modern CPU — multi-core recommended for high-load tests |

### Installing k6

```bash
# Install on macOS
brew install k6

# Verify the installation
k6 --version
```

Expected output:

```text
k6 v0.xx.x
```

### Project Setup

```bash
# Create the project directory
mkdir my-k6-project
cd my-k6-project

# Initialize a Node.js project
npm init

# Install the k6 package and its type definitions
npm install k6
npm install --save-dev @types/k6
```

Set the module type to ES modules in `package.json`:

```json
{
  "name": "my-k6-project",
  "version": "1.0.0",
  "type": "module"
}
```

> **Note:** the npm packages only contain TypeScript definitions and metadata for editor autocompletion — not the actual k6 runtime, which is the Go binary installed above.

---

## 📝 Writing Scripts

### Generating a Script

```bash
# Create a script file from k6's template
k6 new script.js
```

This creates a file at the given location containing a simple performance testing script. You can also write one by hand:

```javascript
// test.js
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  vus: 10,
  duration: '30s',
};

export default function () {
  http.get('http://localhost:3000/ping');
  sleep(1);
}
```

### Script Structure

| Part | Description |
| --- | --- |
| **`options`** | Configuration — number of virtual users (VUs), test duration, stages, thresholds, scenarios |
| **`default` function** | The code each virtual user executes, according to `options` |

---

## 🏃 Running Tests

```bash
k6 run script.js
```

When running a script, k6:

- Reads the configuration from `options`
- Starts the configured number of virtual users in parallel
- Has each virtual user call the `default` function repeatedly
- Keeps iterating until the specified duration is reached
- Pauses between iterations wherever `sleep()` is called

---

## 🔧 Options

### Basic Options

```javascript
export const options = {
  vus: 10,          // Number of virtual users
  duration: '30s',  // Test duration
};
```

| Option | Description |
| --- | --- |
| `vus` | Number of virtual users running concurrently |
| `duration` | How long the test runs |
| `stages` | Ramp the number of VUs up and down over time |
| `summaryTrendStats` | Statistics shown in the end-of-test summary — see [Summary Statistics](#summary-statistics) |
| `thresholds` | Pass/fail criteria — see [Thresholds](#-thresholds) |
| `scenarios` | Multiple workloads in one script — see [Scenarios](#-scenarios) |

> Reference: [k6 options reference](https://grafana.com/docs/k6/latest/using-k6/k6-options/reference/)

### Stages

`stages` increases the number of users over one duration and decreases it over another:

```javascript
export const options = {
  stages: [
    { duration: '10s', target: 20 },  // Ramp up to 20 users
    { duration: '10s', target: 10 },  // Ramp down to 10 users
    { duration: '10s', target: 0 },   // Ramp down to 0 users
  ],
};
```

> Reference: [k6-options/reference#stages](https://grafana.com/docs/k6/latest/using-k6/k6-options/reference/#stages)

---

## 🌐 HTTP Requests

k6 includes the `k6/http` module for HTTP testing. Nearly every HTTP method is supported. It is **not** a browser testing library — it sends raw HTTP requests.

> Reference: [javascript-api/k6-http](https://grafana.com/docs/k6/latest/javascript-api/k6-http/)

### GET Request

```javascript
import http from 'k6/http';

export default function () {
  http.get('http://localhost:3000/api/users');
}
```

### POST Request with JSON Body

```javascript
import http from 'k6/http';

export default function () {
  const uniqueId = new Date().getTime();
  const body = {
    username: `user-${uniqueId}`,
    password: 'secret',
    name: 'Dzaru Rizky Fathan Fortuna'
  };

  http.post('http://localhost:3000/api/users', JSON.stringify(body), {
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json'
    }
  });
}
```

> **Note:** the body must be serialised with `JSON.stringify()` and sent with a `Content-Type: application/json` header — k6 does not encode objects as JSON for you.

---

## 📡 Working with Responses

Every function in `k6/http` returns an HTTP **response**. Capture it to reuse data — such as an auth token — in the next request:

```javascript
import http from 'k6/http';

export default function () {
  const uniqueId = new Date().getTime();
  const body = {
    username: `user-${uniqueId}`,
    password: 'secret',
    name: 'Dzaru Rizky Fathan Fortuna'
  };

  // 1. Register
  http.post('http://localhost:3000/api/users', JSON.stringify(body), {
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json'
    }
  });

  // 2. Login
  const loginBody = {
    username: `user-${uniqueId}`,
    password: 'secret',
  };

  const response = http.post('http://localhost:3000/api/users/login', JSON.stringify(loginBody), {
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json'
    }
  });

  // 3. Use the token from the login response
  const responseBody = response.json();

  http.get('http://localhost:3000/api/users/current', {
    headers: {
      'Accept': 'application/json',
      'Authorization': responseBody.data.token
    }
  });
}
```

> Reference: [javascript-api/k6-http/response](https://grafana.com/docs/k6/latest/javascript-api/k6-http/response/)

---

## ✅ Test Validation

| Function | On failure | Use for |
| --- | --- | --- |
| `fail()` | Stops the current iteration — remaining code is skipped, the next iteration starts | Steps the rest of the iteration depends on |
| `check()` | Records the failure and **continues** execution; returns a boolean | Assertions reported as success/failure percentages |

### fail()

```javascript
import { fail } from 'k6';
import http from 'k6/http';

export default function () {
  const uniqueId = new Date().getTime();
  const registerBody = {
    username: `user-${uniqueId}`,
    password: 'secret',
    name: 'Dzaru Rizky Fathan Fortuna'
  };

  const registerRequest = http.post('http://localhost:3000/api/users', JSON.stringify(registerBody), {
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json'
    }
  });

  if (registerRequest.status !== 200) {
    fail(`Failed to register user-${uniqueId}`);
  }
}
```

> Reference: [javascript-api/k6/fail](https://grafana.com/docs/k6/latest/javascript-api/k6/fail/)

### check()

`check()` works like an assertion in a unit test, except a failing check does **not** throw. At the end of the run, k6 reports the percentage of checks that passed and failed.

```javascript
import { check, fail } from 'k6';
import http from 'k6/http';

export default function () {
  const uniqueId = new Date().getTime();
  const registerBody = {
    username: `user-${uniqueId}`,
    password: 'secret',
    name: 'Dzaru Rizky Fathan Fortuna'
  };

  const registerRequest = http.post('http://localhost:3000/api/users', JSON.stringify(registerBody), {
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json'
    }
  });

  const checkRegister = check(registerRequest, {
    'register response status 200': (response) => response.status === 200,
    'register response data must not null': (response) => response.json().data !== null
  });

  if (!checkRegister) {
    fail(`Failed to register user-${uniqueId}`);
  }
}
```

> **Tip:** combine the two — `check()` to record the result, then `fail()` when the check returns `false` and the rest of the iteration cannot continue.

---

## 🔍 Execution Context

The `k6/execution` module exposes information about the running test — iteration ID, virtual user ID, scenario, and more.

```javascript
import exec from 'k6/execution';
import http from 'k6/http';
import { check, fail } from 'k6';

export default function () {
  // Each VU logs in as its own user: check1, check2, ...
  const username = `check${exec.vu.idInInstance}`;

  const loginBody = {
    username: username,
    password: 'secret',
  };

  const loginRequest = http.post('http://localhost:3000/api/users/login', JSON.stringify(loginBody), {
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json'
    }
  });

  const checkLogin = check(loginRequest, {
    'login response status must 200': (response) => response.status === 200,
    'login response token must exists': (response) => response.json().data.token !== null
  });

  if (!checkLogin) {
    fail(`Failed to login user-${username}`);
  }
}
```

> Reference: [javascript-api/k6-execution](https://grafana.com/docs/k6/latest/javascript-api/k6-execution/)

---

## 🔄 Test Lifecycle

### Lifecycle Stages

A k6 script runs in four stages:

| Stage | Runs | Required | Description |
| --- | --- | --- | --- |
| **Init** | Once per VU | ✅ Yes | Reads the script — loads imports, defines `options`, initialises variables |
| **`setup()`** | Once, at the start | ❌ No | Prepares data; its return value is passed to `default` and `teardown` |
| **`default()`** | Repeatedly, until the test ends | ✅ Yes | The test itself; receives the `setup()` data as a parameter |
| **`teardown()`** | Once, at the end | ❌ No | Cleans up after testing; also receives the `setup()` data |

### Complete Lifecycle Example

```javascript
import { fail, check } from 'k6';
import exec from 'k6/execution';
import http from 'k6/http';

// Init stage
export const options = {
  vus: 10,
  duration: '10s',
};

// Setup stage — runs once
export function setup() {
  const data = [];
  for (let i = 0; i < 10; i++) {
    data.push({
      first_name: 'Contact',
      last_name: `${i}`,
      email: `contact${i}@example.com`
    });
  }

  return data;
}

export function getToken() {
  const username = `check${exec.vu.idInInstance}`;

  const loginBody = {
    username: username,
    password: 'secret',
  };

  const loginRequest = http.post('http://localhost:3000/api/users/login', JSON.stringify(loginBody), {
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json'
    }
  });

  const checkLogin = check(loginRequest, {
    'login response status must 200': (response) => response.status === 200,
    'login response token must exists': (response) => response.json().data.token !== null
  });

  if (!checkLogin) {
    fail(`Failed to login user-${username}`);
  }

  return loginRequest.json().data.token;
}

// Default function — runs repeatedly, receives setup() data
export default function (data) {
  const token = getToken();
  for (let i = 0; i < data.length; i++) {
    const contact = data[i];
    const response = http.post('http://localhost:3000/api/contacts', JSON.stringify(contact), {
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
        'Authorization': token
      }
    });
    check(response, {
      'create contact status is 200': (response) => response.status === 200
    });
  }
}

// Teardown stage — runs once
export function teardown(data) {
  console.info(`Finish create ${data.length} contacts`);
}
```

---

## 📦 Modular Scripts

Code can live in separate files and be imported like any JavaScript module. This keeps large scripts maintainable and avoids duplicating request code.

```text
src/
├── main.js
└── helper/
    ├── user.js
    └── contact.js
```

### Helper Modules

`src/helper/user.js`

```javascript
import http from 'k6/http';

export function registerUser(body) {
  return http.post('http://localhost:3000/api/users', JSON.stringify(body), {
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json'
    }
  });
}

export function loginUser(body) {
  return http.post('http://localhost:3000/api/users/login', JSON.stringify(body), {
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json'
    }
  });
}

export function getUser(token) {
  return http.get('http://localhost:3000/api/users/current', {
    headers: {
      'Accept': 'application/json',
      'Authorization': token
    }
  });
}
```

`src/helper/contact.js`

```javascript
import http from 'k6/http';

export function createContact(token, contact) {
  return http.post('http://localhost:3000/api/contacts', JSON.stringify(contact), {
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json',
      'Authorization': token
    }
  });
}
```

### Main Script

`src/main.js`

```javascript
import { fail, check } from 'k6';
import exec from 'k6/execution';

import { loginUser } from './helper/user.js';
import { createContact } from './helper/contact.js';

export const options = {
  vus: 10,
  duration: '10s',
};

export function setup() {
  const data = [];
  for (let i = 0; i < 10; i++) {
    data.push({
      first_name: 'Contact',
      last_name: `${i}`,
      email: `contact${i}@example.com`
    });
  }

  return data;
}

export function getToken() {
  const username = `check${exec.vu.idInInstance}`;

  const loginBody = {
    username: username,
    password: 'secret',
  };

  const loginRequest = loginUser(loginBody);

  const checkLogin = check(loginRequest, {
    'login response status must 200': (response) => response.status === 200,
    'login response token must exists': (response) => response.json().data.token !== null
  });

  if (!checkLogin) {
    fail(`Failed to login user-${username}`);
  }

  return loginRequest.json().data.token;
}

export default function (data) {
  const token = getToken();
  for (let i = 0; i < data.length; i++) {
    const response = createContact(token, data[i]);
    check(response, {
      'create contact status is 200': (response) => response.status === 200
    });
  }
}

export function teardown(data) {
  console.info(`Finish create ${data.length} contacts`);
}
```

---

## 🌍 Environment Variables

Settings that should not be hard-coded in the script are read from operating system environment variables through the global `__ENV` object.

```bash
# Set the environment variable
export TOTAL_CONTACT=20
```

```javascript
export function setup() {
  const data = [];
  const totalContact = Number(__ENV.TOTAL_CONTACT) || 10;  // Fallback to 10
  for (let i = 0; i < totalContact; i++) {
    data.push({
      first_name: 'Contact',
      last_name: `${i}`,
      email: `contact${i}@example.com`
    });
  }

  return data;
}
```

> **Note:** every value in `__ENV` is a **string** — convert it with `Number()` before using it as a number, and always provide a fallback.

---

## 🎬 Scenarios

As tests grow, a single `default` function becomes hard to maintain. Splitting them into separate script files means they can no longer run together. **Scenarios** solve this: one script can define multiple functions, each with its own options, all running in the same test.

> Reference: [using-k6/scenarios](https://grafana.com/docs/k6/latest/using-k6/scenarios/)

### Executors

Each scenario's virtual users are driven by an **executor**. Executors fall into three groups:

**Based on number of iterations**

| Executor | Description | Reference |
| --- | --- | --- |
| `shared-iterations` | A total number of iterations is **shared** across all VUs | [docs](https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/shared-iterations/) |
| `per-vu-iterations` | **Each** VU runs a fixed number of iterations — e.g. 10 VUs × 100 iterations each | [docs](https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/per-vu-iterations/) |

**Based on number of virtual users**

| Executor | Description | Reference |
| --- | --- | --- |
| `constant-vus` | A fixed number of VUs iterate continuously until the duration is reached | [docs](https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/constant-vus/) |
| `ramping-vus` | The number of VUs scales up or down stage by stage until all stages complete | [docs](https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/ramping-vus/) |

**Based on iteration rate**

| Executor | Description | Reference |
| --- | --- | --- |
| `constant-arrival-rate` | Starts a constant number of iterations per time unit — e.g. 100 iterations every second for 30 seconds | [docs](https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/constant-arrival-rate/) |
| `ramping-arrival-rate` | Like constant arrival rate, but the iteration rate scales up or down through stages | [docs](https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/ramping-arrival-rate/) |

> **Key Insight:** VU-based executors control *how many users* are active — if the server slows down, fewer iterations happen. Arrival-rate executors control *how many iterations start* per second regardless of response time, which is closer to real-world traffic.

### Shared Iterations Example

```javascript
import { registerUser } from './helper/user.js';

export const options = {
  scenarios: {
    userRegistration: {
      exec: 'userRegistration',      // Function to run
      executor: 'shared-iterations',
      vus: 10,
      iterations: 200,               // Shared across the 10 VUs
      maxDuration: '10s'
    }
  }
};

export function userRegistration() {
  const uniqueId = new Date().getTime();
  const registerRequest = {
    username: `user-${uniqueId}`,
    password: 'secret',
    name: 'Dzaru Rizky Fathan Fortuna'
  };

  registerUser(registerRequest);
}
```

---

## 📊 Metrics

k6 collects built-in metrics automatically, and you can create custom metrics to track specific behaviour.

> Reference: [using-k6/metrics](https://grafana.com/docs/k6/latest/using-k6/metrics/)

### Built-in Metrics

| Metric | Description |
| --- | --- |
| `http_reqs` | Total number of HTTP requests |
| `http_req_duration` | Time taken by HTTP requests |
| `http_req_failed` | Rate of failed requests |
| `vus` | Number of active virtual users |
| `iterations` | Number of completed iterations |

### Custom Metrics

| Type | Description |
| --- | --- |
| **Counter** | Cumulative — counts occurrences |
| **Gauge** | Keeps the latest value |
| **Rate** | Percentage of added values that are non-zero |
| **Trend** | Calculates statistics — min, max, avg, percentiles |

```javascript
import { Counter } from 'k6/metrics';

const myCounter = new Counter('my_custom_counter');

export default function () {
  myCounter.add(1);
}
```

---

## 🎯 Thresholds

By default, a test is always considered **successful**, whether errors occurred or not. **Thresholds** define the limits that decide whether a test passes or fails — if any threshold is not met, the test fails.

```javascript
import { Counter } from 'k6/metrics';
import { registerUser } from './helper/user.js';

const registerCounterSuccess = new Counter('user_registration_counter_success');
const registerCounterError = new Counter('user_registration_counter_error');

export const options = {
  thresholds: {
    user_registration_counter_success: ['count>190'],
    user_registration_counter_error: ['count<10']
  },
  scenarios: {
    userRegistration: {
      exec: 'userRegistration',
      executor: 'shared-iterations',
      vus: 10,
      iterations: 200,
      maxDuration: '10s'
    }
  }
};

export function userRegistration() {
  const uniqueId = new Date().getTime();
  const registerRequest = {
    username: `user-${uniqueId}`,
    password: 'secret',
    name: 'Dzaru Rizky Fathan Fortuna'
  };

  const response = registerUser(registerRequest);
  if (response.status === 200) {
    registerCounterSuccess.add(1);
  } else {
    registerCounterError.add(1);
  }
}
```

> **Tip:** a failed threshold makes `k6 run` exit with a non-zero code — exactly what a CI pipeline needs to fail a build on a performance regression.
>
> Reference: [using-k6/thresholds](https://grafana.com/docs/k6/latest/using-k6/thresholds/)

---

## 📈 Output & Reporting

### Summary Output

After a test finishes, k6 prints a **summary** of the results to the terminal. It can also be saved to a file:

```bash
k6 run script.js --summary-export=summary.json
```

> Reference: [metrics/reference](https://grafana.com/docs/k6/latest/using-k6/metrics/reference/)

### Summary Statistics

By default each trend metric shows `avg, min, med, max, p(90), p(95)`. Customise the list with `summaryTrendStats`:

```javascript
export const options = {
  vus: 10,
  duration: '10s',
  summaryTrendStats: ['avg', 'min', 'med', 'max', 'p(90)', 'p(95)', 'p(99)']
};
```

> Reference: [k6-options/reference#summary-trend-stats](https://grafana.com/docs/k6/latest/using-k6/k6-options/reference/#summary-trend-stats)

### Real-time Output

The summary is only produced once the test ends. For data **while** the test runs, use real-time output — by default k6 can write it to CSV or JSON:

| Format | Command | Reference |
| --- | --- | --- |
| CSV | `k6 run --out csv=test_results.csv script.js` | [docs](https://grafana.com/docs/k6/latest/results-output/real-time/csv/) |
| JSON | `k6 run --out json=test_results.json script.js` | [docs](https://grafana.com/docs/k6/latest/results-output/real-time/json/) |

### Third-party Services

Real-time output can also be streamed to third-party services, but not out of the box — k6 must be **rebuilt** with the extension for that service.

> Reference: [results-output/real-time](https://grafana.com/docs/k6/latest/results-output/real-time/)

### Web Dashboard

k6 has a built-in web dashboard showing real-time and summary output while the test runs:

```bash
export K6_WEB_DASHBOARD=true
k6 run script.js
```

> Reference: [results-output/web-dashboard](https://grafana.com/docs/k6/latest/results-output/web-dashboard/)

---

## 📚 JavaScript Libraries

k6 publishes JavaScript libraries at **jslib.k6.io** that simplify common tasks. Import them by URL:

```javascript
import { uuidv4 } from 'https://jslib.k6.io/k6-utils/1.4.0/index.js';
import { registerUser } from './helper/user.js';

export function userRegistration() {
  const uniqueId = uuidv4();  // Truly unique, unlike Date().getTime() across VUs
  const registerRequest = {
    username: `user-${uniqueId}`,
    password: 'secret',
    name: 'Dzaru Rizky Fathan Fortuna'
  };

  registerUser(registerRequest);
}
```

> **Note:** `new Date().getTime()` can return the same value for two VUs running in the same millisecond, producing duplicate usernames. `uuidv4()` avoids that.
>
> Reference: [jslib.k6.io](https://jslib.k6.io/)

---

## 🔁 Test Execution Flow

Putting it all together, this is what happens on `k6 run script.js`:

```text
k6 run script.js
  ↓
Init stage                 imports, options, variables (once per VU)
  ↓
setup()                    once — returns data
  ↓
Scenarios / executors      start VUs according to options
  ↓
default(data) / exec fn    repeated per VU until iterations or duration end
  ├─ http requests         → built-in metrics (http_reqs, http_req_duration …)
  ├─ check()               → checks pass/fail rate
  ├─ custom metrics        → Counter, Gauge, Rate, Trend
  └─ fail()                → aborts the current iteration only
  ↓
teardown(data)             once — cleanup
  ↓
Thresholds evaluated       → exit code 0 (pass) or non-zero (fail)
  ↓
Summary output             terminal / --summary-export / --out / web dashboard
```

---

## 🎯 Quick Reference

| Concept | Purpose | Key Syntax |
| --- | --- | --- |
| **Install** | Install k6 | `brew install k6`, `k6 --version` |
| **New script** | Generate a template | `k6 new script.js` |
| **Run** | Execute a test | `k6 run script.js` |
| **Options** | Configure load | `export const options = { vus: 10, duration: '30s' }` |
| **Stages** | Ramp VUs up/down | `stages: [{ duration: '10s', target: 20 }]` |
| **HTTP** | Send requests | `http.get(url)`, `http.post(url, JSON.stringify(body), params)` |
| **Response** | Read response data | `response.status`, `response.json()` |
| **fail** | Abort the iteration | `fail('message')` |
| **check** | Non-fatal assertion | `check(res, { 'status 200': (r) => r.status === 200 })` |
| **Execution** | Runtime info | `exec.vu.idInInstance` |
| **Lifecycle** | Setup / test / cleanup | `setup()`, `default(data)`, `teardown(data)` |
| **Env vars** | External settings | `__ENV.TOTAL_CONTACT` |
| **Scenarios** | Multiple workloads | `scenarios: { name: { exec, executor, ... } }` |
| **Executors** | How VUs/iterations run | `shared-iterations`, `per-vu-iterations`, `constant-vus`, `ramping-vus`, `constant-arrival-rate`, `ramping-arrival-rate` |
| **Custom metric** | Track your own values | `new Counter('name')`, `Gauge`, `Rate`, `Trend` |
| **Thresholds** | Pass/fail criteria | `thresholds: { metric: ['count>190'] }` |
| **Summary** | Export results | `--summary-export=summary.json`, `summaryTrendStats` |
| **Real-time** | Stream results | `--out csv=file.csv`, `--out json=file.json` |
| **Dashboard** | Live web UI | `K6_WEB_DASHBOARD=true` |
| **jslib** | Helper libraries | `import { uuidv4 } from 'https://jslib.k6.io/...'` |

---

## 💡 Best Practices

**✅ Do This**

- **Start small** — begin with few VUs and a short duration, then scale up
- **Use realistic data** — test with production-like data and scenarios
- **Monitor system resources** — watch CPU, memory, and network on both the load generator and the server
- **Set clear thresholds** — define success criteria *before* running the test
- **Validate responses with `check()`** — a fast response that returns an error is not a success
- **Move shared request code into helper modules** to keep scripts small and consistent
- **Read configurable values from `__ENV`** with a sensible fallback
- **Generate unique test data** with `uuidv4()` rather than timestamps

**❌ Avoid This**

- **Expecting Node.js APIs or npm packages to work** — k6 runs on Goja, not Node.js
- **Forgetting `JSON.stringify()` and the `Content-Type` header** on JSON request bodies
- **Running heavy logic in the `default` function** — prepare data once in `setup()`
- **Treating a green run as a passed test without thresholds** — without them, k6 always reports success
- **Load testing a production system without permission or monitoring in place**

> Reference: [grafana.com/docs/k6](https://grafana.com/docs/k6/latest/) · [k6 options reference](https://grafana.com/docs/k6/latest/using-k6/k6-options/reference/) · [jslib.k6.io](https://jslib.k6.io/)
