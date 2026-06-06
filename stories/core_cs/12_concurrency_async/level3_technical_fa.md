# سطح ۳ — توضیح فنی: Concurrency & Async Programming ⚙️

> دیگر قصه‌ای در کار نیست. این سند مرجعِ فنی است — همان چیزهایی که در مصاحبه و کارِ واقعی لازم داری، از دیدِ **برنامه‌نویس**. (دیدِ سیستم‌عامل به scheduling و context switch در فصلِ سیستم‌عامل آمده؛ اینجا تکرارش نمی‌کنیم.)

---

## ۱. Concurrency در برابر Parallelism

دو مفهومی که مدام قاطی می‌شوند:

- **Concurrency (هم‌روندی):** ساختاردهیِ برنامه به چند کارِ مستقل که پیشرفتشان **همپوشانی زمانی** دارد. لزوماً همزمانِ واقعی نیست — روی یک هسته هم با تعویضِ سریع ممکن است. مسئله‌ی **مدیریتِ کارها**ست.
- **Parallelism (موازی‌سازی):** اجرای واقعاً **هم‌زمانِ** چند محاسبه روی چند هسته/CPU. مسئله‌ی **سرعتِ اجرا**ست.

جمله‌ی معروفِ Rob Pike: «Concurrency is about *dealing with* lots of things at once; parallelism is about *doing* lots of things at once.»

```
Parallelism (۲ هسته):            Concurrency (۱ هسته، interleaving):
core0: AAAAAAAA                  core0: AABBAABBAABB
core1: BBBBBBBB                         (نوبتی، به نظر همزمان)
```

> نکته‌ی مصاحبه: Concurrency بدونِ Parallelism کاملاً ممکن است (مثلاً asyncio روی یک هسته). Parallelism بدونِ Concurrency هم در حالتِ خاص (data parallelism روی داده‌ی مستقل) معنا دارد. این دو متعامد (orthogonal) هستند.

---

## ۲. Race Condition

**تعریف:** وقتی درستیِ نتیجه به **ترتیب/زمان‌بندیِ** اجرای چند نخ وابسته باشد و آن ترتیب تضمین‌شده نباشد.

نمونه‌ی کلاسیک، همان `counter += 1` است که یک عملِ **اتمی نیست**، بلکه یک **Read-Modify-Write** سه‌مرحله‌ای است:

```
LOAD   counter → register      (READ)
ADD    register, 1             (MODIFY)
STORE  register → counter      (WRITE)
```

اگر بینِ READ و WRITEِ یک نخ، نخِ دیگری همان متغیر را بخواند، هر دو روی مقدارِ قدیمی حساب می‌کنند و یک به‌روزرسانی **گم** می‌شود (lost update). با دو نخ که هر کدام یک میلیون بار افزایش می‌دهند، نتیجه‌ی نهایی تقریباً همیشه **کمتر از** ۲٬۰۰۰٬۰۰۰ درمی‌آید.

> نکته‌ی مصاحبه (مهم‌ترین تله‌ی این فصل): «GIL مگر اجازه نمی‌دهد فقط یک نخ کار کند؟ پس چرا race داریم؟» — GIL فقط تضمین می‌کند هر **بایت‌کد** اتمی اجرا شود، ولی `counter += 1` به **چند بایت‌کد** کامپایل می‌شود و مفسر می‌تواند وسطِ آن‌ها نخ را عوض کند (هر چند بایت‌کد، یا با آزاد شدنِ GIL). پس race در پایتون **با وجودِ GIL** هم واقعی است. خودِ `counter += 1` اتمی نیست، حتی اگر `counter.append(x)` روی یک list معمولاً اتمی باشد. به این رفتارهای ظریف تکیه نکن؛ صریحاً قفل بگذار.

**Critical Section:** بخشی از کد که حداکثر یک نخ هم‌زمان باید واردش شود. هدفِ همگام‌سازی، محافظت از همین بخش است.

---

## ۳. Lock / Mutex و عملیاتِ اتمی

**Mutex (Mutual Exclusion):** قفلی که هم‌زمان فقط یک نخ نگهش می‌دارد. ورود به critical section را سریالی می‌کند.

```python
import threading

lock = threading.Lock()
with lock:                 # acquire خودکار
    counter += 1
# release خودکار در پایانِ بلوک — حتی هنگامِ exception
```

انواع و مفاهیمِ مرتبط:

| ابزار | کاربرد |
|---|---|
| `Lock` | قفلِ ساده، غیرِ بازگشتی (همان نخ دوبار acquire کند → deadlock با خودش) |
| `RLock` | قفلِ بازگشتی؛ همان نخ می‌تواند چند بار acquire کند (برای کدِ بازگشتی) |
| `Semaphore(n)` | اجازه به حداکثر `n` نخِ هم‌زمان (مثلاً سقفِ ۱۰ اتصالِ همزمان) |
| `Event` | سیگنالِ یک‌بارمصرف بینِ نخ‌ها (منتظرِ یک شرط بمان) |
| `Condition` | انتظار/بیدارباش روی یک شرط (الگوی producer/consumer) |
| `threading.local()` | داده‌ی مختصِ هر نخ — اشتراکی نیست، پس race ندارد |

- **دانه‌بندیِ قفل (Granularity):** قفلِ **درشت** (یک قفل برای کلِ ساختار) ساده ولی نقطه‌ی گلوگاه (contention) می‌سازد. قفلِ **ریز** (قفل به‌ازای هر بخش) همروندی بیشتر ولی پیچیدگی و خطرِ deadlock بالاتر. تعادل را بر اساسِ contention واقعی پیدا کن، نه حدس.
- **عملیاتِ اتمی (Atomic):** عملی که تجزیه‌ناپذیر است. در پایتون به‌جای قفل برای شمارش می‌توان از `itertools.count` یا قفلِ خود کرد؛ در زبان‌های سیستمی از atomic CPU instructions (مثلِ Compare-And-Swap / CAS) استفاده می‌شود که پایه‌ی ساختارهای **lock-free** است.
- **هزینه:** قفل، سریالی‌سازی و در نتیجه افتِ همروندی می‌آورد. کمترین کدِ ممکن را داخلِ قفل بگذار (critical section را کوتاه نگه دار).

---

## ۴. Deadlock، Livelock، Starvation

### Deadlock و چهار شرطِ Coffman

Deadlock وقتی رخ می‌دهد که مجموعه‌ای از نخ‌ها هر کدام منتظرِ منبعی‌اند که دستِ دیگری است — یک **حلقه‌ی انتظار**. برای ممکن شدنِ deadlock هر **چهار** شرطِ زیر باید هم‌زمان برقرار باشند؛ **شکستنِ هر کدام** کافی است:

| شرط (Coffman) | معنی | راهِ شکستن |
|---|---|---|
| Mutual Exclusion | منبع فقط دستِ یک نفر | منابعِ اشتراک‌پذیر (کم‌کاربرد) |
| Hold and Wait | منبعی را نگه‌داشته و منتظرِ بعدی است | همه‌ی قفل‌ها را یک‌جا بگیر یا هیچ |
| No Preemption | نمی‌توان منبع را به‌زور پس گرفت | timeout روی acquire + رهاسازی |
| **Circular Wait** | حلقه‌ی انتظار | **Lock Ordering** ← رایج‌ترین درمان |

**درمانِ عملی و رایج — Lock Ordering:** همیشه قفل‌ها را به **یک ترتیبِ سراسریِ ثابت** بگیر (مثلاً بر اساسِ `id` یا نامِ منبع). اگر همه‌ی نخ‌ها اول `A` بعد `B` بگیرند، حلقه‌ی `A→B→A` هرگز بسته نمی‌شود.

```python
# به جای اینکه نخی اول l1 و نخی اول l2 بگیرد، همیشه به ترتیبِ id قفل کن:
def transfer(a, b):
    first, second = sorted([a, b], key=id)
    with first.lock:
        with second.lock:
            ...
```

راهِ دیگر: **`acquire(timeout=...)`** — اگر در مهلت نگرفتی، آنچه داری را رها کن و دوباره تلاش کن (می‌تواند به livelock منجر شود، پس backoff تصادفی اضافه کن).

### Livelock

نخ‌ها **گیر نکرده‌اند** (مدام در حال تغییرِ حالت‌اند) ولی **هیچ پیشرفتی** نمی‌کنند — مثلِ دو نفر در راهرو که هم‌زمان به یک سمت کنار می‌روند و باز روبه‌رو می‌شوند. علتِ رایج: واکنشِ متقارنِ همه به برخورد. درمان: نامتقارن کردن (backoff با مقدارِ **تصادفی** متفاوت برای هر نخ).

### Starvation

نخی هرگز به منبع نمی‌رسد چون نخ‌های دیگر (با اولویتِ بالاتر یا شانسِ بیشتر) همیشه جلویش را می‌گیرند. خودش گیر نیست، فقط نوبتش نمی‌شود. درمان: زمان‌بندیِ منصف، **aging** (بالا بردنِ تدریجیِ اولویتِ منتظرانِ قدیمی)، یا قفل‌های منصف (fair lock / FIFO).

---

## ۵. Threads در برابر Processes و GIL

| | Thread | Process |
|---|---|---|
| فضای آدرس | مشترک (heap, globals) | جدا/ایزوله |
| ارتباط | مستقیم از حافظه‌ی مشترک (نیاز به قفل) | IPC: pipe، queue، shared memory، socket |
| هزینه‌ی ساخت | سبک | سنگین (fork/spawn) |
| ایزولاسیونِ خطا | ضعیف (crash یکی = کلِ پروسه) | قوی |
| اثرِ GIL در پایتون | محدود به یک هسته برای کارِ CPU | هر پروسه GIL مستقل → موازیِ واقعی |
| مناسبِ | I/O-bound، اشتراکِ سریعِ state | CPU-bound، ایزولاسیونِ امنیتی/پایداری |

### GIL (Global Interpreter Lock)

- در **CPython** یک قفلِ سراسری است که می‌گوید: در هر لحظه فقط **یک نخ** بایت‌کدِ پایتون اجرا کند.
- **چرا threadها کارِ CPU-bound را در پایتون تند نمی‌کنند؟** چون چند نخ نمی‌توانند هم‌زمان بایت‌کد اجرا کنند؛ نوبتی روی یک هسته می‌روند. افزودنِ نخ نه‌تنها کمک نمی‌کند، با هزینه‌ی context switch و رقابت بر سرِ GIL گاهی **کندتر** می‌شود.
- **چرا برای I/O-bound *خوب* است؟** چون هنگامِ عملیاتِ I/O (شبکه، دیسک)، نخ **GIL را آزاد می‌کند** و نخِ دیگری کار می‌کند. پس threadها برای کارِ پر-انتظار سودمندند.
- **جوابِ CPU-bound:** `multiprocessing` — هر پروسه مفسرِ پایتون و GIL مستقلِ خودش را دارد → موازیِ واقعی روی چند هسته.
- C-extensionها (NumPy و...) معمولاً هنگامِ محاسبه‌ی سنگینِ C، GIL را آزاد می‌کنند؛ پس کارِ سنگینِ numerical گاهی با thread هم موازی می‌شود.

> نکته‌ی مصاحبه: CPython 3.13 یک buildِ آزمایشیِ **free-threaded (بدونِ GIL)** و نیز JIT اولیه معرفی کرد، ولی هنوز پیش‌فرض و پایدار نیست. در مصاحبه مبنا را همان «GIL هست» بگذار و این را به‌عنوانِ روندِ آینده ذکر کن.

---

## ۶. I/O-bound در برابر CPU-bound — جدولِ تصمیم

اول کارت را دسته‌بندی کن، بعد ابزار را انتخاب کن:

| نوعِ کار | گلوگاه | نشانه | ابزارِ پیشنهادی (پایتون) |
|---|---|---|---|
| **I/O-bound** | انتظارِ شبکه/دیسک/DB | CPU بی‌کار، زمان صرفِ wait | `asyncio` (مقیاسِ بالا، هزاران اتصال) یا `ThreadPoolExecutor` |
| **CPU-bound** | محاسبه‌ی سنگین | یک هسته ۱۰۰٪، بقیه بی‌کار | `ProcessPoolExecutor` / `multiprocessing` |
| **ترکیبی** | هر دو | — | پروسه برای محاسبه + async/thread برای I/O داخلِ هر پروسه |

```
                    ┌───────────────────────────┐
   کار I/O-bound?   │  بله → async یا threads    │
   ───────────────► │      (هزاران اتصال → async) │
                    └───────────────────────────┘
                    ┌───────────────────────────┐
   کار CPU-bound?   │  بله → multiprocessing     │
   ───────────────► │      (به‌تعدادِ هسته‌ها)     │
                    └───────────────────────────┘
```

> نکته‌ی مصاحبه: «async تندتر است یا threads؟» — برای کارِ I/O-bondِ پرمقیاس، async (تک‌نخ، event loop) سربارِ کمتری از هزاران thread دارد (هر thread پشته و هزینه‌ی context switch دارد). ولی async کلِ کدِ مسیر را «async-aware» می‌خواهد؛ یک فراخوانیِ مسدودکننده کلِ loop را می‌خواباند. threads با کدِ blocking موجود سازگارترند.

---

## ۷. Async / Await

### اجزای مدل

- **Event Loop:** هسته‌ی asyncio. یک حلقه که taskهای آماده را اجرا می‌کند، و وقتی task به `await` می‌رسد، آن را معلق و سراغِ بعدی می‌رود. وقتی I/O آماده شد (با `select`/`epoll`)، task را ادامه می‌دهد.
- **Coroutine:** تابعِ `async def`. صدا زدنش **اجرایش نمی‌کند**، فقط یک شیء coroutine می‌سازد؛ باید `await` یا روی loop schedule شود.
- **`await`:** نقطه‌ای که coroutine کنترل را به loop پس می‌دهد تا کارِ دیگری پیش برود.
- **Task:** یک coroutine که روی loop **زمان‌بندی** شده تا همروند اجرا شود (`asyncio.create_task`).
- **Non-blocking I/O:** عملیاتی که به‌جای مسدود کردنِ نخ، فوراً برمی‌گردد و بعداً «آماده شدن» را اطلاع می‌دهد.

```python
import asyncio

async def fetch(name, delay):
    print(f"{name}: شروع")
    await asyncio.sleep(delay)          # I/O شبیه‌سازی‌شده، non-blocking
    print(f"{name}: تمام")
    return name

async def main():
    # سه کار همروند؛ gather نتایج را به‌ترتیبِ ورودی برمی‌گرداند
    results = await asyncio.gather(
        fetch("A", 2), fetch("B", 1), fetch("C", 3),
    )
    print(results)                      # ['A', 'B', 'C']  (کلِ زمان ~۳ث نه ۶ث)

asyncio.run(main())
```

### قانونِ طلایی: «حلقه را بند نیار» (Don't block the loop)

asyncio **تک‌نخ** است. هر کارِ مسدودکننده داخلِ یک coroutine، **کلِ** برنامه را می‌خواباند — همه‌ی taskها، همه‌ی اتصال‌ها:

```python
# ❌ غلط — کلِ loop را ۵ ثانیه می‌خواباند
async def handler():
    time.sleep(5)               # blocking! همه‌ی درخواست‌های دیگر هم منتظر می‌مانند

# ✅ درست — I/O غیرِ مسدودکننده
async def handler():
    await asyncio.sleep(5)

# ✅ برای کارِ CPU-bound یا کتابخانه‌ی blocking: ببرش روی executor
async def handler():
    loop = asyncio.get_running_loop()
    await loop.run_in_executor(None, blocking_function, arg)   # روی thread/process pool
```

### الگوهای پرکاربرد

- **`gather`** — چند coroutine را همروند اجرا کن و منتظرِ همه بمان. (پیش‌فرض با اولین exception می‌شکند مگر `return_exceptions=True`.)
- **`asyncio.wait_for(coro, timeout)`** — مهلت بگذار.
- **`asyncio.Queue`** — صفِ async برای producer/consumer روی همان loop.
- **`asyncio.Lock` / `Semaphore`** — همگام‌سازی **بینِ taskها** (نه threadها). مثلاً `Semaphore` برای محدود کردنِ تعدادِ درخواست‌های هم‌زمان.

> نکته‌ی مصاحبه: HTTP واقعی در async به کتابخانه‌ی async مثلِ `aiohttp`/`httpx` نیاز دارد. استفاده از `requests` (که blocking است) داخلِ coroutine، کلِ loop را می‌خواباند — اشتباهِ بسیار رایج.

---

## ۸. Thread Safety در عمل

سه استراتژی، به ترتیبِ ارجحیت:

1. **حافظه‌ی مشترک را حذف کن، پیام رد و بدل کن.** به‌جای قفل روی state مشترک، از `queue.Queue` (بینِ threadها) یا `asyncio.Queue` (بینِ taskها) استفاده کن. این الگوی «communicate, don't share» است.
2. **تغییرناپذیری (Immutability).** داده‌ای که عوض نمی‌شود، نیازی به همگام‌سازی ندارد. از tuple، frozenset، dataclass(frozen=True) و کپی به‌جای تغییر در محل استفاده کن.
3. **ساختارهای thread-safe.** `queue.Queue`، `collections.deque` (append/pop از دو سر اتمی است)، و در سطحِ پایین‌تر atomic/CAS. در سایر موارد، خودت با `Lock`/`RLock` محافظت کن.

```python
import queue, threading

q = queue.Queue()
def producer(): 
    for i in range(100): q.put(i)
def consumer():
    while True:
        item = q.get()              # thread-safe، خودش قفل دارد
        process(item)
        q.task_done()
```

> نکته‌ی مصاحبه: `print` از چند نخ ممکن است خروجی را درهم کند چون چند فراخوانی است؛ برای لاگِ امنِ همروند از `logging` (که قفل دارد) یا یک queue استفاده کن.

---

## ۹. باگ‌های رایج

| باگ | علت | درمان |
|---|---|---|
| **`await` فراموش‌شده** | `foo()` به‌جای `await foo()` | coroutine ساخته شد ولی اجرا نشد؛ هشدارِ `coroutine was never awaited`. همیشه await کن یا `create_task`. |
| **Shared mutable default** | `def f(x, acc=[])` | لیست یک‌بار ساخته و بینِ همه‌ی فراخوانی‌ها مشترک می‌ماند. از `acc=None` و `acc = acc or []` استفاده کن. |
| **قفلی که آزاد نمی‌شود** | exception بینِ acquire و release | همیشه `with lock:` (context manager) تا آزادسازیِ تضمین‌شده. |
| **blocking در coroutine** | `time.sleep`/`requests` داخلِ async | کلِ loop می‌خوابد؛ از معادلِ async یا `run_in_executor` استفاده کن. |
| **deadlock با ترتیبِ ناهماهنگ** | نخ‌ها قفل‌ها را به ترتیب‌های مختلف می‌گیرند | Lock ordering سراسری. |
| **thread برای CPU-bound در پایتون** | انتظارِ شتاب از threadها | به‌خاطرِ GIL شتاب نمی‌گیرد؛ از multiprocessing استفاده کن. |

---

## 🔬 آزمایشگاه

سه برنامه‌ی **اجراشدنی** (فقط کتابخانه‌ی استاندارد، بدونِ نصبِ چیزی). هرکدام را در یک فایل بریز و با `python file.py` اجرا کن.

### آزمایش ۱ — Race Condition: بدونِ قفل (غلط) در برابر با قفل (درست)

```python
import threading

ITERS = 1_000_000

def run(use_lock):
    counter = 0
    lock = threading.Lock()

    def worker():
        nonlocal counter
        for _ in range(ITERS):
            if use_lock:
                with lock:
                    counter += 1          # محافظت‌شده — اتمی نسبت به نخِ دیگر
            else:
                counter += 1              # READ-MODIFY-WRITE محافظت‌نشده → race

    t1 = threading.Thread(target=worker)
    t2 = threading.Thread(target=worker)
    t1.start(); t2.start()
    t1.join(); t2.join()
    return counter

expected = 2 * ITERS
print(f"انتظار:            {expected}")
print(f"بدونِ قفل (غلط):    {run(False)}   ← معمولاً کمتر؛ به‌روزرسانی‌ها گم می‌شوند")
print(f"با قفل (درست):     {run(True)}    ← همیشه درست")
```

خروجیِ نمونه (نسخه‌ی بدونِ قفل معمولاً عددی **کمتر** از ۲٬۰۰۰٬۰۰۰ و متفاوت در هر اجرا می‌دهد):

```
انتظار:            2000000
بدونِ قفل (غلط):    1284551       ← هر اجرا عددی متفاوت و کمتر
با قفل (درست):     2000000
```

> ⚠️ نکته‌ی مهم درباره‌ی این آزمایش: گم شدنِ به‌روزرسانی **احتمالی** است و به نسخه‌ی پایتون، `switch interval` و میزانِ رقابت بستگی دارد. در CPython جدید (مثلاً 3.13/3.14) به‌خاطرِ مفسرِ تطبیقیِ (adaptive) که حلقه را فشرده نگه می‌دارد، ممکن است در یک حلقه‌ی تنگ همان ۲٬۰۰۰٬۰۰۰ را ببینی — این **شانسِ زمان‌بندی** است، نه امن بودن. `counter += 1` همچنان اتمی **نیست** (به LOAD / BINARY_OP / STORE کامپایل می‌شود و تعویضِ نخ می‌تواند بینِ آن‌ها بیفتد) و قفل همچنان **واجب** است. اگر race را ندیدی، با `sys.setswitchinterval(1e-6)` و تعدادِ نخِ بیشتر، پنجره‌ی آسیب‌پذیر را بازتر کن تا گم شدنِ به‌روزرسانی ظاهر شود.

### آزمایش ۲ — asyncio.gather در برابر اجرای پشتِ سرِ هم (زمان‌بندی)

```python
import asyncio, time

async def fake_request(name, delay):
    await asyncio.sleep(delay)          # شبیه‌سازیِ I/O شبکه (non-blocking)
    return f"{name} بعدِ {delay}s"

DELAYS = [("A", 1), ("B", 1), ("C", 1), ("D", 1), ("E", 1)]

async def sequential():
    t0 = time.perf_counter()
    for name, d in DELAYS:
        await fake_request(name, d)     # یکی‌یکی، پشتِ هم
    return time.perf_counter() - t0

async def concurrent():
    t0 = time.perf_counter()
    await asyncio.gather(*(fake_request(n, d) for n, d in DELAYS))
    return time.perf_counter() - t0

async def main():
    print(f"پشتِ سرِ هم: {await sequential():.2f}s   (~۵ ثانیه = جمعِ همه)")
    print(f"با gather:   {await concurrent():.2f}s   (~۱ ثانیه = طولانی‌ترین)")

asyncio.run(main())
# نکته: این sleepها جایگزینِ I/O واقعی شبکه‌اند. برای HTTP واقعی به aiohttp/httpx
# نیاز داری (pip install) — چون requests مسدودکننده است و loop را می‌خواباند.
```

### آزمایش ۳ — ThreadPool در برابر ProcessPool روی کارِ CPU-bound (اثرِ GIL)

```python
import time
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def cpu_heavy(n):
    # کارِ محاسباتیِ خالص — هیچ I/O ای ندارد
    total = 0
    for i in range(n):
        total += i * i
    return total

TASKS = [10_000_000] * 4               # ۴ کارِ سنگین

def timed(executor_cls, label):
    t0 = time.perf_counter()
    with executor_cls(max_workers=4) as ex:
        list(ex.map(cpu_heavy, TASKS))
    print(f"{label:18}: {time.perf_counter() - t0:.2f}s")

if __name__ == "__main__":             # لازم برای multiprocessing روی ویندوز/مک
    timed(ThreadPoolExecutor,  "ThreadPool (GIL)")    # کند: نخ‌ها روی یک هسته نوبتی
    timed(ProcessPoolExecutor, "ProcessPool")         # سریع‌تر: چند هسته‌ی واقعی
```

نتیجه‌ی موردِ انتظار روی ماشینِ چند-هسته‌ای: `ProcessPool` به‌مراتب سریع‌تر است، چون threadها به‌خاطرِ GIL نمی‌توانند بایت‌کدِ CPU-bound را موازی اجرا کنند، اما processها هر کدام GIL خودشان را دارند.

---

## ✅ چک‌لیست تسلط

اگر بتوانی به این‌ها جواب بدهی، این فصل را بلدی:

- [ ] فرقِ دقیقِ Concurrency و Parallelism؟ یک مثال از هرکدام بدونِ آن یکی.
- [ ] چرا `counter += 1` از دو نخ به‌روزرسانی گم می‌کند؟ سه مرحله‌اش را نام ببر.
- [ ] چرا GIL باعث **نمی‌شود** که `counter += 1` امن باشد؟
- [ ] چهار شرطِ Coffman برای deadlock؟ کدام را با Lock Ordering می‌شکنیم؟
- [ ] فرقِ Deadlock، Livelock و Starvation در یک جمله برای هرکدام؟
- [ ] چرا threadهای پایتون کارِ CPU-bound را تند نمی‌کنند؟ جایگزینش؟
- [ ] جدولِ تصمیم: I/O-bound → ؟  و  CPU-bound → ؟
- [ ] Event loop، coroutine و await هرکدام چه می‌کنند؟
- [ ] قانونِ «حلقه را بند نیار» یعنی چه و چطور با `run_in_executor` رعایتش می‌کنی؟
- [ ] چرا Queue امن‌تر از حافظه‌ی مشترک است؟ سه ستونِ thread safety؟
- [ ] سه باگِ رایج (await فراموش‌شده، default تغییرپذیر، قفلِ آزادنشده) و درمانشان؟

---

## 🎯 سوالات مصاحبه

> روی هر سوال کلیک کن تا جواب را ببینی. اول خودت جواب بده، بعد چک کن!

### Concurrency در برابر Parallelism

<details><summary>**۱. فرقِ Concurrency و Parallelism چیست؟**</summary>

Concurrency یعنی **مدیریتِ** چند کار با پیشرفتِ همپوشان (حتی روی یک هسته با interleaving)؛ مسئله‌ی ساختار است. Parallelism یعنی اجرای **واقعاً هم‌زمانِ** چند محاسبه روی چند هسته؛ مسئله‌ی سرعتِ اجراست. می‌توان Concurrency بدونِ Parallelism داشت (asyncio روی یک هسته).

</details>

<details><summary>**۲. آیا می‌شود concurrency داشت بدونِ هیچ parallelism؟**</summary>

بله. asyncio روی یک هسته دقیقاً همین است: چندین task پیشرفتِ همپوشان دارند، ولی در هر لحظه فقط یکی روی CPU است. همروندی از طریقِ تعویض در نقاطِ `await` حاصل می‌شود، نه از چند هسته.

</details>

### Race Condition

<details><summary>**۳. چرا `counter += 1` از دو نخ ممکن است به‌روزرسانی گم کند؟**</summary>

چون `+=` یک عملِ اتمی نیست بلکه Read-Modify-Write است: مقدار را می‌خوانَد، یکی اضافه می‌کند، می‌نویسد. اگر نخِ دوم بینِ READ و WRITEِ نخِ اول، همان مقدارِ قدیمی را بخواند، هر دو روی آن حساب می‌کنند و یکی از افزایش‌ها بازنویسی (گم) می‌شود.

</details>

<details><summary>**۴. Critical Section چیست؟**</summary>

بخشی از کد که دسترسی به منبعِ مشترک دارد و باید تضمین شود حداکثر یک نخ هم‌زمان واردش می‌شود. هدفِ قفل/همگام‌سازی محافظت از همین ناحیه است؛ هرچه کوتاه‌تر، همروندی بهتر.

</details>

<details><summary>**۵. سناریو: یک تستِ همروند گاهی پاس می‌شود و گاهی فِیل — احتمالاً چه خبر است؟**</summary>

نشانه‌ی کلاسیکِ race condition (یا «heisenbug»): نتیجه به ترتیبِ زمان‌بندیِ نخ‌ها وابسته است که هر اجرا فرق می‌کند. دنبالِ stateِ مشترکِ محافظت‌نشده بگرد؛ با قفل، تغییرناپذیری یا انتقال به queue حلش کن. تستِ صرفِ «چند بار پاس شد» اثباتِ درستی نیست.

</details>

### Lock / Mutex

<details><summary>**۶. Mutex چیست و چرا lock granularity مهم است؟**</summary>

Mutex قفلی است که هم‌زمان فقط یک نخ نگهش می‌دارد و critical section را سریالی می‌کند. Granularity درشت (یک قفلِ بزرگ) ساده ولی گلوگاهِ contention است؛ ریز (چند قفلِ کوچک) همروندیِ بیشتر ولی پیچیدگی و خطرِ deadlock بالاتر. تعادل بر اساسِ contention واقعی.

</details>

<details><summary>**۷. فرقِ `Lock` و `RLock` در پایتون؟**</summary>

`Lock` غیرِبازگشتی است: اگر همان نخ که قفل را دارد دوباره acquire کند، با خودش deadlock می‌شود. `RLock` (reentrant) شمارنده نگه می‌دارد و اجازه می‌دهد همان نخ چند بار acquire کند (به همان تعداد باید release کند) — برای توابعِ بازگشتی یا متدهایی که هم‌دیگر را زیرِ یک قفل صدا می‌زنند.

</details>

<details><summary>**۸. چرا همیشه باید از `with lock:` به‌جای acquire/release دستی استفاده کرد؟**</summary>

چون اگر بینِ `acquire()` و `release()` یک exception رخ دهد، قفل هرگز آزاد نمی‌شود و بقیه‌ی نخ‌ها برای همیشه گیر می‌کنند. context manager (`with`) آزادسازی را حتی هنگامِ خطا تضمین می‌کند.

</details>

<details><summary>**۹. عملیاتِ اتمی یعنی چه و چه ربطی به ساختارهای lock-free دارد؟**</summary>

عملِ اتمی تجزیه‌ناپذیر است و بدونِ قفل امن اجرا می‌شود. CPUها دستوراتی مثلِ Compare-And-Swap (CAS) دارند که پایه‌ی ساختارهای lock-free (مثلِ صف‌ها و شمارنده‌های بدونِ قفل) هستند — کاراییِ بالاتر بدونِ هزینه‌ی قفل، ولی پیاده‌سازیِ بسیار دشوارتر و مستعدِ باگ‌های ظریف.

</details>

### Deadlock، Livelock، Starvation

<details><summary>**۱۰. چهار شرطِ Coffman برای deadlock چیست؟**</summary>

Mutual Exclusion (منبع انحصاری)، Hold and Wait (نگه‌داشتن و انتظار)، No Preemption (عدمِ پس‌گیریِ اجباری)، Circular Wait (حلقه‌ی انتظار). هر چهار باید هم‌زمان برقرار باشند؛ شکستنِ هر کدام deadlock را ناممکن می‌کند.

</details>

<details><summary>**۱۱. سناریو: دو worker هرکدام یکی از دو قفل را می‌گیرند و برنامه می‌خوابد — تشخیص و رفع؟**</summary>

این deadlock با Circular Wait است: worker اول قفلِ A را دارد و منتظرِ B، دومی B را دارد و منتظرِ A. **رفع:** Lock Ordering — همه‌ی نخ‌ها قفل‌ها را به یک ترتیبِ سراسریِ ثابت بگیرند (مثلاً بر اساسِ `id` منبع: `for l in sorted(locks, key=id): ...`). راهِ مکمل: `acquire(timeout=...)` که اگر نگرفت، رها کند و با backoff تصادفی دوباره تلاش کند.

</details>

<details><summary>**۱۲. فرقِ Deadlock، Livelock و Starvation؟**</summary>

Deadlock: نخ‌ها در حلقه‌ی انتظار **متوقف** شده‌اند، هیچ حرکتی نیست. Livelock: نخ‌ها **در حال حرکت‌اند** (مدام واکنش نشان می‌دهند) ولی هیچ پیشرفتی نمی‌کنند. Starvation: نخی **زنده** است ولی هرگز به منبع نمی‌رسد چون دیگران همیشه جلویش را می‌گیرند.

</details>

<details><summary>**۱۳. Lock Ordering چطور deadlock را حذف می‌کند؟**</summary>

با شکستنِ شرطِ Circular Wait: اگر همه‌ی نخ‌ها قفل‌ها را به یک ترتیبِ کلیِ یکسان بگیرند، هیچ حلقه‌ای از وابستگی شکل نمی‌گیرد (نمی‌شود هم A→B و هم B→A داشت)، پس انتظارِ دایره‌ای غیرممکن می‌شود.

</details>

<details><summary>**۱۴. چطور livelock را رفع می‌کنی؟**</summary>

علتِ رایجِ livelock واکنشِ **متقارن** است (همه به یک شکل عقب می‌کشند و باز برخورد می‌کنند). راه‌حل، نامتقارن کردن است: backoff با مقدارِ **تصادفی** (jitter) متفاوت برای هر نخ، تا یکی زودتر جلو بیفتد و حلقه شکسته شود.

</details>

### Threads در برابر Processes و GIL

<details><summary>**۱۵. سناریو: thread اضافه کردم و برنامه‌ی پایتونم *کندتر* شد — چرا؟**</summary>

کارت احتمالاً CPU-bound است. به‌خاطرِ **GIL** فقط یک نخ هم‌زمان بایت‌کد اجرا می‌کند، پس threadها CPU را موازی نمی‌کنند؛ به‌جایش هزینه‌ی context switch و رقابت بر سرِ GIL اضافه شد و کندتر شد. **رفع:** برای CPU-bound از `multiprocessing`/`ProcessPoolExecutor` استفاده کن (هر پروسه GIL مستقل دارد).

</details>

<details><summary>**۱۶. GIL چیست و دقیقاً چه چیزی را تضمین می‌کند و چه چیزی را نه؟**</summary>

قفلِ سراسریِ مفسرِ CPython که هم‌زمان فقط اجرای یک نخِ بایت‌کد را اجازه می‌دهد. تضمین می‌کند **هر بایت‌کد** اتمی اجرا شود، اما **نه** اینکه عملیاتِ سطحِ زبان مثلِ `+=` (که چند بایت‌کد است) اتمی باشد. پس از race جلوگیری نمی‌کند و باز هم به قفل نیاز داری.

</details>

<details><summary>**۱۷. اگر GIL هست، چرا threadها برای کارِ I/O-bound *مفیدند*؟**</summary>

چون هنگامِ عملیاتِ I/O (شبکه/دیسک)، نخ **GIL را آزاد می‌کند** و نخِ دیگری می‌تواند پیش برود. وقتِ انتظار تلف نمی‌شود، پس threadها برای کارِ پر-انتظار همروندیِ واقعی می‌دهند — برخلافِ کارِ CPU-bound.

</details>

<details><summary>**۱۸. فرقِ thread و process از دیدِ حافظه و ایزولاسیون؟**</summary>

threadهای یک پروسه فضای آدرس/heap/globals را **مشترک** دارند (ارتباطِ سریع، ولی نیاز به قفل و خطرِ race؛ crash یکی کلِ پروسه را می‌اندازد). processها فضای آدرسِ **جدا** دارند (ایزوله و امن، crash یکی بقیه را نمی‌اندازد، ولی ارتباط از طریقِ IPC گران‌تر است و ساختشان سنگین‌تر).

</details>

<details><summary>**۱۹. کارِ سنگینِ NumPy را با thread موازی کنم یا process؟**</summary>

برای NumPy غالباً thread هم کار می‌کند، چون عملیاتِ سنگینِ numerical در کدِ C پیاده شده و معمولاً GIL را هنگامِ محاسبه **آزاد** می‌کنند. برای پایتونِ خالصِ CPU-bound اما thread بی‌فایده است و باید process بگیری.

</details>

### I/O-bound در برابر CPU-bound

<details><summary>**۲۰. چطور تصمیم می‌گیری async، threads یا multiprocessing؟**</summary>

اول کار را دسته‌بندی کن. I/O-bound (انتظارِ شبکه/دیسک): async برای مقیاسِ بالا (هزاران اتصال، سربارِ کم) یا threads برای کدِ blockingِ موجود. CPU-bound (محاسبه‌ی سنگین): multiprocessing به‌تعدادِ هسته‌ها. ترکیبی: process برای محاسبه + async/thread برای I/O داخلِ هر process.

</details>

<details><summary>**۲۱. چطور می‌فهمی برنامه‌ات I/O-bound است یا CPU-bound؟**</summary>

اگر هنگامِ کُند بودن، CPU عمدتاً بی‌کار است و زمان صرفِ انتظار (شبکه/دیسک/DB) می‌شود → I/O-bound. اگر یک هسته ۱۰۰٪ مشغول است و بقیه بی‌کارند → CPU-bound (تک‌نخ). پروفایلر و مانیتورِ CPU/IO ابزارِ تشخیص‌اند.

</details>

<details><summary>**۲۲. برای ۱۰٬۰۰۰ اتصالِ شبکه‌ی هم‌زمان، async یا threads؟ چرا؟**</summary>

async. هر thread پشته (مگابایت‌ها) و هزینه‌ی context switch دارد؛ ۱۰٬۰۰۰ thread سنگین و ناکارآمد است. async با یک نخ و event loop هزاران اتصالِ I/O-bound را با سربارِ بسیار کمتر مدیریت می‌کند (مدلِ C10k).

</details>

### Async / Await

<details><summary>**۲۳. event loop چیست و چطور کار می‌کند؟**</summary>

حلقه‌ای تک‌نخ که taskهای آماده را اجرا می‌کند؛ وقتی task به `await`ِ یک I/O می‌رسد، آن را معلق و سراغِ task بعدیِ آماده می‌رود. با مکانیزمِ OS (epoll/kqueue/select) منتظرِ آماده شدنِ I/Oها می‌ماند و taskِ مربوط را ادامه می‌دهد. همروندی از این تعویض در نقاطِ await می‌آید.

</details>

<details><summary>**۲۴. سناریو: یک endpoint داخلش یک کوئریِ کندِ DB را await می‌کند — چه بر سرِ بقیه‌ی درخواست‌ها می‌آید؟**</summary>

بستگی دارد: اگر کوئری با درایورِ **async** و `await` واقعی غیرِ مسدودکننده باشد، event loop در حینِ انتظار به درخواست‌های دیگر رسیدگی می‌کند — بقیه روان‌اند. اما اگر داخلِ coroutine یک فراخوانیِ **مسدودکننده** (درایورِ blocking، `time.sleep`، محاسبه‌ی سنگین) باشد، کلِ event loop می‌خوابد و **همه‌ی** درخواست‌ها تا پایانِ آن منتظر می‌مانند. این جانِ قانونِ «حلقه را بند نیار» است؛ برای کارِ blocking از `run_in_executor` استفاده کن.

</details>

<details><summary>**۲۵. `asyncio.gather` چه می‌کند و چرا تندتر از await پشتِ سرِ هم است؟**</summary>

`gather` چند coroutine را هم‌زمان روی loop زمان‌بندی می‌کند و منتظرِ همه می‌ماند. چون انتظارهای I/O **همپوشانی** می‌کنند، کلِ زمان ≈ طولانی‌ترین کار است، نه جمعِ همه. await پشتِ سرِ هم هر کار را تا آخر منتظر می‌ماند بعد سراغِ بعدی می‌رود (جمعِ زمان‌ها).

</details>

<details><summary>**۲۶. «await فراموش‌شده» چه باگی می‌سازد و چطور می‌فهمی؟**</summary>

اگر coroutine را بدونِ `await` صدا بزنی (`foo()` به‌جای `await foo()`)، فقط یک شیء coroutine ساخته می‌شود که هرگز اجرا نمی‌شود — کار بی‌صدا انجام نمی‌شود. پایتون هشدارِ `RuntimeWarning: coroutine 'foo' was never awaited` می‌دهد. رفع: await کن یا با `create_task` زمان‌بندی کن.

</details>

<details><summary>**۲۷. چرا استفاده از `requests` داخلِ یک coroutine اشتباه است؟**</summary>

`requests` کتابخانه‌ی **مسدودکننده** است؛ هنگامِ منتظرِ پاسخِ شبکه، نخ را بلوکه می‌کند و چون asyncio تک‌نخ است، **کلِ** event loop می‌خوابد و همروندی از بین می‌رود. به‌جایش از کتابخانه‌ی async مثلِ `aiohttp`/`httpx` استفاده کن، یا اگر مجبوری، آن را با `run_in_executor` روی thread pool ببر.

</details>

### Thread Safety و باگ‌های رایج

<details><summary>**۲۸. چرا Queue راهِ امن‌تری از حافظه‌ی مشترک برای ارتباطِ نخ‌هاست؟**</summary>

`queue.Queue` همگام‌سازی را **داخلِ خودش** انجام می‌دهد (thread-safe)، پس به‌جای اینکه چند نخ روی stateِ مشترک قفل بگذارند و ریسکِ race/deadlock کنند، فقط پیام رد و بدل می‌کنند (الگوی «communicate, don't share»). این مالکیتِ داده را واضح می‌کند و خطای انسانی را کم.

</details>

<details><summary>**۲۹. سه ستونِ thread safety در عمل؟**</summary>

۱) حذفِ حافظه‌ی مشترک با انتقالِ پیام (Queue). ۲) تغییرناپذیری (immutable data نیازی به همگام‌سازی ندارد). ۳) استفاده از ساختارهای thread-safe یا محافظت با قفل آنجا که اشتراکِ stateِ تغییرپذیر اجتناب‌ناپذیر است.

</details>

<details><summary>**۳۰. باگِ shared mutable default argument چیست؟**</summary>

`def f(x, acc=[])` — مقدارِ پیش‌فرض **یک‌بار** هنگامِ تعریفِ تابع ساخته می‌شود و بینِ همه‌ی فراخوانی‌ها مشترک می‌ماند؛ پس داده از یک فراخوانی به فراخوانیِ بعد «نشت» می‌کند (و در محیطِ همروند بدتر می‌شود). رفع: `def f(x, acc=None): acc = [] if acc is None else acc`.

</details>

<details><summary>**۳۱. سناریو: لاگِ چند نخ در هم می‌ریزد و خطوط قاطی می‌شوند — چرا و چه کنم؟**</summary>

چون نوشتنِ یک خط ممکن است چند فراخوانیِ I/O باشد و نخ‌ها وسطِ هم می‌نویسند. از ماژولِ `logging` استفاده کن که داخلش قفل دارد و هر رکورد را اتمی می‌نویسد، یا لاگ‌ها را به یک queue بفرست و یک نخِ اختصاصی آن‌ها را بنویسد.

</details>

<details><summary>**۳۲. سناریو: یک سرویس گاهی برای همیشه هنگ می‌کند ولی crash نمی‌دهد — از کجا شروع می‌کنی؟**</summary>

مشکوک به deadlock: thread dump بگیر (در پایتون `faulthandler.dump_traceback` یا `py-spy dump`) و ببین نخ‌ها کجا روی acquireِ قفل گیر کرده‌اند و چه حلقه‌ای از انتظار هست. اگر حلقه دیدی، Lock Ordering را پیاده کن. اگر نخ‌ها در حال کارند ولی پیشرفت نیست، به livelock مشکوک شو.

</details>
