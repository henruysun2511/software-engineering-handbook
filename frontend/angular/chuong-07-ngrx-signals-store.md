# Chương 7: NgRx Signals Store

## Giới thiệu chương

NgRx Signals Store là thế hệ tiếp theo của state management trong Angular — được giới thiệu cùng với NgRx 17 và chính thức stable từ NgRx 18. Khác với NgRx cổ điển (Store/Actions/Effects/Selectors/Reducers) đòi hỏi lượng boilerplate lớn, NgRx Signals Store xây dựng trên nền tảng Angular Signals và cung cấp API gọn hơn đáng kể trong khi vẫn giữ được tính predictable và testability.

Chương này đi từ lý do tại sao cần state management, đến cấu trúc cơ bản, các tính năng nâng cao như `withEntities`, `withCallState`, và cuối cùng là các patterns thực tế cho ứng dụng production.

---

## 7.1 Tại sao cần State Management

### Vấn đề của Component-local State

Khi ứng dụng phát triển, một số state cần được chia sẻ giữa nhiều component không có quan hệ cha-con trực tiếp. Component-local state và service với `BehaviorSubject` (như đã thấy ở chương trước) giải quyết được phần nào, nhưng có giới hạn rõ ràng:

```
Vấn đề 1 — Shared State:
  HeaderComponent (hiển thị user avatar)
  ├── cần User state
  SidebarComponent (hiển thị role-based menu)
  ├── cần User state
  DashboardComponent (greeting)
  ├── cần User state

  → Nếu mỗi component tự fetch, có 3 HTTP requests cho cùng 1 data
  → Nếu dùng service, cần quản lý lifecycle và sync thủ công

Vấn đề 2 — Predictability:
  UserService.updateUser() được gọi từ component A
  → Component B và C không biết state đã thay đổi
  → Phải dùng Subject.next() thủ công, dễ quên, dễ bug

Vấn đề 3 — Async Complexity:
  Load users → Loading state
  Load thành công → Success state + data
  Load thất bại → Error state
  → Mỗi service phải tự implement pattern này, không nhất quán
```

NgRx Signals Store giải quyết bằng cách cung cấp một **centralized, reactive, và type-safe state container** với convention rõ ràng.

### Khi nào dùng NgRx Signals Store

Không phải mọi state đều cần NgRx. Quyết định theo nguyên tắc:

- **Component-local Signal**: UI state chỉ dùng trong một component (modal open/close, tab active, loading state cục bộ)
- **Service + Signal**: State chia sẻ giữa vài component liên quan (CartService, NotificationService)
- **NgRx Signals Store**: Feature state phức tạp, nhiều component dùng, có async operations và cần tracing (User management, Product catalog, Order processing)

---

## 7.2 Cấu trúc cơ bản của NgRx Signals Store

### `signalStore()` — Tạo store

```typescript
// features/user/store/user.store.ts
import {
  signalStore,
  withState,
  withComputed,
  withMethods,
  patchState,
} from '@ngrx/signals';

// Định nghĩa shape của state
interface UserState {
  users: User[];
  selectedUserId: string | null;
  isLoading: boolean;
  error: string | null;
  filter: {
    search: string;
    role: UserRole | 'all';
  };
}

// Giá trị khởi tạo
const initialState: UserState = {
  users: [],
  selectedUserId: null,
  isLoading: false,
  error: null,
  filter: {
    search: '',
    role: 'all',
  },
};

export const UserStore = signalStore(
  // Scope: 'root' = singleton, hoặc component-level (xem phần 7.5)
  { providedIn: 'root' },

  // 1. withState — khởi tạo state, tự động tạo signal cho từng property
  withState(initialState),

  // 2. withComputed — derived state, tương tự computed() nhưng trong store
  withComputed((store) => ({
    // store.users() → Signal<User[]>
    // store.filter() → Signal<{ search: string; role: UserRole | 'all' }>

    filteredUsers: computed(() => {
      const users = store.users();
      const { search, role } = store.filter();

      return users.filter((user) => {
        const matchesSearch =
          !search ||
          user.displayName.toLowerCase().includes(search.toLowerCase()) ||
          user.email.toLowerCase().includes(search.toLowerCase());

        const matchesRole = role === 'all' || user.role === role;

        return matchesSearch && matchesRole;
      });
    }),

    selectedUser: computed(() =>
      store.users().find((u) => u.id === store.selectedUserId()) ?? null
    ),

    totalUsers: computed(() => store.users().length),

    isEmpty: computed(
      () => !store.isLoading() && store.filteredUsers().length === 0
    ),
  })),

  // 3. withMethods — actions và mutations
  withMethods((store, userService = inject(UserService)) => ({
    // Synchronous method — cập nhật state trực tiếp
    selectUser(userId: string | null): void {
      patchState(store, { selectedUserId: userId });
    },

    setFilter(filter: Partial<UserState['filter']>): void {
      patchState(store, (state) => ({
        filter: { ...state.filter, ...filter },
      }));
    },

    clearFilter(): void {
      patchState(store, { filter: initialState.filter });
    },

    // Async method — gọi service và cập nhật state
    async loadUsers(): Promise<void> {
      patchState(store, { isLoading: true, error: null });

      try {
        const response = await firstValueFrom(userService.getUsers());
        patchState(store, { users: response.data, isLoading: false });
      } catch (error) {
        patchState(store, {
          error: 'Không thể tải danh sách người dùng',
          isLoading: false,
        });
      }
    },

    async createUser(dto: CreateUserDto): Promise<User> {
      const user = await firstValueFrom(userService.createUser(dto));
      patchState(store, (state) => ({
        users: [...state.users, user],
      }));
      return user;
    },

    async updateUser(id: string, dto: UpdateUserDto): Promise<void> {
      const updated = await firstValueFrom(userService.updateUser(id, dto));
      patchState(store, (state) => ({
        users: state.users.map((u) => (u.id === id ? updated : u)),
      }));
    },

    async deleteUser(id: string): Promise<void> {
      await firstValueFrom(userService.deleteUser(id));
      patchState(store, (state) => ({
        users: state.users.filter((u) => u.id !== id),
        selectedUserId:
          state.selectedUserId === id ? null : state.selectedUserId,
      }));
    },
  }))
);
```

### Sử dụng Store trong Component

```typescript
// features/user/user-list/user-list.component.ts
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [
    CommonModule,
    MatTableModule,
    MatInputModule,
    MatSelectModule,
    MatButtonModule,
    MatIconModule,
    MatProgressSpinnerModule,
    MatMenuModule,
  ],
  templateUrl: './user-list.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserListComponent implements OnInit {
  // Inject store — giống inject service thông thường
  protected readonly store = inject(UserStore);

  protected readonly displayedColumns = ['name', 'email', 'role', 'status', 'actions'];

  ngOnInit(): void {
    // Load data khi component khởi tạo
    this.store.loadUsers();
  }

  protected onSearchChange(query: string): void {
    this.store.setFilter({ search: query });
  }

  protected onRoleChange(role: UserRole | 'all'): void {
    this.store.setFilter({ role });
  }

  protected onSelectUser(user: User): void {
    this.store.selectUser(user.id);
  }

  protected async onDeleteUser(user: User): Promise<void> {
    if (!confirm(`Xóa người dùng "${user.displayName}"?`)) return;

    await this.store.deleteUser(user.id);
  }
}
```

```html
<!-- user-list.component.html -->
<div class="user-list">
  <!-- Toolbar -->
  <mat-toolbar>
    <mat-form-field class="search-field">
      <mat-icon matPrefix>search</mat-icon>
      <input
        matInput
        placeholder="Tìm kiếm..."
        (input)="onSearchChange($any($event.target).value)"
      />
    </mat-form-field>

    <mat-form-field>
      <mat-select
        [value]="store.filter().role"
        (selectionChange)="onRoleChange($event.value)"
      >
        <mat-option value="all">Tất cả vai trò</mat-option>
        <mat-option value="ADMIN">Admin</mat-option>
        <mat-option value="EDITOR">Editor</mat-option>
        <mat-option value="VIEWER">Viewer</mat-option>
      </mat-select>
    </mat-form-field>

    <span class="spacer"></span>

    <span class="count">{{ store.totalUsers() }} người dùng</span>
  </mat-toolbar>

  <!-- Loading state -->
  @if (store.isLoading()) {
    <div class="loading-container">
      <mat-spinner />
    </div>
  }

  <!-- Error state -->
  @if (store.error()) {
    <div class="error-state">
      <mat-icon color="warn">error_outline</mat-icon>
      <p>{{ store.error() }}</p>
      <button mat-stroked-button (click)="store.loadUsers()">Thử lại</button>
    </div>
  }

  <!-- Empty state -->
  @if (store.isEmpty()) {
    <div class="empty-state">
      <mat-icon>people_outline</mat-icon>
      <p>Không tìm thấy người dùng nào</p>
      @if (store.filter().search || store.filter().role !== 'all') {
        <button mat-stroked-button (click)="store.clearFilter()">
          Xóa bộ lọc
        </button>
      }
    </div>
  }

  <!-- Data table -->
  @if (!store.isLoading() && !store.isEmpty()) {
    <mat-table [dataSource]="store.filteredUsers()">
      <ng-container matColumnDef="name">
        <mat-header-cell *matHeaderCellDef>Họ tên</mat-header-cell>
        <mat-cell *matCellDef="let user">{{ user.displayName }}</mat-cell>
      </ng-container>

      <ng-container matColumnDef="email">
        <mat-header-cell *matHeaderCellDef>Email</mat-header-cell>
        <mat-cell *matCellDef="let user">{{ user.email }}</mat-cell>
      </ng-container>

      <ng-container matColumnDef="role">
        <mat-header-cell *matHeaderCellDef>Vai trò</mat-header-cell>
        <mat-cell *matCellDef="let user">
          <span [class]="'role-badge role-badge--' + user.role.toLowerCase()">
            {{ user.role }}
          </span>
        </mat-cell>
      </ng-container>

      <ng-container matColumnDef="actions">
        <mat-header-cell *matHeaderCellDef>Thao tác</mat-header-cell>
        <mat-cell *matCellDef="let user">
          <button mat-icon-button [matMenuTriggerFor]="menu">
            <mat-icon>more_vert</mat-icon>
          </button>
          <mat-menu #menu>
            <button mat-menu-item (click)="onSelectUser(user)">
              <mat-icon>visibility</mat-icon> Xem chi tiết
            </button>
            <button mat-menu-item [routerLink]="[user.id, 'edit']">
              <mat-icon>edit</mat-icon> Chỉnh sửa
            </button>
            <button mat-menu-item color="warn" (click)="onDeleteUser(user)">
              <mat-icon>delete</mat-icon> Xóa
            </button>
          </mat-menu>
        </mat-cell>
      </ng-container>

      <mat-header-row *matHeaderRowDef="displayedColumns" />
      <mat-row
        *matRowDef="let user; columns: displayedColumns"
        [class.selected]="user.id === store.selectedUserId()"
        (click)="onSelectUser(user)"
      />
    </mat-table>
  }
</div>
```

---

## 7.3 rxMethod — Tích hợp RxJS vào Store

`rxMethod` cho phép dùng RxJS operators bên trong store methods — cần thiết khi cần debounce, switchMap, hoặc các operators phức tạp:

```typescript
import { rxMethod } from '@ngrx/signals/rxjs-interop';
import { tapResponse } from '@ngrx/operators';

export const UserStore = signalStore(
  { providedIn: 'root' },
  withState(initialState),
  withComputed(/* ... */),
  withMethods((store, userService = inject(UserService)) => ({
    // rxMethod cho search với debounce — không thể làm với async/await
    readonly loadUsersWithFilter = rxMethod<UserQueryParams>(
      pipe(
        debounceTime(300),
        distinctUntilChanged(
          (a, b) => JSON.stringify(a) === JSON.stringify(b)
        ),
        tap(() => patchState(store, { isLoading: true, error: null })),
        switchMap((params) =>
          userService.getUsers(params).pipe(
            tapResponse({
              next: (response) =>
                patchState(store, {
                  users: response.data,
                  isLoading: false,
                }),
              error: () =>
                patchState(store, {
                  error: 'Tải dữ liệu thất bại',
                  isLoading: false,
                }),
            })
          )
        )
      )
    ),

    // rxMethod cho real-time updates qua WebSocket
    readonly connectToUserUpdates = rxMethod<void>(
      pipe(
        switchMap(() =>
          userWebSocketService.userUpdates$.pipe(
            tapResponse({
              next: (updatedUser) =>
                patchState(store, (state) => ({
                  users: state.users.map((u) =>
                    u.id === updatedUser.id ? updatedUser : u
                  ),
                })),
              error: (err) => console.error('WebSocket error:', err),
            })
          )
        )
      )
    ),
  }))
);
```

```typescript
// Sử dụng rxMethod trong component
@Component({ ... })
export class UserListComponent implements OnInit {
  protected readonly store = inject(UserStore);

  // Computed filter params để truyền vào rxMethod
  private readonly filterParams = computed<UserQueryParams>(() => ({
    search: this.store.filter().search || undefined,
    role: this.store.filter().role !== 'all' ? this.store.filter().role : undefined,
  }));

  ngOnInit(): void {
    // rxMethod tự động subscribe và cleanup khi component destroy
    this.store.loadUsersWithFilter(this.filterParams);
    this.store.connectToUserUpdates();
  }
}
```

---

## 7.4 withEntities — CRUD Entity Management

`withEntities` là feature plugin của NgRx Signals Store chuyên cho CRUD operations với entity collections — tự động tạo normalized state và các selector thông dụng:

```typescript
import {
  withEntities,
  setAllEntities,
  addEntity,
  updateEntity,
  removeEntity,
  setEntity,
} from '@ngrx/signals/entities';

export const ProductStore = signalStore(
  { providedIn: 'root' },

  // withEntities tự động tạo:
  // - entities: Record<string, Product>
  // - ids: string[]
  // - entityMap: computed signal
  withEntities<Product>(),

  // Thêm state khác nếu cần
  withState({
    isLoading: false,
    error: null as string | null,
    selectedId: null as string | null,
    filter: { search: '', category: 'all' },
  }),

  withComputed(({ entities, ids, selectedId, filter }) => ({
    // Lấy tất cả entities theo đúng thứ tự ids
    products: computed(() => ids().map((id) => entities()[id])),

    selectedProduct: computed(() =>
      selectedId() ? entities()[selectedId()!] ?? null : null
    ),

    filteredProducts: computed(() => {
      const allProducts = ids().map((id) => entities()[id]);
      const { search, category } = filter();

      return allProducts.filter((p) => {
        const matchesSearch =
          !search || p.name.toLowerCase().includes(search.toLowerCase());
        const matchesCategory = category === 'all' || p.category === category;
        return matchesSearch && matchesCategory;
      });
    }),

    totalCount: computed(() => ids().length),
  })),

  withMethods(
    (store, productService = inject(ProductService)) => ({
      async loadProducts(): Promise<void> {
        patchState(store, { isLoading: true, error: null });
        try {
          const products = await firstValueFrom(productService.getProducts());
          // setAllEntities — thay thế toàn bộ collection
          patchState(store, setAllEntities(products));
          patchState(store, { isLoading: false });
        } catch {
          patchState(store, { error: 'Tải sản phẩm thất bại', isLoading: false });
        }
      },

      async createProduct(dto: CreateProductDto): Promise<void> {
        const product = await firstValueFrom(productService.createProduct(dto));
        // addEntity — thêm entity mới
        patchState(store, addEntity(product));
      },

      async updateProduct(id: string, dto: UpdateProductDto): Promise<void> {
        const updated = await firstValueFrom(
          productService.updateProduct(id, dto)
        );
        // updateEntity — cập nhật entity theo id
        patchState(store, updateEntity({ id, changes: updated }));
      },

      async deleteProduct(id: string): Promise<void> {
        await firstValueFrom(productService.deleteProduct(id));
        // removeEntity — xóa theo id
        patchState(store, removeEntity(id));
      },

      selectProduct(id: string | null): void {
        patchState(store, { selectedId: id });
      },
    })
  )
);
```

---

## 7.5 withCallState — Standardized Loading/Error State

`withCallState` là một feature tiện ích để chuẩn hóa loading/error state pattern — không cần tự viết lại cho mỗi store:

```typescript
import { withCallState, setLoading, setLoaded, setError } from '@ngrx/signals/entities';
// Nếu package chưa có, tự implement:

// shared/store/call-state.feature.ts
export type CallState = 'init' | 'loading' | 'loaded' | { error: string };

export function withCallState() {
  return signalStoreFeature(
    withState<{ callState: CallState }>({ callState: 'init' }),
    withComputed(({ callState }) => ({
      isLoading: computed(() => callState() === 'loading'),
      isLoaded: computed(() => callState() === 'loaded'),
      error: computed(() => {
        const state = callState();
        return typeof state === 'object' ? state.error : null;
      }),
    }))
  );
}

export const setLoading = (): { callState: CallState } => ({
  callState: 'loading',
});

export const setLoaded = (): { callState: CallState } => ({
  callState: 'loaded',
});

export const setError = (error: string): { callState: CallState } => ({
  callState: { error },
});
```

```typescript
// Sử dụng withCallState trong store
export const OrderStore = signalStore(
  { providedIn: 'root' },
  withEntities<Order>(),
  withCallState(),  // Thêm isLoading, isLoaded, error tự động

  withMethods((store, orderService = inject(OrderService)) => ({
    async loadOrders(): Promise<void> {
      patchState(store, setLoading());
      try {
        const orders = await firstValueFrom(orderService.getOrders());
        patchState(store, setAllEntities(orders), setLoaded());
      } catch (error: unknown) {
        const message = error instanceof Error ? error.message : 'Lỗi không xác định';
        patchState(store, setError(message));
      }
    },
  }))
);
```

---

## 7.6 Store Features — Tái sử dụng Logic giữa Stores

Store features cho phép đóng gói logic tái sử dụng và compose vào nhiều stores khác nhau:

```typescript
// shared/store/features/pagination.feature.ts
import { signalStoreFeature, withState, withComputed, withMethods } from '@ngrx/signals';

export interface PaginationState {
  currentPage: number;
  pageSize: number;
  totalItems: number;
}

const initialPaginationState: PaginationState = {
  currentPage: 1,
  pageSize: 20,
  totalItems: 0,
};

export function withPagination() {
  return signalStoreFeature(
    withState(initialPaginationState),

    withComputed(({ currentPage, pageSize, totalItems }) => ({
      totalPages: computed(() => Math.ceil(totalItems() / pageSize())),
      hasNextPage: computed(
        () => currentPage() < Math.ceil(totalItems() / pageSize())
      ),
      hasPreviousPage: computed(() => currentPage() > 1),
      offset: computed(() => (currentPage() - 1) * pageSize()),
    })),

    withMethods((store) => ({
      setPage(page: number): void {
        patchState(store, { currentPage: page });
      },

      nextPage(): void {
        patchState(store, (state) => ({
          currentPage: Math.min(
            state.currentPage + 1,
            Math.ceil(state.totalItems / state.pageSize)
          ),
        }));
      },

      previousPage(): void {
        patchState(store, (state) => ({
          currentPage: Math.max(1, state.currentPage - 1),
        }));
      },

      setPageSize(size: number): void {
        patchState(store, { pageSize: size, currentPage: 1 });
      },

      setTotalItems(total: number): void {
        patchState(store, { totalItems: total });
      },
    }))
  );
}
```

```typescript
// shared/store/features/sort.feature.ts
export type SortDirection = 'asc' | 'desc';

export interface SortState<T extends string = string> {
  sortField: T | null;
  sortDirection: SortDirection;
}

export function withSort<T extends string = string>(defaultField?: T) {
  return signalStoreFeature(
    withState<SortState<T>>({
      sortField: defaultField ?? null,
      sortDirection: 'asc',
    }),

    withMethods((store) => ({
      sort(field: T): void {
        patchState(store, (state) => ({
          sortField: field,
          sortDirection:
            state.sortField === field && state.sortDirection === 'asc'
              ? 'desc'
              : 'asc',
        }));
      },

      clearSort(): void {
        patchState(store, { sortField: null, sortDirection: 'asc' });
      },
    }))
  );
}
```

```typescript
// Compose features vào store
type UserSortField = 'displayName' | 'email' | 'createdAt';

export const UserStore = signalStore(
  { providedIn: 'root' },
  withEntities<User>(),
  withCallState(),
  withPagination(),
  withSort<UserSortField>('displayName'),

  withState({ filter: { search: '', role: 'all' as UserRole | 'all' } }),

  withComputed((store) => ({
    sortedAndFilteredUsers: computed(() => {
      let users = store.ids().map((id) => store.entities()[id]);

      // Apply filter
      const { search, role } = store.filter();
      if (search) {
        users = users.filter((u) =>
          u.displayName.toLowerCase().includes(search.toLowerCase())
        );
      }
      if (role !== 'all') {
        users = users.filter((u) => u.role === role);
      }

      // Apply sort
      const { sortField, sortDirection } = store;
      if (sortField()) {
        users = [...users].sort((a, b) => {
          const field = sortField()!;
          const aVal = String(a[field]);
          const bVal = String(b[field]);
          const cmp = aVal.localeCompare(bVal);
          return sortDirection() === 'asc' ? cmp : -cmp;
        });
      }

      // Apply pagination
      const start = store.offset();
      const end = start + store.pageSize();
      return users.slice(start, end);
    }),
  })),

  withMethods((store, userService = inject(UserService)) => ({
    async loadUsers(): Promise<void> {
      patchState(store, setLoading());
      try {
        const response = await firstValueFrom(
          userService.getUsers({
            page: store.currentPage(),
            limit: store.pageSize(),
          })
        );
        patchState(
          store,
          setAllEntities(response.data),
          setLoaded()
        );
        store.setTotalItems(response.pagination.total);
      } catch {
        patchState(store, setError('Không thể tải danh sách người dùng'));
      }
    },
  }))
);
```

---

## 7.7 withHooks — Lifecycle của Store

`withHooks` cho phép thực thi logic khi store được khởi tạo hoặc hủy:

```typescript
import { withHooks } from '@ngrx/signals';

export const UserStore = signalStore(
  { providedIn: 'root' },
  withEntities<User>(),
  withCallState(),

  withMethods((store, userService = inject(UserService)) => ({
    loadUsers: rxMethod<void>(
      pipe(
        tap(() => patchState(store, setLoading())),
        switchMap(() =>
          userService.getUsers().pipe(
            tapResponse({
              next: (response) =>
                patchState(store, setAllEntities(response.data), setLoaded()),
              error: () =>
                patchState(store, setError('Tải dữ liệu thất bại')),
            })
          )
        )
      )
    ),
  })),

  // Chạy logic khi store được khởi tạo
  withHooks({
    onInit(store) {
      // Load data ngay khi store được tạo
      store.loadUsers();

      // Có thể inject services ở đây
      const analyticsService = inject(AnalyticsService);
      analyticsService.track('user_store_initialized');
    },

    onDestroy(store) {
      // Cleanup khi store bị hủy (component-level store)
      console.log('UserStore destroyed');
    },
  })
);
```

---

## 7.8 Component-level Store — Instance riêng biệt

Đôi khi cần mỗi component có store riêng, không chia sẻ state với component khác — ví dụ wizard form, multi-step checkout:

```typescript
// Không dùng providedIn: 'root' — chỉ tạo khi được provide trong component
export const WizardStore = signalStore(
  withState({
    steps: [] as WizardStep[],
    currentStepIndex: 0,
    formData: {} as Record<string, unknown>,
    isSubmitting: false,
  }),

  withComputed(({ steps, currentStepIndex, formData }) => ({
    currentStep: computed(() => steps()[currentStepIndex()]),
    isFirstStep: computed(() => currentStepIndex() === 0),
    isLastStep: computed(
      () => currentStepIndex() === steps().length - 1
    ),
    completionPercentage: computed(() =>
      steps().length === 0
        ? 0
        : Math.round((currentStepIndex() / steps().length) * 100)
    ),
  })),

  withMethods((store) => ({
    initialize(steps: WizardStep[]): void {
      patchState(store, { steps, currentStepIndex: 0, formData: {} });
    },

    nextStep(stepData: Record<string, unknown>): void {
      patchState(store, (state) => ({
        formData: { ...state.formData, ...stepData },
        currentStepIndex: Math.min(
          state.currentStepIndex + 1,
          state.steps.length - 1
        ),
      }));
    },

    previousStep(): void {
      patchState(store, (state) => ({
        currentStepIndex: Math.max(0, state.currentStepIndex - 1),
      }));
    },

    goToStep(index: number): void {
      patchState(store, { currentStepIndex: index });
    },
  }))
);

// Component provide store cho chính nó và component con
@Component({
  selector: 'app-registration-wizard',
  standalone: true,
  providers: [WizardStore],  // Instance mới cho component này
  template: `
    <div class="wizard">
      <app-wizard-progress
        [steps]="store.steps()"
        [currentIndex]="store.currentStepIndex()"
        [percentage]="store.completionPercentage()"
      />
      <router-outlet />
    </div>
  `,
})
export class RegistrationWizardComponent implements OnInit {
  protected readonly store = inject(WizardStore);

  ngOnInit(): void {
    this.store.initialize([
      { id: 'personal', label: 'Thông tin cá nhân' },
      { id: 'address', label: 'Địa chỉ' },
      { id: 'payment', label: 'Thanh toán' },
      { id: 'review', label: 'Xác nhận' },
    ]);
  }
}

// Component con inject cùng store instance từ parent
@Component({
  selector: 'app-personal-info-step',
  standalone: true,
  template: `...`,
})
export class PersonalInfoStepComponent {
  // Nhận WizardStore instance từ RegistrationWizardComponent provider
  protected readonly store = inject(WizardStore);

  protected onNext(data: PersonalInfoData): void {
    this.store.nextStep(data);
  }
}
```

---

## 7.9 Testing NgRx Signals Store

Store là plain class — dễ test hơn NgRx cổ điển không cần TestBed phức tạp:

```typescript
// features/user/store/user.store.spec.ts
import { TestBed } from '@angular/core/testing';
import { UserStore } from './user.store';
import { UserService } from '../services/user.service';

describe('UserStore', () => {
  let store: InstanceType<typeof UserStore>;
  let userServiceMock: jest.Mocked<UserService>;

  const mockUsers: User[] = [
    { id: '1', displayName: 'Alice', email: 'alice@example.com', role: 'ADMIN', isActive: true, createdAt: '', updatedAt: '' },
    { id: '2', displayName: 'Bob', email: 'bob@example.com', role: 'EDITOR', isActive: true, createdAt: '', updatedAt: '' },
  ];

  beforeEach(() => {
    userServiceMock = {
      getUsers: jest.fn().mockReturnValue(
        of({ data: mockUsers, pagination: { page: 1, limit: 20, total: 2, totalPages: 1 } })
      ),
      deleteUser: jest.fn().mockReturnValue(of(void 0)),
    } as unknown as jest.Mocked<UserService>;

    TestBed.configureTestingModule({
      providers: [
        UserStore,
        { provide: UserService, useValue: userServiceMock },
      ],
    });

    store = TestBed.inject(UserStore);
  });

  it('should initialize with empty state', () => {
    expect(store.users().length).toBe(0);
    expect(store.isLoading()).toBe(false);
    expect(store.error()).toBeNull();
  });

  it('should load users', async () => {
    await store.loadUsers();

    expect(store.users().length).toBe(2);
    expect(store.isLoading()).toBe(false);
    expect(userServiceMock.getUsers).toHaveBeenCalledTimes(1);
  });

  it('should filter users by search', async () => {
    await store.loadUsers();
    store.setFilter({ search: 'alice' });

    expect(store.filteredUsers().length).toBe(1);
    expect(store.filteredUsers()[0].displayName).toBe('Alice');
  });

  it('should delete user and remove from state', async () => {
    await store.loadUsers();
    await store.deleteUser('1');

    expect(store.users().length).toBe(1);
    expect(store.users().find((u) => u.id === '1')).toBeUndefined();
  });

  it('should set error state on load failure', async () => {
    userServiceMock.getUsers.mockReturnValue(
      throwError(() => new Error('Network error'))
    );

    await store.loadUsers();

    expect(store.error()).not.toBeNull();
    expect(store.isLoading()).toBe(false);
  });
});
```

---

## Tổng kết chương

NgRx Signals Store là bước tiến lớn so với NgRx cổ điển — ít boilerplate hơn đáng kể trong khi vẫn giữ được tính predictable và composable. Những điểm cốt lõi:

1. **Ba building blocks**: `withState()` khai báo state, `withComputed()` tạo derived state, `withMethods()` định nghĩa actions. Đây là cấu trúc nhất quán cho mọi store.

2. **`patchState()`** là cách duy nhất để cập nhật state — immutable, predictable, và Angular DevTools có thể trace.

3. **`rxMethod()`** là cầu nối giữa NgRx Signals Store và RxJS — cần thiết khi cần debounce, switchMap, hay WebSocket integration.

4. **`withEntities()`** giảm 80% boilerplate cho CRUD entity management. Dùng `setAllEntities`, `addEntity`, `updateEntity`, `removeEntity` thay vì tự viết array manipulation.

5. **Store Features** (`withPagination`, `withSort`, `withCallState`) là cách đúng đắn để tái sử dụng logic state — compose vào store thay vì copy-paste.

6. **Component-level store** (không có `providedIn`) cho phép mỗi component có instance store riêng — hoàn hảo cho wizard, multi-step forms, và isolated feature widgets.

Chương tiếp theo sẽ đi vào **Angular Material & SCSS** — hệ thống component UI và styling chuyên nghiệp, bao gồm custom theming, SCSS architecture, và các pattern styling thực tế.
