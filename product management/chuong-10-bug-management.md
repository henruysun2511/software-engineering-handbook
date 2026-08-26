# CHƯƠNG 10: BUG MANAGEMENT — QUẢN LÝ LỖI PHẦN MỀM

---

## 10.1. Bug trong Jira là gì?

Trong phát triển phần mềm, **Bug** (lỗi) là bất kỳ hành vi nào của hệ thống không khớp với yêu cầu đã được xác định hoặc kỳ vọng hợp lý của người dùng. Bug có thể là lỗi chức năng (tính năng không hoạt động đúng), lỗi giao diện (hiển thị sai), lỗi hiệu năng (tải quá chậm) hay lỗi bảo mật (dữ liệu bị lộ).

Trong Jira, **Bug** là một loại Issue chuyên biệt được thiết kế để ghi nhận, theo dõi và quản lý toàn bộ vòng đời xử lý lỗi — từ lúc phát hiện cho đến khi xác nhận đã được sửa hoàn toàn.

Khác với Story hay Task mô tả công việc *cần làm*, Bug mô tả một vấn đề *đang tồn tại* trong hệ thống. Sự phân biệt này quan trọng vì Bug thường mang tính phản ứng (reactive) — phát sinh ngoài kế hoạch — trong khi Story và Task là công việc được lên kế hoạch chủ động.

---

## 10.2. Tại sao cần quản lý Bug có hệ thống?

Không có phần mềm nào hoàn toàn không có lỗi. Điều quan trọng không phải là tránh hoàn toàn Bug, mà là phát hiện, ghi nhận và xử lý Bug một cách có hệ thống. Quản lý Bug tốt mang lại:

- **Không bỏ sót lỗi:** Mọi Bug được ghi nhận đều được theo dõi đến khi giải quyết xong.
- **Ưu tiên đúng:** Không phải Bug nào cũng cần sửa ngay — hệ thống giúp phân loại và xử lý theo mức độ nghiêm trọng.
- **Trách nhiệm rõ ràng:** Mỗi Bug được phân công cho một người cụ thể chịu trách nhiệm.
- **Lịch sử đầy đủ:** Toàn bộ quá trình xử lý được lưu lại, giúp tra cứu và phân tích về sau.
- **Đo lường chất lượng:** Số lượng và tốc độ xử lý Bug là chỉ số quan trọng để đánh giá chất lượng phần mềm và hiệu quả của quy trình kiểm thử.

---

## 10.3. Tạo Bug Report

**Bug Report** là bản mô tả chi tiết về một lỗi phần mềm. Chất lượng của Bug Report ảnh hưởng trực tiếp đến tốc độ và hiệu quả xử lý — một Bug Report rõ ràng giúp Developer tái hiện và sửa lỗi nhanh hơn nhiều so với một mô tả mơ hồ.

### Cách tạo Bug trong Jira:

**Bước 1:** Nhấn **Create** trên thanh điều hướng (hoặc phím tắt **C**).

**Bước 2:** Chọn **Issue Type = Bug**.

**Bước 3:** Điền đầy đủ thông tin theo cấu trúc Bug Report chuẩn (xem mục 10.4).

**Bước 4:** Nhấn **Create** để lưu. Bug mới sẽ xuất hiện trong Backlog với loại Issue là Bug.

---

## 10.4. Cấu trúc một Bug Report chuẩn

Một Bug Report đầy đủ cần có các thành phần sau:

### 10.4.1. Summary (Tiêu đề)

Tiêu đề ngắn gọn, mô tả rõ ràng lỗi gì xảy ra ở đâu. Tránh tiêu đề chung chung như *"Lỗi trang thanh toán"* — thay bằng *"Người dùng không thể hoàn tất thanh toán khi dùng thẻ Visa trên Safari"*.

Công thức tiêu đề hiệu quả: **[Đối tượng] + [Hành động] + [Kết quả không mong muốn]**

### 10.4.2. Description — Mô tả lỗi

Phần mô tả tổng quan ngắn gọn về lỗi: lỗi xảy ra trong tình huống nào, ảnh hưởng đến chức năng gì, mức độ ảnh hưởng đến người dùng ra sao.

### 10.4.3. Steps to Reproduce (Các bước tái hiện)

Đây là phần **quan trọng nhất** của Bug Report. Liệt kê từng bước cụ thể để tái hiện lỗi, bất kỳ ai đọc vào cũng có thể làm theo và thấy lỗi tương tự.

Ví dụ:
```
1. Đăng nhập vào tài khoản người dùng thông thường
2. Thêm ít nhất 1 sản phẩm vào giỏ hàng
3. Vào trang Checkout
4. Chọn phương thức thanh toán "Thẻ tín dụng Visa"
5. Điền đầy đủ thông tin thẻ hợp lệ
6. Nhấn nút "Thanh toán ngay"
```

### 10.4.4. Expected Result (Kết quả mong đợi)

Mô tả điều gì *nên* xảy ra nếu hệ thống hoạt động đúng.

Ví dụ: *"Hệ thống xử lý thanh toán thành công và chuyển hướng người dùng đến trang xác nhận đơn hàng."*

### 10.4.5. Actual Result (Kết quả thực tế)

Mô tả điều gì *thực sự* xảy ra — hành vi sai lệch của hệ thống.

Ví dụ: *"Trang trắng hiện ra, không có thông báo lỗi, đơn hàng không được tạo nhưng tiền vẫn bị trừ khỏi tài khoản."*

### 10.4.6. Environment (Môi trường)

Thông tin về môi trường khi lỗi xảy ra. Thông tin này giúp Developer tái hiện lỗi trong đúng điều kiện.

```
- Môi trường: Production / Staging / Development
- Trình duyệt: Safari 17.2 trên macOS Sonoma 14.1
- Thiết bị: MacBook Pro M2
- Phiên bản ứng dụng: v2.4.1
- Tài khoản test: user_test@example.com
```

### 10.4.7. Priority và Severity

**Priority** (Mức độ ưu tiên) phản ánh *tầm quan trọng kinh doanh* — Bug này cần sửa nhanh đến mức nào?

**Severity** (Mức độ nghiêm trọng) phản ánh *mức độ ảnh hưởng kỹ thuật* — Bug này phá vỡ hệ thống nghiêm trọng đến mức nào?

| Severity | Mô tả |
|---|---|
| **Critical / Blocker** | Hệ thống hoàn toàn không hoạt động, không có workaround |
| **Major / High** | Chức năng quan trọng bị ảnh hưởng, workaround khó |
| **Medium** | Chức năng bị ảnh hưởng nhưng có workaround |
| **Minor / Low** | Lỗi nhỏ, hầu như không ảnh hưởng đến người dùng |

Hai tiêu chí này đôi khi không đồng nhất. Ví dụ: lỗi chính tả trên trang đăng nhập (Severity: Minor) nhưng vì trang đăng nhập là mặt tiền của sản phẩm nên Priority có thể là High.

### 10.4.8. Attachment (Tệp đính kèm)

Đính kèm **ảnh chụp màn hình** hoặc **video ghi lại màn hình** minh họa lỗi. Đây là bằng chứng trực quan giúp Developer hiểu ngay vấn đề mà không cần tự tái hiện. Đối với các lỗi phức tạp, đính kèm thêm **log lỗi** từ console hoặc server.

---

## 10.5. Bug Workflow — Vòng đời của một Bug

Bug có vòng đời riêng biệt so với Story hay Task, phản ánh chu trình phát hiện — xử lý — xác nhận đặc thù của lỗi phần mềm.

```
Open (Mới phát hiện)
  ↓
In Progress (Developer đang sửa)
  ↓
Resolved / Fixed (Đã sửa, chờ QA kiểm tra)
  ↓
  ├── Testing passed → Closed (Xác nhận đã sửa xong)
  └── Testing failed → Reopened (Lỗi vẫn còn)
                          ↓
                       In Progress (Tiếp tục sửa)
```

### Giải thích các trạng thái:

**Open:** Bug vừa được tạo và chưa có người nhận xử lý. Đây là trạng thái mặc định khi Bug được tạo mới.

**In Progress:** Developer đã nhận Bug, đang phân tích nguyên nhân và viết bản sửa lỗi (fix). Bug nên chỉ ở trạng thái này khi Developer đang tích cực làm việc trên nó — không phải khi đang chờ thông tin thêm.

**Resolved / Fixed:** Developer tuyên bố đã sửa xong Bug và cần QA kiểm tra lại. Code fix đã được merge vào nhánh chính hoặc môi trường Staging.

**Closed:** QA đã kiểm tra và xác nhận Bug được sửa hoàn toàn. Không còn tái hiện lỗi trong điều kiện ban đầu. Đây là trạng thái kết thúc của Bug.

**Reopened:** QA kiểm tra và phát hiện lỗi vẫn còn tồn tại hoặc sửa Bug này gây ra Bug khác (regression). Bug được mở lại và quay về "In Progress".

**Won't Fix / Rejected:** Bug được ghi nhận nhưng nhóm quyết định không sửa — có thể vì mức độ ảnh hưởng quá nhỏ, chi phí sửa quá cao so với lợi ích, hoặc hành vi đó thực ra là tính năng chủ ý (not a bug).

---

## 10.6. Assign Bug (Phân công xử lý)

Sau khi Bug được tạo, bước tiếp theo là phân công cho đúng người. Có hai cách phân công phổ biến:

**Phân công bởi Scrum Master / Tech Lead:** Người quản lý xem xét danh sách Bug và phân công dựa trên chuyên môn của từng Developer và tải công việc hiện tại.

**Tự nhận (Self-assign):** Developer chủ động nhận Bug từ Backlog — phù hợp với văn hóa nhóm tự quản lý. Để tự nhận, mở Bug → nhấn **Assignee → Assign to me**.

### Nguyên tắc phân công Bug:

- Phân công dựa trên **chuyên môn** — Developer nào quen với module liên quan nên nhận Bug đó
- Không nên để Bug quan trọng ở trạng thái **Open** mà không có Assignee quá 24 giờ
- Đối với Bug Critical/Blocker, phân công ngay lập tức và đặt Due Date rõ ràng

---

## 10.7. Theo dõi Bug

### Theo dõi tổng thể bằng JQL:

```jql
-- Tất cả Bug chưa xử lý, sắp xếp theo Priority
issuetype = Bug AND status NOT IN (Closed, "Won't Fix") ORDER BY priority DESC

-- Bug Critical chưa có người nhận
issuetype = Bug AND priority = Highest AND assignee IS EMPTY

-- Bug được Reopen trong tuần này
issuetype = Bug AND status = Reopened AND updated >= startOfWeek()

-- Bug của tôi đang xử lý
issuetype = Bug AND assignee = currentUser() AND status = "In Progress"
```

### Bug Dashboard:

Tạo một Dashboard riêng cho Bug với các Gadget:
- **Issue Statistics:** Phân bổ Bug theo Status hoặc Priority
- **Created vs Resolved Chart:** So sánh số Bug mới tạo và số Bug đã đóng theo thời gian
- **Assigned to Me:** Danh sách Bug đang được giao cho tôi

---

## 10.8. Reopen Bug — Mở lại Bug

**Reopen** là hành động đặc trưng trong Bug Workflow, xảy ra khi QA kiểm tra Bug đã được tuyên bố Resolved nhưng phát hiện lỗi vẫn còn tồn tại.

### Quy trình Reopen đúng cách:

**Bước 1:** QA kiểm tra Bug trên đúng môi trường và phiên bản mà Developer đã sửa.

**Bước 2:** Tái hiện lại lỗi theo đúng Steps to Reproduce trong Bug Report gốc.

**Bước 3:** Nếu lỗi vẫn còn — chuyển Bug về trạng thái **Reopened**.

**Bước 4:** Thêm Comment mô tả chi tiết: lỗi vẫn tái hiện như thế nào, trên môi trường nào, phiên bản nào. Đính kèm ảnh chụp màn hình mới nếu cần.

**Bước 5:** Ping Developer qua Mention để thông báo Bug được Reopen.

### Lưu ý khi Reopen:

- Không Reopen nếu điều kiện kiểm tra khác với điều kiện tái hiện ban đầu — đây có thể là một Bug mới, cần tạo Issue riêng.
- Ghi rõ nguyên nhân Reopen trong Comment để Developer hiểu ngay vấn đề là gì.
- Nếu một Bug bị Reopen nhiều lần, đây là tín hiệu cần xem lại quy trình Code Review và Testing.

---

## 10.9. Các chỉ số đo lường chất lượng qua Bug

Dữ liệu Bug trong Jira cung cấp thông tin quý giá để đánh giá chất lượng phần mềm và hiệu quả quy trình:

| Chỉ số | Ý nghĩa |
|---|---|
| **Bug Escape Rate** | Tỷ lệ Bug do người dùng phát hiện trên production so với Bug phát hiện trong testing |
| **Mean Time to Resolve (MTTR)** | Thời gian trung bình từ khi Bug được Open đến khi Closed |
| **Reopen Rate** | Tỷ lệ Bug bị Reopen — chỉ số phản ánh chất lượng fix và testing |
| **Bug Density** | Số lượng Bug trên mỗi tính năng hoặc module — chỉ ra khu vực code kém chất lượng |

---

## Tóm tắt chương

Quản lý Bug hiệu quả đòi hỏi sự phối hợp chặt chẽ giữa QA (tạo Bug Report chất lượng cao), Developer (xử lý nhanh và đúng nguyên nhân) và Scrum Master (ưu tiên và theo dõi tiến độ). Jira cung cấp đầy đủ công cụ để thực hiện toàn bộ vòng đời Bug — từ khi phát hiện đến khi xác nhận đã sửa xong.

| Khái niệm | Ý nghĩa ngắn gọn |
|---|---|
| Bug | Loại Issue ghi nhận lỗi phần mềm |
| Bug Report | Bản mô tả chi tiết về một lỗi |
| Steps to Reproduce | Các bước tái hiện lỗi |
| Expected vs Actual Result | Kết quả mong đợi và thực tế |
| Severity | Mức độ ảnh hưởng kỹ thuật của Bug |
| Priority | Mức độ ưu tiên xử lý theo góc nhìn kinh doanh |
| Reopened | Trạng thái Bug được mở lại sau khi fix chưa thành công |
| Reopen Rate | Tỷ lệ Bug bị mở lại — chỉ số chất lượng |

---

*Chương tiếp theo: **Chương 11 — Dashboard & Reports: Báo cáo và Thống kê***
