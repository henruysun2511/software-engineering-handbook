# Chương 7: State Management

State management là cách tổ chức và quản lý dữ liệu (state) trong ứng dụng React. Chọn đúng giải pháp phụ thuộc vào phạm vi, độ phức tạp và tần suất thay đổi của dữ liệu.

```
Local State    → useState, useReducer  (dữ liệu của một component)
Shared State   → Context, Zustand      (dữ liệu chia sẻ nhiều component)
Server State   → TanStack Query        (dữ liệu từ server — Chương 8)
```

**Nguyên tắc chọn giải pháp:** đừng bắt đầu bằng câu hỏi "dùng thư viện nào", hãy bắt đầu bằng câu hỏi "state này ai cần đọc, ai cần sửa, và nó thay đổi bao nhiêu lần mỗi giây". State chỉ một component dùng → local state. State nhiều component không liên quan (khác cây component) cùng cần → shared state. Dữ liệu vốn dĩ "sống" trên server và chỉ được cache lại ở client → server state (Chương 8), tuyệt đối không nhét vào Redux/Zustand rồi tự viết loading/error/cache tay.

---

## 7.1. Local State

Local state là state chỉ tồn tại và có ý nghĩa trong phạm vi **một component**. Đây là loại state đơn giản nhất, quản lý bằng `useState` hoặc `useReducer`.

**Dùng khi:**
- Trạng thái UI nội bộ: đóng/mở modal, tab đang chọn, giá trị input.
- Dữ liệu không cần chia sẻ ra ngoài component.

### Bản chất của `useState`

`useState` không phải là "một biến số". Mỗi lần component re-render, toàn bộ hàm component chạy lại từ đầu — nhưng React lưu giá trị state ở **ngoài** function component, trong một cấu trúc gọi là *fiber* gắn với vị trí của component trong cây render. Gọi `setState` không thay đổi giá trị ngay lập tức trong lần render hiện tại; nó **lên lịch một lần re-render mới**, và ở lần render đó, `useState` sẽ trả về giá trị mới.

Đây là lý do vì sao đoạn code sau luôn log ra giá trị cũ:

```tsx
function Example() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1);
    console.log(count); // vẫn là giá trị TRƯỚC khi click, vì `count` ở đây
                         // là biến const của lần render này, không tự cập nhật
  };

  return <button onClick={handleClick}>{count}</button>;
}
```

Khi cần dựa vào giá trị state trước đó để tính giá trị mới, luôn dùng **functional update** thay vì đọc trực tiếp biến state — vì trong React 18 trở lên (đặc biệt là dưới Strict Mode hoặc khi có nhiều lần gọi `setState` dồn trong cùng một event), các lần gọi có thể được batch lại:

```tsx
// Sai: nếu gọi 2 lần liên tiếp trong cùng handler, chỉ +1 chứ không +2
setCount(count + 1);
setCount(count + 1);

// Đúng: mỗi lần đều nhận state mới nhất từ React
setCount((prev) => prev + 1);
setCount((prev) => prev + 1);
```

```tsx
function Accordion({ title, children }: AccordionProps) {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div>
      <button onClick={() => setIsOpen((prev) => !prev)}>{title}</button>
      {isOpen && <div>{children}</div>}
    </div>
  );
}
```

### Khi nào chuyển từ `useState` sang `useReducer`

`useState` phù hợp khi state là một giá trị đơn giản (số, chuỗi, boolean) hoặc một object nhỏ mà các trường không phụ thuộc lẫn nhau. Vấn đề xuất hiện khi:

- State có **nhiều trường liên quan chặt chẽ**, và một hành động của người dùng phải cập nhật nhiều trường cùng lúc theo đúng thứ tự (nếu dùng nhiều `useState` riêng lẻ, dễ quên cập nhật một trường → state rơi vào trạng thái không nhất quán).
- Logic cập nhật state **phức tạp, có điều kiện rẽ nhánh** (thêm/sửa/xóa/toggle...) — nếu để trong component, các hàm `onClick` sẽ phình to và khó test.
- Bạn muốn **tách bạch rõ ràng** giữa "chuyện gì đã xảy ra" (action) và "state thay đổi như thế nào" (reducer), giúp code dễ đọc, dễ test độc lập (test reducer là hàm thuần, không cần render component).

`useReducer` áp dụng đúng mô hình Redux thu nhỏ: `(state, action) => newState`. Reducer **phải là hàm thuần** (pure function) — không side effect, không gọi API, không mutate trực tiếp `state` đầu vào, chỉ nhận state cũ + action rồi trả về state mới.

### Ví dụ đầy đủ: Todo App với `useReducer`

```tsx
// types.ts
interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

// Định nghĩa tất cả các "hành động" có thể xảy ra với danh sách todo.
// Dùng discriminated union để TypeScript tự suy ra kiểu payload theo từng action.type
type TodoAction =
  | { type: "ADD"; text: string }
  | { type: "TOGGLE"; id: string }
  | { type: "REMOVE"; id: string }
  | { type: "EDIT"; id: string; text: string }
  | { type: "CLEAR_COMPLETED" }
  | { type: "SET_FILTER"; filter: Filter };

type Filter = "all" | "active" | "completed";

interface TodoState {
  todos: Todo[];
  filter: Filter;
}

const initialState: TodoState = {
  todos: [],
  filter: "all",
};
```

```tsx
// todoReducer.ts

// Reducer là hàm thuần: cùng input luôn cho cùng output, không side effect.
// Luôn trả về OBJECT MỚI, không mutate `state` trực tiếp (khác với Immer trong RTK ở mục 7.4).
function todoReducer(state: TodoState, action: TodoAction): TodoState {
  switch (action.type) {
    case "ADD": {
      const trimmed = action.text.trim();
      if (!trimmed) return state; // guard: không thêm todo rỗng, trả nguyên state cũ

      const newTodo: Todo = {
        id: crypto.randomUUID(),
        text: trimmed,
        completed: false,
      };
      return { ...state, todos: [...state.todos, newTodo] };
    }

    case "TOGGLE":
      return {
        ...state,
        todos: state.todos.map((todo) =>
          todo.id === action.id ? { ...todo, completed: !todo.completed } : todo
        ),
      };

    case "REMOVE":
      return {
        ...state,
        todos: state.todos.filter((todo) => todo.id !== action.id),
      };

    case "EDIT":
      return {
        ...state,
        todos: state.todos.map((todo) =>
          todo.id === action.id ? { ...todo, text: action.text.trim() || todo.text } : todo
        ),
      };

    case "CLEAR_COMPLETED":
      return {
        ...state,
        todos: state.todos.filter((todo) => !todo.completed),
      };

    case "SET_FILTER":
      return { ...state, filter: action.filter };

    default:
      // TypeScript sẽ báo lỗi ở đây nếu có action.type chưa được xử lý (exhaustiveness check)
      return state;
  }
}
```

```tsx
// TodoApp.tsx
import { useReducer, useState, useMemo } from "react";

function TodoApp() {
  const [state, dispatch] = useReducer(todoReducer, initialState);
  const [input, setInput] = useState("");

  // Lọc danh sách hiển thị theo filter — dùng useMemo để tránh lọc lại
  // mỗi lần render nếu todos và filter không đổi
  const visibleTodos = useMemo(() => {
    switch (state.filter) {
      case "active":
        return state.todos.filter((t) => !t.completed);
      case "completed":
        return state.todos.filter((t) => t.completed);
      default:
        return state.todos;
    }
  }, [state.todos, state.filter]);

  const remainingCount = state.todos.filter((t) => !t.completed).length;

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    dispatch({ type: "ADD", text: input });
    setInput("");
  };

  return (
    <div>
      <form onSubmit={handleSubmit}>
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="Thêm việc cần làm..."
        />
        <button type="submit">Thêm</button>
      </form>

      <ul>
        {visibleTodos.map((todo) => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => dispatch({ type: "TOGGLE", id: todo.id })}
            />
            <span style={{ textDecoration: todo.completed ? "line-through" : "none" }}>
              {todo.text}
            </span>
            <button onClick={() => dispatch({ type: "REMOVE", id: todo.id })}>Xóa</button>
          </li>
        ))}
      </ul>

      <div>
        <span>{remainingCount} việc còn lại</span>
        {(["all", "active", "completed"] as const).map((f) => (
          <button
            key={f}
            onClick={() => dispatch({ type: "SET_FILTER", filter: f })}
            disabled={state.filter === f}
          >
            {f}
          </button>
        ))}
        <button onClick={() => dispatch({ type: "CLEAR_COMPLETED" })}>
          Xóa việc đã hoàn thành
        </button>
      </div>
    </div>
  );
}
```

**Vì sao ví dụ này chọn `useReducer` thay vì nhiều `useState`?** Nếu viết bằng `useState`, ta sẽ cần ít nhất `const [todos, setTodos] = useState<Todo[]>([])` và `const [filter, setFilter] = useState<Filter>("all")`, và mỗi thao tác (toggle, remove, edit...) sẽ là một hàm riêng viết logic `map`/`filter` ngay trong component — code dồn hết vào JSX, khó tách ra test, và mỗi lần thêm một loại thao tác mới lại phải viết thêm một hàm `handleXxx` mới. Với `useReducer`, toàn bộ logic "todos thay đổi như thế nào" nằm gọn trong một hàm `todoReducer` thuần, có thể unit test độc lập:

```tsx
// todoReducer.test.ts
test("ADD thêm todo mới vào cuối danh sách", () => {
  const state = { todos: [], filter: "all" as const };
  const next = todoReducer(state, { type: "ADD", text: "Học React" });
  expect(next.todos).toHaveLength(1);
  expect(next.todos[0].text).toBe("Học React");
});
```

> **Lưu ý:** `useReducer` vẫn là **local state** — state này chỉ tồn tại trong `TodoApp` và biến mất khi component unmount. Nếu cần chia sẻ danh sách todo cho nhiều component ở xa nhau trong cây, hoặc cần persist qua reload trang, đây là lúc cân nhắc Zustand (7.3) hoặc Redux Toolkit (7.4) — bản chất reducer bên trong gần như giữ nguyên, chỉ khác nơi state được lưu trữ.

---

## 7.2. Context

Context là cơ chế React tích hợp sẵn để chia sẻ state giữa nhiều component mà không cần truyền props qua từng tầng (**prop drilling**). Đã được trình bày kỹ ở mục `4.3.6. useContext`.

**Dùng khi:**
- Dữ liệu ít thay đổi, mang tính toàn cục: theme, ngôn ngữ, thông tin user đăng nhập.
- Ứng dụng nhỏ đến trung bình, chưa cần thư viện state management.

### Bản chất: vì sao Context gây re-render toàn bộ subscriber

Context hoạt động theo cơ chế publish-subscribe rất đơn giản: `<Context.Provider value={...}>` là nơi "publish", còn mọi component gọi `useContext(Context)` bên trong cây con là "subscriber". Khi `value` truyền vào Provider thay đổi (so sánh bằng `Object.is`), **React re-render toàn bộ subscriber**, bất kể subscriber đó có thực sự dùng đến phần dữ liệu vừa đổi hay không — vì Context không có khái niệm "chọn một phần của value", nó chỉ trả về nguyên object `value`.

Lỗi thường gặp: tạo `value` là một object literal mới mỗi lần Provider render, khiến subscriber re-render dù dữ liệu bên trong không đổi:

```tsx
// Sai: mỗi lần AppProvider render, {user, theme} là object MỚI
// (dù user, theme không đổi) → mọi subscriber đều re-render
function AppProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [theme, setTheme] = useState<"light" | "dark">("light");

  return (
    <AppContext.Provider value={{ user, setUser, theme, setTheme }}>
      {children}
    </AppContext.Provider>
  );
}
```

```tsx
// Đúng hơn: dùng useMemo để value chỉ đổi khi user hoặc theme thực sự đổi
function AppProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [theme, setTheme] = useState<"light" | "dark">("light");

  const value = useMemo(
    () => ({ user, setUser, theme, setTheme }),
    [user, theme]
  );

  return <AppContext.Provider value={value}>{children}</AppContext.Provider>;
}
```

Nhưng `useMemo` chỉ giải quyết được trường hợp value không đổi mà vẫn re-render *oan*; nó **không** giải quyết được vấn đề gốc: khi `user` thật sự đổi, mọi subscriber của `AppContext` — kể cả component chỉ đọc `theme` — vẫn re-render theo, vì cả hai nằm chung một Context. Cách khắc phục đúng là **tách Context theo tần suất/nhóm thay đổi** (một Context cho `user`, một Context riêng cho `theme`) thay vì gộp hết vào một Context lớn. Đây chính là hạn chế cấu trúc mà Zustand giải quyết triệt để hơn nhờ cơ chế selector.

**Hạn chế:** Mỗi khi giá trị Context thay đổi, toàn bộ component đang dùng `useContext` với context đó sẽ re-render — kể cả những component không dùng đến phần dữ liệu vừa thay đổi. Đây là lý do chính để chuyển sang Zustand hoặc Redux Toolkit khi state phức tạp hơn.

---

## 7.3. Zustand

Zustand là thư viện state management nhỏ gọn (~1KB), không cần boilerplate, không cần Provider bao ngoài. State được lưu ngoài React, component chỉ subscribe vào đúng phần state mình cần.

### Bản chất: vì sao Zustand không cần Provider và không re-render toàn bộ

Zustand lưu state trong một store **độc lập với cây component React** — thực chất là một object JavaScript thường, kèm một danh sách các listener (hàm callback). Khi gọi `create()`, Zustand trả về một custom hook (ví dụ `useCounterStore`) đã "đóng gói" sẵn tham chiếu tới store đó — đây là lý do không cần `<Provider>`: store không nằm trong React tree, nó tồn tại ở module scope, mọi nơi `import` store vào đều trỏ tới cùng một instance.

Khi component gọi `useCounterStore((state) => state.count)`:
1. Zustand đăng ký component đó làm **subscriber**, nhưng không subscribe vào toàn bộ store — nó lưu lại **hàm selector** `(state) => state.count`.
2. Mỗi khi `set()` được gọi, Zustand chạy lại selector với state mới, rồi so sánh kết quả mới với kết quả lần trước bằng `Object.is` (so sánh tham chiếu nông — shallow reference equality).
3. Chỉ khi kết quả so sánh **khác nhau**, Zustand mới trigger re-render cho đúng component đó.

Đây là khác biệt cốt lõi so với Context: Context so sánh cả object `value`, còn Zustand so sánh **kết quả của selector** — nên hai component subscribe cùng một store nhưng lấy hai trường khác nhau sẽ re-render độc lập nhau, đúng như ví dụ `NavBar` / `WelcomeMessage` ở dưới.

### create

`create` là hàm khởi tạo store. Store là nơi chứa state và các action để thay đổi state.

```tsx
// store/useCounterStore.ts
import { create } from "zustand";

interface CounterState {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
}

const useCounterStore = create<CounterState>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}));
```

Dùng trong component — **không cần Provider**:

```tsx
function Counter() {
  const count = useCounterStore((state) => state.count);
  const increment = useCounterStore((state) => state.increment);

  return (
    <div>
      <p>{count}</p>
      <button onClick={increment}>Tăng</button>
    </div>
  );
}
```

`set` trong Zustand mặc định làm **shallow merge** với state hiện tại — `set({ count: 0 })` chỉ ghi đè trường `count`, các trường khác trong store giữ nguyên (khác với `setState` thay thế hoàn toàn của `useReducer`, nơi bạn phải tự `...state` để giữ các trường cũ).

### Selector

Selector là hàm được truyền vào store hook để lấy ra **đúng phần state cần dùng**. Component chỉ re-render khi phần state đó thay đổi — đây là cơ chế tối ưu hiệu suất cốt lõi của Zustand.

```tsx
// store/useUserStore.ts
interface UserState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  setUser: (user: User, token: string) => void;
  logout: () => void;
}

const useUserStore = create<UserState>((set) => ({
  user: null,
  token: null,
  isAuthenticated: false,

  setUser: (user, token) => set({ user, token, isAuthenticated: true }),
  logout: () => set({ user: null, token: null, isAuthenticated: false }),
}));

// Component A — chỉ cần isAuthenticated, chỉ re-render khi trường đó đổi
function NavBar() {
  const isAuthenticated = useUserStore((state) => state.isAuthenticated);
  return <nav>{isAuthenticated ? <UserMenu /> : <LoginButton />}</nav>;
}

// Component B — chỉ cần user.name
function WelcomeMessage() {
  const name = useUserStore((state) => state.user?.name);
  return <p>Xin chào, {name}</p>;
}
```

> Tránh lấy toàn bộ store `useUserStore()` không có selector — component sẽ re-render mỗi khi bất kỳ field nào thay đổi.

**Selector trả về nhiều trường cùng lúc:** nếu selector trả về một object mới mỗi lần gọi (ví dụ `useUserStore((s) => ({ name: s.user?.name, email: s.user?.email }))`), object đó sẽ luôn khác tham chiếu với lần trước dù `name`/`email` không đổi → component vẫn re-render oan, giống lỗi Context ở trên. Cách xử lý: dùng `useShallow` (Zustand v4.4+) để so sánh nông từng trường bên trong object thay vì so sánh tham chiếu object:

```tsx
import { useShallow } from "zustand/react/shallow";

function UserCard() {
  const { name, email } = useUserStore(
    useShallow((state) => ({ name: state.user?.name, email: state.user?.email }))
  );
  return <div>{name} — {email}</div>;
}
```

### Đọc/ghi state ngoài component (actions gọi API)

Vì store tồn tại độc lập với React, action bên trong store có thể `get()` state hiện tại và gọi API mà không cần truyền tham số qua props:

```tsx
// store/useCartStore.ts
interface CartState {
  items: CartItem[];
  isLoading: boolean;
  addItem: (item: CartItem) => void;
  checkout: () => Promise<void>;
}

const useCartStore = create<CartState>((set, get) => ({
  items: [],
  isLoading: false,

  addItem: (item) =>
    set((state) => ({ items: [...state.items, item] })),

  checkout: async () => {
    set({ isLoading: true });
    try {
      const { items } = get(); // đọc state hiện tại ngay trong action
      await fetch("/api/checkout", {
        method: "POST",
        body: JSON.stringify({ items }),
      });
      set({ items: [], isLoading: false });
    } catch (err) {
      set({ isLoading: false });
      throw err;
    }
  },
}));
```

### Persist

`persist` middleware cho phép đồng bộ state với `localStorage` hoặc `sessionStorage`, giúp state tồn tại sau khi refresh trang.

```tsx
// store/useSettingsStore.ts
import { create } from "zustand";
import { persist, createJSONStorage } from "zustand/middleware";

interface SettingsState {
  theme: "light" | "dark";
  language: "vi" | "en";
  setTheme: (theme: "light" | "dark") => void;
  setLanguage: (lang: "vi" | "en") => void;
}

const useSettingsStore = create<SettingsState>()(
  persist(
    (set) => ({
      theme: "light",
      language: "vi",
      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
    }),
    {
      name: "app-settings",                          // Key trong localStorage
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({                      // Chỉ persist một số field
        theme: state.theme,
        language: state.language,
      }),
    }
  )
);
```

**Bản chất của `persist`:** middleware này "bọc" hàm khởi tạo store gốc — mỗi khi `set()` được gọi, ngoài việc cập nhật state trong bộ nhớ, nó còn ghi state (đã qua `partialize`) xuống `storage` bằng `JSON.stringify`. Khi app khởi động lại, `persist` đọc `storage`, `JSON.parse`, rồi **merge** với state khởi tạo trước khi component đầu tiên render — vì vậy lần render đầu tiên trên client có thể có một nhịp "chớp" giá trị mặc định trước khi giá trị đã lưu được áp vào (cần lưu ý khi dùng với Next.js SSR, dễ gây *hydration mismatch* nếu giá trị persisted khác giá trị server render ra).

---

## 7.4. Redux Toolkit (RTK)

Redux Toolkit là bộ công cụ chính thức để viết Redux, giúp giảm đáng kể boilerplate so với Redux thuần. RTK phù hợp với ứng dụng lớn, logic phức tạp, cần DevTools mạnh và khả năng kiểm soát cao.

### Bản chất: Redux hoạt động như thế nào

Redux dựa trên ba nguyên tắc cốt lõi:

1. **Một nguồn sự thật duy nhất (single source of truth):** toàn bộ state của ứng dụng nằm trong một object JavaScript duy nhất (`store`).
2. **State chỉ đọc (read-only):** cách duy nhất để thay đổi state là **dispatch một action** — một object mô tả "chuyện gì đã xảy ra" (ví dụ `{ type: "cart/addItem", payload: {...} }*). Component không bao giờ được sửa state trực tiếp.
3. **Thay đổi bằng hàm thuần (pure reducer):** khi một action được dispatch, Redux gọi `reducer(currentState, action)` để tính ra state mới. Reducer không được gây side effect và không được mutate state cũ.

Về bản chất, `configureStore` chỉ là một vòng lặp: mỗi lần `dispatch(action)` được gọi, Redux chạy `rootReducer(state, action)`, `rootReducer` này thực chất gộp tất cả các slice reducer lại — mỗi slice chỉ nhận đúng phần state của mình (ví dụ `cartReducer` chỉ nhận `state.cart`), tính ra phần state mới, rồi Redux ghép các phần lại thành state tổng mới. Sau đó Redux thông báo cho mọi component đã `subscribe` (thông qua `useSelector`) để chúng kiểm tra xem phần state mình quan tâm có đổi không.

`useSelector` hoạt động theo cơ chế tương tự selector của Zustand: nó chạy hàm selector trên state cũ và state mới, so sánh bằng `===`, chỉ re-render nếu khác nhau. Đây là lý do best practice là luôn `useSelector` lấy đúng phần cần dùng (`state.cart.items`), không lấy nguyên `state.cart` nếu chỉ cần `items`.

### Slice

`createSlice` kết hợp **reducer** và **action creator** vào một chỗ. Tên slice sẽ là prefix của action type.

```tsx
// store/cartSlice.ts
import { createSlice, PayloadAction } from "@reduxjs/toolkit";

interface CartItem {
  id: number;
  name: string;
  price: number;
  quantity: number;
}

interface CartState {
  items: CartItem[];
  isOpen: boolean;
}

const initialState: CartState = {
  items: [],
  isOpen: false,
};

const cartSlice = createSlice({
  name: "cart",
  initialState,
  reducers: {
    addItem(state, action: PayloadAction<CartItem>) {
      const existing = state.items.find((i) => i.id === action.payload.id);
      if (existing) {
        existing.quantity += 1; // Immer cho phép mutate trực tiếp
      } else {
        state.items.push(action.payload);
      }
    },
    removeItem(state, action: PayloadAction<number>) {
      state.items = state.items.filter((i) => i.id !== action.payload);
    },
    toggleCart(state) {
      state.isOpen = !state.isOpen;
    },
    clearCart(state) {
      state.items = [];
    },
  },
});

export const { addItem, removeItem, toggleCart, clearCart } = cartSlice.actions;
export default cartSlice.reducer;
```

> RTK tích hợp **Immer** — cho phép viết code "mutate" state trực tiếp trong reducer, Immer sẽ tự tạo bản sao immutable phía sau.

**Bản chất của Immer:** khi bạn viết `state.items.push(...)` bên trong reducer của `createSlice`, bạn **không** thực sự mutate object state gốc. Immer bọc `state` bằng một `Proxy` — mọi phép gán/`push`/xóa trên proxy đó được Immer ghi lại thành một "bản nháp" (draft) thay đổi, sau khi hàm reducer chạy xong, Immer dựa vào bản nháp đó để tạo ra một **object hoàn toàn mới** cho các phần đã đổi (structural sharing: phần nào không đổi thì giữ nguyên tham chiếu cũ, giúp `useSelector` so sánh `===` vẫn hoạt động đúng). Vì vậy "mutate" trong RTK chỉ là cú pháp thuận tiện — dưới cùng vẫn tuân thủ nguyên tắc immutability của Redux. Lưu ý: cú pháp mutate này **chỉ hợp lệ bên trong `createSlice`/`createReducer`**, vì chỉ ở đó Immer mới bọc proxy; nếu bạn tự viết reducer thuần bằng `switch` như ở mục 7.1, mutate trực tiếp state là **lỗi thực sự**.

---

### Store

Store là nơi tổng hợp tất cả slice reducer:

```tsx
// store/index.ts
import { configureStore } from "@reduxjs/toolkit";
import cartReducer from "./cartSlice";
import userReducer from "./userSlice";

export const store = configureStore({
  reducer: {
    cart: cartReducer,
    user: userReducer,
  },
});

// TypeScript: Lấy kiểu từ store
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

Cung cấp store cho toàn ứng dụng:

```tsx
// app/providers.tsx
"use client";
import { Provider } from "react-redux";
import { store } from "@/store";

export function Providers({ children }: { children: React.ReactNode }) {
  return <Provider store={store}>{children}</Provider>;
}
```

Dùng typed hooks để truy cập store an toàn:

```tsx
// store/hooks.ts
import { useDispatch, useSelector } from "react-redux";
import type { RootState, AppDispatch } from "./index";

// Dùng hai hook này thay vì useSelector/useDispatch gốc
export const useAppSelector = useSelector.withTypes<RootState>();
export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
```

```tsx
// components/Cart.tsx
import { useAppSelector, useAppDispatch } from "@/store/hooks";
import { removeItem, clearCart } from "@/store/cartSlice";

function Cart() {
  const items = useAppSelector((state) => state.cart.items);
  const dispatch = useAppDispatch();

  return (
    <div>
      {items.map((item) => (
        <div key={item.id}>
          <span>{item.name}</span>
          <button onClick={() => dispatch(removeItem(item.id))}>Xóa</button>
        </div>
      ))}
      <button onClick={() => dispatch(clearCart())}>Xóa tất cả</button>
    </div>
  );
}
```

---

### Thunk

Thunk là middleware cho phép action creator trả về **một hàm** thay vì một plain object. Dùng để xử lý **logic bất đồng bộ** (gọi API) bên trong Redux.

**Vì sao cần middleware để làm việc này?** Bản thân `dispatch` trong Redux thuần chỉ chấp nhận plain object (action). Nếu bạn `dispatch(fetchUser())` mà `fetchUser` trả về một `Promise` hoặc một hàm `async`, reducer sẽ không biết xử lý — nó không phải là action hợp lệ. Middleware `thunk` (đã được `configureStore` tự động thêm sẵn) **chặn action trước khi nó tới reducer**: nếu action đó là một hàm, middleware sẽ gọi hàm đó với `(dispatch, getState)` thay vì chuyển thẳng cho reducer; nếu là object thường, middleware để nó đi tiếp bình thường. Đây là cách Redux "dạy" cho một hệ thống vốn chỉ hiểu object thuần biết cách xử lý bất đồng bộ.

`createAsyncThunk` tạo thunk với ba action tự động: `pending`, `fulfilled`, `rejected`.

```tsx
// store/userSlice.ts
import { createSlice, createAsyncThunk } from "@reduxjs/toolkit";

interface UserState {
  data: User | null;
  status: "idle" | "loading" | "succeeded" | "failed";
  error: string | null;
}

// Tạo async thunk
export const fetchCurrentUser = createAsyncThunk(
  "user/fetchCurrent",
  async (_, { rejectWithValue }) => {
    try {
      const res = await fetch("/api/me");
      if (!res.ok) throw new Error("Unauthorized");
      return await res.json() as User;
    } catch (err) {
      return rejectWithValue((err as Error).message);
    }
  }
);

const userSlice = createSlice({
  name: "user",
  initialState: { data: null, status: "idle", error: null } as UserState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchCurrentUser.pending, (state) => {
        state.status = "loading";
      })
      .addCase(fetchCurrentUser.fulfilled, (state, action) => {
        state.status = "succeeded";
        state.data = action.payload;
      })
      .addCase(fetchCurrentUser.rejected, (state, action) => {
        state.status = "failed";
        state.error = action.payload as string;
      });
  },
});

export default userSlice.reducer;
```

Khi `dispatch(fetchCurrentUser())` được gọi, `createAsyncThunk` tự động dispatch action `user/fetchCurrent/pending` **ngay lập tức** (đồng bộ), sau đó chạy hàm `async` bạn truyền vào; khi hàm đó `resolve`, nó dispatch tiếp `user/fetchCurrent/fulfilled` kèm `payload` là giá trị return; nếu hàm `reject` hoặc gọi `rejectWithValue`, nó dispatch `user/fetchCurrent/rejected`. `extraReducers` chính là nơi lắng nghe ba action tự động này — gọi là "extra" vì chúng không được định nghĩa trong khối `reducers` thông thường của slice (action type của chúng không có prefix `user/` đơn thuần mà đã được `createAsyncThunk` sinh sẵn).

Dispatch thunk từ component:

```tsx
function App() {
  const dispatch = useAppDispatch();
  const { data: user, status } = useAppSelector((state) => state.user);

  useEffect(() => {
    dispatch(fetchCurrentUser());
  }, [dispatch]);

  if (status === "loading") return <p>Đang tải...</p>;
  if (!user) return null;
  return <h1>Xin chào, {user.name}</h1>;
}
```

> **Lưu ý thực tế:** mẫu `status: "idle" | "loading" | "succeeded" | "failed"` này là cách làm phổ biến trước khi có TanStack Query, nhưng phải tự tay quản lý cache, tự invalidate khi data cũ, tự tránh gọi trùng request. Với dữ liệu đến từ server (user profile, danh sách sản phẩm, v.v.), Chương 8 sẽ giới thiệu TanStack Query — giải quyết đúng bài toán này với ít code hơn nhiều và có sẵn cache, refetch, retry. Redux Toolkit + `createAsyncThunk` vẫn rất hợp lý cho state **không thuần server** (giỏ hàng, wizard nhiều bước, undo/redo...).

---

### So sánh Context / Zustand / Redux Toolkit

| Tiêu chí | Context | Zustand | Redux Toolkit |
|---|---|---|---|
| Boilerplate | Thấp | Rất thấp | Trung bình |
| Bundle size | 0 (built-in) | ~1 KB | ~11 KB |
| Nơi lưu state | Trong cây React (Provider) | Ngoài React (module scope) | Ngoài React (module scope) |
| DevTools | Không | Có (middleware) | Rất mạnh |
| Re-render | Toàn bộ subscriber | Chỉ subscriber liên quan (so sánh kết quả selector) | Chỉ subscriber liên quan (so sánh kết quả selector) |
| Cập nhật state | Trực tiếp qua setState | `set()` — shallow merge | Reducer thuần (được Immer hỗ trợ "mutate") |
| Async | Thủ công | Thủ công | `createAsyncThunk` (có pending/fulfilled/rejected) |
| Middleware | Không | Có (persist, immer...) | Có (sẵn Immer, Thunk) |
| Cần Provider | Có | Không | Có |
| Dùng khi | Theme, auth đơn giản, state ít thay đổi | App vừa, muốn đơn giản, không cần Provider | App lớn, team lớn, cần chuẩn hóa nghiêm ngặt + DevTools mạnh |

**Ghi nhớ bản chất chung:** cả ba giải pháp shared-state đều xoay quanh cùng một câu hỏi — *"làm sao để component chỉ re-render khi đúng phần dữ liệu nó cần thay đổi?"*. Context trả lời bằng cách so sánh cả `value`; Zustand và Redux (qua `useSelector`) trả lời bằng cách so sánh **kết quả của một hàm chọn lọc (selector)** áp lên state. Hiểu được nguyên lý selector này là hiểu được vì sao Zustand/RTK "nhanh" hơn Context ở quy mô lớn — không phải vì bản thân thư viện nhanh hơn, mà vì cơ chế subscribe của chúng có độ chi tiết (granularity) cao hơn.