# ⚡ k6 Load Testing - Complete Guide

A comprehensive guide for performance testing with k6, an open-source load testing tool for developers.

---

## 📋 Table of Contents

- [What is k6](#-what-is-k6)
- [System Requirements](#-system-requirements)
- [Installation](#-installation)
- [Project Setup](#-project-setup)
- [Creating Your First Script](#-creating-your-first-script)
- [Running Tests](#-running-tests)
- [Understanding Options](#-understanding-options)
- [HTTP Testing](#-http-testing)
- [Working with Responses](#-working-with-responses)
- [Test Validation](#-test-validation)
- [Execution Context](#-execution-context)
- [Test Lifecycle](#-test-lifecycle)
- [Modular Scripts](#-modular-scripts)
- [Environment Variables](#-environment-variables)
- [Scenarios](#-scenarios)
- [Metrics](#-metrics)
- [Thresholds](#-thresholds)
- [Output & Reporting](#-output--reporting)
- [Advanced Features](#-advanced-features)
- [Best Practices](#-best-practices)
- [Troubleshooting](#-troubleshooting)

---

## 🎯 What is k6

k6 is an open-source load testing tool that makes performance testing easy and developer-friendly. It's free, focused on developers, and easily extensible. k6 helps test application performance, find issues early, and ensures applications perform well under load.

### Key Features

- **Terminal-based application** - Runs on any system, even without GUI
- **JavaScript scripting** - Write test scripts using JavaScript
- **Built-in libraries** - Provides libraries to simplify test scenario creation

### What k6 Does NOT Do

- **Does not run in browser** - Terminal-based tool, doesn't render web pages
- **Does not run on Node.js** - Execution happens in k6 (built with Golang), not Node.js
- **Does not support Node modules** - Cannot directly import npm packages

### Limitations

- Scripts are written in JavaScript, but not all features are supported
- k6 is built using Golang
- Uses Goja library to execute JavaScript code in Golang
- JavaScript features are limited to what Goja supports

---

## 💻 System Requirements

### Minimum Requirements

- **OS:** macOS, Linux, or Windows
- **RAM:** 512 MB
- **Storage:** 100 MB

### Recommended Requirements

- **RAM:** 2 GB+
- **Storage:** 1 GB+

---

## 📦 Installation

### Install on macOS

```bash
brew install k6
```

### Install on Linux (Debian/Ubuntu)

```bash
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6
```

### Install on Windows

```bash
choco install k6
```

### Verify installation

```bash
k6 --version
```

Expected output:

```
k6 v0.xx.x
```

---

## 🚀 Project Setup

- **Create project directory**
  
  ```bash
  mkdir my-k6-project
  cd my-k6-project
  ```

- **Initialize Node.js project**
  
  ```bash
  npm init
  ```

- **Update package.json**
  
  Set the module type to ES6:
  
  ```json
  {
    "name": "my-k6-project",
    "version": "1.0.0",
    "type": "module"
  }
  ```

- **Install k6 library**
  
  ```bash
  npm install k6
  ```

- **Install k6 type definitions**
  
  ```bash
  npm install --save-dev @types/k6
  ```
  
  Note: The npm package only contains TypeScript definitions and metadata, not the actual k6 runtime (which is built in Go).

---

## 📝 Creating Your First Script

- **Generate script template**
  
  ```bash
  k6 new script.js
  ```
  
  This creates a file at the specified location with a simple k6 performance testing script.

- **Create script manually**
  
  Create a file `test.js`:
  
  ```javascript
  import http from 'k6/http';
  import { sleep } from 'k6';
  
  export const options = {
    vus: 10,
    duration: '30s',
  };
  
  export default function() {
    http.get('http://localhost:3000/ping');
    sleep(1);
  }
  ```

### Script Structure

k6 scripts consist of two main parts:

- **Options** - Configuration settings like number of virtual users (VUs) and test duration
- **Default function** - Function executed by k6 according to the options configuration

---

## ▶️ Running Tests

- **Run performance test**
  
  ```bash
  k6 run script.js
  ```

### How k6 Works

When running a script, k6:
- Reads configuration information from options
- Executes the default function according to specified settings
- Calls the default function in parallel across multiple processes
- Continues execution for the specified duration
- Pauses between iterations if sleep() is included

---

## ⚙️ Understanding Options

- **Basic options configuration**
  
  ```bash
  export const options = {
    vus: 10,           // Number of virtual users
    duration: '30s',   // Test duration
  };
  ```

- **Customize summary statistics**
  
  By default, statistics used for each metric are: `[avg, min, med, max, p(90), p(95)]`
  
  To customize, add `summaryTrendStats` key to options:
  
  ```javascript
  export const options = {
    vus: 10,
    duration: '10s',
    summaryTrendStats: ['avg', 'min', 'med', 'max', 'p(90)', 'p(95)', 'p(99)']
  };
  ```
  
  Reference: https://grafana.com/docs/k6/latest/using-k6/k6-options/reference/#summary-trend-stats

- **Use stages for ramping**
  
  ```javascript
  export const options = {
    stages: [
      { duration: '10s', target: 20 },  // Ramp up to 20 users
      { duration: '10s', target: 10 },  // Ramp down to 10 users
      { duration: '10s', target: 0 },   // Ramp down to 0 users
    ],
  };
  ```
  
  This allows increasing user count over a certain duration and decreasing users over another duration.
  
  Reference: https://grafana.com/docs/k6/latest/using-k6/k6-options/reference/#stages

---

## 🌐 HTTP Testing

k6 includes the `k6/http` library for HTTP testing. Note that this library is not for web browser testing. Nearly all HTTP methods are supported by `k6/http`.

Reference: https://grafana.com/docs/k6/latest/javascript-api/k6-http/

- **Perform GET request**
  
  ```javascript
  import http from 'k6/http';
  
  export default function() {
    http.get('http://localhost:3000/api/users');
  }
  ```

- **Perform POST request with JSON body**
  
  ```javascript
  import http from 'k6/http';
  
  export default function() {
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

---

## 📡 Working with Responses

Every function call in the `k6/http` library returns an HTTP response. You can capture the HTTP response information and use it in subsequent HTTP requests.

Reference: https://grafana.com/docs/k6/latest/javascript-api/k6-http/response/

- **Capture and use response**
  
  ```javascript
  export default function() {
    const uniqueId = new Date().getTime();
    const body = {
      username: `user-${uniqueId}`,
      password: 'secret',
      name: "Dzaru Rizky Fathan Fortuna"
    };
  
    http.post('http://localhost:3000/api/users', JSON.stringify(body), {
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json'
      }
    });
  
    const loginBody = {
      username: `user-${uniqueId}`,
      password: 'secret',
    };
  
    const response = http.post('http://localhost:3000/api/users/login', 
      JSON.stringify(loginBody), {
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json'
      }
    });
  
    const responseBody = response.json();
    if (!responseBody?.data?.token) {
      console.error("TOKEN NOT FOUND. Raw response:", response.body);
      return;
    }
  
    http.get('http://localhost:3000/api/users/current', {
      headers: {
        'Accept': 'application/json',
        "Authorization": responseBody.data.token
      }
    });
  }
  ```

---

## ✅ Test Validation

### Use fail() Function

When performing performance tests, sometimes you want to know if a request succeeded or failed. To indicate that a test failed, use the `fail()` function from the k6 library. When `fail()` is called, the current iteration stops, subsequent code is not executed, and execution continues to the next iteration.

Reference: https://grafana.com/docs/k6/latest/javascript-api/k6/fail/

- **Example using fail()**
  
  ```javascript
  import { fail } from 'k6';
  import http from 'k6/http';
  
  export default function() {
    const uniqueId = new Date().getTime();
    const registerBody = {
      username: `user-${uniqueId}`,
      password: 'secret',
      name: "Dzaru Rizky Fathan Fortuna"
    };
  
    const registerRequest = http.post('http://localhost:3000/api/users',
      JSON.stringify(registerBody), {
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json'
      }
    });
  
    if (registerRequest.status !== 200) {
      fail(`Failed to register user-${uniqueId}`);
    }
  
    const loginBody = {
      username: `user-${uniqueId}`,
      password: 'secret',
    };
  
    const loginRequest = http.post('http://localhost:3000/api/users/login',
      JSON.stringify(loginBody), {
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json'
      }
    });
  
    if (loginRequest.status !== 200) {
      fail(`Failed to login user-${uniqueId}`);
    }
  
    const loginResponse = loginRequest.json();
  
    const currentResponse = http.get('http://localhost:3000/api/users/current', {
      headers: {
        'Accept': 'application/json',
        "Authorization": loginResponse.data.token
      }
    });
  
    if (currentResponse.status !== 200) {
      fail(`Failed to get user`);
    }
  }
  ```

### Use checks for Validation

k6 has a feature to perform checks using the `check()` function. Checks are similar to assertions in unit tests, but the difference is if a check fails, it does not cause an error, meaning code execution continues. `check()` returns a boolean indicating whether the check succeeded or failed. After checks complete, k6 reports the success and failure percentages.

- **Example using check()**
  
  ```javascript
  import { check, fail } from 'k6';
  import http from 'k6/http';
  
  export default function() {
    const uniqueId = new Date().getTime();
    const registerBody = {
      username: `user-${uniqueId}`,
      password: 'secret',
      name: "Dzaru Rizky Fathan Fortuna"
    };
  
    const registerRequest = http.post('http://localhost:3000/api/users',
      JSON.stringify(registerBody), {
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json'
      }
    });
  
    const checkRegister = check(registerRequest, {
      "register response status 200": (response) => response.status === 200,
      "register response data must not null": (response) => response.json().data !== null
    });
  
    if (!checkRegister) {
      fail(`Failed to register user-${uniqueId}`);
    }
  }
  ```

---

## 🔍 Execution Context

When running tests, sometimes you need information about the execution k6 is performing, such as iteration ID, virtual user ID, and more. k6 provides the `k6/execution` module containing this information.

Reference: https://grafana.com/docs/k6/latest/javascript-api/k6-execution/

- **Example using execution context**
  
  ```javascript
  import exec from "k6/execution";
  import http from 'k6/http';
  import { check, fail } from 'k6';
  
  export default function() {
    const username = `check${exec.vu.idInInstance}`;
    console.log(username);
    
    const loginBody = {
      username: username,
      password: 'rahasia',
    };
  
    const loginRequest = http.post('http://localhost:3000/api/users/login',
      JSON.stringify(loginBody), {
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json'
      }
    });
  
    const checkLogin = check(loginRequest, {
      "login response status must 200": (response) => response.status === 200,
      "login response token must exists": (response) => response.json().data.token !== null
    });
  
    if (!checkLogin) {
      fail(`Failed to login user-${username}`);
    }
  }
  ```

---

## 🔄 Test Lifecycle

When k6 runs a script, it executes in several phases called the lifecycle. k6 phases start from init, setup, default function, and teardown.

### Lifecycle Phases

- **Init phase**
  
  The phase where k6 reads all script files. Executed once and is mandatory. Loads imports, defines options, and initializes variables.

- **Setup function**
  
  Called once at the beginning, used to prepare data. Can return data that will be used by the default function. Not mandatory to create.

- **Default function**
  
  Called continuously until the test duration is complete. If `setup()` returns data, the default function can receive it as a parameter. Mandatory to create.

- **Teardown function**
  
  Executed after testing is complete. Not mandatory to create.

- **Complete lifecycle example**
  
  ```javascript
  import { fail, check } from 'k6';
  import exec from "k6/execution";
  import http from 'k6/http';
  
  export const options = {
    vus: 10,
    duration: '10s',
  };
  
  export function setup() {
    const data = [];
    for (let i = 0; i < 10; i++) {
      data.push({
        "first_name": "Contact",
        "last_name": `${i}`,
        "email": `contact${i}@example.com`
      });
    }
  
    return data;
  }
  
  export function getToken() {
    const username = `check${exec.vu.idInInstance}`;
    
    const loginBody = {
      username: username,
      password: 'rahasia',
    };
  
    const loginRequest = http.post('http://localhost:3000/api/users/login',
      JSON.stringify(loginBody), {
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json'
      }
    });
  
    const checkLogin = check(loginRequest, {
      "login response status must 200": (response) => response.status === 200,
      "login response token must exists": (response) => response.json().data.token !== null
    });
  
    if (!checkLogin) {
      fail(`Failed to login user-${username}`);
    }
  
    return loginRequest.json().data.token;
  }
  
  export default function(data) {
    const token = getToken();
    for (let i = 0; i < data.length; i++) {
      const contact = data[i];
      const response = http.post('http://localhost:3000/api/contacts',
        JSON.stringify(contact), {
        headers: {
          'Accept': 'application/json',
          'Content-Type': 'application/json',
          'Authorization': token
        }
      });
      check(response, {
        "create contact status is 200": (response) => response.status === 200
      });
    }
  }
  
  export function teardown(data) {
    console.info(`Finish create ${data.length} contacts`);
  }
  ```

---

## 📦 Modular Scripts

JavaScript has a modules feature that can be used to store code in separate files from k6 scripts. To use it, simply import the file like standard JavaScript modules. This is very useful when scripts become large, and to avoid code duplication.

### Project Structure Example

```
my-k6-project/
├── src/
│   └── main.js
├── helper/
│   ├── user.js
│   └── contact.js
└── package.json
```

### Main Script

- **src/main.js**
  
  ```javascript
  import { fail, check } from 'k6';
  import exec from "k6/execution";
  
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
        "first_name": "Contact",
        "last_name": `${i}`,
        "email": `contact${i}@example.com`
      });
    }
  
    return data;
  }
  
  export function getToken() {
    const username = `check${exec.vu.idInInstance}`;
    
    const loginBody = {
      username: username,
      password: 'rahasia',
    };
  
    const loginRequest = loginUser(loginBody);
  
    const checkLogin = check(loginRequest, {
      "login response status must 200": (response) => response.status === 200,
      "login response token must exists": (response) => response.json().data.token !== null
    });
  
    if (!checkLogin) {
      fail(`Failed to login user-${username}`);
    }
  
    return loginRequest.json().data.token;
  }
  
  export default function(data) {
    const token = getToken();
    for (let i = 0; i < data.length; i++) {
      const contact = data[i];
      const response = createContact(token, contact);
      check(response, {
        "create contact status is 200": (response) => response.status === 200
      });
    }
  }
  
  export function teardown(data) {
    console.info(`Finish create ${data.length} contacts`);
  }
  ```

### Helper Modules

- **helper/contact.js**
  
  ```javascript
  import http from 'k6/http';
  import { check } from 'k6';
  
  export function createContact(token, contact) {
    const response = http.post('http://localhost:3000/api/contacts',
        JSON.stringify(contact), {
        headers: {
          'Accept': 'application/json',
          'Content-Type': 'application/json',
          'Authorization': token
        }
      });
      check(response, {
        "create contact status is 200": (response) => response.status === 200
      });
    
    return response;
  }
  ```

- **helper/user.js**
  
  ```javascript
  import http from 'k6/http';
  import { check } from 'k6';
  
  export function registerUser(body) {
    const registerRequest = http.post('http://localhost:3000/api/users',
      JSON.stringify(body), {
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json'
      }
    });
  
    check(registerRequest, {
      "register response status 200": (response) => response.status === 200,
      "register response data must not null": (response) => response.json().data !== null
    });
  
    return registerRequest;
  }
  
  export function loginUser(body) {
      const loginRequest = http.post('http://localhost:3000/api/users/login',
      JSON.stringify(body), {
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json'
      }
    });
  
    check(loginRequest, {
      "login response status must 200": (response) => response.status === 200,
      "login response token must exists": (response) => response.json().data.token !== null
    });
  
    return loginRequest;
  }
  
  export function getUser(token) {
    const currentResponse = http.get('http://localhost:3000/api/users/current', {
      headers: {
        'Accept': 'application/json',
        "Authorization": token
      }
    });
  
    check(currentResponse, {
      "current response status must 200": (response) => response.status === 200,
      "current response data must not null": (response) => response.json().data !== null
    });
  
    return currentResponse;
  }
  ```

---

## 🌍 Environment Variables

Sometimes when creating scripts, there are settings that cannot be hardcoded in the script. These settings are usually stored in operating system environment variables. To read environment variables, use the `__ENV` variable in your script.

- **Set environment variable**
  
  Linux/macOS:
  
  ```bash
  export TOTAL_CONTACT=20
  ```
  
  Windows (Command Prompt):
  
  ```cmd
  set TOTAL_CONTACT=20
  ```
  
  Windows (PowerShell):
  
  ```powershell
  $env:TOTAL_CONTACT=20
  ```

- **Use environment variable in script**
  
  ```javascript
  export function setup() {
    const data = [];
    const totalContact = Number(__ENV.TOTAL_CONTACT) || 10;
    for (let i = 0; i < totalContact; i++) {
      data.push({
        "first_name": "Contact",
        "last_name": `${i}`,
        "email": `contact${i}@example.com`
      });
    }
  
    return data;
  }
  ```

---

## 🎬 Scenarios

As test scenarios grow, the default function code becomes larger. More code makes maintenance more difficult. To make maintenance easier, you can create multiple k6 script files. However, the problem is you cannot run them all at once. Fortunately, k6 has a scenario feature where you can create multiple functions with different options.

### Scenario Executors

For each scenario, virtual users are executed by executors. There are many types of executors available based on number of iterations, number of virtual users, or iteration capacity (iteration rate).

### Number of Iterations Executors

- **Shared iterations**
  
  Executor where total iterations are shared across all virtual users.
  
  Reference: https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/shared-iterations/
  
  Example:
  
  ```javascript
  import { registerUser } from "./helper/user.js";
  
  export const options = {
    scenarios: {
      userRegistration: {
        exec: "userRegistration",
        executor: "shared-iterations",
        vus: 10,
        iterations: 200,
        maxDuration: "10s"
      }
    }
  };
  
  export function userRegistration() {
    const uniqueId = new Date().getTime();
    const registerRequest = {
      username: `user-${uniqueId}`,
      password: `secret`,
      name: "Dzaru Rizky Fathan Fortuna"
    };
  
    const response = registerUser(registerRequest);
    if (response.status === 200) {
      registerCounterSuccess.add(1);
    } else {
      registerCounterError.add(1);
    }
  }
  ```

---

## 🎯 Thresholds

By default, test results are always considered successful, whether there are errors or not. You can set thresholds to determine the limit for whether a test is successful or failed. If test results meet the specified thresholds, they are considered successful; if not, they are considered failed.

Reference: https://grafana.com/docs/k6/latest/using-k6/thresholds/

### Metric Thresholds

Thresholds are applied to metrics, whether built-in or custom metrics you create. Threshold rules must follow the metric type being used.

Reference: https://grafana.com/docs/k6/latest/using-k6/metrics/

- **Example using thresholds**
  
  ```javascript
  import { Counter } from 'k6/metrics';
  import { registerUser } from "./helper/user.js";
  
  const registerCounterSuccess = new Counter("user_registration_counter_success");
  const registerCounterError = new Counter("user_registration_counter_error");
  
  export const options = {
    thresholds: {
      user_registration_counter_success: ["count>190"],
      user_registration_counter_error: ["count<10"]
    },
    scenarios: {
      userRegistration: {
        exec: "userRegistration",
        executor: "shared-iterations",
        vus: 10,
        iterations: 200,
        maxDuration: "10s"
      }
    }
  };
  
  export function userRegistration() {
    const uniqueId = new Date().getTime();
    const registerRequest = {
      username: `user-${uniqueId}`,
      password: `secret`,
      name: "Dzaru Rizky Fathan Fortuna"
    };
  
    const response = registerUser(registerRequest);
    if (response.status === 200) {
      registerCounterSuccess.add(1);
    } else {
      registerCounterError.add(1);
    }
  }
  ```

### Threshold Syntax

Counter/Rate metrics:
- `count>100` - Count greater than 100
- `rate<0.1` - Rate less than 10%

Trend metrics:
- `p(90)<300` - 90th percentile under 300ms
- `p(95)<500` - 95th percentile under 500ms
- `avg<200` - Average under 200ms
- `med<150` - Median under 150ms
- `min<50` - Minimum under 50ms
- `max<2000` - Maximum under 2000ms

### Threshold Operators

- `>` - Greater than
- `>=` - Greater than or equal
- `<` - Less than
- `<=` - Less than or equal
- `==` - Equal to
- `!=` - Not equal to

---

## 📈 Output & Reporting

### Summary Output

After k6 runs performance testing from your script, it produces a summary output of the test results. By default, the output is displayed in the console/terminal.

- **Save summary to JSON file**
  
  ```bash
  k6 run script.js --summary-export=summary.json
  ```
  
  Reference: https://grafana.com/docs/k6/latest/using-k6/metrics/reference/

### Summary Statistics

By default, statistics used for each metric are: `[avg, min, med, max, p(90), p(95)]`

- **Customize summary statistics**
  
  ```javascript
  export const options = {
    vus: 10,
    duration: '10s',
    summaryTrendStats: ['avg', 'min', 'med', 'max', 'p(90)', 'p(95)', 'p(99)']
  };
  ```
  
  Reference: https://grafana.com/docs/k6/latest/using-k6/k6-options/reference/#summary-trend-stats

### Real-time Output

Summary output only creates reports after the performance test is complete. If you need real-time information, you can use real-time output from k6. By default, k6 can send real-time output to JSON or CSV files.

- **Output to CSV**
  
  ```bash
  k6 run --out csv=test_results.csv script.js
  ```
  
  Reference: https://grafana.com/docs/k6/latest/results-output/real-time/csv/

- **Output to JSON**
  
  ```bash
  k6 run --out json=test_results.json script.js
  ```
  
  Reference: https://grafana.com/docs/k6/latest/results-output/real-time/json/

### Stream to Third-party Services

Besides files, real-time output can also be sent to third-party services. However, sending to third-party services cannot be done by default. You must rebuild the k6 application by adding the third-party library.

Reference: https://grafana.com/docs/k6/latest/results-output/real-time/

### Web Dashboard

k6 has a web dashboard feature to view real-time output and summary output while k6 is running tests.

- **Enable web dashboard**
  
  ```bash
  export K6_WEB_DASHBOARD=true
  k6 run script.js
  ```
  
  Reference: https://grafana.com/docs/k6/latest/results-output/web-dashboard/

---

## 🚀 Advanced Features

### JavaScript Libraries

k6 provides JavaScript libraries that can be used to simplify script creation. It's recommended to read the documentation to understand the purpose of each JavaScript library provided.

Reference: https://grafana.com/docs/k6/latest/javascript-api/

### Remote Modules

Besides JavaScript libraries provided directly in k6, k6 has libraries that can be used remotely.

Reference: https://jslib.k6.io/

- **Example using remote module**
  
  ```javascript
  import { uuidv4 } from 'https://jslib.k6.io/k6-utils/1.4.0/index.js';
  import { registerUser } from "./helper/user.js";
  
  export function userRegistration() {
    const uniqueId = uuidv4();
    const registerRequest = {
      username: `user-${uniqueId}`,
      password: `secret`,
      name: "Dzaru Rizky Fathan Fortuna"
    };
  
    const response = registerUser(registerRequest);
    if (response.status === 200) {
      registerCounterSuccess.add(1);
    } else {
      registerCounterError.add(1);
    }
  }
  ```

---

## 💡 Best Practices

### Script Organization

Organize your project with clear directory structure:

```
k6-project/
├── src/
│   ├── tests/
│   │   ├── load-test.js
│   │   ├── stress-test.js
│   │   └── spike-test.js
│   └── helpers/
│       ├── api/
│       │   ├── user.js
│       │   └── product.js
│       └── utils/
│           └── common.js
├── data/
│   ├── users.json
│   └── products.csv
└── package.json
```

### Performance Optimization

- **Optimize script execution**
  
  - Reuse connections with `http.batch()`
  - Minimize `sleep()` duration
  - Use appropriate VU count
  - Avoid excessive logging
  - Cache static data in setup phase

- **Batch requests example**
  
  ```javascript
  import http from 'k6/http';
  
  export default function() {
    const responses = http.batch([
      ['GET', 'http://localhost:3000/api/users'],
      ['GET', 'http://localhost:3000/api/products'],
      ['GET', 'http://localhost:3000/api/orders']
    ]);
  }
  ```

### Test Types

- **Load test**
  
  Validate performance under expected load:
  
  ```javascript
  export const options = {
    stages: [
      { duration: '5m', target: 100 },  // Ramp up
      { duration: '10m', target: 100 }, // Stay at peak
      { duration: '5m', target: 0 },    // Ramp down
    ],
  };
  ```

- **Stress test**
  
  Find system breaking point:
  
  ```javascript
  export const options = {
    stages: [
      { duration: '2m', target: 100 },
      { duration: '5m', target: 100 },
      { duration: '2m', target: 200 },
      { duration: '5m', target: 200 },
      { duration: '2m', target: 300 },
      { duration: '5m', target: 300 },
      { duration: '10m', target: 0 },
    ],
  };
  ```

- **Spike test**
  
  Validate system behavior under sudden load:
  
  ```javascript
  export const options = {
    stages: [
      { duration: '10s', target: 100 },
      { duration: '1m', target: 100 },
      { duration: '10s', target: 1400 },  // Spike
      { duration: '3m', target: 1400 },
      { duration: '10s', target: 100 },
      { duration: '3m', target: 100 },
      { duration: '10s', target: 0 },
    ],
  };
  ```

- **Soak test**
  
  Verify system stability over extended time:
  
  ```javascript
  export const options = {
    stages: [
      { duration: '2m', target: 400 },
      { duration: '3h56m', target: 400 }, // Long duration
      { duration: '2m', target: 0 },
    ],
  };
  ```

### Error Handling

- **Always handle errors**
  
  ```javascript
  import { check, fail } from 'k6';
  import http from 'k6/http';
  
  export default function() {
    try {
      const response = http.get('http://localhost:3000/api/users');
      
      const success = check(response, {
        'status is 200': (r) => r.status === 200,
        'response has data': (r) => r.json().data !== undefined
      });
      
      if (!success) {
        fail('Request validation failed');
      }
    } catch (error) {
      console.error(`Error: ${error.message}`);
    }
  }
  ```

### Data Management

- **Use external data files**
  
  ```javascript
  import { SharedArray } from 'k6/data';
  import papaparse from 'https://jslib.k6.io/papaparse/5.1.1/index.js';
  
  const csvData = new SharedArray('users', function() {
    return papaparse.parse(open('./data/users.csv'), { header: true }).data;
  });
  
  export default function() {
    const user = csvData[Math.floor(Math.random() * csvData.length)];
    // Use user data
  }
  ```

### Environment Configuration

- **Use environment variables for configuration**
  
  ```javascript
  const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';
  const API_KEY = __ENV.API_KEY;
  
  export default function() {
    http.get(`${BASE_URL}/api/users`, {
      headers: {
        'Authorization': `Bearer ${API_KEY}`
      }
    });
  }
  ```

- **Run with environment variables**
  
  ```bash
  BASE_URL=https://api.production.com API_KEY=secret123 k6 run script.js
  ```

---

## 🐛 Troubleshooting

### Common Issues

- **Script fails with "module not found"**
  
  Solution:
  - Verify import path is correct
  - Use relative paths: `./helper/user.js`
  - Ensure file exists in specified location

- **High memory usage**
  
  Solution:
  - Reduce number of VUs
  - Optimize script (remove unnecessary operations)
  - Use `--compatibility-mode=base` flag
  - Increase system resources

- **Connection timeouts**
  
  Solution:
  
  ```javascript
  export const options = {
    httpDebug: 'full', // Enable HTTP debugging
  };
  
  // Or increase timeout
  http.get('http://localhost:3000/api', {
    timeout: '60s'
  });
  ```

- **"Cannot read property of undefined"**
  
  Solution - Add null checks:
  
  ```javascript
  const data = response.json();
  if (data && data.token) {
    // Use token
  }
  ```

- **Test results vary between runs**
  
  Solution:
  - Use fixed seed for random data
  - Implement proper warm-up period
  - Ensure consistent test environment
  - Check for external dependencies

### Debug Mode

- **Enable verbose logging**
  
  ```bash
  k6 run --http-debug="full" script.js
  ```

- **Log request/response**
  
  ```javascript
  import http from 'k6/http';
  
  export default function() {
    const response = http.get('http://localhost:3000/api/users');
    console.log(`Status: ${response.status}`);
    console.log(`Body: ${response.body}`);
    console.log(`Headers: ${JSON.stringify(response.headers)}`);
  }
  ```

### Performance Debugging

- **Check VU execution time**
  
  ```javascript
  import { check } from 'k6';
  import http from 'k6/http';
  
  export default function() {
    const start = Date.now();
    
    http.get('http://localhost:3000/api/users');
    
    const duration = Date.now() - start;
    console.log(`Request took ${duration}ms`);
  }
  ```

---

## 📚 Additional Resources

### Official Documentation

- [k6 Documentation](https://grafana.com/docs/k6/latest/)
- [k6 API Reference](https://grafana.com/docs/k6/latest/javascript-api/)
- [k6 Examples](https://grafana.com/docs/k6/latest/examples/)

### Community Resources

- [k6 GitHub Repository](https://github.com/grafana/k6)
- [k6 Community Forum](https://community.grafana.com/c/grafana-k6/)
- [k6 Extensions](https://k6.io/docs/extensions/)

### Learning Resources

- [Load Testing Best Practices](https://grafana.com/docs/k6/latest/testing-guides/)
- [Performance Testing Guide](https://grafana.com/docs/k6/latest/testing-guides/api-load-testing/)
- [k6 YouTube Channel](https://www.youtube.com/@k6test)

### Related Tools

- **Grafana** - Visualization and dashboards
- **InfluxDB** - Time-series database for k6 metrics
- **Prometheus** - Monitoring and alerting
- **Jenkins** - CI/CD integration
- **GitHub Actions** - Automated testing workflows

---

## 🎓 Quick Reference

### Common Commands

```bash
# Run test
k6 run script.js

# Run with custom VUs and duration
k6 run --vus 10 --duration 30s script.js

# Export results
k6 run --out json=results.json script.js

# Enable web dashboard
K6_WEB_DASHBOARD=true k6 run script.js

# Run with environment variables
API_URL=https://api.example.com k6 run script.js

# Quiet mode (less output)
k6 run --quiet script.js

# Summary only
k6 run --summary-export=summary.json script.js
```

### Script Template

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 10,
  duration: '30s',
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.1'],
  },
};

export function setup() {
  // Setup code (runs once)
  return { data: 'test' };
}

export default function(data) {
  // Main test code (runs repeatedly)
  const response = http.get('https://test.k6.io');
  
  check(response, {
    'status is 200': (r) => r.status === 200,
  });
  
  sleep(1);
}

export function teardown(data) {
  // Teardown code (runs once)
  console.log('Test completed');
}
```

---

## 🎯 Quick Reference Table

| Command | Description |
|---------|-------------|
| `k6 --version` | Show k6 version |
| `k6 new` | Generate script template |
| `k6 run` | Execute test script |
| `k6 run --vus` | Set virtual users |
| `k6 run --duration` | Set test duration |
| `k6 run --out` | Export results to file |
| `k6 run --summary-export` | Export summary as JSON |
| `k6 run --http-debug` | Enable HTTP debugging |
| `k6 run --quiet` | Reduce output verbosity |
| `export K6_WEB_DASHBOARD=true` | Enable web dashboard |

---

## 📝 License

k6 is open-source software licensed under the [AGPL-3.0 License](https://github.com/grafana/k6/blob/master/LICENSE.md).

---

## 🤝 Contributing

Contributions are welcome! Check out the [k6 Contributing Guide](https://github.com/grafana/k6/blob/master/CONTRIBUTING.md) for more information.

---

**Happy Load Testing! 🚀**
      password: `secret`,
      name: "Dzaru Rizky Fathan Fortuna"
    };
  
    registerUser(registerRequest);
  }
  ```

- **Per VU iterations**
  
  Executor where each virtual user is assigned a specific number of iterations.
  
  Reference: https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/per-vu-iterations/

### Number of Virtual Users Executors

- **Constant VUs**
  
  Executor with a specified number of virtual users, where each virtual user continuously performs iterations until the specified duration time.
  
  Reference: https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/constant-vus/
  
  Example:
  
  ```javascript
  import { createContact } from "./helper/contact.js";
  import exec from "k6/execution";
  
  export const options = {
    scenarios: {
      contactCreation: {
        exec: "contactCreation",
        executor: "constant-vus",
        vus: 10,
        duration: "10s"
      }
    }
  };
  
  export function contactCreation() {
    const number = (exec.vu.idInInstance % 9) + 1;
    const username = `check${number}`;
    const loginRequest = { username: username, password: 'rahasia' };
  
    const loginResponse = loginUser(loginRequest);
    const token = loginResponse.json().data.token;
  
    const contact = {
      "first_name": "User",
      "last_name": `${number}`,
      "email": `user${number}@gmail.com`
    };
  
    createContact(token, contact);
  }
  ```

- **Ramping VUs**
  
  Executor that creates virtual users according to the number specified in each stage, moving up or down following the next stage until all stages complete.
  
  Reference: https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/ramping-vus/

### Iteration Rate Executors

- **Constant arrival rate**
  
  Executor that performs iterations at a constant rate as specified.
  
  Reference: https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/constant-arrival-rate/

- **Ramping arrival rate**
  
  Executor similar to constant arrival rate, except the number of iterations can increase and decrease following specified stages.
  
  Reference: https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/ramping-arrival-rate/

### Execution Function

One advantage of scenarios is you can specify which function to run in each iteration. Each scenario can use the same or different functions. Specify the function name in the `exec` attribute.

- **Multiple scenarios example**
  
  ```javascript
  import { registerUser, loginUser } from "./helper/user.js";
  import { createContact } from "./helper/contact.js";
  import exec from "k6/execution";
  
  export const options = {
    scenarios: {
      userRegistration: {
        exec: "userRegistration",
        executor: "shared-iterations",
        vus: 10,
        iterations: 200,
        maxDuration: "10s"
      },
      contactCreation: {
        exec: "contactCreation",
        executor: "constant-vus",
        vus: 10,
        duration: "10s"
      }
    }
  };
  
  export function userRegistration() {
    const uniqueId = new Date().getTime();
    const registerRequest = {
      username: `user-${uniqueId}`,
      password: `secret`,
      name: "Dzaru Rizky Fathan Fortuna"
    };
  
    registerUser(registerRequest);
  }
  
  export function contactCreation() {
    const number = (exec.vu.idInInstance % 9) + 1;
    const username = `check${number}`;
    const loginRequest = { username: username, password: 'rahasia' };
  
    const loginResponse = loginUser(loginRequest);
    const token = loginResponse.json().data.token;
  
    const contact = {
      "first_name": "User",
      "last_name": `${number}`,
      "email": `user${number}@gmail.com`
    };
  
    createContact(token, contact);
  }
  ```

---

## 📊 Metrics

When running tests using k6, the output results are in the form of metric data. k6 divides metrics into several categories:

- **Counters** - Count quantities
- **Gauges** - Track smallest, largest, and latest values
- **Rates** - Track how often non-zero values appear
- **Trends** - Calculate statistics for multiple values (like average, percentiles, etc.)

### Built-in Metrics

By default, k6 provides built-in metrics, so you don't need to create metrics manually. You can see the results from metrics in the output.

Reference: https://grafana.com/docs/k6/latest/using-k6/metrics/reference/

### Custom Metrics

If the built-in metrics from k6 are not sufficient, you can create your own metrics. k6 provides libraries to create all available metric types. Note that you must manually add data to the metrics.

Reference: https://grafana.com/docs/k6/latest/using-k6/metrics/

- **Example creating custom metrics**
  
  ```javascript
  import { Counter } from 'k6/metrics';
  import { registerUser } from "./helper/user.js";
  
  const registerCounterSuccess = new Counter("user_registration_counter_success");
  const registerCounterError = new Counter("user_registration_counter_error");
  
  export const options = {
    scenarios: {
      userRegistration: {
        exec: "userRegistration",
        executor: "shared-iterations",
        vus: 10,
        iterations: 200,
        maxDuration: "10s"
      }
    }
  };
  
  export function userRegistration() {
    const uniqueId = new Date().getTime();
    const registerRequest = {
      username: `user-${uniqueId}`,