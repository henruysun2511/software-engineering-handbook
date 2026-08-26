# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# CHƯƠNG 9: UNIT TESTING

---

## 9.1 Unit Test là gì?

**Unit Testing** là kiểm thử các đơn vị nhỏ nhất của phần mềm — một hàm, một method, một class — một cách **hoàn toàn cô lập** với phần còn lại của hệ thống.

**Bản chất của Unit Test:**
- Nhanh: chạy trong mili-giây
- Cô lập: không phụ thuộc database, network, filesystem
- Xác định: cùng input, luôn cho cùng output
- Tự mô tả: tên test nói lên hành vi được kiểm thử

---

## 9.2 AAA Pattern — Arrange, Act, Assert

AAA là cấu trúc chuẩn cho mọi unit test, giúp test rõ ràng và dễ đọc.

```typescript
test("Tính tổng giỏ hàng với discount 20%", () => {
    // ARRANGE — Chuẩn bị
    const cart = new ShoppingCart();
    cart.addItem({ name: "Áo thun", price: 200_000, quantity: 2 });
    cart.addItem({ name: "Quần jeans", price: 500_000, quantity: 1 });
    // Total = 200k*2 + 500k*1 = 900k

    // ACT — Thực hiện
    const result = cart.totalWithDiscount(20);

    // ASSERT — Kiểm tra
    expect(result).toBe(720_000);  // 900k * 0.8 = 720k
});
```

---

## 9.3 Test Isolation, Mock, Stub, Spy

### 9.3.1 Test Isolation

**Isolation** nghĩa là test không bị ảnh hưởng bởi:
- Kết quả test trước (test order independent)
- Môi trường bên ngoài (database, API, thời gian hệ thống)
- Các unit khác của hệ thống

### 9.3.2 Stub — Thay thế phụ thuộc đơn giản

**Stub** trả về dữ liệu cố định thay cho phụ thuộc thật, nhưng không quan tâm đến cách nó được gọi.

```typescript
// Hàm cần test
class OrderService {
    constructor(private emailService: EmailService) {}

    async createOrder(data: OrderData): Promise<Order> {
        const order = await this.saveToDatabase(data);
        await this.emailService.sendConfirmation(order);  // phụ thuộc bên ngoài
        return order;
    }
}

// Test với Stub — thay EmailService bằng stub
test("Tạo order thành công", async () => {
    // Arrange
    const emailStub = {
        sendConfirmation: async () => {}  // stub — không làm gì
    };
    const orderService = new OrderService(emailStub as any);

    // Act
    const order = await orderService.createOrder({
        userId: 1,
        items: [{ productId: "P001", quantity: 2 }]
    });

    // Assert
    expect(order.status).toBe("pending");
    expect(order.userId).toBe(1);
});
```

### 9.3.3 Mock — Kiểm tra cả hành vi gọi

**Mock** không chỉ trả về dữ liệu mà còn **xác nhận rằng nó được gọi đúng cách** (số lần gọi, tham số, thứ tự).

```typescript
// Jest mock
import { jest } from "@jest/globals";

test("Email xác nhận được gửi khi tạo order", async () => {
    // Arrange
    const mockEmailService = {
        sendConfirmation: jest.fn().mockResolvedValue(undefined)
    };
    const orderService = new OrderService(mockEmailService as any);

    // Act
    const order = await orderService.createOrder({
        userId: 1,
        items: [{ productId: "P001", quantity: 2 }]
    });

    // Assert — verify mock WAS CALLED correctly
    expect(mockEmailService.sendConfirmation).toHaveBeenCalledTimes(1);
    expect(mockEmailService.sendConfirmation).toHaveBeenCalledWith(
        expect.objectContaining({ userId: 1, status: "pending" })
    );
});
```

### 9.3.4 Spy — Wrap function thật, theo dõi calls

**Spy** "bao bọc" hàm thật — hàm vẫn chạy như bình thường nhưng ghi lại thông tin về cách nó được gọi.

```typescript
test("Logger được gọi khi có lỗi", () => {
    // Arrange
    const logger = { error: console.error };
    const spy = jest.spyOn(logger, "error");  // spy on real function
    const service = new PaymentService(logger);

    // Act
    service.processPayment({ amount: -100 });  // số tiền âm → gây lỗi

    // Assert — hàm thật vẫn chạy, nhưng verify nó được gọi
    expect(spy).toHaveBeenCalledWith(
        expect.stringContaining("Invalid amount")
    );

    spy.mockRestore();  // restore hàm gốc
});
```

**Tóm tắt phân biệt:**

| | Stub | Mock | Spy |
|---|---|---|---|
| Thay thế hàm thật | ✅ | ✅ | ❌ (wrap) |
| Trả dữ liệu giả | ✅ | ✅ | ❌ |
| Verify cách gọi | ❌ | ✅ | ✅ |
| Hàm thật vẫn chạy | ❌ | ❌ | ✅ |

---

## 9.4 Code Coverage

**Code Coverage** đo tỷ lệ code được thực thi bởi test suite.

**Các loại coverage:**

**Line Coverage:** Tỷ lệ dòng code được thực thi
```typescript
function calculateShipping(weight: number, distance: number): number {
    if (weight <= 0) {                    // dòng 1
        throw new Error("Invalid weight"); // dòng 2 — có test không?
    }
    const base = weight * 5000;           // dòng 3
    if (distance > 100) {                 // dòng 4
        return base * 1.5;               // dòng 5 — có test không?
    }
    return base;                          // dòng 6
}
```

**Branch Coverage:** Tỷ lệ các nhánh điều kiện được kiểm thử
```
calculateShipping:
  Branch 1: weight <= 0 → TRUE (ném lỗi) → có test không?
  Branch 2: weight <= 0 → FALSE (tiếp tục) → có test không?
  Branch 3: distance > 100 → TRUE (nhân 1.5) → có test không?
  Branch 4: distance > 100 → FALSE (return base) → có test không?
```

**Chạy coverage với Jest:**
```bash
# Chạy test với coverage report
npx jest --coverage

# Output:
# ------------------|---------|----------|---------|---------|
# File              | % Stmts | % Branch | % Funcs | % Lines |
# ------------------|---------|----------|---------|---------|
# cart.service.ts   |   92.3  |   85.7   |  100    |   91.2  |
# order.service.ts  |   78.5  |   66.7   |   90    |   78.5  |
```

**Coverage threshold (ngưỡng tối thiểu):**
```javascript
// jest.config.js
module.exports = {
    coverageThreshold: {
        global: {
            branches: 80,
            functions: 90,
            lines: 85,
            statements: 85
        }
    }
};
```

---

## 9.5 Ví dụ Unit Testing Hoàn chỉnh

### 9.5.1 Unit Testing Backend — Service Layer (TypeScript/Jest)

```typescript
// cart.service.ts
export class CartService {
    constructor(
        private readonly productRepo: ProductRepository,
        private readonly couponRepo: CouponRepository
    ) {}

    async addItem(userId: number, productId: string, quantity: number): Promise<Cart> {
        if (quantity <= 0) {
            throw new ValidationError("Quantity must be greater than 0");
        }

        const product = await this.productRepo.findById(productId);
        if (!product) {
            throw new NotFoundError(`Product ${productId} not found`);
        }
        if (product.stock < quantity) {
            throw new BusinessError(`Insufficient stock. Available: ${product.stock}`);
        }

        return this.productRepo.addToCart(userId, productId, quantity);
    }

    async applyCoupon(userId: number, couponCode: string): Promise<DiscountResult> {
        const coupon = await this.couponRepo.findByCode(couponCode);
        if (!coupon) {
            throw new NotFoundError("Coupon not found");
        }
        if (coupon.expiresAt < new Date()) {
            throw new BusinessError("Coupon has expired");
        }
        if (coupon.usageCount >= coupon.maxUsage) {
            throw new BusinessError("Coupon usage limit reached");
        }

        return { discountPercent: coupon.discountPercent, couponId: coupon.id };
    }
}
```

```typescript
// cart.service.test.ts
import { CartService } from "./cart.service";
import { ValidationError, NotFoundError, BusinessError } from "./errors";

describe("CartService", () => {
    let cartService: CartService;
    let mockProductRepo: jest.Mocked<ProductRepository>;
    let mockCouponRepo: jest.Mocked<CouponRepository>;

    beforeEach(() => {
        // Reset mocks trước mỗi test
        mockProductRepo = {
            findById: jest.fn(),
            addToCart: jest.fn(),
        } as any;

        mockCouponRepo = {
            findByCode: jest.fn(),
        } as any;

        cartService = new CartService(mockProductRepo, mockCouponRepo);
    });

    // ===== addItem =====
    describe("addItem", () => {
        test("Thêm sản phẩm thành công", async () => {
            // Arrange
            const mockProduct = { id: "P001", name: "Áo thun", price: 200_000, stock: 10 };
            const mockCart = { userId: 1, items: [{ productId: "P001", quantity: 2 }] };
            mockProductRepo.findById.mockResolvedValue(mockProduct);
            mockProductRepo.addToCart.mockResolvedValue(mockCart as any);

            // Act
            const result = await cartService.addItem(1, "P001", 2);

            // Assert
            expect(mockProductRepo.findById).toHaveBeenCalledWith("P001");
            expect(mockProductRepo.addToCart).toHaveBeenCalledWith(1, "P001", 2);
            expect(result).toEqual(mockCart);
        });

        test("Ném ValidationError khi quantity = 0", async () => {
            await expect(cartService.addItem(1, "P001", 0))
                .rejects.toThrow(ValidationError);
            await expect(cartService.addItem(1, "P001", 0))
                .rejects.toThrow("Quantity must be greater than 0");
        });

        test("Ném ValidationError khi quantity âm", async () => {
            await expect(cartService.addItem(1, "P001", -5))
                .rejects.toThrow(ValidationError);
        });

        test("Ném NotFoundError khi sản phẩm không tồn tại", async () => {
            mockProductRepo.findById.mockResolvedValue(null);

            await expect(cartService.addItem(1, "INVALID", 1))
                .rejects.toThrow(NotFoundError);
            await expect(cartService.addItem(1, "INVALID", 1))
                .rejects.toThrow("Product INVALID not found");
        });

        test("Ném BusinessError khi tồn kho không đủ", async () => {
            const mockProduct = { id: "P001", name: "Áo thun", price: 200_000, stock: 3 };
            mockProductRepo.findById.mockResolvedValue(mockProduct);

            await expect(cartService.addItem(1, "P001", 5))
                .rejects.toThrow(BusinessError);
            await expect(cartService.addItem(1, "P001", 5))
                .rejects.toThrow("Insufficient stock. Available: 3");
        });

        test("Thành công khi quantity đúng bằng tồn kho (ranh giới)", async () => {
            const mockProduct = { id: "P001", name: "Áo thun", price: 200_000, stock: 5 };
            mockProductRepo.findById.mockResolvedValue(mockProduct);
            mockProductRepo.addToCart.mockResolvedValue({} as any);

            await expect(cartService.addItem(1, "P001", 5)).resolves.toBeDefined();
        });
    });

    // ===== applyCoupon =====
    describe("applyCoupon", () => {
        test("Áp dụng coupon hợp lệ thành công", async () => {
            const mockCoupon = {
                id: 1, code: "SUMMER20", discountPercent: 20,
                expiresAt: new Date("2099-12-31"),
                usageCount: 5, maxUsage: 100
            };
            mockCouponRepo.findByCode.mockResolvedValue(mockCoupon);

            const result = await cartService.applyCoupon(1, "SUMMER20");

            expect(result.discountPercent).toBe(20);
            expect(result.couponId).toBe(1);
        });

        test("Ném NotFoundError khi coupon không tồn tại", async () => {
            mockCouponRepo.findByCode.mockResolvedValue(null);

            await expect(cartService.applyCoupon(1, "INVALID"))
                .rejects.toThrow(NotFoundError);
        });

        test("Ném BusinessError khi coupon hết hạn", async () => {
            const expiredCoupon = {
                id: 1, code: "WINTER10", discountPercent: 10,
                expiresAt: new Date("2020-01-01"),  // đã qua
                usageCount: 5, maxUsage: 100
            };
            mockCouponRepo.findByCode.mockResolvedValue(expiredCoupon);

            await expect(cartService.applyCoupon(1, "WINTER10"))
                .rejects.toThrow("Coupon has expired");
        });

        test("Ném BusinessError khi coupon hết lượt dùng", async () => {
            const usedUpCoupon = {
                id: 1, code: "FLASH50", discountPercent: 50,
                expiresAt: new Date("2099-12-31"),
                usageCount: 100, maxUsage: 100  // đã dùng hết
            };
            mockCouponRepo.findByCode.mockResolvedValue(usedUpCoupon);

            await expect(cartService.applyCoupon(1, "FLASH50"))
                .rejects.toThrow("Coupon usage limit reached");
        });
    });
});
```

### 9.5.2 Unit Testing Frontend — React Component (Vitest + Testing Library)

```typescript
// components/ProductCard.tsx
interface ProductCardProps {
    product: {
        id: string;
        name: string;
        price: number;
        stock: number;
        imageUrl: string;
    };
    onAddToCart: (productId: string) => void;
}

export function ProductCard({ product, onAddToCart }: ProductCardProps) {
    const isOutOfStock = product.stock === 0;

    return (
        <div data-testid="product-card">
            <img src={product.imageUrl} alt={product.name} />
            <h3>{product.name}</h3>
            <p className="price">{product.price.toLocaleString("vi-VN")}đ</p>
            {isOutOfStock && (
                <span className="out-of-stock-badge">Hết hàng</span>
            )}
            <button
                onClick={() => onAddToCart(product.id)}
                disabled={isOutOfStock}
                data-testid="add-to-cart-btn"
            >
                {isOutOfStock ? "Hết hàng" : "Thêm vào giỏ"}
            </button>
        </div>
    );
}
```

```typescript
// components/ProductCard.test.tsx
import { render, screen, fireEvent } from "@testing-library/react";
import { expect, test, describe, vi } from "vitest";
import { ProductCard } from "./ProductCard";

const mockProduct = {
    id: "P001",
    name: "Áo thun Basic",
    price: 200_000,
    stock: 10,
    imageUrl: "/images/ao-thun.jpg"
};

describe("ProductCard Component", () => {
    test("Render đúng thông tin sản phẩm", () => {
        render(<ProductCard product={mockProduct} onAddToCart={vi.fn()} />);

        expect(screen.getByText("Áo thun Basic")).toBeInTheDocument();
        expect(screen.getByText(/200\.000đ/)).toBeInTheDocument();
        expect(screen.getByAltText("Áo thun Basic")).toBeInTheDocument();
    });

    test("Nút 'Thêm vào giỏ' khi còn hàng", () => {
        render(<ProductCard product={mockProduct} onAddToCart={vi.fn()} />);

        const button = screen.getByTestId("add-to-cart-btn");
        expect(button).toBeEnabled();
        expect(button).toHaveTextContent("Thêm vào giỏ");
        expect(screen.queryByText("Hết hàng")).not.toBeInTheDocument();
    });

    test("Nút bị disabled và hiển thị badge khi hết hàng", () => {
        const outOfStockProduct = { ...mockProduct, stock: 0 };
        render(<ProductCard product={outOfStockProduct} onAddToCart={vi.fn()} />);

        const button = screen.getByTestId("add-to-cart-btn");
        expect(button).toBeDisabled();
        expect(screen.getByText("Hết hàng")).toBeInTheDocument();
    });

    test("Gọi onAddToCart với đúng productId khi click", () => {
        const mockOnAddToCart = vi.fn();
        render(<ProductCard product={mockProduct} onAddToCart={mockOnAddToCart} />);

        fireEvent.click(screen.getByTestId("add-to-cart-btn"));

        expect(mockOnAddToCart).toHaveBeenCalledTimes(1);
        expect(mockOnAddToCart).toHaveBeenCalledWith("P001");
    });

    test("Không gọi onAddToCart khi sản phẩm hết hàng", () => {
        const mockOnAddToCart = vi.fn();
        const outOfStockProduct = { ...mockProduct, stock: 0 };
        render(<ProductCard product={outOfStockProduct} onAddToCart={mockOnAddToCart} />);

        fireEvent.click(screen.getByTestId("add-to-cart-btn"));

        expect(mockOnAddToCart).not.toHaveBeenCalled();
    });
});
```

---
