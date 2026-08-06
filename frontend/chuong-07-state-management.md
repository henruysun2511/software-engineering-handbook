# Chương 7: State Management

State management là cách tổ chức và quản lý dữ liệu (state) trong ứng dụng React. Chọn đúng giải pháp phụ thuộc vào phạm vi, độ phức tạp và tần suất thay đổi của dữ liệu.

```
Local State    → useState, useReducer  (dữ liệu của một component)
Shared State   → Context, Zustand      (dữ liệu chia sẻ nhiều component)
Server State   → TanStack Query        (dữ liệu từ server — Chương 8)
```

---

## 7.1. Local State

Local state là state chỉ tồn tại và có ý nghĩa trong phạm vi **một component**. Đây là loại state đơn giản nhất, quản lý bằng `useState` hoặc `useReducer`.

**Dùng khi:**
- Trạng thái UI nội bộ: đóng/mở modal, tab đang chọn, giá trị input.
- Dữ liệu không cần chia sẻ ra ngoài component.

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

---

## 7.2. Context

Context là cơ chế React tích hợp sẵn để chia sẻ state giữa nhiều component mà không cần truyền props qua từng tầng (**prop drilling**). Đã được trình bày kỹ ở mục `4.3.6. useContext`.

**Dùng khi:**
- Dữ liệu ít thay đổi, mang tính toàn cục: theme, ngôn ngữ, thông tin user đăng nhập.
- Ứng dụng nhỏ đến trung bình, chưa cần thư viện state management.

**Hạn chế:** Mỗi khi giá trị Context thay đổi, toàn bộ component đang dùng `useContext` với context đó sẽ re-render — kể cả những component không dùng đến phần dữ liệu vừa thay đổi. Đây là lý do chính để chuyển sang Zustand hoặc Redux Toolkit khi state phức tạp hơn.

---

## 7.3. Zustand

Zustand là thư viện state management nhỏ gọn (~1KB), không cần boilerplate, không cần Provider bao ngoài. State được lưu ngoài React, component chỉ subscribe vào đúng phần state mình cần.

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

---

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

---

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

---

## 7.4. Redux Toolkit (RTK)

Redux Toolkit là bộ công cụ chính thức để viết Redux, giúp giảm đáng kể boilerplate so với Redux thuần. RTK phù hợp với ứng dụng lớn, logic phức tạp, cần DevTools mạnh và khả năng kiểm soát cao.

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

---

### So sánh Context / Zustand / Redux Toolkit

| Tiêu chí | Context | Zustand | Redux Toolkit |
|---|---|---|---|
| Boilerplate | Thấp | Rất thấp | Trung bình |
| Bundle size | 0 (built-in) | ~1 KB | ~11 KB |
| DevTools | Không | Có (middleware) | Rất mạnh |
| Re-render | Toàn bộ subscriber | Chỉ subscriber liên quan | Chỉ subscriber liên quan |
| Async | Thủ công | Thủ công | `createAsyncThunk` |
| Middleware | Không | Có (persist, immer...) | Có (sẵn Immer, Thunk) |
| Dùng khi | Theme, auth đơn giản | App vừa, muốn đơn giản | App lớn, team lớn |
