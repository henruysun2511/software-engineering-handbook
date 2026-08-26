# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# CHƯƠNG 5: MANUAL TESTING THỰC CHIẾN

---

## 5.1 Tổng quan quy trình Manual Testing

Manual testing không phải là việc ngồi click ngẫu nhiên vào phần mềm. Đây là một quy trình có hệ thống, bắt đầu từ khi đọc yêu cầu và kết thúc khi sản phẩm được phát hành. Hiểu và thực hiện đúng quy trình là sự khác biệt giữa Tester nghiệp dư và Tester chuyên nghiệp.

**Toàn bộ quy trình:**

```
[1] Nhận Requirement
         ↓
[2] Phân tích & Đặt câu hỏi
         ↓
[3] Viết Test Scenario
         ↓
[4] Viết Test Case
         ↓
[5] Chuẩn bị Test Data & Môi trường
         ↓
[6] Thực thi kiểm thử (Test Execution)
         ↓
[7] Báo cáo Bug
         ↓
[8] Retest sau khi Developer fix
         ↓
[9] Regression Test
         ↓
[10] UAT (nếu có)
         ↓
[11] Sign-off & Release
```

---

## 5.2 Bước 1 — Nhận và Phân tích Requirement

### 5.2.1 Các dạng tài liệu yêu cầu

Trong thực tế dự án, yêu cầu có thể đến dưới nhiều dạng:

**User Story (Agile):**
```
As a [vai trò],
I want to [hành động],
So that [lợi ích/mục đích].

Acceptance Criteria:
- Given [ngữ cảnh], When [hành động], Then [kết quả]
```

**SRS — Software Requirements Specification (Waterfall):**
Tài liệu dài, mô tả chi tiết từng chức năng với Use Case, sequence diagram, business rules.

**Wireframe/Mockup:**
Thiết kế UI từ designer — Tester cần đọc và suy ra các luồng tương tác.

**Verbal/Informal:**
Trong các team nhỏ hoặc startup, đôi khi yêu cầu chỉ là cuộc trò chuyện qua Slack hoặc họp miệng. Tester cần tự ghi lại và xác nhận lại.

---

### 5.2.2 Phân tích Acceptance Criteria

Acceptance Criteria (AC) là "thước đo" xác định User Story hoàn thành. Phân tích AC tốt là kỹ năng quan trọng nhất của Tester.

**Ví dụ User Story và Acceptance Criteria thực tế:**

```
User Story: Tính năng Đăng ký tài khoản

AC-01: Người dùng có thể đăng ký bằng email và password
  - Email phải đúng định dạng (có @ và domain)
  - Password tối thiểu 8 ký tự, có ít nhất 1 chữ hoa, 1 số, 1 ký tự đặc biệt
  - Confirm Password phải khớp với Password

AC-02: Hệ thống kiểm tra email trùng lặp
  - Email đã tồn tại → hiển thị lỗi "Email đã được sử dụng"
  - Không phân biệt hoa thường cho email

AC-03: Sau đăng ký thành công
  - Gửi email xác nhận đến địa chỉ vừa đăng ký
  - Email chứa link xác nhận, hết hạn sau 24 giờ
  - Tài khoản trạng thái "pending" đến khi xác nhận

AC-04: Tài khoản chưa xác nhận email
  - Không thể đăng nhập
  - Hiển thị thông báo "Vui lòng xác nhận email trước khi đăng nhập"
  - Có tùy chọn "Gửi lại email xác nhận"
```

**Checklist khi đọc Acceptance Criteria:**

```
□ Mỗi AC có thể kiểm tra được không? (Testable)
□ AC có mơ hồ không? ("nhanh", "đẹp", "hợp lý" là các từ nguy hiểm)
□ Có điều kiện biên nào không được đề cập?
□ Điều gì xảy ra khi có lỗi/ngoại lệ?
□ AC có mâu thuẫn với AC khác không?
□ Yêu cầu phi chức năng (performance, security) có được đề cập không?
```

---

### 5.2.3 Đặt câu hỏi hiệu quả với BA/PM

**Kỹ năng đặt câu hỏi** là một trong những kỹ năng quan trọng nhất của Tester, giúp phát hiện vấn đề ngay từ giai đoạn yêu cầu.

**Framework đặt câu hỏi — WHAT/WHEN/WHO/WHERE/HOW/WHAT IF:**

```
WHAT (Cái gì):
- "Cụ thể hành vi mong đợi là gì trong trường hợp này?"
- "Định dạng dữ liệu đầu ra là gì?"

WHEN (Khi nào):
- "Quy tắc này áp dụng trong trường hợp nào?"
- "Khi nào thì hiển thị thông báo này?"

WHO (Ai):
- "Tính năng này dành cho role nào?"
- "Admin có quyền khác user thường không?"

HOW (Như thế nào):
- "Hệ thống tính toán giá trị này như thế nào?"
- "Email xác nhận được gửi bằng cơ chế nào?"

WHAT IF (Điều gì nếu):
← ĐÂY LÀ NHÓM QUAN TRỌNG NHẤT
- "Điều gì xảy ra nếu người dùng nhập email với chữ hoa?"
- "Điều gì xảy ra nếu link xác nhận bị click hai lần?"
- "Điều gì xảy ra nếu mất kết nối khi đang thanh toán?"
- "Điều gì xảy ra nếu người dùng đang xem trang thì session hết hạn?"
```

**Ví dụ email đặt câu hỏi chuyên nghiệp:**

```
Subject: [Câu hỏi] Story #123 - Tính năng Đăng ký tài khoản

Hi [BA Name],

Sau khi đọc AC của Story #123, tôi có một số câu hỏi cần làm rõ 
trước khi viết test case:

1. [AC-01] Password chứa khoảng trắng có được chấp nhận không?
   Ví dụ: "My Pass@1" (có space giữa)

2. [AC-02] Email so sánh không phân biệt hoa thường — điều này 
   có áp dụng cho local part không?
   Ví dụ: "User@example.com" có trùng với "user@example.com" không?

3. [AC-03] Link xác nhận hết hạn sau 24h — sau khi hết hạn, 
   người dùng cần làm gì? Họ có thể yêu cầu gửi lại không?
   Và link cũ có bị vô hiệu hóa ngay không?

4. [AC-04] Nếu người dùng đăng ký lại bằng email chưa xác nhận, 
   hệ thống xử lý thế nào? Cập nhật lại hay báo lỗi "email đã tồn tại"?

Xin cảm ơn!
[Tên Tester]
```

---

## 5.3 Bước 2 — Viết Test Scenario

Test Scenario được rút ra từ yêu cầu. Mỗi AC thường sinh ra ít nhất 2-3 scenario: happy path, unhappy path, và edge case.

**Cách rút ra Test Scenario từ AC:**

Lấy **AC-01** ở trên làm ví dụ:

```
AC-01: Người dùng có thể đăng ký bằng email và password

Happy Path:
TS-01: Đăng ký thành công với đầy đủ thông tin hợp lệ

Unhappy Path — Email:
TS-02: Đăng ký thất bại khi email sai định dạng (nhiều trường hợp con)
TS-03: Đăng ký thất bại khi email trống

Unhappy Path — Password:
TS-04: Đăng ký thất bại khi password không đủ 8 ký tự
TS-05: Đăng ký thất bại khi password không có chữ hoa
TS-06: Đăng ký thất bại khi password không có số
TS-07: Đăng ký thất bại khi password không có ký tự đặc biệt
TS-08: Đăng ký thất bại khi Confirm Password không khớp
TS-09: Đăng ký thất bại khi Password trống

Edge Cases:
TS-10: Đăng ký với password đúng đủ 8 ký tự (ranh giới dưới)
TS-11: Đăng ký với password đúng 20 ký tự (ranh giới trên nếu có)
TS-12: Đăng ký với email chứa ký tự đặc biệt hợp lệ (+, ., -)
TS-13: Đăng ký với email có ký tự hoa và thường lẫn lộn
```

---

## 5.4 Bước 3 — Viết Test Case

Từ mỗi Test Scenario, viết Test Case chi tiết. Phần này đã được hướng dẫn trong Chương 2. Dưới đây là ví dụ áp dụng vào feature thực tế.

### 5.4.1 Thực hành: Bộ Test Case hoàn chỉnh cho tính năng Đặt hàng (Checkout)

**Yêu cầu nghiệp vụ:**
- Người dùng đã đăng nhập mới có thể checkout
- Giỏ hàng phải có ít nhất 1 sản phẩm
- Bắt buộc nhập địa chỉ giao hàng
- Chọn phương thức thanh toán: COD hoặc Thẻ ngân hàng
- Sau khi đặt hàng thành công: gửi email xác nhận, trừ tồn kho

---

**TC_CHK_001 — Đặt hàng thành công với COD**

| Trường | Nội dung |
|---|---|
| **ID** | TC_CHK_001 |
| **Title** | Đặt hàng thành công với phương thức COD |
| **Priority** | Critical |
| **Preconditions** | Đăng nhập `buyer@test.com`. Giỏ hàng có 1 sản phẩm "Áo thun Basic" (200,000đ, còn 10 sản phẩm tồn kho). |
| **Step 1** | Vào trang Giỏ hàng (`/cart`) |
| **Step 2** | Click "Tiến hành thanh toán" |
| **Step 3** | Điền địa chỉ: "123 Nguyễn Huệ, Q1, TP.HCM" |
| **Step 4** | Điền SĐT: "0901234567" |
| **Step 5** | Chọn phương thức "COD (Thanh toán khi nhận hàng)" |
| **Step 6** | Click "Đặt hàng" |
| **Expected** | ① Chuyển đến trang xác nhận đơn hàng với Order ID. ② Email xác nhận gửi đến `buyer@test.com`. ③ DB: Đơn hàng được tạo, trạng thái `pending`. ④ Tồn kho "Áo thun Basic" giảm từ 10 xuống 9. ⑤ Giỏ hàng trống sau khi đặt. |

---

**TC_CHK_002 — Không thể checkout khi giỏ hàng trống**

| Trường | Nội dung |
|---|---|
| **ID** | TC_CHK_002 |
| **Title** | Chặn checkout khi giỏ hàng không có sản phẩm |
| **Priority** | High |
| **Preconditions** | Đăng nhập `buyer@test.com`. Giỏ hàng đang trống. |
| **Step 1** | Truy cập trực tiếp URL `/checkout` |
| **Expected** | Hệ thống chuyển hướng về trang giỏ hàng `/cart`. Hiển thị thông báo "Giỏ hàng trống, vui lòng thêm sản phẩm". Không hiển thị form checkout. |

---

**TC_CHK_003 — Không thể checkout khi chưa đăng nhập**

| Trường | Nội dung |
|---|---|
| **ID** | TC_CHK_003 |
| **Title** | Chuyển hướng đến trang đăng nhập khi checkout mà chưa login |
| **Priority** | High |
| **Preconditions** | Chưa đăng nhập. Trỏ đến `/checkout`. |
| **Step 1** | Truy cập URL `/checkout` |
| **Expected** | Chuyển hướng đến `/login?redirect=/checkout`. Sau khi đăng nhập, tự động quay lại trang checkout. |

---

**TC_CHK_004 — Validation địa chỉ giao hàng bắt buộc**

| Trường | Nội dung |
|---|---|
| **ID** | TC_CHK_004 |
| **Title** | Hiển thị lỗi khi bỏ trống địa chỉ giao hàng |
| **Priority** | High |
| **Preconditions** | Đăng nhập, giỏ hàng có sản phẩm, đang ở trang checkout. |
| **Step 1** | Bỏ trống trường địa chỉ |
| **Step 2** | Điền SĐT "0901234567" |
| **Step 3** | Chọn COD |
| **Step 4** | Click "Đặt hàng" |
| **Expected** | Không gửi request. Hiển thị lỗi đỏ bên dưới trường địa chỉ: "Vui lòng nhập địa chỉ giao hàng". Focus tự động vào trường địa chỉ. |

---

**TC_CHK_005 — Tồn kho cập nhật đúng khi đặt nhiều sản phẩm**

| Trường | Nội dung |
|---|---|
| **ID** | TC_CHK_005 |
| **Title** | Tồn kho giảm đúng số lượng sau khi đặt hàng thành công |
| **Priority** | High |
| **Preconditions** | Sản phẩm A tồn kho: 5, Sản phẩm B tồn kho: 3. Giỏ hàng: SP-A × 2, SP-B × 1. |
| **Steps** | Thực hiện checkout và đặt hàng thành công |
| **Expected** | DB: Tồn kho SP-A = 3 (5-2). DB: Tồn kho SP-B = 2 (3-1). |

---

**TC_CHK_006 — Race condition: Sản phẩm hết hàng khi đang checkout**

| Trường | Nội dung |
|---|---|
| **ID** | TC_CHK_006 |
| **Title** | Xử lý đúng khi sản phẩm hết hàng trong lúc checkout |
| **Priority** | High |
| **Preconditions** | Sản phẩm "Giày limited" tồn kho: 1. Người dùng A và Người dùng B cùng thêm sản phẩm vào giỏ. |
| **Steps** | 1. Người dùng A và B cùng click "Đặt hàng" gần như đồng thời. |
| **Expected** | Người dùng A (đặt trước 1 giây): Đặt hàng thành công. Người dùng B: Nhận thông báo "Sản phẩm vừa hết hàng". Tồn kho không âm. |

---

## 5.5 Bước 4 — Chuẩn bị Test Data

**Test Data** là dữ liệu cụ thể dùng để thực thi test case. Test data tốt giúp kiểm thử toàn diện và tái hiện được.

### 5.5.1 Các loại Test Data

**Static Test Data:**
Dữ liệu cố định, tạo một lần và dùng lâu dài. Ví dụ: tài khoản test cố định, danh sách sản phẩm mẫu.

**Dynamic Test Data:**
Dữ liệu tạo mới cho mỗi lần test. Ví dụ: email đăng ký mới, đơn hàng mới.

**Production Data (Anonymized):**
Sao chép và ẩn danh hóa dữ liệu thực từ production để phản ánh đúng thực tế. Không bao giờ dùng dữ liệu production thật để kiểm thử.

### 5.5.2 Chiến lược quản lý Test Data

**Vấn đề phổ biến:**
- Tester A dùng tài khoản test, Tester B dùng chung và thay đổi dữ liệu → kết quả kiểm thử không tin cậy
- Dữ liệu bị "bẩn" sau nhiều lần chạy → test case fail vô lý

**Giải pháp:**

```
1. Tạo bộ test data riêng cho từng Tester
   - buyer_a@test.com, buyer_b@test.com
   - admin_test1@test.com, admin_test2@test.com

2. Dùng naming convention rõ ràng
   - Tên sản phẩm test: [TEST] Áo thun Basic
   - Email test: qa.[tên].[mục đích]@test.com
   - Ví dụ: qa.alice.checkout@test.com

3. Reset data trước khi bắt đầu test run
   - Script SQL reset về trạng thái baseline
   - Hoặc dùng API endpoint /admin/reset-test-data

4. Tạo data thông qua API thay vì UI
   - Nhanh hơn, đáng tin cậy hơn
   - Dễ tự động hóa

5. Document test data
   - Ghi lại tài khoản test, mật khẩu, điều kiện
   - Chia sẻ với toàn team
```

### 5.5.3 Ví dụ Test Data Document

```
=== TEST DATA — Project E-commerce ===

[ACCOUNTS]
Regular User:     buyer@test.com / Test@123
VIP User:         vip@test.com / Test@123  (VIP member, tier: Gold)
New User:         (tạo mới mỗi lần test)
Locked User:      locked@test.com / Test@123 (đã bị khóa)
Unverified User:  unverified@test.com / Test@123 (chưa xác nhận email)
Admin:            admin@test.com / Admin@123

[PRODUCTS]
In Stock:         [TEST] Áo thun Basic - ID: P001 - Price: 200,000 - Stock: 50
Out of Stock:     [TEST] Giày limited - ID: P002 - Price: 500,000 - Stock: 0
Sale Product:     [TEST] Váy hè - ID: P003 - Price: 300,000 - Sale: 50% - Stock: 10
Max Price:        [TEST] Túi xách cao cấp - ID: P004 - Price: 9,999,000 - Stock: 5

[COUPON CODES]
Valid 20% off:    SUMMER20 (hết hạn: 31/12/2025, dùng tối đa 100 lần)
Expired:          WINTER10 (đã hết hạn: 01/01/2025)
Used up:          FLASH50 (đã hết số lần dùng)
VIP only:         VIP30 (chỉ dành cho VIP members)

[PAYMENT]
Valid card:       4111 1111 1111 1111 / 12/26 / 123
Declined card:   4000 0000 0000 0002 (luôn bị từ chối)
Insufficient:    4000 0000 0000 9995 (luôn báo không đủ tiền)
```

---

## 5.6 Bước 5 — Thực Thi Kiểm Thử (Test Execution)

### 5.6.1 Quy trình thực thi chuyên nghiệp

**Trước khi bắt đầu:**
```
□ Entry Criteria đã được thỏa mãn
□ Build đã được deploy đúng version
□ Test data đã được chuẩn bị
□ Môi trường kiểm thử sẵn sàng
□ TestRail/Jira đã được cấu hình Test Run
□ Mở DevTools → Network tab (preserve log)
□ Sẵn sàng công cụ chụp màn hình/quay video
```

**Trong khi thực thi:**

1. **Đọc kỹ precondition** trước mỗi test case — không assume trạng thái từ test case trước
2. **Thực hiện đúng từng bước** — không bỏ qua, không làm tắt
3. **Quan sát cả những thứ không được đề cập** trong test case — behavior bất thường đáng ngờ
4. **Ghi Actual Result ngay** — không nhớ sau
5. **Khi phát hiện bug ngoài test case đang chạy** → note lại, tạo bug sau khi hoàn thành test case hiện tại
6. **Chụp ảnh/quay video** ngay khi thấy lỗi — trước khi làm thêm bất kỳ thao tác nào

**Kỹ năng quan sát:**
Tester giỏi không chỉ kiểm tra Expected Result — họ quan sát cả:
- URL có đúng không sau khi navigate?
- Console log có error không?
- Network request có gửi đúng không? Response code là gì?
- Dữ liệu trong DB có đúng không?
- Email có được gửi không? Nội dung có đúng không?
- Tác động lên các phần khác của hệ thống?

---

### 5.6.2 Kiểm thử tính năng Leave Request (Nghỉ phép) — Ví dụ đầy đủ

**Nghiệp vụ:**
- Nhân viên tạo đơn xin nghỉ phép, chọn loại nghỉ và ngày
- Manager nhận thông báo qua email, phê duyệt hoặc từ chối
- Số ngày phép năm tự động trừ khi đơn được phê duyệt
- Nhân viên nhận thông báo kết quả

**Thực thi test case TC_LEAVE_001:**

```
Test Case: Tạo đơn nghỉ phép thành công

Precondition: 
  - Đăng nhập: employee@test.com (có 12 ngày phép còn lại)
  - Manager của employee này: manager@test.com

Step 1: Click menu "Nghỉ phép" → "Tạo đơn mới"
→ Quan sát: Form hiện ra với các trường
→ Network: GET /api/leave/balance → trả về {annual: 12, sick: 5}
→ ✓ Số ngày phép hiển thị đúng: 12 ngày

Step 2: Chọn "Loại nghỉ: Nghỉ phép năm"
→ Quan sát: Dropdown hiển thị đúng các loại

Step 3: Chọn ngày bắt đầu: 20/01/2025, ngày kết thúc: 22/01/2025
→ Quan sát: Tự động tính 3 ngày làm việc (20, 21, 22 đều là ngày thường)
→ ✓ Hiển thị "Tổng: 3 ngày"

Step 4: Nhập lý do: "Nghỉ dưỡng sức"

Step 5: Click "Gửi đơn"
→ Network: POST /api/leave/requests
   Request body: {type: "annual", start: "2025-01-20", end: "2025-01-22", reason: "Nghỉ dưỡng sức"}
   Response: 201 Created {id: "LR-2025-001", status: "pending"}

→ Actual Result:
   ① Hiển thị thông báo "Đơn nghỉ phép đã được gửi thành công"
   ② Đơn xuất hiện trong danh sách với trạng thái "Chờ duyệt"
   
→ Verify email (check hộp thư manager@test.com):
   ③ Manager nhận email thông báo "Nhân viên [Tên] xin nghỉ phép 3 ngày (20-22/01)"

→ Verify DB:
   ④ Bảng leave_requests: record mới status='pending', days=3
   ⑤ Bảng employee_leave_balance: CHƯA trừ (chỉ trừ khi phê duyệt)

→ STATUS: PASS ✅
```

---

## 5.7 Bước 6 — Báo cáo và theo dõi Bug

Sau khi phát hiện lỗi trong test execution, quy trình báo cáo chuẩn như đã học trong Chương 3. Phần này tập trung vào **giao tiếp hiệu quả** sau khi tạo bug.

### 5.7.1 Giao tiếp với Developer về Bug

**Nguyên tắc giao tiếp:**
- Mô tả bug, không phán xét người viết code: ❌ "Code sai rồi" → ✅ "Hành vi hiện tại là X, yêu cầu là Y"
- Cung cấp đủ thông tin ngay từ đầu để giảm qua lại
- Sẵn sàng hỗ trợ Developer tái hiện nếu họ gặp khó khăn
- Không để bug "trôi" — follow up nếu không thấy tiến độ

**Khi Developer nói "Không phải bug":**
- Lắng nghe giải thích của họ — có thể bạn hiểu sai yêu cầu
- Nếu bạn tin đây là bug, trích dẫn AC hoặc tài liệu yêu cầu cụ thể
- Nếu vẫn không đồng ý, escalate với BA/PM để xác nhận yêu cầu
- Tránh "tranh luận" cảm xúc — luôn dùng bằng chứng từ tài liệu

---

## 5.8 Bước 7 — Retest

Retest là thực thi lại **đúng test case đã fail** sau khi Developer báo đã fix.

### 5.8.1 Quy trình Retest chuẩn

```
1. Developer cập nhật trạng thái bug → "Resolved/Fixed"
   và ghi thêm: commit hash, branch, nguyên nhân gốc rễ (root cause)

2. Tester verify build chứa fix đã được deploy
   (không retest trên build cũ)

3. Tester đọc lại bug report gốc — nhớ lại chính xác
   những gì đã xảy ra

4. Thực thi lại đúng các Steps to Reproduce

5. Kiểm tra:
   ✓ Expected Result có đạt không?
   ✓ Fix có gây ra lỗi mới (side effect) không?
   ✓ Edge case liên quan có hoạt động không?

6. Kết quả:
   → PASS: Cập nhật status "Verified", close bug
   → FAIL: Cập nhật Actual Result mới, Reopen bug,
            ghi rõ "Vẫn còn lỗi sau khi fix"
```

### 5.8.2 Ví dụ Retest thực tế

```
Bug #BUG-234: Mã giảm giá không áp dụng với sản phẩm sale

Developer fix: 
  - Root cause: Logic kiểm tra loại giảm giá bị sai điều kiện
  - Fix: Thay đổi hàm calculateDiscount() trong cart.service.ts
  - Commit: abc123def
  - Deploy: Staging v2.3.1

RETEST:

Bước 1: Verify build
  → Staging header: X-App-Version: 2.3.1 ✓

Bước 2: Thực thi lại steps gốc
  → Thêm "Áo thun Basic" (sale 50%) vào giỏ → 100,000đ
  → Nhập SUMMER20 → Click Áp dụng
  → Actual: Hiển thị "Giảm SUMMER20: -20,000đ", Tổng: 80,000đ ✓
  → Expected: Giảm 20% trên 100,000đ = 80,000đ ✓

Bước 3: Kiểm tra edge case liên quan
  → Test với 2 sản phẩm (1 sale, 1 không sale)
    SP1: 100,000đ (sau sale 50%) + SP2: 200,000đ = 300,000đ
    SUMMER20 giảm 20%: -60,000đ → Tổng: 240,000đ ✓
  → Test với sản phẩm không sale (vẫn phải hoạt động)
    SP: 200,000đ → SUMMER20 giảm 20% → 160,000đ ✓

Kết quả: PASS ✅
Action: Cập nhật Jira → Verified → Closed
```

---

## 5.9 Bước 8 — Regression Testing

### 5.9.1 Xác định phạm vi Regression

Không phải toàn bộ test suite cần chạy sau mỗi thay đổi. Phân tích impact để chọn đúng phạm vi:

```
Thay đổi: Fix bug trong hàm calculateDiscount() của CartService

Impact Analysis:
  Trực tiếp:
  → CartService → Tất cả test liên quan đến giỏ hàng và giảm giá

  Gián tiếp:
  → CheckoutService (dùng CartService) → Test checkout
  → OrderService (tạo order từ cart) → Test tạo đơn hàng
  → Invoice (dùng order total) → Test tạo hóa đơn

  Không liên quan:
  → AuthService → Không cần chạy
  → UserProfileService → Không cần chạy
  → ProductService → Không cần chạy (nếu không thay đổi)
```

### 5.9.2 Xây dựng Regression Test Suite

**Regression Test Suite** là tập test case được duy trì và chạy định kỳ. Tiêu chí chọn test case vào Regression Suite:

```
Luôn có trong Regression Suite:
□ Test case cho critical path (luồng quan trọng nhất)
□ Test case cho chức năng cốt lõi của business
□ Test case cho các bug đã từng xảy ra (để đảm bảo không tái phát)
□ Test case cho tích hợp giữa các module chính
□ Smoke test (tập con nhỏ nhất, luôn chạy đầu tiên)

Thêm vào Regression Suite khi:
□ Bug mới được fixed (thêm test case verify fix đó)
□ Feature mới được release (thêm happy path)
□ Chức năng được xác định là high-risk

Loại ra khỏi Regression Suite khi:
□ Feature bị deprecated
□ Test case trùng lặp với test case khác
□ Test case quá chậm mà coverage thấp
```

---

## 5.10 Bước 9 — UAT (User Acceptance Testing)

### 5.10.1 UAT là gì và ai thực hiện?

**UAT (User Acceptance Testing)** là giai đoạn kiểm thử cuối cùng, được thực hiện bởi **khách hàng hoặc người dùng cuối** (không phải Tester nội bộ) để xác nhận phần mềm đáp ứng nhu cầu thực tế của họ.

**Vai trò của Tester trong UAT:**
- Tester KHÔNG thực hiện UAT — người dùng thực tế mới là người test
- Tester chuẩn bị môi trường, test data, và tài liệu hướng dẫn cho người dùng
- Tester hỗ trợ kỹ thuật khi người dùng gặp vấn đề
- Tester ghi nhận bug phát hiện trong UAT và triage
- Tester báo cáo tiến độ UAT cho stakeholders

### 5.10.2 Chuẩn bị cho UAT

**UAT Test Plan mẫu:**

```
=== UAT PLAN — Hệ thống Quản lý Nhân sự v2.0 ===

Mục tiêu: Xác nhận 5 tính năng mới đáp ứng nghiệp vụ HR

Người tham gia:
  - Business User: Chị Lan (HR Manager), Anh Minh (HR Staff)
  - Technical Support: Nguyễn Văn A (QA), Trần Thị B (BA)

Thời gian: 15/01 - 19/01/2025 (5 ngày làm việc)

Môi trường: UAT Server (uat.hrm.company.com)
  - Account mẫu đã được tạo sẵn
  - Dữ liệu test đã được import (dữ liệu giả theo cấu trúc thật)

Phạm vi test:
  1. Quản lý nghỉ phép (Leave Management)
  2. Đánh giá nhân viên (Performance Review)
  3. Bảng lương tháng (Payroll)
  4. Báo cáo HR Dashboard
  5. Quản lý hợp đồng

Quy trình báo cáo lỗi:
  - Người dùng gửi vấn đề qua Google Form
  - QA triage và tạo bug trong Jira trong 24h
  - Fix Critical/High: trong 2 ngày
  - Fix Medium/Low: trong sprint tiếp theo

Exit Criteria cho UAT:
  - 90% test scenario được thực hiện
  - Không còn bug Critical hoặc High
  - Business sign-off từ HR Manager
```

---

## 5.11 Bước 10 — Sign-off và Release

### 5.11.1 Go/No-Go Decision

Trước khi release, team họp **Go/No-Go meeting** để quyết định có phát hành hay không. Tester trình bày Test Summary Report (đã học ở Chương 3) để cung cấp thông tin.

**Các yếu tố xem xét:**
```
✓ Pass rate có đạt ngưỡng không? (thường ≥ 95%)
✓ Không còn bug Critical/High nào Open?
✓ Các bug chấp nhận release đã được document?
✓ Regression test pass?
✓ UAT được sign-off?
✓ Performance test đạt SLA?
✓ Security scan không phát hiện lỗ hổng nghiêm trọng?
✓ Documentation đã cập nhật?
✓ Rollback plan đã chuẩn bị?
```

### 5.11.2 Smoke Test sau Deployment

Sau khi release lên production, Tester thực hiện **Production Smoke Test** để xác nhận hệ thống hoạt động cơ bản:

```
Production Smoke Test — E-commerce v2.3.0
Thực hiện: 15 phút sau deployment

□ Trang chủ load thành công (< 3s)
□ Tìm kiếm sản phẩm hoạt động
□ Trang sản phẩm hiển thị đúng (ảnh, giá, tồn kho)
□ Thêm vào giỏ hàng thành công
□ Đăng nhập thành công
□ Thanh toán test (dùng thẻ sandbox) thành công
□ Email xác nhận được gửi
□ Trang admin load thành công
□ API health check: /api/health → 200 OK

Kết quả: Pass/Fail
Người thực hiện: [Tên Tester]
Thời gian: [Timestamp]
```

Nếu Smoke Test fail → Notify ngay cho team, cân nhắc rollback.

---

## 5.12 Thực hành tổng hợp — Kiểm thử tính năng Payroll

**Mô tả nghiệp vụ:**
Hệ thống tính lương tháng tự động dựa trên:
- Lương cơ bản theo hợp đồng
- Số ngày công thực tế (tổng ngày làm - ngày nghỉ phép)
- Thưởng KPI (0%, 5%, 10%, 15% theo mức đạt)
- Các khoản khấu trừ: BHXH (8%), BHYT (1.5%), BHTN (1%), Thuế TNCN
- Phụ cấp: ăn trưa (30,000đ/ngày), xăng xe (theo chức vụ)

**Bộ Test Scenario:**
```
Happy Path:
S01: Tính lương đúng cho nhân viên đi làm đầy đủ (23 ngày công), không nghỉ
S02: Tính lương đúng khi có nghỉ phép được duyệt (3 ngày phép năm)
S03: Tính lương đúng với thưởng KPI 10%

Negative:
S04: Nhân viên nghỉ không phép (không lương những ngày đó)
S05: Nhân viên chưa có hợp đồng → không tính lương
S06: Tháng tính lương chưa đến → không cho phép chạy

Edge Cases:
S07: Nhân viên mới vào giữa tháng (tính pro-rata)
S08: Nhân viên nghỉ việc giữa tháng
S09: Tháng có 28 ngày (tháng 2) vs 31 ngày
S10: Ngưỡng thuế TNCN (thu nhập vừa vượt mức chịu thuế)
```

**Ví dụ test case tính lương:**

**TC_PAY_001 — Tính lương cơ bản đầy đủ ngày công**

| Trường | Nội dung |
|---|---|
| **Preconditions** | Nhân viên: Nguyễn Văn A. Lương cơ bản: 15,000,000đ/tháng. Tháng 1/2025 có 23 ngày làm việc. Đi làm đủ 23 ngày. KPI: 0% (không thưởng). Chức vụ: Nhân viên (phụ cấp xăng 500,000đ/tháng). |
| **Expected** | Lương gross = 15,000,000đ + 500,000đ (xăng) + 23×30,000đ (ăn trưa) = 16,190,000đ |
| | BHXH = 15,000,000 × 8% = 1,200,000đ |
| | BHYT = 15,000,000 × 1.5% = 225,000đ |
| | BHTN = 15,000,000 × 1% = 150,000đ |
| | Thu nhập tính thuế = 16,190,000 - 1,575,000 - 11,000,000 (giảm trừ bản thân) = 3,615,000đ |
| | Thuế TNCN = 3,615,000 × 5% = 180,750đ |
| | Lương net = 16,190,000 - 1,575,000 - 180,750 = **14,434,250đ** |
| **Verify** | ① UI hiển thị đúng các dòng tính toán. ② DB: bảng payroll_records có record đúng. ③ Export PDF/Excel có số liệu khớp. |

> **Lưu ý thực tế:** Với test case phức tạp như tính lương, Tester nên chuẩn bị bảng tính Excel song song để verify kết quả hệ thống. Nếu số liệu không khớp nhưng chênh lệch nhỏ, cần xác nhận với kế toán về quy tắc làm tròn.
