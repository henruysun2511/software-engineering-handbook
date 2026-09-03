# Phần 9: Cache Redis & Audit Log

Hướng dẫn về hai cơ chế cắt ngang (cross-cutting) của dự án: **cache** (tăng tốc đọc dữ liệu) và **audit log** (ghi lại lịch sử thay đổi dữ liệu).

## 9.1. Cache Redis — tổng quan

- Cache được dùng cho các thao tác **đọc** để tránh query DB lặp lại.
- Cấu hình trung tâm: `shared/config/CacheConfig.java` (`@EnableCaching`).
- Bật cache chung: `spring.cache.type=redis` trong `application.properties`.
- Tên các cache được khai báo là **hằng số** trong `CacheConfig` (VD: `EMPLOYEE_CACHE = "employees"`).

### Kiến trúc cache (Fallback)

- **`RedisCacheManager`** — cache chính.
- **`FallbackCacheManager`** bọc quanh `RedisCacheManager` → nếu **Redis lỗi/mất kết nối**, tự động tắt cache và quay về query DB (app **vẫn chạy**, không crash).
- `FallbackCache` (`shared/cache/FallbackCache.java`) bắt `RedisConnectionFailureException` và trả về `null` → Spring gọi lại method để query DB.
- Bật/tắt fallback: `app.cache.fallback.enabled`.

## 9.2. Dùng cache trong Service

### Đọc dữ liệu — `@Cacheable`

```java
@Cacheable(value = CacheConfig.EMPLOYEE_CACHE, key = "#employeeId.toString()")
public EmployeeResponse getEmployeeById(UUID employeeId) {
    Employee employee = employeeRepository.findActiveById(employeeId)
            .orElseThrow(() -> EmployeeNotFoundException.byId(employeeId.toString()));
    return employeeMapper.toEmployeeResponse(employee);
}
```

- `value` = tên cache; `key` = SpEL chỉ ra khóa (VD `#employeeId`).
- **Chỉ method public gọi từ bean khác mới được cache** (Spring AOP proxy) — gọi nội bộ trong cùng class sẽ không cache.

### Ghi dữ liệu — `@CacheEvict`

Khi dữ liệu thay đổi, phải xóa cache cũ:

```java
@Transactional
@CacheEvict(value = CacheConfig.EMPLOYEE_CACHE, allEntries = true)   // xóa toàn bộ cache employees
@AuditAction(type = ActionType.CREATE, entity = "EMPLOYEE")
public EmployeeResponse createEmployee(CreateEmployeeRequest request, UUID createdBy) { ... }
```

- `allEntries = true` — xóa toàn bộ entry của cache đó (an toàn khi dữ liệu liên quan nhau).
- Mọi method **ghi** (create/update/delete) đều cần `@CacheEvict` tương ứng.

### TTL của từng cache

| Cache | TTL |
| ----- | --- |
| `users`, `usersByEmail`, `employees`, `employeeSalaries`, `contracts`, `payrolls`, `insurances`, `degrees`, `relatives` | 30 phút |
| `roles`, `rolesByCode`, `permissions`, `permissionByCode`, `activeRoles`, `activePermissions`, `departments`, `workShifts`, `stores` | 1 giờ |
| `systemRoles`, `systemPermissions` | 2 giờ |
| `userRoles`, `userPermissions` | 15 phút |
| `rolePermissions` | 30 phút |
| `leaveRequests` | 15 phút |
| `attendances` | 10 phút |

### Kiểm tra cache hoạt động

- `RedisHealthCheckService` (`shared/cache`) kiểm tra kết nối Redis.
- `CacheController` (`shared/cache`) — API kiểm tra/thao tác cache (nếu được expose).

## 9.3. Audit Log — tổng quan

- Ghi lại **ai**, **khi nào**, **thay đổi field nào** trên dữ liệu (CREATE/UPDATE/DELETE).
- Cơ chế: **AOP aspect** (`AuditActionAspect`) + **annotation** `@AuditAction` + `AuditContext` (ThreadLocal).
- Cấu hình: `app.audit.*` (bật/tắt, log params/response, async...).

### Kiến trúc

```
Service method có @AuditAction
   │  gọi AuditContext.registerCreated/snapshot/registerUpdated/registerDeleted
   ▼
AuditActionAspect (@Around, @Order(10))  — chạy sau khi method thực hiện xong
   │  đọc dữ liệu từ AuditContext
   │  tạo danh sách field thay đổi (AuditLogDetailDto)
   ▼
AuditService.createAuditLogWithDetails(...)  → lưu bảng audit_logs + audit_log_details
```

- `AuditContext` dùng **ThreadLocal** → đúng ngữ cảnh từng request; `clear()` ở cuối aspect.
- `AuditProperties` cấu hình: `enabled`, `asyncLogging`, `logParameters`, `logResponse`, `retentionDays`, `compressOldLogs`.
- `AuditCleanupService` + `AuditConfiguration` — dọn log cũ theo `retentionDays` (kèm `@Scheduled`).

## 9.4. Cách dùng `@AuditAction` trong Service

### Annotation

```java
@Target(ElementType.METHOD)
public @interface AuditAction {
    ActionType type();     // CREATE, UPDATE, DELETE
    String entity();       // "EMPLOYEE", "DEPARTMENT"...
    String actionCode() default "";   // mặc định tự sinh "{TYPE}_{ENTITY}"
}
```

### CREATE — gọi `AuditContext.registerCreated(saved)`

```java
@Transactional
@AuditAction(type = ActionType.CREATE, entity = "EMPLOYEE")
public EmployeeResponse createEmployee(CreateEmployeeRequest request, UUID createdBy) {
    ...
    Employee saved = employeeRepository.save(employee);
    AuditContext.registerCreated(saved);        // <-- khai báo entity vừa tạo
    return employeeMapper.toEmployeeResponse(saved);
}
```

### UPDATE — `snapshot()` trước khi sửa + `registerUpdated()` sau khi lưu

```java
@Transactional
@AuditAction(type = ActionType.UPDATE, entity = "EMPLOYEE")
public EmployeeResponse updateEmployee(UUID employeeId, UpdateEmployeeRequest request, UUID updatedBy) {
    Employee employee = employeeRepository.findActiveById(employeeId).orElseThrow(...);

    AuditContext.snapshot(employee);            // <-- 1. chụp trạng thái CŨ

    if (request.getFirstName() != null) employee.setFirstName(request.getFirstName());
    ...

    Employee saved = employeeRepository.save(employee);
    AuditContext.registerUpdated(saved);        // <-- 2. khai báo entity MỚI

    return employeeMapper.toEmployeeResponse(saved);
}
```

> Aspect sẽ **diff** snapshot cũ vs entity mới → chỉ ghi các field **thay đổi** (`oldValue`/`newValue`).

### DELETE — `registerDeleted()` trước khi xóa mềm

```java
@Transactional
@AuditAction(type = ActionType.DELETE, entity = "EMPLOYEE")
public void softDeleteEmployee(UUID employeeId, UUID deletedBy) {
    Employee employee = employeeRepository.findActiveById(employeeId).orElseThrow(...);

    AuditContext.registerDeleted(employee);     // <-- trước khi đánh dấu xóa

    employee.setIsDeleted(true);
    employee.setDeletedAt(Instant.now());
    employeeRepository.save(employee);
}
```

## 9.5. Những field được audit

- Dựa trên `@AuditableEntity(ignoreFields = {...})` trên Entity:

```java
@AuditableEntity(ignoreFields = {
    "id", "password", "updatedAt", "updatedBy",
    "createdAt", "createdBy", "isDeleted", "deletedAt", "deletedBy"
})
public class User { ... }
```

- Các field trong `ignoreFields` **không** được ghi audit (id, timestamp, soft-delete, password...).
- Field enum lưu dưới dạng `name()`, field quan hệ lưu dạng `EntityClass:id`.

## 9.6. Checklist khi thêm cache & audit cho method mới

**Cache:**
- [ ] Đọc dữ liệu tần suất cao → thêm `@Cacheable(value = CacheConfig.XXX_CACHE, key = "...")`.
- [ ] Method ghi dữ liệu → thêm `@CacheEvict(value = CacheConfig.XXX_CACHE, allEntries = true)`.
- [ ] Dùng hằng số cache từ `CacheConfig` (không viết chuỗi trực tiếp).
- [ ] TTL phù hợp cho cache mới trong `CacheConfig.getCacheConfigurations()`.

**Audit:**
- [ ] Method ghi dữ liệu → thêm `@AuditAction(type = ..., entity = "XXX")`.
- [ ] CREATE: `AuditContext.registerCreated(saved)`.
- [ ] UPDATE: `AuditContext.snapshot(entity)` trước sửa + `AuditContext.registerUpdated(saved)` sau lưu.
- [ ] DELETE: `AuditContext.registerDeleted(entity)` trước khi xóa.
- [ ] Entity có `@AuditableEntity(ignoreFields = {...})`.
