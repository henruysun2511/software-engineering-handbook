# Zod – Tổng hợp kiến thức đầy đủ

> Docs chính thức: https://zod.dev

---

## 1. Zod là gì?

Zod là thư viện **schema declaration và validation** cho TypeScript. Zod tự động suy ra kiểu TypeScript từ schema — không cần viết type và validation riêng biệt.

**Tại sao dùng Zod:**
- Một schema = vừa validate runtime vừa có TypeScript type
- Tích hợp tốt với React Hook Form, TanStack Query, NestJS, tRPC
- Type inference mạnh mẽ — không bao giờ type bị lệch với validation
- Parse thay vì chỉ validate — transform data khi parse

**So sánh với class-validator (NestJS):**

| | Zod | class-validator |
|---|---|---|
| Type inference | ✅ Tự động | ❌ Phải viết tay |
| Dùng ở client | ✅ | ❌ (decorator cần reflect-metadata) |
| Composable | ✅ Rất tốt | ⚠️ Hạn chế |
| Dùng với RHF | ✅ `@hookform/resolvers/zod` | ⚠️ |
| Transform data | ✅ `.transform()` | ❌ |

---

## 2. Cài đặt

```bash
npm install zod
```

---

## 3. Primitive Types

```typescript
import { z } from 'zod';

// String
z.string()
z.string().min(1)
z.string().max(100)
z.string().length(8)                     // đúng 8 ký tự
z.string().email()
z.string().url()
z.string().uuid()
z.string().cuid()
z.string().regex(/^[A-Z]{3}\d{3}$/)
z.string().startsWith('prefix_')
z.string().endsWith('.com')
z.string().includes('@')
z.string().trim()                        // trim trước khi validate
z.string().toLowerCase()                 // chuyển lowercase
z.string().toUpperCase()
z.string().datetime()                    // ISO 8601 datetime
z.string().ip()                          // IPv4 hoặc IPv6
z.string().nonempty()                    // = min(1)

// Number
z.number()
z.number().int()                         // số nguyên
z.number().positive()                    // > 0
z.number().negative()                    // < 0
z.number().nonnegative()                 // >= 0
z.number().min(1)
z.number().max(100)
z.number().multipleOf(5)                 // chia hết cho 5
z.number().finite()                      // không phải Infinity
z.number().safe()                        // trong phạm vi Number.MAX_SAFE_INTEGER

// Boolean
z.boolean()

// Date
z.date()
z.date().min(new Date('2000-01-01'))
z.date().max(new Date())

// BigInt
z.bigint()

// Null / Undefined / Void
z.null()
z.undefined()
z.void()
z.never()                                // không bao giờ valid

// Any / Unknown
z.any()
z.unknown()
```

---

## 4. Object Schema

```typescript
import { z } from 'zod';

const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string().min(2).max(50),
  age: z.number().int().min(1).max(120).optional(),
  role: z.enum(['admin', 'user', 'moderator']),
  isActive: z.boolean().default(true),
  createdAt: z.date().default(() => new Date()),
  metadata: z.record(z.string()),          // { [key: string]: string }
});

// Suy ra TypeScript type từ schema
type User = z.infer<typeof UserSchema>;
// type User = {
//   id: string;
//   email: string;
//   name: string;
//   age?: number;
//   role: 'admin' | 'user' | 'moderator';
//   isActive: boolean;
//   createdAt: Date;
//   metadata: Record<string, string>;
// }

// Parse (throw nếu lỗi)
const user = UserSchema.parse({
  id: 'abc-123',
  email: 'alice@example.com',
  name: 'Alice',
  role: 'admin',
  metadata: {},
});

// SafeParse (không throw, trả về result object)
const result = UserSchema.safeParse(data);
if (result.success) {
  console.log(result.data);     // typed User
} else {
  console.error(result.error);  // ZodError với chi tiết
}
```

### Object methods

```typescript
const BaseSchema = z.object({
  id: z.string(),
  createdAt: z.date(),
});

const UpdateSchema = z.object({
  name: z.string(),
  email: z.string().email(),
});

// .extend() — thêm fields
const ExtendedSchema = BaseSchema.extend({
  name: z.string(),
  email: z.string(),
});

// .merge() — merge 2 schema
const MergedSchema = BaseSchema.merge(UpdateSchema);

// .pick() — chỉ lấy một số fields
const LoginSchema = UpdateSchema.pick({ email: true });

// .omit() — bỏ một số fields
const PublicUserSchema = UserSchema.omit({ password: true, refreshToken: true });

// .partial() — tất cả fields thành optional (dùng cho Update DTO)
const PatchUserSchema = UserSchema.partial();

// .partial() chỉ một số fields
const PartialSchema = UserSchema.partial({ age: true, metadata: true });

// .required() — tất cả fields thành required
const RequiredSchema = UserSchema.required();

// .strip() — xóa unknown fields (mặc định)
// .passthrough() — giữ unknown fields
// .strict() — throw nếu có unknown fields
const StrictSchema = UserSchema.strict();
```

---

## 5. Array & Tuple

```typescript
// Array
z.array(z.string())
z.array(z.string()).min(1)           // ít nhất 1 phần tử
z.array(z.string()).max(10)          // tối đa 10 phần tử
z.array(z.string()).length(3)        // đúng 3 phần tử
z.array(z.string()).nonempty()       // không rỗng
z.string().array()                   // cú pháp khác

// Array of objects
const ProductListSchema = z.array(
  z.object({
    id: z.string(),
    name: z.string(),
    price: z.number().positive(),
  })
);

// Tuple — mảng cố định kiểu từng phần tử
const CoordSchema = z.tuple([z.number(), z.number()]);
// [lat, lng]

const RGBSchema = z.tuple([
  z.number().min(0).max(255),  // R
  z.number().min(0).max(255),  // G
  z.number().min(0).max(255),  // B
]);

// Tuple với rest
const LogSchema = z.tuple([
  z.string(),             // level
  z.date(),               // timestamp
]).rest(z.unknown());     // ...args
```

---

## 6. Enum & Union & Literal

```typescript
// Enum
const RoleSchema = z.enum(['admin', 'user', 'moderator']);
type Role = z.infer<typeof RoleSchema>;
// type Role = 'admin' | 'user' | 'moderator'

// Lấy enum values
const roles = RoleSchema.options; // ['admin', 'user', 'moderator']

// Enum từ const object (tái sử dụng ở nhiều nơi)
const STATUS = {
  PENDING:   'pending',
  PAID:      'paid',
  CANCELLED: 'cancelled',
} as const;

const StatusSchema = z.enum([STATUS.PENDING, STATUS.PAID, STATUS.CANCELLED]);
// Hoặc ngắn hơn:
const StatusSchema = z.nativeEnum(STATUS);

// Literal — giá trị cố định
z.literal('admin')
z.literal(42)
z.literal(true)
z.literal(null)

// Union — một trong nhiều kiểu
const StringOrNumber = z.union([z.string(), z.number()]);
// Cú pháp ngắn hơn:
const StringOrNumber = z.string().or(z.number());

// Discriminated Union — union có trường phân biệt
const NotificationSchema = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('email'),
    to: z.string().email(),
    subject: z.string(),
  }),
  z.object({
    type: z.literal('sms'),
    phone: z.string(),
  }),
  z.object({
    type: z.literal('push'),
    deviceToken: z.string(),
  }),
]);

type Notification = z.infer<typeof NotificationSchema>;
// TypeScript biết chính xác fields dựa vào `type`

// Intersection — kết hợp schemas
const AdminSchema = UserSchema.and(
  z.object({ permissions: z.array(z.string()) })
);
```

---

## 7. Optional, Nullable, Default

```typescript
// Optional — có thể undefined
z.string().optional()
// type: string | undefined

// Nullable — có thể null
z.string().nullable()
// type: string | null

// Nullish — optional + nullable
z.string().nullish()
// type: string | null | undefined

// Default — giá trị mặc định khi undefined
z.string().default('guest')
z.boolean().default(false)
z.number().default(0)
z.array(z.string()).default([])
z.date().default(() => new Date())     // dùng function để tránh reference cũ

// Ví dụ thực tế
const CreatePostSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1),
  status: z.enum(['draft', 'published']).default('draft'),
  tags: z.array(z.string()).default([]),
  publishedAt: z.date().nullable().default(null),
  authorId: z.string().uuid(),
});
```

---

## 8. Transform & Preprocess

### .transform() — Biến đổi data sau khi validate

```typescript
// Chuyển string thành số
const NumberFromString = z.string().transform((val) => parseInt(val, 10));

// Chuyển về chữ hoa
const UpperEmail = z.string().email().transform((val) => val.toLowerCase());

// Transform object
const UserSchema = z.object({
  firstName: z.string(),
  lastName: z.string(),
}).transform((data) => ({
  ...data,
  fullName: `${data.firstName} ${data.lastName}`,
}));

type User = z.infer<typeof UserSchema>;
// { firstName: string; lastName: string; fullName: string }

// Trim + lowercase email
const EmailSchema = z
  .string()
  .trim()
  .toLowerCase()
  .email('Email không hợp lệ');

// Chuyển string date thành Date object
const DateFromString = z.string().datetime().transform((val) => new Date(val));
```

### z.preprocess() — Xử lý data TRƯỚC khi validate

```typescript
// Chuyển string sang number trước khi validate number
const NumberSchema = z.preprocess(
  (val) => (typeof val === 'string' ? parseFloat(val) : val),
  z.number().positive()
);

// Chuyển string sang boolean (từ form HTML)
const BooleanFromString = z.preprocess(
  (val) => val === 'true' || val === '1' || val === true,
  z.boolean()
);

// Parse JSON string
const ParsedJSON = z.preprocess(
  (val) => (typeof val === 'string' ? JSON.parse(val) : val),
  z.object({ id: z.string(), name: z.string() })
);

// Coerce — shorthand của preprocess phổ biến
z.coerce.string()    // String(input)
z.coerce.number()    // Number(input) — hữu ích với query params
z.coerce.boolean()   // Boolean(input)
z.coerce.date()      // new Date(input)

// Thực tế: query params từ URL luôn là string
const QuerySchema = z.object({
  page:   z.coerce.number().int().positive().default(1),
  limit:  z.coerce.number().int().min(1).max(100).default(10),
  search: z.string().optional(),
  active: z.coerce.boolean().optional(),
});
```

---

## 9. Refinement – Validation phức tạp

```typescript
// .refine() — custom validation
const PasswordSchema = z
  .string()
  .min(8, 'Tối thiểu 8 ký tự')
  .refine(
    (val) => /[A-Z]/.test(val),
    'Phải có ít nhất 1 chữ hoa'
  )
  .refine(
    (val) => /[0-9]/.test(val),
    'Phải có ít nhất 1 số'
  )
  .refine(
    (val) => /[^A-Za-z0-9]/.test(val),
    'Phải có ít nhất 1 ký tự đặc biệt'
  );

// .refine() với options
const AgeSchema = z.number().refine(
  (val) => val >= 18,
  {
    message: 'Phải đủ 18 tuổi',
    path: ['age'],       // đường dẫn field lỗi
  }
);

// .superRefine() — control lỗi nhiều hơn
const RegisterSchema = z
  .object({
    password: z.string().min(8),
    confirmPassword: z.string(),
  })
  .superRefine(({ password, confirmPassword }, ctx) => {
    if (password !== confirmPassword) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: 'Mật khẩu không khớp',
        path: ['confirmPassword'],    // lỗi xuất hiện ở field confirmPassword
      });
    }
  });

// Object-level validation
const DateRangeSchema = z
  .object({
    startDate: z.date(),
    endDate: z.date(),
  })
  .refine(
    ({ startDate, endDate }) => endDate > startDate,
    {
      message: 'Ngày kết thúc phải sau ngày bắt đầu',
      path: ['endDate'],
    }
  );
```

---

## 10. Error Handling & Custom Messages

```typescript
// Custom error message
z.string().min(1, 'Không được để trống')
z.string().email('Email không hợp lệ')
z.number().positive('Phải là số dương')
z.string().max(100, { message: 'Tối đa 100 ký tự' })

// Error map toàn cục (custom message cho từng loại lỗi)
const CustomErrorMap: z.ZodErrorMap = (issue, ctx) => {
  if (issue.code === z.ZodIssueCode.invalid_type) {
    if (issue.expected === 'string') return { message: 'Phải là chuỗi ký tự' };
    if (issue.expected === 'number') return { message: 'Phải là số' };
  }
  if (issue.code === z.ZodIssueCode.too_small) {
    return { message: `Giá trị tối thiểu là ${issue.minimum}` };
  }
  if (issue.code === z.ZodIssueCode.too_big) {
    return { message: `Giá trị tối đa là ${issue.maximum}` };
  }
  return { message: ctx.defaultError };
};

z.setErrorMap(CustomErrorMap);

// Parse và xử lý lỗi
const result = UserSchema.safeParse(data);

if (!result.success) {
  // Lấy tất cả lỗi
  const errors = result.error.errors;
  // [{ path: ['email'], message: 'Email không hợp lệ', code: 'invalid_string' }]

  // Flatten lỗi thành object key-value
  const flattened = result.error.flatten();
  // {
  //   formErrors: [],    // lỗi ở root level
  //   fieldErrors: {     // lỗi theo từng field
  //     email: ['Email không hợp lệ'],
  //     name: ['Tối thiểu 2 ký tự'],
  //   }
  // }

  // Format lỗi theo path
  const formatted = result.error.format();
  // { email: { _errors: ['Email không hợp lệ'] } }
}
```

---

## 11. Tích hợp với React Hook Form

```bash
npm install react-hook-form @hookform/resolvers zod
```

```typescript
// schemas/userSchema.ts
import { z } from 'zod';

export const CreateUserSchema = z.object({
  email: z.string().email('Email không hợp lệ'),
  password: z
    .string()
    .min(8, 'Tối thiểu 8 ký tự')
    .regex(/[A-Z]/, 'Phải có chữ hoa')
    .regex(/[0-9]/, 'Phải có số'),
  confirmPassword: z.string(),
  name: z.string().min(2, 'Tối thiểu 2 ký tự').max(50),
  role: z.enum(['admin', 'user']).default('user'),
  age: z.coerce.number().int().min(1).max(120).optional(),
}).refine(
  (data) => data.password === data.confirmPassword,
  {
    message: 'Mật khẩu không khớp',
    path: ['confirmPassword'],
  }
);

export type CreateUserInput = z.infer<typeof CreateUserSchema>;

// components/CreateUserForm.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { CreateUserSchema, CreateUserInput } from '../schemas/userSchema';

function CreateUserForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    reset,
    watch,
    setValue,
  } = useForm<CreateUserInput>({
    resolver: zodResolver(CreateUserSchema),
    defaultValues: {
      role: 'user',
    },
  });

  const onSubmit = async (data: CreateUserInput) => {
    // data đã được validate và typed chính xác
    await createUser(data);
    reset();
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <div>
        <label>Email</label>
        <input {...register('email')} type="email" />
        {errors.email && <p className="text-red-500">{errors.email.message}</p>}
      </div>

      <div>
        <label>Mật khẩu</label>
        <input {...register('password')} type="password" />
        {errors.password && <p className="text-red-500">{errors.password.message}</p>}
      </div>

      <div>
        <label>Xác nhận mật khẩu</label>
        <input {...register('confirmPassword')} type="password" />
        {errors.confirmPassword && (
          <p className="text-red-500">{errors.confirmPassword.message}</p>
        )}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Đang tạo...' : 'Tạo tài khoản'}
      </button>
    </form>
  );
}
```

---

## 12. Tích hợp với Next.js API Route

```typescript
// schemas/api.schema.ts
import { z } from 'zod';

export const CreatePostSchema = z.object({
  title:   z.string().min(1).max(200),
  content: z.string().min(1),
  tags:    z.array(z.string()).max(5).default([]),
  status:  z.enum(['draft', 'published']).default('draft'),
});

export const PaginationSchema = z.object({
  page:   z.coerce.number().int().positive().default(1),
  limit:  z.coerce.number().int().min(1).max(100).default(10),
  search: z.string().trim().optional(),
  sortBy: z.enum(['createdAt', 'title', 'views']).default('createdAt'),
  order:  z.enum(['asc', 'desc']).default('desc'),
});

// app/api/posts/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { CreatePostSchema, PaginationSchema } from '@/schemas/api.schema';

export async function GET(req: NextRequest) {
  // Validate query params
  const params = Object.fromEntries(req.nextUrl.searchParams);
  const result = PaginationSchema.safeParse(params);

  if (!result.success) {
    return NextResponse.json(
      { error: 'Invalid params', details: result.error.flatten().fieldErrors },
      { status: 400 }
    );
  }

  const { page, limit, search, sortBy, order } = result.data;
  // data đã được coerce và có default
  const posts = await postsService.findAll({ page, limit, search, sortBy, order });
  return NextResponse.json(posts);
}

export async function POST(req: NextRequest) {
  const body = await req.json();
  const result = CreatePostSchema.safeParse(body);

  if (!result.success) {
    return NextResponse.json(
      {
        error: 'Validation failed',
        details: result.error.flatten().fieldErrors,
      },
      { status: 422 }
    );
  }

  const post = await postsService.create(result.data);
  return NextResponse.json(post, { status: 201 });
}
```

---

## 13. Schema Composition – Tái sử dụng

```typescript
import { z } from 'zod';

// ── Schemas nhỏ tái sử dụng ──────────────────────
const IdSchema = z.string().uuid();
const EmailSchema = z.string().email().trim().toLowerCase();
const PasswordSchema = z
  .string()
  .min(8, 'Tối thiểu 8 ký tự')
  .regex(/[A-Z]/, 'Phải có chữ hoa')
  .regex(/[0-9]/, 'Phải có số');

const TimestampsSchema = z.object({
  createdAt: z.date().default(() => new Date()),
  updatedAt: z.date().default(() => new Date()),
});

const PaginationSchema = z.object({
  page:  z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().min(1).max(100).default(10),
});

// ── Compose thành schemas lớn hơn ────────────────
const BaseUserSchema = z.object({
  id:    IdSchema,
  email: EmailSchema,
  name:  z.string().min(2).max(50),
  role:  z.enum(['admin', 'user', 'moderator']).default('user'),
});

// Create — không cần id (server tự tạo)
export const CreateUserSchema = BaseUserSchema.omit({ id: true }).extend({
  password: PasswordSchema,
  confirmPassword: z.string(),
}).refine(d => d.password === d.confirmPassword, {
  message: 'Mật khẩu không khớp',
  path: ['confirmPassword'],
});

// Update — tất cả optional trừ id
export const UpdateUserSchema = BaseUserSchema
  .omit({ id: true })
  .partial();

// Response — có timestamps, không có password
export const UserResponseSchema = BaseUserSchema
  .merge(TimestampsSchema)
  .omit({ password: true } as any);

// Login
export const LoginSchema = z.object({
  email:    EmailSchema,
  password: z.string().min(1, 'Mật khẩu không được để trống'),
});

// Query params cho user list
export const GetUsersSchema = PaginationSchema.extend({
  search: z.string().trim().optional(),
  role:   z.enum(['admin', 'user', 'moderator']).optional(),
  active: z.coerce.boolean().optional(),
});

// Export types
export type CreateUserInput   = z.infer<typeof CreateUserSchema>;
export type UpdateUserInput   = z.infer<typeof UpdateUserSchema>;
export type UserResponse      = z.infer<typeof UserResponseSchema>;
export type LoginInput        = z.infer<typeof LoginSchema>;
export type GetUsersQuery     = z.infer<typeof GetUsersSchema>;
```

---

## 14. Advanced Patterns

### Recursive Schema (cây dữ liệu)

```typescript
// Schema tham chiếu chính nó
type Category = {
  id: string;
  name: string;
  children: Category[];
};

const CategorySchema: z.ZodType<Category> = z.lazy(() =>
  z.object({
    id:       z.string(),
    name:     z.string(),
    children: z.array(CategorySchema),
  })
);
```

### Record & Map

```typescript
// Record — object với key động
const MetadataSchema = z.record(z.string());
// { [key: string]: string }

const ScoreSchema = z.record(z.string(), z.number());
// { [key: string]: number }

// Map
const MapSchema = z.map(z.string(), z.number());
// Map<string, number>

// Set
const SetSchema = z.set(z.string());
// Set<string>
```

### z.infer với nested generics

```typescript
// Wrapper type tái sử dụng
function createPaginatedSchema<T extends z.ZodType>(itemSchema: T) {
  return z.object({
    data:       z.array(itemSchema),
    total:      z.number(),
    page:       z.number(),
    totalPages: z.number(),
  });
}

const PaginatedUsersSchema = createPaginatedSchema(UserResponseSchema);
type PaginatedUsers = z.infer<typeof PaginatedUsersSchema>;
// { data: UserResponse[]; total: number; page: number; totalPages: number }

// Dùng
const PaginatedPostsSchema = createPaginatedSchema(PostSchema);
```

### Schema từ TypeScript type (z.ZodType)

```typescript
// Khi đã có interface, muốn Zod validate
interface Config {
  apiUrl:  string;
  timeout: number;
  debug:   boolean;
}

const ConfigSchema: z.ZodType<Config> = z.object({
  apiUrl:  z.string().url(),
  timeout: z.number().positive(),
  debug:   z.boolean().default(false),
});
```

---

## 15. Zod với Environment Variables

```typescript
// lib/env.ts — Validate .env khi app khởi động
import { z } from 'zod';

const EnvSchema = z.object({
  // Server
  DATABASE_URL:           z.string().url(),
  JWT_SECRET:             z.string().min(32, 'JWT secret phải ít nhất 32 ký tự'),
  JWT_EXPIRES_IN:         z.string().default('15m'),
  REFRESH_TOKEN_SECRET:   z.string().min(32),

  // Optional
  REDIS_URL:              z.string().url().optional(),
  CLOUDINARY_URL:         z.string().optional(),

  // Client (Next.js)
  NEXT_PUBLIC_API_URL:    z.string().url(),
  NEXT_PUBLIC_APP_NAME:   z.string().default('My App'),

  // Node env
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT:     z.coerce.number().default(3000),
});

// Validate và throw nếu thiếu
const parsed = EnvSchema.safeParse(process.env);

if (!parsed.success) {
  console.error('❌ Biến môi trường không hợp lệ:');
  console.error(parsed.error.flatten().fieldErrors);
  process.exit(1);  // Crash ngay khi thiếu env
}

export const env = parsed.data;
// env.DATABASE_URL — typed string, không bao giờ undefined

// Dùng
import { env } from '@/lib/env';
const db = new Database(env.DATABASE_URL);
```

---

## 16. Checklist Zod

- [ ] Export `z.infer<typeof Schema>` type cùng với schema — không viết type tay
- [ ] Dùng `safeParse` thay `parse` để handle lỗi không throw
- [ ] Dùng `z.coerce.number()` cho query params (luôn là string từ URL)
- [ ] Dùng `.default()` thay vì check undefined thủ công
- [ ] Validate `.env` với Zod khi app khởi động — crash ngay nếu thiếu
- [ ] Tạo schemas nhỏ tái sử dụng (`EmailSchema`, `PasswordSchema`, `IdSchema`)
- [ ] Dùng `.omit()` / `.pick()` / `.partial()` thay vì viết schema duplicate
- [ ] Dùng `z.discriminatedUnion` thay `z.union` khi có trường phân biệt — nhanh hơn và báo lỗi rõ hơn
- [ ] Dùng `result.error.flatten().fieldErrors` để lấy lỗi theo field
- [ ] Với React Hook Form: dùng `zodResolver` từ `@hookform/resolvers/zod`
- [ ] Đặt tất cả schema vào `schemas/` folder, export types cùng file
