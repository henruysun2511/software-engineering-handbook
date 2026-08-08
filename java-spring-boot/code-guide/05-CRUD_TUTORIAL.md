# Phần 5: Tạo API CRUD mẫu (Tutorial chi tiết)

Hướng dẫn từng bước dựng một API CRUD hoàn chỉnh, lấy **module `employee`** làm mẫu. Đọc xong phần này bạn có thể tự dựng module mới (VD `salary`) theo đúng chuẩn dự án.

> Trước khi bắt đầu, nên đọc `04-ARCHITECTURE_GUIDE.md` để nắm trách nhiệm từng tầng.

## 5.1. Tổng quan: các file cần tạo

Một module CRUD có đủ **7 file** theo thứ tự xây dựng:

```
employee/
├── 1. domain/Employee.java          → Entity (ánh xạ bảng)
├── 2. dto/request/CreateEmployeeRequest.java
├── 2. dto/request/UpdateEmployeeRequest.java
├── 2. dto/response/EmployeeResponse.java
├── 3. mapper/EmployeeMapper.java    → MapStruct
├── 4. repository/EmployeeRepository.java
├── 5. service/EmployeeService.java  → logic nghiệp vụ
└── 6. controller/EmployeeController.java
```

Quy trình đề xuất: **Entity → DTO → Mapper → Repository → Service → Controller → (bổ sung) → test**.

---

## 5.2. Bước 1: Tạo Entity (`domain/Employee.java`)

### Điểm cần nắm

1. **Bảng**: `@Table(name = "employees")` — tên bảng số nhiều, snake_case.
2. **Khóa chính**: UUID sinh tự động:

```java
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id;
```

3. **Audit + soft delete**: mọi entity cần các field:

```java
@CreationTimestamp
@Column(name = "created_at")
private Instant createdAt;

@UpdateTimestamp
@Column(name = "updated_at")
private Instant updatedAt;

@Column(name = "created_by")
private UUID createdBy;

@Column(name = "updated_by")
private UUID updatedBy;

@Column(name = "is_deleted")
@Builder.Default
private Boolean isDeleted = false;

@Column(name = "deleted_at")
private Instant deletedAt;

@Column(name = "deleted_by")
private UUID deletedBy;
```

4. **Audit log cho entity**: gắn `@AuditableEntity` với danh sách field **không** cần diff:

```java
@AuditableEntity(ignoreFields = {
    "id", "updatedAt", "updatedBy", "createdAt", "createdBy",
    "isDeleted", "deletedAt", "deletedBy"
})
```

5. **Enum**: lưu dạng chuỗi để dễ đọc:

```java
@Enumerated(EnumType.STRING)
@Column(name = "gender", length = 10)
private Gender gender;
```

6. **Quan hệ**: `@ManyToOne` với `FetchType.LAZY` (tránh N+1, tránh load toàn bộ):

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "department_id")
private Department department;
```

7. **Mã hóa field nhạy cảm** (CCCD, số tài khoản, địa chỉ...):

```java
@Convert(converter = EncryptedStringConverter.class)
@Column(name = "id_card_number", length = 512)
private String idCardNumber;
```

8. **Lombok**: `@Data @Builder @NoArgsConstructor @AllArgsConstructor` — Builder rất quan trọng, Service dùng `Employee.builder()...build()`.

9. **Giá trị mặc định** bằng `@Builder.Default` (VD trạng thái mặc định ACTIVE).

### Lưu ý khi tạo Entity

- Mọi `@Column` đặt rõ tên cột (snake_case) và `length`.
- Cột `unique` đánh dấu để DB ràng buộc: `@Column(unique = true)`.
- Field enum không `null` nên có default qua `@Builder.Default`.

---

## 5.3. Bước 2: Tạo DTO (`dto/request` + `dto/response`)

### Request — `dto/request/CreateEmployeeRequest.java`

- Chứa **input từ client**, kèm validation bằng Bean Validation (`jakarta.validation.*`).
- Tên field theo camelCase của request, Service tự map sang Entity.
- Khác Entity ở chỗ: quan hệ chỉ truyền `UUID` (departmentId, storeId, userId), không nhúng object.

```java
@NotBlank(message = "Employee code is required")
@Size(max = 20)
private String employeeCode;

@NotBlank(message = "Email is required")
@Email(message = "Invalid email format")
@Size(max = 100)
private String email;
```

### Request — `dto/request/UpdateEmployeeRequest.java`

- **Mọi field optional** (không `@NotBlank`): client chỉ gửi field muốn sửa, Service check `!= null` trước khi set.
- Enum truyền dưới dạng **String** (`gender`, `employmentStatus`) → Service tự `Enum.valueOf(...)`.

### Response — `dto/response/EmployeeResponse.java`

- Chỉ chứa field cần trả ra client.
- Quan hệ trả về `UUID` + tên hiển thị (VD `departmentId`, `departmentName`), không trả object đầy đủ.
- Không có các field nội bộ như `isDeleted`, `deletedAt`...

---

## 5.4. Bước 3: Tạo Mapper (`mapper/EmployeeMapper.java`)

MapStruct tự sinh code chuyển đổi lúc biên dịch. `componentModel = "spring"` để Spring tạo bean.

```java
@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
public interface EmployeeMapper {

    @Mapping(target = "userId", source = "user.id")
    @Mapping(target = "departmentId", expression = "java(employee.getDepartment() != null ? employee.getDepartment().getId() : null)")
    @Mapping(target = "departmentName", expression = "java(employee.getDepartment() != null ? employee.getDepartment().getName() : null)")
    @Mapping(target = "gender", expression = "java(employee.getGender() != null ? employee.getGender().name() : null)")
    EmployeeResponse toEmployeeResponse(Employee employee);
}
```

### Giải thích

- Field cùng tên (employeeCode, email...) MapStruct tự map.
- Field khác tên dùng `@Mapping(target = ..., source = ...)`.
- Field cần xử lý null/enum dùng `expression = "java(...)"`.
- File `lombok-mapstruct-binding` trong `build.gradle` giúp MapStruct hiểu getter/setter của Lombok.

---

## 5.5. Bước 4: Tạo Repository (`repository/EmployeeRepository.java`)

```java
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, UUID> {
    ...
}
```

### Nguyên tắc soft delete

**Mọi query phải lọc `isDeleted = false`** — dùng JPQL `@Query`:

```java
@Query("SELECT e FROM Employee e WHERE e.id = :id AND e.isDeleted = false")
Optional<Employee> findActiveById(@Param("id") UUID id);

@Query("SELECT e FROM Employee e WHERE e.isDeleted = false")
Page<Employee> findAllActive(Pageable pageable);
```

### Các dạng method

| Dạng | Ví dụ | Trả về |
| ---- | ----- | ------ |
| Tìm theo id | `findActiveById` | `Optional<Employee>` |
| Liệt kê phân trang | `findAllActive(Pageable)` | `Page<Employee>` |
| Tìm theo quan hệ | `findActiveByDepartmentId(id, pageable)` | `Page<Employee>` |
| Kiểm tra tồn tại | `existsActiveByEmail(email)` | `boolean` |
| Tìm kiếm LIKE nhiều cột | `searchActive(keyword, pageable)` | `Page<Employee>` |
| Đếm | `countByIsDeletedFalse()` | `long` |

**Phân trang** luôn dùng `Pageable` (do client truyền, Controller khai báo `@PageableDefault`).

---

## 5.6. Bước 5: Tạo Service (`service/EmployeeService.java`)

Đây là tầng quan trọng nhất. Trách nhiệm:

### a) Tiêm phụ thuộc bằng `@RequiredArgsConstructor` (constructor injection)

```java
@Service
@RequiredArgsConstructor
public class EmployeeService {

    private final EmployeeRepository employeeRepository;
    private final DepartmentRepository departmentRepository;
    private final EmployeeMapper employeeMapper;
}
```

### b) Đọc dữ liệu — không cần `@Transactional`, có thể thêm `@Cacheable`

```java
@Cacheable(value = CacheConfig.EMPLOYEE_CACHE, key = "#employeeId.toString()")
public EmployeeResponse getEmployeeById(UUID employeeId) {
    Employee employee = employeeRepository.findActiveById(employeeId)
            .orElseThrow(() -> EmployeeNotFoundException.byId(employeeId.toString()));
    return employeeMapper.toEmployeeResponse(employee);
}

public Page<EmployeeResponse> getAllActiveEmployees(Pageable pageable) {
    return employeeRepository.findAllActive(pageable)
            .map(employeeMapper::toEmployeeResponse);
}
```

> `orElseThrow` ném exception → `GlobalExceptionHandler` trả 404. Không bao giờ trả về `null`.

### c) Create — `@Transactional` + `@CacheEvict` + `@AuditAction`

```java
@Transactional
@CacheEvict(value = CacheConfig.EMPLOYEE_CACHE, allEntries = true)
@AuditAction(type = ActionType.CREATE, entity = "EMPLOYEE")
public EmployeeResponse createEmployee(CreateEmployeeRequest request, UUID createdBy) {
    // 1. Kiểm tra trùng dữ liệu
    if (employeeRepository.existsActiveByEmployeeCode(request.getEmployeeCode())) {
        throw ValidationException.duplicateField("employee_code", request.getEmployeeCode());
    }
    if (employeeRepository.existsActiveByEmail(request.getEmail())) {
        throw ValidationException.duplicateField("email", request.getEmail());
    }

    // 2. Build Entity từ request (chỉ set field cần thiết)
    Employee employee = Employee.builder()
            .employeeCode(request.getEmployeeCode())
            .firstName(request.getFirstName())
            .lastName(request.getLastName())
            .fullName(request.getLastName() + " " + request.getFirstName())
            .email(request.getEmail())
            .createdBy(createdBy)
            .build();

    // 3. Gắn quan hệ (nếu có) — lấy entity thật, không set id
    if (request.getDepartmentId() != null) {
        Department department = departmentRepository.findActiveById(request.getDepartmentId())
                .orElseThrow(() -> DepartmentNotFoundException.byId(request.getDepartmentId().toString()));
        employee.setDepartment(department);
    }

    // 4. Lưu
    Employee saved = employeeRepository.save(employee);
    AuditContext.registerCreated(saved);

    // 5. Trả DTO, không trả Entity
    return employeeMapper.toEmployeeResponse(saved);
}
```

### d) Update — snapshot trước khi sửa để audit diff

```java
@Transactional
@CacheEvict(value = CacheConfig.EMPLOYEE_CACHE, allEntries = true)
@AuditAction(type = ActionType.UPDATE, entity = "EMPLOYEE")
public EmployeeResponse updateEmployee(UUID employeeId, UpdateEmployeeRequest request, UUID updatedBy) {
    Employee employee = employeeRepository.findActiveById(employeeId)
            .orElseThrow(() -> EmployeeNotFoundException.byId(employeeId.toString()));

    AuditContext.snapshot(employee);   // lưu trạng thái cũ cho audit

    // Chỉ set field khác null (partial update)
    if (request.getFirstName() != null) employee.setFirstName(request.getFirstName());
    if (request.getEmail() != null) {
        if (employeeRepository.existsActiveByEmail(request.getEmail())) {
            throw ValidationException.duplicateField("email", request.getEmail());
        }
        employee.setEmail(request.getEmail());
    }
    employee.setUpdatedBy(updatedBy);

    Employee saved = employeeRepository.save(employee);
    AuditContext.registerUpdated(saved);
    return employeeMapper.toEmployeeResponse(saved);
}
```

### e) Delete — soft delete, không xóa vật lý

```java
@Transactional
@CacheEvict(value = CacheConfig.EMPLOYEE_CACHE, allEntries = true)
@AuditAction(type = ActionType.DELETE, entity = "EMPLOYEE")
public void softDeleteEmployee(UUID employeeId, UUID deletedBy) {
    Employee employee = employeeRepository.findActiveById(employeeId)
            .orElseThrow(() -> EmployeeNotFoundException.byId(employeeId.toString()));

    AuditContext.registerDeleted(employee);

    employee.setIsDeleted(true);
    employee.setDeletedAt(Instant.now());
    employee.setDeletedBy(deletedBy);
    employee.setEmploymentStatus(EmploymentStatus.TERMINATED);
    employeeRepository.save(employee);
}
```

### Checklist Service

- [ ] `@Transactional` trên mọi method ghi dữ liệu.
- [ ] Kiểm tra trùng dữ liệu trước khi lưu.
- [ ] `orElseThrow` với exception chuẩn thay vì trả null.
- [ ] Gắn quan hệ bằng entity thật (find) không phải gán id.
- [ ] `@CacheEvict` khi dữ liệu thay đổi.
- [ ] `@AuditAction` + `AuditContext` cho thao tác ghi.
- [ ] Trả về DTO, không trả Entity.

---

## 5.7. Bước 6: Tạo Controller (`controller/EmployeeController.java`)

```java
@RestController
@RequestMapping("/api/v1/employees")
@RequiredArgsConstructor
@SecurityRequirement(name = "Bearer Authentication")
public class EmployeeController {

    private final EmployeeService employeeService;

    // Liệt kê + phân trang
    @GetMapping
    @RequirePermission(PermissionType.VIEW_EMPLOYEE)
    public ResponseEntity<PaginatedApiResponse<EmployeeResponse>> getAllEmployees(
            @PageableDefault(size = 10) Pageable pageable) {
        Page<EmployeeResponse> response = employeeService.getAllActiveEmployees(pageable);
        return ResponseEntity.ok(PaginatedApiResponse.success(response));
    }

    // Xem chi tiết
    @GetMapping("/{employeeId}")
    @RequirePermission(PermissionType.VIEW_EMPLOYEE)
    public ResponseEntity<ApiResponse<EmployeeResponse>> getEmployeeById(@PathVariable UUID employeeId) {
        return ResponseEntity.ok(ApiResponse.success(employeeService.getEmployeeById(employeeId)));
    }

    // Lấy thông tin user đang đăng nhập
    @GetMapping("/me")
    @RequirePermission(PermissionType.VIEW_PROFILE)
    public ResponseEntity<ApiResponse<EmployeeResponse>> getCurrentEmployee(
            @AuthenticationPrincipal UserPrincipal userPrincipal) {
        return ResponseEntity.ok(ApiResponse.success(employeeService.getEmployeeByUserId(userPrincipal.getId())));
    }

    // Tạo mới
    @PostMapping
    @RequirePermission(PermissionType.CREATE_EMPLOYEE)
    public ResponseEntity<ApiResponse<EmployeeResponse>> createEmployee(
            @Valid @RequestBody CreateEmployeeRequest request,
            @AuthenticationPrincipal UserPrincipal userPrincipal) {
        EmployeeResponse response = employeeService.createEmployee(request, userPrincipal.getId());
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success("Employee created successfully", response));
    }

    // Cập nhật
    @PutMapping("/{employeeId}")
    @RequirePermission(PermissionType.UPDATE_EMPLOYEE)
    public ResponseEntity<ApiResponse<EmployeeResponse>> updateEmployee(
            @PathVariable UUID employeeId,
            @Valid @RequestBody UpdateEmployeeRequest request,
            @AuthenticationPrincipal UserPrincipal userPrincipal) {
        EmployeeResponse response = employeeService.updateEmployee(employeeId, request, userPrincipal.getId());
        return ResponseEntity.ok(ApiResponse.success("Employee updated successfully", response));
    }

    // Xóa mềm
    @DeleteMapping("/{employeeId}")
    @RequirePermission(PermissionType.DELETE_EMPLOYEE)
    public ResponseEntity<ApiResponse<Void>> deleteEmployee(
            @PathVariable UUID employeeId,
            @AuthenticationPrincipal UserPrincipal userPrincipal) {
        employeeService.softDeleteEmployee(employeeId, userPrincipal.getId());
        return ResponseEntity.ok(ApiResponse.success("Employee deleted successfully", null));
    }
}
```

### Điểm cần nắm ở Controller

1. **Route**: `/api/v1/<resource>` — `@RequestMapping("/api/v1/employees")`.
2. **Validation**: `@Valid` trước `@RequestBody` → lỗi validation do `GlobalExceptionHandler` bắt, trả 400 + danh sách `errors`.
3. **Phân quyền**: `@RequirePermission(PermissionType.XXX)` trên từng method.
4. **User đang đăng nhập**: `@AuthenticationPrincipal UserPrincipal` — lấy `getId()` để ghi `createdBy/updatedBy/deletedBy`.
5. **Response chuẩn**:
   - 1 object → `ApiResponse.success(...)`.
   - danh sách/phân trang → `PaginatedApiResponse.success(page)`.
   - tạo mới → `HttpStatus.CREATED` (201).
6. **Swagger**: `@Operation(summary = ...)`, `@Tag`, `@SecurityRequirement` để hiển thị trong UI.

---

## 5.8. Bước 7: Kiểm thử API bằng Swagger / Postman

### Chuẩn bị

- Chạy project: `.\mvnw.cmd spring-boot:run`.
- Mở Swagger UI: `http://localhost:8080/swagger-ui.html`.
- Cần token: đăng nhập lấy JWT rồi bấm **Authorize**.

### Các request mẫu

**POST /api/v1/employees**

```json
{
  "employeeCode": "EMP001",
  "firstName": "Nguyen",
  "lastName": "Van A",
  "email": "vana@example.com",
  "phone": "0900000000",
  "gender": "MALE"
}
```

**Response (201):**

```json
{
  "success": true,
  "message": "Employee created successfully",
  "data": {
    "id": "550e8400-...",
    "employeeCode": "EMP001",
    "fullName": "Van A Nguyen",
    "email": "vana@example.com"
  },
  "timestamp": "2026-08-07T..."
}
```

**GET /api/v1/employees?page=0&size=10**

**GET /api/v1/employees/{id}**

**PUT /api/v1/employees/{id}** — gửi JSON chỉ chứa field muốn sửa:

```json
{
  "phone": "0911111111",
  "position": "Developer"
}
```

**DELETE /api/v1/employees/{id}** → soft delete, record vẫn còn trong DB nhưng mọi query `findActive*` sẽ không trả về.

---

## 5.9. Tóm tắt quy trình dựng 1 module CRUD

| Bước | Việc cần làm | File |
| ---- | ------------ | ---- |
| 1 | Khai báo bảng + audit + soft delete + mã hóa | `domain/X.java` |
| 2 | DTO input (validation) + output | `dto/request`, `dto/response` |
| 3 | MapStruct convert | `mapper/XMapper.java` |
| 4 | Query (luôn lọc isDeleted = false) | `repository/XRepository.java` |
| 5 | Logic + `@Transactional` + audit/cache + exception | `service/XService.java` |
| 6 | Endpoint + `@RequirePermission` + bọc response | `controller/XController.java` |
| 7 | Test bằng Swagger/Postman | — |

Sau khi module chạy được, đọc tiếp các phần sau để thêm: security/JWT (`06`), JPA nâng cao (`07`), response/exception chuẩn (`08`), cache/audit chi tiết (`09`), và viết test (`10`).
