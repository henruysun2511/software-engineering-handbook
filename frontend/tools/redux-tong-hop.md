# Tổng Hợp Kiến Thức Redux: Từ Nền Tảng Đến Nâng Cao

> **Phạm vi tài liệu:** Redux Core · Redux Thunk · Redux Toolkit · RTK Query  
> **Đối tượng:** Lập trình viên JavaScript / React có kiến thức nền về ES6+  
> **Mục tiêu:** Cung cấp cái nhìn hệ thống, khái niệm chính xác và code mẫu thực tế theo chuẩn hiện đại

---

## Mục Lục

1. [Tổng quan về Quản lý Trạng thái (State Management)](#1-tổng-quan-về-quản-lý-trạng-thái)
2. [Redux Core — Nền tảng kiến trúc](#2-redux-core)
3. [Redux Thunk — Xử lý tác vụ bất đồng bộ](#3-redux-thunk)
4. [Redux Toolkit — Chuẩn hoá và tối giản hoá](#4-redux-toolkit)
5. [RTK Query — Quản lý dữ liệu từ server](#5-rtk-query)
6. [So sánh tổng thể và khuyến nghị lựa chọn](#6-so-sánh-và-khuyến-nghị)

---

## 1. Tổng Quan Về Quản Lý Trạng Thái

### 1.1 Vấn đề đặt ra

Trong các ứng dụng web quy mô lớn, trạng thái (state) của giao diện người dùng trở nên phức tạp khi nhiều thành phần (component) cần chia sẻ và đồng bộ dữ liệu với nhau. Các vấn đề thường gặp bao gồm:

- **Prop drilling**: truyền dữ liệu qua nhiều cấp component trung gian không cần thiết.
- **Trạng thái phân tán**: dữ liệu cùng loại tồn tại ở nhiều nơi, dễ gây mất đồng bộ.
- **Khó theo dõi thay đổi**: không có cơ chế rõ ràng để biết trạng thái thay đổi khi nào và tại sao.

Redux ra đời như một giải pháp quản lý trạng thái tập trung (centralized state management), lấy cảm hứng từ kiến trúc Flux của Facebook và ngôn ngữ hàm Elm.

### 1.2 Triết lý cốt lõi

Redux tuân theo ba nguyên tắc nền tảng:

| Nguyên tắc | Ý nghĩa |
|---|---|
| **Single Source of Truth** | Toàn bộ trạng thái ứng dụng được lưu trong một store duy nhất |
| **State is Read-Only** | Trạng thái chỉ có thể thay đổi bằng cách dispatch một action |
| **Changes via Pure Functions** | Reducer phải là hàm thuần túy (pure function) |

---

## 2. Redux Core

### 2.1 Các khái niệm nền tảng

#### 2.1.1 Store

Store là trung tâm của kiến trúc Redux — nơi lưu trữ toàn bộ state tree của ứng dụng. Trong một ứng dụng Redux, chỉ tồn tại **duy nhất một store**. Store cung cấp các phương thức:

- `getState()`: trả về state hiện tại.
- `dispatch(action)`: gửi một action để thay đổi state.
- `subscribe(listener)`: đăng ký hàm lắng nghe mỗi khi state thay đổi.

#### 2.1.2 Action

Action là một plain JavaScript object mô tả **ý định thay đổi** state. Mọi action bắt buộc phải có trường `type` dưới dạng chuỗi ký tự mô tả tên sự kiện. Ngoài ra, action có thể chứa thêm trường `payload` để truyền dữ liệu kèm theo.

```js
// Dạng action đơn giản
{ type: 'counter/increment' }

// Action kèm dữ liệu
{ type: 'todos/addTodo', payload: { id: 1, text: 'Học Redux', completed: false } }
```

Để tránh lặp lại chuỗi type và chuẩn hoá cấu trúc, người ta thường dùng **Action Creator** — hàm trả về action object:

```js
const addTodo = (text) => ({
  type: 'todos/addTodo',
  payload: { id: Date.now(), text, completed: false },
});
```

#### 2.1.3 Reducer

Reducer là một **pure function** nhận vào `(state, action)` và trả về state mới. Tuyệt đối không được:

- Thay đổi trực tiếp (mutate) state đầu vào.
- Thực hiện tác vụ có side effect (gọi API, đọc file, tạo số ngẫu nhiên…).
- Phụ thuộc vào giá trị không xác định (Date.now(), Math.random()…).

```js
const initialState = { count: 0 };

function counterReducer(state = initialState, action) {
  switch (action.type) {
    case 'counter/increment':
      return { ...state, count: state.count + 1 };
    case 'counter/decrement':
      return { ...state, count: state.count - 1 };
    case 'counter/incrementByAmount':
      return { ...state, count: state.count + action.payload };
    default:
      return state;
  }
}
```

#### 2.1.4 Dispatch và Data Flow

Luồng dữ liệu trong Redux là **một chiều (unidirectional)**:

```
UI (Component)
     │  dispatch(action)
     ▼
  Store
     │  chuyển (state, action) cho
     ▼
  Reducer
     │  trả về state mới
     ▼
  Store (cập nhật state)
     │  thông báo subscribers
     ▼
UI re-render
```

### 2.2 Ví dụ Redux Core hoàn chỉnh (không dùng React)

```js
import { createStore, combineReducers } from 'redux';

// --- Reducers ---
function counterReducer(state = { count: 0 }, action) {
  switch (action.type) {
    case 'counter/increment':
      return { count: state.count + 1 };
    case 'counter/decrement':
      return { count: state.count - 1 };
    default:
      return state;
  }
}

function todosReducer(state = [], action) {
  switch (action.type) {
    case 'todos/add':
      return [...state, { id: Date.now(), text: action.payload, done: false }];
    case 'todos/toggle':
      return state.map((todo) =>
        todo.id === action.payload ? { ...todo, done: !todo.done } : todo
      );
    default:
      return state;
  }
}

// --- Kết hợp nhiều reducer ---
const rootReducer = combineReducers({
  counter: counterReducer,
  todos: todosReducer,
});

// --- Tạo store ---
const store = createStore(rootReducer);

// --- Lắng nghe thay đổi ---
store.subscribe(() => {
  console.log('State mới:', store.getState());
});

// --- Dispatch actions ---
store.dispatch({ type: 'counter/increment' });
// State mới: { counter: { count: 1 }, todos: [] }

store.dispatch({ type: 'todos/add', payload: 'Học Redux Core' });
// State mới: { counter: { count: 1 }, todos: [{ id: ..., text: 'Học Redux Core', done: false }] }
```

### 2.3 Tích hợp Redux Core với React

Thư viện `react-redux` cung cấp hai API chính để kết nối React với Redux store:

- **`Provider`**: component bao bọc toàn bộ ứng dụng, truyền store xuống mọi component con qua React Context.
- **`useSelector`**: hook lấy dữ liệu từ store.
- **`useDispatch`**: hook lấy hàm dispatch.

```jsx
// index.jsx
import React from 'react';
import { createRoot } from 'react-dom/client';
import { Provider } from 'react-redux';
import { createStore } from 'redux';
import App from './App';
import rootReducer from './reducers';

const store = createStore(rootReducer);

createRoot(document.getElementById('root')).render(
  <Provider store={store}>
    <App />
  </Provider>
);
```

```jsx
// Counter.jsx
import { useSelector, useDispatch } from 'react-redux';

export default function Counter() {
  const count = useSelector((state) => state.counter.count);
  const dispatch = useDispatch();

  return (
    <div>
      <p>Giá trị: {count}</p>
      <button onClick={() => dispatch({ type: 'counter/increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'counter/decrement' })}>-</button>
    </div>
  );
}
```

### 2.4 Middleware trong Redux

Middleware là cơ chế mở rộng khả năng của `dispatch`. Về bản chất, middleware là một hàm dạng currying nhận vào `store API`, trả về hàm nhận `next`, trả về hàm nhận `action`:

```
(store) => (next) => (action) => { ... }
```

Middleware được áp dụng khi tạo store thông qua `applyMiddleware`:

```js
import { createStore, applyMiddleware } from 'redux';

const loggerMiddleware = (store) => (next) => (action) => {
  console.log('Dispatching:', action);
  const result = next(action);
  console.log('State sau:', store.getState());
  return result;
};

const store = createStore(rootReducer, applyMiddleware(loggerMiddleware));
```

---

## 3. Redux Thunk

### 3.1 Vấn đề của logic bất đồng bộ

Trong Redux Core, `dispatch` chỉ chấp nhận **plain object**. Điều này gây khó khăn khi cần thực hiện các tác vụ bất đồng bộ như gọi API trước khi cập nhật state.

Redux Thunk giải quyết vấn đề này bằng cách mở rộng `dispatch` để chấp nhận **function** (gọi là "thunk"). Khi dispatch nhận một function thay vì object, Thunk middleware sẽ gọi hàm đó và truyền vào `dispatch` và `getState`.

### 3.2 Cài đặt

```bash
npm install redux-thunk
```

```js
import { createStore, applyMiddleware } from 'redux';
import thunk from 'redux-thunk';

const store = createStore(rootReducer, applyMiddleware(thunk));
```

### 3.3 Cấu trúc một Thunk

```js
// Thunk action creator
const fetchUser = (userId) => async (dispatch, getState) => {
  // Có thể đọc state hiện tại nếu cần
  // const currentUser = getState().user;

  dispatch({ type: 'user/fetchStart' });

  try {
    const response = await fetch(`/api/users/${userId}`);
    const data = await response.json();
    dispatch({ type: 'user/fetchSuccess', payload: data });
  } catch (error) {
    dispatch({ type: 'user/fetchError', payload: error.message });
  }
};

// Sử dụng
store.dispatch(fetchUser(42));
```

### 3.4 Ví dụ thực tế: Quản lý danh sách bài viết

```js
// postsReducer.js
const initialState = {
  items: [],
  loading: false,
  error: null,
};

export function postsReducer(state = initialState, action) {
  switch (action.type) {
    case 'posts/fetchStart':
      return { ...state, loading: true, error: null };
    case 'posts/fetchSuccess':
      return { ...state, loading: false, items: action.payload };
    case 'posts/fetchError':
      return { ...state, loading: false, error: action.payload };
    default:
      return state;
  }
}

// postsThunks.js
export const fetchPosts = () => async (dispatch) => {
  dispatch({ type: 'posts/fetchStart' });
  try {
    const res = await fetch('https://jsonplaceholder.typicode.com/posts');
    const data = await res.json();
    dispatch({ type: 'posts/fetchSuccess', payload: data });
  } catch (err) {
    dispatch({ type: 'posts/fetchError', payload: err.message });
  }
};

export const createPost = (postData) => async (dispatch) => {
  try {
    const res = await fetch('https://jsonplaceholder.typicode.com/posts', {
      method: 'POST',
      body: JSON.stringify(postData),
      headers: { 'Content-Type': 'application/json' },
    });
    const data = await res.json();
    dispatch({ type: 'posts/add', payload: data });
  } catch (err) {
    dispatch({ type: 'posts/fetchError', payload: err.message });
  }
};
```

```jsx
// PostList.jsx
import { useEffect } from 'react';
import { useSelector, useDispatch } from 'react-redux';
import { fetchPosts } from './postsThunks';

export default function PostList() {
  const { items, loading, error } = useSelector((state) => state.posts);
  const dispatch = useDispatch();

  useEffect(() => {
    dispatch(fetchPosts());
  }, [dispatch]);

  if (loading) return <p>Đang tải...</p>;
  if (error) return <p>Lỗi: {error}</p>;

  return (
    <ul>
      {items.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### 3.5 Hạn chế của Redux Thunk

Mặc dù đơn giản và linh hoạt, Redux Thunk có một số nhược điểm đáng lưu ý:

- Code lặp lại nhiều: mỗi thao tác async đòi hỏi ít nhất 3 action types (start, success, error) và viết tay toàn bộ reducer logic.
- Không có chuẩn hoá: mỗi nhà phát triển có thể viết theo phong cách khác nhau.
- Khó quản lý trạng thái loading/error ở quy mô lớn.

Những hạn chế này được giải quyết trong Redux Toolkit.

---

## 4. Redux Toolkit

### 4.1 Giới thiệu

Redux Toolkit (RTK) là thư viện chính thức được đội ngũ Redux phát triển và **khuyến nghị dùng thay cho cách tiếp cận Redux Core thuần tuý**. RTK ra đời nhằm giải quyết ba vấn đề lớn nhất của Redux truyền thống:

1. **Cấu hình store quá phức tạp.**
2. **Quá nhiều boilerplate code** (lặp lại không có giá trị).
3. **Cần cài đặt nhiều gói phụ thuộc** (redux-thunk, immer, reselect…).

RTK tích hợp sẵn các thư viện:

| Thư viện | Vai trò |
|---|---|
| **Immer** | Cho phép viết logic "mutate" nhưng thực tế tạo bản sao bất biến |
| **Redux Thunk** | Xử lý async mặc định |
| **Reselect** | Tạo selector có bộ nhớ cache (memoized) |

### 4.2 Cài đặt

```bash
npm install @reduxjs/toolkit react-redux
```

### 4.3 `configureStore` — Cấu hình store

Thay thế `createStore` với cấu hình mặc định tốt hơn: tự động áp dụng Redux Thunk, Redux DevTools Extension, và middleware kiểm tra serializability.

```js
// store.js
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from './features/counter/counterSlice';
import postsReducer from './features/posts/postsSlice';

export const store = configureStore({
  reducer: {
    counter: counterReducer,
    posts: postsReducer,
  },
  // middleware có thể tuỳ chỉnh nếu cần
  // middleware: (getDefaultMiddleware) => getDefaultMiddleware().concat(myMiddleware),
});

// Xuất kiểu dữ liệu (hữu ích khi dùng TypeScript)
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

### 4.4 `createSlice` — Định nghĩa Reducer và Action

`createSlice` là API trung tâm và quan trọng nhất của RTK. Nó nhận một object cấu hình và tự động sinh ra:

- **Action creators** tương ứng với từng reducer function.
- **Action type strings** theo dạng `'sliceName/actionName'`.
- Một **reducer** kết hợp tất cả case.

Nhờ Immer được tích hợp sẵn, bên trong `reducers`, ta có thể viết code "mutate" trực tiếp mà không vi phạm nguyên tắc bất biến (immutability).

```js
// features/counter/counterSlice.js
import { createSlice } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: {
    value: 0,
    status: 'idle',
  },
  reducers: {
    // Nhờ Immer, có thể viết trực tiếp state.value thay vì spread
    increment(state) {
      state.value += 1;
    },
    decrement(state) {
      state.value -= 1;
    },
    incrementByAmount(state, action) {
      state.value += action.payload;
    },
    reset(state) {
      state.value = 0;
    },
  },
});

// Export action creators (được tự động sinh ra)
export const { increment, decrement, incrementByAmount, reset } = counterSlice.actions;

// Export reducer
export default counterSlice.reducer;

// Selector
export const selectCount = (state) => state.counter.value;
```

```jsx
// Counter.jsx
import { useSelector, useDispatch } from 'react-redux';
import { increment, decrement, incrementByAmount, selectCount } from './counterSlice';

export default function Counter() {
  const count = useSelector(selectCount);
  const dispatch = useDispatch();

  return (
    <div>
      <h2>Bộ đếm: {count}</h2>
      <button onClick={() => dispatch(increment())}>+1</button>
      <button onClick={() => dispatch(decrement())}>-1</button>
      <button onClick={() => dispatch(incrementByAmount(5))}>+5</button>
      <button onClick={() => dispatch(reset())}>Reset</button>
    </div>
  );
}
```

### 4.5 `createAsyncThunk` — Xử lý bất đồng bộ chuẩn hoá

`createAsyncThunk` nhận vào một action type string và một hàm async "payload creator". Nó tự động dispatch 3 action lifecycle:

- `pending`: ngay khi bắt đầu gọi hàm async.
- `fulfilled`: khi hàm async resolve thành công.
- `rejected`: khi hàm async throw lỗi.

```js
// features/posts/postsSlice.js
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

// Định nghĩa thunk
export const fetchPosts = createAsyncThunk(
  'posts/fetchPosts', // action type prefix
  async (_, { rejectWithValue }) => {
    try {
      const response = await fetch('https://jsonplaceholder.typicode.com/posts?_limit=10');
      if (!response.ok) throw new Error('Server lỗi');
      return await response.json();
    } catch (error) {
      return rejectWithValue(error.message);
    }
  }
);

export const addPost = createAsyncThunk(
  'posts/addPost',
  async (newPost, { rejectWithValue }) => {
    try {
      const response = await fetch('https://jsonplaceholder.typicode.com/posts', {
        method: 'POST',
        body: JSON.stringify(newPost),
        headers: { 'Content-Type': 'application/json' },
      });
      return await response.json();
    } catch (error) {
      return rejectWithValue(error.message);
    }
  }
);

// Slice
const postsSlice = createSlice({
  name: 'posts',
  initialState: {
    items: [],
    selectedPost: null,
    loading: false,
    error: null,
  },
  reducers: {
    // Reducer đồng bộ thông thường
    clearError(state) {
      state.error = null;
    },
  },
  // extraReducers xử lý action từ createAsyncThunk
  extraReducers: (builder) => {
    builder
      // fetchPosts
      .addCase(fetchPosts.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchPosts.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload;
      })
      .addCase(fetchPosts.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload ?? 'Có lỗi xảy ra';
      })
      // addPost
      .addCase(addPost.fulfilled, (state, action) => {
        state.items.unshift(action.payload);
      });
  },
});

export const { clearError } = postsSlice.actions;
export default postsSlice.reducer;

// Selectors
export const selectAllPosts = (state) => state.posts.items;
export const selectPostsLoading = (state) => state.posts.loading;
export const selectPostsError = (state) => state.posts.error;
```

```jsx
// PostList.jsx
import { useEffect } from 'react';
import { useSelector, useDispatch } from 'react-redux';
import { fetchPosts, addPost, selectAllPosts, selectPostsLoading, selectPostsError } from './postsSlice';

export default function PostList() {
  const dispatch = useDispatch();
  const posts = useSelector(selectAllPosts);
  const loading = useSelector(selectPostsLoading);
  const error = useSelector(selectPostsError);

  useEffect(() => {
    dispatch(fetchPosts());
  }, [dispatch]);

  const handleAdd = () => {
    dispatch(addPost({ title: 'Bài viết mới', body: 'Nội dung...', userId: 1 }));
  };

  if (loading) return <p>Đang tải dữ liệu...</p>;
  if (error) return <p style={{ color: 'red' }}>Lỗi: {error}</p>;

  return (
    <div>
      <button onClick={handleAdd}>Thêm bài viết</button>
      <ul>
        {posts.map((post) => (
          <li key={post.id}>
            <strong>{post.id}.</strong> {post.title}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### 4.6 `createSelector` — Memoized Selectors

Được tái xuất từ thư viện Reselect, `createSelector` tạo ra các selector có khả năng ghi nhớ kết quả (memoize). Selector chỉ tính toán lại khi các input thay đổi, tránh re-render không cần thiết.

```js
import { createSelector } from '@reduxjs/toolkit';

const selectPosts = (state) => state.posts.items;
const selectFilter = (state) => state.posts.filter;

// Selector kết hợp — chỉ tính lại khi posts hoặc filter thay đổi
export const selectFilteredPosts = createSelector(
  [selectPosts, selectFilter],
  (posts, filter) => {
    if (!filter) return posts;
    return posts.filter((post) =>
      post.title.toLowerCase().includes(filter.toLowerCase())
    );
  }
);

// Sử dụng trong component
const filteredPosts = useSelector(selectFilteredPosts);
```

### 4.7 `createEntityAdapter` — Quản lý danh sách dữ liệu chuẩn hoá

Khi làm việc với danh sách dữ liệu dạng collection (mảng các object có ID), RTK cung cấp `createEntityAdapter` để tự động hoá các thao tác CRUD phổ biến.

```js
import { createSlice, createAsyncThunk, createEntityAdapter } from '@reduxjs/toolkit';

// Adapter tự động tạo các hàm CRUD và selectors
const usersAdapter = createEntityAdapter({
  selectId: (user) => user.id,
  sortComparer: (a, b) => a.name.localeCompare(b.name),
});

export const fetchUsers = createAsyncThunk('users/fetchAll', async () => {
  const response = await fetch('https://jsonplaceholder.typicode.com/users');
  return response.json();
});

const usersSlice = createSlice({
  name: 'users',
  initialState: usersAdapter.getInitialState({ loading: false }),
  reducers: {
    userAdded: usersAdapter.addOne,
    userUpdated: usersAdapter.updateOne,
    userRemoved: usersAdapter.removeOne,
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchUsers.pending, (state) => { state.loading = true; })
      .addCase(fetchUsers.fulfilled, (state, action) => {
        state.loading = false;
        usersAdapter.setAll(state, action.payload);
      });
  },
});

export const { userAdded, userUpdated, userRemoved } = usersSlice.actions;
export default usersSlice.reducer;

// Adapter tự động tạo selectors
export const {
  selectAll: selectAllUsers,
  selectById: selectUserById,
  selectIds: selectUserIds,
  selectTotal: selectUsersTotal,
} = usersAdapter.getSelectors((state) => state.users);
```

---

## 5. RTK Query

### 5.1 Giới thiệu và triết lý

RTK Query là một giải pháp quản lý dữ liệu từ server (server-state management) được tích hợp trong `@reduxjs/toolkit`. Nó được thiết kế để loại bỏ hoàn toàn boilerplate code khi thực hiện các tác vụ fetching, caching, synchronization, và invalidation dữ liệu.

RTK Query được lấy cảm hứng từ React Query và SWR nhưng có lợi thế là **tích hợp hoàn toàn với Redux store**, cho phép lưu trữ server state trong cùng một store với client state.

Các vấn đề RTK Query giải quyết tự động:

- Theo dõi trạng thái loading, success, error.
- Cache và deduplicate (loại trùng) các request giống nhau.
- Tự động invalidate và refetch dữ liệu khi có thay đổi.
- Optimistic updates.
- Polling (tự động refetch định kỳ).

### 5.2 `createApi` — Định nghĩa API Service

`createApi` là hàm trung tâm của RTK Query. Mỗi lời gọi `createApi` tạo ra một "API service" chứa tất cả endpoint definitions.

```js
// services/postsApi.js
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const postsApi = createApi({
  reducerPath: 'postsApi',    // Key trong Redux store
  baseQuery: fetchBaseQuery({
    baseUrl: 'https://jsonplaceholder.typicode.com',
    // Có thể thêm headers chung
    prepareHeaders: (headers, { getState }) => {
      const token = getState().auth.token;
      if (token) headers.set('Authorization', `Bearer ${token}`);
      return headers;
    },
  }),
  tagTypes: ['Post', 'User'],  // Dùng để quản lý cache invalidation
  endpoints: (builder) => ({

    // Query — dùng để đọc dữ liệu (GET)
    getPosts: builder.query({
      query: (limit = 10) => `/posts?_limit=${limit}`,
      providesTags: (result) =>
        result
          ? [
              ...result.map(({ id }) => ({ type: 'Post', id })),
              { type: 'Post', id: 'LIST' },
            ]
          : [{ type: 'Post', id: 'LIST' }],
    }),

    getPostById: builder.query({
      query: (id) => `/posts/${id}`,
      providesTags: (result, error, id) => [{ type: 'Post', id }],
    }),

    // Mutation — dùng để ghi dữ liệu (POST, PUT, DELETE)
    addPost: builder.mutation({
      query: (newPost) => ({
        url: '/posts',
        method: 'POST',
        body: newPost,
      }),
      // Sau khi thêm thành công, invalidate cache danh sách
      invalidatesTags: [{ type: 'Post', id: 'LIST' }],
    }),

    updatePost: builder.mutation({
      query: ({ id, ...patch }) => ({
        url: `/posts/${id}`,
        method: 'PUT',
        body: patch,
      }),
      invalidatesTags: (result, error, { id }) => [{ type: 'Post', id }],
    }),

    deletePost: builder.mutation({
      query: (id) => ({
        url: `/posts/${id}`,
        method: 'DELETE',
      }),
      invalidatesTags: (result, error, id) => [
        { type: 'Post', id },
        { type: 'Post', id: 'LIST' },
      ],
    }),
  }),
});

// RTK Query tự động tạo hooks từ endpoint definitions
export const {
  useGetPostsQuery,
  useGetPostByIdQuery,
  useAddPostMutation,
  useUpdatePostMutation,
  useDeletePostMutation,
} = postsApi;
```

### 5.3 Tích hợp vào Store

```js
// store.js
import { configureStore } from '@reduxjs/toolkit';
import { postsApi } from './services/postsApi';
import { setupListeners } from '@reduxjs/toolkit/query';

export const store = configureStore({
  reducer: {
    // Thêm reducer của RTK Query service
    [postsApi.reducerPath]: postsApi.reducer,
  },
  middleware: (getDefaultMiddleware) =>
    // Thêm middleware của RTK Query (xử lý caching, invalidation...)
    getDefaultMiddleware().concat(postsApi.middleware),
});

// Bật tính năng refetchOnFocus và refetchOnReconnect
setupListeners(store.dispatch);
```

### 5.4 Sử dụng Query Hook trong Component

```jsx
// PostList.jsx
import { useState } from 'react';
import { useGetPostsQuery, useDeletePostMutation } from '../services/postsApi';

export default function PostList() {
  const [limit, setLimit] = useState(10);

  // Hook tự động fetch khi component mount
  const {
    data: posts,       // Dữ liệu trả về
    isLoading,         // Đang fetch lần đầu
    isFetching,        // Đang fetch (kể cả refetch)
    isError,
    error,
    refetch,           // Hàm để fetch lại thủ công
  } = useGetPostsQuery(limit);

  const [deletePost, { isLoading: isDeleting }] = useDeletePostMutation();

  if (isLoading) return <p>Đang tải lần đầu...</p>;
  if (isError) return <p>Lỗi: {error.status}</p>;

  return (
    <div>
      {isFetching && <span> (Đang cập nhật...)</span>}
      <button onClick={refetch}>Tải lại</button>
      <select value={limit} onChange={(e) => setLimit(Number(e.target.value))}>
        <option value={5}>5 bài</option>
        <option value={10}>10 bài</option>
        <option value={20}>20 bài</option>
      </select>
      <ul>
        {posts?.map((post) => (
          <li key={post.id}>
            {post.title}
            <button
              onClick={() => deletePost(post.id)}
              disabled={isDeleting}
            >
              Xoá
            </button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### 5.5 Sử dụng Mutation Hook

```jsx
// AddPostForm.jsx
import { useState } from 'react';
import { useAddPostMutation } from '../services/postsApi';

export default function AddPostForm() {
  const [title, setTitle] = useState('');
  const [body, setBody] = useState('');

  const [addPost, { isLoading, isSuccess, isError, error, reset }] =
    useAddPostMutation();

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      await addPost({ title, body, userId: 1 }).unwrap();
      // .unwrap() sẽ throw nếu có lỗi, giúp dùng try/catch
      setTitle('');
      setBody('');
    } catch (err) {
      console.error('Thêm thất bại:', err);
    }
  };

  return (
    <div>
      <h3>Thêm bài viết</h3>
      {isSuccess && <p style={{ color: 'green' }}>Đã thêm thành công!</p>}
      {isError && <p style={{ color: 'red' }}>Lỗi: {error.message}</p>}
      <input
        value={title}
        onChange={(e) => setTitle(e.target.value)}
        placeholder="Tiêu đề"
      />
      <textarea
        value={body}
        onChange={(e) => setBody(e.target.value)}
        placeholder="Nội dung"
      />
      <button onClick={handleSubmit} disabled={isLoading || !title}>
        {isLoading ? 'Đang gửi...' : 'Thêm bài viết'}
      </button>
      <button onClick={reset}>Reset trạng thái</button>
    </div>
  );
}
```

### 5.6 Các tính năng nâng cao của RTK Query

#### 5.6.1 Polling — Tự động refetch định kỳ

```jsx
// Tự động refetch mỗi 30 giây
const { data } = useGetPostsQuery(10, {
  pollingInterval: 30000,
  skipPollingIfUnfocused: true, // Dừng polling khi tab không focus
});
```

#### 5.6.2 Conditional Fetching — Fetch có điều kiện

```jsx
// Chỉ fetch khi userId tồn tại
const { data: user } = useGetUserQuery(userId, {
  skip: !userId,
});
```

#### 5.6.3 Optimistic Updates — Cập nhật UI trước khi server phản hồi

```js
updatePost: builder.mutation({
  query: ({ id, ...patch }) => ({
    url: `/posts/${id}`,
    method: 'PATCH',
    body: patch,
  }),
  async onQueryStarted({ id, ...patch }, { dispatch, queryFulfilled }) {
    // Cập nhật cache ngay lập tức
    const patchResult = dispatch(
      postsApi.util.updateQueryData('getPostById', id, (draft) => {
        Object.assign(draft, patch);
      })
    );
    try {
      await queryFulfilled;
    } catch {
      // Rollback nếu server báo lỗi
      patchResult.undo();
    }
  },
}),
```

#### 5.6.4 Transforming Responses

```js
getPosts: builder.query({
  query: () => '/posts',
  // Biến đổi dữ liệu trước khi lưu vào cache
  transformResponse: (response) => ({
    posts: response,
    total: response.length,
    ids: response.map((p) => p.id),
  }),
  // Biến đổi lỗi
  transformErrorResponse: (response) => ({
    status: response.status,
    message: response.data?.message ?? 'Lỗi không xác định',
  }),
}),
```

#### 5.6.5 Lazy Query — Fetch thủ công

```jsx
import { useLazyGetPostByIdQuery } from '../services/postsApi';

function SearchPost() {
  const [trigger, result] = useLazyGetPostByIdQuery();

  return (
    <div>
      <button onClick={() => trigger(1)}>Tải bài viết #1</button>
      {result.data && <p>{result.data.title}</p>}
    </div>
  );
}
```

---

## 6. So Sánh và Khuyến Nghị

### 6.1 Bảng so sánh tổng thể

| Tiêu chí | Redux Core | Redux Thunk | Redux Toolkit | RTK Query |
|---|---|---|---|---|
| Mục đích chính | Client state | Client state + Async | Client state (tối giản) | Server state |
| Boilerplate | Rất nhiều | Nhiều | Ít | Gần như không có |
| Cấu hình | Thủ công | Thủ công + middleware | `configureStore` | `createApi` |
| Async handling | Không | Thủ công | `createAsyncThunk` | Tự động |
| Caching | Không | Không | Không | Tự động |
| Loading/Error state | Thủ công | Thủ công | Tự động (với thunk) | Tự động hoàn toàn |
| Học thuộc | Trung bình | Trung bình | Thấp | Thấp |
| Khuyến nghị dùng | Học lý thuyết | Legacy code | Ứng dụng mới | Tương tác với API |

### 6.2 Khi nào dùng gì

**Dùng Redux Core** khi cần hiểu sâu cơ chế hoạt động hoặc làm việc với các dự án rất cũ.

**Dùng Redux Thunk** khi đang maintain codebase cũ chưa dùng RTK, hoặc cần logic async đặc thù không phù hợp với `createAsyncThunk`.

**Dùng Redux Toolkit** cho mọi dự án mới cần client state management. Đây là chuẩn chính thức được khuyến nghị bởi đội Redux.

**Dùng RTK Query** khi ứng dụng cần tương tác nhiều với REST API hoặc GraphQL. RTK Query thay thế hoàn toàn việc viết thunk để fetch data, đồng thời xử lý caching và synchronization tự động.

### 6.3 Kiến trúc dự án thực tế

Trong một ứng dụng React hiện đại, thường kết hợp cả RTK + RTK Query theo cấu trúc:

```
src/
├── app/
│   └── store.js              # configureStore
├── features/
│   ├── auth/
│   │   └── authSlice.js      # RTK createSlice (client state: token, user info)
│   ├── ui/
│   │   └── uiSlice.js        # RTK createSlice (sidebar open/close, theme...)
│   └── cart/
│       └── cartSlice.js      # RTK createSlice (giỏ hàng, chỉ client state)
└── services/
    ├── productsApi.js         # RTK Query createApi
    ├── ordersApi.js           # RTK Query createApi
    └── usersApi.js            # RTK Query createApi
```

**Quy tắc phân chia:**
- **RTK Slice** → trạng thái thuộc về client (UI state, authentication, shopping cart…).
- **RTK Query** → dữ liệu đến từ server (danh sách sản phẩm, đơn hàng, người dùng…).

---

## Tổng Kết

Redux đã trải qua quá trình phát triển đáng kể từ một thư viện đòi hỏi nhiều cấu hình thủ công đến một hệ sinh thái hoàn chỉnh với Redux Toolkit và RTK Query. Lộ trình học đề xuất:

1. **Nắm vững Redux Core** để hiểu store, action, reducer và data flow một chiều.
2. **Tiếp cận Redux Thunk** để hiểu cách middleware hoạt động và tại sao cần xử lý async bên ngoài reducer.
3. **Chuyển sang Redux Toolkit** như là cách viết Redux chuẩn, hiện đại cho mọi dự án.
4. **Áp dụng RTK Query** để loại bỏ toàn bộ boilerplate liên quan đến fetching và caching dữ liệu từ server.

Kết hợp RTK + RTK Query hiện nay là cách tiếp cận được cộng đồng và đội ngũ Redux chính thức khuyến nghị cho các ứng dụng React trong môi trường production.

---

*Tài liệu tham khảo: Redux Official Documentation (redux.js.org) · Redux Toolkit Docs (redux-toolkit.js.org) · React Redux Docs (react-redux.js.org)*
