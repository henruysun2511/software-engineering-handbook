# CHƯƠNG 6: BẢO MẬT VÀ XÁC THỰC

> Tài liệu đào tạo Java Backend Developer — dành cho người đã có nền tảng Backend (Node.js/Express/NestJS), chuyển sang hệ sinh thái Java/Spring Boot.

## Mục lục

- [Giới thiệu](#giới-thiệu)
- [6.1. Spring Security cơ bản](#61-spring-security-cơ-bản)
  - [6.1.1. Kiến trúc Spring Security: Filter Chain, Authentication, Authorization](#611-kiến-trúc-spring-security-filter-chain-authentication-authorization)
  - [6.1.2. Cấu hình `SecurityFilterChain` (Spring Security 6+)](#612-cấu-hình-securityfilterchain-spring-security-6)
  - [6.1.3. `UserDetailsService`, `PasswordEncoder`](#613-userdetailsservice-passwordencoder)
  - [6.1.4. CORS (Cross-Origin Resource Sharing)](#614-cors-cross-origin-resource-sharing)
  - [6.1.5. CSRF Protection — khi nào cần bật/tắt trong REST API](#615-csrf-protection-khi-nào-cần-bậttắt-trong-rest-api)
- [6.2. JWT Authentication](#62-jwt-authentication)
  - [6.2.1. JWT là gì, cấu trúc JWT](#621-jwt-là-gì-cấu-trúc-jwt)
  - [6.2.2. Xây dựng luồng đăng nhập trả về JWT Token](#622-xây-dựng-luồng-đăng-nhập-trả-về-jwt-token)
  - [6.2.3. Filter kiểm tra JWT trong mỗi request](#623-filter-kiểm-tra-jwt-trong-mỗi-request)
  - [6.2.4. Refresh Token](#624-refresh-token)
  - [6.2.5. Phân quyền: `@PreAuthorize`, `@Secured`, Role-based Access Control](#625-phân-quyền-preauthorize-secured-role-based-access-control)
- [6.3. OAuth2 (nâng cao hơn)](#63-oauth2-nâng-cao-hơn)
  - [6.3.1. OAuth2 là gì](#631-oauth2-là-gì)
  - [6.3.2. Login bằng Google với Spring Security OAuth2 Client](#632-login-bằng-google-với-spring-security-oauth2-client)
- [Ví dụ Code: Tổng hợp toàn bộ chương](#ví-dụ-code-tổng-hợp-toàn-bộ-chương)
- [So sánh tổng hợp Chương 6](#so-sánh-tổng-hợp-chương-6)
- [Best Practices](#best-practices)
- [Anti-patterns](#anti-patterns)
- [Bài tập](#bài-tập)
- [Tổng kết](#tổng-kết)

## Giới thiệu

Nếu bạn từng dùng Passport.js hay NestJS Guards, khái niệm "middleware chặn request trước khi tới Controller" không xa lạ. Spring Security làm điều tương tự nhưng ở quy mô lớn hơn nhiều — một **chuỗi Filter (Filter Chain)** xử lý mọi request trước khi nó chạm tới `DispatcherServlet` (thành phần điều phối request của Spring MVC, học chi tiết ở Chương Spring MVC).

Đây là 1 trong những module Spring có đường cong học tập dốc nhất — không phải vì khó, mà vì có **quá nhiều lớp trừu tượng** (Filter, AuthenticationManager, AuthenticationProvider, UserDetailsService, SecurityContext) chồng lên nhau. Chương này đi theo hướng "hiểu luồng chạy thực tế trước, nhớ tên class sau" — khi đã hiểu request đi qua đâu, annotation/class cụ thể sẽ tự nhiên dễ nhớ.

---

## 6.1. Spring Security cơ bản

### 6.1.1. Kiến trúc Spring Security: Filter Chain, Authentication, Authorization

**Khái niệm**: Spring Security hoạt động bằng cách chèn 1 chuỗi các **Servlet Filter** vào trước `DispatcherServlet` — mỗi Filter đảm nhiệm 1 nhiệm vụ cụ thể (parse JWT, kiểm tra CSRF, ghi log...), request phải đi qua **toàn bộ chuỗi** trước khi tới Controller.

```mermaid
flowchart LR
    Request["HTTP Request"] --> F1["SecurityContextPersistenceFilter"]
    F1 --> F2["Custom JWT Filter (bạn tự viết)"]
    F2 --> F3["UsernamePasswordAuthenticationFilter"]
    F3 --> F4["ExceptionTranslationFilter"]
    F4 --> F5["FilterSecurityInterceptor (kiểm tra Authorization)"]
    F5 --> Dispatcher["DispatcherServlet -> Controller"]
```

**Phân biệt 2 khái niệm cốt lõi** (thường bị nhầm lẫn ngay cả với người có kinh nghiệm):

- **Authentication** (Xác thực): trả lời câu hỏi *"Bạn là ai?"* — xác minh danh tính (username/password đúng không, JWT token hợp lệ không). Kết quả là 1 object `Authentication` chứa thông tin người dùng đã xác thực, lưu trong `SecurityContext`.
- **Authorization** (Phân quyền): trả lời câu hỏi *"Bạn được phép làm gì?"* — dựa trên danh tính đã xác thực, kiểm tra có quyền truy cập resource/thực hiện hành động cụ thể không (`ROLE_ADMIN` mới được xóa user, `ROLE_USER` chỉ được xem đơn hàng của chính mình).

**Luồng Authentication tổng quát**:

```mermaid
sequenceDiagram
    participant Client
    participant Filter as Authentication Filter
    participant Manager as AuthenticationManager
    participant Provider as AuthenticationProvider
    participant UDS as UserDetailsService
    participant Context as SecurityContext

    Client->>Filter: Request kèm credential (username/password hoặc JWT)
    Filter->>Manager: authenticate(authenticationRequest)
    Manager->>Provider: Ủy quyền xử lý xác thực thực tế
    Provider->>UDS: loadUserByUsername(username)
    UDS-->>Provider: UserDetails (từ Database)
    Provider->>Provider: So khớp password (PasswordEncoder.matches())
    Provider-->>Manager: Authentication (đã xác thực thành công)
    Manager-->>Filter: Authentication
    Filter->>Context: Lưu vào SecurityContext (dùng cho suốt vòng đời request)
    Filter->>Client: Cho phép request đi tiếp tới Controller
```

### 6.1.2. Cấu hình `SecurityFilterChain` (Spring Security 6+)

**Lưu ý quan trọng về lịch sử**: Trước Spring Security 5.7, cấu hình phải kế thừa `WebSecurityConfigurerAdapter` — cách này **đã bị deprecated và xóa hoàn toàn** từ Spring Security 6 (đi cùng Spring Boot 3.x). Cách chuẩn hiện tại là khai báo `SecurityFilterChain` như 1 `@Bean` thông thường — nếu bạn tra cứu tutorial cũ dùng `WebSecurityConfigurerAdapter`, đó là kiến thức **lỗi thời, không dùng được với Spring Boot 3.x**.

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity // bật @PreAuthorize, @PostAuthorize (học ở mục 6.2)
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http, JwtAuthenticationFilter jwtFilter) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable) // giải thích chi tiết ở mục 6.1.5
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(session ->
                    session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)) // REST API dùng JWT, không cần session
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/api/v1/auth/**", "/api/v1/public/**").permitAll()
                    .requestMatchers("/actuator/health").permitAll()
                    .requestMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
                    .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated() // mọi request khác BẮT BUỘC phải xác thực
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                    .authenticationEntryPoint((request, response, authException) ->
                            response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Chưa xác thực"))
                    .accessDeniedHandler((request, response, accessDeniedException) ->
                            response.sendError(HttpServletResponse.SC_FORBIDDEN, "Không đủ quyền"))
            );

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // strength=12, cân bằng giữa bảo mật và hiệu năng
    }
}
```

**Giải thích các thành phần quan trọng**:
- **`sessionCreationPolicy(STATELESS)`**: bắt buộc với REST API dùng JWT — server **không lưu session** giữa các request (khác hoàn toàn với ứng dụng web truyền thống dùng cookie-session). Mỗi request phải tự chứa đủ thông tin xác thực (JWT token trong header).
- **`.requestMatchers(...).permitAll()`**: whitelist các endpoint không cần xác thực (login, register, health check, tài liệu API công khai).
- **`.anyRequest().authenticated()`**: nguyên tắc **"deny by default"** — mọi endpoint không được liệt kê tường minh đều yêu cầu xác thực, đây là **Best Practice bảo mật quan trọng nhất** (secure by default, thay vì phải nhớ chặn từng endpoint mới).
- **`addFilterBefore(jwtFilter, ...)`**: chèn Custom Filter của bạn vào đúng vị trí trong chuỗi Filter Chain (chi tiết filter JWT ở mục 6.2).

### 6.1.3. `UserDetailsService`, `PasswordEncoder`

**`UserDetailsService`**: interface đơn giản chỉ có 1 method, là **cầu nối** giữa Spring Security và nguồn dữ liệu người dùng thực tế của bạn (database).

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public CustomUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("Không tìm thấy user: " + username));

        return org.springframework.security.core.userdetails.User.builder()
                .username(user.getUsername())
                .password(user.getPasswordHash()) // đã được hash sẵn trong DB, KHÔNG BAO GIỜ lưu plain text
                .authorities(user.getRoles().stream()
                        .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                        .toList())
                .disabled(!user.isActive())
                .build();
    }
}
```

**`PasswordEncoder` — BCrypt**: **Không bao giờ** lưu password dạng plain text hay chỉ hash bằng MD5/SHA (quá nhanh, dễ brute-force). BCrypt là **thuật toán hash chậm có chủ đích** (adaptive hashing), có "salt" tự động sinh ngẫu nhiên cho mỗi password — 2 user cùng password vẫn cho ra hash khác nhau hoàn toàn.

```java
@Service
public class UserRegistrationService {

    private final PasswordEncoder passwordEncoder;
    private final UserRepository userRepository;

    public User register(RegisterRequest request) {
        String hashedPassword = passwordEncoder.encode(request.password()); // BCrypt hash, KHÔNG reversible

        User user = new User(request.username(), hashedPassword);
        return userRepository.save(user);
    }

    public boolean verifyPassword(String rawPassword, String hashedPassword) {
        return passwordEncoder.matches(rawPassword, hashedPassword); // so khớp, KHÔNG "giải mã" ngược
    }
}
```

**Best Practices**:
- BCrypt strength (`work factor`) nên từ 10-12 — cao hơn thì an toàn hơn nhưng chậm hơn đáng kể, cần cân bằng với trải nghiệm người dùng (thời gian xử lý login).
- Không bao giờ tự viết thuật toán hash password — luôn dùng thư viện đã được kiểm chứng (BCrypt, Argon2).
- `UserDetailsService` chỉ nên **đọc** dữ liệu người dùng, không nên chứa logic nghiệp vụ khác — giữ đúng trách nhiệm đơn nhất.

### 6.1.4. CORS (Cross-Origin Resource Sharing)

**Khái niệm**: CORS là cơ chế **của trình duyệt** (không phải của server) — trình duyệt chặn JavaScript từ 1 origin (domain+port+protocol) gọi API tới origin khác, trừ khi server đó **tường minh cho phép** qua HTTP header phản hồi (`Access-Control-Allow-Origin`...).

**Tại sao quan trọng với REST API**: Nếu frontend chạy ở `https://app.company.com` gọi API ở `https://api.company.com`, đây là 2 origin khác nhau — trình duyệt sẽ chặn response trừ khi backend cấu hình CORS đúng.

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.company.com", "http://localhost:3000"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setAllowCredentials(true); // cho phép gửi cookie/Authorization header kèm request
        config.setMaxAge(3600L); // trình duyệt cache kết quả preflight OPTIONS trong 1h, giảm số lần gọi

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

**Best Practices**: Không bao giờ dùng `setAllowedOrigins(List.of("*"))` kết hợp `setAllowCredentials(true)` trong production — trình duyệt hiện đại thực tế **từ chối** tổ hợp này vì lý do bảo mật, và về logic, nó tương đương "cho phép MỌI website gọi API kèm credential người dùng" — rủi ro CSRF/data leak nghiêm trọng. Luôn whitelist tường minh danh sách domain được phép.

### 6.1.5. CSRF Protection — khi nào cần bật/tắt trong REST API

**Khái niệm**: CSRF (Cross-Site Request Forgery) là kiểu tấn công lợi dụng việc trình duyệt **tự động đính kèm cookie** khi gọi request, khiến 1 website độc hại có thể "mượn" phiên đăng nhập của người dùng để thực hiện hành động ngoài ý muốn (VD: 1 trang web lạ chứa form tự động submit `POST /api/transfer-money` tới ngân hàng bạn đang đăng nhập).

**Tại sao REST API dùng JWT thường TẮT CSRF protection**: CSRF chỉ thực sự là mối đe dọa khi ứng dụng dùng **cookie-based session** (trình duyệt tự động gửi cookie mà không cần JavaScript can thiệp). Với REST API dùng JWT lưu ở `localStorage`/`sessionStorage` và gửi qua header `Authorization: Bearer <token>` (không phải cookie), **request giả mạo từ site khác không thể tự động đính kèm token này** — do đó CSRF không áp dụng được, và Spring Security CSRF protection (vốn thiết kế cho mô hình cookie-session) trở nên không cần thiết, thậm chí gây phiền toái (phải quản lý CSRF token thêm).

**Khi nào VẪN CẦN bật CSRF**: Nếu ứng dụng dùng cookie để lưu JWT (`HttpOnly` cookie — cách an toàn hơn để tránh XSS đánh cắp token), hoặc bất kỳ cơ chế nào dựa vào cookie tự động gửi kèm request, **bắt buộc phải bật CSRF protection**.

**So sánh: JWT trong `localStorage` vs `HttpOnly Cookie`**

| Tiêu chí | JWT trong `localStorage` | JWT trong `HttpOnly Cookie` |
|---|---|---|
| Rủi ro XSS (script độc hại đọc token) | ❌ Cao — JavaScript đọc được `localStorage` | ✅ Thấp — `HttpOnly` cookie JavaScript không đọc được |
| Rủi ro CSRF | ✅ Thấp — token không tự động gửi kèm | ❌ Cao — cookie tự động gửi kèm mọi request |
| Cần bật CSRF Protection | ❌ Không cần | ✅ Bắt buộc |
| Độ phổ biến trong REST API hiện đại | ✅ Phổ biến hơn (đơn giản hơn cho SPA/mobile) | Dùng khi ưu tiên chống XSS hơn |

**Best Practices**: Với kiến trúc REST API + SPA (React/Vue) hoặc mobile app dùng JWT qua header `Authorization`, tắt CSRF là hợp lý và phổ biến. Nhưng **luôn phải có chiến lược chống XSS tương ứng** (sanitize input, Content-Security-Policy header) vì đây là rủi ro đánh đổi khi chọn `localStorage`.

---

## 6.2. JWT Authentication

### 6.2.1. JWT là gì, cấu trúc JWT

**Khái niệm**: JWT (JSON Web Token) là 1 chuẩn mở để truyền thông tin **tự xác thực** (self-contained) giữa các bên dưới dạng JSON, được ký số (signed) để đảm bảo tính toàn vẹn — server có thể **xác minh token hợp lệ mà không cần lưu trạng thái (stateless)**, khác hoàn toàn với session truyền thống (server phải lưu session ID trong bộ nhớ/Redis).

**Cấu trúc JWT** — 3 phần, phân cách bởi dấu `.`, mã hóa Base64:

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyMTIzIiwicm9sZXMiOlsiVVNFUiJdLCJleHAiOjE3MDk1NjAwMDB9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
└──────── Header ────────┘└────────────── Payload ──────────────┘└──────── Signature ────────┘
```

- **Header**: thuật toán ký (`alg`) và loại token (`typ`).
- **Payload**: chứa "claims" — thông tin về user (`sub` = subject/username, `roles`, `exp` = expiration time...). **Lưu ý quan trọng**: Payload chỉ **encode Base64**, KHÔNG mã hóa — bất kỳ ai cũng đọc được nội dung nếu có token, **tuyệt đối không đặt thông tin nhạy cảm** (password, số thẻ tín dụng) vào payload.
- **Signature**: chữ ký số, được server tính từ Header + Payload + secret key (HMAC) hoặc private key (RSA) — dùng để **xác minh token không bị giả mạo/chỉnh sửa**.

### 6.2.2. Xây dựng luồng đăng nhập trả về JWT Token

```java
@Service
public class JwtService {

    @Value("${jwt.secret}")
    private String secretKey; // BẮT BUỘC lấy từ biến môi trường/secret manager, KHÔNG hardcode

    @Value("${jwt.expiration-minutes:60}")
    private long expirationMinutes;

    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList());

        return Jwts.builder()
                .claims(claims)
                .subject(userDetails.getUsername())
                .issuedAt(new Date())
                .expiration(Date.from(Instant.now().plus(expirationMinutes, ChronoUnit.MINUTES)))
                .signWith(getSigningKey(), Jwts.SIG.HS256)
                .compact();
    }

    public String extractUsername(String token) {
        return extractClaims(token).getSubject();
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractClaims(token).getExpiration().before(new Date());
    }

    private Claims extractClaims(String token) {
        return Jwts.parser().verifyWith(getSigningKey()).build()
                .parseSignedClaims(token).getPayload();
    }

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(secretKey.getBytes(StandardCharsets.UTF_8));
    }
}

@RestController
@RequestMapping("/api/v1/auth")
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(@Valid @RequestBody LoginRequest request) {
        // AuthenticationManager tự động gọi UserDetailsService + PasswordEncoder để xác thực
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(request.username(), request.password()));

        UserDetails userDetails = (UserDetails) authentication.getPrincipal();
        String accessToken = jwtService.generateToken(userDetails);
        String refreshToken = jwtService.generateRefreshToken(userDetails);

        return ResponseEntity.ok(new LoginResponse(accessToken, refreshToken, "Bearer"));
    }
}
```

**Lưu ý**: `AuthenticationManager` **KHÔNG BAO GIỜ ném lỗi cho caller biết "sai username" hay "sai password"** riêng biệt — Spring Security cố ý gộp chung thành `BadCredentialsException` để tránh kẻ tấn công dò ra được username nào tồn tại trong hệ thống (chống **User Enumeration Attack**).

### 6.2.3. Filter kiểm tra JWT trong mỗi request

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                     FilterChain filterChain) throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response); // không có token -> để request đi tiếp,
            return;                                   // SecurityFilterChain sẽ tự chặn nếu endpoint cần auth
        }

        String jwt = authHeader.substring(7);
        String username = jwtService.extractUsername(jwt);

        // Chỉ xác thực nếu chưa có Authentication nào trong SecurityContext (tránh xử lý lặp lại)
        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            if (jwtService.isTokenValid(jwt, userDetails)) {
                UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

                SecurityContextHolder.getContext().setAuthentication(authToken);
                // Từ đây, mọi Controller/Service trong request này đều có thể lấy thông tin user
                // đã xác thực qua SecurityContextHolder.getContext().getAuthentication()
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

### 6.2.4. Refresh Token

**Tại sao cần**: Access Token JWT thường có thời hạn ngắn (15-60 phút) để giảm thiệt hại nếu bị đánh cắp. Nhưng bắt người dùng đăng nhập lại liên tục mỗi giờ gây trải nghiệm tệ — Refresh Token giải quyết bằng cách cấp 1 token **thời hạn dài hơn nhiều** (7-30 ngày), chỉ dùng để **xin cấp Access Token mới**, không dùng trực tiếp để gọi API nghiệp vụ.

```java
@PostMapping("/refresh")
public ResponseEntity<LoginResponse> refresh(@RequestBody RefreshRequest request) {
    String refreshToken = request.refreshToken();

    if (!jwtService.isRefreshTokenValid(refreshToken)) {
        throw new InvalidTokenException("Refresh token không hợp lệ hoặc đã hết hạn");
    }

    String username = jwtService.extractUsername(refreshToken);
    UserDetails userDetails = userDetailsService.loadUserByUsername(username);

    String newAccessToken = jwtService.generateToken(userDetails);
    return ResponseEntity.ok(new LoginResponse(newAccessToken, refreshToken, "Bearer"));
}
```

**Best Practices Refresh Token**:
- Lưu Refresh Token trong DB (hoặc Redis) kèm trạng thái `revoked` — cho phép **thu hồi** (VD: khi người dùng đăng xuất, đổi password, hoặc phát hiện token bị đánh cắp) — điều mà JWT thuần túy (stateless) không tự làm được.
- Refresh Token nên **xoay vòng** (rotate) — mỗi lần dùng để refresh, cấp Refresh Token mới và vô hiệu hóa cái cũ, giảm thiểu rủi ro nếu Refresh Token bị đánh cắp mà không bị phát hiện ngay.

### 6.2.5. Phân quyền: `@PreAuthorize`, `@Secured`, Role-based Access Control

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    // Chỉ ROLE_ADMIN mới được xem TOÀN BỘ đơn hàng trong hệ thống
    @PreAuthorize("hasRole('ADMIN')")
    @GetMapping
    public List<OrderDTO> getAllOrders() { /* ... */ }

    // Kiểm tra phức tạp hơn - user CHỈ được xem đơn hàng của CHÍNH MÌNH, trừ khi là ADMIN
    @PreAuthorize("hasRole('ADMIN') or #customerId == authentication.principal.id")
    @GetMapping("/customer/{customerId}")
    public List<OrderDTO> getCustomerOrders(@PathVariable Long customerId) { /* ... */ }

    // @Secured - cú pháp đơn giản hơn @PreAuthorize nhưng KHÔNG hỗ trợ SpEL phức tạp
    @Secured("ROLE_ADMIN")
    @DeleteMapping("/{id}")
    public void deleteOrder(@PathVariable Long id) { /* ... */ }
}
```

**So sánh: `@PreAuthorize` vs `@Secured`**

| Tiêu chí | `@PreAuthorize` | `@Secured` |
|---|---|---|
| Biểu thức điều kiện | SpEL (Spring Expression Language) — linh hoạt, so sánh tham số method, gọi method khác | Chỉ liệt kê role đơn giản, không hỗ trợ điều kiện phức tạp |
| Kiểm tra kết quả trả về (`@PostAuthorize`) | ✅ Hỗ trợ | ❌ Không |
| Độ phổ biến hiện tại | ✅ Khuyến nghị mặc định | Ít dùng hơn, giữ lại vì tương thích ngược |

**Best Practices JWT & Authorization**:
- Access Token thời hạn ngắn (15-60 phút), Refresh Token thời hạn dài hơn, lưu trạng thái để có thể thu hồi.
- Luôn dùng `@PreAuthorize` thay vì tự viết `if (user.hasRole(...))` rải rác trong code nghiệp vụ — tách biệt rõ ràng Authorization khỏi business logic.
- Secret key ký JWT phải đủ độ dài (tối thiểu 256-bit cho HS256), lấy từ biến môi trường/secret manager, xoay vòng định kỳ trong hệ thống nhạy cảm.
- Không đặt dữ liệu nhạy cảm vào JWT payload — payload chỉ encode, không mã hóa.

---

## 6.3. OAuth2 (nâng cao hơn)

### 6.3.1. OAuth2 là gì

**Khái niệm**: OAuth2 là 1 giao thức **ủy quyền (authorization)** chuẩn công nghiệp, cho phép ứng dụng của bạn truy cập tài nguyên của người dùng trên 1 dịch vụ khác (Google, Facebook, GitHub) **mà không cần biết password của họ** — thay vào đó, dịch vụ đó cấp 1 Access Token có phạm vi quyền hạn (scope) giới hạn.

**Phân biệt quan trọng**: OAuth2 bản chất là giao thức **ủy quyền**, không phải xác thực. **OpenID Connect (OIDC)** là lớp mở rộng **trên OAuth2** bổ sung khả năng xác thực danh tính chuẩn hóa (ID Token dạng JWT chứa thông tin người dùng) — đây là thứ thực sự đứng sau tính năng phổ biến "Đăng nhập bằng Google".

**Các vai trò trong OAuth2**:
- **Resource Owner**: người dùng cuối.
- **Client**: ứng dụng của bạn (Spring Boot backend).
- **Authorization Server**: Google/Facebook — nơi cấp token.
- **Resource Server**: nơi chứa dữ liệu được bảo vệ (VD: Google People API).

### 6.3.2. Login bằng Google với Spring Security OAuth2 Client

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
```

```java
@Configuration
@EnableWebSecurity
public class OAuth2SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/", "/login").permitAll()
                    .anyRequest().authenticated())
            .oauth2Login(oauth2 -> oauth2
                    .successHandler(oAuth2AuthenticationSuccessHandler()) // xử lý sau khi Google xác thực thành công
            );
        return http.build();
    }

    @Bean
    public AuthenticationSuccessHandler oAuth2AuthenticationSuccessHandler() {
        return (request, response, authentication) -> {
            OAuth2User oAuth2User = (OAuth2User) authentication.getPrincipal();
            String email = oAuth2User.getAttribute("email");
            String name = oAuth2User.getAttribute("name");

            // Tìm hoặc tạo user trong DB nội bộ dựa trên email từ Google,
            // sau đó cấp JWT của HỆ THỐNG BẠN (không phải token của Google) cho các request tiếp theo
            User user = userService.findOrCreateFromOAuth(email, name);
            String jwt = jwtService.generateToken(toUserDetails(user));

            response.sendRedirect("/oauth2/redirect?token=" + jwt);
        };
    }
}
```

**Luồng xử lý thực tế trong enterprise**: Sau khi người dùng đăng nhập thành công qua Google, ứng dụng **không dùng trực tiếp token của Google cho các API nội bộ** — thay vào đó, hệ thống tạo/tìm user tương ứng trong database nội bộ, rồi cấp **JWT riêng của hệ thống** (như đã xây dựng ở mục 6.2). Google chỉ đóng vai trò xác thực danh tính ban đầu ("đây đúng là email này"), không tham gia vào việc phân quyền nghiệp vụ nội bộ sau đó.

**Best Practices OAuth2**:
- Luôn dùng OIDC (không phải OAuth2 thuần) khi mục đích là xác thực danh tính người dùng.
- Không tin tưởng hoàn toàn thông tin từ Provider bên ngoài — luôn map sang model User nội bộ của hệ thống, kiểm soát quyền hạn nghiệp vụ độc lập với Provider.
- Redirect URI phải được đăng ký chính xác (whitelist) trên Google Console — đây là điểm kiểm soát bảo mật quan trọng để tránh Authorization Code bị đánh cắp qua redirect giả mạo.

---

## Ví dụ Code: Tổng hợp toàn bộ chương

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .cors(Customizer.withDefaults())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/api/v1/auth/**").permitAll()
                    .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated())
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

---

## So sánh tổng hợp Chương 6

| Tiêu chí | Session (cookie truyền thống) | JWT (stateless) |
|---|---|---|
| Lưu trạng thái ở server | ✅ Có (session store) | ❌ Không |
| Khả năng thu hồi ngay lập tức | ✅ Dễ (xóa session) | ❌ Khó (cần blacklist hoặc chờ hết hạn) |
| Phù hợp Microservices | ⚠️ Cần session store dùng chung (Redis) | ✅ Tự nhiên phù hợp (mỗi service tự verify) |
| CSRF Protection | ✅ Bắt buộc | ❌ Thường không cần (nếu không dùng cookie) |
| Kích thước request | Nhỏ (chỉ session ID) | Lớn hơn (toàn bộ claims trong mỗi request) |

---

## Best Practices

- Nguyên tắc "deny by default": `anyRequest().authenticated()`, chỉ whitelist tường minh những gì thực sự công khai.
- BCrypt cho password, không bao giờ tự viết thuật toán hash.
- Access Token ngắn hạn + Refresh Token dài hạn có khả năng thu hồi.
- `@PreAuthorize` cho authorization, tách biệt khỏi business logic.
- CORS whitelist tường minh domain, không dùng `*` kèm credential.
- OAuth2/OIDC chỉ dùng để xác thực danh tính ban đầu, luôn cấp JWT nội bộ riêng cho các API sau đó.

## Anti-patterns

- Lưu password dạng plain text hoặc hash bằng MD5/SHA thuần không có salt.
- Đặt dữ liệu nhạy cảm vào JWT payload.
- Dùng `WebSecurityConfigurerAdapter` (đã bị xóa từ Spring Security 6/Spring Boot 3.x).
- CORS `allowedOrigins("*")` kết hợp `allowCredentials(true)`.
- Không có cơ chế thu hồi Refresh Token khi người dùng đăng xuất hoặc đổi password.
- Tự viết logic kiểm tra role bằng `if-else` rải rác thay vì dùng `@PreAuthorize`.

## Bài tập

1. **Dễ**: Cấu hình `SecurityFilterChain` cơ bản với BCrypt, whitelist `/api/v1/auth/**`, mọi endpoint khác yêu cầu xác thực.
2. **Trung bình**: Xây dựng đầy đủ luồng đăng ký/đăng nhập trả JWT, viết `JwtAuthenticationFilter` xác thực token trong mỗi request.
3. **Trung bình**: Implement Refresh Token có khả năng thu hồi (lưu trong DB kèm trạng thái `revoked`), viết endpoint `/logout` vô hiệu hóa Refresh Token.
4. **Khó**: Thêm `@PreAuthorize` với biểu thức SpEL kiểm tra user chỉ được sửa đơn hàng của chính mình (trừ ADMIN), viết test xác nhận user khác không thể truy cập đơn hàng không thuộc về mình dù đã đăng nhập hợp lệ.

## Tổng kết

Chương này đã trang bị nền tảng bảo mật cốt lõi cho Spring Boot Backend Developer: kiến trúc Filter Chain và sự phân biệt rõ ràng giữa Authentication/Authorization; cấu hình `SecurityFilterChain` theo chuẩn Spring Security 6+ (không còn `WebSecurityConfigurerAdapter`); xây dựng luồng JWT hoàn chỉnh từ sinh token, verify token qua Custom Filter, tới Refresh Token có khả năng thu hồi; phân quyền chi tiết bằng `@PreAuthorize`; và hiểu đúng đắn mối quan hệ giữa OAuth2/OIDC với hệ thống JWT nội bộ. Với nền tảng Chương 5 (Database) và Chương 6 (Security), bạn đã có đủ kiến thức để xây dựng 1 REST API enterprise hoàn chỉnh: lưu trữ dữ liệu đúng chuẩn, và bảo vệ nó đúng cách.

