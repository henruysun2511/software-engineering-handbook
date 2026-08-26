# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# CHƯƠNG 3: BUG & TEST MANAGEMENT

---

## 3.1 Bug / Defect Management

### 3.1.1 Bug là gì? Phân biệt Bug, Defect, Issue

Trong ngữ cảnh kiểm thử hàng ngày, **Bug**, **Defect**, và **Issue** thường được dùng thay thế cho nhau, nhưng có sự khác biệt tinh tế:

- **Bug:** Thuật ngữ thông dụng, chỉ bất kỳ sự khác biệt nào giữa hành vi thực tế và hành vi mong đợi. Nguồn gốc lịch sử từ năm 1947 khi một con ngài (moth) kẹt vào máy tính Harvard Mark II gây ra sự cố — Grace Hopper đã ghi lại là "bug" đầu tiên.

- **Defect:** Thuật ngữ kỹ thuật chính thức hơn, dùng trong quy trình và báo cáo. Nhấn mạnh khiếm khuyết trong sản phẩm.

- **Issue:** Thuật ngữ rộng hơn — có thể là bug, có thể là yêu cầu cải tiến, hoặc vấn đề về quy trình. Thường dùng trong Jira để gộp chung nhiều loại.

> **Thực tế làm việc:** Trong giao tiếp hàng ngày, dùng "bug" là bình thường. Trong báo cáo chính thức và tài liệu, dùng "defect". Khi tạo ticket trong Jira, phân biệt loại: Bug / Story / Task / Improvement.

---

### 3.1.2 Bug Life Cycle — Vòng đời của Bug

Bug Life Cycle là tập hợp các trạng thái mà một bug trải qua từ khi được phát hiện đến khi được đóng. Hiểu vòng đời giúp toàn bộ team phối hợp hiệu quả.

#### Sơ đồ Bug Life Cycle đầy đủ

```
                    ┌─────────────────────────────────┐
                    │                                 │
              [New/Open]                              │
           Tester phát hiện,                         │
           tạo bug report                            │
                    │                                 │
                    ▼                                 │
            [Assigned]                               │
           Manager/Lead                              │
           giao cho Dev                              │
                    │                                 │
         ┌──────────┴──────────┐                     │
         ▼                     ▼                     │
   [In Progress]        [Rejected]                   │
   Dev đang sửa         Dev từ chối:                 │
                        "Không phải bug"             │
                        "Won't fix"                  │
         │              "Duplicate"                  │
         ▼                                           │
    [Resolved/Fixed]                                 │
    Dev đã sửa xong,                                 │
    chờ Tester verify                                │
         │                                           │
    ┌────┴────┐                                      │
    ▼         ▼                                      │
 [Verified] [Reopened]──────────────────────────────┘
 Bug đã sửa  Bug chưa
 đúng, đóng  sửa đúng
    │
    ▼
 [Closed]
 Bug hoàn thành
 vòng đời
```

#### Mô tả chi tiết từng trạng thái

**New / Open:**
Tester vừa phát hiện và tạo bug report. Bug chưa được xem xét bởi team. Đây là trạng thái khởi đầu.

**Assigned:**
Manager hoặc Tech Lead đã xem xét bug, xác nhận đây là bug hợp lệ, và giao cho Developer cụ thể để sửa.

**In Progress:**
Developer đang phân tích và sửa bug. Tester không thực hiện verify trong giai đoạn này.

**Resolved / Fixed:**
Developer đã sửa xong và cập nhật trạng thái. Developer thường ghi thêm: sửa ở file nào, commit nào, nguyên nhân gốc rễ là gì. Tester bắt đầu retest.

**Verified:**
Tester đã retest và xác nhận bug được sửa đúng. Trạng thái này xác nhận fix hoạt động đúng với test case đã fail trước đó.

**Closed:**
Bug hoàn thành vòng đời. Trạng thái này thường được set bởi Manager hoặc tự động sau khi Verified. Bug đã closed không được mở lại — nếu lỗi tương tự xuất hiện, tạo bug mới và tham chiếu đến bug cũ.

**Reopened:**
Sau khi Developer báo Fixed, Tester retest nhưng bug vẫn còn. Tester cập nhật Actual Result mới, ghi chú và chuyển về Reopened. Bug quay lại chu trình từ Assigned.

**Rejected:**
Developer xem xét và kết luận đây không phải bug. Các lý do phổ biến:
- "Cannot Reproduce" — không tái hiện được
- "Not a Bug" — đây là hành vi đúng theo thiết kế
- "Duplicate" — bug này đã được báo cáo trước đó
- "Won't Fix" — bug được thừa nhận nhưng quyết định không sửa (rủi ro thấp, chi phí sửa cao)

> **Lưu ý:** Khi bug bị Rejected, Tester có quyền thảo luận lại. Nếu Tester tin rằng đây là bug thực sự, hãy cung cấp thêm bằng chứng: video, log, môi trường cụ thể. Không nên im lặng chấp nhận mà không có lý do thuyết phục.

---

### 3.1.3 Severity — Mức độ nghiêm trọng

**Severity** đo lường **tác động kỹ thuật** của bug đến hệ thống — bug gây hại bao nhiêu đến chức năng của phần mềm. Severity do **Tester** xác định khi báo cáo bug.

| Mức độ | Định nghĩa | Ví dụ |
|---|---|---|
| **Critical (S1)** | Hệ thống crash hoàn toàn, dữ liệu bị mất, chức năng cốt lõi không dùng được, không có workaround | Ứng dụng crash khi khởi động. Mất toàn bộ dữ liệu đơn hàng. API thanh toán luôn trả lỗi 500. |
| **High (S2)** | Chức năng chính bị ảnh hưởng nghiêm trọng, có workaround nhưng phức tạp | Không thể đặt hàng nhưng vẫn xem được sản phẩm. Không tìm kiếm được theo tên nhưng có thể browse danh mục. |
| **Medium (S3)** | Chức năng phụ bị ảnh hưởng, có workaround đơn giản | Bộ lọc giá không hoạt động nhưng có thể sắp xếp theo giá. Ảnh sản phẩm không load trong một số trường hợp. |
| **Low (S4)** | Vấn đề giao diện, lỗi chính tả, cải tiến nhỏ, không ảnh hưởng chức năng | Lỗi chính tả trong thông báo lỗi. Màu nút không đúng theo design. Tooltip hiển thị sai vị trí. |

---

### 3.1.4 Priority — Mức độ ưu tiên

**Priority** đo lường **mức độ khẩn cấp** cần sửa bug từ góc độ kinh doanh — bug này cần được sửa nhanh đến đâu. Priority do **Product Manager / Project Manager** xác định, dựa trên yếu tố kinh doanh.

| Mức độ | Định nghĩa | Ví dụ |
|---|---|---|
| **Urgent (P1)** | Phải sửa ngay lập tức, trước mọi việc khác | Bug ảnh hưởng trực tiếp đến doanh thu, vi phạm bảo mật, sự cố production đang diễn ra. |
| **High (P2)** | Phải sửa trong sprint hiện tại | Chức năng quan trọng bị ảnh hưởng, gần đến ngày release. |
| **Medium (P3)** | Sửa trong sprint tiếp theo | Bug rõ ràng nhưng không ảnh hưởng release hiện tại. |
| **Low (P4)** | Sửa khi có thời gian | Lỗi nhỏ, ít người dùng gặp, có thể đưa vào backlog. |

---

### 3.1.5 Severity vs Priority — Ma trận kết hợp và ví dụ thực tế

**Đây là điểm nhiều Tester mới nhầm lẫn.** Severity và Priority là hai chiều độc lập — một bug có thể có Severity cao nhưng Priority thấp, hoặc ngược lại.

**Ma trận 4 tình huống quan trọng:**

**Trường hợp 1: Severity HIGH + Priority HIGH**
- Bug: Chức năng thanh toán bị lỗi trên hệ thống thương mại điện tử đang chạy production
- Tại sao: Ảnh hưởng trực tiếp đến doanh thu ngay bây giờ → phải sửa ngay

**Trường hợp 2: Severity HIGH + Priority LOW**
- Bug: Chức năng "Xuất báo cáo theo năm" bị lỗi
- Tại sao: Chức năng này chỉ dùng 1 lần/năm vào cuối năm, hiện tại đang là tháng 3 → severity cao (chức năng không dùng được) nhưng priority thấp (chưa cần dùng đến)

**Trường hợp 3: Severity LOW + Priority HIGH**
- Bug: Logo công ty bị hiển thị sai trên trang chủ (méo, màu sai)
- Tại sao: Về kỹ thuật không ảnh hưởng chức năng nào (severity thấp) nhưng CEO vừa phát hiện, ngày mai có buổi demo với đối tác lớn → phải sửa ngay (priority cao)

**Trường hợp 4: Severity LOW + Priority LOW**
- Bug: Lỗi chính tả trong phần "Điều khoản sử dụng" mà rất ít người đọc
- Tại sao: Không ảnh hưởng chức năng, không ảnh hưởng kinh doanh → vào backlog

> **Nguyên tắc:** Tester xác định Severity dựa trên đánh giá kỹ thuật khách quan. Priority là quyết định kinh doanh của PM — Tester không nên tự ý set Priority cao để "ép" Developer sửa nhanh.

---

## 3.2 Cách Viết Bug Report Hiệu Quả

### 3.2.1 Tại sao Bug Report quan trọng?

Một bug report tốt giúp Developer:
- Hiểu chính xác lỗi là gì
- Tái hiện lỗi nhanh chóng
- Xác định vị trí lỗi trong code
- Xác minh fix đã đúng chưa

Một bug report tệ dẫn đến:
- Developer không tái hiện được → mark "Cannot Reproduce"
- Mất thời gian qua lại hỏi thêm thông tin
- Fix sai → Tester phải reopen
- Xung đột và mất tin tưởng giữa Dev và Tester

### 3.2.2 Cấu trúc Bug Report chuẩn

**Tiêu đề (Title/Summary):**
Phải ngắn gọn, cụ thể, mô tả được: *đang làm gì* + *điều gì xảy ra*.

- ✅ Tốt: *"[Checkout] Áp dụng mã giảm giá SUMMER20 không giảm giá khi giỏ hàng có sản phẩm sale"*
- ❌ Không tốt: *"Lỗi mã giảm giá"* — quá chung chung, không tái hiện được

**Môi trường (Environment):**
Thông tin môi trường cụ thể là yếu tố quan trọng để Developer tái hiện:
- Môi trường: Staging / Production / Development
- OS: Windows 11 / macOS 14.2 / Ubuntu 22.04
- Trình duyệt & phiên bản: Chrome 120.0.6099.109
- Thiết bị: MacBook Pro M2 / iPhone 15 / Samsung Galaxy S23
- Phiên bản ứng dụng: v2.3.1 (build #456)
- Tài khoản test: user@example.com (role: Customer)

**Độ ưu tiên & Mức nghiêm trọng:**
- Severity: Critical / High / Medium / Low
- Priority: Urgent / High / Medium / Low

**Mô tả (Description):**
Mô tả ngắn gọn vấn đề trong 1-2 câu, cung cấp ngữ cảnh.

**Steps to Reproduce (Các bước tái hiện):**
Đây là phần quan trọng nhất. Mỗi bước phải cụ thể, đủ để người khác làm theo mà không cần đoán:

```
Precondition: Đăng nhập bằng tài khoản customer@test.com. 
              Tài khoản này KHÔNG phải VIP member.
              Giỏ hàng đang trống.

1. Vào trang sản phẩm: /products
2. Thêm sản phẩm "Áo thun Basic" (ID: P001, giá: 200,000đ, đang sale 50%) vào giỏ hàng
3. Vào trang giỏ hàng: /cart
4. Nhập mã giảm giá "SUMMER20" vào ô "Mã giảm giá"
5. Click nút "Áp dụng"
6. Quan sát kết quả hiển thị
```

**Expected Result (Kết quả mong đợi):**
Mô tả chính xác những gì phải xảy ra. Tham chiếu đến yêu cầu nếu có:

```
Theo tài liệu yêu cầu UC-023:
- Mã SUMMER20 giảm 20% trên tổng giá trị đơn hàng
- Tổng trước giảm: 200,000đ
- Giảm 20%: -40,000đ  
- Tổng sau giảm: 160,000đ
- Hiển thị dòng "Giảm giá SUMMER20: -40,000đ" trong bảng tóm tắt
```

**Actual Result (Kết quả thực tế):**
Mô tả chính xác những gì thực sự xảy ra:

```
Sau khi click "Áp dụng":
- Hiển thị thông báo màu xanh: "Mã giảm giá đã được áp dụng thành công!"
- Tuy nhiên tổng tiền vẫn hiển thị: 200,000đ (không thay đổi)
- Không có dòng "Giảm giá SUMMER20" trong bảng tóm tắt
- Khi proceed to checkout, tổng thanh toán vẫn là 200,000đ
```

**Tần suất xuất hiện (Frequency):**
- Always (100%) — tái hiện mỗi lần
- Often (>50%) — tái hiện thường xuyên
- Sometimes (<50%) — thỉnh thoảng
- Rarely — hiếm khi

**Evidence (Bằng chứng):**
- Screenshot: chụp màn hình với annotation (mũi tên, khoanh tròn) chỉ rõ vấn đề
- Video: quay màn hình toàn bộ quá trình tái hiện (dùng Loom, OBS, hoặc native screen recording)
- Log: browser console log, network request/response, server log nếu có

---

### 3.2.3 Ví dụ Bug Report hoàn chỉnh

```
TITLE: [Cart] Mã giảm giá SUMMER20 không giảm giá khi giỏ hàng 
       chứa sản phẩm đang sale

ENVIRONMENT:
- Môi trường: Staging (https://staging.shop.com)
- OS: macOS Sonoma 14.2
- Browser: Chrome 120.0.6099.109
- Tài khoản: customer@test.com (Regular customer, non-VIP)
- Phiên bản: v2.3.1 (build #456)

SEVERITY: High
PRIORITY: High

DESCRIPTION:
Mã giảm giá SUMMER20 (giảm 20%) hiển thị thông báo áp dụng thành công 
nhưng thực tế không giảm giá trên tổng đơn hàng khi giỏ hàng chứa ít 
nhất một sản phẩm đang trong chương trình sale.

PRECONDITION:
- Đăng nhập bằng customer@test.com
- Mã SUMMER20 còn hiệu lực (hết hạn: 31/12/2025)
- Giỏ hàng đang trống

STEPS TO REPRODUCE:
1. Vào /products
2. Thêm "Áo thun Basic" (P001, 200,000đ, đang sale 50%) vào giỏ
3. Vào /cart
4. Nhập "SUMMER20" vào ô mã giảm giá
5. Click "Áp dụng"
6. Quan sát bảng tóm tắt đơn hàng

EXPECTED RESULT:
- Thông báo áp dụng thành công
- Hiển thị dòng "Giảm giá SUMMER20: -40,000đ" (20% của 200,000đ)
- Tổng sau giảm: 160,000đ

ACTUAL RESULT:
- Thông báo "Áp dụng thành công" xuất hiện
- Không có dòng giảm giá trong bảng tóm tắt
- Tổng vẫn là 200,000đ
- Console log: không có error

FREQUENCY: Always (100%) - tái hiện mỗi lần

WORKAROUND: Không có workaround. Người dùng không thể áp dụng mã 
            giảm giá cho đơn hàng có sản phẩm sale.

ADDITIONAL NOTES:
- Đã test với sản phẩm KHÔNG sale: SUMMER20 hoạt động đúng
- Đã test với mã giảm giá khác (WINTER10) trên sản phẩm sale: cùng lỗi
- Có thể liên quan đến logic kết hợp hai loại giảm giá

ATTACHMENTS:
- screenshot_cart_before.png
- screenshot_cart_after_coupon.png  
- screen_recording.mp4
- console_log.txt
```

---

### 3.2.4 Duplicate Bug

**Duplicate Bug** xảy ra khi một bug được báo cáo nhiều lần bởi các Tester khác nhau, hoặc Tester báo cáo một bug đã tồn tại mà không biết.

**Cách phòng tránh:**
- Trước khi tạo bug mới, tìm kiếm trong hệ thống với từ khóa liên quan
- Tìm theo module, chức năng, hoặc thông báo lỗi
- Hỏi team nếu không chắc

**Cách xử lý khi phát hiện duplicate:**
- Developer hoặc Manager mark trạng thái "Duplicate"
- Ghi rõ "Duplicate of #BUG-123" (ID bug gốc)
- Đóng bug duplicate, giữ nguyên bug gốc
- Tất cả thông tin bổ sung từ bug duplicate có thể được merge vào bug gốc

---

### 3.2.5 Cannot Reproduce

**Cannot Reproduce** là trạng thái Developer gán khi không thể tái hiện bug theo các bước được mô tả.

**Nguyên nhân phổ biến:**
- Steps to Reproduce không đủ chi tiết
- Môi trường khác nhau (Tester dùng staging, Dev dùng local)
- Dữ liệu test khác nhau
- Bug phụ thuộc vào timing hoặc điều kiện race condition
- Bug đã được fix vô tình trong commit khác

**Cách xử lý khi bị mark "Cannot Reproduce":**
- Cung cấp thêm thông tin: video recording, log đầy đủ, môi trường chi tiết
- Tái hiện cùng Dev — screen share và làm lại từng bước
- Kiểm tra xem có phụ thuộc vào dữ liệu đặc biệt không
- Nếu bug xảy ra không thường xuyên, ghi rõ điều kiện bạn thấy lần gần nhất

---

### 3.2.6 Won't Fix và Deferred Bug

**Won't Fix:**
Bug được xác nhận là tồn tại nhưng team quyết định không sửa. Lý do phổ biến:
- Chi phí sửa cao, rủi ro regression cao, lợi ích thấp
- Chức năng liên quan sắp bị loại bỏ trong roadmap
- Số người dùng bị ảnh hưởng quá ít
- Có workaround chấp nhận được

**Deferred (Hoãn lại):**
Bug hợp lệ, sẽ được sửa nhưng không phải trong sprint/version hiện tại. Thường do ưu tiên kinh doanh hoặc giới hạn thời gian.

> **Tư duy chuyên nghiệp:** Tester không nên "chiến đấu" với mọi quyết định Won't Fix. Đây là quyết định kinh doanh có lý do. Tuy nhiên, Tester có trách nhiệm đảm bảo rủi ro được hiểu rõ và ghi nhận. Nếu bug Won't Fix gây hậu quả sau này, có tài liệu chứng minh Tester đã báo cáo.

---

## 3.3 Test Management

### 3.3.1 Test Plan — Kế hoạch kiểm thử

**Định nghĩa:** Test Plan là tài liệu chính thức mô tả toàn bộ chiến lược, phạm vi, nguồn lực, lịch trình, và cách tiếp cận kiểm thử cho một dự án hoặc một giai đoạn phát triển.

**Bản chất:** Test Plan trả lời 5 câu hỏi cốt lõi:
- **What:** Cần kiểm thử cái gì?
- **Why:** Tại sao kiểm thử những thứ đó?
- **How:** Kiểm thử như thế nào?
- **Who:** Ai thực hiện?
- **When:** Khi nào?

**Cấu trúc Test Plan theo IEEE 829:**

**1. Test Plan ID và Version:**
Định danh tài liệu, lịch sử phiên bản, ngày tạo, người tạo.

**2. Introduction (Giới thiệu):**
Mục đích của Test Plan, phạm vi tài liệu, tài liệu tham chiếu (SRS, Design Document...).

**3. Test Items (Đối tượng kiểm thử):**
Liệt kê cụ thể các tính năng, module, API, màn hình sẽ được kiểm thử. Ví dụ: Module Authentication (Login, Register, Forgot Password), Module Checkout (Cart, Payment, Order Confirmation).

**4. Features to be Tested (Phạm vi kiểm thử):**
Chi tiết những gì sẽ được kiểm thử trong phạm vi của Test Plan này.

**5. Features NOT to be Tested (Phạm vi ngoài kiểm thử):**
Quan trọng không kém — liệt kê rõ những gì sẽ KHÔNG được kiểm thử và lý do (không thuộc phạm vi sprint, sẽ kiểm thử trong giai đoạn khác, đã có automated test bao phủ...).

**6. Test Approach (Cách tiếp cận kiểm thử):**
Kỹ thuật kiểm thử sử dụng, mức độ kiểm thử (unit, integration, system, UAT), công cụ, môi trường.

**7. Entry Criteria (Điều kiện bắt đầu):**
Những điều kiện phải được thỏa mãn trước khi bắt đầu kiểm thử. Ví dụ:
- Build đã được deploy lên staging thành công
- Smoke test pass
- Tài liệu yêu cầu đã được approve
- Môi trường kiểm thử đã sẵn sàng
- Test data đã được chuẩn bị

**8. Exit Criteria (Điều kiện kết thúc):**
Những điều kiện phải được thỏa mãn để tuyên bố kiểm thử hoàn thành. Ví dụ:
- 100% test case đã được thực thi
- Không còn bug Critical hoặc High nào đang Open
- Pass rate ≥ 95%
- Regression test pass
- UAT được approve bởi Product Owner

**9. Test Schedule (Lịch trình):**
Timeline chi tiết: ai làm gì, từ ngày nào đến ngày nào.

**10. Resources (Nguồn lực):**
Nhân sự (tên, vai trò, trách nhiệm), công cụ, môi trường cần thiết.

**11. Risk Management (Quản lý rủi ro):**
Các rủi ro có thể xảy ra và kế hoạch ứng phó. Ví dụ:
- Rủi ro: Môi trường staging không ổn định → Kế hoạch: Có backup environment
- Rủi ro: Tester nghỉ phép đột xuất → Kế hoạch: Backup Tester đã được brief

---

### 3.3.2 Test Strategy vs Test Plan

| Tiêu chí | Test Strategy | Test Plan |
|---|---|---|
| Phạm vi | Toàn tổ chức / toàn dự án | Một release / một sprint cụ thể |
| Mức độ | Cấp cao, tổng quát | Chi tiết, cụ thể |
| Thay đổi | Ít thay đổi | Cập nhật theo từng release |
| Nội dung | Triết lý, nguyên tắc, cách tiếp cận | Timeline, nguồn lực, scope cụ thể |
| Tác giả | QA Manager / Test Lead | Test Lead / Senior Tester |

**Test Strategy** trả lời: *"Chúng ta tiếp cận kiểm thử như thế nào về mặt triết lý?"*

**Test Plan** trả lời: *"Trong release này, chúng ta sẽ làm gì cụ thể?"*

---

### 3.3.3 Test Scope — Phạm vi kiểm thử

Test Scope xác định rõ ràng những gì nằm trong phạm vi (in-scope) và ngoài phạm vi (out-of-scope) của đợt kiểm thử.

**Tại sao cần xác định Test Scope rõ ràng:**
- Tránh hiểu lầm giữa các bên về những gì Tester chịu trách nhiệm
- Quản lý kỳ vọng của stakeholder
- Phân bổ nguồn lực hiệu quả

**Ví dụ Test Scope cho Sprint 5 của dự án E-commerce:**

*In-scope:*
- Tính năng Wishlist (thêm, xóa, xem, chia sẻ)
- Tính năng Product Review (viết, edit, xóa review)
- Regression test cho module Cart (bị thay đổi trong sprint này)
- API kiểm thử cho 3 endpoint mới

*Out-of-scope:*
- Module Admin (sprint khác phụ trách)
- Performance testing (sẽ thực hiện riêng trước release)
- Mobile app (chưa hoàn thiện tính năng tương đương)
- Tích hợp payment mới (đang development, chưa sẵn sàng)

---

### 3.3.4 Entry và Exit Criteria

**Entry Criteria** (Tiêu chí bắt đầu) là điều kiện phải thỏa mãn trước khi bắt đầu giai đoạn kiểm thử. Nếu Entry Criteria không được thỏa mãn, kiểm thử không nên bắt đầu — vì sẽ lãng phí thời gian.

**Ví dụ Entry Criteria cho System Testing:**
```
□ Build ổn định đã được deploy lên staging
□ Smoke test pass (100%)
□ Tài liệu yêu cầu (SRS) đã được sign-off
□ Test case đã được review và approve
□ Test data đã được tạo
□ Môi trường kiểm thử đã được cấu hình xong
□ Các công cụ kiểm thử đã sẵn sàng (Jira, TestRail, Postman)
```

**Exit Criteria** (Tiêu chí kết thúc) là điều kiện phải thỏa mãn để tuyên bố kiểm thử hoàn thành và sẵn sàng chuyển sang giai đoạn tiếp theo.

**Ví dụ Exit Criteria:**
```
□ 100% test case đã được thực thi (kể cả Blocked phải có lý do)
□ Pass rate ≥ 95%
□ Không có bug Critical hoặc High nào ở trạng thái Open
□ Tất cả bug Medium đã được triage (Accept/Defer/Won't Fix)
□ Regression test suite pass 100%
□ UAT được sign-off bởi Product Owner
□ Test Summary Report đã được gửi
```

---

## 3.4 Test Metrics — Chỉ số đo lường chất lượng

Test Metrics là các số liệu định lượng giúp đánh giá tiến độ, chất lượng kiểm thử, và chất lượng phần mềm. Metrics tốt giúp ra quyết định dựa trên dữ liệu, không phải cảm tính.

### 3.4.1 Test Coverage — Độ phủ kiểm thử

**Định nghĩa:** Test Coverage đo lường tỷ lệ phần trăm yêu cầu hoặc code đã được kiểm thử so với tổng thể.

**Requirement Coverage:**
```
Requirement Coverage = (Số requirement có ít nhất 1 test case) / (Tổng số requirement) × 100%
```

Ví dụ: Có 50 acceptance criteria, đã viết test case cho 45 cái → Coverage = 90%

**Code Coverage (Line/Branch Coverage):**
```
Line Coverage = (Số dòng code được thực thi bởi test) / (Tổng số dòng code) × 100%
```

> **Lưu ý quan trọng:** Coverage cao không đồng nghĩa với chất lượng cao. 100% code coverage nhưng test case viết tệ (chỉ kiểm tra happy path, không có assertion thực sự) vẫn có thể bỏ sót nhiều lỗi. Coverage là chỉ số cần thiết nhưng không đủ.

---

### 3.4.2 Defect Density — Mật độ lỗi

**Định nghĩa:** Defect Density đo số lượng lỗi trên một đơn vị kích thước phần mềm, thường là 1,000 dòng code (KLOC) hoặc theo function point.

```
Defect Density = Số defect phát hiện / Kích thước phần mềm (KLOC)
```

Ví dụ: Module Checkout có 5,000 dòng code, phát hiện 15 bug → Defect Density = 3 bugs/KLOC

**Ứng dụng:**
- So sánh chất lượng giữa các module: module nào có density cao cần được kiểm thử kỹ hơn
- So sánh theo thời gian: density giảm qua các sprint là dấu hiệu chất lượng cải thiện
- Dự đoán số bug còn lại ở các phần chưa kiểm thử

---

### 3.4.3 Pass Rate — Tỷ lệ pass

```
Pass Rate = (Số test case Pass) / (Tổng số test case đã thực thi) × 100%
```

Ví dụ: Thực thi 200 test case, 185 Pass, 15 Fail → Pass Rate = 92.5%

**Cách diễn giải:**
- Pass rate > 95%: Chất lượng tốt, có thể cân nhắc release
- Pass rate 80-95%: Cần xem xét các bug đang fail trước khi quyết định
- Pass rate < 80%: Chất lượng chưa đạt, cần thêm thời gian fix

---

### 3.4.4 Defect Leakage — Lỗi lọt qua kiểm thử

**Định nghĩa:** Defect Leakage là số lượng bug mà team QA bỏ sót (không phát hiện trong quá trình kiểm thử) và chỉ được phát hiện sau khi phần mềm đến tay người dùng.

```
Defect Leakage = (Bug phát hiện trong/sau UAT hoặc Production) / (Tổng bug phát hiện) × 100%
```

**Đây là chỉ số quan trọng nhất đánh giá hiệu quả của team QA.**

Ví dụ: Sprint 5 tổng cộng có 40 bug được phát hiện — 35 bug trong kiểm thử, 5 bug phát hiện sau khi release → Defect Leakage = 5/40 = 12.5%

**Ngưỡng chấp nhận:** Defect Leakage < 5% là tốt. Nếu > 10%, cần phân tích nguyên nhân: bỏ sót test case nào? Loại bug nào hay bị bỏ sót?

---

### 3.4.5 Test Summary Report

**Test Summary Report** là tài liệu tổng hợp kết quả kiểm thử của một sprint/release, gửi cho stakeholders để họ ra quyết định có phát hành hay không.

**Nội dung Test Summary Report:**

```
TEST SUMMARY REPORT — Sprint 5 / v2.3.0

1. THÔNG TIN CHUNG
   - Sprint: 5 | Release: v2.3.0
   - Giai đoạn kiểm thử: 01/11 - 15/11/2025
   - Tester: Nguyễn Văn A, Trần Thị B

2. PHẠM VI KIỂM THỬ
   - Module: Wishlist, Product Review, Cart (regression)
   - Môi trường: Staging v2.3.0-rc1

3. KẾT QUẢ THỰC TỔNG QUAN
   - Tổng test case: 245
   - Executed: 245 (100%)
   - Pass: 232 (94.7%)
   - Fail: 10 (4.1%)
   - Blocked: 3 (1.2%)

4. TỔNG KẾT BUG
   - Tổng bug phát hiện: 28
   - Critical: 0 | High: 3 | Medium: 12 | Low: 13
   - Đã closed: 22 | Đang open: 6 (High: 1, Medium: 3, Low: 2)

5. RỦI RO & VẤN ĐỀ
   - Còn 1 bug High (#BUG-234: Wishlist không sync đúng khi đăng nhập 
     từ nhiều thiết bị) — Developer đang investigate
   - 3 test case bị Blocked do môi trường thanh toán sandbox không ổn định

6. ĐÁNH GIÁ & KHUYẾN NGHỊ
   Chất lượng sprint này: ĐẠT (Pass rate 94.7%, không có Critical bug)
   Khuyến nghị: Release với điều kiện fix xong bug High #BUG-234
   
7. KÝ XÁC NHẬN
   QA Lead: _____________ | Date: _____________
   PM: _____________      | Date: _____________
```

---

## 3.5 Công Cụ Quản Lý Kiểm Thử

### 3.5.1 Jira — Quản lý dự án và theo dõi lỗi

**Jira** (Atlassian) là công cụ quản lý dự án phổ biến nhất trong ngành phần mềm. Đối với Tester, Jira là nơi tạo và theo dõi bug report, xem User Story, và theo dõi tiến độ sprint.

**Các khái niệm cần nắm trong Jira:**

**Issue Types:**
- **Bug:** Defect cần sửa
- **Story:** User Story mô tả tính năng
- **Task:** Công việc kỹ thuật không phải feature
- **Epic:** Nhóm Story liên quan theo theme lớn
- **Sub-task:** Task con của Story hoặc Bug

**Jira Fields quan trọng khi tạo Bug:**
- **Summary:** Tiêu đề bug (viết theo format đã học)
- **Description:** Steps to reproduce, Expected, Actual, Environment
- **Assignee:** Developer được giao
- **Reporter:** Người tạo bug (tự động)
- **Priority:** Urgent/High/Medium/Low
- **Labels/Components:** Phân loại theo module (Authentication, Checkout...)
- **Fix Version:** Version mà bug sẽ được sửa
- **Linked Issues:** Liên kết với Story liên quan, bug duplicate

**Workflow trong Jira:**
Jira cho phép tùy chỉnh workflow. Workflow phổ biến:
```
Open → In Progress → Code Review → Testing → Done
                                    ↓ (Fail)
                               Reopened → In Progress
```

**Jira Query Language (JQL) — Tìm kiếm nâng cao:**
```
// Tìm tất cả bug High đang Open trong project hiện tại
project = "SHOP" AND issuetype = Bug AND priority = High AND status = Open

// Bug được tạo trong sprint hiện tại
project = "SHOP" AND issuetype = Bug AND sprint in openSprints()

// Bug của tôi chưa được đóng
assignee = currentUser() AND issuetype = Bug AND status != Closed

// Bug chưa tái hiện được
project = "SHOP" AND issuetype = Bug AND status = "Cannot Reproduce"
```

---

### 3.5.2 TestRail — Quản lý Test Case

**TestRail** là công cụ chuyên biệt để quản lý test case, tổ chức test run, và tạo báo cáo kiểm thử.

**Cấu trúc trong TestRail:**
```
Project
└── Test Suite (nhóm test case theo module)
    ├── Section (nhóm nhỏ hơn)
    │   ├── Test Case 1
    │   ├── Test Case 2
    │   └── Test Case 3
    └── Section 2
        └── ...
```

**Test Run trong TestRail:**
Khi bắt đầu thực thi, tạo **Test Run** — là một phiên kiểm thử cụ thể với danh sách test case được chọn. Sau mỗi test case, Tester cập nhật kết quả: Passed / Failed / Blocked / Retest.

**Tích hợp TestRail với Jira:**
Khi test case fail, Tester có thể tạo bug trong Jira trực tiếp từ TestRail, kèm link tham chiếu hai chiều.

---

### 3.5.3 Git / GitHub — Kiến thức cơ bản cho Tester

Tester cần hiểu Git cơ bản để:
- Checkout branch mới để kiểm thử tính năng mới
- Hiểu commit history để biết có gì thay đổi
- Phối hợp với Developer khi báo cáo bug liên quan đến commit

**Các lệnh Git cơ bản Tester cần biết:**
```bash
# Clone repository về máy
git clone https://github.com/company/project.git

# Xem branch hiện tại
git branch

# Chuyển sang branch cần test
git checkout feature/user-wishlist

# Cập nhật code mới nhất
git pull origin feature/user-wishlist

# Xem lịch sử commit gần đây
git log --oneline -10

# Xem thay đổi trong commit cụ thể
git show <commit-hash>
```

---

### 3.5.4 Developer Tools (F12) — Công cụ debug trên trình duyệt

Developer Tools là "vũ khí bí mật" của Tester — cho phép kiểm tra chi tiết kỹ thuật mà không cần nhờ Developer.

**Tab Elements:**
- Inspect HTML structure của trang
- Kiểm tra CSS, xem element có hiển thị đúng không
- Verify text content, attribute của element

**Tab Console:**
- Xem JavaScript errors
- Log messages từ ứng dụng
- Khi phát hiện lỗi JavaScript, copy toàn bộ error message vào bug report

**Tab Network:**
- Xem tất cả HTTP requests và responses
- Kiểm tra API call: URL, method, request body, response body, status code
- Xem headers (Authorization, Content-Type)
- Identify slow requests (performance issue)
- Copy request dưới dạng cURL để Developer tái hiện

**Cách dùng Network tab khi tìm bug:**
1. Mở DevTools (F12) → Tab Network
2. Tick "Preserve log" để không mất log khi navigate
3. Thực hiện thao tác cần kiểm tra
4. Tìm request liên quan (lọc bằng ô tìm kiếm)
5. Click vào request → xem Headers, Payload, Response
6. Chụp màn hình hoặc copy request/response đính kèm vào bug report

**Ví dụ thực tế:** Tester nhấn "Đặt hàng" và thấy lỗi. Thay vì chỉ chụp màn hình lỗi, mở Network tab, tìm request POST `/api/orders`, copy response body:
```json
{
  "error": "INVALID_COUPON_COMBINATION",
  "message": "Cannot combine sale discount with coupon code",
  "details": {
    "productId": "P001",
    "saleDiscount": 0.5,
    "couponCode": "SUMMER20"
  }
}
```
Thêm thông tin này vào bug report → Developer biết ngay vấn đề nằm ở đâu.
