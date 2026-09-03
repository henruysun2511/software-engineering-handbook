# CƠ CHẾ HOẠT ĐỘNG CỦA TYPESCRIPT

## I. Đặt vấn đề

JavaScript là ngôn ngữ kiểu động (dynamically typed): kiểu dữ liệu của biến chỉ được xác định lúc chạy (runtime), không được kiểm tra trước. Điều này giúp JS linh hoạt, nhưng cũng là nguồn gốc của rất nhiều lỗi chỉ phát hiện được khi chương trình đã chạy — ví dụ gọi nhầm phương thức trên `undefined`, truyền sai kiểu tham số cho hàm...

TypeScript (TS) ra đời để giải quyết vấn đề đó bằng cách bổ sung hệ thống kiểu tĩnh (static type system) vào JavaScript, giúp phát hiện lỗi ngay khi viết code — trước khi chương trình được chạy.

---

## II. Bản chất của TypeScript

### 1. TypeScript là một "superset" của JavaScript
Về bản chất, TypeScript không phải một ngôn ngữ hoàn toàn mới, mà là JavaScript được bổ sung cú pháp khai báo kiểu (type annotation). Mọi mã JavaScript hợp lệ đều là mã TypeScript hợp lệ — TS chỉ mở rộng thêm, không thay thế.

```typescript
let age: number = 25;       // TypeScript: khai báo rõ kiểu
let name = 'Nguyen Van A';   // TypeScript vẫn tự suy luận kiểu string
```

### 2. TypeScript chỉ tồn tại ở "compile-time" — không tồn tại lúc runtime
Đây là bản chất quan trọng nhất cần nắm: trình duyệt và Node.js không hiểu TypeScript. Chúng chỉ chạy được JavaScript thuần. Do đó, mọi đoạn mã TS đều phải trải qua một bước biên dịch (compile) để chuyển thành JavaScript trước khi thực thi — bước này gọi là transpiling (biên dịch từ ngôn ngữ này sang ngôn ngữ khác ở cùng mức trừu tượng).

Cơ chế cốt lõi ở đây là **type erasure** (xóa kiểu): toàn bộ thông tin về kiểu dữ liệu (`: number`, `: string`, `interface`...) chỉ tồn tại trong quá trình biên dịch để trình biên dịch (compiler) kiểm tra logic, sau đó bị loại bỏ hoàn toàn khi sinh ra mã JavaScript cuối cùng — file `.js` sinh ra không còn bất kỳ dấu vết nào của kiểu dữ liệu.

```typescript
// input.ts
function add(a: number, b: number): number {
  return a + b;
}
```

```javascript
// output.js (sau khi biên dịch)
function add(a, b) {
  return a + b;
}
```

---

## III. Luồng biên dịch của TypeScript Compiler (tsc)

Khi chạy lệnh `tsc`, trình biên dịch TypeScript thực hiện tuần tự qua 3 giai đoạn chính:

```text
  File .ts (mã nguồn)
          │
          ▼
┌───────────────────┐
│  1. Parsing         │  → phân tích cú pháp, tạo AST (Abstract Syntax Tree)
└─────────┬───────────┘
          ▼
┌───────────────────┐
│  2. Type Checking   │  → đối chiếu kiểu dữ liệu, báo lỗi nếu sai kiểu
└─────────┬───────────┘
          ▼
┌───────────────────┐
│  3. Emit (Transform)│  → xóa toàn bộ annotation kiểu, sinh ra mã .js thuần
└─────────┬───────────┘
          ▼
      File .js (chạy được trên Node.js/trình duyệt)
```

> **Điểm mấu chốt**: Bước Type Checking (2) và bước Emit (3) là độc lập với nhau. Ngay cả khi Type Checking phát hiện lỗi kiểu (ví dụ gán string cho biến number), theo mặc định trình biên dịch vẫn sinh ra file `.js` (trừ khi bật cấu hình `noEmitOnError`). Điều này khẳng định rõ: TypeScript chỉ là công cụ kiểm tra tĩnh (static analysis) hỗ trợ lập trình viên, không phải là một "lá chắn" runtime — khi đã ra tới JavaScript, không còn gì kiểm tra kiểu nữa.

---

## IV. Hệ thống kiểu (Type System)

### 1. Structural Typing (Kiểu cấu trúc — "duck typing")
Khác với các ngôn ngữ như Java/C# dùng Nominal Typing (hai kiểu được coi là tương thích chỉ khi cùng tên/cùng khai báo kế thừa), TypeScript dùng Structural Typing: hai kiểu được coi là tương thích nếu chúng có cùng cấu trúc (cùng thuộc tính, cùng kiểu dữ liệu tương ứng), bất kể tên gọi khác nhau.

```typescript
interface Point { x: number; y: number; }

function printPoint(p: Point) { console.log(p.x, p.y); }

const obj = { x: 10, y: 20, z: 30 }; // không khai báo là Point
printPoint(obj); // vẫn hợp lệ, vì obj có đủ cấu trúc { x: number, y: number }
```

### 2. Type Inference (Suy luận kiểu)
TypeScript không bắt buộc khai báo kiểu ở mọi nơi — trình biên dịch có khả năng tự suy luận kiểu dựa vào giá trị khởi tạo:

```typescript
let count = 10; // TS tự suy luận count: number, không cần viết ": number"
```

### 3. Các khối xây dựng kiểu chính

| Khái niệm | Vai trò | Ví dụ |
| :--- | :--- | :--- |
| **interface** | Định nghĩa hình dạng (shape) của object | `interface User { id: number; name: string }` |
| **type** | Tương tự interface, nhưng linh hoạt hơn (union, intersection...) | `type ID = number \| string` |
| **Generics** | Viết code tái sử dụng được với nhiều kiểu dữ liệu khác nhau | `function identity<T>(arg: T): T` |
| **Enum** | Tập hợp các hằng số có tên | `enum Role { Admin, User }` |
| **Union / Intersection** | Kết hợp nhiều kiểu | `type A = string \| number`, `type B = X & Y` |

---

## V. Ví dụ minh họa tổng hợp

```typescript
interface Product {
  id: number;
  name: string;
  price: number;
}

function getDiscountedPrice(product: Product, percent: number): number {
  return product.price * (1 - percent / 100);
}

const laptop: Product = { id: 1, name: 'Laptop', price: 1000 };
console.log(getDiscountedPrice(laptop, 10)); // 900

// getDiscountedPrice(laptop, '10'); // ❌ Lỗi compile-time: tham số phải là number
```

Nếu gọi hàm với tham số `percent` là chuỗi `'10'`, trình biên dịch sẽ báo lỗi ngay khi viết code (trong IDE hoặc khi chạy `tsc`) — thay vì phải đợi đến lúc chạy chương trình mới phát hiện ra lỗi logic, như cách JavaScript thuần vẫn làm.

---

## VI. So sánh TypeScript và JavaScript

| Tiêu chí | JavaScript | TypeScript |
| :--- | :--- | :--- |
| **Hệ thống kiểu** | Động (dynamic), kiểm tra lúc runtime | Tĩnh (static), kiểm tra lúc compile-time |
| **Phát hiện lỗi** | Khi chương trình chạy | Ngay khi viết code / biên dịch |
| **Cần biên dịch?** | Không (chạy trực tiếp) | Có (phải transpile ra .js) |
| **Hỗ trợ công cụ (IDE)** | Gợi ý kiểu hạn chế | Autocomplete, kiểm tra kiểu, refactor an toàn |
| **Tính năng OOP nâng cao** | Cơ bản (ES6 class) | Đầy đủ: interface, generic, decorator, access modifier (private, protected) |
| **Độ phù hợp** | Dự án nhỏ, script nhanh | Dự án lớn, nhiều người tham gia, cần độ ổn định cao |
| **Tồn tại lúc runtime** | Có | Không (bị xóa hoàn toàn qua type erasure) |

---

## VII. Ưu điểm của TypeScript

- **Phát hiện lỗi sớm**: Hầu hết lỗi liên quan đến sai kiểu dữ liệu được bắt ngay tại thời điểm viết code, giảm đáng kể lỗi runtime trong môi trường production.
- **Tăng khả năng đọc hiểu và bảo trì code**: Kiểu dữ liệu đóng vai trò như tài liệu sống (self-documenting), giúp lập trình viên khác hiểu nhanh input/output của hàm mà không cần đọc hết thân hàm.
- **Hỗ trợ IDE mạnh mẽ**: Autocomplete chính xác, gợi ý refactor an toàn (đổi tên biến/hàm ảnh hưởng toàn dự án mà không sợ sót chỗ).
- **An toàn khi refactor các dự án lớn**: Khi thay đổi cấu trúc dữ liệu, trình biên dịch chỉ ra ngay tất cả những nơi bị ảnh hưởng.
- **Tương thích ngược hoàn toàn với JavaScript**: Có thể áp dụng dần dần (gradual adoption) vào dự án JS hiện có mà không cần viết lại từ đầu.
- **Là nền tảng của các framework hiện đại**: Angular, NestJS được thiết kế xoay quanh TypeScript (dùng Decorator, Metadata), nên nắm vững TypeScript là điều kiện cần để khai thác hết sức mạnh của các framework này.

---

## VIII. Kết luận

Bản chất của TypeScript là một công cụ kiểm tra tĩnh (static type checker) đặt trước JavaScript trong quy trình phát triển, không phải một runtime hay ngôn ngữ hoàn toàn tách biệt. Toàn bộ giá trị của TypeScript nằm ở giai đoạn biên dịch: phân tích cú pháp, đối chiếu kiểu dữ liệu, rồi xóa bỏ hoàn toàn thông tin kiểu để sinh ra JavaScript thuần túy. Nhờ cơ chế này, TypeScript giữ được sự tương thích tuyệt đối với hệ sinh thái JavaScript/Node.js hiện có, đồng thời mang lại độ an toàn và khả năng mở rộng vượt trội — lý do vì sao TypeScript đã trở thành lựa chọn mặc định cho hầu hết các dự án backend (NestJS) lẫn frontend (Angular, React với TS) quy mô vừa và lớn hiện nay.
