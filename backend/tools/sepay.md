# SePay – Tổng hợp kiến thức (đặc biệt cho NestJS)

> Tài liệu chính thức: https://docs.sepay.vn  
> API endpoint: https://my.sepay.vn/userapi

---

## 1. SePay là gì?

SePay là nền tảng **thanh toán chuyển khoản ngân hàng tự động** tại Việt Nam. Thay vì tích hợp từng ngân hàng, bạn chỉ cần kết nối với SePay — SePay đứng giữa làm cầu nối với 18+ ngân hàng lớn.

**SePay làm được gì:**
- Nhận webhook real-time khi có tiền vào/ra tài khoản ngân hàng
- Tạo QR Code VietQR động (có số tiền + nội dung tự động điền)
- API truy vấn giao dịch ngân hàng
- Virtual Account (VA) theo đơn hàng — mỗi đơn 1 STK ảo riêng
- Chia sẻ biến động số dư qua Telegram, Viber, Lark

**Luồng thanh toán cơ bản:**
```
Khách đặt hàng → Backend tạo đơn + mã thanh toán
→ Hiển thị QR cho khách quét
→ Khách chuyển khoản
→ Ngân hàng báo SePay
→ SePay bắn Webhook về backend
→ Backend cập nhật đơn hàng = Paid
```

---

## 2. Authentication – API Token

API Token cần được đưa vào header mỗi khi request đến SePay API với cấu trúc `Authorization: Bearer API_TOKEN`.

**Tạo API Token:** Vào https://my.sepay.vn → Settings → API Token

```typescript
// sepay.config.ts
export const SEPAY_CONFIG = {
  BASE_URL: 'https://my.sepay.vn/userapi',
  API_TOKEN: process.env.SEPAY_API_TOKEN,
  WEBHOOK_API_KEY: process.env.SEPAY_WEBHOOK_API_KEY,
};

// Header mẫu
const headers = {
  'Content-Type': 'application/json',
  'Authorization': `Bearer ${process.env.SEPAY_API_TOKEN}`,
};
```

---

## 3. Webhook – Nhận thông báo giao dịch

Đây là tính năng **quan trọng nhất** khi tích hợp SePay. Webhook giúp SePay push thông tin giao dịch ngay khi phát sinh.

### 3.1. Payload Webhook từ SePay

SePay gửi HTTP POST với body JSON khi có giao dịch:

```json
{
  "id": 92704,
  "gateway": "Vietcombank",
  "transactionDate": "2024-07-02 11:08:33",
  "accountNumber": "1017588888",
  "subAccount": "",
  "code": "SEVN63DC8E5C",
  "content": "SEVN63DC8E5C chuyen tien",
  "transferType": "in",
  "description": "NGUYEN VAN A chuyen tien",
  "transferAmount": 5000000,
  "accumulated": 105000000,
  "referenceCode": "FT24012345678"
}
```

**Giải thích các trường:**

| Trường | Kiểu | Ý nghĩa |
|---|---|---|
| `id` | number | ID giao dịch duy nhất trên SePay (dùng để chống duplicate) |
| `gateway` | string | Tên ngân hàng (Vietcombank, MB, Techcombank...) |
| `transactionDate` | string | Thời điểm giao dịch |
| `accountNumber` | string | Số tài khoản thật |
| `subAccount` | string | Tài khoản ảo (VA) nếu có |
| `code` | string | Mã giao dịch SePay (thường có trong nội dung CK) |
| `content` | string | Nội dung chuyển khoản đầy đủ |
| `transferType` | string | `"in"` = tiền vào, `"out"` = tiền ra |
| `transferAmount` | number | Số tiền (VND, không có decimal) |
| `accumulated` | number | Số dư tích lũy sau giao dịch |
| `referenceCode` | string | Mã tham chiếu ngân hàng |

### 3.2. Yêu cầu response

SePay tính là thành công khi endpoint trả về HTTP status 200 hoặc 201, body JSON có `{"success": true}`, và hoàn tất trong 30 giây.

### 3.3. Xác thực Webhook (API Key)

Với chứng thực API Key, SePay gửi header `"Authorization": "Apikey API_KEY_CUA_BAN"`.

### 3.4. Retry tự động

Khi endpoint trả lỗi, SePay tự gọi lại tối đa 7 lần, tối đa 5 giờ kể từ lần đầu thất bại, với thời gian giữa các lần tăng theo dãy Fibonacci.

### 3.5. IP Whitelist của SePay

Danh sách IPv4 gửi webhook của SePay: `172.236.138.20`, `172.233.83.68`, `171.244.35.2`, `151.158.108.68`, `151.158.109.79`, `103.255.238.139`.

---

## 4. Tích hợp Webhook trong NestJS

### 4.1. DTO

```typescript
// dto/sepay-webhook.dto.ts
import { IsNumber, IsString, IsIn, IsOptional } from 'class-validator';

export class SepayWebhookDto {
  @IsNumber()
  id: number;

  @IsString()
  gateway: string;

  @IsString()
  transactionDate: string;

  @IsString()
  accountNumber: string;

  @IsOptional()
  @IsString()
  subAccount: string | null;

  @IsOptional()
  @IsString()
  code: string | null;

  @IsString()
  content: string;

  @IsIn(['in', 'out'])
  transferType: 'in' | 'out';

  @IsNumber()
  transferAmount: number;

  @IsNumber()
  accumulated: number;

  @IsOptional()
  @IsString()
  referenceCode: string | null;
}
```

### 4.2. Guard xác thực API Key

```typescript
// guards/sepay-webhook.guard.ts
import { CanActivate, ExecutionContext, Injectable, UnauthorizedException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Request } from 'express';

@Injectable()
export class SepayWebhookGuard implements CanActivate {
  constructor(private config: ConfigService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest<Request>();
    const authHeader = request.headers['authorization'];
    const expectedKey = `Apikey ${this.config.get('SEPAY_WEBHOOK_API_KEY')}`;

    if (!authHeader || authHeader !== expectedKey) {
      throw new UnauthorizedException('Invalid SePay webhook key');
    }
    return true;
  }
}
```

### 4.3. Controller

```typescript
// sepay.controller.ts
import { Controller, Post, Body, UseGuards, HttpCode, Logger } from '@nestjs/common';
import { SepayWebhookGuard } from './guards/sepay-webhook.guard';
import { SepayWebhookDto } from './dto/sepay-webhook.dto';
import { SepayService } from './sepay.service';

@Controller('webhooks/sepay')
export class SepayController {
  private readonly logger = new Logger(SepayController.name);

  constructor(private readonly sepayService: SepayService) {}

  @Post()
  @HttpCode(200)
  @UseGuards(SepayWebhookGuard)
  async handleWebhook(@Body() payload: SepayWebhookDto) {
    this.logger.log(`Webhook received: id=${payload.id}, amount=${payload.transferAmount}`);

    await this.sepayService.processWebhook(payload);

    // BẮT BUỘC trả về đúng format này
    return { success: true };
  }
}
```

### 4.4. Service xử lý nghiệp vụ

```typescript
// sepay.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { SepayWebhookDto } from './dto/sepay-webhook.dto';
import { Order } from '../orders/order.entity';
import { Transaction } from './transaction.entity';

@Injectable()
export class SepayService {
  private readonly logger = new Logger(SepayService.name);

  constructor(
    @InjectRepository(Order) private orderRepo: Repository<Order>,
    @InjectRepository(Transaction) private txRepo: Repository<Transaction>,
  ) {}

  async processWebhook(payload: SepayWebhookDto): Promise<void> {
    // BƯỚC 1: Chống duplicate — kiểm tra id đã xử lý chưa
    const existing = await this.txRepo.findOne({ where: { sepayId: payload.id } });
    if (existing) {
      this.logger.warn(`Duplicate webhook id=${payload.id}, skip`);
      return;
    }

    // BƯỚC 2: Chỉ xử lý tiền VÀO
    if (payload.transferType !== 'in') return;

    // BƯỚC 3: Lưu transaction vào DB
    await this.txRepo.save({
      sepayId: payload.id,
      gateway: payload.gateway,
      accountNumber: payload.accountNumber,
      amount: payload.transferAmount,
      content: payload.content,
      transferType: payload.transferType,
      transactionDate: new Date(payload.transactionDate),
      referenceCode: payload.referenceCode,
      rawPayload: JSON.stringify(payload),
    });

    // BƯỚC 4: Tìm đơn hàng từ nội dung chuyển khoản
    const orderId = this.extractOrderId(payload.content);
    if (!orderId) {
      this.logger.warn(`Cannot extract orderId from content: "${payload.content}"`);
      return;
    }

    const order = await this.orderRepo.findOne({ where: { id: orderId } });
    if (!order) {
      this.logger.warn(`Order not found: ${orderId}`);
      return;
    }

    // BƯỚC 5: Kiểm tra số tiền
    if (order.amount !== payload.transferAmount) {
      this.logger.warn(`Amount mismatch: expected ${order.amount}, got ${payload.transferAmount}`);
      return;
    }

    // BƯỚC 6: Cập nhật trạng thái đơn hàng
    if (order.paymentStatus === 'paid') {
      this.logger.warn(`Order ${orderId} already paid`);
      return;
    }

    await this.orderRepo.update(orderId, {
      paymentStatus: 'paid',
      paidAt: new Date(),
      sepayTransactionId: payload.id,
    });

    this.logger.log(`Order ${orderId} marked as PAID`);

    // BƯỚC 7: Trigger các action tiếp theo (gửi email, kích hoạt dịch vụ...)
    // await this.emailService.sendPaymentConfirmation(order);
  }

  // Trích xuất mã đơn hàng từ nội dung CK
  // Ví dụ: "DH2024001 thanh toan don hang" → "DH2024001"
  private extractOrderId(content: string): string | null {
    const match = content.match(/DH\d{7}/i);
    return match ? match[0].toUpperCase() : null;
  }
}
```

### 4.5. Module

```typescript
// sepay.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule } from '@nestjs/config';
import { SepayController } from './sepay.controller';
import { SepayService } from './sepay.service';
import { SepayApiService } from './sepay-api.service';
import { Order } from '../orders/order.entity';
import { Transaction } from './transaction.entity';

@Module({
  imports: [
    ConfigModule,
    TypeOrmModule.forFeature([Order, Transaction]),
  ],
  controllers: [SepayController],
  providers: [SepayService, SepayApiService],
  exports: [SepayService, SepayApiService],
})
export class SepayModule {}
```

---

## 5. QR Code VietQR

SePay cung cấp công cụ tạo ảnh QR Code động tại `qr.sepay.vn`. Cấu trúc link nhúng:

```
https://qr.sepay.vn/img?acc=SO_TAI_KHOAN&bank=NGAN_HANG&amount=SO_TIEN&des=NOI_DUNG
```

Các tham số: `acc` (bắt buộc) là số tài khoản, `bank` (bắt buộc) là tên ngân hàng, `amount` (không bắt buộc) là số tiền, `des` (không bắt buộc) là nội dung chuyển khoản.

### Tạo QR URL trong NestJS

```typescript
// sepay-api.service.ts
@Injectable()
export class SepayApiService {
  generateQrUrl(params: {
    accountNumber: string;
    bankName: string;
    amount?: number;
    description?: string;
  }): string {
    const url = new URL('https://qr.sepay.vn/img');
    url.searchParams.set('acc', params.accountNumber);
    url.searchParams.set('bank', params.bankName);
    if (params.amount) url.searchParams.set('amount', String(params.amount));
    if (params.description) url.searchParams.set('des', params.description);
    return url.toString();
  }

  // Tạo nội dung chuyển khoản duy nhất cho từng đơn hàng
  generatePaymentContent(orderId: string): string {
    return `DH${orderId.padStart(7, '0')}`;
  }

  // Tạo QR hoàn chỉnh cho đơn hàng
  generateOrderQr(order: { id: string; amount: number }): string {
    const content = this.generatePaymentContent(order.id);
    return this.generateQrUrl({
      accountNumber: process.env.BANK_ACCOUNT_NUMBER,
      bankName: process.env.BANK_NAME,  // ví dụ: 'Vietcombank'
      amount: order.amount,
      description: content,
    });
  }
}
```

### Nhúng QR vào response API

```typescript
@Get('orders/:id/payment')
async getPaymentInfo(@Param('id') id: string) {
  const order = await this.orderService.findById(id);
  const qrUrl = this.sepayApiService.generateOrderQr(order);
  const content = this.sepayApiService.generatePaymentContent(order.id);

  return {
    orderId: order.id,
    amount: order.amount,
    bankAccount: process.env.BANK_ACCOUNT_NUMBER,
    bankName: process.env.BANK_NAME,
    content,           // nội dung KH phải nhập khi CK
    qrCodeUrl: qrUrl, // URL ảnh QR để hiển thị
    expiredAt: new Date(Date.now() + 15 * 60 * 1000), // hết hạn sau 15 phút
  };
}
```

---

## 6. SePay API – Truy vấn giao dịch

Dùng để **chủ động kiểm tra** giao dịch (polling fallback khi webhook fail).

### 6.1. Lấy chi tiết một giao dịch

```
GET https://my.sepay.vn/userapi/transactions/details/{transaction_id}
Authorization: Bearer API_TOKEN
```

Response trả về thông tin chi tiết giao dịch bao gồm: `id`, `transaction_date`, `account_number`, `sub_account`, `amount_in`, `amount_out`, `accumulated`, `transaction_content`, `reference_number`, `bank_brand_name`, `bank_account_id`.

### 6.2. Lấy danh sách giao dịch

```
GET https://my.sepay.vn/userapi/transactions/list
```

Các filter có thể dùng: `account_number`, `transaction_date_min`, `transaction_date_max`, `since_id`, `limit` (tối đa 5000), `reference_number`, `amount_in`, `amount_out`.

### 6.3. Service gọi API

```typescript
// sepay-api.service.ts (bổ sung)
import { HttpService } from '@nestjs/axios';
import { firstValueFrom } from 'rxjs';

@Injectable()
export class SepayApiService {
  private readonly BASE_URL = 'https://my.sepay.vn/userapi';

  constructor(
    private http: HttpService,
    private config: ConfigService,
  ) {}

  private get headers() {
    return {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${this.config.get('SEPAY_API_TOKEN')}`,
    };
  }

  // Lấy chi tiết 1 giao dịch
  async getTransaction(transactionId: number) {
    const { data } = await firstValueFrom(
      this.http.get(`${this.BASE_URL}/transactions/details/${transactionId}`, {
        headers: this.headers,
      }),
    );
    return data.transaction;
  }

  // Lấy danh sách giao dịch
  async getTransactions(params?: {
    accountNumber?: string;
    sinceId?: number;
    limit?: number;
    dateMin?: string;  // 'yyyy-mm-dd'
    dateMax?: string;
    amountIn?: number;
  }) {
    const query = new URLSearchParams();
    if (params?.accountNumber) query.set('account_number', params.accountNumber);
    if (params?.sinceId) query.set('since_id', String(params.sinceId));
    if (params?.limit) query.set('limit', String(params.limit));
    if (params?.dateMin) query.set('transaction_date_min', params.dateMin);
    if (params?.dateMax) query.set('transaction_date_max', params.dateMax);
    if (params?.amountIn) query.set('amount_in', String(params.amountIn));

    const { data } = await firstValueFrom(
      this.http.get(`${this.BASE_URL}/transactions/list?${query}`, {
        headers: this.headers,
      }),
    );
    return data.transactions;
  }

  // Polling fallback: kiểm tra đơn hàng đã thanh toán chưa
  async checkOrderPaid(amount: number, content: string): Promise<boolean> {
    const transactions = await this.getTransactions({
      amountIn: amount,
      dateMin: new Date(Date.now() - 24 * 60 * 60 * 1000)
        .toISOString().split('T')[0],
    });

    return transactions.some((tx) =>
      tx.transaction_content?.includes(content)
    );
  }
}
```

---

## 7. Virtual Account (VA) theo Đơn hàng

API VA (Virtual Account) theo đơn hàng là giải pháp tự động hóa xác nhận thanh toán. Thay vì sử dụng một VA cố định, mỗi đơn hàng sẽ được cấp một VA riêng với số tiền khớp chính xác.

**Ưu điểm so với QR thường:**
- VA chỉ chấp nhận thanh toán đúng số tiền của đơn hàng, nếu khách hàng chuyển sai số tiền thì app bank sẽ báo lỗi ngay lập tức.
- Mỗi đơn hàng có một VA riêng nên không phụ thuộc vào nội dung chuyển khoản, tránh được các lỗi do khách hàng điền sai nội dung.
- VA tự động hủy sau khi thanh toán thành công hoặc hết hạn, tránh hoàn toàn tình trạng thanh toán trùng lặp.

Hiện tại VA theo đơn hàng hỗ trợ **BIDV doanh nghiệp**. Xem tài liệu chi tiết: https://docs.sepay.vn/api-va-theo-don-hang-bidv.html

---

## 8. Entity & Migration

```typescript
// transaction.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn, Index } from 'typeorm';

@Entity('sepay_transactions')
export class Transaction {
  @PrimaryGeneratedColumn()
  id: number;

  @Index({ unique: true })  // QUAN TRỌNG: chống duplicate
  @Column({ name: 'sepay_id', unique: true })
  sepayId: number;

  @Column()
  gateway: string;

  @Column({ name: 'account_number' })
  accountNumber: string;

  @Column({ name: 'sub_account', nullable: true })
  subAccount: string | null;

  @Column({ type: 'bigint' })
  amount: number;

  @Column({ name: 'transfer_type', length: 3 })
  transferType: 'in' | 'out';

  @Column({ type: 'text', nullable: true })
  content: string;

  @Column({ name: 'reference_code', nullable: true })
  referenceCode: string | null;

  @Column({ name: 'transaction_date', type: 'datetime' })
  transactionDate: Date;

  @Column({ name: 'raw_payload', type: 'text', nullable: true })
  rawPayload: string;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;
}
```

---

## 9. Patterns & Best Practices

### 9.1. Chống duplicate (quan trọng nhất)

Cùng một giao dịch có thể nhận được webhook nhiều lần do retry tự động, gửi lại thủ công, hoặc nhiều webhook trỏ về cùng endpoint. Cần kiểm tra trùng dựa trên trường `id` trước khi xử lý.

```typescript
// Cách 1: Check trước khi save
const existing = await this.txRepo.findOne({ where: { sepayId: payload.id } });
if (existing) return; // idempotent

// Cách 2: Dùng INSERT IGNORE (MySQL) — thông qua QueryBuilder
await this.txRepo
  .createQueryBuilder()
  .insert()
  .into(Transaction)
  .values({ sepayId: payload.id, ... })
  .orIgnore()  // INSERT IGNORE
  .execute();
```

### 9.2. Mã hóa nội dung chuyển khoản

```typescript
// Pattern nội dung chuyển khoản rõ ràng, dễ tìm kiếm
const CONTENT_PREFIX = 'DH';  // Đơn Hàng

function generateContent(orderId: string): string {
  return `${CONTENT_PREFIX}${orderId.padStart(7, '0')}`;
  // → "DH0001234"
}

function extractOrderId(content: string): string | null {
  const match = content.match(/DH(\d{7})/i);
  return match ? String(parseInt(match[1])) : null;
}
```

### 9.3. Xử lý bất đồng bộ với BullMQ

Webhook cần trả về `{ success: true }` trong 30s — delegate xử lý nặng sang queue:

```typescript
@Post()
@HttpCode(200)
@UseGuards(SepayWebhookGuard)
async handleWebhook(@Body() payload: SepayWebhookDto) {
  // Lưu transaction nhanh để chống duplicate
  const isNew = await this.sepayService.saveTransaction(payload);

  if (isNew && payload.transferType === 'in') {
    // Đẩy sang queue xử lý async
    await this.paymentQueue.add('process-payment', payload, {
      attempts: 3,
      backoff: { type: 'exponential', delay: 2000 },
    });
  }

  // Trả về ngay trong vòng vài ms
  return { success: true };
}
```

### 9.4. Polling fallback

Khi webhook thất bại, dùng cron job để polling lại:

```typescript
// payment-polling.service.ts
@Injectable()
export class PaymentPollingService {
  @Cron('*/5 * * * *') // mỗi 5 phút
  async checkPendingOrders() {
    const pendingOrders = await this.orderRepo.find({
      where: { paymentStatus: 'pending', createdAt: MoreThan(subHours(new Date(), 2)) },
    });

    for (const order of pendingOrders) {
      const content = this.sepayApiService.generatePaymentContent(order.id);
      const paid = await this.sepayApiService.checkOrderPaid(order.amount, content);
      if (paid) {
        await this.orderRepo.update(order.id, { paymentStatus: 'paid' });
      }
    }
  }
}
```

### 9.5. Biến môi trường

```env
# .env
SEPAY_API_TOKEN=your_api_token_here
SEPAY_WEBHOOK_API_KEY=your_webhook_api_key_here
BANK_ACCOUNT_NUMBER=1017588888
BANK_NAME=Vietcombank
PAYMENT_CONTENT_PREFIX=DH
```

---

## 10. Kiểm thử Webhook

Trên trang chi tiết webhook, có tính năng **Gửi thử** để bắn payload mẫu sang URL của bạn mà không cần phát sinh giao dịch thật. Sau đó vào Nhật ký webhooks để xem các lần đã gửi.

**Dùng ngrok để test local:**
```bash
ngrok http 3000
# → Lấy URL https://abc123.ngrok.io
# → Cấu hình Webhook URL: https://abc123.ngrok.io/webhooks/sepay
```

**Test thủ công bằng curl:**
```bash
curl -X POST http://localhost:3000/webhooks/sepay \
  -H "Content-Type: application/json" \
  -H "Authorization: Apikey YOUR_WEBHOOK_API_KEY" \
  -d '{
    "id": 99999,
    "gateway": "Vietcombank",
    "transactionDate": "2024-07-02 11:08:33",
    "accountNumber": "1017588888",
    "subAccount": null,
    "code": "TEST001",
    "content": "DH0001234 thanh toan don hang",
    "transferType": "in",
    "transferAmount": 250000,
    "accumulated": 1000000,
    "referenceCode": "TEST123"
  }'
```

---

## 11. Checklist trước khi deploy

- [ ] Xác thực webhook bằng API Key (header `Authorization: Apikey ...`)
- [ ] Chống duplicate dựa trên `id` của webhook payload
- [ ] Trả về `{ success: true }` với status 200 trong vòng 30s
- [ ] Lưu toàn bộ `rawPayload` để debug sau này
- [ ] Chỉ xử lý `transferType === "in"`
- [ ] Kiểm tra đúng số tiền trước khi mark paid
- [ ] Có polling fallback khi webhook thất bại
- [ ] Whitelist IP của SePay nếu server có firewall
- [ ] Không để API Token trong code — dùng `.env`
- [ ] Log đầy đủ: nhận webhook, lý do skip, kết quả xử lý
