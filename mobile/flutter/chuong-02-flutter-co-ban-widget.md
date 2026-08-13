# Chương 2: Flutter Cơ Bản & Hệ Thống Widget

---

## 2.1. Tổng Quan Kiến Trúc Flutter

### 2.1.1. Flutter Là Gì?

Flutter là một UI toolkit đa nền tảng do Google phát triển, cho phép xây dựng ứng dụng cho mobile (Android, iOS), web, desktop và embedded từ một codebase duy nhất. Điểm khác biệt căn bản của Flutter so với các framework như React Native hay Xamarin nằm ở cách nó render giao diện: **Flutter không sử dụng native UI component của hệ điều hành**, mà thay vào đó tự vẽ toàn bộ pixel thông qua Skia (và Impeller từ Flutter 3.x trở đi) — một engine đồ họa 2D hiệu năng cao.

Kiến trúc tổng thể của Flutter bao gồm ba tầng chính:

```
┌────────────────────────────────────┐
│         Framework (Dart)           │  ← Developer làm việc ở đây
│   Material / Cupertino / Widgets   │
│   Rendering / Animation / Gestures │
├────────────────────────────────────┤
│            Engine (C++)            │  ← Skia / Impeller, text layout
│     Dart runtime, Platform APIs    │
├────────────────────────────────────┤
│         Embedder (Platform)        │  ← Android / iOS / macOS / ...
└────────────────────────────────────┘
```

**Hệ quả thực tiễn:** Vì Flutter tự vẽ pixel, UI trông hoàn toàn nhất quán trên mọi nền tảng và phiên bản OS. Không có rủi ro "trông khác nhau trên Android 10 và Android 13" do thay đổi native component.

---

### 2.1.2. Ba Cây Cốt Lõi: Widget Tree, Element Tree, RenderObject Tree

Đây là kiến thức nền tảng quan trọng nhất để hiểu Flutter hoạt động như thế nào, và cũng là kiến thức thường bị bỏ qua khi học từ tutorial.

#### Widget Tree — Bản Thiết Kế Bất Biến

Widget trong Flutter là một **đối tượng bất biến (immutable)** mô tả một phần của giao diện. Khi state thay đổi, Flutter không chỉnh sửa widget cũ mà tạo ra một widget tree hoàn toàn mới. Đây là thiết kế cố ý, cho phép Flutter tối ưu hóa việc so sánh thay đổi.

```dart
// Widget tree này được tạo mới hoàn toàn mỗi khi build() gọi lại
// Flutter KHÔNG sửa widget cũ — nó tạo object mới
Widget build(BuildContext context) {
  return Column(
    children: [
      Text('Hello'),     // Widget object mới
      ElevatedButton(    // Widget object mới
        onPressed: () {},
        child: Text('Click'),
      ),
    ],
  );
}
```

#### Element Tree — Cầu Nối Sống Động

Element là đối tượng **có trạng thái (mutable)** tồn tại xuyên suốt vòng đời của widget. Khi widget tree được tạo mới, Flutter không tạo element mới mà **tái sử dụng element cũ** nếu widget type không thay đổi, chỉ cập nhật tham chiếu sang widget mới.

Cơ chế này giải thích tại sao Flutter nhanh dù widget tree liên tục được tái tạo: chi phí tạo widget object (Dart object) rất thấp, còn element và renderobject — phần tốn kém — được tái sử dụng tối đa.

#### RenderObject Tree — Người Thực Thi

RenderObject là nơi tính toán layout (vị trí, kích thước) và thực sự vẽ pixel. Tầng này tốn nhiều tài nguyên nhất. Flutter cố gắng tối thiểu hóa số lần cập nhật RenderObject bằng cách so sánh widget tree (diffing) để tìm ra phần thực sự thay đổi.

```
Widget Tree       Element Tree      RenderObject Tree
(blueprint)       (live bridge)     (actual rendering)

Column      →     ColumnElement  →  RenderFlex
├─ Text     →     TextElement    →  RenderParagraph
└─ Button   →     ButtonElement  →  RenderBox
```

**Ý nghĩa thực tiễn:** Hiểu ba cây này giúp lý giải vì sao dùng `const` constructor tiết kiệm chi phí (widget không được tái tạo), và tại sao đặt `key` đúng chỗ lại quan trọng khi reorder list.

---

## 2.2. StatelessWidget và StatefulWidget

### 2.2.1. StatelessWidget — Widget Không Có Trạng Thái Nội Tại

`StatelessWidget` là widget mà giao diện chỉ phụ thuộc vào các tham số (props) được truyền vào từ bên ngoài. Nó không tự lưu trữ trạng thái nào.

```dart
// ✅ CHUẨN — StatelessWidget cho component thuần hiển thị
class UserAvatar extends StatelessWidget {
  const UserAvatar({
    super.key,
    required this.imageUrl,
    required this.displayName,
    this.radius = 24.0,
  });

  final String imageUrl;
  final String displayName;
  final double radius;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);

    return Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        CircleAvatar(
          radius: radius,
          backgroundImage: NetworkImage(imageUrl),
          onBackgroundImageError: (_, __) {},
          child: imageUrl.isEmpty
              ? Text(
                  displayName.isNotEmpty ? displayName[0].toUpperCase() : '?',
                  style: theme.textTheme.titleMedium?.copyWith(
                    color: theme.colorScheme.onPrimary,
                  ),
                )
              : null,
        ),
        const SizedBox(height: 4),
        Text(
          displayName,
          style: theme.textTheme.labelMedium,
          maxLines: 1,
          overflow: TextOverflow.ellipsis,
        ),
      ],
    );
  }
}
```

**Quy tắc sử dụng:** Bắt đầu với `StatelessWidget`. Chỉ chuyển sang `StatefulWidget` khi có nhu cầu lưu trạng thái cục bộ thực sự (animation controller, text controller, scroll controller, tab index tạm thời...).

---

### 2.2.2. StatefulWidget — Widget Có Vòng Đời và Trạng Thái

`StatefulWidget` gồm hai class: bản thân widget (immutable, bị tái tạo thường xuyên) và `State` object (mutable, tồn tại lâu dài). Sự tách biệt này là chủ ý của thiết kế Flutter.

```dart
// ✅ CHUẨN — StatefulWidget cho component có local state
class QuantitySelector extends StatefulWidget {
  const QuantitySelector({
    super.key,
    required this.initialValue,
    required this.onChanged,
    this.min = 1,
    this.max = 99,
  });

  final int initialValue;
  final ValueChanged<int> onChanged;
  final int min;
  final int max;

  @override
  State<QuantitySelector> createState() => _QuantitySelectorState();
}

class _QuantitySelectorState extends State<QuantitySelector> {
  late int _quantity;

  @override
  void initState() {
    super.initState();
    // initState: gọi MỘT LẦN khi State được tạo
    // Dùng để khởi tạo giá trị ban đầu, controllers, subscriptions
    _quantity = widget.initialValue;
  }

  @override
  void didUpdateWidget(QuantitySelector oldWidget) {
    super.didUpdateWidget(oldWidget);
    // didUpdateWidget: gọi khi widget cha rebuild với tham số mới
    // Dùng để đồng bộ state khi props từ bên ngoài thay đổi
    if (oldWidget.initialValue != widget.initialValue) {
      _quantity = widget.initialValue;
    }
  }

  @override
  void dispose() {
    // dispose: gọi khi widget bị remove khỏi tree
    // Dùng để giải phóng: controller, subscription, timer, stream
    super.dispose();
  }

  void _increment() {
    if (_quantity >= widget.max) return;
    setState(() => _quantity++);
    widget.onChanged(_quantity);
  }

  void _decrement() {
    if (_quantity <= widget.min) return;
    setState(() => _quantity--);
    widget.onChanged(_quantity);
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);

    return Row(
      mainAxisSize: MainAxisSize.min,
      children: [
        IconButton.outlined(
          onPressed: _quantity <= widget.min ? null : _decrement,
          icon: const Icon(Icons.remove),
          iconSize: 18,
        ),
        Padding(
          padding: const EdgeInsets.symmetric(horizontal: 12),
          child: Text(
            '$_quantity',
            style: theme.textTheme.titleMedium,
          ),
        ),
        IconButton.outlined(
          onPressed: _quantity >= widget.max ? null : _increment,
          icon: const Icon(Icons.add),
          iconSize: 18,
        ),
      ],
    );
  }
}
```

### 2.2.3. Vòng Đời Đầy Đủ của State

```
createState()
     ↓
initState()          ← Khởi tạo: controller, listen stream, set initial value
     ↓
didChangeDependencies() ← Khi InheritedWidget (Theme, MediaQuery...) thay đổi
     ↓
build()              ← Render UI. Gọi lại mỗi khi setState() hoặc dep thay đổi
     ↓
didUpdateWidget()    ← Widget cha rebuild với props mới
     ↓
setState() → build() ← Vòng lặp chính
     ↓
deactivate()         ← Widget tạm thời bị remove (GlobalKey move)
     ↓
dispose()            ← Widget bị remove vĩnh viễn. Giải phóng tài nguyên!
```

> **Lỗi phổ biến:** Quên gọi `dispose()` cho `TextEditingController`, `AnimationController`, `StreamSubscription`. Dẫn đến memory leak, và Flutter sẽ cảnh báo trong debug mode.

---

## 2.3. Hệ Thống Layout

Hiểu layout trong Flutter là hiểu **giao thức "constraints đi xuống, sizes đi lên"**. Parent truyền constraints (min/max width, min/max height) xuống child. Child tự quyết định kích thước của mình trong giới hạn đó và báo lại cho parent. Parent dùng kích thước đó để định vị child.

### 2.3.1. Column và Row

`Column` và `Row` là hai layout widget sử dụng nhiều nhất. Cả hai đều implement `Flex` layout, giống với Flexbox trong CSS nhưng có vài điểm khác biệt quan trọng.

```dart
// ✅ CHUẨN — Column với alignment và spacing đúng cách
class ProductCard extends StatelessWidget {
  const ProductCard({
    super.key,
    required this.product,
  });

  final Product product;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);

    return Card(
      clipBehavior: Clip.antiAlias,
      child: Column(
        // mainAxisAlignment: Căn chỉnh dọc theo trục chính (dọc với Column)
        // crossAxisAlignment: Căn chỉnh theo trục ngang
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          // AspectRatio đảm bảo ảnh luôn tỷ lệ 16:9
          AspectRatio(
            aspectRatio: 16 / 9,
            child: Image.network(
              product.imageUrl,
              fit: BoxFit.cover,
            ),
          ),
          Padding(
            padding: const EdgeInsets.all(12),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(
                  product.name,
                  style: theme.textTheme.titleSmall,
                  maxLines: 2,
                  overflow: TextOverflow.ellipsis,
                ),
                const SizedBox(height: 4),
                Row(
                  // mainAxisAlignment trên Row: căn chỉnh theo chiều ngang
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    Text(
                      product.formattedPrice,
                      style: theme.textTheme.titleMedium?.copyWith(
                        color: theme.colorScheme.primary,
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                    // Expanded chiếm toàn bộ không gian còn lại
                    // Dùng trong Row/Column, không dùng ngoài Flex
                    FilledButton.tonal(
                      onPressed: () {},
                      child: const Text('Thêm'),
                    ),
                  ],
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

**So sánh `Expanded` vs `Flexible`:**

| Thuộc tính | `Expanded` | `Flexible` |
|---|---|---|
| Hành vi mặc định | Chiếm toàn bộ không gian còn lại | Có thể nhỏ hơn không gian được cấp |
| `fit` | `FlexFit.tight` (bắt buộc) | `FlexFit.loose` (mặc định) |
| Dùng khi | Muốn child lấp đầy | Muốn child linh hoạt nhưng không ép |

```dart
// Ví dụ phân biệt Expanded vs Flexible
Row(
  children: [
    // Flexible: TextField chiếm không gian còn lại nhưng
    // không bị ép phải rộng hơn nội dung cần
    Flexible(
      child: TextField(
        decoration: InputDecoration(hintText: 'Tìm kiếm...'),
      ),
    ),
    const SizedBox(width: 8),
    // Widget cố định không dùng Expanded/Flexible
    ElevatedButton(
      onPressed: () {},
      child: const Text('Tìm'),
    ),
  ],
),
```

---

### 2.3.2. Stack — Layout Chồng Lớp

`Stack` cho phép các widget chồng lên nhau. Tương tự `position: absolute` trong CSS, nhưng được kiểm soát qua `Positioned` widget.

```dart
// ✅ CHUẨN — Stack cho badge notification trên avatar
class BadgedAvatar extends StatelessWidget {
  const BadgedAvatar({
    super.key,
    required this.imageUrl,
    required this.badgeCount,
  });

  final String imageUrl;
  final int badgeCount;

  @override
  Widget build(BuildContext context) {
    return Stack(
      // clipBehavior: Clip.none cho phép badge tràn ra ngoài Stack boundary
      clipBehavior: Clip.none,
      children: [
        CircleAvatar(
          radius: 24,
          backgroundImage: NetworkImage(imageUrl),
        ),
        // Positioned: đặt child tại vị trí tuyệt đối trong Stack
        if (badgeCount > 0)
          Positioned(
            top: -4,
            right: -4,
            child: Badge.count(
              count: badgeCount,
            ),
          ),
      ],
    );
  }
}
```

---

### 2.3.3. Wrap — Layout Tự Xuống Dòng

`Wrap` tương tự `Row` nhưng tự động xuống dòng khi không đủ không gian. Lý tưởng cho tag, chip, filter.

```dart
// ✅ CHUẨN — Wrap cho danh sách filter chip
class FilterChipGroup extends StatelessWidget {
  const FilterChipGroup({
    super.key,
    required this.options,
    required this.selected,
    required this.onToggle,
  });

  final List<String> options;
  final Set<String> selected;
  final ValueChanged<String> onToggle;

  @override
  Widget build(BuildContext context) {
    return Wrap(
      spacing: 8,         // Khoảng cách ngang giữa các chip
      runSpacing: 8,      // Khoảng cách dọc giữa các dòng
      children: options.map((option) {
        return FilterChip(
          label: Text(option),
          selected: selected.contains(option),
          onSelected: (_) => onToggle(option),
        );
      }).toList(),
    );
  }
}
```

---

## 2.4. Widget Hiển Thị Nội Dung

### 2.4.1. Text và RichText

```dart
// ✅ CHUẨN — Text với style đúng cách từ Theme
class PriceDisplay extends StatelessWidget {
  const PriceDisplay({
    super.key,
    required this.originalPrice,
    required this.discountedPrice,
    required this.discountPercent,
  });

  final double originalPrice;
  final double discountedPrice;
  final int discountPercent;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);

    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        // RichText: Nhiều style trong một dòng
        RichText(
          text: TextSpan(
            children: [
              TextSpan(
                text: '${discountedPrice.toStringAsFixed(0)}đ',
                style: theme.textTheme.headlineSmall?.copyWith(
                  color: theme.colorScheme.error,
                  fontWeight: FontWeight.bold,
                ),
              ),
              const TextSpan(text: '  '),
              TextSpan(
                // Text gạch ngang cho giá gốc
                text: '${originalPrice.toStringAsFixed(0)}đ',
                style: theme.textTheme.bodyMedium?.copyWith(
                  color: theme.colorScheme.outline,
                  decoration: TextDecoration.lineThrough,
                ),
              ),
            ],
          ),
        ),
        const SizedBox(height: 4),
        // Badge giảm giá
        Container(
          padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 2),
          decoration: BoxDecoration(
            color: theme.colorScheme.errorContainer,
            borderRadius: BorderRadius.circular(4),
          ),
          child: Text(
            '-$discountPercent%',
            style: theme.textTheme.labelSmall?.copyWith(
              color: theme.colorScheme.onErrorContainer,
              fontWeight: FontWeight.bold,
            ),
          ),
        ),
      ],
    );
  }
}
```

### 2.4.2. Image — Xử Lý Ảnh Đúng Cách

```dart
// ✅ CHUẨN — Image với loading state và error fallback
class AppNetworkImage extends StatelessWidget {
  const AppNetworkImage({
    super.key,
    required this.url,
    this.width,
    this.height,
    this.fit = BoxFit.cover,
    this.borderRadius,
  });

  final String url;
  final double? width;
  final double? height;
  final BoxFit fit;
  final BorderRadius? borderRadius;

  @override
  Widget build(BuildContext context) {
    Widget image = Image.network(
      url,
      width: width,
      height: height,
      fit: fit,
      // loadingBuilder: hiển thị trong khi ảnh đang tải
      loadingBuilder: (context, child, loadingProgress) {
        if (loadingProgress == null) return child;
        return Container(
          width: width,
          height: height,
          color: Theme.of(context).colorScheme.surfaceContainerHighest,
          child: Center(
            child: CircularProgressIndicator(
              value: loadingProgress.expectedTotalBytes != null
                  ? loadingProgress.cumulativeBytesLoaded /
                      loadingProgress.expectedTotalBytes!
                  : null,
            ),
          ),
        );
      },
      // errorBuilder: hiển thị khi ảnh lỗi
      errorBuilder: (context, error, stackTrace) {
        return Container(
          width: width,
          height: height,
          color: Theme.of(context).colorScheme.surfaceContainerHighest,
          child: Icon(
            Icons.broken_image_outlined,
            color: Theme.of(context).colorScheme.outline,
          ),
        );
      },
    );

    if (borderRadius != null) {
      image = ClipRRect(
        borderRadius: borderRadius!,
        child: image,
      );
    }

    return image;
  }
}
```

---

## 2.5. Widget Scroll: ListView và GridView

### 2.5.1. ListView — Danh Sách Hiệu Năng Cao

```dart
// ✅ CHUẨN — ListView.builder cho danh sách dài
// ListView.builder chỉ tạo widget khi item xuất hiện trên màn hình
// Không bao giờ dùng ListView(children: list.map(...).toList())
// cho danh sách có nhiều item

class ProductListScreen extends StatelessWidget {
  const ProductListScreen({super.key, required this.products});
  final List<Product> products;

  @override
  Widget build(BuildContext context) {
    if (products.isEmpty) {
      return const Center(child: Text('Không có sản phẩm'));
    }

    return ListView.separated(
      // padding cho toàn bộ list
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      itemCount: products.length,
      // separatorBuilder: widget ngăn cách giữa các item
      separatorBuilder: (context, index) => const SizedBox(height: 12),
      itemBuilder: (context, index) {
        final product = products[index];
        return ProductCard(product: product);
      },
    );
  }
}
```

**Phân biệt các dạng ListView:**

| Dạng | Khi nào dùng |
|---|---|
| `ListView()` | Ít item, đã biết trước, không bao giờ >20 item |
| `ListView.builder()` | Danh sách dài, lazy loading, dữ liệu từ API |
| `ListView.separated()` | Như builder nhưng cần separator giữa items |
| `ListView.custom()` | Khi cần kiểm soát hoàn toàn việc tạo item |

### 2.5.2. GridView — Lưới Sản Phẩm

```dart
// ✅ CHUẨN — GridView responsive với SliverGridDelegate
class ProductGrid extends StatelessWidget {
  const ProductGrid({super.key, required this.products});
  final List<Product> products;

  @override
  Widget build(BuildContext context) {
    return GridView.builder(
      padding: const EdgeInsets.all(16),
      gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: 2,        // Số cột
        crossAxisSpacing: 12,     // Khoảng cách ngang
        mainAxisSpacing: 12,      // Khoảng cách dọc
        childAspectRatio: 0.75,   // Tỷ lệ width:height của mỗi cell
      ),
      itemCount: products.length,
      itemBuilder: (context, index) {
        return ProductCard(product: products[index]);
      },
    );
  }
}

// ✅ NÂNG CAO — Responsive grid: 2 cột trên mobile, 3 cột trên tablet
class ResponsiveProductGrid extends StatelessWidget {
  const ResponsiveProductGrid({super.key, required this.products});
  final List<Product> products;

  @override
  Widget build(BuildContext context) {
    // SliverGridDelegateWithMaxCrossAxisExtent tự tính số cột
    // dựa trên maxCrossAxisExtent
    return GridView.builder(
      padding: const EdgeInsets.all(16),
      gridDelegate: const SliverGridDelegateWithMaxCrossAxisExtent(
        maxCrossAxisExtent: 220, // Mỗi item tối đa 220px rộng
        crossAxisSpacing: 12,
        mainAxisSpacing: 12,
        childAspectRatio: 0.75,
      ),
      itemCount: products.length,
      itemBuilder: (context, index) {
        return ProductCard(product: products[index]);
      },
    );
  }
}
```

---

## 2.6. Widget Input và Gesture

### 2.6.1. TextField — Nhập Liệu

```dart
// ✅ CHUẨN — TextField với controller và validation
class SearchBar extends StatefulWidget {
  const SearchBar({
    super.key,
    required this.onSearch,
  });

  final ValueChanged<String> onSearch;

  @override
  State<SearchBar> createState() => _SearchBarState();
}

class _SearchBarState extends State<SearchBar> {
  // TextEditingController PHẢI được dispose trong dispose()
  final _controller = TextEditingController();
  final _focusNode = FocusNode();

  @override
  void dispose() {
    _controller.dispose();
    _focusNode.dispose(); // Cũng phải dispose FocusNode
    super.dispose();
  }

  void _handleSubmit() {
    final query = _controller.text.trim();
    if (query.isEmpty) return;
    widget.onSearch(query);
    _focusNode.unfocus(); // Đóng bàn phím
  }

  @override
  Widget build(BuildContext context) {
    return TextField(
      controller: _controller,
      focusNode: _focusNode,
      textInputAction: TextInputAction.search, // Nút "Search" trên bàn phím
      onSubmitted: (_) => _handleSubmit(),
      decoration: InputDecoration(
        hintText: 'Tìm kiếm sản phẩm...',
        prefixIcon: const Icon(Icons.search),
        suffixIcon: ListenableBuilder(
          listenable: _controller,
          builder: (context, _) {
            // Hiện nút xóa chỉ khi có text
            return _controller.text.isNotEmpty
                ? IconButton(
                    icon: const Icon(Icons.clear),
                    onPressed: () {
                      _controller.clear();
                      _focusNode.requestFocus();
                    },
                  )
                : const SizedBox.shrink();
          },
        ),
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(12),
        ),
      ),
    );
  }
}
```

### 2.6.2. Gesture Detection

```dart
// ✅ CHUẨN — GestureDetector cho long press, double tap
class SwipeableCard extends StatelessWidget {
  const SwipeableCard({
    super.key,
    required this.child,
    this.onTap,
    this.onLongPress,
  });

  final Widget child;
  final VoidCallback? onTap;
  final VoidCallback? onLongPress;

  @override
  Widget build(BuildContext context) {
    // InkWell: như GestureDetector nhưng có ripple effect Material
    // Dùng InkWell thay GestureDetector khi trong Material context
    return InkWell(
      onTap: onTap,
      onLongPress: onLongPress,
      borderRadius: BorderRadius.circular(12),
      child: child,
    );
  }
}
```

---

## 2.7. Bài Tập Thực Hành: Màn Hình Profile

Sau khi nắm lý thuyết, bài tập sau yêu cầu kết hợp các widget đã học để xây dựng một màn hình Profile hoàn chỉnh không dùng thư viện ngoài.

**Yêu cầu:**
- Header với avatar (Stack + CircleAvatar), tên và bio
- Stats row (Followers, Following, Posts) bằng Row + Column
- Tab bar đơn giản với IndexedStack để switch content
- Grid ảnh 3 cột khi tab "Posts" được chọn

```dart
// Gợi ý cấu trúc
class ProfileScreen extends StatefulWidget {
  const ProfileScreen({super.key});

  @override
  State<ProfileScreen> createState() => _ProfileScreenState();
}

class _ProfileScreenState extends State<ProfileScreen> {
  int _selectedTab = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: CustomScrollView(
        slivers: [
          // SliverAppBar: AppBar thu lại khi scroll
          const SliverAppBar(
            expandedHeight: 200,
            pinned: true,
            flexibleSpace: FlexibleSpaceBar(
              background: _ProfileHeader(),
            ),
          ),
          // SliverPersistentHeader: Tab bar cố định khi scroll
          SliverPersistentHeader(
            pinned: true,
            delegate: _TabBarDelegate(
              selectedIndex: _selectedTab,
              onTabChanged: (i) => setState(() => _selectedTab = i),
            ),
          ),
          // Content theo tab
          if (_selectedTab == 0)
            SliverGrid(
              delegate: SliverChildBuilderDelegate(
                (context, index) => const _PostThumbnail(),
                childCount: 30,
              ),
              gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                crossAxisCount: 3,
                crossAxisSpacing: 2,
                mainAxisSpacing: 2,
              ),
            )
          else
            const SliverFillRemaining(
              child: Center(child: Text('Reels / Tagged')),
            ),
        ],
      ),
    );
  }
}
```

---

## Tóm Tắt Chương 2

| Khái niệm | Điểm Cốt Lõi |
|---|---|
| Kiến trúc Flutter | Tự render pixel, không dùng native UI component |
| Widget tree | Immutable, tái tạo mỗi lần build |
| Element tree | Mutable, tái sử dụng để tối ưu |
| StatelessWidget | Cho component thuần hiển thị, không có local state |
| StatefulWidget | Khi cần local state; nhớ dispose controller |
| Column/Row | Flex layout; dùng Expanded/Flexible cho dynamic sizing |
| ListView.builder | Luôn dùng cho danh sách dài; lazy rendering |
| TextField | Luôn dispose TextEditingController và FocusNode |

> **Nguyên tắc thiết kế:** Bắt đầu nhỏ — mỗi widget chỉ làm một việc. Một `ProductCard` không nên đồng thời fetch data, xử lý logic và render UI. Tách thành presentation widget (nhận data qua props) và container widget (xử lý logic). Đây là nền tảng cho Clean Architecture ở các chương sau.
