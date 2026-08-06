# MongoDB – Tổng hợp kiến thức (đặc biệt cho NestJS)

> Docs: https://docs.nestjs.com/techniques/mongodb | https://mongoosejs.com/docs

---

## 1. MongoDB là gì?

MongoDB là **NoSQL document database** — lưu dữ liệu dạng JSON (BSON) thay vì bảng quan hệ. Mỗi document có thể có cấu trúc khác nhau (schema-less), nhưng trong thực tế thường dùng Mongoose để định nghĩa schema.

**Khi nào dùng MongoDB:**
- Dữ liệu phi cấu trúc hoặc thay đổi schema thường xuyên
- Cần scale horizontal dễ dàng
- Dữ liệu dạng lồng nhau (nested) tự nhiên (posts + comments + likes)
- Real-time analytics, logging, event sourcing
- Catalog sản phẩm với attributes đa dạng

**Các khái niệm cơ bản:**

| SQL | MongoDB |
|---|---|
| Database | Database |
| Table | Collection |
| Row | Document |
| Column | Field |
| JOIN | `$lookup` / Populate |
| Index | Index |
| Transaction | Multi-document Transaction |

---

## 2. Cài đặt NestJS + Mongoose

```bash
npm install @nestjs/mongoose mongoose
npm install -D @types/mongoose
```

```typescript
// app.module.ts
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (config: ConfigService) => ({
        uri: config.get<string>('MONGODB_URI'),
        // Options
        dbName: 'myapp',
        maxPoolSize: 10,
        serverSelectionTimeoutMS: 5000,
        socketTimeoutMS: 45000,
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

```env
# .env
MONGODB_URI=mongodb://localhost:27017/myapp
# MongoDB Atlas:
MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/myapp?retryWrites=true&w=majority
```

---

## 3. Schema & Model

### 3.1. Định nghĩa Schema với Decorator

```typescript
// users/schemas/user.schema.ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document, Types } from 'mongoose';

export type UserDocument = User & Document;

export enum UserRole {
  USER = 'user',
  ADMIN = 'admin',
}

@Schema({
  timestamps: true,           // tự động thêm createdAt, updatedAt
  collection: 'users',        // tên collection (mặc định = tên model lowercase + 's')
  versionKey: false,          // bỏ __v field
  toJSON: {
    virtuals: true,           // include virtual fields khi JSON.stringify
    transform: (_, ret) => {
      delete ret.__id;
      delete ret.password;    // ẩn password khỏi JSON output
      return ret;
    },
  },
})
export class User {
  @Prop({ required: true, unique: true, lowercase: true, trim: true })
  email: string;

  @Prop({ select: false })    // không trả về password khi query
  password: string;

  @Prop({ trim: true })
  displayName: string;

  @Prop()
  avatar: string;

  @Prop({ type: String, enum: UserRole, default: UserRole.USER })
  role: UserRole;

  @Prop({ default: true })
  isActive: boolean;

  @Prop({ default: false })
  isEmailVerified: boolean;

  // Nested object
  @Prop({
    type: {
      street: String,
      city: String,
      country: { type: String, default: 'VN' },
    },
    _id: false,               // không tạo _id cho sub-document
  })
  address: {
    street: string;
    city: string;
    country: string;
  };

  // Array of strings
  @Prop({ type: [String], default: [] })
  tags: string[];

  // Reference đến collection khác (populate)
  @Prop({ type: Types.ObjectId, ref: 'Category' })
  categoryId: Types.ObjectId;

  // Array of references
  @Prop({ type: [{ type: Types.ObjectId, ref: 'Product' }] })
  wishlist: Types.ObjectId[];

  // Soft delete
  @Prop({ default: null })
  deletedAt: Date | null;
}

export const UserSchema = SchemaFactory.createForClass(User);

// Index
UserSchema.index({ email: 1 });
UserSchema.index({ createdAt: -1 });
UserSchema.index({ role: 1, isActive: 1 });
UserSchema.index({ displayName: 'text', email: 'text' }); // text search

// Virtual field
UserSchema.virtual('fullName').get(function () {
  return `${this.firstName} ${this.lastName}`;
});

// Pre-save hook
UserSchema.pre('save', async function (next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

// Pre-find hook (tự động filter soft deleted)
UserSchema.pre(/^find/, function () {
  (this as any).where({ deletedAt: null });
});
```

### 3.2. Schema lồng nhau (Embedded Document)

```typescript
// schemas/comment.schema.ts
@Schema({ _id: true, timestamps: true })
export class Comment {
  @Prop({ required: true })
  content: string;

  @Prop({ type: Types.ObjectId, ref: 'User', required: true })
  authorId: Types.ObjectId;
}
export const CommentSchema = SchemaFactory.createForClass(Comment);

// schemas/post.schema.ts
@Schema({ timestamps: true, versionKey: false })
export class Post {
  @Prop({ required: true })
  title: string;

  @Prop({ required: true })
  content: string;

  @Prop({ type: Types.ObjectId, ref: 'User' })
  authorId: Types.ObjectId;

  // Embed array of sub-documents
  @Prop({ type: [CommentSchema], default: [] })
  comments: Comment[];

  @Prop({ default: 0 })
  viewCount: number;

  @Prop({ type: [String], default: [] })
  tags: string[];
}
export const PostSchema = SchemaFactory.createForClass(Post);
```

### 3.3. Đăng ký Model trong Module

```typescript
// users/users.module.ts
import { MongooseModule } from '@nestjs/mongoose';
import { User, UserSchema } from './schemas/user.schema';
import { Post, PostSchema } from './schemas/post.schema';

@Module({
  imports: [
    MongooseModule.forFeature([
      { name: User.name, schema: UserSchema },
      { name: Post.name, schema: PostSchema },
    ]),
  ],
  providers: [UsersService],
  controllers: [UsersController],
  exports: [MongooseModule],
})
export class UsersModule {}
```

---

## 4. CRUD cơ bản

```typescript
// users/users.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model, Types, FilterQuery } from 'mongoose';
import { User, UserDocument } from './schemas/user.schema';

@Injectable()
export class UsersService {
  constructor(
    @InjectModel(User.name) private userModel: Model<UserDocument>,
  ) {}

  // ===== CREATE =====
  async create(data: Partial<User>): Promise<UserDocument> {
    const user = new this.userModel(data);
    return user.save();
  }

  // Hoặc dùng .create()
  async createBulk(data: Partial<User>[]): Promise<UserDocument[]> {
    return this.userModel.insertMany(data);
  }

  // ===== READ =====
  async findAll(): Promise<UserDocument[]> {
    return this.userModel.find().exec();
  }

  async findById(id: string): Promise<UserDocument> {
    if (!Types.ObjectId.isValid(id)) throw new NotFoundException('ID không hợp lệ');

    const user = await this.userModel.findById(id).exec();
    if (!user) throw new NotFoundException(`User ${id} không tồn tại`);
    return user;
  }

  async findByEmail(email: string): Promise<UserDocument | null> {
    return this.userModel.findOne({ email: email.toLowerCase() }).select('+password').exec();
    // .select('+password') → lấy password dù có { select: false }
  }

  // ===== UPDATE =====
  async update(id: string, data: Partial<User>): Promise<UserDocument> {
    const user = await this.userModel.findByIdAndUpdate(
      id,
      { $set: data },
      { new: true, runValidators: true },  // new: trả về document mới sau update
    ).exec();
    if (!user) throw new NotFoundException(`User ${id} không tồn tại`);
    return user;
  }

  // ===== DELETE =====
  async remove(id: string): Promise<void> {
    const result = await this.userModel.findByIdAndDelete(id).exec();
    if (!result) throw new NotFoundException(`User ${id} không tồn tại`);
  }

  // Soft delete
  async softDelete(id: string): Promise<UserDocument> {
    return this.userModel.findByIdAndUpdate(
      id,
      { $set: { deletedAt: new Date() } },
      { new: true },
    ).exec();
  }
}
```

---

## 5. Query nâng cao

### 5.1. Filter, Sort, Pagination

```typescript
async findWithFilter(params: {
  page?: number;
  limit?: number;
  search?: string;
  role?: string;
  isActive?: boolean;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
}) {
  const {
    page = 1, limit = 10, search, role, isActive,
    sortBy = 'createdAt', sortOrder = 'desc',
  } = params;

  const filter: FilterQuery<UserDocument> = { deletedAt: null };

  if (search) {
    filter.$or = [
      { email: { $regex: search, $options: 'i' } },
      { displayName: { $regex: search, $options: 'i' } },
    ];
  }
  if (role) filter.role = role;
  if (isActive !== undefined) filter.isActive = isActive;

  const sort = { [sortBy]: sortOrder === 'asc' ? 1 : -1 };

  const [data, total] = await Promise.all([
    this.userModel
      .find(filter)
      .sort(sort)
      .skip((page - 1) * limit)
      .limit(limit)
      .select('-password')
      .lean()           // trả về plain object, nhanh hơn Mongoose Document
      .exec(),
    this.userModel.countDocuments(filter),
  ]);

  return { data, total, page, limit, totalPages: Math.ceil(total / limit) };
}
```

### 5.2. Update Operators

```typescript
// $set – cập nhật field cụ thể
await this.userModel.updateOne({ _id: id }, { $set: { displayName: 'Alice' } });

// $inc – tăng/giảm số
await this.postModel.updateOne({ _id: id }, { $inc: { viewCount: 1 } });

// $push – thêm vào array
await this.userModel.updateOne(
  { _id: userId },
  { $push: { wishlist: productId } },
);

// $addToSet – thêm nếu chưa có (unique array)
await this.userModel.updateOne(
  { _id: userId },
  { $addToSet: { tags: 'vip' } },
);

// $pull – xóa khỏi array theo condition
await this.userModel.updateOne(
  { _id: userId },
  { $pull: { wishlist: productId } },
);

// $unset – xóa field
await this.userModel.updateOne({ _id: id }, { $unset: { resetToken: '' } });

// Cập nhật element trong array (array filter)
await this.postModel.updateOne(
  { _id: postId, 'comments._id': commentId },
  { $set: { 'comments.$.content': newContent } },
);
```

### 5.3. Populate (tương tự JOIN)

```typescript
// Populate đơn giản
const post = await this.postModel
  .findById(id)
  .populate('authorId', 'displayName avatar email')  // chọn field từ User
  .exec();

// Populate lồng nhau
const post = await this.postModel
  .findById(id)
  .populate({
    path: 'authorId',
    select: 'displayName avatar',
    populate: {                          // populate bên trong populate
      path: 'categoryId',
      select: 'name',
    },
  })
  .populate('comments.authorId', 'displayName')  // populate trong array
  .exec();

// Populate có filter
const user = await this.userModel
  .findById(id)
  .populate({
    path: 'orders',
    match: { status: 'paid' },           // chỉ lấy orders đã paid
    options: { sort: { createdAt: -1 }, limit: 10 },
    select: 'totalAmount status createdAt',
  })
  .exec();
```

---

## 6. Aggregation Pipeline

Aggregation là tính năng mạnh nhất của MongoDB — xử lý dữ liệu theo pipeline nhiều stage.

```typescript
// Doanh thu theo tháng
const revenueByMonth = await this.orderModel.aggregate([
  // Stage 1: Filter
  { $match: { status: 'paid', createdAt: { $gte: new Date('2024-01-01') } } },

  // Stage 2: Nhóm theo tháng
  {
    $group: {
      _id: {
        year: { $year: '$createdAt' },
        month: { $month: '$createdAt' },
      },
      totalRevenue: { $sum: '$totalAmount' },
      orderCount: { $sum: 1 },
      avgOrder: { $avg: '$totalAmount' },
    },
  },

  // Stage 3: Sort
  { $sort: { '_id.year': -1, '_id.month': -1 } },

  // Stage 4: Reshape output
  {
    $project: {
      _id: 0,
      year: '$_id.year',
      month: '$_id.month',
      totalRevenue: 1,
      orderCount: 1,
      avgOrder: { $round: ['$avgOrder', 0] },
    },
  },
]);

// $lookup (tương tự JOIN)
const ordersWithUser = await this.orderModel.aggregate([
  { $match: { status: 'pending' } },
  {
    $lookup: {
      from: 'users',          // tên collection (không phải model name)
      localField: 'userId',
      foreignField: '_id',
      as: 'user',
      pipeline: [             // pipeline bên trong lookup
        { $project: { displayName: 1, email: 1, avatar: 1 } },
      ],
    },
  },
  { $unwind: '$user' },       // biến array [user] thành object user
]);

// $facet – nhiều aggregation song song
const dashboard = await this.orderModel.aggregate([
  { $match: { createdAt: { $gte: startOfMonth } } },
  {
    $facet: {
      totalRevenue: [{ $group: { _id: null, total: { $sum: '$totalAmount' } } }],
      byStatus: [{ $group: { _id: '$status', count: { $sum: 1 } } }],
      recentOrders: [{ $sort: { createdAt: -1 } }, { $limit: 5 }],
    },
  },
]);

// $unwind + group (flatten array và aggregate)
const tagStats = await this.postModel.aggregate([
  { $unwind: '$tags' },
  { $group: { _id: '$tags', count: { $sum: 1 } } },
  { $sort: { count: -1 } },
  { $limit: 10 },
]);
```

---

## 7. Transaction (Multi-document)

```typescript
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class OrdersService {
  constructor(
    @InjectConnection() private connection: Connection,
    @InjectModel(Order.name) private orderModel: Model<OrderDocument>,
    @InjectModel(Product.name) private productModel: Model<ProductDocument>,
  ) {}

  async createOrder(userId: string, items: OrderItemDto[]) {
    const session = await this.connection.startSession();

    try {
      session.startTransaction();

      // Kiểm tra và trừ stock
      for (const item of items) {
        const result = await this.productModel.findOneAndUpdate(
          { _id: item.productId, stock: { $gte: item.quantity } },
          { $inc: { stock: -item.quantity } },
          { session, new: true },
        );
        if (!result) throw new BadRequestException(`Sản phẩm ${item.productId} hết hàng`);
      }

      // Tạo order
      const [order] = await this.orderModel.create(
        [{ userId, items, status: 'pending' }],
        { session },
      );

      await session.commitTransaction();
      return order;
    } catch (error) {
      await session.abortTransaction();
      throw error;
    } finally {
      session.endSession();
    }
  }
}
```

---

## 8. Indexes

```typescript
// Trong Schema
UserSchema.index({ email: 1 }, { unique: true });
UserSchema.index({ createdAt: -1 });
UserSchema.index({ role: 1, isActive: 1 });          // compound index
UserSchema.index({ displayName: 'text', bio: 'text' }); // text index
UserSchema.index({ location: '2dsphere' });            // geo index
UserSchema.index(
  { createdAt: 1 },
  { expireAfterSeconds: 86400 },                       // TTL index — tự xóa sau 24h
);

// Dùng @Prop decorator
@Prop({ index: true })
slug: string;

@Prop({ unique: true })
email: string;

// Compound index qua SchemaFactory
@Schema()
export class Session {
  @Prop({ required: true })
  userId: string;

  @Prop({ required: true })
  token: string;

  @Prop()
  expiresAt: Date;
}

// Ngoài class
SessionSchema.index({ userId: 1, token: 1 });
SessionSchema.index({ expiresAt: 1 }, { expireAfterSeconds: 0 }); // xóa khi expiresAt tới
```

---

## 9. Text Search & Geo Query

```typescript
// Text Search
PostSchema.index({ title: 'text', content: 'text', tags: 'text' });

async searchPosts(query: string) {
  return this.postModel.find(
    { $text: { $search: query } },
    { score: { $meta: 'textScore' } },  // tính relevance score
  )
  .sort({ score: { $meta: 'textScore' } })
  .limit(10)
  .exec();
}

// Geo Query (tìm địa điểm gần)
@Schema()
export class Place {
  @Prop()
  name: string;

  @Prop({
    type: { type: String, enum: ['Point'], default: 'Point' },
    coordinates: { type: [Number] },   // [longitude, latitude]
  })
  location: { type: string; coordinates: number[] };
}
PlaceSchema.index({ location: '2dsphere' });

async findNearby(lng: number, lat: number, radiusMeters = 5000) {
  return this.placeModel.find({
    location: {
      $near: {
        $geometry: { type: 'Point', coordinates: [lng, lat] },
        $maxDistance: radiusMeters,
      },
    },
  }).exec();
}
```

---

## 10. Change Streams (Real-time)

Change Streams cho phép lắng nghe thay đổi trong collection theo thời gian thực — tích hợp tốt với Socket.IO.

```typescript
// change-stream.service.ts
@Injectable()
export class ChangeStreamService implements OnModuleInit, OnModuleDestroy {
  private changeStream: mongoose.ChangeStream;

  constructor(
    @InjectModel(Order.name) private orderModel: Model<OrderDocument>,
    private notificationGateway: NotificationGateway,
  ) {}

  onModuleInit() {
    this.changeStream = this.orderModel.watch(
      [{ $match: { 'fullDocument.status': 'paid' } }],  // filter
      { fullDocument: 'updateLookup' },
    );

    this.changeStream.on('change', (change) => {
      if (change.operationType === 'update') {
        const order = change.fullDocument;
        // Push real-time notification
        this.notificationGateway.sendToUser(order.userId, {
          type: 'ORDER_PAID',
          orderId: order._id,
        });
      }
    });

    this.changeStream.on('error', (err) => {
      console.error('Change stream error:', err);
    });
  }

  onModuleDestroy() {
    this.changeStream?.close();
  }
}
```

---

## 11. Discriminators (Polymorphism)

Dùng khi nhiều loại document có chung base schema nhưng có thêm field riêng.

```typescript
// base schema
@Schema({ discriminatorKey: 'type', timestamps: true })
export class Notification {
  @Prop({ required: true })
  userId: string;

  @Prop({ default: false })
  isRead: boolean;

  @Prop({ required: true })
  type: string;
}
export const NotificationSchema = SchemaFactory.createForClass(Notification);

// sub schema
@Schema()
export class OrderNotification {
  @Prop({ required: true })
  orderId: string;

  @Prop({ required: true })
  orderStatus: string;
}

// Đăng ký discriminator
MongooseModule.forFeature([
  { name: Notification.name, schema: NotificationSchema },
  {
    name: 'OrderNotification',
    schema: SchemaFactory.createForClass(OrderNotification),
    discriminators: [{ name: 'order', schema: SchemaFactory.createForClass(OrderNotification) }],
  },
]);
```

---

## 12. Validation với class-validator + DTO

```typescript
// dto/create-user.dto.ts
import { IsEmail, IsString, IsOptional, IsEnum, MinLength } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class CreateUserDto {
  @ApiProperty({ example: 'alice@example.com' })
  @IsEmail()
  email: string;

  @ApiProperty({ minLength: 8 })
  @IsString()
  @MinLength(8)
  password: string;

  @ApiProperty({ required: false })
  @IsOptional()
  @IsString()
  displayName?: string;

  @ApiProperty({ enum: UserRole, default: UserRole.USER })
  @IsOptional()
  @IsEnum(UserRole)
  role?: UserRole;
}

// Trong service — map DTO sang schema
async create(dto: CreateUserDto): Promise<UserDocument> {
  const existing = await this.userModel.findOne({ email: dto.email });
  if (existing) throw new ConflictException('Email đã tồn tại');

  return this.userModel.create(dto);
}
```

---

## 13. Các lỗi thường gặp & xử lý

```typescript
import { MongoError } from 'mongodb';

// Exception Filter cho MongoDB errors
@Catch(MongoError)
export class MongoExceptionFilter implements ExceptionFilter {
  catch(exception: MongoError, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();

    switch (exception.code) {
      case 11000: // Duplicate key
        const field = Object.keys((exception as any).keyPattern)[0];
        return res.status(409).json({
          statusCode: 409,
          message: `${field} đã tồn tại`,
        });
      default:
        return res.status(500).json({ statusCode: 500, message: 'Database error' });
    }
  }
}

// Validate ObjectId
import { PipeTransform, Injectable, BadRequestException } from '@nestjs/common';
import { Types } from 'mongoose';

@Injectable()
export class ParseObjectIdPipe implements PipeTransform {
  transform(value: string) {
    if (!Types.ObjectId.isValid(value)) {
      throw new BadRequestException(`${value} không phải ObjectId hợp lệ`);
    }
    return new Types.ObjectId(value);
  }
}

// Dùng trong controller
@Get(':id')
findOne(@Param('id', ParseObjectIdPipe) id: Types.ObjectId) {
  return this.usersService.findById(id.toString());
}
```

---

## 14. Best Practices & Checklist

### Schema Design
- [ ] Dùng `timestamps: true` thay vì tự định nghĩa `createdAt`/`updatedAt`
- [ ] `versionKey: false` để bỏ `__v` field
- [ ] `select: false` cho các field nhạy cảm (password, token)
- [ ] Đặt `_id: false` cho sub-document nếu không cần ID riêng
- [ ] Dùng `toJSON.transform` để ẩn field nhạy cảm khỏi response

### Performance
- [ ] `.lean()` khi chỉ đọc data, không cần Mongoose methods
- [ ] Đặt index cho các field thường dùng trong query
- [ ] Dùng `select()` để chỉ lấy field cần thiết
- [ ] Dùng `countDocuments()` thay vì `.find().length`
- [ ] Aggregation thay vì nhiều query riêng lẻ
- [ ] Tránh `$where` và JavaScript expression trong query

### Relations
- [ ] Embed document khi data thường được truy cập cùng nhau (post + comments)
- [ ] Reference (ObjectId) khi data được truy cập độc lập (user, product)
- [ ] Tránh populate quá sâu (> 2 level) — dùng aggregation `$lookup` thay thế

### NestJS
- [ ] `@Global()` cho MongooseModule nếu dùng nhiều nơi
- [ ] Inject `@InjectConnection()` để dùng transaction
- [ ] `ParseObjectIdPipe` để validate ObjectId params
- [ ] Exception filter cho `MongoError` code 11000
- [ ] Dùng `FilterQuery<T>` type thay vì `any` cho query object
