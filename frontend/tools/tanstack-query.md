# TanStack Query – Tổng hợp kiến thức đầy đủ (React / Next.js)

> Docs chính thức: https://tanstack.com/query/latest

---

## 1. TanStack Query là gì?

TanStack Query (trước đây là React Query) là thư viện **server state management** — chuyên xử lý việc fetch, cache, sync và update dữ liệu từ server. Nó **không thay thế** Zustand/Redux mà bổ sung cho chúng.

**Phân biệt Client State vs Server State:**

| | Client State | Server State |
|---|---|---|
| Ví dụ | Modal open, theme, form input | Users, Products, Orders từ API |
| Lưu ở đâu | Zustand / Redux | TanStack Query cache |
| Đặc điểm | App kiểm soát hoàn toàn | Có thể thay đổi từ server bất cứ lúc nào |
| Vấn đề | Ít | Stale data, loading, caching, re-fetch |

**TanStack Query giải quyết:**
- Tự động cache và deduplication request
- Background refetch khi data stale
- Loading / error / success state tự động
- Pagination và infinite scroll
- Optimistic updates
- Prefetching

---

## 2. Cài đặt

```bash
npm install @tanstack/react-query
# DevTools (optional nhưng rất nên dùng)
npm install @tanstack/react-query-devtools
```

---

## 3. Setup QueryClient & Provider

```typescript
// app/providers/QueryProvider.tsx (Next.js App Router)
'use client';

import { useState } from 'react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

export function QueryProvider({ children }: { children: React.ReactNode }) {
  // useState đảm bảo mỗi request/user có QueryClient riêng (Next.js SSR)
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000,        // data fresh trong 1 phút
            gcTime: 5 * 60 * 1000,       // giữ cache 5 phút sau khi unused
            retry: 3,                    // retry 3 lần khi lỗi
            retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30000),
            refetchOnWindowFocus: false, // không refetch khi focus tab
          },
          mutations: {
            retry: 0, // mutation không retry mặc định
          },
        },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      {process.env.NODE_ENV === 'development' && (
        <ReactQueryDevtools initialIsOpen={false} />
      )}
    </QueryClientProvider>
  );
}

// app/layout.tsx
import { QueryProvider } from './providers/QueryProvider';

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <QueryProvider>{children}</QueryProvider>
      </body>
    </html>
  );
}
```

---

## 4. useQuery – Fetch dữ liệu

`useQuery` là hook cơ bản nhất — dùng để **đọc dữ liệu** từ server.

### 4.1. Cú pháp cơ bản

```typescript
import { useQuery } from '@tanstack/react-query';

function UserList() {
  const {
    data,           // dữ liệu trả về
    isLoading,      // true khi fetch lần đầu, chưa có data
    isFetching,     // true khi đang fetch (kể cả background refetch)
    isError,        // true khi có lỗi
    error,          // error object
    isSuccess,      // true khi fetch thành công
    refetch,        // gọi lại fetch thủ công
    status,         // 'pending' | 'error' | 'success'
    fetchStatus,    // 'fetching' | 'paused' | 'idle'
  } = useQuery({
    queryKey: ['users'],        // key duy nhất cho cache
    queryFn: () => fetch('/api/users').then(r => r.json()),
  });

  if (isLoading) return <Spinner />;
  if (isError) return <p>Lỗi: {error.message}</p>;

  return (
    <ul>
      {data?.map((user) => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
}
```

### 4.2. Query Key – Trái tim của caching

Query Key là mảng định danh cache. Khi key thay đổi → fetch lại tự động.

```typescript
// Key tĩnh
useQuery({ queryKey: ['users'], queryFn: fetchUsers })

// Key động — thay đổi khi tham số thay đổi
useQuery({ queryKey: ['users', userId], queryFn: () => fetchUser(userId) })
useQuery({ queryKey: ['users', { page, limit, search }], queryFn: () => fetchUsers({ page, limit, search }) })
useQuery({ queryKey: ['products', categoryId, 'reviews'], queryFn: ... })

// Convention — tổ chức query key thành factory
export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  list: (filters: object) => [...userKeys.lists(), filters] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail: (id: string) => [...userKeys.details(), id] as const,
};

// Dùng
useQuery({ queryKey: userKeys.detail(userId), queryFn: () => fetchUser(userId) })
// Invalidate tất cả user queries
queryClient.invalidateQueries({ queryKey: userKeys.all })
// Invalidate chỉ detail queries
queryClient.invalidateQueries({ queryKey: userKeys.details() })
```

### 4.3. Query Function

```typescript
// queryFn nhận QueryFunctionContext
const { data } = useQuery({
  queryKey: ['users', userId],
  queryFn: async ({ queryKey, signal }) => {
    const [, id] = queryKey;  // destructure key

    // signal để cancel request khi component unmount
    const res = await fetch(`/api/users/${id}`, { signal });
    if (!res.ok) throw new Error('Fetch failed');  // phải throw để trigger error state
    return res.json();
  },
});
```

### 4.4. Tổ chức với API layer riêng

```typescript
// api/users.api.ts
import axios from './axiosInstance';

export const usersApi = {
  getAll: (params?: { page?: number; limit?: number; search?: string }) =>
    axios.get('/users', { params }).then(r => r.data),

  getById: (id: string) =>
    axios.get(`/users/${id}`).then(r => r.data),

  create: (data: CreateUserDto) =>
    axios.post('/users', data).then(r => r.data),

  update: (id: string, data: UpdateUserDto) =>
    axios.put(`/users/${id}`, data).then(r => r.data),

  delete: (id: string) =>
    axios.delete(`/users/${id}`).then(r => r.data),
};

// hooks/useUsers.ts — Custom hooks bọc useQuery
export function useUsers(params?: { page?: number; search?: string }) {
  return useQuery({
    queryKey: userKeys.list(params ?? {}),
    queryFn: () => usersApi.getAll(params),
  });
}

export function useUser(id: string) {
  return useQuery({
    queryKey: userKeys.detail(id),
    queryFn: () => usersApi.getById(id),
    enabled: !!id,  // không fetch nếu id rỗng
  });
}
```

---

## 5. useQuery Options quan trọng

```typescript
useQuery({
  queryKey: ['users'],
  queryFn: fetchUsers,

  // ── Stale & Cache ──────────────────────────────
  staleTime: 5 * 60 * 1000,   // data fresh 5 phút — không refetch trong thời gian này
  gcTime: 10 * 60 * 1000,     // giữ cache 10 phút sau khi không dùng (v5, trước là cacheTime)

  // ── Enabled / Disabled ─────────────────────────
  enabled: !!userId,           // chỉ fetch khi userId có giá trị
  enabled: isAuthenticated,    // chỉ fetch khi đã đăng nhập

  // ── Refetch behavior ───────────────────────────
  refetchOnWindowFocus: true,  // refetch khi user quay lại tab
  refetchOnReconnect: true,    // refetch khi có lại internet
  refetchOnMount: true,        // refetch khi component mount
  refetchInterval: 30_000,     // polling mỗi 30 giây
  refetchIntervalInBackground: false, // không poll khi tab ẩn

  // ── Retry ──────────────────────────────────────
  retry: 3,                    // retry 3 lần khi lỗi
  retryDelay: 1000,            // chờ 1 giây giữa các retry
  retryDelay: (attempt) => Math.min(attempt * 1000, 30000), // exponential

  // ── Data transformation ────────────────────────
  select: (data) => data.users,  // transform data trước khi trả về component
  select: (data) => data.sort((a, b) => a.name.localeCompare(b.name)),

  // ── Placeholder & Initial data ─────────────────
  placeholderData: { users: [], total: 0 },   // dùng khi chưa có data
  placeholderData: keepPreviousData,           // giữ data cũ khi key thay đổi (pagination)
  initialData: cachedUsers,                    // data ban đầu (tính là fresh)
  initialDataUpdatedAt: Date.now() - 60_000,  // đánh dấu initialData đã 1 phút tuổi
});
```

---

## 6. useMutation – Thay đổi dữ liệu

`useMutation` dùng để **ghi dữ liệu**: POST, PUT, PATCH, DELETE.

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';

function CreateUserForm() {
  const queryClient = useQueryClient();

  const {
    mutate,          // gọi mutation (fire and forget)
    mutateAsync,     // gọi mutation với async/await
    isPending,       // đang xử lý
    isError,
    isSuccess,
    error,
    data,            // kết quả trả về
    reset,           // reset về trạng thái idle
  } = useMutation({
    mutationFn: (newUser: CreateUserDto) => usersApi.create(newUser),

    onSuccess: (data, variables, context) => {
      // data: kết quả từ server
      // variables: tham số đã truyền vào mutate()

      // Invalidate cache → tự động refetch list
      queryClient.invalidateQueries({ queryKey: userKeys.lists() });

      // Hoặc update cache trực tiếp (không cần refetch)
      queryClient.setQueryData(userKeys.detail(data.id), data);
    },

    onError: (error, variables, context) => {
      console.error('Failed to create user:', error.message);
    },

    onSettled: (data, error, variables) => {
      // Chạy dù thành công hay thất bại
      queryClient.invalidateQueries({ queryKey: userKeys.lists() });
    },
  });

  const handleSubmit = (formData: CreateUserDto) => {
    // Cách 1: mutate (callback style)
    mutate(formData, {
      onSuccess: () => toast.success('Tạo user thành công!'),
      onError: (err) => toast.error(err.message),
    });
  };

  const handleSubmitAsync = async (formData: CreateUserDto) => {
    // Cách 2: mutateAsync (promise style)
    try {
      const newUser = await mutateAsync(formData);
      toast.success(`Đã tạo ${newUser.name}`);
      router.push(`/users/${newUser.id}`);
    } catch (err) {
      toast.error('Tạo thất bại');
    }
  };

  return (
    <form onSubmit={...}>
      <button type="submit" disabled={isPending}>
        {isPending ? 'Đang tạo...' : 'Tạo user'}
      </button>
    </form>
  );
}
```

---

## 7. Invalidate & Update Cache

Sau khi mutation, cần đồng bộ cache với server. Có 2 cách:

### 7.1. invalidateQueries – Đơn giản, luôn đúng

```typescript
const queryClient = useQueryClient();

// Invalidate tất cả queries có key bắt đầu bằng 'users'
queryClient.invalidateQueries({ queryKey: ['users'] });

// Invalidate chính xác 1 query
queryClient.invalidateQueries({ queryKey: userKeys.list({}) });

// Invalidate với predicate
queryClient.invalidateQueries({
  predicate: (query) => query.queryKey[0] === 'users',
});
```

### 7.2. setQueryData – Nhanh hơn, không cần refetch

```typescript
// Set toàn bộ data
queryClient.setQueryData(userKeys.detail(userId), updatedUser);

// Update một phần data
queryClient.setQueryData(userKeys.detail(userId), (oldData) => ({
  ...oldData,
  name: 'New Name',
}));

// Thêm item mới vào list
queryClient.setQueryData(userKeys.lists(), (oldData) => [
  ...oldData,
  newUser,
]);
```

### 7.3. Kết hợp cả hai

```typescript
useMutation({
  mutationFn: updateUser,
  onSuccess: (updatedUser) => {
    // Update detail cache ngay lập tức
    queryClient.setQueryData(userKeys.detail(updatedUser.id), updatedUser);
    // Invalidate list để refetch
    queryClient.invalidateQueries({ queryKey: userKeys.lists() });
  },
});
```

---

## 8. Optimistic Update

Cập nhật UI ngay lập tức trước khi server phản hồi — rollback nếu thất bại.

```typescript
const queryClient = useQueryClient();

const updateUserMutation = useMutation({
  mutationFn: ({ id, data }: { id: string; data: Partial<User> }) =>
    usersApi.update(id, data),

  onMutate: async ({ id, data }) => {
    // 1. Cancel query đang chạy để tránh overwrite optimistic update
    await queryClient.cancelQueries({ queryKey: userKeys.detail(id) });

    // 2. Lưu data cũ để rollback
    const previousUser = queryClient.getQueryData(userKeys.detail(id));

    // 3. Update cache ngay lập tức (optimistic)
    queryClient.setQueryData(userKeys.detail(id), (old: User) => ({
      ...old,
      ...data,
    }));

    // 4. Return context để dùng trong onError
    return { previousUser };
  },

  onError: (err, { id }, context) => {
    // Rollback nếu lỗi
    if (context?.previousUser) {
      queryClient.setQueryData(userKeys.detail(id), context.previousUser);
    }
  },

  onSettled: (data, error, { id }) => {
    // Luôn sync lại với server sau cùng
    queryClient.invalidateQueries({ queryKey: userKeys.detail(id) });
  },
});
```

---

## 9. Pagination

### 9.1. Offset-based Pagination

```typescript
import { keepPreviousData } from '@tanstack/react-query';

function UserTable() {
  const [page, setPage] = useState(1);
  const limit = 10;

  const { data, isLoading, isFetching, isPlaceholderData } = useQuery({
    queryKey: ['users', 'list', { page, limit }],
    queryFn: () => usersApi.getAll({ page, limit }),
    placeholderData: keepPreviousData, // giữ data trang cũ khi đang load trang mới
  });

  return (
    <div>
      {isFetching && <span>Đang tải...</span>}
      <table>
        {data?.users.map(user => <tr key={user.id}><td>{user.name}</td></tr>)}
      </table>
      <div>
        <button onClick={() => setPage(p => p - 1)} disabled={page === 1}>Trước</button>
        <span>Trang {page} / {Math.ceil((data?.total ?? 0) / limit)}</span>
        <button onClick={() => setPage(p => p + 1)} disabled={isPlaceholderData || !data?.hasNextPage}>Sau</button>
      </div>
    </div>
  );
}
```

### 9.2. Infinite Scroll với useInfiniteQuery

```typescript
import { useInfiniteQuery } from '@tanstack/react-query';

function InfinitePostList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
  } = useInfiniteQuery({
    queryKey: ['posts', 'infinite'],
    queryFn: ({ pageParam }) =>
      postsApi.getAll({ cursor: pageParam, limit: 10 }),

    initialPageParam: undefined,  // v5: bắt buộc khai báo
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
  });

  // Flatten tất cả pages thành 1 mảng
  const posts = data?.pages.flatMap(page => page.posts) ?? [];

  // Intersection Observer để load more khi scroll tới cuối
  const { ref, entry } = useIntersection({ threshold: 0.5 });

  useEffect(() => {
    if (entry?.isIntersecting && hasNextPage && !isFetchingNextPage) {
      fetchNextPage();
    }
  }, [entry?.isIntersecting]);

  if (isLoading) return <Spinner />;

  return (
    <div>
      {posts.map(post => <PostCard key={post.id} post={post} />)}
      <div ref={ref}>
        {isFetchingNextPage ? <Spinner /> : hasNextPage ? <p>Cuộn để xem thêm</p> : <p>Đã hết</p>}
      </div>
    </div>
  );
}
```

---

## 10. Prefetching

Fetch data trước khi user cần — cải thiện UX đáng kể.

```typescript
import { useQueryClient } from '@tanstack/react-query';

// Prefetch khi hover link
function UserListItem({ userId }: { userId: string }) {
  const queryClient = useQueryClient();

  const prefetchUser = () => {
    queryClient.prefetchQuery({
      queryKey: userKeys.detail(userId),
      queryFn: () => usersApi.getById(userId),
      staleTime: 60_000, // không prefetch lại nếu data còn fresh
    });
  };

  return (
    <li onMouseEnter={prefetchUser} onFocus={prefetchUser}>
      <Link href={`/users/${userId}`}>Xem chi tiết</Link>
    </li>
  );
}

// Prefetch trong Next.js Server Component
// app/users/page.tsx
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query';

export default async function UsersPage() {
  const queryClient = new QueryClient();

  // Prefetch trên server — không cần fetch lại ở client
  await queryClient.prefetchQuery({
    queryKey: userKeys.lists(),
    queryFn: () => usersApi.getAll(),
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <UserList />
    </HydrationBoundary>
  );
}
```

---

## 11. useQueryClient – Thao tác cache thủ công

```typescript
import { useQueryClient } from '@tanstack/react-query';

const queryClient = useQueryClient();

// Đọc cache
const cachedUser = queryClient.getQueryData(userKeys.detail(userId));

// Ghi cache thủ công
queryClient.setQueryData(userKeys.detail(userId), newUserData);

// Xóa cache
queryClient.removeQueries({ queryKey: userKeys.detail(userId) });

// Reset về trạng thái ban đầu (xóa data + error)
queryClient.resetQueries({ queryKey: ['users'] });

// Cancel tất cả queries đang chạy
queryClient.cancelQueries({ queryKey: ['users'] });

// Xóa toàn bộ cache (khi logout)
queryClient.clear();

// Kiểm tra query đang fetch không
const isFetching = queryClient.isFetching({ queryKey: ['users'] });

// Lấy tất cả cached queries
const queries = queryClient.getQueriesData({ queryKey: ['users'] });
```

---

## 12. Dependent Queries

```typescript
function UserProfile({ userId }: { userId: string }) {
  // Query 1: Lấy user
  const { data: user } = useQuery({
    queryKey: userKeys.detail(userId),
    queryFn: () => usersApi.getById(userId),
  });

  // Query 2: Chỉ chạy khi Query 1 xong và có organizationId
  const { data: organization } = useQuery({
    queryKey: ['organizations', user?.organizationId],
    queryFn: () => orgsApi.getById(user!.organizationId),
    enabled: !!user?.organizationId,  // disabled cho đến khi có data
  });

  // Query 3: Fetch nhiều items song song
  const todoQueries = useQueries({
    queries: user?.todoIds?.map((id) => ({
      queryKey: ['todos', id],
      queryFn: () => todosApi.getById(id),
    })) ?? [],
  });

  return <div>...</div>;
}
```

---

## 13. Error Handling

```typescript
// Global error handler trong QueryClient
new QueryClient({
  queryCache: new QueryCache({
    onError: (error, query) => {
      // Chỉ show toast cho background refetch (đã có data cũ)
      if (query.state.data !== undefined) {
        toast.error(`Lỗi cập nhật: ${error.message}`);
      }
    },
  }),
  mutationCache: new MutationCache({
    onError: (error) => {
      toast.error(`Lỗi: ${error.message}`);
    },
  }),
});

// Error Boundary cho từng query
import { QueryErrorResetBoundary } from '@tanstack/react-query';
import { ErrorBoundary } from 'react-error-boundary';

function App() {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundary
          onReset={reset}
          fallbackRender={({ error, resetErrorBoundary }) => (
            <div>
              <p>Đã xảy ra lỗi: {error.message}</p>
              <button onClick={resetErrorBoundary}>Thử lại</button>
            </div>
          )}
        >
          <UserList />
        </ErrorBoundary>
      )}
    </QueryErrorResetBoundary>
  );
}

// throwOnError option
useQuery({
  queryKey: ['users'],
  queryFn: fetchUsers,
  throwOnError: (error) => error.status >= 500, // chỉ throw lỗi 5xx lên Error Boundary
});
```

---

## 14. Suspense Mode

```typescript
// useSuspenseQuery — throw Promise khi loading
import { useSuspenseQuery } from '@tanstack/react-query';

function UserDetail({ userId }: { userId: string }) {
  // Không cần check isLoading — Suspense xử lý
  // Không cần check isError — ErrorBoundary xử lý
  const { data: user } = useSuspenseQuery({
    queryKey: userKeys.detail(userId),
    queryFn: () => usersApi.getById(userId),
  });

  return <div>{user.name}</div>;
}

// Dùng với Suspense + ErrorBoundary
function UserPage({ userId }: { userId: string }) {
  return (
    <ErrorBoundary fallback={<p>Lỗi rồi!</p>}>
      <Suspense fallback={<Spinner />}>
        <UserDetail userId={userId} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

---

## 15. Custom Hooks Pattern – Best Practice

```typescript
// hooks/useUsers.ts — Tổ chức toàn bộ logic vào 1 file

// ── Query Keys ────────────────────────────────────
export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  list: (filters: object) => [...userKeys.lists(), filters] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail: (id: string) => [...userKeys.details(), id] as const,
};

// ── Queries ───────────────────────────────────────
export function useUsers(params?: { page?: number; limit?: number; search?: string }) {
  return useQuery({
    queryKey: userKeys.list(params ?? {}),
    queryFn: () => usersApi.getAll(params),
    placeholderData: keepPreviousData,
    staleTime: 30_000,
  });
}

export function useUser(id: string) {
  return useQuery({
    queryKey: userKeys.detail(id),
    queryFn: () => usersApi.getById(id),
    enabled: !!id,
  });
}

// ── Mutations ─────────────────────────────────────
export function useCreateUser() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: usersApi.create,
    onSuccess: (newUser) => {
      queryClient.invalidateQueries({ queryKey: userKeys.lists() });
      queryClient.setQueryData(userKeys.detail(newUser.id), newUser);
    },
  });
}

export function useUpdateUser() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Partial<User> }) =>
      usersApi.update(id, data),
    onSuccess: (updatedUser) => {
      queryClient.setQueryData(userKeys.detail(updatedUser.id), updatedUser);
      queryClient.invalidateQueries({ queryKey: userKeys.lists() });
    },
  });
}

export function useDeleteUser() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: usersApi.delete,
    onSuccess: (_, deletedId) => {
      queryClient.removeQueries({ queryKey: userKeys.detail(deletedId) });
      queryClient.invalidateQueries({ queryKey: userKeys.lists() });
    },
  });
}

// ── Dùng trong component ──────────────────────────
function UserPage({ userId }: { userId: string }) {
  const { data: user, isLoading } = useUser(userId);
  const updateUser = useUpdateUser();
  const deleteUser = useDeleteUser();

  if (isLoading) return <Spinner />;

  return (
    <div>
      <h1>{user?.name}</h1>
      <button
        onClick={() => updateUser.mutate({ id: userId, data: { name: 'New Name' } })}
        disabled={updateUser.isPending}
      >
        Cập nhật
      </button>
      <button onClick={() => deleteUser.mutate(userId)} disabled={deleteUser.isPending}>
        Xóa
      </button>
    </div>
  );
}
```

---

## 16. isLoading vs isFetching vs isPending

Đây là điểm dễ nhầm nhất khi dùng TanStack Query v5.

| State | Ý nghĩa | Khi nào dùng |
|---|---|---|
| `isLoading` | Đang fetch lần đầu, chưa có data | Hiện skeleton/spinner lần đầu |
| `isFetching` | Đang fetch (kể cả background) | Hiện loading indicator nhỏ |
| `isPending` | Không có data trong cache | Tương tự isLoading nhưng chính xác hơn |
| `isRefetching` | Đang refetch, đã có data cũ | Hiện "Đang cập nhật..." |

```typescript
const { data, isLoading, isFetching, isRefetching } = useQuery(...);

// Pattern chuẩn
if (isLoading) return <FullPageSpinner />;        // chỉ lần đầu

return (
  <div>
    {isRefetching && <SmallRefetchIndicator />}   // background update
    <UserList data={data} />
  </div>
);
```

---

## 17. staleTime vs gcTime

```
Timeline sau khi fetch xong:
  0s ─── fetch xong ─── staleTime ─── gcTime ─── xóa khỏi cache

         [ FRESH ]       [ STALE ]    [ UNUSED ]
         không refetch   background   inactive
         tự động         refetch      but cached
                         khi cần
```

```typescript
// Theo loại data
// Data ít thay đổi (config, categories)
staleTime: 10 * 60 * 1000  // 10 phút

// Data thay đổi thường xuyên (notifications, messages)
staleTime: 0  // luôn stale, refetch mỗi khi mount/focus

// Data real-time (prices, live scores)
refetchInterval: 5000  // polling mỗi 5 giây
```

---

## 18. Checklist TanStack Query

- [ ] Setup `QueryClient` với `defaultOptions` phù hợp (staleTime, retry)
- [ ] Dùng `ReactQueryDevtools` khi development
- [ ] Tổ chức **Query Key Factory** (`userKeys.detail(id)`, `userKeys.lists()`)
- [ ] Đặt tất cả query/mutation logic vào **custom hooks** riêng
- [ ] Dùng `enabled` để kiểm soát khi nào fetch
- [ ] Dùng `keepPreviousData` cho pagination
- [ ] `invalidateQueries` sau mutation hoặc `setQueryData` cho optimistic update
- [ ] Dùng `select` để transform data, tránh tính toán trong component
- [ ] Phân biệt `isLoading` vs `isFetching` để hiện đúng loading state
- [ ] Dùng `queryClient.clear()` khi user logout
- [ ] Với Next.js: dùng `HydrationBoundary` + `prefetchQuery` ở Server Component
- [ ] Global error handler trong `QueryCache` + `MutationCache`
