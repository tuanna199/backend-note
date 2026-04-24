# Database Indexing — Kiến thức chuyên sâu cho Backend Engineer (PostgreSQL)

> File này là phần lý thuyết & chiến lược đi kèm với `db-lab.md`.
> Toàn bộ ví dụ SQL dùng schema: `users` (500k), `products` (50k), `orders` (2M), `order_items` (6M).

---

## 1. Tại sao Index tồn tại?

### 1.1 PostgreSQL lưu dữ liệu như thế nào

PostgreSQL lưu mỗi bảng dưới dạng một tập hợp **heap pages**, mỗi page có kích thước mặc định **8 KB**.
Mỗi page chứa nhiều **tuple** (dòng). Không có page nào được sắp xếp theo thứ tự logic của dữ liệu —
dòng mới nhất thường nằm cuối file, dòng bị xóa để lại "dead tuple" tại chỗ.

```
Heap file của bảng users (500.000 dòng ≈ ~90 MB):
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Page 0     │ │  Page 1     │ │  Page 2     │ ...
│ (tuple 1-8) │ │ (tuple 9-16)│ │ (tuple 17..)│
└─────────────┘ └─────────────┘ └─────────────┘
```

Khi bạn chạy `SELECT * FROM users WHERE email = 'user12345@gmail.com'` **không có index**,
PostgreSQL phải đọc **toàn bộ ~11.000 pages** (90 MB) để tìm 1 dòng duy nhất → **Sequential Scan**.

### 1.2 Chi phí thực tế

| Thao tác                           | Không có index          | Có index                  |
| ----------------------------------- | ------------------------- | -------------------------- |
| Tìm 1 email trong 500k users       | ~90 MB đọc, ~50–200 ms | ~32 KB đọc, < 1 ms       |
| Lọc orders theo user_id (2M dòng) | full 2M scan              | B-Tree traversal O(log n)  |
| JOIN order_items (6M dòng)         | nested loop O(n²)        | hash/merge join với index |

### 1.3 Đánh đổi: Index không miễn phí

Index tăng tốc **đọc** nhưng làm **chậm ghi**:

- **INSERT**: PostgreSQL phải cập nhật tất cả index liên quan đến bảng đó
- **UPDATE**: nếu cột được index thay đổi → xóa entry cũ, thêm entry mới trong index
- **DELETE**: đánh dấu entry trong index là dead (không xóa ngay, dọn khi VACUUM chạy)

Quy tắc: bảng **write-heavy** (log, event stream) → ít index. Bảng **read-heavy** (catalog, user profile) → nhiều index.

---

## 2. Các loại Index trong PostgreSQL

### 2.1 Bảng tổng quan

| Loại             | Cấu trúc              | Dùng cho                                 | Operators hỗ trợ                                                                                      |
| ----------------- | ----------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| **B-Tree**  | Cây cân bằng         | Mặc định, hầu hết mọi trường hợp | `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `LIKE 'prefix%'`, `IN`, `IS NULL`, `ORDER BY` |
| **Hash**    | Hash table              | Chỉ equality, key rất dài              | `=` (chỉ duy nhất)                                                                                  |
| **GIN**     | Inverted index          | Nhiều giá trị trên 1 dòng            | `@>`, `<@`, `&&`, `?`, `@@` (full-text)                                                       |
| **GiST**    | Generalized Search Tree | Geometry, range, nearest-neighbor         | `<<`, `>>`, `&&`, `<->`, `@>`, `<@`                                                         |
| **BRIN**    | Block Range Index       | Bảng lớn, cột monotonic                | `=`, `<`, `>`, `BETWEEN` (chỉ hiệu quả khi sorted on disk)                                   |
| **SP-GiST** | Space-partitioned GiST  | IP, phone, text prefix trees              | `=`, `<<`, `>>`, `<@`                                                                           |

### 2.2 B-Tree

B+Tree là cấu trúc mặc định của PostgreSQL index. Gồm 3 loại node:

```
                    [Root Node]
                   /     |      \
          [Internal]  [Internal]  [Internal]
          /     \       /   \       /    \
      [Leaf] [Leaf] [Leaf] [Leaf] [Leaf] [Leaf]
         ↓      ↓      ↓      ↓      ↓      ↓
      (heap) (heap) (heap) (heap) (heap) (heap)
```

- **Internal node**: chứa key dùng để route (không trỏ trực tiếp vào heap)
- **Leaf node**: chứa key thực + **TID** (Tuple ID = page number + slot) trỏ đến heap
- Các leaf node được **liên kết thành danh sách đôi** → range query chạy theo danh sách, không cần quay lại root

**Ví dụ traversal với `idx_users_email`:**

```
Tìm: email = 'user12345@gmail.com'

Root → Internal [user5... | user8...] →
  nhánh trái Internal [user1... | user3...] →
    Leaf ['user12344@...', TID(450,3)]
         ['user12345@gmail.com', TID(451,1)]  ← tìm thấy
         ['user12346@...', TID(451,2)]
→ đọc page 451, slot 1 từ heap → trả về dòng
```

Chiều cao B+Tree của 500k users ~ **3–4 levels** (log₃₀₀(500000) ≈ 3.4).
Đọc 3–4 page index + 1 page heap = **4–5 random I/O** thay vì 11.000 I/O.

```sql
CREATE INDEX idx_users_email ON users(email);       -- equality lookup
CREATE INDEX idx_users_age   ON users(age);         -- range query
CREATE INDEX idx_orders_created ON orders(created_at DESC); -- ORDER BY
```

Hỗ trợ `LIKE 'prefix%'` **chỉ khi** pattern bắt đầu bằng ký tự cố định (không bắt đầu bằng `%`).

**Index Correlation:**

Index correlation đo mức độ trùng khớp giữa thứ tự vật lý của heap và thứ tự trong index (từ -1 đến 1).

```sql
SELECT attname, correlation
FROM pg_stats
WHERE tablename = 'users'
ORDER BY abs(correlation) DESC;
```

- `correlation ≈ 1.0` (ví dụ: `id`, `created_at`): index scan rất hiệu quả vì các heap page liên tiếp nhau
- `correlation ≈ 0` (ví dụ: `email`): mỗi leaf node trỏ đến heap page khác nhau → nhiều random I/O hơn

Khi correlation thấp và số dòng trả về nhiều, PostgreSQL chuyển sang **Bitmap Heap Scan** để gom nhiều TID lại trước khi đọc heap theo thứ tự.

### 2.3 Hash Index

```sql
CREATE INDEX idx_users_email_hash ON users USING HASH(email);
```

- Kích thước nhỏ hơn B-Tree ~20% khi key dài (UUID, token)
- **Không** hỗ trợ range, ORDER BY, multicolumn
- Sau PostgreSQL 10: Hash index được WAL-logged (an toàn khi crash)
- Trong thực tế: B-Tree vẫn ưu tiên hơn vì linh hoạt hơn

### 2.4 GIN — Inverted Index

GIN phù hợp khi **1 dòng có nhiều giá trị cần index** (array, JSONB, tsvector). Lưu một **bảng ánh xạ ngược**: mỗi giá trị → danh sách TID của các dòng chứa nó.

```
Cấu trúc GIN cho cột tsvector:

┌──────────────────────┐     ┌───────────────────────────────────┐
│  B-Tree của keys     │     │  Posting Lists (sorted TID lists) │
│  ─────────────────── │     │  ─────────────────────────────── │
│  "portable"          │ ──► │  [TID(12,1), TID(34,2), TID(89,1)]│
│  "usb"               │ ──► │  [TID(5,3), TID(89,1), TID(201,2)]│
│  "wireless"          │ ──► │  [TID(2,1), TID(34,2), TID(500,4)]│
└──────────────────────┘     └───────────────────────────────────┘
```

Lookup `WHERE description @@ 'wireless & portable'`: lấy posting list của "wireless" và "portable" → intersect → đọc heap. Build chậm hơn B-Tree nhưng lookup rất nhanh (O(log n) + posting list merge).

**`fastupdate`:**

- **`fastupdate = on` (mặc định)**: ghi INSERT vào "pending list" (heap buffer), flush sang index theo batch → write nhanh hơn, lookup phải check thêm pending list
- **`fastupdate = off`**: ghi thẳng vào index, consistency ngay lập tức, phù hợp khi read latency quan trọng

```sql
CREATE INDEX idx_products_fts ON products
    USING GIN(to_tsvector('english', description))
    WITH (fastupdate = off);
```

**GIN với JSONB — hai opclass:**

```sql
-- Mặc định: hỗ trợ @>, <@, ?, ?|, ?&
CREATE INDEX idx_users_metadata ON users USING GIN(metadata);

-- jsonb_path_ops: nhỏ hơn ~30%, chỉ hỗ trợ @>
CREATE INDEX idx_users_metadata_path ON users
    USING GIN(metadata jsonb_path_ops);
```

Dùng `jsonb_path_ops` khi chỉ cần `@>` (containment check) và muốn tiết kiệm disk/RAM.

**Full-Text Search:**

```sql
-- Expression index
CREATE INDEX idx_products_fts ON products
    USING GIN(to_tsvector('english', description));

SELECT name FROM products
WHERE to_tsvector('english', description) @@ to_tsquery('wireless & portable');

-- Ranking kết quả
SELECT name,
       ts_rank(to_tsvector('english', description),
               to_tsquery('wireless')) AS rank
FROM products
WHERE to_tsvector('english', description) @@ to_tsquery('wireless')
ORDER BY rank DESC
LIMIT 10;
```

**Stored tsvector column** (hiệu quả hơn khi bảng update nhiều):

```sql
ALTER TABLE products ADD COLUMN description_fts TSVECTOR;

CREATE OR REPLACE FUNCTION update_products_fts() RETURNS TRIGGER AS $$
BEGIN
    NEW.description_fts := to_tsvector('english', coalesce(NEW.description, ''));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trig_products_fts
    BEFORE INSERT OR UPDATE ON products
    FOR EACH ROW EXECUTE FUNCTION update_products_fts();

CREATE INDEX idx_products_fts_stored ON products USING GIN(description_fts);

SELECT name FROM products
WHERE description_fts @@ to_tsquery('wireless & portable');
```

| Cách                                      | Khi nào dùng                                     |
| ------------------------------------------ | -------------------------------------------------- |
| Expression index `GIN(to_tsvector(...))` | Bảng ít update, đơn giản hơn                 |
| Stored tsvector column + trigger           | Bảng update nhiều, hiệu năng query quan trọng |

---

## 3. Thiết kế Index hiệu quả

### 3.1 Composite Index & Leftmost Prefix Rule

Composite index `(col_A, col_B)` hoạt động như sách điện thoại sắp xếp theo **họ rồi đến tên**.

```sql
-- Từ LAB 2 trong db-init.md
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

| Query                                        | Dùng được index? | Lý do             |
| -------------------------------------------- | -------------------- | ------------------ |
| `WHERE user_id = 1234 AND status = 'paid'` | Có (full)           | Cả 2 cột         |
| `WHERE user_id = 1234`                     | Có (partial)        | Leftmost prefix    |
| `WHERE status = 'paid'`                    | Không               | Bỏ qua cột đầu |

**Quy tắc chọn thứ tự cột:**

1. Cột được dùng trong equality filter (`=`) → đặt trước
2. Cột có **selectivity cao hơn** → đặt trước (ít dòng trùng hơn)
3. Cột dùng trong range filter (`>`, `<`, `BETWEEN`) → đặt sau cùng

```sql
-- Ví dụ: query thường xuyên là user_id + status + created_at range
-- Đặt equality filters trước, range filter cuối
CREATE INDEX idx_orders_optimal ON orders(user_id, status, created_at);

-- Query này tận dụng đầy đủ index:
SELECT * FROM orders
WHERE user_id = 1234
  AND status = 'paid'
  AND created_at > now() - interval '30 days';
```

### 3.2 Partial Index

Partial index chỉ index tập con của dữ liệu thỏa mãn điều kiện `WHERE`.

```sql
-- Từ LAB 3 trong db-init.md
-- Chỉ 5% users bị banned → index nhỏ hơn 20x so với full index
CREATE INDEX idx_users_banned ON users(country)
    WHERE status = 'banned';

-- Kiểm tra kích thước:
SELECT indexname, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
WHERE tablename = 'users';
```

**Khi nào dùng Partial Index:**

- Cột status có nhiều giá trị nhưng bạn chỉ query 1 giá trị thiểu số (`banned`, `pending`, `failed`)
- Soft delete: chỉ query dòng chưa bị xóa

```sql
-- Soft delete: chỉ index dòng chưa xóa
CREATE INDEX idx_orders_active ON orders(user_id, created_at)
    WHERE status != 'cancelled';

-- Unique constraint cho soft delete (email unique trong active users)
CREATE UNIQUE INDEX idx_users_email_active ON users(email)
    WHERE status != 'deleted';
```

**Quan trọng**: Query phải có điều kiện `WHERE` **khớp chính xác** với điều kiện partial index mới dùng được.

### 3.3 Covering Index (INCLUDE)

Covering index nhét thêm cột vào index để PostgreSQL không cần đọc heap (Index-Only Scan).

```sql
-- Từ LAB 4 trong db-init.md
CREATE INDEX idx_users_country_cover ON users(country)
    INCLUDE (email, username);

-- Query này → Index Only Scan: không đọc bảng gốc
EXPLAIN ANALYZE
SELECT email, username FROM users WHERE country = 'VN';
```

**`INCLUDE` vs thêm vào key:**

| Cách                                           | Ý nghĩa                                                                                  |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `ON users(country, email, username)`          | Email và username ảnh hưởng đến sort order của index, có thể dùng để filter    |
| `ON users(country) INCLUDE (email, username)` | Email và username chỉ để đọc, không ảnh hưởng sort, kích thước leaf nhỏ hơn |

Dùng `INCLUDE` khi cột thêm vào **không cần filter hay ORDER BY**, chỉ cần SELECT.

**Điều kiện để Index-Only Scan hoạt động**: các page trong heap phải được đánh dấu **all-visible** trong visibility map (VACUUM phải chạy sau lần ghi cuối).

### 3.4 Expression / Functional Index

Index trên biểu thức cho phép PostgreSQL sử dụng index khi query có hàm bọc cột.

```sql
-- Case-insensitive email lookup
CREATE INDEX idx_users_email_lower ON users(lower(email));

-- Query phải dùng đúng expression:
SELECT * FROM users WHERE lower(email) = 'user12345@gmail.com';

-- Date truncation — group by day hiệu quả
CREATE INDEX idx_orders_day ON orders(date_trunc('day', created_at));

SELECT count(*) FROM orders
WHERE date_trunc('day', created_at) = '2024-01-15';

-- JSON field
CREATE INDEX idx_users_city ON users((metadata->>'city'));
```

**Lưu ý**: Expression phải **deterministic** (cùng input → cùng output). Không dùng `random()`, `now()`.

### 3.5 Unique Index

```sql
-- PostgreSQL tự tạo unique index khi có PRIMARY KEY hoặc UNIQUE constraint
-- Nhưng có thể tạo thủ công để thêm WHERE clause:
CREATE UNIQUE INDEX idx_products_sku ON products(sku) WHERE status = 'active';
```

---

## 4. Đọc EXPLAIN ANALYZE

### 4.1 Các loại Scan và khi nào PostgreSQL chọn từng loại

#### Sequential Scan (Seq Scan)

PostgreSQL đọc toàn bộ heap từ đầu đến cuối. Chọn khi:

- Không có index phù hợp
- Tỷ lệ dòng trả về cao (ví dụ: lọc `status = 'active'` → 80% bảng)
- Bảng nhỏ (cost của random I/O cao hơn seq scan)

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'paid';
-- Seq Scan on orders (cost=0.00..52000.00 rows=500000 ...)
-- Vì 'paid' chiếm ~25% bảng → planner thấy index không đáng
```

#### Index Scan

PostgreSQL traverse B+Tree, lấy TID, rồi đọc từng heap page theo TID. Chọn khi:

- Selectivity cao (ít dòng trả về)
- Correlation tốt

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user12345@gmail.com';
-- Index Scan using idx_users_email on users
--   (cost=0.42..8.44 rows=1 width=65)
--   Index Cond: (email = 'user12345@gmail.com')
```

#### Index-Only Scan

Tất cả cột cần thiết có trong index, không cần đọc heap. Nhanh nhất.

```sql
EXPLAIN ANALYZE SELECT email, username FROM users WHERE country = 'VN';
-- Index Only Scan using idx_users_country_cover on users
--   Heap Fetches: 0  ← không đọc heap nếu all-visible
```

#### Bitmap Heap Scan

PostgreSQL thu thập tất cả TID từ index vào một bitmap, sort theo page number, rồi đọc heap theo thứ tự. Chọn khi:

- Selectivity trung bình (vài % đến vài chục %)
- Nhiều điều kiện kết hợp với `BitmapAnd` / `BitmapOr`

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE age BETWEEN 25 AND 30;
-- Bitmap Heap Scan on users
--   ->  Bitmap Index Scan on idx_users_age
--         Index Cond: ((age >= 25) AND (age <= 30))
--   Recheck Cond: ((age >= 25) AND (age <= 30))
```

### 4.2 Đọc các trường quan trọng

```
Seq Scan on orders  (cost=0.00..52341.00 rows=2000000 width=38)
                     ^start_cost  ^total_cost  ^est_rows  ^row_size

                    (actual time=0.023..892.451 rows=2000000 loops=1)
                              ^first_row_ms  ^last_row_ms ^actual_rows ^executions

Buffers: shared hit=28947 read=2345
         ^từ cache          ^từ disk
```

| Trường          | Ý nghĩa                                                                  |
| ----------------- | -------------------------------------------------------------------------- |
| `cost=X..Y`     | X: chi phí lấy row đầu tiên; Y: tổng chi phí (đơn vị tùy ý)    |
| `rows`          | Số dòng planner ước tính                                              |
| `actual rows`   | Số dòng thực tế —**nếu lệch nhiều → statistics lỗi thời** |
| `loops`         | Node chạy bao nhiêu lần (nested loop)                                   |
| `Buffers: hit`  | Page đọc từ shared buffer cache (RAM)                                   |
| `Buffers: read` | Page đọc từ disk — nhiều là dấu hiệu cache miss                    |

### 4.3 Nhận biết plan xấu

```sql
-- Kích hoạt thêm thông tin
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, o.total, oi.quantity
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.user_id = 5000;
```

Dấu hiệu plan có vấn đề:

- `rows=1` nhưng `actual rows=50000` → estimate sai hoàn toàn → chạy `ANALYZE`
- `Nested Loop` với `loops=2000000` → thiếu index trên cột JOIN
- `Seq Scan` trên bảng lớn khi bạn chắc chắn query selective → kiểm tra index có tồn tại không
- `cost` rất cao nhưng `actual time` thấp → planner sử dụng plan không tốt nhất

---

## 5. Statistics & Query Planner

### 5.1 pg_stats — Cơ sở dữ liệu của Planner

PostgreSQL planner ước tính số dòng trả về dựa trên **statistics** được lưu trong `pg_stats`.

```sql
-- Xem statistics của cột status trong bảng orders
SELECT
    attname,
    n_distinct,          -- số giá trị unique (-1 = unique hoàn toàn)
    most_common_vals,    -- giá trị phổ biến nhất
    most_common_freqs,   -- tần suất của mỗi giá trị
    histogram_bounds     -- phân phối dữ liệu cho range queries
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

### 5.2 ANALYZE — Cập nhật Statistics

```sql
-- Chạy sau bulk insert (như trong db-init.md BƯỚC 2)
ANALYZE users;
ANALYZE orders;

-- ANALYZE toàn database
ANALYZE;

-- Autovacuum chạy ANALYZE tự động khi bảng thay đổi > 20% (mặc định)
-- Kiểm tra khi nào lần cuối được analyze:
SELECT relname, last_analyze, last_autoanalyze, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE relname IN ('users', 'orders');
```

### 5.3 Tăng Statistics Target cho dữ liệu lệch

```sql
-- Mặc định: default_statistics_target = 100 (sample ~30k dòng)
-- Với bảng có phân phối rất lệch, tăng lên:
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders;
-- Planner có ước tính chính xác hơn cho cột 'status'
```

### 5.4 Cấu hình Planner quan trọng

```sql
-- Xem cấu hình hiện tại
SHOW random_page_cost;       -- mặc định 4.0 (HDD); giảm xuống 1.1 cho SSD/NVMe
SHOW effective_cache_size;   -- PostgreSQL ước tính RAM dành cho cache OS

-- Điều chỉnh cho SSD:
SET random_page_cost = 1.1;
SET effective_cache_size = '8GB';  -- điều chỉnh theo RAM thực tế

-- Tắt/bật để debug (KHÔNG làm trong production):
SET enable_seqscan = off;    -- bắt buộc dùng index (để test)
SET enable_indexscan = off;  -- bắt buộc dùng seq scan
```

---

## 6. Index Maintenance & Health

### 6.1 Index Bloat

Mỗi lần `UPDATE` hoặc `DELETE`, PostgreSQL không xóa entry cũ trong index ngay — để lại **dead tuple**.
Theo thời gian, index phình to (bloat) → scan chậm hơn và tốn RAM hơn.

```sql
-- Ước tính bloat của index (cần extension pgstattuple hoặc query thủ công)
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT * FROM pgstatindex('idx_orders_user_status');
-- leaf_fragmentation: tỷ lệ % page rỗng/lãng phí
-- avg_leaf_density: mật độ data trong leaf page (thấp = bloat cao)
```

### 6.2 VACUUM và AUTOVACUUM

```sql
-- VACUUM: dọn dead tuples, cập nhật visibility map (cho Index-Only Scan)
VACUUM users;

-- VACUUM FULL: compact lại toàn bộ bảng, giải phóng disk — LOCKS TABLE
-- Chỉ dùng khi bloat rất nghiêm trọng và có maintenance window
VACUUM FULL users;

-- Kiểm tra autovacuum hoạt động có đúng không:
SELECT
    relname,
    n_dead_tup,
    n_live_tup,
    round(100.0 * n_dead_tup / nullif(n_live_tup, 0), 1) AS dead_ratio,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE relname IN ('users', 'orders', 'order_items')
ORDER BY n_dead_tup DESC;
```

### 6.3 REINDEX CONCURRENTLY — Rebuild không lock

```sql
-- Rebuild index mà không block read/write (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_orders_user_status;

-- Rebuild toàn bộ index của bảng
REINDEX TABLE CONCURRENTLY orders;
```

### 6.4 CREATE INDEX CONCURRENTLY — Tạo index trong production

```sql
-- KHÔNG dùng (lock table):
CREATE INDEX idx_users_score ON users(score);

-- LUÔN dùng trong production:
CREATE INDEX CONCURRENTLY idx_users_score ON users(score);
-- Build index ở background, không block đọc/ghi
-- Nhược điểm: chậm hơn, không thể chạy trong transaction block
```

### 6.5 Monitoring Index Health

```sql
-- Tổng quan tất cả index (mở rộng từ BƯỚC 4 trong db-init.md)
SELECT
    s.tablename,
    s.indexname,
    pg_size_pretty(pg_relation_size(s.indexrelid)) AS index_size,
    s.idx_scan                                      AS scans,
    s.idx_tup_read                                  AS tuples_read,
    s.idx_tup_fetch                                 AS tuples_fetched,
    round(100.0 * s.idx_tup_fetch / nullif(s.idx_tup_read, 0), 1) AS fetch_ratio
FROM pg_stat_user_indexes s
ORDER BY pg_relation_size(s.indexrelid) DESC;

-- Index không bao giờ được dùng → cân nhắc xóa
SELECT indexname, tablename, pg_size_pretty(pg_relation_size(indexrelid)) AS wasted_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelid NOT IN (SELECT conindid FROM pg_constraint WHERE contype IN ('p','u'))
ORDER BY pg_relation_size(indexrelid) DESC;

-- Xóa index không dùng (sau khi xác nhận):
-- DROP INDEX CONCURRENTLY idx_ten_index_khong_dung;
```

---

## 7. Anti-patterns & Sai lầm thường gặp

### 7.1 Viết Query Không Dùng Được Index (SARGability)

**SARG** = Search ARGument — điều kiện WHERE mà index có thể áp dụng trực tiếp.
Các pattern dưới đây làm PostgreSQL bỏ qua index dù index đã tồn tại.

#### Bọc cột trong function

```sql
-- XẤU: index bị bỏ qua hoàn toàn
WHERE UPPER(email) = 'USER12345@GMAIL.COM'

-- TỐT: dùng expression index
CREATE INDEX idx_users_email_upper ON users(upper(email));
WHERE upper(email) = 'USER12345@GMAIL.COM'

-- HOẶC: nếu email luôn lowercase, so sánh trực tiếp
WHERE email = lower('USER12345@GMAIL.COM')
```

#### Implicit type cast

```sql
-- XẤU: user_id là INT nhưng so sánh với TEXT → PostgreSQL không dùng index
WHERE user_id = '1234'

-- TỐT:
WHERE user_id = 1234
```

#### Phép tính trên cột

```sql
-- XẤU: đẩy phép tính lên cột → index scan không áp dụng được
WHERE created_at + interval '1 day' > now()

-- TỐT: đảo phép tính sang vế phải
WHERE created_at > now() - interval '1 day'
```

#### LIKE với wildcard đầu

```sql
-- XẤU: B-Tree không index wildcard-prefix pattern
WHERE description LIKE '%wireless%'

-- TỐT: dùng GIN + pg_trgm
CREATE INDEX idx_products_name_trgm ON products USING GIN(name gin_trgm_ops);
WHERE name ILIKE '%wireless%'

-- HOẶC: Full-Text Search cho ngôn ngữ tự nhiên
WHERE to_tsvector('english', description) @@ to_tsquery('wireless')
```

#### OR thay vì IN

```sql
-- XẤU: OR thường ngăn index sử dụng hiệu quả
SELECT * FROM orders WHERE status = 'paid' OR status = 'shipped';

-- TỐT: IN (PostgreSQL chuyển thành BitmapOr tự động)
SELECT * FROM orders WHERE status IN ('paid', 'shipped');

-- HOẶC: UNION ALL nếu 2 query rất khác nhau về plan
SELECT * FROM orders WHERE status = 'paid'
UNION ALL
SELECT * FROM orders WHERE status = 'shipped';
```

#### IS NOT NULL trên cột nullable

```sql
-- Có thể scan nhiều dòng nếu NULL ít
WHERE status IS NOT NULL

-- TỐT hơn: Partial index nếu NULL rất ít
CREATE INDEX idx_orders_status_notnull ON orders(status) WHERE status IS NOT NULL;
```

### 7.2 Over-indexing

```sql
-- Ví dụ xấu: mỗi cột đều có index riêng
CREATE INDEX ON orders(user_id);
CREATE INDEX ON orders(status);
CREATE INDEX ON orders(created_at);
CREATE INDEX ON orders(total);
-- 4 index riêng + 1 PRIMARY KEY → mỗi INSERT vào orders cập nhật 5 structures

-- Tốt hơn: phân tích query patterns thực tế và tạo composite index
-- Nếu 90% query là: WHERE user_id = ? AND status = ?
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
-- Index này cũng cover: WHERE user_id = ? (leftmost prefix)
```

### 7.3 Index trên cột low-cardinality

```sql
-- Cột status chỉ có 4 giá trị → B-Tree index ít có ích
-- PostgreSQL thường Seq Scan vì dù có index cũng phải đọc 25% bảng

-- Trường hợp đặc biệt: low-cardinality + cần lọc subset nhỏ → Partial Index
CREATE INDEX idx_orders_cancelled ON orders(user_id, created_at)
    WHERE status = 'cancelled';  -- nếu cancelled << tổng số orders
```

### 7.4 Quên Foreign Key Index

PostgreSQL tự tạo index cho PRIMARY KEY và UNIQUE constraint, nhưng **KHÔNG tự tạo index cho FOREIGN KEY**.

```sql
-- Khi DELETE một user, PostgreSQL kiểm tra orders.user_id
-- Nếu không có index → Seq Scan toàn bộ orders (2M dòng!)
-- Tương tự cho JOIN queries

-- Luôn index cột FK:
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);
```

### 7.5 Quên ANALYZE sau bulk load

```sql
-- Sau khi bulk insert như trong db-init.md BƯỚC 2,
-- statistics chưa được cập nhật → planner ước tính sai → chọn plan sai
INSERT INTO orders ... (2.000.000 dòng);
ANALYZE orders;  -- BẮT BUỘC chạy sau khi load xong
```

### 7.6 NULL và Index

B-Tree index lưu NULL values (khác Oracle). Nhưng:

```sql
-- IS NULL và IS NOT NULL đều có thể dùng B-Tree index
-- Tuy nhiên nếu NULL nhiều và bạn chỉ query NOT NULL:
CREATE INDEX idx_orders_paid_total ON orders(total)
    WHERE total IS NOT NULL;
-- Nhỏ hơn và nhanh hơn nếu NULL chiếm nhiều dòng
```

---

## 8. Practical Checklist cho Backend Engineer

### Trước khi thêm Index

```sql
-- 1. Kiểm tra index hiện có
\d orders   -- psql: hiện tất cả index của bảng

-- 2. Chạy EXPLAIN không có index để hiểu plan hiện tại
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;

-- 3. Xác nhận index chưa tồn tại
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'orders';
```

### Sau khi thêm Index

```sql
-- 1. Cập nhật statistics
ANALYZE orders;

-- 2. Verify plan đã thay đổi
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
-- Kiểm tra: scan type thay đổi? actual time giảm?

-- 3. Theo dõi sau khi có traffic thực
SELECT indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE tablename = 'orders' AND indexname = 'idx_orders_user_status';
-- idx_scan tăng → index đang được dùng
```

### Checklist triển khai Production

- [ ] Dùng `CREATE INDEX CONCURRENTLY` — không bao giờ tạo index blocking trong production
- [ ] Test plan trên staging với data volume tương tự production
- [ ] Chạy `ANALYZE` sau bulk load hoặc migration
- [ ] Monitor `pg_stat_user_indexes` sau 24h để xác nhận index được dùng
- [ ] Định kỳ kiểm tra unused indexes (idx_scan = 0) và xóa nếu không cần
- [ ] Kiểm tra autovacuum chạy đúng: `SELECT last_autovacuum FROM pg_stat_user_tables`
- [ ] Với SSD/NVMe: set `random_page_cost = 1.1` và `effective_cache_size` đúng

### Maintenance Schedule gợi ý

| Tần suất         | Việc cần làm                                                         |
| ------------------ | ----------------------------------------------------------------------- |
| Sau mỗi bulk load | `ANALYZE <table>`                                                     |
| Hàng tuần        | Kiểm tra index bloat và unused indexes                                |
| Hàng tháng       | Review slow query log, xem xét thêm/xóa index                        |
| Khi bloat > 30%    | `REINDEX CONCURRENTLY` hoặc `VACUUM FULL` trong maintenance window |

---

## Tài liệu tham khảo thêm

- **`docs/db-init.md`** — SQL thực hành cho tất cả các loại index (LAB 1–6)
- `pg_stat_user_indexes` — monitoring index usage trong PostgreSQL
- `pg_stats` — thống kê dùng bởi query planner
- PostgreSQL docs: [Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
- PostgreSQL docs: [Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
