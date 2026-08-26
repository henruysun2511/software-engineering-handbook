# GIÁO TRÌNH REACT NATIVE
### Dành cho lập trình viên đã có nền tảng React / Next.js

# CHƯƠNG 2: CORE COMPONENTS & STYLING

## 2.1. Các Component nền tảng

### 2.1.1. Khái niệm

Trong React Native, không tồn tại thẻ HTML. Thay vào đó, RN cung cấp sẵn một bộ **Core Components** ánh xạ sang native view tương ứng trên từng nền tảng.

| RN Component | Tương đương Web | Native thực thi (iOS/Android) |
|---|---|---|
| `<View>` | `<div>` | `UIView` / `android.view.View` |
| `<Text>` | `<span>`/`<p>` | `UILabel` / `TextView` |
| `<Image>` | `<img>` | `UIImageView` / `ImageView` |
| `<ScrollView>` | `overflow: scroll` | `UIScrollView` / `ScrollView` |
| `<FlatList>` / `<FlashList>` | Virtualized list (không có tương đương HTML thuần) | `RecyclerView`-like |
| `<Pressable>` / `<TouchableOpacity>` | `<button>` | Touch gesture handler |
| `<TextInput>` | `<input>` | `UITextField` / `EditText` |

### 2.1.2. Nguyên tắc quan trọng cần nắm

1. **Mọi văn bản BẮT BUỘC phải nằm trong `<Text>`.** Khác với Web (text tự do trong `<div>`), RN sẽ báo lỗi runtime nếu bạn đặt chuỗi text trực tiếp trong `<View>`.
2. **`<View>` mặc định không cuộn (scroll)** — muốn cuộn phải dùng `<ScrollView>` hoặc `<FlatList>`.
3. **Danh sách dài PHẢI dùng `<FlatList>`/`<FlashList>`**, không dùng `.map()` trong `<ScrollView>` — vì `.map()` render toàn bộ item cùng lúc (không ảo hóa), gây tràn bộ nhớ với danh sách lớn.

### 2.1.3. Ví dụ production: Danh sách sản phẩm

```tsx
// src/components/ProductList.tsx
import { FlatList, View, Text, Image, Pressable } from 'react-native';
import { useRouter } from 'expo-router';

interface Product {
  id: string;
  name: string;
  price: number;
  imageUrl: string;
}

interface ProductListProps {
  products: Product[];
  isLoading?: boolean;
}

export function ProductList({ products, isLoading }: ProductListProps) {
  const router = useRouter();

  if (isLoading) {
    return (
      <View className="flex-1 items-center justify-center">
        <Text className="text-gray-500">Đang tải...</Text>
      </View>
    );
  }

  return (
    <FlatList
      data={products}
      keyExtractor={(item) => item.id}
      // contentContainerStyle thay cho padding trực tiếp trên FlatList
      contentContainerStyle={{ padding: 16, gap: 12 }}
      renderItem={({ item }) => (
        <Pressable
          onPress={() => router.push(`/product/${item.id}`)}
          className="flex-row items-center bg-white rounded-xl p-3 shadow-sm active:opacity-70"
        >
          <Image
            source={{ uri: item.imageUrl }}
            className="w-16 h-16 rounded-lg"
            resizeMode="cover"
          />
          <View className="ml-3 flex-1">
            <Text className="text-base font-semibold text-gray-900" numberOfLines={1}>
              {item.name}
            </Text>
            <Text className="text-sm text-gray-500 mt-1">
              {item.price.toLocaleString('vi-VN')} đ
            </Text>
          </View>
        </Pressable>
      )}
      // Tối ưu hiệu năng cho danh sách dài
      initialNumToRender={10}
      windowSize={5}
      removeClippedSubviews
    />
  );
}
```

**Giải thích các điểm production quan trọng:**
- `keyExtractor`: bắt buộc để FlatList tối ưu re-render, tương tự `key` trong `.map()` của React nhưng RN yêu cầu tường minh qua prop riêng.
- `contentContainerStyle` vs `style`: `style` áp cho khung chứa list (container ngoài), `contentContainerStyle` áp cho nội dung bên trong (giống `padding` cho `<ul>` khi có scroll) — nhầm lẫn hai prop này là lỗi rất phổ biến.
- `active:opacity-70` (NativeWind): xử lý trạng thái nhấn giống pseudo-class `:active` trên Web.
- `numberOfLines={1}`: RN không tự có `text-overflow: ellipsis` qua CSS thường — phải khai báo tường minh qua prop này.
- `initialNumToRender`, `windowSize`, `removeClippedSubviews`: các prop tối ưu hiệu năng đặc thù của FlatList, không có tương đương bên Web.

> **Khuyến nghị production:** Với danh sách rất dài (>100 item) hoặc cần hiệu năng cuộn mượt tuyệt đối, nên thay `FlatList` bằng **`FlashList`** (thư viện của Shopify), API gần như tương thích 100% nhưng hiệu năng vượt trội nhờ cơ chế tái sử dụng cell thông minh hơn.

```bash
npx expo install @shopify/flash-list
```

```tsx
import { FlashList } from '@shopify/flash-list';

<FlashList
  data={products}
  renderItem={({ item }) => <ProductCard product={item} />}
  estimatedItemSize={88} // BẮT BUỘC: ước lượng chiều cao 1 item để tối ưu tính toán
/>
```

---

## 2.2. Styling: StyleSheet API và NativeWind

### 2.2.1. StyleSheet API — nền tảng gốc

React Native dùng cú pháp giống CSS nhưng viết bằng object JavaScript (camelCase thay vì kebab-case), không có CSS file thật.

```tsx
import { View, Text, StyleSheet } from 'react-native';

export function Card() {
  return (
    <View style={styles.card}>
      <Text style={styles.title}>Tiêu đề</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  card: {
    backgroundColor: '#ffffff',
    borderRadius: 12,
    padding: 16,
    // Shadow trên iOS và Android hoạt động khác nhau — phải khai báo cả hai
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 8,
    elevation: 3, // elevation là thuộc tính riêng của Android
  },
  title: {
    fontSize: 16,
    fontWeight: '600', // fontWeight là STRING, không phải number như CSS Web
    color: '#111827',
  },
});
```

**Điểm khác biệt cần ghi nhớ so với CSS Web:**

| CSS Web | React Native StyleSheet | Ghi chú |
|---|---|---|
| `background-color` | `backgroundColor` | camelCase |
| `font-weight: 600` | `fontWeight: '600'` | Bắt buộc là string |
| Không có | `elevation` (Android) | Thay cho box-shadow trên Android |
| `display: flex` (mặc định `row`) | Mặc định **`flexDirection: 'column'`** | Khác biệt lớn nhất, dễ gây bug layout |
| `%`, `vh`, `vw` | Chỉ hỗ trợ `%` cho width/height, không có `vh/vw` | Dùng `Dimensions`/`useWindowDimensions` thay thế |
| Pseudo-class `:hover` | Không tồn tại (thiết bị cảm ứng) | Thay bằng `onPressIn`/`onPressOut` hoặc trạng thái `pressed` |

### 2.2.2. NativeWind — Tailwind CSS cho React Native

**Khái niệm:** NativeWind biên dịch class Tailwind (viết trong prop `className`) thành `StyleSheet` object native tại thời điểm build, giúp bạn **tái sử dụng gần như nguyên vẹn tư duy Tailwind đã có**.

**Cài đặt (v4 — chuẩn hiện tại):**

```bash
npm install nativewind tailwindcss
```

```js
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ['./app/**/*.{js,jsx,ts,tsx}', './src/**/*.{js,jsx,ts,tsx}'],
  presets: [require('nativewind/preset')],
  theme: {
    extend: {
      colors: {
        primary: '#4F46E5',
      },
    },
  },
  plugins: [],
};
```

```css
/* src/global.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

```tsx
// app/_layout.tsx
import '../src/global.css';
import { Stack } from 'expo-router';

export default function RootLayout() {
  return <Stack />;
}
```

**So sánh mức độ tương thích:**

| Tính năng Tailwind (Web) | Hỗ trợ trong NativeWind | Ghi chú |
|---|---|---|
| Utility class cơ bản (`p-4`, `flex-1`, `text-lg`) | ✅ Đầy đủ | |
| Responsive breakpoint (`md:`, `lg:`) | ✅ (theo `useWindowDimensions`) | Dùng cho tablet |
| Dark mode (`dark:`) | ✅ | Theo `useColorScheme` |
| `hover:` | ⚠️ Chỉ hiệu lực khi chạy bản web (react-native-web) | Mobile không có hover |
| `active:` | ✅ | Ánh xạ sang trạng thái pressed |
| Animation utilities (`animate-spin`) | ⚠️ Hạn chế | Nên dùng Reanimated cho animation phức tạp (Chương 7) |
| Arbitrary values (`w-[123px]`) | ✅ | |

### 2.2.3. Ví dụ production: Form đăng nhập với NativeWind

```tsx
// src/components/LoginForm.tsx
import { View, Text, TextInput, Pressable, ActivityIndicator } from 'react-native';
import { useState } from 'react';

interface LoginFormProps {
  onSubmit: (email: string, password: string) => void;
  isSubmitting: boolean;
  errorMessage?: string;
}

export function LoginForm({ onSubmit, isSubmitting, errorMessage }: LoginFormProps) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  return (
    <View className="w-full px-6">
      <Text className="text-2xl font-bold text-gray-900 mb-6">Đăng nhập</Text>

      <View className="mb-4">
        <Text className="text-sm font-medium text-gray-700 mb-1.5">Email</Text>
        <TextInput
          value={email}
          onChangeText={setEmail}
          placeholder="you@example.com"
          keyboardType="email-address"
          autoCapitalize="none"
          className="border border-gray-300 rounded-lg px-4 py-3 text-base
                     focus:border-primary"
        />
      </View>

      <View className="mb-6">
        <Text className="text-sm font-medium text-gray-700 mb-1.5">Mật khẩu</Text>
        <TextInput
          value={password}
          onChangeText={setPassword}
          placeholder="••••••••"
          secureTextEntry
          className="border border-gray-300 rounded-lg px-4 py-3 text-base"
        />
      </View>

      {errorMessage ? (
        <Text className="text-sm text-red-500 mb-4">{errorMessage}</Text>
      ) : null}

      <Pressable
        onPress={() => onSubmit(email, password)}
        disabled={isSubmitting}
        className="bg-primary rounded-lg py-3.5 items-center active:opacity-80 disabled:opacity-50"
      >
        {isSubmitting ? (
          <ActivityIndicator color="white" />
        ) : (
          <Text className="text-white font-semibold text-base">Đăng nhập</Text>
        )}
      </Pressable>
    </View>
  );
}
```

**Giải thích các điểm production quan trọng:**
- `keyboardType="email-address"`: RN điều khiển bàn phím ảo qua prop, không có `<input type="email">` như Web.
- `secureTextEntry`: thay cho `type="password"`.
- `ActivityIndicator`: component loading spinner gốc của RN, không dùng SVG animation như Web thường làm.
- `disabled:opacity-50` kết hợp `disabled={isSubmitting}`: ngăn double-submit — vấn đề rất hay gặp trên mobile vì người dùng dễ chạm nhiều lần khi mạng chậm.
- Component này là **controlled component thuần túy**, logic gọi API tách biệt hoàn toàn (sẽ tích hợp với TanStack Query ở Chương 4) — tuân thủ nguyên tắc tách UI khỏi logic mà bạn đã quen trong React.

---

## 2.3. Flexbox trong React Native

### 2.3.1. Khái niệm và khác biệt cốt lõi

React Native dùng chung thuật toán Flexbox với Web (Yoga Layout Engine), nhưng có **một khác biệt mặc định gây nhầm lẫn nhiều nhất** với người đã quen CSS Web:

```
Web:            display: flex  →  flex-direction mặc định là "row"
React Native:   mọi View đều là flex container → flexDirection mặc định là "column"
```

### 2.3.2. Bảng đối chiếu thuộc tính Flexbox

| Thuộc tính | Web (mặc định) | React Native (mặc định) |
|---|---|---|
| `flexDirection` | `row` | **`column`** |
| `flexShrink` | `1` | **`0`** |
| Đơn vị `flex` | Cần `display:flex` khai báo trước | Áp dụng ngay, mọi View đều flexible |
| `gap` | Hỗ trợ từ CSS Flexbox Gap | Hỗ trợ từ RN 0.71+ |

### 2.3.3. Ví dụ production: Layout dashboard responsive

```tsx
// src/components/DashboardStats.tsx
import { View, Text } from 'react-native';

interface StatCardProps {
  label: string;
  value: string;
}

function StatCard({ label, value }: StatCardProps) {
  return (
    // flex-1 giúp 3 card chia đều không gian ngang — giống flex: 1 trên Web
    <View className="flex-1 bg-white rounded-xl p-4 shadow-sm">
      <Text className="text-xs text-gray-500">{label}</Text>
      <Text className="text-xl font-bold text-gray-900 mt-1">{value}</Text>
    </View>
  );
}

export function DashboardStats() {
  return (
    // flex-row BẮT BUỘC phải khai báo tường minh vì mặc định là column
    <View className="flex-row gap-3 px-4">
      <StatCard label="Đơn hàng" value="128" />
      <StatCard label="Doanh thu" value="45.2M" />
      <StatCard label="Khách mới" value="12" />
    </View>
  );
}
```

### 2.3.4. Bảng thuộc tính Flexbox thường dùng trong production

| Thuộc tính | Giá trị phổ biến | Công dụng |
|---|---|---|
| `flex: 1` | Chiếm hết không gian còn lại | Giống `flex-1` Tailwind, dùng cho container chính |
| `justifyContent` | `flex-start`, `center`, `space-between` | Căn theo trục chính (main axis) |
| `alignItems` | `flex-start`, `center`, `stretch` | Căn theo trục phụ (cross axis) |
| `flexWrap: 'wrap'` | | Cho phép xuống dòng, dùng khi làm grid bằng flex |
| `gap` | số nguyên (px logic) | Khoảng cách giữa các item, thay thế margin thủ công |

---

## 2.4. Responsive Design: Dimensions, Safe Area

### 2.4.1. Dimensions và useWindowDimensions

**Khái niệm:** Vì không có `vh`/`vw`/media query như CSS Web, RN cung cấp API để lấy kích thước màn hình runtime.

```tsx
// src/hooks/useResponsive.ts
import { useWindowDimensions } from 'react-native';

const BREAKPOINT_TABLET = 768;

export function useResponsive() {
  const { width, height } = useWindowDimensions();

  return {
    width,
    height,
    isTablet: width >= BREAKPOINT_TABLET,
    // Số cột grid động theo kích thước màn hình
    numColumns: width >= BREAKPOINT_TABLET ? 3 : 2,
  };
}
```

```tsx
// Sử dụng trong component
import { FlatList } from 'react-native';
import { useResponsive } from '../hooks/useResponsive';

export function ProductGrid({ products }: { products: Product[] }) {
  const { numColumns } = useResponsive();

  return (
    <FlatList
      data={products}
      numColumns={numColumns}
      key={numColumns} // Bắt buộc: FlatList cần remount khi đổi numColumns
      renderItem={({ item }) => <ProductCard product={item} />}
    />
  );
}
```

> **Lưu ý kỹ thuật quan trọng:** `useWindowDimensions` là **hook**, tự động re-render khi xoay màn hình (orientation change) — ưu tiên dùng hook này thay vì `Dimensions.get('window')` (API tĩnh, không tự cập nhật, dễ gây bug khi người dùng xoay ngang thiết bị).

### 2.4.2. Safe Area — khái niệm không tồn tại trên Web

**Khái niệm:** Các thiết bị hiện đại có "tai thỏ" (notch), thanh trạng thái (status bar), thanh cử chỉ (home indicator). "Safe Area" là vùng màn hình đảm bảo nội dung không bị các phần tử hệ thống này che khuất.

```bash
npx expo install react-native-safe-area-context
```

```tsx
// app/_layout.tsx
import { SafeAreaProvider } from 'react-native-safe-area-context';
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <SafeAreaProvider>
      <Stack />
    </SafeAreaProvider>
  );
}
```

```tsx
// app/(tabs)/index.tsx
import { SafeAreaView } from 'react-native-safe-area-context';
import { View, Text } from 'react-native';

export default function HomeScreen() {
  return (
    // edges chỉ định cạnh nào cần tránh — tránh phủ toàn bộ khiến tab bar bị đẩy sai
    <SafeAreaView edges={['top']} className="flex-1 bg-white">
      <View className="px-4 py-3">
        <Text className="text-2xl font-bold">Trang chủ</Text>
      </View>
    </SafeAreaView>
  );
}
```

**Giải thích:** `edges={['top']}` chỉ áp dụng padding an toàn cho cạnh trên (tránh status bar/notch) vì cạnh dưới đã được `Tab Navigator` tự xử lý riêng. Áp dụng `SafeAreaView` bao toàn bộ mọi màn hình mà không chỉ định `edges` là lỗi phổ biến gây thừa khoảng trắng không mong muốn.

---

## Tổng kết Chương 1 & 2

| Khái niệm Web đã biết | Tương đương/thay thế trong React Native |
|---|---|
| DOM, `<div>`, `<span>` | Native View Tree, `<View>`, `<Text>` |
| Webpack/Turbopack | Metro Bundler |
| CSS/Tailwind file | StyleSheet API / NativeWind |
| `flex-direction: row` mặc định | `flexDirection: 'column'` mặc định |
| `vh`, `vw`, media query | `useWindowDimensions`, breakpoint NativeWind |
| Không có khái niệm notch | Safe Area (`react-native-safe-area-context`) |
| `next dev` | `npx expo start` |
| Next.js App Router | Expo Router (file-based, cú pháp tương tự) |

**Chương tiếp theo (Chương 3 – Navigation)** sẽ đi sâu vào Expo Router và React Navigation — nơi tư duy routing của Next.js sẽ được ánh xạ sang, kèm ví dụ luồng điều hướng thực tế (Stack, Tab, Modal, truyền tham số, deep linking).
