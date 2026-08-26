# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# CHƯƠNG 2: TEST CASE & KỸ THUẬT THIẾT KẾ

---

## 2.1 Test Scenario và Test Case

### 2.1.1 Test Scenario là gì?

**Định nghĩa:** Test Scenario (kịch bản kiểm thử) là một câu mô tả cấp cao về một tình huống cần kiểm thử, phản ánh một hành vi hoặc chức năng của hệ thống từ góc nhìn người dùng.

**Đặc điểm:**
- Mô tả *cái gì* cần kiểm thử, không phải *như thế nào*
- Ngắn gọn, thường một câu
- Được dẫn xuất từ User Story hoặc tài liệu yêu cầu
- Một scenario có thể dẫn đến nhiều test case

**Ví dụ Test Scenario cho tính năng Đăng nhập:**
1. Đăng nhập thành công với thông tin hợp lệ
2. Đăng nhập thất bại với password sai
3. Đăng nhập thất bại với email không tồn tại
4. Đăng nhập sau khi tài khoản bị khóa
5. Đăng nhập khi đã đăng nhập trên thiết bị khác
6. Đăng nhập với các loại ký tự đặc biệt trong email

---

### 2.1.2 Test Case là gì?

**Định nghĩa:** Test Case (ca kiểm thử) là một tập hợp các điều kiện và bước thực hiện được đặc tả chi tiết, kèm theo dữ liệu đầu vào cụ thể và kết quả mong đợi tương ứng, dùng để xác minh một chức năng hoặc điều kiện cụ thể của phần mềm.

**Bản chất:** Test case là "bản hướng dẫn thực hiện" — bất kỳ ai đọc vào đều phải hiểu chính xác cần làm gì và phần mềm phải phản hồi thế nào. Kết quả kiểm thử phải **có thể tái tạo** và **không phụ thuộc vào người thực hiện**.

---

### 2.1.3 Test Case vs Test Scenario

| Tiêu chí | Test Scenario | Test Case |
|---|---|---|
| Mức độ chi tiết | Cấp cao (high-level) | Chi tiết (low-level) |
| Mô tả | Cái gì cần kiểm thử | Như thế nào để kiểm thử |
| Dữ liệu | Không có | Cụ thể |
| Bước thực hiện | Không có | Chi tiết từng bước |
| Kết quả mong đợi | Không có | Rõ ràng, cụ thể |
| Số lượng | Ít hơn | Nhiều hơn (1 scenario → nhiều test case) |
| Tác giả | BA, Tester | Tester |

---

## 2.2 Cấu Trúc Một Test Case Chuẩn

Một test case đầy đủ gồm các trường sau:

### Các trường bắt buộc và ý nghĩa

**Test Case ID:** Định danh duy nhất, theo quy ước của dự án.
- Ví dụ: `TC_LOGIN_001`, `TC_REG_002`

**Test Case Title:** Mô tả ngắn gọn, rõ ràng nội dung test case.
- Tốt: *"Đăng nhập thành công với email và password hợp lệ"*
- Không tốt: *"Test đăng nhập"*

**Module / Feature:** Chức năng thuộc về phần nào của ứng dụng.

**Priority:** Mức độ ưu tiên — Critical / High / Medium / Low.

**Preconditions (Điều kiện tiên quyết):** Trạng thái hệ thống phải tồn tại *trước khi* thực hiện test case. Nếu precondition không được thỏa mãn, test case không thể thực hiện được và không nên coi là fail.
- Ví dụ: *"Tài khoản test@example.com đã tồn tại và chưa bị khóa"*

**Test Steps (Bước thực hiện):** Các bước thực hiện tuần tự, mỗi bước đủ cụ thể để người khác thực hiện chính xác.

**Test Data (Dữ liệu kiểm thử):** Dữ liệu đầu vào cụ thể dùng trong test case.
- Ví dụ: `Email: test@example.com`, `Password: Test@123`

**Expected Result (Kết quả mong đợi):** Mô tả chính xác những gì phải xảy ra sau khi thực hiện các bước. Phải cụ thể, có thể đo lường được.
- Tốt: *"Hệ thống chuyển hướng đến trang Dashboard, hiển thị tên người dùng 'Nguyễn Văn A' ở góc trên phải"*
- Không tốt: *"Đăng nhập thành công"*

**Actual Result (Kết quả thực tế):** Điền vào sau khi thực thi — những gì thực sự xảy ra.

**Status:** Pass / Fail / Blocked / Skipped
- **Pass:** Actual result = Expected result
- **Fail:** Actual result ≠ Expected result
- **Blocked:** Không thể thực thi do phụ thuộc vào bug khác hoặc thiếu điều kiện
- **Skipped:** Bỏ qua có lý do (thường do thiếu thời gian, chức năng chưa sẵn sàng)

---

### 2.2.1 Thực hành: Test Case cho tính năng Login

**User Story:** *"Là người dùng đã đăng ký, tôi muốn đăng nhập bằng email và password để truy cập tài khoản của mình."*

**Acceptance Criteria:**
- Email và password đúng → đăng nhập thành công, chuyển về Dashboard
- Email không tồn tại → hiển thị thông báo lỗi
- Password sai → hiển thị thông báo lỗi, không tiết lộ email có tồn tại không
- Sai password 5 lần liên tiếp → khóa tài khoản 30 phút

---

**TC_LOGIN_001 — Đăng nhập thành công**

| Trường | Nội dung |
|---|---|
| **ID** | TC_LOGIN_001 |
| **Title** | Đăng nhập thành công với email và password hợp lệ |
| **Priority** | Critical |
| **Preconditions** | Tài khoản `test@example.com` tồn tại, chưa bị khóa, đang ở trang đăng nhập |
| **Test Data** | Email: `test@example.com` / Password: `Test@123` |
| **Step 1** | Nhập `test@example.com` vào trường Email |
| **Step 2** | Nhập `Test@123` vào trường Password |
| **Step 3** | Click nút "Đăng nhập" |
| **Expected Result** | Hệ thống chuyển hướng đến trang Dashboard (`/dashboard`). Hiển thị tên người dùng ở header. Không hiển thị thông báo lỗi. |
| **Actual Result** | *(Điền sau khi thực thi)* |
| **Status** | *(Pass / Fail)* |

---

**TC_LOGIN_002 — Đăng nhập thất bại với password sai**

| Trường | Nội dung |
|---|---|
| **ID** | TC_LOGIN_002 |
| **Title** | Đăng nhập thất bại khi nhập password sai |
| **Priority** | High |
| **Preconditions** | Tài khoản `test@example.com` tồn tại, chưa bị khóa, đang ở trang đăng nhập |
| **Test Data** | Email: `test@example.com` / Password: `WrongPass123` |
| **Step 1** | Nhập `test@example.com` vào trường Email |
| **Step 2** | Nhập `WrongPass123` vào trường Password |
| **Step 3** | Click nút "Đăng nhập" |
| **Expected Result** | Hệ thống ở lại trang đăng nhập. Hiển thị thông báo: *"Email hoặc mật khẩu không đúng"* (không nêu rõ là email hay password sai — bảo mật). Trường Password bị xóa. URL không thay đổi. |
| **Actual Result** | *(Điền sau khi thực thi)* |
| **Status** | *(Pass / Fail)* |

---

**TC_LOGIN_003 — Khóa tài khoản sau 5 lần sai password**

| Trường | Nội dung |
|---|---|
| **ID** | TC_LOGIN_003 |
| **Title** | Tài khoản bị khóa sau 5 lần nhập sai password liên tiếp |
| **Priority** | High |
| **Preconditions** | Tài khoản `test@example.com` tồn tại, số lần sai hiện tại = 0 |
| **Test Data** | Email: `test@example.com` / Password sai: `Wrong@123` |
| **Step 1** | Thực hiện đăng nhập với password sai 4 lần liên tiếp |
| **Step 2** | Nhập đúng email, nhập password sai lần thứ 5 |
| **Step 3** | Click nút "Đăng nhập" |
| **Expected Result** | Hiển thị thông báo: *"Tài khoản của bạn đã bị tạm khóa do đăng nhập sai nhiều lần. Vui lòng thử lại sau 30 phút."* Tài khoản không thể đăng nhập ngay cả với password đúng trong 30 phút tiếp theo. |
| **Actual Result** | *(Điền sau khi thực thi)* |
| **Status** | *(Pass / Fail)* |

---

**TC_LOGIN_004 — Validation email trống**

| Trường | Nội dung |
|---|---|
| **ID** | TC_LOGIN_004 |
| **Title** | Hiển thị lỗi validation khi bỏ trống trường Email |
| **Priority** | Medium |
| **Preconditions** | Đang ở trang đăng nhập |
| **Test Data** | Email: *(để trống)* / Password: `Test@123` |
| **Step 1** | Bỏ trống trường Email |
| **Step 2** | Nhập `Test@123` vào trường Password |
| **Step 3** | Click nút "Đăng nhập" |
| **Expected Result** | Hệ thống không gửi request. Hiển thị thông báo validation bên dưới trường Email: *"Vui lòng nhập email"*. Focus tự động vào trường Email. |
| **Actual Result** | *(Điền sau khi thực thi)* |
| **Status** | *(Pass / Fail)* |

---

### 2.2.2 Thực hành: Test Case cho tính năng Register

**Acceptance Criteria:**
- Email phải đúng định dạng và chưa được đăng ký
- Password tối thiểu 8 ký tự, có chữ hoa, chữ thường, số, ký tự đặc biệt
- Confirm Password phải khớp với Password
- Sau khi đăng ký thành công, gửi email xác nhận

---

**TC_REG_001 — Đăng ký thành công**

| Trường | Nội dung |
|---|---|
| **ID** | TC_REG_001 |
| **Title** | Đăng ký thành công với đầy đủ thông tin hợp lệ |
| **Priority** | Critical |
| **Preconditions** | Email `newuser@example.com` chưa được đăng ký. Đang ở trang đăng ký. |
| **Test Data** | Email: `newuser@example.com` / Password: `NewPass@123` / Confirm: `NewPass@123` |
| **Step 1** | Nhập `newuser@example.com` vào trường Email |
| **Step 2** | Nhập `NewPass@123` vào trường Password |
| **Step 3** | Nhập `NewPass@123` vào trường Confirm Password |
| **Step 4** | Click nút "Đăng ký" |
| **Expected Result** | Hiển thị thông báo thành công: *"Đăng ký thành công! Vui lòng kiểm tra email để xác nhận tài khoản."* Gửi email xác nhận đến `newuser@example.com`. Tài khoản được tạo trong DB với `status = pending_verification`. |
| **Actual Result** | *(Điền sau khi thực thi)* |
| **Status** | *(Pass / Fail)* |

---

**TC_REG_002 — Email đã tồn tại**

| Trường | Nội dung |
|---|---|
| **ID** | TC_REG_002 |
| **Title** | Đăng ký thất bại khi sử dụng email đã tồn tại |
| **Priority** | High |
| **Preconditions** | Tài khoản `existing@example.com` đã được đăng ký. Đang ở trang đăng ký. |
| **Test Data** | Email: `existing@example.com` / Password: `Test@123` / Confirm: `Test@123` |
| **Step 1** | Nhập `existing@example.com` vào trường Email |
| **Step 2** | Nhập `Test@123` vào trường Password |
| **Step 3** | Nhập `Test@123` vào trường Confirm Password |
| **Step 4** | Click nút "Đăng ký" |
| **Expected Result** | Hiển thị thông báo lỗi bên dưới trường Email: *"Email này đã được sử dụng. Vui lòng dùng email khác hoặc đăng nhập."* Không tạo tài khoản mới. Không gửi email. |
| **Actual Result** | *(Điền sau khi thực thi)* |
| **Status** | *(Pass / Fail)* |

---

**TC_REG_003 — Password không khớp với Confirm Password**

| Trường | Nội dung |
|---|---|
| **ID** | TC_REG_003 |
| **Title** | Đăng ký thất bại khi Confirm Password không khớp |
| **Priority** | High |
| **Preconditions** | Đang ở trang đăng ký |
| **Test Data** | Email: `test2@example.com` / Password: `Test@123` / Confirm: `Test@456` |
| **Step 1** | Nhập `test2@example.com` vào trường Email |
| **Step 2** | Nhập `Test@123` vào trường Password |
| **Step 3** | Nhập `Test@456` vào trường Confirm Password |
| **Step 4** | Click nút "Đăng ký" |
| **Expected Result** | Hiển thị thông báo lỗi bên dưới trường Confirm Password: *"Mật khẩu xác nhận không khớp."* Không gửi request lên server. |
| **Actual Result** | *(Điền sau khi thực thi)* |
| **Status** | *(Pass / Fail)* |

---

## 2.3 Kỹ Thuật Thiết Kế Test Case

Kỹ thuật thiết kế test case giúp Tester chọn lọc test case **hiệu quả nhất** trong vô số test case có thể viết — đảm bảo phủ rộng các tình huống với số lượng test case tối ưu.

### 2.3.1 Equivalence Partitioning (EP) — Phân hoạch tương đương

**Bản chất:** EP chia tập hợp đầu vào thành các **phân lớp tương đương** — nhóm các giá trị mà phần mềm xử lý theo cùng một cách. Lý thuyết: nếu một giá trị trong phân lớp phát hiện lỗi, mọi giá trị khác trong cùng phân lớp cũng sẽ phát hiện lỗi đó. Do đó chỉ cần kiểm thử **một đại diện** từ mỗi phân lớp.

**Các loại phân lớp:**
- **Valid partition:** Đầu vào hợp lệ mà hệ thống chấp nhận
- **Invalid partition:** Đầu vào không hợp lệ mà hệ thống phải từ chối

**Ví dụ thực hành — Trường "Tuổi" với yêu cầu: 18 ≤ tuổi ≤ 60:**

| Phân lớp | Khoảng giá trị | Đại diện | Loại |
|---|---|---|---|
| EP1 | < 18 | 10 | Invalid |
| EP2 | 18 đến 60 | 35 | Valid |
| EP3 | > 60 | 75 | Invalid |

Thay vì kiểm thử 100 giá trị, chỉ cần 3 test case đại diện — một cho mỗi phân lớp.

**Ví dụ thực hành — Trường "Email":**

| Phân lớp | Mô tả | Đại diện | Loại |
|---|---|---|---|
| EP1 | Email đúng định dạng | `user@example.com` | Valid |
| EP2 | Không có ký tự @ | `userexample.com` | Invalid |
| EP3 | Không có domain | `user@` | Invalid |
| EP4 | Không có local part | `@example.com` | Invalid |
| EP5 | Có khoảng trắng | `user @example.com` | Invalid |

---

### 2.3.2 Boundary Value Analysis (BVA) — Phân tích giá trị biên

**Bản chất:** Lỗi có xu hướng xảy ra tại **ranh giới** của các phân lớp tương đương hơn là ở giữa. Lập trình viên thường mắc lỗi với điều kiện biên: dùng `>` thay vì `>=`, hoặc bỏ quên giá trị cuối phạm vi. BVA bổ sung cho EP bằng cách kiểm thử các giá trị tại và xung quanh ranh giới.

**Các điểm biên cần kiểm thử:**
- Giá trị ngay tại ranh giới (on-point)
- Giá trị ngay dưới ranh giới (off-point dưới)
- Giá trị ngay trên ranh giới (off-point trên)

**Ví dụ thực hành — Trường "Tuổi": 18 ≤ tuổi ≤ 60:**

| Giá trị | Mô tả | Kết quả mong đợi |
|---|---|---|
| 17 | Dưới ranh giới dưới | Invalid — từ chối |
| **18** | **Ranh giới dưới** | **Valid — chấp nhận** |
| 19 | Trên ranh giới dưới | Valid — chấp nhận |
| 35 | Giữa khoảng | Valid — chấp nhận (từ EP) |
| 59 | Dưới ranh giới trên | Valid — chấp nhận |
| **60** | **Ranh giới trên** | **Valid — chấp nhận** |
| 61 | Trên ranh giới trên | Invalid — từ chối |

Kết hợp EP + BVA: 3 đại diện EP + 4 giá trị biên bổ sung = 7 test case phủ toàn bộ.

**Ví dụ thực hành — Trường "Password": tối thiểu 8, tối đa 20 ký tự:**

| Giá trị | Ký tự | Kết quả mong đợi |
|---|---|---|
| 7 ký tự | `Abc@123` | Invalid |
| **8 ký tự** | `Abc@1234` | **Valid** |
| 9 ký tự | `Abc@12345` | Valid |
| 19 ký tự | `Abc@1234567890123` | Valid |
| **20 ký tự** | `Abc@12345678901234` | **Valid** |
| 21 ký tự | `Abc@123456789012345` | Invalid |

---

### 2.3.3 Decision Table Testing — Kiểm thử bảng quyết định

**Bản chất:** Kỹ thuật này dùng khi chức năng có nhiều **điều kiện kết hợp** tác động đến kết quả. Bảng quyết định liệt kê tất cả tổ hợp điều kiện và kết quả tương ứng, đảm bảo không bỏ sót tổ hợp nào.

**Cấu trúc bảng quyết định:**
- Hàng trên: các điều kiện (conditions)
- Hàng dưới: các hành động (actions/results)
- Cột: mỗi cột là một tổ hợp điều kiện

**Ví dụ thực hành — Logic áp dụng mã giảm giá:**

**Quy tắc nghiệp vụ:**
- Khách hàng VIP được giảm 10%
- Đơn hàng trên 1,000,000 VNĐ được giảm thêm 5%
- Nếu cả hai điều kiện đều đúng, giảm tổng cộng 15%
- Nếu không có điều kiện nào, không giảm

| Điều kiện / Tổ hợp | TC1 | TC2 | TC3 | TC4 |
|---|---|---|---|---|
| Khách VIP | T | T | F | F |
| Đơn > 1,000,000đ | T | F | T | F |
| **Kết quả: Giảm giá** | **15%** | **10%** | **5%** | **0%** |

Từ bảng quyết định trên, cần 4 test case để phủ toàn bộ tổ hợp logic.

**Ví dụ thực hành — Logic giao dịch ngân hàng:**

**Quy tắc:** Cho phép rút tiền khi: Tài khoản đủ số dư VÀ thẻ không bị khóa VÀ số tiền ≤ hạn mức ngày

| Điều kiện | TC1 | TC2 | TC3 | TC4 | TC5 | TC6 | TC7 | TC8 |
|---|---|---|---|---|---|---|---|---|
| Đủ số dư | T | T | T | T | F | F | F | F |
| Thẻ không bị khóa | T | T | F | F | T | T | F | F |
| Trong hạn mức ngày | T | F | T | F | T | F | T | F |
| **Cho phép rút** | **Có** | Không | Không | Không | Không | Không | Không | Không |

---

### 2.3.4 State Transition Testing — Kiểm thử chuyển trạng thái

**Bản chất:** Kỹ thuật này áp dụng khi hệ thống có các **trạng thái** khác nhau và hành vi phụ thuộc vào trạng thái hiện tại. Tester vẽ sơ đồ trạng thái và kiểm thử tất cả các chuyển tiếp hợp lệ và không hợp lệ.

**Ví dụ thực hành — Vòng đời tài khoản người dùng:**

```
[Mới tạo]
    ↓ Gửi email xác nhận
[Chờ xác nhận]
    ↓ Click link xác nhận      ↓ Link hết hạn (24h)
[Đã kích hoạt]            [Link hết hạn]
    ↓ Đăng nhập sai 5 lần       ↓ Yêu cầu gửi lại
[Bị khóa]                 [Chờ xác nhận]
    ↓ 30 phút / Admin mở khóa
[Đã kích hoạt]
    ↓ Admin vô hiệu hóa
[Bị vô hiệu hóa]
    ↓ Admin kích hoạt lại
[Đã kích hoạt]
```

**Test Case từ State Transition:**

| TC | Trạng thái ban đầu | Hành động | Trạng thái kết quả |
|---|---|---|---|
| TC_ST_001 | Chờ xác nhận | Click link xác nhận hợp lệ | Đã kích hoạt |
| TC_ST_002 | Chờ xác nhận | Link xác nhận hết hạn | Link hết hạn |
| TC_ST_003 | Đã kích hoạt | Sai password 5 lần | Bị khóa |
| TC_ST_004 | Bị khóa | Đăng nhập với password đúng | Vẫn bị khóa (không cho vào) |
| TC_ST_005 | Bị khóa | Chờ 30 phút → đăng nhập đúng | Đã kích hoạt |
| TC_ST_006 | Đã kích hoạt | Admin vô hiệu hóa | Bị vô hiệu hóa |
| TC_ST_007 | Chờ xác nhận | Thử đăng nhập | Lỗi: tài khoản chưa xác nhận |

---

### 2.3.5 Use Case Testing

**Bản chất:** Kỹ thuật này thiết kế test case từ Use Case — mô tả các luồng tương tác giữa actor (người dùng/hệ thống) và hệ thống để đạt một mục tiêu cụ thể. Test case bao gồm luồng chính (main flow) và các luồng ngoại lệ (alternate/exception flows).

**Ví dụ thực hành — Use Case: Đặt phòng khách sạn:**

**Main Flow (Luồng chính):**
1. Người dùng tìm kiếm phòng theo ngày và địa điểm
2. Hệ thống hiển thị danh sách phòng trống
3. Người dùng chọn phòng
4. Người dùng điền thông tin khách
5. Người dùng thanh toán
6. Hệ thống xác nhận đặt phòng và gửi email

**Alternate Flows (Luồng thay thế):**
- 2a: Không có phòng trống → hiển thị thông báo và gợi ý ngày khác
- 5a: Thanh toán thất bại → thông báo lỗi, giữ nguyên thông tin
- 5b: Timeout thanh toán → hủy giữ phòng, thông báo người dùng

**Test Case từ Use Case:**

| TC | Luồng | Mô tả |
|---|---|---|
| TC_UC_001 | Main Flow | Đặt phòng thành công với tất cả thông tin hợp lệ |
| TC_UC_002 | 2a | Tìm kiếm không có kết quả |
| TC_UC_003 | 5a | Thanh toán bị từ chối bởi ngân hàng |
| TC_UC_004 | 5b | Timeout sau 15 phút không hoàn thành thanh toán |

---

### 2.3.6 Pairwise Testing (All-Pairs Testing)

**Bản chất:** Khi có nhiều tham số đầu vào với nhiều giá trị, số tổ hợp tăng theo cấp số nhân. Pairwise testing dựa trên nghiên cứu thực nghiệm rằng phần lớn lỗi được phát hiện bởi sự kết hợp của **tối đa 2 tham số**. Do đó, đảm bảo mọi cặp giá trị của 2 tham số bất kỳ đều được kiểm thử ít nhất một lần — giảm đáng kể số test case cần thiết.

**Ví dụ thực hành — Kiểm thử tương thích:**

| Tham số | Giá trị |
|---|---|
| Trình duyệt | Chrome, Firefox, Safari |
| Hệ điều hành | Windows, macOS, Linux |
| Ngôn ngữ | Tiếng Việt, Tiếng Anh |

Toàn bộ tổ hợp: 3 × 3 × 2 = **18 test case**

Pairwise giảm xuống còn **9 test case** mà vẫn đảm bảo mọi cặp giá trị đều được kiểm thử ít nhất một lần:

| TC | Trình duyệt | Hệ điều hành | Ngôn ngữ |
|---|---|---|---|
| 1 | Chrome | Windows | Tiếng Việt |
| 2 | Chrome | macOS | Tiếng Anh |
| 3 | Chrome | Linux | Tiếng Anh |
| 4 | Firefox | Windows | Tiếng Anh |
| 5 | Firefox | macOS | Tiếng Việt |
| 6 | Firefox | Linux | Tiếng Việt |
| 7 | Safari | Windows | Tiếng Anh |
| 8 | Safari | macOS | Tiếng Việt |
| 9 | Safari | Linux | Tiếng Anh |

---

### 2.3.7 Error Guessing — Đoán lỗi

**Bản chất:** Error guessing là kỹ thuật dựa vào **kinh nghiệm, kiến thức domain, và trực giác** của Tester để dự đoán những nơi lỗi có thể ẩn náu — những tình huống mà developer thường quên xử lý.

**Danh sách các "điểm nguy hiểm" phổ biến:**

**Giá trị đặc biệt:**
- Giá trị 0 (chia cho 0, giỏ hàng 0 sản phẩm)
- Giá trị âm (-1, -100)
- Giá trị rất lớn (MAX_INT, 999999999)
- Chuỗi rỗng `""`
- Khoảng trắng ` `
- Null / None / undefined

**Ký tự đặc biệt:**
- SQL injection: `'; DROP TABLE users; --`
- XSS: `<script>alert('xss')</script>`
- Ký tự đặc biệt: `!@#$%^&*()`
- Unicode, emoji: `😀`, `Ä`, `中文`
- Ký tự xuống dòng: `\n`, `\r\n`

**Điều kiện biên thời gian:**
- Ngày 29/2 (năm nhuận và không nhuận)
- Ngày cuối tháng (30, 31)
- Thay đổi múi giờ
- Ngày hết hạn vào đúng ngày hiện tại

**Trạng thái đặc biệt:**
- Thao tác đồng thời (hai người mua sản phẩm cuối cùng cùng lúc)
- Mất kết nối giữa chừng
- Timeout
- Tải file kích thước 0 byte hoặc file rất lớn

---

### 2.3.8 Checklist-based Testing

**Bản chất:** Tester sử dụng checklist — danh sách các điểm cần kiểm tra — được tổng hợp từ kinh nghiệm, tiêu chuẩn, và yêu cầu dự án. Checklist đảm bảo các điểm quan trọng không bị bỏ sót.

**Ví dụ Checklist cho Form (áp dụng cho mọi form trong dự án):**

```
UI & Layout
□ Tất cả các trường hiển thị đúng, không bị cắt
□ Placeholder text rõ ràng, mô tả đúng
□ Thứ tự tab hợp lý (Tab key di chuyển đúng thứ tự)
□ Responsive trên mobile, tablet, desktop

Validation
□ Trường bắt buộc hiển thị lỗi khi để trống
□ Định dạng email hợp lệ được kiểm tra
□ Giới hạn độ dài được thực thi (cả client và server)
□ Ký tự đặc biệt được xử lý đúng
□ Thông báo lỗi rõ ràng, chỉ đúng trường bị lỗi

Bảo mật
□ Trường password ẩn ký tự
□ Không autocomplete cho trường nhạy cảm
□ Dữ liệu được validate ở server-side (không chỉ client)
□ CSRF token có mặt

Chức năng
□ Submit với đầy đủ dữ liệu hợp lệ → thành công
□ Submit với dữ liệu không hợp lệ → lỗi đúng
□ Double submit (click nút 2 lần nhanh) → không tạo duplicate
□ Back button sau khi submit → không resubmit

Trải nghiệm
□ Thông báo thành công rõ ràng
□ Redirect đúng trang sau thành công
□ Loading indicator khi đang submit
```

---

### 2.3.9 Combinatorial Testing

**Bản chất:** Là phiên bản tổng quát hóa của Pairwise Testing — thay vì đảm bảo mọi cặp 2 tham số (t=2), Combinatorial Testing đảm bảo mọi tổ hợp t tham số đều được kiểm thử (t có thể là 2, 3, 4...). Khi t tăng, độ phủ tăng nhưng số test case cũng tăng. Công cụ như ACTS (Automated Combinatorial Testing for Software) tự động sinh test case.

**Khi nào dùng Combinatorial Testing:**
- Hệ thống có nhiều tham số cấu hình tương tác phức tạp
- Yêu cầu độ phủ cao hơn Pairwise
- Kiểm thử embedded system, giao thức mạng, cấu hình server

---

### 2.3.10 Tổng hợp: Khi nào dùng kỹ thuật nào?

| Tình huống | Kỹ thuật phù hợp |
|---|---|
| Trường có khoảng giá trị liên tục | EP + BVA |
| Logic nhiều điều kiện kết hợp | Decision Table |
| Hệ thống có nhiều trạng thái | State Transition |
| Có Use Case đặc tả đầy đủ | Use Case Testing |
| Nhiều tham số, ít thời gian | Pairwise Testing |
| Kiểm thử bổ sung, tìm lỗi ẩn | Error Guessing |
| Kiểm thử lặp lại theo chuẩn | Checklist-based |
| Tổ hợp tham số phức tạp, yêu cầu cao | Combinatorial Testing |

> **Thực tế:** Một Tester giỏi không chỉ dùng một kỹ thuật mà **kết hợp nhiều kỹ thuật** cho cùng một chức năng. Ví dụ: dùng EP + BVA cho các trường riêng lẻ, Decision Table cho logic nghiệp vụ, và Error Guessing để kiểm tra thêm các edge case.

---

