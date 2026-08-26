# CHƯƠNG 5: SPRINT — QUẢN LÝ CHU KỲ PHÁT TRIỂN

---

## 5.1. Sprint là gì?

**Sprint** là một khoảng thời gian làm việc có giới hạn và cố định, trong đó nhóm phát triển cam kết hoàn thành một tập hợp công việc đã được lựa chọn từ Backlog. Sprint là đơn vị lặp lại cốt lõi trong phương pháp Scrum — toàn bộ quy trình phát triển được tổ chức xoay quanh nhịp điệu đều đặn của các Sprint liên tiếp nhau.

Thời lượng của một Sprint thường dao động từ **1 đến 4 tuần**, trong đó phổ biến nhất là Sprint **2 tuần**. Thời lượng này được cố định và nhất quán trong suốt dự án — nhóm không rút ngắn hay kéo dài Sprint tùy tiện vì lý do tiến độ.

### Tại sao làm việc theo Sprint?

Làm việc theo chu kỳ Sprint mang lại một số lợi ích quan trọng:

- **Tạo nhịp điệu làm việc ổn định:** Nhóm biết rõ khi nào bắt đầu, khi nào kết thúc, và mục tiêu của từng giai đoạn là gì.
- **Phản hồi sớm và liên tục:** Sau mỗi Sprint, nhóm có sản phẩm cụ thể để trình bày, nhận phản hồi và điều chỉnh hướng đi nếu cần.
- **Kiểm soát rủi ro:** Vấn đề được phát hiện sớm sau mỗi Sprint thay vì tích lũy đến cuối dự án.
- **Tăng tính minh bạch:** Mọi người đều thấy được tiến độ thực tế qua từng Sprint.

---

## 5.2. Tạo Sprint

Trong Jira, Sprint được tạo từ giao diện Backlog. Trong Team-managed, **bất kỳ thành viên nào** có quyền Admin của Project đều có thể tạo Sprint — không yêu cầu vai trò Scrum Master riêng.

**Bước 1:** Vào **Backlog** của Project (chọn **Backlog** ở menu bên trái).

**Bước 2:** Nhấn nút **Create Sprint** ở phía trên vùng Backlog. Jira sẽ tạo một Sprint mới và hiển thị ngay phía trên vùng Backlog.

**Bước 3:** Nhấn vào biểu tượng **...** cạnh tên Sprint → chọn **Edit Sprint** → nhập tên Sprint, ngày bắt đầu, ngày kết thúc và Sprint Goal → nhấn **Update**.

> **Lưu ý:** Jira cho phép tạo nhiều Sprint cùng lúc để lập kế hoạch trước, nhưng chỉ một Sprint được phép **Active** (đang chạy) tại một thời điểm trong cùng một Board.

> 📌 **Khác biệt với Team-managed:** Giao diện Backlog trong Team-managed trông đơn giản hơn — không có thanh Epic Panel bên trái như Company-managed. Nút **Create Sprint** nằm ở trên cùng của vùng Backlog hoặc bên dưới Sprint cuối cùng.

---

## 5.3. Sprint Planning (Lập kế hoạch Sprint)

**Sprint Planning** là buổi họp mở đầu mỗi Sprint, nơi toàn bộ nhóm — bao gồm Product Owner, Scrum Master và Development Team — cùng xác định mục tiêu và danh sách công việc cho Sprint sắp tới.

### Hai câu hỏi trọng tâm của Sprint Planning:

**1. Nhóm sẽ làm gì trong Sprint này?**
Product Owner trình bày các Issue ưu tiên cao nhất trong Backlog. Nhóm thảo luận và chọn ra những Issue phù hợp với khả năng của nhóm trong thời gian Sprint.

**2. Nhóm sẽ làm điều đó như thế nào?**
Nhóm phát triển thảo luận về cách tiếp cận kỹ thuật, phân công ban đầu và ước lượng lại nếu cần. Các Issue lớn có thể được chia thành Sub-task ngay trong buổi này.

### Đầu vào và đầu ra của Sprint Planning:

| | Nội dung |
|---|---|
| **Đầu vào** | Product Backlog đã được sắp xếp ưu tiên, Velocity của Sprint trước, Capacity của nhóm |
| **Đầu ra** | Sprint Goal, Sprint Backlog (danh sách Issue được chọn), kế hoạch thực hiện ban đầu |

---

## 5.4. Chọn Issue vào Sprint

Việc chọn Issue nào để đưa vào Sprint là quyết định quan trọng nhất trong Sprint Planning. Nhóm cần cân bằng giữa tham vọng và thực tế.

### Nguyên tắc chọn Issue:

**Theo thứ tự ưu tiên:** Ưu tiên các Issue đứng đầu Backlog — đây là những việc quan trọng nhất mà Product Owner đã xác định.

**Theo Capacity:** Tổng Story Point của các Issue được chọn nên xấp xỉ với Velocity trung bình của nhóm trong các Sprint gần nhất. Không nên chọn quá nhiều — cam kết thực tế quan trọng hơn cam kết tham vọng.

**Theo Sprint Goal:** Các Issue được chọn nên có sự liên kết với nhau và cùng hướng đến một mục tiêu chung. Tránh chọn các Issue rời rạc, không có chủ đề thống nhất.

**Theo sự phụ thuộc:** Kiểm tra xem Issue có bị chặn bởi Issue khác chưa được hoàn thành không. Đưa vào Sprint một Issue đang bị block là lãng phí capacity.

---

## 5.5. Sprint Goal (Mục tiêu Sprint)

**Sprint Goal** là một câu mô tả ngắn gọn mục tiêu tổng thể mà nhóm muốn đạt được sau khi kết thúc Sprint. Sprint Goal không liệt kê từng công việc cụ thể mà nắm bắt **giá trị** hay **kết quả** mà Sprint hướng đến.

### Đặc điểm của một Sprint Goal tốt:

- Ngắn gọn, có thể đọc và nhớ dễ dàng
- Phản ánh giá trị mang lại cho người dùng hoặc hệ thống
- Đủ linh hoạt để nhóm có thể điều chỉnh cách thực hiện nhưng không thay đổi mục tiêu

> **Ví dụ Sprint Goal:**
> - *"Hoàn thiện toàn bộ luồng đăng ký và xác thực tài khoản người dùng."*
> - *"Tích hợp cổng thanh toán và cho phép người dùng thực hiện giao dịch đầu tiên."*
> - *"Giảm thời gian tải trang chủ xuống dưới 2 giây."*

Trong Jira, Sprint Goal được nhập khi tạo hoặc chỉnh sửa Sprint và hiển thị rõ ràng trên Board trong suốt thời gian Sprint chạy.

---

## 5.6. Theo dõi Sprint

Khi Sprint đã được khởi động (Start Sprint), nhóm chuyển sang giai đoạn thực hiện. Jira cung cấp nhiều công cụ để theo dõi tiến độ Sprint theo thời gian thực.

### Scrum Board

Đây là công cụ theo dõi chính trong Sprint. Board hiển thị tất cả Issue trong Sprint, được phân chia theo trạng thái (To Do, In Progress, Done...). Mỗi thành viên tự di chuyển Issue của mình sang trạng thái phù hợp khi có tiến triển.

### Burndown Chart

**Burndown Chart** là biểu đồ trực quan thể hiện tốc độ hoàn thành công việc trong Sprint. Trục ngang là thời gian (các ngày trong Sprint), trục dọc là tổng Story Point còn lại cần hoàn thành.

- **Đường lý tưởng (Ideal line):** Đường thẳng từ tổng Story Point ban đầu về 0 vào ngày cuối Sprint — thể hiện tốc độ hoàn thành đều đặn.
- **Đường thực tế (Actual line):** Đường biểu diễn Story Point thực sự còn lại mỗi ngày.

Khi đường thực tế nằm **trên** đường lý tưởng, nhóm đang chậm hơn kế hoạch. Khi nằm **dưới**, nhóm đang đi nhanh hơn dự kiến.

> 📌 **Khác biệt với Team-managed:** Burndown Chart trong Team-managed được truy cập qua **Backlog → chọn Sprint → xem biểu đồ** hoặc qua menu **Reports** ở thanh bên trái. Giao diện đơn giản hơn Company-managed nhưng vẫn đầy đủ thông tin cơ bản.

### Sprint Report

**Sprint Report** tổng hợp kết quả của một Sprint đã hoàn thành, bao gồm: danh sách Issue hoàn thành, Issue chưa hoàn thành, và so sánh giữa kế hoạch ban đầu với kết quả thực tế.

---

## 5.7. Chỉnh sửa Sprint

Trong quá trình Sprint đang chạy, người có quyền quản lý có thể chỉnh sửa một số thông tin của Sprint như tên, ngày kết thúc hoặc Sprint Goal nếu có sự thay đổi cần thiết.

Để chỉnh sửa, vào **Backlog** → nhấn vào biểu tượng **...** (More) cạnh tên Sprint → chọn **Edit Sprint**.

> **Lưu ý quan trọng:** Không nên thêm Issue mới vào Sprint đang chạy một cách tùy tiện. Nếu có công việc phát sinh khẩn cấp, Scrum Master và Product Owner cần cân nhắc kỹ — thêm Issue mới đồng nghĩa với việc nhóm phải làm nhiều hơn cam kết ban đầu hoặc phải bỏ bớt Issue đã có.

---

## 5.8. Complete/Close Sprint (Kết thúc Sprint)

Khi Sprint đến ngày kết thúc, người quản lý tiến hành đóng Sprint trong Jira.

**Bước 1:** Vào **Board** của Sprint đang chạy.

**Bước 2:** Nhấn nút **Complete Sprint** ở góc trên bên phải Board.

> 📌 **Với Team-managed:** Nút này có thể có tên là **"Complete sprint"** và nằm trong menu **"..."** ở góc trên bên phải, hoặc hiển thị trực tiếp trên Board tùy phiên bản Jira.

**Bước 3:** Jira hiển thị một hộp thoại tóm tắt: số Issue đã hoàn thành và số Issue chưa hoàn thành.

**Bước 4:** Chọn nơi chuyển các Issue chưa hoàn thành — về **Backlog** hoặc vào Sprint tiếp theo (nếu đã tạo sẵn).

**Bước 5:** Nhấn **Complete sprint** để xác nhận. Sprint sẽ được đóng lại.

---

## 5.9. Xử lý Issue chưa hoàn thành

Không phải Sprint nào cũng hoàn thành 100% công việc đã cam kết — đây là điều bình thường trong thực tế. Điều quan trọng là nhóm xử lý các Issue dang dở một cách minh bạch và có hệ thống.

### Các lựa chọn xử lý:

**a) Chuyển về Backlog:** Issue chưa hoàn thành được đưa trở lại Backlog, sắp xếp lại thứ tự ưu tiên và xem xét đưa vào Sprint tiếp theo.

**b) Chuyển thẳng vào Sprint tiếp theo:** Nếu Issue đang dở dang và cần tiếp tục ngay, có thể chuyển thẳng vào Sprint kế tiếp.

**c) Chia nhỏ Issue:** Nếu Issue quá lớn và không thể hoàn thành trong một Sprint, nên chia thành nhiều Issue nhỏ hơn trước khi đưa vào Sprint mới.

### Phân tích nguyên nhân:

Khi nhiều Issue không hoàn thành, nhóm nên tìm hiểu nguyên nhân trong buổi **Sprint Retrospective**:
- Issue được estimate quá thấp so với thực tế?
- Có phát sinh công việc ngoài kế hoạch?
- Có thành viên bị ốm hoặc vắng mặt?
- Có sự phụ thuộc kỹ thuật không được phát hiện sớm?

---

## 5.10. Sprint Review và Sprint Retrospective

Mỗi Sprint kết thúc với hai buổi họp quan trọng, thường được tổ chức vào ngày cuối của Sprint.

### Sprint Review (Đánh giá Sprint)

**Sprint Review** là buổi họp để nhóm **trình bày kết quả** cho Product Owner và các stakeholder. Mục tiêu là nhận phản hồi về sản phẩm đã hoàn thiện trong Sprint và điều chỉnh hướng phát triển nếu cần.

Đây không phải buổi báo cáo trạng thái — mà là buổi demo thực tế sản phẩm. Nhóm chỉ trình bày những gì đã **Done** theo định nghĩa đã thống nhất, không trình bày công việc dang dở.

**Kết quả của Sprint Review:**
- Danh sách tính năng đã hoàn thành và được chấp thuận
- Phản hồi từ stakeholder
- Điều chỉnh Product Backlog nếu cần

### Sprint Retrospective (Cải tiến quy trình)

**Sprint Retrospective** là buổi họp nội bộ của nhóm — không có stakeholder bên ngoài — nhằm **nhìn lại quy trình làm việc** và tìm cách cải thiện trong Sprint tới.

Ba câu hỏi trọng tâm của Retrospective:

| Câu hỏi | Mục đích |
|---|---|
| **Điều gì đã làm tốt?** | Ghi nhận và duy trì những điểm mạnh |
| **Điều gì chưa tốt?** | Nhận diện vấn đề cần khắc phục |
| **Chúng ta sẽ cải thiện điều gì trong Sprint tới?** | Xác định hành động cụ thể |

Retrospective là cơ chế cải tiến liên tục (continuous improvement) — một trong những nguyên tắc nền tảng của Agile.

---

## Tóm tắt chương

Sprint là đơn vị tổ chức thời gian cốt lõi trong Scrum. Hiểu rõ vòng đời của một Sprint — từ lập kế hoạch, thực hiện, theo dõi cho đến đánh giá và cải tiến — giúp nhóm làm việc có nhịp điệu, minh bạch và hiệu quả hơn qua từng chu kỳ.

| Khái niệm | Ý nghĩa ngắn gọn |
|---|---|
| Sprint | Chu kỳ làm việc cố định (1–4 tuần) |
| Sprint Planning | Buổi lập kế hoạch đầu Sprint |
| Sprint Goal | Mục tiêu tổng thể của Sprint |
| Sprint Backlog | Danh sách Issue được chọn cho Sprint |
| Burndown Chart | Biểu đồ theo dõi tiến độ hoàn thành |
| Sprint Review | Buổi trình bày kết quả cho stakeholder |
| Sprint Retrospective | Buổi họp cải tiến quy trình nội bộ |
| Velocity | Tổng Story Point hoàn thành trong Sprint |

---

*Chương tiếp theo: **Chương 6 — Jira Board: Theo dõi Công việc Trực quan***
