# İlaç Uyumu ve Yaşam Tarzı Koçu

Tip 2 diyabet ve hipertansiyon hastalarının ilaç ve ölçüm takibini yaptığı, LLM koçun **tıbbi tavsiye vermeden** kişiselleştirilmiş, veriye dayalı ve kaynaklı geri bildirim verdiği, hekim için uyum dashboard'u sunan sağlık platformu.

**Öne çıkan mimari özellikler (plan)**

- **Güvenlik LLM'e bırakılmaz:** acil belirti ve doz soruları deterministik filtreyle LLM'den önce yakalanır; red-team seti CI'da koşar.
- **Ölçeklenebilir:** Nginx reverse proxy + load balancer arkasında durumsuz FastAPI kopyaları; token bucket rate limit.
- **Dayanıklı:** idempotent yazma uçları, kuyruk + worker, retry/backoff, dead-letter queue, circuit breaker, lider seçimli zamanlayıcı.
- **Gözlemlenebilir:** Prometheus + Grafana, OpenTelemetry + Jaeger; k6 ile yük testi.

> ⚠️ Koç teşhis koymaz ve ilaç dozu önermez. Acil durumlarda **112**'yi arayın. Proje yalnızca sentetik demo verisiyle çalışır.

Proje tanımı: [docs/PROJE_TANIMI.md](docs/PROJE_TANIMI.md)

## Durum

| Teslim | Tarih | Durum |
|---|---|---|
| M1 — Öneri + Mimari | 23 Eki 2026 | 🟡 Hazırlanıyor ([docs/M1](docs/M1)) |
| M2 — Veritabanı + Backend | 13 Kas 2026 | ⏳ |
| M3 — Arayüz MVP | 11 Ara 2026 | ⏳ |
| M4 — Yapay Zekâ Bileşeni | 1 Oca 2027 | ⏳ |
| M5 — Final Rapor + Demo | 17 Oca 2027 | ⏳ |

## M1 belgeleri

| Belge | İçerik |
|---|---|
| [01 — Proje önerisi](docs/M1/01-proje-onerisi.md) | Problem, hedef, kapsam, roller, teknoloji seçimi, riskler, teslim planı |
| [02 — Guardrail politikası](docs/M1/02-guardrail-politikasi.md) | Koçun asla yapmadıkları, 3 katmanlı kontrol, acil belirti kategorileri, sabit yanıtlar |
| [03 — Veri gizliliği](docs/M1/03-veri-gizliligi.md) | KVKK ilkeleri, rol bazlı erişim matrisi, LLM'e giden veri, güven sınırları |
| [04 — Kullanıcı akışları](docs/M1/04-kullanici-akislari.md) | Hasta, hekim, bakım veren ve acil belirti akışları |
| [05 — Sistem mimarisi](docs/M1/05-mimari.md) | Bağlam, bileşen, istek yolu, koç kuyruğu, circuit breaker, hatırlatma, openFDA, dağıtım; ölçeklenebilirlik/dayanıklılık, rate limit politikası, yük testi planı |
| [06 — ER diyagramı (taslak)](docs/M1/06-er-diyagrami.md) | Planlanan veri modeli |

## Mimari (özet)

![mimari-ozet](docs/M1/diyagramlar/mimari-ozet.png)

<details><summary>Mermaid kaynağı</summary>

```mermaid
flowchart LR
    U["Hasta · Hekim · Bakım veren<br/>Next.js PWA"] -- HTTPS --> NGX["Nginx<br/>reverse proxy · TLS<br/>load balancer"]
    NGX --> PX["Next.js<br/>httpOnly çerez"] --> NGX
    NGX --> API["FastAPI × N<br/>rate limit · idempotency<br/>JWT + roller · guardrail"]
    API --> DB[("PostgreSQL<br/>+ pgvector")]
    API --> RD[("Redis<br/>önbellek · kuyruk")]
    API --> FDA["openFDA"]
    RD --> WK["Worker × M<br/>koç · zamanlayıcı (lider)<br/>circuit breaker"]
    WK --> LLM["LLM sağlayıcısı<br/>tool use · M4'te seçilecek"]
    WK --> DB
    WK --> WP["Web Push (VAPID)"] -.-> U
```

</details>

Ayrıntı: [docs/M1/05-mimari.md](docs/M1/05-mimari.md)

## Teknoloji yığını

| Katman | Teknoloji |
|---|---|
| Frontend | Next.js (PWA), TypeScript, Recharts, shadcn/ui |
| Backend | FastAPI, SQLAlchemy, Alembic, APScheduler, pywebpush |
| LLM | Sağlayıcıdan bağımsız koç katmanı (tool use + structured output); model M3–M4'te adaylar denenerek seçilecek ([ayrıntı](docs/M1/01-proje-onerisi.md#51-llm-seçimi-deneyerek-karar)), pgvector (RAG) |
| Altyapı | PostgreSQL, Redis, Nginx (reverse proxy + LB), Docker Compose, GitHub Actions |
| Gözlemlenebilirlik / test | Prometheus, Grafana, OpenTelemetry, Jaeger, k6 |
