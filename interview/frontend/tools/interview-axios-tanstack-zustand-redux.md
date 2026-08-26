# 🔗 Phỏng vấn Frontend: Axios, TanStack Query, Zustand, Redux Toolkit, RTK Query

---

## 📡 PHẦN 1: AXIOS VS FETCH API

### Câu hỏi 1: Axios là gì? Khác Fetch như thế nào?

**Trả lời:**

Axios là HTTP client library, Fetch là browser API tích hợp sẵn.

**So sánh chi tiết:**

| Tiêu chí | Axios | Fetch |
|---------|-------|-------|
| **Source** | NPM package (phải install) | Browser built-in |
| **Bundle size** | ~13KB | 0KB |
| **Syntax** | Simple, friendly | Verbose |
| **Request/Response** | Auto transform JSON | Manual JSON.stringify/parse |
| **Timeout** | Built-in timeout support | Phải dùng AbortController |
| **Interceptors** | Có sẵn | Không có |
| **Error handling** | Status 4xx/5xx throw error | Chỉ throw khi network error |
| **Cancel requests** | CancelToken (v0.x) hoặc AbortController | AbortController |
| **CSRF protection** | Tự động thêm headers | Phải manual |
| **Download progress** | onDownloadProgress | Không có |

### Câu hỏi 2: Làm sao setup Axios?

**Trả lời:**

```bash
npm install axios
```

```javascript
import axios from 'axios';

// Tạo instance với config
const api = axios.create({
  baseURL: 'https://api.example.com',
  timeout: 5000,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Request interceptor (thêm auth token)
api.interceptors.request.use(
  config => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  error => Promise.reject(error)
);

// Response interceptor (handle 401)
api.interceptors.response.use(
  response => response.data,
  error => {
    if (error.response?.status === 401) {
      // Redirect to login
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### Câu hỏi 3: Làm sao dùng Axios trong React?

**Trả lời:**

```jsx
import { useState, useEffect } from 'react';
import api from './api';

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchUser = async () => {
      setLoading(true);
      try {
        const data = await api.get(`/users/${userId}`);
        setUser(data);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchUser();
  }, [userId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return <div>{user?.name}</div>;
}
```

### Câu hỏi 4: Axios vs Fetch - ưu/nhược điểm?

**Trả lời:**

**Axios:**
✅ Ưu:
- Interceptors tích hợp (không cần custom logic)
- Auto JSON transform
- Request/response timeout
- Download progress tracking
- CSRF protection tự động
- Simplified error handling

❌ Nhược:
- Thêm dependency
- Bundle size
- Learning curve

**Fetch:**
✅ Ưu:
- Built-in, không cần install
- Standard (chính thức)
- Nhỏ gọn cho simple requests
- Tương lai của web

❌ Nhược:
- Verbose (phải manual JSON stringify/parse)
- Không có built-in interceptors
- Status 2xx/3xx không throw error (phải check manually)
- Không có timeout support sẵn
- Phức tạp hơn cho complex scenarios

### Câu hỏi 5: Khi nào dùng Axios? Khi nào dùng Fetch?

**Trả lời:**

**Dùng Axios khi:**
- ✅ Cần interceptors
- ✅ API complex (upload file, progress tracking)
- ✅ Cần request cancellation
- ✅ Muốn simple error handling
- ✅ Team không quen Fetch API

**Dùng Fetch khi:**
- ✅ Chỉ simple GET/POST requests
- ✅ Muốn minimize dependencies
- ✅ Project nhỏ
- ✅ Build tool modern (auto polyfill)

### Câu hỏi 6: So sánh code: Fetch vs Axios

**Trả lời:**

**Fetch API:**
```javascript
// Phức tạp hơn
async function fetchUsers() {
  try {
    const res = await fetch('https://api.example.com/users', {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      }
    });

    // ❌ Phải check status manually
    if (!res.ok) {
      throw new Error(`HTTP error! status: ${res.status}`);
    }

    // ❌ Phải parse JSON
    const data = await res.json();
    return data;
  } catch (error) {
    console.error('Error:', error);
  }
}

// Timeout phải tự implement
async function fetchWithTimeout(url, timeout = 5000) {
  const controller = new AbortController();
  const id = setTimeout(() => controller.abort(), timeout);

  try {
    const response = await fetch(url, { signal: controller.signal });
    clearTimeout(id);
    return response.json();
  } catch (error) {
    clearTimeout(id);
    throw error;
  }
}
```

**Axios:**
```javascript
// Đơn giản hơn
async function fetchUsers() {
  try {
    const data = await api.get('/users');
    // ✅ Auto parse JSON
    // ✅ Auto handle errors
    // ✅ Interceptor tự động thêm token
    return data;
  } catch (error) {
    console.error('Error:', error);
  }
}

// Timeout built-in
const api = axios.create({
  timeout: 5000 // ✅ Đơn giản
});
```

---

## 🔄 PHẦN 2: STATE MANAGEMENT - ZUSTAND VS REDUX TOOLKIT

### Câu hỏi 1: Zustand là gì? Redux Toolkit là gì?

**Trả lời:**

**Zustand:**
- Minimalist state management library
- ~2KB bundle size
- Không boilerplate
- Hooks-based API
- Dễ học, dễ dùng

**Redux Toolkit (RTK):**
- Official Redux library
- ~60KB bundle size
- Built on Redux (reducer pattern)
- Có DevTools support
- Nhiều features hơn

**Dùng khi nào:**

| Zustand | Redux Toolkit |
|---------|---------------|
| Simple state | Complex state |
| Startup/Prototype | Large enterprise |
| Learning | Production apps |
| Lightweight | Need DevTools |

### Câu hỏi 2: Zustand - Setup & Usage

**Trả lời:**

```bash
npm install zustand
```

```javascript
// store.js
import { create } from 'zustand';

// ✅ Cách 1: Simple store
const useCounterStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 })
}));

// ✅ Cách 2: Complex store
const useUserStore = create((set) => ({
  user: null,
  loading: false,
  error: null,

  setUser: (user) => set({ user }),
  
  fetchUser: async (id) => {
    set({ loading: true });
    try {
      const res = await fetch(`/api/users/${id}`);
      const user = await res.json();
      set({ user, error: null });
    } catch (error) {
      set({ error: error.message });
    } finally {
      set({ loading: false });
    }
  }
}));

export { useCounterStore, useUserStore };
```

```jsx
// Component
import { useCounterStore, useUserStore } from './store';

function Counter() {
  const count = useCounterStore((state) => state.count);
  const increment = useCounterStore((state) => state.increment);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>+</button>
    </div>
  );
}

function UserProfile() {
  const user = useUserStore((state) => state.user);
  const loading = useUserStore((state) => state.loading);
  const fetchUser = useUserStore((state) => state.fetchUser);

  useEffect(() => {
    fetchUser(123);
  }, []);

  if (loading) return <div>Loading...</div>;
  return <div>{user?.name}</div>;
}
```

**Advantages Zustand:**
✅ Minimal boilerplate
✅ Easy to learn
✅ Hooks API
✅ Lightweight
✅ TypeScript friendly

### Câu hỏi 3: Redux Toolkit - Setup & Usage

**Trả lời:**

```bash
npm install @reduxjs/toolkit react-redux
```

```javascript
// store.js
import { createSlice, configureStore } from '@reduxjs/toolkit';

// 1. Create slice
const counterSlice = createSlice({
  name: 'counter',
  initialState: { count: 0 },
  reducers: {
    increment: (state) => {
      state.count += 1; // Immer handles immutability
    },
    decrement: (state) => {
      state.count -= 1;
    }
  }
});

// 2. Create async thunk
import { createAsyncThunk } from '@reduxjs/toolkit';

export const fetchUser = createAsyncThunk(
  'user/fetchUser',
  async (id) => {
    const res = await fetch(`/api/users/${id}`);
    return res.json();
  }
);

const userSlice = createSlice({
  name: 'user',
  initialState: { data: null, loading: false, error: null },
  extraReducers: (builder) => {
    builder
      .addCase(fetchUser.pending, (state) => {
        state.loading = true;
      })
      .addCase(fetchUser.fulfilled, (state, action) => {
        state.data = action.payload;
        state.loading = false;
      })
      .addCase(fetchUser.rejected, (state, action) => {
        state.error = action.error.message;
        state.loading = false;
      });
  }
});

// 3. Configure store
const store = configureStore({
  reducer: {
    counter: counterSlice.reducer,
    user: userSlice.reducer
  }
});

export const { increment, decrement } = counterSlice.actions;
export default store;
```

```jsx
// App.js
import { Provider } from 'react-redux';
import store from './store';

function App() {
  return (
    <Provider store={store}>
      <Counter />
      <UserProfile />
    </Provider>
  );
}

// Counter.js
import { useDispatch, useSelector } from 'react-redux';
import { increment, decrement } from './store';

function Counter() {
  const dispatch = useDispatch();
  const count = useSelector((state) => state.counter.count);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => dispatch(increment())}>+</button>
      <button onClick={() => dispatch(decrement())}>-</button>
    </div>
  );
}

// UserProfile.js
import { useEffect } from 'react';
import { useDispatch, useSelector } from 'react-redux';
import { fetchUser } from './store';

function UserProfile() {
  const dispatch = useDispatch();
  const { data: user, loading, error } = useSelector((state) => state.user);

  useEffect(() => {
    dispatch(fetchUser(123));
  }, []);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  return <div>{user?.name}</div>;
}
```

### Câu hỏi 4: Zustand vs Redux Toolkit - So sánh

**Trả lời:**

| Tiêu chí | Zustand | Redux Toolkit |
|---------|---------|---------------|
| **Bundle size** | ~2KB | ~60KB |
| **Learning curve** | Rất dễ | Khó (boilerplate) |
| **Boilerplate** | Tối thiểu | Nhiều |
| **DevTools** | Basic | Excellent |
| **Middleware** | Không | Có |
| **Time travel debugging** | Không | Có |
| **Community** | Nhỏ | Lớn |
| **Type safety** | ✅ TypeScript | ✅ TypeScript |
| **Async actions** | Manual hoặc middleware | createAsyncThunk |

### Câu hỏi 5: Khi nào dùng Zustand? Khi nào dùng Redux?

**Trả lời:**

**Dùng Zustand khi:**
✅ Simple global state
✅ Prototype/Startup
✅ Muốn minimal dependencies
✅ Team quen hooks
✅ Project nhỏ-medium

**Dùng Redux Toolkit khi:**
✅ Complex state tree
✅ Large team (consistency)
✅ Cần time-travel debugging
✅ Cần middleware
✅ Enterprise apps
✅ State updates được track carefully

### Câu hỏi 6: Middleware trong Zustand

**Trả lời:**

```javascript
// Zustand middleware
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

const useUserStore = create(
  devtools(
    persist(
      (set) => ({
        user: null,
        setUser: (user) => set({ user })
      }),
      {
        name: 'user-storage' // LocalStorage key
      }
    ),
    { name: 'User Store' } // DevTools name
  )
);
```

### Câu hỏi 7: Migration từ Redux sang Zustand

**Trả lời:**

```javascript
// Redux version
const counterSlice = createSlice({
  name: 'counter',
  initialState: { count: 0 },
  reducers: {
    increment: (state) => { state.count += 1; }
  }
});

// Zustand version (30% code)
const useCounterStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 }))
}));
```

---

## 🎯 PHẦN 3: DATA FETCHING - TANSTACK QUERY VS RTK QUERY

### Câu hỏi 1: TanStack Query là gì? RTK Query là gì?

**Trả lời:**

**TanStack Query (React Query):**
- Data fetching & caching library
- Khác state management
- Auto background refetch
- Query key-based caching
- ~40KB bundle size
- Standalone (independent)

**RTK Query:**
- Built on Redux Toolkit
- Integrated state management + data fetching
- Code generation support
- API slice pattern
- ~20KB (on top of RTK)

### Câu hỏi 2: TanStack Query - Setup & Usage

**Trả lời:**

```bash
npm install @tanstack/react-query
```

```javascript
// queryClient.js
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 min
      gcTime: 1000 * 60 * 10,   // 10 min
      retry: 1,
    },
  },
});
```

```jsx
// App.js
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from './queryClient';

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <UserList />
    </QueryClientProvider>
  );
}

// UserList.js
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

function UserList() {
  // ✅ Query data
  const { data: users, isLoading, error } = useQuery({
    queryKey: ['users'],
    queryFn: async () => {
      const res = await fetch('/api/users');
      return res.json();
    },
    staleTime: 1000 * 60 * 5
  });

  const queryClient = useQueryClient();

  // ✅ Mutation
  const { mutate: deleteUser, isPending } = useMutation({
    mutationFn: async (userId) => {
      const res = await fetch(`/api/users/${userId}`, {
        method: 'DELETE'
      });
      return res.json();
    },
    onSuccess: () => {
      // Invalidate & refetch
      queryClient.invalidateQueries({ queryKey: ['users'] });
    }
  });

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      {users?.map(user => (
        <div key={user.id}>
          <h3>{user.name}</h3>
          <button 
            onClick={() => deleteUser(user.id)}
            disabled={isPending}
          >
            Delete
          </button>
        </div>
      ))}
    </div>
  );
}
```

**Features:**
✅ Automatic caching
✅ Background refetch
✅ Query invalidation
✅ Pagination support
✅ Infinite queries
✅ Optimistic updates

### Câu hỏi 3: RTK Query - Setup & Usage

**Trả lời:**

```bash
npm install @reduxjs/toolkit react-redux
```

```javascript
// api.js
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const usersApi = createApi({
  reducerPath: 'usersApi',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  endpoints: (builder) => ({
    // Query
    getUsers: builder.query({
      query: () => '/users',
      // Invalidate after mutation
      providesTags: ['Users']
    }),

    getUser: builder.query({
      query: (id) => `/users/${id}`,
      providesTags: (result, error, id) => [{ type: 'User', id }]
    }),

    // Mutation
    deleteUser: builder.mutation({
      query: (id) => ({
        url: `/users/${id}`,
        method: 'DELETE'
      }),
      invalidatesTags: ['Users']
    }),

    createUser: builder.mutation({
      query: (newUser) => ({
        url: '/users',
        method: 'POST',
        body: newUser
      }),
      invalidatesTags: ['Users']
    })
  })
});

export const {
  useGetUsersQuery,
  useGetUserQuery,
  useDeleteUserMutation,
  useCreateUserMutation
} = usersApi;
```

```javascript
// store.js
import { configureStore } from '@reduxjs/toolkit';
import { usersApi } from './api';

const store = configureStore({
  reducer: {
    [usersApi.reducerPath]: usersApi.reducer
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(usersApi.middleware)
});

export default store;
```

```jsx
// Component
import { useGetUsersQuery, useDeleteUserMutation } from './api';

function UserList() {
  // ✅ Hook generated automatically
  const { data: users, isLoading, error } = useGetUsersQuery();

  const [deleteUser, { isLoading: isDeleting }] = useDeleteUserMutation();

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      {users?.map(user => (
        <div key={user.id}>
          <h3>{user.name}</h3>
          <button 
            onClick={() => deleteUser(user.id)}
            disabled={isDeleting}
          >
            Delete
          </button>
        </div>
      ))}
    </div>
  );
}
```

### Câu hỏi 4: TanStack Query vs RTK Query - So sánh

**Trả lời:**

| Tiêu chí | TanStack Query | RTK Query |
|---------|--------|----------|
| **Bundle size** | ~40KB | ~20KB (extra) |
| **State management** | Không | Redux (integrated) |
| **Setup complexity** | Simple | Medium |
| **API definition** | Hook-based | API slice |
| **Code generation** | Không | Có (OpenAPI, GraphQL) |
| **DevTools** | React Query DevTools | Redux DevTools |
| **Learning curve** | Dễ | Trung bình |
| **Community** | Lớn | Medium |
| **Flexibility** | Cao | Bị ràng buộc Redux |
| **Offline support** | Có plugin | Phải tự implement |

### Câu hỏi 5: So sánh thực tế: Fetching user

**Trả lời:**

**TanStack Query:**
```jsx
const { data: user } = useQuery({
  queryKey: ['user', id],
  queryFn: () => api.get(`/users/${id}`),
  staleTime: 5 * 60 * 1000
});
```

**RTK Query:**
```jsx
const { data: user } = useGetUserQuery(id);
```

RTK Query ngắn gọn hơn (code generation), nhưng TanStack Query linh hoạt hơn.

### Câu hỏi 6: Caching strategy

**Trả lời:**

**TanStack Query:**
```javascript
useQuery({
  queryKey: ['users'],
  queryFn: fetchUsers,
  staleTime: 5 * 60 * 1000,        // 5 min - khi data coi như stale
  gcTime: 10 * 60 * 1000,          // 10 min - khi remove khỏi cache
  refetchOnWindowFocus: true,
  refetchInterval: 5 * 60 * 1000,   // Background refetch every 5 min
});
```

**RTK Query:**
```javascript
getUsers: builder.query({
  query: () => '/users',
  keepUnusedDataFor: 60,  // seconds
  pollingInterval: 0,     // no polling by default
  providesTags: ['Users']
})
```

### Câu hỏi 7: Pagination trong TanStack Query

**Trả lời:**

```jsx
function UserPagination() {
  const [page, setPage] = useState(1);

  const { data, isPreviousData } = useQuery({
    queryKey: ['users', page],
    queryFn: () => api.get(`/users?page=${page}`),
    placeholderData: keepPreviousData  // Show old data while loading new
  });

  return (
    <div>
      {data?.users.map(u => <div key={u.id}>{u.name}</div>)}
      <button 
        onClick={() => setPage(p => p - 1)}
        disabled={page === 1 || isPreviousData}
      >
        Previous
      </button>
      <button 
        onClick={() => setPage(p => p + 1)}
        disabled={isPreviousData}
      >
        Next
      </button>
    </div>
  );
}
```

### Câu hỏi 8: Infinite queries

**Trả lời:**

```jsx
import { useInfiniteQuery } from '@tanstack/react-query';

function InfiniteUserList() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = 
    useInfiniteQuery({
      queryKey: ['users'],
      queryFn: ({ pageParam = 1 }) => 
        api.get(`/users?page=${pageParam}`),
      getNextPageParam: (lastPage, pages) =>
        lastPage.hasMore ? pages.length + 1 : undefined
    });

  return (
    <div>
      {data?.pages.map((page, i) => (
        <div key={i}>
          {page.users.map(u => (
            <div key={u.id}>{u.name}</div>
          ))}
        </div>
      ))}
      <button 
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage ? 'Loading...' : 'Load more'}
      </button>
    </div>
  );
}
```

### Câu hỏi 9: Optimistic updates

**Trả lời:**

**TanStack Query:**
```jsx
const { mutate: updateUser } = useMutation({
  mutationFn: (updatedUser) => api.put(`/users/${updatedUser.id}`, updatedUser),
  
  onMutate: async (newData) => {
    // Cancel ongoing queries
    await queryClient.cancelQueries({ queryKey: ['user', newData.id] });
    
    // Snapshot old data
    const previousData = queryClient.getQueryData(['user', newData.id]);
    
    // Optimistic update
    queryClient.setQueryData(['user', newData.id], newData);
    
    return { previousData };
  },
  
  onError: (err, newData, context) => {
    // Rollback on error
    queryClient.setQueryData(['user', newData.id], context.previousData);
  },
  
  onSuccess: () => {
    // Refetch to sync
    queryClient.invalidateQueries({ queryKey: ['user'] });
  }
});
```

**RTK Query:**
```javascript
updateUser: builder.mutation({
  query: (user) => ({
    url: `/users/${user.id}`,
    method: 'PUT',
    body: user
  }),
  
  async onQueryStarted(user, { dispatch, queryFulfilled, getState }) {
    // Optimistic update
    const patchResult = dispatch(
      usersApi.util.updateQueryData('getUser', user.id, (draft) => {
        Object.assign(draft, user);
      })
    );

    try {
      await queryFulfilled;
    } catch {
      // Rollback on error
      patchResult.undo();
    }
  }
})
```

### Câu hỏi 10: Khi nào dùng TanStack? Khi nào dùng RTK Query?

**Trả lời:**

**Dùng TanStack Query khi:**
✅ State management từ Redux/Zustand
✅ Flexible data fetching strategy
✅ Standalone project
✅ Complex caching needs
✅ Team quen React hooks
✅ Offline support cần thiết

**Dùng RTK Query khi:**
✅ Đã dùng Redux Toolkit
✅ Muốn integrated state management
✅ Code generation từ API spec
✅ Simple, consistent API pattern
✅ Centralized state management
✅ GraphQL hoặc OpenAPI integration

---

## 🔗 PHẦN 4: AXIOS + TANSTACK QUERY (BEST PRACTICE)

### Setup pattern

```javascript
// api/client.js
import axios from 'axios';

export const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  timeout: 10000
});

// Add token
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Handle 401
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

```javascript
// api/queries.js
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query';
import { apiClient } from './client';

// Query
export function useGetUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: async () => {
      const { data } = await apiClient.get('/users');
      return data;
    }
  });
}

export function useGetUser(id) {
  return useQuery({
    queryKey: ['user', id],
    queryFn: async () => {
      const { data } = await apiClient.get(`/users/${id}`);
      return data;
    }
  });
}

// Mutation
export function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (newUser) => {
      const { data } = await apiClient.post('/users', newUser);
      return data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    }
  });
}

export function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (user) => {
      const { data } = await apiClient.put(`/users/${user.id}`, user);
      return data;
    },
    onSuccess: (updatedUser) => {
      queryClient.setQueryData(['user', updatedUser.id], updatedUser);
      queryClient.invalidateQueries({ queryKey: ['users'] });
    }
  });
}
```

```jsx
// Component
import { useGetUsers, useCreateUser, useUpdateUser } from './api/queries';

function UserManager() {
  const { data: users, isLoading } = useGetUsers();
  const createUserMutation = useCreateUser();
  const updateUserMutation = useUpdateUser();

  const handleCreate = async (newUser) => {
    await createUserMutation.mutateAsync(newUser);
  };

  const handleUpdate = async (user) => {
    await updateUserMutation.mutateAsync(user);
  };

  return (
    <div>
      {isLoading ? 'Loading...' : users?.map(u => (...))}
    </div>
  );
}
```

---

## 🔗 PHẦN 5: AXIOS + ZUSTAND (PATTERN)

```javascript
// store/userStore.js
import { create } from 'zustand';
import { apiClient } from '../api/client';

export const useUserStore = create((set) => ({
  users: [],
  currentUser: null,
  loading: false,
  error: null,

  fetchUsers: async () => {
    set({ loading: true, error: null });
    try {
      const { data } = await apiClient.get('/users');
      set({ users: data, loading: false });
    } catch (error) {
      set({ error: error.message, loading: false });
    }
  },

  createUser: async (newUser) => {
    try {
      const { data } = await apiClient.post('/users', newUser);
      set((state) => ({ users: [...state.users, data] }));
      return data;
    } catch (error) {
      set({ error: error.message });
      throw error;
    }
  }
}));
```

---

## 📊 QUICK COMPARISON MATRIX

### HTTP Clients
| Feature | Axios | Fetch |
|---------|-------|-------|
| Install | Yes | No |
| Timeout | ✅ | ❌ |
| Interceptors | ✅ | ❌ |
| Error handling | ✅✅ | Limited |
| Best for | APIs | Simple requests |

### State Management
| Feature | Zustand | Redux |
|---------|---------|-------|
| Learning | ✅✅ | ❌ |
| Boilerplate | ✅✅ | ❌ |
| Features | Basic | Complete |
| Best for | Simple | Complex |

### Data Fetching
| Feature | TanStack | RTK Query |
|---------|----------|----------|
| Flexibility | ✅✅ | Limited |
| Integration | Standalone | Redux only |
| Code gen | ❌ | ✅ |
| Best for | Complex | Integrated |

---

## 🎯 DECISION TREE

```
Cần fetch data?
├─ Đơn giản → Axios + useState
├─ Caching cần thiết → TanStack Query
└─ Integrated → RTK Query

Cần global state?
├─ Đơn giản → Zustand
├─ Form state → React Hook Form
└─ Complex → Redux Toolkit

Kết hợp tốt nhất:
├─ Simple: Fetch + Zustand
├─ Medium: Axios + TanStack
└─ Enterprise: Redux + RTK Query
```

---

## 💡 INTERVIEWER TIPS

**Magic keywords:**
- "Request interceptors"
- "Query invalidation"
- "Optimistic updates"
- "Caching strategy"
- "Request deduping"

**Common mistakes:**
❌ "Redux tốt nhất cho mọi project"
❌ Không hiểu trade-off bundle size vs features
❌ Không mention interceptors
❌ "Dùng library X là được"

**Show knowledge:**
✅ Know when NOT to use library
✅ Compare trade-offs
✅ Real-world scenarios
✅ Performance implications
✅ Migration strategies

---

## 📝 PRACTICE SCENARIOS

**Scenario 1:** Setup API client with Axios interceptors
- Add auth token
- Handle 401
- Transform response

**Scenario 2:** Implement TanStack Query with pagination
- Fetch by page
- Show old data while loading
- Button disabled states

**Scenario 3:** Replace Redux with Zustand
- Write Zustand stores equivalent
- Keep same API surface
- Async operations handling

**Scenario 4:** RTK Query with optimistic updates
- Update local immediately
- Rollback on error
- Invalidate related queries

---

**Good luck with your interview! 🚀**
