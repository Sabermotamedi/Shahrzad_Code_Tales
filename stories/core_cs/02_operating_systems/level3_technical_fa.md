# سطح ۳ — توضیح فنی: Operating Systems & Linux ⚙️

> دیگر قصه‌ای در کار نیست. این سند مرجع فنی است — همان چیزهایی که در مصاحبه و کار واقعی لازم داری. هر بخش با یک اشاره‌ی کوتاه به معادلِ قصه‌ای‌اش شروع می‌شود تا ذهنت وصل بماند.

---

## ۱. ساختار سیستم‌عامل (User Mode / Kernel Mode)

> سرآشپز قدرتِ ویژه دارد، آشپزها محدودند.

سیستم‌عامل واسطه‌ی بین برنامه‌ها و سخت‌افزار است و منابع (CPU، حافظه، I/O) را مدیریت می‌کند. هسته‌ی آن (**Kernel**) در **Kernel Mode** اجرا می‌شود (دسترسی کامل به سخت‌افزار)، و برنامه‌های کاربر در **User Mode** (دسترسی محدود).

- عبور از User Mode به Kernel Mode فقط از سه راه: **System Call**، **Interrupt** (سخت‌افزار)، **Exception/Trap** (مثل page fault یا تقسیم بر صفر).
- این تفکیک با پشتیبانیِ سخت‌افزار (rings در x86: ring 0 = kernel، ring 3 = user) اجرا می‌شود.

| نوع Kernel | توضیح | مثال |
|---|---|---|
| Monolithic | همه‌ی سرویس‌ها در یک فضای kernel | Linux |
| Microkernel | حداقل در kernel، بقیه در user space | Minix، QNX |
| Hybrid | ترکیبی | Windows NT، macOS (XNU) |

---

## ۲. پروسه‌ها (Processes)

> هر آشپز = یک پروسه با میزِ جداگانه.

پروسه = یک برنامه‌ی در حالِ اجرا، با فضای آدرسِ مستقل. هسته برای هر پروسه یک **PCB** (Process Control Block) نگه می‌دارد: PID، حالت، رجیسترها، page table، file descriptorها و...

### چرخه‌ی حیات

```
new → ready ⇄ running → terminated
              ↓   ↑
            (I/O) waiting/blocked
```

### ساختِ پروسه: fork + exec

- `fork()` یک کپیِ تقریباً کامل از پروسه‌ی والد می‌سازد (با **Copy-on-Write**: صفحات تا وقتی یکی بنویسد مشترک می‌مانند).
- `exec()` تصویرِ برنامه‌ی فعلی را با برنامه‌ی جدید جایگزین می‌کند.
- الگوی کلاسیک shell: `fork()` بعد در فرزند `exec()`، و والد `wait()` می‌کند.

### حالت‌های خاص

- **Zombie**: فرزند تمام شده ولی والد هنوز `wait()` نکرده تا exit code را بخواند. در `ps` با `Z` و وضعیت `defunct`. خطرناک نیست مگر زیاد جمع شوند (نشت PID).
- **Orphan**: والد قبل از فرزند مرده؛ فرزند را `init`/`systemd` (PID 1) به فرزندی می‌پذیرد.
- **Daemon**: پروسه‌ی پس‌زمینه‌ی بلندمدت، بی‌ارتباط با terminal.

> نکته‌ی مصاحبه: فرق zombie و orphan؟ Zombie = مرده ولی هنوز در جدول پروسه (منتظر reap)؛ Orphan = زنده ولی والدش مرده (re-parent به PID 1).

### IPC (ارتباط بین پروسه‌ها)

چون حافظه‌ها جداست، برای ارتباط به کمکِ kernel نیاز است: **Pipes** (`|`)، **Named pipes (FIFO)**، **Sockets**، **Shared memory** (سریع‌ترین)، **Message queues**، **Signals**.

```bash
kill -TERM 1234     # درخواستِ مودبانه‌ی خاتمه (قابل گرفتن/نادیده‌گرفتن)
kill -KILL 1234     # = kill -9 ؛ غیرقابل‌گرفتن، فوری
kill -HUP 1234      # معمولاً = «config را دوباره بخوان»
```

---

## ۳. نخ‌ها (Threads) و هم‌روندی

> چند آشپز روی یک ایستگاهِ مشترک.

نخ = کوچک‌ترین واحدِ زمان‌بندی. نخ‌های یک پروسه **فضای آدرس، فایل‌ها و heap را مشترک** دارند، اما هر کدام **stack و رجیسترِ** خودش را دارد.

| | Process | Thread |
|---|---|---|
| فضای آدرس | جدا | مشترک |
| هزینه‌ی ساخت/سوییچ | بالا | پایین |
| ایزولاسیون خطا | قوی | ضعیف (یک thread خراب = کل پروسه) |
| ارتباط | IPC (kernel) | حافظه‌ی مشترک (سریع) |

### Concurrency در برابر Parallelism

- **Concurrency**: چند کار «در جریان»‌اند (با تعویضِ سریع روی یک هسته هم ممکن است).
- **Parallelism**: چند کار **واقعاً هم‌زمان** روی چند هسته.

### مشکلات هم‌زمانی و راه‌حل‌ها

| مشکل | توضیح | راه‌حل |
|---|---|---|
| Race condition | نتیجه به ترتیبِ اجرای نخ‌ها وابسته است | همگام‌سازی |
| Critical section | بخشی که فقط یک نخ هم‌زمان باید واردش شود | Mutex / Lock |
| Deadlock | دو نخ هرکدام منتظرِ قفلِ دیگری | جلوگیری از یکی از ۴ شرطِ Coffman |
| Starvation | نخی هیچ‌وقت نوبت نمی‌گیرد | اولویتِ عادلانه (aging) |

ابزارهای همگام‌سازی: **Mutex** (قفلِ انحصاری)، **Semaphore** (شمارنده، n دسترسی هم‌زمان)، **Spinlock** (در حلقه منتظر می‌ماند — برای انتظارهای خیلی کوتاه در kernel)، **Condition Variable**، **Atomic operations**.

> نکته‌ی مصاحبه: چهار شرطِ deadlock (Coffman): Mutual Exclusion، Hold-and-Wait، No Preemption، Circular Wait. شکستنِ هر کدام، deadlock را ناممکن می‌کند.

---

## ۴. زمان‌بندی (CPU Scheduling)

> سرآشپز نوبتِ اجاق می‌دهد.

Scheduler تصمیم می‌گیرد کدام پروسه‌ی `ready` روی CPU برود. تعویض = **Context Switch** (ذخیره‌ی رجیسترها/حالت پروسه‌ی قبلی و بارگذاریِ بعدی) — هزینه دارد، پس باید کم باشد.

| الگوریتم | منطق | اشکال |
|---|---|---|
| FCFS | اول‌آمده اول‌سرویس | Convoy effect (یک کارِ طولانی همه را معطل می‌کند) |
| SJF / SRTF | کوتاه‌ترین کار اول | Starvation کارهای بلند؛ طولِ کار را باید حدس زد |
| Round Robin | هر کس یک time slice، نوبتی | حساس به اندازه‌ی quantum |
| Priority | اولویت‌دار اول | Starvation (راه‌حل: aging) |
| MLFQ | چند صفِ چنداولویتی، تطبیقی | پیچیده؛ پایه‌ی schedulerهای واقعی |

- **Preemptive** (هسته می‌تواند پروسه‌ی در حال اجرا را کنار بزند) در برابر **Non-preemptive**.
- Linux سال‌ها از **CFS** (Completely Fair Scheduler) استفاده می‌کرد که به جای time slice ثابت، «زمانِ مجازیِ» منصفانه بین پروسه‌ها توزیع می‌کند. از کرنل 6.6 به بعد، scheduler پیش‌فرض **EEVDF** (Earliest Eligible Virtual Deadline First) است که بر همان ایده‌ی انصافِ CFS بنا شده ولی deadlineِ مجازی هم در نظر می‌گیرد. اولویت با **nice** (۲۰- تا ۱۹) تنظیم می‌شود.

```bash
nice -n 10 ./job.sh        # شروع با اولویتِ پایین‌تر
renice -n -5 -p 1234       # تغییر اولویتِ پروسه‌ی در حال اجرا (منفی نیاز به root)
chrt -f 99 ./realtime_app  # زمان‌بندیِ real-time (FIFO، اولویت ۹۹)
taskset -c 0,1 ./app       # محدود کردن به هسته‌های ۰ و ۱ (CPU affinity)
```

> نکته‌ی مصاحبه: **Load average** تعدادِ متوسطِ پروسه‌های `runnable` **به‌علاوه‌ی** `uninterruptible (D-state)` در ۱/۵/۱۵ دقیقه است — نه درصدِ CPU. load بالا با CPU پایین معمولاً یعنی پروسه‌ها منتظرِ I/O (دیسک/شبکه) قفل شده‌اند.

---

## ۵. مدیریت حافظه (Memory Management)

> میزِ کار، تقسیم‌شده با دفترِ نقشه.

### فضای آدرسِ یک پروسه

```
آدرس بالا  ┌──────────────┐
           │   Stack      │ ↓ (متغیرهای محلی، فریم‌های تابع؛ رشد به پایین)
           │   ...        │
           │   ↑ Heap     │ (malloc/new؛ رشد به بالا)
           │   BSS        │ (متغیرهای global بدون مقدار)
           │   Data       │ (متغیرهای global با مقدار)
آدرس پایین │   Text/Code  │ (دستورالعمل‌ها، read-only)
           └──────────────┘
```

### Paging و آدرس مجازی

- حافظه به **Page** (مجازی) و **Frame** (فیزیکی) تقسیم می‌شود؛ معمولاً ۴KB.
- **Page Table** نگاشتِ virtual→physical را نگه می‌دارد؛ هر پروسه page table خودش را دارد → ایزولاسیون.
- **MMU** (سخت‌افزار) این ترجمه را در لحظه انجام می‌دهد؛ **TLB** کشِ ترجمه‌های اخیر است (سریع‌کردنِ کار).
- مزایا: ایزولاسیون، اشتراکِ صفحه (مثلاً کتابخانه‌ها)، حذفِ external fragmentation.

### تخصیص و مشکلات

- **Fragmentation**: داخلی (هدررفت داخلِ یک page) و خارجی (سوراخ‌های پراکنده — paging حلش می‌کند).
- **Memory leak**: حافظه‌ی گرفته‌شده آزاد نمی‌شود (`malloc` بدون `free`).
- **OOM Killer** در Linux: وقتی حافظه واقعاً تمام شود، kernel پروسه‌ای را با امتیازِ `oom_score` بالا می‌کشد.

---

## ۶. حافظه‌ی مجازی (Virtual Memory و Swap)

> انباریِ پشتی برای وقتی میز پر می‌شود.

Virtual Memory به هر پروسه توهمِ یک فضای آدرسِ بزرگ و پیوسته می‌دهد، حتی بزرگ‌تر از RAM فیزیکی.

- **Demand paging**: صفحات فقط وقتی واقعاً لازم شوند به RAM آورده می‌شوند.
- **Page Fault**: دسترسی به صفحه‌ای که در RAM نیست → trap به kernel → آوردنِ صفحه از دیسک.
  - **Minor fault**: صفحه در حافظه هست ولی در page table این پروسه map نشده (ارزان).
  - **Major fault**: باید از دیسک خوانده شود (گران).
- **Swapping**: انتقالِ صفحاتِ کم‌استفاده به دیسک (پارتیشن/فایلِ swap).
- **Thrashing**: وقتی RAM آن‌قدر کم است که سیستم بیشترِ وقتش را صرفِ page in/out می‌کند تا کارِ مفید → throughput سقوط می‌کند.
- الگوریتم‌های جایگزینی صفحه: **LRU** (تقریبی، با clock/second-chance)، **FIFO** (با Bélády's anomaly)، **Optimal** (نظری).

```bash
free -h                # RAM و swap
cat /proc/meminfo      # جزئیات کامل حافظه
vmstat 1               # si/so بالا = swapping فعال
swapon --show          # دستگاه‌های swap
```

> نکته‌ی مصاحبه: چرا swap با SSD/RAM زیاد هنوز مفید است؟ صفحاتِ مرده (anonymous pages که هیچ‌وقت لمس نمی‌شوند) را از RAM بیرون می‌برد تا برای page cache جا باز شود. اما swap زیاد در دیتابیس‌ها = تأخیرِ کشنده.

---

## ۷. System Calls

> آشپز قبل از دست‌زدن به منبعِ مشترک، اجازه می‌گیرد.

System call تنها رابطِ برنامه‌ی user با kernel است. کتابخانه‌ی C (glibc) wrapperهای راحت دورِ آن‌ها دارد.

| دسته | فراخوانی‌ها |
|---|---|
| فایل | `open`, `read`, `write`, `close`, `lseek`, `stat` |
| پروسه | `fork`, `execve`, `wait`, `exit`, `getpid`, `kill` |
| حافظه | `mmap`, `munmap`, `brk`, `mprotect` |
| شبکه | `socket`, `bind`, `listen`, `accept`, `connect`, `send`, `recv` |
| همگام‌سازی/IPC | `pipe`, `futex`, `shmget`, `semop` |

- **File Descriptor**: عددِ صحیحی که kernel به یک فایل/socket/pipe باز می‌دهد. ۰=stdin، ۱=stdout، ۲=stderr.
- هزینه‌ی هر system call = یک mode switch (نه لزوماً context switch). برنامه‌های پرسرعت آن‌ها را batch می‌کنند (مثلاً `io_uring`, `epoll`).

```bash
strace -c ls           # شمارشِ system callها و زمانشان
strace -f -e trace=openat,read ./app   # ردیابیِ فراخوانی‌های خاص
```

---

## ۸. سیستم فایل لینوکس (Linux File System)

> کشوهای مرتبِ دستور پخت، با یک ریشه‌ی واحد.

Linux یک درختِ واحد از `/` دارد (Unified hierarchy)؛ دیسک‌ها و دستگاه‌ها روی نقاطی **mount** می‌شوند.

### سلسله‌مراتب (FHS)

| مسیر | محتوا |
|---|---|
| `/` | ریشه |
| `/etc` | فایل‌های تنظیمات سیستم (متنی) |
| `/home` | پوشه‌ی کاربران (`/home/user`) |
| `/root` | خانه‌ی کاربرِ root |
| `/var` | داده‌ی متغیر: `/var/log`، `/var/lib`، صف‌ها، کش |
| `/usr` | برنامه‌ها و کتابخانه‌های نصب‌شده (`/usr/bin`, `/usr/lib`) |
| `/bin`, `/sbin` | باینری‌های ضروری (اغلب symlink به `/usr/...`) |
| `/tmp` | فایل موقت (پاک‌شونده) |
| `/dev` | فایل‌های دستگاه (`/dev/sda`, `/dev/null`) |
| `/proc` | فایل‌سیستمِ مجازی: وضعیت kernel/پروسه‌ها |
| `/sys` | فایل‌سیستمِ مجازی: دستگاه‌ها و درایورها |
| `/opt` | نرم‌افزارهای جانبی |
| `/boot` | kernel و bootloader |

### مفاهیم کلیدی

- **Inode**: ساختاری که متادیتای فایل (مالک، دسترسی، اندازه، بلوک‌های داده) را نگه می‌دارد — **اما نه نام را**. نام در directory entry است که به inode اشاره می‌کند.
- **Hard link**: نامِ دیگری به همان inode (شمارنده‌ی link). فایل تا آخرین link حذف نشود نمی‌میرد.
- **Soft/Symbolic link**: فایلی که مسیرِ یک فایلِ دیگر را نگه می‌دارد (می‌تواند بشکند).
- **Everything is a file**: دستگاه‌ها، socketها، pipeها همه از طریق file descriptor.
- فایل‌سیستم‌های رایج: **ext4**، **XFS**، **Btrfs**، **ZFS**.

```bash
ls -lai            # نمایش inode، link count، permissions
df -h              # فضای پارتیشن‌ها (mount pointها)
du -sh /var/log    # حجمِ واقعیِ یک پوشه
ln file hard       # hard link
ln -s file soft    # symbolic link
mount | column -t  # چه چیزی کجا mount شده
```

---

## ۹. دسترسی‌ها (Permissions)

> کلیدِ کشوها: چه کسی بخواند/بنویسد/اجرا کند.

```
-rwxr-x---  owner=sam group=chefs
 │└┬┘└┬┘└┬┘
 │ u  g  o   →  r=4  w=2  x=1
```

- سه کلاس: **u**ser / **g**roup / **o**thers. هر کلاس rwx.
- روی **فایل**: `x` = اجرا. روی **پوشه**: `x` = ورود/پیمایش، `r` = لیست‌کردن، `w` = ساخت/حذفِ فایل داخلش.

### بیت‌های خاص

| بیت | روی فایل اجرایی | روی پوشه |
|---|---|---|
| **setuid** (4xxx) | با هویتِ مالکِ فایل اجرا می‌شود (مثل `passwd`) | — |
| **setgid** (2xxx) | با گروهِ فایل اجرا می‌شود | فایل‌های جدید گروهِ پوشه را می‌گیرند |
| **sticky** (1xxx) | — | فقط مالک می‌تواند فایلِ خودش را حذف کند (مثل `/tmp`) |

```bash
chmod 750 f          # rwxr-x---
chmod u+x,o-r f      # نمادین
chmod 4755 /usr/bin/prog   # setuid
chown sam:chefs f    # تغییر مالک و گروه
chgrp chefs f        # فقط گروه
umask 022            # دسترسیِ پیش‌فرضِ فایل‌های جدید (از مجوزِ کامل کسر می‌زند)
getfacl f; setfacl -m u:ali:rw f   # ACLهای دقیق‌تر فراتر از سه کلاس
```

- کاربران در `/etc/passwd`، گروه‌ها در `/etc/group`، رمزها (hash) در `/etc/shadow`.
- **root** (UID 0) از همه‌ی بررسی‌ها عبور می‌کند. `sudo` اجرای موقتِ یک فرمان با امتیازِ بالا (با ثبت در لاگ).

> نکته‌ی مصاحبه: چرا `passwd` نیاز به setuid دارد؟ کاربرِ عادی باید `/etc/shadow` (مالکش root) را تغییر دهد؛ setuid باعث می‌شود برنامه با هویتِ root اجرا شود تا این یک کارِ خاص ممکن شود.

---

## ۱۰. Shell Scripting و Bash

> چک‌لیستِ خودکارِ آشپزخانه.

Shell هم مفسرِ تعاملی است هم زبانِ اسکریپت. Bash رایج‌ترین است (sh/dash/zsh جایگزین‌ها).

```bash
#!/usr/bin/env bash
set -euo pipefail        # خطایابیِ سخت‌گیرانه: توقف روی خطا، متغیرِ تعریف‌نشده، خطای pipe

NAME="${1:-world}"       # آرگومان اول، با مقدار پیش‌فرض
readonly LOG=/var/log/app.log

for f in *.txt; do
  [[ -f "$f" ]] || continue
  echo "processing $f"
done

if grep -q "ERROR" "$LOG"; then
  echo "found errors" >&2     # نوشتن روی stderr
  exit 1
fi

result=$(date +%F)            # command substitution
count=$(( 2 + 3 ))           # محاسبه‌ی عددی
```

نکات کلیدی:
- **همیشه متغیرها را در `"..."` بگذار** تا word splitting و glob رخ ندهد.
- `[[ ]]` (bash) امن‌تر از `[ ]` (POSIX) است.
- **Exit code**: `0` = موفقیت، غیرصفر = خطا؛ `$?` آخرین کد را دارد.
- **Pipe و redirection**: `cmd1 | cmd2`، `> file` (بازنویسی)، `>> file` (افزودن)، `2>&1` (ادغامِ stderr در stdout).
- **Quoting**: `'...'` تحت‌اللفظی، `"..."` با بسط متغیر.

```bash
# نمونه‌ی اتوماسیون: پاک‌سازیِ لاگ‌های قدیمی‌تر از ۷ روز، هر شب با cron
find /var/log/app -name '*.log' -mtime +7 -delete
# در crontab:  0 2 * * *  /opt/scripts/cleanup.sh
```

---

## ۱۱. متغیرهای محیطی (Environment Variables)

> تابلوی اعلاناتِ روی دیوار.

جفت‌های `KEY=VALUE` که محیطِ هر پروسه را می‌سازند و به فرزندان **به ارث** می‌رسند (با `fork`/`exec`).

| متغیر | کار |
|---|---|
| `PATH` | پوشه‌هایی که برای یافتنِ دستورات جست‌وجو می‌شوند |
| `HOME` | پوشه‌ی خانه‌ی کاربر |
| `USER`, `LOGNAME` | نامِ کاربر |
| `SHELL` | شِلِ پیش‌فرض |
| `PWD` | دایرکتوریِ فعلی |
| `LANG`, `LC_*` | locale/زبان |
| `LD_LIBRARY_PATH` | مسیرِ جست‌وجوی کتابخانه‌های اشتراکی |

```bash
echo $PATH
export FOO=bar         # تعریف برای پروسه‌ی فعلی و فرزندانش
FOO=bar ./app          # فقط برای همین یک اجرا
env                    # نمایشِ کلِ محیط (یا: printenv)
unset FOO              # حذف
```

- تفاوتِ **shell variable** (فقط در همین شِل) و **environment variable** (با `export`، به فرزندان می‌رسد).
- ماندگاری: `~/.bashrc` (هر شِلِ تعاملی)، `~/.profile`/`~/.bash_profile` (هنگام login)، `/etc/environment` (سیستمی).

> نکته‌ی مصاحبه: چرا secrets را در env var می‌گذارند ولی بعضی آن را ضدالگو می‌دانند؟ راحت و جداازکُد است، اما هر child آن را به ارث می‌برد و در `/proc/<pid>/environ` یا crash dumpها قابل دیدن است — برای secretهای حساس، secret managerها بهترند.

---

## ۱۲. سرویس‌ها و systemd

> ایستگاه‌های همیشه‌روشن.

systemd در بیشترِ توزیع‌های مدرن، سیستمِ init است (**PID 1**) و چرخه‌ی حیاتِ سرویس‌ها، mountها، socketها و timerها را مدیریت می‌کند. واحدِ کار = **unit** (`.service`, `.timer`, `.socket`, `.target`).

```ini
# /etc/systemd/system/web.service
[Unit]
Description=Web API
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
ExecStart=/usr/local/bin/web --port 8080
Restart=on-failure
RestartSec=3
User=www-data
Environment=NODE_ENV=production
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload          # بعد از تغییر unit file
systemctl enable --now web       # فعال‌سازیِ بوت + شروعِ فوری
systemctl status web
systemctl restart web
systemctl list-units --type=service --state=running
journalctl -u web -f --since "10 min ago"   # لاگ‌های متمرکز
systemctl list-timers            # جایگزینِ مدرنِ cron
```

- **Target** جایگزینِ runlevelهای قدیمی است (`multi-user.target` ≈ runlevel 3، `graphical.target` ≈ 5).
- وابستگی‌ها: `After=`/`Before=` ترتیب، `Requires=`/`Wants=` نیازمندی.
- `Restart=on-failure|always` = خود-ترمیمیِ سرویس.

> نکته‌ی مصاحبه: فرق `enable` و `start`؟ `start` همین حالا اجرا می‌کند (تا ریبوت)؛ `enable` symlink می‌سازد تا در بوتِ بعدی خودکار بالا بیاید. `enable --now` هر دو.

---

## 🔬 آزمایشگاه

دستورهای واقعی و قابلِ اجرا. روی یک سیستمِ Linux امتحان کن (برخی `strace`/`top` ممکن است نیاز به نصب یا sudo داشته باشند).

```bash
# === سیستم فایل و دسترسی‌ها ===
ls -la /etc                       # محتوا + permissions + مالک + گروه
ls -lai /tmp                      # همان + شماره‌ی inode
stat /etc/passwd                  # متادیتای کامل یک فایل (inode, ctime, perms)
chmod 640 /tmp/secret.txt         # rw-r----- (مالک بخواند/بنویسد، گروه بخواند)
chmod u+x /tmp/run.sh             # افزودن اجرا برای مالک (نمادین)
chown $USER:$USER /tmp/secret.txt # تغییر مالک و گروه
umask                             # دیدنِ ماسکِ پیش‌فرض
ln -s /etc/hostname /tmp/hn_link  # ساخت symlink و بررسی با ls -l

# === پروسه‌ها و نخ‌ها ===
ps aux | head                     # همه‌ی پروسه‌ها با CPU/MEM
ps -ef --forest | head -30        # درختِ والد-فرزند
ps -eLf | head                    # نمایشِ نخ‌ها (ستون LWP)
pstree -p                         # درختِ پروسه‌ها با PID
pgrep -a sshd                     # یافتنِ PID با نام
top                               # داشبوردِ زنده (q برای خروج)
top -H                            # نمایشِ نخ‌ها به‌جای پروسه‌ها
htop                              # نسخه‌ی تعاملی‌تر (اگر نصب باشد)
uptime                            # load average در ۱/۵/۱۵ دقیقه
nproc                             # تعدادِ هسته‌ها (برای تفسیرِ load)

# === زمان‌بندی و اولویت ===
nice -n 19 sleep 100 &            # اجرا با پایین‌ترین اولویت
renice -n 5 -p $!                 # تغییرِ اولویتِ آخرین job
taskset -cp $$                    # دیدنِ CPU affinity شِلِ فعلی

# === حافظه و حافظه‌ی مجازی ===
free -h                           # RAM و swap
cat /proc/meminfo | head          # جزئیات
vmstat 1 5                        # ۵ نمونه؛ ستون si/so = swap in/out
swapon --show                     # دستگاه‌های swap
cat /proc/$$/status | grep -i vm  # مصرفِ حافظه‌ی شِلِ فعلی

# === System calls ===
strace -c ls /                    # خلاصه‌ی شمارشِ system callها
strace -f -e trace=openat,read,write ls / 2>&1 | head -20
ltrace ls 2>&1 | head             # ردیابیِ فراخوانیِ کتابخانه‌ها (اگر نصب باشد)

# === کاوش در /proc ===
cat /proc/cpuinfo | grep -c processor   # تعدادِ پردازنده‌ها
cat /proc/loadavg                       # load + پروسه‌ی در حال اجرا/کل
ls -l /proc/$$/fd                       # file descriptorهای بازِ شِلِ فعلی
cat /proc/$$/cmdline | tr '\0' ' '; echo
cat /proc/$$/environ | tr '\0' '\n' | head  # محیطِ این پروسه
cat /proc/1/comm                        # نامِ PID 1 (systemd?)

# === متغیرهای محیطی ===
printenv PATH
export DEMO_KEY=hello && env | grep DEMO_KEY
echo "$HOME / $USER / $SHELL"

# === systemd / سرویس‌ها ===
systemctl status                        # وضعیتِ کلیِ سیستم
systemctl list-units --type=service --state=running | head
systemctl status ssh    2>/dev/null || systemctl status sshd
journalctl -p err -b --no-pager | tail  # خطاهای این بوت
systemctl list-timers --no-pager | head

# === شل‌اسکریپتِ کوچک ===
cat > /tmp/demo.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
for i in 1 2 3; do echo "step $i on $(hostname)"; done
EOF
chmod +x /tmp/demo.sh && /tmp/demo.sh
```

---

## ✅ چک‌لیست تسلط

اگر بتوانی به این‌ها جواب بدهی، این فصل را بلدی:

- [ ] فرق User Mode و Kernel Mode و سه راهِ عبور به kernel؟
- [ ] دقیقاً چه اتفاقی در `fork()` + `exec()` می‌افتد؟ Copy-on-Write چیست؟
- [ ] فرق zombie و orphan؟ هر کدام چطور درست می‌شوند و چطور reap می‌شوند؟
- [ ] فرق process و thread در حافظه، هزینه و ایزولاسیون؟
- [ ] Concurrency در برابر Parallelism؟ روی یک‌هسته کدام ممکن است؟
- [ ] چهار شرطِ Coffman برای deadlock و راه‌های شکستنشان؟
- [ ] چرا context switch هزینه دارد؟ CFS چطور انصاف را تضمین می‌کند؟
- [ ] Load average دقیقاً چه چیزی را می‌شمارد و چرا با درصد CPU فرق دارد؟
- [ ] virtual→physical چطور ترجمه می‌شود؟ نقشِ MMU، Page Table و TLB؟
- [ ] Page fault (minor در برابر major)، swapping و thrashing؟
- [ ] فرق hard link و soft link در سطحِ inode؟
- [ ] خروجیِ `-rwxr-x---` را کلمه‌به‌کلمه و عددی (`750`) شرح بده.
- [ ] setuid/setgid/sticky هرکدام چه می‌کنند؟ چرا `passwd` setuid است؟
- [ ] فرق `enable` و `start` در systemd؟ نقشِ PID 1 چیست؟
- [ ] فرق shell variable و environment variable؟ ارث‌بریِ env چطور کار می‌کند؟
- [ ] چرا `set -euo pipefail` و quoting در Bash مهم‌اند؟

---

## 🎯 سوالات مصاحبه

### سیستم فایل و دسترسی‌ها (File System & Permissions)

<details><summary>**مجوزِ `750` روی یک پوشه دقیقاً چه اجازه‌ای می‌دهد؟**</summary>

`750` یعنی owner=`rwx`(7)، group=`r-x`(5)، others=`---`(0). روی **پوشه**: مالک می‌تواند لیست کند، وارد شود و فایل بسازد/حذف کند؛ گروه می‌تواند لیست کند و وارد شود ولی فایل نسازد/نحذفد؛ دیگران اصلاً دسترسی ندارند. توجه: روی پوشه `x` یعنی «اجازه‌ی ورود/پیمایش» و `r` یعنی «اجازه‌ی لیست‌کردنِ نام‌ها» — می‌شود `x` بدون `r` داشت (وارد شد ولی نام‌ها را ندید).

</details>

<details><summary>**فرق hard link و symbolic link چیست؟**</summary>

Hard link نامِ دیگری برای همان inode است؛ link countِ inode را بالا می‌برد و تا آخرین link پاک نشود، داده باقی می‌ماند. نمی‌تواند به فایل‌سیستمِ دیگر یا پوشه اشاره کند. Symbolic link یک فایلِ جداست که فقط **مسیرِ** فایلِ هدف را نگه می‌دارد؛ اگر هدف پاک شود، link «می‌شکند» (dangling). symlink می‌تواند cross-filesystem و به پوشه باشد.

</details>

<details><summary>**Inode چیست و چه چیزی را نگه نمی‌دارد؟**</summary>

Inode ساختاری است که متادیتای فایل را نگه می‌دارد: مالک، گروه، permissions، اندازه، timestampها (ctime/mtime/atime) و اشاره به بلوک‌های داده. نکته: نامِ فایل را نگه **نمی‌دارد** — نام در directory entry است که به شماره‌ی inode اشاره می‌کند. به همین دلیل یک inode می‌تواند چند نام (hard link) داشته باشد.

</details>

<details><summary>**کاربری می‌گوید فایل را پاک کرد ولی فضای دیسک آزاد نشد — چرا؟**</summary>

دو علتِ رایج: (۱) یک پروسه هنوز فایل را **باز** نگه داشته؛ تا آن file descriptor بسته نشود، فضا آزاد نمی‌شود (با `lsof | grep deleted` پیدا می‌شود — راه‌حل: ری‌استارتِ پروسه). (۲) فایل **hard link** دیگری دارد، پس داده تا آخرین link زنده است. همچنین ممکن است `df` و `du` به‌خاطر همین فایل‌های deleted-but-open اختلاف نشان دهند.

</details>

<details><summary>**setuid روی یک باینری یعنی چه و چه ریسکی دارد؟**</summary>

setuid باعث می‌شود برنامه با هویتِ **مالکِ فایل** (نه اجراکننده) اجرا شود. مثلاً `passwd` مالکش root است و setuid دارد تا کاربرِ عادی بتواند `/etc/shadow` را تغییر دهد. ریسک: اگر یک باینریِ setuid-root باگ داشته باشد (buffer overflow و...)، مهاجم می‌تواند به امتیازِ root برسد (privilege escalation). به همین دلیل تعداد باینری‌های setuid را کم نگه می‌دارند.

</details>

### پروسه‌ها، نخ‌ها و زمان‌بندی (Processes, Threads, Scheduling)

<details><summary>**فرق process و thread چیست؟**</summary>

Process یک برنامه‌ی در حالِ اجرا با فضای آدرسِ **مستقل** است. Threadها واحدهای اجرا داخلِ یک process‌اند که **حافظه (heap, code, فایل‌ها) را مشترک** دارند ولی stack و رجیسترِ خودشان را دارند. Thread سبک‌تر است (ساخت/سوییچِ ارزان‌تر، ارتباطِ آسان از طریق حافظه‌ی مشترک) ولی ایزولاسیونِ خطا ندارد — یک thread که crash کند می‌تواند کلِ process را بیندازد.

</details>

<details><summary>**`fork()` چه می‌کند و Copy-on-Write یعنی چه؟**</summary>

`fork()` یک process فرزند می‌سازد که تقریباً کپیِ کاملِ والد است (همان کد، داده، file descriptorها). برای صرفه‌جویی، Linux از Copy-on-Write استفاده می‌کند: صفحاتِ حافظه ابتدا بین والد و فرزند **مشترک و read-only** می‌مانند؛ فقط لحظه‌ای که یکی بخواهد بنویسد، آن صفحه کپی می‌شود. به همین دلیل `fork` سریع است حتی برای پروسه‌های بزرگ.

</details>

<details><summary>**Zombie process چیست و چطور از بین می‌رود؟**</summary>

Zombie پروسه‌ای است که `exit` کرده ولی والدش هنوز `wait()` نکرده تا exit code را بخواند؛ پس entry‌اش در جدولِ پروسه می‌ماند (در `ps` با حالت `Z`/`defunct`). خودش منبعی مصرف نمی‌کند جز یک PID. وقتی والد `wait()` کند reap می‌شود. اگر والد بمیرد، zombie به PID 1 (systemd) re-parent می‌شود که خودش reapشان می‌کند. تجمعِ زیادِ zombie = باگ در والد (عدمِ wait).

</details>

<details><summary>**Concurrency و Parallelism چه فرقی دارند؟**</summary>

Concurrency یعنی چند task «در جریان» باشند و مدیریتشان به‌صورتِ درهم‌تنیده پیش برود — حتی روی یک هسته با time-slicing ممکن است (در هر لحظه فقط یکی واقعاً اجرا می‌شود). Parallelism یعنی چند task **واقعاً هم‌زمان** اجرا شوند، که به چند هسته/CPU نیاز دارد. جمله‌ی معروف: «Concurrency درباره‌ی ساختار است، Parallelism درباره‌ی اجرا».

</details>

<details><summary>**Deadlock چیست و چطور جلوگیری می‌شود؟**</summary>

Deadlock وقتی است که مجموعه‌ای از threadها هرکدام منتظرِ منبعی‌اند که دیگری گرفته، و هیچ‌کدام پیش نمی‌رود. چهار شرطِ Coffman لازم است: Mutual Exclusion، Hold-and-Wait، No Preemption، Circular Wait. شکستنِ هر یک کافی است — مثلاً همیشه قفل‌ها را با **ترتیبِ ثابتِ سراسری** بگیر (شکستنِ circular wait)، یا همه‌ی قفل‌ها را یک‌جا بگیر/هیچ (شکستنِ hold-and-wait)، یا timeout بگذار.

</details>

<details><summary>**Context switch چیست و چرا هزینه دارد؟**</summary>

Context switch یعنی CPU از اجرای یک thread/process به دیگری سوییچ کند. kernel باید حالتِ فعلی (رجیسترها، program counter، اطلاعاتِ حافظه) را ذخیره و حالتِ بعدی را بارگذاری کند. هزینه: مستقیم (چند هزار سیکل) و غیرمستقیم (پرشدنِ مجددِ cache و TLB → cache misses). به همین دلیل سوییچِ بیش‌ازحد (مثلاً تعدادِ زیادی thread) عملکرد را پایین می‌آورد.

</details>

<details><summary>**Mutex و Semaphore چه فرقی دارند؟**</summary>

Mutex یک قفلِ باینری برای دسترسیِ **انحصاری** به critical section است؛ مفهومِ «مالکیت» دارد (همان thread که قفل کرده باید آزاد کند). Semaphore یک شمارنده است که اجازه می‌دهد تا N منبع هم‌زمان استفاده شوند؛ مالکیت ندارد و هر thread می‌تواند signal کند. mutex حالتِ خاصِ semaphore با مقدارِ ۱ است، اما کاربردِ معنایی متفاوتی دارند (mutual exclusion در برابر شمارشِ منابع/سیگنال‌دهی).

</details>

<details><summary>**سناریو: load average سرور ۴۰ است ولی استفاده‌ی CPU فقط ۵٪. چه خبر است؟**</summary>

Load average فقط CPU را نمی‌شمارد؛ تعدادِ پروسه‌های **runnable به‌علاوه‌ی uninterruptible (D-state)** را می‌شمارد. CPU پایین + load بالا تقریباً همیشه یعنی پروسه‌ها در D-state گیر کرده‌اند و **منتظرِ I/O**اند — معمولاً دیسکِ کند/اشباع‌شده یا NFS/شبکه‌ی قطع. بررسی: `top`/`vmstat` ستونِ `wa` (I/O wait) و `b` (blocked)، `iostat -x` برای اشباعِ دیسک، و `ps -eo state,cmd | grep '^D'` برای یافتنِ پروسه‌های قفل‌شده. (در صورتِ NFS hang، حتی `kill -9` هم کار نمی‌کند.)

</details>

<details><summary>**سناریو: یک process در `ps` با حالتِ `D` دیده می‌شود و با `kill -9` نمی‌میرد — چرا؟**</summary>

حالتِ `D` یعنی **uninterruptible sleep**: پروسه داخلِ یک system call در حالِ انتظار برای I/O است (معمولاً دیسک یا NFS) و سیگنال‌ها — حتی `SIGKILL` — تا وقتی آن I/O تمام نشود تحویل داده نمی‌شوند. این برای جلوگیری از خرابیِ داده طراحی شده. راه‌حل واقعی: رفعِ مشکلِ زیرینِ I/O (مثلاً اتصالِ NFS را برقرار کن یا دیسکِ خراب را جدا کن)؛ گاهی فقط reboot.

</details>

<details><summary>**nice value چیست و چه بازه‌ای دارد؟**</summary>

nice اولویتِ زمان‌بندیِ یک پروسه را تعیین می‌کند، از `-20` (بالاترین اولویت) تا `+19` (پایین‌ترین). مقدارِ پیش‌فرض `0`. عددِ کمتر = سهمِ بیشترِ CPU. کاربرِ عادی فقط می‌تواند nice را **بالا** ببرد (کم‌اولویت‌تر کند)؛ کاهشِ آن (اولویت‌دادن) نیاز به root دارد. با `nice` هنگام اجرا و `renice` برای پروسه‌ی در حال اجرا تنظیم می‌شود.

</details>

### حافظه و حافظه‌ی مجازی (Memory & Virtual Memory)

<details><summary>**Virtual memory چیست و چه مزایایی دارد؟**</summary>

Virtual memory به هر process توهمِ یک فضای آدرسِ بزرگ، پیوسته و خصوصی می‌دهد که مستقل از RAM فیزیکی است. مزایا: **ایزولاسیون** (هر process فقط حافظه‌ی خودش را می‌بیند)، امکانِ استفاده از فضای بیش از RAM (با swap)، **اشتراکِ صفحه** (کتابخانه‌های مشترک یک کپی در RAM)، و سادگیِ تخصیص (آدرس‌های مجازیِ پیوسته که می‌توانند به framهای پراکنده map شوند). ترجمه با MMU و page table انجام می‌شود.

</details>

<details><summary>**Page fault چیست؟ minor در برابر major؟**</summary>

Page fault وقتی است که process به صفحه‌ای دسترسی پیدا می‌کند که در page table معتبر/حاضر نیست؛ یک trap به kernel می‌خورد. **Minor fault**: صفحه در RAM هست ولی هنوز به این process map نشده (مثلاً صفحه‌ی مشترک یا تازه‌تخصیص‌یافته) — ارزان. **Major fault**: صفحه باید از **دیسک** خوانده شود (swap یا فایلِ memory-mapped) — گران (میلی‌ثانیه‌ها). نرخِ بالای major fault = فشارِ حافظه/swapping.

</details>

<details><summary>**Thrashing چیست؟**</summary>

Thrashing حالتی است که RAM آن‌قدر کم است که سیستم بیشترِ وقتش را صرفِ swap in/out صفحات می‌کند به‌جای کارِ مفید. CPU درگیرِ مدیریتِ صفحات می‌شود، throughput سقوط می‌کند و سیستم تقریباً قفل می‌شود. علتش معمولاً working set کلِ پروسه‌های فعال بزرگ‌تر از RAM است. نشانه: `vmstat` با `si/so` بسیار بالا. راه‌حل: کاهشِ بار، افزایشِ RAM، یا کشتنِ پروسه‌ی پرمصرف.

</details>

<details><summary>**Stack و Heap چه فرقی دارند؟**</summary>

Stack حافظه‌ی هر thread برای فریم‌های تابع، متغیرهای محلی و آدرسِ بازگشت است؛ خودکار با فراخوانی/بازگشتِ تابع رشد و کوچک می‌شود (LIFO)، سریع، با اندازه‌ی محدود (overflow → crash). Heap حافظه‌ی تخصیصِ پویا (`malloc`/`new`) است که برنامه‌نویس آزادش می‌کند؛ انعطاف‌پذیر و بزرگ ولی کندتر و مستعدِ memory leak و fragmentation.

</details>

<details><summary>**OOM Killer چیست و چطور قربانی را انتخاب می‌کند؟**</summary>

وقتی Linux واقعاً حافظه کم بیاورد (حتی swap هم پر شود) و نتواند درخواستِ حافظه را برآورده کند، **OOM Killer** فعال می‌شود و یک پروسه را می‌کشد تا حافظه آزاد شود. انتخاب بر اساسِ `oom_score` است که به مصرفِ حافظه، عمر و تنظیماتِ `oom_score_adj` بستگی دارد. می‌توان با نوشتن در `/proc/<pid>/oom_score_adj` (مثلاً `-1000`) یک پروسه را در عمل مصون کرد. در لاگ (`dmesg`/`journalctl`) با «Out of memory: Killed process» دیده می‌شود.

</details>

### System Calls و Kernel

<details><summary>**System call چیست و چرا لازم است؟**</summary>

System call تنها رابطِ کنترل‌شده‌ای است که برنامه‌ی user (در User Mode) از طریق آن از kernel (در Kernel Mode) درخواستِ خدمت می‌کند: خواندن فایل، تخصیصِ حافظه، شبکه و... لازم است چون برنامه‌ها به‌دلایلِ امنیت و پایداری اجازه‌ی دسترسیِ مستقیم به سخت‌افزار را ندارند؛ kernel این درخواست‌ها را اعتبارسنجی و هماهنگ می‌کند تا پروسه‌ها به هم آسیب نزنند. هر فراخوانی یک **mode switch** به همراه دارد.

</details>

<details><summary>**فرق system call و یک تابعِ معمولیِ کتابخانه چیست؟**</summary>

تابعِ کتابخانه (مثلِ `strlen`) کاملاً در User Mode اجرا می‌شود و فقط یک پرشِ معمولی است. system call (مثلِ `read`) باعثِ گذار به Kernel Mode می‌شود (با دستورِ ویژه‌ای مثلِ `syscall` در x86-64) که هزینه‌بَر است. توابعی مثلِ `printf` در glibc در نهایت یک system call (`write`) می‌زنند ولی buffering و فرمت‌کردن را در user space انجام می‌دهند تا تعدادِ syscall کم شود.

</details>

<details><summary>**File descriptor چیست؟ مقادیرِ ۰، ۱، ۲ چه هستند؟**</summary>

File descriptor یک عددِ صحیحِ غیرمنفی است که kernel به هر منبعِ I/O باز (فایل، socket، pipe، device) برای یک process اختصاص می‌دهد؛ index در جدولِ فایل‌های بازِ آن process. به‌طورِ قراردادی: `0`=stdin، `1`=stdout، `2`=stderr. redirectionِ شل (مثلاً `2>&1`) در واقع همین FDها را دستکاری می‌کند. محدودیتِ تعدادِ FD با `ulimit -n` کنترل می‌شود.

</details>

<details><summary>**سناریو: برنامه‌ای با خطای «Too many open files» کرش می‌کند. تشخیص و رفع؟**</summary>

برنامه به سقفِ تعدادِ file descriptor رسیده. بررسی: `ulimit -n` (سقفِ نرم)، و `ls /proc/<pid>/fd | wc -l` برای شمارشِ FDهای بازِ آن پروسه؛ اگر مدام بالا می‌رود = **fd leak** (socket/فایلِ بازشده ولی بسته‌نشده). رفع: باگِ نشت را در کد ببند (همیشه `close`/`with`)، و اگر واقعاً به FD زیاد نیاز است سقف را بالا ببر (`ulimit -n`، یا `LimitNOFILE` در unit فایلِ systemd، یا `/etc/security/limits.conf`).

</details>

### Shell, Bash و Environment

<details><summary>**فرق `'...'`، `"..."` و بدونِ quote در Bash چیست؟**</summary>

`'single'` کاملاً تحت‌اللفظی است — هیچ بسطی رخ نمی‌دهد. `"double"` متغیرها (`$VAR`)، command substitution (`$(...)`) و `\` را بسط می‌دهد ولی glob و word splitting را مهار می‌کند. بدونِ quote، Bash هم متغیر را بسط می‌دهد و هم نتیجه را روی فاصله‌ها **می‌شکند** و glob می‌کند — منشأِ باگ‌های فراوان. قاعده: همیشه متغیرها را در `"..."` بگذار مگر دلیلِ خاصی داشته باشی.

</details>

<details><summary>**`set -euo pipefail` چه می‌کند؟**</summary>

`-e`: اسکریپت با اولین دستوری که کدِ غیرصفر برگرداند فوراً متوقف می‌شود. `-u`: استفاده از متغیرِ تعریف‌نشده خطا می‌دهد (به‌جای رشته‌ی خالی). `-o pipefail`: کدِ خروجیِ یک pipe برابرِ آخرین دستورِ ناموفق است (نه فقط آخرین دستور) — مثلاً `false | true` دیگر موفق حساب نمی‌شود. این سه با هم اسکریپت را «سخت‌گیر و امن» می‌کنند و باگ‌های خاموش را آشکار.

</details>

<details><summary>**فرق shell variable و environment variable چیست؟**</summary>

Shell variable فقط در همان نمونه‌ی شِل وجود دارد و به پروسه‌های فرزند به ارث نمی‌رسد. وقتی با `export` آن را environment variable کنیم، در محیطِ پروسه قرار می‌گیرد و هنگامِ `fork`/`exec` به **رونوشتِ محیطِ فرزندان** کپی می‌شود. به همین دلیل `FOO=bar` تنها در شِل دیده می‌شود ولی `export FOO=bar` در برنامه‌هایی که از آن شِل اجرا می‌کنی هم در دسترس است.

</details>

<details><summary>**`PATH` چیست و چرا گاهی `./script` به‌جای `script` لازم است؟**</summary>

`PATH` فهرستی از دایرکتوری‌هاست که شِل برای یافتنِ یک فرمان (که با نام صدا می‌زنی) به‌ترتیب می‌گردد. اگر دایرکتوریِ جاری (`.`) در `PATH` نباشد (که از نظرِ امنیتی توصیه می‌شود نباشد)، باید برنامه‌ی محلی را با مسیرِ صریح `./script` صدا بزنی تا شِل بداند از کجا بگیرد و سراغِ `PATH` نرود. گذاشتنِ `.` در `PATH` خطرناک است چون اجرای فایلِ مخربِ هم‌نامِ یک دستور را ممکن می‌کند.

</details>

<details><summary>**فرق `cmd > file` و `cmd 2>&1 > file` و `cmd > file 2>&1`؟**</summary>

`> file` فقط stdout (fd 1) را به file می‌فرستد؛ stderr همچنان به ترمینال. ترتیب در redirect مهم است: `> file 2>&1` اول stdout را به file می‌برد، **بعد** stderr را به «هرجا که stdout اشاره می‌کند» (یعنی file) — هر دو در file. اما `2>&1 > file` اول stderr را به مقصدِ فعلیِ stdout (ترمینال) می‌بندد و **بعد** stdout را به file می‌برد — پس stderr به ترمینال می‌ماند و فقط stdout در file. این یک گاتچای کلاسیک است.

</details>

### systemd و سرویس‌ها

<details><summary>**فرق `systemctl start` و `systemctl enable`؟**</summary>

`start` سرویس را همین حالا اجرا می‌کند ولی این انتخاب تا ریبوت دوام دارد (در بوتِ بعدی خبری نیست). `enable` با ساختِ symlink در targetِ مناسب، سرویس را برای **شروعِ خودکار در بوت** ثبت می‌کند ولی همین الان اجرایش نمی‌کند. برای هر دو با هم: `systemctl enable --now`. متضادها: `stop` و `disable`.

</details>

<details><summary>**PID 1 چیست و چرا مهم است؟**</summary>

PID 1 اولین پروسه‌ی user-space است که kernel هنگامِ بوت اجرا می‌کند (امروز معمولاً **systemd**، قبلاً SysV init). مسئولیت‌ها: راه‌اندازیِ بقیه‌ی سرویس‌ها بر اساسِ وابستگی‌ها، مدیریتِ چرخه‌ی حیاتِ آن‌ها (شروع/توقف/ری‌استارت)، و **reap کردنِ orphanها** (هر پروسه‌ای که والدش بمیرد به PID 1 می‌رسد). اگر PID 1 بمیرد، **kernel panic** می‌شود. در کانتینرها هم یک init سبک به‌عنوانِ PID 1 برای reap کردنِ zombieها توصیه می‌شود.

</details>

<details><summary>**یک سرویس مدام crash و restart می‌شود (restart loop). چطور دیباگ می‌کنی؟**</summary>

`systemctl status myapp` برای دیدنِ حالت و آخرین خطوط، و `journalctl -u myapp -e` برای لاگِ کامل و علتِ خروج (exit code/signal). علل رایج: فایلِ کانفیگ/پورتِ اشغال‌شده، وابستگیِ آماده‌نشده (نیاز به `After=`/`Requires=`)، نبودِ مجوز یا متغیرِ محیطی. systemd با `StartLimitIntervalSec`/`StartLimitBurst` بعد از چند restartِ پیاپی سرویس را در حالتِ `failed` رها می‌کند تا حلقه قطع شود؛ با `systemctl reset-failed` می‌توان ریست کرد. تنظیمِ `RestartSec` فاصله‌ی restart را کنترل می‌کند.

</details>

<details><summary>**فرق `Requires=` و `Wants=` در یک unit چیست؟**</summary>

هر دو وابستگی تعریف می‌کنند، اما `Requires=` یک وابستگیِ **سخت** است: اگر واحدِ وابسته شکست بخورد یا متوقف شود، این سرویس هم متوقف/شکست می‌خورد. `Wants=` وابستگیِ **نرم** است: تلاش می‌کند واحدِ دیگر را هم بالا بیاورد، اما اگر آن شکست بخورد این سرویس همچنان اجرا می‌شود. توجه: این‌ها فقط وابستگی‌اند نه ترتیب — برای ترتیب باید `After=`/`Before=` هم بگذاری.

</details>
