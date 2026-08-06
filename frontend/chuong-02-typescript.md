# Chương 2: TypeScript

TypeScript là ngôn ngữ mở rộng của JavaScript, bổ sung hệ thống kiểu tĩnh (static type system). TypeScript được biên dịch về JavaScript trước khi chạy, giúp phát hiện lỗi ngay tại thời điểm viết code thay vì lúc runtime.

---

## 2.1. Basic Types

### interface

`interface` dùng để mô tả hình dạng (shape) của một object. Đây là cách phổ biến nhất để định nghĩa kiểu dữ liệu trong TypeScript.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age?: number; // Thuộc tính tùy chọn
}

const user: User = {
  id: 1,
  name: "An",
  email: "an@example.com",
};
```

`interface` hỗ trợ **extends** để kế thừa:

```typescript
interface Admin extends User {
  role: "admin" | "superadmin";
  permissions: string[];
}
```

### type

`type` (Type Alias) dùng để đặt tên cho bất kỳ kiểu dữ liệu nào — không chỉ object.

```typescript
type ID = string | number;

type Point = {
  x: number;
  y: number;
};

type Status = "active" | "inactive" | "banned";
```

`type` hỗ trợ **intersection** để kết hợp:

```typescript
type AdminUser = User & {
  role: string;
};
```

### interface vs type

| Tiêu chí | `interface` | `type` |
|---|---|---|
| Mô tả object | ✅ Tốt nhất | ✅ Được |
| Union / Intersection | ❌ | ✅ |
| Primitive, Tuple | ❌ | ✅ |
| Mở rộng | `extends` | `&` (intersection) |
| Declaration merging | ✅ (có thể khai báo lại để mở rộng) | ❌ |

> **Quy tắc thực tế:** Dùng `interface` cho object/class. Dùng `type` cho union, primitive alias, tuple, hoặc kiểu phức tạp.

---

## 2.2. Function Types

TypeScript cho phép định nghĩa kiểu đầy đủ cho hàm, bao gồm tham số và giá trị trả về.

```typescript
// Định nghĩa kiểu hàm bằng type alias
type MathOperation = (a: number, b: number) => number;

const add: MathOperation = (a, b) => a + b;
const subtract: MathOperation = (a, b) => a - b;
```

Định nghĩa hàm thông thường với kiểu rõ ràng:

```typescript
function formatCurrency(amount: number, currency: string = "VND"): string {
  return `${amount.toLocaleString()} ${currency}`;
}
```

Hàm có tham số tùy chọn và rest parameters:

```typescript
function log(message: string, ...tags: string[]): void {
  console.log(`[${tags.join(", ")}] ${message}`);
}

log("Server started", "info", "server"); // [info, server] Server started
```

---

## 2.3. Generic

### Generic là gì?

Generic cho phép viết hàm, class, hoặc interface hoạt động với **nhiều kiểu dữ liệu** mà vẫn đảm bảo type-safe. Thay vì dùng `any` làm mất thông tin kiểu, Generic giữ lại mối quan hệ giữa đầu vào và đầu ra.

```typescript
// Không dùng Generic — mất type information
function identity(value: any): any {
  return value;
}

// Dùng Generic — TypeScript biết kiểu trả về khớp với đầu vào
function identity<T>(value: T): T {
  return value;
}

const name = identity("An");    // TypeScript biết: string
const age  = identity(25);      // TypeScript biết: number
```

Ứng dụng thực tế — hàm lấy phần tử đầu của mảng:

```typescript
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

const firstUser = first([{ id: 1, name: "An" }]); // User | undefined
const firstNum  = first([1, 2, 3]);               // number | undefined
```

### Generic Constraint (extends)

Dùng `extends` để giới hạn kiểu Generic chỉ chấp nhận những kiểu có thuộc tính nhất định.

```typescript
interface HasId {
  id: number;
}

function findById<T extends HasId>(items: T[], id: number): T | undefined {
  return items.find((item) => item.id === id);
}

const users = [{ id: 1, name: "An" }, { id: 2, name: "Bình" }];
const found = findById(users, 1); // { id: 1, name: "An" }
```

Generic với nhiều tham số:

```typescript
function merge<T, U>(obj1: T, obj2: U): T & U {
  return { ...obj1, ...obj2 };
}

const merged = merge({ name: "An" }, { age: 25 });
// { name: string; age: number }
```

---

## 2.4. Utility Types

TypeScript cung cấp sẵn các Utility Types để biến đổi kiểu dữ liệu hiện có.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}
```

### Partial\<T>

Chuyển tất cả thuộc tính thành tùy chọn (`?`). Dùng khi cập nhật một phần dữ liệu.

```typescript
type UpdateUserDto = Partial<User>;
// { id?: number; name?: string; email?: string; age?: number }

function updateUser(id: number, data: Partial<User>): void {
  // data có thể chỉ có một số thuộc tính
}

updateUser(1, { name: "Bình" }); // Hợp lệ
```

### Required\<T>

Ngược lại với `Partial` — chuyển tất cả thuộc tính thành bắt buộc.

```typescript
interface Config {
  host?: string;
  port?: number;
}

type StrictConfig = Required<Config>;
// { host: string; port: number }
```

### Pick\<T, K>

Lấy một tập con các thuộc tính từ kiểu gốc.

```typescript
type UserPreview = Pick<User, "id" | "name">;
// { id: number; name: string }
```

### Omit\<T, K>

Loại bỏ một số thuộc tính khỏi kiểu gốc.

```typescript
type UserWithoutEmail = Omit<User, "email">;
// { id: number; name: string; age: number }

// Tạo DTO khi tạo mới (không có id vì chưa tồn tại)
type CreateUserDto = Omit<User, "id">;
```

### Record\<K, V>

Tạo kiểu object với key kiểu `K` và value kiểu `V`.

```typescript
type UserMap = Record<number, User>;
// { [key: number]: User }

const cache: Record<string, number> = {
  "user:1": 1,
  "user:2": 2,
};
```

### ReturnType\<T>

Lấy kiểu trả về của một hàm.

```typescript
function getUser() {
  return { id: 1, name: "An", email: "an@example.com" };
}

type UserReturn = ReturnType<typeof getUser>;
// { id: number; name: string; email: string }
```

### So sánh các Utility Types

| Utility | Mô tả | Ví dụ dùng khi |
|---|---|---|
| `Partial<T>` | Tất cả optional | PATCH request, form partial update |
| `Required<T>` | Tất cả required | Đảm bảo config đầy đủ |
| `Pick<T, K>` | Lấy một số field | Response preview, select columns |
| `Omit<T, K>` | Bỏ một số field | Create DTO (bỏ id), bỏ nhạy cảm |
| `Record<K, V>` | Dictionary / Map | Cache, lookup table |
| `ReturnType<T>` | Kiểu trả về của hàm | Tái dùng kiểu không export |

---

## 2.5. Advanced Types

### keyof

`keyof` trả về union của tất cả key (dưới dạng string literal) của một kiểu.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

type UserKeys = keyof User; // "id" | "name" | "email"

// Ứng dụng: hàm lấy giá trị theo key an toàn
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user: User = { id: 1, name: "An", email: "an@example.com" };
const name = getProperty(user, "name"); // string
// getProperty(user, "phone"); // Lỗi biên dịch
```

### typeof

`typeof` trong TypeScript (khác `typeof` trong JS runtime) dùng để lấy kiểu của một biến, hàm, hay object ở thời điểm biên dịch.

```typescript
const config = {
  api: "https://api.example.com",
  timeout: 5000,
  debug: true,
};

type Config = typeof config;
// { api: string; timeout: number; debug: boolean }

function processConfig(cfg: typeof config): void {
  console.log(cfg.api);
}
```

### Mapped Type

Mapped Type tạo kiểu mới bằng cách biến đổi từng thuộc tính của một kiểu có sẵn.

```typescript
// Tạo readonly version của bất kỳ kiểu nào
type Readonly<T> = {
  readonly [K in keyof T]: T[K];
};

// Tạo nullable version
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

type NullableUser = Nullable<User>;
// { id: number | null; name: string | null; email: string | null }
```

### Conditional Type

Conditional Type cho phép định nghĩa kiểu dựa trên điều kiện, tương tự toán tử ternary.

```typescript
type IsArray<T> = T extends any[] ? true : false;

type A = IsArray<string[]>; // true
type B = IsArray<string>;   // false

// Ứng dụng thực tế: unwrap kiểu từ Promise
type Awaited<T> = T extends Promise<infer U> ? U : T;

type Result = Awaited<Promise<User>>; // User
```

---

## 2.6. Type Narrowing

Type Narrowing là quá trình TypeScript **thu hẹp** kiểu của một biến trong một nhánh code cụ thể dựa trên các điều kiện kiểm tra.

### typeof guard

```typescript
function formatValue(value: string | number): string {
  if (typeof value === "string") {
    return value.toUpperCase(); // TypeScript biết: string
  }
  return value.toFixed(2);     // TypeScript biết: number
}
```

### in operator

Dùng `in` để kiểm tra sự tồn tại của một thuộc tính trong object.

```typescript
interface Cat {
  meow(): void;
}

interface Dog {
  bark(): void;
}

function makeSound(animal: Cat | Dog): void {
  if ("meow" in animal) {
    animal.meow(); // TypeScript biết: Cat
  } else {
    animal.bark(); // TypeScript biết: Dog
  }
}
```

### User-defined Type Guard

Khi cần kiểm tra phức tạp hơn, định nghĩa hàm guard với return type `value is T`.

```typescript
interface ApiError {
  code: number;
  message: string;
}

function isApiError(error: unknown): error is ApiError {
  return (
    typeof error === "object" &&
    error !== null &&
    "code" in error &&
    "message" in error
  );
}

async function fetchData() {
  try {
    const response = await fetch("/api/data");
    return await response.json();
  } catch (error) {
    if (isApiError(error)) {
      console.log(`Lỗi ${error.code}: ${error.message}`);
    }
  }
}
```

---

## 2.7. Union, Literal & Discriminated Union

### Union Type

Union Type cho phép một biến nhận một trong nhiều kiểu khác nhau.

```typescript
type ID = string | number;
type Status = "loading" | "success" | "error";

function processId(id: ID): string {
  return String(id);
}
```

### Literal Type

Literal Type giới hạn giá trị của biến chỉ vào một tập hợp cụ thể (thường dùng với string hoặc number).

```typescript
type Direction = "north" | "south" | "east" | "west";
type DiceRoll = 1 | 2 | 3 | 4 | 5 | 6;

function move(direction: Direction): void {
  console.log(`Moving ${direction}`);
}

move("north"); // Hợp lệ
// move("up"); // Lỗi biên dịch
```

### Discriminated Union

Discriminated Union là pattern dùng một thuộc tính chung (discriminant) để phân biệt các variant trong union. Đây là cách xử lý union type an toàn và có thể mở rộng nhất.

```typescript
interface LoadingState {
  status: "loading";
}

interface SuccessState {
  status: "success";
  data: User[];
}

interface ErrorState {
  status: "error";
  message: string;
}

type FetchState = LoadingState | SuccessState | ErrorState;

function renderState(state: FetchState): string {
  switch (state.status) {
    case "loading":
      return "Đang tải...";
    case "success":
      return `Có ${state.data.length} người dùng`;
    case "error":
      return `Lỗi: ${state.message}`;
  }
}
```

### as const

`as const` chuyển một object/array thành **readonly** và thu hẹp kiểu của từng giá trị về dạng literal (giá trị chính xác, không phải kiểu rộng).

```typescript
// Không dùng as const
const config = {
  role: "admin",    // Kiểu: string (rộng)
  maxRetry: 3,      // Kiểu: number (rộng)
};

// Dùng as const
const CONFIG = {
  role: "admin",    // Kiểu: "admin" (literal)
  maxRetry: 3,      // Kiểu: 3 (literal)
} as const;

// Ứng dụng phổ biến: định nghĩa constants
const ROUTES = {
  HOME: "/",
  ABOUT: "/about",
  PROFILE: "/profile",
} as const;

type Route = (typeof ROUTES)[keyof typeof ROUTES];
// "/" | "/about" | "/profile"
```

`as const` đặc biệt hữu ích khi định nghĩa các hằng số và muốn TypeScript hiểu chính xác giá trị thay vì kiểu rộng.
