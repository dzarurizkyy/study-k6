# K6 Load Testing - Hands-on Practice

A practical guide to API load testing using K6 with ramping VUs, custom metrics, and thresholds.

## 📋 Prerequisites

- K6 installed
- Node.js and npm (for project initialization)
- Basic understanding of JavaScript

## 🚀 Getting Started

- **Project Setup**

  Create a new folder and initialize the project:
  
  ```bash
  mkdir k6-handson && cd k6-handson
  npm init -y
  ```

- **Create Helper Script**

  Create a file named `request_helper.js` to separate HTTP logic:
  
  ```javascript
  import http from 'k6/http';
  
  export function createPost(payload) {
      const url = 'https://jsonplaceholder.typicode.com/posts';
      const params = {
          headers: {
              'Content-Type': 'application/json; charset=UTF-8',
          },
      };
      return http.post(url, JSON.stringify(payload), params);
  }
  ```

- **Create Main Test Script**

  Create a file named `load_test.js` with the following content:
  
  ```javascript
  import { check, sleep } from 'k6';
  import { Trend, Counter } from 'k6/metrics';
  import { createPost } from './request_helper.js';
  
  // 1. Custom Metrics
  const postDurationTrend = new Trend('custom_post_duration');
  const successCounter = new Counter('custom_success_count');
  
  // 2. Options & Scenarios
  export const options = {
      stages: [
          { duration: '30s', target: 5 },  // Ramp-up: 0 to 5 users in 30 seconds
          { duration: '1m', target: 5 },   // Stay: Maintain 5 users for 1 minute
          { duration: '10s', target: 0 },  // Ramp-down: Scale down to 0
      ],
      thresholds: {
          http_req_failed: ['rate<0.01'],        // Failure rate must be below 1%
          http_req_duration: ['p(95)<500'],      // 95% of requests must be under 500ms
          custom_success_count: ['count>50'],    // Must have at least 50 successes
      },
  };
  
  // 3. Setup Phase (Runs once at the beginning)
  export function setup() {
      console.log('--- Starting Test Execution ---');
      return { testType: 'Stress Test Small' };
  }
  
  // 4. Default Function (Main Load)
  export default function (data) {
      const payload = {
          title: 'Learning k6',
          body: 'k6 makes developer testing much easier',
          userId: 1,
      };
  
      const res = createPost(payload);
  
      // Response Validation
      const checkRes = check(res, {
          'status is 201': (r) => r.status === 201,
          'has id': (r) => r.json().hasOwnProperty('id'),
      });
  
      // Update Custom Metrics
      if (checkRes) {
          successCounter.add(1);
      }
      postDurationTrend.add(res.timings.duration);
  
      // Simulate user think time
      sleep(1);
  }
  
  // 5. Teardown Phase (Runs once at the end)
  export function teardown(data) {
      console.log(`--- ${data.testType} Completed ---`);
  }
  ```

## 🎯 Running the Tests

- **Option A: Standard Execution**

  Run the test with standard terminal output:
  
  ```bash
  k6 run load_test.js
  ```

- **Option B: Web Dashboard (Visual Mode)**

  Run the test with a real-time web dashboard:
  
  ```bash
  K6_WEB_DASHBOARD=true k6 run load_test.js
  ```
  
  After running this command, open your browser and navigate to `http://localhost:5665` to view real-time graphs and metrics.

## 📊 Understanding the Test Configuration

  - **Test Stages**
  
    - **Ramp-up** (30s): Gradually increase from 0 to 5 virtual users
    - **Steady State** (1m): Maintain 5 virtual users
    - **Ramp-down** (10s): Gradually decrease to 0 virtual users
  
  - **Thresholds**
  
    - **http_req_failed**: Failure rate must be below 1%
    - **http_req_duration**: 95th percentile must be under 500ms
    - **custom_success_count**: Must have at least 50 successful requests
  
  - **Custom Metrics**
  
    - **custom_post_duration**: Tracks the duration of POST requests
    - **custom_success_count**: Counts successful requests

## 📈 Interpreting Results

After the test completes, K6 will display:

- HTTP request metrics (duration, rate, failures)
- Custom metrics values
- Threshold pass/fail status
- Virtual user statistics

`Example Output:`

```
✓ status is 201
✓ has id
✓ http_req_failed............: 0.00%
✓ http_req_duration...........: avg=245ms p(95)=320ms
✓ custom_success_count........: 150
```
