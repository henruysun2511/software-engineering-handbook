# CƠ CHẾ HOẠT ĐỘNG CỦA EXPRESS.JS

## I. Đặt vấn đề

Node.js cung cấp module `http` để dựng server, nhưng module này khá thô sơ: mọi request đều phải tự tay parse URL, phân loại phương thức (`GET`/`POST`...), xử lý body, định tuyến (routing)... Khi ứng dụng lớn lên, viết thủ công như vậy trở nên cồng kềnh và khó bảo trì.

Express.js là một framework nhẹ, xây dựng trên module `http` của Node.js, cung cấp một lớp trừu tượng giúp lập trình viên định tuyến, xử lý request/response và tổ chức logic ứng dụng theo cách có cấu trúc, thông qua một cơ chế cốt lõi duy nhất: **Middleware**.

---

## II. Bản chất của Express.js

Về bản chất, một ứng dụng Express vẫn là một server HTTP thuần của Node.js — Express chỉ "bọc" (wrap) quanh `http.createServer()` và thêm vào đó một hàng ống dẫn xử lý (pipeline) gồm nhiều hàm nối tiếp nhau, gọi là middleware.

```javascript
const express = require('express');
const app = express();

app.listen(3000);
```

Thực chất, `app` ở trên là một hàm callback được truyền vào `http.createServer(app)` phía dưới. Mỗi khi có request đến, Express không xử lý ngay bằng một hàm cố định, mà đẩy request đó đi qua một chuỗi middleware đã được đăng ký trước, theo đúng thứ tự khai báo.

---

## III. Middleware — cơ chế cốt lõi

### 1. Middleware là gì?
Middleware là một hàm có dạng:

```javascript
function middleware(req, res, next) {
  // xử lý logic (đọc, ghi, kiểm tra req/res)
  next(); // chuyển quyền xử lý cho middleware tiếp theo
}
```

Ba tham số:
- `req`: Object chứa thông tin request (headers, body, params, query...)
- `res`: Object dùng để phản hồi (gửi dữ liệu, set status, header...)
- `next`: Hàm gọi để chuyển tiếp sang middleware kế tiếp trong chuỗi. Nếu không gọi `next()` (và cũng không gửi response), request sẽ bị "treo" mãi mãi (client chờ vô thời hạn).

### 2. Bản chất: một hàng đợi (pipeline) tuần tự
Khi đăng ký nhiều middleware bằng `app.use()` hoặc `app.get()`/`post()`..., Express lưu chúng thành một mảng có thứ tự. Với mỗi request đến, Express khởi tạo một con trỏ chạy tuần tự qua mảng này: middleware nào gọi `next()` thì con trỏ nhảy sang middleware kế; nếu một middleware gửi response (`res.send()`, `res.json()`...) thì chuỗi dừng lại tại đó.

```text
Request đến
     │
     ▼
┌─────────────┐   next()   ┌─────────────┐   next()   ┌──────────────┐
│ Middleware 1│ ─────────► │ Middleware 2│ ─────────► │  Route Handler│
│ (vd: log)   │            │(vd: auth)   │            │ (gửi response)│
└─────────────┘            └─────────────┘            └──────────────┘
                                   │
                          (nếu lỗi) next(err)
                                   ▼
                          ┌─────────────────┐
                          │ Error-handling   │
                          │   middleware     │
                          └─────────────────┘
```

### 3. Phân loại middleware

| Loại | Mô tả | Ví dụ |
| :--- | :--- | :--- |
| **Application-level** | Gắn trực tiếp vào app | `app.use(express.json())` |
| **Router-level** | Gắn vào một đối tượng `express.Router()` riêng | `router.use(...)` |
| **Built-in** | Có sẵn trong Express | `express.json()`, `express.static()` |
| **Third-party** | Thư viện bên ngoài | `cors`, `morgan`, `helmet` |
| **Error-handling** | Có 4 tham số `(err, req, res, next)`, chỉ chạy khi có lỗi được `next(err)` truyền vào | `app.use((err, req, res, next) => {...})` |

---

## IV. Luồng xử lý một request (Request-Response Cycle)

Khi client gửi request đến server Express, luồng xử lý diễn ra theo các bước:
1. Node.js nhận request thô ở tầng http (thông qua Event Loop / libuv như đã trình bày ở phần Node.js Runtime).
2. Express nhận request, tạo `req` và `res` với các phương thức mở rộng tiện lợi (`req.params`, `res.json()`...).
3. Express duyệt tuần tự qua stack middleware toàn cục đã đăng ký bằng `app.use()`.
4. Đến bước định tuyến (routing): Express so khớp method + path của request với các route đã khai báo (`app.get('/users', ...)`).
5. Nếu khớp, chạy tiếp middleware/route-handler tương ứng với route đó.
6. Route handler thường kết thúc chuỗi bằng cách gọi `res.send()`/`json()`/`end()` để trả response về client.
7. Nếu tại bất kỳ bước nào xảy ra lỗi (`throw`, hoặc gọi `next(err)`), Express bỏ qua toàn bộ middleware thường, nhảy thẳng tới middleware xử lý lỗi (error-handling middleware) gần nhất.
8. Nếu không middleware/route nào khớp và xử lý response, Express trả về mặc định lỗi 404.

```text
       ┌───────────────────────────────┐
        │      Request từ client        │
        └──────────────┬─────────────────┘
                        ▼
        ┌───────────────────────────────┐
        │  Middleware toàn cục (app.use) │  (parser, log, cors...)
        └──────────────┬─────────────────┘
                        ▼
        ┌───────────────────────────────┐
        │   So khớp Route (method+path)  │
        └──────────────┬─────────────────┘
                        ▼
        ┌───────────────────────────────┐
        │  Route handler (business logic)│
        └──────────────┬─────────────────┘
                        ▼
              res.send() / res.json()
                        ▼
        ┌───────────────────────────────┐
        │       Response về client       │
        └───────────────────────────────┘
```

---

## V. Cơ chế Routing

Express cho phép định nghĩa route gắn với phương thức HTTP và đường dẫn cụ thể:

```javascript
app.get('/users/:id', (req, res) => {
  res.json({ id: req.params.id });
});
```

- `:id` là route parameter, Express tự động parse và gán vào `req.params.id`.
- Express so khớp route theo thứ tự khai báo từ trên xuống — route nào khai báo trước, khớp trước sẽ được ưu tiên xử lý.
- Có thể nhóm route bằng `express.Router()` để tổ chức theo module (ví dụ tách riêng `userRouter`, `productRouter`), sau đó gắn vào ứng dụng chính bằng `app.use('/api/users', userRouter)`.

---

## VI. Ví dụ minh họa tổng hợp

```javascript
const express = require('express');
const app = express();

// Middleware toàn cục: parse JSON body
app.use(express.json());

// Middleware log request
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

// Route
app.get('/users/:id', (req, res) => {
  res.json({ id: req.params.id, name: 'Nguyen Van A' });
});

// Middleware xử lý lỗi (đặt cuối cùng)
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message });
});

app.listen(3000, () => console.log('Server đang chạy tại cổng 3000'));
```

Khi client gọi `GET /users/5`: request đi qua middleware `express.json()` $\rightarrow$ middleware log (in ra console) $\rightarrow$ khớp route `/users/:id` $\rightarrow$ trả về JSON. Nếu route handler ném lỗi (`throw`), Express sẽ chuyển thẳng đến middleware xử lý lỗi ở cuối, bỏ qua các bước còn lại.

---

## VII. Kết luận

Bản chất của Express.js không nằm ở việc thay thế Node.js, mà là xây dựng một lớp điều phối (pipeline) middleware chạy tuần tự trên nền `http` module sẵn có. Mọi tính năng của Express — parse body, định tuyến, xử lý lỗi, xác thực... — đều được triển khai dưới dạng middleware nối tiếp nhau, tuân theo nguyên tắc "request đi qua từng trạm xử lý cho đến khi có response được gửi đi". Hiểu rõ cơ chế middleware và luồng request-response chính là chìa khóa để làm chủ Express.js trong xây dựng ứng dụng backend với Node.js.
