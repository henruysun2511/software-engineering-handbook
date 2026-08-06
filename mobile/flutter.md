# Tổng Hợp Kiến Thức Flutter Quan Trọng

## 1. Tổng Quan Flutter

- **Flutter** là framework UI mã nguồn mở của Google, dùng ngôn ngữ **Dart**, cho phép build app đa nền tảng (iOS, Android, Web, Desktop, Embedded) từ **một codebase duy nhất**.
- Flutter không dùng widget native của hệ điều hành mà **tự vẽ (render)** toàn bộ UI thông qua engine đồ họa riêng (Skia / Impeller) → giao diện đồng nhất trên mọi nền tảng, hiệu năng cao.
- **Everything is a Widget**: từ layout, text, đến padding, đều là widget.

---

## 2. Kiến Trúc Cốt Lõi

### 2.1. Ba cây (Trees)
- **Widget Tree**: cấu hình khai báo (immutable), mô tả UI trông như thế nào.
- **Element Tree**: cầu nối giữa Widget và RenderObject, quản lý vòng đời, giữ trạng thái.
- **RenderObject Tree**: chịu trách nhiệm layout, paint thực tế lên màn hình.

### 2.2. StatelessWidget vs StatefulWidget
- **StatelessWidget**: không có state nội bộ, chỉ build lại khi widget cha thay đổi.
- **StatefulWidget**: có `State` object riêng, tồn tại xuyên suốt vòng đời, dùng `setState()` để rebuild UI khi dữ liệu thay đổi.

### 2.3. Widget Lifecycle (StatefulWidget)
```
createState() → initState() → didChangeDependencies() 
→ build() → didUpdateWidget() (khi cha rebuild) 
→ setState() → build() (lặp lại)
→ deactivate() → dispose()
```
- `initState()`: khởi tạo 1 lần, gọi `super.initState()` đầu tiên.
- `dispose()`: giải phóng tài nguyên (controller, stream, listener...) — **bắt buộc** để tránh memory leak.

---

## 3. Layout Cơ Bản

| Widget | Công dụng |
|---|---|
| `Container` | Box đa năng: padding, margin, decoration, size |
| `Row` / `Column` | Sắp xếp theo chiều ngang / dọc |
| `Stack` / `Positioned` | Xếp chồng widget, định vị tự do |
| `Expanded` / `Flexible` | Chia không gian còn lại trong Row/Column |
| `ListView` / `GridView` | Danh sách cuộn, hỗ trợ `.builder()` cho list dài (lazy load) |
| `SizedBox` | Khoảng cách cố định hoặc ép kích thước |
| `Wrap` | Tự xuống dòng khi hết không gian |
| `SingleChildScrollView` | Cuộn cho nội dung tràn màn hình |

**Nguyên tắc layout của Flutter**: *"Constraints go down. Sizes go up. Parent sets position."*
→ Widget cha truyền ràng buộc (constraints) xuống con, con tự quyết định size rồi trả lên, cha quyết định vị trí.

---

## 4. Quản Lý State (State Management)

### 4.1. Local state
- `setState()` — đơn giản, dùng cho state cục bộ trong 1 widget.

### 4.2. Các giải pháp phổ biến (từ đơn giản → phức tạp)
| Giải pháp | Đặc điểm |
|---|---|
| **Provider** | Đơn giản, dùng InheritedWidget bên dưới, phổ biến cho dự án vừa/nhỏ |
| **Riverpod** | Kế thừa Provider, an toàn hơn (compile-time), không phụ thuộc BuildContext, hỗ trợ code-gen |
| **Bloc / Cubit** | Theo mô hình Event → State, tách biệt rõ business logic, phù hợp dự án lớn, dễ test |
| **GetX** | All-in-one (state, route, DI), gọn nhẹ nhưng ít khuyến khích dùng ở dự án lớn vì "quá phép thuật" |
| **MobX** | Reactive, dùng observable/action, cần code-gen |
| **Redux** | Single source of truth, ít dùng hơn trong Flutter hiện đại |

### 4.3. Xu hướng hiện tại
- **Riverpod + code generation** và **Bloc** là hai lựa chọn phổ biến nhất cho ứng dụng production quy mô vừa/lớn.
- InheritedWidget / InheritedModel là nền tảng thấp mà nhiều thư viện state management dựa vào.

---

## 5. Điều Hướng (Navigation & Routing)

- **Navigator 1.0**: imperative, dùng `Navigator.push()` / `pop()` với `MaterialPageRoute`.
- **Navigator 2.0 / Router API**: declarative, phù hợp cho deep linking, web, quản lý stack phức tạp.
- **go_router**: package chính thức được khuyến nghị, đơn giản hóa Navigator 2.0, hỗ trợ deep link, nested navigation, redirect, tốt cho cả mobile & web.

```dart
Navigator.push(context, MaterialPageRoute(builder: (_) => DetailPage()));
Navigator.pop(context);
```

---

## 6. Async & Networking

- **Future**: giá trị bất đồng bộ trả về 1 lần. Dùng `async` / `await`.
- **Stream**: chuỗi giá trị bất đồng bộ theo thời gian (dùng cho `StreamBuilder`, real-time data).
- **FutureBuilder / StreamBuilder**: build UI dựa trên trạng thái của Future/Stream (loading, data, error).
- Networking phổ biến: package **http** (đơn giản) hoặc **dio** (interceptor, cancel token, upload/download mạnh hơn).
- Serialize JSON: `json_serializable` + `build_runner`, hoặc `freezed` cho immutable model + union type.

---

## 7. Quản Lý Bất Đồng Bộ Nâng Cao

- **Isolate**: Dart là single-threaded, dùng Isolate để chạy tác vụ nặng (CPU-bound) song song, tránh block UI thread. `compute()` là helper đơn giản để spawn isolate.
- **async/await** không tạo thread mới — nó chỉ nhường quyền điều khiển trong event loop (dùng cho I/O-bound).

---

## 8. Testing

| Loại test | Mục đích | Package |
|---|---|---|
| Unit test | Test logic thuần Dart | `test` |
| Widget test | Test UI 1 widget riêng lẻ | `flutter_test` |
| Integration test | Test toàn bộ luồng app trên thiết bị thật/emulator | `integration_test` |

---

## 9. Tối Ưu Hiệu Năng

- Dùng `const` constructor khi có thể → tránh rebuild không cần thiết.
- Tránh gọi hàm nặng trong `build()`.
- Dùng `ListView.builder` / `GridView.builder` thay vì tạo hết children cùng lúc.
- Tách widget nhỏ (`extract widget`) thay vì method trả về widget, giúp Flutter tối ưu rebuild qua Element tree.
- Dùng `RepaintBoundary` để cô lập vùng cần repaint riêng.
- Kiểm tra hiệu năng bằng **DevTools** (Performance tab, Widget rebuild stats, Memory).
- Tránh dùng `Opacity` cho animation liên tục → ưu tiên `AnimatedOpacity` hoặc `FadeTransition`.

---

## 10. Animation

- **Implicit animation**: `AnimatedContainer`, `AnimatedOpacity`, `AnimatedPositioned` — đơn giản, tự động animate khi giá trị thay đổi.
- **Explicit animation**: `AnimationController` + `Tween` + `AnimatedBuilder` — kiểm soát chi tiết (duration, curve, lặp lại...).
- `Hero` widget: animation chuyển cảnh mượt giữa 2 màn hình cho cùng 1 phần tử.

---

## 11. Dependency Injection & Kiến Trúc Dự Án

- Các pattern kiến trúc phổ biến: **MVVM**, **Clean Architecture** (data / domain / presentation layers).
- DI phổ biến: `get_it` (service locator), hoặc built-in của Riverpod.
- Tách biệt rõ: UI layer — Business logic (Bloc/Riverpod) — Data layer (Repository, API/DB).

---

## 12. Các Lệnh CLI Quan Trọng

```bash
flutter create my_app          # Tạo project mới
flutter run                    # Chạy app (hot reload sẵn)
flutter build apk / ios / web  # Build release
flutter pub get                # Cài dependencies
flutter pub upgrade            # Cập nhật package
flutter clean                  # Xóa build cache
flutter analyze                # Kiểm tra lỗi/lint
flutter test                   # Chạy test
```

- **Hot Reload**: cập nhật code ngay lập tức, giữ nguyên state.
- **Hot Restart**: build lại toàn bộ app, mất state.

---

## 13. Những Lỗi Thường Gặp Cần Lưu Ý

- Quên gọi `dispose()` cho `TextEditingController`, `AnimationController`, `StreamSubscription` → memory leak.
- Gọi `setState()` sau khi widget đã bị dispose → dùng `mounted` để kiểm tra trước.
- Dùng `BuildContext` sau `await` mà không kiểm tra `context.mounted`.
- Lạm dụng `Provider`/`InheritedWidget` ở gốc cây khiến toàn bộ app rebuild không cần thiết.
- Không dùng `key` khi làm việc với list widget động → sai trạng thái khi reorder/thêm/xóa.

---

## 14. Tài Nguyên Tham Khảo Thêm

- Trang chính thức: https://flutter.dev
- Package repository: https://pub.dev
- Dart docs: https://dart.dev
- Flutter DevTools: công cụ debug/profiling chính thức đi kèm SDK

---

*Tài liệu này tổng hợp các khái niệm cốt lõi và thực hành tốt nhất trong phát triển ứng dụng Flutter, phù hợp làm tài liệu ôn tập hoặc tra cứu nhanh.*
