# Chương 8: Server State

Server state là dữ liệu **thuộc sở hữu của server** — ứng dụng chỉ hiển thị bản sao tạm thời. Bản chất của server state là: có thể thay đổi bởi người dùng khác, cần được đồng bộ định kỳ, và có vòng đời riêng (loading, stale, error).

Quản lý server state bằng `useState` + `useEffect` thủ công rất tốn công: phải tự xử lý loading, error, cache, invalidation, retry, và race condition. Các thư viện chuyên biệt giải quyết toàn bộ vấn đề này.

---

## 8.1. TanStack Query ⭐⭐⭐⭐⭐

TanStack Query (trước đây là React Query) là thư viện quản lý server state phổ biến nhất trong hệ sinh thái React. Nó cung cấp caching thông minh, background refetching, và synchronization tự động.

### Cài đặt và cấu hình

```tsx
// app/providers.tsx
"use client";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";
import { useState } from "react";

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000, // 1 phút
            retry: 2,
          },
        },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

---

### useQuery

`useQuery` dùng để **fetch và cache dữ liệu**. Mỗi query được định danh bằng `queryKey` — key này quyết định cache nào được dùng, invalidate, hoặc refetch.

```tsx
import { useQuery } from "@tanstack/react-query";

interface User {
  id: number;
  name: string;
  email: string;
}

async function fetchUsers(): Promise<User[]> {
  const res = await fetch("/api/users");
  if (!res.ok) throw new Error("Không thể lấy danh sách users");
  return res.json();
}

function UserList() {
  const {
    data: users,
    isLoading,
    isError,
    error,
    isFetching,   // true khi đang refetch (kể cả có data cũ)
    refetch,
  } = useQuery({
    queryKey: ["users"],          // Định danh cache
    queryFn: fetchUsers,          // Hàm fetch dữ liệu
    staleTime: 5 * 60 * 1000,    // 5 phút trước khi coi data là stale
  });

  if (isLoading) return <p>Đang tải...</p>;
  if (isError) return <p>Lỗi: {error.message}</p>;

  return (
    <div>
      {isFetching && <span>Đang cập nhật...</span>}
      <ul>
        {users?.map((user) => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
      <button onClick={() => refetch()}>Tải lại</button>
    </div>
  );
}
```

Query có tham số — `queryKey` phải chứa tham số để cache riêng biệt:

```tsx
async function fetchUser(id: number): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error("Not found");
  return res.json();
}

function UserProfile({ userId }: { userId: number }) {
  const { data: user } = useQuery({
    queryKey: ["users", userId],     // Mỗi userId có cache riêng
    queryFn: () => fetchUser(userId),
    enabled: userId > 0,             // Chỉ fetch khi có userId hợp lệ
  });

  return <h1>{user?.name}</h1>;
}
```

---

### useMutation

`useMutation` dùng để **thực hiện các thao tác thay đổi dữ liệu** (POST, PUT, PATCH, DELETE). Không có cache, không tự động chạy — chỉ chạy khi gọi `mutate()`.

```tsx
import { useMutation, useQueryClient } from "@tanstack/react-query";

async function createUser(data: { name: string; email: string }): Promise<User> {
  const res = await fetch("/api/users", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(data),
  });
  if (!res.ok) throw new Error("Tạo user thất bại");
  return res.json();
}

function CreateUserForm() {
  const queryClient = useQueryClient();

  const { mutate, isPending, isError, error } = useMutation({
    mutationFn: createUser,
    onSuccess: () => {
      // Invalidate cache — useQuery["users"] sẽ tự fetch lại
      queryClient.invalidateQueries({ queryKey: ["users"] });
    },
    onError: (err) => {
      console.error("Lỗi:", err.message);
    },
  });

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const form = new FormData(e.currentTarget);
    mutate({
      name: form.get("name") as string,
      email: form.get("email") as string,
    });
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" required />
      <input name="email" type="email" required />
      {isError && <p className="error">{error.message}</p>}
      <button type="submit" disabled={isPending}>
        {isPending ? "Đang tạo..." : "Tạo user"}
      </button>
    </form>
  );
}
```

---

### invalidateQueries

`invalidateQueries` đánh dấu cache của một hoặc nhiều query là **stale**, kích hoạt refetch tự động nếu query đang được dùng trên màn hình.

```tsx
const queryClient = useQueryClient();

// Invalidate đúng một query
queryClient.invalidateQueries({ queryKey: ["users"] });

// Invalidate tất cả query bắt đầu bằng "users" (partial match)
queryClient.invalidateQueries({ queryKey: ["users"] });
// Ảnh hưởng: ["users"], ["users", 1], ["users", 2, "posts"]

// Invalidate nhiều query cùng lúc sau khi xóa user
const deleteMutation = useMutation({
  mutationFn: deleteUser,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ["users"] });
    queryClient.invalidateQueries({ queryKey: ["stats"] });
  },
});
```

---

### staleTime và gcTime

Đây là hai thông số quan trọng nhất điều chỉnh hành vi cache của TanStack Query.

**staleTime** — Thời gian (ms) data được coi là **tươi (fresh)**. Trong khoảng này, query sẽ không fetch lại dù component remount hay window refocus.

**gcTime** (garbage collection time, trước đây là `cacheTime`) — Thời gian data **không còn được dùng** trước khi bị xóa khỏi bộ nhớ. Đồng hồ bắt đầu khi không còn subscriber nào.

```
Fetch xong
   ↓
[← staleTime →] Data là FRESH — không refetch tự động
   ↓
Data trở thành STALE — refetch khi window focus, remount, ...
   ↓
Tất cả component unmount → bắt đầu đếm gcTime
   ↓
[← gcTime →] Cache vẫn tồn tại (hiển thị data cũ ngay nếu remount)
   ↓
Cache bị xóa khỏi bộ nhớ
```

```tsx
useQuery({
  queryKey: ["products"],
  queryFn: fetchProducts,
  staleTime: 5 * 60 * 1000,   // 5 phút — data fresh, không refetch vô ích
  gcTime: 10 * 60 * 1000,     // 10 phút — giữ cache 10 phút sau khi unmount
});
```

| | staleTime | gcTime |
|---|---|---|
| Mô tả | Bao lâu data còn fresh | Bao lâu giữ cache sau khi không dùng |
| Mặc định | 0 (stale ngay) | 5 phút |
| Giá trị cao | Ít refetch hơn | Giữ cache lâu hơn |
| Dùng khi | Data ít thay đổi | Muốn quay lại nhanh không loading |

---

### retry

Khi query thất bại, TanStack Query tự động thử lại. Mặc định thử lại **3 lần** với exponential backoff.

```tsx
useQuery({
  queryKey: ["users"],
  queryFn: fetchUsers,
  retry: 2,                          // Số lần thử lại tối đa
  retryDelay: (attempt) =>           // Thời gian chờ trước mỗi lần retry
    Math.min(1000 * 2 ** attempt, 30000), // 1s, 2s, 4s, ... tối đa 30s
});

// Không retry (ví dụ: lỗi 404 không có ý nghĩa retry)
useQuery({
  queryKey: ["user", id],
  queryFn: () => fetchUser(id),
  retry: (failureCount, error) => {
    if (error.status === 404) return false; // Không retry
    return failureCount < 3;
  },
});
```

---

### Optimistic Update

Optimistic update là kỹ thuật **cập nhật UI ngay lập tức** trước khi server phản hồi, tạo cảm giác ứng dụng nhanh hơn. Nếu server trả về lỗi, UI sẽ rollback về trạng thái cũ.

```tsx
function TodoList() {
  const queryClient = useQueryClient();

  const toggleTodo = useMutation({
    mutationFn: (todo: Todo) =>
      fetch(`/api/todos/${todo.id}`, {
        method: "PATCH",
        body: JSON.stringify({ completed: !todo.completed }),
      }),

    onMutate: async (todo) => {
      // 1. Hủy bất kỳ refetch nào đang chạy
      await queryClient.cancelQueries({ queryKey: ["todos"] });

      // 2. Lưu snapshot state hiện tại
      const previous = queryClient.getQueryData<Todo[]>(["todos"]);

      // 3. Cập nhật cache ngay lập tức (optimistic)
      queryClient.setQueryData<Todo[]>(["todos"], (old) =>
        old?.map((t) =>
          t.id === todo.id ? { ...t, completed: !t.completed } : t
        )
      );

      return { previous }; // Truyền snapshot vào onError
    },

    onError: (_err, _todo, context) => {
      // Rollback nếu lỗi
      if (context?.previous) {
        queryClient.setQueryData(["todos"], context.previous);
      }
    },

    onSettled: () => {
      // Dù thành công hay thất bại đều đồng bộ lại với server
      queryClient.invalidateQueries({ queryKey: ["todos"] });
    },
  });
}
```

---

### Prefetch

Prefetch cho phép fetch và cache dữ liệu **trước khi component cần đến**, giúp trang hiển thị ngay lập tức thay vì loading.

```tsx
// Prefetch khi hover vào link
function UserListItem({ user }: { user: User }) {
  const queryClient = useQueryClient();

  function prefetchUserDetail() {
    queryClient.prefetchQuery({
      queryKey: ["users", user.id],
      queryFn: () => fetchUser(user.id),
      staleTime: 5 * 60 * 1000,
    });
  }

  return (
    <li onMouseEnter={prefetchUserDetail}>
      <Link href={`/users/${user.id}`}>{user.name}</Link>
    </li>
  );
}
```

Prefetch trên server với Next.js (Hydration):

```tsx
// app/users/page.tsx (Server Component)
import {
  dehydrate,
  HydrationBoundary,
  QueryClient,
} from "@tanstack/react-query";

export default async function UsersPage() {
  const queryClient = new QueryClient();

  // Prefetch trên server — client không cần fetch lại
  await queryClient.prefetchQuery({
    queryKey: ["users"],
    queryFn: fetchUsers,
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <UserList /> {/* Client Component — có sẵn data, không loading */}
    </HydrationBoundary>
  );
}
```

---

## 8.2. RTK Query

RTK Query là giải pháp data fetching tích hợp sẵn trong Redux Toolkit. Nếu ứng dụng đã dùng Redux Toolkit, RTK Query là lựa chọn tự nhiên để tránh thêm dependency.

### Định nghĩa API

```tsx
// store/api/userApi.ts
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";

export const userApi = createApi({
  reducerPath: "userApi",
  baseQuery: fetchBaseQuery({
    baseUrl: "/api",
    prepareHeaders: (headers) => {
      const token = localStorage.getItem("token");
      if (token) headers.set("Authorization", `Bearer ${token}`);
      return headers;
    },
  }),
  tagTypes: ["User"],                 // Dùng để invalidate cache
  endpoints: (builder) => ({
    getUsers: builder.query<User[], void>({
      query: () => "/users",
      providesTags: ["User"],         // Cache được gắn tag "User"
    }),
    getUserById: builder.query<User, number>({
      query: (id) => `/users/${id}`,
      providesTags: (_result, _error, id) => [{ type: "User", id }],
    }),
    createUser: builder.mutation<User, Partial<User>>({
      query: (body) => ({
        url: "/users",
        method: "POST",
        body,
      }),
      invalidatesTags: ["User"],      // Tự động invalidate cache có tag "User"
    }),
    deleteUser: builder.mutation<void, number>({
      query: (id) => ({
        url: `/users/${id}`,
        method: "DELETE",
      }),
      invalidatesTags: (_result, _error, id) => [{ type: "User", id }],
    }),
  }),
});

// Export auto-generated hooks
export const {
  useGetUsersQuery,
  useGetUserByIdQuery,
  useCreateUserMutation,
  useDeleteUserMutation,
} = userApi;
```

Thêm vào store:

```tsx
// store/index.ts
import { configureStore } from "@reduxjs/toolkit";
import { userApi } from "./api/userApi";

export const store = configureStore({
  reducer: {
    [userApi.reducerPath]: userApi.reducer,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(userApi.middleware),
});
```

Dùng trong component:

```tsx
function UserList() {
  const { data: users, isLoading, isError } = useGetUsersQuery();
  const [deleteUser] = useDeleteUserMutation();

  if (isLoading) return <p>Đang tải...</p>;
  if (isError) return <p>Có lỗi xảy ra</p>;

  return (
    <ul>
      {users?.map((user) => (
        <li key={user.id}>
          {user.name}
          <button onClick={() => deleteUser(user.id)}>Xóa</button>
        </li>
      ))}
    </ul>
  );
}
```

---

### So sánh TanStack Query vs RTK Query

| Tiêu chí | TanStack Query | RTK Query |
|---|---|---|
| Độc lập | Hoàn toàn | Phụ thuộc Redux Toolkit |
| Bundle size | ~13 KB | Tích hợp trong RTK |
| Cấu hình | Linh hoạt, ít cấu hình | Cần định nghĩa API trước |
| Hooks | Tạo thủ công (`useQuery`) | Tự động generate từ endpoint |
| Cache key | `queryKey` array | Endpoint + argument |
| Invalidation | `invalidateQueries()` thủ công | `invalidatesTags` khai báo sẵn |
| Optimistic update | Hỗ trợ đầy đủ | Hỗ trợ nhưng phức tạp hơn |
| DevTools | React Query Devtools | Redux DevTools |
| Middleware/transform | `select`, `transform` | `transformResponse` |
| Dùng khi | Dự án không dùng Redux | Dự án đã dùng Redux Toolkit |

**Tóm lại:** TanStack Query là lựa chọn mặc định cho hầu hết dự án nhờ tính linh hoạt và DX tốt. RTK Query phù hợp khi dự án đã có Redux Toolkit và muốn giảm số lượng dependency.
