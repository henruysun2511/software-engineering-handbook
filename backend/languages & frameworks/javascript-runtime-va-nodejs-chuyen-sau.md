# CƠ CHẾ HOẠT ĐỘNG CỦA JAVASCRIPT RUNTIME VÀ NODE.JS

## I. Đặt vấn đề

JavaScript là ngôn ngữ lập trình đơn luồng (single-threaded) — tại một thời điểm chỉ thực thi được một tác vụ. Tuy nhiên, trong thực tế, JavaScript vẫn có thể xử lý hàng nghìn kết nối mạng, đọc file, hẹn giờ... một cách "song song" mà không bị chặn (non-blocking). Nghịch lý này được giải quyết nhờ **JavaScript Runtime** — một môi trường bao quanh JS Engine, cung cấp các cơ chế bất đồng bộ (asynchronous).

Báo cáo này trình bày bản chất của JS Runtime nói chung, cơ chế Event Loop, và đặc thù riêng của Node.js — runtime cho phép JavaScript chạy ngoài trình duyệt.

---

## II. Bản chất của JavaScript Runtime

JS Runtime không phải là bản thân ngôn ngữ JavaScript, mà là một "hệ sinh thái" gồm nhiều thành phần phối hợp với nhau. Runtime khác nhau tùy môi trường (trình duyệt hay Node.js), nhưng đều có chung 4 thành phần cốt lõi:

| Thành phần | Vai trò |
| :--- | :--- |
| **JS Engine** | Đọc và thực thi mã JavaScript (chứa Call Stack + Heap) |
| **Web APIs / Node APIs** | Cung cấp các chức năng bất đồng bộ mà bản thân JS Engine không có (`setTimeout`, `fetch`, `fs`, `network`...) |
| **Callback Queue (Macrotask Queue)** | Hàng đợi chứa các callback đã sẵn sàng chạy sau khi tác vụ bất đồng bộ hoàn tất |
| **Event Loop** | Cơ chế điều phối, liên tục kiểm tra Call Stack và các hàng đợi để quyết định code nào được thực thi tiếp theo |

### 1. JS Engine — Call Stack và Heap
- **Call Stack**: Cấu trúc dữ liệu dạng ngăn xếp (LIFO), lưu lại các "khung thực thi" (execution context) của hàm đang chạy. Mỗi khi một hàm được gọi, nó được đẩy (push) lên đỉnh stack; khi hàm kết thúc, nó bị lấy ra (pop). Vì chỉ có một Call Stack nên JS chỉ chạy được một dòng lệnh tại một thời điểm — đây chính là lý do JS "đơn luồng".
- **Heap**: Vùng nhớ không có cấu trúc thứ tự, dùng để cấp phát bộ nhớ cho object, array, closure...
- Nếu một hàm chạy quá lâu trên Call Stack (ví dụ vòng lặp nặng), toàn bộ chương trình bị "đứng" (blocking) vì không có gì khác được xử lý — đây là lý do JS cần các API bất đồng bộ để không làm nghẽn luồng chính.

### 2. Web APIs / Node APIs
Đây là các API không thuộc về ngôn ngữ JavaScript, mà do môi trường runtime cung cấp (trình duyệt hoặc Node.js), ví dụ: `setTimeout`, `fetch`, xử lý sự kiện DOM (trình duyệt), hay `fs.readFile`, giao tiếp mạng, timer (Node.js). Khi JS Engine gọi các API này, tác vụ được "giao" cho runtime xử lý ở bên ngoài Call Stack, giúp luồng chính rảnh để chạy tiếp code khác.

### 3. Callback Queue và Microtask Queue
Khi một tác vụ bất đồng bộ hoàn tất (ví dụ hết thời gian `setTimeout`, nhận được response mạng), callback tương ứng không được thực thi ngay mà được đưa vào hàng đợi:
- **Macrotask Queue (Callback Queue)**: Chứa callback của `setTimeout`, `setInterval`, sự kiện I/O, sự kiện DOM...
- **Microtask Queue**: Chứa callback của `Promise.then/catch/finally`, `queueMicrotask`, và (trong Node.js) `process.nextTick` (ưu tiên cao nhất).
- **Nguyên tắc ưu tiên**: Sau mỗi lần Call Stack rỗng, Event Loop luôn xử lý toàn bộ Microtask Queue trước, rồi mới lấy một tác vụ từ Macrotask Queue.

### 4. Event Loop — "trái tim" của cơ chế bất đồng bộ
Event Loop là một vòng lặp vô hạn, liên tục thực hiện tuần tự:
1. Kiểm tra Call Stack có rỗng không.
2. Nếu rỗng $\rightarrow$ lấy hết các tác vụ trong Microtask Queue ra chạy.
3. Sau đó lấy một tác vụ đầu tiên trong Macrotask Queue đưa vào Call Stack để thực thi.
4. Lặp lại từ bước 1.

**Sơ đồ luồng tổng quát:**

```text
Call Stack rỗng?
        │
        ▼ (có)
 Chạy hết Microtask Queue (Promise, process.nextTick)
        │
        ▼
 Lấy 1 tác vụ từ Macrotask Queue (setTimeout, I/O callback...)
        │
        ▼
 Đưa vào Call Stack thực thi
        │
        └────────────► quay lại bước đầu
```

### 5. Ví dụ minh họa thứ tự thực thi

```javascript
console.log('1. Bắt đầu');

setTimeout(() => console.log('2. setTimeout (macrotask)'), 0);

Promise.resolve().then(() => console.log('3. Promise (microtask)'));

console.log('4. Kết thúc đồng bộ');
```

- **Kết quả**: `1 → 4 → 3 → 2`
- **Giải thích**: `console.log` đồng bộ chạy ngay trên Call Stack (1, 4). `setTimeout` đưa callback vào Macrotask Queue dù thời gian chờ là 0ms. `Promise.then` đưa callback vào Microtask Queue. Khi Call Stack rỗng, Event Loop ưu tiên xử lý Microtask Queue (in ra 3) trước khi lấy Macrotask Queue (in ra 2).

---

## III. Node.js Runtime — đặc thù riêng ngoài trình duyệt

Node.js là một runtime cho phép chạy JavaScript bên ngoài trình duyệt, chủ yếu ở phía server. Kiến trúc gồm 3 lớp chính:

```text
┌─────────────────────────────────────┐
│           Mã JavaScript              │
├─────────────────────────────────────┤
│   Node.js APIs (fs, http, net...)    │  ← lớp binding C++
├──────────────────┬────────────────────┤
│    V8 Engine     │      libuv         │
│ (thực thi JS)    │ (Event Loop, I/O,  │
│                  │  Thread Pool)      │
└──────────────────┴────────────────────┘
```

- **V8**: JS Engine của Google (cũng dùng trong Chrome), chịu trách nhiệm biên dịch và thực thi JavaScript, quản lý Call Stack và Heap.
- **libuv**: Thư viện C viết riêng để cung cấp Event Loop, xử lý I/O bất đồng bộ, và một Thread Pool (mặc định 4 luồng) cho các tác vụ nặng (đọc/ghi file, nén dữ liệu, mã hóa, DNS lookup...) — những việc mà hệ điều hành không hỗ trợ bất đồng bộ ở tầng OS.
- **Node.js Bindings**: Lớp trung gian bằng C++ kết nối API JavaScript (`fs`, `http`, `crypto`...) với các chức năng cấp thấp của libuv/OS.

### Event Loop trong Node.js — 6 giai đoạn (phase)
Không giống mô hình đơn giản trên trình duyệt, Event Loop của Node.js (do libuv quản lý) chia thành 6 giai đoạn, chạy tuần hoàn theo thứ tự:

| Giai đoạn | Nhiệm vụ |
| :--- | :--- |
| **1. Timers** | Thực thi callback của `setTimeout`, `setInterval` đã đến hạn |
| **2. Pending callbacks** | Thực thi một số callback I/O bị hoãn từ vòng lặp trước |
| **3. Idle, prepare** | Nội bộ Node.js sử dụng, không can thiệp |
| **4. Poll** | Lấy các sự kiện I/O mới (đọc file, network...); nếu không có timer nào sắp chạy, giai đoạn này có thể "chờ" tại đây |
| **5. Check** | Thực thi callback của `setImmediate()` |
| **6. Close callbacks** | Xử lý sự kiện đóng, ví dụ `socket.on('close')` |

Giữa mỗi giai đoạn, Node.js đều xử lý hết Microtask Queue (`Promise`) và đặc biệt là `process.nextTick` Queue — hàng đợi có độ ưu tiên cao nhất, được xử lý trước cả Promise microtask.

### Ví dụ minh họa đặc thù Node.js

```javascript
console.log('Bắt đầu');

setTimeout(() => console.log('setTimeout'), 0);
setImmediate(() => console.log('setImmediate'));

process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('Promise'));

console.log('Kết thúc');
```

- **Kết quả (thường gặp)**: `Bắt đầu → Kết thúc → nextTick → Promise → setTimeout → setImmediate`
- **Giải thích**: `process.nextTick` luôn được ưu tiên chạy trước Promise microtask; cả hai chạy trước khi Event Loop bước vào giai đoạn Timers (`setTimeout`) hay Check (`setImmediate`). Thứ tự giữa `setTimeout(0)` và `setImmediate` khi gọi ở top-level không được đảm bảo tuyệt đối, phụ thuộc hiệu năng hệ thống — nhưng trong I/O callback thì `setImmediate` luôn chạy trước `setTimeout`.

---

## IV. So sánh Runtime trình duyệt và Node.js

| Tiêu chí | Trình duyệt | Node.js |
| :--- | :--- | :--- |
| **JS Engine** | V8 (Chrome), SpiderMonkey (Firefox)... | V8 |
| **API bất đồng bộ** | Web APIs (DOM, fetch, timer) | Node APIs (fs, net, http, timer) |
| **Xử lý I/O nặng** | Do trình duyệt/OS quản lý | Thread Pool của libuv |
| **Cấu trúc Event Loop** | Đơn giản: Macrotask $\leftrightarrow$ Microtask | 6 giai đoạn cụ thể (libuv) |
| **Hàng đợi ưu tiên đặc biệt** | Không có | `process.nextTick` (ưu tiên tuyệt đối) |
| **Mục tiêu** | Tương tác giao diện người dùng | Xử lý server, I/O tập trung |

---

## V. Kết luận

JavaScript Runtime là lớp hạ tầng giúp một ngôn ngữ đơn luồng như JavaScript vẫn có khả năng xử lý bất đồng bộ hiệu quả, thông qua sự phối hợp giữa Call Stack, API ngoài (Web/Node APIs), hàng đợi callback (Macrotask/Microtask) và Event Loop. Node.js kế thừa nguyên lý này nhưng mở rộng thêm libuv với Event Loop 6 giai đoạn và Thread Pool, cho phép JavaScript vượt ra khỏi trình duyệt để xử lý các tác vụ I/O ở phía server một cách hiệu quả, không chặn luồng chính (non-blocking I/O) — đây chính là yếu tố cốt lõi làm nên hiệu năng của Node.js trong các hệ thống có lượng kết nối đồng thời lớn.
