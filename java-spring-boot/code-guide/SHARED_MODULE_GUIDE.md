# Hướng dẫn chi tiết module `shared` - HRM_BE (Worksphere)

Module **`shared`** nằm tại `src/main/java/com/hainam/worksphere/shared/`, chứa các thành phần **dùng chung** cho toàn bộ backend: cấu hình (config), các chuẩn API Response/DTO, hệ thống exception/handler, audit logging, rate limiting, caching với fallback, mã hóa dữ liệu trước khi lưu DB, và các tiện ích gọi API bên ngoài.

Khác với các module nghiệp vụ (employee, department, ...) chỉ gồm các package `controller/domain/dto/mapper/repository/service`, module `shared` là **nền tảng kỹ thuật (infrastructure layer)**, không chứa nghiệp vụ cụ thể của từng phân hệ.

Cấu trúc package:

```
shared/
├── audit/          # Hệ thống ghi nhật ký kiểm toán (audit logging)
├── cache/          # Cache fallback khi Redis lỗi + health check
├── config/         # Các class cấu hình Spring (Redis, Jackson, Swagger, Cloudinary...)
├── constant/       # Hằng số (quyền hạn)
├── domain/         # Enum dùng chung (EntityType)
├── dto/            # DTO chuẩn phản hồi API
├── exception/      # Hệ thống exception + GlobalExceptionHandler
├── persistence/    # Mã hóa dữ liệu khi lưu DB
├── ratelimit/      # Giới hạn tốc độ request (rate limiting)
├── security/       # Annotation bảo mật dùng chung
├── service/        # Service dùng chung (Cloudinary)
├── util/           # Tiện ích (IP, environment, Face API client)
└── web/            # Enum liên quan web (HttpMethod)
```

---

## 1. Package `dto` — Chuẩn phản hồi API

Các DTO này là "hợp đồng" trả về cho client, được dùng ở hầu hết controllers của mọi module.

### 1.1 `ApiResponse.java` — Response chuẩn không phân trang

`ApiResponse<T>` là DTO gốc, mọi endpoint đơn lẻ đều đóng gói dữ liệu bằng class này.

Điểm quan trọng:
```java
@JsonInclude(JsonInclude.Include.NON_NULL)   // Bỏ qua field null khi serialize JSON
private boolean success;
private String code;      // mã lỗi (VD: RESOURCE_NOT_FOUND)
private String message;
private T data;           // dữ liệu thực tế (generic)
private List<String> errors;   // danh sách lỗi validate
@Builder.Default private Instant timestamp = Instant.now();
```
- Các **static factory method** là điểm quan trọng nhất giúp viết code ngắn gọn:
  - `success(T data)` / `success(String message, T data)` → trả về chuẩn thành công.
  - `error(message)` / `error(message, errors)` / `error(code, message, data)` → trả về chuẩn lỗi.

> Lý do quan trọng: nhờ `@Builder` + static method, controller chỉ cần `ApiResponse.success(data)` là có response thống nhất trên toàn hệ thống.

### 1.2 `ErrorResponse.java` — Phản hồi lỗi kiểu cũ
- DTO đơn giản gồm `status`, `error`, `message`, `timestamp`, `path`. Khác `ApiResponse` ở chỗ chứa thêm `path` (đường dẫn request lỗi) và HTTP status. Hiện ít dùng vì `ApiResponse.error()` đã thay thế.

### 1.3 `PageInfo.java` — Thông tin phân trang
- Chứa `page`, `size`, `totalItems`, `totalPages`.
- **Static method `of(Page<?> page)`** lấy trực tiếp từ `Page` của Spring Data:
```java
public static PageInfo of(Page<?> page) {
    return PageInfo.builder()
            .page(page.getNumber())
            .size(page.getSize())
            .totalItems(page.getTotalElements())
            .totalPages(page.getTotalPages())
            .build();
}
```
> Chuyển `Page` của Spring thành JSON mỏng gọn, không lộ các chi tiết kỹ thuật bên trong Spring.

### 1.4 `PaginatedApiResponse.java` — Phản hồi phân trang chuẩn
- Tương tự `ApiResponse` nhưng `data` là `List<T>` kèm `pagination` (kiểu `PageInfo`).
- Static factory `success(Page<T> page)` lấy `page.getContent()` làm data và `PageInfo.of(page)` làm phân trang — chỗ tái sử dụng quan trọng với 2 DTO trên.

### 1.5 `Resource ApiStatsResponse.java` — Thống kê tài nguyên
- DTO đơn giản chứa `total`, `active`, `inactive`, `deleted`. Dùng để thống kê trạng thái các entity dùng soft-delete (`isDeleted`).

---

## 2. Package `exception` — Hệ thống lỗi

### 2.1 `BaseException.java` — Lớp base (abstract)
Point: tất cả exception nghiệp vụ đều kế thừa class này, bắt buộc cung cấp:
```java
public abstract class BaseException extends RuntimeException {
    private final HttpStatus status;   // HTTP status trả về
    private final String errorCode;    // mã lỗi (VD: RESOURCE_NOT_FOUND)
}
```
> Nhờ vậy `GlobalExceptionHandler` chỉ cần bắt `BaseException` là xử lý được toàn bộ lỗi nghiệp vụ, đọc `status` và `errorCode` từ chính exception.

### 2.2 `ResourceNotFoundException.java` — Exception "không tìm thấy tài nguyên"
- Mở rộng `BaseException`, fix `ERROR_CODE = "RESOURCE_NOT_FOUND"` và `HttpStatus.NOT_FOUND`.
- Constructor gắn cấp 3 tham số `(resourceType, identifier)` tự sinh message: `"%s not found with identifier: %s"` — dùng trong service khi `repository.findById(id).orElseThrow(() -> new ResourceNotFoundException("Employee", id))`.

### 2.3 `ValidationException.java` — Lỗi xác thực dữ liệu
- Mã `VALIDATION_ERROR`, `HttpStatus.BAD_REQUEST`.
- Chứa các **factory method** tiện lợi: `fieldError(field, error)`, `passwordMismatch()`, `duplicateField(field, value)` — giúp tạo lỗi validate ngắn gọn từ service.

### 2.4 Các exception chuyên biệt khác
Nằm cùng package, đều kế thừa `BaseException`. Chia 2 nhóm:
- **NotFound theo entity**: `EmployeesNotFoundException`, `DepartmentNotFoundException`, `ContractNotFoundException`, `DegreeNotFoundException`, `InsuranceNotFoundException`, `LeaveRequestNotFoundException`, `PayrollNotFoundException`, `RelativeNotFoundException`, `StoreNotFoundException`, `WorkShiftNotFoundException`, `AttendanceNotFoundException`. Mỗi class gần giống `ResourceNotFoundException` nhưng có `ERROR_CODE` riêng.
- **Nghiệp vụ khác**:
  - `DuplicateResourceException` → 409 CONFLICT.
  - `InvalidCredentialsException`, `InvalidTokenException`, `RefreshTokenException`, `OAuth2ValidationException`, `UserInactiveException`, `UserNotFoundException` → lỗi auth.
  - `AccessDeniedException`, `EmailAlreadyExistsException`, `BusinessRuleViolationException`.
  - `RateLimitExceededException`, `RateLimitBannedException` → 429, kèm `retryAfterSeconds` và `bannedKey`.

### 2.5 `GlobalExceptionHandler.java` — Bộ xử lý lỗi trung tâm — **QUAN TRỌNG NHẤT package**
`@RestControllerAdvice` trung tâm chuyển exception thành `ResponseEntity<ApiResponse<Void>>`.

Các `@ExceptionHandler` đáng chú ý:
- `handleBaseException(BaseException)` → trả về `status` ngay từ exception, `ApiResponse.error(message)`.
- `handleValidationExceptions(MethodArgumentNotValidException)` — **quan trọng**: nhận lỗi từ Spring Bean Validation, gộp tất cả `FieldError.getDefaultMessage()` thành `List<String>` rồi trả `ApiResponse.error("Validation failed", errors)`. Đây là cách một request sai nhiều field được trả về đầy đủ lỗi.
- `handleAuthenticationExceptions({BadCredentialsException, UsernameNotFoundException})` → 401 với gởi thông báo thống nhất "Invalid email or password" (không lộ lý do cụ thể — bảo mật).
- 2 handler riêng cho JWT: `ExpiredJwtException` → 401 "Token has expired"; còn `MalformedJwtException`/`UnsupportedJwtException` → 401 "Invalid token".
- `handleRateLimitExceededException`/`handleRateLimitBannedException` → **429 + header `Retry-After`** (số giây client phải chờ).
- `handleAccessDeniedException` → 403.
- Handler đáy như `handleRuntimeException` và `handleGenericException` → catch mọi thứ còn sót lại, trả 500 SAP "An internal server error occurred..." — **bảo vệ không để lộ stack trace cho client**.

> Ngoài ra đầy đủ `NoResourceFoundException` (404), `IllegalArgumentException` (400).

---

## 3. Package `config` — Cấu hình chung

### 3.1 Config chuẩn Spring Boot

#### `CacheConfig.java` — Cấu hình cache Redis — **LINH HOẠT với fallback**
- Định nghĩa **các hằng số tên cache** (`USER_CACHE="users"`, `ROLE_CACHE`, `EMPLOYEE_CACHE`...) dùng ở các `@Cacheable`/`@CacheEvict` trong toàn dự án. Đây là nơi duy nhất định nghĩa tên cache → tránh sai tên gõ lẫn.
- **`redisSerializer()`** dùng `GenericJackson2JsonRedisSerializer` với `BasicPolymorphicTypeValidator` + `activateDefaultTyping` để giữ **thông tin kiểu dữ liệu (class)** khi serialize object vào Redis — mới khi deserialize lại đúng class gốc. Có `JavaTimeModule` để hỗ trợ JSR-310.
- Bean `redisTemplate` cấu hình key serializer là `StringRedisSerializer`, value serializer JSON.
- Bean `redisCacheManager` đặt TTL mặc định 30 phút, disable cache null value.
- **`getCacheConfigurations()`**: định nghĩa TTL theo từng loại cache — logic quan trọng:
  - Tài nguyên ít đổi (roles, permissions, departments, workShifts, stores) → 1 giờ.
  - Dữ liệu động (attendance) → 10 phút.
  - user/employee/contract/payroll/insurance/degree/relative → 30 phút.
- Bean `cacheManager` `@Primary` bọc `RedisCacheManager` bằng `FallbackCacheManager(...)` — nối với package `cache` (đọc phần 4). TTL `@ConditionalOnProperty` tạo `noOpCacheManager` khi fallback tắt.

#### `JacksonConfig.java`
- Bean `ObjectMapper` `@Primary`, gắn `JavaTimeModule`, đồng thời tắt: `WRITE_DATES_AS_TIMESTAMPS` (ngày giờ thành ISO string, không phải epoch), `WRITE_DATE_TIMESTAMPS_AS_NANOSECONDS`, `READ_DATE_TIMESTAMPS_AS_NANOSECONDS`.

#### `MapStructConfig.java`
- `@ComponentScan` để quét các package mapper (`user.mapper`, `auth.mapper`) cho MapStruct — đảm bảo các mapper được tạo thành Bean.

#### `SwaggerConfig.java`
- Tạo bean `OpenAPI`: title/desc/version từ `@Value`, thêm server `http://localhost:{port}`, gắn **SecurityScheme Bearer JWT** (`type HTTP, scheme bearer, bearerFormat JWT`) + `SecurityRequirement` để mọi endpoint Swagger cần JWT.

#### `CloudinaryConfig.java`
- Bean `Cloudinary` đọc `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET` từ env (mặc định rỗng nếu không có) và `secure: true`. Dùng cho upload ảnh/file.

#### `DotenvConfiguration.java`
- `validateRequiredEnvironmentVariables(Environment)`: kiểm tra đầu app có đủ env bắt buộc (`JWT_SECRET`, `DB_HOST`, `DB_PASSWORD`, `SERVER_PORT`, `FRONTEND_URL`, `OAUTH_SUCCESS_REDIRECT_PATH`...) hay không; thiếu → `IllegalStateException` chặn khởi động. Tránh app chạy nhưng cấu hình thiếu.

### 3.2 Class Properties ràng buộc (`@ConfigurationProperties`)

#### `SecurityProperties.java` (prefix `app.security`)
- Chứa `Rbac rbac` (`enabled = true`) và `publicEndpoints`.
- **`getPublicEndpointsList()`**: trả danh sách các endpoint không cần auth. Nếu `publicEndpoints` rỗng thì dùng **list mặc định** (`/api/v1/auth/register`, login, refresh, oauth2, swagger-api, actuator/health...); ngược lại split theo dấu phẩy. Nơi security filter quyết định endpoint nào công khai.

#### `RateLimitProperties.java` (prefix `app.rate-limit`)
- Các tham số giới hạn tốc độ: `enabled`, `defaultRequestsPerMinute` (100), `loginRequestsPerMinute` (10), `registerRequestsPerMinute` (5), `refreshTokenRequestsPerMinute` (30), `anonymousRequestsPerMinute` (50), `banDurationMinutes` (15), `maxViolationsBeforeBan` (5). — được `RateLimitService` đọc để tính limit từng loại endpoint.

#### `properties/CacheFallbackProperties.java` (prefix `app.cache.fallback`)
- Cấu hình fallback cache: `enabled`, `logLevel`, `periodicHealthCheck`, `healthCheckIntervalSeconds`, `maxConsecutiveFailures`, `enableStats`, `statsReportingIntervalSeconds`. — `RedisHealthCheckService` dùng cho health check.

---

## 4. Package `cache` — Cache với fallback khi Redis lỗi (quan trọng cho độ sẵn sàng)

Mục đích: nếu Redis ngừng hoạt động, ứng dụng **không sập** mà tự động chuyển sang truy vấn DB.

### 4.1 `CacheConfig` hội tụ → `FallbackCacheManager.java`
- Là `CacheManager` custom, `@Primary` trong `CacheConfig`.
- `getCache(name)`: nếu `enableFallback` tắt → trả thẳng `redisCacheManager.getCache(name)`; bật → lấy/lưu dựng cache vào `cacheInstances` qua `computeIfAbsent` để tránh dựng lại nhiều lần.
- `createCacheWithFallback(name)`: thử lấy cache từ Redis; thành công gói vào `FallbackCache`, thất bại catch exception → trả `NoOpCache` (cache rỗng, không lỗi). → **System tiếp tục chạy dù Redis chết**.

### 4.2 `FallbackCache.java` — Cache wrapper tự suy giảm (fallback)
- Bọc `Cache redisCache` thật. Trong mỗi operation (`get/put/evict/clear...`):
  - Nếu `!redisAvailable` → bỏ qua operation một cách an toàn (trả null / bỏ qua), **không throw**.
  - Thử operation; bắt `RedisConnectionFailureException` → `handleRedisFailure` đặt `redisAvailable=false`, log warning "Falling back to database queries".
  - Hàm `get(key, valueLoader)` đặc biệt: khi Redis lỗi hoặc lỗi thì **trực tiếp gọi `valueLoader.call()`** — tức là chạy hàm load dữ liệu thật từ DB, đúng ý tưởng "fallback to DB".
  - Khi operation thành công → `markRedisAvailable()` đặt lại cờ true (tự phục hồi).

> Đây là lõi: mọi annotation `@Cacheable` đều đi qua `FallbackCache`; nếu Redis/linh, hệ thống không ném lỗi mà tự đi query DB.

### 4.3 `RedisHealthCheckService.java` — Giám sát sức khỏe Redis
- `@Scheduled(fixedDelayString = ...)` chạy định kỳ (mặc định 30s): set 1 key test TTL rồi xóa → kiểm tra kết nối.
- Dùng `AtomicBoolean redisAvailable`, `AtomicInteger consecutiveFailures`. Gọi `markHealthCheckFailure` tăng `consecutiveFailures`; đạt `maxConsecutiveFailures` (3) thì đánh dấu `redisAvailable=false`.
- Cung cấp `isRedisAvailable()`, `getRedisConnectionInfo()` (lấy version bằng `connection.info()`), `testRedisConnectivity()` (test thủ công). Được `CacheController` gọi để hiển thị trạng thái.

### 4.4 `CacheController.java` — Admin Cache API
- `@RequestMapping("/api/v1/admin/cache")`, yêu cầu role `ADMIN`/`SUPER_ADMIN`.
- Endpoints quan trọng: GET `/names` (danh sách), DELETE `/{name}` (clear 1 cache), DELETE `/clear-all`, DELETE `/{name}/key/{key}` (evict 1 key), GET `/stats` (đếm key `cacheName::*` qua `redisTemplate.keys`), GET `/health` (gọi RedisHealthCheckService + RateLimitService để hiển thị tình trạng), POST `/test-redis`, GET `/config`.

---

## 5. Package `cache` (tổng hợp)

Cache fallback & health check là tính năng quan trọng giúp hệ thống không phụ thuộc cứng vào Redis. `FallbackCacheManager` được đánh dấu `@Primary` trong `CacheConfig`, nên toàn dự án đều dùng bản wrapper này.

---

## 6. Package `audit` — Hệ thống audit logging (nhật ký kiểm toán)

Ghi lại **ai làm gì, khi nào, thay đổi trước/sau như thế nào** đối với dữ liệu nhạy cảm. Kết hợp nhiều tầng: annotation → AOP → service → entity/repo.

### 6.1 Annotation
- `@Auditable` — method-level: khai báo `action`, `entityType`, `description`, `logParameters`, `logResponse`, `logOnlyOnSuccess`. Được `AuditAspect` xử lý. (Phong cách cũ.)
- `@AuditAction` — method-level cho giao dịch CRUD: `type` (`ActionType`), `entity` (tên entity), `actionCode` (nếu trống auto thành `"{TYPE}_{ENTITY}"`). Được `AuditActionAspect` xử lý.
- `@AuditableEntity` — class-level (entity) khai báo `ignoreFields` (danh sách field bỏ qua khi audit, mặc định `id`, `createdAt`, `updatedAt`, `version`, `isDeleted`...). — Hai là điều vol các trường metadata/hệ thống không được log.

### 6.2 Aspect (AOP)
#### `AuditAspect.java` — xử lý `@Auditable`
- `@Around("@annotation(auditable)")` — bọc method:
  - Sinh `requestId` xuyên suốt ng loạt bằng `RequestContextUtil.generateRequestId()`.
  - `@Proceeding` thực thi method.
  - Ghi log entry/success thông lem `auditService.createAuditLog(action + "_STARTED"/action, ...)`.
  - Bắt exception → `logMethodFailure` ghi status `FAILED` + `errorMessage`.
  - `finally` → `RequestContextUtil.clearRequestId()` (dọn ThreadLocal, tránh rò rỉ giữa các thread).
- `extractEntityId(args)`: tự tìm request param là `UserPrincipal` (id user) → `UUID` → số `Long`/`Integer`/chuỗi số để gắn vào `entityId`.

#### `AuditActionAspect.java` — xử lý `@AuditAction` (mới, case tế)
- `@Order(10)` đảm bảo chạy trước một số aspect khác. `@Around`:
  - Gọi `joinPoint.proceed()` trước; sau khi thành công sẽ `switch` theo action type:
    - **CREATE**: đọc `AuditContext.getCreatedEntity()`, build field details (chỉ NEW), tạo audit.
    - **UPDATE**: đọc `AuditContext.getSnapshot()` (trước) và `getUpdatedEntity()` (sau), dùng reflection `field.getDeclaredFields()` so sánh từng field (bỏ `ignoreFields`), chỉ ghi những field khác nhau → chứa. oldValue vs newValue (multiple-field detail).
    - **DELETE**: đọc `getDeletedEntity()` (hoặc fallback tìm UUID trong args) ghi OLD_ONLY.
  - `finally` → `AuditContext.clear()` (luôn dọn ThreadLocal).
- Helper: `resolveActionCode` (`type_entity` hoặc `actionCode` custom), `toSnakeCase` (camel→snake khi sang DB), `serializeValue` (xử lý string/number/enum/collection/entity).

> Điểm cấu thành quan trọng: trong `finally` của cả 2 aspect luôn clear ThreadLocal — tránh thread reuse ở request pool vi phạm dữ liệu audit lẫn nhau.

### 6.3 Context & util
#### `AuditContext.java` — ThreadLocal chống "dùng chung state giữa request"
- thread-local: `SNAPSHOT` (Map field→value trước khi sửa), `CREATED_ENTITY`, `UPDATED_ENTITY`, `DELETED_ENTITY`.
- Các static API cho service: `snapshot(entity)` (chụp trước khi thay đổi), `registerCreated/ registerUpdated / registerDeleted`, và `clear()`.
- `getSnapshot/getCreatedEntity/...` để aspect đọc. Ghi rõ cách dùng trong Javadoc (VD update: snapshot trước khi đổi → sau khi save thì registerUpdated).

#### `RequestContextUtil.java`
- ThreadLocal `REQUEST_ID`, `generateRequestId()` sinh ID dạng `REQ-<millis>-<8hex>`, `getRequestId()` (nếu null sẽ sinh), `clearRequestId()`. Dùng để nhóm nhiều audit log của cùng 1 request.

#### `AuditDiffUtil.java` — helper thao tác thủ công
- Là bộ `@Component` dùng cho các service muốn tự gọi audit (không qua AOP): chứa `snapshot()`, auditUpdate, auditAllChanges, auditCreate, auditDelete. Reflection đọc field, loại field ignorefields, so sánh, tạo `AuditLogDetailDto` list. Giải pháp cho service không muốn dùng annotation.

### 6.4 Domain (entity)
#### `ActionType.java` — enum CREATE/READ/UPDATE/DELETE.
#### `AuditStatus.java` — enum SUCCESS/FAILED/PARTIAL_SUCCESS/CANCELLED.
#### `AuditLog.java` — table `audit_logs`
- Entity chính: `id` UUID (`@UuidGenerator`), `actionType` enum → String, `actionCode`, `entityType` enum → String, `entityId`, `details` (OneToMany lazy), và thông tin diễn viên: `userId`, `username`, `ipAddress`, `userAgent`; request context: `requestId`, `requestMethod`, `requestUrl`; trạng thái: `timestamp`, `status`, `errorMessage`.
- @Table có 6 index hỗ trợ truy vấn (entity, user, actionType, actionCode, requestId, timestamp).
- `@PrePersist onCreate()`: nếu `timestamp`/`status` null → đặt mặc định (`Instant.now()`, `SUCCESS`).

#### `AuditLogDetail.java` — table `audit_log_details`
- Chi tiết field-level: `Long id` (auto), `auditLogId` (FK UUID), `fieldName`, `oldValue`, `newValue` (TEXT).
- `@ManyToOne` đến `AuditLog` với `insertable=false, updatable=false` (không tạo vòng join, chỉ để đọc).

### 6.5 Repository
- `AuditLogRepository` (JpaRepository<AuditLog, UUID>): các find theo user/entity/actionType/actionCode/range; **`findByCriteria`** (filter động bằng `:param IS NULL OR ...`), **`findByAllCriteria`** (kết hợp với field-level detail bằng LEFT JOIN `a.details d`), stats `GROUP BY`, và `findTop1000ByTimestampBefore...` cho cleanup.
- `AuditLogDetailRepository` (JpaRepository<AuditLogDetail, Long>): find theo `auditLogId`, theo `fieldName`/value, `deleteByAuditLogId(In)` cho cleanup.

### 6.6 Service
#### `AuditService.java` — service chính
- Quan trọng: nhiều overload `createAuditLog(...)` (phiên bản mới vs phiên bản cũ legacy string) để tương thích code cũ; tất cả **chuyển dessc và chuyển về phiên bản đầy đủ** `createAuditLog(actionType, actionCode, entityType, entityId, fieldName, oldValue, newValue, status, errorMessage, requestId)`.
- Nếu `!auditProperties.isEnabled()` → return sớm (bỏ qua audit nếu cấu hình tắt).
- `enrichWithContextData(auditLog)`: từ `SecurityContextHolder` lấy user, từ `RequestContextHolder`/`ServletRequest` lấy client IP (ưu tiên `X-Forwarded-For`), User-Agent, method, URL; sinh `requestId` nếu còn thiếu. → tự động thêm context.
- `createAuditLogWithDetails(...)` ghi nhiều field changes: lưu header `AuditLog` rồi `saveAll` các detail.
- Đọc: `searchAuditLogs` quyết định dùng repo nào (có field search hay không), `getAuditLogsByUser/ByEntity/ByFailed`; thống kê `getAuditStatistics...`.
- `truncateValue`: giới hạn dài value theo `auditProperties.getMaxValueLength()`.
- Các `parseActionType/parseEntityType/parseAuditStatus/parseHttpMethod`: chuyển chuỗi legacy → enum (mặc định fallback khi không khớp).

#### `AsyncAuditService.java` — audit bất đồng bộ
- `@ConditionalOnProperty("app.audit.async-logging"`, default true), methods `@Async("auditTaskExecutor")` để không làm chậm request. Nhận context truyền sẵn (vì `SecurityContext`/`RequestContext` không được kế hưởng qua async thread) — ghi đi rồi `auditLogRepository.save`. Cần truyền đủ thông tin user/ip/user-agent/url vào params.

#### `AuditCleanupService.java` — dọn dẹp định kỳ
- `@Scheduled(cron="0 0 2 * * ?")` 2h sáng hàng ngày: xóa log cũ > `retentionDays`; **xóa theo batch 1000** để tránh lỗi truy vấn lớn; log - luu `count`.
- `@Scheduled(fixedRate=3600000)`: log thống kê số lượng (mức tham khảo).

### 6.7 Config (audit)
- `AuditProperties.java` (prefix `app.audit`): `enabled`, `logParameters`, `logResponse`, `logOnlySuccess`, `maxValueLength=10000`, `asyncLogging`, `retentionDays=365`, `compressOldLogs`.— hướng toàn hệ audit.
- `AuditConfiguration.java`: bật `@EnableAspectJAutoProxy`, `@EnableAsync`; bean `auditTaskExecutor` (ThreadPool `corePoolSize=2`, `maxPoolSize=5`, `queueCapacity=100`, prefix "Audit-", rejection handler là `CallerRunsPolicy` m để không mất log khi hàng đợi đầy).

### 6.8 Controller & DTO
- `AuditController.java` (`/api/v1/audit`): các endpoint protected bằng `@RequirePermission(PermissionType.VIEW_AUDIT_LOGS)`: GET `/logs` (tìm kiếm), `/logs/user/{userId}`, `/logs/entity/{type}/{id}`, `/logs/failed`, `/statistics`.
- DTO: `AuditLogDto`, `AuditLogDetailDto`, `AuditLogSearchRequest`, `AuditStatisticDto`. Lưu ý `AuditLogDto` chứa cả field "new structured" lẫn các field backward-compat (`fieldName`, `oldValue`, `newValue`) được `convertToDto` điền từ detail đầu tiên.

---

## 7. Package `persistence/encryption` — Mã hóa dữ liệu nhạy cảm trước khi lưu DB

### 7.1 `AesGcmStringEncryptor.java` — lõi mật mã AES-GCM
- Dùng **AES/GCM/NoPadding** — GCM vừa mã hóa cả **xác thực** (tag 128-bit) chống bị sửa.
- đọc key từ `app.security.encryption.key-base64`; nếu trống → tự sinh 256-bit (`loadOrGenerateKey`). Cho phép key 16/24/32 bytes.
- `encrypt(plainText)`: tạo IV 12 bytes ngẫu nhiên mỗi lần (an toàn), mã hóa, đặt `[IV][cipherBytes]` vào 1 buffer rồi Base64.→ **1 ngẫu nhiên IV/encrypt → mỗi lần encrypt cùng văn bản cho ciphertext khác nhau** (an toàn chống dò pattern).
- `decrypt(encryptedText)`: Base64 decode; **tự xử lý tương thích dữ liệu cũ**: nếu không decode được (legacy plaintext) hoặc đầy đủ kích thước không đủ IV → trả chính chuỗi (không coi là lỗi). Gặp lỗi khác → log + return null.

### 7.2 `EncryptedStringConverter.java` — JPA AttributeConverter cho String
- `@Converter` + `@Component` (tự nạp encryptor qua setter static). `convertToDatabaseColumn` → `encryptor.encrypt(attribute)`; `convertToEntityAttribute` → `encryptor.decrypt(dbData)`. Dùng cho mang mọi field nhạy cảm (VD: SĐT, CMND, BHYT) bằng `@Convert(converter = EncryptedStringConverter.class)`.

### 7.3 `EncryptedLocalDateConverter.java` — cho LocalDate
- Entity cũng là `AttributeConverter<LocalDate, String>` — lưu dạng chuỗi ngày đã mã hóa. Đọc: decrypt rồi `LocalDate.parse`. Xử lý trường hợp dữ liệu plaintext cũ (không mã hóa) bằng cách thử trực tiếp parse khi encryptor chưa được nạp hoặc decrypt trả null.

> Lưu ý kỹ thuật trong cả 2 converter: dùng `setEncryptor` qua `@Autowired` gắn vào 1 `static` field vì JPA có thể tạo instance converter không qua container trong một số trường hợp.

---

## 8. Package `service`, `util` — Tiện ích tổng

### 8.1 `service/CloudinaryService.java`
- `upload(MultipartFile file, String folder)`: gọi Cloudinary upload với `resource_type: "auto"`, lấy `secure_url` → trả về public URL của file tải lên. Dùng để upload ảnh avatar nhân viên, file đính kèm notification,... Ném `ValidationException("Upload file thất bại")` khi lỗi IO.

### 8.2 `util/EnvironmentUtil.java`
- `@Component` bọc Spring `Environment`, `getString/getInt/getLong/getBoolean` với defaults, `getRequiredString` (thiếu → throw có ý nghĩa), `getActiveProfile`, `isDevelopment/isProduction`.

### 8.3 `util/IpAddressUtil.java`
- Static method `getClientIp(HttpServletRequest)`: duyệt headers proxy (`X-Forwarded-For`, `X-Real-IP`, `Proxy-Client-IP`, `WL-Proxy-Client-IP`, `HTTP_X_FORWARDED_FOR`, `HTTP_CLIENT_IP`), nếu có và khác "unknown" → lấy IP đầu tiên (vì `X-Forwarded-For` có thể có nhiều). Tránh bị lừa IP khi có reverse proxy.

### 8.4 `util/FaceApiClient.java` — gọi Python Face Recognition API (quan trọng cho chấm công)
- Khởi tạo `RestTemplate`, `baseUrl` từ `face-api.base-url` (default http://localhost:8000).
- `verifyFace(MultipartFile photo, String employeeId)`: POST multipart `/face/verify`, parse JSON, đọc `data.matched` bool. — dùng để cho xác minh khuôn mặt khi chấm công.
- `isFaceRegistered/employeeId)`, `getFaceStatus` : GET `/face/status/{employeeId}` → `data.registered`.
- `registerFace(video, employeeId)`: PUT multipart `/face/register` trả job info.
- `getRegistrationJobStatus(jobId)`: GET `/face/register/status/{jobId}`.
- Phân bố logging + ném `RuntimeException` thông báo tiếng Việt (có ý giảm false). Các method getStatus/status trả Map với `registered=false` khi lỗi (không phát trap exception cho chấm công).

---

## 9. Package `constant`, `domain`, `web`, `security` — Hằng số & enum dùng chung

### `constant/PermissionType.java` — Hằng số quyền (quan trọng cho RBAC)
- Rất nhiều hằng số string dùng làm quyền trong DB + `@RequirePermission` (VD `VIEW_PROFILE`, `MANAGE_USER`, `VIEW_EMPLOYEE`, `DELETE_PAYROLL`...).
- Class `PermissionDef { code, description, resource, action }` — metadata.
- **`all()`**: trả danh sách `PermissionDef` cho toàn bộ quyền — dung để **seed dữ liệu** vào DB khi khởi động (đặt ở one place). `descriptions()`: map code→desc.

### `domain/EntityType.java`
- Enum liệt kê các loại entity (`USER`, `EMPLOYEE`, `ROLE`, `DEPARTMENT`, `ATTENDANCE`, `LEAVE_REQUEST`, `PAYROLL`, `CONTRACT`, `WORK_SHIFT`, `INSURANCE`, `DEGREE`, `RELATIVE`, `STORE`, `SETTING`...) Dùng cho audit (phân biệt log), thống kê, v.v.

### `web/HttpMethod.java`
- Enum HTTP methods (`GET`, `POST`, ...) dùng cho audit (`requestMethod`).

### `security/SecureEndpoint.java`
- Annotation method/class: dùng để đánh dấu endpoint cần bảo vệ RBAC.
- Với `@ConditionalOnProperty("app.security.rbac.enabled", havingValue="true", matchIfMissing=true)` — annotation chỉ "hiệu lực" khi bật RBAC (đúng theo SecurityProperties).

---

## 10. Cuối cùng: vai trò & vị trí trong kiến trúc tổng

- Module `shared` là **tầng hạ tầng (infrastructure) chung**, được toàn bộ module nghiệp vụ (auth, user, employee, attendance, payroll, ...) sử dụng thông qua:
  - `ApiResponse`/`PaginatedApiResponse` → controller trả về.
  - `BaseException`/`GlobalExceptionHandler` → service ném lỗi chuẩn.
  - `AuditAction`/`AuditContext` + `@AuditableEntity` → ghi kiểm toán.
  - `@RateLimit` + `RateLimitFilter` → chống lạm dụng endpoint.
  - `@Cacheable` đi qua `FallbackCacheManager` → cache Redis có chế độ fallback sang DB.
  - `@Convert(converter = EncryptedStringConverter)` → mã hóa dữ liệu nhạy cảm.
  - `CloudinaryService`, `FaceApiClient`, `PermissionType`, `IpAddressUtil`, `EnvironmentUtil` → tiện ích.

Thiết kế đặt nhiều nguyên tắc quan trọng: **chuẩn hóa đáp ứng API, xử lý lỗi tập trung, giảm sự phụ thuộc vào Redis (graceful degradation), ghi audit rõ ràng trước/sau thay đổi, và mã hóa dữ liệu nhạy cảm khi lưu trữ**.