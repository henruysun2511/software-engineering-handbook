# JAVASCRIPT CHUYÊN SÂU: HOISTING, SCOPE, TDZ, CLOSURE, `this` VÀ BẤT ĐỒNG BỘ

Đây là một chương giáo trình về các cơ chế cốt lõi dễ gây lỗi nhất trong JavaScript backend: vòng đời của biến, phạm vi truy cập, ngữ cảnh `this`, và cách dữ liệu/luồng thực thi đi qua callback, Promise, `async`/`await`.

Mỗi phần đi theo cùng một mạch: **khái niệm là gì → engine đang làm gì → ví dụ → lỗi thường gặp → quy tắc cần nhớ**. Không cần học thuộc tất cả; mục tiêu là có một mô hình tư duy để tự dự đoán code sẽ chạy thế nào.

## Mục tiêu sau chương này

Bạn có thể giải thích được vì sao một biến là `undefined` hoặc ném `ReferenceError`, vì sao callback trong vòng lặp in sai giá trị, vì sao method bị mất `this`, và vì sao một đoạn `await` nhìn tuần tự nhưng ứng dụng vẫn không bị đứng.

## Bản đồ tư duy trước khi bắt đầu

Hầu hết nội dung trong chương xoay quanh hai câu hỏi:

1. **Tên biến này được tìm ở đâu?** Câu trả lời đến từ scope, TDZ và closure.
2. **Đoạn code này chạy lúc nào?** Câu trả lời đến từ call stack, callback, Promise và event loop.

Hãy coi JavaScript engine như một nhân viên làm việc với một chồng việc (call stack). Khi gặp việc cần chờ như đọc file hay gọi API, engine giao việc đó cho runtime, tiếp tục xử lý chồng việc hiện tại, và chỉ quay lại callback/Promise khi kết quả đã sẵn sàng.

---

## 1. Execution context và lexical environment

### Khái niệm

Mỗi lần JavaScript bắt đầu chạy chương trình hoặc gọi một hàm, engine tạo một **execution context** (ngữ cảnh thực thi). Có thể hình dung đây là “bàn làm việc” của lần chạy đó: engine đặt lên bàn các biến, hàm, tham số, giá trị `this` và đường dẫn tới scope bên ngoài. Mỗi lần gọi hàm có một bàn làm việc riêng.

Một execution context thường được hiểu qua hai giai đoạn:

1. **Creation phase**: đăng ký khai báo biến/hàm vào môi trường thực thi.
2. **Execution phase**: chạy lần lượt các câu lệnh, gán giá trị và gọi hàm.

### Lexical scope: hàm được sinh ra ở đâu quan trọng hơn nó được gọi ở đâu

JavaScript xác định nơi một identifier được tìm kiếm dựa trên vị trí code được **khai báo**, không phải nơi hàm được gọi. Đây là **lexical scope**. Hãy đọc chữ “lexical” như “dựa vào vị trí trong mã nguồn”.

```javascript
const prefix = 'API';

function makeName(name) {
  return `${prefix}: ${name}`;
}

function run() {
  const prefix = 'LOCAL';
  console.log(makeName('users')); // API: users
}

run();
```

`makeName` được khai báo ở global scope nên nó nhìn thấy `prefix = 'API'`, dù được gọi bên trong `run`.

**Quy tắc nhớ:** scope được quyết định khi viết/khai báo hàm; `this` (trình bày ở phần sau) lại thường được quyết định khi gọi hàm. Đừng nhầm hai cơ chế này.

---

## 2. Hoisting: khai báo được xử lý trước khi chạy code

### Khái niệm

Khi mới học, nhiều người nghe “hoisting” là “JavaScript kéo biến lên đầu file”. Cách nói này tiện nhưng không chính xác. JavaScript **không đổi vị trí code**. Đúng hơn, trước khi thực hiện từng dòng lệnh, engine đã đi qua scope để tạo các **binding** (ô lưu trữ có tên) cho khai báo. Việc binding đã tồn tại hay chưa được khởi tạo tạo ra các kết quả khác nhau.

Hãy hình dung giai đoạn tạo context là lúc dán nhãn các ngăn tủ; giai đoạn chạy code là lúc đặt đồ vào từng ngăn. `var` được đặt sẵn giá trị `undefined`; function declaration có sẵn thân hàm; `let`/`const` có ngăn tủ nhưng bị khóa tạm thời.

| Khai báo | Có binding trước khi chạy? | Đọc trước dòng khai báo | Ghi chú |
| --- | --- | --- | --- |
| `var` | Có | `undefined` | Binding thuộc function/global scope |
| Function declaration | Có | Gọi được hàm | Toàn bộ thân hàm đã sẵn sàng |
| `let`, `const`, `class` | Có | Lỗi `ReferenceError` | Nằm trong TDZ cho đến khi được khởi tạo |
| Function expression / arrow | Theo biến chứa nó | Phụ thuộc `var`/`let`/`const` | Không hoist như function declaration |

```javascript
console.log(a); // undefined
var a = 1;

sayHello(); // Hello
function sayHello() {
  console.log('Hello');
}

console.log(user); // ReferenceError: Cannot access 'user' before initialization
const user = 'Lan';
```

### `var` và function scope

`var` không bị giới hạn bởi block (`if`, `for`, `while`), nên có thể làm rò rỉ biến ra ngoài block.

```javascript
if (true) {
  var status = 'ready';
  let token = 'secret';
}

console.log(status); // ready
console.log(token);  // ReferenceError
```

Với code mới, ưu tiên `const`; dùng `let` chỉ khi cần gán lại; gần như không dùng `var`.

### Khi nào hoisting thật sự làm bạn gặp lỗi?

Hoisting thường không phải thứ cần “tận dụng”; nó là điều cần hiểu để đọc lỗi. Nếu một giá trị bất ngờ là `undefined`, hãy kiểm tra xem bạn có đọc `var` trước dòng gán hay dùng function expression trước khi khởi tạo không. Cách viết khai báo trước, sử dụng sau sẽ làm code dễ hiểu nhất.

### Function declaration và function expression

```javascript
declared(); // hoạt động
function declared() {}

expressed(); // TypeError: expressed is not a function
var expressed = function () {};

arrow(); // ReferenceError (TDZ)
const arrow = () => {};
```

---

## 3. Scope và scope chain

### Khái niệm

**Scope** là vùng trong mã nguồn mà tại đó một tên biến có thể được nhìn thấy. Mỗi scope như một căn phòng: code trong phòng có thể nhìn đồ ở phòng đó và các phòng bao bên ngoài, nhưng không thể nhìn đồ trong phòng con hoặc phòng khác.

JavaScript có các phạm vi chính:

- **Global scope**: biến ngoài cùng của module/script.
- **Function scope**: biến trong hàm; `var`, `let`, `const` đều không thoát khỏi hàm.
- **Block scope**: `let`, `const`, `class`, `catch` binding chỉ tồn tại trong `{}`.
- **Module scope**: file ES module có scope riêng; biến top-level không tự động thành thuộc tính global.

Khi tìm một biến, engine tìm từ scope hiện tại rồi lần lượt ra scope ngoài theo **scope chain**. Nếu không có binding nào, truy cập biến gây `ReferenceError`.

```javascript
const app = 'billing';

function createRequest() {
  const requestId = 'req-01';

  function log(message) {
    console.log(`[${app}] [${requestId}] ${message}`);
  }

  log('started');
}
```

`log` tìm `message` ở local scope, sau đó `requestId`, rồi `app` ở scope ngoài. Chuỗi tìm kiếm hướng ra ngoài này gọi là **scope chain**. Engine dừng ngay ở binding đầu tiên tìm thấy.

### Shadowing

Binding trong scope gần hơn che (shadow) binding ở scope ngoài.

```javascript
const timeout = 5_000;

function callService() {
  const timeout = 1_000;
  return timeout;
}
```

Shadowing hợp lệ nhưng nên tránh khi cùng tên làm khó đọc. Không thể khai báo lại một `let`/`const` ngay trong cùng scope.

---

## 4. TDZ (Temporal Dead Zone)

### Khái niệm

**TDZ (Temporal Dead Zone)** là “vùng cấm truy cập tạm thời”: khoảng từ khi block bắt đầu đến ngay trước khi lệnh `let`/`const`/`class` được thực thi. Binding đã được engine tạo ra khi chuẩn bị context, nhưng chưa có giá trị hợp lệ và không được phép đọc hoặc ghi.

Vì vậy TDZ không có nghĩa biến không tồn tại. Biến tồn tại, nhưng cố tình bị khóa để lỗi được phát hiện sớm. Đây là khác biệt cốt lõi với `var`.

```javascript
{
  // TDZ của config bắt đầu tại đây
  console.log(config); // ReferenceError
  const config = { retries: 3 };
  // TDZ kết thúc tại đây
}
```

TDZ giúp phát hiện sớm các lỗi dùng biến trước khi khởi tạo, thay vì âm thầm nhận `undefined` như `var`.

Một lỗi hay gặp là shadowing kết hợp TDZ:

```javascript
const region = 'ap-southeast-1';

function buildConfig() {
  // `region` bên trong che biến bên ngoài cho toàn bộ block hàm.
  // Do nó đang ở TDZ, dòng sau ném ReferenceError.
  const region = region ?? 'local';
}
```

Đổi tên biến hoặc tham chiếu biến ngoài qua tên khác là cách sửa đúng.

**Quy tắc nhớ:** `let` và `const` cũng có hoisting theo nghĩa binding được tạo sớm, nhưng không có “hoisting hữu dụng” như `var` vì TDZ ngăn bạn dùng chúng trước lúc khai báo.

---

## 5. Closure: hàm nhớ lexical environment nơi nó được tạo

### Khái niệm

**Closure** xuất hiện khi một hàm vẫn giữ quyền truy cập các biến ở scope ngoài của nó, kể cả sau khi scope ngoài đã `return`. Nói đơn giản: hàm “nhớ” căn phòng nơi nó được sinh ra, không chỉ nhớ mã lệnh của chính nó. Closure là nền tảng của private state, factory function, middleware và callback bất đồng bộ.

Không phải closure sao chép giá trị tại thời điểm tạo hàm. Nó giữ liên kết tới binding; nếu binding thay đổi, lần gọi sau sẽ thấy giá trị mới.

```javascript
function createCounter() {
  let value = 0;

  return {
    increment() {
      value += 1;
      return value;
    },
    current() {
      return value;
    },
  };
}

const counter = createCounter();
counter.increment(); // 1
counter.current();   // 1
```

`value` không thể được truy cập trực tiếp từ bên ngoài, nhưng hai method vẫn dùng được nó nhờ closure.

Điều này giải thích vì sao `value` vẫn tồn tại sau khi `createCounter()` đã chạy xong: vẫn còn các hàm `increment` và `current` có thể truy cập nó.

### Closure trong vòng lặp

`var` có một binding dùng chung cho toàn bộ function; `let` tạo binding mới ở mỗi vòng lặp. Đây là nguyên nhân kinh điển của lỗi callback.

```javascript
for (var i = 0; i < 3; i += 1) {
  setTimeout(() => console.log(i), 0);
}
// 3, 3, 3

for (let j = 0; j < 3; j += 1) {
  setTimeout(() => console.log(j), 0);
}
// 0, 1, 2
```

### Cẩn thận bộ nhớ

Closure giữ tham chiếu đến các giá trị còn được dùng. Không nên vô tình đóng over object lớn trong listener, timer hoặc cache sống lâu. Khi không còn cần thiết, hủy listener/timer và tránh giữ reference tới request, response hoặc buffer lớn.

---

## 6. `this`: được quyết định bởi cách gọi hàm

### Khái niệm

`this` là một giá trị đặc biệt giúp hàm regular biết nó đang được gọi với “đối tượng nào”. Khác với biến trong lexical scope, `this` **không** được quyết định bằng vị trí khai báo hàm. Với **regular function**, nó chủ yếu được xác định tại thời điểm gọi (call site).

Đọc `this` đúng cách là nhìn vào phần ngay trước dấu chấm ở lúc gọi: trong `account.regular()`, `this` là `account`. Nếu tách hàm ra thành `const fn = account.regular; fn()`, dấu chấm biến mất nên context cũng biến mất.

| Cách gọi | Giá trị `this` |
| --- | --- |
| `obj.method()` | `obj` |
| `fn()` trong strict mode / ES module | `undefined` |
| `new Constructor()` | object instance mới |
| `fn.call(x)`, `fn.apply(x)` | `x` |
| `fn.bind(x)` | cố định là `x` |
| Arrow function | kế thừa `this` lexical từ scope ngoài |

```javascript
'use strict';

const account = {
  name: 'admin',
  regular() {
    return this.name;
  },
  arrow: () => this?.name,
};

account.regular(); // admin
account.arrow();   // undefined: arrow không nhận this từ account
```

### Mất context khi tách method

```javascript
const service = {
  name: 'EmailService',
  send() {
    console.log(this.name);
  },
};

const send = service.send;
send(); // TypeError trong strict mode: this là undefined
```

Sửa bằng `bind` khi truyền method làm callback:

```javascript
setTimeout(service.send.bind(service), 100);
```

### Arrow function phù hợp khi nào?

Arrow function không tự tạo `this`; nó lấy `this` từ scope bên ngoài tại lúc được tạo. Vì vậy arrow hữu ích để giữ `this` của method bao quanh, đặc biệt trong callback. Không dùng arrow làm method khi bạn muốn `this` là object gọi method, và không dùng arrow làm constructor (`new` không hoạt động).

```javascript
class Job {
  constructor(name) {
    this.name = name;
  }

  start() {
    setTimeout(() => console.log(this.name), 100); // giữ this của Job
  }
}
```

---

## 7. Callback: truyền hành vi cho một hàm khác

### Khái niệm

Callback là một hàm được truyền vào một hàm/API khác để API đó gọi lại tại thời điểm thích hợp. “Callback” không đồng nghĩa với “bất đồng bộ”: `array.map(callback)` gọi callback ngay và đồng bộ, còn callback của `readFile` chạy sau khi I/O hoàn tất.

Vấn đề của callback là quyền điều khiển được giao sang bên khác: caller phải tin API sẽ gọi callback đúng lúc, đúng một lần và truyền đúng lỗi/kết quả. Promise được sinh ra một phần để chuẩn hóa contract này.

```javascript
function validateUser(user, onSuccess, onError) {
  if (!user.email) return onError(new Error('Email is required'));
  onSuccess(user);
}
```

Node.js truyền thống dùng **error-first callback**: tham số đầu là `error`; khi thành công error là `null`.

```javascript
import { readFile } from 'node:fs';

readFile('config.json', 'utf8', (error, content) => {
  if (error) {
    console.error('Cannot read config', error);
    return;
  }
  console.log(JSON.parse(content));
});
```

### Callback hell và lỗi xử lý nhiều lần

Callback lồng sâu làm luồng lỗi khó theo dõi. Ngoài ra, một callback không đáng tin cậy có thể bị gọi hơn một lần. Với public API, nên quy định rõ callback được gọi một lần hoặc chuyển sang Promise.

```javascript
// Khó đọc, khó gom lỗi khi quy trình dài.
getUser(id, (error, user) => {
  if (error) return handleError(error);
  getOrders(user.id, (error, orders) => {
    if (error) return handleError(error);
    saveReport(orders, handleError);
  });
});
```

---

## 8. Promise: biểu diễn kết quả của công việc bất đồng bộ

### Khái niệm

Promise là một object đại diện cho **kết quả trong tương lai** của một công việc. Nó không nhất thiết là công việc đó; nó là “biên nhận” để code khác đăng ký nhận kết quả hoặc lỗi.

Promise có ba trạng thái: `pending` (đang chờ), `fulfilled` (thành công) và `rejected` (thất bại). Khi đã fulfilled hoặc rejected, Promise được **settled** và không đổi trạng thái nữa.

```javascript
function fetchUser(id) {
  return fetch(`/users/${id}`)
    .then((response) => {
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      return response.json();
    });
}

fetchUser('42')
  .then((user) => console.log(user))
  .catch((error) => console.error(error))
  .finally(() => console.log('request finished'));
```

### Chain và error propagation

- Giá trị `return` trong `.then()` trở thành giá trị của Promise tiếp theo.
- `throw` hoặc Promise rejected sẽ bỏ qua các `.then()` kế tiếp cho tới `.catch()` gần nhất.
- Luôn `return` Promise bên trong `.then()` để chain chờ đúng công việc.

Hãy coi mỗi `.then()` như một mắt xích tạo ra Promise mới. Khi quên `return`, bạn đã tách công việc thành một nhánh “fire-and-forget”; mắt xích phía ngoài không còn biết phải chờ nó.

```javascript
// Sai: processOrder kết thúc trước saveAudit.
function processOrder(order) {
  saveOrder(order).then(() => {
    saveAudit(order.id); // thiếu return
  });
}

// Đúng
function processOrder(order) {
  return saveOrder(order).then(() => saveAudit(order.id));
}
```

### Các combinator quan trọng

| API | Kết quả |
| --- | --- |
| `Promise.all(items)` | Thành công khi tất cả thành công; fail-fast khi một promise reject |
| `Promise.allSettled(items)` | Luôn trả kết quả từng promise, phù hợp batch độc lập |
| `Promise.race(items)` | Settled theo promise đầu tiên settled |
| `Promise.any(items)` | Thành công theo promise đầu tiên fulfilled; reject khi tất cả reject |

`Promise.all` tạo concurrency nhưng không tự hủy những công việc còn lại khi một task thất bại. Với `fetch`, cần `AbortController` nếu muốn hủy request.

---

## 9. `async` / `await`: cú pháp tuần tự trên nền Promise

### Khái niệm

`async`/`await` không thay thế Promise; nó là cú pháp giúp đọc code Promise theo thứ tự từ trên xuống. Hàm `async` luôn trả về Promise. `await value` chuyển `value` sang Promise bằng `Promise.resolve(value)`, rồi tạm dừng **riêng hàm async đó** cho đến khi Promise settled.

Điểm quan trọng: `await` không chặn toàn bộ JavaScript hay event loop. Nó chỉ tạm ngưng phần còn lại của hàm hiện tại và trả quyền điều khiển về runtime. Do đó server vẫn có thể tiếp tục xử lý request khác trong lúc chờ database/network, miễn là bạn không chạy CPU-bound code đồng bộ quá lâu.

```javascript
async function getProfile(id) {
  try {
    const response = await fetch(`/profiles/${id}`);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return await response.json();
  } catch (error) {
    // Có thể ghi log hoặc thêm ngữ cảnh, sau đó ném lại.
    throw new Error(`Cannot load profile ${id}`, { cause: error });
  }
}
```

### Tuần tự và song song

Chỉ `await` tuần tự khi task sau thực sự phụ thuộc task trước. Nếu độc lập, khởi tạo Promise trước rồi await cùng lúc để tránh tăng latency.

```javascript
// Tuần tự: tổng thời gian gần bằng A + B
const user = await getUser(id);
const permissions = await getPermissions(id);

// Song song: tổng thời gian gần bằng task chậm nhất
const [parallelUser, parallelPermissions] = await Promise.all([
  getUser(id),
  getPermissions(id),
]);
```

### Những lỗi thường gặp

- Quên `await`: biến nhận Promise thay vì dữ liệu.
- Gọi hàm async mà không `await`/`return`/`.catch()`: dễ tạo unhandled rejection.
- Dùng `await` trong `Array.prototype.forEach`: `forEach` không chờ callback async.

```javascript
// Sai: hàm ngoài không đợi save hoàn tất.
ids.forEach(async (id) => {
  await save(id);
});

// Tuần tự
for (const id of ids) {
  await save(id);
}

// Song song
await Promise.all(ids.map((id) => save(id)));
```

---

## 10. Microtask, callback và thứ tự thực thi

### Khái niệm

Call stack chỉ chạy một việc JavaScript tại một thời điểm. Khi stack rỗng, event loop chọn việc đã sẵn sàng trong các hàng đợi. Callback `.then()` và phần tiếp tục sau `await` được đưa vào **microtask queue**; callback timer/I/O thường đi qua **macrotask queue**. Engine phải làm cạn microtask queue trước khi lấy macrotask tiếp theo.

Đó là lý do Promise thường in trước `setTimeout(..., 0)`: `0` chỉ là thời gian tối thiểu để timer sẵn sàng, không phải lời hứa “chạy ngay lập tức”.

```javascript
console.log('1');

setTimeout(() => console.log('2: timeout'), 0);

Promise.resolve().then(() => console.log('3: promise'));

async function demo() {
  console.log('4: async start');
  await null;
  console.log('5: after await');
}

demo();
console.log('6');
// 1, 4, 6, 3, 5, 2
```

Không tạo vòng lặp vô hạn bằng microtask: liên tục `queueMicrotask` hoặc chain Promise có thể làm timer/I/O không có cơ hội chạy (microtask starvation).

---

## 11. Checklist thực chiến

- Dùng `const` mặc định, `let` khi cần gán lại; tránh `var`.
- Không dựa vào hoisting để đọc code “ngược”; khai báo trước khi dùng.
- Đặt tên khác nhau để tránh shadowing gây TDZ hoặc đọc nhầm scope.
- Khi truyền method làm callback, kiểm tra `this`; dùng arrow callback hoặc `.bind()` khi cần.
- Luôn `return`/`await` Promise để caller biết khi công việc kết thúc.
- Chọn `Promise.all` cho task độc lập cần thành công cùng nhau; chọn `allSettled` khi muốn thu toàn bộ kết quả.
- Bọc ranh giới I/O bằng `try/catch`, thêm context cho lỗi và không nuốt lỗi im lặng.
- Hạn chế closure/listener sống lâu giữ object lớn; giải phóng listener và hủy request khi phù hợp.

Nắm được các nguyên lý trên giúp giải thích phần lớn lỗi “biến là undefined”, “mất `this`”, “code chạy sai thứ tự” và “request kết thúc khi dữ liệu chưa được lưu” trong JavaScript backend. Khi debug, hãy lần lượt hỏi: giá trị này được khai báo ở scope nào, được khởi tạo lúc nào, callback giữ closure gì, `this` được gọi bằng cách nào, và Promise nào đang thực sự được `return` hoặc `await`.
