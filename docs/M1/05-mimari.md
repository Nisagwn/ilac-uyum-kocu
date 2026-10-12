# Sistem Mimarisi

Bu belge M1 aşamasında **planlanan** mimariyi anlatır. Bileşen adları ve sınırlar M2–M5'te koda dönüşürken bu belge referans alınır; değişiklik olursa burada güncellenir.

Mimari üç soruya cevap verecek şekilde tasarlandı:

1. **Güvenli mi?** Koç tıbbi karar vermez, hasta verisi rol dışına çıkmaz (§9, [02](02-guardrail-politikasi.md), [03](03-veri-gizliligi.md)).
2. **Ölçeklenir mi?** API durumsuzdur ve yatayda çoğaltılır; önde reverse proxy + load balancer vardır (§2, §7).
3. **Dayanıklı mı?** Dış servis çökse, istek iki kez gelse, bir kopya ölse bile veri bozulmaz ve hasta acil yönlendirmeyi her koşulda alır (§4–§6).

## 1. Bağlam diyagramı

Sistemi kimler kullanır, hangi dış servislerle konuşur?

![mimari-baglam](diyagramlar/mimari-baglam.png)

<details><summary>Mermaid kaynağı</summary>

```mermaid
flowchart TB
    P(["👤 Hasta<br/>Tip 2 diyabet / hipertansiyon<br/>telefon, PWA"])
    C(["🩺 Hekim / diyabet hemşiresi<br/>masaüstü"])
    G(["👥 Bakım veren<br/>salt okunur"])

    SYS["<b>İlaç Uyumu ve Yaşam Tarzı Koçu</b><br/>ilaç planı · doz kaydı · ölçüm<br/>LLM koç · hekim dashboard'u"]

    LLM["LLM sağlayıcısı (M4'te seçilecek)<br/>tool use + structured output"]
    FDA["openFDA<br/>Drug Label API"]
    WP["Tarayıcı push servisi<br/>FCM · Mozilla · APNs"]
    E112["112 Acil<br/>(sistem dışı, hasta arar)"]

    P -- "doz işaretler, ölçüm girer,<br/>koçla yazışır" --> SYS
    C -- "hasta listesi, uyum skoru,<br/>risk bayrakları, haftalık özet" --> SYS
    G -- "bağlı hastayı izler" --> SYS
    SYS -- "takma kimlik + sayısal özet" --> LLM
    SYS -- "etken madde adı" --> FDA
    SYS -- "hatırlatma bildirimi<br/>(VAPID)" --> WP
    WP -.-> P
    SYS -. "acil belirtide yönlendirir" .-> E112
```

</details>

## 2. Bileşen diyagramı

![mimari-bilesen](diyagramlar/mimari-bilesen.png)

<details><summary>Mermaid kaynağı</summary>

```mermaid
flowchart TB
    subgraph Browser["Tarayıcı (PWA)"]
        UI_P["Hasta ekranları<br/>Bugün · Ölçüm · Koç · İlaçlar · Özet"]
        UI_D["Hekim dashboard'u<br/>Hasta listesi · Hasta detayı · Uyarılar"]
        UI_C["Bakım veren ekranları<br/>salt okunur"]
    end

    subgraph Edge["Kenar katmanı"]
        NGX["Nginx<br/>reverse proxy · TLS sonlandırma<br/>gzip · güvenlik başlıkları<br/>kaba IP rate limit"]
    end

    subgraph Next["Next.js sunucusu"]
        AUTHN["Giriş / çıkış<br/>JWT → httpOnly çerez"]
        PROXY["API proxy<br/>çerez → Authorization: Bearer"]
    end

    LB{{"Nginx upstream<br/>load balancer<br/>least_conn + pasif sağlık kontrolü"}}

    subgraph API["FastAPI × N kopya (durumsuz)"]
        MW["Middleware zinciri<br/>request ID · token bucket rate limit<br/>ETag · yapılandırılmış log · metrik"]
        SEC["Kimlik doğrulama + rol kontrolü<br/>yetkisiz → 404"]
        IDEM["Idempotency-Key<br/>doz · ölçüm · koç mesajı"]
        CRUD["/v1/patients · medications · doses<br/>measurements · alerts · summaries"]
        CHAT["/v1/coach/chat<br/>→ 202 + job_id"]
        DRUG["/v1/drugs/label"]
        HLT["/health/live · /health/ready"]
        TH["Eşik kontrolü<br/>eşik dışı ölçüm → alert"]
        ADH["Uyum skoru<br/>deterministik"]
        GR1["Guardrail Katman 1<br/>acil · doz tespiti"]
        OFS["openFDA servisi"]
        CB1["Circuit breaker<br/>openFDA"]
    end

    subgraph Worker["Worker × M kopya"]
        Q["Kuyruk tüketici<br/>3 deneme + backoff"]
        COACH["Koç orkestrasyonu<br/>tool-use döngüsü + Guardrail Katman 3"]
        CB2["Circuit breaker<br/>LLM sağlayıcısı<br/>açıkken → kural tabanlı koç"]
        TOOLS["Tool'lar + RAG"]
        SCH["APScheduler<br/>yalnızca lider kopya<br/>(PostgreSQL advisory lock)"]
        PUSH["pywebpush (VAPID)"]
    end

    DB[("PostgreSQL + pgvector")]
    RD[("Redis<br/>önbellek · rate limit kovaları<br/>iş kuyruğu · DLQ")]
    CL["LLM sağlayıcısı<br/>OpenRouter / Anthropic"]
    FDA["openFDA"]
    WPS["Push servisi<br/>→ tarayıcıdaki Service Worker"]

    subgraph Obs["Gözlemlenebilirlik"]
        PROM["Prometheus"] --> GRAF["Grafana"]
        JAE["Jaeger<br/>OpenTelemetry izleri"]
    end

    UI_P & UI_D & UI_C -- HTTPS --> NGX
    NGX --> AUTHN & PROXY
    AUTHN & PROXY --> LB
    LB --> MW --> SEC --> IDEM --> CRUD & CHAT & DRUG
    LB -. sağlık .-> HLT
    CRUD --> TH & ADH
    CRUD --> DB
    IDEM --> DB
    MW -. kova .-> RD
    CHAT --> GR1
    GR1 -- "temiz → iş" --> RD
    GR1 -- "acil / doz → sabit yanıt + alert" --> DB
    DRUG --> CB1 --> OFS --> FDA
    OFS -. önbellek .-> RD
    RD --> Q --> COACH --> CB2 --> CL
    CL -- tool_use --> TOOLS --> DB
    Q -- "3 kez başarısız" --> RD
    SCH --> DB
    SCH --> PUSH --> WPS
    API & Worker -. metrik / iz .-> Obs
```

</details>

| Bileşen | Sorumluluk |
|---|---|
| Nginx (reverse proxy) | Tek giriş noktası; TLS sonlandırma, gzip, güvenlik başlıkları (HSTS, CSP, X-Frame-Options), `X-Forwarded-For`'u istemci IP'siyle yeniden yazma, IP başına kaba istek sınırı (`limit_req`) |
| Nginx upstream (load balancer) | İstekleri API kopyalarına `least_conn` ile dağıtır; hata veren kopyayı geçici olarak dışarıda bırakır (`max_fails`, `fail_timeout`) |
| Next.js proxy | Token'ı tarayıcıdan saklar; tüm API çağrılarını aynı kökenden geçirir |
| Middleware zinciri | Her isteğe request ID, kullanıcı başına token bucket rate limit, ETag / `304`, JSON log ve metrik |
| Rol kontrolü | Her istekte JWT doğrulama ve hasta-erişim kuralı; yetkisizse 404 |
| Idempotency | Aynı `Idempotency-Key` ile tekrar gelen yazma isteğine ilk yanıt aynen döner, kayıt ikinci kez oluşmaz |
| Guardrail | Katman 1 API'de (LLM'den önce, eşzamanlı), Katman 3 worker'da (LLM çıktısından sonra) |
| Koç | Worker'da LLM ile tool-use döngüsü; structured output üretir. Model yapılandırmadan seçilir (bkz. [01 §5.1](01-proje-onerisi.md#51-llm-seçimi-deneyerek-karar)) |
| Circuit breaker | Dış servis art arda hata verirse bir süre çağırmaz, doğrudan yedek yola geçer |
| Worker | Koç işleri, hatırlatmalar; başarısız iş 3 denemeden sonra DLQ'ya |
| Lider kilidi | Birden çok worker kopyasından yalnızca biri zamanlayıcıyı çalıştırır |

## 3. Bir isteğin backend katmanlarından geçişi

Örnek: hasta "aldım" düğmesine basıyor.

![dizi-istek-yolu](diyagramlar/dizi-istek-yolu.png)

<details><summary>Mermaid kaynağı</summary>

```mermaid
sequenceDiagram
    autonumber
    participant H as Hasta (PWA)
    participant X as Nginx
    participant N as Next.js proxy
    participant L as Load balancer
    participant A as API kopyası 2
    participant R as Redis
    participant D as PostgreSQL

    H->>X: HTTPS POST /api/proxy/v1/doses (Idempotency-Key: k-123)
    X->>X: TLS sonlandır, IP limit_req, X-Forwarded-For yaz
    X->>N: iç ağ HTTP
    N->>L: Authorization: Bearer JWT
    L->>A: least_conn ile seçilen kopya
    A->>A: request ID üret, JWT doğrula
    A->>R: token bucket (kullanıcı başına) - Lua ile atomik
    alt kova boş
        R-->>A: reddet
        A-->>H: 429 + Retry-After + X-RateLimit-Remaining 0
    else izin
        A->>A: rol kontrolü - hasta kendi kaydı mı?
        A->>D: BEGIN - idempotency_keys (user, k-123) ekle
        alt anahtar zaten var
            D-->>A: çakışma - kayıtlı yanıtı oku
            A-->>H: ilk yanıtın aynısı (Idempotent-Replayed true)
        else yeni
            A->>D: dose_logs ekle + yanıtı anahtara yaz - COMMIT
            A-->>H: 201 Created
        end
    end
```

</details>

## 4. Koç isteğinin yaşam döngüsü (kuyruk + worker)

LLM çağrısı saniyeler sürebilir. API'yi bekletmemek için koç mesajı kuyruğa alınır; acil durum ise **kuyruğa hiç girmeden** API'de anında yanıtlanır.

![dizi-koc-istegi](diyagramlar/dizi-koc-istegi.png)

<details><summary>Mermaid kaynağı</summary>

```mermaid
sequenceDiagram
    autonumber
    participant H as Hasta (PWA)
    participant A as API
    participant G as Guardrail Katman 1
    participant Q as Redis kuyruğu
    participant W as Worker
    participant CB as Circuit breaker
    participant L as LLM sağlayıcısı
    participant D as PostgreSQL

    H->>A: POST /v1/coach/chat (Idempotency-Key)
    A->>G: girdi kontrolü (eşzamanlı)
    alt acil belirti / kendine zarar
        G-->>A: sabit 112 yanıtı
        A->>D: kritik alert
        A-->>H: 200 + acil kartı (LLM ve kuyruk yok)
    else doz / ilaç değişikliği talebi
        G-->>A: sabit "hekiminize danışın" yanıtı + bilgi alert'i
        A-->>H: 200
    else temiz
        A->>D: coach_jobs (queued) + bütçe kontrolü
        A->>Q: job_id
        A-->>H: 202 Accepted + job_id
        W->>Q: işi al
        W->>CB: LLM çağrılabilir mi?
        alt devre açık / bütçe dolu
            CB-->>W: kural tabanlı yedek koç
        else devre kapalı
            loop tool_use (en fazla 4 tur)
                W->>L: mesaj + tool sonuçları (ad/e-posta yok)
                L-->>W: tool_use isteği
                W->>D: get_adherence / get_measurements / search_education
            end
            L-->>W: {message, risk_level, escalate_to_clinician, sources}
            W->>W: kaynak doğrulama + Guardrail Katman 3
        end
        alt işlem hata verdi
            W->>Q: backoff ile yeniden dene (en fazla 3) - sonra DLQ
        end
        W->>D: coach_messages + llm_usage (+ alert) - job done
        H->>A: GET /v1/coach/jobs/{id} (ETag ile yoklama)
        A-->>H: yanıt + kaynaklar
    end
```

</details>

## 5. Circuit breaker durumları

LLM sağlayıcısı ve openFDA için ayrı birer devre kesici çalışır. Durum Redis'te tutulur, böylece tüm kopyalar aynı kararı paylaşır.

![durum-circuit-breaker](diyagramlar/durum-circuit-breaker.png)

<details><summary>Mermaid kaynağı</summary>

```mermaid
stateDiagram-v2
    [*] --> Kapali
    Kapali: Kapalı - istekler geçer
    Acik: Açık - dış servis çağrılmaz, yedek yol
    YariAcik: Yarı açık - tek deneme isteği

    Kapali --> Acik : 60 sn içinde 5 hata veya zaman aşımı
    Acik --> YariAcik : 30 sn bekleme doldu
    YariAcik --> Kapali : deneme başarılı
    YariAcik --> Acik : deneme başarısız
```

</details>

| Servis | Devre açıkken davranış |
|---|---|
| LLM sağlayıcısı | Kural tabanlı koç yanıt verir; guardrail'ler aynen geçerlidir. Haftalık özet şablon metinle üretilir. |
| openFDA | Önbellekteki son etiket döner; yoksa "ilaç bilgisi şu an alınamıyor" mesajı. |

## 6. Hatırlatma (web push) akışı

![dizi-web-push](diyagramlar/dizi-web-push.png)

<details><summary>Mermaid kaynağı</summary>

```mermaid
sequenceDiagram
    autonumber
    participant H as Hasta tarayıcısı
    participant SW as Service Worker
    participant A as API
    participant D as PostgreSQL
    participant W1 as Worker 1
    participant W2 as Worker 2
    participant P as Push servisi

    H->>SW: bildirim izni + PushManager.subscribe(VAPID public key)
    H->>A: POST /v1/push/subscribe {endpoint, keys}
    A->>D: push_subscriptions
    W1->>D: pg_try_advisory_lock(hatirlatma)
    D-->>W1: true - lider
    W2->>D: pg_try_advisory_lock(hatirlatma)
    D-->>W2: false - yalnızca koç işleri
    loop her dakika (yalnızca lider)
        W1->>D: zamanı gelmiş ve kaydı olmayan dozlar
        W1->>P: şifreli yük (VAPID imzalı) - sağlık verisi yok
        W1->>D: reminder_logs (schedule, gün) tekil - tekrar göndermez
    end
    P-->>SW: push olayı
    SW-->>H: "İlaç zamanı" bildirimi → Bugün ekranı
    Note over W1,W2: Lider ölürse bağlantısı kapanır, kilit serbest kalır, W2 lider olur
```

</details>

## 7. openFDA entegrasyonu

![dizi-openfda](diyagramlar/dizi-openfda.png)

<details><summary>Mermaid kaynağı</summary>

```mermaid
sequenceDiagram
    autonumber
    participant U as Hasta / hekim
    participant A as API /v1/drugs/label
    participant R as Redis
    participant CB as Circuit breaker
    participant F as openFDA
    participant D as PostgreSQL

    U->>A: ilaç adı (ör. "Glucophage")
    A->>A: Türk ticari adı → etken madde (eşleme tablosu)
    A->>R: önbellekte var mı?
    alt önbellekte yok
        A->>CB: openFDA çağrılabilir mi?
        alt devre kapalı
            A->>F: /drug/label.json?search=openfda.generic_name:"metformin"
            F-->>A: etiket (indications, warnings, drug_interactions)
            A->>R: 24 saat önbellek
            A->>D: özet → education_contents (kind = openfda)
        else devre açık
            CB-->>A: "şu an alınamıyor" yanıtı
        end
    end
    A-->>U: etken madde, uyarılar, etkileşimler
```

</details>

openFDA yalnızca **etken madde adıyla** sorgulanır; hastaya ait hiçbir veri dışarı gönderilmez.

## 8. Dağıtım (Docker Compose)

![dagitim-docker](diyagramlar/dagitim-docker.png)

<details><summary>Mermaid kaynağı</summary>

```mermaid
flowchart LR
    user(["Tarayıcı"]) -- "HTTPS :443" --> nginx

    subgraph host["Docker Compose"]
        nginx["nginx<br/>reverse proxy + LB"]
        web["web<br/>Next.js"]
        subgraph apis["api × 3"]
            api1["api-1"]
            api2["api-2"]
            api3["api-3"]
        end
        subgraph workers["worker × 2"]
            w1["worker-1<br/>lider"]
            w2["worker-2"]
        end
        migrate["migrate<br/>alembic upgrade head"]
        db[("db<br/>PostgreSQL 16 + pgvector")]
        redis[("redis")]
        prom["prometheus"]
        graf["grafana"]
        jae["jaeger"]
    end

    nginx --> web
    web --> nginx
    nginx --> api1 & api2 & api3
    apis --> db & redis
    workers --> db & redis
    migrate --> db
    prom -. scrape .-> apis & workers
    graf --> prom
    apis & workers -. OTLP .-> jae
    apis -. dış .-> ext["openFDA"]
    workers -. dış .-> ext2["LLM sağlayıcısı · Push servisi"]
```

</details>

| Servis | Görev | Dışarı açık mı? |
|---|---|---|
| `nginx` | Reverse proxy, TLS, load balancer, gzip, güvenlik başlıkları | Evet (80 → 443) |
| `web` | Next.js arayüz + proxy | Hayır (Nginx arkasında) |
| `api` × 3 | Durumsuz REST API; `docker compose up --scale api=3` | Hayır |
| `worker` × 2 | Koç kuyruğu; lider olan zamanlayıcıyı da çalıştırır | Hayır |
| `migrate` | Başlangıçta şema güncelleme; api/worker bunu bekler | Hayır |
| `db` | Tüm kalıcı veri | Hayır |
| `redis` | Önbellek, rate limit kovaları, kuyruk, DLQ, devre durumu | Hayır |
| `prometheus` / `grafana` | Metrik toplama ve panolar | Yalnızca yerel |
| `jaeger` | Dağıtık iz (trace) görüntüleme | Yalnızca yerel |

**Neden yatayda ölçeklenebilir?** API kopyaları bellekte durum tutmaz: oturum JWT'de, sayaçlar/kuyruk/devre durumu Redis'te, veri PostgreSQL'de. Bu yüzden load balancer isteği herhangi bir kopyaya gönderebilir (sticky session gerekmez).

**Sıfır kesintili güncelleme:** Kopya kapanırken `SIGTERM` alır, readiness `503` döner, Nginx yeni istek göndermez, süren istekler biter (graceful shutdown). Worker o anki işi bitirmeden çıkmaz.

## 9. Ölçeklenebilirlik ve dayanıklılık özellikleri

| Kavram | Ne yapar | Bu projede neden gerekli | Teslim |
|---|---|---|---|
| Reverse proxy (Nginx) | Tek giriş noktası, TLS, gzip, güvenlik başlıkları | API ve web dışarı açılmaz; IP bilgisi güvenilir hale gelir (rate limit buna dayanır) | M2 |
| Load balancing | İstekleri API kopyalarına dağıtır | Sabah 08:00'de tüm hastalar aynı anda "aldım"a basar: tepe yük | M2 |
| Liveness / readiness | `/health/live` süreç ayakta mı; `/health/ready` DB + Redis + migration sürümü hazır mı | Hazır olmayan kopyaya trafik gitmez | M2 |
| Rate limiting (token bucket) | Kullanıcı başına kova; Redis + Lua ile atomik; `X-RateLimit-*` başlıkları | Sabit pencerenin sınır patlamasını önler; kötüye kullanım ve maliyet kontrolü | M2 |
| Idempotency-Key | Tekrarlanan yazma isteği ikinci kez kayıt oluşturmaz | Zayıf mobil bağlantıda çift "aldım" uyum skorunu bozar | M2 |
| API sürümleme | `/v1/...` | PWA güncellenmeden backend değişebilir | M2 |
| ETag / `304` | Değişmeyen yanıt tekrar gönderilmez | Hekim paneli ve koç yoklaması sık yenilenir | M3 |
| Cursor sayfalama | Liste uçlarında `?cursor=&limit=` | Ölçüm ve doz geçmişi zamanla büyür | M3 |
| Mesaj kuyruğu + worker | LLM işleri API dışında | API isteği saniyelerce bloklanmaz | M4 |
| Retry + üstel backoff | Geçici hatada yeniden dener | LLM sağlayıcısı 429 / 5xx dönebilir | M4 |
| Dead-letter queue | 3 denemeden sonra iş DLQ'ya, metrikte görünür | Sessizce kaybolan koç mesajı olmaz | M4 |
| Circuit breaker | Art arda hatada dış servisi bir süre çağırmaz | LLM sağlayıcısı veya openFDA çökerse sistem yavaşlamaz, yedek yol hemen devreye girer | M4 |
| Dağıtık kilit (lider seçimi) | PostgreSQL advisory lock | Worker çoğaltılsa da hatırlatma tek kez gider | M4 |
| Graceful shutdown | SIGTERM'de süren işi bitir | Güncelleme sırasında yarım kalan koç işi olmaz | M4 |
| Gözlemlenebilirlik | Prometheus + Grafana, JSON log + request ID, OpenTelemetry + Jaeger | Bir koç isteğinin Nginx → API → kuyruk → worker → LLM yolu tek zaman çizelgesinde görülür | M2–M5 |
| Yük testi (k6) | 1 / 3 API kopyası ile istek/sn ve p95 gecikme ölçümü | Load balancing'in etkisi raporda sayıyla gösterilir | M5 |

### Rate limit politikası (taslak)

| Uç nokta grubu | Anahtar | Limit | Katman |
|---|---|---|---|
| Tüm istekler | IP | 20 istek/sn, 40 patlama | Nginx `limit_req` |
| `/auth/login` | IP + e-posta | 5/dk (IP), 10/15 dk (e-posta) | API |
| `/auth/register` | IP | 5/saat | API |
| Yazma uçları (doz, ölçüm) | kullanıcı | 60/dk | API token bucket |
| `/coach/chat` | kullanıcı | 10/dk + günlük token bütçesi | API token bucket + bütçe |
| Okuma uçları | kullanıcı | 300/dk | API token bucket |

### Yük testi planı (M5)

- **Senaryo:** 200 sanal hasta sabah 08:00 yoğunluğunu taklit eder: giriş → bugün ekranı → "aldım" → ölçüm girişi. 20 sanal hekim paneli yeniler.
- **Karşılaştırma:** `--scale api=1` ve `--scale api=3`; istek/sn, p95/p99 gecikme, hata oranı.
- **Dayanıklılık:** Test sürerken bir API kopyası durdurulur. Beklenti: hata oranı geçici olarak artar, sonra sıfıra iner. LLM sağlayıcısı taklidi hata verdirilir. Beklenti: devre açılır, koç yedek yanıtla devam eder.
- **Başarı ölçütü:** 3 kopyada p95 < 300 ms (LLM hariç uçlar), hata oranı < %1.

## 10. Planlanan monorepo yapısı

```
ilac-uyum-kocu/
├── backend/            FastAPI uygulaması (M2)
│   ├── app/            routers · services · middleware · models · deps
│   ├── alembic/        migration'lar
│   └── tests/          pytest + guardrail_scenarios.yaml (M4)
├── frontend/           Next.js PWA (M3)
├── infra/
│   ├── nginx/          reverse proxy + upstream yapılandırması (M2)
│   ├── prometheus/     scrape yapılandırması, Grafana panoları (M2–M5)
│   └── k6/             yük testi senaryoları (M5)
├── docs/               M1–M5 belgeleri, ER diyagramı
├── docker-compose.yml  (M2)
└── .github/workflows/  CI: pytest + guardrail seti + build (M2'den itibaren)
```

## 11. Tasarım kararları

1. **Uyum skoru deterministik.** LLM skoru hesaplamaz, tool ile okur. Böylece skor test edilebilir ve her seferinde aynıdır.
2. **Güvenlik LLM'e bırakılmaz.** Girdi ve çıktı filtreleri saf kod; red-team setiyle CI'da doğrulanır. Acil yol LLM'e ve kuyruğa hiç uğramaz; API anahtarı olmasa da, worker'lar dursa da çalışır.
3. **Hasta kimliğini tool'a sunucu bağlar.** Tool şemalarında `patient_id` parametresi yoktur; model başka bir hastanın verisini isteyemez.
4. **Kaynaklar sunucuda doğrulanır.** Modelin döndürdüğü `sources`, aynı turda `search_education`'ın gerçekten döndürdüğü kayıtlarla eşleşmezse atılır.
5. **Her dış bağımlılığın yedeği var.** Bütçe dolarsa, devre açıksa veya API hata verirse kural tabanlı koç devreye girer; openFDA yoksa önbellek kullanılır.
6. **Token tarayıcıya verilmez.** httpOnly çerez + aynı kökenli proxy; XSS ile token çalınması ve CORS yüzeyi ortadan kalkar.
7. **Yetkisiz erişim 404.** 403 yerine 404 döndürülerek başka hastaların varlığı sızdırılmaz.
8. **API durumsuz.** Tüm paylaşılan durum Redis veya PostgreSQL'de; kopya sayısı yük testine göre ayarlanır.
9. **Yazma uçları idempotent.** Mobil ağda tekrar gönderim normal kabul edilir; veri bütünlüğü istemciye bırakılmaz.
10. **Tek lider zamanlayıcı.** Hatırlatmalar advisory lock'u tutan worker'dan gider; `reminder_logs` tekil kısıtı ikinci bir güvencedir.
