# CHƯƠNG 6: JIRA BOARD — THEO DÕI CÔNG VIỆC TRỰC QUAN

---

## 6.1. Board là gì?

**Board** (Bảng công việc) là giao diện trực quan trung tâm trong Jira, nơi toàn bộ công việc của nhóm được hiển thị dưới dạng các thẻ (card) Issue, được sắp xếp vào các cột tương ứng với từng trạng thái trong quy trình làm việc.

Board cung cấp cái nhìn tổng thể, tức thì về tình trạng hiện tại của dự án: ai đang làm gì, công việc nào đang tiến triển, công việc nào đang bị tắc nghẽn và bao nhiêu việc đã hoàn thành. Thay vì đọc báo cáo dài dòng hay hỏi từng người, mọi thành viên chỉ cần nhìn vào Board là nắm được bức tranh chung.

Triết lý đằng sau Board bắt nguồn từ **Kanban** — một phương pháp quản lý công việc của Toyota (Nhật Bản) được áp dụng rộng rãi trong phát triển phần mềm, dựa trên nguyên tắc trực quan hóa luồng công việc để phát hiện sớm điểm tắc nghẽn.

---

## 6.2. Scrum Board

**Scrum Board** được sử dụng trong các Scrum Project. Board này chỉ hiển thị các Issue thuộc **Sprint đang Active** — không hiển thị toàn bộ Backlog. Điều này giúp nhóm tập trung hoàn toàn vào công việc đã cam kết trong Sprint hiện tại.

### Đặc điểm của Scrum Board:

- Hiển thị Issue theo Sprint đang chạy
- Tích hợp với Burndown Chart và Sprint Report
- Thể hiện Sprint Goal ở đầu trang
- Khi Sprint kết thúc, Board tự động chuyển sang Sprint tiếp theo

Scrum Board phù hợp với các nhóm có lịch làm việc cố định theo chu kỳ, cần kiểm soát chặt chẽ tiến độ từng Sprint.

---

## 6.3. Kanban Board

**Kanban Board** được sử dụng trong Kanban Project. Khác với Scrum Board, Kanban Board hiển thị **tất cả Issue đang hoạt động** mà không chia theo Sprint. Công việc liên tục đổ vào từ Backlog và liên tục được hoàn thành — không có điểm bắt đầu hay kết thúc cố định theo chu kỳ.

### Đặc điểm của Kanban Board:

- Hiển thị luồng công việc liên tục
- Hỗ trợ giới hạn WIP (Work In Progress) cho từng cột
- Không có Sprint, không có Burndown Chart
- Tập trung vào tốc độ xử lý (throughput) và thời gian hoàn thành (cycle time)

### WIP Limit là gì?

**WIP Limit** (Work In Progress Limit) là giới hạn số lượng Issue tối đa được phép tồn tại đồng thời trong một cột (trạng thái). Ví dụ, nếu cột "In Progress" có WIP Limit là 3, thì không ai được bắt đầu công việc thứ 4 khi đã có 3 Issue đang xử lý.

WIP Limit giúp phát hiện tắc nghẽn sớm và khuyến khích nhóm hoàn thành công việc hiện tại trước khi nhận việc mới, thay vì xử lý quá nhiều thứ cùng lúc.

---

## 6.4. Các trạng thái phổ biến trên Board

Mỗi cột trên Board đại diện cho một **trạng thái** (Status) trong quy trình làm việc. Số lượng và tên cột phụ thuộc vào Workflow mà nhóm đang sử dụng. Dưới đây là các trạng thái phổ biến nhất trong nhóm phát triển phần mềm:

### 6.4.1. To Do (Cần làm)

Issue đã được đưa vào Sprint (hoặc Board) nhưng chưa có ai bắt đầu thực hiện. Đây là trạng thái mặc định khi Issue được tạo mới hoặc vừa được chuyển vào Sprint.

### 6.4.2. In Progress (Đang thực hiện)

Issue đang được một thành viên tích cực xử lý. Thông thường, mỗi người chỉ nên có một hoặc hai Issue ở trạng thái này cùng lúc để đảm bảo sự tập trung.

### 6.4.3. Review / Code Review

Issue đã được thực hiện xong về mặt kỹ thuật và đang chờ người khác kiểm tra lại (peer review). Trong phát triển phần mềm, đây thường là bước đồng nghiệp xem xét mã nguồn trước khi hợp nhất (merge).

### 6.4.4. Testing / QA

Issue đang được kiểm thử (testing) bởi đội QA hoặc chính người phát triển. Bước này đảm bảo tính năng hoạt động đúng theo yêu cầu trước khi được chấp thuận là hoàn thành.

### 6.4.5. Done (Hoàn thành)

Issue đã vượt qua tất cả các bước kiểm tra và đáp ứng **Definition of Done** — tiêu chí chung mà nhóm đã thống nhất để xác định một công việc thực sự hoàn tất.

### Definition of Done là gì?

**Definition of Done (DoD)** là danh sách các tiêu chí mà một Issue phải đáp ứng trước khi được chuyển sang trạng thái Done. Ví dụ:

> - Code đã được review và approved bởi ít nhất một người khác
> - Unit test đã được viết và pass 100%
> - Không có lỗi Critical hoặc High còn tồn đọng
> - Tài liệu kỹ thuật đã được cập nhật
> - Tính năng đã được test trên môi trường Staging

DoD giúp tránh tình trạng "Done nhưng chưa thực sự Done" — một vấn đề phổ biến khi nhóm không có tiêu chí rõ ràng.

---

## 6.5. Di chuyển Issue trên Board

Di chuyển Issue giữa các cột trên Board là thao tác thường xuyên nhất trong quá trình làm việc hàng ngày.

### Cách di chuyển:

**Cách 1 — Kéo thả (Drag & Drop):** Giữ chuột vào thẻ Issue và kéo sang cột mong muốn. Đây là cách trực quan và nhanh nhất.

**Cách 2 — Thay đổi trạng thái trong Issue:** Mở Issue → nhấn vào nút trạng thái hiện tại (ví dụ: "To Do") → chọn trạng thái mới từ danh sách. Jira sẽ tự động cập nhật vị trí của Issue trên Board.

### Nguyên tắc di chuyển:

- Chỉ người đang thực hiện Issue mới nên di chuyển Issue của mình
- Khi chuyển Issue sang "Done", đảm bảo tất cả tiêu chí trong Definition of Done đã được đáp ứng
- Nếu một Issue bị tắc nghẽn (blocked), đánh dấu bằng Label hoặc Flag để cả nhóm nhận biết

---

## 6.6. Board Filter

**Board Filter** cho phép lọc các Issue hiển thị trên Board theo nhiều tiêu chí khác nhau, giúp tập trung vào phần công việc cần quan tâm mà không bị phân tâm bởi toàn bộ Issue trên Board.

### Các tiêu chí lọc phổ biến:

- **Assignee:** Chỉ hiển thị Issue của một thành viên cụ thể — hữu ích khi Daily Standup và mỗi người báo cáo công việc của mình
- **Epic:** Lọc theo Epic để xem tiến độ của một tính năng lớn
- **Label / Component:** Lọc theo nhãn hoặc module kỹ thuật
- **Priority:** Tập trung vào các Issue có mức độ ưu tiên cao

Bộ lọc được áp dụng trực tiếp trên Board và không ảnh hưởng đến dữ liệu thực tế — chỉ thay đổi những gì đang hiển thị trên màn hình.

---

## 6.7. Swimlane

**Swimlane** là tính năng phân chia Board theo chiều ngang thành các "làn bơi" (lane), mỗi làn nhóm một tập hợp Issue theo một tiêu chí nhất định.

### Các kiểu Swimlane phổ biến:

**Theo Assignee:** Mỗi làn là một thành viên trong nhóm — giúp thấy rõ khối lượng công việc của từng người và phát hiện sự mất cân bằng.

**Theo Epic:** Mỗi làn là một Epic — giúp theo dõi đồng thời tiến độ của nhiều tính năng lớn.

**Theo Priority:** Phân chia theo mức độ ưu tiên — các Issue Critical hoặc High hiển thị ở làn trên cùng.

**Không dùng Swimlane (No Swimlane):** Toàn bộ Issue được hiển thị chung trên một mặt phẳng — phù hợp với nhóm nhỏ hoặc khi muốn nhìn tổng thể đơn giản nhất.

### Cách bật Swimlane:

Trên Board → nhấn **Board Settings** (hoặc biểu tượng cài đặt) → chọn tab **Swimlanes** → chọn kiểu Swimlane mong muốn.

> 📌 **Khác biệt với Team-managed:** Tính năng **Swimlane** trong Team-managed bị giới hạn hơn. Trong một số phiên bản, bạn có thể bật Swimlane theo **Epic** bằng cách vào **Board → "Group by" → chọn Epic**. Không có tùy chọn cấu hình Swimlane đầy đủ như Company-managed.

---

## 6.8. Quick Filter

**Quick Filter** là bộ lọc nhanh được hiển thị dưới dạng các nút bấm ngay trên Board, cho phép chuyển đổi nhanh giữa các chế độ xem mà không cần vào menu cài đặt.

Các Quick Filter phổ biến thường được cấu hình sẵn:

- **Only My Issues:** Chỉ hiện Issue của tôi
- **Recently Updated:** Chỉ hiện Issue được cập nhật gần đây
- **Unassigned:** Chỉ hiện Issue chưa được phân công

Nhóm có thể tự tạo thêm Quick Filter theo nhu cầu riêng trong **Board Settings → Quick Filters**, sử dụng cú pháp JQL để định nghĩa điều kiện lọc.

> 📌 **Khác biệt với Team-managed:** Trong Team-managed, Quick Filter tùy chỉnh bằng JQL **không có sẵn**. Thay vào đó, Board cung cấp bộ lọc nhanh cơ bản theo **Assignee** và **Label** ngay trên thanh công cụ Board — nhấn vào avatar của thành viên hoặc chọn Label để lọc nhanh.

---

## 6.9. Theo dõi tiến độ trên Board

Board không chỉ là nơi di chuyển Issue — đây còn là công cụ theo dõi tiến độ quan trọng trong Daily Scrum.

### Daily Standup và Board

**Daily Standup** (hay Daily Scrum) là cuộc họp ngắn hàng ngày, thường kéo dài không quá 15 phút. Nhóm nhìn vào Board và mỗi thành viên trả lời ba câu hỏi:

1. Hôm qua tôi đã làm gì?
2. Hôm nay tôi sẽ làm gì?
3. Có điều gì đang cản trở tôi không?

Board giúp cuộc họp này diễn ra nhanh chóng và tập trung vì mọi người đều thấy trực tiếp công việc của nhau.

### Các dấu hiệu cần chú ý khi nhìn Board:

| Dấu hiệu | Ý nghĩa |
|---|---|
| Nhiều Issue tắc ở cột "In Progress" | Nhóm đang làm quá nhiều việc cùng lúc |
| Cột "Review" hoặc "Testing" quá đầy | Bottleneck ở bước kiểm tra, cần thêm nguồn lực |
| Cột "To Do" gần hết mà Sprint chưa xong | Tốt — nhóm đang hoàn thành nhanh hơn kế hoạch |
| Một thành viên có quá nhiều Issue cùng lúc | Cần cân bằng lại phân công |
| Issue nằm mãi không di chuyển | Issue có thể đang bị blocked, cần hỗ trợ |

---

## Tóm tắt chương

Board là giao diện làm việc hàng ngày của nhóm — nơi mọi tiến độ được phản ánh tức thì và mọi tắc nghẽn được phát hiện sớm. Sử dụng Board hiệu quả, kết hợp với Swimlane, Filter và các quy ước làm việc rõ ràng, là yếu tố then chốt để nhóm vận hành trơn tru trong từng Sprint.

| Khái niệm | Ý nghĩa ngắn gọn |
|---|---|
| Board | Giao diện trực quan theo dõi trạng thái công việc |
| Scrum Board | Board hiển thị Issue theo Sprint đang chạy |
| Kanban Board | Board hiển thị luồng công việc liên tục |
| WIP Limit | Giới hạn số Issue tối đa đang xử lý cùng lúc |
| Definition of Done | Tiêu chí xác định một Issue thực sự hoàn thành |
| Swimlane | Phân chia Board theo chiều ngang theo tiêu chí |
| Quick Filter | Bộ lọc nhanh hiển thị ngay trên Board |
| Daily Standup | Họp ngắn hàng ngày dựa trên Board |

---

*Chương tiếp theo: **Chương 7 — Workflow: Quy trình Chuyển trạng thái Công việc***
