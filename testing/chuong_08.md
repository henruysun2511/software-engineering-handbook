# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# CHƯƠNG 8: AUTOMATION TESTING — NỀN TẢNG

---

## 8.1 Tư duy Automation Testing

### 8.1.1 Tại sao cần Automation?

Manual testing có giới hạn cốt lõi: **con người không thể chạy 500 test case trong 10 phút**. Khi dự án phát triển, số lượng test case tăng theo cấp số nhân — regression testing một mình tay sẽ mất hàng ngày, thậm chí hàng tuần.

**So sánh Manual vs Automation:**

| Tiêu chí | Manual Testing | Automation Testing |
|---|---|---|
| Tốc độ | Chậm | Nhanh (hàng trăm lần) |
| Chi phí ban đầu | Thấp | Cao (thời gian viết script) |
| Chi phí dài hạn | Cao (nhân sự liên tục) | Thấp hơn theo thời gian |
| Độ chính xác | Có thể nhầm lỗi | Nhất quán |
| Khả năng lặp lại | Tốn sức | Không giới hạn |
| Exploratory Testing | Tốt | Không làm được |
| Phát hiện UI issues | Tốt | Cơ bản |
| Regression Testing | Tốn thời gian | Lý tưởng |
| ROI | Ngay lập tức | Sau 3-6 tháng |

---

### 8.1.2 Khi nào NÊN automation?

```
✅ Nên automation khi:
- Test case lặp lại nhiều lần (regression, smoke test)
- Dữ liệu lớn cần kiểm thử (data-driven: 1000 tổ hợp đầu vào)
- Kiểm thử trên nhiều browser/device cùng lúc (cross-browser)
- Kiểm thử hiệu suất với hàng trăm user đồng thời
- CI/CD: chạy sau mỗi commit trong vài phút
- Critical path của ứng dụng (login, checkout, payment)
- Test case ổn định, ít thay đổi
```

### 8.1.3 Khi nào KHÔNG NÊN automation?

```
❌ Không nên automation khi:
- Test case chạy 1-2 lần rồi bỏ (setup chi phí > lợi ích)
- UI thay đổi liên tục (automation bị hỏng liên tục)
- Exploratory testing (cần óc sáng tạo của con người)
- Usability testing (cần cảm nhận chủ quan)
- Tính năng đang phát triển dở dang (moving target)
- Team chưa có kỹ năng (automation tệ còn tệ hơn không có)
```

---

### 8.1.4 Test Automation Pyramid

```
            /\
           /  \
          / E2E\              5-10%
         /──────\             ← Ít, chậm, đắt, dễ flaky
        /        \
       /Integration\          20-30%
      /────────────\          ← Vừa phải
     /              \
    /   Unit Tests   \        60-70%
   /──────────────────\       ← Nhiều, nhanh, rẻ, đáng tin
```

**Nguyên tắc áp dụng:**
- Đầu tư nhiều nhất vào Unit Tests — nhanh, rẻ, feedback ngay
- Integration Tests kiểm thử sự kết hợp các component
- E2E Tests chỉ cho critical user journeys — tránh viết quá nhiều

**Anti-pattern — Ice Cream Cone (cần tránh):**
```
     /\
    /  \
   / E2E\              Nhiều E2E manual
  /──────\
 / Manual \             Manual testing là chủ yếu
/──────────\
/ Unit Tests \          Rất ít unit tests
```
Đây là thực trạng của nhiều dự án: chậm, tốn kém, không ổn định.

---

## 8.2 Lựa chọn Ngôn ngữ Lập trình

### 8.2.1 JavaScript / TypeScript (Khuyến nghị)

**Lý do chọn:**
- Playwright và Cypress — hai framework E2E hàng đầu — được viết bằng JS/TS
- TypeScript cho phép type-safe, phát hiện lỗi sớm hơn khi viết test
- Một ngôn ngữ dùng cho cả frontend, API testing (Node.js), và E2E
- Cộng đồng lớn nhất, tài liệu phong phú

**Phù hợp với:**
- Dự án web frontend (React, Vue, Angular)
- Team đã biết JavaScript
- Muốn dùng Playwright hoặc Cypress

### 8.2.2 Python

**Lý do chọn:**
- Cú pháp đơn giản, dễ học cho người mới
- Pytest là framework kiểm thử mạnh và linh hoạt
- Mạnh trong data-driven testing và API testing (requests library)
- Phổ biến trong AI/ML testing và data pipeline testing

**Phù hợp với:**
- Người mới bắt đầu lập trình
- Dự án backend Python (Django, FastAPI)
- Automation API testing

### 8.2.3 Java

**Lý do chọn:**
- Phổ biến trong doanh nghiệp lớn, tài chính, ngân hàng
- TestNG và JUnit là framework chuẩn enterprise
- Selenium WebDriver gốc viết bằng Java
- Type-safe, hiệu suất tốt

**Phù hợp với:**
- Doanh nghiệp lớn, banking/finance
- Team đã có background Java
- Dự án backend Java (Spring Boot)

---

## 8.3 Kiến thức Lập trình Cần thiết

### 8.3.1 Biến và Kiểu dữ liệu (TypeScript)

```typescript
// Khai báo biến
let userName: string = "Nguyễn Văn A";
const BASE_URL: string = "https://api.example.com";  // constant
let isLoggedIn: boolean = false;
let itemCount: number = 0;
let price: number = 199.99;

// Array
let products: string[] = ["Áo thun", "Quần jeans", "Giày"];
let prices: number[] = [200000, 500000, 350000];

// Object
interface User {
    id: number;
    email: string;
    name: string;
    role: "admin" | "customer" | "manager";
}

const testUser: User = {
    id: 123,
    email: "test@example.com",
    name: "Test User",
    role: "customer"
};

// Union types
type Status = "pending" | "processing" | "delivered" | "cancelled";
let orderStatus: Status = "pending";

// Optional properties
interface Product {
    id: number;
    name: string;
    price: number;
    discount?: number;  // optional
}
```

### 8.3.2 Điều kiện và Vòng lặp

```typescript
// If / Else
function getDiscountMessage(discount: number): string {
    if (discount === 0) {
        return "Không có giảm giá";
    } else if (discount < 20) {
        return `Giảm ${discount}% — giảm ít`;
    } else if (discount < 50) {
        return `Giảm ${discount}% — giảm tốt`;
    } else {
        return `Giảm ${discount}% — siêu sale!`;
    }
}

// Switch
function getStatusLabel(status: string): string {
    switch (status) {
        case "pending":   return "Chờ xử lý";
        case "processing": return "Đang xử lý";
        case "shipped":   return "Đang giao";
        case "delivered": return "Đã giao";
        case "cancelled": return "Đã hủy";
        default:          return "Không xác định";
    }
}

// For loop
const testEmails = ["user1@test.com", "user2@test.com", "user3@test.com"];
for (const email of testEmails) {
    console.log(`Testing with: ${email}`);
}

// For với index
for (let i = 0; i < 5; i++) {
    console.log(`Lần thử ${i + 1}`);
}

// While
let retryCount = 0;
while (retryCount < 3) {
    // thử lại
    retryCount++;
}

// Array methods
const orders = [
    { id: 1, status: "delivered", total: 200000 },
    { id: 2, status: "cancelled", total: 150000 },
    { id: 3, status: "delivered", total: 300000 },
];

// Filter
const deliveredOrders = orders.filter(o => o.status === "delivered");

// Map
const totals = orders.map(o => o.total);

// Find
const firstDelivered = orders.find(o => o.status === "delivered");

// Every / Some
const allDelivered = orders.every(o => o.status === "delivered");
const hasDelivered = orders.some(o => o.status === "delivered");

// Reduce
const totalRevenue = orders.reduce((sum, o) => sum + o.total, 0);
```

### 8.3.3 Hàm / Function

```typescript
// Function thường
function calculateTotal(price: number, quantity: number, discount: number = 0): number {
    const subtotal = price * quantity;
    return subtotal * (1 - discount / 100);
}

// Arrow function
const formatCurrency = (amount: number): string => {
    return amount.toLocaleString("vi-VN", {
        style: "currency",
        currency: "VND"
    });
};

// Async function (quan trọng trong test automation)
async function loginAndGetToken(email: string, password: string): Promise<string> {
    const response = await fetch("/api/auth/login", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    return data.token;
}

// Function với nhiều return type
function findUser(id: number): User | null {
    // ... logic
    return null;  // không tìm thấy
}
```

### 8.3.4 Class và OOP — Nền tảng cho Page Object Model

```typescript
// Class cơ bản
class LoginPage {
    private baseUrl: string;

    constructor(baseUrl: string) {
        this.baseUrl = baseUrl;
    }

    getUrl(): string {
        return `${this.baseUrl}/login`;
    }

    getEmailSelector(): string {
        return "#email";
    }

    getPasswordSelector(): string {
        return "#password";
    }

    getSubmitButtonSelector(): string {
        return 'button[type="submit"]';
    }
}

// Kế thừa (Inheritance)
class AdminLoginPage extends LoginPage {
    constructor(baseUrl: string) {
        super(baseUrl);
    }

    getUrl(): string {
        return `${super.getUrl()}/admin`;  // override
    }
}

// Sử dụng
const loginPage = new LoginPage("https://app.example.com");
console.log(loginPage.getUrl());  // https://app.example.com/login
```

### 8.3.5 Exception Handling

```typescript
// Try / Catch / Finally
async function safeApiCall(url: string): Promise<any> {
    try {
        const response = await fetch(url);

        if (!response.ok) {
            throw new Error(`API Error: ${response.status} ${response.statusText}`);
        }

        return await response.json();
    } catch (error) {
        if (error instanceof TypeError) {
            console.error("Network error — server unreachable");
        } else {
            console.error("Unexpected error:", error);
        }
        throw error;  // re-throw để test fail rõ ràng
    } finally {
        console.log("API call completed");  // luôn chạy
    }
}

// Custom Error
class TestDataError extends Error {
    constructor(message: string) {
        super(message);
        this.name = "TestDataError";
    }
}

function getTestUser(role: string): User {
    const users: Record<string, User> = {
        customer: { id: 1, email: "buyer@test.com", name: "Buyer", role: "customer" },
        admin: { id: 2, email: "admin@test.com", name: "Admin", role: "admin" },
    };

    const user = users[role];
    if (!user) {
        throw new TestDataError(`No test user found for role: ${role}`);
    }
    return user;
}
```

### 8.3.6 JSON và Data Manipulation

```typescript
// Parse JSON
const jsonString = '{"id": 123, "name": "Test", "active": true}';
const data = JSON.parse(jsonString);
console.log(data.id);  // 123

// Stringify
const user = { id: 456, email: "test@example.com" };
const json = JSON.stringify(user, null, 2);  // pretty print
console.log(json);

// Làm việc với nested object
interface OrderResponse {
    success: boolean;
    data: {
        order: {
            id: string;
            status: string;
            items: Array<{
                productId: string;
                quantity: number;
                price: number;
            }>;
        };
    };
}

function extractOrderTotal(response: OrderResponse): number {
    return response.data.order.items.reduce(
        (total, item) => total + item.quantity * item.price,
        0
    );
}

// Optional chaining (an toàn với nested properties)
const orderResponse: OrderResponse | null = null;
const orderId = orderResponse?.data?.order?.id ?? "unknown";
```

---

## 8.4 Các Khái niệm Automation Quan trọng

### 8.4.1 Page Object Model (POM)

**Page Object Model** là design pattern tổ chức code automation: mỗi trang/màn hình trong ứng dụng được đại diện bởi một class (Page Object), chứa các selector và hành động của trang đó.

**Tại sao cần POM:**

**Không có POM — code lộn xộn:**
```typescript
// test_login.spec.ts — selector rải rác khắp nơi
test("Đăng nhập thành công", async ({ page }) => {
    await page.goto("https://app.example.com/login");
    await page.fill("#email", "user@test.com");
    await page.fill("#password", "Test@123");
    await page.click('button[type="submit"]');
    await expect(page).toHaveURL(/dashboard/);
});

// test_navigation.spec.ts — lặp lại selector
test("Vào profile sau login", async ({ page }) => {
    await page.goto("https://app.example.com/login");
    await page.fill("#email", "user@test.com");  // ← Trùng!
    await page.fill("#password", "Test@123");     // ← Trùng!
    await page.click('button[type="submit"]');    // ← Trùng!
    // Khi selector #email đổi thành #user-email → phải sửa ở MỌI FILE
});
```

**Có POM — code sạch, dễ bảo trì:**
```typescript
// pages/LoginPage.ts
import { Page, Locator } from "@playwright/test";

export class LoginPage {
    readonly page: Page;
    readonly emailInput: Locator;
    readonly passwordInput: Locator;
    readonly submitButton: Locator;
    readonly errorMessage: Locator;

    constructor(page: Page) {
        this.page = page;
        this.emailInput = page.locator("#email");
        this.passwordInput = page.locator("#password");
        this.submitButton = page.locator('button[type="submit"]');
        this.errorMessage = page.locator(".error-message");
    }

    async goto(): Promise<void> {
        await this.page.goto("/login");
    }

    async login(email: string, password: string): Promise<void> {
        await this.emailInput.fill(email);
        await this.passwordInput.fill(password);
        await this.submitButton.click();
    }

    async getErrorMessage(): Promise<string> {
        return await this.errorMessage.textContent() ?? "";
    }
}

// pages/DashboardPage.ts
import { Page, Locator } from "@playwright/test";

export class DashboardPage {
    readonly page: Page;
    readonly userDisplayName: Locator;
    readonly logoutButton: Locator;

    constructor(page: Page) {
        this.page = page;
        this.userDisplayName = page.locator(".user-display-name");
        this.logoutButton = page.locator('[data-testid="logout-btn"]');
    }

    async getUserName(): Promise<string> {
        return await this.userDisplayName.textContent() ?? "";
    }

    async isVisible(): Promise<boolean> {
        return await this.page.url().includes("/dashboard");
    }
}

// tests/login.spec.ts — Test file gọn gàng
import { test, expect } from "@playwright/test";
import { LoginPage } from "../pages/LoginPage";
import { DashboardPage } from "../pages/DashboardPage";

test("Đăng nhập thành công", async ({ page }) => {
    const loginPage = new LoginPage(page);
    const dashboardPage = new DashboardPage(page);

    await loginPage.goto();
    await loginPage.login("user@test.com", "Test@123");

    await expect(page).toHaveURL(/dashboard/);
    expect(await dashboardPage.getUserName()).toContain("Nguyễn Văn A");
});

test("Đăng nhập thất bại với password sai", async ({ page }) => {
    const loginPage = new LoginPage(page);

    await loginPage.goto();
    await loginPage.login("user@test.com", "WrongPassword");

    const error = await loginPage.getErrorMessage();
    expect(error).toContain("Email hoặc mật khẩu không đúng");
    await expect(page).toHaveURL(/login/);
});
// Khi selector đổi → chỉ sửa LoginPage.ts, không sửa test files
```

---

### 8.4.2 Test Fixtures

**Fixture** là trạng thái hoặc dữ liệu được thiết lập sẵn trước khi test chạy, và dọn dẹp sau khi test kết thúc. Fixture đảm bảo mỗi test chạy trong môi trường sạch, không phụ thuộc nhau.

```typescript
// Playwright fixtures
import { test as base, expect } from "@playwright/test";
import { LoginPage } from "./pages/LoginPage";
import { DashboardPage } from "./pages/DashboardPage";

// Mở rộng test với custom fixtures
type MyFixtures = {
    loginPage: LoginPage;
    dashboardPage: DashboardPage;
    authenticatedPage: { page: any; token: string };
};

export const test = base.extend<MyFixtures>({
    // Fixture: LoginPage instance
    loginPage: async ({ page }, use) => {
        const loginPage = new LoginPage(page);
        await use(loginPage);
        // Cleanup sau test (nếu cần)
    },

    // Fixture: User đã đăng nhập sẵn
    authenticatedPage: async ({ page }, use) => {
        // Setup: đăng nhập trước khi test
        const loginPage = new LoginPage(page);
        await loginPage.goto();
        await loginPage.login("buyer@test.com", "Test@123");
        await page.waitForURL(/dashboard/);

        const token = await page.evaluate(() =>
            localStorage.getItem("auth_token")
        );

        await use({ page, token: token! });

        // Cleanup: đăng xuất sau test
        await page.goto("/logout");
    },
});

// Sử dụng fixture trong test
test("Xem danh sách đơn hàng — cần đăng nhập", async ({ authenticatedPage }) => {
    const { page } = authenticatedPage;
    await page.goto("/orders");
    await expect(page.locator(".order-list")).toBeVisible();
});
```

**Pytest Fixtures (Python):**
```python
# conftest.py
import pytest
import requests

BASE_URL = "https://api-staging.example.com"

@pytest.fixture(scope="session")
def api_client():
    """Fixture: HTTP session dùng chung cho cả test session"""
    session = requests.Session()
    session.headers.update({
        "Content-Type": "application/json",
        "Accept": "application/json"
    })
    yield session
    session.close()

@pytest.fixture(scope="function")
def auth_token(api_client):
    """Fixture: Token mới cho mỗi test function"""
    response = api_client.post(f"{BASE_URL}/api/auth/login", json={
        "email": "buyer@test.com",
        "password": "Test@123"
    })
    assert response.status_code == 200
    token = response.json()["data"]["token"]
    yield token
    # Cleanup: logout sau mỗi test
    api_client.post(f"{BASE_URL}/api/auth/logout",
                    headers={"Authorization": f"Bearer {token}"})

@pytest.fixture(scope="function")
def test_product(api_client, auth_token):
    """Fixture: Tạo sản phẩm test, xóa sau khi test xong"""
    # Setup
    response = api_client.post(
        f"{BASE_URL}/api/admin/products",
        json={
            "name": "[TEST] Sản phẩm tự động",
            "price": 100000,
            "stock": 50
        },
        headers={"Authorization": f"Bearer {auth_token}"}
    )
    assert response.status_code == 201
    product_id = response.json()["data"]["id"]

    yield product_id  # Truyền product_id cho test

    # Teardown: xóa sản phẩm sau test
    api_client.delete(
        f"{BASE_URL}/api/admin/products/{product_id}",
        headers={"Authorization": f"Bearer {auth_token}"}
    )

# Sử dụng trong test
def test_add_product_to_cart(api_client, auth_token, test_product):
    response = api_client.post(
        f"{BASE_URL}/api/cart/items",
        json={"productId": test_product, "quantity": 2},
        headers={"Authorization": f"Bearer {auth_token}"}
    )
    assert response.status_code == 200
```

---

### 8.4.3 Mock và Stub

**Mock** và **Stub** là các kỹ thuật thay thế phụ thuộc thật (database, API bên ngoài, email service) bằng phiên bản giả để kiểm thử độc lập.

**Stub:** Trả về dữ liệu cố định, không kiểm tra cách gọi.
**Mock:** Trả về dữ liệu cố định VÀ kiểm tra rằng nó được gọi đúng cách.

```typescript
// Ví dụ trong Playwright: Mock API response
test("Hiển thị danh sách sản phẩm từ API", async ({ page }) => {
    // Mock API response — không cần server thật
    await page.route("**/api/products", async (route) => {
        await route.fulfill({
            status: 200,
            contentType: "application/json",
            body: JSON.stringify({
                success: true,
                data: [
                    { id: 1, name: "Mock Product A", price: 100000 },
                    { id: 2, name: "Mock Product B", price: 200000 }
                ]
            })
        });
    });

    await page.goto("/products");
    await expect(page.locator(".product-card")).toHaveCount(2);
    await expect(page.locator(".product-card").first()).toContainText("Mock Product A");
});

// Mock API lỗi
test("Hiển thị thông báo lỗi khi API thất bại", async ({ page }) => {
    await page.route("**/api/products", async (route) => {
        await route.fulfill({
            status: 500,
            contentType: "application/json",
            body: JSON.stringify({ error: "Internal Server Error" })
        });
    });

    await page.goto("/products");
    await expect(page.locator(".error-banner")).toBeVisible();
    await expect(page.locator(".error-banner")).toContainText("Không thể tải sản phẩm");
});
```

---

### 8.4.4 Test Data Management

**Chiến lược quản lý test data trong automation:**

```typescript
// data/testData.ts
export const TEST_ACCOUNTS = {
    buyer: {
        email: "buyer@test.com",
        password: "Test@123",
        name: "Test Buyer"
    },
    admin: {
        email: "admin@test.com",
        password: "Admin@123",
        name: "Test Admin"
    },
    lockedUser: {
        email: "locked@test.com",
        password: "Test@123"
    }
} as const;

export const TEST_PRODUCTS = {
    inStock: {
        id: "P001",
        name: "[TEST] Áo thun Basic",
        price: 200_000,
        stock: 50
    },
    outOfStock: {
        id: "P002",
        name: "[TEST] Giày limited",
        price: 500_000,
        stock: 0
    },
    saleProduct: {
        id: "P003",
        name: "[TEST] Váy hè",
        price: 300_000,
        salePercent: 50
    }
} as const;

export const COUPON_CODES = {
    valid20: "SUMMER20",
    expired: "WINTER10",
    vipOnly: "VIP30"
} as const;

// Helpers tạo data động
export function generateUniqueEmail(): string {
    return `qa_${Date.now()}_${Math.random().toString(36).slice(2, 7)}@test.com`;
}

export function generateOrderData(productId: string, quantity: number) {
    return {
        items: [{ productId, quantity }],
        shippingAddress: {
            street: "123 Test Street",
            district: "Test District",
            city: "Test City"
        },
        paymentMethod: "cod"
    };
}
```

---

### 8.4.5 Assertion

**Assertion** là các câu lệnh xác nhận kết quả thực tế khớp với kết quả mong đợi. Nếu assertion fail, test case được đánh dấu FAIL.

**Playwright assertions:**
```typescript
import { expect } from "@playwright/test";

// Element visibility
await expect(page.locator(".success-message")).toBeVisible();
await expect(page.locator(".error-message")).toBeHidden();

// Text content
await expect(page.locator("h1")).toHaveText("Trang chủ");
await expect(page.locator(".error")).toContainText("không hợp lệ");

// URL
await expect(page).toHaveURL("https://app.example.com/dashboard");
await expect(page).toHaveURL(/dashboard/);

// Attribute
await expect(page.locator("button[type='submit']")).toBeEnabled();
await expect(page.locator("input[name='email']")).toBeDisabled();
await expect(page.locator("img.product")).toHaveAttribute("alt", "Product image");

// Count
await expect(page.locator(".product-card")).toHaveCount(12);
await expect(page.locator(".product-card")).toHaveCount(0);  // không có element nào

// Input value
await expect(page.locator("#email")).toHaveValue("user@test.com");

// Soft assertions — test vẫn chạy tiếp dù assertion fail
await expect.soft(page.locator(".price")).toContainText("200,000");
await expect.soft(page.locator(".discount")).toContainText("20%");
// Tổng hợp tất cả soft assertion fails ở cuối test
```

---

### 8.4.6 Parallel Testing

**Parallel Testing** chạy nhiều test đồng thời để giảm tổng thời gian chạy.

```typescript
// playwright.config.ts
import { defineConfig } from "@playwright/test";

export default defineConfig({
    // Số lượng worker chạy song song
    workers: process.env.CI ? 2 : 4,

    // Mỗi file test chạy độc lập — có thể parallel
    fullyParallel: true,

    // Retry khi test fail (để xử lý flaky test)
    retries: process.env.CI ? 2 : 0,

    // Timeout
    timeout: 30_000,
    expect: { timeout: 5_000 },
});
```

**Lưu ý khi chạy parallel:**
- Tests phải độc lập — không chia sẻ state, không phụ thuộc thứ tự
- Test data phải không xung đột — mỗi test worker dùng data riêng
- Database phải xử lý được concurrent requests

---

### 8.4.7 Retry Logic và Flaky Test

**Flaky Test** là test đôi khi pass, đôi khi fail mà không có thay đổi code — nguyên nhân thường là:
- Timing issues: element chưa load kịp khi assertion chạy
- Network latency: API chậm hơn bình thường
- Race condition: hai action xảy ra đồng thời
- Test data không sạch: data từ test trước ảnh hưởng

**Cách xử lý Flaky Test:**

```typescript
// 1. Dùng proper waits thay vì hardcode timeout
// ❌ Tệ
await page.waitForTimeout(3000);  // chờ 3 giây cứng nhắc

// ✅ Tốt — chờ đến khi element xuất hiện
await page.waitForSelector(".success-message");
await expect(page.locator(".success-message")).toBeVisible();

// ✅ Tốt — chờ API response
await page.waitForResponse("**/api/orders");

// 2. Retry cho toàn bộ test (cấu hình)
// playwright.config.ts
retries: 2  // retry 2 lần nếu fail

// 3. Retry cho action cụ thể
const maxRetries = 3;
for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
        await page.click(".submit-btn");
        await expect(page).toHaveURL(/dashboard/);
        break;  // thành công, thoát loop
    } catch (error) {
        if (attempt === maxRetries) throw error;
        await page.waitForTimeout(1000);  // chờ 1s rồi thử lại
    }
}

// 4. Test Isolation — mỗi test bắt đầu từ trạng thái sạch
test.beforeEach(async ({ page }) => {
    // Reset state
    await page.evaluate(() => {
        localStorage.clear();
        sessionStorage.clear();
    });
    await page.context().clearCookies();
});
```

---

### 8.4.8 Báo cáo — Screenshot, Video, Trace

```typescript
// playwright.config.ts
export default defineConfig({
    use: {
        // Chụp screenshot khi test fail
        screenshot: "only-on-failure",

        // Quay video
        video: "retain-on-failure",  // chỉ giữ video khi fail

        // Trace — record toàn bộ actions để debug
        trace: "on-first-retry",  // trace khi retry lần đầu
    },

    reporter: [
        ["html", { open: "never" }],  // HTML report
        ["json", { outputFile: "results/test-results.json" }],
        ["junit", { outputFile: "results/junit.xml" }],  // cho CI
    ],
});
```

**Xem Trace trong Playwright:**
```bash
# Sau khi test chạy, xem trace
npx playwright show-trace test-results/login-test/trace.zip
```

Trace Viewer cho phép xem từng bước test như video, kèm network requests, console logs — rất hữu ích để debug flaky test.
