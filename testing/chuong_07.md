# GIÁO TRÌNH KIỂM THỬ VÀ ĐẢM BẢO CHẤT LƯỢNG PHẦN MỀM

---

# CHƯƠNG 7: DATABASE TESTING

---

## 7.1 Database Testing là gì?

**Database Testing** là quá trình kiểm thử các thành phần liên quan đến cơ sở dữ liệu: schema, dữ liệu, stored procedure, trigger, ràng buộc toàn vẹn, và tính nhất quán của dữ liệu khi hệ thống hoạt động.

Kiểm thử UI và API xác minh rằng phần mềm *hoạt động đúng* từ góc nhìn người dùng. Database Testing xác minh rằng dữ liệu *được lưu đúng, truy xuất đúng, và nhất quán* ở tầng thấp nhất.

**Tại sao Database Testing quan trọng:**
- Bug có thể "ẩn" trong DB mà UI không phản ánh ngay: dữ liệu sai nhưng hiển thị bình thường
- Dữ liệu sai tích lũy theo thời gian gây hậu quả nghiêm trọng (sai số lương, sai tồn kho)
- Ràng buộc DB là lớp bảo vệ cuối cùng — nếu ràng buộc sai, dữ liệu rác lọt vào
- Performance của truy vấn ảnh hưởng trực tiếp đến tốc độ ứng dụng

---

## 7.2 SQL cần biết cho Tester

### 7.2.1 SELECT và WHERE

```sql
-- Lấy tất cả cột
SELECT * FROM users;

-- Lấy cột cụ thể
SELECT id, email, name, created_at FROM users;

-- Lọc theo điều kiện
SELECT * FROM orders WHERE status = 'pending';
SELECT * FROM products WHERE price > 100000 AND stock > 0;
SELECT * FROM users WHERE email LIKE '%@test.com';

-- Toán tử IN, BETWEEN, IS NULL
SELECT * FROM orders WHERE status IN ('pending', 'processing');
SELECT * FROM products WHERE price BETWEEN 100000 AND 500000;
SELECT * FROM users WHERE deleted_at IS NULL;  -- chưa bị xóa mềm
SELECT * FROM users WHERE deleted_at IS NOT NULL;  -- đã xóa mềm

-- Phủ định
SELECT * FROM orders WHERE status != 'cancelled';
SELECT * FROM users WHERE NOT email LIKE '%@test.com';
```

### 7.2.2 JOIN — Kết hợp nhiều bảng

```sql
-- INNER JOIN: Chỉ lấy các bản ghi có match ở CẢ HAI bảng
SELECT
    o.id AS order_id,
    o.status,
    o.total,
    u.email AS customer_email,
    u.name AS customer_name
FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending';

-- LEFT JOIN: Lấy TẤT CẢ bản ghi bảng trái, NULL nếu không có match bên phải
SELECT
    p.id,
    p.name,
    p.price,
    c.name AS category_name
FROM products p
LEFT JOIN categories c ON p.category_id = c.id;
-- Sản phẩm chưa có category vẫn xuất hiện, category_name = NULL

-- RIGHT JOIN: Ngược lại LEFT JOIN (ít dùng)

-- Nhiều JOIN
SELECT
    oi.order_id,
    oi.quantity,
    oi.unit_price,
    p.name AS product_name,
    o.status AS order_status,
    u.email AS customer_email
FROM order_items oi
INNER JOIN orders o ON oi.order_id = o.id
INNER JOIN products p ON oi.product_id = p.id
INNER JOIN users u ON o.user_id = u.id
WHERE o.created_at >= '2025-01-01';
```

### 7.2.3 GROUP BY, HAVING, ORDER BY

```sql
-- GROUP BY: Nhóm và tổng hợp
SELECT
    status,
    COUNT(*) AS total_orders,
    SUM(total) AS total_revenue,
    AVG(total) AS avg_order_value
FROM orders
GROUP BY status;

-- HAVING: Lọc sau GROUP BY (WHERE không dùng được với aggregate)
SELECT
    user_id,
    COUNT(*) AS order_count,
    SUM(total) AS total_spent
FROM orders
GROUP BY user_id
HAVING COUNT(*) >= 5  -- chỉ lấy user đặt từ 5 đơn trở lên
ORDER BY total_spent DESC;

-- ORDER BY
SELECT * FROM products
ORDER BY price DESC, name ASC;  -- giá giảm dần, tên tăng dần
```

### 7.2.4 INSERT, UPDATE, DELETE

```sql
-- INSERT
INSERT INTO test_users (email, name, role, created_at)
VALUES ('qa_test@example.com', 'QA Test User', 'customer', NOW());

-- INSERT nhiều dòng
INSERT INTO test_products (name, price, stock) VALUES
    ('[TEST] Sản phẩm A', 100000, 50),
    ('[TEST] Sản phẩm B', 200000, 30),
    ('[TEST] Sản phẩm C', 300000, 0);

-- UPDATE
UPDATE products
SET stock = stock - 2
WHERE id = 'P001';

-- UPDATE có điều kiện
UPDATE orders
SET status = 'cancelled', cancelled_at = NOW()
WHERE status = 'pending'
  AND created_at < NOW() - INTERVAL '24 hours';

-- DELETE
DELETE FROM test_users WHERE email LIKE '%@test.com';

-- Soft delete (thực tế thường dùng thay DELETE)
UPDATE users
SET deleted_at = NOW()
WHERE id = 123;
```

### 7.2.5 Subquery

```sql
-- Tìm users chưa bao giờ đặt hàng
SELECT id, email, name
FROM users
WHERE id NOT IN (
    SELECT DISTINCT user_id FROM orders
);

-- Tìm sản phẩm có giá cao hơn giá trung bình
SELECT id, name, price
FROM products
WHERE price > (SELECT AVG(price) FROM products);

-- Tìm đơn hàng lớn nhất của mỗi user
SELECT o.*
FROM orders o
INNER JOIN (
    SELECT user_id, MAX(total) AS max_total
    FROM orders
    GROUP BY user_id
) AS max_orders ON o.user_id = max_orders.user_id
                AND o.total = max_orders.max_total;
```

---

## 7.3 Các Loại Database Testing

### 7.3.1 CRUD Testing

Kiểm thử cơ bản nhất: Create, Read, Update, Delete — đảm bảo API và DB phối hợp đúng.

**Ví dụ CRUD Testing cho module Employee:**

```sql
-- 1. CREATE: Sau khi POST /api/employees
-- Verify record được tạo đúng
SELECT
    id, name, email, department, position,
    salary, status, created_at
FROM employees
WHERE email = 'qa_test_emp@company.com';

-- Kết quả mong đợi:
-- id: auto-generated
-- status: 'active'
-- created_at: gần bằng thời điểm POST
-- salary: đúng với request body

-- 2. READ: Sau khi GET /api/employees/:id
-- Verify query không trả dữ liệu nhạy cảm không cần thiết
-- (kiểm tra API response, không phải DB trực tiếp)

-- 3. UPDATE: Sau khi PATCH /api/employees/:id với {salary: 18000000}
-- Verify chỉ salary thay đổi, các field khác giữ nguyên
SELECT
    id, name, email, salary,
    updated_at  -- phải cập nhật
FROM employees
WHERE id = 123;

-- 4. DELETE (Soft delete): Sau khi DELETE /api/employees/:id
-- Verify soft delete, không xóa khỏi DB
SELECT id, email, deleted_at
FROM employees
WHERE id = 123;
-- Expected: deleted_at = timestamp, không NULL

-- Verify employee không hiện trong danh sách active
SELECT COUNT(*)
FROM employees
WHERE id = 123 AND deleted_at IS NULL;
-- Expected: 0
```

---

### 7.3.2 Data Integrity Testing

**Data Integrity** đảm bảo dữ liệu chính xác, nhất quán, và hoàn chỉnh trong suốt vòng đời của nó.

**4 loại Data Integrity:**

**Entity Integrity:** Mỗi bảng phải có Primary Key, không NULL, không trùng lặp.
```sql
-- Verify không có duplicate primary key
SELECT id, COUNT(*) FROM users GROUP BY id HAVING COUNT(*) > 1;
-- Expected: 0 rows

-- Verify không có NULL primary key
SELECT COUNT(*) FROM users WHERE id IS NULL;
-- Expected: 0
```

**Referential Integrity:** Foreign Key phải tham chiếu đến record tồn tại.
```sql
-- Verify không có order của user không tồn tại
SELECT o.id, o.user_id
FROM orders o
LEFT JOIN users u ON o.user_id = u.id
WHERE u.id IS NULL;
-- Expected: 0 rows (tất cả user_id phải hợp lệ)

-- Verify không có order_items của order không tồn tại
SELECT oi.id, oi.order_id
FROM order_items oi
LEFT JOIN orders o ON oi.order_id = o.id
WHERE o.id IS NULL;
-- Expected: 0 rows
```

**Domain Integrity:** Giá trị trong column phải hợp lệ.
```sql
-- Verify giá sản phẩm không âm
SELECT id, name, price FROM products WHERE price < 0;
-- Expected: 0 rows

-- Verify status chỉ có giá trị hợp lệ
SELECT DISTINCT status FROM orders;
-- Expected: chỉ có: 'pending', 'processing', 'shipped', 'delivered', 'cancelled'

-- Verify rating trong khoảng 1-5
SELECT id, product_id, rating FROM reviews WHERE rating < 1 OR rating > 5;
-- Expected: 0 rows

-- Verify email đúng format
SELECT id, email FROM users WHERE email NOT LIKE '%@%.%';
-- Expected: 0 rows
```

**User-defined Integrity:** Quy tắc nghiệp vụ tùy chỉnh.
```sql
-- Verify ngày kết thúc hợp đồng sau ngày bắt đầu
SELECT id, start_date, end_date
FROM contracts
WHERE end_date IS NOT NULL AND end_date <= start_date;
-- Expected: 0 rows

-- Verify tổng đơn hàng >= tổng items
SELECT
    o.id,
    o.total AS order_total,
    SUM(oi.quantity * oi.unit_price) AS calculated_total
FROM orders o
INNER JOIN order_items oi ON o.id = oi.order_id
GROUP BY o.id, o.total
HAVING o.total != SUM(oi.quantity * oi.unit_price);
-- Expected: 0 rows (tổng phải khớp)
```

---

### 7.3.3 Constraint Testing — Kiểm thử ràng buộc

**Primary Key:**
```sql
-- Test: Insert record với ID đã tồn tại
INSERT INTO users (id, email) VALUES (123, 'new@test.com');
-- Expected: ERROR - duplicate key value violates unique constraint "users_pkey"

-- Test: Insert với NULL primary key
INSERT INTO users (id, email) VALUES (NULL, 'new@test.com');
-- Expected: ERROR - null value in column "id" violates not-null constraint
```

**Foreign Key:**
```sql
-- Test: Tạo order với user_id không tồn tại
INSERT INTO orders (user_id, total, status)
VALUES (99999, 100000, 'pending');
-- Expected: ERROR - insert or update on table "orders" violates
-- foreign key constraint "orders_user_id_fkey"

-- Test: Xóa user đang có order
DELETE FROM users WHERE id = 123;
-- Expected: ERROR - update or delete on table "users" violates
-- foreign key constraint on table "orders"
-- (Hoặc cascade delete nếu được cấu hình)
```

**UNIQUE Constraint:**
```sql
-- Test: Đăng ký email đã tồn tại
INSERT INTO users (email, name) VALUES ('existing@test.com', 'Test');
-- Expected: ERROR - duplicate key value violates unique constraint "users_email_key"
```

**NOT NULL Constraint:**
```sql
-- Test: Insert order không có user_id
INSERT INTO orders (total, status) VALUES (100000, 'pending');
-- Expected: ERROR - null value in column "user_id" violates not-null constraint
```

**CHECK Constraint:**
```sql
-- Giả sử có: CHECK (price > 0)
INSERT INTO products (name, price) VALUES ('Test Product', -100);
-- Expected: ERROR - new row for relation "products" violates check constraint "products_price_check"
```

---

### 7.3.4 Transaction Testing

**Transaction** đảm bảo một chuỗi thao tác DB được thực hiện **tất cả hoặc không** (ACID properties).

**ACID:**
- **Atomicity:** Transaction thực hiện toàn bộ hoặc rollback hoàn toàn
- **Consistency:** DB chuyển từ trạng thái hợp lệ sang trạng thái hợp lệ khác
- **Isolation:** Transactions đồng thời không ảnh hưởng lẫn nhau
- **Durability:** Sau khi commit, dữ liệu được lưu vĩnh viễn

**Ví dụ kiểm thử Transaction cho tính năng chuyển khoản:**

```sql
-- Scenario: Chuyển 100,000đ từ account A (balance: 500,000) sang B (balance: 200,000)

-- Trước khi thực hiện
SELECT id, balance FROM accounts WHERE id IN (1, 2);
-- A: 500,000 | B: 200,000

-- Thực hiện transfer qua API: POST /api/transfers
-- { fromAccountId: 1, toAccountId: 2, amount: 100000 }

-- Verify sau khi thành công
SELECT id, balance FROM accounts WHERE id IN (1, 2);
-- Expected: A: 400,000 | B: 300,000

-- Tổng tiền không thay đổi (bảo toàn)
SELECT SUM(balance) FROM accounts WHERE id IN (1, 2);
-- Expected: 700,000 (trước và sau đều phải là 700,000)
```

**Kiểm thử Rollback — khi transfer thất bại:**
```sql
-- Scenario: Chuyển 1,000,000đ từ account A (balance: 500,000) → Không đủ tiền

-- Trước khi thực hiện
SELECT id, balance FROM accounts WHERE id IN (1, 2);
-- A: 500,000 | B: 200,000

-- Thực hiện transfer: POST /api/transfers
-- { fromAccountId: 1, toAccountId: 2, amount: 1000000 }
-- → API trả về 422: "Insufficient balance"

-- Verify KHÔNG có thay đổi gì trong DB
SELECT id, balance FROM accounts WHERE id IN (1, 2);
-- Expected: A: 500,000 | B: 200,000 (giống trước)

-- Verify không có transaction record được tạo
SELECT * FROM transfer_logs
WHERE from_account_id = 1 AND amount = 1000000
ORDER BY created_at DESC LIMIT 1;
-- Expected: 0 rows HOẶC record với status = 'failed'
```

---

### 7.3.5 Data Consistency Testing

Kiểm tra dữ liệu nhất quán **giữa các bảng** liên quan sau khi thực hiện nghiệp vụ.

**Ví dụ: Sau khi đặt hàng thành công**

```sql
-- 1. Tổng đơn hàng = tổng items
SELECT
    o.id,
    o.total AS order_total,
    SUM(oi.quantity * oi.unit_price) AS items_total,
    (o.total = SUM(oi.quantity * oi.unit_price)) AS is_consistent
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.id = 'ORD-2025-001'
GROUP BY o.id, o.total;

-- 2. Tồn kho đã giảm đúng số lượng
-- Trước đặt hàng: SP-A stock = 10, SP-B stock = 5
-- Đặt: SP-A × 2, SP-B × 1
SELECT id, name, stock
FROM products
WHERE id IN ('P001', 'P003');
-- Expected: SP-A stock = 8, SP-B stock = 4

-- 3. Điểm loyalty của user tăng đúng (nếu có chức năng này)
SELECT user_id, loyalty_points, updated_at
FROM user_loyalty
WHERE user_id = 123;
-- Expected: tăng thêm points tương ứng với order total

-- 4. Email log có record
SELECT recipient, template, status, sent_at
FROM email_logs
WHERE reference_id = 'ORD-2025-001' AND template = 'order_confirmation';
-- Expected: 1 record với status = 'sent'
```

---

### 7.3.6 API ↔ Database Testing — Kiểm thử End-to-End qua DB

Đây là kỹ thuật kiểm thử mạnh nhất: trigger hành động qua API, verify kết quả trong DB.

**Ví dụ: Kiểm thử tính năng Đăng ký tài khoản**

```
Bước 1: Gọi API đăng ký
POST /api/v1/auth/register
{
  "email": "qa_new_user@test.com",
  "password": "Test@123",
  "name": "QA Test User"
}
→ Response: 201 Created, {"userId": 456}

Bước 2: Verify trong DB - Users table
SELECT
    id, email, name, status,
    password_hash,  -- phải là hash, không phải plaintext
    email_verified,
    created_at
FROM users
WHERE email = 'qa_new_user@test.com';

Kết quả mong đợi:
- id: 456 (khớp với response)
- status: 'pending_verification'
- email_verified: false
- password_hash: bắt đầu bằng $2b$ (bcrypt hash)
- created_at: timestamp vừa tạo

Bước 3: Verify Email Queue
SELECT recipient, subject, status
FROM email_queue
WHERE recipient = 'qa_new_user@test.com'
  AND template = 'email_verification';

Kết quả mong đợi:
- 1 record với status = 'queued' hoặc 'sent'

Bước 4: Verify Token
SELECT token, user_id, expires_at, used
FROM verification_tokens
WHERE user_id = 456;

Kết quả mong đợi:
- 1 record với expires_at = created_at + 24 giờ
- used: false

Bước 5: Thử đăng nhập khi chưa verify
POST /api/v1/auth/login
{"email": "qa_new_user@test.com", "password": "Test@123"}
→ Expected: 403, {"error": "Email chưa được xác nhận"}

Bước 6: Giả lập click link xác nhận
POST /api/v1/auth/verify-email
{"token": "[token từ DB]"}
→ Expected: 200 OK

Bước 7: Verify DB sau khi xác nhận
SELECT status, email_verified FROM users WHERE id = 456;
-- Expected: status = 'active', email_verified = true

SELECT used FROM verification_tokens WHERE user_id = 456;
-- Expected: used = true (token đã dùng, không dùng lại được)
```

---

## 7.4 SQL Nâng cao cho Tester

### 7.4.1 Window Functions — Hàm cửa sổ

```sql
-- Xếp hạng sản phẩm bán chạy nhất theo category
SELECT
    p.name,
    p.category_id,
    SUM(oi.quantity) AS total_sold,
    RANK() OVER (
        PARTITION BY p.category_id
        ORDER BY SUM(oi.quantity) DESC
    ) AS rank_in_category
FROM products p
JOIN order_items oi ON p.id = oi.product_id
JOIN orders o ON oi.order_id = o.id
WHERE o.status = 'delivered'
GROUP BY p.id, p.name, p.category_id;

-- Tổng tích lũy doanh thu theo ngày
SELECT
    DATE(created_at) AS date,
    SUM(total) AS daily_revenue,
    SUM(SUM(total)) OVER (ORDER BY DATE(created_at)) AS cumulative_revenue
FROM orders
WHERE status = 'delivered'
  AND created_at >= '2025-01-01'
GROUP BY DATE(created_at);
```

### 7.4.2 CTE — Common Table Expressions

```sql
-- Tìm users VIP (đặt trên 10 triệu trong 30 ngày)
WITH recent_orders AS (
    SELECT user_id, SUM(total) AS total_spent
    FROM orders
    WHERE status IN ('delivered', 'processing')
      AND created_at >= NOW() - INTERVAL '30 days'
    GROUP BY user_id
),
vip_users AS (
    SELECT user_id, total_spent
    FROM recent_orders
    WHERE total_spent >= 10000000
)
SELECT
    u.id,
    u.email,
    u.name,
    v.total_spent
FROM vip_users v
JOIN users u ON v.user_id = u.id
ORDER BY v.total_spent DESC;
```

### 7.4.3 Các Query hữu ích cho Tester

```sql
-- 1. Tìm duplicate data
SELECT email, COUNT(*) AS count
FROM users
GROUP BY email
HAVING COUNT(*) > 1;

-- 2. Data được tạo trong khoảng thời gian
SELECT * FROM orders
WHERE created_at BETWEEN '2025-01-15 00:00:00' AND '2025-01-15 23:59:59';

-- 3. N records mới nhất
SELECT * FROM error_logs
ORDER BY created_at DESC
LIMIT 10;

-- 4. Tìm orphan records (FK bị broken)
SELECT o.id, o.user_id
FROM orders o
WHERE NOT EXISTS (
    SELECT 1 FROM users u WHERE u.id = o.user_id
);

-- 5. Kiểm tra dữ liệu sau migration
-- So sánh count giữa bảng cũ và mới
SELECT
    (SELECT COUNT(*) FROM orders_old) AS old_count,
    (SELECT COUNT(*) FROM orders_new) AS new_count,
    (SELECT COUNT(*) FROM orders_old) = (SELECT COUNT(*) FROM orders_new) AS counts_match;

-- 6. Kiểm tra không có khoảng trắng thừa
SELECT id, name
FROM products
WHERE name != TRIM(name)  -- tên có leading/trailing space
   OR name LIKE '%  %';   -- có double space

-- 7. Xem size của các bảng (PostgreSQL)
SELECT
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    n_live_tup AS row_count
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- 8. Tìm slow queries (PostgreSQL)
SELECT
    query,
    calls,
    total_exec_time / calls AS avg_time_ms,
    rows / calls AS avg_rows
FROM pg_stat_statements
ORDER BY avg_time_ms DESC
LIMIT 20;
```

---

## 7.5 Checklist Database Testing

```
Schema Validation
□ Tất cả bảng có Primary Key
□ Foreign Keys được định nghĩa đúng
□ Indexes có trên các cột thường dùng để query/join
□ Kiểu dữ liệu phù hợp với dữ liệu lưu trữ
□ Default values hợp lý

Data Integrity
□ Không có NULL ở cột bắt buộc
□ Không có duplicate ở cột UNIQUE
□ FK không trỏ đến record không tồn tại
□ Giá trị nằm trong domain hợp lệ (range, enum)

CRUD Operations
□ INSERT tạo đúng record với đúng giá trị
□ SELECT trả về đúng dữ liệu, đúng số lượng
□ UPDATE chỉ thay đổi fields được yêu cầu
□ DELETE (hoặc soft delete) hoạt động đúng

Business Logic
□ Tính toán (tổng, discount) cho kết quả chính xác
□ Timestamps (created_at, updated_at) cập nhật đúng
□ Status transitions hợp lệ
□ Không có data leak giữa các user/tenant

Performance
□ Query trên dataset lớn có index phù hợp
□ Không có N+1 query issue
□ Response time API với DB call nằm trong SLA
```
