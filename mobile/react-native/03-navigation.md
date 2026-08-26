# GIÁO TRÌNH REACT NATIVE
### Dành cho lập trình viên đã có nền tảng React / Next.js

---

# CHƯƠNG 3: NAVIGATION

> **Ngữ cảnh áp dụng:** Xuyên suốt chương này, mình dùng lại đúng bài toán bạn đã có ở phần Web — module **quản lý khóa học (Course Management)** với luồng: danh sách khóa học → xem chi tiết → thêm/sửa (dialog) → xóa (confirm). Mục tiêu là để bạn thấy rõ **route nào giữ nguyên tư duy Next.js, route nào phải đổi cách nghĩ**.

## 3.1. Vì sao Navigation trong Mobile khác Web

### 3.1.1. Khái niệm

Trên Web, "điều hướng" (navigation) về bản chất là **thay đổi URL** — trình duyệt tự quản lý lịch sử (history stack) qua nút Back/Forward. Trên Mobile, không có thanh địa chỉ, không có nút Back của hệ điều hành trên iOS (Android có nút Back vật lý/gesture) — vì vậy **bản thân ứng dụng phải tự quản lý ngăn xếp điều hướng (navigation stack)**.

### 3.1.2. Bảng đối chiếu khái niệm

| Next.js (App Router) | React Native (Expo Router) | Ghi chú |
|---|---|---|
| `app/admin/course/page.tsx` | `app/(admin)/course/index.tsx` | File-based routing — cấu trúc thư mục gần như giống hệt |
| `app/admin/course/[id]/page.tsx` | `app/(admin)/course/[id].tsx` | Dynamic route — cú pháp `[param]` giữ nguyên |
| `router.push('/admin/course/123')` | `router.push('/course/123')` | API gần như giống hệt (đều từ `expo-router` hoặc `next/navigation`) |
| `<Link href="...">` | `<Link href="...">` | Cùng tên component, hành vi tương tự |
| Layout lồng nhau (`layout.tsx`) | `_layout.tsx` | Cùng cơ chế nested layout |
| Modal (shadcn `<Dialog>` — overlay trong cùng 1 trang) | **Modal Route** (route riêng, presentation kiểu modal) hoặc Component `<Modal>` | Khác biệt lớn nhất — nói kỹ ở mục 3.5 |
| Không có khái niệm Tab Bar gốc | `Tab Navigator` — điều hướng chính của hầu hết app mobile | Đặc thù mobile |
| History stack của trình duyệt | `Stack Navigator` — ứng dụng tự quản lý, có animation chuyển màn hình | Đặc thù mobile |

> **Kết luận quan trọng:** Nếu bạn dùng **Expo Router**, tư duy file-based routing từ Next.js **tái sử dụng được đến 80%**. Phần còn lại (Tab, Modal, Stack animation) là kiến thức mới cần bổ sung — đây chính là nội dung chương này.

---

## 3.2. Expo Router — File-based Routing

### 3.2.1. Cấu trúc thư mục cho module Course Management

Ánh xạ từ cấu trúc Next.js `app/admin/course/` sang Expo Router:

```
app/
├── _layout.tsx                    # Root layout — tương đương app/layout.tsx
├── (admin)/                       # Route Group — không xuất hiện trên URL, dùng để nhóm layout
│   ├── _layout.tsx                # Layout riêng cho khu vực admin (vd: kiểm tra quyền)
│   ├── course/
│   │   ├── index.tsx              # Danh sách khóa học — tương đương page.tsx bạn gửi
│   │   ├── [id].tsx                # Chi tiết khóa học — tương đương app/admin/course/[id]/page.tsx
│   │   └── _layout.tsx             # (tuỳ chọn) layout con cho nhánh course
├── (modals)/
│   └── course-form.tsx            # Route dạng Modal — thay thế <CourseDialog> overlay
└── +not-found.tsx
```

**Giải thích `Route Group` `(admin)`:** Dấu ngoặc đơn quanh tên thư mục nghĩa là **nhóm route nhưng không xuất hiện trong URL thực tế**. Ví dụ file `app/(admin)/course/index.tsx` sẽ có route thực tế là `/course`, không phải `/admin/course`. Cơ chế này **giống hệt Route Group `(group)` của Next.js App Router** — dùng để áp layout/kiểm tra quyền riêng cho một nhóm màn hình mà không ảnh hưởng URL.

### 3.2.2. Root Layout — nơi khai báo Provider toàn cục

```tsx
// app/_layout.tsx
import '../src/global.css';
import { Stack } from 'expo-router';
import { SafeAreaProvider } from 'react-native-safe-area-context';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { Toaster } from 'sonner-native'; // bản RN của thư viện "sonner" bạn đang dùng

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 2,
      staleTime: 60 * 1000,
    },
  },
});

export default function RootLayout() {
  return (
    <SafeAreaProvider>
      <QueryClientProvider client={queryClient}>
        <Stack screenOptions={{ headerShown: false }} />
        <Toaster />
      </QueryClientProvider>
    </SafeAreaProvider>
  );
}
```

**Đối chiếu với thực tế dự án bạn:** Đây là phiên bản RN của `app/layout.tsx` — nơi bạn thường bọc `<QueryClientProvider>` trong dự án Next.js. Logic **giống hệt 100%**, chỉ khác thư viện toast (`sonner-native` thay `sonner`) vì `sonner` gốc dùng DOM.

### 3.2.3. Layout riêng cho khu vực Admin (kiểm tra quyền)

```tsx
// app/(admin)/_layout.tsx
import { Redirect, Stack } from 'expo-router';
import { useAuthStore } from '@/store/auth.store';

export default function AdminLayout() {
  const role = useAuthStore((state) => state.user?.role);

  // Tương đương middleware.ts hoặc kiểm tra role trong Server Component ở Next.js,
  // nhưng ở đây thực hiện phía client vì RN không có khái niệm Server Component/Middleware
  if (role !== 'admin') {
    return <Redirect href="/unauthorized" />;
  }

  return <Stack screenOptions={{ headerShown: false }} />;
}
```

> **Khác biệt cần lưu ý:** Next.js có thể chặn quyền truy cập **ở tầng server** (middleware) trước khi trả HTML về — an toàn tuyệt đối. React Native luôn chạy phía client, nên việc kiểm tra role như trên **chỉ có tác dụng về mặt UX** (ẩn màn hình), **không thay thế được việc backend phải tự kiểm tra quyền trên từng API** — nguyên tắc bảo mật "never trust the client" càng quan trọng hơn trong mobile.

---

## 3.3. Stack Navigator — Điều hướng Danh sách → Chi tiết

### 3.3.1. Khái niệm

`Stack Navigator` quản lý các màn hình theo mô hình **ngăn xếp (LIFO)** — mỗi lần `push` một màn hình mới sẽ chồng lên trên, có animation trượt, và nút Back sẽ `pop` màn hình hiện tại để quay lại màn trước. Đây là bản chất của **mọi route thông thường trong Expo Router** (không cần khai báo gì thêm — Stack là mặc định).

### 3.3.2. Ví dụ production: Danh sách khóa học → Chi tiết

Đây là bản RN của `page.tsx` (danh sách khóa học), giữ nguyên toàn bộ hook TanStack Query, chỉ thay UI:

```tsx
// app/(admin)/course/index.tsx
import { View, Text, Pressable } from 'react-native';
import { FlashList } from '@shopify/flash-list';
import { useRouter } from 'expo-router';
import { useState } from 'react';
import { PlusIcon } from 'lucide-react-native'; // bản RN của lucide-react, cùng bộ icon
import { useGetCourses } from '@/queries/useCourseQuery';
import { CourseSortBy, SortOrder } from '@/constants/sort';
import { CourseParams } from '@/schemas/course.schema';
import { CourseListItem } from '@/components/course/CourseListItem';

export default function AdminCourseScreen() {
  const router = useRouter();
  const [params, setParams] = useState<CourseParams>({
    page: 1,
    limit: 10,
    sortOrder: SortOrder.DESC,
    sortBy: CourseSortBy.COURSE_NAME,
  });

  // Hook TanStack Query TÁI SỬ DỤNG NGUYÊN VẸN từ dự án Web — xem chi tiết ở Chương 4
  const { data, isLoading, refetch, isRefetching } = useGetCourses(params);
  const courses = data?.data || [];

  return (
    <View className="flex-1 bg-gray-50">
      <View className="flex-row items-center justify-between px-4 pt-14 pb-4 bg-white border-b border-gray-100">
        <Text className="text-2xl font-black text-gray-900 tracking-tight">
          Danh sách khóa học
        </Text>
        <Pressable
          // Điều hướng đến route Modal — tương đương mở <CourseDialog> ở bản Web
          onPress={() => router.push('/course-form')}
          className="size-10 rounded-full bg-primary items-center justify-center active:opacity-80"
        >
          <PlusIcon color="white" size={20} />
        </Pressable>
      </View>

      <FlashList
        data={courses}
        estimatedItemSize={96}
        contentContainerStyle={{ padding: 16, gap: 12 }}
        onRefresh={refetch}
        refreshing={isRefetching}
        renderItem={({ item }) => (
          <CourseListItem
            course={item}
            // push kèm param động — tương đương router.push(`/admin/course/${id}`) bên Web
            onPress={() => router.push(`/course/${item.courseId}`)}
          />
        )}
        ListEmptyComponent={
          !isLoading ? (
            <Text className="text-center text-gray-400 mt-10">Chưa có khóa học nào</Text>
          ) : null
        }
      />
    </View>
  );
}
```

**Giải thích các điểm khác biệt quan trọng so với `page.tsx` gốc:**
- `onRefresh` + `refreshing`: RN có cơ chế **"kéo để tải lại" (pull-to-refresh)** gắn thẳng vào list — đây là pattern UX chuẩn của mobile, không tồn tại trên Web. Kết hợp trực tiếp với `refetch` của TanStack Query.
- Không còn `DataTable`/cột (`course-columns.tsx`): vì màn hình mobile hẹp, **không có khái niệm bảng dữ liệu dạng table** — toàn bộ thông tin được gộp vào 1 "card list item" duy nhất (thay thế cho cả `CourseCard` lẫn `columns` bên Web — chi tiết ở mục 4.6).
- `router.push('/course-form')` thay vì mở `<Dialog>`: đây là khác biệt cốt lõi, giải thích ở mục 3.5.

### 3.3.3. Màn hình chi tiết với Dynamic Route

```tsx
// app/(admin)/course/[id].tsx
import { View, Text, ScrollView } from 'react-native';
import { useLocalSearchParams, Stack } from 'expo-router';
import { useGetCourseById } from '@/queries/useCourseQuery';

export default function CourseDetailScreen() {
  // useLocalSearchParams tương đương params trong page.tsx của Next.js Dynamic Route
  const { id } = useLocalSearchParams<{ id: string }>();
  const { data, isLoading } = useGetCourseById(id);
  const course = data?.data;

  return (
    <>
      {/* Cấu hình header của riêng màn hình này — tương đương metadata title trong Next.js */}
      <Stack.Screen options={{ title: course?.courseName ?? 'Chi tiết khóa học' }} />

      <ScrollView className="flex-1 bg-white">
        {isLoading || !course ? (
          <Text className="p-4 text-gray-400">Đang tải...</Text>
        ) : (
          <View className="p-4">
            <Text className="text-2xl font-black text-gray-900">{course.courseName}</Text>
            <Text className="text-gray-500 mt-2">{course.description}</Text>
          </View>
        )}
      </ScrollView>
    </>
  );
}
```

**Giải thích:** `useLocalSearchParams` là hook đọc tham số route động (`[id]`), thay thế cho việc Next.js truyền `params` qua props của Server Component. `<Stack.Screen options={{...}} />` là cách khai báo tiêu đề/header động cho từng màn hình — tương đương `generateMetadata()` trong Next.js nhưng chạy phía client.

---

## 3.4. Tab Navigator

### 3.4.1. Khái niệm

Tab Navigator hiển thị **thanh điều hướng cố định** (thường ở đáy màn hình) cho các khu vực chính của ứng dụng — không có khái niệm tương đương trực tiếp bên Web (gần giống Sidebar cố định trong dashboard, nhưng luôn hiển thị và chiếm vị trí UI riêng biệt).

### 3.4.2. Ví dụ production: Tab cho khu vực Admin

```tsx
// app/(admin)/(tabs)/_layout.tsx
import { Tabs } from 'expo-router';
import { HomeIcon, BookOpenIcon, UsersIcon, SettingsIcon } from 'lucide-react-native';

export default function AdminTabsLayout() {
  return (
    <Tabs
      screenOptions={{
        headerShown: false,
        tabBarActiveTintColor: '#4F46E5', // trùng màu primary trong tailwind.config.js
        tabBarInactiveTintColor: '#9CA3AF',
        tabBarStyle: { height: 60, paddingBottom: 8 },
      }}
    >
      <Tabs.Screen
        name="index"
        options={{
          title: 'Tổng quan',
          tabBarIcon: ({ color, size }) => <HomeIcon color={color} size={size} />,
        }}
      />
      <Tabs.Screen
        name="course/index"
        options={{
          title: 'Khóa học',
          tabBarIcon: ({ color, size }) => <BookOpenIcon color={color} size={size} />,
        }}
      />
      <Tabs.Screen
        name="student/index"
        options={{
          title: 'Học viên',
          tabBarIcon: ({ color, size }) => <UsersIcon color={color} size={size} />,
        }}
      />
      <Tabs.Screen
        name="settings"
        options={{
          title: 'Cài đặt',
          tabBarIcon: ({ color, size }) => <SettingsIcon color={color} size={size} />,
        }}
      />
    </Tabs>
  );
}
```

**Lưu ý kiến trúc:** Mỗi tab (`course/index`, `student/index`...) **thực chất là một Stack Navigator con** — nghĩa là khi bạn ở tab "Khóa học" và bấm vào 1 khóa học để xem chi tiết, màn hình chi tiết sẽ **push chồng lên trong stack của tab đó**, tab bar vẫn hiển thị nguyên trạng thái đang chọn. Đây là mô hình **Nested Navigator** — không có khái niệm tương đương trực tiếp bên Web.

---

## 3.5. Modal — Thay thế cho `<Dialog>` (shadcn)

### 3.5.1. Khái niệm và khác biệt cốt lõi

Đây là điểm **khác biệt lớn nhất** giữa tư duy Web và Mobile mà bạn cần điều chỉnh. Trong `course-dialog.tsx` (Web), `<Dialog>` là một **overlay hiển thị đè lên trang hiện tại**, quản lý bằng state `open`/`onOpenChange` cục bộ trong `page.tsx`.

Trong React Native, có **hai cách tiếp cận** cho cùng nhu cầu:

| Cách tiếp cận | Khi nào dùng | Tương đương Web |
|---|---|---|
| **Modal Route** (route riêng, `presentation: 'modal'`) | Form phức tạp, nhiều trường, cần chiếm toàn màn hình, có thể có sub-navigation riêng | Gần giống mở trang riêng (nhưng có animation trượt từ dưới lên) |
| **`<Modal>` Component** (từ `react-native` hoặc thư viện Bottom Sheet) | Overlay đơn giản, nằm trong cùng 1 màn hình, không cần đổi URL | Giống hệt `<Dialog>` shadcn — overlay cục bộ |

> **Khuyến nghị production:** Với form phức tạp như `CourseDialog` (upload ảnh, nhiều field, validate Zod), nên dùng **Modal Route** — vừa tách bạch code, vừa tận dụng được animation & gesture vuốt-để-đóng có sẵn của hệ điều hành.

### 3.5.2. Cấu hình Modal Route

```tsx
// app/(admin)/_layout.tsx — khai báo route con dạng modal
import { Stack } from 'expo-router';

export default function AdminLayout() {
  return (
    <Stack screenOptions={{ headerShown: false }}>
      <Stack.Screen name="course/index" />
      <Stack.Screen name="course/[id]" />
      <Stack.Screen
        name="course-form"
        options={{
          presentation: 'modal',        // Hiệu ứng trượt từ dưới lên, giống Dialog/Sheet
          headerShown: true,
          title: 'Khóa học',
          gestureEnabled: true,          // Cho phép vuốt xuống để đóng (iOS)
        }}
      />
    </Stack>
  );
}
```

### 3.5.3. Ví dụ production: Chuyển `CourseDialog` thành Modal Route

```tsx
// app/(admin)/course-form.tsx
import { View, ScrollView } from 'react-native';
import { useRouter, useLocalSearchParams } from 'expo-router';
import { CourseForm } from '@/components/course/CourseForm';
import { useCreateCourse, useUpdateCourse, useGetCourseById } from '@/queries/useCourseQuery';
import { CourseInput } from '@/schemas/course.schema';
import { toast } from 'sonner-native';

export default function CourseFormModal() {
  const router = useRouter();
  // Truyền id qua query param khi edit: router.push(`/course-form?id=${course.courseId}`)
  const { id } = useLocalSearchParams<{ id?: string }>();
  const isEditing = !!id;

  const { data } = useGetCourseById(id ?? '');
  const createMutation = useCreateCourse();
  const updateMutation = useUpdateCourse();

  const handleSubmit = async (formData: CourseInput) => {
    try {
      if (isEditing && id) {
        await updateMutation.mutateAsync(
          { id, data: formData },
          {
            onSuccess: (response: any) => {
              toast.success(response?.message || 'Cập nhật khóa học thành công');
              router.back(); // Đóng modal — tương đương onOpenChange(false)
            },
          }
        );
      } else {
        await createMutation.mutateAsync(formData, {
          onSuccess: (response: any) => {
            toast.success(response?.message || 'Thêm khóa học thành công');
            router.back();
          },
        });
      }
    } catch (error) {
      console.error('Submit failed', error);
    }
  };

  return (
    <ScrollView className="flex-1 bg-white" contentContainerStyle={{ padding: 20 }}>
      <CourseForm
        initialData={isEditing ? data?.data : null}
        onSubmit={handleSubmit}
        loading={createMutation.isPending || updateMutation.isPending}
      />
    </ScrollView>
  );
}
```

**Giải thích các điểm chuyển đổi tư duy quan trọng:**
- `open`/`onOpenChange` (state cục bộ ở `page.tsx`) → **biến mất hoàn toàn**, thay bằng việc `router.push()`/`router.back()` chính là hành động mở/đóng.
- Truyền dữ liệu "đang sửa cái gì" không còn qua prop `initialData` truyền trực tiếp từ component cha, mà qua **query param `id` trên URL** (`?id=xxx`) rồi tự fetch lại bằng `useGetCourseById` — đúng triết lý "URL là nguồn sự thật" mà Next.js cũng khuyến khích.
- Toàn bộ phần `<form>` (field, validate, upload ảnh) được tách thành component `<CourseForm>` riêng để tái sử dụng — nội dung chi tiết của component này (bao gồm ánh xạ `react-hook-form` + `zodResolver`) trình bày ở **Chương 4, mục 4.6**.

### 3.5.4. Bottom Sheet — lựa chọn thay thế cho Modal đơn giản

Với các trường hợp đơn giản hơn (vd: `CourseFilter` — bộ lọc trình độ/sắp xếp), dùng **Bottom Sheet** thường trải nghiệm tốt hơn Modal toàn màn hình:

```bash
npx expo install @gorhom/bottom-sheet react-native-gesture-handler react-native-reanimated
```

```tsx
// src/components/course/CourseFilterSheet.tsx
import { forwardRef, useMemo } from 'react';
import { View, Text } from 'react-native';
import BottomSheet, { BottomSheetView, BottomSheetBackdrop } from '@gorhom/bottom-sheet';

interface CourseFilterSheetProps {
  onApply: (filters: { level?: string; sortBy?: string }) => void;
}

export const CourseFilterSheet = forwardRef<BottomSheet, CourseFilterSheetProps>(
  ({ onApply }, ref) => {
    const snapPoints = useMemo(() => ['50%'], []);

    return (
      <BottomSheet
        ref={ref}
        index={-1} // Ẩn mặc định — tương đương open=false
        snapPoints={snapPoints}
        enablePanDownToClose
        backdropComponent={(props) => (
          <BottomSheetBackdrop {...props} disappearsOnIndex={-1} appearsOnIndex={0} />
        )}
      >
        <BottomSheetView className="p-5">
          <Text className="text-lg font-bold mb-4">Bộ lọc khóa học</Text>
          {/* Nội dung filter — chi tiết Select/Picker ở Chương 4 */}
        </BottomSheetView>
      </BottomSheet>
    );
  }
);
```

**Cách mở/đóng từ component cha** (giống việc gọi `sheetRef.current?.expand()`):

```tsx
import { useRef } from 'react';
import BottomSheet from '@gorhom/bottom-sheet';
import { Pressable, Text } from 'react-native';
import { CourseFilterSheet } from '@/components/course/CourseFilterSheet';

export default function CourseScreenWithFilter() {
  const filterSheetRef = useRef<BottomSheet>(null);

  return (
    <>
      <Pressable onPress={() => filterSheetRef.current?.expand()}>
        <Text>Bộ lọc</Text>
      </Pressable>
      <CourseFilterSheet ref={filterSheetRef} onApply={(filters) => {/* ... */}} />
    </>
  );
}
```

---

## 3.6. Confirm Dialog — Thay thế `ConfirmDialog`

Với hộp thoại xác nhận đơn giản (như `handleDeleteTrigger` trong `page.tsx` gốc), RN có API dựng sẵn `Alert`, không cần tự dựng component:

```tsx
// src/hooks/useConfirmDelete.ts
import { Alert } from 'react-native';

export function confirmDelete(courseName: string, onConfirm: () => void) {
  Alert.alert(
    'Xác nhận xóa khóa học',
    `Bạn có chắc chắn muốn xóa "${courseName}"? Hành động này không thể hoàn tác.`,
    [
      { text: 'Hủy bỏ', style: 'cancel' },
      { text: 'Xóa', style: 'destructive', onPress: onConfirm },
    ]
  );
}
```

```tsx
// Sử dụng trong CourseListItem
import { confirmDelete } from '@/hooks/useConfirmDelete';
import { useDeleteCourse } from '@/queries/useCourseQuery';

const deleteMutation = useDeleteCourse();

const handleDelete = (course: Course) => {
  confirmDelete(course.courseName, () => {
    deleteMutation.mutate(course.courseId!);
  });
};
```

**So sánh:** `Alert.alert` render **native alert của hệ điều hành** (khác giao diện iOS/Android), không cần tự style như `<ConfirmDialog>` shadcn — đánh đổi giữa tính nhất quán thương hiệu (branding) và trải nghiệm quen thuộc với người dùng hệ điều hành. Nếu cần giao diện tùy biến theo thương hiệu, dùng `<Modal>` tự dựng thay vì `Alert.alert`.

---

## 3.7. Deep Linking

### 3.7.1. Khái niệm

Deep Linking cho phép mở thẳng một màn hình cụ thể trong app từ bên ngoài (link trong email, thông báo đẩy, quét QR) — tương đương việc một URL Web có thể mở thẳng `/admin/course/123`.

### 3.7.2. Cấu hình

```ts
// app.config.ts
export default {
  expo: {
    scheme: 'eduapp', // Custom scheme: eduapp://
    ios: { associatedDomains: ['applinks:app.example.com'] },
    android: {
      intentFilters: [
        {
          action: 'VIEW',
          data: [{ scheme: 'https', host: 'app.example.com' }],
          category: ['BROWSABLE', 'DEFAULT'],
        },
      ],
    },
  },
};
```

**Kết quả:** Với Expo Router, cấu trúc thư mục `app/(admin)/course/[id].tsx` **tự động** ánh xạ thành cả deep link `eduapp://course/123` lẫn Universal Link `https://app.example.com/course/123` — không cần khai báo route thủ công như React Navigation truyền thống (`Linking config` object).

---

## Tổng kết Chương 3

| Nhu cầu điều hướng | Web (Next.js) | React Native (Expo Router) |
|---|---|---|
| Danh sách → Chi tiết | `router.push('/course/123')` | `router.push('/course/123')` — giống hệt |
| Layout dùng chung | `layout.tsx` | `_layout.tsx` |
| Nhóm route không lộ URL | `(group)` | `(group)` — giống hệt |
| Form overlay (Dialog) | `<Dialog open={...}>` state cục bộ | Modal Route (`presentation: 'modal'`) hoặc `<Modal>`/Bottom Sheet |
| Điều hướng chính của app | Sidebar/Navbar tự dựng | `Tab Navigator` dựng sẵn |
| Xác nhận hành động | `<ConfirmDialog>` tự dựng | `Alert.alert()` dựng sẵn (native) |
| Mở link ngoài vào app | Không áp dụng (đã là URL) | Deep Linking (`scheme` + Universal Links) |

**Chương tiếp theo (Chương 4)** đi sâu vào phần bạn sẽ thấy quen thuộc nhất: State Management & Data Fetching — nơi `course.service.ts`, `course.schema.ts`, `useCourseQuery.ts` gần như **copy nguyên** sang RN, và cách chuyển đổi toàn bộ UI còn lại (`CourseCard`, `CourseFilter`, `CourseDialog`, `DataTable`) sang thành phần tương ứng của mobile.
