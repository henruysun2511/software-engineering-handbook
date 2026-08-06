# Chương 4: React

React là thư viện JavaScript dùng để xây dựng giao diện người dùng, phát triển bởi Meta. React tập trung vào một triết lý cốt lõi: **UI là hàm của state** — khi state thay đổi, UI tự động cập nhật.

---

## 4.1. React Fundamentals

### Virtual DOM

DOM thật (Real DOM) chậm vì mỗi thao tác thay đổi đều có thể kích hoạt reflow và repaint tốn kém (xem Chương 3). React giải quyết vấn đề này bằng **Virtual DOM** — một bản sao nhẹ của DOM thật, được lưu trong bộ nhớ JavaScript.

Thay vì cập nhật DOM thật trực tiếp, React:
1. Tạo Virtual DOM mới khi state thay đổi.
2. So sánh Virtual DOM mới với Virtual DOM cũ (**Diffing**).
3. Chỉ cập nhật đúng những phần thay đổi lên DOM thật (**Reconciliation**).

```
State thay đổi
      ↓
Virtual DOM mới được tạo
      ↓
Diffing (so sánh cũ vs mới)
      ↓
Tìm ra danh sách thay đổi (patch)
      ↓
Chỉ cập nhật phần đó lên Real DOM
```

### Diffing

Diffing là thuật toán React dùng để so sánh hai cây Virtual DOM. Để tối ưu, React áp dụng hai giả định:

1. **Hai phần tử khác type** → xóa cây cũ, tạo cây mới hoàn toàn.
2. **Phần tử có `key`** → dùng `key` để nhận dạng phần tử qua các lần render.

### Reconciliation

Reconciliation là quá trình áp dụng kết quả Diffing lên DOM thật — thêm, xóa, hoặc cập nhật đúng những node cần thiết. Thuật toán này được triển khai trong **React Fiber** (từ React 16 trở đi), cho phép chia nhỏ công việc, ưu tiên task quan trọng, và không block main thread.

### Component

Component là đơn vị xây dựng UI cơ bản trong React. Một component là một hàm nhận **props** và trả về **JSX**.

```tsx
// Function Component — cách viết chuẩn hiện đại
interface WelcomeProps {
  name: string;
}

function Welcome({ name }: WelcomeProps) {
  return <h1>Xin chào, {name}!</h1>;
}
```

### Props

Props (Properties) là dữ liệu được truyền từ component cha xuống component con. Props là **read-only** — component con không được sửa props.

```tsx
interface ButtonProps {
  label: string;
  variant?: "primary" | "secondary";
  onClick: () => void;
}

function Button({ label, variant = "primary", onClick }: ButtonProps) {
  return (
    <button className={`btn btn--${variant}`} onClick={onClick}>
      {label}
    </button>
  );
}

// Sử dụng
<Button label="Lưu" variant="primary" onClick={handleSave} />
```

### State

State là dữ liệu nội bộ của component. Khi state thay đổi, React tự động re-render component đó và các component con phụ thuộc vào nó.

```tsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Đếm: {count}</p>
      <button onClick={() => setCount((prev) => prev + 1)}>Tăng</button>
    </div>
  );
}
```

### One-way Data Flow

React áp dụng luồng dữ liệu một chiều: dữ liệu chỉ đi **từ cha xuống con** qua props. Con không thể trực tiếp thay đổi state của cha — thay vào đó, cha truyền một hàm callback xuống để con gọi khi cần.

```
App (state: users)
  ↓ props: users
UserList
  ↓ props: user
UserCard
```

Luồng này giúp ứng dụng dễ debug và dễ dự đoán hơn so với two-way binding.

---

## 4.2. Rendering

### React Rendering hoạt động như thế nào?

Quá trình React đưa UI lên màn hình gồm hai phase:

**Render Phase:** React gọi hàm component để tạo Virtual DOM mới, sau đó chạy thuật toán Diffing. Phase này là **pure** — không có side effect, không chạm vào DOM thật.

**Commit Phase:** React áp dụng kết quả Diffing lên DOM thật. Đây là lúc DOM thực sự thay đổi, sau đó React chạy `useEffect`.

### Khi nào React render lần đầu?

- Khi component được mount (lần đầu xuất hiện trong cây component).

### Khi nào React re-render?

React re-render một component khi:

| Nguyên nhân | Ví dụ |
|---|---|
| **State thay đổi** | `setState(newValue)` |
| **Props thay đổi** | Component cha truyền giá trị props mới |
| **Context thay đổi** | Giá trị Provider thay đổi |
| **Component cha re-render** | Mặc định, con re-render theo cha |

> **Lưu ý quan trọng:** React re-render không có nghĩa là DOM thật luôn thay đổi. Nếu kết quả render giống hệt lần trước, React sẽ không cập nhật DOM (nhờ Diffing).

---

## 4.3. Hooks

Hooks là các hàm đặc biệt cho phép function component sử dụng state, lifecycle, và các tính năng React khác. Hooks phải được gọi ở **top level** của component, không được gọi bên trong vòng lặp, điều kiện, hay hàm lồng nhau.

---

### 4.3.1. useState

`useState` là hook cơ bản nhất để quản lý state cục bộ trong component.

```tsx
const [state, setState] = useState(initialValue);
```

```tsx
import { useState } from "react";

interface User {
  name: string;
  email: string;
}

function ProfileForm() {
  const [user, setUser] = useState<User>({ name: "", email: "" });
  const [isLoading, setIsLoading] = useState(false);

  const handleNameChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    // Dùng functional update khi state mới phụ thuộc state cũ
    setUser((prev) => ({ ...prev, name: e.target.value }));
  };

  return (
    <input value={user.name} onChange={handleNameChange} />
  );
}
```

> **Quy tắc:** Khi giá trị state mới phụ thuộc vào giá trị cũ, luôn dùng dạng callback `setState(prev => ...)` để đảm bảo tính chính xác trong concurrent mode.

---

### 4.3.2. useEffect

`useEffect` dùng để thực hiện **side effects** — các tác vụ không thuần túy như gọi API, đăng ký event listener, hay thao tác DOM.

```tsx
useEffect(() => {
  // Side effect chạy sau render
  return () => {
    // Cleanup chạy trước khi effect chạy lại hoặc component unmount
  };
}, [dependencies]);
```

#### Dependency Array

| Dependency | Hành vi |
|---|---|
| Không truyền | Chạy sau **mỗi** lần render |
| `[]` (mảng rỗng) | Chỉ chạy **một lần** sau mount |
| `[a, b]` | Chạy lại khi `a` hoặc `b` thay đổi |

#### Cleanup

Cleanup function được trả về từ `useEffect`, chạy trước khi effect chạy lại hoặc khi component unmount. Đây là cách ngăn memory leak.

```tsx
useEffect(() => {
  const controller = new AbortController();

  fetch("/api/users", { signal: controller.signal })
    .then((res) => res.json())
    .then(setUsers)
    .catch((err) => {
      if (err.name !== "AbortError") console.error(err);
    });

  return () => controller.abort(); // Hủy request khi component unmount
}, []);
```

Ví dụ đăng ký / hủy event:

```tsx
useEffect(() => {
  function handleResize() {
    setWidth(window.innerWidth);
  }

  window.addEventListener("resize", handleResize);
  return () => window.removeEventListener("resize", handleResize);
}, []);
```

#### Infinite Loop

Vòng lặp vô hạn xảy ra khi dependency thay đổi sau mỗi render, kích hoạt effect, effect lại thay đổi state, state lại gây render...

```tsx
// Sai — object/array mới tạo mỗi render → dependency thay đổi mỗi lần
useEffect(() => {
  fetchData(options);
}, [{ page: 1 }]); // Object literal tạo mới mỗi render!

// Sai — setState trong effect mà không có guard
useEffect(() => {
  setCount(count + 1); // count thay đổi → effect chạy lại → vô hạn
}, [count]);

// Đúng — dùng primitive hoặc useMemo cho dependency
const options = useMemo(() => ({ page: 1 }), []);
useEffect(() => {
  fetchData(options);
}, [options]);
```

---

### 4.3.3. useMemo

`useMemo` cache kết quả của một **tính toán tốn kém**, chỉ tính lại khi dependency thay đổi.

```tsx
const memoizedValue = useMemo(() => expensiveComputation(a, b), [a, b]);
```

#### Khi nào nên dùng?

- Tính toán phức tạp (lọc/sort mảng lớn, tính toán nặng).
- Tạo object/array được truyền làm prop hoặc dependency của hook khác.

```tsx
function ProductList({ products, categoryFilter }: Props) {
  // Chỉ lọc lại khi products hoặc categoryFilter thay đổi
  const filtered = useMemo(
    () => products.filter((p) => p.category === categoryFilter),
    [products, categoryFilter]
  );

  return (
    <ul>
      {filtered.map((p) => <li key={p.id}>{p.name}</li>)}
    </ul>
  );
}
```

> **Không nên lạm dụng:** `useMemo` có chi phí riêng (lưu trữ, so sánh dependency). Chỉ dùng khi có vấn đề hiệu suất thực sự.

---

### 4.3.4. useCallback

`useCallback` cache một **hàm**, trả về cùng một reference hàm giữa các lần render nếu dependency không đổi.

```tsx
const memoizedFn = useCallback(() => doSomething(a, b), [a, b]);
```

#### useMemo vs useCallback

| | `useMemo` | `useCallback` |
|---|---|---|
| Cache | **Giá trị** trả về của hàm | **Hàm** (reference) |
| Dùng khi | Tính toán tốn kém | Hàm truyền xuống component con hoặc làm dependency |
| Tương đương | `useMemo(() => fn, deps)` | `useMemo(() => () => fn(), deps)` |

```tsx
function Parent() {
  const [count, setCount] = useState(0);

  // Không có useCallback — hàm mới tạo mỗi render
  // → Child re-render dù không cần thiết (nếu Child dùng React.memo)
  const handleClick = useCallback(() => {
    console.log("Clicked");
  }, []); // Reference ổn định

  return <Child onClick={handleClick} />;
}

const Child = React.memo(({ onClick }: { onClick: () => void }) => {
  console.log("Child render");
  return <button onClick={onClick}>Click</button>;
});
```

---

### 4.3.5. useRef

`useRef` tạo một object `{ current: value }` tồn tại suốt vòng đời component. Thay đổi `.current` **không gây re-render**.

**Hai mục đích chính:**

**1. Truy cập DOM node trực tiếp:**

```tsx
function FocusInput() {
  const inputRef = useRef<HTMLInputElement>(null);

  function focusInput() {
    inputRef.current?.focus();
  }

  return (
    <>
      <input ref={inputRef} type="text" />
      <button onClick={focusInput}>Focus</button>
    </>
  );
}
```

**2. Lưu giá trị giữa các render mà không trigger re-render:**

```tsx
function Timer() {
  const [elapsed, setElapsed] = useState(0);
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);

  function start() {
    intervalRef.current = setInterval(
      () => setElapsed((prev) => prev + 1),
      1000
    );
  }

  function stop() {
    if (intervalRef.current) clearInterval(intervalRef.current);
  }

  return (
    <div>
      <p>{elapsed}s</p>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </div>
  );
}
```

---

### 4.3.6. useContext

`useContext` dùng để tiêu thụ giá trị từ một React Context mà không cần truyền props qua nhiều tầng (**prop drilling**).

```tsx
import { createContext, useContext, useState } from "react";

interface ThemeContextValue {
  theme: "light" | "dark";
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextValue | null>(null);

// Custom hook để dùng an toàn hơn
function useTheme(): ThemeContextValue {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme phải dùng trong ThemeProvider");
  return ctx;
}

function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<"light" | "dark">("light");

  const toggleTheme = () =>
    setTheme((prev) => (prev === "light" ? "dark" : "light"));

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// Dùng ở bất kỳ component con nào
function Header() {
  const { theme, toggleTheme } = useTheme();
  return (
    <header className={`header--${theme}`}>
      <button onClick={toggleTheme}>Đổi theme</button>
    </header>
  );
}
```

> **Lưu ý:** Khi giá trị Context thay đổi, **tất cả** component đang dùng `useContext` với context đó sẽ re-render. Nên tách context nếu có nhiều giá trị thay đổi độc lập.

---

### 4.3.7. useReducer

`useReducer` là lựa chọn thay thế cho `useState` khi logic state phức tạp, có nhiều sub-values, hoặc state tiếp theo phụ thuộc vào state trước.

```tsx
const [state, dispatch] = useReducer(reducer, initialState);
```

```tsx
interface CartItem {
  id: number;
  name: string;
  quantity: number;
}

type CartAction =
  | { type: "ADD_ITEM"; payload: CartItem }
  | { type: "REMOVE_ITEM"; payload: { id: number } }
  | { type: "CLEAR_CART" };

interface CartState {
  items: CartItem[];
}

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case "ADD_ITEM":
      return { items: [...state.items, action.payload] };
    case "REMOVE_ITEM":
      return { items: state.items.filter((i) => i.id !== action.payload.id) };
    case "CLEAR_CART":
      return { items: [] };
  }
}

function Cart() {
  const [cart, dispatch] = useReducer(cartReducer, { items: [] });

  return (
    <div>
      <p>Có {cart.items.length} sản phẩm</p>
      <button onClick={() => dispatch({ type: "CLEAR_CART" })}>
        Xóa giỏ hàng
      </button>
    </div>
  );
}
```

#### useState vs useReducer

| | `useState` | `useReducer` |
|---|---|---|
| Phù hợp khi | State đơn giản, độc lập | State phức tạp, nhiều action |
| Logic | Nằm trong component | Tập trung trong reducer |
| Test | Khó test logic | Dễ test (reducer là pure function) |
| Dùng với Context | Bình thường | Tốt hơn khi có nhiều action |

---

## 4.4. React.memo

`React.memo` là Higher-Order Component (HOC) bọc quanh một function component để **ngăn re-render không cần thiết** khi component cha re-render nhưng props không thay đổi.

```tsx
const MemoizedComponent = React.memo(MyComponent);
```

React.memo so sánh props theo **shallow comparison** (so sánh nông — kiểm tra reference, không kiểm tra sâu bên trong).

```tsx
interface UserCardProps {
  user: { id: number; name: string };
  onDelete: (id: number) => void;
}

const UserCard = React.memo(({ user, onDelete }: UserCardProps) => {
  console.log(`UserCard ${user.id} render`);
  return (
    <div>
      <p>{user.name}</p>
      <button onClick={() => onDelete(user.id)}>Xóa</button>
    </div>
  );
});

function UserList() {
  const [users, setUsers] = useState([{ id: 1, name: "An" }]);
  const [count, setCount] = useState(0);

  // Phải dùng useCallback — nếu không hàm mới mỗi render
  // → React.memo vô hiệu vì onDelete prop luôn thay đổi reference
  const handleDelete = useCallback((id: number) => {
    setUsers((prev) => prev.filter((u) => u.id !== id));
  }, []);

  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>
        Count: {count}
      </button>
      {users.map((user) => (
        <UserCard key={user.id} user={user} onDelete={handleDelete} />
      ))}
    </div>
  );
}
```

> `React.memo` chỉ hiệu quả khi kết hợp với `useCallback` cho hàm props và `useMemo` cho object/array props.

---

## 4.5. Controlled vs Uncontrolled Component

### Controlled Component

State của form element được quản lý hoàn toàn bởi React state. React là "nguồn sự thật duy nhất" (single source of truth).

```tsx
function ControlledForm() {
  const [email, setEmail] = useState("");

  return (
    <input
      type="email"
      value={email}                                      // React kiểm soát giá trị
      onChange={(e) => setEmail(e.target.value)}         // Cập nhật qua setState
    />
  );
}
```

### Uncontrolled Component

Giá trị của form element do DOM tự quản lý. React truy cập giá trị qua `ref` khi cần (thường khi submit).

```tsx
function UncontrolledForm() {
  const emailRef = useRef<HTMLInputElement>(null);

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    console.log(emailRef.current?.value); // Đọc khi cần
  }

  return (
    <form onSubmit={handleSubmit}>
      <input type="email" ref={emailRef} defaultValue="" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

### So sánh

| | Controlled | Uncontrolled |
|---|---|---|
| Nguồn sự thật | React state | DOM |
| Truy cập giá trị | Luôn có qua state | Chỉ qua `ref` khi cần |
| Validation realtime | Dễ dàng | Phức tạp hơn |
| React Hook Form | Uncontrolled (dùng ref) | — |
| Phù hợp | Form phức tạp, validation | Form đơn giản, file input |

> Trong thực tế, hầu hết dự án dùng **React Hook Form** — thư viện tận dụng uncontrolled inputs để tối ưu hiệu suất (không re-render mỗi keystroke).

---

## 4.6. Key

`key` là prop đặc biệt giúp React **nhận dạng** từng phần tử trong một danh sách. Khi danh sách thay đổi (thêm, xóa, sắp xếp lại), React dùng `key` để xác định phần tử nào thay đổi thay vì render lại toàn bộ.

### Tại sao cần key?

Không có `key`, React so sánh các phần tử theo **vị trí**. Khi thêm phần tử vào đầu danh sách, React nghĩ tất cả phần tử đều thay đổi và re-render toàn bộ — kém hiệu quả và có thể sai state.

```tsx
const users = [
  { id: 1, name: "An" },
  { id: 2, name: "Bình" },
];

// Sai — dùng index làm key, gây bug khi sort/filter/xóa
{users.map((user, index) => (
  <UserCard key={index} user={user} />
))}

// Đúng — dùng ID duy nhất, ổn định
{users.map((user) => (
  <UserCard key={user.id} user={user} />
))}
```

**Tại sao không dùng index làm key?** Khi xóa phần tử giữa danh sách, index của các phần tử sau thay đổi. React nhìn thấy key cũ tại vị trí mới và tái sử dụng DOM node — dẫn đến state của component (như giá trị input) bị gán nhầm cho phần tử sai.

---

## 4.7. Lifting State Up

Khi hai component anh em (siblings) cần chia sẻ dữ liệu, giải pháp là **đẩy state lên component cha chung gần nhất**. Component cha sẽ giữ state và truyền xuống cả hai con qua props.

```tsx
// Trước — mỗi component có state riêng, không chia sẻ được
function TemperatureInput() {
  const [value, setValue] = useState("");
  return <input value={value} onChange={(e) => setValue(e.target.value)} />;
}

// Sau — cha giữ state, truyền xuống hai con
interface TempInputProps {
  value: string;
  onChange: (value: string) => void;
}

function TemperatureInput({ value, onChange }: TempInputProps) {
  return (
    <input value={value} onChange={(e) => onChange(e.target.value)} />
  );
}

function TemperatureConverter() {
  const [celsius, setCelsius] = useState("");

  const fahrenheit = celsius ? String((Number(celsius) * 9) / 5 + 32) : "";

  return (
    <div>
      <TemperatureInput value={celsius} onChange={setCelsius} />
      <TemperatureInput value={fahrenheit} onChange={() => {}} />
    </div>
  );
}
```

---

## 4.8. Custom Hook

Custom Hook là hàm JavaScript bắt đầu bằng `use`, bên trong có dùng các hook của React. Mục đích là **tái sử dụng stateful logic** giữa nhiều component.

```tsx
// Hook tái sử dụng: fetch data với loading/error state
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    setIsLoading(true);
    fetch(url, { signal: controller.signal })
      .then((res) => {
        if (!res.ok) throw new Error(`HTTP error: ${res.status}`);
        return res.json() as Promise<T>;
      })
      .then(setData)
      .catch((err) => {
        if (err.name !== "AbortError") setError(err);
      })
      .finally(() => setIsLoading(false));

    return () => controller.abort();
  }, [url]);

  return { data, isLoading, error };
}

// Dùng trong component
function UserProfile({ userId }: { userId: number }) {
  const { data: user, isLoading, error } = useFetch<User>(`/api/users/${userId}`);

  if (isLoading) return <p>Đang tải...</p>;
  if (error) return <p>Lỗi: {error.message}</p>;
  if (!user) return null;

  return <h1>{user.name}</h1>;
}
```

Custom Hook giúp tách biệt **logic** khỏi **UI**, làm component gọn hơn và logic dễ test hơn.

---

## 4.9. Error Boundary

Error Boundary là component React có khả năng **bắt lỗi JavaScript** xảy ra trong cây component con (trong quá trình render, lifecycle methods), hiển thị UI dự phòng thay vì crash toàn bộ ứng dụng.

> **Lưu ý:** Error Boundary chỉ hoạt động với **class component** trong React hiện tại. Tuy nhiên có thể dùng thư viện `react-error-boundary` để có DX tốt hơn với function component.

```tsx
import { ErrorBoundary } from "react-error-boundary";

function ErrorFallback({
  error,
  resetErrorBoundary,
}: {
  error: Error;
  resetErrorBoundary: () => void;
}) {
  return (
    <div role="alert">
      <p>Đã xảy ra lỗi:</p>
      <pre>{error.message}</pre>
      <button onClick={resetErrorBoundary}>Thử lại</button>
    </div>
  );
}

function App() {
  return (
    <ErrorBoundary FallbackComponent={ErrorFallback}>
      <UserDashboard />
    </ErrorBoundary>
  );
}
```

Error Boundary **không bắt được** lỗi trong: event handlers, async code (setTimeout, fetch), và server-side rendering.

---

## 4.10. Suspense

`Suspense` cho phép component "treo" (suspend) render trong khi chờ một tác vụ bất đồng bộ hoàn tất (thường là lazy load component hoặc data fetching), và hiển thị UI fallback trong thời gian chờ.

```tsx
import { Suspense, lazy } from "react";

// Lazy load component
const HeavyChart = lazy(() => import("./HeavyChart"));

function Dashboard() {
  return (
    <Suspense fallback={<div>Đang tải biểu đồ...</div>}>
      <HeavyChart />
    </Suspense>
  );
}
```

Trong Next.js App Router, Suspense kết hợp với Server Component và Streaming để render từng phần trang khi sẵn sàng:

```tsx
// app/dashboard/page.tsx
import { Suspense } from "react";

export default function DashboardPage() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<StatsSkeleton />}>
        <Stats />  {/* Server Component fetch data */}
      </Suspense>
      <Suspense fallback={<TableSkeleton />}>
        <DataTable />
      </Suspense>
    </div>
  );
}
```

---

## 4.11. Lazy Loading

Lazy Loading là kỹ thuật trì hoãn tải JavaScript bundle cho đến khi thực sự cần. Thay vì tải toàn bộ code ứng dụng lúc khởi động, chỉ tải code của route/component khi người dùng điều hướng đến.

`React.lazy()` kết hợp với dynamic `import()` để tách bundle:

```tsx
import { lazy, Suspense } from "react";

// Component được tách thành file bundle riêng
const AdminPanel = lazy(() => import("./AdminPanel"));
const UserSettings = lazy(() => import("./UserSettings"));

function App() {
  const [view, setView] = useState<"admin" | "settings">("admin");

  return (
    <Suspense fallback={<LoadingSpinner />}>
      {view === "admin" ? <AdminPanel /> : <UserSettings />}
    </Suspense>
  );
}
```

### Lợi ích

- Giảm kích thước bundle ban đầu (initial bundle size).
- Trang tải nhanh hơn vì chỉ tải code cần thiết.
- Tự động tách chunk nhờ bundler (Webpack, Vite).

### Lazy Loading theo Route (Next.js)

Trong Next.js App Router, mỗi `page.tsx` tự động được code-split. Ngoài ra có thể dùng `next/dynamic`:

```tsx
import dynamic from "next/dynamic";

// Chỉ render phía client, không SSR
const RichTextEditor = dynamic(() => import("./RichTextEditor"), {
  ssr: false,
  loading: () => <p>Đang tải editor...</p>,
});

// Preload khi hover để giảm delay
const Chart = dynamic(() => import("./Chart"));
```

| | `React.lazy` | `next/dynamic` |
|---|---|---|
| Framework | React thuần | Next.js |
| SSR control | Không | Có (`ssr: false`) |
| Loading state | Qua `Suspense` | Qua `loading` option |
| Preloading | Thủ công | Tự động |
