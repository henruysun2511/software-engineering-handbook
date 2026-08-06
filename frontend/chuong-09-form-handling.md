# Chương 9: Form Handling

Xử lý form là một trong những tác vụ phổ biến nhất trong frontend. Một form hoàn chỉnh cần xử lý: thu thập giá trị, validate dữ liệu, hiển thị lỗi, và submit. Làm thủ công bằng `useState` nhanh chóng trở nên phức tạp khi form có nhiều trường.

Bộ đôi **React Hook Form** + **Zod** là giải pháp phổ biến nhất trong hệ sinh thái React hiện đại, giải quyết toàn bộ vấn đề trên một cách hiệu quả và type-safe.

---

## React Hook Form

React Hook Form (RHF) là thư viện quản lý form dựa trên **uncontrolled inputs** — giá trị được lưu trong DOM thay vì React state. Điều này giúp form không re-render mỗi lần người dùng gõ phím, cải thiện hiệu suất đáng kể so với cách tiếp cận controlled.

### Các khái niệm cốt lõi

| Khái niệm | Mô tả |
|---|---|
| `useForm()` | Hook khởi tạo form, trả về các utility function |
| `register()` | Đăng ký input với form, cung cấp ref và event handlers |
| `handleSubmit()` | Wrapper cho submit — tự validate trước khi gọi callback |
| `formState` | Trạng thái của form: `errors`, `isSubmitting`, `isDirty`, `isValid` |
| `watch()` | Theo dõi giá trị của một hoặc nhiều field theo thời gian thực |
| `setValue()` | Gán giá trị cho field từ bên ngoài form |
| `reset()` | Reset toàn bộ form về giá trị mặc định |

### Form cơ bản

```tsx
import { useForm } from "react-hook-form";

interface LoginFormValues {
  email: string;
  password: string;
}

function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<LoginFormValues>();

  async function onSubmit(data: LoginFormValues) {
    // data đã được validate — kiểu chính xác là LoginFormValues
    await loginUser(data);
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <label>Email</label>
        <input
          type="email"
          {...register("email", {
            required: "Email là bắt buộc",
            pattern: {
              value: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
              message: "Email không hợp lệ",
            },
          })}
        />
        {errors.email && <p className="error">{errors.email.message}</p>}
      </div>

      <div>
        <label>Mật khẩu</label>
        <input
          type="password"
          {...register("password", {
            required: "Mật khẩu là bắt buộc",
            minLength: { value: 8, message: "Tối thiểu 8 ký tự" },
          })}
        />
        {errors.password && <p className="error">{errors.password.message}</p>}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Đang đăng nhập..." : "Đăng nhập"}
      </button>
    </form>
  );
}
```

---

## Zod

Zod là thư viện schema validation TypeScript-first. Thay vì định nghĩa validation trong từng field riêng lẻ, Zod cho phép mô tả toàn bộ cấu trúc và quy tắc dữ liệu trong một **schema** — schema này có thể tái sử dụng ở cả frontend lẫn backend.

### Schema cơ bản

```tsx
import { z } from "zod";

const loginSchema = z.object({
  email: z
    .string()
    .min(1, "Email là bắt buộc")
    .email("Email không hợp lệ"),
  password: z
    .string()
    .min(8, "Tối thiểu 8 ký tự")
    .max(100, "Tối đa 100 ký tự"),
});

// Tự động suy ra TypeScript type từ schema — không cần định nghĩa interface riêng
type LoginFormValues = z.infer<typeof loginSchema>;
// { email: string; password: string }
```

Zod hỗ trợ các kiểu phức tạp hơn:

```tsx
const registerSchema = z
  .object({
    name: z.string().min(2, "Tên tối thiểu 2 ký tự").trim(),
    email: z.string().email("Email không hợp lệ"),
    password: z.string().min(8, "Tối thiểu 8 ký tự"),
    confirmPassword: z.string(),
    role: z.enum(["admin", "user", "moderator"]),
    age: z.number().int().min(18, "Phải đủ 18 tuổi").optional(),
    website: z.string().url("URL không hợp lệ").optional().or(z.literal("")),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: "Mật khẩu không khớp",
    path: ["confirmPassword"], // Lỗi gắn vào field confirmPassword
  });

type RegisterFormValues = z.infer<typeof registerSchema>;
```

---

## Tích hợp React Hook Form + Zod

`@hookform/resolvers` là cầu nối giúp RHF dùng Zod schema để validate thay vì validation nội bộ.

```bash
npm install react-hook-form zod @hookform/resolvers
```

```tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

// 1. Định nghĩa schema
const profileSchema = z.object({
  name: z.string().min(2, "Tên tối thiểu 2 ký tự"),
  email: z.string().email("Email không hợp lệ"),
  bio: z.string().max(200, "Tối đa 200 ký tự").optional(),
  role: z.enum(["admin", "user"]),
});

// 2. Suy ra type từ schema
type ProfileFormValues = z.infer<typeof profileSchema>;

// 3. Dùng trong component
function ProfileForm({ defaultValues }: { defaultValues: ProfileFormValues }) {
  const {
    register,
    handleSubmit,
    watch,
    formState: { errors, isSubmitting, isDirty },
  } = useForm<ProfileFormValues>({
    resolver: zodResolver(profileSchema), // Kết nối Zod với RHF
    defaultValues,
  });

  const bio = watch("bio");

  async function onSubmit(data: ProfileFormValues) {
    await updateProfile(data);
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <div>
        <label>Họ tên</label>
        <input {...register("name")} />
        {errors.name && <p className="error">{errors.name.message}</p>}
      </div>

      <div>
        <label>Email</label>
        <input type="email" {...register("email")} />
        {errors.email && <p className="error">{errors.email.message}</p>}
      </div>

      <div>
        <label>Giới thiệu</label>
        <textarea {...register("bio")} />
        <span>{bio?.length ?? 0}/200</span>
        {errors.bio && <p className="error">{errors.bio.message}</p>}
      </div>

      <div>
        <label>Vai trò</label>
        <select {...register("role")}>
          <option value="user">User</option>
          <option value="admin">Admin</option>
        </select>
        {errors.role && <p className="error">{errors.role.message}</p>}
      </div>

      <button type="submit" disabled={isSubmitting || !isDirty}>
        {isSubmitting ? "Đang lưu..." : "Lưu thay đổi"}
      </button>
    </form>
  );
}
```

---

## Validation

### Chiến lược validate

React Hook Form hỗ trợ nhiều thời điểm validate qua option `mode`:

| Mode | Validate khi nào |
|---|---|
| `onSubmit` (mặc định) | Chỉ khi submit |
| `onBlur` | Khi rời khỏi field |
| `onChange` | Mỗi lần thay đổi |
| `onTouched` | Lần đầu blur, sau đó onChange |
| `all` | onChange + onBlur |

```tsx
const { register, handleSubmit } = useForm({
  resolver: zodResolver(schema),
  mode: "onTouched", // Validate sau khi user rời field, rồi real-time
});
```

### Tái sử dụng schema ở server

Một trong những lợi ích lớn nhất của Zod là schema có thể dùng chung:

```tsx
// lib/schemas/user.ts — dùng được cả frontend và backend
import { z } from "zod";

export const createUserSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  password: z.string().min(8),
});

export type CreateUserDto = z.infer<typeof createUserSchema>;
```

```tsx
// Server Action (Next.js) — validate bằng cùng schema
"use server";
import { createUserSchema } from "@/lib/schemas/user";

export async function createUserAction(formData: FormData) {
  const raw = {
    name: formData.get("name"),
    email: formData.get("email"),
    password: formData.get("password"),
  };

  const result = createUserSchema.safeParse(raw);

  if (!result.success) {
    return { error: result.error.flatten().fieldErrors };
  }

  // result.data có kiểu CreateUserDto — hoàn toàn type-safe
  await db.user.create({ data: result.data });
}
```

### So sánh cách quản lý form

| | useState thuần | React Hook Form | RHF + Zod |
|---|---|---|---|
| Re-render mỗi keystroke | Có | Không | Không |
| Validation | Thủ công | Built-in | Schema tập trung |
| TypeScript | Thủ công | Partial | Đầy đủ, infer từ schema |
| Tái sử dụng validation | Khó | Khó | Dễ (share schema) |
| Độ phức tạp setup | Thấp | Trung bình | Trung bình |
| Phù hợp | Form 1-2 field đơn giản | Form trung bình | Form phức tạp, production |
