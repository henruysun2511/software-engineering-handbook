# Chương 1: JavaScript

JavaScript là ngôn ngữ lập trình thông dịch, đơn luồng, chạy trên trình duyệt và môi trường Node.js. Đây là nền tảng bắt buộc của mọi frontend developer.

---

## 1.1. Scope (Phạm vi biến)

Scope xác định vùng mà một biến có thể được truy cập. JavaScript có ba loại khai báo biến: `var`, `let`, và `const`, mỗi loại có hành vi scope khác nhau.

### var

- Có phạm vi **function scope** (chỉ bị giới hạn bởi hàm bao ngoài).
- Có thể khai báo lại trong cùng một scope.
- Bị **hoisting** — nghĩa là khai báo được đẩy lên đầu scope nhưng giá trị chưa được gán.

```js
function example() {
  console.log(x); // undefined (không lỗi)
  var x = 10;
  console.log(x); // 10
}
```

### let

- Có phạm vi **block scope** (giới hạn trong cặp `{}`).
- Không thể khai báo lại trong cùng một scope.
- Bị hoisting nhưng không khởi tạo → nằm trong **Temporal Dead Zone (TDZ)**.

### const

- Tương tự `let` về block scope và TDZ.
- Bắt buộc phải gán giá trị ngay khi khai báo.
- Không thể gán lại giá trị, nhưng **có thể thay đổi nội dung** của object hoặc array.

```js
const user = { name: "An" };
user.name = "Bình"; // Hợp lệ
user = {};          // Lỗi: Assignment to constant variable
```

### So sánh var / let / const

| Tiêu chí | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function | Block | Block |
| Hoisting | Có (undefined) | Có (TDZ) | Có (TDZ) |
| Khai báo lại | Được | Không | Không |
| Gán lại | Được | Được | Không |

> **Quy tắc thực tế:** Ưu tiên dùng `const`. Dùng `let` khi cần gán lại. Không dùng `var` trong code hiện đại.

---

### Hoisting

Hoisting là cơ chế JavaScript đưa phần **khai báo** biến và hàm lên đầu scope trước khi thực thi code.

```js
console.log(a); // undefined (không lỗi)
var a = 1;
```

**Giải thích:** JavaScript thực thi đoạn trên như sau:

```js
var a;          // Khai báo được đẩy lên đầu
console.log(a); // undefined — biến tồn tại nhưng chưa có giá trị
a = 1;          // Gán giá trị ở đúng vị trí ban đầu
```

Hoisting cũng áp dụng cho **function declaration** (khác với function expression):

```js
greet(); // "Hello" — hoạt động bình thường

function greet() {
  console.log("Hello");
}
```

---

### Temporal Dead Zone (TDZ)

TDZ là khoảng thời gian từ khi một biến `let`/`const` được hoisting đến khi dòng khai báo thực sự được thực thi. Trong khoảng này, truy cập biến sẽ ném ra lỗi `ReferenceError`.

```js
console.log(b); // ReferenceError: Cannot access 'b' before initialization
let b = 2;
```

TDZ giúp code dễ đoán hơn và tránh các lỗi ngầm từ `var`.

---

## 1.2. Closure

### Lexical Scope

Lexical scope có nghĩa là một hàm có thể truy cập các biến được định nghĩa trong phạm vi bao ngoài nó tại **thời điểm viết code**, không phải thời điểm gọi hàm.

```js
function outer() {
  const message = "Hello";

  function inner() {
    console.log(message); // Truy cập được biến của outer
  }

  inner();
}
```

### Closure là gì?

Closure là khả năng một hàm **ghi nhớ và truy cập** các biến từ scope bên ngoài ngay cả khi hàm bên ngoài đã kết thúc thực thi.

```js
function counter() {
  let count = 0;
  return () => ++count;
}

const increment = counter();
console.log(increment()); // 1
console.log(increment()); // 2
console.log(increment()); // 3
```

**Giải thích:** Hàm `counter` đã chạy xong, nhưng biến `count` vẫn được giữ sống bởi hàm con được trả về. Mỗi lần gọi `increment()`, biến `count` được cập nhật và giữ nguyên giữa các lần gọi.

### Closure dùng để làm gì?

**1. Tạo private state (ẩn dữ liệu nội bộ)**

```js
function createBankAccount(initialBalance) {
  let balance = initialBalance;

  return {
    deposit: (amount) => { balance += amount; },
    withdraw: (amount) => { balance -= amount; },
    getBalance: () => balance,
  };
}

const account = createBankAccount(1000);
account.deposit(500);
console.log(account.getBalance()); // 1500
// Không thể truy cập `balance` trực tiếp từ bên ngoài
```

**2. Tạo hàm có cấu hình sẵn (partial application)**

```js
function multiply(x) {
  return (y) => x * y;
}

const double = multiply(2);
const triple = multiply(3);

console.log(double(5)); // 10
console.log(triple(5)); // 15
```

---

## 1.3. `this`

`this` là từ khóa tham chiếu đến **ngữ cảnh (context)** mà hàm đang được thực thi. Giá trị của `this` phụ thuộc vào **cách hàm được gọi**, không phải nơi hàm được định nghĩa — ngoại trừ arrow function.

### `this` trong Function thông thường

```js
const user = {
  name: "An",
  greet() {
    console.log(this.name); // "An" — this là object user
  },
};

user.greet();
```

Nếu tách hàm ra khỏi object:

```js
const greet = user.greet;
greet(); // undefined hoặc lỗi (this là undefined trong strict mode)
```

### `this` trong Arrow Function

Arrow function **không có `this` riêng**. Nó kế thừa `this` từ scope bên ngoài tại thời điểm định nghĩa.

```js
const user = {
  name: "An",
  greet: () => {
    console.log(this.name); // undefined — this kế thừa từ module/window scope
  },
};
```

> Arrow function không nên được dùng làm method của object.

### bind, call, apply

Ba phương thức này cho phép **gán `this` thủ công** khi gọi hàm.

| Phương thức | Gọi ngay? | Truyền đối số |
|---|---|---|
| `call(ctx, a, b)` | Có | Từng giá trị |
| `apply(ctx, [a, b])` | Có | Mảng |
| `bind(ctx)` | Không (trả về hàm mới) | Từng giá trị |

```js
function greet(greeting) {
  console.log(`${greeting}, ${this.name}`);
}

const user = { name: "An" };

greet.call(user, "Xin chào");    // Xin chào, An
greet.apply(user, ["Xin chào"]); // Xin chào, An

const boundGreet = greet.bind(user);
boundGreet("Xin chào");           // Xin chào, An
```

---

## 1.4. Event Loop

JavaScript là ngôn ngữ **đơn luồng** — chỉ xử lý một tác vụ tại một thời điểm. Event Loop là cơ chế giúp JavaScript xử lý các tác vụ bất đồng bộ mà không bị blocking.

### Các thành phần

```
Call Stack  →  Web APIs  →  Task Queues  →  Call Stack
```

- **Call Stack:** Nơi thực thi code theo thứ tự, mỗi lần gọi hàm được đẩy vào stack.
- **Web APIs:** Môi trường trình duyệt xử lý các tác vụ async như `setTimeout`, `fetch`, event listener.
- **Microtask Queue:** Hàng đợi ưu tiên cao — chứa callback từ `Promise.then()`, `async/await`, `queueMicrotask()`. **Luôn chạy hết trước khi xử lý macrotask.**
- **Macrotask Queue (Task Queue):** Hàng đợi thông thường — chứa callback từ `setTimeout`, `setInterval`, I/O events.

### Thứ tự thực thi

1. Chạy hết code đồng bộ trong Call Stack.
2. Chạy hết tất cả Microtask.
3. Chạy một Macrotask.
4. Lặp lại từ bước 2.

### Ví dụ phân tích

```js
console.log(1);

setTimeout(() => console.log(2), 0);

Promise.resolve().then(() => console.log(3));

console.log(4);

// Kết quả: 1 → 4 → 3 → 2
```

**Giải thích:**
- `1` và `4`: Code đồng bộ, chạy ngay lập tức.
- `3`: Promise callback → vào Microtask Queue, chạy trước macrotask.
- `2`: setTimeout callback → vào Macrotask Queue, chạy sau cùng.

### async / await

`async/await` là cú pháp đường (syntactic sugar) của Promise, giúp code bất đồng bộ trông như đồng bộ.

```js
async function fetchUser() {
  try {
    const response = await fetch("/api/user");
    const data = await response.json();
    return data;
  } catch (error) {
    console.error("Lỗi:", error);
  }
}
```

> `await` chỉ dừng thực thi **trong hàm đó**, không block toàn bộ chương trình.

---

## 1.5. Promise

Promise là đối tượng đại diện cho kết quả của một tác vụ bất đồng bộ trong tương lai. Một Promise có ba trạng thái: **pending**, **fulfilled**, **rejected**.

### Các phương thức kết hợp Promise

| Phương thức | Mô tả | Khi nào dùng |
|---|---|---|
| `Promise.all()` | Chờ **tất cả** hoàn thành. Một cái lỗi → reject toàn bộ | Các request độc lập, cần tất cả |
| `Promise.race()` | Lấy kết quả của cái **hoàn thành đầu tiên** | Timeout pattern |
| `Promise.any()` | Lấy kết quả của cái **thành công đầu tiên**. Tất cả lỗi mới reject | Thử nhiều nguồn |
| `Promise.allSettled()` | Chờ **tất cả kết thúc** (kể cả lỗi), trả về trạng thái từng cái | Cần biết cái nào lỗi |

```js
const p1 = fetch("/api/users");
const p2 = fetch("/api/posts");
const p3 = fetch("/api/comments");

// Chờ cả ba — nếu một cái lỗi thì vào catch
const [users, posts, comments] = await Promise.all([p1, p2, p3]);

// Kiểm tra từng cái có thành công không
const results = await Promise.allSettled([p1, p2, p3]);
results.forEach((result) => {
  if (result.status === "fulfilled") {
    console.log("Thành công:", result.value);
  } else {
    console.log("Lỗi:", result.reason);
  }
});
```

---

## 1.6. Prototype & Prototype Chain

### Prototype là gì?

Mọi object trong JavaScript đều có một thuộc tính ẩn `[[Prototype]]`, trỏ đến một object khác gọi là **prototype**. Khi truy cập một thuộc tính không tồn tại trên object, JavaScript sẽ tự động tìm lên prototype của nó.

```
Đối tượng (object)
      ↓ [[Prototype]]
Prototype của object
      ↓ [[Prototype]]
Object.prototype
      ↓ [[Prototype]]
null
```

### Prototype Chain

Chuỗi prototype là quá trình JavaScript tra cứu thuộc tính/phương thức theo chiều từ object → prototype → prototype của prototype → ... cho đến khi gặp `null`.

```js
const animal = {
  eat() {
    console.log("Đang ăn...");
  },
};

const dog = Object.create(animal);
dog.bark = function () {
  console.log("Gâu gâu!");
};

dog.bark(); // "Gâu gâu!" — tìm thấy trên dog
dog.eat();  // "Đang ăn..." — không có trên dog, tìm lên animal
```

### Ứng dụng với Class (ES6)

`class` trong JavaScript thực chất là cú pháp đường phía trên prototype:

```js
class Animal {
  eat() {
    console.log("Đang ăn...");
  }
}

class Dog extends Animal {
  bark() {
    console.log("Gâu gâu!");
  }
}

const rex = new Dog();
rex.bark(); // "Gâu gâu!"
rex.eat();  // "Đang ăn..." — kế thừa qua prototype chain
```

> Dù dùng `class` hay `Object.create`, cơ chế hoạt động phía sau đều là prototype chain.

---

## 1.7. Memory (Bộ nhớ)

### Stack và Heap

JavaScript quản lý bộ nhớ qua hai vùng:

| | Stack | Heap |
|---|---|---|
| Chứa | Primitive values, tham chiếu | Objects, Arrays, Functions |
| Kích thước | Nhỏ, cố định | Lớn, động |
| Tốc độ | Nhanh | Chậm hơn |
| Quản lý | Tự động (LIFO) | Garbage Collector |

```js
// Primitive — lưu trên Stack
let a = 10;
let b = a; // b là bản sao độc lập
b = 20;
console.log(a); // 10 — không bị ảnh hưởng

// Object — lưu trên Heap, Stack chứa tham chiếu
let obj1 = { name: "An" };
let obj2 = obj1; // obj2 trỏ đến cùng địa chỉ Heap
obj2.name = "Bình";
console.log(obj1.name); // "Bình" — bị ảnh hưởng
```

### Garbage Collection

JavaScript tự động thu hồi bộ nhớ khi một object không còn được tham chiếu đến. Thuật toán phổ biến nhất là **Mark-and-Sweep**: đánh dấu tất cả object có thể truy cập từ root (global), sau đó giải phóng những cái không được đánh dấu.

### Memory Leak

Memory leak xảy ra khi bộ nhớ không còn cần thiết nhưng không được giải phóng, khiến ứng dụng ngốn RAM ngày càng nhiều.

**Các nguyên nhân phổ biến:**

**1. Event listener không được gỡ bỏ**

```js
// Sai — listener tích lũy mỗi lần hàm chạy
function setup() {
  document.addEventListener("click", handleClick);
}

// Đúng — gỡ bỏ khi không cần nữa
function teardown() {
  document.removeEventListener("click", handleClick);
}
```

**2. Closure giữ tham chiếu không cần thiết**

```js
function createLeak() {
  const largeData = new Array(1000000).fill("data");

  return function () {
    // largeData bị giữ trong closure dù không dùng
    console.log("hello");
  };
}
```

**3. Trong React — không cleanup useEffect**

```js
useEffect(() => {
  const interval = setInterval(fetchData, 5000);
  return () => clearInterval(interval); // Bắt buộc phải cleanup
}, []);
```
