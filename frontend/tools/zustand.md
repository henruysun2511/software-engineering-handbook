# Zustand – Tổng hợp kiến thức đầy đủ (React / Next.js)

> Docs chính thức: https://zustand-demo.pmnd.rs | https://github.com/pmndrs/zustand

---

## 1. Zustand là gì?

Zustand là thư viện **state management** nhỏ gọn, đơn giản cho React. Không cần Provider bọc ngoài, không boilerplate, không Redux DevTools phức tạp — chỉ cần tạo store và dùng.

**So sánh với Redux Toolkit:**

| | Zustand | Redux Toolkit |
|---|---|---|
| Boilerplate | Rất ít | Trung bình |
| Provider | ❌ Không cần | ✅ Cần `<Provider>` |
| DevTools | ✅ Plugin | ✅ Tích hợp sẵn |
| Middleware | ✅ Có | ✅ Có |
| TypeScript | ✅ Xuất sắc | ✅ Tốt |
| Bundle size | ~1KB | ~11KB |
| Async | Viết thẳng trong store | createAsyncThunk |
| Learning curve | Rất thấp | Trung bình |

**Khi nào dùng Zustand:**
- State cần chia sẻ giữa nhiều component nhưng không muốn Redux phức tạp
- Project nhỏ-trung bình cần global state nhanh
- Thay thế Context API khi bị re-render nhiều

---

## 2. Cài đặt

```bash
npm install zustand
```

---

## 3. Tạo Store cơ bản

Store trong Zustand là một hook. Gọi `create()` với một function nhận `set` và trả về object chứa state + actions.

```typescript
// stores/counterStore.ts
import { create } from 'zustand';

interface CounterState {
  count: number;
  step: number;
  increment: () => void;
  decrement: () => void;
  addAmount: (amount: number) => void;
  reset: () => void;
  setStep: (step: number) => void;
}

export const useCounterStore = create<CounterState>((set) => ({
  // State
  count: 0,
  step: 1,

  // Actions — gọi set() để update state
  increment: () => set((state) => ({ count: state.count + state.step })),
  decrement: () => set((state) => ({ count: state.count - state.step })),
  addAmount: (amount) => set((state) => ({ count: state.count + amount })),
  reset: () => set({ count: 0 }),
  setStep: (step) => set({ step }),
}));
```

```typescript
// Component dùng store — không cần Provider!
import { useCounterStore } from '../stores/counterStore';

function Counter() {
  const count = useCounterStore((state) => state.count);
  const increment = useCounterStore((state) => state.increment);
  const decrement = useCounterStore((state) => state.decrement);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
    </div>
  );
}
```

---

## 4. Selector – Tránh re-render không cần thiết

Khi dùng `useStore()` không có selector, component sẽ re-render mỗi khi **bất kỳ** field nào trong store thay đổi. Dùng selector để chỉ subscribe field cần thiết.

```typescript
// ❌ Sai — re-render mỗi khi store thay đổi dù không dùng tới
const state = useCounterStore();

// ✅ Đúng — chỉ re-render khi count thay đổi
const count = useCounterStore((state) => state.count);

// ✅ Lấy nhiều giá trị — dùng useShallow để so sánh shallow
import { useShallow } from 'zustand/react/shallow';

const { count, step } = useCounterStore(
  useShallow((state) => ({ count: state.count, step: state.step }))
);
```

> **Lưu ý:** `useShallow` thay thế cho `shallow` của phiên bản cũ. Khi selector trả về object/array, cần dùng `useShallow` để tránh re-render liên tục do reference mới.

---

## 5. get() – Đọc state trong action

`get` cho phép đọc state hiện tại bên trong action mà không cần `set`.

```typescript
import { create } from 'zustand';

interface CartState {
  items: { id: string; quantity: number; price: number }[];
  totalAmount: () => number;
  addItem: (item: { id: string; price: number }) => void;
  removeItem: (id: string) => void;
}

export const useCartStore = create<CartState>((set, get) => ({
  items: [],

  // Computed value dùng get()
  totalAmount: () => {
    const { items } = get();  // đọc state hiện tại
    return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  },

  addItem: (newItem) => {
    const { items } = get();  // kiểm tra item đã tồn tại chưa
    const existing = items.find((i) => i.id === newItem.id);

    if (existing) {
      set((state) => ({
        items: state.items.map((i) =>
          i.id === newItem.id ? { ...i, quantity: i.quantity + 1 } : i
        ),
      }));
    } else {
      set((state) => ({
        items: [...state.items, { ...newItem, quantity: 1 }],
      }));
    }
  },

  removeItem: (id) =>
    set((state) => ({
      items: state.items.filter((i) => i.id !== id),
    })),
}));

// Dùng trong component
function Cart() {
  const items = useCartStore((s) => s.items);
  const totalAmount = useCartStore((s) => s.totalAmount);
  const removeItem = useCartStore((s) => s.removeItem);

  return (
    <div>
      <p>Total: {totalAmount()}đ</p>
      {items.map((item) => (
        <div key={item.id}>
          {item.id} x{item.quantity}
          <button onClick={() => removeItem(item.id)}>Xóa</button>
        </div>
      ))}
    </div>
  );
}
```

---

## 6. Async Actions

Zustand không cần middleware đặc biệt cho async. Viết thẳng `async/await` trong action.

```typescript
// stores/usersStore.ts
import { create } from 'zustand';

interface User {
  id: string;
  name: string;
  email: string;
}

interface UsersState {
  users: User[];
  selectedUser: User | null;
  isLoading: boolean;
  error: string | null;

  fetchUsers: () => Promise<void>;
  fetchUserById: (id: string) => Promise<void>;
  createUser: (data: Partial<User>) => Promise<void>;
  deleteUser: (id: string) => Promise<void>;
  clearError: () => void;
}

export const useUsersStore = create<UsersState>((set, get) => ({
  users: [],
  selectedUser: null,
  isLoading: false,
  error: null,

  fetchUsers: async () => {
    set({ isLoading: true, error: null });
    try {
      const res = await fetch('/api/users');
      if (!res.ok) throw new Error('Lỗi khi tải users');
      const users = await res.json();
      set({ users });
    } catch (err) {
      set({ error: (err as Error).message });
    } finally {
      set({ isLoading: false });
    }
  },

  fetchUserById: async (id) => {
    set({ isLoading: true });
    try {
      const res = await fetch(`/api/users/${id}`);
      const user = await res.json();
      set({ selectedUser: user });
    } catch (err) {
      set({ error: (err as Error).message });
    } finally {
      set({ isLoading: false });
    }
  },

  createUser: async (data) => {
    set({ isLoading: true });
    try {
      const res = await fetch('/api/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      });
      const newUser = await res.json();
      // Thêm vào list hiện tại thay vì fetch lại
      set((state) => ({ users: [...state.users, newUser] }));
    } catch (err) {
      set({ error: (err as Error).message });
    } finally {
      set({ isLoading: false });
    }
  },

  deleteUser: async (id) => {
    // Optimistic update — xóa ngay, rollback nếu lỗi
    const previousUsers = get().users;
    set((state) => ({ users: state.users.filter((u) => u.id !== id) }));

    try {
      await fetch(`/api/users/${id}`, { method: 'DELETE' });
    } catch (err) {
      set({ users: previousUsers, error: 'Xóa thất bại' }); // rollback
    }
  },

  clearError: () => set({ error: null }),
}));

// Component
function UserList() {
  const { users, isLoading, error, fetchUsers, deleteUser } = useUsersStore();

  useEffect(() => { fetchUsers(); }, []);

  if (isLoading) return <p>Loading...</p>;
  if (error) return <p>Lỗi: {error}</p>;

  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>
          {u.name}
          <button onClick={() => deleteUser(u.id)}>Xóa</button>
        </li>
      ))}
    </ul>
  );
}
```

---

## 7. Middleware – persist (Lưu vào localStorage)

`persist` middleware tự động lưu state vào `localStorage`/`sessionStorage` và khôi phục khi reload.

```typescript
// stores/authStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

interface AuthState {
  user: { id: string; name: string; email: string } | null;
  accessToken: string | null;
  isAuthenticated: boolean;

  login: (credentials: { email: string; password: string }) => Promise<void>;
  logout: () => void;
  setToken: (token: string) => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      accessToken: null,
      isAuthenticated: false,

      login: async ({ email, password }) => {
        const res = await fetch('/api/auth/login', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ email, password }),
        });
        const { user, accessToken } = await res.json();
        set({ user, accessToken, isAuthenticated: true });
      },

      logout: () => set({ user: null, accessToken: null, isAuthenticated: false }),

      setToken: (token) => set({ accessToken: token }),
    }),
    {
      name: 'auth-storage',               // key trong localStorage
      storage: createJSONStorage(() => localStorage),

      // Chỉ persist một số field (bảo mật)
      partialize: (state) => ({
        user: state.user,
        isAuthenticated: state.isAuthenticated,
        // KHÔNG persist accessToken vào localStorage
      }),

      // Chạy sau khi hydrate từ storage
      onRehydrateStorage: () => (state) => {
        console.log('Hydrated from storage:', state);
      },
    }
  )
);
```

```typescript
// Dùng sessionStorage thay localStorage
storage: createJSONStorage(() => sessionStorage),

// Dùng cookie (Next.js SSR-friendly)
import Cookies from 'js-cookie';

const cookieStorage = {
  getItem: (name: string) => Cookies.get(name) ?? null,
  setItem: (name: string, value: string) => Cookies.set(name, value, { expires: 7 }),
  removeItem: (name: string) => Cookies.remove(name),
};

storage: createJSONStorage(() => cookieStorage),
```

---

## 8. Middleware – devtools (Redux DevTools)

Kết nối Zustand với Redux DevTools Extension để debug state.

```typescript
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';

export const useCounterStore = create<CounterState>()(
  devtools(
    (set) => ({
      count: 0,
      increment: () => set(
        (state) => ({ count: state.count + 1 }),
        false,             // false = merge, true = replace toàn bộ state
        'counter/increment' // tên action hiển thị trong DevTools
      ),
      decrement: () => set(
        (state) => ({ count: state.count - 1 }),
        false,
        'counter/decrement'
      ),
    }),
    { name: 'CounterStore' } // tên store trong DevTools
  )
);
```

---

## 9. Middleware – immer (Mutate state trực tiếp)

`immer` middleware cho phép viết mutation syntax thay vì spread operator — tương tự Redux Toolkit.

```typescript
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';

interface TodoState {
  todos: { id: string; text: string; done: boolean }[];
  addTodo: (text: string) => void;
  toggleTodo: (id: string) => void;
  removeTodo: (id: string) => void;
  editTodo: (id: string, text: string) => void;
}

export const useTodoStore = create<TodoState>()(
  immer((set) => ({
    todos: [],

    addTodo: (text) =>
      set((state) => {
        // Mutate trực tiếp — Immer tạo copy phía sau
        state.todos.push({ id: crypto.randomUUID(), text, done: false });
      }),

    toggleTodo: (id) =>
      set((state) => {
        const todo = state.todos.find((t) => t.id === id);
        if (todo) todo.done = !todo.done;  // mutate trực tiếp
      }),

    removeTodo: (id) =>
      set((state) => {
        const index = state.todos.findIndex((t) => t.id === id);
        if (index !== -1) state.todos.splice(index, 1);
      }),

    editTodo: (id, text) =>
      set((state) => {
        const todo = state.todos.find((t) => t.id === id);
        if (todo) todo.text = text;
      }),
  }))
);
```

---

## 10. Kết hợp nhiều Middleware

```typescript
import { create } from 'zustand';
import { devtools, persist, immer } from 'zustand/middleware';

// Thứ tự middleware: devtools bọc ngoài cùng
export const useStore = create<State>()(
  devtools(
    persist(
      immer((set, get) => ({
        // state và actions
      })),
      { name: 'my-store' }
    ),
    { name: 'MyStore' }
  )
);
```

---

## 11. Chia nhỏ Store – Slice Pattern

Khi store lớn, chia thành các slice riêng rồi kết hợp.

```typescript
// stores/slices/authSlice.ts
import { StateCreator } from 'zustand';

export interface AuthSlice {
  user: User | null;
  isAuthenticated: boolean;
  login: (user: User) => void;
  logout: () => void;
}

export const createAuthSlice: StateCreator<
  AuthSlice & CartSlice,  // toàn bộ store type
  [],
  [],
  AuthSlice               // type của slice này
> = (set) => ({
  user: null,
  isAuthenticated: false,
  login: (user) => set({ user, isAuthenticated: true }),
  logout: () => set({ user: null, isAuthenticated: false }),
});

// stores/slices/cartSlice.ts
export interface CartSlice {
  cartItems: CartItem[];
  addToCart: (item: CartItem) => void;
  clearCart: () => void;
}

export const createCartSlice: StateCreator<
  AuthSlice & CartSlice,
  [],
  [],
  CartSlice
> = (set, get) => ({
  cartItems: [],
  addToCart: (item) =>
    set((state) => ({ cartItems: [...state.cartItems, item] })),
  clearCart: () => set({ cartItems: [] }),
});

// stores/useBoundStore.ts — Kết hợp tất cả slice
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';
import { createAuthSlice, AuthSlice } from './slices/authSlice';
import { createCartSlice, CartSlice } from './slices/cartSlice';

type BoundStore = AuthSlice & CartSlice;

export const useBoundStore = create<BoundStore>()(
  devtools(
    persist(
      (...args) => ({
        ...createAuthSlice(...args),
        ...createCartSlice(...args),
      }),
      { name: 'app-store' }
    )
  )
);

// Dùng trong component
const user = useBoundStore((s) => s.user);
const cartItems = useBoundStore((s) => s.cartItems);
```

---

## 12. Subscribe – Lắng nghe thay đổi bên ngoài Component

`subscribe` cho phép lắng nghe state change mà không gây re-render — hữu ích cho side effects.

```typescript
import { useEffect } from 'react';
import { useAuthStore } from '../stores/authStore';

// Dùng trong component
useEffect(() => {
  // Subscribe lắng nghe token thay đổi
  const unsub = useAuthStore.subscribe(
    (state) => state.accessToken,      // selector — chỉ lắng nghe token
    (token, previousToken) => {
      if (token) {
        // Cập nhật axios header khi token thay đổi
        axiosInstance.defaults.headers.Authorization = `Bearer ${token}`;
      } else {
        delete axiosInstance.defaults.headers.Authorization;
      }
    }
  );

  return unsub; // cleanup khi unmount
}, []);

// Subscribe ở cấp module (ngoài component) — không cần cleanup
useAuthStore.subscribe(
  (state) => state.isAuthenticated,
  (isAuthenticated) => {
    if (!isAuthenticated) {
      // Redirect về login khi logout
      window.location.href = '/login';
    }
  }
);
```

---

## 13. getState & setState – Dùng ngoài Component

Đôi khi cần đọc/ghi store từ bên ngoài React component (axios interceptor, utility function...).

```typescript
import { useAuthStore } from '../stores/authStore';
import { useCartStore } from '../stores/cartStore';

// axios interceptor — dùng getState() để lấy token
axiosInstance.interceptors.request.use((config) => {
  const token = useAuthStore.getState().accessToken;
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Auto logout khi 401
axiosInstance.interceptors.response.use(
  (res) => res,
  (error) => {
    if (error.response?.status === 401) {
      useAuthStore.getState().logout();   // gọi action trực tiếp
      useCartStore.getState().clearCart(); // xóa cart khi logout
    }
    return Promise.reject(error);
  }
);

// Utility function
export function isUserLoggedIn(): boolean {
  return useAuthStore.getState().isAuthenticated;
}
```

---

## 14. Zustand với Next.js (App Router)

Next.js App Router có thể gây **state bị chia sẻ giữa các request** khi dùng store singleton ở Server. Cần tạo store mới cho mỗi request.

### 14.1. Vấn đề với Next.js SSR

```typescript
// ❌ Vấn đề: store singleton bị share giữa các request server
export const useStore = create(() => ({ user: null }));
// Nếu user A login, user B request ngay sau có thể thấy user A's state
```

### 14.2. Giải pháp: createStore + Context

```typescript
// stores/createAppStore.ts
import { createStore } from 'zustand';

export type AppStore = ReturnType<typeof createAppStore>;

export const createAppStore = (initState = { count: 0 }) =>
  createStore<CounterState>()((set) => ({
    ...initState,
    increment: () => set((s) => ({ count: s.count + 1 })),
    decrement: () => set((s) => ({ count: s.count - 1 })),
  }));
```

```typescript
// providers/AppStoreProvider.tsx
'use client';

import { createContext, useContext, useRef } from 'react';
import { useStore } from 'zustand';
import { createAppStore, AppStore } from '../stores/createAppStore';

const AppStoreContext = createContext<AppStore | null>(null);

export function AppStoreProvider({
  children,
  initialState,
}: {
  children: React.ReactNode;
  initialState?: Partial<CounterState>;
}) {
  // useRef đảm bảo chỉ tạo store 1 lần per component tree
  const storeRef = useRef<AppStore>();
  if (!storeRef.current) {
    storeRef.current = createAppStore(initialState);
  }

  return (
    <AppStoreContext.Provider value={storeRef.current}>
      {children}
    </AppStoreContext.Provider>
  );
}

// Custom hook để dùng store
export function useAppStore<T>(selector: (state: CounterState) => T): T {
  const store = useContext(AppStoreContext);
  if (!store) throw new Error('useAppStore must be used within AppStoreProvider');
  return useStore(store, selector);
}
```

```typescript
// app/layout.tsx
import { AppStoreProvider } from '../providers/AppStoreProvider';

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <AppStoreProvider>
          {children}
        </AppStoreProvider>
      </body>
    </html>
  );
}

// Component dùng store
'use client';
import { useAppStore } from '../providers/AppStoreProvider';

function Counter() {
  const count = useAppStore((s) => s.count);
  const increment = useAppStore((s) => s.increment);
  return <button onClick={increment}>{count}</button>;
}
```

### 14.3. Truyền Server Data xuống Store

```typescript
// app/page.tsx — Server Component
export default async function Page() {
  // Fetch data ở server
  const initialData = await fetch('/api/data').then(r => r.json());

  return (
    // Truyền data server xuống Provider để khởi tạo store
    <AppStoreProvider initialState={{ count: initialData.count }}>
      <Counter />
    </AppStoreProvider>
  );
}
```

---

## 15. Tổng hợp Pattern thực tế

### 15.1. Auth Store hoàn chỉnh

```typescript
// stores/authStore.ts
import { create } from 'zustand';
import { persist, devtools } from 'zustand/middleware';

interface User { id: string; name: string; email: string; role: string; }

interface AuthState {
  user: User | null;
  accessToken: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;

  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  refreshToken: () => Promise<void>;
  updateProfile: (data: Partial<User>) => Promise<void>;
}

export const useAuthStore = create<AuthState>()(
  devtools(
    persist(
      (set, get) => ({
        user: null,
        accessToken: null,
        isAuthenticated: false,
        isLoading: false,

        login: async (email, password) => {
          set({ isLoading: true });
          try {
            const res = await fetch('/api/auth/login', {
              method: 'POST',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({ email, password }),
            });
            if (!res.ok) throw new Error('Sai email hoặc mật khẩu');
            const { user, accessToken } = await res.json();
            set({ user, accessToken, isAuthenticated: true });
          } finally {
            set({ isLoading: false });
          }
        },

        logout: async () => {
          await fetch('/api/auth/logout', { method: 'POST' });
          set({ user: null, accessToken: null, isAuthenticated: false });
        },

        refreshToken: async () => {
          const res = await fetch('/api/auth/refresh', { method: 'POST' });
          if (!res.ok) {
            get().logout();
            return;
          }
          const { accessToken } = await res.json();
          set({ accessToken });
        },

        updateProfile: async (data) => {
          const { user } = get();
          if (!user) return;
          const res = await fetch(`/api/users/${user.id}`, {
            method: 'PUT',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(data),
          });
          const updated = await res.json();
          set({ user: updated });
        },
      }),
      {
        name: 'auth-store',
        partialize: (state) => ({ user: state.user, isAuthenticated: state.isAuthenticated }),
      }
    ),
    { name: 'AuthStore' }
  )
);
```

### 15.2. Notification Store với subscribe

```typescript
// stores/notificationStore.ts
import { create } from 'zustand';

interface Notification {
  id: string;
  type: 'success' | 'error' | 'info' | 'warning';
  message: string;
  duration?: number;
}

interface NotificationState {
  notifications: Notification[];
  add: (notification: Omit<Notification, 'id'>) => void;
  remove: (id: string) => void;
  clear: () => void;
}

export const useNotificationStore = create<NotificationState>((set) => ({
  notifications: [],

  add: (notification) => {
    const id = crypto.randomUUID();
    set((state) => ({
      notifications: [...state.notifications, { ...notification, id }],
    }));

    // Tự động xóa sau duration
    if (notification.duration !== 0) {
      setTimeout(() => {
        useNotificationStore.getState().remove(id);
      }, notification.duration ?? 3000);
    }
  },

  remove: (id) =>
    set((state) => ({
      notifications: state.notifications.filter((n) => n.id !== id),
    })),

  clear: () => set({ notifications: [] }),
}));

// Helper functions gọi ngoài component
export const notify = {
  success: (message: string, duration?: number) =>
    useNotificationStore.getState().add({ type: 'success', message, duration }),
  error: (message: string) =>
    useNotificationStore.getState().add({ type: 'error', message, duration: 5000 }),
  info: (message: string) =>
    useNotificationStore.getState().add({ type: 'info', message }),
};

// Dùng ở bất kỳ đâu — kể cả trong axios interceptor
notify.error('Phiên đăng nhập hết hạn');
notify.success('Lưu thành công!');
```

---

## 16. Checklist Zustand

- [ ] Dùng **selector** khi `useStore` để tránh re-render thừa
- [ ] Dùng **`useShallow`** khi selector trả về object/array
- [ ] Dùng **`persist`** cho state cần giữ sau reload (auth, cart, theme)
- [ ] Dùng **`devtools`** trong development để debug
- [ ] Dùng **`immer`** khi state lồng sâu, nhiều nested update
- [ ] Gọi **`getState()`** khi cần store ngoài component (axios interceptor)
- [ ] Dùng **`subscribe`** cho side effects không cần re-render
- [ ] Với **Next.js App Router**: dùng `createStore` + Context, không dùng singleton
- [ ] **Không lưu accessToken** vào localStorage qua `persist` (dùng `partialize`)
- [ ] Chia store lớn thành **slice** để dễ maintain
