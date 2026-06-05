# سطح ۳ — توضیح فنی: Data Structures, Algorithms & Complexity ⚙️

> دیگر قصه‌ای در کار نیست. این سند مرجع فنی است — همان چیزهایی که در مصاحبه و کارِ واقعی لازم داری. هر بخش با معادلِ قصه‌ای‌اش شروع می‌شود تا ذهنت وصل بماند.

---

## ۱. پیچیدگی و نماد Big O

«اگر هزار تا بود چی؟» = نرخِ رشد. Big O **کرانِ بالایی** رشدِ زمان یا فضا را بر حسبِ اندازه‌ی ورودی `n` توصیف می‌کند، **مستقل از سخت‌افزار**. ضریب‌های ثابت و جمله‌های مرتبه‌پایین حذف می‌شوند: `O(3n² + 5n + 9) → O(n²)`.

| Big O | اسم | مثالِ واقعی | n=۱۰ | n=۱٬۰۰۰ | n=۱٬۰۰۰٬۰۰۰ |
|---|---|---|---|---|---|
| `O(1)` | ثابت | دسترسی به اندیسِ آرایه، lookup در hash | ۱ | ۱ | ۱ |
| `O(log n)` | لگاریتمی | binary search، عملیاتِ BST متوازن | ~۳ | ~۱۰ | ~۲۰ |
| `O(n)` | خطی | linear search، یک پیمایشِ آرایه | ۱۰ | ۱٬۰۰۰ | ۱۰⁶ |
| `O(n log n)` | خطی-لگاریتمی | merge/quick/heap sort | ~۳۳ | ~۱۰⁴ | ~۲×۱۰⁷ |
| `O(n²)` | درجه‌دو | bubble/insertion sort، دو حلقه‌ی تو در تو | ۱۰۰ | ۱۰⁶ | ۱۰¹² |
| `O(2ⁿ)` | نمایی | فیبوناچیِ بازگشتیِ ساده، زیرمجموعه‌ها | ~۱۰³ | غیرممکن | غیرممکن |
| `O(n!)` | فاکتوریل | جایگشت‌ها، brute-force TSP | ~۳.۶×۱۰⁶ | غیرممکن | غیرممکن |

- **Time Complexity** = تعدادِ عملیات بر حسبِ `n`.
- **Space Complexity** = حافظه‌ی **اضافه‌ی** مصرفی (auxiliary)، جدا از خودِ ورودی. recursion هم از طریقِ call stack فضا مصرف می‌کند (عمقِ بازگشت).
- سه کرانِ مجزا: **O** (بدترین/کرانِ بالا)، **Ω** (بهترین)، **Θ** (دقیق، وقتی بالا و پایین یکی‌اند). در مصاحبه معمولاً منظور از «complexity» همان **worst-case O** است.

> نکته‌ی مصاحبه: «**Amortized**» با «**average**» فرق دارد. amortized یعنی میانگینِ هزینه روی یک **دنباله از عملیات** با تضمین (مثلِ `append` در dynamic array)؛ average یعنی میانگینِ آماری روی ورودی‌های تصادفی (مثلِ quick sort).

---

## ۲. ساختارهای داده

### آرایه (Array / Dynamic Array)

«ردیفِ شماره‌دار». حافظه‌ی پیوسته با اندیسِ صفر‌مبنا. `list` پایتون یک **dynamic array** است که با پُرشدن، ظرفیتش را (معمولاً) دو برابر می‌کند؛ به همین دلیل `append` به‌طور **amortized O(1)** است هرچند گاهی یک کپیِ O(n) رخ می‌دهد.

```python
a = [10, 20, 30]
a[1]            # دسترسی O(1)
a.append(40)    # amortized O(1)
a.insert(0, 5)  # O(n) — همه شیفت می‌خورند
a.pop()         # از ته O(1) ؛ a.pop(0) از سر O(n)
```

### لیست پیوندی (Linked List)

«عروسک‌های دست‌به‌دست». هر node یک `data` و یک یا دو pointer دارد (singly / doubly).

```python
class Node:
    __slots__ = ("data", "next")
    def __init__(self, data, nxt=None):
        self.data, self.next = data, nxt
```

- مزیت: درج/حذفِ O(1) **اگر** اشاره‌گر به محل را داشته باشی؛ بدونِ شیفت.
- عیب: دسترسیِ تصادفی O(n)، cache locality بد (node‌ها پراکنده در حافظه)، سربارِ pointer.
- **Doubly linked list** پایه‌ی `deque`؛ پیمایشِ دوطرفه و حذفِ O(1) با اشاره‌گرِ گره.

### پشته (Stack) — LIFO

«بشقاب‌های روی هم». `push`/`pop`/`peek` همگی O(1). با `list` (ته) یا `collections.deque` پیاده می‌شود. کاربردها: call stack، undo، تطبیقِ پرانتز، DFS، ارزیابیِ عبارتِ پسوندی.

### صف (Queue) — FIFO

«صفِ سُرسُره». `enqueue`/`dequeue` باید O(1) باشند → از `collections.deque` استفاده کن، نه `list` (که `pop(0)` آن O(n) است). انواع: **deque** (دوطرفه)، **circular buffer** (ظرفیتِ ثابت)، **priority queue** (با heap، خروجی بر اساسِ اولویت نه ترتیبِ ورود).

```python
import heapq
pq = []
heapq.heappush(pq, (2, "task-B"))   # (priority, item)
heapq.heappush(pq, (1, "task-A"))
heapq.heappop(pq)                   # (1, 'task-A') — کم‌ترین اول، O(log n)
```

### جدول هش (Hash Table / dict / set)

«کمدِ جادویی». `hash(key)` → اندیسِ bucket. درج/حذف/lookup به‌طور **average O(1)**.

- **Collision** (دو کلید، یک bucket) اجتناب‌ناپذیر است (اصلِ لانه‌کبوتری). دو استراتژی:
  - **Separate Chaining**: هر bucket یک لیست/درخت؛ Java 8+ پس از آستانه‌ای bucket را به درختِ متوازن تبدیل می‌کند تا بدترین حالت O(log n) شود.
  - **Open Addressing** (linear/quadratic probing، double hashing): همان آرایه، probe تا خانه‌ی خالی. حذف نیازمندِ tombstone است. پایتون از این استفاده می‌کند.
- **Load Factor** = `n/buckets`؛ با عبور از آستانه (مثلاً ۰.۶۶ در پایتون)، **resize + rehash** رخ می‌دهد (O(n) ولی amortized).
- **degrade به O(n)**: تابعِ هشِ بد یا کلیدهای متخاصم (Hash DoS) → همه در یک bucket.
- کلیدها باید **hashable و immutable** باشند (`__hash__` و `__eq__` سازگار).

### درخت (Tree) و درختِ جستجوی دودویی (BST)

«درختِ خانوادگی». درختِ ریشه‌دار بدونِ دور. **BST**: زیردرختِ چپ < گره < زیردرختِ راست.

```python
class TreeNode:
    def __init__(self, val):
        self.val, self.left, self.right = val, None, None

def bst_search(root, target):           # O(h) — h ارتفاع
    while root:
        if target == root.val: return root
        root = root.left if target < root.val else root.right
    return None
```

- پیمایش‌ها: **In-order** (چپ-ریشه-راست → خروجیِ **مرتب** در BST)، **Pre-order** (کپیِ ساختار)، **Post-order** (آزادسازی/حذف)، **Level-order** (BFS با صف).
- **چرا توازن مهم است**: عملیاتِ BST همگی O(h)اند. در BSTِ متوازن `h ≈ log n` → O(log n)؛ اما با درجِ مرتب، درخت به یک زنجیره دژنره می‌شود، `h = n` → O(n).
- **درخت‌های خودمتوازن**: **AVL** (سخت‌گیرتر در توازن، جستجوی سریع‌تر)، **Red-Black** (درج/حذفِ ارزان‌تر، انتخابِ کتابخانه‌ها مثل `std::map` و `TreeMap`). **B-Tree/B+Tree** پایه‌ی ایندکسِ دیتابیس و فایل‌سیستم (fan-out بالا، دوست‌دارِ دیسک). **Heap** درختِ کاملِ مبتنی بر آرایه برای priority queue.

### گراف (Graph)

«نقشه‌ی دوستی‌ها». `V` رأس، `E` یال. **Directed/Undirected**، **Weighted/Unweighted**، **Cyclic/Acyclic** (DAG).

| نمایش | حافظه | بررسیِ یالِ (u,v) | پیمایشِ همسایه‌ها | مناسبِ |
|---|---|---|---|---|
| **Adjacency List** | O(V+E) | O(deg u) | O(deg u) | گرافِ تُنُک (Sparse) — پیش‌فرض |
| **Adjacency Matrix** | O(V²) | O(1) | O(V) | گرافِ متراکم (Dense)، بررسیِ یالِ مکرر |

```python
from collections import defaultdict
g = defaultdict(list)
g[0] += [1, 2]; g[1] += [0]; g[2] += [0]   # adjacency list، بدون‌جهت
```

الگوریتم‌های کلیدی: **BFS/DFS** (پیمایش)، **Dijkstra** (کوتاه‌ترین مسیر، وزنِ نامنفی، O((V+E)log V) با heap)، **Bellman-Ford** (وزنِ منفی)، **Topological Sort** (DAG)، **Union-Find** (مؤلفه‌های همبند، near-O(1) amortized).

---

## ۳. الگوریتم‌ها

### مرتب‌سازی (Sorting)

| الگوریتم | میانگین | بدترین | فضا | پایدار؟ | یادداشت |
|---|---|---|---|---|---|
| **Bubble** | O(n²) | O(n²) | O(1) | ✅ | فقط آموزشی |
| **Insertion** | O(n²) | O(n²) | O(1) | ✅ | عالی برای nِ کوچک یا تقریباً مرتب |
| **Selection** | O(n²) | O(n²) | O(1) | ❌ | کم‌ترین تعدادِ swap |
| **Merge** | O(n log n) | O(n log n) | O(n) | ✅ | تضمینی، divide & conquer، خوب برای linked list/external sort |
| **Quick** | O(n log n) | O(n²) | O(log n) | ❌ | in-place، سریع‌ترین در عمل؛ pivotِ بد → O(n²) |
| **Heap** | O(n log n) | O(n log n) | O(1) | ❌ | تضمینی + in-place، ثابتِ بزرگ‌تر |

- **کرانِ پایینِ مرتب‌سازیِ مقایسه‌ای**: `Ω(n log n)`. عبور از آن فقط با مرتب‌سازیِ **غیرمقایسه‌ای** (Counting/Radix/Bucket sort) برای دامنه‌ی محدود → O(n+k).
- **Stable** یعنی عناصرِ مساوی ترتیبِ نسبیِ اولیه‌شان را حفظ می‌کنند (مهم در مرتب‌سازیِ چندکلیده).
- **Timsort** (پایتون، Java) = merge + insertion هیبریدی، adaptive، پایدار، O(n) روی دادهٔ تقریباً مرتب.

```python
def quicksort(a):
    if len(a) <= 1: return a
    p = a[len(a)//2]
    return (quicksort([x for x in a if x < p]) + [x for x in a if x == p]
            + quicksort([x for x in a if x > p]))
```

### جستجو (Searching)

- **Linear**: O(n)، روی هر داده‌ای.
- **Binary**: O(log n)، فقط روی دادهٔ **مرتب**. مراقبِ overflow در `mid` و حلقه‌ی بی‌پایان از مرزها باش. در پایتون: `bisect`.
- **BFS** (صف، O(V+E)): کوتاه‌ترین مسیر در گرافِ **بی‌وزن**؛ پیمایشِ لایه‌به‌لایه.
- **DFS** (پشته/recursion، O(V+E)): کشفِ مسیر، تشخیصِ دور، topological sort، مؤلفه‌های همبند.

```python
def binary_search(a, t):
    lo, hi = 0, len(a) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if a[mid] == t: return mid
        if a[mid] < t: lo = mid + 1
        else:          hi = mid - 1
    return -1
```

### بازگشت (Recursion)

«عروسکِ تو در تو». هر فراخوانی یک **stack frame** روی call stack می‌سازد.

- **Base case** اجباری است؛ نبودش → `RecursionError`/Stack Overflow. پایتون سقفِ پیش‌فرض ~۱۰۰۰ دارد (`sys.setrecursionlimit`).
- **Recursive step** باید به base case نزدیک شود.
- **Tail recursion**: فراخوانیِ بازگشتی آخرین عمل است؛ بعضی زبان‌ها بهینه‌اش می‌کنند (TCO) — **پایتون نمی‌کند**، پس عمقِ زیاد را به حلقه تبدیل کن.
- هر بازگشتی به حلقه + پشته‌ی صریح قابلِ تبدیل است.

### برنامه‌نویسیِ پویا (Dynamic Programming)

«دفترچه‌ی یادداشت». وقتی مسئله **Optimal Substructure** + **Overlapping Subproblems** دارد.

- **Memoization** (top-down): بازگشت + کش. طبیعی، فقط زیرمسئله‌های لازم.
- **Tabulation** (bottom-up): پُرکردنِ جدول از پایه. بدونِ سربارِ recursion، اغلب بهینه‌سازیِ فضا ممکن.

```python
# Fibonacci: O(2ⁿ) ساده → O(n) با DP
def fib(n):
    a, b = 0, 1
    for _ in range(n): a, b = b, a + b
    return a            # O(n) زمان، O(1) فضا

# Coin Change (کم‌ترین تعدادِ سکه): O(amount × len(coins))
def coin_change(coins, amount):
    dp = [0] + [float('inf')] * amount
    for m in range(1, amount + 1):
        for c in coins:
            if c <= m:
                dp[m] = min(dp[m], dp[m - c] + 1)
    return dp[amount] if dp[amount] != float('inf') else -1
```

> نکته‌ی مصاحبه: قبل از DP، **حالت** (state) و **رابطه‌ی بازگشتی** (recurrence) را روی کاغذ بنویس؛ سپس تصمیم بگیر memoization یا tabulation. تشخیصِ DP = «انتخاب در هر گام + همان زیرمسئله بارها تکرار می‌شود».

---

## 🔬 آزمایشگاه

کد زیر کامل و runnable است (Python 3). پشته/صف می‌سازد، linear را با binary روی لیستِ یک‌میلیونی با `timeit` می‌سنجد، و فیبوناچیِ memoized را با ساده مقایسه می‌کند.

```python
from collections import deque
import timeit, bisect

# ── ۱) پیاده‌سازیِ Stack (LIFO) و Queue (FIFO) ──
class Stack:
    def __init__(self): self._d = []
    def push(self, x): self._d.append(x)      # O(1)
    def pop(self):     return self._d.pop()   # O(1)
    def peek(self):    return self._d[-1]
    def is_empty(self): return not self._d

class Queue:
    def __init__(self): self._d = deque()
    def enqueue(self, x): self._d.append(x)     # O(1)
    def dequeue(self):    return self._d.popleft()  # O(1) — deque، نه list!
    def is_empty(self):   return not self._d

s = Stack();  [s.push(c) for c in "ABC"]
print("Stack pop  :", [s.pop() for _ in range(3)])      # ['C','B','A']  (LIFO)
q = Queue();  [q.enqueue(c) for c in "ABC"]
print("Queue deq  :", [q.dequeue() for _ in range(3)])  # ['A','B','C']  (FIFO)

# ── ۲) Linear O(n) در برابر Binary O(log n) روی لیستِ مرتبِ بزرگ ──
N = 1_000_000
data = list(range(N))
target = N - 1                       # بدترین حالت برای linear

def linear(a, t):
    for i, v in enumerate(a):
        if v == t: return i
    return -1

def binary(a, t):                    # نیازمندِ دادهٔ مرتب
    i = bisect.bisect_left(a, t)
    return i if i < len(a) and a[i] == t else -1

t_lin = timeit.timeit(lambda: linear(data, target), number=10)
t_bin = timeit.timeit(lambda: binary(data, target), number=10)
print(f"Linear : {t_lin:.4f}s")
print(f"Binary : {t_bin:.6f}s   (~{t_lin/t_bin:,.0f}x faster)")

# ── ۳) Fibonacci: ساده O(2ⁿ) در برابر memoized O(n) ──
def fib_naive(n):
    if n <= 1: return n
    return fib_naive(n-1) + fib_naive(n-2)

def fib_memo(n, memo={}):
    if n <= 1: return n
    if n in memo: return memo[n]
    memo[n] = fib_memo(n-1, memo) + fib_memo(n-2, memo)
    return memo[n]

n = 35
t_naive = timeit.timeit(lambda: fib_naive(n), number=1)
t_memo  = timeit.timeit(lambda: fib_memo(n),  number=1)
print(f"fib_naive({n}): {t_naive:.4f}s")
print(f"fib_memo({n}) : {t_memo:.8f}s  (~{t_naive/max(t_memo,1e-9):,.0f}x faster)")
```

خروجیِ نمونه (اعداد بسته به ماشین فرق می‌کنند، اما **نسبت‌ها** پایدارند):

```
Stack pop  : ['C', 'B', 'A']
Queue deq  : ['A', 'B', 'C']
Linear : 0.1670s
Binary : 0.000032s   (~5,157x faster)
fib_naive(35): 0.6069s
fib_memo(35) : 0.00001191s  (~50,946x faster)
```

درسِ آزمایشگاه: تفاوتِ O(n) با O(log n) و O(2ⁿ) با O(n) در عمل **هزاران برابر** است — نه یک نکته‌ی تئوریک.

---

## ✅ چک‌لیست تسلط

اگر بتوانی به این‌ها جواب بدهی، این فصل را بلدی:

- [ ] کِی آرایه، کِی لیست پیوندی؟ هزینه‌ی دسترسی/درج/حذفِ هرکدام؟
- [ ] چرا `list.pop(0)` بد است و `deque.popleft()` خوب؟
- [ ] LIFO در برابر FIFO — یک کاربردِ واقعی برای هرکدام بگو.
- [ ] چرا hash lookup به‌طور amortized O(1) است و **کِی** به O(n) دژنره می‌شود؟
- [ ] collision را با chaining و open addressing چطور حل می‌کنی؟
- [ ] چرا BSTِ نامتوازن O(n) می‌شود و AVL/Red-Black چه می‌کنند؟
- [ ] In-order traversal یک BST چه چیزی می‌دهد؟
- [ ] adjacency list در برابر matrix — حافظه و trade-off؟
- [ ] BFS چه ساختاری می‌خواهد، DFS چه؟ کدام برای کوتاه‌ترین مسیرِ بی‌وزن؟
- [ ] merge در برابر quick: پایداری، فضا، بدترین حالت؟ کرانِ پایینِ Ω(n log n) چرا؟
- [ ] دو جزءِ اجباریِ recursion؟ ربطش با call stack؟
- [ ] دو شرطِ لازم برای DP؟ فرقِ memoization و tabulation؟
- [ ] جدولِ Big O را از O(1) تا O(2ⁿ) با یک مثالِ واقعی برای هر کدام بازگو کن.
- [ ] فرقِ amortized و average؟ فرقِ time و space complexity؟

---

## 🎯 سوالات مصاحبه

### ساختارهای داده — آرایه و لیست پیوندی

<details><summary>**فرقِ آرایه و لیست پیوندی در یک جمله چیست و کِی کدام را انتخاب می‌کنی؟**</summary>

آرایه دسترسیِ تصادفیِ O(1) و locality عالی دارد ولی درج/حذفِ وسط O(n) است؛ لیست پیوندی درج/حذفِ O(1) (با اشاره‌گر) دارد ولی دسترسی O(n) و cache-locality بد. اگر زیاد ایندکس می‌زنی → آرایه؛ اگر زیاد از وسط درج/حذف می‌کنی و اشاره‌گر داری → لیست پیوندی. در عمل به‌خاطرِ cache، آرایه اغلب حتی برای درج/حذف هم برنده است.

</details>

<details><summary>**چطور یک دور (cycle) را در لیست پیوندی تشخیص می‌دهی؟**</summary>

الگوریتمِ **Floyd (لاک‌پشت و خرگوش)**: دو اشاره‌گر، `slow` یک قدم و `fast` دو قدم. اگر دور وجود داشته باشد، `fast` به `slow` می‌رسد؛ اگر `fast` به `None` برسد، دوری نیست. O(n) زمان، O(1) فضا. برای یافتنِ **شروعِ** دور: بعد از برخورد، یکی را به head برگردان و هر دو را یک‌قدمی پیش ببر؛ نقطه‌ی برخوردِ دوم شروعِ دور است.

```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
        if slow is fast: return True
    return False
```

</details>

<details><summary>**چطور یک لیست پیوندی را معکوس می‌کنی؟**</summary>

سه اشاره‌گر `prev, curr, nxt`؛ در هر گام `next` گرهِ فعلی را به `prev` برگردان و جلو برو. O(n) زمان، O(1) فضا.

```python
def reverse(head):
    prev = None
    while head:
        head.next, prev, head = prev, head, head.next
    return prev
```

</details>

<details><summary>**dynamic array چطور amortized O(1) برای append می‌دهد با اینکه گاهی کپیِ O(n) دارد؟**</summary>

با **doubling**: وقتی پر می‌شود ظرفیت دو برابر می‌شود. هزینه‌ی کلِ n درج روی هم O(n) است (سری هندسیِ n + n/2 + n/4 + ... = 2n)، پس میانگینِ هر درج O(1). به این تحلیل می‌گویند aggregate/amortized analysis.

</details>

### ساختارهای داده — پشته و صف

<details><summary>**با دو پشته چطور یک صف می‌سازی؟**</summary>

`in_stack` برای enqueue. هنگام dequeue، اگر `out_stack` خالی بود، همه‌ی `in_stack` را در آن خالی کن (ترتیب معکوس می‌شود → FIFO) و از `out_stack` pop کن. هر عنصر حداکثر یک بار جابه‌جا می‌شود → amortized O(1).

</details>

<details><summary>**چطور با پشته تشخیص می‌دهی پرانتزها متوازن‌اند؟**</summary>

روی پرانتزِ باز push کن؛ روی بسته، بالای پشته را pop کن و چک کن جفت باشد. در پایان پشته باید خالی باشد. O(n) زمان، O(n) فضا.

</details>

<details><summary>**Stack و Queue را در دنیای واقعی کجا می‌بینی؟**</summary>

Stack: call stack تابع‌ها، undo/redo، back button، DFS، ارزیابیِ عبارت. Queue: زمان‌بندیِ پردازه، صفِ پیام (Kafka/RabbitMQ)، صفِ چاپ، BFS، rate limiting. Priority queue: Dijkstra، زمان‌بندیِ بر اساسِ اولویت، event simulation.

</details>

### ساختارهای داده — جدول هش

<details><summary>**چرا hash table lookup به‌طور amortized O(1) است و کِی دژنره می‌شود؟**</summary>

تابعِ هش کلید را مستقیم به یک bucket نگاشت می‌کند، پس به‌طور میانگین جستجو ثابت است؛ amortized به‌خاطرِ هزینه‌ی گاه‌به‌گاهِ resize/rehash است. **دژنره به O(n)** وقتی رخ می‌دهد که اکثرِ کلیدها به یک bucket بیفتند: تابعِ هشِ ضعیف، load factor خیلی بالا، یا حمله‌ی متخاصم (Hash-Flooding DoS). دفاع: تابعِ هشِ خوب + resize خودکار + (در زبان‌هایی مثلِ Java) تبدیلِ bucket به درختِ متوازن.

</details>

<details><summary>**collision چیست و چطور حلش می‌کنی؟**</summary>

دو کلیدِ مختلف، یک اندیسِ هش (طبقِ اصلِ لانه‌کبوتری اجتناب‌ناپذیر). **Separate chaining**: هر bucket یک لیست/درخت. **Open addressing**: probe به خانه‌ی خالیِ بعدی (linear/quadratic/double hashing)، با tombstone برای حذف. chaining در load factor بالا مقاوم‌تر؛ open addressing cache-friendlyتر و کم‌حافظه‌تر.

</details>

<details><summary>**چه چیزی می‌تواند کلیدِ hash table باشد؟ چرا list نمی‌تواند؟**</summary>

کلید باید **hashable** باشد یعنی `__hash__` ثابت و `__eq__` سازگار داشته باشد — که عملاً یعنی **immutable**. `list` تغییرپذیر است؛ اگر بعد از درج عوض شود، hashش تغییر می‌کند و دیگر در bucket درست پیدا نمی‌شود. tuple (از عناصرِ immutable) قابلِ هش است.

</details>

<details><summary>**چطور تشخیص می‌دهی یک آرایه عنصرِ تکراری دارد؟ (O(n))**</summary>

یک `set` بساز و حین پیمایش هر عنصر را چک/اضافه کن؛ اولین تکرار جواب است. O(n) زمان، O(n) فضا — مبادله‌ی فضا برای زمان در برابرِ مرتب‌سازیِ O(n log n) با فضای O(1).

</details>

### ساختارهای داده — درخت

<details><summary>**چرا BST می‌تواند O(n) شود و راه‌حل چیست؟**</summary>

عملیاتِ BST همگی O(ارتفاع)اند. با درجِ داده‌ی **مرتب**، درخت به یک زنجیره دژنره می‌شود (`h=n`)، پس O(n). راه‌حل: درختِ **خودمتوازن** — AVL یا Red-Black — که با rotation ارتفاع را نزدیکِ `log n` نگه می‌دارد و O(log n) را تضمین می‌کند.

</details>

<details><summary>**چهار پیمایشِ درخت و کاربردشان؟**</summary>

In-order (چپ-ریشه-راست): در BST خروجیِ **مرتب**. Pre-order (ریشه-چپ-راست): کپی/serialize ساختار. Post-order (چپ-راست-ریشه): حذف/آزادسازی، محاسبه‌ی پایین‌به‌بالا. Level-order (BFS با صف): پیمایشِ لایه‌به‌لایه.

</details>

<details><summary>**چطور بررسی می‌کنی یک درخت BSTِ معتبر است؟**</summary>

با DFS و حملِ بازه‌ی مجاز `(low, high)`؛ هر گره باید `low < val < high` باشد، و برای فرزندِ چپ `high=val` و راست `low=val` می‌شود. صرفِ مقایسه‌ی گره با فرزندانِ مستقیمش **کافی نیست** (خطای کلاسیک). جایگزین: in-order traversal باید اکیداً صعودی باشد.

</details>

<details><summary>**فرقِ BST، Heap و B-Tree؟**</summary>

BST: ترتیبِ چپ<ریشه<راست برای جستجوی O(log n). Heap: فقط ترتیبِ والد-فرزند (min/max)، برای دسترسیِ سریع به کمینه/بیشینه و priority queue، نه جستجوی دلخواه. B-Tree: درختِ چندشاخه با fan-out بالا برای ایندکسِ دیتابیس/دیسک (کاهشِ تعدادِ خواندنِ دیسک).

</details>

### ساختارهای داده — گراف

<details><summary>**adjacency list یا matrix — کدام و کِی؟**</summary>

List: حافظه‌ی O(V+E)، عالی برای گرافِ تُنُک و پیمایشِ همسایه‌ها (اکثرِ گراف‌های واقعی). Matrix: حافظه‌ی O(V²) ولی بررسیِ وجودِ یالِ (u,v) در O(1) و خوب برای گرافِ متراکم یا الگوریتم‌هایی که زیاد یال چک می‌کنند.

</details>

<details><summary>**چطور دور را در گرافِ جهت‌دار تشخیص می‌دهی؟**</summary>

DFS با **سه رنگ** (white/gray/black) یا مجموعه‌ی «در مسیرِ فعلی»: اگر در حینِ DFS به گرهِ gray (در حالِ پردازش) برخوردی، **back edge** و یعنی دور. در گرافِ بی‌جهت کافی است به گرهِ دیده‌شده‌ای جز والدِ مستقیم برسی. راهِ دیگرِ DAG: اگر **topological sort** ناممکن شد، دور دارد.

</details>

<details><summary>**چه زمانی BFS و چه زمانی DFS؟**</summary>

BFS: کوتاه‌ترین مسیر در گرافِ **بی‌وزن**، پیمایشِ لایه‌ای، نزدیک‌ترین هدف — از صف استفاده می‌کند. DFS: کشفِ وجودِ مسیر، تشخیصِ دور، topological sort، مؤلفه‌های همبند، مسائلِ backtracking — از پشته/recursion. هر دو O(V+E).

</details>

<details><summary>**Dijkstra چه می‌کند و محدودیتش چیست؟**</summary>

کوتاه‌ترین مسیر از یک مبدأ در گرافِ با وزنِ **نامنفی**، با min-heap در O((V+E)log V). با وزنِ **منفی** نادرست می‌شود → آنجا **Bellman-Ford** (O(VE)، تشخیصِ دورِ منفی هم می‌دهد).

</details>

### الگوریتم‌ها — مرتب‌سازی و جستجو

<details><summary>**merge sort یا quick sort — تفاوت‌های عملی؟**</summary>

Merge: همیشه O(n log n)، **پایدار**، اما O(n) حافظه‌ی اضافه؛ خوب برای linked list و external sort. Quick: میانگین O(n log n) و **in-place** (O(log n) استک) و در عمل سریع‌تر به‌خاطرِ locality، اما بدترین حالت O(n²) با pivotِ بد (دفاع: pivotِ تصادفی یا median-of-three). quick ناپایدار است.

</details>

<details><summary>**چرا هیچ مرتب‌سازیِ مقایسه‌ای نمی‌تواند بهتر از O(n log n) باشد؟**</summary>

درختِ تصمیمِ هر الگوریتمِ مقایسه‌ای باید `n!` خروجیِ ممکن (جایگشت‌ها) را تفکیک کند؛ ارتفاعِ چنین درختی حداقل `log₂(n!) = Θ(n log n)` است. عبور از این کران فقط با مرتب‌سازیِ غیرمقایسه‌ای (Counting/Radix) برای کلیدهای با دامنه‌ی محدود ممکن است → O(n+k).

</details>

<details><summary>**پیش‌شرطِ binary search چیست و دامِ رایجش؟**</summary>

داده باید **مرتب** باشد. دام‌ها: محاسبه‌ی `mid` با overflow در زبان‌های با intِ ثابت (از `lo + (hi-lo)//2` استفاده کن)، شرطِ مرزِ حلقه (`<=` در برابرِ `<`)، و به‌روزرسانیِ نادرستِ `lo/hi` که حلقه‌ی بی‌پایان می‌سازد. O(log n).

</details>

<details><summary>**Timsort چیست و چرا پایتون از آن استفاده می‌کند؟**</summary>

هیبریدِ merge و insertion sort که **run**های مرتبِ موجود را شناسایی و ادغام می‌کند. **adaptive** (روی دادهٔ تقریباً مرتب O(n))، **پایدار**، بدترین حالت O(n log n). برای دادهٔ واقعی که اغلب نیمه‌مرتب است بسیار سریع — پیش‌فرضِ `sorted()` و `list.sort()`.

</details>

### الگوریتم‌ها — بازگشت و DP

<details><summary>**دو جزءِ اجباریِ هر تابعِ بازگشتی چیست؟**</summary>

**Base case** (شرطِ توقف که بدونِ بازگشت برمی‌گردد) و **recursive step** (که مسئله را کوچک‌تر کرده و خودش را روی آن صدا می‌زند، با حرکت به‌سمتِ base case). نبودِ base case یا عدمِ همگرایی به آن → بازگشتِ بی‌پایان و Stack Overflow.

</details>

<details><summary>**recursion چه ربطی به call stack دارد و خطرش چیست؟**</summary>

هر فراخوانی یک stack frame (پارامترها، متغیرهای محلی، آدرسِ بازگشت) روی call stack می‌گذارد. عمقِ زیاد → سرریزِ پشته (در پایتون `RecursionError` در ~۱۰۰۰). راه‌حل: تبدیل به حلقه با پشته‌ی صریح، یا در زبان‌هایی با TCO، tail recursion (پایتون TCO ندارد).

</details>

<details><summary>**دو شرطِ لازم برای استفاده از DP چیست؟**</summary>

**Optimal substructure** (جوابِ بهینه از جوابِ بهینه‌ی زیرمسئله‌ها ساخته می‌شود) و **overlapping subproblems** (همان زیرمسئله‌ها بارها تکرار می‌شوند). اگر زیرمسئله‌ها تکراری نباشند، divide & conquer ساده کافی است، نه DP.

</details>

<details><summary>**memoization یا tabulation — تفاوت؟**</summary>

Memoization (top-down): بازگشت + کش؛ فقط زیرمسئله‌های لازم را حساب می‌کند، طبیعی برای کدنویسی، اما سربارِ recursion و خطرِ stack overflow. Tabulation (bottom-up): جدول را از پایه پر می‌کند؛ بدونِ recursion، اغلب اجازه‌ی بهینه‌سازیِ فضا (مثلاً فیبوناچی با دو متغیر، O(1)).

</details>

<details><summary>**چرا فیبوناچیِ بازگشتیِ ساده O(2ⁿ) است و DP چطور آن را O(n) می‌کند؟**</summary>

هر فراخوانی دو فراخوانی می‌زند و زیرمسئله‌های مشترکِ فراوان دوباره حساب می‌شوند → درختِ نمایی. با memoization هر `fib(k)` فقط یک بار محاسبه و کش می‌شود → O(n) زمان. با tabulation و نگه‌داشتنِ فقط دو مقدارِ آخر، فضا هم به O(1) می‌رسد.

</details>

<details><summary>**مسئله‌ی Coin Change را چطور با DP حل می‌کنی؟**</summary>

`dp[m]` = کم‌ترین تعدادِ سکه برای مبلغِ m. `dp[0]=0`، بقیه بی‌نهایت. برای هر مبلغ m و هر سکه‌ی c≤m: `dp[m] = min(dp[m], dp[m-c] + 1)`. جواب `dp[amount]` (یا ناممکن اگر بی‌نهایت ماند). O(amount × len(coins)) زمان، O(amount) فضا.

</details>

### پیچیدگی

<details><summary>**فرقِ time و space complexity؟ و time-space trade-off؟**</summary>

Time = تعدادِ عملیات بر حسبِ n؛ Space = حافظه‌ی **اضافه‌ی** مصرفی (auxiliary، جدا از ورودی، شاملِ عمقِ recursion). اغلب می‌توان زمان را با خرجِ حافظه کم کرد (memoization، hash برای lookup سریع) یا برعکس (in-place با حافظه‌ی کم ولی کندتر) — این همان trade-off است.

</details>

<details><summary>**فرقِ Big O، Ω و Θ؟**</summary>

O کرانِ **بالا** (بدترین حالت)، Ω کرانِ **پایین** (بهترین حالت)، Θ وقتی بالا و پایین از یک مرتبه‌اند (کرانِ دقیق). در محاوره «complexity» معمولاً worst-case O است. مثلاً quick sort: O(n²) بدترین، Ω(n log n) بهترین، Θ ندارد چون بالا و پایین فرق دارند.

</details>

<details><summary>**فرقِ amortized و average-case؟**</summary>

Amortized: میانگینِ هزینه روی یک **دنباله‌ی تضمین‌شده** از عملیات، حتی در بدترین ورودی (مثلِ append در dynamic array). Average: میانگینِ **آماری** روی توزیعِ ورودی‌های تصادفی (مثلِ O(n log n) میانگینِ quick sort). amortized تضمینِ قطعی می‌دهد، average فرضِ توزیع.

</details>

<details><summary>**در عمل ضریب‌های ثابت چقدر مهم‌اند؟ آیا همیشه O کوچک‌تر بهتر است؟**</summary>

برای nِ بزرگ مرتبه برنده است، اما برای nِ کوچک ثابت‌ها تعیین‌کننده‌اند: insertion sort (O(n²)) روی آرایه‌های کوچک از merge sort (O(n log n)) سریع‌تر است — برای همین Timsort روی run‌های کوچک از insertion استفاده می‌کند. cache locality و ثابت‌ها در دنیای واقعی واقعاً مهم‌اند.

</details>

<details><summary>**یک تابع با دو حلقه‌ی متوالی (نه تو در تو) چه پیچیدگی‌ای دارد؟**</summary>

`O(n) + O(n) = O(2n) = O(n)` — جمع می‌شوند و ضریب حذف می‌شود. فقط حلقه‌های **تو در تو** ضرب می‌شوند (O(n²)). و همیشه فقط جمله‌ی غالب می‌ماند: `O(n² + n) = O(n²)`.

</details>
