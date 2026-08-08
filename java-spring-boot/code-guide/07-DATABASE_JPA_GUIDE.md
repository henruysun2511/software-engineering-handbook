# Phần 7: Database & JPA

Hướng dẫn chi tiết về tầng dữ liệu của dự án: cấu hình kết nối, Entity, auditing, soft delete, mã hóa field, quan hệ và query.

> Nên đọc `03-CONFIGURATION_GUIDE.md` (cấu hình DB) và `05-CRUD_TUTORIAL.md` (cách dựng Entity trong module CRUD).

## 7.1. Cấu hình kết nối DB & JPA

### Dev (`application-development.properties`)

```properties
spring.datasource.url=jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:worksphere}
spring.datasource.username=${DB_USERNAME:postgres}
spring.datasource.password=${DB_PASSWORD:postgres}
spring.datasource.driver-class-name=org.postgresql.Driver

spring.jpa.hibernate.ddl-auto=update
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.show-sql=true
```

### Chạy DB bằng Docker (có trong `docker-compose.yml`)

```bash
docker-compose up -d     # khởi động PostgreSQL + Redis
```

### Chế độ test

Dự án có `runtimeOnly 'com.h2database:h2'` — H2 dùng cho test khi không có PostgreSQL.

## 7.2. Entity cơ bản — `domain/X.java`

### Khai báo chuẩn

```java
@Entity
@Table(name = "users")                // tên bảng snake_case, số nhiều
@AuditableEntity(ignoreFields = {     // các field KHÔNG đưa vào audit diff
    "id", "password", "updatedAt", "updatedBy",
    "createdAt", "createdBy", "isDeleted", "deletedAt", "deletedBy"
})
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)   // UUID tự sinh
    private UUID id;
}
```

### Quy tắc

| Quy tắc | Ví dụ |
| ------- | ----- |
| Khóa chính là `UUID` sinh tự động | `@GeneratedValue(strategy = GenerationType.UUID)` |
| Đặt tên cột rõ ràng bằng `@Column(name = "...")` | `@Column(name = "employee_code")` |
| Ràng buộc DB bằng thuộc tính column | `nullable = false`, `unique = true`, `length = 50` |
| Enum lưu dạng chuỗi | `@Enumerated(EnumType.STRING)` |
| Giá trị mặc định khi dùng Builder | `@Builder.Default private Boolean isDeleted = false;` |
| Không trả Entity trực tiếp ra API | luôn qua Mapper + DTO |

## 7.3. Audit fields (tự động ghi thời gian)

Entity nào cũng có bộ field này:

```java
@CreationTimestamp
@Column(name = "created_at")
private Instant createdAt;

@UpdateTimestamp
@Column(name = "updated_at")
private Instant updatedAt;

@Column(name = "created_by")   // ID người tạo
private UUID createdBy;

@Column(name = "updated_by")   // ID người sửa cuối
private UUID updatedBy;
```

- `@CreationTimestamp` / `@UpdateTimestamp` (Hibernate) — **tự động** điền `createdAt` khi insert, cập nhật `updatedAt` khi update. Không cần code.
- `createdBy`/`updatedBy` — do **Service** set thủ công từ `@AuthenticationPrincipal`:

```java
employee.setCreatedBy(userPrincipal.getId());   // khi tạo
employee.setUpdatedBy(userPrincipal.getId());   // khi sửa
```

## 7.4. Soft delete (xóa mềm)

**Dự án không dùng `DELETE` vật lý.** Mọi entity đều có:

```java
@Column(name = "is_deleted")
@Builder.Default
private Boolean isDeleted = false;

@Column(name = "deleted_at")
private Instant deletedAt;

@Column(name = "deleted_by")
private UUID deletedBy;
```

### Xóa mềm trong Service

```java
public void softDeleteEmployee(UUID employeeId, UUID deletedBy) {
    Employee employee = employeeRepository.findActiveById(employeeId)
            .orElseThrow(() -> EmployeeNotFoundException.byId(employeeId.toString()));

    employee.setIsDeleted(true);
    employee.setDeletedAt(Instant.now());
    employee.setDeletedBy(deletedBy);
    employeeRepository.save(employee);
}
```

### Nguyên tắc khi viết query

**MỌI query phải lọc `isDeleted = false`** — thường đặt tên có tiền tố `Active`/`findActive`:

```java
@Query("SELECT e FROM Employee e WHERE e.id = :id AND e.isDeleted = false")
Optional<Employee> findActiveById(@Param("id") UUID id);

@Query("SELECT e FROM Employee e WHERE e.isDeleted = false")
Page<Employee> findAllActive(Pageable pageable);
```

## 7.5. Mã hóa field nhạy cảm

Các field như CCCD, số tài khoản, địa chỉ được **mã hóa trước khi lưu DB** bằng **AES-GCM**.

### Converter — `shared/persistence/encryption/EncryptedStringConverter.java`

```java
@Converter
public class EncryptedStringConverter implements AttributeConverter<String, String> {
    @Override
    public String convertToDatabaseColumn(String attribute) {  // khi save → mã hóa
        return encryptor.encrypt(attribute);
    }
    @Override
    public String convertToEntityAttribute(String dbData) {    // khi load → giải mã
        return encryptor.decrypt(dbData);
    }
}
```

### Encryptor — `AesGcmStringEncryptor.java`

- Dùng **AES/GCM/NoPadding**, key 256-bit, IV 12 byte, tag 128 bit.
- Key lấy từ `app.security.encryption.key-base64` (biến môi trường).
- Nếu không cấu hình key → tự sinh key ngẫu nhiên lúc khởi động (**cảnh báo**: dữ liệu mã hóa bằng key cũ sẽ không đọc lại được nếu restart đổi key — nên luôn cấu hình key cố định trong production).
- Khi giải mã dữ liệu cũ chưa mã hóa (plaintext) → trả nguyên văn (không crash).

### Dùng trong Entity

```java
@Convert(converter = EncryptedStringConverter.class)
@Column(name = "id_card_number", length = 512)   // để dư độ dài cho bản mã hóa
private String idCardNumber;
```

> Field bị mã hóa nên **không thể tìm kiếm/chỉ mục** bằng query tìm kiếm LIKE trên giá trị gốc.

## 7.6. Quan hệ JPA

### ManyToOne (nhiều-nhất) — ví dụ Employee → Department

```java
@ManyToOne(fetch = FetchType.LAZY)     // LUÔN dùng LAZY
@JoinColumn(name = "department_id")
private Department department;
```

- `LAZY` tránh tải toàn bộ đối tượng liên quan khi không cần.
- Khi gán quan hệ trong Service, **lấy entity thật** chứ không gán id:

```java
Department department = departmentRepository.findActiveById(id).orElseThrow(...);
employee.setDepartment(department);
```

### OneToOne — ví dụ Employee → User

```java
@OneToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "user_id", unique = true)
private User user;
```

### OneToMany (chiều ngược, dùng qua query)

Không khai báo danh sách trong Entity — dự án **truy vấn theo khóa ngoại** từ phía "nhiều" thay vì load danh sách:

```java
// Thay vì employee.getRefreshTokens(), dùng repository:
@Query("SELECT r FROM RefreshToken r WHERE r.user.id = :userId AND r.isRevoked = false")
```

### Lấy id của quan hệ mà không load — dùng trong Mapper/Response

```java
@Mapping(target = "departmentId", expression = "java(employee.getDepartment() != null ? employee.getDepartment().getId() : null)")
```

## 7.7. Repository & query — `repository/XRepository.java`

### Các kiểu method

| Cách viết | Ví dụ | Trả về |
| --------- | ----- | ------ |
| Method mặc định của JpaRepository | `findById`, `save`, `deleteById`, `count` | có sẵn |
| Tên method tự sinh (Spring Data) | `countByEmploymentStatusAndIsDeletedFalse(...)` | `long` |
| JPQL `@Query` | `findActiveById`, `searchActive` | `Optional`/`Page` |

### JPQL với tham số

```java
@Query("SELECT e FROM Employee e WHERE e.id = :id AND e.isDeleted = false")
Optional<Employee> findActiveById(@Param("id") UUID id);
```

- Dùng `@Param("...")` khớp với `:...` trong câu JPQL.
- Truy cập field của quan hệ qua dấu chấm: `e.user.id`, `e.department.id`.

### Phân trang

```java
@Query("SELECT e FROM Employee e WHERE e.isDeleted = false")
Page<Employee> findAllActive(Pageable pageable);
```

- `Pageable` do Controller cung cấp (`@PageableDefault(size = 10)`).
- Service map từng phần tử qua `page.map(mapper::toResponse)`.

### Tìm kiếm LIKE nhiều cột

```java
@Query("SELECT e FROM Employee e WHERE e.isDeleted = false AND (" +
        "LOWER(e.fullName) LIKE LOWER(CONCAT('%', :keyword, '%')) OR " +
        "LOWER(e.email) LIKE LOWER(CONCAT('%', :keyword, '%')))")
Page<Employee> searchActive(@Param("keyword") String keyword, Pageable pageable);
```

> Lưu ý `LOWER` + `CONCAT` để tìm không phân biệt hoa thường, tránh SQL injection nhờ tham số hóa.

## 7.8. Quản lý schema

- `spring.jpa.hibernate.ddl-auto=update` — Hibernate tự tạo/cập nhật bảng theo Entity (thuận tiện dev, **không nên** dùng production với dữ liệu quan trọng).
- Production dùng biến `DDL_AUTO` (`application-production.properties`).
- Khi thay đổi Entity → dựa vào `ddl-auto` cập nhật schema; nếu cần thay đổi cấu trúc lớn nên dùng migration (Flyway/Liquibase) — dự án hiện chưa có.

## 7.9. Checklist khi tạo Entity mới

- [ ] `@Entity` + `@Table(name = "ten_bang")`.
- [ ] Khóa chính `UUID` + `@GeneratedValue(strategy = GenerationType.UUID)`.
- [ ] Đủ bộ audit fields: `createdAt`, `updatedAt`, `createdBy`, `updatedBy`.
- [ ] Đủ bộ soft delete: `isDeleted`, `deletedAt`, `deletedBy` (`@Builder.Default`).
- [ ] `@AuditableEntity(ignoreFields = {...})` với các field không cần audit.
- [ ] Enum dùng `@Enumerated(EnumType.STRING)`.
- [ ] Field nhạy cảm dùng `@Convert(converter = EncryptedStringConverter.class)`.
- [ ] Quan hệ `LAZY`, tên cột rõ ràng, đặt `unique`/`nullable` đúng.
- [ ] Mọi query trong Repository lọc `isDeleted = false`.
- [ ] Trả ra API qua Mapper → DTO, không trả Entity.

## 7.10. JOIN bảng trong JPQL

Khi cần dữ liệu từ **nhiều bảng liên quan** trong 1 query, dùng JPQL JOIN. Khác SQL thuần: JPQL join qua **tên field quan hệ** của Entity, không viết tên bảng.

### Các kiểu JOIN

| Kiểu | Cú pháp | Ý nghĩa |
| ---- | ------- | ------- |
| `JOIN` (inner) | `JOIN e.department d` | Chỉ lấy bản ghi **có** bản ghi liên quan |
| `LEFT JOIN` | `LEFT JOIN e.department d` | Lấy hết bên trái, bên phải không có thì `null` |
| `JOIN FETCH` | `JOIN FETCH e.department` | JOIN + **tải luôn** quan hệ (tránh N+1) |

### JOIN theo quan hệ (không fetch)

```java
// Lấy employee theo tên phòng ban — join qua field department
@Query("SELECT e FROM Employee e JOIN e.department d " +
       "WHERE d.name = :name AND e.isDeleted = false")
List<Employee> findActiveByDepartmentName(@Param("name") String name);
```

- Bí danh `d` ở đây là **Department** (kiểu của field `e.department`), viết theo tên field của Entity.
- `WHERE` truy cập field của bảng join qua bí danh: `d.name`.

### `JOIN FETCH` — tải sớm quan hệ LAZY, chống N+1

Entity dùng `fetch = FetchType.LAZY`. Nếu trả về danh sách rồi mới truy cập `entity.getWorkShift()` từng phần tử → mỗi lần 1 query phụ (N+1). Dùng `JOIN FETCH` tải trước trong cùng query:

```java
// Ví dụ thật trong EmployeeWorkShiftRepository
@Query("SELECT ews FROM EmployeeWorkShift ews " +
       "JOIN FETCH ews.workShift ws JOIN FETCH ews.employee emp " +
       "WHERE ews.date = :date AND ews.isDeleted = false")
List<EmployeeWorkShift> findActiveByDate(@Param("date") LocalDate date);
```

- `JOIN FETCH` **bắt buộc** có bản ghi liên quan (khác `LEFT JOIN FETCH`).
- Không cần bí danh nếu chỉ fetch, không filter; nếu cần filter/dùng field quan hệ thì đặt bí danh (VD `ws`, `emp`).
- **Không fetch quá 1 collection** trong cùng query (OneToMany) — dễ tạo Cartesian product. Khi cần, tách query hoặc dùng `@EntityGraph`.

### `LEFT JOIN` — lấy cả bản ghi không có quan hệ

```java
// Lấy mọi employee kèm tên department; ai chưa có department thì departmentName = null
@Query("SELECT e.fullName, d.name FROM Employee e LEFT JOIN e.department d " +
       "WHERE e.isDeleted = false")
List<Object[]> findNamesWithDepartment();
```

### LEFT JOIN theo 1-nhiều với điều kiện — ví dụ thật trong `AuditLogRepository`

```java
// kết hợp log cha với chi tiết field-level bằng LEFT JOIN a.details d
@Query("SELECT a, d FROM AuditLog a LEFT JOIN a.details d WHERE ...")
```

### Chống N+1 — checklist

- [ ] Query trả về **danh sách** và Service sẽ đọc field quan hệ → dùng `JOIN FETCH` (hoặc `@EntityGraph(attributePaths = "workShift")`).
- [ ] Chỉ fetch quan hệ **thật sự cần**; fetch thừa = query nặng.
- [ ] Không fetch 2+ collection cùng lúc.
- [ ] `findById`/`Optional` đơn lẻ không cần fetch — LAZY sẽ load khi cần (1 query thêm không đáng kể).
