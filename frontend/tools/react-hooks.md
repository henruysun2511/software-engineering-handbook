# TỔNG HỢP KIẾN THỨC REACT HOOKS

> **Môn học:** Lập trình Web nâng cao  
> **Chủ đề:** React Hooks – Khái niệm, Phân loại và Ứng dụng  

---

## MỤC LỤC

1. [Tổng quan về React Hooks](#1-tổng-quan-về-react-hooks)
2. [useState](#2-usestate)
3. [useEffect](#3-useeffect)
4. [useContext](#4-usecontext)
5. [useReducer](#5-usereducer)
6. [useRef](#6-useref)
7. [useMemo](#7-usememo)
8. [useCallback](#8-usecallback)
9. [useLayoutEffect](#9-uselayouteffect)
10. [useId](#10-useid)
11. [Custom Hook](#11-custom-hook)
12. [Bảng so sánh tổng hợp](#12-bảng-so-sánh-tổng-hợp)

---

## 1. Tổng quan về React Hooks

### 1.1 Định nghĩa

**React Hooks** là các hàm đặc biệt được giới thiệu từ React 16.8 (2019), cho phép **Function Component** sử dụng các tính năng vốn chỉ dành riêng cho Class Component như: quản lý state, vòng đời component, context, v.v.

Trước khi có Hooks, Function Component chỉ là các hàm thuần túy (pure function) không có trạng thái. Hooks đã xóa bỏ ranh giới đó.

### 1.2 Quy tắc sử dụng Hooks (Rules of Hooks)

React yêu cầu bắt buộc tuân thủ hai quy tắc sau:

| # | Quy tắc | Mô tả |
|---|---------|-------|
| 1 | **Chỉ gọi ở cấp cao nhất** | Không gọi Hook bên trong vòng lặp, điều kiện, hoặc hàm lồng nhau |
| 2 | **Chỉ gọi trong React Function** | Chỉ dùng trong Function Component hoặc Custom Hook, không dùng trong hàm JS thông thường |

### 1.3 Phân loại Hooks

```
React Hooks
├── State Hooks        → useState, useReducer
├── Effect Hooks       → useEffect, useLayoutEffect
├── Context Hooks      → useContext
├── Ref Hooks          → useRef
├── Performance Hooks  → useMemo, useCallback
├── Utility Hooks      → useId
└── Custom Hooks       → do người dùng tự định nghĩa
```

---

## 2. useState

### 2.1 Định nghĩa

`useState` là Hook cơ bản nhất, dùng để **khai báo và quản lý state** (trạng thái) bên trong Function Component. Mỗi khi state thay đổi, React sẽ tự động re-render lại component để cập nhật giao diện.

### 2.2 Cú pháp

```javascript
const [state, setState] = useState(initialValue);
```

| Tham số / Giá trị | Mô tả |
|-------------------|-------|
| `initialValue` | Giá trị khởi tạo (số, chuỗi, mảng, object, ...) |
| `state` | Giá trị state hiện tại |
| `setState` | Hàm cập nhật state, kích hoạt re-render |

### 2.3 Ví dụ: Bộ đếm (Counter)

```jsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Số lần bấm: {count}</p>
      <button onClick={() => setCount(count + 1)}>Tăng</button>
      <button onClick={() => setCount(count - 1)}>Giảm</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}
```

**Giải thích:** `count` giữ giá trị đếm hiện tại. Khi người dùng bấm nút, `setCount` được gọi với giá trị mới, React re-render lại component và hiển thị số mới trên màn hình.

### 2.4 Ví dụ nâng cao: Quản lý object state

```jsx
function UserForm() {
  const [user, setUser] = useState({ name: '', age: '' });

  const handleChange = (field, value) => {
    // Dùng spread operator để giữ các field khác không bị ghi đè
    setUser(prev => ({ ...prev, [field]: value }));
  };

  return (
    <div>
      <input
        placeholder="Tên"
        value={user.name}
        onChange={e => handleChange('name', e.target.value)}
      />
      <input
        placeholder="Tuổi"
        value={user.age}
        onChange={e => handleChange('age', e.target.value)}
      />
      <p>Kết quả: {user.name} - {user.age} tuổi</p>
    </div>
  );
}
```

> **Lưu ý:** `setState` không merge tự động với object như `this.setState` trong Class Component. Phải dùng spread `{ ...prev }` để giữ lại các thuộc tính cũ.

---

## 3. useEffect

### 3.1 Định nghĩa

`useEffect` là Hook dùng để thực hiện các **side effect** (tác dụng phụ) bên trong Function Component. Side effect là những thao tác không thuộc về việc render UI, ví dụ: gọi API, đăng ký sự kiện, thao tác DOM, đặt timer, v.v.

Hook này tương đương với sự kết hợp của `componentDidMount`, `componentDidUpdate`, và `componentWillUnmount` trong Class Component.

### 3.2 Cú pháp

```javascript
useEffect(() => {
  // Logic side effect

  return () => {
    // Cleanup (tùy chọn) – chạy khi component unmount hoặc trước lần chạy tiếp theo
  };
}, [dependencies]); // Mảng phụ thuộc
```

### 3.3 Các chế độ hoạt động

| Dependency Array | Thời điểm chạy |
|-----------------|----------------|
| Không truyền | Chạy sau **mỗi lần** render |
| `[]` (mảng rỗng) | Chỉ chạy **một lần** sau lần render đầu tiên (như `componentDidMount`) |
| `[a, b]` | Chạy lại khi `a` hoặc `b` **thay đổi** |

### 3.4 Ví dụ: Gọi API khi component mount

```jsx
import { useState, useEffect } from 'react';

function UserList() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Gọi API khi component lần đầu xuất hiện
    fetch('https://jsonplaceholder.typicode.com/users')
      .then(res => res.json())
      .then(data => {
        setUsers(data);
        setLoading(false);
      });
  }, []); // [] → chỉ chạy một lần

  if (loading) return <p>Đang tải...</p>;

  return (
    <ul>
      {users.map(u => <li key={u.id}>{u.name}</li>)}
    </ul>
  );
}
```

### 3.5 Ví dụ: Cleanup – Hủy đăng ký sự kiện

```jsx
function WindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);

    // Đăng ký sự kiện
    window.addEventListener('resize', handleResize);

    // Cleanup: hủy đăng ký khi component unmount
    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []);

  return <p>Chiều rộng cửa sổ: {width}px</p>;
}
```

> **Lưu ý:** Nếu quên cleanup, sự kiện sẽ vẫn tồn tại sau khi component bị xóa khỏi DOM, gây **memory leak**.

---

## 4. useContext

### 4.1 Định nghĩa

`useContext` là Hook cho phép component **đọc giá trị từ React Context** mà không cần dùng Consumer wrapper. Context là cơ chế chia sẻ dữ liệu giữa nhiều component ở các cấp khác nhau mà không cần truyền props qua từng tầng (prop drilling).

### 4.2 Cú pháp

```javascript
// Bước 1: Tạo Context
const MyContext = createContext(defaultValue);

// Bước 2: Cung cấp giá trị (ở component cha)
<MyContext.Provider value={someValue}>
  <ChildComponent />
</MyContext.Provider>

// Bước 3: Sử dụng trong component con
const value = useContext(MyContext);
```

### 4.3 Ví dụ: Theme sáng/tối

```jsx
import { createContext, useContext, useState } from 'react';

// 1. Tạo Context
const ThemeContext = createContext('light');

// 2. Component cha – cung cấp giá trị
function App() {
  const [theme, setTheme] = useState('light');

  return (
    <ThemeContext.Provider value={theme}>
      <div>
        <Toolbar />
        <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
          Đổi theme
        </button>
      </div>
    </ThemeContext.Provider>
  );
}

// 3. Component con – sử dụng Context
function Toolbar() {
  const theme = useContext(ThemeContext);

  return (
    <div style={{
      background: theme === 'light' ? '#fff' : '#333',
      color: theme === 'light' ? '#000' : '#fff',
      padding: '10px'
    }}>
      Theme hiện tại: {theme}
    </div>
  );
}
```

**Giải thích:** `Toolbar` không cần nhận `theme` qua props. Nó tự đọc giá trị từ `ThemeContext` thông qua `useContext`, dù nằm sâu bao nhiêu cấp trong cây component.

---

## 5. useReducer

### 5.1 Định nghĩa

`useReducer` là Hook quản lý state phức tạp theo mô hình **Reducer**, tương tự Redux. Thay vì gọi trực tiếp hàm set như `useState`, ta mô tả **hành động (action)** và một hàm reducer thuần túy sẽ tính toán state mới dựa trên action đó.

Phù hợp khi state có nhiều trường hoặc các thao tác cập nhật phức tạp, có logic liên quan đến nhau.

### 5.2 Cú pháp

```javascript
const [state, dispatch] = useReducer(reducer, initialState);

function reducer(state, action) {
  switch (action.type) {
    case 'ACTION_TYPE':
      return { ...state, /* state mới */ };
    default:
      return state;
  }
}
```

### 5.3 Ví dụ: Giỏ hàng đơn giản

```jsx
import { useReducer } from 'react';

// Hàm reducer – logic xử lý tập trung
function cartReducer(state, action) {
  switch (action.type) {
    case 'ADD_ITEM':
      return { ...state, items: [...state.items, action.payload], count: state.count + 1 };
    case 'REMOVE_ITEM':
      return {
        ...state,
        items: state.items.filter((_, i) => i !== action.payload),
        count: state.count - 1
      };
    case 'CLEAR':
      return { items: [], count: 0 };
    default:
      return state;
  }
}

function ShoppingCart() {
  const [cart, dispatch] = useReducer(cartReducer, { items: [], count: 0 });

  return (
    <div>
      <h3>Giỏ hàng ({cart.count} sản phẩm)</h3>
      <button onClick={() => dispatch({ type: 'ADD_ITEM', payload: 'Sản phẩm ' + (cart.count + 1) })}>
        Thêm sản phẩm
      </button>
      <button onClick={() => dispatch({ type: 'CLEAR' })}>
        Xóa tất cả
      </button>
      <ul>
        {cart.items.map((item, i) => (
          <li key={i}>
            {item}
            <button onClick={() => dispatch({ type: 'REMOVE_ITEM', payload: i })}>✕</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

> **Khi nào dùng `useReducer` thay `useState`?**  
> Khi state có nhiều trường liên quan, khi logic cập nhật phức tạp, hoặc khi muốn tách biệt rõ ràng **logic** (reducer) và **UI** (component).

---

## 6. useRef

### 6.1 Định nghĩa

`useRef` tạo ra một **đối tượng tham chiếu** (ref object) có thuộc tính `.current` có thể thay đổi mà **không gây re-render** khi bị thay đổi. Có hai mục đích chính:

- **Truy cập trực tiếp phần tử DOM** (focus, scroll, đo kích thước)
- **Lưu giá trị giữa các lần render** mà không cần trigger re-render

### 6.2 Cú pháp

```javascript
const ref = useRef(initialValue);
// Truy cập: ref.current
```

### 6.3 Ví dụ 1: Focus vào input khi bấm nút

```jsx
import { useRef } from 'react';

function AutoFocusInput() {
  const inputRef = useRef(null);

  const handleFocus = () => {
    // Truy cập trực tiếp DOM element
    inputRef.current.focus();
  };

  return (
    <div>
      <input ref={inputRef} type="text" placeholder="Nhập tên..." />
      <button onClick={handleFocus}>Focus vào ô nhập</button>
    </div>
  );
}
```

### 6.4 Ví dụ 2: Lưu giá trị trước đó (previous value)

```jsx
import { useState, useRef, useEffect } from 'react';

function TrackPreviousValue() {
  const [count, setCount] = useState(0);
  const prevCount = useRef(0);

  useEffect(() => {
    // Cập nhật ref sau mỗi render (không gây re-render)
    prevCount.current = count;
  });

  return (
    <div>
      <p>Hiện tại: {count} | Trước đó: {prevCount.current}</p>
      <button onClick={() => setCount(c => c + 1)}>Tăng</button>
    </div>
  );
}
```

> **Phân biệt `useRef` và `useState`:**  
> `useState` → thay đổi giá trị **gây re-render**.  
> `useRef` → thay đổi `.current` **không gây re-render**, phù hợp lưu các giá trị nội bộ thuần túy.

---

## 7. useMemo

### 7.1 Định nghĩa

`useMemo` là Hook dùng để **ghi nhớ (cache) kết quả của một phép tính tốn kém**. Kết quả chỉ được tính lại khi các dependency thay đổi, giúp tránh tính toán lặp lại không cần thiết trong mỗi lần render.

### 7.2 Cú pháp

```javascript
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

### 7.3 Ví dụ: Lọc danh sách lớn

```jsx
import { useState, useMemo } from 'react';

// Giả sử danh sách có hàng nghìn phần tử
const bigList = Array.from({ length: 10000 }, (_, i) => ({
  id: i,
  name: `Sản phẩm ${i}`,
  price: Math.floor(Math.random() * 1000000)
}));

function ProductFilter() {
  const [query, setQuery] = useState('');
  const [sortAsc, setSortAsc] = useState(true);

  // useMemo: chỉ lọc + sắp xếp lại khi query hoặc sortAsc thay đổi
  const filteredProducts = useMemo(() => {
    console.log('Đang lọc danh sách...'); // Chỉ in khi dependency thay đổi
    return bigList
      .filter(p => p.name.toLowerCase().includes(query.toLowerCase()))
      .sort((a, b) => sortAsc ? a.price - b.price : b.price - a.price);
  }, [query, sortAsc]);

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Tìm kiếm..."
      />
      <button onClick={() => setSortAsc(s => !s)}>
        Sắp xếp: {sortAsc ? 'Tăng dần' : 'Giảm dần'}
      </button>
      <p>Tìm thấy {filteredProducts.length} sản phẩm</p>
      <ul>
        {filteredProducts.slice(0, 10).map(p => (
          <li key={p.id}>{p.name} – {p.price.toLocaleString()} đ</li>
        ))}
      </ul>
    </div>
  );
}
```

> **Lưu ý:** Không nên dùng `useMemo` cho mọi phép tính. Bản thân việc ghi nhớ cũng có chi phí. Chỉ dùng khi phép tính **thực sự nặng** hoặc khi giá trị được dùng làm dependency cho Hook khác.

---

## 8. useCallback

### 8.1 Định nghĩa

`useCallback` là Hook dùng để **ghi nhớ (cache) một hàm**. Hàm chỉ được tạo lại khi các dependency thay đổi. Điều này đặc biệt hữu ích khi truyền hàm như props xuống component con được bọc bởi `React.memo`, tránh re-render không cần thiết.

### 8.2 Cú pháp

```javascript
const memoizedCallback = useCallback(() => {
  doSomething(a, b);
}, [a, b]);
```

### 8.3 Ví dụ: Tránh re-render component con

```jsx
import { useState, useCallback, memo } from 'react';

// Component con được bọc bởi React.memo
// Chỉ re-render khi props thực sự thay đổi
const Button = memo(({ onClick, label }) => {
  console.log(`Render button: ${label}`);
  return <button onClick={onClick}>{label}</button>;
});

function Parent() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');

  // Không dùng useCallback: hàm mới tạo ra mỗi render → Button luôn re-render
  // Dùng useCallback: hàm giữ nguyên tham chiếu → Button chỉ re-render khi cần
  const handleIncrement = useCallback(() => {
    setCount(c => c + 1);
  }, []); // Không có dependency → hàm được tạo 1 lần duy nhất

  return (
    <div>
      <input value={text} onChange={e => setText(e.target.value)} placeholder="Nhập gì đó..." />
      <p>Count: {count}</p>
      {/* Button này sẽ KHÔNG re-render khi text thay đổi */}
      <Button onClick={handleIncrement} label="Tăng" />
    </div>
  );
}
```

### 8.4 So sánh useMemo và useCallback

| Hook | Cache | Dùng khi |
|------|-------|----------|
| `useMemo` | **Kết quả** của hàm (giá trị) | Tính toán nặng, tránh tính lại |
| `useCallback` | **Bản thân** hàm (tham chiếu) | Truyền hàm vào component con tối ưu |

---

## 9. useLayoutEffect

### 9.1 Định nghĩa

`useLayoutEffect` có cú pháp giống hệt `useEffect`, nhưng **chạy đồng bộ ngay sau khi React cập nhật DOM** và **trước khi trình duyệt vẽ lại màn hình**. Phù hợp khi cần đọc hoặc thay đổi DOM để tránh hiện tượng nhấp nháy giao diện (flash/flicker).

### 9.2 Cú pháp

```javascript
useLayoutEffect(() => {
  // Đọc/thay đổi DOM ngay sau khi React cập nhật
  return () => { /* cleanup */ };
}, [dependencies]);
```

### 9.3 Ví dụ: Đo kích thước DOM element

```jsx
import { useState, useRef, useLayoutEffect } from 'react';

function MeasureBox() {
  const boxRef = useRef(null);
  const [height, setHeight] = useState(0);

  useLayoutEffect(() => {
    // Đọc kích thước TRƯỚC KHI trình duyệt vẽ lại
    // Dùng useEffect sẽ gây ra "nhấp nháy" vì đọc sau khi vẽ
    if (boxRef.current) {
      setHeight(boxRef.current.getBoundingClientRect().height);
    }
  }, []);

  return (
    <div>
      <div ref={boxRef} style={{ padding: '20px', background: '#eee' }}>
        Nội dung của box
      </div>
      <p>Chiều cao box: {height}px</p>
    </div>
  );
}
```

### 9.4 So sánh useEffect và useLayoutEffect

| | `useEffect` | `useLayoutEffect` |
|-|------------|-------------------|
| Thời điểm chạy | Sau khi trình duyệt vẽ xong | Sau DOM update, trước khi vẽ |
| Đặc tính | Bất đồng bộ | Đồng bộ (blocking) |
| Ứng dụng | Gọi API, subscriptions | Đọc/sửa layout DOM |
| Rủi ro | Có thể nhấp nháy | Chặn render → dùng khi cần thiết |

---

## 10. useId

### 10.1 Định nghĩa

`useId` (React 18+) là Hook tạo ra **ID duy nhất** ổn định cho mỗi lần gọi, đảm bảo ID nhất quán giữa server-side rendering (SSR) và client-side rendering. Thường dùng để liên kết các thuộc tính HTML như `id`, `htmlFor`, `aria-*`.

### 10.2 Cú pháp

```javascript
const id = useId();
```

### 10.3 Ví dụ: Form field có label liên kết

```jsx
import { useId } from 'react';

// Component có thể tái sử dụng nhiều lần trên cùng trang
function LabeledInput({ label, type = 'text' }) {
  const id = useId(); // Mỗi instance có ID riêng, không trùng nhau

  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} type={type} />
    </div>
  );
}

function LoginForm() {
  return (
    <form>
      {/* Mỗi component tự sinh ID riêng → không bao giờ trùng */}
      <LabeledInput label="Email" type="email" />
      <LabeledInput label="Mật khẩu" type="password" />
    </form>
  );
}
```

> **Lưu ý:** Không dùng `useId` để tạo key trong danh sách. Key phải đến từ dữ liệu, không phải từ Hook.

---

## 11. Custom Hook

### 11.1 Định nghĩa

**Custom Hook** là hàm JavaScript thông thường do người dùng tự định nghĩa, **tên bắt đầu bằng `use`**, có thể gọi các Hook khác bên trong. Custom Hook cho phép **tách biệt và tái sử dụng logic** có trạng thái (stateful logic) giữa nhiều component khác nhau.

### 11.2 Lợi ích

- Tái sử dụng logic mà không cần chia sẻ state (khác với Context)
- Component sạch hơn, tập trung vào UI
- Dễ test độc lập

### 11.3 Ví dụ 1: useFetch – Gọi API

```jsx
import { useState, useEffect } from 'react';

// Custom Hook: đóng gói logic gọi API
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    fetch(url)
      .then(res => {
        if (!res.ok) throw new Error('Lỗi mạng');
        return res.json();
      })
      .then(data => { setData(data); setLoading(false); })
      .catch(err => { setError(err.message); setLoading(false); });
  }, [url]);

  return { data, loading, error };
}

// Sử dụng trong component – rất gọn gàng
function PostList() {
  const { data, loading, error } = useFetch('https://jsonplaceholder.typicode.com/posts');

  if (loading) return <p>Đang tải...</p>;
  if (error) return <p>Lỗi: {error}</p>;

  return (
    <ul>
      {data.slice(0, 5).map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### 11.4 Ví dụ 2: useLocalStorage – Lưu state vào localStorage

```jsx
import { useState } from 'react';

function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = (value) => {
    try {
      setStoredValue(value);
      localStorage.setItem(key, JSON.stringify(value));
    } catch (error) {
      console.error(error);
    }
  };

  return [storedValue, setValue];
}

// Sử dụng: giống useState nhưng tự động lưu vào localStorage
function Settings() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');

  return (
    <div>
      <p>Theme: {theme}</p>
      <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
        Đổi theme
      </button>
    </div>
  );
}
```

---

## 12. Bảng so sánh tổng hợp

| Hook | Mục đích chính | Re-render? | Ghi chú |
|------|---------------|------------|---------|
| `useState` | Quản lý state đơn giản | ✅ Có | Hook cơ bản nhất |
| `useReducer` | Quản lý state phức tạp / nhiều action | ✅ Có | Thay thế useState khi logic phức tạp |
| `useEffect` | Side effects (API, timer, event) | ❌ Không | Chạy sau khi render |
| `useLayoutEffect` | Side effects cần đọc/sửa DOM ngay | ❌ Không | Chạy trước khi trình duyệt vẽ |
| `useContext` | Đọc giá trị từ Context | ✅ Khi context đổi | Tránh prop drilling |
| `useRef` | Tham chiếu DOM / lưu giá trị nội bộ | ❌ Không | `.current` thay đổi không gây render |
| `useMemo` | Cache kết quả tính toán nặng | ❌ Không | Tối ưu hiệu năng |
| `useCallback` | Cache tham chiếu hàm | ❌ Không | Dùng với React.memo |
| `useId` | Tạo ID duy nhất (SSR-safe) | ❌ Không | React 18+, dùng cho accessibility |
| Custom Hook | Đóng gói và tái sử dụng logic | Tùy thuộc | Tên bắt đầu bằng `use` |

---

## Tài liệu tham khảo

- React Official Documentation: https://react.dev/reference/react
- React Hooks RFC: https://github.com/reactjs/rfcs/blob/main/text/0068-react-hooks.md
- Dan Abramov, "Making Sense of React Hooks" (2018): https://medium.com/@dan_abramov

---

*Tài liệu được biên soạn phục vụ mục đích học tập và báo cáo tiểu luận.*
