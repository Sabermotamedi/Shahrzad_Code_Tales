# سطح ۳ — توضیح فنی: Cloud & Infrastructure ⚙️

> دیگر قصه‌ای در کار نیست. این سند مرجع فنی است — همان چیزهایی که در مصاحبه و کار واقعی لازم داری. هر بخش با معادل قصه‌ای‌اش شروع می‌شود تا ذهنت وصل بماند.

---

## ۱. Containers — Docker و Containerization

> در قصه: «جعبه‌ی استانداردِ غذا که همه‌جا یک‌جور کار می‌کند.»

### Image در برابر Container

| | Image | Container |
|---|---|---|
| ماهیت | قالبِ **ثابت** و فقط‌خواندنی (read-only) | نمونه‌ی **در حالِ اجرا**ی یک image |
| قیاس | کلاس / فایل اجرایی | آبجکت / پروسه |
| تعداد | یک image | از هر image **چند container** می‌توان ساخت |
| لایه‌ها | پشته‌ای از لایه‌های read-only | + یک لایه‌ی **writable** بالای همه |

`docker run image` یعنی: یک لایه‌ی نوشتنی روی image بگذار، namespace و cgroup بساز، و پروسه را اجرا کن.

### Dockerfile و لایه‌ها (Layers)

هر دستورِ `RUN`/`COPY`/`ADD` یک **layer** جدید می‌سازد؛ `FROM`/`ENV`/`CMD`/`WORKDIR` متادیتا هستند. لایه‌ها **کش** می‌شوند و در ساخت‌های بعدی و بین imageها **به اشتراک** گذاشته می‌شوند (content-addressable با hash).

```dockerfile
# مثال بهینه: ترتیب از کم‌تغییر به پرتغییر
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .                 # لایه‌ای که کم عوض می‌شود
RUN pip install --no-cache-dir -r requirements.txt
COPY . .                                # کد که زیاد عوض می‌شود ← آخر
EXPOSE 8080
CMD ["python", "app.py"]
```

نکات بهینه‌سازی:
- **ترتیب اهمیت دارد**: چیزهای کم‌تغییر را اول COPY کن تا cache بیشتر hit شود. اگر `COPY . .` را اول بگذاری، با هر تغییرِ کوچکِ کد، نصبِ dependencyها دوباره اجرا می‌شود.
- **Multi-stage build** برای کوچک کردن image نهایی (build در یک stage، فقط خروجی به stage آخر):

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN go build -o /app

FROM gcr.io/distroless/base       # image نهاییِ کوچک و بدون ابزار اضافه
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```

- **`.dockerignore`** تا `node_modules`/`.git` وارد context نشوند.
- **`CMD` در برابر `ENTRYPOINT`**: `ENTRYPOINT` فرمانِ ثابت، `CMD` آرگومان‌های پیش‌فرضِ قابل‌override. ترکیبشان رایج است.

> نکته‌ی مصاحبه: چرا `COPY requirements.txt` قبل از `COPY . .`؟ → برای حداکثرِ layer cache؛ تا وقتی dependencyها عوض نشده، لایه‌ی نصب از کش می‌آید و build سریع می‌ماند.

### Registry

انبارِ imageها. عمومی (Docker Hub) یا خصوصی (ECR، GCR، GitHub Container Registry، Harbor).

```bash
docker build -t myregistry.io/cat-api:1.4 .
docker push  myregistry.io/cat-api:1.4
docker pull  myregistry.io/cat-api:1.4
```

> نکته‌ی مصاحبه: از تگ `latest` در production استفاده نکن — قابل بازتولید نیست (mutable). تگِ نسخه‌دار یا **digest** (`@sha256:...`) که immutable است را استفاده کن.

### Containerization در برابر VM

| | Container | Virtual Machine |
|---|---|---|
| جداسازی | سطحِ پروسه (kernel مشترک) | سطحِ سخت‌افزار (kernel جدا) |
| اندازه | مگابایت | گیگابایت |
| زمان روشن شدن | میلی‌ثانیه | ثانیه تا دقیقه |
| سربار | کم | زیاد (hypervisor + OS کامل) |
| ایزولاسیون امنیتی | ضعیف‌تر (kernel مشترک) | قوی‌تر |

مکانیزم‌های لینوکس که container را می‌سازند:
- **Namespaces** — جداسازیِ دید: `pid` (لیست پروسه‌ها)، `net` (شبکه)، `mnt` (فایل‌سیستم)، `uts` (hostname)، `user` (UID/GID)، `ipc`. هر container فکر می‌کند تنهاست.
- **cgroups (control groups)** — محدودسازیِ منابع: سهمیه‌ی CPU، حافظه، I/O. اگر container از سقفِ memory بگذرد، **OOM-killed** می‌شود.
- **Union filesystem** (overlayfs) — لایه‌های read-only + لایه‌ی writable روی هم.

> نکته‌ی مصاحبه: container سیستم‌عامل جدا **ندارد**؛ kernelِ host را share می‌کند. به همین خاطر container لینوکسی مستقیم روی kernel ویندوز اجرا نمی‌شود (روی Docker Desktop یک VM سبکِ لینوکس زیرش هست).

---

## ۲. Orchestration — Kubernetes

> در قصه: «آقای بندر که هزاران جعبه را می‌چیند، خراب‌ها را عوض می‌کند، تعداد را کم و زیاد می‌کند.»

### آبجکت‌های اصلی

| آبجکت | نقش |
|---|---|
| **Pod** | کوچک‌ترین واحدِ قابلِ زمان‌بندی؛ یک یا چند container که شبکه و storage مشترک دارند. **فانی** است. |
| **ReplicaSet** | تضمین می‌کند تعداد مشخصی از یک Pod زنده باشد (معمولاً مستقیم با آن کار نمی‌کنی). |
| **Deployment** | ReplicaSet را مدیریت می‌کند + **rolling update** و **rollback**. واحدِ اصلیِ کارِ تو. |
| **Service** | IP/DNSِ **ثابت** برای دسترسی به مجموعه‌ای از Podها (که IPشان مدام عوض می‌شود) + load balancing داخلی. |
| **Node** | یک ماشینِ worker (VM یا فیزیکی) که Podها رویش اجرا می‌شوند. |
| **Namespace** | جداسازیِ منطقیِ منابع داخل یک cluster (مثلاً `dev` / `prod`). |
| **ConfigMap / Secret** | تنظیمات و رازها، جدا از image. |
| **Ingress** | روتینگِ L7 از بیرون به Serviceهای داخل (host/path-based). |

انواع **Service**:
- **ClusterIP** (پیش‌فرض) — فقط داخل cluster قابل دسترسی.
- **NodePort** — روی پورتی ثابت روی همه‌ی Nodeها باز می‌شود.
- **LoadBalancer** — یک load balancer ابری بیرونی می‌سازد.

### معماری (سطح بالا)

```
Control Plane (مغز):                  Worker Node (بازو):
┌──────────────────────┐             ┌──────────────────────┐
│ api-server  (دروازه) │             │ kubelet  (مأمور Node) │
│ etcd        (حافظه)  │             │ kube-proxy (شبکه)     │
│ scheduler   (چیننده) │   ◄──────►  │ container runtime     │
│ controller-manager   │             │   (containerd)        │
│   (حلقه‌ی reconcile)  │             │   └─ Pods             │
└──────────────────────┘             └──────────────────────┘
```

- **etcd** — پایگاه key-value که **کلِ state** (desired و فعلی) آن‌جاست.
- **scheduler** — تصمیم می‌گیرد هر Pod روی کدام Node برود.
- **controller-manager** — حلقه‌های **reconciliation**: مدام واقعیت را با desired state مقایسه و اصلاح می‌کند.
- **kubelet** — روی هر Node، Podها را زنده نگه می‌دارد و به api-server گزارش می‌دهد.

### Self-healing و Scaling

- **Self-healing**: مدلِ k8s **declarative** است؛ تو desired state را اعلام می‌کنی (`replicas: 3`). اگر Podی بمیرد یا Nodeای بیفتد، controllerها اختلافِ واقعیت و خواسته را می‌بینند و جبران می‌کنند. **liveness probe** (مرده است؟ restart کن) و **readiness probe** (آماده‌ی ترافیک است؟ وگرنه از Service کنار بگذار).
- **Scaling**:
  - دستی: `kubectl scale ... --replicas=N`
  - خودکارِ افقی: **HPA (HorizontalPodAutoscaler)** بر اساس CPU/memory/متریک سفارشی تعداد Pod را کم/زیاد می‌کند.
  - **Cluster Autoscaler** تعداد **Node**ها را تنظیم می‌کند.

### Deployment + Service واقعی

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cat-api
spec:
  replicas: 3
  selector:
    matchLabels: { app: cat-api }
  strategy:
    type: RollingUpdate            # بدون downtime نسخه‌ی جدید را تدریجی جایگزین کن
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  template:
    metadata:
      labels: { app: cat-api }
    spec:
      containers:
        - name: cat-api
          image: myregistry.io/cat-api:1.4
          ports:
            - containerPort: 8080
          resources:               # cgroup limits
            requests: { cpu: "100m", memory: "128Mi" }
            limits:   { cpu: "500m", memory: "256Mi" }
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 5
          livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: cat-api
spec:
  selector:
    app: cat-api                   # هر Podی با این label پشتِ این Service است
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### kubectl پایه

```bash
kubectl apply -f deploy.yaml          # اعمالِ declarative (idempotent)
kubectl get pods -o wide              # وضعیت + Node + IP
kubectl describe pod <pod>            # رویدادها (Events) — برای دیباگ طلاست
kubectl logs <pod> [-c container] [-f] [--previous]   # لاگ‌ها؛ previous = کانتینرِ قبلِ crash
kubectl exec -it <pod> -- sh          # شل داخل Pod
kubectl rollout status deploy/cat-api
kubectl rollout undo deploy/cat-api   # بازگشت به نسخه‌ی قبل
kubectl scale deploy/cat-api --replicas=10
kubectl get events --sort-by=.lastTimestamp
```

> نکته‌ی مصاحبه: فرق `requests` و `limits` — `requests` چیزی است که scheduler برای جا دادن Pod **تضمین** می‌کند؛ `limits` سقفی است که عبور از آن باعث throttle (CPU) یا OOM-kill (memory) می‌شود.

---

## ۳. DevOps — CI/CD و Infrastructure as Code

### CI/CD

> در قصه: «نوار نقاله‌ای که هر تغییر را خودکار می‌سازد، می‌چشد و می‌فرستد.»

- **CI (Continuous Integration)**: هر commit → build + test خودکار. هدف: کشفِ زودِ شکستن، ادغامِ مکررِ کوچک به جای merge‌های بزرگ و دردناک.
- **Continuous Delivery**: خروجیِ آماده‌ی deploy است، اما کلیدِ آخر **دستی** زده می‌شود.
- **Continuous Deployment**: حتی کلیدِ آخر هم خودکار است — هر تغییرِ سبز مستقیم به production می‌رود.

مراحلِ رایجِ pipeline:

```
Lint → Build → Unit Test → Build Image → Push → Integration Test → Deploy (staging → prod)
```

استراتژی‌های deploy (نکته‌ی مصاحبه):
- **Rolling** — تدریجی Podهای قدیم با جدید عوض می‌شوند (پیش‌فرضِ k8s).
- **Blue-Green** — دو محیطِ کامل؛ ترافیک یک‌باره از آبی به سبز سوییچ می‌شود، rollback فوری.
- **Canary** — درصد کمی از ترافیک به نسخه‌ی جدید، در صورت سالم بودن تدریجی بیشتر.

### GitHub Actions workflow کامل

```yaml
# .github/workflows/deploy.yml
name: build-test-deploy
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r requirements.txt
      - run: pytest -q                       # مرحله‌ی Test

  build-and-push:
    needs: test                              # فقط اگر test سبز شد
    runs-on: ubuntu-latest
    permissions: { contents: read, packages: write }
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6     # مرحله‌ی Build + Push
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - run: |                                # مرحله‌ی Deploy
          kubectl set image deployment/cat-api \
            cat-api=ghcr.io/${{ github.repository }}:${{ github.sha }}
```

> نکته‌ی مصاحبه: تگ image را با `${{ github.sha }}` بزن نه `latest` — تا هر deploy دقیقاً قابلِ ردیابی و rollback باشد. رازها هرگز در فایلِ workflow نباشند؛ از `secrets` بیایند.

### docker-compose (محیطِ توسعه‌ی چندسرویسی)

```yaml
# docker-compose.yml — برای dev/test محلی، نه مدیریتِ production در مقیاس
services:
  api:
    build: .
    ports: ["8080:8080"]
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/app
    depends_on: [db]
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```

```bash
docker compose up -d         # همه‌ی سرویس‌ها را بالا بیاور
docker compose logs -f api
docker compose down -v       # خاموش کن + volumeها را هم پاک کن
```

> نکته‌ی مصاحبه: `docker-compose` در برابر Kubernetes — compose برای **یک ماشین** و توسعه/تست است؛ k8s برای **چند ماشین**، production، self-healing و scale. compose orchestratorِ توزیع‌شده نیست.

### Infrastructure as Code (Terraform)

> در قصه: «نقشه‌ی کلِ بندر روی کاغذ، تا هر جای دیگری دوباره ساخته شود.»

- زیرساخت (سرور، شبکه، DB، DNS) به‌شکلِ **کدِ نسخه‌دار در git** تعریف می‌شود → قابلِ بازبینی، بازتولید و audit.
- **declarative در برابر imperative**:

| | Imperative | Declarative |
|---|---|---|
| تمرکز | **چطور** (گام‌به‌گام) | **چه چیزی** (حالتِ نهایی) |
| مثال | اسکریپت bash که دستور اجرا می‌کند | Terraform / k8s manifest |
| تکرار | اجرای دوباره ممکن است خراب کند | **idempotent** — اجرای دوباره امن است |

- Terraform یک **state file** نگه می‌دارد (چه چیزی واقعاً ساخته) و با اجرای `plan`، **diff** بین کدِ تو و واقعیت را نشان می‌دهد، بعد `apply` آن diff را اعمال می‌کند.

```hcl
terraform {
  required_providers { aws = { source = "hashicorp/aws" } }
}
provider "aws" { region = "eu-central-1" }

resource "aws_instance" "web" {
  ami           = "ami-0abc123"
  instance_type = "t3.micro"
  tags = { Name = "cat-api" }
}
```

```bash
terraform init      # دانلودِ providerها
terraform plan      # diff: چه چیزی ساخته/تغییر/حذف می‌شود؟
terraform apply     # اعمال
terraform destroy   # نابودیِ همه‌چیز
```

> نکته‌ی مصاحبه: ابزارهای دیگر — **Ansible** بیشتر برای configuration management (و نیمه‌imperative)، **Pulumi** همان IaC ولی با زبان‌های برنامه‌نویسی واقعی، **CloudFormation** نسخه‌ی بومیِ AWS.

---

## ۴. Operations — Monitoring، Logging، Tracing، Observability

> در قصه: «دوربین‌ها (عدد)، دفترها (ثبت رویداد)، حسگرها (ردگیری مسیر).»

### سه ستونِ Observability

| ستون | داده | جواب می‌دهد به | ابزار |
|---|---|---|---|
| **Metrics** | اعداد در طول زمان (time-series) | «چقدر؟ چند تا؟ چه روندی؟» | Prometheus، Grafana |
| **Logs** | رویدادهای گسسته‌ی متنی | «دقیقاً چه اتفاقی و کِی افتاد؟» | ELK/Loki، Fluentd |
| **Traces** | مسیرِ یک درخواست بین سرویس‌ها | «وقت کجا تلف شد؟ کدام سرویس مقصر است؟» | Jaeger، Tempo، OpenTelemetry |

### Monitoring (Prometheus / Grafana)

- **Prometheus** متریک‌ها را با مدلِ **pull** از endpointِ `/metrics` هر سرویس **scrape** می‌کند و در یک TSDB ذخیره می‌کند. کوئری با زبانِ **PromQL**.
- چهار نوع متریک: **Counter** (فقط بالا می‌رود؛ تعداد درخواست)، **Gauge** (بالا/پایین؛ مصرف حافظه)، **Histogram** و **Summary** (توزیع، مثل latency percentileها).
- **Grafana** داشبورد و نمودار می‌سازد؛ **Alertmanager** هشدارها را مدیریت می‌کند (دسته‌بندی، silence، route به Slack/PagerDuty).
- **چهار سیگنال طلایی (Golden Signals)**: Latency، Traffic، Errors، Saturation.

```
# نمونه‌ی خروجیِ /metrics
http_requests_total{method="GET",status="200"} 10234
http_request_duration_seconds_bucket{le="0.1"} 9000
```

### Logging

- **Structured logging**: به جای متنِ آزاد، **JSON** بنویس تا قابلِ پارس، فیلتر و جستجو باشد:

```json
{"ts":"2026-06-05T12:00:00Z","level":"error","service":"cat-api",
 "trace_id":"abc123","msg":"db timeout","duration_ms":5021}
```

- **Aggregation**: در محیطِ توزیع‌شده، لاگ‌ها از همه‌ی containerها به یک سیستمِ متمرکز می‌روند (stack مثل **ELK** = Elasticsearch + Logstash + Kibana، یا **Loki**). در k8s، container باید به **stdout/stderr** بنویسد و یک agent (Fluentd/Fluent Bit) آن‌ها را جمع می‌کند — نه نوشتن در فایلِ داخل container.
- **سطوح (levels)**: DEBUG < INFO < WARN < ERROR. در production معمولاً INFO به بالا.

### Tracing (Distributed Tracing)

- یک **trace** نمایشِ کلِ سفرِ یک درخواست در سیستمِ توزیع‌شده است؛ از چند **span** ساخته شده.
- هر **span** = یک واحدِ کار (مثلاً «کوئریِ DB» یا «فراخوانیِ سرویسِ پرداخت») با زمانِ شروع/پایان، نام، و رابطه‌ی parent/child.
- **trace_id** در طولِ کلِ مسیر propagate می‌شود (در headerها، مثل W3C `traceparent`)؛ به همین خاطر می‌توان همه‌ی spanها را به هم چسباند و دید **کدام قدم کُند بود**.
- **OpenTelemetry (OTel)** استانداردِ باز برای تولید و انتقالِ هر سه ستون است.

```
Trace: GET /checkout  (trace_id=abc123)  ── total 520ms
 ├─ span: api-gateway           20ms
 ├─ span: cart-service          50ms
 ├─ span: payment-service      400ms  ◄── گلوگاه اینجاست
 └─ span: db.query             400ms     (فرزندِ payment)
```

### Monitoring در برابر Observability

| Monitoring | Observability |
|---|---|
| پاسخ به سؤال‌های **از پیش تعریف‌شده** | پاسخ به سؤال‌هایی که **از قبل پیش‌بینی نکرده‌ای** |
| **known unknowns** («آیا CPU بالاست؟») | **unknown unknowns** («چرا فقط کاربرانِ اندرویدیِ ساعت ۳ صبح کند هستند؟») |
| داشبورد و هشدارِ ثابت | کاوشِ آزاد در داده‌ی غنی و high-cardinality |
| می‌گوید **چیزی خراب است** | کمک می‌کند بفهمی **چرا** |

> نکته‌ی مصاحبه: monitoring زیرمجموعه‌ای از observability است نه مترادفش. سیستمِ observable آن‌قدر داده‌ی غنی (با ابعادِ زیاد) تولید می‌کند که بتوانی سؤالِ جدید بپرسی **بدون deploy کردنِ کدِ تازه**. سه ستون (metrics/logs/traces) ابزارهای رسیدن به observability‌اند.

---

## 🔬 آزمایشگاه

```bash
# ۱) اولین container: تستِ نصبِ Docker
docker run hello-world            # image را pull و یک container اجرا می‌کند و خارج می‌شود

# ۲) یک Dockerfile کوچک بساز و build کن
mkdir lab && cd lab
cat > Dockerfile <<'EOF'
FROM alpine:3.20
CMD ["sh", "-c", "echo سلام از داخل container && sleep 3600"]
EOF
docker build -t mylab:1 .         # از Dockerfile یک image بساز
docker images | grep mylab        # image ساخته‌شده را ببین

# ۳) اجرا در پس‌زمینه و مشاهده‌ی container زنده
docker run -d --name lab1 mylab:1 # -d = detached (پس‌زمینه)
docker ps                         # container‌های در حالِ اجرا
docker logs lab1                  # خروجیِ stdout/stderr را ببین
docker exec -it lab1 sh           # شل داخل container (بعد: exit)

# ۴) لایه‌های یک image را ببین
docker history mylab:1            # هر لایه + اندازه + دستوری که ساختش
docker inspect mylab:1            # متادیتای کامل (JSON)

# ۵) تمیزکاری
docker stop lab1 && docker rm lab1
docker rmi mylab:1

# ۶) (اختیاری) خوشه‌ی k8s محلی برای تمرین
#    minikube start            # یک cluster تک-نودی روی VM/داکر
#    kind create cluster       # k8s داخلِ Docker (سبک‌تر، برای CI عالی)
#    سپس: kubectl apply -f deploy.yaml ; kubectl get pods
```

---

## ✅ چک‌لیست تسلط

اگر بتوانی به این‌ها جواب بدهی، این فصل را بلدی:

- [ ] فرقِ image و container را با قیاسِ کلاس/آبجکت توضیح بده.
- [ ] چرا ترتیبِ دستورهای Dockerfile روی سرعتِ build اثر می‌گذارد؟ (layer cache)
- [ ] فرقِ container و VM در سطحِ namespace/cgroup و kernel چیست؟
- [ ] فرقِ `CMD` و `ENTRYPOINT`؟ چرا `latest` در production بد است؟
- [ ] چرا Pod فانی است و Service چه مشکلی را حل می‌کند؟
- [ ] self-healing در k8s دقیقاً چطور کار می‌کند؟ (desired state + reconciliation loop)
- [ ] فرقِ `requests` و `limits`؟ فرقِ liveness و readiness probe؟
- [ ] سه مرحله‌ی اصلیِ یک pipeline و فرقِ Continuous Delivery و Deployment؟
- [ ] فرقِ declarative و imperative؛ چرا Terraform و k8s declarative‌اند؟
- [ ] سه ستونِ observability را نام ببر و بگو هر کدام به چه سؤالی جواب می‌دهد.
- [ ] فرقِ trace و span؟ trace_id چطور propagate می‌شود؟
- [ ] چرا monitoring مترادفِ observability نیست؟ (known vs unknown unknowns)

---

## 🎯 سوالات مصاحبه

### 📦 Containers / Docker

<details><summary>**فرقِ image و container چیست؟**</summary>

image یک قالبِ **ثابت و read-only** است (مثل کلاس یا فایلِ اجرایی). container یک نمونه‌ی **در حالِ اجرا**ی آن image است (مثل آبجکت یا پروسه) که یک لایه‌ی **writable** بالای لایه‌های image دارد. از یک image می‌توان چند container ساخت.

</details>

<details><summary>**فرقِ container و VM چیست؟**</summary>

VM کلِ یک سیستم‌عامل را روی hypervisor شبیه‌سازی می‌کند (kernel جدا، گیگابایت، روشن‌شدن کند، ایزولاسیونِ قوی). container فقط پروسه را با **namespaces** و **cgroups** جدا می‌کند و kernelِ host را **share** می‌کند (مگابایت، روشن‌شدن میلی‌ثانیه‌ای، ایزولاسیونِ ضعیف‌تر).

</details>

<details><summary>**namespace و cgroup هر کدام چه می‌کنند؟**</summary>

**namespace** = جداسازیِ **دید** (هر container لیستِ پروسه، شبکه، فایل‌سیستم و hostnameِ خودش را می‌بیند و فکر می‌کند تنهاست). **cgroup** = محدودسازیِ **منابع** (سهمیه‌ی CPU، memory، I/O). namespace می‌گوید «چه می‌بینی»، cgroup می‌گوید «چقدر می‌توانی مصرف کنی».

</details>

<details><summary>**Docker layer چیست و چرا ترتیبِ Dockerfile مهم است؟**</summary>

هر `RUN`/`COPY`/`ADD` یک layer می‌سازد که کش می‌شود. اگر یک layer عوض شود، همه‌ی layerهای بعد از آن باید دوباره ساخته شوند. پس چیزهای کم‌تغییر (مثل نصب dependencyها) را **قبل** از کدِ پرتغییر بگذار تا cache بیشتر hit شود و build سریع بماند.

</details>

<details><summary>**`CMD` در برابر `ENTRYPOINT`؟**</summary>

`ENTRYPOINT` فرمانِ اصلی و ثابتِ container است؛ `CMD` آرگومان‌های پیش‌فرض که هنگامِ `docker run` قابلِ override هستند. اگر هر دو باشند، `CMD` به‌عنوانِ آرگومانِ `ENTRYPOINT` پاس می‌شود.

</details>

<details><summary>**چرا تگِ `latest` در production بد است؟**</summary>

`latest` mutable است؛ دو نفر در دو زمان ممکن است image متفاوتی بگیرند و deploy قابلِ بازتولید/rollback نیست. از تگِ نسخه‌دار یا **digest** (`@sha256:...`) که immutable است استفاده کن.

</details>

<details><summary>**Registry چیست و multi-stage build چه فایده‌ای دارد؟**</summary>

Registry انبارِ imageها است (Docker Hub، ECR، GHCR). **multi-stage build** اجازه می‌دهد در یک stage build/compile کنی و فقط خروجیِ نهایی را به یک image پایه‌ی کوچک (مثل distroless) کپی کنی → image نهاییِ کوچک‌تر، امن‌تر و بدونِ ابزارهای build.

</details>

### 🚢 Kubernetes

<details><summary>**فرقِ Pod، Deployment و Service؟**</summary>

**Pod** کوچک‌ترین واحدِ اجرا (یک/چند container) و فانی است. **Deployment** تعداد و نسخه‌ی Podها را مدیریت می‌کند و rolling update/rollback می‌دهد. **Service** یک IP/DNSِ ثابت برای دسترسی به مجموعه‌ای از Podها (که IPشان عوض می‌شود) می‌سازد و ترافیک را بینشان پخش می‌کند.

</details>

<details><summary>**چرا Service لازم است وقتی Pod خودش IP دارد؟**</summary>

چون Podها فانی‌اند: می‌میرند و با IP جدید دوباره ساخته می‌شوند. اتکا به IPِ مستقیمِ Pod شکننده است. Service یک نقطه‌ی ورودیِ پایدار (با selector روی labelها) می‌دهد که همیشه به Podهای زنده اشاره می‌کند.

</details>

<details><summary>**self-healing در k8s چطور کار می‌کند؟**</summary>

مدل **declarative** است: تو desired state را اعلام می‌کنی (`replicas: 3`). controllerها در یک **reconciliation loop** مدام واقعیت را با desired مقایسه می‌کنند؛ اگر Podی بمیرد، فوراً یکی نو می‌سازند تا دوباره به حالتِ مطلوب برسد. **liveness probe** restart می‌کند، **readiness probe** Podِ ناآماده را از Service کنار می‌گذارد.

</details>

<details><summary>**فرقِ `requests` و `limits`؟**</summary>

`requests` منابعی است که scheduler هنگامِ جا دادنِ Pod روی Node **تضمین** می‌کند (مبنای زمان‌بندی). `limits` سقفِ مصرف است: عبور از limitِ CPU باعث **throttle** و عبور از limitِ memory باعث **OOM-kill** می‌شود.

</details>

<details><summary>**فرقِ liveness و readiness probe؟**</summary>

**liveness**: «آیا Pod زنده است؟» اگر fail شود، k8s container را **restart** می‌کند. **readiness**: «آیا آماده‌ی دریافتِ ترافیک است؟» اگر fail شود، Pod از endpointهای Service **حذف** می‌شود (restart نمی‌شود). startup طولانی → readiness؛ deadlock → liveness.

</details>

<details><summary>**انواعِ Service و کاربردشان؟**</summary>

**ClusterIP** (داخلی، پیش‌فرض)، **NodePort** (پورتِ ثابت روی هر Node)، **LoadBalancer** (LB ابریِ بیرونی). برای روتینگِ L7ِ HTTP از بیرون معمولاً **Ingress** روی ClusterIPها استفاده می‌شود.

</details>

### 🎢 CI/CD

<details><summary>**CI و CD چه فرقی دارند؟**</summary>

**CI (Continuous Integration)**: هر commit خودکار build و test می‌شود تا شکستن زود کشف شود. **Continuous Delivery**: خروجی همیشه آماده‌ی deploy است ولی کلیدِ آخر دستی است. **Continuous Deployment**: حتی کلیدِ آخر هم خودکار است و هر تغییرِ سبز مستقیم به production می‌رود.

</details>

<details><summary>**مراحلِ یک pipeline معمولی؟**</summary>

Lint → Build → Unit Test → Build Image → Push to Registry → Integration Test → Deploy (staging سپس prod). اگر هر مرحله fail شود، pipeline می‌ایستد و کدِ خراب جلوتر نمی‌رود.

</details>

<details><summary>**فرقِ Rolling، Blue-Green و Canary deployment؟**</summary>

**Rolling**: تدریجی Podهای قدیم با جدید عوض می‌شوند (پیش‌فرضِ k8s). **Blue-Green**: دو محیطِ کاملِ موازی؛ سوییچِ یک‌باره‌ی ترافیک و rollbackِ فوری. **Canary**: درصدِ کمی از ترافیک به نسخه‌ی جدید می‌رود و در صورتِ سالم بودن تدریجی بیشتر می‌شود.

</details>

<details><summary>**چرا image را با commit SHA تگ می‌کنیم نه `latest`؟**</summary>

تا هر deploy دقیقاً به یک نسخه‌ی مشخصِ کد گره بخورد؛ ردیابی، بازتولید و rollback ممکن شود. `latest` این تضمین را از بین می‌برد.

</details>

### 📐 Infrastructure as Code

<details><summary>**Infrastructure as Code یعنی چه و چه فایده‌ای دارد؟**</summary>

تعریفِ زیرساخت (سرور، شبکه، DB، DNS) به‌شکلِ کدِ نسخه‌دار در git به‌جای کلیکِ دستی. فایده: قابلِ بازبینی (code review)، بازتولیدِ مو-به-مو، audit، و رفعِ drift و دانشِ شفاهی.

</details>

<details><summary>**فرقِ declarative و imperative؟**</summary>

**Imperative**: گام‌به‌گام می‌گویی **چطور** (اسکریپتِ bash). **Declarative**: فقط **حالتِ نهایی** را اعلام می‌کنی و ابزار راهِ رسیدن را پیدا می‌کند (Terraform، k8s). declarative معمولاً **idempotent** است؛ اجرای دوباره امن است.

</details>

<details><summary>**Terraform state چیست و چرا مهم است؟**</summary>

فایلی که Terraform در آن نگه می‌دارد **چه منابعی را واقعاً ساخته** و mapping بین کد و دنیای واقعی. با `plan`، diff بینِ کد و state را نشان می‌دهد. state معمولاً remote (S3 + lock) نگه‌داری می‌شود تا تیم همزمان خرابش نکند. گم شدنِ state یعنی Terraform دیگر نمی‌داند چه چیزی مالِ اوست.

</details>

### 📷 Observability

<details><summary>**سه ستونِ observability چیست؟**</summary>

**Metrics** (اعدادِ time-series؛ «چقدر؟»)، **Logs** (رویدادهای متنیِ گسسته؛ «دقیقاً چه و کِی؟»)، **Traces** (مسیرِ یک درخواست بینِ سرویس‌ها؛ «وقت کجا تلف شد؟»).

</details>

<details><summary>**فرقِ monitoring و observability؟**</summary>

monitoring پاسخ به سؤال‌های **از پیش تعریف‌شده** (known unknowns) با داشبورد/هشدارِ ثابت است. observability توانایی پاسخ به سؤال‌های **پیش‌بینی‌نشده** (unknown unknowns) با کاوش در داده‌ی غنی است — بدونِ deploy کردنِ کدِ تازه. monitoring زیرمجموعه‌ی observability است، نه مترادفش.

</details>

<details><summary>**فرقِ trace و span؟**</summary>

**trace** کلِ سفرِ یک درخواست در سیستمِ توزیع‌شده است؛ از چند **span** ساخته شده. هر **span** یک واحدِ کار (مثلاً کوئریِ DB) با زمانِ شروع/پایان و رابطه‌ی parent/child است. یک **trace_id** مشترک در headerها propagate می‌شود تا spanها به هم بچسبند.

</details>

<details><summary>**انواعِ متریک در Prometheus؟**</summary>

**Counter** (فقط بالا می‌رود؛ تعدادِ درخواست)، **Gauge** (بالا/پایین؛ مصرفِ حافظه)، **Histogram** و **Summary** (توزیع، برای latency percentileها). Prometheus با مدلِ **pull** از `/metrics` جمع می‌کند، Grafana نمودار می‌کشد.

</details>

<details><summary>**چهار Golden Signal چیست؟**</summary>

**Latency** (تأخیر)، **Traffic** (حجمِ درخواست)، **Errors** (نرخِ خطا)، **Saturation** (اشباعِ منابع). اگر فقط چهار چیز را مانیتور کنی، همین‌ها.

</details>

<details><summary>**چرا structured logging و چرا stdout در k8s؟**</summary>

**structured (JSON)**: قابلِ پارس، فیلتر و جستجو و قابلِ گره خوردن به trace_id. **stdout/stderr در k8s**: container نباید در فایلِ داخلیِ خودش (که با مرگش پاک می‌شود) بنویسد؛ به stdout بنویسد تا agentِ جمع‌آوری (Fluentd/Fluent Bit) آن را به سیستمِ متمرکز (ELK/Loki) ببرد.

</details>

### 🛠️ سناریو / دیباگ

<details><summary>**برنامه روی لپ‌تاپ من کار می‌کند ولی در production کرش می‌کند — چطور دیباگ کنم؟**</summary>

اول مطمئن شو **همان image** اجرا می‌شود (تگِ دقیق/digest، نه `latest`). تفاوت‌های محیطی را چک کن: env varها و Secret/ConfigMapها، منابع (آیا OOM-killed شده؟ `kubectl describe pod` → Reason: OOMKilled)، دسترسیِ شبکه به DB/سرویس‌ها، فایل‌سیستمِ read-only یا مجوزها، معماریِ CPU (arm در برابر amd64). لاگ‌ها را با `kubectl logs --previous` ببین و با `docker run` همان image را با envهای production محلی اجرا کن.

</details>

<details><summary>**یک Pod در حالتِ `CrashLoopBackOff` است — چه چیزهایی را چک می‌کنی؟**</summary>

۱) `kubectl logs <pod> --previous` — خروجیِ کانتینرِ قبل از crash. ۲) `kubectl describe pod <pod>` — بخشِ Events و Reason (OOMKilled؟ خطای probe؟ image pull؟). ۳) آیا liveness probe خیلی سخت‌گیر است و قبل از آماده شدنِ app آن را می‌کشد؟ ۴) آیا CMD/ENTRYPOINT درست است و پروسه فوراً exit نمی‌کند؟ ۵) آیا env/Secret لازم تنظیم شده؟ ۶) `resources.limits` کافی است؟ CrashLoopBackOff یعنی k8s مدام restart می‌کند و با backoffِ نمایی صبر می‌کند.

</details>

<details><summary>**Pod در حالتِ `Pending` گیر کرده — چرا؟**</summary>

معمولاً scheduler نمی‌تواند Node مناسب پیدا کند: منابعِ کافی نیست (`requests` بزرگ‌تر از ظرفیتِ Nodeها)، taint/toleration یا nodeSelector/affinity جور نمی‌شود، یا PersistentVolume در دسترس نیست. `kubectl describe pod` در بخشِ Events دلیل را می‌گوید (مثلاً `0/3 nodes available: insufficient memory`).

</details>

<details><summary>**استقرارِ جدید را زدم و سایت پایین آمد — اولین کار؟**</summary>

`kubectl rollout undo deployment/<name>` برای بازگشتِ فوری به نسخه‌ی قبل (چون RollingUpdate نسخه‌ی قبلی را نگه می‌دارد). بعد با آرامش روی نسخه‌ی خراب در staging دیباگ کن. برای آینده: readiness probe درست + `maxUnavailable: 0` تا ترافیک به Podِ ناآماده نرود، و canary/blue-green برای کاهشِ شعاعِ انفجار.

</details>