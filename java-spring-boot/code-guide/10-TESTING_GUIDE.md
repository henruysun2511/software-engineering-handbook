# Phần 10: Testing (Unit Test)

Hướng dẫn viết unit test cho dự án bằng **JUnit 5 + Mockito + AssertJ**, đúng chuẩn của codebase hiện tại.

## 10.1. Cấu trúc thư mục test

```
src/test/java/com/example/<project_name>/
├── BaseUnitTest.java                    ← class cha cho mọi unit test
├── TestFixtures.java                    ← dữ liệu mẫu dùng chung
├── auth/util/JwtUtilTest.java
├── auth/...                             ← test cho phần auth
├── employee/service/EmployeeServiceTest.java
├── department/service/DepartmentServiceTest.java
├── contract/service/ContractServiceTest.java
├── attendance/service/AttendanceServiceTest.java
├── payroll/service/PayrollServiceTest.java
├── user/service/UserServiceTest.java
└── ...
```

## 10.2. Base class — `BaseUnitTest`

Mọi test **kế thừa** `BaseUnitTest` để nhận sẵn:

```java
@ExtendWith(MockitoExtension.class)   // Tự động khởi tạo @Mock, @InjectMocks
@ActiveProfiles("test")               // Dùng profile test khi cần Spring context
public abstract class BaseUnitTest {
    protected static final String TEST_EMAIL = "test@example.com";
    protected static final String TEST_PASSWORD = "testPassword123";
    protected static final String TEST_IP_ADDRESS = "192.168.1.1";

    @BeforeEach
    void baseSetUp() {
        SecurityContextHolder.clearContext();   // Reset SecurityContext trước mỗi test
    }
}
```

> Vì hầu hết test là **unit test thuần** (không cần Spring context), `MockitoExtension` đủ dùng. `@ActiveProfiles("test")` chỉ có tác dụng khi test có Spring context (VD `@SpringBootTest`).

## 10.3. Quy ước viết test

- Tên class: `{ClassName}Test` (VD `EmployeeServiceTest`).
- Mỗi test dùng mẫu **Given → When → Then** (viết bằng comment `// Given`, `// When`, `// Then`).
- Dùng `@DisplayName("Mô tả tiếng Anh")` cho từng test.
- Mock dependency bằng `@Mock`, đưa vào service bằng `@InjectMocks`:

```java
class EmployeeServiceTest extends BaseUnitTest {

    @Mock
    private EmployeeRepository employeeRepository;

    @Mock
    private EmployeeMapper employeeMapper;

    @InjectMocks
    private EmployeeService employeeService;
}
```

- Không cần `@BeforeEach` khởi tạo service — `@InjectMocks` tự inject mock.

## 10.4. Test Happy path

```java
@Test
@DisplayName("Should return employee when found")
void shouldReturnEmployeeWhenFound() {
    // Given
    UUID employeeId = UUID.randomUUID();
    Employee employee = TestFixtures.sampleEmployee();
    when(employeeRepository.findActiveById(employeeId)).thenReturn(Optional.of(employee));
    when(employeeMapper.toEmployeeResponse(employee)).thenReturn(new EmployeeResponse());

    // When
    EmployeeResponse result = employeeService.getEmployeeById(employeeId);

    // Then
    assertThat(result).isNotNull();
    verify(employeeRepository).findActiveById(employeeId);
}
```

## 10.5. Test lỗi — `assertThatThrownBy`

```java
@Test
@DisplayName("Should throw EmployeeNotFoundException when employee not found")
void shouldThrowExceptionWhenEmployeeNotFound() {
    // Given
    UUID employeeId = UUID.randomUUID();
    when(employeeRepository.findActiveById(employeeId)).thenReturn(Optional.empty());

    // When & Then
    assertThatThrownBy(() -> employeeService.getEmployeeById(employeeId))
            .isInstanceOf(EmployeeNotFoundException.class);
}
```

## 10.6. Test có xác minh tương tác — `verify` / `verifyNoInteractions`

```java
// Kiểm tra repository được gọi đúng số lần
verify(employeeRepository).save(any(Employee.class));

// Kiểm tra KHÔNG gọi sai dependency khi dữ liệu không hợp lệ
verifyNoInteractions(employeeRepository);
```

## 10.7. Test `JwtUtil` — ví dụ đầy đủ (đã có sẵn trong dự án)

Tham khảo `auth/util/JwtUtilTest.java`:

- Dùng `@MockitoSettings(strictness = Strictness.LENIENT)` khi có mock config dùng chung để tránh `UnnecessaryStubbingException`.
- `lenient().when(...)` cho các stub dùng chung.
- Tạo token "quá hạn" bằng `Jwts.builder()` với `setExpiration(new Date(System.currentTimeMillis() - 1000))`.
- Assert nhiều điều kiện cùng lúc bằng `assertAll(...)`:

```java
assertAll(
    () -> assertThat(token).isNotNull(),
    () -> assertThat(token).isNotEmpty(),
    () -> assertThat(token.split("\\.")).hasSize(3)
);
```

## 10.8. Test khi service gọi `AuditContext` / `@Cacheable`

- `AuditContext` dùng ThreadLocal — trong unit test chỉ cần gọi trực tiếp `AuditContext.registerCreated(entity)` như service thật; các service cấp thấp (AuditService) nên được mock.
- `@Cacheable`/`@CacheEvict` **không chạy** trong unit test thuần (không có Spring proxy) → không cần xử lý cache khi test service. Cache được kiểm tra ở tầng integration test riêng.

## 10.9. Lệnh chạy test

```bash
# Chạy tất cả test
.\mvnw.cmd test

# Chạy 1 class test cụ thể
.\mvnw.cmd test -Dtest=EmployeeServiceTest

# Chạy test theo pattern
.\mvnw.cmd test -Dtest="*ServiceTest"

# Chạy test kèm báo cáo coverage (nếu cấu hình Jacoco)
.\mvnw.cmd test jacoco:report
```

## 10.10. Checklist test mới

- [ ] Kế thừa `BaseUnitTest`.
- [ ] Mock tất cả dependency với `@Mock` + inject bằng `@InjectMocks`.
- [ ] Dữ liệu mẫu dùng từ `TestFixtures` (hoặc tạo riêng nếu cần).
- [ ] Viết đủ: happy path + trường hợp ngoại lệ.
- [ ] `@DisplayName` mô tả rõ hành vi.
- [ ] Chạy `.\mvnw.cmd test` trước khi commit.
