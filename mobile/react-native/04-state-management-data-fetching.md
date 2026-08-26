# GIÁO TRÌNH REACT NATIVE
### Dành cho lập trình viên đã có nền tảng React / Next.js

---

# CHƯƠNG 4: STATE MANAGEMENT & DATA FETCHING

> **Ngữ cảnh áp dụng:** Chương này ánh xạ trực tiếp toàn bộ module **Course Management** bạn đã gửi (`course.service.ts`, `course.schema.ts`, `useCourseQuery.ts`, `course-card.tsx`, `course-columns.tsx`, `course-dialog.tsx`, `course-filter.tsx`, `page.tsx`) sang React Native. Đây là chương quan trọng nhất với bạn, vì phần lớn kiến thức Zustand/RTK/TanStack Query/Axios/Zod đã có **tái sử dụng gần như tuyệt đối** — điều cần học chỉ là những chỗ môi trường mobile buộc phải khác.

## 4.1. Nguyên tắc tổng quát: Lớp nào tái sử dụng, lớp nào viết lại

Trước khi đi vào chi tiết, cần nắm rõ **ranh giới kiến trúc** — đây là điều giúp bạn tiết kiệm 70% thời gian khi porting ứng dụng Web sang RN:

```
┌─────────────────────────────────────┐
│  UI Layer (Component)                │  ← Viết lại gần như 100%
│  course-card.tsx, course-dialog.tsx  │     (View/Text thay div/span)
├─────────────────────────────────────┤
│  Hook Layer (TanStack Query hooks)   │  ← Tái sử dụng gần như 100%
│  useCourseQuery.ts                   │     (chỉ đổi cách persist cache)
├─────────────────────────────────────┤
│  Service Layer (Axios calls)         │  ← Tái sử dụng ~95%
│  course.service.ts                   │     (chỉ đổi nơi lưu token)
├─────────────────────────────────────┤
│  Schema Layer (Zod)                  │  ← Tái sử dụng 100%
│  course.schema.ts                    │     (thuần logic, không đụng UI/DOM)
└─────────────────────────────────────┘
```

> **Nguyên tắc thiết kế production quan trọng nhất khi porting Web → Mobile:** Nếu dự án Web của bạn đã tách rõ 4 lớp như trên (điều mà bộ code bạn gửi đã làm rất tốt), việc chuyển sang React Native **chỉ động đến lớp UI**. Đây chính là lý do kiến trúc "service/schema/hook tách khỏi component" không chỉ tốt cho khả năng bảo trì trên Web, mà còn là **điều kiện tiên quyết** để chia sẻ code giữa Web và Mobile (kể cả trong monorepo).

---

## 4.2. Schema Layer: Zod — Tái sử dụng 100%

### 4.2.1. Khái niệm

Zod là thư viện validate thuần JavaScript/TypeScript, hoàn toàn không phụ thuộc DOM hay bất kỳ API nào của trình duyệt. Vì vậy, **toàn bộ file `course.schema.ts` bạn gửi có thể copy nguyên xi sang React Native mà không cần sửa một dòng nào.**

```ts
// src/schemas/course.schema.ts — GIỐNG HỆT bản Web, copy nguyên
import { CourseSortBy, SortOrder } from "@/constants/sort";
import { CourseLevel } from "@/constants/type";
import z from "zod";

export const courseSchema = z.object({
  courseId: z.string().uuid(),
  courseName: z.string().min(1, "Tên khóa học là bắt buộc").max(255),
  tuitionFee: z.number(),
  level: z.nativeEnum(CourseLevel).optional().nullable(),
  totalSessions: z.number().int().min(1, "Số buổi học phải ít nhất là 1"),
  image: z.string().url().optional().nullable(),
  description: z.string().optional().nullable(),
  totalClasses: z.number().optional(),
  estimatedRevenue: z.number().optional(),
});

export type Course = z.infer<typeof courseSchema>;

export const courseInputSchema = z.object({
  courseName: z.string().min(1, "Tên khóa học là bắt buộc").max(255),
  tuitionFee: z.string().regex(/^\d+(\.\d{1,2})?$/, "Học phí phải là số hợp lệ"),
  level: z.string().max(20).optional().nullable(),
  totalSessions: z.coerce.number().int().min(1, "Số buổi học phải ít nhất là 1"),
  image: z.string().optional().nullable(),
  description: z.string().optional().nullable(),
});

export type CourseInput = z.input<typeof courseInputSchema>;
```

**Điểm mấu chốt cần nhấn mạnh:** Đây là minh chứng rõ nhất cho nguyên tắc ở mục 4.1 — **schema/type layer là tài sản dùng chung tuyệt đối** giữa Web và Mobile. Trong các dự án lớn dùng monorepo (Turborepo/Nx), lớp này thường được tách thành package riêng (`@repo/schemas`) để cả app Next.js lẫn app Expo cùng import.

---

## 4.3. Service Layer: Axios — Tái sử dụng ~95%

### 4.3.1. Phần tái sử dụng nguyên vẹn

```ts
// src/services/course.service.ts — GIỐNG HỆT bản Web, copy nguyên
import { ApiResponse } from "@/constants/apiResponse";
import { Course, CourseInput, CourseParams } from "@/schemas/course.schema";
import http from "@/utils/http";

const prefix = "/courses";

const courseService = {
  getCourses: (params?: CourseParams) => {
    return http.get<ApiResponse<Course[]>>(prefix, { params });
  },
  createCourse: (data: CourseInput) => {
    return http.post<ApiResponse<Course>>(prefix, data);
  },
  updateCourse: (id: string, data: CourseInput) => {
    return http.put<ApiResponse<Course>>(`${prefix}/${id}`, data);
  },
  deleteCourse: (id: string) => {
    return http.delete<ApiResponse<any>>(`${prefix}/${id}`);
  },
  getCourseById: (id: string) => {
    return http.get<ApiResponse<Course>>(`${prefix}/${id}`);
  },
};

export default courseService;
```

**Toàn bộ logic gọi API giữ nguyên 100%** — vì Axios hoạt động độc lập với môi trường (hoạt động trên cả Node.js, trình duyệt, và React Native Hermes engine).

### 4.3.2. Phần BẮT BUỘC phải sửa: nơi lưu token (`utils/http.ts`)

Đây là **5% khác biệt duy nhất** của lớp Service — vì `localStorage`/`cookie` không tồn tại trên mobile.

| Web (`http.ts` gốc) | React Native | Lý do |
|---|---|---|
| `localStorage.getItem('accessToken')` | `SecureStore.getItemAsync('accessToken')` | Không có `localStorage`; `SecureStore` mã hoá dữ liệu bằng Keychain (iOS) / Keystore (Android) — an toàn hơn cho token |
| Cookie `httpOnly` (nếu dùng) | Không dùng được — RN không có cơ chế cookie tự động như trình duyệt | Phải tự đính token vào header thủ công |

```ts
// src/utils/http.ts — bản React Native
import axios from 'axios';
import * as SecureStore from 'expo-secure-store';

const http = axios.create({
  baseURL: process.env.EXPO_PUBLIC_API_URL,
  timeout: 10000,
});

// Interceptor gắn token — tương tự bản Web nhưng đổi nguồn lưu trữ
http.interceptors.request.use(async (config) => {
  const token = await SecureStore.getItemAsync('accessToken');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Interceptor xử lý refresh token khi 401 — LOGIC GIỐNG HỆT bản Web
http.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      const refreshToken = await SecureStore.getItemAsync('refreshToken');
      try {
        const { data } = await axios.post(`${process.env.EXPO_PUBLIC_API_URL}/auth/refresh`, {
          refreshToken,
        });
        await SecureStore.setItemAsync('accessToken', data.accessToken);
        originalRequest.headers.Authorization = `Bearer ${data.accessToken}`;
        return http(originalRequest);
      } catch (refreshError) {
        await SecureStore.deleteItemAsync('accessToken');
        await SecureStore.deleteItemAsync('refreshToken');
        // Điều hướng về login — dùng router thay vì window.location
        // router.replace('/login') gọi từ nơi bắt lỗi toàn cục (xem 4.4.3)
      }
    }
    return Promise.reject(error);
  }
);

export default http;
```

```bash
npx expo install expo-secure-store
```

> **Lưu ý bảo mật production:** `EXPO_PUBLIC_API_URL` — tiền tố `EXPO_PUBLIC_` là **bắt buộc** để biến môi trường được nhúng vào bundle client (tương đương `NEXT_PUBLIC_` bên Next.js). Biến không có tiền tố này sẽ **không** truy cập được từ code chạy trên máy người dùng — cơ chế bảo vệ secret khỏi bị lộ trong bundle.

---

## 4.4. Hook Layer: TanStack Query — Tái sử dụng gần như 100%

### 4.4.1. Phần tái sử dụng nguyên vẹn

```ts
// src/queries/useCourseQuery.ts — GIỐNG HỆT bản Web, copy nguyên
import { CourseInput, CourseParams } from "@/schemas/course.schema";
import courseService from "@/services/course.service";
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";

export const COURSE_QUERY_KEY = ["courses"];

export const useGetCourses = (params?: CourseParams) => {
  return useQuery({
    queryKey: [...COURSE_QUERY_KEY, params],
    queryFn: () => courseService.getCourses(params).then((res) => res.data),
  });
};

export const useCreateCourse = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (data: CourseInput) => courseService.createCourse(data),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: COURSE_QUERY_KEY }),
  });
};

export const useUpdateCourse = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: CourseInput }) =>
      courseService.updateCourse(id, data),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: COURSE_QUERY_KEY }),
  });
};

export const useDeleteCourse = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (id: string) => courseService.deleteCourse(id),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: COURSE_QUERY_KEY }),
  });
};

export const useGetCourseById = (id: string) => {
  return useQuery({
    queryKey: [...COURSE_QUERY_KEY, id],
    queryFn: () => courseService.getCourseById(id).then((res) => res.data),
    enabled: !!id,
  });
};
```

**Không có gì thay đổi** — `useQuery`, `useMutation`, `invalidateQueries`, `queryKey` hoạt động độc lập với môi trường render. Đây là lý do TanStack Query được xem là lựa chọn hàng đầu cho ứng dụng cross-platform.

### 4.4.2. Điểm khác biệt hành vi cần cấu hình lại: Refetch khi quay lại App

**Khái niệm khác biệt:** Trên Web, TanStack Query mặc định tự động refetch khi **cửa sổ trình duyệt lấy lại focus** (`window focus`) — vd: người dùng chuyển tab rồi quay lại. Trên Mobile, không có khái niệm "tab" hay "window focus" — thay vào đó là **AppState** (ứng dụng chuyển từ background → foreground, vd: người dùng mở app khác rồi quay lại).

```tsx
// src/lib/queryClient.ts
import { QueryClient, focusManager } from '@tanstack/react-query';
import { AppState, Platform } from 'react-native';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: { retry: 2, staleTime: 60 * 1000 },
  },
});

// Đăng ký AppState thay cho window "focus"/"blur" event của Web
AppState.addEventListener('change', (status) => {
  if (Platform.OS !== 'web') {
    focusManager.setFocused(status === 'active');
  }
});
```

**Điểm khác biệt thứ hai — mất kết nối mạng:** Mobile có tỷ lệ mất mạng/chuyển đổi Wifi-4G cao hơn Web đáng kể. Nên tích hợp thêm `NetInfo` để TanStack Query biết chính xác trạng thái mạng thay vì chỉ dựa vào `navigator.onLine` (API này không đáng tin cậy trên RN):

```bash
npx expo install @react-native-community/netinfo
```

```tsx
import { onlineManager } from '@tanstack/react-query';
import NetInfo from '@react-native-community/netinfo';

onlineManager.setEventListener((setOnline) => {
  return NetInfo.addEventListener((state) => {
    setOnline(!!state.isConnected);
  });
});
```

### 4.4.3. Xử lý lỗi toàn cục kết hợp Navigation

```tsx
// src/lib/queryClient.ts (mở rộng)
import { QueryCache, MutationCache } from '@tanstack/react-query';
import { router } from 'expo-router';
import { toast } from 'sonner-native';

export const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error: any) => {
      if (error?.response?.status === 401) {
        router.replace('/login'); // Điều hướng bằng router thay vì window.location
      }
    },
  }),
  mutationCache: new MutationCache({
    onError: (error: any) => {
      toast.error(error?.response?.data?.message || 'Đã có lỗi xảy ra');
    },
  }),
});
```

---

## 4.5. State Management thuần UI: Zustand

### 4.5.1. Khái niệm

Với state không liên quan server (vd: `viewMode` trong `page.tsx`, hoặc theme, auth state cục bộ), Zustand hoạt động **giống hệt Web**. Khác biệt duy nhất nằm ở middleware `persist` — cần đổi storage engine.

```ts
// src/store/auth.store.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import * as SecureStore from 'expo-secure-store';

interface AuthState {
  user: { id: string; role: string } | null;
  setUser: (user: AuthState['user']) => void;
  logout: () => void;
}

// Adapter để expo-secure-store tương thích interface Storage mà Zustand yêu cầu
const secureStorage = {
  getItem: async (name: string) => (await SecureStore.getItemAsync(name)) ?? null,
  setItem: async (name: string, value: string) => SecureStore.setItemAsync(name, value),
  removeItem: async (name: string) => SecureStore.deleteItemAsync(name),
};

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      setUser: (user) => set({ user }),
      logout: () => set({ user: null }),
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => secureStorage), // Thay localStorage bằng SecureStore
    }
  )
);
```

**So sánh:**

| Web | React Native | Lý do đổi |
|---|---|---|
| `persist` mặc định dùng `localStorage` | Phải chỉ định `storage: createJSONStorage(() => secureStorage)` | Không có `localStorage` |
| Dữ liệu thường (không nhạy cảm) | `AsyncStorage` thay vì `SecureStore` | `SecureStore` giới hạn dung lượng nhỏ (phù hợp token), dữ liệu lớn dùng `AsyncStorage` |

> **Nguyên tắc bảo mật:** Chỉ dùng `SecureStore` cho dữ liệu **nhạy cảm, dung lượng nhỏ** (token, PIN). Với state UI thông thường (filter đã chọn, giỏ hàng nháp...), dùng `@react-native-async-storage/async-storage` — nhanh hơn và không giới hạn dung lượng ngặt như SecureStore.

---

## 4.6. Chuyển đổi toàn bộ UI Layer

Đây là phần thay đổi nhiều nhất. Mình đi theo đúng thứ tự các file bạn gửi.

### 4.6.1. `course-card.tsx` → `CourseListItem.tsx`

Vì mobile không có khái niệm "grid card" cạnh "table row" tách biệt (không đủ không gian màn hình cho bảng), **gộp cả `CourseCard` lẫn `course-columns.tsx` thành một component list-item duy nhất**, dùng chung cho `FlatList`/`FlashList`.

```tsx
// src/components/course/CourseListItem.tsx
import { View, Text, Image, Pressable } from 'react-native';
import { BadgeLevel } from '@/components/common/BadgeLevel';
import { Course } from '@/schemas/course.schema';
import { ClockIcon, UsersIcon, ChevronRightIcon } from 'lucide-react-native';
import * as ContextMenu from 'zeego/context-menu'; // Thay thế DropdownMenu — xem giải thích bên dưới

interface CourseListItemProps {
  course: Course;
  onPress: () => void;
  onEdit: () => void;
  onDelete: () => void;
}

export function CourseListItem({ course, onPress, onEdit, onDelete }: CourseListItemProps) {
  return (
    <ContextMenu.Root>
      <ContextMenu.Trigger>
        <Pressable
          onPress={onPress}
          className="flex-row bg-white rounded-2xl overflow-hidden border border-gray-100 shadow-sm active:opacity-80"
        >
          <View className="size-20">
            {course.image ? (
              <Image source={{ uri: course.image }} className="w-full h-full" resizeMode="cover" />
            ) : (
              <View className="w-full h-full bg-primary/5 items-center justify-center">
                <Text className="text-primary font-black text-lg uppercase">
                  {course.courseName.substring(0, 2)}
                </Text>
              </View>
            )}
          </View>

          <View className="flex-1 p-3 justify-center">
            <View className="flex-row items-center justify-between mb-1">
              <Text className="font-bold text-gray-800 flex-1" numberOfLines={1}>
                {course.courseName}
              </Text>
              <BadgeLevel level={course.level || 'Unknown'} />
            </View>

            <View className="flex-row items-center gap-3 mt-1">
              <View className="flex-row items-center gap-1">
                <ClockIcon size={12} color="#94a3b8" />
                <Text className="text-xs text-gray-500">{course.totalSessions} buổi</Text>
              </View>
              <View className="flex-row items-center gap-1">
                <UsersIcon size={12} color="#94a3b8" />
                <Text className="text-xs text-gray-500">{course.totalClasses || 0} lớp</Text>
              </View>
            </View>

            <Text className="text-base font-black text-gray-900 mt-2">
              {Number(course.tuitionFee).toLocaleString('vi-VN')}đ
            </Text>
          </View>

          <View className="items-center justify-center pr-3">
            <ChevronRightIcon size={18} color="#cbd5e1" />
          </View>
        </Pressable>
      </ContextMenu.Trigger>

      {/* Thay thế DropdownMenu: giữ nguyên vào để sửa/xóa, nhưng kích hoạt bằng long-press */}
      <ContextMenu.Content>
        <ContextMenu.Item key="edit" onSelect={onEdit}>
          <ContextMenu.ItemTitle>Chỉnh sửa</ContextMenu.ItemTitle>
        </ContextMenu.Item>
        <ContextMenu.Item key="delete" destructive onSelect={onDelete}>
          <ContextMenu.ItemTitle>Xóa khóa học</ContextMenu.ItemTitle>
        </ContextMenu.Item>
      </ContextMenu.Content>
    </ContextMenu.Root>
  );
}
```

**Giải thích thay đổi UX quan trọng nhất:**
- `DropdownMenu` (kích hoạt bằng click vào icon `⋮`) → **Context Menu** (kích hoạt bằng **long-press**, giữ ngón tay). Đây không phải bắt buộc kỹ thuật, mà là **chuẩn UX của mobile**: long-press-để-xem-thêm-tùy-chọn là pattern quen thuộc với người dùng iOS/Android (giống hệt việc giữ 1 app icon để hiện menu). `zeego` là thư viện tạo Context Menu native thực sự (không phải giả lập bằng View), cho cảm giác giống app gốc.
- Bấm trực tiếp vào toàn bộ card (`onPress` bên ngoài) để xem chi tiết — không cần nút "Chi tiết" riêng như bản Web, vì trên mobile **toàn bộ diện tích chạm được nên được tận dụng** thay vì chỉ có 1 nút nhỏ.

### 4.6.2. `course-filter.tsx` → Bottom Sheet Filter

Vì mobile không có đủ không gian ngang để đặt 4 `<Select>` cạnh nhau như bản Web, gộp toàn bộ filter vào Bottom Sheet (đã giới thiệu khung sườn ở Chương 3, mục 3.5.4), nội dung bên trong dùng **Picker native** thay `<Select>` của shadcn:

```tsx
// src/components/course/CourseFilterSheet.tsx (phần nội dung bên trong BottomSheetView)
import { View, Text, TextInput, Pressable } from 'react-native';
import { useState } from 'react';
import { CourseLevel } from '@/constants/type';
import { CourseSortBy, SortOrder } from '@/constants/sort';
import { CourseParams } from '@/schemas/course.schema';

interface FilterContentProps {
  onApply: (filters: Partial<CourseParams>) => void;
  onClose: () => void;
}

export function FilterContent({ onApply, onClose }: FilterContentProps) {
  const [level, setLevel] = useState<string | undefined>();
  const [sortBy, setSortBy] = useState<CourseSortBy>(CourseSortBy.COURSE_NAME);
  const [sortOrder, setSortOrder] = useState<SortOrder>(SortOrder.DESC);

  return (
    <View className="gap-6">
      <View>
        <Text className="text-xs font-bold text-gray-400 uppercase mb-2">Trình độ</Text>
        <View className="flex-row flex-wrap gap-2">
          {/* Thay <Select> bằng Chip lựa chọn — pattern phổ biến hơn trên mobile
              vì số lượng option ít (≤5) không cần mở thêm 1 lớp overlay nữa */}
          {Object.values(CourseLevel).map((lvl) => (
            <Pressable
              key={lvl}
              onPress={() => setLevel(level === lvl ? undefined : lvl)}
              className={`px-4 py-2 rounded-full border ${
                level === lvl ? 'bg-primary border-primary' : 'bg-white border-gray-200'
              }`}
            >
              <Text className={level === lvl ? 'text-white font-semibold' : 'text-gray-600'}>
                {lvl}
              </Text>
            </Pressable>
          ))}
        </View>
      </View>

      <View>
        <Text className="text-xs font-bold text-gray-400 uppercase mb-2">Sắp xếp theo</Text>
        <View className="flex-row gap-2">
          {[
            { value: CourseSortBy.COURSE_NAME, label: 'Tên' },
            { value: CourseSortBy.TUITION_FEE, label: 'Học phí' },
            { value: CourseSortBy.LEVEL, label: 'Trình độ' },
          ].map((opt) => (
            <Pressable
              key={opt.value}
              onPress={() => setSortBy(opt.value)}
              className={`px-4 py-2 rounded-full border ${
                sortBy === opt.value ? 'bg-primary border-primary' : 'bg-white border-gray-200'
              }`}
            >
              <Text className={sortBy === opt.value ? 'text-white font-semibold' : 'text-gray-600'}>
                {opt.label}
              </Text>
            </Pressable>
          ))}
        </View>
      </View>

      <Pressable
        onPress={() => {
          onApply({ level, sortBy, sortOrder });
          onClose();
        }}
        className="bg-primary rounded-xl py-4 items-center mt-2"
      >
        <Text className="text-white font-bold">Áp dụng bộ lọc</Text>
      </Pressable>
    </View>
  );
}
```

**Giải thích quyết định thiết kế:** Với số lượng lựa chọn ít (level chỉ có vài giá trị enum), thay `<Select>` (mở thêm 1 lớp dropdown/overlay) bằng **Chip/Segmented control** hiển thị ngay tất cả lựa chọn — giảm số lần chạm (tap) cần thiết, phù hợp nguyên tắc thiết kế mobile "giảm thao tác, tăng khả năng nhìn thấy trực tiếp".

Riêng ô tìm kiếm (`searchValue`) và số dòng/trang (`limit`) không còn cần thiết theo đúng nghĩa cũ: tìm kiếm chuyển thành thanh search cố định phía trên list (không cần nhấn Enter — debounce trực tiếp), còn khái niệm "rows per page" **biến mất hoàn toàn** vì mobile dùng **infinite scroll** (`useInfiniteQuery`) thay vì phân trang bấm số — trình bày ở mục 4.7.

### 4.6.3. `course-dialog.tsx` → `CourseForm.tsx`

Đây là phần lớn nhất — form với `react-hook-form` + `zodResolver` **tái sử dụng gần như nguyên vẹn logic**, chỉ khác input UI và cách upload ảnh.

```tsx
// src/components/course/CourseForm.tsx
import { View, Text, TextInput, Pressable, ActivityIndicator, Image } from 'react-native';
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { useEffect, useState } from 'react';
import * as ImagePicker from 'expo-image-picker';
import { Course, CourseInput, courseInputSchema } from '@/schemas/course.schema';
import { CourseLevel } from '@/constants/type';
import { useImageUpload } from '@/hooks/useImageUpload'; // Bản RN — xem 4.6.4
import { ImageIcon, XIcon, BookOpenIcon } from 'lucide-react-native';

interface CourseFormProps {
  initialData?: Course | null;
  onSubmit: (data: CourseInput) => void;
  loading?: boolean;
}

export function CourseForm({ initialData, onSubmit, loading }: CourseFormProps) {
  const {
    control,
    handleSubmit,
    reset,
    watch,
    setValue,
    formState: { errors },
    // Logic zodResolver GIỐNG HỆT bản Web — validate rule không đổi 1 ký tự
  } = useForm<CourseInput>({ resolver: zodResolver(courseInputSchema) });

  const { handleUpload, isUploading } = useImageUpload();
  const currentImage = watch('image');
  const currentLevel = watch('level');

  useEffect(() => {
    if (initialData) {
      reset({
        courseName: initialData.courseName,
        tuitionFee: initialData.tuitionFee.toString(),
        level: initialData.level,
        totalSessions: initialData.totalSessions,
        image: initialData.image,
        description: initialData.description,
      });
    } else {
      reset({ courseName: '', tuitionFee: '', level: '', totalSessions: 0, image: null, description: '' });
    }
  }, [initialData, reset]);

  // Thay <input type="file"> bằng expo-image-picker — mở thư viện ảnh hoặc camera của máy
  const pickImage = async () => {
    const permission = await ImagePicker.requestMediaLibraryPermissionsAsync();
    if (!permission.granted) return;

    const result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      quality: 0.8,
      allowsEditing: true,
      aspect: [16, 9],
    });

    if (!result.canceled) {
      const uploaded = await handleUpload(result.assets[0].uri, 'courses');
      if (uploaded) setValue('image', uploaded.url);
    }
  };

  return (
    <View className="gap-6">
      {/* Upload ảnh */}
      <View className="gap-2">
        <Text className="text-xs font-black uppercase text-gray-400">Hình ảnh khóa học</Text>
        {currentImage ? (
          <View className="relative h-48 rounded-2xl overflow-hidden">
            <Image source={{ uri: currentImage }} className="w-full h-full" resizeMode="cover" />
            <Pressable
              onPress={() => setValue('image', null)}
              className="absolute top-3 right-3 size-9 rounded-full bg-white/90 items-center justify-center"
            >
              <XIcon size={18} color="#ef4444" />
            </Pressable>
          </View>
        ) : (
          <Pressable
            onPress={pickImage}
            disabled={isUploading}
            className="h-48 rounded-2xl border-2 border-dashed border-gray-200 items-center justify-center bg-gray-50"
          >
            {isUploading ? (
              <ActivityIndicator color="#4F46E5" />
            ) : (
              <>
                <ImageIcon size={28} color="#9ca3af" />
                <Text className="text-sm font-bold text-gray-500 mt-2">Chọn ảnh từ thư viện</Text>
              </>
            )}
          </Pressable>
        )}
      </View>

      {/* Tên khóa học — dùng Controller vì TextInput của RN không hỗ trợ register() trực tiếp như input HTML */}
      <View className="gap-2">
        <Text className="text-xs font-black uppercase text-gray-400">Tên khóa học</Text>
        <Controller
          control={control}
          name="courseName"
          render={({ field: { onChange, value } }) => (
            <TextInput
              value={value}
              onChangeText={onChange}
              placeholder="VD: English Foundation A1"
              className="h-14 rounded-2xl border border-gray-200 px-4 text-base font-bold"
            />
          )}
        />
        {errors.courseName && (
          <Text className="text-xs text-red-500 font-bold">{errors.courseName.message}</Text>
        )}
      </View>

      {/* Trình độ — Chip chọn thay <Select>, giống CourseFilterSheet */}
      <View className="gap-2">
        <Text className="text-xs font-black uppercase text-gray-400">Trình độ</Text>
        <View className="flex-row flex-wrap gap-2">
          {Object.values(CourseLevel).map((lvl) => (
            <Pressable
              key={lvl}
              onPress={() => setValue('level', lvl)}
              className={`px-4 py-2.5 rounded-full border ${
                currentLevel === lvl ? 'bg-primary border-primary' : 'bg-white border-gray-200'
              }`}
            >
              <Text className={currentLevel === lvl ? 'text-white font-bold' : 'text-gray-600'}>
                {lvl}
              </Text>
            </Pressable>
          ))}
        </View>
      </View>

      {/* Số buổi học */}
      <View className="gap-2">
        <Text className="text-xs font-black uppercase text-gray-400">Số buổi học</Text>
        <Controller
          control={control}
          name="totalSessions"
          render={({ field: { onChange, value } }) => (
            <TextInput
              value={value?.toString() ?? ''}
              onChangeText={(text) => onChange(Number(text) || 0)}
              keyboardType="numeric" // Bật bàn phím số — không tồn tại khái niệm này trên Web input
              placeholder="VD: 24"
              className="h-14 rounded-2xl border border-gray-200 px-4 text-base font-bold"
            />
          )}
        />
        {errors.totalSessions && (
          <Text className="text-xs text-red-500 font-bold">{errors.totalSessions.message}</Text>
        )}
      </View>

      {/* Học phí */}
      <View className="gap-2">
        <Text className="text-xs font-black uppercase text-gray-400">Học phí (VNĐ)</Text>
        <Controller
          control={control}
          name="tuitionFee"
          render={({ field: { onChange, value } }) => (
            <TextInput
              value={value}
              onChangeText={onChange}
              keyboardType="numeric"
              placeholder="VD: 2500000"
              className="h-14 rounded-2xl border border-gray-200 px-4 text-lg font-black text-primary"
            />
          )}
        />
        {errors.tuitionFee && (
          <Text className="text-xs text-red-500 font-bold">{errors.tuitionFee.message}</Text>
        )}
      </View>

      {/* Mô tả */}
      <View className="gap-2">
        <Text className="text-xs font-black uppercase text-gray-400">Mô tả chi tiết</Text>
        <Controller
          control={control}
          name="description"
          render={({ field: { onChange, value } }) => (
            <TextInput
              value={value ?? ''}
              onChangeText={onChange}
              placeholder="Nhập thông tin giới thiệu..."
              multiline
              numberOfLines={4}
              textAlignVertical="top" // Bắt buộc trên Android để text bắt đầu từ trên xuống, không căn giữa
              className="min-h-[100px] rounded-2xl border border-gray-200 p-4 text-sm"
            />
          )}
        />
      </View>

      <Pressable
        onPress={handleSubmit(onSubmit)}
        disabled={loading || isUploading}
        className="bg-primary rounded-2xl py-4 items-center mt-2 disabled:opacity-50"
      >
        {loading ? (
          <ActivityIndicator color="white" />
        ) : (
          <Text className="text-white font-black text-base">
            {initialData ? 'Cập nhật khóa học' : 'Lưu khóa học'}
          </Text>
        )}
      </Pressable>
    </View>
  );
}
```

**Bảng đối chiếu chi tiết những gì thay đổi trong form:**

| Bản Web (`course-dialog.tsx`) | Bản React Native | Giải thích |
|---|---|---|
| `{...register("courseName")}` | `<Controller control={control} name="courseName" render={...} />` | `TextInput` của RN không hỗ trợ `ref` theo interface mà `register()` của react-hook-form cần cho input HTML thuần — bắt buộc dùng `Controller` |
| `<input type="file" onChange={onFileChange}>` | `expo-image-picker` → `launchImageLibraryAsync()` | Không có input file trên mobile — phải mở thư viện ảnh/camera hệ thống qua permission |
| `<Select>` (shadcn) | Chip lựa chọn (`Pressable` + điều kiện style) | Giảm số lớp overlay, phù hợp UX chạm |
| `<Textarea>` | `<TextInput multiline textAlignVertical="top">` | RN không có thẻ textarea riêng — dùng `TextInput` với `multiline` |
| Validate rule (Zod) | **Không đổi 1 dòng** | Đây là điểm mấu chốt: business logic validate hoàn toàn độc lập UI |

### 4.6.4. `useImageUpload` — thay đổi nguồn dữ liệu ảnh

```tsx
// src/hooks/useImageUpload.ts
import { useState } from 'react';
import http from '@/utils/http';

export function useImageUpload() {
  const [isUploading, setIsUploading] = useState(false);

  const handleUpload = async (localUri: string, folder: string) => {
    setIsUploading(true);
    try {
      // Trên Web: FormData nhận trực tiếp đối tượng File từ <input>
      // Trên RN: phải tự dựng object mô tả file từ local URI (không có File API)
      const formData = new FormData();
      formData.append('file', {
        uri: localUri,
        name: `${folder}-${Date.now()}.jpg`,
        type: 'image/jpeg',
      } as any);
      formData.append('folder', folder);

      const { data } = await http.post('/media/upload', formData, {
        headers: { 'Content-Type': 'multipart/form-data' },
      });
      return { url: data.url, publicId: data.publicId };
    } finally {
      setIsUploading(false);
    }
  };

  return { handleUpload, isUploading };
}
```

**Giải thích khác biệt kỹ thuật quan trọng:** Trên Web, `e.target.files[0]` trả về một đối tượng `File` chuẩn mà `FormData` hiểu trực tiếp. Trên RN, ảnh chỉ tồn tại dưới dạng **đường dẫn file cục bộ** (`localUri`, dạng `file:///...`) — phải tự dựng object có 3 trường `uri`/`name`/`type` để `FormData` (bản polyfill của RN) hiểu và đóng gói đúng khi gửi multipart request.

### 4.6.5. `page.tsx` → Màn hình tổng hợp

```tsx
// app/(admin)/(tabs)/course/index.tsx
import { View, Text, TextInput } from 'react-native';
import { FlashList } from '@shopify/flash-list';
import { useRef, useState } from 'react';
import { useRouter } from 'expo-router';
import BottomSheet from '@gorhom/bottom-sheet';
import { useDebounce } from '@/hooks/useDebounce';
import { useGetCourses, useDeleteCourse } from '@/queries/useCourseQuery';
import { CourseParams } from '@/schemas/course.schema';
import { CourseSortBy, SortOrder } from '@/constants/sort';
import { CourseListItem } from '@/components/course/CourseListItem';
import { CourseFilterSheet } from '@/components/course/CourseFilterSheet';
import { confirmDelete } from '@/hooks/useConfirmDelete';
import { SearchIcon, SlidersHorizontalIcon } from 'lucide-react-native';
import { Pressable } from 'react-native';

export default function AdminCourseScreen() {
  const router = useRouter();
  const filterSheetRef = useRef<BottomSheet>(null);

  const [searchInput, setSearchInput] = useState('');
  const debouncedSearch = useDebounce(searchInput, 400); // Thay thế "nhấn Enter để search" bản Web

  const [params, setParams] = useState<CourseParams>({
    page: 1,
    limit: 20,
    sortOrder: SortOrder.DESC,
    sortBy: CourseSortBy.COURSE_NAME,
  });

  const { data, isLoading, refetch, isRefetching } = useGetCourses({
    ...params,
    search: debouncedSearch,
  });
  const deleteMutation = useDeleteCourse();
  const courses = data?.data || [];

  return (
    <View className="flex-1 bg-gray-50">
      <View className="px-4 pt-14 pb-3 bg-white border-b border-gray-100 gap-3">
        <Text className="text-2xl font-black text-gray-900">Khóa học</Text>
        <View className="flex-row items-center gap-2">
          <View className="flex-1 flex-row items-center bg-gray-100 rounded-xl px-3 h-11">
            <SearchIcon size={16} color="#9ca3af" />
            <TextInput
              value={searchInput}
              onChangeText={setSearchInput}
              placeholder="Tìm khóa học..."
              className="flex-1 ml-2 text-sm"
            />
          </View>
          <Pressable
            onPress={() => filterSheetRef.current?.expand()}
            className="size-11 rounded-xl bg-gray-100 items-center justify-center"
          >
            <SlidersHorizontalIcon size={18} color="#4b5563" />
          </Pressable>
        </View>
      </View>

      <FlashList
        data={courses}
        estimatedItemSize={100}
        contentContainerStyle={{ padding: 16, gap: 10 }}
        onRefresh={refetch}
        refreshing={isRefetching}
        renderItem={({ item }) => (
          <CourseListItem
            course={item}
            onPress={() => router.push(`/course/${item.courseId}`)}
            onEdit={() => router.push(`/course-form?id=${item.courseId}`)}
            onDelete={() =>
              confirmDelete(item.courseName, () => deleteMutation.mutate(item.courseId!))
            }
          />
        )}
        ListEmptyComponent={
          !isLoading ? (
            <Text className="text-center text-gray-400 mt-10">Không tìm thấy khóa học</Text>
          ) : null
        }
      />

      <CourseFilterSheet
        ref={filterSheetRef}
        onApply={(filters) => setParams((prev) => ({ ...prev, ...filters, page: 1 }))}
      />
    </View>
  );
}
```

```ts
// src/hooks/useDebounce.ts — thay thế cơ chế "nhấn Enter mới search" bằng debounce tự động
import { useEffect, useState } from 'react';

export function useDebounce<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);
  return debounced;
}
```

**Giải thích các quyết định thiết kế khác biệt cuối cùng:**
- Không còn `viewMode` (table/grid) — vì mobile chỉ có **một bố cục list duy nhất phù hợp màn hình hẹp**, không cần switch giữa 2 chế độ hiển thị như Web.
- `onSearch` dạng `<form onSubmit>` (yêu cầu nhấn Enter) → **debounce tự động khi gõ** — vì bàn phím ảo mobile không có phím Enter rõ ràng như bàn phím vật lý, và gõ-tìm-ngay là hành vi người dùng mobile mong đợi.
- Nút "Thêm khóa học" không còn nằm ở header cùng công tắc `viewMode` — đã chuyển thành nút `+` nổi hoặc icon trên header (xem lại `app/(admin)/course/index.tsx` ở Chương 3) để tối ưu không gian màn hình hẹp.

---

## 4.7. Phân trang (Pagination) → Infinite Scroll

### 4.7.1. Khái niệm khác biệt

`DataPagination` (bấm số trang 1, 2, 3...) là pattern phù hợp màn hình rộng, nơi người dùng dùng chuột để bấm chính xác. Trên mobile, pattern chuẩn là **Infinite Scroll** — tự động tải thêm dữ liệu khi cuộn gần đến cuối danh sách.

```tsx
// src/queries/useCourseQuery.ts — bổ sung hook mới cho RN, KHÔNG thay thế hook cũ
import { useInfiniteQuery } from '@tanstack/react-query';

export const useGetCoursesInfinite = (baseParams: Omit<CourseParams, 'page'>) => {
  return useInfiniteQuery({
    queryKey: [...COURSE_QUERY_KEY, 'infinite', baseParams],
    queryFn: ({ pageParam = 1 }) =>
      courseService.getCourses({ ...baseParams, page: pageParam }).then((res) => res.data),
    initialPageParam: 1,
    getNextPageParam: (lastPage, allPages) => {
      const hasMore = allPages.flatMap((p) => p.data).length < (lastPage.meta?.total ?? 0);
      return hasMore ? allPages.length + 1 : undefined;
    },
  });
};
```

```tsx
// Sử dụng trong FlashList
const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useGetCoursesInfinite(params);
const courses = data?.pages.flatMap((page) => page.data) ?? [];

<FlashList
  data={courses}
  onEndReached={() => hasNextPage && fetchNextPage()}
  onEndReachedThreshold={0.5} // Kích hoạt khi còn cách đáy 50% chiều cao viewport
  ListFooterComponent={isFetchingNextPage ? <ActivityIndicator className="my-4" /> : null}
  // ... renderItem giữ nguyên
/>
```

**Giải thích:** `useInfiniteQuery` là API riêng của TanStack Query dành cho chính xác nhu cầu này — tự động quản lý mảng nhiều "trang" dữ liệu, gộp lại thành 1 danh sách phẳng qua `data.pages.flatMap()`. `onEndReached` là prop của `FlashList`/`FlatList` tự động gọi callback khi người dùng cuộn gần đến cuối — đây là cơ chế **không có API tương đương trực tiếp bên Web** (Web thường tự triển khai bằng `IntersectionObserver`).

---

## Tổng kết Chương 4

| Lớp kiến trúc | Mức tái sử dụng từ Web | Thay đổi chính |
|---|---|---|
| Zod Schema | 100% | Không đổi |
| Axios Service | ~95% | Đổi nơi lưu token (`SecureStore` thay `localStorage`) |
| TanStack Query Hooks | ~95% | Đổi cơ chế refetch (`AppState` thay `window focus`), thêm `useInfiniteQuery` |
| Zustand Store | ~90% | Đổi storage adapter cho `persist` |
| UI Component | ~10–20% | Viết lại gần như hoàn toàn theo component RN + pattern UX mobile (Bottom Sheet, Chip, Context Menu, Image Picker, Infinite Scroll) |

**Thông điệp cốt lõi của chương:** Nếu kiến trúc dự án Web của bạn đã tách bạch rõ ràng 4 lớp (đúng như bộ code bạn cung cấp), thì việc "port" một module CRUD hoàn chỉnh sang React Native **không phải viết lại từ đầu** — mà là **giữ nguyên bộ não (business logic), thay bộ da (UI)**. Đây là lợi thế lớn nhất khi một đội ngũ đã quen Next.js chuyển sang làm mobile bằng React Native.

**Chương tiếp theo (Chương 5)** sẽ đi vào Forms nâng cao & UI Component Library — nơi xây dựng bộ Design System riêng (thay thế vai trò của shadcn/ui) để tái sử dụng nhất quán across toàn bộ ứng dụng.
