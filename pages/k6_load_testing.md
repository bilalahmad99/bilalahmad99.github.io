---
layout: default
---

## Load Testing with k6

A practical guide to performance testing with k6, covering test types, scenarios, thresholds as SLOs, and integrating with CI and Grafana.

### What is k6?

k6 is a load testing tool built for developers and DevOps teams. It uses JavaScript for test scripts and focuses on developer experience with good CI/CD integration.

### Installation & Setup

```bash
# Install k6
brew install k6  # macOS
# or
sudo apt-get install k6  # Ubuntu
# or download from https://k6.io/docs/getting-started/installation/

# Verify installation
k6 version
```

### Basic Test Structure

```javascript
// basic-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 100 }, // Ramp up
    { duration: '5m', target: 100 }, // Stay at 100 users
    { duration: '2m', target: 0 },   // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% of requests under 500ms
    http_req_failed: ['rate<0.1'],     // Error rate under 10%
  },
};

export default function () {
  let response = http.get('https://httpbin.org/get');
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  sleep(1);
}
```

### Test Types & Scenarios

#### 1. Smoke Test
Quick validation that the system works under minimal load.

```javascript
// smoke-test.js
import http from 'k6/http';
import { check } from 'k6';

export let options = {
  vus: 1, // 1 virtual user
  duration: '1m',
};

export default function () {
  let response = http.get('https://api.example.com/health');
  
  check(response, {
    'health check passes': (r) => r.status === 200,
    'response time < 100ms': (r) => r.timings.duration < 100,
  });
}
```

#### 2. Load Test
Test normal expected load conditions.

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 10 },
    { duration: '5m', target: 10 },
    { duration: '2m', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<2000'],
    http_req_failed: ['rate<0.05'],
  },
};

export default function () {
  // Test multiple endpoints
  let endpoints = [
    'https://api.example.com/users',
    'https://api.example.com/products',
    'https://api.example.com/orders',
  ];
  
  for (let endpoint of endpoints) {
    let response = http.get(endpoint);
    
    check(response, {
      [`${endpoint} status is 200`]: (r) => r.status === 200,
      [`${endpoint} response time < 2s`]: (r) => r.timings.duration < 2000,
    });
    
    sleep(0.5);
  }
}
```

#### 3. Stress Test
Find the breaking point of your system.

```javascript
// stress-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 10 },
    { duration: '5m', target: 20 },
    { duration: '5m', target: 50 },
    { duration: '5m', target: 100 },
    { duration: '5m', target: 200 },
    { duration: '10m', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<5000'],
    http_req_failed: ['rate<0.1'],
  },
};

export default function () {
  let response = http.get('https://api.example.com/heavy-endpoint');
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 5s': (r) => r.timings.duration < 5000,
  });
  
  sleep(1);
}
```

#### 4. Spike Test
Test system behavior under sudden load spikes.

```javascript
// spike-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '1m', target: 10 },
    { duration: '1m', target: 100 }, // Spike
    { duration: '1m', target: 10 },
    { duration: '1m', target: 200 }, // Bigger spike
    { duration: '1m', target: 10 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<3000'],
    http_req_failed: ['rate<0.2'],
  },
};

export default function () {
  let response = http.get('https://api.example.com/spike-endpoint');
  
  check(response, {
    'status is 200': (r) => r.status === 200,
  });
  
  sleep(0.1); // Faster requests during spike
}
```

### Advanced Scenarios

#### Authentication & Sessions

```javascript
// auth-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  vus: 10,
  duration: '5m',
};

export default function () {
  // Login
  let loginPayload = JSON.stringify({
    username: 'testuser',
    password: 'testpass',
  });
  
  let loginResponse = http.post('https://api.example.com/login', loginPayload, {
    headers: { 'Content-Type': 'application/json' },
  });
  
  check(loginResponse, {
    'login successful': (r) => r.status === 200,
  });
  
  let token = loginResponse.json('token');
  
  // Use authenticated requests
  let authHeaders = {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json',
  };
  
  let protectedResponse = http.get('https://api.example.com/protected', {
    headers: authHeaders,
  });
  
  check(protectedResponse, {
    'protected endpoint accessible': (r) => r.status === 200,
  });
  
  sleep(1);
}
```

#### Data-Driven Testing

```javascript
// data-driven-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { SharedArray } from 'k6/data';

const testData = new SharedArray('test data', function () {
  return JSON.parse(open('./test-data.json'));
});

export let options = {
  vus: 5,
  duration: '5m',
};

export default function () {
  let data = testData[Math.floor(Math.random() * testData.length)];
  
  let response = http.post('https://api.example.com/users', JSON.stringify(data), {
    headers: { 'Content-Type': 'application/json' },
  });
  
  check(response, {
    'user created successfully': (r) => r.status === 201,
    'response time < 1s': (r) => r.timings.duration < 1000,
  });
  
  sleep(1);
}
```

#### Custom Metrics

```javascript
// custom-metrics.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Trend, Counter } from 'k6/metrics';

let customResponseTime = new Trend('custom_response_time');
let customErrors = new Counter('custom_errors');

export let options = {
  vus: 10,
  duration: '5m',
};

export default function () {
  let response = http.get('https://api.example.com/metrics-endpoint');
  
  customResponseTime.add(response.timings.duration);
  
  if (response.status !== 200) {
    customErrors.add(1);
  }
  
  check(response, {
    'status is 200': (r) => r.status === 200,
  });
  
  sleep(1);
}
```

### Thresholds as SLOs

Define Service Level Objectives directly in your tests.

```javascript
// slo-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  vus: 50,
  duration: '10m',
  thresholds: {
    // Availability SLO: 99.9% success rate
    http_req_failed: ['rate<0.001'],
    
    // Latency SLO: 95% of requests under 500ms
    http_req_duration: ['p(95)<500'],
    
    // Custom SLO: 99% of requests under 1s
    http_req_duration: ['p(99)<1000'],
    
    // Error budget: Max 0.1% error rate
    http_req_failed: ['rate<0.001'],
  },
};

export default function () {
  let response = http.get('https://api.example.com/slo-endpoint');
  
  check(response, {
    'SLO: status is 200': (r) => r.status === 200,
    'SLO: response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  sleep(1);
}
```

### CI/CD Integration

#### GitHub Actions Example

```yaml
# .github/workflows/load-test.yml
name: Load Testing

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  load-test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Install k6
      run: |
        sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
        echo "deb https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
        sudo apt-get update
        sudo apt-get install k6
    
    - name: Run smoke test
      run: k6 run --out json=smoke-results.json tests/smoke-test.js
    
    - name: Run load test
      if: github.ref == 'refs/heads/main'
      run: k6 run --out json=load-results.json tests/load-test.js
    
    - name: Upload results
      uses: actions/upload-artifact@v3
      with:
        name: k6-results
        path: |
          smoke-results.json
          load-results.json
```

#### Jenkins Pipeline Example

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    stages {
        stage('Load Test') {
            steps {
                sh 'k6 run tests/smoke-test.js'
            }
        }
        
        stage('Performance Test') {
            when {
                branch 'main'
            }
            steps {
                sh 'k6 run tests/load-test.js --out json=results.json'
                archiveArtifacts artifacts: 'results.json'
            }
        }
    }
    
    post {
        always {
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'results',
                reportFiles: 'index.html',
                reportName: 'K6 Report'
            ])
        }
    }
}
```

### Grafana Integration

#### InfluxDB Output

```javascript
// grafana-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  vus: 10,
  duration: '5m',
  ext: {
    influxdb: {
      url: 'http://influxdb:8086/k6',
    },
  },
};

export default function () {
  let response = http.get('https://api.example.com/grafana-endpoint');
  
  check(response, {
    'status is 200': (r) => r.status === 200,
  });
  
  sleep(1);
}
```

Run with InfluxDB output:
```bash
k6 run --out influxdb=http://influxdb:8086/k6 grafana-test.js
```

### Best Practices

#### Test Organization
```
tests/
├── smoke/
│   └── smoke-test.js
├── load/
│   └── load-test.js
├── stress/
│   └── stress-test.js
├── data/
│   └── test-data.json
└── utils/
    └── auth.js
```

#### Environment Configuration
```javascript
// config.js
export const config = {
  baseUrl: __ENV.BASE_URL || 'https://api.example.com',
  vus: __ENV.VUS || 10,
  duration: __ENV.DURATION || '5m',
  token: __ENV.API_TOKEN || '',
};

// usage in test
import { config } from './config.js';

export let options = {
  vus: config.vus,
  duration: config.duration,
};

export default function () {
  let response = http.get(`${config.baseUrl}/endpoint`);
  // ...
}
```

#### Running Tests
```bash
# Smoke test
k6 run tests/smoke/smoke-test.js

# Load test with custom VUs
k6 run --vus 50 tests/load/load-test.js

# Test with environment variables
BASE_URL=https://staging.example.com k6 run tests/load/load-test.js

# Test with InfluxDB output
k6 run --out influxdb=http://influxdb:8086/k6 tests/load/load-test.js

# Test with JSON output
k6 run --out json=results.json tests/load/load-test.js
```

### Common Patterns

#### API Testing Suite
```javascript
// api-suite.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 10 },
    { duration: '5m', target: 10 },
    { duration: '2m', target: 0 },
  ],
};

export default function () {
  // GET request
  let getResponse = http.get('https://api.example.com/users');
  check(getResponse, { 'GET status is 200': (r) => r.status === 200 });
  
  // POST request
  let postData = JSON.stringify({ name: 'Test User', email: 'test@example.com' });
  let postResponse = http.post('https://api.example.com/users', postData, {
    headers: { 'Content-Type': 'application/json' },
  });
  check(postResponse, { 'POST status is 201': (r) => r.status === 201 });
  
  // PUT request
  let putData = JSON.stringify({ name: 'Updated User' });
  let putResponse = http.put('https://api.example.com/users/1', putData, {
    headers: { 'Content-Type': 'application/json' },
  });
  check(putResponse, { 'PUT status is 200': (r) => r.status === 200 });
  
  sleep(1);
}
```

### Monitoring & Alerting

Set up alerts based on k6 metrics in your monitoring system:

- **Error Rate**: Alert when error rate exceeds 1%
- **Response Time**: Alert when 95th percentile exceeds 2 seconds
- **Throughput**: Alert when requests per second drops below threshold
- **Availability**: Alert when success rate drops below 99.9%

[back](../)
