# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# CHƯƠNG 14: SECURITY TESTING CƠ BẢN

---

## 14.1 Tại sao Security Testing quan trọng?

Security testing không phải lĩnh vực riêng của chuyên gia bảo mật — **mọi Tester đều cần hiểu và thực hành** các bài kiểm tra bảo mật cơ bản. Hầu hết các lỗ hổng nghiêm trọng không phải do hacker kỳ diệu khai thác — chúng là các lỗi lập trình phổ biến mà developer và Tester đã bỏ sót.

**Hậu quả của lỗ hổng bảo mật:**
- Lộ dữ liệu cá nhân người dùng (vi phạm GDPR → phạt hàng triệu USD)
- Tài khoản bị chiếm đoạt
- Mất dữ liệu kinh doanh
- Uy tín thương hiệu bị tổn hại không thể phục hồi

---

## 14.2 OWASP Top 10

**OWASP (Open Web Application Security Project)** là tổ chức phi lợi nhuận công bố danh sách **10 lỗ hổng bảo mật phổ biến nhất** trong ứng dụng web. Đây là tài liệu tham khảo chuẩn cho mọi Tester bảo mật.

### 14.2.1 A01: Broken Access Control

**Mô tả:** Người dùng có thể thực hiện các hành động hoặc truy cập tài nguyên vượt quá quyền hạn cho phép.

**Ví dụ lỗ hổng:**
```
// URL: /api/orders/12345
// User A (id=100) đang đăng nhập
// Họ sửa URL thành /api/orders/12346 (đơn hàng của User B)
// Nếu server không kiểm tra quyền → User A đọc được order của User B!

// Lỗ hổng thứ 2: IDOR (Insecure Direct Object Reference)
GET /api/users/profile?id=1      → data của admin
GET /api/invoices/download/100   → invoice của user khác
```

**Kiểm thử:**
```
Test 1: Horizontal privilege escalation
  - Đăng nhập User A (id=101)
  - Gọi API: GET /api/orders/ORD-001 (đơn của User B, id=102)
  - Expected: 403 Forbidden hoặc 404 Not Found

Test 2: Vertical privilege escalation
  - Đăng nhập Regular User
  - Gọi Admin API: DELETE /api/admin/users/123
  - Expected: 403 Forbidden

Test 3: URL manipulation
  - Đăng nhập với user thường
  - Truy cập: /admin/dashboard
  - Expected: Redirect về login HOẶC 403

Test 4: Method override
  - Server chỉ cho phép GET /api/products/:id
  - Thử DELETE /api/products/:id với X-HTTP-Method-Override: DELETE
  - Expected: 405 Method Not Allowed
```

### 14.2.2 A02: Cryptographic Failures

**Mô tả:** Dữ liệu nhạy cảm không được mã hóa đúng cách — truyền dữ liệu qua HTTP thay vì HTTPS, lưu password dạng plaintext, dùng thuật toán mã hóa yếu.

**Kiểm thử:**
```bash
# Test 1: Kiểm tra HTTPS
curl -I http://api.example.com/api/users
# Expected: 301 Redirect to HTTPS, không phải 200

# Test 2: Kiểm tra SSL certificate
openssl s_client -connect api.example.com:443 -servername api.example.com
# Xem: protocol version (TLS 1.2+ là minimum), cipher suite, cert expiry

# Test 3: Kiểm tra HSTS header
curl -I https://api.example.com
# Expected: Strict-Transport-Security: max-age=31536000; includeSubDomains

# Test 4: Verify password không lưu plaintext
# Sau khi đăng ký user mới, query DB:
SELECT password FROM users WHERE email = 'newuser@test.com';
# Expected: $2b$10$xxx... (bcrypt hash), không phải "Test@123"
```

### 14.2.3 A03: Injection

**SQL Injection** là lỗ hổng phổ biến nhất: attacker chèn SQL code vào input để thao túng database.

**Kiểm thử SQL Injection:**
```
Tình huống: Form tìm kiếm sản phẩm

Input bình thường: "áo thun"
SQL tạo ra: SELECT * FROM products WHERE name LIKE '%áo thun%'

Input tấn công: áo thun' OR '1'='1
SQL tạo ra: SELECT * FROM products WHERE name LIKE '%áo thun' OR '1'='1%'
→ Điều kiện luôn đúng → trả về TẤT CẢ sản phẩm (bao gồm hidden/deleted)

Input nguy hiểm hơn: '; DROP TABLE products; --
SQL tạo ra: SELECT * FROM products WHERE name LIKE ''; DROP TABLE products; --'
→ XÓA toàn bộ bảng products!

Input extract data: ' UNION SELECT email, password, null FROM users --
→ Lấy email và password từ bảng users!
```

**Cách test SQL Injection:**
```python
# Danh sách payloads cơ bản để test
sql_injection_payloads = [
    "'",                          # Single quote — gây syntax error nếu dễ bị inject
    "''",
    "' OR '1'='1",
    "' OR '1'='1' --",
    "' OR '1'='1' /*",
    "1; DROP TABLE users --",
    "' UNION SELECT NULL --",
    "admin'--",
    "1' AND SLEEP(5) --",        # Time-based: server delay = dễ bị
]

import requests

BASE_URL = "https://api-staging.example.com"

def test_sql_injection_in_search():
    for payload in sql_injection_payloads:
        response = requests.get(
            f"{BASE_URL}/api/products",
            params={"search": payload},
            headers={"Authorization": f"Bearer {TEST_TOKEN}"}
        )
        # Không được trả về dữ liệu bất thường
        assert response.status_code in [200, 400], \
            f"Unexpected status {response.status_code} with payload: {payload}"
        # Không được trả về stack trace hay SQL error
        assert "syntax error" not in response.text.lower(), \
            f"SQL error exposed with payload: {payload}"
        assert "ORA-" not in response.text, \
            f"Oracle error exposed with payload: {payload}"
```

### 14.2.4 A07: Identification and Authentication Failures

**Kiểm thử Authentication:**
```
Test 1: Brute Force Protection
  - Gửi sai password 10 lần liên tiếp cho cùng tài khoản
  - Expected: Tài khoản bị lock HOẶC CAPTCHA xuất hiện

Test 2: Credential stuffing protection
  - Gửi 50 request login trong 1 phút từ 1 IP
  - Expected: Rate limiting (429 Too Many Requests)

Test 3: Password complexity
  - Đăng ký với password "123456"
  - Expected: Validation error, không cho phép

Test 4: Forgot password — token lifetime
  - Request forgot password → nhận token
  - Chờ 1 giờ (token hết hạn theo spec)
  - Dùng token để reset
  - Expected: 400 "Token expired"

Test 5: Session fixation
  - Ghi lại session token trước khi đăng nhập
  - Đăng nhập thành công
  - Kiểm tra session token có thay đổi không
  - Expected: Token mới sau login (không giữ token cũ)

Test 6: Logout
  - Đăng nhập → lấy token → đăng xuất
  - Dùng token cũ để gọi API
  - Expected: 401 Unauthorized (token đã bị invalidate)
```

### 14.2.5 JWT Testing

```python
import base64
import json
import requests

def decode_jwt_payload(token: str) -> dict:
    """Decode JWT payload mà không verify signature"""
    parts = token.split(".")
    payload_b64 = parts[1]
    # Thêm padding nếu cần
    padding = 4 - len(payload_b64) % 4
    payload_b64 += "=" * padding
    payload_bytes = base64.urlsafe_b64decode(payload_b64)
    return json.loads(payload_bytes)

# Lấy token từ đăng nhập
response = requests.post(f"{BASE_URL}/api/auth/login",
    json={"email": "buyer@test.com", "password": "Test@123"})
token = response.json()["data"]["token"]

# Decode và kiểm tra payload
payload = decode_jwt_payload(token)
print(payload)
# {"sub": "123", "role": "customer", "exp": 1705296400, "iat": 1705210000}

# Test 1: Kiểm tra exp (expiration) hợp lý
import time
current_time = int(time.time())
token_lifetime = payload["exp"] - payload["iat"]
assert token_lifetime <= 86400, f"Token lifetime quá dài: {token_lifetime}s"

# Test 2: Thử tamper token (đổi role thành admin)
# Lấy header.payload.signature
header_b64, payload_b64, signature = token.split(".")
# Decode payload, sửa role
modified_payload = payload.copy()
modified_payload["role"] = "admin"
modified_payload_b64 = base64.urlsafe_b64encode(
    json.dumps(modified_payload).encode()
).rstrip(b"=").decode()
# Tạo token giả với signature cũ
tampered_token = f"{header_b64}.{modified_payload_b64}.{signature}"

# Dùng token giả để gọi admin API
response = requests.get(
    f"{BASE_URL}/api/admin/users",
    headers={"Authorization": f"Bearer {tampered_token}"}
)
# Expected: 401 Unauthorized (signature không khớp)
assert response.status_code == 401, \
    "SECURITY BUG: Tampered JWT được chấp nhận!"

# Test 3: Algorithm confusion attack — thử với "none" algorithm
# (Server phải từ chối JWT với alg=none)
none_header = base64.urlsafe_b64encode(
    json.dumps({"alg": "none", "typ": "JWT"}).encode()
).rstrip(b"=").decode()
none_token = f"{none_header}.{payload_b64}."
response = requests.get(
    f"{BASE_URL}/api/orders/my",
    headers={"Authorization": f"Bearer {none_token}"}
)
assert response.status_code == 401, \
    "SECURITY BUG: JWT với alg=none được chấp nhận!"
```

---

## 14.3 XSS — Cross-Site Scripting

**XSS** là lỗ hổng cho phép attacker chèn và thực thi JavaScript độc hại trong trình duyệt của nạn nhân.

**3 loại XSS:**

**Reflected XSS:** Script trong URL, server reflect về response
```
URL: https://shop.com/search?q=<script>alert('XSS')</script>
Nếu server trả về: "Kết quả tìm kiếm cho: <script>alert('XSS')</script>"
→ Script được thực thi trong trình duyệt
```

**Stored XSS:** Script được lưu vào database, hiển thị cho tất cả user
```
Review sản phẩm: "Sản phẩm rất tốt! <script>document.cookie</script>"
→ Mỗi người xem review → script chạy → cookie bị gửi cho attacker
```

**DOM-based XSS:** Script trong DOM, không qua server
```javascript
// Vulnerable code:
document.getElementById("greeting").innerHTML = "Hello " + location.hash.substring(1);
// URL: page.html#<img src=x onerror=alert('XSS')>
// → Script chạy trong DOM
```

**Cách test XSS:**
```python
xss_payloads = [
    "<script>alert('XSS')</script>",
    "<img src=x onerror=alert('XSS')>",
    "javascript:alert('XSS')",
    "<svg onload=alert('XSS')>",
    "'><script>alert('XSS')</script>",
    "<iframe src=javascript:alert('XSS')>",
    "\" onmouseover=\"alert('XSS')",
]

# Test trong form review sản phẩm
for payload in xss_payloads:
    # Submit review với XSS payload
    response = requests.post(f"{BASE_URL}/api/products/P001/reviews",
        json={"rating": 5, "content": payload},
        headers={"Authorization": f"Bearer {TOKEN}"}
    )
    # Sau khi submit, GET lại review
    review_response = requests.get(f"{BASE_URL}/api/products/P001/reviews")
    reviews = review_response.json()["data"]
    
    for review in reviews:
        # Payload phải được escape, không còn dạng nguyên bản
        assert "<script>" not in review["content"], \
            f"SECURITY BUG: XSS not sanitized! Payload: {payload}"
        # Hoặc payload bị reject hoàn toàn (status != 200/201)
```

---

## 14.4 CORS và CSRF

### 14.4.1 CORS Testing

**CORS (Cross-Origin Resource Sharing)** kiểm soát domain nào được phép gọi API.

```bash
# Test CORS headers
curl -I -H "Origin: https://evil.com" https://api.example.com/api/users
# Expected:
#   Access-Control-Allow-Origin: https://example.com  (không phải *)
# Không muốn thấy:
#   Access-Control-Allow-Origin: *  ← cho phép mọi domain gọi API

# Test Preflight
curl -X OPTIONS \
  -H "Origin: https://evil.com" \
  -H "Access-Control-Request-Method: DELETE" \
  https://api.example.com/api/users/123
# Expected: 403 Forbidden (evil.com không được phép)
```

### 14.4.2 CSRF Testing

**CSRF (Cross-Site Request Forgery):** Trang web độc hại dùng session của user để gọi API theo ý attacker.

```html
<!-- Trang evil.com -->
<form action="https://bank.com/api/transfer" method="POST">
    <input type="hidden" name="to" value="attacker_account">
    <input type="hidden" name="amount" value="1000000">
</form>
<script>document.forms[0].submit();</script>

<!-- Nếu user đang đăng nhập bank.com → request được gửi với cookie hợp lệ →
     Transfer tiền xảy ra mà user không hay biết! -->
```

**Kiểm thử CSRF Protection:**
```python
# Test 1: Request không có CSRF token phải bị từ chối
response = requests.post(
    f"{BASE_URL}/api/transfer",
    json={"to": "attacker", "amount": 100000},
    headers={
        "Authorization": f"Bearer {TOKEN}",
        # Không có X-CSRF-Token header
    }
)
# Expected: 403 Forbidden

# Test 2: CSRF token sai phải bị từ chối
response = requests.post(
    f"{BASE_URL}/api/transfer",
    json={"to": "attacker", "amount": 100000},
    headers={
        "Authorization": f"Bearer {TOKEN}",
        "X-CSRF-Token": "invalid_csrf_token",
    }
)
# Expected: 403 Forbidden

# Test 3: CSRF token đúng phải được chấp nhận
# Lấy CSRF token hợp lệ từ response header hoặc cookie
csrf_token = get_csrf_token()  # từ GET request
response = requests.post(
    f"{BASE_URL}/api/transfer",
    json={"to": "friend", "amount": 100000},
    headers={
        "Authorization": f"Bearer {TOKEN}",
        "X-CSRF-Token": csrf_token,
    }
)
# Expected: 200 OK
```

---

## 14.5 Sensitive Data Exposure

```python
# Test 1: Response không lộ dữ liệu nhạy cảm
response = requests.get(f"{BASE_URL}/api/users/me",
    headers={"Authorization": f"Bearer {TOKEN}"})
user = response.json()["data"]

# Password không được có trong response
assert "password" not in user, "SECURITY: Password exposed in response"
assert "passwordHash" not in user, "SECURITY: Password hash exposed"
assert "password_hash" not in user

# Chỉ những field cần thiết
sensitive_fields = ["password", "password_hash", "secret_key", "api_key",
                    "credit_card", "cvv", "ssn", "social_security"]
for field in sensitive_fields:
    assert field not in str(user).lower(), \
        f"SECURITY: Sensitive field '{field}' found in response"

# Test 2: Error message không lộ thông tin hệ thống
response = requests.get(f"{BASE_URL}/api/products/nonexistent")
error_body = response.text.lower()

internal_info = ["stack trace", "at line", "exception", "postgresql",
                 "mysql", "mongodb", "/home/", "/var/www/", "c:\\users\\"]
for info in internal_info:
    assert info not in error_body, \
        f"SECURITY: Internal info '{info}' exposed in error response"

# Test 3: HTTP headers không lộ thông tin server
response = requests.get(f"{BASE_URL}/api/products")
headers = response.headers

# Không muốn lộ version của server/framework
assert "X-Powered-By" not in headers, \
    f"SECURITY: X-Powered-By header exposed: {headers.get('X-Powered-By')}"

# Test 4: Secure cookie attributes
response = requests.post(f"{BASE_URL}/api/auth/login",
    json={"email": "user@test.com", "password": "Test@123"})

for cookie in response.cookies:
    if "session" in cookie.name.lower() or "token" in cookie.name.lower():
        assert cookie.secure, f"SECURITY: Session cookie missing Secure flag"
        assert cookie.has_nonstandard_attr("HttpOnly"), \
            f"SECURITY: Session cookie missing HttpOnly flag"
```

---

## 14.6 Công cụ Security Testing

### 14.6.1 Burp Suite

**Burp Suite** là công cụ security testing mạnh nhất cho web application. Community Edition miễn phí.

**Các tính năng chính:**
- **Proxy:** Intercept và modify HTTP/HTTPS requests
- **Repeater:** Gửi lại và modify request thủ công
- **Intruder:** Brute force và fuzzing tự động
- **Scanner:** Tự động scan lỗ hổng (Pro version)

**Workflow cơ bản:**
```
1. Cấu hình trình duyệt dùng Burp proxy (127.0.0.1:8080)
2. Truy cập ứng dụng → tất cả request bị intercept
3. Forward request bình thường để test chức năng
4. Gửi request nghi ngờ sang Repeater để phân tích
5. Modify request trong Repeater để test security
```

### 14.6.2 OWASP ZAP

**OWASP ZAP** (Zed Attack Proxy) là công cụ miễn phí và dễ dùng hơn Burp.

```bash
# Chạy ZAP automated scan từ CLI
docker run -t owasp/zap2docker-stable zap-baseline.py \
  -t https://staging.example.com \
  -r zap-report.html

# Full scan (lâu hơn, tìm nhiều hơn)
docker run -t owasp/zap2docker-stable zap-full-scan.py \
  -t https://staging.example.com \
  -r zap-full-report.html

# Tích hợp CI/CD
- name: OWASP ZAP Security Scan
  uses: zaproxy/action-baseline@v0.9.0
  with:
    target: "https://staging.example.com"
    fail_action: true  # fail CI nếu có lỗ hổng High/Critical
```

---
