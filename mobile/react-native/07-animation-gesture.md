# GIÁO TRÌNH REACT NATIVE
### Dành cho lập trình viên đã có nền tảng React / Next.js

---

# CHƯƠNG 7: ANIMATION & GESTURE

## 7.1. Vì sao cần Reanimated thay vì `Animated` gốc

### 7.1.1. Khái niệm và vấn đề cốt lõi

React Native có sẵn API `Animated`, nhưng theo mặc định, animation được **tính toán trên JS Thread** rồi gửi lệnh qua Bridge đến Native Thread ở mỗi frame (60 lần/giây). Nếu JS Thread đang bận (vd: xử lý API call, re-render danh sách), animation sẽ **giật (jank)** — vấn đề gần như không tồn tại với CSS animation trên Web (CSS animation chạy hoàn toàn ở compositor thread của trình duyệt, độc lập với JS).

**Reanimated** giải quyết vấn đề này bằng cách cho phép định nghĩa animation logic chạy **trực tiếp trên UI Thread** (thông qua JSI đã học ở Chương 1), không phụ thuộc JS Thread — đảm bảo animation mượt 60fps ngay cả khi JS Thread đang tải nặng.

### 7.1.2. So sánh với Framer Motion / CSS Animation (Web)

| Tiêu chí | Framer Motion / CSS (Web) | Reanimated (RN) |
|---|---|---|
| Nơi chạy animation | Compositor thread (trình duyệt) | UI Thread (native, qua JSI) |
| Cú pháp khai báo | Declarative (`animate={{ x: 100 }}`) | Hybrid: hook + Worklet (hàm chạy trên UI thread) |
| Gesture kết hợp | `whileDrag`, `useDrag` | `react-native-gesture-handler` + Reanimated |
| Layout animation tự động | `layout` prop (Framer Motion) | `Layout` animation API (Reanimated 3) |

```bash
npx expo install react-native-reanimated
```

---

## 7.2. Reanimated 3 — Kiến thức nền tảng

### 7.2.1. Shared Value — tương đương `useState` nhưng chạy trên UI Thread

```tsx
import { useSharedValue, useAnimatedStyle, withTiming } from 'react-native-reanimated';
import Animated from 'react-native-reanimated';
import { Pressable } from 'react-native';

export function LikeButton() {
  const scale = useSharedValue(1); // Giống useState, nhưng giá trị "sống" trên UI Thread

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  const handlePress = () => {
    scale.value = withTiming(1.3, { duration: 150 }, () => {
      scale.value = withTiming(1, { duration: 150 });
    });
  };

  return (
    <Pressable onPress={handlePress}>
      <Animated.View style={animatedStyle} className="size-12 bg-primary rounded-full" />
    </Pressable>
  );
}
```

**Giải thích khái niệm cốt lõi:**
- `useSharedValue`: tạo một giá trị "sống" đồng thời ở cả JS Thread lẫn UI Thread — đọc/ghi `.value` không gây re-render React như `useState`, giúp animation mượt tuyệt đối.
- `useAnimatedStyle`: một **Worklet** — hàm được Reanimated biên dịch để chạy trực tiếp trên UI Thread, tương tự việc CSS Animation chạy trên compositor thread mà không cần JS can thiệp mỗi frame.
- `withTiming`: hàm tạo animation dạng easing theo thời gian — tương đương `transition: { duration: ... }` trong Framer Motion.

### 7.2.2. Ví dụ production: Card khóa học có hiệu ứng nhấn (áp dụng vào `CourseListItem` Chương 4)

```tsx
// src/components/course/AnimatedCourseCard.tsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
} from 'react-native-reanimated';
import { Pressable } from 'react-native';

export function AnimatedPressable({ children, onPress }: { children: React.ReactNode; onPress: () => void }) {
  const scale = useSharedValue(1);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  return (
    <Pressable
      onPressIn={() => (scale.value = withSpring(0.97))}
      onPressOut={() => (scale.value = withSpring(1))}
      onPress={onPress}
    >
      <Animated.View style={animatedStyle}>{children}</Animated.View>
    </Pressable>
  );
}
```

**So sánh:** Đây là bản RN của hiệu ứng `active:scale-95` mà bạn có thể đã dùng trong Tailwind (`transition-transform` + `:active`) — nhưng vì RN không có pseudo-class CSS, hiệu ứng "nhấn co lại" phải tự dựng bằng `onPressIn`/`onPressOut` kết hợp Shared Value.

### 7.2.3. Layout Animation — tự động animate khi danh sách thay đổi

```tsx
// src/components/course/CourseListItem.tsx (bổ sung animation khi xóa item khỏi list)
import Animated, { FadeIn, FadeOut, Layout } from 'react-native-reanimated';

export function CourseListItem({ course, ...props }: CourseListItemProps) {
  return (
    <Animated.View
      entering={FadeIn.duration(300)}   // Animation khi item xuất hiện (vd: sau khi tạo mới)
      exiting={FadeOut.duration(200)}   // Animation khi item biến mất (vd: sau khi xóa)
      layout={Layout.springify()}       // Tự động animate vị trí các item còn lại khi 1 item bị xóa
    >
      {/* Nội dung card giữ nguyên như Chương 4 */}
    </Animated.View>
  );
}
```

**Giải thích:** `entering`/`exiting`/`layout` là bộ API "Layout Animation" của Reanimated 3 — tương đương `layout` prop hoặc `AnimatePresence` của Framer Motion, tự động tính toán và animate sự thay đổi vị trí/kích thước mà không cần lập trình viên tự viết animation thủ công cho từng trường hợp.

### 7.2.4. Skeleton Loading — thay thế `<Skeleton>` shadcn

```tsx
// src/components/ui/Skeleton.tsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withRepeat,
  withTiming,
  Easing,
} from 'react-native-reanimated';
import { useEffect } from 'react';
import { View, ViewProps } from 'react-native';

export function Skeleton({ className, ...props }: ViewProps & { className?: string }) {
  const opacity = useSharedValue(0.3);

  useEffect(() => {
    opacity.value = withRepeat(
      withTiming(0.7, { duration: 800, easing: Easing.inOut(Easing.ease) }),
      -1, // Lặp vô hạn
      true // Đảo chiều mỗi lần lặp (tạo hiệu ứng nhấp nháy qua lại)
    );
  }, []);

  const animatedStyle = useAnimatedStyle(() => ({ opacity: opacity.value }));

  return <Animated.View style={animatedStyle} className={`bg-gray-200 rounded-xl ${className}`} {...props} />;
}
```

```tsx
// Sử dụng thay cho CourseListSkeleton trong page.tsx gốc
<View className="gap-3">
  {[1, 2, 3].map((i) => (
    <View key={i} className="flex-row gap-3 p-3 bg-white rounded-2xl">
      <Skeleton className="size-20" />
      <View className="flex-1 gap-2 justify-center">
        <Skeleton className="h-4 w-3/4" />
        <Skeleton className="h-3 w-1/2" />
      </View>
    </View>
  ))}
</View>
```

---

## 7.3. Gesture Handler

### 7.3.1. Khái niệm

`react-native-gesture-handler` xử lý cử chỉ chạm (tap, pan, pinch, swipe, long-press) ở **native thread**, khác với hệ thống Touch event gốc của RN (chạy qua JS Thread, dễ giật khi thao tác phức tạp). Không có khái niệm tương đương chính xác trên Web — gần giống việc `touch-action` CSS kết hợp Pointer Events, nhưng RN Gesture Handler mạnh và thống nhất hơn nhiều.

```bash
npx expo install react-native-gesture-handler
```

### 7.3.2. Ví dụ production: Vuốt để xóa (Swipe-to-delete) — nâng cấp `CourseListItem`

Đây là pattern UX rất phổ biến trên mobile (thay thế hoàn toàn cho nút "Xóa" trong dropdown menu ở bản Web):

```tsx
// src/components/course/SwipeableCourseItem.tsx
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  runOnJS,
} from 'react-native-reanimated';
import { View, Text } from 'react-native';
import { TrashIcon } from 'lucide-react-native';

interface SwipeableCourseItemProps {
  children: React.ReactNode;
  onDelete: () => void;
}

const DELETE_THRESHOLD = -80;

export function SwipeableCourseItem({ children, onDelete }: SwipeableCourseItemProps) {
  const translateX = useSharedValue(0);

  const panGesture = Gesture.Pan()
    .onUpdate((event) => {
      // Chỉ cho vuốt sang trái (giá trị âm), không cho vuốt sang phải
      translateX.value = Math.min(0, event.translationX);
    })
    .onEnd(() => {
      if (translateX.value < DELETE_THRESHOLD) {
        translateX.value = withTiming(-1000, {}, () => {
          runOnJS(onDelete)(); // Gọi hàm JS từ Worklet chạy trên UI Thread
        });
      } else {
        translateX.value = withTiming(0); // Bật lại vị trí cũ nếu vuốt chưa đủ xa
      }
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }],
  }));

  return (
    <View className="relative">
      <View className="absolute right-0 top-0 bottom-0 w-20 bg-red-500 items-center justify-center rounded-2xl">
        <TrashIcon color="white" size={20} />
      </View>
      <GestureDetector gesture={panGesture}>
        <Animated.View style={animatedStyle}>{children}</Animated.View>
      </GestureDetector>
    </View>
  );
}
```

```tsx
// Áp dụng vào màn hình danh sách khóa học (thay thế Context Menu long-press nếu muốn UX vuốt-xóa)
<SwipeableCourseItem onDelete={() => deleteMutation.mutate(item.courseId!)}>
  <CourseListItem course={item} onPress={() => router.push(`/course/${item.courseId}`)} />
</SwipeableCourseItem>
```

**Giải thích các điểm kỹ thuật quan trọng:**
- `Gesture.Pan()`: khai báo cử chỉ kéo — API declarative mới của Gesture Handler v2, thay thế `PanResponder` cũ (khó dùng, hiệu năng kém hơn).
- `runOnJS(onDelete)()`: vì `onUpdate`/`onEnd` là Worklet chạy trên **UI Thread**, không thể gọi trực tiếp hàm JS thường (như `onDelete` — vốn gọi `deleteMutation.mutate`, một hàm JS Thread bình thường). `runOnJS` là cầu nối bắt buộc để "nhảy" từ UI Thread về JS Thread khi cần thực thi logic JS thường (gọi API, cập nhật state React).
- Đây chính là minh chứng thực tế cho kiến trúc JSI đã học ở Chương 1 — animation/gesture chạy độc lập UI Thread, chỉ "giao tiếp ngược" về JS Thread khi thực sự cần thiết (gọi API xóa).

---

## 7.4. Lottie — Animation phức tạp từ After Effects

### 7.4.1. Khái niệm

Lottie là thư viện render animation dạng JSON (xuất từ After Effects qua plugin Bodymovin) — dùng cho các animation minh họa phức tạp (illustration, loading animation đặc thù thương hiệu) mà tự code bằng Reanimated sẽ tốn công. Có phiên bản chạy trên cả Web (`lottie-web`) lẫn RN (`lottie-react-native`) — cùng 1 file JSON dùng chung được cho cả hai nền tảng.

```bash
npx expo install lottie-react-native
```

```tsx
// src/components/EmptyState.tsx
import LottieView from 'lottie-react-native';
import { View, Text } from 'react-native';

export function EmptyCoursesState() {
  return (
    <View className="items-center justify-center py-10">
      <LottieView
        source={require('@/assets/animations/empty-box.json')}
        autoPlay
        loop
        style={{ width: 160, height: 160 }}
      />
      <Text className="text-gray-400 mt-2">Chưa có khóa học nào</Text>
    </View>
  );
}
```

**So sánh với Web:** Nếu dự án Next.js của bạn từng dùng `lottie-react` cho minh họa trạng thái rỗng (empty state) hoặc loading, đây chính là **cùng 1 file JSON animation** có thể tái sử dụng, chỉ đổi component render.

---

## Tổng kết Chương 7

| Nhu cầu | Web | React Native |
|---|---|---|
| Animation mượt, không giật khi JS bận | Compositor thread (mặc định của CSS) | Reanimated (chạy trên UI Thread qua JSI) |
| Hiệu ứng nhấn nút | `:active` + `transition` CSS | `onPressIn`/`onPressOut` + Shared Value |
| Animation khi list thay đổi | `AnimatePresence` (Framer Motion) | `entering`/`exiting`/`layout` (Reanimated Layout Animation) |
| Vuốt để xóa | Không phổ biến trên Web | `react-native-gesture-handler` + `Gesture.Pan()` |
| Animation minh họa phức tạp | `lottie-web` | `lottie-react-native` — dùng chung file JSON |
| Gọi logic JS từ animation | Không cần phân biệt thread | Bắt buộc `runOnJS()` khi ở trong Worklet |

**Chương tiếp theo (Chương 8)** đi vào Performance — tối ưu re-render, cấu hình FlashList nâng cao, hiểu Hermes Engine, và bộ công cụ debug hiệu năng chuyên dụng cho mobile.
