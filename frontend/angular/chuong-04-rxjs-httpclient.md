# Chương 4: RxJS & HttpClient

## Giới thiệu chương

RxJS (Reactive Extensions for JavaScript) là một trong những thành phần bắt buộc của Angular. Không giống như React — nơi bạn có thể sử dụng hoàn toàn với `async/await` và không cần biết đến RxJS — Angular tích hợp RxJS vào tận core của framework: Router phát ra Observable, HttpClient trả về Observable, Reactive Forms có `valueChanges` và `statusChanges` là Observable, Angular Material dùng Observable ở nhiều nơi.

Chương này không cố dạy toàn bộ RxJS — điều đó sẽ tốn cả một cuốn sách riêng. Thay vào đó, chương tập trung vào **những gì cần thiết trong thực tế Angular**: hiểu đúng các khái niệm cốt lõi, nắm vững các operators hay dùng nhất, và kết hợp hiệu quả với HttpClient và Zod.

---

## 4.1 Observable — Nền tảng của RxJS

### Observable là gì

Một **Observable** là một nguồn dữ liệu có thể phát ra nhiều giá trị theo thời gian. Điều này khác với Promise — Promise chỉ resolve một lần với một giá trị duy nhất.

```
Promise:   ──────────────●  (một giá trị duy nhất, sau đó kết thúc)
Observable: ──●──●──●──●──X  (nhiều giá trị, có thể kết thúc hoặc báo lỗi)
Observable: ──●──●──●──●──▶  (stream vô tận như WebSocket, interval)
```

```typescript
import { Observable, of, from, interval, fromEvent } from 'rxjs';

// Tạo Observable từ giá trị tĩnh
const static$ = of(1, 2, 3);
// Phát: 1, 2, 3 rồi complete

// Tạo từ array/Promise
const fromArray$ = from([10, 20, 30]);
const fromPromise$ = from(fetch('/api/data').then(r => r.json()));

// Tạo từ interval
const ticker$ = interval(1000);
// Phát: 0, 1, 2, 3... mỗi giây, không bao giờ complete

// Tạo từ DOM event
const clicks$ = fromEvent(document, 'click');

// Tạo Observable tùy chỉnh
const custom$ = new Observable<number>((observer) => {
  observer.next(1);
  observer.next(2);

  setTimeout(() => {
    observer.next(3);
    observer.complete();
  }, 1000);

  // Cleanup function — chạy khi unsubscribe
  return () => {
    console.log('Unsubscribed');
  };
});
```

### Subscribe — Kích hoạt Observable

Observable là **lazy** — không có gì xảy ra cho đến khi có ai đó `.subscribe()`. Đây là điểm khác biệt quan trọng so với Promise (eager — chạy ngay khi tạo).

```typescript
// HttpClient.get() trả về Observable — CHƯA gọi API
const request$ = this.http.get<User[]>('/api/users');

// Chỉ khi subscribe mới thực sự gọi API
const subscription = request$.subscribe({
  next: (users) => console.log('Nhận được:', users),
  error: (err) => console.error('Lỗi:', err),
  complete: () => console.log('Hoàn thành'),
});

// Hủy subscription — hủy luôn HTTP request đang chờ
subscription.unsubscribe();
```

### Hot vs Cold Observable — Khái niệm quan trọng nhất

**Cold Observable**: Mỗi subscriber nhận toàn bộ sequence từ đầu. HttpClient là Cold — mỗi lần subscribe là một HTTP request mới.

```typescript
const request$ = this.http.get<User[]>('/api/users');

// Hai subscribe = hai HTTP request riêng biệt
request$.subscribe(users => console.log('Sub 1:', users));
request$.subscribe(users => console.log('Sub 2:', users));
```

**Hot Observable**: Nhiều subscriber chia sẻ cùng một nguồn dữ liệu. Subject là Hot — subscriber mới chỉ nhận giá trị từ thời điểm subscribe.

```typescript
const subject = new Subject<number>();

subject.subscribe(v => console.log('Sub 1:', v));
subject.next(1); // Sub 1: 1

subject.subscribe(v => console.log('Sub 2:', v));
subject.next(2); // Sub 1: 2, Sub 2: 2
subject.next(3); // Sub 1: 3, Sub 2: 3
```

**`shareReplay` — chuyển Cold thành Hot có cache:**

```typescript
@Injectable({ providedIn: 'root' })
export class ConfigService {
  private readonly http = inject(HttpClient);

  // Không dùng shareReplay — mỗi component inject ConfigService
  // và gọi getConfig() sẽ tạo ra một HTTP request mới
  getConfigBad(): Observable<AppConfig> {
    return this.http.get<AppConfig>('/api/config');
  }

  // Dùng shareReplay(1) — chỉ một HTTP request, kết quả được cache
  // và chia sẻ cho tất cả subscriber
  readonly config$ = this.http.get<AppConfig>('/api/config').pipe(
    shareReplay(1)
  );
}
```

### Subject và các biến thể

```typescript
// Subject — Hot Observable, không có initial value
const subject = new Subject<string>();
subject.next('A'); // Subscriber chưa có → mất
const sub = subject.subscribe(console.log);
subject.next('B'); // In: B
subject.next('C'); // In: C

// BehaviorSubject — giữ giá trị hiện tại, subscriber mới nhận ngay
const behavior = new BehaviorSubject<number>(0); // Initial value: 0
behavior.subscribe(v => console.log('Sub 1:', v)); // In ngay: 0
behavior.next(1); // In: 1
behavior.subscribe(v => console.log('Sub 2:', v)); // In ngay: 1 (giá trị hiện tại)
behavior.next(2); // In: 2, 2

// ReplaySubject — buffer N giá trị gần nhất cho subscriber mới
const replay = new ReplaySubject<string>(3); // Buffer 3 giá trị
replay.next('X');
replay.next('Y');
replay.next('Z');
replay.next('W');
replay.subscribe(v => console.log(v)); // In: Y, Z, W (3 giá trị cuối)
```

**Khi nào dùng loại nào:**

| Subject | Dùng khi |
|---------|----------|
| `Subject` | Event bus, fire-and-forget, không cần giá trị ban đầu |
| `BehaviorSubject` | State management — luôn có current value (giỏ hàng, user đăng nhập) |
| `ReplaySubject` | Cache N event gần nhất (WebSocket messages, notification history) |

---

## 4.2 Operators — Trái tim của RxJS

### Hiểu Operator Pipeline

Operators là pure functions nhận Observable và trả về Observable mới. Chúng được chain bằng `.pipe()`:

```typescript
source$.pipe(
  operator1(),
  operator2(),
  operator3(),
).subscribe(result => console.log(result));
```

Mỗi operator tạo ra một Observable mới — không mutate Observable gốc.

### map, filter, tap — Operators cơ bản

```typescript
import { map, filter, tap } from 'rxjs/operators';

this.http.get<ApiResponse<User[]>>('/api/users').pipe(
  // tap — side effect, không thay đổi giá trị (dùng để log, loading state)
  tap(() => this.isLoading.set(true)),

  // map — transform giá trị
  map((response) => response.data),

  // filter — chỉ cho qua giá trị thỏa điều kiện
  map((users) => users.filter((u) => u.isActive)),

  // map lần nữa — transform tiếp
  map((users) =>
    users.sort((a, b) => a.displayName.localeCompare(b.displayName))
  ),

  // tap để kết thúc loading
  tap(() => this.isLoading.set(false)),
).subscribe({
  next: (users) => this.users.set(users),
  error: () => this.isLoading.set(false),
});
```

### Flattening Operators — Quan trọng nhất cần hiểu

Đây là nhóm operators hay gây nhầm lẫn nhất nhưng cực kỳ quan trọng trong Angular. Tất cả đều nhận Observable phát ra Observable (Observable of Observables) và "flatten" chúng:

**`switchMap` — Hủy request cũ khi có request mới**

```typescript
// Use case: Search autocomplete, route params change
// Khi user gõ query mới → hủy request tìm kiếm cũ, bắt đầu request mới

searchQuery$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap((query) =>
    // Nếu query thay đổi trước khi request hoàn thành,
    // request cũ bị hủy tự động
    this.searchService.search(query)
  )
).subscribe(results => this.results.set(results));

// Minh họa behavior:
// query: 'ang' ──────────────────────●(results)
// query: 'angu' ──────●(results)
// query: 'angul' ────────────────────●(results)
// switchMap output:              ●(results cho 'angul')
// (request của 'ang' và 'angu' bị hủy)
```

**`mergeMap` — Thực thi tất cả song song**

```typescript
// Use case: Upload nhiều file cùng lúc, không quan tâm thứ tự
// Tất cả requests chạy song song, output theo thứ tự hoàn thành

selectedFiles$.pipe(
  mergeMap((file) =>
    this.uploadService.upload(file).pipe(
      map((result) => ({ file, result }))
    )
  )
).subscribe(({ file, result }) => {
  this.markFileAsUploaded(file.id, result.url);
});
```

**`concatMap` — Thực thi tuần tự, đợi cái trước xong mới làm cái sau**

```typescript
// Use case: Chuỗi action phải theo thứ tự (step 1 → step 2 → step 3)
// Tạo user → tạo profile → gửi welcome email

of(userDto).pipe(
  concatMap((dto) => this.userService.createUser(dto)),
  concatMap((user) => this.profileService.createProfile(user.id)),
  concatMap((profile) => this.emailService.sendWelcome(profile.userId))
).subscribe({
  next: () => this.notificationService.success('Tạo tài khoản thành công'),
  error: (err) => this.notificationService.error(err.message),
});
```

**`exhaustMap` — Bỏ qua event mới khi đang xử lý**

```typescript
// Use case: Submit button — bỏ qua click tiếp theo khi request chưa xong
// Ngăn double-submit

fromEvent(submitButton, 'click').pipe(
  exhaustMap(() =>
    // Click tiếp theo trong khi request đang chạy sẽ bị bỏ qua hoàn toàn
    this.formService.submit(this.form.value)
  )
).subscribe({
  next: () => this.router.navigate(['/success']),
  error: (err) => this.showError(err),
});
```

**Bảng quyết định nhanh:**

| Operator | Khi nào dùng | Ví dụ |
|----------|--------------|-------|
| `switchMap` | Request mới làm vô hiệu request cũ | Search, route change |
| `mergeMap` | Tất cả chạy song song | Multi-file upload |
| `concatMap` | Thứ tự quan trọng, tuần tự | Wizard steps, ordered actions |
| `exhaustMap` | Bỏ qua khi đang bận | Submit button, login button |

### Error Handling

```typescript
import { catchError, retry, retryWhen, timer } from 'rxjs';

// catchError — bắt lỗi và trả về giá trị dự phòng
this.http.get<User[]>('/api/users').pipe(
  catchError((error: HttpErrorResponse) => {
    if (error.status === 404) {
      return of([]); // Trả về array rỗng thay vì lỗi
    }
    return throwError(() => error); // Re-throw lỗi khác
  })
);

// retry — tự động thử lại N lần
this.http.get<Config>('/api/config').pipe(
  retry({ count: 3, delay: 1000 }) // Thử lại 3 lần, cách nhau 1 giây
);

// Exponential backoff — thử lại với delay tăng dần
this.http.get<Data>('/api/data').pipe(
  retryWhen((errors) =>
    errors.pipe(
      mergeMap((error, attempt) => {
        if (attempt >= 3) return throwError(() => error);
        const delay = Math.pow(2, attempt) * 1000; // 1s, 2s, 4s
        return timer(delay);
      })
    )
  )
);
```

### Combination Operators

```typescript
import { combineLatest, forkJoin, merge } from 'rxjs';

// forkJoin — đợi tất cả hoàn thành, lấy giá trị cuối cùng của mỗi cái
// Dùng cho: load data song song, tất cả phải xong mới dùng
forkJoin({
  users: this.userService.getUsers(),
  roles: this.roleService.getRoles(),
  departments: this.departmentService.getDepartments(),
}).subscribe(({ users, roles, departments }) => {
  // Tất cả đã sẵn sàng
  this.initializeForm(users, roles, departments);
});

// combineLatest — emit khi bất kỳ source nào thay đổi
// Dùng cho: filter + search + sort kết hợp
combineLatest({
  search: this.searchQuery$,
  role: this.selectedRole$,
  page: this.currentPage$,
}).pipe(
  debounceTime(100), // Tránh emit quá nhiều lần khi nhiều filter thay đổi cùng lúc
  switchMap(({ search, role, page }) =>
    this.userService.getUsers({ search, role, page })
  )
).subscribe(users => this.users.set(users));

// merge — gộp nhiều streams thành một
// Dùng cho: lắng nghe nhiều nguồn event
merge(
  fromEvent(document, 'keydown'),
  fromEvent(document, 'mousedown'),
  fromEvent(document, 'touchstart'),
).pipe(
  debounceTime(60_000) // Reset sau 60 giây không hoạt động
).subscribe(() => this.sessionService.resetTimer());
```

### Utility Operators hay dùng

```typescript
import {
  debounceTime, distinctUntilChanged, startWith,
  finalize, takeUntil, takeUntilDestroyed,
  share, shareReplay
} from 'rxjs/operators';

// debounceTime — đợi X ms sau emit cuối cùng
searchInput$.pipe(debounceTime(300));

// distinctUntilChanged — bỏ qua nếu giá trị không thay đổi
searchInput$.pipe(distinctUntilChanged());

// startWith — emit giá trị ban đầu trước khi source emit
userList$.pipe(startWith([]));

// finalize — chạy cleanup dù success hay error
request$.pipe(
  finalize(() => this.isLoading.set(false))
);

// takeUntilDestroyed — unsubscribe tự động khi component destroy
request$.pipe(
  takeUntilDestroyed(this.destroyRef)
);

// shareReplay — cache + chia sẻ cho nhiều subscriber
expensiveRequest$.pipe(
  shareReplay({ bufferSize: 1, refCount: true })
);
```

---

## 4.3 HttpClient — Angular HTTP Layer

### Setup và Cấu hình

```typescript
// app.config.ts
import {
  provideHttpClient,
  withInterceptors,
  withFetch,          // Dùng Fetch API thay XMLHttpRequest (Angular 18)
} from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withFetch(),                              // Sử dụng Fetch API (SSR-compatible)
      withInterceptors([
        authInterceptor,
        errorInterceptor,
        loadingInterceptor,
      ])
    ),
  ],
};
```

### Các HTTP Methods

```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = `${inject(API_URL)}/users`;

  // GET — lấy danh sách
  getUsers(params?: UserQueryParams): Observable<PaginatedResponse<User>> {
    let httpParams = new HttpParams();

    if (params?.page) httpParams = httpParams.set('page', params.page);
    if (params?.limit) httpParams = httpParams.set('limit', params.limit);
    if (params?.search) httpParams = httpParams.set('search', params.search);
    if (params?.role) httpParams = httpParams.set('role', params.role);

    return this.http.get<PaginatedResponse<User>>(this.baseUrl, {
      params: httpParams,
    });
  }

  // GET — lấy một record
  getUserById(id: string): Observable<User> {
    return this.http.get<User>(`${this.baseUrl}/${id}`);
  }

  // POST — tạo mới
  createUser(dto: CreateUserDto): Observable<User> {
    return this.http.post<User>(this.baseUrl, dto);
  }

  // PUT — thay thế toàn bộ
  replaceUser(id: string, dto: CreateUserDto): Observable<User> {
    return this.http.put<User>(`${this.baseUrl}/${id}`, dto);
  }

  // PATCH — cập nhật một phần
  updateUser(id: string, dto: UpdateUserDto): Observable<User> {
    return this.http.patch<User>(`${this.baseUrl}/${id}`, dto);
  }

  // DELETE
  deleteUser(id: string): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`);
  }

  // Upload file với progress tracking
  uploadAvatar(userId: string, file: File): Observable<HttpEvent<{ url: string }>> {
    const formData = new FormData();
    formData.append('avatar', file);

    return this.http.post<{ url: string }>(
      `${this.baseUrl}/${userId}/avatar`,
      formData,
      {
        reportProgress: true,    // Bật progress events
        observe: 'events',       // Nhận tất cả HTTP events, không chỉ response body
      }
    );
  }
}
```

### Đọc HTTP Response đầy đủ

```typescript
// Mặc định HttpClient chỉ trả về response body
// Dùng observe: 'response' để nhận full HttpResponse

this.http
  .get<User>(`/api/users/${id}`, { observe: 'response' })
  .subscribe((response: HttpResponse<User>) => {
    console.log(response.status);          // 200
    console.log(response.headers.get('X-Total-Count')); // Custom header
    console.log(response.body);            // User object
  });

// observe: 'events' cho upload progress
this.uploadService.uploadAvatar(userId, file).pipe(
  filter((event): event is HttpUploadProgressEvent =>
    event.type === HttpEventType.UploadProgress
  ),
  map((event) =>
    event.total ? Math.round((100 * event.loaded) / event.total) : 0
  )
).subscribe((progress) => this.uploadProgress.set(progress));
```

---

## 4.4 HTTP Interceptors — Xử lý Request/Response Toàn cục

Interceptors là nơi xử lý các concern xuyên suốt: authentication, error handling, logging, loading indicator. Angular 15+ dùng functional interceptors — ngắn gọn hơn class-based.

### Auth Interceptor — Gắn JWT Token

```typescript
// core/interceptors/auth.interceptor.ts
import {
  HttpInterceptorFn,
  HttpRequest,
  HttpHandlerFn,
} from '@angular/common/http';

export const authInterceptor: HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
) => {
  const authService = inject(AuthService);
  const token = authService.accessToken();

  // Không gắn token cho public endpoints
  if (!token || isPublicUrl(req.url)) {
    return next(req);
  }

  // Clone request — HttpRequest là immutable
  const authReq = req.clone({
    setHeaders: {
      Authorization: `Bearer ${token}`,
    },
  });

  return next(authReq);
};

const PUBLIC_URLS = ['/api/auth/login', '/api/auth/refresh', '/api/public'];

function isPublicUrl(url: string): boolean {
  return PUBLIC_URLS.some((publicUrl) => url.includes(publicUrl));
}
```

### Error Interceptor — Xử lý lỗi HTTP toàn cục

```typescript
// core/interceptors/error.interceptor.ts
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const notificationService = inject(NotificationService);
  const router = inject(Router);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      switch (error.status) {
        case 401:
          // Token hết hạn — redirect về login
          authService.logout();
          router.navigate(['/auth/login'], {
            queryParams: { returnUrl: router.url },
          });
          break;

        case 403:
          notificationService.error('Bạn không có quyền thực hiện thao tác này');
          router.navigate(['/forbidden']);
          break;

        case 404:
          // 404 được xử lý ở từng service — không show global notification
          break;

        case 422: {
          // Validation error từ server
          const validationErrors = error.error?.errors as Record<string, string[]>;
          if (validationErrors) {
            const messages = Object.values(validationErrors).flat().join(', ');
            notificationService.error(messages);
          }
          break;
        }

        case 429:
          notificationService.error('Quá nhiều yêu cầu. Vui lòng thử lại sau.');
          break;

        case 500:
        case 502:
        case 503:
          notificationService.error('Lỗi máy chủ. Vui lòng thử lại sau.');
          break;

        default:
          if (error.status === 0) {
            // Network error — không có internet hoặc server không phản hồi
            notificationService.error('Không thể kết nối đến máy chủ');
          }
      }

      // Re-throw để service có thể xử lý thêm nếu cần
      return throwError(() => error);
    })
  );
};
```

### Refresh Token Interceptor — Tự động làm mới token

```typescript
// core/interceptors/token-refresh.interceptor.ts
export const tokenRefreshInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      // Chỉ xử lý 401 cho requests có token (không phải login/refresh)
      if (
        error.status !== 401 ||
        req.url.includes('/auth/') ||
        !authService.accessToken()
      ) {
        return throwError(() => error);
      }

      // Refresh token rồi retry request gốc
      return authService.refreshToken().pipe(
        switchMap((newToken) => {
          const retryReq = req.clone({
            setHeaders: { Authorization: `Bearer ${newToken}` },
          });
          return next(retryReq);
        }),
        catchError((refreshError) => {
          // Refresh thất bại → logout
          authService.logout();
          return throwError(() => refreshError);
        })
      );
    })
  );
};
```

### Loading Interceptor

```typescript
// core/interceptors/loading.interceptor.ts
@Injectable({ providedIn: 'root' })
export class LoadingService {
  private readonly activeRequests = signal(0);
  readonly isLoading = computed(() => this.activeRequests() > 0);

  increment(): void {
    this.activeRequests.update((n) => n + 1);
  }

  decrement(): void {
    this.activeRequests.update((n) => Math.max(0, n - 1));
  }
}

export const loadingInterceptor: HttpInterceptorFn = (req, next) => {
  const loadingService = inject(LoadingService);

  // Bỏ qua background requests (polling, analytics)
  if (req.headers.has('X-Skip-Loading')) {
    return next(req);
  }

  loadingService.increment();

  return next(req).pipe(
    finalize(() => loadingService.decrement())
  );
};
```

---

## 4.5 Zod — Validate HTTP Response

### Tại sao cần Zod dù đã có TypeScript

TypeScript chỉ kiểm tra type lúc **compile time**. Khi runtime, data từ API có thể không khớp với type bạn khai báo — API thay đổi, server trả về field không đúng, null thay vì array — TypeScript không thể bắt những lỗi này.

Zod validate data tại **runtime**, đảm bảo data thực sự đúng shape trước khi vào ứng dụng.

```typescript
// Không có Zod — TypeScript tin tưởng API
interface User {
  id: string;
  email: string;
  displayName: string;
}

// API trả về: { id: 123, email: null, display_name: "John" }
// TypeScript: ✓ (không biết)
// Runtime: user.email.toLowerCase() → TypeError: Cannot read properties of null
```

### Định nghĩa Zod Schema

```typescript
// models/user.schema.ts
import { z } from 'zod';

export const UserRoleSchema = z.enum(['ADMIN', 'EDITOR', 'VIEWER']);

export const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  displayName: z.string().min(1).max(100),
  role: UserRoleSchema,
  isActive: z.boolean(),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
  // Optional fields
  avatarUrl: z.string().url().nullable().optional(),
  phoneNumber: z.string().nullable().optional(),
});

// Nested schema
export const AddressSchema = z.object({
  street: z.string(),
  city: z.string(),
  country: z.string().length(2), // ISO country code
});

export const UserDetailSchema = UserSchema.extend({
  address: AddressSchema.nullable(),
  permissions: z.array(z.string()),
});

// Pagination wrapper — tái sử dụng cho nhiều entity
export const PaginatedSchema = <T extends z.ZodTypeAny>(itemSchema: T) =>
  z.object({
    data: z.array(itemSchema),
    pagination: z.object({
      page: z.number().int().positive(),
      limit: z.number().int().positive(),
      total: z.number().int().nonnegative(),
      totalPages: z.number().int().nonnegative(),
    }),
  });

// Infer TypeScript types từ schema — single source of truth
export type User = z.infer<typeof UserSchema>;
export type UserRole = z.infer<typeof UserRoleSchema>;
export type UserDetail = z.infer<typeof UserDetailSchema>;
export type PaginatedUsers = z.infer<ReturnType<typeof PaginatedSchema<typeof UserSchema>>>;
```

### Tích hợp Zod vào HttpClient Service

```typescript
// core/utils/zod-http.util.ts
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import { z } from 'zod';

/**
 * Parse và validate HTTP response với Zod schema.
 * Throw ZodError nếu response không đúng shape.
 */
export function parseResponse<T extends z.ZodTypeAny>(
  schema: T
): (source: Observable<unknown>) => Observable<z.infer<T>> {
  return (source) =>
    source.pipe(
      map((data) => {
        const result = schema.safeParse(data);

        if (!result.success) {
          // Log chi tiết lỗi validation để debug
          console.error('API Response validation failed:', {
            errors: result.error.flatten(),
            data,
          });
          throw new Error(
            `Invalid API response: ${result.error.message}`
          );
        }

        return result.data;
      })
    );
}
```

```typescript
// features/user/services/user.service.ts
import { z } from 'zod';
import {
  UserSchema,
  UserDetailSchema,
  PaginatedSchema,
} from '@features/user/models/user.schema';
import { parseResponse } from '@core/utils/zod-http.util';

const PaginatedUsersSchema = PaginatedSchema(UserSchema);

@Injectable({ providedIn: 'root' })
export class UserService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = `${inject(API_URL)}/users`;

  getUsers(params?: UserQueryParams): Observable<z.infer<typeof PaginatedUsersSchema>> {
    return this.http
      .get<unknown>(this.baseUrl, { params: this.buildParams(params) })
      .pipe(
        parseResponse(PaginatedUsersSchema),
        catchError(this.handleError('getUsers'))
      );
  }

  getUserById(id: string): Observable<z.infer<typeof UserDetailSchema>> {
    return this.http
      .get<unknown>(`${this.baseUrl}/${id}`)
      .pipe(
        parseResponse(UserDetailSchema),
        catchError(this.handleError('getUserById'))
      );
  }

  createUser(dto: CreateUserDto): Observable<z.infer<typeof UserSchema>> {
    return this.http
      .post<unknown>(this.baseUrl, dto)
      .pipe(
        parseResponse(UserSchema),
        catchError(this.handleError('createUser'))
      );
  }

  private buildParams(
    params?: UserQueryParams
  ): Record<string, string | number> {
    const result: Record<string, string | number> = {};
    if (params?.page) result['page'] = params.page;
    if (params?.limit) result['limit'] = params.limit;
    if (params?.search) result['search'] = params.search;
    if (params?.role) result['role'] = params.role;
    return result;
  }

  private handleError(operation: string) {
    return (error: unknown): Observable<never> => {
      console.error(`UserService.${operation} failed:`, error);
      return throwError(() => error);
    };
  }
}
```

### Zod Transform — Chuẩn hóa data từ API

API thường trả về `snake_case` trong khi TypeScript convention dùng `camelCase`. Zod transform giải quyết điều này tại tầng validation:

```typescript
// Khi API trả về snake_case
const ApiUserSchema = z.object({
  id: z.string(),
  display_name: z.string(),
  created_at: z.string().datetime(),
  is_active: z.boolean(),
});

// Transform sang camelCase — output type là object mới
export const UserSchema = ApiUserSchema.transform((data) => ({
  id: data.id,
  displayName: data.display_name,
  createdAt: new Date(data.created_at), // Chuyển luôn thành Date object
  isActive: data.is_active,
}));

export type User = z.infer<typeof UserSchema>;
// User = { id: string; displayName: string; createdAt: Date; isActive: boolean }
```

---

## 4.6 Pattern Thực Tế — State với RxJS trong Service

Đây là pattern dùng BehaviorSubject để quản lý state loading/error/data trong service, trước khi học NgRx:

```typescript
// Định nghĩa state interface
interface UserListState {
  users: User[];
  isLoading: boolean;
  error: string | null;
  pagination: Pagination | null;
}

const initialState: UserListState = {
  users: [],
  isLoading: false,
  error: null,
  pagination: null,
};

@Injectable({ providedIn: 'root' })
export class UserListFacade {
  private readonly userService = inject(UserService);
  private readonly state = new BehaviorSubject<UserListState>(initialState);

  // Selectors — expose read-only slices of state
  readonly users$ = this.state.pipe(map((s) => s.users));
  readonly isLoading$ = this.state.pipe(map((s) => s.isLoading));
  readonly error$ = this.state.pipe(map((s) => s.error));
  readonly pagination$ = this.state.pipe(map((s) => s.pagination));

  readonly isEmpty$ = this.users$.pipe(
    map((users) => users.length === 0),
    distinctUntilChanged()
  );

  // Actions
  loadUsers(params?: UserQueryParams): void {
    this.patchState({ isLoading: true, error: null });

    this.userService
      .getUsers(params)
      .subscribe({
        next: (response) => {
          this.patchState({
            users: response.data,
            pagination: response.pagination,
            isLoading: false,
          });
        },
        error: (err) => {
          this.patchState({
            error: 'Không thể tải danh sách người dùng',
            isLoading: false,
          });
        },
      });
  }

  addUser(user: User): void {
    this.patchState({
      users: [...this.state.value.users, user],
    });
  }

  removeUser(userId: string): void {
    this.patchState({
      users: this.state.value.users.filter((u) => u.id !== userId),
    });
  }

  private patchState(partial: Partial<UserListState>): void {
    this.state.next({ ...this.state.value, ...partial });
  }
}
```

---

## Tổng kết chương

RxJS và HttpClient là bộ đôi không thể tách rời trong Angular. Những điểm cốt lõi cần nhớ:

1. **Observable là lazy**: Không có gì xảy ra cho đến khi subscribe. HttpClient tạo ra Cold Observable — mỗi subscribe là một HTTP request mới.

2. **Bốn flattening operators**: `switchMap` (cancel old), `mergeMap` (parallel), `concatMap` (sequential), `exhaustMap` (ignore new). Chọn đúng operator là chìa khóa để tránh race conditions.

3. **Interceptors** là nơi xử lý cross-cutting concerns: auth token, error handling, loading state. Luôn dùng functional interceptors trong Angular 15+.

4. **Zod validate runtime**: TypeScript chỉ an toàn lúc compile time. Zod bảo vệ ứng dụng khỏi unexpected API responses lúc runtime.

5. **Luôn unsubscribe**: Dùng `takeUntilDestroyed(this.destroyRef)` hoặc `AsyncPipe` trong template. Memory leak từ subscription chưa unsubscribe là lỗi phổ biến nhất trong Angular.

Chương tiếp theo sẽ đi vào **Routing** — cách Angular quản lý điều hướng, lazy loading, và bảo vệ routes với Guards.
