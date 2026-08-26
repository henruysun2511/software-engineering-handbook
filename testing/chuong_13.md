# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# CHƯƠNG 13: PERFORMANCE TESTING

---

## 13.1 Khái niệm và Phân loại

**Performance Testing** là quá trình đánh giá hành vi của hệ thống dưới các điều kiện tải khác nhau — đo lường tốc độ, khả năng xử lý, và tính ổn định.

### 13.1.1 Phân loại Performance Testing

**Load Testing — Kiểm thử tải bình thường:**
Kiểm tra hệ thống dưới tải người dùng **dự kiến**. Mục tiêu: xác nhận hệ thống hoạt động đúng SLA ở mức tải bình thường.

```
Ví dụ: Hệ thống thương mại điện tử vào ngày thường
→ 500 concurrent users
→ 100 requests/second
→ Mục tiêu: p95 response time < 500ms
```

**Stress Testing — Kiểm thử giới hạn:**
Tăng tải vượt quá mức bình thường để tìm **điểm gãy** (breaking point). Mục tiêu: biết giới hạn của hệ thống và cách nó fail (gracefully hay crash).

```
→ Tăng từ 500 đến 2000, 5000 concurrent users
→ Tìm điểm hệ thống bắt đầu suy giảm
→ Kiểm tra recovery: khi tải giảm, hệ thống có phục hồi không?
```

**Spike Testing — Kiểm thử đột biến:**
Mô phỏng tải tăng đột ngột trong thời gian ngắn, sau đó giảm ngay.

```
Ví dụ: Flash sale 0:00 ngày 11/11
0:00:00 → 100 users
0:00:05 → 5000 users (spike)
0:01:00 → 200 users (trở lại bình thường)
```

**Endurance Testing (Soak Testing) — Kiểm thử bền bỉ:**
Chạy hệ thống dưới tải bình thường trong **thời gian dài** (8-24 giờ) để phát hiện vấn đề tích lũy theo thời gian như memory leak, connection pool exhaustion.

**Scalability Testing — Kiểm thử khả năng mở rộng:**
Kiểm tra hệ thống scale như thế nào khi thêm resource (server, RAM, CPU).

---

### 13.1.2 Các chỉ số Performance quan trọng

**Response Time:**
```
p50 (Median): 50% requests hoàn thành trong thời gian này
p95: 95% requests hoàn thành trong thời gian này
p99: 99% requests hoàn thành trong thời gian này

Ví dụ:
p50 = 120ms → Đa số user thấy response nhanh
p95 = 450ms → 5% user thấy hơi chậm
p99 = 2000ms → 1% user thấy rất chậm

→ SLA thường dựa trên p95 hoặc p99, không phải mean/average
(Average bị kéo bởi outliers, không phản ánh trải nghiệm đa số)
```

**Throughput:**
```
Requests per second (RPS) / Transactions per second (TPS)
→ Hệ thống xử lý được bao nhiêu request trong 1 giây?

Target: API search phải đạt 500 RPS trước khi latency tăng
```

**Error Rate:**
```
Error Rate = (Số request lỗi / Tổng số request) × 100%
SLA tiêu biểu: Error rate < 0.1%
```

**Concurrent Users:**
```
Số người dùng đang tương tác với hệ thống cùng lúc
≠ Số người đang online (nhiều người online nhưng không request)
```

---

## 13.2 k6 — Modern Performance Testing

### 13.2.1 Tại sao chọn k6?

- Script bằng JavaScript — developer friendly
- Tích hợp CI/CD tốt
- Output đẹp, metrics chi tiết
- Hỗ trợ threshold (fail test nếu không đạt SLA)
- Cloud option (k6 Cloud) cho phân tán

### 13.2.2 Cài đặt

```bash
# macOS
brew install k6

# Ubuntu
sudo apt-get install k6

# Docker
docker run grafana/k6 run /path/to/script.js

# Windows (Chocolatey)
choco install k6
```

### 13.2.3 Script k6 cơ bản

```javascript
// load-test.js
import http from "k6/http";
import { check, sleep } from "k6";
import { Rate, Trend } from "k6/metrics";

// Custom metrics
const errorRate = new Rate("error_rate");
const loginDuration = new Trend("login_duration");

// Cấu hình test
export const options = {
    // Scenarios: cách tăng tải
    stages: [
        { duration: "1m", target: 100 },   // Ramp up: 0 → 100 users trong 1 phút
        { duration: "3m", target: 100 },   // Steady state: 100 users trong 3 phút
        { duration: "1m", target: 200 },   // Ramp up: 100 → 200 users
        { duration: "3m", target: 200 },   // Steady state: 200 users
        { duration: "1m", target: 0 },     // Ramp down: về 0
    ],

    // Thresholds: điều kiện PASS/FAIL
    thresholds: {
        http_req_duration: ["p(95)<500", "p(99)<1000"],  // 95% < 500ms, 99% < 1s
        http_req_failed: ["rate<0.01"],                   // Error rate < 1%
        error_rate: ["rate<0.005"],                       // Custom: < 0.5%
    },
};

// Test data
const BASE_URL = "https://api-staging.example.com";
const testUsers = [
    { email: "test1@example.com", password: "Test@123" },
    { email: "test2@example.com", password: "Test@123" },
    { email: "test3@example.com", password: "Test@123" },
];

// Setup: chạy 1 lần trước tất cả VU
export function setup() {
    console.log("Starting performance test...");
}

// Main function: chạy cho mỗi Virtual User
export default function () {
    // Chọn user ngẫu nhiên
    const user = testUsers[Math.floor(Math.random() * testUsers.length)];

    // Scenario 1: Login
    const loginStart = Date.now();
    const loginRes = http.post(
        `${BASE_URL}/api/auth/login`,
        JSON.stringify({ email: user.email, password: user.password }),
        {
            headers: { "Content-Type": "application/json" },
            tags: { name: "login" },  // để group metrics
        }
    );
    loginDuration.add(Date.now() - loginStart);

    const loginOk = check(loginRes, {
        "login: status 200": (r) => r.status === 200,
        "login: có token": (r) => {
            try {
                return JSON.parse(r.body).data.token !== undefined;
            } catch {
                return false;
            }
        },
        "login: response time < 1s": (r) => r.timings.duration < 1000,
    });

    errorRate.add(!loginOk);

    if (!loginOk) {
        console.error(`Login failed: ${loginRes.status} - ${loginRes.body}`);
        sleep(1);
        return;
    }

    const token = JSON.parse(loginRes.body).data.token;
    const headers = {
        Authorization: `Bearer ${token}`,
        "Content-Type": "application/json",
    };

    sleep(1);  // Think time giữa các action

    // Scenario 2: Xem danh sách sản phẩm
    const productsRes = http.get(`${BASE_URL}/api/products?page=1&limit=20`, {
        headers,
        tags: { name: "get_products" },
    });

    check(productsRes, {
        "products: status 200": (r) => r.status === 200,
        "products: có data": (r) => {
            try {
                return JSON.parse(r.body).data.length > 0;
            } catch {
                return false;
            }
        },
        "products: response time < 300ms": (r) => r.timings.duration < 300,
    });

    sleep(2);

    // Scenario 3: Tìm kiếm sản phẩm
    const searchRes = http.get(`${BASE_URL}/api/products?search=áo+thun&limit=10`, {
        headers,
        tags: { name: "search_products" },
    });

    check(searchRes, {
        "search: status 200": (r) => r.status === 200,
        "search: response time < 500ms": (r) => r.timings.duration < 500,
    });

    sleep(1);

    // Scenario 4: Thêm vào giỏ hàng
    const cartRes = http.post(
        `${BASE_URL}/api/cart/items`,
        JSON.stringify({ productId: "P001", quantity: 1 }),
        { headers, tags: { name: "add_to_cart" } }
    );

    check(cartRes, {
        "cart: status 200": (r) => r.status === 200 || r.status === 201,
        "cart: response time < 400ms": (r) => r.timings.duration < 400,
    });

    sleep(3);  // User đang xem giỏ hàng
}

// Teardown: chạy 1 lần sau tất cả VU
export function teardown(data) {
    console.log("Performance test completed.");
}
```

**Chạy k6:**
```bash
# Chạy và xem kết quả ngay
k6 run load-test.js

# Chạy với output JSON
k6 run --out json=results.json load-test.js

# Chạy với InfluxDB + Grafana (real-time dashboard)
k6 run --out influxdb=http://localhost:8086/k6 load-test.js

# Kết quả mẫu:
# ✓ http_req_duration............: avg=234ms p(95)=478ms p(99)=892ms
# ✓ http_req_failed...............: 0.23% ✓ (< 1%)
# ✓ error_rate....................: 0.15% ✓ (< 0.5%)
# ✗ http_req_duration............: p(95)=534ms > 500ms ← THRESHOLD FAILED
```

---

### 13.2.4 Scenarios nâng cao

```javascript
// Nhiều scenarios chạy đồng thời — mô phỏng hành vi thực tế
export const options = {
    scenarios: {
        // 70% users chỉ browse
        browsers: {
            executor: "ramping-vus",
            startVUs: 0,
            stages: [
                { duration: "2m", target: 70 },
                { duration: "5m", target: 70 },
                { duration: "1m", target: 0 },
            ],
            exec: "browsing",
        },
        // 20% users search và add to cart
        shoppers: {
            executor: "ramping-vus",
            startVUs: 0,
            stages: [
                { duration: "2m", target: 20 },
                { duration: "5m", target: 20 },
                { duration: "1m", target: 0 },
            ],
            exec: "shopping",
        },
        // 10% users checkout
        buyers: {
            executor: "ramping-vus",
            startVUs: 0,
            stages: [
                { duration: "2m", target: 10 },
                { duration: "5m", target: 10 },
                { duration: "1m", target: 0 },
            ],
            exec: "checkout",
        },
    },
};

export function browsing() { /* ... */ }
export function shopping() { /* ... */ }
export function checkout() { /* ... */ }
```

---

## 13.3 Apache JMeter

### 13.3.1 Tổng quan

**Apache JMeter** là công cụ performance testing mã nguồn mở, phổ biến trong enterprise vì GUI dễ dùng và không cần viết code.

**Khi nào dùng JMeter thay k6:**
- Team không quen JavaScript
- Cần GUI để thiết kế test plan trực quan
- Dự án enterprise Java đã có JMeter infrastructure
- Cần record HTTP traffic (JMeter có proxy recorder)

### 13.3.2 Cấu trúc JMeter Test Plan

```
Test Plan
└── Thread Group (nhóm user)
    ├── HTTP Request Defaults (URL, port, encoding)
    ├── HTTP Cookie Manager
    ├── HTTP Header Manager
    ├── Login Request (HTTP Sampler)
    │   ├── JSON Extractor (lấy token từ response)
    │   └── Response Assertion (verify status = 200)
    ├── Think Time (Constant Timer: 2000ms)
    ├── Get Products Request
    │   └── Response Assertion
    ├── Add to Cart Request
    │   └── Response Assertion
    └── Listeners
        ├── Summary Report
        ├── View Results in Table
        └── Aggregate Report
```

### 13.3.3 Chạy JMeter không GUI (CLI)

```bash
# Chạy test plan
jmeter -n -t test-plan.jmx -l results.jtl -e -o report/

# Tham số:
# -n: non-GUI mode
# -t: test plan file
# -l: log file
# -e: generate HTML report
# -o: output folder cho report

# Override variables từ command line
jmeter -n -t test-plan.jmx \
  -Jthreads=200 \
  -Jrampup=60 \
  -Jduration=300 \
  -l results.jtl

# Tích hợp CI/CD (GitHub Actions)
- name: Run JMeter Performance Test
  run: |
    jmeter -n -t tests/performance/api-load-test.jmx \
      -l results/jmeter-results.jtl \
      -e -o results/jmeter-report
```

---

## 13.4 Phân tích kết quả Performance Test

### 13.4.1 Identify Bottleneck

```
Kịch bản: Test thất bại — p95 latency = 2500ms (target: < 500ms)

Bước 1: Xác định API nào chậm nhất
→ k6 tags: /api/search response time trung bình 1800ms
→ /api/products/detail: 200ms (OK)
→ /api/cart: 150ms (OK)

Bước 2: Profile /api/search
→ Mở APM (Datadog, New Relic): query nào chậm?
→ Phát hiện: SELECT query trong search không có index

Bước 3: Fix
→ Thêm index: CREATE INDEX idx_products_name ON products(name);
→ Thêm full-text search index

Bước 4: Re-test
→ /api/search: 120ms (cải thiện 15 lần)
→ p95 overall: 280ms ✅
```

### 13.4.2 Báo cáo Performance Test

```markdown
# PERFORMANCE TEST REPORT — v2.3.0

## Thông tin test
- Ngày: 15/01/2025
- Môi trường: Staging (4 CPU, 8GB RAM, 1 DB instance)
- Tool: k6 v0.47
- Duration: 30 phút

## Cấu hình tải
- Max concurrent users: 200
- Ramp up: 5 phút
- Steady state: 20 phút
- Ramp down: 5 phút

## Kết quả tổng quan

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| p50 response time | < 200ms | 145ms | ✅ |
| p95 response time | < 500ms | 423ms | ✅ |
| p99 response time | < 1000ms | 867ms | ✅ |
| Error rate | < 0.1% | 0.03% | ✅ |
| Throughput | > 300 RPS | 412 RPS | ✅ |

## Kết quả theo endpoint

| Endpoint | p50 | p95 | Error Rate |
|----------|-----|-----|------------|
| POST /auth/login | 120ms | 280ms | 0% |
| GET /products | 85ms | 195ms | 0% |
| GET /products/search | 145ms | 423ms | 0% |
| POST /cart/items | 98ms | 210ms | 0.02% |

## Kết luận
Hệ thống đạt SLA với 200 concurrent users. Bottleneck nhỏ ở search endpoint nhưng vẫn trong ngưỡng cho phép.

## Khuyến nghị
- Monitor search endpoint khi user tăng lên 300+
- Xem xét cache kết quả tìm kiếm phổ biến (Redis)
- Scale test lên 500 users trước Black Friday
```
