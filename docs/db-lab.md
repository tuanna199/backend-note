# PostgreSQL Index Practice

> Workflow: chạy `EXPLAIN ANALYZE` **trước** khi tạo index → tạo index → chạy lại → so sánh.
> Schema: `users` (500k), `products` (50k), `orders` (2M), `order_items` (6M).

---

## Bước 1: Tạo Tables

```sql
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
    description TEXT,
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
```

---

## Bước 2: Sinh Dữ Liệu Giả

Tắt autovacuum tạm thời để INSERT nhanh hơn, bật lại và chạy `ANALYZE` sau khi xong.

```sql
ALTER TABLE users       SET (autovacuum_enabled = false);
ALTER TABLE products    SET (autovacuum_enabled = false);
ALTER TABLE orders      SET (autovacuum_enabled = false);
ALTER TABLE order_items SET (autovacuum_enabled = false);
```

**users — 500.000 dòng** (80% active, 15% inactive, 5% banned):

```sql
INSERT INTO users (email, username, age, country, status, score, created_at)
SELECT
    'user' || i || '@' ||
        (ARRAY['gmail.com','yahoo.com','outlook.com','hotmail.com'])[1 + (i % 4)],
    'user_' || i,
    18 + (i % 62),
    (ARRAY['VN','US','JP','KR','SG','TH','ID','MY'])[1 + (i % 8)],
    CASE
        WHEN i % 20 = 0 THEN 'banned'
        WHEN i % 7  = 0 THEN 'inactive'
        ELSE 'active'
    END,
    ROUND((random() * 9900 + 100)::NUMERIC, 2),
    now() - (random() * INTERVAL '3 years')
FROM generate_series(1, 500000) AS i;
```

**products — 50.000 dòng** (description có từ khóa để test Full-Text):

```sql
INSERT INTO products (name, description, category, price, stock, created_at)
SELECT
    'Product ' || i,
    'High quality ' ||
        (ARRAY['wireless','bluetooth','ergonomic','portable','smart','fast'])[1 + (i % 6)] ||
        ' product for ' ||
        (ARRAY['home use','office','gaming','travel','outdoor','professional'])[1 + (i % 6)] ||
        '. Category: ' ||
        (ARRAY['Electronics','Clothing','Books','Sports','Kitchen','Toys'])[1 + (i % 6)],
    (ARRAY['Electronics','Clothing','Books','Sports','Kitchen','Toys'])[1 + (i % 6)],
    ROUND((random() * 4900 + 100)::NUMERIC, 2),
    (random() * 500)::INT,
    now() - (random() * INTERVAL '2 years')
FROM generate_series(1, 50000) AS i;
```

**orders — 2.000.000 dòng:**

```sql
INSERT INTO orders (user_id, status, total, created_at)
SELECT
    1 + (random() * 499999)::INT,
    (ARRAY['pending','paid','shipped','cancelled'])[1 + (i % 4)],
    ROUND((random() * 4900 + 100)::NUMERIC, 2),
    now() - (random() * INTERVAL '2 years')
FROM generate_series(1, 2000000) AS i;
```

**order_items — ~6.000.000 dòng** (3 items/order):

```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT
    o.id,
    1 + (random() * 49999)::INT,
    1 + (random() * 5)::INT,
    ROUND((random() * 999 + 1)::NUMERIC, 2)
FROM orders o
CROSS JOIN generate_series(1, 3);
```

Bật lại autovacuum và cập nhật statistics:

```sql
ALTER TABLE users       SET (autovacuum_enabled = true);
ALTER TABLE products    SET (autovacuum_enabled = true);
ALTER TABLE orders      SET (autovacuum_enabled = true);
ALTER TABLE order_items SET (autovacuum_enabled = true);

ANALYZE users;
ANALYZE products;
ANALYZE orders;
ANALYZE order_items;
```

---

## Bước 3: Labs

### LAB 1 — B-Tree Index

**Tình huống:** tìm user theo email và lọc theo age range.

```sql
-- [TRƯỚC] Seq Scan — đọc toàn bộ 500k dòng
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user4@gmail.com';

-- Tạo index
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_age   ON users(age);

-- [SAU] Index Scan — 3-4 I/O thay vì 11.000
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user12345@gmail.com';

-- B-Tree hỗ trợ range query và ORDER BY
EXPLAIN ANALYZE
SELECT * FROM users WHERE age BETWEEN 25 AND 30 ORDER BY age;
```

---

### LAB 2 — Composite Index

**Tình huống:** lọc orders theo `user_id + status`. Cột equality filter đặt trước, range filter đặt sau.

```sql
-- [TRƯỚC]
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 1234 AND status = 'paid';

-- Tạo composite index — thứ tự cột quan trọng
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- [SAU]
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 1234 AND status = 'paid';

-- Leftmost prefix: vẫn dùng được index
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 1234;

-- Bỏ qua cột đầu: KHÔNG dùng được index → Seq Scan
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'paid';
```

---

### LAB 3 — Partial Index

**Tình huống:** chỉ query `banned` users (5% bảng) → full index lãng phí, partial index nhỏ hơn 20x.

```sql
-- [TRƯỚC]
EXPLAIN ANALYZE
SELECT * FROM users WHERE status = 'banned' AND country = 'VN';

-- Partial index: chỉ index dòng có status = 'banned'
CREATE INDEX idx_users_banned ON users(country)
    WHERE status = 'banned';

-- [SAU] — query phải có WHERE status = 'banned' để dùng được index
EXPLAIN ANALYZE
SELECT * FROM users WHERE status = 'banned' AND country = 'VN';

-- So sánh kích thước index
SELECT indexname, pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE tablename = 'users';
```

---

### LAB 4 — Covering Index (Index-Only Scan)

**Tình huống:** query chỉ cần một vài cột → nhét cột đó vào index, PostgreSQL không cần đọc heap.

```sql
-- [TRƯỚC]
EXPLAIN ANALYZE
SELECT email, username FROM users WHERE country = 'VN';

-- Covering index: INCLUDE các cột cần SELECT
CREATE INDEX idx_users_country_cover ON users(country)
    INCLUDE (email, username);

-- [SAU] "Index Only Scan" — Heap Fetches: 0
EXPLAIN ANALYZE
SELECT email, username FROM users WHERE country = 'VN';
```

> **Lưu ý:** Index-Only Scan cần heap pages được đánh dấu all-visible (VACUUM phải chạy sau lần ghi cuối).

---

### LAB 5 — Full-Text Index (GIN)

**Tình huống:** tìm sản phẩm theo từ khóa trong `description`.

```sql
-- [TRƯỚC] ILIKE không dùng được B-Tree → Seq Scan
EXPLAIN ANALYZE
SELECT name, description FROM products
WHERE description ILIKE '%wireless%';

-- Tạo GIN Full-Text index
CREATE INDEX idx_products_fts ON products
    USING GIN(to_tsvector('english', description));

-- [SAU] Dùng cú pháp Full-Text Search
EXPLAIN ANALYZE
SELECT name, description FROM products
WHERE to_tsvector('english', description) @@ to_tsquery('wireless');

-- Tìm nhiều từ khóa (AND)
EXPLAIN ANALYZE
SELECT name FROM products
WHERE to_tsvector('english', description) @@ to_tsquery('wireless & portable');
```

---

### LAB 6 — Index trên Foreign Key (JOIN)

**Tình huống:** JOIN `orders` với `order_items` — thiếu index FK → Nested Loop O(n²).

```sql
-- [TRƯỚC] Nested Loop không có index
EXPLAIN ANALYZE
SELECT o.id, o.total, oi.quantity, oi.unit_price
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.user_id = 5000;

-- Tạo index trên FK (PostgreSQL KHÔNG tự tạo)
CREATE INDEX idx_orders_user_id        ON orders(user_id);
CREATE INDEX idx_order_items_order_id  ON order_items(order_id);

-- [SAU] Hash Join / Merge Join với index
EXPLAIN ANALYZE
SELECT o.id, o.total, oi.quantity, oi.unit_price
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
WHERE o.user_id = 5000;
```

---

## Bước 4: Diagnostic Queries

**Tổng quan index — kích thước và tần suất dùng:**

```sql
SELECT
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    idx_scan     AS times_used,
    idx_tup_read AS tuples_read
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Phát hiện index không bao giờ được dùng:**

```sql
SELECT indexname, tablename, pg_size_pretty(pg_relation_size(indexrelid)) AS wasted_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelid NOT IN (SELECT conindid FROM pg_constraint WHERE contype IN ('p','u'))
ORDER BY pg_relation_size(indexrelid) DESC;
```

**So sánh kích thước table vs index:**

```sql
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
```

**Reset statistics để đo lại từ đầu:**

```sql
SELECT pg_stat_reset();
```
