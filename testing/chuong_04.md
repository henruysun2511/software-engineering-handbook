# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# CHƯƠNG 4: TDD & BDD — TƯ DUY KIỂM THỬ ĐỊNH HƯỚNG PHÁT TRIỂN

---

## 4.1 Giới thiệu: Kiểm thử trước hay sau?

Trong quy trình phát triển truyền thống, kiểm thử là hoạt động diễn ra **sau** khi code được viết xong. Developer hoàn thành tính năng, Tester nhận build và kiểm tra. Vấn đề của mô hình này: lỗi được phát hiện muộn, chi phí sửa cao, và Tester luôn trong trạng thái "chạy theo" deadline.

TDD (Test-driven Development) và BDD (Behavior-driven Development) đảo ngược tư duy này — kiểm thử không phải là bước cuối, mà là **điểm khởi đầu** của quá trình phát triển.

---

## 4.2 TDD — Test-driven Development

### 4.2.1 TDD là gì?

**TDD (Test-driven Development)** là kỹ thuật phát triển phần mềm trong đó Developer viết test **trước khi viết code** triển khai. Code được viết chỉ nhằm mục đích làm cho test đang fail trở thành pass.

> **Định nghĩa cốt lõi:** TDD không phải là kỹ thuật kiểm thử — đây là kỹ thuật **thiết kế phần mềm** sử dụng test như công cụ dẫn đường.

### 4.2.2 Chu trình Red — Green — Refactor

TDD hoạt động theo một chu trình ngắn, lặp đi lặp lại liên tục, gọi là **Red-Green-Refactor**:

```
        ┌─────────────────────────────────────────┐
        │                                         │
        ▼                                         │
    🔴 RED                                        │
    Viết một test nhỏ mô tả                      │
    hành vi mong muốn.                            │
    Test PHẢI fail (vì chưa có code).             │
        │                                         │
        ▼                                         │
    🟢 GREEN                                      │
    Viết lượng code ÍT NHẤT có thể               │
    để test vừa viết trở thành pass.             │
    Không cần viết đẹp, chỉ cần pass.            │
        │                                         │
        ▼                                         │
    🔵 REFACTOR                                   │
    Cải thiện code (và test) mà                  │
    không làm thay đổi hành vi.                  │
    Tất cả test vẫn phải pass sau refactor.      │
        │                                         │
        └─────────────────────────────────────────┘
                  (Lặp lại với feature tiếp theo)
```

**Nguyên tắc quan trọng:**
- Mỗi vòng lặp rất ngắn — 2 đến 10 phút
- Chỉ viết đủ code để pass test đang fail, không viết thêm
- Không bỏ qua bước Refactor — đây là nơi code trở nên sạch và bền vững

---

### 4.2.3 Ví dụ thực hành TDD từ đầu đến cuối

**Bài toán:** Xây dựng class `ShoppingCart` với tính năng tính tổng giá trị giỏ hàng sau khi áp dụng giảm giá.

**Yêu cầu:**
- Tính tổng giá các sản phẩm trong giỏ
- Áp dụng mã giảm giá theo phần trăm (0-100%)
- Không cho phép giảm giá âm hoặc trên 100%
- Giỏ hàng trống trả về 0

---

**Bước 1 🔴 — Viết test đầu tiên (giỏ hàng trống):**

```python
# test_shopping_cart.py
import pytest
from shopping_cart import ShoppingCart  # File này chưa tồn tại

def test_empty_cart_returns_zero():
    cart = ShoppingCart()
    assert cart.total() == 0
```

Chạy test → **FAIL** với `ModuleNotFoundError: No module named 'shopping_cart'`
Đây là trạng thái Red mong đợi.

---

**Bước 2 🟢 — Viết code tối thiểu để pass:**

```python
# shopping_cart.py
class ShoppingCart:
    def total(self):
        return 0
```

Chạy test → **PASS** ✅
Code này trông buồn cười — nhưng đúng TDD. Chưa có gì để refactor.

---

**Bước 3 🔴 — Test tiếp theo (thêm sản phẩm):**

```python
def test_cart_with_one_item():
    cart = ShoppingCart()
    cart.add_item(name="Áo thun", price=200_000, quantity=1)
    assert cart.total() == 200_000

def test_cart_with_multiple_items():
    cart = ShoppingCart()
    cart.add_item(name="Áo thun", price=200_000, quantity=2)
    cart.add_item(name="Quần jeans", price=500_000, quantity=1)
    assert cart.total() == 900_000  # 200k*2 + 500k*1
```

Chạy → **FAIL** (`AttributeError: 'ShoppingCart' object has no attribute 'add_item'`)

---

**Bước 4 🟢 — Mở rộng code:**

```python
class ShoppingCart:
    def __init__(self):
        self.items = []

    def add_item(self, name: str, price: float, quantity: int):
        self.items.append({"name": name, "price": price, "quantity": quantity})

    def total(self) -> float:
        return sum(item["price"] * item["quantity"] for item in self.items)
```

Chạy tất cả test → **PASS** ✅

---

**Bước 5 🔴 — Test áp dụng giảm giá:**

```python
def test_apply_valid_discount():
    cart = ShoppingCart()
    cart.add_item(name="Áo thun", price=200_000, quantity=1)
    assert cart.total_with_discount(20) == 160_000  # giảm 20%

def test_apply_zero_discount():
    cart = ShoppingCart()
    cart.add_item(name="Áo thun", price=200_000, quantity=1)
    assert cart.total_with_discount(0) == 200_000

def test_apply_full_discount():
    cart = ShoppingCart()
    cart.add_item(name="Áo thun", price=200_000, quantity=1)
    assert cart.total_with_discount(100) == 0

def test_negative_discount_raises_error():
    cart = ShoppingCart()
    cart.add_item(name="Áo thun", price=200_000, quantity=1)
    with pytest.raises(ValueError, match="Discount must be between 0 and 100"):
        cart.total_with_discount(-5)

def test_discount_over_100_raises_error():
    cart = ShoppingCart()
    cart.add_item(name="Áo thun", price=200_000, quantity=1)
    with pytest.raises(ValueError):
        cart.total_with_discount(150)
```

---

**Bước 6 🟢 — Implement:**

```python
def total_with_discount(self, discount_percent: float) -> float:
    if discount_percent < 0 or discount_percent > 100:
        raise ValueError("Discount must be between 0 and 100")
    return self.total() * (1 - discount_percent / 100)
```

Chạy tất cả test → **PASS** ✅

---

**Bước 7 🔵 — Refactor:**

Nhìn lại code, `items` là list of dict — nên dùng dataclass cho rõ ràng hơn:

```python
from dataclasses import dataclass
from typing import List

@dataclass
class CartItem:
    name: str
    price: float
    quantity: int

    @property
    def subtotal(self) -> float:
        return self.price * self.quantity


class ShoppingCart:
    def __init__(self):
        self._items: List[CartItem] = []

    def add_item(self, name: str, price: float, quantity: int) -> None:
        self._items.append(CartItem(name=name, price=price, quantity=quantity))

    def total(self) -> float:
        return sum(item.subtotal for item in self._items)

    def total_with_discount(self, discount_percent: float) -> float:
        if not (0 <= discount_percent <= 100):
            raise ValueError("Discount must be between 0 and 100")
        return self.total() * (1 - discount_percent / 100)

    def item_count(self) -> int:
        return len(self._items)
```

Chạy lại toàn bộ test → **PASS** ✅ Code sạch hơn, không thay đổi hành vi.

---

### 4.2.4 Lợi ích và hạn chế của TDD

**Lợi ích:**

- **Thiết kế tốt hơn:** Khi phải viết test trước, Developer buộc phải nghĩ về API và interface của code từ góc độ người dùng — kết quả là code dễ dùng và ít phụ thuộc chặt hơn.
- **Tài liệu sống (Living documentation):** Bộ test case mô tả chính xác hành vi của hệ thống, luôn được cập nhật.
- **Tự tin khi refactor:** Có test bao phủ đầy đủ, Developer refactor mà không sợ phá hỏng chức năng.
- **Phát hiện lỗi sớm:** Lỗi được bắt ngay trong vòng phát triển, không phải sau nhiều ngày.
- **Giảm debugging:** Khi test fail, phạm vi code cần điều tra rất nhỏ.

**Hạn chế:**

- **Học curve cao:** TDD đòi hỏi tư duy hoàn toàn khác, cần thực hành nhiều mới quen.
- **Chậm hơn ban đầu:** Viết test trước mất thêm thời gian. Nhưng thường tiết kiệm thời gian tổng thể do giảm bug và debugging.
- **Không phù hợp mọi tình huống:** UI phức tạp, code thử nghiệm prototype, code tích hợp third-party khó áp dụng TDD thuần túy.
- **Test có thể sai:** Nếu hiểu sai yêu cầu, viết test sai → code sai nhưng test vẫn pass.

### 4.2.5 Khi nào nên áp dụng TDD

**Nên áp dụng:**
- Business logic phức tạp (tính toán, quy tắc nghiệp vụ, validation)
- Các hàm/class có thể test độc lập
- Code sẽ được nhiều người dùng và thay đổi thường xuyên
- Khi yêu cầu rõ ràng

**Không nên hoặc khó áp dụng:**
- Code UI phức tạp (dùng kết hợp với Cypress/Playwright thay vì unit test)
- Prototype, spike để thử nghiệm công nghệ
- Tích hợp third-party API (nên mock và test ở tầng khác)

---

## 4.3 BDD — Behavior-driven Development

### 4.3.1 BDD là gì và tại sao ra đời?

**BDD (Behavior-driven Development)** được Dan North giới thiệu vào năm 2003 như một sự phát triển của TDD. BDD ra đời để giải quyết một vấn đề cốt lõi: **khoảng cách giao tiếp giữa business và kỹ thuật**.

Trong TDD, test được viết bằng code kỹ thuật mà BA và Product Owner không đọc được. Kết quả là Tester viết test dựa trên hiểu biết của mình, Developer viết code dựa trên hiểu biết của mình, và khi hai bên không đồng thuận về yêu cầu, lỗi phát sinh.

**BDD giải quyết bằng cách:**
- Dùng ngôn ngữ tự nhiên (Gherkin) để mô tả hành vi hệ thống
- BA, Developer, và Tester cùng viết và đọc được test
- Tạo ra một "ngôn ngữ chung" (Ubiquitous Language) cho toàn team

> **Bản chất BDD:** BDD là phương pháp cộng tác, không chỉ là công cụ kiểm thử. Giá trị lớn nhất không phải ở automation mà ở **cuộc trò chuyện** diễn ra khi viết Gherkin.

---

### 4.3.2 Gherkin — Ngôn ngữ đặc tả hành vi

**Gherkin** là ngôn ngữ đặc tả hành vi phần mềm dưới dạng văn bản tự nhiên có cấu trúc. Gherkin được thiết kế để cả người kỹ thuật lẫn phi kỹ thuật đều đọc và viết được.

**Cú pháp cơ bản:**

```gherkin
Feature: Tên tính năng
  Mô tả ngắn về tính năng (optional)

  Background:            # Các bước chung cho tất cả scenario
    Given ...
    And ...

  Scenario: Tên kịch bản cụ thể
    Given [ngữ cảnh ban đầu]
    When  [hành động người dùng thực hiện]
    Then  [kết quả mong đợi]
    And   [kết quả bổ sung]
    But   [ngoại lệ]
```

**Giải thích từng từ khóa:**

| Từ khóa | Ý nghĩa | Mô tả |
|---|---|---|
| `Feature` | Tính năng | Nhóm các scenario liên quan, mô tả tính năng lớn |
| `Scenario` | Kịch bản | Một tình huống cụ thể cần kiểm thử |
| `Given` | Ngữ cảnh | Trạng thái ban đầu của hệ thống trước khi hành động |
| `When` | Hành động | Điều người dùng hoặc hệ thống thực hiện |
| `Then` | Kết quả | Kết quả mong đợi sau hành động |
| `And` | Nối tiếp | Tiếp tục Given/When/Then trước đó |
| `But` | Ngoại lệ | Kết quả phủ định, thường dùng sau `Then` |
| `Background` | Nền chung | Bước Given chung cho mọi scenario trong Feature |
| `Scenario Outline` | Kịch bản tham số | Scenario chạy với nhiều bộ dữ liệu khác nhau |
| `Examples` | Bảng dữ liệu | Bảng dữ liệu cho Scenario Outline |

---

### 4.3.3 Ví dụ thực hành Gherkin — Tính năng Login

```gherkin
Feature: Đăng nhập vào hệ thống
  Là người dùng đã đăng ký,
  Tôi muốn đăng nhập vào hệ thống
  Để truy cập vào tài khoản cá nhân của mình.

  Background:
    Given tôi đang ở trang đăng nhập
    And tài khoản "user@example.com" đã được đăng ký và kích hoạt

  Scenario: Đăng nhập thành công với thông tin hợp lệ
    When tôi nhập email "user@example.com"
    And tôi nhập password "SecurePass@123"
    And tôi click nút "Đăng nhập"
    Then tôi được chuyển đến trang Dashboard
    And tôi thấy tên "Nguyễn Văn A" ở góc trên phải

  Scenario: Đăng nhập thất bại với password sai
    When tôi nhập email "user@example.com"
    And tôi nhập password "WrongPassword"
    And tôi click nút "Đăng nhập"
    Then tôi thấy thông báo lỗi "Email hoặc mật khẩu không đúng"
    And tôi vẫn ở trang đăng nhập
    But tôi không thấy thông tin tài khoản

  Scenario: Tài khoản bị khóa sau 5 lần sai
    When tôi nhập sai password 5 lần liên tiếp với email "user@example.com"
    Then tôi thấy thông báo "Tài khoản đã bị tạm khóa trong 30 phút"
    And tôi không thể đăng nhập dù nhập đúng password

  Scenario Outline: Validation trường bắt buộc
    When tôi nhập email "<email>"
    And tôi nhập password "<password>"
    And tôi click nút "Đăng nhập"
    Then tôi thấy thông báo lỗi "<thong_bao>"

    Examples:
      | email              | password      | thong_bao                    |
      |                    | SecurePass@123| Vui lòng nhập email          |
      | user@example.com   |               | Vui lòng nhập mật khẩu      |
      | invalid-email      | SecurePass@123| Email không đúng định dạng  |
      |                    |               | Vui lòng nhập email          |
```

---

### 4.3.4 Ví dụ thực hành Gherkin — Tính năng Giỏ hàng

```gherkin
Feature: Quản lý giỏ hàng
  Là người mua hàng,
  Tôi muốn quản lý giỏ hàng của mình
  Để kiểm soát những sản phẩm tôi muốn mua.

  Background:
    Given tôi đã đăng nhập với tài khoản "buyer@example.com"
    And giỏ hàng của tôi đang trống

  Scenario: Thêm sản phẩm vào giỏ hàng
    Given sản phẩm "Áo thun Basic" có giá 200,000đ và còn hàng
    When tôi vào trang sản phẩm "Áo thun Basic"
    And tôi chọn size "M"
    And tôi click nút "Thêm vào giỏ"
    Then giỏ hàng hiển thị 1 sản phẩm
    And icon giỏ hàng ở header hiển thị số "1"
    And tổng giá trị giỏ hàng là 200,000đ

  Scenario: Tăng số lượng sản phẩm trong giỏ
    Given giỏ hàng đang có "Áo thun Basic" với số lượng 1
    When tôi tăng số lượng lên 3
    Then giỏ hàng hiển thị số lượng 3
    And tổng giá trị giỏ hàng là 600,000đ

  Scenario: Xóa sản phẩm khỏi giỏ hàng
    Given giỏ hàng đang có "Áo thun Basic" và "Quần jeans"
    When tôi click icon xóa bên cạnh "Áo thun Basic"
    Then giỏ hàng chỉ còn "Quần jeans"
    And tổng tiền chỉ tính giá "Quần jeans"

  Scenario: Không thể thêm sản phẩm hết hàng
    Given sản phẩm "Giày limited" đang hết hàng
    When tôi vào trang sản phẩm "Giày limited"
    Then nút "Thêm vào giỏ" bị vô hiệu hóa
    And hiển thị thông báo "Sản phẩm tạm hết hàng"

  Scenario: Áp dụng mã giảm giá hợp lệ
    Given giỏ hàng có "Áo thun Basic" giá 200,000đ
    And mã giảm giá "SUMMER20" có hiệu lực và giảm 20%
    When tôi nhập mã "SUMMER20" vào ô mã giảm giá
    And tôi click "Áp dụng"
    Then hiển thị dòng "Giảm giá SUMMER20: -40,000đ"
    And tổng thanh toán là 160,000đ
```

---

### 4.3.5 Triển khai BDD với Cucumber (Java) và Behave (Python)

Gherkin chỉ là đặc tả — để tự động hóa, cần framework kết nối từng step trong Gherkin với code thực thi. Hai framework phổ biến nhất là **Cucumber** (Java/JavaScript) và **Behave** (Python).

**Ví dụ triển khai với Behave (Python):**

Cấu trúc thư mục:
```
project/
├── features/
│   ├── login.feature        ← File Gherkin
│   └── steps/
│       └── login_steps.py   ← Step definitions
└── behave.ini
```

```python
# features/steps/login_steps.py
from behave import given, when, then
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

@given('tôi đang ở trang đăng nhập')
def step_open_login_page(context):
    context.driver = webdriver.Chrome()
    context.driver.get("https://app.example.com/login")
    context.wait = WebDriverWait(context.driver, 10)

@given('tài khoản "{email}" đã được đăng ký và kích hoạt')
def step_account_exists(context, email):
    # Trong môi trường test, assume tài khoản đã được setup sẵn
    # Hoặc gọi API để tạo test account
    pass

@when('tôi nhập email "{email}"')
def step_enter_email(context, email):
    email_field = context.driver.find_element(By.ID, "email")
    email_field.clear()
    email_field.send_keys(email)

@when('tôi nhập password "{password}"')
def step_enter_password(context, password):
    password_field = context.driver.find_element(By.ID, "password")
    password_field.clear()
    password_field.send_keys(password)

@when('tôi click nút "{button_text}"')
def step_click_button(context, button_text):
    button = context.driver.find_element(
        By.XPATH, f"//button[contains(text(), '{button_text}')]"
    )
    button.click()

@then('tôi được chuyển đến trang Dashboard')
def step_redirected_to_dashboard(context):
    context.wait.until(EC.url_contains("/dashboard"))
    assert "/dashboard" in context.driver.current_url

@then('tôi thấy tên "{name}" ở góc trên phải')
def step_see_username(context, name):
    user_display = context.wait.until(
        EC.presence_of_element_located((By.CSS_SELECTOR, ".user-display-name"))
    )
    assert name in user_display.text

@then('tôi thấy thông báo lỗi "{message}"')
def step_see_error_message(context, message):
    error_element = context.wait.until(
        EC.presence_of_element_located((By.CSS_SELECTOR, ".error-message"))
    )
    assert message in error_element.text

@then('tôi vẫn ở trang đăng nhập')
def step_still_on_login_page(context):
    assert "/login" in context.driver.current_url
```

**Chạy Behave:**
```bash
# Cài đặt
pip install behave selenium

# Chạy tất cả feature
behave

# Chạy feature cụ thể
behave features/login.feature

# Chạy scenario cụ thể (theo tag)
behave --tags @smoke

# Tạo báo cáo HTML
behave --format html --outfile reports/report.html
```

---

### 4.3.6 Playwright với BDD (JavaScript)

```bash
# Cài đặt
npm install @playwright/test
npm install @cucumber/cucumber
npm install @badeball/cypress-cucumber-preprocessor  # nếu dùng Cypress
```

```javascript
// features/steps/login.steps.ts
import { Given, When, Then } from '@cucumber/cucumber';
import { expect } from '@playwright/test';
import { chromium, Browser, Page } from 'playwright';

let browser: Browser;
let page: Page;

Given('tôi đang ở trang đăng nhập', async () => {
    browser = await chromium.launch();
    page = await browser.newPage();
    await page.goto('https://app.example.com/login');
});

When('tôi nhập email {string}', async (email: string) => {
    await page.fill('#email', email);
});

When('tôi nhập password {string}', async (password: string) => {
    await page.fill('#password', password);
});

When('tôi click nút {string}', async (buttonText: string) => {
    await page.click(`button:has-text("${buttonText}")`);
});

Then('tôi được chuyển đến trang Dashboard', async () => {
    await page.waitForURL('**/dashboard');
    expect(page.url()).toContain('/dashboard');
});

Then('tôi thấy tên {string} ở góc trên phải', async (name: string) => {
    const userDisplay = page.locator('.user-display-name');
    await expect(userDisplay).toContainText(name);
});

Then('tôi thấy thông báo lỗi {string}', async (message: string) => {
    const errorMsg = page.locator('.error-message');
    await expect(errorMsg).toBeVisible();
    await expect(errorMsg).toContainText(message);
});
```

---

### 4.3.7 TDD vs BDD — Khi nào dùng cái nào?

| Tiêu chí | TDD | BDD |
|---|---|---|
| Người viết test | Developer | BA + Developer + Tester cùng nhau |
| Ngôn ngữ | Code (Python, Java, JS) | Gherkin (ngôn ngữ tự nhiên) |
| Tầng kiểm thử | Unit, Integration | Acceptance, E2E |
| Góc nhìn | Kỹ thuật (hàm, class, method) | Nghiệp vụ (user story, behavior) |
| Tài liệu | Test code | Feature file là tài liệu |
| Độ phức tạp | Thấp hơn | Cao hơn (cần setup nhiều hơn) |
| Phù hợp | Business logic, algorithm | User-facing features, acceptance criteria |

**Trong thực tế:** TDD và BDD không phải là lựa chọn hoặc/hoặc — chúng bổ trợ nhau:
- TDD ở tầng Unit: Developer dùng khi implement logic
- BDD ở tầng Acceptance: Toàn team dùng để định nghĩa hành vi từ góc nhìn người dùng

**Dùng BDD khi:**
- Feature có nhiều stakeholder cần cùng hiểu
- Yêu cầu phức tạp cần được làm rõ qua ví dụ cụ thể
- Team muốn Acceptance Criteria tự động hóa được

**Dùng TDD khi:**
- Implement business logic phức tạp
- Refactor code cũ cần có safety net
- Developer muốn thiết kế tốt hơn thông qua test-first

---

### 4.3.8 Quy trình Three Amigos áp dụng BDD

Ba Amigos (BA, Developer, Tester) ngồi lại, dùng Example Mapping để khám phá yêu cầu:

**Example Mapping là gì:**
Kỹ thuật workshop ngắn (30-60 phút) sử dụng thẻ màu để tổ chức thảo luận:
- **Thẻ vàng:** User Story
- **Thẻ xanh:** Rule (quy tắc nghiệp vụ)
- **Thẻ xanh lam:** Example (ví dụ cụ thể cho mỗi rule — đây là Gherkin Scenario)
- **Thẻ đỏ:** Question (câu hỏi chưa có câu trả lời)

```
User Story: Áp dụng mã giảm giá khi checkout

Rule 1: Mã giảm giá phải còn hiệu lực
  Example: Mã SUMMER20 còn hạn → áp dụng thành công
  Example: Mã WINTER10 hết hạn → hiển thị lỗi "Mã đã hết hạn"

Rule 2: Mỗi đơn hàng chỉ dùng được 1 mã giảm giá
  Example: Nhập mã thứ 2 khi đã có mã → hỏi có muốn thay thế không

Rule 3: Mã giảm giá có số lần sử dụng giới hạn
  Example: Mã đã được dùng đủ số lần → "Mã đã đạt giới hạn sử dụng"

❓ Question: Mã giảm giá có áp dụng được cho sản phẩm sale không?
   → Cần PO xác nhận trước khi viết Gherkin
```

Kết thúc session, các Example đã thống nhất được viết thành Gherkin Scenario — đây chính là Acceptance Criteria tự động hóa được.
