# Chương 13: Frontend Design Pattern

Design pattern trong frontend là các giải pháp đã được kiểm chứng cho những bài toán tổ chức component và logic thường gặp. Hiểu và áp dụng đúng pattern giúp code dễ đọc, dễ test, dễ mở rộng, và giảm thiểu sự phụ thuộc lẫn nhau giữa các phần của ứng dụng.

---

## 13.1. Composition Pattern ⭐⭐⭐⭐⭐

### Khái niệm

Composition (kết hợp) là cách xây dựng UI bằng cách **lắp ghép nhiều component nhỏ lại với nhau**, thay vì tạo một component lớn nhận vô số props để kiểm soát mọi biến thể. Đây là pattern cốt lõi trong React, thể hiện qua prop `children` và các "slot" tùy chỉnh.

### Vấn đề Composition giải quyết

Khi component phát triển thêm tính năng, cách tiếp cận không dùng composition dẫn đến **Props Explosion** — component nhận quá nhiều props và trở nên cứng nhắc:

```tsx
// ❌ Props explosion — mỗi nhu cầu mới thêm một prop
<Modal
  title="Xác nhận"
  subtitle="Hành động này không thể hoàn tác"
  showCloseButton
  closeButtonLabel="Hủy"
  showConfirmButton
  confirmButtonLabel="Xóa"
  confirmButtonVariant="danger"
  showFooter
  footerAlign="right"
  onClose={handleClose}
  onConfirm={handleConfirm}
  isLoading={isDeleting}
/>
```

### Giải pháp với Composition

Thay vì kiểm soát mọi thứ qua props, chia Modal thành các phần nhỏ có thể lắp ghép tự do:

```tsx
// ✅ Composition — linh hoạt, dễ mở rộng
<Modal onClose={handleClose}>
  <Modal.Header>
    <h2>Xác nhận</h2>
    <p>Hành động này không thể hoàn tác</p>
  </Modal.Header>

  <Modal.Body>
    <p>Bạn có chắc muốn xóa người dùng <strong>An</strong>?</p>
  </Modal.Body>

  <Modal.Footer>
    <Button variant="ghost" onClick={handleClose}>Hủy</Button>
    <Button variant="danger" onClick={handleConfirm} loading={isDeleting}>
      Xóa
    </Button>
  </Modal.Footer>
</Modal>
```

### Triển khai

```tsx
// components/Modal/Modal.tsx
interface ModalProps {
  children: React.ReactNode;
  onClose: () => void;
}

function Modal({ children, onClose }: ModalProps) {
  return (
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>
  );
}

function ModalHeader({ children }: { children: React.ReactNode }) {
  return <div className="modal__header">{children}</div>;
}

function ModalBody({ children }: { children: React.ReactNode }) {
  return <div className="modal__body">{children}</div>;
}

function ModalFooter({ children }: { children: React.ReactNode }) {
  return <div className="modal__footer">{children}</div>;
}

// Gắn các sub-component vào component chính
Modal.Header = ModalHeader;
Modal.Body = ModalBody;
Modal.Footer = ModalFooter;

export default Modal;
```

### children như "slot"

Composition còn thể hiện qua việc truyền component như một prop để render ở vị trí tùy chỉnh:

```tsx
interface CardProps {
  header?: React.ReactNode;   // Slot tùy chỉnh
  footer?: React.ReactNode;   // Slot tùy chỉnh
  children: React.ReactNode;  // Nội dung chính
}

function Card({ header, footer, children }: CardProps) {
  return (
    <div className="card">
      {header && <div className="card__header">{header}</div>}
      <div className="card__body">{children}</div>
      {footer && <div className="card__footer">{footer}</div>}
    </div>
  );
}

// Dùng
<Card
  header={<Badge color="green">Mới</Badge>}
  footer={<Button>Xem thêm</Button>}
>
  <p>Nội dung thẻ sản phẩm</p>
</Card>
```

---

## 13.2. Container / Presentational Pattern

### Khái niệm

Pattern này tách component thành hai vai trò rõ ràng:

- **Container Component:** Chịu trách nhiệm về **logic** — fetch data, quản lý state, xử lý sự kiện. Không quan tâm đến giao diện trông như thế nào.
- **Presentational Component:** Chịu trách nhiệm về **giao diện** — nhận data qua props và render UI. Không biết data đến từ đâu, không có logic nghiệp vụ.

### Triển khai

```tsx
// components/UserList/UserListContainer.tsx — Chỉ lo logic
function UserListContainer() {
  const [users, setUsers] = useState<User[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [searchQuery, setSearchQuery] = useState("");

  useEffect(() => {
    fetchUsers().then(setUsers).finally(() => setIsLoading(false));
  }, []);

  const filteredUsers = users.filter((u) =>
    u.name.toLowerCase().includes(searchQuery.toLowerCase())
  );

  function handleDelete(id: number) {
    setUsers((prev) => prev.filter((u) => u.id !== id));
  }

  return (
    <UserListView
      users={filteredUsers}
      isLoading={isLoading}
      searchQuery={searchQuery}
      onSearchChange={setSearchQuery}
      onDelete={handleDelete}
    />
  );
}
```

```tsx
// components/UserList/UserListView.tsx — Chỉ lo UI
interface UserListViewProps {
  users: User[];
  isLoading: boolean;
  searchQuery: string;
  onSearchChange: (query: string) => void;
  onDelete: (id: number) => void;
}

function UserListView({
  users,
  isLoading,
  searchQuery,
  onSearchChange,
  onDelete,
}: UserListViewProps) {
  if (isLoading) return <UserListSkeleton />;

  return (
    <div>
      <input
        value={searchQuery}
        onChange={(e) => onSearchChange(e.target.value)}
        placeholder="Tìm kiếm..."
      />
      <ul>
        {users.map((user) => (
          <li key={user.id}>
            <span>{user.name}</span>
            <button onClick={() => onDelete(user.id)}>Xóa</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### Lợi ích

- `UserListView` có thể render với dữ liệu giả để test hoặc làm Storybook story độc lập.
- Đổi nguồn data (từ REST sang GraphQL, hay từ fetch sang TanStack Query) chỉ cần sửa Container, View không đổi.

> **Lưu ý:** Với sự phổ biến của Custom Hook, phần logic của Container ngày nay thường được đặt trong hook thay vì component riêng. Pattern này vẫn có giá trị về mặt tư duy phân tách trách nhiệm.

---

## 13.3. Compound Component Pattern

### Khái niệm

Compound Component là pattern xây dựng một nhóm component **hoạt động cùng nhau như một đơn vị**, chia sẻ state nội bộ qua Context mà không để lộ ra ngoài. Người dùng component chỉ cần lắp ghép các phần, không cần biết cơ chế bên trong.

Ví dụ điển hình: `<select>` và `<option>` trong HTML — chúng phối hợp mà không cần props truyền tay.

### Triển khai — Accordion

```tsx
// components/Accordion/Accordion.tsx
import { createContext, useContext, useState } from "react";

interface AccordionContextValue {
  openItem: string | null;
  toggle: (id: string) => void;
}

const AccordionContext = createContext<AccordionContextValue | null>(null);

function useAccordion() {
  const ctx = useContext(AccordionContext);
  if (!ctx) throw new Error("Phải dùng bên trong <Accordion>");
  return ctx;
}

// Root component — giữ state, cung cấp context
function Accordion({ children }: { children: React.ReactNode }) {
  const [openItem, setOpenItem] = useState<string | null>(null);

  function toggle(id: string) {
    setOpenItem((prev) => (prev === id ? null : id));
  }

  return (
    <AccordionContext.Provider value={{ openItem, toggle }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

// Sub-component — tự lấy state từ context, không cần nhận props
function AccordionItem({
  id,
  children,
}: {
  id: string;
  children: React.ReactNode;
}) {
  return <div className="accordion__item">{children}</div>;
}

function AccordionTrigger({ id, children }: { id: string; children: React.ReactNode }) {
  const { openItem, toggle } = useAccordion();
  const isOpen = openItem === id;

  return (
    <button
      className="accordion__trigger"
      aria-expanded={isOpen}
      onClick={() => toggle(id)}
    >
      {children}
      <span>{isOpen ? "▲" : "▼"}</span>
    </button>
  );
}

function AccordionContent({ id, children }: { id: string; children: React.ReactNode }) {
  const { openItem } = useAccordion();
  if (openItem !== id) return null;
  return <div className="accordion__content">{children}</div>;
}

Accordion.Item = AccordionItem;
Accordion.Trigger = AccordionTrigger;
Accordion.Content = AccordionContent;

export default Accordion;
```

```tsx
// Sử dụng — API trực quan, state ẩn bên trong
<Accordion>
  <Accordion.Item id="faq-1">
    <Accordion.Trigger id="faq-1">Giao hàng mất bao lâu?</Accordion.Trigger>
    <Accordion.Content id="faq-1">
      <p>Từ 2 đến 5 ngày làm việc tùy khu vực.</p>
    </Accordion.Content>
  </Accordion.Item>

  <Accordion.Item id="faq-2">
    <Accordion.Trigger id="faq-2">Có đổi trả không?</Accordion.Trigger>
    <Accordion.Content id="faq-2">
      <p>Hỗ trợ đổi trả trong vòng 30 ngày.</p>
    </Accordion.Content>
  </Accordion.Item>
</Accordion>
```

### So sánh Compound Component vs Composition

| | Composition | Compound Component |
|---|---|---|
| Chia sẻ state | Không (mỗi phần độc lập) | Có (qua Context ẩn) |
| Giao tiếp giữa sub-component | Qua props từ cha | Tự động qua Context |
| Độ phức tạp | Thấp | Trung bình |
| Dùng khi | Layout, slot đơn giản | Tab, Accordion, Select, Dropdown |

---

## 13.4. Provider Pattern

### Khái niệm

Provider Pattern dùng React Context để **cung cấp dữ liệu hoặc dịch vụ** cho toàn bộ cây component con mà không cần truyền props qua từng tầng. Đây là pattern nền tảng cho state management, theming, và dependency injection trong React.

### Triển khai chuẩn

```tsx
// providers/AuthProvider.tsx
import { createContext, useContext, useState, useEffect } from "react";

interface User {
  id: number;
  name: string;
  role: "admin" | "user";
}

interface AuthContextValue {
  user: User | null;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthContextValue | null>(null);

// Custom hook — luôn tạo kèm để tránh dùng context sai chỗ
export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth phải dùng trong <AuthProvider>");
  return ctx;
}

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    // Kiểm tra session hiện có khi app khởi động
    getCurrentUser()
      .then(setUser)
      .catch(() => setUser(null))
      .finally(() => setIsLoading(false));
  }, []);

  async function login(email: string, password: string) {
    const user = await loginApi(email, password);
    setUser(user);
  }

  function logout() {
    logoutApi();
    setUser(null);
  }

  return (
    <AuthContext.Provider value={{ user, isLoading, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}
```

```tsx
// app/layout.tsx — Bọc ở root
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="vi">
      <body>
        <AuthProvider>
          <ThemeProvider>
            {children}
          </ThemeProvider>
        </AuthProvider>
      </body>
    </html>
  );
}

// Dùng ở bất kỳ component con nào
function NavBar() {
  const { user, logout } = useAuth();
  return (
    <nav>
      {user ? (
        <>
          <span>Xin chào, {user.name}</span>
          <button onClick={logout}>Đăng xuất</button>
        </>
      ) : (
        <Link href="/login">Đăng nhập</Link>
      )}
    </nav>
  );
}
```

---

## 13.5. Custom Hook Pattern ⭐⭐⭐⭐⭐

### Khái niệm

Custom Hook là hàm bắt đầu bằng `use`, đóng gói **stateful logic có thể tái sử dụng** — bao gồm state, effect, và bất kỳ hook nào khác. Đây là pattern mạnh nhất và phổ biến nhất trong React hiện đại để chia sẻ logic giữa nhiều component.

Pattern này đã được giới thiệu ở mục `4.8`. Phần này trình bày sâu hơn về thiết kế và các use case thực tế.

### Nguyên tắc thiết kế

Một Custom Hook tốt nên:
- **Làm đúng một việc** — có trách nhiệm rõ ràng.
- **Trả về đúng những gì cần thiết** — không trả về dư.
- **Có interface ổn định** — đổi implementation không làm vỡ caller.
- **Có thể test độc lập** — không phụ thuộc vào component cụ thể.

### Ví dụ 1: useLocalStorage

```tsx
// hooks/useLocalStorage.ts
import { useState, useEffect } from "react";

function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    if (typeof window === "undefined") return initialValue;
    try {
      const item = window.localStorage.getItem(key);
      return item ? (JSON.parse(item) as T) : initialValue;
    } catch {
      return initialValue;
    }
  });

  function setValue(value: T | ((prev: T) => T)) {
    const valueToStore =
      value instanceof Function ? value(storedValue) : value;
    setStoredValue(valueToStore);
    localStorage.setItem(key, JSON.stringify(valueToStore));
  }

  function removeValue() {
    setStoredValue(initialValue);
    localStorage.removeItem(key);
  }

  return [storedValue, setValue, removeValue] as const;
}

// Dùng
function Settings() {
  const [theme, setTheme] = useLocalStorage<"light" | "dark">("theme", "light");
  return (
    <button onClick={() => setTheme((t) => (t === "light" ? "dark" : "light"))}>
      Theme: {theme}
    </button>
  );
}
```

### Ví dụ 2: useDebounce

```tsx
// hooks/useDebounce.ts
import { useState, useEffect } from "react";

function useDebounce<T>(value: T, delay: number = 500): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Dùng — tìm kiếm không gọi API mỗi keystroke
function SearchBar() {
  const [query, setQuery] = useState("");
  const debouncedQuery = useDebounce(query, 400);

  const { data } = useQuery({
    queryKey: ["search", debouncedQuery],
    queryFn: () => searchUsers(debouncedQuery),
    enabled: debouncedQuery.length > 1,
  });

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Tìm kiếm..."
      />
      {data?.map((user) => <p key={user.id}>{user.name}</p>)}
    </div>
  );
}
```

### Ví dụ 3: useIntersectionObserver (Infinite Scroll)

```tsx
// hooks/useIntersectionObserver.ts
import { useEffect, useRef, useState } from "react";

function useIntersectionObserver(options?: IntersectionObserverInit) {
  const ref = useRef<HTMLDivElement>(null);
  const [isIntersecting, setIsIntersecting] = useState(false);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;

    const observer = new IntersectionObserver(([entry]) => {
      setIsIntersecting(entry.isIntersecting);
    }, options);

    observer.observe(el);
    return () => observer.disconnect();
  }, [options]);

  return { ref, isIntersecting };
}

// Dùng — load thêm khi scroll đến cuối trang
function InfiniteList() {
  const { ref, isIntersecting } = useIntersectionObserver({ threshold: 0.5 });
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } =
    useInfiniteQuery({ queryKey: ["posts"], queryFn: fetchPosts });

  useEffect(() => {
    if (isIntersecting && hasNextPage) fetchNextPage();
  }, [isIntersecting, hasNextPage, fetchNextPage]);

  return (
    <div>
      {data?.pages.map((page) =>
        page.posts.map((post) => <PostCard key={post.id} post={post} />)
      )}
      {/* Sentinel element — khi hiển thị trên màn hình thì load thêm */}
      <div ref={ref}>{isFetchingNextPage && <Spinner />}</div>
    </div>
  );
}
```

---

## 13.6. HOC — Higher-Order Component

### Khái niệm

HOC là hàm nhận một component làm đầu vào và trả về một component mới đã được **bổ sung thêm logic hoặc props**. Pattern này dùng để tái sử dụng logic giữa các component mà không thay đổi component gốc.

```
HOC(Component) → EnhancedComponent
```

### Triển khai

```tsx
// hocs/withAuth.tsx
import { useRouter } from "next/navigation";
import { useAuth } from "@/providers/AuthProvider";
import type { ComponentType } from "react";

function withAuth<P extends object>(WrappedComponent: ComponentType<P>) {
  function AuthGuard(props: P) {
    const { user, isLoading } = useAuth();
    const router = useRouter();

    if (isLoading) return <LoadingSpinner />;

    if (!user) {
      router.push("/login");
      return null;
    }

    return <WrappedComponent {...props} />;
  }

  // Giữ tên component gốc cho DevTools
  AuthGuard.displayName = `withAuth(${WrappedComponent.displayName ?? WrappedComponent.name})`;
  return AuthGuard;
}

// Dùng
function DashboardPage() {
  return <h1>Dashboard bí mật</h1>;
}

export default withAuth(DashboardPage);
```

HOC với role-based access:

```tsx
function withRole<P extends object>(
  WrappedComponent: ComponentType<P>,
  requiredRole: "admin" | "user"
) {
  function RoleGuard(props: P) {
    const { user } = useAuth();

    if (user?.role !== requiredRole) {
      return <p>Bạn không có quyền truy cập trang này.</p>;
    }

    return <WrappedComponent {...props} />;
  }

  RoleGuard.displayName = `withRole(${requiredRole}, ${WrappedComponent.name})`;
  return RoleGuard;
}

const AdminPanel = withRole(AdminPanelComponent, "admin");
```

### HOC vs Custom Hook

| | HOC | Custom Hook |
|---|---|---|
| Trả về | Component mới | Giá trị / hàm |
| Dùng cho | Bọc component, thêm props | Tái sử dụng logic trong component |
| Nesting | Dễ bị "HOC hell" khi dùng nhiều | Gọn, không nested |
| Debug | Khó hơn (thêm layer) | Dễ hơn |
| Xu hướng hiện đại | Giảm dần | Được ưa chuộng hơn |

> **Thực tế:** Custom Hook đã thay thế HOC trong phần lớn trường hợp. HOC vẫn hữu ích khi cần bọc component và kiểm soát render (như `withAuth`, `withErrorBoundary`, `React.memo`).

---

## 13.7. Render Props Pattern

### Khái niệm

Render Props là kỹ thuật chia sẻ logic bằng cách truyền một **hàm render** làm prop. Component nhận hàm này, cung cấp dữ liệu nội bộ vào hàm, và để caller quyết định render ra gì.

```tsx
// Prop là một hàm: children: (data) => ReactNode
<Component>{(data) => <UI data={data} />}</Component>
```

### Triển khai

```tsx
// components/MouseTracker.tsx
interface MouseTrackerProps {
  children: (position: { x: number; y: number }) => React.ReactNode;
}

function MouseTracker({ children }: MouseTrackerProps) {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  function handleMouseMove(e: React.MouseEvent) {
    setPosition({ x: e.clientX, y: e.clientY });
  }

  return (
    <div style={{ height: "100vh" }} onMouseMove={handleMouseMove}>
      {/* Gọi children như hàm, truyền vào dữ liệu nội bộ */}
      {children(position)}
    </div>
  );
}

// Dùng — caller quyết định render gì với position
<MouseTracker>
  {({ x, y }) => (
    <div
      style={{
        position: "fixed",
        left: x,
        top: y,
        pointerEvents: "none",
      }}
    >
      🐭
    </div>
  )}
</MouseTracker>
```

Render Props với data fetching:

```tsx
interface DataFetcherProps<T> {
  url: string;
  children: (state: {
    data: T | null;
    isLoading: boolean;
    error: Error | null;
  }) => React.ReactNode;
}

function DataFetcher<T>({ url, children }: DataFetcherProps<T>) {
  const [data, setData] = useState<T | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    fetch(url)
      .then((r) => r.json())
      .then(setData)
      .catch(setError)
      .finally(() => setIsLoading(false));
  }, [url]);

  return <>{children({ data, isLoading, error })}</>;
}

// Dùng
<DataFetcher<User[]> url="/api/users">
  {({ data: users, isLoading, error }) => {
    if (isLoading) return <Spinner />;
    if (error) return <ErrorMessage error={error} />;
    return <UserTable users={users ?? []} />;
  }}
</DataFetcher>
```

### Render Props vs Custom Hook

| | Render Props | Custom Hook |
|---|---|---|
| Interface | Hàm truyền vào prop | Hàm hook gọi trong component |
| Khả năng đọc | Nesting, khó đọc khi lồng nhiều | Tuyến tính, dễ đọc hơn |
| Tính linh hoạt | Cao — caller toàn quyền render | Cao — caller dùng data tùy ý |
| Xu hướng hiện đại | Giảm dần | Được ưa chuộng hơn |

> **Thực tế:** Như HOC, Render Props phần lớn đã được Custom Hook thay thế. Tuy nhiên, pattern này vẫn hữu ích trong một số thư viện UI và trường hợp cần tách hoàn toàn logic render khỏi logic data.

---

## 13.8. Headless Component (biết)

### Khái niệm

Headless Component là component cung cấp **logic và behavior, nhưng không có UI**. Toàn bộ giao diện do người dùng quyết định. Pattern này tách biệt hoàn toàn "cái gì" (logic) khỏi "trông như thế nào" (UI).

Đây là nền tảng của các thư viện UI phổ biến như **Radix UI**, **Headless UI**, và **React Aria**.

### Ví dụ đơn giản — useToggle

```tsx
// hooks/useToggle.ts — Logic không có UI
function useToggle(defaultValue = false) {
  const [isOn, setIsOn] = useState(defaultValue);

  return {
    isOn,
    toggle: () => setIsOn((prev) => !prev),
    setOn: () => setIsOn(true),
    setOff: () => setIsOn(false),
  };
}

// Component A — dùng làm Switch
function ThemeSwitch() {
  const { isOn, toggle } = useToggle(false);
  return (
    <button
      role="switch"
      aria-checked={isOn}
      onClick={toggle}
      className={`switch ${isOn ? "switch--on" : "switch--off"}`}
    >
      {isOn ? "Dark" : "Light"}
    </button>
  );
}

// Component B — cùng logic, UI hoàn toàn khác
function FeatureFlag({ label }: { label: string }) {
  const { isOn, toggle } = useToggle(false);
  return (
    <label>
      <input type="checkbox" checked={isOn} onChange={toggle} />
      {label} {isOn ? "(Bật)" : "(Tắt)"}
    </label>
  );
}
```

### Ví dụ thực tế — Headless Dropdown

```tsx
// hooks/useDropdown.ts — Pure logic, không có HTML
function useDropdown<T>(items: T[]) {
  const [isOpen, setIsOpen] = useState(false);
  const [selectedItem, setSelectedItem] = useState<T | null>(null);
  const [highlightedIndex, setHighlightedIndex] = useState(0);
  const containerRef = useRef<HTMLDivElement>(null);

  // Đóng khi click ra ngoài
  useEffect(() => {
    function handleClickOutside(e: MouseEvent) {
      if (!containerRef.current?.contains(e.target as Node)) {
        setIsOpen(false);
      }
    }
    document.addEventListener("mousedown", handleClickOutside);
    return () => document.removeEventListener("mousedown", handleClickOutside);
  }, []);

  // Keyboard navigation
  function handleKeyDown(e: React.KeyboardEvent) {
    switch (e.key) {
      case "ArrowDown":
        setHighlightedIndex((i) => Math.min(i + 1, items.length - 1));
        break;
      case "ArrowUp":
        setHighlightedIndex((i) => Math.max(i - 1, 0));
        break;
      case "Enter":
        if (isOpen) select(items[highlightedIndex]);
        else setIsOpen(true);
        break;
      case "Escape":
        setIsOpen(false);
        break;
    }
  }

  function select(item: T) {
    setSelectedItem(item);
    setIsOpen(false);
  }

  return {
    isOpen,
    selectedItem,
    highlightedIndex,
    containerRef,
    open: () => setIsOpen(true),
    close: () => setIsOpen(false),
    toggle: () => setIsOpen((prev) => !prev),
    select,
    handleKeyDown,
  };
}
```

```tsx
// Dùng headless hook với UI tùy chỉnh hoàn toàn
interface Country { code: string; name: string; }

function CountrySelect({ countries }: { countries: Country[] }) {
  const {
    isOpen, selectedItem, highlightedIndex,
    containerRef, toggle, select, handleKeyDown,
  } = useDropdown(countries);

  return (
    <div
      ref={containerRef}
      onKeyDown={handleKeyDown}
      tabIndex={0}
      className="select"
    >
      <button type="button" onClick={toggle} className="select__trigger">
        {selectedItem?.name ?? "Chọn quốc gia"}
        <span>{isOpen ? "▲" : "▼"}</span>
      </button>

      {isOpen && (
        <ul className="select__list" role="listbox">
          {countries.map((country, index) => (
            <li
              key={country.code}
              role="option"
              aria-selected={selectedItem?.code === country.code}
              className={`select__item ${index === highlightedIndex ? "select__item--highlighted" : ""}`}
              onClick={() => select(country)}
            >
              {country.name}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

## Tổng kết và So sánh

| Pattern | Mục đích | Khi nào dùng | Xu hướng |
|---|---|---|---|
| **Composition** | Lắp ghép UI linh hoạt | Mọi lúc — đây là cách React hoạt động | ⭐⭐⭐⭐⭐ |
| **Container/Presentational** | Tách logic khỏi UI | Khi muốn test UI độc lập | ⭐⭐⭐ (Custom Hook thay phần logic) |
| **Compound Component** | Nhóm component chia sẻ state ẩn | Tab, Accordion, Select, Menu | ⭐⭐⭐⭐ |
| **Provider Pattern** | Cung cấp dữ liệu/dịch vụ toàn cục | Theme, Auth, Config | ⭐⭐⭐⭐⭐ |
| **Custom Hook** | Tái sử dụng stateful logic | Mọi lúc có logic dùng lại | ⭐⭐⭐⭐⭐ |
| **HOC** | Bọc component, thêm behavior | Auth guard, error boundary | ⭐⭐⭐ (Custom Hook ưu tiên hơn) |
| **Render Props** | Chia sẻ logic, caller tự render | Thư viện UI linh hoạt | ⭐⭐ (Custom Hook ưu tiên hơn) |
| **Headless Component** | Logic không UI, tái sử dụng tối đa | Design system, thư viện UI | ⭐⭐⭐⭐ |
