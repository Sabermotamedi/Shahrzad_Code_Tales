# سطح ۳ — توضیح فنی: Databases ⚙️

> دیگر قصه‌ای در کار نیست. این سند مرجع فنی است — همان چیزهایی که در مصاحبه و کار واقعی لازم داری. مثال‌ها با طعمِ **PostgreSQL** هستند. هر بخش با معادل قصه‌ای‌اش شروع می‌شود تا ذهنت وصل بماند.

---

## ۱. SQL — پرس‌وجو از داده

### JOINها (چسباندنِ قفسه‌ها)

JOIN ردیف‌های دو (یا چند) جدول را بر اساس یک شرطِ تطبیق (`ON`) ترکیب می‌کند.

| نوع | چه برمی‌گرداند | کاربرد رایج |
|---|---|---|
| `INNER JOIN` | فقط ردیف‌هایی که در **هر دو** طرف تطبیق دارند | حالت پیش‌فرض و رایج |
| `LEFT [OUTER] JOIN` | **همه‌ی** ردیف‌های جدولِ چپ + تطبیقِ راست (یا `NULL`) | «همه‌ی کاربران، حتی آن‌ها که سفارش ندارند» |
| `RIGHT [OUTER] JOIN` | همه‌ی راست + تطبیقِ چپ (یا `NULL`) | عملاً همان LEFT با جای عوض‌شده |
| `FULL OUTER JOIN` | همه‌ی ردیف‌های هر دو طرف، تطبیق‌نشده‌ها با `NULL` | پیدا کردن ردیف‌های بی‌جفت در هر دو طرف |
| `CROSS JOIN` | حاصل‌ضرب دکارتی (هر ردیف × هر ردیف) | تولید ترکیب‌ها — مراقب انفجار اندازه باش |

```sql
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;     -- کاربرانِ بی‌سفارش با total = NULL می‌آیند
```

> نکته‌ی مصاحبه: «کاربرانی که **هیچ** سفارشی نداده‌اند» = `LEFT JOIN ... WHERE o.id IS NULL` (الگوی anti-join). شرطِ تطبیق در `ON` با شرطِ فیلتر در `WHERE` در OUTER JOIN فرق دارد: شرطِ روی جدولِ بیرونی اگر در `WHERE` بیاید، OUTER را عملاً به INNER تبدیل می‌کند.

### Aggregation و GROUP BY / HAVING

توابعِ خلاصه‌کننده: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`. `GROUP BY` ردیف‌ها را به دسته می‌شکند و تابعِ خلاصه روی هر دسته اجرا می‌شود.

```sql
SELECT user_id, COUNT(*) AS cnt, SUM(total) AS revenue
FROM orders
WHERE status = 'paid'          -- ① فیلتر روی ردیف‌ها، قبل از گروه‌بندی
GROUP BY user_id
HAVING SUM(total) > 1000       -- ② فیلتر روی دسته‌ها، بعد از گروه‌بندی
ORDER BY revenue DESC;
```

**ترتیبِ منطقیِ اجرا** (نه ترتیبِ نوشتن!):
```
FROM/JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
```

> نکته‌ی مصاحبه: `WHERE` نمی‌تواند تابعِ aggregate داشته باشد (چون قبل از گروه‌بندی اجرا می‌شود)؛ برای فیلترِ روی نتیجه‌ی aggregate از `HAVING` استفاده کن. هر ستونِ غیرaggregate در `SELECT` باید در `GROUP BY` هم باشد.

### Subqueryها

| نوع | یعنی | مثال |
|---|---|---|
| Scalar | یک مقدار برمی‌گرداند | `WHERE price > (SELECT AVG(price) FROM products)` |
| `IN` / `NOT IN` | مجموعه‌ای از مقادیر | `WHERE id IN (SELECT user_id FROM orders)` |
| `EXISTS` | فقط وجود/عدم وجود (بولی) | `WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id)` |
| Correlated | به ردیفِ بیرونی **وابسته** است، به ازای هر ردیف اجرا می‌شود | همان `EXISTS` بالا — `o.user_id = u.id` |
| Derived / CTE | یک جدولِ موقت | `WITH t AS (...) SELECT ... FROM t` |

> نکته‌ی مصاحبه: `NOT IN` با وجودِ `NULL` در زیرمجموعه، نتیجه‌ی عجیب (هیچ ردیف) می‌دهد؛ `NOT EXISTS` امن‌تر است. Correlated subquery می‌تواند کُند باشد چون per-row اجرا می‌شود — معمولاً با JOIN قابل بازنویسی است.

---

## ۲. Data Modeling — مدل‌سازی داده

### Normalization (نرمال‌سازی)

هدف: حذفِ افزونگی (redundancy) تا هر واقعیت فقط **یک‌جا** ذخیره شود و از Anomalyهای insert/update/delete جلوگیری شود.

| فرم | شرط | نقضِ نمونه |
|---|---|---|
| **1NF** | مقادیرِ اتمی (تک‌مقداری)، بدون گروهِ تکراری | ستونِ `phones = '0912..., 0935...'` در یک خانه |
| **2NF** | 1NF + هیچ ستونِ غیرکلیدی به **بخشی** از کلیدِ مرکب وابسته نباشد | در کلیدِ `(order_id, product_id)`، ستونِ `product_name` فقط به `product_id` وابسته است |
| **3NF** | 2NF + هیچ ستونِ غیرکلیدی به ستونِ غیرکلیدیِ دیگر وابسته نباشد (transitive) | `zip → city` در همان جدول — `city` به `zip` وابسته است نه به کلید |
| **BCNF** | نسخه‌ی سفت‌تر 3NF؛ هر determinant باید کلیدِ کاندید باشد | حالت‌های لبه‌ایِ کلیدهای چندگانه |

> نکته‌ی مصاحبه: یادگیریِ سریع — «هر ستونِ غیرکلیدی به **کلید**، به **کلِ** کلید، و به **هیچ‌چیز جز** کلید وابسته باشد» (the key, the whole key, nothing but the key). **Denormalization** عمدیِ معکوس است: افزودنِ افزونگیِ کنترل‌شده برای سرعتِ خواندن (مثلاً ذخیره‌ی `comment_count`) — با هزینه‌ی پیچیدگیِ نگه‌داریِ سازگاری.

### Relationships (روابط)

| رابطه | پیاده‌سازی | مثال |
|---|---|---|
| **1:1** | FK + قیدِ `UNIQUE` (یا کلید مشترک) | `users` ↔ `user_profiles` |
| **1:N** | FK در سمتِ «N» | `authors`(1) ↔ `books`(N): `books.author_id` |
| **N:M** | **جدولِ واسط** (junction/join table) با دو FK، PK مرکب | `students` ↔ `courses` از طریق `enrollments(student_id, course_id)` |

```sql
CREATE TABLE enrollments (
    student_id INT NOT NULL REFERENCES students(id) ON DELETE CASCADE,
    course_id  INT NOT NULL REFERENCES courses(id),
    enrolled_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (student_id, course_id)        -- جلوی ثبت‌نامِ تکراری را می‌گیرد
);
```

> نکته‌ی مصاحبه: `ON DELETE CASCADE` در مقابل `RESTRICT`/`SET NULL` — رفتارِ حذفِ والد روی فرزندان. قیدهای دیگر: `NOT NULL`, `UNIQUE`, `CHECK`, `DEFAULT`. این‌ها همان «C» در ACID را در سطحِ schema تضمین می‌کنند.

---

## ۳. Performance — کارایی

### Indexing

ایندکس ساختارِ کمکیِ **مرتب** است که جست‌وجو را از `O(n)` (Seq Scan) به `O(log n)` می‌رساند، به قیمتِ فضای دیسک و کُندتر شدنِ write (هر INSERT/UPDATE باید ایندکس را هم به‌روز کند).

| نوع | کاربرد |
|---|---|
| **B-tree** (پیش‌فرض) | تساوی و بازه (`=`, `<`, `>`, `BETWEEN`, `ORDER BY`, `LIKE 'abc%'`) |
| **Composite** (چندستونی) | کوئری‌های چندشرطی؛ تابعِ **قانونِ leftmost-prefix** |
| **Covering** | همه‌ی ستون‌های موردنیازِ کوئری در خودِ ایندکس → **index-only scan** |
| Hash | فقط تساوی `=` |
| GIN / GiST | جست‌وجوی متن کامل، JSONB، داده‌ی مکانی، آرایه‌ها |
| Partial | `WHERE` روی زیرمجموعه — `... WHERE active` (ایندکسِ کوچک‌تر) |

**Composite و leftmost-prefix:** ایندکسِ `(a, b, c)` می‌تواند برای `WHERE a=?`، `WHERE a=? AND b=?`، و `WHERE a=? AND b=? AND c=?` استفاده شود — اما برای `WHERE b=?` به‌تنهایی **نه** (چون از سمتِ چپ شروع نشده). ترتیبِ ستون‌ها در composite حیاتی است.

**Covering index (index-only scan):** اگر همه‌ی ستون‌هایی که کوئری لازم دارد در ایندکس باشند، دیتابیس اصلاً به جدولِ اصلی (heap) دست نمی‌زند:

```sql
-- کوئری: SELECT email FROM users WHERE tenant_id = 42;
CREATE INDEX idx_users_tenant ON users(tenant_id) INCLUDE (email);
--                                     ▲ کلیدِ جست‌وجو      ▲ ستونِ همراه برای covering
```

> نکته‌ی مصاحبه: ایندکس روی ستونِ **کم‌تنوع** (low cardinality مثل `is_active` بولی) معمولاً بی‌فایده است — planner ترجیح می‌دهد Seq Scan کند. ایندکس روی ستونی که در `WHERE`/`JOIN`/`ORDER BY` می‌آید و **انتخابگری بالا** (high selectivity) دارد بیشترین سود را می‌دهد. اعمالِ تابع روی ستون (`WHERE lower(email)=...`) ایندکسِ معمولی را بی‌اثر می‌کند مگر **functional index** بسازی.

### Query Optimization با EXPLAIN

`EXPLAIN` پلنِ تخمینیِ planner را نشان می‌دهد؛ `EXPLAIN ANALYZE` واقعاً کوئری را **اجرا** می‌کند و زمان و تعدادِ ردیفِ **واقعی** را هم می‌دهد.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 42;
```

نشانه‌هایی که باید بشناسی:

| در پلن | یعنی | معمولاً خوب/بد |
|---|---|---|
| `Seq Scan` | خواندنِ کلِ جدول | بد روی جدولِ بزرگ با فیلترِ انتخابگر → ایندکس بساز |
| `Index Scan` / `Index Only Scan` | استفاده از ایندکس | خوب |
| `Bitmap Heap Scan` | ترکیبِ چند ایندکس | معمولاً خوب |
| `Nested Loop` | برای جوینِ مجموعه‌های کوچک | روی مجموعه‌ی بزرگ بد |
| `Hash Join` / `Merge Join` | جوینِ مجموعه‌های بزرگ | معمولاً خوب |
| `rows=` تخمینی≠واقعی | آمارِ planner کهنه است | `ANALYZE` بزن |

> نکته‌ی مصاحبه: اختلافِ زیادِ بینِ `rows` تخمینی و `actual rows` در `EXPLAIN ANALYZE` یعنی آمار قدیمی است → `ANALYZE table;` (یا autovacuum). مراقبِ هزینه‌ی `EXPLAIN ANALYZE` روی `UPDATE/DELETE` باش — واقعاً اجرا می‌شود؛ داخلِ یک تراکنش و `ROLLBACK` بزن.

---

## ۴. Reliability — قابلیت اطمینان

### Transaction و ACID

تراکنش واحدی از کار است که یا کامل ثبت می‌شود (`COMMIT`) یا کاملاً بی‌اثر می‌شود (`ROLLBACK`).

| حرف | تضمین | مکانیزم در PostgreSQL |
|---|---|---|
| **Atomicity** | همه یا هیچ | WAL + rollback |
| **Consistency** | قیدها و قوانین همیشه برقرار | constraints, triggers |
| **Isolation** | تراکنش‌های هم‌زمان روی هم اثرِ ناخواسته نگذارند | MVCC + lockها |
| **Durability** | بعد از commit، حتی با قطعِ برق گم نشود | WAL (Write-Ahead Log) + fsync |

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;        -- اگر بینِ دو UPDATE خطا رخ دهد → ROLLBACK، هیچ پولی گم/تکثیر نمی‌شود
```

### Isolation Levels و Anomalyها

سه آشوبِ کلاسیک:
- **Dirty Read**: خواندنِ تغییری که هنوز commit نشده (و شاید rollback شود).
- **Non-Repeatable Read**: خواندنِ دوباره‌ی **همان ردیف** نتیجه‌ی متفاوت می‌دهد (یکی وسط، آن ردیف را update/delete کرد).
- **Phantom Read**: اجرای دوباره‌ی **همان کوئریِ بازه‌ای** ردیف‌های جدید/کم‌تری برمی‌گرداند (یکی insert/delete کرد).

جدولِ استانداردِ ANSI:

| سطح | Dirty Read | Non-Repeatable | Phantom |
|---|---|---|---|
| READ UNCOMMITTED | ممکن | ممکن | ممکن |
| READ COMMITTED | جلوگیری | ممکن | ممکن |
| REPEATABLE READ | جلوگیری | جلوگیری | ممکن (در ANSI) |
| SERIALIZABLE | جلوگیری | جلوگیری | جلوگیری |

> نکته‌ی مصاحبه (طلاییِ PostgreSQL): این جدول **استاندارد** است، اما PostgreSQL متفاوت رفتار می‌کند:
> - فقط **۳ سطحِ واقعی** دارد؛ اگر `READ UNCOMMITTED` بخواهی، عملاً `READ COMMITTED` می‌گیری — **PostgreSQL هرگز dirty read نمی‌دهد**.
> - پیش‌فرض = `READ COMMITTED`.
> - `REPEATABLE READ` در Postgres با **snapshot isolation** پیاده شده و **phantom read را هم جلوگیری می‌کند** (برخلافِ ANSI). اما **write-skew** را نمی‌گیرد.
> - `SERIALIZABLE` با **SSI** پیاده شده و ممکن است تراکنش را با خطای `serialization_failure` (SQLSTATE **40001**) لغو کند → اپلیکیشن باید **retry** کند.

> نکته‌ی مصاحبه: قفل‌ها دو نوعند — **pessimistic** (`SELECT ... FOR UPDATE` ردیف را قفل می‌کند) و **optimistic** (نسخه/version را چک می‌کنی و در تضاد دوباره تلاش می‌کنی). **Deadlock** وقتی دو تراکنش قفل‌های هم را برعکس می‌گیرند رخ می‌دهد؛ Postgres یکی را قربانی و لغو می‌کند — همیشه قفل‌ها را به **ترتیبِ ثابت** بگیر.

---

## ۵. Scalability — مقیاس‌پذیری

### Replication (تکثیر)

کپیِ کاملِ داده روی چند سرور. یک **Primary** (نوشتن) و یک یا چند **Replica/Standby** (خواندن).

| روش | یعنی | بده‌بستان |
|---|---|---|
| **Synchronous** | commit تا وقتی replica تأیید نکند، تمام‌شده نیست | بدونِ data loss، اما تأخیرِ بیشترِ write |
| **Asynchronous** | primary بلافاصله commit می‌کند، بعد به replica می‌رساند | سریع، اما **replication lag** و خطرِ از دست رفتنِ آخرین تراکنش‌ها در failover |

- **Read replica**: کوئریهای خواندن را به replicaها می‌فرستی تا بارِ primary کم شود.
- **خطرِ stale read**: کاربری چیزی می‌نویسد و بلافاصله می‌خواند ولی به replicaِ عقب‌مانده می‌رسد → نتیجه‌ی قدیمی. راه‌حل: read-your-writes را به primary بفرست یا از replica با lag پایین بخوان.
- **Failover**: اگر primary بمیرد، یک replica ارتقا می‌یابد (promotion).

### Sharding در مقابل Partitioning

| | Partitioning | Sharding |
|---|---|---|
| چیست | شکستنِ یک جدولِ بزرگ به تکه‌ها (partition) | پخشِ داده روی **چند سرور/دیتابیس مجزا** |
| کجا | معمولاً **یک** سرور | **چند** ماشین |
| هدف | مدیریتِ جدولِ بزرگ، حذفِ سریعِ بازه، pruning | عبور از سقفِ یک ماشین (داده/throughput) |
| روش | Range / List / Hash partitioning | Hash / Range روی **shard key** |

```sql
-- Partitioning بومیِ PostgreSQL (declarative)
CREATE TABLE events (id BIGSERIAL, created_at DATE, payload JSONB)
    PARTITION BY RANGE (created_at);
CREATE TABLE events_2026 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
```

> نکته‌ی مصاحبه: شاردینگ تقریباً «partitioning افقی روی ماشین‌های مختلف» است. سختیِ اصلیِ شاردینگ: انتخابِ **shard key** خوب (برای جلوگیری از hot shard و توزیعِ یکنواخت)، و اینکه **JOIN و transaction بینِ شاردها** گران/پیچیده می‌شود (نیاز به cross-shard query یا distributed transaction). انتخابِ بدِ shard key بعداً اصلاحش بسیار دردناک است.

---

## ۶. Theory — CAP Theorem

در یک سیستمِ توزیع‌شده، هنگامِ **network partition** نمی‌توانی هم‌زمان هر سه را داشته باشی:

| حرف | یعنی |
|---|---|
| **Consistency** | هر خواندن، آخرین نوشته را می‌بیند (linearizability) |
| **Availability** | هر درخواست جواب می‌گیرد (هرچند شاید قدیمی) |
| **Partition Tolerance** | سیستم با وجودِ قطعِ شبکه کار می‌کند |

```
چون شبکه در واقعیت قطع می‌شود، P اجباری است.
پس انتخابِ واقعی همیشه بینِ C و A است، وقتی partition رخ می‌دهد:

   CP → هنگامِ partition، خواندن/نوشتنِ غلط را رد کن (مثل: HBase, etcd, dist. RDBMS)
   AP → هنگامِ partition، جواب بده حتی اگر قدیمی (مثل: Cassandra, DynamoDB, Riak)
```

> نکته‌ی مصاحبه: CAP فقط **هنگامِ partition** معنا دارد؛ وقتی شبکه سالم است، سیستم می‌تواند هم C هم A بدهد. مدلِ کامل‌ترِ امروزی **PACELC** است: اگر Partition (P) → بینِ A و C؛ Else (E) → بینِ Latency (L) و Consistency (C). اکثرِ دیتابیس‌های NoSQL، **eventual consistency** را برای availability انتخاب می‌کنند.

---

## 🔬 آزمایشگاه — خودت اجرا کن

> با `psql` (PostgreSQL) یا `sqlite3` قابل اجراست. بخشِ تراکنشِ هم‌زمان به **دو ترمینال** نیاز دارد.

```sql
-- ═══ ۱. ساختِ schema و دادهٔ نمونه (psql) ═══
CREATE TABLE students (id SERIAL PRIMARY KEY, name TEXT, city TEXT);
CREATE TABLE books    (id SERIAL PRIMARY KEY, title TEXT, available INT DEFAULT 1);
CREATE TABLE loans (
    student_id INT REFERENCES students(id),
    book_id    INT REFERENCES books(id),
    PRIMARY KEY (student_id, book_id)
);
INSERT INTO students(name, city) VALUES ('نرگس','تهران'),('سام','شیراز'),('علی','تهران');
INSERT INTO books(title) VALUES ('گربه نارنجی'),('ماهی سیاه'),('کلاغ');
INSERT INTO loans VALUES (1,1),(1,2),(2,1);

-- ═══ ۲. انواع JOIN ═══
SELECT s.name, b.title
FROM loans l
INNER JOIN students s ON s.id = l.student_id
INNER JOIN books    b ON b.id = l.book_id;

-- کاربرانی که هیچ کتابی امانت نگرفته‌اند (anti-join):
SELECT s.name FROM students s
LEFT JOIN loans l ON l.student_id = s.id
WHERE l.student_id IS NULL;

-- ═══ ۳. Aggregation: شمارش به‌ازای هر دانش‌آموز، فقط پُرکتاب‌ها ═══
SELECT s.name, COUNT(*) AS cnt
FROM loans l JOIN students s ON s.id = l.student_id
GROUP BY s.name
HAVING COUNT(*) >= 2;

-- ═══ ۴. ایندکس و EXPLAIN ═══
EXPLAIN ANALYZE SELECT * FROM students WHERE city = 'تهران';   -- احتمالاً Seq Scan
CREATE INDEX idx_students_city ON students(city);
ANALYZE students;
EXPLAIN ANALYZE SELECT * FROM students WHERE city = 'تهران';   -- حالا Index Scan
-- covering / index-only:
CREATE INDEX idx_city_name ON students(city) INCLUDE (name);
EXPLAIN ANALYZE SELECT name FROM students WHERE city = 'تهران';  -- Index Only Scan

-- ═══ ۵. تراکنش اتمیک ═══
BEGIN;
  UPDATE books SET available = available - 1 WHERE id = 3;
  INSERT INTO loans VALUES (3, 3);
COMMIT;     -- یا ROLLBACK برای بی‌اثر کردنِ هر دو

-- ═══ ۶. ایزولیشن با دو ترمینال (snapshot در REPEATABLE READ) ═══
-- ترمینال A:
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT available FROM books WHERE id = 1;     -- مثلاً 1
-- (هنوز COMMIT نکن)

-- ترمینال B (هم‌زمان):
UPDATE books SET available = 99 WHERE id = 1;  -- و COMMIT می‌شود

-- برگرد ترمینال A:
SELECT available FROM books WHERE id = 1;     -- هنوز 1 می‌بینی (snapshot ثابت = non-repeatable read جلوگیری)
COMMIT;
SELECT available FROM books WHERE id = 1;     -- حالا 99

-- ═══ ۷. قفلِ صریح برای جلوگیری از race ═══
BEGIN;
SELECT available FROM books WHERE id = 1 FOR UPDATE;   -- ردیف قفل شد تا COMMIT
UPDATE books SET available = available - 1 WHERE id = 1;
COMMIT;

-- ═══ ۸. partitioning ═══
CREATE TABLE events (id BIGSERIAL, day DATE) PARTITION BY RANGE (day);
CREATE TABLE events_2026 PARTITION OF events FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
```

```bash
# بازرسی از خطِ فرمان
psql -d mydb -c '\d+ students'       # ساختار جدول + ایندکس‌ها
psql -d mydb -c '\di'                # لیست ایندکس‌ها
psql -d mydb -c "SELECT * FROM pg_stat_activity;"   # تراکنش‌های فعال/قفل‌ها
```

---

## ✅ چک‌لیست تسلط

اگر بتوانی به این‌ها جواب بدهی، این فصل را بلدی:

- [ ] فرق چهار JOIN را با دیاگرام بکش؛ anti-join چطور نوشته می‌شود؟
- [ ] فرق `WHERE` و `HAVING`؟ ترتیبِ منطقیِ اجرای یک کوئری SELECT؟
- [ ] correlated subquery چیست و چرا ممکن است کُند باشد؟
- [ ] 1NF، 2NF، 3NF را با یک مثالِ نقض توضیح بده؛ کِی denormalize می‌کنی؟
- [ ] N:M را چطور پیاده می‌کنی؟ چرا junction table لازم است؟
- [ ] قانونِ leftmost-prefix در ایندکسِ composite؟ covering index چه می‌کند؟
- [ ] چرا ایندکسِ زیاد، write را کُند می‌کند؟ کِی ایندکس بی‌فایده است؟
- [ ] فرق `EXPLAIN` و `EXPLAIN ANALYZE`؟ `Seq Scan` کِی بد است؟
- [ ] چهار حرفِ ACID و مکانیزمِ هر کدام در Postgres؟
- [ ] سه anomaly را تعریف کن و به سطوحِ isolation نگاشت کن.
- [ ] رفتارِ خاصِ PostgreSQL در `REPEATABLE READ` و `READ UNCOMMITTED` چیست؟
- [ ] فرق sync و async replication؟ replication lag چه مشکلی می‌سازد؟
- [ ] فرق sharding و partitioning؟ انتخابِ shard key چرا مهم است؟
- [ ] CAP را توضیح بده؛ چرا انتخابِ واقعی بینِ C و A است؟

---

## 🎯 سوالات مصاحبه

### SQL — JOIN، Aggregation، Subquery

<details><summary>**فرق `INNER JOIN` و `LEFT JOIN` چیست؟**</summary>

`INNER JOIN` فقط ردیف‌هایی را برمی‌گرداند که در هر دو جدول تطبیق دارند. `LEFT JOIN` همه‌ی ردیف‌های جدولِ چپ را نگه می‌دارد و برای ردیف‌های بدونِ تطبیق در سمتِ راست، مقادیرِ `NULL` می‌گذارد.

</details>

<details><summary>**چطور کاربرانی را پیدا می‌کنی که هیچ سفارشی نداده‌اند؟**</summary>

الگوی anti-join:
```sql
SELECT u.* FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;
```
یا با `NOT EXISTS` (که با `NULL` امن‌تر از `NOT IN` است).

</details>

<details><summary>**اگر شرطِ فیلتر روی جدولِ راست را در `WHERE` بگذاری به‌جای `ON`، چه می‌شود؟**</summary>

در `LEFT JOIN`، گذاشتنِ شرطِ جدولِ راست در `WHERE` ردیف‌هایی را که آن ستونشان `NULL` است حذف می‌کند و عملاً JOIN را به `INNER JOIN` تبدیل می‌کند. اگر می‌خواهی outer بماند، شرط را در `ON` بگذار.

</details>

<details><summary>**فرق `WHERE` و `HAVING`؟**</summary>

`WHERE` قبل از `GROUP BY` روی تک‌تکِ ردیف‌ها فیلتر می‌کند و نمی‌تواند تابعِ aggregate داشته باشد. `HAVING` بعد از گروه‌بندی روی نتیجه‌ی دسته‌ها فیلتر می‌کند و می‌تواند روی `COUNT`/`SUM`/... شرط بگذارد.

</details>

<details><summary>**ترتیبِ منطقیِ اجرای یک کوئری SELECT چیست؟**</summary>

`FROM/JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT`. به همین دلیل alias تعریف‌شده در `SELECT` معمولاً در `WHERE` در دسترس نیست (هنوز اجرا نشده) ولی در `ORDER BY` هست.

</details>

<details><summary>**فرق subquery معمولی و correlated چیست؟**</summary>

subquery معمولی مستقل است و یک‌بار اجرا می‌شود. correlated subquery به ردیفِ بیرونی ارجاع دارد (مثل `WHERE o.user_id = u.id`) و مفهوماً به‌ازای هر ردیفِ بیرونی اجرا می‌شود — می‌تواند کُند باشد و اغلب با JOIN قابل بازنویسی است.

</details>

<details><summary>**`COUNT(*)` با `COUNT(col)` چه فرقی دارد؟**</summary>

`COUNT(*)` همه‌ی ردیف‌ها را می‌شمارد. `COUNT(col)` فقط ردیف‌هایی را که آن ستون `NULL` نیست می‌شمارد. `COUNT(DISTINCT col)` مقادیرِ یکتای غیرNULL را می‌شمارد.

</details>

### Data Modeling — Normalization و روابط

<details><summary>**3NF را با یک مثال توضیح بده.**</summary>

جدولی در 3NF است اگر در 2NF باشد و هیچ ستونِ غیرکلیدی به ستونِ غیرکلیدیِ دیگری وابستگیِ گذرا (transitive) نداشته باشد. مثالِ نقض: جدولِ `orders(id, zip, city)` که `city` به `zip` وابسته است نه به کلید — باید `city/zip` را به جدولِ جدا برد.

</details>

<details><summary>**جمله‌ی کوتاهی که 2NF و 3NF را خلاصه کند؟**</summary>

«هر ستونِ غیرکلیدی به the key (کلید)، the whole key (کلِ کلید — یعنی 2NF)، و nothing but the key (هیچ‌چیز جز کلید — یعنی 3NF) وابسته باشد.»

</details>

<details><summary>**رابطه‌ی N:M را چطور در دیتابیسِ رابطه‌ای پیاده می‌کنی؟**</summary>

با یک **junction table** (جدولِ واسط) که دو کلید خارجی دارد و معمولاً PK مرکب از همان دو. مثلاً `enrollments(student_id, course_id)`. دیتابیسِ رابطه‌ای نمی‌تواند N:M را مستقیم نمایش دهد.

</details>

<details><summary>**کِی normalization را قربانی می‌کنی (denormalization)؟**</summary>

وقتی کوئری‌های خواندن گران و پرتکرارند و JOINها گلوگاه شده‌اند. مثلاً ذخیره‌ی `comment_count` در جدولِ پست به‌جای `COUNT` هر بار. هزینه‌اش پیچیدگیِ نگه‌داشتنِ سازگاری (با trigger یا اپلیکیشن) و خطرِ داده‌ی متناقض است.

</details>

<details><summary>**Foreign key چه تضمینی می‌دهد و `ON DELETE CASCADE` یعنی چه؟**</summary>

FK تضمینِ یکپارچگیِ ارجاعی (referential integrity) می‌دهد: نمی‌توانی به ردیفی ارجاع دهی که وجود ندارد. `ON DELETE CASCADE` یعنی با حذفِ والد، فرزندانش هم خودکار حذف شوند (در مقابلِ `RESTRICT` که حذف را بلاک و `SET NULL` که FK را تهی می‌کند).

</details>

### Performance — Indexing و Optimization

<details><summary>**چرا افزودنِ ایندکسِ بیشتر همیشه خوب نیست؟**</summary>

هر ایندکس فضای دیسک می‌گیرد و باید در هر `INSERT/UPDATE/DELETE` به‌روز شود، پس writeها را کُند می‌کند. ایندکسِ بلااستفاده فقط هزینه است. ایندکس را بر اساسِ کوئری‌های واقعی بساز.

</details>

<details><summary>**قانونِ leftmost-prefix در ایندکسِ composite چیست؟**</summary>

ایندکسِ `(a, b, c)` فقط وقتی استفاده می‌شود که از سمتِ چپ شروع کنی: `a`, یا `a,b`, یا `a,b,c`. برای `WHERE b = ?` یا `WHERE c = ?` به‌تنهایی قابل استفاده نیست. بنابراین ترتیبِ ستون‌ها مهم است.

</details>

<details><summary>**covering index چیست و چرا سریع‌تر است؟**</summary>

ایندکسی که همه‌ی ستون‌های موردنیازِ کوئری را شامل می‌شود (در Postgres با `INCLUDE`). دیتابیس می‌تواند جواب را فقط از ایندکس بدهد بدونِ مراجعه به جدولِ اصلی (heap) — این **index-only scan** است و یک مرحله I/O را حذف می‌کند.

</details>

<details><summary>**فرق `EXPLAIN` و `EXPLAIN ANALYZE`؟**</summary>

`EXPLAIN` فقط پلنِ تخمینیِ planner را نشان می‌دهد بدونِ اجرا. `EXPLAIN ANALYZE` کوئری را واقعاً اجرا می‌کند و زمانِ واقعی و تعدادِ ردیفِ واقعی را هم می‌دهد — مراقب باش روی `UPDATE/DELETE` واقعاً اعمال می‌شود.

</details>

<details><summary>**سناریو: یک کوئری بعد از اینکه جدول به ۱۰ میلیون ردیف رسید کُند شد. چطور دیباگ می‌کنی؟**</summary>

۱) `EXPLAIN ANALYZE` بزن و دنبالِ `Seq Scan` روی جدولِ بزرگ بگرد. ۲) ببین فیلتر/جوین/مرتب‌سازی روی چه ستونی است و آیا ایندکس دارد. ۳) اگر `rows` تخمینی با `actual` خیلی فرق دارد، `ANALYZE` بزن (آمارِ کهنه). ۴) ایندکسِ مناسب (تک‌ستونی یا composite با ترتیبِ درست) بساز؛ اگر فقط چند ستون لازم است، covering. ۵) چک کن آیا تابعی روی ستون اعمال شده که ایندکس را بی‌اثر می‌کند. ۶) اگر داده ذاتاً بزرگ است، partitioning را بررسی کن.

</details>

<details><summary>**چرا `WHERE lower(email) = 'x'` ممکن است ایندکس را استفاده نکند؟**</summary>

اعمالِ تابع روی ستون، ایندکسِ معمولیِ آن ستون را بی‌اثر می‌کند چون ایندکس روی مقدارِ خام است نه `lower(...)`. راه‌حل: **functional/expression index**: `CREATE INDEX ON users(lower(email));`.

</details>

<details><summary>**کِی planner ترجیح می‌دهد Seq Scan کند حتی با وجودِ ایندکس؟**</summary>

وقتی کوئری بخشِ بزرگی از جدول را برمی‌گرداند (selectivity پایین) یا جدول کوچک است؛ خواندنِ ترتیبیِ کلِ جدول ارزان‌تر از پرشِ تصادفیِ ایندکس + مراجعه به heap می‌شود.

</details>

### Reliability — Transactions و ACID

<details><summary>**ACID مخففِ چیست؟**</summary>

Atomicity (همه یا هیچ)، Consistency (قیدها همیشه برقرار)، Isolation (تراکنش‌های هم‌زمان روی هم اثرِ ناخواسته نگذارند)، Durability (بعد از commit ماندگار حتی با قطعِ برق).

</details>

<details><summary>**سه anomaly همزمانی را نام ببر و تعریف کن.**</summary>

**Dirty read**: خواندنِ دادهٔ commit‌نشده. **Non-repeatable read**: خواندنِ دوباره‌ی همان ردیف نتیجه‌ی متفاوت می‌دهد (update/delete وسط). **Phantom read**: اجرای دوباره‌ی همان کوئریِ بازه‌ای، مجموعه‌ی متفاوتی از ردیف‌ها برمی‌گرداند (insert/delete وسط).

</details>

<details><summary>**هر سطحِ isolation کدام anomaly را جلوگیری می‌کند؟**</summary>

READ UNCOMMITTED: هیچ. READ COMMITTED: dirty read. REPEATABLE READ: + non-repeatable. SERIALIZABLE: همه‌ی سه. (در ANSI، RR هنوز phantom را اجازه می‌دهد.)

</details>

<details><summary>**رفتارِ PostgreSQL در سطوحِ isolation چه تفاوتی با استاندارد دارد؟**</summary>

Postgres فقط ۳ سطحِ واقعی دارد: `READ UNCOMMITTED` عملاً `READ COMMITTED` است (هرگز dirty read نمی‌دهد). `REPEATABLE READ` با snapshot isolation حتی phantom را هم جلوگیری می‌کند (برخلافِ ANSI) ولی write-skew را نه. `SERIALIZABLE` با SSI ممکن است تراکنش را با خطای 40001 لغو کند → باید retry کنی.

</details>

<details><summary>**deadlock چیست و چطور از آن جلوگیری می‌کنی؟**</summary>

وقتی دو (یا چند) تراکنش قفل‌هایی را برعکسِ هم نگه می‌دارند و هر کدام منتظرِ دیگری است. Postgres آن را تشخیص می‌دهد و یکی را لغو می‌کند. جلوگیری: قفل‌ها را همیشه به **ترتیبِ ثابت** بگیر، تراکنش‌ها را کوتاه نگه دار، و در صورتِ لزوم retry کن.

</details>

<details><summary>**فرق pessimistic و optimistic locking؟**</summary>

Pessimistic: ردیف را زودتر قفل می‌کنی (`SELECT ... FOR UPDATE`) تا دیگران نتوانند تغییر دهند — برای رقابتِ بالا. Optimistic: قفل نمی‌گذاری، فقط هنگامِ نوشتن یک ستونِ نسخه (version) را چک می‌کنی؛ اگر کسی تغییر داده، شکست می‌خوری و retry می‌کنی — برای رقابتِ کم بهتر است.

</details>

<details><summary>**سناریو: دو کاربر هم‌زمان آخرین بلیت را می‌خرند؛ هر دو موفق می‌شوند! چرا و چطور درست می‌کنی؟**</summary>

این یک race condition است: هر دو موجودی را خواندند (مثلاً ۱)، بعد هر دو کم کردند. راه‌حل‌ها: `SELECT ... FOR UPDATE` روی ردیفِ موجودی داخلِ تراکنش، یا `UPDATE tickets SET qty = qty - 1 WHERE id = ? AND qty > 0` و چکِ تعدادِ ردیفِ متأثر، یا قیدِ `CHECK (qty >= 0)`، یا سطحِ SERIALIZABLE با retry.

</details>

### Scalability — Replication، Sharding، Partitioning

<details><summary>**فرق synchronous و asynchronous replication؟**</summary>

Sync: commit تا تأییدِ replica کامل نمی‌شود — بدونِ از دست رفتنِ داده ولی تأخیرِ بیشترِ write. Async: primary بلافاصله commit می‌کند و بعداً به replica می‌رساند — سریع‌تر ولی با replication lag و خطرِ از دست رفتنِ آخرین تراکنش‌ها هنگامِ failover.

</details>

<details><summary>**سناریو: کاربر پروفایلش را ذخیره می‌کند ولی صفحه‌ی بعدی هنوز نامِ قدیمی را نشان می‌دهد. چرا؟**</summary>

stale read از read replicaِ عقب‌مانده (replication lag). نوشتن به primary رفت ولی خواندنِ بعدی به replicaِ هنوز به‌روزنشده. راه‌حل: read-your-writes (خواندنِ بلافاصله بعد از نوشتن را به primary بفرست) یا منتظرِ همگام‌سازیِ replica بمان.

</details>

<details><summary>**فرق sharding و partitioning؟**</summary>

Partitioning شکستنِ یک جدولِ بزرگ به تکه‌هاست، معمولاً روی یک سرور (range/list/hash). Sharding پخشِ داده روی چند سرور/دیتابیسِ مجزاست تا از سقفِ یک ماشین عبور کنی. شاردینگ تقریباً partitioning افقی روی ماشین‌های مختلف است.

</details>

<details><summary>**انتخابِ shard key چرا حیاتی است؟**</summary>

shard key توزیعِ داده و بار را تعیین می‌کند. کلیدِ بد باعثِ **hot shard** (یک شارد همه‌ی بار را می‌گیرد) یا توزیعِ نامتوازن می‌شود. همچنین کوئری‌هایی که shard key ندارند باید همه‌ی شاردها را بپرسند (scatter-gather). تغییرِ shard key بعد از استقرار بسیار پرهزینه (re-sharding) است.

</details>

<details><summary>**شاردینگ چه چیزهایی را سخت می‌کند؟**</summary>

JOIN بینِ شاردها، transactionهای توزیع‌شده (cross-shard ACID)، `UNIQUE` سراسری، و aggregationهایی که باید همه‌ی شاردها را بپرسند. به همین خاطر تا جای ممکن از sharding پرهیز می‌کنی و اول read replica و vertical scaling را امتحان می‌کنی.

</details>

<details><summary>**چرا read replica به‌تنهایی مشکلِ بارِ نوشتن را حل نمی‌کند؟**</summary>

همه‌ی نوشتن‌ها هنوز باید به primary بروند (و به replicaها هم تکثیر شوند). read replica فقط بارِ خواندن را پخش می‌کند. برای مقیاسِ نوشتن به sharding/partitioning نیاز داری.

</details>

### Theory — CAP

<details><summary>**قضیه‌ی CAP چه می‌گوید؟**</summary>

در یک سیستمِ توزیع‌شده، هنگامِ network partition نمی‌توان هم‌زمان Consistency و Availability را تضمین کرد؛ باید یکی را قربانی کنی. چون partition اجتناب‌ناپذیر است (P)، انتخابِ واقعی همیشه بینِ C و A است.

</details>

<details><summary>**یک سیستمِ CP و یک سیستمِ AP مثال بزن.**</summary>

CP (هنگامِ partition، consistency را نگه می‌دارد و در دسترس بودن را فدا می‌کند): etcd، ZooKeeper، HBase، اکثرِ RDBMSهای توزیع‌شده. AP (در دسترس می‌ماند با احتمالِ دادهٔ قدیمی): Cassandra، DynamoDB، Riak.

</details>

<details><summary>**اگر partition نباشد، آیا باید بینِ C و A انتخاب کنی؟**</summary>

نه. CAP فقط هنگامِ partition اعمال می‌شود. وقتی شبکه سالم است، سیستم می‌تواند هم consistent هم available باشد. به همین خاطر مدلِ **PACELC** کامل‌تر است: در حالتِ عادی (E)، بده‌بستان بینِ Latency و Consistency است.

</details>

<details><summary>**eventual consistency یعنی چه؟**</summary>

اگر نوشتنِ جدیدی نیاید، همه‌ی replicaها سرانجام به یک مقدار همگرا می‌شوند — اما در کوتاه‌مدت ممکن است خواندن‌ها مقادیرِ متفاوت/قدیمی بدهند. مدلِ رایج در سیستم‌های AP برای دستیابی به در دسترس بودن و تأخیرِ پایین.

</details>

<details><summary>**سناریو: چه زمانی consistency قوی را به availability ترجیح می‌دهی؟**</summary>

وقتی دادهٔ غلط فاجعه‌بار است: تراکنش‌های مالی/بانکی، موجودیِ انبار، رزروِ صندلی. در مقابل، برای شمارشِ لایک، فید اجتماعی یا کش، availability و تأخیرِ پایین مهم‌تر از دقتِ لحظه‌ای است و eventual consistency کافی است.

</details>
