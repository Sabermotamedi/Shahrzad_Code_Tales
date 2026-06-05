# سطح ۳ — توضیح فنی: Distributed Systems ⚙️

> دیگر قصه‌ای در کار نیست. این سند مرجع فنی است — همان چیزهایی که در مصاحبه و کار واقعی لازم داری. هر بخش با معادل قصه‌ای‌اش (لانه‌ی مورچه‌ها) شروع می‌شود تا ذهنت وصل بماند.

---

## ۱. مبانی سیستم‌های توزیع‌شده

> قصه: هزار مورچه به‌جای یک مورچه‌ی غول‌پیکر.

**سیستم توزیع‌شده** = مجموعه‌ای از کامپیوترهای مستقل (Node) که از طریق شبکه با هم کار می‌کنند و از دید کاربر مثل **یک سیستم واحد** رفتار می‌کنند.

### چرا توزیع می‌کنیم؟

| انگیزه | توضیح |
|---|---|
| **Scalability** | یک ماشین به سقفِ CPU/RAM می‌رسد؛ horizontal scaling (افزودن Node) تقریباً نامحدود است |
| **Fault Tolerance** | یک Node می‌افتد، سرویس زنده می‌ماند |
| **Latency** | Node نزدیک به کاربر = پاسخ سریع‌تر (geo-distribution) |
| **Throughput** | پردازشِ موازی روی چند ماشین |

هزینه‌اش: **پیچیدگی**. هماهنگی، سازگاری داده، و خرابیِ بخشی، مسائلِ تازه‌ای می‌آورند که در یک ماشینِ تنها وجود ندارند.

### هشت تصورِ غلط (The 8 Fallacies of Distributed Computing)

فرض‌هایی که تازه‌کارها می‌کنند و در عمل **همه‌شان غلط‌اند**:

1. The network is reliable — شبکه قابل‌اعتماد است
2. Latency is zero — تأخیر صفر است
3. Bandwidth is infinite — پهنای باند بی‌نهایت است
4. The network is secure — شبکه امن است
5. Topology doesn't change — توپولوژی ثابت است
6. There is one administrator — یک مدیر هست
7. Transport cost is zero — هزینه‌ی انتقال صفر است
8. The network is homogeneous — شبکه یکدست است

### خرابیِ بخشی (Partial Failure)

تفاوتِ بنیادینِ سیستم توزیع‌شده با برنامه‌ی تک‌ماشینی: در تک‌ماشین، همه‌چیز با هم کار می‌کند یا با هم می‌میرد. در توزیع‌شده، **بخشی** از سیستم می‌میرد و بقیه نمی‌دانند.

> نکته‌ی مصاحبه: «چرا نمی‌توانی فرق یک Node **مرده** و یک Node **کُند** را تشخیص بدهی؟» چون تنها شواهدت **نبودِ پاسخ تا یک timeout** است؛ و timeout هرگز نمی‌تواند قطعاً بگوید طرف مقابل مرده یا فقط دیر می‌کند. این عدم‌قطعیت، ریشه‌ی سختیِ اجماع و طراحیِ timeout/retry است.

### قضیه‌ی CAP (مرورِ کوتاه)

در حضورِ **Partition** (P) شبکه (که اجتناب‌ناپذیر است)، مجبوری بین **Consistency** (همه یک جواب) و **Availability** (همیشه جواب می‌دهد) یکی را انتخاب کنی:

- **CP**: در صورت partition، جواب نمی‌دهد تا سازگاری حفظ شود (مثل etcd/ZooKeeper).
- **AP**: همیشه جواب می‌دهد، حتی به قیمتِ داده‌ی کهنه (مثل Cassandra، DNS).

---

## ۲. الگوریتم‌های اجماع (Consensus)

> قصه: رأی‌گیری مورچه‌ها؛ اکثریت برنده می‌شود حتی اگر چند مورچه خواب باشند.

**مسئله:** گروهی از Nodeها، با وجودِ پیام‌های گم‌شونده و Nodeهای از کار افتاده، باید روی **یک مقدار/یک ترتیب از تصمیم‌ها** توافق کنند، طوری که همه به یک نتیجه برسند (Agreement) و تصمیمِ Commit‌شده هرگز عوض نشود (Safety).

### Quorum / اکثریت

- با `N` عضو، quorum برابر `floor(N/2) + 1` است.
- چون هر دو quorum **حداقل یک عضوِ مشترک** دارند، دو تصمیمِ متناقض نمی‌تواند هم‌زمان Commit شود.
- تعداد را **فرد** می‌گیرند (۳/۵/۷). یک خوشه‌ی ۵تایی، خرابیِ **۲** Node را تحمل می‌کند (`N=5` → quorum=3).

| N | Quorum | خرابیِ قابل‌تحمل |
|---|---|---|
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

### Raft

الگوریتمِ اجماعِ محبوب، طراحی‌شده برای **فهم‌پذیری**. هر Node یکی از سه نقش را دارد: **Leader / Follower / Candidate**. زمان به **Term**ها (دوره‌ها) تقسیم می‌شود.

**۱) Leader Election:** Followerها هرکدام یک تایمرِ تصادفی (election timeout) دارند. اگر در این مدت heartbeatی از Leader نرسید، Follower تبدیل به Candidate می‌شود، Term را +۱ می‌کند، به خودش رأی می‌دهد و از بقیه رأی می‌خواهد. هرکس **اکثریت** رأی گرفت، Leader می‌شود. تصادفی‌بودنِ تایمر از رأیِ منقسم (split vote) جلوگیری می‌کند.

**۲) Log Replication:** همه‌ی نوشتن‌ها از Leader می‌گذرند. Leader ورودی را به Logِ خودش اضافه می‌کند، با `AppendEntries` به Followerها می‌فرستد، و وقتی **اکثریت** آن را نوشتند، ورودی **Committed** می‌شود و به ماشینِ حالت (state machine) اعمال می‌گردد. Logها روی همه‌ی Nodeها **یکسان و هم‌ترتیب** نگه داشته می‌شوند.

```
Leader Log:   [x=1][y=2][x=3]  ← committed تا index 2 (اکثریت نوشتند)
Follower A:   [x=1][y=2][x=3]
Follower B:   [x=1][y=2]       ← هنوز عقب است؛ Leader دوباره می‌فرستد
```

> **Paxos** قدیمی‌ترین الگوریتمِ اجماعِ اثبات‌شده است (Leslie Lamport). درست اما بدنام به سختیِ فهم؛ Raft عمداً به‌عنوان جایگزینِ فهم‌پذیر ساخته شد. Multi-Paxos تقریباً معادلِ Raft است. کافی است بدانی Paxos وجود دارد و چرا Raft جایگزینش شد.

> نکته‌ی مصاحبه: «چرا تعداد اعضای خوشه را فرد می‌گذاری؟» چون ۴ عضو همان تحملِ خطای ۳ عضو را دارد (هر دو quorum=۳... برای ۴، quorum=۳ یعنی تحملِ ۱) ولی هزینه‌ی بیشتر؛ عددِ فرد بهترین نسبتِ تحمل‌خطا به تعدادِ Node است و از تساویِ رأی جلوگیری می‌کند.

---

## ۳. صف پیام (Message Queue)

> قصه: تخته‌پیامِ بوها — می‌گذاری و می‌روی؛ دیگری سرِ فرصت برمی‌دارد.

**اجزا:** Producer (تولید) → Broker/Queue (واسط) → Consumer (مصرف).

**هدف اصلی: Decoupling.** Producer و Consumer از هم مستقل‌اند؛ نه از وجودِ هم خبر دارند، نه هم‌زمان باید بالا باشند. مزایا:

- **Buffering**: اگر Consumer کند شد، پیام‌ها در صف صبر می‌کنند (تحملِ بار ناگهانی / spike).
- **Resilience**: خرابیِ Consumer، Producer را قفل نمی‌کند.
- **Load leveling** و توزیع کار بین چند Consumer.

### تضمین‌های تحویل

| تضمین | مکانیزم | خطر |
|---|---|---|
| **At-most-once** | بدونِ ack/بدونِ retry — fire and forget | از دست رفتنِ پیام |
| **At-least-once** | ack + redelivery اگر ack نرسید | پردازشِ تکراری |
| **Exactly-once** | at-least-once + idempotency/dedup در سطحِ پردازش | پیچیده و گران |

**Exactly-once در تحویلِ شبکه‌ای نشدنی است** (به خاطرِ گم‌شدنِ ack نمی‌توانی بفهمی پیام رسید و فقط ackش گم شد، یا اصلاً نرسید). راهِ عملی: **at-least-once + idempotent consumer**.

### Idempotency

پردازنده را طوری بنویس که اجرای چندباره با یک‌باره **هم‌نتیجه** باشد. روش‌ها:

- **Dedup key**: هر پیام یک `message_id` یکتا دارد؛ پردازش‌شده‌ها را در یک store (مثل Redis SET) نگه دار و تکراری‌ها را رد کن.
- **Upsert** به‌جای insert؛ **set** به‌جای increment.
- عملیاتِ ذاتاً idempotent (مثلِ `PUT` در REST).

**RabbitMQ**: یک message broker با مدلِ exchange → queue → binding. ویژگی‌ها: ack دستی، prefetch، dead-letter queue (DLQ) برای پیام‌های شکست‌خورده، routing بر اساس key. مدلش **push** (broker به consumer hold می‌دهد) و پیام پس از ack **حذف** می‌شود.

---

## ۴. Kafka

> قصه: ردِّ بوییِ همیشگی که هرگز پاک نمی‌شود و هرکس از اول می‌تواند راهش برود.

Kafka یک **distributed append-only log** است، نه یک صفِ معمولی. تفاوتِ بنیادی: پیام پس از خواندن **حذف نمی‌شود**؛ تا پایانِ retention سرِ جایش می‌ماند و **چندین** Consumer مستقل می‌توانند بخوانندش.

| اصطلاح | تعریف |
|---|---|
| **Topic** | یک جریانِ نام‌دار از رویدادها |
| **Partition** | تقسیمِ یک Topic به چند log موازی؛ واحدِ موازی‌سازی و ترتیب |
| **Offset** | شماره‌ی ترتیبیِ یکتای هر پیام **درونِ یک partition** |
| **Consumer Group** | مجموعه‌ای از Consumer که partitionها بینشان تقسیم می‌شود |
| **Retention** | مدت/حجمِ نگه‌داریِ پیام‌ها (time-based یا size-based؛ یا log compaction) |
| **Broker** | یک Node از خوشه‌ی Kafka |
| **Replication factor** | تعداد کپیِ هر partition روی brokerهای مختلف (برای دوام) |

**نکات کلیدی:**

- ترتیب **فقط درونِ یک partition** تضمین می‌شود، نه در کلِ Topic. اگر ترتیبِ کلیدِ خاصی مهم است، با همان key پارتیشن‌بندی کن (`hash(key) % partitions`) تا همیشه به یک partition برود.
- در یک Consumer Group، **هر partition فقط به یک Consumer** داده می‌شود → موازی‌سازی محدود به تعدادِ partitionهاست. Consumerهای بیشتر از partition، بیکار می‌مانند.
- چند Consumer Groupِ مختلف، **مستقل** و هرکدام با offsetِ خودش، همان Topic را می‌خوانند (pub/sub در سطحِ گروه).
- Consumer می‌تواند offset را **عقب ببرد** و تاریخ را **replay** کند — قدرتِ اصلیِ مدلِ log.

```
Topic "orders" (replication=3):
 Partition 0: [0][1][2][3][4][5]   committed offset گروهِ A = 3
 Partition 1: [0][1][2][3]
 Partition 2: [0][1][2][3][4]
   هر partition روی ۳ broker کپی می‌شود؛ یکی leader، بقیه follower (ISR)
```

> نکته‌ی مصاحبه: «Kafka در برابر RabbitMQ؟» RabbitMQ صفِ smart-broker است (routing پیچیده، پیام پس از مصرف حذف، تأخیرِ کم، کارهای task-queue). Kafka log با throughput بالا، نگه‌داری و replay، برای event streaming و خطوطِ داده. اگر «replay» یا «چند مصرف‌کننده‌ی مستقلِ همان داده» می‌خواهی → Kafka.

---

## ۵. Event Streaming و معماری رویدادمحور

> قصه: «برو غذا بیار» (فرمان) در برابر «غذا پیدا شد» (رویداد).

| | Command | Event |
|---|---|---|
| معنا | درخواستِ انجامِ کاری | اعلامِ چیزی که **اتفاق افتاده** (گذشته) |
| گیرنده | یک گیرنده‌ی مشخص | هرکس مشترک شده |
| نام‌گذاری | امری: `CreateOrder` | گذشته: `OrderCreated` |
| اتصال | coupling بیشتر | coupling کمتر |

**Event-Driven Architecture:** اجزا به‌جای فراخوانیِ مستقیمِ هم، رویداد **منتشر (publish)** می‌کنند و دیگران **مشترک (subscribe)** می‌شوند. مزیت: افزودنِ مصرف‌کننده‌ی جدید بدونِ تغییرِ تولیدکننده.

الگوهای رایج:

- **Event Notification**: رویداد فقط می‌گوید «چیزی شد»؛ گیرنده برای جزئیات دوباره می‌پرسد.
- **Event-Carried State Transfer**: رویداد خودش داده‌ی لازم را حمل می‌کند (گیرنده نیازی به پرسیدنِ دوباره ندارد).
- **Event Sourcing**: حالتِ سیستم = حاصلِ replay همه‌ی رویدادها از اول (همان ردِّ همیشگیِ Kafka). به‌جای ذخیره‌ی «حالتِ فعلی»، **تاریخِ کاملِ تغییرات** را ذخیره می‌کنی.
- **CQRS**: جداکردنِ مسیرِ نوشتن (command) از مسیرِ خواندن (query)؛ اغلب با event sourcing جفت می‌شود.

---

## ۶. کشینگ (Caching)

> قصه: خوراکی‌های پرطرف‌دار را دمِ درِ لانه نگه می‌داری.

لایه‌ای سریع (معمولاً in-memory) جلوی منبعِ کندِ داده (DB/سرویس) برای کاهشِ latency و بارِ منبع.

### الگوهای کش

| الگو | مسیر خواندن | مسیر نوشتن | نکته |
|---|---|---|---|
| **Cache-Aside (Lazy)** | cache → miss → DB → پر کردن cache | مستقیم به DB، cache را invalidate کن | رایج‌ترین؛ فقط داده‌ی واقعاً خواسته‌شده کش می‌شود |
| **Write-Through** | از cache | هم‌زمان cache و DB | cache همیشه تازه؛ نوشتن کندتر |
| **Write-Back (Write-Behind)** | از cache | اول cache، بعداً async به DB | سریع‌ترین نوشتن؛ خطرِ از دست رفتنِ داده اگر cache بمیرد |
| **Read-Through** | از cache؛ خودِ cache در miss از DB می‌خواند | — | منطقِ بارگیری داخلِ لایه‌ی cache |

### دو مشکلِ سخت + خطرها

> «There are only two hard things in Computer Science: cache invalidation and naming things.» — Phil Karlton

- **TTL (Time To Live)**: انقضای خودکارِ هر کلید؛ ساده‌ترین راهِ تازه‌نگه‌داشتن، اما پنجره‌ی staleness دارد.
- **Invalidation**: حذف/به‌روزرسانیِ کلید هنگام تغییرِ داده‌ی اصلی. استراتژی‌ها: write-through (هم‌زمان آپدیت)، **delete-on-write** (نوشتن به DB + حذفِ کلید، تا خواندنِ بعدی دوباره از DB پر کند — معمولاً امن‌تر از آپدیتِ هم‌زمانِ cache به‌خاطرِ race)، یا event-based invalidation.
- **Cache Stampede / Thundering Herd**: وقتی یک کلیدِ داغ منقضی می‌شود، هزاران درخواست هم‌زمان miss می‌خورند و به DB هجوم می‌برند. راه‌حل‌ها:
  - **Locking / single-flight**: فقط یک درخواست بازسازی کند، بقیه منتظر بمانند یا مقدارِ کهنه بگیرند.
  - **TTL jitter**: انقضاها را پراکنده کن تا با هم منقضی نشوند.
  - **Early/probabilistic recomputation**: کمی قبل از انقضا، پیشاپیش بازسازی کن.

> نکته‌ی مصاحبه: «cache و DB ناسازگار شدند، کدام invalidation؟» معمولاً الگوی **cache-aside + delete-on-write** (نه update-on-write): پس از نوشتن در DB، کلید را **حذف** کن نه آپدیت؛ این پنجره‌ی race بینِ نوشتن‌های هم‌زمان را کوچک می‌کند. برای سازگاریِ قوی‌تر می‌توان از versioning یا CDC (Change Data Capture) برای invalidation استفاده کرد.

---

## ۷. Redis

> قصه: انبارکِ سریعِ دمِ در.

پایگاه‌داده‌ی **in-memory و single-threaded** (برای دستورهای داده) با latency در حدِ زیرمیلی‌ثانیه. دوام اختیاری با **RDB** (snapshot) و **AOF** (append-only log).

### ساختارهای داده

| ساختار | کاربرد نمونه |
|---|---|
| **String** | کش ساده، شمارنده (با `INCR`) |
| **Hash** | شیء با چند فیلد (پروفایل کاربر) |
| **List** | صف/پشته (`LPUSH`/`RPOP`) |
| **Set** | عضویت یکتا، dedup |
| **Sorted Set (ZSet)** | leaderboard، صفِ اولویت‌دار، rate limiting با sliding window |
| **Stream** | log مثلِ Kafkaِ سبک (consumer groups دارد) |
| **Bitmap / HyperLogLog** | شمارشِ تقریبی، حضور/غیاب |

### کاربردها

| کاربرد | چطور |
|---|---|
| **Cache** | `SET key val EX 300` (با TTL) |
| **Session store** | session id → داده‌ی کاربر، با TTL |
| **Distributed Lock** | `SET lock token NX PX 30000` (فقط اگر نبود + انقضا)؛ آزادسازیِ امن با Lua script که token را چک کند. برای استحکام بیشتر: **Redlock** |
| **Queue** | List با `LPUSH`/`BRPOP` (مسدودکننده) |
| **Pub/Sub** | `PUBLISH channel msg` / `SUBSCRIBE channel` — پخشِ آنی (پیام نگه داشته نمی‌شود؛ مشترکِ آفلاین از دست می‌دهد) |
| **Rate limiting** | `INCR` + `EXPIRE`، یا sorted set |

> نکته‌ی مصاحبه: «قفلِ Redis چه ریسکی دارد؟» اگر صاحبِ قفل قبل از آزادسازی کند شود و TTL منقضی شود، دیگری قفل را می‌گیرد و **دو نفر هم‌زمان** واردِ بخشِ بحرانی می‌شوند. به همین خاطر آزادسازی باید **token را چک کند** (فقط صاحبِ واقعی آزاد کند) و عملیاتِ بحرانی بهتر است idempotent باشد. Redlock هم بحث‌برانگیز است.

---

## ۸. تحمل خطا (Fault Tolerance)

> قصه: یک مورچه می‌افتد، لانه نمی‌خوابد.

| تکنیک | توضیح |
|---|---|
| **Redundancy** | چند کپی/چند instance؛ هیچ single point of failure نباشد |
| **Timeout** | هرگز نامحدود منتظر نمان؛ مهلت بگذار تا منبعِ thread/connection آزاد شود |
| **Retry + Backoff + Jitter** | تلاشِ مجدد، با تأخیرِ نماییِ فزاینده + تصادفی‌سازی |
| **Circuit Breaker** | قطعِ موقتِ تماس با سرویسِ خراب |
| **Bulkhead** | جداسازیِ منابع (مثلِ مخزن‌های جدا) تا خرابیِ یک بخش بقیه را غرق نکند |
| **Graceful Degradation** | در خرابی، پاسخِ ناقص/کهنه بده به‌جای خطای کامل (مثلاً از cache) |

### Retry با Backoff و Jitter

retryِ ساده و بلافاصله **خطرناک** است: اگر سرور به‌خاطرِ بار افتاده، هجومِ هم‌زمانِ retryها آن را دوباره زمین می‌زند (**retry storm**). راه درست:

- **Exponential backoff**: تأخیر = `base * 2^attempt` (۱s, ۲s, ۴s, ۸s...).
- **Jitter**: یک مؤلفه‌ی تصادفی اضافه کن تا کلاینت‌ها **هم‌زمان** retry نکنند (full jitter: `random(0, base*2^attempt)`).
- **سقفِ تلاش** و **سقفِ تأخیر** بگذار؛ فقط روی خطاهای **گذرا** retry کن (نه `400 Bad Request`).
- retry را با **idempotency** ترکیب کن، وگرنه عملیاتِ تکراری داده را خراب می‌کند.

### Circuit Breaker (سه حالت)

```
            failures ≥ threshold
   CLOSED ───────────────────────→ OPEN
   (عبور)                          (رد فوری، بدون تماس)
     ↑                                │ پس از cooldown
     │ success در HALF-OPEN           ▼
     └──────────────────────────  HALF-OPEN
            failure → OPEN          (چند درخواست آزمایشی)
```

- **CLOSED**: همه‌چیز عادی، تماس‌ها عبور می‌کنند، خطاها شمرده می‌شوند.
- **OPEN**: نرخِ خطا از حد گذشت → تماس‌ها **فوراً رد** می‌شوند (fail fast)، بدونِ زدن به سرویسِ مریض.
- **HALF-OPEN**: بعد از یک مهلت، چند درخواستِ آزمایشی عبور می‌کند؛ موفق → CLOSED، ناموفق → OPEN.

---

## ۹. دسترس‌پذیری بالا (High Availability)

> قصه: لانه‌ای که هرگز نمی‌خوابد.

### جدول نُه‌ها

| Availability | اسم | Downtime/سال | Downtime/ماه |
|---|---|---|---|
| 99% | two nines | ~3.65 روز | ~7.2 ساعت |
| 99.9% | three nines | ~8.77 ساعت | ~43.8 دقیقه |
| 99.99% | four nines | ~52.6 دقیقه | ~4.4 دقیقه |
| 99.999% | five nines | ~5.26 دقیقه | ~26 ثانیه |

`Availability = MTBF / (MTBF + MTTR)` — هم با کم‌کردنِ خرابی (MTBF بالا) و هم با سریع‌ترکردنِ ترمیم (MTTR پایین) بهبود می‌یابد.

### مکانیزم‌ها

- **Health Check**: بررسیِ دوره‌ای (مثلِ `GET /healthz`). تفاوتِ **liveness** (زنده‌ای؟ اگر نه restart) و **readiness** (آماده‌ی ترافیکی؟ اگر نه از LB خارج کن).
- **Failover**: انتقالِ خودکارِ بار به نسخه‌ی پشتیبان هنگامِ خرابیِ اصلی.

| | Active-Passive | Active-Active |
|---|---|---|
| کار | یکی فعال، یکی standby | همه فعال، بار تقسیم |
| استفاده از منابع | پشتیبان بیکار | همه مشغول |
| Failover | بیدارکردنِ standby (کمی کندتر) | بقیه بار را برمی‌دارند (تقریباً آنی) |
| پیچیدگی | کمتر | بیشتر (هماهنگیِ داده، احتمالِ split-brain) |

> نکته‌ی مصاحبه: «split-brain چیست؟» در active-active یا failover، اگر بر اثرِ partition شبکه **دو طرف هم‌زمان** خود را اصلی بپندارند و هر دو بنویسند، داده دوشاخه و خراب می‌شود. راه‌حل: fencing، quorum برای انتخابِ اصلی، یا STONITH.

---

## 🔬 آزمایشگاه — خودت اجرا کن

### ۱) راه‌اندازی Redis با Docker

```bash
docker run -d --name lab-redis -p 6379:6379 redis:7
docker exec -it lab-redis redis-cli       # یا: redis-cli -h 127.0.0.1 -p 6379
```

### ۲) پایه‌ها: SET / GET / EXPIRE / TTL / INCR

```bash
SET user:1 "sam"             # ذخیره
GET user:1                   # → "sam"
SET session:1 "data" EX 10   # با TTL ده‌ثانیه‌ای
TTL session:1                # → ثانیه‌های باقی‌مانده
EXPIRE user:1 30             # انقضا برای کلیدِ موجود
INCR page:views              # → 1   (شمارنده‌ی اتمیک)
INCR page:views              # → 2
INCRBY page:views 10         # → 12
```

### ۳) List به‌عنوان صف (Producer/Consumer)

```bash
# Producer: پیام را به انتهای صف اضافه کن
LPUSH queue:jobs "job-1"
LPUSH queue:jobs "job-2"
LLEN  queue:jobs             # → 2  (طول صف)

# Consumer: از سرِ دیگر بردار (FIFO)
RPOP  queue:jobs             # → "job-1"
# نسخه‌ی مسدودکننده: تا رسیدنِ کار صبر کن (worker واقعی)
BRPOP queue:jobs 5           # حداکثر ۵ ثانیه منتظر بماند
```

### ۴) Pub/Sub در دو ترمینال

```bash
# ترمینال ۱ — مشترک:
docker exec -it lab-redis redis-cli
SUBSCRIBE news               # منتظرِ پیام می‌ماند

# ترمینال ۲ — ناشر:
docker exec -it lab-redis redis-cli
PUBLISH news "hello ants"    # → ترمینال ۱ پیام را آنی می‌گیرد
```

> توجه: pub/sub پیام را **نگه نمی‌دارد**؛ مشترکی که آن لحظه آنلاین نباشد پیام را از دست می‌دهد. برای دوام و replay از Redis **Stream** یا Kafka استفاده کن.

### ۵) قفلِ توزیع‌شده (طرحِ ساده)

```bash
SET lock:order:7 "token-abc" NX PX 30000   # فقط اگر نبود، با انقضای ۳۰ ثانیه
# ... کارِ بحرانی ...
# آزادسازیِ امن باید token را چک کند (در عمل با Lua script اتمیک)
```

### ۶) Retry با Exponential Backoff + Jitter (Python)

```python
import time, random

def call_with_retry(fn, max_attempts=5, base=0.5, cap=30.0):
    for attempt in range(max_attempts):
        try:
            return fn()
        except TransientError as e:
            if attempt == max_attempts - 1:
                raise
            # exponential backoff با سقف
            backoff = min(cap, base * (2 ** attempt))
            # full jitter: بازه‌ی تصادفی [0, backoff]
            sleep = random.uniform(0, backoff)
            time.sleep(sleep)
    # روی خطاهای دائمی (مثل 400) اصلاً retry نکن
```

### ۷) Consumer ساده‌ی idempotent (شبه‌کد)

```python
def handle(msg):
    # at-least-once → ممکن است msg تکراری باشد
    if redis.sadd("processed", msg.id) == 0:
        return                      # قبلاً دیده‌ایم → رد کن (dedup)
    redis.expire("processed", 86400)
    process(msg)                    # کارِ اصلی؛ بهتر است خودش هم idempotent باشد
```

### پاک‌سازی

```bash
docker rm -f lab-redis
```

---

## ✅ چک‌لیست تسلط

اگر بتوانی به این‌ها جواب بدهی، این فصل را بلدی:

- [ ] چرا توزیع می‌کنیم و هزینه‌اش چیست؟ هر ۸ fallacy را نام ببر.
- [ ] partial failure یعنی چه و چرا نمی‌توانی Node مرده را از کُند تشخیص بدهی؟
- [ ] quorum چطور از تصمیمِ متناقض جلوگیری می‌کند؟ چرا تعداد را فرد می‌گذاری؟
- [ ] دو کارِ اصلیِ Raft (leader election + log replication) را شرح بده؛ نسبتش با Paxos؟
- [ ] فرق at-most-once، at-least-once و exactly-once؟ چرا exactly-once در تحویل نشدنی است؟
- [ ] idempotency چیست و چطور پیاده می‌شود؟
- [ ] فرق RabbitMQ (صف، حذف پس از مصرف) و Kafka (log، retention، replay)؟
- [ ] Topic/Partition/Offset/Consumer Group/Retention را تعریف کن. ترتیب کجا تضمین می‌شود؟
- [ ] فرق event و command؟ event sourcing و CQRS را توضیح بده.
- [ ] cache-aside در برابر write-through؟ delete-on-write چرا امن‌تر از update-on-write است؟
- [ ] cache stampede چیست و سه راه‌حلش؟ «دو مشکلِ سخت» کدام‌اند؟
- [ ] پنج کاربردِ Redis و ساختارِ مناسب هرکدام. ریسکِ قفلِ Redis؟
- [ ] retry چرا باید backoff + jitter داشته باشد؟ retry storm چیست؟
- [ ] سه حالتِ circuit breaker و گذارها؟
- [ ] جدولِ نُه‌ها: 99.99% یعنی چند دقیقه downtime در سال؟
- [ ] active-active در برابر active-passive؛ split-brain چیست و چطور جلوگیری می‌شود؟

---

## 🎯 سوالات مصاحبه

### مبانی

<details><summary>**سیستم توزیع‌شده دقیقاً چیست و چرا به آن می‌رویم؟**</summary>

مجموعه‌ای از Nodeهای مستقل که از طریق شبکه با هم کار می‌کنند و از دید کاربر مثل یک سیستمِ واحد رفتارند. انگیزه‌ها: scalability (افقی، فراتر از سقفِ یک ماشین)، fault tolerance، کاهشِ latency با geo-distribution، و throughputِ موازی. هزینه‌اش پیچیدگیِ هماهنگی و سازگاری داده است.

</details>

<details><summary>**هشت تصور غلط (fallacies) سیستم‌های توزیع‌شده چیست؟**</summary>

شبکه قابل‌اعتماد است، تأخیر صفر است، پهنای باند بی‌نهایت است، شبکه امن است، توپولوژی ثابت است، یک مدیر هست، هزینه‌ی انتقال صفر است، شبکه یکدست است. همه غلط‌اند؛ نادیده‌گرفتنشان منشأ بیشترِ باگ‌های توزیع‌شده است.

</details>

<details><summary>**partial failure یعنی چه و چرا سخت است؟**</summary>

برخلافِ تک‌ماشین که یا کامل کار می‌کند یا کامل می‌میرد، در توزیع‌شده بخشی می‌میرد و بقیه نمی‌دانند. تنها شاهدت نبودِ پاسخ تا یک timeout است، و timeout نمی‌تواند فرقِ Node **مرده** و **کُند** را قطعی کند. همین عدم‌قطعیت، طراحیِ timeout/retry/consensus را پیچیده می‌کند.

</details>

<details><summary>**قضیه‌ی CAP را با یک مثال توضیح بده.**</summary>

در حضورِ network partition، بین Consistency (همه یک جواب) و Availability (همیشه جواب) باید یکی را برگزینی. CP (مثل etcd): در partition جواب نمی‌دهد تا سازگار بماند. AP (مثل Cassandra/DNS): همیشه جواب می‌دهد حتی با داده‌ی کهنه. در غیابِ partition هر دو ممکن است.

</details>

### اجماع (Consensus)

<details><summary>**مسئله‌ی consensus چیست و quorum چطور حلش می‌کند؟**</summary>

رساندنِ گروهی از Nodeها به توافق روی یک مقدار/ترتیب، با وجودِ خطا. quorum (اکثریت، `floor(N/2)+1`) کلید است: چون هر دو quorum حداقل یک عضوِ مشترک دارند، دو تصمیمِ متناقض نمی‌تواند هم‌زمان commit شود.

</details>

<details><summary>**چرا تعداد اعضای خوشه را فرد می‌گذارند؟**</summary>

تحملِ خطا را عددِ کوچک‌ترِ نصف تعیین می‌کند. ۵ عضو تحملِ ۲ خرابی، ۴ عضو هم فقط تحملِ ۱ خرابی دارد ولی هزینه‌ی بیشتر — پس عددِ فرد بهترین نسبتِ تحمل‌خطا به Node است. ضمناً از تساویِ رأی (split vote) جلوگیری می‌کند.

</details>

<details><summary>**دو کار اصلی Raft را شرح بده.**</summary>

(۱) Leader Election: با election timeoutهای تصادفی یک Candidate رأیِ اکثریت می‌گیرد و Leader می‌شود؛ تصادفی‌بودن از split vote جلوگیری می‌کند. (۲) Log Replication: همه‌ی نوشتن‌ها از Leader می‌گذرند؛ ورودی وقتی **اکثریتِ** Followerها نوشتند Commit می‌شود و Logها یکسان و هم‌ترتیب می‌مانند.

</details>

<details><summary>**فرق Raft و Paxos؟**</summary>

هر دو الگوریتمِ اجماع‌اند. Paxos قدیمی‌تر و اثبات‌شده اما بدنام به سختیِ فهم و پیاده‌سازی است. Raft عمداً برای فهم‌پذیری ساخته شد (تفکیکِ صریحِ نقش‌ها، leader قوی). Multi-Paxos عملاً معادلِ Raft است. در عمل Raft رایج‌تر است (etcd، Consul).

</details>

<details><summary>**یک خوشه‌ی ۵تایی چند خرابی را تحمل می‌کند و اگر ۳ Node بیفتد چه؟**</summary>

quorum=۳، پس تحملِ ۲ خرابی دارد. اگر ۳ تا بیفتد، فقط ۲ Node می‌ماند که quorum نمی‌سازد → خوشه نمی‌تواند تصمیمِ جدید commit کند (برای حفظِ safety می‌ایستد؛ این یک سیستمِ CP است).

</details>

### صف پیام و Kafka

<details><summary>**چرا message queue؟ چه چیزی را decouple می‌کند؟**</summary>

Producer و Consumer را از هم مستقل می‌کند: نه باید هم‌زمان بالا باشند، نه از هم خبر داشته باشند. مزایا: buffering در برابر spike، resilience (خرابیِ مصرف‌کننده تولیدکننده را قفل نمی‌کند)، load leveling و توزیعِ کار.

</details>

<details><summary>**مصرف‌کننده‌ها بعضی پیام‌ها را دوبار پردازش می‌کنند — چرا، و آیا exactly-once ممکن است؟**</summary>

چون بیشترِ سیستم‌ها at-least-once‌اند: اگر ack گم شود، broker پیام را دوباره می‌فرستد و نمی‌توان فهمید پیام رسید و فقط ack گم شد یا اصلاً نرسید. exactly-once در **تحویلِ شبکه‌ای** نشدنی است؛ راهِ عملی at-least-once + **idempotent consumer** (با dedup key یا عملیاتِ upsert/set) است که اثرِ تکرار را خنثی می‌کند.

</details>

<details><summary>**at-most-once در برابر at-least-once در برابر exactly-once؟**</summary>

at-most-once: بدونِ retry، ممکن است پیام گم شود. at-least-once: با ack+redelivery، ممکن است تکراری شود. exactly-once: آرزو؛ در عمل با at-least-once + idempotency/dedup شبیه‌سازی می‌شود. انتخاب به تحملِ گم‌شدن در برابر تحملِ تکرار بستگی دارد.

</details>

<details><summary>**idempotency چیست و چطور پیاده‌اش می‌کنی؟**</summary>

عملیاتی که اجرای چندباره‌اش با یک‌باره هم‌نتیجه است. روش‌ها: dedup با message_id یکتا (نگه‌داریِ processed idها در یک store)، upsert به‌جای insert، set به‌جای increment، یا عملیاتِ ذاتاً idempotent مثلِ PUT.

</details>

<details><summary>**Kafka در برابر RabbitMQ — کِی کدام؟**</summary>

RabbitMQ: صفِ smart-broker، routing پیچیده، پیام پس از مصرف **حذف**، مناسبِ task queueها و latency پایین. Kafka: append-only log با throughput بالا، **retention و replay**، چند Consumer Groupِ مستقل روی همان داده. اگر replay یا چند مصرف‌کننده‌ی مستقل می‌خواهی → Kafka؛ اگر routing پیچیده و work-queue → RabbitMQ.

</details>

<details><summary>**Topic، Partition، Offset، Consumer Group و Retention را تعریف کن.**</summary>

Topic: جریانِ نام‌دارِ رویداد. Partition: تقسیمِ Topic به چند log موازی (واحدِ موازی‌سازی و ترتیب). Offset: شماره‌ی ترتیبیِ یکتا درونِ یک partition. Consumer Group: گروهی که partitionها بینش تقسیم می‌شود (هر partition به یک Consumer). Retention: مدت/حجمِ نگه‌داریِ پیام‌ها.

</details>

<details><summary>**ترتیب در Kafka چطور تضمین می‌شود؟ اگر ترتیبِ کلیدِ یک کاربر مهم باشد چه می‌کنی؟**</summary>

ترتیب فقط **درونِ یک partition** تضمین می‌شود، نه در کلِ Topic. برای حفظِ ترتیبِ یک موجودیت، با همان کلید پارتیشن‌بندی کن (`hash(key) % partitions`) تا همه‌ی پیام‌های آن کلید به یک partition بروند و ترتیبشان حفظ شود.

</details>

<details><summary>**اگر Consumerهای یک گروه بیشتر از partitionها باشند چه می‌شود؟**</summary>

چون هر partition فقط به یک Consumer در گروه داده می‌شود، Consumerهای اضافی **بیکار** می‌مانند. موازی‌سازیِ مصرف محدود به تعدادِ partitionهاست؛ برای موازی‌سازیِ بیشتر باید partition بیشتری بسازی.

</details>

### Event Streaming

<details><summary>**فرق event و command چیست؟**</summary>

Command درخواستِ انجامِ کاری به گیرنده‌ی مشخص است (امری: CreateOrder) و انتظارِ اجرا دارد. Event اعلامِ چیزی است که **اتفاق افتاده** (گذشته: OrderCreated)، به گیرنده‌ی خاصی نیست و هرکس مشترک شد واکنش می‌دهد. Event، coupling کمتری می‌سازد.

</details>

<details><summary>**event sourcing و CQRS را توضیح بده.**</summary>

Event Sourcing: به‌جای ذخیره‌ی حالتِ فعلی، **کلِ تاریخِ رویدادها** را ذخیره می‌کنی و حالت را با replayِ آن‌ها می‌سازی (audit کامل، replay، time-travel). CQRS: جداکردنِ مدلِ نوشتن (command) از مدلِ خواندن (query) تا هرکدام مستقل بهینه شوند؛ اغلب با event sourcing جفت می‌شود.

</details>

<details><summary>**مزیت اصلی معماری رویدادمحور چیست؟**</summary>

decoupling در سطحِ سیستم: تولیدکننده نمی‌داند چه کسانی گوش می‌دهند. افزودنِ یک سرویسِ جدید فقط یک subscribe است، بدونِ تغییرِ تولیدکننده — قابلیتِ توسعه و انعطافِ بالا. هزینه‌اش: ردیابی و دیباگِ سخت‌تر (eventual consistency).

</details>

### کشینگ و Redis

<details><summary>**cache-aside در برابر write-through؟**</summary>

cache-aside: کد اول cache را می‌خواند؛ miss شد از DB می‌خواند و cache را پر می‌کند؛ هنگامِ نوشتن، DB را آپدیت و کلید را invalidate می‌کند. فقط داده‌ی واقعاً خواسته‌شده کش می‌شود. write-through: هر نوشتن هم‌زمان به cache و DB می‌رود؛ cache همیشه تازه ولی نوشتن کندتر و ممکن است داده‌ی بی‌مصرف کش شود.

</details>

<details><summary>**cache و DB ناسازگار شده‌اند — کدام استراتژی invalidation؟**</summary>

معمولاً cache-aside + **delete-on-write**: پس از نوشتن در DB، کلید را **حذف** کن (نه آپدیت)، تا خواندنِ بعدی دوباره از DB پرش کند. این پنجره‌ی race بینِ نوشتن‌های هم‌زمان را کوچک می‌کند (update-on-write می‌تواند مقدارِ کهنه را بنویسد). برای سازگاریِ قوی‌تر: versioning یا CDC.

</details>

<details><summary>**cache stampede چیست و چطور جلوگیری می‌کنی؟**</summary>

وقتی یک کلیدِ داغ منقضی می‌شود، هزاران درخواست هم‌زمان miss می‌خورند و به DB هجوم می‌برند و می‌خوابانندش. راه‌حل‌ها: locking/single-flight (فقط یک بازسازی، بقیه منتظر یا مقدارِ کهنه)، TTL jitter (پراکنده‌کردنِ انقضاها)، و recomputationِ زودهنگام/احتمالاتی قبل از انقضا.

</details>

<details><summary>**«دو مشکل سخت علوم کامپیوتر» چیست؟**</summary>

نقلِ معروف: «cache invalidation و naming things» (و شوخی‌اش: off-by-one errors). مفهوم: تشخیصِ اینکه **چه زمانی** کشِ کهنه شده و باید پاک شود، یکی از سخت‌ترین مسائلِ عملی است.

</details>

<details><summary>**پنج کاربرد Redis و ساختار دادهٔ مناسب هرکدام؟**</summary>

Cache (String با TTL)، Session (String/Hash با TTL)، Distributed Lock (`SET NX PX`)، Queue (List با `LPUSH`/`BRPOP`)، Pub/Sub (`PUBLISH`/`SUBSCRIBE`). به‌علاوه leaderboard/rate-limit با Sorted Set و log سبک با Stream.

</details>

<details><summary>**قفل توزیع‌شدهٔ Redis چه ریسکی دارد؟**</summary>

اگر صاحبِ قفل قبل از آزادسازی کند شود و TTL منقضی شود، دیگری همان قفل را می‌گیرد و دو نفر هم‌زمان واردِ بخشِ بحرانی می‌شوند. لذا آزادسازی باید **token را اتمیک چک کند** (با Lua) و عملیاتِ بحرانی بهتر است idempotent باشد. برای استحکام بیشتر Redlock مطرح است اما بحث‌برانگیز.

</details>

### تحمل خطا و دسترس‌پذیری

<details><summary>**چرا retry باید backoff + jitter داشته باشد؟**</summary>

retryِ فوری و هم‌زمانِ همه‌ی کلاینت‌ها سرورِ در حالِ ترمیم را دوباره زمین می‌زند (**retry storm**). exponential backoff فاصله را فزاینده می‌کند و jitter آن را تصادفی می‌کند تا کلاینت‌ها هم‌زمان هجوم نیاورند. باید سقفِ تلاش/تأخیر داشته باشد و فقط روی خطاهای گذرا (نه 400) اعمال شود.

</details>

<details><summary>**سه حالت circuit breaker و گذارها؟**</summary>

CLOSED (عبورِ عادی، شمارشِ خطا) → با عبورِ نرخِ خطا از حد → OPEN (ردِ فوری بدون تماس، fail-fast) → پس از cooldown → HALF-OPEN (چند درخواستِ آزمایشی) → موفق به CLOSED، ناموفق به OPEN. هدف: ندادنِ فشار به سرویسِ مریض و آزادکردنِ سریعِ منابع.

</details>

<details><summary>**timeout چرا حیاتی است؟**</summary>

بدونِ timeout، یک Nodeِ کُند/مرده می‌تواند threadها و connectionهای کلاینت را نامحدود مشغول نگه دارد و باعثِ cascading failure شود. timeout منابع را آزاد می‌کند و امکانِ تصمیمِ retry/circuit-breaker را می‌دهد. باید کوتاه‌تر از صبرِ کاربر و متناسب با p99 سرویس باشد.

</details>

<details><summary>**جدول نُه‌ها: 99.9% در برابر 99.99% چقدر downtime؟**</summary>

99.9% (سه نُه) ≈ ۸.۸ ساعت در سال؛ 99.99% (چهار نُه) ≈ ۵۲ دقیقه در سال؛ 99.999% (پنج نُه) ≈ ۵ دقیقه در سال. هر نُهِ اضافه یعنی ~۱۰ برابر کاهشِ downtime و معمولاً جهشِ بزرگِ هزینه و پیچیدگی.

</details>

<details><summary>**active-active در برابر active-passive؟**</summary>

active-passive: یک نسخه فعال، یکی standby؛ ساده ولی پشتیبان بیکار و failover کمی کندتر. active-active: همه فعال و بار تقسیم‌شده؛ بهره‌ورتر و failover تقریباً آنی، ولی پیچیده‌تر (هماهنگیِ داده و خطرِ split-brain).

</details>

<details><summary>**split-brain چیست و چطور جلوگیری می‌شود؟**</summary>

در partition شبکه، دو طرف هم‌زمان خود را اصلی می‌پندارند و هر دو می‌نویسند → داده دوشاخه و خراب می‌شود. جلوگیری: quorum برای انتخابِ تنها یک اصلی، fencing tokenها، یا STONITH (خاموش‌کردنِ Nodeِ مشکوک). به همین خاطر سیستم‌های CP در partition ترجیح می‌دهند بایستند تا بنویسند.

</details>

<details><summary>**فرق liveness و readiness probe؟**</summary>

liveness: «آیا پروسه زنده است؟» اگر نه، orchestrator آن را **restart** می‌کند. readiness: «آیا آماده‌ی دریافتِ ترافیک است؟» اگر نه، از load balancer **خارج** می‌شود اما restart نمی‌شود (مثلاً هنگامِ گرم‌شدن یا قطعِ موقتِ وابستگی). اشتباه‌گرفتنشان باعثِ restartهای بی‌مورد یا فرستادنِ ترافیک به Nodeِ ناآماده می‌شود.

</details>

<details><summary>**یک Node کُند چطور می‌تواند کل سیستم را بخواباند (cascading failure) و چه دفاعی داری؟**</summary>

Nodeِ کُند، threadها/connectionهای فراخوان را مشغول نگه می‌دارد؛ صف‌ها پر می‌شوند و کندی به سرویس‌های بالادست سرایت می‌کند. دفاع‌ها: timeoutِ سخت‌گیرانه، circuit breaker (fail-fast)، bulkhead (جداسازیِ منابع)، load shedding، و backpressure.

</details>

<details><summary>**چطور یک سرویس بدون‌حالت (stateless) به HA کمک می‌کند؟**</summary>

اگر سرور stateless باشد (state در Redis/DB مشترک)، هر درخواست به هر instance می‌تواند برود؛ افزودن/حذف instance و failover ساده می‌شود و نیازی به sticky session نیست. این پایه‌ی horizontal scaling و active-active است.

</details>

> این مجموعه همه‌ی زیرموضوع‌های فصل را پوشش می‌دهد: از مبانی و اجماع تا صف، Kafka، event streaming، caching، Redis، تحمل خطا و HA.
