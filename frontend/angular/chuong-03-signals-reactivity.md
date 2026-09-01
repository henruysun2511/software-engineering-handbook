# Chương 3: Signals & Reactivity

## Giới thiệu chương

Angular Signals là thay đổi lớn nhất của Angular kể từ khi framework ra đời. Được giới thiệu ở Angular 16 (developer preview) và chính thức stable ở Angular 17, Signals đại diện cho một mô hình reactivity mới — **fine-grained reactivity** — thay thế cho cơ chế change detection dựa trên Zone.js vốn đã tồn tại từ những ngày đầu của Angular.

Để hiểu tại sao Signals quan trọng, cần hiểu vấn đề mà nó giải quyết.

---

## 3.1 Vấn đề của Zone.js và Change Detection cũ

### Zone.js hoạt động như thế nào

Angular truyền thống sử dụng **Zone.js** để biết khi nào cần cập nhật giao diện. Zone.js "patch" hầu hết các async API của trình duyệt (setTimeout, Promise, addEventListener, XHR…) để khi bất kỳ async operation nào hoàn thành, Angular được thông báo và chạy **change detection** — duyệt qua toàn bộ component tree để kiểm tra xem có gì thay đổi không.

Cơ chế này hoạt động tốt cho ứng dụng nhỏ, nhưng có vấn đề ở quy mô lớn:

- **Expensive**: Mỗi lần change detection phải kiểm tra mọi component trong cây
- **Unpredictable**: Bất kỳ async event nào cũng trigger change detection, kể cả những event không liên quan đến UI
- **Black box**: Developer khó biết chính xác khi nào và tại sao component re-render

### OnPush — Giải pháp tạm thời

`ChangeDetectionStrategy.OnPush` là cách giảm thiểu vấn đề: Angular chỉ kiểm tra component khi:
1. `@Input` reference thay đổi
2. Component hoặc component con phát ra event
3. Observable subscribe thông qua `AsyncPipe` emit giá trị
4. `ChangeDetectorRef.markForCheck()` được gọi thủ công

```typescript
@Component({
  selector: 'app-user-card',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,  // Opt-in
  template: `...`,
})
export class UserCardComponent {
  @Input({ required: true }) user!: User;
}
```

OnPush cải thiện performance đáng kể nhưng đòi hỏi developer phải hiểu rõ và tuân theo immutability. Signals giải quyết vấn đề này ở tầng sâu hơn.

### Signals — Reactivity chính xác

Thay vì hỏi "component nào có thể đã thay đổi?", Signals cho phép Angular biết chính xác **"giá trị nào vừa thay đổi và template nào đang dùng giá trị đó"**. Đây là fine-grained reactivity — chỉ re-render phần template thực sự bị ảnh hưởng.

---

## 3.2 Signals — Ba Primitives Cốt Lõi

### `signal()` — Writable State

`signal()` tạo ra một reactive value. Đọc signal bằng cách gọi nó như một function — đây là điểm khác biệt cú pháp quan trọng so với React's `useState`.

```typescript
import { signal } from '@angular/core';

// Khai báo signal với type inference
const count = signal(0);                    // Signal<number>
const name = signal('');                    // Signal<string>
const user = signal<User | null>(null);     // Signal<User | null>
const users = signal<User[]>([]);           // Signal<User[]>

// Đọc giá trị — gọi như function
console.log(count());   // 0
console.log(name());    // ''

// Cập nhật — set, update, mutate
count.set(5);                               // Set giá trị mới
count.update((current) => current + 1);    // Update dựa trên giá trị hiện tại
users.update((list) => [...list, newUser]); // Immutable update

// Với object/array — mutate (Angular 17+, cẩn thận khi dùng)
// mutate thay đổi in-place và notify, không tạo object mới
// Tuy nhiên set/update với spread operator được khuyến khích hơn
```

**Trong component:**

```typescript
@Component({
  selector: 'app-counter',
  standalone: true,
  template: `
    <div class="counter">
      <p>Số đếm: {{ count() }}</p>
      <p>Số đếm nhân đôi: {{ doubleCount() }}</p>

      <button mat-button (click)="decrement()">−</button>
      <button mat-button (click)="increment()">+</button>
      <button mat-stroked-button (click)="reset()">Reset</button>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class CounterComponent {
  // Signals là class properties — không cần lifecycle hook
  protected readonly count = signal(0);

  // computed — tự động update khi count thay đổi
  protected readonly doubleCount = computed(() => this.count() * 2);

  protected increment(): void {
    this.count.update((n) => n + 1);
  }

  protected decrement(): void {
    this.count.update((n) => Math.max(0, n - 1));
  }

  protected reset(): void {
    this.count.set(0);
  }
}
```

### `computed()` — Derived State

`computed()` tạo ra một signal **chỉ đọc** có giá trị được tính từ các signal khác. Angular tự động track dependencies và chỉ recalculate khi dependency thay đổi — tương tự `useMemo` trong React nhưng tự động hơn (không cần khai báo dependency array).

```typescript
import { signal, computed } from '@angular/core';

// Ví dụ đơn giản
const price = signal(100_000);
const quantity = signal(3);
const discount = signal(0.1);  // 10%

const subtotal = computed(() => price() * quantity());
const discountAmount = computed(() => subtotal() * discount());
const total = computed(() => subtotal() - discountAmount());

// computed chỉ recalculate khi dependency thực sự thay đổi
// Nếu price thay đổi: subtotal, discountAmount, total đều recalculate
// Nếu chỉ discount thay đổi: chỉ discountAmount và total recalculate
```

**Ví dụ thực tế — filter và sort:**

```typescript
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [CommonModule, MatTableModule, MatInputModule, MatSelectModule],
  templateUrl: './user-list.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserListComponent implements OnInit {
  private readonly userService = inject(UserService);
  private readonly destroyRef = inject(DestroyRef);

  // State signals
  private readonly allUsers = signal<User[]>([]);
  protected readonly searchQuery = signal('');
  protected readonly selectedRole = signal<UserRole | 'all'>('all');
  protected readonly sortDirection = signal<'asc' | 'desc'>('asc');
  protected readonly isLoading = signal(false);

  // Derived state — tự động cập nhật khi bất kỳ dependency nào thay đổi
  protected readonly filteredUsers = computed(() => {
    const users = this.allUsers();
    const query = this.searchQuery().toLowerCase();
    const role = this.selectedRole();

    return users.filter((user) => {
      const matchesSearch =
        user.displayName.toLowerCase().includes(query) ||
        user.email.toLowerCase().includes(query);

      const matchesRole = role === 'all' || user.role === role;

      return matchesSearch && matchesRole;
    });
  });

  protected readonly sortedUsers = computed(() => {
    const users = [...this.filteredUsers()]; // Copy để không mutate
    const direction = this.sortDirection();

    return users.sort((a, b) => {
      const comparison = a.displayName.localeCompare(b.displayName);
      return direction === 'asc' ? comparison : -comparison;
    });
  });

  protected readonly userCount = computed(() => this.filteredUsers().length);

  protected readonly isEmpty = computed(
    () => !this.isLoading() && this.sortedUsers().length === 0
  );

  ngOnInit(): void {
    this.isLoading.set(true);
    this.userService
      .getUsers()
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        finalize(() => this.isLoading.set(false))
      )
      .subscribe((response) => this.allUsers.set(response.data));
  }

  protected onSearch(query: string): void {
    this.searchQuery.set(query);
  }

  protected onRoleChange(role: UserRole | 'all'): void {
    this.selectedRole.set(role);
  }

  protected toggleSort(): void {
    this.sortDirection.update((d) => (d === 'asc' ? 'desc' : 'asc'));
  }
}
```

### `effect()` — Side Effects

`effect()` chạy một function mỗi khi bất kỳ signal nào nó đọc thay đổi. Đây là nơi xử lý side effects như logging, localStorage, analytics.

```typescript
import { effect, signal, inject } from '@angular/core';

@Component({
  selector: 'app-theme-toggle',
  standalone: true,
  template: `
    <button mat-icon-button (click)="toggleTheme()">
      {{ isDarkMode() ? '☀️' : '🌙' }}
    </button>
  `,
})
export class ThemeToggleComponent {
  protected readonly isDarkMode = signal(
    localStorage.getItem('theme') === 'dark'
  );

  constructor() {
    // effect() phải được tạo trong injection context (constructor)
    effect(() => {
      const isDark = this.isDarkMode();

      // Side effect: cập nhật DOM và localStorage
      document.documentElement.classList.toggle('dark-theme', isDark);
      localStorage.setItem('theme', isDark ? 'dark' : 'light');
    });
  }

  protected toggleTheme(): void {
    this.isDarkMode.update((dark) => !dark);
  }
}
```

**Quan trọng về `effect()` cleanup:**

```typescript
@Component({ ... })
export class DataStreamComponent {
  private readonly connectionStatus = signal<'connected' | 'disconnected'>('disconnected');

  constructor() {
    effect((onCleanup) => {
      const status = this.connectionStatus();

      if (status === 'connected') {
        const subscription = someWebSocketService.connect();

        // Cleanup function chạy trước mỗi lần effect re-run và khi component destroy
        onCleanup(() => {
          subscription.unsubscribe();
        });
      }
    });
  }
}
```

**Khi nào dùng `effect()` — và khi nào không:**

```typescript
// ✓ ĐÚNG — dùng effect cho side effects thực sự
effect(() => {
  analyticsService.track('page_view', { userId: this.currentUser()?.id });
});

// ✓ ĐÚNG — đồng bộ với external system
effect(() => {
  this.chartInstance?.update(this.chartData());
});

// ❌ SAI — đừng dùng effect để update signal khác
// Dùng computed() thay thế
effect(() => {
  this.filteredList.set(
    this.allItems().filter((item) => item.isActive)
  );
});

// ✓ ĐÚNG — dùng computed() cho derived state
const filteredList = computed(() =>
  this.allItems().filter((item) => item.isActive)
);
```

---

## 3.3 Signal-based Inputs và Outputs (Angular 17+)

### Input Signals

Angular 17 giới thiệu `input()` function như một thay thế reactive cho `@Input` decorator:

```typescript
import { Component, input, output, computed } from '@angular/core';

@Component({
  selector: 'app-product-card',
  standalone: true,
  imports: [CommonModule, MatCardModule, MatButtonModule],
  template: `
    <mat-card [class.featured]="isFeatured()">
      <mat-card-header>
        <mat-card-title>{{ product().name }}</mat-card-title>
        <mat-card-subtitle>{{ formattedPrice() }}</mat-card-subtitle>
      </mat-card-header>

      <mat-card-content>
        <p>{{ product().description }}</p>
        @if (stockStatus(); as status) {
          <span [class]="'stock-badge stock-badge--' + status.type">
            {{ status.label }}
          </span>
        }
      </mat-card-content>

      <mat-card-actions>
        <button
          mat-flat-button
          color="primary"
          [disabled]="product().stock === 0"
          (click)="addToCart.emit(product())"
        >
          Thêm vào giỏ
        </button>
      </mat-card-actions>
    </mat-card>
  `,
})
export class ProductCardComponent {
  // input.required() — bắt buộc, Angular sẽ báo lỗi nếu không truyền
  readonly product = input.required<Product>();

  // input() với giá trị mặc định — không bắt buộc
  readonly currency = input<string>('VND');
  readonly isFeatured = input<boolean>(false);

  // output() — thay thế @Output() EventEmitter
  readonly addToCart = output<Product>();
  readonly wishlistToggled = output<{ product: Product; isWishlisted: boolean }>();

  // computed() — tự động reactive với input signals
  protected readonly formattedPrice = computed(() =>
    new Intl.NumberFormat('vi-VN', {
      style: 'currency',
      currency: this.currency(),
    }).format(this.product().price)
  );

  protected readonly stockStatus = computed(() => {
    const stock = this.product().stock;
    if (stock === 0) return { type: 'out', label: 'Hết hàng' };
    if (stock <= 5) return { type: 'low', label: `Còn ${stock} sản phẩm` };
    return { type: 'available', label: 'Còn hàng' };
  });
}
```

**So sánh `@Input` và `input()`:**

```typescript
// @Input — cách cũ
@Input({ required: true }) product!: Product;
@Input() currency = 'VND';

// Cần ngOnChanges để react với thay đổi
ngOnChanges(changes: SimpleChanges): void {
  if (changes['product']) {
    this.updateFormattedPrice();
  }
}

// input() — cách mới
readonly product = input.required<Product>();
readonly currency = input('VND');

// computed() tự động reactive — không cần ngOnChanges
readonly formattedPrice = computed(() => formatPrice(this.product(), this.currency()));
```

### model() — Two-way Binding với Signals

`model()` tạo ra signal có thể được cập nhật từ cả parent lẫn component tự thân — dùng cho two-way binding:

```typescript
@Component({
  selector: 'app-quantity-input',
  standalone: true,
  imports: [CommonModule, MatButtonModule, MatInputModule],
  template: `
    <div class="quantity-input">
      <button mat-icon-button (click)="decrement()" [disabled]="quantity() <= min()">
        −
      </button>

      <input
        type="number"
        [value]="quantity()"
        (change)="onInputChange($event)"
        [min]="min()"
        [max]="max()"
      />

      <button mat-icon-button (click)="increment()" [disabled]="quantity() >= max()">
        +
      </button>
    </div>
  `,
})
export class QuantityInputComponent {
  // model() — two-way bindable signal
  readonly quantity = model<number>(1);

  readonly min = input<number>(1);
  readonly max = input<number>(99);

  protected increment(): void {
    this.quantity.update((q) => Math.min(q + 1, this.max()));
  }

  protected decrement(): void {
    this.quantity.update((q) => Math.max(q - 1, this.min()));
  }

  protected onInputChange(event: Event): void {
    const value = parseInt((event.target as HTMLInputElement).value, 10);
    if (!isNaN(value)) {
      this.quantity.set(Math.min(Math.max(value, this.min()), this.max()));
    }
  }
}
```

```html
<!-- Sử dụng với two-way binding -->
<app-quantity-input
  [(quantity)]="cartItemQuantity"
  [min]="1"
  [max]="product.stock"
/>
```

---

## 3.4 Kết hợp Signals và RxJS

Angular không bỏ RxJS — Signals và RxJS giải quyết các bài toán khác nhau và bổ sung cho nhau. Signals phù hợp cho **synchronous state**, RxJS phù hợp cho **async data streams**.

### `toSignal()` — Observable sang Signal

```typescript
import { toSignal } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-user-profile',
  standalone: true,
  imports: [CommonModule, MatCardModule],
  template: `
    @if (isLoading()) {
      <mat-spinner />
    } @else if (user(); as currentUser) {
      <mat-card>
        <mat-card-title>{{ currentUser.displayName }}</mat-card-title>
      </mat-card>
    } @else if (error()) {
      <p class="error">{{ error() }}</p>
    }
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserProfileComponent {
  private readonly userService = inject(UserService);
  private readonly route = inject(ActivatedRoute);

  // Chuyển route params thành signal
  private readonly userId = toSignal(
    this.route.paramMap.pipe(map((params) => params.get('id') ?? ''))
  );

  // switchMap để load user khi userId thay đổi
  private readonly userState = toSignal(
    toObservable(this.userId).pipe(
      filter(Boolean),
      switchMap((id) =>
        this.userService.getUserById(id).pipe(
          map((user) => ({ user, isLoading: false, error: null })),
          startWith({ user: null, isLoading: true, error: null }),
          catchError((err) =>
            of({ user: null, isLoading: false, error: err.message })
          )
        )
      )
    ),
    { initialValue: { user: null, isLoading: true, error: null } }
  );

  // Destructure state sang signals riêng lẻ
  protected readonly user = computed(() => this.userState().user);
  protected readonly isLoading = computed(() => this.userState().isLoading);
  protected readonly error = computed(() => this.userState().error);
}
```

### `toObservable()` — Signal sang Observable

```typescript
import { toObservable } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-search',
  standalone: true,
  imports: [CommonModule, MatInputModule, MatListModule],
  template: `
    <mat-form-field>
      <input
        matInput
        placeholder="Tìm kiếm..."
        (input)="searchQuery.set($any($event.target).value)"
      />
    </mat-form-field>

    @if (isSearching()) {
      <mat-spinner diameter="24" />
    }

    <mat-list>
      @for (result of searchResults(); track result.id) {
        <mat-list-item>{{ result.title }}</mat-list-item>
      }
    </mat-list>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class SearchComponent {
  private readonly searchService = inject(SearchService);

  readonly searchQuery = signal('');
  readonly isSearching = signal(false);

  // Chuyển signal thành Observable để dùng RxJS operators
  readonly searchResults = toSignal(
    toObservable(this.searchQuery).pipe(
      debounceTime(300),                 // Đợi người dùng ngừng gõ
      distinctUntilChanged(),            // Bỏ qua nếu query không đổi
      filter((query) => query.length >= 2),  // Chỉ search khi đủ ký tự
      tap(() => this.isSearching.set(true)),
      switchMap((query) =>               // Hủy request cũ khi query mới
        this.searchService.search(query)
      ),
      tap(() => this.isSearching.set(false)),
      catchError(() => {
        this.isSearching.set(false);
        return of([]);
      })
    ),
    { initialValue: [] as SearchResult[] }
  );
}
```

---

## 3.5 Change Detection với Signals

### `ChangeDetectionStrategy.OnPush` — Bắt buộc với Signals

Khi dùng Signals, luôn kết hợp với `OnPush`. Signals thông báo Angular chính xác khi nào cần re-render — `OnPush` đảm bảo Angular không chạy change detection thừa.

```typescript
@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './dashboard.component.html',
  // Luôn dùng OnPush với Signals
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class DashboardComponent {
  // Tất cả state là Signals — Angular biết chính xác khi nào cần update
  protected readonly stats = signal<DashboardStats | null>(null);
  protected readonly isLoading = signal(false);
  protected readonly activeTab = signal<'overview' | 'analytics' | 'reports'>('overview');

  protected readonly tabTitle = computed(() => {
    const titles = {
      overview: 'Tổng quan',
      analytics: 'Phân tích',
      reports: 'Báo cáo',
    };
    return titles[this.activeTab()];
  });
}
```

### `viewChild()` và `viewChildren()` — Signal-based DOM Queries

```typescript
@Component({
  selector: 'app-form-container',
  standalone: true,
  template: `
    <form>
      <app-text-input #nameInput label="Họ tên" />
      <app-text-input #emailInput label="Email" />

      @for (field of dynamicFields(); track field.id) {
        <app-text-input [label]="field.label" />
      }
    </form>
  `,
})
export class FormContainerComponent implements AfterViewInit {
  // viewChild — signal, không cần AfterViewInit để đọc (nhưng vẫn cần cho side effects)
  private readonly nameInput = viewChild.required<TextInputComponent>('nameInput');
  private readonly emailInput = viewChild<TextInputComponent>('emailInput');

  // viewChildren — signal trả về readonly array
  private readonly allInputs = viewChildren<TextInputComponent>(TextInputComponent);

  readonly dynamicFields = signal<{ id: string; label: string }[]>([]);

  // computed có thể dùng viewChildren signal
  readonly hasErrors = computed(() =>
    this.allInputs().some((input) => input.hasError())
  );

  focusFirstInvalidInput(): void {
    const firstInvalid = this.allInputs().find((input) => input.hasError());
    firstInvalid?.focus();
  }
}
```

---

## 3.6 Pattern Thực Tế — State Management với Signals

Trước khi học NgRx Signals Store (Chương 7), dưới đây là cách dùng Signals thuần để quản lý state trong một feature nhỏ:

```typescript
// features/cart/cart.service.ts
export interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
}

@Injectable({ providedIn: 'root' })
export class CartService {
  // Private writable state
  private readonly items = signal<CartItem[]>([]);

  // Public readonly computed state
  readonly cartItems = this.items.asReadonly();

  readonly itemCount = computed(() =>
    this.items().reduce((sum, item) => sum + item.quantity, 0)
  );

  readonly subtotal = computed(() =>
    this.items().reduce((sum, item) => sum + item.price * item.quantity, 0)
  );

  readonly isEmpty = computed(() => this.items().length === 0);

  addItem(product: Product, quantity = 1): void {
    this.items.update((items) => {
      const existingIndex = items.findIndex(
        (item) => item.productId === product.id
      );

      if (existingIndex >= 0) {
        // Update số lượng nếu đã có trong giỏ
        return items.map((item, index) =>
          index === existingIndex
            ? { ...item, quantity: item.quantity + quantity }
            : item
        );
      }

      // Thêm mới
      return [
        ...items,
        {
          productId: product.id,
          name: product.name,
          price: product.price,
          quantity,
        },
      ];
    });
  }

  removeItem(productId: string): void {
    this.items.update((items) =>
      items.filter((item) => item.productId !== productId)
    );
  }

  updateQuantity(productId: string, quantity: number): void {
    if (quantity <= 0) {
      this.removeItem(productId);
      return;
    }

    this.items.update((items) =>
      items.map((item) =>
        item.productId === productId ? { ...item, quantity } : item
      )
    );
  }

  clear(): void {
    this.items.set([]);
  }
}
```

```typescript
// Sử dụng trong component
@Component({
  selector: 'app-cart-icon',
  standalone: true,
  imports: [MatBadgeModule, MatIconButton, MatIcon],
  template: `
    <button mat-icon-button [matBadge]="cartService.itemCount()" matBadgeColor="warn">
      <mat-icon>shopping_cart</mat-icon>
    </button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class CartIconComponent {
  // inject CartService — Signals tự động reactive, không cần subscribe
  protected readonly cartService = inject(CartService);
}
```

---

## Tổng kết chương

Signals là hướng đi tương lai của Angular — không chỉ là cú pháp mới mà là một mô hình reactivity căn bản khác. Những điểm cốt lõi:

1. **Fine-grained reactivity**: Angular biết chính xác value nào thay đổi và chỉ update phần template liên quan — hiệu quả hơn Zone.js về mặt lý thuyết và thực tế.

2. **Ba primitives**: `signal()` cho writable state, `computed()` cho derived state, `effect()` cho side effects. Nắm vững ba cái này là đủ cho 90% use case.

3. **Signal-based APIs**: `input()`, `output()`, `model()`, `viewChild()`, `viewChildren()` là cách Angular hiện đại hóa các decorator truyền thống — reactive và type-safe hơn.

4. **Kết hợp với RxJS**: `toSignal()` và `toObservable()` là cầu nối quan trọng — Signals cho state đồng bộ, RxJS cho async streams phức tạp.

5. **Luôn dùng OnPush**: Signals và `ChangeDetectionStrategy.OnPush` là cặp đôi hoàn hảo — bắt buộc trong production code.

Chương tiếp theo sẽ đi vào **RxJS & HttpClient** — hệ thống xử lý async data của Angular, nơi Signals và RxJS giao nhau nhiều nhất trong thực tế.
