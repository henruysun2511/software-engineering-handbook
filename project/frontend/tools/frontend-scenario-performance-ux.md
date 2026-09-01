# 📱 Phỏng vấn Frontend: Tình huống, Hiệu năng, Re-render & UX

---

## 🎯 PHẦN 1: CÂU HỎI TÌNH HUỐNG (SCENARIO-BASED)

### Tình huống 1: List 10,000 items hiển thị không được

**Câu hỏi:**
"Bạn có một list chứa 10,000 sản phẩm, khi scroll app bị lag nặng. Làm sao fix?"

**Vấn đề:**
- DOM render 10,000 elements cùng lúc rất nặng
- Browser phải calculate layout + paint tất cả
- Memory dùng quá nhiều
- FPS thấp = janky scrolling

**Giải pháp:**

**Cách 1: Virtual Scrolling (BEST)**
```jsx
import { FixedSizeList as List } from 'react-window';

export default function ProductList({ products }) {
  const Row = ({ index, style }) => (
    <div style={style} className="p-4 border-b">
      <h3>{products[index].name}</h3>
      <p>${products[index].price}</p>
    </div>
  );

  return (
    <List
      height={600}           // Chiều cao container
      itemCount={10000}      // Tổng items
      itemSize={80}          // Chiều cao mỗi item
      width="100%"
    >
      {Row}
    </List>
  );
}
// ✅ Chỉ render ~15 items visible + buffer, không phải 10,000
```

**Cách 2: Pagination**
```jsx
export default function ProductList() {
  const [page, setPage] = useState(1);
  const itemsPerPage = 20;
  const startIdx = (page - 1) * itemsPerPage;
  const paginatedProducts = products.slice(startIdx, startIdx + itemsPerPage);

  return (
    <>
      {paginatedProducts.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
      <button onClick={() => setPage(page + 1)}>Next</button>
    </>
  );
}
```

**Cách 3: Infinite Scroll với Intersection Observer**
```jsx
export default function InfiniteList() {
  const [items, setItems] = useState([]);
  const [page, setPage] = useState(1);
  const observerTarget = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting) {
        // Load more khi scroll gần cuối
        loadMoreItems(page + 1);
        setPage(p => p + 1);
      }
    });

    if (observerTarget.current) {
      observer.observe(observerTarget.current);
    }

    return () => observer.disconnect();
  }, [page]);

  return (
    <div>
      {items.map(item => <div key={item.id}>{item.name}</div>)}
      <div ref={observerTarget}>Loading...</div>
    </div>
  );
}
```

**So sánh:**
| Cách | Ưu điểm | Nhược điểm |
|------|---------|-----------|
| Virtual Scroll | Smooth, scroll nhanh | Phức tạp hơn |
| Pagination | Đơn giản, SEO tốt | User phải click next |
| Infinite Scroll | UX tốt | Khó biết có bao nhiêu items |

---

### Tình huống 2: API call chậm, page bị freeze

**Câu hỏi:**
"Bạn gọi API mất 5 giây, trong lúc chờ UI bị freeze, user không thể interact. Làm sao?"

**Vấn đề:**
- API call blocking main thread
- UI không response được
- Không có loading indicator

**Giải pháp:**

**Cách 1: Loading state + async/await**
```jsx
export default function UserProfile() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchUser = async () => {
      setLoading(true);
      setError(null);
      try {
        const res = await fetch('/api/user/123');
        const data = await res.json();
        setUser(data);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchUser();
  }, []);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return <div>{user?.name}</div>;
}
```

**Cách 2: Skeleton Loading (Better UX)**
```jsx
import Skeleton from 'react-loading-skeleton';
import 'react-loading-skeleton/dist/skeleton.css';

export default function UserProfile() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    // Fetch logic...
  }, []);

  return (
    <div>
      {loading ? (
        <>
          <Skeleton width={200} height={30} />
          <Skeleton count={3} />
        </>
      ) : (
        <>
          <h1>{user?.name}</h1>
          <p>{user?.email}</p>
        </>
      )}
    </div>
  );
}
```

**Cách 3: Request cancellation (Cancel outdated requests)**
```jsx
export default function SearchUsers() {
  const abortControllerRef = useRef(null);

  const handleSearch = async (query) => {
    // Cancel request cũ
    abortControllerRef.current?.abort();
    abortControllerRef.current = new AbortController();

    try {
      const res = await fetch(`/api/search?q=${query}`, {
        signal: abortControllerRef.current.signal
      });
      const data = await res.json();
      setResults(data);
    } catch (err) {
      if (err.name !== 'AbortError') {
        console.error(err);
      }
    }
  };

  return <input onChange={(e) => handleSearch(e.target.value)} />;
}
```

---

### Tình huống 3: Form có 50 input fields, submit chậm

**Câu hỏi:**
"Form có 50 trường input, khi user type mỗi ký tự app lag. Làm sao optimize?"

**Vấn đề:**
- Mỗi keystroke trigger onChange → setState → re-render toàn bộ form
- 50 inputs × re-render = rất nặng

**Giải pháp:**

**Cách 1: useRef + Manual submit (No re-render on input)**
```jsx
export default function HugeForm() {
  const formRef = useRef(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    // Lấy data từ form DOM trực tiếp
    const formData = new FormData(formRef.current);
    const data = Object.fromEntries(formData);
    
    await submitForm(data);
  };

  return (
    <form ref={formRef} onSubmit={handleSubmit}>
      {/* 50 inputs */}
      {[...Array(50)].map((_, i) => (
        <input key={i} name={`field_${i}`} />
      ))}
      <button type="submit">Submit</button>
    </form>
  );
}
```

**Cách 2: useDeferredValue (Defer heavy updates)**
```jsx
import { useDeferredValue, useState } from 'react';

function SearchableForm({ items }) {
  const [searchTerm, setSearchTerm] = useState('');
  const deferredSearchTerm = useDeferredValue(searchTerm);

  const filteredItems = items.filter(item =>
    item.name.includes(deferredSearchTerm)
  );

  return (
    <>
      <input
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
        placeholder="Search..."
      />
      <LargeList items={filteredItems} />
    </>
  );
}
// ✅ Input responsive, filtering deferred
```

**Cách 3: React Hook Form (Efficient form management)**
```jsx
import { useForm } from 'react-hook-form';

export default function EfficientForm() {
  const { register, handleSubmit, watch } = useForm({
    defaultValues: {
      // 50 fields...
    }
  });

  const onSubmit = (data) => console.log(data);

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {[...Array(50)].map((_, i) => (
        <input key={i} {...register(`field_${i}`)} />
      ))}
      <button type="submit">Submit</button>
    </form>
  );
}
// ✅ Không re-render toàn form, chỉ subscribed fields
```

---

### Tình huống 4: Filter/Sort data, lập tức hiển thị results

**Câu hỏi:**
"User click filter button, list phải update ngay lập tức. Mà data có 50,000 items. Làm sao?"

**Vấn đề:**
- Filter 50,000 items = tính toán nặng
- Không thể block UI trong khi filter

**Giải pháp:**

**Cách 1: Web Worker (Run heavy computation off main thread)**
```javascript
// worker.js
self.onmessage = (event) => {
  const { items, filter } = event.data;
  
  const filtered = items.filter(item =>
    item.category === filter.category &&
    item.price >= filter.minPrice
  );

  self.postMessage(filtered);
};
```

```jsx
// Component
export default function ProductFilter() {
  const [results, setResults] = useState([]);
  const workerRef = useRef(null);

  useEffect(() => {
    workerRef.current = new Worker('/filter-worker.js');
    workerRef.current.onmessage = (e) => {
      setResults(e.data);
    };

    return () => workerRef.current.terminate();
  }, []);

  const handleFilter = (filterCriteria) => {
    workerRef.current.postMessage({
      items: largeDataset,
      filter: filterCriteria
    });
  };

  return (
    <>
      <button onClick={() => handleFilter({ category: 'electronics' })}>
        Electronics
      </button>
      {results.map(item => <div key={item.id}>{item.name}</div>)}
    </>
  );
}
```

**Cách 2: Debounce + Memoize results**
```jsx
import { useMemo, useState, useCallback } from 'react';
import { debounce } from 'lodash';

export default function SmartFilter() {
  const [filterCriteria, setFilterCriteria] = useState({});

  // Memoize filtered results
  const filteredItems = useMemo(() => {
    return items.filter(item =>
      Object.entries(filterCriteria).every(([key, value]) =>
        item[key] === value
      )
    );
  }, [filterCriteria]);

  // Debounce filter changes
  const handleFilterChange = useCallback(
    debounce((criteria) => setFilterCriteria(criteria), 300),
    []
  );

  return (
    <>
      <select onChange={(e) => handleFilterChange({ category: e.target.value })}>
        <option>All</option>
        <option>Electronics</option>
      </select>
      {filteredItems.map(item => <div key={item.id}>{item.name}</div>)}
    </>
  );
}
```

---

### Tình huống 5: User input text search, 5000 items, phải filter real-time

**Câu hỏi:**
"Autocomplete search: user type từng ký tự, phải search 5000 items real-time, không được lag. Làm sao?"

**Vấn đề:**
- Real-time search trên 5000 items rất nặng
- Mỗi keystroke = 1 search operation

**Giải pháp:**

**Cách 1: Debounce + useMemo**
```jsx
import { debounce } from 'lodash';
import { useMemo, useState, useCallback } from 'react';

export default function SearchAutocomplete() {
  const [searchTerm, setSearchTerm] = useState('');

  const results = useMemo(() => {
    if (!searchTerm) return [];
    
    return items
      .filter(item => 
        item.name.toLowerCase().includes(searchTerm.toLowerCase())
      )
      .slice(0, 10); // Limit results
  }, [searchTerm]);

  const handleSearch = useCallback(
    debounce((value) => setSearchTerm(value), 300),
    []
  );

  return (
    <>
      <input
        onChange={(e) => handleSearch(e.target.value)}
        placeholder="Search..."
      />
      <ul>
        {results.map(item => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </>
  );
}
```

**Cách 2: Trie Data Structure (Fastest)**
```javascript
class Trie {
  constructor() {
    this.root = {};
  }

  insert(word) {
    let node = this.root;
    for (let char of word.toLowerCase()) {
      if (!node[char]) node[char] = {};
      node = node[char];
    }
    node.isEnd = true;
  }

  search(prefix) {
    let node = this.root;
    for (let char of prefix.toLowerCase()) {
      if (!node[char]) return [];
      node = node[char];
    }
    return this.dfs(node, prefix);
  }

  dfs(node, prefix) {
    let results = [];
    for (let char in node) {
      if (char !== 'isEnd') {
        if (node[char].isEnd) results.push(prefix + char);
        results = results.concat(this.dfs(node[char], prefix + char));
      }
    }
    return results;
  }
}

// Usage
const trie = new Trie();
items.forEach(item => trie.insert(item.name));

const searchResults = trie.search('app'); // O(m) complexity, m = prefix length
```

**Cách 3: Server-side search**
```jsx
export default function ServerSearch() {
  const [searchTerm, setSearchTerm] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);

  const handleSearch = useCallback(
    debounce(async (query) => {
      if (!query) {
        setResults([]);
        return;
      }

      setLoading(true);
      try {
        const res = await fetch(`/api/search?q=${query}`);
        const data = await res.json();
        setResults(data);
      } finally {
        setLoading(false);
      }
    }, 300),
    []
  );

  return (
    <>
      <input
        value={searchTerm}
        onChange={(e) => {
          setSearchTerm(e.target.value);
          handleSearch(e.target.value);
        }}
      />
      {loading && <p>Searching...</p>}
      <ul>
        {results.map(item => <li key={item.id}>{item.name}</li>)}
      </ul>
    </>
  );
}
```

---

## ⚡ PHẦN 2: HIỆU NĂNG (PERFORMANCE)

### Vấn đề 1: Bundle size quá lớn

**Câu hỏi:**
"App bundle 500KB, load page mất 10 giây. Làm sao giảm?"

**Nguyên nhân:**
- Import thư viện không cần thiết
- Unused dependencies
- Code duplication
- Không tree-shaking

**Giải pháp:**

```bash
# 1. Analyze bundle
npm install -g webpack-bundle-analyzer
npm run build

# 2. Kiểm tra size của mỗi library
npx bundlesize

# 3. Identify unused packages
npx depcheck
```

```javascript
// webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin()
  ]
}
```

**Cách fix:**

**Option 1: Code splitting**
```jsx
// Trước: import toàn bộ
import AdminDashboard from './pages/AdminDashboard';

// Sau: lazy load
import { lazy, Suspense } from 'react';

const AdminDashboard = lazy(() => import('./pages/AdminDashboard'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <AdminDashboard />
    </Suspense>
  );
}
```

**Option 2: Dynamic imports**
```jsx
const modules = {
  utils: () => import('./utils'),
  helpers: () => import('./helpers'),
};

// Load only when needed
const utils = await modules.utils();
```

**Option 3: Tree-shaking**
```javascript
// ❌ Bad: Import default export (not tree-shakeable)
import lodash from 'lodash';

// ✅ Good: Import named export (tree-shakeable)
import { debounce, throttle } from 'lodash-es';
```

**Option 4: Replace heavy libraries**
```javascript
// ❌ lodash: 70KB
// ✅ lodash-es: tree-shakeable version

// ❌ moment.js: 67KB
// ✅ dayjs: 2KB

// ❌ axios: 13KB
// ✅ fetch API (built-in): 0KB
```

---

### Vấn đề 2: Image loading chậm

**Câu hỏi:**
"Website có 20 hình ảnh lớn (5MB mỗi cái), user phải chờ lâu. Làm sao?"

**Vấn đề:**
- Download images nặng lâu
- Không lazy load
- Không optimize format

**Giải pháp:**

```jsx
// Cách 1: Lazy load images
export default function ImageGallery() {
  return (
    <div>
      {images.map(img => (
        <img
          key={img.id}
          src={img.src}
          loading="lazy"  // Native lazy loading
          alt={img.alt}
        />
      ))}
    </div>
  );
}
```

```jsx
// Cách 2: Intersection Observer (Advanced)
function LazyImage({ src, alt }) {
  const ref = useRef(null);
  const [imageSrc, setImageSrc] = useState(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      entries => {
        if (entries[0].isIntersecting) {
          setImageSrc(src);
          observer.unobserve(ref.current);
        }
      },
      { rootMargin: '50px' }
    );

    if (ref.current) observer.observe(ref.current);
    return () => observer.disconnect();
  }, [src]);

  return <img ref={ref} src={imageSrc} alt={alt} />;
}
```

```jsx
// Cách 3: Responsive images (WebP + fallback)
export default function OptimizedImage() {
  return (
    <picture>
      <source srcSet="image.webp" type="image/webp" />
      <source srcSet="image.jpg" type="image/jpeg" />
      <img src="image.jpg" alt="Example" />
    </picture>
  );
}
```

```jsx
// Cách 4: Image compression + CDN
// Dùng Next.js Image component
import Image from 'next/image';

export default function OptimizedImage() {
  return (
    <Image
      src="/photo.jpg"
      alt="Photo"
      width={600}
      height={400}
      priority={false}        // Lazy load
      quality={75}            // Compress
      placeholder="blur"      // Blur effect
    />
  );
}
```

---

### Vấn đề 3: API requests quá nhiều

**Câu hỏi:**
"Mỗi lần user click, app gửi 5 requests. Làm sao giảm?"

**Vấn đề:**
- N+1 queries
- Duplicate requests
- Không batch operations

**Giải pháp:**

```jsx
// Cách 1: Request debouncing + caching
class APIClient {
  constructor() {
    this.cache = new Map();
    this.pendingRequests = new Map();
  }

  async fetch(url) {
    // Kiểm tra cache
    if (this.cache.has(url)) {
      return this.cache.get(url);
    }

    // Kiểm tra pending request
    if (this.pendingRequests.has(url)) {
      return this.pendingRequests.get(url);
    }

    // Thực hiện request
    const promise = fetch(url).then(r => r.json());
    this.pendingRequests.set(url, promise);

    const result = await promise;
    this.cache.set(url, result);
    this.pendingRequests.delete(url);

    return result;
  }
}

const apiClient = new APIClient();
```

```jsx
// Cách 2: Batch requests (GraphQL)
const query = gql`
  query GetUserData($ids: [ID!]!) {
    users(ids: $ids) {
      id name email
    }
    posts(userIds: $ids) {
      id title
    }
  }
`;

// 1 request thay vì 2
const data = await client.query({ query, variables: { ids: [1, 2, 3] } });
```

```jsx
// Cách 3: SWR / React Query (Smart caching)
import useSWR from 'swr';

export default function UserList() {
  const { data: users } = useSWR('/api/users', fetcher, {
    revalidateOnFocus: false,
    dedupingInterval: 60000  // Dedupe requests trong 1 phút
  });

  return <div>{users?.map(u => <div key={u.id}>{u.name}</div>)}</div>;
}
```

---

### Vấn đề 4: Memory leak từ event listeners

**Câu hỏi:**
"App dùng lâu lâu thì bị lag, memory tăng từ 50MB → 300MB. Sao?"

**Vấn đề:**
- Event listeners không được remove
- Timers không clear
- Subscriptions không unsubscribe

**Giải pháp:**

```jsx
// ❌ Bad: Memory leak
function BadComponent() {
  useEffect(() => {
    window.addEventListener('resize', handleResize);
    // Quên remove → memory tăng!
  }, []);
}

// ✅ Good: Cleanup
function GoodComponent() {
  useEffect(() => {
    const handleResize = () => {
      console.log('Window resized');
    };

    window.addEventListener('resize', handleResize);

    // Cleanup
    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []);
}

// ✅ Better: Custom hook
function useWindowResize(callback) {
  useEffect(() => {
    window.addEventListener('resize', callback);
    return () => window.removeEventListener('resize', callback);
  }, [callback]);
}
```

```jsx
// Timers cleanup
function CountdownTimer() {
  useEffect(() => {
    const timer = setInterval(() => {
      // ...
    }, 1000);

    return () => clearInterval(timer); // ✅ Cleanup
  }, []);
}

// API abort
function FetchUser() {
  useEffect(() => {
    const controller = new AbortController();

    fetch('/api/user', { signal: controller.signal });

    return () => controller.abort(); // ✅ Cancel request
  }, []);
}
```

---

## 🔄 PHẦN 3: RE-RENDER ISSUES

### Vấn đề 1: Toàn bộ component re-render khi state thay đổi

**Câu hỏi:**
"Parent state thay đổi, toàn bộ children re-render. Làm sao tránh?"

**Vấn đề:**
```jsx
function Parent() {
  const [count, setCount] = useState(0);
  
  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      <HeavyChild /> {/* Re-render mỗi lần count thay đổi! */}
    </>
  );
}
```

**Giải pháp:**

**Cách 1: React.memo**
```jsx
const HeavyChild = React.memo(({ data }) => {
  console.log('HeavyChild rendered');
  return <div>{data}</div>;
});

// Chỉ re-render khi props thay đổi
```

**Cách 2: useMemo**
```jsx
function Parent() {
  const [count, setCount] = useState(0);

  const memoizedChild = useMemo(() => {
    return <HeavyChild />;
  }, []); // Không dependency, chỉ create 1 lần

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      {memoizedChild}
    </>
  );
}
```

**Cách 3: State lifting - tách state**
```jsx
function ParentWithInput() {
  return (
    <>
      <CounterSection />      {/* Tách riêng */}
      <HeavyChild />          {/* Không affected */}
    </>
  );
}

function CounterSection() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}
```

---

### Vấn đề 2: Callback function thay đổi mỗi render

**Câu hỏi:**
"Mỗi render, function callback tạo mới → child re-render. Sao?"

**Vấn đề:**
```jsx
function Parent() {
  const handleClick = () => {
    // Function này tạo mới mỗi render
    console.log('Clicked');
  };

  return <MemoizedChild onClick={handleClick} />; // Re-render vì callback mới
}

const MemoizedChild = React.memo(({ onClick }) => {
  console.log('Child rendered');
  return <button onClick={onClick}>Click me</button>;
});
```

**Giải pháp:**

```jsx
// Cách 1: useCallback
function Parent() {
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []); // Memoize callback

  return <MemoizedChild onClick={handleClick} />;
}

// Cách 2: useCallback with dependencies
function Parent({ userId }) {
  const handleClick = useCallback(() => {
    console.log(`User ${userId} clicked`);
  }, [userId]); // Chỉ tạo mới khi userId thay đổi

  return <MemoizedChild onClick={handleClick} />;
}
```

---

### Vấn đề 3: Object/Array props tạo mới mỗi render

**Câu hỏi:**
"Props là object/array, mỗi render tạo mới → child luôn re-render?"

**Vấn đề:**
```jsx
function Parent() {
  const config = { theme: 'dark' }; // Tạo mới mỗi render
  
  return <MemoizedChild config={config} />; // Luôn re-render
}
```

**Giải pháp:**

```jsx
// Cách 1: useMemo
function Parent() {
  const config = useMemo(() => ({ theme: 'dark' }), []);
  
  return <MemoizedChild config={config} />;
}

// Cách 2: Định nghĩa bên ngoài
const DEFAULT_CONFIG = { theme: 'dark' };

function Parent() {
  return <MemoizedChild config={DEFAULT_CONFIG} />;
}

// Cách 3: useCallback cho factories
function Parent() {
  const getConfig = useCallback(() => ({ theme: 'dark' }), []);
  
  return <MemoizedChild configFactory={getConfig} />;
}
```

---

### Vấn đề 4: Lớp component hierarchy, re-render quá sâu

**Câu hỏi:**
"Component A → B → C → D, A thay đổi, D render lại (cascade). Làm sao tối ưu?"

**Vấn đề:**
```jsx
function A() {
  const [state, setState] = useState(0);
  return <B state={state} />; // B re-render
}

function B({ state }) {
  return <C state={state} />; // C re-render
}

function C({ state }) {
  return <D state={state} />; // D re-render
}

// Cả 3 component re-render chỉ vì A thay đổi
```

**Giải pháp:**

**Cách 1: Context API (Skip intermediate components)**
```jsx
const StateContext = createContext();

function A() {
  const [state, setState] = useState(0);
  return (
    <StateContext.Provider value={state}>
      <B /> {/* Không pass props */}
    </StateContext.Provider>
  );
}

function B() {
  return <C />;
}

function C() {
  return <D />;
}

function D() {
  const state = useContext(StateContext); // Lấy trực tiếp
  return <div>{state}</div>;
}
```

**Cách 2: Compound Components**
```jsx
function Parent() {
  const [activeTab, setActiveTab] = useState(0);

  return (
    <Tabs>
      <Tabs.Header activeTab={activeTab} onChange={setActiveTab} />
      <Tabs.Content activeTab={activeTab} />
    </Tabs>
  );
}
```

**Cách 3: State management (Redux/Zustand)**
```jsx
import { useSelector } from 'react-redux';

function D() {
  const state = useSelector(state => state.app.value);
  return <div>{state}</div>;
}
// Subscribe trực tiếp, không qua intermediates
```

---

### Vấn đề 5: Unmounted component state update

**Câu hỏi:**
"Component unmount rồi, setState vẫn trigger warning?"

**Vấn đề:**
```jsx
function Component() {
  useEffect(() => {
    fetch('/api/data')
      .then(res => res.json())
      .then(data => {
        setState(data); // ❌ Warning nếu component unmounted
      });
  }, []);
}
```

**Giải pháp:**

```jsx
// Cách 1: Track mount status
function Component() {
  const isMountedRef = useRef(true);

  useEffect(() => {
    fetch('/api/data')
      .then(res => res.json())
      .then(data => {
        if (isMountedRef.current) {
          setState(data); // ✅ An toàn
        }
      });

    return () => {
      isMountedRef.current = false;
    };
  }, []);
}

// Cách 2: useEffect cleanup (AbortController)
function Component() {
  useEffect(() => {
    const controller = new AbortController();

    fetch('/api/data', { signal: controller.signal })
      .then(res => res.json())
      .then(data => setState(data));

    return () => controller.abort(); // ✅ Cancel request on unmount
  }, []);
}

// Cách 3: Custom hook
function useFetch(url) {
  const [data, setData] = useState(null);

  useEffect(() => {
    let cancelled = false;

    fetch(url)
      .then(res => res.json())
      .then(data => {
        if (!cancelled) setData(data);
      });

    return () => {
      cancelled = true;
    };
  }, [url]);

  return data;
}
```

---

## 👥 PHẦN 4: TRẢI NGHIỆM NGƯỜI DÙNG (UX) & ACCESSIBILITY

### Vấn đề 1: Form fields không accessible

**Câu hỏi:**
"Làm sao tạo form accessible cho screen reader users?"

**Vấn đề:**
```jsx
// ❌ Bad
<input type="text" />
<input type="password" />

// Screen reader không biết field là gì
```

**Giải pháp:**

```jsx
// ✅ Good: Proper labels
<label htmlFor="email">Email</label>
<input id="email" type="email" required />

<label htmlFor="password">Password</label>
<input id="password" type="password" required />

// ✅ Better: Aria attributes
<input
  id="email"
  type="email"
  aria-label="Email address"
  aria-describedby="email-hint"
  required
/>
<p id="email-hint">Must be a valid email address</p>

// ✅ Form group
<fieldset>
  <legend>Choose your preferred contact method</legend>
  <label>
    <input type="radio" name="contact" value="email" />
    Email
  </label>
  <label>
    <input type="radio" name="contact" value="phone" />
    Phone
  </label>
</fieldset>
```

---

### Vấn đề 2: Images không có alt text

**Câu hỏi:**
"Làm sao làm images accessible?"

**Giải pháp:**

```jsx
// ❌ Bad
<img src="photo.jpg" />

// ✅ Good: Descriptive alt text
<img src="user-avatar.jpg" alt="Profile picture of John Doe" />

// ✅ For decorative images
<img src="decoration.png" alt="" aria-hidden="true" />

// ✅ For complex images
<img
  src="chart.png"
  alt="Monthly revenue chart showing 20% increase"
/>
```

---

### Vấn đề 3: Keyboard navigation không hoạt động

**Câu hỏi:**
"User dùng keyboard tab, không thể navigate. Sao?"

**Vấn đề:**
```jsx
// ❌ Bad: div hoạt động như button nhưng không accessible
<div onClick={handleClick} className="button">
  Click me
</div>
```

**Giải pháp:**

```jsx
// ✅ Good: Use semantic HTML
<button onClick={handleClick}>
  Click me
</button>

// ✅ For custom components with keyboard support
function CustomButton({ onClick, children }) {
  const handleKeyDown = (e) => {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault();
      onClick?.();
    }
  };

  return (
    <div
      role="button"
      tabIndex={0}
      onClick={onClick}
      onKeyDown={handleKeyDown}
      className="custom-button"
    >
      {children}
    </div>
  );
}

// ✅ Manage focus
function Modal({ isOpen, onClose }) {
  const closeButtonRef = useRef(null);

  useEffect(() => {
    if (isOpen) {
      closeButtonRef.current?.focus(); // Focus on modal open
    }
  }, [isOpen]);

  return isOpen ? (
    <div role="dialog" aria-modal="true">
      <button ref={closeButtonRef} onClick={onClose}>
        Close
      </button>
    </div>
  ) : null;
}
```

---

### Vấn đề 4: Color contrast quá thấp

**Câu hỏi:**
"Text màu xám nhạt, người dị sắc không đọc được. Làm sao?"

**Vấn đề:**
```jsx
// ❌ Bad: Low contrast
<p style={{ color: '#cccccc', background: '#ffffff' }}>
  This is hard to read
</p>
```

**Giải pháp:**

```jsx
// ✅ Good: High contrast (WCAG AA minimum 4.5:1)
<p style={{ color: '#333333', background: '#ffffff' }}>
  This is easy to read
</p>

// Check contrast with tools:
// WebAIM Contrast Checker
// axe DevTools
// WAVE Browser Extension
```

---

### Vấn đề 5: Loading state không rõ

**Câu hỏi:**
"User không biết app đang load hay error. Làm sao?"

**Giải pháp:**

```jsx
function DataFetcher() {
  const [state, setState] = useState('idle');
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetch = async () => {
      setState('loading');
      try {
        const res = await fetchData();
        setData(res);
        setState('success');
      } catch (err) {
        setError(err);
        setState('error');
      }
    };

    fetch();
  }, []);

  // ✅ Clear states
  if (state === 'loading') {
    return (
      <div role="status" aria-live="polite">
        Loading data...
      </div>
    );
  }

  if (state === 'error') {
    return (
      <div role="alert">
        <h2>Error</h2>
        <p>{error?.message}</p>
        <button onClick={() => window.location.reload()}>
          Try again
        </button>
      </div>
    );
  }

  if (state === 'success') {
    return <div>{data}</div>;
  }

  return null;
}
```

---

### Vấn đề 6: Mobile responsiveness

**Câu hỏi:**
"Layout bị lỗi trên mobile. Làm sao?"

**Giải pháp:**

```jsx
// ✅ Mobile-first approach
function ResponsiveLayout() {
  return (
    <div className="flex flex-col md:flex-row gap-4">
      {/* Mobile: column, Tablet+: row */}
      <aside className="w-full md:w-1/4">Sidebar</aside>
      <main className="w-full md:w-3/4">Content</main>
    </div>
  );
}

// ✅ Viewport meta tag (HTML head)
<meta name="viewport" content="width=device-width, initial-scale=1.0" />

// ✅ Touch-friendly sizes
button {
  min-height: 48px; /* Minimum tap target */
  min-width: 48px;
}

// ✅ Avoid horizontal scroll
img {
  max-width: 100%;
  height: auto;
}
```

---

## 📊 CHEAT SHEET NHANH

### Performance Checklist
```
□ Bundle size < 200KB
□ First contentful paint < 1.5s
□ Largest contentful paint < 2.5s
□ Cumulative layout shift < 0.1
□ Images lazy-loaded
□ Code splitting implemented
□ Unused dependencies removed
□ API requests batched/cached
```

### Re-render Optimization
```
□ React.memo cho heavy components
□ useCallback cho event handlers
□ useMemo cho expensive calculations
□ useContext instead of prop drilling
□ State lifted appropriately
```

### Accessibility (A11y)
```
□ Semantic HTML (button, input, etc)
□ Alt text for images
□ Labels for form inputs
□ Keyboard navigation working
□ Color contrast WCAG AA (4.5:1)
□ Focus indicators visible
□ ARIA labels where needed
```

### UX Best Practices
```
□ Loading states clear
□ Error messages helpful
□ Forms have good feedback
□ Mobile responsive
□ Touch targets 48x48px min
□ Fast response to user input
□ Graceful error handling
```

---

## 💡 TIPS PHỎNG VẤN

**Khi được hỏi scenario:**
1. Identify vấn đề thực tế
2. Propose multiple solutions (3 cách tốt hơn 1)
3. Trade-offs: complexity vs performance
4. Code examples cụ thể
5. Testing strategy

**Từ khóa "magic":**
- "Virtual scrolling"
- "Debouncing/throttling"
- "Memoization"
- "Code splitting"
- "Web Workers"
- "React.memo + useCallback"

**Tránh:**
- "Dùng library X thôi" (quá generic)
- "Chúng tôi không có performance issue" (unrealistic)
- Không mention trade-offs

---

**Good luck! 🚀**
