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

## 📦 Installation

- Install on macOS

  ```bash
  brew install k6
  ```

 - Verify installation

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

- **Script structure**

  k6 scripts consist of two main parts:
  
  - **Options** - Configuration settings like number of virtual users (VUs) and test duration
  - **Default function** - Function executed by k6 according to the options configuration

---

## ▶️ Running Tests

- **Run performance test**
  
  ```bash
  k6 run script.js
  ```

- **How k6 works**

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
  
  Reference: `https://grafana.com/docs/k6/latest/using-k6/k6-options/reference/#summary-trend-stats`

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
  
  Reference: `https://grafana.com/docs/k6/latest/using-k6/k6-options/reference/#stages`

---

## 🌐 HTTP Testing

k6 includes the `k6/http` library for HTTP testing. Note that this library is not for web browser testing. Nearly all HTTP methods are supported by `k6/http`.


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
- **Reference**: `https://grafana.com/docs/k6/latest/javascript-api/k6-http`

---

## 📡 Working with Responses

Every function call in the `k6/http` library returns an HTTP response. You can capture the HTTP response information and use it in subsequent HTTP requests.

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
    
      const response = http.post('http://localhost:3000/api/users/login',  JSON.stringify(loginBody), {
        headers: {
          'Accept': 'application/json',
          'Content-Type': 'application/json'
        }
      });
    
      const responseBody = response.json();
      
      http.get('http://localhost:3000/api/users/current', {
        headers: {
          'Accept': 'application/json',
          "Authorization": responseBody.data.token
        }
      });
    }
    ```
    
- **Reference:** `https://grafana.com/docs/k6/latest/javascript-api/k6-http/response`  
  
---

## ✅ Test Validation

- **Use fail() Function**

  When performing performance tests, sometimes you want to know if a request succeeded or failed. To indicate that a test failed, use the `fail()` function from the k6 library. When `fail()` is called, the current iteration stops, subsequent code is not executed, and execution continues to the next iteration.

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
    
  **Reference**: `https://grafana.com/docs/k6/latest/javascript-api/k6/fail`
    
- **Use checks for Validation**

  k6 has a feature to perform checks using the `check()` function. Checks are similar to assertions in unit tests, but the difference is if a check fails, it does not cause an error, meaning code execution continues. `check()` returns a boolean indicating whether the check succeeded or failed. After checks complete, k6 reports the success and failure percentages.
    
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
  
    const registerRequest = http.post('http://localhost:3000/api/users', JSON.stringify(registerBody), {
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

- **Example using execution context**
  
  ```javascript
  import exec from "k6/execution";
  import http from 'k6/http';
  import { check, fail } from 'k6';
  
  export default function() {
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
      "login response status must 200": (response) => response.status === 200,
      "login response token must exists": (response) => response.json().data.token !== null
    });
  
    if (!checkLogin) {
      fail(`Failed to login user-${username}`);
    }
  }
  ```
  
- **Reference:** `https://grafana.com/docs/k6/latest/javascript-api/k6-execution`

---

## 🔄 Test Lifecycle

When k6 runs a script, it executes in several phases called the lifecycle. k6 phases start from init, setup, default function, and teardown.

- **Lifecycle phases**

  - **Init phase:** The phase where k6 reads all script files. Executed once and is mandatory. Loads imports, defines options, and initializes variables.
  - **Setup function:** Called once at the beginning, used to prepare data. Can return data that will be used by the default function. Not mandatory to create.
  - **Default function:** Called continuously until the test duration is complete. If `setup()` returns data, the default function can receive it as a parameter. Mandatory to create.
  - **Teardown function:** Executed after testing is complete. Not mandatory to create.

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
      password: 'secret',
    };
  
    const loginRequest = http.post('http://localhost:3000/api/users/login', JSON.stringify(loginBody), {
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
      const response = http.post('http://localhost:3000/api/contacts', JSON.stringify(contact), {
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

- **Main Script**

  `src/main.js`
  
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

- **Helper Modules**

  `helper/contact.js`
    
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
      
      return response;
    }
    ```
  
  `helper/user.js`
    
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
    
      return loginRequest;
    }
    
    export function getUser(token) {
      const currentResponse = http.get('http://localhost:3000/api/users/current', {
        headers: {
          'Accept': 'application/json',
          "Authorization": token
        }
      });
    
      return currentResponse;
    }
    ```

---

## 🌍 Environment Variables

Sometimes when creating scripts, there are settings that cannot be hardcoded in the script. These settings are usually stored in operating system environment variables. To read environment variables, use the `__ENV` variable in your script.

- **Set environment variable**
  
  ```bash
  export TOTAL_CONTACT=20
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

- **Number of Iterations Executors**

   - **Shared iterations**
   
     Executor where total iterations are shared across all virtual users.
          
      `Example:`
      
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
            password: 'secret',
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
     
     **Reference:** `https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/shared-iterations`
     
   - **Pre-VU Iterations**
  
      An executor where each virtual user has a predetermined number of iterations. For example, creating 10 virtual users where each virtual user will perform 100 iterations.

      **Reference:** `https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/per-vu-iterations`
      
- **Number of Virtual Users**

  - **Constant VUs**
  
    An executor where the number of virtual users is determined, and each virtual user will continuously perform iterations until the specified duration is reached.
    
    **Reference:** `https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/constant-vus`
    
  - **Ramping VUs:**
  
    An executor that creates a specified number of virtual users at each stage, and will scale up or down following the next stage until all stages are completed.
    
    **Reference**: `https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/constant-vus`
    
- **Iteration Rate**

  - **Constant Arrival Rate**
  
    An executor that performs a constant number of iterations as specified. For example, setting 100 iterations per 1 second for 30 seconds means it will execute 100 iterations every second for 30 seconds.
    
    **Reference:** `https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/constant-arrival-rate`
    
  - **Ramping Arrival Rate:**
  
    An executor similar to constant arrival rate, except the number of iterations can scale up or down following the determined stages.
    
    **Reference:** `https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/ramping-arrival-rate`
     
---

## 🎯 Thresholds

By default, test results are always considered successful, whether there are errors or not. You can set thresholds to determine the limit for whether a test is successful or failed. If test results meet the specified thresholds, they are considered successful; if not, they are considered failed.

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

- **Reference:**

  `https://grafana.com/docs/k6/latest/using-k6/thresholds`  & `https://grafana.com/docs/k6/latest/using-k6/metrics`
  
---

## 📈 Output & Reporting

- **Summary Output**

  After k6 runs performance testing from your script, it produces a summary output of the test results. By default, the output is displayed in the console/terminal.

  `Save summary to JSON file`
    
  ```bash
  k6 run script.js --summary-export=summary.json
  ```
  
  **Reference:** `https://grafana.com/docs/k6/latest/using-k6/metrics/reference`

- **Summary Statistics**

  By default, statistics used for each metric are: `[avg, min, med, max, p(90), p(95)]`

  `Customize summary statistics`
  
  ```javascript
  export const options = {
    vus: 10,
    duration: '10s',
    summaryTrendStats: ['avg', 'min', 'med', 'max', 'p(90)', 'p(95)', 'p(99)']
  };
  ```
  
  **Reference:** `https://grafana.com/docs/k6/latest/using-k6/k6-options/reference/#summary-trend-stats`

- **Real-time Output**

  Summary output only creates reports after the performance test is complete. If you need real-time information, you can use real-time output from k6. By default, k6 can send real-time output to JSON or CSV files.

  `Output to CSV`
    
  ```bash
  k6 run --out csv=test_results.csv script.js
  ```
    
  **Reference:** `https://grafana.com/docs/k6/latest/results-output/real-time/csv`
  
  `Output to JSON`
    
  ```bash
  k6 run --out json=test_results.json script.js
  ```
  
  **Reference:** `https://grafana.com/docs/k6/latest/results-output/real-time/json`

- **Stream to Third-party Services**

  Besides files, real-time output can also be sent to third-party services. However, sending to third-party services cannot be done by default. You must rebuild the k6 application by adding the third-party library.
  
  **Reference:** `https://grafana.com/docs/k6/latest/results-output/real-time`

- **Web Dashboard**

  k6 has a web dashboard feature to view real-time output and summary output while k6 is running tests.
  
  ```bash
  export K6_WEB_DASHBOARD=true
  k6 run script.js
  ```
  
  **Reference:** `https://grafana.com/docs/k6/latest/results-output/web-dashboard`

---

## 🏛️ JavaScript Libraries

k6 provides JavaScript libraries that can be used to simplify script creation. It's recommended to read the documentation to understand the purpose of each JavaScript library provided

- **Example using JavaScript Libraries**
  
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

 - **Reference:** `https://jslib.k6.io`
