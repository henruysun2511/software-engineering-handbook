# Chương 6: CSS

CSS (Cascading Style Sheets) là ngôn ngữ tạo kiểu cho HTML. Nắm vững CSS là yêu cầu cơ bản của mọi frontend developer — không chỉ để làm đẹp giao diện, mà còn để kiểm soát layout, responsive, và hiệu suất render.

---

## 6.1. Box Model

Mọi phần tử HTML đều được trình duyệt xem là một hình chữ nhật (box). **Box Model** mô tả cách không gian của một phần tử được tính toán, gồm bốn lớp từ trong ra ngoài:

```
┌──────────────────────────────────────┐
│              margin                  │
│   ┌──────────────────────────────┐   │
│   │           border             │   │
│   │   ┌──────────────────────┐   │   │
│   │   │       padding        │   │   │
│   │   │   ┌──────────────┐   │   │   │
│   │   │   │   content    │   │   │   │
│   │   │   └──────────────┘   │   │   │
│   │   └──────────────────────┘   │   │
│   └──────────────────────────────┘   │
└──────────────────────────────────────┘
```

| Lớp | Mô tả |
|---|---|
| **content** | Vùng chứa nội dung thực tế (text, hình ảnh) |
| **padding** | Khoảng cách giữa nội dung và border — có màu nền |
| **border** | Đường viền bao quanh padding |
| **margin** | Khoảng cách bên ngoài border — trong suốt |

### box-sizing

`box-sizing` xác định cách trình duyệt tính toán tổng kích thước của phần tử:

```css
/* Mặc định — width chỉ tính content */
/* width=200px nhưng thực tế chiếm 200 + 20 + 4 = 224px */
.box {
  box-sizing: content-box;
  width: 200px;
  padding: 10px;
  border: 2px solid;
}

/* Chuẩn hiện đại — width bao gồm cả padding và border */
/* width=200px là tổng chiều rộng thực tế */
.box {
  box-sizing: border-box;
  width: 200px;
  padding: 10px;
  border: 2px solid;
}
```

**Best practice — Reset toàn bộ dự án về `border-box`:**

```css
*,
*::before,
*::after {
  box-sizing: border-box;
}
```

Với `border-box`, `width: 100%` luôn vừa khít container bất kể padding hay border — tránh tràn layout.

### Margin Collapse

Hai margin dọc (top/bottom) của các phần tử kề nhau **gộp lại thành một** (lấy giá trị lớn hơn), không cộng vào nhau. Điều này chỉ xảy ra theo chiều dọc, không xảy ra với chiều ngang hay với flexbox/grid.

```css
/* p1 margin-bottom: 24px, p2 margin-top: 16px */
/* Khoảng cách thực tế giữa hai đoạn văn: 24px (không phải 40px) */
p { margin-bottom: 24px; }
h2 { margin-top: 16px; }
```

---

## 6.2. Flexbox

Flexbox là mô hình layout một chiều — sắp xếp các phần tử theo **một trục** (ngang hoặc dọc). Phù hợp cho layout component, navigation bar, căn giữa phần tử.

Flexbox có hai loại trục: **main axis** (trục chính, theo `flex-direction`) và **cross axis** (trục chéo, vuông góc với main axis).

### flex-direction

Xác định hướng của main axis:

```css
.container {
  display: flex;
  flex-direction: row;            /* → mặc định: trái sang phải */
  /* flex-direction: row-reverse;    ← phải sang trái */
  /* flex-direction: column;         ↓ trên xuống dưới */
  /* flex-direction: column-reverse; ↑ dưới lên trên */
}
```

### justify-content

Căn chỉnh items theo **main axis**:

```css
.container {
  justify-content: flex-start;    /* Items dồn về đầu (mặc định) */
  justify-content: flex-end;      /* Items dồn về cuối */
  justify-content: center;        /* Items căn giữa */
  justify-content: space-between; /* Khoảng cách đều, không có ở mép */
  justify-content: space-around;  /* Khoảng cách đều, có ở mép (nửa) */
  justify-content: space-evenly;  /* Khoảng cách đều hoàn toàn */
}
```

### align-items

Căn chỉnh items theo **cross axis**:

```css
.container {
  align-items: stretch;     /* Kéo dãn theo chiều cao container (mặc định) */
  align-items: flex-start;  /* Dồn lên đầu cross axis */
  align-items: flex-end;    /* Dồn xuống cuối cross axis */
  align-items: center;      /* Căn giữa cross axis */
  align-items: baseline;    /* Căn theo baseline của text */
}
```

**Căn giữa hoàn hảo — pattern dùng nhiều nhất:**

```css
.center {
  display: flex;
  justify-content: center; /* Căn giữa ngang */
  align-items: center;     /* Căn giữa dọc */
}
```

### flex-wrap

Kiểm soát việc xuống dòng khi không đủ chỗ:

```css
.container {
  flex-wrap: nowrap;   /* Không xuống dòng — mặc định, có thể tràn */
  flex-wrap: wrap;     /* Xuống dòng khi cần */
  flex-wrap: wrap-reverse; /* Xuống dòng theo chiều ngược */
}
```

### gap

Khoảng cách giữa các flex items — thay thế cho margin giữa items:

```css
.container {
  display: flex;
  gap: 16px;           /* Khoảng cách đều cả hàng và cột */
  gap: 8px 16px;       /* row-gap: 8px, column-gap: 16px */
}
```

### flex (shorthand)

`flex` là shorthand của `flex-grow`, `flex-shrink`, `flex-basis`:

```css
.item {
  flex: 1;       /* flex-grow: 1, flex-shrink: 1, flex-basis: 0 */
                 /* Item chiếm phần không gian còn lại đều nhau */

  flex: 0 0 200px; /* Không co, không dãn, chiều rộng cố định 200px */

  flex: 2;       /* Chiếm gấp đôi so với item có flex: 1 */
}
```

```css
/* Ví dụ thực tế: Sidebar + Main content */
.layout {
  display: flex;
  gap: 24px;
}

.sidebar { flex: 0 0 240px; } /* Cố định 240px */
.main    { flex: 1; }          /* Chiếm toàn bộ phần còn lại */
```

---

## 6.3. Grid

CSS Grid là mô hình layout hai chiều — sắp xếp phần tử theo cả **hàng** và **cột** cùng lúc. Phù hợp cho layout tổng thể trang, card grid, dashboard.

### grid-template-columns

Định nghĩa số cột và kích thước mỗi cột:

```css
.grid {
  display: grid;

  /* 3 cột bằng nhau */
  grid-template-columns: 1fr 1fr 1fr;

  /* Shorthand với repeat() */
  grid-template-columns: repeat(3, 1fr);

  /* Sidebar cố định + nội dung linh hoạt */
  grid-template-columns: 240px 1fr;

  /* Tự động tạo cột với kích thước tối thiểu — responsive không cần media query */
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
}
```

### grid-template-rows

Định nghĩa chiều cao của các hàng:

```css
.page-layout {
  display: grid;
  grid-template-rows: 64px 1fr auto; /* Header cố định, nội dung dãn, footer tự động */
  min-height: 100vh;
}
```

### gap

Khoảng cách giữa các grid cells:

```css
.grid {
  gap: 24px;           /* Khoảng cách đều */
  gap: 16px 24px;      /* row-gap: 16px, column-gap: 24px */
  row-gap: 16px;
  column-gap: 24px;
}
```

### grid-column và grid-row

Xác định phần tử **chiếm bao nhiêu cột/hàng**:

```css
/* Grid 3 cột */
.grid { grid-template-columns: repeat(3, 1fr); }

/* Item chiếm toàn bộ chiều rộng (cả 3 cột) */
.full-width { grid-column: 1 / -1; }

/* Item chiếm 2 cột đầu */
.span-two { grid-column: 1 / 3; }

/* Hoặc dùng span */
.span-two { grid-column: span 2; }

/* Item chiếm 2 hàng */
.tall-item { grid-row: span 2; }
```

**Ví dụ layout trang hoàn chỉnh:**

```css
.page {
  display: grid;
  grid-template-columns: 240px 1fr;
  grid-template-rows: 64px 1fr auto;
  grid-template-areas:
    "header  header"
    "sidebar main"
    "footer  footer";
  min-height: 100vh;
  gap: 0;
}

.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main    { grid-area: main; }
.footer  { grid-area: footer; }
```

### So sánh Flexbox vs Grid

| | Flexbox | Grid |
|---|---|---|
| Chiều | Một chiều (hàng **hoặc** cột) | Hai chiều (hàng **và** cột) |
| Dùng cho | Component, navigation, căn giữa | Layout tổng thể, card grid, dashboard |
| Kiểm soát | Từ item ra ngoài | Từ container vào item |
| Responsive | Tốt với `flex-wrap` | Tốt với `auto-fill`, `minmax` |
| Dùng khi | Xếp items theo một hướng | Cần kiểm soát cả hàng lẫn cột |

---

## 6.4. Position

`position` xác định cách phần tử được định vị trong trang. Các thuộc tính `top`, `right`, `bottom`, `left` chỉ có tác dụng khi `position` khác `static`.

### static

Giá trị mặc định. Phần tử nằm theo luồng văn bản bình thường, `top/left/...` không có tác dụng.

### relative

Phần tử nằm theo luồng bình thường, nhưng có thể dịch chuyển **so với vị trí ban đầu của chính nó**. Không gian ban đầu vẫn được giữ lại — phần tử khác không lấp vào.

```css
/* Dịch chuyển 10px xuống dưới và sang phải so với vị trí gốc */
.box {
  position: relative;
  top: 10px;
  left: 10px;
}
```

> **Công dụng chính:** Làm **containing block** cho `absolute` con bên trong.

### absolute

Phần tử **thoát khỏi luồng** — không chiếm không gian trong layout. Định vị **so với ancestor gần nhất có `position` khác `static`**. Nếu không có, định vị so với `<html>`.

```css
/* Container làm reference point */
.card {
  position: relative; /* Làm containing block */
}

/* Badge định vị góc trên phải của card */
.badge {
  position: absolute;
  top: 8px;
  right: 8px;
}
```

### fixed

Phần tử thoát khỏi luồng và định vị **so với viewport**. Không di chuyển khi scroll — phù hợp cho header cố định, floating button, toast notification.

```css
.navbar {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  height: 64px;
  z-index: 100;
}

/* Tránh nội dung bị che bởi navbar cố định */
.page-content {
  padding-top: 64px;
}
```

### sticky

Phần tử hành xử như `relative` cho đến khi đến vị trí ngưỡng trong khi scroll — sau đó "dán" lại như `fixed`. Tự động bỏ dán khi ra khỏi container cha.

```css
/* Thead dính ở đầu khi scroll bảng dài */
thead th {
  position: sticky;
  top: 0;
  background: white;
  z-index: 1;
}

/* Sidebar dính khi scroll */
.sidebar {
  position: sticky;
  top: 80px; /* Khoảng cách từ đỉnh viewport */
  height: fit-content;
}
```

### z-index

`z-index` kiểm soát thứ tự xếp chồng theo trục Z (trục chiều sâu). Chỉ hoạt động với phần tử có `position` khác `static`.

```css
/* Thứ tự: modal > overlay > header > nội dung */
.content  { z-index: 0; }
.header   { position: fixed; z-index: 100; }
.overlay  { position: fixed; z-index: 200; }
.modal    { position: fixed; z-index: 300; }
.tooltip  { position: absolute; z-index: 400; }
```

> **Lưu ý:** `z-index` hoạt động trong **stacking context**. Một phần tử tạo stacking context mới khi có `position` + `z-index`, `transform`, `opacity < 1`, `filter`, v.v. — khiến `z-index` của các con chỉ so sánh trong phạm vi đó.

### So sánh các giá trị Position

| | static | relative | absolute | fixed | sticky |
|---|---|---|---|---|---|
| Trong luồng | Có | Có | Không | Không | Có (ban đầu) |
| Tham chiếu | — | Vị trí gốc | Ancestor có position | Viewport | Viewport (sau ngưỡng) |
| Scroll theo trang | Có | Có | Có | Không | Một phần |
| Dùng khi | Mặc định | Containing block, dịch nhẹ | Badge, dropdown, tooltip | Header, FAB, toast | Sidebar, sticky header bảng |

---

## 6.5. Overflow

`overflow` kiểm soát cách nội dung hiển thị khi vượt quá kích thước của container.

### overflow

```css
.box {
  overflow: visible; /* Nội dung tràn ra ngoài — mặc định */
  overflow: hidden;  /* Cắt nội dung thừa — thường dùng với border-radius */
  overflow: scroll;  /* Luôn hiển thị scrollbar */
  overflow: auto;    /* Hiển thị scrollbar khi cần — dùng nhiều nhất */

  /* Kiểm soát riêng từng trục */
  overflow-x: auto;
  overflow-y: hidden;
}
```

**Ứng dụng phổ biến:**

```css
/* Bảng dữ liệu có scroll ngang */
.table-container {
  overflow-x: auto;
  -webkit-overflow-scrolling: touch; /* Scroll mượt trên iOS */
}

/* Card ảnh với border-radius — cắt ảnh không bị tràn */
.image-card {
  border-radius: 12px;
  overflow: hidden;
}
```

### text-overflow

Kiểm soát hiển thị khi text dài hơn container. **Bắt buộc dùng kèm** `overflow: hidden` và `white-space: nowrap`.

```css
/* Hiển thị dấu "..." khi text bị cắt */
.truncate {
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
}

/* Cắt nhiều dòng — WebKit */
.clamp-3 {
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
```

### white-space

Kiểm soát cách trình duyệt xử lý khoảng trắng và xuống dòng trong text:

```css
.element {
  white-space: normal;   /* Xuống dòng khi cần — mặc định */
  white-space: nowrap;   /* Không xuống dòng — dùng với text-overflow */
  white-space: pre;      /* Giữ nguyên khoảng trắng và xuống dòng */
  white-space: pre-wrap; /* Giữ khoảng trắng, xuống dòng khi cần */
}
```

---

## 6.6. Responsive Design

Responsive Design là kỹ thuật xây dựng giao diện tự thích ứng với mọi kích thước màn hình — từ điện thoại đến màn hình lớn — mà không cần tạo trang riêng.

### Đơn vị đo lường

| Đơn vị | Tương đối với | Mô tả | Dùng khi |
|---|---|---|---|
| `px` | Không (tuyệt đối) | Pixel cố định | Border, shadow, icon size |
| `%` | Phần tử cha | Phần trăm chiều rộng/cao cha | Width layout, padding theo tỉ lệ |
| `rem` | Font-size của `<html>` | Thường 1rem = 16px | Font-size, spacing (margin, padding) |
| `em` | Font-size của phần tử hiện tại | Thay đổi theo ngữ cảnh | Ít dùng hơn rem |
| `vw` | 1% chiều rộng viewport | Thay đổi theo cửa sổ | Hero section, fluid typography |
| `vh` | 1% chiều cao viewport | Thay đổi theo cửa sổ | Full-screen layout, hero height |
| `dvh` | 1% chiều cao viewport động | Tính đúng với mobile browser bar | Thay `vh` trên mobile |

```css
/* Thực tế: Kết hợp đơn vị hợp lý */
.container {
  width: 100%;          /* Chiếm đầy cha */
  max-width: 1200px;    /* Giới hạn trên desktop */
  margin: 0 auto;       /* Căn giữa */
  padding: 0 1rem;      /* Spacing responsive */
}

.hero {
  height: 100dvh;       /* Toàn chiều cao viewport, đúng trên mobile */
}

/* Fluid typography — font tự scale theo viewport */
h1 {
  font-size: clamp(1.5rem, 4vw, 3rem); /* Tối thiểu 1.5rem, tối đa 3rem */
}
```

### rem vs px

```css
/* rem — người dùng có thể tăng font-size trình duyệt → UI scale theo */
.button {
  font-size: 1rem;    /* 16px mặc định, scale khi user thay đổi */
  padding: 0.75rem 1.5rem;
  border-radius: 0.5rem;
}

/* px — cứng nhắc, không scale theo cài đặt trình duyệt */
.border {
  border: 1px solid;  /* px hợp lý cho border mỏng */
}
```

> **Best practice:** Dùng `rem` cho font-size và spacing. Dùng `px` cho border, shadow, và các giá trị cần cố định chính xác.

### Media Query

Media Query cho phép áp dụng CSS khác nhau tùy theo điều kiện của thiết bị (chiều rộng màn hình, orientation, v.v.):

```css
/* Cú pháp cơ bản */
@media (max-width: 768px) {
  /* CSS áp dụng khi màn hình ≤ 768px */
}

@media (min-width: 768px) and (max-width: 1024px) {
  /* CSS áp dụng khi 768px ≤ màn hình ≤ 1024px */
}

/* Breakpoint phổ biến */
/* xs: < 480px  — điện thoại nhỏ */
/* sm: ≥ 480px  — điện thoại lớn */
/* md: ≥ 768px  — tablet */
/* lg: ≥ 1024px — laptop */
/* xl: ≥ 1280px — desktop */
```

### Mobile First

Mobile First là chiến lược viết CSS mặc định cho **màn hình nhỏ nhất** trước, rồi dùng `min-width` media query để thêm style cho màn hình lớn hơn. Đây là best practice hiện đại vì phần lớn traffic đến từ di động.

```css
/* Mobile First — CSS mặc định cho mobile */
.grid {
  display: grid;
  grid-template-columns: 1fr;   /* 1 cột trên mobile */
  gap: 16px;
}

/* Tablet trở lên */
@media (min-width: 768px) {
  .grid {
    grid-template-columns: repeat(2, 1fr); /* 2 cột */
  }
}

/* Desktop trở lên */
@media (min-width: 1024px) {
  .grid {
    grid-template-columns: repeat(3, 1fr); /* 3 cột */
    gap: 24px;
  }
}
```

**Mobile First vs Desktop First:**

| | Mobile First (`min-width`) | Desktop First (`max-width`) |
|---|---|---|
| Viết CSS cho | Mobile trước | Desktop trước |
| Override theo hướng | Lên (thêm style) | Xuống (ghi đè style) |
| Performance | Tốt hơn (mobile tải ít CSS hơn) | Kém hơn |
| Xu hướng | ✅ Best practice hiện đại | Ít dùng hơn |

### Responsive với Tailwind CSS

Trong các dự án Next.js hiện đại, Tailwind CSS là lựa chọn phổ biến — responsive được xử lý qua prefix `sm:`, `md:`, `lg:`, `xl:`:

```tsx
// Tailwind CSS — Mobile First theo mặc định
function ProductGrid() {
  return (
    <div className="
      grid
      grid-cols-1        /* mobile: 1 cột */
      sm:grid-cols-2     /* ≥640px: 2 cột */
      lg:grid-cols-3     /* ≥1024px: 3 cột */
      xl:grid-cols-4     /* ≥1280px: 4 cột */
      gap-4
      lg:gap-6
    ">
      {products.map((p) => <ProductCard key={p.id} product={p} />)}
    </div>
  );
}
```
