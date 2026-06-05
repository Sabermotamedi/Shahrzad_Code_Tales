# سطح ۳ — توضیح فنی: Networking & Internet ⚙️

> دیگر قصه‌ای در کار نیست. این سند مرجع فنی است — همان چیزهایی که در مصاحبه و کار واقعی لازم داری. هر بخش با معادل قصه‌ای‌اش شروع می‌شود تا ذهنت وصل بماند.

---

## ۱. HTTP/HTTPS

### چرخه‌ی حیات یک درخواست (Request Lifecycle)

از لحظه‌ی Enter زدن تا دیدن صفحه:

1. **URL Parsing** — مرورگر URL را تجزیه می‌کند: scheme + host + port + path + query
2. **DNS Resolution** — تبدیل hostname به IP (جزئیات در بخش ۳)
3. **TCP Connection** — اتصال three-way handshake به پورت 443
4. **TLS Handshake** — توافق روی کلید رمزنگاری (در HTTPS)
5. **Request ارسال می‌شود** — متد + مسیر + هدرها + بدنه
6. **سرور پردازش می‌کند** — روتینگ، منطق، دیتابیس
7. **Response برمی‌گردد** — status code + هدرها + بدنه
8. **مرورگر رندر می‌کند** — HTML → DOM، و درخواست‌های بعدی برای CSS/JS/تصاویر (موازی)

### متدها (Methods)

| متد | کاربرد | Idempotent? | Safe? | بدنه دارد؟ |
|---|---|---|---|---|
| `GET` | خواندن داده | ✅ | ✅ | ❌ |
| `POST` | ساختن منبع جدید / عملیات | ❌ | ❌ | ✅ |
| `PUT` | جایگزینی کامل منبع | ✅ | ❌ | ✅ |
| `PATCH` | تغییر جزئی منبع | ❌* | ❌ | ✅ |
| `DELETE` | حذف منبع | ✅ | ❌ | معمولاً ❌ |
| `HEAD` | مثل GET ولی فقط هدرها | ✅ | ✅ | ❌ |
| `OPTIONS` | قابلیت‌های سرور (مهم در CORS) | ✅ | ✅ | ❌ |

- **Safe** = حالت سرور را تغییر نمی‌دهد
- **Idempotent** = تکرارش نتیجه را عوض نمی‌کند (یک بار DELETE یا ده بار، نتیجه یکی است). به همین دلیل retry روی PUT/DELETE امن است ولی روی POST نه.

### کدهای وضعیت (Status Codes)

| دسته | معنی | مهم‌ترین‌ها |
|---|---|---|
| `1xx` | اطلاع‌رسانی | `101 Switching Protocols` (ارتقا به WebSocket) |
| `2xx` | موفقیت | `200 OK`، `201 Created`، `204 No Content` |
| `3xx` | تغییر مسیر | `301` دائمی، `302` موقت، `304 Not Modified` (کش معتبر است) |
| `4xx` | خطای کلاینت | `400 Bad Request`، `401 Unauthorized` (احراز هویت نشده!)، `403 Forbidden` (شناختمت ولی اجازه نداری)، `404`، `429 Too Many Requests` |
| `5xx` | خطای سرور | `500 Internal Server Error`، `502 Bad Gateway` (پروکسی جواب بد گرفت)، `503 Service Unavailable`، `504 Gateway Timeout` |

> نکته‌ی مصاحبه: فرق `401` و `403` — اولی یعنی «کی هستی؟ معلوم نیست»، دومی یعنی «می‌دانم کی هستی، ولی حق نداری».

### هدرهای مهم (Headers)

```http
# درخواست
Host: example.com              # کدام سایت روی این سرور (virtual hosting)
Authorization: Bearer eyJ...   # توکن احراز هویت
Content-Type: application/json # نوع بدنه‌ی ارسالی
Accept: application/json       # چه نوعی را می‌پذیرم
User-Agent: Mozilla/5.0 ...    # مشخصات کلاینت

# پاسخ
Content-Type: text/html; charset=utf-8
Cache-Control: max-age=3600    # چقدر کش کن
ETag: "abc123"                 # اثرانگشت محتوا برای کش شرطی
Set-Cookie: session=xyz; HttpOnly; Secure; SameSite=Lax
Access-Control-Allow-Origin: * # CORS
```

### Cookie و Session

- **Cookie**: داده‌ی کوچک که سرور با `Set-Cookie` می‌فرستد و مرورگر در هر درخواستِ بعدی به همان دامنه **خودکار** برمی‌گرداند.
- فلگ‌های امنیتی حیاتی:
  - `HttpOnly` — جاوااسکریپت نمی‌تواند بخواند (دفاع در برابر XSS)
  - `Secure` — فقط روی HTTPS فرستاده می‌شود
  - `SameSite=Lax/Strict` — دفاع در برابر CSRF
- **Session**: حالتِ سمت سرور. کوکی فقط **شناسه‌ی** سشن را حمل می‌کند (`session=xyz`)؛ داده‌ی واقعی (کاربر کیست، سبد خریدش چیست) در سرور است — در حافظه، دیتابیس یا Redis.
- HTTP ذاتاً **stateless** است؛ کوکی + سشن این محدودیت را دور می‌زنند.
- جایگزین مدرن: **JWT** — حالت را خودِ توکن حمل می‌کند (سمت کلاینت)، سرور فقط امضا را چک می‌کند.

### نسخه‌های HTTP

| نسخه | ویژگی کلیدی |
|---|---|
| HTTP/1.1 | یک درخواست در هر لحظه روی هر اتصال (head-of-line blocking)؛ keep-alive |
| HTTP/2 | Multiplexing — چندین درخواست موازی روی یک اتصال TCP؛ فشرده‌سازی هدر (HPACK) |
| HTTP/3 | روی **QUIC** (مبتنی بر UDP!) — حذف HOL blocking در لایه‌ی انتقال، handshake سریع‌تر |

---

## ۲. TCP/IP

### مدل لایه‌ای

```
لایه‌ی کاربرد   (Application)  → HTTP, DNS, SMTP, SSH
لایه‌ی انتقال   (Transport)    → TCP, UDP          ← پورت‌ها اینجا هستند
لایه‌ی اینترنت  (Internet)     → IP, ICMP          ← آدرس IP اینجاست
لایه‌ی لینک     (Link)         → Ethernet, Wi-Fi   ← MAC Address اینجاست
```

هر لایه داده‌ی لایه‌ی بالایی را در پاکت خودش می‌گذارد (Encapsulation).

### آدرس‌دهی IP

- **IPv4**: ۳۲ بیت → `192.168.1.10` — حدود ۴.۳ میلیارد آدرس (تمام شده!)
- **IPv6**: ۱۲۸ بیت → `2001:db8::8a2e:370:7334`
- **محدوده‌های خصوصی (Private)**: `10.0.0.0/8`، `172.16.0.0/12`، `192.168.0.0/16` — در اینترنت route نمی‌شوند
- **NAT**: روتر خانگی، آدرس‌های خصوصی داخل خانه را پشت یک IP عمومی پنهان می‌کند
- **CIDR**: `/24` یعنی ۲۴ بیت اول شبکه است → `192.168.1.0/24` = ۲۵۶ آدرس
- `127.0.0.1` = localhost، `0.0.0.0` = «همه‌ی اینترفیس‌ها» (در bind کردن سرور)

### پورت‌ها

- ۱۶ بیتی: ۰ تا ۶۵۵۳۵. پورت‌های زیر ۱۰۲۴ نیاز به دسترسی root دارند.
- یک اتصال با **۴ مؤلفه** یکتا می‌شود: `(src IP, src port, dst IP, dst port)`

| پورت | سرویس | | پورت | سرویس |
|---|---|---|---|---|
| 22 | SSH | | 443 | HTTPS |
| 53 | DNS | | 3306 | MySQL |
| 80 | HTTP | | 5432 | PostgreSQL |
| 25/587 | SMTP | | 6379 | Redis |

### TCP Handshake و چرخه‌ی اتصال

```
باز کردن (3-way):                 بستن (4-way):
Client → SYN     (seq=x)          A → FIN
Client ← SYN-ACK (seq=y, ack=x+1) A ← ACK
Client → ACK     (ack=y+1)        A ← FIN
                                  A → ACK  → TIME_WAIT (2×MSL)
```

تضمین‌های TCP و مکانیزم‌هایشان:

| تضمین | مکانیزم |
|---|---|
| ترتیب | Sequence numbers |
| تحویل | ACK + Retransmission (timeout یا 3 duplicate ACKs) |
| سالم بودن | Checksum |
| غرق نکردن گیرنده | Flow control (receive window) |
| غرق نکردن شبکه | Congestion control (slow start, AIMD, cubic) |

### TCP در برابر UDP

| | TCP | UDP |
|---|---|---|
| اتصال | Connection-oriented | Connectionless |
| تضمین تحویل و ترتیب | ✅ | ❌ |
| سربار | بیشتر (handshake، ACK، state) | حداقلی (هدر ۸ بایتی) |
| کاربرد | وب، ایمیل، انتقال فایل | DNS، استریم، VoIP، بازی، QUIC |

دستورهای کاربردی:

```bash
ss -tlnp                  # چه پورت‌هایی listen هستند (پروسه + PID)
ping example.com          # دسترسی‌پذیری در سطح ICMP
traceroute example.com    # مسیر hop به hop
tcpdump -i any port 443   # شنود پکت‌ها
nc -zv host 5432          # تست باز بودن یک پورت
```

---

## ۳. DNS

### فرایند Resolution (کامل)

```
مرورگر ← کش مرورگر ← کش OS ← /etc/hosts
   │ (پیدا نشد)
   ▼
Recursive Resolver (مثلاً 8.8.8.8 یا 1.1.1.1)
   │ (در کشش نبود — پرس‌وجوی تکرارشونده:)
   ├─→ Root Server:       «.com را کی جواب می‌دهد؟»  → آدرس TLD server
   ├─→ TLD Server (.com): «example.com را کی؟»       → آدرس Authoritative
   └─→ Authoritative NS:  «example.com = 93.184.216.34» ✅
   │
   ▼ (جواب + کش با TTL)
مرورگر اتصال TCP را به آن IP باز می‌کند
```

- درخواست کاربر→resolver: **بازگشتی (Recursive)** — «جواب نهایی را بده»
- درخواست‌های resolver→سرورها: **تکرارشونده (Iterative)** — «بگو از کی بپرسم»

### رکوردهای DNS

| رکورد | کار | مثال |
|---|---|---|
| `A` | اسم → IPv4 | `example.com → 93.184.216.34` |
| `AAAA` | اسم → IPv6 | |
| `CNAME` | اسم → اسم دیگر (alias) | `www → example.com` |
| `MX` | سرور ایمیل دامنه | `10 mail.example.com` |
| `TXT` | متن دلخواه | SPF/DKIM ایمیل، اثبات مالکیت دامنه |
| `NS` | سرورهای name این دامنه | |
| `SOA` | اطلاعات مرجع zone | serial، refresh |
| `PTR` | IP → اسم (reverse DNS) | |

> نکته: روی ریشه‌ی دامنه (apex مثل `example.com`) نمی‌توان CNAME گذاشت — راه‌حل: ALIAS/ANAME در DNSهای مدرن.

### کشینگ و TTL

- هر رکورد یک **TTL** (ثانیه) دارد — هر لایه (مرورگر، OS، resolver) تا انقضای آن کش می‌کند.
- ترفند عملی: قبل از مهاجرت سرور، TTL را از ۸۶۴۰۰ به ۳۰۰ کم کن؛ بعد از مهاجرت برگردان.
- **Propagation delay** در واقع همین انتظار برای انقضای کش‌هاست — DNS چیزی را «منتشر» نمی‌کند!

```bash
dig example.com               # رکورد A + TTL
dig example.com MX +short     # فقط جواب
dig @8.8.8.8 example.com      # از resolver مشخص بپرس
dig +trace example.com        # کل زنجیره از root تا authoritative
```

---

## ۴. فناوری‌های وب

### SSL/TLS

- SSL مرده است (آخرین نسخه ۱۹۹۶)؛ نام درست امروزی **TLS** است (نسخه‌های رایج: 1.2 و 1.3).
- **TLS 1.3 Handshake** (1-RTT):
  1. `ClientHello` — نسخه‌ها، cipherها، **و key share** (همین اول!)
  2. `ServerHello` + Certificate + key share → کلید مشترک از همین‌جا ساخته می‌شود
  3. کلاینت certificate را با زنجیره‌ی **CA** اعتبارسنجی می‌کند
- دو نوع رمزنگاری در کار است:
  - **نامتقارن** (RSA/ECDSA) — فقط برای هویت‌سنجی و توافق کلید (گران)
  - **متقارن** (AES-GCM, ChaCha20) — برای کل داده‌ی بعدی (ارزان)
- **Key exchange** با (EC)DHE → خاصیت **Forward Secrecy**: لو رفتن کلید خصوصی سرور، ترافیک گذشته را لو نمی‌دهد.
- **Let's Encrypt** certificate رایگان با چالش ACME می‌دهد.
- **SNI**: کلاینت در ClientHello اسم دامنه را می‌گوید تا یک IP بتواند certificate چند سایت را سرو کند.

### WebSocket

- ارتباط **full-duplex و پایدار** روی یک اتصال TCP. شروعش یک HTTP Upgrade است:

```http
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==

→ HTTP/1.1 101 Switching Protocols
```

- بعد از 101، دیگر HTTP نیست — قاب‌های (frame) دوطرفه است. Scheme: `ws://` و `wss://`
- مقایسه با جایگزین‌ها:
  - **Polling**: درخواست دوره‌ای — ساده، پرتأخیر و پرهزینه
  - **SSE (Server-Sent Events)**: فقط سرور→کلاینت، روی HTTP ساده — برای فید خبری کافی است
  - **WebSocket**: دوطرفه — چت، بازی، collaborative editing
- چالش عملی: load balancerها باید اتصال‌های طولانی را پشتیبانی کنند (timeoutها، sticky sessions در صورت stateful بودن).

### REST API

اصول:
- **منابع** با اسم جمع: `/users/42/orders` (نه `/getUserOrders?id=42`)
- **متدها** معنا را حمل می‌کنند: GET=خواندن، POST=ساختن، PUT/PATCH=ویرایش، DELETE=حذف
- **Stateless**: هر درخواست همه‌ی context لازم را دارد (توکن در هدر)
- Status code درست برگردان: ساختن=201، حذف موفق=204، نبود=404

طراحی عملی:

```
GET    /api/v1/users?page=2&limit=20&sort=-created_at   # pagination + sort
POST   /api/v1/users                  → 201 + Location: /api/v1/users/43
PATCH  /api/v1/users/43               → 200
DELETE /api/v1/users/43               → 204
```

- نسخه‌بندی: مسیر (`/v1/`) یا هدر (`Accept: application/vnd.api+json;version=1`)
- خطاها با بدنه‌ی ساخت‌یافته: `{"error": {"code": "USER_NOT_FOUND", "message": "..."}}`

### GraphQL

- **یک endpoint** (`POST /graphql`)، کلاینت **شکل دقیق داده** را تعیین می‌کند:

```graphql
query {
  user(id: 42) {
    name
    orders(last: 5) { total status }
  }
}
```

- حل دو مشکل REST: **over-fetching** (فیلدهای اضافه) و **under-fetching** (نیاز به چند درخواست — مسئله‌ی N+1 در سمت کلاینت)
- اجزا: **Schema** (typeهای strongly-typed)، **Resolver** (تابع پشت هر فیلد)، سه عمل: `query` / `mutation` / `subscription`
- هزینه‌ها: کش HTTP سخت‌تر (همه چیز POST است)، خطر کوئری‌های سنگین (نیاز به depth/complexity limit)، مسئله‌ی N+1 در resolverها (راه‌حل: **DataLoader** و batching)
- انتخاب: REST برای APIهای ساده و عمومی و کش‌پذیر؛ GraphQL برای کلاینت‌های متنوع (موبایل/وب) با نیازهای داده‌ای متفاوت.

---

## ۵. زیرساخت

### Load Balancer

توزیع ترافیک بین چند سرور پشتی (backend).

- **L4 (Transport)**: فقط IP/port را می‌بیند — سریع، بدون فهم HTTP
- **L7 (Application)**: محتوای HTTP را می‌فهمد — روتینگ بر اساس path/header، اما گران‌تر

الگوریتم‌ها:

| الگوریتم | منطق |
|---|---|
| Round Robin | نوبتی — ساده و پیش‌فرض |
| Weighted RR | سرور قوی‌تر، سهم بیشتر |
| Least Connections | به خلوت‌ترین بفرست |
| IP Hash | کلاینت همیشه به همان سرور (نوعی stickiness) |

- **Health Check**: بازرسی دوره‌ای (مثلاً `GET /healthz`)؛ سرور ناسالم از چرخش خارج می‌شود.
- **Sticky Session**: وقتی state در حافظه‌ی سرور است، کلاینت باید به همان سرور برگردد — ولی طراحی بهتر، stateless کردن سرورها و بردن state به Redis است.
- نمونه‌ها: Nginx، HAProxy، AWS ALB/NLB، Cloudflare

### Reverse Proxy

سرور واسطی **جلوی** سرورهای backend (برعکسِ forward proxy که جلوی *کلاینت‌هاست*):

- **TLS Termination** — رمزگشایی HTTPS در لبه، ترافیک داخلی ساده
- **Compression، Caching، Rate limiting، WAF**
- مخفی‌سازی توپولوژی داخلی؛ یک ورودی واحد

```nginx
server {
    listen 443 ssl;
    server_name api.example.com;

    location / {
        proxy_pass http://backend_pool;       # upstream با چند سرور = LB
        proxy_set_header X-Forwarded-For $remote_addr;  # IP واقعی کلاینت
        proxy_set_header Host $host;
    }
}
```

> در عمل Nginx همزمان reverse proxy و load balancer است — مرز این دو مفهوم در ابزارها محو است.

### CDN

شبکه‌ای از **edge serverها** در سراسر دنیا که محتوای کش‌شده را از نزدیک‌ترین نقطه به کاربر می‌دهند.

- **Origin** = سرور اصلی تو؛ **Edge/PoP** = نقاط حضور CDN
- **Cache hit** = جواب از edge؛ **Cache miss** = edge از origin می‌گیرد، کش می‌کند، می‌دهد
- کنترل با هدرها:

```http
Cache-Control: public, max-age=31536000, immutable   # asset نسخه‌دار (app.4f2a.js)
Cache-Control: no-store                              # داده‌ی شخصی — هرگز کش نکن
Cache-Control: s-maxage=3600, stale-while-revalidate=60  # کش CDN جدا از مرورگر
```

- **Cache busting**: hash محتوا در اسم فایل → با هر تغییر، URL جدید → کش قدیمی بی‌اثر
- **Invalidation/Purge**: پاک‌کردن دستی کش edge وقتی محتوا عوض شده
- CDNهای مدرن (Cloudflare, Fastly, CloudFront) فراتر از کش: DDoS protection، WAF، و **edge compute** (اجرای کد در لبه)

---

## 🔬 آزمایشگاه: کل مسیر را خودت ببین

```bash
# ۱. DNS
dig +trace example.com

# ۲. TCP + TLS + HTTP با جزئیات کامل
curl -v https://example.com/ 2>&1 | head -50

# ۳. زمان‌بندی هر مرحله از lifecycle
curl -w 'DNS: %{time_namelookup}s\nTCP: %{time_connect}s\nTLS: %{time_appconnect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n' \
     -o /dev/null -s https://example.com/

# ۴. certificate را بازرسی کن
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -dates -issuer -subject

# ۵. هدرهای کش CDN
curl -sI https://example.com/ | grep -iE 'cache|cf-|x-served|age'
```

---

## ✅ چک‌لیست تسلط

اگر بتوانی به این‌ها جواب بدهی، این فصل را بلدی:

- [ ] از Enter تا رندر صفحه، هر ۸ مرحله را با جزئیات توضیح بده
- [ ] فرق `401` و `403`؟ فرق `301` و `302`؟ فرق `502` و `504`؟
- [ ] چرا retry روی POST خطرناک است ولی روی PUT نه؟
- [ ] سه پیام TCP handshake و نقش sequence numberها؟
- [ ] چرا DNS «propagate» نمی‌شود و TTL واقعاً چه می‌کند؟
- [ ] Forward Secrecy یعنی چه و (EC)DHE چطور آن را می‌دهد؟
- [ ] کی WebSocket، کی SSE، کی polling؟
- [ ] فرق L4 و L7 load balancing؟ Sticky session چرا ضدالگوست؟
- [ ] `Cache-Control: public, max-age=31536000, immutable` را کلمه به کلمه شرح بده

---

## 🎯 سوالات مصاحبه

> روی هر سوال کلیک کن تا جواب را ببینی. اول خودت جواب بده، بعد چک کن!

### HTTP/HTTPS

<details><summary>**۱. وقتی در مرورگر `https://example.com` را می‌زنی و Enter می‌کنی، دقیقاً چه اتفاقی می‌افتد؟**</summary>

تجزیه‌ی URL → جستجوی DNS (کش مرورگر → OS → resolver → root → TLD → authoritative) → اتصال TCP با three-way handshake به پورت 443 → TLS handshake و توافق کلید → ارسال HTTP request → پردازش سرور → دریافت response → رندر HTML و درخواست‌های موازی برای CSS/JS/تصاویر. (این محبوب‌ترین سوال مصاحبه‌ی شبکه است — هر مرحله را باید بتوانی باز کنی.)

</details>

<details><summary>**۲. فرق `PUT` و `PATCH` چیست؟ فرق `PUT` و `POST` چطور؟**</summary>

`PUT` کل منبع را **جایگزین** می‌کند و idempotent است؛ `PATCH` فقط بخشی را تغییر می‌دهد و لزوماً idempotent نیست. `POST` منبع جدید می‌سازد (سرور URL را تعیین می‌کند) و idempotent نیست؛ `PUT` در آدرس مشخصی که کلاینت تعیین می‌کند می‌نویسد.

</details>

<details><summary>**۳. چرا retry خودکار روی POST خطرناک است ولی روی PUT و DELETE امن؟**</summary>

چون POST idempotent نیست — تکرارش ممکن است دو سفارش، دو پرداخت یا دو رکورد بسازد. PUT و DELETE با تکرار همان نتیجه را می‌دهند. راه‌حل برای POST: **Idempotency-Key** در هدر (مثل Stripe) تا سرور درخواست تکراری را تشخیص دهد.

</details>

<details><summary>**۴. فرق `401` و `403`؟**</summary>

`401 Unauthorized` یعنی «هویتت معلوم نیست — لاگین کن» (مشکل Authentication). `403 Forbidden` یعنی «می‌دانم کی هستی، ولی اجازه‌ی این کار را نداری» (مشکل Authorization).

</details>

<details><summary>**۵. فرق `301` و `302`؟ چرا اشتباه گرفتنشان در production دردسرساز است؟**</summary>

`301` انتقال دائمی و `302` موقت است. مرورگرها و CDNها `301` را **سخت کش می‌کنند** — اگر اشتباهی 301 بدهی و بعد پشیمان شوی، کاربرانِ قدیمی تا مدت‌ها به آدرس اشتباه می‌روند و کنترلی رویش نداری.

</details>

<details><summary>**۶. HTTP چرا stateless است و این محدودیت چطور حل می‌شود؟**</summary>

هر request مستقل است و سرور چیزی از request قبلی به یاد ندارد — این طراحی مقیاس‌پذیری را ساده می‌کند. حالت (state) با Cookie + Session سمت سرور، یا با توکن خودحامل مثل JWT برمی‌گردد.

</details>

<details><summary>**۷. فلگ‌های `HttpOnly`، `Secure` و `SameSite` روی کوکی چه می‌کنند؟**</summary>

`HttpOnly`: جاوااسکریپت نمی‌تواند کوکی را بخواند (دفاع XSS). `Secure`: فقط روی HTTPS ارسال می‌شود. `SameSite=Lax/Strict`: کوکی در درخواست‌های cross-site فرستاده نمی‌شود (دفاع CSRF).

</details>

<details><summary>**۸. HTTP/2 چه مشکلی از HTTP/1.1 را حل کرد؟ HTTP/3 چه چیزی را؟**</summary>

HTTP/1.1 روی هر اتصال یک درخواست در لحظه دارد (HOL blocking در لایه‌ی HTTP) — HTTP/2 با **multiplexing** چندین stream را روی یک اتصال TCP موازی کرد. اما گم‌شدن یک پکت TCP همه‌ی streamها را متوقف می‌کند (HOL در لایه‌ی انتقال) — HTTP/3 با رفتن روی **QUIC/UDP** این را هم حل کرد و handshake را سریع‌تر کرد.

</details>

### TCP/IP

<details><summary>**۹. چرا TCP handshake سه مرحله‌ای است و دو مرحله کافی نیست؟**</summary>

هر دو طرف باید مطمئن شوند که **هم فرستادن و هم شنیدنِ** طرف مقابل کار می‌کند و sequence number اولیه‌ی هر دو رد و بدل شود. با دو پیام، فرستنده‌ی SYN-ACK هرگز نمی‌فهمد جوابش رسیده یا نه — و اتصال‌های نیمه‌باز و پکت‌های قدیمیِ سرگردان قابل تشخیص نمی‌شوند.

</details>

<details><summary>**۱۰. TCP چطور تضمین می‌کند داده‌ها به ترتیب و کامل برسند؟**</summary>

Sequence number برای ترتیب، ACK برای تأیید دریافت، retransmission با timeout یا 3 duplicate ACK برای جبران گم‌شدن، checksum برای سلامت، و sliding window برای کنترل جریان.

</details>

<details><summary>**۱۱. کی UDP را به TCP ترجیح می‌دهی؟ مثال واقعی بزن.**</summary>

وقتی **تازگی** مهم‌تر از **کامل بودن** است: تماس تصویری، استریم زنده، بازی آنلاین — فریم گم‌شده را نمی‌خواهی دوباره بفرستی چون دیگر دیر شده. همچنین DNS (یک پرسش/پاسخ کوچک — هزینه‌ی handshake نمی‌ارزد) و QUIC/HTTP3 که reliability را خودش روی UDP پیاده کرده.

</details>

<details><summary>**۱۲. حالت TIME_WAIT چیست و چرا سرورِ پرترافیک ممکن است از آن رنج ببرد؟**</summary>

بعد از بستن اتصال، طرفی که اول FIN داده تا 2×MSL صبر می‌کند تا پکت‌های سرگردانِ اتصال قدیمی بمیرند و با اتصال جدید قاطی نشوند. سروری که هزاران اتصال کوتاه باز و بسته می‌کند، هزاران سوکت در TIME_WAIT جمع می‌کند و ممکن است ephemeral portهایش تمام شود — راه‌حل: connection pooling و keep-alive.

</details>

<details><summary>**۱۳. NAT چیست و چرا با وجود آن، چند دستگاه خانه‌ات با یک IP عمومی کار می‌کنند؟**</summary>

روتر، آدرس‌های خصوصی داخلی (`192.168.x.x`) را پشت IP عمومی پنهان می‌کند: برای هر اتصال خروجی یک نگاشت `(IP داخلی, پورت داخلی) ↔ (IP عمومی, پورت جدید)` در جدولش می‌سازد و جواب‌ها را برمی‌گرداند. به همین دلیل اتصالِ **ورودی** بدون port forwarding ممکن نیست.

</details>

<details><summary>**۱۴. یک اتصال TCP با چه چیزی یکتا شناخته می‌شود؟ پس چطور هزاران کلاینت همزمان به پورت 443 یک سرور وصل‌اند؟**</summary>

با چهارتایی `(src IP, src port, dst IP, dst port)`. پورت 443 مقصد ثابت است، اما هر کلاینت IP و پورتِ مبدأ متفاوتی دارد — پس هر اتصال چهارتاییِ یکتای خودش را دارد.

</details>

### DNS

<details><summary>**۱۵. فرق query بازگشتی (Recursive) و تکرارشونده (Iterative)؟**</summary>

کاربر از resolver می‌پرسد و **جواب نهایی** می‌خواهد (بازگشتی). resolver خودش از root → TLD → authoritative قدم‌به‌قدم می‌پرسد و هر بار فقط «آدرس قدم بعدی» می‌گیرد (تکرارشونده).

</details>

<details><summary>**۱۶. «DNS هنوز propagate نشده» — این جمله دقیقاً چه ایرادی دارد؟**</summary>

DNS چیزی را push یا «منتشر» نمی‌کند. رکورد قدیمی در کشِ resolverها تا انقضای **TTL** زنده می‌ماند؛ هر resolver وقتی کشش منقضی شد، خودش جواب تازه را می‌گیرد. «انتظار propagate» در واقع «انتظار انقضای TTL کش‌ها» است — و راه‌حل حرفه‌ای، کم کردن TTL **قبل از** مهاجرت است.

</details>

<details><summary>**۱۷. فرق رکورد A و CNAME؟ چرا روی apex دامنه CNAME ممنوع است؟**</summary>

`A` اسم را مستقیم به IPv4 می‌رساند؛ `CNAME` اسم را به اسمِ دیگری حواله می‌دهد. استاندارد می‌گوید CNAME نمی‌تواند کنار رکوردهای دیگر بنشیند، و apex حتماً رکوردهای SOA و NS دارد — پس تداخل می‌شود. DNSهای مدرن با ALIAS/ANAME دورش می‌زنند.

</details>

<details><summary>**۱۸. سناریو: دامنه را به سرور جدید منتقل کرده‌ای؛ نیمی از کاربران سایت قدیمی را می‌بینند. چرا و چه می‌کنی؟**</summary>

resolverهای مختلف، کشِ با TTLهای متفاوت دارند — بعضی هنوز IP قدیمی را نگه داشته‌اند. کاری از دستت برنمی‌آید جز صبر تا انقضای TTL. درس: دفعه‌ی بعد ۲۴–۴۸ ساعت قبل از مهاجرت، TTL را به ۳۰۰ ثانیه کم کن، و سرور قدیمی را تا انقضای کامل، روشن (یا redirect) نگه دار.

</details>

<details><summary>**۱۹. خروجی `dig +trace example.com` چه مراحلی را نشان می‌دهد؟**</summary>

کل زنجیره‌ی resolution را بدون کش: پاسخ root serverها (معرفی TLD)، پاسخ سرورهای `.com` (معرفی authoritative)، و در نهایت رکورد A از authoritative nameserver خود دامنه.

</details>

### TLS و امنیت انتقال

<details><summary>**۲۰. در TLS، رمزنگاری متقارن و نامتقارن هرکدام کجا استفاده می‌شوند و چرا هر دو لازم‌اند؟**</summary>

نامتقارن (RSA/ECDSA + ECDHE) فقط در handshake: اثبات هویت سرور و توافق امن روی کلید — چون کند و گران است. بعد از آن، کل داده با رمزنگاری متقارن (AES-GCM/ChaCha20) می‌رود که سریع است. نامتقارن مشکلِ «تبادل کلید روی کانال ناامن» را حل می‌کند، متقارن مشکلِ سرعت را.

</details>

<details><summary>**۲۱. Forward Secrecy چیست و چرا مهم است؟**</summary>

یعنی لو رفتنِ کلید خصوصی سرور در آینده، ترافیکِ ضبط‌شده‌ی گذشته را قابل رمزگشایی نمی‌کند — چون کلید هر session با (EC)DHE جداگانه و موقتی ساخته شده و هیچ‌جا ذخیره نیست. بدون آن، حمله‌ی «الان ضبط کن، بعداً رمزگشایی کن» ممکن می‌شود.

</details>

<details><summary>**۲۲. مرورگر از کجا مطمئن می‌شود certificate سایت جعلی نیست؟**</summary>

زنجیره‌ی اعتماد: certificate سایت توسط یک CA میانی امضا شده، آن هم توسط Root CA — و کلید عمومی Root CAها از قبل در مرورگر/OS نصب است. مرورگر امضاها را تا ریشه چک می‌کند، به‌علاوه‌ی تاریخ انقضا، تطابق دامنه (SAN) و revocation.

</details>

<details><summary>**۲۳. SNI چه مشکلی را حل می‌کند؟**</summary>

سرور با یک IP میزبان چند دامنه است، اما TLS **قبل از** HTTP اتفاق می‌افتد و سرور نمی‌داند کدام certificate را بدهد. کلاینت در ClientHello اسم دامنه را می‌فرستد تا سرور certificate درست را انتخاب کند.

</details>

### WebSocket، REST و GraphQL

<details><summary>**۲۴. WebSocket چطور برقرار می‌شود و فرقش با polling و SSE چیست؟**</summary>

با یک HTTP request معمولی با هدر `Upgrade: websocket` شروع می‌شود؛ سرور با `101 Switching Protocols` جواب می‌دهد و از آن به بعد اتصال TCP به کانال دوطرفه‌ی پایدار تبدیل می‌شود. Polling = پرسیدن دوره‌ای (پرهزینه/پرتأخیر)؛ SSE = جریان یک‌طرفه‌ی سرور→کلاینت روی HTTP؛ WebSocket = دوطرفه‌ی کامل.

</details>

<details><summary>**۲۵. REST API برای حذف منبعی که وجود ندارد باید 404 برگرداند یا 204؟ استدلال کن.**</summary>

هر دو قابل دفاع است و مصاحبه‌گر استدلال می‌خواهد: `404` دقیق‌تر است («چنین منبعی نیست»)، اما چون DELETE idempotent است، برخی `204` می‌دهند تا retry همان نتیجه را بگیرد. مهم: در کل API **یکدست** باش و مستندش کن.

</details>

<details><summary>**۲۶. GraphQL چه دو مشکلی از REST را حل می‌کند و در عوض چه هزینه‌هایی دارد؟**</summary>

حل می‌کند: over-fetching (فیلدهای اضافه) و under-fetching (چند round-trip برای یک صفحه). هزینه‌ها: کش HTTP سخت‌تر (همه POST)، خطر کوئری‌های عمیق/سنگین (نیاز به complexity limit)، مسئله‌ی N+1 در resolverها (راه‌حل: DataLoader/batching)، و پیچیدگی بیشتر سمت سرور.

</details>

### زیرساخت و سناریوهای ترکیبی

<details><summary>**۲۷. فرق Load Balancer لایه ۴ و لایه ۷؟ هرکدام کی؟**</summary>

L4 فقط IP/port را می‌بیند — سریع و ارزان، برای ترافیک خام TCP/UDP. L7 محتوای HTTP را می‌فهمد — روتینگ بر اساس path/header/cookie، TLS termination، WAF؛ ولی پردازش بیشتری می‌خواهد. APIهای وب معمولاً L7، دیتابیس و ترافیک غیر HTTP معمولاً L4.

</details>

<details><summary>**۲۸. چرا Sticky Session ضدالگو محسوب می‌شود و جایگزینش چیست؟**</summary>

چون توزیع بار را نامتوازن می‌کند، با مرگ یک سرور سشن‌های آن از بین می‌روند، و scale-in/out را دردناک می‌کند. جایگزین: سرورهای stateless و بردن state به یک store مشترک مثل Redis، یا توکن خودحامل (JWT).

</details>

<details><summary>**۲۹. فرق `502 Bad Gateway` و `504 Gateway Timeout`؟ هرکدام به چه مشکلی اشاره دارد؟**</summary>

هر دو از زبان proxy/LB هستند: `502` یعنی backend جوابِ **نامعتبر** داد یا اتصال رد شد (پروسه crash کرده، پورت اشتباه، backend down). `504` یعنی backend در مهلت مقرر **اصلاً جواب نداد** (کوئری کند، deadlock، timeout ناهماهنگ بین LB و اپ).

</details>

<details><summary>**۳۰. سناریو: سایتت فقط برای کاربران آسیا کند است. قدم‌به‌قدم چطور دیباگ می‌کنی؟**</summary>

۱) از همان منطقه تست بگیر (synthetic monitoring یا VPN). ۲) با `curl -w` ببین زمان کجا می‌رود: DNS؟ TCP/TLS (نشانه‌ی RTT بالا = نبود PoP نزدیک)؟ TTFB (مشکل سرور)؟ دانلود (پهنای باند)؟ ۳) چک کن CDN در آن منطقه edge دارد و cache hit ratio چقدر است؛ هدر `cf-cache-status`/`x-cache` را ببین. ۴) اگر محتوا dynamic است، هر درخواست به origin دور برمی‌گردد — راه‌حل: edge caching بیشتر، origin منطقه‌ای، یا CDN با شبکه‌ی بهتر در آسیا.

</details>

<details><summary>**۳۱. سناریو: بعد از deploy، نیمی از درخواست‌ها 404 می‌گیرند و نیمی درست‌اند. حدس اولت چیست؟**</summary>

پشت load balancer چند سرور هست و deploy فقط روی بعضی‌ها انجام شده (یا rolling deploy وسط راه مانده/شکست خورده) — درخواست‌ها بسته به اینکه به سرور قدیمی یا جدید بیفتند نتیجه‌ی متفاوت می‌گیرند. چک: نسخه‌ی در حال اجرا روی تک‌تک backendها؛ درخواستِ assetهای hash‌دارِ جدید از سرورهای قدیمی، نمونه‌ی کلاسیک همین مشکل است.

</details>

<details><summary>**۳۲. سناریو: کاربری می‌گوید «سایت شما باز نمی‌شود» ولی برای تو باز می‌شود. چه چیزهایی را به چه ترتیبی چک می‌کنی؟**</summary>

۱) DNS سمت او — `nslookup` (کش قدیمی؟ DNS مسدود؟). ۲) شبکه‌ی او — `ping`/`traceroute` (فایروال شرکتی، ISP). ۳) TLS — ساعتِ سیستمش غلط نیست؟ (خطای certificate). ۴) کش مرورگر / extensionها — حالت incognito. ۵) سمت خودت: لاگ edge/CDN برای IP او، آیا منطقه/IPاش بلاک شده، آیا برای segment خاصی خطای 4xx/5xx ثبت شده.

</details>
