# Phần 2: Cấu trúc dự án

## 2.1. Tổng quan thư mục gốc

```
<project_name>/
├── src/
│   ├── main/
│   │   ├── java/com/example/<project_name>/   → mã nguồn chính
│   │   └── resources/                    → cấu hình + tài nguyên
│   └── test/                             → test
├── build.gradle          → khai báo dependency, plugin, task (dự án dùng Gradle)
├── settings.gradle       → rootProject.name = '<project_name>'
├── gradlew               → Gradle wrapper
├── .env.example          → mẫu biến môi trường
├── docker-compose.yml    → chạy PostgreSQL + Redis (dev)
├── Dockerfile            → đóng image chạy production
└── README.md
```

## 2.2. Entry point

`src/main/java/com/example/<project_name>/<ProjectName>Application.java`

- `@SpringBootApplication` — khởi động Spring Boot.
- `@EnableScheduling` — bật scheduler (dùng để dọn token hết hạn...).
- Trong `main()`: set **timezone mặc định là UTC** cho JVM, và đăng ký **spring-dotenv** (`DotenvPropertySource`) để đọc biến từ file `.env`.

## 2.3. Cấu trúc package `com.example.<project_name>`

### Module nghiệp vụ (Business modules)

Mỗi module là một package con ở root, ví dụ: `auth`, `employee`, `department`, `attendance`, `contract`, `degree`, `insurance`, `leave`, `notification`, `payroll`, `relative`, `store`, `user`, `workshift`, `authorization`.

Mỗi module nghiệp vụ thường có **đủ 6 package con** theo đúng thứ tự luồng:

```
employee/
├── controller/   → REST API (nhận request HTTP)
├── dto/
│   ├── request/  → object nhận dữ liệu từ client
│   └── response/ → object trả về client
├── domain/       → Entity JPA (ánh xạ bảng DB)
├── mapper/       → MapStruct: chuyển Entity ↔ DTO
├── repository/   → Spring Data JPA (thao tác DB)
└── service/      → logic nghiệp vụ (gọi repository, xử lý)
```

Một số module có package riêng:

- `auth` — thêm `config`, `security`, `util` (JWT, filter, OAuth2, cấu hình).
- `authorization` — thêm `security` (RBAC: `@RequirePermission`).
- `notification` — thêm `scheduler` (gửi thông báo định kỳ).

### Module dùng chung `shared`

Chứa các thành phần tái sử dụng cho mọi module:

```
shared/
├── audit/        → ghi log thao tác (annotation, aspect, service, domain, dto...)
├── cache/        → cấu hình cache Redis + fallback
├── config/       → cấu hình chung + properties
├── constant/     → hằng số (EntityType, PermissionType, HttpMethod...)
├── domain/       → base entity (`@AuditableEntity`)
├── dto/          → `ApiResponse`, `PaginatedApiResponse`, `PageInfo`, `ErrorResponse`...
├── exception/    → `BaseException`, `GlobalExceptionHandler` + các exception cụ thể
├── persistence/  → encryptor + JPA converter (mã hoá field nhạy cảm)
├── ratelimit/    → giới hạn số request/phút
├── security/     → helpers bảo mật (lấy user hiện tại...)
├── service/      → service dùng chung (Cloudinary, Face API, IP address...)
├── util/         → tiện ích
└── web/          → cấu hình web (CORS, Jackson...)
```

## 2.4. Quy tắc phân lớp (Layer rule)

- **Controller** chỉ nhận request, gọi Service, không chứa logic.
- **Service** chứa logic nghiệp vụ, gọi Repository, có thể đánh dấu `@Transactional`.
- **Repository** thao tác DB, query theo Entity.
- **Entity** (domain) là JPA entity, không trả thẳng ra API.
- **DTO** tách biệt: `request` (input) và `response` (output) — không lộ field không cần thiết ra client.
- **Mapper** (MapStruct) chuyển đổi Entity ↔ DTO.
- Controller trả về `ApiResponse<T>` hoặc `PaginatedApiResponse<T>` (xem Phần xử lý response).
