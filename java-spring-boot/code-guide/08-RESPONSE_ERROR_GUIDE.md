# Phần 8: Response chuẩn & Xử lý lỗi

Hướng dẫn về format response và cơ chế xử lý exception tập trung của dự án. Đây là phần mà **mọi API đều phải tuân theo**.

## 8.1. Response chuẩn — `shared/dto/ApiResponse.java`

Mọi endpoint trả về **`ApiResponse<T>`**:

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)   // field null KHÔNG xuất hiện trong JSON
public class ApiResponse<T> {
    private boolean success;
    private String code;
    private String message;
    private T data;
    private List<String> errors;
    @Builder.Default
    private Instant timestamp = Instant.now();
}
```

### Các factory method

| Method | Khi dùng | Kết quả |
| ------ | -------- | ------- |
| `ApiResponse.success(data)` | Thành công, không cần message | `success: true` + `data` |
| `ApiResponse.success(message, data)` | Thành công + thông báo | `success: true` + `message` + `data` |
| `ApiResponse.error(message)` | Lỗi đơn giản | `success: false` + `message` |
| `ApiResponse.error(message, errors)` | Lỗi kèm danh sách (validation) | `success: false` + `message` + `errors` |
| `ApiResponse.error(code, message, data)` | Lỗi có mã + dữ liệu bổ sung | `success: false` + `code` + `message` + `data` |

### JSON thành công (không phân trang)

```json
{
  "success": true,
  "message": "Employee created successfully",
  "data": {
    "id": "550e8400-...",
    "employeeCode": "EMP001",
    "fullName": "Van A Nguyen"
  },
  "timestamp": "2026-08-07T10:00:00Z"
}
```

> `code` và `errors` không xuất hiện vì null (do `NON_NULL`).

## 8.2. Response phân trang — `PaginatedApiResponse.java` + `PageInfo.java`

Endpoint trả danh sách dùng **`PaginatedApiResponse<T>`**:

```java
public class PaginatedApiResponse<T> {
    private boolean success;
    private String code;
    private String message;
    private List<T> data;          // là LIST, không phải object
    private PageInfo pagination;
    private Instant timestamp;
}
```

`PageInfo` lấy trực tiếp từ `Page<T>` của Spring:

```java
public static PageInfo of(Page<?> page) {
    return PageInfo.builder()
            .page(page.getNumber())      // bắt đầu từ 0
            .size(page.getSize())
            .totalItems(page.getTotalElements())
            .totalPages(page.getTotalPages())
            .build();
}
```

### Cách dùng trong Controller

```java
@GetMapping
public ResponseEntity<PaginatedApiResponse<EmployeeResponse>> getAllEmployees(
        @PageableDefault(size = 10) Pageable pageable) {
    Page<EmployeeResponse> response = employeeService.getAllActiveEmployees(pageable);
    return ResponseEntity.ok(PaginatedApiResponse.success(response));   // nhận trực tiếp Page
}
```

### JSON phân trang

```json
{
  "success": true,
  "data": [ { "id": "...", "employeeCode": "EMP001" } ],
  "pagination": {
    "page": 0,
    "size": 10,
    "totalItems": 57,
    "totalPages": 6
  },
  "timestamp": "2026-08-07T10:00:00Z"
}
```

> Lưu ý: `page` bắt đầu từ **0**, `totalPages` làm tròn lên.

## 8.3. `ErrorResponse` — `shared/dto/ErrorResponse.java`

Tồn tại như DTO lỗi có cấu trúc (status, error, message, timestamp, path). Hiện tại `GlobalExceptionHandler` dùng `ApiResponse.error(...)` làm format chính; `ErrorResponse` dùng cho các trường hợp cần chi tiết hơn.

## 8.4. Exception gốc — `shared/exception/BaseException.java`

Tất cả exception nghiệp vụ đều kế thừa `BaseException`:

```java
public abstract class BaseException extends RuntimeException {
    private final HttpStatus status;     // HTTP status của lỗi
    private final String errorCode;      // mã lỗi nghiệp vụ
}
```

Ví dụ một exception cụ thể:

```java
public class EmployeeNotFoundException extends BaseException {
    private static final String ERROR_CODE = "EMPLOYEE_NOT_FOUND";

    public EmployeeNotFoundException(String message) {
        super(message, HttpStatus.NOT_FOUND, ERROR_CODE);   // 404
    }

    public static EmployeeNotFoundException byId(String id) {
        return new EmployeeNotFoundException("Employee not found with id: " + id);
    }
}
```

### Danh sách exception có sẵn (trong `shared/exception`)

| Exception | HTTP | Khi nào ném |
| --------- | ---- | ----------- |
| `ResourceNotFoundException` + các biến thể (`EmployeeNotFoundException`, `DepartmentNotFoundException`...) | 404 | Không tìm thấy tài nguyên |
| `ValidationException` | 400 | Dữ liệu sai/trùng |
| `DuplicateResourceException` / `EmailAlreadyExistsException` | 409 | Trùng dữ liệu |
| `BusinessRuleViolationException` | 400 | Vi phạm nghiệp vụ |
| `InvalidCredentialsException` | 401 | Sai email/mật khẩu |
| `InvalidTokenException` | 401 | Token sai/hết hạn |
| `RefreshTokenException` | 401 | Refresh token hết hạn/đã thu hồi |
| `UserInactiveException` | 403 | User bị vô hiệu |
| `RateLimitExceededException` / `RateLimitBannedException` | 429 | Vượt giới hạn request |
| `OAuth2ValidationException` | 400 | Thiếu thông tin OAuth2 |

## 8.5. GlobalExceptionHandler — `shared/exception/GlobalExceptionHandler.java`

**Tất cả exception được bắt tập trung ở đây** — Service/Controller chỉ ném, không bắt rải rác.

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
```

### Bảng các handler

| Handler | Ném khi | Status | Body |
| ------- | ------- | ------ | ---- |
| `handleBaseException` | mọi `BaseException` | `ex.getStatus()` | `ApiResponse.error(message)` |
| `handleValidationExceptions` | `@Valid` fail | 400 | `message="Validation failed"` + `errors=[...]` |
| `handleAuthenticationExceptions` | `BadCredentialsException`, `UsernameNotFoundException` | 401 | `"Invalid email or password"` |
| `handleExpiredJwtException` | JWT hết hạn | 401 | `"Token has expired"` |
| `handleJwtException` | JWT sai định dạng | 401 | `"Invalid token"` |
| `handleIllegalArgumentException` | tham số sai | 400 | `ex.getMessage()` |
| `handleNoResourceFoundException` | URL không tồn tại | 404 | `"Resource not found"` |
| `handleRateLimitExceededException` | vượt rate limit | 429 | message + header `Retry-After` |
| `handleRateLimitBannedException` | bị ban | 429 | message + header `Retry-After` |
| `handleAccessDeniedException` | thiếu quyền | 403 | `ex.getMessage()` |
| `handleRuntimeException` | lỗi runtime không lường trước | 500 | `"An internal server error occurred: ..."` |
| `handleGenericException` | mọi lỗi còn lại | 500 | `"Internal server error"` |

### Ví dụ JSON lỗi

**404 — không tìm thấy:**

```json
{
  "success": false,
  "message": "Employee not found with id: 550e8400-...",
  "timestamp": "2026-08-07T10:00:00Z"
}
```

**400 — validation:**

```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    "Employee code is required",
    "Email must be valid"
  ],
  "timestamp": "2026-08-07T10:00:00Z"
}
```

**401 — chưa đăng nhập / token hết hạn** (do `CustomAuthenticationEntryPoint`):

```json
{
  "success": false,
  "code": "UNAUTHORIZED",
  "message": "Bạn cần đăng nhập để truy cập tài nguyên này",
  "timestamp": "2026-08-07T10:00:00Z"
}
```

**403 — thiếu quyền** (do `CustomAccessDeniedHandler`):

```json
{
  "success": false,
  "code": "FORBIDDEN",
  "message": "Bạn không có quyền truy cập tài nguyên này",
  "timestamp": "2026-08-07T10:00:00Z"
}
```

## 8.6. Cách ném exception đúng chuẩn trong Service

### 1. Tài nguyên không tồn tại → dùng factory method

```java
Employee employee = employeeRepository.findActiveById(id)
        .orElseThrow(() -> EmployeeNotFoundException.byId(id.toString()));
```

### 2. Dữ liệu trùng → `ValidationException.duplicateField`

```java
if (employeeRepository.existsActiveByEmployeeCode(request.getEmployeeCode())) {
    throw ValidationException.duplicateField("employee_code", request.getEmployeeCode());
}
```

### 3. Vi phạm nghiệp vụ → `BusinessRuleViolationException`

```java
throw new BusinessRuleViolationException("Cannot delete department that still has employees");
```

### 4. Chưa có exception phù hợp → tạo mới

```java
public class SalaryNotFoundException extends BaseException {
    private static final String ERROR_CODE = "SALARY_NOT_FOUND";
    public SalaryNotFoundException(String message) {
        super(message, HttpStatus.NOT_FOUND, ERROR_CODE);
    }
    public static SalaryNotFoundException byId(String id) {
        return new SalaryNotFoundException("Salary not found with id: " + id);
    }
}
```

> Không cần sửa gì ở `GlobalExceptionHandler` — `BaseException` đã được `handleBaseException` xử lý tự động (status lấy từ `ex.getStatus()`).

## 8.7. Nguyên tắc viết response/error

1. **Thành công** → Controller trả `ApiResponse.success(...)` hoặc `PaginatedApiResponse.success(page)`.
2. **Lỗi** → ném exception từ Service; để `GlobalExceptionHandler` lo phần trả về. Không tự catch trong Controller.
3. **Không trả `null`** khi có lỗi — luôn ném exception.
4. **`@Valid`** trên `@RequestBody` để validation tự chạy, không cần tự kiểm tra thủ công.
5. **HTTP status đúng**: 200 OK, 201 Created (tạo mới), 400 lỗi dữ liệu, 401 chưa đăng nhập, 403 thiếu quyền, 404 không thấy, 409 trùng dữ liệu, 429 quá nhiều request.
6. **`code`/`errors` chỉ xuất hiện khi có giá trị** (do `@JsonInclude(NON_NULL)`).

## 8.8. Checklist khi code 1 API

- [ ] Response bọc `ApiResponse<T>` / `PaginatedApiResponse<T>`.
- [ ] Tạo mới trả `201 CREATED`.
- [ ] `@Valid` trên `@RequestBody`.
- [ ] Không tìm thấy → ném `XxxNotFoundException` (→ 404).
- [ ] Trùng dữ liệu → `ValidationException.duplicateField` (→ 400).
- [ ] Không để exception trần thoát ra ngoài Service.
- [ ] Kiểm tra trường hợp token thiếu (401) / thiếu quyền (403) nếu endpoint có phân quyền.
