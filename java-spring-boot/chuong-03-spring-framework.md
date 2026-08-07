# CHƯƠNG 3: SPRING FRAMEWORK CƠ BẢN

> Tài liệu đào tạo Java Backend Developer — dành cho người đã có nền tảng Backend (Node.js/Express/NestJS), chuyển sang hệ sinh thái Java/Spring Boot.

## Giới thiệu

Nếu bạn từng dùng NestJS, bạn đã quen với `@Injectable()`, `@Module()`, constructor injection — thực ra **NestJS lấy cảm hứng trực tiếp từ Spring Framework**. Chương này cho bạn thấy "bản gốc" hoạt động ra sao, và quan trọng hơn: **tại sao** nó được thiết kế như vậy, dựa trên nền tảng Reflection + Design Pattern đã học ở Chương 2.

Spring giải quyết một vấn đề cốt lõi trong enterprise development: khi hệ thống có hàng trăm class phụ thuộc lẫn nhau (`OrderService` cần `InventoryService`, `PaymentService`, `NotificationService`...), việc tự tay `new` từng object và nối chúng lại với nhau trở thành cơn ác mộng bảo trì. Spring giải quyết bằng **IoC Container** — một "nhà máy" trung tâm tự động tạo object, tự động nối dependency, quản lý toàn bộ vòng đời của chúng.

Chương này là nền tảng **quan trọng nhất** trong toàn bộ tài liệu — mọi thứ trong Spring Boot (Web MVC, Data JPA, Security) đều xây dựng trên IoC Container.

---

## 3.1. Spring là gì, hệ sinh thái Spring

**Khái niệm**: Spring Framework là một framework mã nguồn mở cho Java, ra đời năm 2003 nhằm thay thế mô hình lập trình nặng nề của J2EE/EJB thời đó. Cốt lõi của Spring là **IoC Container** — mọi module khác trong hệ sinh thái Spring đều được xây dựng dựa trên nền tảng này.

**Hệ sinh thái Spring** — với người từ Node.js, có thể hình dung Spring giống như "Express + NestJS + TypeORM + Passport.js + BullMQ" nhưng thống nhất trong 1 hệ sinh thái chính thức, được duy trì bởi cùng 1 tổ chức (VMware/Broadcom):

| Module | Vai trò | Tương đương bên Node.js |
|---|---|---|
| **Spring Core** | IoC Container, DI — nền tảng của mọi module khác | Container DI của NestJS |
| **Spring MVC / WebFlux** | Xây dựng REST API, xử lý HTTP request/response | Express.js / Fastify |
| **Spring Data** | Truy cập dữ liệu (JPA, MongoDB, Redis...) qua abstraction thống nhất | TypeORM, Prisma, Mongoose |
| **Spring Security** | Authentication, Authorization | Passport.js, NestJS Guards |
| **Spring Cloud** | Microservices: Service Discovery, Config Server, Gateway | Không có tương đương trực tiếp phổ biến |
| **Spring Batch** | Xử lý batch job khối lượng lớn | Bull/BullMQ (một phần) |
| **Spring Boot** | Lớp "đóng gói" giúp khởi động nhanh, tự động cấu hình toàn bộ các module trên | Tương tự triết lý "convention over configuration" của NestJS CLI |

**Điểm khác biệt triết lý quan trọng**: Spring (không Boot) yêu cầu bạn **tự cấu hình mọi thứ tường minh** (XML hoặc Java Config) — rất linh hoạt nhưng tốn nhiều boilerplate. Đây là lý do Spring Boot ra đời (học chi tiết ở Chương 4) — cung cấp cấu hình mặc định hợp lý (sensible defaults), chỉ cần override khi cần khác biệt.

**Best Practices**: Trong dự án mới 100% ngày nay, luôn dùng **Spring Boot**, không dùng Spring thuần (trừ khi có ràng buộc đặc biệt từ hệ thống legacy). Toàn bộ tài liệu này từ đây trở đi mặc định là Spring Boot.

---

## 3.2. Inversion of Control (IoC) là gì

**Khái niệm**: IoC là một nguyên lý thiết kế, trong đó **quyền kiểm soát việc tạo object và quản lý vòng đời của object bị đảo ngược** — từ chỗ code của bạn tự `new` object và tự quản lý, sang việc một **container bên ngoài** làm việc đó thay bạn.

**Tại sao cần**: So sánh trước/sau khi áp dụng IoC:

```java
// KHÔNG có IoC - code tự kiểm soát việc tạo dependency (tight coupling)
public class OrderService {
    private final InventoryService inventoryService;

    public OrderService() {
        this.inventoryService = new InventoryServiceImpl(new InventoryRepositoryImpl());
        // OrderService phải BIẾT và tự tạo toàn bộ cây dependency.
        // Muốn đổi implementation -> phải sửa code OrderService.
        // Muốn test -> không thể thay InventoryServiceImpl bằng mock dễ dàng.
    }
}

// CÓ IoC - quyền kiểm soát được "đảo ngược" cho Container
@Service
public class OrderService {
    private final InventoryService inventoryService;

    // OrderService chỉ khai báo "tôi CẦN 1 InventoryService", KHÔNG quan tâm implementation nào,
    // KHÔNG tự tạo nó. Spring Container sẽ "tiêm" implementation phù hợp vào đây.
    public OrderService(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }
}
```

**Cách hoạt động**: Đây chính là "Hollywood Principle" — *"Don't call us, we'll call you"*. Thay vì `OrderService` chủ động gọi `new` để lấy dependency, Container chủ động tạo `OrderService` và cung cấp sẵn dependency cho nó.

**Dependency Injection (DI)** là **kỹ thuật cụ thể** để hiện thực hóa nguyên lý IoC — tiêm dependency vào object thông qua constructor, setter, hoặc field (chi tiết ở mục 3.3).

**Best Practices**: Thiết kế class nghiệp vụ luôn theo hướng "khai báo cái tôi cần" (qua constructor parameter kiểu interface) thay vì "tự đi tìm/tạo cái tôi cần". Đây là nguyên tắc nền tảng của **Dependency Inversion Principle** (chữ D trong SOLID).

**Sai lầm thường gặp**: Dùng `new` để tạo Service/Repository ngay trong code nghiệp vụ thay vì để Spring inject — phá vỡ hoàn toàn lợi ích của IoC, khiến class không thể test độc lập.

---

## 3.3. Dependency Injection: Constructor, Setter, Field Injection

Spring hỗ trợ 3 cách tiêm dependency vào Bean. Hiểu rõ sự khác biệt và biết chọn đúng cách là kỹ năng cơ bản nhưng bị đánh giá thấp — rất nhiều codebase enterprise mắc lỗi này.

```java
// 1. Constructor Injection - Best Practice, khuyến nghị CHÍNH THỨC bởi Spring team
@Service
public class OrderService {
    private final InventoryService inventoryService;
    private final PaymentService paymentService;

    // Từ Spring 4.3+, nếu class chỉ có DUY NHẤT 1 constructor,
    // KHÔNG CẦN @Autowired nữa - Spring tự động dùng constructor đó để inject
    public OrderService(InventoryService inventoryService, PaymentService paymentService) {
        this.inventoryService = inventoryService;
        this.paymentService = paymentService;
    }
}

// 2. Setter Injection - dùng cho dependency OPTIONAL (không bắt buộc phải có)
@Service
public class NotificationService {
    private SmsSender smsSender; // optional - có thể không cấu hình SMS trong 1 số môi trường

    @Autowired(required = false)
    public void setSmsSender(SmsSender smsSender) {
        this.smsSender = smsSender;
    }
}

// 3. Field Injection - CHỈ nên dùng trong test code, KHÔNG dùng trong production
@Service
public class LegacyService {
    @Autowired
    private ReportGenerator reportGenerator; // Anti-pattern trong production
}
```

**Tại sao Constructor Injection là lựa chọn đúng gần như 100% trường hợp**:

1. **Bất biến (Immutability)**: field khai báo `final`, không thể bị thay đổi sau khi khởi tạo — tránh bug do vô tình gán lại dependency giữa chừng.
2. **Fail-fast**: nếu thiếu dependency bắt buộc, ứng dụng **không khởi động được ngay lập tức** với thông báo lỗi rõ ràng — thay vì phát hiện `NullPointerException` lúc runtime khi request thực tế gọi tới.
3. **Testability**: viết Unit Test cực kỳ dễ — chỉ cần `new OrderService(mockInventory, mockPayment)`, không cần Spring Context, không cần Reflection để set field private.
4. **Phát hiện Circular Dependency ngay lập tức**: nếu `A` cần `B` trong constructor và `B` cần `A` trong constructor, Spring báo lỗi `BeanCurrentlyInCreationException` ngay lúc khởi động — buộc phải refactor thiết kế (thường là dấu hiệu vi phạm Single Responsibility Principle).
5. **Giữ class nghiệp vụ gần với POJO thuần túy** — constructor của `OrderService` không có bất kỳ annotation Spring nào bên trong, dễ tái sử dụng logic ngoài Spring nếu cần (VD: viết test không cần Spring context).

**Circular Dependency — vấn đề thực tế thường gặp**:

```java
// ❌ Circular Dependency - A cần B, B cần A
@Service
public class OrderService {
    private final CustomerService customerService;
    public OrderService(CustomerService customerService) { this.customerService = customerService; }
}

@Service
public class CustomerService {
    private final OrderService orderService;
    public CustomerService(OrderService orderService) { this.orderService = orderService; }
}
// Kết quả: ứng dụng KHÔNG khởi động được, ném BeanCurrentlyInCreationException
```

**Cách giải quyết đúng đắn** (không phải dùng `@Lazy` để "né" lỗi — đó chỉ là giải pháp tạm bợ che giấu vấn đề thiết kế):
- Tách logic dùng chung ra 1 Service thứ 3 mà cả 2 bên cùng phụ thuộc vào.
- Dùng `ApplicationEventPublisher` (Observer Pattern) để decouple 2 Service thay vì gọi trực tiếp lẫn nhau.

**So sánh: Constructor vs Setter vs Field Injection**

| Tiêu chí | Constructor Injection | Setter Injection | Field Injection |
|---|---|---|---|
| Immutability | ✅ Field `final` | ❌ Field không thể `final` | ❌ Field không thể `final` |
| Dependency bắt buộc | ✅ Phù hợp | ❌ Không phù hợp (dùng cho optional) | ⚠️ Có thể nhưng không rõ ràng |
| Dependency optional | ⚠️ Cần overload constructor phức tạp | ✅ Phù hợp nhất | ⚠️ Có thể nhưng khó kiểm soát |
| Testability | ✅ Dễ nhất | ⚠️ Trung bình | ❌ Khó nhất |
| Phát hiện lỗi thiếu dependency | ✅ Lúc khởi động | ⚠️ Có thể trễ tới lúc gọi method | ❌ Lúc runtime khi dùng tới |
| Khuyến nghị | ✅ Mặc định cho mọi trường hợp | Chỉ dùng cho dependency optional | ❌ Chỉ chấp nhận trong test |

---

## 3.4. ApplicationContext và BeanFactory

**Khái niệm**: IoC Container trong Spring có 2 interface chính tạo thành 1 hệ thống phân cấp:

- **BeanFactory**: interface gốc, cơ bản nhất — chỉ cung cấp cơ chế tạo và tra cứu Bean, dùng **lazy initialization** (chỉ tạo Bean khi thực sự được gọi `getBean()`).
- **ApplicationContext**: mở rộng `BeanFactory`, bổ sung: publish/lắng nghe event (`ApplicationEventPublisher`), quốc tế hóa (i18n), quản lý resource (file, classpath), và quan trọng nhất — **eager initialization theo mặc định** (tạo toàn bộ Singleton Bean ngay lúc khởi động ứng dụng, giúp phát hiện lỗi cấu hình sớm thay vì lúc runtime khi người dùng thực sự gọi tới).

**Cách hoạt động**: Khi ứng dụng Spring Boot khởi động qua `SpringApplication.run()`, Spring:
1. Quét (scan) toàn bộ package tìm class có `@Component`/`@Service`/`@Repository`/`@Controller`.
2. Đăng ký **BeanDefinition** cho mỗi class tìm thấy (chỉ là "bản thiết kế", chưa tạo object thật).
3. Phân tích dependency graph giữa các Bean, xác định thứ tự khởi tạo hợp lý.
4. Dùng Reflection tạo instance thật (`newInstance()`), gọi constructor với dependency đã resolve.

**Trong thực tế Spring Boot, bạn luôn làm việc với `ApplicationContext`** (cụ thể là `AnnotationConfigServletWebServerApplicationContext` khi chạy web app) — `BeanFactory` gần như chỉ còn ý nghĩa lý thuyết/lịch sử, hiếm khi bạn phải tự tương tác trực tiếp với nó.

```java
// Trong 1 số trường hợp đặc biệt cần lấy Bean thủ công (VD: viết tool CLI riêng)
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(Application.class, args);
        OrderService orderService = context.getBean(OrderService.class);
        // Cách này CHỈ dùng cho tình huống đặc biệt - KHÔNG dùng trong luồng xử lý request bình thường
    }
}
```

**Best Practices**: Không bao giờ tự tạo `ApplicationContext` thủ công hoặc tự gọi `getBean()` trong luồng xử lý nghiệp vụ bình thường — hãy để Dependency Injection lo việc đó. Việc tự gọi `context.getBean(...)` là dấu hiệu của **Service Locator Pattern**, một anti-pattern so với DI vì nó che giấu dependency thực sự của class (phải đọc cả thân method mới biết class cần gì, thay vì đọc constructor).

---

## 3.5. Spring Bean: khai báo, scope, lifecycle

### 3.5.1. Hai cách khai báo Bean

**Cách 1 — Component Scanning (stereotype annotation)**:

```java
@Component  // Bean chung chung, không thuộc layer cụ thể
public class SlugGenerator { /* ... */ }

@Service    // Bean thuộc tầng business logic - về mặt kỹ thuật KHÔNG khác @Component,
            // nhưng mang ý nghĩa ngữ nghĩa rõ ràng, giúp code dễ đọc
public class OrderService { /* ... */ }

@Repository // Bean thuộc tầng truy cập dữ liệu - Spring tự động "dịch" exception của tầng
            // persistence (VD: SQLException) sang DataAccessException thống nhất
public class OrderRepositoryImpl { /* ... */ }

@RestController // = @Controller + @ResponseBody, trả JSON trực tiếp (dùng cho REST API)
public class OrderController { /* ... */ }
```

**Cách 2 — Khai báo tường minh qua `@Configuration` + `@Bean`** (dùng cho class bên thứ 3 bạn không sở hữu source code, hoặc logic khởi tạo phức tạp):

```java
@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource dataSource(
            @Value("${spring.datasource.url}") String url,
            @Value("${spring.datasource.username}") String username,
            @Value("${spring.datasource.password}") String password) {

        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setUsername(username);
        config.setPassword(password);
        config.setMaximumPoolSize(20);
        return new HikariDataSource(config);
        // DataSource là class của thư viện HikariCP, không thể gắn @Component vào nó
        // -> BẮT BUỘC dùng @Bean để đăng ký vào Container
    }

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
                .registerModule(new JavaTimeModule())
                .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }
}
```

**So sánh: `@Component` vs `@Bean`**

| Tiêu chí | `@Component` + stereotype | `@Configuration` + `@Bean` |
|---|---|---|
| Áp dụng cho | Class bạn tự viết, có source code | Class bên thứ 3 (thư viện), hoặc logic khởi tạo phức tạp |
| Cách Spring tìm thấy | Quét classpath tự động (component scan) | Khai báo tường minh trong method |
| Độ linh hoạt khởi tạo | Thấp — chỉ dùng constructor mặc định | Cao — viết logic Java tùy ý trong method |
| Khi nào dùng | 90% trường hợp: Service, Repository, Controller tự viết | DataSource, ObjectMapper, RestTemplate, bất kỳ Bean cần cấu hình phức tạp |

### 3.5.2. Bean Scope

**Khái niệm**: Scope xác định **số lượng instance** và **vòng đời** của 1 Bean trong Container.

| Scope | Số lượng instance | Khi nào dùng |
|---|---|---|
| `singleton` (mặc định) | 1 instance duy nhất, dùng chung cho toàn bộ ứng dụng | 95% trường hợp: Service, Repository, Controller — vì chúng thường **stateless** |
| `prototype` | Tạo instance mới mỗi lần `getBean()`/inject | Bean có state riêng biệt cần thiết mỗi lần dùng (hiếm gặp trong Web API) |
| `request` | 1 instance/1 HTTP request (chỉ dùng được trong web app) | Dữ liệu cần cô lập theo từng request, VD: `RequestContextHolder` |
| `session` | 1 instance/1 HTTP session | Dữ liệu cần cô lập theo phiên người dùng |

```java
@Service
@Scope("prototype") // mỗi lần cần, Spring tạo 1 instance MỚI thay vì dùng chung
public class ReportBuilder {
    private final List<String> sections = new ArrayList<>(); // state riêng biệt mỗi lần build

    public ReportBuilder addSection(String section) {
        sections.add(section);
        return this;
    }
}
```

**Cảnh báo quan trọng**: Vì `singleton` là mặc định và chiếm 95% Bean trong ứng dụng thực tế, đây chính là lý do vì sao **Bean của bạn phải là stateless** (không có mutable field mang tính nghiệp vụ) — nếu không, mọi request đồng thời sẽ chia sẻ cùng 1 field, gây race condition (đã phân tích chi tiết ở Chương 2, mục Multithreading).

### 3.5.3. Bean Lifecycle

**Khái niệm**: Từ lúc Container khởi tạo Bean tới lúc hủy Bean, Spring cho phép "móc" (hook) vào nhiều thời điểm để chạy logic tùy chỉnh.

**Thứ tự thực thi** (rút gọn các bước quan trọng nhất với backend developer):

1. Constructor được gọi (dependency injection qua constructor xảy ra ở đây).
2. Setter injection (nếu có) được gọi.
3. `@PostConstruct` — method được gọi **ngay sau khi mọi dependency đã inject xong**, dùng để chạy logic khởi tạo cần dependency sẵn sàng (VD: warm-up cache, validate config).
4. Bean sẵn sàng sử dụng trong suốt vòng đời ứng dụng.
5. `@PreDestroy` — method được gọi **ngay trước khi Container hủy Bean** (lúc ứng dụng shutdown), dùng để giải phóng tài nguyên (đóng connection pool, thread pool...).

```java
@Component
public class CacheWarmupService {

    private final ProductRepository productRepository;
    private Map<String, Product> productCache;

    public CacheWarmupService(ProductRepository productRepository) {
        this.productRepository = productRepository;
        // KHÔNG nên gọi productRepository ở đây - dependency có thể chưa hoàn tất khởi tạo đầy đủ
    }

    @PostConstruct
    public void warmUpCache() {
        // An toàn để gọi dependency ở đây - mọi Bean liên quan đã sẵn sàng
        this.productCache = productRepository.findAll().stream()
                .collect(Collectors.toMap(Product::getSku, p -> p));
        log.info("Đã warm-up cache với {} sản phẩm", productCache.size());
    }

    @PreDestroy
    public void cleanup() {
        log.info("Giải phóng cache trước khi ứng dụng shutdown");
        this.productCache.clear();
    }
}
```

**Best Practices Bean Lifecycle**:
- Không thực hiện logic phụ thuộc vào Bean khác ngay trong constructor — dùng `@PostConstruct` để đảm bảo mọi dependency đã sẵn sàng.
- Luôn giải phóng tài nguyên (connection, thread pool, file handle) trong `@PreDestroy`, không phó mặc cho GC.
- Với thread pool tự quản lý (`ExecutorService`), luôn `shutdown()` trong `@PreDestroy` — nếu không, ứng dụng có thể "treo" khi cố gắng shutdown vì còn thread chưa dừng (non-daemon thread).

---

## Ví dụ Code: Tổng hợp toàn bộ chương trong 1 tình huống thực tế

```java
// Interface - hợp đồng nghiệp vụ
public interface InventoryService {
    void reserveStock(String sku, int quantity);
}

// Implementation - đăng ký làm Bean qua Component Scanning
@Service
public class InventoryServiceImpl implements InventoryService {

    private final InventoryRepository inventoryRepository;
    private final ApplicationEventPublisher eventPublisher;

    // Constructor Injection - Best Practice
    public InventoryServiceImpl(InventoryRepository inventoryRepository,
                                 ApplicationEventPublisher eventPublisher) {
        this.inventoryRepository = inventoryRepository;
        this.eventPublisher = eventPublisher;
    }

    @PostConstruct
    public void logStartup() {
        log.info("InventoryService đã sẵn sàng");
    }

    @Override
    public void reserveStock(String sku, int quantity) {
        Inventory inventory = inventoryRepository.findBySku(sku)
                .orElseThrow(() -> new SkuNotFoundException(sku));
        inventory.reserve(quantity);
        inventoryRepository.save(inventory);
        eventPublisher.publishEvent(new StockReservedEvent(sku, quantity));
    }
}

// Bean khai báo tường minh cho thư viện bên thứ 3
@Configuration
public class AppConfig {
    @Bean
    public RestClient inventoryApiClient(@Value("${external.inventory-api.url}") String baseUrl) {
        return RestClient.builder().baseUrl(baseUrl).build();
    }
}

// Service sử dụng, được Spring tự động inject InventoryService (interface) -> InventoryServiceImpl
@Service
public class OrderService {
    private final InventoryService inventoryService;

    public OrderService(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }

    public void checkout(String sku, int quantity) {
        inventoryService.reserveStock(sku, quantity);
    }
}
```

---

## So sánh: Bean Definition tổng hợp

| Tiêu chí | `@Component` (scan) | `@Bean` (Java Config) |
|---|---|---|
| Class tự viết | ✅ | Có thể nhưng ít dùng |
| Class thư viện bên thứ 3 | ❌ Không thể | ✅ Bắt buộc |
| Logic khởi tạo phức tạp/điều kiện | ❌ Hạn chế | ✅ Linh hoạt (viết Java thuần) |

| Tiêu chí | Singleton Scope | Prototype Scope |
|---|---|---|
| Số lượng instance | 1 | Nhiều, mỗi lần request 1 cái mới |
| Phù hợp cho | Bean stateless (Service, Repository) | Bean cần state riêng biệt mỗi lần dùng |
| Rủi ro | Race condition nếu có mutable state | Tốn chi phí tạo object nếu dùng tần suất cao |

---

## Best Practices

- Luôn dùng Spring Boot thay vì Spring thuần cho dự án mới.
- Constructor Injection là mặc định cho mọi dependency bắt buộc; Setter Injection chỉ cho dependency optional; tuyệt đối tránh Field Injection trong production code.
- Giữ Bean singleton **stateless** — không có mutable field mang tính nghiệp vụ.
- Dùng `@PostConstruct`/`@PreDestroy` đúng mục đích: khởi tạo sau khi dependency sẵn sàng, giải phóng tài nguyên trước khi shutdown.
- Dùng `@Bean` cho class bên thứ 3, `@Component`/stereotype cho class tự viết.

## Anti-patterns

- Field Injection trong production code.
- Tự gọi `context.getBean(...)` trong luồng xử lý nghiệp vụ (Service Locator anti-pattern) thay vì để DI tự động.
- Dùng `@Lazy` để né lỗi Circular Dependency thay vì refactor lại thiết kế.
- Bean singleton có mutable field không đồng bộ hóa, dùng chung cho mọi request.
- Gọi dependency Bean khác ngay trong constructor thay vì trong `@PostConstruct`.

## Bài tập

1. **Dễ**: Viết `DiscountService` interface + implementation, đăng ký làm Bean qua `@Service`, inject vào `CheckoutService` bằng Constructor Injection.
2. **Trung bình**: Tạo tình huống Circular Dependency giữa 2 Service, quan sát lỗi `BeanCurrentlyInCreationException`, sau đó refactor đúng cách bằng cách tách logic dùng chung ra Service thứ 3.
3. **Khó**: Viết 1 Bean `@Scope("prototype")` đại diện cho `InvoiceBuilder` có state riêng biệt mỗi lần build, so sánh hành vi với việc khai báo cùng class dưới dạng `singleton` (quan sát lỗi state bị "dính" giữa các lần gọi khi để singleton).

## Tổng kết

Chương này đã xây dựng nền tảng cốt lõi của Spring Framework: Spring là hệ sinh thái các module xây dựng trên nền IoC Container; IoC/DI đảo ngược quyền kiểm soát việc tạo và quản lý object từ code nghiệp vụ sang Container; Constructor Injection là lựa chọn đúng đắn gần như tuyệt đối nhờ tính bất biến, fail-fast và khả năng test; ApplicationContext là Container thực tế bạn luôn làm việc cùng trong Spring Boot; và Bean — với cách khai báo, scope, lifecycle — là đơn vị quản lý cơ bản nhất của toàn bộ Container. Chương 4 sẽ đưa những kiến thức này vào thực hành với Spring Boot cụ thể: cách khởi tạo project, cấu trúc thư mục chuẩn, và cơ chế Auto-configuration giúp Spring Boot "biết" nên tạo Bean nào mà không cần bạn khai báo XML thủ công.
