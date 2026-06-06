# سطح ۳ — توضیح فنی: Git & Version Control ⚙️

> دیگر قصه‌ای در کار نیست. این سند مرجع فنی است — همان چیزهایی که در مصاحبه و کار واقعی لازم داری. هر بخش با یک اشاره‌ی کوتاه به معادل قصه‌ای‌اش شروع می‌شود تا ذهنت وصل بماند.

---

## ۱. مبانی Git

> در قصه: «دفترچه‌ی جادویی» که هر نسخه را با یک یادداشت نگه می‌دارد.

### مدلِ ذهنی: Git یک گرافِ snapshot است، نه diff

برخلاف تصور رایج، Git تغییرات (delta) را ذخیره نمی‌کند؛ هر **commit** یک **snapshot** کاملِ درختِ فایل‌هاست (با اشتراکِ هوشمندِ محتوای تکراری). هر commit این‌ها را دارد: یک یا چند **parent**، یک اشاره به درختِ فایل‌ها، نویسنده، تاریخ، و پیام. شناسه‌ی هر commit یک **SHA-1/SHA-256 hash** از محتوای آن است — پس تغییرِ هر چیزی، شناسه را عوض می‌کند.

### سه درخت (The Three Trees)

| درخت | معادل قصه | نقش |
|---|---|---|
| **Working Directory** | میز کار | فایل‌های واقعی که ویرایش می‌کنی |
| **Staging Area / Index** | سینیِ آماده‌سازی | عکسِ بعدی از این ساخته می‌شود |
| **HEAD / Repository** | آلبومِ ثبت‌شده | آخرین commitِ ثبت‌شده |

```
Working Directory ──git add──▶ Staging (Index) ──git commit──▶ HEAD (history)
        ▲                            ▲                              │
        └──── git restore <f> ───────┴──── git restore --staged ────┘
```

### دستورهای پایه

| دستور | کار |
|---|---|
| `git init` | ساخت repo تازه (پوشه‌ی `.git`) |
| `git status` | وضعیت working dir و staging |
| `git add <f>` / `git add -p` | افزودن به staging (`-p` = تکه‌تکه و تعاملی) |
| `git commit -m "..."` | ثبت snapshot از staging |
| `git commit --amend` | اصلاح آخرین commit (پیام یا محتوا) — ⚠️ تاریخ را بازنویسی می‌کند |
| `git log --oneline --graph --all` | تاریخچه به‌صورت گراف |
| `git diff` / `git diff --staged` | تفاوتِ working↔index / index↔HEAD |
| `git show <commit>` | جزئیات و diffِ یک commit |

### HEAD، رفرنس‌ها و navigation

- **HEAD**: اشاره‌گر به «جایی که الان هستی» — معمولاً به یک شاخه، که آن هم به یک commit اشاره می‌کند.
- **Detached HEAD**: وقتی مستقیماً روی یک commit (نه شاخه) می‌ایستی (`git checkout <sha>`) — commit جدیدت به هیچ شاخه‌ای بند نیست و با جابه‌جایی گم می‌شود.
- مرجع‌های نسبی: `HEAD~1` (یک قبل)، `HEAD~3` (سه قبل)، `HEAD^` (parent اول)، `HEAD^2` (parent دومِ یک merge commit).

### `.gitignore`

الگوهای فایل/پوشه‌ای که Git باید نادیده بگیرد. **فقط روی فایل‌های untracked اثر دارد** — اگر فایلی قبلاً tracked شده، `.gitignore` نادیده‌اش نمی‌گیرد؛ باید با `git rm --cached <f>` از index خارجش کنی.

```gitignore
node_modules/
*.log
.env            # ← کلیدها و رمزها هرگز نباید commit شوند
dist/
.DS_Store
```

### Conventional Commits

قراردادی برای پیامِ ساخت‌یافته که tooling (تولید changelog، تعیین خودکار نسخه‌ی semver) رویش حساب می‌کند:

```
<type>(<scope>): <subject>

feat(auth): add OAuth login
fix(api): handle null user id
docs: update README
refactor!: drop deprecated endpoint   # «!» = breaking change
```

types رایج: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`.

> نکته‌ی مصاحبه: آیا Git تغییرات را ذخیره می‌کند یا snapshotها را؟ پاسخ درست **snapshot** است (با content-addressable storage و اشتراکِ blobهای یکسان) — نه delta مثل برخی VCSهای قدیمی.

---

## ۲. پس گرفتن و خنثی‌کردن (Undoing)

> در قصه: «سفر در زمان» و «پاک کردن خطا».

### `restore` در برابر `reset` در برابر `revert`

| دستور | چه چیزی را عوض می‌کند | تاریخچه را بازنویسی می‌کند؟ | امن روی تاریخچه‌ی مشترک؟ |
|---|---|---|---|
| `git restore <f>` | working dir (یک فایل) | ❌ | ✅ |
| `git restore --staged <f>` | فقط index (un-stage) | ❌ | ✅ |
| `git reset` | HEAD + (بسته به حالت) index/worktree | ✅ (commit می‌اندازد) | ❌ خطرناک |
| `git revert <commit>` | commit جدیدِ معکوس می‌سازد | ❌ (اضافه می‌کند) | ✅ امن |

### سه حالتِ `reset`

```
                 HEAD  Index  Working Dir
git reset --soft   ✏️    —       —      ← فقط HEAD عقب؛ تغییرات staged می‌مانند
git reset --mixed  ✏️    ✏️      —      ← (پیش‌فرض) HEAD+index؛ تغییرات unstaged روی میز
git reset --hard   ✏️    ✏️      ✏️ ⚠️  ← همه‌چیز عقب؛ کارِ ذخیره‌نشده نابود می‌شود
```

- `--soft`: برای «این چند commit را به یک commit تمیز فشرده کنم» (squash دستی).
- `--mixed`: برای «تغییرات را نگه دار ولی از staging و commit بیرونشان بیاور».
- `--hard`: فقط وقتی مطمئنی کارِ ذخیره‌نشده‌ای از دست نمی‌رود.

### قاعده‌ی طلایی: کِی `reset`، کِی `revert`

> **commit خصوصی (هنوز push نشده) → `reset` آزاد است.**
> **commit مشترک (push شده، دیگران pull کرده‌اند) → فقط `revert`.**
> چون `reset`/`rebase`/`commit --amend` تاریخچه را **بازنویسی** می‌کنند؛ اگر آن تاریخچه مشترک باشد، force-push لازم می‌شود و کارِ بقیه به هم می‌ریزد. `revert` به‌جایش یک commit تازه می‌سازد که اثرِ قدیمی را خنثی می‌کند و هیچ تاریخی را عوض نمی‌کند.

> نکته‌ی مصاحبه: «commit اشتباهی را که push شده چطور پس بگیرم؟» → اگر دیگران دارند رویش کار می‌کنند **`git revert`**؛ هرگز `reset --hard` + force-push روی شاخه‌ی مشترک.

---

## ۳. شاخه‌بندی، Merge و Rebase

> در قصه: «کپیِ رویایی»، «بافتنِ دو نخ»، و «دعوای آرش و کاوه».

### شاخه چیست؟

یک branch صرفاً یک **اشاره‌گرِ سبکِ متحرک** به یک commit است (یک فایلِ ۴۱ بایتی در `.git/refs/`). ساختن شاخه ارزان است — هیچ کپیِ فایلی اتفاق نمی‌افتد.

```bash
git branch                 # فهرست شاخه‌ها
git switch -c feature/x     # بساز و برو رویش (مدرن)
git checkout -b feature/x   # معادلِ قدیمی
git branch -d feature/x     # حذف (اگر merge شده)
git branch -D feature/x     # حذف اجباری
```

### Merge: Fast-forward در برابر Merge commit

```
Fast-forward (main از انشعاب جلو نرفته):       Merge commit (هر دو خط جلو رفته‌اند):

  o───o───o───o───o   main                       o───o───o───────M   main
            (فقط اشاره‌گر جلو کشیده شد)                  \       /
                                                          o─────o     feature
                                                       M = commit دو-والدی
```

```bash
git merge feature/x              # ff اگر ممکن باشد، وگرنه merge commit
git merge --no-ff feature/x      # همیشه merge commit بساز (تاریخِ شاخه را حفظ کن)
git merge --squash feature/x     # همه‌ی تغییرات را یک‌جا، بدون تاریخِ شاخه
```

### Conflict و حلِ آن

وقتی دو شاخه **همان ناحیه** از همان فایل را تغییر دهند، Git علامت‌گذاری می‌کند:

```
<<<<<<< HEAD
نسخه‌ی شاخه‌ی فعلی (ours)
=======
نسخه‌ی شاخه‌ی واردشونده (theirs)
>>>>>>> feature/x
```

مراحل حل: فایل را ویرایش کن (علامت‌ها را پاک، نسخه‌ی نهایی را بساز) → `git add <f>` → `git commit` (یا `git merge --continue`). برای لغو کل عملیات: `git merge --abort`.

### Rebase: بازنویسی برای تاریخچه‌ی خطی

```
قبل از rebase:                          بعد از  git rebase main  (روی feature):

o───o───A───B   main                    o───o───A───B           main
     \                                              \
      C───D   feature                                C'──D'      feature
                                         (C,D دوباره روی نوک main پخش شدند → SHAهای تازه)
```

```bash
git rebase main                  # commitهای شاخه‌ی فعلی را روی نوک main بازپخش کن
git rebase -i HEAD~4             # تعاملی: squash / reword / drop / reorder
git rebase --continue / --abort  # بعد از حل conflict / لغو
```

### merge در برابر rebase — جدول تصمیم

| معیار | Merge | Rebase |
|---|---|---|
| شکلِ تاریخچه | شاخه‌دار، با merge commit | خطی و تمیز |
| حفظِ تاریخِ واقعی | ✅ دقیقاً همان اتفاق | ❌ بازنویسی‌شده |
| SHAها | حفظ می‌شوند | عوض می‌شوند |
| امن روی شاخه‌ی مشترک؟ | ✅ | ❌ (قاعده‌ی طلایی) |
| مناسبِ | یکی‌کردنِ شاخه‌ها در main | تمیزکاریِ شاخه‌ی خصوصی قبل از PR |

> ⚖️ **قاعده‌ی طلاییِ rebase:** هرگز commitهایی را که در جای عمومی منتشر شده‌اند rebase نکن. rebase تاریخچه را بازنویسی می‌کند؛ روی شاخه‌ی مشترک، divergence و درد به بار می‌آورد.

> نکته‌ی مصاحبه: «فرق `merge` و `rebase`؟» — merge تاریخِ واقعی و non-destructive؛ rebase تاریخِ خطی ولی بازنویسی‌کننده. هر دو همان محتوای نهایی را می‌دهند، فرق در *شکلِ گراف* و *امنیت روی اشتراک* است.

---

## ۴. Remoteها

> در قصه: «کتابخانه‌ی شهر».

- **remote**: یک repo که جایی دیگر (سرور) زندگی می‌کند. اسمِ پیش‌فرض **`origin`**.
- **clone**: کپیِ کاملِ یک remote + ثبتِ خودکارِ `origin` + checkout شاخه‌ی پیش‌فرض.
- **tracking branch (upstream)**: رابطه‌ی شاخه‌ی محلی با جفتش روی remote (مثل `main` ↔ `origin/main`)؛ به همین خاطر `push`/`pull` بدون آرگومان می‌دانند کجا بروند.
- **remote-tracking refs** مثل `origin/main`: عکسِ محلیِ «آخرین چیزی که از remote دیدم» — با `fetch` به‌روز می‌شوند.

```bash
git remote -v                          # فهرست remoteها
git remote add origin <url>
git clone <url>
git fetch origin                       # فقط دانلودِ تازه‌ها؛ به working dir دست نمی‌زند
git pull                               # = fetch + merge (یا با --rebase = fetch + rebase)
git push -u origin feature/x           # push + تنظیم upstream (اولین بار)
git push                               # دفعات بعد
git push --force-with-lease            # force امن‌تر: اگر کسی دیگر push کرده، رد می‌کند
```

> **`fetch` در برابر `pull`:** `fetch` فقط دادهٔ تازه را می‌آورد و `origin/*` را به‌روز می‌کند، بدون دست‌زدن به کارِ تو. `pull = fetch + merge`. اگر `pull --rebase` بزنی، می‌شود `fetch + rebase`.

> نکته‌ی مصاحبه: چرا `--force-with-lease` بهتر از `--force` است؟ چون `--force` کورکورانه remote را بازنویسی می‌کند و ممکن است کارِ تازه‌ی همکار را پاک کند؛ `--force-with-lease` فقط وقتی push می‌کند که remote از آخرین باری که دیدی تغییر نکرده باشد.

---

## ۵. Pull Requestها

> در قصه: «لطفاً اول صفحه‌هایم را بخوان، بعد به اصل اضافه کن».

- **PR/MR** یک قابلیتِ **پلتفرم** است (GitHub/GitLab/Bitbucket)، نه خودِ Git: پیشنهادِ ادغامِ یک شاخه در شاخه‌ی دیگر، همراه با بستری برای **review**.
- **جریان review**: push شاخه → باز کردن PR → بحث و کامنتِ خط‌به‌خط → درخواست تغییر یا Approve → عبور از CI (تست/لینت) → merge.
- **استراتژی‌های merge در PR**:
  - *Merge commit*: کلِ تاریخِ شاخه + یک merge commit.
  - *Squash and merge*: همه‌ی commitهای شاخه در یک commit تمیز روی main — رایج‌ترین برای تاریخچه‌ی مرتب.
  - *Rebase and merge*: commitها به‌صورت خطی روی main، بدون merge commit.
- **Draft PR (پیش‌نویس)**: PR که هنوز آماده‌ی merge نیست؛ برای گرفتنِ بازخوردِ زودهنگام یا اجرای CI روی کارِ ناتمام. قابلیتِ merge تا «Ready for review» شدن قفل است.
- **CODEOWNERS / required reviews / branch protection**: قوانینی که جلوی merge بدونِ تأیید یا بدون عبور از CI را می‌گیرند.

> نکته‌ی مصاحبه: PR کجا در Git است؟ — جایی نیست! PR مفهومِ پلتفرم است. خودِ Git فقط `push` و `merge` می‌شناسد.

---

## ۶. استراتژی‌ها و Workflowها

> در قصه: «نقشه‌های کار کردن با هم».

| Workflow | شاخه‌های اصلی | عمرِ شاخه‌ی feature | بهترین برای |
|---|---|---|---|
| **GitHub Flow** | فقط `main` (همیشه deployable) | کوتاه؛ هر feature یک PR | وب‌اپ و سرویس‌ها با continuous deployment |
| **Git Flow** | `main` + `develop` + `release/*` + `hotfix/*` | متوسط؛ چرخه‌ی release رسمی | محصولاتِ نسخه‌دار (دسکتاپ/موبایل) با release برنامه‌ریزی‌شده |
| **Trunk-Based** | فقط `trunk`/`main` | بسیار کوتاه (ساعت/روز)، یا commit مستقیم پشتِ feature flag | تیم‌های بزرگ، CI/CD بالغ، سرعتِ بالا |

- **Feature branch**: واحدِ کارِ همه‌ی این workflowها — شاخه‌ی کوتاه‌عمر برای یک تغییرِ مشخص، تا `main` همیشه سالم بماند.
- **GitHub Flow ساده است**: شاخه بزن، کار کن، PR، review، merge، deploy. هیچ شاخه‌ی بلندمدتِ دیگری نیست.
- **Git Flow ساختارمند ولی سنگین است**: شاخه‌های release و hotfix جدا — برای جایی که چند نسخه‌ی منتشرشده را همزمان نگه می‌داری.
- **Trunk-based روی شاخه‌های کوتاه و feature flag تکیه دارد**: integration مداوم، کمترین divergence، کمترین conflict — ولی نیازمندِ تست خودکارِ قوی.

> نکته‌ی مصاحبه: کِی Git Flow و کِی GitHub Flow؟ — Git Flow وقتی release رسمی و نسخه‌بندی و پشتیبانی همزمانِ چند نسخه داری؛ GitHub Flow وقتی به production مداوم deploy می‌کنی و «main همیشه آماده» کافی است. بسیاری تیم‌های مدرن trunk-based را برای کاهشِ merge hell ترجیح می‌دهند.

---

## ۷. ابزارهای کاربردی (Practical Extras)

> در قصه: «جیب مخفی»، «برداشتنِ یک عکس»، «برچسب طلایی»، و «پیدا کردنِ صفحه‌ی خراب».

### Stash — کنار گذاشتنِ موقت

```bash
git stash                  # کنار گذاشتنِ تغییراتِ tracked
git stash -u               # شاملِ فایل‌های untracked
git stash list             # فهرستِ جیب‌ها
git stash pop              # برگرداندن و حذف از stash
git stash apply            # برگرداندن بدون حذف
```

### Cherry-pick — انتقالِ یک commit خاص

```bash
git cherry-pick <sha>           # همان commit را روی شاخه‌ی فعلی بازتولید کن (SHA تازه)
git cherry-pick <sha1>..<sha2>  # یک بازه
```
کاربرد: انتقالِ یک hotfix از یک شاخه به شاخه‌ی دیگر بدون merge کاملِ آن.

### Tags — نشان‌گذاریِ نسخه‌ها

```bash
git tag v1.2.0                          # سبک (lightweight)
git tag -a v1.2.0 -m "release 1.2.0"    # annotated (دارای نویسنده/تاریخ/پیام — توصیه‌شده)
git push origin --tags
```
معمولاً با **SemVer** (`MAJOR.MINOR.PATCH`) ترکیب می‌شود.

### `git bisect` — شکارِ باگ با جستجوی دودویی

```bash
git bisect start
git bisect bad                 # commit فعلی خراب است
git bisect good v1.0           # این نسخه سالم بود
# Git به‌صورت دودویی commitها را checkout می‌کند؛ هر بار تست کن و علامت بزن:
git bisect good   # یا  git bisect bad
# ...تا Git اولین commitِ خراب را پیدا کند
git bisect reset               # برگرد به حالت عادی
# خودکار با اسکریپتِ تست:
git bisect run ./test.sh
```
در میانِ ۱۰۰۰ commit، تنها ~۱۰ بار تست لازم است (log₂).

### `git reflog` — تورِ نجات

`reflog` لاگِ **محلیِ** هر جابه‌جاییِ HEAD است (commit، reset، checkout، rebase...). حتی بعد از `reset --hard` یا rebaseِ خراب، commitِ «گم‌شده» هنوز در reflog هست و قابلِ بازیابی است:

```bash
git reflog                     # تاریخچه‌ی حرکاتِ HEAD با SHA
git reset --hard HEAD@{2}      # برگرد به حالتِ دو حرکت قبل
git branch rescue <sha>        # شاخه‌ی نجات روی commitِ گم‌شده بساز
```

> نکته‌ی مصاحبه: «همکار روی کارم force-push کرد و commitهایم رفت — چه کنم؟» → `git reflog` (یا `git fsck --lost-found`) برای پیدا کردنِ SHA، بعد شاخه/reset به آن، سپس `push --force-with-lease` برای بازگرداندن.

---

## 🔬 آزمایشگاه — یک جلسه‌ی کامل و قابلِ اجرا

> این بلوک را خط‌به‌خط در یک پوشه‌ی موقت اجرا کن. هر کاری از init تا conflict و recovery را خودت لمس می‌کنی.

```bash
# ───────────── ۱. ساختِ repo و چند commit ─────────────
mkdir git-lab && cd git-lab
git init -b main            # -b main تا شاخه‌ی اولیه main باشد (نه master)
git config user.name  "Sam"
git config user.email "sam@example.com"

printf "title: My Story\nhero: Arash\n" > story.txt
git add story.txt
git commit -m "feat: initial story with hero Arash"

printf "title: My Story\nhero: Arash\nvillain: Dragon\n" > story.txt
git commit -am "feat: add the dragon villain"

git log --oneline --graph          # دو commit روی main

# ───────────── ۲. شاخه زدن و کار موازی ─────────────
git switch -c feature/rename-hero
# روی شاخه، همان خطِ hero را عوض می‌کنیم → کاوه
printf "title: My Story\nhero: Kaveh\nvillain: Dragon\n" > story.txt
git commit -am "feat: rename hero to Kaveh"

# ───────────── ۳. ساختِ عمدیِ یک conflict ─────────────
git switch main
# روی main، *همان خطِ hero* را جور دیگری عوض می‌کنیم → سهراب
printf "title: My Story\nhero: Sohrab\nvillain: Dragon\n" > story.txt
git commit -am "feat: rename hero to Sohrab"

git merge feature/rename-hero       # 💥 CONFLICT روی خطِ hero
git status                          # «both modified: story.txt»
cat story.txt                       # علامت‌های <<<<<<< ======= >>>>>>> را ببین

# ───────────── حلِ conflict ─────────────
# فایل را دستی ویرایش کن و خطِ نهایی را بگذار  hero: Kaveh
printf "title: My Story\nhero: Kaveh\nvillain: Dragon\n" > story.txt
git add story.txt                   # «حلش کردم»
git commit -m "merge: resolve hero name conflict -> Kaveh"
git log --oneline --graph           # merge commit دو-والدی را ببین

# ───────────── ۴. reset در برابر revert ─────────────
echo "oops: unwanted line" >> story.txt
git commit -am "chore: oops, an unwanted change"

# revert: امن، یک commit معکوس می‌سازد (تاریخ دست‌نخورده)
git revert --no-edit HEAD
git log --oneline                   # هم commitِ بد، هم revertش هست

# reset --soft: HEAD یکی عقب، تغییرات staged می‌مانند
git reset --soft HEAD~1
git status                          # تغییرات در staging

# reset --hard: ⚠️ نابودکننده (فقط چون کار مهمی نداریم)
git reset --hard HEAD~1

# نجات با reflog: همان commit را که با hard انداختیم برگردان
git reflog                          # SHA را پیدا کن
# git reset --hard <sha-from-reflog>   ← (در صورت نیاز)

# ───────────── ۵. stash ─────────────
echo "WIP scribble" >> story.txt    # کارِ نیمه‌تمام
git stash                           # کنارش بگذار (working dir تمیز شد)
git stash list
git stash pop                       # برش گردان و ادامه بده

# ───────────── ۶. tag ─────────────
git tag -a v1.0 -m "first complete draft"
git tag                             # v1.0
```

---

## ✅ چک‌لیست تسلط

اگر بتوانی به این‌ها جواب بدهی، این فصل را بلدی:

- [ ] سه درختِ Git را نام ببر و بگو هر فایل با کدام دستور بینشان جابه‌جا می‌شود.
- [ ] Git تغییرات را ذخیره می‌کند یا snapshot؟ چرا این فرق مهم است؟
- [ ] فرقِ `restore`، `reset` و `revert` چیست؟ کدام روی تاریخچه‌ی مشترک امن است؟
- [ ] سه حالتِ `reset` (soft/mixed/hard) با working dir و index چه می‌کنند؟
- [ ] فرقِ fast-forward و merge commit؟ کِی `--no-ff` بزنم؟
- [ ] فرقِ `merge` و `rebase` در شکلِ گراف و در امنیت؟ قاعده‌ی طلایی چیست؟
- [ ] یک merge conflict را قدم‌به‌قدم چطور حل می‌کنی؟
- [ ] فرقِ `fetch` و `pull`؟ `pull = ?`
- [ ] tracking branch چیست و چرا `git push` بدون آرگومان کار می‌کند؟
- [ ] PR چیست و در کدام لایه است (Git یا پلتفرم)؟ Draft PR کِی؟
- [ ] GitHub Flow، Git Flow و trunk-based هرکدام کِی مناسب‌اند؟
- [ ] کاربردِ stash، cherry-pick، tag و bisect را با یک مثال بگو.
- [ ] commitِ گم‌شده بعد از `reset --hard` یا force-push را چطور برمی‌گردانی؟

---

## 🎯 سوالات مصاحبه

> روی هر سوال کلیک کن تا جواب را ببینی. اول خودت جواب بده، بعد چک کن!

### مبانی و سه درخت

<details><summary>**۱. سه درختِ Git چیست و یک فایل چطور بینشان حرکت می‌کند؟**</summary>

Working Directory (فایل‌های واقعی) → با `git add` → Staging/Index (آماده‌ی commit) → با `git commit` → Repository/HEAD (snapshot دائمی). برگشت: `git restore --staged` از index به working، `git restore` تغییراتِ working را دور می‌ریزد.

</details>

<details><summary>**۲. Git تغییرات (diff) را ذخیره می‌کند یا snapshot؟**</summary>

snapshot. هر commit یک تصویرِ کاملِ درختِ فایل‌هاست؛ Git محتوای تکراری را با content-addressable storage به اشتراک می‌گذارد، پس کارآمد است. diffها هنگامِ نمایش *محاسبه* می‌شوند، نه ذخیره.

</details>

<details><summary>**۳. HEAD دقیقاً چیست؟ Detached HEAD یعنی چه؟**</summary>

HEAD اشاره‌گر به «جایی که الان هستی» است — معمولاً به یک شاخه (که خودش به یک commit اشاره می‌کند). Detached HEAD یعنی مستقیماً روی یک commit ایستاده‌ای، نه روی شاخه؛ commitِ جدیدی که اینجا بسازی به هیچ شاخه‌ای بند نیست و با جابه‌جایی گم می‌شود (مگر شاخه بزنی).

</details>

<details><summary>**۴. چرا یک Staging Area وسطی وجود دارد؟**</summary>

تا بتوانی commitهایت را با دقت بسازی: از میانِ چند تغییرِ پراکنده، فقط آن‌هایی را که منطقاً به هم مربوط‌اند انتخاب و در یک commit جمع کنی (`git add -p` حتی تکه‌تکه‌ی یک فایل). نتیجه: تاریخچه‌ی تمیز و قابل‌فهم.

</details>

<details><summary>**۵. `.gitignore` یک فایلِ از قبل tracked شده را نادیده می‌گیرد؟**</summary>

نه. `.gitignore` فقط روی فایل‌های untracked اثر دارد. برای فایلی که قبلاً commit شده، باید `git rm --cached <file>` بزنی تا از index خارج شود، سپس commit کنی؛ آنگاه ignore اثر می‌کند.

</details>

<details><summary>**۶. Conventional Commits چیست و چه فایده‌ای دارد؟**</summary>

قراردادی برای پیامِ commit به شکلِ `type(scope): subject` (مثل `feat:`, `fix:`). ابزارها از رویش changelog می‌سازند و نسخه‌ی semver را خودکار تعیین می‌کنند (`fix`→patch، `feat`→minor، `!`/`BREAKING`→major).

</details>

### Undo، Reset و Revert

<details><summary>**۷. فرقِ `git reset` و `git revert` چیست؟ کدام روی تاریخچه‌ی push‌شده امن است؟**</summary>

`reset` اشاره‌گرِ HEAD را عقب می‌برد و تاریخچه را **بازنویسی** می‌کند (commitها می‌افتند) — فقط روی کارِ خصوصی امن است. `revert` یک commitِ **جدید** می‌سازد که اثرِ commitِ قدیمی را خنثی می‌کند بدون دست‌زدن به تاریخ — روی تاریخچه‌ی مشترک امن است. روی شاخه‌ی مشترک همیشه `revert`.

</details>

<details><summary>**۸. سه حالتِ `reset` (soft/mixed/hard) چه فرقی دارند؟**</summary>

هر سه HEAD را عقب می‌برند. `--soft`: index و working دست‌نخورده (تغییرات staged می‌مانند). `--mixed` (پیش‌فرض): index هم ریست می‌شود (تغییرات روی working، unstaged). `--hard`: index و working هم بازنویسی می‌شوند — کارِ ذخیره‌نشده **نابود** می‌شود.

</details>

<details><summary>**۹. سناریو: آخرین commit را push کرده‌ای و حالا می‌خواهی پسش بگیری، ولی همکارت قبلاً pull کرده. چه می‌کنی؟**</summary>

`git revert <sha>` و push. این یک commitِ معکوس اضافه می‌کند و تاریخچه‌ی مشترک را عوض نمی‌کند، پس کارِ همکار به هم نمی‌ریزد. هرگز `reset --hard` + force-push روی شاخه‌ای که دیگران دارند.

</details>

<details><summary>**۱۰. فرقِ `git restore <file>` و `git checkout <file>` در کارِ روزمره؟**</summary>

`git restore <file>` (مدرن) صریحاً تغییراتِ working dir آن فایل را به نسخه‌ی HEAD/index برمی‌گرداند. `git checkout <file>` همان کار را می‌کند ولی overload شده (هم فایل، هم شاخه)؛ Git مدرن کار را به `restore` (فایل) و `switch` (شاخه) تقسیم کرده تا ابهام کم شود.

</details>

### شاخه‌بندی، Merge و Conflict

<details><summary>**۱۱. در Git یک branch دقیقاً چیست؟ چرا ساختنش این‌قدر ارزان است؟**</summary>

یک branch فقط یک اشاره‌گرِ متحرک (یک فایلِ کوچک حاویِ یک SHA) به یک commit است. هیچ فایلی کپی نمی‌شود؛ به همین خاطر ساخت/حذفِ شاخه آنی و ارزان است.

</details>

<details><summary>**۱۲. فرقِ fast-forward و merge commit؟ `--no-ff` چه می‌کند و کِی لازم است؟**</summary>

اگر شاخه‌ی مقصد از نقطه‌ی انشعاب جلو نرفته باشد، Git فقط اشاره‌گر را جلو می‌کشد (fast-forward، بدون commit تازه). اگر هر دو جلو رفته باشند، یک merge commitِ دو-والدی می‌سازد. `--no-ff` حتی وقتی ff ممکن است، عمداً merge commit می‌سازد تا مرزِ یک feature در تاریخچه دیده شود.

</details>

<details><summary>**۱۳. یک merge conflict را قدم‌به‌قدم چطور حل می‌کنی؟**</summary>

۱) `git status` فایل‌های متضاد را نشان می‌دهد. ۲) فایل را باز کن؛ بینِ `<<<<<<<`، `=======`، `>>>>>>>` هر دو نسخه هست. ۳) محتوای نهاییِ درست را بساز و علامت‌ها را پاک کن. ۴) `git add <file>`. ۵) `git commit` (یا `git merge --continue`). برای لغوِ کامل: `git merge --abort`.

</details>

<details><summary>**۱۴. در نشانه‌گذاریِ conflict، «ours» و «theirs» کدام است؟**</summary>

بخشِ بالا (`HEAD`) **ours** = شاخه‌ای که رویش ایستاده‌ای؛ بخشِ پایین **theirs** = شاخه‌ای که داری merge می‌کنی. (توجه: در rebase این برعکس می‌شود، چون rebase تغییراتِ تو را روی نوکِ شاخه‌ی پایه «بازپخش» می‌کند.)

</details>

### Merge در برابر Rebase

<details><summary>**۱۵. فرقِ `merge` و `rebase` چیست؟**</summary>

merge دو خطِ تاریخ را با یک merge commitِ دو-والدی به هم وصل می‌کند — تاریخِ واقعی و non-destructive، ولی گراف شاخه‌دار. rebase commitهای شاخه را روی نوکِ شاخه‌ی دیگر بازپخش می‌کند — تاریخِ خطی و تمیز، ولی SHAها عوض می‌شوند و تاریخ بازنویسی می‌شود. محتوای نهایی یکی است؛ فرق در شکلِ گراف و امنیت.

</details>

<details><summary>**۱۶. قاعده‌ی طلاییِ rebase چیست و چرا؟**</summary>

«هرگز تاریخچه‌ی منتشرشده/مشترک را rebase نکن». چون rebase commitهای قدیمی را با commitهای جدید (SHA متفاوت) جایگزین می‌کند؛ اگر دیگران آن commitهای قدیمی را داشته باشند، تاریخ‌ها واگرا می‌شوند، force-push لازم می‌شود و کارشان خراب می‌شود. rebase فقط برای شاخه‌ی خصوصیِ خودت قبل از اشتراک.

</details>

<details><summary>**۱۷. سناریو: شاخه‌ی feature ده commit دارد ولی شلوغ است؛ می‌خواهی قبل از PR تمیزش کنی. چه می‌کنی؟**</summary>

`git rebase -i main` (یا `HEAD~10`) و در حالتِ تعاملی commitها را `squash`/`fixup`/`reword`/`reorder` کن تا چند commitِ معنادار شود. چون شاخه هنوز خصوصی (push‌نشده یا فقط مالِ خودت) است، بازنویسی بی‌خطر است.

</details>

### Remoteها

<details><summary>**۱۸. فرقِ `git fetch` و `git pull`؟**</summary>

`fetch` دادهٔ تازه را از remote دانلود می‌کند و `origin/*` را به‌روز می‌کند، ولی به working dir و شاخه‌ی فعلی‌ات دست نمی‌زند. `pull = fetch + merge` (یا با `--rebase` می‌شود `fetch + rebase`). یعنی `pull` خودکار کارِ remote را با کارِ تو ادغام می‌کند.

</details>

<details><summary>**۱۹. tracking branch (upstream) چیست؟ چرا `git push` بدون آرگومان کار می‌کند؟**</summary>

رابطه‌ای بینِ شاخه‌ی محلی و جفتش روی remote (مثل `main` ↔ `origin/main`). با `git push -u origin <branch>` تنظیم می‌شود؛ بعد از آن `git push`/`git pull` بدون نوشتنِ remote و شاخه می‌دانند کجا بروند.

</details>

<details><summary>**۲۰. فرقِ `--force` و `--force-with-lease` چیست و چرا دومی امن‌تر است؟**</summary>

`--force` کورکورانه شاخه‌ی remote را با مالِ تو بازنویسی می‌کند، حتی اگر کسی بعدِ آخرین fetchِ تو چیزی push کرده باشد (کارش را می‌کشد). `--force-with-lease` فقط وقتی push می‌کند که remote دقیقاً همان چیزی باشد که آخرین بار دیدی؛ وگرنه رد می‌کند. روی شاخه‌های مشترک همیشه `--force-with-lease`.

</details>

### Pull Request و Workflowها

<details><summary>**۲۱. Pull Request چیست و در کدام لایه قرار دارد؟**</summary>

PR یک قابلیتِ **پلتفرم** (GitHub/GitLab) است، نه خودِ Git: پیشنهادِ ادغامِ یک شاخه در دیگری همراه با محیطِ review (کامنت، تأیید، CI). خودِ Git فقط `push` و `merge` می‌فهمد؛ PR لایه‌ی همکاری و کنترلِ کیفیت روی آن است.

</details>

<details><summary>**۲۲. Draft PR کِی به درد می‌خورد؟**</summary>

وقتی کار هنوز تمام نشده ولی می‌خواهی زود بازخورد بگیری یا CI روی کارت اجرا شود، بدونِ اینکه کسی اشتباهاً merge کند. تا «Ready for review» نشود، دکمه‌ی merge قفل است.

</details>

<details><summary>**۲۳. GitHub Flow، Git Flow و trunk-based هرکدام کِی مناسب‌اند؟**</summary>

GitHub Flow: `main` همیشه deployable، هر کار یک شاخه‌ی کوتاه و PR — برای وب/سرویس با continuous deployment. Git Flow: شاخه‌های develop/release/hotfix جدا — برای محصولاتِ نسخه‌دار با releaseهای منظم و پشتیبانیِ همزمانِ چند نسخه. Trunk-based: همه نزدیک به main با شاخه‌های خیلی کوتاه و feature flag — برای تیم‌های بزرگ با CI/CD قوی و کمترین merge conflict.

</details>

<details><summary>**۲۴. فرقِ سه استراتژیِ merge در PR (merge / squash / rebase) چیست؟**</summary>

*Merge commit*: کلِ تاریخِ شاخه + یک merge commit (همه‌چیز حفظ می‌شود ولی شلوغ). *Squash and merge*: همه‌ی commitهای شاخه در یک commitِ تمیز روی main (تاریخچه‌ی مرتب، ولی جزئیاتِ داخلی گم می‌شود). *Rebase and merge*: commitها خطی روی main بدون merge commit. انتخاب بستگی به سلیقه‌ی تیم برای تمیزیِ تاریخچه دارد.

</details>

### ابزارها و سناریوهای نجات

<details><summary>**۲۵. کاربردِ `git stash` چیست؟**</summary>

کنار گذاشتنِ موقتِ تغییراتِ ناتمام تا working dir تمیز شود (مثلاً برای جابه‌جایی فوری به شاخه‌ی دیگر یا pull). بعد با `git stash pop` برمی‌گردانی. `-u` فایل‌های untracked را هم شامل می‌شود.

</details>

<details><summary>**۲۶. `cherry-pick` چه می‌کند و یک کاربردِ واقعی‌اش؟**</summary>

یک commitِ مشخص از شاخه‌ای دیگر را روی شاخه‌ی فعلی بازتولید می‌کند (با SHA تازه). کاربردِ کلاسیک: انتقالِ یک hotfix از یک release branch به `main` (یا برعکس) بدون merge کاملِ شاخه.

</details>

<details><summary>**۲۷. tag سبک (lightweight) در برابر annotated چه فرقی دارد؟ کدام برای release؟**</summary>

lightweight فقط یک اشاره‌گرِ نام‌دار به یک commit است. annotated (`-a`) یک object کاملِ Git با نویسنده، تاریخ، پیام و امکانِ امضای GPG است. برای releaseهای رسمی **annotated** توصیه می‌شود.

</details>

<details><summary>**۲۸. `git bisect` چطور کمک می‌کند یک باگ را پیدا کنی؟**</summary>

با جستجوی دودویی روی تاریخچه: یک commitِ `good` (سالم) و یک `bad` (خراب) معرفی می‌کنی؛ Git هر بار commitِ وسط را checkout می‌کند تا تست کنی و `good`/`bad` بزنی. در میانِ N commit تنها ~log₂(N) قدم لازم است. با `git bisect run ./test.sh` کاملاً خودکار می‌شود.

</details>

<details><summary>**۲۹. سناریو: همکار روی شاخه‌ی مشترک force-push کرد و چند commitِ من ناپدید شد. چطور برشان می‌گردانم؟**</summary>

commitها هنوز در `git reflog` محلیِ من زنده‌اند (یا با `git fsck --lost-found`). SHAی آخرین commitِ خوبم را از reflog پیدا می‌کنم، یک شاخه‌ی نجات می‌سازم (`git branch rescue <sha>`)، و در صورتِ نیاز با `git push --force-with-lease` شاخه‌ی مشترک را به حالتِ درست برمی‌گردانم. درس: روی شاخه‌ی مشترک هرگز `--force` خام نزنید.

</details>

<details><summary>**۳۰. سناریو: یک کلیدِ API محرمانه را ۳ commit پیش commit و push کرده‌ای. الان چه می‌کنی؟**</summary>

اول و فوری: **کلید را revoke/rotate کن** — چون به محضِ push، عملاً لو رفته و حتی پاک کردنش از تاریخچه آن را امن نمی‌کند. سپس کلید را از تاریخچه‌ی کل repo پاک کن (`git filter-repo` یا BFG)، که تاریخچه را بازنویسی می‌کند و نیازمندِ `--force-with-lease` و هماهنگی با تیم است (همه باید دوباره clone/rebase کنند). در آخر آن فایل/الگو را به `.gitignore` اضافه کن تا تکرار نشود.

</details>

<details><summary>**۳۱. سناریو: `git reset --hard` زدی و کلی کارِ commit‌شده پرید. راهی هست؟**</summary>

بله — `reset --hard` فقط اشاره‌گر را جابه‌جا می‌کند؛ commitها هنوز در object store و در `git reflog` هستند. `git reflog` بزن، SHAی قبلی را پیدا کن، و `git reset --hard <sha>` (یا `git branch rescue <sha>`). تنها چیزی که قابلِ نجات نیست، کارِ **هرگز commit‌نشده‌ای** است که `--hard` از working dir پاک کرد.

</details>

<details><summary>**۳۲. سناریو: پیامِ آخرین commit غلط است (و هنوز push نکرده‌ای). و اگر push کرده باشی چه؟**</summary>

اگر push نشده: `git commit --amend -m "پیامِ درست"`. اگر push شده و شاخه مشترک است: amend تاریخ را بازنویسی می‌کند، پس یا با `--force-with-lease` و هماهنگیِ تیم push کن، یا (اگر مشترک و حساس است) بی‌خیالِ پیام شو و در commitِ بعدی توضیح بده — بازنویسیِ تاریخِ مشترک ارزشِ دردسرش را ندارد.

</details>
