# Study k6 ⚡

This repository contains examples of basic k6 load testing commands, script structure, and practical scenarios to practice and strengthen your performance testing skills with k6.

## Installation 🔧

1. **macOS** (using Homebrew):
   ```bash
   brew install k6
   ```

2. **Windows** (using Chocolatey):
   ```bash
   choco install k6
   ```

3. Verify installation:
   ```bash
   k6 --version
   ```

## List of Material 📚

* 📘 **k6 Basic**
  
  Contains examples of basic k6 load testing, including:
  
  ```javascript
  import http from 'k6/http';
  import { sleep, check } from 'k6';
  
  // Configure test options
  export const options = {
    vus: 10,           // 10 virtual users
    duration: '30s',   // Run for 30 seconds
  };
  
  // Main test function
  export default function() {
    // Make HTTP request
    const response = http.get('http://localhost:3000/api/users');
    
    // Validate response
    check(response, {
      'status is 200': (r) => r.status === 200,
      'response time < 500ms': (r) => r.timings.duration < 500,
    });
    
    // Wait 1 second before next iteration
    sleep(1);
  }
  ```
  
  Run the test:
  ```bash
  k6 run script.js
  ```
  
  Common commands:
  ```bash
  # Run with custom VUs and duration
  k6 run --vus 50 --duration 1m script.js
  
  # Export results to JSON
  k6 run --out json=results.json script.js
  
  # Enable web dashboard
  K6_WEB_DASHBOARD=true k6 run script.js
  ```

## 📍 References

* [Udemy](https://www.udemy.com/course/belajar-k6/)

## 👨‍💻 Contributors

* [Dzaru Rizky Fathan Fortuna](https://www.linkedin.com/in/dzarurizky)