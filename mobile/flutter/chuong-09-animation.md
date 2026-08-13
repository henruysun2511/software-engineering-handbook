# Chương 8: UI Nâng Cao & Material 3

---

## 8.1. Tổng Quan Material 3 Trong Flutter

### 8.1.1. Material 3 Là Gì?

Material Design 3 (còn gọi là Material You) là phiên bản mới nhất của design system từ Google, ra mắt năm 2021 và được tích hợp đầy đủ vào Flutter 3.x. Khác với Material 2, Material 3 tập trung vào **dynamic color** — khả năng tự sinh palette màu từ một seed color duy nhất, và **expressive components** — các component có form dạng mềm mại, rounded hơn.

Để bật Material 3, chỉ cần một dòng trong `ThemeData`:

```dart
ThemeData(useMaterial3: true)
```

Từ Flutter 3.16 trở đi, `useMaterial3: true` là mặc định.

---

## 8.2. Navigation Components

### 8.2.1. NavigationBar — Bottom Navigation

`NavigationBar` là component bottom navigation chuẩn Material 3, thay thế `BottomNavigationBar` của Material 2.

```dart
// ✅ CHUẨN — NavigationBar với indicator animation
class AppShell extends StatefulWidget {
  const AppShell({super.key, required this.child});
  final Widget child;

  @override
  State<AppShell> createState() => _AppShellState();
}

class _AppShellState extends State<AppShell> {
  static const _tabs = [
    (path: '/', icon: Icons.home_outlined, activeIcon: Icons.home, label: 'Trang chủ'),
    (path: '/products', icon: Icons.grid_view_outlined, activeIcon: Icons.grid_view, label: 'Sản phẩm'),
    (path: '/cart', icon: Icons.shopping_cart_outlined, activeIcon: Icons.shopping_cart, label: 'Giỏ hàng'),
    (path: '/profile', icon: Icons.person_outlined, activeIcon: Icons.person, label: 'Tôi'),
  ];

  int _selectedIndex(BuildContext context) {
    final location = GoRouterState.of(context).uri.path;
    final index = _tabs.indexWhere((t) => location.startsWith(t.path));
    return index < 0 ? 0 : index;
  }

  @override
  Widget build(BuildContext context) {
    final selectedIndex = _selectedIndex(context);

    return Scaffold(
      body: widget.child,
      bottomNavigationBar: NavigationBar(
        selectedIndex: selectedIndex,
        onDestinationSelected: (index) => context.go(_tabs[index].path),
        // Kiểu hiển thị label
        labelBehavior: NavigationDestinationLabelBehavior.alwaysShow,
        // Elevation — 0 = flat style
        elevation: 0,
        // Surface tint từ colorScheme
        indicatorColor: context.colorScheme.secondaryContainer,
        destinations: _tabs.map((tab) {
          return NavigationDestination(
            icon: Icon(tab.icon),
            selectedIcon: Icon(tab.activeIcon),
            label: tab.label,
            // Badge cho notification / cart count
          );
        }).toList(),
      ),
    );
  }
}

// NavigationBar với Badge trên icon giỏ hàng
class CartNavigationDestination extends ConsumerWidget {
  const CartNavigationDestination({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final cartCount = ref.watch(cartItemCountProvider);

    return NavigationDestination(
      icon: Badge(
        isLabelVisible: cartCount > 0,
        label: Text('$cartCount'),
        child: const Icon(Icons.shopping_cart_outlined),
      ),
      selectedIcon: Badge(
        isLabelVisible: cartCount > 0,
        label: Text('$cartCount'),
        child: const Icon(Icons.shopping_cart),
      ),
      label: 'Giỏ hàng',
    );
  }
}
```

### 8.2.2. NavigationDrawer — Sidebar Navigation

```dart
// ✅ CHUẨN — NavigationDrawer cho tablet / desktop
class AppDrawer extends ConsumerWidget {
  const AppDrawer({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final user = ref.watch(currentUserProvider);
    final currentPath = GoRouterState.of(context).uri.path;

    return NavigationDrawer(
      selectedIndex: _getSelectedIndex(currentPath),
      onDestinationSelected: (index) {
        context.pop(); // Đóng drawer
        _navigate(context, index);
      },
      children: [
        // Header với thông tin user
        Padding(
          padding: const EdgeInsets.fromLTRB(16, 28, 16, 16),
          child: user != null
              ? _UserHeader(user: user)
              : const _GuestHeader(),
        ),

        const Divider(height: 1),
        const SizedBox(height: 8),

        // Navigation items
        const NavigationDrawerDestination(
          icon: Icon(Icons.home_outlined),
          selectedIcon: Icon(Icons.home),
          label: Text('Trang chủ'),
        ),
        const NavigationDrawerDestination(
          icon: Icon(Icons.category_outlined),
          selectedIcon: Icon(Icons.category),
          label: Text('Danh mục'),
        ),
        const NavigationDrawerDestination(
          icon: Icon(Icons.favorite_outlined),
          selectedIcon: Icon(Icons.favorite),
          label: Text('Yêu thích'),
        ),
        const NavigationDrawerDestination(
          icon: Icon(Icons.history_outlined),
          selectedIcon: Icon(Icons.history),
          label: Text('Lịch sử đơn hàng'),
        ),

        const Padding(
          padding: EdgeInsets.symmetric(horizontal: 16, vertical: 8),
          child: Divider(),
        ),

        // Section label — không phải destination
        const Padding(
          padding: EdgeInsets.symmetric(horizontal: 16),
          child: Text('Tài khoản', style: TextStyle(fontSize: 12)),
        ),

        const NavigationDrawerDestination(
          icon: Icon(Icons.settings_outlined),
          selectedIcon: Icon(Icons.settings),
          label: Text('Cài đặt'),
        ),

        // Logout ở cuối — dùng ListTile thay vì destination
        const SizedBox(height: 8),
        ListTile(
          leading: const Icon(Icons.logout),
          title: const Text('Đăng xuất'),
          onTap: () => ref.read(authProvider.notifier).logout(),
        ),
      ],
    );
  }

  int _getSelectedIndex(String path) {
    if (path.startsWith('/products')) return 1;
    if (path.startsWith('/favorites')) return 2;
    if (path.startsWith('/orders')) return 3;
    if (path.startsWith('/settings')) return 4;
    return 0;
  }

  void _navigate(BuildContext context, int index) {
    final paths = ['/', '/products', '/favorites', '/orders', '/settings'];
    if (index < paths.length) context.go(paths[index]);
  }
}
```

---

## 8.3. AppBar và SearchBar

### 8.3.1. AppBar Nâng Cao

```dart
// ✅ CHUẨN — SliverAppBar với scroll behavior
class ProductListScreen extends StatelessWidget {
  const ProductListScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: CustomScrollView(
        slivers: [
          // SliverAppBar: thu lại khi scroll xuống, hiện lại khi scroll lên
          SliverAppBar.medium(
            title: const Text('Sản phẩm'),
            // floating: AppBar hiện lại ngay khi scroll lên một chút
            floating: true,
            // snap: AppBar phải hiện/ẩn hoàn toàn (không dừng giữa chừng)
            snap: true,
            // pinned: AppBar luôn cố định — chỉ thu nhỏ, không ẩn
            pinned: false,
            actions: [
              IconButton(
                icon: const Icon(Icons.search),
                onPressed: () => context.push('/search'),
              ),
              IconButton(
                icon: const Icon(Icons.filter_list),
                onPressed: () => _showFilterSheet(context),
              ),
            ],
          ),

          // Content
          SliverPadding(
            padding: const EdgeInsets.all(16),
            sliver: SliverGrid(
              delegate: SliverChildBuilderDelegate(
                (context, index) => const ProductCard(),
                childCount: 20,
              ),
              gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                crossAxisCount: 2,
                crossAxisSpacing: 12,
                mainAxisSpacing: 12,
                childAspectRatio: 0.75,
              ),
            ),
          ),
        ],
      ),
    );
  }

  void _showFilterSheet(BuildContext context) {
    showModalBottomSheet(
      context: context,
      isScrollControlled: true, // Cho phép sheet cao hơn 50% màn hình
      useSafeArea: true,
      builder: (_) => const FilterBottomSheet(),
    );
  }
}
```

### 8.3.2. SearchBar Material 3

```dart
// ✅ CHUẨN — SearchBar theo spec Material 3
class AppSearchBar extends ConsumerStatefulWidget {
  const AppSearchBar({super.key});

  @override
  ConsumerState<AppSearchBar> createState() => _AppSearchBarState();
}

class _AppSearchBarState extends ConsumerState<AppSearchBar> {
  final _searchController = SearchController();

  @override
  void dispose() {
    _searchController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return SearchAnchor(
      searchController: _searchController,
      // Builder: giao diện thanh search khi chưa mở
      builder: (context, controller) {
        return SearchBar(
          controller: controller,
          hintText: 'Tìm kiếm sản phẩm...',
          leading: const Icon(Icons.search),
          trailing: [
            if (controller.text.isNotEmpty)
              IconButton(
                icon: const Icon(Icons.close),
                onPressed: () {
                  controller.clear();
                  ref.read(searchControllerProvider.notifier).updateQuery('');
                },
              ),
          ],
          onTap: controller.openView,
          onChanged: (query) {
            ref.read(searchControllerProvider.notifier).updateQuery(query);
          },
          padding: const WidgetStatePropertyAll(
            EdgeInsets.symmetric(horizontal: 16),
          ),
        );
      },
      // suggestionsBuilder: gợi ý khi đang gõ
      suggestionsBuilder: (context, controller) async {
        final query = controller.text;
        if (query.isEmpty) return _buildRecentSearches(context);
        return _buildSearchSuggestions(context, query);
      },
    );
  }

  List<Widget> _buildRecentSearches(BuildContext context) {
    final recent = ['iPhone 15', 'Giày Nike', 'Áo khoác'];
    return [
      Padding(
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
        child: Text('Tìm kiếm gần đây',
            style: context.textTheme.labelMedium),
      ),
      ...recent.map((term) => ListTile(
            leading: const Icon(Icons.history),
            title: Text(term),
            trailing: const Icon(Icons.north_west, size: 16),
            onTap: () {
              _searchController.text = term;
              context.push('/search?q=${Uri.encodeComponent(term)}');
            },
          )),
    ];
  }

  Future<List<Widget>> _buildSearchSuggestions(
    BuildContext context,
    String query,
  ) async {
    // Gọi API tìm gợi ý (debounced)
    final suggestions = await ref
        .read(productRepositoryProvider)
        .fetchSuggestions(query);

    return suggestions
        .map((s) => ListTile(
              leading: const Icon(Icons.search),
              title: Text(s),
              onTap: () {
                _searchController.closeView(s);
                context.push('/search?q=${Uri.encodeComponent(s)}');
              },
            ))
        .toList();
  }
}
```

---

## 8.4. Bottom Sheet, Dialog và Snackbar

### 8.4.1. Modal Bottom Sheet

```dart
// ✅ CHUẨN — Bottom sheet với DraggableScrollableSheet
void showProductFilterSheet(BuildContext context) {
  showModalBottomSheet(
    context: context,
    // Cho phép kéo để đóng
    enableDrag: true,
    // Không đóng khi tap ngoài (nếu có state phức tạp)
    isDismissible: true,
    // isScrollControlled: true để sheet có thể cao hơn 60%
    isScrollControlled: true,
    // useSafeArea: tránh home indicator
    useSafeArea: true,
    // Builder với DraggableScrollableSheet
    builder: (context) => DraggableScrollableSheet(
      // Chiều cao ban đầu: 50% màn hình
      initialChildSize: 0.5,
      // Chiều cao tối thiểu: 30%
      minChildSize: 0.3,
      // Chiều cao tối đa: 90%
      maxChildSize: 0.9,
      // snap: nhảy về các anchor point thay vì dừng tự do
      snap: true,
      snapSizes: const [0.5, 0.9],
      builder: (context, scrollController) {
        return Container(
          decoration: BoxDecoration(
            color: context.colorScheme.surface,
            borderRadius: const BorderRadius.vertical(
              top: Radius.circular(28),
            ),
          ),
          child: Column(
            children: [
              // Drag handle
              Center(
                child: Container(
                  margin: const EdgeInsets.symmetric(vertical: 12),
                  width: 32,
                  height: 4,
                  decoration: BoxDecoration(
                    color: context.colorScheme.outlineVariant,
                    borderRadius: BorderRadius.circular(2),
                  ),
                ),
              ),
              // Title
              Padding(
                padding: const EdgeInsets.symmetric(horizontal: 24),
                child: Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    Text('Bộ lọc', style: context.textTheme.titleLarge),
                    TextButton(
                      onPressed: () {/* Reset filters */},
                      child: const Text('Xóa tất cả'),
                    ),
                  ],
                ),
              ),
              const Divider(),
              // Scrollable content
              Expanded(
                child: ListView(
                  controller: scrollController,
                  padding: const EdgeInsets.symmetric(horizontal: 24),
                  children: const [
                    FilterSection(title: 'Danh mục', child: CategoryFilter()),
                    FilterSection(title: 'Khoảng giá', child: PriceRangeFilter()),
                    FilterSection(title: 'Đánh giá', child: RatingFilter()),
                    SizedBox(height: 16),
                  ],
                ),
              ),
              // Action buttons
              Padding(
                padding: const EdgeInsets.fromLTRB(24, 8, 24, 24),
                child: FilledButton(
                  onPressed: () {
                    Navigator.pop(context);
                    // Apply filters
                  },
                  style: FilledButton.styleFrom(
                    minimumSize: const Size.fromHeight(52),
                  ),
                  child: const Text('Áp dụng bộ lọc'),
                ),
              ),
            ],
          ),
        );
      },
    ),
  );
}
```

### 8.4.2. Dialog Material 3

```dart
// ✅ CHUẨN — AlertDialog và custom Dialog
Future<bool?> showDeleteConfirmDialog(
  BuildContext context, {
  required String itemName,
}) {
  return showDialog<bool>(
    context: context,
    // barrierDismissible: false nếu action quan trọng
    barrierDismissible: true,
    builder: (context) => AlertDialog(
      icon: Icon(
        Icons.delete_outline,
        color: context.colorScheme.error,
        size: 32,
      ),
      title: const Text('Xác nhận xóa'),
      content: Text(
        'Bạn có chắc chắn muốn xóa "$itemName"?\n'
        'Hành động này không thể hoàn tác.',
      ),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context, false),
          child: const Text('Hủy'),
        ),
        FilledButton(
          style: FilledButton.styleFrom(
            backgroundColor: context.colorScheme.error,
            foregroundColor: context.colorScheme.onError,
          ),
          onPressed: () => Navigator.pop(context, true),
          child: const Text('Xóa'),
        ),
      ],
    ),
  );
}

// Sử dụng
Future<void> handleDeleteProduct(BuildContext context, Product product) async {
  final confirmed = await showDeleteConfirmDialog(
    context,
    itemName: product.name,
  );

  if (confirmed == true && context.mounted) {
    // context.mounted: kiểm tra widget còn trong tree sau await
    await ref.read(productActionsProvider).deleteProduct(product.id);
  }
}

// Custom full-screen dialog
void showImagePicker(BuildContext context) {
  showDialog(
    context: context,
    useSafeArea: false,
    builder: (context) => Dialog.fullscreen(
      child: Scaffold(
        appBar: AppBar(
          title: const Text('Chọn ảnh'),
          leading: IconButton(
            icon: const Icon(Icons.close),
            onPressed: () => Navigator.pop(context),
          ),
        ),
        body: const ImagePickerGrid(),
      ),
    ),
  );
}
```

### 8.4.3. SnackBar — Notification Ngắn

```dart
// ✅ CHUẨN — SnackBar đúng cách trong Material 3
// Dùng ScaffoldMessenger thay vì Scaffold.of(context)

// Helper để show snackbar từ bất cứ đâu
class AppSnackBar {
  static void showSuccess(BuildContext context, String message) {
    ScaffoldMessenger.of(context)
      ..hideCurrentSnackBar()
      ..showSnackBar(
        SnackBar(
          content: Row(
            children: [
              const Icon(Icons.check_circle_outline, color: Colors.white, size: 20),
              const SizedBox(width: 8),
              Expanded(child: Text(message)),
            ],
          ),
          backgroundColor: Colors.green.shade700,
          behavior: SnackBarBehavior.floating, // Nổi lên trên bottom nav
          shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(8)),
          margin: const EdgeInsets.all(16),
          duration: const Duration(seconds: 3),
        ),
      );
  }

  static void showError(BuildContext context, String message) {
    ScaffoldMessenger.of(context)
      ..hideCurrentSnackBar()
      ..showSnackBar(
        SnackBar(
          content: Text(message),
          backgroundColor: Theme.of(context).colorScheme.error,
          behavior: SnackBarBehavior.floating,
          shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(8)),
          margin: const EdgeInsets.all(16),
          action: SnackBarAction(
            label: 'Thử lại',
            textColor: Colors.white,
            onPressed: () { /* retry */ },
          ),
        ),
      );
  }

  static void showCartAdded(BuildContext context, String productName) {
    ScaffoldMessenger.of(context)
      ..hideCurrentSnackBar()
      ..showSnackBar(
        SnackBar(
          content: Text('Đã thêm "$productName" vào giỏ hàng'),
          behavior: SnackBarBehavior.floating,
          shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(8)),
          margin: const EdgeInsets.all(16),
          action: SnackBarAction(
            label: 'Xem giỏ hàng',
            onPressed: () => context.go('/cart'),
          ),
        ),
      );
  }
}
```

---

## 8.5. Chip và Badge

```dart
// ✅ CHUẨN — Các loại Chip trong Material 3

// FilterChip — bộ lọc có thể chọn/bỏ chọn
class CategoryFilter extends ConsumerWidget {
  const CategoryFilter({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final categories = ref.watch(categoriesProvider).valueOrNull ?? [];
    final selectedIds = ref.watch(filterProvider.select((s) => s.categoryIds));

    return Wrap(
      spacing: 8,
      runSpacing: 8,
      children: categories.map((category) {
        final isSelected = selectedIds.contains(category.id);
        return FilterChip(
          label: Text(category.name),
          selected: isSelected,
          onSelected: (_) =>
              ref.read(filterProvider.notifier).toggleCategory(category.id),
          avatar: isSelected ? null : Text(category.emoji),
          showCheckmark: true,
        );
      }).toList(),
    );
  }
}

// InputChip — chip có thể xóa
class SelectedTagList extends StatelessWidget {
  const SelectedTagList({
    super.key,
    required this.tags,
    required this.onRemove,
  });

  final List<String> tags;
  final ValueChanged<String> onRemove;

  @override
  Widget build(BuildContext context) {
    return Wrap(
      spacing: 8,
      runSpacing: 4,
      children: tags.map((tag) {
        return InputChip(
          label: Text(tag),
          onDeleted: () => onRemove(tag),
          deleteIcon: const Icon(Icons.close, size: 16),
          materialTapTargetSize: MaterialTapTargetSize.shrinkWrap,
        );
      }).toList(),
    );
  }
}

// AssistChip — gợi ý hành động
class QuickActionChips extends StatelessWidget {
  const QuickActionChips({super.key});

  @override
  Widget build(BuildContext context) {
    return SingleChildScrollView(
      scrollDirection: Axis.horizontal,
      padding: const EdgeInsets.symmetric(horizontal: 16),
      child: Row(
        children: [
          AssistChip(
            label: const Text('Chia sẻ'),
            avatar: const Icon(Icons.share_outlined),
            onPressed: () {},
          ),
          const SizedBox(width: 8),
          AssistChip(
            label: const Text('Thêm vào yêu thích'),
            avatar: const Icon(Icons.favorite_outline),
            onPressed: () {},
          ),
          const SizedBox(width: 8),
          AssistChip(
            label: const Text('So sánh'),
            avatar: const Icon(Icons.compare_arrows),
            onPressed: () {},
          ),
        ],
      ),
    );
  }
}
```

---

## 8.6. Tạo Custom Widget Tái Sử Dụng

### 8.6.1. Skeleton Loading — Shimmer Effect

```dart
// ✅ CHUẨN — Skeleton widget cho loading state
// Dùng thay vì CircularProgressIndicator để UX tốt hơn
class SkeletonBox extends StatefulWidget {
  const SkeletonBox({
    super.key,
    required this.width,
    required this.height,
    this.borderRadius = 8.0,
  });

  final double? width;
  final double height;
  final double borderRadius;

  @override
  State<SkeletonBox> createState() => _SkeletonBoxState();
}

class _SkeletonBoxState extends State<SkeletonBox>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 1200),
    )..repeat(reverse: true);

    _animation = Tween<double>(begin: 0.4, end: 1.0).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeInOut),
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _animation,
      builder: (context, child) {
        return Opacity(
          opacity: _animation.value,
          child: Container(
            width: widget.width,
            height: widget.height,
            decoration: BoxDecoration(
              color: context.colorScheme.surfaceContainerHighest,
              borderRadius: BorderRadius.circular(widget.borderRadius),
            ),
          ),
        );
      },
    );
  }
}

// Skeleton cho Product Card
class ProductCardSkeleton extends StatelessWidget {
  const ProductCardSkeleton({super.key});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          // Image skeleton
          const SkeletonBox(width: double.infinity, height: 160, borderRadius: 0),
          Padding(
            padding: const EdgeInsets.all(12),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                SkeletonBox(width: double.infinity, height: 14),
                const SizedBox(height: 6),
                SkeletonBox(width: 120, height: 14),
                const SizedBox(height: 12),
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    SkeletonBox(width: 80, height: 18),
                    SkeletonBox(width: 64, height: 32, borderRadius: 16),
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

// Skeleton grid
class ProductGridSkeleton extends StatelessWidget {
  const ProductGridSkeleton({super.key, this.count = 6});
  final int count;

  @override
  Widget build(BuildContext context) {
    return GridView.builder(
      padding: const EdgeInsets.all(16),
      gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: 2,
        crossAxisSpacing: 12,
        mainAxisSpacing: 12,
        childAspectRatio: 0.75,
      ),
      itemCount: count,
      itemBuilder: (_, __) => const ProductCardSkeleton(),
    );
  }
}
```

### 8.6.2. Empty State và Error State

```dart
// ✅ CHUẨN — Empty state với illustration
class EmptyState extends StatelessWidget {
  const EmptyState({
    super.key,
    required this.icon,
    required this.title,
    this.subtitle,
    this.action,
  });

  final IconData icon;
  final String title;
  final String? subtitle;
  final Widget? action;

  // Factory constructors cho common empty states
  factory EmptyState.cart({required VoidCallback onShop}) => EmptyState(
        icon: Icons.shopping_cart_outlined,
        title: 'Giỏ hàng trống',
        subtitle: 'Hãy thêm sản phẩm vào giỏ hàng',
        action: FilledButton.icon(
          onPressed: onShop,
          icon: const Icon(Icons.store),
          label: const Text('Mua sắm ngay'),
        ),
      );

  factory EmptyState.search({required String query}) => EmptyState(
        icon: Icons.search_off,
        title: 'Không tìm thấy kết quả',
        subtitle: 'Không có sản phẩm nào phù hợp với "$query"',
      );

  factory EmptyState.favorites() => const EmptyState(
        icon: Icons.favorite_border,
        title: 'Chưa có sản phẩm yêu thích',
        subtitle: 'Nhấn ♡ để lưu sản phẩm yêu thích',
      );

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(32),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(icon, size: 80, color: context.colorScheme.outline),
            const SizedBox(height: 16),
            Text(
              title,
              style: context.textTheme.titleMedium,
              textAlign: TextAlign.center,
            ),
            if (subtitle != null) ...[
              const SizedBox(height: 8),
              Text(
                subtitle!,
                style: context.textTheme.bodyMedium?.copyWith(
                  color: context.colorScheme.onSurfaceVariant,
                ),
                textAlign: TextAlign.center,
              ),
            ],
            if (action != null) ...[
              const SizedBox(height: 24),
              action!,
            ],
          ],
        ),
      ),
    );
  }
}

// ✅ CHUẨN — Error state với retry
class ErrorView extends StatelessWidget {
  const ErrorView({
    super.key,
    required this.message,
    this.onRetry,
  });

  final String message;
  final VoidCallback? onRetry;

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(32),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(
              Icons.cloud_off_outlined,
              size: 72,
              color: context.colorScheme.outline,
            ),
            const SizedBox(height: 16),
            Text('Đã xảy ra lỗi', style: context.textTheme.titleMedium),
            const SizedBox(height: 8),
            Text(
              message,
              style: context.textTheme.bodySmall?.copyWith(
                color: context.colorScheme.onSurfaceVariant,
              ),
              textAlign: TextAlign.center,
            ),
            if (onRetry != null) ...[
              const SizedBox(height: 24),
              OutlinedButton.icon(
                onPressed: onRetry,
                icon: const Icon(Icons.refresh),
                label: const Text('Thử lại'),
              ),
            ],
          ],
        ),
      ),
    );
  }
}
```

### 8.6.3. Pull-to-Refresh

```dart
// ✅ CHUẨN — RefreshIndicator với Riverpod
class RefreshableProductList extends ConsumerWidget {
  const RefreshableProductList({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final productsAsync = ref.watch(productListProvider);

    return RefreshIndicator(
      onRefresh: () async {
        // invalidate: buộc provider fetch lại từ đầu
        ref.invalidate(productListProvider);
        // Chờ đến khi data mới về
        await ref.read(productListProvider.future);
      },
      child: productsAsync.when(
        loading: () => const ProductGridSkeleton(),
        error: (e, _) => ErrorView(
          message: e.toString(),
          onRetry: () => ref.invalidate(productListProvider),
        ),
        data: (state) => state.products.isEmpty
            ? EmptyState.search(query: '')
            : ProductGrid(products: state.products),
      ),
    );
  }
}
```

---

## 8.7. Bài Tập: Trang Chủ E-Commerce

Xây dựng màn hình Home hoàn chỉnh bao gồm:

```dart
class HomeScreen extends ConsumerWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Scaffold(
      body: CustomScrollView(
        slivers: [
          // 1. SliverAppBar với SearchBar
          SliverAppBar(
            floating: true,
            title: const AppSearchBar(),
            actions: [
              const CartIconButton(), // Badge với số lượng
              const NotificationButton(),
            ],
          ),

          // 2. Banner carousel
          const SliverToBoxAdapter(
            child: BannerCarousel(),
          ),

          // 3. Category horizontal scroll
          SliverToBoxAdapter(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                const SectionHeader(title: 'Danh mục'),
                CategoryHorizontalList(),
              ],
            ),
          ),

          // 4. Featured products header
          const SliverToBoxAdapter(
            child: SectionHeader(
              title: 'Sản phẩm nổi bật',
              action: Text('Xem tất cả'),
            ),
          ),

          // 5. Featured products grid
          SliverPadding(
            padding: const EdgeInsets.symmetric(horizontal: 16),
            sliver: Consumer(
              builder: (context, ref, _) {
                final featuredAsync = ref.watch(featuredProductsProvider);
                return featuredAsync.when(
                  loading: () => SliverGrid(
                    delegate: SliverChildBuilderDelegate(
                      (_, __) => const ProductCardSkeleton(),
                      childCount: 4,
                    ),
                    gridDelegate: _gridDelegate,
                  ),
                  error: (e, _) => SliverToBoxAdapter(
                    child: ErrorView(message: e.toString()),
                  ),
                  data: (products) => SliverGrid(
                    delegate: SliverChildBuilderDelegate(
                      (_, i) => ProductCard(product: products[i]),
                      childCount: products.length,
                    ),
                    gridDelegate: _gridDelegate,
                  ),
                );
              },
            ),
          ),
        ],
      ),
    );
  }

  static const _gridDelegate = SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 2,
    crossAxisSpacing: 12,
    mainAxisSpacing: 12,
    childAspectRatio: 0.75,
  );
}
```

---

## Tóm Tắt Chương 8

| Khái niệm | Điểm Cốt Lõi |
|---|---|
| NavigationBar | Thay thế BottomNavigationBar trong M3; tích hợp Badge tốt |
| NavigationDrawer | Sidebar cho tablet/desktop; section header với label |
| SliverAppBar | Thu nhỏ khi scroll; floating/pinned/snap behavior |
| SearchAnchor | SearchBar + dropdown gợi ý theo spec M3 |
| ModalBottomSheet | `isScrollControlled: true` + `DraggableScrollableSheet` cho UX tốt |
| AlertDialog | `icon` field mới trong M3; action buttons rõ ràng |
| SnackBar | `behavior: floating`, `hideCurrentSnackBar()` trước khi show mới |
| FilterChip | Trạng thái selected/unselected — lý tưởng cho bộ lọc |
| Skeleton Loading | Tốt hơn spinner — người dùng thấy layout trước khi data về |
| EmptyState | Factory constructor cho các loại empty state phổ biến |

> **UX tip quan trọng:** Đừng bao giờ để màn hình trắng trong khi load. Thứ tự ưu tiên: (1) Skeleton loading → user thấy layout, (2) Optimistic update → thay đổi UI trước khi API confirm, (3) Error state với retry button → không để user bị kẹt. Ba pattern này tạo ra sự khác biệt lớn giữa app "cảm giác chậm" và app "cảm giác nhanh" dù cùng tốc độ mạng.