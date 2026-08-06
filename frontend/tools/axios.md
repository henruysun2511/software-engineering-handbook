# Axios – Tổng hợp kiến thức đầy đủ (React / Next.js)

> Docs chính thức: https://axios-http.com/docs/intro

---

## 1. Axios là gì?

Axios là thư viện HTTP client cho **browser và Node.js**, dựa trên Promise. So với `fetch` API native, Axios cung cấp nhiều tính năng hơn out-of-the-box.

**So sánh Axios vs Fetch:**

| | Axios | Fetch |
|---|---|---|
| Tự động parse JSON | ✅ | ❌ phải `.json()` thủ công |
| Request timeout | ✅ built-in | ❌ phải dùng AbortController |
| Interceptors | ✅ | ❌ |
| Upload progress | ✅ | ❌ |
| Tự động transform data | ✅ | ❌ |
| Error khi 4xx/5xx | ✅ throw Error | ❌ không throw, phải check `res.ok` |
| Cancel request | ✅ `AbortController` | ✅ |
| Bundle size | ~13KB | 0KB (native) |

---

## 2. Cài đặt

```bash
npm install axios
```

---

## 3. Request cơ bản

```typescript
import axios from 'axios';

// GET
const res = await axios.get('/api/users');
console.log(res.data);        // data đã parse JSON tự động

// POST
const res = await axios.post('/api/users', {
  name: 'Alice',
  email: 'alice@example.com',
});

// PUT
await axios.put(`/api/users/${id}`, { name: 'Alice Updated' });

// PATCH
await axios.patch(`/api/users/${id}`, { name: 'Alice' });

// DELETE
await axios.delete(`/api/users/${id}`);

// Cú pháp config object (linh hoạt hơn)
const res = await axios({
  method: 'POST',
  url: '/api/users',
  data: { name: 'Alice' },
  headers: { 'X-Custom-Header': 'value' },
  params: { page: 1, limit: 10 },
  timeout: 5000,
});
```

---

## 4. Response Structure

```typescript
const res = await axios.get('/api/users');

// res là AxiosResponse object
console.log(res.data);       // body response (đã parse JSON)
console.log(res.status);     // HTTP status code: 200, 201...
console.log(res.statusText); // "OK", "Created"...
console.log(res.headers);    // response headers
console.log(res.config);     // config đã dùng để tạo request
console.log(res.request);    // request object gốc

// Thường chỉ cần .data
const users = res.data;
```

---

## 5. Axios Instance – Cấu hình tập trung

Tạo instance riêng thay vì dùng `axios` global — dễ cấu hình baseURL, headers, timeout một lần.

```typescript
// lib/axiosInstance.ts
import axios from 'axios';

const axiosInstance = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3000',
  timeout: 15_000,             // 15 giây
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  },
  withCredentials: true,       // gửi cookie theo mỗi request
});

export default axiosInstance;

// Dùng trong service
import axiosInstance from '../lib/axiosInstance';

const users = await axiosInstance.get('/users').then(r => r.data);
```

---

## 6. Interceptors – Tính năng quan trọng nhất

Interceptors cho phép **can thiệp vào request/response** trước khi chúng được xử lý.

### 6.1. Request Interceptor

```typescript
// Tự động đính kèm JWT token vào mọi request
axiosInstance.interceptors.request.use(
  (config) => {
    // Lấy token từ store hoặc localStorage
    const token = localStorage.getItem('accessToken');
    // hoặc: const token = useAuthStore.getState().accessToken;

    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }

    // Log request khi development
    if (process.env.NODE_ENV === 'development') {
      console.log(`→ ${config.method?.toUpperCase()} ${config.url}`, config.data);
    }

    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);
```

### 6.2. Response Interceptor

```typescript
// Xử lý response và error tập trung
axiosInstance.interceptors.response.use(
  (response) => {
    // Response thành công — có thể transform data ở đây
    return response;
  },
  (error) => {
    const status = error.response?.status;

    switch (status) {
      case 401:
        // Unauthorized — token hết hạn
        // Xử lý ở interceptor riêng (xem phần 7)
        break;
      case 403:
        // Forbidden — không có quyền
        toast.error('Bạn không có quyền thực hiện hành động này');
        break;
      case 404:
        toast.error('Không tìm thấy tài nguyên');
        break;
      case 422:
        // Validation error từ server
        break;
      case 500:
        toast.error('Lỗi server, vui lòng thử lại sau');
        break;
      default:
        if (!navigator.onLine) {
          toast.error('Không có kết nối internet');
        }
    }

    return Promise.reject(error);
  }
);
```

### 6.3. Xóa Interceptor

```typescript
// Lưu lại ID để xóa sau
const requestInterceptor = axiosInstance.interceptors.request.use(fn);
const responseInterceptor = axiosInstance.interceptors.response.use(fn);

// Xóa
axiosInstance.interceptors.request.eject(requestInterceptor);
axiosInstance.interceptors.response.eject(responseInterceptor);
```

---

## 7. Auto Refresh Token với Interceptor

Pattern quan trọng nhất khi tích hợp JWT Authentication.

```typescript
// lib/axiosInstance.ts
import axios, { AxiosInstance } from 'axios';

let isRefreshing = false;
let failedQueue: Array<{
  resolve: (token: string) => void;
  reject: (error: any) => void;
}> = [];

// Xử lý các request đang chờ trong lúc refresh token
const processQueue = (error: any, token: string | null = null) => {
  failedQueue.forEach((prom) => {
    if (error) {
      prom.reject(error);
    } else {
      prom.resolve(token!);
    }
  });
  failedQueue = [];
};

const axiosInstance: AxiosInstance = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  withCredentials: true, // gửi refresh token cookie
});

// Request interceptor — đính kèm access token
axiosInstance.interceptors.request.use((config) => {
  const token = useAuthStore.getState().accessToken;
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Response interceptor — tự động refresh khi 401
axiosInstance.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // Chỉ xử lý 401 và chưa retry
    if (error.response?.status !== 401 || originalRequest._retry) {
      return Promise.reject(error);
    }

    if (isRefreshing) {
      // Đang refresh → đưa request vào hàng đợi
      return new Promise((resolve, reject) => {
        failedQueue.push({ resolve, reject });
      })
        .then((token) => {
          originalRequest.headers.Authorization = `Bearer ${token}`;
          return axiosInstance(originalRequest);
        })
        .catch((err) => Promise.reject(err));
    }

    // Bắt đầu refresh
    originalRequest._retry = true;
    isRefreshing = true;

    try {
      // Gọi refresh endpoint — refresh token gửi qua cookie tự động
      const { data } = await axios.post(
        `${process.env.NEXT_PUBLIC_API_URL}/auth/refresh`,
        {},
        { withCredentials: true }
      );

      const newToken = data.accessToken;
      useAuthStore.getState().setToken(newToken);

      // Cập nhật header cho request gốc
      originalRequest.headers.Authorization = `Bearer ${newToken}`;

      // Xử lý tất cả request đang đợi
      processQueue(null, newToken);

      return axiosInstance(originalRequest); // retry request gốc
    } catch (refreshError) {
      // Refresh thất bại → logout
      processQueue(refreshError, null);
      useAuthStore.getState().logout();
      window.location.href = '/login';
      return Promise.reject(refreshError);
    } finally {
      isRefreshing = false;
    }
  }
);

export default axiosInstance;
```

---

## 8. Query Params

```typescript
// Truyền params qua config
const res = await axiosInstance.get('/users', {
  params: {
    page: 1,
    limit: 10,
    search: 'alice',
    role: 'admin',
    isActive: true,
  },
  // → GET /users?page=1&limit=10&search=alice&role=admin&isActive=true
});

// Params là array
const res = await axiosInstance.get('/products', {
  params: { ids: [1, 2, 3] },
  // Default: /products?ids[]=1&ids[]=2&ids[]=3
  // Tùy chỉnh serializer:
  paramsSerializer: {
    serialize: (params) => {
      // /products?ids=1,2,3
      return Object.entries(params)
        .map(([k, v]) => `${k}=${Array.isArray(v) ? v.join(',') : v}`)
        .join('&');
    },
  },
});

// Dùng URLSearchParams
const params = new URLSearchParams({ page: '1', limit: '10' });
await axiosInstance.get(`/users?${params}`);
```

---

## 9. Upload File

```typescript
// Upload single file
async function uploadAvatar(file: File) {
  const formData = new FormData();
  formData.append('file', file);
  formData.append('type', 'avatar');

  const res = await axiosInstance.post('/upload', formData, {
    headers: {
      'Content-Type': 'multipart/form-data', // Axios tự set, nhưng để rõ ràng
    },
    // Theo dõi tiến trình upload
    onUploadProgress: (progressEvent) => {
      const percent = Math.round(
        (progressEvent.loaded * 100) / (progressEvent.total ?? 1)
      );
      console.log(`Upload: ${percent}%`);
      setUploadProgress(percent);
    },
  });

  return res.data;
}

// Upload nhiều file
async function uploadFiles(files: FileList) {
  const formData = new FormData();
  Array.from(files).forEach((file) => {
    formData.append('files', file);
  });

  return axiosInstance.post('/upload/bulk', formData);
}

// Download file
async function downloadFile(fileId: string, filename: string) {
  const res = await axiosInstance.get(`/files/${fileId}/download`, {
    responseType: 'blob', // quan trọng
    onDownloadProgress: (progressEvent) => {
      const percent = Math.round(
        (progressEvent.loaded * 100) / (progressEvent.total ?? 1)
      );
      setDownloadProgress(percent);
    },
  });

  // Tạo link tải về
  const url = window.URL.createObjectURL(new Blob([res.data]));
  const link = document.createElement('a');
  link.href = url;
  link.setAttribute('download', filename);
  document.body.appendChild(link);
  link.click();
  link.remove();
  window.URL.revokeObjectURL(url);
}
```

---

## 10. Cancel Request với AbortController

```typescript
import { useEffect } from 'react';
import axiosInstance from '../lib/axiosInstance';

// Trong React component — cancel khi unmount hoặc dependency thay đổi
function SearchResults({ query }: { query: string }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    const controller = new AbortController();

    const search = async () => {
      try {
        const res = await axiosInstance.get('/search', {
          params: { q: query },
          signal: controller.signal, // đính kèm signal
        });
        setResults(res.data);
      } catch (err) {
        if (axios.isCancel(err)) {
          console.log('Request cancelled'); // không phải lỗi thật
        } else {
          console.error(err);
        }
      }
    };

    if (query) search();

    // Cleanup: cancel request khi query thay đổi hoặc component unmount
    return () => controller.abort();
  }, [query]);

  return <ul>{results.map(r => <li key={r.id}>{r.name}</li>)}</ul>;
}

// Cancel thủ công
const controller = new AbortController();
const promise = axiosInstance.get('/long-running-task', {
  signal: controller.signal,
});

// Cancel sau 5 giây
setTimeout(() => controller.abort(), 5000);
```

---

## 11. Error Handling

```typescript
import axios, { AxiosError } from 'axios';

async function fetchUser(id: string) {
  try {
    const res = await axiosInstance.get(`/users/${id}`);
    return res.data;
  } catch (error) {
    if (axios.isAxiosError(error)) {
      // Lỗi từ Axios (có response hoặc không)
      const axiosError = error as AxiosError<{ message: string; errors: any[] }>;

      if (axiosError.response) {
        // Server trả về response với status code lỗi
        const { status, data } = axiosError.response;
        console.error(`Server error ${status}:`, data.message);
        throw new Error(data.message || `HTTP Error ${status}`);

      } else if (axiosError.request) {
        // Request đã gửi nhưng không nhận được response
        // (network error, timeout, server down)
        throw new Error('Không thể kết nối đến server');

      } else {
        // Lỗi khi cấu hình request
        throw new Error('Lỗi cấu hình request');
      }
    }

    // Lỗi không phải từ Axios
    throw error;
  }
}

// Type-safe error helper
export function getErrorMessage(error: unknown): string {
  if (axios.isAxiosError(error)) {
    return error.response?.data?.message
      ?? error.response?.data?.error
      ?? error.message
      ?? 'Đã xảy ra lỗi';
  }
  if (error instanceof Error) return error.message;
  return 'Đã xảy ra lỗi không xác định';
}
```

---

## 12. Timeout

```typescript
// Timeout toàn cục
const axiosInstance = axios.create({
  timeout: 15_000, // 15 giây
});

// Timeout cho từng request
await axiosInstance.get('/slow-endpoint', {
  timeout: 5000, // override 5 giây cho request này
});

// Xử lý timeout error
try {
  await axiosInstance.get('/api/data');
} catch (error) {
  if (axios.isAxiosError(error) && error.code === 'ECONNABORTED') {
    console.error('Request timeout!');
  }
}
```

---

## 13. Concurrent Requests

```typescript
import axios from 'axios';

// Gọi nhiều API song song
const [users, products, orders] = await Promise.all([
  axiosInstance.get('/users').then(r => r.data),
  axiosInstance.get('/products').then(r => r.data),
  axiosInstance.get('/orders').then(r => r.data),
]);

// axios.all + axios.spread (legacy, dùng Promise.all thay thế)
const [userRes, productRes] = await axios.all([
  axiosInstance.get('/users'),
  axiosInstance.get('/products'),
]);

// Với error handling riêng từng request
const results = await Promise.allSettled([
  axiosInstance.get('/users'),
  axiosInstance.get('/products'),
]);

results.forEach((result) => {
  if (result.status === 'fulfilled') {
    console.log(result.value.data);
  } else {
    console.error('Failed:', result.reason.message);
  }
});
```

---

## 14. TypeScript – Generic Types

```typescript
import axios, { AxiosResponse } from 'axios';

interface User {
  id: string;
  name: string;
  email: string;
}

interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  totalPages: number;
}

// Type-safe response
const res: AxiosResponse<User> = await axiosInstance.get<User>(`/users/${id}`);
const user: User = res.data; // TypeScript biết đây là User

// Generic API function
async function get<T>(url: string, params?: object): Promise<T> {
  const res = await axiosInstance.get<T>(url, { params });
  return res.data;
}

async function post<TRequest, TResponse>(
  url: string,
  data: TRequest
): Promise<TResponse> {
  const res = await axiosInstance.post<TResponse>(url, data);
  return res.data;
}

// Dùng
const users = await get<PaginatedResponse<User>>('/users', { page: 1 });
const newUser = await post<CreateUserDto, User>('/users', { name: 'Alice', email: 'a@b.com' });
```

---

## 15. Tổ chức API Layer

Pattern tổ chức code sạch cho dự án thực tế.

```typescript
// lib/axiosInstance.ts — 1 instance duy nhất
import axios from 'axios';

const axiosInstance = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  timeout: 15_000,
  withCredentials: true,
});

// Setup interceptors
setupRequestInterceptor(axiosInstance);
setupResponseInterceptor(axiosInstance);

export default axiosInstance;
```

```typescript
// api/users.api.ts — Tất cả endpoint của users
import axiosInstance from '../lib/axiosInstance';
import { User, CreateUserDto, UpdateUserDto, PaginatedResponse } from '../types';

export const usersApi = {
  getAll: (params?: { page?: number; limit?: number; search?: string }) =>
    axiosInstance
      .get<PaginatedResponse<User>>('/users', { params })
      .then((r) => r.data),

  getById: (id: string) =>
    axiosInstance.get<User>(`/users/${id}`).then((r) => r.data),

  create: (data: CreateUserDto) =>
    axiosInstance.post<User>('/users', data).then((r) => r.data),

  update: (id: string, data: UpdateUserDto) =>
    axiosInstance.put<User>(`/users/${id}`, data).then((r) => r.data),

  patch: (id: string, data: Partial<UpdateUserDto>) =>
    axiosInstance.patch<User>(`/users/${id}`, data).then((r) => r.data),

  delete: (id: string) =>
    axiosInstance.delete(`/users/${id}`).then((r) => r.data),

  uploadAvatar: (id: string, file: File) => {
    const formData = new FormData();
    formData.append('avatar', file);
    return axiosInstance
      .post<{ avatarUrl: string }>(`/users/${id}/avatar`, formData)
      .then((r) => r.data);
  },
};

// api/index.ts — Export tất cả
export { usersApi } from './users.api';
export { productsApi } from './products.api';
export { ordersApi } from './orders.api';
export { authApi } from './auth.api';
```

```typescript
// Dùng trong component / custom hook
import { usersApi } from '../api';

function UserPage() {
  const [users, setUsers] = useState<User[]>([]);

  useEffect(() => {
    usersApi.getAll({ page: 1, limit: 10 }).then(setUsers);
  }, []);
}

// Hoặc kết hợp với TanStack Query
import { useQuery } from '@tanstack/react-query';
import { usersApi } from '../api';

function useUsers(params?: { page?: number }) {
  return useQuery({
    queryKey: ['users', params],
    queryFn: () => usersApi.getAll(params),
  });
}
```

---

## 16. Retry Logic thủ công

```typescript
// Retry wrapper
async function axiosWithRetry<T>(
  requestFn: () => Promise<T>,
  retries = 3,
  delay = 1000
): Promise<T> {
  try {
    return await requestFn();
  } catch (error) {
    if (retries === 0) throw error;

    // Không retry với lỗi 4xx (client error)
    if (axios.isAxiosError(error) && error.response?.status < 500) {
      throw error;
    }

    await new Promise((resolve) => setTimeout(resolve, delay));
    return axiosWithRetry(requestFn, retries - 1, delay * 2); // exponential backoff
  }
}

// Dùng
const data = await axiosWithRetry(() => axiosInstance.get('/api/data').then(r => r.data));
```

---

## 17. Axios trong Next.js

```typescript
// Next.js Server Component / API Route — dùng axios bình thường
// pages/api/users.ts hoặc app/api/users/route.ts
import axiosInstance from '../../lib/axiosInstance';

export async function GET() {
  try {
    // Gọi backend API từ server
    const users = await axiosInstance.get('/users').then(r => r.data);
    return Response.json(users);
  } catch (error) {
    return Response.json({ error: 'Failed' }, { status: 500 });
  }
}

// Tạo instance riêng cho Server (không cần cookie, có thể có API key khác)
export const serverAxios = axios.create({
  baseURL: process.env.INTERNAL_API_URL, // URL nội bộ, không expose ra client
  headers: {
    'X-API-Key': process.env.INTERNAL_API_KEY,
  },
  timeout: 30_000,
});

// Client instance — NEXT_PUBLIC_ prefix để expose ra browser
export const clientAxios = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  withCredentials: true,
});
```

---

## 18. Checklist Axios

- [ ] Tạo **Axios instance** với `baseURL`, `timeout`, `withCredentials`
- [ ] **Request interceptor** tự động đính kèm JWT token
- [ ] **Response interceptor** xử lý lỗi tập trung (401, 403, 500)
- [ ] Implement **auto refresh token** với queue cho concurrent requests
- [ ] Dùng `axios.isAxiosError()` để phân biệt lỗi Axios vs lỗi khác
- [ ] **Cancel request** với `AbortController` trong `useEffect`
- [ ] Tổ chức **API layer** riêng (`usersApi`, `productsApi`) — không gọi axios trực tiếp trong component
- [ ] Dùng **Generic types** `axios.get<User>()` cho TypeScript type-safe
- [ ] Xử lý **upload progress** với `onUploadProgress`
- [ ] Không hardcode URL — dùng biến môi trường `process.env`
- [ ] Tạo **2 instance riêng** cho client và server trong Next.js
