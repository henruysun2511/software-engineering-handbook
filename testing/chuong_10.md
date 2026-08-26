# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# CHƯƠNG 10: INTEGRATION TESTING

---

## 10.1 Integration Testing là gì?

**Integration Testing** kiểm thử sự tương tác và giao tiếp giữa các component, module, hoặc service khi chúng được kết hợp lại. Không giống Unit Test (cô lập hoàn toàn), Integration Test để các phần thật giao tiếp với nhau — ít nhất là một phần.

**Ranh giới giữa Unit Test và Integration Test:**

```
Unit Test:
  CartService (test cô lập)
  ↕ [Mock]
  ProductRepository (không thật)
  ↕ [Mock]
  Database (không thật)

Integration Test:
  CartService (thật)
  ↕ [Thật]
  ProductRepository (thật)
  ↕ [Thật]
  Test Database (database thật nhưng chỉ dùng cho test)
```

---

## 10.2 Các Dạng Integration Test

### 10.2.1 API Integration Test — NestJS + Supertest

```typescript
// Cấu trúc:
// Test gọi HTTP request đến API thật → API xử lý → Trả response
// Verify response VÀ verify dữ liệu trong test database

// app.e2e-spec.ts (NestJS)
import { Test, TestingModule } from "@nestjs/testing";
import { INestApplication } from "@nestjs/common";
import * as request from "supertest";
import { AppModule } from "../src/app.module";
import { DataSource } from "typeorm";

describe("Auth API Integration Tests", () => {
    let app: INestApplication;
    let dataSource: DataSource;

    beforeAll(async () => {
        const moduleFixture: TestingModule = await Test.createTestingModule({
            imports: [AppModule],  // Load toàn bộ app với test DB
        }).compile();

        app = moduleFixture.createNestApplication();
        await app.init();

        dataSource = moduleFixture.get<DataSource>(DataSource);
    });

    afterAll(async () => {
        await dataSource.destroy();
        await app.close();
    });

    beforeEach(async () => {
        // Xóa data test trước mỗi test
        await dataSource.query("DELETE FROM users WHERE email LIKE '%@test-integration.com'");
    });

    // ===== Register =====
    describe("POST /api/auth/register", () => {
        test("Đăng ký thành công — 201 và tạo user trong DB", async () => {
            const registerDto = {
                email: `qa_${Date.now()}@test-integration.com`,
                password: "Test@123",
                name: "Integration Test User"
            };

            const response = await request(app.getHttpServer())
                .post("/api/auth/register")
                .send(registerDto)
                .expect(201);

            // Verify response
            expect(response.body.success).toBe(true);
            expect(response.body.data.userId).toBeDefined();

            // Verify database
            const [user] = await dataSource.query(
                "SELECT id, email, status, email_verified FROM users WHERE email = $1",
                [registerDto.email]
            );
            expect(user).toBeDefined();
            expect(user.status).toBe("pending_verification");
            expect(user.email_verified).toBe(false);

            // Verify verification token được tạo
            const [token] = await dataSource.query(
                "SELECT * FROM verification_tokens WHERE user_id = $1",
                [user.id]
            );
            expect(token).toBeDefined();
            expect(token.used).toBe(false);
        });

        test("Đăng ký thất bại với email trùng — 409", async () => {
            const email = `duplicate_${Date.now()}@test-integration.com`;

            // Tạo user đầu tiên
            await request(app.getHttpServer())
                .post("/api/auth/register")
                .send({ email, password: "Test@123", name: "User 1" })
                .expect(201);

            // Đăng ký lại email đó
            const response = await request(app.getHttpServer())
                .post("/api/auth/register")
                .send({ email, password: "Test@123", name: "User 2" })
                .expect(409);

            expect(response.body.error).toContain("Email");

            // Verify không tạo user thứ 2
            const users = await dataSource.query(
                "SELECT id FROM users WHERE email = $1",
                [email]
            );
            expect(users).toHaveLength(1);
        });

        test("Đăng ký thất bại thiếu password — 400", async () => {
            const response = await request(app.getHttpServer())
                .post("/api/auth/register")
                .send({ email: `test@test-integration.com`, name: "Test" })
                .expect(400);

            expect(response.body.error).toBeDefined();
        });
    });

    // ===== Login =====
    describe("POST /api/auth/login", () => {
        let testUserEmail: string;

        beforeEach(async () => {
            // Tạo và activate user test
            testUserEmail = `login_test_${Date.now()}@test-integration.com`;
            await dataSource.query(
                `INSERT INTO users (email, password_hash, name, status, email_verified)
                 VALUES ($1, $2, $3, 'active', true)`,
                [testUserEmail, "$2b$10$hashedpassword", "Login Test User"]
            );
        });

        test("Đăng nhập thành công — 200 và nhận JWT token", async () => {
            const response = await request(app.getHttpServer())
                .post("/api/auth/login")
                .send({ email: testUserEmail, password: "Test@123" })
                .expect(200);

            expect(response.body.data.token).toBeDefined();
            expect(response.body.data.token.split(".")).toHaveLength(3); // JWT format

            expect(response.body.data.user).toMatchObject({
                email: testUserEmail,
                role: "customer"
            });
            expect(response.body.data.user.password).toBeUndefined();
        });

        test("Đăng nhập thất bại với password sai — 401", async () => {
            await request(app.getHttpServer())
                .post("/api/auth/login")
                .send({ email: testUserEmail, password: "WrongPassword" })
                .expect(401);
        });
    });
});
```

### 10.2.2 API Integration Test — Spring Boot (Java)

```java
// AuthControllerIntegrationTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")  // dùng application-test.properties với test DB
@Transactional  // rollback sau mỗi test
class AuthControllerIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private UserRepository userRepository;

    @Test
    @DisplayName("Đăng ký thành công — 201 Created")
    void register_WithValidData_Returns201() {
        // Arrange
        RegisterRequest request = new RegisterRequest(
            "integration_test@example.com",
            "Test@123",
            "Integration Tester"
        );

        // Act
        ResponseEntity<ApiResponse> response = restTemplate.postForEntity(
            "/api/auth/register",
            request,
            ApiResponse.class
        );

        // Assert — Response
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().isSuccess()).isTrue();

        // Assert — Database
        Optional<User> savedUser = userRepository.findByEmail("integration_test@example.com");
        assertThat(savedUser).isPresent();
        assertThat(savedUser.get().getStatus()).isEqualTo(UserStatus.PENDING_VERIFICATION);
        assertThat(savedUser.get().getEmailVerified()).isFalse();
    }

    @Test
    @DisplayName("Đăng ký email trùng — 409 Conflict")
    void register_WithDuplicateEmail_Returns409() {
        // Arrange — tạo user sẵn
        userRepository.save(User.builder()
            .email("existing@example.com")
            .passwordHash("$2a$...")
            .name("Existing User")
            .status(UserStatus.ACTIVE)
            .build());

        RegisterRequest request = new RegisterRequest(
            "existing@example.com", "Test@123", "New User"
        );

        // Act
        ResponseEntity<ApiResponse> response = restTemplate.postForEntity(
            "/api/auth/register", request, ApiResponse.class
        );

        // Assert
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CONFLICT);
        assertThat(userRepository.countByEmail("existing@example.com")).isEqualTo(1);
    }
}
```

---

## 10.3 Database Integration Test

```typescript
// Kiểm thử trực tiếp Repository với test database
describe("ProductRepository Integration", () => {
    let dataSource: DataSource;
    let productRepo: ProductRepository;

    beforeAll(async () => {
        dataSource = new DataSource({
            type: "postgres",
            url: process.env.TEST_DATABASE_URL,
            entities: [Product, Category],
            synchronize: false  // không auto-migrate trong test
        });
        await dataSource.initialize();
        productRepo = new ProductRepository(dataSource);
    });

    afterAll(async () => await dataSource.destroy());

    beforeEach(async () => {
        await dataSource.query("DELETE FROM products WHERE name LIKE '[TEST]%'");
    });

    test("findById trả về đúng product", async () => {
        // Insert test data
        const [inserted] = await dataSource.query(
            `INSERT INTO products (name, price, stock)
             VALUES ('[TEST] Product', 100000, 10) RETURNING id`
        );

        const product = await productRepo.findById(inserted.id);

        expect(product).toBeDefined();
        expect(product!.name).toBe("[TEST] Product");
        expect(product!.price).toBe(100000);
    });

    test("findById trả về null khi không tìm thấy", async () => {
        const product = await productRepo.findById("non-existent-id");
        expect(product).toBeNull();
    });

    test("decreaseStock cập nhật đúng", async () => {
        const [inserted] = await dataSource.query(
            `INSERT INTO products (name, price, stock)
             VALUES ('[TEST] Stock Test', 100000, 10) RETURNING id`
        );

        await productRepo.decreaseStock(inserted.id, 3);

        const [result] = await dataSource.query(
            "SELECT stock FROM products WHERE id = $1", [inserted.id]
        );
        expect(result.stock).toBe(7);
    });
});
```

---

## 10.4 Service Integration Test — Kiểm thử nhiều Service phối hợp

```typescript
// Kiểm thử OrderService gọi ProductService và EmailService thật
describe("OrderService Integration", () => {
    let orderService: OrderService;
    let productService: ProductService;
    let dataSource: DataSource;

    // Email service vẫn mock (không muốn gửi email thật)
    const mockEmailService = {
        sendOrderConfirmation: jest.fn().mockResolvedValue(undefined)
    };

    beforeAll(async () => {
        dataSource = await createTestDataSource();
        productService = new ProductService(dataSource);
        orderService = new OrderService(productService, mockEmailService as any, dataSource);
    });

    afterAll(async () => await dataSource.destroy());

    test("Tạo order trừ tồn kho và gửi email", async () => {
        // Setup test data
        await dataSource.query(
            "INSERT INTO products (id, name, price, stock) VALUES ('P-TEST', 'Test Product', 100000, 5)"
        );
        await dataSource.query(
            "INSERT INTO users (id, email, name, status) VALUES (999, 'test@integration.com', 'Test', 'active')"
        );

        // Act
        const order = await orderService.createOrder({
            userId: 999,
            items: [{ productId: "P-TEST", quantity: 2 }],
            paymentMethod: "cod"
        });

        // Assert — Order được tạo
        expect(order.status).toBe("pending");
        expect(order.total).toBe(200000);

        // Assert — Tồn kho giảm (ProductService thật chạy)
        const [product] = await dataSource.query(
            "SELECT stock FROM products WHERE id = 'P-TEST'"
        );
        expect(product.stock).toBe(3);  // 5 - 2 = 3

        // Assert — Email được gửi (mock verify)
        expect(mockEmailService.sendOrderConfirmation).toHaveBeenCalledTimes(1);
        expect(mockEmailService.sendOrderConfirmation).toHaveBeenCalledWith(
            expect.objectContaining({ userId: 999, total: 200000 })
        );

        // Cleanup
        await dataSource.query("DELETE FROM orders WHERE user_id = 999");
        await dataSource.query("DELETE FROM users WHERE id = 999");
        await dataSource.query("DELETE FROM products WHERE id = 'P-TEST'");
    });
});
```

---

## 10.5 Cấu hình Test Database

**Nguyên tắc:** Không bao giờ dùng production database để integration test.

```yaml
# application-test.yml (Spring Boot)
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/myapp_test
    username: test_user
    password: test_password
  jpa:
    hibernate:
      ddl-auto: create-drop  # Tạo schema mới và drop sau khi test

# Hoặc dùng H2 in-memory database
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
```

```typescript
// jest.config.ts — setup test database
export default {
    globalSetup: "./test/setup/globalSetup.ts",
    globalTeardown: "./test/setup/globalTeardown.ts",
    setupFilesAfterEach: "./test/setup/jest.setup.ts",
};

// test/setup/globalSetup.ts
export default async function globalSetup() {
    // Tạo test database và chạy migrations
    await exec("npx typeorm migration:run --dataSource=src/data-source.test.ts");
    console.log("Test database ready");
}

// test/setup/globalTeardown.ts
export default async function globalTeardown() {
    await exec("npx typeorm schema:drop --dataSource=src/data-source.test.ts");
    console.log("Test database cleaned up");
}
```
