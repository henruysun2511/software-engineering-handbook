# Phần 6: Xác thực (Authentication) & Phân quyền (Authorization)

Hướng dẫn chi tiết về cơ chế bảo mật của dự án: JWT, login/register, refresh token, Google OAuth2, RBAC (`@RequirePermission`) và rate limit.

> Nên đọc `04-ARCHITECTURE_GUIDE.md` trước để nắm chung luồng request.

## 6.1. Tổng quan cơ chế bảo mật

Dự án dùng **JWT stateless** kết hợp **RBAC**:

```
Client (Frontend / Postman)
   │  gửi: POST /api/v1/auth/login { email, password }
   ▼
AuthController → AuthenticationService.login()
   │  Xác thực mật khẩu (DaoAuthenticationProvider + BCrypt)
   ▼
Tạo access token (JWT, chứa roles + permissions) + refresh token (lưu DB)
   │  trả về client
   ▼
Client gọi API khác → header:  Authorization: Bearer <accessToken>
   │
   ▼
Security Filter Chain:
   1. RateLimitFilter        → giới hạn số request/phút
   2. JwtAuthenticationFilter → đọc JWT, tạo UserPrincipal, đặt vào SecurityContext
   ▼
Controller + @RequirePermission (AOP)  → kiểm tra user có quyền không
   ▼
Trả ApiResponse / ném 401 (chưa đăng nhập) hoặc 403 (không đủ quyền)
```

## 6.2. SecurityFilterChain — `auth/config/SecurityConfig.java`

Đây là nơi khai báo toàn bộ luồng bảo mật.

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                .csrf(AbstractHttpConfigurer::disable)                      // API thuần, không dùng CSRF token
                .cors(cors -> cors.configurationSource(corsConfigurationSource()))
                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)) // không dùng session
                .authenticationProvider(authenticationProvider())
                .authorizeHttpRequests(authz -> authz
                        .requestMatchers(
                                "/api/v1/auth/**",     // login/register/refresh/logout
                                "/login/oauth2/**",    // Google OAuth2
                                "/oauth2/**",
                                "/swagger-ui/**",      // Swagger UI
                                "/v3/api-docs/**",     // OpenAPI JSON
                                "/actuator/**",
                                "/h2-console/**"
                        ).permitAll()                                     // công khai, không cần token
                        .anyRequest().authenticated()                     // MỌI endpoint còn lại cần đăng nhập
                )
                .exceptionHandling(ex -> ex
                        .authenticationEntryPoint(authenticationEntryPoint) // 401 chưa đăng nhập
                        .accessDeniedHandler(accessDeniedHandler)           // 403 thiếu quyền
                )
                .oauth2Login(oauth2 -> oauth2.successHandler(oAuth2AuthenticationSuccessHandler))
                .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
                .addFilterBefore(rateLimitFilter, JwtAuthenticationFilter.class);
        return http.build();
    }
}
```

### Giải thích từng phần

- **`csrf disable`**: API REST dùng token (không dùng cookie/session) nên không cần CSRF.
- **`STATELESS`**: không tạo session server-side — mỗi request tự xác thực qua JWT.
- **`.permitAll()`**: danh sách endpoint công khai (login, register, swagger...).
  > Ngoài ra còn cấu hình `app.security.public-endpoints` trong `application.properties` — xem mục 6.8.
- **`.anyRequest().authenticated()`**: **mọi endpoint khác bắt buộc có token hợp lệ**.
- **`authenticationProvider`**: `DaoAuthenticationProvider` dùng `CustomUserDetailsService` + `BCryptPasswordEncoder` để check mật khẩu khi login.
- **Thứ tự filter**: `RateLimitFilter` → `JwtAuthenticationFilter` → xử lý controller.

### CORS — `corsConfigurationSource()`

Cho phép frontend (localhost:3000) gọi API từ trình duyệt:

```java
configuration.setAllowedOriginPatterns(List.of("*"));
configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
configuration.setAllowedHeaders(List.of("*"));
configuration.setAllowCredentials(true);
```

## 6.3. JWT — `auth/util/JwtUtil.java`

Chịu trách nhiệm tạo và kiểm tra token. Cấu hình từ `JwtProperties` (đọc biến môi trường).

### Cấu hình — `auth/config/JwtProperties.java`

```java
@Value("${JWT_SECRET:mySecretKey...}")           // khóa ký token
private String secret;

@Value("${JWT_ACCESS_TOKEN_EXPIRATION:900000}")   // access token sống 15 phút (ms)
private long accessTokenExpiration;

@Value("${JWT_REFRESH_TOKEN_EXPIRATION:604800000}") // refresh token sống 7 ngày
private long refreshTokenExpiration;
```

### Tạo access token

```java
public String generateAccessToken(UserDetails userDetails) {
    Map<String, Object> extraClaims = new HashMap<>();
    if (userDetails instanceof UserPrincipal principal) {
        extraClaims.put("roles", principal.getRoles().stream().map(Role::getCode).toList());
        extraClaims.put("permissions", principal.getPermissions().stream().map(Permission::getCode).toList());
    }
    return buildToken(extraClaims, userDetails, jwtProperties.getAccessTokenExpiration());
}
```

**Điểm quan trọng**: roles + permissions được nhúng vào JWT → `JwtAuthenticationFilter` tái tạo `UserPrincipal` có sẵn quyền mà không cần query DB lại mỗi lần.

### Kiểm tra token

```java
private Claims extractAllClaims(String token) {
    try {
        return Jwts.parserBuilder().setSigningKey(getSignInKey()).build()
                .parseClaimsJws(token).getBody();
    } catch (ExpiredJwtException e)     { throw InvalidTokenException.expired(); }
    catch (UnsupportedJwtException e)   { throw InvalidTokenException.invalid(); }
    catch (MalformedJwtException e)     { throw InvalidTokenException.malformed(); }
    catch (SecurityException | IllegalArgumentException e) { throw InvalidTokenException.invalid(); }
}
```

- Ký bằng **HS256** với khóa từ `JWT_SECRET` (decode base64).
- Lỗi token → ném `InvalidTokenException` (→ handler trả 401 với message phù hợp).

## 6.4. JwtAuthenticationFilter — `auth/security/JwtAuthenticationFilter.java`

Filter chạy **trước mỗi request**, nhiệm vụ đọc token và đặt user vào `SecurityContext`.

```java
@Override
protected void doFilterInternal(...) {
    // 1. Lấy header Authorization
    final String authHeader = request.getHeader("Authorization");
    if (authHeader == null || !authHeader.startsWith("Bearer ")) {
        filterChain.doFilter(request, response);   // không có token → đi tiếp, sẽ bị 401
        return;
    }

    // 2. Tách JWT (bỏ tiền tố "Bearer ")
    jwt = authHeader.substring(7);

    try {
        // 3. Lấy email (subject) từ token
        userEmail = jwtUtil.extractUsername(jwt);

        // 4. Nếu hợp lệ và chưa xác thực → load user + set SecurityContext
        if (userEmail != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(userEmail);

            if (jwtUtil.isTokenValid(jwt, userDetails)) {
                UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
    } catch (Exception e) {
        log.error("Cannot set user authentication: {}", e.getMessage());
    }
    filterChain.doFilter(request, response);
}
```

### UserPrincipal — `auth/security/UserPrincipal.java`

Là `UserDetails` của project, giữ id, email, roles, permissions:

- `getAuthorities()` trả về:
  - `ROLE_<CODE>` cho từng role.
  - `<PERMISSION_CODE>` cho từng permission.
  - Fallback `ROLE_USER` nếu trống.
- `@AuthenticationPrincipal UserPrincipal userPrincipal` trong Controller để lấy user đang đăng nhập (VD `userPrincipal.getId()` để ghi `createdBy`).

### CustomUserDetailsService — `auth/security/CustomUserDetailsService.java`

Load user từ DB bằng email:

```java
@Override
public UserDetails loadUserByUsername(String email) {
    User user = userRepository.findActiveByEmailAndEnabled(email)  // chỉ lấy user còn active
            .orElseThrow(() -> UserNotFoundException.byEmail(email));
    List<Role> userRoles = authorizationService.getUserRoles(user.getId());
    List<Permission> userPermissions = authorizationService.getUserPermissions(user.getId());
    return UserPrincipal.create(user, userRoles, userPermissions);
}
```

## 6.5. Login & Register — `auth/service/AuthenticationService.java`

### Login

```java
@Transactional
public AuthenticationResponse login(LoginRequest request) {
    try {
        // 1. AuthenticationManager xác thực email + mật khẩu
        //    → dùng DaoAuthenticationProvider + CustomUserDetailsService + BCrypt
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(request.getEmail(), request.getPassword()));

        UserPrincipal userPrincipal = (UserPrincipal) authentication.getPrincipal();

        // 2. Lấy lại user + roles + permissions từ DB
        User user = userRepository.findActiveById(userPrincipal.getId()).orElseThrow(...);
        List<Role> userRoles = authorizationService.getUserRoles(user.getId());
        List<Permission> userPermissions = authorizationService.getUserPermissions(user.getId());
        UserPrincipal enhanced = UserPrincipal.create(user, userRoles, userPermissions);

        // 3. Tạo 2 token
        String accessToken = jwtUtil.generateAccessToken(enhanced);
        RefreshToken refreshToken = refreshTokenService.createRefreshToken(user);

        // 4. Trả về response chuẩn
        return authResponseMapper.toAuthenticationResponse(
                accessToken, refreshToken.getToken(),
                jwtUtil.getAccessTokenExpiration() / 1000, userWithAuth);
    } catch (BadCredentialsException e) {
        throw InvalidCredentialsException.create();   // sai mật khẩu → 401 "Invalid email or password"
    }
}
```

### Register

```java
@Transactional
public AuthenticationResponse register(RegisterRequest request) {
    if (userRepository.existsActiveByEmail(request.getEmail())) {
        throw EmailAlreadyExistsException.withEmail(request.getEmail());  // email đã tồn tại
    }
    User user = authMapper.toUser(request);
    user.setPassword(passwordEncoder.encode(request.getPassword()));   // LUÔN mã hóa mật khẩu

    User savedUser = userRepository.save(user);
    Employee employee = createEmployeeForUser(savedUser, ...);
    assignDefaultUserRole(savedUser.getId());   // tự gán role USER

    // ... tạo token, trả response (giống login)
}
```

**Nguyên tắc bảo mật quan trọng**: không bao giờ lưu mật khẩu dạng plaintext — luôn qua `passwordEncoder.encode()`.

### Endpoints — `auth/controller/AuthController.java`

| Method | Path | Chức năng | Cần token? |
| ------ | ---- | --------- | ---------- |
| POST | `/api/v1/auth/register` | Đăng ký | Không |
| POST | `/api/v1/auth/login` | Đăng nhập | Không |
| POST | `/api/v1/auth/refresh-token` | Lấy access token mới | Không |
| POST | `/api/v1/auth/logout` | Đăng xuất (revoke refresh token) | Không |
| POST | `/api/v1/auth/logout-all` | Đăng xuất mọi thiết bị | Có |

## 6.6. Refresh Token — `auth/service/RefreshTokenService.java`

- **Lưu trong DB** (bảng `refresh_tokens`) — ngược với access token (stateless).
- Mỗi lần login/register tạo refresh token mới → **revoke toàn bộ token cũ của user** (chỉ 1 phiên hoạt động).

```java
@Transactional
public RefreshToken createRefreshToken(User user) {
    refreshTokenRepository.revokeAllTokensByUser(user);
    RefreshToken refreshToken = RefreshToken.builder()
            .token(UUID.randomUUID().toString())
            .user(user)
            .expiresAt(Instant.now().plusSeconds(jwtUtil.getRefreshTokenExpiration() / 1000))
            .build();
    return refreshTokenRepository.save(refreshToken);
}
```

- Khi refresh: kiểm tra token còn hiệu lực (`verifyExpiration`) → tạo **access token mới**.
- Logout: đặt `isRevoked = true`.
- **Dọn dẹp định kỳ**: `@Scheduled(fixedRate = 86400000)` (mỗi ngày) xóa token hết hạn/đã revoke — nhờ `@EnableScheduling` ở `WorksphereApplication`.

## 6.7. Google OAuth2 Login

### Cấu hình

Trong `application.properties`:

```properties
spring.security.oauth2.client.registration.google.client-id=${GOOGLE_OAUTH2_CLIENT_ID}
spring.security.oauth2.client.registration.google.client-secret=${GOOGLE_OAUTH2_CLIENT_SECRET}
spring.security.oauth2.client.registration.google.scope=openid,profile,email
spring.security.oauth2.client.registration.google.redirect-uri=${GOOGLE_OAUTH2_REDIRECT_URI}
```

### Luồng hoạt động

1. Người dùng bấm "Login with Google" → gọi `GET /oauth2/authorization/google`.
2. Google xác thực → redirect về `{redirect-uri}`.
3. `SecurityConfig` kích hoạt `oauth2Login` → `OAuth2AuthenticationSuccessHandler`:
   - Lấy `email`, `googleId`, `given_name`, `family_name` từ `OAuth2User`.
   - Gọi `authenticationService.processGoogleOAuth2Login(...)`:
     - User chưa tồn tại → tạo mới + tạo Employee + gán role USER.
     - User tồn tại → cập nhật `googleId` nếu cần.
   - Tạo access + refresh token.
   - **Redirect về frontend** kèm token trong URL query: `{frontendUrl}/oauth/callback?access_token=...&refresh_token=...`.

> Vì dự án là API thuần (frontend riêng), OAuth2 dùng **Authorization Code flow** với redirect về frontend chứ không render trang backend.

## 6.8. RBAC — Phân quyền theo role & permission

### Vòng đời quyền

- Bảng `roles`, `permissions`, `user_roles`, `role_permissions` (module `authorization`).
- `AuthorizationService.getUserRoles/getUserPermissions` lấy quyền của user.
- Khi login/register/refresh, quyền được nạp vào `UserPrincipal` + nhúng vào JWT.
- `AuthorizationInitializerService` (nếu có) tự tạo role/permission mặc định lúc khởi động.

### Annotation `@RequirePermission` — `authorization/security/RequirePermission.java`

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface RequirePermission {
    String value();
}
```

Dùng trên endpoint:

```java
@GetMapping("/{employeeId}")
@RequirePermission(PermissionType.VIEW_EMPLOYEE)
public ResponseEntity<ApiResponse<EmployeeResponse>> getEmployeeById(...) { ... }
```

### Aspect kiểm tra quyền — `RequirePermissionAspect.java`

```java
@Aspect
@Component
@Order(0)
@ConditionalOnProperty(value = "app.security.rbac.enabled", havingValue = "true", matchIfMissing = true)
public class RequirePermissionAspect {

    @Before("@annotation(...RequirePermission)")
    public void checkPermission(JoinPoint joinPoint) {
        // 1. Lấy permission code từ annotation
        // 2. Lấy userId từ SecurityContext (UserPrincipal)
        // 3. authorizationService.hasPermission(userId, permissionKey)
        // 4. Không có quyền → ném AccessDeniedException → 403
    }
}
```

**Giải thích**:
- `@Before("@annotation(...)")` — chạy **trước** method được gọi.
- `@Order(0)` — chạy sớm nhất trong các aspect (trước aspect audit...).
- `@ConditionalOnProperty(... rbac.enabled=true)` — chỉ hoạt động khi RBAC bật (`app.security.rbac.enabled=true`).
- Không có quyền → `AccessDeniedException` → `CustomAccessDeniedHandler` trả 403 JSON.

> Các quyền dùng trong project được định nghĩa trong `PermissionType` (`shared/constant`). Khi tạo endpoint mới hãy dùng đúng quyền tương ứng (VD: `CREATE_EMPLOYEE`, `VIEW_EMPLOYEE`, `UPDATE_EMPLOYEE`, `DELETE_EMPLOYEE`).

## 6.9. Xử lý 401 / 403 — `auth/security/`

- **401 chưa đăng nhập** — `CustomAuthenticationEntryPoint`:

```java
ApiResponse.error("UNAUTHORIZED", "Bạn cần đăng nhập để truy cập tài nguyên này", null)
// status 401
```

- **403 thiếu quyền** — `CustomAccessDeniedHandler`:

```java
ApiResponse.error("FORBIDDEN", "Bạn không có quyền truy cập tài nguyên này", null)
// status 403
```

Cả hai đều trả JSON chuẩn `ApiResponse` (không phải trang HTML mặc định của Spring Security).

## 6.10. Rate Limit — `shared/ratelimit/RateLimitFilter.java`

Chạy **trước** `JwtAuthenticationFilter`, giới hạn số request/phút.

- **Xác định client**: 
  - đã đăng nhập → key `user:<email>`;
  - ẩn danh → key `ip:<địa chỉ IP>` (đọc `X-Forwarded-For`, `X-Real-IP`).
- **Loại limit** theo path:
  - `/auth/login` → `LOGIN`
  - `/auth/register` → `REGISTER`
  - `/auth/refresh` → `REFRESH_TOKEN`
  - đã đăng nhập → `AUTHENTICATED`
  - còn lại → `ANONYMOUS`
- Vượt limit → 429 kèm `Retry-After`; vượt nhiều lần → bị **ban tạm thời** (`RateLimitBannedException`).
- Thêm header `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.
- Không áp dụng cho: static, swagger, `/actuator/health`.
- Bật/tắt bằng `app.rate-limit.enabled`; cấu hình số request/phút theo từng loại trong `.env`.

## 6.11. Checklist khi bảo vệ một endpoint mới

Khi bạn thêm 1 API mới:

1. **Mặc định đã an toàn**: không cần làm gì — `.anyRequest().authenticated()` tự chặn nếu chưa đăng nhập.
2. **Phân quyền**: thêm `@RequirePermission(PermissionType.XXX)` lên method (nếu endpoint yêu cầu quyền cụ thể).
3. **Lấy user hiện tại**: khai báo tham số `@AuthenticationPrincipal UserPrincipal userPrincipal` để dùng `getId()`.
4. **Chỉ mở công khai khi thực sự cần**: thêm path vào `app.security.public-endpoints` trong `application.properties`.
5. **Kiểm tra**: gọi API khi:
   - không gửi token → **401**;
   - gửi token nhưng không đủ quyền → **403**;
   - token hết hạn → **401 "Token has expired"**;
   - đúng quyền → trả data.

## 6.12. Các lỗi thường gặp

| Triệu chứng | Nguyên nhân | Cách xử lý |
| ----------- | ----------- | ---------- |
| 401 `"Token has expired"` | Access token quá 15 phút | Gọi `/api/v1/auth/refresh-token` với refresh token |
| 403 kèm `"Insufficient permissions: XXX"` | User không có permission `XXX` | Gán role chứa permission cho user |
| 401 `"Invalid email or password"` | Sai email/mật khẩu | Kiểm tra lại thông tin |
| Login OAuth2 redirect lỗi | Thiếu cấu hình Google / redirect-uri sai | Kiểm tra `GOOGLE_OAUTH2_CLIENT_ID/SECRET/REDIRECT_URI` và `FRONTEND_URL` |
| Mọi request đều 403 | RBAC bật nhưng user chưa có role/permission | Gán role cho user (role phải có permission) |
| Mọi request đều 429 | Vượt rate limit | Chờ 1 phút hoặc bật `app.rate-limit.enabled=false` lúc dev |
