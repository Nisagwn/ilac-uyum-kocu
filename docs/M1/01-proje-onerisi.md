# M1 — Proje Önerisi ve Teknoloji Seçimi

**Proje:** Kronik Hastalar için İlaç Uyumu ve Yaşam Tarzı Koçu Platformu
**Hazırlayan:** Hayrunnisa Güven
**Teslim:** 23 Ekim 2026

## 1. Problem

Kronik hastalıklarda tedavi başarısının en büyük engeli ilaç uyumsuzluğudur. DSÖ'nün uzun süreli tedavilere uyum raporuna göre gelişmiş ülkelerde bile hastaların yaklaşık yarısı tedavilerine düzenli uymamaktadır. Tip 2 diyabet ve hipertansiyon gibi belirtisi az ama sonuçları ağır hastalıklarda bu oran daha da kritik hale gelir.

Mevcut uygulamalar büyük ölçüde hatırlatıcıdan ibarettir. Eksik olan katman:

- Hastanın kendi verisini **yorumlayan** ve onu **motive eden** bir geri bildirim,
- Hekime kontrol aralarında **anlamlı, özetlenmiş** bir uyum ve risk görünümü.

## 2. Hedef

Hastanın ilaç planını, kan şekeri/tansiyon ölçümlerini ve günlük notlarını girdiği; LLM tabanlı koçun bu veriye dayanarak **tıbbi tavsiye vermeden** motivasyonel geri bildirim, eğitim içeriği ve "hekime danışın" tetikleyicileri ürettiği; hekimin hastalarının uyum skorunu ve risk sinyallerini görebildiği bir platform.

### Örnek senaryo

1. Hasta sabah metformin dozunu "aldım" olarak işaretler, açlık kan şekerini girer; grafik güncellenir.
2. Koç, son 7 günde 3 atlanmış doz görür; motive edici bir mesajla "ilaç saatini kahvaltıya bağla" ipucu verir. Doz değişikliği önermez.
3. Hasta "göğsümde baskı var" yazar; guardrail acil yönlendirme (112) üretir ve hekime kritik risk bayrağı düşer.

### Başarı ölçütleri

| Ölçüt | Hedef |
|---|---|
| Guardrail test seti başarısı | En az 40 senaryo, tamamı geçer (CI'da zorunlu) |
| Acil belirti → 112 yönlendirmesi | %100, LLM çağrısından bağımsız |
| Koç yanıtlarında kaynak | Eğitim içeriği kullanılan her yanıtta en az 1 kaynak |
| Uyum skoru doğruluğu | Elle hesaplanmış örneklerle birebir (birim testleri) |
| Rol bazlı erişim | Yetkisiz erişim denemelerinin tümü 404 döner (testli) |
| Ölçeklenebilirlik | 3 API kopyasıyla p95 < 300 ms (LLM hariç uçlar), hata oranı < %1 (k6 yük testi) |
| Veri bütünlüğü | Aynı istek tekrar gönderildiğinde çift doz/ölçüm kaydı oluşmaz (idempotency testi) |

## 3. Kapsam

**Kapsam içi**
- İlaç planı, zamanlanmış dozlar, "aldım / almadım" kaydı
- Açlık/tokluk kan şekeri ve tansiyon ölçümleri, grafikler
- LLM koç (tool-use + structured output + RAG + guardrail)
- Hekim dashboard'u: hasta listesi, uyum skoru, 30 günlük eğriler, risk bayrakları, haftalık özet
- Bakım veren (hasta yakını) için salt okunur erişim
- openFDA ilaç etiketi entegrasyonu, web push hatırlatmaları

**Kapsam dışı**
- Teşhis koyma, ilaç dozu önerme veya değiştirme
- Glukometre / tansiyon aleti gibi cihaz entegrasyonları
- e-Nabız bağlantısı
- Native mobil uygulama (platform mobil öncelikli PWA olarak geliştirilir)

## 4. Hedef kullanıcılar ve roller

| Rol | Cihaz | Yetki |
|---|---|---|
| Hasta | Telefon (PWA) | Kendi ilaç, doz, ölçüm ve koç verisini okur/yazar |
| Hekim / diyabet hemşiresi | Masaüstü | Kendine atanmış hastaların verisini okur, uyarıları kapatır |
| Bakım veren | Telefon (PWA) | Bağlı olduğu hastanın verisini **salt okunur** görür |

## 5. Teknoloji seçimi ve gerekçeleri

| Katman | Seçim | Gerekçe |
|---|---|---|
| Frontend | Next.js (PWA), TypeScript, shadcn/ui, Recharts | Tek kod tabanıyla mobil (hasta) ve masaüstü (hekim) arayüzü; PWA ile ana ekrana ekleme ve web push; Recharts ölçüm grafikleri için yeterli |
| Backend | FastAPI, SQLAlchemy 2, Pydantic | Otomatik OpenAPI/Swagger; LLM ve RAG kütüphaneleriyle aynı dil (Python); tip güvenli şemalar |
| Veritabanı | PostgreSQL 16 + pgvector, Alembic | İlişkisel klinik veri ve vektör araması tek veritabanında; Alembic ile sürümlü şema |
| Zamanlayıcı | APScheduler + pywebpush (VAPID) | Hatırlatmalar için ayrı bir kuyruk ürününe gerek yok; PostgreSQL advisory lock ile birden çok worker'da tek lider çalışır |
| Önbellek / kuyruk | Redis | openFDA yanıt önbelleği, token bucket rate limit kovaları, koç iş kuyruğu + dead-letter queue, circuit breaker durumu |
| Reverse proxy / LB | Nginx | TLS sonlandırma, gzip, güvenlik başlıkları, API kopyaları arasında yük dağıtımı (`least_conn`) |
| Gözlemlenebilirlik | Prometheus + Grafana, OpenTelemetry + Jaeger | Metrik panoları ve bir isteğin servisler arası izinin (trace) görülmesi |
| Yük testi | k6 | Tek ve çok kopya karşılaştırması; dayanıklılık senaryoları |
| LLM | Sağlayıcıdan bağımsız koç katmanı; OpenRouter (tek anahtarla çok model) + doğrudan Anthropic desteği | Model henüz sabitlenmedi; adaylar aynı test setiyle denenip seçilecek (bkz. §5.1). Tool use ve JSON şemalı structured output destekleyen her model takılabilir |
| Altyapı | Docker Compose, GitHub Actions | Tek komutla kurulum; her push'ta test + guardrail seti + build |

### 5.1 LLM seçimi: deneyerek karar

Koçta kullanılacak model M1'de sabitlenmez. Koç katmanı sağlayıcıdan bağımsız yazılır: model adı yapılandırmadan (`LLM_MODEL`) değişir, kod değişmez. Seçim M3'teki prototip koçla başlayan denemelerden sonra **M4'te** aşağıdaki ölçütlerle yapılır ve sonuç M4 raporuna yazılır.

**Aday modeller**

| Aday | Erişim | Neden aday? | Dikkat edilecek nokta |
|---|---|---|---|
| DeepSeek Flash | OpenRouter | Çok düşük maliyet; tool calling ve JSON şema desteği | Sağlayıcının veri işleme konumu (bkz. [03 §3](03-veri-gizliligi.md#3-llme-giden-veri)); Türkçe kalite ölçülmeli |
| Claude Sonnet / Haiku | OpenRouter veya Anthropic API | Proje tanımında önerilen sağlayıcı; tool use ve structured output yerleşik | Maliyet daha yüksek |
| Ücretsiz modeller | OpenRouter | Maliyetsiz geliştirme ve deneme | Sıkı istek sınırları (429), modellerin kaldırılabilmesi, istemlerin kaydedilme olasılığı; demo için riskli |

**Değerlendirme ölçütleri**

| Ölçüt | Nasıl ölçülür | Kabul eşiği |
|---|---|---|
| Güvenlik | Guardrail red-team seti, Katman 3'ün engellediği yanıt oranı | Seti %100 geçer; engellenen yanıt oranı düşük |
| Tool-use doğruluğu | Senaryolarda doğru tool'u doğru parametreyle çağırma oranı | ≥ %90 |
| Structured output | JSON şemasına uymayan yanıt oranı | ≤ %2 |
| Kaynak gösterme | `sources` alanı dolu ve doğrulanmış yanıt oranı | Eğitim içeriği kullanılan yanıtların tamamı |
| Türkçe kalite | 20 örnek yanıtın elle puanlanması (anlaşılırlık, ton, motivasyon) | Ortalama ≥ 4/5 |
| Gecikme | p95 yanıt süresi | ≤ 10 sn |
| Maliyet | Mesaj başına ortalama USD (`llm_usage` kaydı) | Aylık bütçe tavanı içinde |
| Gizlilik | Sağlayıcının veri saklama ve eğitimde kullanma politikası | Eğitimde kullanmama seçeneği olmalı |

Güvenlik garantisi modelden gelmez: acil belirti ve doz tespiti LLM'den önce, çıktı doğrulaması LLM'den sonra deterministik kodla yapılır. Bu yüzden hangi model seçilirse seçilsin guardrail davranışı aynı kalır; modeller kalite, maliyet ve hız açısından karşılaştırılır. Seçilen model dışındakiler yapılandırmayla seçilebilir alternatif olarak kalır.

## 6. Riskler ve önlemler

| Risk | Önlem |
|---|---|
| LLM'in doz önermesi veya teşhis koyması | 3 katmanlı guardrail (bkz. [02-guardrail-politikasi.md](02-guardrail-politikasi.md)), çıktı doğrulayıcı, CI'da red-team testi |
| Acil belirtinin gözden kaçması | Acil tespiti LLM'den **önce**, deterministik kurallarla yapılır |
| Hasta verisinin sızması | Rol bazlı yetki, LLM'e kimlik bilgisi gönderilmemesi (bkz. [03-veri-gizliligi.md](03-veri-gizliligi.md)) |
| openFDA'nın ABD kaynaklı olması | Türk ticari isimleri için etken madde eşleme tablosu; raporda sınırlılık olarak belirtilir |
| LLM maliyetinin kontrolden çıkması | Hasta başına günlük istek/token limiti ve aylık bütçe; her çağrının maliyeti veritabanında kaydedilir |
| LLM servisine erişilememesi | Circuit breaker devreyi açar, kural tabanlı yedek koç hemen devreye girer; guardrail'ler yedekte de geçerlidir |
| Tepe yük (sabah ilaç saati) | API durumsuz ve yatayda çoğaltılır; Nginx yükü dağıtır; kullanıcı başına rate limit |
| Zayıf mobil bağlantıda tekrar gönderim | Yazma uçlarında `Idempotency-Key`; çift kayıt oluşmaz |
| Başarısız arka plan işleri | Retry + üstel backoff, 3 denemeden sonra dead-letter queue ve metrik |

## 7. Ara teslim planı

| Teslim | Tarih | Çıktı |
|---|---|---|
| M1 | 23 Eki 2026 | Bu belge seti: öneri, kullanıcı akışları, guardrail politikası, veri gizliliği, mimari ve ER diyagramı |
| M2 | 13 Kas 2026 | Şema + migration, JWT rolleri, ilaç/doz/ölçüm CRUD, openFDA uç noktası, uyum skoru, Swagger, CI; Nginx reverse proxy + load balancing, liveness/readiness, token bucket rate limit, idempotency, `/v1` sürümleme |
| M3 | 11 Ara 2026 | Hasta günlük akışı ve hekim dashboard'u uçtan uca, PWA + web push, prototip koç; ETag / `304`, cursor sayfalama |
| M4 | 1 Oca 2027 | LLM aday karşılaştırması ve model seçimi, tool-use koç, RAG, 40 senaryoluk guardrail seti, uyum skoru doğrulaması, maliyet limiti, haftalık özet; kuyruk + worker, retry/backoff, dead-letter queue, circuit breaker, lider kilidi, graceful shutdown |
| M5 | 17 Oca 2027 | Final rapor (etik/güvenlik, sınırlılıklar, AI Tools Usage), demo videosu, kurulum talimatı; k6 yük testi raporu, OpenTelemetry izleme, Grafana panoları |
