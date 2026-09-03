# CÁC KIỂU TYPESCRIPT THƯỜNG DÙNG

TypeScript kiểm tra kiểu ở thời điểm biên dịch và xóa toàn bộ annotation khi sinh JavaScript. Vì vậy, type giúp mô tả và bảo vệ hợp đồng trong code, nhưng không tự xác thực dữ liệu đến từ HTTP, database hay biến môi trường lúc runtime.

---

## 1. Suy luận kiểu và annotation

TypeScript thường tự suy luận kiểu từ giá trị ban đầu; chỉ thêm annotation khi nó làm rõ API, giới hạn giá trị hoặc khi compiler không thể suy ra.

```typescript
const port = 3000;             // 3000 (literal type, vì là const)
let retryCount = 0;            // number (có thể gán lại)
let serviceName: string;       // annotation cần thiết vì chưa có giá trị
serviceName = 'payment';

function add(a: number, b: number): number {
  return a + b;
}
```

Ưu tiên để TypeScript suy luận kiểu cục bộ; hãy khai báo rõ kiểu ở ranh giới như tham số hàm public, giá trị trả về quan trọng, DTO và cấu hình.

---

## 2. Kiểu nguyên thủy (primitive types)

| Kiểu | Giá trị mô tả | Ví dụ |
| --- | --- | --- |
| `string` | Chuỗi văn bản | `'done'`, `` `user-${id}` `` |
| `number` | Số thực dấu phẩy động IEEE 754 | `1`, `3.14`, `NaN` |
| `bigint` | Số nguyên lớn | `9007199254740993n` |
| `boolean` | Đúng/sai | `true`, `false` |
| `symbol` | Giá trị định danh duy nhất | `Symbol('id')` |
| `null` | Giá trị rỗng chủ ý | `null` |
| `undefined` | Chưa có giá trị | `undefined` |

```typescript
const email: string = 'dev@example.com';
const amount: number = 125_000;
const sequence: bigint = 9_007_199_254_740_993n;
const enabled: boolean = true;
const requestKey: symbol = Symbol('request');
```

`number` không phân biệt `int`/`float`. Không trộn `number` và `bigint` trong phép tính. Khi bật `strictNullChecks`, `null` và `undefined` chỉ gán được nếu kiểu đích cho phép chúng.

---

## 3. Literal types và `as const`

Literal type biểu diễn một giá trị cụ thể, hữu ích cho trạng thái, role và giá trị cấu hình cố định.

```typescript
type HttpMethod = 'GET' | 'POST' | 'PATCH' | 'DELETE';
type HttpStatus = 200 | 201 | 400 | 404 | 500;

function request(method: HttpMethod, path: string) {}
request('GET', '/users');
// request('PUT', '/users'); // lỗi
```

`as const` làm object/array trở thành readonly và giữ literal thay vì mở rộng sang `string` hoặc `number`.

```typescript
const roles = ['admin', 'member'] as const;
type Role = (typeof roles)[number]; // 'admin' | 'member'

const config = {
  environment: 'production',
  retries: 3,
} as const;
// config.environment có kiểu 'production', không phải string
```

---

## 4. Array, readonly array và tuple

Hai cách viết array tương đương: `string[]` và `Array<string>`. Cú pháp `Array<T>` tiện hơn cho generic phức tạp.

```typescript
const ids: string[] = ['u1', 'u2'];
const results: Array<Promise<number>> = [];
const immutableIds: readonly string[] = ['u1', 'u2'];
// immutableIds.push('u3'); // lỗi
```

**Tuple** xác định số phần tử, thứ tự và kiểu tại từng vị trí.

```typescript
type Paginated<T> = [items: T[], total: number];

const page: Paginated<string> = [['a', 'b'], 2];
const [items, total] = page;
```

Không dùng tuple cho object có quá nhiều trường: object có tên thuộc tính sẽ dễ đọc và dễ mở rộng hơn.

---

## 5. Object type, thuộc tính tùy chọn và index signature

```typescript
type User = {
  id: string;
  email: string;
  displayName?: string; // string | undefined khi đọc
  readonly createdAt: Date;
};

const user: User = {
  id: 'u_01',
  email: 'lan@example.com',
  createdAt: new Date(),
};
```

Thuộc tính `?` nghĩa là thuộc tính có thể không tồn tại. Nó khác với trường luôn tồn tại nhưng giá trị có thể `undefined`:

```typescript
type OptionalField = { name?: string };
type UndefinedField = { name: string | undefined };

const a: OptionalField = {};                 // hợp lệ
const b: UndefinedField = { name: undefined }; // bắt buộc có key name
```

Khi key không biết trước, dùng `Record` hoặc index signature:

```typescript
type Headers = Record<string, string>;
type Metrics = { [metricName: string]: number };

const headers: Headers = { 'content-type': 'application/json' };
```

Index signature quá rộng có thể làm mất kiểm tra key cụ thể; nếu tập key đã biết, ưu tiên union key hoặc `Record<K, V>`.

---

## 6. `type` và `interface`

Cả hai đều mô tả shape của object và có tính structural typing: object tương thích nếu có đủ cấu trúc yêu cầu, không cần cùng tên kiểu.

```typescript
interface CreateUserInput {
  email: string;
  password: string;
}

type UserId = string;
type UserWithRole = CreateUserInput & { role: 'admin' | 'member' };
```

| Nhu cầu | Ưu tiên |
| --- | --- |
| Object contract, nhất là public API/có thể mở rộng | `interface` |
| Union, tuple, mapped/conditional type, alias primitive | `type` |
| Kế thừa object type | Cả `extends` (`interface`) và `&` (`type`) |

`interface` hỗ trợ declaration merging; dùng có chủ đích, vì khai báo cùng tên ở nhiều nơi sẽ được gộp. `type` không thể khai báo lại cùng tên.

---

## 7. Union, narrowing và discriminated union

Union (`|`) có nghĩa một giá trị có thể thuộc một trong các kiểu. Trước khi dùng thuộc tính riêng, TypeScript yêu cầu **narrowing**.

```typescript
function formatId(value: string | number): string {
  if (typeof value === 'number') return value.toFixed(0);
  return value.toUpperCase();
}
```

Các cách narrowing phổ biến: `typeof`, `instanceof`, toán tử `in`, so sánh literal và type guard tự viết.

```typescript
type ApiResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: { code: string; message: string } };

function unwrap<T>(result: ApiResult<T>): T {
  if (result.ok) return result.data;
  throw new Error(result.error.message);
}
```

`ok` là **discriminant**. Cách thiết kế này an toàn hơn `{ data?: T; error?: Error }`, vì nó cấm trạng thái mơ hồ có cả hai hoặc không có cả hai.

---

## 8. Intersection (`&`) và cách kết hợp type

Intersection yêu cầu giá trị thỏa tất cả type được ghép.

```typescript
type HasId = { id: string };
type HasTimestamps = { createdAt: Date; updatedAt: Date };
type PersistedUser = HasId & HasTimestamps & { email: string };
```

Tránh ghép các thuộc tính cùng tên nhưng không tương thích; chúng có thể biến thành `never`.

```typescript
type Broken = { id: string } & { id: number };
// Broken['id'] là never
```

---

## 9. Kiểu hàm, overload và callback

```typescript
type Logger = (message: string, context?: Record<string, unknown>) => void;

type FindUser = (id: string) => Promise<User | null>;

function withRetry<T>(task: () => Promise<T>, retries = 3): Promise<T> {
  return task();
}
```

Tham số optional (`arg?: T`) và tham số mặc định đều được caller quyền bỏ qua. Khi hàm nhận callback, hãy khai báo kiểu callback để contract rõ ràng.

**Overload** dùng khi cùng tên hàm có các input/output liên hệ khác nhau:

```typescript
function toArray(value: string): string[];
function toArray(value: string[]): string[];
function toArray(value: string | string[]): string[] {
  return Array.isArray(value) ? value : [value];
}
```

Chỉ dùng overload nếu overload giúp caller suy ra return type tốt hơn union; nếu không, một signature union đơn giản hơn.

---

## 10. `any`, `unknown`, `never`, `void`, `object` và `{}`

| Kiểu | Ý nghĩa | Khi dùng |
| --- | --- | --- |
| `any` | Tắt gần như toàn bộ kiểm tra kiểu | Chỉ là lối thoát tạm thời khi migrate |
| `unknown` | Giá trị chưa biết, phải narrow trước khi dùng | Input từ JSON, `catch`, external API |
| `never` | Không có giá trị nào; hàm không return bình thường | Hàm luôn throw, exhaustive check |
| `void` | Hàm không dùng giá trị return | Callback side effect |
| `object` | Mọi giá trị không primitive | Hiếm khi là shape mong muốn |
| `{}` | Mọi giá trị trừ `null`/`undefined` | Thường không nên dùng làm “object rỗng” |

```typescript
function parseJson(text: string): unknown {
  return JSON.parse(text);
}

const payload = parseJson('{"id":"u1"}');
// payload.id; // lỗi: unknown phải được kiểm tra

function isUser(value: unknown): value is User {
  return typeof value === 'object' && value !== null
    && 'id' in value && 'email' in value;
}
```

Hãy ưu tiên `unknown` thay `any` tại mọi boundary không đáng tin. `unknown` buộc code xử lý kiểm tra runtime thay vì đưa lỗi sang chỗ khác.

```typescript
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${String(value)}`);
}

type Role = 'admin' | 'member';
function labelRole(role: Role): string {
  switch (role) {
    case 'admin': return 'Administrator';
    case 'member': return 'Member';
    default: return assertNever(role);
  }
}
```

---

## 11. Nullability: `null`, `undefined` và optional chaining

Khi bật `strictNullChecks`, hãy mô tả khả năng vắng mặt trong kiểu:

```typescript
async function findUser(id: string): Promise<User | null> {
  return null;
}

const user = await findUser('u1');
const name = user?.displayName ?? 'Anonymous';
```

- `?.` chỉ dừng truy cập khi vế trái là `null` hoặc `undefined`.
- `??` chỉ lấy fallback cho `null`/`undefined`; khác `||`, vốn fallback cả với `0`, `false`, chuỗi rỗng.
- Dùng `!` (non-null assertion) rất hạn chế: nó chỉ lừa compiler, không thêm kiểm tra runtime.

---

## 12. Enum và lựa chọn thay thế

```typescript
enum OrderStatus {
  Pending = 'PENDING',
  Paid = 'PAID',
  Cancelled = 'CANCELLED',
}
```

`enum` sinh mã JavaScript ở runtime (trừ `const enum` trong các điều kiện build phù hợp). Với nhiều codebase, union literal + `as const` đơn giản, dễ tương tác JSON và không phát sinh runtime object:

```typescript
const ORDER_STATUS = ['PENDING', 'PAID', 'CANCELLED'] as const;
type OrderStatus = (typeof ORDER_STATUS)[number];
```

---

## 13. Generic: tái sử dụng type mà không mất thông tin

Generic nhận type parameter, thường ký hiệu `T`, `K`, `V`. Nó bảo toàn mối quan hệ giữa input và output.

```typescript
type ApiResponse<T> = {
  data: T;
  meta: { requestId: string };
};

function first<T>(items: readonly T[]): T | undefined {
  return items[0];
}

const id = first(['u1', 'u2']); // string | undefined
```

Ràng buộc generic bằng `extends` khi code cần thuộc tính nhất định:

```typescript
function getId<T extends { id: string }>(value: T): string {
  return value.id;
}
```

`keyof` lấy union các key; `typeof` lấy type của một giá trị:

```typescript
function pick<T, K extends keyof T>(object: T, key: K): T[K] {
  return object[key];
}
```

---

## 14. Utility types dựng sẵn

| Utility type | Tác dụng | Ví dụ phổ biến |
| --- | --- | --- |
| `Partial<T>` | Toàn bộ field thành optional | DTO cập nhật |
| `Required<T>` | Toàn bộ field bắt buộc | Sau bước normalize |
| `Readonly<T>` | Field readonly | View model immutable |
| `Pick<T, K>` | Chọn một nhóm field | Public user response |
| `Omit<T, K>` | Bỏ một nhóm field | Bỏ `passwordHash` |
| `Record<K, V>` | Map từ key sang value | Cache, lookup table |
| `Exclude<U, M>` | Bỏ thành viên khỏi union | Loại role cấm |
| `Extract<U, M>` | Giữ thành viên phù hợp | Lọc event union |
| `NonNullable<T>` | Bỏ `null | undefined` | Giá trị đã validate |
| `ReturnType<F>` | Lấy return type hàm | Đồng bộ service contract |
| `Parameters<F>` | Lấy tuple tham số hàm | Wrapper/decorator |
| `Awaited<T>` | Mở Promise lồng nhau | Kiểu dữ liệu sau `await` |

```typescript
type UserEntity = User & { passwordHash: string };
type PublicUser = Omit<UserEntity, 'passwordHash'>;
type UpdateUserInput = Partial<Pick<User, 'email' | 'displayName'>>;
type UserById = Record<string, User>;
```

Lưu ý `Partial<T>` chỉ nông (shallow). Với object lồng nhau, cần mô tả DTO riêng hoặc mapped type đệ quy có kiểm soát.

---

## 15. Mapped type, conditional type và template literal type

Đây là các công cụ nâng cao nhưng thường gặp trong thư viện và codebase lớn.

```typescript
// Mapped type: biến đổi toàn bộ key
type Nullable<T> = { [K in keyof T]: T[K] | null };

// Conditional type: chọn type theo điều kiện
type IdOf<T> = T extends { id: infer I } ? I : never;

// Template literal type: ghép string type
type EventName = `user:${'created' | 'deleted'}`;
```

`infer` chỉ dùng trong conditional type để rút ra một phần type. Không cần ưu tiên các kỹ thuật này trước type object/union rõ ràng; chọn cách dễ đọc nhất đáp ứng contract.

---

## 16. Type assertion, `satisfies` và kiểm tra runtime

`as SomeType` chỉ nói với compiler “hãy tin tôi”; nó không chuyển đổi dữ liệu hoặc xác thực runtime.

```typescript
const raw = JSON.parse('{"id": 123}') as User; // nguy hiểm: id thực tế là number
```

Với cấu hình, `satisfies` kiểm tra shape mà vẫn giữ literal type của giá trị:

```typescript
type AppConfig = { port: number; environment: 'dev' | 'production' };

const appConfig = {
  port: 3000,
  environment: 'production',
} satisfies AppConfig;
```

Ở boundary HTTP/database/environment, dùng schema validator (ví dụ Zod, class-validator hoặc validator phù hợp) để kiểm tra dữ liệu runtime, rồi chuyển kết quả đã hợp lệ thành type đáng tin cậy.

---

## 17. Quy ước chọn type trong backend

- Bật `strict: true`, đặc biệt `strictNullChecks` và `noImplicitAny`.
- Khai báo input/output của controller, service public và repository; để local variable được suy luận khi rõ ràng.
- Dùng `unknown` cho JSON, `catch` error và dữ liệu bên thứ ba; narrow hoặc validate trước khi dùng.
- Mô hình hóa trạng thái có nhánh bằng discriminated union thay vì nhiều cờ boolean/optional field mơ hồ.
- Ưu tiên union literal + `as const` cho trạng thái cố định; chỉ dùng `enum` khi cần object runtime của enum.
- Tránh `any`, type assertion bừa bãi, `{}` và `object` khi bạn thực sự biết shape cần có.
- Không dùng type thay validation: TypeScript không thể bảo vệ dữ liệu ngoài runtime.

Mục tiêu của type không phải là thêm annotation ở mọi nơi, mà là làm các hợp đồng quan trọng chính xác, dễ đọc và khó bị dùng sai.
