# CHƯƠNG 14: QUY TRÌNH SCRUM THỰC TẾ TRÊN JIRA

---

## 14.1. Tổng quan

Các chương trước đã giới thiệu từng thành phần riêng lẻ của Jira — Issue, Backlog, Sprint, Board, Workflow, JQL... Chương này kết nối tất cả lại, trình bày **cách một nhóm phát triển phần mềm vận hành toàn bộ quy trình Scrum trong thực tế** trên nền tảng Jira.

Đây là chương mang tính ứng dụng cao nhất — không phải lý thuyết về từng tính năng, mà là bức tranh toàn cảnh về cách mọi thứ kết hợp với nhau trong một chu kỳ phát triển thực tế, từ yêu cầu ban đầu cho đến lúc sản phẩm được phát hành.

---

## 14.2. Bức tranh toàn cảnh — Vòng đời một Sprint

```
Product Backlog (tồn tại liên tục)
        ↓
  Backlog Refinement
        ↓
   Sprint Planning
        ↓
  Sprint Backlog (bắt đầu Sprint)
        ↓
     Development
     /    |    \
  Code  Review  Test
        ↓
   Daily Scrum (hàng ngày)
        ↓
  Sprint Review (cuối Sprint)
        ↓
Sprint Retrospective (cuối Sprint)
        ↓
   Sprint tiếp theo
```

Vòng lặp này diễn ra đều đặn — thường là mỗi 2 tuần — trong suốt thời gian tồn tại của dự án. Không có điểm kết thúc cố định cho đến khi sản phẩm hoàn chỉnh hoặc dự án kết thúc.

---

## 14.3. Giai đoạn 1: Từ Requirement đến Product Backlog

### Bước 1 — Tiếp nhận yêu cầu

Mọi công việc trong Jira đều bắt đầu từ **yêu cầu** (requirement). Yêu cầu có thể đến từ nhiều nguồn: khách hàng, người dùng cuối, stakeholder nội bộ, hoặc từ chính nhóm phát triển (cải tiến kỹ thuật).

**Product Owner** là người chịu trách nhiệm tiếp nhận, làm rõ và chuyển hóa các yêu cầu thành Issue trong Jira.

### Bước 2 — Tạo Epic

Với các tính năng lớn hoặc mục tiêu kinh doanh quan trọng, Product Owner tạo **Epic** để nhóm chúng lại. Epic là cấp độ cao nhất trong phân cấp Issue, thường phản ánh một tính năng hoàn chỉnh hoặc một mục tiêu Sprint dài hạn.

Ví dụ: Dự án xây dựng hệ thống thương mại điện tử có thể có các Epic:
- `APP-1` — Hệ thống xác thực người dùng
- `APP-2` — Danh mục sản phẩm
- `APP-3` — Giỏ hàng và thanh toán
- `APP-4` — Quản lý đơn hàng

### Bước 3 — Phân rã Epic thành Story và Task

Mỗi Epic được phân rã thành các **Story** nhỏ hơn — đủ nhỏ để hoàn thành trong một Sprint (thường không quá 8 Story Point). Với công việc kỹ thuật không gắn với yêu cầu người dùng, tạo **Task** thay vì Story.

```
Epic: APP-1 — Hệ thống xác thực người dùng
├── Story: APP-10 — Đăng ký tài khoản mới
├── Story: APP-11 — Đăng nhập bằng email/mật khẩu
├── Story: APP-12 — Đăng nhập bằng Google OAuth
├── Story: APP-13 — Quên mật khẩu và đặt lại qua email
├── Story: APP-14 — Xác thực hai yếu tố (2FA)
└── Task:  APP-15 — Cấu hình JWT và refresh token
```

### Bước 4 — Thêm Issue vào Product Backlog

Sau khi tạo, tất cả Story và Task tự động xuất hiện trong **Backlog**. Product Owner sắp xếp thứ tự ưu tiên: Issue quan trọng nhất đặt lên đầu danh sách.

---

## 14.4. Giai đoạn 2: Backlog Refinement

Trước Sprint Planning khoảng 2–3 ngày, nhóm tổ chức buổi **Backlog Refinement** (30–60 phút) để chuẩn bị các Issue ưu tiên cao cho Sprint sắp tới.

### Checklist Backlog Refinement:

**Với từng Issue được xem xét:**

- [ ] Summary rõ ràng, đủ nghĩa khi đứng một mình
- [ ] Description đầy đủ — ai cũng hiểu cần làm gì
- [ ] Acceptance Criteria được xác định rõ (khi nào thì Issue này được coi là Done?)
- [ ] Story Point đã được estimate
- [ ] Sự phụ thuộc (dependency) với Issue khác đã được xác định
- [ ] Issue không quá lớn (≤ 8 Story Point) — nếu lớn hơn, cần chia nhỏ

**Trong Jira:**

Mở **Backlog** → duyệt từng Issue từ trên xuống → điền thông tin còn thiếu → estimate Story Point → sắp xếp lại thứ tự ưu tiên nếu cần.

---

## 14.5. Giai đoạn 3: Sprint Planning

Sprint Planning là buổi họp chính thức mở đầu Sprint mới. Toàn bộ nhóm cùng tham gia.

### Trình tự Sprint Planning trong Jira:

**Bước 1 — Tạo Sprint mới (nếu chưa có):**

Vào **Backlog** → nhấn **Create Sprint** → đặt tên Sprint theo quy ước của nhóm (ví dụ: *Sprint 12 — 03/06 đến 14/06*).

**Bước 2 — Xác định Sprint Goal:**

Product Owner trình bày mục tiêu tổng thể. Cả nhóm thảo luận và thống nhất Sprint Goal — một câu mô tả giá trị mà Sprint này hướng đến.

Nhập Sprint Goal khi chỉnh sửa Sprint: **Edit Sprint → Sprint Goal**.

**Bước 3 — Chọn Issue vào Sprint:**

Kéo Issue từ phần Backlog lên phần Sprint trong giao diện Backlog. Chọn theo thứ tự ưu tiên và dựa trên:
- Velocity trung bình của 3 Sprint gần nhất
- Capacity thực tế của nhóm trong Sprint này
- Sự phù hợp với Sprint Goal

**Bước 4 — Phân chia thành Sub-task (nếu cần):**

Với các Story phức tạp, nhóm có thể tạo Sub-task ngay trong Sprint Planning để phân công chi tiết hơn. Mở Story → **Child Issues → Create child issue → Sub-task**.

**Bước 5 — Start Sprint:**

Khi danh sách Issue đã thống nhất → nhấn **Start Sprint** → xác nhận ngày bắt đầu và ngày kết thúc → Sprint chính thức bắt đầu.

Board sẽ chuyển sang hiển thị các Issue trong Sprint vừa khởi động.

---

## 14.6. Giai đoạn 4: Development — Thực hiện Sprint

Đây là giai đoạn dài nhất và cốt lõi nhất của Sprint. Developer làm việc hàng ngày theo chu kỳ:

### Chu kỳ làm việc hàng ngày của Developer:

**Sáng — Daily Standup (15 phút):**

Cả nhóm nhìn vào Scrum Board. Từng người trả lời:
1. Hôm qua đã làm gì? (chỉ vào Issue trên Board)
2. Hôm nay sẽ làm gì?
3. Có bị tắc nghẽn gì không?

**Trong ngày — Làm việc và cập nhật Jira:**

```
1. Chọn Issue từ cột "To Do" trên Board
2. Tự assign cho mình (nếu chưa có Assignee)
3. Kéo Issue sang "In Progress"
4. Tạo branch Git: feature/APP-11-user-login
5. Viết code, commit thường xuyên với message đúng format:
   "APP-11: Add login form validation"
6. Cập nhật Comment nếu có tiến triển quan trọng hoặc phát sinh vấn đề
7. Khi hoàn thành code: tạo Pull Request, kéo Issue sang "Code Review"
8. Sau khi PR được merge: kéo Issue sang "Testing"
9. Sau khi QA pass: kéo Issue sang "Done"
```

### Xử lý các tình huống phát sinh:

**Khi Issue bị tắc (Blocked):**

Thêm Label `blocked` vào Issue → Comment giải thích đang bị tắc vì lý do gì và cần ai hỗ trợ → Mention Scrum Master hoặc người có thể gỡ tắc → Chuyển sang Issue khác trong khi chờ.

**Khi phát hiện Bug trong Sprint:**

Nếu Bug thuộc tính năng đang phát triển trong Sprint hiện tại → tạo Bug Issue → ưu tiên sửa trước khi Story được đóng. Nếu Bug thuộc tính năng đã có từ trước → tạo Bug Issue → đưa vào Backlog để Product Owner ưu tiên.

**Khi Issue lớn hơn estimate:**

Cập nhật Remaining Estimate → Comment thông báo nguyên nhân và ước lượng mới → thảo luận với Scrum Master về việc có cần điều chỉnh Sprint không.

---

## 14.7. Giai đoạn 5: Code Review

Code Review là bước kiểm soát chất lượng quan trọng trước khi code được merge vào nhánh chính.

### Quy trình Code Review tích hợp Jira-Git:

```
Developer hoàn thành code
        ↓
Tạo Pull Request trên GitHub/GitLab
Tiêu đề PR: "APP-11: Implement user login"
        ↓
Jira tự động cập nhật Issue: PR đang Open
        ↓
Chuyển Issue sang "Code Review" trên Board
        ↓
Mention Reviewer trong Comment Jira hoặc GitHub
        ↓
Reviewer kiểm tra code
    ├── Approve → PR được merge → Issue sang "Testing"
    └── Request Changes → Developer sửa → Push thêm commit
                              → Reviewer review lại
```

### Tiêu chí Code Review tốt:

- Code đúng logic và đáp ứng yêu cầu của Issue
- Không có security vulnerability rõ ràng
- Code dễ đọc, có comment ở những chỗ phức tạp
- Không có code thừa hoặc debug code bị để lại
- Unit test bao phủ các trường hợp quan trọng

---

## 14.8. Giai đoạn 6: Testing

Sau khi PR được merge, Issue chuyển sang giai đoạn **Testing** — QA kiểm thử toàn diện tính năng trên môi trường Staging.

### Quy trình Testing trong Jira:

**Bước 1:** QA xem thông tin trong Issue: Description, Acceptance Criteria, Sub-task liên quan.

**Bước 2:** QA kiểm tra theo Acceptance Criteria đã định nghĩa.

**Bước 3a — Pass:** Tính năng hoạt động đúng → QA chuyển Issue sang "Done" → Comment xác nhận pass trên môi trường nào, phiên bản nào.

**Bước 3b — Fail:** Phát hiện lỗi → QA tạo Bug Issue mới (không sửa Issue gốc) với đầy đủ Steps to Reproduce → Link Bug với Story gốc → Assign Bug cho Developer → Story gốc giữ nguyên trạng thái "Testing".

### Phân biệt Bug trong Testing và Bug ngoài Production:

| | Bug trong Testing | Bug ngoài Production |
|---|---|---|
| **Phát hiện bởi** | QA trên Staging | Người dùng trên Production |
| **Mức độ ưu tiên** | Thường Medium–High | Thường High–Critical |
| **Xử lý** | Sửa trong Sprint hiện tại nếu liên quan | Có thể cần Hotfix khẩn cấp |

---

## 14.9. Giai đoạn 7: Sprint Review

Vào ngày cuối Sprint, nhóm tổ chức **Sprint Review** để demo kết quả với Product Owner và stakeholder.

### Chuẩn bị Sprint Review trong Jira:

**Bước 1:** Xem **Sprint Report** trong Jira (Reports → Sprint Report → chọn Sprint hiện tại) để nắm danh sách Issue hoàn thành và chưa hoàn thành.

**Bước 2:** Chuẩn bị demo cho từng Story đã Done — không demo Story chưa hoàn thành dù đã gần xong.

**Bước 3:** Chuẩn bị giải thích cho các Issue chưa hoàn thành — nguyên nhân và kế hoạch xử lý.

### Trong buổi Sprint Review:

- Demo từng tính năng đã hoàn thành trực tiếp trên môi trường Staging
- Product Owner và stakeholder cho phản hồi → ghi nhận thành Issue mới trong Backlog nếu cần
- Cập nhật Product Backlog dựa trên phản hồi nhận được

---

## 14.10. Giai đoạn 8: Complete Sprint và Retrospective

### Complete Sprint trong Jira:

**Bước 1:** Vào Board → nhấn **Complete Sprint**.

**Bước 2:** Jira hiển thị tóm tắt: X Issue hoàn thành, Y Issue chưa hoàn thành.

**Bước 3:** Chọn nơi chuyển các Issue chưa hoàn thành (Backlog hoặc Sprint tiếp theo).

**Bước 4:** Nhấn **Complete** để xác nhận. Sprint đóng lại, không thể chỉnh sửa thêm.

### Sprint Retrospective:

Ngay sau khi Sprint kết thúc, nhóm họp Retrospective nội bộ (không có stakeholder). Nhìn vào **Sprint Report** và **Velocity Chart** trong Jira để có dữ liệu thực tế làm cơ sở thảo luận.

**Câu hỏi dựa trên dữ liệu Jira:**

- Velocity Sprint này là bao nhiêu so với các Sprint trước? Xu hướng tăng hay giảm?
- Issue nào chiếm nhiều thời gian hơn estimate? Tại sao?
- Bug nào bị Reopen? Nguyên nhân là gì?
- Bao nhiêu Issue không hoàn thành? Vì sao?

**Kết quả Retrospective:**

Danh sách **action item** cụ thể để cải thiện trong Sprint tới. Mỗi action item nên được tạo thành **Task trong Backlog** để theo dõi — tránh để kết quả Retrospective chỉ tồn tại trong biên bản họp.

---

## 14.11. Quy trình phát triển một Feature hoàn chỉnh

Tổng hợp toàn bộ quy trình cho một Story cụ thể — từ yêu cầu đến Done:

```
Product Owner nhận yêu cầu
        ↓
Tạo Epic (nếu là tính năng lớn)
        ↓
Tạo Story với Summary, Description, Acceptance Criteria
        ↓
Backlog Refinement: estimate Story Point, làm rõ yêu cầu
        ↓
Sprint Planning: kéo Story vào Sprint
        ↓
Developer nhận Story → tạo branch: feature/APP-11-user-login
        ↓
Viết code → commit: "APP-11: Implement login form"
        ↓
Tạo Pull Request: "APP-11: User login feature"
        ↓
Code Review → sửa nếu cần → Approve → Merge
        ↓
Deploy lên Staging
        ↓
QA Testing theo Acceptance Criteria
        ↓
Pass → Story chuyển sang Done
        ↓
Demo trong Sprint Review
        ↓
Deploy lên Production (Release)
```

---

## 14.12. Quy trình Release

**Release** là quá trình đưa tính năng từ môi trường Staging lên môi trường Production — nơi người dùng thực sự sử dụng.

### Quản lý Release trong Jira:

**Jira Versions** là công cụ để nhóm các Issue liên quan đến một phiên bản phát hành cụ thể. Ví dụ: tất cả Story và Bug fix cho phiên bản `v2.5.0` được gán vào Version `2.5.0`.

**Tạo Version trong Jira:**

Project Settings → **Versions** → **Create version** → đặt tên (ví dụ: `v2.5.0`) và ngày phát hành dự kiến.

**Gán Issue vào Version:**

Mở Issue → trường **Fix Version** → chọn Version tương ứng. Khi tất cả Issue trong Version đã Done → tiến hành Release.

**Release Version:**

Project Settings → **Versions** → chọn Version cần release → **Release** → xác nhận ngày phát hành. Jira tạo **Version Report** tổng kết toàn bộ Issue trong phiên bản đó.

---

## 14.13. Xử lý các tình huống ngoại lệ

### Tình huống 1: Phát sinh yêu cầu khẩn cấp giữa Sprint

**Ví dụ:** Khách hàng báo lỗi nghiêm trọng trên Production giữa Sprint đang chạy.

**Xử lý:**
1. Tạo Bug Issue, đặt Priority = Critical/Highest
2. Scrum Master và Product Owner thảo luận: có thực sự cần xử lý ngay trong Sprint này không?
3. Nếu có → đưa Bug vào Sprint, loại bỏ một Issue kém ưu tiên hơn để giữ Capacity cân bằng
4. Nếu không → đưa vào Backlog với ưu tiên cao, xử lý đầu Sprint sau

### Tình huống 2: Sprint không hoàn thành đúng kế hoạch

**Ví dụ:** Kết thúc Sprint còn 30% Issue chưa Done.

**Xử lý:**
1. Complete Sprint trong Jira, chuyển Issue chưa Done về Backlog
2. Không kéo dài Sprint vì lý do tiến độ
3. Retrospective tìm nguyên nhân: estimate sai? Phát sinh ngoài kế hoạch? Dependency?
4. Điều chỉnh cách estimate hoặc Capacity cho Sprint sau

### Tình huống 3: Thành viên nghỉ đột xuất giữa Sprint

**Xử lý:**
1. Xem danh sách Issue của thành viên đó trên Board
2. Scrum Master cùng nhóm ưu tiên lại: Issue nào quan trọng nhất cần tiếp tục?
3. Reassign Issue quan trọng cho thành viên khác
4. Issue kém ưu tiên → chuyển về Backlog

---

## Tóm tắt chương

Quy trình Scrum trên Jira không phải là một loạt các bước máy móc — mà là một vòng lặp liên tục của lập kế hoạch, thực hiện, kiểm tra và cải tiến. Jira là công cụ hỗ trợ vòng lặp đó diễn ra minh bạch, có thể đo lường và liên tục cải thiện qua từng Sprint.

| Giai đoạn | Hoạt động chính trong Jira |
|---|---|
| Backlog | Tạo Epic, Story, Task; sắp xếp ưu tiên |
| Refinement | Làm rõ Issue, estimate Story Point |
| Sprint Planning | Tạo Sprint, chọn Issue, Start Sprint |
| Development | Cập nhật trạng thái Issue, log work, commit có Issue Key |
| Code Review | Tạo PR liên kết Issue, theo dõi trong tab Development |
| Testing | QA kiểm tra, tạo Bug nếu fail, chuyển Done nếu pass |
| Sprint Review | Xem Sprint Report, demo tính năng Done |
| Retrospective | Phân tích Velocity/Burndown, tạo action item |
| Release | Quản lý Version, gán Issue vào Version, Release |

---

*Chương tiếp theo: **Chương 15 — Cấu hình Nâng cao và Tùy chỉnh Jira***