# Redis – Tổng hợp kiến thức (đặc biệt cho NestJS)

---

## 1. Redis là gì?

Redis (Remote Dictionary Server) là **in-memory data store** mã nguồn mở, hoạt động theo mô hình key-value. Nó cực kỳ nhanh (sub-millisecond latency) vì dữ liệu được lưu trên RAM, và có thể persist xuống disk.

**Dùng Redis khi:**
- Cần cache để giảm tải database
- Session/token storage
- Rate limiting
- Pub/Sub messaging
- Job queues
- Real-time leaderboard / counters

---

## 2. Các kiểu dữ liệu (Data Types)

| Type | Lệnh cơ bản | Dùng khi |
|---|---|---|
| **String** | SET, GET, INCR, TTL | Cache đơn giản, counter |
| **Hash** | HSET, HGET, HGETALL | Lưu object (user profile) |
| **List** | LPUSH, RPUSH, LRANGE | Queue, log, timeline |
| **Set** | SADD, SMEMBERS, SINTER | Tags, unique visitors |
| **Sorted Set** | ZADD, ZRANGE, ZRANK | Leaderboard, priority queue |
| **Stream** | XADD, XREAD | Event log, message broker |
| **Bitmap / HyperLogLog** | SETBIT, PFADD | Tracking, approximation |

---

## 3. Các lệnh Redis cần nhớ

```bash
# String
SET key value EX 3600       # set với TTL 1h
GET key
INCR counter
MSET k1 v1 k2 v2

# Hash
HSET user:1 name "Alice" age 25
HGET user:1 name
HGETALL user:1

# List
LPUSH queue task1
RPOP queue
LRANGE mylist 0 -1

# Set
SADD tags nodejs redis
SMEMBERS tags

# Sorted Set
ZADD leaderboard 1000 "Alice"
ZRANGE leaderboard 0 -1 WITHSCORES REV

# Key management
TTL key
EXPIRE key 60
DEL key
KEYS pattern          # tránh dùng ở production
SCAN 0 MATCH * COUNT 100  # dùng thay KEYS
```

---

## 4. Redis trong NestJS

### 4.1. Cài đặt

```bash
# Cách 1: ioredis (khuyến nghị)
npm install ioredis @nestjs-modules/ioredis

# Cách 2: cache-manager (tích hợp sẵn NestJS)
npm install cache-manager cache-manager-ioredis-yet ioredis

# Cách 3: BullMQ (queue)
npm install @nestjs/bullmq bullmq ioredis
```

---

### 4.2. Caching với CacheModule (cache-manager)

```typescript
// app.module.ts
import { CacheModule } from '@nestjs/cache-manager';
import { redisStore } from 'cache-manager-ioredis-yet';

@Module({
  imports: [
    CacheModule.registerAsync({
      isGlobal: true,
      useFactory: async () => ({
        store: await redisStore({
          host: 'localhost',
          port: 6379,
          ttl: 60 * 1000, // ms
        }),
      }),
    }),
  ],
})
export class AppModule {}
```

```typescript
// service
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class ProductService {
  constructor(@Inject(CACHE_MANAGER) private cache: Cache) {}

  async getProduct(id: string) {
    const cached = await this.cache.get(`product:${id}`);
    if (cached) return cached;

    const product = await this.db.find(id);
    await this.cache.set(`product:${id}`, product, 300_000); // 5 phút
    return product;
  }
}
```

```typescript
// Dùng decorator @CacheKey / @CacheTTL
@Controller('products')
@UseInterceptors(CacheInterceptor)
export class ProductController {
  @Get(':id')
  @CacheKey('product')
  @CacheTTL(30)
  getProduct(@Param('id') id: string) { ... }
}
```

---

### 4.3. Dùng ioredis trực tiếp (linh hoạt hơn)

```typescript
// redis.module.ts
import { Module, Global } from '@nestjs/common';
import Redis from 'ioredis';

@Global()
@Module({
  providers: [
    {
      provide: 'REDIS_CLIENT',
      useFactory: () => new Redis({ host: 'localhost', port: 6379 }),
    },
  ],
  exports: ['REDIS_CLIENT'],
})
export class RedisModule {}
```

```typescript
// Inject và sử dụng
@Injectable()
export class AuthService {
  constructor(@Inject('REDIS_CLIENT') private redis: Redis) {}

  async saveRefreshToken(userId: string, token: string) {
    await this.redis.set(`refresh:${userId}`, token, 'EX', 604800); // 7 ngày
  }

  async getRefreshToken(userId: string) {
    return this.redis.get(`refresh:${userId}`);
  }

  async revokeToken(userId: string) {
    await this.redis.del(`refresh:${userId}`);
  }
}
```

---

### 4.4. Session Storage với Redis

```bash
npm install express-session connect-redis ioredis
```

```typescript
// main.ts
import * as session from 'express-session';
import { createClient } from 'redis';
import RedisStore from 'connect-redis';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const client = createClient({ url: 'redis://localhost:6379' });
  await client.connect();

  app.use(session({
    store: new RedisStore({ client }),
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    cookie: { maxAge: 86400000 },
  }));

  await app.listen(3000);
}
```

---

### 4.5. Rate Limiting với Redis

```bash
npm install @nestjs/throttler
```

```typescript
// Dùng ThrottlerModule + Redis Storage
import { ThrottlerModule } from '@nestjs/throttler';
import { ThrottlerStorageRedisService } from '@nest-lab/throttler-storage-redis';

@Module({
  imports: [
    ThrottlerModule.forRoot({
      throttlers: [{ ttl: 60000, limit: 10 }],
      storage: new ThrottlerStorageRedisService({ host: 'localhost', port: 6379 }),
    }),
  ],
})
export class AppModule {}
```

---

### 4.6. Job Queue với BullMQ

```typescript
// queue.module.ts
import { BullModule } from '@nestjs/bullmq';

@Module({
  imports: [
    BullModule.forRoot({ connection: { host: 'localhost', port: 6379 } }),
    BullModule.registerQueue({ name: 'email' }),
  ],
})
export class QueueModule {}
```

```typescript
// producer
@Injectable()
export class EmailService {
  constructor(@InjectQueue('email') private emailQueue: Queue) {}

  async sendWelcomeEmail(userId: string) {
    await this.emailQueue.add('welcome', { userId }, { delay: 0, attempts: 3 });
  }
}
```

```typescript
// consumer
@Processor('email')
export class EmailProcessor extends WorkerHost {
  async process(job: Job): Promise<void> {
    if (job.name === 'welcome') {
      await this.sendEmail(job.data.userId);
    }
  }
}
```

---

### 4.7. Pub/Sub với Redis

```typescript
// subscriber.service.ts
@Injectable()
export class NotificationSubscriber implements OnModuleInit {
  private subscriber: Redis;
  private publisher: Redis;

  constructor(@Inject('REDIS_CLIENT') private redis: Redis) {
    this.subscriber = redis.duplicate();
    this.publisher = redis.duplicate();
  }

  async onModuleInit() {
    await this.subscriber.subscribe('notifications');
    this.subscriber.on('message', (channel, message) => {
      console.log(`[${channel}]:`, JSON.parse(message));
    });
  }

  async publish(data: object) {
    await this.publisher.publish('notifications', JSON.stringify(data));
  }
}
```

---

## 5. Patterns & Best Practices

### 5.1. Cache-Aside Pattern (phổ biến nhất)
```
GET cache → miss → GET DB → SET cache → return
```

### 5.2. Naming conventions cho key
```
# Dùng dấu : để phân cấp
user:1001:profile
product:sku:ABC123:stock
session:abc123def456
refresh_token:user:1001
```

### 5.3. TTL – Luôn set TTL
```typescript
// Không bao giờ để key tồn tại vĩnh viễn (trừ khi thực sự cần)
await redis.set(key, value, 'EX', 3600); // EX = seconds
await redis.set(key, value, 'PX', 60000); // PX = milliseconds
```

### 5.4. Tránh KEYS * ở production
```typescript
// ❌ Sai – block server
await redis.keys('user:*');

// ✅ Đúng – dùng SCAN
let cursor = '0';
do {
  const [next, keys] = await redis.scan(cursor, 'MATCH', 'user:*', 'COUNT', '100');
  cursor = next;
  // process keys
} while (cursor !== '0');
```

### 5.5. Serialization
```typescript
// Luôn serialize object thành JSON
await redis.set(key, JSON.stringify(data));
const data = JSON.parse(await redis.get(key));
```

---

## 6. Redis Persistence

| Mode | Ý nghĩa | Dùng khi |
|---|---|---|
| **RDB** | Snapshot theo interval | Backup, dữ liệu không quá critical |
| **AOF** | Ghi log từng lệnh | Cần durability cao |
| **RDB + AOF** | Kết hợp cả hai | Production quan trọng |
| **No persistence** | Chỉ in-memory | Cache thuần túy |

---

## 7. Redis Cluster & High Availability

- **Sentinel**: Tự động failover, monitor primary/replica
- **Cluster**: Sharding tự động, scale ngang, 6 node tối thiểu (3 primary + 3 replica)
- **Replica**: Read replicas để giảm tải read

```typescript
// ioredis cluster
import { Cluster } from 'ioredis';

const cluster = new Cluster([
  { host: 'redis-1', port: 6379 },
  { host: 'redis-2', port: 6379 },
  { host: 'redis-3', port: 6379 },
]);
```

---

## 8. Monitoring & Debug

```bash
# CLI
redis-cli monitor          # xem lệnh real-time
redis-cli info memory      # thông tin memory
redis-cli info stats
redis-cli slowlog get 10   # 10 lệnh chậm nhất
redis-cli --latency

# Memory
MEMORY USAGE key           # bytes của một key
DEBUG OBJECT key           # encoding, refcount
```

**Tools:**
- **RedisInsight** (GUI chính thức, miễn phí)
- **Another Redis Desktop Manager**
- **Grafana + Redis Data Source**

---

## 9. Checklist khi dùng Redis trong NestJS

- [ ] Đặt TTL cho mọi key cache
- [ ] Dùng prefix/namespace theo môi trường (dev:, prod:)
- [ ] Xử lý Redis connection error (try/catch, fallback to DB)
- [ ] Không lưu sensitive data không mã hóa (password, card number)
- [ ] Dùng connection pool (ioredis tự xử lý)
- [ ] Monitor memory usage, set `maxmemory-policy` phù hợp
- [ ] Health check Redis trong NestJS TerminusModule
- [ ] Dùng pipeline/multi cho batch operations

```typescript
// Pipeline – gộp nhiều lệnh thành 1 request
const pipeline = redis.pipeline();
pipeline.set('k1', 'v1');
pipeline.set('k2', 'v2');
pipeline.incr('counter');
const results = await pipeline.exec();
```

---

## 10. maxmemory-policy

| Policy | Ý nghĩa |
|---|---|
| `noeviction` | Trả lỗi khi full (default) |
| `allkeys-lru` | Xóa key ít dùng nhất (khuyến nghị cho cache) |
| `volatile-lru` | Xóa key có TTL ít dùng nhất |
| `allkeys-random` | Xóa ngẫu nhiên |
| `volatile-ttl` | Xóa key sắp hết hạn nhất |

```bash
# redis.conf
maxmemory 256mb
maxmemory-policy allkeys-lru
```
