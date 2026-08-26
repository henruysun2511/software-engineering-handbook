# CHƯƠNG 8: ESTIMATION & TRACKING — ƯỚC LƯỢNG VÀ THEO DÕI TIẾN ĐỘ

---

## 8.1. Tổng quan về Estimation và Tracking

Trong quản lý dự án phần mềm, hai hoạt động song hành và bổ trợ lẫn nhau là **Estimation** (ước lượng) và **Tracking** (theo dõi).

- **Estimation** diễn ra *trước* khi công việc bắt đầu — nhóm đánh giá mức độ phức tạp, thời gian hoặc công sức cần thiết để hoàn thành một Issue.
- **Tracking** diễn ra *trong và sau* khi công việc diễn ra — ghi nhận thực tế đã xảy ra, so sánh với ước lượng ban đầu và điều chỉnh kế hoạch nếu cần.

Hai hoạt động này tạo thành một vòng phản hồi (feedback loop): kết quả Tracking của Sprint này trở thành dữ liệu để Estimation của Sprint sau chính xác hơn.

---

## 8.2. Story Point — Ôn lại và mở rộng

Story Point đã được giới thiệu trong Chương 4. Chương này đi sâu hơn vào cách sử dụng Story Point trong thực tế nhóm.

### Quy tắc sử dụng Story Point hiệu quả:

**a) Estimation là hoạt động tập thể**

Không một cá nhân nào — kể cả Scrum Master hay Tech Lead — đơn phương quyết định Story Point cho toàn bộ Issue. Mọi thành viên trong nhóm phát triển đều tham gia thảo luận, vì mỗi người có góc nhìn khác nhau về mức độ phức tạp.

**b) Dùng Issue tham chiếu (Reference Story)**

Khi nhóm mới bắt đầu dùng Story Point, nên chọn một Issue tiêu biểu làm "mốc tham chiếu" — thường là một Issue có độ phức tạp trung bình và được toàn nhóm hiểu rõ. Sau đó, các Issue mới được ước lượng so sánh với mốc này: *"Issue này phức tạp hơn hay đơn giản hơn Issue tham chiếu?"*

**c) Không quy đổi Story Point sang giờ**

Story Point và giờ làm việc là hai đơn vị đo lường hoàn toàn khác nhau về bản chất. Một nhóm có thể hoàn thành 10 Story Point trong 20 giờ, nhóm khác mất 30 giờ cho cùng 10 Story Point — và cả hai đều hoàn toàn bình thường. Điều quan trọng là nhất quán *trong nội bộ nhóm*.

**d) Không so sánh Story Point giữa các nhóm**

Story Point chỉ có ý nghĩa tương đối trong phạm vi một nhóm cụ thể. So sánh Story Point giữa các nhóm khác nhau là vô nghĩa và dễ gây hiểu lầm.

### Planning Poker

**Planning Poker** là kỹ thuật Estimation phổ biến nhất trong Scrum. Mỗi thành viên nhóm có một bộ bài với các con số theo dãy Fibonacci. Khi estimate một Issue:

1. Product Owner trình bày Issue, giải thích yêu cầu
2. Mỗi người chọn một con số trong bộ bài mà họ cho là phù hợp (chọn bí mật, không trao đổi trước)
3. Tất cả đồng thời lật bài
4. Nếu có sự chênh lệch lớn, những người chọn số cao nhất và thấp nhất giải thích lý do
5. Thảo luận và estimate lại cho đến khi đạt đồng thuận

Planning Poker giúp phát hiện các góc nhìn khác nhau về độ phức tạp của cùng một Issue, tránh hiệu ứng "anchoring" — khi một người đưa ra con số trước và vô tình ảnh hưởng đến phán đoán của người khác.

---

## 8.3. Time Tracking (Theo dõi thời gian)

Ngoài Story Point, Jira còn hỗ trợ **Time Tracking** — tính năng cho phép nhóm ước lượng và ghi nhận thời gian thực tế bỏ ra cho từng Issue theo đơn vị giờ và phút.

Time Tracking và Story Point không loại trừ nhau — nhiều nhóm sử dụng cả hai cùng lúc: Story Point để lập kế hoạch Sprint và đo Velocity, Time Tracking để quản lý ngân sách và báo cáo chi tiết.

> 📌 **Khác biệt với Team-managed:** Tính năng **Time Tracking** (Original Estimate, Remaining Estimate, Log Work) **không được hỗ trợ** trong Team-managed project. Các mục 8.4, 8.5, 8.6 dưới đây mô tả tính năng chỉ có trong **Company-managed**. Nếu bạn đang dùng Team-managed, hãy bỏ qua các mục đó và tập trung vào **Story Point** (mục 8.2) và **Velocity** (mục 8.7) để theo dõi tiến độ.

### Khi nào nên dùng Time Tracking?

- Dự án có yêu cầu báo cáo thời gian thực tế (timesheet) cho khách hàng hoặc quản lý
- Nhóm cần theo dõi sát sao việc phân bổ thời gian cho từng loại công việc
- Dự án tính phí theo giờ (time & material contract)

---

## 8.4. Original Estimate (Ước lượng ban đầu)

**Original Estimate** là thời gian nhóm dự kiến cần để hoàn thành một Issue, được nhập trước khi bắt đầu làm. Đây là "hợp đồng" ban đầu về kỳ vọng thời gian.

Original Estimate được nhập trong trường **Estimated** (hoặc **Original Estimate**) trên trang chi tiết Issue, theo định dạng: `2h` (2 giờ), `1d` (1 ngày), `3h 30m` (3 giờ 30 phút)...

> **Nguyên tắc quan trọng:** Original Estimate không bao giờ được thay đổi sau khi công việc bắt đầu — dù thực tế có khác xa dự kiến. Giá trị này được giữ nguyên để làm cơ sở so sánh và rút kinh nghiệm.

---

## 8.5. Remaining Estimate (Thời gian còn lại)

**Remaining Estimate** là ước lượng thời gian còn cần để hoàn thành Issue, được cập nhật liên tục trong quá trình làm việc.

Khi Issue bắt đầu, Remaining Estimate ban đầu bằng với Original Estimate. Sau mỗi lần log công việc, người thực hiện cập nhật Remaining Estimate để phản ánh thực tế còn bao nhiêu công việc chưa làm.

Ví dụ: Issue được estimate là 8 giờ. Sau khi làm được 3 giờ, người thực hiện nhận ra công việc phức tạp hơn dự kiến — họ cập nhật Remaining Estimate lên 7 giờ (thay vì 5 giờ như tính toán đơn giản).

Remaining Estimate chính xác giúp Burndown Chart phản ánh đúng tiến độ thực tế của Sprint.

---

## 8.6. Logged Work (Thời gian đã ghi nhận)

**Logged Work** (hay Work Log) là tổng thời gian thực tế đã được ghi nhận cho một Issue thông qua tính năng **Log Work**.

### Cách log công việc:

**Bước 1:** Mở Issue cần log → nhấn **Log Work** (hoặc vào menu **More → Log Work**).

**Bước 2:** Điền **Time Spent** — thời gian đã làm trong lần này (ví dụ: `2h 30m`).

**Bước 3:** Chọn **Date Started** — ngày bắt đầu làm việc đó (thường là hôm nay).

**Bước 4:** Cập nhật **Remaining Estimate** — thời gian còn lại sau khi trừ phần vừa làm.

**Bước 5:** Nhập mô tả ngắn về công việc đã thực hiện (tùy chọn nhưng nên có).

**Bước 6:** Nhấn **Save**.

### Tần suất log:

Tốt nhất là log công việc hàng ngày — cuối mỗi ngày làm việc hoặc ngay sau khi hoàn thành một phần công việc. Tránh để tích lũy và log một lúc nhiều ngày — độ chính xác sẽ giảm đáng kể.

---

## 8.7. Velocity (Vận tốc nhóm)

**Velocity** là tổng Story Point mà nhóm hoàn thành trong một Sprint. Đây là chỉ số phản ánh **năng lực thực tế** của nhóm — không phải năng lực lý thuyết hay kỳ vọng từ bên ngoài.

### Cách tính Velocity:

Velocity = Tổng Story Point của tất cả Issue đạt trạng thái **Done** trong Sprint

Ví dụ: Sprint có 12 Issue, Story Point tương ứng là 2, 3, 5, 3, 8, 5, 2, 1, 3, 5, 3, 2. Nếu hoàn thành 10 Issue đầu, Velocity = 2+3+5+3+8+5+2+1+3+5 = **37 Story Point**.

### Velocity trung bình:

Không nên dựa vào Velocity của một Sprint duy nhất để lập kế hoạch. Velocity có thể dao động do nhiều yếu tố: thành viên nghỉ phép, Issue được estimate sai, công việc phát sinh ngoài kế hoạch...

Thay vào đó, dùng **Velocity trung bình của 3–5 Sprint gần nhất** để ước lượng khả năng của nhóm trong Sprint tiếp theo.

### Velocity Chart trong Jira:

Jira tự động tạo **Velocity Chart** — biểu đồ cột hiển thị Velocity của từng Sprint trong lịch sử. Mỗi Sprint có hai cột: màu xám (Story Point đã cam kết) và màu xanh (Story Point thực tế hoàn thành). Biểu đồ này giúp nhận biết xu hướng và tính nhất quán của nhóm.

---

## 8.8. Capacity (Năng lực khả dụng của nhóm)

**Capacity** là lượng thời gian thực tế mà nhóm có thể dành cho công việc trong một Sprint, sau khi trừ đi các yếu tố không làm việc như: nghỉ phép, ngày lễ, họp hành, công việc hành chính...

### Tại sao Capacity quan trọng?

Velocity trung bình đo năng lực *trong điều kiện bình thường*. Nhưng không phải Sprint nào cũng có điều kiện bình thường. Nếu Sprint tới có 2 người nghỉ phép và 2 ngày lễ, Capacity thực tế sẽ thấp hơn đáng kể — nhóm không nên cam kết cùng lượng Story Point như Sprint bình thường.

### Cách tính Capacity đơn giản:

```
Capacity Sprint = Σ (Số ngày làm việc của từng thành viên) × (% thời gian dành cho Sprint)
```

Ví dụ: Nhóm 4 người, Sprint 10 ngày. Một người nghỉ 2 ngày, một người có 20% thời gian dành cho công việc khác:

- Người 1: 10 ngày × 100% = 10 ngày
- Người 2: 8 ngày × 100% = 8 ngày (nghỉ 2 ngày)
- Người 3: 10 ngày × 80% = 8 ngày (20% thời gian cho việc khác)
- Người 4: 10 ngày × 100% = 10 ngày
- **Tổng Capacity: 36 ngày-người**

Dựa trên Velocity lịch sử và Capacity tính toán, nhóm điều chỉnh lượng Story Point cam kết cho Sprint phù hợp.

---

## 8.9. Burndown Chart — Phân tích chuyên sâu

Burndown Chart đã được đề cập trong Chương 5. Phần này phân tích các mẫu (pattern) phổ biến trên Burndown Chart và ý nghĩa của chúng.

### Các mẫu Burndown Chart thường gặp:

**Mẫu 1 — Lý tưởng (Ideal):**
Đường thực tế bám sát đường lý tưởng. Nhóm hoàn thành đều đặn mỗi ngày, không có tắc nghẽn lớn. Đây là mẫu mong muốn nhưng hiếm gặp trong thực tế.

**Mẫu 2 — Chậm đầu Sprint, nhanh cuối Sprint (Late Rush):**
Đường thực tế nằm cao hơn đường lý tưởng trong phần lớn Sprint, rồi giảm mạnh vào cuối. Thường xảy ra khi nhóm trì hoãn công việc hoặc chỉ tập trung làm khi sắp đến deadline. Đây là dấu hiệu của văn hóa làm việc cần cải thiện.

**Mẫu 3 — Đường phẳng (Flat Line):**
Story Point không giảm trong nhiều ngày liên tiếp. Dấu hiệu rõ ràng của tắc nghẽn (bottleneck) — có thể do Issue đang bị blocked, nhóm đang chờ phản hồi bên ngoài, hoặc quy trình review/testing bị nghẽn.

**Mẫu 4 — Đường tăng (Upward Spike):**
Story Point tăng lên thay vì giảm. Thường xảy ra khi Issue mới được thêm vào Sprint trong lúc đang chạy, hoặc khi estimate lại một Issue và tăng Story Point.

**Mẫu 5 — Giảm đột ngột về 0 trước hạn:**
Nhóm hoàn thành sớm hơn kế hoạch. Có thể là tín hiệu tốt, hoặc cho thấy nhóm đã estimate quá cao (over-estimate) — cam kết ít hơn khả năng thực sự.

---

## 8.10. Mối liên hệ giữa các chỉ số

Các chỉ số Estimation và Tracking trong Jira không tồn tại độc lập — chúng tạo thành một hệ thống phản hồi liên tục:

```
Capacity → Lập kế hoạch Sprint → Commit Story Point
     ↓
  Sprint thực hiện
     ↓
Velocity (kết quả thực tế) → Burndown Chart
     ↓
So sánh Original vs Logged Work → Cải thiện Estimation
     ↓
  Sprint tiếp theo (lập kế hoạch tốt hơn)
```

Nhóm sử dụng dữ liệu từ Velocity và so sánh Estimate vs Actual để trả lời các câu hỏi:
- Nhóm đang estimate chính xác hay liên tục sai lệch theo một hướng?
- Loại công việc nào thường mất nhiều thời gian hơn dự kiến?
- Capacity của nhóm có đang được sử dụng hiệu quả không?

---

## Tóm tắt chương

Estimation và Tracking là hai mặt của cùng một đồng xu: ước lượng tốt giúp lập kế hoạch Sprint thực tế, theo dõi chính xác giúp cải thiện ước lượng trong tương lai. Kết hợp Story Point, Time Tracking, Velocity và Burndown Chart, nhóm có đủ công cụ để vừa kiểm soát hiện tại vừa học hỏi cho Sprint sau.

| Khái niệm | Ý nghĩa ngắn gọn |
|---|---|
| Story Point | Đơn vị đo độ phức tạp tương đối của Issue |
| Planning Poker | Kỹ thuật estimation tập thể dùng bài số |
| Time Tracking | Ghi nhận thời gian thực tế làm việc |
| Original Estimate | Ước lượng thời gian ban đầu, không thay đổi sau khi bắt đầu |
| Remaining Estimate | Thời gian còn lại, cập nhật liên tục |
| Logged Work | Tổng thời gian thực tế đã ghi nhận |
| Velocity | Story Point hoàn thành trong một Sprint |
| Capacity | Thời gian thực tế nhóm có thể làm việc trong Sprint |
| Burndown Chart | Biểu đồ theo dõi tiến độ hoàn thành theo ngày |

---

*Chương tiếp theo: **Chương 9 — JQL & Search: Tìm kiếm và Lọc Issue Nâng cao***
