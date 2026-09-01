# Notification Flow — Recalio Backend

## Tổng quan

Hệ thống notification gồm 3 luồng chính:

1. **User tương tác API** — GET settings/notifications, POST tạo notification (admin)
2. **Cron tự động** — quét DB, tạo notification Study Reminder / Cards Due
3. **Queue xử lý bất đồng bộ** — gửi email qua `NotificationProcessor` (dùng chung queue `notification`)

---

## Danh sách file

### Module notifications

| File | Vai trò |
|------|---------|
| `src/modules/notifications/notification.constant.ts` | Hằng số (DEFAULT_LIMIT, MAX_LIMIT) |
| `src/modules/notifications/notification.controller.ts` | Route handler (GET, POST, PATCH) |
| `src/modules/notifications/notification.dto.ts` | DTO validate request / serialize response |
| `src/modules/notifications/notification.error.ts` | Error factory (notFound, channelNotAllowed) |
| `src/modules/notifications/notification.module.ts` | NestJS module wiring |
| `src/modules/notifications/notification.processor.ts` | Worker consume queue `notification` (gửi notif + email) |
| `src/modules/notifications/notification.repository.ts` | Prisma queries |
| `src/modules/notifications/notification.service.ts` | Business logic |
| `src/modules/notifications/notification.cron.ts` | Cron job: tạo notification + đẩy email vào queue |

### Infrastructure mail & queue

| File | Vai trò |
|------|---------|
| `src/infrastructures/mailer/mailer.service.ts` | Gửi email SMTP qua nodemailer |
| `src/infrastructures/mailer/mailer.module.ts` | Module mail (chỉ service) |
| `src/infrastructures/queue/queue.constant.ts` | Định nghĩa QueueName / JobName |
| `src/infrastructures/queue/queue.module.ts` | Đăng ký queue `add-note`, `notification` |
| `src/infrastructures/queue/producers/notification.producer.ts` | Push job vào queue `notification` |

### Config

| File | Vai trò |
|------|---------|
| `src/config/mailer.config.ts` | SMTP config (host, port, auth) |
| `src/config/queue.config.ts` | BullMQ default options (attempts, backoff) |
| `src/config/redis.config.ts` | Redis connection config |

---

## Luồng chi tiết

### 1. User GET /notification-settings

```
Controller → Service.getSettings() → Repository.getSettings()
```

**Controller** `notification.controller.ts:16`
```ts
@Get('notification-settings')
@ResponseMessage('Lấy cài đặt thông báo')
async getSettings(@CurrentUser('id') userId: string) {
    return this.service.getSettings(userId);
}
```
Lấy userId từ JWT, gọi service.

**Service** `notification.service.ts:26`
```ts
async getSettings(userId: string): Promise<NotificationSettingResponseDto> {
    const settings = await this.repo.getSettings(userId);
    return settings ?? { emailEnabled: true, pushEnabled: true, studyReminder: true, reminderTime: '20:00' };
}
```
Nếu chưa có setting → trả giá trị mặc định (tất cả bật, giờ nhắc 20:00).

**Repository** `notification.repository.ts:20`
```ts
async getSettings(userId: string) {
    return this.prisma.notificationSetting.findUnique({ where: { userId } });
}
```
Query table `notification_settings` theo userId.

---

### 2. User GET /notifications

```
Controller → Service.list() → Repository.findMany()
```

**Controller** `notification.controller.ts:30`
```ts
@Get('notifications')
@ResponseMessage('Lấy danh sách thông báo')
async list(@CurrentUser('id') userId: string, @Query() dto: NotificationQueryDto) {
    return this.service.list(userId, dto);
}
```
Query params: `type`, `isRead`, `page`, `limit`.

**Service** `notification.service.ts:37`
```ts
async list(userId: string, dto: NotificationQueryDto) {
    const page = dto.page ?? 1;
    const limit = dto.limit ?? NOTIFICATION_CONSTANTS.DEFAULT_LIMIT;

    const where: Record<string, unknown> = {};
    if (dto.type) where.type = dto.type;
    if (dto.isRead !== undefined) where.isRead = dto.isRead;

    const { items, total } = await this.repo.findMany(userId, where as any, page, limit);
    return paginate(items, total, { page, limit } as PaginationDto);
}
```
Xây dynamic `where` từ query params. Gọi `paginate()` wrapper trả về `{ data, meta }`.

**Repository** `notification.repository.ts:33`
```ts
async findMany(userId: string, where?, page = 1, limit = 20) {
    const baseWhere: Prisma.NotificationWhereInput = { userId, ...(where ?? {}) };

    const [items, total] = await Promise.all([
        this.prisma.notification.findMany({
            where: baseWhere,
            orderBy: { sentAt: 'desc' },
            skip: (page - 1) * limit,
            take: limit,
            select: notifSelect,
        }),
        this.prisma.notification.count({ where: baseWhere }),
    ]);
    return { items, total };
}
```
`Promise.all([findMany, count])` — chạy song song. `notifSelect` là Prisma select object khai báo đầu file, không include `userId` để tránh leak.

---

### 3. User GET /notifications/unread-count

**Controller** `notification.controller.ts:37`
```ts
@Get('notifications/unread-count')
@ResponseMessage('Lấy số thông báo chưa đọc')
async countUnread(@CurrentUser('id') userId: string) {
    return this.service.countUnread(userId);
}
```

**Service** `notification.service.ts:61`
```ts
async countUnread(userId: string) {
    return { unread: await this.repo.countUnread(userId) };
}
```

**Repository** `notification.repository.ts:72`
```ts
async countUnread(userId: string) {
    return this.prisma.notification.count({
        where: { userId, isRead: false },
    });
}
```

---

### 4. Admin POST /notifications

```
Controller → Service.createNotification() → Producer.addSendJob() / addBroadcastJob()
                                                      ↓
                                         NotificationProcessor.process()
```

**Controller** `notification.controller.ts:60`
```ts
@Post('notifications')
@Roles('ADMIN')
@ResponseMessage('Gửi thông báo')
async create(@CurrentUser('id') adminId: string, @Body() dto: CreateNotificationDto) {
    return this.service.createNotification(adminId, dto);
}
```
Yêu cầu role `ADMIN`. Body gồm `type?`, `title`, `body?`, `data?`, `channel?`, `userId?`.

**Service** `notification.service.ts:65`
```ts
async createNotification(adminId: string, dto: CreateNotificationDto) {
    const { userId, type: rawType, title, body, data, channel } = dto;
    const type = rawType ?? NotificationType.SYSTEM;

    const resolvedChannel = channel ?? (userId ? 'WEB_PUSH' : 'EMAIL');
    if (resolvedChannel === 'EMAIL' && !this.EMAIL_ELIGIBLE_TYPES.has(type)) {
        throw NotificationError.channelNotAllowed();
    }

    if (userId) {
        await this.notificationProducer.addSendJob({ userId, type, title, body, data, channel: resolvedChannel });
        return { queued: true };
    }

    await this.notificationProducer.addBroadcastJob({ adminId, type, title, body, data, channel: resolvedChannel });
    return { queued: true };
}
```
- Nếu có `userId` → gửi riêng (queue notification → insert DB).
- Nếu không có `userId` → broadcast tới user đủ điều kiện (queue notification → insert Many).
- Không cho phép gửi STUDY_REMINDER/CARDS_DUE qua EMAIL từ API admin (chỉ cron mới tạo).

**Producer** `producers/notification.producer.ts:36`
```ts
async addSendJob(data: SendNotificationJobData) {
    return this.queue.add(JobName.SEND_NOTIFICATION, data, {
        attempts: 3, backoff: { type: 'exponential', delay: 2000 },
        removeOnComplete: { count: 1000 }, removeOnFail: { count: 5000 },
    });
}
```
Push job `send-notification` vào queue `notification` với retry 3 lần, backoff 2s.

**Processor** `notification.processor.ts:18`
```ts
case JobName.SEND_NOTIFICATION: {
    const data = job.data as SendNotificationJobData;
    await this.repo.createOne({
        type, title, body: body ?? '',
        data: (payload ?? {}) as any,
        channel: channel ?? 'WEB_PUSH',
        user: { connect: { id: userId } },
    });
    break;
}
```
Insert 1 notification vào DB.

---

### 5. Cron — Study Reminder (mỗi phút)

```
Cron → findUsersWithStudyReminder() → createOne()
       → sendPendingEmails() → NotificationProducer.addEmailJob()
                                → NotificationProcessor → MailerService.sendMail()
```

**Cron** `notification.cron.ts:16`
```ts
@Cron(CronExpression.EVERY_MINUTE)
async createStudyReminders() {
    const now = new Date();
    const userIds = await this.repo.findUsersWithStudyReminder(now);
    if (!userIds.length) return;

    const today = new Date(now.getFullYear(), now.getMonth(), now.getDate());
    for (const userId of userIds) {
        const exists = await this.repo.hasRecentNotification(userId, 'STUDY_REMINDER', today);
        if (exists) continue;
        await this.repo.createOne({
            type: NotificationType.STUDY_REMINDER,
            title: 'Đã đến giờ học',
            body: 'Hãy dành chút thời gian ôn tập từ vựng nhé!',
            channel: 'EMAIL' as any,
            user: { connect: { id: userId } },
        });
    }
}
```
- Chạy mỗi phút.
- Tìm user có timezone = 20:00 local, studyReminder + emailEnabled = true.
- Chỉ tạo 1 notification/user/ngày (hasRecentNotification check).

---

### 6. Cron — Cards Due (30 phút / lần)

**Cron** `notification.cron.ts:45`
```ts
@Cron(CronExpression.EVERY_30_MINUTES)
async createCardsDueNotifications() {
    const userIds = await this.repo.findUsersWithDueCards();
    // ... tìm user có card due, tạo notification với count
}
```

---

### 7. Cron — Send Pending Emails (mỗi phút)

**Cron** `notification.cron.ts:77`
```ts
@Cron(CronExpression.EVERY_MINUTE)
async sendPendingEmails() {
    const notifications = await this.repo.findUnsentEmails();
    if (!notifications.length) return;

    // Group theo user, tạo HTML template
    const grouped = new Map</* ... */>();
    for (const n of notifications) { /* group by user.id */ }

    // Mark sent ngay tránh cron chạy lại
    const allIds = notifications.map((n) => n.id);
    await this.repo.markEmailSent(allIds);

    for (const [, group] of grouped) {
        const html = buildEmailHtml(group);
        await this.notificationProducer.addEmailJob({
            email: group.email,
            subject: 'Thông báo từ Recalio',
            html,
            notificationIds: group.ids,
        });
    }
}
```
- Quét notification có `channel = EMAIL`, `emailSentAt = null`.
- Mark `emailSentAt` ngay lập tức để cron sau không quét lại.
- Group theo user → build HTML → push job `send-email-notification` vào queue `notification`.

**Processor** `notification.processor.ts:57` — xử lý job email
```ts
case JobName.SEND_EMAIL_NOTIFICATION: {
    const data = job.data as SendEmailNotificationJobData;
    const { email, subject, html, notificationIds } = data;

    const ok = await this.mailer.sendMail(email, subject, html);
    if (ok) {
        await this.repo.markEmailSent(notificationIds);
    } else {
        throw new Error(`Failed to send email to ${email}`);
    }
    break;
}
```

**MailerService** `mailer/mailer.service.ts:21`
```ts
async sendMail(to: string, subject: string, html: string): Promise<boolean> {
    if (!mailerConfig.auth.user) { /* skip + warn */ return false; }
    try {
        await this.transporter.sendMail({
            from: `"${mailerConfig.from.name}" <${mailerConfig.from.address}>`,
            to, subject, html,
        });
        return true;
    } catch (err) {
        this.logger.error(`Failed to send email to ${to}: ${(err as Error).message}`);
        return false;
    }
}
```

---

## Sơ đồ luồng

```
┌──────────────┐
│   Cron job   │◄── mỗi phút / 30 phút
│  (schedule)  │
└──────┬───────┘
       │ findUnsentEmails / createReminders / createCardsDue
       ▼
┌──────────────┐     push job       ┌──────────────────────────┐
│  Repository   │──────────────────►│  NotificationProducer   │
│  (Prisma)     │                   │  (queue: notification)  │
└──────┬───────┘                    └───────────┬──────────────┘
       │ create notification                    │ consume
       ▼                                        ▼
┌──────────────┐                    ┌──────────────────────────┐
│ Notification  │                    │  NotificationProcessor  │
│ (DB table)    │                    │  (WorkerHost)           │
└──────────────┘                    │  ├─ SEND_NOTIFICATION   │
                                    │  ├─ BROADCAST_NOTIF     │
                                    │  └─ SEND_EMAIL_NOTIF    │
                                    │       │                 │
                                    │       ▼                 │
                                    │  MailerService.sendMail │
                                    └──────────────────────────┘

┌──────────────┐     POST /notifications
│   Client     │─────────────────────►┌──────────────────┐
│   (Admin)    │                      │  Notification    │
└──────────────┘                      │  Controller      │
                                      └────────┬─────────┘
                                               │ createNotification()
                                               ▼
                                      ┌──────────────────┐
                                      │  Notification    │
                                      │  Service         │
                                      └────────┬─────────┘
                                               │ addSendJob / addBroadcastJob
                                               ▼
                                      ┌──────────────────┐
                                      │  Notification    │
                                      │  Producer        │
                                      │  (queue: notif)  │
                                      └────────┬─────────┘
                                               │ consume
                                               ▼
                                      ┌──────────────────┐
                                      │  Notification    │
                                      │  Processor       │
                                      │  → insert DB     │
                                      └──────────────────┘
```
