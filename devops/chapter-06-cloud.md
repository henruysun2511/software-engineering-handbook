# Chương 6: Cloud

> **Mức độ quan trọng:** ⭐⭐⭐⭐  
> **Đối tượng:** Backend Developer (NestJS/Express), trình độ Intern → Junior  
> **Mục tiêu chương:** Hiểu các thành phần hạ tầng cloud cơ bản — VPS, Object Storage, CDN, DNS, Cloud Database — đủ để tự vận hành một ứng dụng backend trên môi trường production thực tế và trao đổi hiệu quả với DevOps Engineer.

---

## 6.1. VPS — Virtual Private Server

### 6.1.1. VPS là gì?

**VPS (Virtual Private Server)** là một máy chủ ảo được tạo ra bằng cách phân chia một máy chủ vật lý thành nhiều máy ảo độc lập thông qua công nghệ ảo hóa (virtualization). Mỗi VPS có CPU, RAM, ổ đĩa và địa chỉ IP riêng, hoạt động như một máy chủ độc lập.

### 6.1.2. So sánh các lựa chọn hosting

```
Shared Hosting < VPS < Dedicated Server < Cloud (auto-scale)
     ↑               ↑           ↑               ↑
 Rẻ nhất,       Cân bằng    Mạnh nhất,     Linh hoạt nhất,
 ít kiểm soát   tốt nhất    đắt nhất       trả theo dùng
                cho startup
```

| Tiêu chí | Shared Hosting | VPS | Dedicated | Cloud VM |
|---|---|---|---|---|
| **Giá** | $3–10/tháng | $5–50/tháng | $100+/tháng | Pay-as-you-go |
| **Root access** | Không | Có | Có | Có |
| **Tài nguyên** | Chia sẻ | Cố định | Toàn bộ | Linh hoạt |
| **Kiểm soát** | Rất ít | Cao | Cao | Cao |
| **Scale** | Khó | Thủ công | Khó | Tự động |
| **Phù hợp** | Blog, web nhỏ | Startup, API | Traffic lớn | Doanh nghiệp |

### 6.1.3. Các nhà cung cấp VPS phổ biến

| Provider | Gói rẻ nhất | Điểm mạnh |
|---|---|---|
| **DigitalOcean** | $6/tháng (Droplet) | Đơn giản, tài liệu tốt, phù hợp người mới |
| **Vultr** | $5/tháng | Nhiều region, giá cạnh tranh |
| **Linode (Akamai)** | $5/tháng | Ổn định, hỗ trợ tốt |
| **Hetzner** | €4/tháng | Rất rẻ, datacenter EU |
| **AWS EC2** | ~$8/tháng (t3.micro) | Tích hợp hệ sinh thái AWS |
| **GCP Compute Engine** | ~$7/tháng (e2-micro) | Free tier 1 máy nhỏ |

### 6.1.4. Cấu hình VPS tối thiểu cho NestJS + PostgreSQL + Redis

```
Ứng dụng nhỏ (< 1000 user):
  CPU:  1 vCPU
  RAM:  1 GB
  Disk: 20 GB SSD
  → DigitalOcean Droplet Basic: $6/tháng

Ứng dụng trung bình (1000–10000 user):
  CPU:  2 vCPU
  RAM:  4 GB
  Disk: 80 GB SSD
  → DigitalOcean Droplet Basic: $24/tháng

Quy tắc thực tế:
  NestJS:    ~100–200 MB RAM khi idle
  PostgreSQL: ~100–300 MB RAM khi idle  
  Redis:     ~10–50 MB RAM khi idle
  OS + misc: ~200–300 MB RAM
  ─────────────────────────────────────
  Tổng tối thiểu: ~512 MB → Dùng 1 GB để có buffer
```

### 6.1.5. Tạo và kết nối VPS — DigitalOcean

```bash
# ─────────────────────────────────────────
# Trên DigitalOcean UI:
# 1. Create → Droplets
# 2. Chọn region gần người dùng
# 3. Chọn OS: Ubuntu 22.04 LTS
# 4. Chọn plan phù hợp
# 5. Authentication: SSH Keys (không dùng password!)
# 6. Create Droplet
# ─────────────────────────────────────────

# Thêm SSH key trước khi tạo Droplet:
# 1. Tạo key pair trên máy local:
ssh-keygen -t ed25519 -C "my-server-key" -f ~/.ssh/do_server

# 2. Copy public key:
cat ~/.ssh/do_server.pub
# Paste vào DigitalOcean → Settings → Security → SSH Keys

# 3. Kết nối sau khi tạo:
ssh -i ~/.ssh/do_server root@YOUR_DROPLET_IP

# Tạo config SSH để không phải gõ dài:
# File: ~/.ssh/config
Host myserver
    HostName 203.0.113.10
    User deploy
    IdentityFile ~/.ssh/do_server

# Sau đó kết nối đơn giản:
ssh myserver
```

### 6.1.6. Chọn Region

Nguyên tắc: Chọn region **gần với phần lớn người dùng** của bạn.

```
Người dùng Việt Nam / Đông Nam Á:
  → Singapore (sgp1) — DigitalOcean, Vultr, AWS ap-southeast-1
  → Tokyo (tyo1) — thường nhanh với VN
  Latency từ VN: ~30-50ms

Người dùng Mỹ:
  → New York, San Francisco
  Latency từ VN: ~200-250ms (dùng CDN để giảm)
```

---

## 6.2. Object Storage

### 6.2.1. Vấn đề lưu file trên server

Khi người dùng upload avatar, tài liệu hay ảnh sản phẩm, cách đơn giản nhất là lưu vào ổ đĩa của VPS:

```
❌ Vấn đề khi lưu file trên VPS disk:

1. Disk sẽ đầy → Server ngừng hoạt động
2. Scale horizontally → File không sync giữa các instance
3. Backup phức tạp
4. Server hỏng → Mất toàn bộ file user
5. Phục vụ file tĩnh tốn tài nguyên CPU/RAM
6. Không có CDN → Chậm với user ở xa
```

**Giải pháp:** Dùng **Object Storage** — dịch vụ lưu trữ file chuyên dụng với độ bền cao, giá rẻ và tích hợp CDN.

### 6.2.2. Object Storage là gì?

**Object Storage** là hình thức lưu trữ trong đó mỗi file (object) được lưu cùng metadata và một unique identifier (key), có thể truy cập qua HTTP URL.

```
Traditional File System:    Object Storage:
/uploads/                   bucket-name/
  /users/                     avatars/user-123.jpg
    /123/                       (URL: https://cdn.myapp.com/avatars/user-123.jpg)
      avatar.jpg              documents/contract-456.pdf
      doc.pdf                 products/product-789-thumb.jpg
```

### 6.2.3. Các nhà cung cấp Object Storage

| Provider | Tên dịch vụ | Giá lưu trữ | Tương thích S3 |
|---|---|---|---|
| **AWS** | S3 | $0.023/GB/tháng | Chuẩn gốc |
| **DigitalOcean** | Spaces | $0.02/GB, $5 tối thiểu | ✅ |
| **Cloudflare** | R2 | $0.015/GB, free egress | ✅ |
| **Backblaze** | B2 | $0.006/GB | ✅ |
| **MinIO** | Tự host | Chi phí server | ✅ (S3-compatible) |

> 💡 **Khuyến nghị:** **Cloudflare R2** — giá rẻ nhất, không tính phí băng thông (egress free), tích hợp Cloudflare CDN. Hoặc **DigitalOcean Spaces** nếu đang dùng DO cho VPS.

### 6.2.4. Tích hợp S3/Object Storage vào NestJS

Dùng AWS SDK v3 — tương thích với tất cả provider S3-compatible:

```bash
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
npm install multer @types/multer
```

```typescript
// src/storage/storage.service.ts
import { Injectable, BadRequestException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import {
  S3Client,
  PutObjectCommand,
  DeleteObjectCommand,
  GetObjectCommand,
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { v4 as uuidv4 } from 'uuid';
import * as path from 'path';

@Injectable()
export class StorageService {
  private readonly s3: S3Client;
  private readonly bucket: string;
  private readonly publicUrl: string;

  constructor(private readonly config: ConfigService) {
    // Cấu hình S3 client — tương thích với bất kỳ provider nào
    this.s3 = new S3Client({
      region: this.config.get<string>('STORAGE_REGION', 'auto'),
      endpoint: this.config.get<string>('STORAGE_ENDPOINT'),  // Khác nhau theo provider
      credentials: {
        accessKeyId: this.config.get<string>('STORAGE_ACCESS_KEY'),
        secretAccessKey: this.config.get<string>('STORAGE_SECRET_KEY'),
      },
      // Cloudflare R2 và một số provider cần forcePathStyle
      forcePathStyle: this.config.get<string>('STORAGE_FORCE_PATH_STYLE') === 'true',
    });

    this.bucket = this.config.get<string>('STORAGE_BUCKET');
    this.publicUrl = this.config.get<string>('STORAGE_PUBLIC_URL');
  }

  /**
   * Upload file lên Object Storage
   * @param file - File từ multer
   * @param folder - Thư mục (vd: 'avatars', 'documents')
   * @returns URL công khai của file
   */
  async uploadFile(
    file: Express.Multer.File,
    folder: string = 'uploads',
  ): Promise<{ key: string; url: string }> {
    // Validate file type
    const allowedMimeTypes = [
      'image/jpeg', 'image/png', 'image/webp', 'image/gif',
      'application/pdf',
      'application/msword',
      'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
    ];

    if (!allowedMimeTypes.includes(file.mimetype)) {
      throw new BadRequestException(
        `File type not allowed: ${file.mimetype}`,
      );
    }

    // Validate file size (10MB)
    const maxSize = 10 * 1024 * 1024;
    if (file.size > maxSize) {
      throw new BadRequestException('File size exceeds 10MB limit');
    }

    // Tạo key duy nhất — tránh overwrite và directory traversal
    const ext = path.extname(file.originalname).toLowerCase();
    const key = `${folder}/${uuidv4()}${ext}`;

    // Upload lên S3
    await this.s3.send(
      new PutObjectCommand({
        Bucket: this.bucket,
        Key: key,
        Body: file.buffer,
        ContentType: file.mimetype,
        ContentLength: file.size,
        // Metadata hữu ích để debug
        Metadata: {
          originalName: file.originalname,
          uploadedAt: new Date().toISOString(),
        },
      }),
    );

    const url = `${this.publicUrl}/${key}`;

    return { key, url };
  }

  /**
   * Xóa file khỏi Object Storage
   */
  async deleteFile(key: string): Promise<void> {
    await this.s3.send(
      new DeleteObjectCommand({
        Bucket: this.bucket,
        Key: key,
      }),
    );
  }

  /**
   * Tạo presigned URL — cho phép client upload trực tiếp lên S3
   * mà không cần qua server (tránh tốn băng thông server)
   */
  async getPresignedUploadUrl(
    key: string,
    contentType: string,
    expiresIn: number = 300, // 5 phút
  ): Promise<string> {
    const command = new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      ContentType: contentType,
    });

    return getSignedUrl(this.s3, command, { expiresIn });
  }

  /**
   * Tạo presigned URL để download file private
   */
  async getPresignedDownloadUrl(
    key: string,
    expiresIn: number = 3600, // 1 giờ
  ): Promise<string> {
    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: key,
    });

    return getSignedUrl(this.s3, command, { expiresIn });
  }
}
```

```typescript
// src/users/users.controller.ts — Upload avatar endpoint
import {
  Controller, Post, UseInterceptors,
  UploadedFile, UseGuards, Req,
} from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { memoryStorage } from 'multer';
import { StorageService } from '../storage/storage.service';
import { UsersService } from './users.service';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';

@Controller('users')
@UseGuards(JwtAuthGuard)
export class UsersController {
  constructor(
    private readonly storage: StorageService,
    private readonly users: UsersService,
  ) {}

  @Post('me/avatar')
  @UseInterceptors(
    FileInterceptor('file', {
      storage: memoryStorage(), // Lưu vào RAM trước khi upload lên S3
      limits: { fileSize: 5 * 1024 * 1024 }, // 5MB
      fileFilter: (_, file, cb) => {
        if (!file.mimetype.match(/^image\/(jpeg|png|webp)$/)) {
          return cb(new BadRequestException('Only image files allowed'), false);
        }
        cb(null, true);
      },
    }),
  )
  async uploadAvatar(
    @UploadedFile() file: Express.Multer.File,
    @Req() req,
  ) {
    const { key, url } = await this.storage.uploadFile(file, 'avatars');

    // Xóa avatar cũ nếu có
    const user = await this.users.findOne(req.user.id);
    if (user.avatarKey) {
      await this.storage.deleteFile(user.avatarKey).catch(() => {
        // Không fail nếu file cũ đã bị xóa
      });
    }

    // Cập nhật user
    await this.users.update(req.user.id, { avatarKey: key, avatarUrl: url });

    return { avatarUrl: url };
  }
}
```

**Cấu hình `.env` cho các provider khác nhau:**

```bash
# AWS S3
STORAGE_ENDPOINT=               # Để trống — dùng mặc định AWS
STORAGE_REGION=ap-southeast-1
STORAGE_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE
STORAGE_SECRET_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
STORAGE_BUCKET=myapp-uploads
STORAGE_PUBLIC_URL=https://myapp-uploads.s3.ap-southeast-1.amazonaws.com
STORAGE_FORCE_PATH_STYLE=false

# Cloudflare R2
STORAGE_ENDPOINT=https://ACCOUNT_ID.r2.cloudflarestorage.com
STORAGE_REGION=auto
STORAGE_ACCESS_KEY=your-r2-access-key
STORAGE_SECRET_KEY=your-r2-secret-key
STORAGE_BUCKET=myapp-uploads
STORAGE_PUBLIC_URL=https://assets.myapp.com  # Custom domain qua Cloudflare
STORAGE_FORCE_PATH_STYLE=true

# DigitalOcean Spaces (region Singapore)
STORAGE_ENDPOINT=https://sgp1.digitaloceanspaces.com
STORAGE_REGION=sgp1
STORAGE_ACCESS_KEY=your-spaces-key
STORAGE_SECRET_KEY=your-spaces-secret
STORAGE_BUCKET=myapp-uploads
STORAGE_PUBLIC_URL=https://myapp-uploads.sgp1.cdn.digitaloceanspaces.com
STORAGE_FORCE_PATH_STYLE=false

# MinIO (self-hosted, dùng cho development)
STORAGE_ENDPOINT=http://localhost:9000
STORAGE_REGION=us-east-1
STORAGE_ACCESS_KEY=minioadmin
STORAGE_SECRET_KEY=minioadmin
STORAGE_BUCKET=myapp-uploads
STORAGE_PUBLIC_URL=http://localhost:9000/myapp-uploads
STORAGE_FORCE_PATH_STYLE=true
```

**MinIO trong Docker Compose (môi trường development):**

```yaml
# docker-compose.yml — thêm service minio
  minio:
    image: minio/minio:latest
    container_name: myapp-minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"   # API
      - "9001:9001"   # Console UI: http://localhost:9001
    volumes:
      - minio-data:/data
    networks:
      - app-network

volumes:
  minio-data:
```

---

## 6.3. CDN — Content Delivery Network

### 6.3.1. CDN là gì?

**CDN (Content Delivery Network)** là mạng lưới các server được đặt ở nhiều vị trí địa lý trên thế giới (gọi là PoP — Points of Presence). Khi user request file, CDN phục vụ từ server gần nhất thay vì từ server gốc.

```
Không có CDN:                    Có CDN:
User ở Hà Nội                    User ở Hà Nội
    │                                │
    │ 200ms                          │ 15ms
    ▼                                ▼
Server ở Mỹ              CDN PoP Singapore
                                     │
                                     │ (Cache miss — lần đầu)
                                     │ 200ms
                                     ▼
                              Server ở Mỹ (origin)
```

### 6.3.2. CDN giải quyết vấn đề gì?

| Vấn đề | CDN giải quyết như thế nào |
|---|---|
| **Latency cao** | Phục vụ từ PoP gần user nhất |
| **Server bị quá tải** | Cache và phục vụ static file, giảm tải server gốc |
| **DDoS attack** | Hấp thụ và lọc traffic tại CDN layer |
| **Single point of failure** | Nhiều PoP, nếu một PoP down thì PoP khác tiếp tục |
| **Tốn băng thông server** | CDN trả về cached response, server gốc không cần xử lý |

### 6.3.3. Loại nội dung phù hợp với CDN

```
✅ Nên cache trên CDN:
   - File tĩnh: JS, CSS, fonts, images
   - File upload của user: avatars, documents
   - Video, audio

❌ Không nên cache trên CDN:
   - API response động (dữ liệu thay đổi theo user)
   - Trang cần authentication
   - Real-time data
   (Có thể cache API response nếu dữ liệu ít thay đổi và không cần auth)
```

### 6.3.4. Cloudflare — CDN phổ biến nhất

**Cloudflare** cung cấp CDN miễn phí với gói Free, tích hợp DNS, DDoS protection và nhiều tính năng bảo mật. Là lựa chọn hàng đầu cho startup và dự án nhỏ đến vừa.

**Cấu hình Cloudflare cho domain:**

```
Bước 1: Đăng ký tài khoản tại cloudflare.com
Bước 2: Add site → Nhập domain của bạn
Bước 3: Chọn gói Free
Bước 4: Cloudflare scan DNS records hiện tại
Bước 5: Cloudflare cung cấp 2 nameserver:
  bob.ns.cloudflare.com
  elsa.ns.cloudflare.com
Bước 6: Vào nhà cung cấp domain → Thay đổi nameserver
Bước 7: Chờ 24-48h để DNS propagate
```

**Cấu hình DNS records trên Cloudflare:**

```
Type  Name              Content          Proxy
A     @                 203.0.113.10     ✅ Proxied (qua Cloudflare)
A     api               203.0.113.10     ✅ Proxied
A     www               203.0.113.10     ✅ Proxied
CNAME assets            myapp.r2.dev     ✅ Proxied (R2 bucket)
MX    @                 mail.myapp.com   ❌ DNS only (không proxy mail)
```

**Proxy status:**
- **Proxied (cam):** Traffic đi qua Cloudflare — có CDN, DDoS protection, ẩn IP thật của server.
- **DNS only (xám):** Cloudflare chỉ làm DNS, không có CDN.

### 6.3.5. Cache-Control Headers

CDN dựa vào response headers từ server gốc để quyết định cache bao lâu:

```typescript
// src/common/interceptors/cache-control.interceptor.ts
import {
  Injectable, NestInterceptor, ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class CacheControlInterceptor implements NestInterceptor {
  constructor(private readonly maxAge: number) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const response = context.switchToHttp().getResponse();

    return next.handle().pipe(
      tap(() => {
        response.setHeader(
          'Cache-Control',
          `public, max-age=${this.maxAge}, stale-while-revalidate=${this.maxAge * 2}`,
        );
      }),
    );
  }
}
```

```typescript
// Dùng trong controller
import { UseInterceptors } from '@nestjs/common';

@Controller('products')
export class ProductsController {

  // Cache 10 phút — dữ liệu sản phẩm ít thay đổi
  @Get()
  @UseInterceptors(new CacheControlInterceptor(600))
  findAll() {
    return this.productsService.findAll();
  }

  // Không cache — dữ liệu riêng của user
  @Get('me/cart')
  @UseInterceptors(new CacheControlInterceptor(0))
  getCart(@Req() req) {
    return this.cartService.getUserCart(req.user.id);
  }
}
```

---

## 6.4. Domain và DNS

### 6.4.1. Domain là gì?

**Domain** (tên miền) là địa chỉ dễ nhớ của website/API, thay cho địa chỉ IP số khó nhớ.

```
203.0.113.10  ←→  api.myapp.com
   IP                Domain
(máy tính hiểu)   (người dùng nhớ)
```

**Cấu trúc domain:**

```
api.myapp.com.
 ↑     ↑    ↑ ↑
Sub  Second Top Root
dom  level  level (ẩn)
ain  domain domain
```

### 6.4.2. DNS là gì?

**DNS (Domain Name System)** là hệ thống "danh bạ" của internet, chuyển đổi tên domain thành địa chỉ IP.

**Quy trình DNS resolution:**

```
Browser muốn truy cập api.myapp.com

1. Kiểm tra DNS cache của OS
   → Có rồi? → Dùng luôn

2. Hỏi Recursive Resolver (thường là DNS của ISP hoặc 8.8.8.8)
   → Đã cache? → Trả về

3. Recursive Resolver hỏi Root Name Server (.)
   → "ai quản lý .com?"
   → Trả về địa chỉ TLD Name Server (.com)

4. Hỏi TLD Name Server (.com)
   → "ai quản lý myapp.com?"
   → Trả về địa chỉ Authoritative Name Server của myapp.com

5. Hỏi Authoritative Name Server (Cloudflare/Route53/...)
   → "api.myapp.com là gì?"
   → Trả về: 203.0.113.10

6. Browser kết nối đến 203.0.113.10
```

### 6.4.3. Các loại DNS Record

| Type | Mục đích | Ví dụ |
|---|---|---|
| **A** | Domain → IPv4 | `api.myapp.com → 203.0.113.10` |
| **AAAA** | Domain → IPv6 | `api.myapp.com → 2001:db8::1` |
| **CNAME** | Domain → Domain khác | `www.myapp.com → myapp.com` |
| **MX** | Mail server | `myapp.com → mail.myapp.com` |
| **TXT** | Văn bản tùy ý | SPF, DKIM, domain verification |
| **NS** | Name Server | `myapp.com → bob.ns.cloudflare.com` |
| **SOA** | Start of Authority | Thông tin zone (tự động) |

### 6.4.4. Các lệnh kiểm tra DNS

```bash
# Kiểm tra A record
dig api.myapp.com A
nslookup api.myapp.com

# Kiểm tra tất cả record
dig myapp.com ANY

# Kiểm tra từ DNS server cụ thể (bỏ qua cache)
dig @8.8.8.8 api.myapp.com A       # Dùng Google DNS
dig @1.1.1.1 api.myapp.com A       # Dùng Cloudflare DNS

# Kiểm tra DNS propagation (domain vừa đổi nameserver)
dig @bob.ns.cloudflare.com api.myapp.com   # Hỏi thẳng Cloudflare NS

# Trace toàn bộ chain DNS
dig +trace api.myapp.com

# Kiểm tra TTL còn lại
dig api.myapp.com | grep -A2 "ANSWER"
# api.myapp.com. 300 IN A 203.0.113.10
#               ↑
#               TTL = 300 giây còn lại trong cache
```

### 6.4.5. TTL — Time To Live

**TTL** là thời gian (giây) DNS record được cache. Ảnh hưởng trực tiếp đến tốc độ propagation khi thay đổi DNS.

```
TTL = 86400 (1 ngày): Thay đổi DNS mất đến 24h propagate
TTL = 300 (5 phút):   Thay đổi DNS mất 5 phút propagate

Chiến lược khi cần migrate server:
  1. Giảm TTL xuống 300 trước 24h (chờ cache cũ hết hạn)
  2. Thực hiện thay đổi IP
  3. Chờ 5-10 phút để propagate
  4. Tăng TTL lại về 3600 hoặc 86400
```

---

## 6.5. Cloud Database

### 6.5.1. Managed Database là gì?

**Managed Database** (hay Cloud Database) là dịch vụ database được nhà cung cấp cloud quản lý hoàn toàn — bao gồm cài đặt, cấu hình, backup, update, failover và monitoring. Developer chỉ cần kết nối và dùng.

**So sánh tự cài vs Managed:**

| Tiêu chí | Tự cài trên VPS | Managed Database |
|---|---|---|
| **Chi phí** | Thấp (chỉ VPS) | Cao hơn 2-3x |
| **Công sức** | Cao (tự lo tất cả) | Thấp (chỉ dùng) |
| **Backup** | Tự thiết lập | Tự động |
| **High Availability** | Phức tạp | Sẵn có |
| **Security patch** | Tự cập nhật | Tự động |
| **Phù hợp** | Dev/Staging, startup | Production, scale |

### 6.5.2. Khi nào nên dùng Managed Database?

```
Dùng PostgreSQL trên VPS khi:
  • Development, staging
  • Budget hạn chế
  • Team nhỏ, chấp nhận rủi ro
  • Đã có DevOps chuyên trách

Dùng Managed Database khi:
  • Production với dữ liệu quan trọng
  • Cần High Availability (HA) — failover tự động
  • Không có người chuyên quản lý DB
  • Cần compliance (GDPR, HIPAA...)
  • Scale lên nhiều user
```

### 6.5.3. Các dịch vụ phổ biến

**PostgreSQL Managed:**

| Provider | Tên | Gói rẻ nhất | Điểm mạnh |
|---|---|---|---|
| **Neon** | Neon Serverless | Miễn phí | Serverless, auto-scale, free tier |
| **Supabase** | Supabase DB | Miễn phí | Open source, có Auth/Storage kèm |
| **DigitalOcean** | Managed PostgreSQL | $15/tháng | Đơn giản, cùng region với Droplet |
| **AWS** | RDS PostgreSQL | ~$25/tháng | Tích hợp hệ sinh thái AWS |
| **Render** | PostgreSQL | Miễn phí (có giới hạn) | Deploy đơn giản |

**Redis Managed:**

| Provider | Tên | Gói rẻ nhất |
|---|---|---|
| **Upstash** | Redis | Miễn phí (pay-per-request) |
| **Redis Cloud** | Redis Enterprise Cloud | Miễn phí 30MB |
| **DigitalOcean** | Managed Redis | $15/tháng |
| **AWS** | ElastiCache | ~$20/tháng |

### 6.5.4. Kết nối Managed Database trong NestJS

```bash
# Connection string format:
# postgresql://USER:PASSWORD@HOST:PORT/DATABASE?sslmode=require
# redis://default:PASSWORD@HOST:PORT

# Ví dụ Neon (serverless PostgreSQL):
DATABASE_URL=postgresql://myapp_user:password@ep-cool-mountain-12345.us-east-2.aws.neon.tech/myapp?sslmode=require

# Ví dụ Supabase:
DATABASE_URL=postgresql://postgres:password@db.abcdefghijklmnop.supabase.co:5432/postgres

# Ví dụ DigitalOcean Managed PostgreSQL:
DATABASE_URL=postgresql://doadmin:password@myapp-db-do-user-123456-0.db.ondigitalocean.com:25060/defaultdb?sslmode=require
```

```typescript
// src/database/database.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigService } from '@nestjs/config';

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => {
        const isProd = config.get('NODE_ENV') === 'production';

        // Hỗ trợ cả connection string lẫn từng param riêng
        const databaseUrl = config.get<string>('DATABASE_URL');

        if (databaseUrl) {
          // Dùng connection string (Managed DB thường cung cấp dạng này)
          return {
            type: 'postgres',
            url: databaseUrl,
            ssl: isProd ? { rejectUnauthorized: false } : false,
            entities: [__dirname + '/../**/*.entity{.ts,.js}'],
            migrations: [__dirname + '/../database/migrations/*{.ts,.js}'],
            synchronize: false,
            logging: !isProd,
            // Connection pool — quan trọng cho production
            extra: {
              max: 10,              // Tối đa 10 connections
              min: 2,               // Giữ tối thiểu 2 connections
              idleTimeoutMillis: 30000,
              connectionTimeoutMillis: 2000,
            },
          };
        }

        // Dùng từng param riêng (tự cài trên VPS)
        return {
          type: 'postgres',
          host: config.get('DB_HOST'),
          port: config.get<number>('DB_PORT'),
          database: config.get('DB_NAME'),
          username: config.get('DB_USER'),
          password: config.get('DB_PASSWORD'),
          ssl: isProd ? { rejectUnauthorized: false } : false,
          entities: [__dirname + '/../**/*.entity{.ts,.js}'],
          migrations: [__dirname + '/../database/migrations/*{.ts,.js}'],
          synchronize: false,
          logging: !isProd,
          extra: { max: 10, min: 2 },
        };
      },
    }),
  ],
})
export class DatabaseModule {}
```

---

## 6.6. Cloud Services — Tổng Quan

### 6.6.1. Các dịch vụ cloud thường gặp với Backend Developer

Bên cạnh VPS, Object Storage, CDN và Database, backend developer thường cần làm việc với các dịch vụ sau:

**Email Service:**

| Dịch vụ | Free tier | Dùng khi |
|---|---|---|
| **Resend** | 3,000 email/tháng | Modern API, dễ dùng |
| **SendGrid** | 100 email/ngày | Phổ biến, nhiều tính năng |
| **Mailgun** | 100 email/ngày | Tốt cho transactional |
| **AWS SES** | 62,000 email/tháng (nếu gửi từ EC2) | Chi phí thấp nhất |

```typescript
// src/mail/mail.service.ts — Dùng Resend
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Resend } from 'resend';

@Injectable()
export class MailService {
  private readonly resend: Resend;
  private readonly from: string;

  constructor(private readonly config: ConfigService) {
    this.resend = new Resend(config.get('RESEND_API_KEY'));
    this.from = config.get('MAIL_FROM', 'noreply@myapp.com');
  }

  async sendWelcomeEmail(to: string, name: string): Promise<void> {
    await this.resend.emails.send({
      from: this.from,
      to,
      subject: `Chào mừng ${name} đến với MyApp!`,
      html: `
        <h1>Xin chào ${name}!</h1>
        <p>Tài khoản của bạn đã được tạo thành công.</p>
        <a href="https://myapp.com/login">Đăng nhập ngay</a>
      `,
    });
  }

  async sendPasswordResetEmail(to: string, resetToken: string): Promise<void> {
    const resetUrl = `https://myapp.com/reset-password?token=${resetToken}`;

    await this.resend.emails.send({
      from: this.from,
      to,
      subject: 'Đặt lại mật khẩu MyApp',
      html: `
        <p>Nhấn vào link bên dưới để đặt lại mật khẩu:</p>
        <a href="${resetUrl}">${resetUrl}</a>
        <p>Link có hiệu lực trong 1 giờ.</p>
        <p>Nếu bạn không yêu cầu, hãy bỏ qua email này.</p>
      `,
    });
  }
}
```

**Payment Gateway:**

```
Stripe    → Phổ biến nhất, SDK đầy đủ, sandbox tốt
PayPal    → Phổ biến với B2C
MoMo      → Thanh toán nội địa Việt Nam
VNPay     → Thanh toán nội địa Việt Nam, kết nối ngân hàng
```

**Push Notification:**

```
Firebase Cloud Messaging (FCM) → Mobile (iOS/Android), miễn phí
OneSignal                       → Multi-platform, free tier tốt
```

### 6.6.2. Firewall và Security Group

Mọi VPS và Cloud VM đều có **firewall** — chỉ mở port cần thiết, chặn tất cả còn lại.

```bash
# UFW — Uncomplicated Firewall (Ubuntu)

# Kiểm tra trạng thái
sudo ufw status verbose

# Chính sách mặc định
sudo ufw default deny incoming   # Chặn tất cả traffic vào
sudo ufw default allow outgoing  # Cho phép tất cả traffic ra

# Mở các port cần thiết
sudo ufw allow ssh               # Port 22 — SSH
sudo ufw allow http              # Port 80 — HTTP
sudo ufw allow https             # Port 443 — HTTPS

# KHÔNG mở các port sau ra internet (chỉ dùng nội bộ):
# Port 5432 — PostgreSQL (chỉ app cùng server mới cần)
# Port 6379 — Redis
# Port 3000 — NestJS (chỉ Nginx mới cần)

# Cho phép từ IP cụ thể (văn phòng, developer)
sudo ufw allow from 203.0.113.5 to any port 5432 comment "Office PostgreSQL access"

# Xem rules hiện tại (có số)
sudo ufw status numbered

# Xóa rule theo số
sudo ufw delete 3

# Bật firewall
sudo ufw enable

# Xem log
sudo tail -f /var/log/ufw.log
```

### 6.6.3. SSH Key Management

```bash
# ─────────────────────────────────────────
# Trên máy developer local
# ─────────────────────────────────────────

# Tạo SSH key pair (mỗi server nên có key riêng)
ssh-keygen -t ed25519 -C "myapp-production" -f ~/.ssh/myapp_prod
ssh-keygen -t ed25519 -C "myapp-staging" -f ~/.ssh/myapp_staging

# File sinh ra:
# ~/.ssh/myapp_prod      ← Private key (TUYỆT ĐỐI KHÔNG chia sẻ)
# ~/.ssh/myapp_prod.pub  ← Public key (copy lên server)

# Cấu hình SSH config để tiện kết nối
cat >> ~/.ssh/config << 'EOF'
Host myapp-prod
    HostName 203.0.113.10
    User deploy
    IdentityFile ~/.ssh/myapp_prod
    ServerAliveInterval 60

Host myapp-staging
    HostName 203.0.113.20
    User deploy
    IdentityFile ~/.ssh/myapp_staging
EOF

# Kết nối:
ssh myapp-prod      # Thay vì: ssh -i ~/.ssh/myapp_prod deploy@203.0.113.10

# ─────────────────────────────────────────
# Trên server
# ─────────────────────────────────────────

# Thêm public key của developer mới
echo "ssh-ed25519 AAAA... new-developer-key" >> ~/.ssh/authorized_keys

# Thu hồi quyền khi developer nghỉ việc
# Mở file và xóa dòng chứa key của họ:
vim ~/.ssh/authorized_keys

# Tắt đăng nhập bằng password (chỉ dùng key)
sudo vim /etc/ssh/sshd_config
# Sửa:
# PasswordAuthentication no
# PermitRootLogin no
# MaxAuthTries 3
sudo systemctl restart sshd
```

---

## 6.7. Quy Trình Thực Tế: Setup Hạ Tầng Từ Đầu

Kịch bản: Bạn vừa tạo dự án NestJS và cần setup hạ tầng production.

```
Tuần 1: MVP — Nhanh và đơn giản
  ✅ 1 VPS (DigitalOcean $12/tháng, 1vCPU/2GB)
  ✅ PostgreSQL trên cùng VPS
  ✅ Redis trên cùng VPS
  ✅ Cloudflare Free (DNS + CDN + SSL)
  ✅ DigitalOcean Spaces ($5/tháng) cho file upload
  Tổng: ~$17/tháng

Tháng 3: Scale — Khi có user
  ✅ Upgrade VPS lên 2vCPU/4GB ($24/tháng)
  ✅ Tách PostgreSQL → Managed DB ($15/tháng)
  ✅ Tách Redis → Managed Redis hoặc Upstash (free/pay-per-use)
  Tổng: ~$44/tháng

Tháng 6: High Availability
  ✅ 2 VPS + Load Balancer
  ✅ PostgreSQL HA với read replica
  ✅ Redis Cluster
  ✅ Monitoring chuyên sâu
  Tổng: ~$100+/tháng
```

**Quy trình setup theo thứ tự:**

```bash
# 1. Mua domain (namecheap, godaddy, tenten.vn...)
# 2. Tạo VPS
# 3. Đăng ký Cloudflare → Thêm domain → Đổi nameserver
# 4. Thêm DNS records:
#    A @ → IP VPS
#    A api → IP VPS
#    A www → IP VPS
# 5. Setup VPS (xem Chương 5.5.1)
# 6. Deploy ứng dụng (xem Chương 5.5.2)
# 7. Nginx + SSL (xem Chương 4)
# 8. Tạo S3/Spaces bucket cho file upload
# 9. Cấu hình CORS trên bucket:
#    Allowed Origins: https://myapp.com
#    Allowed Methods: GET, PUT, POST, DELETE
# 10. Test toàn bộ flow

# Kiểm tra sau khi setup xong:
curl https://api.myapp.com/health                  # API hoạt động
curl -I https://api.myapp.com | grep "strict-transport"  # HTTPS đúng
curl https://assets.myapp.com/test.jpg             # CDN hoạt động
```

---

## 6.8. Best Practices

### 6.8.1. Bảo mật hạ tầng

```bash
# ✅ Chỉ mở port cần thiết
sudo ufw allow ssh http https
# Không mở: 5432, 6379, 3000

# ✅ Không dùng root SSH
PermitRootLogin no  # Trong /etc/ssh/sshd_config

# ✅ Chỉ dùng SSH key, không dùng password
PasswordAuthentication no

# ✅ Thay đổi port SSH mặc định (giảm brute force bot)
Port 2222  # Thay vì 22

# ✅ Cài fail2ban để block IP brute force
sudo apt install fail2ban -y
# fail2ban tự block IP thử đăng nhập sai nhiều lần

# ✅ Không lưu credential trong code
# Dùng .env, secret manager
```

### 6.8.2. Object Storage

```
✅ Đặt bucket là private, dùng presigned URL cho file private
✅ Bật versioning để khôi phục file bị xóa nhầm
✅ Đặt lifecycle rule: tự xóa file temp sau 24h
✅ Validate MIME type phía server, không tin client
✅ Giới hạn kích thước file upload
✅ Scan virus trước khi cho user download (nếu là file quan trọng)
✅ Không đặt tên file theo input user (dùng UUID)
```

### 6.8.3. DNS

```
✅ TTL = 3600 (1h) bình thường, giảm xuống 300 trước khi migrate
✅ Dùng Cloudflare Proxy (cam) để ẩn IP server thật
✅ Bật DNSSEC nếu provider hỗ trợ
✅ Thêm SPF, DKIM, DMARC records nếu gửi email từ domain
✅ Giữ lại NS record cũ ít nhất 24h sau khi chuyển
```

---

## Tóm Tắt Chương 6

| Thành phần | Vai trò | Công cụ phổ biến |
|---|---|---|
| **VPS** | Server chạy ứng dụng | DigitalOcean, Hetzner, Vultr |
| **Object Storage** | Lưu file upload, assets | Cloudflare R2, S3, DO Spaces |
| **CDN** | Phân phối nội dung tĩnh nhanh | Cloudflare, CloudFront |
| **Domain** | Địa chỉ dễ nhớ của app | Namecheap, GoDaddy |
| **DNS** | Phân giải domain → IP | Cloudflare DNS, Route53 |
| **Cloud Database** | DB được managed, HA | Neon, Supabase, RDS |
| **Email Service** | Gửi email transactional | Resend, SendGrid |
| **Firewall** | Kiểm soát traffic vào ra | UFW, Security Group |
| **SSH Key** | Xác thực an toàn với server | ed25519 key pair |

---

> **Chương tiếp theo:** [Chương 7 — Monitoring](./chapter-07-monitoring.md)  
> Hạ tầng đã sẵn sàng và ứng dụng đang chạy. Câu hỏi tiếp theo: làm sao biết ứng dụng đang hoạt động tốt hay không? Chương 7 sẽ trả lời bằng Logging, Metrics, Health Check và Alerting — nền tảng để vận hành production như một professional.
