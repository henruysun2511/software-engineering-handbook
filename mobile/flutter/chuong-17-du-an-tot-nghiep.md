# Chương 17: Dự Án Tốt Nghiệp & Portfolio

---

## 17.1. Gợi Ý Ý Tưởng Dự Án Portfolio

### 17.1.1. Tiêu Chí Chọn Dự Án Portfolio

Một dự án portfolio tốt cho Flutter junior phải:

- **Có tính năng thực tế** — không chỉ là clone tutorial TodoList
- **Dùng đủ tech stack** — Riverpod, Go Router, Dio, Formz, Animation
- **Có thể demo được** — AI, camera, maps, push notification, realtime
- **Hoàn thiện UI** — dark mode, skeleton loading, empty state, error state
- **Có code sạch** — folder structure rõ ràng, naming convention nhất quán
- **Chạy được trên cả Android lẫn iOS**

### 17.1.2. Ba Hướng Dự Án Đề Xuất

---

**Hướng 1: ShopMate — E-Commerce App**

Phù hợp nếu muốn làm ở công ty product, startup.

```
Tính năng:
├── Auth: Email/Password + Google Sign-In + Biometric
├── Product: Danh sách, filter, search debounce, detail, ảnh zoom
├── Cart: Add/remove, quantity, coupon code, tính total
├── Checkout: Address, payment method, place order
├── Order: Lịch sử, tracking realtime (Firestore stream)
├── Profile: Avatar upload, edit info, change password
├── Wishlist: Offline (Hive), sync khi login
├── Notification: FCM push, in-app notification
├── Map: Chọn địa chỉ giao hàng từ Google Maps
└── Dark mode + 3 seed color để chọn
```

---

**Hướng 2: TaskFlow — Project Management App**

Phù hợp nếu muốn làm ở công ty outsource, enterprise.

```
Tính năng:
├── Auth: Workspace invite link, role-based access
├── Project: Tạo project, thành viên, deadline
├── Task: Kanban board (drag & drop), task detail, comment
├── Realtime: Firestore — mọi người cùng thấy update ngay
├── File: Upload attachment, preview PDF/image
├── Notification: Push khi được assign, deadline sắp đến
├── Calendar view: Task theo ngày
├── Analytics: Burndown chart, task completion rate
└── Offline: Hive cache, sync khi có mạng
```

---

**Hướng 3: FoodLog — Health Tracking App**

Phù hợp nếu muốn làm ở health tech, startup.

```
Tính năng:
├── Auth: Apple Sign-In + Health Kit integration
├── Food: Barcode scan (camera), search food database (API)
├── Meal log: Breakfast/lunch/dinner/snack, ảnh bữa ăn
├── Nutrition: Macro tracking (protein/carb/fat), pie chart
├── Goal: Calo target, weight goal, BMI calculator
├── Streak: Consecutive days tracking, gamification
├── Widget: iOS/Android home screen widget
├── Chart: Line chart (cân nặng), bar chart (calo theo tuần)
└── Export: Chia sẻ progress dưới dạng ảnh
```

---

## 17.2. Kế Hoạch Xây Dựng ShopMate (4 Tuần)

### Tuần 1: Nền tảng

```
□ Setup project: flavor dev/prod, cấu trúc thư mục
□ Core: ThemeData, AppRouter, DioClient, error types
□ Auth: Firebase Auth (email + Google), AuthProvider
□ Splash screen: Check auth state, navigate đúng màn hình
□ Login/Register: Form với Formz validation đầy đủ
□ Home skeleton: Shell với NavigationBar 4 tab
```

### Tuần 2: Core Features

```
□ Product list: Infinite scroll, filter bottom sheet
□ Product search: SearchBar với debounce, suggestions
□ Product detail: Image gallery, add to cart, share
□ Cart: CartProvider với Hive persist, coupon
□ Checkout: Address picker (Google Maps), payment method
□ Order success: Lottie animation
```

### Tuần 3: Polish và Extra Features

```
□ Profile: Avatar upload (camera/gallery), edit info
□ Order history: List + detail với Firestore stream
□ Wishlist: Hive offline, sync khi login
□ Push notification: FCM setup, deep link
□ Dark mode: Theme switcher trong Settings
□ Skeleton loading: Mọi màn hình
□ Error state: Retry button mọi nơi
```

### Tuần 4: Hoàn Thiện

```
□ Animation: Hero transition, page transition, add-to-cart effect
□ Performance: DevTools profiling, fix jank nếu có
□ Testing: Unit test business logic, widget test form
□ README: Mô tả, screenshots, cách chạy
□ Deploy: Build APK demo, TestFlight
□ Code review: Clean up, remove TODO, đặt tên nhất quán
```

---

## 17.3. Checklist Hoàn Thiện Dự Án

### 17.3.1. Code Quality

```
□ Không có unused import, unused variable
□ Mọi widget có const constructor khi có thể
□ Không có print() statement trong production code
□ Không có hardcoded string (dùng constant)
□ Không có hardcoded color (dùng colorScheme)
□ Không có magic number (đặt tên constant)
□ Tất cả controller, subscription đã được dispose
□ Mọi async function có error handling
□ Không có empty catch block
□ Dead code đã được xóa
```

### 17.3.2. UX/UI Checklist

```
□ Dark mode hoạt động đúng trên mọi màn hình
□ Skeleton loading cho tất cả màn hình có data từ API
□ Empty state có illustration và CTA button
□ Error state có message rõ ràng và retry button
□ Loading button (disable + spinner khi đang submit)
□ Pull-to-refresh trên list màn hình
□ Keyboard tự đóng khi tap ngoài TextField
□ TextField focus đúng thứ tự khi nhấn Next
□ SafeArea đầy đủ (không bị notch/home indicator che)
□ Text không bị overflow (dùng maxLines + ellipsis)
□ Image có fallback khi load lỗi
□ Tất cả button có trạng thái disabled rõ ràng
```

### 17.3.3. Kỹ Thuật Checklist

```
□ API key không commit lên Git
□ Prod config trong .gitignore
□ Flavor dev/prod hoạt động
□ Release build không có debug banner
□ Code obfuscation bật cho prod build
□ Crash reporting hoạt động (Crashlytics)
□ Deep link hoạt động (share product link)
□ Push notification hoạt động (background + foreground)
□ App hoạt động offline với cached data
□ App recover gracefully khi mất mạng
```

---

## 17.4. Viết README Chuyên Nghiệp

````markdown
# ShopMate — Flutter E-Commerce App

> Ứng dụng mua sắm hiện đại xây dựng với Flutter và Firebase

[![Flutter](https://img.shields.io/badge/Flutter-3.24-blue.svg)](https://flutter.dev)
[![Dart](https://img.shields.io/badge/Dart-3.5-blue.svg)](https://dart.dev)

## 📱 Screenshots

| Home | Product | Cart | Profile |
|------|---------|------|---------|
| <img src="screenshots/home.png" width="200"/> | <img src="screenshots/product.png" width="200"/> | <img src="screenshots/cart.png" width="200"/> | <img src="screenshots/profile.png" width="200"/> |

## ✨ Tính Năng

- **Authentication:** Email/Password, Google Sign-In, Biometric
- **Product:** Danh sách, filter, search với debounce
- **Cart:** Thêm/xóa, coupon code, tính giá tự động
- **Checkout:** Chọn địa chỉ trên Maps, đặt hàng
- **Real-time:** Theo dõi đơn hàng live với Firestore
- **Offline:** Wishlist lưu local với Hive
- **Notifications:** Push notification khi đơn hàng cập nhật
- **Dark Mode:** Giao diện tối/sáng theo system

## 🏗️ Tech Stack

| Layer | Technology |
|-------|------------|
| Language | Dart 3.5 |
| Framework | Flutter 3.24 |
| State Management | Riverpod 2.x + Code Gen |
| Navigation | Go Router 14 |
| HTTP Client | Dio 5.x |
| Local Storage | Hive + flutter_secure_storage |
| Backend | Firebase (Auth, Firestore, Storage, FCM) |
| Styling | Material 3 + NativeWind concepts |
| Validation | Formz |
| Maps | Google Maps Flutter |

## 🗂️ Cấu Trúc Dự Án

```
lib/
├── core/           # Shared utilities, network, router
├── features/
│   ├── auth/
│   ├── products/
│   ├── cart/
│   ├── checkout/
│   ├── orders/
│   └── profile/
└── main_dev.dart
```

## 🚀 Bắt Đầu

### Yêu Cầu

- Flutter 3.24+
- Dart 3.5+
- Android Studio / VS Code
- Firebase project

### Cài Đặt

```bash
# Clone
git clone https://github.com/username/shopmate-flutter.git
cd shopmate-flutter

# Install dependencies
flutter pub get

# Code generation
dart run build_runner build --delete-conflicting-outputs

# Copy config mẫu
cp config/dev.example.json config/dev.json
# Chỉnh sửa config/dev.json với thông tin của bạn

# Chạy
flutter run --flavor dev -t lib/main_dev.dart \
  --dart-define-from-file=config/dev.json
```

### Firebase Setup

1. Tạo Firebase project tại [console.firebase.google.com](https://console.firebase.google.com)
2. Chạy `flutterfire configure`
3. Bật Authentication (Email, Google)
4. Tạo Firestore database
5. Cập nhật `config/dev.json` với Firebase config

## 📋 Môi Trường

```json
// config/dev.example.json
{
  "APP_ENV": "dev",
  "API_BASE_URL": "https://dev-api.example.com",
  "FIREBASE_PROJECT_ID": "your-firebase-project"
}
```

## 🧪 Testing

```bash
# Unit tests
flutter test test/unit/

# Widget tests
flutter test test/widget/

# All tests với coverage
flutter test --coverage
```

## 📦 Build

```bash
# Android AAB
flutter build appbundle --flavor prod -t lib/main_prod.dart \
  --dart-define-from-file=config/prod.json

# iOS IPA
flutter build ipa --flavor prod -t lib/main_prod.dart \
  --dart-define-from-file=config/prod.json
```

## 👨‍💻 Tác Giả

**Nguyễn Văn A**
- GitHub: [@username](https://github.com/username)
- LinkedIn: [linkedin.com/in/username](https://linkedin.com/in/username)

## 📄 License

MIT License
````

---

## 17.5. Chuẩn Bị Phỏng Vấn Flutter Junior

### 17.5.1. Câu Hỏi Kỹ Thuật Thường Gặp

**Dart & Flutter Cơ Bản:**

```
Q: Sự khác biệt giữa StatelessWidget và StatefulWidget?
A: StatelessWidget không có state nội tại, UI chỉ phụ thuộc vào props.
   StatefulWidget có State object sống lâu dài, có thể gọi setState() để rebuild.
   Nên dùng StatelessWidget trước, chỉ chuyển StatefulWidget khi thực sự cần local state.

Q: Widget tree, Element tree và RenderObject tree khác nhau thế nào?
A: Widget: immutable blueprint, tạo mới mỗi lần build.
   Element: cầu nối sống, tái sử dụng để tránh tạo mới tốn kém.
   RenderObject: tính toán layout và vẽ pixel, tốn kém nhất.

Q: Null safety trong Dart là gì?
A: System đảm bảo NullPointerException không xảy ra nếu code compile.
   Mặc định mọi biến non-nullable. Thêm ? để nullable.
   Toán tử: ?. (null-aware), ?? (null coalescing), ! (force unwrap).

Q: Future vs Stream khác nhau thế nào?
A: Future: một giá trị trong tương lai, resolve một lần (như Promise).
   Stream: nhiều giá trị theo thời gian (như Observable/EventEmitter).
```

**State Management:**

```
Q: Tại sao dùng Riverpod thay vì setState?
A: setState không scale: prop drilling, state không đồng bộ giữa nhiều màn hình,
   business logic lẫn UI. Riverpod giải quyết bằng cách tách state ra khỏi widget,
   type-safe, dễ test, DI tích hợp sẵn.

Q: ref.watch vs ref.read khác nhau thế nào?
A: ref.watch: subscribe, widget rebuild khi provider thay đổi. Dùng trong build().
   ref.read: đọc một lần, không subscribe. Dùng trong callback, event handler.

Q: select() dùng để làm gì?
A: Giới hạn rebuild — chỉ rebuild khi phần được select thay đổi.
   Thay vì watch toàn CartProvider, chỉ watch totalQuantity.
   Giảm rebuild không cần thiết → tốt hơn cho performance.
```

**Navigation:**

```
Q: Tại sao dùng Go Router thay vì Navigator 1.0?
A: Go Router: URL-based, deep link, web support, declarative routes,
   nested navigation với ShellRoute, redirect/guard tích hợp.
   Navigator 1.0 thiếu những tính năng này.

Q: context.go() vs context.push() khác nhau thế nào?
A: go(): replace history stack, không thể back. Dùng cho tab switch, after login.
   push(): thêm vào history, có thể back. Dùng cho detail, modal.
```

**Performance:**

```
Q: Làm sao giảm widget rebuild?
A: 1. const constructor cho widget không phụ thuộc runtime value
   2. Tách widget nhỏ — giới hạn scope rebuild
   3. select() trong Riverpod — chỉ watch phần cần
   4. Consumer chỉ quanh widget cần rebuild, không quanh toàn màn hình

Q: Khi nào dùng compute()?
A: Khi có heavy computation block UI thread: parse JSON lớn (>500 items),
   image processing, complex sorting/filtering.
   compute() chạy function trên background isolate, không block UI.
```

### 17.5.2. Code Challenge Thường Gặp

```dart
// Challenge 1: Implement debounced search
class SearchState extends _$SearchState {
  Timer? _timer;

  @override
  String build() => '';

  void updateQuery(String query) {
    _timer?.cancel();
    _timer = Timer(const Duration(milliseconds: 500), () {
      state = query;
    });
  }
}

// Challenge 2: Infinite scroll
@riverpod
class InfiniteProductList extends _$InfiniteProductList {
  @override
  Future<List<Product>> build() async {
    return _fetchPage(1);
  }

  Future<void> loadMore() async {
    final current = state.valueOrNull ?? [];
    final nextPage = (current.length / 20).ceil() + 1;
    final more = await _fetchPage(nextPage);
    state = AsyncData([...current, ...more]);
  }

  Future<List<Product>> _fetchPage(int page) async {
    final result = await ref.read(productRepositoryProvider)
        .fetchProducts(page: page);
    return result.items;
  }
}

// Challenge 3: Optimistic update
class Cart extends _$Cart {
  void addItem(Product product) {
    // Cập nhật UI ngay lập tức (optimistic)
    final optimisticState = state.copyWith(
      items: [...state.items, CartItem(product: product, quantity: 1)],
    );
    state = optimisticState;

    // Gọi API background
    _syncWithServer(optimisticState).catchError((_) {
      // Rollback nếu API fail
      state = state; // Giữ state cũ
    });
  }
}
```

### 17.5.3. Hành Vi Trong Phỏng Vấn

```
✅ NÊN:
- Hỏi lại nếu không hiểu yêu cầu
- Nói to suy nghĩ khi giải quyết vấn đề
- Nêu trade-off của từng approach
- Nhận mình chưa biết thay vì bịa

✅ CHỦ ĐỀ CẦN CHUẨN BỊ:
- Walk-through dự án portfolio
- Giải thích kiến trúc: tại sao chọn cách này
- Một vấn đề khó gặp trong dự án và cách giải quyết
- Kế hoạch học Flutter tiếp theo

❌ TRÁNH:
- Copy code không hiểu
- Chỉ biết "làm theo tutorial"
- Không thể giải thích code của mình viết
```

---

## 17.6. Roadmap Phát Triển Sau Junior

```
Junior Flutter Developer (0-1 năm)
├── Nắm vững: Dart, Widget, Riverpod, Go Router, Dio, Firebase
├── Làm được: CRUD app, API integration, basic animation
└── Portfolio: 1-2 app hoàn chỉnh

↓

Mid Flutter Developer (1-3 năm)
├── Nắm vững: Clean Architecture, Testing, CI/CD, Performance
├── Làm được: Complex state, Custom painter, Platform channel
└── Thêm: 1 app production với 1000+ user

↓

Senior Flutter Developer (3+ năm)
├── Nắm vững: Flutter internals, Package development, Team lead
├── Làm được: Flutter plugin, multi-module architecture, mentor team
└── Đóng góp: Open source, conference talk

Kỹ năng bổ trợ theo thời gian:
├── Design: Figma, UI/UX principles
├── Backend: Firebase, Supabase, hoặc REST API design
├── DevOps: Fastlane, GitHub Actions, App Store optimization
└── Business: App monetization, analytics, A/B testing
```

---

## Tóm Tắt Toàn Bộ Giáo Trình

| Chương | Chủ đề | Kỹ năng đạt được |
|--------|--------|-----------------|
| 1 | Dart cơ bản | Null safety, OOP, async/await, sealed class |
| 2 | Flutter & Widget | Widget tree, lifecycle, layout, scroll |
| 3 | Styling & Theme | Material 3, ThemeData, dark mode, responsive |
| 4 | Navigation | Go Router, ShellRoute, deep link, guard |
| 5 | Riverpod | State management, DI, provider patterns |
| 6 | Dio & API | HTTP client, interceptor, repository pattern |
| 7 | Formz | Form validation, state machine, UX pattern |
| 8 | UI Nâng cao | M3 components, skeleton, empty/error state |
| 9 | Animation | Implicit, explicit, Hero, Lottie |
| 10 | Local Storage | SharedPrefs, SecureStorage, Hive, cache |
| 11 | Architecture | Clean arch, folder structure, DI, error handling |
| 12 | Testing | Unit, widget, golden, integration test |
| 13 | Flavors & ENV | Multi-environment, CI/CD, API key security |
| 14 | Native Platform | Firebase, FCM, camera, maps, biometric |
| 15 | Performance | DevTools, rebuild optimization, compute() |
| 16 | Deploy | Android/iOS signing, Play Store, TestFlight |
| 17 | Portfolio | Dự án thực tế, phỏng vấn, roadmap |

> **Lời kết:** Flutter là framework đang phát triển rất nhanh. Phiên bản, API và best practice thay đổi thường xuyên. Thói quen quan trọng nhất: đọc release notes mỗi khi Flutter ra version mới, theo dõi Flutter community (pub.dev, r/FlutterDev, Flutter Discord), và luôn cập nhật dependencies. Code đẹp nhất là code dễ đọc, dễ test, và dễ thay đổi — không phải code ngắn nhất hay dùng nhiều tính năng nhất.

**Chúc bạn thành công trên hành trình Flutter! 🎯**
