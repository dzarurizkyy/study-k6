# Study k6 ⚡

This repository contains a comprehensive reference guide for k6 — covering installation, writing and running scripts, options, HTTP requests, validation, the test lifecycle, scenarios, metrics, thresholds, and reporting, worked through hands-on against a REST API.

## Installation 🔧

1. **Install k6**:

    ```bash
    # macOS (Homebrew)
    brew install k6
    ```

   > **Windows**: `choco install k6`

2. **Verify the Installation**:

   ```bash
   k6 --version
   ```

   > Reference: [grafana.com/docs/k6/latest/set-up/install-k6](https://grafana.com/docs/k6/latest/set-up/install-k6/)

## List of Material 📚

- ⚡ **[k6 Load Testing](001-k6-basics.md)**

  Script structure, options and stages, HTTP requests, checks, the test lifecycle, modular scripts, environment variables, scenarios and executors, custom metrics, thresholds, and output & reporting:

  ```javascript
  import http from "k6/http";
  import { check, sleep } from "k6";

  export const options = {
    vus: 10,
    duration: "30s",
  };

  export default function () {
    const response = http.get("http://localhost:3000/api/users");

    check(response, {
      "status is 200": (r) => r.status === 200,
      "response time < 500ms": (r) => r.timings.duration < 500,
    });

    sleep(1);
  }
  ```

  Run the test:

  ```bash
  k6 run --vus 50 --duration 1m script.js
  ```

## 📍 References

- [Udemy](https://www.udemy.com/course/belajar-k6/)

## 👨‍💻 Contributors

- [Dzaru Rizky Fathan Fortuna](https://www.linkedin.com/in/dzarurizky)
