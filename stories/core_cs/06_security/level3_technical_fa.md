# سطح ۳ — توضیح فنی: Application & Web Security ⚙️🛡️

> دیگر قصه‌ای در کار نیست. این سند مرجع فنیِ امنیتِ تدافعی است — همان چیزهایی که در مصاحبه و کارِ واقعی لازم داری. نگاه از زاویه‌ی **مدافع**: حمله مفهوماً چطور کار می‌کند و **چطور جلویش را بگیریم**.

---

## ۱. Authentication (AuthN)

«احراز هویت» = اثباتِ اینکه کاربر همانی است که ادعا می‌کند. سه استراتژیِ نگه‌داشتنِ هویت روی پروتکلِ stateless وب:

### Session-based (Stateful)

- سرور بعد از login یک **session record** می‌سازد (در حافظه/Redis/DB) و یک **session ID** تصادفی در کوکی می‌گذارد.
- کوکی فقط **شناسه** را حمل می‌کند؛ داده در سرور است. سرور با هر درخواست شناسه را lookup می‌کند.
- خروج (logout) = حذفِ رکورد در سرور → فوری و قطعی.
- چالشِ مقیاس: state مشترک لازم است (Redis) یا sticky session.

### JWT (Stateless)

ساختار: سه بخشِ **base64url** که با نقطه جدا می‌شوند — `header.payload.signature`

```
header:    {"alg":"HS256","typ":"JWT"}
payload:   {"sub":"42","role":"chef","exp":1735689600,"iat":1735688700}
signature: HMAC-SHA256(base64(header) + "." + base64(payload), secret)
```

- `header` و `payload` فقط **encode** شده‌اند، **نه encrypt** — هرکس می‌تواند بخواندشان. هرگز داده‌ی محرمانه در payload نگذار.
- امنیت از **signature** می‌آید: دستکاریِ payload → امضا نمی‌خواند → رد.
- `alg`: متقارن (`HS256` با secret مشترک) یا نامتقارن (`RS256` با کلیدِ خصوصی/عمومی).

| | Session | JWT |
|---|---|---|
| State | سمت سرور | سمت کلاینت (خودکفا) |
| مقیاس‌پذیری افقی | نیاز به store مشترک | عالی (بدون lookup) |
| ابطالِ فوری (logout) | ساده | سخت (نیاز به blocklist) |
| اندازه | کوچک (فقط ID) | بزرگ‌تر (در هر درخواست) |

**Expiry / Refresh:** الگوی استاندارد دو توکن است:
- **Access token**: عمرِ کوتاه (۵–۱۵ دقیقه)، در هر درخواست در هدرِ `Authorization: Bearer ...`.
- **Refresh token**: عمرِ طولانی، فقط برای گرفتنِ access token تازه؛ امن نگه‌داری می‌شود (HttpOnly cookie) و قابلِ ابطال است (در DB نگه می‌داریم → rotation).

> نکته‌ی مصاحبه: «`exp` چک نمی‌شود اگر فقط امضا را verify کنی؟» — بعضی لایبرری‌ها به‌صورتِ پیش‌فرض `exp` را چک می‌کنند، بعضی نه. همیشه مطمئن شو `exp`، `nbf`، `aud`، `iss` اعتبارسنجی می‌شوند، نه فقط امضا.

> نکته‌ی مصاحبه: حمله‌ی کلاسیک **`alg: none`** و **algorithm confusion** (سرور `RS256` انتظار دارد ولی مهاجم با کلیدِ عمومی به‌عنوان secretِ `HS256` توکن می‌سازد). دفاع: `alg` را در کد **whitelist** کن، هرگز به `alg`ِ داخلِ توکن اعتماد نکن.

### OAuth2 & OIDC

OAuth2 یک **چارچوبِ authorization** است (نه authentication؛ آن لایه را **OIDC** اضافه می‌کند). هدف: دادنِ دسترسیِ محدود بدونِ به‌اشتراک‌گذاریِ رمز.

نقش‌ها: **Resource Owner** (کاربر) · **Client** (اپ) · **Authorization Server** (صادرکننده‌ی توکن) · **Resource Server** (API).

**Authorization Code Flow** (مناسب برای اپ‌های با backend):

```
Client → AuthServer:  /authorize?response_type=code&client_id=..&redirect_uri=..&scope=..&state=..
User   → AuthServer:  لاگین + رضایت (رمز فقط اینجا، نه پیش Client)
AuthServer → Client:  redirect با ?code=ABC&state=..
Client → AuthServer:  POST /token  (code + client_id + client_secret)   ← کانالِ پشتی
AuthServer → Client:  { access_token, refresh_token, id_token(OIDC) }
Client → ResourceSrv: GET /api  با  Authorization: Bearer access_token
```

- **چرا code، نه token مستقیم؟** code از مرورگر (قابلِ افشا) رد می‌شود اما token در کانالِ server-to-server با `client_secret` گرفته می‌شود؛ ربودنِ code بدونِ secret بی‌فایده است.
- **چرا پارامترِ `state`؟** دفاع در برابرِ CSRF روی خودِ flowِ OAuth.
- **PKCE** (`code_challenge`/`code_verifier`): برای کلاینت‌های بدونِ secret (SPA/موبایل) جایگزینِ `client_secret` می‌شود؛ امروز برای *همه* توصیه می‌شود.
- **چرا نه password sharing؟** اگر کاربر رمزِ گوگلش را به اپ بدهد: اپ دسترسیِ کامل و دائمی دارد، رمز قابلِ scope-محدودسازی و ابطال نیست، و در صورتِ نشتِ اپ رمزِ اصلی لو می‌رود.

---

## ۲. Authorization (AuthZ)

AuthN جداست از AuthZ. اول می‌پرسیم «کی هستی؟»، بعد «حق داری؟».

| | Authentication | Authorization |
|---|---|---|
| سؤال | identity | permission |
| کدِ شکست | `401 Unauthorized` | `403 Forbidden` |
| ترتیب | اول | بعد |

### RBAC و Permissions

- **RBAC**: `user → role(s) → permissions`. مدیریت ساده، نقش‌محور.
- **Permission** واحدِ ریزِ مجوز است (`order:read`, `order:delete`)؛ نقش مجموعه‌ای از permissionهاست.
- مدل‌های دیگر: **ABAC** (بر اساسِ attributeها و context، مثلاً «فقط در ساعتِ کاری» یا «فقط مالکِ منبع»)، **ReBAC** (رابطه‌محور، مثلِ Google Zanzibar).

```python
def require(permission):
    def decorator(fn):
        def wrapper(*a, **k):
            if permission not in current_user.permissions:
                abort(403)
            return fn(*a, **k)
        return wrapper
    return decorator

@app.route("/orders/<int:order_id>")
@require("order:read")
def get_order(order_id):
    order = Order.get(order_id)
    # ⚠️ حیاتی: علاوه بر permission، مالکیت را هم چک کن (جلوگیری از IDOR)
    if order.owner_id != current_user.id and not current_user.is_admin:
        abort(403)
    return order
```

> نکته‌ی مصاحبه: **IDOR / Broken Object Level Authorization** پرتکرارترین باگِ AuthZ است. چک کردنِ «لاگین هستی؟» کافی نیست؛ باید چک کنی «این *منبعِ خاص* مالِ توست؟». همیشه authorization را **سمتِ سرور** انجام بده — مخفی‌کردنِ دکمه در UI، امنیت نیست.

---

## ۳. Cryptography

### Hashing (برای رمزِ عبور)

| ویژگی | توضیح |
|---|---|
| یک‌طرفه | از خروجی نمی‌توان به ورودی رسید |
| Deterministic | همان ورودی → همان خروجی (به‌علاوه‌ی salt) |
| Avalanche | تغییرِ یک بیت → خروجیِ کاملاً متفاوت |

- **برای رمزِ عبور MD5/SHA1/SHA256 ممنوع** — چون سریع‌اند → میلیاردها brute-force در ثانیه + rainbow tables.
- از **password hashing**های عمدی‌کُند استفاده کن: **bcrypt**، **scrypt**، یا (توصیه‌ی امروز) **argon2id**.
- **Salt**: مقدارِ تصادفیِ یکتا per-password؛ rainbow table را بی‌اثر می‌کند و hashهای یکسان را متفاوت. (bcrypt salt را خودش تولید و در خروجی embed می‌کند.)
- **Pepper** (اختیاری): یک secretِ سراسری که جدا از DB (مثلاً در HSM/env) نگه داشته می‌شود.
- **Work factor** را با سخت‌افزار به‌روز نگه دار (bcrypt cost، argon2 memory/time).

```python
# ❌ آسیب‌پذیر
import hashlib
stored = hashlib.sha256(password.encode()).hexdigest()   # سریع، بدون salt → شکستنی

# ✅ امن
import bcrypt
stored = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
valid  = bcrypt.checkpw(attempt.encode(), stored)

# ✅ امن‌تر (توصیه‌ی مدرن)
from argon2 import PasswordHasher
ph = PasswordHasher()
stored = ph.hash(password)
ph.verify(stored, attempt)   # exception اگر غلط
```

> نکته‌ی مصاحبه: «hash در برابر encrypt برای رمزِ عبور؟» encrypt **برگشت‌پذیر** است — کلید لو برود، همه‌ی رمزها لو می‌روند. رمزِ عبور را نباید بشود برگرداند، پس **hash** (یک‌طرفه). برای مقایسه‌ی hash از **constant-time comparison** استفاده کن تا timing attack نشود.

### Encryption

| | Symmetric | Asymmetric |
|---|---|---|
| کلید | یک کلیدِ مشترک | جفتِ public/private |
| الگوریتم | **AES** (AES-256-GCM), ChaCha20 | **RSA**, ECC (ECDSA/ECDH) |
| سرعت | سریع | کند (۱۰۰۰× کندتر) |
| مشکلی که حل می‌کند | رمزِ حجمِ زیادِ داده | توزیعِ کلید، امضای دیجیتال |
| محدودیت | توزیعِ امنِ کلید سخت است | فقط برای داده‌ی کوچک |

- **عملاً ترکیبی (hybrid):** TLS از RSA/ECDHE فقط برای **توافقِ یک session keyِ متقارن** استفاده می‌کند، بعد کلِ داده با AES-GCM رمز می‌شود.
- **AEAD** (مثلِ AES-GCM, ChaCha20-Poly1305): همزمان confidentiality + integrity می‌دهد — به‌جای حالت‌های قدیمی مثلِ AES-CBC که نیاز به MAC جدا دارند.
- **امضای دیجیتال:** با کلیدِ **خصوصی** امضا، با **عمومی** verify → اثباتِ اصالت و integrity (نه محرمانگی). برعکسِ رمزنگاری که با عمومی قفل و با خصوصی باز می‌شود.
- هرگز کلید را hard-code یا در گیت نگذار؛ از KMS/Vault/Secrets Manager استفاده کن. کلید را rotate کن.

> نکته‌ی مصاحبه: «داده در حالِ انتقال در برابر داده در حالِ سکون؟» **In transit** = TLS. **At rest** = رمزِ دیسک/فیلد در DB با AES. هر دو لازم‌اند.

---

## ۴. Web Security

### OWASP Top 10 (2021)

| # | دسته | خلاصه‌ی دفاع |
|---|---|---|
| A01 | Broken Access Control | چکِ AuthZ سمتِ سرور + جلوگیری از IDOR |
| A02 | Cryptographic Failures | TLS همه‌جا، AES at-rest، رمزِ عبور با argon2 |
| A03 | Injection (SQLi/XSS/...) | parameterized queries، output encoding |
| A04 | Insecure Design | threat modeling، secure-by-design |
| A05 | Security Misconfiguration | سخت‌سازی، حذفِ پیش‌فرض‌ها، هدرهای امنیتی |
| A06 | Vulnerable & Outdated Components | SCA، به‌روزرسانی، `npm audit`/`pip-audit` |
| A07 | Identification & Auth Failures | MFA، rate-limit، مدیریتِ امنِ session |
| A08 | Software & Data Integrity Failures | امضای artifactها، قفلِ dependency |
| A09 | Security Logging & Monitoring Failures | لاگِ رخدادهای امنیتی + alerting |
| A10 | SSRF | allowlist مقصد، مسدودکردنِ متادیتای داخلی |

### SQL Injection

ریشه: ترکیبِ untrusted input با کوئری به‌صورتِ رشته → داده به‌عنوان **کد** تفسیر می‌شود.

```python
# ❌ آسیب‌پذیر
cur.execute("SELECT * FROM users WHERE email = '%s'" % email)
#  email = "x' OR '1'='1"  → bypass؛   "x'; DROP TABLE users; --"  → تخریب

# ✅ امن — parameterized / prepared statement
cur.execute("SELECT * FROM users WHERE email = ?", (email,))   # sqlite/python
cur.execute("SELECT * FROM users WHERE email = %s", (email,))  # psycopg
```

```javascript
// ❌ آسیب‌پذیر (Node)
db.query(`SELECT * FROM users WHERE email = '${email}'`)
// ✅ امن
db.query("SELECT * FROM users WHERE email = $1", [email])
```

دفاعِ لایه‌ای: parameterized queries (اصلی) · ORM با bound params · اصلِ **least privilege** برای کاربرِ DB · اعتبارسنجیِ ورودی · هرگز پیامِ خطای خامِ DB را به کاربر نشان نده. نکته: identifierها (نامِ جدول/ستون) parameterize نمی‌شوند → از allowlist استفاده کن.

### XSS (Cross-Site Scripting)

اجرای اسکریپتِ مهاجم در مرورگرِ قربانی → دزدیِ session، keylogging، تغییرِ DOM.

| نوع | منبع | دامنه |
|---|---|---|
| **Stored** | در DB ذخیره و به همه سرو می‌شود | بالا — هر بیننده |
| **Reflected** | در URL/پاسخِ بلافاصله بازتاب می‌شود | فقط قربانیِ کلیک‌کننده |
| **DOM-based** | جاوااسکریپتِ سمتِ کلاینت داده‌ی کاربر را در DOM می‌ریزد | سمتِ کلاینت |

```javascript
// ❌ آسیب‌پذیر — DOM-based
el.innerHTML = location.hash;            // <img src=x onerror=alert(1)>
// ✅ امن
el.textContent = location.hash;          // به‌عنوان متن، نه HTML
```

دفاع:
1. **Output encoding / contextual escaping** هنگامِ رندر (نه ورودی) — فریم‌ورک‌های مدرن autoescape دارند؛ از `innerHTML`/`dangerouslySetInnerHTML`/`|safe` دوری کن یا با DOMPurify پاک‌سازی کن.
2. **CSP**: `Content-Security-Policy: default-src 'self'; script-src 'self'` — از `unsafe-inline`/`unsafe-eval` بپرهیز؛ از nonce/hash استفاده کن.
3. کوکیِ session را **`HttpOnly`** کن (جاوااسکریپت دسترسی ندارد).

### CSRF (Cross-Site Request Forgery)

سوءاستفاده از credentialِ خودکارِ مرورگر (کوکی) برای انجامِ عملِ تغییردهنده به‌جای کاربر. تنها به درخواست‌های **state-changing** (POST/PUT/DELETE) مربوط است.

```html
<!-- صفحه‌ی مهاجم -->
<form action="https://bank.com/transfer" method="POST">
  <input name="to" value="attacker"><input name="amount" value="9999">
</form><script>document.forms[0].submit()</script>
```

دفاع:
1. **`SameSite` cookie**: `SameSite=Lax` (پیش‌فرضِ مرورگرهای مدرن) یا `Strict` — کوکی در درخواست‌های cross-site فرستاده نمی‌شود.
2. **CSRF token** (synchronizer token / double-submit): توکنِ تصادفیِ per-session/per-request که سایتِ مهاجم نمی‌داند.
3. اعتبارسنجیِ هدرِ `Origin`/`Referer` برای درخواست‌های حساس.

```http
Set-Cookie: session=...; HttpOnly; Secure; SameSite=Lax
```

> نکته‌ی مصاحبه: **XSS در برابر CSRF.** XSS = اجرای کدِ مهاجم در مرورگرِ قربانی (نقصِ اعتماد به *محتوا*). CSRF = جعلِ درخواست با credentialِ موجودِ قربانی (نقصِ اعتماد به *منشأ*). اگر XSS داری، CSRF token هم بی‌فایده است (اسکریپت توکن را می‌خواند). APIهای مبتنی بر `Authorization: Bearer` (نه کوکی) ذاتاً در برابرِ CSRF مصون‌اند چون مرورگر هدر را خودکار نمی‌فرستد.

### هدرهای امنیتیِ کلیدی

```http
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload   # اجبارِ HTTPS
Content-Security-Policy: default-src 'self'                                # ضدِ XSS
X-Content-Type-Options: nosniff                                           # ضدِ MIME-sniffing
X-Frame-Options: DENY                                                     # ضدِ clickjacking
Referrer-Policy: strict-origin-when-cross-origin
Set-Cookie: id=...; HttpOnly; Secure; SameSite=Lax
```

---

## 🔬 آزمایشگاه

تمرینِ عملیِ امن و محلی. همه‌چیز روی ماشینِ خودت.

### ۱. یک JWT را با base64 رمزگشایی کن

```bash
# یک JWT نمونه (همان امضای جعلی—فقط برای دیدنِ ساختار)
JWT='eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiIsInJvbGUiOiJjaGVmIiwiZXhwIjoxNzM1Njg5NjAwfQ.abc123signature'

# header را decode کن (بخشِ اول، تا نقطه‌ی اول)
echo "$JWT" | cut -d. -f1 | tr '_-' '/+' | base64 -d 2>/dev/null; echo
# → {"alg":"HS256","typ":"JWT"}

# payload را decode کن (بخشِ دوم)
echo "$JWT" | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null; echo
# → {"sub":"42","role":"chef","exp":1735689600}
```

نتیجه‌ی کلیدی که باید با چشمِ خودت ببینی: **payload رمز نشده، فقط encode شده.** هرکس با همین دستور می‌تواند بخواندش → هرگز secret/PII در payload نگذار.

### ۲. یک رمز عبور را با bcrypt در پایتون hash کن

```bash
pip install bcrypt
```

```python
import bcrypt

password = b"correct horse battery staple"
hashed = bcrypt.hashpw(password, bcrypt.gensalt(rounds=12))
print(hashed)            # $2b$12$...  (الگوریتم + cost + salt + hash، همه با هم)

# همان رمز را دو بار hash کن — خروجی متفاوت است (salt تصادفی)!
print(bcrypt.hashpw(password, bcrypt.gensalt()))

# اعتبارسنجی
print(bcrypt.checkpw(b"correct horse battery staple", hashed))  # True
print(bcrypt.checkpw(b"wrong password", hashed))                # False
```

ببین که دو بار hash کردنِ یک رمز، دو خروجیِ متفاوت می‌دهد (به‌لطفِ salt) ولی هر دو با `checkpw` معتبرند.

### ۳. Parameterized query در برابر string concat روی sqlite محلی

```python
import sqlite3
db = sqlite3.connect(":memory:")
db.executescript("""
  CREATE TABLE users(id INTEGER, name TEXT, secret TEXT);
  INSERT INTO users VALUES (1,'alice','A-SECRET'),(2,'bob','B-SECRET');
""")

evil = "x' OR '1'='1"

# ❌ آسیب‌پذیر — همه‌ی رازها لو می‌رود
q = "SELECT name, secret FROM users WHERE name = '%s'" % evil
print("vulnerable:", db.execute(q).fetchall())
# → [('alice','A-SECRET'), ('bob','B-SECRET')]   ← bypass شد!

# ✅ امن — ورودیِ بد فقط یک رشته‌ی جستجوست
print("safe:", db.execute(
    "SELECT name, secret FROM users WHERE name = ?", (evil,)).fetchall())
# → []   ← هیچ کاربری به اسمِ "x' OR '1'='1" وجود ندارد
```

### ۴. فلگ‌های کوکی را در DevTools مرورگر بازرسی کن

1. هر سایتی که login دارد را باز کن (مثلاً github.com).
2. DevTools → تب **Application** (کروم) یا **Storage** (فایرفاکس) → **Cookies**.
3. ستون‌های **HttpOnly**، **Secure**، **SameSite** را نگاه کن.
4. تأیید کن کوکیِ session هر سه را دارد. در Console اجرا کن:

```javascript
document.cookie   // کوکیِ session نباید اینجا دیده شود اگر HttpOnly باشد
```

نتیجه‌ی کلیدی: کوکیِ نشستِ امن باید `HttpOnly` + `Secure` + `SameSite` داشته باشد و در `document.cookie` **نمایان نشود**.

---

## ✅ چک‌لیست تسلط

اگر بتوانی به این‌ها جواب بدهی، این فصل را بلدی:

- [ ] فرقِ AuthN و AuthZ؟ کدام `401` و کدام `403` می‌دهد؟
- [ ] سه بخشِ JWT و نقشِ هرکدام؟ چرا payload «امن» نیست برای داده‌ی محرمانه؟
- [ ] چرا JWT مقیاس‌پذیر است ولی logout فوری در آن سخت است؟ access در برابر refresh token؟
- [ ] چهار نقشِ OAuth2 و گام‌به‌گامِ authorization code flow؟ چرا code می‌دهند نه token؟ PKCE چرا؟
- [ ] RBAC چیست و IDOR چطور رخ می‌دهد و چطور جلویش را می‌گیری؟
- [ ] چرا رمزِ عبور را hash می‌کنیم نه encrypt؟ چرا MD5/SHA256 بد و bcrypt/argon2 خوب؟ salt چه می‌کند؟
- [ ] symmetric در برابر asymmetric — هرکدام کِی؟ TLS چطور هر دو را با هم به کار می‌برد؟
- [ ] SQLi چطور رخ می‌دهد و parameterized query دقیقاً چرا جلویش را می‌گیرد؟
- [ ] stored در برابر reflected XSS؟ نقشِ output encoding، CSP و HttpOnly؟
- [ ] CSRF چطور کار می‌کند و SameSite و CSRF token هرکدام چطور دفاع می‌کنند؟
- [ ] چرا اگر XSS داشته باشی CSRF token بی‌فایده است؟

---

## 🎯 سوالات مصاحبه

### Authentication

<details><summary>**فرق Session-based auth و JWT چیست؟**</summary>

Session **stateful** است: سرور یک رکورد نگه می‌دارد و کوکی فقط یک session ID حمل می‌کند؛ هر درخواست lookup می‌خورد. JWT **stateless** است: همه‌ی claimها داخلِ خودِ توکن‌اند و سرور فقط امضا را verify می‌کند (بدونِ lookup). Session ابطالِ فوری دارد ولی به store مشترک نیاز دارد؛ JWT افقی عالی مقیاس می‌خورد ولی ابطالِ فوری سخت است (blocklist لازم می‌شود).

</details>

<details><summary>**JWT در localStorage در برابر httpOnly cookie — معامله‌ها؟**</summary>

`localStorage`: جاوااسکریپت می‌تواند بخواند → اگر **XSS** رخ دهد، توکن دزدیده می‌شود؛ اما در برابرِ CSRF مصون است (مرورگر خودکار نمی‌فرستد). `HttpOnly` cookie: جاوااسکریپت دسترسی ندارد → در برابرِ XSS-سرقت مقاوم؛ اما چون خودکار فرستاده می‌شود نیاز به دفاعِ **CSRF** (SameSite + token) دارد. جمع‌بندی: httpOnly + Secure + SameSite + CSRF token معمولاً امن‌تر است؛ بسته به مدلِ تهدید انتخاب می‌شود.

</details>

<details><summary>**سه بخش JWT چیست و امنیتش از کجا می‌آید؟**</summary>

`header.payload.signature`. header الگوریتم را می‌گوید، payload claimها را (sub، role، exp، ...)، signature امضای HMAC یا RSA روی دو بخشِ اول است. header و payload فقط **base64url-encode** شده‌اند (نه encrypt) — قابلِ خواندن برای همه. امنیت فقط از **signature** می‌آید: هر دستکاری در payload امضا را می‌شکند.

</details>

<details><summary>**چرا access token عمرِ کوتاه و refresh token عمرِ بلند دارد؟**</summary>

چون JWT را نمی‌توان فوری ابطال کرد، عمرِ کوتاهِ access token پنجره‌ی سوءاستفاده در صورتِ سرقت را کم می‌کند. refresh token (طولانی، امن‌ذخیره، قابلِ ابطال در DB) اجازه می‌دهد بدونِ login دوباره، access token تازه بگیریم. با **rotation** هر refresh یک‌بارمصرف می‌شود تا سرقت قابلِ تشخیص شود.

</details>

<details><summary>**حمله‌ی `alg: none` و algorithm confusion چیست؟**</summary>

`alg:none`: مهاجم فیلدِ الگوریتم را `none` می‌گذارد تا سرور بدونِ بررسیِ امضا قبول کند. algorithm confusion: سرور `RS256` انتظار دارد ولی مهاجم با کلیدِ **عمومی** (که معلوم است) به‌عنوانِ secretِ `HS256` توکنِ معتبر می‌سازد. دفاع: الگوریتم را در کد **whitelist** کن و هرگز به `alg`ِ داخلِ توکن اعتماد نکن.

</details>

<details><summary>**OAuth2 و OpenID Connect چه فرقی دارند؟**</summary>

OAuth2 چارچوبِ **authorization** است (دادنِ دسترسیِ محدود به منابع — «اپ اجازه دارد فایل‌هایم را بخواند»). OIDC لایه‌ای روی OAuth2 برای **authentication** است که `id_token` (یک JWT با هویتِ کاربر) اضافه می‌کند — «این کاربر کیست». «ورود با گوگل» در واقع OIDC است.

</details>

<details><summary>**در authorization code flow چرا code می‌دهند نه مستقیم access token؟**</summary>

code از مرورگرِ کاربر (کانالِ front، قابلِ افشا در URL/history) عبور می‌کند، اما تبدیلِ code به token در یک درخواستِ **server-to-server** با `client_secret` انجام می‌شود. پس حتی اگر کسی code را برباید، بدونِ `client_secret` بی‌فایده است. در کلاینت‌های بدونِ secret (SPA/موبایل) همین نقش را **PKCE** بازی می‌کند.

</details>

<details><summary>**چرا به‌جای OAuth، رمزِ گوگل کاربر را مستقیم نگیریم؟**</summary>

دادنِ رمز یعنی اپ دسترسیِ کامل و دائمی پیدا می‌کند، نمی‌توان scope را محدود کرد، نمی‌توان دسترسی را بدونِ تغییرِ رمز ابطال کرد، و نشتِ اپ یعنی نشتِ رمزِ اصلی. OAuth توکنِ محدود-به-scope و قابلِ-ابطال می‌دهد بی‌آنکه رمز فاش شود.

</details>

### Authorization

<details><summary>**فرق Authentication و Authorization؟**</summary>

Authentication = «تو کی هستی؟» (اثباتِ هویت؛ شکست → `401`). Authorization = «اجازه داری این کار را بکنی؟» (بررسیِ مجوز؛ شکست → `403`). اول authN، بعد authZ. شناختنِ کاربر به‌معنیِ مجاز بودنش نیست.

</details>

<details><summary>**RBAC چیست و چه وقت ABAC بهتر است؟**</summary>

RBAC: `user → role → permissions`؛ ساده و رایج، اما برای قوانینِ ریز/زمینه‌ای ضعیف است. ABAC بر اساسِ attributeها و context تصمیم می‌گیرد (مثلاً «فقط مالکِ منبع»، «فقط در ساعتِ کاری»، «از این کشور»). وقتی منطقِ دسترسی فراتر از نقشِ ثابت می‌رود، ABAC انعطافِ لازم را می‌دهد.

</details>

<details><summary>**IDOR چیست و چطور جلویش را می‌گیری؟**</summary>

Insecure Direct Object Reference: کاربر شناسه‌ی منبع را در URL عوض می‌کند (`/orders/123` → `/124`) و به منبعِ دیگران دسترسی پیدا می‌کند، چون کد فقط «لاگین هستی؟» را چک کرده نه «این منبع مالِ توست؟». دفاع: همیشه **مالکیت/مجوزِ سطحِ شیء** را سمتِ سرور چک کن؛ از شناسه‌های قابلِ‌حدس‌نزدنی (UUID) هم می‌توان کمک گرفت ولی جایگزینِ چکِ authZ نیست.

</details>

<details><summary>**چرا authorization باید سمتِ سرور باشد؟**</summary>

هر چیزی سمتِ کلاینت (مخفی‌کردنِ دکمه، disable کردنِ فیلد) صرفاً UX است و با ابزارهایی مثلِ curl/Postman دور زده می‌شود. تصمیمِ نهاییِ «اجازه دارد یا نه» باید روی سرور و روی *هر* درخواست گرفته شود.

</details>

### Cryptography

<details><summary>**چرا رمزِ عبور را hash می‌کنیم نه encrypt؟**</summary>

encryption برگشت‌پذیر است؛ اگر کلید لو برود همه‌ی رمزها فاش می‌شوند. رمزِ عبور هیچ‌وقت لازم نیست برگردانده شود — فقط باید *تطبیق* داده شود. پس hash یک‌طرفه به‌کار می‌بریم: حتی با نشتِ کاملِ DB، مهاجم رمزِ خام را ندارد.

</details>

<details><summary>**چرا MD5/SHA256 برای رمزِ عبور بد است؟**</summary>

این‌ها برای رمزِ عبور **بیش از حد سریع‌اند**؛ GPU میلیاردها hash در ثانیه می‌زند و rainbow tableها از پیش ساخته شده‌اند. رمزِ عبور به الگوریتمِ **عمدی‌کُند و قابل‌تنظیم** نیاز دارد: bcrypt، scrypt یا argon2id، که brute-force را به چند هزار در ثانیه می‌رسانند.

</details>

<details><summary>**Salt چیست و چه مشکلی را حل می‌کند؟**</summary>

مقدارِ تصادفیِ یکتا per-password که قبل از hash اضافه می‌شود. باعث می‌شود رمزهای یکسان hashهای متفاوت بگیرند، rainbow tableهای از پیش‌ساخته بی‌اثر شوند، و مهاجم نتواند یک حمله را روی همه‌ی کاربران هم‌زمان اجرا کند. (bcrypt salt را خودش می‌سازد و در خروجی embed می‌کند.)

</details>

<details><summary>**Salt در برابر Pepper؟**</summary>

Salt تصادفی، per-user و **کنارِ hash در DB** ذخیره می‌شود (مخفی نیست؛ هدفش جلوگیری از rainbow table است). Pepper یک secretِ سراسری است که **جدا از DB** (در env/HSM) نگه داشته می‌شود؛ اگر فقط DB لو برود، بدونِ pepper شکستن سخت‌تر است.

</details>

<details><summary>**Symmetric در برابر asymmetric — هرکدام کِی؟**</summary>

Symmetric (AES): یک کلیدِ مشترک، سریع، برای رمزِ حجمِ زیادِ داده (دیسک، body در TLS). Asymmetric (RSA/ECC): جفتِ public/private، کند، برای توزیعِ کلید و امضای دیجیتال. در عمل **hybrid**: asymmetric یک session keyِ متقارن را امن مبادله می‌کند، بعد همه‌ی داده با AES رمز می‌شود.

</details>

<details><summary>**امضای دیجیتال چطور کار می‌کند؟**</summary>

با کلیدِ **خصوصی** روی hashِ پیام امضا می‌زنی؛ هرکس با کلیدِ **عمومی** verify می‌کند. این authenticity (واقعاً از فرستنده است) و integrity (دستکاری نشده) می‌دهد، اما **محرمانگی نه**. برعکسِ رمزنگاری که با کلیدِ عمومی قفل و با خصوصی باز می‌شود.

</details>

<details><summary>**Encryption at rest در برابر in transit؟**</summary>

In transit = داده در حالِ انتقالِ شبکه، با **TLS** محافظت می‌شود. At rest = داده‌ی ذخیره‌شده روی دیسک/DB، با AES (full-disk یا field-level) رمز می‌شود. هر دو لازم‌اند؛ یکی جای دیگری را نمی‌گیرد.

</details>

### Web Security — SQLi / XSS / CSRF

<details><summary>**SQL Injection چطور رخ می‌دهد و چطور جلویش را می‌گیری؟**</summary>

وقتی ورودیِ کاربر مستقیم درونِ رشته‌ی کوئری قرار می‌گیرد، داده می‌تواند به‌عنوانِ **کد** تفسیر شود (`' OR '1'='1`، `; DROP TABLE`). دفاعِ اصلی **parameterized queries / prepared statements** است: داده در placeholder جدا می‌رود و موتورِ DB همیشه آن را داده می‌بیند نه دستور. مکمل‌ها: ORM، least-privilege برای کاربرِ DB، اعتبارسنجیِ ورودی، پنهان‌کردنِ خطای خام.

</details>

<details><summary>**چرا parameterized query دقیقاً امن است؟**</summary>

چون ساختارِ کوئری (با placeholderها) **جدا از** داده به موتورِ DB compile می‌شود؛ سپس مقادیر bind می‌شوند. داده هیچ‌وقت بخشی از متنِ SQL parse نمی‌شود، پس نمی‌تواند ساختارِ دستور را تغییر دهد — صرفِ escape کردنِ دستی این تضمین را نمی‌دهد.

</details>

<details><summary>**فرق Stored و Reflected XSS؟**</summary>

Stored: payload در سرور **ذخیره** می‌شود (کامنت، پروفایل) و به هر بیننده سرو می‌شود — تأثیرِ گسترده. Reflected: payload در URL/درخواست است و فقط در پاسخِ بلافاصله به **همان قربانیِ کلیک‌کننده** بازتاب می‌شود. (نوعِ سوم DOM-based است که کاملاً سمتِ کلاینت رخ می‌دهد.)

</details>

<details><summary>**XSS را چطور دفع می‌کنی؟**</summary>

۱) **Output encoding/contextual escaping** هنگامِ رندر (HTML/attribute/JS/URL context) — فریم‌ورک‌های مدرن autoescape دارند. ۲) **CSP** برای محدودکردنِ منبعِ اسکریپت‌ها. ۳) کوکی `HttpOnly` تا سرقتِ session سخت شود. ۴) برای HTMLِ کاربرپسند از sanitizer مثلِ DOMPurify استفاده کن، نه escape دستی.

</details>

<details><summary>**CSP چیست و چه چیزی را حل می‌کند؟**</summary>

Content-Security-Policy هدری است که به مرورگر می‌گوید منابع (script/style/img/...) از کجا مجازند. با `script-src 'self'` اسکریپتِ inline و دامنه‌های ناشناس بلاک می‌شوند → حتی اگر مهاجم بتواند payload تزریق کند، مرورگر اجرایش نمی‌کند. یک لایه‌ی دفاعیِ مکمل برای XSS است، نه جایگزینِ escaping.

</details>

<details><summary>**CSRF چطور کار می‌کند و چطور دفاع می‌کنی؟**</summary>

سایتِ مهاجم مرورگرِ قربانیِ لاگین‌کرده را وادار به ارسالِ درخواستِ state-changing به سایتِ هدف می‌کند؛ مرورگر کوکیِ session را **خودکار** می‌چسباند و سرور آن را معتبر می‌بیند. دفاع: **`SameSite` cookie** (Lax/Strict)، **CSRF token** غیرقابل‌حدس در فرم، و بررسیِ `Origin`/`Referer`.

</details>

<details><summary>**چرا API با `Authorization: Bearer` در برابر CSRF مصون است؟**</summary>

CSRF به credentialی متکی است که مرورگر **خودکار** ضمیمه می‌کند (کوکی). توکنِ Bearer را باید جاوااسکریپتِ خودِ اپ صریحاً در هدر بگذارد؛ مرورگر آن را به‌طورِ خودکار به درخواست‌های cross-site اضافه نمی‌کند، پس سایتِ مهاجم نمی‌تواند درخواستِ معتبر بسازد.

</details>

<details><summary>**چرا اگر XSS داشته باشی، CSRF token نجاتت نمی‌دهد؟**</summary>

CSRF token بر این فرض است که مهاجم نمی‌تواند token را بخواند. اما XSS یعنی کدِ مهاجم درونِ origin اجرا می‌شود و می‌تواند token را از DOM/cookie بخواند و درخواستِ معتبر بسازد. پس XSS اولویتِ بالاتری برای رفع دارد؛ CSRF defense را دور می‌زند.

</details>

<details><summary>**SameSite=Lax در برابر Strict؟**</summary>

`Strict`: کوکی در هیچ درخواستِ cross-site فرستاده نمی‌شود — حتی وقتی کاربر از سایتِ دیگری روی لینک کلیک می‌کند (ممکن است UX را خراب کند، کاربر «لاگ‌اوت» به‌نظر برسد). `Lax`: کوکی در ناوبریِ top-level GET (کلیک روی لینک) فرستاده می‌شود ولی نه در POSTهای cross-site یا درخواست‌های فرعی — تعادلِ خوب، و پیش‌فرضِ مرورگرهای مدرن.

</details>

### Headers & General

<details><summary>**HSTS چه می‌کند و چرا preload مهم است؟**</summary>

`Strict-Transport-Security` به مرورگر می‌گوید این دامنه را فقط با HTTPS باز کن، حتی اگر کاربر `http://` تایپ کند → جلوگیری از SSL-stripping. اولین بازدید هنوز آسیب‌پذیر است؛ **preload** دامنه را در فهرستِ سرسختِ مرورگرها قرار می‌دهد تا حتی بازدیدِ اول هم HTTPS اجباری باشد.

</details>

<details><summary>**`X-Content-Type-Options: nosniff` چه چیزی را می‌بندد؟**</summary>

جلوی **MIME-sniffing** مرورگر را می‌گیرد؛ یعنی مرورگر مجبور است به `Content-Type` اعلام‌شده احترام بگذارد و فایلی که `text/plain` است را به‌عنوانِ HTML/JS اجرا نکند. جلوی یک مسیرِ XSS از طریقِ سوءتعبیرِ نوعِ محتوا را می‌بندد.

</details>

<details><summary>**رمزِ عبورها را چطور به‌صورتِ امن ذخیره می‌کنی (طراحیِ کامل)؟**</summary>

هر رمز با **argon2id** (یا bcrypt با cost مناسب) hash می‌شود، با **salt یکتا per-user** (که الگوریتم خودش embed می‌کند)، اختیاراً با **pepperِ** سراسری در env/HSM. work factor متناسب با سخت‌افزار تنظیم و دوره‌ای بالا برده می‌شود. مقایسه با constant-time. هرگز رمزِ خام لاگ یا ذخیره نمی‌شود؛ سیاستِ rate-limit و MFA هم لایه می‌شود.

</details>

### Scenario

<details><summary>**کاربر گزارش می‌دهد حسابش بدونِ او عمل انجام داده — تحقیقت را قدم‌به‌قدم بگو.**</summary>

۱) دامنه: فقط همین کاربر یا چندتا؟ (نشتِ گسترده در برابرِ هدفمند). ۲) لاگ‌های auth را بررسی کن: login از IP/دستگاه/جغرافیای ناآشنا؟ زمانِ عمل؟ ۳) بردارِ احتمالی: **CSRF** (آیا endpointِ مشکوک SameSite/token دارد؟)، **XSS** (سرقتِ session — آیا کوکی HttpOnly است؟ CSP فعال؟)، **session fixation/hijack**، رمزِ ضعیف/نشت‌شده، یا **IDOR** (آیا واقعاً حسابِ خودش بوده؟). ۴) مهار: ابطالِ همه‌ی session/token کاربر، اجبارِ ریست رمز، فعال‌کردن MFA. ۵) ریشه‌یابی: درخواستِ مهاجم را در لاگ بازسازی کن، هدرها/Origin/Referer را ببین. ۶) رفع و پیشگیری: وصله‌ی همان بردار، افزودنِ alerting، بازبینیِ کنترل‌های مشابه.

</details>

<details><summary>**تستِ نفوذ یک endpoint با `' OR '1'='1` همه‌ی رکوردها را برگرداند. چه کار می‌کنی؟**</summary>

این SQLi تأییدشده است. فوری: ورودی را به **parameterized query** منتقل کن (نه escape دستی). بعد: کلِ کدبیس را برای الگوی string-concat در کوئری‌ها audit کن، کاربرِ DB را least-privilege کن، WAF را به‌عنوانِ لایه‌ی موقت فعال کن، لاگ‌ها را برای بهره‌برداریِ قبلی بررسی کن، و اگر داده‌ی حساس افشا شده فرایندِ incident را آغاز کن.

</details>

<details><summary>**باید بینِ JWT و session برای یک API جدید انتخاب کنی. چطور تصمیم می‌گیری؟**</summary>

سؤال‌ها: آیا چند سرویس/دامنه باید توکن را مستقل verify کنند؟ (به نفعِ JWT). آیا logout/ابطالِ فوری حیاتی است؟ (به نفعِ session یا JWT + blocklist). مقیاسِ افقی و بی‌حالتی مهم است؟ (JWT). کلاینت مرورگر است یا سرویسِ ماشین؟ برای مرورگر معمولاً session-cookie یا JWT در httpOnly cookie + CSRF defense؛ برای سرویس‌به‌سرویس، JWT/Bearer. غالباً ترکیب: refresh token مثلِ session مدیریت می‌شود، access token کوتاه‌عمرِ JWT.

</details>

<details><summary>**یک کتابخانه‌ی front-end می‌خواهد HTMLِ کاربر را رندر کند (rich text). چطور امن نگهش می‌داری؟**</summary>

escape ساده کافی نیست چون HTML باید واقعی رندر شود. از یک **HTML sanitizer** مثلِ DOMPurify با allowlistِ سخت‌گیرانه‌ی تگ/attribute استفاده کن (حذفِ `<script>`، `on*` handlerها، `javascript:` URLها). علاوه بر آن **CSP** بگذار و کوکی را HttpOnly کن تا حتی در صورتِ bypass، تأثیر محدود شود. هرگز ورودیِ خام را در `innerHTML` نگذار.

</details>
