# TÀI LIỆU THAM KHẢO: CÁC ANNOTATION QUAN TRỌNG TRONG SPRING BOOT

> Tài liệu tra cứu nhanh — mỗi annotation gồm mô tả ngắn gọn và ví dụ tối thiểu để hiểu cách dùng. Dùng kèm với bộ tài liệu Chương 1-8 (Java Backend Developer) để tra cứu khi code, không thay thế phần giải thích chi tiết cơ chế hoạt động đã có trong các chương tương ứng.

---

## Mục lục nhóm

1. [Core Spring — IoC & Dependency Injection](#1-core-spring--ioc--dependency-injection)
2. [Spring Boot — Khởi động & Cấu hình](#2-spring-boot--khởi-động--cấu-hình)
3. [Spring Web MVC — REST API](#3-spring-web-mvc--rest-api)
4. [Bean Validation](#4-bean-validation)
5. [Spring Data JPA](#5-spring-data-jpa)
6. [Transaction](#6-transaction)
7. [Spring Security](#7-spring-security)
8. [Async, Scheduling & Retry](#8-async-scheduling--retry)
9. [Spring AOP](#9-spring-aop)
10. [Caching](#10-caching)
11. [Messaging (RabbitMQ / Kafka)](#11-messaging-rabbitmq--kafka)
12. [Testing](#12-testing)
13. [Lombok](#13-lombok)
14. [Jackson (JSON Serialization/Deserialization)](#14-jackson-json-serializationdeserialization)
15. [Spring Cloud & Microservices](#15-spring-cloud--microservices)
16. [WebSocket, Spring Batch & Spring AI](#16-websocket-spring-batch--spring-ai)

---

## 1. Core Spring — IoC & Dependency Injection

### `@Component`
Đánh dấu 1 class là Bean chung chung, được `@ComponentScan` tự động phát hiện và đăng ký vào IoC Container.

```java
@Component
public class SlugGenerator {
    public String generate(String title) {
        return title.toLowerCase().replace(" ", "-");
    }
}
```

### `@Service`
Chuyên biệt hóa của `@Component`, đánh dấu Bean thuộc tầng business logic. Về kỹ thuật hoạt động giống hệt `@Component`, khác biệt chỉ ở ý nghĩa ngữ nghĩa giúp code dễ đọc.

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    public OrderService(OrderRepository orderRepository) { this.orderRepository = orderRepository; }
}
```

### `@Repository`
Chuyên biệt hóa của `@Component` cho tầng truy cập dữ liệu. Spring tự động "dịch" exception gốc của tầng persistence (VD: `SQLException`) sang `DataAccessException` thống nhất.

```java
@Repository
public class LegacyOrderDao {
    // Thao tác JDBC thuần, không dùng Spring Data JPA
}
```

### `@Controller` / `@RestController`
`@Controller` đánh dấu Bean xử lý HTTP request, trả về tên view (MVC truyền thống). `@RestController` = `@Controller` + `@ResponseBody`, trả JSON/XML trực tiếp — dùng cho REST API.

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    @GetMapping("/{id}")
    public OrderDTO getOrder(@PathVariable Long id) { return orderService.findById(id); }
}
```

### `@Autowired`
Yêu cầu Spring tiêm (inject) dependency. Từ Spring 4.3+, nếu class chỉ có **duy nhất 1 constructor**, không cần annotation này nữa — Spring tự động dùng constructor đó. Chỉ thực sự cần khi có **nhiều constructor** hoặc dùng Setter/Field Injection.

```java
@Service
public class NotificationService {
    private SmsSender smsSender;

    @Autowired(required = false) // optional dependency
    public void setSmsSender(SmsSender smsSender) { this.smsSender = smsSender; }
}
```

### `@Qualifier`
Chỉ định rõ Bean nào cần inject khi có **nhiều Bean cùng implement 1 interface**, tránh lỗi `NoUniqueBeanDefinitionException`.

```java
@Service
public class CheckoutService {
    private final ShippingFeeStrategy strategy;

    public CheckoutService(@Qualifier("expressShipping") ShippingFeeStrategy strategy) {
        this.strategy = strategy;
    }
}
```

### `@Primary`
Đánh dấu 1 Bean là lựa chọn **mặc định** khi có nhiều Bean cùng loại và không có `@Qualifier` chỉ định cụ thể.

```java
@Component
@Primary
public class StandardShippingStrategy implements ShippingFeeStrategy { /* ... */ }
```

### `@Configuration`
Đánh dấu class là nguồn khai báo Bean tường minh (Java Config), thường đi kèm các method `@Bean` bên trong.

```java
@Configuration
public class AppConfig {
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper().registerModule(new JavaTimeModule());
    }
}
```

### `@Bean`
Khai báo 1 method trả về object sẽ được đăng ký làm Bean trong Container — dùng cho class bên thứ 3 (không có source code để gắn `@Component`) hoặc logic khởi tạo phức tạp.

```java
@Bean
public DataSource dataSource(@Value("${spring.datasource.url}") String url) {
    HikariConfig config = new HikariConfig();
    config.setJdbcUrl(url);
    return new HikariDataSource(config);
}
```

### `@Scope`
Xác định vòng đời/số lượng instance của Bean. Mặc định là `singleton` (1 instance duy nhất). `prototype` tạo instance mới mỗi lần inject.

```java
@Component
@Scope("prototype")
public class ReportBuilder {
    private final List<String> sections = new ArrayList<>(); // state riêng biệt mỗi lần dùng
}
```

### `@PostConstruct` / `@PreDestroy`
`@PostConstruct` chạy ngay sau khi mọi dependency đã inject xong — dùng để khởi tạo (warm-up cache, validate config). `@PreDestroy` chạy trước khi Container hủy Bean — dùng để giải phóng tài nguyên.

```java
@Component
public class CacheWarmupService {
    @PostConstruct
    public void warmUp() { /* load dữ liệu vào cache */ }

    @PreDestroy
    public void cleanup() { /* đóng connection, thread pool */ }
}
```

### `@Value`
Inject 1 giá trị đơn lẻ từ file cấu hình (`application.yml`) hoặc biểu thức SpEL vào field/tham số.

```java
@Component
public class MailSender {
    @Value("${mail.sender.address}")
    private String senderAddress;
}
```

### `@Lazy`
Trì hoãn khởi tạo Bean tới khi thực sự được dùng lần đầu, thay vì eager khởi tạo ngay lúc start ứng dụng (mặc định của `ApplicationContext`). Cẩn trọng: không nên dùng để "né" lỗi Circular Dependency — đó là dấu hiệu cần refactor thiết kế.

```java
@Service
@Lazy
public class HeavyReportGenerator { /* khởi tạo tốn nhiều tài nguyên, chỉ tạo khi cần */ }
```

---

## 2. Spring Boot — Khởi động & Cấu hình

### `@SpringBootApplication`
Annotation tổng hợp, gộp 3 annotation: `@SpringBootConfiguration` (= `@Configuration`), `@EnableAutoConfiguration` (bật cơ chế tự động cấu hình Bean dựa trên classpath), `@ComponentScan` (quét package hiện tại + package con).

```java
@SpringBootApplication
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

### `@EnableAutoConfiguration`
Bật cơ chế Auto-configuration — Spring Boot tự động đăng ký Bean dựa trên thư viện có trong classpath (thường không cần dùng riêng lẻ vì đã gộp trong `@SpringBootApplication`).

### `@ComponentScan`
Chỉ định (hoặc mở rộng) phạm vi package Spring sẽ quét để tìm `@Component` và các stereotype liên quan. Cần dùng riêng khi Bean nằm ngoài package con của class `@SpringBootApplication`.

```java
@SpringBootApplication
@ComponentScan(basePackages = {"com.company.orderservice", "com.company.shared"})
public class OrderServiceApplication { }
```

### `@ConfigurationProperties`
Bind cả 1 **nhóm** cấu hình liên quan từ `application.yml` vào 1 object type-safe (thường là Record), thay vì dùng nhiều `@Value` rời rạc.

```java
@ConfigurationProperties(prefix = "mail.sender")
public record MailSenderProperties(String address, String displayName, Duration timeout) {}
```

```yaml
mail:
  sender:
    address: no-reply@company.com
    display-name: "Company System"
    timeout: 5s
```

### `@EnableConfigurationProperties`
Kích hoạt 1 class `@ConfigurationProperties` để Spring quản lý như Bean (cần thiết nếu class đó không tự có `@Component`).

```java
@SpringBootApplication
@EnableConfigurationProperties(MailSenderProperties.class)
public class OrderServiceApplication { }
```

### `@Profile`
Chỉ đăng ký Bean khi ứng dụng chạy với profile tương ứng (`dev`, `staging`, `prod`) — dùng để có hành vi/Bean khác nhau giữa các môi trường.

```java
@Configuration
public class MailConfig {
    @Bean
    @Profile("prod")
    public MailSender realMailSender() { return new SmtpMailSender(); }

    @Bean
    @Profile("dev")
    public MailSender fakeMailSender() { return new ConsoleLogMailSender(); }
}
```

### `@ConditionalOnClass` / `@ConditionalOnMissingBean` / `@ConditionalOnProperty`
Bộ annotation nền tảng của cơ chế Auto-configuration — thường gặp khi đọc source code các starter, hoặc khi tự viết auto-configuration riêng cho thư viện nội bộ công ty.

```java
@Configuration
@ConditionalOnClass(DataSource.class)       // chỉ kích hoạt nếu classpath có driver JDBC
@ConditionalOnMissingBean(DataSource.class) // chỉ kích hoạt nếu người dùng CHƯA tự khai báo DataSource
public class CustomDataSourceAutoConfiguration { /* ... */ }
```

---

## 3. Spring Web MVC — REST API

### `@RequestMapping`
Annotation gốc để ánh xạ URL + HTTP method tới Controller/method. Các annotation `@GetMapping`, `@PostMapping`... là dạng rút gọn chuyên biệt của nó.

```java
@RequestMapping(value = "/api/v1/orders", method = RequestMethod.GET)
public List<OrderDTO> getAllOrders() { /* ... */ }
```

### `@GetMapping` / `@PostMapping` / `@PutMapping` / `@PatchMapping` / `@DeleteMapping`
Dạng rút gọn của `@RequestMapping` cho từng HTTP method cụ thể — cách dùng phổ biến nhất trong thực tế.

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    @GetMapping public List<OrderDTO> getAll() { /* ... */ }
    @GetMapping("/{id}") public OrderDTO getById(@PathVariable Long id) { /* ... */ }
    @PostMapping public OrderDTO create(@RequestBody CreateOrderRequest req) { /* ... */ }
    @PutMapping("/{id}") public OrderDTO update(@PathVariable Long id, @RequestBody UpdateOrderRequest req) { /* ... */ }
    @DeleteMapping("/{id}") public void delete(@PathVariable Long id) { /* ... */ }
}
```

### `@PathVariable`
Lấy giá trị từ 1 phần của URL path (`/orders/{id}`).

```java
@GetMapping("/orders/{orderId}/items/{itemId}")
public OrderItemDTO getItem(@PathVariable Long orderId, @PathVariable Long itemId) { /* ... */ }
```

### `@RequestParam`
Lấy giá trị từ query string (`?status=PENDING&page=0`).

```java
@GetMapping("/orders")
public Page<OrderDTO> search(
        @RequestParam(required = false) OrderStatus status,
        @RequestParam(defaultValue = "0") int page) { /* ... */ }
```

### `@RequestBody`
Deserialize toàn bộ body của HTTP request (thường là JSON) thành object Java.

```java
@PostMapping("/orders")
public OrderDTO create(@RequestBody CreateOrderRequest request) { /* ... */ }
```

### `@ResponseBody`
Chỉ định giá trị method trả về được serialize trực tiếp thành body response (JSON), không phải tên view. Đã gộp sẵn trong `@RestController`, chỉ cần khai báo riêng khi dùng `@Controller` thuần.

```java
@Controller
public class LegacyController {
    @GetMapping("/api/ping")
    @ResponseBody
    public String ping() { return "pong"; }
}
```

### `@RequestHeader`
Lấy giá trị từ HTTP header của request.

```java
@PostMapping("/webhooks/vnpay")
public ResponseEntity<Void> handleWebhook(
        @RequestBody String payload,
        @RequestHeader("X-VNPay-Signature") String signature) { /* ... */ }
```

### `@ResponseStatus`
Chỉ định HTTP status code trả về, thường gắn trên class exception để tự động map sang status code tương ứng khi exception được ném ra.

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(String orderId) { super("Không tìm thấy đơn hàng: " + orderId); }
}
```

### `@ControllerAdvice` / `@RestControllerAdvice`
Đánh dấu class xử lý exception **tập trung** cho toàn bộ (hoặc 1 nhóm) Controller, kết hợp `@ExceptionHandler`. `@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(OrderNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(new ErrorResponse(ex.getMessage()));
    }
}
```

### `@ExceptionHandler`
Đánh dấu method xử lý 1 loại exception cụ thể, dùng bên trong `@RestControllerAdvice` (toàn cục) hoặc trực tiếp trong 1 Controller (cục bộ).

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<Map<String, String>> handleValidation(MethodArgumentNotValidException ex) {
    Map<String, String> errors = new HashMap<>();
    ex.getBindingResult().getFieldErrors()
            .forEach(err -> errors.put(err.getField(), err.getDefaultMessage()));
    return ResponseEntity.badRequest().body(errors);
}
```

### `@CrossOrigin`
Cho phép CORS ở cấp Controller/method — dùng cho case đơn giản, dự án lớn nên cấu hình CORS tập trung qua `CorsConfigurationSource` (Chương 6) thay vì rải annotation này khắp nơi.

```java
@CrossOrigin(origins = "https://app.company.com")
@GetMapping("/api/v1/public/products")
public List<ProductDTO> getPublicProducts() { /* ... */ }
```

---

## 4. Bean Validation

### `@Valid`
Kích hoạt validate cho object (thường là `@RequestBody`) dựa trên các annotation ràng buộc bên trong nó (`@NotNull`, `@Size`...).

```java
@PostMapping("/orders")
public OrderDTO create(@Valid @RequestBody CreateOrderRequest request) { /* ... */ }
```

### `@NotNull` / `@NotBlank` / `@NotEmpty`
`@NotNull`: không được `null` (nhưng chuỗi rỗng `""` vẫn hợp lệ). `@NotBlank`: chuỗi không `null`, không rỗng, không chỉ chứa khoảng trắng. `@NotEmpty`: Collection/String không `null` và có ít nhất 1 phần tử/ký tự.

```java
public record CreateOrderRequest(
        @NotNull Long customerId,
        @NotBlank String shippingAddress,
        @NotEmpty List<OrderItemRequest> items
) {}
```

### `@Size`
Ràng buộc độ dài chuỗi hoặc kích thước Collection.

```java
public record RegisterRequest(
        @Size(min = 8, max = 64, message = "Mật khẩu phải từ 8-64 ký tự") String password
) {}
```

### `@Min` / `@Max`
Ràng buộc giá trị số tối thiểu/tối đa.

```java
public record CreateOrderItemRequest(
        @Min(1) @Max(1000) int quantity
) {}
```

### `@DecimalMin` / `@DecimalMax`
Giống `@Min`/`@Max` nhưng dùng cho kiểu số thập phân chính xác cao (`BigDecimal`), nhận giá trị dạng chuỗi.

```java
public record CreateOrderItemRequest(
        @DecimalMin(value = "0.01", message = "Đơn giá phải lớn hơn 0") BigDecimal unitPrice
) {}
```

### `@Email`
Kiểm tra chuỗi đúng định dạng email.

```java
public record RegisterRequest(@Email String email) {}
```

### `@Positive` / `@PositiveOrZero` / `@Negative`
Ràng buộc số dương/không âm/số âm.

```java
public record CreateOrderItemRequest(@Positive BigDecimal unitPrice) {}
```

### `@Pattern`
Kiểm tra chuỗi khớp với regular expression.

```java
public record CreateProductRequest(
        @Pattern(regexp = "^[A-Z0-9-]+$", message = "SKU chỉ được chứa chữ hoa, số và dấu gạch ngang") String sku
) {}
```

### `@Past` / `@Future` / `@PastOrPresent` / `@FutureOrPresent`
Ràng buộc giá trị ngày/giờ phải ở quá khứ hoặc tương lai so với thời điểm validate.

```java
public record CreatePromotionRequest(
        @Future(message = "Ngày kết thúc khuyến mãi phải ở tương lai") LocalDate endDate
) {}
```

### `@AssertTrue` / `@AssertFalse`
Ràng buộc giá trị `boolean` phải là `true`/`false` — thường dùng cho field kiểu "đồng ý điều khoản".

```java
public record RegisterRequest(
        @AssertTrue(message = "Bạn phải đồng ý điều khoản sử dụng") boolean termsAccepted
) {}
```

### `@Validated`
Biến thể của `@Valid` ở cấp class, kích hoạt validate cho tham số của method (dùng nhiều trong Service layer, không chỉ Controller), hỗ trợ validation group.

```java
@Validated
@Service
public class OrderService {
    public void createOrder(@Valid CreateOrderRequest request) { /* ... */ }
}
```

---

## 5. Spring Data JPA

### `@Entity`
Đánh dấu class là 1 Entity JPA, ánh xạ tới 1 bảng trong database.

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
}
```

### `@Table`
Tùy chỉnh tên bảng, index, unique constraint tương ứng với Entity (mặc định lấy theo tên class nếu không khai báo).

```java
@Table(name = "orders", indexes = @Index(name = "idx_orders_status", columnList = "status"))
```

### `@Id` / `@GeneratedValue`
`@Id` đánh dấu field là khóa chính. `@GeneratedValue` chỉ định chiến lược sinh giá trị tự động (`IDENTITY`, `SEQUENCE`, `UUID`).

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

### `@Column`
Tùy chỉnh ánh xạ cột: tên, độ dài, nullable, unique, precision.

```java
@Column(name = "total_amount", precision = 19, scale = 2, nullable = false)
private BigDecimal totalAmount;
```

### `@Enumerated`
Chỉ định cách lưu enum xuống database. **Luôn dùng `EnumType.STRING`**, không dùng `ORDINAL` (mặc định) — tránh rủi ro dữ liệu sai khi thứ tự enum thay đổi.

```java
@Enumerated(EnumType.STRING)
private OrderStatus status;
```

### `@Version`
Hiện thực Optimistic Locking — tự động tăng mỗi lần update, ném `OptimisticLockException` nếu phát hiện xung đột ghi đồng thời.

```java
@Version
private Long version;
```

### `@ManyToOne` / `@OneToMany` / `@OneToOne` / `@ManyToMany`
4 loại quan hệ ánh xạ theo khóa ngoại. **Luôn ép `fetch = FetchType.LAZY`** cho mọi loại quan hệ (kể cả `@ManyToOne`/`@OneToOne` vốn mặc định là `EAGER`) để tránh N+1 Query.

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "customer_id", nullable = false)
private Customer customer;

@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderItem> items = new ArrayList<>();
```

### `@JoinColumn`
Chỉ định cột khóa ngoại tương ứng cho quan hệ `@ManyToOne`/`@OneToOne`.

```java
@JoinColumn(name = "customer_id", nullable = false)
```

### `@EntityGraph`
Khai báo tường minh những quan hệ cần load kèm khi query, giải pháp khắc phục N+1 Query mà vẫn tương thích với `Pageable`.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    @EntityGraph(attributePaths = {"customer", "items"})
    Page<Order> findAll(Pageable pageable);
}
```

### `@Query`
Viết JPQL hoặc Native SQL tường minh khi Query Method (sinh từ tên method) không đủ diễn tả logic phức tạp.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.orderCode = :code")
    Optional<Order> findByOrderCodeWithItems(@Param("code") String code);
}
```

### `@Param`
Ánh xạ tham số method Java vào named parameter (`:paramName`) trong `@Query`.

```java
@Query("SELECT o FROM Order o WHERE o.status = :status")
List<Order> findByStatus(@Param("status") OrderStatus status);
```

### `@Modifying`
Bắt buộc đi kèm `@Query` khi câu query là `UPDATE`/`DELETE` (không phải `SELECT`), cần kết hợp `@Transactional`.

```java
@Modifying
@Transactional
@Query("UPDATE Order o SET o.status = :newStatus WHERE o.status = :oldStatus")
int bulkUpdateStatus(@Param("oldStatus") OrderStatus oldStatus, @Param("newStatus") OrderStatus newStatus);
```

---

## 6. Transaction

### `@Transactional`
Bọc method trong 1 transaction database — commit nếu thành công, rollback nếu có unchecked exception. Hoạt động dựa trên AOP Proxy nên **chỉ tác dụng khi gọi từ bên ngoài object** (self-invocation qua `this` sẽ vô hiệu hóa) và **không hoạt động trên method `private`**.

```java
@Service
public class OrderService {
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(new Order(/* ... */));
        inventoryService.reserveStock(request.sku(), request.quantity()); // lỗi -> rollback cả save() ở trên
        return order;
    }

    @Transactional(readOnly = true) // tối ưu: bỏ qua dirty-checking khi chỉ đọc
    public List<OrderDTO> getOrders() { /* ... */ }

    @Transactional(propagation = Propagation.REQUIRES_NEW) // luôn transaction MỚI độc lập
    public void logAudit(String action) { /* ... */ }
}
```

---

## 7. Spring Security

### `@EnableWebSecurity`
Bật cấu hình Spring Security tùy chỉnh cho ứng dụng web.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
        return http.build();
    }
}
```

### `@EnableMethodSecurity`
Bật khả năng phân quyền ở cấp method (`@PreAuthorize`, `@PostAuthorize`, `@Secured`).

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig { }
```

### `@PreAuthorize` / `@PostAuthorize`
Kiểm tra quyền **trước**/**sau** khi method thực thi, hỗ trợ biểu thức SpEL linh hoạt (so sánh tham số, gọi method khác).

```java
@PreAuthorize("hasRole('ADMIN') or #customerId == authentication.principal.id")
@GetMapping("/customer/{customerId}")
public List<OrderDTO> getCustomerOrders(@PathVariable Long customerId) { /* ... */ }
```

### `@Secured`
Cú pháp đơn giản hơn `@PreAuthorize`, chỉ kiểm tra role, không hỗ trợ biểu thức SpEL phức tạp.

```java
@Secured("ROLE_ADMIN")
@DeleteMapping("/{id}")
public void deleteOrder(@PathVariable Long id) { /* ... */ }
```

### `@AuthenticationPrincipal`
Lấy trực tiếp object `UserDetails` (hoặc custom principal) của user đang đăng nhập trong tham số method Controller.

```java
@GetMapping("/me")
public UserProfileDTO getProfile(@AuthenticationPrincipal UserDetails userDetails) {
    return userService.getProfile(userDetails.getUsername());
}
```

---

## 8. Async, Scheduling & Retry

### `@EnableAsync`
Bật khả năng xử lý bất đồng bộ cho ứng dụng, cần thiết để `@Async` hoạt động.

```java
@Configuration
@EnableAsync
public class AsyncConfig { }
```

### `@Async`
Method chạy trên thread riêng biệt, trả về ngay lập tức không chờ hoàn tất. Cũng dựa trên AOP Proxy nên có cùng hạn chế self-invocation như `@Transactional`.

```java
@Async("notificationExecutor")
public void sendOrderConfirmationEmail(String orderId, String email) { /* ... */ }

@Async("notificationExecutor")
public CompletableFuture<Boolean> sendSmsAsync(String phone, String message) {
    return CompletableFuture.completedFuture(smsClient.send(phone, message));
}
```

### `@EnableScheduling`
Bật khả năng chạy tác vụ định kỳ, cần thiết để `@Scheduled` hoạt động.

```java
@Configuration
@EnableScheduling
public class SchedulingConfig { }
```

### `@Scheduled`
Đánh dấu method chạy định kỳ theo cron expression hoặc khoảng thời gian cố định. Lưu ý: chạy độc lập trên **mỗi instance** JVM — cần ShedLock nếu ứng dụng chạy nhiều instance để tránh trùng lặp.

```java
@Scheduled(cron = "0 0 2 * * *") // 2h sáng mỗi ngày
public void cancelStaleOrders() { /* ... */ }

@Scheduled(fixedDelay = 60000) // 60s sau khi lần chạy TRƯỚC hoàn tất
public void syncInventorySnapshot() { /* ... */ }
```

### `@SchedulerLock`
(Thư viện ShedLock, không phải Spring gốc) Đảm bảo chỉ 1 instance thực sự chạy job khi ứng dụng scale nhiều instance.

```java
@Scheduled(cron = "0 0 2 * * *")
@SchedulerLock(name = "cancelStaleOrders", lockAtMostFor = "10m")
public void cancelStaleOrders() { /* ... */ }
```

### `@Retryable` / `@Recover`
(Spring Retry) Tự động thử lại method khi gặp exception chỉ định, `@Recover` định nghĩa method fallback khi hết số lần retry.

```java
@Retryable(retryFor = RestClientException.class, maxAttempts = 3, backoff = @Backoff(delay = 2000))
public void deliverWebhook(WebhookPayload payload) { /* ... */ }

@Recover
public void recoverDeliverWebhook(RestClientException e, WebhookPayload payload) {
    log.error("Gửi webhook thất bại sau nhiều lần thử", e);
}
```

---

## 9. Spring AOP

### `@Aspect`
Đánh dấu class chứa logic "cắt ngang" (cross-cutting concern) áp dụng cho nhiều method khác nhau — logging, audit, đo hiệu năng.

```java
@Aspect
@Component
public class AuditLogAspect {
    @Around("@annotation(auditLog)")
    public Object logAudit(ProceedingJoinPoint joinPoint, AuditLog auditLog) throws Throwable {
        Object result = joinPoint.proceed();
        log.info("AUDIT [{}] method={}", auditLog.action(), joinPoint.getSignature().getName());
        return result;
    }
}
```

### `@Before` / `@After` / `@Around` / `@AfterReturning` / `@AfterThrowing`
Các loại Advice — thời điểm logic AOP được chèn vào so với method gốc. `@Around` mạnh nhất (kiểm soát toàn bộ, kể cả có gọi method gốc hay không), các loại còn lại chỉ chèn tại 1 thời điểm cố định.

```java
@Before("execution(* com.company.orderservice.service.*.*(..))")
public void logBeforeServiceCall(JoinPoint joinPoint) {
    log.debug("Gọi method: {}", joinPoint.getSignature());
}
```

### `@Pointcut`
Định nghĩa 1 biểu thức điều kiện (matching rule) tái sử dụng được cho nhiều Advice khác nhau.

```java
@Pointcut("within(com.company.orderservice.service..*)")
public void serviceLayer() { }

@Around("serviceLayer()")
public Object measureTime(ProceedingJoinPoint joinPoint) throws Throwable { /* ... */ }
```

---

## 10. Caching

### `@EnableCaching`
Bật cơ chế Spring Cache Abstraction cho ứng dụng.

```java
@Configuration
@EnableCaching
public class CacheConfig { }
```

### `@Cacheable`
Lưu kết quả trả về của method vào cache theo key chỉ định; lần gọi sau với cùng key sẽ trả thẳng từ cache, không chạy lại method.

```java
@Cacheable(value = "products", key = "#sku")
public ProductDTO getProduct(String sku) { /* ... */ }
```

### `@CachePut`
Luôn chạy method thật, đồng thời cập nhật lại giá trị mới vào cache — dùng cho thao tác update.

```java
@CachePut(value = "products", key = "#result.sku()")
public ProductDTO updateProduct(String sku, UpdateProductRequest request) { /* ... */ }
```

### `@CacheEvict`
Xóa 1 entry (hoặc toàn bộ, với `allEntries = true`) khỏi cache — dùng khi dữ liệu bị xóa hoặc cần làm mới toàn bộ.

```java
@CacheEvict(value = "products", key = "#sku")
public void deleteProduct(String sku) { /* ... */ }
```

---

## 11. Messaging (RabbitMQ / Kafka)

### `@RabbitListener`
Đánh dấu method là Consumer lắng nghe message từ 1 queue RabbitMQ cụ thể.

```java
@RabbitListener(queues = "order.created.queue")
public void handleOrderCreated(OrderCreatedEvent event) {
    inventoryService.deductStock(event.sku(), event.quantity());
}
```

### `@KafkaListener`
Đánh dấu method là Consumer lắng nghe message từ 1 topic Kafka cụ thể.

```java
@KafkaListener(topics = "order-events", groupId = "inventory-service")
public void consume(OrderCreatedEvent event, Acknowledgment ack) {
    inventoryService.deductStock(event.sku(), event.quantity());
    ack.acknowledge();
}
```

---

## 12. Testing

### `@Test`
(JUnit 5/Jupiter) Đánh dấu 1 test method.

```java
@Test
void calculateTotal_shouldSumAllItems() {
    assertEquals(BigDecimal.valueOf(250_000), order.calculateTotal());
}
```

### `@BeforeEach` / `@AfterEach` / `@BeforeAll` / `@AfterAll`
Hook chạy trước/sau mỗi test method (`Each`) hoặc trước/sau toàn bộ test class (`All`, method phải `static`).

```java
@BeforeEach
void setUp() { order = new Order("ORD-001", 1L); }
```

### `@DisplayName`
Đặt tên hiển thị dễ đọc cho test trong báo cáo, thay vì dùng tên method.

```java
@Test
@DisplayName("Tính tổng tiền đơn hàng phải cộng đúng tất cả OrderItem")
void calculateTotal_shouldSumAllItems() { /* ... */ }
```

### `@ParameterizedTest`
Chạy cùng 1 logic test với nhiều bộ dữ liệu đầu vào khác nhau, kết hợp `@ValueSource`, `@CsvSource`...

```java
@ParameterizedTest
@ValueSource(ints = {-1, -100, -9999})
void orderItem_shouldRejectNegativeQuantity(int invalidQuantity) {
    assertThrows(IllegalArgumentException.class, () -> new OrderItem("SKU-001", invalidQuantity, BigDecimal.TEN));
}
```

### `@ExtendWith(MockitoExtension.class)`
Kích hoạt Mockito trong JUnit 5, cho phép dùng `@Mock`/`@InjectMocks` mà không cần khởi động Spring Context.

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock private OrderRepository orderRepository;
    @InjectMocks private OrderService orderService;
}
```

### `@Mock` / `@InjectMocks`
`@Mock` tạo object giả cho dependency. `@InjectMocks` tạo object thật của class đang test, tự động tiêm các `@Mock` vào qua constructor.

```java
@Mock private InventoryService inventoryService;
@InjectMocks private OrderService orderService;
```

### `@SpringBootTest`
Khởi động toàn bộ (hoặc gần như toàn bộ) `ApplicationContext` thật — dùng cho Integration Test cần xác nhận nhiều Bean phối hợp đúng.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class OrderServiceIntegrationTest {
    @Autowired private OrderService orderService;
}
```

### `@WebMvcTest`
Chỉ load tầng Controller + Bean liên quan MVC, không load Service/Repository thật — nhanh hơn nhiều so với `@SpringBootTest` khi chỉ cần test tầng HTTP.

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired private MockMvc mockMvc;
    @MockBean private OrderService orderService;
}
```

### `@MockBean`
Thay thế 1 Bean thật trong Spring Context bằng mock — dùng trong `@WebMvcTest`/`@SpringBootTest` khi cần cô lập 1 phần phụ thuộc.

```java
@MockBean
private OrderService orderService;
```

### `@DataJpaTest`
Chỉ load tầng Repository/JPA (dùng DB in-memory hoặc Testcontainers), không load Controller/Service — dùng để test riêng Query Method/`@Query`.

```java
@DataJpaTest
class OrderRepositoryTest {
    @Autowired private OrderRepository orderRepository;
}
```

### `@Testcontainers` / `@Container`
`@Testcontainers` kích hoạt JUnit 5 extension tự động quản lý vòng đời Docker container. `@Container` đánh dấu field container cụ thể (VD: PostgreSQL thật) sẽ được tự động khởi động trước khi test chạy.

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
}
```

### `@WithMockUser`
Mô phỏng 1 user đã xác thực với username/role chỉ định — dùng để test tầng phân quyền mà không cần login thật.

```java
@Test
@WithMockUser(roles = "ADMIN")
void deleteOrder_shouldReturn204_whenUserIsAdmin() throws Exception {
    mockMvc.perform(delete("/api/v1/orders/ORD-001")).andExpect(status().isNoContent());
}
```

---

## 13. Lombok

> Lombok là thư viện sinh code tự động lúc compile (getter/setter/constructor/builder...), không thuộc hệ sinh thái Spring nhưng gần như luôn xuất hiện cùng dự án Spring Boot. Từ Java 16+, **Record** (Chương 2) đã thay thế phần lớn nhu cầu dùng Lombok cho DTO bất biến — Lombok vẫn hữu ích cho JPA Entity (cần mutable, cần constructor rỗng).

### `@Getter` / `@Setter`
Tự động sinh getter/setter cho mọi field (đặt ở cấp class) hoặc 1 field cụ thể (đặt ở cấp field).

```java
@Getter @Setter
public class Product {
    private String name;
    private BigDecimal price;

    @Setter(AccessLevel.NONE) // field này KHÔNG sinh setter, dù class có @Setter
    private final String sku;
}
```

### `@ToString`
Tự sinh method `toString()` liệt kê mọi field. Dùng `exclude` để loại field nhạy cảm hoặc field gây vòng lặp vô hạn.

```java
@ToString(exclude = "password")
public class User {
    private String username;
    private String password;
}
```

### `@EqualsAndHashCode`
Tự sinh `equals()`/`hashCode()` dựa trên field. Với JPA Entity, nên dùng `of = {...}` để chỉ định đúng business key, tránh dựa vào toàn bộ field (bao gồm cả quan hệ `LAZY`, dễ gây lỗi/hiệu năng kém).

```java
@EqualsAndHashCode(of = "sku") // chỉ so sánh dựa trên sku, không phải mọi field
@Entity
public class Product {
    @Id private Long id;
    private String sku;
}
```

### `@NoArgsConstructor` / `@AllArgsConstructor` / `@RequiredArgsConstructor`
Sinh constructor không tham số / đủ tham số cho mọi field / chỉ tham số cho field `final` + `@NonNull` (phù hợp tự nhiên với Constructor Injection).

```java
@RequiredArgsConstructor
@Service
public class OrderService {
    private final OrderRepository orderRepository; // Lombok tự sinh constructor nhận tham số này
}
```

```java
@NoArgsConstructor // BẮT BUỘC cho JPA Entity - Hibernate cần constructor rỗng (đã học ở Chương 5)
@AllArgsConstructor
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;
    private String orderCode;
}
```

### `@Data`
Gộp `@Getter` + `@Setter` + `@ToString` + `@EqualsAndHashCode` + `@RequiredArgsConstructor` — annotation "tất cả trong 1" phổ biến trong tutorial, **nhưng cần cẩn trọng khi dùng cho JPA Entity** (xem cảnh báo cuối mục).

```java
@Data
public class ProductSearchCriteria { // phù hợp cho class DTO đơn giản, KHÔNG phải Entity
    private String keyword;
    private BigDecimal minPrice;
}
```

### `@Value` (Lombok)
Tạo class **bất biến** (immutable) — mọi field tự động `private final`, chỉ có getter, không có setter, tự sinh constructor đủ tham số. **Lưu ý dễ nhầm**: đây là annotation của **Lombok**, khác hoàn toàn với `@Value` của **Spring** (inject giá trị cấu hình) dù trùng tên — 2 annotation đến từ 2 package khác nhau (`lombok.Value` vs `org.springframework.beans.factory.annotation.Value`).

```java
@lombok.Value // nên import tường minh hoặc dùng full qualified name để tránh nhầm với Spring @Value
public class Money {
    BigDecimal amount;
    String currency;
}
```

### `@Builder`
Tự động sinh Builder Pattern (đã học ở Chương 2) cho class, giảm boilerplate khi object có nhiều field/nhiều field optional.

```java
@Builder
public class ProductSearchCriteria {
    private String keyword;
    private BigDecimal minPrice;
    private int page;
}
// Sử dụng: ProductSearchCriteria.builder().keyword("laptop").page(0).build();
```

### `@Slf4j`
Tự động khai báo sẵn field `log` (SLF4J Logger) cho class, không cần viết `LoggerFactory.getLogger(...)` thủ công.

```java
@Slf4j
@Service
public class OrderService {
    public void createOrder() {
        log.info("Tạo đơn hàng mới");
    }
}
```

### `@With`
Sinh method "wither" — tạo 1 object **mới** với 1 field được thay đổi, giữ nguyên các field khác (phù hợp tư duy immutable, tương tự spread operator `{...obj, field: newValue}` trong JS).

```java
@With
@AllArgsConstructor
public class OrderDTO {
    private final String orderCode;
    private final OrderStatus status;
}
// Sử dụng: OrderDTO updated = original.withStatus(OrderStatus.CONFIRMED); // original KHÔNG đổi
```

### `@SneakyThrows`
Cho phép ném checked exception mà không cần khai báo `throws` trong method signature — dùng thận trọng, vì nó "che giấu" checked exception khỏi compiler, đi ngược triết lý rõ ràng đã bàn ở Chương 1 (Exception Handling).

```java
@SneakyThrows
public String readConfigFile(Path path) {
    return Files.readString(path); // ném IOException (checked) nhưng không cần khai báo throws
}
```

**Cảnh báo quan trọng khi dùng Lombok với JPA Entity**: Tránh dùng `@Data`/`@ToString`/`@EqualsAndHashCode` mặc định (dựa trên mọi field) cho Entity — dễ vô tình load quan hệ `LAZY` khi gọi `toString()` (gây N+1 hoặc `LazyInitializationException`), hoặc gây `StackOverflowError` khi 2 Entity tham chiếu vòng lẫn nhau. Nên khai báo riêng lẻ `@Getter`/`@Setter`/`@NoArgsConstructor`, dùng `@EqualsAndHashCode(of = "businessKey")` và `@ToString(exclude = {...})` tường minh cho từng quan hệ.

---

## 14. Jackson (JSON Serialization/Deserialization)

> Jackson là thư viện xử lý JSON mặc định của Spring Boot (tự động cấu hình sẵn `ObjectMapper` khi có `spring-boot-starter-web`) — chịu trách nhiệm convert giữa object Java và JSON ở cả 2 chiều: `@RequestBody` (JSON → Java) và response trả về (Java → JSON).

### `@JsonProperty`
Ánh xạ tên field Java sang tên khác trong JSON (khi tên JSON không theo chuẩn `camelCase`, hoặc muốn tên public API khác với tên field nội bộ).

```java
public record ProductDTO(
        @JsonProperty("product_sku") String sku,
        @JsonProperty("product_name") String name
) {}
```

### `@JsonIgnore`
Loại bỏ hoàn toàn 1 field khỏi cả serialize (Java → JSON) lẫn deserialize (JSON → Java) — dùng cho dữ liệu nhạy cảm (password) hoặc field gây vòng lặp giữa các Entity quan hệ.

```java
public class User {
    private String username;

    @JsonIgnore // KHÔNG BAO GIỜ trả password ra API response
    private String password;
}
```

### `@JsonIgnoreProperties`
Đặt ở cấp class, bỏ qua field không xác định lúc deserialize (tránh lỗi khi JSON có field thừa so với class Java), hoặc chỉ định bỏ qua 1 số field cụ thể.

```java
@JsonIgnoreProperties(ignoreUnknown = true) // JSON có field lạ -> bỏ qua thay vì ném exception
public record ExchangeRateResponse(String base, Map<String, BigDecimal> rates) {}
```

### `@JsonInclude`
Chỉ định điều kiện 1 field có được đưa vào JSON output hay không — phổ biến nhất là loại bỏ field có giá trị `null` để response gọn hơn.

```java
@JsonInclude(JsonInclude.Include.NON_NULL) // field null sẽ KHÔNG xuất hiện trong JSON response
public record OrderDTO(String orderCode, String discountCode) {}
```

### `@JsonFormat`
Định dạng cách serialize/deserialize kiểu dữ liệu ngày giờ (`LocalDateTime`, `LocalDate`) — mặc định Jackson có thể trả về dạng mảng số `[2026,8,8,10,30]` nếu thiếu module `JavaTimeModule`, `@JsonFormat` giúp kiểm soát tường minh định dạng chuỗi mong muốn.

```java
public record OrderDTO(
        @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss") LocalDateTime createdAt
) {}
```

### `@JsonCreator` / `@JsonProperty` (trên constructor)
Chỉ định tường minh constructor nào Jackson dùng để deserialize JSON thành object — cần thiết khi class có nhiều constructor hoặc dùng Record với constructor tùy biến (compact constructor có validate, đã học ở Chương 2).

```java
public record OrderItemRequest(String sku, int quantity, BigDecimal unitPrice) {

    @JsonCreator
    public OrderItemRequest(
            @JsonProperty("sku") String sku,
            @JsonProperty("quantity") int quantity,
            @JsonProperty("unit_price") BigDecimal unitPrice) { // JSON dùng snake_case, Java dùng camelCase
        this.sku = sku;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
    }
}
```

### `@JsonNaming`
Áp dụng 1 chiến lược đặt tên cho **toàn bộ field** của class cùng lúc, thay vì phải `@JsonProperty` từng field — phổ biến nhất là chuyển `camelCase` (Java) sang `snake_case` (JSON, quy ước phổ biến của nhiều API).

```java
@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)
public record ProductDTO(String productSku, BigDecimal unitPrice) {}
// Java: productSku, unitPrice -> JSON: product_sku, unit_price (tự động, không cần @JsonProperty từng field)
```

### `@JsonManagedReference` / `@JsonBackReference`
Giải quyết vấn đề **vòng lặp vô hạn** khi serialize 2 Entity có quan hệ 2 chiều (VD: `Order` có `List<OrderItem>`, `OrderItem` có `Order`) — `@JsonManagedReference` (phía "cha", được serialize bình thường) và `@JsonBackReference` (phía "con", bị loại khỏi JSON để cắt vòng lặp).

```java
public class Order {
    @JsonManagedReference
    private List<OrderItem> items;
}

public class OrderItem {
    @JsonBackReference // KHÔNG serialize field "order" này, tránh Order -> items -> order -> items -> ...
    private Order order;
}
```

**Lưu ý thực tế quan trọng**: Trong kiến trúc REST API chuẩn (đã xây dựng xuyên suốt Chương 4-8), **Entity JPA không nên trả trực tiếp ra API** — luôn convert sang DTO/Record riêng trước khi serialize (đã nhấn mạnh ở Chương 1 mục Encapsulation và Chương 5). Nếu tuân thủ đúng nguyên tắc này, phần lớn annotation Jackson ở trên (`@JsonManagedReference`, `@JsonIgnore` cho password...) chỉ cần áp dụng cho **DTO**, hiếm khi cần áp dụng trực tiếp lên Entity — vì DTO vốn đã được thiết kế phẳng, không mang quan hệ vòng lặp hay field nhạy cảm.

---

## 15. Spring Cloud & Microservices

> Nhóm annotation dùng trong kiến trúc Microservices (Chương 8, mục 8.3) — Service Discovery, giao tiếp nội bộ giữa các service, cấu hình tập trung.

### `@EnableEurekaServer`
Biến 1 ứng dụng Spring Boot thành Eureka Server — "sổ đăng ký" trung tâm để các Microservice khác đăng ký/tra cứu lẫn nhau.

```java
@SpringBootApplication
@EnableEurekaServer
public class DiscoveryServerApplication {
    public static void main(String[] args) { SpringApplication.run(DiscoveryServerApplication.class, args); }
}
```

### `@EnableDiscoveryClient`
Đánh dấu 1 Microservice là **client** của Service Discovery — tự động đăng ký bản thân vào Eureka Server khi khởi động. Từ Spring Cloud phiên bản mới, thường **không bắt buộc** khai báo tường minh nếu đã có dependency `spring-cloud-starter-netflix-eureka-client` trong classpath (tự động kích hoạt qua Auto-configuration, giống nguyên lý ở Chương 4).

```java
@SpringBootApplication
@EnableDiscoveryClient
public class OrderServiceApplication { }
```

### `@EnableFeignClients`
Kích hoạt cơ chế quét và tạo implementation tự động cho mọi interface `@FeignClient` trong package chỉ định — bắt buộc phải có annotation này ở class `@SpringBootApplication`, nếu không mọi `@FeignClient` sẽ không hoạt động.

```java
@SpringBootApplication
@EnableFeignClients(basePackages = "com.company.orderservice.client")
public class OrderServiceApplication { }
```

### `@FeignClient`
Khai báo 1 interface là "client ảo" gọi tới Microservice khác — Feign tự tra cứu địa chỉ qua Eureka (dựa trên `name`) và tự sinh implementation lúc runtime (dùng Dynamic Proxy, cùng cơ chế với Spring Data JPA Repository đã học ở Chương 5).

```java
@FeignClient(name = "inventory-service")
public interface InventoryClient {
    @PostMapping("/api/v1/inventory/reserve")
    ReservationResponse reserveStock(@RequestBody ReserveStockRequest request);
}
```

### `@EnableConfigServer`
Biến 1 ứng dụng Spring Boot thành Config Server — phục vụ cấu hình tập trung (đọc từ Git repository) cho mọi Microservice khác trong hệ thống.

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication { }
```

### `@RefreshScope`
Đánh dấu Bean có thể **refresh lại cấu hình lúc runtime** (khi Config Server có thay đổi, kết hợp Spring Cloud Bus) mà không cần restart ứng dụng — Bean sẽ được tạo lại với giá trị cấu hình mới nhất.

```java
@RefreshScope
@Component
public class FeatureToggleConfig {
    @Value("${feature.new-checkout-flow.enabled}")
    private boolean newCheckoutFlowEnabled;
}
```

### `@CircuitBreaker`
(Resilience4j) Bọc method bằng cơ chế Circuit Breaker — tự động "ngắt mạch" khi tỷ lệ lỗi vượt ngưỡng, gọi `fallbackMethod` thay vì tiếp tục gọi service đang gặp sự cố.

```java
@CircuitBreaker(name = "inventoryService", fallbackMethod = "reserveStockFallback")
public ReservationResponse reserveStock(ReserveStockRequest request) {
    return inventoryClient.reserveStock(request);
}

public ReservationResponse reserveStockFallback(ReserveStockRequest request, Throwable t) {
    return ReservationResponse.pendingManualReview(request.sku());
}
```

### `@Retry`
(Resilience4j) Tự động thử lại method khi gặp lỗi, cấu hình số lần thử và thời gian chờ giữa các lần qua `application.yml` (tương tự `@Retryable` của Spring Retry ở mục 8, nhưng thuộc hệ sinh thái Resilience4j — 2 thư viện không dùng chung annotation).

```java
@Retry(name = "inventoryService")
@CircuitBreaker(name = "inventoryService", fallbackMethod = "reserveStockFallback")
public ReservationResponse reserveStock(ReserveStockRequest request) { /* ... */ }
```

### `@TimeLimiter`
(Resilience4j) Giới hạn thời gian tối đa method (bất đồng bộ, trả về `CompletableFuture`) được phép chạy — quá thời gian sẽ tự động timeout và kích hoạt fallback.

```java
@TimeLimiter(name = "inventoryService")
public CompletableFuture<ReservationResponse> reserveStockAsync(ReserveStockRequest request) {
    return CompletableFuture.supplyAsync(() -> inventoryClient.reserveStock(request));
}
```

### `@NewSpan` / `@SpanTag`
(Micrometer Tracing) Tạo tường minh 1 "span" mới trong Distributed Tracing cho method không tự động được trace (VD: method nội bộ không qua HTTP), `@SpanTag` gắn thêm metadata vào span đó.

```java
@NewSpan("reserve-stock-processing")
public void processReservation(@SpanTag("sku") String sku) { /* ... */ }
```

---

## 16. WebSocket, Spring Batch & Spring AI

### `@EnableWebSocketMessageBroker`
Bật hạ tầng WebSocket dùng giao thức STOMP (Simple Text Oriented Messaging Protocol) cho ứng dụng — cần thiết để đăng ký message broker và STOMP endpoint.

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic");
        registry.setApplicationDestinationPrefixes("/app");
    }
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").withSockJS();
    }
}
```

### `@MessageMapping`
Ánh xạ message STOMP gửi tới 1 destination cụ thể (VD: `/app/chat.send`) tới method xử lý — tương đương `@RequestMapping` nhưng cho WebSocket thay vì HTTP.

```java
@Controller
public class ChatController {
    @MessageMapping("/chat.send") // client gửi message tới /app/chat.send
    @SendTo("/topic/chat")        // kết quả được broadcast tới mọi client subscribe /topic/chat
    public ChatMessage sendMessage(ChatMessage message) {
        return message;
    }
}
```

### `@SendTo` / `@SendToUser`
`@SendTo` broadcast kết quả trả về của method tới 1 topic cho **mọi** client đang subscribe. `@SendToUser` chỉ gửi riêng cho **1 user cụ thể** (dùng cho thông báo cá nhân, không phải broadcast).

```java
@MessageMapping("/order.status")
@SendToUser("/queue/order-updates") // chỉ gửi cho đúng user đã gọi, không broadcast cho tất cả
public OrderStatusUpdate getStatus(@Payload String orderCode) { /* ... */ }
```

### `@SubscribeMapping`
Xử lý ngay khi client **mới subscribe** vào 1 destination (khác `@MessageMapping` xử lý khi có message gửi tới) — thường dùng để trả dữ liệu ban đầu ngay lúc client vừa kết nối.

```java
@SubscribeMapping("/topic/orders/{orderId}")
public OrderStatusUpdate onSubscribe(@DestinationVariable String orderId) {
    return orderService.getCurrentStatus(orderId); // gửi trạng thái hiện tại ngay khi client vừa subscribe
}
```

### `@EnableBatchProcessing`
Bật hạ tầng Spring Batch cho ứng dụng (Job Repository, Job Launcher...) — cần thiết trước khi khai báo `Job`/`Step` như đã học ở Chương 8, mục 8.5.4. Lưu ý: từ Spring Boot 2.5+, annotation này thường **không cần khai báo tường minh** nữa vì Spring Boot đã tự động cấu hình sẵn khi có `spring-boot-starter-batch` trong classpath.

```java
@Configuration
@EnableBatchProcessing // chỉ cần khai báo tường minh khi cần TÙY BIẾN cấu hình mặc định
public class BatchConfig { }
```

### `@StepScope`
Đánh dấu Bean chỉ được tạo mới **cho mỗi lần Step chạy** (thay vì Singleton dùng chung) — bắt buộc khi Bean cần đọc tham số truyền vào lúc chạy Job (`JobParameters`), như ví dụ `FlatFileItemReader` đọc tên file đầu vào ở Chương 8.

```java
@Bean
@StepScope
public FlatFileItemReader<OrderRecord> reader(
        @Value("#{jobParameters['inputFile']}") String inputFile) {
    return new FlatFileItemReaderBuilder<OrderRecord>()
            .name("orderRecordReader")
            .resource(new FileSystemResource(inputFile))
            .build();
}
```

### `@Tool`
(Spring AI) Đánh dấu 1 method Java là "công cụ" (function calling) mà LLM có thể **chủ động gọi** khi cần dữ liệu/hành động thực tế, thay vì chỉ trả lời dựa trên training data — đã học chi tiết ở Chương 8, mục 8.11.4.

```java
public class OrderLookupTool {
    @Tool(description = "Tra cứu trạng thái hiện tại của 1 đơn hàng theo mã đơn hàng")
    public String getOrderStatus(String orderCode) {
        return orderRepository.findByOrderCode(orderCode)
                .map(order -> "Trạng thái: " + order.getStatus())
                .orElse("Không tìm thấy đơn hàng");
    }
}
```

**Cảnh báo bảo mật (nhắc lại từ Chương 8)**: Không đăng ký `@Tool` cho method có khả năng thay đổi dữ liệu nhạy cảm (xóa, hoàn tiền, đổi quyền) mà không có lớp xác thực/phê duyệt bổ sung — input dẫn tới việc gọi tool cần được coi nguy hiểm tương đương input từ người dùng chưa xác thực (rủi ro Prompt Injection).

---

## Bảng tra nhanh theo tình huống thường gặp

| Bạn đang làm gì | Annotation cần nhớ |
|---|---|
| Khai báo 1 Bean tự viết | `@Component` / `@Service` / `@Repository` |
| Khai báo Bean cho thư viện bên ngoài | `@Configuration` + `@Bean` |
| Inject dependency | Constructor injection (không cần `@Autowired` nếu chỉ có 1 constructor) |
| Viết REST endpoint | `@RestController`, `@GetMapping`/`@PostMapping`... |
| Validate dữ liệu đầu vào | `@Valid` + `@NotNull`/`@Size`/`@Email`... |
| Xử lý lỗi tập trung | `@RestControllerAdvice` + `@ExceptionHandler` |
| Ánh xạ Entity-bảng | `@Entity`, `@Id`, `@Column`, `@ManyToOne(fetch = LAZY)` |
| Bọc logic trong transaction | `@Transactional` |
| Phân quyền API | `@PreAuthorize` |
| Chạy nền không chờ kết quả | `@Async` |
| Chạy định kỳ | `@Scheduled` (+ `@SchedulerLock` nếu nhiều instance) |
| Cache kết quả method | `@Cacheable` |
| Sinh boilerplate cho Entity | `@Getter`/`@Setter`/`@NoArgsConstructor` (Lombok, tránh `@Data` cho Entity) |
| Đổi tên field khi trả JSON | `@JsonProperty` hoặc `@JsonNaming` (snake_case toàn class) |
| Ẩn field nhạy cảm khỏi JSON | `@JsonIgnore` |
| Bỏ field `null` khỏi JSON response | `@JsonInclude(NON_NULL)` |
| Gọi Microservice khác qua Service Discovery | `@FeignClient` (+ `@EnableFeignClients` ở class chính) |
| Chống cascading failure khi gọi service khác | `@CircuitBreaker` + `@Retry` (Resilience4j) |
| Đẩy dữ liệu realtime qua WebSocket | `@MessageMapping` + `@SendTo`/`@SendToUser` |
| Đọc tham số Job lúc chạy Spring Batch | `@StepScope` |
| Cho AI gọi lại method Java thật | `@Tool` (Spring AI) |
| Viết Unit Test cô lập | `@ExtendWith(MockitoExtension.class)` + `@Mock`/`@InjectMocks` |
| Viết Integration Test | `@SpringBootTest` hoặc `@WebMvcTest`/`@DataJpaTest` |
