# LỘ TRÌNH HỌC JAVA SPRING BOOT (CƠ BẢN → NÂNG CAO)

## GIAI ĐOẠN 0: NỀN TẢNG BẮT BUỘC TRƯỚC KHI HỌC SPRING BOOT

### 0.1. Java Core
- Cú pháp cơ bản: biến, kiểu dữ liệu, toán tử, vòng lặp, điều kiện
- OOP: Class, Object, Kế thừa (Inheritance), Đa hình (Polymorphism), Trừu tượng (Abstraction), Đóng gói (Encapsulation)
- Interface, Abstract class
- Collection Framework: List, Set, Map, Queue
- Generic
- Exception Handling (try-catch-finally, custom exception)
- Java 8+ Features: Lambda Expression, Stream API, Optional, Functional Interface
- Multithreading cơ bản (Thread, Runnable, ExecutorService)
- I/O cơ bản (đọc/ghi file)

### 0.1.1. Java nâng cao (nên học song song hoặc sau khi vững Core)
- Design Patterns thường dùng: Singleton, Factory, Builder, Strategy, Observer
- JVM Memory Model & Garbage Collection (cơ chế, các vùng nhớ Heap/Stack)
- Reflection API & tự viết Custom Annotation (giúp hiểu cơ chế hoạt động bên trong Spring)

### 0.2. Công cụ hỗ trợ
- Cài đặt JDK (khuyến nghị JDK 17 hoặc 21 - LTS)
- IDE: IntelliJ IDEA (khuyến nghị) hoặc Eclipse/VS Code
- Maven hoặc Gradle (quản lý dependency, build project)
- Git & GitHub cơ bản

### 0.3. Kiến thức nền khác
- HTML/CSS cơ bản (để hiểu về web)
- HTTP là gì: Request/Response, Method (GET, POST, PUT, DELETE), Status Code
- REST API là gì
- JSON/XML format
- SQL cơ bản (SELECT, INSERT, UPDATE, DELETE, JOIN)

---

## GIAI ĐOẠN 1: NHẬP MÔN SPRING FRAMEWORK & SPRING BOOT

### 1.1. Spring Framework cơ bản
- Spring là gì, hệ sinh thái Spring (Spring Core, MVC, Data, Security...)
- Inversion of Control (IoC) là gì
- Dependency Injection (DI): Constructor Injection, Setter Injection, Field Injection
- ApplicationContext, BeanFactory
- Spring Bean: khai báo, scope (singleton, prototype...), lifecycle

### 1.2. Làm quen với Spring Boot
- Spring Boot là gì, tại sao dùng Spring Boot thay vì Spring thuần
- Spring Initializr (start.spring.io) - tạo project
- Cấu trúc thư mục 1 project Spring Boot
- File `application.properties` / `application.yml`
- Annotation cơ bản: `@SpringBootApplication`, `@Component`, `@Service`, `@Repository`, `@Controller`, `@Autowired`
- Auto-configuration, Starter dependencies
- Chạy thử ứng dụng Spring Boot đầu tiên ("Hello World")

---

## GIAI ĐOẠN 2: XÂY DỰNG REST API VỚI SPRING BOOT

### 2.1. Spring MVC & REST Controller
- `@RestController`, `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`
- `@PathVariable`, `@RequestParam`, `@RequestBody`, `@ResponseBody`
- Trả về ResponseEntity, xử lý Status Code
- Xây dựng CRUD API đơn giản (chưa kết nối DB - dùng List/Map lưu tạm)

### 2.2. Xử lý dữ liệu đầu vào
- Validation với `@Valid`, Bean Validation (`@NotNull`, `@Size`, `@Email`...)
- Xử lý exception toàn cục: `@ControllerAdvice`, `@ExceptionHandler`
- Custom Response format (chuẩn hóa response trả về)

### 2.3. Cấu trúc project chuẩn
- Kiến trúc phân lớp: Controller - Service - Repository - Entity/DTO
- DTO (Data Transfer Object) và Mapping (dùng ModelMapper hoặc MapStruct)
- Package theo layer hoặc theo feature

### 2.4. Tài liệu hóa & chuẩn hóa API
- Swagger/OpenAPI với springdoc-openapi (tự sinh tài liệu API)
- API Versioning (URI versioning, Header versioning...)
- Giao tiếp HTTP ra bên ngoài: RestTemplate / WebClient (gọi API khác)
- Rate Limiting cơ bản (giới hạn số request)
- Idempotency trong thiết kế API (đặc biệt quan trọng với API thanh toán/đặt hàng)

---

## GIAI ĐOẠN 3: LÀM VIỆC VỚI CƠ SỞ DỮ LIỆU

### 3.1. Spring Data JPA
- ORM là gì, Hibernate là gì
- `@Entity`, `@Id`, `@GeneratedValue`, `@Column`, `@Table`
- Relationship mapping: `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`
- JpaRepository, CrudRepository - các method có sẵn
- Custom query: Query Method, `@Query` (JPQL/Native SQL)
- Pagination & Sorting (Pageable, Sort)

### 3.2. Kết nối Database
- Kết nối MySQL/PostgreSQL với Spring Boot
- Cấu hình DataSource
- Flyway hoặc Liquibase (quản lý migration database)
- H2 Database (dùng để test)

### 3.3. Transaction
- `@Transactional`
- Transaction propagation, isolation level cơ bản

### 3.4. Chủ đề mở rộng về dữ liệu
- Hibernate Caching: First-level cache và Second-level cache
- Kết nối nhiều nguồn dữ liệu (Multiple DataSource) trong 1 ứng dụng
- NoSQL với Spring Data: MongoDB, Redis (dùng làm database, không chỉ cache)

---

## GIAI ĐOẠN 4: BẢO MẬT VÀ XÁC THỰC

### 4.1. Spring Security cơ bản
- Kiến trúc Spring Security: Filter Chain, Authentication, Authorization
- Cấu hình SecurityFilterChain (Spring Security 6+)
- UserDetailsService, PasswordEncoder (BCrypt)
- Đăng ký/Đăng nhập cơ bản với session
- CORS (Cross-Origin Resource Sharing) - cấu hình cho phép frontend gọi API
- CSRF Protection - khi nào cần bật/tắt trong REST API

### 4.2. JWT Authentication
- JWT là gì, cấu trúc JWT
- Xây dựng luồng đăng nhập trả về JWT Token
- Filter kiểm tra JWT trong mỗi request
- Refresh Token
- Phân quyền: `@PreAuthorize`, `@Secured`, Role-based Access Control

### 4.3. OAuth2 (nâng cao hơn)
- OAuth2 là gì
- Login bằng Google/Facebook với Spring Security OAuth2 Client

---

## GIAI ĐOẠN 5: KIỂM THỬ (TESTING)

### 5.1. Unit Test
- JUnit 5 cơ bản
- Mockito: mock dependency, verify, when-thenReturn
- Test Service layer

### 5.2. Integration Test
- `@SpringBootTest`
- Test Controller với MockMvc (hoặc RestAssured)
- Testcontainers (test với DB thật trong container)
- Đo Test Coverage với JaCoCo

---

## GIAI ĐOẠN 6: CÁC CHỦ ĐỀ NÂNG CAO

### 6.1. Caching
- Spring Cache Abstraction: `@Cacheable`, `@CacheEvict`, `@CachePut`
- Tích hợp Redis làm cache

### 6.2. Xử lý bất đồng bộ & Message Queue
- `@Async` trong Spring
- Giới thiệu Message Queue: RabbitMQ, Kafka
- Tích hợp Spring Boot với RabbitMQ/Kafka (Producer - Consumer)

### 6.3. Microservices với Spring Cloud
- Kiến trúc Microservices là gì (so với Monolithic)
- Spring Cloud Netflix Eureka (Service Discovery)
- Spring Cloud Gateway (API Gateway)
- OpenFeign (giao tiếp giữa các service)
- Config Server (quản lý cấu hình tập trung)
- Circuit Breaker: Resilience4j
- Distributed Tracing: Zipkin, Sleuth/Micrometer Tracing
- Event-Driven Architecture cơ bản (giao tiếp bất đồng bộ giữa các service qua event)
- Domain-Driven Design (DDD) - khái niệm cơ bản (Bounded Context, Aggregate, Entity/Value Object)

### 6.4. Logging & Monitoring
- Logback/SLF4J - cấu hình log
- Spring Boot Actuator (health check, metrics)
- Tích hợp Prometheus + Grafana giám sát ứng dụng
- Centralized Logging với ELK Stack (Elasticsearch - Logstash - Kibana)

### 6.5. Performance & Tối ưu
- N+1 Query Problem và cách khắc phục
- Connection Pool (HikariCP)
- Tối ưu JPA/Hibernate (Lazy vs Eager Loading, Fetch Join)
- Batch Processing với Spring Batch

### 6.6. Các tính năng nâng cao khác
- Scheduling: `@Scheduled`, Quartz Scheduler
- WebSocket với Spring Boot (tính năng real-time)
- GraphQL với Spring Boot (tùy chọn, thay thế REST trong 1 số trường hợp)

---

## GIAI ĐOẠN 7: TRIỂN KHAI (DEPLOYMENT) & DEVOPS CƠ BẢN

### 7.1. Đóng gói & Triển khai
- Build file JAR/WAR
- Docker: viết Dockerfile cho Spring Boot app
- Docker Compose (app + database + redis...)

### 7.2. CI/CD cơ bản
- Giới thiệu CI/CD
- GitHub Actions hoặc Jenkins pipeline cơ bản
- Deploy lên cloud: AWS/Render/Railway (một trong số đó)

### 7.3. Môi trường (Profiles)
- Spring Profiles: dev, staging, production
- Quản lý biến môi trường/secret (không hardcode thông tin nhạy cảm)

---

## GIAI ĐOẠN 8: DỰ ÁN THỰC HÀNH (PROJECT)

Gợi ý lộ trình làm project để áp dụng kiến thức:

1. **Project 1 (Cơ bản)**: Quản lý Todo List / Quản lý sinh viên - CRUD API đơn giản, chưa có bảo mật
2. **Project 2 (Trung bình)**: Blog/Ecommerce nhỏ - có JPA, quan hệ giữa các bảng, JWT authentication, phân quyền
3. **Project 3 (Nâng cao)**: Hệ thống đặt hàng/quản lý bán hàng - có thanh toán giả lập, cache Redis, gửi email, upload file, Message Queue
4. **Project 4 (Microservices)**: Tách hệ thống trên thành nhiều service nhỏ, dùng Eureka, API Gateway, Docker Compose để chạy toàn bộ hệ thống

---

## GIAI ĐOẠN 9: CHUẨN BỊ PHỎNG VẤN

### 9.1. Ôn tập lý thuyết
- Tổng hợp câu hỏi phỏng vấn Java Core thường gặp
- Tổng hợp câu hỏi phỏng vấn Spring/Spring Boot thường gặp
- Câu hỏi tình huống về thiết kế hệ thống (system design cơ bản)

### 9.2. Thực hành
- Luyện giải bài tập coding (thuật toán, cấu trúc dữ liệu cơ bản)
- Chuẩn bị 1-2 project demo để trình bày trong phỏng vấn (nên lấy từ Giai đoạn 8)
- Chuẩn bị CV/GitHub profile thể hiện rõ kỹ năng đã học

---

## TÀI NGUYÊN HỌC TẬP GỢI Ý
- Tài liệu chính thức: spring.io/guides, spring.io/projects/spring-boot
- Đọc source code các dự án mẫu trên GitHub (tìm "spring boot microservices example")
- Luyện tập trên các bài toán thực tế thay vì chỉ học lý thuyết

---

## THỜI GIAN THAM KHẢO (nếu học full-time)
| Giai đoạn | Thời gian ước tính |
|---|---|
| Giai đoạn 0 (Java Core) | 2-4 tuần (nếu đã biết lập trình có thể rút ngắn) |
| Giai đoạn 1-2 (Nhập môn + REST API) | 2 tuần |
| Giai đoạn 3 (Database/JPA) | 2 tuần |
| Giai đoạn 4 (Security) | 2 tuần |
| Giai đoạn 5 (Testing) | 1 tuần |
| Giai đoạn 6 (Nâng cao) | 3-4 tuần |
| Giai đoạn 7 (Deploy/DevOps) | 1-2 tuần |
| Giai đoạn 8 (Project thực hành) | Song song trong suốt quá trình học |
| Giai đoạn 9 (Chuẩn bị phỏng vấn) | 1 tuần (sau khi hoàn thành ít nhất 2 project) |

*Lưu ý: Thời gian trên chỉ mang tính tham khảo, tùy vào nền tảng và thời gian học mỗi ngày của mỗi người.*
