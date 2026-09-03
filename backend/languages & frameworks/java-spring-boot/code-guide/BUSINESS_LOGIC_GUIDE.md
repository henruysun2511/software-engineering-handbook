# Logic nghiệp vụ các module

Giải thích **quy trình nghiệp vụ** của từng module trong dự án (không lặp lại pattern viết code đã có ở các phần trước, mà tập trung vào **business rule**). Đọc phần này khi bạn cần sửa/xây dựng logic nghiệp vụ thực tế.

## Tổng quan luồng nghiệp vụ chính

```
Phân công ca (workshift)
   │  EmployeeWorkShiftService.assignWorkShift
   ▼
Chấm công (attendance)
   │  AttendanceService.checkIn / checkOut  (+ nhận diện khuôn mặt)
   ▼
Nghỉ phép (leave)
   │  LeaveRequestService.create → approve / reject / cancel
   ▼
Hợp đồng (contract)  ──►  nguồn baseSalary
   ▼
Bảng lương (payroll)
   │  PayrollService.create / autoGenerate → confirm
   ▼
Thông báo (notification)  ──►  SSE realtime cho user
```

## Employee — dữ liệu nhân sự gốc

Module trung tâm; các module khác **tham chiếu** tới `Employee`.

- **Tạo mới** (`createEmployee`): kiểm tra trùng `employee_code` và `email` trước khi lưu (ném `ValidationException.duplicateField`).
- `fullName` được **tự ghép** từ `lastName + " " + firstName`.
- Có thể liên kết với `User` (tài khoản đăng nhập) — nếu có `userId`, validate user tồn tại.
- Field nhạy cảm (`idCardNumber`, `bankAccountNumber`, `taxCode`...) được **mã hóa** khi lưu (xem phần 7).
- Mọi query đều là soft delete (`findActiveBy...`, lọc `isDeleted = false`).

## WorkShift — ca làm việc & phân công

### `WorkShift` (định nghĩa ca)
- Có `startTime`, `endTime`, `breakDuration` (giờ nghỉ giữa ca, mặc định 1.0).
- Ca có thể **qua đêm** (VD 22:00 → 06:00).

### `EmployeeWorkShift` (gán ca cho nhân viên theo ngày)
- `assignWorkShift`: kiểm tra **trùng lặp** — cùng employee + cùng ca + cùng ngày → báo lỗi (`existsActiveAssignment`).
- `getByUserIdAndDate` / `getAllByUserId`: tìm employee theo `userId` (dùng cho user đang đăng nhập tự xem ca của mình).
- Là **nguồn dữ liệu** cho: chấm công (tìm ca phù hợp) và scheduler nhắc ca (phần 12).

## Attendance — chấm công (check-in/check-out)

### Check-in (`checkIn`)
Thứ tự xử lý:
1. **Xác thực khuôn mặt**: gọi `FaceApiClient.verifyFace(photo, employeeId)` (Python API nội bộ) → không khớp thì từ chối.
2. **Tìm ca phù hợp** với thời gian hiện tại (`findMatchingShift`):
   - Tìm ca nằm trong cửa sổ `[start - 4h, end + 4h]` (buffer `CHECKIN_EARLY_BUFFER`/`CHECKOUT_LATE_BUFFER`).
   - Không khớp thì: nếu **chỉ có 1 ca** hôm nay → chấp nhận ca đó; nếu nhiều ca → chọn ca **gần nhất**.
   - Không có ca nào → lỗi "Nhân viên không có ca làm việc được phân công cho hôm nay".
3. **Chống check-in trùng**: nếu đã có record check-in cho ca đó hôm nay → báo lỗi.
4. **Tính phút đi muộn** (`lateMinutes`): `now > startTime` → `LATE`, ngược lại `PRESENT`.
5. Lưu record (nếu đã có bản ghi mở cho ca đó → **cập nhật** thay vì tạo mới).

### Check-out (`checkOut`)
1. **Xác thực khuôn mặt** tương tự.
2. Tìm các record **đang mở** (đã check-in, chưa check-out) trong ngày.
3. Chọn record có ca khớp cửa sổ thời gian hiện tại; nếu chỉ có 1 record mở → dùng nó; nhiều record không khớp → báo lỗi liệt kê ca đang chờ checkout.
4. **Tính toán**:
   - `workingHours = (checkOut - checkIn) - breakDuration` (làm tròn 2 số).
   - `earlyLeaveMinutes`: `now < endTime` → số phút về sớm.
   - **Cập nhật status**: đi muộn + về sớm → `HALF_DAY`; về sớm → `EARLY_LEAVE`; nếu không thì giữ nguyên.
5. Lưu và cập nhật `checkOutTime`, IP, tọa độ.

### Rules quan trọng
| Tình huống | Kết quả |
| ---------- | ------- |
| Check-in sau giờ bắt đầu ca | `LATE` + `lateMinutes` |
| Về sớm | `EARLY_LEAVE` + `earlyLeaveMinutes` |
| Đi muộn + về sớm | `HALF_DAY` |
| Đúng giờ | `PRESENT` |

- **Data nguồn cho payroll**: `countPresentByEmployeeIdAndWorkDateBetween`, `sumLateMinutes...`, `sumWorkingHours...`, `countLate...`.

## Leave — nghỉ phép

### Trạng thái (`LeaveRequestStatus`)
`PENDING` → `APPROVED` | `REJECTED` | `CANCELLED`

### Luồng nghiệp vụ
1. **Tạo đơn** (`createLeaveRequest`):
   - Validate: `endDate >= startDate`, `startDate` **không được ở quá khứ**.
   - `totalDays = (endDate - startDate) + 1` ngày.
   - Đơn tạo ra ở trạng thái `PENDING`.
2. **Duyệt/Từ chối** (`approveLeaveRequest`) — dành cho người quản lý:
   - **Chỉ đơn `PENDING`** mới được duyệt/từ chối.
   - Set `approver`, `approvedAt`, `approverComment`, `updatedBy`.
   - **Gửi thông báo realtime** cho nhân viên: `LEAVE_APPROVED` / `LEAVE_REJECTED`.
3. **Hủy** (`cancelLeaveRequest`):
   - **Chỉ chủ nhân viên** hủy được đơn của mình (`employeeId` phải khớp).
   - Chỉ đơn `PENDING` mới hủy được → `CANCELLED`.

### Liên quan payroll
- Đơn nghỉ phép **APPROVED** được tính như **ngày công có mặt** (`presentDays = attendedShifts + approvedLeavesDays`) trong quá trình tính lương (xem phần Payroll).

## Contract — hợp đồng (nguồn lương)

- **Tạo**: validate `endDate >= startDate`, upload file đính kèm qua `CloudinaryService.upload(file, "contracts")`.
- `ContractType`:
  - `MONTHLY` → lương theo tháng, prorate theo số ngày làm việc.
  - loại khác (VD theo ca) → tính theo `baseSalary * số ca`.
- `ContractStatus`: `ACTIVE`, `EXPIRED`, `TERMINATED`...
- **Vai trò**: `baseSalary`, `salaryCoefficient`, `allowance` từ hợp đồng `ACTIVE` là nguồn dữ liệu khi tạo/tự động sinh payroll.

## Payroll — bảng lương (quy trình tính toán)

### Trạng thái (`PayrollStatus`)
`DRAFT` → `CONFIRMED` (chỉ từ DRAFT mới confirm được).

### Công thức tính (`calculatePayroll`)
```
proratedSalary  = (baseSalary * salaryCoefficient / workingDays) * actualWorkingDays
totalIncome     = proratedSalary + overtimePay + allowance + bonus
totalDeductions = socialInsurance + healthInsurance + unemploymentInsurance + personalIncomeTax + latePenalty
netSalary       = totalIncome - totalDeductions
```

- `workingDays` nếu ≤ 0 thì coi là 1 (tránh chia 0).

### Tạo thủ công (`createPayroll`)
- **Chống trùng**: mỗi nhân viên mỗi tháng chỉ 1 payroll (`findActiveByEmployeeIdAndMonthAndYear`).
- `baseSalary` nếu không truyền → lấy từ hợp đồng `ACTIVE`; không có hợp đồng → lỗi yêu cầu cung cấp.

### Tự động sinh (`autoGeneratePayrolls`) — dành cho chạy cả tháng
Duyệt tất cả employee đang hoạt động, với mỗi người:
1. **Bỏ qua** nếu: đã có payroll không phải `DRAFT`, hoặc có nhưng không cho overwrite.
2. **Bỏ qua** nếu **không có hợp đồng `ACTIVE`** (đếm `skippedNoContract`).
3. **Bỏ qua** nếu **không có ca nào** được phân công và không có chấm công (đếm `skippedNoAttendance`).
4. **Tính toán** theo loại hợp đồng:
   - `MONTHLY`: `baseSalaryForMonth = contractBaseSalary * presentDays / scheduledShifts`, với
     `presentDays = số ca đã chấm công + số ngày nghỉ phép APPROVED`;
     phạt đi muộn = `(lateMinutes / 60) * latePenaltyPerHour`.
   - Loại theo ca: `baseSalaryForMonth = contractBaseSalary * scheduledShifts`;
     phạt đi muộn = `latePenaltyPerShift * lateCount`.
5. `allowance = request.allowance + contract.allowance`.
6. Gọi `calculatePayroll` → lưu → trả về danh sách kết quả.
7. Nếu **không tạo được payroll nào** → ném `ValidationException` kèm số người thiếu hợp đồng/không chấm công.

### Xác nhận (`confirmPayroll`)
- Chỉ payroll `DRAFT` → `CONFIRMED`. Sau khi confirm, `autoGenerate` sẽ **không** chạm vào payroll này.

### Thông báo
- Tạo/cập nhật payroll → gửi `NotificationType.SALARY` cho nhân viên có `user`.

## Notification — nhắc nhở & thông báo (chi tiết ở phần 12)

- Gọi từ: duyệt nghỉ phép (`LEAVE_APPROVED`/`LEAVE_REJECTED`), payroll (`SALARY`), scheduler (`REMINDER`).
- Điều kiện gửi: nhân viên phải có liên kết `user != null`.

## Những điều cần chú ý khi sửa logic nghiệp vụ

1. **Luôn giữ tính nhất quán trạng thái**: VD chỉ `PENDING` được duyệt, chỉ `DRAFT` được confirm — nếu thêm trạng thái mới phải cập nhật đủ các chỗ validate.
2. **Kiểm tra chống trùng** trước khi tạo: employee_code/email (employee), employee+shift+date (workshift), employee+month+year (payroll).
3. **Không tạo payroll trùng lặp** và không overwrite payroll đã confirm.
4. **Validate thời gian**: `endDate >= startDate`, startDate không ở quá khứ (leave), cửa sổ chấm công ±4h.
5. **Mọi thay đổi ghi dữ liệu** phải có `@Transactional`, `@CacheEvict`, `@AuditAction` + `AuditContext`.
6. **Thông báo gửi sau khi transaction thành công** — tránh thông báo khi dữ liệu bị rollback.
7. **Face API**: chấm công bắt buộc xác thực khuôn mặt — nếu đổi provider phải cập nhật `FaceApiClient`.
