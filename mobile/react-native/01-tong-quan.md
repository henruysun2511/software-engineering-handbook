# GIÁO TRÌNH REACT NATIVE
### Dành cho lập trình viên đã có nền tảng React / Next.js

# CHƯƠNG 1: TỔNG QUAN VỀ REACT NATIVE

## 1.1. React Native là gì

### 1.1.1. Khái niệm

React Native (RN) là một framework mã nguồn mở do Meta phát triển, cho phép xây dựng ứng dụng di động (iOS, Android) bằng JavaScript/TypeScript và React, nhưng render ra **thành phần giao diện gốc (native UI components)** thay vì DOM của trình duyệt.

Điểm khác biệt cốt lõi so với React (Web):

| Tiêu chí | React (Web) | React Native |
|---|---|---|
| Nền tảng render | DOM (HTML/CSS) trên trình duyệt | Native View (UIView trên iOS, android.view trên Android) |
| Ngôn ngữ style | CSS/Tailwind | StyleSheet API (subset CSS, camelCase) |
| Thẻ cơ bản | `<div>`, `<span>`, `<p>` | `<View>`, `<Text>`, `<Image>` |
| Routing | React Router / Next.js Router | React Navigation / Expo Router |
| Môi trường thực thi | V8 (Chrome) / trình duyệt | Hermes Engine (JS Engine tối ưu cho mobile) |
| Build output | HTML/JS/CSS bundle | File `.ipa` (iOS) / `.apk`, `.aab` (Android) |

> **Lưu ý sư phạm:** Vì bạn đã nắm React, phần lớn tư duy về component, hooks, unidirectional data flow, JSX là **tái sử dụng 100%**. Những gì cần học lại là: cách UI được render, cách style hoạt động, cách điều hướng, và cách tương tác với phần cứng thiết bị.

### 1.1.2. Kiến trúc vận hành (Architecture)

React Native có 2 kiến trúc:

**a. Kiến trúc cũ (Legacy/Bridge Architecture)**

```
JavaScript Thread  <--- Bridge (JSON serialize/deserialize, bất đồng bộ) --->  Native Thread (UI)
```

- JS Thread chạy code React, tính toán logic.
- Native Thread quản lý UI thật (UIKit/Android View).
- Giao tiếp qua **Bridge**: mọi lệnh (vd: cập nhật style, gọi API camera) phải serialize thành JSON, gửi bất đồng bộ qua Bridge → gây độ trễ, nghẽn khi có nhiều thao tác (scroll nhanh, animation phức tạp).

**b. Kiến trúc mới (New Architecture – mặc định từ RN 0.76+)**

```
JavaScript Thread  <--- JSI (JavaScript Interface, đồng bộ, gọi trực tiếp) --->  Native Thread
```

Gồm 3 thành phần chính:

- **JSI (JavaScript Interface):** Cho phép JS gọi trực tiếp hàm C++ native mà không cần serialize qua Bridge → đồng bộ, nhanh hơn nhiều lần.
- **Fabric:** Renderer mới, quản lý cây UI (Shadow Tree) hiệu quả hơn, hỗ trợ concurrent rendering giống React 18 (Suspense, transitions).
- **TurboModules:** Cơ chế native module mới, load module theo kiểu lazy (chỉ load khi cần dùng) thay vì load toàn bộ lúc khởi động.

> **So sánh tương tự:** Nếu bạn từng làm việc với Next.js Server Components vs Client Components — sự khác biệt Bridge vs JSI cũng tương tự như việc chuyển từ "gọi API bất đồng bộ qua network" sang "gọi hàm trực tiếp trong cùng runtime".

### 1.1.3. Không có DOM — hệ quả thực tế

Vì không có DOM, các khái niệm sau **không tồn tại** trong RN:
- `document`, `window` (không dùng `localStorage`, `window.location`)
- Thẻ HTML thuần (`<div>`, `<button>`, `<input>`) — RN thay bằng component riêng
- CSS file `.css`, CSS-in-JS thư viện Web (Tailwind chuẩn không chạy được — phải dùng NativeWind, sẽ nói ở Chương 2)
- Sự kiện DOM chuẩn (`onClick` → `onPress`, không có event bubbling giống hệt Web)

---

## 1.2. Expo và React Native CLI (Bare Workflow)

### 1.2.1. Khái niệm

Đây là **hai cách khởi tạo và quản lý một dự án React Native**, không phải hai framework khác nhau — bản chất cả hai đều build ra ứng dụng React Native.

- **React Native CLI (Bare Workflow):** Bạn quản lý trực tiếp thư mục `ios/` và `android/` (mã nguồn native Xcode/Android Studio). Toàn quyền kiểm soát, nhưng phải tự cấu hình native code khi cần thư viện native.
- **Expo (Managed/Prebuild Workflow):** Một bộ công cụ + SDK bọc quanh React Native CLI, cung cấp sẵn hàng trăm native module (camera, location, notification...) dưới dạng package JS thuần, không cần đụng vào Xcode/Android Studio trong phần lớn trường hợp.

### 1.2.2. Bảng so sánh

| Tiêu chí | Expo (Managed) | React Native CLI (Bare) |
|---|---|---|
| Tốc độ khởi tạo dự án | Rất nhanh (`npx create-expo-app`) | Cần cài Xcode, Android Studio đầy đủ |
| Native module tùy biến | Hạn chế hơn (nhưng có Config Plugins & Development Build để mở rộng) | Toàn quyền viết native code |
| Build ứng dụng | EAS Build (cloud build, không cần máy Mac để build iOS) | Phải build local hoặc tự cấu hình CI |
| OTA Update (cập nhật không qua store) | Có sẵn (Expo Updates) | Cần tự tích hợp (CodePush hoặc tự viết) |
| File cấu hình native (`Info.plist`, `AndroidManifest.xml`) | Quản lý qua `app.json`/`app.config.ts` (Config Plugin) | Sửa trực tiếp |
| Độ phù hợp | 95% ứng dụng thương mại hiện nay | Dự án cần SDK native đặc thù, hiệu năng cực hạn, hoặc tích hợp sâu native hiện có |
| Kích thước cộng đồng/tài liệu | Rất lớn, cập nhật nhanh theo New Architecture | Lớn nhưng phân mảnh hơn |

> **Kết luận khuyến nghị (2026):** Nên bắt đầu với **Expo**. Từ SDK 50 trở đi, Expo hỗ trợ đầy đủ New Architecture, và với **Expo Development Build**, bạn hoàn toàn có thể thêm bất kỳ thư viện native nào (kể cả chưa hỗ trợ Expo Go) mà không mất đi lợi ích quản lý của Expo. Ranh giới "Expo bị giới hạn" gần như đã biến mất.

### 1.2.3. Khởi tạo dự án chuẩn production

```bash
# Tạo project với TypeScript template
npx create-expo-app@latest my-app --template

# Chọn template: "Navigation (TypeScript)" để có sẵn cấu trúc Expo Router

cd my-app

# Cài đặt các thư viện nền tảng sẽ dùng xuyên suốt giáo trình
npx expo install expo-router react-native-safe-area-context react-native-screens
npm install zustand @tanstack/react-query axios zod
npm install nativewind tailwindcss
```

**Giải thích lệnh:**
- `npx expo install` khác `npm install` ở chỗ: Expo sẽ tự chọn **đúng phiên bản thư viện tương thích** với SDK Expo hiện tại của bạn — tránh lỗi version mismatch, vốn là nguyên nhân phổ biến nhất gây crash app khi build production.
- Các package thuần JS logic (`zustand`, `@tanstack/react-query`, `axios`, `zod`) dùng `npm install` bình thường vì không đụng đến native code.

### 1.2.4. Cấu trúc thư mục chuẩn (Expo Router)

```
my-app/
├── app/                      # Routing (file-based, giống Next.js App Router)
│   ├── (tabs)/               # Route group - Tab Navigator
│   │   ├── index.tsx         # Màn hình Home
│   │   └── profile.tsx
│   ├── _layout.tsx           # Layout gốc (giống layout.tsx của Next.js)
│   └── +not-found.tsx        # Trang 404
├── src/
│   ├── components/           # Component tái sử dụng
│   ├── hooks/                # Custom hooks
│   ├── store/                # Zustand stores
│   ├── services/             # Axios instance, API calls
│   ├── schemas/              # Zod schemas
│   └── utils/
├── assets/                   # Ảnh, font
├── app.config.ts             # Cấu hình app (thay app.json để dùng được TS + logic động)
├── babel.config.js
├── tailwind.config.js
└── package.json
```

---

## 1.3. Cài đặt môi trường phát triển

### 1.3.1. Các công cụ cần thiết

| Công cụ | Mục đích | Bắt buộc? |
|---|---|---|
| Node.js (LTS) | Chạy Metro bundler, CLI | Bắt buộc |
| Xcode (macOS only) | Build/chạy giả lập iOS | Chỉ khi build native iOS local |
| Android Studio | Build/chạy giả lập Android, quản lý SDK | Chỉ khi build native Android local |
| Expo Go (app trên điện thoại) | Chạy thử nhanh không cần build | Khuyến nghị cho giai đoạn học/prototype |
| EAS CLI | Build cloud, submit lên store | Bắt buộc khi lên production |

### 1.3.2. Quy trình chạy dự án

```bash
# Chạy Metro bundler + mở lựa chọn nền tảng
npx expo start

# Các phím tắt trong terminal khi Metro đang chạy:
# a -> mở trên Android emulator
# i -> mở trên iOS simulator (chỉ macOS)
# w -> mở bản web (Expo hỗ trợ web qua react-native-web)
# r -> reload app
```

> **Lưu ý quan trọng về Expo Go vs Development Build:**
> - **Expo Go**: app có sẵn trên App Store/Play Store, cho phép chạy thử ngay lập tức nhưng **chỉ hỗ trợ các native module có sẵn trong Expo SDK**.
> - **Development Build**: bản build riêng của app bạn (`npx expo run:ios` / `npx expo run:android`), cho phép cài thêm bất kỳ thư viện native nào (vd: Reanimated phiên bản mới, Firebase, thư viện bên thứ ba tùy biến).
> - Với dự án production thực tế, gần như luôn cần chuyển sang **Development Build** ngay từ đầu để tránh giới hạn của Expo Go.

---

## 1.4. Metro Bundler

### 1.4.1. Khái niệm

Metro là JavaScript bundler chính thức của React Native — đóng vai trò tương đương **Webpack/Turbopack trong Next.js**, nhưng được thiết kế riêng cho môi trường mobile.

### 1.4.2. So sánh với Webpack/Turbopack (Next.js)

| Tiêu chí | Metro (React Native) | Webpack/Turbopack (Next.js) |
|---|---|---|
| Mục tiêu | Bundle cho JS Engine mobile (Hermes) | Bundle cho trình duyệt |
| Code splitting | Hạn chế (mobile app thường load 1 bundle) | Rất mạnh (route-based splitting) |
| Fast Refresh | Có, tối ưu cho reload native | Có (Fast Refresh/HMR) |
| Asset resolution | Resolve ảnh, font thành require() native | Resolve qua public/ hoặc import |
| Cấu hình | `metro.config.js` | `next.config.js` |

### 1.4.3. Cấu hình Metro cơ bản (khi tích hợp NativeWind)

```js
// metro.config.js
const { getDefaultConfig } = require('expo/metro-config');
const { withNativeWind } = require('nativewind/metro');

const config = getDefaultConfig(__dirname);

module.exports = withNativeWind(config, { input: './src/global.css' });
```

**Giải thích:** Metro cần được "bọc" (wrap) thêm cấu hình của NativeWind để có thể biên dịch class Tailwind thành style native lúc build. Đây là điểm khác biệt với Next.js — nơi Tailwind hoạt động qua PostCSS trực tiếp trên CSS.

