# Phần 12: Notification & Background Job (Scheduler)

Hướng dẫn về module **notification** (thông báo realtime qua SSE) và **scheduler** (công việc nền chạy định kỳ) trong project.

## 12.1. Tổng quan

Module `notification` có 2 vai trò:

1. **Lưu & quản lý thông báo** — lưu DB, lấy danh sách, đánh dấu đã đọc.
2. **Đẩy thông báo realtime** — dùng **SSE (Server-Sent Events)** gửi ngay tới user đang online.
3. **Sinh thông báo tự động** — `NotificationScheduler` chạy nền mỗi phút để nhắc lịch làm/ca làm.

Cấu trúc package:

```
notification/
├── controller/NotificationController.java   ← REST + SSE endpoint
├── domain/Notification.java                  ← Entity
├── domain/NotificationType.java              ← enum loại thông báo
├── dto/NotificationResponse.java
├── mapper/NotificationMapper.java
├── repository/NotificationRepository.java
├── service/NotificationService.java          ← logic + SSE emitter
└── scheduler/NotificationScheduler.java      ← @Scheduled job
```

## 12.2. Bật Scheduling & Async

- `@EnableScheduling` được bật ở **`<ProjectName>Application`** → các method `@Scheduled` tự chạy.
- `@EnableAsync` bật trong `AuditConfiguration` (`shared/audit/config`) → dùng chung cho `@Async` khi cần xử lý nền (VD ghi audit async).

## 12.3. Entity & enum

### `NotificationType` — loại thông báo

```java
public enum NotificationType {
    LEAVE_APPROVED,   // nghỉ phép được duyệt
    LEAVE_REJECTED,   // nghỉ phép bị từ chối
    REMINDER,         // nhắc nhở
    SALARY,           // lương
    SYSTEM            // hệ thống
}
```

### `Notification` — bảng `notifications`

```java
@Entity
@Table(name = "notifications")
public class Notification {
    @Id @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;                       // user nhận thông báo

    @Enumerated(EnumType.STRING)
    @Column(name = "type", nullable = false, length = 30)
    private NotificationType type;

    @Column(name = "title", nullable = false)
    private String title;

    @Column(name = "message", columnDefinition = "TEXT", nullable = false)
    private String message;

    @Column(name = "is_read")
    @Builder.Default
    private Boolean isRead = false;

    @CreationTimestamp
    @Column(name = "created_at")
    private Instant createdAt;
}
```

> Lưu ý: `Notification` **không** kế thừa base entity soft delete — thông báo là dữ liệu lịch sử, không cần xóa mềm.

### Repository

```java
public interface NotificationRepository extends JpaRepository<Notification, UUID> {
    Page<Notification> findByUserIdOrderByCreatedAtDesc(UUID userId, Pageable pageable);
    long countByUserIdAndIsReadFalse(UUID userId);
}
```

## 12.4. SSE — đẩy thông báo realtime

`NotificationService` giữ **map `userId → SseEmitter`** (`ConcurrentHashMap`) để nhớ ai đang online.

### Đăng ký nhận — `subscribe(userId)`

```java
private final Map<UUID, SseEmitter> emitters = new ConcurrentHashMap<>();

public SseEmitter subscribe(UUID userId) {
    SseEmitter emitter = new SseEmitter(30 * 60 * 1000L);   // timeout 30 phút
    emitters.put(userId, emitter);

    // Xóa emitter khi kết thúc/timeout/lỗi
    emitter.onCompletion(() -> emitters.remove(userId));
    emitter.onTimeout(() -> emitters.remove(userId));
    emitter.onError((e) -> emitters.remove(userId));

    try {
        emitter.send(SseEmitter.event().name("INIT").data("Connected"));
    } catch (IOException e) {
        emitters.remove(userId);
    }
    return emitter;
}
```

### Gửi — `sendNotification(...)`

```java
@Transactional
public void sendNotification(UUID userId, NotificationType type, String title, String message) {
    User user = userRepository.findById(userId)
            .orElseThrow(() -> new IllegalArgumentException("User not found: " + userId));

    Notification notification = Notification.builder()
            .user(user).type(type).title(title).message(message).isRead(false)
            .build();
    Notification saved = notificationRepository.save(notification);
    NotificationResponse response = notificationMapper.toResponse(saved);

    // Nếu user đang online → đẩy ngay qua SSE
    SseEmitter emitter = emitters.get(userId);
    if (emitter != null) {
        try {
            emitter.send(SseEmitter.event().name("NOTIFICATION").data(response));
        } catch (IOException e) {
            log.error("Error sending notification to user {}: {}", userId, e.getMessage());
            emitters.remove(userId);
        }
    }
    // User offline → thông báo vẫn được lưu DB, lấy khi user đăng nhập lại
}
```

> Cơ chế: thông báo **luôn lưu DB**, SSE chỉ là kênh đẩy "tức thì" cho user online. User offline khi đăng nhập lại vẫn xem được qua API.

## 12.5. Controller — endpoint

```java
@RestController
@RequestMapping("/api/v1/notifications")
public class NotificationController {

    // GET /api/v1/notifications/stream — SSE, lấy user từ JWT
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter streamNotifications(@AuthenticationPrincipal UserPrincipal userDetails) {
        if (userDetails == null) throw new IllegalArgumentException("User not authenticated");
        return notificationService.subscribe(userDetails.getId());
    }

    // GET /api/v1/notifications — danh sách (phân trang)
    @GetMapping
    public ResponseEntity<ApiResponse<Page<NotificationResponse>>> getNotifications(
            @AuthenticationPrincipal UserPrincipal userDetails, Pageable pageable) {
        return ResponseEntity.ok(ApiResponse.success(notificationService.getNotifications(userDetails.getId(), pageable)));
    }

    // GET /api/v1/notifications/unread-count
    @GetMapping("/unread-count")
    public ResponseEntity<ApiResponse<Long>> getUnreadCount(@AuthenticationPrincipal UserPrincipal userDetails) {
        return ResponseEntity.ok(ApiResponse.success(notificationService.getUnreadCount(userDetails.getId())));
    }

    // PATCH /api/v1/notifications/{id}/read
    @PatchMapping("/{id}/read")
    public ResponseEntity<ApiResponse<Void>> markAsRead(@PathVariable UUID id) {
        notificationService.markAsRead(id);
        return ResponseEntity.ok(ApiResponse.success("Marked as read", null));
    }

    // PATCH /api/v1/notifications/read-all
    @PatchMapping("/read-all")
    public ResponseEntity<ApiResponse<Void>> markAllAsRead(@AuthenticationPrincipal UserPrincipal userDetails) {
        notificationService.markAllAsRead(userDetails.getId());
        return ResponseEntity.ok(ApiResponse.success("All marked as read", null));
    }
}
```

## 12.6. `NotificationScheduler` — nhắc ca làm

Chạy **mỗi phút** (`cron = "0 * * * * *"`), duyệt danh sách ca làm hôm nay và nhắc theo 2 mốc:

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class NotificationScheduler {

    // Chạy mỗi phút
    @Scheduled(cron = "0 * * * * *")
    public void sendWorkShiftReminders() {
        LocalDate today = LocalDate.now();
        LocalTime now = LocalTime.now();

        List<EmployeeWorkShift> shiftsToday = employeeWorkShiftRepository.findActiveByDate(today);

        for (EmployeeWorkShift ews : shiftsToday) {
            LocalTime startTime = ews.getWorkShift().getStartTime();
            if (startTime == null) continue;

            // 1. Nhắc TRƯỚC giờ làm 15 phút
            if (now.isAfter(startTime.minusMinutes(16)) && now.isBefore(startTime.minusMinutes(14))) {
                if (ews.getEmployee().getUser() != null) {
                    notificationService.sendNotification(
                        ews.getEmployee().getUser().getId(),
                        NotificationType.REMINDER,
                        "Upcoming Shift Reminder",
                        String.format("Your shift '%s' starts in 15 minutes.", ews.getWorkShift().getName()));
                }
            }

            // 2. Nhắc CHECK-IN (chưa chấm công sau 15 phút kể từ giờ bắt đầu)
            if (now.isAfter(startTime.plusMinutes(14)) && now.isBefore(startTime.plusMinutes(16))) {
                Optional<Attendance> attendance = attendanceRepository
                        .findActiveByEmployeeIdAndWorkDateAndWorkShift(
                            ews.getEmployee().getId(), today, ews.getWorkShift().getId());

                if (attendance.isEmpty() || attendance.get().getCheckInTime() == null) {
                    if (ews.getEmployee().getUser() != null) {
                        notificationService.sendNotification(
                            ews.getEmployee().getUser().getId(),
                            NotificationType.REMINDER,
                            "Check-in Reminder",
                            String.format("You haven't checked in for your shift '%s'. Please check in as soon as possible.", ews.getWorkShift().getName()));
                    }
                }
            }
        }
    }
}
```

Giải thích 2 cửa sổ thời gian (vì job chạy 1 lần/phút nên kiểm tra trong khoảng ±1 phút):
- `now` nằm trong `(startTime - 16 phút, startTime - 14 phút)` → **còn 15 phút nữa bắt đầu** → nhắc trước.
- `now` nằm trong `(startTime + 14 phút, startTime + 16 phút)` → **đã qua 15 phút** → kiểm tra đã check-in chưa; chưa thì nhắc.

## 12.7. Khi nào gọi `sendNotification` trong business

Ngoài scheduler, `NotificationService.sendNotification(...)` được gọi từ logic nghiệp vụ, VD:

- Duyệt/từ chối nghỉ phép → `LEAVE_APPROVED` / `LEAVE_REJECTED`.
- Payroll chốt lương → `SALARY`.

Quy tắc: **gọi sau khi transaction nghiệp vụ thành công** (tránh thông báo khi dữ liệu bị rollback). Nếu không chắc, có thể bọc bằng `@TransactionalEventListener` sau commit hoặc `@Async`.

## 12.8. Ghi chú & hạn chế cần biết

- **SSE dùng bộ nhớ** (`ConcurrentHashMap`) — chỉ hoạt động trong **một instance**. Nếu deploy nhiều instance phía sau load balancer, cần dùng Redis pub/sub hoặc WebSocket chuyển tải qua message broker.
- `SseEmitter` timeout 30 phút — client phải tự reconnect.
- Nếu user chưa đăng nhập (không có `UserPrincipal`) thì `/stream` trả lỗi — frontend cần đăng nhập trước.
- `NotificationScheduler` query `findActiveByDate` **mỗi phút** — khi dữ liệu lớn nên đánh index cho cột `date` (kèm `is_deleted = false`).

## 12.9. Checklist khi thêm thông báo mới

- [ ] Chọn `NotificationType` phù hợp (thêm enum nếu cần).
- [ ] Gọi `notificationService.sendNotification(userId, type, title, message)` sau khi nghiệp vụ thành công.
- [ ] Nếu là thông báo tự động theo thời gian → thêm `@Scheduled` method trong `NotificationScheduler`.
- [ ] Kiểm tra user có tồn tại (`getUser() != null`) trước khi gửi.
- [ ] Frontend đăng ký `/stream` bằng SSE và xử lý event `INIT` + `NOTIFICATION`.
