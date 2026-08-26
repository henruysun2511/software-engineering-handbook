# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# CHƯƠNG 11: E2E TESTING

---

## 11.1 E2E Testing là gì?

**End-to-End (E2E) Testing** kiểm thử toàn bộ luồng ứng dụng từ góc nhìn người dùng thực sự — từ giao diện người dùng (UI) qua backend, database, và các dịch vụ liên quan. E2E test mô phỏng chính xác hành vi của người dùng trên trình duyệt thật.

**Bản chất của E2E Test:**

```
E2E Test (Playwright/Cypress)
    ↓  mở trình duyệt, click, nhập liệu
[Browser / UI]
    ↓  HTTP request
[Backend API]
    ↓  query
[Database]
    ↓  nếu có
[External Services: Email, Payment, SMS...]
```

Không có mock, không có stub — tất cả đều thật (trừ payment gateway thường dùng sandbox).

---

### 11.1.1 Khi nào cần E2E Testing?

**Nên dùng E2E cho:**
- Critical user journeys: đăng ký, đăng nhập, mua hàng, thanh toán
- Luồng nghiệp vụ phức tạp qua nhiều màn hình
- Kiểm tra sự tích hợp giữa frontend và backend trong môi trường thật
- Smoke test sau mỗi deployment

**Không nên dùng E2E cho:**
- Kiểm thử tất cả validation logic (dùng Unit Test nhanh hơn)
- Kiểm thử tất cả error case của API (dùng API test)
- Regression đầy đủ cho mọi tính năng (sẽ quá chậm)

---

### 11.1.2 E2E Test Flow và User Journey

Mỗi E2E test nên phản ánh một **User Journey** — chuỗi hành động hoàn chỉnh của người dùng thực:

```
User Journey: Mua hàng lần đầu

1. Người dùng mở trang chủ
2. Tìm kiếm "áo thun"
3. Chọn sản phẩm đầu tiên
4. Chọn size M, số lượng 2
5. Thêm vào giỏ hàng
6. Vào trang giỏ hàng
7. Nhập mã giảm giá
8. Tiến hành thanh toán
9. Điền thông tin giao hàng
10. Chọn COD
11. Xác nhận đặt hàng
12. Xem trang xác nhận với Order ID
```

---

## 11.2 Playwright — Framework E2E Hiện đại

### 11.2.1 Tại sao chọn Playwright?

| Tiêu chí | Playwright | Cypress | Selenium |
|---|---|---|---|
| Trình duyệt hỗ trợ | Chromium, Firefox, WebKit | Chromium (chủ yếu) | Tất cả |
| Tốc độ | Rất nhanh | Nhanh | Trung bình |
| Auto-wait | ✅ Tự động | ✅ Tự động | ❌ Phải set thủ công |
| Cross-browser parallel | ✅ Native | ❌ Cần plugin | ✅ |
| Network interception | ✅ Mạnh | ✅ | ❌ |
| Mobile emulation | ✅ | ❌ | Cần Appium |
| Trace viewer | ✅ Tuyệt vời | ❌ | ❌ |
| Cộng đồng | Đang tăng mạnh | Lớn | Rất lớn (cũ) |

### 11.2.2 Cài đặt và cấu hình

```bash
# Khởi tạo project Playwright
npm init playwright@latest

# Hoặc thêm vào project có sẵn
npm install -D @playwright/test
npx playwright install  # tải browser binaries
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
    // Thư mục chứa test files
    testDir: "./tests",

    // Pattern file test
    testMatch: "**/*.spec.ts",

    // Chạy song song
    fullyParallel: true,

    // Retry khi fail (chỉ trong CI)
    retries: process.env.CI ? 2 : 0,

    // Số workers song song
    workers: process.env.CI ? 2 : undefined,

    // Reporter
    reporter: [
        ["html"],
        ["list"],
    ],

    // Cấu hình chung cho tất cả test
    use: {
        baseURL: process.env.BASE_URL ?? "http://localhost:3000",
        screenshot: "only-on-failure",
        video: "retain-on-failure",
        trace: "on-first-retry",
        locale: "vi-VN",
        timezoneId: "Asia/Ho_Chi_Minh",
    },

    // Chạy trên nhiều browser
    projects: [
        {
            name: "chromium",
            use: { ...devices["Desktop Chrome"] },
        },
        {
            name: "firefox",
            use: { ...devices["Desktop Firefox"] },
        },
        {
            name: "webkit",
            use: { ...devices["Desktop Safari"] },
        },
        // Mobile
        {
            name: "mobile-chrome",
            use: { ...devices["Pixel 7"] },
        },
        {
            name: "mobile-safari",
            use: { ...devices["iPhone 14"] },
        },
    ],

    // Chạy dev server trước khi test
    webServer: {
        command: "npm run dev",
        url: "http://localhost:3000",
        reuseExistingServer: !process.env.CI,
    },
});
```

---

### 11.2.3 Locators và Selectors

**Playwright ưu tiên Locators** theo thứ tự từ bền vững nhất đến dễ hỏng nhất:

```typescript
// ✅ Tốt nhất — Accessibility locators (semantic, không đổi khi UI thay đổi)
page.getByRole("button", { name: "Đăng nhập" })
page.getByRole("textbox", { name: "Email" })
page.getByRole("heading", { name: "Giỏ hàng" })
page.getByLabel("Mật khẩu")
page.getByPlaceholder("Nhập email của bạn")
page.getByText("Đặt hàng thành công")
page.getByAltText("Ảnh sản phẩm")

// ✅ Tốt — data-testid (được thêm vào HTML cho mục đích testing)
page.getByTestId("submit-button")
page.getByTestId("product-card")

// ⚠️ Chấp nhận được — CSS selector ổn định
page.locator("#email")
page.locator(".product-card")
page.locator("[name='email']")

// ❌ Tránh — dễ hỏng khi thay đổi UI
page.locator("div > div > button:nth-child(3)")
page.locator("//div[@class='container']/button[1]")  // XPath phức tạp
```

**Thêm data-testid vào HTML:**
```html
<!-- Trong React/Vue/HTML -->
<button type="submit" data-testid="login-submit-btn">
    Đăng nhập
</button>

<div class="product-card" data-testid="product-card" data-product-id="P001">
    ...
</div>
```

**Filtering và chaining Locators:**
```typescript
// Tìm trong container cụ thể
const productCard = page.locator('[data-testid="product-card"]').filter({
    hasText: "Áo thun Basic"
});
await productCard.getByRole("button", { name: "Thêm vào giỏ" }).click();

// Lấy phần tử thứ N
const firstProduct = page.locator('[data-testid="product-card"]').first();
const thirdProduct = page.locator('[data-testid="product-card"]').nth(2);

// Kiểm tra count
await expect(page.locator('[data-testid="product-card"]')).toHaveCount(12);
```

---

### 11.2.4 Actions — Tương tác với trang

```typescript
// Navigation
await page.goto("/login");
await page.goto("https://example.com");
await page.goBack();
await page.reload();

// Click
await page.getByRole("button", { name: "Đặt hàng" }).click();
await page.getByTestId("product-card").first().click();
// Double click
await page.getByText("Sản phẩm").dblclick();
// Right click
await page.getByText("Sản phẩm").click({ button: "right" });

// Nhập liệu
await page.getByLabel("Email").fill("user@test.com");
await page.getByLabel("Email").clear();
await page.getByLabel("Email").pressSequentially("user@test.com", { delay: 50 });

// Select dropdown
await page.getByLabel("Thành phố").selectOption("TP.HCM");
await page.getByLabel("Tỉnh/TP").selectOption({ label: "Hà Nội" });

// Checkbox và Radio
await page.getByLabel("Tôi đồng ý với điều khoản").check();
await page.getByLabel("COD").check();
await expect(page.getByLabel("COD")).toBeChecked();

// File upload
await page.getByLabel("Ảnh đại diện").setInputFiles("./test-files/avatar.jpg");
await page.getByLabel("Ảnh sản phẩm").setInputFiles([
    "./test-files/product1.jpg",
    "./test-files/product2.jpg"
]);

// Hover
await page.getByText("Tài khoản").hover();  // mở dropdown menu

// Keyboard
await page.keyboard.press("Tab");
await page.keyboard.press("Enter");
await page.keyboard.press("Escape");
await page.getByLabel("Tìm kiếm").press("Enter");

// Scroll
await page.evaluate(() => window.scrollTo(0, 1000));
await page.getByTestId("load-more-btn").scrollIntoViewIfNeeded();
```

---

### 11.2.5 Assertions

```typescript
import { expect } from "@playwright/test";

// ===== Visibility =====
await expect(page.getByText("Đặt hàng thành công")).toBeVisible();
await expect(page.getByTestId("error-msg")).toBeHidden();
await expect(page.getByTestId("loading-spinner")).not.toBeVisible();

// ===== Text content =====
await expect(page.getByRole("heading")).toHaveText("Giỏ hàng của bạn");
await expect(page.getByTestId("total-price")).toContainText("200,000");
await expect(page.getByTestId("error-msg")).toHaveText(/không hợp lệ/);

// ===== URL =====
await expect(page).toHaveURL("/dashboard");
await expect(page).toHaveURL(/dashboard/);
await expect(page).not.toHaveURL(/login/);

// ===== Element state =====
await expect(page.getByRole("button", { name: "Đặt hàng" })).toBeEnabled();
await expect(page.getByRole("button", { name: "Hết hàng" })).toBeDisabled();
await expect(page.getByLabel("Email")).toBeFocused();

// ===== Attribute =====
await expect(page.getByTestId("product-img")).toHaveAttribute("alt", "Áo thun Basic");
await expect(page.getByLabel("Email")).toHaveValue("user@test.com");
await expect(page.getByTestId("coupon-input")).toHaveAttribute("placeholder", "Nhập mã giảm giá");

// ===== Count =====
await expect(page.getByTestId("product-card")).toHaveCount(12);
await expect(page.getByTestId("cart-item")).toHaveCount(3);

// ===== CSS class =====
await expect(page.getByTestId("nav-home")).toHaveClass(/active/);

// ===== Page title =====
await expect(page).toHaveTitle("Trang chủ | Shop Online");

// ===== Soft assertions (không dừng test khi fail) =====
await expect.soft(page.getByTestId("product-name")).toHaveText("Áo thun Basic");
await expect.soft(page.getByTestId("product-price")).toContainText("200,000");
await expect.soft(page.getByTestId("product-stock")).toContainText("Còn hàng");
// Nếu một soft assertion fail, test vẫn chạy tiếp, báo tất cả lỗi ở cuối
```

---

### 11.2.6 Wait Strategies

Playwright **tự động chờ** trước khi thực hiện action — đây là điểm mạnh lớn so với Selenium. Nhưng có những trường hợp cần wait tường minh:

```typescript
// Playwright tự động chờ:
// - Element hiển thị trước khi click
// - Element enabled trước khi fill
// - Navigation hoàn thành sau khi click link

// Tường minh — chờ element xuất hiện
await page.waitForSelector(".success-toast");

// Chờ URL thay đổi
await page.waitForURL(/dashboard/);

// Chờ navigation hoàn thành
await Promise.all([
    page.waitForNavigation(),
    page.click('a[href="/checkout"]')
]);

// Chờ network request
await page.waitForResponse("**/api/orders");
await page.waitForResponse(response =>
    response.url().includes("/api/orders") && response.status() === 201
);

// Chờ element biến mất
await expect(page.getByTestId("loading-spinner")).toBeHidden();

// Chờ điều kiện tùy chỉnh
await page.waitForFunction(() => {
    const counter = document.querySelector(".cart-counter");
    return counter && parseInt(counter.textContent!) > 0;
});

// ❌ Tránh dùng — không tin cậy
await page.waitForTimeout(3000);  // hardcode wait
```

---

### 11.2.7 Network Interception

```typescript
// Mock API response
await page.route("**/api/products", route => {
    route.fulfill({
        status: 200,
        contentType: "application/json",
        body: JSON.stringify({
            data: [
                { id: "P001", name: "Mock Product", price: 100000 }
            ]
        })
    });
});

// Chặn request (simulate offline)
await page.route("**/api/**", route => route.abort());

// Modify request
await page.route("**/api/orders", async route => {
    const request = route.request();
    const body = JSON.parse(request.postData() || "{}");
    body.source = "test-automation";  // thêm field

    await route.continue({
        postData: JSON.stringify(body)
    });
});

// Capture request để verify
const requestPromise = page.waitForRequest("**/api/orders");
await page.getByRole("button", { name: "Đặt hàng" }).click();
const request = await requestPromise;
const requestBody = JSON.parse(request.postData() || "{}");
expect(requestBody.paymentMethod).toBe("cod");

// Capture response để verify
const responsePromise = page.waitForResponse("**/api/orders");
await page.getByRole("button", { name: "Đặt hàng" }).click();
const response = await responsePromise;
expect(response.status()).toBe(201);
const responseBody = await response.json();
expect(responseBody.data.orderId).toBeDefined();
```

---

### 11.2.8 Authentication trong E2E — Tối ưu tốc độ

Đăng nhập qua UI cho mỗi test rất chậm. Playwright cho phép lưu authentication state và tái sử dụng:

```typescript
// tests/auth.setup.ts — chạy một lần trước các test
import { test as setup, expect } from "@playwright/test";
import path from "path";

const authFile = path.join(__dirname, "../.auth/buyer.json");

setup("Đăng nhập và lưu session", async ({ page }) => {
    await page.goto("/login");
    await page.getByLabel("Email").fill("buyer@test.com");
    await page.getByLabel("Mật khẩu").fill("Test@123");
    await page.getByRole("button", { name: "Đăng nhập" }).click();
    await expect(page).toHaveURL(/dashboard/);

    // Lưu cookies + localStorage vào file
    await page.context().storageState({ path: authFile });
});

// playwright.config.ts — cấu hình dependencies
export default defineConfig({
    projects: [
        {
            name: "setup",
            testMatch: /auth.setup.ts/,
        },
        {
            name: "authenticated-tests",
            testMatch: /.*\.spec\.ts/,
            dependencies: ["setup"],
            use: {
                storageState: ".auth/buyer.json",  // load saved session
            },
        },
    ],
});

// tests/checkout.spec.ts — không cần login lại
test("Checkout thành công", async ({ page }) => {
    // Session đã được load từ .auth/buyer.json
    // Người dùng đã đăng nhập sẵn
    await page.goto("/cart");
    // ... tiếp tục test
});
```

**Hoặc đăng nhập qua API (nhanh hơn qua UI nhiều lần):**
```typescript
setup("Đăng nhập qua API", async ({ request, page }) => {
    // Gọi API đăng nhập trực tiếp — không cần load UI
    const response = await request.post("/api/auth/login", {
        data: { email: "buyer@test.com", password: "Test@123" }
    });
    const { token } = (await response.json()).data;

    // Inject token vào browser storage
    await page.goto("/");
    await page.evaluate((t) => {
        localStorage.setItem("auth_token", t);
    }, token);

    await page.context().storageState({ path: ".auth/buyer.json" });
});
```

---

### 11.2.9 Screenshots, Videos, và Trace Viewer

```typescript
// Chụp screenshot thủ công trong test
await page.screenshot({ path: "screenshots/checkout-page.png" });

// Chụp element cụ thể
await page.getByTestId("order-summary").screenshot({
    path: "screenshots/order-summary.png"
});

// So sánh visual (snapshot testing)
await expect(page).toHaveScreenshot("homepage.png");
await expect(page.getByTestId("product-card").first()).toHaveScreenshot("product-card.png");
// Lần đầu chạy: tạo snapshot baseline
// Lần sau: so sánh với baseline, fail nếu khác
```

**Xem Trace sau khi test fail:**
```bash
# Trace được lưu tự động khi config trace: "on-first-retry"
npx playwright show-trace test-results/checkout-test/trace.zip

# Trace Viewer cho phép xem:
# - Từng action được thực hiện (timeline)
# - Screenshot trước/sau mỗi action
# - Network requests và responses
# - Console logs
# - DOM snapshot tại mỗi thời điểm
```

---

## 11.3 Cypress — Framework E2E Phổ biến

### 11.3.1 Cài đặt và cấu hình

```bash
npm install -D cypress
npx cypress open  # mở Cypress Test Runner
```

```javascript
// cypress.config.js
const { defineConfig } = require("cypress");

module.exports = defineConfig({
    e2e: {
        baseUrl: "http://localhost:3000",
        specPattern: "cypress/e2e/**/*.cy.js",
        viewportWidth: 1280,
        viewportHeight: 720,
        defaultCommandTimeout: 10000,
        video: true,
        screenshotOnRunFailure: true,
    },
});
```

### 11.3.2 Cú pháp Cypress

```javascript
// cypress/e2e/login.cy.js
describe("Tính năng Đăng nhập", () => {
    beforeEach(() => {
        cy.visit("/login");
    });

    it("Đăng nhập thành công với thông tin hợp lệ", () => {
        cy.get("#email").type("user@test.com");
        cy.get("#password").type("Test@123");
        cy.get('[type="submit"]').click();

        cy.url().should("include", "/dashboard");
        cy.get(".user-display-name").should("contain", "Nguyễn Văn A");
    });

    it("Hiển thị lỗi khi password sai", () => {
        cy.get("#email").type("user@test.com");
        cy.get("#password").type("WrongPassword");
        cy.get('[type="submit"]').click();

        cy.get(".error-message")
            .should("be.visible")
            .and("contain", "Email hoặc mật khẩu không đúng");
        cy.url().should("include", "/login");
    });
});
```

**Cypress Custom Commands:**
```javascript
// cypress/support/commands.js
Cypress.Commands.add("login", (email, password) => {
    cy.session([email, password], () => {
        cy.visit("/login");
        cy.get("#email").type(email);
        cy.get("#password").type(password);
        cy.get('[type="submit"]').click();
        cy.url().should("include", "/dashboard");
    });
});

Cypress.Commands.add("addToCart", (productId, quantity = 1) => {
    cy.request({
        method: "POST",
        url: "/api/cart/items",
        body: { productId, quantity },
        headers: {
            Authorization: `Bearer ${localStorage.getItem("auth_token")}`
        }
    });
});

// Sử dụng trong test
it("Checkout sau khi thêm sản phẩm", () => {
    cy.login("buyer@test.com", "Test@123");
    cy.addToCart("P001", 2);
    cy.visit("/cart");
    cy.get('[data-testid="checkout-btn"]').click();
    // ...
});
```

**Fixtures và Mocking trong Cypress:**
```javascript
// cypress/fixtures/products.json
[
    { "id": "P001", "name": "Áo thun", "price": 200000 },
    { "id": "P002", "name": "Quần jeans", "price": 500000 }
]

// Test sử dụng fixture
it("Hiển thị danh sách sản phẩm từ API mock", () => {
    cy.intercept("GET", "/api/products", { fixture: "products.json" }).as("getProducts");

    cy.visit("/products");
    cy.wait("@getProducts");

    cy.get('[data-testid="product-card"]').should("have.length", 2);
    cy.get('[data-testid="product-card"]').first().should("contain", "Áo thun");
});

// Mock API error
it("Hiển thị lỗi khi API thất bại", () => {
    cy.intercept("GET", "/api/products", {
        statusCode: 500,
        body: { error: "Internal Server Error" }
    });

    cy.visit("/products");
    cy.get('[data-testid="error-banner"]').should("be.visible");
});
```

---

## 11.4 Phạm vi UI Testing trong E2E

### 11.4.1 Form Testing và Validation

```typescript
// Playwright — kiểm thử validation form đầy đủ
test.describe("Form đăng ký — Validation", () => {
    test.beforeEach(async ({ page }) => {
        await page.goto("/register");
    });

    test("Hiển thị lỗi validation khi submit form trống", async ({ page }) => {
        await page.getByRole("button", { name: "Đăng ký" }).click();

        await expect(page.getByTestId("email-error")).toBeVisible();
        await expect(page.getByTestId("email-error")).toHaveText("Vui lòng nhập email");
        await expect(page.getByTestId("password-error")).toBeVisible();
        await expect(page.getByTestId("name-error")).toBeVisible();
    });

    test("Hiển thị lỗi khi email sai định dạng", async ({ page }) => {
        await page.getByLabel("Email").fill("not-an-email");
        await page.getByLabel("Email").blur();  // trigger validation

        await expect(page.getByTestId("email-error"))
            .toHaveText("Email không đúng định dạng");
    });

    test("Hiển thị lỗi khi password không đủ mạnh", async ({ page }) => {
        await page.getByLabel("Mật khẩu").fill("weak");
        await page.getByLabel("Mật khẩu").blur();

        await expect(page.getByTestId("password-error")).toBeVisible();
        // Kiểm tra xem password strength indicator có cập nhật không
        await expect(page.getByTestId("password-strength")).toHaveText("Yếu");
    });

    test("Confirm password không khớp — hiển thị lỗi inline", async ({ page }) => {
        await page.getByLabel("Mật khẩu").fill("Test@123");
        await page.getByLabel("Xác nhận mật khẩu").fill("Test@456");
        await page.getByLabel("Xác nhận mật khẩu").blur();

        await expect(page.getByTestId("confirm-password-error"))
            .toHaveText("Mật khẩu xác nhận không khớp");
    });
});
```

### 11.4.2 Responsive Testing

```typescript
// Test trên nhiều viewport sizes
const viewports = [
    { name: "mobile", width: 375, height: 812 },
    { name: "tablet", width: 768, height: 1024 },
    { name: "desktop", width: 1440, height: 900 },
];

for (const viewport of viewports) {
    test(`Navigation menu hoạt động đúng trên ${viewport.name}`, async ({ page }) => {
        await page.setViewportSize({ width: viewport.width, height: viewport.height });
        await page.goto("/");

        if (viewport.width < 768) {
            // Mobile: hamburger menu
            await expect(page.getByTestId("mobile-menu-btn")).toBeVisible();
            await expect(page.getByTestId("desktop-nav")).toBeHidden();

            await page.getByTestId("mobile-menu-btn").click();
            await expect(page.getByTestId("mobile-nav")).toBeVisible();
        } else {
            // Desktop: full nav
            await expect(page.getByTestId("desktop-nav")).toBeVisible();
            await expect(page.getByTestId("mobile-menu-btn")).toBeHidden();
        }
    });
}
```

### 11.4.3 Accessibility Testing

```typescript
import { checkA11y } from "axe-playwright";

test("Trang đăng nhập đáp ứng tiêu chuẩn Accessibility", async ({ page }) => {
    await page.goto("/login");

    // Kiểm tra với axe-core
    await checkA11y(page, undefined, {
        detailedReport: true,
        detailedReportOptions: { html: true }
    });
});

test("Keyboard navigation hoạt động đúng", async ({ page }) => {
    await page.goto("/login");

    // Tab qua các element theo thứ tự đúng
    await page.keyboard.press("Tab");
    await expect(page.getByLabel("Email")).toBeFocused();

    await page.keyboard.press("Tab");
    await expect(page.getByLabel("Mật khẩu")).toBeFocused();

    await page.keyboard.press("Tab");
    await expect(page.getByRole("button", { name: "Đăng nhập" })).toBeFocused();

    // Enter để submit
    await page.getByLabel("Email").fill("user@test.com");
    await page.getByLabel("Mật khẩu").fill("Test@123");
    await page.keyboard.press("Enter");

    await expect(page).toHaveURL(/dashboard/);
});
```

---

## 11.5 Flaky Test — Nguyên nhân và Cách xử lý

**Flaky test** là test đôi khi pass, đôi khi fail mà không có thay đổi code. Đây là vấn đề phổ biến nhất trong E2E testing.

### 11.5.1 Nguyên nhân và giải pháp

**Nguyên nhân 1: Timing issues — Element chưa ready**
```typescript
// ❌ Tệ — race condition
await page.click(".submit-btn");
await expect(page.locator(".success-msg")).toBeVisible();
// Đôi khi fail vì .success-msg chưa render kịp

// ✅ Tốt — Playwright tự động wait
await page.getByRole("button", { name: "Xác nhận" }).click();
await expect(page.getByTestId("success-msg")).toBeVisible(); // auto-wait built-in
```

**Nguyên nhân 2: Animation chưa hoàn thành**
```typescript
// Chờ animation kết thúc
await page.getByTestId("modal").waitFor({ state: "visible" });
// Hoặc disable animation trong test environment
await page.addStyleTag({
    content: "*, *::before, *::after { animation-duration: 0s !important; }"
});
```

**Nguyên nhân 3: Data race — Test data bị ảnh hưởng bởi test khác**
```typescript
// Mỗi test tạo data riêng, không share
test("Test A", async ({ page }) => {
    const email = `test_a_${Date.now()}@test.com`;
    // dùng email này, không ảnh hưởng Test B
});

test("Test B", async ({ page }) => {
    const email = `test_b_${Date.now()}@test.com`;
    // email riêng
});
```

**Nguyên nhân 4: Network timeout**
```typescript
// Tăng timeout cho test chậm
test("Upload file lớn", async ({ page }) => {
    test.setTimeout(60_000);  // 60 giây cho test này
    // ...
});
```

### 11.5.2 E2E Test Isolation

```typescript
// Mỗi test phải:
// 1. Không phụ thuộc thứ tự chạy
// 2. Không để lại dữ liệu ảnh hưởng test khác
// 3. Không phụ thuộc vào state của test khác

test.describe("Cart tests", () => {
    let testOrderId: string;

    test.beforeEach(async ({ page, request }) => {
        // Đảm bảo giỏ hàng trống trước mỗi test
        await request.delete("/api/cart/clear", {
            headers: { Authorization: `Bearer ${process.env.TEST_TOKEN}` }
        });
    });

    test.afterEach(async ({ request }) => {
        // Xóa order test nếu được tạo
        if (testOrderId) {
            await request.delete(`/api/test/orders/${testOrderId}`, {
                headers: { Authorization: `Bearer ${process.env.ADMIN_TOKEN}` }
            });
        }
    });
});
```

---

## 11.6 Ví dụ thực hành — E2E Test Hoàn chỉnh

### 11.6.1 Luồng Mua Hàng E2E (Playwright)

```typescript
// tests/e2e/purchase-flow.spec.ts
import { test, expect } from "@playwright/test";
import { LoginPage } from "../pages/LoginPage";
import { ProductsPage } from "../pages/ProductsPage";
import { CartPage } from "../pages/CartPage";
import { CheckoutPage } from "../pages/CheckoutPage";
import { OrderConfirmationPage } from "../pages/OrderConfirmationPage";

test.describe("Luồng mua hàng hoàn chỉnh", () => {
    test("Tìm kiếm → Thêm giỏ → Checkout → Đặt hàng COD thành công", async ({ page }) => {
        // Pages
        const productsPage = new ProductsPage(page);
        const cartPage = new CartPage(page);
        const checkoutPage = new CheckoutPage(page);
        const confirmPage = new OrderConfirmationPage(page);

        // Bước 1: Tìm kiếm sản phẩm
        await productsPage.goto();
        await productsPage.search("áo thun");
        await expect(productsPage.productCards).toHaveCount(5);

        // Bước 2: Chọn sản phẩm đầu tiên
        const firstProduct = productsPage.productCards.first();
        const productName = await firstProduct.getByTestId("product-name").textContent();
        await firstProduct.click();

        // Bước 3: Thêm vào giỏ
        await page.getByLabel("Size").selectOption("M");
        await page.getByLabel("Số lượng").fill("2");
        await page.getByRole("button", { name: "Thêm vào giỏ" }).click();

        // Verify toast thông báo
        await expect(page.getByTestId("toast-success")).toBeVisible();
        await expect(page.getByTestId("cart-badge")).toHaveText("2");

        // Bước 4: Vào giỏ hàng
        await cartPage.goto();
        await expect(cartPage.cartItems).toHaveCount(1);
        await expect(cartPage.cartItems.first()).toContainText(productName!);

        // Bước 5: Áp dụng mã giảm giá
        await cartPage.applyCoupon("SUMMER20");
        await expect(cartPage.discountRow).toBeVisible();
        await expect(cartPage.discountRow).toContainText("-");

        // Bước 6: Checkout
        await cartPage.proceedToCheckout();
        await expect(page).toHaveURL(/checkout/);

        // Bước 7: Điền thông tin giao hàng
        await checkoutPage.fillShippingInfo({
            fullName: "Nguyễn Văn Test",
            phone: "0901234567",
            address: "123 Đường Test",
            district: "Quận 1",
            city: "TP.HCM"
        });

        // Bước 8: Chọn phương thức thanh toán
        await checkoutPage.selectPaymentMethod("cod");

        // Bước 9: Xem lại đơn hàng
        const orderSummary = await checkoutPage.getOrderSummary();
        expect(orderSummary.productName).toContain(productName!.trim());
        expect(orderSummary.discountApplied).toBe(true);

        // Bước 10: Xác nhận đặt hàng
        const [response] = await Promise.all([
            page.waitForResponse(r => r.url().includes("/api/orders") && r.status() === 201),
            checkoutPage.confirmOrder()
        ]);

        const responseBody = await response.json();
        const orderId = responseBody.data.orderId;

        // Bước 11: Verify trang xác nhận
        await expect(page).toHaveURL(/order-confirmation/);
        await expect(confirmPage.orderIdText).toContainText(orderId);
        await expect(confirmPage.successMessage).toBeVisible();
        await expect(confirmPage.shippingInfo).toContainText("123 Đường Test");

        // Bước 12: Verify giỏ hàng trống sau khi đặt
        await cartPage.goto();
        await expect(cartPage.emptyCartMessage).toBeVisible();
    });

    test("Không thể checkout khi giỏ hàng trống — redirect về cart", async ({ page }) => {
        await page.goto("/checkout");
        await expect(page).toHaveURL(/cart/);
        await expect(page.getByTestId("empty-cart-msg")).toBeVisible();
    });

    test("Sản phẩm hết hàng trong lúc checkout — thông báo rõ ràng", async ({ page, request }) => {
        // Thêm sản phẩm có stock = 1 vào giỏ
        await request.post("/api/cart/items", {
            data: { productId: "P-LIMITED-1", quantity: 1 },
            headers: { Authorization: `Bearer ${process.env.TEST_BUYER_TOKEN}` }
        });

        // Giả lập: người khác vừa mua hết sản phẩm đó
        await request.post("/api/test/exhaust-stock", {
            data: { productId: "P-LIMITED-1" },
            headers: { Authorization: `Bearer ${process.env.TEST_ADMIN_TOKEN}` }
        });

        // Thử checkout
        const checkoutPage = new CheckoutPage(page);
        await page.goto("/checkout");
        await checkoutPage.fillShippingInfo({ /* ... */ } as any);
        await checkoutPage.confirmOrder();

        // Verify thông báo lỗi
        await expect(page.getByTestId("stock-error")).toBeVisible();
        await expect(page.getByTestId("stock-error")).toContainText("vừa hết hàng");
        await expect(page).not.toHaveURL(/order-confirmation/);
    });
});
```

---

## 11.7 Selenium — Tổng quan và Khi nào dùng

**Selenium WebDriver** là framework E2E cũ nhất và phổ biến nhất trong lịch sử — vẫn được dùng rộng rãi trong doanh nghiệp.

### 11.7.1 Ưu điểm và Nhược điểm

**Vẫn dùng Selenium khi:**
- Dự án cũ đã có bộ test Selenium lớn
- Team đã quen và không muốn migrate
- Cần test trên IE/Edge cũ hoặc browser đặc thù
- Tích hợp với framework Java như TestNG/JUnit trong pipeline enterprise

**Chuyển sang Playwright/Cypress khi:**
- Bắt đầu dự án mới
- Muốn DX (Developer Experience) tốt hơn
- Cần performance và reliability cao hơn
- Muốn built-in auto-wait, trace viewer

### 11.7.2 Ví dụ Selenium với Python

```python
# test_login_selenium.py
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import pytest

@pytest.fixture(scope="function")
def driver():
    options = webdriver.ChromeOptions()
    options.add_argument("--headless")  # chạy không có UI
    options.add_argument("--no-sandbox")
    driver = webdriver.Chrome(options=options)
    driver.implicitly_wait(10)
    yield driver
    driver.quit()

class TestLogin:
    BASE_URL = "http://localhost:3000"

    def test_login_success(self, driver):
        # Arrange
        driver.get(f"{self.BASE_URL}/login")
        wait = WebDriverWait(driver, 10)

        # Act
        email_input = wait.until(EC.presence_of_element_located((By.ID, "email")))
        email_input.send_keys("user@test.com")

        password_input = driver.find_element(By.ID, "password")
        password_input.send_keys("Test@123")

        submit_btn = driver.find_element(By.CSS_SELECTOR, 'button[type="submit"]')
        submit_btn.click()

        # Assert
        wait.until(EC.url_contains("/dashboard"))
        assert "/dashboard" in driver.current_url

        user_name = wait.until(
            EC.presence_of_element_located((By.CLASS_NAME, "user-display-name"))
        )
        assert "Nguyễn Văn A" in user_name.text

    def test_login_fail_wrong_password(self, driver):
        driver.get(f"{self.BASE_URL}/login")
        wait = WebDriverWait(driver, 10)

        driver.find_element(By.ID, "email").send_keys("user@test.com")
        driver.find_element(By.ID, "password").send_keys("WrongPassword")
        driver.find_element(By.CSS_SELECTOR, 'button[type="submit"]').click()

        error_msg = wait.until(
            EC.visibility_of_element_located((By.CLASS_NAME, "error-message"))
        )
        assert "Email hoặc mật khẩu không đúng" in error_msg.text
        assert "/login" in driver.current_url
```
