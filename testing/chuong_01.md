# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# CHƯƠNG 1: KIẾN THỨC CƠ BẢN VỀ KIỂM THỬ PHẦN MỀM

---

## 1.1 Khái Niệm Kiểm Thử Phần Mềm

### 1.1.1 QA, QC và Testing — Phân biệt bản chất

Trong ngành công nghiệp phần mềm, ba thuật ngữ **QA (Quality Assurance)**, **QC (Quality Control)** và **Testing** thường bị dùng lẫn lộn. Hiểu đúng bản chất của từng khái niệm là nền tảng để định hình tư duy nghề nghiệp.

#### Quality Assurance (QA) — Đảm bảo chất lượng

QA là tập hợp các **hoạt động có hệ thống** nhằm đảm bảo rằng quy trình phát triển phần mềm được thực hiện đúng cách, từ đó sản phẩm đáp ứng các tiêu chuẩn chất lượng đã định. QA mang tính **phòng ngừa** — nó tập trung vào việc cải thiện quy trình để ngăn lỗi phát sinh ngay từ đầu.

> **Bản chất:** QA là hoạt động về *quy trình* (process-oriented). QA hỏi câu hỏi: *"Chúng ta đang làm đúng cách không?"*

Ví dụ các hoạt động QA:
- Định nghĩa và chuẩn hóa quy trình phát triển (SDLC)
- Thiết lập tiêu chuẩn viết code (coding standards)
- Tổ chức code review, quy trình peer review
- Đánh giá và cải tiến quy trình liên tục (process improvement)

#### Quality Control (QC) — Kiểm soát chất lượng

QC là tập hợp các **hoạt động kiểm tra sản phẩm** nhằm phát hiện các khiếm khuyết trong sản phẩm thực tế. QC mang tính **phát hiện** — nó tập trung vào việc tìm ra lỗi sau khi sản phẩm hoặc một phần sản phẩm đã được tạo ra.

> **Bản chất:** QC là hoạt động về *sản phẩm* (product-oriented). QC hỏi câu hỏi: *"Sản phẩm có đúng không?"*

Ví dụ các hoạt động QC:
- Kiểm tra chức năng của phần mềm
- Xem xét tài liệu, thiết kế (document review)
- Kiểm tra theo checklist
- Testing (kiểm thử)

#### Testing — Kiểm thử

Testing là một **hoạt động cụ thể trong QC** — là quá trình thực thi phần mềm với mục đích tìm ra lỗi và xác minh rằng sản phẩm đáp ứng các yêu cầu đã đặt ra.

> **Bản chất:** Testing là *công cụ thực thi* của QC.

**Bảng so sánh tổng hợp:**

| Tiêu chí | QA | QC | Testing |
|---|---|---|---|
| Mục tiêu | Ngăn lỗi xảy ra | Phát hiện lỗi trong sản phẩm | Thực thi để tìm lỗi |
| Định hướng | Quy trình | Sản phẩm | Sản phẩm |
| Tính chất | Chủ động (Proactive) | Bị động (Reactive) | Bị động (Reactive) |
| Phạm vi | Toàn bộ vòng đời dự án | Giai đoạn sản phẩm đã tạo ra | Giai đoạn thực thi phần mềm |
| Người thực hiện | QA Engineer, toàn team | QC Engineer, Tester | Tester |

---

### 1.1.2 Mục đích và lợi ích của kiểm thử

#### Mục đích của kiểm thử

Kiểm thử phần mềm phục vụ nhiều mục đích vượt xa việc đơn thuần "tìm lỗi":

**1. Phát hiện khiếm khuyết (Defect Detection)**
Mục đích trực tiếp nhất — xác định các điểm khác biệt giữa hành vi thực tế của phần mềm và hành vi mong đợi theo yêu cầu.

**2. Đánh giá chất lượng (Quality Evaluation)**
Cung cấp thông tin khách quan để các bên liên quan ra quyết định: phần mềm có đủ chất lượng để phát hành chưa?

**3. Xây dựng sự tin tưởng (Confidence Building)**
Kiểm thử thành công không chứng minh phần mềm không có lỗi, nhưng xây dựng mức độ tin cậy rằng phần mềm hoạt động đúng trong các điều kiện đã kiểm tra.

**4. Ngăn ngừa khiếm khuyết (Defect Prevention)**
Thông qua quá trình phân tích test case, Tester thường phát hiện vấn đề trong tài liệu yêu cầu và thiết kế trước khi code được viết — đây là hình thức phòng ngừa hiệu quả.

**5. Đáp ứng yêu cầu pháp lý và hợp đồng**
Nhiều lĩnh vực (ngân hàng, y tế, hàng không) yêu cầu kiểm thử nghiêm ngặt theo tiêu chuẩn quy định.

#### Lợi ích kinh tế của kiểm thử

Một trong những lập luận mạnh nhất ủng hộ kiểm thử là **chi phí sửa lỗi tăng lũy thừa theo thời gian phát hiện**.

Nghiên cứu của IBM và nhiều tổ chức phần mềm lớn cho thấy:

| Giai đoạn phát hiện lỗi | Chi phí tương đối |
|---|---|
| Yêu cầu (Requirements) | 1x |
| Thiết kế (Design) | 5x |
| Lập trình (Coding) | 10x |
| Kiểm thử (Testing) | 20x |
| Phát hành (Production) | 100x hoặc hơn |

> **Ý nghĩa thực tế:** Một lỗi logic trong tài liệu yêu cầu, nếu phát hiện ngay, tốn 1 giờ sửa. Nếu phát hiện sau khi hệ thống đã ra production, có thể tốn hàng trăm giờ để điều tra, sửa, kiểm thử lại, phát hành bản vá, và xử lý tác động với khách hàng.

---

### 1.1.3 Tác động của bug đến doanh nghiệp

Bug trong phần mềm không chỉ là vấn đề kỹ thuật — chúng có thể gây hậu quả kinh doanh nghiêm trọng.

#### Các dạng tác động

**Tác động tài chính trực tiếp:**
- Mất doanh thu do hệ thống ngừng hoạt động (downtime)
- Chi phí sửa chữa và tái phát hành
- Bồi thường theo điều khoản hợp đồng SLA (Service Level Agreement)
- Phạt vi phạm quy định (GDPR, PCI-DSS...)

**Tác động uy tín:**
- Mất niềm tin của người dùng
- Tin tức tiêu cực lan truyền nhanh trên mạng xã hội
- Ảnh hưởng lâu dài đến thương hiệu

**Tác động vận hành:**
- Nhân viên tốn thời gian xử lý workaround
- Dữ liệu bị sai lệch dẫn đến quyết định kinh doanh sai
- Ảnh hưởng đến các hệ thống phụ thuộc

#### Ví dụ thực tế

- **Ariane 5 Rocket (1996):** Lỗi chuyển đổi kiểu dữ liệu (float sang integer) khiến tên lửa nổ 40 giây sau khi phóng — thiệt hại 370 triệu USD.
- **Knight Capital (2012):** Lỗi phần mềm giao dịch tự động gây thiệt hại 440 triệu USD trong 45 phút.
- **Therac-25 (1985-1987):** Lỗi trong phần mềm điều khiển máy xạ trị khiến ít nhất 3 bệnh nhân tử vong do bị chiếu liều phóng xạ gấp 100 lần.

---

### 1.1.4 Vòng đời phát triển phần mềm (SDLC)

**SDLC (Software Development Life Cycle)** là quy trình có cấu trúc mô tả các giai đoạn trong quá trình phát triển phần mềm từ khởi đầu đến bảo trì.

#### Các giai đoạn chính của SDLC

```
1. Planning (Lập kế hoạch)
        ↓
2. Requirements Analysis (Phân tích yêu cầu)
        ↓
3. System Design (Thiết kế hệ thống)
        ↓
4. Implementation / Coding (Lập trình)
        ↓
5. Testing (Kiểm thử)
        ↓
6. Deployment (Triển khai)
        ↓
7. Maintenance (Bảo trì)
```

**Giai đoạn 1 — Planning:** Xác định phạm vi dự án, khả thi về kỹ thuật và kinh tế, nguồn lực, timeline.

**Giai đoạn 2 — Requirements Analysis:** Thu thập và tài liệu hóa yêu cầu từ stakeholder. Đây là giai đoạn mà Tester cần tham gia sớm để phát hiện mâu thuẫn, mơ hồ trong yêu cầu.

**Giai đoạn 3 — System Design:** Kiến trúc hệ thống, thiết kế database, thiết kế UI/UX. Tester xem xét tài liệu thiết kế để lập kế hoạch kiểm thử.

**Giai đoạn 4 — Implementation:** Developer viết code. Tester chuẩn bị test case, test data, môi trường kiểm thử.

**Giai đoạn 5 — Testing:** Thực thi kiểm thử, báo cáo lỗi, theo dõi sửa lỗi, kiểm thử hồi quy.

**Giai đoạn 6 — Deployment:** Phát hành ra môi trường production. Tester thực hiện smoke test sau deployment.

**Giai đoạn 7 — Maintenance:** Sửa lỗi phát hiện sau khi phát hành, phát triển tính năng mới — mỗi lần lại bắt đầu một chu trình SDLC mới.

#### Các mô hình SDLC phổ biến

| Mô hình | Đặc điểm | Phù hợp |
|---|---|---|
| **Waterfall** | Tuần tự, từng giai đoạn hoàn thành trước khi sang giai đoạn tiếp | Dự án yêu cầu rõ ràng, ít thay đổi |
| **Agile** | Lặp lại (iterative), phân phối theo sprint, thích nghi thay đổi | Dự án yêu cầu thay đổi thường xuyên |
| **Scrum** | Agile framework cụ thể với Sprint 2-4 tuần, roles rõ ràng | Đội nhỏ, sản phẩm phức tạp |
| **Kanban** | Luồng công việc liên tục, không sprint cố định | Dự án bảo trì, support |
| **V-Model** | Mỗi giai đoạn phát triển tương ứng với giai đoạn kiểm thử | Dự án yêu cầu kiểm thử nghiêm ngặt |

---

## 1.2 Nguyên Tắc Kiểm Thử

### 1.2.1 Bảy nguyên tắc kiểm thử cơ bản

Bảy nguyên tắc này được ISTQB (International Software Testing Qualifications Board) xác lập, là nền tảng tư duy cho mọi Tester chuyên nghiệp.

---

**Nguyên tắc 1: Kiểm thử cho thấy sự hiện diện của lỗi, không phải sự vắng mặt**
*(Testing shows the presence of defects, not their absence)*

Dù kiểm thử có kỹ đến đâu, kết quả "pass" chỉ có nghĩa là *trong điều kiện đã kiểm thử, phần mềm hoạt động đúng* — không có nghĩa là phần mềm không còn lỗi nào. Không bao giờ có thể kiểm thử 100% mọi tình huống có thể xảy ra.

> **Ý nghĩa thực hành:** Đừng nói "phần mềm không có lỗi". Hãy nói "phần mềm đã vượt qua các test case được thực thi".

---

**Nguyên tắc 2: Kiểm thử toàn diện là không thể**
*(Exhaustive testing is impossible)*

Số lượng tổ hợp đầu vào, trạng thái hệ thống, và đường thực thi có thể là vô hạn. Một form đơn giản với 10 trường, mỗi trường có 100 giá trị hợp lệ tạo ra 100^10 tổ hợp — con số lớn hơn số nguyên tử trong vũ trụ.

> **Ý nghĩa thực hành:** Tester phải **ưu tiên** và **chọn lọc** — dùng kỹ thuật thiết kế test case để đạt hiệu quả tối đa với nguồn lực có hạn.

---

**Nguyên tắc 3: Kiểm thử sớm tiết kiệm chi phí**
*(Early testing saves time and money)*

Lỗi phát hiện càng sớm trong SDLC, chi phí sửa càng thấp (như đã phân tích ở mục 1.1.2). Tester nên tham gia ngay từ giai đoạn phân tích yêu cầu, không chờ đến khi có sản phẩm.

> **Ý nghĩa thực hành:** Review tài liệu yêu cầu, tham gia buổi grooming, đặt câu hỏi sớm — đây là những hoạt động kiểm thử có giá trị cao.

---

**Nguyên tắc 4: Lỗi có xu hướng tập trung**
*(Defects cluster together)*

Thực nghiệm cho thấy 80% lỗi thường tập trung ở 20% module của phần mềm (nguyên lý Pareto). Một module phức tạp, mới viết, hoặc do lập trình viên thiếu kinh nghiệm viết thường chứa nhiều lỗi hơn.

> **Ý nghĩa thực hành:** Xác định và tập trung kiểm thử kỹ hơn vào các "vùng rủi ro cao" — module mới, module phức tạp, module thay đổi nhiều.

---

**Nguyên tắc 5: Thuốc lặp lại mất tác dụng**
*(Beware of the pesticide paradox)*

Nếu cùng một bộ test case được lặp đi lặp lại nhiều lần, chúng sẽ không còn tìm ra lỗi mới nữa. Hệ thống "miễn dịch" với các test case cũ vì các lỗi cũ đã được sửa, trong khi lỗi mới có thể tồn tại ở những nơi chưa được kiểm thử.

> **Ý nghĩa thực hành:** Cập nhật và mở rộng test suite định kỳ. Kết hợp exploratory testing với test case có sẵn để phát hiện lỗi ở "góc khuất".

---

**Nguyên tắc 6: Kiểm thử phụ thuộc vào ngữ cảnh**
*(Testing is context dependent)*

Cách kiểm thử một ứng dụng ngân hàng online khác hoàn toàn với cách kiểm thử một game mobile. Tiêu chuẩn chất lượng, mức độ kiểm thử, và kỹ thuật kiểm thử phải phù hợp với đặc thù của từng dự án.

> **Ý nghĩa thực hành:** Không áp dụng máy móc một quy trình kiểm thử cho mọi dự án. Phải hiểu bối cảnh nghiệp vụ và rủi ro để thiết kế chiến lược kiểm thử phù hợp.

---

**Nguyên tắc 7: Không có lỗi không có nghĩa là thành công**
*(Absence of errors fallacy)*

Phần mềm không có lỗi kỹ thuật nhưng không đáp ứng nhu cầu thực sự của người dùng vẫn là thất bại. Chất lượng không chỉ là "không có bug".

> **Ý nghĩa thực hành:** Luôn đối chiếu với yêu cầu nghiệp vụ và trải nghiệm người dùng thực tế, không chỉ với đặc tả kỹ thuật.

---

### 1.2.2 Error, Defect và Failure — Phân biệt chính xác

Ba khái niệm này mô tả ba giai đoạn trong chuỗi nguyên nhân — hệ quả của một sự cố phần mềm.

#### Error (Lỗi do con người)

**Error** là hành động sai của con người dẫn đến việc tạo ra một khiếm khuyết. Error là **nguyên nhân gốc rễ**, thường xảy ra trong đầu người.

Ví dụ: Lập trình viên hiểu nhầm yêu cầu — nghĩ rằng số tuổi tối đa là 99 nhưng thực ra là 120. Sự hiểu nhầm đó là error.

#### Defect (Khiếm khuyết / Bug)

**Defect** là biểu hiện cụ thể của error trong sản phẩm — có thể là trong code, tài liệu thiết kế, hoặc tài liệu yêu cầu. Defect là thứ Tester **phát hiện và báo cáo**.

Ví dụ: Từ error trên, lập trình viên viết điều kiện kiểm tra `if (age > 99)` thay vì `if (age > 120)`. Đoạn code sai này là defect.

#### Failure (Sự cố)

**Failure** xảy ra khi defect được thực thi và gây ra hành vi sai lệch mà **người dùng có thể quan sát thấy**. Failure là biểu hiện của defect trong thực tế vận hành.

Ví dụ: Khi người dùng 105 tuổi điền thông tin, hệ thống báo lỗi "Tuổi không hợp lệ" — đó là failure.

**Quan hệ nhân quả:**
```
Error (con người nhầm)
    → Defect (code/tài liệu sai)
        → Failure (hệ thống hoạt động sai)
```

> **Lưu ý quan trọng:** Không phải mọi defect đều gây ra failure — một defect trong đoạn code chưa bao giờ được thực thi sẽ không tạo ra failure. Và failure đôi khi có thể xảy ra mà không có defect trong code — ví dụ do lỗi môi trường, phần cứng, hoặc dữ liệu đầu vào bất thường.

---

### 1.2.3 Mô hình V (V-Model)

Mô hình V là mô hình SDLC mở rộng từ Waterfall, biểu diễn mối quan hệ trực tiếp giữa mỗi giai đoạn phát triển và giai đoạn kiểm thử tương ứng. Hình dạng chữ V thể hiện: nhánh trái là phát triển (từ yêu cầu xuống code), nhánh phải là kiểm thử (từ unit test lên acceptance test).

```
Requirements Analysis  ←────────────→  Acceptance Testing (UAT)
        ↓                                        ↑
  System Design        ←────────────→  System Testing
        ↓                                        ↑
  Architecture Design  ←────────────→  Integration Testing
        ↓                                        ↑
    Module Design      ←────────────→  Unit Testing
        ↓                                        ↑
         └──────────── Implementation ───────────┘
```

**Ý nghĩa của từng mối quan hệ:**

- **Requirements ↔ UAT:** Acceptance test được thiết kế từ tài liệu yêu cầu — kiểm tra xem hệ thống có đáp ứng yêu cầu nghiệp vụ không.
- **System Design ↔ System Testing:** System test được thiết kế từ tài liệu thiết kế hệ thống — kiểm tra toàn bộ hệ thống tích hợp.
- **Architecture Design ↔ Integration Testing:** Integration test kiểm tra sự tương tác giữa các module theo kiến trúc đã thiết kế.
- **Module Design ↔ Unit Testing:** Unit test kiểm tra từng module nhỏ nhất theo thiết kế chi tiết.

**Điểm mạnh của V-Model:**
- Mỗi giai đoạn phát triển có giai đoạn kiểm thử tương ứng rõ ràng
- Kế hoạch kiểm thử được xây dựng sớm, song song với phát triển
- Phù hợp với dự án có yêu cầu rõ ràng, ít thay đổi

**Hạn chế:**
- Kém linh hoạt khi yêu cầu thay đổi
- Không phù hợp với dự án Agile

---

### 1.2.4 Test Pyramid

Test Pyramid là mô hình trực quan hóa phân phối lý tưởng của các loại test trong một dự án phần mềm, được Mike Cohn giới thiệu trong cuốn *Succeeding with Agile*.

```
           /\
          /  \
         / E2E\          ← Ít test, chạy chậm, chi phí cao
        /──────\
       /        \
      /Integration\      ← Vừa phải
     /────────────\
    /              \
   /   Unit Tests   \    ← Nhiều test, chạy nhanh, chi phí thấp
  /──────────────────\
```

**Tầng đáy — Unit Tests:**
- Số lượng: nhiều nhất (60-70%)
- Tốc độ: nhanh (mili-giây mỗi test)
- Chi phí duy trì: thấp
- Phạm vi: một hàm/method đơn lẻ

**Tầng giữa — Integration Tests:**
- Số lượng: vừa phải (20-30%)
- Tốc độ: trung bình (giây)
- Phạm vi: sự tương tác giữa các component

**Tầng đỉnh — E2E Tests:**
- Số lượng: ít nhất (5-10%)
- Tốc độ: chậm (phút)
- Chi phí duy trì: cao (hay bị "flaky")
- Phạm vi: toàn bộ luồng từ UI đến database

> **Anti-pattern — Ice Cream Cone:** Nhiều dự án thực tế có hình dạng ngược lại — nhiều manual test và E2E test, ít unit test. Đây là dấu hiệu của quy trình kiểm thử kém hiệu quả: chậm, tốn kém, và dễ bị lỗi do môi trường.

---

## 1.3 Cấp Độ Kiểm Thử (Test Level)

Các cấp độ kiểm thử phân chia hoạt động kiểm thử theo phạm vi — từ nhỏ đến lớn, từ chi tiết đến tổng thể.

### 1.3.1 Unit Testing — Kiểm thử đơn vị

**Định nghĩa:** Unit testing là kiểm thử các đơn vị nhỏ nhất của phần mềm — thường là một hàm (function), một phương thức (method), hoặc một class — một cách độc lập, cô lập với phần còn lại của hệ thống.

**Bản chất:** Unit test kiểm tra **logic** của một đơn vị code trong môi trường được kiểm soát hoàn toàn. Mọi phụ thuộc bên ngoài (database, API, file system) đều được thay thế bằng mock/stub để đảm bảo tính cô lập.

**Người thực hiện:** Chủ yếu là Developer. Trong một số tổ chức áp dụng TDD, Developer viết unit test trước khi viết code.

**Ví dụ:** Kiểm thử hàm tính tổng giá trị giỏ hàng sau khi áp dụng mã giảm giá:
```python
def apply_discount(total: float, discount_percent: float) -> float:
    if discount_percent < 0 or discount_percent > 100:
        raise ValueError("Discount must be between 0 and 100")
    return total * (1 - discount_percent / 100)

# Unit test
def test_apply_discount_normal():
    assert apply_discount(100, 20) == 80.0

def test_apply_discount_invalid():
    with pytest.raises(ValueError):
        apply_discount(100, 150)
```

---

### 1.3.2 Integration Testing — Kiểm thử tích hợp

**Định nghĩa:** Integration testing kiểm tra sự tương tác và giao tiếp giữa các module, component, hoặc service đã được unit test riêng lẻ khi chúng được kết hợp lại với nhau.

**Bản chất:** Hai module hoạt động đúng độc lập không đảm bảo chúng hoạt động đúng khi tích hợp — lỗi có thể nằm ở **giao diện** (interface) giữa chúng: định dạng dữ liệu truyền, thứ tự gọi hàm, xử lý lỗi giữa các tầng.

**Ví dụ điển hình:** Tester gọi API `POST /orders` và kiểm tra không chỉ response trả về đúng mà còn verify trong database rằng đơn hàng thực sự được tạo ra với đúng dữ liệu.

**Các chiến lược tích hợp:**
- **Big Bang:** Tích hợp tất cả cùng lúc — khó xác định nguồn gốc lỗi
- **Top-down:** Tích hợp từ module cấp cao xuống, dùng stub cho module cấp thấp
- **Bottom-up:** Tích hợp từ module cấp thấp lên, dùng driver cho module cấp cao
- **Incremental:** Tích hợp từng phần nhỏ, phổ biến nhất trong Agile

---

### 1.3.3 System Testing — Kiểm thử hệ thống

**Định nghĩa:** System testing kiểm tra toàn bộ hệ thống đã được tích hợp đầy đủ, đánh giá xem hệ thống có đáp ứng các yêu cầu đã đặc tả hay không.

**Bản chất:** Đây là cấp độ kiểm thử "hộp đen" — Tester không cần biết chi tiết kỹ thuật bên trong, chỉ tương tác với hệ thống như người dùng thông qua giao diện chính thức (UI, API).

**Phạm vi:** Bao gồm cả functional testing (chức năng) và non-functional testing (hiệu suất, bảo mật, khả năng sử dụng).

**Môi trường:** System test phải được thực hiện trên môi trường gần giống production nhất có thể (staging environment).

---

### 1.3.4 Acceptance Testing — Kiểm thử chấp nhận

**Định nghĩa:** Acceptance testing xác định xem hệ thống có đáp ứng các tiêu chí chấp nhận (acceptance criteria) của khách hàng hay không, và liệu hệ thống có sẵn sàng để phát hành hay không.

**Bản chất:** Đây là "phiên tòa" cuối cùng trước khi phần mềm đến tay người dùng thực sự. Không phải Tester mà là **khách hàng hoặc người dùng cuối** thực hiện.

**Các dạng Acceptance Testing:**
- **UAT (User Acceptance Testing):** Người dùng thực tế kiểm tra trong môi trường gần giống production
- **Alpha Testing:** Người dùng nội bộ hoặc nhóm nhỏ kiểm tra trong môi trường của nhà phát triển
- **Beta Testing:** Phát hành giới hạn cho nhóm người dùng thực để thu thập phản hồi
- **Contract Acceptance Testing:** Kiểm tra theo điều khoản hợp đồng ký với khách hàng

---

### 1.3.5 Component Testing

**Định nghĩa:** Component testing kiểm tra các component phần mềm riêng lẻ sau khi chúng được tích hợp nội bộ nhưng trước khi tích hợp với phần còn lại của hệ thống.

**So sánh với Unit Testing:** Unit test kiểm tra một hàm đơn lẻ; Component test kiểm tra một component hoàn chỉnh (ví dụ: module xác thực người dùng gồm nhiều class và hàm phối hợp với nhau).

---

### 1.3.6 End-to-End Testing (E2E)

**Định nghĩa:** E2E testing kiểm tra toàn bộ luồng nghiệp vụ từ điểm đầu đến điểm cuối, mô phỏng chính xác hành vi của người dùng thực sự trong môi trường gần production.

**Bản chất:** E2E test đi qua tất cả các tầng của ứng dụng: UI → Backend API → Database → (có thể) External Services. Nó xác minh rằng toàn bộ hệ thống hoạt động đồng bộ như một thể thống nhất.

**Ví dụ luồng E2E cho tính năng mua hàng:**
```
User mở trình duyệt
    → Vào trang sản phẩm
    → Thêm sản phẩm vào giỏ hàng
    → Điền thông tin giao hàng
    → Chọn phương thức thanh toán
    → Xác nhận đặt hàng
    → Nhận email xác nhận
    → Verify trong DB: đơn hàng được tạo, tồn kho giảm, email được gửi
```

---

### 1.3.7 Mối quan hệ giữa các cấp độ

| Cấp độ | Phạm vi | Tốc độ | Chi phí | Người thực hiện |
|---|---|---|---|---|
| Unit | 1 hàm/method | Mili-giây | Thấp | Developer |
| Component | 1 module | Giây | Thấp-Trung bình | Developer/Tester |
| Integration | Nhiều module tương tác | Giây-Phút | Trung bình | Developer/Tester |
| System | Toàn bộ hệ thống | Phút-Giờ | Cao | Tester |
| E2E | Luồng nghiệp vụ đầy đủ | Phút-Giờ | Cao | Tester/QA |
| Acceptance | Yêu cầu nghiệp vụ | Ngày-Tuần | Rất cao | Khách hàng/Người dùng |

---

## 1.4 Loại Kiểm Thử (Test Type)

Nếu Test Level phân chia theo *phạm vi*, thì Test Type phân chia theo *mục tiêu* của kiểm thử.

### 1.4.1 Functional Testing — Kiểm thử chức năng

**Định nghĩa:** Functional testing kiểm tra xem phần mềm có thực hiện đúng các chức năng theo yêu cầu đã đặc tả hay không. Nó trả lời câu hỏi: *"Phần mềm làm đúng những gì cần làm không?"*

**Bản chất:** Functional testing là kiểm thử "hộp đen" — Tester đưa đầu vào và kiểm tra đầu ra, không quan tâm đến cách triển khai bên trong. Dựa hoàn toàn vào tài liệu yêu cầu (SRS, User Stories, Acceptance Criteria).

**Các dạng Functional Testing:**

- **Kiểm thử chức năng cơ bản:** Xác minh từng chức năng hoạt động đúng theo yêu cầu. Ví dụ: Form đăng nhập với email và password hợp lệ phải đăng nhập thành công.

- **Kiểm thử nghiệp vụ (Business Logic Testing):** Kiểm tra các quy tắc nghiệp vụ phức tạp. Ví dụ: Đơn hàng trên 500,000 VNĐ được giảm 10%, nhưng không áp dụng cho danh mục "Khuyến mãi đặc biệt".

- **Kiểm thử input/output:** Kiểm tra tất cả các loại đầu vào (hợp lệ, không hợp lệ, biên) và đầu ra tương ứng.

---

### 1.4.2 Non-Functional Testing — Kiểm thử phi chức năng

Non-functional testing kiểm tra *cách* phần mềm hoạt động, không phải *những gì* nó làm. Nó trả lời câu hỏi: *"Phần mềm hoạt động tốt như thế nào?"*

#### Performance Testing — Kiểm thử hiệu suất
Đánh giá tốc độ, thời gian phản hồi, và khả năng xử lý của hệ thống. Ví dụ: API lấy danh sách sản phẩm phải trả kết quả trong dưới 300ms.

#### Load Testing — Kiểm thử tải
Kiểm tra hệ thống dưới tải người dùng dự kiến. Ví dụ: Hệ thống phải xử lý được 1,000 người dùng đồng thời mà không suy giảm hiệu suất.

#### Stress Testing — Kiểm thử căng thẳng
Đẩy hệ thống vượt quá giới hạn bình thường để tìm điểm gãy. Ví dụ: Tăng dần tải lên 5,000 người dùng để xem hệ thống xử lý thế nào và có phục hồi được khi tải giảm không.

#### Security Testing — Kiểm thử bảo mật
Phát hiện các lỗ hổng bảo mật: SQL Injection, XSS, CSRF, xác thực yếu, phân quyền sai.

#### Usability Testing — Kiểm thử khả năng sử dụng
Đánh giá mức độ dễ sử dụng của phần mềm từ góc độ người dùng thực sự. Bao gồm tính trực quan, độ dễ học, hiệu quả sử dụng.

#### Compatibility Testing — Kiểm thử tương thích
Kiểm tra phần mềm hoạt động đúng trên các trình duyệt, hệ điều hành, thiết bị, và phiên bản khác nhau.

#### Accessibility Testing — Kiểm thử khả năng tiếp cận
Đảm bảo phần mềm có thể sử dụng được bởi người dùng khuyết tật (mù màu, khiếm thị, khó nghe...). Tuân theo tiêu chuẩn WCAG.

#### Reliability Testing — Kiểm thử độ tin cậy
Kiểm tra hệ thống có hoạt động ổn định và nhất quán trong thời gian dài không. Đo MTBF (Mean Time Between Failures) và MTTR (Mean Time To Recovery).

---

### 1.4.3 Regression Testing — Kiểm thử hồi quy

**Định nghĩa:** Regression testing là kiểm thử lại các chức năng đã hoạt động đúng trước đây để đảm bảo rằng các thay đổi mới (sửa lỗi, thêm tính năng, cập nhật) không vô tình gây ra lỗi mới trong các phần đó.

**Bản chất:** Phần mềm là hệ thống phức tạp với nhiều phụ thuộc lẫn nhau. Thay đổi một phần có thể gây hiệu ứng domino — ảnh hưởng đến những phần tưởng như không liên quan. Regression testing là lưới an toàn để bắt những ảnh hưởng không mong muốn này.

**Khi nào cần Regression Testing:**
- Sau khi sửa bất kỳ bug nào
- Sau khi thêm tính năng mới
- Sau khi thay đổi cấu hình hoặc môi trường
- Trước mỗi lần phát hành
- Sau khi tích hợp component của bên thứ ba

**Regression Test Suite:**
Không phải toàn bộ test case đều cần chạy mỗi lần regression. Xây dựng **Regression Test Suite** là tập hợp test case được chọn lọc:
- Test case cho các chức năng cốt lõi (critical path)
- Test case cho các chức năng liên quan đến thay đổi
- Test case cho các lỗi đã từng xảy ra (để không tái phát)

**Regression trong Agile:**
Trong môi trường Agile với các sprint 2 tuần, regression test được chạy liên tục — thường được tự động hóa và tích hợp vào CI/CD pipeline. Mỗi khi có commit mới, hệ thống tự động chạy regression test suite.

---

### 1.4.4 Smoke Testing

**Định nghĩa:** Smoke testing là kiểm thử nhanh và sơ bộ để xác định xem bản build mới có đủ ổn định để tiến hành kiểm thử chi tiết hơn hay không.

**Nguồn gốc tên gọi:** Thuật ngữ bắt nguồn từ ngành điện tử — khi lắp ráp xong mạch điện, thử nghiệm đơn giản nhất là cấp điện và xem liệu có "bốc khói" không. Nếu không khói, thiết bị có thể hoạt động cơ bản.

**Đặc điểm:**
- Chỉ kiểm tra các chức năng quan trọng nhất
- Thời gian thực hiện ngắn (15-30 phút)
- Mục tiêu: Go/No-Go — tiếp tục kiểm thử hay từ chối build này

**Ví dụ Smoke Test cho ứng dụng thương mại điện tử:**
- Ứng dụng khởi động thành công không?
- Trang chủ load được không?
- Đăng nhập được không?
- Tìm kiếm sản phẩm được không?
- Thêm vào giỏ hàng được không?

Nếu một trong các bước trên thất bại → **Reject build**, trả lại Developer mà không cần kiểm thử thêm.

---

### 1.4.5 Sanity Testing

**Định nghĩa:** Sanity testing là kiểm thử hẹp và sâu vào một chức năng cụ thể sau khi bug đã được sửa, để xác minh rằng sửa chữa đó hoạt động đúng trước khi thực hiện kiểm thử đầy đủ hơn.

**Phân biệt Smoke vs Sanity:**

| Tiêu chí | Smoke Testing | Sanity Testing |
|---|---|---|
| Phạm vi | Rộng — toàn bộ ứng dụng | Hẹp — một chức năng cụ thể |
| Độ sâu | Sơ lược | Sâu hơn |
| Thực hiện khi | Có build mới | Sau khi sửa bug cụ thể |
| Mục tiêu | Build có ổn định không? | Bug này đã được sửa đúng chưa? |
| Tài liệu | Thường không có script | Thường không có script (exploratory) |

**Ví dụ Sanity Test:** Bug được báo cáo: "Không thể áp dụng mã giảm giá khi đơn hàng có sản phẩm giảm giá sẵn". Developer sửa xong. Sanity test: kiểm tra chính xác tình huống này với nhiều tổ hợp — mã giảm 10% + sản phẩm sale 20%, mã giảm 50% + sản phẩm thường, v.v.

---

### 1.4.6 Retesting

**Định nghĩa:** Retesting là thực thi lại chính xác test case đã phát hiện bug, sau khi bug đó được báo cáo là đã sửa, để xác nhận rằng bug không còn tồn tại nữa.

**Phân biệt Retesting vs Regression Testing:**

| Tiêu chí | Retesting | Regression Testing |
|---|---|---|
| Mục tiêu | Xác nhận bug cụ thể đã được sửa | Đảm bảo không có bug mới phát sinh |
| Phạm vi | Đúng test case đã fail | Tập hợp test case rộng hơn |
| Thực hiện khi | Sau khi developer fix bug | Sau bất kỳ thay đổi nào |
| Tính chất | Kiểm thử xác nhận | Kiểm thử phòng ngừa |

> **Quy trình thực tế:** Khi developer báo fix một bug → Tester thực hiện retest (chạy lại đúng test case đó). Nếu pass → chuyển sang regression test các chức năng liên quan. Nếu fail → reopen bug, ghi chú và trả lại developer.

---

### 1.4.7 Exploratory Testing

**Định nghĩa:** Exploratory testing là phương pháp kiểm thử trong đó Tester đồng thời thiết kế, thực thi, và học hỏi về hệ thống — không có script định sẵn, dựa vào kiến thức, kinh nghiệm, và sự sáng tạo của Tester.

**Bản chất:** Exploratory testing không phải là kiểm thử ngẫu nhiên. Nó là kiểm thử có định hướng nhưng linh hoạt — Tester đặt ra một "charter" (mục tiêu khám phá) và tự do tìm kiếm trong phạm vi đó.

**Khi nào dùng Exploratory Testing:**
- Khi không có tài liệu yêu cầu đầy đủ
- Khi cần tìm lỗi ngoài những gì đã biết
- Khi kiểm thử chức năng mới lần đầu
- Khi có ít thời gian và cần hiệu quả cao nhanh
- Để bổ sung cho scripted testing

**Kỹ thuật Exploratory Testing:**
- **Session-based:** Kiểm thử trong các phiên có thời hạn (ví dụ 90 phút), mỗi phiên có charter rõ ràng
- **Persona-based:** Khám phá từ góc độ của một người dùng cụ thể (người mua hàng lần đầu, admin, người dùng cao tuổi)
- **Risk-based:** Tập trung khám phá các vùng rủi ro cao

**Ví dụ Charter:** *"Khám phá tính năng thanh toán, tập trung vào các edge case của mã giảm giá và tích hợp payment gateway trong 90 phút"*

---

### 1.4.8 Ad-hoc Testing

**Định nghĩa:** Ad-hoc testing là kiểm thử không có kế hoạch, không có tài liệu, dựa hoàn toàn vào trực giác và kinh nghiệm của Tester.

**Phân biệt Ad-hoc vs Exploratory:**

| Tiêu chí | Ad-hoc Testing | Exploratory Testing |
|---|---|---|
| Kế hoạch | Không có | Có charter/mục tiêu |
| Tài liệu | Không | Có session notes |
| Định hướng | Tùy hứng | Có định hướng |
| Tái tạo | Khó | Có thể tái tạo từ notes |
| Giá trị | Tìm lỗi ngẫu nhiên | Học hỏi có hệ thống |

Ad-hoc testing có giá trị nhất khi được thực hiện bởi Tester có kinh nghiệm sâu về domain nghiệp vụ.

---

## 1.5 Testing trong Agile / Scrum

### 1.5.1 Vai trò Tester trong Scrum

Trong Scrum, không có role "Tester" chính thức — chỉ có **Development Team** gồm các thành viên đa năng. Tuy nhiên, trong thực tế, nhiều team vẫn có thành viên chuyên về testing.

**Trách nhiệm của Tester trong Scrum:**
- Tham gia Sprint Planning để hiểu User Story và đặt câu hỏi về acceptance criteria
- Làm rõ tiêu chí chấp nhận (Acceptance Criteria) cùng với BA/PO
- Kiểm thử trong sprint — không chờ đến cuối sprint
- Tham gia Daily Standup: báo cáo tiến độ và blocker
- Tham gia Sprint Review: demo và nhận feedback
- Tham gia Sprint Retrospective: cải tiến quy trình

---

### 1.5.2 Definition of Done (DoD)

**Definition of Done** là danh sách các tiêu chí mà một User Story hoặc Sprint phải đáp ứng để được coi là "hoàn thành". DoD giúp đảm bảo chất lượng và tạo sự đồng thuận trong team.

**Ví dụ DoD điển hình của một team:**
```
Một User Story được coi là Done khi:
□ Code đã được viết và merge vào nhánh chính
□ Unit test đã được viết và pass (coverage ≥ 80%)
□ Code review đã được thực hiện bởi ít nhất 1 người
□ Acceptance criteria đã được kiểm thử và pass
□ Regression test liên quan đã pass
□ Không có bug mức độ Critical/High còn mở
□ Tài liệu kỹ thuật đã được cập nhật (nếu cần)
□ Performance không suy giảm so với baseline
```

---

### 1.5.3 Testing trong Sprint Cycle

```
Sprint Planning
    ↓
    Tester đọc User Story
    Tester đặt câu hỏi: edge case? error case? acceptance criteria?
    Tester ước tính effort kiểm thử
    ↓
Sprint Execution (2 tuần)
    ↓
    Day 1-3:  Developer viết code
              Tester viết test case, chuẩn bị test data
    ↓
    Day 4-8:  Developer hoàn thành feature đầu tiên
              Tester bắt đầu kiểm thử ngay (không chờ cuối sprint)
              Phát hiện bug → Developer sửa ngay trong sprint
    ↓
    Day 9-10: Regression test
              UAT nếu cần
              Smoke test trên staging
              Chuẩn bị Sprint Review
    ↓
Sprint Review  → Demo → Nhận feedback
Sprint Retrospective → Cải tiến quy trình
```

**Nguyên tắc quan trọng:** Trong Agile, kiểm thử không phải là giai đoạn cuối mà là **hoạt động liên tục** diễn ra song song với phát triển. Tester không ngồi chờ toàn bộ sprint kết thúc mới bắt đầu kiểm thử.

---

### 1.5.4 Trao đổi trong Agile Team — The Three Amigos

**Three Amigos** là kỹ thuật trong Agile, trong đó ba vai trò — **BA/PO (Business)**, **Developer (Technology)**, và **Tester (Quality)** — cùng ngồi lại để thảo luận về một User Story trước khi phát triển.

**Mục tiêu của Three Amigos:**
- Đảm bảo tất cả hiểu đúng yêu cầu
- Phát hiện sự mơ hồ và mâu thuẫn sớm
- BA/PO cung cấp ngữ cảnh nghiệp vụ
- Developer đề xuất hướng kỹ thuật
- Tester đặt câu hỏi về edge case và điều kiện lỗi

**Ví dụ Three Amigos cho User Story "Người dùng có thể đặt hàng":**

- **BA/PO:** "Người dùng chọn sản phẩm, điền địa chỉ, chọn thanh toán, xác nhận đặt hàng."
- **Developer:** "Tôi cần biết: timeout là bao lâu? Có giới hạn số sản phẩm không?"
- **Tester:** "Điều gì xảy ra nếu sản phẩm hết hàng ngay khi đang thanh toán? Nếu payment gateway timeout thì đơn hàng có được tạo không? Người dùng chưa đăng nhập có thể đặt hàng không?"

---

