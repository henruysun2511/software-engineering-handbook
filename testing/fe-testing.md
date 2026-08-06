# Frontend Testing

Testing là quá trình kiểm tra phần mềm hoạt động đúng như kỳ vọng. Một chiến lược testing tốt giúp phát hiện lỗi sớm, tự tin refactor code, và đảm bảo tính ổn định khi ứng dụng phát triển theo thời gian.

---

## Tại sao cần Testing?

| Không có test | Có test |
|---|---|
| Sợ refactor — có thể làm hỏng thứ khác | Tự tin thay đổi code |
| Bug phát hiện muộn (production) | Bug phát hiện sớm (development) |
| Review code mất thời gian | Test là tài liệu sống cho code |
| Deploy phụ thuộc vào test thủ công | CI/CD tự động chạy test |

---

## Testing Pyramid

Testing được tổ chức theo mô hình kim tự tháp — càng lên cao, test càng chậm, tốn kém và ít số lượng hơn:

```
         /‾‾‾‾‾‾‾‾‾‾‾‾‾\
        /   E2E Tests    \       ← Ít nhất, chậm nhất, test luồng người dùng
       /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
      / Integration Tests  \     ← Vừa phải, test nhiều component cùng nhau
     /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
    /      Unit Tests        \   ← Nhiều nhất, nhanh nhất, test từng đơn vị nhỏ
   /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
```

| Loại | Công cụ | Tốc độ | Mục đích |
|---|---|---|---|
| **Unit Test** | Jest / Vitest | Nhanh (ms) | Test hàm, hook, component đơn lẻ |
| **Integration Test** | React Testing Library | Trung bình (giây) | Test nhiều component phối hợp |
| **E2E Test** | Playwright | Chậm (giây–phút) | Test luồng người dùng trên trình duyệt thật |

---

## 14.1. Unit Test với Vitest

### Vitest là gì?

Vitest là test runner hiện đại, tương thích với API của Jest nhưng tích hợp sẵn với Vite — nhanh hơn đáng kể và không cần cấu hình phức tạp. Là lựa chọn mặc định cho các dự án Vite và Next.js mới.

```bash
npm install -D vitest @vitest/ui
```

```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "jsdom",  // Giả lập DOM
    globals: true,         // Dùng describe/it/expect mà không cần import
    setupFiles: ["./src/test/setup.ts"],
  },
});
```

### Cú pháp cơ bản

```typescript
import { describe, it, expect, beforeEach, vi } from "vitest";

describe("Tên nhóm test", () => {
  beforeEach(() => {
    // Chạy trước mỗi test trong nhóm — reset state
  });

  it("mô tả điều test này kiểm tra", () => {
    // Arrange — chuẩn bị
    // Act — thực thi
    // Assert — kiểm tra kết quả
  });
});
```

### Test hàm thuần túy

Hàm thuần túy (pure function) là loại dễ test nhất — không có side effect, không phụ thuộc state ngoài.

```typescript
// utils/format.ts
export function formatCurrency(amount: number, currency = "VND"): string {
  return new Intl.NumberFormat("vi-VN", {
    style: "currency",
    currency,
  }).format(amount);
}

export function truncateText(text: string, maxLength: number): string {
  if (text.length <= maxLength) return text;
  return text.slice(0, maxLength).trimEnd() + "...";
}
```

```typescript
// utils/format.test.ts
import { describe, it, expect } from "vitest";
import { formatCurrency, truncateText } from "./format";

describe("formatCurrency", () => {
  it("định dạng số thành tiền VND", () => {
    expect(formatCurrency(1000000)).toBe("1.000.000 ₫");
  });

  it("hỗ trợ currency khác", () => {
    expect(formatCurrency(100, "USD")).toContain("100");
  });
});

describe("truncateText", () => {
  it("giữ nguyên text nếu đủ ngắn", () => {
    expect(truncateText("Xin chào", 20)).toBe("Xin chào");
  });

  it("cắt text và thêm dấu ... khi quá dài", () => {
    const result = truncateText("Đây là một đoạn văn rất dài", 10);
    expect(result).toHaveLength(13); // 10 ký tự + "..."
    expect(result.endsWith("...")).toBe(true);
  });

  it("cắt đúng tại maxLength", () => {
    expect(truncateText("abcdefghij", 5)).toBe("abcde...");
  });
});
```

### Test với Mock

Mock cho phép thay thế các dependency (API call, module ngoài) bằng giá trị giả — test chạy nhanh, không phụ thuộc network.

```typescript
// services/userService.ts
export async function fetchUser(id: number): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error("User not found");
  return res.json();
}
```

```typescript
// services/userService.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { fetchUser } from "./userService";

// Mock global fetch
const mockFetch = vi.fn();
global.fetch = mockFetch;

describe("fetchUser", () => {
  beforeEach(() => {
    mockFetch.mockClear(); // Reset mock trước mỗi test
  });

  it("trả về user khi request thành công", async () => {
    const mockUser = { id: 1, name: "An", email: "an@example.com" };

    mockFetch.mockResolvedValueOnce({
      ok: true,
      json: async () => mockUser,
    });

    const user = await fetchUser(1);

    expect(user).toEqual(mockUser);
    expect(mockFetch).toHaveBeenCalledWith("/api/users/1");
    expect(mockFetch).toHaveBeenCalledTimes(1);
  });

  it("throw lỗi khi response không ok", async () => {
    mockFetch.mockResolvedValueOnce({ ok: false });

    await expect(fetchUser(999)).rejects.toThrow("User not found");
  });
});
```

### Test Custom Hook

Custom hook không thể gọi trực tiếp — cần `renderHook` từ React Testing Library:

```typescript
// hooks/useCounter.ts
import { useState } from "react";

export function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);
  return {
    count,
    increment: () => setCount((c) => c + 1),
    decrement: () => setCount((c) => c - 1),
    reset: () => setCount(initialValue),
  };
}
```

```typescript
// hooks/useCounter.test.ts
import { describe, it, expect } from "vitest";
import { renderHook, act } from "@testing-library/react";
import { useCounter } from "./useCounter";

describe("useCounter", () => {
  it("khởi tạo với giá trị mặc định 0", () => {
    const { result } = renderHook(() => useCounter());
    expect(result.current.count).toBe(0);
  });

  it("khởi tạo với giá trị tùy chỉnh", () => {
    const { result } = renderHook(() => useCounter(10));
    expect(result.current.count).toBe(10);
  });

  it("tăng count khi gọi increment", () => {
    const { result } = renderHook(() => useCounter());

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  it("reset về giá trị ban đầu", () => {
    const { result } = renderHook(() => useCounter(5));

    act(() => {
      result.current.increment();
      result.current.increment();
      result.current.reset();
    });

    expect(result.current.count).toBe(5);
  });
});
```

---

## 14.2. Integration Test với React Testing Library

### React Testing Library là gì?

React Testing Library (RTL) là thư viện test component React theo cách **người dùng tương tác** — không quan tâm đến implementation detail (state nội bộ, method), mà kiểm tra những gì thực sự hiển thị trên màn hình.

> **Triết lý cốt lõi:** *"The more your tests resemble the way your software is used, the more confidence they can give you."*

```bash
npm install -D @testing-library/react @testing-library/user-event @testing-library/jest-dom
```

```typescript
// src/test/setup.ts
import "@testing-library/jest-dom"; // Thêm matchers: toBeInTheDocument, toHaveValue...
```

### Các query phổ biến

RTL cung cấp nhiều cách tìm phần tử — ưu tiên theo thứ tự sau (từ tốt nhất đến kém nhất):

| Query | Dùng khi | Ví dụ |
|---|---|---|
| `getByRole` | Phần tử có semantic role | `getByRole("button", { name: "Gửi" })` |
| `getByLabelText` | Input gắn với label | `getByLabelText("Email")` |
| `getByPlaceholderText` | Input có placeholder | `getByPlaceholderText("Nhập email")` |
| `getByText` | Phần tử theo nội dung text | `getByText("Xin chào")` |
| `getByTestId` | Khi không có cách nào khác | `getByTestId("user-avatar")` |

`getBy*` throw nếu không tìm thấy. `queryBy*` trả về `null`. `findBy*` là async, chờ phần tử xuất hiện.

### Test component cơ bản

```tsx
// components/Button/Button.tsx
interface ButtonProps {
  label: string;
  onClick: () => void;
  disabled?: boolean;
  variant?: "primary" | "danger";
}

export function Button({ label, onClick, disabled, variant = "primary" }: ButtonProps) {
  return (
    <button
      onClick={onClick}
      disabled={disabled}
      className={`btn btn--${variant}`}
    >
      {label}
    </button>
  );
}
```

```typescript
// components/Button/Button.test.tsx
import { describe, it, expect, vi } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { Button } from "./Button";

describe("Button", () => {
  it("hiển thị label đúng", () => {
    render(<Button label="Lưu" onClick={vi.fn()} />);
    expect(screen.getByRole("button", { name: "Lưu" })).toBeInTheDocument();
  });

  it("gọi onClick khi click", async () => {
    const handleClick = vi.fn();
    render(<Button label="Gửi" onClick={handleClick} />);

    await userEvent.click(screen.getByRole("button", { name: "Gửi" }));

    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it("không gọi onClick khi disabled", async () => {
    const handleClick = vi.fn();
    render(<Button label="Gửi" onClick={handleClick} disabled />);

    await userEvent.click(screen.getByRole("button", { name: "Gửi" }));

    expect(handleClick).not.toHaveBeenCalled();
  });
});
```

### Test form với validation

```tsx
// components/LoginForm/LoginForm.tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schema = z.object({
  email: z.string().email("Email không hợp lệ"),
  password: z.string().min(8, "Tối thiểu 8 ký tự"),
});

type FormValues = z.infer<typeof schema>;

interface LoginFormProps {
  onSubmit: (data: FormValues) => Promise<void>;
}

export function LoginForm({ onSubmit }: LoginFormProps) {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<FormValues>({ resolver: zodResolver(schema) });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <label htmlFor="email">Email</label>
        <input id="email" type="email" {...register("email")} />
        {errors.email && <p role="alert">{errors.email.message}</p>}
      </div>

      <div>
        <label htmlFor="password">Mật khẩu</label>
        <input id="password" type="password" {...register("password")} />
        {errors.password && <p role="alert">{errors.password.message}</p>}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Đang đăng nhập..." : "Đăng nhập"}
      </button>
    </form>
  );
}
```

```typescript
// components/LoginForm/LoginForm.test.tsx
import { describe, it, expect, vi } from "vitest";
import { render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { LoginForm } from "./LoginForm";

describe("LoginForm", () => {
  it("hiển thị lỗi khi email không hợp lệ", async () => {
    render(<LoginForm onSubmit={vi.fn()} />);

    await userEvent.type(screen.getByLabelText("Email"), "not-an-email");
    await userEvent.click(screen.getByRole("button", { name: "Đăng nhập" }));

    expect(await screen.findByRole("alert")).toHaveTextContent(
      "Email không hợp lệ"
    );
  });

  it("hiển thị lỗi khi password quá ngắn", async () => {
    render(<LoginForm onSubmit={vi.fn()} />);

    await userEvent.type(screen.getByLabelText("Mật khẩu"), "abc");
    await userEvent.click(screen.getByRole("button", { name: "Đăng nhập" }));

    expect(await screen.findByRole("alert")).toHaveTextContent(
      "Tối thiểu 8 ký tự"
    );
  });

  it("gọi onSubmit với dữ liệu đúng khi form hợp lệ", async () => {
    const handleSubmit = vi.fn().mockResolvedValue(undefined);
    render(<LoginForm onSubmit={handleSubmit} />);

    await userEvent.type(screen.getByLabelText("Email"), "an@example.com");
    await userEvent.type(screen.getByLabelText("Mật khẩu"), "password123");
    await userEvent.click(screen.getByRole("button", { name: "Đăng nhập" }));

    await waitFor(() => {
      expect(handleSubmit).toHaveBeenCalledWith({
        email: "an@example.com",
        password: "password123",
      });
    });
  });

  it("disable nút submit khi đang gửi", async () => {
    // onSubmit không resolve ngay — giả lập đang loading
    const handleSubmit = vi.fn(() => new Promise(() => {}));
    render(<LoginForm onSubmit={handleSubmit} />);

    await userEvent.type(screen.getByLabelText("Email"), "an@example.com");
    await userEvent.type(screen.getByLabelText("Mật khẩu"), "password123");
    await userEvent.click(screen.getByRole("button", { name: "Đăng nhập" }));

    await waitFor(() => {
      expect(screen.getByRole("button", { name: "Đang đăng nhập..." }))
        .toBeDisabled();
    });
  });
});
```

### Test với Mock API (MSW)

**Mock Service Worker (MSW)** cho phép mock API ở tầng network — không cần mock `fetch` hay `axios` thủ công, test gần với thực tế hơn.

```bash
npm install -D msw
```

```typescript
// src/test/handlers.ts
import { http, HttpResponse } from "msw";

export const handlers = [
  http.get("/api/users", () => {
    return HttpResponse.json([
      { id: 1, name: "An", email: "an@example.com" },
      { id: 2, name: "Bình", email: "binh@example.com" },
    ]);
  }),

  http.post("/api/users", async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: 3, ...body }, { status: 201 });
  }),
];
```

```typescript
// src/test/setup.ts
import "@testing-library/jest-dom";
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

const server = setupServer(...handlers);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers()); // Reset sau mỗi test
afterAll(() => server.close());
```

```typescript
// components/UserList/UserList.test.tsx
import { describe, it, expect } from "vitest";
import { render, screen } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { UserList } from "./UserList";

function renderWithProviders(ui: React.ReactElement) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });
  return render(
    <QueryClientProvider client={queryClient}>{ui}</QueryClientProvider>
  );
}

describe("UserList", () => {
  it("hiển thị danh sách user sau khi tải xong", async () => {
    renderWithProviders(<UserList />);

    // Loading state
    expect(screen.getByText("Đang tải...")).toBeInTheDocument();

    // Sau khi data load — dùng findBy* để chờ
    expect(await screen.findByText("An")).toBeInTheDocument();
    expect(await screen.findByText("Bình")).toBeInTheDocument();
  });
});
```

---

## 14.3. E2E Test với Playwright

### Playwright là gì?

Playwright là công cụ E2E (End-to-End) testing của Microsoft, cho phép điều khiển trình duyệt thật (Chromium, Firefox, WebKit) để test toàn bộ luồng người dùng từ đầu đến cuối — click, nhập liệu, điều hướng, kiểm tra kết quả.

```bash
npm install -D @playwright/test
npx playwright install  # Tải về các trình duyệt
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./e2e",
  timeout: 30000,
  retries: process.env.CI ? 2 : 0,  // Retry khi chạy trên CI
  use: {
    baseURL: "http://localhost:3000",
    screenshot: "only-on-failure",   // Chụp màn hình khi test fail
    video: "retain-on-failure",      // Quay video khi test fail
  },
  projects: [
    { name: "chromium", use: { ...devices["Desktop Chrome"] } },
    { name: "Mobile Safari", use: { ...devices["iPhone 14"] } },
  ],
  webServer: {
    command: "npm run build && npm run start",
    port: 3000,
    reuseExistingServer: !process.env.CI,
  },
});
```

### Cú pháp cơ bản

```typescript
import { test, expect } from "@playwright/test";

test.describe("Tên nhóm test", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("/");  // Điều hướng trước mỗi test
  });

  test("mô tả test case", async ({ page }) => {
    // Thao tác với trang
    await page.click("button");
    await page.fill("input", "text");

    // Kiểm tra kết quả
    await expect(page.getByText("Thành công")).toBeVisible();
  });
});
```

### Test luồng đăng nhập

```typescript
// e2e/auth.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Đăng nhập", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("/login");
  });

  test("đăng nhập thành công với thông tin hợp lệ", async ({ page }) => {
    // Nhập thông tin
    await page.getByLabel("Email").fill("an@example.com");
    await page.getByLabel("Mật khẩu").fill("password123");
    await page.getByRole("button", { name: "Đăng nhập" }).click();

    // Kiểm tra redirect về dashboard
    await expect(page).toHaveURL("/dashboard");
    await expect(page.getByText("Xin chào, An")).toBeVisible();
  });

  test("hiển thị lỗi với mật khẩu sai", async ({ page }) => {
    await page.getByLabel("Email").fill("an@example.com");
    await page.getByLabel("Mật khẩu").fill("wrongpassword");
    await page.getByRole("button", { name: "Đăng nhập" }).click();

    await expect(
      page.getByText("Email hoặc mật khẩu không đúng")
    ).toBeVisible();

    // Vẫn ở trang login
    await expect(page).toHaveURL("/login");
  });

  test("hiển thị validation khi để trống", async ({ page }) => {
    await page.getByRole("button", { name: "Đăng nhập" }).click();

    await expect(page.getByText("Email là bắt buộc")).toBeVisible();
    await expect(page.getByText("Mật khẩu là bắt buộc")).toBeVisible();
  });
});
```

### Test luồng CRUD phức tạp

```typescript
// e2e/products.spec.ts
import { test, expect, Page } from "@playwright/test";

// Hàm helper tái sử dụng — đăng nhập trước khi test
async function loginAsAdmin(page: Page) {
  await page.goto("/login");
  await page.getByLabel("Email").fill("admin@example.com");
  await page.getByLabel("Mật khẩu").fill("admin123");
  await page.getByRole("button", { name: "Đăng nhập" }).click();
  await page.waitForURL("/dashboard");
}

test.describe("Quản lý sản phẩm", () => {
  test.beforeEach(async ({ page }) => {
    await loginAsAdmin(page);
    await page.goto("/dashboard/products");
  });

  test("tạo sản phẩm mới", async ({ page }) => {
    await page.getByRole("button", { name: "Thêm sản phẩm" }).click();

    // Điền form
    await page.getByLabel("Tên sản phẩm").fill("MacBook Pro M3");
    await page.getByLabel("Giá").fill("50000000");
    await page.getByLabel("Mô tả").fill("Laptop cao cấp của Apple");
    await page.getByRole("button", { name: "Lưu" }).click();

    // Kiểm tra thông báo thành công
    await expect(page.getByText("Tạo sản phẩm thành công")).toBeVisible();

    // Kiểm tra sản phẩm xuất hiện trong danh sách
    await expect(page.getByText("MacBook Pro M3")).toBeVisible();
  });

  test("tìm kiếm sản phẩm", async ({ page }) => {
    await page.getByPlaceholder("Tìm kiếm...").fill("MacBook");

    // Chờ kết quả (debounce)
    await page.waitForTimeout(500);

    // Chỉ hiển thị sản phẩm khớp
    const rows = page.getByRole("row");
    await expect(rows).toHaveCount(2); // Header + 1 sản phẩm

    await expect(page.getByText("MacBook Pro M3")).toBeVisible();
  });

  test("xóa sản phẩm", async ({ page }) => {
    // Click nút xóa của sản phẩm đầu tiên
    await page.getByRole("row").nth(1).getByRole("button", { name: "Xóa" }).click();

    // Xác nhận trong dialog
    await expect(page.getByRole("dialog")).toBeVisible();
    await page.getByRole("button", { name: "Xác nhận xóa" }).click();

    // Kiểm tra thông báo
    await expect(page.getByText("Đã xóa sản phẩm")).toBeVisible();
  });
});
```

### Page Object Model (POM)

Khi test phức tạp, Page Object Model giúp tách logic tương tác với trang khỏi logic kiểm tra — dễ bảo trì khi UI thay đổi:

```typescript
// e2e/pages/LoginPage.ts
import { Page, Locator, expect } from "@playwright/test";

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel("Email");
    this.passwordInput = page.getByLabel("Mật khẩu");
    this.submitButton = page.getByRole("button", { name: "Đăng nhập" });
    this.errorMessage = page.getByRole("alert");
  }

  async goto() {
    await this.page.goto("/login");
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async expectError(message: string) {
    await expect(this.errorMessage).toContainText(message);
  }
}

// Dùng trong test — gọn và rõ ràng hơn
test("đăng nhập với thông tin sai", async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login("an@example.com", "wrongpass");
  await loginPage.expectError("Email hoặc mật khẩu không đúng");
});
```

---

## So sánh tổng kết

| | Unit Test (Vitest) | Integration Test (RTL) | E2E Test (Playwright) |
|---|---|---|---|
| Test gì | Hàm, hook, logic đơn lẻ | Component + tương tác người dùng | Luồng nghiệp vụ hoàn chỉnh |
| Tốc độ | Nhanh nhất (ms) | Trung bình (giây) | Chậm nhất (giây–phút) |
| Môi trường | Node.js + jsdom | Node.js + jsdom | Trình duyệt thật |
| Phụ thuộc mạng | Không (mock) | Không (MSW mock) | Có (hoặc mock) |
| Độ tự tin | Thấp–trung bình | Trung bình–cao | Cao nhất |
| Bảo trì | Dễ | Trung bình | Khó nhất |
| Tỷ lệ nên có | ~70% | ~20% | ~10% |

### Nguyên tắc thực tế

- **Viết test cho logic quan trọng trước** — utility function, custom hook, business logic.
- **Ưu tiên `getByRole` và `getByLabelText`** trong RTL — giống cách người dùng và screen reader tìm phần tử.
- **E2E test cho happy path và critical flows** — đăng nhập, thanh toán, luồng chính — không cần test mọi edge case.
- **Chạy test trong CI/CD** — mỗi pull request phải pass test trước khi merge.
- **Tránh test implementation detail** — không test state nội bộ, tên class CSS hay cấu trúc DOM cụ thể — test hành vi như người dùng thấy.