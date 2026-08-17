# Chương 15: Performance & Debug

---

## 15.1. Tư Duy Về Performance Trong Flutter

### 15.1.1. Flutter Render Pipeline

Để tối ưu performance, cần hiểu Flutter render frame như thế nào. Mỗi frame phải hoàn thành trong **16ms** (60fps) hoặc **8ms** (120fps). Nếu vượt quá ngưỡng này, frame bị drop — người dùng thấy giật (jank).

```
Input Events → Build → Layout → Paint → Composite → GPU
     ↑                                                ↓
     └──────────── 16ms per frame ───────────────────┘
```

**Build phase:** Dart code chạy — widget tree được tạo lại. Đây là nơi developer có thể tối ưu nhiều nhất.

**Layout phase:** RenderObject tính toán kích thước và vị trí.

**Paint phase:** RenderObject vẽ lên canvas.

**Composite phase:** Các layer được ghép lại và gửi lên GPU.

### 15.1.2. Các Nguyên Nhân Jank Phổ Biến

| Nguyên nhân | Biểu hiện | Giải pháp |
|---|---|---|
| Rebuild không cần thiết | Toàn màn hình rebuild khi 1 widget nhỏ thay đổi | `const`, `select()`, tách widget |
| Heavy computation trên main thread | UI freeze khi xử lý data | `compute()` hoặc `Isolate` |
| Image decode không cache | Chậm khi scroll list có ảnh | `memCacheWidth/Height`, `cached_network_image` |
| Layout thrashing | `intrinsicHeight/Width` trong list | Tránh dùng, dùng kích thước cố định |
| Overdraw | Vẽ pixel bị che khuất nhiều lần | `RepaintBoundary`, tối giản decoration |
| Shader compilation jank | Giật lần đầu mở màn hình | Shader warm-up |

---

## 15.2. Flutter DevTools

### 15.2.1. Khởi Chạy DevTools

```bash
# Chạy app ở debug mode
flutter run

# Mở DevTools trong browser
flutter pub global activate devtools
flutter pub global run devtools

# Hoặc VS Code: Cmd+Shift+P → "Flutter: Open DevTools"
# Hoặc click link "A Dart DevTools debugger and profiler" trong terminal
```

### 15.2.2. Widget Inspector

Widget Inspector giúp debug UI mà không cần đọc code:

```
Tính năng chính:
├── Select widget mode: tap vào widget trên màn hình → highlight trong tree
├── Widget tree: xem toàn bộ widget hierarchy
├── Details pane: xem property của widget đang chọn
├── Layout explorer: visualize Flex/Column/Row sizing
└── Slow animations: giảm tốc animation 5x để xem kỹ
```

**Workflow debug UI:**
1. Bật "Select Widget Mode"
2. Tap vào widget bị vấn đề trên màn hình
3. DevTools highlight widget trong tree
4. Xem constraints trong Layout Explorer
5. Kiểm tra padding, size, color trong Details pane

### 15.2.3. Performance Profiler

```
Performance tab:
├── Frame chart: mỗi cột = 1 frame
│   ├── UI thread (xanh): Build + Layout + Paint
│   └── Raster thread (xanh lá): Composite + GPU
│   
├── Frame bị drop: cột màu đỏ vượt quá 16ms
│   
└── Flame chart: xem function nào tốn nhiều thời gian
    ├── Build: widget nào rebuild tốn nhiều nhất
    ├── Layout: widget nào layout phức tạp
    └── Paint: widget nào paint tốn kém
```

**Cách đọc Flame Chart:**
- Chiều rộng cột = thời gian function đó tốn
- Stack càng cao = call stack càng sâu
- Tìm cột rộng nhất ở tầng cao nhất → đó là bottleneck

### 15.2.4. Memory Profiler

```dart
// Detect memory leak: theo dõi object count tăng không giảm

// Trong DevTools → Memory tab:
// 1. Take heap snapshot
// 2. Perform action (navigate, scroll, etc.)
// 3. Take another heap snapshot
// 4. Compare: nếu object count tăng liên tục → có leak

// Nguyên nhân leak phổ biến:
// - Không dispose AnimationController
// - Không cancel StreamSubscription
// - Không dispose TextEditingController
// - Closure giữ reference đến BuildContext
```

---

## 15.3. Kỹ Thuật Tối Ưu

### 15.3.1. const Widget — Tối Ưu Quan Trọng Nhất

```dart
// ❌ Tệ — Widget tạo mới mỗi lần rebuild parent
Widget build(BuildContext context) {
  return Column(
    children: [
      Padding(
        padding: EdgeInsets.all(16), // Object mới mỗi lần
        child: Text('Tiêu đề'),      // Object mới mỗi lần
      ),
    ],
  );
}

// ✅ Tốt — const: Flutter tái sử dụng widget, không tạo mới
Widget build(BuildContext context) {
  return Column(
    children: [
      const Padding(
        padding: EdgeInsets.all(16),  // Tạo một lần duy nhất
        child: Text('Tiêu đề'),       // Tạo một lần duy nhất
      ),
    ],
  );
}

// Bật lint rule để cảnh báo khi quên const
// analysis_options.yaml
// lints:
//   prefer_const_constructors: true
//   prefer_const_literals_to_create_immutables: true
```

### 15.3.2. Tách Widget Nhỏ — Giới Hạn Rebuild Scope

```dart
// ❌ Tệ — Cả màn hình rebuild khi cart count thay đổi
class HomeScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final cartCount = ref.watch(cartItemCountProvider); // Trigger rebuild toàn screen

    return Scaffold(
      appBar: AppBar(
        actions: [
          Badge.count(count: cartCount, child: const Icon(Icons.shopping_cart)),
        ],
      ),
      body: const ProductGrid(), // Toàn bộ này rebuild dù không liên quan cart
    );
  }
}

// ✅ Tốt — Chỉ CartBadge rebuild, ProductGrid không bị ảnh hưởng
class HomeScreen extends StatelessWidget {  // Không cần Consumer ở đây
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        actions: const [CartBadge()], // Widget riêng xử lý cart state
      ),
      body: const ProductGrid(),      // const — không bao giờ rebuild
    );
  }
}

class CartBadge extends ConsumerWidget {  // Chỉ widget nhỏ này watch cart
  const CartBadge({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(cartItemCountProvider);
    return Badge.count(count: count, child: const Icon(Icons.shopping_cart));
  }
}
```

### 15.3.3. ListView.builder — Lazy Rendering

```dart
// ❌ Tệ — Tạo 1000 widget cùng lúc dù chỉ hiển thị 10
ListView(
  children: products.map((p) => ProductCard(product: p)).toList(),
)

// ✅ Tốt — Chỉ tạo widget khi xuất hiện trên màn hình
ListView.builder(
  itemCount: products.length,
  // itemExtent: nếu tất cả item cùng chiều cao → Flutter tính scroll position nhanh hơn
  itemExtent: 120.0,
  itemBuilder: (context, index) => ProductCard(product: products[index]),
)

// ✅ Còn tốt hơn với GridView
GridView.builder(
  gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 2,
    childAspectRatio: 0.75,
  ),
  itemCount: products.length,
  itemBuilder: (context, index) => ProductCard(product: products[index]),
)
```

### 15.3.4. RepaintBoundary — Giới Hạn Vùng Repaint

```dart
// RepaintBoundary tạo layer riêng cho widget
// Widget bên ngoài animate/thay đổi không khiến widget bên trong repaint

// ✅ Dùng cho widget animate phức tạp nhưng content không thay đổi
class AnimatedBackground extends StatefulWidget {
  const AnimatedBackground({super.key, required this.child});
  final Widget child;

  @override
  State<AnimatedBackground> createState() => _AnimatedBackgroundState();
}

class _AnimatedBackgroundState extends State<AnimatedBackground>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(vsync: this, duration: const Duration(seconds: 3))
      ..repeat(reverse: true);
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) {
        return DecoratedBox(
          decoration: BoxDecoration(
            gradient: LinearGradient(
              colors: [
                Color.lerp(Colors.blue, Colors.purple, _controller.value)!,
                Color.lerp(Colors.purple, Colors.pink, _controller.value)!,
              ],
            ),
          ),
          // child được bọc trong RepaintBoundary
          // → background animate nhưng child KHÔNG bị repaint
          child: child,
        );
      },
      child: RepaintBoundary(child: widget.child),
    );
  }
}

// ✅ Dùng cho list item phức tạp
ListView.builder(
  itemBuilder: (context, index) => RepaintBoundary(
    // Mỗi item có layer riêng — scroll không khiến toàn list repaint
    child: ComplexListItem(item: items[index]),
  ),
)
```

### 15.3.5. compute() — Offload to Background Isolate

```dart
// compute(): chạy function thuần Dart trên background isolate
// Không block UI thread
// Giới hạn: function phải là top-level hoặc static, tham số phải serializable

// ✅ Xử lý JSON lớn trên background
Future<List<Product>> parseProductsInBackground(String jsonString) async {
  // compute trả về Future — gọi await để đợi kết quả
  return compute(_parseProducts, jsonString);
}

// Function này chạy trên background isolate — KHÔNG được dùng Flutter/BuildContext
List<Product> _parseProducts(String jsonString) {
  final list = jsonDecode(jsonString) as List;
  return list.map((e) => Product.fromJson(e as Map<String, dynamic>)).toList();
}

// ✅ Resize ảnh trên background
Future<Uint8List> resizeImageInBackground(Uint8List imageBytes) async {
  return compute(_resizeImage, imageBytes);
}

Uint8List _resizeImage(Uint8List bytes) {
  final image = img.decodeImage(bytes)!;
  final resized = img.copyResize(image, width: 800);
  return Uint8List.fromList(img.encodeJpg(resized, quality: 85));
}

// ✅ Lọc và sort danh sách lớn
Future<List<Product>> filterAndSortProducts({
  required List<Product> products,
  required FilterParams params,
}) {
  return compute(_filterAndSort, (products, params));
}

List<Product> _filterAndSort((List<Product>, FilterParams) args) {
  final (products, params) = args;
  return products
      .where((p) {
        if (params.category != null && p.categoryId != params.category) return false;
        if (p.price < params.minPrice || p.price > params.maxPrice) return false;
        return true;
      })
      .toList()
    ..sort((a, b) => params.sortAscending
        ? a.price.compareTo(b.price)
        : b.price.compareTo(a.price));
}
```

### 15.3.6. Image Optimization

```dart
// ✅ memCacheWidth/Height: resize ảnh trong memory
// Tránh load ảnh 2000x2000 vào RAM khi chỉ hiển thị 200x200

// Trong GridView (2 cột, mỗi cell ~200px):
CachedNetworkImage(
  imageUrl: product.imageUrl,
  memCacheWidth: 400,  // 2x cho high DPI display
  memCacheHeight: 400,
  fit: BoxFit.cover,
)

// Trong list (full width ~400px):
CachedNetworkImage(
  imageUrl: article.imageUrl,
  memCacheWidth: 800,
  memCacheHeight: 450,
  fit: BoxFit.cover,
)

// Precache ảnh quan trọng trước khi hiển thị
@override
void didChangeDependencies() {
  super.didChangeDependencies();
  // Precache ảnh khi screen load, trước khi user scroll đến
  for (final product in featuredProducts) {
    precacheImage(
      CachedNetworkImageProvider(product.imageUrl),
      context,
    );
  }
}
```

### 15.3.7. Riverpod select() — Tối Ưu Rebuild

```dart
// ❌ Tệ — rebuild mỗi khi bất kỳ thứ gì trong cart thay đổi
final cart = ref.watch(cartProvider);
Text('${cart.totalQuantity} sản phẩm')

// ✅ Tốt — chỉ rebuild khi totalQuantity thay đổi
final count = ref.watch(cartProvider.select((s) => s.totalQuantity));
Text('$count sản phẩm')

// ✅ Tốt — chỉ rebuild khi isInCart thay đổi cho product này
final isInCart = ref.watch(
  cartProvider.select((s) => s.items.any((i) => i.product.id == productId)),
);
```

---

## 15.4. Phát Hiện Vấn Đề

### 15.4.1. Debug Paint

```dart
// main.dart — Bật debug paint để thấy layout trực quan
void main() {
  // Chỉ bật trong debug mode
  if (kDebugMode) {
    debugPaintSizeEnabled = true;    // Hiển thị bounding box của mọi widget
    debugPaintBaselinesEnabled = true; // Hiển thị text baseline
    debugRepaintRainbowEnabled = true; // Widget repaint hiển thị màu ngẫu nhiên
    // Nếu màu thay đổi liên tục khi scroll/animate → widget repaint quá nhiều
  }
  runApp(const ProviderScope(child: MyApp()));
}
```

### 15.4.2. Timeline Events — Custom Profiling

```dart
// Đánh dấu code block trong profiler để dễ identify
import 'dart:developer' as dev;

Future<List<Product>> loadProducts() async {
  dev.Timeline.startSync('Load Products'); // Bắt đầu đánh dấu
  try {
    final products = await repository.fetchProducts();
    dev.Timeline.startSync('Parse Products');
    final parsed = _parseProducts(products);
    dev.Timeline.finishSync();
    return parsed;
  } finally {
    dev.Timeline.finishSync(); // Kết thúc đánh dấu
  }
}

// log() thay cho print() — xuất hiện trong DevTools Logging tab
dev.log(
  'Loaded ${products.length} products',
  name: 'ProductRepository',
  time: DateTime.now(),
);
```

### 15.4.3. assert() — Debug-Only Checks

```dart
// assert() chỉ chạy trong debug mode — không có overhead trong production
class CartItem {
  CartItem({required this.product, required this.quantity}) {
    assert(quantity > 0, 'Quantity phải lớn hơn 0, nhận được: $quantity');
    assert(product.stock >= quantity,
        'Không đủ hàng: cần $quantity nhưng chỉ có ${product.stock}');
  }

  final Product product;
  final int quantity;
}

// assert trong widget để kiểm tra constraints
class ProductCard extends StatelessWidget {
  const ProductCard({
    super.key,
    required this.product,
    this.width,
  }) : assert(width == null || width > 0, 'Width phải dương hoặc null');

  final Product product;
  final double? width;
}
```

---

## 15.5. Error Tracking Trong Production

### 15.5.1. Sentry Integration

```yaml
# pubspec.yaml
dependencies:
  sentry_flutter: ^8.0.0
```

```dart
// main.dart — Sentry thay thế hoặc bổ sung cho Crashlytics
Future<void> main() async {
  await SentryFlutter.init(
    (options) {
      options.dsn = AppConfig.sentryDsn;
      options.environment = AppConfig.env;     // 'dev', 'staging', 'prod'
      options.release = '${AppConfig.appName}@${AppConfig.appVersion}';

      // Tỷ lệ sample performance traces (0.0 - 1.0)
      options.tracesSampleRate = AppConfig.isProd ? 0.2 : 1.0;

      // Chụp screenshot khi crash
      options.attachScreenshot = true;
      options.screenshotQuality = SentryScreenshotQuality.medium;

      // Không gửi PII trong production
      options.sendDefaultPii = !AppConfig.isProd;
    },
    appRunner: () => runApp(
      const ProviderScope(child: MyApp()),
    ),
  );
}

// Gắn thông tin user
Future<void> setUser(AppUser user) async {
  await Sentry.configureScope((scope) {
    scope.setUser(SentryUser(
      id: user.id,
      email: AppConfig.isProd ? null : user.email, // Ẩn email trong prod
    ));
  });
}

// Log lỗi thủ công với context
Future<void> captureError(Object error, StackTrace? stackTrace, {
  String? context,
  Map<String, dynamic>? extras,
}) async {
  await Sentry.captureException(
    error,
    stackTrace: stackTrace,
    withScope: (scope) {
      if (context != null) scope.setTag('context', context);
      if (extras != null) {
        extras.forEach((key, value) => scope.setExtra(key, value));
      }
    },
  );
}
```

### 15.5.2. Custom Error Widget

```dart
// Thay thế màn hình đỏ xấu xí của Flutter trong production

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    // Custom error widget thay vì màn hình đỏ
    ErrorWidget.builder = (details) {
      // Trong debug: giữ màn hình đỏ với stack trace
      if (kDebugMode) return ErrorWidget(details.exception);

      // Trong production: hiển thị UI thân thiện
      return MaterialApp(
        home: Scaffold(
          body: Center(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                const Icon(Icons.error_outline, size: 64, color: Colors.red),
                const SizedBox(height: 16),
                const Text('Đã xảy ra lỗi không mong đợi'),
                const SizedBox(height: 24),
                ElevatedButton(
                  onPressed: () {
                    // Restart app
                    Phoenix.rebirth(context);
                  },
                  child: const Text('Khởi động lại'),
                ),
              ],
            ),
          ),
        ),
      );
    };

    return MaterialApp.router(routerConfig: appRouter);
  }
}
```

---

## 15.6. App Size Optimization

```bash
# Phân tích size của APK/AAB
flutter build apk --analyze-size
flutter build appbundle --analyze-size

# Obfuscate code — giảm size và bảo vệ code
flutter build apk \
  --obfuscate \
  --split-debug-info=build/debug-info

# Split APK theo ABI — giảm size download đáng kể
flutter build apk --split-per-abi
# Sinh ra: armeabi-v7a, arm64-v8a, x86_64

# Xem breakdown size
flutter build apk --target-platform android-arm64 --analyze-size
```

```yaml
# pubspec.yaml — loại bỏ asset không dùng
flutter:
  assets:
    - assets/images/    # Chỉ include folder cần thiết
  fonts:
    - family: Inter
      fonts:
        - asset: assets/fonts/Inter-Regular.ttf  # Chỉ weight đang dùng
        - asset: assets/fonts/Inter-Medium.ttf
        # Bỏ các weight không dùng
```

---

## 15.7. Bài Tập: Debug Jank Thực Tế

Scenario: ProductGrid bị jank khi scroll. Quy trình debug:

```dart
// Bước 1: Bật Performance Overlay
MaterialApp(
  showPerformanceOverlay: true, // Bật trong main_dev.dart
  // ...
)

// Bước 2: Bật debugRepaintRainbowEnabled — xem widget nào repaint liên tục
debugRepaintRainbowEnabled = kDebugMode;

// Bước 3: Identify vấn đề — ví dụ ProductCard rebuild liên tục

// ❌ Nguyên nhân: Provider watch quá rộng
class ProductCard extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final cart = ref.watch(cartProvider); // Rebuild khi BẤT KỲ cart change
    final isInCart = cart.items.any((i) => i.product.id == widget.product.id);
    // ...
  }
}

// ✅ Fix: Dùng select() để giới hạn rebuild
class ProductCard extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final isInCart = ref.watch(
      cartProvider.select((cart) =>  // Chỉ rebuild khi isInCart thay đổi
          cart.items.any((i) => i.product.id == widget.product.id)),
    );
    // ...
  }
}

// Bước 4: Thêm RepaintBoundary cho list item
ListView.builder(
  itemBuilder: (context, index) => RepaintBoundary(
    child: ProductCard(product: products[index]),
  ),
)

// Bước 5: Thêm memCacheWidth cho ảnh
CachedNetworkImage(
  memCacheWidth: 400,
  memCacheHeight: 400,
)

// Bước 6: Verify fix bằng Performance Profiler — frame time phải giảm
```

---

## Tóm Tắt Chương 15

| Kỹ thuật | Tác động | Khi nào dùng |
|---|---|---|
| `const` constructor | Cao — tránh tạo object | Luôn luôn khi có thể |
| Tách widget nhỏ | Cao — giới hạn rebuild scope | Widget có state/provider |
| `ListView.builder` | Cao — lazy render | Mọi list > 20 item |
| `select()` | Trung bình — giảm rebuild | Watch một phần của provider |
| `RepaintBoundary` | Trung bình — giới hạn repaint | Animate, heavy item |
| `compute()` | Cao — unblock UI thread | JSON parse lớn, image resize |
| `memCacheWidth/Height` | Cao — giảm RAM | Mọi ảnh trong list/grid |
| `itemExtent` | Trung bình — tính scroll nhanh | List item cùng chiều cao |

| DevTools Tab | Dùng để |
|---|---|
| Widget Inspector | Debug layout, size, constraints |
| Performance | Tìm frame drop, flame chart bottleneck |
| Memory | Detect memory leak, object count |
| Network | Xem HTTP request, response time |
| Logging | `dev.log()` output, error trace |

> **Quy tắc tối ưu:** Không tối ưu sớm (premature optimization). Trước tiên, làm cho app hoạt động đúng. Sau đó đo lường bằng DevTools để tìm bottleneck thực sự. Chỉ tối ưu phần đo được là vấn đề. Tối ưu mà không đo là phỏng đoán — thường tốn công nhưng không cải thiện gì đáng kể.
