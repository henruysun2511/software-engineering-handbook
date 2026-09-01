# Chương 11: Authentication

## Giới thiệu chương

Authentication là một trong những phần phức tạp nhất của ứng dụng web — không phải vì logic đăng nhập khó, mà vì phải xử lý đúng đắn toàn bộ vòng đời của phiên làm việc: lưu token an toàn, tự động đính kèm vào request, làm mới khi hết hạn, bảo vệ routes, và xử lý đồng thời nhiều request trong lúc đang refresh. Làm sai bất kỳ bước nào cũng dẫn đến lỗ hổng bảo mật hoặc UX tệ.

Chương này xây dựng một hệ thống authentication hoàn chỉnh với JWT, refresh token, và tất cả các edge case quan trọng.

---

## 11.1 Kiến trúc Authentication

### Tổng quan JWT Flow

```
1. User POST /auth/login { email, password }
   ← Server trả về { accessToken, refreshToken, user }

2. Client lưu accessToken (memory) và refreshToken (httpOnly cookie hoặc localStorage)

3. Mọi request tiếp theo: Authorization: Bearer <accessToken>

4. Khi accessToken hết hạn (401):
   a. Client POST /auth/refresh { refreshToken }
   b. Server trả về { accessToken mới }
   c. Retry request gốc với token mới

5. Khi refreshToken hết hạn: logout, redirect về login
```

### Chiến lược lưu Token

| Vị trí | Ưu điểm | Nhược điểm |
|--------|---------|------------|
| **Memory** | An toàn nhất (không XSS) | Mất khi refresh page |
| **localStorage** | Persist qua refresh | Dễ bị XSS |
| **sessionStorage** | Persist trong tab | Mất khi đóng tab |
| **HttpOnly Cookie** | Không thể đọc bằng JS | Cần backend hỗ trợ, CSRF risk |

**Chiến lược thực tế khuyến khích:**
- `accessToken`: Lưu trong memory (service property) — ngắn hạn (15 phút), không persist
- `refreshToken`: `HttpOnly Cookie` (backend set) hoặc `localStorage` — dài hạn (7 ngày)

---

## 11.2 Auth Service hoàn chỉnh

```typescript
// core/models/auth.model.ts
export interface LoginDto {
  email: string;
  password: string;
}

export interface AuthResponse {
  accessToken: string;
  refreshToken: string;
  expiresIn: number; // seconds
  user: AuthUser;
}

export interface AuthUser {
  id: string;
  email: string;
  displayName: string;
  role: UserRole;
  permissions: string[];
}

export interface RefreshResponse {
  accessToken: string;
  expiresIn: number;
}
```

```typescript
// core/services/auth.service.ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly http = inject(HttpClient);
  private readonly router = inject(Router);
  private readonly apiUrl = inject(API_URL);

  // accessToken lưu trong memory — không persist, an toàn hơn localStorage
  private accessToken: string | null = null;
  private tokenExpiryTimer: ReturnType<typeof setTimeout> | null = null;

  // Signals cho reactive state
  private readonly currentUserSignal = signal<AuthUser | null>(null);

  // Public read-only API
  readonly currentUser = this.currentUserSignal.asReadonly();
  readonly isAuthenticated = computed(() => this.currentUserSignal() !== null);
  readonly userRole = computed(() => this.currentUserSignal()?.role ?? null);

  constructor() {
    // Khôi phục session từ refreshToken khi app khởi động
    this.restoreSession();
  }

  // ─── Public Methods ───────────────────────────────────────────────────────

  login(dto: LoginDto): Observable<AuthUser> {
    return this.http
      .post<AuthResponse>(`${this.apiUrl}/auth/login`, dto)
      .pipe(
        tap((response) => this.handleAuthResponse(response)),
        map((response) => response.user)
      );
  }

  logout(): void {
    // Gọi API để invalidate refreshToken trên server
    const refreshToken = this.getRefreshToken();
    if (refreshToken) {
      this.http
        .post(`${this.apiUrl}/auth/logout`, { refreshToken })
        .pipe(catchError(() => of(null)))
        .subscribe();
    }

    this.clearSession();
    this.router.navigate(['/auth/login']);
  }

  refreshToken(): Observable<string> {
    const refreshToken = this.getRefreshToken();
    if (!refreshToken) {
      return throwError(() => new Error('No refresh token'));
    }

    return this.http
      .post<RefreshResponse>(`${this.apiUrl}/auth/refresh`, { refreshToken })
      .pipe(
        tap((response) => {
          this.accessToken = response.accessToken;
          this.scheduleTokenRefresh(response.expiresIn);
        }),
        map((response) => response.accessToken),
        catchError((error) => {
          this.clearSession();
          this.router.navigate(['/auth/login']);
          return throwError(() => error);
        })
      );
  }

  getAccessToken(): string | null {
    return this.accessToken;
  }

  hasPermission(permission: string): boolean {
    return this.currentUserSignal()?.permissions.includes(permission) ?? false;
  }

  hasRole(role: UserRole | UserRole[]): boolean {
    const userRole = this.currentUserSignal()?.role;
    if (!userRole) return false;

    const roles = Array.isArray(role) ? role : [role];
    return roles.includes(userRole);
  }

  // ─── Private Methods ──────────────────────────────────────────────────────

  private handleAuthResponse(response: AuthResponse): void {
    this.accessToken = response.accessToken;
    this.currentUserSignal.set(response.user);
    this.saveRefreshToken(response.refreshToken);
    this.scheduleTokenRefresh(response.expiresIn);
  }

  private clearSession(): void {
    this.accessToken = null;
    this.currentUserSignal.set(null);
    this.removeRefreshToken();

    if (this.tokenExpiryTimer) {
      clearTimeout(this.tokenExpiryTimer);
      this.tokenExpiryTimer = null;
    }
  }

  // Tự động refresh trước khi token hết hạn 30 giây
  private scheduleTokenRefresh(expiresInSeconds: number): void {
    if (this.tokenExpiryTimer) {
      clearTimeout(this.tokenExpiryTimer);
    }

    const refreshAfterMs = (expiresInSeconds - 30) * 1000;
    if (refreshAfterMs <= 0) return;

    this.tokenExpiryTimer = setTimeout(() => {
      this.refreshToken().subscribe();
    }, refreshAfterMs);
  }

  private restoreSession(): void {
    const refreshToken = this.getRefreshToken();
    if (!refreshToken) return;

    // Có refreshToken → thử lấy accessToken mới
    this.refreshToken().subscribe({
      error: () => {
        // RefreshToken hết hạn → clear session im lặng
        this.clearSession();
      },
    });
  }

  // ─── Token Storage ────────────────────────────────────────────────────────
  // Đóng gói storage logic — dễ thay đổi sau này

  private readonly REFRESH_TOKEN_KEY = 'refresh_token';

  private saveRefreshToken(token: string): void {
    localStorage.setItem(this.REFRESH_TOKEN_KEY, token);
  }

  private getRefreshToken(): string | null {
    return localStorage.getItem(this.REFRESH_TOKEN_KEY);
  }

  private removeRefreshToken(): void {
    localStorage.removeItem(this.REFRESH_TOKEN_KEY);
  }
}
```

---

## 11.3 HTTP Interceptors

### Auth Interceptor — Gắn Token

```typescript
// core/interceptors/auth.interceptor.ts
import {
  HttpInterceptorFn,
  HttpRequest,
  HttpHandlerFn,
} from '@angular/common/http';

// URL không cần token
const PUBLIC_ENDPOINTS = [
  '/auth/login',
  '/auth/register',
  '/auth/refresh',
  '/auth/forgot-password',
];

function isPublicEndpoint(url: string): boolean {
  return PUBLIC_ENDPOINTS.some((endpoint) => url.includes(endpoint));
}

export const authInterceptor: HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
) => {
  const authService = inject(AuthService);
  const token = authService.getAccessToken();

  if (!token || isPublicEndpoint(req.url)) {
    return next(req);
  }

  const authReq = req.clone({
    setHeaders: { Authorization: `Bearer ${token}` },
  });

  return next(authReq);
};
```

### Token Refresh Interceptor — Xử lý 401

Đây là phần phức tạp nhất: khi nhận 401, cần refresh token và retry request gốc. Nếu có nhiều request đồng thời đều nhận 401, chỉ nên gọi refresh một lần và share kết quả cho tất cả:

```typescript
// core/interceptors/token-refresh.interceptor.ts
import {
  HttpInterceptorFn,
  HttpRequest,
  HttpHandlerFn,
  HttpErrorResponse,
} from '@angular/common/http';
import { Observable, throwError, BehaviorSubject } from 'rxjs';
import {
  catchError,
  filter,
  switchMap,
  take,
  finalize,
} from 'rxjs/operators';

// Shared state để handle concurrent 401s
// Dùng module-level variables vì interceptor function là stateless
let isRefreshing = false;
const refreshTokenSubject = new BehaviorSubject<string | null>(null);

export const tokenRefreshInterceptor: HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
) => {
  const authService = inject(AuthService);

  return next(req).pipe(
    catchError((error: unknown) => {
      if (
        !(error instanceof HttpErrorResponse) ||
        error.status !== 401 ||
        isPublicEndpoint(req.url) ||
        !authService.getAccessToken()
      ) {
        return throwError(() => error);
      }

      return handle401Error(req, next, authService);
    })
  );
};

function handle401Error(
  req: HttpRequest<unknown>,
  next: HttpHandlerFn,
  authService: AuthService
): Observable<unknown> {
  if (isRefreshing) {
    // Refresh đang chạy — đợi token mới rồi retry
    return refreshTokenSubject.pipe(
      filter((token): token is string => token !== null),
      take(1),
      switchMap((token) => next(addToken(req, token)))
    );
  }

  // Bắt đầu refresh
  isRefreshing = true;
  refreshTokenSubject.next(null); // Signal cho các request đang đợi

  return authService.refreshToken().pipe(
    switchMap((newToken) => {
      refreshTokenSubject.next(newToken); // Phát token mới cho tất cả
      return next(addToken(req, newToken));
    }),
    catchError((refreshError) => {
      // Refresh thất bại — logout
      refreshTokenSubject.next(null);
      authService.logout();
      return throwError(() => refreshError);
    }),
    finalize(() => {
      isRefreshing = false;
    })
  );
}

function addToken(
  req: HttpRequest<unknown>,
  token: string
): HttpRequest<unknown> {
  return req.clone({
    setHeaders: { Authorization: `Bearer ${token}` },
  });
}

function isPublicEndpoint(url: string): boolean {
  const PUBLIC_ENDPOINTS = ['/auth/login', '/auth/refresh', '/auth/register'];
  return PUBLIC_ENDPOINTS.some((ep) => url.includes(ep));
}
```

### Error Interceptor — Xử lý HTTP Errors Toàn cục

```typescript
// core/interceptors/error.interceptor.ts
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const notificationService = inject(NotificationService);
  const router = inject(Router);

  return next(req).pipe(
    catchError((error: unknown) => {
      if (!(error instanceof HttpErrorResponse)) {
        return throwError(() => error);
      }

      // 401 được xử lý bởi tokenRefreshInterceptor
      if (error.status === 401) {
        return throwError(() => error);
      }

      switch (error.status) {
        case 403:
          notificationService.error(
            'Bạn không có quyền thực hiện thao tác này'
          );
          router.navigate(['/403']);
          break;

        case 404:
          // Để service tự xử lý 404 — không show global notification
          break;

        case 409:
          // Conflict — thường có message từ server
          notificationService.error(
            error.error?.message ?? 'Dữ liệu đã tồn tại'
          );
          break;

        case 422: {
          // Validation error
          const errors = error.error?.errors as Record<string, string[]> | undefined;
          if (errors) {
            const message = Object.values(errors).flat().slice(0, 3).join('. ');
            notificationService.error(message);
          } else {
            notificationService.error(
              error.error?.message ?? 'Dữ liệu không hợp lệ'
            );
          }
          break;
        }

        case 429:
          notificationService.error(
            'Quá nhiều yêu cầu. Vui lòng thử lại sau ít phút.'
          );
          break;

        case 500:
        case 502:
        case 503:
        case 504:
          notificationService.error(
            'Lỗi máy chủ. Đội kỹ thuật đã được thông báo.'
          );
          break;

        default:
          if (error.status === 0) {
            notificationService.error(
              'Không thể kết nối đến máy chủ. Vui lòng kiểm tra kết nối mạng.'
            );
          }
      }

      return throwError(() => error);
    })
  );
};
```

### Đăng ký Interceptors theo đúng thứ tự

```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(
      withFetch(),
      withInterceptors([
        // Thứ tự quan trọng: auth trước, refresh sau, error cuối
        authInterceptor,        // 1. Gắn token vào request
        tokenRefreshInterceptor, // 2. Xử lý 401 và refresh
        errorInterceptor,       // 3. Xử lý các lỗi HTTP khác
      ])
    ),
    provideAnimationsAsync(),
  ],
};
```

---

## 11.4 Route Guards

### Auth Guard

```typescript
// core/guards/auth.guard.ts
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  // Redirect về login, giữ returnUrl để redirect sau khi login
  return router.createUrlTree(['/auth/login'], {
    queryParams: { returnUrl: state.url },
  });
};
```

### Role Guard — Kiểm tra quyền theo route

```typescript
// core/guards/role.guard.ts
export const roleGuard = (
  requiredRoles: UserRole | UserRole[]
): CanActivateFn => {
  return () => {
    const authService = inject(AuthService);
    const router = inject(Router);

    if (!authService.isAuthenticated()) {
      return router.createUrlTree(['/auth/login']);
    }

    if (authService.hasRole(requiredRoles)) {
      return true;
    }

    // Đã login nhưng không đủ quyền
    return router.createUrlTree(['/403']);
  };
};
```

### Permission Guard — Kiểm tra permission cụ thể

```typescript
// core/guards/permission.guard.ts
export const permissionGuard = (
  requiredPermission: string
): CanActivateFn => {
  return () => {
    const authService = inject(AuthService);
    const router = inject(Router);

    if (!authService.isAuthenticated()) {
      return router.createUrlTree(['/auth/login']);
    }

    if (authService.hasPermission(requiredPermission)) {
      return true;
    }

    return router.createUrlTree(['/403']);
  };
};
```

```typescript
// Sử dụng trong routes
export const routes: Routes = [
  {
    path: 'auth',
    loadChildren: () =>
      import('./features/auth/auth.routes').then((m) => m.AUTH_ROUTES),
  },
  {
    path: 'dashboard',
    canActivate: [authGuard],
    loadComponent: () =>
      import('./features/dashboard/dashboard.component').then(
        (m) => m.DashboardComponent
      ),
  },
  {
    path: 'users',
    canActivate: [authGuard, roleGuard(['ADMIN', 'EDITOR'])],
    loadChildren: () =>
      import('./features/user/user.routes').then((m) => m.USER_ROUTES),
  },
  {
    path: 'admin',
    canActivate: [authGuard, roleGuard('ADMIN')],
    loadChildren: () =>
      import('./features/admin/admin.routes').then((m) => m.ADMIN_ROUTES),
  },
  {
    path: '403',
    loadComponent: () =>
      import('./shared/pages/forbidden/forbidden.component').then(
        (m) => m.ForbiddenComponent
      ),
  },
];
```

---

## 11.5 Login Component hoàn chỉnh

```typescript
// features/auth/login/login.component.ts
@Component({
  selector: 'app-login',
  standalone: true,
  imports: [
    ReactiveFormsModule,
    MatFormFieldModule,
    MatInputModule,
    MatButtonModule,
    MatIconModule,
    MatCheckboxModule,
    MatProgressSpinnerModule,
    RouterLink,
  ],
  templateUrl: './login.component.html',
  styleUrl: './login.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class LoginComponent {
  private readonly fb = inject(FormBuilder);
  private readonly authService = inject(AuthService);
  private readonly router = inject(Router);
  private readonly route = inject(ActivatedRoute);

  protected readonly isLoading = signal(false);
  protected readonly serverError = signal<string | null>(null);
  protected readonly isPasswordVisible = signal(false);

  protected readonly form = this.fb.group({
    email: this.fb.control('', {
      validators: [Validators.required, Validators.email],
      nonNullable: true,
    }),
    password: this.fb.control('', {
      validators: [Validators.required],
      nonNullable: true,
    }),
    rememberMe: this.fb.control(false, { nonNullable: true }),
  });

  // Lấy returnUrl từ query params (set bởi authGuard)
  private readonly returnUrl = this.route.snapshot.queryParams['returnUrl'] as string | undefined;

  protected onSubmit(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }

    this.isLoading.set(true);
    this.serverError.set(null);

    const { email, password } = this.form.getRawValue();

    this.authService.login({ email, password }).subscribe({
      next: () => {
        const redirectUrl = this.getSafeReturnUrl();
        this.router.navigateByUrl(redirectUrl);
      },
      error: (err: HttpErrorResponse) => {
        this.isLoading.set(false);

        if (err.status === 401) {
          this.serverError.set('Email hoặc mật khẩu không đúng');
        } else if (err.status === 423) {
          this.serverError.set('Tài khoản đã bị khóa. Vui lòng liên hệ quản trị viên.');
        } else {
          this.serverError.set('Đã xảy ra lỗi. Vui lòng thử lại.');
        }

        // Focus vào email field để user điền lại
        this.form.controls.password.reset();
      },
    });
  }

  protected togglePasswordVisibility(): void {
    this.isPasswordVisible.update((v) => !v);
  }

  // Chỉ chấp nhận relative URLs để tránh open redirect
  private getSafeReturnUrl(): string {
    const url = this.returnUrl;
    if (!url || !url.startsWith('/') || url.startsWith('//')) {
      return '/dashboard';
    }
    return url;
  }
}
```

```html
<!-- login.component.html -->
<div class="login-page">
  <div class="login-card">
    <div class="login-header">
      <img src="/assets/logo.svg" alt="Logo" class="logo" />
      <h1>Đăng nhập</h1>
      <p>Chào mừng bạn trở lại</p>
    </div>

    <form [formGroup]="form" (ngSubmit)="onSubmit()" class="login-form">
      <!-- Email -->
      <mat-form-field>
        <mat-label>Email</mat-label>
        <input
          matInput
          type="email"
          formControlName="email"
          autocomplete="email"
          placeholder="you@example.com"
        />
        <mat-icon matPrefix>mail_outline</mat-icon>

        @if (form.controls.email.hasError('required') && form.controls.email.touched) {
          <mat-error>Email là bắt buộc</mat-error>
        }
        @if (form.controls.email.hasError('email') && form.controls.email.touched) {
          <mat-error>Email không đúng định dạng</mat-error>
        }
      </mat-form-field>

      <!-- Password -->
      <mat-form-field>
        <mat-label>Mật khẩu</mat-label>
        <input
          matInput
          [type]="isPasswordVisible() ? 'text' : 'password'"
          formControlName="password"
          autocomplete="current-password"
        />
        <mat-icon matPrefix>lock_outline</mat-icon>
        <button
          matSuffix
          mat-icon-button
          type="button"
          (click)="togglePasswordVisibility()"
          [attr.aria-label]="isPasswordVisible() ? 'Ẩn mật khẩu' : 'Hiện mật khẩu'"
        >
          <mat-icon>{{ isPasswordVisible() ? 'visibility_off' : 'visibility' }}</mat-icon>
        </button>

        @if (form.controls.password.hasError('required') && form.controls.password.touched) {
          <mat-error>Mật khẩu là bắt buộc</mat-error>
        }
      </mat-form-field>

      <!-- Options row -->
      <div class="form-options">
        <mat-checkbox formControlName="rememberMe">Ghi nhớ đăng nhập</mat-checkbox>
        <a routerLink="/auth/forgot-password" class="forgot-link">
          Quên mật khẩu?
        </a>
      </div>

      <!-- Server error -->
      @if (serverError()) {
        <div class="server-error" role="alert">
          <mat-icon>error_outline</mat-icon>
          <span>{{ serverError() }}</span>
        </div>
      }

      <!-- Submit -->
      <button
        mat-flat-button
        color="primary"
        type="submit"
        class="submit-button"
        [disabled]="isLoading()"
      >
        @if (isLoading()) {
          <mat-spinner diameter="20" />
          <span>Đang đăng nhập...</span>
        } @else {
          <span>Đăng nhập</span>
        }
      </button>
    </form>
  </div>
</div>
```

---

## 11.6 Auto Logout khi Token hết hạn

```typescript
// core/services/session.service.ts
@Injectable({ providedIn: 'root' })
export class SessionService {
  private readonly authService = inject(AuthService);
  private readonly notificationService = inject(NotificationService);

  private inactivityTimer: ReturnType<typeof setTimeout> | null = null;
  private readonly INACTIVITY_TIMEOUT = 30 * 60 * 1000; // 30 phút

  startInactivityTimer(): void {
    this.resetTimer();

    // Lắng nghe các sự kiện hoạt động của user
    const events = ['mousedown', 'keypress', 'scroll', 'touchstart'];

    merge(
      ...events.map((event) => fromEvent(document, event))
    )
      .pipe(
        debounceTime(1000),
        takeUntil(
          toObservable(this.authService.isAuthenticated).pipe(
            filter((isAuth) => !isAuth)
          )
        )
      )
      .subscribe(() => this.resetTimer());
  }

  private resetTimer(): void {
    if (this.inactivityTimer) {
      clearTimeout(this.inactivityTimer);
    }

    this.inactivityTimer = setTimeout(() => {
      this.notificationService.warning(
        'Phiên làm việc đã hết hạn do không hoạt động'
      );
      this.authService.logout();
    }, this.INACTIVITY_TIMEOUT);
  }
}
```

---

## 11.7 Permission-based UI

Ngoài route guards, cần ẩn/hiện UI elements dựa trên quyền:

```typescript
// shared/directives/has-permission.directive.ts
@Directive({
  selector: '[appHasPermission]',
  standalone: true,
})
export class HasPermissionDirective implements OnInit {
  private readonly template = inject(TemplateRef<unknown>);
  private readonly viewContainer = inject(ViewContainerRef);
  private readonly authService = inject(AuthService);

  readonly appHasPermission = input<string | string[]>('');

  ngOnInit(): void {
    effect(() => {
      const required = this.appHasPermission();
      const permissions = Array.isArray(required) ? required : [required];
      const hasAccess = permissions.some((p) =>
        this.authService.hasPermission(p)
      );

      this.viewContainer.clear();
      if (hasAccess) {
        this.viewContainer.createEmbeddedView(this.template);
      }
    });
  }
}
```

```typescript
// shared/directives/has-role.directive.ts
@Directive({
  selector: '[appHasRole]',
  standalone: true,
})
export class HasRoleDirective implements OnInit {
  private readonly template = inject(TemplateRef<unknown>);
  private readonly viewContainer = inject(ViewContainerRef);
  private readonly authService = inject(AuthService);

  readonly appHasRole = input<UserRole | UserRole[]>([]);

  ngOnInit(): void {
    effect(() => {
      const hasAccess = this.authService.hasRole(this.appHasRole());
      this.viewContainer.clear();
      if (hasAccess) {
        this.viewContainer.createEmbeddedView(this.template);
      }
    });
  }
}
```

```html
<!-- Sử dụng trong template -->
<mat-toolbar>
  <!-- Hiện cho mọi user đã login -->
  <span>{{ authService.currentUser()?.displayName }}</span>

  <!-- Chỉ hiện cho ADMIN -->
  <button *appHasRole="'ADMIN'" mat-button routerLink="/admin">
    Quản trị
  </button>

  <!-- Chỉ hiện khi có permission cụ thể -->
  <button *appHasPermission="'users:delete'" mat-icon-button color="warn">
    <mat-icon>delete</mat-icon>
  </button>

  <!-- Nhiều roles hoặc permissions -->
  <div *appHasRole="['ADMIN', 'EDITOR']">
    <app-content-tools />
  </div>
</mat-toolbar>
```

---

## 11.8 Forgot Password và Reset Password

```typescript
// features/auth/forgot-password/forgot-password.component.ts
@Component({
  selector: 'app-forgot-password',
  standalone: true,
  imports: [ReactiveFormsModule, MatFormFieldModule, MatInputModule, MatButtonModule],
  template: `
    <div class="auth-card">
      <h2>Quên mật khẩu</h2>
      <p>Nhập email để nhận link đặt lại mật khẩu</p>

      @if (!isSubmitted()) {
        <form [formGroup]="form" (ngSubmit)="onSubmit()">
          <mat-form-field>
            <mat-label>Email</mat-label>
            <input matInput type="email" formControlName="email" />
          </mat-form-field>

          <button mat-flat-button color="primary" type="submit" [disabled]="isLoading()">
            Gửi link đặt lại
          </button>
        </form>
      } @else {
        <div class="success-message">
          <mat-icon color="primary">mark_email_read</mat-icon>
          <p>
            Link đặt lại mật khẩu đã được gửi đến
            <strong>{{ submittedEmail() }}</strong>
          </p>
          <p class="hint">Vui lòng kiểm tra hộp thư, kể cả thư mục spam.</p>
          <a mat-button routerLink="/auth/login">Quay lại đăng nhập</a>
        </div>
      }
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ForgotPasswordComponent {
  private readonly fb = inject(FormBuilder);
  private readonly authService = inject(AuthService);

  protected readonly isLoading = signal(false);
  protected readonly isSubmitted = signal(false);
  protected readonly submittedEmail = signal('');

  protected readonly form = this.fb.group({
    email: this.fb.control('', {
      validators: [Validators.required, Validators.email],
      nonNullable: true,
    }),
  });

  protected onSubmit(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }

    this.isLoading.set(true);
    const { email } = this.form.getRawValue();

    this.authService.forgotPassword(email).subscribe({
      next: () => {
        this.submittedEmail.set(email);
        this.isSubmitted.set(true);
        this.isLoading.set(false);
      },
      error: () => {
        // Luôn hiện success message dù email có tồn tại hay không
        // để không leak thông tin account
        this.submittedEmail.set(email);
        this.isSubmitted.set(true);
        this.isLoading.set(false);
      },
    });
  }
}
```

---

## Tổng kết chương

Authentication là phần đòi hỏi sự chú ý đến từng chi tiết nhỏ. Những điểm cốt lõi:

1. **JWT flow đầy đủ**: `accessToken` ngắn hạn trong memory, `refreshToken` dài hạn trong localStorage. `scheduleTokenRefresh` tự động làm mới trước khi hết hạn — UX mượt mà.

2. **Token Refresh Interceptor** xử lý concurrent 401s bằng `BehaviorSubject` — chỉ gọi refresh một lần dù có bao nhiêu request đồng thời, tất cả đều được retry với token mới.

3. **Thứ tự interceptors** quan trọng: auth → token-refresh → error. Sai thứ tự sẽ dẫn đến behavior không đoán trước được.

4. **Guards kết hợp**: `authGuard` kiểm tra authentication, `roleGuard` kiểm tra role, `permissionGuard` kiểm tra permission granular. Compose nhiều guards cho một route.

5. **Security best practices**: Validate returnUrl để tránh open redirect. Luôn show success message cho forgot password dù email có tồn tại hay không — tránh leak thông tin.

6. **UI directives** (`*appHasRole`, `*appHasPermission`) cho phép ẩn/hiện UI elements theo quyền — bổ sung cho route guards, không thay thế.

Chương tiếp theo và cuối cùng sẽ đi vào **SSR & Deployment** — Server-Side Rendering với `@angular/ssr`, tối ưu build, và các chiến lược deploy lên Vercel, Node.js server, và Docker.
