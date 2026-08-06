# Chương 4: Database

## Giới thiệu

Cơ sở dữ liệu (Database) là thành phần lưu trữ và quản lý dữ liệu của hệ thống backend. Việc hiểu đúng cách một cơ sở dữ liệu xử lý giao dịch, đảm bảo tính nhất quán, tối ưu truy vấn và mở rộng quy mô là nền tảng bắt buộc đối với bất kỳ lập trình viên backend nào, bất kể sử dụng framework hay ORM nào. Chương này trình bày các khái niệm cốt lõi: Transaction, Isolation Level, Locking, Index, Scaling Database và ORM.

---

## 4.1. Transaction và ACID

### 4.1.1. Transaction là gì?

**Transaction (giao dịch)** là một chuỗi các thao tác đọc/ghi dữ liệu được nhóm lại và xử lý như một đơn vị duy nhất, không thể chia nhỏ. Toàn bộ các thao tác trong một transaction hoặc được thực hiện thành công tất cả, hoặc không thao tác nào được thực hiện.

Ví dụ kinh điển: chuyển tiền giữa hai tài khoản ngân hàng gồm hai bước — trừ tiền tài khoản A và cộng tiền tài khoản B. Nếu hệ thống gặp sự cố sau khi trừ tiền A nhưng chưa kịp cộng tiền B, dữ liệu sẽ sai lệch nếu không dùng transaction.

Các lệnh cơ bản khi làm việc với transaction:

| Lệnh | Ý nghĩa |
|---|---|
| `BEGIN` / `START TRANSACTION` | Bắt đầu một giao dịch |
| `COMMIT` | Xác nhận, lưu vĩnh viễn các thay đổi |
| `ROLLBACK` | Hủy bỏ toàn bộ thay đổi trong giao dịch |
| `SAVEPOINT` | Đánh dấu một điểm khôi phục trung gian trong giao dịch |

### 4.1.2. Tính chất ACID

ACID là bốn tính chất đảm bảo một transaction được xử lý an toàn và đáng tin cậy:

| Tính chất | Ý nghĩa |
|---|---|
| **Atomicity** (Tính nguyên tử) | Tất cả thao tác trong transaction thành công hoàn toàn hoặc thất bại hoàn toàn, không có trạng thái "làm dở". |
| **Consistency** (Tính nhất quán) | Dữ liệu luôn chuyển từ trạng thái hợp lệ này sang trạng thái hợp lệ khác, tuân thủ mọi ràng buộc (constraint, khóa ngoại...). |
| **Isolation** (Tính cô lập) | Các transaction chạy đồng thời không ảnh hưởng lẫn nhau như thể chúng chạy tuần tự. |
| **Durability** (Tính bền vững) | Sau khi commit, dữ liệu được lưu vĩnh viễn, kể cả khi hệ thống gặp sự cố ngay sau đó. |

> ACID là nền tảng lý thuyết, còn mức độ *Isolation* thực tế được cấu hình linh hoạt thông qua **Isolation Level**, trình bày ở phần tiếp theo.

---

## 4.2. Isolation Level và Locking

### 4.2.1. Isolation Level

Khi nhiều transaction chạy đồng thời, việc cô lập tuyệt đối (mỗi transaction chạy hoàn toàn độc lập) sẽ rất tốn hiệu năng. Vì vậy các hệ quản trị cơ sở dữ liệu (DBMS) cho phép cấu hình **mức độ cô lập (Isolation Level)** để cân bằng giữa hiệu năng và tính nhất quán.

Có ba hiện tượng bất thường (anomaly) có thể xảy ra khi isolation không đủ chặt:

- **Dirty Read**: đọc được dữ liệu chưa commit của transaction khác.
- **Non-repeatable Read**: đọc cùng một dòng dữ liệu hai lần trong một transaction nhưng nhận kết quả khác nhau do transaction khác đã commit thay đổi ở giữa.
- **Phantom Read**: chạy lại cùng một truy vấn nhưng số lượng dòng trả về khác nhau do có dòng mới được thêm/xóa bởi transaction khác.

Bốn mức Isolation Level theo chuẩn SQL, từ lỏng đến chặt:

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read | Hiệu năng |
|---|---|---|---|---|
| Read Uncommitted | Có thể xảy ra | Có thể xảy ra | Có thể xảy ra | Cao nhất |
| Read Committed | Không | Có thể xảy ra | Có thể xảy ra | Cao |
| Repeatable Read | Không | Không | Có thể xảy ra | Trung bình |
| Serializable | Không | Không | Không | Thấp nhất |

**Nguyên tắc chung**: mức cô lập càng chặt thì dữ liệu càng nhất quán nhưng hệ thống càng dễ bị nghẽn (do phải khóa nhiều hơn) và thông lượng càng giảm. Đa số hệ thống thực tế dùng **Read Committed** (mặc định của PostgreSQL, Oracle) làm điểm cân bằng hợp lý.

### 4.2.2. Locking

**Locking (khóa)** là cơ chế mà DBMS sử dụng để kiểm soát quyền truy cập đồng thời vào cùng một dữ liệu, nhằm hiện thực hóa các Isolation Level ở trên.

Hai loại khóa phổ biến:

| Loại khóa | Mô tả | Khi dùng |
|---|---|---|
| **Shared Lock (S Lock)** | Cho phép nhiều transaction cùng đọc một dòng dữ liệu, nhưng không transaction nào được ghi | Khi chỉ cần đọc dữ liệu |
| **Exclusive Lock (X Lock)** | Chỉ một transaction được truy cập (đọc hoặc ghi), các transaction khác phải chờ | Khi cần ghi/sửa dữ liệu |

Ngoài ra còn có phân loại theo phạm vi khóa:

- **Row-level Locking**: khóa ở cấp độ từng dòng dữ liệu — mức độ chi tiết cao, ít xung đột, được hầu hết DBMS hiện đại (PostgreSQL, MySQL InnoDB) sử dụng mặc định.
- **Table-level Locking**: khóa toàn bộ bảng — đơn giản nhưng dễ gây nghẽn khi nhiều transaction cùng thao tác trên một bảng.

**Deadlock (khóa chết)** là tình huống hai hoặc nhiều transaction chờ nhau giải phóng khóa mà không transaction nào có thể tiếp tục. DBMS thường tự động phát hiện và hủy (rollback) một trong các transaction để giải quyết deadlock.

---

## 4.3. Index

### 4.3.1. Index là gì?

**Index (chỉ mục)** là cấu trúc dữ liệu bổ sung giúp DBMS tìm kiếm dữ liệu nhanh hơn mà không cần quét toàn bộ bảng (full table scan). Có thể hình dung Index như mục lục của một cuốn sách: thay vì đọc từng trang để tìm một chủ đề, ta tra mục lục để nhảy thẳng đến đúng trang.

Cấu trúc phổ biến nhất là **B-Tree**, cho phép tìm kiếm, thêm, xóa với độ phức tạp O(log n).

### 4.3.2. Lợi ích và đánh đổi

| Lợi ích | Đánh đổi |
|---|---|
| Tăng tốc độ truy vấn `SELECT`, đặc biệt với mệnh đề `WHERE`, `ORDER BY`, `JOIN` | Tốn thêm dung lượng lưu trữ |
| Tăng tốc độ tìm kiếm trên tập dữ liệu lớn | Làm chậm thao tác `INSERT`, `UPDATE`, `DELETE` vì phải cập nhật lại index |

Vì vậy, nguyên tắc thực hành là chỉ đánh index trên các cột thường xuyên được dùng để tìm kiếm, lọc hoặc sắp xếp (ví dụ: khóa ngoại, cột dùng trong `WHERE` thường xuyên), tránh đánh index tràn lan.

### 4.3.3. Một số loại Index thường gặp

- **Primary Index**: tự động tạo trên khóa chính.
- **Unique Index**: đảm bảo giá trị trong cột là duy nhất.
- **Composite Index**: đánh index trên nhiều cột cùng lúc, hữu ích khi truy vấn thường lọc theo nhiều điều kiện kết hợp.
- **Full-text Index**: tối ưu cho tìm kiếm văn bản.

---

## 4.4. Scaling Database

Khi lượng dữ liệu và số lượng truy vấn tăng lên, một máy chủ database đơn lẻ sẽ không còn đủ khả năng đáp ứng. Có hai hướng tiếp cận chính để mở rộng quy mô database, tương tự khái niệm Vertical/Horizontal Scaling đã trình bày ở Chương 3.

### 4.4.1. Read Replica (Nhân bản đọc)

Tạo ra một hoặc nhiều bản sao (replica) của database chính (primary/master), chuyên phục vụ các truy vấn đọc (`SELECT`), trong khi database chính chỉ xử lý ghi (`INSERT`, `UPDATE`, `DELETE`). Dữ liệu được đồng bộ từ primary sang các replica.

- **Ưu điểm**: giảm tải cho database chính, tăng khả năng chịu đọc đồng thời rất lớn.
- **Hạn chế**: dữ liệu ở replica có độ trễ đồng bộ nhất định (eventual consistency), không phù hợp với các truy vấn yêu cầu dữ liệu chính xác tức thời.

### 4.4.2. Sharding (Phân mảnh dữ liệu)

Chia nhỏ dữ liệu thành nhiều phần (shard) và lưu trên nhiều máy chủ khác nhau, thường dựa trên một khóa phân vùng (ví dụ: chia theo `user_id`, theo khu vực địa lý).

- **Ưu điểm**: mở rộng gần như không giới hạn về dung lượng lưu trữ và khả năng xử lý ghi.
- **Hạn chế**: kiến trúc phức tạp hơn nhiều, các truy vấn liên quan đến nhiều shard (join, tổng hợp) khó thực hiện.

### 4.4.3. So sánh Read Replica và Sharding

| Tiêu chí | Read Replica | Sharding |
|---|---|---|
| Mục tiêu chính | Mở rộng khả năng đọc | Mở rộng khả năng đọc và ghi |
| Độ phức tạp triển khai | Thấp | Cao |
| Dữ liệu | Toàn bộ dữ liệu được nhân bản đầy đủ ở mỗi replica | Dữ liệu được chia nhỏ, mỗi shard chỉ chứa một phần |
| Truy vấn liên bảng/tổng hợp | Dễ dàng (dữ liệu đầy đủ) | Khó khăn, thường cần xử lý ở tầng ứng dụng |

Trong thực tế, các hệ thống thường áp dụng Read Replica trước vì đơn giản, chỉ tiến hành Sharding khi quy mô dữ liệu thực sự vượt ngưỡng mà một máy chủ (hoặc cụm read replica) không thể xử lý.

---

## 4.5. ORM và Data Access Layer

### 4.5.1. ORM là gì?

**ORM (Object-Relational Mapping)** là kỹ thuật ánh xạ giữa các đối tượng trong ngôn ngữ lập trình hướng đối tượng (class, object) với các bảng trong cơ sở dữ liệu quan hệ. Thay vì viết câu lệnh SQL thuần, lập trình viên thao tác với dữ liệu thông qua các đối tượng và phương thức.

Ví dụ, thay vì viết:

```sql
SELECT * FROM users WHERE id = 1;
```

Lập trình viên có thể viết bằng ORM:

```ts
const user = await userRepository.findOne({ where: { id: 1 } });
```

### 4.5.2. So sánh các cách tiếp cận truy cập dữ liệu

| Cách tiếp cận | Mô tả | Ưu điểm | Nhược điểm |
|---|---|---|---|
| **Raw SQL** | Viết trực tiếp câu lệnh SQL | Toàn quyền kiểm soát, hiệu năng tối ưu | Dễ lỗi cú pháp, không tận dụng được kiểu dữ liệu của ngôn ngữ, khó bảo trì khi schema thay đổi |
| **Query Builder** | Xây dựng câu lệnh SQL bằng API của ngôn ngữ (ví dụ: Knex.js) | Linh hoạt hơn raw SQL, vẫn gần với SQL gốc | Vẫn cần hiểu SQL, không có mô hình đối tượng đầy đủ |
| **ORM** | Ánh xạ bảng thành đối tượng, thao tác qua entity/model | Code ngắn gọn, an toàn kiểu dữ liệu, hỗ trợ migration, giảm rủi ro SQL Injection | Có thể sinh ra câu lệnh SQL không tối ưu nếu dùng sai cách, có độ trễ hiệu năng nhất định so với raw SQL |

### 4.5.3. N+1 Query Problem

Đây là một trong những lỗi hiệu năng phổ biến nhất khi dùng ORM. Vấn đề xảy ra khi lấy danh sách N bản ghi, sau đó với **mỗi** bản ghi lại thực hiện thêm một truy vấn riêng để lấy dữ liệu liên quan — dẫn đến tổng cộng N+1 truy vấn thay vì chỉ 1 hoặc 2 truy vấn tối ưu.

Ví dụ: lấy 100 bài viết (1 truy vấn), sau đó với mỗi bài viết lại truy vấn riêng để lấy thông tin tác giả (100 truy vấn) — tổng cộng 101 truy vấn.

**Cách khắc phục phổ biến**: sử dụng kỹ thuật *eager loading* (tải trước dữ liệu liên quan trong cùng một truy vấn, thông qua `JOIN` hoặc `include`) thay vì *lazy loading* (chỉ tải khi được truy cập).

### 4.5.4. Migration

**Migration** là tập hợp các thay đổi về cấu trúc (schema) của database — như tạo bảng, thêm cột, đổi kiểu dữ liệu — được quản lý dưới dạng các file có phiên bản, tương tự cách Git quản lý lịch sử thay đổi mã nguồn.

Lợi ích của migration:

- Đồng bộ cấu trúc database giữa các môi trường (local, staging, production) và giữa các thành viên trong nhóm.
- Cho phép rollback về phiên bản schema trước đó khi cần.
- Lưu lại lịch sử thay đổi cấu trúc dữ liệu theo thời gian.

Hầu hết các ORM hiện đại (Prisma, TypeORM) đều tích hợp sẵn công cụ quản lý migration, sẽ được trình bày chi tiết cùng ví dụ thực hành trong Chương 5 (NestJS Core).

---

## Tổng kết chương

Chương này đã trình bày các khái niệm nền tảng để làm việc an toàn và hiệu quả với cơ sở dữ liệu: đảm bảo tính toàn vẹn dữ liệu qua Transaction và ACID, kiểm soát truy cập đồng thời qua Isolation Level và Locking, tối ưu tốc độ truy vấn qua Index, mở rộng quy mô qua Read Replica và Sharding, và cuối cùng là vai trò của ORM trong việc đơn giản hóa thao tác dữ liệu. Đây là kiến thức nền cần thiết trước khi bước sang Chương 5, nơi các khái niệm này được áp dụng cụ thể trong framework NestJS.
