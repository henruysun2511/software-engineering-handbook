# CHƯƠNG 11: DASHBOARD & REPORTS — BÁO CÁO VÀ THỐNG KÊ

---

## 11.1. Tổng quan về Dashboard và Reports

Jira không chỉ là nơi tạo và theo dõi Issue — đây còn là một nguồn dữ liệu phong phú về hiệu suất làm việc của nhóm, tiến độ dự án và chất lượng sản phẩm. **Dashboard** và **Reports** là hai công cụ giúp khai thác và trực quan hóa dữ liệu đó.

- **Dashboard** là trang tổng hợp thông tin tùy chỉnh, hiển thị nhiều loại dữ liệu cùng lúc thông qua các widget gọi là Gadget. Dashboard phù hợp cho việc theo dõi hàng ngày với cái nhìn tổng thể nhanh.

- **Reports** là các báo cáo chuyên sâu được tích hợp sẵn trong từng Project, tập trung phân tích một khía cạnh cụ thể như tiến độ Sprint, tốc độ nhóm hay luồng công việc. Reports phù hợp cho phân tích định kỳ sau mỗi Sprint.

---

## 11.2. Dashboard

### 11.2.1. Dashboard là gì?

**Dashboard** trong Jira là một trang web có thể tùy chỉnh hoàn toàn, cho phép người dùng đặt nhiều **Gadget** (widget hiển thị dữ liệu) theo bố cục tự chọn. Mỗi Gadget kết nối với một Filter hoặc Project cụ thể và hiển thị dữ liệu theo thời gian thực.

Jira cho phép mỗi người dùng tạo nhiều Dashboard khác nhau phục vụ các mục đích khác nhau — ví dụ: Dashboard cá nhân để theo dõi công việc của bản thân, Dashboard nhóm để theo dõi tiến độ Sprint, hay Dashboard quản lý để tổng hợp tình trạng nhiều Project.

### 11.2.2. Tạo Dashboard mới

**Bước 1:** Từ thanh điều hướng trên cùng → nhấn **Dashboards** → chọn **Create Dashboard**.

**Bước 2:** Đặt tên Dashboard và mô tả ngắn gọn mục đích sử dụng.

**Bước 3:** Chọn chế độ chia sẻ:
- *Private:* Chỉ mình tôi thấy
- *Shared with group:* Chia sẻ với một nhóm cụ thể
- *Public:* Toàn bộ tổ chức có thể xem

**Bước 4:** Nhấn **Create** để hoàn tất. Dashboard trống sẽ được tạo ra, sẵn sàng để thêm Gadget.

### 11.2.3. Các Gadget phổ biến

Sau khi có Dashboard, người dùng thêm Gadget bằng cách nhấn **Add Gadget** và chọn từ thư viện. Dưới đây là các Gadget được sử dụng nhiều nhất:

> 📌 **Khác biệt với Team-managed:** Dashboard và Gadget hoạt động tương tự nhau ở cả hai loại project. Tuy nhiên một số Gadget nâng cao (như **Two-Dimensional Filter Statistics**) có thể không khả dụng hoặc hiển thị khác trong Team-managed.

**a) Issue Statistics**

Hiển thị phân bổ Issue theo một tiêu chí bất kỳ (Status, Priority, Assignee, Issue Type...) dưới dạng biểu đồ tròn hoặc cột. Ví dụ: phân bổ Bug theo mức Priority giúp thấy ngay bao nhiêu Bug Critical đang tồn đọng.

**b) Assigned to Me**

Danh sách tất cả Issue đang được giao cho người dùng hiện tại, không phân biệt Project. Đây là Gadget hữu ích nhất cho Dashboard cá nhân — thức dậy buổi sáng mở Jira là thấy ngay danh sách việc cần làm hôm nay.

**c) Filter Results**

Hiển thị kết quả của một Filter JQL đã lưu dưới dạng danh sách Issue tùy chỉnh. Gadget này cực kỳ linh hoạt — bất kỳ tập hợp Issue nào có thể định nghĩa bằng JQL đều có thể hiển thị ở đây.

**d) Sprint Health**

Hiển thị tổng quan tình trạng Sprint đang chạy: số Issue theo từng trạng thái, Story Point đã hoàn thành, Story Point còn lại và số ngày còn lại trong Sprint.

**e) Created vs Resolved Chart**

Biểu đồ đường so sánh số Issue được tạo mới và số Issue được đóng (Resolved) theo thời gian. Khi đường "Created" luôn cao hơn đường "Resolved", đây là tín hiệu nợ kỹ thuật (technical debt) đang tích lũy.

**f) Two-Dimensional Filter Statistics**

Hiển thị dữ liệu Issue theo hai chiều — ví dụ: Assignee theo Status, tạo thành bảng ma trận giúp nhìn ngay ai đang có bao nhiêu việc ở từng trạng thái.

**g) Activity Stream**

Hiển thị nhật ký hoạt động theo thời gian thực — ai vừa tạo Issue gì, ai vừa chuyển trạng thái, ai vừa comment... Gadget này giúp nắm bắt nhanh những gì đang xảy ra trong Project mà không cần vào từng Issue.

---

## 11.3. Reports (Báo cáo tích hợp)

Khác với Dashboard (tùy chỉnh), **Reports** là các báo cáo được thiết kế sẵn trong từng Project, tập trung phân tích chuyên sâu một khía cạnh cụ thể. Để xem Reports, vào Project → chọn **Reports** trên menu điều hướng bên trái.

> 📌 **Khác biệt với Team-managed:** Trong Team-managed, các báo cáo được truy cập qua menu **Reports** ở thanh bên trái của Project. Giao diện đơn giản hơn nhưng vẫn có các báo cáo cơ bản: **Burndown Chart**, **Velocity Chart** và **Sprint Report**. Một số báo cáo nâng cao như **Version Report** và **Epic Burndown** không có trong Team-managed.

### 11.3.1. Burndown Chart

**Burndown Chart** đã được đề cập trong Chương 5 (Sprint) và Chương 8 (Estimation & Tracking). Đây là báo cáo quan trọng nhất trong Scrum, theo dõi tốc độ hoàn thành công việc trong một Sprint.

**Cách đọc Burndown Chart:**

- Trục ngang (X): Các ngày trong Sprint
- Trục dọc (Y): Story Point còn lại cần hoàn thành
- Đường xám (Guideline): Tốc độ hoàn thành lý tưởng — giảm đều đặn từ tổng Story Point về 0
- Đường xanh (Actual): Tốc độ hoàn thành thực tế

Khi đường Actual nằm **phía trên** đường Guideline → nhóm đang chậm hơn kế hoạch.
Khi đường Actual nằm **phía dưới** đường Guideline → nhóm đang hoàn thành nhanh hơn.

### 11.3.2. Velocity Chart

**Velocity Chart** hiển thị Velocity của nhóm qua nhiều Sprint liên tiếp, dưới dạng biểu đồ cột đôi cho mỗi Sprint.

**Cách đọc Velocity Chart:**

- Cột xám: Story Point đã *cam kết* (đưa vào Sprint Planning)
- Cột xanh: Story Point thực tế *hoàn thành*

Velocity Chart cho thấy xu hướng dài hạn của nhóm. Nếu cột xanh luôn thấp hơn cột xám nhiều — nhóm đang liên tục cam kết quá mức so với khả năng thực tế. Nếu biểu đồ nhấp nhô không đều — nhóm đang gặp khó khăn về tính nhất quán, cần xem lại quy trình.

Velocity trung bình tính từ Velocity Chart là cơ sở để lập kế hoạch cho Sprint tiếp theo.

### 11.3.3. Sprint Report

**Sprint Report** là báo cáo tổng kết sau khi một Sprint kết thúc, tự động được tạo bởi Jira khi nhấn "Complete Sprint".

Sprint Report bao gồm:
- Danh sách Issue **hoàn thành** trong Sprint (với Story Point tương ứng)
- Danh sách Issue **chưa hoàn thành** (được chuyển về Backlog hoặc Sprint tiếp theo)
- Tổng Story Point cam kết so với tổng Story Point hoàn thành
- Biểu đồ Burndown của Sprint vừa kết thúc

Sprint Report là tài liệu quan trọng để trình bày trong buổi Sprint Review và lưu trữ làm lịch sử dự án.

### 11.3.4. Cumulative Flow Diagram (CFD)

**Cumulative Flow Diagram (CFD)** là báo cáo đặc trưng của Kanban, hiển thị số lượng Issue tích lũy ở từng trạng thái theo thời gian dưới dạng biểu đồ vùng (area chart).

**Cách đọc CFD:**

Mỗi vùng màu đại diện cho một Status. Độ rộng theo chiều dọc của mỗi vùng tại một thời điểm = số Issue đang ở Status đó.

- Nếu vùng "In Progress" **ngày càng rộng ra** → Issue đang tích tụ, luồng công việc đang bị nghẽn
- Nếu các vùng **song song và đều nhau** → luồng công việc ổn định, tốc độ xử lý nhất quán
- **Khoảng cách theo chiều ngang** giữa khi Issue vào một Status và khi ra = thời gian trung bình xử lý (Cycle Time)

CFD đặc biệt hữu ích để phát hiện tắc nghẽn hệ thống (system bottleneck) — trạng thái nào đang tích lũy Issue nhiều bất thường.

### 11.3.5. Version Report

**Version Report** theo dõi tiến độ hoàn thành công việc theo từng phiên bản phát hành (Version/Release). Báo cáo này hữu ích với các dự án có lịch phát hành định kỳ, giúp biết được phiên bản sắp tới sẽ hoàn thành đúng hạn hay không dựa trên tốc độ hiện tại.

### 11.3.6. Epic Report và Epic Burndown

**Epic Report** hiển thị tiến độ hoàn thành của từng Epic — bao nhiêu Story/Task đã xong, bao nhiêu còn lại — giúp theo dõi tiến độ phát triển từng tính năng lớn.

**Epic Burndown** tương tự Burndown Chart của Sprint nhưng áp dụng cho một Epic cụ thể, theo dõi tốc độ hoàn thành các Issue bên trong Epic đó theo thời gian.

---

## 11.4. Lựa chọn Dashboard hay Reports?

| Tiêu chí | Dashboard | Reports |
|---|---|---|
| **Tần suất sử dụng** | Hàng ngày | Định kỳ (cuối Sprint, cuối tháng) |
| **Phạm vi** | Nhiều Project, nhiều chiều | Một Project, một chủ đề |
| **Tùy chỉnh** | Cao (chọn Gadget, bố cục) | Thấp (báo cáo cố định) |
| **Mục đích** | Theo dõi nhanh, tổng quan | Phân tích sâu, retrospective |
| **Người dùng chính** | Toàn bộ nhóm | Scrum Master, Project Manager |

Trong thực tế, hai công cụ này bổ trợ lẫn nhau: Dashboard cho cái nhìn hàng ngày, Reports cho phân tích định kỳ và cải thiện quy trình.

---

## 11.5. Thực hành: Xây dựng Dashboard nhóm hiệu quả

Một Dashboard nhóm tốt nên trả lời được ba câu hỏi cốt lõi chỉ bằng một cái nhìn:

1. **Sprint đang đi đến đâu?** → Dùng Sprint Health Gadget + Burndown Chart
2. **Ai đang làm gì?** → Dùng Two-Dimensional Filter Statistics (Assignee × Status)
3. **Có vấn đề gì cần chú ý?** → Dùng Filter Results với JQL lọc Bug Critical hoặc Issue quá hạn

### Gợi ý bố cục Dashboard nhóm:

```
┌─────────────────────┬─────────────────────┐
│   Sprint Health     │   Burndown Chart    │
├─────────────────────┴─────────────────────┤
│      Two-Dimensional: Assignee × Status   │
├─────────────────────┬─────────────────────┤
│  Bug Critical/High  │   Overdue Issues    │
│   (Filter Result)   │   (Filter Result)   │
└─────────────────────┴─────────────────────┘
```

---

## Tóm tắt chương

Dashboard và Reports biến Jira từ một công cụ quản lý công việc đơn thuần thành một hệ thống thông tin dự án toàn diện. Sử dụng thành thạo hai công cụ này giúp nhóm luôn nắm bắt được thực trạng dự án, phát hiện sớm vấn đề và đưa ra quyết định dựa trên dữ liệu thay vì cảm tính.

| Khái niệm | Ý nghĩa ngắn gọn |
|---|---|
| Dashboard | Trang tổng hợp thông tin tùy chỉnh |
| Gadget | Widget hiển thị một loại dữ liệu cụ thể trên Dashboard |
| Filter Results Gadget | Gadget hiển thị kết quả của Filter JQL |
| Burndown Chart | Theo dõi tốc độ hoàn thành Story Point trong Sprint |
| Velocity Chart | Hiển thị Velocity qua nhiều Sprint liên tiếp |
| Sprint Report | Báo cáo tổng kết sau khi Sprint kết thúc |
| Cumulative Flow Diagram | Theo dõi luồng công việc qua các trạng thái theo thời gian |
| Epic Report | Theo dõi tiến độ hoàn thành của một Epic |

---

*Chương tiếp theo: **Chương 12 — Jira & Git: Tích hợp với Hệ thống Quản lý Mã nguồn***
