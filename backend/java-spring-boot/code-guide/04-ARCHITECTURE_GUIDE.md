# Phần 4: Kiến trúc code

## 4.1. Sơ đồ luồng request

```
HTTP Request
   │
   ▼
Security Filter Chain (JWT xác thực + Rate limit)
   │
   ▼
@RequirePermission (AOP kiểm tra quyền RBAC)
   │
   ▼
Controller  →  nhận request, gọi Service, bọc ApiResponse
   │
   ▼
Service  →  logic nghiệp vụ (@Transactional, @Cacheable, @AuditAction)
   │
   ▼
Repository  →  Spring Data JPA, thao tác DB
   │
   ▼
Database (PostgreSQL) + Redis (cache)
   │
   ▼
Trả ngược về client: ApiResponse<T> / PaginatedApiResponse<T>
```

## 4.2. Trách nhiệm từng tầng (lấy module `employee` làm ví dụ thực tế)

### Controller — `employee/controller/EmployeeController.java`

- `@RestController` + `@RequestMapping("/api/v1/employees")` — khai báo endpoint.
- **Không chứa logic nghiệp vụ**, chỉ:
  - nhận tham số (`@PathVariable`, `@RequestParam`, `@RequestBody`, `@PageableDefault`),
  - lấy user đang đăng nhập qua `@AuthenticationPrincipal UserPrincipal`,
  - gọi Service,
  - bọc kết quả vào `ApiResponse` / `PaginatedApiResponse`.
- Đánh quyền trên từng endpoint bằng `@RequirePermission(PermissionType.XXX)`.

Ví dụ:

```java
@GetMapping("/{employeeId}")
@RequirePermission(PermissionType.VIEW_EMPLOYEE)
public ResponseEntity<ApiResponse<EmployeeResponse>> getEmployeeById(@PathVariable UUID employeeId) {
    EmployeeResponse response = employeeService.getEmployeeById(employeeId);
    return ResponseEntity.ok(ApiResponse.success(response));
}
```

### Service — `employee/service/EmployeeService.java`

- `@Service` — tầng chứa **logic nghiệp vụ**.
- `@Transactional` trên các method ghi (create/update/delete) — đảm bảo atomic.
- **Không trả Entity ra ngoài**: luôn trả DTO (`EmployeeResponse`).
- Thường gọi nhiều Repository (Employee + Department + Store + User).
- Ném exception chuẩn khi không tìm thấy/trùng dữ liệu (→ bị `GlobalExceptionHandler` bắt).
- Đánh dấu các annotation cắt ngang:
  - `@Cacheable` / `@CacheEvict` — cache Redis.
  - `@AuditAction(type = ..., entity = "EMPLOYEE")` — ghi audit log (kèm `AuditContext.snapshot/register...`).

Ví dụ:

```java
@Transactional
@CacheEvict(value = CacheConfig.EMPLOYEE_CACHE, allEntries = true)
@AuditAction(type = ActionType.CREATE, entity = "EMPLOYEE")
public EmployeeResponse createEmployee(CreateEmployeeRequest request, UUID createdBy) {
    if (employeeRepository.existsActiveByEmployeeCode(request.getEmployeeCode())) {
        throw ValidationException.duplicateField("employee_code", request.getEmployeeCode());
    }
    Employee employee = Employee.builder()...build();
    ...
    return employeeMapper.toEmployeeResponse(employeeRepository.save(employee));
}
```

### Repository — `employee/repository/EmployeeRepository.java`

- Kế thừa `JpaRepository<Employee, UUID>` → có sẵn CRUD.
- Method tùy chỉnh theo quy tắc đặt tên của Spring Data, và luôn đi kèm **soft delete**:
  - `findActiveById`, `findAllActive`, `findActiveByDepartmentId`, `searchActive`, `existsActiveByEmail`...
- Trả về `Optional<...>` / `Page<...>` để Service xử lý.

### Mapper — `employee/mapper/EmployeeMapper.java`

- Dùng **MapStruct** (`@Mapper(componentModel = "spring")`).
- Tự sinh code chuyển đổi: `toEntity`, `toResponse`, `updateEntity`.

```java
@Mapper(componentModel = "spring")
public interface EmployeeMapper {
    EmployeeResponse toEmployeeResponse(Employee employee);
}
```

### Entity — `employee/domain/Employee.java`

- `@Entity` ánh xạ bảng DB.
- Kế thừa base entity có auditing (`createdAt`, `updatedAt`...) và field soft delete (`isDeleted`, `deletedAt`, `deletedBy`).
- Field nhạy cảm (VD số CCCD, số tài khoản) đánh dấu `@Convert` để **mã hoá khi lưu**.

### DTO — `employee/dto/request` + `employee/dto/response`

- `request/` — nhận input từ client, dùng `@Valid` + validation annotation (`@NotBlank`...).
- `response/` — trả về client, chỉ chứa field cần thiết.

## 4.3. Một request đi qua project như thế nào

Ví dụ `GET /api/v1/employees/{id}` (xem employee theo ID):

1. Request vào **Security Filter Chain** → nếu thiếu/trái phép JWT → trả 401.
2. Filter `@RequirePermission(VIEW_EMPLOYEE)` (AOP) kiểm tra user có quyền → nếu không → 403.
3. **Controller** `getEmployeeById` đón `@PathVariable`, gọi `employeeService.getEmployeeById(id)`.
4. **Service** `getEmployeeById`:
   - hỏi **cache Redis** trước (nhờ `@Cacheable`) — nếu có sẵn thì không cần query DB;
   - gọi `employeeRepository.findActiveById(id)` (query có điều kiện `isDeleted = false`);
   - nếu không có → ném `EmployeeNotFoundException` (→ handler trả 404);
   - map Entity → `EmployeeResponse`.
5. **Controller** bọc vào `ApiResponse.success(response)` → client nhận JSON chuẩn.

## 4.4. Quy tắc viết code mới

Khi thêm **một module mới** (VD `salary`), tạo đủ 6 package theo đúng chuẩn dự án:

```
salary/
├── controller/SalaryController.java
├── dto/request/SalaryRequest.java
├── dto/response/SalaryResponse.java
├── domain/Salary.java
├── mapper/SalaryMapper.java
├── repository/SalaryRepository.java
└── service/SalaryService.java
```

Quy trình gợi ý khi code 1 API:

1. Tạo **Entity** trước (ánh xạ bảng).
2. Tạo **DTO request/response** + validation.
3. Tạo **Mapper** (MapStruct).
4. Tạo **Repository** + query.
5. Tạo **Service** (logic + `@Transactional` + exception + audit/cache).
6. Tạo **Controller** + `@RequirePermission` + bọc `ApiResponse`.
7. Thêm endpoint vào `app.security.public-endpoints` nếu là API công khai.

## 4.5. Thứ tự viết code

> Lưu ý: thứ tự file trong `00-GUIDE_INDEX.md` (01 → 11) là **thứ tự học/đọc**, không phải thứ tự viết code.

### a) Thứ tự dựng dự án từ đầu

Khi code cả project từ con số 0, làm theo thứ tự **từ hạ tầng → xác thực → nghiệp vụ**:

1. **Cấu hình & shared** — `application.properties`, `shared/` (base entity, exception, `ApiResponse`, `GlobalExceptionHandler`, `BaseService`...).
2. **`user` + `auth`** — tạo user, đăng nhập JWT, refresh token (mọi API khác đều cần xác thực).
3. **`authorization`** — role, permission, RBAC (`@RequirePermission`).
4. **Module nền tảng** — `department`, `store`, `workshift` (dữ liệu cha, module khác tham chiếu tới).
5. **Module nghiệp vụ chính** — `employee` → `contract` → `insurance` → `degree` → `relative` (thuộc employee).
6. **Module vận hành** — `attendance`, `leave`, `payroll` (tính toán, phụ thuộc dữ liệu ở trên).
7. **`notification`** — gửi thông báo, background job (`NotificationScheduler`).

### b) Thứ tự viết code trong mỗi module (thao tác hằng ngày)

Trong một module/tính năng, code **từ dưới lên theo từng lớp**:

1. **Domain** — Entity (`Employee`).
2. **DTO** — `dto/request` + `dto/response` (+ validation).
3. **Mapper** — MapStruct (`@Mapper(componentModel = "spring")`).
4. **Repository** — `JpaRepository` + method truy vấn soft delete.
5. **Service** — logic nghiệp vụ + `@Transactional` + exception + `@Cacheable`/`@AuditAction`.
6. **Controller** — endpoint + `@RequirePermission` + bọc `ApiResponse`.
7. **Test** — unit test (viết sau khi Service xong).

Lý do: Service gọi Repository, Repository dùng Entity, Mapper dùng DTO... nên cứ viết hết tầng dữ liệu trước, rồi tầng Service, cuối cùng mới đến tầng trình bày (Controller).

## 4.6. Điểm đặc trưng của dự án khi code

- **Không bao giờ trả Entity ra client** — luôn qua DTO + Mapper.
- **Xoá là soft delete** — đặt `isDeleted = true` + `deletedAt`, không `DELETE` cứng; mọi query phải lọc `isDeleted = false`.
- **Exception ném trong Service, xử lý tập trung ở `GlobalExceptionHandler`** — không bắt exception rải rác.
- **Request ghi (POST/PUT/DELETE) cần `@Transactional`** trong Service.
- **Mã hoá field nhạy cảm** (thừa kế cấu hình trong `shared/persistence/encryption`).
- Tài liệu API tự sinh qua Swagger (`/swagger-ui.html`).
