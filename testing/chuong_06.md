# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# CHƯƠNG 6: API TESTING

---

## 6.1 API là gì?

### 6.1.1 Khái niệm và vai trò

**API (Application Programming Interface)** là tập hợp các quy tắc và giao thức cho phép các ứng dụng phần mềm giao tiếp với nhau. API đóng vai trò như một "hợp đồng" — định nghĩa rõ ràng: ai có thể gọi, gọi theo cách nào, và nhận lại gì.

Trong kiến trúc phần mềm hiện đại, hầu hết ứng dụng web và mobile hoạt động theo mô hình:

```
[Frontend / Mobile App]
          ↕  HTTP/HTTPS
      [REST API]
          ↕
   [Business Logic]
          ↕
     [Database]
```

**Tại sao Tester cần kiểm thử API?**

Kiểm thử chỉ qua UI có nhiều hạn chế:
- UI che giấu nhiều logic backend — bug trong API có thể không biểu hiện ra UI ngay
- UI thay đổi thường xuyên làm test case nhanh lỗi thời
- Không thể kiểm thử các trường hợp mà UI không cho phép nhập (ví dụ: giá trị âm, chuỗi quá dài)
- Không thể kiểm tra chính xác response data, status code, headers

API testing tiếp cận trực tiếp tầng logic — phát hiện lỗi sớm hơn, nhanh hơn, và toàn diện hơn.

---

### 6.1.2 REST API

**REST (Representational State Transfer)** là kiến trúc thiết kế API phổ biến nhất hiện nay, dựa trên giao thức HTTP.

**6 nguyên tắc của REST:**

1. **Client-Server:** Client và Server tách biệt, giao tiếp qua interface chuẩn
2. **Stateless:** Mỗi request phải chứa đủ thông tin để server xử lý, server không lưu trạng thái của client
3. **Cacheable:** Response có thể được cache để tăng hiệu suất
4. **Uniform Interface:** Interface thống nhất, dùng HTTP methods và resource-based URLs
5. **Layered System:** Client không biết mình đang nói chuyện trực tiếp với server hay qua proxy
6. **Code on Demand (optional):** Server có thể gửi code thực thi về client

**Resource và URL:**
Trong REST, mọi thứ đều là **resource** (tài nguyên), được định danh bằng URL:

```
/users              → tập hợp tất cả user
/users/123          → user cụ thể có ID = 123
/users/123/orders   → tất cả đơn hàng của user 123
/users/123/orders/456 → đơn hàng 456 của user 123
```

**Quy ước đặt tên URL chuẩn REST:**
```
✅ /products              (danh từ số nhiều)
✅ /products/42           (resource cụ thể)
✅ /products/42/reviews   (sub-resource)
❌ /getProducts           (không dùng động từ trong URL)
❌ /product               (không dùng số ít)
❌ /Products              (không dùng chữ hoa)
```

---

## 6.2 HTTP Methods — Phương thức HTTP

HTTP Methods xác định **loại hành động** muốn thực hiện trên resource.

### 6.2.1 Các Method và ý nghĩa

| Method | Mục đích | Idempotent? | Safe? | Request Body? |
|---|---|---|---|---|
| **GET** | Lấy dữ liệu | ✅ | ✅ | Không |
| **POST** | Tạo mới | ❌ | ❌ | Có |
| **PUT** | Thay thế hoàn toàn | ✅ | ❌ | Có |
| **PATCH** | Cập nhật một phần | ❌* | ❌ | Có |
| **DELETE** | Xóa | ✅ | ❌ | Thường không |

**Giải thích:**
- **Idempotent:** Gọi nhiều lần cho kết quả giống gọi 1 lần. `DELETE /users/123` gọi 10 lần: lần đầu xóa, lần 2-10 báo 404 — kết quả cuối cùng giống nhau (user không còn tồn tại).
- **Safe:** Không thay đổi dữ liệu trên server. GET chỉ đọc, không ghi.

**Phân biệt PUT vs PATCH:**

```json
// Dữ liệu hiện tại
{
  "id": 123,
  "name": "Nguyễn Văn A",
  "email": "a@test.com",
  "phone": "0901234567"
}

// PUT /users/123 — thay thế toàn bộ, phải gửi đủ fields
{
  "name": "Nguyễn Văn B",
  "email": "a@test.com",
  "phone": "0901234567"
}
// Kết quả: name thay đổi, các field khác giữ nguyên VÌ đã gửi đủ

// PUT /users/123 — nếu chỉ gửi name
{ "name": "Nguyễn Văn B" }
// Kết quả: email và phone bị NULL/xóa — nguy hiểm!

// PATCH /users/123 — chỉ cập nhật field được gửi
{ "name": "Nguyễn Văn B" }
// Kết quả: name thay đổi, email và phone KHÔNG bị ảnh hưởng
```

---

## 6.3 HTTP Status Codes — Mã trạng thái HTTP

Status code là con số 3 chữ số server trả về, cho biết kết quả của request.

### 6.3.1 Các nhóm Status Code

**1xx — Informational (Thông tin):**
Ít gặp trong API testing. Server đang xử lý.

**2xx — Success (Thành công):**

| Code | Tên | Ý nghĩa | Dùng khi |
|---|---|---|---|
| 200 | OK | Thành công | GET, PUT, PATCH thành công |
| 201 | Created | Tạo mới thành công | POST tạo resource mới |
| 204 | No Content | Thành công, không có dữ liệu trả về | DELETE thành công |

**3xx — Redirection (Chuyển hướng):**

| Code | Tên | Ý nghĩa |
|---|---|---|
| 301 | Moved Permanently | URL đã chuyển vĩnh viễn |
| 302 | Found | Chuyển hướng tạm thời |
| 304 | Not Modified | Cache còn hợp lệ, không cần gửi lại data |

**4xx — Client Error (Lỗi từ phía client):**

| Code | Tên | Ý nghĩa | Ví dụ |
|---|---|---|---|
| 400 | Bad Request | Request sai định dạng hoặc thiếu field | JSON sai cú pháp, thiếu required field |
| 401 | Unauthorized | Chưa xác thực | Không có token, token hết hạn |
| 403 | Forbidden | Đã xác thực nhưng không có quyền | User thường truy cập API admin |
| 404 | Not Found | Resource không tồn tại | GET /users/99999 khi user không tồn tại |
| 409 | Conflict | Xung đột dữ liệu | Đăng ký email đã tồn tại |
| 422 | Unprocessable Entity | Dữ liệu đúng format nhưng sai về nghĩa | Ngày kết thúc trước ngày bắt đầu |
| 429 | Too Many Requests | Vượt rate limit | Gọi API quá nhiều lần/phút |

**5xx — Server Error (Lỗi từ phía server):**

| Code | Tên | Ý nghĩa |
|---|---|---|
| 500 | Internal Server Error | Lỗi không xác định phía server |
| 502 | Bad Gateway | Server upstream trả lỗi |
| 503 | Service Unavailable | Server đang bảo trì hoặc quá tải |
| 504 | Gateway Timeout | Server upstream timeout |

**Phân biệt 401 vs 403:**
```
401 Unauthorized: "Tôi không biết bạn là ai"
→ Chưa đăng nhập hoặc token không hợp lệ

403 Forbidden: "Tôi biết bạn là ai, nhưng bạn không được phép làm điều này"
→ Đã đăng nhập nhưng không đủ quyền
```

---

## 6.4 Cấu trúc Request và Response

### 6.4.1 Cấu trúc HTTP Request

```
POST /api/v1/auth/login HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer eyJhbGci...
Accept: application/json
X-Request-ID: abc-123-def

{
  "email": "user@example.com",
  "password": "SecurePass@123"
}
```

**Các thành phần:**

**Request Line:** Method + URL + HTTP version
```
POST /api/v1/auth/login HTTP/1.1
```

**Headers:** Metadata về request

| Header | Mô tả | Ví dụ |
|---|---|---|
| `Content-Type` | Định dạng body gửi lên | `application/json` |
| `Authorization` | Token xác thực | `Bearer eyJhbGci...` |
| `Accept` | Định dạng response mong muốn | `application/json` |
| `X-Request-ID` | ID để trace request | `abc-123-def` |
| `Cookie` | Session cookie | `session_id=xyz` |

**URL Parameters:**

*Path Parameters* — phần của URL:
```
GET /api/users/123/orders/456
                ↑           ↑
            userId=123   orderId=456
```

*Query Parameters* — sau dấu `?`:
```
GET /api/products?category=shoes&minPrice=100&maxPrice=500&sort=price&order=asc&page=2&limit=20
```

**Request Body:** Dữ liệu gửi lên (thường JSON):
```json
{
  "email": "user@example.com",
  "password": "SecurePass@123"
}
```

---

### 6.4.2 Cấu trúc HTTP Response

```
HTTP/1.1 200 OK
Content-Type: application/json
X-Request-ID: abc-123-def
Cache-Control: no-store
Set-Cookie: session_id=xyz; HttpOnly; Secure

{
  "success": true,
  "data": {
    "token": "eyJhbGci...",
    "user": {
      "id": 123,
      "email": "user@example.com",
      "name": "Nguyễn Văn A",
      "role": "customer"
    }
  },
  "meta": {
    "timestamp": "2025-01-15T10:30:00Z"
  }
}
```

---

## 6.5 Authentication — Xác thực

### 6.5.1 Các phương thức Authentication

**Basic Authentication:**
Gửi username:password dưới dạng Base64 trong header. Ít dùng trong API hiện đại vì không an toàn (dễ decode).
```
Authorization: Basic dXNlcjpwYXNzd29yZA==
```

**API Key:**
Gửi key cố định trong header hoặc query param. Thường dùng cho server-to-server.
```
X-API-Key: sk_live_abc123def456
// hoặc
GET /api/data?api_key=sk_live_abc123def456
```

**Bearer Token (JWT):**
Phổ biến nhất. Đăng nhập → nhận token → gửi token trong mọi request sau.
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**JWT (JSON Web Token)** có 3 phần phân cách bởi dấu `.`:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9  ← Header (Base64)
.eyJzdWIiOiIxMjMiLCJyb2xlIjoiY3VzdG9tZXIiLCJleHAiOjE3MDUyOTY0MDB9  ← Payload (Base64)
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  ← Signature
```

Decode payload → đọc được thông tin user (không cần key):
```json
{
  "sub": "123",
  "role": "customer",
  "exp": 1705296400
}
```

**OAuth 2.0:**
Framework ủy quyền phức tạp hơn. Dùng khi "Đăng nhập bằng Google/Facebook". Flow cơ bản:
```
Client → Authorization Server (Google) → Authorization Code
Client + Code → Token Endpoint → Access Token
Client + Access Token → Resource Server → Data
```

---

## 6.6 Các Loại Kiểm Thử API

### 6.6.1 Positive Testing — Kiểm thử trường hợp hợp lệ

Kiểm thử với dữ liệu hợp lệ, đảm bảo API hoạt động đúng trong điều kiện bình thường.

**Ví dụ: Kiểm thử API tạo đơn hàng**

```
POST /api/v1/orders
Authorization: Bearer {valid_token}
Content-Type: application/json

{
  "items": [
    {"productId": "P001", "quantity": 2},
    {"productId": "P003", "quantity": 1}
  ],
  "shippingAddress": {
    "street": "123 Nguyễn Huệ",
    "district": "Quận 1",
    "city": "TP.HCM"
  },
  "paymentMethod": "cod"
}
```

**Kiểm tra trong Positive Testing:**
- Status code: `201 Created`
- Response có `orderId` không?
- Response có `status: "pending"` không?
- `total` trong response có khớp với giá sản phẩm × số lượng không?
- Header `Location` có chứa URL đến order mới không? (`/api/v1/orders/ORD-2025-001`)
- DB: order được tạo đúng?
- DB: tồn kho giảm đúng?
- Email xác nhận được gửi?

---

### 6.6.2 Negative Testing — Kiểm thử trường hợp không hợp lệ

Kiểm thử với dữ liệu sai, thiếu, hoặc không hợp lệ — đảm bảo API xử lý lỗi đúng cách và không crash.

**Bộ Negative Test cho API tạo đơn hàng:**

**Test 1: Không có Authorization token**
```
POST /api/v1/orders
(không có Authorization header)
→ Expected: 401 Unauthorized
→ Body: {"error": "Authentication required"}
```

**Test 2: Token hết hạn**
```
POST /api/v1/orders
Authorization: Bearer {expired_token}
→ Expected: 401 Unauthorized
→ Body: {"error": "Token expired"}
```

**Test 3: Thiếu required field (items)**
```json
{
  "shippingAddress": {"street": "123 Nguyễn Huệ", ...},
  "paymentMethod": "cod"
}
→ Expected: 400 Bad Request
→ Body: {"error": "items is required"}
```

**Test 4: productId không tồn tại**
```json
{
  "items": [{"productId": "INVALID_ID", "quantity": 1}],
  ...
}
→ Expected: 404 Not Found hoặc 422 Unprocessable Entity
→ Body: {"error": "Product INVALID_ID not found"}
```

**Test 5: Số lượng âm**
```json
{
  "items": [{"productId": "P001", "quantity": -5}],
  ...
}
→ Expected: 400 Bad Request hoặc 422
→ Body: {"error": "quantity must be greater than 0"}
```

**Test 6: Số lượng vượt tồn kho**
```json
{
  "items": [{"productId": "P001", "quantity": 9999}],
  ...
}
// Tồn kho P001 chỉ còn 10
→ Expected: 409 Conflict hoặc 422
→ Body: {"error": "Insufficient stock", "available": 10}
```

**Test 7: JSON sai cú pháp**
```
Body: {items: [{productId: P001}]}  ← thiếu dấu ngoặc kép
→ Expected: 400 Bad Request
→ Body: {"error": "Invalid JSON format"}
```

---

### 6.6.3 Validation Testing

Kiểm thử sâu hơn về validation từng field — áp dụng kỹ thuật EP và BVA từ Chương 2 vào API.

**Ví dụ: API đăng ký tài khoản**

```
POST /api/v1/auth/register
{
  "email": "...",
  "password": "...",
  "name": "..."
}
```

**Validation Matrix:**

| Field | Rule | Test case | Input | Expected |
|---|---|---|---|---|
| email | Required | Thiếu field | `{}` | 400 |
| email | Format | Không có @ | `abc.com` | 422 |
| email | Format | Không có domain | `abc@` | 422 |
| email | Unique | Đã tồn tại | `existing@test.com` | 409 |
| email | Max length | 256 ký tự | `a×250@b.com` | 422 |
| password | Required | Thiếu field | `{}` | 400 |
| password | Min length | 7 ký tự | `Abc@123` | 422 |
| password | Min length | 8 ký tự (ranh giới) | `Abc@1234` | 201 |
| password | Uppercase | Không có chữ hoa | `abc@1234` | 422 |
| password | Number | Không có số | `Abcd@xyz` | 422 |
| password | Special char | Không có ký tự đặc biệt | `Abcd1234` | 422 |
| name | Required | Thiếu field | `{}` | 400 |
| name | Min length | 1 ký tự | `A` | 201 |
| name | Max length | 101 ký tự | `A×101` | 422 |
| name | Special chars | Emoji trong tên | `Nguyễn 😀` | ? (cần confirm) |

---

### 6.6.4 Authentication và Authorization Testing

**Authentication Testing:**

```
// Test 1: Không có token
GET /api/v1/orders/my
→ 401

// Test 2: Token format sai
Authorization: Bearer not_a_valid_token
→ 401

// Test 3: Token hết hạn
Authorization: Bearer {expired_jwt}
→ 401 với message "Token expired"

// Test 4: Token bị revoke (logout rồi dùng lại)
POST /api/v1/auth/logout → blacklist token
GET /api/v1/orders/my với token vừa logout
→ 401

// Test 5: Token của user khác (không phải tấn công, chỉ verify logic)
Authorization: Bearer {token_of_user_A}
GET /api/v1/orders/my
→ 200, chỉ thấy order của user A
```

**Authorization Testing (RBAC — Role-based Access Control):**

```
Giả sử hệ thống có 3 roles: customer, manager, admin

// Test: Customer thử truy cập API admin
Authorization: Bearer {customer_token}
GET /api/v1/admin/users
→ 403 Forbidden

// Test: Manager thử xóa user (chỉ admin mới xóa được)
Authorization: Bearer {manager_token}
DELETE /api/v1/users/123
→ 403 Forbidden

// Test: User A thử xem order của User B
Authorization: Bearer {user_A_token}
GET /api/v1/orders/ORD-B-001  ← đơn hàng của User B
→ 403 hoặc 404 (không để lộ sự tồn tại của resource)
```

---

### 6.6.5 Pagination, Filtering, Sorting

API danh sách thường hỗ trợ phân trang, lọc, và sắp xếp. Đây là nhóm cần kiểm thử kỹ.

**Kiểm thử Pagination:**

```
// Test 1: Page đầu tiên
GET /api/v1/products?page=1&limit=10
→ 200, data.length = 10
→ meta.total = tổng số sản phẩm
→ meta.currentPage = 1
→ links.next có URL không?

// Test 2: Page cuối cùng
GET /api/v1/products?page=5&limit=10  (nếu có 50 sản phẩm)
→ 200, data.length = 10
→ links.next = null hoặc không có

// Test 3: Page vượt quá số trang
GET /api/v1/products?page=999&limit=10
→ 200, data = [] hoặc 404

// Test 4: Limit = 0
GET /api/v1/products?page=1&limit=0
→ 400 hoặc dùng default limit

// Test 5: Limit rất lớn
GET /api/v1/products?page=1&limit=10000
→ 400 (nếu có max limit) hoặc 200 với max_limit items
```

**Kiểm thử Filtering:**

```
// Lọc theo category
GET /api/v1/products?category=shoes
→ Tất cả sản phẩm phải có category = "shoes"

// Lọc theo price range
GET /api/v1/products?minPrice=100000&maxPrice=500000
→ Tất cả sản phẩm có price trong khoảng [100000, 500000]

// Lọc kết hợp
GET /api/v1/products?category=shoes&minPrice=100000&inStock=true
→ Sản phẩm thỏa mãn TẤT CẢ điều kiện

// Edge case: minPrice > maxPrice
GET /api/v1/products?minPrice=500000&maxPrice=100000
→ 400 hoặc data = []

// Filter không hợp lệ
GET /api/v1/products?category=nonexistent_category
→ 200, data = [] (category không có sản phẩm nào)
```

**Kiểm thử Sorting:**

```
// Sắp xếp theo giá tăng dần
GET /api/v1/products?sort=price&order=asc
→ Verify: products[0].price ≤ products[1].price ≤ ... ≤ products[n].price

// Sắp xếp theo tên
GET /api/v1/products?sort=name&order=asc
→ Verify: thứ tự alphabetical

// Sort field không tồn tại
GET /api/v1/products?sort=invalid_field
→ 400 hoặc dùng default sort
```

---

### 6.6.6 Error Handling

Kiểm thử API xử lý lỗi đúng cách, nhất quán, và có thông tin hữu ích.

**Chuẩn Error Response:**

```json
// ✅ Error response tốt
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Dữ liệu không hợp lệ",
    "details": [
      {
        "field": "email",
        "message": "Email không đúng định dạng"
      },
      {
        "field": "password",
        "message": "Password phải có ít nhất 8 ký tự"
      }
    ]
  },
  "requestId": "abc-123"
}

// ❌ Error response tệ
{
  "message": "error"
}
```

**Các điểm cần kiểm tra:**
- Error message rõ ràng, đủ thông tin để client xử lý
- Không lộ thông tin nhạy cảm (stack trace, tên bảng DB, đường dẫn file)
- Error code nhất quán (cùng lỗi, cùng code)
- HTTP status code đúng với loại lỗi

---

### 6.6.7 Rate Limiting

**Rate Limiting** giới hạn số lượng request trong một khoảng thời gian, bảo vệ API khỏi abuse.

**Kiểm thử:**

```
// Kiểm tra headers rate limit
GET /api/v1/products
→ Headers:
  X-RateLimit-Limit: 100       ← tổng số request cho phép
  X-RateLimit-Remaining: 99   ← còn lại
  X-RateLimit-Reset: 1705296400 ← thời điểm reset (Unix timestamp)

// Test vượt rate limit
Gửi 101 request trong 1 phút:
→ Request thứ 101: 429 Too Many Requests
→ Body: {"error": "Rate limit exceeded", "retryAfter": 45}
→ Header: Retry-After: 45

// Test sau khi reset
Chờ hết thời gian reset → request tiếp theo phải thành công
```

---

### 6.6.8 File Upload Testing

```
// Test upload ảnh sản phẩm
POST /api/v1/products/123/images
Content-Type: multipart/form-data
Authorization: Bearer {token}

file: [binary image data]

// Positive: ảnh JPEG hợp lệ
→ 201, trả về URL của ảnh đã upload

// Negative: file quá lớn (>5MB)
→ 413 Payload Too Large

// Negative: format không hỗ trợ (PDF, EXE)
→ 415 Unsupported Media Type

// Negative: file rỗng (0 bytes)
→ 400 Bad Request

// Negative: không có file trong request
→ 400 Bad Request, {"error": "file is required"}

// Edge: tên file chứa ký tự đặc biệt
file: "ảnh sản phẩm (1).jpg"
→ Server xử lý tên file đúng, không crash
```

---

## 6.7 Postman — Công cụ API Testing

### 6.7.1 Tổng quan và cài đặt

**Postman** là công cụ GUI phổ biến nhất để thiết kế, kiểm thử, và tài liệu hóa API. Tải tại: https://www.postman.com/downloads/

**Cấu trúc tổ chức trong Postman:**
```
Workspace
└── Collection (nhóm API theo project/module)
    ├── Folder (nhóm theo feature)
    │   ├── Request 1
    │   ├── Request 2
    │   └── ...
    └── Folder 2
        └── ...
```

---

### 6.7.2 Environment và Variables

**Tại sao cần Environment?**
Cùng một API nhưng URL, token, và dữ liệu khác nhau ở các môi trường (dev, staging, production). Thay vì sửa tay từng request, dùng Environment Variables.

**Tạo Environment trong Postman:**

```
Environment: "Staging"
Variables:
  base_url    = https://api-staging.example.com
  token       = (để trống, sẽ set sau khi login)
  admin_token = eyJhbGci...
  test_user_id = 123

Environment: "Production"
Variables:
  base_url    = https://api.example.com
  token       = (để trống)
  admin_token = eyJhbGci...
  test_user_id = 456
```

**Dùng Variables trong Request:**
```
URL: {{base_url}}/api/v1/users/{{test_user_id}}
Header: Authorization: Bearer {{token}}
```

**Các loại Variables (theo scope, ưu tiên từ cao đến thấp):**
```
Global Variables    → toàn bộ Postman
Collection Variables → toàn bộ Collection
Environment Variables → Environment đang chọn
Local Variables      → chỉ trong 1 request (set qua script)
```

---

### 6.7.3 Pre-request Script

Pre-request Script chạy **trước** khi request được gửi. Dùng để:
- Generate dữ liệu dynamic (timestamp, random ID)
- Set variable từ calculation
- Lấy token tự động trước khi gọi API cần auth

**Ví dụ 1: Generate unique email cho mỗi lần test:**
```javascript
// Pre-request Script của request "Đăng ký tài khoản"
const timestamp = Date.now();
const randomEmail = `test_${timestamp}@example.com`;
pm.environment.set("test_email", randomEmail);
pm.environment.set("test_name", "Test User " + timestamp);
```

**Ví dụ 2: Auto-login và lưu token:**
```javascript
// Pre-request Script — tự động lấy token nếu chưa có
const token = pm.environment.get("token");
if (!token) {
    pm.sendRequest({
        url: pm.environment.get("base_url") + "/api/v1/auth/login",
        method: "POST",
        header: { "Content-Type": "application/json" },
        body: {
            mode: "raw",
            raw: JSON.stringify({
                email: "buyer@test.com",
                password: "Test@123"
            })
        }
    }, function(err, response) {
        if (!err && response.code === 200) {
            const token = response.json().data.token;
            pm.environment.set("token", token);
        }
    });
}
```

---

### 6.7.4 Test Script (Assertions)

Test Script chạy **sau** khi nhận response. Đây là nơi viết assertion để tự động verify kết quả.

**Cú pháp cơ bản:**
```javascript
// Kiểm tra status code
pm.test("Status code là 200", function() {
    pm.response.to.have.status(200);
});

// Kiểm tra response time
pm.test("Response time dưới 500ms", function() {
    pm.expect(pm.response.responseTime).to.be.below(500);
});

// Kiểm tra header
pm.test("Content-Type là JSON", function() {
    pm.response.to.have.header("Content-Type", "application/json; charset=utf-8");
});

// Parse JSON và kiểm tra giá trị
pm.test("Response có cấu trúc đúng", function() {
    const body = pm.response.json();
    pm.expect(body.success).to.be.true;
    pm.expect(body.data).to.have.property("token");
    pm.expect(body.data.token).to.be.a("string");
    pm.expect(body.data.token).to.not.be.empty;
});

// Lưu giá trị vào variable để dùng trong request tiếp theo
pm.test("Lưu token vào environment", function() {
    const body = pm.response.json();
    pm.environment.set("token", body.data.token);
    pm.environment.set("user_id", body.data.user.id);
});
```

**Ví dụ Test Script đầy đủ cho API đăng nhập:**

```javascript
// Test Script cho POST /api/v1/auth/login

pm.test("TC_LOGIN_001: Đăng nhập thành công - Status 200", function() {
    pm.response.to.have.status(200);
});

pm.test("TC_LOGIN_001: Response time < 1000ms", function() {
    pm.expect(pm.response.responseTime).to.be.below(1000);
});

pm.test("TC_LOGIN_001: Response body có cấu trúc đúng", function() {
    const body = pm.response.json();
    
    // Kiểm tra top-level
    pm.expect(body).to.have.property("success", true);
    pm.expect(body).to.have.property("data");
    
    // Kiểm tra data
    const data = body.data;
    pm.expect(data).to.have.property("token");
    pm.expect(data).to.have.property("user");
    
    // Kiểm tra user object
    const user = data.user;
    pm.expect(user).to.have.property("id");
    pm.expect(user).to.have.property("email", "user@example.com");
    pm.expect(user).to.have.property("role", "customer");
    
    // Đảm bảo không có field nhạy cảm
    pm.expect(user).to.not.have.property("password");
    pm.expect(user).to.not.have.property("passwordHash");
});

pm.test("TC_LOGIN_001: Token hợp lệ (JWT format)", function() {
    const body = pm.response.json();
    const token = body.data.token;
    const parts = token.split(".");
    pm.expect(parts).to.have.lengthOf(3);
});

// Lưu token để dùng ở request tiếp theo
if (pm.response.code === 200) {
    const body = pm.response.json();
    pm.environment.set("token", body.data.token);
    pm.environment.set("current_user_id", body.data.user.id);
}
```

---

### 6.7.5 Newman — Chạy Postman trên CLI

**Newman** là công cụ CLI cho phép chạy Postman Collection từ terminal, tích hợp vào CI/CD pipeline.

**Cài đặt:**
```bash
npm install -g newman
npm install -g newman-reporter-htmlextra  # Reporter đẹp hơn
```

**Export Collection từ Postman:**
Collection → ... → Export → Collection v2.1 → `my-api-tests.json`
Environment → ... → Export → `staging.json`

**Chạy cơ bản:**
```bash
# Chạy với environment
newman run my-api-tests.json \
  --environment staging.json \
  --reporters cli,json \
  --reporter-json-export results/newman-report.json

# Chạy với HTML report
newman run my-api-tests.json \
  --environment staging.json \
  --reporters cli,htmlextra \
  --reporter-htmlextra-export reports/api-test-report.html \
  --reporter-htmlextra-title "API Test Report - Sprint 5"

# Chạy folder cụ thể
newman run my-api-tests.json \
  --environment staging.json \
  --folder "Authentication"

# Chạy với timeout và retry
newman run my-api-tests.json \
  --environment staging.json \
  --timeout-request 5000 \
  --delay-request 100 \
  --iteration-count 3
```

**Tích hợp GitHub Actions:**
```yaml
# .github/workflows/api-tests.yml
name: API Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  api-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install Newman
        run: npm install -g newman newman-reporter-htmlextra
      
      - name: Run API Tests
        run: |
          newman run postman/collection.json \
            --environment postman/staging.json \
            --reporters cli,htmlextra \
            --reporter-htmlextra-export reports/api-report.html
      
      - name: Upload Test Report
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: api-test-report
          path: reports/api-report.html
```

---

## 6.8 Ví dụ thực hành — Bộ API Test hoàn chỉnh

### 6.8.1 Kiểm thử API CRUD Employee

**Giả định API:**
```
Base URL: {{base_url}}/api/v1
Auth: Bearer {{admin_token}}

Endpoints:
GET    /employees              ← Danh sách
POST   /employees              ← Tạo mới
GET    /employees/:id          ← Chi tiết
PUT    /employees/:id          ← Cập nhật
DELETE /employees/:id          ← Xóa
```

**Flow kiểm thử (thứ tự có phụ thuộc):**

```
Request 1: POST /employees — Tạo employee mới
Pre-request Script:
  const ts = Date.now();
  pm.environment.set("emp_email", `emp_${ts}@test.com`);

Body:
{
  "name": "Trần Thị Test",
  "email": "{{emp_email}}",
  "department": "Engineering",
  "position": "Junior Developer",
  "salary": 15000000,
  "startDate": "2025-01-15"
}

Test Script:
  pm.test("201 Created", () => pm.response.to.have.status(201));
  pm.test("Response có employee ID", () => {
      const body = pm.response.json();
      pm.expect(body.data.id).to.be.a("number");
      pm.environment.set("emp_id", body.data.id);
  });
  pm.test("Dữ liệu trả về đúng", () => {
      const data = pm.response.json().data;
      pm.expect(data.name).to.equal("Trần Thị Test");
      pm.expect(data.salary).to.equal(15000000);
      pm.expect(data.status).to.equal("active");
  });

---

Request 2: GET /employees/{{emp_id}} — Lấy employee vừa tạo
Test Script:
  pm.test("200 OK", () => pm.response.to.have.status(200));
  pm.test("Lấy đúng employee", () => {
      const data = pm.response.json().data;
      pm.expect(data.id).to.equal(pm.environment.get("emp_id"));
      pm.expect(data.email).to.equal(pm.environment.get("emp_email"));
  });

---

Request 3: PATCH /employees/{{emp_id}} — Cập nhật lương
Body: { "salary": 18000000 }
Test Script:
  pm.test("200 OK", () => pm.response.to.have.status(200));
  pm.test("Lương đã cập nhật", () => {
      const data = pm.response.json().data;
      pm.expect(data.salary).to.equal(18000000);
      // Các field khác không thay đổi
      pm.expect(data.name).to.equal("Trần Thị Test");
      pm.expect(data.email).to.equal(pm.environment.get("emp_email"));
  });

---

Request 4: DELETE /employees/{{emp_id}} — Xóa employee
Test Script:
  pm.test("204 No Content", () => pm.response.to.have.status(204));

---

Request 5: GET /employees/{{emp_id}} — Verify đã xóa
Test Script:
  pm.test("404 Not Found sau khi xóa", () => pm.response.to.have.status(404));
```

---

## 6.9 Database Testing trong API Testing

Kiểm thử API hiệu quả không chỉ dừng ở response — cần verify dữ liệu trong database để đảm bảo tính toàn vẹn.

### 6.9.1 Chiến lược verify database

**Cách 1: Dùng GET API để verify (không cần truy cập DB trực tiếp)**

```
// Sau POST /orders
// Dùng GET /orders/:id để verify dữ liệu được lưu đúng
pm.test("Order được lưu đúng", async function() {
    const orderId = pm.environment.get("order_id");
    const response = await pm.sendRequest({
        url: `${pm.environment.get("base_url")}/api/v1/orders/${orderId}`,
        method: "GET",
        header: { Authorization: `Bearer ${pm.environment.get("token")}` }
    });
    const order = response.json().data;
    pm.expect(order.status).to.equal("pending");
    pm.expect(order.total).to.equal(300000);
});
```

**Cách 2: SQL trực tiếp (khi có quyền truy cập DB)**

```sql
-- Sau khi POST /api/v1/orders với 2 sản phẩm

-- Verify order được tạo
SELECT id, status, total, user_id, created_at
FROM orders
WHERE id = 'ORD-2025-001';

-- Verify order items
SELECT oi.product_id, oi.quantity, oi.unit_price, p.stock
FROM order_items oi
JOIN products p ON oi.product_id = p.id
WHERE oi.order_id = 'ORD-2025-001';

-- Verify tồn kho giảm đúng
SELECT id, name, stock
FROM products
WHERE id IN ('P001', 'P003');

-- Verify email được queue
SELECT recipient, subject, status, created_at
FROM email_queue
WHERE reference_id = 'ORD-2025-001'
ORDER BY created_at DESC;
```
