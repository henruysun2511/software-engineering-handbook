# GIÁO TRÌNH REACT NATIVE
### Dành cho lập trình viên đã có nền tảng React / Next.js

---

# CHƯƠNG 8: PERFORMANCE

## 8.1. Vì sao Performance quan trọng hơn trên Mobile

### 8.1.1. Khác biệt bối cảnh

Trên Web, người dùng thường chạy trên máy tính có CPU/RAM mạnh, và trình duyệt có engine JS tối ưu cao (V8). Trên Mobile, ứng dụng chạy trên **hàng nghìn loại thiết bị khác nhau về cấu hình**, từ flagship mới đến máy phổ thông vài năm tuổi, với RAM hạn chế và pin có giới hạn — một đoạn code re-render lãng phí trên Web có thể "chấp nhận được", nhưng trên mobile tầm trung dễ gây giật lag rõ rệt, tốn pin, và tệ nhất là bị hệ điều hành **kill app** khi thiếu RAM.

---

## 8.2. Tối ưu Re-render — Nguyên tắc giống React, nhưng khắt khe hơn

### 8.2.1. `React.memo`, `useMemo`, `useCallback` — kiến thức tái sử dụng 100%

Toàn bộ nguyên tắc tối ưu re-render bạn đã biết từ React (tránh tạo object/hàm mới mỗi render, `React.memo` cho component con, `useMemo` cho tính toán nặng) **áp dụng y hệt** trong RN. Điểm khác biệt là **mức độ khắt khe**: vì RN không có Virtual DOM diffing hiệu quả như React DOM cho các thay đổi style nhỏ, và mỗi lần cập nhật UI cần phản ánh qua Fabric xuống native view thật, chi phí re-render trên RN có xu hướng cao hơn tương đối trên thiết bị yếu.

### 8.2.2. Ví dụ production: Tối ưu `CourseListItem` trong danh sách dài

```tsx
// src/components/course/CourseListItem.tsx (đã tối ưu)
import { memo } from 'react';

export const CourseListItem = memo(
  function CourseListItem({ course, onPress, onEdit, onDelete }: CourseListItemProps) {
    // ... nội dung như Chương 4
  },
  // So sánh tùy chỉnh: chỉ re-render khi các field thực sự hiển thị thay đổi,
  // bỏ qua field không ảnh hưởng UI (vd: estimatedRevenue không hiển thị ở đây)
  (prevProps, nextProps) => {
    return (
      prevProps.course.courseId === nextProps.course.courseId &&
      prevProps.course.courseName === nextProps.course.courseName &&
      prevProps.course.image === nextProps.course.image &&
      prevProps.course.tuitionFee === nextProps.course.tuitionFee
    );
  }
);
```

### 8.2.3. Lỗi phổ biến: tạo hàm inline trong `renderItem`

```tsx
// SAI — tạo hàm mới mỗi lần FlashList render lại (khiến React.memo của CourseListItem vô nghĩa)
<FlashList
  renderItem={({ item }) => (
    <CourseListItem course={item} onPress={() => router.push(`/course/${item.courseId}`)} />
  )}
/>
```

```tsx
// ĐÚNG — dùng useCallback để hàm callback ổn định giữa các lần render
const handlePress = useCallback(
  (courseId: string) => router.push(`/course/${courseId}`),
  [router]
);

<FlashList
  renderItem={({ item }) => <CourseListItem course={item} onPress={() => handlePress(item.courseId!)} />}
/>
```

> **Lưu ý:** Ngay cả với `useCallback` ở `handlePress`, arrow function `() => handlePress(item.courseId!)` bên trong `renderItem` **vẫn tạo hàm mới mỗi lần**. Giải pháp triệt để nhất là truyền `id` qua prop riêng và để component tự gọi, hoặc dùng thư viện quản lý callback theo id (`useEvent` pattern) — với danh sách rất lớn (>500 item) nên cân nhắc kỹ mức tối ưu này.

---

## 8.3. FlashList nâng cao

### 8.3.1. Nhắc lại và mở rộng từ Chương 2

Ở Chương 2 đã giới thiệu `estimatedItemSize` là bắt buộc. Dưới đây là các kỹ thuật tối ưu sâu hơn cho danh sách khóa học với nhiều loại item khác nhau (vd: item quảng cáo xen giữa danh sách):

```tsx
// src/components/course/CourseListWithAds.tsx
import { FlashList } from '@shopify/flash-list';

type ListItem = { type: 'course'; data: Course } | { type: 'ad'; data: AdBanner };

<FlashList
  data={mixedData}
  // getItemType giúp FlashList tái sử dụng "khuôn" (recycling pool) đúng loại,
  // tránh việc tái sử dụng nhầm giữa card khóa học và banner quảng cáo (kích thước khác nhau)
  getItemType={(item) => item.type}
  renderItem={({ item }) =>
    item.type === 'course' ? <CourseListItem course={item.data} /> : <AdBannerItem ad={item.data} />
  }
  estimatedItemSize={100}
/>
```

### 8.3.2. So sánh hiệu năng FlatList vs FlashList (số liệu tham khảo thực tế)

| Chỉ số | FlatList | FlashList |
|---|---|---|
| Cơ chế tái sử dụng cell | Tái tạo view khi cuộn nhanh | Tái sử dụng view có sẵn (giống RecyclerView Android) |
| Bộ nhớ tiêu thụ với list 1000+ item | Cao hơn | Thấp hơn đáng kể |
| Thời gian render frame đầu | Chậm hơn với list lớn | Nhanh hơn ~5-10 lần theo benchmark của Shopify |
| Yêu cầu cấu hình | Không cần | Bắt buộc `estimatedItemSize` |

---

## 8.4. Hermes Engine & Bundle Size

### 8.4.1. Hermes là gì

Hermes là JS Engine do Meta phát triển riêng cho React Native (thay thế JavaScriptCore mặc định của iOS/V8), tối ưu cho: thời gian khởi động app nhanh hơn (precompile bytecode thay vì parse JS runtime), bộ nhớ tiêu thụ thấp hơn, kích thước app nhỏ hơn. Từ RN 0.70+, Hermes là **mặc định** — bạn hầu như không cần cấu hình gì thêm, nhưng cần hiểu để debug đúng công cụ (mục 8.5).

### 8.4.2. Giảm kích thước Bundle — checklist production

```bash
# Kiểm tra kích thước bundle trước khi build production
npx expo export --platform android
npx react-native-bundle-visualizer  # Trực quan hoá package nào chiếm nhiều dung lượng nhất
```

| Kỹ thuật | Tác dụng |
|---|---|
| Xóa `console.log` khi build production | Giảm nhẹ bundle, tránh lộ thông tin debug |
| Tránh import toàn bộ thư viện (`import _ from 'lodash'`) | Dùng `import debounce from 'lodash/debounce'` để tree-shake |
| Lazy load màn hình ít dùng | Dùng `React.lazy` + `Suspense` (RN 0.75+ hỗ trợ) hoặc chia theo route Expo Router (tự động code-split theo file) |
| Nén ảnh trong `assets/` | Dùng định dạng WebP thay PNG khi có thể |
| Kiểm tra thư viện native trùng lặp | 2 thư viện cùng làm 1 việc (vd: 2 thư viện date) tăng đáng kể kích thước |

---

## 8.5. Debug hiệu năng: Flipper / React Native DevTools

### 8.5.1. React Native DevTools (công cụ mặc định hiện nay)

Từ RN 0.74+, **React Native DevTools** (dựa trên Chrome DevTools Protocol) đã thay thế Flipper làm công cụ debug mặc định — mở bằng phím `j` trong terminal Metro đang chạy hoặc qua Expo Dev Menu (lắc thiết bị hoặc `Cmd+D`/`Cmd+M`).

```
npx expo start
# Nhấn "j" trong terminal → mở React Native DevTools
```

**Các tab quan trọng cho performance:**

| Tab | Công dụng | Tương đương Web DevTools |
|---|---|---|
| Console | Log, warning, error | Console tab |
| Network | Theo dõi request Axios | Network tab |
| React Profiler | Đo thời gian render component, phát hiện re-render thừa | React DevTools Profiler (extension) |
| Memory | Snapshot bộ nhớ, phát hiện memory leak | Memory tab |

### 8.5.2. Đo hiệu năng render với React Profiler

```tsx
// Bọc tạm thời quanh vùng nghi ngờ re-render thừa để đo — gỡ trước khi commit production
import { Profiler } from 'react';

<Profiler
  id="CourseList"
  onRender={(id, phase, actualDuration) => {
    console.log(`${id} [${phase}] mất ${actualDuration.toFixed(2)}ms`);
  }}
>
  <FlashList data={courses} renderItem={renderItem} />
</Profiler>
```

### 8.5.3. Kiểm tra FPS trực tiếp trên thiết bị

```
Mở Dev Menu (lắc thiết bị) → Show Perf Monitor
```

Hiển thị 2 chỉ số song song:
- **JS FPS**: tốc độ khung hình của JS Thread (nơi logic React chạy)
- **UI FPS**: tốc độ khung hình của Native/UI Thread (nơi Reanimated chạy — xem lại Chương 7)

**Nguyên tắc chẩn đoán:** Nếu `UI FPS` mượt (gần 60) nhưng `JS FPS` tụt — vấn đề nằm ở logic JS (re-render thừa, tính toán nặng trong component). Nếu cả hai đều tụt — thường là do animation không dùng Reanimated (chạy qua Bridge cũ) hoặc render quá nhiều native view cùng lúc (danh sách không ảo hóa đúng cách — xem lại nguyên tắc `FlatList`/`FlashList` ở Chương 2).

---

## 8.6. Checklist Performance trước khi Release

| Hạng mục | Đã kiểm tra? |
|---|---|
| Mọi danh sách dài dùng `FlashList`/`FlatList`, không dùng `.map()` trong `ScrollView` | ☐ |
| Component trong `renderItem` đã `React.memo` | ☐ |
| Không có `console.log` trong code production | ☐ |
| Ảnh đã nén/resize trước khi hiển thị và upload | ☐ |
| Animation phức tạp dùng Reanimated, không dùng `Animated` gốc | ☐ |
| Đã test trên thiết bị Android tầm trung (không chỉ test trên máy cấu hình cao/simulator) | ☐ |
| Đã kiểm tra bundle size bằng `bundle-visualizer` | ☐ |
| Không có memory leak (kiểm tra qua DevTools Memory tab khi chuyển màn hình liên tục) | ☐ |

---

## Tổng kết Chương 8

| Chủ đề | Nguyên tắc Web | Điều chỉnh cho React Native |
|---|---|---|
| Re-render | `React.memo`, `useMemo`, `useCallback` | Giống hệt, nhưng khắt khe hơn với danh sách dài |
| Danh sách dài | Virtualization tùy chọn (thường trình duyệt xử lý ổn) | `FlashList` gần như bắt buộc |
| JS Engine | V8 (trình duyệt lo) | Hermes — hiểu để debug đúng, ít khi cần cấu hình |
| Công cụ debug | Chrome DevTools | React Native DevTools (Chrome DevTools Protocol, tương tự giao diện) |
| Chỉ số cần theo dõi riêng | FPS chung | Tách JS FPS và UI FPS để chẩn đoán đúng nguyên nhân |

**Chương tiếp theo (Chương 9)** chuyển sang Testing & CI/CD — viết unit test, E2E test, và quy trình build/release tự động bằng EAS.
