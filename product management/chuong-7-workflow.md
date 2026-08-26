# CHƯƠNG 7: WORKFLOW — QUY TRÌNH CHUYỂN TRẠNG THÁI CÔNG VIỆC

---

## 7.1. Workflow là gì?

**Workflow** (Quy trình làm việc) trong Jira là tập hợp các **trạng thái (Status)** mà một Issue có thể tồn tại, cùng các **chuyển trạng thái (Transition)** được phép giữa các trạng thái đó. Workflow định nghĩa "con đường" mà một Issue đi qua từ khi được tạo ra cho đến khi hoàn thành.

Nếu Board là bức tranh hiển thị **vị trí hiện tại** của Issue, thì Workflow là bản đồ xác định **những con đường nào được phép đi** giữa các vị trí đó.

Ví dụ đơn giản nhất: một Issue không thể nhảy thẳng từ "To Do" sang "Done" nếu Workflow không cho phép chuyển trạng thái trực tiếp đó — nó phải đi qua "In Progress" và "Testing" trước.

Workflow là cầu nối giữa **quy trình thực tế của nhóm** và **giao diện Board trong Jira** — cấu hình Workflow tốt sẽ khiến Board phản ánh đúng cách nhóm thực sự làm việc.

---

## 7.2. Status (Trạng thái) và Transition (Chuyển trạng thái)

### Status (Trạng thái)

**Status** là một điểm dừng cụ thể trong vòng đời của Issue. Mỗi Status đại diện cho một giai đoạn trong quy trình làm việc và được hiển thị dưới dạng một cột trên Board.

Trong Jira, mỗi Status thuộc về một trong ba **loại trạng thái (Status Category)**:

| Loại trạng thái | Màu sắc | Ý nghĩa |
|---|---|---|
| **To Do** | Xám | Công việc chưa được bắt đầu |
| **In Progress** | Xanh dương | Công việc đang được thực hiện |
| **Done** | Xanh lá | Công việc đã hoàn thành |

Phân loại này quan trọng vì Jira sử dụng nó để tính toán các báo cáo như Burndown Chart và Sprint Report — chỉ khi Issue thuộc loại "Done" mới được tính là hoàn thành.

### Transition (Chuyển trạng thái)

**Transition** là hành động chuyển một Issue từ Status này sang Status khác. Mỗi Transition có:

- **Tên:** Mô tả hành động (ví dụ: "Start Progress", "Send to Review", "Mark as Done")
- **Trạng thái nguồn:** Issue đang ở đâu trước khi chuyển
- **Trạng thái đích:** Issue sẽ chuyển đến đâu

Trong giao diện Board, khi kéo thả một Issue sang cột khác, người dùng đang thực hiện một Transition. Trong trang chi tiết Issue, nút bấm trạng thái cũng kích hoạt Transition tương ứng.

---

## 7.3. Workflow cơ bản

Jira cung cấp sẵn một Workflow mặc định đơn giản gồm ba trạng thái:

```
To Do  →  In Progress  →  Done
```

Workflow này phù hợp với các nhóm mới bắt đầu hoặc dự án đơn giản. Tuy nhiên, trong thực tế phát triển phần mềm, quy trình thường phức tạp hơn và cần nhiều bước trung gian hơn.

---

## 7.4. Workflow của Team phát triển phần mềm

Một Workflow phổ biến trong nhóm phát triển phần mềm có thể có dạng:

```
To Do
  ↓
In Progress          (Developer đang viết code)
  ↓
Code Review          (Đồng nghiệp kiểm tra mã nguồn)
  ↓  ↑ (Nếu cần sửa, quay lại In Progress)
Testing / QA         (QA kiểm thử tính năng)
  ↓  ↑ (Nếu phát hiện lỗi, quay lại In Progress)
Done                 (Hoàn thành, sẵn sàng release)
```

### Giải thích từng bước:

**To Do:** Issue đã được lên kế hoạch trong Sprint nhưng chưa ai bắt tay vào làm.

**In Progress:** Developer nhận Issue và bắt đầu viết code. Ở bước này, Issue được gán cho một Developer cụ thể.

**Code Review:** Developer hoàn thành code và tạo Pull Request. Issue chuyển sang trạng thái chờ đồng nghiệp kiểm tra. Nếu reviewer yêu cầu chỉnh sửa, Issue có thể quay lại "In Progress".

**Testing / QA:** Code đã được review và merge. QA tiến hành kiểm thử toàn diện tính năng. Nếu phát hiện bug, Issue quay lại "In Progress" để sửa.

**Done:** Tính năng đã pass toàn bộ kiểm thử và đáp ứng Definition of Done. Issue được đóng lại.

---

## 7.5. Điều kiện chuyển trạng thái

Trong Jira, mỗi Transition có thể được cấu hình với các **điều kiện (Conditions)** — quy tắc xác định ai được phép thực hiện Transition đó và trong điều kiện nào.

### Các loại điều kiện phổ biến:

**a) Điều kiện về người dùng (User Condition)**

Ví dụ: Chỉ người được giao Issue (Assignee) mới được phép chuyển sang "In Progress". Chỉ QA mới được chuyển sang "Done".

**b) Điều kiện về trường thông tin (Field Condition)**

Ví dụ: Issue phải có Assignee trước khi được chuyển sang "In Progress". Issue phải có Story Point được nhập trước khi bắt đầu.

**c) Validators (Kiểm tra hợp lệ)**

Validators chạy khi Transition được thực hiện và có thể ngăn chặn nếu điều kiện không thỏa mãn. Ví dụ: Yêu cầu người dùng nhập comment lý do khi chuyển Issue về "To Do" từ "In Progress".

**d) Post Functions (Hành động tự động sau Transition)**

Sau khi Transition hoàn thành, Jira có thể tự động thực hiện một số hành động như: gán Assignee tự động, gửi thông báo email, cập nhật trường thông tin...

> **Lưu ý:** Cấu hình Conditions, Validators và Post Functions là tính năng nâng cao, thường do **Project Administrator** hoặc **Jira Administrator** thiết lập. Thành viên thông thường chỉ cần hiểu và tuân theo Workflow đã được cấu hình.

> 📌 **Khác biệt với Team-managed:** Tính năng **Conditions**, **Validators** và **Post Functions** **không có** trong Team-managed. Đây là tính năng dành riêng cho Company-managed. Trong Team-managed, không thể đặt điều kiện ràng buộc khi chuyển trạng thái — bất kỳ thành viên nào cũng có thể kéo Issue sang bất kỳ cột nào.

---

## 7.6. Quy trình xử lý Bug

Bug có một Workflow riêng biệt, phản ánh chu trình đặc thù từ khi phát hiện lỗi đến khi xác nhận lỗi đã được sửa:

```
Open (Mới phát hiện)
  ↓
In Progress (Developer đang sửa)
  ↓
Resolved / Fixed (Đã sửa xong, chờ kiểm tra)
  ↓
Testing (QA kiểm tra lại)
  ↓
Closed (Xác nhận đã sửa xong)
        hoặc
Reopened (Lỗi chưa được sửa, mở lại)
  ↓
In Progress (Tiếp tục sửa...)
```

### Trạng thái đặc trưng của Bug Workflow:

**Open:** Bug mới được ghi nhận, chưa được phân công xử lý.

**In Progress:** Developer đã nhận Bug và đang tìm nguyên nhân, viết fix.

**Resolved / Fixed:** Developer tuyên bố đã sửa xong và cần QA xác nhận.

**Testing:** QA kiểm tra lại xem Bug có thực sự được sửa hay chưa, trên đúng môi trường và điều kiện ban đầu.

**Closed:** QA xác nhận Bug đã được sửa hoàn toàn. Issue được đóng lại.

**Reopened:** QA phát hiện Bug chưa được sửa hoặc đã xuất hiện trở lại. Issue được mở lại và quay về "In Progress".

---

## 7.7. Workflow và Scrum Board

Có một mối liên hệ trực tiếp và quan trọng giữa Workflow và cách Board hiển thị:

**Mỗi cột trên Board tương ứng với một hoặc nhiều Status trong Workflow.** Khi Project Administrator cấu hình Board, họ ánh xạ (map) các Status vào các cột. Ví dụ:

| Cột trên Board | Status trong Workflow |
|---|---|
| To Do | To Do |
| In Progress | In Progress, Code Review |
| Testing | QA Testing |
| Done | Done, Closed |

Điều này có nghĩa là: hai Status khác nhau trong Workflow (ví dụ "In Progress" và "Code Review") có thể hiển thị trong cùng một cột trên Board nếu được ánh xạ như vậy.

> 📌 **Khác biệt quan trọng với Team-managed:** Trong Team-managed, **Workflow và cột trên Board là một** — không có sự tách biệt giữa hai khái niệm này. Mỗi cột trên Board chính là một Status, và bạn **chỉnh Workflow bằng cách thêm/xóa/đổi tên cột trực tiếp trên Board**:
>
> - Thêm cột mới = thêm Status mới vào Workflow
> - Xóa cột = xóa Status khỏi Workflow
> - Đổi tên cột = đổi tên Status
>
> Để chỉnh Board trong Team-managed: vào **Board** → nhấn **"..."** ở góc trên phải → chọn **"Manage board"** hoặc kéo để tạo cột mới trực tiếp. Không cần vào bất kỳ màn hình cấu hình Workflow riêng biệt nào.

### Tại sao cần hiểu mối liên hệ này?

Khi nhìn vào Board, người dùng thấy cột — nhưng thực tế bên trong, Issue đang ở một Status cụ thể của Workflow. Hiểu được điều này giúp tránh nhầm lẫn khi một Issue được kéo vào cột nhưng trạng thái thực tế không thay đổi theo kỳ vọng — nguyên nhân thường là do Workflow không có Transition tương ứng.

---

## 7.8. Nguyên tắc thiết kế Workflow hiệu quả

Mặc dù người dùng thông thường không trực tiếp thiết kế Workflow, việc hiểu các nguyên tắc thiết kế giúp sử dụng và phản hồi về Workflow hiện tại hiệu quả hơn.

**Chỉ tạo Status khi thực sự cần thiết:** Workflow phức tạp với quá nhiều trạng thái gây khó nhớ và khó vận hành. Mỗi Status phải có một người chịu trách nhiệm và hành động rõ ràng.

**Tên Status phải thể hiện hành động hoặc trạng thái rõ ràng:** Tránh tên mơ hồ như "Pending" hay "Hold" — thay bằng "Waiting for Design Review" hay "On Hold - Blocked by API".

**Hạn chế vòng lặp ngược:** Việc cho phép Issue quay lại quá nhiều bước có thể gây hỗn loạn. Chỉ cho phép quay lại các bước thực sự cần thiết.

**Đồng bộ Workflow với quy trình thực tế:** Workflow trong Jira nên phản ánh đúng cách nhóm thực sự làm việc — không phải cách lý tưởng trên giấy tờ. Nếu nhóm không thực sự có bước Code Review, không cần tạo Status đó.

---

## Tóm tắt chương

Workflow là "bộ xương" quy định cách Issue di chuyển trong Jira. Một Workflow được thiết kế đúng sẽ phản ánh chính xác quy trình làm việc thực tế của nhóm, ngăn chặn các bước bị bỏ qua và tạo nền tảng cho Board hoạt động hiệu quả.

| Khái niệm | Ý nghĩa ngắn gọn |
|---|---|
| Workflow | Tập hợp Status và Transition của một loại Issue |
| Status | Trạng thái hiện tại của Issue trong quy trình |
| Transition | Hành động chuyển Issue từ Status này sang Status khác |
| Status Category | Phân loại To Do / In Progress / Done |
| Condition | Điều kiện để một Transition được phép thực hiện |
| Validator | Kiểm tra hợp lệ trước khi Transition hoàn tất |
| Post Function | Hành động tự động xảy ra sau khi Transition |
| Reopened | Trạng thái mở lại Bug sau khi xác nhận chưa được sửa |

---

*Chương tiếp theo: **Chương 8 — Estimation & Tracking: Ước lượng và Theo dõi Tiến độ***
