# Tổng hợp kiến thức Jest quan trọng (React, Express, NestJS)

## Mục lục
1. [Jest cơ bản](#1-jest-cơ-bản)
2. [Cấu hình Jest](#2-cấu-hình-jest)
3. [Matchers thường dùng](#3-matchers-thường-dùng)
4. [Mocking trong Jest](#4-mocking-trong-jest)
5. [Async testing](#5-async-testing)
6. [Testing với React (React Testing Library)](#6-testing-với-react-react-testing-library)
7. [Testing với Express (Supertest)](#7-testing-với-express-supertest)
8. [Testing với NestJS](#8-testing-với-nestjs)
9. [Coverage & CI](#9-coverage--ci)
10. [Best practices & lỗi thường gặp](#10-best-practices--lỗi-thường-gặp)
11. [Testing với Prisma](#11-testing-với-prisma)
12. [Testing Authentication](#12-testing-authentication)
13. [Testing Exception](#13-testing-exception)
14. [Testing Cache & Redis](#14-testing-cache--redis)
15. [Testing Logger & Time](#15-testing-logger--time)
16. [Testing UUID / External Libraries](#16-testing-uuid--external-libraries)

---

## 1. Jest cơ bản

Jest tổ chức test theo mô hình cây: `describe` là một nhóm test (không bắt buộc, nhưng giúp gom các test liên quan lại và tạo output dễ đọc), còn `test`/`it` là một trường hợp kiểm thử cụ thể. Các hook `beforeEach`/`afterEach`/`beforeAll`/`afterAll` dùng để chuẩn bị và dọn dẹp dữ liệu, tránh lặp code và tránh test này ảnh hưởng tới test khác.

### Cấu trúc test cơ bản

```js
describe('Calculator', () => {
  beforeAll(() => { /* chạy 1 lần trước tất cả test trong block */ });
  afterAll(() => { /* chạy 1 lần sau tất cả test */ });
  beforeEach(() => { /* chạy trước mỗi test */ });
  afterEach(() => { /* chạy sau mỗi test */ });

  test('cộng hai số', () => {
    expect(1 + 2).toBe(3);
  });

  it('trừ hai số', () => { // it là alias của test
    expect(5 - 2).toBe(3);
  });
});
```

**Giải thích:**
- `beforeAll`/`afterAll` phù hợp cho các thao tác tốn kém chỉ cần làm một lần (mở kết nối DB, khởi động server test).
- `beforeEach`/`afterEach` phù hợp để reset state, mock, dữ liệu giữa từng test — đảm bảo mỗi test độc lập, không phụ thuộc kết quả test trước.
- `test` và `it` hoàn toàn giống nhau về chức năng, chỉ khác cách đọc câu văn (`it('should ...')` đọc tự nhiên hơn theo tiếng Anh).

### Các biến thể test hữu ích

```js
test.skip('bỏ qua test này', () => {});
test.only('chỉ chạy test này trong file', () => {});
test.todo('sẽ viết sau');

test.each([
  [1, 1, 2],
  [2, 3, 5],
])('cộng %i + %i = %i', (a, b, expected) => {
  expect(a + b).toBe(expected);
});

describe.each([[1], [2]])('trường hợp %i', (n) => {
  test(`test ${n}`, () => {});
});
```

**Giải thích:**
- `test.skip`: hữu ích khi test đang bị lỗi tạm thời (do bug chưa fix) nhưng chưa muốn xóa, tránh làm đỏ toàn bộ suite.
- `test.only`: dùng khi đang debug một test cụ thể, chỉ muốn chạy nó để tiết kiệm thời gian — **nhớ xóa trước khi commit**, vì nó sẽ khiến các test khác trong file bị bỏ qua.
- `test.todo`: đánh dấu placeholder cho việc lên kế hoạch test, xuất hiện trong report như một nhắc nhở.
- `test.each`/`describe.each`: tránh lặp code khi cần chạy cùng một logic test với nhiều bộ dữ liệu đầu vào khác nhau (data-driven testing).

---

## 2. Cấu hình Jest

File cấu hình Jest quyết định môi trường chạy test (`node` hay `jsdom`), cách transform code (TypeScript, JSX), cách map alias đường dẫn, và các rule liên quan coverage. Cấu hình sai môi trường là nguyên nhân phổ biến nhất khiến test bị lỗi ngay từ bước setup.

### File `jest.config.js` (Node/Express/NestJS - TypeScript)

```js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  testRegex: '.*\\.spec\\.ts$',
  collectCoverageFrom: ['**/*.(t|j)s'],
  coverageDirectory: '../coverage',
  moduleFileExtensions: ['js', 'json', 'ts'],
  moduleNameMapper: {
    '^@app/(.*)$': '<rootDir>/src/$1', // path alias
  },
};
```

**Giải thích:**
- `preset: 'ts-jest'`: cho phép Jest hiểu và biên dịch trực tiếp file TypeScript mà không cần build trước.
- `testEnvironment: 'node'`: vì backend (Express/NestJS) không cần DOM, dùng môi trường Node thuần sẽ nhanh hơn `jsdom`.
- `testRegex`: quy định Jest chỉ nhận diện các file có đuôi `.spec.ts` là file test (chuẩn NestJS mặc định).
- `moduleNameMapper`: ánh xạ lại path alias (ví dụ `@app/...`) khớp với cấu hình `tsconfig.json`, nếu không Jest sẽ báo lỗi "Cannot find module".

### File `jest.config.js` (React)

```js
module.exports = {
  testEnvironment: 'jsdom', // bắt buộc để test DOM
  setupFilesAfterEach: ['<rootDir>/jest.setup.js'],
  moduleNameMapper: {
    '\\.(css|less|scss)$': 'identity-obj-proxy',
    '^@/(.*)$': '<rootDir>/src/$1',
  },
  transform: {
    '^.+\\.(js|jsx|ts|tsx)$': 'babel-jest',
  },
};
```

**Giải thích:**
- `testEnvironment: 'jsdom'`: giả lập trình duyệt (DOM, `window`, `document`) để có thể `render()` component và truy vấn phần tử như trên browser thật.
- `setupFilesAfterEach`: file chạy sau khi test framework được setup nhưng trước mỗi test file — nơi thường import `@testing-library/jest-dom` để có thêm các matcher như `toBeInTheDocument()`.
- `moduleNameMapper` cho CSS: vì Jest không hiểu cú pháp import CSS, cần "giả lập" nó bằng `identity-obj-proxy` để tránh lỗi parse.
- `transform`: chỉ định dùng Babel để biên dịch JSX/TSX sang JS thuần trước khi Jest chạy.

```js
import '@testing-library/jest-dom';
```

> Với Create React App / Vite, thường dùng `vitest` thay Jest, nhưng cấu hình khái niệm tương tự (test environment, setup file, alias...).

---

## 3. Matchers thường dùng

Matcher là các hàm đi kèm `expect()` để kiểm tra giá trị thực tế có đúng như mong đợi không. Chọn đúng matcher không chỉ giúp test đúng mà còn giúp thông báo lỗi rõ ràng, dễ debug hơn khi test fail.

```js
// Equality
expect(value).toBe(4);              // so sánh strict (===)
expect(obj).toEqual({ a: 1 });      // so sánh deep equal
expect(obj).toStrictEqual({ a: 1 }); // deep equal + check kiểu, undefined props

// Truthiness
expect(value).toBeNull();
expect(value).toBeUndefined();
expect(value).toBeDefined();
expect(value).toBeTruthy();
expect(value).toBeFalsy();

// Numbers
expect(value).toBeGreaterThan(3);
expect(value).toBeGreaterThanOrEqual(3);
expect(value).toBeLessThan(5);
expect(value).toBeCloseTo(0.3); // so sánh số thực

// Strings
expect('Hello World').toMatch(/World/);
expect('Hello').toContain('ell');

// Arrays / Iterables
expect(['a', 'b']).toContain('a');
expect(arr).toHaveLength(3);

// Objects
expect(obj).toHaveProperty('name', 'John');
expect(obj).toMatchObject({ name: 'John' }); // chỉ check các field chỉ định

// Exceptions
expect(() => fn()).toThrow();
expect(() => fn()).toThrow('error message');
expect(() => fn()).toThrow(CustomError);

// Snapshot
expect(component).toMatchSnapshot();
```

**Giải thích các điểm dễ nhầm:**
- `toBe` vs `toEqual`: `toBe` dùng `Object.is` (giống `===`), chỉ dùng cho primitive (số, chuỗi, boolean) hoặc so sánh reference của object. `toEqual` so sánh giá trị bên trong object/array một cách đệ quy — đây là matcher đúng khi so sánh object hoặc array.
- `toStrictEqual` khắt khe hơn `toEqual`: nó phân biệt `undefined` property có tồn tại hay không, và phân biệt kiểu instance (ví dụ `class A {}` khác `class B {}` dù cùng field).
- `toMatchObject`: rất hữu ích khi object thực tế có nhiều field nhưng bạn chỉ muốn assert một vài field quan trọng, không cần liệt kê toàn bộ.
- `expect(() => fn()).toThrow()`: **bắt buộc** phải bọc lời gọi hàm trong một arrow function, nếu gọi trực tiếp `fn()` thì exception sẽ ném ra trước khi `expect` kịp bắt, làm test crash thay vì fail rõ ràng.
- Snapshot (`toMatchSnapshot`): lưu lại "ảnh chụp" cấu trúc output (thường dùng cho component React) vào file `.snap`, lần chạy sau so sánh với snapshot cũ. Hữu ích để phát hiện thay đổi không mong muốn, nhưng dễ bị lạm dụng — nên dùng có chọn lọc.

---

## 4. Mocking trong Jest

Mocking là kỹ thuật thay thế một hàm/module/dependency thật bằng phiên bản giả lập có thể kiểm soát được, giúp test cô lập (isolate) đơn vị đang test khỏi các phụ thuộc bên ngoài (API, DB, thời gian, module khác) — làm test nhanh hơn, ổn định hơn, và không phụ thuộc môi trường thật.

### Mock function

```js
const mockFn = jest.fn();
mockFn('a', 1);

expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledTimes(1);
expect(mockFn).toHaveBeenCalledWith('a', 1);
expect(mockFn).toHaveBeenLastCalledWith('a', 1);

// Định nghĩa return value
mockFn.mockReturnValue(42);
mockFn.mockReturnValueOnce(1).mockReturnValueOnce(2);
mockFn.mockResolvedValue({ data: 'ok' });   // cho async
mockFn.mockRejectedValue(new Error('fail'));
mockFn.mockImplementation((x) => x * 2);
```

**Giải thích:**
- `jest.fn()` tạo ra một hàm giả, tự động ghi lại mọi lần gọi (arguments, số lần gọi, giá trị trả về) — dùng để assert hành vi gọi hàm (ai gọi, gọi với gì, gọi mấy lần) thay vì chỉ assert kết quả cuối.
- `mockReturnValueOnce` cho phép định nghĩa giá trị trả về khác nhau cho từng lần gọi liên tiếp — hữu ích khi test logic retry hoặc phân trang.
- `mockResolvedValue`/`mockRejectedValue` là cách viết tắt của `mockImplementation(() => Promise.resolve(...))`/`reject(...)`, dùng khi mock các hàm bất đồng bộ (gọi API, query DB).

### Mock module

```js
// Auto-mock toàn bộ module
jest.mock('./userService');

// Mock có implementation cụ thể
jest.mock('./userService', () => ({
  getUser: jest.fn(() => ({ id: 1, name: 'John' })),
}));

// Mock third-party module (axios ví dụ)
jest.mock('axios');
import axios from 'axios';
axios.get.mockResolvedValue({ data: { id: 1 } });
```

**Giải thích:**
- `jest.mock('./userService')` (không có factory) sẽ tự động thay mọi export của module bằng mock function rỗng — Jest tự "hoist" (đưa lệnh này lên đầu file) trước cả các `import`, nên vị trí đặt trong file không quan trọng.
- Khi truyền factory function (tham số thứ 2), bạn kiểm soát chính xác mock trả về gì — dùng khi cần mock có logic hoặc giá trị mặc định cụ thể.
- Mock `axios` là ví dụ điển hình khi test code gọi API bên ngoài: thay vì gọi mạng thật (chậm, không ổn định), ta giả lập response ngay trong test.

### Spy on method (giữ implementation gốc hoặc override)

```js
const spy = jest.spyOn(object, 'method');
const spy2 = jest.spyOn(object, 'method').mockImplementation(() => 'mocked');

// khôi phục lại bản gốc
spy.mockRestore();
```

**Giải thích:**
- `jest.spyOn` khác `jest.fn()` ở chỗ nó "bọc" quanh hàm thật đang tồn tại trên object, mặc định vẫn gọi implementation gốc — chỉ dùng để theo dõi (spy) việc hàm có được gọi hay không.
- Nếu muốn vừa theo dõi vừa thay đổi hành vi, gọi thêm `.mockImplementation(...)` sau `spyOn`.
- `mockRestore()` trả object về trạng thái ban đầu (bỏ spy) — quan trọng khi spy trên các hàm global như `console.error`, `Date.now()` để tránh ảnh hưởng các test khác.

### Mock timers

```js
jest.useFakeTimers();
setTimeout(() => callback(), 1000);
jest.advanceTimersByTime(1000);
expect(callback).toHaveBeenCalled();
jest.useRealTimers();
```

**Giải thích:**
- Khi code có `setTimeout`/`setInterval`/`Date`, test thật sự chờ 1000ms sẽ làm suite chạy chậm. `jest.useFakeTimers()` thay thế các timer API bằng bản giả, cho phép "tua nhanh" thời gian bằng `advanceTimersByTime` mà không cần chờ thật.
- Luôn gọi `jest.useRealTimers()` sau khi dùng xong (thường trong `afterEach`) để không ảnh hưởng các test khác không liên quan đến timer.

### Reset / Clear mock giữa các test

```js
afterEach(() => {
  jest.clearAllMocks();  // xóa lịch sử gọi (mock.calls)
  jest.resetAllMocks();  // clear + xóa cả implementation
  jest.restoreAllMocks(); // restore về hàm gốc (đối với spyOn)
});
```

**Giải thích sự khác biệt:**
- `clearAllMocks`: chỉ xóa lịch sử gọi (`mock.calls`, `mock.results`), giữ nguyên implementation đã mock — dùng khi muốn mock logic vẫn giữ nguyên qua các test nhưng cần đếm lại số lần gọi từ đầu.
- `resetAllMocks`: xóa cả lịch sử lẫn implementation, mock function trở về trạng thái rỗng (`jest.fn()`) — an toàn nhất để tránh test trước ảnh hưởng test sau.
- `restoreAllMocks`: chỉ có tác dụng với các mock tạo bằng `jest.spyOn`, đưa hàm về đúng bản gốc (không còn là mock nữa).

> Nên đặt `clearMocks: true` trong `jest.config.js` để tự động clear mock giữa mỗi test, tránh leak state mà không cần viết `afterEach` thủ công ở từng file.

---

## 5. Async testing

Phần lớn code thực tế (gọi API, query DB, đọc file) đều bất đồng bộ. Nếu không xử lý đúng cách, Jest có thể kết thúc test **trước khi** promise resolve, khiến test luôn pass giả (false positive) dù logic bên trong sai.

```js
// Promise
test('async với return promise', () => {
  return fetchData().then((data) => {
    expect(data).toBe('value');
  });
});

// async/await (khuyến nghị)
test('async với await', async () => {
  const data = await fetchData();
  expect(data).toBe('value');
});

// resolves / rejects
test('resolves matcher', async () => {
  await expect(fetchData()).resolves.toBe('value');
});

test('rejects matcher', async () => {
  await expect(fetchDataFail()).rejects.toThrow('error');
});

// callback style
test('callback done', (done) => {
  fetchData((data) => {
    expect(data).toBe('value');
    done();
  });
});
```

**Giải thích:**
- Cách `return fetchData().then(...)` bắt buộc phải `return` promise, nếu quên `return`, Jest không biết phải chờ promise và test sẽ kết thúc (pass) trước khi assertion chạy.
- `async/await` là cách viết rõ ràng, dễ đọc nhất và được khuyến nghị dùng mặc định.
- `resolves`/`rejects` là cú pháp gọn giúp assert trực tiếp trên kết quả của promise mà không cần `await` giá trị ra biến trước — lưu ý vẫn phải `await` chính câu `expect(...)`.
- Kiểu `done` callback dùng cho code không dùng promise (API kiểu callback cũ) — nếu quên gọi `done()`, test sẽ timeout và fail với lỗi "Exceeded timeout".

---

## 6. Testing với React (React Testing Library)

Triết lý của React Testing Library (RTL) là **test theo cách người dùng thật sự tương tác** với ứng dụng (click, gõ chữ, đọc nội dung hiển thị), thay vì test chi tiết implementation nội bộ (state, props). Điều này khiến test ít bị vỡ khi refactor code miễn hành vi bên ngoài không đổi.

### Cài đặt
```bash
npm install --save-dev jest @testing-library/react @testing-library/jest-dom @testing-library/user-event jest-environment-jsdom
```

### Test component cơ bản

```jsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import Counter from './Counter';

test('render và tăng số đếm khi click', async () => {
  const user = userEvent.setup();
  render(<Counter />);

  expect(screen.getByText('Count: 0')).toBeInTheDocument();

  await user.click(screen.getByRole('button', { name: /increment/i }));

  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

**Giải thích:**
- `render()` "mount" component vào một DOM ảo (jsdom) để có thể truy vấn và tương tác.
- `screen` là object chứa các hàm query để tìm phần tử trong DOM đã render — không cần lưu kết quả `render()` ra biến để query.
- `userEvent` mô phỏng hành vi người dùng thật (click, gõ phím) sát thực tế hơn `fireEvent`, vì nó bắn ra đầy đủ chuỗi sự kiện DOM (mousedown → mouseup → click...) giống trình duyệt thật. Luôn `await` các thao tác `userEvent` vì chúng bất đồng bộ.

### Query ưu tiên (theo thứ tự khuyến nghị của Testing Library)

```js
// 1. Accessible bằng mọi người (ưu tiên nhất)
screen.getByRole('button', { name: /submit/i });
screen.getByLabelText('Email');
screen.getByPlaceholderText('Enter email');
screen.getByText('Xin chào');

// 2. Semantic queries
screen.getByAltText('logo');
screen.getByTitle('close');

// 3. Test IDs (chỉ dùng khi không còn cách nào khác)
screen.getByTestId('custom-element');
```

**Giải thích:**
- Thứ tự ưu tiên này phản ánh cách người dùng thật (kể cả người dùng công nghệ hỗ trợ như screen reader) tìm và tương tác với phần tử: qua **role** (button, textbox...) và **label**, không phải qua `id` hay `class` kỹ thuật.
- `getByTestId` là "lối thoát cuối cùng" — nếu dùng quá nhiều nghĩa là component có thể thiếu tính accessible (thiếu role/label rõ ràng), nên cân nhắc sửa component trước khi thêm `data-testid`.

```js
const item = await screen.findByText('Loaded data');
expect(screen.queryByText('Error')).not.toBeInTheDocument();
```

**Giải thích 3 nhóm hàm query:**
- `getBy*`: throw lỗi ngay nếu không tìm thấy — dùng khi chắc chắn phần tử phải tồn tại tại thời điểm gọi.
- `queryBy*`: trả về `null` thay vì throw nếu không tìm thấy — dùng riêng cho việc assert một phần tử **không tồn tại** (`getBy*` sẽ throw trước khi kịp assert `not.toBeInTheDocument()`).
- `findBy*`: trả về Promise, tự động retry trong vài giây — dùng khi phần tử xuất hiện **sau một khoảng trễ** (ví dụ sau khi fetch API xong), tránh phải dùng `setTimeout` thủ công.

### Test form / input

```jsx
test('nhập và submit form', async () => {
  const handleSubmit = jest.fn();
  const user = userEvent.setup();
  render(<LoginForm onSubmit={handleSubmit} />);

  await user.type(screen.getByLabelText('Email'), 'test@example.com');
  await user.type(screen.getByLabelText('Password'), '123456');
  await user.click(screen.getByRole('button', { name: /login/i }));

  expect(handleSubmit).toHaveBeenCalledWith({
    email: 'test@example.com',
    password: '123456',
  });
});
```

**Giải thích:** đây là ví dụ điển hình test "hành vi" thay vì "implementation" — ta không quan tâm component quản lý state input bằng `useState` hay thư viện form nào, chỉ quan tâm: người dùng gõ đúng dữ liệu, bấm nút, thì callback `onSubmit` có được gọi với đúng dữ liệu hay không.

### Mock API call trong component (dùng MSW - khuyến nghị hơn jest.mock cho fetch)

```js
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

const server = setupServer(
  http.get('/api/users', () => {
    return HttpResponse.json([{ id: 1, name: 'John' }]);
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('hiển thị danh sách user từ API', async () => {
  render(<UserList />);
  expect(await screen.findByText('John')).toBeInTheDocument();
});
```

**Giải thích:** MSW (Mock Service Worker) chặn request ở tầng network (interceptor), thay vì mock trực tiếp module `fetch`/`axios`. Ưu điểm: code component không cần biết gì về việc đang bị test — gọi `fetch('/api/users')` y hệt lúc chạy thật, giúp test sát với hành vi production hơn so với mock thủ công từng hàm.

### Test custom hook

```js
import { renderHook, act } from '@testing-library/react';
import useCounter from './useCounter';

test('useCounter tăng giá trị', () => {
  const { result } = renderHook(() => useCounter());

  act(() => {
    result.current.increment();
  });

  expect(result.current.count).toBe(1);
});
```

**Giải thích:** hook không thể gọi trực tiếp ngoài component (vi phạm Rules of Hooks), nên `renderHook` tạo một component ẩn để chạy hook đó. `result.current` luôn trỏ tới giá trị mới nhất mà hook trả về. `act()` đảm bảo mọi cập nhật state bên trong đã được React xử lý xong (flush) trước khi assert.

### Test Context / Provider

```jsx
function renderWithProvider(ui, { theme = 'light' } = {}) {
  return render(<ThemeContext.Provider value={theme}>{ui}</ThemeContext.Provider>);
}

test('component đọc theme từ context', () => {
  renderWithProvider(<ThemedButton />, { theme: 'dark' });
  expect(screen.getByRole('button')).toHaveClass('dark');
});
```

**Giải thích:** khi component phụ thuộc Context (theme, auth, i18n...), cần bọc nó trong `Provider` tương ứng khi test, nếu không sẽ nhận giá trị default của context (thường gây lỗi hoặc sai kết quả). Viết một hàm helper `renderWithProvider` giúp tái sử dụng logic này ở nhiều test file.

---

## 7. Testing với Express (Supertest)

Supertest cho phép gửi HTTP request giả tới một Express `app` ngay trong process test (không cần start server thật, không tốn cổng mạng), rồi assert trực tiếp trên response (status code, body, header).

### Cài đặt
```bash
npm install --save-dev jest supertest @types/jest @types/supertest
```

### Nguyên tắc: tách `app.js` khỏi `server.listen()`

```js
// app.js
const express = require('express');
const app = express();
app.use(express.json());

app.get('/users/:id', (req, res) => {
  res.json({ id: req.params.id, name: 'John' });
});

module.exports = app;
```

```js
// server.js
const app = require('./app');
app.listen(3000);
```

**Giải thích:** đây là nguyên tắc quan trọng nhất khi test Express — tách phần định nghĩa `app` (routes, middleware) ra khỏi phần `listen()` (khởi động cổng mạng thật). Supertest chỉ cần object `app`, không cần server đang chạy, nên nếu gộp chung `listen()` vào file `app.js`, mỗi lần import để test sẽ vô tình mở cổng thật, gây xung đột cổng và làm test không tự đóng được.

### Test API endpoint

```js
const request = require('supertest');
const app = require('./app');

describe('GET /users/:id', () => {
  test('trả về user theo id', async () => {
    const res = await request(app).get('/users/1');

    expect(res.status).toBe(200);
    expect(res.body).toEqual({ id: '1', name: 'John' });
  });

  test('trả về 404 khi không tìm thấy', async () => {
    const res = await request(app).get('/users/999');
    expect(res.status).toBe(404);
  });
});

describe('POST /users', () => {
  test('tạo user mới', async () => {
    const res = await request(app)
      .post('/users')
      .send({ name: 'Alice' })
      .set('Accept', 'application/json');

    expect(res.status).toBe(201);
    expect(res.body).toHaveProperty('id');
  });
});
```

**Giải thích:** `request(app).get(...)`/`.post(...)` build request y hệt như dùng Postman/curl nhưng chạy trong bộ nhớ (in-process), nên rất nhanh. Việc test cả trường hợp thành công (200/201) lẫn thất bại (404) là để đảm bảo route xử lý đúng ở cả "happy path" lẫn "edge case" — một lỗi phổ biến là chỉ test happy path, bỏ sót lỗi xử lý input không hợp lệ.

### Mock middleware / service layer (unit test route handler)

```js
jest.mock('./services/userService');
const userService = require('./services/userService');

test('trả lỗi 500 khi service throw', async () => {
  userService.getUser.mockRejectedValue(new Error('DB error'));

  const res = await request(app).get('/users/1');
  expect(res.status).toBe(500);
});
```

**Giải thích:** đây là kỹ thuật **unit test controller/route** một cách cô lập — mock hẳn tầng service (nơi thường chứa logic gọi DB) để không cần DB thật, chỉ tập trung test xem route handler có xử lý đúng lỗi từ service hay không (ví dụ trả về đúng status code 500 kèm message phù hợp).

### Test middleware auth (JWT)

```js
test('từ chối khi không có token', async () => {
  const res = await request(app).get('/protected');
  expect(res.status).toBe(401);
});

test('cho phép khi có token hợp lệ', async () => {
  const token = generateTestToken({ userId: 1 });
  const res = await request(app)
    .get('/protected')
    .set('Authorization', `Bearer ${token}`);
  expect(res.status).toBe(200);
});
```

**Giải thích:** với route có middleware xác thực, cần test cả 2 chiều: **thiếu quyền** (không token/token sai → phải bị chặn) và **đủ quyền** (token hợp lệ → phải được xử lý bình thường). Đây là loại test quan trọng về bảo mật, hay bị bỏ sót nếu chỉ tập trung test "chức năng chính".

### Test kết nối database (dùng DB test thật hoặc in-memory)

```js
beforeAll(async () => {
  await mongoose.connect(process.env.TEST_DB_URI);
});

afterEach(async () => {
  await User.deleteMany({}); // dọn dữ liệu sau mỗi test
});

afterAll(async () => {
  await mongoose.connection.close();
});
```

**Giải thích:** khi test cần chạm tới DB thật (integration test), nên dùng một DB **riêng cho môi trường test** (`TEST_DB_URI` khác DB dev/production) để tránh làm bẩn dữ liệu thật. `afterEach` xóa dữ liệu vừa tạo để mỗi test bắt đầu từ trạng thái sạch (tránh test sau bị ảnh hưởng bởi dữ liệu test trước để lại). `afterAll` đóng kết nối để process Jest có thể tự thoát, tránh treo test do connection còn mở.

---

## 8. Testing với NestJS

NestJS dùng Jest mặc định (đã cấu hình sẵn khi `nest new`). NestJS có 2 loại test chính, phục vụ 2 mục đích khác nhau:
- **Unit test** (`.spec.ts`): test từng class (Service, Controller, Pipe...) riêng lẻ, mock hết dependency — chạy rất nhanh, giúp định vị lỗi chính xác tới từng class.
- **E2E test** (`.e2e-spec.ts`): khởi động toàn bộ (hoặc gần toàn bộ) module thật, gửi HTTP request qua Supertest — kiểm tra các phần ráp lại với nhau có hoạt động đúng không.

Điểm mấu chốt khi test NestJS là tận dụng cơ chế **Dependency Injection (DI)**: dùng `TestingModule` để thay thế dependency thật bằng mock, mà không cần sửa code class đang test.

### Unit test cho Service

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersService } from './users.service';
import { getRepositoryToken } from '@nestjs/typeorm';
import { User } from './user.entity';

describe('UsersService', () => {
  let service: UsersService;
  let repo: jest.Mocked<Record<string, any>>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: getRepositoryToken(User),
          useValue: {
            find: jest.fn(),
            findOneBy: jest.fn(),
            save: jest.fn(),
            delete: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    repo = module.get(getRepositoryToken(User));
  });

  it('nên được định nghĩa', () => {
    expect(service).toBeDefined();
  });

  it('findAll trả về danh sách user', async () => {
    repo.find.mockResolvedValue([{ id: 1, name: 'John' }]);
    const result = await service.findAll();
    expect(result).toEqual([{ id: 1, name: 'John' }]);
    expect(repo.find).toHaveBeenCalled();
  });
});
```

**Giải thích:**
- `Test.createTestingModule({...}).compile()` dựng lên một Nest module thu nhỏ chỉ chứa provider cần cho test, mô phỏng cơ chế DI thật của Nest.
- `getRepositoryToken(User)`: TypeORM inject `Repository<User>` thông qua một token đặc biệt, không phải class thường — phải dùng đúng hàm này để "đánh lừa" Nest cung cấp mock repository thay vì repository thật kết nối DB.
- `useValue`: cung cấp trực tiếp một object giả (chứa các method là `jest.fn()`) thay cho dependency thật — đây là mock đơn giản nhất, phù hợp khi không cần logic phức tạp.
- Test `'nên được định nghĩa'` (`toBeDefined()`) là test tối thiểu để đảm bảo module test compile đúng, DI hoạt động — nên có ở đầu mỗi file spec.

### Unit test cho Controller (mock Service qua DI)

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

describe('UsersController', () => {
  let controller: UsersController;
  let service: UsersService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [
        {
          provide: UsersService,
          useValue: {
            findOne: jest.fn().mockResolvedValue({ id: 1, name: 'John' }),
            create: jest.fn(),
          },
        },
      ],
    }).compile();

    controller = module.get<UsersController>(UsersController);
    service = module.get<UsersService>(UsersService);
  });

  it('getUser trả về user đúng', async () => {
    const result = await controller.getUser('1');
    expect(result).toEqual({ id: 1, name: 'John' });
    expect(service.findOne).toHaveBeenCalledWith('1');
  });
});
```

**Giải thích:** khác với test Service (nơi test logic nghiệp vụ thật), khi unit test Controller ta **mock toàn bộ Service** — vì Controller không nên chứa logic nghiệp vụ, chỉ có nhiệm vụ nhận request và gọi đúng service method. Test này chỉ cần xác nhận: controller gọi đúng method của service với đúng tham số, và trả về đúng những gì service trả về.

### Mock Guard / Pipe / Interceptor khi test Controller

```ts
const module: TestingModule = await Test.createTestingModule({
  controllers: [UsersController],
  providers: [{ provide: UsersService, useValue: mockUsersService }],
})
  .overrideGuard(AuthGuard)
  .useValue({ canActivate: () => true }) // bypass guard trong unit test
  .compile();
```

**Giải thích:** khi Controller có gắn `@UseGuards(AuthGuard)`, nếu không override, Nest sẽ chạy guard thật (có thể cần request thật, JWT thật...) làm unit test phức tạp không cần thiết. `.overrideGuard(...).useValue({ canActivate: () => true })` thay guard bằng phiên bản luôn cho phép đi qua, giúp unit test tập trung vào logic controller, còn việc test guard sẽ được test riêng ở phần dưới hoặc ở e2e test.

### Test Pipe / Custom Decorator riêng lẻ

```ts
import { ValidationPipe, BadRequestException } from '@nestjs/common';

describe('ValidationPipe', () => {
  it('throw BadRequestException khi dữ liệu không hợp lệ', async () => {
    const pipe = new ValidationPipe();
    await expect(
      pipe.transform({ email: 'not-an-email' }, {
        type: 'body',
        metatype: CreateUserDto,
      })
    ).rejects.toThrow(BadRequestException);
  });
});
```

**Giải thích:** Pipe, Guard, Interceptor trong Nest thường là các class thuần (POJO class implement interface), có thể khởi tạo trực tiếp bằng `new` và gọi method (`transform`, `canActivate`...) mà **không cần dựng cả TestingModule** — đây là cách test nhanh nhất, phù hợp khi logic của chúng không phụ thuộc DI phức tạp.

### E2E test (test toàn bộ app + HTTP thật qua supertest)

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('UsersController (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  it('/users (GET)', () => {
    return request(app.getHttpServer())
      .get('/users')
      .expect(200)
      .expect((res) => {
        expect(Array.isArray(res.body)).toBe(true);
      });
  });

  it('/users (POST) - validation fail', () => {
    return request(app.getHttpServer())
      .post('/users')
      .send({ email: 'invalid' })
      .expect(400);
  });
});
```

**Giải thích:** khác hẳn unit test, e2e test import nguyên `AppModule` thật (toàn bộ dependency thật, trừ khi override) rồi tạo một `INestApplication` để lấy HTTP server nội bộ (`getHttpServer()`), sau đó dùng Supertest gửi request y như test Express. Mục tiêu là xác nhận toàn bộ pipeline (routing → guard → pipe validate → controller → service → response) hoạt động đúng khi ráp lại với nhau — điều mà unit test riêng lẻ không phát hiện được (ví dụ lỗi wiring module, thiếu provider).

### Test Repository/DB thật bằng module Testing riêng (Integration test)

```ts
const module: TestingModule = await Test.createTestingModule({
  imports: [
    TypeOrmModule.forRoot({
      type: 'sqlite',
      database: ':memory:', // DB in-memory cho test nhanh, độc lập
      entities: [User],
      synchronize: true,
    }),
    TypeOrmModule.forFeature([User]),
  ],
  providers: [UsersService],
}).compile();
```

**Giải thích:** đây là "integration test" — nằm giữa unit test (toàn mock) và e2e test (toàn thật): dùng SQLite `:memory:` để có một DB thật (SQL thật chạy) nhưng nhanh và tự hủy sau khi test xong, không cần cài đặt DB server riêng. `synchronize: true` tự tạo schema từ entity, tiện cho test nhưng **không nên dùng trong production** vì có thể làm mất dữ liệu khi entity thay đổi.

### Test Exception Filter

```ts
import { ArgumentsHost } from '@nestjs/common';
import { HttpExceptionFilter } from './http-exception.filter';

it('format lỗi đúng chuẩn', () => {
  const filter = new HttpExceptionFilter();
  const mockJson = jest.fn();
  const mockStatus = jest.fn().mockReturnValue({ json: mockJson });
  const mockHost = {
    switchToHttp: () => ({
      getResponse: () => ({ status: mockStatus }),
      getRequest: () => ({ url: '/test' }),
    }),
  } as unknown as ArgumentsHost;

  filter.catch(new BadRequestException('Lỗi'), mockHost);

  expect(mockStatus).toHaveBeenCalledWith(400);
  expect(mockJson).toHaveBeenCalledWith(
    expect.objectContaining({ message: 'Lỗi' })
  );
});
```

**Giải thích:** `ArgumentsHost` là object Nest cung cấp thật sự khi có exception thật, nên khi unit test filter một cách cô lập, ta phải tự tạo một "fake host" giả lập đúng shape mà `switchToHttp().getResponse()`/`getRequest()` trả về, để có thể gọi `filter.catch(...)` trực tiếp mà không cần bật cả HTTP server. `expect.objectContaining(...)` cho phép assert object response chỉ cần chứa field `message` đúng, không quan tâm các field khác.

---

## 9. Coverage & CI

Coverage đo tỷ lệ phần trăm code được "chạm" tới khi chạy test (theo dòng, nhánh if/else, function, statement). Đây là chỉ số hữu ích để phát hiện code chưa được test, nhưng **coverage cao không đồng nghĩa test tốt** — code có thể được "chạm" tới mà không có assertion ý nghĩa nào.

### Chạy coverage

```bash
jest --coverage
```

**Giải thích:** lệnh này sinh ra report (mặc định ở thư mục `coverage/`) gồm cả bản HTML trực quan (`coverage/lcov-report/index.html`) để xem chi tiết dòng nào chưa được test.

### Cấu hình threshold (fail build nếu coverage thấp)

```js
// jest.config.js
module.exports = {
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

**Giải thích:** khi đặt threshold, lệnh `jest --coverage` sẽ **fail (exit code khác 0)** nếu coverage thực tế thấp hơn ngưỡng quy định — dùng để chặn merge code (qua CI) khi coverage bị giảm sút, giữ chất lượng test ổn định theo thời gian. Có thể đặt threshold riêng cho từng thư mục/file (không chỉ `global`) nếu muốn siết chặt hơn ở các module quan trọng.

### Script package.json phổ biến

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "test:e2e": "jest --config ./test/jest-e2e.json",
    "test:ci": "jest --ci --coverage --maxWorkers=2"
  }
}
```

**Giải thích:**
- `test:watch`: chạy Jest ở chế độ theo dõi file thay đổi, chỉ chạy lại các test liên quan đến file vừa sửa — dùng khi code hàng ngày (dev loop nhanh).
- `test:e2e`: dùng file config riêng vì e2e test thường cần `testEnvironment`, timeout, hoặc setup khác với unit test.
- `--ci`: tắt các tính năng tương tác — ví dụ khi gặp snapshot mới (chưa từng có), thay vì tự động tạo snapshot mới (hành vi mặc định khi dev local), nó sẽ báo fail để tránh CI "âm thầm" chấp nhận snapshot sai.
- `--maxWorkers=2`: giới hạn số tiến trình worker chạy song song, tránh CI runner (thường có ít CPU/RAM hơn máy dev) bị quá tải hoặc bị kill do hết bộ nhớ.

---

## 10. Best practices & lỗi thường gặp

### Best practices

- **AAA pattern**: Arrange (chuẩn bị dữ liệu/mock) – Act (thực hiện hành động cần test) – Assert (kiểm tra kết quả), giữ mỗi test rõ ràng 3 phần giúp người đọc dễ theo dõi logic test mà không cần đọc code implementation.
- **Test hành vi, không test implementation**: assert output/DOM/response, không assert biến nội bộ hay việc gọi đúng hàm private nào — vì implementation có thể đổi khi refactor dù hành vi bên ngoài không đổi; test bám implementation sẽ vỡ dù code vẫn đúng.
- **Một test chỉ nên assert một hành vi logic** (có thể nhiều `expect` nhưng cùng một ý) — giúp khi test fail, biết ngay chính xác hành vi nào bị sai, không phải dò qua nhiều logic trộn lẫn.
- **Đặt tên test mô tả rõ**: `it('nên trả về 404 khi user không tồn tại')` thay vì `it('test 1')` — tên test chính là tài liệu sống, giúp người khác (và chính bạn sau này) hiểu ngay ý định mà không cần đọc code bên trong.
- **Dùng `beforeEach` để reset state**, tránh test phụ thuộc thứ tự chạy — mỗi test nên chạy độc lập, đảo thứ tự chạy vẫn phải cho kết quả đúng.
- **Ưu tiên test qua public API** (route, hook, component render) thay vì gọi thẳng hàm private — giữ test ổn định qua các lần refactor nội bộ.
- **Với React**: ưu tiên `getByRole` để test theo cách người dùng thật sự tương tác (accessibility-first) — đồng thời gián tiếp kiểm tra tính accessible của UI.
- **Với Express/NestJS**: tách rõ unit test (mock dependency, chạy cực nhanh, chạy nhiều lần trong lúc code) và e2e/integration test (DB/HTTP thật, chạy ít hơn, thường ở CI hoặc trước khi merge) — cân bằng giữa tốc độ phản hồi và độ tin cậy.
- **Isolate side-effects**: mock network, timers, `Date.now()` để test deterministic (kết quả không đổi giữa các lần chạy) — test không nên phụ thuộc vào thời điểm chạy thật hay kết nối mạng thật.

### Lỗi thường gặp

| Lỗi | Nguyên nhân | Cách fix |
|---|---|---|
| `ReferenceError: fetch is not defined` (Node) | Node < 18 không có `fetch` global | Dùng `node-fetch` hoặc mock, hoặc nâng Node version |
| Test React bị "not wrapped in act(...)" | Cập nhật state async chưa được đợi | Dùng `await` với `findBy*`, hoặc `await waitFor(...)` |
| Mock không hoạt động dù đã `jest.mock` | Đường dẫn `jest.mock` sai, hoặc mock đặt sau `require`/`import` | `jest.mock` luôn được hoist lên đầu file tự động, kiểm tra path chính xác |
| Test bị leak giữa các file (state cũ) | Không dùng `clearAllMocks`/đóng kết nối DB | Thêm `afterEach(() => jest.clearAllMocks())`, đóng DB ở `afterAll` |
| Test NestJS e2e treo mãi không kết thúc | Quên `await app.close()` hoặc DB connection còn mở | Đóng app/connection trong `afterAll` |
| `toEqual` fail dù nhìn giống nhau | Khác kiểu dữ liệu (string vs number) hoặc field `undefined` thừa | Dùng `toMatchObject` nếu chỉ cần check 1 phần, hoặc kiểm tra kỹ kiểu dữ liệu |
| Snapshot test fail liên tục khi refactor nhỏ | Snapshot quá lớn, nhạy với thay đổi không quan trọng | Dùng snapshot có chọn lọc (`toMatchSnapshot` trên phần nhỏ), tránh snapshot toàn bộ component phức tạp |

---

## 11. Testing với Prisma

Prisma tạo ra một `PrismaClient` sinh code tự động, kết nối DB thật. Khi unit test Service dùng Prisma (thường thông qua một `PrismaService` wrapper trong NestJS), ta **không** muốn chạm DB thật — thay vào đó mock hẳn `PrismaService`, giả lập các phương thức CRUD trả về dữ liệu mong muốn để test riêng logic nghiệp vụ của Service.

### Mock `PrismaService`

```ts
// prisma.service.ts (thường extend PrismaClient để inject qua DI)
@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect();
  }
}
```

```ts
// users.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersService } from './users.service';
import { PrismaService } from '../prisma/prisma.service';

const mockPrismaService = {
  user: {
    findUnique: jest.fn(),
    findMany: jest.fn(),
    create: jest.fn(),
    update: jest.fn(),
    delete: jest.fn(),
  },
  $transaction: jest.fn(),
};

describe('UsersService', () => {
  let service: UsersService;
  let prisma: typeof mockPrismaService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        { provide: PrismaService, useValue: mockPrismaService },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    prisma = module.get(PrismaService);
    jest.clearAllMocks();
  });

  it('nên được định nghĩa', () => {
    expect(service).toBeDefined();
  });
});
```

**Giải thích:**
- Vì `PrismaService` được inject qua DI như một provider bình thường (khác với `Repository` của TypeORM không cần token đặc biệt), chỉ cần `provide: PrismaService, useValue: mockPrismaService` là đủ để thay thế toàn bộ client thật.
- Mock object được thiết kế theo đúng cấu trúc API thật của Prisma: mỗi model (`user`, `post`...) là một object con chứa các method CRUD riêng — giữ đúng shape này giúp code Service không cần sửa gì khi chạy test.
- Gọi `jest.clearAllMocks()` trong `beforeEach` để đảm bảo mock trả về/lịch sử gọi của test trước không rò rỉ sang test sau (đặc biệt quan trọng với Prisma vì rất nhiều method dùng chung một object mock).

### Mock CRUD (`findUnique`, `findMany`, `create`, `update`, `delete`)

```ts
describe('UsersService CRUD', () => {
  it('findOne gọi đúng findUnique với where id', async () => {
    prisma.user.findUnique.mockResolvedValue({ id: 1, name: 'John' });

    const result = await service.findOne(1);

    expect(result).toEqual({ id: 1, name: 'John' });
    expect(prisma.user.findUnique).toHaveBeenCalledWith({ where: { id: 1 } });
  });

  it('findOne trả null khi không tìm thấy -> Service ném NotFoundException', async () => {
    prisma.user.findUnique.mockResolvedValue(null);

    await expect(service.findOne(999)).rejects.toThrow(NotFoundException);
  });

  it('findAll trả về danh sách', async () => {
    prisma.user.findMany.mockResolvedValue([{ id: 1 }, { id: 2 }]);

    const result = await service.findAll();

    expect(result).toHaveLength(2);
  });

  it('create gọi đúng dữ liệu đầu vào', async () => {
    const dto = { name: 'Alice', email: 'alice@test.com' };
    prisma.user.create.mockResolvedValue({ id: 3, ...dto });

    const result = await service.create(dto);

    expect(prisma.user.create).toHaveBeenCalledWith({ data: dto });
    expect(result.id).toBe(3);
  });

  it('update gọi đúng where + data', async () => {
    prisma.user.update.mockResolvedValue({ id: 1, name: 'Updated' });

    await service.update(1, { name: 'Updated' });

    expect(prisma.user.update).toHaveBeenCalledWith({
      where: { id: 1 },
      data: { name: 'Updated' },
    });
  });

  it('delete gọi đúng where id', async () => {
    prisma.user.delete.mockResolvedValue({ id: 1 });

    await service.remove(1);

    expect(prisma.user.delete).toHaveBeenCalledWith({ where: { id: 1 } });
  });
});
```

**Giải thích:**
- Nguyên tắc chung: mock giá trị trả về của Prisma method (`mockResolvedValue`), sau đó assert **hai điều**: (1) Service trả về đúng kết quả mong đợi, và (2) Prisma method được gọi với đúng tham số (`where`, `data`) — vế thứ 2 rất quan trọng vì nó xác nhận Service build đúng query, không chỉ "trả về đúng vì mock nói vậy".
- Trường hợp `findUnique` trả `null` là pattern rất hay gặp: Prisma không tự throw lỗi khi không tìm thấy record (khác với TypeORM `findOneOrFail`), nên Service thường phải tự kiểm tra và ném `NotFoundException` — đây là nhánh logic quan trọng cần test riêng.
- Với `create`/`update`, kiểm tra đối số gọi vào (`toHaveBeenCalledWith`) giúp phát hiện lỗi kiểu Service quên truyền field, hoặc gửi nhầm cấu trúc `data`.

### Mock `$transaction`

```ts
it('transferBalance dùng $transaction đúng cách', async () => {
  prisma.$transaction.mockImplementation(async (callback) => {
    // Giả lập Prisma gọi callback với chính prisma instance (dạng interactive transaction)
    return callback(prisma);
  });
  prisma.user.update.mockResolvedValueOnce({ id: 1, balance: 90 });
  prisma.user.update.mockResolvedValueOnce({ id: 2, balance: 110 });

  await service.transferBalance(1, 2, 10);

  expect(prisma.$transaction).toHaveBeenCalled();
  expect(prisma.user.update).toHaveBeenCalledTimes(2);
});

// Trường hợp $transaction dạng mảng promise: prisma.$transaction([p1, p2])
it('$transaction dạng mảng', async () => {
  prisma.$transaction.mockResolvedValue([{ id: 1 }, { id: 2 }]);

  const result = await service.batchCreate([{ name: 'A' }, { name: 'B' }]);

  expect(prisma.$transaction).toHaveBeenCalledWith(expect.any(Array));
  expect(result).toHaveLength(2);
});
```

**Giải thích:** Prisma hỗ trợ 2 kiểu transaction, và cách mock khác nhau tùy kiểu Service đang dùng:
- **Interactive transaction** (`$transaction(async (tx) => {...})`): mock bằng `mockImplementation` để tự gọi lại callback được truyền vào, thường truyền chính `prisma` mock làm tham số `tx` — nhờ vậy các lệnh gọi `tx.user.update(...)` bên trong callback vẫn đi qua đúng mock đã setup.
- **Batch transaction** (`$transaction([promise1, promise2])`): chỉ cần `mockResolvedValue([...])` trả thẳng mảng kết quả mong muốn, vì Service không tự xử lý logic bên trong transaction này — Prisma tự chạy song song các promise.

---

## 12. Testing Authentication

Module Authentication trong NestJS (thường dùng `@nestjs/jwt` + `@nestjs/passport`) có nhiều dependency ngoài (JwtService ký/giải token, ConfigService đọc secret key, bcrypt hash password) — cần mock đầy đủ để test AuthService/Strategy độc lập, không phụ thuộc biến môi trường thật hay thời gian sống token thật.

### Mock `JwtService`

```ts
const mockJwtService = {
  sign: jest.fn().mockReturnValue('mocked.jwt.token'),
  signAsync: jest.fn().mockResolvedValue('mocked.jwt.token'),
  verify: jest.fn(),
  verifyAsync: jest.fn(),
  decode: jest.fn(),
};
```

**Giải thích:** `JwtService` thật sẽ ký/giải token dựa trên secret key thật — trong unit test ta không quan tâm thuật toán JWT có đúng chuẩn hay không (đó là trách nhiệm của thư viện `jsonwebtoken`, đã được test sẵn), mà chỉ quan tâm **AuthService có gọi đúng `sign`/`verify` với đúng payload hay không**. Vì vậy mock trả về chuỗi token giả cố định là đủ.

### Mock `ConfigService`

```ts
const mockConfigService = {
  get: jest.fn((key: string) => {
    const config = {
      JWT_SECRET: 'test-secret',
      JWT_EXPIRES_IN: '1h',
      JWT_REFRESH_SECRET: 'test-refresh-secret',
    };
    return config[key];
  }),
};
```

**Giải thích:** `ConfigService.get('KEY')` được gọi rất nhiều nơi (đọc secret, thời gian hết hạn token...) — mock dạng hàm nhận `key` rồi tra trong một object cấu hình giả giúp mock hoạt động đúng cho **mọi key** mà không cần set riêng từng `mockReturnValueOnce`, tránh phải sửa mock mỗi khi Service đọc thêm biến môi trường mới.

### Test `AuthService`

```ts
describe('AuthService', () => {
  let service: AuthService;
  let usersService: { findByEmail: jest.Mock };
  let jwtService: typeof mockJwtService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        AuthService,
        { provide: UsersService, useValue: { findByEmail: jest.fn() } },
        { provide: JwtService, useValue: mockJwtService },
        { provide: ConfigService, useValue: mockConfigService },
      ],
    }).compile();

    service = module.get(AuthService);
    usersService = module.get(UsersService);
    jwtService = module.get(JwtService);
    jest.clearAllMocks();
  });

  it('login thành công trả về access token', async () => {
    const hashed = await bcrypt.hash('123456', 10);
    usersService.findByEmail.mockResolvedValue({
      id: 1,
      email: 'a@test.com',
      password: hashed,
    });

    const result = await service.login('a@test.com', '123456');

    expect(result).toHaveProperty('accessToken', 'mocked.jwt.token');
    expect(jwtService.sign).toHaveBeenCalledWith(
      expect.objectContaining({ sub: 1, email: 'a@test.com' })
    );
  });

  it('login thất bại khi sai mật khẩu -> UnauthorizedException', async () => {
    const hashed = await bcrypt.hash('123456', 10);
    usersService.findByEmail.mockResolvedValue({ id: 1, password: hashed });

    await expect(service.login('a@test.com', 'wrong-pass')).rejects.toThrow(
      UnauthorizedException
    );
  });

  it('login thất bại khi user không tồn tại -> UnauthorizedException', async () => {
    usersService.findByEmail.mockResolvedValue(null);

    await expect(service.login('notfound@test.com', '123456')).rejects.toThrow(
      UnauthorizedException
    );
  });
});
```

**Giải thích:**
- Ba nhánh test (đăng nhập đúng, sai mật khẩu, không tồn tại user) bao phủ đầy đủ các luồng chính của logic xác thực — đây là phần **bắt buộc phải test kỹ** vì liên quan trực tiếp bảo mật.
- Có thể dùng `bcrypt.hash` thật (không mock) trong test này vì tốc độ hash với salt round thấp vẫn đủ nhanh cho unit test, và nó xác nhận đúng luồng `bcrypt.compare` thật trong `AuthService` — chỉ nên mock `bcrypt` khi cần cô lập hoàn toàn khỏi tốc độ/side-effect (xem mục 16).
- Assert `jwtService.sign` được gọi với đúng payload (`sub`, `email`) giúp phát hiện lỗi nếu Service lỡ đưa nhầm thông tin nhạy cảm (như password) vào payload JWT.

### Test `JwtStrategy`

```ts
describe('JwtStrategy', () => {
  let strategy: JwtStrategy;
  let usersService: { findById: jest.Mock };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        JwtStrategy,
        { provide: UsersService, useValue: { findById: jest.fn() } },
        { provide: ConfigService, useValue: mockConfigService },
      ],
    }).compile();

    strategy = module.get(JwtStrategy);
    usersService = module.get(UsersService);
  });

  it('validate trả về user khi payload hợp lệ', async () => {
    usersService.findById.mockResolvedValue({ id: 1, email: 'a@test.com' });

    const result = await strategy.validate({ sub: 1, email: 'a@test.com' });

    expect(result).toEqual({ id: 1, email: 'a@test.com' });
  });

  it('validate throw UnauthorizedException khi user không còn tồn tại', async () => {
    usersService.findById.mockResolvedValue(null);

    await expect(
      strategy.validate({ sub: 999, email: 'ghost@test.com' })
    ).rejects.toThrow(UnauthorizedException);
  });
});
```

**Giải thích:** `JwtStrategy` (extends `PassportStrategy(Strategy)`) có method `validate(payload)` được Passport tự động gọi **sau khi** đã xác minh chữ ký token hợp lệ — vì vậy test strategy không cần quan tâm việc verify JWT (Passport lo phần đó), chỉ cần test logic riêng của app: từ payload đã giải mã, có tra đúng user trong DB không, và có xử lý đúng trường hợp user đã bị xóa/khóa (token còn hạn nhưng user không còn tồn tại) hay không.

---

## 13. Testing Exception

NestJS có sẵn các class exception chuẩn HTTP (`NotFoundException`, `UnauthorizedException`, `BadRequestException`, `ConflictException`...), mỗi class tự map sang đúng HTTP status code. Test các exception này ở 2 tầng: **tầng Service/unit** (đúng loại exception được throw đúng điều kiện) và **tầng e2e/HTTP** (đúng status code + message trả về client).

### `NotFoundException`

```ts
// Unit test (tầng Service)
it('findOne throw NotFoundException khi id không tồn tại', async () => {
  prisma.user.findUnique.mockResolvedValue(null);

  await expect(service.findOne(999)).rejects.toThrow(NotFoundException);
  await expect(service.findOne(999)).rejects.toThrow('User with id 999 not found');
});
```

```ts
// E2E test (tầng HTTP)
it('GET /users/:id trả 404 khi không tồn tại', () => {
  return request(app.getHttpServer())
    .get('/users/999')
    .expect(404)
    .expect((res) => {
      expect(res.body.message).toContain('not found');
    });
});
```

**Giải thích:** nên test **cả 2 tầng** vì mỗi tầng xác nhận một điều khác nhau — unit test xác nhận đúng logic nghiệp vụ (khi nào nên throw), e2e test xác nhận Nest's built-in exception filter đã convert đúng exception đó thành response HTTP 404 với đúng format body (`{ statusCode, message, error }`).

### `UnauthorizedException`

```ts
it('validate token hết hạn -> UnauthorizedException', async () => {
  jwtService.verifyAsync.mockRejectedValue(new Error('jwt expired'));

  await expect(service.refreshToken('expired.token')).rejects.toThrow(
    UnauthorizedException
  );
});

it('GET /protected không có token trả 401', () => {
  return request(app.getHttpServer()).get('/protected').expect(401);
});
```

**Giải thích:** `UnauthorizedException` (401) khác `ForbiddenException` (403) về ý nghĩa: 401 nghĩa là "chưa xác thực được danh tính" (thiếu/sai token), còn 403 nghĩa là "đã biết bạn là ai nhưng không đủ quyền". Khi viết test, cần đảm bảo Service/Guard throw đúng loại exception tương ứng đúng tình huống, tránh nhầm lẫn hai mã lỗi này.

### `BadRequestException`

```ts
it('create throw BadRequestException khi thiếu field bắt buộc', async () => {
  await expect(service.create({ email: '' } as any)).rejects.toThrow(
    BadRequestException
  );
});

// Test qua ValidationPipe ở tầng e2e (thường gặp hơn test thủ công trong Service)
it('POST /users trả 400 khi email sai định dạng', () => {
  return request(app.getHttpServer())
    .post('/users')
    .send({ email: 'not-an-email', name: 'A' })
    .expect(400)
    .expect((res) => {
      expect(res.body.message).toEqual(
        expect.arrayContaining([expect.stringContaining('email')])
      );
    });
});
```

**Giải thích:** trong NestJS, `BadRequestException` phần lớn **không phải do Service tự throw thủ công** mà đến từ `ValidationPipe` (kết hợp `class-validator` trên DTO) tự động validate request — vì vậy test 400 thường hiệu quả nhất ở tầng e2e, gửi payload sai để xác nhận cả chuỗi DTO → decorator validate → ValidationPipe → response hoạt động đúng, thay vì chỉ test Service throw exception thủ công.

### `ConflictException`

```ts
it('create throw ConflictException khi email đã tồn tại', async () => {
  prisma.user.findUnique.mockResolvedValue({ id: 1, email: 'a@test.com' });

  await expect(
    service.create({ email: 'a@test.com', name: 'A' })
  ).rejects.toThrow(ConflictException);
});

// Trường hợp bắt lỗi unique constraint trực tiếp từ Prisma (P2002)
it('create throw ConflictException khi Prisma báo lỗi unique constraint', async () => {
  prisma.user.create.mockRejectedValue({
    code: 'P2002',
    meta: { target: ['email'] },
  });

  await expect(service.create({ email: 'a@test.com', name: 'A' })).rejects.toThrow(
    ConflictException
  );
});
```

**Giải thích:** có hai cách phổ biến để phát hiện trùng lặp dữ liệu — (1) Service tự query kiểm tra trước khi insert (`findUnique` rồi so sánh), hoặc (2) để DB tự chặn qua unique constraint và bắt lỗi Prisma mã `P2002` rồi convert sang `ConflictException`. Nên test cả hai nhánh nếu code xử lý cả hai trường hợp, vì race condition (2 request tạo cùng email cùng lúc) chỉ nhánh (2) mới bắt được, còn nhánh (1) đơn thuần chỉ optimize UX (báo lỗi sớm, thân thiện hơn).

---

## 14. Testing Cache & Redis

Cache (thường qua `@nestjs/cache-manager` hoặc Redis client trực tiếp như `ioredis`) là một dependency I/O bên ngoài — cũng như DB, cần mock khi unit test để không phụ thuộc Redis server thật đang chạy, đồng thời test được cả 2 kịch bản quan trọng: cache hit và cache miss.

### Mock `CacheManager`

```ts
const mockCacheManager = {
  get: jest.fn(),
  set: jest.fn(),
  del: jest.fn(),
  reset: jest.fn(),
};

const module: TestingModule = await Test.createTestingModule({
  providers: [
    UsersService,
    { provide: CACHE_MANAGER, useValue: mockCacheManager }, // token từ @nestjs/cache-manager
    { provide: PrismaService, useValue: mockPrismaService },
  ],
}).compile();

service = module.get(UsersService);
cacheManager = module.get(CACHE_MANAGER);
```

**Giải thích:** `@nestjs/cache-manager` inject cache instance qua token đặc biệt `CACHE_MANAGER` (tương tự `getRepositoryToken` của TypeORM), không phải qua class — phải dùng đúng token này khi khai báo provider giả, nếu không Nest sẽ báo lỗi không tìm thấy dependency khi resolve DI.

### Mock `Redis`

```ts
// Trường hợp dùng trực tiếp ioredis client (không qua CacheManager)
jest.mock('ioredis');
import Redis from 'ioredis';

const mockRedisInstance = {
  get: jest.fn(),
  set: jest.fn(),
  del: jest.fn(),
  expire: jest.fn(),
  incr: jest.fn(),
};

(Redis as unknown as jest.Mock).mockImplementation(() => mockRedisInstance);
```

**Giải thích:** khi Service tự khởi tạo `new Redis(...)` trực tiếp (không thông qua DI của Nest), cách gọn nhất là `jest.mock('ioredis')` để mock cả class, rồi ép `mockImplementation` trả về một object giả có đủ các method Service dùng tới. Cách này phù hợp cho các đoạn code dùng Redis client thuần, ví dụ để làm rate-limiting hay lock phân tán, không đi qua `CacheManager`.

### Kiểm tra cache hit/miss

```ts
describe('getUserWithCache', () => {
  it('cache hit: trả dữ liệu từ cache, không gọi DB', async () => {
    cacheManager.get.mockResolvedValue({ id: 1, name: 'John (cached)' });

    const result = await service.getUserWithCache(1);

    expect(result).toEqual({ id: 1, name: 'John (cached)' });
    expect(cacheManager.get).toHaveBeenCalledWith('user:1');
    expect(prisma.user.findUnique).not.toHaveBeenCalled(); // quan trọng: không chạm DB
  });

  it('cache miss: query DB rồi set lại vào cache', async () => {
    cacheManager.get.mockResolvedValue(undefined); // hoặc null tùy implementation
    prisma.user.findUnique.mockResolvedValue({ id: 1, name: 'John' });

    const result = await service.getUserWithCache(1);

    expect(result).toEqual({ id: 1, name: 'John' });
    expect(prisma.user.findUnique).toHaveBeenCalledWith({ where: { id: 1 } });
    expect(cacheManager.set).toHaveBeenCalledWith(
      'user:1',
      { id: 1, name: 'John' },
      expect.any(Number) // TTL
    );
  });

  it('xóa cache sau khi update', async () => {
    prisma.user.update.mockResolvedValue({ id: 1, name: 'Updated' });

    await service.updateUser(1, { name: 'Updated' });

    expect(cacheManager.del).toHaveBeenCalledWith('user:1');
  });
});
```

**Giải thích:** đây là 3 nhánh logic **bắt buộc phải test riêng** khi Service có cache layer, vì lỗi cache rất khó phát hiện qua code review thông thường (bug thường chỉ lộ ra khi chạy thật với dữ liệu cũ bị "stale"):
- **Cache hit**: assert quan trọng nhất là `prisma.user.findUnique` **không được gọi** — nếu Service lỡ query DB dù cache đã có dữ liệu, đó là bug làm mất hết lợi ích của cache.
- **Cache miss**: assert Service có gọi DB **và** ghi lại kết quả vào cache (`cacheManager.set`) kèm đúng key/TTL — quên bước `set` là lỗi phổ biến khiến mọi request sau đều miss cache.
- **Cache invalidation**: sau khi update/delete dữ liệu, cache cũ (key liên quan) phải được xóa (`del`) hoặc ghi đè — nếu quên bước này, hệ thống sẽ trả dữ liệu cũ (stale data) cho tới khi cache tự hết hạn.

---

## 15. Testing Logger & Time

Logger và thời gian hệ thống (`Date.now()`, `new Date()`) là hai nguồn "non-determinism" phổ biến nhất khiến test không ổn định (flaky) nếu không kiểm soát — cần mock để test cho ra kết quả giống nhau ở mọi lần chạy, bất kể chạy lúc mấy giờ.

### Mock `Logger`

```ts
import { Logger } from '@nestjs/common';

it('log lỗi khi tạo user thất bại', async () => {
  const loggerSpy = jest.spyOn(Logger.prototype, 'error').mockImplementation();
  prisma.user.create.mockRejectedValue(new Error('DB down'));

  await expect(service.create({ email: 'a@test.com', name: 'A' })).rejects.toThrow();

  expect(loggerSpy).toHaveBeenCalledWith(
    expect.stringContaining('DB down'),
    expect.any(String) // stack trace
  );
  loggerSpy.mockRestore();
});
```

**Giải thích:** dùng `jest.spyOn(Logger.prototype, 'error')` để chặn method `error` của **toàn bộ instance Logger** trong Nest (vì Nest tạo Logger instance nội bộ, khó mock qua DI thông thường) — vừa để assert Service có log đúng lỗi quan trọng, vừa để **ngăn output log rác** làm rối terminal khi chạy test (test log lỗi cố ý vẫn in ra console nếu không mock implementation rỗng). Luôn `mockRestore()` sau đó để không ảnh hưởng log của các test khác.

### Mock `Date.now()`

```ts
it('tạo record với timestamp cố định', () => {
  const fixedTimestamp = 1700000000000; // 14/11/2023
  jest.spyOn(Date, 'now').mockReturnValue(fixedTimestamp);

  const result = service.createTimestampedRecord();

  expect(result.createdAt).toBe(fixedTimestamp);

  (Date.now as jest.Mock).mockRestore();
});
```

**Giải thích:** nếu không mock, mỗi lần chạy test `Date.now()` trả về một giá trị khác nhau, khiến không thể assert bằng `toBe`/`toEqual` với một con số cụ thể — phải dùng cách khác kém chính xác hơn (như `toBeGreaterThan`). Mock `Date.now` cố định giá trị, cho phép assert chính xác tuyệt đối, đồng thời giúp test logic liên quan thời gian (tính tuổi token, hạn hết hiệu lực...) mà không cần chờ thời gian thật trôi qua.

### `jest.setSystemTime()`

```ts
describe('Token expiry logic', () => {
  beforeEach(() => {
    jest.useFakeTimers();
    jest.setSystemTime(new Date('2026-01-01T00:00:00Z'));
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  it('token còn hạn khi chưa quá 1 giờ', () => {
    const token = { issuedAt: new Date('2026-01-01T00:00:00Z'), ttlHours: 1 };
    expect(service.isTokenExpired(token)).toBe(false);
  });

  it('token hết hạn sau khi tua thời gian thêm 2 giờ', () => {
    const token = { issuedAt: new Date('2026-01-01T00:00:00Z'), ttlHours: 1 };

    jest.setSystemTime(new Date('2026-01-01T02:00:00Z')); // tua "đồng hồ hệ thống"

    expect(service.isTokenExpired(token)).toBe(true);
  });
});
```

**Giải thích:** `jest.setSystemTime()` mạnh hơn việc chỉ mock riêng `Date.now()` — nó thay đổi luôn "đồng hồ hệ thống" mà Jest fake timer dùng, khiến **mọi** lệnh gọi `new Date()` (không chỉ `Date.now()`) trong suốt quá trình test đều trả về đúng thời điểm giả định. Đây là cách chuẩn để test logic phụ thuộc nhiều thời điểm khác nhau (ví dụ so sánh "hiện tại" với "thời điểm hết hạn"), vì có thể "tua đồng hồ" tới bất kỳ mốc nào mà không cần chờ thời gian thực trôi qua. Yêu cầu bắt buộc phải bật `jest.useFakeTimers()` trước khi gọi `setSystemTime`.

---

## 16. Testing UUID / External Libraries

Các thư viện bên thứ ba (tạo ID ngẫu nhiên, hash mật khẩu, gọi API ngoài, SDK của dịch vụ cloud) đều có đặc điểm chung: kết quả không xác định trước (`uuid`), tốn thời gian tính toán (`bcrypt`), hoặc phụ thuộc mạng/dịch vụ ngoài (`axios`, SDK bên thứ 3) — nên mock để test nhanh, ổn định, và không tốn chi phí/quota dịch vụ thật khi chạy test.

### Mock `uuid`

```ts
jest.mock('uuid', () => ({
  v4: jest.fn(() => 'fixed-uuid-1234'),
}));
import { v4 as uuidv4 } from 'uuid';

it('create sinh id bằng uuid v4', async () => {
  prisma.user.create.mockResolvedValue({ id: 'fixed-uuid-1234', name: 'A' });

  const result = await service.create({ name: 'A' });

  expect(uuidv4).toHaveBeenCalled();
  expect(result.id).toBe('fixed-uuid-1234');
});
```

**Giải thích:** UUID sinh ra ngẫu nhiên mỗi lần gọi, nên nếu không mock, không thể assert `result.id` bằng một giá trị cụ thể trong test (`toBe('some-fixed-value')`) — mock trả về giá trị cố định giúp test dễ viết assertion chính xác, đồng thời xác nhận được rằng Service **có gọi** `uuidv4()` để sinh ID (thay vì để DB tự sinh, hoặc quên sinh ID).

### Mock `bcrypt`

```ts
jest.mock('bcrypt');
import * as bcrypt from 'bcrypt';

it('register hash password trước khi lưu', async () => {
  (bcrypt.hash as jest.Mock).mockResolvedValue('hashed-password-abc');
  prisma.user.create.mockResolvedValue({ id: 1, password: 'hashed-password-abc' });

  await service.register({ email: 'a@test.com', password: '123456' });

  expect(bcrypt.hash).toHaveBeenCalledWith('123456', 10); // salt round = 10
  expect(prisma.user.create).toHaveBeenCalledWith({
    data: expect.objectContaining({ password: 'hashed-password-abc' }),
  });
});

it('login so sánh password bằng bcrypt.compare', async () => {
  (bcrypt.compare as jest.Mock).mockResolvedValue(true);
  usersService.findByEmail.mockResolvedValue({ id: 1, password: 'hashed' });

  const result = await service.login('a@test.com', 'plain-text-pass');

  expect(bcrypt.compare).toHaveBeenCalledWith('plain-text-pass', 'hashed');
  expect(result).toBeDefined();
});
```

**Giải thích:** mock `bcrypt` phù hợp khi muốn **cô lập hoàn toàn** test khỏi tốc độ hash thật (hash với salt round cao có thể mất hàng trăm ms, làm chậm cả suite nếu có nhiều test), hoặc khi chỉ muốn test logic điều phối của Service (có gọi đúng hàm, đúng tham số hay không) mà không quan tâm thuật toán hash đúng sai (trách nhiệm của thư viện `bcrypt`, không phải của Service). Nếu muốn test cả luồng hash/compare thật đúng nhau (như ví dụ AuthService ở mục 12), có thể để `bcrypt` chạy thật thay vì mock.

### Mock `axios`

```ts
jest.mock('axios');
import axios from 'axios';

it('gọi API thời tiết bên ngoài và parse kết quả', async () => {
  (axios.get as jest.Mock).mockResolvedValue({
    data: { temp: 28, city: 'Hanoi' },
  });

  const result = await weatherService.getWeather('Hanoi');

  expect(axios.get).toHaveBeenCalledWith(
    expect.stringContaining('/weather'),
    expect.objectContaining({ params: { city: 'Hanoi' } })
  );
  expect(result.temp).toBe(28);
});

it('xử lý đúng khi API bên ngoài trả lỗi', async () => {
  (axios.get as jest.Mock).mockRejectedValue({
    response: { status: 503, data: { message: 'Service Unavailable' } },
  });

  await expect(weatherService.getWeather('Hanoi')).rejects.toThrow();
});
```

**Giải thích:** khi Service gọi API bên ngoài qua `axios`, cần test cả nhánh **thành công** (parse đúng response) lẫn nhánh **API ngoài bị lỗi/timeout** (Service có xử lý gracefully, không để lỗi rò rỉ nguyên dạng ra client hay không) — nhánh lỗi thường bị bỏ sót nhưng lại là nhánh quan trọng nhất về mặt độ ổn định hệ thống trong thực tế (API bên thứ ba luôn có khả năng down).

### Mock các SDK bên thứ ba (Cloudinary, SePay, Gemini...)

```ts
// Cloudinary (upload ảnh)
jest.mock('cloudinary', () => ({
  v2: {
    uploader: {
      upload: jest.fn(),
      destroy: jest.fn(),
    },
  },
}));
import { v2 as cloudinary } from 'cloudinary';

it('upload ảnh trả về secure_url', async () => {
  (cloudinary.uploader.upload as jest.Mock).mockResolvedValue({
    secure_url: 'https://res.cloudinary.com/demo/image/upload/abc.jpg',
    public_id: 'abc',
  });

  const result = await mediaService.uploadImage('/tmp/fake-path.jpg');

  expect(result.url).toBe('https://res.cloudinary.com/demo/image/upload/abc.jpg');
});
```

```ts
// SDK có class riêng (ví dụ SePay webhook client, Gemini SDK) - mock cả class
jest.mock('@google/generative-ai');
import { GoogleGenerativeAI } from '@google/generative-ai';

const mockGenerateContent = jest.fn();
(GoogleGenerativeAI as jest.Mock).mockImplementation(() => ({
  getGenerativeModel: () => ({
    generateContent: mockGenerateContent,
  }),
}));

it('gọi Gemini và parse response text', async () => {
  mockGenerateContent.mockResolvedValue({
    response: { text: () => 'Đây là câu trả lời từ AI' },
  });

  const result = await aiService.ask('Xin chào');

  expect(result).toBe('Đây là câu trả lời từ AI');
});
```

**Giải thích:**
- Với SDK dạng object export sẵn method (như Cloudinary: `cloudinary.uploader.upload`), mock bằng factory function trả về đúng shape object đó là đủ, tương tự cách mock module thông thường.
- Với SDK dạng class phải khởi tạo (`new GoogleGenerativeAI(apiKey)`), cần `jest.mock(...)` cả module rồi override `mockImplementation` của constructor để trả về một object giả có đủ các method con (`getGenerativeModel().generateContent()`) mà code thực tế gọi tới — chú ý mock đúng **toàn bộ chuỗi method chaining** mà SDK yêu cầu, thiếu một bước sẽ khiến mock trả về `undefined` gây lỗi runtime trong lúc test.
- Nguyên tắc chung cho mọi SDK bên thứ ba (thanh toán như SePay, AI như Gemini, storage như Cloudinary...): **không bao giờ gọi API thật trong unit test** (tốn phí, không ổn định, cần API key thật, có thể có side-effect thật như trừ tiền/tạo giao dịch) — luôn mock ở biên giới giữa code của bạn và SDK, chỉ test logic của bạn (Service parse response, xử lý lỗi, map dữ liệu) đúng hay sai.

---

## Tham khảo nhanh: cấu trúc thư mục test phổ biến

```
src/
  users/
    users.controller.ts
    users.controller.spec.ts   # unit test controller
    users.service.ts
    users.service.spec.ts      # unit test service
test/
  app.e2e-spec.ts              # e2e test (NestJS)
  jest-e2e.json                # config riêng cho e2e
```

**Giải thích:** NestJS quy ước đặt file `.spec.ts` **ngay cạnh** file source tương ứng (co-location) thay vì gom vào thư mục `__tests__` riêng — giúp dễ tìm test khi đang xem code, và nhắc nhở lập trình viên viết test song song với viết feature. Riêng e2e test được tách ra thư mục `test/` ở gốc project vì nó test ở tầng ứng dụng (nhiều module ráp lại), không gắn với một file source cụ thể nào.
