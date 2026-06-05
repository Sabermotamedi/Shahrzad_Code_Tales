# سطح ۳ — توضیح فنی: Software Engineering ⚙️

> دیگر قصه‌ای در کار نیست. این سند مرجع فنی است — همان چیزهایی که در مصاحبه و کار واقعی لازم داری. هر بخش با معادل قصه‌ای‌اش شروع می‌شود تا ذهنت وصل بماند.

---

## ۱. Clean Code (کدِ تمیز)

> در قصه: نقشه‌ی تمیز عمو، اتاق‌های کوچک با اسم درست.

### نام‌گذاری (Naming)

- اسم باید **قصدِ** متغیر/تابع را برساند بدون نیاز به کامنت. `elapsed_days` بهتر از `d`.
- از کلاس‌ها اسمِ **اسم** (Noun)، از توابع اسمِ **فعل** (Verb): `User`، `calculate_total()`.
- اعداد جادویی را به ثابتِ بانام تبدیل کن: `MAX_RETRIES = 3` به‌جای `3`.
- یک مفهوم = یک کلمه؛ همه‌جا `fetch` یا همه‌جا `get`، نه قاطی.

### توابع کوچک (Small Functions)

- **Single Level of Abstraction**: داخل یک تابع، همه‌ی دستورها در یک سطح از انتزاع باشند.
- ایده‌آل: تابع یک کار، در یک صفحه، با کمترین آرگومان (۰ تا ۲ بهترین؛ ۳+ بوی بد).
- از **flag argument** (`do(x, True)`) پرهیز کن — معمولاً یعنی تابع دو کار می‌کند؛ دو تابع بساز.
- پرتاب زود (Guard Clause) به‌جای تو در تویِ `if`:

```python
# ❌ تو در تو
def pay(user):
    if user is not None:
        if user.active:
            if user.balance > 0:
                ...

# ✅ Guard clauses — مسیر اصلی صاف می‌ماند
def pay(user):
    if user is None: raise ValueError("no user")
    if not user.active: raise ValueError("inactive")
    if user.balance <= 0: raise ValueError("no funds")
    ...
```

### کامنت (Comments)

- کامنتِ خوب «**چرا**» را توضیح می‌دهد، نه «چه» را (کد خودش «چه» را می‌گوید).
- کامنت‌های بد: تکرار کد، کامنتِ کهنه/دروغ، کدِ کامنت‌شده (به‌جایش از Git استفاده کن).
- کامنت‌های خوب: توضیح تصمیم، هشدارِ عواقب، `TODO`، docstring قراردادِ عمومی.

> نکته‌ی مصاحبه: «کامنت، عذرخواهی بابت کدِ نارساست.» هر وقت خواستی کامنت بنویسی، اول بپرس آیا می‌توانم با بهتر کردن اسم‌ها، کامنت را حذف کنم؟

---

## ۲. Refactoring

> در قصه: عمو اتاق‌به‌اتاق مرتب می‌کرد، رفتار خانه ثابت می‌ماند.

تعریف دقیق: تغییرِ **ساختار داخلی** کد **بدون تغییر رفتار بیرونی قابل‌مشاهده**. اگر رفتار عوض شد، refactoring نیست.

### Code Smells (بوهای بد)

| Smell | نشانه | درمان رایج |
|---|---|---|
| Long Method | تابع چند ده‌خطی | Extract Method |
| Large Class / God Class | کلاس همه‌کاره | Extract Class |
| Duplicated Code | کپی‌پیست | Extract Method/Class |
| Long Parameter List | ۴+ آرگومان | Introduce Parameter Object |
| Feature Envy | متد بیشتر با دیتای کلاسِ دیگر کار می‌کند | Move Method |
| Primitive Obsession | همه‌چیز `str`/`int` | معرفی Value Object |
| Shotgun Surgery | یک تغییر، دست‌زدن به ده فایل | تجمیع مسئولیت |
| Data Clumps | چند فیلد که همیشه با هم می‌آیند | گروه‌بندی در یک شیء |

### مرحله‌های امنِ Refactoring

1. **تور ایمنی**: مطمئن شو تستِ پوشش‌دهنده‌ی رفتار فعلی وجود دارد (در نبودش، اول **Characterization Test** بنویس).
2. **قدم‌های ریز**: هر بار یک تبدیلِ کوچک و نام‌دار (Extract Method، Rename، Inline، Introduce Parameter Object...).
3. **سبز بمان**: بعد از هر قدم تست را اجرا کن. قرمز شد → فوراً برگرد.
4. **کامیت‌های کوچک**: هر refactor یک کامیتِ جدا، جدا از تغییرِ رفتار.

> نکته‌ی مصاحبه: refactor و feature را در یک کامیت/PR قاطی نکن. «Make the change easy, then make the easy change» (Kent Beck) — اول refactor تا تغییر آسان شود، بعد تغییر را بده.

---

## ۳. SOLID

> در قصه: قانون‌های بنّاییِ سالمِ عمو. هدفِ همه: کاهشِ هزینه‌ی تغییر.

### S — Single Responsibility Principle

یک کلاس فقط **یک دلیل برای تغییر** داشته باشد.

```python
# ❌ دو مسئولیت: محاسبه + ذخیره
class Invoice:
    def total(self): ...
    def save_to_db(self): ...      # دلیل دومِ تغییر

# ✅ جدا شده
class Invoice:
    def total(self): ...
class InvoiceRepository:
    def save(self, invoice): ...
```

### O — Open/Closed Principle

برای **توسعه باز**، برای **تغییر بسته**. قابلیت نو را با افزودنِ کد بده، نه با دست‌زدن به کدِ موجود.

```python
# ❌ هر نوعِ جدید = ویرایش این تابع
def area(shape):
    if shape.kind == "circle": ...
    elif shape.kind == "square": ...

# ✅ نوعِ جدید = کلاسِ جدید، بدون لمسِ کدِ قبلی
class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...
class Circle(Shape):
    def area(self): return math.pi * self.r ** 2
```

### L — Liskov Substitution Principle

هر زیرکلاس باید بتواند **جایگزینِ** کلاسِ پایه شود بدون شکستنِ رفتارِ مورد انتظار.

```python
# ❌ نقضِ Liskov — Square رفتارِ Rectangle را می‌شکند
class Rectangle:
    def set_width(self, w): self.w = w
    def set_height(self, h): self.h = h
class Square(Rectangle):
    def set_width(self, w): self.w = self.h = w   # کدی که با Rectangle کار می‌کرد، اینجا غافلگیر می‌شود
```

نشانه‌ی نقض: زیرکلاس یک متد را override کند و `NotImplementedError` بدهد، یا precondition را سخت‌تر / postcondition را ضعیف‌تر کند.

### I — Interface Segregation Principle

کلاینت نباید مجبور به وابستگی به متدهایی شود که استفاده نمی‌کند. اینترفیس‌های چاق را بشکن.

```python
# ❌ یک اینترفیس چاق
class Worker(ABC):
    def work(self): ...
    def eat(self): ...     # ربات که غذا نمی‌خورد!

# ✅ تفکیک‌شده
class Workable(ABC):
    def work(self): ...
class Eatable(ABC):
    def eat(self): ...
```

### D — Dependency Inversion Principle

ماژول‌های سطح‌بالا و سطح‌پایین هر دو به **انتزاع** وابسته باشند، نه به جزئیات.

```python
# ❌ سرویس مستقیم به کلاسِ بتنی وابسته است
class ReportService:
    def __init__(self): self.db = PostgresDB()   # قفل‌شده به Postgres

# ✅ وابسته به انتزاع، تزریق از بیرون (Dependency Injection)
class ReportService:
    def __init__(self, store: DataStore):        # DataStore یک ABC است
        self.store = store
```

> نکته‌ی مصاحبه: **DIP** اصل است؛ **Dependency Injection** یک تکنیک برای پیاده‌سازی آن. یکی‌شان نگیر.

---

## ۴. Design Patterns

> در قصه: دفترچه‌ی طرح‌های آماده‌ی عمو. سه دسته:

| دسته | هدف | الگوهای این سند |
|---|---|---|
| **Creational** | ساختِ شیء | Singleton، Factory Method / Abstract Factory |
| **Structural** | ترکیب و رابطِ اشیا | Decorator، Adapter |
| **Behavioral** | تعاملِ بینِ اشیا | Strategy، Observer |

### Singleton (Creational)

یک کلاس فقط **یک نمونه** و یک نقطه‌ی دسترسی سراسری دارد. کاربرد: config، connection pool، logger.

```python
class Config:
    _instance = None
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

> گاتچا: Singleton اغلب **ضدالگو** محسوب می‌شود — حالتِ سراسریِ پنهان، تستِ سخت، مشکل در thread/multiprocess. در پایتون اغلب یک ماژولِ سطح‌بالا یا تزریق وابستگی جایگزینِ بهتری است.

### Factory (Creational)

منطقِ ساختِ شیء را از مصرف‌کننده جدا می‌کند. **Factory Method**: متدی که زیرکلاس‌ها نوعِ خروجی را تعیین می‌کنند. **Abstract Factory**: کارخانه‌ای که خانواده‌ای از اشیای مرتبط می‌سازد.

```python
class DialogFactory:
    def create_button(self) -> Button: ...
class WindowsFactory(DialogFactory):
    def create_button(self): return WindowsButton()
```

### Decorator (Structural)

قابلیت را به‌صورت **پویا و لایه‌لایه** به یک شیء اضافه می‌کند، بدون تغییر کلاسِ اصلی. (دکوراتورهای پایتون نمونه‌ی زبانیِ همین ایده‌اند.)

```python
class Coffee:
    def cost(self): return 5
class MilkDecorator:
    def __init__(self, coffee): self.coffee = coffee
    def cost(self): return self.coffee.cost() + 2    # لایه روی لایه
# MilkDecorator(SugarDecorator(Coffee())).cost()
```

### Adapter (Structural)

رابطِ یک کلاس را به رابطِ مورد انتظارِ کلاینت **ترجمه** می‌کند تا دو چیزِ ناسازگار با هم کار کنند.

```python
class StripeAPI:               # کتابخانه‌ی بیرونی
    def make_payment(self, cents): ...
class PaymentAdapter:          # رابطی که کدِ ما انتظار دارد
    def __init__(self, stripe): self.stripe = stripe
    def pay(self, dollars):              # رابطِ یکپارچه‌ی داخلی
        self.stripe.make_payment(int(dollars * 100))
```

### Strategy (Behavioral)

خانواده‌ای از الگوریتم‌ها را قابلِ‌تعویض می‌کند؛ الگوریتم را در زمانِ اجرا انتخاب می‌کنی. جایگزینِ `if/elif`های بزرگ.

```python
class Sorter:
    def __init__(self, strategy): self.strategy = strategy
    def run(self, data): return self.strategy(data)
# Sorter(quick_sort).run(xs)  /  Sorter(merge_sort).run(xs)
```

### Observer (Behavioral)

وقتی حالتِ یک شیء (Subject) عوض شد، همه‌ی ناظرانِ مشترک‌شده (Observers) خودکار خبردار می‌شوند. پایه‌ی event/pub-sub و UI binding.

```python
class Subject:
    def __init__(self): self._observers = []
    def subscribe(self, obs): self._observers.append(obs)
    def notify(self, event):
        for obs in self._observers: obs.update(event)
```

> نکته‌ی مصاحبه: فرقِ **Strategy** و **State**: هر دو شیءِ رفتار را عوض می‌کنند، اما در Strategy کلاینت الگوریتم را انتخاب می‌کند؛ در State خودِ اشیا با تغییرِ وضعیت، رفتار را عوض می‌کنند. فرقِ **Observer** با **Pub/Sub**: در Observer معمولاً Subject مستقیماً Observers را می‌شناسد؛ در Pub/Sub یک broker وسط است (coupling کمتر).

---

## ۵. Architecture

### Monolith

کلِ اپلیکیشن در یک codebase و یک deployable unit. مزایا: سادگیِ توسعه/تست/استقرار، تراکنشِ ACID آسان، فراخوانیِ درون‌فرایندی ارزان. معایب در مقیاس: codebase کوپل‌شده، استقرارِ همه‌یاهیچ، scale فقط به‌صورت کلی. **Modular Monolith** (ماژول‌های با مرزِ روشن داخل یک monolith) اغلب نقطه‌ی شیرینِ آغاز است.

### Microservices

تجزیه به سرویس‌های کوچکِ مستقل، هر کدام با مالکیتِ داده‌ی خودش، که از طریق شبکه (HTTP/gRPC/پیام) حرف می‌زنند.

| محور | Monolith | Microservices |
|---|---|---|
| استقرار | یکپارچه | مستقل per-service |
| مقیاس | کلی | انتخابیِ per-service |
| داده | یک DB، تراکنش آسان | DB per service، تراکنشِ توزیع‌شده سخت |
| خطا | blast radius بزرگ | ایزوله ولی شکستِ شبکه‌ای جدید |
| عملیات | ساده | نیازمند CI/CD، observability، service discovery |
| تیم | یک تیم | تیم‌های مستقل (Conway's Law) |

تله‌ها: **shared database** (کوپلِ پنهان → distributed monolith)، تراکنشِ توزیع‌شده (راه‌حل: **Saga** + eventual consistency)، latency و شکستِ شبکه (نیاز به retry، timeout، **circuit breaker**)، پیچیدگیِ debug توزیع‌شده (نیاز به **distributed tracing**).

> نکته‌ی مصاحبه: «اول monolith، بعد در صورت نیاز جدا کن» توصیه‌ی رایج است. microservices مشکلِ **سازمانی و مقیاسی** را حل می‌کند، نه مشکلِ کد. اگر مرزهای دامنه را نمی‌شناسی، جدا کردنِ زودهنگام مرزها را اشتباه می‌برّی.

### Event-Driven Architecture (EDA)

اجزا از طریقِ **رویداد** ارتباط برقرار می‌کنند، نه فراخوانیِ مستقیم. **Producer** رویداد را منتشر می‌کند، **Broker** (Kafka, RabbitMQ, SQS) آن را می‌رساند، **Consumer**ها واکنش نشان می‌دهند.

- **Decoupling**: Producer نمی‌داند و اهمیت نمی‌دهد چه کسی مصرف می‌کند.
- الگوها: **Event Notification** (فقط خبر «چیزی شد»)، **Event-Carried State Transfer** (داده‌ی کامل در رویداد)، **Event Sourcing** (حالت = جمعِ همه‌ی رویدادها)، **CQRS** (مدلِ خواندن جدا از نوشتن).
- تضمین‌های تحویل: at-most-once / **at-least-once** (رایج → consumer باید **idempotent** باشد) / exactly-once (سخت).
- چالش‌ها: ترتیب رویدادها، پیام‌های تکراری، eventual consistency، debug سخت‌تر.

```
Producer ──event──▶ [ Broker / Topic ] ──▶ Consumer A (idempotent)
                                       └──▶ Consumer B
```

### Domain-Driven Design (DDD)

مدل‌سازی نرم‌افزار حولِ **دامنه‌ی کسب‌وکار**. تمرکز بر زبان و مرزها:

| مفهوم | تعریف |
|---|---|
| **Ubiquitous Language** | یک واژگانِ مشترکِ دقیق بینِ توسعه‌دهنده و خبره‌ی دامنه؛ همان کلمات در کد ظاهر می‌شوند. |
| **Bounded Context** | مرزی صریح که داخلش یک مدل و یک زبان معتبر است؛ همان واژه در دو context معنیِ متفاوت دارد. |
| **Entity** | شیئی که هویتش با **شناسه‌ی پایدار** تعریف می‌شود (نه مقادیرش). دو `User` با id متفاوت، متفاوت‌اند حتی اگر فیلدهایشان یکی باشد. |
| **Value Object** | با **مقدارش** تعریف می‌شود، بدون شناسه، تغییرناپذیر (مثل `Money`, `Address`). |
| **Aggregate** | خوشه‌ای از Entity/Value Object که یک واحدِ یکپارچگی است؛ دسترسی فقط از **Aggregate Root** و invariantها آنجا حفظ می‌شوند. |

> نکته‌ی مصاحبه: مرزهای **Bounded Context** اغلب نقشه‌ی خوبی برای مرزهای microservice‌اند. یک microservice = یک (یا چند) bounded context؛ نه تقسیم بر اساس لایه‌ی فنی.

---

## ۶. Testing

> در قصه: بازرسِ ساختمان و سه نوع بازرسی.

### انواع تست و هرم (Test Pyramid)

| نوع | محدوده | سرعت | تعداد | mock؟ |
|---|---|---|---|---|
| **Unit** | یک تابع/کلاس جدا | خیلی سریع | زیاد | وابستگی‌ها mock می‌شوند |
| **Integration** | چند جزء با هم (کد+DB) | متوسط | متوسط | وابستگیِ بیرونی واقعی یا fake |
| **E2E** | کلِ سیستم از دیدِ کاربر | کند | کم | تقریباً بدون mock |

```
   /\     E2E      (کم)
  /--\
 / In \   Integration (متوسط)
/------\
| Unit |  (زیاد، پایه)
```

ضدالگوها: **Ice-Cream Cone** (E2E زیاد، unit کم → کند و شکننده)، **Hourglass** (unit و E2E زیاد، integration خالی).

### Unit Testing با pytest

- ساختار **AAA**: Arrange (آماده‌سازی) → Act (اجرا) → Assert (بررسی).
- ویژگی‌های تستِ خوب — **FIRST**: Fast، Independent، Repeatable، Self-validating، Timely.
- هر تست یک رفتار؛ اسمِ تست رفتار را توضیح دهد: `test_withdraw_more_than_balance_raises`.

```python
def test_late_fee_charges_per_day():
    fee = calculate_late_fee(days_overdue=3, daily_rate=2)
    assert fee == 6
```

### Integration Testing

به‌جای mock کردنِ دیتابیس، یک DB واقعی (یا containerِ موقت مثل **testcontainers**، یا SQLite در حافظه) بالا می‌آوری و قراردادِ واقعیِ بینِ کد و وابستگی را می‌سنجی. کندتر اما باگ‌های اتصالی را می‌گیرد که unit نمی‌بیند.

### Mocking

جایگزینیِ یک وابستگی با بدلِ کنترل‌شده برای ایزوله‌کردنِ واحدِ تحتِ تست:

| واژه | معنی |
|---|---|
| **Stub** | جوابِ از-پیش-تعیین‌شده می‌دهد (بدونِ بررسیِ تماس) |
| **Mock** | جواب می‌دهد **و** بررسی می‌کند چطور صدا زده شد (assert on calls) |
| **Fake** | پیاده‌سازیِ سبکِ واقعی (مثل in-memory DB) |
| **Spy** | تماس‌ها را ثبت می‌کند، اصل را هم اجرا می‌کند |

```python
from unittest.mock import Mock

def test_sends_email_on_signup():
    mailer = Mock()
    register_user("a@b.com", mailer=mailer)
    mailer.send.assert_called_once_with("a@b.com")
```

> نکته‌ی مصاحبه: **mock نه** کن، نه چیزی که مالکش نیستی. mockِ بیش‌ازحد باعث تستِ شکننده می‌شود که فقط پیاده‌سازی را قفل می‌کند، نه رفتار را. وابستگیِ بیرونیِ خودت را پشت یک adapter ببر و آن adapter را mock کن.

### TDD — Red / Green / Refactor

```
🔴 RED      تستِ شکست‌خورده‌ای بنویس که رفتارِ خواسته‌شده را بیان کند
🟢 GREEN    کمترین کد را بنویس تا سبز شود (حتی اگر زشت)
🔵 REFACTOR با پوششِ تست، کد را تمیز کن؛ تست همچنان سبز
            ↺
```

مزایا: طراحیِ بهتر (کد testable می‌شود)، رگرسیونِ کمتر، مستندِ زنده. هزینه: انضباط و زمانِ اولیه. نسخه‌ی سطح‌بالاترش **BDD** است (سناریوهای Given/When/Then با زبانِ کسب‌وکار).

---

## 🔬 آزمایشگاه

### تمرین ۱ — refactor قدم‌به‌قدمِ یک تابعِ بدبو

تابعِ زیر چند بوی بد دارد: نامِ مبهم، تابعِ طولانیِ چندمسئولیتی، اعداد جادویی، تو در توییِ عمیق، کدِ تکراری.

```python
# نسخه‌ی اولیه (بدبو)
def p(o):
    t = 0
    for i in o:
        if i["type"] == "book":
            t += i["price"] * i["qty"]
        elif i["type"] == "food":
            t += i["price"] * i["qty"] * 0.9   # ۱۰٪ تخفیف غذا
    if t > 100:
        t = t * 0.95                            # ۵٪ تخفیف سفارش بزرگ
    return t
```

**قدم ۰ — تور ایمنی (تست characterization):**

```python
def test_total_baseline():
    order = [{"type": "book", "price": 50, "qty": 2},
             {"type": "food", "price": 20, "qty": 5}]
    assert p(order) == 180.5   # رفتار فعلی را قفل می‌کنیم
```

**قدم ۱ — Rename**: اسم‌های گویا (`p`→`calculate_order_total`, `o`→`items`, `t`→`total`).

**قدم ۲ — حذف اعداد جادویی**: ثابت‌های بانام.

```python
FOOD_DISCOUNT = 0.10
BULK_THRESHOLD = 100
BULK_DISCOUNT = 0.05
```

**قدم ۳ — Extract Method** برای قیمتِ هر قلم (حذفِ تکرارِ `price * qty`):

```python
def line_total(item):
    subtotal = item["price"] * item["qty"]
    if item["type"] == "food":
        subtotal *= (1 - FOOD_DISCOUNT)
    return subtotal
```

**قدم ۴ — تابعِ نهاییِ تمیز** (هر قدم تست را دوباره سبز نگه داشتیم):

```python
def calculate_order_total(items):
    total = sum(line_total(item) for item in items)
    if total > BULK_THRESHOLD:
        total *= (1 - BULK_DISCOUNT)
    return total
# test_total_baseline هنوز سبز است → رفتار حفظ شد ✅
```

### تمرین ۲ — یک unit test کامل با pytest

```python
# bank.py
class InsufficientFunds(Exception): ...

class Account:
    def __init__(self, balance=0):
        self.balance = balance
    def withdraw(self, amount):
        if amount <= 0:
            raise ValueError("amount must be positive")
        if amount > self.balance:
            raise InsufficientFunds()
        self.balance -= amount
        return self.balance

# test_bank.py
import pytest
from bank import Account, InsufficientFunds

def test_withdraw_reduces_balance():      # Arrange-Act-Assert
    acc = Account(balance=100)
    acc.withdraw(30)
    assert acc.balance == 70

def test_withdraw_too_much_raises():
    acc = Account(balance=50)
    with pytest.raises(InsufficientFunds):
        acc.withdraw(80)
    assert acc.balance == 50              # حالت دست‌نخورده ماند

@pytest.mark.parametrize("bad", [0, -5])
def test_withdraw_nonpositive_rejected(bad):
    with pytest.raises(ValueError):
        Account(100).withdraw(bad)
```

اجرا: `pytest -q test_bank.py`

### تمرین ۳ — یک چرخه‌ی کاملِ TDD (FizzBuzz)

```python
# 🔴 RED — اول تست، تابع هنوز نیست
def test_fizzbuzz():
    assert fizzbuzz(1) == "1"
    assert fizzbuzz(3) == "Fizz"
    assert fizzbuzz(5) == "Buzz"
    assert fizzbuzz(15) == "FizzBuzz"
# اجرا → NameError: fizzbuzz تعریف نشده (قرمز)

# 🟢 GREEN — کمترین کدی که سبز کند
def fizzbuzz(n):
    if n % 15 == 0: return "FizzBuzz"
    if n % 3 == 0:  return "Fizz"
    if n % 5 == 0:  return "Buzz"
    return str(n)
# اجرا → سبز ✅

# 🔵 REFACTOR — تمیزتر، تست هنوز سبز
def fizzbuzz(n):
    out = ("Fizz" if n % 3 == 0 else "") + ("Buzz" if n % 5 == 0 else "")
    return out or str(n)
```

---

## ✅ چک‌لیست تسلط

اگر بتوانی به این‌ها جواب بدهی، این فصل را بلدی:

- [ ] سه قانونِ نام‌گذاریِ خوب، و فرقِ کامنتِ «چرا» با «چه»؟
- [ ] چرا flag argument و long parameter list بوی بد هستند؟
- [ ] تعریفِ دقیقِ refactoring؟ چرا تست **پیش‌نیازِ** آن است؟
- [ ] پنج تا code smell نام ببر و درمانِ هرکدام؟
- [ ] هر پنج حرفِ SOLID را با یک مثالِ یک‌خطی توضیح بده.
- [ ] فرقِ DIP و Dependency Injection؟
- [ ] هر شش الگو را در دسته‌ی درست (creational/structural/behavioral) بگذار.
- [ ] فرقِ Strategy با State؟ Observer با Pub/Sub؟ Adapter با Decorator؟
- [ ] چرا Singleton اغلب ضدالگوست؟
- [ ] چهار trade-offِ کلیدیِ monolith در برابر microservices؟
- [ ] چرا «دیتابیس مشترک بین microserviceها» مشکل‌ساز است؟
- [ ] Producer/Broker/Consumer چیست و decoupling در EDA یعنی چه؟ چرا consumer باید idempotent باشد؟
- [ ] Bounded Context، Ubiquitous Language، Entity، Aggregate را تعریف کن.
- [ ] فرقِ Entity و Value Object؟
- [ ] هرمِ تست را بکش؛ Ice-Cream Cone چه ضدالگویی است؟
- [ ] فرقِ Stub، Mock، Fake، Spy؟
- [ ] چرخه‌ی Red-Green-Refactor را قدم‌به‌قدم اجرا کن.

---

## 🎯 سوالات مصاحبه

### Clean Code & Refactoring

<details><summary>**فرقِ کامنتِ خوب و بد چیست؟**</summary>

کامنتِ خوب «چرا» را توضیح می‌دهد (تصمیم، عواقب، قانونِ کسب‌وکار، هشدار) و چیزی را می‌گوید که کد نمی‌تواند. کامنتِ بد «چه» را تکرار می‌کند (کاری که خودِ کد روشن انجام می‌دهد)، یا کهنه و دروغ شده، یا کدِ کامنت‌شده است. قاعده: اول با بهتر کردنِ نام‌ها کامنت را حذف کن؛ کامنت عذرخواهی بابتِ کدِ نارساست.

</details>

<details><summary>**یک تابع را چه چیز «بیش‌ازحد بزرگ» می‌کند و چطور کوچکش می‌کنی؟**</summary>

نشانه‌ها: چند سطحِ انتزاع در هم، نیاز به اسکرول، بیش از یک «دلیلِ تغییر»، نیاز به کلمه‌ی «و» برای توصیفِ کارش. درمان: Extract Method برای بلوک‌های منطقی، Guard Clause برای کاهشِ تو در تویی، Introduce Parameter Object برای آرگومان‌های زیاد، و جداکردنِ مسئولیت‌ها (SRP).

</details>

<details><summary>**تعریفِ دقیقِ refactoring چیست و چرا تست پیش‌نیازِ آن است؟**</summary>

refactoring یعنی تغییرِ ساختارِ داخلیِ کد بدونِ تغییرِ رفتارِ بیرونیِ قابل‌مشاهده. چون رفتار نباید عوض شود، به یک تور ایمنی نیاز داری که ثابت کند رفتار حفظ شده: مجموعه‌ی تست. بدونِ تست، نمی‌توانی مطمئن باشی که «فقط ساختار» را عوض کرده‌ای و ناخواسته رفتار را نشکسته‌ای.

</details>

<details><summary>**پنج code smell نام ببر و درمانِ هرکدام را بگو.**</summary>

Long Method → Extract Method. Duplicated Code → استخراج و اشتراک‌گذاری. Long Parameter List → Introduce Parameter Object. God Class → Extract Class بر اساس مسئولیت. Feature Envy → Move Method به کلاسی که داده‌اش را بیشتر استفاده می‌کند. (Primitive Obsession → Value Object، Shotgun Surgery → تجمیعِ مسئولیتِ پراکنده.)

</details>

<details><summary>**سناریو: یک PR هم refactor بزرگ دارد، هم یک فیچر جدید. چه می‌گویی؟**</summary>

درخواستِ تفکیک می‌کنم. قاطی‌کردنِ refactor و تغییرِ رفتار، review را تقریباً غیرممکن می‌کند (نمی‌توان فهمید کدام diff رفتار را عوض کرده) و revertِ امن را از بین می‌برد. قاعده‌ی Kent Beck: اول refactor (در یک کامیت/PR) تا تغییر آسان شود، بعد فیچر را در کامیتِ جدا بده.

</details>

### SOLID

<details><summary>**Single Responsibility را با مثال توضیح بده.**</summary>

یک کلاس فقط یک «دلیلِ تغییر» داشته باشد. مثلاً کلاسِ `Invoice` که هم total را حساب می‌کند هم خودش را در DB ذخیره می‌کند، دو دلیلِ تغییر دارد (منطقِ محاسبه، و فناوریِ ذخیره). جدا می‌کنیم: `Invoice` برای محاسبه و `InvoiceRepository` برای ذخیره.

</details>

<details><summary>**Open/Closed یعنی چه؟ چطور یک switch بزرگ را OCP-friendly می‌کنی؟**</summary>

باز برای توسعه، بسته برای تغییر: قابلیتِ نو را با افزودنِ کدِ جدید بده، نه با ویرایشِ کدِ موجود. یک `if/elif` روی نوع، با هر نوعِ جدید باید ویرایش شود (نقضِ OCP). راه‌حل: polymorphism — یک کلاسِ پایه/اینترفیس، و هر نوعِ جدید یک زیرکلاسِ جدید بدونِ لمسِ کدِ قبلی (مثلِ Strategy یا Factory).

</details>

<details><summary>**یک نقضِ کلاسیکِ Liskov مثال بزن.**</summary>

مسئله‌ی Rectangle/Square: `Square` از `Rectangle` ارث می‌برد و `set_width` را طوری override می‌کند که height هم عوض شود. کدی که با `Rectangle` کار می‌کرد و انتظار داشت width و height مستقل باشند، با `Square` غافلگیر و خراب می‌شود — پس Square نمی‌تواند جایگزینِ امنِ Rectangle باشد.

</details>

<details><summary>**فرقِ Dependency Inversion و Dependency Injection چیست؟**</summary>

DIP یک اصلِ طراحی است: ماژولِ سطح‌بالا به انتزاع (interface) وابسته باشد نه به جزئیاتِ بتنی. DI یک تکنیکِ پیاده‌سازی است: وابستگی را از بیرون به شیء «تزریق» می‌کنی (سازنده، setter، یا framework) به‌جای new کردن داخل. DI یکی از راه‌های محققِ DIP است.

</details>

<details><summary>**Interface Segregation چه مشکلی را حل می‌کند؟**</summary>

اینترفیس‌های چاق کلاینت را مجبور می‌کنند به متدهایی وابسته شود که استفاده نمی‌کند؛ تغییر در یکی، همه‌ی پیاده‌سازی‌ها را تحتِ‌تأثیر می‌گذارد (مثل `Robot` که مجبور به پیاده‌سازیِ `eat()` شود). ISP می‌گوید اینترفیس‌های کوچک و نقش‌محور بساز تا هر کلاینت فقط به آنچه نیاز دارد وابسته باشد.

</details>

### Design Patterns

<details><summary>**شش الگوی Singleton, Factory, Decorator, Adapter, Strategy, Observer را دسته‌بندی کن.**</summary>

Creational (ساخت شیء): Singleton، Factory. Structural (ترکیب/رابط): Decorator، Adapter. Behavioral (تعامل): Strategy، Observer.

</details>

<details><summary>**فرقِ Strategy و State چیست؟**</summary>

هر دو رفتار را به شیئی بیرون می‌سپارند و قابلِ‌تعویض می‌کنند. در Strategy، کلاینت الگوریتم را صریحاً انتخاب می‌کند و الگوریتم‌ها معمولاً از هم بی‌خبرند. در State، خودِ context با تغییرِ وضعیتِ داخلی، state و در نتیجه رفتار را عوض می‌کند و stateها اغلب گذارِ به یکدیگر را می‌دانند.

</details>

<details><summary>**فرقِ Adapter و Decorator؟**</summary>

Adapter رابطِ یک شیء را **عوض** می‌کند تا با چیزی سازگار شود (هدف: سازگاری). Decorator رابط را **حفظ** می‌کند ولی رفتار را **اضافه** می‌کند (هدف: گسترشِ قابلیت، اغلب لایه‌لایه). یعنی Decorator همان interface را برمی‌گرداند، Adapter یک interface متفاوت.

</details>

<details><summary>**چرا Singleton اغلب ضدالگو خوانده می‌شود؟**</summary>

حالتِ سراسریِ پنهان می‌سازد که coupling و وابستگی‌های نامرئی ایجاد می‌کند، تست را سخت می‌کند (state بینِ تست‌ها نشت می‌کند، نمی‌توان mock کرد)، و در محیطِ چندنخی/چندفرایندی مشکل‌ساز است. جایگزین: تزریقِ وابستگی یا یک ماژولِ سطح‌بالا.

</details>

<details><summary>**Observer در دنیای واقعی کجا استفاده می‌شود؟**</summary>

سیستم‌های event/pub-sub، data binding در UI (تغییرِ مدل → به‌روزرسانیِ view)، listenerهای رویداد در GUI، سیستم‌های نوتیفیکیشن، و reactive programming (streams). EDA در سطحِ معماری، تعمیمِ همین ایده با یک broker وسط است.

</details>

<details><summary>**سناریو: یک تابع پر از if/elif روی «نوع پرداخت» داری که مدام بزرگ‌تر می‌شود. کدام الگو؟**</summary>

Strategy: هر روشِ پرداخت یک کلاس با رابطِ مشترک (`pay`)، و در زمانِ اجرا strategy مناسب تزریق می‌شود. افزودنِ روشِ نو فقط یک کلاسِ جدید است (OCP)، بدونِ لمسِ کدِ موجود. برای ساختِ خودِ شیءِ strategy هم می‌توانی Factory بگذاری.

</details>

### Architecture — Monolith / Microservices / EDA

<details><summary>**چهار trade-offِ اصلیِ monolith در برابر microservices؟**</summary>

۱) سادگیِ توسعه/استقرار (monolith برنده) در برابر استقرار و مقیاسِ مستقل (microservices برنده). ۲) تراکنشِ ACID آسان (monolith) در برابر داده‌ی توزیع‌شده و eventual consistency (microservices). ۳) blast radiusِ خرابی بزرگ (monolith) در برابر ایزولاسیونِ خطا اما شکستِ شبکه‌ای جدید (microservices). ۴) ارتباطِ درون‌فرایندیِ ارزان (monolith) در برابر latency و پیچیدگیِ عملیاتیِ شبکه (microservices).

</details>

<details><summary>**سناریو: microserviceهای تیمِ شما یک دیتابیسِ مشترک دارند. چه مشکلاتی پیش می‌آید؟**</summary>

این یک distributed monolith می‌سازد: سرویس‌ها از طریقِ schema به‌هم کوپل می‌شوند، پس تغییرِ schema یک سرویس بقیه را می‌شکند و دیگر مستقل deploy نمی‌شوند. مالکیتِ داده مبهم می‌شود، contention و قفل روی DB مشترک، و scaleِ مستقل غیرممکن. درست: هر سرویس دیتابیسِ خودش؛ تبادلِ داده از طریقِ API یا event، نه خواندن مستقیمِ جدولِ هم‌دیگر.

</details>

<details><summary>**تراکنش روی چند microservice چطور انجام می‌شود؟**</summary>

تراکنشِ ACID توزیع‌شده عملاً اجتناب می‌شود (2PC شکننده و کند است). به‌جایش الگوی **Saga**: زنجیره‌ای از تراکنش‌های محلی که با رویداد به‌هم وصل‌اند، و در صورتِ شکست، **compensating transaction**‌ها اثر را برمی‌گردانند. نتیجه eventual consistency است، نه اتمیکِ آنی.

</details>

<details><summary>**EDA چیست و چرا decoupling مهم است؟ چرا consumer باید idempotent باشد؟**</summary>

EDA یعنی اجزا با انتشار و مصرفِ رویداد ارتباط می‌گیرند، نه فراخوانیِ مستقیم. Producer نمی‌داند چه کسی مصرف می‌کند (decoupling)، پس افزودن/حذفِ consumer بدونِ لمسِ producer ممکن است و سیستم منعطف‌تر می‌شود. چون اکثرِ brokerها تحویلِ at-least-once می‌دهند، یک پیام ممکن است تکراری برسد؛ اگر consumer idempotent باشد، پردازشِ دوباره اثرِ اضافه ندارد.

</details>

<details><summary>**Event Sourcing و CQRS را کوتاه توضیح بده.**</summary>

Event Sourcing: به‌جای ذخیره‌ی حالتِ فعلی، توالیِ همه‌ی رویدادها را ذخیره می‌کنی و حالت را با پخشِ آن‌ها بازمی‌سازی (audit و time-travel رایگان، اما بازسازی و schema evolution سخت). CQRS: مدلِ نوشتن (command) را از مدلِ خواندن (query) جدا می‌کنی تا هرکدام جداگانه بهینه/scale شوند؛ اغلب با Event Sourcing همراه می‌شود اما الزامی نیست.

</details>

<details><summary>**سناریو: کِی monolith و کِی microservices را انتخاب می‌کنی؟**</summary>

پیش‌فرض monolith (یا modular monolith): تیمِ کوچک، دامنه‌ی هنوز ناشناخته، نیاز به سرعتِ اولیه. microservices وقتی: تیم‌های متعدد که می‌خواهند مستقل deploy کنند، نیازِ scaleِ بسیار متفاوت بینِ بخش‌ها، و مرزهای دامنه روشن شده‌اند. microservices مشکلِ سازمانی/مقیاسی را حل می‌کند نه مشکلِ کد؛ زودهنگام رفتن، پیچیدگیِ توزیع‌شده را بی‌جهت تحمیل می‌کند.

</details>

### Domain-Driven Design

<details><summary>**Ubiquitous Language چیست و چرا مهم است؟**</summary>

زبانِ مشترک و دقیقی که توسعه‌دهنده و خبره‌ی دامنه به‌کار می‌برند و عیناً در کد (نام کلاس‌ها، متدها) ظاهر می‌شود. مهم است چون شکافِ ترجمه بینِ کسب‌وکار و کد را می‌بندد و سوءتفاهم را کم می‌کند؛ وقتی کد همان واژگانِ دامنه را دارد، گفتگو درباره‌ی رفتار دقیق و بی‌ابهام می‌شود.

</details>

<details><summary>**Bounded Context را توضیح بده — چرا یک «User» سراسری ایده‌ی بدی است؟**</summary>

Bounded Context مرزی صریح است که داخلش یک مدل و یک زبان معتبر است. یک مدلِ غول‌پیکرِ `User` که برای همه‌ی سیستم استفاده شود، فیلدها و قوانینِ متناقضِ همه‌ی context‌ها (احراز هویت، صورتحساب، پشتیبانی) را در خود جمع می‌کند و به یک God Object کوپل می‌شود. بهتر است هر context مدلِ خودش را داشته باشد: `Account` در auth، `Customer` در billing — هرکدام فقط آنچه نیاز دارد.

</details>

<details><summary>**فرقِ Entity و Value Object؟**</summary>

Entity با **شناسه‌ی پایدارش** هویت دارد: دو Entity با id متفاوت، متفاوت‌اند حتی با فیلدهای یکسان، و در طول زمان تغییر می‌کند (مثل `Customer`). Value Object با **مقدارش** تعریف می‌شود، شناسه ندارد و تغییرناپذیر است: دو `Money(10, "USD")` کاملاً یکی‌اند (مثل `Address`, `Money`, `DateRange`).

</details>

<details><summary>**Aggregate و Aggregate Root چیست؟**</summary>

Aggregate خوشه‌ای از Entity و Value Objectهاست که یک واحدِ یکپارچگیِ تراکنشی تشکیل می‌دهند (مثلِ `Order` با `OrderLine`‌هایش). Aggregate Root تنها نقطه‌ی ورودِ بیرونی به این خوشه است؛ همه‌ی تغییرات از طریقِ آن انجام می‌شود تا invariantها (مثلِ «جمعِ خطوط = total») همیشه حفظ بمانند. کدِ بیرونی نباید مستقیم به اجزای داخلی دست بزند.

</details>

<details><summary>**رابطه‌ی Bounded Context و مرزِ microservice چیست؟**</summary>

Bounded Context نقشه‌ی طبیعیِ مرزبندیِ microservice‌هاست: یک سرویس حولِ یک (یا چند) bounded context شکل می‌گیرد، با مدل و دیتابیسِ خودش. تقسیم بر اساسِ لایه‌ی فنی (همه‌ی DAOها یک سرویس، همه‌ی controllerها یکی) ضدالگوست و chatty coupling می‌سازد.

</details>

### Testing

<details><summary>**هرمِ تست را توضیح بده و چرا این شکل است.**</summary>

از پایین: unitهای زیاد (سریع، ارزان، ایزوله)، integrationِ متوسط، و E2E‌های کم (کند، شکننده، گران). شکلِ هرمی چون می‌خواهی بیشترِ پوشش را از تست‌های سریع و قطعی بگیری و فقط چند مسیرِ بحرانی را E2E بزنی. وارونه‌اش (Ice-Cream Cone: E2E زیاد، unit کم) باعثِ suiteِ کند و شکننده و دیباگِ سخت می‌شود.

</details>

<details><summary>**فرقِ unit و integration test؟**</summary>

unit یک واحدِ منفرد (تابع/کلاس) را جدا از وابستگی‌هایش (که mock می‌شوند) می‌سنجد — خیلی سریع و قطعی. integration تعاملِ چند جزءِ واقعی با هم را می‌سنجد (مثلِ کد + دیتابیسِ واقعی) و باگ‌های مرزی/اتصالی را می‌گیرد که unit با mock پنهان می‌کند. integration کندتر است.

</details>

<details><summary>**فرقِ Stub، Mock، Fake و Spy؟**</summary>

Stub جوابِ از-پیش-آماده می‌دهد (state-based، بدونِ بررسیِ تماس). Mock علاوه بر جواب، انتظارات درباره‌ی نحوه‌ی صدا زدن را verify می‌کند (interaction-based). Fake یک پیاده‌سازیِ سبکِ کارآمد است (مثلِ in-memory DB). Spy تماس‌ها را ثبت می‌کند ضمن اینکه پیاده‌سازیِ واقعی را هم اجرا می‌کند.

</details>

<details><summary>**چرخه‌ی TDD را توضیح بده. چه مزیتی دارد؟**</summary>

Red: یک تستِ شکست‌خورده بنویس که رفتارِ خواسته‌شده را بیان کند. Green: کمترین کد را بنویس تا سبز شود. Refactor: با پوششِ تست، کد را تمیز کن و سبز نگه‌دار؛ تکرار. مزایا: طراحیِ testable و کوپلِ کم، مستندِ زنده، رگرسیونِ کم، و تمرکز روی رفتار به‌جای پیاده‌سازی. هزینه: انضباط و سربارِ اولیه.

</details>

<details><summary>**سناریو: کدت یک API بیرونی (مثلاً پرداخت) صدا می‌زند. چطور تستش می‌کنی؟**</summary>

API را پشتِ یک adapter/interface ببر (مثلِ `PaymentGateway`). در unit test، یک mock/fake از آن adapter تزریق کن تا منطقِ خودت را بدونِ شبکه و قطعی بسنجی. جداگانه یک تستِ integrationِ کوچک (یا contract test) روی خودِ adapter بزن تا مطمئن شوی واقعاً با API درست حرف می‌زند. اصل: «mockِ آنچه مالکش نیستی» را نکن — پشتش adapter بگذار.

</details>

<details><summary>**over-mocking چه مشکلی می‌سازد؟**</summary>

تستِ شکننده‌ای می‌سازد که به جزئیاتِ پیاده‌سازی قفل می‌شود نه رفتار: هر refactorِ بی‌ضرر تست‌ها را قرمز می‌کند، و تست‌ها ممکن است سبز بمانند درحالی‌که سیستم واقعاً کار نمی‌کند (چون همه‌چیز جعلی است). راه‌حل: فقط مرزهای واقعی (I/O، شبکه، زمان) را mock کن و بیشتر روی رفتارِ قابل‌مشاهده تست بنویس.

</details>

<details><summary>**سناریو: یک باگ در production پیدا شد. قدمِ اولِ تست‌محورت چیست؟**</summary>

اول یک تستِ شکست‌خورده می‌نویسم که دقیقاً همان باگ را بازتولید می‌کند (red). این هم باگ را تثبیت می‌کند هم رگرسیونِ آینده را می‌گیرد. بعد کد را اصلاح می‌کنم تا سبز شود، و در پایان در صورتِ نیاز refactor می‌کنم. این‌طور مطمئنم باگ واقعاً رفع شده و دیگر برنمی‌گردد.

</details>

<details><summary>**ویژگی‌های یک تستِ خوب (FIRST) چیست؟**</summary>

Fast (سریع، تا مدام اجرا شود)، Independent (مستقل، بدونِ وابستگی به ترتیب یا تستِ دیگر)، Repeatable (در هر محیط نتیجه‌ی یکسان، بدونِ flakiness)، Self-validating (خودش pass/fail را تعیین کند، نه بازرسیِ دستی)، Timely (به‌موقع، ترجیحاً قبل/همزمانِ کد).

</details>
