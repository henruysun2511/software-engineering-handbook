# Tổng hợp kiến thức TypeORM cho NestJS

## Mục lục
1. [Cài đặt & cấu hình](#1-cài-đặt--cấu-hình)
2. [Entity](#2-entity)
3. [Quan hệ (Relations)](#3-quan-hệ-relations)
4. [Repository Pattern trong NestJS](#4-repository-pattern-trong-nestjs)
5. [Service & Controller ví dụ đầy đủ](#5-service--controller-ví-dụ-đầy-đủ)
6. [QueryBuilder](#6-querybuilder)
7. [Transaction](#7-transaction)
8. [Migration](#8-migration)
9. [Custom Repository](#9-custom-repository)
10. [Các thao tác thường dùng (Cheatsheet)](#10-các-thao-tác-thường-dùng-cheatsheet)
11. [Best Practices](#11-best-practices)

---

## 1. Cài đặt & cấu hình

### Cài package

```bash
npm install --save @nestjs/typeorm typeorm pg
# pg cho PostgreSQL, dùng mysql2 nếu là MySQL
```

### Cấu hình module gốc (AppModule)

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        type: 'postgres',
        host: config.get('DB_HOST'),
        port: config.get<number>('DB_PORT'),
        username: config.get('DB_USER'),
        password: config.get('DB_PASS'),
        database: config.get('DB_NAME'),
        entities: [__dirname + '/**/*.entity{.ts,.js}'],
        synchronize: false, // KHÔNG dùng true ở production
        migrations: [__dirname + '/migrations/*{.ts,.js}'],
        logging: config.get('NODE_ENV') === 'development',
      }),
    }),
  ],
})
export class AppModule {}
```

> **Lưu ý:** `synchronize: true` tự động đồng bộ schema với entity, tiện cho dev nhưng **cực kỳ nguy hiểm** ở production vì có thể xóa/sửa dữ liệu. Production nên dùng migration.

---

## 2. Entity

Entity là class ánh xạ tới một bảng trong database, dùng decorator để khai báo.

```typescript
// user.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  DeleteDateColumn,
  OneToMany,
  ManyToOne,
  ManyToMany,
  JoinTable,
  Index,
} from 'typeorm';
import { Post } from '../posts/post.entity';
import { Profile } from './profile.entity';
import { Role } from '../roles/role.entity';

export enum UserStatus {
  ACTIVE = 'active',
  INACTIVE = 'inactive',
}

@Entity('users') // tên bảng, nếu bỏ trống sẽ lấy tên class
@Index(['email'], { unique: true })
export class User {
  @PrimaryGeneratedColumn('uuid') // hoặc 'increment' cho auto-increment number
  id: string;

  @Column({ length: 100 })
  fullName: string;

  @Column({ unique: true })
  email: string;

  @Column({ select: false }) // mặc định không trả về khi query
  password: string;

  @Column({
    type: 'enum',
    enum: UserStatus,
    default: UserStatus.ACTIVE,
  })
  status: UserStatus;

  @Column({ type: 'int', default: 0 })
  age: number;

  @Column({ type: 'jsonb', nullable: true })
  metadata: Record<string, any>;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @DeleteDateColumn() // hỗ trợ soft delete
  deletedAt: Date | null;

  // --- Quan hệ ---
  @OneToMany(() => Post, (post) => post.author)
  posts: Post[];

  @OneToOne(() => Profile, (profile) => profile.user, { cascade: true })
  @JoinColumn()
  profile: Profile;

  @ManyToMany(() => Role)
  @JoinTable() // bên chủ sở hữu bảng trung gian
  roles: Role[];
}
```

### Các kiểu cột phổ biến

| Decorator | Mô tả |
|---|---|
| `@PrimaryColumn()` | Khóa chính tự định nghĩa |
| `@PrimaryGeneratedColumn()` | Khóa chính tự tăng (`'uuid'`, `'rowid'`, mặc định số nguyên) |
| `@Column()` | Cột thường, nhận options: `type`, `length`, `nullable`, `default`, `unique`, `select`, `comment` |
| `@CreateDateColumn()` | Tự set khi insert |
| `@UpdateDateColumn()` | Tự cập nhật khi update |
| `@DeleteDateColumn()` | Dùng cho soft delete |
| `@VersionColumn()` | Optimistic locking |
| `@Index()` | Tạo index |

---

## 3. Quan hệ (Relations)

### One-to-Many / Many-to-One

```typescript
// post.entity.ts
@Entity('posts')
export class Post {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  title: string;

  @ManyToOne(() => User, (user) => user.posts, {
    onDelete: 'CASCADE', // hành vi khi User bị xóa
  })
  @JoinColumn({ name: 'author_id' })
  author: User;

  @Column({ name: 'author_id' })
  authorId: string; // giữ FK tường minh để query nhanh không cần join
}
```

### One-to-One

```typescript
// profile.entity.ts
@Entity('profiles')
export class Profile {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ nullable: true })
  bio: string;

  @OneToOne(() => User, (user) => user.profile)
  user: User;
}
```

### Many-to-Many

```typescript
// role.entity.ts
@Entity('roles')
export class Role {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  name: string;

  @ManyToMany(() => User, (user) => user.roles)
  users: User[];
}
```

**Ghi chú:**
- Bên có `@JoinTable()` là chủ sở hữu (owning side), TypeORM sẽ tạo bảng trung gian tại đó.
- `eager: true` trong decorator quan hệ sẽ tự động load quan hệ mỗi khi query entity đó (dùng cẩn thận, dễ gây N+1 hoặc query thừa).
- `lazy: true` trả về Promise cho quan hệ, load khi cần (`await user.posts`).

---

## 4. Repository Pattern trong NestJS

### Đăng ký entity trong feature module

```typescript
// users/users.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';
import { Profile } from './profile.entity';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';

@Module({
  imports: [TypeOrmModule.forFeature([User, Profile])],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [TypeOrmModule], // export để module khác dùng lại Repository<User>
})
export class UsersModule {}
```

`TypeOrmModule.forFeature([...])` đăng ký các entity cho module hiện tại, cho phép inject `Repository<Entity>` bằng `@InjectRepository()`.

---

## 5. Service & Controller ví dụ đầy đủ

### DTO

```typescript
// users/dto/create-user.dto.ts
import { IsEmail, IsString, MinLength } from 'class-validator';

export class CreateUserDto {
  @IsString()
  fullName: string;

  @IsEmail()
  email: string;

  @IsString()
  @MinLength(6)
  password: string;
}
```

```typescript
// users/dto/update-user.dto.ts
import { PartialType } from '@nestjs/mapped-types';
import { CreateUserDto } from './create-user.dto';

export class UpdateUserDto extends PartialType(CreateUserDto) {}
```

### Service

```typescript
// users/users.service.ts
import {
  Injectable,
  NotFoundException,
  ConflictException,
} from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './user.entity';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  async create(dto: CreateUserDto): Promise<User> {
    const existing = await this.userRepository.findOneBy({ email: dto.email });
    if (existing) {
      throw new ConflictException('Email đã tồn tại');
    }

    const user = this.userRepository.create(dto); // tạo instance, chưa lưu DB
    return this.userRepository.save(user); // lưu vào DB
  }

  async findAll(page = 1, limit = 10): Promise<{ data: User[]; total: number }> {
    const [data, total] = await this.userRepository.findAndCount({
      skip: (page - 1) * limit,
      take: limit,
      order: { createdAt: 'DESC' },
      relations: { posts: true, profile: true }, // hoặc ['posts', 'profile']
    });
    return { data, total };
  }

  async findOne(id: string): Promise<User> {
    const user = await this.userRepository.findOne({
      where: { id },
      relations: ['posts'],
    });
    if (!user) throw new NotFoundException('Không tìm thấy user');
    return user;
  }

  async update(id: string, dto: UpdateUserDto): Promise<User> {
    const user = await this.findOne(id);
    Object.assign(user, dto);
    return this.userRepository.save(user);
  }

  async remove(id: string): Promise<void> {
    const result = await this.userRepository.softDelete(id); // dùng delete() nếu xóa cứng
    if (result.affected === 0) {
      throw new NotFoundException('Không tìm thấy user');
    }
  }
}
```

### Controller

```typescript
// users/users.controller.ts
import {
  Controller,
  Get,
  Post,
  Patch,
  Delete,
  Body,
  Param,
  Query,
} from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  create(@Body() dto: CreateUserDto) {
    return this.usersService.create(dto);
  }

  @Get()
  findAll(@Query('page') page?: number, @Query('limit') limit?: number) {
    return this.usersService.findAll(page, limit);
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(id);
  }

  @Patch(':id')
  update(@Param('id') id: string, @Body() dto: UpdateUserDto) {
    return this.usersService.update(id, dto);
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return this.usersService.remove(id);
  }
}
```

---

## 6. QueryBuilder

Dùng khi cần query phức tạp hơn (join có điều kiện, subquery, group by...).

```typescript
async findActiveUsersWithPostCount() {
  return this.userRepository
    .createQueryBuilder('user')
    .leftJoin('user.posts', 'post')
    .loadRelationCountAndMap('user.postCount', 'user.posts')
    .where('user.status = :status', { status: 'active' })
    .andWhere('user.age >= :minAge', { minAge: 18 })
    .orderBy('user.createdAt', 'DESC')
    .take(20)
    .getMany();
}

async searchByName(keyword: string) {
  return this.userRepository
    .createQueryBuilder('user')
    .where('user.fullName ILIKE :keyword', { keyword: `%${keyword}%` })
    .getMany();
}
```

---

## 7. Transaction

### Cách 1: dùng `DataSource` trực tiếp

```typescript
import { DataSource } from 'typeorm';

@Injectable()
export class OrdersService {
  constructor(private readonly dataSource: DataSource) {}

  async createOrder(userId: string, items: OrderItemDto[]) {
    return this.dataSource.transaction(async (manager) => {
      const order = manager.create(Order, { userId });
      await manager.save(order);

      for (const item of items) {
        const orderItem = manager.create(OrderItem, { ...item, orderId: order.id });
        await manager.save(orderItem);
      }

      // Nếu có lỗi ném ra trong block này, toàn bộ transaction sẽ rollback
      return order;
    });
  }
}
```

### Cách 2: dùng QueryRunner (kiểm soát chi tiết hơn)

```typescript
async transferBalance(fromId: string, toId: string, amount: number) {
  const queryRunner = this.dataSource.createQueryRunner();
  await queryRunner.connect();
  await queryRunner.startTransaction();

  try {
    await queryRunner.manager.decrement(Account, { id: fromId }, 'balance', amount);
    await queryRunner.manager.increment(Account, { id: toId }, 'balance', amount);
    await queryRunner.commitTransaction();
  } catch (err) {
    await queryRunner.rollbackTransaction();
    throw err;
  } finally {
    await queryRunner.release();
  }
}
```

---

## 8. Migration

Migration giúp quản lý thay đổi schema có kiểm soát, thay vì dùng `synchronize: true`.

### Cấu hình DataSource riêng cho CLI

```typescript
// data-source.ts
import { DataSource } from 'typeorm';

export const AppDataSource = new DataSource({
  type: 'postgres',
  host: process.env.DB_HOST,
  port: Number(process.env.DB_PORT),
  username: process.env.DB_USER,
  password: process.env.DB_PASS,
  database: process.env.DB_NAME,
  entities: ['src/**/*.entity.ts'],
  migrations: ['src/migrations/*.ts'],
});
```

### Script trong package.json

```json
{
  "scripts": {
    "typeorm": "typeorm-ts-node-commonjs -d src/data-source.ts",
    "migration:generate": "npm run typeorm -- migration:generate src/migrations/Migration",
    "migration:run": "npm run typeorm -- migration:run",
    "migration:revert": "npm run typeorm -- migration:revert"
  }
}
```

### Chạy lệnh

```bash
npm run migration:generate -- src/migrations/AddStatusToUser
npm run migration:run
npm run migration:revert
```

### Ví dụ file migration

```typescript
import { MigrationInterface, QueryRunner, TableColumn } from 'typeorm';

export class AddStatusToUser1699999999999 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.addColumn(
      'users',
      new TableColumn({
        name: 'status',
        type: 'varchar',
        default: `'active'`,
      }),
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropColumn('users', 'status');
  }
}
```

---

## 9. Custom Repository

Từ TypeORM 0.3.x trở lên, cách khuyến nghị là dùng `Repository.extend()` hoặc tạo class kế thừa và inject qua provider tùy chỉnh.

```typescript
// users/user.repository.ts
import { DataSource, Repository } from 'typeorm';
import { Injectable } from '@nestjs/common';
import { User } from './user.entity';

@Injectable()
export class UserRepository extends Repository<User> {
  constructor(private dataSource: DataSource) {
    super(User, dataSource.createEntityManager());
  }

  async findActiveByEmail(email: string): Promise<User | null> {
    return this.findOne({ where: { email, status: 'active' as any } });
  }
}
```

```typescript
// users/users.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService, UserRepository],
  controllers: [UsersController],
})
export class UsersModule {}
```

Sau đó inject `UserRepository` bình thường vào service thay vì `Repository<User>`.

---

## 10. Các thao tác thường dùng (Cheatsheet)

```typescript
// Tìm 1 bản ghi
repo.findOneBy({ id });
repo.findOne({ where: { email }, relations: ['posts'] });

// Tìm nhiều
repo.find({ where: { status: 'active' } });
repo.findBy({ status: 'active' });

// Đếm
repo.count({ where: { status: 'active' } });

// Tạo & lưu
const entity = repo.create(dto);
await repo.save(entity);

// Update theo điều kiện (không load entity trước)
await repo.update({ id }, { status: 'inactive' });

// Xóa cứng / xóa mềm
await repo.delete(id);
await repo.softDelete(id);
await repo.restore(id);

// Upsert
await repo.upsert(
  [{ email: 'a@b.com', fullName: 'A' }],
  { conflictPaths: ['email'] },
);

// Kiểm tra tồn tại
await repo.existsBy({ email });

// Toán tử where nâng cao (import từ 'typeorm')
import { Like, In, Between, MoreThan, IsNull } from 'typeorm';

repo.find({ where: { fullName: Like('%an%') } });
repo.find({ where: { id: In(['1', '2', '3']) } });
repo.find({ where: { age: Between(18, 30) } });
repo.find({ where: { createdAt: MoreThan(new Date('2024-01-01')) } });
repo.find({ where: { deletedAt: IsNull() } });
```

---

## 11. Best Practices

- **Không dùng `synchronize: true` ở production** — luôn dùng migration để kiểm soát thay đổi schema.
- **Tránh N+1 query**: dùng `relations` hoặc `QueryBuilder` với `leftJoinAndSelect` thay vì load quan hệ trong vòng lặp.
- **Hạn chế `eager: true`** trừ khi quan hệ đó luôn cần thiết, vì nó làm chậm mọi query liên quan đến entity.
- **Tách DTO riêng cho input/output**, không expose trực tiếp Entity ra ngoài API (tránh lộ field như `password`).
- **Dùng transaction** cho các thao tác ghi dữ liệu liên quan đến nhiều bảng (ví dụ: tạo đơn hàng + trừ kho).
- **Đặt index** cho các cột thường dùng để filter/sort (`@Index()`), đặc biệt cột dùng trong `WHERE`, `JOIN`.
- **Validate input bằng `class-validator`** ở tầng DTO trước khi vào Service/Repository.
- **Custom Repository** giúp gom logic query phức tạp, tránh Service phình to.
- **Cẩn thận với `cascade: true`** — có thể vô tình insert/update/delete các entity liên quan ngoài ý muốn.
- **Soft delete** (`@DeleteDateColumn`) phù hợp với dữ liệu cần audit/lịch sử; nhớ filter `deletedAt IS NULL` khi cần (TypeORM tự động loại trừ bản ghi đã soft-delete trong query mặc định).
