# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# CHƯƠNG 12: ADVANCED TESTING CONCEPTS

---

## 12.1 Shift Left Testing

### 12.1.1 Khái niệm

**Shift Left Testing** là triết lý đưa hoạt động kiểm thử về phía **trái** trên timeline của dự án — nghĩa là thực hiện càng sớm càng tốt, thay vì chờ đến cuối vòng phát triển.

```
Mô hình truyền thống:
[Requirements] → [Design] → [Development] → [Testing] → [Release]
                                                 ↑
                                        Testing diễn ra ở đây

Shift Left:
[Requirements] → [Design] → [Development] → [Testing] → [Release]
     ↑               ↑            ↑
Testing bắt đầu từ đây — review, phân tích, viết test sớm
```

### 12.1.2 Thực hành Shift Left

**Tại giai đoạn Requirements:**
- Tester đọc và review tài liệu yêu cầu ngay khi có
- Đặt câu hỏi về edge case, ambiguity, missing scenarios
- Tham gia Three Amigos session
- Phát hiện mâu thuẫn trong yêu cầu trước khi code

**Tại giai đoạn Design:**
- Review architecture design — liệu thiết kế có dễ testable không?
- Đề xuất thêm logging, monitoring hook
- Thảo luận về test data strategy

**Tại giai đoạn Development:**
- Developer viết unit test (TDD)
- Tester viết test case song song với developer viết code
- Code review bao gồm review test case

**Lợi ích đo được:**

| Giai đoạn phát hiện | Chi phí sửa |
|---|---|
| Requirements | 1x |
| Design | 5x |
| Coding | 10x |
| Testing | 20x |
| Production | 100x |

---

## 12.2 Shift Right Testing

### 12.2.1 Khái niệm

**Shift Right Testing** kiểm thử **sau khi** phần mềm đã được deploy lên môi trường production (hoặc gần production). Đây không phải thay thế Shift Left mà là **bổ sung** — một số loại lỗi chỉ xuất hiện trong điều kiện thực tế với người dùng thật và dữ liệu thật.

### 12.2.2 Các kỹ thuật Shift Right

**Canary Release:**
Phát hành tính năng mới chỉ cho một tỷ lệ nhỏ người dùng (ví dụ 5%), theo dõi metrics, nếu ổn mới rollout cho 100%.

```
100% users → [Old Version]

Sau Canary Deploy:
 95% users → [Old Version]
  5% users → [New Version] ← monitor closely

Nếu metrics tốt → tăng dần 5% → 20% → 50% → 100%
Nếu metrics xấu → rollback ngay
```

**Feature Flags (Feature Toggles):**
Bật/tắt tính năng mà không cần deploy lại code:

```typescript
// Trong code
if (featureFlags.isEnabled("new-checkout-flow", userId)) {
    return <NewCheckoutFlow />;
} else {
    return <OldCheckoutFlow />;
}

// Config server (LaunchDarkly, Unleash...)
{
    "new-checkout-flow": {
        "enabled": true,
        "rolloutPercentage": 10,  // 10% users
        "whitelist": ["qa@company.com", "internal@company.com"]
    }
}
```

**A/B Testing:**
Chạy song song hai phiên bản để so sánh hiệu quả:
- Version A: nút "Mua ngay" màu xanh
- Version B: nút "Mua ngay" màu đỏ
- Đo conversion rate của mỗi version → chọn version tốt hơn

**Production Monitoring:**
```
Metrics cần theo dõi sau release:
- Error rate: tỷ lệ request trả lỗi 5xx
- Latency: p50, p95, p99 response time
- Throughput: requests/second
- Business metrics: conversion rate, cart abandonment
- User complaints: support tickets, app store reviews
```

---

## 12.3 Risk-based Testing

### 12.3.1 Khái niệm

**Risk-based Testing** là phương pháp ưu tiên test case và phân bổ effort kiểm thử dựa trên **mức độ rủi ro** của từng tính năng — tập trung nhiều hơn vào những vùng có risk cao.

**Risk = Likelihood (xác suất xảy ra) × Impact (mức độ tác động)**

### 12.3.2 Quy trình Risk-based Testing

**Bước 1: Xác định rủi ro**
```
Các nguồn rủi ro:
- Tính năng mới, chưa được test nhiều
- Module phức tạp, logic nghiệp vụ nặng
- Tích hợp với third-party service
- Code do developer mới viết
- Module thay đổi nhiều trong sprint này
- Tính năng ảnh hưởng trực tiếp đến doanh thu
```

**Bước 2: Đánh giá Risk Matrix**

| Tính năng | Likelihood (1-5) | Impact (1-5) | Risk Score | Priority |
|---|---|---|---|---|
| Checkout / Payment | 3 | 5 | 15 | Critical |
| Email notification | 2 | 3 | 6 | Medium |
| Product search | 4 | 3 | 12 | High |
| Admin dashboard | 2 | 2 | 4 | Low |
| Wishlist | 3 | 2 | 6 | Medium |

**Bước 3: Phân bổ effort**
```
Critical (score 12-25): 40% effort — test kỹ nhất, nhiều test case, automation
High (score 8-12):      30% effort — test đầy đủ
Medium (score 4-8):     20% effort — test happy path + critical negative
Low (score 1-4):        10% effort — smoke test cơ bản
```

**Bước 4: Áp dụng trong thực tế**

Ví dụ Sprint 5 có 3 tính năng:
- Checkout mới (Risk: Critical) → 15 test case, automation
- Filter sản phẩm (Risk: Medium) → 8 test case
- Cập nhật giao diện (Risk: Low) → 3 test case visual check

---

## 12.4 Contract Testing

### 12.4.1 Khái niệm và Vấn đề giải quyết

Trong kiến trúc microservices, nhiều service giao tiếp với nhau qua API. **Contract Testing** đảm bảo rằng các service phụ thuộc nhau thực sự "nói chuyện" được — không cần deploy và test toàn bộ hệ thống cùng lúc.

```
Không có Contract Testing:
Service A (Consumer) ──calls──> Service B (Provider)

Vấn đề: Service B thay đổi response format
→ Service A bị broken
→ Phát hiện muộn khi integration test hoặc production

Với Contract Testing:
Service A định nghĩa Contract: "Tôi cần B trả về {id, name, price}"
Service B verify: "Contract của A, tôi có đáp ứng không?"
→ Phát hiện ngay khi Service B thay đổi
```

### 12.4.2 Consumer-driven Contract Testing với Pact

**Pact** là framework Contract Testing phổ biến nhất.

**Consumer (Service A — Frontend/API caller) viết contract:**

```typescript
// pact/consumer.test.ts
import { PactV3, MatchersV3 } from "@pact-foundation/pact";
const { like, eachLike, string, integer } = MatchersV3;

const provider = new PactV3({
    consumer: "FrontendApp",
    provider: "ProductService",
    dir: "./pacts",
});

describe("ProductService Contract", () => {
    test("Lấy sản phẩm theo ID", () => {
        return provider
            .given("Sản phẩm P001 tồn tại")
            .uponReceiving("GET /api/products/P001")
            .withRequest({
                method: "GET",
                path: "/api/products/P001",
                headers: { Accept: "application/json" },
            })
            .willRespondWith({
                status: 200,
                headers: { "Content-Type": "application/json" },
                body: {
                    id: string("P001"),
                    name: string("Áo thun Basic"),
                    price: integer(200000),
                    stock: integer(10),
                    // Không cần khớp 100% — chỉ những field Consumer cần
                },
            })
            .executeTest(async (mockServer) => {
                const client = new ProductApiClient(mockServer.url);
                const product = await client.getProduct("P001");

                expect(product.id).toBe("P001");
                expect(product.price).toBe(200000);
                // Test chạy với Pact mock server
            });
    });

    test("Trả 404 khi sản phẩm không tồn tại", () => {
        return provider
            .given("Sản phẩm INVALID không tồn tại")
            .uponReceiving("GET /api/products/INVALID")
            .withRequest({ method: "GET", path: "/api/products/INVALID" })
            .willRespondWith({
                status: 404,
                body: { error: string("Product not found") },
            })
            .executeTest(async (mockServer) => {
                const client = new ProductApiClient(mockServer.url);
                await expect(client.getProduct("INVALID")).rejects.toThrow("404");
            });
    });
});
// Sau khi chạy: tạo file pacts/FrontendApp-ProductService.json
```

**Provider (Service B — ProductService) verify contract:**

```typescript
// pact/provider.test.ts
import { Verifier } from "@pact-foundation/pact";

describe("ProductService — Provider Verification", () => {
    test("Verify tất cả contracts từ consumers", () => {
        return new Verifier({
            provider: "ProductService",
            providerBaseUrl: "http://localhost:3001",  // ProductService đang chạy

            // Lấy contracts từ Pact Broker hoặc file local
            pactUrls: ["./pacts/FrontendApp-ProductService.json"],

            // State handlers — setup data cho từng "given" state
            stateHandlers: {
                "Sản phẩm P001 tồn tại": async () => {
                    await db.query(
                        "INSERT OR REPLACE INTO products (id, name, price, stock) VALUES ('P001', 'Áo thun Basic', 200000, 10)"
                    );
                },
                "Sản phẩm INVALID không tồn tại": async () => {
                    await db.query("DELETE FROM products WHERE id = 'INVALID'");
                },
            },

            publishVerificationResult: true,
            providerVersion: "1.2.3",
        }).verifyProvider();
    });
});
```

---

## 12.5 Mutation Testing

### 12.5.1 Khái niệm

**Mutation Testing** đánh giá **chất lượng của bộ test** bằng cách cố ý tạo ra các lỗi nhỏ (mutations) trong code và kiểm tra xem test suite có phát hiện ra không.

**Ý tưởng cốt lõi:** Nếu test suite tốt, nó phải fail khi code bị sửa sai. Nếu test suite vẫn pass dù code sai → test suite yếu.

```
Code gốc:                Mutant (code bị sửa):
if (price > 0)    →      if (price >= 0)    ← Mutation: > thành >=
  return valid;           return valid;
```

Nếu test của bạn có case kiểm tra `price = 0` và expect kết quả `invalid` → test sẽ FAIL với mutant → mutation bị "killed" ✅

Nếu test không có case đó → test vẫn PASS dù code sai → mutation "survived" ❌

**Mutation Score:**
```
Mutation Score = (Mutations Killed) / (Total Mutations) × 100%

Score > 80%: Test suite tốt
Score 60-80%: Cần cải thiện
Score < 60%: Test suite yếu
```

### 12.5.2 Các loại Mutation phổ biến

```
Arithmetic Operator Mutation:
  + → -    (a + b → a - b)
  * → /    (a * b → a / b)
  % → *    (a % b → a * b)

Relational Operator Mutation:
  > → >=   (price > 0 → price >= 0)
  == → !=  (status == "active" → status != "active")
  < → <=   (age < 18 → age <= 18)

Logical Connector Mutation:
  && → ||  (isLoggedIn && hasPermission → isLoggedIn || hasPermission)
  || → &&

Statement Deletion:
  Xóa một câu lệnh (return, throw, assignment)

Constant Mutation:
  0 → 1, true → false, "" → "x"
```

### 12.5.3 Stryker — Mutation Testing cho JavaScript

```bash
# Cài đặt
npm install --save-dev @stryker-mutator/core @stryker-mutator/jest-runner

# stryker.config.json
{
  "packageManager": "npm",
  "reporters": ["html", "clear-text", "progress"],
  "testRunner": "jest",
  "coverageAnalysis": "perTest",
  "mutate": ["src/**/*.ts", "!src/**/*.test.ts"]
}

# Chạy
npx stryker run
```

**Kết quả mẫu:**
```
Mutation score: 73.33%

Survived mutants:
- cart.service.ts line 45: Replaced >= with > (survived)
  Code: if (discount >= 0 && discount <= 100)
  Mutant: if (discount > 0 && discount <= 100)
  → Test không cover case discount = 0!

Action: Thêm test case: expect(applyDiscount(100, 0)).toBe(100)
```

---

## 12.6 Chaos Testing

### 12.6.1 Khái niệm và Triết lý

**Chaos Engineering** (hay Chaos Testing) là thực hành **cố ý tạo ra sự cố** trong hệ thống để khám phá điểm yếu trước khi chúng gây ra sự cố thực sự trong production.

> **Triết lý:** "Nếu hệ thống sẽ fail, thà ta gây ra lúc ta kiểm soát được, còn hơn để nó tự fail lúc ta không hay biết."

**Netflix và Chaos Monkey:**
Netflix nổi tiếng với việc chạy **Chaos Monkey** — tool tự động ngẫu nhiên tắt server trong production environment. Ý tưởng: nếu hệ thống không thể chịu đựng việc một server bị tắt, hãy sửa ngay thay vì chờ nó tự xảy ra đêm thứ Sáu.

### 12.6.2 Các loại Chaos Experiment

**Infrastructure Chaos:**
```bash
# Tắt một instance ngẫu nhiên
aws ec2 terminate-instances --instance-ids $(aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=staging" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)

# Mô phỏng high CPU
stress --cpu 8 --timeout 60s

# Mô phỏng full disk
fallocate -l 10G /tmp/stress-test
```

**Network Chaos:**
```bash
# Thêm latency 100ms vào network interface
tc qdisc add dev eth0 root netem delay 100ms 20ms

# Mô phỏng packet loss 10%
tc qdisc add dev eth0 root netem loss 10%

# Bandwidth throttling
tc qdisc add dev eth0 root tbf rate 1mbit burst 32kbit latency 400ms
```

**Application-level Chaos:**
```typescript
// Chaos middleware — ngẫu nhiên inject lỗi
class ChaosMiddleware {
    private failureRate = 0.05;  // 5% requests fail

    async use(req: Request, res: Response, next: NextFunction) {
        if (Math.random() < this.failureRate) {
            return res.status(500).json({ error: "Chaos injected error" });
        }
        next();
    }
}
```

### 12.6.3 Nguyên tắc Chaos Engineering

**Bước 1: Định nghĩa "Steady State"**
Trạng thái bình thường của hệ thống: error rate < 0.1%, p99 latency < 500ms, throughput > 1000 RPS.

**Bước 2: Đặt hypothesis**
"Nếu một database replica bị tắt, hệ thống vẫn hoạt động với latency tăng không quá 50%."

**Bước 3: Chạy experiment**
Tắt database replica, đo metrics.

**Bước 4: So sánh với Steady State**
Nếu metrics vẫn trong ngưỡng → hypothesis đúng → resilience tốt.
Nếu metrics vượt ngưỡng → phát hiện điểm yếu → fix.

**Bước 5: Tự động hóa và chạy liên tục**

---

## 12.7 Property-based Testing

### 12.7.1 Khái niệm

**Property-based Testing** (còn gọi là **Generative Testing**) thay vì kiểm thử với các ví dụ cụ thể (example-based), định nghĩa **tính chất (property)** mà hàm phải thỏa mãn với bất kỳ input hợp lệ nào — framework tự động generate hàng trăm input để tìm counter-example.

**Example-based Testing:**
```typescript
test("Sắp xếp mảng", () => {
    expect(sort([3, 1, 2])).toEqual([1, 2, 3]);
    expect(sort([5, 4])).toEqual([4, 5]);
    // Chỉ test 2 ví dụ cụ thể
});
```

**Property-based Testing:**
```typescript
test("Hàm sort phải thỏa mãn: output đã được sắp xếp VÀ cùng phần tử với input", () => {
    fc.assert(
        fc.property(fc.array(fc.integer()), (arr) => {
            const sorted = sort(arr);

            // Property 1: Output đã sorted
            for (let i = 0; i < sorted.length - 1; i++) {
                expect(sorted[i]).toBeLessThanOrEqual(sorted[i + 1]);
            }

            // Property 2: Cùng số lượng phần tử
            expect(sorted.length).toBe(arr.length);

            // Property 3: Cùng tổng (không mất phần tử)
            expect(sorted.reduce((a, b) => a + b, 0))
                .toBe(arr.reduce((a, b) => a + b, 0));
        })
    );
    // fast-check tự generate 100+ bộ input khác nhau
});
```

### 12.7.2 fast-check (JavaScript)

```typescript
import fc from "fast-check";

// Ví dụ: Kiểm thử hàm tính discount
function applyDiscount(price: number, discountPercent: number): number {
    return price * (1 - discountPercent / 100);
}

test("Property: Giá sau discount luôn <= giá gốc (discount >= 0)", () => {
    fc.assert(
        fc.property(
            fc.float({ min: 0, max: 1_000_000 }),    // price: 0 - 1M
            fc.float({ min: 0, max: 100 }),            // discount: 0% - 100%
            (price, discount) => {
                const result = applyDiscount(price, discount);
                return result <= price;
            }
        )
    );
});

test("Property: Discount 0% không thay đổi giá", () => {
    fc.assert(
        fc.property(
            fc.float({ min: 0, max: 1_000_000 }),
            (price) => {
                expect(applyDiscount(price, 0)).toBeCloseTo(price);
            }
        )
    );
});

test("Property: Discount 100% → giá = 0", () => {
    fc.assert(
        fc.property(
            fc.float({ min: 0, max: 1_000_000 }),
            (price) => {
                expect(applyDiscount(price, 100)).toBeCloseTo(0);
            }
        )
    );
});
```

**Khi fast-check tìm thấy lỗi, nó "shrinks" xuống trường hợp nhỏ nhất:**
```
Error found!
Property failed after 47 tests with counterexample:
  price = 0.1, discount = 99.99999999

Shrinking...
Minimal counterexample:
  price = 0.1, discount = 99.9
→ applyDiscount(0.1, 99.9) = 0.0001 (do floating point precision issue)
```

### 12.7.3 Hypothesis (Python)

```python
from hypothesis import given, strategies as st, assume
import pytest

def calculate_vat(price: float, vat_rate: float) -> float:
    return price * (1 + vat_rate / 100)

@given(
    price=st.floats(min_value=0, max_value=1_000_000, allow_nan=False),
    vat_rate=st.floats(min_value=0, max_value=100, allow_nan=False)
)
def test_vat_price_always_greater_or_equal(price, vat_rate):
    """Giá sau VAT phải >= giá gốc"""
    assume(price >= 0 and vat_rate >= 0)
    result = calculate_vat(price, vat_rate)
    assert result >= price

@given(price=st.floats(min_value=1, max_value=1_000_000, allow_nan=False))
def test_vat_0_percent_no_change(price):
    """VAT 0% không làm thay đổi giá"""
    assume(price > 0)
    assert calculate_vat(price, 0) == pytest.approx(price)
```

---

## 12.8 Tổng hợp: Khi nào áp dụng từng kỹ thuật

| Kỹ thuật | Áp dụng khi | Người thực hiện |
|---|---|---|
| **Shift Left** | Mọi dự án — bắt đầu từ requirements | Tester + toàn team |
| **Shift Right** | Hệ thống production lớn, microservices | DevOps + Senior QA |
| **Risk-based** | Ít thời gian, cần prioritize | QA Lead |
| **Contract Testing** | Microservices, nhiều team | Dev + QA |
| **Mutation Testing** | Muốn đánh giá chất lượng unit test | Developer |
| **Chaos Testing** | Hệ thống distributed, high-availability | DevOps + SRE |
| **Property-based** | Business logic phức tạp, mathematical | Developer + QA |

---
