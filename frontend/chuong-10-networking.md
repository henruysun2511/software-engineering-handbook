# Chương 10: Networking

Networking là nền tảng của mọi ứng dụng web. Frontend developer cần hiểu cách giao tiếp giữa trình duyệt và server diễn ra — từ giao thức HTTP, bảo mật HTTPS, đến cách thiết kế và tiêu thụ REST API.

---

## 10.1. HTTP (HyperText Transfer Protocol)

HTTP là giao thức truyền tải văn bản siêu liên kết, là nền tảng giao tiếp của World Wide Web. Mọi tương tác giữa trình duyệt và server đều thông qua HTTP.

### Request

HTTP Request là thông điệp trình duyệt gửi lên server, gồm bốn phần:

```
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer eyJhbGc...

{
  "name": "An",
  "email": "an@example.com"
}
```

| Thành phần | Mô tả | Ví dụ |
|---|---|---|
| **Request Line** | Method + URI + HTTP version | `POST /api/users HTTP/1.1` |
| **Headers** | Metadata của request | `Content-Type`, `Authorization` |
| **Body** | Dữ liệu gửi kèm (không bắt buộc) | JSON, form data, file |

### Response

HTTP Response là thông điệp server trả về sau khi xử lý request:

```
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/users/42

{
  "id": 42,
  "name": "An",
  "email": "an@example.com"
}
```

| Thành phần | Mô tả | Ví dụ |
|---|---|---|
| **Status Line** | HTTP version + status code + reason | `HTTP/1.1 201 Created` |
| **Headers** | Metadata của response | `Content-Type`, `Cache-Control` |
| **Body** | Dữ liệu trả về | JSON, HTML, file |

### HTTP Methods

| Method | Mục đích | Có Body | Idempotent |
|---|---|---|---|
| `GET` | Lấy dữ liệu | Không | Có |
| `POST` | Tạo mới | Có | Không |
| `PUT` | Thay thế toàn bộ | Có | Có |
| `PATCH` | Cập nhật một phần | Có | Không nhất thiết |
| `DELETE` | Xóa | Không | Có |
| `HEAD` | Như GET nhưng không có body | Không | Có |
| `OPTIONS` | Hỏi server hỗ trợ method nào | Không | Có |

### Stateless

HTTP là giao thức **không trạng thái (stateless)** — mỗi request là độc lập hoàn toàn. Server không lưu bất kỳ thông tin nào về các request trước đó. Điều này có nghĩa:

- Server không biết request này từ cùng người dùng với request trước.
- Mỗi request phải tự mang đầy đủ thông tin cần thiết (ví dụ: token xác thực).
- Ưu điểm: server dễ scale ngang vì không cần lưu session.
- Nhược điểm: cần cơ chế riêng để duy trì trạng thái (cookie, token).

---

## 10.2. HTTPS

HTTPS (HTTP Secure) là HTTP được bảo mật bằng giao thức **TLS (Transport Layer Security)**. Mọi dữ liệu truyền qua HTTPS đều được mã hóa — không ai có thể đọc được kể cả khi chặn được gói tin.

### SSL/TLS

SSL (Secure Sockets Layer) là tiền thân của TLS. Dù thuật ngữ "SSL" vẫn được dùng phổ biến, thực tế hiện nay đều dùng **TLS** (phiên bản 1.2 hoặc 1.3).

TLS cung cấp ba đảm bảo:
- **Mã hóa (Encryption):** Dữ liệu được mã hóa, bên thứ ba không đọc được.
- **Toàn vẹn (Integrity):** Dữ liệu không bị sửa đổi trong quá trình truyền.
- **Xác thực (Authentication):** Đảm bảo đang kết nối đúng server thật.

### TLS Handshake

Trước khi truyền dữ liệu, trình duyệt và server thực hiện "bắt tay" để thống nhất phương thức mã hóa và xác minh danh tính:

```
Client                          Server
  |                               |
  |── ClientHello ───────────────>|  (phiên bản TLS, cipher suites hỗ trợ)
  |<── ServerHello ───────────────|  (chọn cipher suite)
  |<── Certificate ───────────────|  (chứng chỉ SSL của server)
  |<── ServerHelloDone ───────────|
  |                               |
  |  [Xác minh Certificate]       |
  |                               |
  |── ClientKeyExchange ─────────>|  (trao đổi khóa mã hóa)
  |── ChangeCipherSpec ──────────>|
  |── Finished ──────────────────>|
  |<── ChangeCipherSpec ──────────|
  |<── Finished ──────────────────|
  |                               |
  |====== Dữ liệu mã hóa =========|
```

### Certificate (Chứng chỉ SSL)

Certificate là tài liệu số do **Certificate Authority (CA)** uy tín cấp, xác nhận rằng domain thực sự thuộc sở hữu của tổ chức khai báo. Trình duyệt kiểm tra certificate khi kết nối HTTPS; nếu không hợp lệ, hiển thị cảnh báo "Kết nối không an toàn".

Với Next.js, HTTPS thường được xử lý ở tầng infrastructure (Vercel, Nginx, CDN) — không cần cấu hình trong code ứng dụng.

---

## 10.3. REST (Representational State Transfer)

REST là một kiến trúc thiết kế API dựa trên HTTP. API tuân thủ các nguyên tắc REST được gọi là **RESTful API**.

### Resource

Trong REST, mọi thứ đều là **resource** — một đối tượng hoặc khái niệm có thể định danh và thao tác. Resource được đặt tên bằng danh từ số nhiều.

```
/users          → tập hợp users
/users/42       → user có id = 42
/users/42/posts → tập hợp posts của user 42
```

### URI (Uniform Resource Identifier)

URI là địa chỉ định danh duy nhất cho một resource. Nguyên tắc thiết kế URI chuẩn:

```
✅ Đúng                         ❌ Sai
GET    /users                   GET /getUsers
POST   /users                   POST /createUser
GET    /users/42                GET /user/42
PATCH  /users/42                POST /users/42/update
DELETE /users/42                GET  /users/42/delete
GET    /users/42/orders         GET /getUserOrders?userId=42
```

### Stateless

Giống HTTP, REST **không lưu trạng thái** giữa các request. Mỗi request phải tự mang đủ context để server xử lý — thường là token xác thực trong header.

### Idempotent

Một request được gọi là **idempotent** nếu gọi nhiều lần vẫn cho cùng kết quả với lần đầu tiên.

| Method | Idempotent | Giải thích |
|---|---|---|
| `GET` | ✅ | Đọc dữ liệu, không thay đổi |
| `PUT` | ✅ | Thay thế toàn bộ resource — gọi lại cho cùng kết quả |
| `DELETE` | ✅ | Xóa rồi xóa lại — resource vẫn không tồn tại |
| `POST` | ❌ | Mỗi lần gọi tạo ra một record mới |
| `PATCH` | ❌ | Phụ thuộc cách implement |

Tính idempotent quan trọng khi xử lý retry: nếu request idempotent bị lỗi mạng, có thể gọi lại an toàn mà không lo tạo dữ liệu trùng.

---

## 10.4. HTTP Status Code

Status code là con số 3 chữ số server trả về để biểu thị kết quả xử lý request. Nhóm hàng trăm cho biết loại kết quả:

### Nhóm 2xx — Thành công

| Code | Ý nghĩa | Dùng khi |
|---|---|---|
| `200 OK` | Thành công | GET, PUT, PATCH thành công |
| `201 Created` | Tạo mới thành công | POST tạo resource |
| `204 No Content` | Thành công, không có body | DELETE thành công |

### Nhóm 3xx — Chuyển hướng

| Code | Ý nghĩa | Dùng khi |
|---|---|---|
| `301 Moved Permanently` | Chuyển hướng vĩnh viễn | Domain đổi, URL cũ |
| `302 Found` | Chuyển hướng tạm thời | Redirect sau login |
| `304 Not Modified` | Không thay đổi | Cache còn hợp lệ |

### Nhóm 4xx — Lỗi từ Client

| Code | Ý nghĩa | Dùng khi |
|---|---|---|
| `400 Bad Request` | Request không hợp lệ | Validation fail, sai format |
| `401 Unauthorized` | Chưa xác thực | Thiếu hoặc sai token |
| `403 Forbidden` | Không có quyền | Đúng token nhưng không đủ quyền |
| `404 Not Found` | Không tìm thấy | Resource không tồn tại |
| `409 Conflict` | Xung đột | Email đã tồn tại |
| `422 Unprocessable` | Không xử lý được | Validation lỗi nghiệp vụ |
| `429 Too Many Requests` | Quá nhiều request | Rate limit |

### Nhóm 5xx — Lỗi từ Server

| Code | Ý nghĩa | Dùng khi |
|---|---|---|
| `500 Internal Server Error` | Lỗi server không xác định | Bug, exception không bắt được |
| `502 Bad Gateway` | Gateway nhận response lỗi | Server phía sau lỗi |
| `503 Service Unavailable` | Server không sẵn sàng | Đang bảo trì, quá tải |

> **Quy tắc thực tế:** `401` — chưa đăng nhập → redirect về trang login. `403` — đã đăng nhập nhưng không có quyền → hiển thị thông báo lỗi quyền, không redirect.

---

## 10.5. HTTP Header

Header là metadata đính kèm trong request hoặc response, cung cấp thông tin bổ sung về cách xử lý dữ liệu.

### Authorization

Mang thông tin xác thực. Phổ biến nhất là Bearer token (JWT):

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Content-Type

Khai báo định dạng của body trong request hoặc response:

```
Content-Type: application/json          # Dữ liệu JSON
Content-Type: multipart/form-data       # Upload file
Content-Type: application/x-www-form-urlencoded  # Form truyền thống
```

### Accept

Client thông báo cho server định dạng response mong muốn:

```
Accept: application/json
Accept: text/html, application/json;q=0.9
```

### Cookie

Gửi kèm cookie lên server trong mỗi request. Server đặt cookie qua response header `Set-Cookie`:

```
# Response từ server
Set-Cookie: token=abc123; HttpOnly; Secure; SameSite=Strict; Max-Age=3600

# Request từ client (tự động)
Cookie: token=abc123
```

### Cache-Control

Kiểm soát hành vi cache của trình duyệt và proxy:

```
Cache-Control: no-store               # Không cache
Cache-Control: no-cache               # Cache nhưng phải validate với server
Cache-Control: max-age=3600           # Cache tối đa 1 tiếng
Cache-Control: public, max-age=86400  # Cache công khai (CDN), 1 ngày
Cache-Control: private, max-age=600   # Cache riêng tư (chỉ browser), 10 phút
```

### Origin

Khai báo nguồn gốc của request — trình duyệt tự động thêm header này trong cross-origin request. Server dùng để kiểm tra CORS:

```
Origin: https://app.example.com
```

Server phản hồi bằng:

```
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
```

---

## 10.6. HTTP Client — Axios

Axios là thư viện HTTP client phổ biến nhất cho JavaScript, cung cấp API đơn giản hơn `fetch` với nhiều tính năng tích hợp sẵn: tự động parse JSON, interceptor, timeout, và cancel request.

### Cấu hình instance

Thay vì dùng `axios` trực tiếp, tạo **instance** với cấu hình riêng — dễ quản lý và test:

```tsx
// lib/axios.ts
import axios from "axios";

const apiClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL ?? "https://api.example.com",
  timeout: 10000,                            // 10 giây
  headers: {
    "Content-Type": "application/json",
  },
});

export default apiClient;
```

### Interceptor

Interceptor là middleware của Axios — cho phép xử lý **mọi request hoặc response** trước khi chúng được xử lý bởi code nghiệp vụ. Đây là nơi lý tưởng để đặt logic xác thực, logging, và xử lý lỗi tập trung.

**Request Interceptor** — chạy trước khi request gửi đi:

```tsx
// Tự động gắn token vào mọi request
apiClient.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem("access_token");
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);
```

**Response Interceptor** — chạy sau khi nhận response:

```tsx
// Xử lý lỗi tập trung và tự động refresh token
apiClient.interceptors.response.use(
  (response) => response,           // Response thành công — pass through
  async (error) => {
    const original = error.config;

    // Token hết hạn và chưa thử refresh
    if (error.response?.status === 401 && !original._retry) {
      original._retry = true;

      try {
        const { data } = await axios.post("/api/auth/refresh", {
          refreshToken: localStorage.getItem("refresh_token"),
        });

        localStorage.setItem("access_token", data.accessToken);
        original.headers.Authorization = `Bearer ${data.accessToken}`;

        return apiClient(original);  // Thử lại request ban đầu
      } catch {
        // Refresh thất bại — đăng xuất
        localStorage.clear();
        window.location.href = "/login";
      }
    }

    return Promise.reject(error);
  }
);
```

### Timeout

Timeout xác định thời gian tối đa chờ response. Nếu server không phản hồi trong thời gian này, request sẽ bị hủy với lỗi `ECONNABORTED`.

```tsx
// Cấu hình mặc định khi tạo instance
const apiClient = axios.create({
  timeout: 10000, // 10 giây
});

// Ghi đè cho từng request cụ thể
const response = await apiClient.get("/api/large-export", {
  timeout: 60000, // 60 giây cho export lớn
});
```

### AbortController và Cancel Request

`AbortController` là Web API chuẩn để hủy request đang chạy. Trường hợp phổ biến nhất: người dùng điều hướng sang trang khác trước khi request hoàn tất.

```tsx
// hooks/useSearchUsers.ts
import { useState, useEffect } from "react";
import apiClient from "@/lib/axios";

function useSearchUsers(query: string) {
  const [users, setUsers] = useState<User[]>([]);
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    if (!query.trim()) {
      setUsers([]);
      return;
    }

    const controller = new AbortController();
    setIsLoading(true);

    apiClient
      .get<User[]>("/api/users/search", {
        params: { q: query },
        signal: controller.signal,  // Kết nối với AbortController
      })
      .then((res) => setUsers(res.data))
      .catch((err) => {
        if (!axios.isCancel(err)) {
          console.error("Search error:", err);
        }
        // Nếu bị cancel — bỏ qua, không set state
      })
      .finally(() => setIsLoading(false));

    // Cleanup: hủy request cũ mỗi khi query thay đổi hoặc unmount
    return () => controller.abort();
  }, [query]);

  return { users, isLoading };
}
```

### Retry

Retry là cơ chế tự động thử lại request khi gặp lỗi tạm thời (lỗi mạng, 5xx). Có thể implement bằng `axios-retry`:

```tsx
import axiosRetry from "axios-retry";

axiosRetry(apiClient, {
  retries: 3,                           // Thử tối đa 3 lần
  retryDelay: axiosRetry.exponentialDelay, // 1s, 2s, 4s
  retryCondition: (error) => {
    // Chỉ retry khi lỗi mạng hoặc lỗi 5xx
    return (
      axiosRetry.isNetworkError(error) ||
      axiosRetry.isRetryableError(error)
    );
  },
});
```

Hoặc implement thủ công không cần thư viện:

```tsx
async function fetchWithRetry<T>(
  url: string,
  maxRetries: number = 3
): Promise<T> {
  let lastError: Error;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const { data } = await apiClient.get<T>(url);
      return data;
    } catch (error) {
      lastError = error as Error;

      const isRetryable =
        axios.isAxiosError(error) &&
        (!error.response || error.response.status >= 500);

      if (!isRetryable || attempt === maxRetries) break;

      // Exponential backoff: 1s, 2s, 4s
      await new Promise((resolve) =>
        setTimeout(resolve, 1000 * 2 ** attempt)
      );
    }
  }

  throw lastError!;
}
```

### So sánh Axios vs Fetch

| | `fetch` | `axios` |
|---|---|---|
| Built-in | Có | Cần cài |
| Auto JSON parse | Không | Có |
| Interceptor | Không | Có |
| Timeout | Không | Có |
| Cancel request | `AbortController` | `AbortController` |
| Retry | Thủ công | `axios-retry` |
| Error handling | 4xx/5xx không throw | Tự động throw |
| TypeScript | Cần type thủ công | Generic tốt |
| Node.js | Cần polyfill (<18) | Tích hợp sẵn |

> **Quy tắc thực tế:** Với Next.js, dùng `fetch` trong **Server Component** (Next.js mở rộng `fetch` với cache). Dùng `axios` trong **Client Component** khi cần interceptor, retry, hoặc xử lý lỗi tập trung.
