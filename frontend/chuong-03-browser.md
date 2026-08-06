# Chương 3: Browser

Trình duyệt không chỉ là nơi hiển thị HTML — nó cung cấp một bộ APIs phong phú để JavaScript tương tác với giao diện, lưu trữ dữ liệu và kiểm soát cách trang web được render.

---

## 3.1. DOM (Document Object Model)

DOM là cấu trúc dạng cây đại diện cho nội dung của một trang HTML. Mỗi thẻ HTML trở thành một **node** trong cây này. JavaScript có thể đọc, thêm, sửa, xóa bất kỳ node nào thông qua DOM API.

```
document
  └── <html>
        ├── <head>
        └── <body>
              ├── <header>
              └── <main>
                    └── <p>Hello</p>
```

### Truy vấn phần tử

```typescript
// Lấy một phần tử theo CSS selector
const btn = document.querySelector<HTMLButtonElement>(".submit-btn");

// Lấy tất cả phần tử khớp
const items = document.querySelectorAll<HTMLLIElement>(".list-item");
```

### Tạo và thêm phần tử

```typescript
// Tạo phần tử mới
const card = document.createElement("div");
card.classList.add("card");
card.textContent = "Hello, DOM!";

// Thêm vào trong container
const container = document.querySelector(".container");
container?.appendChild(card);

// Thêm nhiều phần tử hiệu quả hơn với fragment (tránh reflow nhiều lần)
const fragment = document.createDocumentFragment();

const users = ["An", "Bình", "Châu"];
users.forEach((name) => {
  const li = document.createElement("li");
  li.textContent = name;
  fragment.appendChild(li);
});

document.querySelector("ul")?.appendChild(fragment);
```

### Các thao tác DOM phổ biến

```typescript
const el = document.querySelector<HTMLDivElement>(".box");
if (!el) return;

// Thay đổi nội dung
el.textContent = "Text thuần";    // An toàn hơn
el.innerHTML = "<b>Bold</b>";     // Dùng khi cần HTML (cẩn thận XSS)

// Thay đổi style
el.style.color = "red";
el.classList.add("active");
el.classList.remove("hidden");
el.classList.toggle("open");

// Xóa phần tử
el.remove();
```

> **Lưu ý:** Trong React và các framework hiện đại, việc thao tác DOM trực tiếp là không cần thiết và thường gây ra lỗi — framework đã quản lý DOM thay cho lập trình viên.

---

## 3.2. Event

### Đăng ký và gỡ bỏ sự kiện

```typescript
const button = document.querySelector<HTMLButtonElement>("#my-btn");

function handleClick(event: MouseEvent): void {
  console.log("Đã click:", event.target);
}

// Đăng ký
button?.addEventListener("click", handleClick);

// Gỡ bỏ — phải dùng cùng reference hàm
button?.removeEventListener("click", handleClick);
```

### Event Bubbling và Capturing

Khi một sự kiện xảy ra trên một phần tử, nó không chỉ kích hoạt trên phần tử đó mà còn di chuyển qua cây DOM theo hai giai đoạn:

```
                document
               /        \
           <html>        \
             |         CAPTURING (trên xuống)
           <body>         |
             |           ↓
           <div>     [click xảy ra ở đây]
             |           ↑
           <button>    |
                     BUBBLING (dưới lên)
```

**Capturing (trên xuống):** Sự kiện di chuyển từ `document` xuống phần tử đích.
**Bubbling (dưới lên):** Sự kiện nổi bọt từ phần tử đích lên `document`.

Mặc định, `addEventListener` lắng nghe ở giai đoạn **bubbling**. Truyền `true` làm tham số thứ ba để chuyển sang capturing.

```typescript
document.querySelector(".parent")?.addEventListener(
  "click",
  (e) => console.log("Parent (capturing)"),
  true  // capturing phase
);

document.querySelector(".child")?.addEventListener("click", (e) => {
  console.log("Child (bubbling)");
  e.stopPropagation(); // Ngăn sự kiện nổi bọt lên parent
});
```

### Event Delegation

Thay vì gắn event listener lên từng phần tử con, gắn một listener duy nhất lên phần tử cha và kiểm tra xem phần tử nào thực sự được click.

**Không dùng delegation (kém hiệu quả):**

```typescript
document.querySelectorAll(".list-item").forEach((item) => {
  item.addEventListener("click", handleItemClick);
});
```

**Dùng delegation (hiệu quả, hỗ trợ element thêm sau):**

```typescript
document.querySelector(".list")?.addEventListener("click", (event) => {
  const target = event.target as HTMLElement;

  if (target.matches(".list-item")) {
    console.log("Clicked:", target.textContent);
  }
});
```

| | Gắn từng phần tử | Event Delegation |
|---|---|---|
| Số listener | Nhiều | Một |
| Phần tử thêm sau | Không tự nhận | Tự nhận |
| Bộ nhớ | Tốn | Tiết kiệm |

---

## 3.3. Storage

Trình duyệt cung cấp nhiều cơ chế lưu trữ dữ liệu phía client, mỗi loại phù hợp với mục đích khác nhau.

### Cookie

Cookie là mẫu dữ liệu nhỏ được gửi kèm theo mọi HTTP request đến server. Phù hợp để lưu session token, authentication.

```typescript
// Thiết lập cookie
document.cookie = "token=abc123; max-age=3600; path=/; Secure; SameSite=Strict";

// Đọc cookie (cần parse thủ công)
function getCookie(name: string): string | null {
  const value = document.cookie
    .split("; ")
    .find((row) => row.startsWith(name + "="))
    ?.split("=")[1];
  return value ?? null;
}
```

### LocalStorage

Lưu trữ dữ liệu dạng key-value không có thời hạn, tồn tại kể cả khi đóng trình duyệt.

```typescript
// Lưu — phải serialize object thành JSON
const user = { id: 1, name: "An" };
localStorage.setItem("user", JSON.stringify(user));

// Đọc — phải parse ngược lại
const saved = localStorage.getItem("user");
const parsed = saved ? JSON.parse(saved) : null;

// Xóa
localStorage.removeItem("user");
localStorage.clear(); // Xóa tất cả
```

### SessionStorage

Giống LocalStorage nhưng dữ liệu **bị xóa khi đóng tab**.

```typescript
sessionStorage.setItem("draft", JSON.stringify(formData));
const draft = sessionStorage.getItem("draft");
```

### IndexedDB

Cơ sở dữ liệu NoSQL trong trình duyệt, hỗ trợ lưu trữ dữ liệu lớn, có index, hỗ trợ transaction. Thường dùng qua thư viện như `idb` thay vì API trực tiếp do API native khá phức tạp.

```typescript
import { openDB } from "idb";

const db = await openDB("my-app", 1, {
  upgrade(db) {
    db.createObjectStore("users", { keyPath: "id" });
  },
});

await db.put("users", { id: 1, name: "An" });
const user = await db.get("users", 1);
```

### So sánh các loại Storage

| | Cookie | LocalStorage | SessionStorage | IndexedDB |
|---|---|---|---|---|
| Dung lượng | ~4 KB | ~5 MB | ~5 MB | Hàng trăm MB |
| Hết hạn | Cấu hình được | Không | Khi đóng tab | Không |
| Gửi lên server | Có (tự động) | Không | Không | Không |
| Truy cập từ JS | Có | Có | Có | Có |
| Dùng cho | Auth token | User prefs, cache | Form draft | Offline app, file |

---

## 3.4. Browser Rendering

Hiểu quá trình trình duyệt render trang giúp tối ưu hóa hiệu suất và tránh các thao tác gây chậm giao diện.

### Quy trình Rendering

```
HTML Parsing  →  DOM Construction
CSS Parsing   →  CSSOM Construction
                        ↓
              Render Tree (DOM + CSSOM hợp nhất,
                          chỉ chứa phần tử hiển thị)
                        ↓
                     Layout
              (Tính toán vị trí, kích thước)
                        ↓
                      Paint
              (Vẽ pixel: màu sắc, text, shadow)
                        ↓
                   Composite
              (Ghép các layer, hiển thị lên màn hình)
```

**Giải thích từng bước:**

- **DOM Construction:** Parser đọc HTML, tạo ra cây DOM.
- **CSSOM Construction:** Parser đọc CSS, tạo ra cây CSSOM (CSS Object Model).
- **Render Tree:** Trình duyệt kết hợp DOM và CSSOM, chỉ giữ lại các node thực sự hiển thị (bỏ qua `display: none`, `<head>`, v.v.).
- **Layout (Reflow):** Tính toán vị trí và kích thước chính xác của từng phần tử.
- **Paint:** Vẽ các pixel — màu sắc, text, đường viền, đổ bóng.
- **Composite:** Ghép các layer lại và đưa ra màn hình.

### Reflow vs Repaint

| | Reflow | Repaint |
|---|---|---|
| Khi nào xảy ra | Thay đổi layout (kích thước, vị trí, DOM) | Thay đổi visual không ảnh hưởng layout |
| Ví dụ | `width`, `height`, `margin`, thêm/xóa DOM | `color`, `background-color`, `box-shadow` |
| Chi phí | **Rất cao** — tính toán lại từ Layout | Cao — vẽ lại |
| Ảnh hưởng | Trigger cả Repaint | Không trigger Reflow |

> **Quy tắc:** Reflow tốn kém hơn Repaint. Repaint tốn kém hơn chỉ Composite.

### Thực hành tối ưu

```typescript
// Tệ — đọc và ghi xen kẽ, gây nhiều reflow
const h1 = element.offsetHeight;
element.style.height = h1 + 10 + "px";
const h2 = element.offsetHeight; // Buộc layout tính lại
element.style.height = h2 + 10 + "px";

// Tốt — đọc hết rồi ghi hết (batch reads/writes)
const h1 = element.offsetHeight;
const h2 = anotherElement.offsetHeight;
element.style.height = h1 + 10 + "px";
anotherElement.style.height = h2 + 10 + "px";
```

**Ưu tiên các thuộc tính chỉ trigger Composite** (không Reflow, không Repaint):
- `transform` thay vì `left/top`
- `opacity` thay vì `visibility`

```css
/* Tệ — trigger reflow */
.box:hover {
  left: 100px;
}

/* Tốt — chỉ trigger composite */
.box:hover {
  transform: translateX(100px);
}
```
