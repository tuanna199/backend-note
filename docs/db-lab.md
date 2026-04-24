-- ============================================================
--  PostgreSQL Index Practice — Dữ liệu thực hành
--  Mục tiêu: hiểu rõ từng loại index qua EXPLAIN ANALYZE
-- ============================================================

-- ============================================================
--  BƯỚC 1: TẠO TABLES
-- ============================================================

DROP TABLE IF EXISTS order_items CASCADE;
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS products CASCADE;
DROP TABLE IF EXISTS users CASCADE;

CREATE TABLE users (
    id          SERIAL PRIMARY KEY,
    email       TEXT NOT NULL,
    username    TEXT NOT NULL,
    age         INT,
    country     TEXT,
    status      TEXT DEFAULT 'active',   -- 'active' | 'inactive' | 'banned'
    score       NUMERIC(8,2),
    created_at  TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE products (
    id          SERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    description TEXT,                    -- dùng để test Full-Text index
    category    TEXT,
    price       NUMERIC(10,2),
    stock       INT DEFAULT 0,
    created_at  TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    user_id     INT REFERENCES users(id),
    status      TEXT DEFAULT 'pending',  -- 'pending' | 'paid' | 'shipped' | 'cancelled'
    total       NUMERIC(10,2),
    created_at  TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE order_items (
    id          SERIAL PRIMARY KEY,
    order_id    INT REFERENCES orders(id),
    product_id  INT REFERENCES products(id),
    quantity    INT,
    unit_price  NUMERIC(10,2)
);

-- ============================================================
--  BƯỚC 2: SINH DỮ LIỆU GIẢ
--  users: 500.000 dòng  → thấy rõ B-Tree, Hash, Partial
--  products: 50.000 dòng → thấy Full-Text
--  orders: 2.000.000 dòng → thấy Composite index
--  order_items: ~6.000.000 dòng → thấy JOIN performance
-- ============================================================

-- Tắt autovacuum tạm thời để insert nhanh hơn
ALTER TABLE users SET (autovacuum_enabled = false);
ALTER TABLE products SET (autovacuum_enabled = false);
ALTER TABLE orders SET (autovacuum_enabled = false);
ALTER TABLE order_items SET (autovacuum_enabled = false);

-- ---- users: 500.000 dòng ----
INSERT INTO users (email, username, age, country, status, score, created_at)
SELECT
    'user' || i || '@' ||
        (ARRAY['gmail.com','yahoo.com','outlook.com','hotmail.com'])[1 + (i % 4)] AS email,

    'user_' || i AS username,

    18 + (i % 62) AS age,   -- tuổi 18–79

    (ARRAY['VN','US','JP','KR','SG','TH','ID','MY'])[1 + (i % 8)] AS country,

    -- 80% active, 15% inactive, 5% banned → dùng để test Partial Index
    CASE
        WHEN i % 20 = 0 THEN 'banned'
        WHEN i % 7  = 0 THEN 'inactive'
        ELSE 'active'
    END AS status,

    ROUND((random() * 9900 + 100)::NUMERIC, 2) AS score,

    now() - (random() * INTERVAL '3 years') AS created_at

FROM generate_series(1, 500000) AS i;

-- ---- products: 50.000 dòng ----
INSERT INTO products (name, description, category, price, stock, created_at)
SELECT
    'Product ' || i AS name,

    -- description dài, có từ khóa lặp → test Full-Text Index
    'High quality ' ||
        (ARRAY['wireless','bluetooth','ergonomic','portable','smart','fast'])[1 + (i % 6)] ||
        ' product for ' ||
        (ARRAY['home use','office','gaming','travel','outdoor','professional'])[1 + (i % 6)] ||
        '. Category: ' ||
        (ARRAY['Electronics','Clothing','Books','Sports','Kitchen','Toys'])[1 + (i % 6)] AS description,

    (ARRAY['Electronics','Clothing','Books','Sports','Kitchen','Toys'])[1 + (i % 6)] AS category,

    ROUND((random() * 4900 + 100)::NUMERIC, 2) AS price,

    (random() * 500)::INT AS stock,

    now() - (random() * INTERVAL '2 years') AS created_at

FROM generate_series(1, 50000) AS i;

-- ---- orders: 2.000.000 dòng ----
INSERT INTO orders (user_id, status, total, created_at)
SELECT
    1 + (random() * 499999)::INT AS user_id,

    (ARRAY['pending','paid','shipped','cancelled'])[1 + (i % 4)] AS status,

    ROUND((random() * 4900 + 100)::NUMERIC, 2) AS total,

    now() - (random() * INTERVAL '2 years') AS created_at

FROM generate_series(1, 2000000) AS i;

-- ---- order_items: ~6.000.000 dòng (3 items per order trung bình) ----
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT
    o.id AS order_id,
    1 + (random() * 49999)::INT AS product_id,
    1 + (random() * 5)::INT AS quantity,
    ROUND((random() * 999 + 1)::NUMERIC, 2) AS unit_price
FROM orders o
CROSS JOIN generate_series(1, 3);   -- 3 items mỗi order

-- Bật lại autovacuum
ALTER TABLE users SET (autovacuum_enabled = true);
ALTER TABLE products SET (autovacuum_enabled = true);
ALTER TABLE orders SET (autovacuum_enabled = true);
ALTER TABLE order_items SET (autovacuum_enabled = true);

-- Cập nhật statistics để planner có thông tin chính xác
ANALYZE users;
ANALYZE products;
ANALYZE orders;
ANALYZE order_items;

-- ============================================================
--  BƯỚC 3: THỰC HÀNH TỪNG LOẠI INDEX
--  Workflow: chạy EXPLAIN ANALYZE TRƯỚC khi tạo index,
--            tạo index, chạy lại → so sánh
-- ============================================================

---

--  LAB 1 — B-Tree Index (mặc định)
--  Tình huống: tìm user theo email

---

-- [TRƯỚC] Chạy để thấy Seq Scan + thời gian
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user4@gmail.com';

![Seq Scan](img/screenshot-seq-scan.png)

-- Tạo B-Tree index
CREATE INDEX idx_users_email ON users(email);

-- [SAU] Chạy lại → thấy Index Scan thay thế Seq Scan
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user12345@gmail.com';

-- B-Tree cũng hỗ trợ range query
EXPLAIN ANALYZE
SELECT * FROM users WHERE age BETWEEN 25 AND 30 ORDER BY age;

CREATE INDEX idx_users_age ON users(age);

EXPLAIN ANALYZE
SELECT * FROM users WHERE age BETWEEN 25 AND 30 ORDER BY age;

---

--  LAB 2 — Composite Index (index nhiều cột)
--  Tình huống: lọc orders theo user_id + status
--  Quy tắc: cột có selectivity cao hơn đặt trước

---

EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 1234 AND status = 'paid';

-- Tạo composite index — thứ tự cột quan trọng!
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 1234 AND status = 'paid';

-- Test leftmost prefix rule:
-- Query này VẪN dùng được index (dùng phần đầu user_id)
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 1234;

-- Query này KHÔNG dùng được index (bỏ qua cột đầu)
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'paid';

---

--  LAB 3 — Partial Index
--  Tình huống: chỉ query users đang active (80% bảng)
--  → index toàn bảng lãng phí, chỉ cần index 5% bị banned

---

-- Tình huống thực: tìm banned users để review
EXPLAIN ANALYZE
SELECT * FROM users WHERE status = 'banned' AND country = 'VN';

-- Partial index: chỉ index dòng có status = 'banned'
CREATE INDEX idx_users_banned ON users(country)
    WHERE status = 'banned';

EXPLAIN ANALYZE
SELECT * FROM users WHERE status = 'banned' AND country = 'VN';

-- So sánh kích thước:
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE tablename = 'users';

---

--  LAB 4 — Covering Index (Index-Only Scan)
--  Tình huống: query chỉ cần đúng các cột có trong index
--  → PostgreSQL không cần đọc heap table, nhanh hơn nữa

---

EXPLAIN ANALYZE
SELECT email, username FROM users WHERE country = 'VN';

-- Covering index: gộp cả cột cần SELECT vào index
CREATE INDEX idx_users_country_cover ON users(country)
    INCLUDE (email, username);

-- Kết quả: "Index Only Scan" — không đọc bảng gốc nữa
EXPLAIN ANALYZE
SELECT email, username FROM users WHERE country = 'VN';

---

--  LAB 5 — Full-Text Index
--  Tình huống: tìm sản phẩm theo từ khóa trong description

---

-- Không có index: Seq Scan + ILIKE rất chậm
EXPLAIN ANALYZE
SELECT name, description FROM products
WHERE description ILIKE '%wireless%';

-- Tạo Full-Text index
CREATE INDEX idx_products_fts ON products
    USING GIN(to_tsvector('english', description));

-- Dùng cú pháp Full-Text Search
EXPLAIN ANALYZE
SELECT name, description FROM products
WHERE to_tsvector('english', description) @@ to_tsquery('wireless');

-- Tìm nhiều từ khóa
EXPLAIN ANALYZE
SELECT name FROM products
WHERE to_tsvector('english', description) @@ to_tsquery('wireless & portable');

---

--  LAB 6 — Index trên JOIN
--  Tình huống: JOIN orders với order_items

---

EXPLAIN ANALYZE
SELECT o.id, o.total, oi.quantity, oi.unit_price
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.user_id = 5000;

CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_orders_user_id ON orders(user_id);

EXPLAIN ANALYZE
SELECT o.id, o.total, oi.quantity, oi.unit_price
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.user_id = 5000;

-- ============================================================
--  BƯỚC 4: CÔNG CỤ KIỂM TRA & DIAGNOSTIC
-- ============================================================

-- Xem tất cả indexes và kích thước
SELECT
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    idx_scan   AS times_used,
    idx_tup_read AS tuples_read
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Phát hiện index không bao giờ được dùng (lãng phí)
SELECT indexname, tablename, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Xem kích thước table vs index
SELECT
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
    pg_size_pretty(pg_relation_size(oid))       AS table_size,
    pg_size_pretty(pg_total_relation_size(oid)
        - pg_relation_size(oid))                AS index_size
FROM pg_class
WHERE relkind = 'r'
  AND relname IN ('users','products','orders','order_items')
ORDER BY pg_total_relation_size(oid) DESC;

-- Reset statistics (để đo lại từ đầu)
SELECT pg_stat_reset();
