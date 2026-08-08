# Phần 11: Build & Deploy (Docker, Production)

Hướng dẫn đóng gói và triển khai project bằng Docker.

## 11.1. Tổng quan hạ tầng

```
[Frontend :3000] --> [Worksphere API :8080 (container)] --> [PostgreSQL]
                          |                                        |
                          +--[Redis :6379 (container)] ------------+
```

- App chạy trong container riêng; **Redis** là container hỗ trợ.
- Sử dụng `application.properties` + biến môi trường để cấu hình theo môi trường.
- Profile production được kích hoạt mặc định trong Dockerfile.

## 11.2. Dockerfile — multi-stage build

Dự án dùng Gradle, `Dockerfile` tại gốc:

```dockerfile
# Stage 1: build
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app
COPY gradlew gradle/ build.gradle settings.gradle ./
RUN chmod +x gradlew
COPY src/ src/
RUN ./gradlew bootJar --no-daemon          # đóng gói jar

# Stage 2: runtime (nhẹ, bảo mật)
FROM eclipse-temurin:17-jre-alpine
RUN addgroup -g 1001 -S worksphere && adduser -S worksphere -u 1001 -G worksphere
RUN mkdir -p /opt/worksphere/logs && chown -R worksphere:worksphere /opt/worksphere
WORKDIR /opt/worksphere
COPY --from=builder /app/build/libs/*.jar worksphere.jar
USER worksphere                            # không chạy root
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD wget --spider http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-Dspring.profiles.active=production", "-jar", "/opt/worksphere/worksphere.jar"]
```

Điểm quan trọng:
- **Multi-stage** → image runtime nhỏ (chỉ JRE, không có JDK/Gradle).
- **Non-root user** `worksphere` → bảo mật.
- **HEALTHCHECK** dựa trên `/actuator/health` (nhớ bật actuator trong production).
- **ENTRYPOINT** kích hoạt profile `production` mặc định.

## 11.3. docker-compose.yml (development)

```yaml
services:
  worksphere:
    build: .
    ports:
      - "${APP_PORT:-8080}:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=${SPRING_PROFILES_ACTIVE:-development}
      - JWT_SECRET=${JWT_SECRET:-dev-secret-key-must-be-at-least-32chars}
      - JWT_ACCESS_TOKEN_EXPIRATION=${JWT_ACCESS_TOKEN_EXPIRATION:-3600000}
      - REDIS_HOST=${REDIS_HOST:-redis}
      - REDIS_PORT=${REDIS_PORT:-6379}
      - GOOGLE_OAUTH2_CLIENT_ID=${GOOGLE_OAUTH2_CLIENT_ID}
      - GOOGLE_OAUTH2_CLIENT_SECRET=${GOOGLE_OAUTH2_CLIENT_SECRET}
      - FRONTEND_URL=${FRONTEND_URL:-http://localhost:3000}
    volumes:
      - ./logs:/opt/worksphere/logs
    depends_on:
      - redis
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes   # ghi log mỗi lệnh để không mất data khi restart
```

Chạy:

```bash
docker compose up -d --build
```

## 11.4. Triển khai production — việc cần làm

1. **JWT_SECRET**: phải override — không dùng giá trị mặc định dev (`dev-secret-key-...`). Dùng secret dài ≥ 32 ký tự.
2. **GOOGLE_OAUTH2_CLIENT_ID / SECRET / REDIRECT_URI**: cấu hình từ Google Cloud Console.
3. **SPRING_PROFILES_ACTIVE=production**: kích hoạt cấu hình production (DB riêng, log mức phù hợp, tắt console debug).
4. **Database**: dùng PostgreSQL ngoài container hoặc service riêng; cấp user/password qua biến môi trường (không hardcode).
5. **Redis password**: nếu cần, set `REDIS_PASSWORD` và cấu hình app tương ứng.

### Dùng file `.env` cho production

Tạo `.env` (KHÔNG commit lên git — thêm vào `.gitignore`):

```env
APP_PORT=8080
SPRING_PROFILES_ACTIVE=production
JWT_SECRET=<thay bằng secret thật dài >= 32 ký tự>
JWT_ACCESS_TOKEN_EXPIRATION=900000
GOOGLE_OAUTH2_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_OAUTH2_CLIENT_SECRET=xxx
FRONTEND_URL=https://frontend.example.com
REDIS_HOST=redis
REDIS_PORT=6379
```

Sau đó:

```bash
docker compose --env-file .env up -d --build
```

## 11.5. Vòng đời container

```bash
# Build & chạy
docker compose up -d --build

# Xem log
docker compose logs -f worksphere

# Kiểm tra health
curl http://localhost:8080/actuator/health

# Dừng
docker compose down

# Dừng + xóa volume (mất luôn dữ liệu Redis)
docker compose down -v
```

## 11.6. Kiểm tra sau khi deploy

- [ ] `GET /actuator/health` trả `{"status":"UP"}`.
- [ ] Đăng nhập được bằng JWT (không lỗi secret).
- [ ] Cache hoạt động: gọi 2 lần cùng endpoint → lần 2 không query DB (check log/Redis).
- [ ] Log app ghi vào `/opt/worksphere/logs` (volume mount `./logs`).
- [ ] App không chạy với quyền root (user `worksphere`).

## 11.7. Lỗi thường gặp

| Lỗi | Nguyên nhân & cách xử lý |
| --- | ------------------------- |
| App crash khi start, lỗi connect Redis | Redis chưa sẵn sàng → thêm `depends_on` + retry, hoặc bật fallback cache (`app.cache.fallback.enabled=true`) |
| `403` khi gọi API | JWT secret khác giữa các môi trường → đảm bảo cùng 1 `JWT_SECRET` |
| `actuator/health` 404 | Chưa bật `spring-boot-starter-actuator` trong production profile |
| OAuth2 redirect sai | `GOOGLE_OAUTH2_REDIRECT_URI` phải khớp chính xác với cấu hình Google Cloud |
