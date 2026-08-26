# GIÁO TRÌNH REACT NATIVE
### Dành cho lập trình viên đã có nền tảng React / Next.js

---

# CHƯƠNG 5: FORMS NÂNG CAO & DESIGN SYSTEM

## 5.1. React Hook Form trong React Native — Tổng kết chuyên sâu

### 5.1.1. Nguyên tắc cốt lõi: `Controller` là bắt buộc, không phải tùy chọn

Ở Chương 4 bạn đã thấy `CourseForm` dùng `Controller` thay cho `register()`. Đây không phải lựa chọn phong cách, mà là **yêu cầu kỹ thuật bắt buộc** với hầu hết input RN.

**Lý do kỹ thuật:** `register()` của react-hook-form hoạt động bằng cách gắn `ref` trực tiếp vào DOM node (`<input>`), rồi tự đọc `.value` qua `ref` khi cần — cơ chế **uncontrolled**. Các component RN như `TextInput`, `Switch`, `Slider` không expose interface DOM-like này qua `ref` (ref của chúng trỏ đến native view instance, không có `.value`). Vì vậy react-hook-form không thể "tự đọc" giá trị — bắt buộc phải qua `Controller`, vốn hoạt động theo mô hình **controlled** (giống `value`/`onChange` bạn đã quen với input controlled trên Web).

| Đặc điểm | `register()` (Web) | `Controller` (RN bắt buộc) |
|---|---|---|
| Mô hình | Uncontrolled (ref-based) | Controlled (value/onChange) |
| Re-render khi gõ | Không (react-hook-form tối ưu, không re-render form cha) | Có (mỗi field tự re-render cục bộ) |
| Hiệu năng với form lớn | Rất tốt | Vẫn tốt nếu tách field thành component riêng (xem 5.1.3) |

### 5.1.2. Custom Hook chuẩn hoá `Controller` — giảm boilerplate

Vì `Controller` lặp lại cấu trúc ở mọi field (thấy rõ trong `CourseForm` Chương 4), nên đóng gói thành component input tái sử dụng ngay từ đầu dự án — đây chính là nền tảng của "Design System" sẽ xây ở phần 5.2.

```tsx
// src/components/ui/FormTextInput.tsx
import { TextInput, TextInputProps, View, Text } from 'react-native';
import { Control, Controller, FieldValues, Path } from 'react-hook-form';

interface FormTextInputProps<T extends FieldValues> extends TextInputProps {
  control: Control<T>;
  name: Path<T>;
  label: string;
  errorMessage?: string;
}

export function FormTextInput<T extends FieldValues>({
  control,
  name,
  label,
  errorMessage,
  ...inputProps
}: FormTextInputProps<T>) {
  return (
    <View className="gap-2">
      <Text className="text-xs font-black uppercase text-gray-400">{label}</Text>
      <Controller
        control={control}
        name={name}
        render={({ field: { onChange, onBlur, value } }) => (
          <TextInput
            value={value ?? ''}
            onChangeText={onChange}
            onBlur={onBlur}
            className={`h-14 rounded-2xl border px-4 text-base ${
              errorMessage ? 'border-red-400' : 'border-gray-200'
            }`}
            {...inputProps}
          />
        )}
      />
      {errorMessage && <Text className="text-xs text-red-500 font-bold">{errorMessage}</Text>}
    </View>
  );
}
```

**Áp dụng lại `CourseForm` với component chuẩn hoá:**

```tsx
// src/components/course/CourseForm.tsx (rút gọn nhờ FormTextInput)
<FormTextInput
  control={control}
  name="courseName"
  label="Tên khóa học"
  placeholder="VD: English Foundation A1"
  errorMessage={errors.courseName?.message}
/>

<FormTextInput
  control={control}
  name="tuitionFee"
  label="Học phí (VNĐ)"
  placeholder="VD: 2500000"
  keyboardType="numeric"
  errorMessage={errors.tuitionFee?.message}
/>
```

**So sánh:** Đây là tư duy y hệt việc bạn từng bọc `<Input>` shadcn + `<Label>` + hiển thị lỗi thành 1 field component tái sử dụng trong dự án Next.js — chỉ khác nền tảng render bên dưới.

### 5.1.3. Tối ưu hiệu năng: `React.memo` cho field phức tạp

Vì mỗi `Controller` re-render cục bộ, form có nhiều field animation/heavy component (vd: rich text, map picker) nên tách riêng và `memo` hoá:

```tsx
export const FormTextInput = React.memo(function FormTextInput<T extends FieldValues>(
  props: FormTextInputProps<T>
) {
  // ... (nội dung như trên)
}) as typeof FormTextInputInner; // ép kiểu generic khi dùng memo — pattern quen thuộc với TS
```

### 5.1.4. Xử lý bàn phím: `KeyboardAvoidingView`

**Khái niệm mới hoàn toàn với người quen Web:** Bàn phím ảo trên mobile **chiếm nửa dưới màn hình** và có thể che mất input đang gõ hoặc nút submit — vấn đề không tồn tại trên Web (bàn phím vật lý không chiếm không gian màn hình).

```tsx
// src/components/course/CourseFormScreen.tsx
import { KeyboardAvoidingView, Platform, ScrollView } from 'react-native';

export function CourseFormScreen({ children }: { children: React.ReactNode }) {
  return (
    <KeyboardAvoidingView
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'} // iOS và Android xử lý khác nhau
      className="flex-1"
      keyboardVerticalOffset={Platform.OS === 'ios' ? 90 : 0} // Bù trừ chiều cao header
    >
      <ScrollView
        contentContainerStyle={{ padding: 20 }}
        keyboardShouldPersistTaps="handled" // Cho phép bấm nút dù bàn phím đang mở
      >
        {children}
      </ScrollView>
    </KeyboardAvoidingView>
  );
}
```

**Giải thích prop quan trọng:**
- `behavior="padding"` (iOS) vs `"height"` (Android): hai nền tảng xử lý việc "đẩy nội dung lên khi bàn phím xuất hiện" khác nhau ở tầng native, buộc phải rẽ nhánh theo `Platform.OS`.
- `keyboardShouldPersistTaps="handled"`: mặc định `ScrollView` sẽ tự đóng bàn phím khi bạn chạm ra ngoài input — nhưng nếu chạm đó là chạm vào 1 nút bấm (vd: nút "Lưu"), giá trị `"handled"` đảm bảo nút vẫn nhận sự kiện `onPress` thay vì bị nuốt bởi hành vi đóng bàn phím.

> **Giải pháp thay thế production:** Với form dài, thư viện `react-native-keyboard-controller` xử lý mượt và ít bug hơn `KeyboardAvoidingView` gốc, đặc biệt khi kết hợp Bottom Sheet — khuyến nghị dùng cho ứng dụng thương mại.

---

## 5.2. Xây dựng Design System — Thay thế vai trò shadcn/ui

### 5.2.1. Vì sao không dùng lại được shadcn/ui

shadcn/ui hoạt động bằng cách generate ra source code component dựa trên **Radix UI Primitives** — thư viện xây trên nền DOM (dùng `aria-*`, focus trap, portal...). Không có phiên bản RN nào của Radix, vì các khái niệm này gắn chặt với trình duyệt.

**Nguyên tắc thay thế:** Bạn không "port" shadcn sang RN, mà **xây một bộ component tối giản riêng** theo đúng token thiết kế (màu, spacing, typography, radius) mà shadcn/Tailwind config cũ của bạn đang dùng — để giữ tính nhất quán thương hiệu giữa Web và Mobile.

### 5.2.2. Thiết lập Design Token dùng chung

```js
// tailwind.config.js — đồng bộ token với dự án Web (copy từ shadcn theme của bạn)
module.exports = {
  presets: [require('nativewind/preset')],
  theme: {
    extend: {
      colors: {
        primary: { DEFAULT: '#4F46E5', foreground: '#ffffff' },
        destructive: { DEFAULT: '#EF4444', foreground: '#ffffff' },
        muted: { DEFAULT: '#F3F4F6', foreground: '#6B7280' },
        border: '#E5E7EB',
      },
      borderRadius: {
        DEFAULT: '12px',
        xl: '20px',
        '2xl': '24px',
      },
    },
  },
};
```

### 5.2.3. Component `<Button>` — tương đương `button.tsx` của shadcn

```tsx
// src/components/ui/Button.tsx
import { Pressable, Text, ActivityIndicator, PressableProps } from 'react-native';
import { cva, type VariantProps } from 'class-variance-authority'; // Dùng lại được CVA — thuần logic, không phụ thuộc DOM
import { cn } from '@/utils/cn';

const buttonVariants = cva('items-center justify-center rounded-2xl flex-row gap-2', {
  variants: {
    variant: {
      default: 'bg-primary active:opacity-80',
      destructive: 'bg-destructive active:opacity-80',
      outline: 'border border-border bg-white active:bg-gray-50',
      ghost: 'active:bg-gray-100',
    },
    size: {
      default: 'h-12 px-6',
      sm: 'h-10 px-4',
      lg: 'h-14 px-8',
      icon: 'size-10',
    },
  },
  defaultVariants: { variant: 'default', size: 'default' },
});

const textVariants = cva('font-bold text-base', {
  variants: {
    variant: {
      default: 'text-white',
      destructive: 'text-white',
      outline: 'text-gray-800',
      ghost: 'text-gray-800',
    },
  },
  defaultVariants: { variant: 'default' },
});

interface ButtonProps extends PressableProps, VariantProps<typeof buttonVariants> {
  children: React.ReactNode;
  loading?: boolean;
  disabled?: boolean;
}

export function Button({ variant, size, children, loading, disabled, className, ...props }: ButtonProps) {
  return (
    <Pressable
      className={cn(buttonVariants({ variant, size }), disabled && 'opacity-50', className as string)}
      disabled={disabled || loading}
      {...props}
    >
      {loading ? (
        <ActivityIndicator color={variant === 'outline' || variant === 'ghost' ? '#111827' : 'white'} />
      ) : typeof children === 'string' ? (
        <Text className={textVariants({ variant })}>{children}</Text>
      ) : (
        children
      )}
    </Pressable>
  );
}
```

```bash
npm install class-variance-authority clsx tailwind-merge
```

```ts
// src/utils/cn.ts — GIỐNG HỆT bản Web
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

**Điểm mấu chốt cần nhấn mạnh:** `class-variance-authority` (CVA) — công cụ định nghĩa variant mà shadcn/ui dùng bên dưới — là **thư viện thuần JS, không phụ thuộc DOM**, nên tái sử dụng được 100% API. Đây là lý do bạn có thể giữ nguyên tư duy "variant-based component" đã quen, chỉ đổi phần render bên trong từ `<button>` sang `<Pressable>`.

### 5.2.4. Component `<Select>` — tương đương `select.tsx` của shadcn

Đây là component phức tạp nhất cần xây lại, vì không có `<select>` gốc trên RN. Cách tiếp cận chuẩn: dùng `Modal` hiển thị danh sách lựa chọn dạng Action Sheet.

```tsx
// src/components/ui/Select.tsx
import { Modal, Pressable, Text, View, FlatList } from 'react-native';
import { useState } from 'react';
import { ChevronDownIcon, CheckIcon } from 'lucide-react-native';

interface SelectOption {
  label: string;
  value: string;
}

interface SelectProps {
  value?: string;
  onValueChange: (value: string) => void;
  options: SelectOption[];
  placeholder?: string;
}

export function Select({ value, onValueChange, options, placeholder = 'Chọn...' }: SelectProps) {
  const [open, setOpen] = useState(false);
  const selectedLabel = options.find((opt) => opt.value === value)?.label;

  return (
    <>
      <Pressable
        onPress={() => setOpen(true)}
        className="h-14 rounded-2xl border border-gray-200 px-4 flex-row items-center justify-between bg-white"
      >
        <Text className={selectedLabel ? 'text-gray-900 font-medium' : 'text-gray-400'}>
          {selectedLabel || placeholder}
        </Text>
        <ChevronDownIcon size={18} color="#9ca3af" />
      </Pressable>

      <Modal visible={open} transparent animationType="slide" onRequestClose={() => setOpen(false)}>
        <Pressable className="flex-1 bg-black/40 justify-end" onPress={() => setOpen(false)}>
          <View className="bg-white rounded-t-3xl max-h-[70%] pt-2 pb-8">
            <View className="w-10 h-1 rounded-full bg-gray-200 self-center mb-4" />
            <FlatList
              data={options}
              keyExtractor={(item) => item.value}
              renderItem={({ item }) => (
                <Pressable
                  onPress={() => {
                    onValueChange(item.value);
                    setOpen(false);
                  }}
                  className="flex-row items-center justify-between px-6 py-4 active:bg-gray-50"
                >
                  <Text className="text-base text-gray-800">{item.label}</Text>
                  {item.value === value && <CheckIcon size={18} color="#4F46E5" />}
                </Pressable>
              )}
            />
          </View>
        </Pressable>
      </Modal>
    </>
  );
}
```

**Áp dụng thay thế `<Select>` trong `CourseFilterSheet` (Chương 4) nếu số lượng option nhiều** (khác với Level chỉ vài giá trị dùng Chip phù hợp hơn — dùng `<Select>` này khi option nhiều, vd: danh sách giảng viên, danh sách lớp):

```tsx
<Select
  value={teacherId}
  onValueChange={setTeacherId}
  options={teachers.map((t) => ({ label: t.name, value: t.id }))}
  placeholder="Chọn giảng viên"
/>
```

**So sánh với `<Select>` shadcn:** Về API sử dụng (`value`, `onValueChange`, `options`) **giữ nguyên tư duy hoàn toàn** — đây là điểm quan trọng: khi thiết kế Design System RN, nên **cố ý giữ props interface giống hệt bản Web** để giảm chi phí chuyển đổi tư duy khi làm việc song song cả hai nền tảng.

### 5.2.5. Component `<Badge>` — tương đương `badge.tsx` (đã dùng ở `BadgeLevel`)

```tsx
// src/components/ui/Badge.tsx
import { View, Text } from 'react-native';
import { cva, type VariantProps } from 'class-variance-authority';

const badgeVariants = cva('px-3 py-1 rounded-full self-start', {
  variants: {
    variant: {
      default: 'bg-primary/10',
      secondary: 'bg-gray-100',
      destructive: 'bg-red-50',
      success: 'bg-green-50',
    },
  },
  defaultVariants: { variant: 'default' },
});

const textColor = {
  default: 'text-primary',
  secondary: 'text-gray-600',
  destructive: 'text-red-600',
  success: 'text-green-600',
};

interface BadgeProps extends VariantProps<typeof badgeVariants> {
  children: string;
}

export function Badge({ variant = 'default', children }: BadgeProps) {
  return (
    <View className={badgeVariants({ variant })}>
      <Text className={`text-xs font-bold ${textColor[variant ?? 'default']}`}>{children}</Text>
    </View>
  );
}
```

### 5.2.6. Bảng đối chiếu tổng quát: shadcn/ui → RN Design System

| shadcn/ui Component | Thay thế RN | Thư viện nền |
|---|---|---|
| `<Button>` | Tự dựng bằng `<Pressable>` + CVA | `class-variance-authority` (dùng lại nguyên) |
| `<Dialog>` | Modal Route hoặc `<Modal>` | Xem Chương 3, mục 3.5 |
| `<Select>` | Tự dựng Action Sheet | `<Modal>` gốc, hoặc `@gorhom/bottom-sheet` |
| `<DropdownMenu>` | `zeego` (Context Menu native) | Đã dùng ở Chương 4, mục 4.6.1 |
| `<Toast>`/`sonner` | `sonner-native` | Port chính thức của sonner cho RN |
| `<Tabs>` | `expo-router` Tab hoặc tự dựng segmented control | Xem Chương 3, mục 3.4 |
| `<Table>` (`@tanstack/react-table`) | Không có tương đương — dùng `FlatList`/`FlashList` | Xem Chương 4, mục 4.6.1 |
| `<Skeleton>` | Tự dựng bằng `Animated`/`react-native-reanimated` shimmer | Xem Chương 7 |
| `<Switch>` | `<Switch>` gốc của `react-native` | API gần giống hệt |
| `<Checkbox>`, `<RadioGroup>` | Tự dựng bằng `<Pressable>` + icon | Tương tự cách dựng `<Badge>` |

---

## 5.3. Icon System

### 5.3.1. `lucide-react-native` — tái sử dụng bộ icon quen thuộc

```bash
npx expo install lucide-react-native react-native-svg
```

```tsx
import { BookOpenIcon } from 'lucide-react-native';

<BookOpenIcon size={24} color="#4F46E5" strokeWidth={2} />
```

**So sánh:** Cùng tên import (`BookOpenIcon`), cùng bộ icon (1500+ icon) với `lucide-react` bạn đang dùng — chỉ khác cách truyền màu (`color` prop thay `className="text-primary"`, vì SVG trong RN không đọc được biến CSS `currentColor` qua className theo cách Web làm).

### 5.3.2. Tối ưu: tạo Icon Wrapper đọc theme

```tsx
// src/components/ui/Icon.tsx
import { LucideIcon } from 'lucide-react-native';

interface IconProps {
  icon: LucideIcon;
  size?: number;
  variant?: 'default' | 'muted' | 'primary' | 'destructive';
}

const colorMap = {
  default: '#111827',
  muted: '#9CA3AF',
  primary: '#4F46E5',
  destructive: '#EF4444',
};

export function Icon({ icon: IconComponent, size = 20, variant = 'default' }: IconProps) {
  return <IconComponent size={size} color={colorMap[variant]} />;
}
```

```tsx
// Sử dụng nhất quán trong toàn app
import { ClockIcon } from 'lucide-react-native';
<Icon icon={ClockIcon} variant="muted" size={14} />
```

---

## Tổng kết Chương 5

| Nhu cầu | Web (shadcn + RHF) | React Native |
|---|---|---|
| Điều khiển giá trị input | `register()` | `Controller` (bắt buộc) |
| Variant component | CVA | CVA — dùng lại nguyên |
| `<Select>`, `<Dialog>` | Radix Primitive | Tự dựng bằng `Modal`/Bottom Sheet |
| Icon | `lucide-react` | `lucide-react-native` — API gần giống hệt |
| Bàn phím che input | Không có vấn đề này | `KeyboardAvoidingView` bắt buộc |

**Chương tiếp theo (Chương 6)** sẽ đi vào phần hoàn toàn mới với người làm Web: Native Capabilities — quyền truy cập thiết bị, lưu trữ cục bộ, camera, push notification.
