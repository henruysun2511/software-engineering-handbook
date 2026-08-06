# Prisma ORM – Tổng hợp kiến thức (đặc biệt cho NestJS)

> Docs chính thức: https://www.prisma.io/docs

---

## 1. Prisma là gì?

Prisma là **Next-generation ORM** cho Node.js và TypeScript. Khác với TypeORM hay Sequelize, Prisma dùng **schema file riêng** (`.prisma`) làm source of truth, sau đó tự sinh ra type-safe Prisma Client.

**Prisma gồm 3 phần:**
- **Prisma Schema**: Định nghĩa data model, relations, enums
- **Prisma Client**: Auto-generated, type-safe query builder
- **Prisma Migrate**: Quản lý migration từ schema

**So sánh Prisma vs TypeORM:**

| | Prisma | TypeORM |
|---|---|---|
| Type safety | ✅ Hoàn toàn | ⚠️ Một phần |
| Schema định nghĩa | File `.prisma` riêng | Decorator trên Entity |
| Auto-complete | ✅ Xuất sắc | ⚠️ Trung bình |
| Raw query | ✅ `$queryRaw` | ✅ |
| Migration | ✅ Prisma Migrate | ✅ |
| Learning curve | Thấp | Trung bình |
| Relations | Rõ ràng, tường minh | Phức tạp hơn |

---

## 2. Cài đặt

```bash
npm install prisma @prisma/client
npx prisma init          # tạo prisma/schema.prisma + .env

# Hoặc chỉ định database ngay
npx prisma init --datasource-provider postgresql
npx prisma init --datasource-provider mysql
npx prisma init --datasource-provider sqlite
```

```env
# .env
DATABASE_URL="postgresql://user:password@localhost:5432/mydb?schema=public"
# MySQL:    "mysql://user:password@localhost:3306/mydb"
# SQLite:   "file:./dev.db"
# MongoDB:  "mongodb+srv://user:pass@cluster.mongodb.net/mydb"
```

---

## 3. Prisma Schema – Cú pháp đầy đủ

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ===== ENUM =====
enum Role {
  USER
  ADMIN
  MODERATOR
}

enum OrderStatus {
  PENDING
  PAID
  SHIPPING
  COMPLETED
  CANCELLED
}

// ===== MODEL USER =====
model User {
  id          String    @id @default(uuid())
  email       String    @unique
  password    String?
  displayName String?   @map("display_name")
  avatar      String?
  role        Role      @default(USER)
  isActive    Boolean   @default(true) @map("is_active")
  createdAt   DateTime  @default(now()) @map("created_at")
  updatedAt   DateTime  @updatedAt @map("updated_at")

  // Relations
  profile     Profile?
  orders      Order[]
  posts       Post[]

  @@map("users")               // tên bảng trong DB
  @@index([email])
}

// ===== MODEL PROFILE (1-1) =====
model Profile {
  id        String   @id @default(uuid())
  bio       String?  @db.Text
  phone     String?
  address   String?
  userId    String   @unique @map("user_id")

  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("profiles")
}

// ===== MODEL ORDER (1-N) =====
model Order {
  id          String      @id @default(uuid())
  code        String      @unique @default(cuid())
  status      OrderStatus @default(PENDING)
  totalAmount Decimal     @map("total_amount") @db.Decimal(12, 2)
  note        String?     @db.Text
  userId      String      @map("user_id")
  createdAt   DateTime    @default(now()) @map("created_at")
  updatedAt   DateTime    @updatedAt @map("updated_at")

  user        User        @relation(fields: [userId], references: [id])
  items       OrderItem[]

  @@map("orders")
  @@index([userId])
  @@index([status])
  @@index([createdAt(sort: Desc)])
}

// ===== MODEL ORDER ITEM =====
model OrderItem {
  id        String  @id @default(uuid())
  quantity  Int
  price     Decimal @db.Decimal(10, 2)
  orderId   String  @map("order_id")
  productId String  @map("product_id")

  order     Order   @relation(fields: [orderId], references: [id], onDelete: Cascade)
  product   Product @relation(fields: [productId], references: [id])

  @@map("order_items")
  @@unique([orderId, productId])  // composite unique
}

// ===== MODEL PRODUCT =====
model Product {
  id          String      @id @default(uuid())
  name        String
  slug        String      @unique
  description String?     @db.Text
  price       Decimal     @db.Decimal(10, 2)
  stock       Int         @default(0)
  isPublished Boolean     @default(false) @map("is_published")
  metadata    Json?                        // JSON field
  createdAt   DateTime    @default(now()) @map("created_at")

  orderItems  OrderItem[]
  tags        Tag[]       @relation("ProductTags")  // Many-to-Many

  @@map("products")
  @@fulltext([name, description])   // Full-text search (MySQL)
}

// ===== MODEL TAG (N-N với Product) =====
model Tag {
  id       String    @id @default(uuid())
  name     String    @unique
  products Product[] @relation("ProductTags")

  @@map("tags")
}

// ===== MODEL POST (Self-relation) =====
model Post {
  id        String   @id @default(uuid())
  title     String
  content   String   @db.Text
  authorId  String   @map("author_id")
  parentId  String?  @map("parent_id")   // Self-relation (comment tree)

  author    User     @relation(fields: [authorId], references: [id])
  parent    Post?    @relation("PostReplies", fields: [parentId], references: [id])
  replies   Post[]   @relation("PostReplies")

  @@map("posts")
}
```

---

## 4. Prisma Migrate – Quản lý DB Schema

```bash
# Development
npx prisma migrate dev                        # tạo migration + áp dụng + regenerate client
npx prisma migrate dev --name add_user_table  # đặt tên migration

# Production
npx prisma migrate deploy   # chỉ áp dụng migration đã có (không tạo mới)

# Reset DB (dev only)
npx prisma migrate reset    # xóa toàn bộ DB + chạy lại tất cả migration

# Xem trạng thái
npx prisma migrate status

# Introspect (kéo schema từ DB có sẵn)
npx prisma db pull

# Tạo/cập nhật Prisma Client sau khi sửa schema
npx prisma generate

# Prisma Studio – GUI quản lý data
npx prisma studio
```

---

## 5. Prisma Client – CRUD cơ bản

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// ===== CREATE =====
const user = await prisma.user.create({
  data: {
    email: 'alice@example.com',
    displayName: 'Alice',
    role: 'USER',
  },
});

// Create với relation
const order = await prisma.order.create({
  data: {
    userId: 'user-id',
    totalAmount: 250000,
    items: {
      create: [                        // tạo OrderItem cùng lúc
        { productId: 'prod-1', quantity: 2, price: 100000 },
        { productId: 'prod-2', quantity: 1, price: 50000 },
      ],
    },
  },
  include: { items: true },            // trả về items cùng
});

// createMany
await prisma.product.createMany({
  data: [
    { name: 'Product A', slug: 'product-a', price: 100 },
    { name: 'Product B', slug: 'product-b', price: 200 },
  ],
  skipDuplicates: true,
});

// ===== READ =====
// findUnique – tìm theo @unique field
const user = await prisma.user.findUnique({
  where: { email: 'alice@example.com' },
  select: { id: true, email: true, displayName: true },  // chọn field
});

// findFirst – tìm bản ghi đầu tiên khớp
const order = await prisma.order.findFirst({
  where: { userId: 'user-id', status: 'PENDING' },
  orderBy: { createdAt: 'desc' },
});

// findMany – danh sách
const orders = await prisma.order.findMany({
  where: {
    userId: 'user-id',
    status: { in: ['PENDING', 'PAID'] },
    totalAmount: { gte: 100000 },
    createdAt: { gte: new Date('2024-01-01') },
  },
  include: {
    items: {
      include: { product: true },    // nested include
    },
    user: {
      select: { email: true, displayName: true },
    },
  },
  orderBy: [
    { createdAt: 'desc' },
    { totalAmount: 'asc' },
  ],
  skip: 0,      // offset
  take: 10,     // limit
});

// ===== UPDATE =====
const updated = await prisma.user.update({
  where: { id: 'user-id' },
  data: {
    displayName: 'Alice Updated',
    updatedAt: new Date(),
  },
});

// updateMany
await prisma.order.updateMany({
  where: { status: 'PENDING', createdAt: { lt: subDays(new Date(), 3) } },
  data: { status: 'CANCELLED' },
});

// upsert – update nếu tồn tại, create nếu chưa có
const profile = await prisma.profile.upsert({
  where: { userId: 'user-id' },
  update: { bio: 'Updated bio' },
  create: { userId: 'user-id', bio: 'New bio' },
});

// ===== DELETE =====
await prisma.user.delete({ where: { id: 'user-id' } });

await prisma.order.deleteMany({
  where: { status: 'CANCELLED', createdAt: { lt: subMonths(new Date(), 6) } },
});
```

---

## 6. Where Filters – Bộ lọc nâng cao

```typescript
// String filters
where: {
  name: { contains: 'alice', mode: 'insensitive' },  // ILIKE '%alice%'
  email: { startsWith: 'alice' },
  slug: { endsWith: '-sale' },
  email: { not: null },
}

// Number / Date filters
where: {
  price: { gte: 100, lte: 500 },    // BETWEEN
  stock: { gt: 0 },
  createdAt: { gte: new Date('2024-01-01'), lt: new Date('2025-01-01') },
}

// Array filters
where: {
  status: { in: ['PENDING', 'PAID'] },
  role: { notIn: ['ADMIN'] },
}

// Relation filters
where: {
  // Có ít nhất 1 order PAID
  orders: { some: { status: 'PAID' } },
  // Tất cả orders đều COMPLETED
  orders: { every: { status: 'COMPLETED' } },
  // Không có order nào
  orders: { none: {} },
  // Filter theo related record
  profile: { is: { phone: { not: null } } },
}

// Logic operators
where: {
  OR: [
    { email: { contains: 'alice' } },
    { displayName: { contains: 'Alice' } },
  ],
  AND: [
    { isActive: true },
    { role: 'USER' },
  ],
  NOT: { status: 'CANCELLED' },
}
```

---

## 7. Select vs Include

```typescript
// SELECT: chọn chính xác field nào trả về
const user = await prisma.user.findUnique({
  where: { id },
  select: {
    id: true,
    email: true,
    profile: {
      select: { bio: true, phone: true },
    },
  },
  // password sẽ KHÔNG có trong kết quả
});

// INCLUDE: lấy tất cả field + thêm relation
const user = await prisma.user.findUnique({
  where: { id },
  include: {
    profile: true,
    orders: {
      where: { status: 'PAID' },
      take: 5,
      orderBy: { createdAt: 'desc' },
      include: { items: true },
    },
  },
});

// Không dùng select và include cùng nhau trên cùng 1 level
```

---

## 8. Pagination

```typescript
// Offset-based pagination
async findAll(page: number, limit: number) {
  const skip = (page - 1) * limit;

  const [data, total] = await prisma.$transaction([
    prisma.user.findMany({ skip, take: limit, orderBy: { createdAt: 'desc' } }),
    prisma.user.count(),
  ]);

  return {
    data,
    total,
    page,
    limit,
    totalPages: Math.ceil(total / limit),
  };
}

// Cursor-based pagination (tốt hơn cho real-time feed)
async findAfterCursor(cursor?: string, take = 10) {
  return prisma.post.findMany({
    take,
    skip: cursor ? 1 : 0,          // bỏ qua cursor item
    cursor: cursor ? { id: cursor } : undefined,
    orderBy: { createdAt: 'desc' },
    include: { author: { select: { displayName: true } } },
  });
}
```

---

## 9. Transaction

```typescript
// Cách 1: $transaction array (tất cả thành công hoặc tất cả rollback)
const [user, profile] = await prisma.$transaction([
  prisma.user.create({ data: { email: 'a@b.com' } }),
  prisma.profile.create({ data: { userId: '...', bio: 'Hello' } }),
]);

// Cách 2: Interactive transaction (có logic điều kiện)
const result = await prisma.$transaction(async (tx) => {
  // Kiểm tra stock
  const product = await tx.product.findUnique({ where: { id: productId } });
  if (product.stock < quantity) throw new Error('Hết hàng');

  // Trừ stock
  await tx.product.update({
    where: { id: productId },
    data: { stock: { decrement: quantity } },
  });

  // Tạo order
  const order = await tx.order.create({
    data: { userId, totalAmount: product.price * quantity },
  });

  return order;
}, {
  maxWait: 5000,   // tối đa 5s chờ transaction bắt đầu
  timeout: 10000,  // tối đa 10s cho toàn bộ transaction
});
```

---

## 10. Aggregation & GroupBy

```typescript
// Aggregate
const stats = await prisma.order.aggregate({
  where: { status: 'PAID' },
  _count: { id: true },
  _sum: { totalAmount: true },
  _avg: { totalAmount: true },
  _min: { totalAmount: true },
  _max: { totalAmount: true },
});
// stats._sum.totalAmount = tổng doanh thu

// GroupBy
const revenueByStatus = await prisma.order.groupBy({
  by: ['status'],
  _count: { id: true },
  _sum: { totalAmount: true },
  having: {
    totalAmount: { _sum: { gt: 0 } },  // filter sau khi group
  },
  orderBy: { _sum: { totalAmount: 'desc' } },
});

// Count
const activeUsers = await prisma.user.count({
  where: { isActive: true, role: 'USER' },
});
```

---

## 11. Raw Query

```typescript
// $queryRaw – SELECT trả về mảng rows (type-safe với generic)
const users = await prisma.$queryRaw<{ id: string; email: string }[]>`
  SELECT id, email FROM users
  WHERE created_at > ${new Date('2024-01-01')}
  LIMIT ${10}
`;

// $executeRaw – INSERT/UPDATE/DELETE trả về số row bị ảnh hưởng
const count = await prisma.$executeRaw`
  UPDATE products SET stock = stock - ${1}
  WHERE id = ${productId} AND stock > 0
`;

// Dùng Prisma.sql để tạo dynamic query an toàn
import { Prisma } from '@prisma/client';

const searchTerm = '%alice%';
const users = await prisma.$queryRaw`
  SELECT * FROM users WHERE email LIKE ${searchTerm}
`;
```

---

## 12. Middleware & Soft Delete

```typescript
// prisma.middleware.ts – Logging
prisma.$use(async (params, next) => {
  const start = Date.now();
  const result = await next(params);
  const duration = Date.now() - start;

  if (duration > 1000) {
    console.warn(`Slow query: ${params.model}.${params.action} - ${duration}ms`);
  }

  return result;
});

// Soft Delete middleware
prisma.$use(async (params, next) => {
  // Bỏ qua các record đã xóa mềm
  if (params.model === 'User') {
    if (params.action === 'findFirst' || params.action === 'findMany') {
      params.args.where = { ...params.args.where, deletedAt: null };
    }
    if (params.action === 'delete') {
      params.action = 'update';
      params.args.data = { deletedAt: new Date() };
    }
    if (params.action === 'deleteMany') {
      params.action = 'updateMany';
      params.args.data = { deletedAt: new Date() };
    }
  }
  return next(params);
});
```

---

## 13. Prisma trong NestJS

### 13.1. PrismaService

```typescript
// prisma/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy, Logger } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(PrismaService.name);

  constructor() {
    super({
      log: [
        { emit: 'event', level: 'query' },
        { emit: 'stdout', level: 'error' },
        { emit: 'stdout', level: 'warn' },
      ],
    });
  }

  async onModuleInit() {
    await this.$connect();
    this.logger.log('Prisma connected');

    // Log slow queries
    this.$on('query' as never, (e: any) => {
      if (e.duration > 500) {
        this.logger.warn(`Slow query (${e.duration}ms): ${e.query}`);
      }
    });
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }

  // Helper: Soft Delete
  async softDelete(model: string, id: string) {
    return (this as any)[model].update({
      where: { id },
      data: { deletedAt: new Date() },
    });
  }

  // Helper: Paginate
  async paginate<T>(
    model: keyof PrismaClient,
    args: { where?: any; orderBy?: any; include?: any; select?: any },
    page: number,
    limit: number,
  ): Promise<{ data: T[]; total: number; page: number; totalPages: number }> {
    const skip = (page - 1) * limit;
    const [data, total] = await this.$transaction([
      (this[model] as any).findMany({ ...args, skip, take: limit }),
      (this[model] as any).count({ where: args.where }),
    ]);
    return { data, total, page, totalPages: Math.ceil(total / limit) };
  }
}
```

### 13.2. PrismaModule

```typescript
// prisma/prisma.module.ts
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()   // inject PrismaService ở mọi nơi không cần import lại
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

### 13.3. Dùng trong Service

```typescript
// users/users.service.ts
import { Injectable, NotFoundException, ConflictException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { Prisma, User } from '@prisma/client';

@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  async findAll(params: {
    page?: number;
    limit?: number;
    search?: string;
    role?: string;
  }) {
    const { page = 1, limit = 10, search, role } = params;

    const where: Prisma.UserWhereInput = {
      deletedAt: null,
      ...(search && {
        OR: [
          { email: { contains: search, mode: 'insensitive' } },
          { displayName: { contains: search, mode: 'insensitive' } },
        ],
      }),
      ...(role && { role: role as any }),
    };

    const [data, total] = await this.prisma.$transaction([
      this.prisma.user.findMany({
        where,
        select: {
          id: true, email: true, displayName: true, role: true, createdAt: true,
        },
        orderBy: { createdAt: 'desc' },
        skip: (page - 1) * limit,
        take: limit,
      }),
      this.prisma.user.count({ where }),
    ]);

    return { data, total, page, totalPages: Math.ceil(total / limit) };
  }

  async findById(id: string): Promise<User> {
    const user = await this.prisma.user.findUnique({
      where: { id },
      include: { profile: true },
    });
    if (!user) throw new NotFoundException(`User ${id} không tồn tại`);
    return user;
  }

  async create(data: Prisma.UserCreateInput): Promise<User> {
    try {
      return await this.prisma.user.create({ data });
    } catch (e) {
      if (e instanceof Prisma.PrismaClientKnownRequestError) {
        if (e.code === 'P2002') {
          throw new ConflictException('Email đã tồn tại');
        }
      }
      throw e;
    }
  }

  async update(id: string, data: Prisma.UserUpdateInput): Promise<User> {
    try {
      return await this.prisma.user.update({ where: { id }, data });
    } catch (e) {
      if (e instanceof Prisma.PrismaClientKnownRequestError && e.code === 'P2025') {
        throw new NotFoundException(`User ${id} không tồn tại`);
      }
      throw e;
    }
  }
}
```

---

## 14. Error Handling – Mã lỗi Prisma

```typescript
import { Prisma } from '@prisma/client';

// Các error code quan trọng
// P2000 – giá trị quá dài cho column
// P2001 – record không tồn tại trong WHERE
// P2002 – unique constraint vi phạm
// P2003 – foreign key constraint vi phạm
// P2004 – constraint vi phạm
// P2025 – record không tìm thấy (update/delete)

// Exception filter xử lý Prisma errors
@Catch(Prisma.PrismaClientKnownRequestError)
export class PrismaExceptionFilter implements ExceptionFilter {
  catch(exception: Prisma.PrismaClientKnownRequestError, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();

    const errorMap = {
      P2002: { status: 409, message: 'Dữ liệu đã tồn tại (unique constraint)' },
      P2025: { status: 404, message: 'Không tìm thấy bản ghi' },
      P2003: { status: 400, message: 'Vi phạm ràng buộc khóa ngoại' },
    };

    const error = errorMap[exception.code] ?? { status: 500, message: 'Database error' };
    res.status(error.status).json({ statusCode: error.status, message: error.message });
  }
}

// main.ts
app.useGlobalFilters(new PrismaExceptionFilter());
```

---

## 15. Seeding Data

```typescript
// prisma/seed.ts
import { PrismaClient } from '@prisma/client';
import * as bcrypt from 'bcrypt';

const prisma = new PrismaClient();

async function main() {
  // Upsert để seed idempotent (chạy nhiều lần không bị lỗi)
  const admin = await prisma.user.upsert({
    where: { email: 'admin@example.com' },
    update: {},
    create: {
      email: 'admin@example.com',
      password: await bcrypt.hash('Admin@123', 10),
      displayName: 'Administrator',
      role: 'ADMIN',
      isEmailVerified: true,
    },
  });

  console.log('Seeded admin:', admin.email);
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

```json
// package.json
{
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
}
```

```bash
npx prisma db seed
```

---

## 16. Type Utilities

```typescript
import { Prisma } from '@prisma/client';

// Lấy type của input/output từ Prisma
type UserCreateInput = Prisma.UserCreateInput;
type UserUpdateInput = Prisma.UserUpdateInput;
type UserWhereInput = Prisma.UserWhereInput;
type UserOrderByInput = Prisma.UserOrderByWithRelationInput;

// Lấy type trả về với include/select
type UserWithProfile = Prisma.UserGetPayload<{
  include: { profile: true };
}>;

type UserBasic = Prisma.UserGetPayload<{
  select: { id: true; email: true; displayName: true };
}>;

// Dùng trong Service
async findById(id: string): Promise<UserWithProfile> {
  return this.prisma.user.findUnique({
    where: { id },
    include: { profile: true },
  });
}
```

---

## 17. Best Practices & Checklist

### Performance
- [ ] Luôn dùng `select` hoặc `include` có chọn lọc — tránh `SELECT *`
- [ ] Đặt `@@index` cho các field thường dùng trong `where` và `orderBy`
- [ ] Dùng cursor-based pagination cho feed real-time, offset cho admin table
- [ ] Batch operations với `createMany`, `updateMany`, `deleteMany`
- [ ] Dùng `$transaction` cho các thao tác cần atomic
- [ ] Log và monitor slow queries (> 500ms)

### Security
- [ ] Không expose Prisma Client trực tiếp ra controller — luôn qua Service
- [ ] Dùng `select` để ẩn field nhạy cảm (password, token)
- [ ] Validate và sanitize input trước khi truyền vào Prisma
- [ ] Dùng `Prisma.UserWhereInput` type thay vì `any`

### Code Quality
- [ ] `PrismaService` extend `PrismaClient` và implement `OnModuleInit`
- [ ] `PrismaModule` là `@Global()` để inject khắp nơi
- [ ] Xử lý `PrismaClientKnownRequestError` với mã lỗi cụ thể
- [ ] Seed data dùng `upsert` để idempotent
- [ ] Tên field trong schema dùng camelCase, `@map` sang snake_case cho DB
- [ ] Tên bảng dùng `@@map("snake_case_plural")`
