
# 🛠️ Advanced Monitoring & Load Testing for Serverless DynamoDB API

This guide extends your existing project (API Gateway + Lambda + DynamoDB) to include deep observability and performance benchmarking using CloudWatch, X-Ray, k6, and Grafana + InfluxDB.

---

## ✅ Core Monitoring & Observability

### 1. API Gateway Monitoring

**Enable Access Logs**
- In Stage Settings → Enable logging
- Sample log format:
```json
{
  "requestId": "$context.requestId",
  "resourcePath": "$context.resourcePath",
  "httpMethod": "$context.httpMethod",
  "status": "$context.status",
  "integrationLatency": "$context.integrationLatency",
  "requestTime": "$context.requestTime"
}
```

**CloudWatch Metrics**
- Metrics: `4XXError`, `5XXError`, `Latency`, `Count`
- Add alarms: latency > 2s, 5XX error spike

---

### 2. Lambda Observability

**Lambda Insights**
- Enable in Lambda settings for memory usage, cold start, init time

**Structured Logging (JSON)**
Use `aws-lambda-powertools` or similar.
```json
{
  "level": "INFO",
  "message": "PutItem successful",
  "operation": "create",
  "requestId": "abc123"
}
```

**Custom Metrics**
```python
cloudwatch.put_metric_data(
    Namespace='MyApp/DynamoOps',
    MetricData=[{
        'MetricName': 'PutItemLatency',
        'Value': elapsed_time,
        'Unit': 'Milliseconds'
    }]
)
```

---

### 3. DynamoDB Observability

**Contributor Insights**
- Identify hot partitions, skewed access

**CloudWatch Metrics**
- Monitor `ConsumedReadCapacityUnits`, `ThrottledRequests`

---

### 4. Distributed Tracing with X-Ray

**Enable on:**
- API Gateway
- Lambda
- DynamoDB SDK (using Powertools or native tracing)

Benefits:
- Full end-to-end request flow
- Visual bottlenecks
- Cold start visibility

---

## 🧪 Load Testing with k6

### Sample Script
```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
  vus: 50,
  duration: '30s',
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const payload = JSON.stringify({
    operation: "create",
    item: {
      id: `item-${__VU}-${__ITER}`,
      name: "Sample Item",
      timestamp: Date.now()
    }
  });

  const res = http.post('https://your-api-id.execute-api.region.amazonaws.com/dev/DynamoDBManager', payload, {
    headers: { 'Content-Type': 'application/json' },
  });

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500
  });

  sleep(1);
}
```

**Key Metrics**:
- `http_req_duration`, `http_req_failed`, `iterations`

---

## 📊 Grafana + InfluxDB for Visualization

### Step-by-Step Setup

#### 1. Install Components
Install via Docker Compose (recommended):
```yaml
version: '3'
services:
  influxdb:
    image: influxdb:1.8
    ports:
      - "8086:8086"
    volumes:
      - ./influxdb:/var/lib/influxdb

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    volumes:
      - ./grafana:/var/lib/grafana
```

Start the stack:
```bash
docker-compose up -d
```

#### 2. Configure InfluxDB as k6 Output
Install k6 output plugin if needed.

Run test:
```bash
K6_OUT=influxdb=http://localhost:8086/k6 k6 run test.js
```

#### 3. Set Up Grafana
- URL: http://localhost:3000
- Default credentials: `admin / admin`
- Add data source: InfluxDB
  - URL: `http://influxdb:8086`
  - Database: `k6`
- Import k6 dashboards from [Grafana.com](https://grafana.com/grafana/dashboards/2587-k6-load-testing-results/)

---

## 🚀 Summary Table

| Layer         | Tools / Features                                      |
|---------------|--------------------------------------------------------|
| API Gateway   | Access logs, 5XX alerts, latency metrics               |
| Lambda        | Insights, structured logs, X-Ray, custom metrics       |
| DynamoDB      | Contributor Insights, capacity metrics, alarms         |
| Load Testing  | k6: VU-based load, thresholds, scripting               |
| Visualization | Grafana + InfluxDB + k6 dashboard                      |
