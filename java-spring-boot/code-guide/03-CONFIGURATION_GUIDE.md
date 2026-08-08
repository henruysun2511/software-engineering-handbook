# Phần 3: Cấu hình môi trường

## 3.1. File cấu hình

```
src/main/resources/
├── application.properties          → cấu hình chung (mọi môi trường)
├── application-development.properties → profile dev
├── application-production.properties  → profile prod
└── logback-spring.xml              → cấu hình log
```

- Chọn profile bằng biến `SPRING_PROFILES_ACTIVE` (VD: `production` hoặc `development`).
- Giá trị cấu hình đọc từ biến môi trường theo cú pháp `${TÊN_BIẾN:mặc_định}`.
- File `.env` (copy từ `.env.example`) được nạp tự động qua **spring-dotenv** — dùng để giữ **secret không commit lên git**.

## 3.2. Nhóm cấu hình chính (trong `application.properties`)

| Nhóm | Key chính | Ý nghĩa |
| ---- | --------- | ------- |
| Server | `server.port` | Cổng chạy app (mặc định 8080) |
| Database | `spring.jpa.hibernate.ddl-auto` | Tự động cập nhật schema (update) |
| Redis | `spring.data.redis.host/port/password` | Kết nối Redis (cho cache + rate limit) |
| Cache | `spring.cache.type=redis` | Dùng Redis làm cache |
| OAuth2 | `spring.security.oauth2.client.registration.google.*` | Google OAuth2 login |
| JWT | `jwt.secret`, `jwt.access-token-expiration`, `jwt.refresh-token-expiration` | Ký & hạn token |
| Security | `app.security.rbac.enabled` | Bật/tắt RBAC |
| Security | `app.security.public-endpoints` | Danh sách endpoint công khai (không cần login) |
| Audit | `app.audit.*` | Cấu hình audit log (bật/tắt, log params/response...) |
| Rate limit | `app.rate-limit.*` | Giới hạn request/phút theo từng loại |
| Upload | `spring.servlet.multipart.*` | Giới hạn size file upload (10MB) |
| JSON | `spring.jackson.*` | Serialize thời gian (UTC, không in field null) |
| Swagger | `springdoc.*` | Đường dẫn `/v3/api-docs`, `/swagger-ui.html` |
| Actuator | `management.*` | Chỉ expose `/actuator/health` |
| Face API | `face-api.base-url` | URL service Python nhận diện khuôn mặt |
| Encryption | `app.security.encryption.key-base64` | Key mã hoá field nhạy cảm |

## 3.3. Sự khác nhau giữa 2 profile

| | `development` | `production` |
| - | ------------- | ------------ |
| SQL log | Hiện (DEBUG/TRACE) | Tắt (`show-sql=false`) |
| Log level | `DEBUG` (worksphere, security) | `ERROR`/`WARN`, ghi vào `logs/worksphere-production.log` |
| Stacktrace lỗi | Luôn hiện | Không hiện |
| DB | PostgreSQL local (`localhost`, user/pass mặc định `postgres/postgres`) | Lấy từ biến môi trường `DB_HOST/DB_PORT/DB_NAME/DB_USERNAME/DB_PASSWORD` |

## 3.4. Các biến môi trường quan trọng (`.env.example`)

- `JWT_SECRET` — chuỗi bí mật ký token (tối thiểu 32 ký tự).
- `JWT_ACCESS_TOKEN_EXPIRATION` / `JWT_REFRESH_TOKEN_EXPIRATION` — thời gian sống token (ms).
- `POSTGRES_DB/USER/PASSWORD/PORT` — thông tin database.
- `REDIS_HOST/PORT/PASSWORD` — thông tin Redis.
- `GOOGLE_OAUTH2_CLIENT_ID/SECRET/REDIRECT_URI` — Google OAuth2.
- `FRONTEND_URL` — URL frontend (redirect sau login OAuth).
- `RBAC_ENABLED` — bật/tắt phân quyền theo vai trò.
- `AUDIT_ENABLED` — bật/tắt audit log.
- `RATE_LIMIT_ENABLED` — bật/tắt giới hạn request.
- `DDL_AUTO` — chiến lược tạo schema (mặc định `update`).
- `SPRING_PROFILES_ACTIVE` — chọn profile `development`/`production`.

## 3.5. Cách chạy với profile khác

```bash
# Dev mặc định (không set profile) — dùng localhost PostgreSQL
.\mvnw.cmd spring-boot:run

# Chạy profile development
set SPRING_PROFILES_ACTIVE=development
.\mvnw.cmd spring-boot:run

# Chạy profile production (cần .env đã điền secret)
set SPRING_PROFILES_ACTIVE=production
.\mvnw.cmd spring-boot:run
```
