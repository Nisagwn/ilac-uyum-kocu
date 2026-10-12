# Sistem Mimarisi

Bu belge M1 aşamasında **planlanan** mimariyi anlatır. Bileşen adları ve sınırlar M2–M4'te koda dönüşürken bu belge referans alınır; değişiklik olursa burada güncellenir.

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

    LLM["Claude API<br/>tool use + structured output"]
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
        SW["Service Worker<br/>push alma + offline kabuk<br/>sağlık verisi önbelleğe alınmaz"]
    end

    subgraph Next["Next.js sunucusu"]
        AUTHN["Giriş / çıkış<br/>JWT → httpOnly çerez"]
        PROXY["API proxy<br/>çerez → Authorization: Bearer"]
    end

    subgraph API["FastAPI (REST + OpenAPI)"]
        SEC["Kimlik doğrulama + rol kontrolü<br/>yetkisiz → 404"]
        CRUD["/patients · /medications · /doses<br/>/measurements · /alerts · /summaries"]
        CHAT["/coach/chat"]
        DRUG["/drugs/label"]
        PUSHR["/push/subscribe"]
        TH["Eşik kontrolü<br/>eşik dışı ölçüm → alert"]
        ADH["Uyum skoru<br/>deterministik"]
        GR["Guardrail<br/>Katman 1 girdi · Katman 3 çıktı"]
        COACH["Koç orkestrasyonu<br/>tool-use döngüsü<br/>↘ kural tabanlı yedek"]
        TOOLS["Tool'lar<br/>get_adherence · get_measurements<br/>get_medications · search_education"]
        BUD["Maliyet / limit kontrolü"]
        RAG["RAG<br/>embedding + pgvector arama"]
        SUM["Haftalık özet<br/>istatistik + LLM anlatısı"]
        OFS["openFDA servisi<br/>ad → etken madde → etiket"]
    end

    subgraph Worker["Worker süreci"]
        SCH["APScheduler<br/>hatırlatma taraması"]
        PUSH["pywebpush (VAPID)"]
    end

    DB[("PostgreSQL + pgvector")]
    RD[("Redis<br/>önbellek · sayaç · kuyruk")]
    CL["Claude API"]
    FDA["openFDA"]
    WPS["Push servisi"]

    UI_P & UI_D & UI_C --> PROXY
    UI_P --> AUTHN
    PROXY --> SEC
    AUTHN --> SEC
    SEC --> CRUD & CHAT & DRUG & PUSHR
    CRUD --> TH & ADH & SUM
    CRUD --> DB
    CHAT --> GR --> COACH
    COACH --> BUD
    COACH --> CL
    CL -- tool_use --> TOOLS
    TOOLS --> ADH & RAG
    TOOLS --> DB
    RAG --> DB
    SUM -.-> CL
    DRUG --> OFS --> FDA
    OFS -.önbellek.-> RD
    OFS -- "etiket özeti → eğitim içeriği" --> DB
    BUD --> DB
    SEC -. istek sınırı .-> RD
    PUSHR --> DB
    SCH --> DB
    SCH --> PUSH --> WPS --> SW
```

</details>

| Bileşen | Sorumluluk |
|---|---|
| Next.js PWA | Hasta (mobil), hekim (masaüstü) ve bakım veren arayüzleri; grafikler Recharts ile |
| Next.js proxy | Token'ı tarayıcıdan saklar; tüm API çağrılarını aynı kökenden geçirir |
| Rol kontrolü | Her istekte JWT doğrulama ve hasta-erişim kuralı; yetkisizse 404 |
| Guardrail | LLM'den önce acil/doz tespiti, LLM'den sonra çıktı doğrulama |
| Koç | Claude ile tool-use döngüsü; structured output üretir |
| Uyum skoru | Planlanan ve kaydedilen dozlardan 7/30 günlük skor; LLM hesaplamaz |
| RAG | Kürate eğitim içeriği + openFDA özetleri üzerinde vektör arama |
| openFDA servisi | İlaç adından etken madde, uyarılar ve etkileşimleri çeker |
| Worker | Zamanlanmış hatırlatmalar ve uzun süren işler; API'yi bloklamaz |

## 3. Koç isteğinin yaşam döngüsü

![dizi-koc-istegi](diyagramlar/dizi-koc-istegi.png)

<details><summary>Mermaid kaynağı</summary>

```mermaid
sequenceDiagram
    autonumber
    participant H as Hasta (PWA)
    participant N as Next.js proxy
    participant A as FastAPI /coach/chat
    participant G as Guardrail (deterministik)
    participant B as Bütçe / limit
    participant L as Claude API
    participant T as Tool'lar
    participant D as PostgreSQL

    H->>N: mesaj (httpOnly çerez)
    N->>A: Authorization: Bearer JWT
    A->>A: rol = hasta? hasta kaydı JWT'den bulunur
    A->>G: Katman 1: girdi kontrolü
    alt acil belirti / kendine zarar
        G-->>A: sabit 112 yanıtı (LLM çağrılmaz)
        A->>D: kritik alert → hekim listesinin en üstü
    else doz / ilaç değişikliği talebi
        G-->>A: sabit "hekiminize danışın" yanıtı + bilgi alert'i
    else temiz
        A->>B: günlük istek/token + aylık bütçe kontrolü
        alt limit aşıldı / API erişilemiyor
            B-->>A: kural tabanlı yedek koç
        else
            A->>L: system prompt + tool tanımları + mesaj (ad/e-posta yok)
            loop tool_use (en fazla N tur)
                L->>T: get_adherence / get_measurements / get_medications / search_education
                T->>D: sorgu (patient_id sunucuda bağlı)
                T-->>L: sayısal özet / eğitim parçaları
            end
            L-->>A: {message, risk_level, escalate_to_clinician, sources}
            A->>A: sources gerçekten search_education'dan mı geldi?
            A->>G: Katman 3: çıktı doğrulama (doz/teşhis ifadesi?)
        end
    end
    A->>D: coach_messages (+ alert, + maliyet kaydı)
    A-->>H: yanıt + kaynaklar
```

</details>

## 4. Hatırlatma (web push) akışı

![dizi-web-push](diyagramlar/dizi-web-push.png)

<details><summary>Mermaid kaynağı</summary>

```mermaid
sequenceDiagram
    autonumber
    participant H as Hasta tarayıcısı
    participant SW as Service Worker
    participant A as FastAPI
    participant D as PostgreSQL
    participant W as Worker (APScheduler)
    participant P as Push servisi

    H->>SW: bildirim izni + PushManager.subscribe(VAPID public key)
    H->>A: POST /push/subscribe {endpoint, keys}
    A->>D: push_subscriptions
    loop her dakika
        W->>D: zamanı gelmiş ve henüz kaydı olmayan dozlar
        W->>D: bu doz için bugün hatırlatma gönderildi mi?
        W->>P: şifreli yük (VAPID imzalı) — sağlık verisi yok
        W->>D: reminder_logs (tekrar göndermeyi önler)
    end
    P-->>SW: push olayı
    SW-->>H: "İlaç zamanı" bildirimi → Bugün ekranı
```

</details>

## 5. openFDA entegrasyonu

![dizi-openfda](diyagramlar/dizi-openfda.png)

<details><summary>Mermaid kaynağı</summary>

```mermaid
sequenceDiagram
    autonumber
    participant U as Hasta / hekim
    participant A as FastAPI /drugs/label
    participant R as Redis
    participant F as openFDA
    participant D as PostgreSQL

    U->>A: ilaç adı (ör. "Glucophage")
    A->>A: Türk ticari adı → etken madde (eşleme tablosu)
    A->>R: önbellekte var mı?
    alt önbellekte yok
        A->>F: /drug/label.json?search=openfda.generic_name:"metformin"
        F-->>A: etiket (indications, warnings, drug_interactions)
        A->>R: 24 saat önbellek
    end
    A->>D: özet → education_contents (kind = openfda)
    A-->>U: etken madde, uyarılar, etkileşimler
```

</details>

openFDA yanıtı yalnızca **etken madde adıyla** sorgulanır; hastaya ait hiçbir veri dışarı gönderilmez.

## 6. Dağıtım (Docker Compose)

![dagitim-docker](diyagramlar/dagitim-docker.png)

<details><summary>Mermaid kaynağı</summary>

```mermaid
flowchart LR
    subgraph host["Docker Compose"]
        web["web<br/>Next.js :3000"]
        api["api<br/>FastAPI :8000"]
        worker["worker<br/>APScheduler + koç kuyruğu"]
        migrate["migrate<br/>alembic upgrade head<br/>(tek seferlik)"]
        db[("db<br/>PostgreSQL 16 + pgvector")]
        redis[("redis")]
    end
    user(["Tarayıcı"]) -- ":3000" --> web
    web -- "iç ağ" --> api
    api --> db & redis
    worker --> db & redis
    migrate --> db
    api -. dış .-> ext["Claude · openFDA"]
    worker -. dış .-> ps["Push servisi"]
```

</details>

| Servis | Görev | Dışarı açık mı? |
|---|---|---|
| `web` | Arayüz + proxy | Evet (3000) |
| `api` | REST API, Swagger `/docs` | Yalnızca yerel geliştirmede |
| `worker` | Hatırlatma zamanlayıcısı, uzun koç işleri | Hayır |
| `migrate` | Başlangıçta şema güncelleme | Hayır |
| `db` | Tüm kalıcı veri | Hayır (yalnızca yerel geliştirme portu) |
| `redis` | Önbellek, sayaç, kuyruk | Hayır |

## 7. Planlanan monorepo yapısı

```
ilac-uyum-kocu/
├── backend/            FastAPI uygulaması (M2)
│   ├── app/            routers · services · models · deps
│   ├── alembic/        migration'lar
│   └── tests/          pytest + guardrail_scenarios.yaml (M4)
├── frontend/           Next.js PWA (M3)
├── docs/               M1–M5 belgeleri, ER diyagramı
├── docker-compose.yml  (M2)
└── .github/workflows/  CI: pytest + guardrail seti + build (M2'den itibaren)
```

## 8. Tasarım kararları

1. **Uyum skoru deterministik.** LLM skoru hesaplamaz, tool ile okur. Böylece skor test edilebilir ve her seferinde aynıdır.
2. **Güvenlik LLM'e bırakılmaz.** Girdi ve çıktı filtreleri saf kod; red-team setiyle CI'da doğrulanır. Acil yol LLM'e hiç uğramaz, API anahtarı olmasa da çalışır.
3. **Hasta kimliğini tool'a sunucu bağlar.** Tool şemalarında `patient_id` parametresi yoktur; model başka bir hastanın verisini isteyemez.
4. **Kaynaklar sunucuda doğrulanır.** Modelin döndürdüğü `sources`, aynı turda `search_education`'ın gerçekten döndürdüğü kayıtlarla eşleşmezse atılır.
5. **Her LLM yolunun yedeği var.** Bütçe dolarsa veya API hata verirse kural tabanlı koç devreye girer; haftalık özet anlatısı üretilemezse istatistikten şablon metin oluşur.
6. **Token tarayıcıya verilmez.** httpOnly çerez + aynı kökenli proxy; XSS ile token çalınması ve CORS yüzeyi ortadan kalkar.
7. **Yetkisiz erişim 404.** 403 yerine 404 döndürülerek başka hastaların varlığı sızdırılmaz.
8. **Uzun işler worker'da.** Zamanlayıcı ve LLM araç döngüsü ayrı süreçte koşar; API birden fazla kopya çalışsa bile hatırlatma bir kez gönderilir.
