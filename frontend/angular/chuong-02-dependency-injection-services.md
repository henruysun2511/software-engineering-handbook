# Chương 2: Dependency Injection & Services

## Giới thiệu chương

Dependency Injection (DI) là hệ thống quan trọng nhất và khác biệt nhất của Angular so với React. Trong khi React giải quyết bài toán chia sẻ logic thông qua custom hooks và Context API, Angular xây dựng toàn bộ framework xung quanh một DI container — một hệ thống tự động quản lý vòng đời và phân phối các dependency giữa các thành phần trong ứng dụng.

Hiểu DI không chỉ là hiểu cú pháp `inject()` hay `@Injectable` — mà là hiểu cách Angular tổ chức và quản lý dependency tree, từ đó đưa ra quyết định kiến trúc đúng đắn cho từng trường hợp.

---

## 2.1 Dependency Injection — Trái Tim của Angular

### DI là gì và tại sao Angular dùng nó

**Dependency Injection** là một design pattern trong đó một object không tự tạo ra các dependency của mình — thay vào đó, dependency được "inject" vào từ bên ngoài. Angular implements pattern này thông qua một **DI container** (hay **Injector**) — một registry trung tâm biết cách tạo và cung cấp mọi dependency trong ứng dụng.

Thay vì viết:

```typescript
// Không dùng DI — component tự tạo dependency
export class UserListComponent {
  private userService = new UserService(new HttpClient(...));
  // Vấn đề: khó test, không tái sử dụng instance, phụ thuộc cứng
}
```

Angular cho phép:

```typescript
// Dùng DI — Angular tự inject dependency
export class UserListComponent {
  private readonly userService = inject(UserService);
  // UserService được tạo và quản lý bởi Angular, không phải component
}
```

Lợi ích của DI trong Angular:

- **Testability**: Dễ dàng thay thế dependency thật bằng mock trong unit test
- **Singleton management**: Angular đảm bảo service chỉ được tạo một lần (nếu cấu hình đúng)
- **Loose coupling**: Component không biết cách tạo service — chỉ biết interface cần dùng
- **Lifecycle management**: Angular tự hủy service khi không còn cần thiết

### Injector Hierarchy

Angular không có một DI container duy nhất — nó có một **cây injector** phản ánh cấu trúc ứng dụng. Khi một component yêu cầu một dependency, Angular tìm kiếm từ dưới lên trên cây injector:

```
Root Injector (providedIn: 'root')
├── Platform Injector (providedIn: 'platform')
├── Environment Injector (provideX() trong app.config.ts)
└── Component Injector (providers: [] trong @Component)
    └── Child Component Injector
        └── Grandchild Component Injector
```

Khi một component yêu cầu `UserService`:
1. Angular kiểm tra Component Injector của component đó
2. Nếu không tìm thấy, leo lên Parent Component Injector
3. Tiếp tục cho đến Environment/Root Injector
4. Nếu không tìm thấy ở đâu → Runtime Error

```typescript
// Minh họa injector hierarchy
@Component({
  selector: 'app-parent',
  providers: [
    // UserService được tạo ở cấp component này
    // Component con sẽ nhận instance NÀY, không phải root instance
    UserService,
  ],
  template: `<app-child />`,
})
export class ParentComponent {}

@Component({
  selector: 'app-child',
  template: `...`,
})
export class ChildComponent {
  // Nhận UserService từ ParentComponent injector — không phải root
  private readonly userService = inject(UserService);
}
```

### `inject()` Function — Cách hiện đại

Angular 14 giới thiệu `inject()` function như một thay thế cho constructor injection. Angular 18 khuyến khích dùng `inject()` vì nó ngắn gọn hơn và hoạt động tốt hơn với functional patterns.

```typescript
// Constructor injection — cách cũ, vẫn hợp lệ
@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(
    private readonly http: HttpClient,
    private readonly router: Router,
    @Inject(API_URL) private readonly apiUrl: string,
  ) {}
}

// inject() function — cách hiện đại, khuyến khích
@Injectable({ providedIn: 'root' })
export class UserService {
  private readonly http = inject(HttpClient);
  private readonly router = inject(Router);
  private readonly apiUrl = inject(API_URL);
}
```

`inject()` chỉ có thể gọi trong **injection context**:
- Trong class field initializer
- Trong constructor body
- Trong factory function của `providers`

```typescript
// ĐÚNG — field initializer là injection context
export class MyComponent {
  private readonly service = inject(MyService); // ✓
}

// ĐÚNG — constructor body là injection context
export class MyComponent {
  private readonly service: MyService;
  constructor() {
    this.service = inject(MyService); // ✓
  }
}

// SAI — ngoài injection context
export class MyComponent {
  ngOnInit() {
    const service = inject(MyService); // ✗ Runtime Error
  }
}
```

---

## 2.2 Services — Tổ chức Business Logic

### Service là gì

Trong Angular, **Service** là bất kỳ class nào được đánh dấu `@Injectable` và chứa logic không thuộc về một component cụ thể. Services thường đảm nhiệm:

- **Data fetching**: Gọi API và xử lý response
- **State management**: Lưu trữ và chia sẻ state giữa các component
- **Business logic**: Các phép tính, transformation, validation
- **Utility functions**: Formatting, logging, error handling

### Pattern: Thin Component, Fat Service

Nguyên tắc quan trọng nhất khi viết Angular: **component chỉ nên xử lý những gì liên quan đến template** — binding data, handle user events, và delegate logic xuống service.

```typescript
// ❌ SAI — component quá nhiều logic
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [CommonModule, MatTableModule],
  templateUrl: './user-list.component.html',
})
export class UserListComponent implements OnInit {
  private readonly http = inject(HttpClient);
  users = signal<User[]>([]);

  ngOnInit() {
    // Logic HTTP, transformation, error handling — tất cả trong component
    this.http.get<ApiResponse<User[]>>('/api/users').pipe(
      map(response => response.data.filter(u => u.isActive)),
      catchError(err => {
        console.error(err);
        return of([]);
      })
    ).subscribe(users => this.users.set(users));
  }
}
```

```typescript
// ✓ ĐÚNG — tách logic vào service
@Injectable({ providedIn: 'root' })
export class UserService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = `${inject(API_URL)}/users`;

  getActiveUsers(): Observable<User[]> {
    return this.http.get<ApiResponse<User[]>>(this.baseUrl).pipe(
      map((response) => response.data.filter((u) => u.isActive)),
      catchError(this.handleError<User[]>('getActiveUsers', []))
    );
  }

  getUserById(id: string): Observable<User> {
    return this.http.get<User>(`${this.baseUrl}/${id}`).pipe(
      catchError(this.handleError<never>('getUserById'))
    );
  }

  createUser(dto: CreateUserDto): Observable<User> {
    return this.http.post<User>(this.baseUrl, dto).pipe(
      catchError(this.handleError<never>('createUser'))
    );
  }

  private handleError<T>(operation: string, fallback?: T) {
    return (error: HttpErrorResponse): Observable<T> => {
      console.error(`${operation} failed:`, error);
      if (fallback !== undefined) return of(fallback as T);
      return throwError(() => error);
    };
  }
}
```

```typescript
// ✓ Component gọn, chỉ xử lý UI
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [CommonModule, MatTableModule, MatProgressSpinnerModule],
  templateUrl: './user-list.component.html',
})
export class UserListComponent implements OnInit {
  private readonly userService = inject(UserService);
  private readonly destroyRef = inject(DestroyRef);

  users = signal<User[]>([]);
  isLoading = signal(false);
  error = signal<string | null>(null);

  ngOnInit(): void {
    this.isLoading.set(true);

    this.userService
      .getActiveUsers()
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        finalize(() => this.isLoading.set(false))
      )
      .subscribe({
        next: (users) => this.users.set(users),
        error: (err) => this.error.set('Không thể tải danh sách người dùng.'),
      });
  }
}
```

### Service Communication giữa các Component

Khi hai component không có quan hệ cha-con cần chia sẻ state, dùng service với `BehaviorSubject` hoặc Signals:

```typescript
// notification.service.ts
export interface Notification {
  id: string;
  type: 'success' | 'error' | 'info' | 'warning';
  message: string;
  duration?: number;
}

@Injectable({ providedIn: 'root' })
export class NotificationService {
  // BehaviorSubject — giữ giá trị hiện tại và phát cho subscriber mới
  private readonly notificationsSubject = new BehaviorSubject<Notification[]>([]);

  // Expose Observable read-only — component không thể push trực tiếp
  readonly notifications$ = this.notificationsSubject.asObservable();

  show(notification: Omit<Notification, 'id'>): void {
    const newNotification: Notification = {
      ...notification,
      id: crypto.randomUUID(),
    };

    this.notificationsSubject.update(notifications => [...notifications, newNotification]);

    // Tự động xóa sau duration
    const duration = notification.duration ?? 5000;
    setTimeout(() => this.dismiss(newNotification.id), duration);
  }

  dismiss(id: string): void {
    this.notificationsSubject.update(notifications =>
      notifications.filter((n) => n.id !== id)
    );
  }

  // Convenience methods
  success(message: string): void {
    this.show({ type: 'success', message });
  }

  error(message: string, duration = 8000): void {
    this.show({ type: 'error', message, duration });
  }
}
```

```typescript
// Bất kỳ component nào cũng có thể dùng
@Component({ ... })
export class UserFormComponent {
  private readonly notificationService = inject(NotificationService);
  private readonly userService = inject(UserService);

  onSubmit(dto: CreateUserDto): void {
    this.userService.createUser(dto).subscribe({
      next: () => this.notificationService.success('Tạo người dùng thành công'),
      error: () => this.notificationService.error('Tạo người dùng thất bại'),
    });
  }
}
```

---

## 2.3 Provider Scopes và Cấu hình

### `providedIn: 'root'` — Singleton toàn app

Đây là cách khai báo phổ biến nhất. Service được tạo một lần và dùng chung cho toàn bộ ứng dụng. Angular cũng tree-shake service này nếu không có component nào inject.

```typescript
@Injectable({
  providedIn: 'root',
})
export class AuthService {
  private readonly http = inject(HttpClient);
  private readonly router = inject(Router);

  private readonly currentUserSignal = signal<AuthUser | null>(null);
  readonly currentUser = this.currentUserSignal.asReadonly();
  readonly isAuthenticated = computed(() => this.currentUserSignal() !== null);

  login(credentials: LoginDto): Observable<void> {
    return this.http.post<AuthResponse>('/api/auth/login', credentials).pipe(
      tap((response) => {
        this.currentUserSignal.set(response.user);
        localStorage.setItem('access_token', response.accessToken);
      }),
      map(() => void 0)
    );
  }

  logout(): void {
    this.currentUserSignal.set(null);
    localStorage.removeItem('access_token');
    this.router.navigate(['/auth/login']);
  }
}
```

### Component-level Provider — Instance riêng biệt

Đôi khi cần mỗi component có instance service riêng — ví dụ một form wizard cần state riêng biệt:

```typescript
@Injectable()  // Không có providedIn — chỉ tạo khi được request
export class WizardStateService {
  private readonly steps = signal<WizardStep[]>([]);
  private readonly currentStepIndex = signal(0);

  readonly currentStep = computed(() => this.steps()[this.currentStepIndex()]);
  readonly isFirstStep = computed(() => this.currentStepIndex() === 0);
  readonly isLastStep = computed(
    () => this.currentStepIndex() === this.steps().length - 1
  );

  initialize(steps: WizardStep[]): void {
    this.steps.set(steps);
    this.currentStepIndex.set(0);
  }

  next(): void {
    if (!this.isLastStep()) {
      this.currentStepIndex.update((i) => i + 1);
    }
  }

  previous(): void {
    if (!this.isFirstStep()) {
      this.currentStepIndex.update((i) => i - 1);
    }
  }
}

@Component({
  selector: 'app-registration-wizard',
  standalone: true,
  providers: [WizardStateService],  // Tạo instance mới cho component này
  template: `...`,
})
export class RegistrationWizardComponent {
  // Instance này chỉ sống trong RegistrationWizardComponent và các component con
  protected readonly wizardState = inject(WizardStateService);
}
```

### Các cách cấu hình Provider

Angular có nhiều cách để cung cấp dependency — không chỉ class:

```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    // 1. useClass — cung cấp một class (mặc định)
    { provide: LoggerService, useClass: LoggerService },

    // 2. useClass với alias — thay thế implementation (hữu ích cho testing)
    { provide: LoggerService, useClass: ConsoleLoggerService },

    // 3. useValue — cung cấp một giá trị tĩnh
    { provide: APP_NAME, useValue: 'My Angular App' },
    {
      provide: APP_CONFIG,
      useValue: {
        apiUrl: 'https://api.example.com',
        timeout: 30000,
      } satisfies AppConfig,
    },

    // 4. useFactory — tạo dependency theo điều kiện
    {
      provide: LoggerService,
      useFactory: () => {
        const env = inject(ENVIRONMENT);
        return env.production
          ? new NoOpLoggerService()
          : new ConsoleLoggerService();
      },
    },

    // 5. useExisting — alias cho provider đã tồn tại
    { provide: AbstractUserService, useExisting: UserService },
  ],
};
```

### InjectionToken — DI cho Non-class Values

Khi muốn inject string, number, object, hoặc function — dùng `InjectionToken`:

```typescript
// tokens.ts — định nghĩa tập trung
import { InjectionToken } from '@angular/core';
import { AppConfig } from './app-config.model';

// Generic type tham số đảm bảo type safety khi inject
export const APP_CONFIG = new InjectionToken<AppConfig>('APP_CONFIG');
export const API_URL = new InjectionToken<string>('API_URL');
export const MAX_RETRIES = new InjectionToken<number>('MAX_RETRIES', {
  // Có thể cung cấp giá trị mặc định trực tiếp trong token
  providedIn: 'root',
  factory: () => 3,
});
```

```typescript
// app.config.ts — cung cấp giá trị
export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: APP_CONFIG,
      useValue: {
        apiUrl: environment.apiUrl,
        timeout: environment.timeout,
      } satisfies AppConfig,
    },
    {
      provide: API_URL,
      useFactory: () => inject(APP_CONFIG).apiUrl,
    },
  ],
};
```

```typescript
// Sử dụng trong service
@Injectable({ providedIn: 'root' })
export class ApiService {
  private readonly apiUrl = inject(API_URL);
  private readonly maxRetries = inject(MAX_RETRIES);
  private readonly http = inject(HttpClient);

  get<T>(endpoint: string): Observable<T> {
    return this.http.get<T>(`${this.apiUrl}${endpoint}`).pipe(
      retry({ count: this.maxRetries, delay: 1000 })
    );
  }
}
```

---

## 2.4 DI Nâng Cao

### APP_INITIALIZER — Chạy code trước khi app khởi động

`APP_INITIALIZER` cho phép chạy một hoặc nhiều async operation trước khi Angular render bất kỳ component nào. Dùng để: load configuration từ server, khởi tạo auth state, load feature flags.

```typescript
// config.service.ts
@Injectable({ providedIn: 'root' })
export class ConfigService {
  private config = signal<RemoteConfig | null>(null);
  private readonly http = inject(HttpClient);

  readonly featureFlags = computed(() => this.config()?.featureFlags ?? {});

  load(): Observable<void> {
    return this.http.get<RemoteConfig>('/api/config').pipe(
      tap((config) => this.config.set(config)),
      map(() => void 0),
      catchError(() => {
        // Nếu load config thất bại, dùng default config
        this.config.set(DEFAULT_CONFIG);
        return of(void 0);
      })
    );
  }
}

// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(),

    // APP_INITIALIZER chạy trước khi app render
    {
      provide: APP_INITIALIZER,
      useFactory: () => {
        const configService = inject(ConfigService);
        // Trả về function trả về Observable hoặc Promise
        return () => configService.load();
      },
      multi: true,  // Cho phép nhiều initializer cùng lúc
    },
  ],
};
```

### Multi Providers — Nhiều Implementation cho Một Token

`multi: true` cho phép nhiều giá trị được cung cấp cho cùng một token. Angular sẽ inject một array chứa tất cả giá trị:

```typescript
// validator.token.ts
export interface GlobalValidator {
  validate(value: unknown): string | null;
}

export const GLOBAL_VALIDATORS = new InjectionToken<GlobalValidator[]>(
  'GLOBAL_VALIDATORS'
);

// validators/email.validator.ts
@Injectable()
export class EmailValidator implements GlobalValidator {
  validate(value: unknown): string | null {
    if (typeof value !== 'string') return null;
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(value) ? null : 'Email không hợp lệ';
  }
}

// validators/profanity.validator.ts
@Injectable()
export class ProfanityValidator implements GlobalValidator {
  validate(value: unknown): string | null {
    // implementation...
    return null;
  }
}

// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    { provide: GLOBAL_VALIDATORS, useClass: EmailValidator, multi: true },
    { provide: GLOBAL_VALIDATORS, useClass: ProfanityValidator, multi: true },
  ],
};

// Sử dụng
@Injectable({ providedIn: 'root' })
export class ValidationService {
  // Nhận array của tất cả validators
  private readonly validators = inject(GLOBAL_VALIDATORS);

  validateAll(value: unknown): string[] {
    return this.validators
      .map((v) => v.validate(value))
      .filter((result): result is string => result !== null);
  }
}
```

### forwardRef — Xử lý Circular Reference

Đôi khi hai class cần tham chiếu lẫn nhau, gây ra vấn đề circular dependency trong JavaScript (class chưa được define khi cần dùng). `forwardRef` giải quyết bằng cách wrap reference trong một function được gọi lazily:

```typescript
// ControlValueAccessor — custom form control cần tham chiếu chính nó
@Component({
  selector: 'app-star-rating',
  standalone: true,
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      // forwardRef vì StarRatingComponent chưa được define hoàn toàn
      useExisting: forwardRef(() => StarRatingComponent),
      multi: true,
    },
  ],
  template: `
    @for (star of stars; track star) {
      <button
        type="button"
        [class.active]="star <= value()"
        (click)="setValue(star)"
        (blur)="onTouched()"
      >
        ★
      </button>
    }
  `,
})
export class StarRatingComponent implements ControlValueAccessor {
  readonly stars = [1, 2, 3, 4, 5];
  value = signal(0);

  private onChange: (value: number) => void = () => {};
  readonly onTouched: () => void = () => {};

  setValue(star: number): void {
    this.value.set(star);
    this.onChange(star);
  }

  // ControlValueAccessor interface
  writeValue(value: number): void {
    this.value.set(value ?? 0);
  }

  registerOnChange(fn: (value: number) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    // Assign to property — TypeScript không cho assign trực tiếp vì readonly
    (this as { onTouched: () => void }).onTouched = fn;
  }
}
```

---

## 2.5 Tổ chức Services trong Dự án Thực Tế

### Core Services — Singleton toàn ứng dụng

```
src/app/core/services/
├── auth.service.ts           # Authentication, authorization
├── api.service.ts            # Base HTTP service với interceptors
├── storage.service.ts        # LocalStorage/SessionStorage abstraction
├── logger.service.ts         # Structured logging
└── notification.service.ts   # Toast notifications
```

### Domain Services — Logic nghiệp vụ theo feature

```
src/app/features/
├── user/
│   └── services/
│       ├── user.service.ts          # CRUD operations
│       └── user-permission.service.ts  # Permission checks
├── product/
│   └── services/
│       └── product.service.ts
```

### Ví dụ Service hoàn chỉnh — Production Ready

```typescript
// core/services/api.service.ts
export interface ApiResponse<T> {
  data: T;
  message?: string;
  pagination?: {
    page: number;
    limit: number;
    total: number;
  };
}

export interface RequestOptions {
  params?: Record<string, string | number | boolean>;
  headers?: Record<string, string>;
}

@Injectable({ providedIn: 'root' })
export class ApiService {
  private readonly http = inject(HttpClient);
  private readonly apiUrl = inject(API_URL);

  get<T>(endpoint: string, options?: RequestOptions): Observable<T> {
    return this.http
      .get<T>(`${this.apiUrl}${endpoint}`, this.buildOptions(options))
      .pipe(catchError(this.handleError));
  }

  post<T, B = unknown>(
    endpoint: string,
    body: B,
    options?: RequestOptions
  ): Observable<T> {
    return this.http
      .post<T>(`${this.apiUrl}${endpoint}`, body, this.buildOptions(options))
      .pipe(catchError(this.handleError));
  }

  put<T, B = unknown>(
    endpoint: string,
    body: B,
    options?: RequestOptions
  ): Observable<T> {
    return this.http
      .put<T>(`${this.apiUrl}${endpoint}`, body, this.buildOptions(options))
      .pipe(catchError(this.handleError));
  }

  patch<T, B = unknown>(
    endpoint: string,
    body: B,
    options?: RequestOptions
  ): Observable<T> {
    return this.http
      .patch<T>(`${this.apiUrl}${endpoint}`, body, this.buildOptions(options))
      .pipe(catchError(this.handleError));
  }

  delete<T>(endpoint: string, options?: RequestOptions): Observable<T> {
    return this.http
      .delete<T>(`${this.apiUrl}${endpoint}`, this.buildOptions(options))
      .pipe(catchError(this.handleError));
  }

  private buildOptions(options?: RequestOptions): {
    params?: HttpParams;
    headers?: HttpHeaders;
  } {
    const result: { params?: HttpParams; headers?: HttpHeaders } = {};

    if (options?.params) {
      let params = new HttpParams();
      Object.entries(options.params).forEach(([key, value]) => {
        params = params.set(key, String(value));
      });
      result.params = params;
    }

    if (options?.headers) {
      let headers = new HttpHeaders();
      Object.entries(options.headers).forEach(([key, value]) => {
        headers = headers.set(key, value);
      });
      result.headers = headers;
    }

    return result;
  }

  private handleError(error: HttpErrorResponse): Observable<never> {
    const message = error.error?.message ?? error.message ?? 'Đã xảy ra lỗi';
    return throwError(() => new Error(message));
  }
}
```

```typescript
// features/user/services/user.service.ts
export interface UserListParams {
  page?: number;
  limit?: number;
  search?: string;
  role?: UserRole;
  isActive?: boolean;
}

@Injectable({ providedIn: 'root' })
export class UserService {
  private readonly api = inject(ApiService);
  private readonly endpoint = '/users';

  getUsers(params?: UserListParams): Observable<ApiResponse<User[]>> {
    return this.api.get<ApiResponse<User[]>>(this.endpoint, {
      params: params as Record<string, string | number | boolean>,
    });
  }

  getUserById(id: string): Observable<User> {
    return this.api.get<User>(`${this.endpoint}/${id}`);
  }

  createUser(dto: CreateUserDto): Observable<User> {
    return this.api.post<User>(this.endpoint, dto);
  }

  updateUser(id: string, dto: UpdateUserDto): Observable<User> {
    return this.api.patch<User>(`${this.endpoint}/${id}`, dto);
  }

  deleteUser(id: string): Observable<void> {
    return this.api.delete<void>(`${this.endpoint}/${id}`);
  }

  // Domain-specific operations
  deactivateUser(id: string): Observable<User> {
    return this.api.patch<User>(`${this.endpoint}/${id}/deactivate`, {});
  }

  resetUserPassword(id: string): Observable<void> {
    return this.api.post<void>(`${this.endpoint}/${id}/reset-password`, {});
  }
}
```

---

## Tổng kết chương

Dependency Injection là nền tảng kiến trúc của Angular. Những điểm cốt lõi:

1. **DI container tự động quản lý** việc tạo, inject và hủy dependency — component chỉ khai báo cần gì, Angular lo phần còn lại.

2. **Injector hierarchy** cho phép kiểm soát scope của service — root singleton, environment-level, hay component-level instance riêng biệt.

3. **`inject()` function** là cách hiện đại, ngắn gọn hơn constructor injection và hoạt động tốt với Signals.

4. **Services là nơi chứa business logic** — nguyên tắc "thin component, fat service" giúp code dễ test và tái sử dụng.

5. **InjectionToken** cho phép inject bất kỳ giá trị nào (string, number, object, function) với type safety đầy đủ.

6. **Multi providers** và **APP_INITIALIZER** là các pattern nâng cao giải quyết các bài toán kiến trúc phức tạp.

Chương tiếp theo sẽ đi vào **Signals & Reactivity** — hệ thống reactivity mới của Angular 16-18, một thay đổi mang tính cách mạng trong cách Angular quản lý state và trigger re-render.
